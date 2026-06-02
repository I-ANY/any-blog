+++
title = "k8s安装-kubeadm（非高可用+高可用）"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 7
+++



# 前言：版本差异

​	Kubernetes社区在2020年7月份发布的版本中已经开始了dockershim的移除计划，在1.20版本中将内置的dockershim进行分离，这个版本依旧还可以使用dockershim，但是在1.24中被删除。

​	但是如果想继续使用docker的话，可以在kubelet和docker之间加上一个中间层cri-docker。cri-docker是一个支持CRI标准的shim（垫片）。一头通过CRI跟kubelet交互，另一头跟docker api交互，从而间接的实现了kubernetes以docker作为容器运行时。但是这种架构缺点也很明显，调用链更长，效率更低。

从1.24开始，大家需要使用其他受到支持的运行时选项：

- containerd

- CRI-O（另一个实现了容器运行时接口（CRI）的高级别容器运行时，可以使用 OCI（开放容器倡议）兼容的运行时， containerd 的一个替代品）

- Docker Engine，需依赖使用cri-dockerd



# 一、kubeadm安装K8s（1.23非高可用版）

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

```powershell
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



### 2. 安装docker/containerd

- docker

操作节点： 所有节点

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

**注意：**1.24版本以上的kubelet已删除docker-shim，不再内部支持CRI，所以需要另外安装cri-docker提供kubelet调用

但是如果想继续使用docker的话，可以在kubelet和docker之间加上一个中间层cri-docker。cri-docker是一个支持CRI标准的shim（垫片）。一头通过CRI跟kubelet交互，另一头跟docker api交互，从而间接的实现了kubernetes以docker作为容器运行时。但是这种架构缺点也很明显，调用链更长，效率更低

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



- containerd

github地址：[https://containerd.io/downloads/](https://gitee.com/link?target=https%3A%2F%2Fcontainerd.io%2Fdownloads%2F)

```bash
containerd-1.6.10-linux-amd64.tar.gz #只包含containerd
cri-containerd-cni-1.6.10-linux-amd64.tar.gz #包含containerd以及cri runc等相关工具包，建议下载本包
```

```bash
#下载tar.gz包
#containerd工具包，包含cri runc等
wget https://github.com/containerd/containerd/releases/download/v1.6.10/cri-containerd-cni-1.6.10-linux-amd64.tar.gz

#containerd包
wget https://github.com/containerd/containerd/releases/download/v1.6.10/containerd-1.6.10-linux-amd64.tar.gz

#备用下载地址
https://d.frps.cn/file/kubernetes/containerd/cri-containerd-cni-1.6.10-linux-amd64.tar.gz
https://d.frps.cn/file/kubernetes/containerd/containerd-1.6.10-linux-amd64.tar.gz
```

```bash
# 解压出来的是：etc  opt  usr 三个目录
# 所以可以直接解压到根目录下，把它直接放置到对应的目录下
tar zxvf cri-containerd-cni-1.6.4-linux-amd64.tar.gz -C /   

# 升级libseccomp，libseccomp需要高于2.4版本
# 卸载原来的，先查看对应的包，有多少个卸载多少个
rpm -qa | grep libseccomp
libseccomp-devel-2.3.1-4.el7.x86_64
libseccomp-2.3.1-4.el7.x86_64

rpm -e libseccomp-devel-2.3.1-4.el7.x86_64 --nodeps
rpm -e libseccomp-2.3.1-4.el7.x86_64 --nodeps

#下载高于2.4以上的包
wget http://rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm
#安装
rpm -ivh libseccomp-2.5.1-1.el8.x86_64.rpm 

#查看当前版本
rpm -qa | grep libseccomp
libseccomp-2.5.1-1.el8.x86_64
```

默认cri-containerd-cni包中会有containerd启动脚本，我们已经解压到对应的目录（/etc/systemd/system/containerd.service），可以直接调用启动

```bash
# 启动
systemctl enable containerd --now && systemctl status containerd 
```

Containerd属于cs架构需要安装`ctr`，通过crt进行管理控制；ctr实际上就是containerd的客户端工具，ctr在我们解压包中已经附带了，直接可以使用

```bash
# 查看版本
ctr version
# 启动容器
ctr -n xxx task start -d nginx  #-d后台运行
#查看当前运行容器
ctr -n xxx task ls 
```



### 3.  linux内核升级

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



## 集群信息
### 1. 节点规划
部署k8s集群的节点按照用途可以划分为如下2类角色：

- **master**：集群的master节点，集群的初始化节点，基础配置不低于2C4G
- **slave**：集群的slave节点，可以多台，基础配置不低于2C4G

**本例为了演示slave节点的添加，会部署一台master+2台slave**，节点规划如下：

|主机名 |  节点ip  | 角色 | 部署组件  |
|:--: | :--:  | :--: | :--------: |
|k8s-master|  | master |  etcd, kube-apiserver, kube-controller-manager, kubectl, kubeadm, kubelet, kube-proxy, flannel|
|k8s-slave1 |  | slave |  kubectl, kubelet, kube-proxy, flannel  |
|k8s-slave2  |  |  slave  | kubectl, kubelet, kube-proxy, flannel |

### 2. 组件版本

| 组件      |    版本 | 说明  |
| :--: | :--:| :-- |
| CentOS  | 7.6.1810 |     |
| Kernel  | Linux 3.10.0-1062.9.1.el7.x86_64 |   |
| etcd    | 3.3.15 | 使用容器方式部署，默认数据挂载到本地路径 |
| coredns | 1.6.2 |  |
| kubeadm | v1.23.6 |  |
| kubectl | v1.23.6 |  |
| kubelet | v1.23.6 |  |
| kube-proxy | v1.23.6 |  |
|  |  |  |



## kubeadm初始化

### 1. 安装 kubeadm, kubelet 和 kubectl
操作节点： 所有的master和slave节点(`k8s-master,k8s-slave`) 需要执行
``` bash
yum install -y kubelet-1.23.6 kubeadm-1.23.6 kubectl-1.23.6 --disableexcludes=kubernetes
## 查看kubeadm 版本
kubeadm version
## 设置kubelet开机启动
systemctl enable kubelet 

