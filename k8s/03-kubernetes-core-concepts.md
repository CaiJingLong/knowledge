# Kubernetes核心概念

Kubernetes系统中包含许多核心概念，这些概念构成了Kubernetes应用的基本构建块。理解这些概念对于有效地使用Kubernetes至关重要。本文将详细介绍这些核心概念及其用途。

## Pod

Pod是Kubernetes中最小的可部署计算单元，代表集群中运行的进程。

### Pod的特点

- Pod是Kubernetes对象模型中最小且最简单的单元
- 一个Pod代表集群上正在运行的一个进程
- Pod封装了一个或多个紧密相关的容器
- Pod中的容器共享存储、网络和运行规范
- Pod中的容器总是被同时调度，并在共享上下文中运行

### Pod示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

### Pod的生命周期

Pod的状态包括：

- **Pending**：Pod已被Kubernetes系统接受，但容器还没有被创建
- **Running**：Pod已经绑定到一个节点上，所有容器都已创建，至少有一个容器正在运行
- **Succeeded**：Pod中的所有容器都已成功终止，且不会重启
- **Failed**：Pod中的所有容器都已终止，且至少有一个容器以失败方式终止
- **Unknown**：由于某种原因无法获取Pod的状态

## ReplicaSet

ReplicaSet的目的是维护一组在任何时候都处于运行状态的Pod副本的稳定集合。

### ReplicaSet的特点

- 确保指定数量的Pod副本在任何时间都在运行
- 通常用于保证给定数量的、完全相同的Pod的可用性
- 现在通常通过Deployment来创建ReplicaSet，而不是直接创建

### ReplicaSet示例

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

## Deployment

Deployment为Pod和ReplicaSet提供声明式更新。

### Deployment的特点

- 描述应用的期望状态
- 控制器以受控的速率将实际状态更改为期望状态
- 提供声明式的应用更新方式
- 支持滚动更新和回滚功能
- 扩展和自动修复能力

### Deployment示例

```yaml
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Deployment更新策略

Deployment支持两种更新策略：

1. **RollingUpdate**（默认）：逐步替换旧Pod，确保服务不中断
2. **Recreate**：先删除所有旧Pod，再创建新Pod，会导致服务短暂中断

## Service

Service是一种抽象，定义了一组Pod的逻辑集合和访问这些Pod的策略。

### Service的特点

- 提供固定的IP地址和DNS名称
- 实现Pod之间的负载均衡
- 支持服务发现和路由
- 允许应用接收流量

### Service类型

- **ClusterIP**（默认）：在集群内部暴露服务，只能从集群内部访问
- **NodePort**：在每个节点的IP上暴露服务，通过`<NodeIP>:<NodePort>`可以从集群外部访问服务
- **LoadBalancer**：使用云提供商的负载均衡器向外部暴露服务
- **ExternalName**：将服务映射到externalName字段的内容，通过返回CNAME记录实现

### Service示例

```yaml
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
```

## ConfigMap与Secret

### ConfigMap

ConfigMap是一种API对象，用来将非机密性的配置数据保存到键值对中。

#### ConfigMap的特点

- 允许将配置信息与容器镜像解耦
- 可以通过环境变量、命令行参数或配置文件使用ConfigMap
- 支持修改配置而无需重建容器镜像

#### ConfigMap示例

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    app.name=MyApp
    app.version=1.0.0
  database.properties: |
    db.host=mysql
    db.port=3306
```

### Secret

Secret是一种包含少量敏感信息的对象，如密码、令牌或密钥。

#### Secret的特点

- 避免将敏感信息暴露在Pod规范或容器镜像中
- 可以通过环境变量、文件或作为容器启动命令的参数使用
- 默认情况下，Secret数据以Base64编码存储

#### Secret示例

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=  # base64编码的"admin"
  password: cGFzc3dvcmQxMjM=  # base64编码的"password123"
```

## Namespace

Namespace提供了一种在多个用户之间划分集群资源的方法。

### Namespace的特点

- 为名称提供作用域
- 将集群资源划分为多个逻辑单元
- 适用于多租户环境
- 允许不同团队或项目共享一个Kubernetes集群

### 默认Namespace

Kubernetes启动时会创建四个初始命名空间：

- **default**：默认命名空间，未指定时使用
- **kube-system**：Kubernetes系统创建的对象所在的命名空间
- **kube-public**：自动创建且所有用户可读的命名空间，主要用于集群使用
- **kube-node-lease**：用于与节点相关的租约对象

### Namespace示例

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

## Label与Selector

### Label

Label是附加到Kubernetes对象上的键值对，用于指定对用户有意义且相关的对象属性。

#### Label的特点

- 用于组织和选择对象子集
- 可以在创建时附加到对象，也可以在创建后添加和修改
- 每个对象可以定义多个标签，但每个键必须是唯一的

#### Label示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    environment: production
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx
```

### Selector

Selector允许客户端/用户识别一组对象。标签选择器是Kubernetes中的核心分组原语。

#### Selector类型

- **基于等式的选择器**：`=`，`==`，`!=`
- **基于集合的选择器**：`in`，`notin`，`exists`

#### Selector示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
```

## 总结

理解这些核心概念是使用Kubernetes的基础。这些概念相互关联，共同构成了Kubernetes的应用模型：

- **Pod**是最基本的部署单元，包含一个或多个容器
- **ReplicaSet**确保指定数量的Pod副本在运行
- **Deployment**提供声明式更新和管理ReplicaSet
- **Service**定义了一组Pod的访问方式
- **ConfigMap和Secret**管理配置信息和敏感数据
- **Namespace**提供了资源隔离的方式
- **Label和Selector**用于组织和选择对象

在下一篇文章中，我们将介绍如何搭建Kubernetes环境，包括Minikube、Kind、kubeadm等工具的使用方法。
