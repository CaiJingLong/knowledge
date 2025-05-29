# Kubernetes配置与安全

在Kubernetes中，配置管理和安全控制是确保应用程序正常运行和保护集群资源的关键方面。本文将深入探讨Kubernetes中的配置与安全机制，包括ConfigMap高级用法、Secret管理最佳实践、RBAC权限控制、NetworkPolicy网络策略、Pod安全策略以及ServiceAccount详解。

## ConfigMap高级用法

ConfigMap是Kubernetes中用于存储非敏感配置数据的资源对象。我们已经在前面的文章中介绍了ConfigMap的基本用法，现在让我们深入了解一些高级用法。

### 多种使用ConfigMap的方式

ConfigMap可以通过多种方式在Pod中使用：

#### 1. 作为环境变量

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
---
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_COLOR
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_MODE
```

你也可以一次性导入ConfigMap中的所有键值对作为环境变量：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    envFrom:
    - configMapRef:
        name: app-config
```

#### 2. 作为命令行参数

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    command: ["./app"]
    args: ["--color=$(APP_COLOR)", "--mode=$(APP_MODE)"]
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_COLOR
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_MODE
```

#### 3. 作为卷挂载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

当ConfigMap作为卷挂载时，每个键值对都会创建一个文件，键是文件名，值是文件内容。

### 动态更新ConfigMap

当ConfigMap作为卷挂载时，如果更新了ConfigMap，挂载的文件也会自动更新。这个过程可能需要几分钟时间。

```bash
# 更新ConfigMap
kubectl edit configmap app-config
```

然而，使用环境变量的方式不会自动更新，需要重启Pod才能获取新的配置值。

### 使用子路径挂载

如果你只想挂载ConfigMap中的特定键，可以使用`subPath`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config.json
      subPath: config.json
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: config.json
        path: config.json
```

### 不可变的ConfigMap

从Kubernetes 1.19开始，你可以创建不可变的ConfigMap，这可以提高性能并防止意外更新：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    immutable: "true"
data:
  APP_COLOR: blue
  APP_MODE: prod
```

## Secret管理最佳实践

Secret用于存储和管理敏感信息，如密码、OAuth令牌和SSH密钥。

### Secret类型

Kubernetes支持多种类型的Secret：

- **Opaque**：默认类型，任意用户定义的数据
- **kubernetes.io/service-account-token**：服务账户令牌
- **kubernetes.io/dockercfg**：序列化的~/.dockercfg文件
- **kubernetes.io/dockerconfigjson**：序列化的~/.docker/config.json文件
- **kubernetes.io/basic-auth**：基本认证凭证
- **kubernetes.io/ssh-auth**：SSH认证凭证
- **kubernetes.io/tls**：TLS认证凭证

### 创建Secret

#### 从命令行创建

```bash
# 从字面值创建
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=password123

# 从文件创建
kubectl create secret generic tls-secret --from-file=cert=path/to/cert.pem --from-file=key=path/to/key.pem
```

#### 使用YAML文件创建

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

注意：YAML文件中的数据必须是base64编码的。你可以使用以下命令进行编码：

```bash
echo -n 'admin' | base64
echo -n 'password123' | base64
```

### 使用Secret

与ConfigMap类似，Secret可以通过多种方式在Pod中使用：

#### 1. 作为环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

#### 2. 作为卷挂载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
```

### Secret管理最佳实践

#### 1. 限制Secret的访问权限

使用RBAC确保只有需要访问Secret的Pod和用户才能访问它们。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
  resourceNames: ["db-secret"]
```

#### 2. 加密Secret

默认情况下，Secret以未加密的形式存储在etcd中。为了增加安全性，应该启用etcd的加密功能。

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-key>
    - identity: {}
```

#### 3. 使用外部Secret管理系统

考虑使用外部Secret管理系统，如HashiCorp Vault、AWS Secrets Manager或Google Cloud Secret Manager，结合Kubernetes External Secrets或Secrets Store CSI Driver使用。

#### 4. 定期轮换Secret

定期更新Secret以减少凭证泄露的风险。

#### 5. 避免在版本控制系统中存储Secret

不要将Secret存储在Git等版本控制系统中，即使是以加密形式也不要。

## RBAC权限控制

基于角色的访问控制（RBAC）是Kubernetes中用于管理授权的机制。RBAC允许你配置谁可以对Kubernetes API执行哪些操作。

### RBAC的核心概念

RBAC API声明了四种Kubernetes对象：

- **Role**：在命名空间内定义权限
- **ClusterRole**：在集群范围内定义权限
- **RoleBinding**：将Role绑定到用户
- **ClusterRoleBinding**：将ClusterRole绑定到用户

### Role和ClusterRole

Role定义了在特定命名空间内可以执行的操作，而ClusterRole定义了集群范围内可以执行的操作。

```yaml
# Role示例
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

# ClusterRole示例
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### RoleBinding和ClusterRoleBinding

RoleBinding将Role或ClusterRole绑定到用户、组或服务账户，授予他们在命名空间内的权限。ClusterRoleBinding则在集群范围内授予权限。

```yaml
# RoleBinding示例
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

# ClusterRoleBinding示例
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 聚合ClusterRole