```
### 2. kubeadm初始化（只在master节点执行）

####  方式一：命令行带参数

```bash	
## 命令行的方式进行简略安装
# kubeadm初始化安装（master执行）：
# 注意注意注意： --apiserver-advertise-address参数，务必改为当前本机ip，apiserver的访问接口
[root@ANY ~]# kubeadm init \
 --apiserver-advertise-address=10.0.0.2 \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version v1.23.6 \
 --service-cidr=10.1.0.0/16 \
 --pod-network-cidr=10.2.0.0/16 \
 --service-dns-domain=cluster.local \
 --ignore-preflight-errors=Swap \
 --ignore-preflight-errors=NumCPU
 # --cri-socket=unix:///var/run/cri-dockerd.sock  # 如果是使用的cri-dockerd接口，需要指定socket

```

#### 方式二：默认配置文件

操作节点： 只在master节点（`k8s-master`）执行

``` yaml
kubeadm config print init-defaults > kubeadm.yaml

cat kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.10  # 配置为当前本机ip，apiserver访问地址
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock    # 注意，如果使用cri-dockerd的运行时调用接口，这里需要改为当当前cri的sock，一般是: unix:///var/run/cri-dockerd.sock
  name: k8s-master    # 修改为本机hostname
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers  # 修改成阿里镜像源
kind: ClusterConfiguration
kubernetesVersion: v1.23.6
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16  #添加 Pod 网段，flannel插件需要使用这个网段
  serviceSubnet: 10.96.0.0/12
scheduler: {}

```
**如果是使用的外部的ETCD集群，注意修改对应etcd部分配置，如下：**

```bash
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.9.30  # 配置为当前本机ip，apiserver访问地址
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock    # 注意，如果使用cri-dockerd的运行时调用接口，这里需要改为当当前cri的sock，一般是: unix:///var/run/cri-dockerd.sock
  name: k8s-master1    # 修改为本机hostname
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  external:      #这里默认是local，由于我们用的外部etcd，所以需要修改
    endpoints:
      - https://192.168.9.30:2379
      - https://192.168.9.31:2379
      - https://192.168.9.32:2379
    caFile: /opt/etcd/ssl/ca.pem             #搭建etcd集群时生成的ca证书
    certFile: /opt/etcd/ssl/server.pem       #搭建etcd集群时生成的客户端证书
    keyFile: /opt/etcd/ssl/server-key.pem    #搭建etcd集群时生成的客户端密钥
imageRepository: registry.aliyuncs.com/google_containers  # 修改成阿里镜像源
kind: ClusterConfiguration
kubernetesVersion: v1.23.6
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16  #添加 Pod 网段，网络插件需要使用这个网段
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

>  对于上面的资源清单的文档比较杂，要想完整了解上面的资源对象对应的属性，可以查看对应的 godoc 文档，地址: https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2。 



查看初始化结果的端口信息：

<img src="images/image-20230914171258727.png" style="zoom:70%;" />

### 3. 初始化master节点

**可提前下载镜像（可忽略）**

操作节点：只在master节点（`k8s-master`）执行

``` python
  # 查看需要使用的镜像列表,若无问题，将得到如下列表
$ kubeadm config images list --config kubeadm.yaml
registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.0
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.0
registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.0
registry.aliyuncs.com/google_containers/kube-proxy:v1.23.0
registry.aliyuncs.com/google_containers/pause:3.6
registry.aliyuncs.com/google_containers/etcd:3.5.1-0
registry.aliyuncs.com/google_containers/coredns:v1.8.6

  # 提前下载镜像到本地
$ kubeadm config images pull --config kubeadm.yaml
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.23.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.23.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.23.0
[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.6
[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.5.1-0
[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:v1.8.6
```

重要更新：如果出现不可用的情况，请使用如下方式来代替：

