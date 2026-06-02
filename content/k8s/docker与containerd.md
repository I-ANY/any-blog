+++
title = "docker与containerd"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 4
+++

参考博客：https://i4t.com/5435.html#CRI_%E8%AF%A6%E8%A7%A3

# docker下容器运行时调用过程

​	Kubernetes社区在2020年7月份发布的版本中已经开始了dockershim的移除计划，在1.20版本中将内置的dockershim进行分离，这个版本依旧还可以使用dockershim，但是在1.24中被删除。

​	从1.24开始，大家需要使用其他受到支持的运行时选项（例如containerd或CRI-O）；如果选择Docker Engine作为运行时，则需要使用cri-dockerd

早期使用docker时，容器进行时的调用过程：

![Kubernetes容器运行时弃用Docker转型Containerd](./images/1ddd90033df752_1_circle.png)

当docker要创建一个容器时，经历以下步骤：

<img src="./images/ce28e6e767d0b.png" alt="Kubernetes容器运行时弃用Docker转型Containerd" style="zoom: 50%;" />

- Kubelet 通过CRI接口(gRPC)调用dockershim，请求创建一个容器。（CRI即容器运行时接口）；
- dockershim 收到请求后，转换成Docker Daemon能听懂的请求，发到Docker Daemon上请求创建容器；
- Docker Daemon 早在1.12版本中就已经针对容器的操作转移到另外一个进程--`containerd`,因此Docker Daemon不会帮我们创建容器，而是要求containerd创建一个容器；
- Containerd收到请求后，并不会直接去操作容器，而是创建一个叫做`containerd-shim`的进程,让containerd-shim去操作容器。这是因为容器进程需要一个父进程来做收集状态，而加入这个父进程就是containerd，那每次containerd挂掉或者升级，整个宿主机上的容器都会退出。而引用containerd-shim就避免了这个问题 (containerd和shim并不是父子进程的关系)；
- `OCI` (Open Container Initiative，开放容器标准，runC实际上就是参考OCI实现，OCI实际上就是一个标准文档，主要规定了容器镜像的结构、以及容器需要接收那些操作指令，比如create、start、stop、delete等)。OCI执行namespace和cgroups，挂载root filesystem等操作，OCI参考RunC，containerd-shim在这一步调用RunC命令行来启动容器。实际上RunC就是一个二进制命令；

- runC启动完成后本身会直接退出，containerd-shim则会为容器进程的父进程，负责收集容器进程的状态，上报给containerd，并在容器中pid为1的进程退出后接管容器中的子进程进行清理，确保不会出现僵尸进程；



​	实际上我们是可以直接通过调用RunC来实现容器的创建，实际上RunC就是调用的我们内核来进行操作。但是我们直接调用Runc不是很方便，所以就有了`OCI`。不需要了解底层原理，也可以通过调用`OCI`来进行容器的创建



# CRI

​	在Kubernetes早起的时候，Kubernetes为了支持Docker，通过硬编码的方式直接调用Docker API。后面随着Docker的不断发展以及Google的主导，出现了更多容器运行时可以使用，Kubernetes为了支持更多精简的容器运行时，google就和redhat主导推出了OCI标准，用于将Kubernetes平台和特定的容器运行时解耦。

**`CRI` (Container Runtime Interface容器运行时接口)本质就是Kubernetes定义的一组与容器运行时进行交互的接口**	

CRI实际上就是一组单纯的gRPC接口，核心有如下:

- RuntimeService 对容器操作的接口，包括创建，启停容器等
- ImageService 对镜像操作的接口，包括镜像的增删改查等

可以通过kubelet中`--container-runtime-endpoint`和`--image-service-endpoint`来手动配置



CRI大概通过了下面的几个项目构成了Kubernetes的Runtime生态

- OCI Compatible: runC
- CRI Compatible: Docker (借助dockershim)，containerd (借助CRI-containerd)

