# Kubernetes常见问题诊断

在Kubernetes集群的日常运维中，故障排查是一项关键技能。无论是生产环境还是开发环境，都可能遇到各种问题，从Pod启动失败、网络连通性问题到资源不足和控制平面故障等。本文将详细介绍Kubernetes常见问题的诊断方法和解决策略，帮助读者快速定位和解决问题，提高集群的可靠性和稳定性。

## Pod启动失败排查

Pod是Kubernetes中最基本的部署单元，也是最常见的故障点。当Pod无法正常启动时，可以按照以下步骤进行排查。

### Pod状态分析

首先，需要了解Pod的当前状态：

```bash
kubectl get pod <pod-name> -n <namespace>
```

Pod可能处于以下几种状态：

1. **Pending**：Pod已创建但尚未调度到节点或正在下载镜像
2. **Running**：Pod已调度到节点并至少有一个容器正在运行
3. **Succeeded**：所有容器已成功终止
4. **Failed**：所有容器已终止，但至少有一个容器失败
5. **Unknown**：无法获取Pod状态，通常是节点通信问题

### 详细信息查看

获取Pod的详细信息，了解更多故障细节：

```bash
kubectl describe pod <pod-name> -n <namespace>
```

重点关注以下几个部分：
- **Events**：记录Pod生命周期中的重要事件
- **Conditions**：Pod当前的状态条件
- **Status**：容器的详细状态
- **Volumes**：存储卷的挂载状态

### 常见Pod问题及解决方法

#### 1. 镜像拉取失败(ImagePullBackOff)

**症状**：
- Pod状态为`Pending`或`ImagePullBackOff`
- Events中有类似`Failed to pull image`的错误

**可能原因**：
- 镜像名称或标签错误
- 私有仓库需要认证
- 网络连接问题
- 镜像不存在

**解决方法**：
- 检查镜像名称和标签是否正确
- 确保创建了正确的镜像拉取密钥(imagePullSecret)
- 检查节点网络连接
- 尝试手动拉取镜像验证

```bash
# 创建镜像拉取密钥
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email>

# 在Pod中使用密钥
spec:
  imagePullSecrets:
  - name: regcred
```

#### 2. 容器启动失败(CrashLoopBackOff)

**症状**：
- Pod状态为`CrashLoopBackOff`
- 容器反复重启

**可能原因**：
- 应用程序内部错误
- 配置错误
- 资源不足
- 健康检查失败

**解决方法**：
- 查看容器日志
```bash
kubectl logs <pod-name> -n <namespace>
# 如果是多容器Pod
kubectl logs <pod-name> -c <container-name> -n <namespace>
# 查看之前的容器日志
kubectl logs <pod-name> -p -n <namespace>
```

- 检查应用配置
- 调整资源限制
- 修改或禁用健康检查

#### 3. Pod卡在Pending状态

**症状**：
- Pod长时间处于`Pending`状态
- 没有被调度到任何节点

**可能原因**：
- 资源不足(CPU、内存等)
- 节点选择器(nodeSelector)或亲和性规则无法满足
- PersistentVolumeClaim未绑定
- 污点(Taint)阻止调度

**解决方法**：
- 检查调度器日志
```bash
kubectl describe pod <pod-name> -n <namespace>
# 查找Events中的调度信息
```

- 检查集群资源使用情况
```bash
kubectl get nodes
kubectl describe node <node-name>
```

- 调整资源请求或增加节点
- 修改节点选择器或亲和性规则
- 检查PVC状态
```bash
kubectl get pvc -n <namespace>
```

#### 4. Pod处于Running状态但应用不可用

**症状**：
- Pod状态为`Running`
- 应用程序无法访问或功能异常

**可能原因**：
- 应用内部错误
- 网络配置问题
- 健康检查配置不当
- 依赖服务不可用

**解决方法**：
- 检查应用日志
- 执行容器内命令诊断
```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
# 在容器内执行诊断命令
```

- 检查网络连接
```bash
kubectl exec -it <pod-name> -n <namespace> -- ping <service-name>
kubectl exec -it <pod-name> -n <namespace> -- curl <service-url>
```

