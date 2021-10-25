# 循序渐进理解CNI机制与Flannel工作原理



CNI，它的全称是 Container Network Interface，即容器网络的 API 接口。kubernetes 网络的发展方向是希望通过插件的方式来集成不同的网络方案， CNI 就是这一努力的结果。CNI 只专注解决容器网络连接和容器销毁时的资源释放，提供一套框架，所以 CNI 可以支持大量不同的网络模式，并且容易实现。

## 从网络模型到 CNI

在理解 CNI 机制以及 Flannel 等具体实现方案之前，首先要理解问题的背景，这里从 kubernetes 网络模型开始回顾。

从底层网络来看，kubernetes 的网络通信可以分为三层去看待：

- Pod 内部容器通信；
- 同主机 Pod 间容器通信；
- 跨主机 Pod 间容器通信；

对于前两点，其网络通信原理其实不难理解。

1. 对于 Pod 内部容器通信，由于 Pod 内部的容器处于同一个 Network Namespace 下（通过 Pause 容器实现），即共享同一网卡，因此可以直接通信。
2. 对于同主机 Pod 间容器通信，Docker 会在每个主机上创建一个 Docker0 网桥，主机上面所有 Pod 内的容器全部接到网桥上，因此可以互通。

而对于第三点，跨主机 Pod 间容器通信，Docker 并没有给出很好的解决方案，而对于 Kubernetes 而言，跨主机 Pod 间容器通信是非常重要的一项工作，但是有意思的是，Kubernetes 并没有自己去解决这个问题，而是专注于容器编排问题，对于跨主机的容器通信则是交给了第三方实现，这就是 CNI 机制。

CNI，它的全称是 Container Network Interface，即容器网络的 API 接口。kubernetes 网络的发展方向是希望通过插件的方式来集成不同的网络方案， CNI 就是这一努力的结果。CNI 只专注解决容器网络连接和容器销毁时的资源释放，提供一套框架，所以 CNI 可以支持大量不同的网络模式，并且容易实现。平时比较常用的 CNI 实现有 Flannel、Calico、Weave 等。

CNI 插件通常有三种实现模式：

- Overlay：靠隧道打通，不依赖底层网络；
- 路由：靠路由打通，部分依赖底层网络；
- Underlay：靠底层网络打通，强依赖底层网络；

在选择 CNI 插件时是要根据自己实际的需求进行考量，比如考虑 NetworkPolicy 是否要支持 Pod 网络间的访问策略，可以考虑 Calico、Weave；Pod 的创建速度，Overlay 或路由模式的 CNI 插件在创建 Pod 时比较快，Underlay 较慢；网络性能，Overlay 性能相对较差，Underlay 及路由模式相对较快。

## Flannel 工作原理

CNI 中经常见到的解决方案是 Flannel，由CoreOS推出，Flannel 采用的便是上面讲到的 Overlay 网络模式。

### Overlay 网络简介

Overlay 网络 (overlay network) 属于应用层网络，它是面向应用层的，不考虑网络层，物理层的问题。

具体而言， Overlay 网络是指建立在另一个网络上的网络。该网络中的结点可以看作通过虚拟或逻辑链路而连接起来的。虽然在底层有很多条物理链路，但是这些虚拟或逻辑链路都与路径一一对应。例如：许多P2P网络就是 Overlay 网络，因为它运行在互连网的上层。 Overlay 网络允许对没有IP地址标识的目的主机路由信息，例如：Freenet 和DHT（分布式哈希表）可以路由信息到一个存储特定文件的结点，而这个结点的IP地址事先并不知道。

Overlay 网络被认为是一条用来改善互连网路由的途径，让二层网络在三层网络中传递，既解决了二层的缺点，又解决了三层的不灵活。

### Flannel的工作原理

Flannel 实质上就是一种 Overlay 网络，也就是将 TCP 数据包装在另一种网络包里面进行路由转发和通信，目前已经支持 UDP、VxLAN、AWS VPC 和 GCE 路由等数据转发方式。

Flannel会在每一个宿主机上运行名为 flanneld 代理，其负责为宿主机预先分配一个子网，并为 Pod 分配IP地址。Flannel 使用Kubernetes 或 etcd 来存储网络配置、分配的子网和主机公共IP等信息。数据包则通过 VXLAN、UDP 或 host-gw 这些类型的后端机制进行转发。

Flannel 规定宿主机下各个Pod属于同一个子网，不同宿主机下的Pod属于不同的子网。

### Flannel 工作模式

支持3种实现：UDP、VxLAN、host-gw，

- UDP 模式：使用设备 flannel.0 进行封包解包，不是内核原生支持，频繁地内核态用户态切换，性能非常差；
- VxLAN 模式：使用 flannel.1 进行封包解包，内核原生支持，性能较强；
- host-gw 模式：无需 flannel.1 这样的中间设备，直接宿主机当作子网的下一跳地址，性能最强；

host-gw的性能损失大约在10%左右，而其他所有基于VxLAN“隧道”机制的网络方案，性能损失在20%~30%左右。

#### UDP 模式

