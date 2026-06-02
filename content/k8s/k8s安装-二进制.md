+++
title = "k8s安装-二进制"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 6
+++



参考博客：https://www.cnblogs.com/liweifeng888/p/17435629.html

## 系统优化

### 1. 调整系统配置

操作节点： 所有的master和slave节点（`k8s-master,k8s-slave`）需要执行

>本章下述操作均以k8s-master为例，其他节点均是相同的操作（ip和hostname的值换成对应机器的真实值）

- **设置安全组开放端口**

如果节点间无安全组限制（内网机器间可以任意访问），可以忽略，否则，至少保证如下端口可通：
k8s-master节点：TCP：6443，2379，2380，60080，60081UDP协议端口全部打开
k8s-slave节点：UDP协议端口全部打开

``` python
# 设置iptables
iptables -P FORWARD ACCEPT

# 关闭swap
swapoff -a
# 防止开机自动挂载 swap 分区
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 关闭selinux和防火墙
setenforce 0
sed -ri 's#(SELINUX=).*#\1disabled#' /etc/selinux/config
systemctl disable firewalld && systemctl stop firewalld
systemctl disable NetworkManager && systemctl stop NetworkManager

# 修改内核参数
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
vm.max_map_count=262144
EOF

cat >> /etc/sysctl.conf<<EOF
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.neigh.default.gc_thresh1=4096
net.ipv4.neigh.default.gc_thresh2=6144
net.ipv4.neigh.default.gc_thresh3=8192
EOF

#修改句柄数
cat <<EOF >  /etc/security/limits.conf 
* soft  nofile       819200
* hard  nofile       819200
* soft  nproc        unlimited
* hard  nproc        unlimited
* soft  memlock      unlimited
* hard  memlock      unlimited
* soft  core         unlimited
EOF

modprobe br_netfilter
#生效
sysctl -p && sysctl --system

```

- **时间统一**

```bash
#修改时区，同步时间
yum install chrony -y
cat >>/etc/chrony.conf <<EOF
ntpdate ntp1.aliyun.com iburst
EOF
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' > /etc/timezone
```

- **设置yum源**

```shell
curl -o /etc/yum.repos.d/Centos-7.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/docker-ce.repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum clean all && yum makecache

```

- **设置hosts解析**

操作节点：所有节点（`k8s-master，k8s-slave`）均需执行

修改hostname
hostname必须只能包含小写字母、数字、","、"-"，且开头结尾必须是小写字母或数字

``` python
# 在master节点
hostnamectl set-hostname k8s-master #设置master节点的hostname
# 在slave-1节点
hostnamectl set-hostname k8s-slave1 #设置slave1节点的hostname
# 在slave-2节点
hostnamectl set-hostname k8s-slave2 #设置slave2节点的hostname
```

添加hosts解析

``` python
$ cat >>/etc/hosts<<EOF
172.21.32.15 k8s-master
172.21.32.11 k8s-slave1
172.21.32.9 k8s-slave2
EOF
```

- **配置ssh免密**

```bash
# 安装：openssh-server openssh-client
yum -y install openssh-server openssh-client

# 1、主机执行（连续回车），生成密钥对，在~/.ssh/目录下会生成两文件：id_rsa：私钥文件id_rsa.pub：公钥文件
# -t 指定算法，也可指定其他算法
ssh-keygen -t rsa

# 2、将公钥copy到其他机器上即可
# -i : 指定公钥文件
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.0.0.3
 
# 删除免密：执行上诉命令之后会在目标服务器对应用户下的：~/.ssh/authorized_keys 文件中添加一行内容，也就是生成的本地公钥，编辑该文件，删除对应主机的公钥内容，也就删除了对应主机的免密操作
```



### 2. 安装docker

操作节点： 所有节点

- **yum安装**

``` python
 ## 查看所有的可用版本
$ yum list docker-ce --showduplicates | sort -r
##安装旧版本（不同版本k8s要求的docker版本不一样） 
yum install docker-ce-cli-18.09.9-3.el7  docker-ce-18.09.9-3.el7

## 安装源里最新版本
$ yum install docker-ce -y

## 配置docker加速
mkdir -p /etc/docker 
cat << EOF >/etc/docker/daemon.json
{
  "insecure-registries": [
  ],
  "registry-mirrors" : [
    "https://bn6ijjfd.mirror.aliyuncs.com",
    "https://docker.1panel.live",
    "https://www.mirrorify.net/"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

#可能还需要修改docker的driver为systemd模式，k8s支持的模式
#在daemon.json文件中加上重启即可：
"exec-opts": ["native.cgroupdriver=systemd"]

## 启动docker
systemctl enable docker && systemctl start docker

```

- **二进制**

```bash
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
tar -xf docker-19.03.9.tgz
mv docker/* /usr/bin/
```

