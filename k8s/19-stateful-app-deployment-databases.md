# Kubernetes有状态应用部署实战：数据库系统

在上一篇文章中，我们介绍了有状态应用的基础概念和StatefulSet的使用方法。本文将深入探讨如何在Kubernetes中部署和管理各类数据库系统，包括关系型数据库和NoSQL数据库。

## 目录
- [MySQL部署与配置](#mysql部署与配置)
- [PostgreSQL部署与配置](#postgresql部署与配置)
- [MongoDB部署与配置](#mongodb部署与配置)
- [数据库高可用架构](#数据库高可用架构)
- [数据库备份与恢复](#数据库备份与恢复)
- [最佳实践与注意事项](#最佳实践与注意事项)

## MySQL部署与配置

MySQL是最流行的关系型数据库之一，在Kubernetes中部署MySQL需要考虑数据持久化、配置管理和高可用性等方面。

### 单节点MySQL部署

首先，我们来看一个基本的MySQL单节点部署示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    default_authentication_plugin=mysql_native_password
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci
    
    # 性能优化配置
    innodb_buffer_pool_size=512M
    max_connections=1000
    
    # 日志配置
    slow_query_log=1
    slow_query_log_file=/var/log/mysql/mysql-slow.log
    long_query_time=2
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  # echo -n 'your-password' | base64
  root-password: eW91ci1wYXNzd29yZA==
  user-password: dXNlci1wYXNzd29yZA==
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    targetPort: 3306
    name: mysql
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 1
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
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_DATABASE
          value: "myapp"
        - name: MYSQL_USER
          value: "myuser"
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: user-password
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d/my.cnf
          subPath: my.cnf
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["mysql", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

这个YAML文件包含了以下几个部分：

1. **ConfigMap**：存储MySQL的配置文件，包括字符集、性能参数等
2. **Secret**：安全存储MySQL的密码信息
3. **Service**：为MySQL提供稳定的网络访问点
4. **StatefulSet**：管理MySQL Pod，确保数据持久化和稳定的网络标识

### MySQL主从复制配置

对于生产环境，通常需要配置MySQL的主从复制以提高可用性和读取性能。以下是一个MySQL主从复制的配置示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-master-config
data:
  my.cnf: |
    [mysqld]
    server-id=1
    log_bin=mysql-bin
    binlog_format=ROW
    binlog_do_db=myapp
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-slave-config
data:
  my.cnf: |
    [mysqld]
    server-id=2
    relay-log=mysql-relay-bin
    log_bin=mysql-bin
    binlog_format=ROW
    replicate_do_db=myapp
    read_only=1
```

然后，我们需要创建一个初始化脚本来配置主从复制：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-init-scripts
data:
  master-init.sh: |
    #!/bin/bash
    mysql -u root -p${MYSQL_ROOT_PASSWORD} << EOF
    CREATE USER 'repl'@'%' IDENTIFIED BY '${MYSQL_REPLICATION_PASSWORD}';
    GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
    FLUSH PRIVILEGES;
    EOF
    
  slave-init.sh: |
    #!/bin/bash
    until mysql -h mysql-master -u root -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1"; do
      echo "Waiting for mysql-master to be ready..."
      sleep 3
    done
    
    MASTER_STATUS=$(mysql -h mysql-master -u root -p${MYSQL_ROOT_PASSWORD} -e "SHOW MASTER STATUS" --skip-column-names)
    MASTER_LOG_FILE=$(echo $MASTER_STATUS | awk '{print $1}')
    MASTER_LOG_POS=$(echo $MASTER_STATUS | awk '{print $2}')
    
    mysql -u root -p${MYSQL_ROOT_PASSWORD} << EOF
    CHANGE MASTER TO
      MASTER_HOST='mysql-master',
      MASTER_USER='repl',
      MASTER_PASSWORD='${MYSQL_REPLICATION_PASSWORD}',
      MASTER_LOG_FILE='${MASTER_LOG_FILE}',
      MASTER_LOG_POS=${MASTER_LOG_POS};
    START SLAVE;
    EOF
```

最后，创建主从节点的StatefulSet和Service：

```yaml
# 主节点StatefulSet和Service配置
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
  labels:
    app: mysql
    role: master
spec:
  ports:
  - port: 3306
    targetPort: 3306
    name: mysql
  selector:
    app: mysql
    role: master
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-master
spec:
  selector:
    matchLabels:
      app: mysql
      role: master
  serviceName: mysql-master
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
        role: master
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        # ... 其他配置与前面类似
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: mysql-master-config
          mountPath: /etc/mysql/conf.d/my.cnf
          subPath: my.cnf
        - name: init-scripts
          mountPath: /docker-entrypoint-initdb.d/init.sh
          subPath: master-init.sh
      volumes:
      - name: mysql-master-config
        configMap:
          name: mysql-master-config
      - name: init-scripts
        configMap:
          name: mysql-init-scripts
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi

# 从节点StatefulSet和Service配置
# ... 类似主节点配置，但使用slave相关配置
```

## PostgreSQL部署与配置

PostgreSQL是一个功能强大的开源关系型数据库系统，在Kubernetes中部署PostgreSQL也需要考虑数据持久化和高可用性。

### 单节点PostgreSQL部署

以下是一个基本的PostgreSQL单节点部署示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    listen_addresses = '*'
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 1GB
    work_mem = 16MB
    maintenance_work_mem = 64MB
    
    # 日志配置
    log_statement = 'none'
    log_duration = off
    log_min_duration_statement = 1000  # 记录执行时间超过1秒的查询
    
  pg_hba.conf: |
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    host    all             all             0.0.0.0/0               md5
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  # echo -n 'postgres-password' | base64
  postgres-password: cG9zdGdyZXMtcGFzc3dvcmQ=
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    targetPort: 5432
    name: postgres
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  serviceName: postgres
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: POSTGRES_DB
          value: "myapp"
        - name: PGDATA
          value: "/var/lib/postgresql/data/pgdata"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        - name: postgres-config
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 5
          periodSeconds: 2
      volumes:
      - name: postgres-config
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

### PostgreSQL高可用配置

对于生产环境，可以使用Patroni或Crunchy Data的PostgreSQL Operator来实现PostgreSQL的高可用配置。以下是使用Patroni的简化示例：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: patroni
spec:
  selector:
    matchLabels:
      app: patroni
  serviceName: patroni
  replicas: 3
  template:
    metadata:
      labels:
        app: patroni
    spec:
      containers:
      - name: patroni
        image: patroni/patroni:latest
        env:
        - name: PATRONI_KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: PATRONI_KUBERNETES_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: PATRONI_KUBERNETES_LABELS
          value: '{app: patroni}'
        - name: PATRONI_SUPERUSER_USERNAME
          value: postgres
        - name: PATRONI_SUPERUSER_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: PATRONI_REPLICATION_USERNAME
          value: replicator
        - name: PATRONI_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: replication-password
        - name: PATRONI_SCOPE
          value: postgres-cluster
        ports:
        - containerPort: 8008
          name: patroni
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-data
          mountPath: /home/postgres/pgdata
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

## MongoDB部署与配置

MongoDB是一个流行的NoSQL数据库，在Kubernetes中部署MongoDB通常涉及到副本集或分片集群的配置。

### MongoDB副本集部署

以下是一个MongoDB副本集的部署示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config
data:
  mongod.conf: |
    storage:
      dbPath: /data/db
    net:
      bindIp: 0.0.0.0
    replication:
      replSetName: rs0
---
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  # echo -n 'mongo-password' | base64
  mongo-password: bW9uZ28tcGFzc3dvcmQ=
  mongo-root-password: bW9uZ28tcm9vdC1wYXNzd29yZA==
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    app: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
    name: mongo
  clusterIP: None
  selector:
    app: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  selector:
    matchLabels:
      app: mongo
  serviceName: mongo
  replicas: 3
  template:
    metadata:
      labels:
        app: mongo
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mongo
        image: mongo:4.4
        command:
        - mongod
        - --config=/etc/mongod.conf
        - --auth
        ports:
        - containerPort: 27017
          name: mongo
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
        - name: mongo-config
          mountPath: /etc/mongod.conf
          subPath: mongod.conf
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
      - name: mongo-sidecar
        image: cvallance/mongo-k8s-sidecar
        env:
        - name: MONGO_SIDECAR_POD_LABELS
          value: "app=mongo"
        - name: KUBERNETES_MONGO_SERVICE_NAME
          value: "mongo"
        - name: MONGODB_USERNAME
          value: "admin"
        - name: MONGODB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-root-password
        - name: MONGODB_DATABASE
          value: "admin"
      volumes:
      - name: mongo-config
        configMap:
          name: mongo-config
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

这个配置使用了一个sidecar容器来自动配置MongoDB副本集。

### MongoDB分片集群部署

对于需要更高性能和更大存储容量的应用，可以部署MongoDB分片集群。这通常包括配置服务器、分片服务器和mongos路由器。由于配置较为复杂，建议使用MongoDB Kubernetes Operator来简化部署过程。

## 数据库高可用架构

在Kubernetes中实现数据库高可用性需要考虑以下几个方面：

1. **多副本部署**：通过StatefulSet部署多个数据库副本
2. **自动故障转移**：使用数据库特定的机制或Operator实现自动故障转移
3. **数据同步**：确保数据在副本之间正确同步
4. **负载均衡**：将读请求分发到多个副本以提高性能
5. **备份和恢复**：定期备份数据并测试恢复流程

### 使用Operator简化数据库管理

Kubernetes Operator是一种扩展Kubernetes API的方式，可以自动化数据库的部署、配置和管理。以下是一些常用的数据库Operator：

1. **MySQL Operator**：如Oracle MySQL Operator或Presslabs MySQL Operator
2. **PostgreSQL Operator**：如Crunchy Data PostgreSQL Operator或Zalando Postgres Operator
3. **MongoDB Operator**：如MongoDB Enterprise Kubernetes Operator

以下是使用Zalando Postgres Operator部署PostgreSQL集群的示例：

```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: postgres-cluster
spec:
  teamId: "myteam"
  volume:
    size: 10Gi
  numberOfInstances: 3
  users:
    myuser:  # 数据库用户
      - superuser
      - createdb
  databases:
    myapp: myuser  # 数据库名称和所有者
  postgresql:
    version: "13"
    parameters:  # PostgreSQL配置参数
      shared_buffers: "256MB"
      max_connections: "100"
      log_statement: "all"
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

## 数据库备份与恢复

在Kubernetes中实现数据库备份和恢复通常有以下几种方法：

1. **使用数据库原生工具**：如MySQL的mysqldump、PostgreSQL的pg_dump等
2. **使用卷快照**：利用云提供商的卷快照功能
3. **使用备份工具**：如Velero、Stash等Kubernetes备份工具
4. **使用Operator内置的备份功能**：许多数据库Operator提供了备份和恢复功能

### 使用CronJob实现定期备份

以下是一个使用CronJob定期备份MySQL数据库的示例：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 2 * * *"  # 每天凌晨2点执行
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mysql-backup
            image: mysql:8.0
            command:
            - /bin/bash
            - -c
            - |
              BACKUP_FILE="/backup/mysql-$(date +%Y%m%d-%H%M%S).sql.gz"
              mysqldump -h mysql -u root -p${MYSQL_ROOT_PASSWORD} --all-databases | gzip > $BACKUP_FILE
              echo "Backup completed: $BACKUP_FILE"
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: mysql-backup-pvc
```

### 使用Velero进行应用级备份

Velero是一个用于备份和恢复Kubernetes集群资源和持久卷的工具。以下是使用Velero备份数据库的基本步骤：

1. 安装Velero
2. 创建备份前的钩子以确保数据一致性
3. 执行备份
4. 在需要时执行恢复

```bash
# 创建备份
velero backup create mysql-backup --include-namespaces=database-ns

# 恢复备份
velero restore create --from-backup mysql-backup
```

## 最佳实践与注意事项

在Kubernetes中部署和管理数据库时，应遵循以下最佳实践：

1. **资源隔离**：将数据库部署在专用的节点上，使用节点亲和性和污点/容忍来确保隔离
2. **合理的资源限制**：为数据库容器设置适当的CPU和内存请求与限制
3. **存储性能**：选择适合数据库工作负载的存储类，如SSD或高性能磁盘
4. **监控和告警**：设置全面的监控和告警系统，及时发现并解决问题
5. **定期备份**：实施定期备份策略并测试恢复流程
6. **安全性**：使用Secret存储敏感信息，配置网络策略限制访问

### 节点亲和性配置示例

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/database
            operator: Exists
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - mysql
        topologyKey: "kubernetes.io/hostname"
```

### 网络策略配置示例

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql-network-policy
spec:
  podSelector:
    matchLabels:
      app: mysql
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 3306
```

## 总结

在Kubernetes中部署和管理数据库系统是一项复杂但可行的任务。通过使用StatefulSet、PersistentVolume和适当的配置，可以在Kubernetes中运行生产级别的数据库服务。对于更复杂的场景，使用专门的数据库Operator可以大大简化管理工作。

在下一篇文章中，我们将探讨如何在Kubernetes中部署和管理消息队列和缓存系统，这些同样是有状态应用的重要组成部分。
