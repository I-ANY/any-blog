+++
title = "一些k8s 面试题"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 2
+++

# 1. 简述etcd及其特点？

etcd 是 CoreOS 团队发起的开源项目，是一个管理配置信息和服务发现（service discovery）的项目，它的目标是构建一个高可用的分布式键值（key-value）数据库，基于 Go 语言实现。
特点：

-  简单：支持 REST 风格的 HTTP+JSON API
- 安全：支持 HTTPS 方式的访问
- 快速：支持并发 1k/s 的写操作
- 可靠：支持分布式结构，基于 Raft 的一致性算法，Raft 是一套通过选举主节点来实现分布式系统一致性的算法

# 2. 简述etcd适应的场景？

etcd基于其优秀的特点，可广泛的应用于以下场景：

- 服务发现(Service Discovery)：服务发现主要解决在同一个分布式集群中的进程或服务，要如何才能找到对方并建立连接。本质上来说，服务发现就是想要了解集群中是否有进程在监听udp或tcp端口，并且通过名字就可以查找和连接。
- 消息发布与订阅：在分布式系统中，最适用的一种组件间通信方式就是消息发布与订阅。即构建一个配置共享中心，数据提供者在这个配置中心发布消息，而消息使用者则订阅他们关心的主题，一旦主题有消息发布，就会实时通知订阅者。通过这种方式可以做到分布式系统配置的集中式管理与动态更新。应用中用到的一些配置信息放到etcd上进行集中管理。
- 负载均衡：在分布式系统中，为了保证服务的高可用以及数据的一致性，通常都会把数据和服务部署多份，以此达到对等服务，即使其中的某一个服务失效了，也不影响使用。etcd本身分布式架构存储的信息访问支持负载均衡。etcd集群化以后，每个etcd的核心节点都可以处理用户的请求。所以，把数据量小但是访问频繁的消息数据直接存储到etcd中也可以实现负载均衡的效果。
-  分布式通知与协调：与消息发布和订阅类似，都用到了etcd 中的Watcher 机制，通过注册与异步通知机制，实现分布式环境下不同系统之间的通知与协调，从而对数据变更做到实时处理。
- 分布式锁：因为etcd使用Raft算法保持了数据的强一致性，某次操作存储到集群中的值必然是全局一致的，所以很容易实现分布式锁。锁服务有两种使用方式，一是保持独占，二是控制时序。
- 集群监控与Leader 竞选：通过etcd 来进行监控实现起来非常简单并且实时性强。

# 3. 简述什么是k8s？

k8s是一个全新的基于容器技术的分布式系统支撑平台。是Google开源的容器集群管理系统（谷歌内部:Borg）。在Docker技术的基础上，为容器化的应用提供部署运行. 资源调度. 服务发现和动态伸缩等一系列完整功能，提高了大规模容器集群管理的便捷性。并且具有完备的集群管理能力，多层次的安全防护和准入机制. 多租户应用支撑能力. 透明的服务注册和发现机制. 內建智能负载均衡器. 强大的故障发现和自我修复能力. 服务滚动升级和在线扩容能力. 可扩展的资源自动调度机制以及多粒度的资源配额管理能力。



# 4. 简述k8s和Docker的关系？

Docker 提供容器的生命周期管理和，Docker 镜像构建运行时容器。它的主要优点是将将软件/应用程序运行所需的设置和依赖项打包到一个容器中，从而实现了可移植性等优点。
k8s 用于关联和编排在多个主机上运行的容器。



# 5. 简述k8s中什么是Minikube. Kubectl. Kubelet？

- Minikube 是一种可以在本地轻松运行一个单节点 k8s 群集的工具。
- Kubectl 是一个命令行工具，可以使用该工具控制k8s集群管理器，如检查群集资源，创建. 删除和更新组件，查看应用程序。
- Kubelet 是一个代理服务，它在每个节点上运行，并使从服务器与主服务器通信。



# 6. 简述k8s常见的部署方式？

常见的k8s部署方式有：

- kubeadm：也是推荐的一种部署方式；
- 二进制：较深入的部署方式；
- minikube：在本地轻松运行一个单节点 k8s 群集的工具；
- kubekey：由青云开源的，第三方式部署方式，封闭kubeadm;
- kubease：使用ansible部署方式；
- sealos：极速部署k8s集群方式，封装kubeadm



# 7. 简述k8s如何实现集群管理？

在集群管理方面，k8s将集群中的机器划分为一个Master节点和一群工作节点Node。

其中，在Master节点运行着集群管理相关的一组进程kubeapiserver. kube-controller-manager和kube-scheduler，这些进程实现了整个集群的资源管理. Pod调度. 弹性伸缩. 安全控制. 系统监控和纠错等管理能力，并且都是全自动完成的。



# 8. 简述k8s的优势. 适应场景及其特点？
k8s作为一个完备的分布式系统支撑平台，其主要优势：

- 容器编排

-  轻量级

-  开源

- 弹性伸缩

- 负载均衡

  

k8s常见场景：

- 快速部署应用

-  快速扩展应用

- 无缝对接新的应用功能

- 节省资源，优化硬件资源的使用

  

k8s相关特点：

-  可移植: 支持公有云. 私有云. 混合云. 多重云（multi-cloud）。
- 可扩展: 模块化,. 插件化. 可挂载. 可组合。
- 自动化: 自动部署. 自动重启. 自动复制. 自动伸缩/扩展。



# 9. 简述k8s的缺点或当前的不足之处？

k8s当前存在的缺点（不足）如下：

- 安装过程和配置相对困难复杂。
- 管理服务相对繁琐。
- 运行和编译需要很多时间。
- 它比其他替代品更昂贵。
- 对于简单的应用程序来说，可能不需要涉及k8s即可满足。



# 10. 简述k8s相关基础概念？

- master：k8s集群的管理节点，负责管理集群，提供集群的资源数据访问入口。拥有Etcd存储服务（可选），运行Api Server进程，Controller Manager服务进程及Scheduler服务进程。
- node（worker）：Node（worker）是k8s集群架构中运行Pod的服务节点，是k8s集群操作的单元，用来承载被分配Pod的运行，是Pod
  运行的宿主机。运行docker eninge服务，守护进程kunelet及负载均衡器kube-proxy。
-  pod：运行于Node节点上，若干相关容器的组合。Pod内包含的容器运行在同一宿主机上，使用相同的网络命名空间. IP地址和端口，能够通过localhost进
  行通信。Pod是Kurbernetes进行创建. 调度和管理的最小单位，它提供了比容器更高层次的抽象，使得部署和管理更加灵活。一个Pod可以包含一个容器或者多个相关容器。
-  label：k8s中的Label实质是一系列的Key/Value键值对，其中key与value可自定义。Label可以附加到各种资源对象上，如Node. Pod. Service. 
  RC等。一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上去。k8s通过Label Selector（标签选择器）查询
  和筛选资源对象。