1. 还原kubeadm.yaml的imageRepository

   ```yaml
   ...
   imageRepository: k8s.gcr.io
   ...
   
   ## 查看使用的镜像源
   kubeadm config images list --config kubeadm.yaml
   k8s.gcr.io/kube-apiserver:v1.23.0
   k8s.gcr.io/kube-controller-manager:v1.23.0
   k8s.gcr.io/kube-scheduler:v1.23.0
   k8s.gcr.io/kube-proxy:v1.23.0
   k8s.gcr.io/pause:3.6
   k8s.gcr.io/etcd:3.5.1-0
   k8s.gcr.io/coredns:v1.8.6
   ```

2. 使用docker hub中的镜像源来下载，注意上述列表中要加上处理器架构，通常我们使用的虚拟机都是amd64

   ```powershell
   $ docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.23.0
   $ docker pull mirrorgooglecontainers/etcd-amd64:3.5.1-0
   ...
   ```



**初始化**

操作节点：只在master节点（`k8s-master`）执行

``` python
kubeadm init --config kubeadm.yaml
```
若初始化成功后，最后会提示如下信息：
``` python
...
#安装完成后，看到如下提示：
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
============================
#继续执行提示信息
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

=============================
#pod分布在多个机器上面，pod之间的连接需要集群网络（选用flannel网络插件，安装即可使用）
You should now deploy a pod network to the cluster.

=============================
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

=============================
#使用如下命令将k8s-node加入集群即可
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.2:6443 --token 63zpjo.rtw8y1j6ugtpgp2t \
    --discovery-token-ca-cert-hash sha256:4c5f07b362b31eaaa14f5caa35b5418251b5e24bda59fdb6b68cf745a13ce1b9 

```
接下来按照上述提示信息操作，配置kubectl客户端的认证
``` python
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> **⚠️注意：**此时使用 kubectl get nodes查看节点应该处于notReady状态，因为还未配置网络插件
>
> 若执行初始化过程中出错，根据错误信息调整后，执行kubeadm reset后再次执行init操作即可
### 4. 添加slave节点到集群中

操作节点：所有的slave节点（`k8s-slave`）需要执行
在每台slave节点，执行如下命令，该命令是在kubeadm init成功后提示信息中打印出来的，需要替换成实际init后打印出的命令。

``` python
kubeadm join 172.21.32.15:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1c4305f032f4bf534f628c32f5039084f4b103c922ff71b12a5f0f98d1ca9a4f
```


### 5.安装网络组件 

#### flannel插件

操作节点：只在master节点（`k8s-master`）执行

下载flannel的yaml文件

```powershell
wget -c https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

修改配置，指定网卡名称，大概在文件的和170行，190行，添加一行配置：

```powershell
$ vi kube-flannel.yml
...      
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=ens33  # 如果机器存在多网卡的话，指定内网网卡的名称，默认不指定的话会找第一块网
        resources:
          requests:
            cpu: "100m"
...
```

(可选)修改flannel的网络模式，如果有修改pod的网段，这里也需同步更改：

```yaml
...
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",	 # pod的网段，如果有修改pod的网段，这里也需同步更改
      "Backend": {		
        "Type": "vxlan"             # 网络模式默认为vxlan模式，可修改为host-gw模式  
        # "Directrouting": true     # 开启DirectRouting模式，仅在vxlan模式下可用，默认不开启，无此配置
      }
    }
...
```

执行安装flannel网络插件

```powershell
# 先拉取镜像,此过程国内速度比较慢
$ docker pull quay.io/coreos/flannel:v0.11.0-amd64
# 执行flannel安装
$ kubectl create -f kube-flannel.yml
```

#### calico插件

Calico有三四种安装方式：

1、使用calico.yaml清单文件安装（推荐使用）

2、二进制安装方式（很少用，不介绍了）

3、插件方式（也很少用了，不介绍了）

4、使用Tigera Calico Operator安装Calico（官方最新指导）

​	Tigera Calico Operator,Calico操作员是一款用于管理Calico安装、升级的管理工具，它用于管理Calico的安装生命周期。从Calico-v3.15版本官方开始使用此工具。



- **第一种：使用calico.yaml清单文件安装**

版本关系：

```bash
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

根据需要下载需要的calico资源文件清单

```bash
# 获取较新的calico资源清单
wget https://docs.projectcalico.org/manifests/calico.yaml --no-check-certificate

# 获取指定版本
wget https://projectcalico.docs.tigera.io/archive/v3.24/manifests/calico.yaml
```

**清单文件一些配置详解：**

该清单文件安装了以下Kubernetes资源：

- 使用DaemonSet在每个主机上安装calico/node容器；

- 使用DaemonSet在每个主机上安装Calico CNI二进制文件和网络配置；
- 使用Deployment运行calico/kube-controller；
- Secert/calico-etcd-secrets提供可选的Calico连接到etcd的TLS密钥信息；
- ConfigMap/calico-config提供安装Calico时的配置参数。

**修改部分：**

1、**calico_backend: "bird"**

设置Calico使用的后端机制。支持值：

- bird：开启BIRD功能，根据Calico-Node的配置来决定主机的网络实现是采用BGP路由模式还是IPIP、VXLAN覆盖网络模式。这个是默认的模式。
- vxlan：纯VXLAN模式，仅能够使用VXLAN协议的覆盖网络模式。

```yaml
  # Configure the backend to use.
  calico_backend: "bird"