配置文件

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "insecure-registries": [    
  ],                          
  "registry-mirrors" : [
    "https://bn6ijjfd.mirror.aliyuncs.com"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

配置systemd管理docker

```bash
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
Environment="PATH=/opt/kube/bin:/bin:/sbin:/usr/bin:/usr/sbin"
ExecStart=/usr/bin/dockerd 
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
ExecReload=/bin/kill -s HUP $MAINPID
Restart=always
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

```bash
systemctl daemon-reload
systemctl enable docker --now
systemctl status docker
```



**注意：**1.24版本以上的kubelet已删除docker-shim，不再内部支持CRI，所以需要另外安装cri-docker提供kubelet调用但是如果想继续使用docker的话，可以在kubelet和docker之间加上一个中间层cri-docker。cri-docker是一个支持CRI标准的shim（垫片）。一头通过CRI跟kubelet交互，另一头跟docker api交互，从而间接的实现了kubernetes以docker作为容器运行时。但是这种架构缺点也很明显，调用链更长，效率更低

```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.8/cri-dockerd-0.3.8-3.el7.x86_64.rpm
yum -y localinstall cri-dockerd-0.3.8-3.el7.x86_64.rpm

# 修改启动文件第10行内容，根据当前使用的pause版本进行更改
vim /usr/lib/systemd/system/cri-docker.service
ExecStart=/usr/bin/cri-dockerd --pod-infra-container-image=registry.k8s.io/pause:3.9 --container-runtime-endpoint fd://

systemctl start cri-docker
systemctl enable cri-docker
```

```bash
# 注意（后续安装会提到详细）
# 如果使用cri-dockerd，需要在kubeadm init的时候指定socket
# 1、使用配置文件初始化的话，修改配置文件中的criSocket配置
criSocket: unix:///var/run/cri-dockerd.sock

# 2、使用命令初始化的话，需要加上参数
--cri-socket=unix:///var/run/cri-dockerd.sock
```

### 3. linux内核升级

可选操作

centos 7.x系统自带的3.10.x内核存在一些bugs，导致运行docker、k8s不稳定。可升级内核版本到4.4以上

查看当前的内核版本

```bash
uname -r
3.10.0-1160.el7.x86_64

uname -a
Linux k8s-master 3.10.0-1160.el7.x86_64 #1 SMP Mon Oct 19 16:18:59 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```

ElRepo 是一个分发企业版[Linux内核](https://so.csdn.net/so/search?q=Linux内核&spm=1001.2101.3001.7020)的社区仓库，在系统中导入ElRepo仓库的公钥，后续将从这个仓库中获取升级内核相关的资源。

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org   #导入ElRepo仓库公钥

#Centos 7 安装 ELRepo 最新版本
yum -y install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm  
#Centos 8 安装 ELRepo 最新版本
yum -y install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm  
#Centos 9 安装 ELRepo 最新版本
yum -y install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm   

#查看可用的稳定版镜像
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available    
```

升级内核

```bash
# 安装指定的 kernel 版本(推荐安装带lt的，为长期支持版内核)
yum install -y --enablerepo=elrepo-kernel kernel-lt-5.4.268-1.el7.elrepo 

# 查看系统可用内核
cat /boot/grub2/grub.cfg | grep menuentry
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg 
0 : CentOS Linux (5.4.268-1.el7.elrepo.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
2 : CentOS Linux (0-rescue-f3a9762dc20a41c1ab712c0f21dd77d2) 7 (Core)

# 设置开机从新内核启动
$ grub2-set-default "CentOS Linux (5.4.268-1.el7.elrepo.x86_64) 7 (Core)"

# 查看内核启动项
$ grub2-editenv list
saved_entry=CentOS Linux (5.4.268-1.el7.elrepo.x86_64) 7 (Core)
```

重启服务器，查看内核版本

```bash
unama -r 
5.4.268-1.el7.elrepo.x86_64
```

完成

## 部署etcd集群

​	Etcd 是一个分布式键值存储系统。Kubernetes使用Etcd进行数据存储，所以先准备一个Etcd数据库，为解决Etcd单点故障，应采用集群方式部署，这里使用3台组建集群，可容忍1台机器故障，当然，你也可以使用5台组建集群，可容忍2台机器故障。每个 etcd cluster 都是有若千个 member 组成的，每个 member 是一个独立运行的 etcd 实例单台机器上可以运行多个 member。
​	在正常运行的状态下，集群中会有一个leader，其余的 member 都是 followers。leader向 followers 同步日志，保证数据在各个 member都有副本。leader 还会定时向所有的 member 发送心跳报文，如果在规定的时间里 follower 没有收到心跳，就会重新进行选举。客户端所有的请求都会先发送给 leader，leader 向所有的 followers 同步日志，等收到超过半数的确认后就把该日志存储到磁盘，并返回响应客户端。每个 etcd 服务有三大主要部分组成: raft实现、WAL日志存储、数据的存储和索引。

服务器规划（测试环境为了节省机器，这里与k8s节点复用，也可以部署在k8s机器之外,只要apiserver能连接到就行。）

| 节点名称 | IP           |
| -------- | ------------ |
| etcd1    | 192.168.9.20 |
| etcd2    | 192.168.9.21 |
| etcd3    | 192.168.9.22 |

### cfssl证书生成工具准备

cfssl是一个开源的证书管理工具，使用json文件生成证书，相比openssl更方便使用。
找任意一台服务器操作，这里用Master1节点。

```bash
#创建目录存放cfssl工具
mkdir /software-cfssl
cd /software-cfssl
#下载相关工具
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 

chmod +x ./cfssl*
mv cfssl_linux-amd64 /usr/bin/cfssl
mv cfssljson_linux-amd64 /usr/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

### 自签证书颁发机构(CA)

在 Kubernetes 中，使用自签名根证书颁发机构可以为 Kubernetes API Server、etcd 和 kubelet 等组件提供安全通信。一般情况下，我们需要将自签名根证书 CA.pem 分发给所有工作节点，并将其添加到容器镜像中，以便后续在 Kubernetes 集群中进行节点间的安全通信验证

- 创建工作目录

  ```bash
  mkdir -p ~/TLS/{etcd,k8s}
  cd ~/TLS/etcd/
  ```

- 生成自签CA配置

  ```bash
  cat > ca-config.json << EOF
  {
    "signing": {
      "default": {
        "expiry": "87600h"  
      },
      "profiles": {
        "www": {
           "expiry": "87600h",
           "usages": [
              "signing",            
              "key encipherment",    
              "server auth",        
              "client auth"          
          ]
        }
      }
    }
  }
  EOF
  
  cat > ca-csr.json << EOF
  {
      "CN": "etcd CA",         
      "key": {
          "algo": "rsa",      
          "size": 2048        
      },
      "names": [
          {
              "C": "CN",           
              "L": "ShenZhen",    
              "ST": "ShenZhen"     
          }
      ]
  }
  EOF
  
  ```

  ```bash
  "expiry": "87600h"    #证书有效期
  "signing",             #允许该证书用于签名其他证书
  "key encipherment",    #允许该证书用于加密其他证书的私钥
  "server auth",         #允许该证书用于验证服务器身份
  "client auth"          #允许该证书用于验证客户端身份
  
  
  "CN": "etcd CA",         #指定生成的证书的 Common Name（通用名称）为 "etcd CA"。通用名称通常用于标识证书所属的实体或主体
  "algo": "rsa",       #RSA 密钥类型
  "size": 2048         #密钥大小为 2048 位，密钥越大，安全性越高，但也会增加计算量
  "C": "CN",           #指定证书颁发机构的名称和所在地，国家/地区（Country/C），CN中国
  "L": "ShenZhen",     #城市（Locality/L）
  "ST": "ShenZhen"     #州/省（State/ST）
  ```

  

- 生成自签CA证书

  ```bash
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
  ```

  当前目录下会生成 ca.pem和ca-key.pem文件

  ```bash
  ls .
  ca-config.json   #（定义生成该根证书所需配置信息的 JSON 格式文件，包括有效期、使用场景） 
  ca.csr           #（生成 CA 证书签名请求的文件，包含了需要签名的信息，如组织名和国家代码等）  
  ca-csr.json      #（定义 CA 签名请求的配置信息，如公钥算法和密钥长度等）  
  ca-key.pem       #（根证书私钥文件，用于对由该 CA 签名的证书进行签名）  
  ca.pem           #（根证书公钥文件，用于验证由该 CA 签名的证书的有效性）
  ```

###  使用自签CA签发etcd https证书

- 创建证书申请文件

  文件中hosts字段中ip为所有etcd节点的集群内部通信ip，一个都不能少，为了方便后期扩容可以多写几个预留的ip。

  ```bash
  cat > etct-server-csr.json << EOF
  {
      "CN": "etcd",
      "hosts": [
      "192.168.9.20",
      "192.168.9.21",
      "192.168.9.22",
      "192.168.9.23",
      "192.168.9.24",
      "192.168.9.25",
      "192.168.9.100"
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "L": "ShenZhen",
              "ST": "ShenZhen"
          }
      ]
  }
  EOF
  
  ```
  
  生成证书：
  
  当前目录下会生成 server.pem 和 server-key.pem
  
  ```bash
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www etct-server-csr.json | cfssljson -bare server
  
  # ls
  ca-config.json  ca-csr.json  ca.pem      server-csr.json  server.pem
  ca.csr          ca-key.pem   server.csr  server-key.pem
  ```

### 安装集群

- 下载etcd包文件

```bash
#下载对应版本的二进制包
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
```

- 部署

  以下操作现在一台机子上执行，后续才将配置等相关文件拷贝传到其他节点上

​	解压二进制包，创建工作目录

```bash
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar -xvf etcd-v3.4.9-linux-amd64.tar.gz
mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

​	创建配置文件

```bash
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.9.20:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.9.20:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.9.20:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.9.20:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.9.20:2380,etcd-2=https://192.168.9.21:2380,etcd-3=https://192.168.9.22:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

```

```bash
#注释
ETCD_NAME="etcd-1"      #节点名称，集群中唯一
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"    #数据目录
ETCD_LISTEN_PEER_URLS="https://192.168.242.51:2380"    #集群通讯监听地址
ETCD_LISTEN_CLIENT_URLS="https://192.168.242.51:2379"    #客户端访问监听地址

#[Clustering]
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.242.51:2380,etcd-2=https://192.168.242.52:2380,etcd-3=https://192.168.242.53:2380"    #集群节点地址
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"     #集群token，用于验证 etcd 集群中的节点，只有使用相同的 token 才能加入到同一个 etcd 集群中；用于控制 etcd 集群的访问权限，只有持有正确的 token 才能对 etcd 集群进行操作（例如写入、读取、删除等)
ETCD_INITIAL_CLUSTER_STATE="new"              #加入集群的状态：new是新集群，existing表示加入已有集群
```

​	创建systemd配置文件

```bash
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

将配置文件拷贝到其他两个节点，然后修改配置文件中的节点名称和当前服务器IP

```bash
#[Member]
ETCD_NAME="etcd-1"    #节点2修改为: etcd-2；节点3修改为: etcd-3
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.9.21:2380"  #修改为对应节点IP
ETCD_LISTEN_CLIENT_URLS="https://192.168.9.21:2379"  #修改为对应节点IP

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.9.21:2380" #修改为对应节点IP
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.242.51:2379" #修改为对应节点IP
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.9.20:2380,etcd-2=https://192.168.9.21:2380,etcd-3=https://192.168.9.22:2380"  
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

**启动**

```bash
systemctl daemon-reload
systemctl enable etcd --now
systemctl status etcd
```

**注意：**etcd须多个节点同时启动，不然执行systemctl start etcd会一直卡在前台，连接其他节点，建议通过批量管理工具或者脚本同时启动etcd。

### **检查**集群状态

```bash
#查看集群健康状态
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.9.20:2379,https://192.168.9.21:2379,https://192.168.9.22:2379" endpoint health --write-out=table
+-----------------------------+--------+-------------+-------+
|          ENDPOINT           | HEALTH |    TOOK     | ERROR |
+-----------------------------+--------+-------------+-------+
| https://192.168.9.20:2379 |   true | 67.267851ms |       |
| https://192.168.9.21:2379 |   true | 67.374967ms |       |
| https://192.168.9.22:2379 |   true | 69.244918ms |       |
+-----------------------------+--------+-------------+-------+

#查看集群所有节点
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.9.20:2379,https://192.168.9.21:2379,https://192.168.9.22:2379" endpoint status --write-out=table
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.9.20:2379 | 9bf174a909bf392e |   3.4.9 |  823 kB |      true |      false |         2 |     104996 |             104996 |        |
| https://192.168.9.21:2379 | c2166168b99a5d2e |   3.4.9 |  815 kB |     false |      false |         2 |     104996 |             104996 |        |
| https://192.168.9.22:2379 | 1c11ab51e5e46a8b |   3.4.9 |  819 kB |     false |      false |         2 |     104996 |             104996 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

```

问题排查，查看日志

```bash
journalctl -u etcd
less /var/log/message
```



## 部署master节点

etcd 的 CA 和 kube-apiserver 的 CA 并不是同一个证书颁发机构

###  自签证书颁发机构(CA)

```bash
cat > ~/TLS/k8s/ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ~/TLS/k8s/ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "ShenZhen",
            "ST": "ShenZhen",
            "O": "k8s",      
            "OU": "System"   
        }
    ]
}
EOF

```

```bash
"O": "k8s",       #组织名（Organization/O）
"OU": "System"    #组织单位名（Organizational Unit/OU）
```

生成证书：

```bash
# 目录下会生成 ca.pem 和 ca-key.pem
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

### 安装集群

- 下载二进制包

```bash
# 下载k8s下载地址文件，版本可自选，二进制文件下载地址在文件
https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.23.md   
# 下载的包名为kubernetes-server-linux-amd64.tar.gz
```

- 创建工作目录并解压文件，拷贝执行文件到对应工作目录下

```bash
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
wget https://dl.k8s.io/v1.23.17/kubernetes-server-linux-amd64.tar.gz
tar zxvf kubernetes-server-linux-amd64.tar.gz

# master节点如不作为工作节点可以不用kubelet，如需要也一起拷贝
cp kubernetes/server/bin/{kube-apiserver,kube-scheduler,kube-controller-manager} /opt/kubernetes/bin && cp kubernetes/server/bin/kubectl /usr/bin/
```

#### bootstrap

- 启动TLS bootstrapping机制

​	在安装 k8s worker node 时，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，但网络通信的基本假设是通信双方谁也不信任谁，必须使用CA签发的有效证书才可以。这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。目前主要用于kubelet组件。kube-proxy还是由我们统一颁发一个证书。

**kubelet启动时候向kube-apiserver进行认证过程：**

1. kubelet启动时查找配置的kubeconfig文件
2. 从kubeconfig文件中得到apiserver的URL和认证信息(一般是TLS key和CA签发的证书)
3. 使用认证信息向apiserver进行认证（这里是Bootstrap Token Authentication认证方式，携带一个token去apiserver认证）

**集群管理需要为kubelet做的事：**

1. 为kubernetes集群创建CA和CA-KEY
2. 将CA和CA-KEY给kube-apiserver使用
3. 为每个node的kubelet创建证书和key（强烈建议每个node使用唯一证书，并设置唯一CN，TLS Bootstrapping旨在简化从这一步开始的步骤，因为这是每个kubelet都需要的。）
4. 使用CA-KEY签发kubelet的证书
5. 将kubelet的证书发送给kubelet使用

**TLS Bootstrap初始化过程：**

1. kubelet进程启动
2. kubelet查找kubeconfig，没找到(因为没配置)
3. kubelet查找到bootstrap-kubeconfig文件
4. kubelet通过bootstrap-kubeconfig文件，查找到apiserver的URL和有限制权限的"token"，该"token"可以通过apiserver的认证，且只能向apiserver发送CSR请求
5. kubelet使用"token"向apiserver认证
6. kubelet现在有权限创建一个CSR并发给apiserver
7. kubelet创建一个key和cert，并使用cert创建一个带signerName=kubernetes.io/kube-apiserver-client-kubelet的CSR，并发送给apiserver
8. apiserver签发该CSR：

```bash
如果配置了自动签发，kube-controller-manager将会自动签发该CSR
可以使用集群外的程序或人使用kubectl或Kubernetes API进行签发
```

9. 为kubelet签发证书(一般证书有效期为1年)
10. 将签发好的证书分发给kubelet
11. kubelet得到签发的证书
12. kubelet使用key和签发的证书生成kubeconfig
13. kubelet现在可以正常与apiserver交互了
14. 可选：如果配置了，kubelet将会在证书快过期的时候，自动发起新的CSR请求更新自己的证书
15. apiserver或手动或自动签发kubelet发送的更新证书的CSR请求

<img src="images/image-20240307235822239.png" alt="image-20240307235822239" style="zoom:50%;" />

​	在 apiserver 配置中指定了一个 token.csv 文件，该文件中是一个预设的用户配置；同时该用户的Token 和 由 apiserver 的 CA 签发的用户被写入了 kubelet 所使用的 bootstrap.kubeconfig 配置文件中；这样在首次请求时，kubelet 使用 bootstrap.kubeconfig 中被 apiserver CA 签发证书时信任的用户来与 apiserver 建立 TLS 通讯，使用 bootstrap.kubeconfig 中的用户 Token 来向apiserver 声明自己的 RBAC 授权身份.

创建上述配置文件中token文件：

文件与`kube-apiserver.conf`的`token-auth-file`相对应

**格式**：token,用户名,UID,用户组

