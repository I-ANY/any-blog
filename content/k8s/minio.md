+++
title = "minio"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 12
+++

# 介绍

官方文档(中文)：http://docs.minio.org.cn/docs/

MinIO 是在 Apache License v2.0 下发布的高性能对象存储。它是与 Amazon S3 云存储服务兼容的 API。使用 MinIO 构建 用于机器学习、分析和应用的高性能基础设施数据工作负载。MinIO 从根本上与众不同,专为企业和私有云设计。MinIO生产部署涵盖了全球。MinIO是全球使用最多和下载量最大的对象存储服务系统，还是全世界增长最快的对象存储系统。

生产 MinIO 部署由至少 4 个具有同构存储和计算资源的 MinIO 主机组成。 这些资源聚合在一起作为一个 服务器池，对外抽象为一个可被访问的对象存储服务

EC

```bash
# ec
纠删码，也叫奇偶校验码，数字表示所有磁盘中要分出多少来进行数据校验，参考RAID
公式
n = k + m
n: 总数据块
k: 原始数据块
m: 校验块，本架构指ec数量，也就意味着集群中可以允许m块磁盘进行故障而不影响读写
冗余度 n+m/n 冗余度越高，磁盘利用率越低
k值影响数据恢复代价
	k值越小，数据分散度越小，故障影响面越大
	k值越大，多路数据拷贝增加的网络和磁盘负载越大
一般推荐使用ec4集群冗余度1.33
```

![image-20240430142519295](./images/image-20240430142519295.png)

# 安装

## k8s

https://www.minio.org.cn/docs/minio/kubernetes/upstream/index.html

## docker

```bash
mkdir -p ~/minio/data

docker run \
   -p 9000:9000 \
   -p 9001:9001 \
   --name minio \
   -v ~/minio/data:/data \
   -e "MINIO_ROOT_USER=minioadmin" \
   -e "MINIO_ROOT_PASSWORD=minioadmin" \
   quay.io/minio/minio server /data --console-address ":9001"
```

```bash
docker run启动 MinIO 容器
-p将本地端口绑定到容器端口
-name为容器创建一个名称
9000表示MinIO服务地址，其上传调用的就是这个服务地址
9090表示MinIO的Web Console地址，Console监听的是一个动态的端口, 可以通过 --console-address ":port" 指定静态端口
/mnt/data/minio/data:/data表示将MiniIO的数据挂载到宿主机上
/mnt/data/minio/config:/root/.minio表示将MiniIO的配置文件挂载到宿主机上
MINIO_ROOT_USER=minioadmin表示MinIO部署的root用户的用户名(accessKey)，不写默认的用户名就是minioadmin
MINIO_ROOT_PASSWORD=minioadmin表示MinIO部署的root用户的密码(secretKey)，不写默认的密码就是minioadmin
```

## 二进制单节点



## 二进制(4磁盘3节点)



官方文档： https://www.minio.org.cn/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

```shell
wget https://dl.minio.org.cn/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

```bash
# 创建数据目录
mkdir -p /data/minio/disk-{1...4}
groupadd -r minio-user
useradd -M -r -g minio-user minio-user
chown minio-user:minio-user /data/minio/*
```



配置环境变量文件

```bash
mkdir /etc/minio/ -p
cat >/etc/minio/config <<EOF
MINIO_VOLUMES="http://192.168.9.2{1...3}:9000/data/minio/disk-{1...4}"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=12345678
MINIO_SERVER_URL="http://192.168.9.30:81" # 负载均衡地址和server端口
EOF
```

服务启动文件

```bash
cat >/usr/lib/systemd/system/minio.service <<EOF
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
Type=notify
WorkingDirectory=/usr/local
User=minio-user
Group=minio-user
ProtectProc=invisible
EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

Restart=always
LimitNOFILE=1048576
MemoryAccounting=no
TasksMax=infinity
TimeoutSec=infinity

SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF

```

启动

```bash
systemctl daemon-reload
systemctl enable  minio.service --now
systemctl status  minio.service
```

命令启动

```bash
/usr/local/bin/minio server --console-address ":9090" --address "192.168.9.20:9000" /data/minio/disk-1
```

