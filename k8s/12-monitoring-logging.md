# Kubernetes监控与日志

在Kubernetes环境中，监控和日志收集是保障系统稳定运行和快速排障的关键。本文将深入探讨Kubernetes中的监控与日志机制，包括Prometheus监控体系、Grafana可视化、EFK/ELK日志收集以及Kubernetes事件处理。

## Prometheus监控体系

Prometheus是一个开源的系统监控和告警工具包，非常适合Kubernetes环境。它采用拉取（pull）模式收集指标，并提供强大的查询语言PromQL。

### Prometheus架构

Prometheus的核心组件包括：

1. **Prometheus Server**：负责抓取和存储时间序列数据
2. **Client Libraries**：用于检测应用程序代码
3. **Pushgateway**：支持短期作业的指标推送
4. **Exporters**：为第三方系统提供指标
5. **Alertmanager**：处理告警

![Prometheus架构](https://prometheus.io/assets/architecture.png)

### 在Kubernetes中部署Prometheus

使用Helm是部署Prometheus的最简单方法：

```bash
# 添加Prometheus社区Helm仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# 更新仓库
helm repo update

# 安装Prometheus
helm install prometheus prometheus-community/prometheus
```

或者使用Prometheus Operator：

```bash
# 安装Prometheus Operator
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
```

### 配置Prometheus抓取Kubernetes指标

Prometheus可以通过Kubernetes的服务发现机制自动发现和抓取Pod、Service等资源的指标。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)

      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
```

### 使用ServiceMonitor

如果你使用Prometheus Operator，可以使用ServiceMonitor资源来配置监控目标：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
    interval: 15s
```

### 为应用程序添加Prometheus指标

要使你的应用程序能够被Prometheus监控，你需要在应用程序中暴露Prometheus格式的指标，并添加适当的注解：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
  labels:
    app: example-app
spec:
  containers:
  - name: example-app
    image: example-app:latest
    ports:
    - name: web
      containerPort: 8080
```

### 常用的Prometheus指标

Prometheus收集的常用Kubernetes指标包括：

- **Node指标**：CPU使用率、内存使用率、磁盘使用率、网络流量
- **Pod指标**：CPU使用率、内存使用率、网络流量
- **容器指标**：CPU使用率、内存使用率、文件系统使用率
- **API Server指标**：请求延迟、请求率、错误率
- **etcd指标**：操作延迟、操作率、领导者变更

### 使用PromQL查询指标

Prometheus提供了强大的查询语言PromQL，用于查询和处理时间序列数据。

```
# 查询节点CPU使用率
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance) / 
sum(rate(node_cpu_seconds_total[5m])) by (instance) * 100

# 查询Pod内存使用率
sum(container_memory_usage_bytes{pod=~"example-app-.*"}) by (pod) / 
sum(container_memory_max_usage_bytes{pod=~"example-app-.*"}) by (pod) * 100

# 查询HTTP请求率
sum(rate(http_requests_total[5m])) by (handler)

# 查询HTTP错误率
sum(rate(http_requests_total{status_code=~"5.."}[5m])) / 
sum(rate(http_requests_total[5m])) * 100
```

### 配置Prometheus告警

Prometheus支持基于PromQL表达式的告警规则：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-alert
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  groups:
  - name: example
    rules:
    - alert: HighCPUUsage
      expr: sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance) / sum(rate(node_cpu_seconds_total[5m])) by (instance) * 100 > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on {{ $labels.instance }}"
        description: "CPU usage is above 80% for 5 minutes."
```

## Grafana可视化

Grafana是一个开源的指标分析和可视化套件，非常适合与Prometheus结合使用，创建强大的监控仪表板。

### 在Kubernetes中部署Grafana

使用Helm部署Grafana：

```bash
# 添加Grafana Helm仓库
helm repo add grafana https://grafana.github.io/helm-charts

# 更新仓库
helm repo update

# 安装Grafana
helm install grafana grafana/grafana
```

### 配置Prometheus数据源

在Grafana中，你需要添加Prometheus作为数据源：

