+++
title = "k8s笔记整理"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 8
+++

# kubernete 之路

## 一、前期介绍

#### 纯容器模式的问题

1. 业务容器数量庞大，哪些容器部署在哪些节点，使用了哪些端口，如何记录、管理，需要登录到每台机器去管理？

2. 跨主机通信，多个机器中的容器之间相互调用如何做，iptables规则手动维护？

3. 跨主机容器间互相调用，配置如何写？写死固定IP+端口？

4. 如何实现业务高可用？多个容器对外提供服务如何实现负载均衡？

5. 容器的业务中断了，如何可以感知到，感知到以后，如何自动启动新的容器?

6. 如何实现滚动升级保证业务的连续性？

   

#### 容器调度管理平台

​		Docker Swarm       Mesos        Google Kubernetes

​		2017年开始Kubernetes凭借强大的容器集群管理功能, 逐步占据市场,目前在容器编排领域一枝独秀

​		 https://kubernetes.io/ 

## 二、架构介绍



#### 架构图

区分组件与资源

![](images/architecture.png)



#### 核心组件

- **ETCD：**分布式高性能键值数据库,存储整个集群的所有元数据
- **ApiServer:**  API服务器,集群资源访问控制入口,提供restAPI及安全访问控制
- **Scheduler：**负责监视新创建的、尚未被运⾏的 Pods，并选择⼀个 Node 供它们运⾏。调度决策考虑到资源需求、硬件/软件/策略约束、负载均衡、亲和性和反亲和性等因素。
- **Controller Manager：**运⾏控制器进程。逻辑上，每个控制器是⼀个单独的进程，但为了降低复杂性，它们都被编译到同⼀个⼆进制⽂件，并在单独的进程中运⾏。这些控制器包括：Node 控制器、Replication 控制器、Endpoints 控制器、Service Account & Token 控制器等。
  - Replication Controller
  - Node controller
  - ResourceQuota Controller
  - Namespace Controller
  - ServiceAccount Controller
  - Tocken Controller
  - Service Controller
  - Endpoints Controller
- **kubelet：**运行在每运行在每个节点上的主要的“节点代理”个节点上的主要的“节点代理”
  - pod 管理：kubelet 定期从所监听的数据源获取节点上 pod/container 的期望状态（运行什么容器、运行的副本数量、网络或者存储如何配置等等），并调用对应的容器平台接口达到这个状态。
  - 容器健康检查：kubelet 创建了容器之后还要查看容器是否正常运行，如果容器运行出错，就要根据 pod 设置的重启策略进行处理.
  - 容器监控：kubelet 会监控所在节点的资源使用情况，并定时向 master 报告，资源使用数据都是通过 cAdvisor 获取的。知道整个集群所有节点的资源情况，对于 pod 的调度和正常运行至关重要
- **CNI实现:** 通用网络接口, 常见的插件有flannel、calico， 实现跨节点通信
- **kubectl:** 命令行接口，用于对 Kubernetes 集群运行命令  https://kubernetes.io/zh/docs/reference/kubectl/ 



#### 各个资源与apiserver对应表

| apiVersion | 含义                                                         |
| :--------- | :----------------------------------------------------------- |
| alpha      | 进入K8s功能的早期候选版本，可能包含Bug，最终不一定进入K8s    |
| beta       | 已经过测试的版本，最终会进入K8s，但功能、对象定义可能会发生变更。 |
| stable     | 可安全使用的稳定版本                                         |
| v1         | stable 版本之后的首个版本，包含了更多的核心对象              |
| apps/v1    | 使用最广泛的版本，像Deployment、ReplicaSets都已进入该版本    |

资源类型与apiVersion对照表

| Kind                  | apiVersion                              |
| :-------------------- | :-------------------------------------- |
| ClusterRoleBinding    | rbac.authorization.k8s.io/v1            |
| ClusterRole           | rbac.authorization.k8s.io/v1            |
| ConfigMap             | v1                                      |
| CronJob               | batch/v1beta1                           |
| DaemonSet             | extensions/v1beta1                      |
| Node                  | v1                                      |
| Namespace             | v1                                      |
| Secret                | v1                                      |
| PersistentVolume      | v1                                      |
| PersistentVolumeClaim | v1                                      |
| Pod                   | v1                                      |
| Deployment            | v1、apps/v1、apps/v1beta1、apps/v1beta2 |
| Service               | v1                                      |
| Ingress               | extensions/v1beta1                      |
| ReplicaSet            | apps/v1、apps/v1beta2                   |
| Job                   | batch/v1                                |
| StatefulSet           | apps/v1、apps/v1beta1、apps/v1beta2     |

## 三、kubectl命令合集

#### 配置各节点可用kubectl

```bash
# 1. 将 master 节点中 /etc/kubernetes/admin.conf 拷贝到需要运行的服务器的 /etc/kubernetes 目录中
scp /etc/kubernetes/admin.conf root@k8s-node1:/etc/kubernetes

# 2. 在对应的服务器上配置环境变量
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

常用命令

#### **创建对象**

```bash
$ kubectl create -f ./my-manifest.yaml           # 创建资源
$ kubectl create -f ./my1.yaml -f ./my2.yaml     # 使用多个文件创建资源
$ kubectl create -f ./dir                        # 使用目录下的所有清单文件来创建资源
$ kubectl create -f https://git.io/vPieo         # 使用 url 来创建资源
$ kubectl run nginx --image=nginx                # 启动一个 nginx 实例
$ kubectl explain pods,svc                       # 获取 pod 和 svc 的文档

```

#### **显示和查找资源**

```bash
# Get commands with basic output
$ kubectl get services                          # 列出所有 namespace 中的所有 service
$ kubectl get pods --all-namespaces             # 列出所有 namespace 中的所有 pod
$ kubectl get pods -o wide                      # 列出所有 pod 并显示详细信息
$ kubectl get deployment my-dep                 # 列出指定 deployment
$ kubectl get pods --include-uninitialized      # 列出该 namespace 中的所有 pod 包括未初始化的

# 使用详细输出来描述命令
$ kubectl describe nodes my-node
$ kubectl describe pods my-pod


$ kubectl get services --sort-by=.metadata.name # List Services Sorted by Name

# 根据重启次数排序列出 pod
$ kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# 获取所有具有 app=cassandra 的 pod 中的 version 标签
$ kubectl get pods --selector=app=cassandra rc -o \
  jsonpath='{.items[*].metadata.labels.version}'

# 获取所有节点的 ExternalIP
$ kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# 列出属于某个 PC 的 Pod 的名字
# “jq”命令用于转换复杂的 jsonpath，参考 https://stedolan.github.io/jq/
$ sel=${$(kubectl get rc my-rc --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
$ echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})

# 查看哪些节点已就绪
$ JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"

# 列出当前 Pod 中使用的 Secret
$ kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq

```

#### 更新资源

```bash
$ kubectl rolling-update frontend-v1 -f frontend-v2.json           # 滚动更新 pod frontend-v1
$ kubectl rolling-update frontend-v1 frontend-v2 --image=image:v2  # 更新资源名称并更新镜像
$ kubectl rolling-update frontend --image=image:v2                 # 更新 frontend pod 中的镜像
$ kubectl rolling-update frontend-v1 frontend-v2 --rollback        # 退出已存在的进行中的滚动更新
$ cat pod.json | kubectl replace -f -                              # 基于 stdin 输入的 JSON 替换 pod

# 强制替换，删除后重新创建资源。会导致服务中断。
$ kubectl replace --force -f ./pod.json

# 为 nginx RC 创建服务，启用本地 80 端口连接到容器上的 8000 端口
$ kubectl expose rc nginx --port=80 --target-port=8000

# 更新单容器 pod 的镜像版本（tag）到 v4
$ kubectl get pod mypod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -

$ kubectl label pods my-pod new-label=awesome                      # 添加标签
$ kubectl annotate pods my-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
$ kubectl autoscale deployment foo --min=2 --max=10                # 自动扩展 deployment “foo”

```

#### 修补资源

```bash
$ kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}' # 部分更新节点

# 更新容器镜像； spec.containers[*].name 是必须的，因为这是合并的关键字
$ kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'

# 使用具有位置数组的 json 补丁更新容器镜像
$ kubectl patch pod valid-pod --type='json' -p='[{"op": "replace", "path": "/spec/containers/0/image", "value":"new image"}]'

# 使用具有位置数组的 json 补丁禁用 deployment 的 livenessProbe
$ kubectl patch deployment valid-deployment  --type json   -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'


```

#### 编辑资源

```bash
$ kubectl edit svc/docker-registry                      # 编辑名为 docker-registry 的 service
$ KUBE_EDITOR="nano" kubectl edit svc/docker-registry   # 使用其它编辑器


```

#### 扩展副本

```bash
$ kubectl scale --replicas=3 rs/foo                                 # Scale a replicaset named 'foo' to 3
$ kubectl scale --replicas=3 -f foo.yaml                            # Scale a resource specified in "foo.yaml" to 3
$ kubectl scale --current-replicas=2 --replicas=3 deployment/mysql  # If the deployment named mysql's current size is 2, scale mysql to 3
$ kubectl scale --replicas=5 rc/foo rc/bar rc/baz                   # Scale multiple replication controllers

```

#### 删除资源

```bash
$ kubectl delete -f ./pod.json                                              # 删除 pod.json 文件中定义的类型和名称的 pod
$ kubectl delete pod,service baz foo                                        # 删除名为“baz”的 pod 和名为“foo”的 service
$ kubectl delete pods,services -l name=myLabel                              # 删除具有 name=myLabel 标签的 pod 和 serivce
$ kubectl delete pods,services -l name=myLabel --include-uninitialized      # 删除具有 name=myLabel 标签的 pod 和 service，包括尚未初始化的
$ kubectl -n my-ns delete po,svc --all                                      # 删除 my-ns namespace 下的所有 pod 和 serivce，包括尚未初始化的


```

#### 与运行的pod交互

```bash
$ kubectl logs my-pod                                 # dump 输出 pod 的日志（stdout）
$ kubectl logs my-pod -c my-container                 # dump 输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
$ kubectl logs -f my-pod                              # 流式输出 pod 的日志（stdout）
$ kubectl logs -f my-pod -c my-container              # 流式输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
$ kubectl run -i --tty busybox --image=busybox -- sh  # 交互式 shell 的方式运行 pod
$ kubectl attach my-pod -i                            # 连接到运行中的容器
$ kubectl port-forward my-pod 5000:6000               # 转发 pod 中的 6000 端口到本地的 5000 端口
$ kubectl exec my-pod -- ls /                         # 在已存在的容器中执行命令（只有一个容器的情况下）
$ kubectl exec my-pod -c my-container -- ls /         # 在已存在的容器中执行命令（pod 中有多个容器的情况下）
$ kubectl top pod POD_NAME --containers               # 显示指定 pod 和容器的指标度量

```

#### 节点操作

```bash
$ kubectl cordon my-node                                                # 标记 my-node 不可调度
$ kubectl drain my-node                                                 # 清空 my-node 以待维护
$ kubectl uncordon my-node                                              # 标记 my-node 可调度
$ kubectl top node my-node                                              # 显示 my-node 的指标度量
$ kubectl cluster-info                                                  # 显示 master 和服务的地址
$ kubectl cluster-info dump                                             # 将当前集群状态输出到 stdout                                    
$ kubectl cluster-info dump --output-directory=/path/to/cluster-state   # 将当前集群状态输出到 /path/to/cluster-state

# 如果该键和影响的污点（taint）已存在，则使用指定的值替换
$ kubectl taint nodes foo dedicated=special-user:NoSchedule

```



## 四、资源介绍

### pod

#### 基本概念

​	最小调度单元 Pod                

docker调度的是容器，在k8s集群中，最小的调度单元是Pod（豆荚）

<img src="C:\Users\ANYYNA\AppData\Roaming\Typora\typora-user-images\image-20230914091636033.png" alt="image-20230914091636033" style="zoom:60%;" />



![](images/pod-demo.png)

 引入Pod

- 与容器引擎解耦

  Docker、Rkt。平台设计与引擎的具体的实现解耦

- 多容器共享网络|存储|进程 空间, 支持的业务场景更加灵活

  <img src="images/image-20230914092246762.png" alt="image-20230914092246762" style="zoom:80%;" />





#### 创建一个pod的流程

![](images/process.png)

1. 用户准备一个资源文件（记录了业务应用的名称、镜像地址等信息），通过调用APIServer执行创建Pod
2. APIServer收到用户的Pod创建请求，通知controller-manager将Pod信息写入到etcd中
3. etcd存好信息后反馈回给api-server
4. api-server告知调度器（scheduler）事件信息，scheduler通过watch机制持续监听，发现有新的pod数据，但还没有绑定到某一个节点中
5. 调度器通过资源清单信息以及调度算法，计算出最适合该pod运行的节点，将pod和node的绑定信息告诉api-erver
6. api-server将调度结果更新到etcd中，更新好pod信息之后，继续反馈回给api-server
7. kubelet组件（目标node）同样通过watch方式，发现有新的pod调度到本机的节点了，根据pod的描述信息，拉取镜像，启动容器，同时生成事件信息
8. 同时，kubelet反馈pod创建结果，把容器的信息、事件及状态给api-server，并且写入到etcd中





如何理解namespace：

命名空间，集群内一个虚拟的概念，类似于资源池的概念，一个池子里可以有各种资源类型，绝大多数的资源都必须属于某一个namespace。集群初始化安装好之后，会默认有如下几个namespace：

```powershell
$ kubectl get namespaces
NAME                   STATUS   AGE
default                Active   84m
kube-node-lease        Active   84m
kube-public            Active   84m
kube-system            Active   84m
kubernetes-dashboard   Active   71m
```

- 所有NAMESPACED的资源，在创建的时候都需要指定namespace，若不指定，默认会在default命名空间下
- 相同namespace下的同类资源不可以重名，不同类型的资源可以重名
- 不同namespace下的同类资源可以重名
- 通常在项目使用的时候，我们会创建带有业务含义的namespace来做逻辑上的整合



#### 如何编写资源yaml

1. 拿来主义，从机器中已有的资源中拿

   ```powershell
   $ kubectl -n kube-system get po,deployment,ds
   ```

2. 学会在官网查找， https://kubernetes.io/docs/home/ 

3. 从kubernetes-api文档中查找， https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#pod-v1-core 

4. kubectl explain 查看具体字段含义

快速获得资源和版本

```powershell
$ kubectl explain pod
$ kubectl explain Pod.apiVersion
```

#### 创建访问编辑删除Pod

```powershell
## 创建namespace, namespace是逻辑上的资源池
$ kubectl create namespace demo

## 使用指定文件创建Pod
$ kubectl create -f demo-pod.yaml

## 查看pod，可以简写po
## 所有的操作都需要指定namespace，如果是在default命名空间下，则可以省略
$ kubectl -n demo get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE    IP             NODE
myblog   2/2     Running   0          3m     10.244.1.146   k8s-slave1

## 使用Pod Ip访问服务,3306和8002
$ curl 10.244.1.146:8002/blog/index/

## 进入容器,执行初始化, 不必到对应的主机执行docker exec
$ kubectl -n demo exec -ti myblog -c myblog bash
/ # env
/ # python3 manage.py migrate
$ kubectl -n demo exec -ti myblog -c mysql bash
/ # mysql -p123456

## 再次访问服务,3306和8002
$ curl 10.244.1.146:8002/blog/index/



## 查看pod调度节点及pod_ip
$ kubectl -n demo get pods -o wide
## 查看完整的yaml
$ kubectl -n demo get po myblog -o yaml
## 查看pod的明细信息及事件
$ kubectl -n demo describe pod myblog



#进入Pod内的容器
$ kubectl -n <namespace> exec <pod_name> -c <container_name> -ti /bin/sh

#查看Pod内容器日志,显示标准或者错误输出日志
$ kubectl -n <namespace> logs -f <pod_name> -c <container_name>

# 更新pod
$ kubectl apply -f demo-pod.yaml


#根据文件删除
$ kubectl delete -f demo-pod.yaml

#根据pod_name删除
$ kubectl -n <namespace> delete pod <pod_name>
```

#### Pause容器(Infra)

Pause容器或Infra容器，每个Pod里都会运行一个特殊的被称之为Pause的容器，而其他容器则为业务容器。这些业务容器共享Pause容器的网络栈和Volume挂载卷，使得它们之间的通信和数据交换更为高效。

**作用与功能**：

- **共享网络**：Pod下的所有容器共享同一个网络命名空间，而Infra容器就是实现这一功能的关键。它使得同一个Pod里的容器之间仅需通过localhost就能互相通信。
- **Linux命名空间共享的基础**：在Pod中，Infra容器担任Linux命名空间共享的基础。它使得Pod中的不同应用程序可以看到其他应用程序的进程ID（PID）。
- **启用pid命名空间和开启init进程**：Infra容器还负责启用pid命名空间并开启init进程，为Pod中的容器提供必要的运行环境。

- **解决网络故障问题**：在某些情况下，如给Pod注入网络故障以模拟网络延迟或丢包时，可能会出现目标容器重启进而导致故障恢复失败的问题。此时，重启相应的Pod可能是恢复故障的唯一方法。而Infra容器作为Pod的基础设施，其稳定性和可靠性对于整个Pod的正常运行至关重要。

- **生命周期**：Pod的生命周期从初始化容器开始，而Infra容器作为Pod的一部分，其生命周期与Pod相同。当Pod被创建时，Infra容器也会被创建；当Pod被删除时，Infra容器也会被相应地删除。

示例

```powershell
# pod：myblog中有三个容器，mysql和myblog程序以及Infra容器
## 登录master节点，查看pod内部的容器ip均相同，为pod ip
$ kubectl -n demo exec -ti myblog -c myblog bash
/ # ifconfig
$ kubectl -n demo exec -ti myblog -c mysql bash
/ # ifconfig
```

#### pod 健康检查

检测容器服务是否健康的手段，若不健康，会根据设置的重启策略（restartPolicy）进行操作，两种检测机制可以分别单独设置，若不设置，默认认为Pod是健康的。

两种机制：

- **LivenessProbe探针**
  
  用于判断容器是否存活，即Pod是否为running状态，如果LivenessProbe探针探测到容器不健康，则kubelet将kill掉容器，并根据容器的重启策略是否重启，如果一个容器不包含LivenessProbe探针，则Kubelet认为容器的LivenessProbe探针的返回值永远成功。 
  
- **ReadinessProbe探针**
  
  用于判断容器是否正常提供服务，即容器的Ready是否为True，是否可以接收请求，如果ReadinessProbe探测失败，则容器的Ready将为False，控制器将此Pod的Endpoint从对应的service的Endpoint列表中移除，从此不再将任何请求调度此Pod上，直到下次探测成功。（剔除此pod不参与接收请求不会将流量转发给此Pod）。
  
-  **startupProbe探针**

  由于有时候不能准确预估应用一定是多长时间启动成功，因此配置另外两种方式不方便配置初始化时长来检测，而配置了 statupProbe 后，只有在应用启动成功了，才会执行另外两种探针，可以更加方便的结合使用另外两种探针使用。

三种类型：

- execAction：通过执行命令来检查服务是否正常，回值为0则表示容器健康
- httpGet方式：通过发送http请求检查服务是否正常，返回200-399状态码则表明容器健康
- tcpSocket：通过容器的IP和Port执行TCP检查，如果能够建立TCP连接，则表明容器健康

示例：

```yaml
  containers:
  - name: myblog
    image: 172.21.32.6:5000/myblog
    ports:
    - containerPort: 8002
    livenessProbe:             # 这里使用livenessProbe为例子，举例配置语法，其他两种探针一样
      httpGet:                 # httpGet方式
        path: /blog/index/    
        port: 8002
        scheme: HTTP           # HTTP or HTTPS
        httpHeaders:           # 可选, 检查的请求头
      	  - name: xxx
            value: xxx
      failureThreshold: 3      # 探测成功后，最少连续探测失败多少次才被认定为失败。默认是3，最小值是1。
      successThreshold: 1      # 检查成功为 1 次表示就绪
      initialDelaySeconds: 10  # 容器启动后第一次执行探测，需要等待多少秒
      periodSeconds: 15 	   # 执行探测的频率，默认是10秒，最小1秒。
      timeoutSeconds: 2		   # 探测超时时间默认1秒，最小1秒。