```

（可选）默认网络模式还是IPIP，如需要改为BGP模式修改如下配置为`Never`

```bash
# 默认使用IPIP模式时设置，如果是第一次安装需要改为BGP模式，直接改为Never：
# Enable IPIP
- name: CALICO_IPV4POOL_IPIP
  value: "Always"
  
# 可选值
Always：只使用IPIP隧道网络，默认值
Never：不使用IPIP隧道网络，使用BGP模式
CrossSubnet：启用混合网络模式
```

2、清单文件中"**CALICO_IPV4POOL_CIDR**"部分配置

再次提醒：此项用于设置安装Calico时要创建的默认IPv4池，PodIP将从该范围中选择，Calico安装完成后修改此值将再无效。

默认情况下calico.yaml中"CALICO_IPV4POOL_CIDR"是注释的，如果kube-controller-manager的"--cluster-cidr"不存在任何值的话，则通常取默认值"192.168.0.0/16,172.16.0.0/16，..，172.31.0.0/16"。
当使用kubeadm时，PodIP的范围应该与kubeadm init的清单文件中的"podSubnet"字段或者"--pod-network-cidr"选项填写的值相同。

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

3、（可选）默认的Calico清单文件中所使用的镜像来源于docker.io国外镜像源，上面我们配置了Docker镜像加速，应删除docker.io前缀以使镜像从国内镜像加速站点下载。

```bash
# cat calico.yaml |grep 'image:'
image: docker.io/calico/cni:v3.23.5
image: docker.io/calico/cni:v3.23.5
image: docker.io/calico/node:v3.23.5
image: docker.io/calico/kube-controllers:v3.23.5

# sed -i 's#docker.io/##g' calico.yaml
# cat calico.yaml |grep 'image:'
image: calico/cni:v3.23.5
image: calico/cni:v3.23.5
image: calico/node:v3.23.5
image: calico/kube-controllers:v3.23.5
```

4、（可选）如果有多张网卡，进行指定网卡，默认选择的第一张网卡，可能不是所需的网卡

在yaml文件中DaemonSet的配置中，通过环境变量来指定的

```bash
......
	    - name: calico-node
          image: calico/node:v3.23.5
          envFrom:
          - configMapRef:
              # Allow KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT to be overridden for eBPF mode.
              name: kubernetes-services-endpoint
              optional: true
          env:
            # Use Kubernetes API as the backing datastore.
            - name: DATASTORE_TYPE
              value: "kubernetes"
            # Wait for the datastore.
            - name: WAIT_FOR_DATASTORE
              value: "true"
            - name: IP_AUTODETECTION_METHOD  # DaemonSet中添加该环境变量
              value: interface=eth0    # 指定内网网卡
......
```

这里如果不修改，可能会存在有的节点的calico启动不正常情况：起了但是又没完全起

![image-20240523182901686](./images/image-20240523182901686.png)

describe一下可以看到报错：

```bash
  Warning  Unhealthy  <invalid> (x3 over <invalid>)  kubelet            Readiness probe failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket: dial unix /var/run/calico/bird.ctl: connect: connection refused
  Warning  Unhealthy  <invalid>                      kubelet            Readiness probe failed: 2024-05-23 17:40:37.585 [INFO][181] confd/health.go 180: Number of node(s) with BGP peering established = 0
calico/node is not ready: BIRD is not ready: BGP not established with 192.168.9.30,192.168.9.34,192.168.9.35
```

执行命令，进入calico-node，打开bird配置文件

```bash
kubectl exec -ti calico-node-xvj55 -n kube-system -- bash

cat etc/calico/confd/config/bird.cfg
```

<img src="./images/image-20240523183051449.png" alt="image-20240523183051449" style="zoom:67%;" />

到此，基本可以确定是节点的calico的BGP网卡设备识别错误导致。

解决方法：

在所出问题的主机上删除`br`开头的网卡即可

![image-20240523183253922](./images/image-20240523183253922.png)

**最后注意：**Calico使用了错误的[网桥](https://so.csdn.net/so/search?q=网桥&spm=1001.2101.3001.7020)导致网络无法连通，所以我们从根本上解决问题需要指定Calico识别网桥的规则，否则过了一段时间问题还会再次出现

其它的不需要更改，默认就好了，也没什么可设置的。



- **第二种：使用Tigera Calico Operator安装Calico（官方最新指导）**

官网安装指导：https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises

```yaml
# 先安装operator管理工具
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.0/manifests/tigera-operator.yaml
```

```yaml
# 下载配置calico所需的自定义资源
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.0/manifests/custom-resources.yaml -O
```

修改Pod分配的子网范围（CIDR）

```yaml
[root@localhost ~]# cat custom-resources.yaml
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16       #修改成--pod-network-cidr
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
 	
---
 
# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

```yaml
# 安装
kubectl create -f custom-resources.yaml

# 查看
kubectl get pod -n calico-system
```



### 6.修改kube-proxy工作模式为ipvs模式

不修改的话，默认模式为iptables模式

查看当前集群使用的模式