1. 登录Grafana
2. 点击"Configuration" > "Data Sources"
3. 点击"Add data source"
4. 选择"Prometheus"
5. 设置URL为Prometheus服务的地址（例如`http://prometheus-server:80`）
6. 点击"Save & Test"

### 导入Kubernetes仪表板

Grafana社区提供了许多预配置的Kubernetes仪表板，你可以直接导入使用：

1. 登录Grafana
2. 点击"+" > "Import"
3. 输入仪表板ID（例如3119、6417、8588）
4. 选择Prometheus数据源
5. 点击"Import"

### 创建自定义仪表板

你也可以创建自定义仪表板，以满足特定的监控需求：

1. 登录Grafana
2. 点击"+" > "Dashboard"
3. 点击"Add new panel"
4. 在"Query"选项卡中输入PromQL查询
5. 在"Visualization"选项卡中选择可视化类型
6. 配置面板标题、描述、单位等
7. 点击"Save"保存仪表板

### Grafana告警

Grafana也支持基于面板的告警：

1. 编辑面板
2. 点击"Alert"选项卡
3. 点击"Create Alert"
4. 配置告警条件、评估间隔、通知等
5. 点击"Save"保存告警

## EFK/ELK日志收集

EFK（Elasticsearch、Fluentd、Kibana）和ELK（Elasticsearch、Logstash、Kibana）是两种流行的日志收集和分析方案。

### EFK/ELK架构

- **Elasticsearch**：分布式搜索和分析引擎，用于存储和查询日志
- **Fluentd/Logstash**：日志收集器，负责收集、处理和转发日志
- **Kibana**：可视化平台，用于搜索、查看和分析日志

### 在Kubernetes中部署EFK

使用Helm部署EFK：

```bash
# 添加Elastic Helm仓库
helm repo add elastic https://helm.elastic.co

# 更新仓库
helm repo update

# 安装Elasticsearch
helm install elasticsearch elastic/elasticsearch

# 安装Kibana
helm install kibana elastic/kibana

# 安装Fluentd
kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch.yaml
```

### 配置Fluentd收集容器日志

Fluentd以DaemonSet形式部署，在每个节点上运行一个Pod，收集该节点上所有容器的日志：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.12.0-debian-elasticsearch7-1.0
        env:
          - name: FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch-client"
          - name: FLUENT_ELASTICSEARCH_PORT
            value: "9200"
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

### 自定义Fluentd配置

你可以通过ConfigMap自定义Fluentd的配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: kube-system
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    <filter kubernetes.**>
      @type kubernetes_metadata
    </filter>

    <match kubernetes.**>
      @type elasticsearch
      host elasticsearch-client
      port 9200
      logstash_format true
      logstash_prefix k8s
    </match>
```

### 在Kibana中查看日志

部署完EFK后，你可以在Kibana中查看和分析日志：

1. 访问Kibana UI
2. 创建索引模式（例如`logstash-*`或`k8s-*`）
3. 使用"Discover"页面搜索和过滤日志
4. 创建可视化和仪表板

### 日志查询示例

在Kibana中，你可以使用Elasticsearch的查询语言来搜索日志：

```
# 查询特定Pod的日志
kubernetes.pod_name: "example-app-7d8f675c7b-abcde"

# 查询特定命名空间的日志
kubernetes.namespace_name: "default"

# 查询包含特定文本的日志
message: "error"

# 查询特定时间范围的日志
@timestamp: [2023-01-01T00:00:00.000Z TO 2023-01-02T00:00:00.000Z]

# 组合查询
kubernetes.namespace_name: "default" AND message: "error" AND NOT message: "timeout"
```

## Kubernetes事件处理

Kubernetes事件（Events）提供了集群中发生的各种活动的记录，如Pod调度、容器启动、资源不足等。

### 查看Kubernetes事件

使用kubectl查看事件：

```bash
# 查看所有事件
kubectl get events

# 查看特定命名空间的事件
kubectl get events -n kube-system

# 查看特定Pod的事件
kubectl describe pod <pod-name>

