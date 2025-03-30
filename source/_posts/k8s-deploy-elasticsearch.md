---
title: K8S 部署 ElasticSearch
date: 2025-02-25 19:28:21
tags:
    - K8S
    - ElasticSearch
categories: 运维部署
---
# K8S部署Elasticsearch

## 1. 准备工作
### NFS服务器配置
```bash
mkdir -p /mnt/nfs/es/pv-es-{0,1,2}
chmod 777 /mnt/nfs/es/pv-es-{0,1,2}
```
## 2. 部署NFS存储
### 2.1 创建PV 和 PVC
```yaml
# es-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-es-0
  labels:
    type: nfs-pv
    app: es
    pv-index: "0"
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: 192.168.3.130
    path: /mnt/nfs/es/pv-es-0
  claimRef:
    name: es-data-es-cluster-0
    namespace: default
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-es-1
  labels:
    type: nfs-pv
    app: es
    pv-index: "1"
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: 192.168.3.130
    path: /mnt/nfs/es/pv-es-1
  claimRef:
    name: es-data-es-cluster-1
    namespace: default
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-es-2
  labels:
    type: nfs-pv
    app: es
    pv-index: "2"
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: 192.168.3.130
    path: /mnt/nfs/es/pv-es-2
  claimRef:
    name: es-data-es-cluster-2
    namespace: default
```
### 2.2 应用配置
```bash
kubectl apply -f es-pv.yaml
```

## 3. 部署Elasticsearch集群
### 3.1 创建ConfigMap配置ES
```yaml
# es-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: es-config
data:
  elasticsearch.yml: |
    cluster.name: es-cluster
    node.name: ${HOSTNAME}
    network.host: 0.0.0.0
    discovery.seed_hosts: es-cluster-headless
    cluster.initial_master_nodes: es-cluster-0,es-cluster-1,es-cluster-2
    path.data: /usr/share/elasticsearch/data
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.http.ssl.enabled: true
    xpack.security.http.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.http.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
```

### 3.2 创建Secret
```yaml
# es-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: es-credentials
type: Opaque
data:
  # echo -n "xxxxxxxxxxx" | base64 换成自己的密码
  elastic: xxxxxxxxxxx
  # echo -n "xxxxxxxxxxx" | base64
  kibana: xxxxxxxxxxx
```

### 3.3 创建StatefulSet部署ES
```yaml
# es-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
spec:
  serviceName: es-cluster-headless
  replicas: 3
  selector:
    matchLabels:
      app: es-cluster
  template:
    metadata:
      labels:
        app: es-cluster
    spec:
      initContainers:
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.16.6
        env:
        - name: ES_JAVA_OPTS
          value: "-Xms2g -Xmx2g"  # JVM堆内存，根据节点调整
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: es-data
          mountPath: /usr/share/elasticsearch/data
        - name: es-config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          subPath: elasticsearch.yml
        - name: certs
          mountPath: /usr/share/elasticsearch/config/certs
        - name: credentials
          mountPath: /usr/share/elasticsearch/config/passwords
      volumes:
      - name: es-config
        configMap:
          name: es-config
      - name: certs
        secret:
          secretName: es-certificates
      - name: credentials
        secret:
          secretName: es-credentials
  volumeClaimTemplates:
  - metadata:
      name: es-data
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName: nfs
      resources:
        requests:
          storage: 5Gi
```

### 3.4 创建Service暴露ES服务
```yaml
# es-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: es-cluster-headless
spec:
  clusterIP: None
  selector:
    app: es-cluster
  ports:
  - port: 9200
    name: http
  - port: 9300
    name: transport
---
apiVersion: v1
kind: Service
metadata:
  name: es-cluster
spec:
  type: NodePort
  selector:
    app: es-cluster
  ports:
  - port: 9200
    targetPort: 9200
    nodePort: 30920
```
### 3.5 应用配置
```bash
kubectl delete -f es-secret.yaml
kubectl delete -f es-configmap.yaml
kubectl delete -f es-statefulset.yaml
kubectl delete -f es-service.yaml
```

### 3.6 证书问题
开启了`xpack.security.enabled: true`，需要证书才能访问。生成证书。

```bash
# 在主节点执行
cd <yaml文件所在的命令>

# 生成CA证书
docker run --rm -v $(pwd):/certificates --user root \
docker.elastic.co/elasticsearch/elasticsearch:8.16.6 \
bin/elasticsearch-certutil ca --out /certificates/elastic-stack-ca.p12 --pass ""

# 生成节点证书
docker run -it --rm -v $(pwd):/certificates --user root \
docker.elastic.co/elasticsearch/elasticsearch:8.16.6 \
bin/elasticsearch-certutil cert --ca /certificates/elastic-stack-ca.p12 --pass "" \
--out /certificates/elastic-certificates.p12

# 创建集群级Secret（自动同步到所有节点）：
kubectl create secret generic es-certificates \
--from-file=elastic-certificates.p12=./elastic-certificates.p12 \
--from-file=elastic-stack-ca.p12=./elastic-stack-ca.p12

# 在其他节点验证节点Secret同步状态：
kubectl get secret es-certificates --output=jsonpath='{.metadata.name}'
```

## 4. 部署Kibana
### 4.1 创建Kibana Deployment
```yaml
# kibana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.16.6
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://es-cluster:9200"
        ports:
        - containerPort: 5601
```
### 4.2 创建Service暴露Kibana服务
```yaml
# kibana-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
spec:
  type: NodePort
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
    nodePort: 30561
```
### 4.3 应用配置
```bash
kubectl apply -f kibana-deployment.yaml
kubectl apply -f kibana-service.yaml
```

## 5. 验证部署

1. 检查Pod状态
   ```bash
   kubectl get pods -l app=es-cluster
   kubectl get pods -l app=kibana
   ```
2. 访问Kibana界面
   浏览器输入`http://<节点IP>:30561`