官方已经不推荐使用 UDP 模式，性能相对较差。

UDP 模式的核心就是通过 TUN 设备 flannel0 实现。TUN设备是工作在三层的虚拟网络设备，功能是：在操作系统内核和用户应用程序之间传递IP包。 相比两台宿主机直接通信，多出了 flanneld 的处理过程，这个过程，使用了 flannel0 这个TUN设备，仅在发出 IP包的过程中经过多次用户态到内核态的数据拷贝（linux的上下文切换代价比较大），所以性能非常差 原理如下：![img](https://blog.yingchi.io/posts/2020/8/k8s-flannel/662544-20191022103006305-198547277.png)

以flannel0为例，操作系统将一个IP包发给flannel0，flannel0把IP包发给创建这个设备的应用程序：flannel进程（内核态->用户态） 相反，flannel进程向flannel0发送一个IP包，IP包会出现在宿主机的网络栈中，然后根据宿主机的路由表进行下一步处理（用户态->内核态） 当IP包从容器经过docker0出现在宿主机，又根据路由表进入flannel0设备后，宿主机上的flanneld进程就会收到这个IP包

flannel管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的“子网”，子网与宿主机的对应关系，存在Etcd中（例如Node1的子网是100.96.1.0/24，container-1的IP地址是100.96.1.2）

当flanneld进程处理flannel0传入的IP包时，就可以根据目的IP地址（如100.96.2.3），匹配到对应的子网（比如100.96.2.0/24），从Etcd中找到这个子网对应的宿主机的IP地址（10.168.0.3）

然后 flanneld 在收到container-1给container-2的包后，把这个包直接封装在UDP包里，发送给Node2（UDP包的源地址，就是Node1，目的地址是Node2）

每台宿主机的flanneld都监听着8285端口，所以flanneld只要把UDP发给Node2的8285端口就行了。然后Node2的flanneld再把IP包发送给它所管理的TUN设备flannel0，flannel0设备再发给docker0

#### VxLAN模式

VxLAN，即Virtual Extensible LAN（虚拟可扩展局域网），是Linux本身支持的一网种网络虚拟化技术。VxLAN可以完全在内核态实现封装和解封装工作，从而通过“隧道”机制，构建出 Overlay 网络（Overlay Network）

VxLAN的设计思想是： 在现有的三层网络之上，“覆盖”一层虚拟的、由内核VxLAN模块负责维护的二层网络，使得连接在这个VxLAN二层网络上的“主机”（虚拟机或容器都可以），可以像在同一个局域网（LAN）里那样自由通信。 为了能够在二层网络上打通“隧道”，VxLAN会在宿主机上设置一个特殊的网络设备作为“隧道”的两端，叫VTEP：VxLAN Tunnel End Point（虚拟隧道端点） 原理如下：

![img](https://blog.yingchi.io/posts/2020/8/k8s-flannel/662544-20191022103037567-147724494.png)

flannel.1设备，就是VxLAN的VTEP，即有IP地址，也有MAC地址 与UDP模式类似，当container-发出请求后，上的地址10.1.16.3的IP包，会先出现在docker网桥，再路由到本机的flannel.1设备进行处理（进站），为了能够将“原始IP包”封装并发送到正常的主机，VxLAN需要找到隧道的出口：宿主机的VTEP设备，这个设备信息，由宿主机的flanneld进程维护

VTEP设备之间通过二层数据桢进行通信 源VTEP设备收到原始IP包后，在上面加上一个目的MAC地址，封装成一个导去数据桢，发送给目的VTEP设备（获取 MAC地址需要通过三层IP地址查询，这是ARP表的功能）![img](https://blog.yingchi.io/posts/2020/8/k8s-flannel/662544-20191022103148437-452134686.png)

封装过程只是加了一个二层头，不会改变“原始IP包”的内容 这些VTEP设备的MAC地址，对宿主机网络来说没什么实际意义，称为内部数据桢，并不能在宿主机的二层网络传输，Linux内核还需要把它进一步封装成为宿主机的一个普通的数据桢，好让它带着“内部数据桢”通过宿主机的eth0进行传输，Linux会在内部数据桢前面，加上一个我死的VxLAN头，VxLAN头里有一个重要的标志叫VNI，它是VTEP识别某个数据桢是不是应该归自己处理的重要标识。 在Flannel中，VNI的默认值是1，这也是为什么宿主机的VTEP设备都叫flannel.1的原因

一个flannel.1设备只知道另一端flannel.1设备的MAC地址，却不知道对应的宿主机地址是什么。 在linux内核里面，网络设备进行转发的依据，来自FDB的转发数据库，这个flannel.1网桥对应的FDB信息，是由flanneld进程维护的 linux内核再在IP包前面加上二层数据桢头，把Node2的MAC地址填进去。这个MAC地址本身，是Node1的ARP表要学习的，需 Flannel维护，这时候Linux封装的“外部数据桢”的格式如下![img](https://blog.yingchi.io/posts/2020/8/k8s-flannel/662544-20191022103125394-40388131.png)

然后Node1的flannel.1设备就可以把这个数据桢从eth0发出去，再经过宿主机网络来到Node2的eth0 Node2的内核网络栈会发现这个数据桢有VxLAN Header，并且VNI为1，Linux内核会对它进行拆包，拿到内部数据桢，根据VNI的值，所它交给Node2的flannel.1设备

#### host-gw模式

Flannel 第三种协议叫 host-gw (host gateway)，这是一种纯三层网络的方案，性能最高，即 Node 节点把自己的网络接口当做 pod 的网关使用，从而使不同节点上的 node 进行通信，这个性能比 VxLAN 高，因为它没有额外开销。不过他有个缺点， 就是各 node 节点必须在同一个网段中 。

![img](https://blog.yingchi.io/posts/2020/8/k8s-flannel/662544-20191022103224393-1472403318.png)

howt-gw 模式的工作原理，就是将每个Flannel子网的下一跳，设置成了该子网对应的宿主机的 IP 地址，也就是说，宿主机（host）充当了这条容器通信路径的“网关”（Gateway），这正是 host-gw 的含义 所有的子网和主机的信息，都保存在 Etcd 中，flanneld 只需要 watch 这些数据的变化 ，实时更新路由表就行了。 核心是IP包在封装成桢的时候，使用路由表的“下一跳”设置上的MAC地址，这样可以经过二层网络到达目的宿主机。

另外，如果两个 pod 所在节点在同一个网段中 ，可以让 VxLAN 也支持 host-gw 的功能， 即直接通过物理网卡的网关路由转发，而不用隧道 flannel 叠加，从而提高了 VxLAN 的性能，这种 flannel 的功能叫 directrouting。

### Flannel 通信过程描述

以 UDP 模式为例，跨主机容器间通信过程如下图所示：

![img](https://blog.yingchi.io/posts/2020/8/k8s-flannel/1178573-20191101152442306-2133215750.png)

上图是 Flannel 官网提供的在 UDP 模式下一个数据包经过封包、传输以及拆包的示意图，从这个图片中可以看出两台机器的 docker0 分别处于不同的段：10.1.20.1/24 和 10.1.15.1/24 ，如果从 Web App Frontend1 pod（10.1.15.2）去连接另一台主机上的 Backend Service2 pod（10.1.20.3），网络包从宿主机 192.168.0.100 发往 192.168.0.200，内层容器的数据包被封装到宿主机的 UDP 里面，并且在外层包装了宿主机的 IP 和 mac 地址。这就是一个经典的 overlay 网络，因为容器的 IP 是一个内部 IP，无法从跨宿主机通信，所以容器的网络互通，需要承载到宿主机的网络之上。

以 VxLAN 模式为例。

在源容器宿主机中的数据传递过程：

1）源容器向目标容器发送数据，数据首先发送给 docker0 网桥

在源容器内容查看路由信息：

```
$ kubectl exec -it -p {Podid} -c {ContainerId} -- ip route
```

2）docker0 网桥接受到数据后，将其转交给flannel.1虚拟网卡处理

docker0 收到数据包后，docker0 的内核栈处理程序会读取这个数据包的目标地址，根据目标地址将数据包发送给下一个路由节点： 查看源容器所在Node的路由信息：

```
$ ip route
```

3）flannel.1 接受到数据后，对数据进行封装，并发给宿主机的eth0

flannel.1收到数据后，flannelid会将数据包封装成二层以太包。 Ethernet Header的信息：

- From:{源容器flannel.1虚拟网卡的MAC地址}
- To:{目录容器flannel.1虚拟网卡的MAC地址}

4）对在flannel路由节点封装后的数据，进行再封装后，转发给目标容器Node的eth0；

由于目前的数据包只是vxlan tunnel上的数据包，因此还不能在物理网络上进行传输。因此，需要将上述数据包再次进行封装，才能源容器节点传输到目标容器节点，这项工作在由linux内核来完成。 Ethernet Header的信息：

- From:{源容器Node节点网卡的MAC地址}
- To:{目录容器Node节点网卡的MAC地址}

IP Header的信息：

- From:{源容器Node节点网卡的IP地址}
- To:{目录容器Node节点网卡的IP地址}

通过此次封装，就可以通过物理网络发送数据包。

在目标容器宿主机中的数据传递过程：

5）目标容器宿主机的eth0接收到数据后，对数据包进行拆封，并转发给flannel.1虚拟网卡；

6）flannel.1 虚拟网卡接受到数据，将数据发送给docker0网桥；

7）最后，数据到达目标容器，完成容器之间的数据通信。

## 参考

- https://blog.csdn.net/alauda_andy/article/details/80132922
- https://blog.csdn.net/liukuan73/article/details/78883847
- https://www.kubernetes.org.cn/6908.html
- https://www.cnblogs.com/chenqionghe/p/11718365.html
- https://zhuanlan.zhihu.com/p/105942115
- https://www.jianshu.com/p/165a256fb1da
- https://www.cnblogs.com/ssgeek/p/11492150.html
- https://www.cnblogs.com/sandshell/p/11777312.html