由于早期Kubernetes在市场没有主导地位，有一些容器运行时可能不会自身实现CRI接口，于是就有了shim，一个shim的职责就是作为适配器，将各种容器运行时的本身的接口适配到Kubernetes的CRI接口上

![Kubernetes容器运行时弃用Docker转型Containerd](./images/dc04b57012a36.png)

cri-runtime和oci-runtime 容器运行时实际上调用步骤如下：

```bash
Orchestration API -> Container API（cri-runtime） -> Kernel API(oci-runtime)
Kubelet通过gRPC 框架与容器运行时或shim进行通信，其中 kubelet 作为客户端，CRI shim（也可能是容器运行时本身）
```



# Containerd 发展史

在Containerd 1.0中，对CRI的适配通过了一个单独的进程CRI-containerd来完成

<img src="./images/1c11e2d2c4399.png" alt="Kubernetes容器运行时弃用Docker转型Containerd" style="zoom: 50%;" />

containerd 1.1中，砍掉了CRI-containerd这个进程，直接把适配逻辑作为插件放进了containerd主进程中

<img src="./images/9c6b5f9f1a13d.png" alt="Kubernetes容器运行时弃用Docker转型Containerd" style="zoom:50%;" />

containerd 1.1中做的事情，实际上Kubernetes社区做了一个更漂亮的`cri-o`，兼容CRI和OCI

![Kubernetes容器运行时弃用Docker转型Containerd](./images/b4fa4dd44d6aa.png)



# Containerd与Docker区别

相同点：

- **容器运行时**：Docker和Containerd都是容器运行时工具，它们用于创建、运行和管理容器。
- **轻量级**：两者都提供了轻量级的解决方案，使得应用程序可以在隔离的环境中运行，同时保持性能和安全性。
- **标准化**：Docker和Containerd都使用标准化技术，使得跨平台的兼容性更好，同时也简化了容器管理。

不同点：

- **核心功能**：Containerd更偏向于容器运行时和容器编排的底层功能，提供了更多的API和插件扩展功能。而Docker则更注重于提供易用的用户界面和工具集，使得用户可以更方便地创建、部署和管理容器。
- **社区支持**：Docker拥有庞大的社区支持和丰富的文档，使得用户可以更容易地找到帮助和支持。而Containerd则相对较小，但其功能更加底层和强大。
- **集成性**：Docker可以与其他工具和服务更好地集成，例如Kubernetes、Docker Swarm等。而Containerd则更多地被用于底层的容器编排和服务管理。
- **API兼容性**：Containerd更加开放，其API兼容性更好，可以与其他容器编排和服务更好地集成。而Docker则更多地依赖于其自己的API和工具集。

实际上containerd只是一个精简版docker，为了更好的支持Kubernetes而已

![Kubernetes容器运行时弃用Docker转型Containerd](./images/b3ac2d4240bbc.png)

**哪些容器运行时引擎支持CRI？**

| 容器运行时      | Kubernetes 平台中的支持                             | 优点                                                         | 缺点                                                         |
| :-------------- | :-------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Containerd      | 谷歌 Kubernetes 引擎、IBM Kubernetes 服务、阿里巴巴 | 经过大规模测试，用于所有 Docker 容器。比 Docker 使用更少的内存和 CPU。支持 Linux 和 Windows | 没有 Docker API 套接字。缺少 Docker 方便的 CLI 工具。        |
| CRI-O           | 红帽 OpenShift，SUSE 容器即服务                     | 轻量级，Kubernetes 所需的所有功能，仅此而已。类似 UNIX 的关注点分离（客户端、注册表、构建） | 主要在RedHat平台内使用不易安装在非RedHat操作系统上仅在Windows Server2019及更高版本中支持 |
| Kata Containers | 开放堆栈                                            | 提供基于 QEMUI 的完全虚拟化改进的安全性与 Docker、CRI-O、containerd 和 Firecracker 集成支持 ARM、x86_64、AMD64 | 更高的资源利用率不适合轻量级容器用例                         |
| AWS Firecracker | 所有 AWS 服务                                       | 可通过直接 API 或使用 seccomp jailer 的 containerdTight 内核访问来访问 | 新项目，不如其他运行时成熟需要更多手动步骤，开发人员体验仍在不断变化 |

