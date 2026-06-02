+++
title = "业务迁移k8s案例"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 1
+++

参考博客：https://www.cuiliangblog.cn/detail/article/52

# 基本流程

1. 第一步就是要编写dockerfile，将原本在Linux服务器上运行的服务打包到容器中运行。
2. 再部署运行环境，配置环境变量，最后上传项目代码到服务器中，配置启动脚本或者守护进程。
3. 创建、上传镜像
4. 编写yaml文件，创建资源
5. 创建service，ingress外部访问

操作分为了三个层面，最底层的操作系统，中间层的运行环境，上层的应用服务。

<img src="./images/image-20240419185837158.png" alt="image-20240419185837158" style="zoom:50%;" />

# 前提准备

- **打包镜像**

dockerfile构建镜像经验

1. 减少镜像的层数，尽量把一些功能上面统一的命令合到一起来做；
2. 注意清理镜像构建的中间产物，比如一些安装包在装完之后就把它删掉；
3. 注意优化网络请求，使用yum源的时候，用一些网络比较好的源站点，可节约时间，减少失败率；
4. 尽量去使用缓存构建，尽量把一些不变的东西或者变动比较少的东西放在前面。例如业务更新，打包新版本镜像时，只有代码发生变化，其他内容不变。此时可以把拷贝代码放最后，前面的构建阶段直接使用缓存即可。



- **使用Pod运行容器**

当我们把应用封装成docker镜像后，接下来就是在kubernetes中启动镜像运行容器。因为Pod是Kubernetes管理的最小单元，Kubernetes不直接管理容器，而是管理Pod，Pod里面包含一个或多个容器。需要考虑是一个Pod中放置多个容器，还是一个Pod中放置一个容器。在一个pod中所有container共享，PID、Network、IPC

在有些业务场景中，例如一个web服务，对应的资源文件需要从远端实时监听更新，就需要使用边车 (sidercar)模式，一个pod中包含一个web容器一个file容器，通过共享存储方式实现。又比如一个用户的微服务包含：User API、User Control、User Data等三个模块，彼此之间紧耦合，对外只需要通过User API，这样类型的应用就可以放置在一个Pod中。

Pod资源清单建议

1. 建议开发项目时，提供healthy健康检查接口，便于检测服务是否正常。
2. 建议运维配置LivenessProbe存活检测探针，防止服务假死。
3. 建议配置resources，避免程序出现异常，并占用大量的系统资源，从而会影响节点上其他的Pod。



- **开放端口访问**

**使用Service访问Pod**

service使用：

- ClusterIP(集群IP)：集群内的服务间通信。 例如，应用程序的前端(front-end)和后端(back-end)组件之间的通信。
- NodePort(节点端口)：k8s集群内部服务暴露给外部时访问，可以通过k8s节点ip+端口方式访问服务。
- LoadBalancer（负载均衡器）：使用云厂商来托管您的 Kubernetes 集群时。由它接入外部客户端的请求并调度至集群节点相应的NodePort之上
- ExternalName（外部名称）：在 Kubernetes 内创建服务来表示映射外部服务名称，例如在 Kubernetes 中使用公有云数据库时，可以创建一个ExternalName资源，后续更换云数据库地址时，更新ExternalName配置即可

**Ingress提供外部访问：**

Ingress同样也只是一个概念，它的具体的实现依赖控制器，Ingress控制器并不直接运行为kube-controller-manager的一部分，它是Kubernetes集群的一个重要附件，类似于CoreDNS，需要在集群上单独部署。Ingress控制器可以由任何具有反向代理（HTTP/HTTPS）功能的服务程序实现，如Nginx、Envoy、HAProxy、Vulcand和Traefik等。



- **ConfigMap管理配置文件**

在DevOps的部署流水线中，其中有一个核心的理念就是代码和配置的分离，这样更容易实现流水线的编排。我们只需要使用一套代码，配合不同的配置文件，就可以实现灵活发布到测试环境、预发布环境、生产环境等。

![image-20240419191733338](./images/image-20240419191733338.png)

kubernetes配置与文件管理方案：

- ConfigMap配置文件：可以从文件、文件夹等途径创建ConfigMap。然后再Pod中挂载使用配置文件，例如nginx配置。
- secret私密文件：使用base64加密的文件，例如存储第三方镜像仓库凭证 配置TLS 类型secret 用于记录证书秘钥等信息。