- 验证健康检查端点

### Pod故障排查工具

除了基本的kubectl命令外，以下工具也有助于Pod故障排查：

1. **kubectl-debug**：在运行中的Pod旁边启动调试容器
```bash
kubectl debug <pod-name> -n <namespace> --image=nicolaka/netshoot
```

2. **stern**：同时查看多个Pod的日志
```bash
stern <pod-name-prefix> -n <namespace>
```

3. **kube-ps1**：在命令提示符中显示当前Kubernetes上下文
4. **kubectx/kubens**：快速切换集群和命名空间

## 网络连通性问题

Kubernetes网络问题通常较难排查，因为涉及多个层面：Pod网络、Service网络、节点网络和外部网络。

### Service连接问题

#### 1. Service无法访问

**症状**：
- 无法通过Service名称或ClusterIP访问Pod

**可能原因**：
- Service选择器与Pod标签不匹配
- Pod不健康
- 端口配置错误
- kube-proxy问题

**排查步骤**：

1. 检查Service定义
```bash
kubectl get service <service-name> -n <namespace> -o yaml
```

2. 验证Service是否选中了Pod
```bash
kubectl get endpoints <service-name> -n <namespace>
# 或使用
kubectl describe service <service-name> -n <namespace>
```

3. 检查Pod是否运行正常
```bash
kubectl get pods -l <service-selector> -n <namespace>
```

4. 直接测试Pod连接
```bash
# 获取Pod IP
kubectl get pod <pod-name> -n <namespace> -o wide
# 测试连接
kubectl exec -it <test-pod> -n <namespace> -- curl <pod-ip>:<port>
```