可以看到这3个的区别，目前Kubernetes官网已经支持`containerd`、`CRI-o`容器运行时支持

![Kubernetes容器运行时弃用Docker转型Containerd](./images/c906f69d53dcf.png)

## Containerd

**Containerd是一个工业级标准的容器运行时，它强调简单性、可移植性**

早期Containerd是在Docker Engine中，目前将containerd从Docker中拆分出来，作为一个独立的开源项目，目标是提供一个更加开放、稳定的容器运行基础设施。分离出来的containerd将具有更多的功能，覆盖整个容器运行时的所有需求，提供更强大的支持。

对于K8s来说，实际需要Containerd即可，中间的垫片(shim)是完全可以省略，减少调用链

<img src="./images/fd8ef6033d4db.png" alt="Kubernetes容器运行时弃用Docker转型Containerd" style="zoom: 67%;" />

**Containerd已经将shim集成到kubelet中，减少了shim**，但是如果我们使用containerd，那么将无法使用docker ps或者docker exec命令来获取容器。可以使用docker pull和docker build命令来构建镜像

<img src="./images/c20d191343037.png" alt="Kubernetes容器运行时弃用Docker转型Containerd" style="zoom: 50%;" />



# containerd安装

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

containerd相比于docker , 多了namespace概念, 每个image和containe都会在各自的namespace下可见, 目前k8s会使用`k8s.io`作为命名空间,默认containerd会使用`default`

```bash
# 查看命令空间
ctr ns ls

# 创建命令空间
ctr ns create xxx

# 删除
ctr ns delete [NameSpace]
```

**注意：**在containerd中拉取docker的相关镜像需要补全信息

```bash
# -n 指定命名空间
# --all-platforms，会将所有平台都下载下来
ctr -n i4t i pull docker.io/library/nginx:alpine --all-platforms
```

**Task任务**
	在containerd中有一个task任务的概念，使用containerd create创建的容器，这时候并没有`running`；在Docker中可以直接run容器，但是在containerd是需要先`create`，再通过`task`启动容器。create 容器并不会启动容器，可以理解只是声明了一个container，并不会启动和执行相关操作。

```bash
[root@k8s-master k8s]# ctr task -h
NAME:
   ctr tasks - manage tasks

USAGE:
   ctr tasks command [command options] [arguments...]

COMMANDS:
   attach                   attach to the IO of a running container
   checkpoint               checkpoint a container
   delete, del, remove, rm  delete one or more tasks
   exec                     execute additional processes in an existing container
   list, ls                 list tasks
   kill                     signal a container (default: SIGTERM)
   pause                    pause an existing container
   ps                       list processes for container
   resume                   resume a paused container
   start                    start a container that has been created
   metrics, metric          get a single data point of metrics for a task with the built-in Linux runtime

OPTIONS:
   --help, -h  show help
```

```bash
# 启动容器
ctr -n xxx task start -d nginx  #-d后台运行

#查看当前运行容器
ctr -n xxx task ls  
TASK     PID     STATUS    
nginx    1465    RUNNING

ctr -n xxx task exec --exec-id 1 -t nginx sh  #进入容器
#--exec-id 设置一个id，唯一即可
#-t --tty为container分配一个tty
#nginx 容器名称
#sh && bash即可

#停止（暂停）容器
ctr -n xxx task pause nginx

# 关闭容器
# 如果我们需要关闭容器，只能通过kill来进行关闭，然后在重新start;在containerd中没有stop和restart参数
ctr -n xxx task kill nginx   #kill停止task任务

# 删除task任务（不会删除container）
ctr -n i4t task rm nginx   

# 删除容器
# c rm代表删除容器
ctr -n xxx c rm nginx
```



**Docker ctr nerdctl命令直接的区别**

`crictl`是kubernetes cri-tools的一部分，是专门为kubernetes使用containerd而专门制作的，提供了Pod、容器和镜像等资源的管理命令。

