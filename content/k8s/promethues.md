+++
title = "promethues"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 13
+++

# 前提说明

本文环境基于：kubadm安装的k8s集群（1.23），kube-prometheus安装的prometheus

并不适用与所有环境，需根据自己环境进行调整，但配置都一样，只是配置的路径位置以及配置生效等实现不一样...

# prometheus

## 架构

<img src="./images/2052820-20221005213455177-1441519849.png" alt="img" style="zoom:80%;" />

![image-20240407095824996](./images/image-20240407095824996.png)

**组件：**

- Prometheus Server：主服务器，负责收集和存储时间序列数据

- client libraies：应用程序代码插桩，将监控指标嵌入到被监控应用程序中

- Pushgateway：推送网关，为支持 short-lived 作业提供一个推送网关

- exporter：专门为一些应用开发的数据摄取组件—exporter，例如：HAProxy、StatsD、Graphite 等等。

- Alertmanager：专门用于处理 alert 的组件

**采集层：**

采集层分为两类，一类是生命周期较短的作业，还有一类是生命周期较长的作业。

- 短作业：直接通过 API，在退出时间指标推送给 Pushgateway。

- 长作业：Retrieval 组件直接从 Job 或者 Exporter 拉取数据。

**应用层**

应用层主要分为两种：

- AlertManager：对接 Pagerduty，是一套付费的监控报警系统。可实现短信报警、5 分钟无人 ack 打电话通知、仍然无人 ack，通知值班人员 Manager，发送邮件等

- 数据可视化：Prometheus build-in WebUI，Grafana



## **PromQL 介绍**

​	PromQL 是 Prometheus **内置的数据查询语言**，其提供对时间序列数据丰富的查询，聚合以及逻辑运算能力的支持。并且被广泛应用在 Prometheus的日常应用当中，包括对数据查询、可视化、告警处理当中。

​	通过指标名称（metrics name）以及对应的一组标签（labelset）唯一定义一条时间序列。指标名称反映了监控样本的基本标识，而 label 则在这个基本特征上为采集到的数据提供了多种特征维度。用户可以基于这些特征维度过滤，聚合，统计从而产生新的计算后的一条时间序列

优秀学习博文：https://yunlzheng.gitbook.io/prometheus-book



# 安装

## kube-prometheus

kube-prometheus项目地址：https://github.com/prometheus-operator/kube-prometheus

