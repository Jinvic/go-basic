# 13-Kubernetes 基础入门

> 💡 **核心提示**：K8s（Kubernetes）是容器编排平台，负责管理成百上千个容器（Docker）的部署、扩缩容、自愈、网络通信。把它理解为"Docker 的管家 + 调度员 + 运维机器人"。

## 1. 为什么需要 K8s？

**没有 K8s 的痛点**：

```
场景：你有一个 Go 应用，要部署 10 个实例

手动运维：
1. 登录 10 台服务器，逐个 docker run
2. 其中一台挂了 → 手动发现 → 手动重启
3. 要升级版本 → 10 台挨个停止 → 拉新镜像 → 启动
4. 流量暴涨 → 手动加机器 → 手动部署新实例
5. 服务互相调用 → IP 写死在配置里 → 一台挂了全乱套

太累了！根本维护不过来！
```

**有 K8s 后**：

```yaml
# 只需要写一个配置文件，告诉 K8s "我要什么"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 10  # 我要 10 个实例

# 然后执行：kubectl apply -f deployment.yaml
# K8s 自动：
# ✅ 创建 10 个 Pod
# ✅ 某个挂了自动重启
# ✅ 滚动升级（先启动新的，再停旧的）
# ✅ 自动负载均衡
```

---

## 2. 核心概念详解

### 2.1 Pod：最小部署单元

**Pod 是什么？**

Pod = 一个或多个容器的组合，共享网络和存储。

```
┌──────────────────────────────────────────┐
│ Pod (my-app-pod)                          │
│ ┌────────────────┐  ┌────────────────┐   │
│ │  Container 1   │  │  Container 2   │   │
│ │  (my-app)      │  │  (sidecar)     │   │
│ │  Port: 8080    │  │  日志收集       │   │
│ └────────────────┘  └────────────────┘   │
│                                          │
│ 共享：                                    │
│ - 网络命名空间（localhost 互通）           │
│ - 存储卷（Volume）                        │
│ - IP 地址（一个 Pod 一个 IP）             │
└──────────────────────────────────────────┘
```

**为什么不直接用 Container？**

```
场景：你的应用需要一个"边车"容器收集日志

方案1（两个独立容器）：
- 两个不同的 IP，需要通过网络通信
- 日志文件需要通过网络传输

方案2（同一个 Pod）：
- 共享 localhost，直接 127.0.0.1 通信
- 共享 Volume，日志文件直接读取
- 生命周期一致，一起启动、一起销毁

Pod 就是为了让"关系密切的容器"更方便协作！
```

**Pod 的生命周期**：

```
Pending → Running → Succeeded/Failed
   ↓
 调度中    运行中     结束（正常/异常）
```

### 2.2 Deployment：Pod 的管理器

**Deployment 是什么？**

Deployment = 声明式地管理 Pod 的"控制器"。它确保：

- Pod 数量始终符合预期（replicas）
- Pod 版本正确（滚动更新）
- 有问题时自动回滚

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app           # Deployment 名称
spec:
  replicas: 3            # 我要 3 个 Pod
  selector:
    matchLabels:
      app: my-app        # 管理带有 app=my-app 标签的 Pod
  template:              # Pod 模板
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:v1.0.0   # 使用这个镜像
        ports:
        - containerPort: 8080
```

**Deployment 的魔法**：

```
你告诉 Deployment：我要 3 个 Pod

当前状态          Deployment 动作           目标状态
─────────────────────────────────────────────────────
0 个 Pod    →    创建 3 个 Pod        →    3 个 Pod ✅
3 个 Pod    →    什么都不做           →    3 个 Pod ✅
2 个 Pod    →    创建 1 个 Pod        →    3 个 Pod ✅
4 个 Pod    →    删除 1 个 Pod        →    3 个 Pod ✅
Pod 挂了    →    检测到后自动重建     →    3 个 Pod ✅
```

**滚动更新（Rolling Update）**：

```
升级 v1 → v2，不中断服务：