```bash
cc9f564a0e4d41272dd2b51615582750,kubelet-bootstrap,10001,"system:node-bootstrapper"
```

创建生成 token.csv 文件

```bash
# token可自行生成替换：
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
cc9f564a0e4d41272dd2b51615582750

# 生成配置
cat > /opt/kubernetes/cfg/token.csv << EOF
cc9f564a0e4d41272dd2b51615582750,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF

```

使用自签CA签发kube-apiserver https证书

#### 部署kube-apiserver

- **创建kube-apiserver csr请求文件：**

配置文件中hosts字段中，为了方便后期扩容，可以多写几个预留的Master/LB/VIP等IP。如果后续新增了ip，只能重新申请证书。

```bash
cat > ~/TLS/k8s/apiserver-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "192.168.9.20",
      "192.168.9.21",
      "192.168.9.22",
      "192.168.9.23",
      "192.168.9.24",
      "192.168.9.25",
      "192.168.9.26",
      "192.168.9.100",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "ShenZhen",
            "ST": "ShenZhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

```

- **生成kube-apiserver证书**

```bash
# 在ca.pem证书目录下执行，当前目录下会生成apiserver.pem 和 apiserver-key.pem文件
# 会有一个WARNING，可以忽略
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes apiserver-csr.json | cfssljson -bare apiserver
```

- **创建聚合证书**

  使用聚合层，用户可以通过附加的API扩展kubernetes，而不局限于kubernetes核心API提供的功能，比如metrics server，或者自己开发的API（CRD）

```bash
# apiserver聚合证书
cat > ~/TLS/k8s/front-proxy-ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
     "algo": "rsa",
     "size": 2048
  },
  "ca": {
    "expiry": "876000h"
  }
}
EOF

cat > ~/TLS/k8s/front-proxy-client-csr.json  <<EOF 
{
  "CN": "front-proxy-client",
  "key": {
     "algo": "rsa",
     "size": 2048
  }
}
EOF

# 生成聚合ca证书
cfssl gencert -initca /root/TLS/k8s/front-proxy-ca-csr.json | cfssljson -bare front-proxy-ca 

# 生成聚合服务器证书
cfssl gencert -ca=/root/TLS/k8s/front-proxy-ca.pem -ca-key=/root/TLS/k8s/front-proxy-ca-key.pem -config=/root/TLS/k8s/ca-config.json -profile=kubernetes /root/TLS/k8s/front-proxy-client-csr.json | cfssljson -bare front-proxy-client
```



- **拷贝生成的证书到所创建的k8s的工作目录下**

```bash
cp ~/TLS/k8s/{ca*pem,apiserver*pem,front-proxy*pem} /opt/kubernetes/ssl/
```

- **创建kube-apiserver的启动参数文件（启动参数较多，所以单独写到一个文件中）**

```bash
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://192.168.9.20:2379,https://192.168.9.21:2379,https://192.168.9.22:2379 \\
--bind-address=0.0.0.0 \\
--secure-port=6443 \\
--advertise-address=192.168.9.20 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/apiserver.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/apiserver-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/apiserver.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/apiserver-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-issuer=https://kubernetes.default.svc.cluster.local \\
--service-account-signing-key-file=/opt/kubernetes/ssl/apiserver-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/front-proxy-ca.pem \\
--proxy-client-cert-file=/opt/kubernetes/ssl/front-proxy-client.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/front-proxy-client-key.pem \\
--requestheader-allowed-names=kubernetes \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--enable-aggregator-routing=true \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF

```

参数详细解释：

```bash
--logtostderr=false: 将日志输出到文件而不是标准输出。
--v=2: 日志级别，2 表示 Info 级别。
--log-dir=/opt/kubernetes/logs: 指定日志文件的输出目录。
--etcd-servers=https://192.168.9.20:2379,https://192.168.9.21:2379,https://192.168.9.22:2379: 指定 Kubernetes API Server 用来连接 etcd 集群的地址，多个地址之间用逗号分隔。
--bind-address=192.168.9.20: 指定 API Server 监听的 IP 地址。
--secure-port=6443: 指定 Kubernetes API Server 监听的端口，默认为 6443。
--advertise-address=192.168.9.20: 指定 API Server 的访问地址，用于和其他组件通信。
--allow-privileged=true: 允许容器使用特权模式，默认为 true。
--service-cluster-ip-range=10.0.0.0/24: 指定 Service IP 地址段，通过此 IP 地址段进行 Service 的请求转发。
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction: 启用 Admission Controller 插件，决策 Kubernetes API Server 是否能够创建和更新资源对象。
--authorization-mode=RBAC,Node: 指定授权模式，RBAC 表示使用 Role-Based Access Control 进行授权，Node 表示使用节点名称进行授权。
--enable-bootstrap-token-auth=true: 允许使用 Bootstrap Token 进行身份验证。
--token-auth-file=/opt/kubernetes/cfg/token.csv: 指定 Token 文件的存放路径。
--service-node-port-range=30000-32767: 指定节点端口范围，用于 NodePort 类型的 Service。
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem: 指定 API Server 向 kubelet 发送请求时使用的客户端证书。
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem: 指定客户端证书的私钥。
--tls-cert-file=/opt/kubernetes/ssl/server.pem: 指定 API Server 的 TLS 证书文件。
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem: 指定 TLS 证书的私钥文件。
--client-ca-file=/opt/kubernetes/ssl/ca.pem: 指定客户端证书颁发机构的证书，用于验证客户端证书的有效性。
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem: 指定 Service Account 的私钥文件。
--service-account-issuer=api: 指定发出 Service Account Token 的 Issuer。
--service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem: 指定 Service Account Token 签名密钥文件。
--etcd-cafile=/opt/etcd/ssl/ca.pem: 指定 etcd 集群 CA 证书文件。
--etcd-certfile=/opt/etcd/ssl/server.pem: 指定 etcd 集群客户端证书文件。
--etcd-keyfile=/opt/etcd/ssl/server-key.pem: 指定 etcd 集群客户端证书的私钥文件。
--requestheader-client-ca-file=/opt/kubernetes/ssl/front-proxy-ca.pem: 指定客户端请求 API Server 使用的 CA 证书。
--proxy-client-cert-file=/opt/kubernetes/ssl/front-proxy-client.pem: 指定 kube-proxy 向 API Server 发送请求时使用的客户端证书。
--proxy-client-key-file=/opt/kubernetes/ssl/front-proxy-client-key.pem: 指定客户端证书的私钥。
--requestheader-allowed-names=kubernetes: 指定允许通过请求头的用户和组。
--requestheader-extra-headers-prefix=X-Remote-Extra-: 指定额外的请求头前缀，用于传递额外的信息。
--requestheader-group-headers=X-Remote-Group: 指定保存在请求头中用户所属组信息的键名。
--requestheader-username-headers=X-Remote-User: 指定保存在请求头中用户名信息的键名。
--enable-aggregator-routing=true: 启用 kube-aggregator 插件。
--audit-log-maxage=30: 指定审计日志保留的时间，单位天
--audit-log-maxbackup=3: 指定审计日志保留的个数。
--audit-log-maxsize=100: 指定审计日志文件的最大大小，单位为 MB。
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log: 指定审计日志输出的路径。
```

- **systemd管理apiserver（读取上面的启动参数文件）**

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

```bash
systemctl daemon-reload
systemctl enable kube-apiserver  --now
systemctl status kube-apiserver 

```



#### 部署kube-controller-manager

- **创建kube-controller-manager证书请求文件** 

```bash
cat > ~/TLS/k8s/kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "ShenZhen", 
      "ST": "ShenZhen",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
EOF

```

- **生成kubeconfig证书**

```bash
# 在ca.pem证书目录下执行
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

- **生成kube-controller-manager.kubeconfig文件**

```
#这里只配置了单个apiserver地址，后面从单主扩为多主要改为负载地址，后面讲到
```

```bash
# shell 执行
KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://192.168.9.20:6443"

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
  
# 设置客户端认证参数
kubectl config set-credentials kube-controller-manager \
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}
 
# 设置当前上下文
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

```

**kubeconfig使用介绍**

```bash
Kubernetes 配置文件通常位于 ~/.kube/config，它包含了连接到 Kubernetes 集群所需的信息，如集群的地址、认证凭据、上下文等。
可在/etc/profile添加变量export KUBECONFIG=/root/.kube/config1 更改配置文件引用，source变量即时生效。

以下是 kubectl config 命令中一些常用的子命令和功能：

kubectl config view：查看当前配置文件的内容。可以显示当前使用的上下文、集群、用户等信息。

kubectl config get-contexts：获取所有的上下文列表，显示当前可用的上下文及其所关联的集群和用户。

kubectl config use-context <context-name>：设置当前使用的上下文。通过指定上下文名称，可以切换到不同的 Kubernetes 集群。

kubectl config set-context <context-name> --cluster=<cluster-name> --user=<user-name> --namespace=<namespace>：创建或修改上下文。可以设置不同的集群、用户和命名空间来定位到特定的环境。

kubectl config delete-context <context-name>：删除指定的上下文。

kubectl config set-cluster <cluster-name> --server=<server-url> --certificate-authority=<ca-file>：设置集群连接信息，包括集群名称、服务器地址和证书等。

kubectl config set-credentials <user-name> --username=<username> --password=<password>：设置用户认证凭据，包括用户名和密码。

kubectl config set-credentials <user-name> --client-certificate=<cert-file> --client-key=<key-file>：设置用户认证凭据，使用客户端证书和密钥。

通过 kubectl config 命令，你可以轻松管理和配置 Kubernetes 集群的连接信息、凭据和上下文等内容，方便在不同环境中切换和操作 Kubernetes 集群。
```

-  **systemd管理controller-manager启动文件**

```bash
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-controller-manager \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/front-proxy-ca.pem \\
--cluster-signing-duration=87600h0m0s
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

```

启动参数详解

```bash
--logtostderr=false：将日志输出到文件而非标准输出。
--v=2：设置日志级别为 2，即输出信息和警告，不包括调试和错误级别的日志。
--log-dir=/opt/kubernetes/logs：设置日志输出目录为 /opt/kubernetes/logs。
--leader-elect=true：开启 leader 选举机制，确保只有一个 kube-controller-manager 作为主节点运行。
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig：指定 kube-controller-manager 的配置文件路径，用于从 API Server 获取集群状态。
--bind-address=127.0.0.1：绑定 IP 地址，仅允许本地访问。
--allocate-node-cidrs=true：开启节点 CIDR 分配，可以在集群中自动为每个节点分配一个唯一的 CIDR 子网。
--cluster-cidr=10.244.0.0/16：设置集群 Pod 网络的 CIDR 子网地址。
--service-cluster-ip-range=10.0.0.0/24：设置服务的虚拟 IP 地址范围，为服务提供一个固定的 IP 地址，方便使用者访问和发现服务。
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem：设置集群签名 CA 证书文件路径，用于签署 TLS 证书等。
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem：设置集群签名 CA 私钥文件路径，用于签署 TLS 证书等。
--root-ca-file=/opt/kubernetes/ssl/ca.pem：指定根证书文件路径，用于验证 kubelet 发送的 CSR 请求。
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem：设置 Service Account 的私钥文件路径，用于生成和签署 Service Account Token。
--cluster-signing-duration=87600h0m0s：设置集群签名证书的有效期为 10 年。
```

```bash
systemctl daemon-reload
systemctl enable kube-controller-manager  --now
systemctl status kube-controller-manager
```



#### 部署 kube-scheduler

- 生成kube-scheduler证书 

```bash
# 创建证书请求文件
cat > ~/TLS/k8s/kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "ShenZhen",
      "ST": "ShenZhen",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