```bash
# 1、查看配置文件中的mode配置
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
kubectl edit cm kube-proxy -n kube-system

# 2、查看日志
kubectl logs -n kube-system kube-proxy-xxxx | grep "Using"
I0221 02:44:50.829527       1 server_others.go:236] "Using ipvs Proxier"

# 3、访问kube-proxy接口，通常不适用于生产环境，因为可能会暴露敏感信息
curl localhost:10249/proxyMode
```

```bash
# 此模式必须在所有master、node节点上安装ipvs内核模块，否则会降级为iptables
yum -y install ipset ipvsadm

# 配置ipvsadm模块开机自动执行加载相关模块
# 1、在/etc/sysconfig/modules/ipvs.modules下配置
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
#授权、运行
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules

# 2、配置开机自执行加载，也可以写到rc.local文件下
cat <<EOF>> etc/rc.local
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
# 授权
chmod +x /etc/rc.d/rc.local

# 检查是否加载
lsmod | grep -e ip_vs -e nf_conntrack
```

修改kube-proxy为ipvs模式

```bash
# 开启ipvs，其中mode原来是空，即默认为iptables模式，需要改为ipvs
[root@k8s-master01 ~]# kubectl edit cm kube-proxy -n kube-system
    ...
    detectLocalMode: ""
    enableProfiling: false
    healthzBindAddress: ""
    hostnameOverride: ""
    iptables:
      masqueradeAll: false
      masqueradeBit: null
      minSyncPeriod: 0s
      syncPeriod: 0s
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""          # scheduler默认是空，默认负载均衡算法为轮询
      strictARP: false
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: "ipvs"              ## 默认为空值，即默认为iptables模式，需要修改为ipvs
    ...
```


**修改完kubeproxy的配置文件，需要重启kubeproxy的pod，逐一删除即可**

```bash
# kubectl delete pod -l k8s-app=kube-proxy -n kube-system
pod "kube-proxy-cctq2" deleted
pod "kube-proxy-qwc8v" deleted
pod "kube-proxy-tg6gm" deleted

# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0
  
# 查看集群的kube-proxy的模式
curl localhost:10249/proxyMode
```

### 7.  设置master节点是否可调度（可选）

操作节点：`k8s-master`

默认部署成功后，master节点无法调度业务pod，如需设置master节点也可以参与pod的调度，需执行：

``` python
# 此步骤操作是去除master节点上的污点，使得master节点也能进行调度安装pod
$ kubectl taint node k8s-master node-role.kubernetes.io/master:NoSchedule-
```
### 8. 验证集群
操作节点： 在master节点（`k8s-master`）执行

``` python
$ kubectl get nodes  #观察集群节点是否全部Ready
NAME                STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   22h   v1.13.3
k8s-slave   Ready    <none>   22h   v1.13.3
```

创建测试nginx服务

``` python
$ kubectl run  test-nginx --image=nginx:alpine
```

查看pod是否创建成功，并访问pod ip测试是否可用

``` powershell
$ kubectl get po -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
test-nginx-5bd8859b98-5nnnw   1/1     Running   0          9s    10.244.1.2   k8s-slave1   <none>           <none>
$ curl 10.244.1.2
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 9. 部署dashboard

- 部署服务

```powershell
# 推荐使用下面这种方式
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/recommended.yaml
$ vi recommended.yaml
# 修改Service为NodePort类型
......
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort  # 加上type=NodePort变成NodePort类型的服务
......
```

- 查看访问地址，本例为30133端口

```powershell
kubectl create -f recommended.yaml
kubectl -n kubernetes-dashboard get svc
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.105.62.124   <none>        8000/TCP        31m
kubernetes-dashboard        NodePort    10.103.74.46    <none>        443:30133/TCP   31m 
```

- 使用浏览器访问 https://62.234.133.177:30133，其中62.234.133.177为master节点的外网ip地址，chrome目前由于安全限制，测试访问不了，使用firefox可以进行访问。

- 创建ServiceAccount进行访问

```powershell
$ vi admin.conf
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard

$ kubectl create -f admin.conf
$ kubectl -n kubernetes-dashboard get secret |grep admin-token
admin-token-fqdpf                  kubernetes.io/service-account-token   3      7m17s
# 使用该命令拿到token，然后粘贴到
$ kubectl -n kubernetes-dashboard get secret admin-token-fqdpf -o jsonpath={.data.token}|base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1rb2xHWHMwbWFPMjJaRzhleGRqaExnVi1BLVNRc2txaEhETmVpRzlDeDQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1mcWRwZiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjYyNWMxNjJlLTQ1ZG...

