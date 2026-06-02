+++
title = "nacos"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 2
+++

# k8s部署nacos

官网：https://nacos.io/zh-cn/docs/v2/quickstart/quick-start-kubernetes.html



一、NFS持久化模式安装，如果不用NFS可忽略

```bash
# 安装客户端 nfs
yum install nfs-utils -y

# 启动 nfs
systemctl start nfs-server
systemctl enable nfs-server

# 查看 nfs 版本
cat /proc/fs/nfsd/versions

# 创建共享目录
mkdir -p /data/nfs
cd /data/nfs
mkdir rw
mkdir ro

# server上设置共享目录 export
vim /etc/exports
/data/nfs/rw 192.168.113.0/24(rw,sync,no_subtree_check,no_root_squash)
/data/nfs/ro 192.168.113.0/24(ro,sync,no_subtree_check,no_root_squash)

# 重新加载
exportfs -f
systemctl reload nfs-server

# 到其他测试节点安装 nfs-utils 并加载测试
showmount -e 192.168.113.121

mkdir -p /mnt/nfs/rw
mount -t nfs 192.168.113.121:/data/nfs/rw /mnt/nfs/rw
```

二、官网clone安装文件

```bash
# 获取安装文件：
git clone https://github.com/nacos-group/nacos-k8s.git

# 进入到nfs目录下创建nfs-client资源
$ cd nacos-k8s/deploy/nfs
$ ls
class.yaml  deployment.yaml  rbac.yaml
```

1、创建nfs-client-provisioner

rbac.yaml：rbac 用户创建，根据需要修改命名空间

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
- kind: ServiceAccount
  name: nfs-client-provisioner
  namespace: psw002-test
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
- kind: ServiceAccount
  name: nfs-client-provisioner    
  # replace with namespace where provisioner is deployed
  namespace: psw002-test
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

官方的修改命令：

```yaml
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
$ NAMESPACE=${NS:-default}
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/nfs/rbac.yaml

```

class.yaml：制备器

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  namespace: psw002-test
provisioner: fuseim.pri/ifs
parameters:
  archiveOnDelete: "false"
```

deployment.yaml ：修改命名空间，镜像等

```yaml

kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: psw002-test      # 根据需要更改命名空间
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  namespace: psw002-test         # 根据需要更改命名空间
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccount: nfs-client-provisioner
      nodeSelector:
        app: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          #image: quay.io/external_storage/nfs-client-provisioner:latest  # 修改镜像，在22版之后，k8s默认不支持selLine ，需要使用较低版本的
          image: registry.cn-beijing.aliyuncs.com/pylixm/nfs-subdir-external-provisioner:v4.0.0
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 10.244.16.21        # 更改server ip
            - name: NFS_PATH
              value: /data/nfs-share/nacos   # 路径
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.244.16.21     # 更改server ip
            path: /data/nfs-share/nacos   # 路径
```

<img src="./images/image-20231211114418023.png" alt="image-20231211114418023" style="zoom: 33%;" />

修改之后直接

```bash
kubectl apply -f .
```

2、导入sql，到mysql，这里我是自建的外部的mysql，所以k8s中没有装mysql，但是需要创建一个service让集群内额pod能够访问外部的mysql

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-nacos
  namespace: psw002-test
spec:
  #clusterIP: 10.96.2.128 #固定clusterIP
  type: NodePort
  ports:
  - port: 3306
    nodePort: 32206
    targetPort: 3306
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-nacos
  namespace: psw002-test
subsets:
  - addresses:
    - ip: 10.14.2.5  #外部数据库地址
    ports:
    - port: 3306
```



https://github.com/alibaba/nacos/blob/develop/config/src/main/resources/META-INF/nacos-db.sql

<img src="./images/image-20231211155729742.png" alt="image-20231211155729742" style="zoom:33%;" />



3、创建nacos StatefulSet

进入到nacos目录下：nacos-k8s/deploy/nacos

根据需要修改：nacos-pvc-nfs.yaml