时刻1: Pod(v1) Pod(v1) Pod(v1)           3个旧版本
时刻2: Pod(v1) Pod(v1) Pod(v1) Pod(v2)   先启动1个新的
时刻3: Pod(v1) Pod(v1) Pod(v2) Pod(v2)   启动第2个，删除1个旧的
时刻4: Pod(v1) Pod(v2) Pod(v2) Pod(v2)   继续...
时刻5: Pod(v2) Pod(v2) Pod(v2)           3个新版本，完成！

全程服务不中断！
```

### 2.3 Service：稳定的访问入口

**问题**：Pod 的 IP 是动态的，怎么稳定访问？

```
Pod 重启 → IP 变了（10.244.1.1 → 10.244.2.3）
Pod 扩容 → 多了新的 IP
Pod 缩容 → 少了一些 IP

如果客户端写死 IP，那就完蛋了！
```

**Service 的解决方案**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app          # 选择带有 app=my-app 标签的 Pod
  ports:
  - port: 80             # Service 暴露的端口
    targetPort: 8080     # Pod 实际监听的端口
  type: ClusterIP        # 只在集群内部可访问
```

**Service 提供两样东西**：

```
1. 稳定的 ClusterIP（虚拟 IP）
   10.96.100.1  ← 永远不变

2. 稳定的 DNS 名称
   my-app-service.default.svc.cluster.local  ← 永远不变
   
其他服务调用：http://my-app-service:80
K8s 自动负载均衡到后面的 Pod
```

**Service 类型**：

| 类型 | 作用 | 访问范围 |
|------|------|----------|
| **ClusterIP** | 集群内部虚拟 IP | 只有集群内部可访问（默认） |
| **NodePort** | 在每个 Node 上开一个端口 | 集群外可通过 NodeIP:Port 访问 |
| **LoadBalancer** | 云厂商提供的负载均衡器 | 公网可访问（要花钱） |
| **Ingress** | HTTP 层路由，支持域名、路径 | 公网可访问，比 LB 更省钱 |

### 2.4 ConfigMap 与 Secret：配置管理

**问题**：配置不应该写死在镜像里！

```
❌ 错误做法：
Dockerfile 里写死 DB_HOST=192.168.1.100
测试环境、生产环境都要重新构建镜像

✅ 正确做法：
镜像里只放代码，配置从外部注入
ConfigMap / Secret → 注入到 Pod → 环境变量或文件
```

**ConfigMap（普通配置）**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  DB_HOST: "mysql.default.svc.cluster.local"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
    database:
      host: mysql
```

**Secret（敏感配置）**：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # base64 编码
  API_KEY: c2VjcmV0a2V5MTIz
```

**在 Pod 中使用**：

```yaml
spec:
  containers:
  - name: my-app
    image: my-app:v1
    env:
    # 从 ConfigMap 读取
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: my-app-config
          key: DB_HOST
    # 从 Secret 读取
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-app-secret
          key: DB_PASSWORD
    # 挂载为文件
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: my-app-config
```

---

## 3. kubectl 常用命令

### 3.1 查看资源

```bash
# 查看所有 Pod
kubectl get pods
kubectl get pods -o wide           # 显示更多信息（IP、Node）
kubectl get pods -n kube-system    # 指定 namespace

# 查看所有资源
kubectl get all

# 查看 Deployment
kubectl get deployments
kubectl get deploy                 # 简写

# 查看 Service
kubectl get services
kubectl get svc                    # 简写

# 查看详情
kubectl describe pod <pod-name>
kubectl describe deploy <deploy-name>
```

### 3.2 创建/更新资源

```bash
# 应用配置文件
kubectl apply -f deployment.yaml
kubectl apply -f .                 # 应用当前目录所有 yaml

# 删除资源
kubectl delete -f deployment.yaml
kubectl delete pod <pod-name>
kubectl delete deploy <deploy-name>

# 快速创建（测试用）
kubectl run nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=3
```