5. 检查kube-proxy日志
```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

#### 2. 外部无法访问Service

**症状**：
- 集群内可以访问Service
- 集群外无法访问NodePort或LoadBalancer类型的Service

**可能原因**：
- 防火墙规则阻止
- 节点网络配置问题
- 云提供商负载均衡器配置
- Service配置错误

**排查步骤**：

1. 检查Service类型和配置
```bash
kubectl get service <service-name> -n <namespace>
```

2. 验证节点端口可访问性
```bash
# 对于NodePort类型
curl <node-ip>:<node-port>
```

3. 检查防火墙规则
```bash
# 在节点上
iptables -L -n
```

4. 检查云提供商负载均衡器状态（对于LoadBalancer类型）

### Pod间网络问题

#### 1. Pod无法相互通信

**症状**：
- Pod之间无法建立网络连接

**可能原因**：
- 网络策略(NetworkPolicy)限制
- CNI插件问题
- 节点网络配置错误
- MTU不匹配

**排查步骤**：

1. 检查是否有NetworkPolicy限制
```bash
kubectl get networkpolicy -n <namespace>
```

2. 使用网络调试工具
```bash
# 在源Pod中
kubectl exec -it <source-pod> -n <namespace> -- ping <destination-pod-ip>
kubectl exec -it <source-pod> -n <namespace> -- traceroute <destination-pod-ip>
```

3. 检查CNI插件状态
```bash
kubectl get pods -n kube-system -l k8s-app=<cni-plugin>
kubectl logs -n kube-system -l k8s-app=<cni-plugin>
```

4. 验证节点间通信
```bash
# 在节点上
ping <other-node-ip>
```

### DNS问题

#### 1. DNS解析失败

**症状**：
- Pod无法解析服务名称
- `nslookup`或`dig`命令失败

**可能原因**：
- CoreDNS/kube-dns问题
- Pod DNS配置错误
- 网络策略限制DNS流量
- 集群DNS服务过载

**排查步骤**：

1. 检查CoreDNS/kube-dns状态
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

2. 在Pod中测试DNS
```bash
kubectl exec -it <pod-name> -n <namespace> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <pod-name> -n <namespace> -- cat /etc/resolv.conf
```

3. 检查DNS服务配置
```bash
kubectl get configmap coredns -n kube-system -o yaml
```

4. 验证DNS服务连接性
```bash
# 获取DNS服务IP
kubectl get service kube-dns -n kube-system
# 测试连接
kubectl exec -it <pod-name> -n <namespace> -- nc -zv <dns-service-ip> 53
```

### 网络故障排查工具

以下工具有助于网络故障排查：

1. **netshoot**：包含多种网络诊断工具的容器镜像
```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash
```

2. **ksniff**：捕获Pod网络流量
```bash
kubectl sniff <pod-name> -n <namespace>
```

3. **kubeshark**：Kubernetes网络流量分析工具
4. **Cilium Hubble**：对于使用Cilium的集群，提供网络流可视化

## 资源不足问题

资源不足是Kubernetes集群中常见的问题，可能导致Pod调度失败、性能下降或节点不稳定。

### 节点资源不足

#### 1. CPU或内存资源不足

**症状**：
- 新Pod无法调度
- 节点状态异常
- 系统Pod被驱逐

**可能原因**：
- 集群负载过高
- 资源请求配置不合理
- 节点故障
- 内存泄漏

**排查步骤**：

1. 检查节点资源使用情况
```bash
kubectl top nodes
kubectl describe node <node-name>
```

2. 检查Pod资源使用情况
```bash
kubectl top pods -A
```

3. 查看节点事件
```bash
kubectl get events --field-selector involvedObject.kind=Node
```

4. 检查系统组件状态
```bash
kubectl get pods -n kube-system
```

#### 2. 磁盘空间不足

**症状**：
- 节点状态为`NotReady`
- kubelet日志中有磁盘空间警告
- 容器无法创建

**可能原因**：
- 容器日志过多
- 镜像占用空间过大
- 临时存储卷填满
- 系统日志过多

**排查步骤**：

1. 检查节点磁盘使用情况
```bash
# 在节点上
df -h
```

2. 查看Docker/containerd存储使用情况
```bash
# 对于Docker
docker system df
# 对于containerd
du -sh /var/lib/containerd
```

3. 清理未使用的镜像和容器
```bash
# 对于Docker
docker system prune -a
# 对于containerd
crictl rmi --prune
```

4. 检查并轮转大型日志文件
```bash
find /var/log -type f -size +100M
```

### Pod资源限制问题

#### 1. Pod因OOM被终止

**症状**：
- Pod状态为`OOMKilled`
- 容器反复重启

**可能原因**：
- 内存限制设置过低
- 应用内存泄漏
- JVM堆大小配置不当
- 并发请求过多

**排查步骤**：

1. 检查Pod内存使用情况
```bash
kubectl top pod <pod-name> -n <namespace>
```

2. 查看Pod事件
```bash
kubectl describe pod <pod-name> -n <namespace>
```

3. 检查容器内存限制
```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

4. 调整内存限制
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

#### 2. Pod CPU节流

**症状**：
- 应用响应缓慢
- 处理时间增加
- CPU使用率达到限制

**可能原因**：
- CPU限制设置过低
- 工作负载突增
- 应用效率低下
- 后台任务占用CPU

**排查步骤**：

1. 检查Pod CPU使用情况
```bash
kubectl top pod <pod-name> -n <namespace>
```

2. 查看容器CPU限制
```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

3. 调整CPU限制
```yaml
resources:
  requests:
    cpu: "100m"
  limits:
    cpu: "500m"
```

4. 使用Vertical Pod Autoscaler分析适当的资源设置

### 资源配额问题

#### 1. 命名空间资源配额超限

**症状**：
- 无法创建新的资源
- 错误消息提示超出配额

**可能原因**：
- 命名空间配额设置过低
- 资源使用效率低下
- 未清理未使用的资源
- 配额计算错误

**排查步骤**：

1. 检查命名空间资源配额
```bash
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <quota-name> -n <namespace>
```

2. 查看命名空间资源使用情况
```bash
kubectl get pods -n <namespace>
kubectl get all -n <namespace>
```

3. 清理未使用的资源
```bash
kubectl delete pod,deployment,service,pvc -l <label> -n <namespace>
```

4. 调整资源配额
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
```

## 控制平面故障

控制平面组件的故障可能影响整个集群的功能，需要优先排查和解决。

### API Server问题

#### 1. API Server不可用