```yaml
# 请阅读Wiki文章
# https://github.com/nacos-group/nacos-k8s/wiki/%E4%BD%BF%E7%94%A8peerfinder%E6%89%A9%E5%AE%B9%E6%8F%92%E4%BB%B6
---
apiVersion: v1
kind: Service
metadata:
  name: nacos-headless
  namespace: psw002-test
  labels:
    app: nacos
spec:
  publishNotReadyAddresses: true 
  ports:
    - port: 8848
      name: server
      targetPort: 8848
    - port: 9848
      name: client-rpc
      targetPort: 9848
    - port: 9849
      name: raft-rpc
      targetPort: 9849
    ## 兼容1.4.x版本的选举端口
    - port: 7848
      name: old-raft-rpc
      targetPort: 7848
  clusterIP: None
  type: ClusterIP          # 这里的service不能改为nodePort模式，不然会出错
  selector:
    app: nacos
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: psw002-test
  name: nacos-cm
data:
  mysql.host: "mysql-nacos"      # 数据库连接地址，如果是连接外部的mysql，需要创建一个外部连接的service
  mysql.db.name: "nacos"         # 库名
  mysql.port: "3306"             # 端口
  mysql.user: "nacos"            # 用户，密码
  mysql.password: "weizhen@nacos123"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: psw002-test
spec:
  podManagementPolicy: Parallel
  serviceName: nacos-headless
  replicas: 2
  template:
    metadata:
      labels:
        app: nacos
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                      - nacos
              topologyKey: "kubernetes.io/hostname"
      serviceAccountName: nfs-client-provisioner
      initContainers:
        - name: peer-finder-plugin-install
          image: nacos/nacos-peer-finder-plugin:1.1
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - mountPath: /home/nacos/plugins/peer-finder
              name: data
              subPath: peer-finder
      nodeSelector:
        nacos: nacos
      containers:
        - name: nacos
          imagePullPolicy: IfNotPresent
          #image: nacos/nacos-server:latest
          image: swr.cn-south-1.myhuaweicloud.com/psw002-test/nacos-server:v2.3.0    # 根据自己需要修改版本
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
          ports:
            - containerPort: 8848
              name: client-port
            - containerPort: 9848
              name: client-rpc
            - containerPort: 9849
              name: raft-rpc
            - containerPort: 7848
              name: old-raft-rpc
          env:
            - name: NACOS_REPLICAS
              value: "1"
            - name: SERVICE_NAME
              value: "nacos-headless"
            - name: DOMAIN_NAME
              value: "cluster.local"
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            - name: MYSQL_SERVICE_HOST
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.host
            - name: MYSQL_SERVICE_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.name
            - name: MYSQL_SERVICE_PORT
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.port
            - name: MYSQL_SERVICE_USER
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.user
            - name: MYSQL_SERVICE_PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.password
            - name: SPRING_DATASOURCE_PLATFORM
              value: "mysql"
            - name: NACOS_SERVER_PORT
              value: "8848"
            - name: NACOS_APPLICATION_PORT
              value: "8848"
            - name: PREFER_HOST_MODE
              value: "hostname"
			
			# nacos2.x之后默认不开启权限认证，访问web端时无需登录，所以需要开启权限认证
            - name: NACOS_AUTH_ENABLE
              value: "true"
            - name: NACOS_AUTH_IDENTITY_KEY
              value: "auth-key"
            - name: NACOS_AUTH_IDENTITY_VALUE
              value: "auth-key"
            - name: NACOS_AUTH_TOKEN
              value: "SecretKey012345678901234567890123456789012345678901234567890123456789"
            - name: NACOS_AUTH_TOKEN_EXPIRE_SECONDS
              value: "18000"

          volumeMounts:
            - name: data
              mountPath: /home/nacos/plugins/peer-finder
              subPath: peer-finder
            - name: data
              mountPath: /home/nacos/data
              subPath: data
            - name: data
              mountPath: /home/nacos/logs
              subPath: logs
  volumeClaimTemplates:
    - metadata:
        name: data
        annotations:
          volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"      # 这里要对应前面创建的制备器
      spec:
        accessModes: [ "ReadWriteMany" ]
        resources:
          requests:
            storage: 10Gi
  selector:
    matchLabels:
      app: nacos

```



nacos启动报错

1、

![image-20231211210734308](./images/image-20231211210734308.png)

2、内置的tomcat起不来

![image-20231211210818516](./images/image-20231211210818516.png)

解决：

因为我在更改了yaml文件中的service网络模式，原来是ClueterIP模式，改为nodePort模式，而实际上不能更改，需要外部访问只能通过另外创建一个nodePort的service或者ingress来实现

<img src="./images/image-20231212091236608.png" alt="image-20231212091236608" style="zoom:50%;" />

参考博客：

​	https://www.cnblogs.com/panwenbin-logs/p/17562928.html

​	https://developer.aliyun.com/ask/529334

![image-20231212113919915](./images/image-20231212113919915.png)



            - name: NACOS_AUTH_ENABLE
              value: "true"
            - name: NACOS_AUTH_IDENTITY_KEY
              value: "auth-key"
            - name: NACOS_AUTH_IDENTITY_VALUE
              value: "auth-key"
            - name: NACOS_AUTH_TOKEN
              value: "SecretKey012345678901234567890123456789012345678901234567890123456789"
            - name: NACOS_AUTH_TOKEN_EXPIRE_SECONDS
              value: "18000"