- **持久化存储数据**

存储卷包括传统的NAS或SAN设备（如NFS、iSCSI、fc）、分布式存储（如GlusterFS、RBD）、云端存储（如gcePersistentDisk、azureDisk、cinder和awsElasticBlockStore）以及建构在各类存储系统之上的抽象管理层（如flocker、portworxVolume和vsphereVolume）等。



# 资源清单部署

## 案例1：springboot

**分析**

- 开发提供jar包：基础镜像直接使用open-jdk即可
- 部署到k8s：多副本管理，滚动更新：需要使用Deployment控制器；Pod多副本负载均衡：需要使用ClusterIP服务资源
- 用户通过域名访问：使用Ingress创建域名路由，流量转发到Service

我们将打包好的jar文件和dockerfile放在同一级目录下，执行docker build指令，即可完成镜像构建

```bash
[root@tiaoban springboot]# ls
demo  demo-v1.jar  demo-v2.jar  Dockerfile
[root@tiaoban springboot]# cat Dockerfile 
FROM openjdk:19-jdk-alpine3.16
ADD demo-v1.jar /opt/app.jar
EXPOSE 8888
WORKDIR /opt
CMD ["java","-jar","app.jar"]
# 打包镜像
[root@tiaoban springboot]# docker build -t springboot:v1 .
# 启动容器，访问测试
[root@k8s-master springboot]# docker run -d -p 8888:8888 springboot:v1
74a8bc2446d7f30195099c8f6f905481b2fdc6b585436e4b810849f1df50185d
[root@k8s-master springboot]# curl 127.0.0.1:8888
Hello SpringBoot Version:v1[root@k8s-master springboot]# 
[root@k8s-master springboot]# curl 127.0.0.1:8888/healthy
ok[root@k8s-master springboot]#
```

**推送至镜像仓库**
构建好镜像测试无误后，接下来就可以推送到harbor私有镜像仓库或者公有云镜像仓库中

**编写deploy yaml文件**

```yaml
cat >deployment.yaml <<EOF 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot
  namesapce: springboot   # 指定命名空间
spec:
  replicas: 2 # 副本数
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
    spec:
      containers:
      - name: springboot
        image: 192.168.10.100/demo/springboot:v1
        resources:
          limits:
            memory: "1Gi"
            cpu: "1"
          requests:
            memory: "128Mi"
            cpu: "100m" 
        ports:
          - containerPort: 8888
            name: web
        livenessProbe:
          httpGet:
            port: web
            path: /healthy
          timeoutSeconds: 2       # 表示容器必须在2s内做出相应反馈给probe，否则视为探测失败
          periodSeconds: 30       # 探测周期，每30s探测一次
        readinessProbe:
          tcpSocket:
            port: web
          initialDelaySeconds: 10 # 容器启动后10s开始探测       
EOF

kubectl apply -f deployment.yaml 

```

创建service资源

```yaml
cat >service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: springboot
  namesapce: springboot
spec:
  type: ClusterIP 
  selector:
    app: springboot 
  ports:
  - name: http
    port: 8888
    protocol: TCP 
    targetPort: 8888 
EOF

kubectl apply -f service.yaml 

```

创建ingress资源

- traefik版本

```yaml
cat >ingress.yaml <<EOF
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: springboot
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`springboot.test.com`) # 域名
    kind: Rule
    services:
      - name: springboot  # 与svc的name一致
        port: 8888     # 与svc的port一致
EOF

kubectl apply -f ingress.yaml 

```

- ingress-nginx版本

```yaml
# networking.k8s.io/v1接口版本：
cat >ingress.yaml	<< EOF  
apiVersion: networking.k8s.io/v1   # 在1.22版本之前使用extensions/v1beta1接口版本
kind: Ingress
metadata:
  name: springboot-ingress
  annotations:
  	kubernetes.io/ingress.class: nginx    # 指定当前使用的ingress-controller的版本类型
spec:
  rules:
  - host: springboot.test.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:   # 注意此处与extensions/v1beta1版本接口配置有所差别
          service:
            name: springboot
            port:
              number: 8888
EOF

kubectl apply -f ingress-app.yaml
```



## 案例2：VUE

**需求分析**

