---
title: K8S 部署 MySQL
date: 2025-02-21 12:02:05
tags: 
  - K8S
  - MySQL
categories: 运维部署
---
## 一、NFS服务搭建
1. 安装NFS服务端（选择一个节点作为NFS服务器）
   ```bash
    # 安装依赖
    yum install -y nfs-utils rpcbind

    # 创建共享目录并配置权限
    mkdir -p /mnt/nfs/mysql
    chmod 777 /mnt/nfs/mysql

    # 编辑NFS配置文件
    echo "/mnt/nfs/mysql *(rw,sync,no_root_squash,no_all_squash)" >> /etc/exports

    # 启动服务并设置开机自启
    systemctl enable --now rpcbind nfs-server
    exportfs -arv  # 刷新配置
   ```
2. 配置防火墙（若启用, 本地防火墙未启用, 安装k8s的时候关了）
   ```bash
    firewall-cmd --permanent --add-service=nfs
    firewall-cmd --permanent --add-service=rpc-bind
    firewall-cmd --reload
   ```

## 二、创建PV（PersistentVolume）与PVC（PersistentVolumeClaim）
1. 定义PV
   ```yaml
   # mysql-pv.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: mysql-pv
    spec:
    storageClassName: manual
    capacity:
        storage: 20Gi
    accessModes:
        - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    nfs:
        path: /mnt/nfs/mysql  # NFS共享目录路径
        server: <NFS_SERVER_IP>  # NFS服务器IP地址
   ```
2. 定义PVC
   ```yaml
    # mysql-pvc.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: mysql-pvc
    spec:
    storageClassName: manual
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
        storage: 20Gi
   ```
3. 应用配置
   ```bash
    kubectl apply -f mysql-pv.yaml
    kubectl apply -f mysql-pvc.yaml
   ```

## 三、部署MySQL
1. 创建MySQL部署
   ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: mysql
    spec:
    selector:
        matchLabels:
        app: mysql
    template:
        metadata:
        labels:
            app: mysql
        spec:
        containers:
            - name: mysql
            image: mysql:5.7
            env:
                - name: MYSQL_ROOT_PASSWORD
                  value: "root"
            ports:
                - containerPort: 3306
            volumeMounts:
                - name: mysql-data
                mountPath: /var/lib/mysql
        volumes:
            - name: mysql-data
            persistentVolumeClaim:
                claimName: mysql-pvc
   ```
2. 应用配置
   ```bash
    kubectl apply -f mysql-deployment.yaml
   ```

## 四、暴露MySQL服务供局域网访问
1. 创建NodePort Service
   ```yaml
    # mysql-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: mysql-service
    spec:
    type: NodePort
    selector:
        app: mysql
    ports:
        - port: 3306
        targetPort: 3306
        nodePort: 30306  # 自定义端口范围30000-32767
   ```
2. 应用Service配置
   ```bash
    kubectl apply -f mysql-service.yaml
   ```
3. 开放防火墙端口
   ```bash
    firewall-cmd --permanent --add-port=30306/tcp
    firewall-cmd --reload
   ```
## 五、配置MySQL远程访问权限
1. 进入MySQL容器授权远程登录
   ```bash
    kubectl exec -it mysql-pod-name -- mysql -u root -p 
   ```
2. 执行以下SQL命令
   ```sql
    ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'your_password';
    FLUSH PRIVILEGES;
   ```
3. 验证远程访问权限
   ```bash
    mysql -u root -p -h <NFS_SERVER_IP> -P 30306
   ```