- Replication Controller：Replication Controller用来管理Pod的副本，保证集群中存在指定数量的Pod副本。集群中副本的数量大于指定数量，则会停止指定
  数量之外的多余容器数量。反之，则会启动少于指定数量个数的容器，保证数量不变。Replication Controller是实现弹性伸缩. 动态扩容和滚动升级的核心。
-  Deployment：Deployment在内部使用了RS来实现目的，Deployment相当于RC的一次升级，其最大的特色为可以随时获知当前Pod的部署进度。
-  HPA（Horizontal Pod Autoscaler）：Pod的横向自动扩容，也是k8s的一种资源，通过追踪分析RC控制的所有Pod目标的负载变化情况，来确定是
  否需要针对性的调整Pod副本数量。
-  Service：Service定义了Pod的逻辑集合和访问该集合的策略，是真实服务的抽象。Service提供了一个统一的服务访问入口以及服务代理和发现机制，关联多个相同Label的Pod，用户不需要了解后台Pod是如何运行。
- Volume：Volume是Pod中能够被多个容器访问的共享目录，k8s中的Volume是定义在Pod上，可以被一个或多个Pod中的容器挂载到某个目录下。
-  Namespace：Namespace用于实现多租户的资源隔离，可将集群内部的资源对象分配到不同的Namespace中，形成逻辑上的不同项目. 小组或用户组，便于不同的Namespace在共享使用整个集群的资源的同时还能被分别管理。



# 11. 简述k8s集群相关组件？

k8s Master控制组件，调度管理整个系统（集群），包含如下组件:

- k8s API Server：作为k8s系统的入口，其封装了核心对象的增删改查操作，以RESTful API接口方式提供给外部客户和内部组件调用，集群内
  各个功能模块之间数据交互和通信的中心枢纽。
-  k8s Scheduler：为新建立的Pod进行节点(node)选择(即分配机器)，负责集群的资源调度。
-  k8s Controller：负责执行各种控制器，目前已经提供了很多控制器来保证k8s的正常运行。
-  Replication Controller：管理维护Replication Controller，关联ReplicationController和Pod，保证Replication Controller定义的副本数量与实际运行
  Pod数量一致。
-  Node Controller：管理维护Node，定期检查Node的健康状态，标识出(失效|未失效)的Node节点。
- Namespace Controller：管理维护Namespace，定期清理无效的Namespace，包括Namesapce下的API对象，比如Pod. Service等。
-  Service Controller：管理维护Service，提供负载以及服务代理。
-  EndPoints Controller：管理维护Endpoints，关联Service和Pod，创建Endpoints为Service的后端，当Pod发生变化时，实时更新Endpoints。
-  Service Account Controller：管理维护Service Account，为每个Namespace创建默认的Service Account，同时为Service Account创建Service Account
  Secret。
-  Persistent Volume Controller：管理维护Persistent Volume和Persistent Volume Claim，为新的Persistent Volume Claim分配Persistent Volume进
  行绑定，为释放的Persistent Volume执行清理回收。
- Daemon Set Controller：管理维护Daemon Set，负责创建Daemon Pod，保证指定的Node上正常的运行Daemon Pod。
- Deployment Controller：管理维护Deployment，关联Deployment和Replication Controller，保证运行指定数量的Pod。当Deployment更新时，
  控制实现Replication Controller和Pod的更新。
-  Job Controller：管理维护Job，为Jod创建一次性任务Pod，保证完成Job指定完成的任务数目
-  Pod Autoscaler Controller：实现Pod的自动伸缩，定时获取监控数据，进行策略匹配，当满足条件时执行Pod的伸缩动作。



# 12. 简述k8s RC的机制？

Replication Controller用来管理Pod的副本，保证集群中存在指定数量的Pod副本。当定义了RC并提交至k8s集群中之后，Master节点上的Controller
Manager组件获悉，并同时巡检系统中当前存活的目标Pod，并确保目标Pod实例的数量刚好等于此RC的期望值，若存在过多的Pod副本在运行，系统会停止一些
Pod，反之则自动创建一些Pod。



# 13. 简述k8s Replica Set 和 Replication Controller 之间有什么区别？

Replica Set 和 Replication Controller 类似，都是确保在任何给定时间运行指定数量的 Pod 副本。不同之处在于RS 使用基于集合的选择器，而 Replication
Controller 使用基于权限的选择器。



# 14. 简述kube-proxy作用？

kube-proxy 运行在所有节点上，它监听 apiserver 中 service 和 endpoint 的变化情况，创建路由规则以提供服务 IP 和负载均衡功能。简单理解此进程是Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上。



# 15. 简述kube-proxy iptables原理？

k8s从1.2版本开始，将iptables作为kube-proxy的默认模式。iptables模式下的kube-proxy不再起到Proxy的作用，其核心功能：通过API
Server的Watch接口实时跟踪Service与Endpoint的变更信息，并更新对应的iptables规则，Client的请求流量则通过iptables的NAT机制“直接路由”到目标
Pod。



# 16. 简述kube-proxy ipvs原理？

IPVS 在k8s1.11 中升级为GA 稳定版。IPVS 则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张，因此被kubeproxy采纳为最新模式。
在IPVS模式下，使用iptables的扩展ipset，而不是直接调用iptables来生成规则链。iptables规则链是一个线性的数据结构，ipset则引入了带索引的数据结构，因此当规则很多时，也可以很高效地查找和匹配。
可以将ipset简单理解为一个IP（段）的集合，这个集合的内容可以是IP地址. IP网段. 端口等，iptables可以直接添加规则对这个“可变的集合”进行操作，这样做的好处在于可以大大减少iptables规则的数量，从而减少性能损耗。



# 17. 简述kube-proxy ipvs和iptables的异同？

iptables与IPVS都是基于Netfilter 实现的，但因为定位不同，二者有着本质的差别：iptables是为防火墙而设计的；IPVS则专门用于高性能负载均衡，并使用更高
效的数据结构（Hash表），允许几乎无限的规模扩张。
与iptables相比，IPVS拥有以下明显优势：

- 1. 为大型集群提供了更好的可扩展性和性能；
-  2. 支持比iptables更复杂的复制均衡算法（最小负载. 最少连接. 加权等）；
-  3. 支持服务器健康检查和连接重试等功能；
- 4. 可以动态修改ipset的集合，即使iptables的规则正在使用这个集合。



# 18. 简述k8s中什么是静态Pod？

静态pod是由kubelet进行管理的仅存在于特定Node的Pod上，他们不能通过API Server进行管理，无法与ReplicationController. Deployment或者
DaemonSet进行关联，并且kubelet无法对他们进行健康检查。静态Pod总是由kubelet进行创建，并且总是在kubelet所在的Node上运行。



# 19. 简述k8s中Pod可能位于的状态？

-  Pending：API Server已经创建该Pod，且Pod内还有一个或多个容器的镜像没有创建，包括正在下载镜像的过程。
- Running：Pod内所有容器均已创建，且至少有一个容器处于运行状态. 正在启动状态或正在重启状态。
- Succeeded：Pod内所有容器均成功执行退出，且不会重启。
- Failed：Pod内所有容器均已退出，但至少有一个容器退出为失败状态。
-  Unknown：由于某种原因无法获取该Pod状态，可能由于网络通信不畅导致。