EOF

```

生成证书

```bash
# 在ca.pem证书目录下执行
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

```

- 生成kube-scheduler.kubeconfig文件

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://192.168.9.20:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

```

- systemd管理schedule启动文件

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-scheduler \\
--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect \\
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \\
--bind-address=127.0.0.1
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

```

启动参数解释

```bash
--kubeconfig ：连接apiserver配置文件
--leader-elect ：当该组件启动多个时,自动选举(HA)。
```

```bash
systemctl daemon-reload
systemctl enable kube-scheduler  --now
systemctl status kube-scheduler
```

#### 配置kubectl

- 生成kubectl连接集群的csr请求文件

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "ShenZhen",
      "ST": "ShenZhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

```

生成证书

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

```

- 生成kubeconfig文件 

```bash
mkdir /root/.kube -p

KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://192.168.9.20:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

#cluster-admin是k8s内置的具有最高权限的集群角色，对整个集群具有完全权限，不是自定义。
kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}
  
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

```

- 查看集群状态

```bash
# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}
```

命令补全：

官方文档：https://kubernetes.io/zh/docs/reference/kubectl/cheatsheet/

```bash
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion 
source <(kubectl completion bash)
kubectl completion bash > ~/.kube/completion.bash.inc
source '/root/.kube/completion.bash.inc' 
source $HOME/.bash_profile
```



## 部署work节点

在master节点上操作，即当前Master节点，也可当Work节点

- 创建工作目录并拷贝二进制文件：kubelet、 kube-proxy（所有work节点执行）

```bash
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
```

```bash
# master上执行，拷贝文件到work节点
scp ~/package/kubernetes/server/bin/{kubelet,kube-proxy} root@192.168.9.21:/opt/kubernetes/bin/

# 存在多个节点可快速执行
for i in {1..3}; do scp /package/kubernetes/server/bin/kubelet kube-proxy root@192.168.9.2$i:/opt/kubernetes/bin/; done

```

### **部署kubelet**

- 创建服务配置文件（要安装的机器）

  ```bash
  cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
  kind: KubeletConfiguration
  apiVersion: kubelet.config.k8s.io/v1beta1
  address: 192.168.9.20        # 改为本机的ip
  port: 10250
  readOnlyPort: 10255
  cgroupDriver: systemd          # 要和 docker 的驱动一致
  clusterDNS:
  - 10.0.0.2
  clusterDomain: cluster.local 
  failSwapOn: false
  authentication:
    anonymous:
      enabled: false
    webhook:
      cacheTTL: 2m0s
      enabled: true
    x509:
      clientCAFile: /opt/kubernetes/ssl/ca.pem 
  authorization:
    mode: Webhook
    webhook:
      cacheAuthorizedTTL: 5m0s
      cacheUnauthorizedTTL: 30s
  evictionHard:
    imagefs.available: 15%
    memory.available: 100Mi
    nodefs.available: 10%
    nodefs.inodesFree: 5%
  maxOpenFiles: 1000000
  maxPods: 110
  EOF
  
  ```

- systemd管理kubelet

  ```bash
  mkdir /var/lib/kubelet
  cat >/usr/lib/systemd/system/kubelet.service <<EOF
  [Unit]
  Description=Kubernetes Kubelet
  After=docker.service
  
  [Service]
  WorkingDirectory=/var/lib/kubelet
  ExecStart=/opt/kubernetes/bin/kubelet \\
  --logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --hostname-override=k8s-master1 \\
  --network-plugin=cni \\
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
  --bootstrap-kubeconfig=/opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig \\
  --config=/opt/kubernetes/cfg/kubelet-config.yml \\
  --cert-dir=/opt/kubernetes/ssl \\
  --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  EOF
  
  ```

- 启动参数解释

  ```bash
  # 注释
  --hostname-override ：显示名称,集群唯一(不可重复)，可设置为节点的hostname
  --network-plugin ：启用CNI。
  --kubeconfig ： 空路径,会自动生成,后面用于连接apiserver。
  --bootstrap-kubeconfig ：首次启动向apiserver申请证书
  --config ：配置文件参数。
  --cert-dir ：kubelet证书目录。
  --pod-infra-container-image ：管理Pod网络容器的镜像 init container
  ```

**创建证书**

#### 手动approved

- 在master1上生成kubelet初次加入集群的引导kubeconfig文件

  ```bash
  # 在master上命令行直接执行
  KUBE_CONFIG="/opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig"
  KUBE_APISERVER="https://192.168.9.20:6443"  # apiserver IP:PORT
  TOKEN=$(awk -F "," '{print $1}' /opt/kubernetes/cfg/token.csv) #与token.csv里一致(/opt/kubernetes/cfg/token.csv) 
  
  # 生成 kubelet bootstrap kubeconfig 配置文件
  kubectl config set-cluster kubernetes \
  	--certificate-authority=/opt/kubernetes/ssl/ca.pem \
  	--embed-certs=true \
  	--server=${KUBE_APISERVER} \
  	--kubeconfig=${KUBE_CONFIG}
    
  kubectl config set-credentials "kubelet-bootstrap" \
  	--token=${TOKEN} --kubeconfig=${KUBE_CONFIG}
    
  kubectl config set-context default \
  	--cluster=kubernetes \
  	--user=kubelet-bootstrap \
  	--kubeconfig=${KUBE_CONFIG}
    
  kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
  
  ```

- 授权kubelet-bootstrap用户允许请求证书

  ```bash
  kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
  
  ```

- 拷贝bootstrap.kubeconfig文件到节点上

  ```bash
  for i in {1..3}; do scp /opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig root@192.168.9.2$i:/opt/kubernetes/cfg; done
  
  ```

- 启动

  ```bash
  systemctl daemon-reload
  systemctl enable kubelet  --now
  systemctl status kubelet
  ```

- 手动允许kubelet申请证书并加入集群

  ```bash
  # worker节点向master节点发送了一个 CSR 请求
  # kubectl get csr
  NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
  node-csr-KbHieprZUMOvTFMHGQ1RNTZEhsSlT5X6wsh2lzfUry4   107s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
  ## 状态为pending，表示并未批准，状态为Approved,Issued，表示已经批准
  
  #批准kubelet节点申请
  # kubectl certificate approve  node-csr-KbHieprZUMOvTFMHGQ1RNTZEhsSlT5X6wsh2lzfUry4
  certificatesigningrequest.certificates.k8s.io/node-csr-KbHieprZUMOvTFMHGQ1RNTZEhsSlT5X6wsh2lzfUry4 approved #代表批准成功
  
  #查看申请
  # kubectl get csr
  NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
  node-csr-KbHieprZUMOvTFMHGQ1RNTZEhsSlT5X6wsh2lzfUry4   2m35s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
  
  #查看节点，由于网络插件还没有部署,节点会没有准备就绪NotReady
  # kubectl get nodes
  NAME          STATUS     ROLES    AGE     VERSION
  k8s-master1   NotReady   <none>   2m11s   v1.20.10
  ```


#### **自动approved申请证书**

在安装Kubernetes 时，我们需要为每一个工作节点上的 Kubelet 分别生成一个证书。由于工作节点可能很多，手动生成 Kubelet 证书的过程会比较繁琐。

为了解决这个问题，Kubernetes 提供了 TLS bootstrapping 的方式来简化 Kubelet 证书的生成过程。其原理是预先提供一个 bootstrapping token，kubelet 通过该 kubelet 调用 kube-apiserver 的证书签发 API 来生成自己需要的证书。

**采用TLS bootstrapping 生成证书的流程如下：**

- 调用 kube-apiserver 生成一个 bootstrap token。
- 将该 bootstrap token 写入到一个 kubeconfig 文件中，作为 kubelet 调用 kube-apiserver 的客户端验证方式。
- 通过--bootstrap-kubeconfig 启动参数将 bootstrap token 传递给 kubelet 进程。
- Kubelet 采用bootstrap token 调用 kube-apiserver API，生成自己所需的服务器和客户端证书。
- 证书生成后，Kubelet 采用生成的证书和 kube-apiserver 进行通信，并删除本地的 kubeconfig 文件，以避免 bootstrap token 泄漏风险。

创建证书

- 随机生成token

```bash
openssl rand -hex 3 | tr -d '\n' && echo -n '.' && openssl rand -hex 8
8ecfba.d406f053c3969668
```

- 创建kubelet-bootstrap.kubeconfig证书文件

```bash
# 在master上命令行直接执行
KUBE_CONFIG="/opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig"
KUBE_APISERVER="https://192.168.9.20:6443"  # apiserver IP:PORT
TOKEN="8ecfba.d406f053c3969668"

# 设置集群参数, 生成 kubelet bootstrap kubeconfig 配置文件
kubectl config set-cluster kubernetes \
	--certificate-authority=/opt/kubernetes/ssl/ca.pem \
	--embed-certs=true \
	--server=${KUBE_APISERVER} \
	--kubeconfig=${KUBE_CONFIG}
	
# 设置一个用户
kubectl config set-credentials tls-bootstrap-token-user \
	--token=${TOKEN} --kubeconfig=${KUBE_CONFIG}
  
# 设置上下文参数
kubectl config set-context tls-bootstrap-token-user@kubernetes --cluster=kubernetes \
--user=tls-bootstrap-token-user --kubeconfig=${KUBE_CONFIG}

# 设置默认上下文环境
kubectl config use-context tls-bootstrap-token-user@kubernetes \
--kubeconfig=${KUBE_CONFIG}

```

- 拷贝bootstrap.kubeconfig文件到节点上

```
 for i in {1..3}; do scp /opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig root@192.168.9.2$i:/opt/kubernetes/cfg; done
 
