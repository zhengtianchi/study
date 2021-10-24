**网络通讯方式**

了解了上面的基本概念后，我们考虑一下K8s集群中docker容器之间是如何通讯的?我们这里需要区分一下不同的场景

1)在同一个POD上Container通信

2)同一个Node,不同POD

3)不同Node，不同POD

我们先来看看上面的不同场景是怎么通信的

**同一个POD上Container通信**

在k8s中每个Pod中管理着一组Docker容器，这些Docker容器共享同一个网络命名空间，Pod中的每个Docker容器拥有与Pod相同的IP和port地址空间，并且由于他们在同一个网络命名空间，他们之间可以通过localhost相互访问。

什么机制让同一个Pod内的多个docker容器相互通信?就是使用Docker的一种网络模型：–net=container

**container模式指定新创建的Docker容器和已经存在的一个容器共享一个网络命名空间**，而不是和宿主机共享。新创建的Docker容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等

在k8s中每个Pod容器有一个pause容器有独立的网络命名空间，在Pod内启动Docker容器时候使用 –net=container就可以让当前Docker容器加入到Pod容器拥有的网络命名空间(pause容器)

![Kubernetes之POD、容器之间的网络通信](https://s3.51cto.com/oss/202007/06/bdd6a008d6a7aa9d5502c39fc22a79f6.jpg)

这里就是为什么k8s在调度pod时，尽量把关系紧密的服务放到一个pod中，这样网络的请求耗时就可以忽略，因为容器之间通信共享了网络空间，就像local本地通信一样。

**同一个Node，不同Pod**

![Kubernetes之POD、容器之间的网络通信](https://s4.51cto.com/oss/202007/06/e5478bd23553e02d93c43d6bdafad6a9.jpg)

![Kubernetes之POD、容器之间的网络通信](https://s5.51cto.com/oss/202007/06/1db60ff9acbf3b016b771fad1621ae3c.jpg)

上图就是同一个node，不同pod之间的通信，就是使用linux虚拟以太网设备或者说是由两个虚拟接口组成的veth对使不同的网络命名空间链接起来，这些虚拟接口分布在多个网络命名空间上(这里是指多个Pod上)。

通过网桥把veth0和veth1组成为一个以太网，他们直接是可以直接通信的，另外这里通过veth对让pod1的eth0和veth0、pod2的eth0和veth1关联起来，从而让pod1和pod2相互通信。

**不同Node，不同Pod**

![Kubernetes之POD、容器之间的网络通信](https://s3.51cto.com/oss/202007/06/36c7aab05b6f154972939a655e5d49c4.jpg)

上图就是不同node之间的pod通信，Node1中的Pod1如何和Node2的Pod4进行通信的，我们来看看具体流程：

1)首先pod1通过自己的以太网设备eth0把数据包发送到关联到root命名空间的veth0上

2)然后数据包被Node1上的网桥设备接受到，网桥查找转发表发现找不到pod4的Mac地址，则会把包转发到默认路由(root命名空间的eth0设备)

3)然后数据包经过eth0就离开了Node1，被发送到网络。

4)数据包到达Node2后，首先会被root命名空间的eth0设备

5)然后通过网桥把数据路由到虚拟设备veth1,最终数据表会被流转到与veth1配对的另外一端(pod4的eth0)

每个Node都知道如何把数据包转发到其内部运行的Pod，当一个数据包到达Node后，其内部数据流就和Node内Pod之间的流转类似了

补充说明：对于如何来配置网络，k8s在网络这块自身并没有实现网络规划的具体逻辑，而是制定了一套CNI(Container Network Interface)接口规范，开放给社区来实现。Flannel就是k8s中比较出名的一个。

**flannel**

flannel组建一个大二层扁平网络，pod的ip分配由flannel统一分配，通讯过程也是走flannel的网桥。

每个node上面都会创建一个flannel0虚拟网卡，用于跨node之间通讯。所以容器直接可以直接使用pod id进行通讯。

跨节点通讯时，发送端数据会从docker0路由到flannel0虚拟网卡，接收端数据会从flannel0路由到docker0。

**总结**

上面老顾介绍了几种网络通信的场景，以及他们的通信流程，k8s的网络通信远远不止这些，还有很重要的集群外如何访问集群内部?以及Service访问是用来做什么的?