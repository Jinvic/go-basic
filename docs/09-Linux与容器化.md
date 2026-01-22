# 9. Linux与容器化

## 9.1 核心知识点

### Linux基础命令
```bash
文件操作：ls, cd, pwd, mkdir, rm, cp, mv, touch
文件查看：cat, less, head, tail, grep
文件权限：chmod, chown
进程管理：ps, top, kill, pkill
网络命令：ping, netstat, curl, wget
系统信息：uname, df, du, free
```

### Docker基础
```bash
docker build -t myapp:1.0 .
docker run -d -p 8080:8080 --name myapp myapp:1.0
docker ps
docker logs myapp
docker exec -it myapp /bin/bash
docker stop myapp
docker rm myapp
docker rmi myapp:1.0
```

### Dockerfile
```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/main .
EXPOSE 8080
CMD ["./main"]
```

### Kubernetes基础
```bash
kubectl get pods
kubectl get services
kubectl get deployments
kubectl apply -f deployment.yaml
kubectl logs -f pod-name
kubectl exec -it pod-name -- /bin/bash
kubectl scale deployment myapp --replicas=3
kubectl delete pod pod-name
```

### Kubernetes资源
```yaml
Deployment:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080

Service:
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer

ConfigMap:
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  database.url: "postgres://localhost:5432/mydb"
  cache.enabled: "true"

Secret:
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  password: cGFzc3dvcmQ=
```

## 9.2 常见面试题

### Q1: 如何编写高效的Dockerfile？

**解题思路**：
1. 说明Dockerfile的最佳实践
2. 解释多阶段构建的优势
3. 演示如何优化镜像大小
4. 说明如何利用Docker缓存

**代码实现**：
```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/main .

EXPOSE 8080

CMD ["./main"]
```

**常见错误分析**：
- 错误1：不使用多阶段构建导致镜像过大
- 错误2：COPY顺序不当导致缓存失效
- 错误3：忘记设置EXPOSE端口

### Q2: Kubernetes的Deployment和StatefulSet有什么区别？

**解题思路**：
1. 解释Deployment的基本特性（无状态应用）
2. 解释StatefulSet的基本特性（有状态应用）
3. 对比两者的适用场景
4. 说明StatefulSet的稳定网络标识和持久化存储

**代码实现**：
```yaml
Deployment（无状态应用）：
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: webapp:1.0
        ports:
        - containerPort: 8080

StatefulSet（有状态应用）：
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

**常见错误分析**：
- 错误1：混淆Deployment和StatefulSet的适用场景
- 错误2：StatefulSet忘记配置serviceName
- 错误3：不理解StatefulSet的有序部署和扩展

### Q3: 如何在Kubernetes中实现配置管理？

**解题思路**：
1. 说明ConfigMap和Secret的用途
2. 演示如何创建ConfigMap和Secret
3. 说明如何在Pod中使用ConfigMap和Secret
4. 展示如何实现配置的热更新

**代码实现**：
```yaml
ConfigMap（非敏感配置）：
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=8080
    database.url=postgres://localhost:5432/mydb
    cache.enabled=true
  log-level: "info"

Secret（敏感配置）：
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  database-password: cGFzc3dvcmQxMjM=
  api-key: YXBpLWtleS0xMjM0NTY3ODkw

使用ConfigMap和Secret：
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log-level
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: database-password
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: app-config
```

**常见错误分析**：
- 错误1：在Secret中存储明文密码
- 错误2：ConfigMap和Secret的value需要base64编码
- 错误3：不理解配置更新的传播机制