```

- 创建secret（注意name、token-id与token-secret的值取值token）

  ```bash
  cat >bootstrap.secret.yaml <<EOF
  apiVersion: v1
  kind: Secret
  metadata:
    name: bootstrap-token-8ecfba   # 格式必须是bootstrap-token-token的前六位
    namespace: kube-system
  type: bootstrap.kubernetes.io/token
  stringData:
    description: "The default bootstrap token generated by 'kubelet '."
    token-id: 8ecfba                   # 截取token的前六位
    token-secret: d406f053c3969668     # token的后16位
    usage-bootstrap-authentication: "true"
    usage-bootstrap-signing: "true"
    auth-extra-groups:  system:bootstrappers:default-node-token,system:bootstrappers:worker,system:bootstrappers:ingress
   
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: kubelet-bootstrap
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:node-bootstrapper
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: system:bootstrappers:default-node-token
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: node-autoapprove-bootstrap
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: system:bootstrappers:default-node-token
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: node-autoapprove-certificate-rotation
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: system:nodes
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    annotations:
      rbac.authorization.kubernetes.io/autoupdate: "true"
    labels:
      kubernetes.io/bootstrapping: rbac-defaults
    name: system:kube-apiserver-to-kubelet
  rules:
    - apiGroups:
        - ""
      resources:
        - nodes/proxy
        - nodes/stats
        - nodes/log
        - nodes/spec
        - nodes/metrics
      verbs:
        - "*"
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: system:kube-apiserver
    namespace: ""
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:kube-apiserver-to-kubelet
  subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: kube-apiserver
  EOF
  
  kubectl apply -f bootstrap.secret.yaml
  ```

- 修改kube-apiserver配置文件

  ```bash
  # 将kube-apiserver中的--token-auth-file配置项删除，重启
  --token-auth-file=/opt/kubernetes/cfg/token.csv
  ```

- 启动

  ```bash
  systemctl daemon-reload
  systemctl enable kubelet  --now
  systemctl status kubelet
  ```

此后节点再加入集群，就不用手动approved进行批准



### 部署kube-proxy

- 生成kube-proxy证书文件

  ```bash
  # 创建证书请求文件
  cat > ~/TLS/k8s/kube-proxy-csr.json << EOF
  {
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "L": "ShenZhen",
        "ST": "ShenZhen",
        "O": "k8s",
        "OU": "System"
      }
    ]
  }
  EOF
  
  
  # 生成证书
  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
  
  ```

- 生成kube-proxy.kubeconfig文件

  ```bash
  # 在master上shell终端直接执行
  KUBE_CONFIG="/opt/kubernetes/cfg/kube-proxy.kubeconfig"
  KUBE_APISERVER="https://192.168.9.20:6443"
  
  kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=${KUBE_CONFIG}
    
  kubectl config set-credentials kube-proxy \
    --client-certificate=./kube-proxy.pem \
    --client-key=./kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=${KUBE_CONFIG}
    
  kubectl config set-context default \
    --cluster=kubernetes \
    --user=kube-proxy \
    --kubeconfig=${KUBE_CONFIG}
    
  kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
  
  
  
  SECRET=$(kubectl -n kube-system get sa/kube-proxy \
       --output=jsonpath='{.secrets[0].name}')
  JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
       --output=jsonpath='{.data.token}' | base64 -d)
  
  # 设置集群参数
  kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem  \
     --embed-certs=true \
     --server=https://192.168.137.88:16443 \
     --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
  
  # 设置一个用户
  kubectl config set-credentials kubernetes --token=${JWT_TOKEN} --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig
  
  # 设置上下文参数
  kubectl config set-context kubernetes --cluster=kubernetes --user=kubernetes --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig
  
  # 设置默认上下文环境
  kubectl config use-context kubernetes --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig
  
  
  ```

- 创建服务配置文件

  ```bash
  cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  bindAddress: 0.0.0.0
  hostnameOverride: k8s-master1        #指定 kube-proxy 的主机名
  clientConnection:
    acceptContentTypes: ""
    burst: 10
    contentType: application/vnd.kubernetes.protobuf
    kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
    qps: 5
  clusterCIDR: 10.244.0.0/16    #与kube-controller-manager.conf中的参数--cluster-cidr 的值相同，为集群Pod网络的CIDR子网地址
  configSyncPeriod: 15m0s
  conntrack:
    max: null
    maxPerCore: 32768
    min: 131072
    tcpCloseWaitTimeout: 1h0m0s
    tcpEstablishedTimeout: 24h0m0s
  enableProfiling: false
  healthzBindAddress: 0.0.0.0:10256
  hostnameOverride: ""
  iptables:
    masqueradeAll: false
    masqueradeBit: 14
    minSyncPeriod: 0s
    syncPeriod: 30s
  ipvs:
    masqueradeAll: true
    minSyncPeriod: 5s
    scheduler: "rr"
    syncPeriod: 30s
  kind: KubeProxyConfiguration
  metricsBindAddress: 192.168.9.20:10249    #指定 kube-proxy 暴露的 Prometheus 监控指标的地址和端口号。
  mode: "ipvs"
  nodePortAddresses: null
  oomScoreAdj: -999
  portRange: ""
  udpIdleTimeout: 250ms
  EOF
  
  ```

- systemd管理kube-proxy

  ```bash
  cat > /usr/lib/systemd/system/kube-proxy.service << EOF
  [Unit]
  Description=Kubernetes Proxy
  After=network.target
  
  [Service]
  ExecStart=/opt/kubernetes/bin/kube-proxy \\
  --logtostderr=false \\
  --v=2 \\
  --log-dir=/opt/kubernetes/logs \\
  --config=/opt/kubernetes/cfg/kube-proxy-config.yml
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  EOF
  
  ```

  ```bash
  systemctl daemon-reload
  systemctl enable kube-proxy --now
  systemctl status kube-proxy
  
  ```
  
  

### 新加入Work节点

- 拷贝以部署好的相关文件到新节点

  在Master节点将涉及文件拷贝到新work节点

  ```bash
  # copy ca.pem证书文件
  for i in {1..3}; do scp -r /opt/kubernetes/ssl/ca.pem root@192.168.9.2$i:/opt/kubernetes/ssl/; done
  
  # 拷贝kube-proxy，kubelet的配置文件、启动参数文件以及到work节点上，注意修改kubelet-config.yml配置文件中的主机ip
  # 1、kubelet正式申请文件：bootstrap.kubeconfig
  # 2、配置文件：kubelet-config.yml，kube-proxy.conf
  # 3、kubeproxy证书文件：kube-proxy.kubeconfig，kubelet的证书文件不用拷，证书申请审批后自动生成的，每个Node不同
  for i in {1..3}; do scp /opt/kubernetes/cfg/{bootstrap.kubeconfig,kubelet-config.yml,kube-proxy-config.yml,kube-proxy.kubeconfig} root@192.168.9.2$i:/opt/kubernetes/cfg/; done
  
  # 拷贝启动文件
  for i in {1..3}; do scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@192.168.9.2$i:/usr/lib/systemd/system; done
  ```

  注意：kubelet证书和kubeconfig文件这几个文件是证书申请审批后自动生成的，每个Node不同,必须删除。

  ```bash
  rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
  rm -f /opt/kubernetes/ssl/kubelet*
  ```

- 修改复制文件中kube-proxy配置文件的主机名以及ip地址

  ```bash
  vi /opt/kubernetes/cfg/kubelet.conf
      hostname-override=k8s-node1
      bindAddress: 192.168.9.21
      metricsBindAddress: 192.168.9.21:10249     #指定 kube-proxy 暴露的 Prometheus 监控指标的地址和端口号。
  ```

  修改复制文件中kubelet配置文件的ip地址

  ```bash
  vi /opt/kubernetes/cfg/kubelet-config.yml
  	address: 192.168.9.21  # 配置为本机ip
  ```

  ```bash
  mkdir /var/lib/kubelet
  systemctl daemon-reload
  systemctl enable kubelet kube-proxy  --now
  systemctl status kubelet kube-proxy
  ```

- 在Master上同意新的Node kubelet证书申请

  ```bash
  #查看证书请求, #状态为pending，表示并未批准，状态为Approved,Issued，表示已经批准
  # kubectl get csr
  NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
  node-csr-gJj8NMSC5zsZeDsP8399sL3VUdJ7yzge6aClOhcBCGU   21m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
  node-csr-vD_u995-W1oEuBQayCKQbKuxg2JVoJKK3kUqTQizupE   21m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
  
  #执行批准命令
  # kubectl certificate approve node-csr-Kn9HdMY0Tw3nQukukh68_YTqkBBIc504X4g0rpaNuNg 
  certificatesigningrequest.certificates.k8s.io/node-csr-Kn9HdMY0Tw3nQukukh68_YTqkBBIc504X4g0rpaNuNg  approved   
  
  # 再次查看变为Approved,Issued状态，表示批准成功
  # kubectl get csr
  NAME                                                   AGE    SIGNERNAME                                    REQUESTOR   ION   CONDITION
  node-csr-4lumCNEGzPGfq_C1qFOHw2HCbubXx8-pmAs7N7FV7eA   112s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-boot      Approved,Issued
  node-csr-Gcp6-FcVu4sSEdPwGu4bEMAVu4A2E0ww-QlcoPbTVxQ   63s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-boot      Approved,Issued
  
  ```

- 查看Node状态，此时看到已经加入了work节点（还没有部署网络组件所以还是NotReady状态）

  ```bash
  # kubectl get nodes
  NAME          STATUS   ROLES    AGE   VERSION
  k8s-node1   NotReady   <none>   11h   v1.23.14
  k8s-node2   NotReady   <none>   11h   v1.23.14
  ```



### 部署网络组件calico

Calico是一个纯三层的数据中心网络方案，k8s主流的网络方案

- 下载calico.yaml，版本按需替换

  ```bash
  wget https://docs.projectcalico.org/v3.23/manifests/calico.yaml --no-check-certificate
  ```

  ```bash
  # 版本对应关系
  Kubernetes 版本    Calico 版本 
  1.18、1.19、1.20    	3.18    	
  https://projectcalico.docs.tigera.io/archive/v3.18/getting-started/kubernetes/requirements    
  https://projectcalico.docs.tigera.io/archive/v3.18/manifests/calico.yaml
  
  1.19、1.20、1.21    	3.19    	
  https://projectcalico.docs.tigera.io/archive/v3.19/getting-started/kubernetes/requirements    
  https://projectcalico.docs.tigera.io/archive/v3.19/manifests/calico.yaml
  
  1.19、1.20、1.21    	3.20    	
  https://projectcalico.docs.tigera.io/archive/v3.20/getting-started/kubernetes/requirements    
  https://projectcalico.docs.tigera.io/archive/v3.20/manifests/calico.yaml
  
  1.20、1.21、1.22    	3.21    	
  https://projectcalico.docs.tigera.io/archive/v3.21/getting-started/kubernetes/requirements    
  https://projectcalico.docs.tigera.io/archive/v3.21/manifests/calico.yaml
  
  1.21、1.22、1.23    	3.22    	
  https://projectcalico.docs.tigera.io/archive/v3.22/getting-started/kubernetes/requirements    
  https://projectcalico.docs.tigera.io/archive/v3.22/manifests/calico.yaml
  
  1.21、1.22、1.23    	3.23    	
  https://projectcalico.docs.tigera.io/archive/v3.23/getting-started/kubernetes/requirements    
  https://projectcalico.docs.tigera.io/archive/v3.23/manifests/calico.yaml
  
  1.22、1.23、1.24、1.25    	3.24    	
  https://projectcalico.docs.tigera.io/archive/v3.24/getting-started/kubernetes/requirements    
  https://projectcalico.docs.tigera.io/archive/v3.24/manifests/calico.yaml
  ```

  **清单文件一些配置详解：**

  该清单文件安装了以下Kubernetes资源：

  - 使用DaemonSet在每个主机上安装calico/node容器；

  - 使用DaemonSet在每个主机上安装Calico CNI二进制文件和网络配置；

  - 使用Deployment运行calico/kube-controller；

  - Secert/calico-etcd-secrets提供可选的Calico连接到etcd的TLS密钥信息；

  - ConfigMap/calico-config提供安装Calico时的配置参数。

    

- 修改配置文件

  **1、calico_backend: "bird"**

  设置Calico使用的后端机制。支持值：
  	**bird：**开启BIRD功能，根据Calico-Node的配置来决定主机的网络实现是采用BGP路由模式还是IPIP、VXLAN覆盖网络模式。这个是默认的模式。
  	**vxlan：**纯VXLAN模式，仅能够使用VXLAN协议的覆盖网络模式。

  ```yaml
    # Configure the backend to use.
    calico_backend: "bird"
  ```

  

  **2、修改配置文件中定义的Pod网络（CALICO_IPV4POOL_CIDR）**

  提醒：此项用于设置安装Calico时要创建的默认IPv4池，Pod IP将从该范围中选择，Calico安装完成后修改此值将再无效。

  默认情况下calico.yaml中"CALICO_IPV4POOL_CIDR"是注释的，如果kube-controller-manager的"--cluster-cidr"不存在任何值的话，则通常取默认值"192.168.0.0/16,172.16.0.0/16，..，172.31.0.0/16"。
  **网段替换成上方kube-controller-manager配置文件指定的cluster-cidr网段一样**

  ```yaml
  ...
            - name: CALICO_IPV4POOL_IPIP
                value: "Always"
              # Set MTU for tunnel device used if ipip is enabled
              - name: FELIX_IPINIPMTU
                valueFrom:
                  configMapKeyRef:
                    name: calico-config
                    key: veth_mtu
              # The default IPv4 pool to create on startup if none exists. Pod IPs will be
              # chosen from this range. Changing this value after installation will have
              # no effect. This should fall within `--cluster-cidr`.
              - name: CALICO_IPV4POOL_CIDR
                value: "10.244.0.0/16"
  ...
  ```

  **3、默认的Calico清单文件中所使用的镜像来源于docker.io国外镜像源，上面我们配置了Docker镜像加速，应删除docker.io前缀以使镜像从国内镜像加速站点下载。**

  ```bash
  # cat calico.yaml |grep 'image:'
  image: docker.io/calico/cni:v3.23.0
  image: docker.io/calico/cni:v3.23.0
  image: docker.io/calico/node:v3.23.0
  image: docker.io/calico/kube-controllers:v3.23.0
  
  # sed -i 's#docker.io/##g' calico.yaml
  # cat calico.yaml |grep 'image:'
  image: calico/cni:v3.23.0
  image: calico/cni:v3.23.0
  image: calico/node:v3.23.0
  image: calico/kube-controllers:v3.23.0
  ```

  其它的不需要更改，默认就好。

- **应用calico.yaml**

  ```bash
  kubectl apply -f calico.yaml
  kubectl get pods -n kube-system
  ```

  ```bash
  # kubectl get pods -n kube-system
  NAME                                      READY   STATUS    RESTARTS   AGE
  calico-kube-controllers-97769f7c7-zcz5d   1/1     Running   0          3m11s
  calico-node-5tnll                         1/1     Running   0          3m11s
  
  # kubectl get nodes
  NAME          STATUS   ROLES    AGE   VERSION
  k8s-master1   Ready    <none>   21m   v1.20.10
  ```

- **授权apiserver访问kubelet**

  ```bash
  cat > apiserver-to-kubelet-rbac.yaml << EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    annotations:
      rbac.authorization.kubernetes.io/autoupdate: "true"
    labels:
      kubernetes.io/bootstrapping: rbac-defaults
    name: system:kube-apiserver-to-kubelet
  rules:
    - apiGroups:
        - ""
      resources:
        - nodes/proxy
        - nodes/stats
        - nodes/log
        - nodes/spec
        - nodes/metrics
        - pods/log
      verbs:
        - "*"
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: system:kube-apiserver
    namespace: ""
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:kube-apiserver-to-kubelet
  subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: kubernetes
  EOF
  
  kubectl apply -f apiserver-to-kubelet-rbac.yaml
  ```



- 有报错，暂未解决，报错信息

现象：calico组件CrashLoopBackOff

```bash
[root@k8s-master1 install-cni]# kubectl get po -nkube-system
NAME                                       READY   STATUS                  RESTARTS         AGE
calico-kube-controllers-595b58b579-j4czr   0/1     Pending                 0                44m
calico-node-bc8zj                          0/1     Init:CrashLoopBackOff   13 (2m46s ago)   44m
```

pod日志报错：

```bash
cat /var/log/pods/kube-system_calico-node-mkh8t_e6766737-dbe8-47e0-8bdd-d5fbebe8ce1c/install-cni/10.log
{"log":"2024-05-14 01:28:21.413 [INFO][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: Installed /host/opt/cni/bin/portmap\n","stream":"stderr","time":"2024-05-14T01:28:21.413966688Z"}
{"log":"2024-05-14 01:28:21.419 [INFO][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: File is already up to date, skipping file=\"/host/opt/cni/bin/tuning\"\n","stream":"stderr","time":"2024-05-14T01:28:21.422272417Z"}
{"log":"2024-05-14 01:28:21.419 [INFO][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: Installed /host/opt/cni/bin/tuning\n","stream":"stderr","time":"2024-05-14T01:28:21.422327019Z"}
{"log":"2024-05-14 01:28:21.419 [INFO][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: Wrote Calico CNI binaries to /host/opt/cni/bin\n","stream":"stderr","time":"2024-05-14T01:28:21.422338264Z"}
{"log":"\n","stream":"stderr","time":"2024-05-14T01:28:21.422344706Z"}
{"log":"2024-05-14 01:28:21.498 [INFO][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: CNI plugin version: v3.24.5\n","stream":"stderr","time":"2024-05-14T01:28:21.499169191Z"}
{"log":"\n","stream":"stderr","time":"2024-05-14T01:28:21.499226331Z"}
{"log":"2024-05-14 01:28:21.499 [INFO][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: /host/secondary-bin-dir is not writeable, skipping\n","stream":"stderr","time":"2024-05-14T01:28:21.499234243Z"}
{"log":"W0514 01:28:21.499123       1 client_config.go:617] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.\n","stream":"stderr","time":"2024-05-14T01:28:21.499358393Z"}
{"log":"2024-05-14 01:28:21.521 [ERROR][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: Unable to create token for CNI kubeconfig error=Unauthorized\n","stream":"stderr","time":"2024-05-14T01:28:21.521444957Z"}
{"log":"2024-05-14 01:28:21.521 [FATAL][1] cni-installer/\u003cnil\u003e \u003cnil\u003e: Unable to create token for CNI kubeconfig error=Unauthorized\n","stream":"stderr","time":"2024-05-14T01:28:21.521480757Z"}