#################
    # tcp连接检测
    livenessProbe:
 	  tcpSocket:
   	  port: 80

#################
    # 执行命令方式
    livenessProbe:
      exec:
        command:
          - cat
          - /health
```

结合探针使用：

会存在特殊情况：比如服务A启动时间很慢，需要60s。这个时候如果还是用上面的livenessProbe探针就会进入死循环，因为上面的探针10s后就开始探测，这时候我们服务并没有起来，发现探测失败就会触发restartPolicy。这时候可能会想到把`initialDelaySeconds`调成60s，但是我们并不能保证这个服务每次起来都是60s，假如新的版本起来要70s，甚至更多的时间，就不好控制。

这时候把`startupProbe`和`livenessProbe`结合起来使用就可以很大程度上解决我们的问题，`startupProbe`探测成功后再交给`livenessProbe`。如下配置：`startupProbe`配置的是`10*5s`，也就是说只要应用在50s内启动都是OK的，而且应用挂掉了10s就会发现问题。

```yaml
apiVersion: v1 # API 的版本号
kind: Pod # 类型 Pod
metadata: # 元数据
    name: nginx-startup # Pod 名称
spec: # 用于定义 Pod 的详细信息
    containers: # 必选，容器列表
    - name: nginx 
      image: nginx:1.15.12 # 容器所用的镜像的地址
      imagePullPolicy: IfNotPresent
      command: # 容器启动执行的命令
      - sh
      - -c
      - sleep 20; nginx -g "daemon off;"
      ports: # 容器需要暴露的端口号列表
      - containerPort: 80 # 端口号

      startupProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 10 # 初始化时间, 健康检查延迟执行时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 10 # 检测间隔
        successThreshold: 1 # 检查成功为 1 次表示就绪
        failureThreshold: 5 # 检测失败 3 次表示未就绪

      readinessProbe: 
        httpGet: # 接口检测方式
          path: /index.html # 检查路径
          port: 80
          scheme: HTTP # HTTP or HTTPS
          #httpHeaders: # 可选, 检查的请求头
          #- name: xxx
          #  value: xxx
        initialDelaySeconds: 10 # 初始化时间, 健康检查延迟执行时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 3 # 检测间隔
        successThreshold: 1 # 检查成功为 1 次表示就绪
        failureThreshold: 2 # 检测失败 2 次表示未就绪

      livenessProbe: 
        tcpSocket: # 端口检测方式
          port: 80
        initialDelaySeconds: 10 # 初始化时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 3 # 检测间隔
        successThreshold: 1 # 检查成功为1次表示就绪
        failureThreshold: 2 # 检测失败2次表示未就绪

      restartPolicy: Always  #重启策略，失败时总会重启
```





#### Pod 重启策略

Pod的重启策略（RestartPolicy）应用于Pod内的所有容器，并且仅在Pod所处的Node上由kubelet进行判断和重启操作。当某个容器异常退出或者健康检查失败时，kubelet将根据RestartPolicy的设置来进行相应的操作。
 Pod的重启策略包括Always、OnFailure和Never，默认值为Always。

- Always：当容器失败时，由kubelet自动重启该容器；
- OnFailure：当容器终止运行且退出码不为0时，有kubelet自动重启该容器；
- Never：不论容器运行状态如何，kubelet都不会重启该容器。

#### Pod 镜像拉取策略

```yaml
spec:
  containers:
  - name: myblog
    image: 172.21.32.6:5000/demo/myblog
    imagePullPolicy: IfNotPresent
```

设置镜像的拉取策略，默认为IfNotPresent

- Always，总是拉取镜像，即使本地有镜像也从仓库拉取
- IfNotPresent ，本地有则使用本地镜像，本地没有则去仓库拉取
- Never，只使用本地镜像，本地没有则报错



#### Pod 资源限制

为了保证充分利用集群资源，且确保重要容器在运行周期内能够分配到足够的资源稳定运行，因此平台需要具备

Pod的资源限制的能力。 对于一个pod来说，资源最基础的2个的指标就是：CPU和内存。

Kubernetes提供了个采用requests和limits 两种类型参数对资源进行预分配和使用限制。

pod-with-resourcelimits.yaml

```yaml
...
  containers:
  - name: myblog
    image: 172.21.32.6:5000/myblog
    env:
    - name: MYSQL_HOST   #  指定root用户的用户名
      value: "127.0.0.1"
    - name: MYSQL_PASSWD
      value: "123456"
    ports:
    - containerPort: 8002
    resources:
      requests:
        memory: 100Mi
        cpu: 50m
      limits:
        memory: 500Mi
        cpu: 100m
...
```

requests：

- 容器使用的最小资源需求,作用于schedule阶段，作为容器调度时资源分配的判断依赖
- 只有当前节点上可分配的资源量 >= request 时才允许将容器调度到该节点
- request参数不限制容器的最大可使用资源
- requests.cpu被转成docker的--cpu-shares参数，与cgroup cpu.shares功能相同 (无论宿主机有多少个cpu或者内核，--cpu-shares选项都会按照比例分配cpu资源）
- requests.memory没有对应的docker参数，仅作为k8s调度依据

limits：

- 容器能使用资源的最大值
- 设置为0表示对使用的资源不做限制, 可无限的使用
- 当pod 内存超过limit时，会被oom
- 当cpu超过limit时，不会被kill，但是会限制不超过limit值
- limits.cpu会被转换成docker的–cpu-quota参数。与cgroup cpu.cfs_quota_us功能相同
- limits.memory会被转换成docker的–memory参数。用来限制容器使用的最大内存

 对于 CPU，我们知道计算机里 CPU 的资源是按`“时间片”`的方式来进行分配的，系统里的每一个操作都需要 CPU 的处理，所以，哪个任务要是申请的 CPU 时间片越多，那么它得到的 CPU 资源就越多。

然后还需要了解下 CGroup 里面对于 CPU 资源的单位换算：

```shell
1 CPU =  1000 millicpu（1 Core = 1000m）
```

 这里的 `m` 就是毫、毫核的意思，Kubernetes 集群中的每一个节点可以通过操作系统的命令来确认本节点的 CPU 内核数量，然后将这个数量乘以1000，得到的就是节点总 CPU 总毫数。比如一个节点有四核，那么该节点的 CPU 总毫量为 4000m。 

`docker run`命令和 CPU 限制相关的所有选项如下：

| 选项                  | 描述                                                    |
| --------------------- | ------------------------------------------------------- |
| `--cpuset-cpus=""`    | 允许使用的 CPU 集，值可以为 0-3,0,1                     |
| `-c`,`--cpu-shares=0` | CPU 共享权值（相对权重）                                |
| `cpu-period=0`        | 限制 CPU CFS 的周期，范围从 100ms~1s，即[1000, 1000000] |
| `--cpu-quota=0`       | 限制 CPU CFS 配额，必须不小于1ms，即 >= 1000，绝对限制  |

```shell
docker run -it --cpu-period=50000 --cpu-quota=25000 ubuntu:16.04 /bin/bash
```

将 CFS 调度的周期设为 50000，将容器在每个周期内的 CPU 配额设置为 25000，表示该容器每 50ms 可以得到 50% 的 CPU 运行时间。

> 注意：若内存使用超出限制，会引发系统的OOM机制，因CPU是可压缩资源，不会引发Pod退出或重建



#### pod 状态与生命周期

Pod的状态如下表所示：

| 状态值            | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| **Pending**       | API Server已经创建该Pod，等待调度器调度                      |
| ContainerCreating | 镜像正在创建                                                 |
| **Running**       | Pod内容器均已创建，且至少有一个容器处于运行状态、正在启动状态或正在重启状态 |
| **Succeeded**     | Pod内所有容器均已成功执行退出，且不再重启                    |
| **Failed**        | Pod内所有容器均已退出，但至少有一个容器退出为失败状态        |
| CrashLoopBackOff  | Pod内有容器启动失败，比如配置文件丢失导致主进程启动失败      |
| **Unknown**       | 由于某种原因无法获取该Pod的状态，可能由于网络通信不畅导致    |

生命周期：

我们一般将pod对象从创建至终的这段时间范围称为pod的生命周期，它主要包含下面的过程：

- pod创建过程
- 运行初始化容器（init container）过程
- 运行主容器（main container）
  - 容器启动后钩子（post start）、容器终止前钩子（pre stop）
  - 容器的存活性探测（liveness probe）、就绪性探测（readiness probe）
- pod终止过程

生命周期示意图：

![](images/fcachl.jpg)

启动和关闭示意：

![](images/AOQgQj.jpg)

![img](images/image-20200412111402706-1626782188724.png)





```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-start-stop
  namespace: demo
  labels:
    component: demo-start-stop
