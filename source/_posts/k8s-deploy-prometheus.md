---
title: K8S 部署 Prometheus
description: K8S 部署 Prometheus
date: 2025-03-01 18:33:23
tags:
    - K8S
    - 监控
    - Prometheus
categories: 运维部署
---
# K8S搭建Prometheus+Grafana监控
## 1. 准备工作
### NFS服务器配置
安装NFS服务器并配置：
```bash
 # 安装依赖
 yum install -y nfs-utils rpcbind
 # 创建共享目录并配置权限
 mkdir -p /mnt/nfs/{prometheus,grafana}
 chmod 777 /mnt/nfs/{prometheus,grafana}
 # 编辑NFS配置文件
 echo "/mnt/nfs/prometheus *(rw,sync,no_root_squash,no_all_squash)" >> /etc/exports
 echo "/mnt/nfs/grafana *(rw,sync,no_root_squash,no_all_squash)" >> /etc/exports
 # 启动服务并设置开机自启
 systemctl enable --now rpcbind nfs-server
 exportfs -arv  # 刷新配置
```

或者只配置`/mnt/nfs`
```bash
# 配置/mnt/nfs, 而不是指定的/mnt/nfs/prometheus和/mnt/nfs/grafana
echo "/mnt/nfs *(rw,sync,no_root_squash,no_all_squash)" >> /etc/exports
```

### 创建命名空间
```bash
kubectl create namespace monitoring
```

## 2. 部署NFS存储

### Prometheus PV和PVC
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv
  namespace: monitoring
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.3.130 # 换成自己的NFS服务器IP
    path: /mnt/nfs/prometheus
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 4Gi
```

### Grafana PV和PVC
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-pv
  namespace: monitoring
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.3.130  # 换成自己的NFS服务器IP
    path: /mnt/nfs/grafana
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

### 应用配置

```bash
kubectl apply -f prometheus-pv.yaml
kubectl apply -f grafana-pv.yaml
```

## 3. 部署Node Exporter
```yaml
# node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($|/)
        ports:
        - containerPort: 9100
        volumeMounts:
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: root
          mountPath: /rootfs
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /

---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    app: node-exporter
  ports:
  - name: http
    port: 9100
    protocol: TCP
```

### 应用配置
```bash
kubectl apply -f node-exporter.yaml
```

## 4. 部署Kube-State-Metrics
从`https://github.com/kubernetes/kube-state-metrics/tree/main/examples/standard`将文件下载下来, 并将文件内的`namespace`改为`monitoring`, 然后应用配置:
```bash
kubectl apply -f cluster-role.yaml
kubectl apply -f cluster-role-binding.yaml
kubectl apply -f service-account.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

## 5. 部署Prometheus
### 创建ConfigMap
```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
        - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_label_app]
          action: keep
          regex: node-exporter
      - job_name: 'kube-state-metrics'
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names: [monitoring]
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_name]
          action: keep
          regex: kube-state-metrics
```

### 创建RBAC
```yaml
# prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

### 创建Deployment和Service
```yaml
# prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.path=/prometheus
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: storage-volume
          mountPath: /prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: storage-volume
        persistentVolumeClaim:
          claimName: prometheus-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30010
```

### 应用
```bash
kubectl apply -f prometheus-config.yaml
kubectl apply -f prometheus-rbac.yaml
kubectl apply -f prometheus-deployment.yaml
```

## 6. 部署Grafana
```yaml
# grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30020
```

### 应用配置
```bash
kubectl apply -f grafana-deployment.yaml
```

## 7. 配置Grafana
1. 访问Grafana: `http://<节点IP>:30020`, 默认用户名和密码都是`admin`, 登录以后会重置密码
2. 添加数据源:
   - 新版本中是`Connections` -> `Data Sources`
   - 选择`Prometheus`
   - URL填写`http://prometheus.monitoring.svc.cluster.local:9090`
3. 导入Dashboard:
   - `Dashboards` -> `Import`
   - ID填`1860` (Node Exporter Full)
   - ID填`17519` (kube-state-metrics-v2-修正版)
   - 如果用ID不行, 可以在`https://grafana.com/grafana/dashboards/`中搜索对应的ID, 将文件下载下来, 然后导入