```

kube-proxy报错：

```bash
# systemctl status kube-proxy
kube-proxy.service - Kubernetes Proxy
   Loaded: loaded (/usr/lib/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-05-14 09:01:12 CST; 32min ago
 Main PID: 62889 (kube-proxy)
    Tasks: 7
   Memory: 11.2M
   CGroup: /system.slice/kube-proxy.service
           └─62889 /opt/kubernetes/bin/kube-proxy --logtostderr=false --v=2 --log-dir=/opt/kubernetes/logs --config=/opt/kubernetes/cfg/kube-proxy-config.yml

May 14 09:01:12 k8s-master1 systemd[1]: Started Kubernetes Proxy.
May 14 09:01:12 k8s-master1 kube-proxy[62889]: Flag --logtostderr has been deprecated, will be removed in a future release, see https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/2845-deprecate-klog-specific-flags-in-k8s-components
May 14 09:01:12 k8s-master1 kube-proxy[62889]: Flag --log-dir has been deprecated, will be removed in a future release, see https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/2845-deprecate-klog-specific-flags-in-k8s-components
May 14 09:01:12 k8s-master1 kube-proxy[62889]: E0514 09:01:12.326220   62889 proxier.go:379] "Can't set sysctl, kernel version doesn't satisfy minimum version requirements" sysctl="net/ipv4/vs/conn_reuse_mode" minimumKernelVersion="4.1"
May 14 09:01:12 k8s-master1 kube-proxy[62889]: E0514 09:01:12.327005   62889 proxier.go:379] "Can't set sysctl, kernel version doesn't satisfy minimum version requirements" sysctl="net/ipv4/vs/conn_reuse_mode" minimumKernelVersion="4.1"
```

kubelet报错：

```bash
[root@k8s-master1 install-cni]# systemctl status kubelet >kubelet.log
[root@k8s-master1 install-cni]# cat kubelet.log 
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-05-14 06:48:16 CST; 2h 47min ago
 Main PID: 7555 (kubelet)
    Tasks: 14
   Memory: 158.4M
   CGroup: /system.slice/kubelet.service
           └─7555 /opt/kubernetes/bin/kubelet --logtostderr=false --v=2 --log-dir=/opt/kubernetes/logs --hostname-override=k8s-master1 --network-plugin=cni --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig --bootstrap-kubeconfig=/opt/kubernetes/cfg/kubelet-bootstrap.kubeconfig --config=/opt/kubernetes/cfg/kubelet-config.yml --cert-dir=/opt/kubernetes/ssl --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6

May 14 09:34:46 k8s-master1 kubelet[7555]: E0514 09:34:46.598622    7555 kubelet.go:2347] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
May 14 09:34:51 k8s-master1 kubelet[7555]: E0514 09:34:51.611976    7555 kubelet.go:2347] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
May 14 09:34:51 k8s-master1 kubelet[7555]: E0514 09:34:51.974279    7555 pod_workers.go:919] "Error syncing pod, skipping" err="failed to \"StartContainer\" for \"install-cni\" with CrashLoopBackOff: \"back-off 5m0s restarting failed container=install-cni pod=calico-node-mkh8t_kube-system(e6766737-dbe8-47e0-8bdd-d5fbebe8ce1c)\"" pod="kube-system/calico-node-mkh8t" podUID=e6766737-dbe8-47e0-8bdd-d5fbebe8ce1c