- VUE项目，只提供源码：构建镜像时，使用Nodejs和NGINX两个基础镜像多阶段构建
- 部署开发与生产环境：一套代码，两套环境，通过configmap创建两套不同配置。
- 使用两个域名访问两个环境：部署两套Controller 、Service、Ingress。
- 从远端定时拉取资源：需要使用sidecar模式，一个vue容器，一个img容器，共享存储路径，实现静态资源远端拉取更新。

在这个案例中，我们只有git仓库的地址。因此我们首先要克隆代码，然后再执行打包并测试

```bash
# 下载代码并运行测试
git clone git@gitee.com:cuiliang0302/vue3_vite_element-plus.git
docker build -f Dockerfile-vue -t vue:v1 .
docker run -d -p 90:80 vue:v1
```

Dockerfile文件

```bash
FROM node:20.11.0 AS build       # 使用node 16.15运行环境 用于打包项目,并将第一阶段命名为build
COPY . /vue                       # 复制项目代码到/vue目录下
WORKDIR /vue                      # 设置工作目录为/vue
RUN npm install --registry https://registry.npm.taobao.org && npm run build  # 安装项目依赖并打包项目

FROM nginx:1.25         							 # 使用nginx 1.20运行环境为基础镜像
COPY --from=build /vue/dist /opt/vue/dist  # 拷贝build阶段生成的打包文件dist到容器目录下
EXPOSE 80                                  # 声明容器暴露的端口
COPY vue.conf /etc/nginx/nginx.d/vue.conf  # 拷贝nginx配置文件到容器nginx配置文件目录下
CMD ["nginx", "-g","daemon off;"]          # 指定启动命令

```



**创建configmap资源**
我们分别创建两套nginx的配置文件，分别对应开发环境和生产环境。

```bash
[root@k8s-master vue]# cat configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: vue-dev
data:
  default.conf: |-
    server {
      listen       80;
      server_name  vue-dev.test.com; # 开发环境域名
      location /img {                # 访问img路径下资源时，重定向到百度页面
          return 301 https://www.baidu.com;
      }
      location / {
          root  /opt/vue/dist;
          index  index.html index.htm;
          try_files $uri $uri/ /index.html;
          add_header Access-Control-Allow-Origin *;
      }
    } 
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: vue
data:
  default.conf: |-
    server {
      listen       80;
      server_name   vue.test.com; # 生产环境域名
      location /img {             # 访问img路径下资源时，从远端获取，保存到/media目录下
          alias /media/;
      }
      location / {
          root  /opt/vue/dist;
          index  index.html index.htm;
          try_files $uri $uri/ /index.html;
          add_header Access-Control-Allow-Origin *;
      }
    }
[root@k8s-master vue]# kubectl apply -f configmap.yaml 
configmap/vue-dev created
configmap/vue created
```

**创建deployment资源**
生产和开发环境对应两个不同的控制器，他俩的区别在于使用的configmap不同，在生产模式下，使用sidecar定期从远端获取资源，存放在/media共享路径下。

```bash
[root@k8s-master vue]# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vue-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vue-dev
  template:
    metadata:
      labels:
        app: vue-dev
    spec:
      containers:
      - name: vue
        image: xxx/vue:v1           # 镜像地址,注意修改
        resources:
          limits:
            memory: "1Gi"
            cpu: "1"
          requests:
            memory: "128Mi"
            cpu: "100m" 
        ports:
          - containerPort: 80
            name: web
        livenessProbe:
          httpGet:
            port: web
            path: /
        readinessProbe:
          tcpSocket:
            port: web
        volumeMounts:
          - mountPath: /etc/nginx/conf.d/default.conf
            subPath: default.conf
            name: nginx-config
      volumes:  # 使用开发环境的nginx配置文件
        - name: nginx-config
          configMap:
            name:  vue-dev
            
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vue
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vue
  template:
    metadata:
      labels:
        app: vue
    spec:
      containers:
      - name: vue
        image: xxx/vue:v1     # 镜像地址,注意修改
        resources:
          limits:
            memory: "1Gi"
            cpu: "1"
          requests:
            memory: "128Mi"
            cpu: "100m" 
        ports:
          - containerPort: 80
            name: web
        livenessProbe:
          httpGet:
            port: web
            path: /
        readinessProbe:
          tcpSocket:
            port: web
        volumeMounts:
          - mountPath: /etc/nginx/conf.d/default.conf
            subPath: default.conf
            name: nginx-config
          - mountPath: /media
            name: media
      - name: img
        image: 192.168.10.100/demo/img:v1
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m" 
        volumeMounts:
          - mountPath: /media
            name: media
      volumes:
        - name:  nginx-config  # 使用生产模式nginx配置文件
          configMap:
            name:  vue
        - name: media          # 挂载空目录，用于存放远端资源
          emptyDir: {}
[root@k8s-master vue]# kubectl apply -f deployment.yaml 
deployment.apps/vue-dev created
deployment.apps/vue created
```

