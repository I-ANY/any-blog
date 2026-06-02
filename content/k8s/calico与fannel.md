+++
title = "calico与fannel"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 3
+++

# calico

## 简介

Calico是一个用于容器、虚拟机和主机的开源网络和网络安全解决方案。它是一个纯三层解决方案，利用BGP协议为容器或虚拟机提供IP地址，并提供网络安全功能，包括网络策略和加密。

Calico 通过将网络策略应用于标签和选择器，提供了一种简单而强大的方法来保护容器或虚拟机之间的通信，并限制容器或虚拟机可以访问的网络资源。它还支持基于 Kubernetes 和 OpenStack 等平台的网络自动化和集成。
Calico 的另一个重要特点是其可扩展性。它使用了基于 BGP 的路由技术，这使得它能够轻松地扩展到非常大规模的网络中，而不会降低性能。

由于Calico是一种纯三层的方案，因此可以避免与二层方案相关的数据包封装的操作，中间没有任何的NAT，没有任何的overlay，所以它的转发效率是所有方案中最高的，因为它的包直接走原生TCP/IP的协议栈，它的隔离也因为这个栈而变得好做。因为TCP/IP的协议栈提供了一整套的防火墙的规则，所以它可以通过IPTABLES的规则达到比较复杂的隔离逻辑。

架构：

![image-20240530183659614](./images/calico/image-20240530183659614.png)

- **Felix：**Calico Agetnt，在K8S集群的每台节点上，主要负责管理和维护该节点上的网络和安全策略，如网络接口管理和监听、路由、ARP管理、ACL管理和同步、状态上报等。
- **ETCD：**分布式键值存储，用来存储网络元数据、安全策略以及节点的状态信息，确保Calico网络状态的一致性和准确性，可以和K8S的etcd何用
- **BGP Client(BIRD)：**跟Felix一样，每一个节点上都部署GBP Client，主要吧Felix写入Kernel的路由信息分发到当前Calico网络，确保各节点间的通信的有效性。
- **BGP Route Reflector（BIRD）：**在大型网络中，仅仅使用BGP Client形成mesh全网互联的方案会导致规模现在，所有节点之间两辆互联，需要N^2个连接，可使用BGP的Router Reflector的方法，会使用BGP Client仅与特定RR节点互联并做路由同步，从而大大减少连接数，可大规模部署使用。

## Calico两种网络工作模式

**IPIP：**把IP层封装到IP层的一个tunnel，其作用基本上就相当于一个基于IP层的网桥。普通的网桥是基于MAC层的，而这个IPIP则是通过两端的路由做一个tunnel，把两个本来不通的网络通过点对点连接起来。当Calico以IPIP模式部署完毕后，node上会有一个`tunl0`的网卡设备，这是ipip做隧道封装用的，也是一种overlay模式的网络。

<img src="./images/calico/image-20240530192807260.png" alt="image-20240530192807260" style="zoom:50%;" />

跨node节点的pod网络通信链路：

