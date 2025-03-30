---
title: K8S 部署 Nacos
date: 2025-02-23 18:40:00
tags: 
  - K8S
  - Nacos
categories: 运维部署
---
## 配置NFS共享目录

1. 安装NFS服务器(mysql文档中已安装)
2. 在NFS服务器创建目录
   ```bash
    sudo mkdir -p /mnt/nfs/nacos-{0,1,2}  # 一次性创建3个目录
    sudo chmod 777 /mnt/nfs/nacos-{0,1,2}  # 设置目录权限（根据实际安全要求调整）
   ```
3. 配置NFS导出目录
   编辑NFS配置文件 `/etc/exports`，添加以下内容：
   ```bash
    /mnt/nfs/nacos-0 *(rw,sync,no_subtree_check,no_root_squash)
    /mnt/nfs/nacos-1 *(rw,sync,no_subtree_check,no_root_squash)
    /mnt/nfs/nacos-2 *(rw,sync,no_subtree_check,no_root_squash)  
   ```
4. 应用NFS配置
   ```bash
    sudo exportfs -arv  # 重新导出所有目录
    sudo systemctl restart nfs-server  # 重启NFS服务（部分系统可能需要重启nfs-kernel-server）
   ```
5. 在Kubernetes节点测试挂载
   在任意节点执行以下命令，验证NFS目录可访问：
    ```bash
    mkdir /mnt/test  # 创建测试目录
    mount -t nfs <NFS_IP>:/mnt/nfs/nacos-0 /mnt/test  # 挂载NFS目录
    umount /mnt/test  # 卸载测试目录
    ```

## 准备数据库

1. 创建数据库及用户
   ```sql
    -- 创建数据库
    CREATE DATABASE nacos_cluster DEFAULT CHARACTER SET utf8mb4;

    -- 创建用户并授权
    CREATE USER 'nacos'@'%' IDENTIFIED BY 'Nacos@123';
    GRANT ALL PRIVILEGES ON nacos_cluster.* TO 'nacos'@'%';
    FLUSH PRIVILEGES;
   ```
2. 初始化 Nacos 数据库表
   执行官方 SQL 脚本
   ```sql
    # 下载 SQL 脚本
    # 注意: 使用的脚本要和安装的 Nacos 版本匹配, 不同的版本可能表名或字段名不同, 将下面的version换成对应版本
    wget https://raw.githubusercontent.com/alibaba/nacos/<version>/distribution/conf/mysql-schema.sql

    # 导入数据库
    mysql -h <MySQL_IP> -u nacos -pNacos@123 nacos_cluster < mysql-schema.sql
   ```
## 部署 Nacos 集群
### 1. 创建 ConfigMap 存储配置
```yaml
# nacos-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nacos-config
data:
  application.properties: |
    server.servlet.contextPath=/nacos
    spring.datasource.platform=mysql
    db.num=1
    db.url.0=jdbc:mysql://mysql-service:3306/nacos_cluster?useUnicode=true&characterEncoding=utf-8
    db.user=nacos
    db.password=Nacos@123
    
    nacos.core.auth.enabled=true
    nacos.core.auth.server.identity.key=authKey
    nacos.core.auth.server.identity.value=xXvY5baMTt5rXZiy4xdntoqmtuTYchzh
    nacos.core.auth.system.type=nacos
    nacos.core.auth.plugin.nacos.token.secret.key=VUpwYzNKbGJXVWdhMlY1Y0dWamFHNXlZMmg1YUhvPQo=
    nacos.core.auth.plugin.nacos.token.expire.seconds=604800
    nacos.core.auth.enable.userAgentAuthWhite=false
    nacos.core.auth.caching.enabled=true
    nacos.core.auth.server.identity.scheme=auth
    nacos.core.auth.plugin.nacos.token.security.trusted-urls=0.0.0.0
```
### 2. Secret 存储密钥
```yaml
# nacos-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nacos-secret
type: Opaque
data:
  # echo -n "xXvY5baMTt5rXZiy4xdntoqmtuTYchzh" | base64
  authKey: eFh2WTViYU1UdDVyWFppeTR4ZG50b3FtdHVUWWNoeHo=
  tokenKey: VUpwYzNKbGJXVWdhMlY1Y0dWamFHNXlZMmg1YUhvPQo=
```

### 3. 创建持久化存储（PV/PVC）
```yaml
# nacos-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nacos-pv-0
  labels:  # 新增标签
    type: nfs-pv
    app: nacos
    pv-index: "0"
spec:
  capacity:
    storage: 2Gi
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /mnt/nfs/nacos-0  # 每个节点独立目录
    server: 192.168.3.130
  claimRef:
    namespace: default
    name: nacos-data-nacos-cluster-0
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nacos-pv-1
  labels:  # 新增标签
    type: nfs-pv
    app: nacos
    pv-index: "1"
spec:
  capacity:
    storage: 2Gi
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /mnt/nfs/nacos-1
    server: 192.168.3.130
  claimRef:
    namespace: default
    name: nacos-data-nacos-cluster-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nacos-pv-2
  labels:  # 新增标签
    type: nfs-pv
    app: nacos
    pv-index: "2"
spec:
  capacity:
    storage: 2Gi
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /mnt/nfs/nacos-2
    server: 192.168.3.130
  claimRef:
    namespace: default
    name: nacos-data-nacos-cluster-2
```

