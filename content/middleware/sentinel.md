+++
title = "sentinel"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 3
+++





# k8s部署sentinel

官网dockerfile：https://github.com/alibaba/Sentinel/blob/master/sentinel-dashboard/Dockerfile

![image-20231212135811524](./images/image-20231212135811524.png)

构建镜像

```yaml
FROM amd64/buildpack-deps:buster-curl as installer

ARG SENTINEL_VERSION=1.8.6

RUN set -x \
    && curl -SL --output /home/sentinel-dashboard.jar https://github.com/alibaba/Sentinel/releases/download/${SENTINEL_VERSION}/sentinel-dashboard-${SENTINEL_VERSION}.jar

FROM openjdk:8-jre-slim

# copy sentinel jar
COPY --from=installer ["/home/sentinel-dashboard.jar", "/home/sentinel-dashboard.jar"]

ENV JAVA_OPTS '-Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080'

RUN chmod -R +x /home/sentinel-dashboard.jar

EXPOSE 8080

CMD java ${JAVA_OPTS} -jar /home/sentinel-dashboard.jar

# 如果下载的慢，可以直接先下载jar然后，再构建镜像：https://github.com/alibaba/Sentinel/releases/download/1.8.6/sentinel-dashboard-1.8.6.jar

# 构建好镜像之后就推送到habor或者公有云的仓库里面，方便下载

```



创建StatefulSet资源

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sentinel-cm
  namespace: psw002-test
data:
  sentinel.server.host: "sentinel"
  sentinel.server.port: "8858"
  sentinel.dashboard.auth.username: "sentinel"
  sentinel.dashboard.auth.password: "sentinel"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sentinel
  namespace: psw002-test
  labels:
    app: sentinel
spec:
  serviceName: sentinel-svc
  replicas: 1
  selector:
    matchLabels:
      app: sentinel
  template:
    metadata:
      labels:
        app: sentinel
    spec:
      containers:
        - name: sentinel
          image: swr.cn-south-1.myhuaweicloud.com/psw002-test/sentinel:v1.8.6   # 基于根据上面dockefile打包的已经推送到云的镜像
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              cpu: 450m
              memory: 1024Mi
            requests:
              cpu: 400m
              memory: 1024Mi
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: JAVA_OPT_EXT
              value: -Dserver.servlet.session.timeout=7200
            - name: SERVER_HOST
              valueFrom:
                configMapKeyRef:
                  name: sentinel-cm
                  key: sentinel.server.host
            - name: SERVER_PORT
              valueFrom:
                configMapKeyRef:
                  name: sentinel-cm
                  key: sentinel.server.port
            - name: USERNAME
              valueFrom:
                  configMapKeyRef:
                    name: sentinel-cm
                    key: sentinel.dashboard.auth.username
            - name: PASSWORD
              valueFrom:
                  configMapKeyRef:
                    name: sentinel-cm
                    key: sentinel.dashboard.auth.password
          ports:  
            - containerPort: 8080 #Dashboard服务的端口 客户端向控制台发送心跳包的控制台地址，指定控制台后客户端会自动向该地址发送心跳包
            - containerPort: 8719 #客户端的端口 提供给Dashboard访问
      imagePullSecrets:
        - name: harbor-secret 
---
apiVersion: v1
kind: Service
metadata:
  namespace: psw002-test
  name: sentinel
  labels:
    app: sentinel
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 31080
      targetPort: 8080
      name: web
    - port: 8719
      targetPort: 8719
      name: api
  selector:
    app: sentinel

```