1. Pod1 发起请求，首先到`veth0` 虚拟网卡(每个 Pod 都有的一个虚拟网卡，该虚拟网卡的一端连接到 `pod` 内部，另一端连接到 Node 上的网桥，通常是 Linux 内核的 `cni0` 或 `caliXXX` 接口)
2. 然后到``caliXXX` 网桥（ Node1 上的网桥接口，用来处理所有在该 Node 上的 Pod 的网络流量），数据包通过 Node 的路由表进行路由决策，基于 Pod 的目标 IP 地址，判断数据包是发往同一 Node 上的 Pod 还是其他 Node 上的 Pod；
3. 然后到`tunl0` 隧道接口（或 IPIP 隧道），`node1` 会根据路由表找到到 `pod2` 所在的 Node (`node2`) 的路由信息，数据包将首先通过 IPIP 隧道进行封装，封装后的包头会包含 `node1` 和 `node2` 的节点 IP 地址，封装后的数据包通过底层网络发送到 `node2`
4. 当数据包到达 `node2` 后，`node2` 的 Calico 将解封装数据包，移除 IPIP 隧道包头，恢复原始的 IP 包（源 IP 是 `pod1`，目标 IP 是 `pod2`）。
5. 然后，数据包通过 `caliXXX` 虚拟接口进入 Node 的网络栈。`node2` 会通过其路由表进行判断，确定数据包的目标 IP (`pod2` 的 IP) 属于本地的 Pod。路由表指向 `caliXXX` 网桥，数据包通过 `veth` 接口被转发给 pod2

**BGP：**边界网关协议(Border Gateway Protocol)，一个去中心化自治路由协议，它通过维护IP路由表或“前缀”来实现自治系统之间的可达性，通常作为大规模数据中心维护不同自治系统之间路由信息的矢量路由协议。

<img src="./images/calico/image-20240530192822426.png" alt="image-20240530192822426" style="zoom:50%;" />

1. Pod1 发起请求，首先到`veth0` 虚拟网卡(每个 Pod 都有的一个虚拟网卡，该虚拟网卡的一端连接到 `pod` 内部，另一端连接到 Node 上的网桥，通常是 Linux 内核的 `cni0` 或 `caliXXX` 接口)
2. 然后到``caliXXX` 网桥（ Node1 上的网桥接口，用来处理所有在该 Node 上的 Pod 的网络流量），数据包通过 Node 的路由表进行路由决策，基于 Pod 的目标 IP 地址，判断数据包是发往同一 Node 上的 Pod 还是其他 Node 上的 Pod，如果是其他node上，Node1 将直接将数据包通过底层网络路由发送到 `node2` 的主机；
3. 当数据包到达 `node2` 后，会通过其路由表进行判断，确定数据包的目标 IP (`pod2` 的 IP) 属于本地的 Pod。路由表指向 `caliXXX` 网桥，数据包通过 `veth` 接口被转发给 pod2

## 配置

Calico的部署文件默认使用的是IPIP 隧道模式。如果要使用纯BGP路由模式或者混合模式可以修改变量CALICO_IPV4POOL_IPIP的值，可用值如下：

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

```bash
# 已经安装了calico之后，修改IPPool中的ipipMode为Never，也就是禁用IPIP模式
# kubectl edit ippool
...
ipipMode: Never
```



# flannel

## 简介

Flannel实质上是一种“覆盖网络(overlay network)”，也就是将TCP数据包装在另一种网络包里面进行路由转发和通信，目前已经支持UDP、VxLAN、AWS VPC和GCE路由等数据转发方式。 

设计目的就是为集群中的所有节点重新规划IP地址的使用规则，从而使得不同节点上的容器能够获得“同属一个内网”且”不重复的”IP地址，并让属于不同节点上的容器能够直接通过内网IP通信。

**工作架构模式：**

![image-20240531093636163](./images/calico与fannel/image-20240531093636163.png)

## **支持的网络模式**

- UDP： 使用用户态udp封装，默认使用8285端口。由于是在用户态封装和解包，性能上有较大的损失，也是性能最差的一种。目前已经被弃用；