# 20. 简述k8s创建一个Pod的主要流程？

k8s中创建一个Pod涉及多个组件之间联动，主要流程如下：

- 1. 客户端提交Pod的配置信息（可以是yaml文件定义的信息）到kubeapiserver。
-  2. Apiserver收到指令后，通知给controller-manager创建一个资源对象。
-  3. Controller-manager通过api-server将pod的配置信息存储到ETCD数据中心中。
-  4. Kube-scheduler检测到pod信息会开始调度预选，会先过滤掉不符合Pod资源配置要求的节点，然后开始调度调优，主要是挑选出更适合运行pod 的节点，然后将pod的资源配置单发送到node节点上的kubelet组件上。
-  5. Kubelet根据scheduler发来的资源配置单运行pod，运行成功后，将pod的运行信息返回给scheduler，scheduler将返回的pod运行状况的信息存储到
  etcd数据中心。



# 21. 简述k8s中Pod的重启策略？

Pod重启策略（RestartPolicy）应用于Pod内的所有容器，并且仅在Pod所处的Node上由kubelet进行判断和重启操作。当某个容器异常退出或者健康检查失败
时，kubelet将根据RestartPolicy的设置来进行相应操作。

Pod的重启策略包括Always. OnFailure和Never，默认值为Always。

- Always：当容器失效时，由kubelet自动重启该容器；
-  OnFailure：当容器终止运行且退出码不为0时，由kubelet自动重启该容器；
-  Never：不论容器运行状态如何，kubelet都不会重启该容器。

同时Pod的重启策略与控制方式关联，当前可用于管理Pod的控制器包括ReplicationController. Job. DaemonSet 及直接管理kubelet 管理（静态Pod）。
不同控制器的重启策略限制如下：

- RC和DaemonSet：必须设置为Always，需要保证该容器持续运行；
- Job：OnFailure或Never，确保容器执行完成后不再重启；
- kubelet：在Pod失效时重启，不论将RestartPolicy设置为何值，也不会对Pod进行健康检查。

# 22. 简述k8s中Pod的健康检查方式？

对Pod的健康检查可以通过两类探针来检查：LivenessProbe和ReadinessProbe。

- LivenessProbe探针：用于判断容器是否存活（running状态），如果LivenessProbe探针探测到容器不健康，则kubelet将杀掉该容器，并根据容器
  的重启策略做相应处理。若一个容器不包含LivenessProbe探针，kubelet认为该容器的LivenessProbe探针返回值用于是“Success”。
- ReadineeProbe探针：用于判断容器是否启动完成（ready状态）。如果ReadinessProbe探针探测到失败，则Pod的状态将被修改。Endpoint Controller将从Service的Endpoint中删除包含该容器所在Pod的Eenpoint。
- startupProbe探针：启动检查机制，应用一些启动缓慢的业务，避免业务长时间启动而被上面两类探针kill掉，配置了 statupProbe 后，只有在应用启动成功了，才会执行另外两种探针，可以更加方便的结合使用另外两种探针使用。。

# 23. 简述k8s Pod的LivenessProbe探针的常见方式？

kubelet定期执行LivenessProbe探针来诊断容器的健康状态，通常有以下三种方式：

-  ExecAction：在容器内执行一个命令，若返回码为0，则表明容器健康。
-  TCPSocketAction：通过容器的IP地址和端口号执行TCP检查，若能建立TCP连接，则表明容器健康。
- HTTPGetAction：通过容器的IP地址. 端口号及路径调用HTTP Get方法，若响应的状态码大于等于200且小于400，则表明容器健康。

# 24. 简述k8s Pod的常见调度方式？

k8s中，Pod通常是容器的载体，主要有如下常见调度方式：

- Deployment 或RC：该调度策略主要功能就是自动部署一个容器应用的多份副本，以及持续监控副本的数量，在集群内始终维持用户指定的副本数量。
- NodeSelector：定向调度，当需要手动指定将Pod调度到特定Node上，可以通过Node的标签（Label）和Pod的nodeSelector 属性相匹配。
-  NodeAffinity 亲和性调度：亲和性调度机制极大的扩展了Pod的调度能力，目前有两种节点亲和力表达：
-  requiredDuringSchedulingIgnoredDuringExecution：硬规则，必须满足指定的规则，调度器才可以调度Pod 至Node 上（类似nodeSelector，语法不同）。
- preferredDuringSchedulingIgnoredDuringExecution：软规则，优先调度至满足的Node的节点，但不强求，多个优先级规则还可以设置权重值。
- Taints和Tolerations（污点和容忍）：
  - Taint：使Node拒绝特定Pod运行；
  - Toleration：为Pod的属性，表示Pod能容忍（运行）标注了Taint的Node。



# 25. 简述k8s初始化容器（init container）？

init container的运行方式与应用容器不同，它们必须先于应用容器执行完成，当设置了多个init container时，将按顺序逐个运行，并且只有前一个init container运行成功后才能运行后一个init container。当所有init container都成功运行后，k8s才会初始化Pod的各种信息，并开始创建和运行应用容器。



# 26. 简述k8s deployment升级过程？

- 初始创建Deployment时，系统创建了一个ReplicaSet，并按用户的需求创建了对应数量的Pod副本。
-  当更新Deployment时，系统创建了一个新的ReplicaSet，并将其副本数量扩展到1，然后将旧ReplicaSet缩减为2。
-  之后，系统继续按照相同的更新策略对新旧两个ReplicaSet进行逐个调整。
-  最后，新的ReplicaSet运行了对应个新版本Pod副本，旧的ReplicaSet副本数量则缩减为0。



# 27. 简述k8s deployment升级策略？
在Deployment的定义中，可以通过spec.strategy指定Pod更新的策略，目前支持
两种策略：Recreate（重建）和RollingUpdate（滚动更新），默认值为RollingUpdate。

-  Recreate：设置spec.strategy.type=Recreate，表示Deployment在更新Pod时，会先杀掉所有正在运行的Pod，然后创建新的Pod。
- RollingUpdate：设置spec.strategy.type=RollingUpdate，表示Deployment会以滚动更新的方式来逐个更新Pod。同时，可以通过设置
  spec.strategy.rollingUpdate下的两个参数（maxUnavailable和maxSurge）来控制滚动更新的过程。



# 28. 简述k8s DaemonSet类型的资源特性？

DaemonSet资源对象会在每个k8s集群中的节点上运行，并且每个节只能运行一个pod，这是它和deployment 资源对象的最大也是唯一的区别。因此，
在定义yaml文件中，不支持定义replicas。
它的一般使用场景如下：

- 在去做每个节点的日志收集工作。
-  监控每个节点的的运行状态。



# 29. 简述k8s自动扩容机制？



k8s使用Horizontal Pod Autoscaler（HPA）的控制器实现基于CPU使用率进行自动Pod扩缩容的功能。HPA控制器周期性地监测目标Pod的资源性能
指标，并与HPA资源对象中的扩缩容条件进行对比，在满足条件时对Pod副本数量进行调整。