spec:
  initContainers:
  - name: init
    image: busybox
    command: ['sh', '-c', 'echo $(date +%s): INIT >> /loap/timing']
    volumeMounts:
    - mountPath: /loap
      name: timing
  containers:
  - name: main
    image: busybox
    command: ['sh', '-c', 'echo $(date +%s): START >> /loap/timing;
sleep 10; echo $(date +%s): END >> /loap/timing;']
    volumeMounts:
    - mountPath: /loap 
      name: timing
    livenessProbe:
      exec:
        command: ['sh', '-c', 'echo $(date +%s): LIVENESS >> /loap/timing']
    readinessProbe:
      exec:
        command: ['sh', '-c', 'echo $(date +%s): READINESS >> /loap/timing']
    lifecycle:
      postStart:
        exec:
          command: ['sh', '-c', 'echo $(date +%s): POST-START >> /loap/timing']
      preStop:
        exec:
          command: ['sh', '-c', 'echo $(date +%s): PRE-STOP >> /loap/timing']
  volumes:
  - name: timing
    hostPath:
      path: /tmp/loap
```

创建pod测试：

```powershell
$ kubectl create -f demo-pod-start.yaml

## 查看demo状态
$ kubectl -n demo get po -o wide -w

## 查看调度节点的/tmp/loap/timing
$ cat /tmp/loap/timing
1585424708: INIT
1585424746: START
1585424746: POST-START
1585424754: READINESS
1585424756: LIVENESS
1585424756: END
```

>  须主动杀掉 Pod 才会触发 `pre-stop hook`，如果是 Pod 自己 Down 掉，则不会执行 `pre-stop hook` 

小结

1. 实现k8s平台与特定的容器运行时解耦，提供更加灵活的业务部署方式，引入了Pod概念
2. k8s使用yaml格式定义资源文件，yaml中Map与List的语法，与json做类比
3. 通过kubectl create | get | exec | logs | delete 等操作k8s资源，必须指定namespace
4. 每启动一个Pod，为了实现网络空间共享，会先创建Infra容器，并把其他容器网络加入该容器
5. 通过livenessProbe和readinessProbe实现Pod的存活性和就绪健康检查
6. 通过requests和limit分别限定容器初始资源申请与最高上限资源申请
7. 通过Pod IP访问具体的Pod服务，实现是

#### Pod 控制器     

只使用Pod, 将会面临如下需求:

1. 业务应用启动多个副本
2. Pod重建后IP会变化，外部如何访问Pod服务
3. 运行业务Pod的某个节点挂了，可以自动帮我把Pod转移到集群中的可用节点启动起来
4. 我的业务应用功能是收集节点监控数据,需要把Pod运行在k8集群的各个节点上

**Workload (工作负载)**

控制器又称工作负载是用于实现管理pod的中间层，确保pod资源符合预期的状态，pod的资源出现故障时，会尝试 进行重启，当根据重启策略无效，则会重新新建pod的资源。 

![](images/workload.png)

- ReplicaSet: 代用户创建指定数量的pod副本数量，确保pod副本数量符合预期状态，并且支持滚动式自动扩容和缩容功能
- Deployment：工作在ReplicaSet之上，用于管理无状态应用，目前来说最好的控制器。支持滚动更新和回滚功能，还提供声明式配置
- DaemonSet：用于确保集群中的每一个节点只运行特定的pod副本，通常用于实现系统级后台任务。比如ELK服务
- Job：只要完成就立即退出，不需要重启或重建
- Cronjob：周期性任务控制，不需要持续后台运行
- StatefulSet：管理有状态应用



#### Pod 驱逐策略

 K8S 有个特色功能叫 pod eviction，它在某些场景下如节点 NotReady，或者资源不足时，把 pod 驱逐至其它节点，这也是出于业务保护的角度去考虑的。

1. Kube-controller-manager: 周期性检查所有节点状态，当节点处于 NotReady 状态超过一段时间后，驱逐该节点上所有 pod。停掉kubelet
   - `pod-eviction-timeout`：NotReady 状态节点超过该时间后，执行驱逐，默认 5 min

2. Kubelet: 周期性检查本节点资源，当资源不足时，按照优先级驱逐部分 pod
   - `memory.available`：节点可用内存
   - `nodefs.available`：节点根盘可用存储空间
   - `nodefs.inodesFree`：节点inodes可用数量
   - `imagefs.available`：镜像存储盘的可用空间
   - `imagefs.inodesFree`：镜像存储盘的inodes可用数量

#### pod 调度策略    

##### **NodeSelector**

 标签匹配，`label`是`kubernetes`中一个非常重要的概念，用户可以非常灵活的利用 label 来管理集群中的资源，POD 的调度可以根据节点的 label 进行特定的部署。 

查看节点的label：

```powershell
$ kubectl get nodes --show-labels
```

为节点打label：

```powershell
$ kubectl label node k8s-master disktype=ssd
```

 当 node 被打上了相关标签后，在调度的时候就可以使用这些标签了，只需要在spec 字段中添加`nodeSelector`字段，里面是我们需要被调度的节点的 label。 

```yaml
...
spec:
  hostNetwork: true	# 声明pod的网络模式为host模式，效果通docker run --net=host
  volumes: 
  - name: mysql-data
    hostPath: 
      path: /opt/mysql/data
  nodeSelector:   # 使用节点选择器将Pod调度到指定label的节点
    component: mysql
  containers:
  - name: mysql
  	image: 172.21.32.6:5000/demo/mysql:5.7
...
```

##### **亲和性**

​		**nodeAffinity：**节点亲和性 ， 比上面的`nodeSelector`更加灵活，它可以进行一些简单的逻辑组合，不只是简单的相等匹配 。

​		**PodAffinity：**Pod 亲和力，将与指定 pod 亲和力相匹配的 pod 部署在同一节点。

​		**PodAntiAffinity：**Pod 反亲和力，根据策略尽量部署或不部署到一块。



分别都有两种特性：

 		**preferredDuringSchedulingIgnoredDuringExecution**：软策略，如果你没有满足调度要求的节点的话，Pod就会忽略这条规则，继续完成调度过程，说白了就是满足条件最好了，没有满足就忽略掉的策略。
 	
 		**requiredDuringSchedulingIgnoredDuringExecution** ： 硬策略，如果没有满足条件的节点的话，就不断重试直到满足条件为止，简单说就是你必须满足我的要求，不然我就不会调度Pod。

节点亲和性配置示例：

```yaml
#示例一：
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity: # 亲和力配置
    nodeAffinity: # 节点亲和力
      requiredDuringSchedulingIgnoredDuringExecution: # 节点必须匹配下方配置
        nodeSelectorTerms: # 选择器
        - matchExpressions: # 匹配表达式
          - key: topology.kubernetes.io/zone # 匹配 label 的 key
            operator: In # 匹配方式，只要匹配成功下方的一个 value 即可
            values:
            - antarctica-east1 # 匹配的 value
            - antarctica-west1 # 匹配的 value
      preferredDuringSchedulingIgnoredDuringExecution: # 节点尽量匹配下方配置
      - weight: 1 # 权重[1,100]，按照匹配规则对所有节点累加权重，最终之和会加入优先级评分，优先级越高被调度的可能性越高
        preference:
          matchExpressions: # 匹配表达式
          - key: another-node-label-key # label 的 key
            operator: In # 匹配方式，满足一个即可
            values:
            - another-node-label-value # 匹配的 value
#      - weight: 20
        ......
  containers:
  - name: with-node-affinity
    image: pause:2.0


```

pod亲和力与反亲和力：

```yaml
# pod亲和力与反亲和力示例2：
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity: # 亲和力配置
    podAffinity: # pod 亲和力配置
      requiredDuringSchedulingIgnoredDuringExecution: # 当前 pod 必须匹配到对应条件 pod 所在的 node 上
      - labelSelector: # 标签选择器
          matchExpressions: # 匹配表达式
          - key: security # 匹配的 key
            operator: In # 匹配方式
            values: # 匹配其中的一个 value
            - S1
        topologyKey: topology.kubernetes.io/zone
    
    podAntiAffinity: # pod 反亲和力配置
      preferredDuringSchedulingIgnoredDuringExecution: #尽量不要将当前节点部署到匹配下列参数的pod所在的node 上
      - weight: 100 # 权重
        podAffinityTerm: # pod 亲和力配置条件
          labelSelector: # 标签选择器
            matchExpressions: # 匹配表达式
            - key: security # 匹配的 key
              operator: In # 匹配的方式
              values:
              - S2 # 匹配的 value
          topologyKey: topology.kubernetes.io/zone
  
  containers:
  - name: with-pod-affinity
    image: pause:2.0
```

这里的匹配逻辑是 label 的值在某个列表中，现在`Kubernetes`提供的操作符有下面的几种：

- In：label 的值在某个列表中
- NotIn：label 的值不在某个列表中
- Gt：label 的值大于某个值
- Lt：label 的值小于某个值
- Exists：某个 label 存在
- DoesNotExist：某个 label 不存在

*如果nodeSelectorTerms下面有多个选项的话，满足任何一个条件就可以了；如果matchExpressions有多个选项的话，则必须同时满足这些条件才能正常调度 Pod*



##### **污点(Taints)与容忍(tolerations)**

对于`nodeAffinity`无论是硬策略还是软策略方式，都是调度 Pod 到预期节点上，而`Taints`恰好与之相反，如果一个节点标记为 Taints ，除非 Pod 也被标识为可以容忍污点节点，否则该 Taints 节点不会被调度Pod。

Taints(污点)是Node的一个属性，设置了Taints(污点)后，因为有了污点，所以Kubernetes是不会将Pod调度到这个Node上的。于是Kubernetes就给Pod设置了个属性Tolerations(容忍)，只要Pod能够容忍Node上的污点，那么Kubernetes就会忽略Node上的污点，就能够(不是必须)把Pod调度过去。

比如用户希望把 Master 节点保留给 Kubernetes 系统组件使用，或者把一组具有特殊资源预留给某些 Pod，则污点就很有用了，Pod 不会再被调度到 taint 标记过的节点。taint 标记节点举例如下：

设置污点：

```bash
$ kubectl taint node [node_name] key=value:[effect]   
      其中[effect] 可取值： [ NoSchedule | PreferNoSchedule | NoExecute ]
       NoSchedule：一定不能被调度。
       PreferNoSchedule：尽量不要调度。
       NoExecute：不仅不会调度，还会驱逐Node上已有的Pod。
  示例：kubectl taint node k8s-master smoke=true:NoSchedule
```

去除污点：

```powershell
去除指定key及其effect：
     kubectl taint nodes [node_name] key:[effect]-    #这里的key不用指定value
                
 去除指定key所有的effect: 
     kubectl taint nodes node_name key-
 
 示例：
     kubectl taint node k8s-master smoke=true:NoSchedule
     kubectl taint node k8s-master smoke:NoExecute-
     kubectl taint node k8s-master smoke-
```

污点演示：

```powershell
## 给k8s-slave1打上污点，smoke=true:NoSchedule
$ kubectl taint node k8s-slave1 smoke=true:NoSchedule
$ kubectl taint node k8s-slave2 drunk=true:NoSchedule


## 扩容myblog的Pod，观察新Pod的调度情况
$ kuebctl -n demo scale deploy myblog --replicas=3
$ kubectl -n demo get po -w    ## pending
```



Pod容忍污点示例：`myblog/deployment/deploy-myblog-taint.yaml`

```powershell
...
spec:
      containers:
      - name: demo
        image: 172.21.32.6:5000/demo/myblog
      tolerations: #设置容忍性
      - key: "smoke" 
        operator: "Equal"  #如果操作符为Exists，那么value属性可省略,不指定operator，默认为Equal
        value: "true"
        effect: "NoSchedule"
      - key: "drunk" 
        operator: "Equal"  #如果操作符为Exists，那么value属性可省略,不指定operator，默认为Equal
        value: "true"
        effect: "NoSchedule"
	  #意思是这个Pod要容忍的有污点的Node的key是smoke Equal true,效果是NoSchedule，
      #tolerations属性下各值必须使用引号，容忍的值都是设置Node的taints时给的值。
```

```powershell
$ kubectl apply -f deploy-myblog-taint.yaml
```

```powershell
spec:
      containers:
      - name: demo
        image: 172.21.32.6:5000/demo/myblog
      tolerations:
        - operator: "Exists"
```



#### Pod 控制器

Pod是kubernetes的最小管理单元，在kubernetes中，按照pod的创建方式可以将其分为两类：

- 自主式pod：kubernetes直接创建出来的Pod，这种pod删除后就没有了，也不会重建
- 控制器创建的pod：kubernetes通过控制器创建的pod，这种pod删除了之后还会自动重建

**什么是Pod控制器**：

​		Pod控制器是管理pod的中间层，使用Pod控制器之后，只需要告诉Pod控制器，想要多少个什么样的Pod就可以了，它会创建出满足条件的Pod并确保每一个Pod资源处于用户期望的目标状态。如果Pod资源在运行中出现故障，它会基于指定策略重新编排Pod。

在kubernetes中，有很多类型的pod控制器，每种都有自己的适合的场景，常见的有下面这些：

- ReplicationController：比较原始的pod控制器，已经被废弃，由ReplicaSet替代
- ReplicaSet：保证副本数量一直维持在期望值，并支持pod数量扩缩容，镜像版本升级
- Deployment：通过控制ReplicaSet来控制Pod，并支持滚动升级、回退版本
- Horizontal Pod Autoscaler：可以根据集群负载自动水平调整Pod的数量，实现削峰填谷
- DaemonSet：在集群中的指定Node上运行且仅运行一个副本，一般用于守护进程类的任务
- Job：它创建出来的pod只要完成任务就立即退出，不需要重启或重建，用于执行一次性任务
- Cronjob：它创建的Pod负责周期性任务控制，不需要持续后台运行
- StatefulSet：管理有状态应用



##### ReplicaSet(RS)

​		ReplicaSet的主要作用是**保证一定数量的pod正常运行**，它会持续监听这些Pod的运行状态，一旦Pod发生故障，就会重启或重建。同时它还支持对pod数量的扩缩容和镜像版本的升降级。

![img](images/image-20200612005334159.png)

配置文件示例：

```yaml
apiVersion: apps/v1 # 版本号
kind: ReplicaSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: rs
spec: # 详情描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

在这里面，需要新了解的配置项就是`spec`下面几个选项：

- replicas：指定副本数量，其实就是当前rs创建出来的pod的数量，默认为1

- selector：选择器，它的作用是建立pod控制器和pod之间的关联关系，采用的Label Selector机制

  在pod模板上定义label，在控制器上定义选择器，就可以表明当前控制器能管理哪些pod了

- template：模板，就是当前控制器创建pod所使用的模板板，里面其实就是前一章学过的pod的定义

**对RS资源进行操作**

扩缩容

1、命令操作：

```bash
kubectl -n dev scale deploy xxx  --replicas=3
```

2、修改yaml配置文件

```bash
kubectl edit deploy xxx -n dev

# 或者编辑yaml文件，更改replicas后，重新进行applay

```

3、镜像升级

```bash
# 配置文件升级
kubectl edit rs pc-replicaset -n dev

# 直接命令行操作
kubectl set image rs pc-replicaset nginx=nginx:1.17.1  -n dev
```

4、删除操作

```bash
# 使用kubectl delete命令会删除此RS以及它管理的Pod
# 在kubernetes删除RS前，会将RS的replicasclear调整为0，等待所有的Pod被删除后，在执行RS对象的删除
kubectl delete rs pc-replicaset -n dev

# 如果希望仅仅删除RS对象（保留Pod），可以使用kubectl delete命令时添加--cascade=false选项（不推荐）。
kubectl delete rs pc-replicaset -n dev --cascade=false


# 也可以使用yaml直接删除(推荐)
kubectl delete -f pc-replicaset.yaml
```



##### Deployment(deploy)

为了更好的解决服务编排的问题，kubernetes在V1.2版本开始，引入了Deployment控制器。值得一提的是，这种控制器并不直接管理pod，而是通过管理ReplicaSet来简介管理Pod，即：Deployment管理ReplicaSet，ReplicaSet管理Pod。所以Deployment比ReplicaSet功能更加强大。

![img](images/image-20200612005524778.png)

Deployment主要功能有下面几个：

- 支持ReplicaSet的所有功能
- 支持发布的停止、继续
- 支持滚动升级和回滚版本

配置示例：

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  revisionHistoryLimit: 3 # 保留历史版本
  paused: false # 暂停部署，默认是false
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

###### **扩缩容**

```bash
# 命令操作
kubectl -n dev scale deploy xxx  --replicas=3

# 在线修改yaml配置文件，或者编辑yaml文件，更改replicas后，重新进行applay
kubectl edit deploy xxx -n dev
```

###### **更新升级**

deployment支持两种更新策略:`重建更新`和`滚动更新`,可以通过`strategy`指定策略类型,支持两个属性:

```yaml
strategy：指定新的Pod替换旧的Pod的策略， 支持两个属性：
  type：指定策略类型，支持两种策略
    Recreate：在创建出新的Pod之前会先杀掉所有已存在的Pod
    RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本Pod
  rollingUpdate：当type为RollingUpdate时生效，用于为RollingUpdate设置参数，支持两个属性：
    maxUnavailable：用来指定在升级过程中不可用Pod的最大数量，默认为25%。
    maxSurge： 用来指定在升级过程中可以超过期望的Pod的最大数量，默认为25%。
```

###### 回退

deployment支持版本升级过程中的暂停、继续功能以及版本回退等诸多功能，下面具体来看.

kubectl rollout： 版本升级相关功能，支持下面的选项：

- status 显示当前升级状态
- history 显示 升级历史记录
- pause 暂停版本升级过程
- resume 继续已经暂停的版本升级过程
- restart 重启版本升级过程
- undo 回滚到上一级版本（可以使用--to-revision回滚到指定版本）

```bash
# 查看当前升级版本的状态
kubectl rollout status deploy pc-deployment -n dev

# 查看升级历史记录
[root@k8s-master01 ~]# kubectl rollout history deploy pc-deployment -n dev
deployment.apps/pc-deployment
REVISION  CHANGE-CAUSE
1         kubectl create --filename=pc-deployment.yaml --record=true
2         kubectl create --filename=pc-deployment.yaml --record=true
3         kubectl create --filename=pc-deployment.yaml --record=true


# 查到版本记录之后，进行回滚
# 这里直接使用--to-revision=1回滚到了1版本， 如果省略这个选项，就是回退到上个版本，就是2版本
[root@k8s-master01 ~]# kubectl rollout undo deployment pc-deployment --to-revision=1 -n dev



# 查看rs，发现第一个rs中有4个pod运行，后面两个版本的rs中pod为运行
# 其实deployment之所以可是实现版本的回滚，就是通过记录下历史rs来实现的，
# 一旦想回滚到哪个版本，只需要将当前版本pod数量降为0，然后将回滚版本的pod提升为目标数量就可以了
[root@k8s-master01 ~]# kubectl get rs -n dev
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6696798b78   4         4         4       78m
pc-deployment-966bf7f44    0         0         0       37m
pc-deployment-c848d767     0         0         0       71m
```



###### 灰度发布

灰度发布：又称金丝雀发布，一种平滑过渡的发布方式。它允许在产品更新过程中，将新版本的功能逐步推送给用户，而不是一次性全面更新。通过这种方式，可以在初始阶段就发现并调整问题，以保证整体系统的稳定。灰度发布的核心在于控制新版本功能的推送范围，从一小部分用户开始，逐步扩大，直到所有用户都迁移到新版本。

比如有一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布。

Deployment控制器支持控制更新过程中的控制，如“暂停(pause)”或“继续(resume)”更新操作。

```bash
# 更新deployment的版本，并配置暂停deployment
[root@k8s-master01 ~]#  kubectl set image deploy pc-deployment nginx=nginx:1.17.4 -n dev && kubectl rollout pause deployment pc-deployment  -n dev
deployment.apps/pc-deployment image updated
deployment.apps/pc-deployment paused

#观察更新状态，监控更新的过程，可以看到已经新增了一个资源，但是并未按照预期的状态去删除一个旧的资源，就是因为使用了pause暂停命令
[root@k8s-master01 ~]# kubectl rollout status deploy pc-deployment -n dev　
Waiting for deployment "pc-deployment" rollout to finish: 2 out of 4 new replicas have been updated...



[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   3         3         3       19m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       14m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   2         2         2       3m16s   nginx        nginx:1.17.4   

[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-5d89bdfbf9-rj8sq   1/1     Running   0          7m33s
pc-deployment-5d89bdfbf9-ttwgg   1/1     Running   0          7m35s
pc-deployment-5d89bdfbf9-v4wvc   1/1     Running   0          7m34s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          3m31s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          3m31s

# 确保更新的pod没问题了，继续更新
[root@k8s-master01 ~]# kubectl rollout resume deploy pc-deployment -n dev
deployment.apps/pc-deployment resumed

# 查看最后的更新情况
[root@k8s-master01 ~]# kubectl get rs -n dev -o wide
NAME                       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         
pc-deployment-5d89bdfbf9   0         0         0       21m     nginx        nginx:1.17.1   
pc-deployment-675d469f8b   0         0         0       16m     nginx        nginx:1.17.2   
pc-deployment-6c9f56fcfb   4         4         4       5m11s   nginx        nginx:1.17.4   

[root@k8s-master01 ~]# kubectl get pods -n dev
NAME                             READY   STATUS    RESTARTS   AGE
pc-deployment-6c9f56fcfb-7bfwh   1/1     Running   0          37s
pc-deployment-6c9f56fcfb-996rt   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-j2gtj   1/1     Running   0          5m27s
pc-deployment-6c9f56fcfb-rf84v   1/1     Running   0          37s
```

##### StatefulSet

管理有状态应用，需要进行持久化存储

示例：

```yml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:  # StatefulSet资源独有配置，一种声明性方式，它会根据模板自动创建PVC
  - metadata:
      name: www
      annotations:
        volume.alpha.kubernetes.io/storage-class: "managed-nfs-storage"  # 指定StorageClass，会自动创建对应的pvc
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```



##### HPA(Horizontal Pod Autoscaler)

###### 原理架构

HPA可以获取每个Pod的cpu，内存利用率，然后和HPA中定义的指标进行对比，同时计算出需要伸缩的具体值，最后实现Pod的数量的调整。其实HPA与之前的Deployment一样，也属于一种Kubernetes资源对象，它通过追踪分析RC控制的所有目标Pod的负载变化情况，来确定是否需要针对性地调整目标Pod的副本数，这是HPA的实现原理。

HorizontalPodAutoscaler 会指示工作负载资（Deployment、StatefulSet 或其他类似资源）扩容或缩减。由于DaemonSet会在每个节点上都会部署，不适用于水平Pod自动扩缩。

<img src="./images/image-20240425160513350.png" alt="image-20240425160513350" style="zoom: 50%;" />

在每个时间段内，控制器管理器都会根据每个 HorizontalPodAutoscaler 定义中指定的指标查询资源利用率：

- 控制器管理器找到由 `scaleTargetRef` 定义的目标资源
- 根据目标资源的 `.spec.selector` 标签选择 Pod
- 从资源指标 API（针对每个 Pod 的资源指标）或自定义指标获取指标 API（适用于所有其他指标）

Pod水平自动扩缩注意事项有以下方面：

- 必须安装metrics-server或其他自定义metrics-server
- 必须配置requests参数
- 不能扩容无法缩放的对象，比如DaemonSet

**查看所有Pod水平自动扩缩接口类型：**

```bash
$ kubectl get apiservices | grep autoscal
v1.autoscaling                         Local                        True        37d
v2.autoscaling                         Local                        True        37d
v2beta1.autoscaling                    Local                        True        37d
v2beta2.autoscaling                    Local                        True        37d
```

- v1：稳定版自动水平伸缩，只支持CPU指标
- v2beta1:支持CPU、内存和自定义指标
- v2beta2:支持CPU、内存、自定义指标Custom和额外指标ExternalMetrics



###### Metrics Server 

Kubernetes Metrics Server 是一个聚合和提供集群中资源使用情况数据的 API 服务。它收集并存储了关于节点和容器的度量数据，如 CPU、内存使用情况等，并通过 API 公开这些数据供其他组件和工具使用。

**主要特点：** 

1. 资源使用情况监控： Metrics Server 收集并存储了关于节点和容器的资源使用情况数据，包括 CPU 使用率、内存使用量、文件系统容量等。
2. 提供 API 接口： Metrics Server 提供了一个 RESTful API，允许用户查询和获取节点和容器的资源使用情况数据。
3. 水平扩展： Metrics Server 可以根据集群规模进行水平扩展，以处理大规模的度量数据。
4. 与 HPA 集成： Metrics Server 通常用于与 Horizontal Pod Autoscaler (HPA) 等控制器一起使用，根据资源使用情况自动调整 Pod 的副本数量。
5. 轻量级和稳定性： Metrics Server 的设计目标是轻量级和稳定，以便于在 Kubernetes 集群中运行，并提供资源监控数据。

**工作原理：**

- Metrics Server 会定期(默认15s，可更改参数- --metric-resolution，进行指定)从 kubelet 中收集节点和容器的度量数据。

- 这些数据通过 API 公开，可以通过 **kubectl top** 命令或其他 API 客户端查询。

- HPA 等控制器可以使用 Metrics Server 提供的数据来自动调整 Pod 的副本数量。

  <img src="./images/image-20240425200620793.png" alt="image-20240425200620793" style="zoom:67%;" />



**安装**

官方文档：https://github.com/kubernetes-sigs/metrics-server

获取到yaml文件

```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/

```

修改配置文件

默认开启了LTS认证，如果不改，将会有如下报错

```bash
I0425 10:02:45.361797       1 server.go:191] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
I0425 10:02:55.357319       1 server.go:191] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
E0425 10:02:59.290947       1 scraper.go:149] "Failed to scrape node" err="Get \"https://192.168.9.30:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 192.168.9.30 because it doesn't contain any IP SANs" node="k8s-master1"
E0425 10:02:59.298406       1 scraper.go:149] "Failed to scrape node" err="Get \"https://192.168.9.31:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 192.168.9.31 because it doesn't contain any IP SANs" node="k8s-node1"
E0425 10:02:59.320046       1 scraper.go:149] "Failed to scrape node" err="Get \"https://192.168.9.32:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 192.168.9.32 because it doesn't contain any IP SANs" node="k8s-node2"
```

```yaml
......
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  #添加配置，禁用lts证书验证，生产环境不建议
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.1
......
```

```bash
kubectl apply -f components.yaml
```

###### **HPA示例**

**Tips：使用HPA 必须要在pod中配置资源限制才生效，从 Kubernetes `v1.12` 版本开始我们可以通过设置 `kube-controller-manager` 组件的`--horizontal-pod-autoscaler-downscale-stabilization` 参数来设置一个持续时间，用于指定在当前操作完成后，`HPA` 必须等待多长时间才能执行另一次缩放操作。默认为5分钟，也就是默认需要等待5分钟后才会开始自动缩放。**

命令创建HPA基于cpu自动扩缩

```bash
kubectl autoscale deployment myapp-cpu --cpu-percent=10 --min=1 --max=10
```

```bash
--cpu-percent=10：指定 Deployment 的目标 CPU 利用率百分比。水平 Pod 自动缩放器将根据当前CPU利用率超过10%时进行自动调整副本数量，以尝试维持此目标。
--min=1：指定 Deployment 应具有的最小副本数。即使 CPU 利用率很低，水平 Pod 自动缩放器也不会将 Deployment 缩小到此数以下。
--max=10：指定 Deployment 应具有的最大副本数。如果 CPU 利用率非常高，水平 Pod 自动缩放器也不会将 Deployment 扩展到此数以上。
```

```bash
# 查看HorizontalPodAutoscaler的当前状态
kubectl get hpa myapp-cpu
```

- **autoscaling/v1**

定义yaml清单文件

```yaml
# cat hpa.yaml
# cpu
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa
  namespace: default
spec:
  scaleTargetRef:      # 定义目标作用对象，可以是Deployment、RS
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
    labelSelector:    # 匹配标签选择（如有多个需全部满足）
      matchLabels:
        app: my-app      # 这里是部署的标签
  maxReplicas: 5    # 最大副本数      
  minReplicas: 1    # 最小副本数
  targetCPUUtilizationPercentage: 10      # CPU利用率超过10%时进行自动调整副本数量，以尝试维持此目标

```

- **autoscaling/v2beta1**

```yaml
# 内存
apiVersion: autoscaling/v2beta1   # 内存必须是v2版本
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-mem-demo
  namespace: default
spec:
  scaleTargetRef:   # 指定对应的控制器
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-mem-demo
  maxReplicas: 5    # 最大副本数
  minReplicas: 1    # 最小副本数
  metrics: # 指定一个内存的配置
  - type: Resource    # 指标数据由metrics-server采集并提供给HPA控制器
    resource:
      name: memory
      targetAverageUtilization: 20         # 不能超过20%的使用率

```

```bash
其中type字段可为：
Resource：基于资源的指标值，指标数据由metrics-server采集并提供给HPA控制器，可以通过metrics.k8s.io这个API查询
Pods：基于Pod的指标
Object：基于某种资源对象，如ingress或者任意自定义的指标，Pods与Object是自定义指标，需要搭建自定义的Metrics Server和监控工具进行指标采集。指标数据通过custom.metrics.k8s.io进行查询，必须先启动自定义的Metrics Server。
```

type=pods

指标数据来源于Pod对象本身，target指标只能是**AverageValue**。

```yaml
  metrics:
  - type: Pods
    pods:
      metrics:
        name: packet-per-second
      target:
        type: AverageValue
        averageValue: 1k
```

type=object

指标数据来自其他资源或者是自定义的指标。target指标为**Value**或者是**AverageValue**

```yaml
#指标名称为requests-per-second，其值来源于Ingress"main-route"的目标值，即Ingress每秒请求量达到2000时触发扩容操作
    metrics：
    - type: Object
      object:
        metric:
          name: requests-per-second
        describeObject:
          apiVersion: extensions/v1beta1
          kind: Ingress
          name: main-route
        target:
          type: Value
          value: 2k


#指标名称为requests-per-second，其值来源于Ingress"main-route"的目标值，即Ingress每秒请求量达到2000时触发扩容操作
    metrics：
    - type: Object
      object:
        metric:
          name: 'http_requests'
          selector: 'verb=GET'
        target:
          type: AverageValue
          averageValue: 500

```



##### DaemonSet(DS)

DaemonSet类型的控制器可以保证在集群中的每一台（或指定）节点上都运行一个副本。一般适用于日志收集、节点监控等场景。也就是说，如果一个Pod提供的功能是节点级别的（每个节点都需要且只需要一个），那么这类Pod就适合使用DaemonSet类型的控制器创建。

![img](images/image-20200612010223537.png)

DaemonSet控制器的特点：

- 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上
- 当节点从集群中移除时，Pod 也就被垃圾回收了

配置示例：

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```



##### Job

Job，主要用于负责**批量处理(一次要处理指定数量任务)**短暂的**一次性(每个任务仅运行一次就结束)**任务。Job特点如下：

- 当Job创建的pod执行成功结束时，Job将记录成功结束的pod数量
- 当成功结束的pod达到指定的数量时，Job将完成执行

![img](images/image-20200618213054113.png)



配置示例

```yaml
apiVersion: batch/v1 # 版本号
kind: Job # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定job需要成功运行Pods的次数。默认值: 1
  parallelism: 1 # 指定job在任一时刻应该并发运行Pods的数量。默认值: 1
  activeDeadlineSeconds: 30 # 指定job可运行的时间期限，超过时间还未结束，系统将会尝试进行终止。
  backoffLimit: 6 # 指定job失败后进行重试的次数。默认是6
  manualSelector: true # 是否可以使用selector选择器选择pod，默认是false
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [counter-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为Never或者OnFailure
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 2;done"]
```



##### CronJob

CronJob控制器以 Job控制器资源为其管控对象，并借助它管理pod资源对象，Job控制器定义的作业任务在其控制器资源创建之后便会立即执行，但CronJob可以以类似于Linux操作系统的周期性任务作业计划的方式控制其运行**时间点**及**重复运行**的方式。也就是说，**CronJob可以在特定的时间点(反复的)去运行job任务**。

![img](images/image-20200618213149531.png)

配置示例：

```yaml
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: cronjob
spec: # 详情描述
  schedule: # cron格式的作业调度运行时间点,用于控制任务在什么时间执行
  concurrencyPolicy: # 并发执行策略，用于定义前一次作业运行尚未完成时是否以及如何运行后一次的作业
  failedJobHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为1
  successfulJobHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为3
  startingDeadlineSeconds: # 启动作业错误的超时时长
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象;下面其实就是job的定义
    metadata:
    spec:
      completions: 1
      parallelism: 1
      activeDeadlineSeconds: 30
      backoffLimit: 6
      manualSelector: true
      selector:
        matchLabels:
          app: counter-pod
        matchExpressions: 规则
          - {key: app, operator: In, values: [counter-pod]}
      template:
        metadata:
          labels:
            app: counter-pod
        spec:
          restartPolicy: Never 
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 20;done"]
```

注意

```yaml
需要重点解释的几个选项：
schedule: cron表达式，用于指定任务的执行时间
    */1    *      *    *     *
    <分钟> <小时> <日> <月份> <星期>

    分钟 值从 0 到 59.
    小时 值从 0 到 23.
    日 值从 1 到 31.
    月 值从 1 到 12.
    星期 值从 0 到 6, 0 代表星期日
    多个时间可以用逗号隔开； 范围可以用连字符给出；*可以作为通配符； /表示每...
concurrencyPolicy:
    Allow:   允许Jobs并发运行(默认)
    Forbid:  禁止并发运行，如果上一次运行尚未完成，则跳过下一次运行
    Replace: 替换，取消当前正在运行的作业并用新作业替换它
```



## Service 

### 介绍

service：负责东西流量（同层级/内部服务网络通信）的通信，相当于一组pod的LB，负责将请求分发给对应的pod。service会为这个LB提供一个IP，一般称为cluster IP 。

service很多时候只是一个概念，真正起到作用的其实是kube-proxy进程服务，每个节点上运行着一个kube-proxy 服务进程，当创建service的时候，会通过apiServer向etcd写入创建service的信息，而kube-proxy会基于监听(watch)的机制，当发现service变动时，它会将最新的service信息转换成对应的访问规则，每个节点上都会生成。

<img src="images/image-20200509121254425.png" alt="img" style="zoom:67%;" />

<img src="images/image-20231110204458077.png" alt="image-20231110204458077" style="zoom:67%;" />



### **kube-proxy**

kube-proxy的主要职责包括两大块：一块是侦听service更新事件，并更新service相关的访问规则，一块是侦听endpoint更新事件，更新endpoint相关的访问规则（如 KUBE-SVC-链中的规则），然后将包请求转入endpoint对应的Pod。如果某个service尚没有Pod创建，那么针对此service的请求将会被drop掉。

**工作模式：**

- **userspace 模式：**（1.1版本之前的默认模式）

​		userspace模式下，kube-proxy会为每一个Service创建一个监听端口，发向Cluster IP的请求被Iptables规则重定向到kube-proxy监听的端口上，kube-proxy根据LB算法选择一个提供服务的Pod并和其建立链接，以将请求转发到Pod上。 该模式下，kube-proxy充当了一个四层负责均衡器的角色。由于kube-proxy运行在userspace中，在进行转发处理时会增加内核和用户空间之间的数据拷贝，虽然比较稳定，但是效率比较低。

<img src="images/image-20200509152947714.png" alt="img" style="zoom:67%;" />

- **iptables 模式：（1.1-1.10版本的默认模式）**

​	iptables模式下，kube-proxy为service后端的每个Pod创建对应的iptables规则，直接将发向Cluster IP的请求重定向到一个Pod IP。 该模式下kube-proxy不承担四层负责均衡器的角色，只负责创建iptables规则。该模式的优点是较userspace模式效率更高，但不能提供灵活的LB策略，当后端Pod不可用时也无法进行重试。

​	iptables是把一些特定规则以“链”的形式挂载到每个Hook点，这些链的规则是固定的，而是是比较通用的，可以通过iptables命令在用户层动态的增删，这些链必须串行的执行。执行到某条规则，如果不匹配，则继续执行下一条，如果匹配，根据规则，可能继续向下执行，也可能跳到某条规则上，也可能下面的规则都跳过。


其核心功能：通过API Server的Watch接口实时跟踪Service与Endpoint的变更信息，并更新对应的iptables规则，Client的请求流量则通过iptables的NAT机制“直接路由”到目标Pod。

​	其实需要在宿主机上设置相当多的 iptables 规则。而且，kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的。不难想到，当你的宿主机上有大量 Pod 的时候，成百上千条 iptables 规则不断地被刷新，由于是线性查找匹配、全量更新等特点，其性能会显著下降，还会大量占用该宿主机的 CPU 资源，甚至会让宿主机“卡”在这个过程中。所以说，一直以来，基于 iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。

<img src="images/image-20200509152947714-170170582600110.png" alt="img" style="zoom:67%;" />

-  **ipvs 模式:**（1.11版本之后的默认模式，需要安装开启宿主机的ipvs模块，如果不启用，自动回退为iptables模式）


​	ipvs模式和iptables类似，kube-proxy监控Pod的变化并创建相应的ipvs规则。ipvs相对iptables转发效率更高。除此以外，ipvs支持更多的LB算法。

IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表）存访问规则，允许几乎无限的规模扩张，因此被kube-proxy采纳为最新模式。

在IPVS模式下，使用iptables的扩展ipset，而不是直接调用iptables来生成规则链。iptables规则链是一个线性的数据结构，ipset则引入了带索引的数据结构，因此当规则很多时，也可以很高效地查找和匹配。可以将ipset简单理解为一个IP（段）的集合，这个集合的内容可以是IP地址、IP网段、端口等，iptables可以直接添加规则对这个“可变的集合”进行操作，这样做的好处在于可以大大减少iptables规则的数量，从而减少性能损耗。

<img src="images/image-20200509153731363.png" alt="img" style="zoom:67%;" />



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



**ipvs的负载均衡策略：**

Service负载分发的策略有：

​		**RoundRobin：**默认为轮询模式，即轮询将请求转发到后端的各个Pod上。

​		**SessionAffinity：**基于客户端IP地址进行会话保持的模式，即第1次将某个客户端发起的请求转发到后端的某个Pod上，之后从相同的客户端发起的请求在10800s内都将被转发到后端相同的Pod上。此模式可以使在spec中添加`sessionAffinity:ClientIP`选项

```bash
# 此模式必须在所有master、node节点上安装ipvs内核模块，否则会降级为iptables
yum -y install ipset ipvsadm

# 配置ipvsadm模块开机自动执行加载相关模块
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

# 配置开机自执行加载，也可以写到rc.local文件下
cat <<EOF>> etc/rc.local
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

# 授权
chmod +x etc/rc.d/rc.local

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

```yaml
# service配置示例：

kind: Service  # 资源类型
apiVersion: v1  # 资源版本
metadata: # 元数据
  name: service # 资源名称
  namespace: dev # 命名空间
spec: # 描述
  selector: # 标签选择器，用于确定当前service代理哪些pod
    app: nginx
  type: # Service类型，指定service的访问方式
  clusterIP:  # 虚拟服务的ip地址
  sessionAffinity: # session亲和性，支持ClientIP、None两个选项
  ports: # 端口信息
    - protocol: TCP 
      port: 3017  # service端口
      targetPort: 5003 # pod端口
      nodePort: 31122 # 主机端口
```



**kube-proxy的 ipvs和iptables模式的异同：**

iptables与IPVS都是基于Netfilter实现的，但因为定位不同，二者有着本质的差别：

- iptables是为防火墙而设计的；

- IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张。


与iptables相比，IPVS拥有以下明显优势：

- 为大型集群提供了更好的可扩展性和性能；

- 支持比iptables更复杂的复制均衡算法（最小负载、最少连接、加权轮询等）；

- 支持服务器健康检查和连接重试等功能；

- 可以动态修改ipset的集合，即使iptables的规则正在使用这个集合；

- 并能够进行会话保持。




### k8s服务发现方式

DNS：可以通过cluster add-on的方式轻松的创建KubeDNS来对集群内的Service进行服务发现————这也是k8s官方强烈推荐的方式。为了让Pod中的容器可以使用kube-dns来解析域名，k8s会修改容器的/etc/resolv.conf配置。

CoreDNS：是一个DNS服务器，Kubernetes默认采用，以Pod部署在集群中， CoreDNS服务监视Kubernetes API，为每一个Service创建DNS记录用于域名解析。

服务发现实现：

 `CoreDNS`是一个`Go`语言实现的链式插件`DNS服务端`，是CNCF成员，是一个高性能、易扩展的`DNS服务端`。 

```powershell
$ kubectl -n kube-system get po -o wide|grep dns
coredns-d4475785-2w4hk             1/1     Running   0          4d22h   10.244.0.64       
coredns-d4475785-s49hq             1/1     Running   0          4d22h   10.244.0.65

# 查看myblog的pod解析配置
$ kubectl -n demo exec -ti myblog-5c97d79cdb-j485f bash
[root@myblog-5c97d79cdb-j485f myblog]# cat /etc/resolv.conf
nameserver 10.96.0.10
search demo.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

## 10.96.0.10 从哪来
$ kubectl -n kube-system get svc
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   51d

## 启动pod的时候，会把kube-dns服务的cluster-ip地址注入到pod的resolve解析配置中，同时添加对应的namespace的search域。 因此跨namespace通过service name访问的话，需要添加对应的namespace名称，
service_name.namespace_name
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   26h
```



### endpoint

Endpoint是kubernetes中的一个资源对象，存储在etcd中，用来记录一个service对应的所有pod的访问地址，它是根据service配置文件中selector描述产生的。

一个Service由一组Pod组成，这些Pod通过Endpoints暴露出来，**Endpoints是实现实际服务的端点集合**。换句话说，service和pod之间的联系是通过endpoints实现的。

service配置selector，endpoint controller才会自动创建对应的endpoint对象；否则，不会生成endpoint对象.

<img src="images/image-20200509191917069.png" alt="image-20200509191917069" style="zoom:67%;" />



例如，k8s集群中创建一个名为k8s-classic-1113-d3的service，就会生成一个同名的endpoint对象，如下图所示。其中ENDPOINTS就是service关联的pod的ip地址和端口。

<img src="images/format,png.png" alt="在这里插入图片描述" style="zoom:67%;" />

**endpoint controller**

endpoint controller是k8s集群控制器的其中一个组件，其功能如下：

​		负责生成和维护所有endpoint对象的控制器

​		负责监听service和对应pod的变化

​				监听到service被删除，则删除和该service同名的endpoint对象

​				监听到新的service被创建，则根据新建service信息获取相关pod列表，然后创建对应endpoint对象

​				监听到service被更新，则根据更新后的service信息获取相关pod列表，然后更新对应endpoint对象

​				监听到pod事件，则更新对应的service的endpoint对象，将podIp记录到endpoint中





### service类型

- ClusterIP：默认值，它是Kubernetes系统自动分配的虚拟IP，只能在集群内部访问
- NodePort：将Service通过指定的Node上的端口暴露给外部，通过此方法，就可以在集群外部访问服务
- LoadBalancer：使用外接负载均衡器完成到服务的负载分发，注意此模式需要外部云环境支持
- ExternalName： 把集群外部的服务引入集群内部，直接使用

#### ClusterIP

 clusterIP 主要在每个 node 节点使用 iptables，将发向 clusterIP 对应端口的数据，转发到 kube-proxy 中。然后 kube-proxy 自己内部实现有负载均衡的方法，并可以查询到这个 service 下对应 pod 的地址和端口，进而把数据转发给对应的 pod 的地址和端口

配置示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-clusterip
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: 10.97.97.97 # service的ip地址，如果不写，默认会生成一个
  type: ClusterIP
  ports:
  - port: 80  # Service端口       
    targetPort: 80 # pod端口
```



#### NodePort

如果希望将Service暴露给集群外部使用，那么就要使用到另外一种类型的Service，称为NodePort类型。NodePort的工作原理其实就是**将service的端口映射到Node的一个端口上**，然后就可以通过`NodeIp:NodePort`来访问service了。

![img](images/image-20200620175731338.png)

配置示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nodeport
  namespace: dev
spec:
  selector:
    app: nginx-pod
  type: NodePort # service类型
  ports:
  - port: 80
    protocol: TCP
    nodePort: 30002 # 指定绑定的node的端口(默认的取值范围是：30000-32767), 如果不指定，会默认分配
    targetPort: 80
```

#### LoadBalancer

LoadBalancer和NodePort很相似，目的都是向外部暴露一个端口，区别在于LoadBalancer会在集群的外部再来做一个负载均衡设备，而这个设备需要外部环境支持的，外部服务发送到这个设备上的请求，会被设备负载之后转发到集群中。

​	<img src="images/image-20200510103945494.png" alt="img" style="zoom:67%;" />



#### ExternalName

ExternalName类型的Service用于引入集群外部的服务，它通过`externalName`属性指定外部一个服务的地址，然后在集群内部访问此service就可以访问到外部的服务了。

![img](images/image-20200510113311209.png)

配置示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-externalname
  namespace: dev
spec:
  type: ExternalName # service类型
  externalName: www.baidu.com  #改成ip地址也可以
```



#### HeadLiness

Headless Services又叫无头服务，是一种特殊的service，其spec:clusterIP表示为None，这样在实际运行时就不会被分配ClusterIP。

**Service的区别**

- Headless不分配clusterIP
- Headless service可以通过解析service的DNS，返回所有Pod的地址和DNS(statefulSet部署的Pod才有DNS)
- 普通的service，只能通过解析service的DNS返回service的ClusterIP

**控制器的区别**

- statefulSet下的Pod有DNS地址,通过解析Pod的DNS可以返回Pod的IP
- deployment下的Pod没有DNS

**解析结果区别**

- **ClusterIP：**一个service对应一组endpoints(所有pod的地址+端口)，客户端使用DNS查询时只会返回Service的ClusterIP地址，具体Client访问的是哪个real server，由iptables或者ipvs决定

  ```yaml
  [root@k8s-master demo]# cat myapp.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: myapp
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: myapp
    template:
      metadata:
        labels:
          app: myapp
      spec:
        containers:
        - name: myapp
          image: ikubernetes/myapp:v1
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: myapp
  spec:
    type: ClusterIP
    selector:
      app: myapp
    ports:
    - port: 80
      targetPort: 80
  [root@k8s-master demo]# kubectl get svc
  NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  myapp            ClusterIP   10.108.21.3     <none>        80/TCP     5s
  
  [root@k8s-master demo]# kubectl exec -it test -- bash      # 进入一个test的pod中进行测试
  [root@test /]# nslookup myapp
  Server:         10.96.0.10
  Address:        10.96.0.10#53
  
  Name:   myapp.default.svc.cluster.local
  Address: 10.108.21.3
  ```

- **Headless service：**客户端可以通过dns查询返回多个endpoint，也就是多个pod地址和DNS，解析pod的DNS也能返回Pod的IP，注意使用headless和StatefulSet搭配使用

  ```yaml
  # 创建的StatefulSet资源
  [root@k8s-master demo]# cat myapp-sts.yaml 
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: mystatefulset
  spec:
    selector:
      matchLabels:
        app: myapp-headless
    serviceName: myapp-headless
    replicas: 2
    template:
      metadata:
        labels:
          app: myapp-headless
      spec:
        containers:
        - name: myapp
          image: ikubernetes/myapp:v2
          ports:
          - containerPort: 80
            name: web
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: myapp-headless
  spec:
    clusterIP: None     # 将clusterIP设置为None，即可创建headliness Service
    selector:
      app: myapp-headless
    ports:
    - port: 80
  [root@k8s-master demo]# kubectl get pod 
  NAME                     READY   STATUS    RESTARTS   AGE     
  mystatefulset-0          1/1     Running   0          103s    
  mystatefulset-1          1/1     Running   0          100s   
  
  [root@k8s-master demo]# kubectl get svc
  NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  myapp-headless   ClusterIP   None            <none>        80/TCP     23h
  
  [root@k8s-master demo]# kubectl exec -it test -- bash   # 进入一个test的pod中进行测试
  [root@test /]# nslookup myapp-headless
  Server:         10.96.0.10
  Address:        10.96.0.10#53
  
  Name:   myapp-headless.default.svc.cluster.local
  Address: 10.244.2.60
  Name:   myapp-headless.default.svc.cluster.local
  Address: 10.244.1.75
  ```

  通过Headless service访问指定的pod

  ```bash
  [root@k8s-master demo]# kubectl exec -it test -- bash
  [root@test /]# curl mystatefulset-0.myapp-headless.default.svc
  ```

  

**应用场景**

service的作用，主要是代理一组pod容器负载均衡服务，但是有时候我们不需要这种负载均衡场景，比如下面的两个例子。

- kubernetes部署某个kafka集群，这种就不需要service来代理，客户端需要的是一组pod的所有的ip。

- 还有一种场景客户端自己处理负载均衡的逻辑，比如kubernates部署两个mysql，两个pod之间需要互相访问，实现主从同步。

  

配置示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  ports:
  - port: 80    
    targetPort: 80
```



### ingress

对于Kubernetes的Service，无论是Cluster-Ip和NodePort均是四层的负载，集群内的服务如何实现七层的负载均衡，缺点：

- NodePort方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- LB方式的缺点是每个service需要一个LB，浪费、麻烦，并且需要kubernetes之外设备的支持



这就需要借助于Ingress，Ingress控制器的实现方式有很多，比如nginx, Contour, Haproxy, trafik, Istio，我们以nginx的实现为例做演示。

实际上，Ingress相当于一个7层的负载均衡器，是kubernetes对反向代理的一个抽象，它的工作原理类似于Nginx，可以理解成在**Ingress里建立诸多映射规则，Ingress Controller通过监听这些配置规则并转化成Nginx的反向代理配置 , 然后对外部提供服务**。，负责统一管理外部对k8s cluster中service的请求。主要包含：

- ingress资源对象：kubernetes中的一个对象，作用是定义请求如何转发到service的规则

- ingress controller：具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发，实现方式有很多，比如nginx, Contour, Haproxy, trafik, Istio等等

  

<img src="images/image-20231112113227785.png" alt="image-20231112113227785" style="zoom:67%;" />



Ingress（以Nginx为例）的工作原理如下：

1. 用户编写Ingress规则，说明哪个域名对应kubernetes集群中的哪个Service
2. ingress controller通过和apiServer交互，动态的去感知集群中ingress规则变化
3. 然后读取ingress规则(规则就是写明了哪个域名对应哪个service)，按照自定义的规则
4. 再写到nginx-ingress-controller的pod里，这个Ingress controller的pod里运行着一个Nginx服务，控制器把生成的nginx配置写入/etc/nginx.conf文件中
5. 然后reload一下使配置生效。以此达到域名分别配置和动态更新的问题。
6. 到此为止，其实真正在工作的就是一个Nginx了，内部配置了用户定义的请求转发规则

<img src="images/image-20200516112704764.png" alt="img" style="zoom:67%;" />



#### 暴露服务的方式

参考：https://blog.csdn.net/m0_73459977/article/details/129563646

**方式一：Deployment+LoadBalancer 模式的 Service**
		如果要把ingress部署在公有云，那用这种方式比较合适。用Deployment部署ingress-controller，创建一个 type为 LoadBalancer 的 service 关联这组 pod。大部分公有云，都会为 LoadBalancer 的 service 自动创建一个负载均衡器，通常还绑定了公网地址。 只要把域名解析指向该地址，就实现了集群服务的对外暴露

**方式二：DaemonSet+HostNetwork+nodeSelector**
		用DaemonSet结合nodeselector来部署ingress-controller到特定的node上，然后使用HostNetwork直接把该pod与宿主机node的网络打通，直接使用宿主机的80/433端口就能访问服务。这时，ingress-controller所在的node机器就很类似传统架构的边缘节点，比如机房入口的nginx服务器。该方式整个请求链路最简单，性能相对NodePort模式更好。缺点是由于直接利用宿主机节点的网络和端口，一个node只能部署一个ingress-controller pod。 比较适合大并发的生产环境使用。

**方式三：Deployment+NodePort模式的Service**
		同样用deployment模式部署ingress-controller，并创建对应的service，但是type为NodePort。这样，ingress就会暴露在集群节点ip的特定端口上。由于nodeport暴露的端口是随机端口，一般会在前面再搭建一套负载均衡器来转发请求。该方式一般用于宿主机是相对固定的环境ip地址不变的场景。

NodePort方式暴露ingress虽然简单方便，但是NodePort多了一层NAT，在请求量级很大时可能对性能会有一定影响。

#### 搭建ingress-nginx环境

官方文档

- GitHub：https://github.com/kubernetes/ingress-nginx
- kubernete：https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress

版本对应关系（在github上都有展示）：

| Ingress-NGINX version | k8s supported version        | Alpine Version | Nginx Version |
| --------------------- | ---------------------------- | -------------- | ------------- |
| v1.6.4                | 1.26, 1.25, 1.24, 1.23       | 3.17.0         | 1.21.6        |
| v1.5.1                | 1.25, 1.24, 1.23             | 3.16.2         | 1.21.6        |
| v1.4.0                | 1.25, 1.24, 1.23, 1.22       | 3.16.2         | 1.19.10†      |
| v1.3.1                | 1.24, 1.23, 1.22, 1.21, 1.20 | 3.16.2         | 1.19.10†      |
| v1.3.0                | 1.24, 1.23, 1.22, 1.21, 1.20 | 3.16.0         | 1.19.10†      |
| v1.2.1                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.6         | 1.19.10†      |
| v1.1.3                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.4         | 1.19.10†      |
| v1.1.2                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.2         | 1.19.9†       |
| v1.1.1                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.2         | 1.19.9†       |
| v1.1.0                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         | 1.19.9†       |
| v1.0.5                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         | 1.19.9†       |
| v1.0.4                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         | 1.19.9†       |
| v1.0.3                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         | 1.19.9†       |
| v1.0.2                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         | 1.19.9†       |
| v1.0.1                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         | 1.19.9†       |
| v1.0.0                | 1.22, 1.21, 1.20, 1.19       | 3.13.5         | 1.20.1        |

在github上找到应版本的信息

<img src="images/image-20231210233254198.png" alt="image-20231210233254198" style="zoom: 67%;" />

找到对应版本ingress-nginx-controller控制器的yaml配置文件：

<img src="images/image-20231210233501298.png" alt="image-20231210233501298" style="zoom:67%;" />



```bash
# 创建文件夹
[root@k8s-master01 ~]# mkdir ingress-controller && cd ingress-controller/

# 1、获取ingress-nginx-controller的yaml文件
# 本次案例使用的是0.30版本，1.22版本之前可安装此版本，不同的k8s的版本所支持的ingress版本会不一样，根据实际情况选择对应版本安装，RBAC相关资源从1.17版本开始改用rbac.authorization.k8s.io/v1，rbac.authorization.k8s.io/v1beta1在1.22版本即将弃用
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml

# 1.22版本之后可使用v1.1.1及以上
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml

# 替换deploy.yaml配置文件中需要的镜像，默认是国外的，访问不了
# 注释的为官方镜像地址，未注释的image为替换后的国内镜像地址（替换三处地方，其中kube-webhook-certgen替换两处）
    swr.cn-south-1.myhuaweicloud.com/myregistry/nginx-ingress-controller:v1.1.1
    swr.cn-south-1.myhuaweicloud.com/myregistry/kube-webhook-certgen:v1.1.1

#个人华为云镜像源，可直接docker pull使用
    swr.cn-south-1.myhuaweicloud.com/myregistry/nginx-ingress-controller:v1.7.0
    swr.cn-south-1.myhuaweicloud.com/myregistry/nginx-ingress-controller:v1.1.1
    swr.cn-south-1.myhuaweicloud.com/myregistry/kube-webhook-certgen:v1.7.0
    swr.cn-south-1.myhuaweicloud.com/myregistry/kube-webhook-certgen:v1.1.1
    
#网上没有对应版本的镜像地址的话，可使用魔法方法下载到本地之后上传到国内云镜像仓库上保存，方便后期下载
#kube-webhook-certgen镜像地址：https://hub.docker.com/r/dyrnq/kube-webhook-certgen


# 2、更改yaml配置文件
# 对外开放的访问模式
#方式一：service模式，创建一个service作为ingress统一的访问入口
# 较低版本在mandatory.yaml中（例如此处的0.30版本）没有直接嵌入service需要手工拉取创建
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml

# 在v1.1.1及以上版本的deploy.yaml中都应该嵌入了service配置，直接修改即可（此处配置文件有所省略不可直接copy）
vim deploy.yaml
...
apiVersion: v1
kind: Service
metadata:
	...
spec:
  externalTrafficPolicy: Local
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    port: 80
    nodePort: 31080  # 指定对外的端口
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    nodePort: 31443  # 指定对外的端口
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  #type: LoadBalancer   # 默认为LoadBalancer模式，这个模式是在公有云上使用的
  type: NodePort        # 修改的为NodePort模式
  ...
 

#方式二：hostnetwork模式（不用service-nodeport.yaml资源文件）
## 只需修改部署节点，此时就是共享宿主机的ip端口模式

$ vim mandatory.yaml
apiVersion: apps/v1
kind: DaemonSet    # 这里改为DaemonSet，使用DaemonSet去部署，使用deployment也可，但DaemonSet更受控制，可以固定到某个节点。就可直接在宿主机上暴露一个端口号，k8s外部的集群就可直接代理到Ingress上。若是Deployment，可能需要deploymentService可能暴露一个notePort，这样性能可能不是很好，推荐使用DaemonSet去部署，DaemonSet在去部署ingress的pod，暴露它的80和443端口，然后在外部的service代理到这个ingress所在的节点上的IP地址和端口号就可。
metadata:
  ...
spec:
  ...
  template:
   	...
    spec:
    	hostNetwork: true # 添加为host模式， 默认注释了hostNetwork 工作方式，以防止端口的在宿主机的冲突，没有绑定到宿主机 80 端口;
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
        ingress: "true"		#增加标签，来决定将ingress部署在哪些机器
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
          args:


# 3、创建ingress-nginx-controller
[root@k8s-master01 ingress-controller]# kubectl apply -f ./

# 查看ingress-nginx相关
[root@k8s-master01 ingress-controller]# kubectl get pod -n ingress-nginx
NAME                                           READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-fbf967dd5-4qpbp   1/1     Running   0          12h

# 查看ingress-nginx对外提供访问的service
[root@k8s-master01 ingress-controller]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.98.75.163   <none>        80:32240/TCP,443:31335/TCP   11h
```



#### 创建ingress-nginx资源

示例：`deployment/ingress.yaml`

```yaml
# extensions/v1beta1接口版本
vim ingress-app.yaml
apiVersion: extensions/v1beta1  # extensions/v1beta1 Ingress 在1.22版本即将弃用，变为networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-app-ingress
  annotations:
  	kubernetes.io/ingress.class: nginx    # 指定当前使用的ingress-controller类型，必须要制定，不然ingress-controller识别不了
spec:
  rules:
  - host: www.mcl.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80


# networking.k8s.io/v1接口版本：
vim ingress-app.yaml	  
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-app-ingress
  annotations:
  	kubernetes.io/ingress.class: nginx    # 指定当前使用的ingress-controller的版本类型
spec:
  rules:
  - host: www.mcl.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:   # 注意此处与extensions/v1beta1版本接口配置有所差别
          service:
            name: nginx
            port:
              number: 80
```

ingress-nginx-controller会动态生成upstream配置：

```yaml
...
                server_name myblog.devops.cn ;

                listen 80  ;
                listen [::]:80  ;
                listen 443  ssl http2 ;
                listen [::]:443  ssl http2 ;

                set $proxy_upstream_name "-";

                ssl_certificate_by_lua_block {
                        certificate.call()
                }

                location / {

                        set $namespace      "demo";
                        set $ingress_name   "myblog";
 ...
```

#### http访问

域名解析服务，将 `myblog.devops.cn`解析到ingress的地址上。ingress是支持多副本的，高可用的情况下，生产的配置是使用lb服务（内网F5设备，公网elb、slb、clb，解析到各ingress的机器，如何域名指向lb地址）

本机，添加如下hosts记录来演示效果。

```json
192.168.136.128 myblog.devops.cn
```

然后，访问 http://myblog.devops.cn/blog/index/



#### HTTPS访问

```powershell
#自签名证书
$ openssl req -x509 -nodes -days 2920 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=*.devops.cn/O=ingress-nginx"

# 证书信息保存到secret对象中，ingress-nginx会读取secret对象解析出证书加载到nginx配置中
$ kubectl -n demo create secret tls https-secret --key tls.key --cert tls.crt
secret/https-secret created
```

修改yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myblog-tls
  namespace: demo
  annotations:
  	kubernetes.io/ingress.class: nginx    # 指定当前使用的ingress-controller的版本类型
spec:
  rules:
  - host: myblog.devops.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: myblog
          servicePort: 80
  tls:
  - hosts:
    - myblog.devops.cn
    secretName: https-secret
```

然后，访问 https://myblog.devops.cn/blog/index/



#### 灰度发布

常见几种发布方式

1. 灰度发布：一种平滑过渡的发布方式。它允许在产品更新过程中，将新版本的功能逐步推送给用户，而不是一次性全面更新。通过这种方式，可以在初始阶段就发现并调整问题，以保证整体系统的稳定。灰度发布的核心在于控制新版本功能的推送范围，从一小部分用户开始，逐步扩大，直到所有用户都迁移到新版本。

2. 蓝绿发布：对服务的新版本进行冗余部署，一般新版本的机器规格和数量与旧版本保持一致，相当于该服务有两套完全相同的部署环境，只不过此时只有旧版本在对外提供服务，新版本作为热备。当服务进行版本升级时，我们只需将流量全部切换到新版本即可，旧版本作为热备。由于冗余部署的缘故，所以不必担心新版本的资源不够。如果新版本上线后出现严重的程序BUG，那么我们只需将流量全部切回至旧版本，大大缩短故障恢复的时间。待新版本完成 BUG 修复并重新部署之后，再将旧版本的流量切换到新版本。

   <img src="./images/image-20240425190814787.png" alt="image-20240425190814787" style="zoom: 33%;" />

3. A/B测试：需要明确的是，A/B测试和蓝绿部署以及金丝雀，完全是两回事，蓝绿部署和金丝雀是发布策略，目标是确保新上线的系统稳定，关注的是新系统的BUG、隐患。
   A/B测试是效果测试，同一时间有多个版本的服务对外服务，这些服务都是经过足够测试，达到了上线标准的服务，有差异但是没有新旧之分（它们上线时可能采用了蓝绿部署的方式）。
   A/B测试关注的是不同版本的服务的实际效果，譬如说转化率、订单情况等。
   A/B测试时，线上同时运行多个版本的服务，这些服务通常会有一些体验上的差异，譬如说页面样式、颜色、操作流程不同。相关人员通过分析各个版本服务的实际效果，选出效果最好的版本。
   在A/B测试中，需要能够控制流量的分配，譬如说，为A版本分配10%的流量，为B版本分配10%的流量，为C版本分配80%的流量。

Nginx Ingress支持通过注解方式annotations来实现不同场景下的发布，可满足灰度发布、蓝绿发布、A/B测试等业务场景。具体实现过程如下：

​	为服务创建两个Ingress，一个为常规Ingress，另一个为带nginx.ingress.kubernetes.io/canary: "true"注解的Ingress，称为Canary Ingress；为Canary Ingress配置流量切分策略annotation，两个Ingress相互配合，即可实现多种场景的发布和测试。

Nginx Ingress的annotation支持以下几种规则：

- **nginx.ingress.kubernetes.io/canary-weight**

​	基于服务权重的流量切分，适用于蓝绿部署。比如，当设置为100，则表示所有流量都将会转发给Canary Ingress对应的后端服务。

- **nginx.ingress.kubernetes.io/canary-by-header：**

​	基于Header的流量的切分，适用于灰度发布。当请求头中包含指定的header名称时，且值为“always”，那就会将该请求转发到Canary Ingress定义对应的后端服务。当值为“never”时，则不转发，适用于回滚到旧的服务版本。当为其他值时，则忽略该annotation，并通过优先级策略，将请求流量请求分发到其他的规则策略。

- **nginx.ingress.kubernetes.io/canary-by-header-value：**

​	必须与canary-by-header一起使用，可自定义请求头的取值（加上这个上面的always，never失效），包含但不限于“always”或“never”。当请求头的值命中指定的自定义值时，请求将会转发给Canary Ingress定义的对应后端服务，如果是其他值则忽略该annotation，并通过优先级将请求流量分配到其他规则。

- **nginx.ingress.kubernetes.io/canary-by-header-pattern：**

​	与canary-by-header-value类似，唯一区别是该annotation用正则表达式匹配请求头的值，而不是某一个固定值。

- **nginx.ingress.kubernetes.io/canary-by-cookie：**

​	基于cookie的流量转发，适用于灰度发布，与canary-by-header类似，这个annotation用于cookie，仅支持“always”和“never”，无法自定义取值。



**优先级顺序：**

canary-by-header-value  >  canary-by-header  >  canary-by-cookie  >  canary-weight



**同时部署两个相同版本的服务：**

示例：

旧版本nginx 

```bash
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: nginx-deploy
  namespace: myapp
spec:
  replicas: 2
  selector: 
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        imagePullPolicy: IfNotPresent
        image: nginx
        ports:
        - name: http
          containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: myapp
spec:
  selector: 
    app: nginx
  type: NodePort
  ports:
  - name: http
    port: 8091
    targetPort: 80


```

新版本nginx：

```bash
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: nginx-deploy-canary
  namespace: myapp
spec:
  replicas: 2
  selector: 
    matchLabels:
      app: nginx-canary
  template:
    metadata:
      labels:
        app: nginx-canary
    spec:
      containers:
      - name: nginx
        imagePullPolicy: IfNotPresent
        image: nginx
        ports:
        - name: http
          containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-canary
  namespace: myapp
spec:
  selector: 
    app: nginx-canary
  type: NodePort
  ports:
  - name: http
    port: 8091
    targetPort: 80

```



**ingress配置：**

旧版本：

```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata: 
  namespace: myapp
  name: ingress-nginx
  annotations:
    kubernetes.io/ingress.class: nginx    # 指定当前使用的ingress-controller的版本类型
spec:
  rules: 
  - host: nginx.any.com
    http:
      paths:
      - path: / 
        backend:
          serviceName: nginx
          servicePort: 80
```

新版本：

```bash
apiVersion: extensions/v1beta1
kind: Ingress
metadata: 
  name: ingress-nginx-canary
  namespace: myapp
  annotations: 
    kubernetes.io/ingress.class: nginx     # 指定当前使用的ingress-controller的版本类型
    nginx.ingress.kubernetes.io/canary: "true"   # 开启灰度发布模式
    #nginx.ingress.kubernetes.io/canary-weight: "30"  # 配置权重流量切分
    nginx.ingress.kubernetes.io/canary-by-header: "Region"  # 配置基于HTTP请求头流量切分的key
    #nginx.ingress.kubernetes.io/canary-by-header-value: "SZ" # 指定自定义请求头key的值
    nginx.ingress.kubernetes.io/canary-by-header-pattern: "SZ|GZ" # 自定义请求头key的值，支持正则
    nginx.ingress.kubernetes.io/canary-by-cookie: "user_from_cd"  # 匹配cookie携带user_from_cd,使用此ingress规则
spec:
  rules: 
  - host: nginx.any.com     # 保持与旧版本域名一致
    http:
      paths:
      - path: / 
        backend:
          serviceName: nginx-canary  # 新版本的service
          servicePort: 80
```

**优先级顺序：**

canary-by-header-value  >  canary-by-header  >  canary-by-cookie  >  canary-weight



curl测试：

```bash
# 使用curl请求时，携带请求头以及cookie，这里的nginx.any.com:31697是配置的域名及ingres的service的外部端口
curl -s -H "Region: SZ" --cookie "user_from_cd=always" nginx.any.com:31697

```





### configMap和Secret

解决问题：环境变量中敏感信息带来的安全隐患

为什么要统一管理环境变量

- 环境变量中有很多敏感的信息，比如账号密码，直接暴漏在yaml文件中存在安全性问题
- 团队内部一般存在多个项目，这些项目直接存在配置相同环境变量的情况，因此可以统一维护管理
- 对于开发、测试、生产环境，由于配置均不同，每套环境部署的时候都要修改yaml，带来额外的开销

k8s提供两类资源，configMap和Secret，可以用来实现业务配置的统一管理， 允许将配置文件与镜像文件分离，以使容器化的应用程序具有可移植性 。

![](images/configmap.png)

- configMap，通常用来管理应用的配置文件或者环境变量，`myblog/two-pod/configmap.yaml`

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: myblog
    namespace: demo
  data:
    MYSQL_HOST: "172.21.32.6"
    MYSQL_PORT: "3306"
  ```

- Secret，管理敏感类的信息，默认会base64编码存储，有三种类型

  - Service Account ：用来访问Kubernetes API，由Kubernetes自动创建，并且会自动挂载到Pod的/run/secrets/kubernetes.io/serviceaccount目录中；创建ServiceAccount后，Pod中指定serviceAccount后，自动创建该ServiceAccount对应的secret；
  - Opaque ： base64编码格式的Secret，用来存储密码、密钥等；
  - kubernetes.io/dockerconfigjson ：用来存储私有docker registry的认证信息。

  `myblog/two-pod/secret.yaml`

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: myblog
    namespace: demo
  type: Opaque
  data:
    MYSQL_USER: cm9vdA==		#注意加-n参数， echo -n root|base64
    MYSQL_PASSWD: MTIzNDU2
  ```

  创建并查看：

  ```powershell
  $ kubectl create -f secret.yaml
  $ kubectl -n demo get secret
  ```

  如果不习惯这种方式，可以通过如下方式：

  ```powershell
  $ cat secret.txt
  MYSQL_USER=root
  MYSQL_PASSWD=123456
  $ kubectl -n demo create secret generic myblog --from-env-file=secret.txt 
  ```

修改后的mysql的yaml，资源路径：`myblog/two-pod/mysql-with-config.yaml`

```yaml
...
spec:
  containers:
  - name: mysql
    image: 172.21.32.6:5000/mysql:5.7-utf8
    env:
    - name: MYSQL_USER
      valueFrom:
        secretKeyRef:
          name: myblog
          key: MYSQL_USER
    - name: MYSQL_PASSWD
      valueFrom:
        secretKeyRef:
          name: myblog
          key: MYSQL_PASSWD
    - name: MYSQL_DATABASE
      value: "myblog"
...
```

修改后的myblog的yaml，资源路径：`myblog/two-pod/myblog-with-config.yaml`

```yaml
spec:
  containers:
  - name: myblog
    image: 172.21.32.6:5000/myblog
    imagePullPolicy: IfNotPresent
    env:
    - name: MYSQL_HOST
      valueFrom:
        configMapKeyRef:
          name: myblog
          key: MYSQL_HOST
    - name: MYSQL_PORT
      valueFrom:
        configMapKeyRef:
          name: myblog
          key: MYSQL_PORT
    - name: MYSQL_USER
      valueFrom:
        secretKeyRef:
          name: myblog
          key: MYSQL_USER
    - name: MYSQL_PASSWD
      valueFrom:
        secretKeyRef:
          name: myblog
          key: MYSQL_PASSWD
```

在部署不同的环境时，pod的yaml无须再变化，只需要在每套环境中维护一套ConfigMap和Secret即可。但是注意configmap和secret不能跨namespace使用，且更新后，pod内的env不会自动更新，重建后方可更新。



## 五、持久化存储

### EmptyDir

EmptyDir是最基础的Volume类型，一个EmptyDir就是Host上的一个空目录。

EmptyDir是在Pod被分配到Node时创建的，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为kubernetes会自动分配一个目录，当Pod销毁时， EmptyDir中的数据也会被永久删除。 EmptyDir用途如下：

- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）

<img src="images/image-20200413174713773.png" alt="img" style="zoom:67%;" />



配置示例：

```bash
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:  # 将logs-volume挂在到nginx容器中，对应的目录为 /var/log/nginx
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件中内容
    volumeMounts:  # 将logs-volume 挂在到busybox容器中，对应的目录为 /logs
    - name: logs-volume
      mountPath: /logs
  volumes: # 声明volume， name为logs-volume，类型为emptyDir
  - name: logs-volume
    emptyDir: {}
```

### HostPath

EmptyDir中数据不会被持久化，它会随着Pod的结束而销毁，如果想简单的将数据持久化到主机中，可以选择HostPath。

HostPath就是将Node主机中一个实际目录挂在到Pod中，以供容器使用，这样的设计就可以保证Pod销毁了，但是数据依据可以存在于Node主机上。

配置示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    hostPath: 
      path: /root/logs
      type: DirectoryOrCreate  # 目录存在就使用，不存在就先创建后使用
      
```

关于type：

```
关于type的值的一点说明：
    DirectoryOrCreate 目录存在就使用，不存在就先创建后使用
    Directory   目录必须存在
    FileOrCreate  文件存在就使用，不存在就先创建后使用
    File 文件必须存在 
    Socket  unix套接字必须存在
    CharDevice  字符设备必须存在
    BlockDevice 块设备必须存在
```

### 网络存储(NFS)

HostPath可以解决数据持久化的问题，但是一旦Node节点故障了，Pod如果转移到了别的节点，又会出现问题了，此时需要准备单独的网络存储系统，比较常用的用NFS、Ceph、GlusterFS、Minio等。如果环境在公有云，可以使用公有云提供的NAS、对象存储等。在Kubernetes中，Volume也支持配置以上存储，用于挂载到Pod中实现数据的持久化

NFS是一个网络文件存储系统，可以搭建一台NFS服务器，然后将Pod中的存储直接连接到NFS系统上，这样的话，无论Pod在节点上怎么转移，只要Node跟NFS的对接没问题，数据就可以成功访问。

<img src="images/image-20200413215133559.png" alt="img" style="zoom:67%;" />



准备NFS服务器：

```bash
# 在nfs上安装nfs服务
[root@nfs ~]# yum install nfs-utils -y

# 准备一个共享目录
[root@nfs ~]# mkdir /root/data/nfs -pv

# 将共享目录以读写权限暴露给192.168.5.0/24网段中的所有主机
[root@nfs ~]# vim /etc/exports
[root@nfs ~]# more /etc/exports
/root/data/nfs     192.168.5.0/24(rw,no_root_squash)

# 启动nfs服务
[root@nfs ~]# systemctl restart nfs


# 在node上安装nfs服务，注意不需要启动
[root@k8s-master01 ~]# yum install nfs-utils -y
```

pod挂载nfs存储配置文件示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-nfs
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] 
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    nfs:
      server: 192.168.5.6  #nfs服务器地址
      path: /root/data/nfs #共享文件路径
```

另外还有nfs + stotageclass搭配进行使用，见高级存储-->制备器章节

### 高级存储

#### PV

前面已经学习了使用NFS提供存储，此时就要求用户会搭建NFS系统，并且会在yaml配置nfs。由于kubernetes支持的存储系统有很多，要求客户全都掌握，显然不现实。为了能够屏蔽底层存储实现的细节，方便用户使用， kubernetes引入PV和PVC两种资源对象。

- PV（Persistent Volume）是持久化卷的意思，是对底层的共享存储的一种抽象。一般情况下PV由kubernetes管理员进行创建和配置，它与底层具体的共享存储技术有关，并通过插件完成与共享存储的对接。
- PVC（Persistent Volume Claim）是持久卷声明的意思，是用户对于存储需求的一种声明。换句话说，PVC其实就是用户向kubernetes系统发出的一种资源需求申请。

<img src="images/image-20200514194111567.png" alt="img" style="zoom:67%;" />



使用了PV和PVC之后，工作可以得到进一步的细分：

- 存储：存储工程师维护
- PV： kubernetes管理员维护
- PVC：kubernetes用户维护

PV资源清单：

```yaml
apiVersion: v1  
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型，与底层真正存储对应
  capacity:  # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes:  # 访问模式
  storageClassName: # 存储类别
  persistentVolumeReclaimPolicy: # 回收策略
  

关键配置参数说明：
存储类型：底层实际存储的类型，kubernetes支持多种存储类型，每种存储类型的配置都有所差异
存储能力（capacity）：目前只支持存储空间的设置( storage=1Gi )，不过未来可能会加入IOPS、吞吐量等指标的配置
访问模式（accessModes）：用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：
    ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
    ReadOnlyMany（ROX）： 只读权限，可以被多个节点挂载
    ReadWriteMany（RWX）：读写权限，可以被多个节点挂载
需要注意的是，底层不同的存储类型可能支持的访问模式不同

回收策略（persistentVolumeReclaimPolicy）：当PV不再被使用了之后，对其的处理方式。目前支持三种策略：
    Retain （保留） 保留数据，需要管理员手工清理数据
    Recycle（回收） 清除 PV 中的数据，效果相当于执行 rm -rf /thevolume/*
    Delete （删除） 与 PV 相连的后端存储完成 volume 的删除操作，当然这常见于云服务商的存储服务
需要注意的是，底层不同的存储类型可能支持的回收策略不同

存储类别：PV可以通过storageClassName（制备器）参数指定一个存储类别
    具有特定类别的PV只能与请求了该类别的PVC进行绑定
    未设定类别的PV则只能与不请求任何类别的PVC进行绑定
    状态（status）

一个 PV 的生命周期中，可能会处于4中不同的阶段：
    Available（可用）： 表示可用状态，还未被任何 PVC 绑定
    Bound（已绑定）： 表示 PV 已经被 PVC 绑定
    Released（已释放）： 表示 PVC 被删除，但是资源还未被集群重新声明
    Failed（失败）： 表示该 PV 的自动回收失败
```

#### PVC

PVC是资源的申请，用来声明对存储空间、访问模式、存储类别需求信息

pvc资源配置示例：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
  namespace: dev
spec:
  accessModes: # 访问模式
  selector: # 采用标签对PV选择
  storageClassName: # 存储类别
  resources: # 请求空间
    requests:
      storage: 5Gi   #资源可以小于 pv 的，但是不能大于，如果大于就会匹配不到 pv
      
      
      
PVC 的关键配置参数说明：

访问模式（accessModes）:用于描述用户应用对存储资源的访问权限

选择条件（selector）：通过Label Selector的设置，可使PVC对于系统中己存在的PV进行筛选

存储类别（storageClassName）：PVC在定义时可以设定需要的后端存储的类别，只有设置了该class的pv才能被系统选出

资源请求（Resources ）：描述对存储资源的请求，
```

与pod绑定

```bash
containers:
  ......
  volumeMounts:
    - mountPath: /tmp/pvc
      name: nfs-pvc-test
volumes:
  - name: nfs-pvc-test
    persistentVolumeClaim:
      claimName: xxx # pvc 的名称
```

#### 制备器

Provisioner，k8s 中提供了一套自动创建 PV 的机制，就是基于 StorageClass 进行的，通过 StorageClass 可以实现仅仅配置 PVC，然后交由 StorageClass 根据 PVC 的需求动态创建 PV。

每个 StorageClass 都有一个制备器（Provisioner），用来决定使用哪个卷插件制备 PV。

pod调用流程： **pod ---》pvc ---》storageClass ---》pv**

statefulset调用流程：**statefulset(配置volumeClaimTemplates) ---》storageClass ---》pv**

**创建制备器**

- 先创用户Service Account

```yaml
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
```

- 创建storageclass资源 + 创建NFS provisioner （可以理解为nfs客户端），结合使用

```yaml
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
              value: 192.168.113.121   # nfs server端地址
            - name: NFS_PATH
              value: /data/nfs/rw      # 共享的目录
      volumes:
        - name: nfs-client-root       #存储卷的名称，和前面定义的保持一致
          nfs:
            server: 192.168.113.121   # nfs server端地址
            path: /data/nfs/rw        # 共享的目录
```

创建好storageclass之后，就可以让pod、statefulset等资源调用

示例：

1. NFS安装（见上面NFS章节）及 nfs provisioner资源准备

2. 创建 storageclass资源

3.  pod、statefulset资源调用

   - 创建pod调用PVC再调用storageClass：**pod ---》pvc ---》storageClass ---》pv**

   ```yaml
   kind: PersistentVolumeClaim         #创建PVC资源
   apiVersion: v1
   metadata:
     name: nginx-pvc         #PVC的名称
   spec:
     accessModes:            #定义对PV的访问模式，代表PV可以被多个PVC以读写模式挂载
       - ReadWriteMany
     resources:              #定义PVC资源的参数
       requests:             #设置具体资源需求
         storage: 200Mi      #表示申请200MI的空间资源
     storageClassName: nfs-storage          #指定存储类的名称，就指定上面创建的那个存储类。
   
   ---
   kind: Pod
   apiVersion: v1
   metadata:
     name: rbd-test-pod
   spec:
     containers:
     - name: rbd-test-pod
       image:  nginx
       volumeMounts:
         - name: pvc
           mountPath: "/mnt"
     volumes:
       - name: pvc
         persistentVolumeClaim:
           claimName: nginx-pvc
   ```

   - statefulset调用（无需创建pvc）：**statefulset(配置volumeClaimTemplates) ---》storageClass ---》pv**

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: web
   spec:
     serviceName: "nginx"
     replicas: 2
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:1.7.9
           ports:
           - containerPort: 80
             name: web
           volumeMounts:
           - name: www
             mountPath: /usr/share/nginx/html
     volumeClaimTemplates:  # StatefulSet资源独有配置，一种声明性方式，它会根据模板自动创建PVC
     - metadata:
         name: www
         annotations:
           volume.alpha.kubernetes.io/storage-class: "nfs-storage"  # 指定StorageClass
       spec:
         accessModes: [ "ReadWriteOnce" ]
         resources:
           requests:
             storage: 1Gi
   ```


#### 生命周期

PVC和PV是一一对应的，PV和PVC之间的相互作用遵循以下生命周期：

- **资源创建**：管理员手动创建底层存储和PV

- **资源绑定**：用户创建PVC，kubernetes负责根据PVC的声明去寻找PV，并绑定

  在用户定义好PVC之后，系统将根据PVC对存储资源的请求在已存在的PV中选择一个满足条件的

  - 一旦找到，就将该PV与用户定义的PVC进行绑定，用户的应用就可以使用这个PVC了
  - 如果找不到，PVC则会无限期处于Pending状态，直到等到系统管理员创建了一个符合其要求的PV

  PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再与其他PVC进行绑定了

- **资源使用**：用户可在pod中像volume一样使用pvc

  Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。

- **资源释放**：用户删除pvc来释放pv

  当存储资源使用完毕后，用户可以删除PVC，与该PVC绑定的PV将会被标记为“已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被留在存储设备上，只有在清除之后该PV才能再次使用。

- **资源回收**：kubernetes根据pv设置的回收策略进行资源的回收

<img src="images/image-20200515002806726.png" alt="img" style="zoom:67%;" />



### 配置存储

#### ConfigMap

ConfigMap是一种比较特殊的存储卷，它的主要作用是用来存储配置信息的。

简单示例：

创建

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap
  namespace: dev
data:
  info: |
    username:admin
    password:123456
```

挂载使用：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将configmap挂载到目录
    - name: config
      mountPath: /configmap/config
  volumes: # 引用configmap
  - name: config
    configMap:
      name: configmap
```



#### Secret

在kubernetes中，还存在一种和ConfigMap非常类似的对象，称为Secret对象。它主要用于存储敏感信息，例如密码、秘钥、证书等等。

配置示例：

创建

```bash
# 首先使用base64对数据进行编码
[root@k8s-master01 ~]# echo -n 'admin' | base64 #准备username
YWRtaW4=
[root@k8s-master01 ~]# echo -n '123456' | base64 #准备password
MTIzNDU2

#secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: dev
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

挂载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将secret挂载到目录
    - name: config
      mountPath: /secret/config
  volumes:
  - name: config
    secret:
      secretName: secret
```



## 六、认证与授权    

Kubernetes作为一个分布式集群的管理工具，保证集群的安全性是其一个重要的任务。所谓的安全性其实就是保证对Kubernetes的各种**客户端**进行**认证和鉴权**操作。

在Kubernetes集群中，客户端通常有两类：

- **User Account**：一般是独立于kubernetes之外的其他服务管理的用户账号。

- **Service Account**：kubernetes管理的账号，用于为Pod中的服务进程在访问Kubernetes时提供身份标识。

  

### APIService 安全控制

ApiServer是访问及管理资源对象的唯一入口。任何一个请求访问ApiServer，都要经过下面三个流程：

<img src="images/image-20200520103942580.png" alt="img" style="zoom: 80%;" />



#### **Authentication：身份认证**

1. 这个环节它面对的输入是整个`http request`，负责对来自client的请求进行身份校验，支持的方法包括:

   - HTTP base 认证：用户名+密码的方式认证

     ​	    这种认证方式是把“用户名:密码”用BASE64算法进行编码后的字符串放在HTTP请求中的Header Authorization域里发送给服务端。服务端收到后进行解码，获取用户名及密码，然后进行用户身份认证的过程。

   - HTTP token：通过一个Token来识别合法用户

     ​		    这种认证方式是用一个很长的难以被模仿的字符串--Token来表明客户身份的一种方式。每个Token对应一个用户名，当客户端发起API调用请求时，需要在HTTP Header里放入Token，API Server接到Token后会跟服务器中保存的token进行比对，然后进行用户身份认证的过程。

   - HTTPS 证书验证（https双向验证）

     ​		    这种认证方式是安全性最高的一种方式，但是同时也是操作起来最麻烦的一种方式。

   - `jwt token`(用于serviceaccount)


2. APIServer启动时，可以指定一种Authentication方法，也可以指定多种方法。如果指定了多种方法，那么APIServer将会逐个使用这些方法对客户端请求进行验证， 只要请求数据通过其中一种方法的验证，APIServer就会认为Authentication成功；

3. 使用kubeadm引导启动的k8s集群的apiserver初始配置中，默认支持`client证书`验证和`serviceaccount`两种身份验证方式。 证书认证通过设置`--client-ca-file`根证书以及`--tls-cert-file`和`--tls-private-key-file`来开启。

4. 在这个环节，apiserver会通过client证书或 `http header`中的字段(比如serviceaccount的`jwt token`)来识别出请求的`用户身份`，包括”user”、”group”等，这些信息将在后面的`authorization`环节用到。



**HTTPS证书认证：（TLS 双向认证）**

<img src="images/image-20200518211037434.png" alt="img" style="zoom: 80%;" />



**HTTPS认证大体分为3个过程：**

1、证书申请和下发

```
  HTTPS通信双方的服务器向CA机构申请证书，CA机构下发根证书、服务端证书及私钥给申请者
```

2、客户端和服务端的双向认证

```
  1> 客户端向服务器端发起请求，服务端下发自己的证书给客户端，
     客户端接收到证书后，通过私钥解密证书，在证书中获得服务端的公钥，
     客户端利用服务器端的公钥认证证书中的信息，如果一致，则认可这个服务器
  2> 客户端发送自己的证书给服务器端，服务端接收到证书后，通过私钥解密证书，
     在证书中获得客户端的公钥，并用该公钥认证证书信息，确认客户端是否合法
```

3、服务器端和客户端进行通信

```
  服务器端和客户端协商好加密方案后，客户端会产生一个随机的秘钥并加密，然后发送到服务器端。
  服务器端接收这个秘钥后，双方接下来通信的所有内容都通过该随机秘钥加密
```



#### **Authorization：鉴权管理**

可以访问哪些资源

每个发送到ApiServer的请求都带上了用户和资源的信息：比如发送请求的用户、请求的路径、请求的动作等，授权就是根据这些信息和授权策略进行比较，如果符合策略，则认为授权通过，否则会返回错误。

1. 这个环节面对的输入是`http request context`中的各种属性，包括：`user`、`group`、`request path`（比如：`/api/v1`、`/healthz`、`/version`等）、 `request verb`(比如：`get`、`list`、`create`等)。
2. APIServer会将这些属性值与事先配置好的访问策略(`access policy`）相比较。APIServer支持多种`authorization mode`：
   - AlwaysDeny：表示拒绝所有请求，一般用于测试
   - AlwaysAllow：允许接收所有请求，相当于集群不需要授权流程（Kubernetes默认的策略）
   - ABAC：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制
   - Webhook：通过调用外部REST服务对用户进行授权
   - Node：是一种专用模式，用于对kubelet发出的请求进行访问控制
   - RBAC：基于角色的访问控制（kubeadm安装方式下的默认选项）
3. APIServer启动时，可以指定一种`authorization mode`，也可以指定多种`authorization mode`，如果是后者，只要Request通过了其中一种mode的授权， 那么该环节的最终结果就是授权成功。在较新版本kubeadm引导启动的k8s集群的apiserver初始配置中，`authorization-mode`的默认配置是`”Node,RBAC”`。



#### Admission Control：[准入控制](http://docs.kubernetes.org.cn/144.html)

一个控制链(层层关卡)，偏集群安全控制、管理方面。

准入控制是一个可配置的控制器列表，可以通过在Api-Server上通过命令行设置选择执行哪些准入控制器：

```bash
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds
```

当前可配置的Admission Control准入控制如下：

- AlwaysAdmit：允许所有请求
- AlwaysDeny：禁止所有请求，一般用于测试
- AlwaysPullImages：在启动容器之前总去下载镜像
- DenyExecOnPrivileged：它会拦截所有想在Privileged Container上执行命令的请求
- ImagePolicyWebhook：这个插件将允许后端的一个Webhook程序来完成admission controller的功能。
- Service Account：实现ServiceAccount实现了自动化
- SecurityContextDeny：这个插件将使用SecurityContext的Pod中的定义全部失效
- ResourceQuota：用于资源配额管理目的，观察所有请求，确保在namespace上的配额不会超标
- LimitRanger：用于资源限制管理，作用于namespace上，确保对Pod进行资源限制
- InitialResources：为未设置资源请求与限制的Pod，根据其镜像的历史资源的使用情况进行设置
- NamespaceLifecycle：如果尝试在一个不存在的namespace中创建资源对象，则该创建请求将被拒绝。当删除一个namespace时，系统将会删除该namespace中所有对象。
- DefaultStorageClass：为了实现共享存储的动态供应，为未指定StorageClass或PV的PVC尝试匹配默认的StorageClass，尽可能减少用户在申请PVC时所需了解的后端存储细节
- DefaultTolerationSeconds：这个插件为那些没有设置forgiveness tolerations并具有notready:NoExecute和unreachable:NoExecute两种taints的Pod设置默认的“容忍”时间，为5min
- PodSecurityPolicy：这个插件用于在创建或修改Pod时决定是否根据Pod的security context和可用的PodSecurityPolicy对Pod的安全策略进行控制
- NodeRestriction， 此插件限制kubelet修改Node和Pod对象，这样的kubelets只允许修改绑定到Node的Pod API对象，以后版本可能会增加额外的限制 。



**以NamespaceLifecycle为例， 该插件确保处于Termination状态的Namespace不再接收新的对象创建请求，并拒绝请求不存在的Namespace。该插件还可以防止删除系统保留的Namespace:default，kube-system，kube-public。** 





### kubectl 的认证授权

kubectl的日志调试级别：

| 信息 | 描述                                                         |
| :--- | :----------------------------------------------------------- |
| v=0  | 通常，这对操作者来说总是可见的。                             |
| v=1  | 当您不想要很详细的输出时，这个是一个合理的默认日志级别。     |
| v=2  | 有关服务和重要日志消息的有用稳定状态信息，这些信息可能与系统中的重大更改相关。这是大多数系统推荐的默认日志级别。 |
| v=3  | 关于更改的扩展信息。                                         |
| v=4  | 调试级别信息。                                               |
| v=6  | 显示请求资源。                                               |
| v=7  | 显示 HTTP 请求头。                                           |
| v=8  | 显示 HTTP 请求内容。                                         |
| v=9  | 显示 HTTP 请求内容，并且不截断内容。                         |

```powershell
$ kubectl get nodes -v=7
I0329 20:20:08.633065    3979 loader.go:359] Config loaded from file /root/.kube/config
I0329 20:20:08.633797    3979 round_trippers.go:416] GET https://192.168.136.128:6443/api/v1/nodes?limit=500
```

`kubeadm init`启动完master节点后，会默认输出类似下面的提示内容：

```bash
... ...
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
... ...
```

这些信息是在告知我们如何配置`kubeconfig`文件。按照上述命令配置后，master节点上的`kubectl`就可以直接使用`$HOME/.kube/config`的信息访问`k8s cluster`了。 并且，通过这种配置方式，`kubectl`也拥有了整个集群的管理员(root)权限。

很多K8s初学者在这里都会有疑问：

- 当`kubectl`使用这种`kubeconfig`方式访问集群时，`Kubernetes`的`kube-apiserver`是如何对来自`kubectl`的访问进行身份验证(`authentication`)和授权(`authorization`)的呢？
- 为什么来自`kubectl`的请求拥有最高的管理员权限呢？

查看`/root/.kube/config`文件：

前面提到过apiserver的authentication支持通过`tls client certificate、basic auth、token`等方式对客户端发起的请求进行身份校验， 从kubeconfig信息来看，kubectl显然在请求中使用了`tls client certificate`的方式，即客户端的证书。 

证书base64解码：

```powershell
$ echo xxxxxxxxxxxxxx |base64 -d > kubectl.crt
```

 说明在认证阶段，`apiserver`会首先使用`--client-ca-file`配置的CA证书去验证kubectl提供的证书的有效性,基本的方式 ：

```powershell
$  openssl verify -CAfile /etc/kubernetes/pki/ca.crt kubectl.crt
kubectl.crt: OK
```

除了认证身份，还会取出必要的信息供授权阶段使用，文本形式查看证书内容：

```powershell
$ openssl x509 -in kubectl.crt -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4736260165981664452 (0x41ba9386f52b74c4)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Feb 10 07:33:39 2020 GMT
            Not After : Feb  9 07:33:40 2021 GMT
        Subject: O=system:masters, CN=kubernetes-admin
        ...
```

认证通过后，提取出签发证书时指定的CN(Common Name),`kubernetes-admin`，作为请求的用户名 (User Name), 从证书中提取O(Organization)字段作为请求用户所属的组 (Group)，`group = system:masters`，然后传递给后面的授权模块。

 kubeadm在init初始引导集群启动过程中，创建了许多`default`的`role、clusterrole、rolebinding`和`clusterrolebinding`， 在k8s有关RBAC的官方文档中，我们看到下面一些`default clusterrole`列表: 

![](C:\Users\liyongxin\Desktop\wework\老男孩\训练营\images\kubeadm-default-clusterrole-list.png)

其中第一个cluster-admin这个cluster role binding绑定了system:masters group，这和authentication环节传递过来的身份信息不谋而合。 沿着system:masters group对应的cluster-admin clusterrolebinding“追查”下去，真相就会浮出水面。

我们查看一下这一binding：

```yaml
$ kubectl describe clusterrolebinding cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  system:masters
```

我们看到在kube-system名字空间中，一个名为cluster-admin的clusterrolebinding将cluster-admin cluster role与system:masters Group绑定到了一起， 赋予了所有归属于system:masters Group中用户cluster-admin角色所拥有的权限。

我们再来查看一下cluster-admin这个role的具体权限信息：

```powershell
$ kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```

非资源类，如查看集群健康状态。

![](images/how-kubectl-be-authorized.png)

### RBAC

 Role-Based Access Control，基于角色的访问控制， apiserver启动参数添加--authorization-mode=RBAC 来启用RBAC认证模式，kubeadm安装的集群默认已开启。[官方介绍](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

查看开启：

```powershell
# master节点查看apiserver进程
$ ps aux |grep apiserver
```

RBAC模式引入了4个资源：

- **Role：角色，一个Role只能授权访问单个namespace** 

  ```yaml
  ## 示例定义一个名为pod-reader的角色，该角色具有读取default这个命名空间下的pods的权限
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    namespace: default
    name: pod-reader
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
    
  ## apiGroups: "","apps", "autoscaling", "batch", kubectl api-versions
  ## resources: "services", "pods","deployments"... kubectl api-resources
  ## verbs: "get", "list", "watch", "create", "update", "patch", "delete", "exec"
  ```
  
- **ClusterRole：一个ClusterRole能够授予和Role一样的权限，但是它是集群范围内的。** 

  ```yaml
  ## 定义一个集群角色，名为secret-reader，该角色可以读取所有的namespace中的secret资源
  kind: ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    # "namespace" omitted since ClusterRoles are not namespaced
    name: secret-reader
  rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "watch", "list"]
  ```
  
- **Rolebinding**

  将role中定义的权限分配给用户和用户组。RoleBinding包含主题（users,groups,或service accounts）和授予角色的引用。对于namespace内的授权使用RoleBinding，集群范围内使用ClusterRoleBinding。

  ```yaml
  ## 定义一个角色绑定，将pod-reader这个role的权限授予给jane这个User，使得jane可以在读取default这个命名空间下的所有的pod数据
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: read-pods
    namespace: default
  subjects:
  - kind: User   #这里可以是User,Group,ServiceAccount
    name: jane 
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role #这里可以是Role或者ClusterRole,若是ClusterRole，则权限也仅限于rolebinding的内部
    name: pod-reader # match the name of the Role or ClusterRole you wish to bind to
    apiGroup: rbac.authorization.k8s.io
  ```

  *注意：rolebinding既可以绑定role，也可以绑定clusterrole，当绑定clusterrole的时候，subject的权限也会被限定于rolebinding定义的namespace内部，若想跨namespace，需要使用clusterrolebinding*

  ```yaml
  ## 定义一个角色绑定，将dave这个用户和secret-reader这个集群角色绑定，虽然secret-reader是集群角色，但是因为是使用rolebinding绑定的，因此dave的权限也会被限制在development这个命名空间内
  apiVersion: rbac.authorization.k8s.io/v1
  # This role binding allows "dave" to read secrets in the "development" namespace.
  # You need to already have a ClusterRole named "secret-reader".
  kind: RoleBinding
  metadata:
    name: read-secrets
    #
    # The namespace of the RoleBinding determines where the permissions are granted.
    # This only grants permissions within the "development" namespace.
    namespace: development
  subjects:
  - kind: User
    name: dave # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: dave # Name is case sensitive
    namespace: demo
  roleRef:
    kind: ClusterRole
    name: secret-reader
    apiGroup: rbac.authorization.k8s.io
  ```

  考虑一个场景：  如果集群中有多个namespace分配给不同的管理员，每个namespace的权限是一样的，就可以只定义一个clusterrole，然后通过rolebinding将不同的namespace绑定到管理员身上，否则就需要每个namespace定义一个Role，然后做一次rolebinding。

- **ClusterRolebingding**

  允许跨namespace进行授权

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  # This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
  kind: ClusterRoleBinding
  metadata:
    name: read-secrets-global
  subjects:
  - kind: Group
    name: manager # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: secret-reader
    apiGroup: rbac.authorization.k8s.io
  ```

  

![](images/rbac-2.jpg)

### kubelet 的认证授权

查看kubelet进程

```powershell
$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Wed 2020-04-01 02:34:13 CST; 1 day 14h ago
     Docs: https://kubernetes.io/docs/
 Main PID: 851 (kubelet)
    Tasks: 21
   Memory: 127.1M
   CGroup: /system.slice/kubelet.service
           └─851 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf
```

查看`/etc/kubernetes/kubelet.conf`，解析证书：

```powershell
$ echo xxxxx |base64 -d >kubelet.crt
$ openssl x509 -in kubelet.crt -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 9059794385454520113 (0x7dbadafe23185731)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Feb 10 07:33:39 2020 GMT
            Not After : Feb  9 07:33:40 2021 GMT
        Subject: O=system:nodes, CN=system:node:master-1
```

得到我们期望的内容：

```bash
Subject: O=system:nodes, CN=system:node:k8s-master
```

我们知道，k8s会把O作为Group来进行请求，因此如果有权限绑定给这个组，肯定在clusterrolebinding的定义中可以找得到。因此尝试去找一下绑定了system:nodes组的clusterrolebinding

```powershell
$ kubectl get clusterrolebinding|awk 'NR>1{print $1}'|xargs kubectl get clusterrolebinding -oyaml|grep -n10 system:nodes
98-  roleRef:
99-    apiGroup: rbac.authorization.k8s.io
100-    kind: ClusterRole
101-    name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
102-  subjects:
103-  - apiGroup: rbac.authorization.k8s.io
104-    kind: Group
105:    name: system:nodes
106-- apiVersion: rbac.authorization.k8s.io/v1
107-  kind: ClusterRoleBinding
108-  metadata:
109-    creationTimestamp: "2020-02-10T07:34:02Z"
110-    name: kubeadm:node-proxier
111-    resourceVersion: "213"
112-    selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/kubeadm%3Anode-proxier

$ kubectl describe clusterrole system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
Name:         system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources                                                      Non-Resource URLs  Resource Names  Verbs
  ---------                                                      -----------------  --------------  -----
  certificatesigningrequests.certificates.k8s.io/selfnodeclient  []                 []              [create]
```

 结局有点意外，除了`system:certificates.k8s.io:certificatesigningrequests:selfnodeclient`外，没有找到system相关的rolebindings，显然和我们的理解不一样。 尝试去找[资料](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)，发现了这么一段 :

| Default ClusterRole            | Default ClusterRoleBinding          | Description                                                  |
| :----------------------------- | :---------------------------------- | :----------------------------------------------------------- |
| system:kube-scheduler          | system:kube-scheduler user          | Allows access to the resources required by the [scheduler](https://kubernetes.io/docs/reference/generated/kube-scheduler/)component. |
| system:volume-scheduler        | system:kube-scheduler user          | Allows access to the volume resources required by the kube-scheduler component. |
| system:kube-controller-manager | system:kube-controller-manager user | Allows access to the resources required by the [controller manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) component. The permissions required by individual controllers are detailed in the [controller roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles). |
| system:node                    | None                                | Allows access to resources required by the kubelet, **including read access to all secrets, and write access to all pod status objects**. You should use the [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) instead of the `system:node` role, and allow granting API access to kubelets based on the Pods scheduled to run on them. The `system:node` role only exists for compatibility with Kubernetes clusters upgraded from versions prior to v1.8. |
| system:node-proxier            | system:kube-proxy user              | Allows access to the resources required by the [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)component. |

大致意思是说：之前会定义system:node这个角色，目的是为了kubelet可以访问到必要的资源，包括所有secret的读权限及更新pod状态的写权限。如果1.8版本后，是建议使用 [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) 来代替这个角色的。

我们目前使用1.16，查看一下授权策略：

```powershell
$ ps axu|grep apiserver
kube-apiserver --authorization-mode=Node,RBAC  --enable-admission-plugins=NodeRestriction
```

查看一下官网对Node authorizer的介绍：

*Node authorization is a special-purpose authorization mode that specifically authorizes API requests made by kubelets.*

*In future releases, the node authorizer may add or remove permissions to ensure kubelets have the minimal set of permissions required to operate correctly.*

*In order to be authorized by the Node authorizer, kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`*



### Service Account

前面说，认证可以通过证书，也可以通过使用ServiceAccount（服务账户）的方式来做认证。大多数时候，我们在基于k8s做二次开发时都是选择通过serviceaccount的方式。我们之前访问dashboard的时候，是如何做的？

```yaml
## 新建一个名为admin的serviceaccount，并且把名为cluster-admin的这个集群角色的权限授予新建的serviceaccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
---
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
```

我们查看一下：

```powershell
$ kubectl -n kubernetes-dashboard get sa admin -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-04-01T11:59:21Z"
  name: admin
  namespace: kubernetes-dashboard
  resourceVersion: "1988878"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/serviceaccounts/admin
  uid: 639ecc3e-74d9-11ea-a59b-000c29dfd73f
secrets:
- name: admin-token-lfsrf
```

注意到serviceaccount上默认绑定了一个名为admin-token-lfsrf的secret，我们查看一下secret

```powershell
$ kubectl -n kubernetes-dashboard describe secret admin-token-lfsrf
Name:         admin-token-lfsrf
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 639ecc3e-74d9-11ea-a59b-000c29dfd73f

Type:  kubernetes.io/service-account-token
Data
====
ca.crt:     1025 bytes
namespace:  4 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZW1vIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFkbWluLXRva2VuLWxmc3JmIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNjM5ZWNjM2UtNzRkOS0xMWVhLWE1OWItMDAwYzI5ZGZkNzNmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlbW86YWRtaW4ifQ.ffGCU4L5LxTsMx3NcNixpjT6nLBi-pmstb4I-W61nLOzNaMmYSEIwAaugKMzNR-2VwM14WbuG04dOeO67niJeP6n8-ALkl-vineoYCsUjrzJ09qpM3TNUPatHFqyjcqJ87h4VKZEqk2qCCmLxB6AGbEHpVFkoge40vHs56cIymFGZLe53JZkhu3pwYuS4jpXytV30Ad-HwmQDUu_Xqcifni6tDYPCfKz2CZlcOfwqHeGIHJjDGVBKqhEeo8PhStoofBU6Y4OjObP7HGuTY-Foo4QindNnpp0QU6vSb7kiOiQ4twpayybH8PTf73dtdFt46UF6mGjskWgevgolvmO8A
```

开发的时候如何去调用k8s的api:

1. curl演示

```powershell
$ curl -k  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZW1vIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFkbWluLXRva2VuLWxmc3JmIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNjM5ZWNjM2UtNzRkOS0xMWVhLWE1OWItMDAwYzI5ZGZkNzNmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlbW86YWRtaW4ifQ.ffGCU4L5LxTsMx3NcNixpjT6nLBi-pmstb4I-W61nLOzNaMmYSEIwAaugKMzNR-2VwM14WbuG04dOeO67niJeP6n8-ALkl-vineoYCsUjrzJ09qpM3TNUPatHFqyjcqJ87h4VKZEqk2qCCmLxB6AGbEHpVFkoge40vHs56cIymFGZLe53JZkhu3pwYuS4jpXytV30Ad-HwmQDUu_Xqcifni6tDYPCfKz2CZlcOfwqHeGIHJjDGVBKqhEeo8PhStoofBU6Y4OjObP7HGuTY-Foo4QindNnpp0QU6vSb7kiOiQ4twpayybH8PTf73dtdFt46UF6mGjskWgevgolvmO8A" https://62.234.214.206:6443/api/v1/namespaces/demo/pods?limit=500
```

2. postman

### 查看etcd数据

拷贝etcdctl命令行工具：

```powershell
$ docker exec -ti  etcd_container which etcdctl
$ docker cp etcd_container:/usr/local/bin/etcdctl /usr/bin/etcdctl
```

查看所有key值：

```powershell
$  ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get / --prefix --keys-only
```

查看具体的key对应的数据：

```powershell
$ ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get /registry/pods/jenkins/sonar-postgres-7fc5d748b6-gtmsb
```



## 七、helm包管理器

### 介绍

Helm是一个基于Kubernetes的程序包（资源包）管理器，它将一个应用的相关资源组织成为Charts，并通过Charts管理程序包，其使用优势可简单总结为如下几个方面：

- 管理复杂应用：Charts能够描述哪怕是最复杂的程序结构，其提供了可重复使用的应用安装的定义。
- 易于升级：使用就地升级和自定义钩子来解决更新的难题。
- 简单分享：Charts易于通过公共或私有服务完成版本化、分享及主机构建。
- 回滚：可使用“helm rollback”命令轻松实现快速回滚。

### Helm关键概念

- **`Helm`——Helm** 是一个命令行下的**客户端工具**。主要用于 Kubernetes 应用程序 Chart 的创建、打包、发布以及创建和管理本地和远程的 Chart 仓库。
- **`Chart`——Chart** 代表着 **Helm 包**。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是 [Homebrew](https://so.csdn.net/so/search?q=Homebrew&spm=1001.2101.3001.7020) formula，Apt dpkg，或 Yum RPM 在 Kubernetes 中的等价物。
- **`Release`——Release** 是运行在 Kubernetes 集群中的 **chart 的实例**。一个 chart 通常可以在同一个集群中安装多次。每一次安装都会创建一个新的 release。
- `Config`：应用程序实例化安装运行时使用的配置信息。
- **`Repoistory`——Repository（仓库）** 是用来存放和共享 charts 的地方。它就像 Perl 的 CPAN 档案库网络 或是 Fedora 的 软件包仓库，只不过它是供 Kubernetes 包所使用的。

### 架构

**1.Helm：**命令行客户端工具，采用Go语言编写，基于gRPC协议与Tillerserver交互。它主要完成如下任务：

- 本地Charts开发。

- 管理Charts仓库。

- 与Tiller服务器交互：发送Charts以安装、查询Release的相关信息以及升级或卸载已有的Release。

**2.Tiller**
server（V3已将Tiller的删除，通过ApiServer与k8s交互）是托管运行于Kubernetes集群之中的容器化服务应用，它接收来自Helm客户端的请求，并在必要时与Kubernetes API Server进行交互。它主要完成以下任务。

- 监听来自于Helm客户端的请求。

- 合并Charts和配置以构建一个Release。

- 向Kubernetes集群安装Charts并对相应的Release进行跟踪。

- 升级和卸载Charts。

通常，用户于Helm客户端本地遵循其格式编写Charts文件，而后即可部署于Kuber-netes集群之上运行为一个特定的Release。仅在有分发需求时，才应该将同一应用的Charts文件打包成归档压缩格式提交到特定的Charts仓库。

### 工作原理

1. Chart Install 过程：

   - Helm从指定的目录或者tgz文件中解析出Chart结构信息

   - Helm将指定的Chart结构和Values信息通过gRPC传递给Tiller

   - Tiller根据Chart和Values生成一个Release

   - Tiller将Release发送给Kubernetes用于生成Release

2. Chart Update过程：

   - Helm从指定的目录或者tgz文件中解析出Chart结构信息

   - Helm将要更新的Release的名称和Chart结构，Values信息传递给Tiller

   - Tiller生成Release并更新指定名称的Release的History

   - Tiller将Release发送给Kubernetes用于更新Release

3. Chart Rollback过程：

   - Helm将要回滚的Release的名称传递给Tiller

   - Tiller根据Release的名称查找History

   - Tiller从History中获取上一个Release

   - Tiller将上一个Release发送给Kubernetes用于替换当前Release

### 安装

二进制安装

```bash
# 1、下载所需版本（系统为centos7.9，以helm-v3.2.4为例）：
curl -L -o helm-v3.2.4-linux-amd64.tar.gz https://file.choerodon.com.cn/kubernetes-helm/v3.2.4/helm-v3.2.4-linux-amd64.tar.gz

# 2、解压缩：
tar -zxvf helm-v3.2.4-linux-amd64.tar.gz

# 3、将文件移动到PATH目录中（以linux-amd64为例）：
mv linux-amd64/helm /usr/local/bin/helm
```

添加库

```bash
# 语法格式：
helm repo add repo-name(自定义) repo-url（仓库地址）

# 添加微软的helm仓库：
helm repo add stable http://mirror.azure.cn/kubernetes/charts 

# 添加阿里的helm仓库:
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 

```

相关命令

```bash
# 列出、增加、更新、删除 chart 仓库
helm repo

#更新存储库地址：
helm repo update 

# 查看配置的存储库：
helm repo list
helm search repo stable
helm search repo aliyun

# 删除存储库： 
helm repo remove aliyun

# 使用命令搜索应用
helm search repo redis

# 拉取远程仓库中的 chart 到本地 
helm pull

# 在本地创建新的 chart 
helm create 

# 根据搜索内容选择安装
helm install 

# 列出所有 release 
helm list 

# 检查 chart 配置是否有误 
helm lint 

# 打包本地 chart 
helm package 

# 回滚 release 到历史版本 
helm rollback 

# 卸载 release 
helm uninstall 

# 升级 release 
helm upgrade 
```

chart 目录结构：

```bash
# 通过helm create命令创建一个新的chart包
helm create nginx

tree nginx
nginx/
├── charts  #依赖其他包的charts文件
├── Chart.yaml # 该chart的描述文件,包括ico地址,版本信息等
├── templates  # #存放k8s模板文件目录
│   ├── deployment.yaml # 创建k8s资源的yaml 模板
│   ├── _helpers.tpl # 下划线开头的文件,可以被其他模板引用
│   ├── hpa.yaml # 弹性扩缩容，配置服务资源CPU 内存
│   ├── ingress.yaml # ingress 配合service域名访问的配置
│   ├── NOTES.txt # 说明文件,helm install之后展示给用户看的内容
│   ├── serviceaccount.yaml # 服务账号配置
│   ├── service.yaml # kubernetes Serivce yaml 模板
│   └── tests # 测试模块
│       └── test-connection.yaml
└── values.yaml # 给模板文件使用的变量
```

chart.yaml

```yaml
apiVersion: chart API 版本 （必需）
name: chart名称 （必需）
version: chart 版本，语义化2 版本（必需）
kubeVersion: 兼容Kubernetes版本的语义化版本（可选）
description: 一句话对这个项目的描述（可选）
type: chart类型 （可选）
keywords:
  - 关于项目的一组关键字（可选）
home: 项目home页面的URL （可选）
sources:
  - 项目源码的URL列表（可选）
dependencies: # chart 必要条件列表 （可选）
  - name: chart名称 (nginx)
    version: chart版本 ("1.2.3")
    repository: （可选）仓库URL ("https://example.com/charts") 或别名 ("@repo-name")
    condition: （可选） 解析为布尔值的yaml路径，用于启用/禁用chart (e.g. subchart1.enabled )
    tags: # （可选）
      - 用于一次启用/禁用 一组chart的tag
    import-values: # （可选）
      - ImportValue 保存源值到导入父键的映射。每项可以是字符串或者一对子/父列表项
    alias: （可选） chart中使用的别名。当你要多次添加相同的chart时会很有用
maintainers: # （可选）
  - name: 维护者名字 （每个维护者都需要）
    email: 维护者邮箱 （每个维护者可选）
    url: 维护者URL （每个维护者可选）
icon: 用做icon的SVG或PNG图片URL （可选）
appVersion: 包含的应用版本（可选）。不需要是语义化，建议使用引号
deprecated: 不被推荐的chart （可选，布尔值）
annotations:
  example: 按名称输入的批注列表 （可选）.
  
  
  
#从 v3.3.2，不再允许额外的字段。推荐的方法是在 annotations 中添加自定义元数据。
#每个 chart 都必须有个版本号（version）。版本必须遵循 语义化版本 2 标准。不像经典 Helm， Helm v2 以及后续版本会使用版本号作为发布标记。仓库中的包通过名称加版本号标识。
```

### 自定义Chart

- 执行命令helm create myapp，会创建一个myapp目录

```bash
helm create myapp
```

- 查看myapp目录结构

```bash
$ tree myapp
myapp
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

- 修改配置文件

  - 编辑自描述文件 Chart.yaml , 修改version和appVersion信息

  ```bash
  apiVersion: v2
  name: myapp
  description: A Helm chart for Kubernetes
  
  # A chart can be either an 'application' or a 'library' chart.
  #
  # Application charts are a collection of templates that can be packaged into versioned archives
  # to be deployed.
  #
  # Library charts provide useful utilities or functions for the chart developer. They're included as
  # a dependency of application charts to inject those utilities and functions into the rendering
  # pipeline. Library charts do not define any templates and therefore cannot be deployed.
  type: application
  
  # This is the chart version. This version number should be incremented each time you make changes
  # to the chart and its templates, including the app version.
  # Versions are expected to follow Semantic Versioning (https://semver.org/)
  version: 1.0.0 # 项目版本号
  
  # This is the version number of the application being deployed. This version number should be
  # incremented each time you make changes to the application. Versions are not expected to
  # follow Semantic Versioning. They should reflect the version the application is using.
  # It is recommended to use it with quotes.
  appVersion: "v1" # 镜像版本
  ```

  - 编辑values.yaml配置文件

  <img src="images/image-20240423221451495.png" alt="image-20240423221451495" style="zoom:67%;" />

- 打包安装Chart

  - 检查chart语法正确性

  ```bash
  helm lint myapp
  ```

  - 打包自定义的chart

  ```bash
  helm package myapp
  ```

  - 安装chart

  ```bash
  helm install myapp myapp-1.tgz
  ```

  - 验证

  <img src="images/image-20240423221820515.png" alt="image-20240423221820515" style="zoom:67%;" />

- 更新

  - 编辑自描述文件 Chart.yaml , 修改version和appVersion信息

  <img src="images/image-20240423221942514.png" alt="image-20240423221942514" style="zoom:67%;" />

  - 重新打包charts

  ```bash
  # 检查chart语法正确性
  helm lint myapp
  # 打包自定义的chart
  helm package myapp
  ```

  - 更新chart

  ```bash
   helm upgrade myapp myapp-2.tgz
  ```

  <img src="images/image-20240423222112552.png" alt="image-20240423222112552" style="zoom:80%;" />

- 回滚

  - 查看当前版本信息

  ```bash
  helm list
  ```

  - 查看历史版本信息

  ```bash
  helm history myapp
  ```

  - 回滚到指定版本

  ```bash
  # 回退到指定的1版本
  hlem rollback myapp 1
  ```

  - 再次验证

### helm导出yaml文件

存在需要导出yaml分析yaml编写情况，而不是直接部署到k8s，这个时候，就需要使用template来实现了

- 拉取charts包

```bash
$ helm pull nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --untar
$ ls
nfs-subdir-external-provisioner 
```

- 创建存放yaml文件的目录并渲染导出

```bash
mkdir -p /opt/k8s/nfs 
cd /opt/k8s/
# 导出
helm template nfs-subdir-external-provisioner --output-dir /opt/k8s/nfs-yaml/
```

```bash
$ cd nfs-yaml/nfs-subdir-external-provisioner/templates/
$ ls
clusterrolebinding.yaml  clusterrole.yaml  deployment.yaml  rolebinding.yaml  role.yaml  serviceaccount.yaml  storageclass.yaml
```





## 八、遇到的问题及处理

**1、kube-proxy报错：**

```bash
...
E0703 13:49:39.609152    1393 proxier.go:1950] Failed to list IPVS destinations, error: parseIP Error ip=[192 168 50 65 0 0 0 0 0 0 0 0 0 0 0 0]
E0703 13:49:39.609230    1393 proxier.go:1192] Failed to sync endpoint for service: 10.244.0.1:443/TCP, err: parseIP Error ip=[192 168 50 65 0 0 0 0 0 0 0 0 0 0 0 0]
E0703 13:49:39.609564    1393 proxier.go:1950] Failed to list IPVS destinations, error: parseIP Error ip=[172 30 1 75 0 0 0 0 0 0 0 0 0 0 0 0]
E0703 13:49:39.609621    1393 proxier.go:1192] Failed to sync endpoint for service: 10.244.75.157:80/TCP, err: parseIP Error ip=[172 30 1 75 0 0 0 0 0 0 0 0 0 0 0 0]
...
```

当前我所使用的linux版本为centos7.6，内核版本3.10，k8s版本：1.18.5

剧官方表示，因为k8s 1.18.x版本以上使用的是比较高版本的ipvs，当前的内核版本不兼容，需要升级到较高版本的内核

升级流程：

```bash
# 载入公钥
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

# 安装 ELRepo 最新版本
$ yum install -y https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm

# 列出可以使用的 kernel 包版本
$ yum list available --disablerepo=* --enablerepo=elrepo-kernel

# 安装指定的 kernel 版本：
$ yum install -y kernel-lt-4.4.222-1.el7.elrepo --enablerepo=elrepo-kernel

# 查看系统可用内核
$ cat /boot/grub2/grub.cfg | grep menuentry

menuentry 'CentOS Linux (3.10.0-1062.el7.x86_64) 7 (Core)' --class centos （略）
menuentry 'CentOS Linux (4.4.222-1.el7.elrepo.x86_64) 7 (Core)' --class centos ...（略）

# 设置开机从新内核启动
$ grub2-set-default "CentOS Linux (4.4.222-1.el7.elrepo.x86_64) 7 (Core)"

# 查看内核启动项
$ grub2-editenv list
saved_entry=CentOS Linux (4.4.222-1.el7.elrepo.x86_64) 7 (Core)
```

重启系统使内核生效，启动完成查看内核版本是否更新：

```bash
uname -r
```