- VXLAN： 虚拟可扩展的局域网,是一种虚拟化隧道通信技术，是一种overlay(覆盖网络)技术，通过三层的网络搭建虚拟的二层网络；vxlan封装，需要配置VNI，Port（默认8472）和GBP;  是Linux内核本身支持的一种网络虚拟化技术，是内核的一个模块，在内核态实现封装解封装，构建出覆盖网络，其实就是一个由各宿主机上的Flannel.1设备组成的虚拟二层网络，由于VXLAN由于额外的封包解包，导致其性能较差；

  ```bash
  # 组网模式
  Network Overlay 隧道封装在物理交换机完成。这种Overlay的优势在于物理网络设备性能转发性能比较高，可以支持非虚拟化的物理服务器之间的组网互通。
  Host Overlay 隧道封装在vSwitch、具备VXLAN功能的软件完成，不用增加新的网络设备即可完成Overlay部署，可以支持虚拟化的服务器之间的组网互通。
  Hybrid Overlay是Network Overlay和Host Overlay的混合组网，可以支持物理服务器和虚拟服务器之间的组网互通。
  ```

  架构图：

  ![image-20240531105413824](./images/calico与fannel/image-20240531105413824.png)

  上图数据流向

  ```yaml
  1、源容器veth0向目标容器发送数据，根据容器内的默认路由，数据首先发送给宿主机的cni0网桥
  2、宿主机cni0网桥接受到数据后，宿主机查询路由表，pod相关的路由都是交由flannel.1网卡，因此，将其转发给flannel.1虚拟网卡处理
  3、flannel.1接受到数据后，查询etcd数据库，获取目标pod网段对应的目标宿主机地址、目标宿主机的flannel网卡的mac地址、vxlan vnid等信息。然后对数据进行udp封装如下：
      udp头封装：
        source port >1024，target port 8427
      udp内部封装：
        vxlan封装：vxlan vnid等信息
        original layer 2 frame封装：source {源 flannel.1网卡mac地址} target{目标flannel.1网卡的mac地址}
  完成以上udp封装后，将数据转发给物理机的eth0网卡
  
  4、宿主机eth0接收到来自flannel.1的udp包，还需要将其封装成正常的通信用的数据包，为它加上通用的ip头、二层头，这项工作在由linux内核来完成。
      Ethernet Header的信息：
          source:{源宿主机机网卡的MAC地址}
          target:{目标宿主机网卡的MAC地址}
      IP Header的信息：
          source:{源宿主机网卡的IP地址}
          target:{目标宿主机网卡的IP地址}
  通过此次封装，一个真正可用的数据包就封装完成，可以通过物理网络传输了。
  5、目标宿主机的eth0接收到数据后，对数据包进行拆封，拆到udp层后，将其转发给8427端口的flannel进程
  6、目标宿主机端flannel拆除udp头、vxlan头，获取到内部的原始数据帧，在原始数据帧内部，解析出源容器ip、目标容器ip，重新封装成通用的数据包，查询路由表，转发给cni0网桥；
  7、最后，cni0网桥将数据送达目标容器内的veth0网卡，完成容器之间的数据通信
  ```

  VXLAN后端支持DirectRouting模式，即在集群的各节点上添加必要的路由信息，让Pod间的报文通过节点的二层网络直接传送。如下图所示，只有通信双方Pod所在节点不在同一个二层网络时才启用传统的VXLAN隧道方式转发流量。假如k8s集群的节点都位于同一个二层网络中，DirectRouting模式下的Pod通信基本接近于直接使用二层网络。即使节点分布在不同的网络中，合理使用也可以节省一部分隧道开销。
  ![image-20240531110059917](./images/calico与fannel/image-20240531110059917.png)

- Host-gw: 直接路由的方式，将容器网络的路由信息直接更新到主机的路由表中，仅适用于二层直接可达的网络  推荐使用，效率极高；

  ```bash
  host-gw模式通过建立主机IP到主机上对应flannel子网的mapping，以直接路由的方式联通flannel的各个子网。这种互联方式没有vxlan等封装方式带来的负担，通过路由机制，实现flannel网络数据包在主机之间的转发。但是这种方式也有不足，那就是所有节点之间都要相互有点对点的路由覆盖，并且所有加入flannel网络的主机需要在同一个LAN里面；所以每个节点上有n-1个路由，而n个节点一共有n(n-1)/2个路由以保证flannel的flat网络能力
  
  host-gw的工作原理，其实就是在宿主机上，添加一条到达每个fannel子网的路由，这条路由的下一跳即网关就是相对应的fannel子网所在的“主机”。这些路由规则是flanneld根据容器部署的场景创建和更新的
  ```

  ![image-20240531100508748](./images/calico与fannel/image-20240531100508748.png)

## 配置

Flannel官方的部署文件默认使用vxlan后端，相关配置定义在kube-flannel名称空间下configmap/kube-flannel-cfg资源对象中，内容如下：