**HPA原理**
k8s中的某个Metrics Server（Heapster或自定义Metrics Server）持续采集所有Pod副本的指标数据。HPA控制器通过Metrics Server的API（Heapster的API或聚合API）获取这些数据，基于用户定义的扩缩容规则进行计算，得到目标Pod副本数量。当目标Pod副本数量与当前副本数量不同时，HPA控制器就向Pod的副本控制器（Deployment. RC或ReplicaSet）发起scale操作，调整Pod的副本数量，完成扩缩容操作。



# 30. 简述k8s Service类型？

通过创建Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求负载分发到后端的各个容器应用上。其主要类型有：

- ClusterIP：虚拟的服务IP 地址，该地址用于k8s 集群内部的Pod 访问，在Node上kube-proxy通过设置的iptables规则进行转发；
- NodePort：使用宿主机的端口，使能够访问各Node的外部客户端通过Node的IP地址和端口号就能访问服务；
-  LoadBalancer：使用外接负载均衡器完成到服务的负载分发，需要在spec.status.loadBalancer 字段指定外部负载均衡器的IP 地址，通常用于公有云。
- external：把k8s集群外的服务引入到k8s集群内



# 31. 简述k8s Service分发后端的策略？

Service负载分发的策略有：RoundRobin和SessionAffinity

- RoundRobin：默认为轮询模式，即轮询将请求转发到后端的各个Pod上。
- SessionAffinity：基于客户端IP地址进行会话保持的模式，即第1次将某个客户端发起的请求转发到后端的某个Pod上，之后从相同的客户端发起的请求都将被
  转发到后端相同的Pod上。



# 32. 简述k8s Headless Service？

在某些应用场景中，若需要人为指定负载均衡器，不使用Service提供的默认负载均衡的功能，或者应用程序希望知道属于同组服务的其他实例。k8s提供了Headless Service 来实现这种功能，即不为Service 设置ClusterIP（入口IP 地址），仅通过Label Selector将后端的Pod列表返回给调用的客户端。



# 33. 简述k8s外部如何访问集群内的服务？

对于k8s，集群外的客户端默认情况，无法通过Pod的IP地址或者Service的虚拟IP地址:虚拟端口号进行访问。通常可以通过以下方式进行访问
k8s集群内的服务：

- 映射Pod到物理机：将Pod端口号映射到宿主机，即在Pod中采用hostPort方式，以使客户端应用能够通过物理机访问容器应用。
- 映射Service到物理机：将Service端口号映射到宿主机，即在Service中采用nodePort方式，以使客户端应用能够通过物理机访问容器应用。
- 映射Service到LoadBalancer：通过设置LoadBalancer映射到云服务商提供的LoadBalancer地址。这种用法仅用于在公有云服务提供商的云平台上设置
  Service的场景。



# 34. 简述k8s ingress？



k8s的Ingress资源对象，用于将不同URL的访问请求转发到后端不同的Service，以实现HTTP层的业务路由机制。k8s使用了Ingress策略和Ingress Controller，两者结合并实现了一个完整的Ingress负载均衡器。使用Ingress进行负载分发时，Ingress Controller 基于Ingress规则将客户端请求直接转发到Service对应的后端Endpoint（Pod）上，从而跳过kube-proxy的转发功能，kube-proxy不再起作用，全过程为：ingress controller + ingress 规则 ----> services。同时当Ingress Controller 提供的是对外服务，则实际上实现的是边缘路由器的功能。



# 35. 简述k8s镜像的下载策略？



K8s的镜像下载策略有三种：Always. Never. IFNotPresent。

- Always：镜像标签为latest时，总是从指定的仓库中获取镜像。
-  Never：禁止从仓库中下载镜像，也就是说只能使用本地镜像。
-  IfNotPresent：仅当本地没有对应镜像时，才从目标仓库中下载。默认的镜像下载策略是：当镜像标签是latest时，默认策略是Always；当镜像标签是自定义时（也就是标签不是latest），那么默认策略是IfNotPresent。



# 36. 简述k8s的负载均衡器？



负载均衡器是暴露服务的最常见和标准方式之一。
根据工作环境使用两种类型的负载均衡器，即内部负载均衡器或外部负载均衡器。内部负载均衡器自动平衡负载并使用所需配置分配容器，而外部负载均衡器将流量从外部负载引导至后端容器。



# 37. 简述k8s各模块如何与API Server通信？



k8s API Server作为集群的核心，负责集群各功能模块之间的通信。集群内的各个功能模块通过API Server将信息存入etcd，当需要获取和操作这些数据
时，则通过API Server 提供的REST 接口（用GET. LIST 或WATCH 方法）来实现，从而实现各模块之间的信息交互。如kubelet进程与API Server的交互：每个Node上的kubelet每隔一个时间周期，就会调用一次API Server的REST接口报告自身状态，API Server在接收到这些信息后，会将节点状态信息更新到etcd中。如kube-controller-manager进程与API Server的交互：kube-controllermanager中的Node Controller 模块通过API Server提供的Watch接口实时监控Node的信息，并做相应处理。如kube-scheduler进程与API Server的交互：Scheduler通过API Server的Watch接口监听到新建Pod副本的信息后，会检索所有符合该Pod要求的Node列表，开始执行Pod调度逻辑，在调度成功后将Pod绑定到目标节点上。



# 38. 简述k8s Scheduler作用及实现原理？



k8s Scheduler是负责Pod调度的重要功能模块，k8sScheduler在整个系统中承担了“承上启下”的重要功能，“承上”是指它负责接收Controller Manager创建的新Pod，为其调度至目标Node；“启下”是指调度完成后，目标Node上的kubelet服务进程接管后继工作，负责Pod接下来生命周期。k8s Scheduler的作用是将待调度的Pod（API新创建的Pod. ControllerManager为补足副本而创建的Pod等）按照特定的调度算法和调度策略绑定（Binding）到集群中某个合适的Node上，并将绑定信息写入etcd中。在整个调度过程中涉及三个对象，分别是待调度Pod列表. 可用Node列表，以及调度算法和策略。k8s Scheduler通过调度算法调度为待调度Pod列表中的每个Pod从Node列表中选择一个最适合的Node来实现Pod的调度。随后，目标节点上的kubelet通过API Server监听到k8s Scheduler产生的Pod绑定事件，然后获取对应的Pod清单，下载Image镜像并启动容器。



# 39. 简述k8s Scheduler使用哪两种算法将Pod绑定到worker节点？



k8s Scheduler根据如下两种调度算法将 Pod 绑定到最合适的工作节点：

-  预选（Predicates）：输入是所有节点，输出是满足预选条件的节点。kubescheduler根据预选策略过滤掉不满足策略的Nodes。如果某节点的资源不足或
  者不满足预选策略的条件则无法通过预选。如“Node的label必须与Pod的Selector一致”。
-  优选（Priorities）：输入是预选阶段筛选出的节点，优选会根据优先策略为通过预选的Nodes进行打分排名，选择得分最高的Node。例如，资源越富裕. 负载越小的Node可能具有越高的排名。



