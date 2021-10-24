# Kubernetes Controller Manager 工作原理

在 Kubernetes Master 节点中，有三个重要组件：ApiServer、ControllerManager、Scheduler，它们一起承担了整个集群的管理工作。本文尝试梳理清楚 ControllerManager 的工作流程和原理。

![img](https://ask.qcloudimg.com/http-save/yehe-1952081/vdq7h9ukkv.jpeg?imageView2/2/w/1620)

## 什么是 Controller Manager

根据官方文档的说法：kube-controller-manager 运行控制器，它们是处理集群中常规任务的后台线程。

说白了，Controller Manager 就是集群内部的管理控制中心，由负责不同资源的多个 Controller 构成，共同负责集群内的 Node、Pod 等所有资源的管理，比如当通过 Deployment 创建的某个 Pod 发生异常退出时，RS Controller 便会接受并处理该退出事件，并创建新的 Pod 来维持预期副本数。

几乎每种特定资源都有特定的 Controller 维护管理以保持预期状态，而 Controller Manager 的职责便是把所有的 Controller 聚合起来：

1. 提供基础设施降低 Controller 的实现复杂度
2. 启动和维持 Controller 的正常运行

可以这么说，**Controller 保证集群内的资源保持预期状态，而 Controller Manager 保证了 Controller 保持在预期状态**。



## 1. 前言

在`K8S`内部通信中，肯定要保证消息的实时性。之前以为方式有两种：

1. 客户端组件(`kubelet`,`scheduler`,`controller-manager`等)轮询 apiserver，
2. `apiserver`通知客户端。
   如果采用`轮询`，势必会大大增加`apiserver`的压力，同时实时性很低。
   如果`apiserver`主动发`HTTP`请求，又如何保证消息的可靠性，以及大量端口占用问题？

当阅读完`list-watch`源码后，先是所有的疑惑云开雾散，进而为`K8S`的`设计理念`所折服。`List-watch`是`K8S`统一的异步消息处理机制，保证了消息的实时性，可靠性，顺序性，性能等等，为声明式风格的`API`奠定了良好的基础，它是优雅的通信方式，是`K8S 架构`的精髓。

## 2. List-Watch 机制具体是什么样的

`Etcd`存储集群的数据信息，`apiserver`作为统一入口，任何对数据的操作都必须经过`apiserver`。客户端(`kubelet`/`scheduler`/`controller-manager`)通过`list-watch`监听`apiserver`中资源(`pod/rs/rc`等等)的`create`,`update`和`delete`事件，并针对`事件类型`调用相应的`事件处理函数`。

那么`list-watch`具体是什么呢，顾名思义，`list-watch`有两部分组成，分别是`list`和`watch`。

- `list`非常好理解，就是调用资源的`list API`罗列资源，基于`HTTP`短链接实现；
- `watch`则是调用资源的`watch API`监听资源变更事件，基于`HTTP 长链接`实现，也是本文重点分析的对象。

以`pod 资源`为例，它的`list`和`watch API`分别为：

 [List API](https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#list-all-namespaces-63)，返回值为 [PodList](https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#podlist-v1-core)，即一组`pod`。

> GET /api/v1/pods

[Watch API](https://link.zhihu.com/?target=https%3A//v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/%23watch-list-all-namespaces-66)，往往带上`watch=true`，表示采用`HTTP 长连接`持续监听`pod 相关事件`，每当有事件来临，返回一个[WatchEvent](https://link.zhihu.com/?target=https%3A//v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/%23watchevent-v1-meta)。

> GET /api/v1/watch/pods

**`K8S`的`informer`模块封装`list-watch API`，用户只需要指定资源，编写事件处理函数**，`AddFunc`,`UpdateFunc`和`DeleteFunc`等。如下图所示，`informer`首先通过`list API`罗列资源，然后调用`watch API`监听资源的变更事件，并将结果放入到一个`FIFO 队列`，队列的另一头有协程从中取出事件，并调用对应的注册函数处理事件。`Informer`还维护了一个只读的`Map Store`缓存，主要为了提升查询的效率，降低`apiserver`的负载。



![img](https://pic4.zhimg.com/80/v2-5263be53b45fdbea59e9812d9df627b7_720w.jpg)

## 3.Watch 是如何实现的

`List`的实现容易理解，那么`Watch`是如何实现的呢？`Watch`是如何通过`HTTP 长链接`接收`apiserver`发来的`资源变更事件`呢？

秘诀就是[Chunked transfer encoding(分块传输编码)](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/zh-hans/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81)，它首次出现在`HTTP/1.1`。正如维基百科所说：

> HTTP 分块传输编码允许服务器为动态生成的内容维持 HTTP 持久链接。通常，持久链接需要服务器在开始发送消息体前发送Content-Length消息头字段，但是对于动态生成的内容来说，在内容创建完之前是不可知的。使用分块传输编码，数据分解成一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小。

当客户端调用`watch API`时，`apiserve`r 在`response`的`HTTP Header`中设置`Transfer-Encoding`的值为`chunked`，表示采用`分块传输`编码，客户端收到该信息后，便和服务端该链接，并等待下一个数据块，即资源的事件信息。例如：

```text
$ curl -i http://{kube-api-server-ip}:8080/api/v1/watch/pods?watch=yes
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 02 Jan 2019 20:22:59 GMT
Transfer-Encoding: chunked

{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"MODIFIED", "object":{"kind":"Pod","apiVersion":"v1",...}}
```

## 4. 谈谈 List-Watch 的设计理念

当设计优秀的一个异步消息的系统时，对`消息机制`有至少如下四点要求：

- 消息可靠性
- 消息实时性
- 消息顺序性
- 高性能

首先`消息`必须是`可靠`的，`list`和`watch`一起保证了消息的可靠性，避免因消息丢失而造成状态不一致场景。具体而言，`list API`可以查询当前的资源及其对应的状态(即期望的状态)，客户端通过拿`期望的状态`和`实际的状态`进行对比，纠正状态不一致的资源。`Watch API`和`apiserver`保持一个`长链接`，接收资源的`状态变更事件`并做相应处理。如果仅调用`watch API`，若某个时间点连接中断，就有可能导致消息丢失，所以需要通过`list API`解决`消息丢失`的问题。从另一个角度出发，我们可以认为`list API`获取全量数据，`watch API`获取增量数据。虽然仅仅通过轮询`list API`，也能达到同步资源状态的效果，但是存在开销大，实时性不足的问题。

消息必须是实时的，`list-watch`机制下，每当`apiserver`的资源产生`状态变更事件`，都会将事件及时的推送给客户端，从而保证了`消息的实时性`。

消息的顺序性也是非常重要的，在并发的场景下，客户端在短时间内可能会收到同一个资源的多个事件，对于`关注最终一致性`的`K8S`来说，它需要知道哪个是最近发生的事件，并保证资源的最终状态如同最近事件所表述的状态一样。`K8S`在每个资源的事件中都带一个`resourceVersion`的标签，这个标签是递增的数字，所以当客户端并发处理同一个资源的事件时，它就可以对比`resourceVersion`来保证最终的状态和最新的事件所期望的状态保持一致。

`List-watch`还具有高性能的特点，虽然仅通过周期性调用`list API`也能达到资源最终一致性的效果，但是周期性频繁的轮询大大的增大了开销，增加`apiserver`的压力。而`watch`作为异步消息通知机制，复用一条长链接，保证实时性的同时也保证了性能。

## 5. Informer介绍

`Informer`是`Client-go`中的一个核心工具包。在`Kubernetes`源码中，如果`Kubernetes`的某个组件，需要`List/Get Kubernetes`中的`Object`，在绝大多 数情况下，会直接使用`Informer`实例中的`Lister()`方法（该方法包含 了 Get 和 List 方法），而很少直接请求`Kubernetes API`。`Informer`最基本 的功能就是`List/Get Kubernetes`中的`Object`。

如下图所示，仅需要十行左右的代码就能实现对`Pod`的`List`和`Get`。



![img](https://pic3.zhimg.com/80/v2-7dd870bf025344f466e54f2be8e59d2e_720w.jpg)



## 6. Informer 设计思路

### 6.1 Informer 设计中的关键点

为了让`Client-go`更快地返回`List/Get`请求的结果、减少对`Kubenetes API`的直接调用，`Informer`被设计实现为一个依赖`Kubernetes List/Watch API`、`可监听事件并触发回调函数`的`二级缓存`工具包。

### 6.2 更快地返回 List/Get 请求，减少对 Kubenetes API 的直接调用

使用`Informer`实例的`Lister()`方法，`List/Get Kubernetes`中的`Object`时，**`Informer`不会去请求`Kubernetes API`**，**而是直接查找`缓存`在本地内存中的数据(这份数据由`Informer`自己维护)。**通过这种方式，`Informer`既可以更快地返回结果，又能减少对`Kubernetes API`的直接调用。

### 6.3 依赖 Kubernetes List/Watch API

`Informer`只会调用`Kubernetes List`和`Watch`两种类型的`API`。`Informer`在初始化的时:

- 先调用`Kubernetes List API`获得某种`resource`的全部`Object`，缓存在`内存`中; 
- 然后，调用`Watch API`去`watch`这种`resource`，去维护这份缓存;
-  最后，`Informer`就不再调用`Kubernetes`的任何 API。

用`List/Watch`去维护缓存、保持一致性是非常典型的做法，但令人费解的是，**`Informer`只在初始化时调用一次`List API`，之后完全依赖`Watch API`去维护缓存，**没有任何`resync`机制。

笔者在阅读`Informer`代码时候，对这种做法十分不解。按照多数人思路，通过`resync`机制，重新`List`一遍`resource`下的所有`Object`，可以更好的保证`Informer 缓存`和`Kubernetes`中数据的一致性。

咨询过`Google`内部`Kubernetes`开发人员之后，得到的回复是:

在`Informer`设计之初，确实存在一个`relist`无法去执`resync`操作， 但后来被取消了。原因是现有的这种`List/Watch`机制，完全能够保证永远不会漏掉任何事件，因此完全没有必要再添加`relist`方法去`resync informer`的缓存。这种做法也说明了`Kubernetes`完全信任`etcd`。

### 6.4 可监听事件并触发回调函数

**`Informer`通过`Kubernetes Watch API`监听某种`resource`下的所有事件**。而且，`Informer`可以添加自定义的回调函数，这个回调函数实例(即`ResourceEventHandler`实例)只需实现`OnAdd`(obj interface{})`OnUpdate`(oldObj, newObj interface{}) 和`OnDelete`(obj interface{}) 三个方法，这三个方法分别对应`informer`监听到`创建`、`更新`和`删除`这三种事件类型。

在`Controller`的设计实现中，会经常用到`informer`的这个功能。

### 6.5 二级缓存

二级缓存属于`Informer`的底层缓存机制，这两级缓存分别是`DeltaFIFO`和`LocalStore`。

这两级缓存的用途各不相同。

- `DeltaFIFO`用来存储`Watch API`返回的各种事件 
- `LocalStore`只会被`Lister`的`List/Get`方法访问 。

虽然`Informer`和`Kubernetes`之间没有`resync`机制，但`Informer`内部的这两级缓存之间存在`resync`机制。

### 6.6 关键逻辑介绍



![img](https://pic4.zhimg.com/80/v2-1a732ac6c3790caf0021038983ee7f2f_720w.jpg)





![img](https://pic4.zhimg.com/80/v2-1a732ac6c3790caf0021038983ee7f2f_720w.jpg)





![img](https://pic4.zhimg.com/80/v2-ab150eedead69d2fb6cae5d95b60ac93_720w.jpg)



1. Informer 在**初始化**时，Reflector 会先 **List API** 获得所有的 Pod
2. Reflect 拿到全部 Pod 后，会将全部 Pod 放到 **Store (缓存)** 中
3. 如果有人调用 Lister 的 List/Get 方法获取 Pod， 那么 Lister 会直接从 Store 中拿数据
4. Informer 初始化完成之后，Reflector 开始 **Watch Pod**，监听 Pod 相关 的所有事件;如果此时 pod_1 被删除，那么 Reflector 会监听到这个事件
5. Reflector 将 pod_1 被删除 的这个事件 **event** 发送到 **DeltaFIFO (workqueue)**
6. DeltaFIFO 首先会将这个事件存储在自己的数据结构中(实际上是一个 queue)，然后会直接操作 Store 中的数据，删除 Store 中的 pod_1
7. DeltaFIFO 再 Pop 这个事件到 Controller 中
8. Controller 收到这个事件，会触发 Processor 的回调函数
9. LocalStore 会周期性地把所有的 Pod 信息重新放到 DeltaFIFO 中



```
每个控制器内部都有两个核心组件：Informer/SharedInformer 和 Workqueue。
其中 Informer/SharedInformer 负责 watch Kubernetes 资源对象的状态变化，然后将相关事件（evenets）发送到 Workqueue 中.
最后再由控制器的 worker 从 Workqueue 中取出事件交给控制器处理程序进行处理。
```

**事件** = **动作**（create, update 或 delete） + **资源的 key**（以 `namespace/name` 的形式表示）



## Controller Manager & Controller 工作原理概述

我们都知道 kubernetes 中管理资源的方式比较简单，通常就是写一个 YAML 清单，简单的可以通过 kubectl 命令直接解决，在这个过程中，我们定义了某个资源的「期望状态」，比如 YAML 清单文件中的 spec 字段，Deployment 的 YAML 中的 spec 字段可能定义了期望的 replicas，我们期望集群的某个 pod 的副本数维持在某个数量上，当我们提交清单给集群，kubernetes 会在一段时间内将集群中的某些资源调整至我们期望的状态；亦或是另一个场景，集群中某个 Pod 挂掉了，或者我们将 Pod 从某个 Worker Node 上驱逐了，然后我们没有做任何操作，Pod 又会自动重建，并且达到指定的副本数，这是很常见的场景。

上面说的这些资源的状态管理是由谁实现的呢？没错，就是 Controller Manager，Controller Manager 是 Kubernetes 的灵魂组件之一，可以说通过定义资源期望状态实现集群资源编排管理的思想其底层就是依赖 Controller Manager 这个组件。

Controller Manager 的作用简而言之：**保证集群中各种资源的实际状态（status）和用户定义的期望状态（spec）一致。**

按照官方定义：kube-controller-manager 运行控制器，它们是处理集群中常规任务的后台线程。

Controller Manager 就是集群内部的管理控制中心，刚才说 Controller Manager 的作用是保证集群中各种资源的实际状态和用户定义的期望状态一致，但是如果出现不一致的情况怎么办？是由 Controller Manager 自己来对各种资源进行调整吗？

这时候就要说到 Controller 的概念了，之所以叫 Controller Manager，是因为 Controller Manager 由负责不同资源的多个 Controller 构成，如 Deployment Controller、Node Controller、Namespace Controller、Service Controller 等，这些 Controllers 各自明确分工负责集群内资源的管理。

![image-20200725172912964](https://blog.yingchi.io/posts/2020/7/k8s-cm-informer/image-20200725172912964.png)

如图，Controller Manager 发现资源的实际状态和期望状态有偏差之后，会触发相应 Controller 注册的 Event Handler，让它们去根据资源本身的特点进行调整。

比如当通过 Deployment 创建的某个 Pod 发生异常退出时，Deployment Controller 便会接受并处理该退出的 Event，并创建新的 Pod 来维持期望副本数。这样设计的原因也很好理解，可以将 Controller Manager 与具体的状态管理工作相解耦，因为不同的资源对于状态的管理多种多样，Deployment Controller 关注 Pod 副本数，而 Service 则关注 Service 的 IP、Port 等等。

## client-go

Controller Manager 中一个很关键的部分就是 client-go，client-go 在 controller manager 中起到向 controllers 进行事件分发的作用。目前 client-go 已经被单独抽取出来成为一个项目了，除了在 kubernetes 中经常被用到，在 kubernetes 的二次开发过程中会经常用到 client-go，比如可以通过 client-go 开发自定义 controller。

client-go 包中一个非常核心的工具就是 informer，informer 可以让与 kube-apiserver 的交互更加优雅。

informer 主要功能可以概括为两点：

- 资源数据缓存功能，缓解对 kube-apiserver 的访问压力；
- 资源事件分发，触发事先注册好的 ResourceEventHandler；

Informer 另外一块内容在于提供了事件 handler 机制，并会触发回调，这样 Controller 就可以基于回调处理具体业务逻辑。因为 Informer 通过 List、Watch 机制可以监控到所有资源的所有事件，因此只要给 Informer 添加ResourceEventHandler 实例的回调函数实例取实现 `OnAdd(obj interface{})`、 `OnUpdate(oldObj, newObj interface{}) ` 和 `OnDelete(obj interface{})` 这三个方法，就可以处理好资源的创建、更新和删除操作

### client-go 工作机制

![img](https://blog.yingchi.io/posts/2020/7/k8s-cm-informer/client-go-controller-interaction.jpeg)

上图是官方给出的 client-go 与自定义 controller 的实现原理。

#### Reflactor

反射器，具有以下几个功能：

- 采用 List、Watch 机制与 kube-apiserver 交互，List 短连接获取全量数据，Watch 长连接获取增量数据；
- 可以 Watch 任何资源包括 CRD；
- Watch 到的增量 Object 添加到 Delta FIFO 队列，然后 Informer 会从队列里面取数据;

#### Informer

Informer 是 client-go 中较为核心的一个模块，其主要作用包括如下两个方面：

- 同步数据到本地缓存。Informer 会不断读取 Delta FIFO 队列中的 Object，在触发事件回调之前先更新本地的 store，如果是新增 Object，如果事件类型是 Added（添加对象），那么 Informer 会通过 Indexer 的库把这个增量里的 API 对象保存到本地的缓存中，并为它创建索引。之后通过 Lister 对资源进行 List / Get 操作时会直接读取本地的 store 缓存，通过这种方式避免对 kube-apiserver 的大量不必要请求，缓解其访问压力；
- 根据对应的事件类型，触发事先注册好的 ResourceEventHandler。client-go 的 informer 模块启动时会创建一个 shardProcessor，各种 controller（如 Deployment Controller、自定义 Controller…）的事件 handler 注册到 informer 的时候会转换为一个 processorListener 实例，然后 processorListener 会被 append 到 shardProcessor 的 Listeners 切片中，shardProcessor 会管理这些 listeners。

processorListener 的重要作用就是当事件到来时触发对应的处理方法，因此不停地从 nextCh 中拿到事件并执行对应的 handler。`sharedProcessor` 的职责便是管理所有的 Handler 以及分发事件，而真正做分发工作的是 `distribute` 方法。

梳理一下这中间的过程：

1. Controller 将 Handler 注册给 Informer；
2. Informer 通过 `sharedProcessor` 维护了所有转换为 processorListener 的 Handler；
3. Informer 收到事件时，通过 `sharedProcessor.distribute` 将事件分发下去；
4. Controller 被触发对应的 Handler 来处理自己的逻辑。

![img](https://blog.yingchi.io/posts/2020/7/k8s-cm-informer/2019-12-15-eventdistribute.jpg)

Reflactor 启动后会执行一个 processLoop 死循环，Loop 中不停地将 Delta FIFO 队列中的事件 Pop 出来，Pop 时会取出该资源的所有事件，并交给 `sharedIndexInformer` 的 `HandleDeltas` 方法（创建 `controller` 时赋值给了 `config.Process`，传递到 Pop 参数的处理函数中 `Pop(PopProcessFunc(c.config.Process))`），`HandleDeltas` 调用了 `processor.distribute` 完成事件的分发。

在注册的 ResourceEventHandler 回调函数中，只是做了一些很简单的过滤，然后将关心变更的 Object 放到 workqueue 里面。之后 Controller 从 workqueue 里面取出 Object，启动一个 worker 来执行自己的业务逻辑，通常是对比资源的当前运行状态与期望状态，做出相应的处理，实现运行状态向期望状态的收敛。

注意，在 worker 中就可以使用 lister 来获取 resource，这个时候不需要频繁的访问 kube-apiserver 了，对于资源的 List / Get 会直接访问 informer 本地 store 缓存，apiserver 中资源的的变更都会反映到这个缓存之中。同时，LocalStore 会周期性地把所有的 Pod 信息重新放到 DeltaFIFO 中。