```yaml
...
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",	
      "Backend": {		
        "Type": "vxlan"
        # "Directrouting": true     # 开启DirectRouting模式，仅在vxlan模式下可用，默认不开启，无此配置
      }
    }
    
...

---
Network：Flannel全局使用的子网，即pod cidr的值
SubnetLen：子网分割的长度，在全局子网掩码小于24时（例如16），默认为24
SubnetMin：分配给节点使用的起始子网，默认为切割完成后的第一个子网
SubnetMax：分给给节点使用的最大子网，默认为切割完成后的最后一个子网
Backend：Flannel使用的后端(vxlan，host-gw)，以及后端的配置
```

另外，flannel还会在运行的节点上生成一个环境变量文件，默认是/run/flannel/subnet.env，其包含本节点使用的子网、mtu等信息。例如下面的示例：

```bash
[root@master-01]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16	#flannel全局网段
FLANNEL_SUBNET=10.244.0.1/24	#本节点子网
FLANNEL_MTU=1450	#容器接口mtu值
FLANNEL_IPMASQ=true	#地址映射

```

**两者区别：**

**calico**

- **主要功能**: Calico 是一个功能全面的网络插件，不仅提供网络通信，还支持细粒度的网络安全策略（Network Policy）。它的设计目标是提供高性能、可扩展的网络解决方案，适合复杂的生产环境。
- **工作模式**:  Calico 通过三层路由实现 Pod 之间的网络通信，默认情况下使用 BGP 来通告路由。它也支持 IPIP 隧道和 VXLAN 隧道。Calico 的架构相对较重，但功能强大。

- BGP 模式: Calico 可以使用 BGP 协议直接通告和交换路由信息，在大多数情况下，无需进行封装或使用隧道（如果底层网络支持直连）。
- 隧道模式（可选）: Calico 支持 IPIP 和 VXLAN 隧道封装，在底层网络不支持直连的情况下使用。

-  网络策略（Network Policy）支持 ：Calico 提供完整的网络策略支持。通过 Kubernetes 的 Network Policy API，Calico 可以实现细粒度的流量控制，如允许或拒绝某些 Pod 之间的流量，基于 IP 地址、命名空间或 Pod 标签等定义的规则。Calico 的网络策略功能非常强大，适合生产环境中的安全需求。
- **特点**: 高性能、可扩展性好、支持网络策略（Network Policy）。

**Flannel**:

- **主要功能**: Flannel 的设计目标是简单地为 Kubernetes 提供网络通信，最早是为了满足 Kubernetes 1.0 的基本网络需求。Flannel 主要用于为 Pod 提供基本的二层网络通信，支持多种后端实现（如 VXLAN 和 UDP）。
- **工作模式**:  Flannel 的网络实现 依赖二层封装（如 VXLAN 和 UDP），数据包通过隧道传输到目标节点。网络结构相对简单，重点在于简化部署和管理。默认使用隧道来封装和传递 Pod 的网络流量。常用的后端包括：

- VXLAN：使用虚拟扩展局域网协议，在网络层封装流量，适用于不同网络环境。
- host-gw 模式：使用主机的网关模式，适用于所有节点之间都能直接路由的情况。
- UDP：一种简单的封装方式，但效率较低，适合简单的网络拓扑。

- **特点**: 轻量级、易于配置和使用、但不支持网络策略（Network Policy）控制。

| 特性     | Calico                     | Flannel                  |
| -------- | -------------------------- | ------------------------ |
| 主要功能 | 网络通信 + 网络策略        | 基本网络通信             |
| 路由方式 | 三层路由，BGP 或 IPIP 隧道 | 隧道模式（VXLAN、UDP）   |
| 网络策略 | 支持（细粒度控制）         | 不支持                   |
| 性能     | 高性能（无隧道时性能最佳） | 较低（使用隧道）         |
| 可扩展性 | 高，可支持大规模集群       | 较低，适合中小型集群     |
| 适用场景 | 企业级生产环境，大规模集群 | 中小型集群，开发测试环境 |