​	Operator是由[ CoreOS](https://blog.z0ukun.com/wp-content/themes/begin4.6/inc/go.php?url=https://coreos.com/) 公司开发的，用来扩展 Kubernetes API，特定的应用程序控制器。它被用来创建、配置和管理复杂的有状态应用，如数据库、缓存和监控系统。Operator 是基于 Kubernetes 的资源和控制器概念之上构建，但同时又包含了应用程序特定的一些专业知识：比如创建一个数据库的Operator，则必须对创建的数据库的各种运维方式非常了解，创建Operator的关键是CRD（自定义资源）的设计。

​	`Operator`是将运维人员对软件操作的知识给代码化，同时利用 Kubernetes 强大的抽象来管理大规模的软件应用。目前`CoreOS`官方提供了几种`Operator`的实现，其中就包括我们今天的主角：`Prometheus Operator`，`Operator`的核心实现就是基于 Kubernetes 的以下两个概念：

- 资源：对象的状态定义
- 控制器：观测、分析和行动，以调节资源的分布

```
注：CRD是对 Kubernetes API 的扩展，Kubernetes 中的每个资源都是一个 API 对象的集合，例如我们在 YAML文件里定义的那些spec都是对 Kubernetes 中的资源对象的定义，所有的自定义资源可以跟 Kubernetes 中内建的资源一样使用 kubectl 操作。
```

架构：

![img](./images/886010-20200826095758254-2043383800.png)



参考博客：https://blog.csdn.net/weixin_46887489/article/details/135350632?spm=1001.2014.3001.5502

Operator部署器是基于已经编写好的yaml文件，prometheus server、alertmanager、grafana、node-exporter、cadvisor等组件⼀键批量部署在k8s集群当中。

```bash
# 克隆，-b指定下载版本，不通k8s版本对应不一样
git clone -b release-0.11 https://github.com/prometheus-operator/kube-prometheus.git

# 或者直接wget下载
wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/heads/release-0.13.zip
```

解压之后进入目录：`kube-prometheus-0.13.0/manifests`

移走自带的networkPolicy相关的网络策略文件信息，否则创建这些资源之后，访问服务都会被阻塞，对应issue信息：

```bash
https://github.com/prometheus-operator/kube-prometheus/issues/1763#issuecomment-1139553506
```

```bash
[root@k8s-master1 manifests]# ll *networkPolicy*
-rw-r--r-- 1 root root  977 Mar 19 20:08 alertmanager-networkPolicy.yaml
-rw-r--r-- 1 root root  722 Mar 19 20:08 blackboxExporter-networkPolicy.yaml
-rw-r--r-- 1 root root  651 Mar 19 20:08 grafana-networkPolicy.yaml
-rw-r--r-- 1 root root  723 Mar 19 20:08 kubeStateMetrics-networkPolicy.yaml
-rw-r--r-- 1 root root  671 Mar 19 20:08 nodeExporter-networkPolicy.yaml
-rw-r--r-- 1 root root  565 Mar 19 20:08 prometheusAdapter-networkPolicy.yaml
-rw-r--r-- 1 root root 1073 Mar 19 20:08 prometheus-networkPolicy.yaml
-rw-r--r-- 1 root root  694 Mar 19 20:08 prometheusOperator-networkPolicy.yaml

# 移动到networkPolicy目录下
[root@k8s-master1 manifests]# mkdir networkPolicy && mv *networkPolicy*.yaml networkPolicy
```

查看镜像，因为有些镜像在国外，没有梯子无法下载：

```bash
[root@k8s-master1 manifests]# grep -i 'image:' -R ./* |awk '{print $3}'
docker.io/bogeit/alertmanager:v0.26.0
docker.io/bogeit/blackbox-exporter:v0.24.0
jimmidyson/configmap-reload:v0.5.0
docker.io/bogeit/kube-rbac-proxy:v0.14.2
docker.io/bogeit/grafana:9.5.3
docker.io/bogeit/kube-state-metrics:v2.9.2
docker.io/bogeit/kube-rbac-proxy:v0.14.2
docker.io/bogeit/kube-rbac-proxy:v0.14.2
docker.io/bogeit/node-exporter:v1.6.1
docker.io/bogeit/kube-rbac-proxy:v0.14.2
docker.io/bogeit/prometheus-adapter:v0.11.1
docker.io/bogeit/prometheus-operator:v0.67.1
docker.io/bogeit/kube-rbac-proxy:v0.14.2
docker.io/bogeit/prometheus:v2.46.0


# 由于上面的镜像中，有部分国内网络无法正常摘取，所以博哥将上述所有镜像已转存至docker hub上，用下面命令批量替换下镜像地址即可
find ./ -type f |xargs  sed -ri 's+quay.io/.*/+docker.io/bogeit/+g'
find ./ -type f |xargs  sed -ri 's+docker.io/cloudnativelabs/+docker.io/bogeit/+g'
find ./ -type f |xargs  sed -ri 's+grafana/+docker.io/bogeit/+g'
find ./ -type f |xargs  sed -ri 's+registry.k8s.io/.*/+docker.io/bogeit/+g'

```

开始创建资源

```bash
# 要创建setup下的资源
kubectl create -f manifests/setup
kubectl create -f manifests/
```

外部网页访问

- 方式1：更改grafana和prometheus的service为nodePort模式


```bash
vim grafana-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 9.5.3
  name: grafana
  namespace: monitoring
spec:
  type: NodePort         # 添加修改模式
  ports:
  - name: http
    port: 3000
    targetPort: http
    nodePort: 33000       # 暴漏的端口
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus

```

```bash
vim prometheus-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.46.0
  name: prometheus-k8s
  namespace: monitoring
spec:
  type: NodePort          # 添加修改模式
  ports:
  - name: web
    port: 9090
    targetPort: web
    nodePort: 39090       # 暴漏的端口
  - name: reloader-web
    port: 8080
    targetPort: reloader-web
  selector:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
  sessionAffinity: ClientIP
```

- 方式2：创建ingress资源


```bash
cat >prometheus-grafana-ingress.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
spec:
  rules:
  - host: prometheus.boge.com
    http:
      paths:
      - backend:
          service:
            name: prometheus-k8s
            port:
              number: 9090
        path: /
        pathType: Prefix

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
spec:
  rules:
  - host: grafana.boge.com
    http:
      paths:
      - backend:
          service:
            name: grafana
            port:
              number: 3000
        path: /
        pathType: Prefix
EOF

```

然后就可以进行网页访问，grafana的默认用户密码：admin/admin



**prometheus数据持久化配置**

```bash
# kube-prometheus使用的statefulset服务来启动，也是我们需要做数据持久化的地方
# kubectl -n monitoring get statefulset,pod|grep prometheus-k8s
statefulset.apps/prometheus-k8s      2/2     3h41m
pod/prometheus-k8s-0                       2/2     Running   1          19m
pod/prometheus-k8s-1                       2/2     Running   1          19m

```

基于nfs的StorageClass动态存储（如果已经创建StorageClass，可跳过安装步骤）

安装NFS服务器：

```bash
# 在nfs上安装nfs服务
[root@nfs ~]# yum install nfs-utils -y
# 准备一个共享目录
[root@nfs ~]# mkdir /root/data/nfs -pv

# 将共享目录以读写权限暴露给192.168.5.0/24网段中的所有主机
[root@nfs ~]# vim /etc/exports
[root@nfs ~]# more /etc/exports
/root/data/nfs/prometheus     192.168.9.0/24(rw,no_root_squash)

# 启动nfs服务
[root@nfs ~]# systemctl restart nfs
# 在node上安装nfs服务，注意不需要启动
[root@k8s-master01 ~]# yum install nfs-utils -y
# 查看可使用的nfs共享目录
[root@k8s-master01 ~]# showmount -e

```

创建制备器

- 创建sa用户

  ```yaml
  cat > nfs-serviceAccount.yaml <<EOF
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: nfs-client-provisioner
    namespace: kube-system
  ---
  kind: ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: nfs-client-provisioner-runner
    namespace: kube-system
  rules:
    - apiGroups: [""]
      resources: ["persistentvolumes"]
      verbs: ["get", "list", "watch", "create", "delete"]
    - apiGroups: [""]
      resources: ["persistentvolumeclaims"]
      verbs: ["get", "list", "watch", "update"]
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
    namespace: kube-system
  subjects:
    - kind: ServiceAccount
      name: nfs-client-provisioner
      namespace: default
  roleRef:
    kind: ClusterRole
    name: nfs-client-provisioner-runner
    apiGroup: rbac.authorization.k8s.io
  ---
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: leader-locking-nfs-client-provisioner
    namespace: kube-system
  rules:
    - apiGroups: [""]
      resources: ["endpoints"]
      verbs: ["get", "list", "watch", "create", "update", "patch"]
  ---
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: leader-locking-nfs-client-provisioner
    namespace: kube-system
  subjects:
    - kind: ServiceAccount
      name: nfs-client-provisioner
  roleRef:
    kind: Role
    name: leader-locking-nfs-client-provisioner
    apiGroup: rbac.authorization.k8s.io
  EOF
  
  kubectl apply -f nfs-serviceAccount.yaml
  ```

- 创建storageclass资源 + 创建NFS provisioner （可以理解为nfs客户端），结合使用

  ```yaml
  cat >nfs-client.yaml <<EOF
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: nfs-storage
    namespace: kube-system
    annotations:
    #注解，是否是默认的存储，注意：KubeSphere默认就需要个默认存储
      storageclass.kubernetes.io/is-default-class: "false"  
  provisioner: fuseim.pri/ifs # 外部制备器名称（自定义）
  parameters:
    archiveOnDelete: "false" # 是否存档，false 表示不存档，会删除 oldPath 下面的数据，true 表示存档，会重命名路径
  reclaimPolicy: Retain # 回收策略，默认为 Delete 可以配置为 Retain
  volumeBindingMode: Immediate # 默认为 Immediate，表示创建 PVC 立即进行绑定，只有 azuredisk 和 AWSelasticblockstore 支持其他值
  
  ---
  # 创建NFS provisioner
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nfs-client-provisioner
    namespace: kube-system
    labels:
      app: nfs-client-provisioner
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
        serviceAccountName: nfs-client-provisioner    # 使用到刚刚创建的SA账户
        containers:
          - name: nfs-client-provisioner
          	# 更换镜像地址为国内源
            image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2
            volumeMounts:
              - name: nfs-client-root  
                mountPath: /persistentvolumes
            env:
              - name: PROVISIONER_NAME
                value: fuseim.pri/ifs    # 需要和StorageClass资源中的provisioner配置名称一致
              - name: NFS_SERVER
                value: 192.168.9.31   # nfs server端地址
              - name: NFS_PATH
                value: /data/nfs/prometheus      # 共享的目录
        volumes:
          - name: nfs-client-root       #存储卷的名称，和前面定义的保持一致
            nfs:
              server: 192.168.9.31   # nfs server端地址
              path: /data/nfs/prometheus        # 共享的目录
  EOF
  
  
  kubectl apply -f nfs-client.yaml
  ```

配置prometheus服务

```bash
# 准备prometheus持久化的pvc配置
# kubectl -n monitoring edit prometheus k8s

spec:
......
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: [ "ReadWriteOnce" ]  
        storageClassName: "nfs-storage"     # 创建的storageClass
        resources:
          requests:
            storage: 1Gi


# 上面修改保存退出后，过一会我们查看下pvc创建情况，以及pod内的数据挂载情况
# kubectl -n monitoring get pvc
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-055e6b11-31b7-4503-ba2b-4f292ba7bd06   1Gi        RWO            nfs-boge       17s
prometheus-k8s-db-prometheus-k8s-1   Bound    pvc-249c344b-3ef8-4a5d-8003-b8ce8e282d32   1Gi        RWO 

```

**grafana配置持久化存储配置**

```bash
# 保存pvc为grafana-pvc.yaml
cat > grafana-pvc.yaml <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana
  namespace: monitoring
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF

# 开始创建pvc
# kubectl apply -f grafana-pvc.yaml 

# 看下创建的pvc
# kubectl -n monitoring get pvc
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
grafana                              Bound    pvc-394a26e1-3274-4458-906e-e601a3cde50d   1Gi        RWX            nfs-storage       3s
prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-055e6b11-31b7-4503-ba2b-4f292ba7bd06   1Gi        RWO            nfs-storage       6m46s
prometheus-k8s-db-prometheus-k8s-1   Bound    pvc-249c344b-3ef8-4a5d-8003-b8ce8e282d32   1Gi        RWO            nfs-storage       6m46s


# 编辑grafana的deployment资源配置
# kubectl -n monitoring edit deployments.apps grafana 

# 旧的配置
  ...
      volumes:
      - emptyDir: {}
        name: grafana-storage
  ...
# 替换成新的配置
  ...
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana
  ...

# 同时加入下面的env环境变量，将登陆密码进行固定修改
    spec:
      containers:
      ......
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin321

# 过一会，等grafana重启完成后，用上面的新密码进行登陆
# kubectl -n monitoring get pod -w|grep grafana
grafana-5698bf94f4-prbr2               0/1     Running   0          3s
grafana-5698bf94f4-prbr2               1/1     Running   0          4s

```



## 二进制

将prometheus的每个组件进行模块化单独部署，其中prometheus Server、grafana、alertmanager、node-exporter等监控组件,使用单独的服务器进行二进制安装或者单独的容器进行部署

- 安装prometheus-server

官网：https://prometheus.io/download/

github地址：https://github.com/prometheus/prometheus

选择稳定正式版本进行安装

<img src="./images/image-20240409143435049.png" alt="image-20240409143435049" style="zoom:80%;" />

解压安装

```bash
tar xvf prometheus-2.37.8.linux-amd64.tar.gz -C /opt/
cd /opt/ && mv prometheus-2.37.8.linux-amd64/ prometheus-2.37.8 && cd prometheus-2.37.8
ln -sf /opt/prometheus-2.37.8/prometheus /usr/bin/prometheus

# 文件夹
[root@k8s-master1 prometheus-2.37.8]# ll
total 207040
drwxr-xr-x 2 1001 docker        38 May  5  2023 console_libraries
drwxr-xr-x 2 1001 docker       173 May  5  2023 consoles
-rw-r--r-- 1 1001 docker     11357 May  5  2023 LICENSE
-rw-r--r-- 1 1001 docker      3773 May  5  2023 NOTICE
-rwxr-xr-x 1 1001 docker 109128753 May  5  2023 prometheus
-rw-r--r-- 1 1001 docker       934 May  5  2023 prometheus.yml
-rwxr-xr-x 1 1001 docker 102856799 May  5  2023 promtool

```

```bash
# 检查配置文件
#./promtool check config prometheus.yml
Checking prometheus.yml
 SUCCESS: prometheus.yml is valid prometheus config file syntax

```

配置systemd启动文件，**默认监听9090端口**

```bash
cat > /etc/systemd/system/prometheus.service << EOF
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network.target
 
[Service]
Restart=on-failure
WorkingDirectory=/usr/local/prometheus/
ExecStart=/opt/prometheus-2.37.8/prometheus --config.file=/opt/prometheus-2.37.8/prometheus.yml --web.enable-lifecycle 
--storage.tsdb.retention=720h
 
[Install]
WantedBy=multi-user.target
EOF
```

```bash
systemctl daemon-reload && systemctl enable prometheus --now && systemctl status prometheus
```

在浏览器访问：ip:9090，即可进入



prometheus服务配置动态热加载

```bash
curl -X POST http://192.168.9.30:9090/-/reload
```



## deployment安装

创建prometheus的配置文件configmap，并且实现**服务发现**的配置

```yaml
cat > prometheus-config.yaml <<EOF 
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitoring 
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 1m
    scrape_configs:
    - job_name: 'kubelet'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token    
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-node-cadvisor'
      kubernetes_sd_configs:
      - role:  node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    - job_name: 'kubernetes-apiserver'
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
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_service_name
        
EOF

```

```bash
kubectl apply -f prometheus-config.yaml
```

创建持久化存储的pv和pvc（NFS的安装参考kube-prometheus章节）

```yaml
cat >prometheus-pvc-pv.yaml <<EOF
# 创建pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-nfs
  labels:
    type: nfs
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: "/data/nfs/prometheus"
    server: 192.168.9.31

---
# 创建pvc
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus-nfs
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      type: nfs
EOF

```

```bash
kubectl apply -f prometheus-pvc-pv.yaml
```

```bash
kubectl get pvc
NAME             STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-nfs   Bound    prometheus-nfs   20Gi       RWO                           32s

kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS   REASON   AGE
prometheus-nfs   20Gi       RWO            Retain           Bound    monitoring/prometheus-nfs                           3m22s
```



对prometheus-server分配集群控制权限，能够动态发现pod

 **创建sa账号**

```bash
kubectl create serviceaccount monitor -n monitoring serviceaccount/monitor created
```

**对sa账号monitor进行clusterrole权限绑定（在此演示给的admin权限，可根据需要控制权限）**

```bash
kubectl create clusterrolebinding monitoring-clusterrolebinding -n monitoring --clusterrole cluster-admin --serviceaccount=monitoring:monitor
clusterrolebinding.rbac.authorization.k8s.io/monitoring-clusterrolebinding created

```

创建proetheus-server

```yaml
cat > prometheus-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: server
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      serviceAccountName: monitor          #指定sa账号赋予集群权限
      containers:
      - name: prometheus
        image: prom/prometheus:v2.31.2
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --storage.tsdb.retention=720h       #保留30天数据
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus/prometheus.yml
          name: prometheus-config
          subPath: prometheus.yml
        - mountPath: /prometheus/
          name: prometheus-storage-volume
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
            items:
              - key: prometheus.yml
                path: prometheus.yml
                mode: 0644
        - name: prometheus-storage-volume
          persistentVolumeClaim:
            claimName: prometheus-nfs

EOF
```

```bash
kubectl apply -f prometheus-deployment.yaml
```

创建svc（nodePort模式可直接访问）

```yaml
cat >prometheus-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 31090
      protocol: TCP
  selector:
    app: prometheus
    component: server
EOF

```

```bash
kubectl apply -f prometheus-svc.yaml
```

浏览器访问prometheus-server：集群ip:31090



# 采集器安装

## node-exporter

k8s各node节点使用⼆进制或者daemonset方式安装node_exporter，⽤于收集各k8s node节点宿主机的监控指标数据，默认监听端口为9100。

### 二进制部署

官网：https://prometheus.io/download/

github：https://github.com/prometheus/prometheus

下载解压安装

```bash
tar xvf node_exporter-1.4.1.linux-amd64.tar.gz -C /opt/
cd /opt && mv node_exporter-1.4.1.linux-amd64 node_exporter-1.4.1
ln -sf /opt/node_exporter-1.4.1/node_exporter /usr/bin/node_exporter
```

配置systemd启动

```bash
root@node1:~\ vim /etc/systemd/system/node-exporter.service
[Unit]
Description=Prometheus Node Exporter
After=network.target
 
[Service]
ExecStart=/opt/node_exporter-1.4.1/node_exporter
 
[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl restart node-exporter --now && systemctl status node-exporter
```

### daemonset部署

github：https://github.com/prometheus/prometheus

<img src="./images/image-20240409154758588.png" alt="image-20240409154758588" style="zoom: 50%;" />



```bash
docker run -d \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

创建yaml文件

```yaml
cat >daemonset-deploy-node-exporter.yaml << EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring 
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
        k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      tolerations:
        - effect: NoSchedule          #master节点污点容忍
          key: node-role.kubernetes.io/master
      containers:
      - image: prom/node-exporter:v1.4.1 
        imagePullPolicy: IfNotPresent
        name: prometheus-node-exporter
        ports:
        - containerPort: 9100    #端口暴露在9100
          hostPort: 9100
          protocol: TCP
          name: metrics
        volumeMounts:
        - mountPath: /host/proc
          name: proc
        - mountPath: /host/sys
          name: sys
        - mountPath: /host
          name: rootfs
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
      hostNetwork: true    #使用宿主机网络
      hostPID: true        #Pod使用宿主机的进程命名空间
EOF

```

``` bash
kubectl apply -f daemonset-deploy-node-exporter.yaml
```



## cadvisor

监控Pod指标数据需要使用cadvisor，cadvisor由谷歌开源，在kubernetes v1.11及之前的版本内置在kubelet中并监听在4194端口(https://github.com/kubernetes/kubernetes/pull/65707)，从v1.12开始kubelet中的cadvisor被移除，因此需要单独通过daemonset等方式部署。

  cadvisor（容器顾问）不仅可以搜集⼀台机器上所有运行的容器信息，还提供基础查询界⾯和http接口，方便其他组件如Prometheus进行数据抓取，cAdvisor可以对节点机器上的容器进行实时监控和性能数据采集，包括容器的CPU使用情况、内存使用情况、网络吞吐量及文件系统使用情况。

### docker部署

cadvisor github项目地址: https://github.com/google/cadvisor/releases

docker容器：

```bash
docker run -it -d \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  registry.cn-guangzhou.aliyuncs.com/any-any/cadvisor:v0.45.0   # 替换为阿里云镜像地址
```

containerd容器：

修改/var/lib/docker为/var/lib/containerd

```bash
docker run -it -d \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/containerd/:/var/lib/containerd:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  registry.cn-guangzhou.aliyuncs.com/any-any/cadvisor:v0.45.0
```

部署后浏览器访问 ip:8080端口

访问cadvisor的metrics页面，显示采集容器的指标数据：http://192.168.9.31:8080/metrics

### daemonset部署

```yaml
cat >daemonset-cadvisor.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cadvisor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: cAdvisor
  template:
    metadata:
      labels:
        app: cAdvisor
    spec:
      tolerations:    #master节点的污点容忍，忽略NoSchedule
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
      hostNetwork: true       #使用宿主机网络
      restartPolicy: Always   #重启策略
      containers:
      - name: cadvisor
        image: registry.cn-guangzhou.aliyuncs.com/any-any/cadvisor:v0.45.0
        imagePullPolicy: IfNotPresent  #镜像拉取策略
        ports:
        - containerPort: 8080
        volumeMounts:
          - name: root
            mountPath: /rootfs
          - name: run
            mountPath: /var/run
          - name: sys
            mountPath: /sys
          - name: containerd
            mountPath: /var/lib/containerd   #对应宿主机容器的存储目录，docker则为/var/lib/docker
          - name: disk
            mountPath: /dev/disk/
      volumes:
      - name: root
        hostPath:
          path: /
      - name: run
        hostPath:
          path: /var/run
      - name: sys
        hostPath:
          path: /sys
      - name: disk
        hostPath:
          path: /dev/disk/
      - name: containerd
        hostPath:
          path: /var/lib/containerd   #对应宿主机容器的存储目录，docker则为/var/lib/docker
EOF

```

部署后查看每个节点的pod是否成功创建，当前pod使用的是hostNetwork网络，因此端口直接暴露在宿主机网络

访问各个节点ip:8080端口，验证cadvisor服务

查看metrics指标数据 URI:/metrics



# 监控流程

![image-20240325171250918](./images/image-20240325171250918-1711357984634-1.png)

## 监控kube-scheduler

安装完之后并没有对 kube-controller-manager 和 kube-scheduler 这两个系统组件进行监控

配置监控

- 修改的kube-scheduler的yaml文件，kubeadm安装的一般都在目录下： /etc/kubernetes/manifests/kube-scheduler.yaml

  这里修改之后自动生效，内部会自动删除重启pod

```bash
# vim /etc/kubernetes/manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=0.0.0.0        # 原来为127.0.0.1，修改为0.0.0.0，让所有ip可连
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
...
```

- 配置servicemonitor资源文件

```yaml
cat > kube-scheduler-servicemonitor.yaml << EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token    # 用户证书文件，所有pod都在这个目录下
    interval: 30s                   # 扫描事件间隔
    path: /metrics
    port: metrics                   # 对应的service的端口名称
    scheme: https                   # https连接
    tlsConfig:
      insecureSkipVerify: true      # 禁止验证证书信息
  jobLabel: app.kubernetes.io/name
  namespaceSelector:
    matchNames:
    - kube-system                  # 需要匹配的service所在命名空间
  selector:
    matchLabels:
      app: kube-scheduler          # 需要匹配的service的标签
EOF
```

- 配置service

查看kube-scheduler的ip

```bash
[root@k8s-master1 prometheus]# kubectl get po -n kube-system -owide
NAME              READY   STATUS      RESTARTS   AGE
...
kube-scheduler    1/1     Running     0          74m     192.168.9.31     k8s-node1     <none>           
```

查看当前kube-scheduler启用的端口

```bash
# netstat -lntup |grep kube-schedul 
tcp6       0      0 :::10259                :::*                    LISTEN      123333/kube-schedul 
```

service和endpoint的yaml文件

```yaml
cat > kube-scheduler-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    app: kube-scheduler
spec:
  ports:
  - name: metrics
    port: 10259
    targetPort: 10259
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: kube-scheduler
  name: kube-scheduler
  namespace: kube-system
subsets:
  - addresses:
      - ip: 192.168.9.30 
    ports:
    - name: metrics
      port: 10259
      protocol: TCP
EOF

```



## 监控kube-control-manager

修改yaml文件：/etc/kubernetes/manifests/kube-controller-manager.yaml

修改之后会自动删除重启pod生效

```bash
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=0.0.0.0        # 原来为127.0.0.1，修改为0.0.0.0，让所有ip可连
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    ...
```

从配置文件中看开放的端口为：10257

也可以使用命令查看：

```bash
$ netstat -lntup|grep controll
tcp6       0      0 :::10257                :::*                    LISTEN      20384/kube-controll 
```

创建endpoint的yaml文件，与上面的kube-scheduler相似，直接参考修改即可



## 监控自建etcd集群

现在关键的服务基本都会留有指标metrics接口支持prometheus的监控

- 查看ERCD暴露得监控指标

```bash
# 替换为自己的证书以及ip端口
curl --cacert /opt/etcd/ssl/ca.pem --cert /opt/etcd/ssl/server.pem  --key /opt/etcd/ssl/server-key.pem https://192.168.9.30:2379/metrics

```

- 配置使ETCD能被prometheus发现并监控

```bash
# 首先把ETCD的证书创建为secret
kubectl -n monitoring create secret generic etcd-certs --from-file=/opt/etcd/ssl/server.pem   --from-file=/opt/etcd/ssl/server-key.pem   --from-file=/opt/etcd/ssl/ca.pem


# 接着在prometheus里面引用这个secrets
kubectl -n monitoring edit prometheus k8s
...
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: alertmanager-main
      namespace: monitoring
      port: web
    secrets:
    - etcd-certs

# 保存退出后，prometheus会自动重启服务pod以加载这个secret配置，过一会，进pod来查看下是不是已经加载到ETCD的证书
kubectl -n monitoring exec -it prometheus-k8s-0 -c prometheus  -- sh 
/prometheus $ ls /etc/prometheus/secrets/etcd-certs/
ca.pem        etcd-key.pem  etcd.pem
```

- 接下来准备创建service、endpoints以及ServiceMonitor的yaml配置

  注意替换下面的node节点IP为实际ETCD所在node内网IP

```bash
# vim prometheus-etcd.yaml 
apiVersion: v1
kind: Service
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: api
    port: 2379
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd
subsets:
- addresses:
  - ip: 10.0.1.201
  - ip: 10.0.1.202
  - ip: 10.0.1.203
  ports:
  - name: api
    port: 2379
    protocol: TCP

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd-k8s
spec:
  jobLabel: k8s-app
  endpoints:
  - port: api            # service中的port的name名称
    interval: 30s        # serviceMonitor的扫描时间间隔
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.pem
      certFile: /etc/prometheus/secrets/etcd-certs/server.pem
      keyFile: /etc/prometheus/secrets/etcd-certs/server-key.pem
      #use insecureSkipVerify only if you cannot use a Subject Alternative Name
      insecureSkipVerify: true 
  selector:
    matchLabels:
      k8s-app: etcd    # 匹配service标签
  namespaceSelector:
    matchNames:
    - monitoring   # 匹配命名空间

```

- 创建资源

```bash
kubectl apply -f prometheus-etcd.yaml 
service/etcd-k8s created
endpoints/etcd-k8s created
servicemonitor.monitoring.coreos.com/etcd-k8s created

```

- 创建好之后在prometheus UI上面查看到ETCD集群是否被监控

![image-20240325174514904](./images/image-20240325174514904.png)

- 配置grafana来展示被监控的ETCD指标

![image-20240325174912659](./images/image-20240325174912659.png)

- 在grafana官网模板中心搜索etcd，下载这个json格式的模板文件 https://grafana.com/grafana/dashboards/3070-etcd/

- 导入下载的模板

![image-20240325175402140](./images/image-20240325175402140.png)



## 监控ingress-nginx-controller

作为整个K8s上所有服务的流量入口组件很关键，把它的metrics指标收集到prometheus来做好相关监控至关重要，因为前面ingress-nginx服务是以daemonset形式部署的，并且映射了自己的端口到宿主机上，那么我可以直接用pod运行NODE上的IP来看下metrics

```bash
# 查看ingress的svc（我这里安装在ingress-nginx空间下，并且是nodePort形式）
# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                      AGE
ingress-nginx-controller             NodePort    10.98.117.69    <none>        80:31228/TCP,443:31992/TCP,10254:31229/TCP   5d
ingress-nginx-controller-admission   ClusterIP   10.105.135.79   <none>        443/TCP                                      5d
```

- 修改ingress-nginx-controller，开启metrics指标（根据业务需要，某些公司需求是不开启监控指标的）

```bash
# 开启metrics指标
kubectl edit deploy -n ingress-nginx ingress-nginx-controller
#找到参数配置：
...
   spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        - --enable-metrics=true  # 添加参数，并置为true，如果已存在直接修改
...

```

- 检查自己的ingress是否有开启10254端口（默认），没有的话，需要开启此端口，来进行指标获取

```yaml
kubectl edit svc -n ingress-nginx ingress-nginx-controller
...
spec:
  clusterIP: 10.98.117.69
  clusterIPs:
  - 10.98.117.69
  externalTrafficPolicy: Local
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    nodePort: 31228
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    nodePort: 31992
    port: 443
    protocol: TCP
    targetPort: https
  - name: metrics         # 如果没有开启此端口，需要开启
    nodePort: 31229
    port: 10254
    protocol: TCP
    targetPort: 10254     # 这里的svc是nodePort模式，可以指定开放的端口，如果不需要指定或者是其他模式，删除此参数
...


# 修改之后再次查看svc，可看到开放了10254端口，
# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                      AGE
ingress-nginx-controller             NodePort    10.98.117.69    <none>        80:31228/TCP,443:31992/TCP,10254:31229/TCP   5d
ingress-nginx-controller-admission   ClusterIP   10.105.135.79   <none>        443/TCP                                      5d

# 测试获取指标
curl http://10.98.117.69:10254/metrics
```

- `prometheus-k8s`这个`ServiceAccount`在`monitoring`命名空间中没有足够的权限去列表（list）`ingress-nginx`命名空间中的`Pod`资源。在该ns下创建一个role绑定到prometheus-k8s上，让其拥有权限

```yaml
cat >ingress-nginx-Role.yaml << EOF
# 在对应的ns中创建角色
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prometheus-k8s
  namespace: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
---
# 绑定角色 prometheus-k8s 角色到 Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prometheus-k8s
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s   # Prometheus 容器使用的 serviceAccount，kube-prometheus默认使用prometheus-k8s这个用户
  namespace: monitoring
EOF

```

- 创建servicemonitor配置让prometheus发现ingress-nginx的metrics

```bash
cat >ingress-nginx-servicemonitor.yaml << EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: ingress-nginx
  name: nginx-ingress-scraping
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    path: /metrics
    port: metrics         # service中的port的name名称
  jobLabel: app.kubernetes.io/name
  namespaceSelector:
    matchNames:
    - ingress-nginx       # ingress-nginx的命名空间
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx        # 匹配ingress-nginx-controller 的svc的标签
EOF

```

如果不创建role绑定到prometheus-k8s用户上，monitoring下的pod：prometheus-k8s 会报错:

```bash 
ts=2024-03-26T10:18:47.042Z caller=klog.go:116 level=error component=k8s_client_runtime func=ErrorDepth msg="pkg/mod/k8s.io/client-go@v0.27.3/tools/cache/reflector.go:231: Failed to watch *v1.Pod: failed to list *v1.Pod: pods is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list resource \"pods\" in API group \"\" in the namespace \"ingress-nginx\""
```

- 过一会儿在prometheus的UI上就会看到监控到了：serviceMonitor/monitoring/nginx-ingress-scraping/0 (2/2 up)
- 下载grafana模板，导入使用

https://grafana.com/grafana/dashboards/14314-kubernetes-nginx-ingress-controller-nextgen-devops-nirvana/



# prometheus自动发现

取自博客：https://blog.csdn.net/2201_75761617/article/details/131750800

Prometheus Server的数据抓取工作于Pull模型，因而，它必需要事先知道各Target的位置，然后才能从相应的Exporter或Instrumentation中抓取数据，获取数据源target的方式主要有两种模式：**静态配置**与**动态服务发现配置**

**静态配置：**

​	promethues的静态服务发现static_configs或者是ServiceMonitor 通过标签匹配Service：每当有一个新的目标实例需要监控，都需要手动修改配置文件配置目标target或者修改ServiceMonitor CRD配置文件。

**动态配置：**

​	现如今动不动就成百上千台节点的集群来说，静态服务发现这种纯手动配置很显然是不切实际的，然而现在的集群往往都有一个很重要的功能叫做服务发现，例如在现在常用的微服务SpringCloud架构的中，Eureka组件作为服务注册和发现的中心，在Kubernetes这类容器管理平台中，Kubernetes也具有服务发现的功能，它们都掌握并管理着所有的容器或者是服务的相关信息，于是Prometheus通过这个中间的代理人（服务发现和注册中心）来获取集群当中监控目标的信息，从而巧妙地实现了prometheus的动态服务发现。


常用的动态服务发现：

- **基于k8s的服务发现kubernetes_sd_configs**
- **基于consul的服务发现consul_sd_config**
- **基于Eureka的服务发现eureka_sd_config**

**Prometheus指标抓取的生命周期：**

​	发现 -> 配置 -> relabel -> 指标数据抓取 -> metrics relabel

- 在每个scrape_interval期间，Prometheus都会检查执行的作业（Job）；
- 这些作业首先会根据Job上指定的发现配置生成target列表，此即服务发现过程；
- 服务发现会返回一个Target列表，其中包含一组称为元数据的标签，这些标签都以“_*meta*”为前缀；
- 服务发现还会根据目标配置来设置其它标签，这些标签带有“__”前缀和后缀，包括“**scheme**”、 “**address**”和“**metrics_path**”，分别保存有target支持使用协议(http或https，默认为http）、target的地址及指标的URI路径（默认为/metrics）；
- 若URI路径中存在任何参数，则它们的前缀会设置为“_*param*；
- 配置标签会在抓取的生命周期中被重复利用以生成其他标签，例如，指标上的instance标签的默认值就来自于__address__标签的值；
- 抓取而来的指标在保存之前，还允许用户对指标重新打标并过滤，在job段metric_relabel_configs配置，通常用来删除不需要的指标、删除敏感或不必要的标签和添加修改标签格式等。

## kubernetes_sd_configs

​	在Kubernetes下，Prometheus 通过与 Kubernetes API 集成主要支持5种服务发现模式又叫角色role：Node、Service、Pod、Endpoints、Ingress。不同的服务发现模式适用于不同的场景，例如：node适用于与主机相关的监控资源，如节点中运行的Kubernetes 组件状态、节点上运行的容器状态等；service 和 ingress 适用于通过黑盒监控的场景，如对服务的可用性以及服务质量的监控；endpoints 和 pod 均可用于获取 Pod 实例的监控数据，如监控用户或者管理员部署的支持 Prometheus 的应用。

Prometheus-additional.yaml配置文件规则详解
	为解决服务发现的问题，kube-prometheus 为我们提供了一个额外的抓取配置来解决这个问题，我们可以通过添加额外的配置来进行服务发现进行自动监控。我们可以在 kube-prometheus 当中去自动发现并监控具有 prometheus.io/scrape=true 这个 annotations 的 Service。
	其中通过 kubernetes_sd_configs 支持监控其各种资源。kubernetes SD 配置允许从 kubernetes REST API 接受搜集指标，且总是和集群保持同步状态，任何一种 role 类型都能够配置来发现我们想要的对象。

规则配置使用 yaml 格式，下面是文件中一级配置项。自动发现 k8s Metrics 接口是通过 scrape_configs 来实现:

```bash

# Kubernetes的API SERVER会暴露API服务，Promethues集成了对Kubernetes的自动发现，它有5种模式：Node、Service
# 、Pod、Endpoints、ingress，下面是Prometheus官方给出的对Kubernetes服务发现的实例。这里你会看到大量的relabel_configs，
# 其实你就是把所有的relabel_configs去掉一样可以对kubernetes做服务发现。relabel_configs仅仅是对采集过来的指标做二次处理，比如
# 要什么不要什么以及替换什么等等。而以__meta_开头的这些元数据标签都是实例中包含的，而relabel则是动态的修改、覆盖、添加删除这些标签
# 或者这些标签对应的值。而且以__开头的标签通常是系统内部使用的，因此这些标签不会被写入样本数据中，如果我们要收集这些东西那么则要进行
# relabel操作。当然reabel操作也不仅限于操作__开头的标签。
#
# action的行为：
# replace：默认行为，不配置action的话就采用这种行为，它会根据regex来去匹配source_labels标签上的值，并将并将匹配到的值写入target_label中
# labelmap：它会根据regex去匹配标签名称，并将匹配到的内容作为新标签的名称，其值作为新标签的值
# keep：仅收集匹配到regex的源标签，而会丢弃没有匹配到的所有标签，用于选择
# drop：丢弃匹配到regex的源标签，而会收集没有匹配到的所有标签，用于排除
# labeldrop：使用regex匹配标签，符合regex规则的标签将从target实例中移除，其实也就是不收集不保存
# labelkeep：使用regex匹配标签，仅收集符合regex规则的标签，不符合的不收集
 
```

配置以及解释yaml

```yaml
global:
  # 间隔时间
  scrape_interval: 30s
  # 超时时间
  scrape_timeout: 10s
  # 另一个独立的规则周期，对告警规则做定期计算
  evaluation_interval: 30s
  # 外部系统标签
  external_labels:
	prometheus: monitoring/k8s
	prometheus_replica: prometheus-k8s-1
 
# 抓取服务端点，整个这个任务都是用来发现node-exporter和kube-state-metrics-service的，这里用的是endpoints角色，这是通过这两者的service来发现后端endpoints。另外需要说明的是如果满足采集条件，那么在service、POD中定义的labels也会被采集进去
scrape_configs: 
  # 定义job名称，是一个拉取单元 
- job_name: "kubernetes-endpoints"
  # 发现endpoints，它是从列出的服务端点发现目标，这个endpoints来自于Kubernetes中的service，每一个service都有对应的endpoints，这里是一个列表
  # 可以是一个IP:PORT也可以是多个，这些IP:PORT就是service通过标签选择器选择的POD的IP和端口。所以endpoints角色就是用来发现server对应的pod的IP的
  # kubernetes会有一个默认的service，通过找到这个service的endpoints就找到了api server的IP:PORT，那endpoints有很多，我怎么知道哪个是api server呢
  # 这个就靠source_labels指定的标签名称了。
  kubernetes_sd_configs:
	# 角色为 endpoints
	- role: endpoints
 
  relabel_configs:
    # 重新打标仅抓取到的具有 "prometheus.io/scrape: true" 的annotation的端点，意思是说如果某个service的annotation具有prometheus.io/scrape: true 声明的则会被抓取
    # annotation本身也是键值结构，所以这里的源标签设置为键，而regex设置值，当值匹配到regex设定的内容时则执行keep动作也就是保留，其余则丢弃.
    # node-exporter这个POD的service里面就有一个叫做prometheus.io/scrape: true的annotations所以就找到了node-exporter这个POD
	- source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
	  # 动作: 删除 regex 与串联不匹配的目标 source_labels
	  action: keep
	  # 通过正式表达式匹配 true
	  regex: true
	# 重新设置scheme
    # 匹配源标签__meta_kubernetes_service_annotation_prometheus_io_scheme也就是annotation中的prometheus.io/scheme字段，如果源标签的值匹配到regex则把值替换为__scheme__对应的值
	- source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
	  action: replace
	  target_label: __scheme__
	  regex: (https?)
	
	# 匹配来自 pod的annotationname prometheus.io/path 字段
	- source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
	  # 获取POD的 annotation 中定义的"prometheus.io/path: XXX"定义的值，这个值就是你的程序暴露符合prometheus规范的metrics的地址，如果你的metrics的地址不是 /metrics 的话，通过这个标签说，那么这里就会把这个值赋值给 __metrics_path__这个变量，因为prometheus是通过这个变量获取路径然后进行拼接出来一个完整的URL，并通过这个URL来获取metrics值的，因为prometheus默认使用的就是 http(s)://X.X.X.X/metrics 这样一个路径来获取的。
	  action: replace
	  # 匹配目标指标路径
	  target_label: __metrics_path__
	  # 匹配全路径
	  regex: (.+)
	
	# 匹配出 Pod ip地址和 Port
	- source_labels:
		[__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
	  action: replace
	  target_label: __address__
	  regex: ([^:]+)(?::d+)?;(d+)
	  replacement: $1:$2
	
	# 下面主要是为了给样本添加额外信息
	- action: labelmap
	  regex: __meta_kubernetes_service_label_(.+)
	# 元标签 服务对象的名称空间
	- source_labels: [__meta_kubernetes_namespace]
	  action: replace
	  target_label: kubernetes_namespace
	# service 对象的名称
	- source_labels: [__meta_kubernetes_service_name]
	  action: replace
	  target_label: kubernetes_name
	# pod对象的名称
	- source_labels: [__meta_kubernetes_pod_name]
	  action: replace
	  target_label: kubernetes_pod_name
```

- **新增 prometheus 在 Kubernetes 下的自动服务发现pod的yaml文件**

  endppoint角色

```yaml
cat > prometheus-additional-endpoints.yaml << EOF
- job_name: 'kubernetes-endpoints'
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
    action: replace
    target_label: __scheme__
    regex: (https?)
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_name
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
EOF
```

- **创建Secret 对象**

将上面文件直接保存为 prometheus-additional.yaml，然后通过这个文件创建一个对应的 Secret 对象：

```bash
kubectl create secret generic additional-configs-endpoint --from-file=prometheus-additional-endpoints.yaml -n monitoring

```

- **创建资源对象**

然后我们需要在声明 prometheus 的资源对象文件（prometheus-prometheus.yaml）中通过 additionalScrapeConfigs 属性添加上这个额外的配置：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.46.0
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: alertmanager-main
      namespace: monitoring
      port: web
  enableFeatures: []
  externalLabels: {}
  image: docker.io/bogeit/prometheus:v2.46.0
  nodeSelector:
    kubernetes.io/os: linux
  podMetadata:
    labels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
      app.kubernetes.io/version: 2.46.0
  podMonitorNamespaceSelector: {}
  podMonitorSelector: {}
  probeNamespaceSelector: {}
  probeSelector: {}
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleNamespaceSelector: {}
  ruleSelector: {}
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: 2.46.0
  #以下为新增的额外配置项
  additionalScrapeConfigs:                 
	name: additional-endpoints
	key: prometheus-additional-endpoints.yaml
```

添加完成后，直接更新 prometheus 这个 CRD 资源对象：

```bash
kubectl apply -f prometheus-prometheus.yaml
```

等一下，在promethues UI界面刷新查看 config，将会查看配置已经生效。

- **创建RBAC权限**

kube-prometheus安装的Prometheus 绑定了一个名为 prometheus-k8s 的 ServiceAccount 对象，而这个对象绑定的是一个名为 prometheus-k8s 的 ClusterRole（原配置 > prometheus-clusterRole.yaml）：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.46.0
  name: prometheus-k8s
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
```

上面的权限规则中我们可以看到明显没有对 Service 或者 Pod 的 list 权限，所以报错了，要解决这个问题，我们只需要添加上需要的权限即可：

```yaml
cat > prometheus-clusterRole-new.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.46.0
  name: prometheus-k8s
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  - /actuator/prometheus
  verbs:
  - get
EOF

```

更新资源对象：

```bash
kubectl apply -f prometheus-clusterRole-new.yaml
```

- **自动发现**

自动发现规则配置好后，看好自己所配置的role角色

​	如果是pod，需要在deployments中的spec.template.metadata.annotations中添加注解配置

​	如果是endpoints，需要在service中添加注解配置

配置的内容如下：

```bash
annotations:
	prometheus.io/path: /actuator/prometheus   # 指标路径地址，不配置，默认就是:/metrics
	prometheus.io/port: "7070"
	prometheus.io/scheme: http
	prometheus.io/scrape: "true"
```

定义好后prometheus即可抓取指定资源内的metrics指标数据了，在prometheus的targets页面即可看到job名称为 kubernetes-endpoints 的target。



## consul_sd_configs



​	由 HashiCorp 开发的一个支持多数据中心的分布式服务发现和键值对存储服务的开源软件，是一个通用的服务发现和注册中心工具，被大量应用于基于微服务的软件架构当中。

通过api将exporter服务注册到 Consul 然后配置 Prometheus 从Consul 中发现实例。

架构图

<img src="./images/image-20240403102117619.png" alt="image-20240403102117619" style="zoom:80%;" />

**1、安装：**

- **二进制**

下载符合自己系统的安装文件：https://www.consul.io/downloads

```bash
wget https://releases.hashicorp.com/consul/1.14.5/consul_1.14.5_linux_amd64.zip
# yum -y install unzip
unzip consul_1.14.5_linux_amd64.zip
mv consul /usr/local/bin
consul version

```

启动consul，为了查看更多的日志信息，我们可以在dev模式下运行consul 如下所示：

```bash
consul agent -dev -client 0.0.0.0
```

启动命令后面使用的client 参数指定了客户端绑定的IP地址 默认为127.0.0.1

- **docker安装**

```bash
docker run -d --name consul -p 8500:8500 consul
```

检查

```bash
docker ps
CONTAINER ID  IMAGE         COMMAND             CREATED         STATUS         PORTS          NAMES
5b219685e431 consul   "docker-entrypoint.s…"   18 seconds ago Up 17 seconds   8300-8302/tcp, 8301-8302/udp, 8600/tcp, 8600/udp, 0.0.0.0:8500->8500/tcp   consul

```

网页访问：8500端口

<img src="./images/image-20240403100202905.png" alt="image-20240403100202905" style="zoom:80%;" />



**2、注册到consul**

- 使用命令注册

二进制安装的可直接命令：

```bash
# 注册服务
consul services register -id=node-1 -name=node-1 -address=192.168.1.1 -port=9100 -tag=node_exporter
consul services register -id=node-2 -name=node-2 -address=192.168.1.2 -port=9100 -tag=node_exporter

# 注销服务
consul services deregister -id node-1
```

docker安装的可以发送put请求：

```bash
curl -X PUT -d '{"id": "node1","name": "node_exporter","address": "node_exporter","port": 9100,"tags": ["exporter"],"meta": {"job": "node_exporter","instance": "我的服务器"},"checks": [{"http": "http://127.0.0.1:9100/metrics", "interval": "5s"}]}' http://192.168.9.30:8500/v1/agent/service/register

```

命令解释：

```bash
-X PUT：使用 PUT 方法发送请求。在这个命令中，PUT 方法用于注册服务到 Consul。
-d '{"id": "node1","name": "node_exporter","address": "node_exporter","port": 9100,"tags": ["exporter"],"meta": {"job": "node_exporter","instance": "Prometheus服务器"},"checks": [{"http": "http://192.168.9.144:9100/metrics", "interval": "5s"}]}'：指定要发送的数据体，即要注册的服务的信息。这里使用 JSON 格式进行定义。
    "id": "node1"：服务的唯一标识符。
    "name": "node_exporter"：服务的名称。
    "address": "node_exporter"：服务的地址。
    "port": 9100：服务的端口号。
    "tags": ["exporter"]：服务的标签，可以是任意字符串数组。
    "meta": {"job": "node_exporter","instance": "Prometheus服务器"}：附加的元数据信息，可以是任意键值对。
    "checks": [{"http": "http://192.168.9.144:9100/metrics", "interval": "5s"}]：与服务相关的健康检查配置。这里定义了一个 HTTP 类型的检查，检查地址为 http://192.168.1.144:9100/metrics，每 5 秒执行一次。
http://localhost:8500/v1/agent/service/register：Consul Agent 的注册服务 API 端点。
```

- 将json数据放在文件中使用文件注册

```bash
mkdir /data/consul &&  cd /data/consul

vim node_exporter.json
{
  "id": "node2",
  "name": "node_exporter",
  "address": "192.168.9.140",
  "port": 9100,
  "tags": ["exporter"],
  "meta": {
    "job": "node_exporter",
    "instance": "test服务器"
  },
  "checks": [{
    "http": "http://192.168.9.140:9100/metrics",
    "interval": "10s"
  }]
}

```

```bash
# 使用json文件注册
curl --request PUT --data @node_exporter.json http://localhost:8500/v1/agent/service/register

# 注销
curl -X PUT http://localhost:8500/v1/agent/service/deregister/service_id
```

**注册完成后在consul页面刷新，就可以看到注册的实例**



**3、配置prometheus**

​	上面通过consul注册了服务，接下来将配置Prometheus通过consul来自动发现node_porter服务，在Prometheus的配置文件prometheus.yml文件中的scrape_configs 部分添加如下所示的抓取配置。

vim prometheus-additional.yaml

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    consul_sd_configs: 
    - server: 192.168.9.30:8500
      tags:
      - node_exporter
...
```

使用配置生效与上面的kubernetes_sd_configs的方法一样，重复操作即可。



# pushgateway

官方：https://github.com/prometheus/pushgateway

Prometheus Pushgateway 的存在是为了允许临时作业和批处理作业将其指标公开给 Prometheus。由于此类工作可能存在的时间不够长而无法被删除，因此他们可以将其指标推送到 Pushgateway。然后 Pushgateway 将这些指标公开给 Prometheus。

官方仅建议在某些有限的情况下使用 Pushgateway。盲目使用 Pushgateway 而不是 Prometheus 通常的拉取模型来进行一般指标收集会存在几个陷阱：

- 当通过单个 Pushgateway 监控多个实例时，Pushgateway 既成为单点故障又成为潜在瓶颈。
- 您将失去 Prometheus 通过`up` 指标（每次抓取时生成）的自动实例运行状况监控。
- Pushgateway 永远不会忘记推送给它的系列，并将它们永远暴露给 Prometheus，除非这些系列通过 Pushgateway 的 API 手动删除。

注意：

- 不支持带时间戳上报，会被忽略
- 当通过单个Pushgateway监视多个实例时，Pushgateway既成为单个故障点，又成为潜在的瓶颈。
- Prometheus为每个采集的target生成的up指标无法使用
- Pushgateway永远不会删除推送到其中的系列，除非通过Pushgateway的API手动删除了这些系列，否则它们将永远暴露给Prometheus

安装好之后，推送数据示例

使用prometheus python sdk向pushgateway推送数据

```python
# 安装sdk
pip3 install prometheus_client
```

```python
# 推送数据
# coding:utf-8
import time

from prometheus_client import CollectorRegistry, Gauge, push_to_gateway, Counter
import random

# 轮训推送间隔
# 初始化CollectorRegistry
r1 = CollectorRegistry()
# pushgateway api地址
push_addr = "1192.168.9.30:9091"

# 初始化一个gauge对象
# labels要在设置时指定好
# 指标类型 gauge,counter,histogram,Summary

g1 = Gauge('kubelet_abc', 'Description of gauge', ['k1', 'k2'], registry=r1)
c1 = Counter('apiserver_abcd', 'HTTP Request', ['method', 'endpoint'], registry=r1)

# 业务埋点
def collect():
    # 设置gauge
    randvalue = random.randint(1, 100)
    g1.labels(k1='v1', k2='v2').set(randvalue)
    c1.labels(method='get', endpoint='/login').inc(10)


if __name__ == '__main__':
    step = 10
    while 1:
        # try:
        collect()
        # 将registry中的数据推送到pushgateway
        # 需要job label
        # 最终调用 put 或post http://pushgateway_addr/metrics/job/some_job
        # put是匹配所有tag相同替换.post是metric_name相同替换
        # res= push_to_gateway(push_addr1, job='2020_09_21_asome_job', registry=r1,handler=custom_handle)
        res = push_to_gateway(push_addr, job='test_job', registry=r1)
        print(res)
        time.sleep(step)

```

将单个pushgateway加入prometheus采集job中

```yaml
  - job_name: 'pushgateway'
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: http
    static_configs:
    - targets:
      - 192.168.9.30:9091
      - 192.168.9.31:9091

```



# 监控第三方程序

这里只举例tomcat，其他例如redis、mysql、nginx等可参考博客：https://www.cnblogs.com/punchlinux/p/16856793.html

## tomcat

组件官网：https://github.com/nlighten/tomcat_exporter

下载所需的jar包：

![image-20240403140953604](./images/image-20240403140953604.png)

如果想下载以往版本解决兼容某些tomcat问题，则点击版本后进入详细版本页面进行下载

注意：根据测试，simpleclient程序包使用0.12.0开始以后版本不对8.5.x 的tomcat兼容，无法显示metrics页面，但tomcat-exporter程序包可以使用最新版本。

程序包列表

```bash
simpleclient-0.8.0.jar
simpleclient_common-0.8.0.jar
simpleclient_hotspot-0.8.0.jar
simpleclient_servlet-0.8.0.jar
tomcat_exporter_client-0.0.17.jar
tomcat_exporter_servlet-0.0.17.war
```

将tomcat_exporter_servlet-0.0.17.war修改为metrics进行tomcat页面发布

制作tomcat镜像，将包含metrics监控指标的jar包导入tomcat镜像内

```bash
cat > Dockerfile <<EOF
FROM harbor.cncf.net/web/tomcat:8.5.43
  
MAINTAINER ANY
 
LABEL Description="tomcat-8.5.43"
 
ADD metrics.war /usr/local/tomcat/webapps/
ADD simpleclient-0.8.0.jar /usr/local/tomcat/lib
ADD simpleclient_common-0.8.0.jar /usr/local/tomcat/lib
ADD simpleclient_hotspot-0.8.0.jar /usr/local/tomcat/lib
ADD simpleclient_servlet-0.8.0.jar /usr/local/tomcat/lib
ADD tomcat_exporter_client-0.0.17.jar /usr/local/tomcat/lib
 
EXPOSE 8080 8443
EOF

```

构建镜像

```bash
docker build -t tomcat-app:v1
# 打标签，推送至仓库
docker tag tomcat-app:v1 registry.cn-guangzhou.aliyuncs.com/any-any/tomcat:v1
docker push registry.cn-guangzhou.aliyuncs.com/any-any/tomcat:v1
```

编写tomcatyaml文件

```yaml
cat > tomcat-deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: default
spec:
  selector: 
    matchLabels: 
     app: tomcat
  replicas: 1 
  template: 
    metadata:
      labels:
        app: tomcat
      annotations:
        prometheus.io/scrape: "true"     #添加prometheus服务发现的抓取注解
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: tomcat
        image: registry.cn-guangzhou.aliyuncs.com/any-any/tomcat:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        securityContext: 
          privileged: true             #给POD privileged权限
EOF
```

创建service

```yaml
cat >tomcat-svc.yaml <<EOF
kind: Service 
apiVersion: v1
metadata:
  annotations:
    prometheus.io/scrape: "true"   #添加prometheus服务发现的抓取注解
    prometheus.io/port: "8080"
  name: tomcat-service
spec:
  selector:
    app: tomcat
  ports:
  - nodePort: 31080
    port: 80
    protocol: TCP
    targetPort: 8080
  type: NodePort
EOF

```

手动配置静态发现（可选，根据自己需要）

```bash
vim prometheus-additional.yml
- job_name: "tomcat"
    static_configs:
      - targets: ["192.168.9.30:31080"]
```



部署导入grafana官方模板：https://github.com/nlighten/tomcat_exporter/tree/master/dashboard



# 监控组件blackbox-exporter

## **安装**

官方下载地址：https://prometheus.io/download/#blackbox_exporter

 优秀博客：https://www.cnblogs.com/panwenbin-logs/p/18454188

prometheus官方提供的一个exporter，可以监控http，https，DNS，TCP，ICMP等目标实例，从而实现对被监控节点进行监和控数据采集（blackbox_exporter比较特殊，它的监控对象需要由prometheus提供）

- HTTP/HTTPS: URL/API可用性
- TCP：监听端口
- ICMP：主机存活性检测
- DNS：域名可用性

![image-20240403151328661](./images/image-20240403151328661.png)



```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz 

tar xvf blackbox_exporter-0.24.0.linux-amd64.tar.gz -C /opt/
cd /opt && mv blackbox_exporter-0.24.0.linux-amd64 blackbox_exporter-0.24.0 && cd blackbox_exporter-0.24.0
```

配置文件模块：

```bash
# cat blackbox.yml 
modules:
  http_2xx:
    prober: http
    http:
      preferred_ip_protocol: "ip4"
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  grpc:
    prober: grpc
    grpc:
      tls: true
      preferred_ip_protocol: "ip4"
  grpc_plain:
    prober: grpc
    grpc:
      tls: false
      service: "service1"
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
      - send: "SSH-2.0-blackbox-ssh-check"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
  icmp_ttl5:
    prober: icmp
    timeout: 5s
    icmp:
      ttl: 5

```

创建blackbox_exporter service启动文件，默认监听9115端口

```bash
cat >/etc/systemd/system/blackbox-exporter.service << EOF
[Unit]
Description=Prometheus Blackbox Exporter
After=network.target
 
 
[Service]
Type=simple
User=root
Group=root
ExecStart=/opt/blackbox_exporter-0.24.0/blackbox_exporter \
  --config.file=/opt/blackbox_exporter-0.24.0/blackbox.yml \
  --web.listen-address=:9115 
 
Restart=on-failure 
 
[Install] 
WantedBy=multi-user.target 
EOF

```

启动

```bash
systemctl daemon-reload && systemctl enable blackbox-exporter.service --now && systemctl status blackbox-exporter
```

访问web界面 IP:9115

访问指标数据：IP:9115/metrics

## url监控

Prometheus URL 监控配置：

```bash
- job_name: 'http_status'      
    metrics_path: /probe    # blackbox抓取数据数据之后存放的位置
    params:
      module: [http_2xx]    # 指定使用的模块
    static_configs:
      - targets: ['http://www.baidu.com']  #可以指定多个target，以逗号分隔
        labels:
          instance: http_status
          group: web
    relabel_configs:
      - source_labels: [__address__]  #将__address__(当前监控目标URL地址的标签)修改为__param_target,用于传递给blackbox_exporter
        target_label: __param_target  #标签key为__param_target、value为 www.baidu.com。
      - source_labels: [__param_target]    #基于__param_target 获取监控目标
        target_label: url
      - target_label: __address__  #新添加一个目标__address__，指向blackbox_exporter 服务器地址，用于将监控请求发送给指定的 blackbox_exporter 服务器
        replacement: 192.168.9.30:9115  #指定 blackbox_exporter 服务器地址
```

在prometheus web端访问验证，查看任务状态

## **ICMP(ping)监控**

Prometheus icmp 监控配置:

```bash
- job_name: 'ping_status'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets: ['192.168.9.30',"192.168.9.31"]
        labels:
          instance: 'ping_status'
          group: 'icmp'
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: ip
    - target_label: __address__
      replacement: 192.168.9.30:9115
```

## 端口监控

prometheus端口监控配置

```bash
- job_name: 'port_status'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
    - targets: ['192.168.9.30:9100', '192.168.9.30:9090','192.168.9.31:22'] #监控对端服务器的端口
      labels:
        instance: 'port_status'           
        group: 'port'       
    relabel_configs:  
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: ip
      - target_label: __address__
        replacement: 192.168.9.30:9115   #blackbox_exporter地址
```



# Adapter + HPA

参考博客：

https://www.jianshu.com/p/76885c56c387

https://blog.csdn.net/lo085213/article/details/111567974

k8s原生的自动扩缩容，支持的指标只有cpu和内存，但我们业务的场景可能使用的不仅限于cpu和内存，或者，cpu和内存不能满足业务扩缩容的需求，这样的话，就需要使用自定义指标的HPA来进行实现，例如，客户端提交的计算任务，服务端使用生产者消费者模式，将客户端的计算任务放在队列中，这个时候，监听队列的积压，从而进行扩缩容是一个相对于cpu，内存来说更优的方案。

如果要使用service相关的数据，如http get请求数之类的信息，需要查阅外部指标相关的API，k8s也同样支持，这里只关注custom metrics api指标

<img src="./images/image-20240520143913686.png" alt="image-20240520143913686" style="zoom:67%;" />

`Prometheus Adapter` 可以帮我们使用 Prometheus 收集的指标并使用它们来制定扩展策略，这些指标都是通过 APIServer 暴露的，而且 HPA 资源对象也可以很轻易的直接使用。

<img src="./images/image-20240520142417394.png" alt="image-20240520142417394" style="zoom:67%;" />

整体的搭建流程

1. 部署prometheus在k8s上
2. 部署service
3. 在prometheus上配置服务的serviceMonitor
4. 部署prometheus adapter在k8s上
5. 配置prometheus的采集规则rule
6. 为service添加HPA配置
7. 验证HPA是否生效

adapter采集prometheus数据的规则

```bash
# 执行命令进行修改，保存之后 adapter会热更新该配置项
kubectl edit configmap ops-prometheus-adapter -n ops 
```

在rule的数组中添加规则，示例规则

```yaml
rules:
- seriesQuery: 'nginx_vts_server_requests_total'
  seriesFilters: []
  resources:
    overrides:
      kubernetes_namespace:
        resource: namespace
      kubernetes_pod_name:
        resource: pod
  name:
    matches: "^(.*)_total"
    as: "${1}_per_second"
  metricsQuery: (sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>))
```

参数解释：

- seriesQuery：查询 Prometheus 的语句，通过这个查询语句查询到的所有指标都可以用于 HPA

- seriesFilters：查询到的指标可能会存在不需要的，可以通过它过滤掉。

- resources：通过 seriesQuery 查询到的只是指标，如果需要查询某个 Pod 的指标，肯定要将它的名称和所在的命名空间作为指标的标签进行查询，resources 就是将指标的标签和 k8s 的资源类型关联起来，最常用的就是 pod 和 namespace。有两种添加标签的方式，一种是 overrides，另一种是 template。

  ```bash
  overrides：它会将指标中的标签和 k8s 资源关联起来。上面示例中就是将指标中的 pod 和 namespace 标签和 k8s 中的 pod 和 namespace 关联起来，因为 pod 和 namespace 都属于核心 api 组，所以不需要指定 api 组。当我们查询某个 pod 的指标时，它会自动将 pod 的名称和名称空间作为标签加入到查询条件中。比如 nginx: {group: "apps", resource: "deployment"} 这么写表示的就是将指标中 nginx 这个标签和 apps 这个 api 组中的 deployment 资源关联起来；
  template：通过 go 模板的形式。比如template: "kube_<<.Group>>_<<.Resource>>" 这么写表示，假如 <<.Group>> 为 apps，<<.Resource>> 为 deployment，那么它就是将指标中 kube_apps_deployment 标签和 deployment 资源关联起来。
  ```

- name：用来给指标重命名的，之所以要给指标重命名是因为有些指标是只增的，比如以 total 结尾的指标。这些指标拿来做 HPA 是没有意义的，我们一般计算它的速率，以速率作为值，那么此时的名称就不能以 total 结尾了，所以要进行重命名。

  ```bash
  matches：通过正则表达式来匹配指标名，可以进行分组
  as：默认值为 $1，也就是第一个分组。as 为空就是使用默认值的意思。
  ```

- metricsQuery：Prometheus 的查询语句，前面的 `seriesQuery` 查询是获得 HPA 指标。当要查某个指标的值时就要通过它指定的查询语句进行了。可以看到查询语句使用了速率和分组，这就是解决上面提到的只增指标的问题。

  ```bash
  Series：表示指标名称
  LabelMatchers：附加的标签，目前只有 pod 和 namespace 两种，因此我们要在之前使用 resources 进行关联
  GroupBy：就是 pod 名称，同样需要使用 resources 进行关联。
  ```


# alertmanager告警

触发一条告警的过程

prometheus ---> 指标触发rule中的阈值 ---> 超出持续时间 ---> alertmanager ---> 分组|抑制|静默 ---> 媒体类型 ---> 邮件|钉钉|微信等

```bash
分组(group)： 将类似性质的报警发送给指定的收件人，比如网络通知发送给网络工程师、数据库通知发送给数据库工程师
静默(sliences)：简单的特定时间静音的机制，例如：服务器要升级维护可以先设置这个时间段的告警静默
抑制(inhibition)：当警报发出之后，停止重复发送由此警报引发的其他警报，即合并成一个故障引起的多个报警事件，可以消除冗余告警
```

<img src="./images/image-20240407111713547.png" alt="image-20240407111713547" style="zoom: 50%;" />

Prometheus的警报有如下几种状态：

- inactive ：警报未被触发。

- Pending：警报已被触发，但还未满足for参数定义的持续时间。

- Firing：警报被触发警，并满足for定义的持续时间的

将alertmanager集成到prometheus中，分为两步：

1. 新建或更新告警规则文件rules.yml
2. 更新prometheus.yml新增alertmanager相关配置

配置文件：在kube-prometheus中，配置文件写为了一个secret

```bash
# 通过这里可以获取需要创建的报警配置secret名称
# kubectl -n monitoring edit statefulsets.apps alertmanager-main
...
      volumes:
      - name: config-volume
        secret:
          defaultMode: 420
          secretName: alertmanager-main-generated
...

# 这里实际使用的secret为：alertmanager-main，配置文件修改
# 注意事先在配置文件 alertmanager.yaml 里面编辑好收件人等信息 ，再执行下面的命令
kubectl -n monitoring delete secret alertmanager-main
kubectl -n monitoring create secret generic  alertmanager-main --from-file=alertmanager.yaml 

```

alertmanager.yaml：配置文件参考解释说明

```yaml
cat > alertmanager.yaml << EOF
# global块配置下的配置选项在本配置文件内的所有配置项下可见
global:
  # 在Alertmanager内管理的每一条告警均有两种状态: "resolved"或者"firing". 在altermanager首次发送告警通知后, 该告警会一直处于firing状态,设置resolve_timeout可以指定处于firing状态的告警间隔多长时间会被设置为resolved状态, 在设置为resolved状态的告警后,altermanager不会再发送firing的告警通知.
  resolve_timeout: 10m  
  # 邮箱配置
  smtp_smarthost: 'smtp.163.com:465'
  smtp_from: 'any@163.com'
  smtp_auth_username: 'any@163.com'
  smtp_auth_password: 'XXXXXXXX'  # 授权码：XXXXXXXX
  smtp_hello: '@163.com'
  smtp_require_tls: false


  # 告警通知模板
templates:
- '/etc/alertmanager/config/*.tmpl'

# route: 根路由,该模块用于该根路由下的节点及子路由routes的定义. 子树节点如果不对相关配置进行配置，则默认会从父路由树继承该配置选项。每一条告警都要进入route，即要求配置选项group_by的值能够匹配到每一条告警的至少一个labelkey(即通过POST请求向altermanager服务接口所发送告警的labels项所携带的<labelname>)，告警进入到route后，将会根据子路由routes节点中的配置项match_re或者match来确定能进入该子路由节点的告警(由在match_re或者match下配置的labelkey: labelvalue是否为告警labels的子集决定，是的话则会进入该子路由节点，否则不能接收进入该子路由节点).
route:
  # 例如所有labelkey:labelvalue含cluster=A及altertname=LatencyHigh labelkey的告警都会被归入单一组中
  group_by: ['job', 'altername', 'cluster', 'service','severity']
  # 若一组新的告警产生，则会等group_wait后再发送通知，该功能主要用于当告警在很短时间内接连产生时，在group_wait内合并为单一的告警后再发送
  group_wait: 10s
  # 再次告警时间间隔
  group_interval: 20s
  # 如果一条告警通知已成功发送，且在间隔repeat_interval后，该告警仍然未被设置为resolved，则会再次发送该告警通知
  repeat_interval: 1m
  # 默认告警通知接收者，凡未被匹配进入各子路由节点的告警均被发送到此接收者
  receiver: 'webhook'
  # 上述route的配置会被传递给子路由节点，子路由节点进行重新配置才会被覆盖

  # 子路由树
  routes:
  # 该配置选项使用正则表达式来匹配告警的labels，以确定能否进入该子路由树
  # match_re和match均用于匹配labelkey为service,labelvalue分别为指定值的告警，被匹配到的告警会将通知发送到对应的receiver
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: 'webhook'
    # 在带有service标签的告警同时有severity标签时，他可以有自己的子路由，同时具有severity != critical的告警则被发送给接收者team-ops-wechat,对severity == critical的告警则被发送到对应的接收者即team-ops-pager
    routes:
    - match:
        severity: critical
      receiver: 'webhook'
  # 比如关于数据库服务的告警，如果子路由没有匹配到相应的owner标签，则都默认由team-DB-pager接收
  - match:
      service: database
    receiver: 'webhook'
  # 我们也可以先根据标签service:database将数据库服务告警过滤出来，然后进一步将所有同时带labelkey为database
  - match:
      severity: critical
    receiver: 'webhook'

# 抑制规则配置，当出现critical告警时 忽略warning
inhibit_rules:
- source_match:      # 资源匹配级别，当匹配成功发出通知，但其他的alertmanage产生的warning级别的告警通知将被抑制
    severity: 'critical'    # 报警的事件级别
  target_match:    
    severity: 'warning'    # 调用source_match的severity，即如果已经有critical级别的告警，那么将匹配目标新产生的告警级别为warning的将被抑制
  equal: ['alertname', 'cluster', 'service']    # 匹配哪些对象的告警
  
# 收件人配置
receivers:
- name: 'webhook'
  webhook_configs:
  - url: 'http://192.168.9.30:18080/prometheusalert?type=wx&tpl=prometheus-wx&wxurl=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=7fe90cd7-08a2-4c85-9ee7-ee7791044d29&at=韦振'
    send_resolved: true
  # email_configs:
  # - to: 'xxxxx@qq.com'

EOF

```

告警规则配置

在prometheus-operator中的规则非常齐全，基本属于开箱即用类型，大家可以根据日常收到的报警，对里面的rules报警规则作针对性的调整，比如把报警观察时长缩短一点等。对于单独部署的alertmanager，就需要单独的进行配置文件的配置

其他服务告警规则：https://github.com/samber/awesome-prometheus-alerts

```bash
# 监控报警规划修改
kubectl -n monitoring edit PrometheusRule kubernetes-monitoring-rules
# 或者修改manifests下的文件：prometheus-prometheusRule.yaml，然后apply


# 示例：
groups:
  - name: prometheus   # 告警规则组名称
    rules:
    - alert: PrometheusBadConfig   # 告警名称
      annotations:
        description: Prometheus {{$labels.namespace}}/{{$labels.pod}} has failed to reload its configuration.
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/prometheus/prometheusbadconfig
        summary: Failed Prometheus configuration reload.
      expr: |          # 告警规则表达式触发器，基于promsql
        max_over_time(prometheus_config_last_reload_successful{job="prometheus-k8s",namespace="monitoring"}[5m]) == 0    
      for: 10m       # 评估等待时间，可选参数。当相关指标触发规则后，在该时间区间内该规则会处于Pending状态，在达到该时间后规则状态变成Firing，并发送告警信息到Alertmanager。
      labels:        # 标签
        severity: critical
....
```

# PrometheusAlert告警通知

项目地址：https://github.com/feiyu563/PrometheusAlert/tree/master

alertmanager 是告警处理模块，但是告警消息的发送方法并不丰富。如果需要将告警接入飞书，钉钉，微信等，还需要有相应的SDK适配。prometheusAlert就是这样的SDK，可以将告警消息发送到各种终端上。

prometheus Alert 是开源的运维告警中心消息转发系统，支持主流的监控系统 prometheus，日志系统 Graylog 和数据可视化系统 Grafana 发出的预警消息。通知渠道支持钉钉、微信、华为云短信、腾讯云短信、腾讯云电话、阿里云短信、阿里云电话等。

![it.png](./images/it.png)

部署prometheusAlert相关步骤：

1. 创建飞书\企业微信机器人，获取机器人webhook地址
2. 准备配置文件
3. 启动 prometheusAlert服务
4. 对接告警服务
5. 调试告警模板





# kube-eventer事件监控

宝藏级秒级事件监控报警的开源软件`kube-eventer`，它是由阿里云开源的，并且还一直有在更新。

能及时灵敏地发现全球各个K8S集群的重要事件报警，使问题能得到及时的处理，维护了K8S集群的稳定性。

`kube-eventer`的github开源地址：

https://github.com/AliyunContainerService/kube-eventer

告警可输出到钉钉，企业微信，飞书，webhook，kafka，mysql等地方，具体配置参考github官方介绍

下面使用webhook简单示例安装配置

```bash
---
apiVersion: v1
data:
  content: >-
    {"EventType": "{{ .Type }}","EventNamespace": "{{
    .InvolvedObject.Namespace }}","EventKind": "{{ .InvolvedObject.Kind }}","EventObject": "{{
    .InvolvedObject.Name }}","EventReason": "{{
    .Reason }}","EventTime": "{{ .LastTimestamp }}","EventMessage": "{{ .Message
    }}"}
kind: ConfigMap
metadata: 
  name: kubeeventer-webhook
  namespace: kube-system 

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: kube-eventer
  name: kube-eventer-webhook
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-eventer
  template:
    metadata:
      labels:
        app: kube-eventer
      annotations:	
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      serviceAccount: kube-eventer
      containers:
        - image: registry.aliyuncs.com/acs/kube-eventer:v1.2.7-ca03be0-aliyun
          name: kube-eventer
          command:
            - "/kube-eventer"
            - "--source=kubernetes:https://192.168.9.30:6443"  # 可配置为域名，对应的解析在下面的hostAliases配置段
            ## .e.g,dingtalk sink demo
            #- --sink=dingtalk:[your_webhook_url]&label=[your_cluster_id]&level=[Normal or Warning(default)]&namespaces=[kube-system,kae-app(all)]
            - --sink=webhook:https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=7fe90cd7-08a2-4c85-9ee7-ee7791044d29?level=Warning&kinds=Pod&method=POST&header=Content-Type=application/json&custom_body_configmap=kube-eventer-webhook&custom_body_configmap_namespace=kube-system
          env:
          # If TZ is assigned, set the TZ value as the time zone
          - name: TZ
            value: "Asia/Shanghai" 
          volumeMounts:
            - name: localtime
              mountPath: /etc/localtime
              readOnly: true
            - name: zoneinfo
              mountPath: /usr/share/zoneinfo
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 500m
              memory: 250Mi
      hostAliases:
      - hostnames:         # 对应的webhook地址域名解析配置
        - xxx.any.com
        ip: 192.168.9.30
      volumes:
        - name: localtime
          hostPath:
            path: /etc/localtime
        - name: zoneinfo
          hostPath:
            path: /usr/share/zoneinfo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-eventer
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - events
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-eventer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-eventer
subjects:
  - kind: ServiceAccount
    name: kube-eventer
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-eventer
  namespace: kube-system

```

# falco容器运行时安全监控

官方网站：https://falco.org/zh-cn/docs/

Falco 可以监测调用Linux 系统调用的行为，并根据其不同的调用、参数及调用进程的属性发出警告。例如，Falco 可轻松检测：

- 容器内运行的 Shell
- 服务器进程产生意外类型的子进程
- 敏感文件读取（如 `/etc/shadow`）
- 非设备文件写入至 `/dev`
- 系统的标准二进制文件（如 `ls`）产生出站流量

实时检测异常活动和配置问题，输出警报

安装

```bash
# helm3 install(可选项)
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# 创建namespace
kubectl create ns falco

# 下载好离线 helm chart包
wget https://github.com/falcosecurity/charts/releases/download/falco-3.8.7/falco-3.8.7.tgz

# 一键式helm命令安装Falco
# 正式安装时需要去掉  --dry-run --debug（列出所需的资源信息）
helm -n falco install falco ./falco-3.8.7.tgz --set falco.jsonOutput=true --set falco.json_output=true --set falco.http_output.enabled=true --set falco.http_output.url=http://falco-falcosidekick:2801/ --set falcosidekick.enabled=true --set falcoctl.artifact.install.enabled=false --set falcoctl.artifact.follow.enabled=false --dry-run --debug

# 卸载
helm -n falco uninstall falco

```

配置发送告警测试

```bash
# 用nc启动一个webhook模拟服务端(可选项)
yum -y install nc
nc -l 9999

# 生成webhook地址的base64编码格式（不能有换行符，否则服务识别出错）
echo -n "http://192.168.9.30:9999" | base64
aHR0cDovLzE5Mi4xNjguMTIuMjMyOjUwMDA=

# 配置 falcosidekick 报警输出的 webhook 地址:
kubectl -n falco edit secrets falco-falcosidekick
  #在下面配置webhook地址的base64编码格式
  WEBHOOK_ADDRESS: aHR0cDovLzE5Mi4xNjguMTIuMjMyOjUwMDA=

# 解码：echo "aHR0cDovLzE5Mi4xNjguMTIuMjMyOjUwMDAvd2ViaG9vawo=" | base64 --decode

# 配置之后，删除pod让其重启生效
kubectl delete po -n falco falco-falcosidekick-695c494db4-sd4rr falco-falcosidekick-695c494db4-x4k6b

```

测试

```bash
# 随便 bash进入一个的pod就会发送告警
kubectl exec -it -n ingress-nginx ingress-nginx-controller-slszv -- bash

# 然后查看服务端，产生告警消息
[root@k8s-master1 ~]# nc -l 9999
POST / HTTP/1.1
Host: 192.168.9.30:9999
User-Agent: Falcosidekick
Content-Length: 1402
Content-Type: application/json; charset=utf-8
Accept-Encoding: gzip

{"uuid":"2435103e-c7ec-4118-b737-5f9979c00310","output":"08:36:31.233598643: Notice A shell was spawned in a container with an attached terminal (evt_type=execve user=www-data user_uid=101 user_loginuid=-1 process=bash proc_exepath=/bin/bash parent=containerd-shim command=bash terminal=34820 exe_flags=0 container_id=d394d17a25b2 container_image=registry.k8s.io/ingress-nginx/controller container_image_tag=latest container_name=k8s_controller_ingress-nginx-controller-slszv_ingress-nginx_3eaeaaf8-be47-4f32-895c-ec9caad3bf8c_0 k8s_ns=ingress-nginx k8s_pod_name=ingress-nginx-controller-slszv)","priority":"Notice","rule":"Terminal shell in container","time":"2024-03-29T08:36:31.233598643Z","output_fields":{"container.id":"d394d17a25b2","container.image.repository":"registry.k8s.io/ingress-nginx/controller","container.image.tag":"latest","container.name":"k8s_controller_ingress-nginx-controller-slszv_ingress-nginx_3eaeaaf8-be47-4f32-895c-ec9caad3bf8c_0","evt.arg.flags":"0","evt.time":1711701391233598643,"evt.type":"execve","k8s.ns.name":"ingress-nginx","k8s.pod.name":"ingress-nginx-controller-slszv","proc.cmdline":"bash","proc.exepath":"/bin/bash","proc.name":"bash","proc.pname":"containerd-shim","proc.tty":34820,"user.loginuid":-1,"user.name":"www-data","user.uid":101},"source":"syscall","tags":["T1059","container","maturity_stable","mitre_execution","shell"],"hostname":"falco-pwf8x"}


```

**使用flask写一个服务端接收告警数据**

安装flask

```python
pip3 install flask
```

示例代码：

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.get_json(force=True)
    print("Received webhook data:", data)

    # 这里可以添加接收到的webhook数据的处理逻辑
    # ...

    return jsonify({"message": "Webhook received!"}), 200

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=9999)

```