**创建service资源**
service资源同样也是两套，唯一不同的区别在于匹配不同的资源标签

```bash
[root@k8s-master vue]# cat service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: vue-dev
spec:
  type: ClusterIP 
  selector:
    app: vue-dev 
  ports:
  - name: http
    port: 80
    protocol: TCP 
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: vue
spec:
  type: ClusterIP 
  selector:
    app: vue
  ports:
  - name: http
    port: 80
    protocol: TCP 
    targetPort: 80
[root@k8s-master vue]# kubectl apply -f service.yaml 
service/vue-dev created
service/vue created
```

**创建ingress资源**
ingress资源同样也是两套，对应不同的环境域名

```yaml
# networking.k8s.io/v1接口版本：
cat >ingress-app.yaml	<< EOF  
apiVersion: networking.k8s.io/v1   # 在1.22版本之前使用extensions/v1beta1接口版本
kind: Ingress
metadata:
  name: nginx-app-ingress
  annotations:
  	kubernetes.io/ingress.class: nginx    # 指定当前使用的ingress-controller的版本类型
spec:
  rules:
  - host: vue-dev.test.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:   # 注意此处与extensions/v1beta1版本接口配置有所差别
          service:
            name: vue-dev
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-app-ingress
  annotations:
  	kubernetes.io/ingress.class: nginx    # 指定当前使用的ingress-controller的版本类型
spec:
  rules:
  - host: vue.test.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:   # 注意此处与extensions/v1beta1版本接口配置有所差别
          service:
            name: vue
            port:
              number: 80
EOF

kubectl apply -f ingress-app.yaml
```



## 案例3：Python爬虫项目

需要安装网络软件包，实现IP代理池的使用，将日志采集并实时上传到ES便于观察，并要求数据目录持久化存储，定时运行项目



**分析**

- 安装网络软件包：不能使用Python基础镜像，只能使用centos、ubuntu等系统镜像
- 定时运行项目：使用cronjob控制器
- 持久化存储数据：使用nfs、ceph共享存储服务
- 将日志采集并实时上传到ES：方案1：使用sidecar模式启动filebeat服务，同一pod包含两个container，共用日志目录。方案2：使用hostpath挂载宿主机目录，使用daemonset控制器启动filebeat服务，采集宿主机目录日志。但是此案例中，python程序仅需几秒便可执行完成，而filebeat需要持续运行，使用sidecar模式会导致filebeat还未开始采集日志，python程序已经运行完成，最终导致pod状态码异常，因此采用daemonset方案。

Dockerfile文件

- 情况1：

  开发的同事提供python代码，需要我打包镜像，要求开启SSH服务，便于开发同事随时连接容器调试代码,这种情况下就需要使用最基础的系统层镜像，然后自定义Python环境，完成镜像封装。

```bash
FROM centos:centos7 # 选取centos8为基础镜像
RUN yum install openssh-server passwd python38 python38-devel -y # 安装相关软件包
RUN /bin/echo "123.com" | passwd --stdin root # 设置ssh密码
RUN /bin/sed -i 's/.session.required.pam_loginuid.so./session optional pam_loginuid.so/g' /etc/pam.d/sshd && /bin/sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config && /bin/sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config # 修改sshd配置文件
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key && ssh-keygen -t rsa -f /etc/ssh/ssh_host_ecdsa_key && ssh-keygen -t rsa -f /etc/ssh/ssh_host_ed25519_key # 创建密钥
RUN echo -e "#! /bin/bash\n/usr/sbin/sshd -D" > /run.sh # 创建sshd服务启动脚本
ADD . /app # 拷贝代码到容器目录
RUN pip3.8 install -r /app/requirements.txt # 安装依赖
EXPOSE 22 # 声明容器暴露的端口
CMD ["/usr/sbin/sshd","-D"] # 指定启动命令
```

- 正常情况下：