**症状**：
- `kubectl`命令返回连接错误
- 集群管理操作失败
- 新资源无法创建

**可能原因**：
- API Server进程崩溃
- etcd连接问题
- 证书过期
- 资源耗尽

**排查步骤**：

1. 检查API Server Pod状态
```bash
kubectl get pods -n kube-system -l component=kube-apiserver
# 如果kubectl不可用，直接在控制平面节点上检查
docker ps | grep kube-apiserver  # 对于Docker
crictl ps | grep kube-apiserver  # 对于containerd
```

2. 查看API Server日志
```bash
kubectl logs -n kube-system -l component=kube-apiserver
# 或在控制平面节点上
journalctl -u kube-apiserver
```

3. 检查etcd健康状态
```bash
kubectl get pods -n kube-system -l component=etcd
kubectl logs -n kube-system -l component=etcd
```

4. 验证证书有效性
```bash
# 在控制平面节点上
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 Validity
```

#### 2. API Server性能下降

**症状**：
- API请求延迟增加
- 超时错误
- 集群操作变慢

**可能原因**：
- 请求量过大
- etcd性能问题
- 资源限制不足
- 审计日志过多

**排查步骤**：

1. 检查API Server指标
```bash
# 如果启用了Prometheus
kubectl get --raw /metrics | grep apiserver_request
```

2. 查看API Server资源使用情况
```bash
kubectl top pods -n kube-system -l component=kube-apiserver
```

3. 检查etcd性能
```bash
kubectl exec -it -n kube-system etcd-<control-plane-node> -- etcdctl endpoint status --cluster -w table
```

4. 调整API Server配置
```bash
# 在控制平面节点上编辑kube-apiserver.yaml
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

### Scheduler和Controller Manager问题

#### 1. Scheduler不工作

**症状**：
- Pod卡在Pending状态
- 没有调度事件
- 新Pod不会被分配到节点

**可能原因**：
- Scheduler进程崩溃
- 配置错误
- 与API Server连接问题
- 锁定问题（在HA设置中）

**排查步骤**：

1. 检查Scheduler Pod状态
```bash
kubectl get pods -n kube-system -l component=kube-scheduler
```

2. 查看Scheduler日志
```bash
kubectl logs -n kube-system -l component=kube-scheduler
```

3. 检查Scheduler领导者选举状态（在HA设置中）
```bash
kubectl get endpoints kube-scheduler -n kube-system -o yaml
```

4. 重启Scheduler Pod
```bash
# 在控制平面节点上
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
# 等待几秒后
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
```

#### 2. Controller Manager问题

**症状**：
- 资源状态不更新
- ReplicaSet不创建Pod
- 服务账户不自动创建
- 节点状态不更新

**可能原因**：
- Controller Manager进程崩溃
- 配置错误
- 与API Server连接问题
- 锁定问题（在HA设置中）

**排查步骤**：

1. 检查Controller Manager Pod状态
```bash
kubectl get pods -n kube-system -l component=kube-controller-manager
```

2. 查看Controller Manager日志
```bash
kubectl logs -n kube-system -l component=kube-controller-manager
```

3. 检查Controller Manager领导者选举状态（在HA设置中）
```bash
kubectl get endpoints kube-controller-manager -n kube-system -o yaml
```

4. 重启Controller Manager Pod
```bash
# 在控制平面节点上
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
# 等待几秒后
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
```

### etcd问题

#### 1. etcd数据不一致

**症状**：
- API Server报错
- 资源状态不一致
- 集群操作失败

**可能原因**：
- etcd成员之间数据不同步
- 磁盘故障
- 网络分区
- 版本不兼容

**排查步骤**：

1. 检查etcd集群健康状态
```bash
kubectl exec -it -n kube-system etcd-<control-plane-node> -- etcdctl endpoint health --cluster
```

2. 检查etcd成员列表
```bash
kubectl exec -it -n kube-system etcd-<control-plane-node> -- etcdctl member list
```

3. 检查etcd日志
```bash
kubectl logs -n kube-system etcd-<control-plane-node>
```

4. 如有必要，从备份恢复etcd数据
```bash
# 停止API Server
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 恢复etcd数据
etcdctl snapshot restore <backup-file> --data-dir=/var/lib/etcd-restore