# 40. 简述k8s kubelet的作用？



在k8s集群中，在每个Node（又称Worker）上都会启动一个kubelet服务进程。该进程用于处理Master 下发到本节点的任务，管理Pod 及Pod 中的容器。每个kubelet进程都会在API Server上注册节点自身的信息，定期向Master汇报节点资源的使用情况，并通过cAdvisor监控容器和节点资源。



# 41. 简述k8s kubelet监控Worker节点资源是使用什么组件来实现的？



kubelet使用cAdvisor对worker节点资源进行监控。在 k8s 系统中，cAdvisor 已被默认集成到 kubelet 组件内，当 kubelet 服务启动时，它会自动启动cAdvisor 服务，然后 cAdvisor 会实时采集所在节点的性能指标及在节点上运行的容器的性能指标。



# 42. 简述k8s如何保证集群的安全性？



k8s通过一系列机制来实现集群的安全控制，主要有如下不同的维度：

- 基础设施方面：保证容器与其所在宿主机的隔离；
- 权限方面：
  - 最小权限原则：合理限制所有组件的权限，确保组件只执行它被授权的行为，
    通过限制单个组件的能力来限制它的权限范围。
  -  用户权限：划分普通用户和管理员的角色。
- 集群方面：
  -  API Server的认证授权：k8s集群中所有资源的访问和变更都是通过k8s API Server来实现的，因此需要建议采用更安全的HTTPS或
    Token来识别和认证客户端身份（Authentication），以及随后访问权限的授权（Authorization）环节。
  - API Server的授权管理：通过授权策略来决定一个API调用是否合法。对合法用户进行授权并且随后在用户访问时进行鉴权，建议采用更安全的RBAC
    方式来提升集群安全授权。
  -  敏感数据引入Secret机制：对于集群敏感数据建议使用Secret方式进行保护。
  -  AdmissionControl（准入机制）：对k8s api的请求过程中，顺序为：先经过认证 & 授权，然后执行准入操作，最后对目标对象进行操作。





# 43. 简述k8s准入机制？



在对集群进行请求时，每个准入控制代码都按照一定顺序执行。如果有一个准入控制拒绝了此次请求，那么整个请求的结果将会立即返回，并提示用户相应的error信息。

准入控制（AdmissionControl）准入控制本质上为一段准入代码，在对k8sapi的请求过程中，顺序为：先经过认证 & 授权，然后执行准入操作，最后对目标对象进行操作。常用组件（控制代码）如下：

- AlwaysAdmit：允许所有请求
- AlwaysDeny：禁止所有请求，多用于测试环境。
- ServiceAccount：它将serviceAccounts实现了自动化，它会辅助serviceAccount做一些事情，比如如果pod没有serviceAccount属性，它会自动添加一个default，并确保pod的serviceAccount始终存在。
-  LimitRanger：观察所有的请求，确保没有违反已经定义好的约束条件，这些条件定义在namespace中LimitRange对象中。
-  NamespaceExists：观察所有的请求，如果请求尝试创建一个不存在的namespace，则这个请求被拒绝。



# 44. 简述k8s RBAC及其特点（优势）？



RBAC是基于角色的访问控制，是一种基于个人用户的角色来管理对计算机或网络资源的访问的方法。
相对于其他授权模式，RBAC具有如下优势：

- 对集群中的资源和非资源权限均有完整的覆盖。
- 整个RBAC完全由几个API对象完成， 同其他API对象一样， 可以用kubectl或API进行操作。
- 可以在运行时进行调整，无须重新启动API Server。



# 45. 简述k8s Secret作用？



Secret对象，主要作用是保管私密数据，比如密码. OAuth Tokens. SSH Keys等信息。将这些私密信息放在Secret对象中比直接放在Pod或Docker Image中更安全，也更便于使用和分发。



# 46. 简述k8s Secret有哪些使用方式？



创建完secret之后，可通过如下三种方式使用：

- 在创建Pod时，通过为Pod指定Service Account来自动使用该Secret。
- 通过挂载该Secret到Pod来使用它。
- 在Docker镜像下载时使用，通过指定Pod的spc.ImagePullSecrets来引用它。





# 47. 简述k8s PodSecurityPolicy机制？



k8s PodSecurityPolicy是为了更精细地控制Pod对资源的使用方式以及提升安全策略。在开启PodSecurityPolicy准入控制器后，k8s默认不允许
创建任何Pod，需要创建PodSecurityPolicy策略和相应的RBAC授权策略（Authorizing Policies），Pod才能创建成功。



# 48. 简述k8s PodSecurityPolicy机制能实现哪些安全策略？


在PodSecurityPolicy对象中可以设置不同字段来控制Pod运行时的各种安全策略，
常见的有：

- 特权模式：privileged是否允许Pod以特权模式运行。
- 宿主机资源：控制Pod对宿主机资源的控制，如hostPID：是否允许Pod共享宿主机的进程空间。
- 用户和组：设置运行容器的用户ID（范围）或组（范围）。
-  提升权限：AllowPrivilegeEscalation：设置容器内的子进程是否可以提升权限，通常在设置非root用户（MustRunAsNonRoot）时进行设置。
-  SELinux：进行SELinux的相关配置。



# 49. 简述k8s网络模型？



k8s网络模型中每个Pod都拥有一个独立的IP地址，并假定所有Pod都在一个可以直接连通的. 扁平的网络空间中。所以不管它们是否运行在同一个
Node（宿主机）中，都要求它们可以直接通过对方的IP进行访问。设计这个原则的原因是，用户不需要额外考虑如何建立Pod之间的连接，也不需要考虑如何将容器端口映射到主机端口等问题。
同时为每个Pod都设置一个IP地址的模型使得同一个Pod内的不同容器会共享同一个网络命名空间，也就是同一个Linux网络协议栈。这就意味着同一个Pod内的容器可以通过localhost来连接对方的端口。
在k8s的集群里，IP是以Pod为单位进行分配的。一个Pod内部的所有容器共享一个网络堆栈（相当于一个网络命名空间，它们的IP地址. 网络设备. 配置等都是共享的）。



# 50. 简述k8s CNI模型？



CNI提供了一种应用容器的插件化网络解决方案，定义对容器网络进行操作和配置的规范，通过插件的形式对CNI接口进行实现。CNI仅关注在创建容器时分配网络
资源，和在销毁容器时删除网络资源。

在CNI模型中只涉及两个概念：容器和网络。

- 容器（Container）：是拥有独立Linux网络命名空间的环境，例如使用Docker或rkt创建的容器。容器需要拥有自己的Linux网络命名空间，这是加入网络的必
  要条件。
-  网络（Network）：表示可以互连的一组实体，这些实体拥有各自独立. 唯一的IP地址，可以是容器. 物理机或者其他网络设备（比如路由器）等。
  对容器网络的设置和操作都通过插件（Plugin）进行具体实现，CNI插件包括两种类型：CNI Plugin和IPAM（IP Address Management）Plugin。
  - CNI Plugin负责为容器配置网络资源
  - IPAM Plugin负责对容器的IP地址进行分配和管理。
  - IPAM Plugin作为CNI Plugin的一部分，与CNI Plugin协同工作。



