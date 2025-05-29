# Kubernetes资源管理与调度

在Kubernetes中，资源管理和调度是确保应用程序高效运行的关键方面。本文将深入探讨Kubernetes中的资源管理与调度机制，包括资源请求与限制、QoS类别、节点亲和性与反亲和性、污点与容忍以及优先级与抢占。

## 资源请求与限制

Kubernetes允许你为容器指定所需的资源量（请求）和允许使用的最大资源量（限制）。这些设置对于资源分配和调度决策至关重要。

### 资源类型

Kubernetes支持两种主要的资源类型：

1. **CPU**：以CPU核心数量表示，可以是小数（如0.5表示半个CPU核心）
2. **内存**：以字节为单位，也可以使用Ki、Mi、Gi等后缀

### 资源请求(Requests)

资源请求定义了容器需要的最小资源量。Kubernetes保证容器能够获得请求的资源。调度器使用请求来决定将Pod放置在哪个节点上。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
```

在这个例子中，容器请求64MiB内存和0.25个CPU核心。

### 资源限制(Limits)

资源限制定义了容器可以使用的最大资源量。如果容器尝试使用超过限制的资源，它可能会被终止（对于内存）或被限制（对于CPU）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

在这个例子中，容器的内存使用限制为128MiB，CPU使用限制为0.5个核心。

### CPU限制的工作原理

CPU是一种可压缩资源，这意味着当容器尝试使用超过其限制的CPU时，它会被限制，而不是被终止。Kubernetes使用Linux内核的CFS（完全公平调度器）来实现CPU限制。

例如，如果一个容器的CPU限制为0.5，那么它在每100ms的周期内最多可以使用50ms的CPU时间。

### 内存限制的工作原理

内存是一种不可压缩资源，这意味着当容器尝试使用超过其限制的内存时，它可能会被OOM（内存不足）杀手终止。

当容器被OOM杀手终止时，它可能会根据Pod的`restartPolicy`重新启动。

### 资源单位

- **CPU**：1 CPU等于1个虚拟核心。可以使用小数（如0.5）或毫核（如500m，等于0.5）表示。
- **内存**：可以使用纯数字（表示字节）或带后缀的数字：
  - E, P, T, G, M, K（十进制：10^18, 10^15, 10^12, 10^9, 10^6, 10^3）
  - Ei, Pi, Ti, Gi, Mi, Ki（二进制：2^60, 2^50, 2^40, 2^30, 2^20, 2^10）

### 资源管理最佳实践

1. **始终设置资源请求**：这有助于调度器做出更好的决策
2. **基于实际使用情况设置限制**：监控应用程序的资源使用情况，并相应地设置限制
3. **考虑使用VPA**：Vertical Pod Autoscaler可以自动调整资源请求和限制
4. **避免过度承诺**：确保节点有足够的资源来满足所有Pod的请求
5. **为不同的环境使用不同的设置**：开发环境和生产环境的资源需求可能不同

## QoS类别

Kubernetes根据Pod的资源请求和限制将其分为三个QoS（服务质量）类别。这些类别决定了Pod在资源不足时的优先级。

### Guaranteed

当Pod中的所有容器都有相同的内存和CPU请求和限制时，Pod被分配到Guaranteed类别。这是最高的QoS级别，在资源不足时最不可能被终止。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "500m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Burstable

当Pod不符合Guaranteed的条件，但至少有一个容器有内存或CPU请求时，Pod被分配到Burstable类别。在资源不足时，Burstable Pod的优先级低于Guaranteed Pod。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### BestEffort

当Pod中的所有容器都没有内存和CPU请求和限制时，Pod被分配到BestEffort类别。这是最低的QoS级别，在资源不足时最可能被终止。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx
```

### QoS和OOM评分

当节点内存不足时，Kubernetes会根据Pod的QoS类别和内存使用情况计算OOM评分。OOM评分越高，Pod越可能被终止。

- BestEffort Pod的OOM评分最高
- Burstable Pod的OOM评分中等
- Guaranteed Pod的OOM评分最低