```

![dashboard](images\dashboard.png)

![dashboard-content](images\dashboard-content.jpg)

### 10. kubeadm reset清理环境

如果你的集群安装过程中遇到了其他问题，我们可以使用下面的命令来进行重置：

```powershell
kubeadm reset -f 
rm -rf ~/.kube/  /etc/kubernetes/* var/lib/etcd/* /var/lib/cni/*
```



# 二、kubeadm安装k8s（1.24版及以上类似）

​	Kubernetes社区在2020年7月份发布的版本中已经开始了dockershim的移除计划，在1.20版本中将内置的dockershim进行分离，这个版本依旧还可以使用dockershim，但是在1.24中被删除。

从1.24开始，大家需要使用其他受到支持的运行时选项：

- containerd

- CRI-O（另一个实现了容器运行时接口（CRI）的高级别容器运行时，可以使用 OCI（开放容器倡议）兼容的运行时， containerd 的一个替代品）

- Docker Engine，需依赖使用cri-dockerd



**本次使用k8s的1.27版本，containerd作为容器运行时进行安装**

## 1、调整系统配置

与上面一致

## 2、安装containd

github地址：https://containerd.io/downloads/

```bash
containerd-1.6.10-linux-amd64.tar.gz 只包含containerd
cri-containerd-cni-1.6.10-linux-amd64.tar.gz 包含containerd以及cri runc等相关工具包，建议下载本包
```

```bash
#下载tar.gz包
#containerd工具包，包含cri runc等
wget https://github.com/containerd/containerd/releases/download/v1.6.10/cri-containerd-cni-1.6.10-linux-amd64.tar.gz

#containerd包
wget https://github.com/containerd/containerd/releases/download/v1.6.10/containerd-1.6.10-linux-amd64.tar.gz

#备用下载地址
https://d.frps.cn/file/kubernetes/containerd/cri-containerd-cni-1.6.10-linux-amd64.tar.gz
https://d.frps.cn/file/kubernetes/containerd/containerd-1.6.10-linux-amd64.tar.gz
```

解压

```bash
# 解压出来的是：etc  opt  usr 三个目录
# 所以可以直接解压到根目录下，把它直接放置到对应的目录下
tar zxvf cri-containerd-cni-1.6.4-linux-amd64.tar.gz -C /   
```



升级libseccomp，libseccomp需要高于2.4版本

因为runc依赖于libseccomp计算库

```bash
# 卸载原来的，先查看对应的包，有多少个卸载多少个
rpm -qa | grep libseccomp
libseccomp-devel-2.3.1-4.el7.x86_64
libseccomp-2.3.1-4.el7.x86_64

rpm -e libseccomp-devel-2.3.1-4.el7.x86_64 --nodeps
rpm -e libseccomp-2.3.1-4.el7.x86_64 --nodeps

#下载高于2.4以上的包
wget http://rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm
#安装
rpm -ivh libseccomp-2.5.1-1.el8.x86_64.rpm 

#查看当前版本
rpm -qa | grep libseccomp
libseccomp-2.5.1-1.el8.x86_64
```



默认cri-containerd-cni包中会有containerd启动脚本，我们已经解压到对应的目录（/etc/systemd/system/containerd.service），可以直接调用启动

```bash
# 启动
# systemctl enable containerd --now

# systemctl status containerd 
  containerd.service - containerd container runtime
   Loaded: loaded (/etc/systemd/system/containerd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-02-22 19:03:55 CST; 2s ago
     Docs: https://containerd.io
  Process: 2043 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
 Main PID: 2046 (containerd)
    Tasks: 9
   Memory: 20.0M
   CGroup: /system.slice/containerd.service
           └─2046 /usr/local/bin/containerd
```

**containerd配置**
每个顶级配置块的命名都是`plugin."io.containerd.xxx.vxx.xxx"`这种形式，其实每个顶级配置块都代表一个插件，其中`io.containerd.xxx.vxx`表示插件类型，`vxx`后面的`xxx`表示 插件ID。并且可以通过命令`ctr`查看到

配置文件详解：https://www.cnblogs.com/FengGeBlog/p/15057399.html

```yaml
...
[plugins]
  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"
  [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
...
```

```bash
# ctr plugin ls
TYPE                                  ID                       PLATFORMS      STATUS    
io.containerd.content.v1              content                  -              ok        
io.containerd.snapshotter.v1          aufs                     linux/amd64    skip      
io.containerd.snapshotter.v1          btrfs                    linux/amd64    skip      
io.containerd.snapshotter.v1          devmapper                linux/amd64    error     
io.containerd.snapshotter.v1          native                   linux/amd64    ok        
io.containerd.snapshotter.v1          overlayfs                linux/amd64    ok        
io.containerd.snapshotter.v1          zfs                      linux/amd64    skip
...
```

Containerd属于cs架构需要安装`ctr`，通过crt进行管理控制；ctr实际上就是containerd的客户端工具

ctr在我们解压包中已经附带了，直接可以使用

```bash
# ctr version
Client:
  Version:  v1.6.10
  Revision: 770bd0108c32f3fb5c73ae1264f7e503fe7b2661
  Go version: go1.18.8

Server:
  Version:  v1.6.10
  Revision: 770bd0108c32f3fb5c73ae1264f7e503fe7b2661
  UUID: 562361a1-761c-48cb-8bc8-f465b715ba1a
```

**更新runc（根据需要升级）**

查看runc版本

```bash
runc --version
runc version 1.1.4
commit: v1.1.4-0-g5fd4c4d1
spec: 1.0.2-dev
go: go1.18.8
libseccomp: 2.5.1
```



下载对应版本的runc

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.amd64
```

赋权

```bash
chmod +x runc.amd64
```

删除contianerd自带的runc

```bash
whereis runc
runc: /usr/local/sbin/runc

rm -rf /usr/local/sbin/runc
```

移动新的runc

```bash
mv runc.amd64 /usr/local/sbin/runc
```

查看测试

```bash
runc --version
runc version 1.1.7
commit: v1.1.7-0-g5fd4c4d1
spec: 1.0.7-dev
go: go1.20.3
libseccomp: 2.5.1
```

重启所有服务器 



## 3、 安装 kubeadm, kubelet 和 kubectl

操作节点： 所有的master和slave节点(`k8s-master,k8s-slave`) 需要执行

``` bash
yum install -y kubelet-1.23.6 kubeadm-1.23.6 kubectl-1.23.6 --disableexcludes=kubernetes
## 查看kubeadm 版本
kubeadm version
## 设置kubelet开机启动
systemctl enable kubelet 
```

修改kubelet配置文件（不加也可，默认就是systemd）

```bash
vi /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
```

## 2、kubeadm初始化（只在master节点执行）

### 方式一：命令行带参数

```bash	
## 命令行的方式进行简略安装
# kubeadm初始化安装（master执行）：
# 注意 注意 注意： --apiserver-advertise-address参数，务必改为当前本机ip，apiserver的访问接口
[root@ANY ~]# kubeadm init \
 --apiserver-advertise-address=192.168.9.24 \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version v1.27.0 \
 --service-cidr=10.1.0.0/16 \
 --pod-network-cidr=10.2.0.0/16 \
 --service-dns-domain=cluster.local \
 --ignore-preflight-errors=Swap \
 --ignore-preflight-errors=NumCPU
 --cri-socket=unix:///var/run/containerd/containerd.sock  # 指定containerd的CRI接口

```

### 方式二：默认配置文件

操作节点： 只在master节点（`k8s-master`）执行

```bash
kubeadm config print init-defaults > kubeadm.yaml
cat kubeadm.yaml
```

``` yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.9.24    # 配置为当前本机ip，apiserver访问地址
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock   # 使用的containerd的socket，不用动
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers  # 修改成阿里镜像源
kind: ClusterConfiguration
kubernetesVersion: 1.27.0
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16  #添加 Pod 网段，flannel插件需要使用这个网段
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

>  对于上面的资源清单的文档比较杂，要想完整了解上面的资源对象对应的属性，可以查看对应的 godoc 文档，地址: https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2。 

**初始化**

操作节点：只在master节点（`k8s-master`）执行

``` python
kubeadm init --config kubeadm.yaml
```

往下操作跟“一”下步骤一致，直接使用即可



# 三、kubeadm安装k8s（1.18高可用版本）

## 集群规划

![image-20231102233730575](images/image-20231102233730575.png)



### 1、服务器节点分配

| 角色         | ip        | 应用                                                         |
| ------------ | --------- | ------------------------------------------------------------ |
| k8s-master-1 | 10.0.0.10 | kubeadm，kubectl, kubelet, kube-proxy, flannel，etcd, kube-apiserver,kube-controller-manager |
| k8s-master-2 | 10.0.0.11 | kubectl, kubelet, kube-proxy, flannel，etcd, kube-apiserver,kube-controller-manager |
| k8s-master-2 | 10.0.0.12 | kubectl, kubelet, kube-proxy, flannel，etcd, kube-apiserver,kube-controller-manager |
| k8s-node-1   | 10.0.0.13 | kubectl, kubelet, kube-proxy, flannel                        |
| k8s-node-2   | 10.0.0.14 | kubectl, kubelet, kube-proxy, flannel                        |

负载均衡分配

| 角色                                                         | 虚拟ip   | 应用       |
| ------------------------------------------------------------ | -------- | ---------- |
| k8s-master-1，k8s-master-2（我虚拟机不够所以直接安装在master上） | 10.0.0.9 | keepalived |
| k8s-master-1，k8s-master-2（我虚拟机不够所以直接安装在master上） |          | haproxy    |
|                                                              |          |            |

**环境配置推荐**（不管运行什么环境，每台机子起码要预留30%的资源）

生产环境建议配置：

- master：8核cpu，16G内存，100G硬盘
- node：16核cpu，64G内存，500G硬盘

### 2、安装keepailve+haproxy

**安装keepalive**（这里测试是安装到master上，生产建议安装到单独机器）

```bash
#两个master都安装
yum -y install epel-release 
yum -y install keepalived haproxy

```

**keepalive配置**

- **master1配置**

```bash

#master1配置
cat > /etc/keepalived/keepalived.conf <<EOF 
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"   # 检测haproxy是否存在
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER   # 主节点
    interface ens33     # ens33 为网卡名称，可以使用ifconfig查看自己的网卡名称
    virtual_router_id 51
    priority 100   # 权重，keepalived的master节点必须大于keepalived的backup节点
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab   # 密钥，master和backup密钥必须一致
    }
    virtual_ipaddress {
        10.0.0.9    # 虚拟ip
    }
    track_script {
        check_haproxy
    }

}
EOF

```

- **master2配置**

```bash
#master2配置
cat > /etc/keepalived/keepalived.conf <<EOF 
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"   # 检测haproxy是否存在
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP   # 从节点
    interface ens33     # ens33 为网卡名称，可以使用ifconfig查看自己的网卡名称
    virtual_router_id 51
    priority 50   # 权重，keepalived的master节点必须大于keepalived的backup节点
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab   # 密钥，master和backup密钥必须一致
    }
    virtual_ipaddress {
        10.0.0.9    # 虚拟ip
    }
    track_script {
        check_haproxy
    }

}
EOF
```

**两台master启动**

```bash
# 启动keepalived
systemctl start keepalived.service
# 设置开机启动
systemctl enable keepalived.service
# 查看启动状态
systemctl status keepalived.service

#检查
ip addr
```



**haproxy配置**

**三台master配置都相同**

```bash
cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon 
       
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------  
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#--------------------------------------------------------------------- 
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443      #默认监听端口16443
    option               tcplog
    default_backend      kubernetes-apiserver    
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server      master-1.k8s.io   10.0.0.10:6443 check    # 修改IP
    server      master-2.k8s.io   10.0.0.11:6443 check    # 修改IP
    server      master-3.k8s.io   10.0.0.11:6443 check    # 修改IP
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
EOF

```

**三台master启动**

```bash
# 启动 haproxy
systemctl start haproxy
# 设置开启自启
systemctl enable haproxy
# 查看启动状态
systemctl status haproxy

#检查，此时kubelet没有起，所以现在haproxy还起不来，等安装安kubelet之后才重新启动
netstat -tunlp | grep haproxy
```



### 3、安装基础组件

所有节点安装

```bash
yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
```

### 4、kubeadm初始化（有vip的master执行）

我这里是master1

```bash
# 创建文件夹
mkdir /usr/local/kubernetes/manifests -p
# 到manifests目录
cd /usr/local/kubernetes/manifests/
# 新建yaml文件



```

```bash
cat >kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2    # 集群主版本，根据集群版本号决定
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.10    #配置为本机ip，apiserver的访问地址
  bindPort: 6443              #apiserver集群端口号
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master-1           #本机hostname
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - k8s-master-1
  - k8s-master-2
  - 10.0.0.10  # master1 ip
  - 10.0.0.11  # master2 ip

  - 10.0.0.9     # 虚拟vip keepalive出来的ip
  - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 2m0s      # 注册时间2分钟
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes      # 集群名字
controlPlaneEndpoint: "10.0.0.9:16443"              # 虚拟ip + haproxy绑定的端口号
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.18.0               # 集群版本，需要与kubeadm版本一致
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16                 # pod 内网ip网段
  serviceSubnet: 10.96.0.0/12               # svc 内网ip网段
scheduler: {}
EOF
```

这里补充一个自动生成`kubeadm-config.yaml`文件的命令，生成模板后需要修改。`kubeadm config print init-defaults > kubeadm-config.yaml`.

**初始化**

```bash
#注意在vip的master上执行
kubeadm init --config kubeadm-config.yaml
```

注意：如果上面的yaml文件已经执行过一次，再次执行就会报错，这时，就需要将集群初始化后才能执行。

```bash
# 1. 还原由 kubeadm init 或 kubeadm join 所做的更改
kubeadm reset -f
# 2. 删除相关文件
rm -rf /etc/kubernetes/*
rm -rf /var/lib/etcd/*
```



**上述步骤完成后**

```bash
# 执行下方命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 查看节点
kubectl get nodes
# 查看集群健康状态
kubectl get cs
# 查看pod
kubectl get pods -n kube-system



#执行结果：
...
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.0.0.9:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:31e764721354492527024c012e30356237a96b05b3d38f8cc82b136cf3ecf399 \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.9:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:31e764721354492527024c012e30356237a96b05b3d38f8cc82b136cf3ecf399 


```

### 5、master加入集群

```bash
# master1中执行，修改IP，分别改为master2/3的IP执行2遍
ssh root@10.0.0.11 mkdir -p /etc/kubernetes/pki/etcd   # 修改IP；master2\master3的IP，用master1连接master2\master3；输入密码

# 复制相应的证书等文件到master2/master3
scp /etc/kubernetes/admin.conf root@10.0.0.11:/etc/kubernetes
   
scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@10.0.0.11:/etc/kubernetes/pki
   
scp /etc/kubernetes/pki/etcd/ca.* root@10.0.0.11:/etc/kubernetes/pki/etcd

```

**在要加入的master节点上执行**

```bash
kubeadm join 10.0.0.9:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:81601163a24e2a10067316e7846b114979781094a41b1b56910d9d26e7db80b5 \
    --control-plane 
```

### 6、node加入集群

```bash
kubeadm join 10.0.0.9:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:81601163a24e2a10067316e7846b114979781094a41b1b56910d9d26e7db80b5
```



### 5、安装网络插件

参考上面步骤

### 6、修改kube-proxy工作模式

参考上面步骤

### 7、验证集群

```bash
# 创建nginx deployment
kubectl create deployment nginx --image=nginx
# 暴露端口
kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
# 查看状态
kubectl get pod,svc

```





***所以，如果有3个master节点，必须要保证2个正常运行；如果有5个master，必须保证3个正常运行。假设有N个master节点，必须保证有(N+1)/2个节点正常运行，才能保证集群正常。***