# 51. 简述k8s网络策略？



为实现细粒度的容器间网络访问隔离策略，k8s引入Network Policy。

Network Policy的主要功能是对Pod间的网络通信进行限制和准入控制，设置允许访问或禁止访问的客户端Pod列表。Network Policy定义网络策略，配合策略控制器（Policy Controller）进行策略的实现。



# 52. 简述k8s网络策略原理？



Network Policy的工作原理主要为：policy controller需要实现一个APIListener，监听用户设置的Network Policy定义，并将网络访问规则通过各Node的
Agent进行实际设置（Agent则需要通过CNI网络插件实现）。



# 53. 简述k8s中flannel的作用？



Flannel可以用于k8s底层网络的实现，主要作用有：

- 它能协助k8s，给每一个Node上的Docker容器都分配互相不冲突的IP地址。
- 它能在这些IP地址之间建立一个覆盖网络（Overlay Network），通过这个覆盖网络，将数据包原封不动地传递到目标容器内。



# 54. 简述k8s Calico网络组件实现原理？


Calico 是一个基于BGP 的纯三层的网络方案，与OpenStack. k8s. AWS. GCE等云平台都能够良好地集成。
Calico在每个计算节点都利用Linux Kernel实现了一个高效的vRouter来负责数据转发。每个vRouter都通过BGP协议把在本节点上运行的容器的路由信息向整个
Calico网络广播，并自动设置到达其他节点的路由转发规则。
Calico保证所有容器之间的数据流量都是通过IP路由的方式完成互联互通的。Calico节点组网时可以直接利用数据中心的网络结构（L2或者L3），不需要额外的NAT. 隧道或者Overlay Network，没有额外的封包解包，能够节约CPU运算，提高网络效率。



# 55. 简述k8s共享存储的作用？



k8s对于有状态的容器应用或者对数据需要持久化的应用，因此需要更加可靠的存储来保存应用产生的重要数据，以便容器应用在重建之后仍然可以使用之前的数据。因此需要使用共享存储。



# 56. 简述k8s数据持久化的方式有哪些？



k8s通过数据持久化来持久化保存重要数据，常见的方式有：

- EmptyDir（空目录）：没有指定要挂载宿主机上的某个目录，直接由Pod内保部映射到宿主机上。类似于docker中的manager volume。
  -  场景：
    - 只需要临时将数据保存在磁盘上，比如在合并/排序算法中；
    - 作为两个容器的共享存储。
  -  特性：
    -  同个pod里面的不同容器，共享同一个持久化目录，当pod节点删除时，volume的数据也会被删除。
    - emptyDir的数据持久化的生命周期和使用的pod一致，一般是作为临时存储使用。
- Hostpath：将宿主机上已存在的目录或文件挂载到容器内部。类似于docker中的bind mount挂载方式。
  - 特性：增加了pod与节点之间的耦合。
- PersistentVolume（简称PV）：如基于NFS服务的PV，也可以基于GlusterFS的PV。它的作用是统一数据持久化目录，方便管理。



# 57. 简述k8s PV和PVC？



- PV是对底层网络共享存储的抽象，将共享存储定义为一种“资源”。
- PVC则是用户对存储资源的一个“申请”。



# 58. 简述k8s PV生命周期内的阶段？



某个PV在生命周期中可能处于以下4个阶段（Phaes）之一。

- Available：可用状态，还未与某个PVC绑定。
- Bound：已与某个PVC绑定。
- Released：绑定的PVC已经删除，资源已释放，但没有被集群回收。
-  Failed：自动资源回收失败。



# 59. 简述k8s所支持的存储供应模式？



k8s支持两种资源的存储供应模式：静态模式（Static）和动态模式（Dynamic）。

-  静态模式：集群管理员手工创建许多PV，在定义PV时需要将后端存储的特性进行设置。
- 动态模式：集群管理员无须手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种类型。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及与PVC的绑定。





# 60. 简述k8s CSI模型？



k8s CSI是k8s推出与容器对接的存储接口标准，存储提供方只需要基于标准接口进行存储插件的实现，就能使用k8s的原生存储机制为容器提供存储服务。CSI使得存储提供方的代码能和k8s代码彻底解耦，部署也与k8s核心组件分离，显然，存储插件的开发由提供方自行维护，就能为k8s用户提供更多的存储功能，也更加安全可靠。

CSI包括CSI Controller和CSI Node：

- CSI Controller的主要功能是提供存储服务视角对存储资源和存储卷进行管理和
  操作。
- CSI Node的主要功能是对主机（Node）上的Volume进行管理和操作。



# 61. 简述k8s Worker节点加入集群的过程？



通常需要对Worker节点进行扩容，从而将应用系统进行水平扩展。主要过程如下：

- 1. 在该Node上安装Docker. kubelet和kube-proxy服务；
- 2. 然后配置kubelet和kubeproxy的启动参数，将Master URL指定为当前k8s集群Master的地址，最后启动这些服务；
- 3. 通过kubelet默认的自动注册机制，新的Worker将会自动加入现有的k8s集群中；
- 4. k8s Master在接受了新Worker的注册之后，会自动将其纳入当前集群的调度范围。



# 62. 简述k8s Pod如何实现对节点的资源控制？



k8s集群里的节点提供的资源主要是计算资源，计算资源是可计量的能被申请. 分配和使用的基础资源。当前k8s集群中的计算资源主要包括CPU. GPU及Memory。CPU与Memory是被Pod使用的，因此在配置Pod时可以通过参数CPU Request及Memory Request为其中的每个容器指定所需使用的CPU与Memory量，k8s会根据Request的值去查找有足够资源的Node来调度此Pod。通常，一个程序所使用的CPU 与Memory 是一个动态的量，确切地说，是一个范围，跟它的负载密切相关：负载增加时，CPU和Memory的使用量也会增加。



# 63. 简述k8s Requests和Limits如何影响Pod的调度？

当一个Pod创建成功时，k8s调度器（Scheduler）会为该Pod选择一个节点来执行。对于每种计算资源（CPU和Memory）而言，每个节点都有一个能用于运行Pod的最大容量值。调度器在调度时，首先要确保调度后该节点上所有Pod的CPU和内存的Requests总和，不超过该节点能提供给Pod使用的CPU和Memory的最大容量值。

# 64. 述k8s Metric Service？

在k8s从1.10版本后采用Metrics Server作为默认的性能数据采集和监控，主要用于提供核心指标（Core Metrics），包括Node. Pod的CPU和内存使
用指标。对其他自定义指标（Custom Metrics）的监控则由Prometheus等组件来完成。

# 65. 简述k8s中，如何使用EFK实现日志的统一管理？

在k8s集群环境中，通常一个完整的应用或服务涉及组件过多，建议对日志系统进行集中化管理，通常采用EFK实现。
EFK是 Elasticsearch. Fluentd 和 Kibana 的组合，其各组件功能如下：