## 节点亲和性与反亲和性

节点亲和性和反亲和性允许你根据节点上的标签来约束Pod可以调度到哪些节点上。

### 节点亲和性(Node Affinity)

节点亲和性有两种类型：

1. **requiredDuringSchedulingIgnoredDuringExecution**：硬性要求，如果没有满足条件的节点，Pod将不会被调度
2. **preferredDuringSchedulingIgnoredDuringExecution**：软性偏好，调度器会尝试满足，但不保证

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: nginx
```

在这个例子中，Pod必须调度到具有标签`kubernetes.io/e2e-az-name=e2e-az1`或`kubernetes.io/e2e-az-name=e2e-az2`的节点上。如果可能，调度器会优先选择具有标签`another-node-label-key=another-node-label-value`的节点。

### 支持的操作符

节点亲和性支持以下操作符：

- `In`：标签值必须匹配其中一个指定的值
- `NotIn`：标签值不能匹配任何指定的值
- `Exists`：节点必须包含指定的标签（不检查值）
- `DoesNotExist`：节点不能包含指定的标签
- `Gt`：标签值必须大于指定的值
- `Lt`：标签值必须小于指定的值

### Pod亲和性和反亲和性(Pod Affinity and Anti-Affinity)

Pod亲和性和反亲和性允许你根据已经在节点上运行的Pod的标签来约束Pod可以调度到哪些节点上。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: nginx
```

在这个例子中，Pod必须调度到与具有标签`security=S1`的Pod在同一区域的节点上。如果可能，调度器会避免将Pod调度到与具有标签`security=S2`的Pod在同一区域的节点上。

### 拓扑键(Topology Key)

拓扑键是节点标签的键，用于定义"拓扑域"。常见的拓扑键包括：

- `kubernetes.io/hostname`：节点级别
- `topology.kubernetes.io/zone`：区域级别
- `topology.kubernetes.io/region`：区域级别

### 亲和性和反亲和性的用例

- **高可用性**：使用Pod反亲和性将应用程序的多个实例分布在不同的节点、区域或区域上
- **性能**：使用Pod亲和性将相互通信的应用程序放在同一节点上，减少网络延迟
- **硬件要求**：使用节点亲和性将需要特定硬件的应用程序调度到具有该硬件的节点上
- **合规性**：使用节点亲和性确保应用程序运行在符合特定合规要求的节点上

## 污点与容忍(Taints and Tolerations)

污点和容忍是一种机制，允许节点排斥一组Pod。污点应用于节点，而容忍应用于Pod。

### 污点(Taints)

污点由键、值和效果组成。可能的效果包括：

- **NoSchedule**：不允许调度不容忍该污点的Pod
- **PreferNoSchedule**：尽量避免调度不容忍该污点的Pod
- **NoExecute**：不允许不容忍该污点的Pod在节点上运行，如果已经运行，则会被驱逐

```bash
# 为节点添加污点
kubectl taint nodes node1 key=value:NoSchedule
```

### 容忍(Tolerations)

容忍允许Pod调度到具有匹配污点的节点上。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

在这个例子中，Pod可以调度到具有污点`key=value:NoSchedule`的节点上。

### 容忍操作符

容忍支持两种操作符：

- **Equal**：键、值和效果必须匹配
- **Exists**：键和效果必须匹配，不检查值

```yaml
# Exists操作符示例
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

### 污点和容忍的用例

- **专用节点**：为特定用户或工作负载保留节点
- **特殊硬件**：标记具有特殊硬件的节点，只允许需要该硬件的Pod运行
- **污点驱逐**：在节点问题出现时自动添加污点，驱逐不容忍该污点的Pod
- **节点维护**：在节点维护前添加污点，防止新Pod调度到该节点

### 内置污点

Kubernetes自动为节点添加一些内置污点：

- `node.kubernetes.io/not-ready`：节点未准备好
- `node.kubernetes.io/unreachable`：节点不可达
- `node.kubernetes.io/out-of-disk`：节点磁盘空间不足
- `node.kubernetes.io/memory-pressure`：节点内存压力
- `node.kubernetes.io/disk-pressure`：节点磁盘压力
- `node.kubernetes.io/network-unavailable`：节点网络不可用
- `node.kubernetes.io/unschedulable`：节点不可调度

## 优先级与抢占

优先级和抢占允许你定义Pod的相对重要性，并在资源不足时优先调度高优先级的Pod。

### Pod优先级

Pod优先级由PriorityClass定义，PriorityClass是一个集群级别的资源，定义了优先级的名称和值。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for high priority service pods only."
```