# 按时间排序查看事件
kubectl get events --sort-by='.lastTimestamp'
```

### 事件类型

Kubernetes事件分为多种类型，包括：

- **Normal**：正常事件，如Pod创建成功
- **Warning**：警告事件，如Pod启动失败、资源不足

### 收集和持久化事件

默认情况下，Kubernetes事件只保留一小时。为了长期保存事件，你可以使用事件导出器：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-exporter
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: event-exporter
  template:
    metadata:
      labels:
        app: event-exporter
    spec:
      containers:
      - name: event-exporter
        image: ghcr.io/opsgenie/kubernetes-event-exporter:v0.11
        args:
        - -conf=/data/config.yaml
        volumeMounts:
        - name: config
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: event-exporter-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-config
  namespace: kube-system
data:
  config.yaml: |
    logLevel: info
    logFormat: json
    route:
      routes:
        - match:
            - receiver: elasticsearch
    receivers:
      - name: elasticsearch
        elasticsearch:
          hosts:
            - http://elasticsearch-client:9200
          index: kube-events
          indexFormat: kube-events-{2006.01.02}
```

### 基于事件的告警

你可以使用事件导出器将事件发送到告警系统，如Prometheus Alertmanager：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-config
  namespace: kube-system
data:
  config.yaml: |
    logLevel: info
    logFormat: json
    route:
      routes:
        - match:
            - type: "Warning"
          receiver: alertmanager
    receivers:
      - name: alertmanager
        webhook:
          endpoint: http://prometheus-alertmanager:9093/api/v1/alerts
          headers:
            Content-Type: application/json
          layout:
            group_key: "{{ .InvolvedObject.Kind }}.{{ .InvolvedObject.Namespace }}.{{ .InvolvedObject.Name }}"
            group: "{{ .Source.Component }}"
            alerts:
              - alert: "{{ .Reason }}"
                expr: "1"
                labels:
                  severity: warning
                  kind: "{{ .InvolvedObject.Kind }}"
                  namespace: "{{ .InvolvedObject.Namespace }}"
                  name: "{{ .InvolvedObject.Name }}"
                  reason: "{{ .Reason }}"
                annotations:
                  summary: "{{ .Message }}"
                  message: "{{ .Message }}"
```

## 监控与日志最佳实践

### 监控最佳实践

1. **定义明确的监控目标**：确定你需要监控什么，为什么需要监控，以及如何响应监控结果
2. **实施多层监控**：监控基础设施、Kubernetes组件、应用程序和业务指标
3. **设置适当的告警阈值**：避免过多的误报或漏报
4. **实施告警分级**：根据严重性和紧急性对告警进行分级
5. **自动化响应**：对常见问题实施自动化响应
6. **定期审查监控策略**：随着应用程序和基础设施的变化，定期审查和更新监控策略

### 日志最佳实践

1. **结构化日志**：使用JSON等结构化格式记录日志，便于解析和查询
2. **包含上下文信息**：在日志中包含足够的上下文信息，如请求ID、用户ID、操作类型等
3. **适当的日志级别**：使用适当的日志级别（DEBUG、INFO、WARN、ERROR），避免日志泛滥
4. **日志轮转**：实施日志轮转，避免日志文件过大
5. **集中式日志收集**：使用EFK/ELK等工具集中收集和分析日志
6. **日志保留策略**：定义日志保留策略，平衡存储成本和合规需求

### 监控与日志集成

1. **关联指标和日志**：通过请求ID等关联指标和日志，便于排障
2. **基于日志的指标**：从日志中提取指标，如错误率、请求延迟等
3. **基于指标的日志收集**：当指标超过阈值时，增加日志收集的详细程度

## 总结

本文深入探讨了Kubernetes中的监控与日志机制，包括Prometheus监控体系、Grafana可视化、EFK/ELK日志收集以及Kubernetes事件处理。这些工具和技术共同构成了Kubernetes的可观察性框架，使你能够全面了解集群和应用程序的状态，快速发现和解决问题。

在下一篇文章中，我们将介绍Kubernetes中的高可用与灾备，包括控制平面高可用配置、多集群管理、备份与恢复策略等内容。