```bash
需要注意的是：使用其他非 kubernetes创建的容器、镜像，crictl是无法看到和调试的，比如说ctr run在未指定namespace情况下运行起来的容器就无法使用crictl看到。当然ctr可以使用-n k8s.io指定操作的namespace为 k8s.io，从而可以看到/操作kubernetes 集群中容器、镜像等资源。可以理解为：crictl 操作的时候指定了containerd 的namespace为k8s.io。
```

`nerdctl` **ctr**功能简单，而且对已经习惯使用`docker cli`的人来说，ctr并不友好（比如无法像 docker cli 那样）。这个时候`nerdctl`就可以替代ctr了。nerdctl是一个与docker cli风格兼容的containerd的cli工具，并且已经被作为子项目加入了 containerd 项目中。从`nerdctl 0.8`开始，nerdctl直接兼容了`docker compose`的语法(不包含 swarm)， 这很大程度上提高了直接将 containerd 作为本地开发、测试和单机容器部署使用的体验。

```bash
需要注意的是：安装 nerdctl 之后，要想可以使用 nerdctl 还需要安装 CNI 相关工具和插件。containerd不包含网络功能的实现，想要实现端口映射这样的容器网络能力，需要额外安装 CNI 相关工具和插件。
nerdctl 也可以使用 -n 指定使用的 namespace。
```

| 命令解释             | docker         | crictl          | ctr (不支持 build,commit)                                    | nerdctl                                                      |
| :------------------- | :------------- | :-------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| 查看容器列表         | docker ps      | crictl ps       | ctr c ls（查看非 kubernetes 中的容器）ctr -n k8s.io c ls（查看 kubernetes 集群中的容器） | nerdctl ps（查看非 kubernetes 中的容器）nerdctl -n k8s.io ps（查看 kubernetes 集群中的容器） |
| 查看容器详情         | docker inspect | crictl inspect  | ctr c info                                                   | nerdctl inspect                                              |
| 查看容器日志         | docker logs    | crictl logs     | 无                                                           | nerdctl logs                                                 |
| 容器内执行命令       | docker exec    | crictl exec     | ctr t exec                                                   | nerdctl exec                                                 |
| 挂载容器             | docker attach  | crictl attach   | ctr t attach                                                 | 无                                                           |
| 显示容器资源使用情况 | docker stats   | crictl stats    | ctr task metrics                                             | 无                                                           |
| 创建容器             | docker create  | crictl create   | ctr c create                                                 | 无                                                           |
| 启动容器             | docker start   | crictl start    | ctr t start                                                  | nerdctl start                                                |
| 运行容器             | docker run     | crictl run      | ctr run                                                      | nerdctl run                                                  |
| 停止容器             | docker stop    | crictl stop     | ctr t kill                                                   | nerdctl stop                                                 |
| 删除容器             | docker rm      | crictl rm       | ctr c rm                                                     | nerdctl rm                                                   |
| 查看镜像列表         | docker images  | crictl images   | ctr i ls                                                     | nerdctl images                                               |
| 查看镜像详情         | docker inspect | crictl inspecti | 无                                                           | nerdctl inspect                                              |
| 拉取镜像             | docker pull    | crictl pull     | ctr i pull                                                   | nerdctl pull                                                 |
| 推送镜像             | docker push    | 无              | ctr i push                                                   | nerdctl push                                                 |
| 删除镜像             | docker rmi     | crictl rmi      | ctr i rm                                                     | nerdctl rmi                                                  |
| 查看Pod列表          | 无             | crictl pods     | 无                                                           | 无                                                           |
| 查看Pod详情          | 无             | crictl inspectp | 无                                                           | 无                                                           |
| 启动Pod              | 无             | crictl runp     | 无                                                           | 无                                                           |
| 停止Pod              | 无             | crictl stopp    | 无                                                           | 无                                                           |
| 打标签               | docker tag     |                 | ctr i tag                                                    |                                                              |