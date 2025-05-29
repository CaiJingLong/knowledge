# Kubernetes无状态应用部署实战

无状态应用是指不需要持久化数据的应用程序，每个请求都是独立的，不依赖于之前的请求。这类应用非常适合在Kubernetes中部署，因为它们可以轻松扩展和迁移。本文将深入探讨Kubernetes中的无状态应用部署实战，包括Web应用部署案例、前后端分离应用部署、自动扩缩容配置以及蓝绿部署与金丝雀发布。

## Web应用部署案例

让我们从一个简单的Web应用部署开始，逐步了解Kubernetes中无状态应用的部署过程。

### 部署简单的Nginx应用

Nginx是一个流行的Web服务器，我们将其作为第一个示例。

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
# nginx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: nginx.example.com
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
# 应用配置
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-ingress.yaml

# 验证部署
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get ingress
```

### 部署带有配置的Web应用

大多数Web应用需要配置文件，我们可以使用ConfigMap来管理这些配置。

```yaml
# nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;

      location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
      }

      location /api {
        proxy_pass http://api-service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }
---
# nginx-deployment-with-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
```

```bash
# 应用配置
kubectl apply -f nginx-config.yaml
kubectl apply -f nginx-deployment-with-config.yaml
```

### 部署带有健康检查的Web应用

健康检查是确保应用程序正常运行的重要机制。Kubernetes支持两种类型的健康检查：存活探针（Liveness Probe）和就绪探针（Readiness Probe）。

```yaml
# nginx-deployment-with-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

```bash
# 应用配置
kubectl apply -f nginx-deployment-with-probes.yaml
```

### 部署带有资源限制的Web应用

为应用程序设置适当的资源限制是良好实践，可以防止单个应用消耗过多资源。

```yaml
# nginx-deployment-with-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

```bash
# 应用配置
kubectl apply -f nginx-deployment-with-resources.yaml
```

## 前后端分离应用部署

现代Web应用通常采用前后端分离架构，前端负责用户界面，后端提供API服务。

### 部署前端应用

前端应用通常是静态文件，可以使用Nginx或其他Web服务器托管。

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: my-org/frontend:1.0.0
        ports:
        - containerPort: 80
---
# frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
# frontend-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### 部署后端API服务

后端API服务通常是处理业务逻辑的应用程序。

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: my-org/backend:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "db-service"
        - name: DB_PORT
          value: "5432"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
---
# backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# backend-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
spec:
  rules:
  - host: api.myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

### 配置前后端通信

前后端通信可以通过API网关或直接通过服务名进行。

#### 使用API网关

```yaml
# api-gateway.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /()(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

#### 使用服务名直接通信

前端应用可以配置为使用后端服务的内部DNS名称。

```javascript
// frontend/src/config.js
export const API_URL = process.env.NODE_ENV === 'production'
  ? 'http://backend-service'
  : 'http://localhost:8080';
```

### 使用Helm部署前后端应用

Helm是Kubernetes的包管理器，可以简化应用部署。

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: A Helm chart for MyApp
type: application
version: 0.1.0
appVersion: 1.0.0

# values.yaml
frontend:
  replicaCount: 3
  image:
    repository: my-org/frontend
    tag: 1.0.0
  service:
    type: ClusterIP
    port: 80

backend:
  replicaCount: 3
  image:
    repository: my-org/backend
    tag: 1.0.0
  service:
    type: ClusterIP
    port: 80
  env:
    DB_HOST: db-service
    DB_PORT: "5432"

ingress:
  enabled: true
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          service: frontend
        - path: /api
          service: backend
```

```bash
# 安装Helm图表
helm install myapp ./myapp
```

## 自动扩缩容配置

Kubernetes提供了多种自动扩缩容机制，可以根据负载自动调整应用程序的实例数量。

### 基于CPU和内存的水平自动扩缩容

Horizontal Pod Autoscaler (HPA) 可以根据CPU和内存使用率自动调整Pod数量。

```yaml
# frontend-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
# 应用HPA配置
kubectl apply -f frontend-hpa.yaml

# 查看HPA状态
kubectl get hpa
```

### 基于自定义指标的自动扩缩容

除了CPU和内存外，HPA还支持基于自定义指标进行扩缩容。

```yaml
# backend-hpa-custom.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 10
```

要使用自定义指标，需要部署指标适配器，如Prometheus Adapter。

```yaml
# prometheus-adapter-values.yaml
rules:
  default: false
  custom:
  - seriesQuery: 'http_requests_total{kubernetes_namespace!="",kubernetes_pod_name!=""}'
    resources:
      overrides:
        kubernetes_namespace: {resource: "namespace"}
        kubernetes_pod_name: {resource: "pod"}
    name:
      matches: "^(.*)_total$"
      as: "${1}_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

```bash
# 安装Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter -f prometheus-adapter-values.yaml
```

### 垂直自动扩缩容

Vertical Pod Autoscaler (VPA) 可以自动调整Pod的CPU和内存请求。

```yaml
# frontend-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: frontend-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: frontend
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 500m
        memory: 512Mi
      controlledResources: ["cpu", "memory"]
```

```bash
# 安装VPA
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vpa-v1-crd.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vpa-rbac.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.10.0/vpa-v1-crd-gen.yaml

# 应用VPA配置
kubectl apply -f frontend-vpa.yaml
```

### 集群自动扩缩容

Cluster Autoscaler可以根据Pod的资源需求自动调整集群节点数量。

```yaml
# cluster-autoscaler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --nodes=3:10:my-node-group-1
        - --nodes=3:10:my-node-group-2
```

```bash
# 部署Cluster Autoscaler
kubectl apply -f cluster-autoscaler-deployment.yaml
```

## 蓝绿部署与金丝雀发布

蓝绿部署和金丝雀发布是两种常用的部署策略，可以减少部署风险。

### 蓝绿部署

蓝绿部署通过同时维护两个相同的环境（蓝色和绿色）来实现零停机部署。

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: my-org/myapp:1.0.0
        ports:
        - containerPort: 8080
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 0  # 初始设置为0
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: my-org/myapp:1.1.0
        ports:
        - containerPort: 8080
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # 初始指向蓝色版本
  ports:
  - port: 80
    targetPort: 8080
```

蓝绿部署过程：

1. 部署蓝色版本（当前生产版本）
2. 部署绿色版本（新版本）
3. 测试绿色版本
4. 将流量从蓝色版本切换到绿色版本
5. 如果出现问题，可以快速回滚到蓝色版本

```bash
# 部署蓝色版本
kubectl apply -f blue-deployment.yaml
kubectl apply -f service.yaml

# 部署绿色版本
kubectl apply -f green-deployment.yaml
kubectl scale deployment myapp-green --replicas=3

# 测试绿色版本
# ...

# 切换流量到绿色版本
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'

# 如果需要回滚
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

### 金丝雀发布

金丝雀发布是一种渐进式发布策略，先将新版本部署到一小部分用户，然后逐步扩大范围。

```yaml
# stable-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
  labels:
    app: myapp
    version: stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: myapp
        image: my-org/myapp:1.0.0
        ports:
        - containerPort: 8080
---
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
  labels:
    app: myapp
    version: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: my-org/myapp:1.1.0
        ports:
        - containerPort: 8080
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp  # 注意这里只选择app标签，不区分版本
  ports:
  - port: 80
    targetPort: 8080
```

金丝雀发布过程：

1. 部署稳定版本（当前生产版本）
2. 部署金丝雀版本（新版本），初始副本数较少
3. 监控金丝雀版本的性能和错误率
4. 如果金丝雀版本表现良好，逐步增加其副本数，同时减少稳定版本的副本数
5. 最终完全替换稳定版本

```bash
# 部署稳定版本
kubectl apply -f stable-deployment.yaml
kubectl apply -f service.yaml

# 部署金丝雀版本
kubectl apply -f canary-deployment.yaml

# 监控金丝雀版本
# ...

# 逐步增加金丝雀版本的比例
kubectl scale deployment myapp-canary --replicas=3
kubectl scale deployment myapp-stable --replicas=7

kubectl scale deployment myapp-canary --replicas=5
kubectl scale deployment myapp-stable --replicas=5

kubectl scale deployment myapp-canary --replicas=10
kubectl scale deployment myapp-stable --replicas=0

# 完成迁移后，可以删除稳定版本的部署
kubectl delete deployment myapp-stable
```

### 使用Istio进行流量分割

Istio服务网格提供了更精细的流量控制能力，可以基于权重、请求头等进行流量分割。

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - myapp.example.com
  http:
  - route:
    - destination:
        host: myapp-service
        subset: stable
      weight: 90
    - destination:
        host: myapp-service
        subset: canary
      weight: 10
---
# destination-rule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp-dr
spec:
  host: myapp-service
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
```

```bash
# 应用Istio配置
kubectl apply -f virtual-service.yaml
kubectl apply -f destination-rule.yaml

# 逐步调整流量比例
kubectl patch virtualservice myapp-vs --type=json -p='[{"op": "replace", "path": "/spec/http/0/route/0/weight", "value": 80}, {"op": "replace", "path": "/spec/http/0/route/1/weight", "value": 20}]'

kubectl patch virtualservice myapp-vs --type=json -p='[{"op": "replace", "path": "/spec/http/0/route/0/weight", "value": 50}, {"op": "replace", "path": "/spec/http/0/route/1/weight", "value": 50}]'

kubectl patch virtualservice myapp-vs --type=json -p='[{"op": "replace", "path": "/spec/http/0/route/0/weight", "value": 0}, {"op": "replace", "path": "/spec/http/0/route/1/weight", "value": 100}]'
```

### 使用Argo Rollouts进行渐进式交付

Argo Rollouts是一个Kubernetes控制器，提供了高级部署功能，如蓝绿部署、金丝雀发布、实验分析等。

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
spec:
  replicas: 10
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
        image: my-org/myapp:1.0.0
        ports:
        - containerPort: 8080
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 1h}
      - setWeight: 20
      - pause: {duration: 1h}
      - setWeight: 50
      - pause: {duration: 1h}
      - setWeight: 100
```

```bash
# 安装Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 部署Rollout
kubectl apply -f rollout.yaml

# 更新应用
kubectl patch rollout myapp-rollout --type=json -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "my-org/myapp:1.1.0"}]'

# 查看Rollout状态
kubectl argo rollouts get rollout myapp-rollout
kubectl argo rollouts dashboard
```

## 总结

本文深入探讨了Kubernetes中的无状态应用部署实战，包括Web应用部署案例、前后端分离应用部署、自动扩缩容配置以及蓝绿部署与金丝雀发布。通过这些实践，可以构建高可用、可扩展的无状态应用，并实现安全、可靠的部署过程。

无状态应用是Kubernetes中最常见的应用类型，掌握其部署和管理技巧对于云原生应用开发至关重要。随着技术的不断发展，Kubernetes生态系统也在不断丰富，提供了越来越多的工具和方法来简化无状态应用的部署和管理。

在下一篇文章中，我们将介绍Kubernetes中的有状态应用部署实战，包括数据库、消息队列、缓存等有状态应用的部署和管理。