- Elasticsearch：是一个搜索引擎，负责存储日志并提供查询接口；
-  Fluentd：负责从 k8s 搜集日志，每个node节点上面的fluentd监控并收集该节点上面的系统日志，并将处理过后的日志信息发送给Elasticsearch；
- Kibana：提供了一个 Web GUI，用户可以浏览和搜索存储在 Elasticsearch 中的日志。

通过在每台node上部署一个以DaemonSet方式运行的fluentd来收集每台node上的日志。Fluentd将docker日志目录/var/lib/docker/containers和/var/log目录
挂载到Pod中，然后Pod会在node节点的/var/log/pods目录中创建新的目录，可以区别不同的容器日志输出，该目录下有一个日志文件链接到/var/lib/docker/contianers目录下的容器日志输出。

# 66. 简述k8s如何进行优雅的节点关机维护？

由于k8s节点运行大量Pod，因此在进行关机维护之前，建议先使用kubectl drain将该节点的Pod进行驱逐，然后进行关机维护。

# 67. 简述k8s集群联邦？

k8s 集群联邦可以将多个k8s 集群作为一个集群进行管理。因此，可以在一个数据中心/云中创建多个k8s集群，并使用集群联邦在一个地方控制/管理所有集群。

# 68. 简述Helm及其优势？

helm 是 k8s 的软件包管理工具。类似 Ubuntu 中使用的apt. Centos中使用的yum 或者Python中的 pip 一样。
Helm能够将一组K8S资源打包统一管理, 是查找. 共享和使用为k8s构建的软件的最佳方式。
Helm中通常每个包称为一个Chart，一个Chart是一个目录（一般情况下会将目录进行打包压缩，形成name-version.tgz格式的单一文件，方便传输和存储）

-  Helm优势
  在 k8s中部署一个可以使用的应用，需要涉及到很多的 k8s 资源的
  共同协作。使用helm则具有如下优势：
  - 统一管理. 配置和更新这些分散的 k8s 的应用资源文件；
  - 分发和复用一套应用模板；
  - 将应用的一系列资源当做一个软件包管理。
    对于应用发布者而言，可以通过 Helm 打包应用. 管理应用依赖关系. 管理应用版本并发布应用到软件仓库。
  -  对于使用者而言，使用 Helm 后不用需要编写复杂的应用部署文件，可以以简单
    的方式在 k8s 上查找. 安装. 升级. 回滚. 卸载应用程序。

# 69. k8s中的pod节点坏掉了，如何处理?

**自动故障转移与调度**：

- 当节点出现故障时，k8s会检测到这一变化，并自动将故障节点上的Pod重新调度到其他可用的节点上。
- k8s使用调度器（Scheduler）来确保Pod被分配到合适的节点上。调度器会考虑节点的资源可用性. Pod的亲和性和反亲和性规则等因素来做出决策（如果有节点标签选择，可临时在其他节点添加标签临时使用）。
- 如果自动重新调度失败，可以手动删除处于Pending状态的Pod，并让k8s重新调度它们到可用节点上。

**手动干预**

- 如果自动调度没有按预期工作，或者需要更精细的控制，管理员可以手动进行干预。例如，使用`kubectl cordon`命令将故障节点标记为不可调度，以防止新的Pod被调度到该节点上。
- 使用`kubectl drain`命令可以驱逐故障节点上的Pod，并确保它们在重新调度之前被优雅地终止。

**替换或修复故障节点**：

- 通过查看节点的日志及事件来定位问题，深入了解故障的原因并且修复节点，定位问题及解决之后做好预防措施，以防再次出现此类问题。
- 如果故障节点无法恢复，需要替换，在新节点准备好后，加入集群，可以使用`kubectl uncordon`命令将其标记为可调度状态，以便k8s可以在其上调度Pod。

# 70. k8s负责网络与存储的组件是什么？有几种？之间的区别是什么？

**网络组件（CNI 插件）**

CNI 插件用于管理 Kubernetes 集群中 Pod 的网络连接。不同的 CNI 插件提供不同的功能，包括网络隔离、网络策略、安全性和性能等。

常见的 CNI 插件

1. **Calico**：
   - 提供网络策略、网络安全和高性能的网络连接。
   - 支持 BGP、VXLAN 和 IPIP 等网络模式。
   - 强大的网络策略功能，可以实现细粒度的网络隔离和访问控制。
2. **Flannel**：
   - 主要用于简单的网络连接。
   - 采用 VXLAN、UDP 或 host-gw 等模式来实现 Pod 间的网络通信。
   - 配置简单，但功能相对较少。
3. **Weave**：
   - 提供自动化的网络配置和管理。
   - 支持加密、网络策略和多租户隔离。
   - 易于安装和配置，适合小型到中型集群。
4. **Cilium**：
   - 基于 eBPF 技术，提供高性能的网络连接。
   - 强大的网络策略和安全功能。
   - 支持 L7（应用层）网络策略，可以实现细粒度的流量控制。
5. **Kube-router**：
   - 提供网络路由、网络策略和负载均衡功能。
   - 结合了 BGP 路由、iptables 和 IPVS 技术。
   - 适合需要复杂网络路由的集群。

**存储组件（CSI 插件）**

CSI 插件用于管理 Kubernetes 集群中的存储资源。不同的 CSI 插件提供不同的存储功能，包括持久化存储、动态卷创建、快照和备份等。

常见的 CSI 插件

1. **Rook**：
   - 用于管理 Ceph 存储集群。
   - 提供块存储、文件存储和对象存储服务。
   - 高可用和可扩展的存储解决方案。
2. **OpenEBS**：
   - 基于容器的存储解决方案。
   - 提供动态卷创建、快照和克隆功能。
   - 支持多种存储引擎（如 Jiva、cStor、LocalPV）。
3. **Longhorn**：
   - 分布式块存储系统。
   - 提供快照、备份和恢复功能。
   - 易于部署和管理，适合小型到中型集群。
4. **GlusterFS**：
   - 分布式文件系统。
   - 提供高可用和可扩展的存储解决方案。
   - 适用于需要文件存储的应用程序。
5. **NFS**：
   - 基于网络文件系统协议。
   - 提供简单的文件存储服务。
   - 适用于需要共享文件存储的应用程序。

**CNI 和 CSI 插件的区别**

- **功能**：
  - CNI 插件负责网络连接和管理，包括网络隔离、安全策略和流量控制。
  - CSI 插件负责存储管理和操作，包括持久化存储、动态卷创建和数据快照。
- **安装和配置**：
  - CNI 插件需要在每个节点上安装和配置，确保 Pod 能够相互通信。
  - CSI 插件通常需要在控制平面和工作节点上安装，并且可能需要外部存储系统的支持。
- **使用场景**：
  - CNI 插件适用于需要复杂网络管理、网络策略和高性能网络连接的场景。
  - CSI 插件适用于需要持久化存储、高可用存储和动态存储管理的场景。

# 71. k8s的QOS服务级别是什么