```

journalctl -f报错信息：

```bash
# journalctl -f
-- Logs begin at Tue 2024-05-14 06:47:35 CST. --
May 14 09:33:57 k8s-master1 kube-controller-manager[5213]: I0514 09:33:57.706580    5213 node_lifecycle_controller.go:868] Node k8s-master2 is NotReady as of 2024-05-14 09:33:57.706574115 +0800 CST m=+9961.472114546. Adding it to the Taint queue.
May 14 09:33:57 k8s-master1 kube-controller-manager[5213]: I0514 09:33:57.706613    5213 node_lifecycle_controller.go:868] Node k8s-master3 is NotReady as of 2024-05-14 09:33:57.70660929 +0800 CST m=+9961.472149720. Adding it to the Taint queue.
May 14 09:33:57 k8s-master1 kube-controller-manager[5213]: I0514 09:33:57.706644    5213 node_lifecycle_controller.go:868] Node k8s-node1 is NotReady as of 2024-05-14 09:33:57.706640483 +0800 CST m=+9961.472180892. Adding it to the Taint queue.
May 14 09:33:59 k8s-master1 kubelet[7555]: E0514 09:33:59.976744    7555 pod_workers.go:919] "Error syncing pod, skipping" err="failed to \"StartContainer\" for \"install-cni\" with CrashLoopBackOff: \"back-off 5m0s restarting failed container=install-cni pod=calico-node-mkh8t_kube-system(e6766737-dbe8-47e0-8bdd-d5fbebe8ce1c)\"" pod="kube-system/calico-node-mkh8t" podUID=e6766737-dbe8-47e0-8bdd-d5fbebe8ce1c
May 14 09:34:01 k8s-master1 kubelet[7555]: E0514 09:34:01.363283    7555 kubelet.go:2347] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
May 14 09:34:02 k8s-master1 kube-apiserver[5208]: E0514 09:34:02.127885    5208 authentication.go:63] "Unable to authenticate the request" err="[invalid bearer token, square/go-jose: error in cryptographic primitive]"
May 14 09:34:02 k8s-master1 kube-controller-manager[5213]: I0514 09:34:02.707186    5213 node_lifecycle_controller.go:868] Node k8s-master3 is NotReady as of 2024-05-14 09:34:02.707163973 +0800 CST m=+9966.472704403. Adding it to the Taint queue.
May 14 09:34:02 k8s-master1 kube-controller-manager[5213]: I0514 09:34:02.707287    5213 node_lifecycle_controller.go:868] Node k8s-node1 is NotReady as of 2024-05-14 09:34:02.707278891 +0800 CST m=+9966.472819320. Adding it to the Taint queue.
May 14 09:34:02 k8s-master1 kube-controller-manager[5213]: I0514 09:34:02.707328    5213 node_lifecycle_controller.go:868] Node k8s-master1 is NotReady as of 2024-05-14 09:34:02.707322785 +0800 CST m=+9966.472863215. Adding it to the Taint queue.
May 14 09:34:02 k8s-master1 kube-controller-manager[5213]: I0514 09:34:02.707454    5213 node_lifecycle_controller.go:868] Node k8s-master2 is NotReady as of 2024-05-14 09:34:02.707444335 +0800 CST m=+9966.472984769. Adding it to the Taint queue.
May 14 09:34:06 k8s-master1 kubelet[7555]: E0514 09:34:06.394100    7555 kubelet.go:2347] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
May 14 09:34:07 k8s-master1 kube-controller-manager[5213]: I0514 09:34:07.708579    5213 node_lifecycle_controller.go:868] Node k8s-master1 is NotReady as of 2024-05-14 09:34:07.708568723 +0800 CST m=+9971.474109116. Adding it to the Taint queue.
May 14 09:34:07 k8s-master1 kube-controller-manager[5213]: I0514 09:34:07.708652    5213 node_lifecycle_controller.go:868] Node k8s-master2 is NotReady as of 2024-05-14 09:34:07.70864692 +0800 CST m=+9971.474187314. Adding it to the Taint queue.
May 14 09:34:07 k8s-master1 kube-controller-manager[5213]: I0514 09:34:07.708690    5213 node_lifecycle_controller.go:868] Node k8s-master3 is NotReady as of 2024-05-14 09:34:07.708687876 +0800 CST m=+9971.474228269. Adding it to the Taint queue.
May 14 09:34:07 k8s-master1 kube-controller-manager[5213]: I0514 09:34:07.708715    5213 node_lifecycle_controller.go:868] Node k8s-node1 is NotReady as of 2024-05-14 09:34:07.708713449 +0800 CST m=+9971.474253841. Adding it to the Taint queue.

```





### 部署CoreDNS

coredns是kubernetes的默认DNS服务器。是利用watch Kubernetes的Service和Pod生成NDS记录，然后通过配置kubelet的DNS选项让新启动的Pod使用CoreDNS提供的kubernetes集群内域名解析服务。

- 获取coredns.yaml

  ```bash
  cd kubernetes   #解压后的k8s软件包目录
  tar -zxvf kubernetes-src.tar.gz 
  cp cluster/addons/dns/coredns/coredns.yaml.base  cluster/addons/dns/coredns/coredns.yaml 
  #或者自行下载https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/coredns/coredns.yaml.base
  ```

- 修改coredns.yaml配置文件

  ```bash
  #1:k8s集群后缀名称__DNS__DOMAIN__，一般为cluster.local
  #77行         kubernetes __DNS__DOMAIN__ in-addr.arpa ip6.arpa {
           修改后kubernetes cluster.local in-addr.arpa ip6.arpa {
  
  #2:coredns谷歌地址为dockerhub地址，容易下载
  #142行         image: k8s.gcr.io/coredns/coredns:v1.8.6   
            修改后image: coredns/coredns:1.8.6
            
  #3:pod启动内存限制大小，300Mi即可
  #146行             memory: __DNS__MEMORY__LIMIT__
                     修改后memory: 300Mi
                     
  #4:coredns的svcIP地址，一般为svc网段的第二位，10.100.0.2，第一位为apiserver的svc
  #212行   clusterIP: __DNS__SERVER__
      修改后clusterIP: 10.0.0.2   #此为kubelet-config.yml中的clusterDNS地址
  ```

- 应用coredns.yaml

  ```bash
  [root@k8s-master1 yaml]# kubectl apply -f coredns.yaml 
  
  [root@k8s-master1 yaml]# kubectl get pods -n kube-system
  NAME                                      READY   STATUS    RESTARTS   AGE
  calico-kube-controllers-97769f7c7-zcz5d   1/1     Running   1          47h
  calico-node-5tnll                         1/1     Running   1          47h
  calico-node-m8sdg                         1/1     Running   0          42m
  calico-node-pqvk9                         1/1     Running   0          56m
  coredns-6cc56c94bd-5hvfb                  1/1     Running   0          37s
  ```

- 测试解析

  ```bash
  [root@k8s-master1 yaml]# kubectl run -it --rm dns-test --image=busybox:1.28.4 sh 
  If you don't see a command prompt, try pressing enter.
  / # ns
  nsenter   nslookup
  / # nslookup kubernetes
  Server:    10.0.0.2
  Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local
  ```

## 部署Dashboard

- 获取dashboard.yaml

  ```bash
  cd kubernetes   #解压后的k8s软件包
  路径：cluster/addons/dashboard/dashboard.yaml     #解压kubernetes-src.tar.gz后的路径
  #或者自行下载，版本号按需替换https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml
  ```

-  应用dashboard.yaml

  ```bash
  kubectl apply -f kubernetes-dashboard.yaml
  
  #查看部署情况
  [root@k8s-master1 yaml]#  kubectl get pods,svc -n kubernetes-dashboard
  NAME                                             READY   STATUS    RESTARTS   AGE
  pod/dashboard-metrics-scraper-7b59f7d4df-k49t9   1/1     Running   0          10m
  pod/kubernetes-dashboard-74d688b6bc-l9jz4        1/1     Running   0          10m
  
  NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
  service/dashboard-metrics-scraper   ClusterIP   10.0.0.206   <none>        8000/TCP        10m
  service/kubernetes-dashboard        NodePort    10.0.0.10    <none>        443:30001/TCP   10m
  ```

  访问地址: https://NodeIP:30001

  创建service account并绑定默认cluster-admin管理员集群角色

  ```bash
  kubectl create serviceaccount dashboard-admin -n kube-system
  kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
  kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
  ```

  使用输出的token登陆Dashboard(如访问提示不是私密连接异常,直接输入“thisisunsafe”即可)





## 增加Master节点（高可用）

​		Kubernetes作为容器集群系统，通过健康检查+重启策略实现了Pod故障自我修复能力，通过调度算法实现将Pod分布式部署，并保持预期副本数，根据Node失效状态自动在其他Node拉起Pod，实现了应用层的高可用性。
​		针对Kubernetes集群，高可用性还应包含以下两个层面的考虑：Etcd数据库的高可用性和Kubernetes Master组件的高可用性。 而Etcd我们已经采用3个节点组建集群实现高可用，本节将对Master节点高可用进行说明和实施。
Master节点扮演着总控中心的角色，通过不断与工作节点上的Kubelet和kube-proxy进行通信来维护整个集群的健康工作状态。如果Master节点故障，将无法使用kubectl工具或者API做任何集群管理。Master节点主要有三个服务kube-apiserver、kube-controller-manager和kube-scheduler，其中kube-controller-manager和kube-scheduler组件自身通过选择机制已经实现了高可用，所以Master高可用主要针对kube-apiserver组件，而该组件是以HTTP API提供服务，因此对他高可用与Web服务器类似，增加负载均衡器对其负载均衡即可，并且可水平扩容。

![img](images/3202470-20230527111619885-1832180806.png)

部署Master2

增加一台新服务器，作为Master2和Node节点，IP是192.168.9.23。
Master2 与已部署的Master1所有操作一致。所以我们只需将Master1所有K8s文件拷贝过来，再修改下服务器IP和主机名启动即可。

- 安装Docker(Master1上拷贝文件)

  步骤按照上面docker安装即可



- 创建etcd证书目录(Master2)

  ```bash
  mkdir -p /opt/etcd/ssl
  ```

  拷贝Master1上所有k8s文件和etcd证书到Master2:

  ```bash
  scp -r /opt/kubernetes root@192.168.9.23:/opt
  scp -r /opt/etcd/ssl root@192.168.9.23:/opt/etcd
  scp /usr/lib/systemd/system/kube* root@192.168.9.23:/usr/lib/systemd/system
  scp /usr/bin/kubectl  root@192.168.9.23:/usr/bin
  scp -r ~/.kube root@192.168.9.23:~
  ```

-  删除证书(Master2)

  删除kubelet和kubeconfig文件

  这几个文件是证书申请审批后自动生成的，每个Node不同，必须删除。

  ```bash
  rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
  rm -f /opt/kubernetes/ssl/kubelet*
  ```

- 修改配置文件和主机名(Master2)

  修改apiserver、kubelet和kube-proxy配置文件为本地IP，主机名：

  ```bash
  # 1、 修改kube-apiserver配置文件
  vi /opt/kubernetes/cfg/kube-apiserver.conf 
  ...
  --bind-address=0.0.0.0 \
  --advertise-address=192.168.9.23 \
  ...
  
  # 2、修改kube-controller-manager配置文件
  vi /opt/kubernetes/cfg/kube-controller-manager.kubeconfig
  server: https://192.168.9.23:6443
  
  # 2、修改kube-scheduler配置文件
  vi /opt/kubernetes/cfg/kube-scheduler.kubeconfig
  server: https://192.168.9.23:6443
  
  # 2、修改kubelet配置文件
  # vi /opt/kubernetes/cfg/kubelet.conf
  vim /usr/lib/systemd/system/kubelet.service
  --hostname-override=k8s-master2
  
  # 2、修改kube-proxy配置文件
  vi /opt/kubernetes/cfg/kube-proxy-config.yml
  bindAddress: 192.168.9.23
  metricsBindAddress: 192.168.9.23:10249     #指定 kube-proxy 暴露的 Prometheus 监控指标的地址和端口号。
  hostnameOverride: k8s-master2
  
  # 2、修改证书配置文件
  vi ~/.kube/config
  ...
  server: https://192.168.9.23:6443
  ```

- 启动并设置开机自启(Master2)

  ```bash
  systemctl daemon-reload
  systemctl enable kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy --now
  systemctl status kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy
  ```

- 查看集群状态(Master2)

  ```bash
  kubectl get cs
  NAME                STATUS    MESSAGE             ERROR
  scheduler             Healthy   ok                  
  controller-manager       Healthy   ok                  
  etcd-1               Healthy   {"health":"true"}   
  etcd-2               Healthy   {"health":"true"}   
  etcd-0               Healthy   {"health":"true"}
  ```

- 审批kubelet证书申请

  此部署master2也作为node节点部署kubelet，所以也要批准证书

  ```bash
  # 查看证书请求
  [root@k8s-master1 ~]# kubectl get csr
  NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
  node-csr-EQoVFfTbo6DcvcWfaRzBbMst4BXmdyds99DEYk2oDDE   33m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
  
  # 同意请求指令
  kubectl certificate approve node-csr-EQoVFfTbo6DcvcWfaRzBbMst4BXmdyds99DEYk2oDDE
  
  #代表批准成功
  certificatesigningrequest.certificates.k8s.io/node-csr-EQoVFfTbo6DcvcWfaRzBbMst4BXmdyds99DEYk2oDDE approved
  ```

  ```bash
  # 查看Node
  [root@k8s-master1 ~]# kubectl get nodes
  NAME          STATUS   ROLES    AGE     VERSION
  k8s-master1   Ready    <none>   6d23h   v1.20.10
  k8s-master2   Ready    <none>   9m11s   v1.20.10
  k8s-node1     Ready    <none>   5d      v1.20.10
  k8s-node2     Ready    <none>   5d      v1.20.10
  ```



## 部署Nginx+Keepalived高可用负载均衡器

<img src="images/image-20240309130212468.png" alt="image-20240309130212468" style="zoom: 67%;" />

- Nginx是一个主流Web服务和反向代理服务器，这里用四层实现对apiserver实现负载均衡。
- Keepalived是一个主流高可用软件，基于VIP绑定实现服务器双机热备，在上述拓扑中，Keepalived主要根据Nginx运行状态判断是否需要故障转移（漂移VIP），例如当Nginx主节点挂掉，VIP会自动绑定在Nginx备节点，从而保证VIP一直可用，实现Nginx高可用。
- 如果你是在公有云上，一般都不支持keepalived，那么你可以直接用它们的负载均衡器产品，直接负载均衡多台Master kube-apiserver，架构与上面一样。
- 此部署配置为非抢占模式，主备切换后，主的恢复后，不要主动切回 VIP。

### **安装nginx**

下载一个版本的nginx

```bash
# 下载地址 : http://nginx.org/download/
wget http://nginx.org/download/nginx-1.20.0.tar.gz

# 下载依赖
yum -y install libxml2 libxml2-dev libxslt-devel gd-devel perl-devel perl-ExtUtils-Embed GeoIP GeoIP-devel GeoIP-data openssl openssl-devel gcc make
# 解压
tar -xf nginx-1.20.1.tar.gz
```

 重新编译Nginx，增加Steam模块

```bash
cd nginx-1.20.1/
##检查模块是否支持，比如这次添加 limit 限流模块 和 stream 模块：
./configure –help | grep limit

##ps：-without-http_limit_conn_module disable 表示已有该模块，编译时，不需要添加

./configure –help | grep stream

##ps：–with-stream enable 表示不支持，编译时要自己添加该模块

##根据上面查到已有的模块，加上本次需新增的模块: --with-stream

./configure --prefix=/opt/nginx --sbin-path=/opt/sbin/nginx --modules-path=/opt/nginx/modules --conf-path=/etc/nginx/nginx.conf  --with-stream
make && make install
```

查看Nginx版本模块

```bash
[root@k8s-master2 nginx-1.20.1]# /opt/sbin/nginx -V
nginx version: nginx/1.20.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
configure arguments:--prefix=/opt/nginx --sbin-path=/opt/sbin/nginx --modules-path=/opt/nginx/modules --conf-path=/etc/nginx/nginx.conf  --with-stream
```

配置nginx服务文件

```bash
vim /usr/lib/systemd/system/nginx.service
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/bin/rm -rf /run/nginx.pid
ExecStartPre=/opt/sbin/nginx -t
ExecStart=/opt/sbin/nginx
ExecStop=/opt/sbin/nginx -s stop
ExecReload=/opt/sbin/nginx -s reload
PrivateTmp=true
[Install]
WantedBy=multi-user.target
```

**Nginx配置文件(主备相同)**

```bash
cat > /etc/nginx/nginx.conf << "EOF"
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server 192.168.9.20:6443;   # Master1 APISERVER IP:PORT
       server 192.168.9.23:6443;   # Master2 APISERVER IP:PORT
    }
    
    server {
       listen 16443; # 由于nginx与master节点复用，这个监听端口不能是6443，否则会冲突
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
        }
    }
}
EOF
```

 启动并设置开机自启(master1/master2)

```bash
systemctl daemon-reload
systemctl enable nginx --now
systemctl status nginx 
```



### 安装keepalived

所有master节点上安装

```bash
yum install epel-release -y
#需要用到killall指令做健康检查
yum install keepalived psmisc -y
```

**配置文件**

master1配置

```bash
cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
    vrrp_skip_check_adv_addr
}

