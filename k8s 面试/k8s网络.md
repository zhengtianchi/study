# 图解kubernetes网络通信原理



## 名词解释

1、网络的命名空间：Linux在网络栈中引入网络命名空间 (`network namespace`)，将独立的网络协议栈隔离到不同的命令空间中，彼此间无法通信；docker利用这一特性，实现不容器间的网络隔离。

2、Veth设备对：也叫虚拟网络接口对。Veth设备对的引入是为了实现在不同网络命名空间的通信。

3、Iptables/Netfilter：Netfilter负责在内核中执行各种挂接的规则(过滤、修改、丢弃等)，运行在内核 模式中；Iptables模式是在用户模式下运行的进程，负责协助维护内核中Netfilter的各种规则表；通过二者的配合来实现整个Linux网络协议栈中灵活的数据包处理机制。

4、网桥：网桥是一个二层网络设备,通过网桥可以将linux支持的不同的端口连接起来,并实现类似交换机那样的多对多的通信。

5、路由：Linux系统包含一个完整的路由功能，当IP层在处理数据发送或转发的时候，会使用路由表来决定发往哪里。



## 令人头大的网络模型

Kubernetes对集群内部的网络进行了重新抽象，以实现整个**集群网络扁平化**。我们可以理解网络模型时，可以完全抽离物理节点去理解，我们用图说话，先有基本印象。