Quality of Service (QoS) 是指Pod的资源管理和调度策略，它基于Pod请求的CPU和内存资源来定义。QoS级别决定了Pod在资源不足时的调度和驱逐行为。Kubernetes定义了三种QoS级别：

**Guaranteed（保证型）**

- Pod 里的每个容器都必须有内存/CPU 限制和请求，而且值必须相等。如果一个容器只指明limit而未设定request，则request的值等于limit值。

- 这种QoS级别提供了最高的优先级，因为Kubernetes会确保这些Pod始终获得它们所请求的资源量。
- 如果集群资源不足，其他较低QoS级别的Pod可能会被驱逐以释放资源给Guaranteed级别的Pod

**Burstable（可突发型）**

- Pod中的至少一个容器设置了CPU或内存的requests，但没有设置相应的limits，或者设置了limits但requests和limits的值不相等。
- 默认情况下，如果Pod没有指定任何requests或limits，它也会被归类为Burstable。
- Burstable Pod可以使用的资源量会基于它们的requests，但在需要时可以“突发”使用到limits所定义的上限。
- 当资源不足时，这些Pod的驱逐优先级低于Guaranteed Pod，但高于BestEffort Pod。

**BestEffort（尽力而为型）**

- Pod中的容器没有设置CPU或内存的requests和limits。
- 这种QoS级别的Pod没有资源保证，它们可以使用任何可用的资源，但在资源不足时也是最先被驱逐的。
- 通常，BestEffort Pod用于批处理作业、测试工作负载或其他非关键性任务。



# 72. pod  长期不更新，导致pod 过于集中，该如何处理

**维护节点健康**

- 节点监控：定期检查节点的健康状况，包括硬件状态、内核日志等，及时发现并处理异常。
- 节点更新：适时对节点进行更新和维护，包括操作系统补丁、Kubernetes组件升级等，确保节点的稳定性。

**调整调度策略**

- 节点亲和性：设置Pod的节点`nodeAffinity`亲和性或反亲和性规则，控制Pod在不同节点上的分布，避免某些节点上Pod数量过多。
- **资源限制**：合理配置Pod的资源请求和限制，确保集群资源的公平分配，避免部分节点资源紧张而其他节点空闲的情况。

**优化资源利用**

- 弹性伸缩：根据应用负载自动扩展或收缩Pod的数量，提高资源利用率并保持应用性能。
- 资源回收：对于长时间处于闲置状态的Pod，考虑自动回收资源以供其他应用使用。

**水平伸缩应用：**

- 如果Pod过于集中是因为某个应用的需求增加，可以考虑水平伸缩该应用，通过增加Pod的副本数来分散负载。
- 使用K8S的Horizontal Pod Autoscaler（HPA）功能，可以根据应用的CPU或内存使用情况自动调整Pod的副本数。

**清理不再需要的Pod和资源：**

- 定期清理不再需要的Pod和相关的资源，如ConfigMap、Secret等，以释放集群资源。
- 清理Terminating Pod，分析处于Terminating状态的Pod，找出删除操作被阻塞的原因，如磁盘空间不足、finalizers设置错误等。
- 强制删除Pod：对于无法正常删除的Pod，可以使用`kubectl delete pod --grace-period=0 --force`命令强制删除。



# 73. k8s中一旦产线 上节点出现 POD 的 网络地址 回环 该如何处理

如果生产环境（产线）上的节点出现Pod的网络地址回环（loopback）问题，这通常意味着Pod无法正确地与其他Pod或外部服务进行通信。处理此类问题通常涉及几个关键步骤，以下是一些建议的解决方案：

**诊断问题**

- 检查Pod状态：使用`kubectl get pods`和`kubectl describe pod <pod-name>`命令来检查Pod的状态和事件。
- 检查网络插件：Kubernetes使用网络插件（如Flannel、Calico等）来管理Pod之间的网络通信。确保这些插件正在正常运行。
- 查看节点日志：检查Kubernetes节点（master和worker）的日志，特别是与网络和CNI（容器网络接口）相关的日志。

**检查网络配置**

- Pod网络CIDR：确保Pod的CIDR（无类别域间路由）没有重叠，并且与节点和服务的CIDR不冲突。
- Service CIDR：确保Service的CIDR没有与Pod或节点的CIDR重叠。
- 网络插件(flanel,calico)：对于插件的配置进行排查

**检查防火墙和安全组设置**

- 节点防火墙：确保节点上的防火墙规则允许Kubernetes网络插件和Pod之间的通信。
- 云提供商安全组：如果你在云上运行Kubernetes（如AWS、Azure、GCP等），请检查与节点和Pod相关的安全组设置。



# 74. K8S 直接删错 namespace, pod会删错不?该怎么处理

在Kubernetes中，如果你直接删除一个Namespace，该Namespace下的所有资源，包括Pods、Deployments、Services、ConfigMaps、Secrets等，都会被级联删除。因为在Kubernetes的设计中，Namespace是一个高级的分组机制，用来隔离不同的资源集合。当一个Namespace被删除时，该Namespace内所有的子资源都会被视为不再需要，并随之被清理。

因此，如果你不小心删除了一个错误的Namespace，那么该Namespace内所有的Pod也会被自动删除，且这一过程通常是不可逆的。所以在执行`kubectl delete namespace <namespace-name>`命令前，务必确认无误，以避免不必要的数据丢失。如果确实发生了误删，恢复数据可能需要依赖于备份或从其他副本（如果有的话）进行恢复。



# k8s 产线上下节点的流程是什么

**节点上线流程（上线新节点到K8s集群）**

1. **准备节点**：确保新的节点已经安装了必要的操作系统、网络配置和Docker等容器运行时。
2. **配置节点**：在新节点上安装kubelet、kube-proxy等K8s组件，并将节点加入到K8s集群中。这通常涉及编辑kubelet的配置文件，指定Master节点的地址和集群的相关参数。
3. **验证节点**：通过kubectl命令检查新的节点是否已成功加入集群。可以使用`kubectl get nodes`命令查看集群中所有节点的状态。

**节点下线流程（从K8s集群中移除节点）**

1. **标记节点为不可调度**：使用`kubectl cordon <NODE_NAME>`命令将需要下线的节点标记为不可调度，以防止新的Pod被调度到该节点上。
2. **驱逐节点上的Pod**：使用`kubectl drain <NODE_NAME> --ignore-daemonsets`命令驱逐节点上运行的Pod（除了DaemonSet类型的Pod）。这将尝试将Pod安全地迁移到集群中的其他节点。
   - 注意：在实际生产环境中，由于驱逐操作可能会导致大量的Pod同时删除和创建，可能会对业务造成一定影响。因此，建议在低峰时段执行此操作，并考虑使用手动删除Pod的方式来确保服务的稳定性。
3. **删除节点**：在确认节点上已没有业务Pod后，可以使用`kubectl delete node <NODE_NAME>`命令从K8s集群中删除该节点。
4. **验证节点删除**：使用`kubectl get nodes`命令验证节点是否已成功从集群中删除。