聚合ClusterRole允许你通过标签选择器组合多个ClusterRole。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # 规则将被自动填充

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["endpoints", "services", "pods"]
  verbs: ["get", "list", "watch"]
```

### 默认角色和角色绑定

Kubernetes自带一些默认的ClusterRole和ClusterRoleBinding：

- **cluster-admin**：超级用户权限
- **admin**：命名空间管理员权限
- **edit**：允许编辑大多数资源
- **view**：只读权限

### RBAC最佳实践

1. **最小权限原则**：只授予用户执行其工作所需的最小权限
2. **使用组**：将用户组织成组，然后为组分配角色
3. **避免使用cluster-admin**：除非绝对必要，否则避免使用cluster-admin角色
4. **定期审查权限**：定期检查和更新RBAC配置
5. **使用命名空间隔离**：使用命名空间和Role/RoleBinding来隔离不同的团队和应用

## NetworkPolicy网络策略

NetworkPolicy是一种规范，用于控制Pod组之间以及Pod与其他网络端点之间的通信。

### NetworkPolicy的工作原理

默认情况下，Kubernetes中的所有Pod可以相互通信，没有任何限制。NetworkPolicy允许你定义规则，限制Pod的入站和出站流量。

注意：要使用NetworkPolicy，你的Kubernetes网络插件必须支持NetworkPolicy。一些支持的网络插件包括Calico、Cilium、Kube-router、Romana和Weave Net。

### NetworkPolicy示例

#### 默认拒绝所有入站流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

#### 默认拒绝所有出站流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

#### 允许特定Pod之间的通信

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

#### 允许从特定命名空间访问

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
    ports:
    - protocol: TCP
      port: 8080
```

#### 允许访问外部服务

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-api
spec:
  podSelector:
    matchLabels:
      app: client
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

### NetworkPolicy最佳实践

1. **默认拒绝策略**：首先应用默认拒绝策略，然后添加允许特定流量的策略
2. **基于标签的选择器**：使用标签选择器来定义策略的目标
3. **命名空间隔离**：使用命名空间级别的策略来隔离不同的环境
4. **记录和监控**：记录和监控网络流量，以便识别和调试问题
5. **定期审查**：定期审查和更新网络策略，以确保它们仍然满足安全要求

## Pod安全策略(PSP)

Pod安全策略（PSP）是集群级别的资源，用于控制Pod规范中的安全敏感方面。

注意：从Kubernetes 1.21开始，PSP已被弃用，计划在未来的版本中移除。建议使用Pod安全准入控制器（Pod Security Admission）或第三方替代品，如OPA Gatekeeper或Kyverno。

### PSP控制的安全方面

PSP可以控制以下安全方面：

- 运行特权容器
- 使用主机命名空间
- 使用主机网络和端口
- 使用卷类型
- 使用主机文件系统
- 允许的FlexVolume驱动程序
- 分配拥有Pod卷的FSGroup
- 要求使用只读根文件系统
- 允许的用户和组ID
- 限制升级权限
- SELinux上下文
- AppArmor配置文件
- Seccomp配置文件
- 允许的sysctl配置文件

### PSP示例

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

### 使用RBAC授权PSP

要使PSP生效，你需要授权服务账户使用它：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-restricted-psp
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['restricted']

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-use-restricted-psp
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-restricted-psp
subjects:
- kind: ServiceAccount
  name: default
  namespace: default
```

## ServiceAccount详解

ServiceAccount提供了一种身份认证机制，用于Pod访问Kubernetes API服务器。

### ServiceAccount的工作原理

每个Pod都与一个ServiceAccount关联，Pod可以使用ServiceAccount的凭证访问API服务器。默认情况下，每个命名空间都有一个名为"default"的ServiceAccount。

### 创建ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
```

### 为Pod指定ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  containers:
  - name: my-container
    image: nginx
```

### 为ServiceAccount添加Secret

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
secrets:
- name: my-secret
```

### 为ServiceAccount分配权限

使用RBAC为ServiceAccount分配权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount令牌

从Kubernetes 1.24开始，ServiceAccount不再自动创建Secret。相反，Pod使用TokenRequest API获取绑定到Pod生命周期的令牌。

如果你需要创建长期令牌，可以手动创建Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-service-account-token
  annotations:
    kubernetes.io/service-account.name: my-service-account
type: kubernetes.io/service-account-token
```

### ServiceAccount最佳实践

1. **最小权限原则**：只授予ServiceAccount执行其工作所需的最小权限
2. **为每个应用创建专用ServiceAccount**：不要使用默认的ServiceAccount
3. **限制Token访问**：限制对ServiceAccount令牌的访问
4. **使用短期令牌**：使用TokenRequest API获取短期令牌，而不是长期令牌
5. **定期轮换令牌**：定期轮换长期令牌

## 总结

本文深入探讨了Kubernetes中的配置与安全机制，包括ConfigMap高级用法、Secret管理最佳实践、RBAC权限控制、NetworkPolicy网络策略、Pod安全策略以及ServiceAccount详解。这些机制共同构成了Kubernetes的安全框架，确保应用程序和集群的安全性。

在下一篇文章中，我们将介绍Kubernetes中的自动扩缩容，包括HPA、VPA、Cluster Autoscaler以及自定义指标扩缩容。