# 更新etcd数据目录
# 编辑etcd静态Pod配置
vim /etc/kubernetes/manifests/etcd.yaml
# 修改volumes部分指向新数据目录

# 重启API Server
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

#### 2. etcd性能问题

**症状**：
- API操作延迟高
- etcd CPU或内存使用率高
- 磁盘I/O高

**可能原因**：
- 数据库过大
- 磁盘性能不足
- 请求量过大
- 配置不当

**排查步骤**：

1. 检查etcd数据库大小
```bash
kubectl exec -it -n kube-system etcd-<control-plane-node> -- etcdctl endpoint status --write-out table
```

2. 监控etcd性能指标
```bash
kubectl exec -it -n kube-system etcd-<control-plane-node> -- etcdctl --write-out table endpoint status
```

3. 压缩etcd数据库
```bash
# 获取当前修订版本
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out="json" | jq '.[][].header.revision'

# 压缩到特定修订版本
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact <revision>
```

4. 碎片整理
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag
```

## 故障排查最佳实践

以下是Kubernetes故障排查的一些最佳实践，可以帮助更高效地解决问题：

### 系统化排查方法

1. **收集信息**：
   - 确定问题的确切症状
   - 记录错误消息和事件
   - 确定问题的影响范围
   - 了解最近的变更

2. **建立基线**：
   - 了解正常状态下的系统行为
   - 收集关键指标的基准值
   - 记录常见操作的预期结果

3. **分层排查**：
   - 从上到下或从下到上系统地检查各层
   - 应用层 → Pod层 → 节点层 → 集群层
   - 隔离问题所在的层级

4. **排除法**：
   - 排除已知正常的组件
   - 缩小问题范围
   - 验证假设

### 预防性措施

1. **监控和告警**：
   - 设置全面的监控系统
   - 配置关键指标的告警
   - 实施主动监控

2. **日志管理**：
   - 集中日志收集
   - 实施日志轮转策略
   - 保留足够的历史日志

3. **定期健康检查**：
   - 自动化集群健康检查
   - 定期审查资源使用情况
   - 测试关键功能

4. **文档和运行手册**：
   - 记录集群配置
   - 创建常见问题的排查指南
   - 维护变更日志

### 故障排查工具集

除了前面提到的工具外，以下是一些有用的故障排查工具：

1. **kubectl插件**：
   - `kubectl-neat`：清理YAML输出
   - `kubectl-tree`：显示资源层次结构
   - `kubectl-trace`：使用eBPF进行调试
   - `kubectl-df-pv`：显示PV使用情况

2. **集群诊断工具**：
   - `sonobuoy`：集群一致性测试
   - `popeye`：集群扫描工具
   - `kube-hunter`：安全漏洞扫描
   - `kube-bench`：CIS基准测试

3. **可观测性工具**：
   - Prometheus + Grafana：监控和可视化
   - Jaeger/Zipkin：分布式追踪
   - Fluentd/Loki：日志聚合
   - Kiali：服务网格可视化

4. **调试容器**：
   - `nicolaka/netshoot`：网络调试
   - `busybox`：基本命令行工具
   - `alpine`：轻量级调试环境
   - `curlimages/curl`：HTTP调试

## 总结

Kubernetes故障排查是一项复杂但必要的技能，掌握系统化的排查方法和常见问题的解决策略，可以大大提高问题解决的效率。本文介绍了Pod启动失败、网络连通性、资源不足和控制平面故障等常见问题的诊断和解决方法，以及故障排查的最佳实践和工具集。

通过遵循这些方法和实践，运维人员和开发者可以更快地定位和解决Kubernetes集群中的问题，提高集群的可靠性和稳定性。同时，建立预防性措施和自动化监控系统，可以帮助及早发现潜在问题，减少生产环境中的故障。

在下一篇文章中，我们将探讨Kubernetes性能优化，包括etcd性能调优、API Server优化、节点性能优化和网络性能调优等内容，帮助读者构建高性能的Kubernetes集群。