![img](https://pic1.zhimg.com/80/v2-6568948daa704d1f3325a9edb1e6495c_720w.jpg)



其中，重点讲解以下几个关键抽象概念。

**一个Service**

Service是Kubernetes为为屏蔽这些后端实例（Pod）的动态变化和对多实例的负载均衡而引入的资源对象。Service通常与deployment绑定，定义了服务的访问入口地址，应用(Pod)可以通过这个入口地址访问其背后的一组由Pod副本组成的集群实例。Service与其后端Pod副本集群之间则是通过Label Selector来实现映射。

Service的类型(Type)决定了Service如何对外提供服务，根据类型不同，服务可以只在Kubernetes cluster中可见，也可以暴露到集群外部。Service有三种类型，ClusterIP，NodePort和LoadBalancer。具体的使用场景会在下文中进行阐述。

在测试环境查看：

```text
$ kubectl get svc --selector app=nginx
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
nginx ClusterIP 172.19.0.166 <none> 80/TCP 1m
$ kubectl describe svc nginx
Name: nginx
Namespace: default
Labels: app=nginx
Annotations: <none>
Selector: app=nginx
Type: ClusterIP
IP: 172.19.0.166
Port: <unset> 80/TCP
TargetPort: 80/TCP
Endpoints: 172.16.2.125:80,172.16.2.229:80
Session Affinity: None
Events: <none>
```

上述信息中该svc后端代理了2个Pod实例:172.16.2.125:80,172.16.2.229:80

**二个IP**

Kubernetes为描述其网络模型的IP对象，抽象出Cluster IP和Pod IP的概念。

PodIP是Kubernetes集群中每个Pod的IP地址。它是Docker Engine 根据docker0网桥的IP地址段进行分配的，是一个虚拟的二层网络。Kubernetes中Pod间能够彼此直接通讯，Pod里的容器访问另外一个Pod里的容器，是通过Pod IP所在进行通信。

Cluster IP仅作用于Service，其没有实体对象所对应，因此Cluster IP无法被ping通。它的作用是为Service后端的实例提供统一的访问入口。当访问ClusterIP时，请求将被转发到后端的实例上，默认是轮询方式。Cluster IP和Service一样由kube-proxy组件维护，其实现方式主要有两种，iptables和IPVS。在1.8版本后kubeproxy开始支持IPVS方式。在上例中，SVC的信息中包含了Cluster IP。

这里未列出nodeip概念，由于其本身是物理机的网卡IP。因此可理解为nodeip就是物理机IP。

**三个Port**

在Kubernetes中，涉及容器，Pod，Service，集群各等多个层级的对象间的通信，为在网络模型中区分各层级的通信端口，这里对Port进行了抽象。

**Port**

该Port非一般意义上的TCP/IP中的Port概念，它是特指Kubernetes中Service的port，是Service间的访问端口，例如Mysql的Service默认3306端口。它仅对进群内容器提供访问权限，而无法从集群外部通过该端口访问服务。

**nodePort**

nodePort为外部机器提供了访问集群内服务的方式。比如一个Web应用需要被其他用户访问，那么需要配置type=NodePort，而且配置nodePort=30001，那么其他机器就可以通过浏览器访问scheme://node:30001访问到该服务，例如[http://node:30001](https://link.zhihu.com/?target=http%3A//node%3A30001)。

**targetPort**

targetPort是容器的端口（最根本的端口入口），与制作容器时暴露的端口一致（DockerFile中EXPOSE），例如[http://docker.io](https://link.zhihu.com/?target=http%3A//docker.io)官方的nginx暴露的是80端口。

举一个例子来看如何配置Service的port：

```text
kind: Service
apiVersion: v1
metadata:
 name: mallh5-service
 namespace: abcdocker
spec:
 selector:
 app: mallh5web
 type: NodePort
 ports:
 - protocol: TCP
 port: 3017
 targetPort: 5003
 nodePort: 31122
```

这里举出了一个service的yaml，其部署在abcdocker的namespace中。这里配置了nodePort，因此其类型Type就是NodePort，注意大小写。若没有配置nodePort，那这里需要填写ClusterIP，即表示只支持集群内部服务访问。

## 集群内部通信

**单节点通信**

集群单节点内的通信，主要包括两种情况，**同一个pod内的多容器间通信**以及**同一节点不同pod间的通信**。由于不涉及跨节点访问，因此流量不会经过物理网卡进行转发。

通过查看路由表，也能窥见一二：

```text
root@node-1:/opt/bin# route -n
Kernel IP routing table
Destination Gateway Genmask Flags Metric Ref Use Iface
0.0.0.0 172.23.100.1 0.0.0.0 UG 0 0 0 eth0
10.1.0.0 0.0.0.0 255.255.0.0 U 0 0 0 flannel.1 #flannel 网络内跨节点的通信会交给 flannel.1 处理
10.1.1.0 0.0.0.0 255.255.255.0 U 0 0 0 docker0 #flannel 网络内节点内的通信会走 docker0
```

**1 Pod内通信**

如下图所示：



![img](https://pic3.zhimg.com/80/v2-1306eda5daf4673099d7c78f1d7e9542_720w.jpg)



这种情况下，**同一个pod内共享网络命名空间**，容器之间通过访问127.0.0.1:（端口）即可。图中的veth*即指veth对的一端（另一端未标注，但实际上是成对出现），该veth对是由Docker Daemon挂载在docker0网桥上，另一端添加到容器所属的网络命名空间，图上显示是容器中的eth0。

图中演示了bridge模式下的容器间通信。docker1向docker2发送请求，docker1，docker2均与docker0建立了veth对进行通讯。

当请求经过docker0时，由于容器和docker0同属于一个**子网**，因此请求经过docker2与docker0的veth*对，转发到docker2，该过程并未跨节点，因此**不经过eth0**。

**2 Pod间通信**

**同节点pod间通信**

由于Pod内共享网络命名空间（**由pause容器创建**），所以本质上也是同节点容器间的通信。同时，同一Node中Pod的默认路由都是docker0的地址，由于它们关联在同一个docker0网桥上，地址网段相同，所有它们之间应当是能直接通信的。来看看实际上这一过程如何实现。如上图，Pod1中容器1和容器2共享网络命名空间，因此对pod外的请求通过pod1和Docker0网桥的veth对（图中挂在eth0和ethx上）实现。



![img](https://pic1.zhimg.com/80/v2-725354d57fa229dc3eca8ce768092ca0_720w.jpg)



访问另一个pod内的容器，其请求的地址是PodIP而非容器的ip，实际上也是同一个子网间通信，直接经过veth对转发即可。

**跨节点通信**

**CNI：容器网络接口**

CNI 是一种标准，它旨在为容器平台提供网络的标准化。不同的容器平台（比如目前的 kubernetes、mesos 和 rkt）能够通过相同的接口调用不同的网络组件。

目前kubernetes支持的CNI组件种类很多，例如：bridge calico calico-ipam dhcp flannel host-local ipvlan loopback macvlan portmap ptp sample tuning vlan。在docker中，主流的跨主机通信方案主要有一下几种：

1）基于隧道的overlay网络：按隧道类型来说，不同的公司或者组织有不同的实现方案。docker原生的overlay网络就是基于vxlan隧道实现的。ovn则需要通过geneve或者stt隧道来实现的。flannel最新版本也开始默认基于vxlan实现overlay网络。

2）基于包封装的overlay网络：基于UDP封装等数据包包装方式，在docker集群上实现跨主机网络。典型实现方案有weave、flannel的早期版本。

3）基于三层实现SDN网络：基于三层协议和路由，直接在三层上实现跨主机网络，并且通过iptables实现网络的安全隔离。典型的方案为Project Calico。同时对不支持三层路由的环境，Project Calico还提供了基于IPIP封装的跨主机网络实现

**通信方式**



![img](https://pic1.zhimg.com/80/v2-133751547942f9a34dc066537bfd70fc_720w.jpg)



集群内跨节点通信涉及到不同的子网间通信，仅靠docker0无法实现，这里需要借助CNI网络插件来实现。图中展示了使用flannel实现跨节点通信的方式。

简单说来，flannel的用户态进程flanneld会为每个node节点创建一个flannel.1的网桥，根据etcd或apiserver的全局统一的集群信息为每个node分配全局唯一的网段，避免地址冲突。同时会为docker0和flannel.1创建veth对，docker0将报文丢给flannel.1,。

Flanneld维护了一份全局node的网络表，通过flannel.1接收到请求后，根据node表，将请求二次封装为UDP包，扔给eth0，由eth0出口进入物理网路发送给目的node。

在另一端以相反的流程。Flanneld解包并发往docker0，进而发往目的Pod中的容器。



![img](https://pic1.zhimg.com/v2-85df1d96ecc0bcfe64e2a3f5768f0a98_b.jpg)



## 外部访问集群

从集群外访问集群有多种方式，比如loadbalancer，Ingress，nodeport，nodeport和loadbalancer是service的两个基本类型，是将service直接对外暴露的方式，ingress则是提供了七层负载均衡，其基本原理将外部流量转发到内部的service，再转发到后端endpoints，在平时的使用中，我们可以依据具体的业务需求选用不同的方式。这里主要介绍nodeport和ingress方式。

**Nodeport**

通过将Service的类型设置为NodePort，就可以在Cluster中的主机上通过一个指定端口暴露服务。注意通过Cluster中每台主机上的该指定端口都可以访问到该服务，发送到该主机端口的请求会被kubernetes路由到提供服务的Pod上。采用这种服务类型，可以在kubernetes cluster网络外通过主机IP：端口的方式访问到服务。



![img](https://pic1.zhimg.com/80/v2-b03a0637da41dd17b49c0fea49541670_720w.jpg)



这里给出一个influxdb的例子，我们也可以针对这个模板去修改成其他的类型：

```text
kind: Service
apiVersion: v1
metadata:
 name: influxdb
spec:
 type: NodePort
 ports:
 - port: 8086
 nodePort: 31112
 selector:
 name: influxdb
```

**Ingress**



![img](https://pic2.zhimg.com/80/v2-afec30e35cbe8fc5ffe0712dd2a9d265_720w.jpg)



Ingress是推荐在生产环境使用的方式，它起到了七层负载均衡器和Http方向代理的作用，可以根据不同的url把入口流量分发到不同的后端Service。外部客户端只看到[http://foo.bar.com](https://link.zhihu.com/?target=http%3A//foo.bar.com)这个服务器，屏蔽了内部多个Service的实现方式。采用这种方式，简化了客户端的访问，并增加了后端实现和部署的灵活性，可以在不影响客户端的情况下对后端的服务部署进行调整。

其部署的yaml可以参考如下模板：

```text
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: test
 annotations:
 ingress.kubernetes.io/rewrite-target: /
spec:
 rules:
 - host: test.name.com
 http:
 paths:
 - path: /test
 backend:
 serviceName: service-1
 servicePort: 8118
 - path: /name
 backend:
 serviceName: service-2
 servicePort: 8228
```

这里我们定义了一个ingress模板，定义通过[http://test.name.com](https://link.zhihu.com/?target=http%3A//test.name.com)来访问服务，在虚拟主机[http://test.name.com](https://link.zhihu.com/?target=http%3A//test.name.com)下面定义了两个Path，其中/test被分发到后端服务s1，/name被分发到后端服务s2。

集群中可以定义多个ingress，来完成不同服务的转发，这里需要一个ingress controller来管理集群中的Ingress规则。Ingress Contronler 通过与 Kubernetes API 交互，动态的去感知集群中 Ingress 规则变化，然后读取它，按照自定义的规则，规则就是写明了哪个域名对应哪个service，生成一段 Nginx 配置，再写到 Nginx-ingress-control的 Pod 里，这个 Ingress Contronler 的pod里面运行着一个nginx服务，控制器会把生成的nginx配置写入/etc/nginx.conf文件中，然后 reload使用配置生效。

Kubernetes提供的Ingress Controller模板如下：

```text
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: test
 annotations:
 ingress.kubernetes.io/rewrite-target: /
spec:
 rules:
 - host: foo.bar.com
 http:
 paths:
 - path: /foo
 backend:
 serviceName: s1
 servicePort: 80
 - path: /bar
 backend:
 serviceName: s2
 servicePort: 80
```





## Netns 探秘 (吉利面试有问)



### Netns 究竟实现了什么

下面简单讲一下，Network Namespace 里面能网络实现的内核基础。狭义上来说 runC 容器技术是不依赖于任何硬件的，它的执行基础就是它的内核里面，进程的内核代表就是 task，它如果不需要隔离，那么用的是主机的空间（ namespace），并不需要特别设置的空间隔离数据结构（ nsproxy-namespace proxy）。



![img](https://static001.infoq.cn/resource/image/98/31/987f6348ef2de66eb3652bd41b2b4e31.png)



相反，如果一个独立的网络 proxy，或者 mount proxy，里面就要填上真正的私有数据。它可以看到的数据结构如上图所示。



从感官上来看一个隔离的网络空间，它会拥有自己的网卡或者说是网络设备。网卡可能是虚拟的，也可能是物理网卡，它会拥有自己的 IP 地址、IP 表和路由表、拥有自己的协议栈状态。这里面特指就是 TCP/Ip 协议栈，它会有自己的 status，会有自己的 iptables、ipvs。



从整个感官上来讲，这就相当于拥有了一个完全独立的网络，它与主机网络是隔离的。当然协议栈的代码还是公用的，只是数据结构不相同。



### Pod 与 Netns 的关系

![img](https://static001.infoq.cn/resource/image/4c/cc/4c3192878fa0d7cec71384a78fdfa1cc.png)



这张图可以清晰表明 pod 里 Netns 的关系，**每个 pod 都有着独立的网络空间**，pod net container 会共享这个网络空间。一般 K8s 会推荐选用 Loopback 接口，在 pod net container 之间进行通信，而所有的 container 通过 pod 的 IP 对外提供服务。另外对于宿主机上的 Root Netns，可以把它看做一个特殊的网络空间，只不过它的 Pid 是 1。



## 主流网络方案简介

### 典型的容器网络实现方案

接下来简单介绍一下典型的容器网络实现方案。容器网络方案可能是 K8s 里最为百花齐放的一个领域，它有着各种各样的实现。容器网络的复杂性，其实在于它需要跟底层 Iass 层的网络做协调、需要在性能跟 IP 分配的灵活性上做一些选择，这个方案是多种多样的。



![img](https://static001.infoq.cn/resource/image/e5/57/e50f7075efaed3af3f257b147138c957.png)



下面简单介绍几个比较主要的方案：分别是 Flannel、Calico、Canal ，最后是 WeaveNet，中间的大多数方案都是采用了跟 Calico 类似的策略路由的方法。



- **Flannel** 是一个比较大一统的方案，它提供了多种的网络 backend。不同的 backend 实现了不同的拓扑，它可以覆盖多种场景；
- **Calico** 主要是采用了策略路由，节点之间采用 BGP 的协议，去进行路由的同步。它的特点是功能比较丰富，尤其是对 Network Point 支持比较好，大家都知道 Calico 对底层网络的要求，一般是需要 mac 地址能够直通，不能跨二层域；
- 当然也有一些社区的同学会把 Flannel 的优点和 Calico 的优点做一些集成。我们称之为嫁接型的创新项目 **Cilium**；
- 最后讲一下 **WeaveNet**，如果大家在使用中需要对数据做一些加密，可以选择用 WeaveNet，它的动态方案可以实现比较好的加密。



### Flannel 方案

![img](https://static001.infoq.cn/resource/image/20/a4/20a734cfabb5d14d2d150efa03e870a4.png)



Flannel 方案是目前使用最为普遍的。如上图所示，可以看到一个典型的容器网方案。它首先要解决的是 container 的包如何到达 Host，这里采用的是加一个 Bridge 的方式。它的 backend 其实是独立的，也就是说这个包如何离开 Host，是采用哪种封装方式，还是不需要封装，都是可选择的。



现在来介绍三种主要的 backend：



- 一种是用户态的 udp，这种是最早期的实现；
- 然后是内核的 Vxlan，这两种都算是 overlay 的方案。Vxlan 的性能会比较好一点，但是它对内核的版本是有要求的，需要内核支持 Vxlan 的特性功能；
- 如果你的集群规模不够大，又处于同一个二层域，也可以选择采用 host-gw 的方式。这种方式的 backend 基本上是由一段广播路由规则来启动的，性能比较高。