vrrp_script check_nginx {
    script "killall -0 nginx"   # 执行脚本
    interval 2   # 检测间隔时间，即两秒检测一次
    fall 2       # 检测失败的最大次数，超过两次认为节点资源发生故障
    rise 2       # 请求两次成功认为节点恢复正常
    weight -50   #一个正整数或负整数。权重值，关系到整个集群角色选举
}

vrrp_instance VI_1 {
    state BACKUP            #非抢占模式都为BACKUP，抢占模式主节点配置MASTER
    interface ens33          #网卡
    virtual_router_id 66    #取值在0-255之间，keepalived集群同一id
    nopreempt               #非抢占模式参数，只在主节点配置
    priority 100            #主备优先级不一致
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass Aa123456
    }
    virtual_ipaddress {
        192.168.9.100/24      #vip,注意填写真实网段
    }
    track_script {
        check_nginx          #对应上方vrrp_script
    }
}
EOF
```

master2配置

```bash
cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
    vrrp_skip_check_adv_addr
}

vrrp_script check_nginx {
    script "killall -0 nginx"   # 执行脚本
    interval 2   # 检测间隔时间，即两秒检测一次
    fall 2       # 检测失败的最大次数，超过两次认为节点资源发生故障
    rise 2       # 请求两次成功认为节点恢复正常
    weight -50   #一个正整数或负整数。权重值，关系到整个集群角色选举
}

vrrp_instance VI_1 {
    state BACKUP            #非抢占模式都为BACKUP，抢占模式主节点配置MASTER
    interface ens33          #网卡
    virtual_router_id 66    #取值在0-255之间，keepalived集群同一id
    priority 90            #主备优先级不一致
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass Aa123456
    }
    virtual_ipaddress {
        192.168.9.100/24      #vip,注意填写真实网段
    }
    track_script {
        check_nginx          #对应上方vrrp_script
    }
}
EOF
```

启动

```bash
systemctl daemon-reload
systemctl enable keepalived --now
systemctl status keepalived 
```



查看keepalived工作状态

```bash
[root@k8s-master1 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:40:1a:d8 brd ff:ff:ff:ff:ff:ff
    inet 192.168.242.51/24 brd 192.168.242.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.242.55/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe40:1ad8/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:f3:e1:d2:e6 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 10.244.159.128/32 brd 10.244.159.128 scope global tunl0
       valid_lft forever preferred_lft forever
5: calia231fca418b@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ecee:eeff:feee:eeee/64 scope lin
```

可以看到，在ens33网卡绑定了192.168.9.100 虚拟IP，说明工作正常。



### Nginx+keepalived高可用测试

关闭主节点Nginx，测试VIP是否漂移到备节点服务器。
在Nginx Master停止nginx: systemctl stop nginx
在Nginx Backup，ip addr命令查看已成功绑定VIP

注：此时backup节点nginx再停止后，vip不会重回master节点，需要手动重启backup的keepalived才可以



### 访问负载均衡器测试

找K8s集群中任意一个节点，使用curl查看K8s版本测试，使用VIP访问：

```bash
[root@k8s-master1 ~]# curl -k https://192.168.9.100:16443/version
{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.10",
  "gitCommit": "8152330a2b6ca3621196e62966ef761b8f5a61bb",
  "gitTreeState": "clean",
  "buildDate": "2021-08-11T18:00:37Z",
  "goVersion": "go1.15.15",
  "compiler": "gc",
  "platform": "linux/amd64"
}[root@k8s-master1 ~]# curl -k https://192.168.9.100:16443/version
^[[A{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.10",
  "gitCommit": "8152330a2b6ca3621196e62966ef761b8f5a61bb",
  "gitTreeState": "clean",
  "buildDate": "2021-08-11T18:00:37Z",
  "goVersion": "go1.15.15",
  "compiler": "gc",
  "platform": "linux/amd64"
}[root@k8s-master1 ~]# curl -k https://192.168.9.100:16443/version
{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.10",
  "gitCommit": "8152330a2b6ca3621196e62966ef761b8f5a61bb",
  "gitTreeState": "clean",
  "buildDate": "2021-08-11T18:00:37Z",
  "goVersion": "go1.15.15",
  "compiler": "gc",
  "platform": "linux/amd64"
}[root@k8s-master1 ~]# curl -k https://192.168.9.100:16443/version
{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.10",
  "gitCommit": "8152330a2b6ca3621196e62966ef761b8f5a61bb",
  "gitTreeState": "clean",
  "buildDate": "2021-08-11T18:00:37Z",
  "goVersion": "go1.15.15",
  "compiler": "gc",
  "platform": "linux/amd64"
```

可以正确获取到K8s版本信息，说明负载均衡器搭建正常。该请求数据流程：curl -> vip(nginx) -> apiserver
通过查看Nginx日志也可以看到转发apiserver IP：

```bash
[root@k8s-master1 ~]# tailf /var/log/nginx/k8s-access.log 
192.168.242.51 192.168.242.51:6443 - [14/Sep/2021:23:53:07 +0800] 200 424
192.168.242.51 192.168.242.54:6443 - [14/Sep/2021:23:53:09 +0800] 200 424
192.168.242.51 192.168.242.51:6443 - [14/Sep/2021:23:53:10 +0800] 200 424
192.168.242.51 192.168.242.54:6443 - [14/Sep/2021:23:53:11 +0800] 200 424
```

### 修改所有的Work Node连接LB VIP

虽然我们增加了Master2 Node和负载均衡器，但是我们是从单Master架构扩容的，也就是说目前所有的Worker Node组件连接都还是Master1 Node，如果不改为连接VIP走负载均衡器，那么Master还是单点故障。

因此要改所有Worker Node（kubectl get node命令查看到的节点）组件配置文件，由原来192.168.9.20修改为192.168.9.100（VIP）。

```bash
sed -i 's#192.168.9.20:6443#192.168.9.21:16443#' /opt/kubernetes/cfg/*
systemctl restart kubelet kube-proxy
```

检查节点状态

```bash
[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
k8s-master1   Ready    <none>   7d     v1.20.10
k8s-master2   Ready    <none>   90m    v1.20.10
k8s-node1     Ready    <none>   5d1h   v1.20.10
k8s-node2     Ready    <none>   5d1h   v1.20.10
```

至此，完！
