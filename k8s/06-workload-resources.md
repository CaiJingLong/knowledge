# 工作负载资源管理

在Kubernetes中，工作负载资源是用于管理和运行容器化应用程序的对象。这些资源提供了不同的部署策略和管理方式，适用于各种应用场景。本文将深入探讨Kubernetes中的工作负载资源，包括Deployment、StatefulSet、DaemonSet、Job和CronJob等。

## Deployment高级配置

Deployment是最常用的工作负载资源，用于部署无状态应用。它提供了声明式更新和自动回滚功能，是部署应用程序的推荐方式。

### 滚动更新配置

Deployment支持两种更新策略：`RollingUpdate`（默认）和`Recreate`。滚动更新策略允许你控制更新的速率和方式。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多可以超出期望副本数的数量
      maxUnavailable: 1  # 最多允许不可用的Pod数量
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

### 暂停和恢复部署

在某些情况下，你可能希望暂停部署，进行多次更新，然后恢复部署以触发新的滚动更新。

```bash
# 暂停部署
kubectl rollout pause deployment/nginx-deployment

# 更新镜像
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

# 恢复部署
kubectl rollout resume deployment/nginx-deployment
```

### 部署状态检查

你可以使用以下命令检查部署的状态：

```bash
# 查看部署状态
kubectl rollout status deployment/nginx-deployment

# 查看部署历史
kubectl rollout history deployment/nginx-deployment
```

### 回滚部署

如果新版本出现问题，你可以回滚到之前的版本：

```bash
# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

# 回滚到特定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### 扩展部署

你可以轻松地扩展或缩减部署中的Pod数量：

```bash
kubectl scale deployment/nginx-deployment --replicas=10
```

### 金丝雀部署

金丝雀部署是一种发布策略，先将新版本发布给一小部分用户，然后逐步扩大范围。在Kubernetes中，可以通过创建两个Deployment来实现：

```yaml
# 稳定版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: nginx
      version: stable
  template:
    metadata:
      labels:
        app: nginx
        version: stable
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
---
# 金丝雀版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: canary
  template:
    metadata:
      labels:
        app: nginx
        version: canary
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
---
# 服务
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx  # 注意这里只选择app=nginx，不区分版本
  ports:
  - port: 80
    targetPort: 80
```

## StatefulSet应用

StatefulSet用于管理有状态应用，如数据库。它为每个Pod提供稳定的网络标识和持久存储。

### StatefulSet的特点

- 稳定、唯一的网络标识符
- 稳定、持久的存储
- 有序、优雅的部署和扩展
- 有序、优雅的删除和终止
- 有序的自动滚动更新

### StatefulSet示例

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
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
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### 更新策略

StatefulSet支持两种更新策略：

- **RollingUpdate**（默认）：按照Pod的序号顺序更新，从最大序号到最小序号
- **OnDelete**：只有在Pod被手动删除时才会创建新的Pod

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2  # 只更新序号大于或等于2的Pod
  # 其他配置...
```

### 分区更新

通过设置`partition`参数，你可以实现StatefulSet的分区更新，只更新序号大于或等于指定值的Pod。

### 扩展和缩减

StatefulSet的扩展和缩减是有序的，扩展时会按照序号从小到大创建Pod，缩减时会按照序号从大到小删除Pod。

```bash
kubectl scale statefulset web --replicas=5
```

## DaemonSet应用

DaemonSet确保所有（或部分）节点运行一个Pod的副本。当节点加入集群时，Pod会被添加到节点上；当节点从集群移除时，Pod会被回收。

### DaemonSet的用途

- 运行集群存储守护程序，如glusterd、ceph
- 运行日志收集守护程序，如fluentd、logstash
- 运行节点监控守护程序，如Prometheus Node Exporter、collectd

### DaemonSet示例

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

### 更新策略

DaemonSet支持两种更新策略：

- **RollingUpdate**（默认）：旧的DaemonSet Pod将被杀死，新的Pod将被创建
- **OnDelete**：只有在Pod被手动删除时才会创建新的Pod

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # 更新过程中最多有多少个Pod不可用
  # 其他配置...
```

### 节点选择

你可以使用节点选择器、亲和性或污点容忍来限制DaemonSet在哪些节点上运行：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  template:
    spec:
      nodeSelector:
        role: logging
      # 其他配置...
```

## Job与CronJob

Job创建一个或多个Pod，并确保指定数量的Pod成功完成。当Job创建的Pod成功完成时，Job将跟踪成功完成的数量。当达到指定的成功完成次数时，任务（即Job）结束。

### Job示例

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 5    # 需要成功完成的Pod数量
  parallelism: 2    # 并行运行的Pod数量
  backoffLimit: 4   # 重试次数
  activeDeadlineSeconds: 100  # 作业超时时间
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

### Job的完成模式

Job支持两种完成模式：

- **NonIndexed**（默认）：成功完成指定数量的Pod
- **Indexed**：为每个Pod分配一个完成索引，从0到completions-1

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo Job completion index: ${JOB_COMPLETION_INDEX}"]
      restartPolicy: Never
```

### CronJob

CronJob创建基于时间调度的Job。

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"  # 每分钟执行一次
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
  startingDeadlineSeconds: 10  # 错过调度的截止时间
  concurrencyPolicy: Allow     # 并发策略：Allow, Forbid, Replace
  successfulJobsHistoryLimit: 3  # 保留的成功Job数量
  failedJobsHistoryLimit: 1      # 保留的失败Job数量
```

### CronJob的并发策略

CronJob支持三种并发策略：

- **Allow**（默认）：允许同时运行多个Job
- **Forbid**：禁止并发运行，如果前一个Job还在运行，则跳过新的Job
- **Replace**：取消当前正在运行的Job，用新的Job替换它

## 滚动更新与回滚策略

Kubernetes提供了强大的滚动更新和回滚功能，使应用程序的更新和回滚变得简单和安全。

### 滚动更新策略

滚动更新是一种更新应用程序的方式，它逐步替换旧版本的Pod，同时保持服务的可用性。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多可以超出期望副本数的数量或百分比
      maxUnavailable: 1  # 最多允许不可用的Pod数量或百分比
  # 其他配置...
```

### 回滚策略

如果更新后的应用程序出现问题，Kubernetes允许你轻松回滚到之前的版本。

```bash
# 查看部署历史
kubectl rollout history deployment/nginx-deployment

# 查看特定版本的详细信息
kubectl rollout history deployment/nginx-deployment --revision=2

# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

# 回滚到特定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### 部署记录

为了能够回滚，Kubernetes会记录Deployment的历史版本。你可以通过设置`revisionHistoryLimit`来控制保留的历史版本数量。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  revisionHistoryLimit: 10  # 保留的历史版本数量
  # 其他配置...
```

## 总结

本文深入探讨了Kubernetes中的工作负载资源管理，包括Deployment、StatefulSet、DaemonSet、Job和CronJob的高级配置和应用场景。这些资源提供了不同的部署策略和管理方式，适用于各种应用场景。理解这些概念对于有效地使用Kubernetes至关重要。

在下一篇文章中，我们将介绍服务发现与负载均衡，包括Service类型详解、Ingress资源与控制器等内容。