在这个例子中，我们定义了一个名为`high-priority`的PriorityClass，其优先级值为1000000。

### 为Pod分配优先级

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: high-priority
```

在这个例子中，Pod使用名为`high-priority`的PriorityClass。

### 抢占(Preemption)

当高优先级的Pod无法调度时，调度器可能会抢占（驱逐）低优先级的Pod，为高优先级的Pod腾出空间。

抢占过程如下：

1. 调度器尝试为高优先级的Pod找到一个节点
2. 如果没有合适的节点，调度器会寻找可以通过抢占低优先级Pod来容纳高优先级Pod的节点
3. 调度器会选择抢占影响最小的节点
4. 被抢占的Pod会被标记为终止，并被优雅地终止

### 优先级和抢占的用例

- **关键服务**：确保关键服务在资源紧张时优先获得资源
- **批处理作业**：为批处理作业分配低优先级，允许它们在资源充足时运行，但在资源紧张时让位给关键服务
- **开发和测试**：为开发和测试环境分配低优先级，确保生产环境优先获得资源

### 内置PriorityClass

Kubernetes包含两个内置的PriorityClass：

- `system-cluster-critical`：用于对集群至关重要的Pod
- `system-node-critical`：用于对节点至关重要的Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: system-cluster-critical
```

## 高级调度概念

### Descheduler

Descheduler是一个Kubernetes组件，用于根据特定策略重新平衡集群中的Pod分布。

```bash
# 安装Descheduler
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/descheduler/master/kubernetes/deployment.yaml
```

Descheduler支持多种策略，包括：

- **RemoveDuplicates**：移除重复的Pod
- **LowNodeUtilization**：将Pod从低利用率节点移到高利用率节点
- **HighNodeUtilization**：将Pod从高利用率节点移到低利用率节点
- **RemovePodsViolatingInterPodAntiAffinity**：移除违反Pod间反亲和性的Pod
- **RemovePodsViolatingNodeAffinity**：移除违反节点亲和性的Pod

### 自定义调度器

Kubernetes允许你部署自定义调度器，与默认调度器并行运行。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  schedulerName: my-custom-scheduler
  containers:
  - name: nginx
    image: nginx
```

在这个例子中，Pod将由名为`my-custom-scheduler`的自定义调度器调度，而不是默认调度器。

### 调度框架

调度框架是Kubernetes 1.15引入的一个可插拔架构，允许开发人员编写插件来影响调度决策。

调度框架定义了多个扩展点，包括：

- **PreFilter**：预过滤阶段，检查Pod是否可以调度到集群中
- **Filter**：过滤阶段，确定哪些节点适合Pod
- **PreScore**：预评分阶段，执行评分前的预处理
- **Score**：评分阶段，为每个节点分配分数
- **Reserve**：预留阶段，为Pod预留资源
- **Permit**：许可阶段，允许或拒绝调度决策
- **PreBind**：预绑定阶段，执行绑定前的预处理
- **Bind**：绑定阶段，将Pod绑定到节点
- **PostBind**：后绑定阶段，执行绑定后的清理

## 总结

本文深入探讨了Kubernetes中的资源管理与调度机制，包括资源请求与限制、QoS类别、节点亲和性与反亲和性、污点与容忍以及优先级与抢占。这些机制共同构成了Kubernetes强大的资源管理和调度系统，使应用程序能够高效地运行在集群中。

在下一篇文章中，我们将介绍Kubernetes中的监控与日志，包括Prometheus监控体系、Grafana可视化、EFK/ELK日志收集等内容。