### 4. 部署 Nacos StatefulSet
注意版本号要与SQL脚本的版本号一致
```yaml
# nacos-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos-cluster
spec:
  serviceName: nacos-headless
  replicas: 3  # 集群节点数
  selector:
    matchLabels:
      app: nacos
  template:
    metadata:
      labels:
        app: nacos
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: ["nacos"]
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nacos
          image: nacos/nacos-server:v2.5.1
          env:
            - name: MODE
              value: cluster
            - name: NACOS_SERVERS
              value: "nacos-0.nacos-headless:8848 nacos-1.nacos-headless:8848 nacos-2.nacos-headless:8848"
            - name: NACOS_AUTH_ENABLE
              value: "true"  # 显式开启鉴权
            - name: NACOS_AUTH_IDENTITY_KEY       # 身份认证密钥名称
              value: authKey
            - name: NACOS_AUTH_IDENTITY_VALUE     # 身份认证密钥值（需与Secret一致）
              valueFrom:
                secretKeyRef:
                  name: nacos-secret
                  key: authKey
            - name: NACOS_AUTH_TOKEN_EXPIRE_SECONDS  # Token过期时间（秒）
              value: "604800"
            - name: NACOS_AUTH_TOKEN_SECRET_KEY      # JWT密钥（需与Secret一致）
              valueFrom:
                secretKeyRef:
                  name: nacos-secret
                  key: tokenKey
            - name: NACOS_AUTH_CACHING_ENABLED      # 开启缓存
              value: "true"
            - name: NACOS_AUTH_TOKEN_SECURITY_TRUSTED_URLS  # 信任的URL
              value: "0.0.0.0"
            - name: SPRING_DATASOURCE_PLATFORM
              value: mysql
            - name: MYSQL_SERVICE_HOST
              value: "mysql-service"
            - name: MYSQL_SERVICE_DB_NAME
              value: "nacos_cluster"
            - name: MYSQL_SERVICE_USER
              value: "nacos"
            - name: MYSQL_SERVICE_PASSWORD
              value: "Nacos@123"
            - name: NACOS_APPLICATION_PORT
              value: "8848"
          ports:
            - containerPort: 8848
              name: http
            - containerPort: 7848
              name: rpc
            - containerPort: 9848
              name: grpc
          volumeMounts:
            - name: nacos-data
              mountPath: /home/nacos/data
            - name: nacos-config
              mountPath: /home/nacos/conf/application.properties
              subPath: application.properties
      volumes:
        - name: nacos-config
          configMap:
            name: nacos-config
  volumeClaimTemplates:
    - metadata:
        name: nacos-data
      spec:
        accessModes: ["ReadWriteMany"]
        resources:
          requests:
            storage: 2Gi
        storageClassName: nfs
```
### 5. 部署 Nacos Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nacos-headless
spec:
  clusterIP: None
  ports:
    - name: http
      port: 8848
    - name: rpc
      port: 7848
    - name: grpc
      port: 9848
  selector:
    app: nacos
---
apiVersion: v1
kind: Service
metadata:
  name: nacos-nodeport
spec:
  type: NodePort
  ports:
    - name: http
      port: 8848
      targetPort: 8848
      nodePort: 30002
  selector:
    app: nacos
```

### 6. 应用配置
```bash
kubectl apply -f nacos-pv.yaml
kubectl apply -f nacos-configmap.yaml
kubectl apply -f nacos-secret.yaml
kubectl apply -f nacos-services.yaml
kubectl apply -f nacos-statefulset.yaml
```
### 7. 验证部署
```bash
kubectl get statefulset,pod,pv,pvc -l app=nacos
```
预期输出:
```
NAME                        READY   AGE
statefulset/nacos-cluster   3/3     5m

NAME                READY   STATUS    RESTARTS   AGE
pod/nacos-cluster-0 1/1     Running   0          5m
pod/nacos-cluster-1 1/1     Running   0          4m
pod/nacos-cluster-2 1/1     Running   0          3m

NAME                          CAPACITY   STATUS   CLAIM               STORAGECLASS
persistentvolume/nacos-pv-0   10Gi       Bound    default/nacos-pvc-0
persistentvolume/nacos-pv-1   10Gi       Bound    default/nacos-pvc-1
persistentvolume/nacos-pv-2   10Gi       Bound    default/nacos-pvc-2

NAME                          STATUS   VOLUME       CAPACITY   STORAGECLASS
persistentvolumeclaim/nacos-pvc-0 Bound    nacos-pv-0   10Gi
persistentvolumeclaim/nacos-pvc-1 Bound    nacos-pv-1   10Gi
persistentvolumeclaim/nacos-pvc-2 Bound    nacos-pv-2   10Gi
```

```bash
kubectl get service
```
预期输出:
```
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
nacos-headless   ClusterIP   None             <none>        8848/TCP,7848/TCP,9848/TCP   38m
nacos-nodeport   NodePort    10.104.8.204     <none>        8848:30002/TCP               38m
```