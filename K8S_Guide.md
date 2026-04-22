以下是针对 **K8S 小白** 的**完整、详细、可直接复制操作**指南。我会按「不同需求场景」一步一步教你部署 **Nginx**，并解释每个场景的 **HA（高可用）**、**冗余**、**负载均衡** 作用。

你的集群情况：
- 1 个 Master（控制面）节点 + 2 个 Worker 节点
- **操作方式完全一样**，不管节点数量是 1+2、3+3 还是 1+10，都用同一套 YAML 和命令。
- **唯一区别**：Master 默认有 taint（污点），Pod 不会调度到 Master 上，只能跑到 2 个 Worker 上。这对我们反而是好事——天然实现「跨节点冗余」。

**前提准备**（必须先做一次）：
```bash
# 1. 确认集群状态
kubectl get nodes -o wide
# 你应该看到 3 个节点，Master 的 ROLE 是 control-plane，两个 Worker 是 <none>

# 2. 确认 kubectl 可以用（小白最常见问题）
kubectl version
```

---

### **场景 1：最基础冗余 + 内部负载均衡（推荐新手先跑这个）**

目标：
- 3 个 Nginx Pod（冗余）
- 自动负载均衡（Service）
- Pod 尽量分散到 2 个 Worker（HA）

#### 步骤 1：创建 Deployment（带反亲和性，保证跨节点）
新建文件 `nginx-deployment-ha.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                          # ← 冗余：同时跑 3 个实例
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25          # 官方最新稳定版
        ports:
        - containerPort: 80
      # === 关键：跨节点 HA 反亲和性 ===
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - nginx
              topologyKey: kubernetes.io/hostname   # 按节点名分散
```

#### 步骤 2：创建 Service（内部负载均衡）
新建文件 `nginx-service.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP          # 内部使用（最简单）
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

#### 步骤 3：部署并验证
```bash
kubectl apply -f nginx-deployment-ha.yaml
kubectl apply -f nginx-service.yaml

# 查看状态（重点看）
kubectl get pods -o wide          # 应该看到 3 个 Pod 分布在 2 个不同 Worker 上
kubectl get svc                   # 看到 nginx-service 的 CLUSTER-IP
kubectl get deployment
```

**测试内部负载均衡**：
```bash
kubectl run curl-pod --image=curlimages/curl -it --rm -- sh
# 在容器里执行：
curl http://nginx-service
# 每次刷新看到的是不同 Pod 的 hostname（证明负载均衡生效）
```

---

### **场景 2：外部访问 + NodePort 负载均衡（最常用生产方式）**

把上面的 Service 改成 NodePort，外部浏览器就能访问。

修改 `nginx-service.yaml`（只需改 type 和 nodePort）：

```yaml
spec:
  type: NodePort                # ← 改这里
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080             # ← 30000-32767 任意一个（推荐 30080）
```

重新应用：
```bash
kubectl apply -f nginx-service.yaml
```

访问方式（在你电脑浏览器或 curl）：
```
http://Worker节点IP:30080
# 例如：http://192.168.10.21:30080
```

**为什么 HA？**
- 任意一个 Worker 挂掉，另外一个还能继续提供服务（因为有 3 个 Pod）
- Service 会自动把流量转发到存活的 Pod

---

### **场景 3：真正生产级 HA + Ingress（推荐 2 个 Worker 以上使用）**

NodePort 缺点：每个节点都要暴露端口，端口冲突麻烦。  
**Ingress + Ingress Controller** 是 Kubernetes 官方推荐的「外部负载均衡 + 域名路由」方式。

#### 步骤（分两步）：

**A. 安装 Nginx Ingress Controller（只需要做一次）**
```bash
# 使用官方 Helm 方式（最简单）
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

等待 1-2 分钟：
```bash
kubectl get pods -n ingress-nginx
```

**B. 创建 Ingress 资源**
新建 `nginx-ingress.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.yourdomain.com          # ← 改成你自己的域名或 hosts 解析
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

```bash
kubectl apply -f nginx-ingress.yaml
```

**访问**：
- 把域名解析到任意 Worker IP（或用 hosts 文件）
- 浏览器打开 `http://nginx.yourdomain.com` → 自动负载均衡 + HA

**优势**：
- 支持 HTTPS（后面加 cert-manager 即可）
- 一个 Ingress Controller 可以代理无数个服务
- 自动健康检查、灰度发布等高级功能

---

### **场景 4：极致冗余 + DaemonSet（每个节点必须跑一个 Nginx）**

如果你希望「每个 Worker 节点都必须有一个 Nginx」（比如边缘节点场景）：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
spec:
  selector:
    matchLabels:
      app: nginx-daemon
  template:
    metadata:
      labels:
        app: nginx-daemon
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

配合 `type: NodePort` 的 Service 使用即可。

---

### **常见问题 & 小白排查命令**

| 问题 | 命令 |
|------|------|
| Pod 没起来 | `kubectl describe pod <pod-name>` |
| 调度不到节点 | `kubectl describe pod` 看 Events（通常是资源不够或 taint） |
| Service 访问不通 | `kubectl get endpoints nginx-service` |
| 想删掉重来 | `kubectl delete -f nginx-deployment-ha.yaml` |

**资源限制建议**（防止 Worker 被打满）：
在 Deployment 的 container 里加：
```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "200m"
    memory: "256Mi"
```

---

**总结：你的 1 Master + 2 Worker 集群完全够用**
- 用 **Deployment + Service (NodePort)** → 90% 场景足够
- 想更优雅 → **加 Ingress**
- 所有操作都**不依赖节点数量**，YAML 完全通用

现在就把上面 **场景 1** 的两个 YAML 文件复制下来，`kubectl apply` 跑一遍吧！  
跑完把 `kubectl get pods -o wide` 的结果贴给我，我帮你确认是否真的实现了跨节点 HA。

需要我再给你 **HTTPS + 域名**、**自动扩缩容 HPA**、**ConfigMap 挂载配置文件** 等进阶版，也可以继续问我！加油，你已经比 90% 小白强了！🚀