```bash
FROM rockylinux:8
RUN dnf -y install wget gcc gcc-c++ net-tools telnet iproute procps tcpdump python38 python38-devel
ADD project /project
RUN pip3.8 install -r /project/requirements.txt -i https://pypi.doubanio.com/simple
WORKDIR /project
CMD ["python3.8","main.py"]
```

```bash
# 修改为自己的镜像仓库地址
# docker tag xxx/python:v1
# docker push xxx/python:v1
```



打包好了镜像并且上传到镜像仓库之后

**创建pvc资源，用于持久化存储爬虫数据**

```bash
[root@k8s-master python]# cat pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: python-pvc
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
[root@k8s-master python]# kubectl apply -f pvc.yaml 
persistentvolumeclaim/python-pvc created
```



**创建cronjob资源，定时执行爬虫程序**

```bash
cat >cronjob.yaml <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: python
spec:
  schedule: "* * * * *"
  jobTemplate:
    metadata:
      name: python
    spec:
      template:
        spec:
          containers:
          - name: python
            image: xxx/python:v1     # 镜像地址,注意修改
            volumeMounts:
            - name: python-data
              mountPath: "/project/data"
            - name: python-log
              mountPath: "/project/log"
            env:
            - name: TZ
              value: Asia/Shanghai
          restartPolicy: Never
          volumes:
          - name: python-data
            persistentVolumeClaim:
              claimName: python-pvc
          - name: python-log
            hostPath:
              path: /data/python-log
              type: DirectoryOrCreate
EOF

kubectl apply -f cronjob.yaml 

```

**导出es证书**
因为es从8开始，安装后默认开启了base基础用户验证和TLS证书。filebeat想要连接ES需要指定SSL证书，否则会报X509证书异常错误。

```bash
[root@tiaoban ~]# docker cp elasticsearch:/usr/share/elasticsearch/config/certs .
Successfully copied 21.5kB to /root/.
[root@tiaoban ~]# ls
anaconda-ks.cfg  certs  elasticsearch.yml
[root@tiaoban ~]# cd certs/
[root@tiaoban certs]# ls
http_ca.crt  http.p12  transport.p12
[root@tiaoban certs]# scp http_ca.crt k8s-master:/root
```

**创建secret资源**
从docker容器中导出es证书后，创建secret资源，后续filebeat直接挂载证书资源即可。

```bash
[root@k8s-master ~]# kubectl create secret generic es --from-file=./http_ca.crt 
secret/es created
[root@k8s-master ~]# kubectl describe secrets es 
Name:         es
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
http_ca.crt:  1915 bytes
```

**创建filebeat的configmap资源**

```bash
cat >configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: log
      enabled: true
      paths:
      - /project/log/*.log
    setup.ilm.enabled:  false #新版本的Filebeat则默认的配置开启了ILM 导致索引的命名规则被ILM策略控制
    setup.template.name:  "python"
    setup.template.pattern:  "python-*"
    setup.template.overwrite:  false
    setup.template.settings:
      index.number_of_shards: 1 #索引分片数
      index.number_of_replicas: 0 #索引副本数
    output.elasticsearch:  #指定ES的配置
      hosts:  ["https://192.168.9.30:9200"]
      username: "elastic"
      password: "HufU-ybqd5aeEvh8xf6y"
      index: "python-%{+yyyy.MM.dd}"
      ssl.certificate_authorities: ["/secret/http_ca.crt"]
EOF

kubectl apply -f configmap.yaml 

```

**创建daemonset资源，每个节点启动一个filebeat**

```yaml
cat >daemonset.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: filebeat
        image: elastic/filebeat:8.6.2
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m" 
          requests:
            memory: "10Mi"
            cpu: "10m" 
        args: ["-c","/etc/filebeat/filebeat.yml","-e"]
        volumeMounts:
        - name: filebeat-config
          mountPath: /etc/filebeat/filebeat.yml
          subPath: filebeat.yml
        - name: python-log
          mountPath: /project/log    # 挂载目录
        - name: es-cert
          mountPath: /secret
          readOnly: true
      volumes:
      - name: python-log
        hostPath:
          path: /data/python-log
          type: DirectoryOrCreate
      - name: filebeat-config
        configMap:
          name: filebeat-config
      - name: es-cert
        secret:
          secretName: es
EOF

kubectl apply -f daemonset.yaml 

```