### 3.3 调试/排查

```bash
# 查看 Pod 日志
kubectl logs <pod-name>
kubectl logs <pod-name> -f         # 实时跟踪
kubectl logs <pod-name> -c <container>  # 多容器时指定

# 进入 Pod 执行命令
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- bash

# 端口转发（本地调试）
kubectl port-forward <pod-name> 8080:8080
kubectl port-forward svc/<service-name> 8080:80

# 查看事件（排查问题神器）
kubectl get events --sort-by='.lastTimestamp'
```

### 3.4 扩缩容/更新

```bash
# 扩缩容
kubectl scale deploy <name> --replicas=5

# 更新镜像
kubectl set image deploy/<name> <container>=<new-image>

# 查看滚动更新状态
kubectl rollout status deploy/<name>

# 回滚到上一版本
kubectl rollout undo deploy/<name>

# 查看历史版本
kubectl rollout history deploy/<name>
```

---

## 4. 实战：部署 ServiceTelemetry 项目

假设你的 ServiceTelemetry 项目要部署到 K8s，需要以下文件：

### 4.1 Deployment（k8s/deployment.yaml）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-telemetry
  labels:
    app: service-telemetry
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-telemetry
  template:
    metadata:
      labels:
        app: service-telemetry
    spec:
      containers:
      - name: service-telemetry
        image: service-telemetry:latest
        ports:
        - containerPort: 8080
        env:
        - name: GIN_MODE
          value: "release"
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:          # 存活检查
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:         # 就绪检查
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
```

### 4.2 Service（k8s/service.yaml）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-telemetry
spec:
  selector:
    app: service-telemetry
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### 4.3 ConfigMap（k8s/configmap.yaml）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-telemetry-config
data:
  config.yaml: |
    server:
      port: 8080
    probes:
      - name: example-api
        type: http
        url: https://api.example.com/health
        interval: 30s
        timeout: 10s
```

### 4.4 部署命令

```bash
# 1. 构建镜像
docker build -t service-telemetry:latest .

# 2. 应用配置
kubectl apply -f k8s/

# 3. 查看状态
kubectl get pods -l app=service-telemetry
kubectl get svc service-telemetry

# 4. 本地测试
kubectl port-forward svc/service-telemetry 8080:80
curl http://localhost:8080/health
```

---

## 5. 核心概念对照表

| 概念 | 作用 | 类比 |
|------|------|------|
| **Pod** | 运行容器的最小单元 | 一个房间 |
| **Deployment** | 管理 Pod 的数量和版本 | 物业管理 |
| **Service** | 提供稳定的访问入口 | 门牌号/前台 |
| **ConfigMap** | 存储配置 | 配置文件柜 |
| **Secret** | 存储敏感信息 | 保险箱 |
| **Namespace** | 资源隔离 | 不同楼栋 |
| **Node** | 物理/虚拟机 | 一栋楼 |
| **Ingress** | HTTP 路由入口 | 小区大门保安 |

---

## 6. 常见问题

### Q1: Pod 一直 Pending？

```bash
kubectl describe pod <name>  # 看 Events

常见原因：
1. 资源不足：Node 的 CPU/内存不够
2. 镜像拉取失败：镜像名错误 / 私有仓库没配置认证
3. 调度失败：NodeSelector / Affinity 配置错误
```

### Q2: Pod 一直 CrashLoopBackOff？

```bash
kubectl logs <name>  # 看应用日志

常见原因：
1. 应用启动失败（配置错误、依赖服务没起来）
2. 健康检查失败（livenessProbe 配置不对）
3. OOMKilled（内存超限）
```

### Q3: Service 访问不通？

```bash
kubectl get endpoints <service-name>  # 看是否有 Endpoint

常见原因：
1. selector 没匹配到 Pod
2. Pod 没有通过 readinessProbe
3. 端口配置错误
```
