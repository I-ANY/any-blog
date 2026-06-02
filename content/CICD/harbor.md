+++
title = "harbor"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 4
+++

# 安装

下载地址：https://github.com/goharbor/harbor/releases 

本次以v2.6.2版本为例，先下载harbor-offline-installer-v2.6.2.tgz到主机上。

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.6.2/harbor-offline-installer-v2.6.2.tgz
#解压软件包
tar xf harbor-offline-installer-v2.6.2.tgz -C /opt
#进入harbor目录
cd /opt/harbor
```

修改配置文件

- http访问

检查80端口是否被占用，若被占用，更改端口

```bash
netstat -tunlp|grep 80
```

```bash
# cp harbor.yml.tmpl harbor.yml

# vim harbor.yml
# 设置域名，也可直接ip地址
# hostname: 192.168.9.20
hostname: harbor.local.com
# 注释https相关配置
# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
#https:
  # https port for harbor, default is 443
  #port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path
```

- https方式(建议)

**生成自签名证书**
这里使用https://github.com/Fishdrowned/ssl提供的shell脚本来生成ssl证书，证书有效期是 2 年，可以修改 ca.cnf 来修改这个年限。

```bash
# 克隆项目
git clone https://github.com/Fishdrowned/ssl.git
# 一键生成证书
cd ssl && ./gen.cert.sh harbor.local.com # 生成harbor.local.com域名的证书

# 查看证书文件
cd out/harbor.local.com/ && ll
总用量 0
drwxr-xr-x 2 root root 101 8月  13 18:49 20231013-1849
lrwxrwxrwx 1 root root  43 8月  13 18:49 harbor.local.com.bundle.crt -> ./20231013-1049/harbor.local.com.bundle.crt
lrwxrwxrwx 1 root root  36 8月  13 18:49 harbor.local.com.crt -> ./20231013-1049/harbor.local.com.crt
lrwxrwxrwx 1 root root  15 8月  13 18:49 harbor.local.com.key.pem -> ../cert.key.pem
lrwxrwxrwx 1 root root  11 8月  13 18:49 root.crt -> ../root.crt

# 拷贝证书至harbor目录
cp harbor.local.com.crt /opt/harbor/
cp harbor.local.com.key.pem /opt/harbor/
```

**修改配置文件**

```bash
# 先备份
# cp harbor.yml.tmpl harbor.yml

# vim harbor.yml
# 设置域名
hostname: harbor.local.com
# 注释http相关配置
# http related config
# http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  # port: 80

# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /opt/harbor/harbor.local.com.crt 
  private_key: /opt/harbor/harbor.local.com.key.pem
data_volume: /data/harbor
```

执行安装脚本

```bash
# 在harbor目录下进行安装
./install.sh
```

安装成功后，在浏览器上访问ip地址即可访问到harbor。账号admin、密码为上面配置所配置的密码。

开机自启文件

```bash
cat > /usr/lib/systemd/system/harbor.service <<EOF
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor

[Service]
Type=simple
Restart=on-failure
RestartSec=5
#注意docker-compose和harbor的安装位置
ExecStart=/usr/bin/docker-compose -f  /opt/harbor/docker-compose.yml up
ExecReload=/usr/local/bin/docker-compose -f /opt/harbor/docker-compose.yml restart
ExecStop=/usr/bin/docker-compose -f /opt/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
EOF
```

```bash
#手动停止
/usr/bin/docker-compose -f /soft/harbor/docker-compose.yml down

#加载配置和设置开机自启
systemctl daemon-reload
systemctl enable harbor.service --now

systemctl status harbor.service
```

访问Harbor并登陆，也可以修改配置文件中的参数指定admin密码：harbor_admin_password: 123456

- 初始用户名：admin
- 初始密码：Harbor12345

# docker授权访问

docker配置文件私有仓库设置

```bash
# vim /etc/docker/daemon.json
{
    "registry-mirrors": [
        "https://mirror.ccs.tencentyun.com",
        "https://o2j0mc5x.mirror.aliyuncs.com"
    ],
    "insecure-registries": [
        "https://harbor.local.com"
    ]
}
```

```bash
# 重启docker
systemctl daemon-reload 
systemctl restart docker

# 登录
docker login harbor.local.com -u admin

# 然后就可推送镜像
```

# Containerd授权访问

修改配置文件

```bash
# mkdir -p /etc/containerd/certs.d/harbor.local.com
# cat > /etc/containerd/certs.d/harbor.local.com/hosts.toml << EOF
server = "https://harbor.local.com"

[host."https://harbor.local.com"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
EOF
```

```bash
# 重启
systemctl restart containerd 

# 登录
nerdctl login -u admin -p Harbor12345 --insecure-registry harbor.local.com

# 然后即可推动镜像
```

# kubernets访问harbor仓库

由于harbor采用了用户名密码认证，所以在镜像下载时，如果仓库类型为私有，则需要配置sercet才可以拉取镜像。

- 创建secret

```bash
kubectl create secret docker-registry harbor-secret --namespace=default --docker-server=harbor.local.com --docker-username=admin --docker-password=Harbor12345
```

- 查看secret

```bash
# kubectl get secrets 
NAME              TYPE                             DATA   AGE
harbor-secret   kubernetes.io/dockerconfigjson   1      9s
```

- 使用相应的私有registry中镜像的Pod资源的定义，即可通过imagePullSecrets字段使用此Secret对象

```yaml
	apiVersion: v1
	kind: Pod 
	metadata:
	  name: secret-imagepull-demo
	  namespace: default
	spec:
	  imagePullSecrets:     # 指定secret对象
	  - name: harbor-secret
	  containers:
	  - image: 192.168.9.20/k8s/nginx:v1
	    name: myapp
```

