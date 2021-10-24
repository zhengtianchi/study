# [Docker底层基石namespace与cgroup](https://www.cnblogs.com/janeysj/p/11274515.html)

 

![img](https://qiankunli.github.io/public/upload/linux/docker_theory.jpg)

两个基本点

1. 数据结构：namespace 和 cgroups 数据在内核中如何组织
2. 算法：内核如何应用namespace 和 cgroups 数据



# 简介



容器本质上是把系统中为同一个业务目标服务的相关进程合成一组，**放在一个叫做namespace的空间中，同一个namespace中的进程能够互相通信，但看不见其他namespace中的进程**。每个namespace可以拥有自己独立的主机名、进程ID系统、IPC、网络、文件系统、用户等等资源。在某种程度上，实现了一个简单的虚拟：让一个主机上可以同时运行多个互不感知的系统。

![Alt text](https://img2018.cnblogs.com/blog/974353/201908/974353-20190829171849502-300506699.png)

此外，**为了限制namespace对物理资源的使用**，对进程能使用的CPU、内存等资源需要做一定的限制。这就是Cgroup技术，Cgroup是Control group的意思。比如我们常说的4c4g的容器，实际上是限制这个容器namespace中所用的进程，最多能够使用4核的计算资源和4GB的内存。
简而言之，**Linux内核提供namespace完成隔离，Cgroup完成资源限制**。namespace+Cgroup构成了容器的底层技术（rootfs是容器文件系统层技术）。

### namespace

一个namespace把一些全局系统资源封装成一个抽象体，该抽象体对于本namespace中的进程来说有它们自己的隔离的全局资源实例。改变这些全局资源对于该namespace中的进程是可见的，而对其他进程来说是不可见的。
Linux 提供一下几种 namespaces:

```
  Namespace   Constant                           Isolates
  -  IPC            CLONE_NEWIPC            System V IPC, POSIX message queues
  -  Network     CLONE_NEWNET           Network devices, stacks, ports, etc.
  -  Mount        CLONE_NEWNS             Mount points
  -  PID            CLONE_NEWPID            Process IDs
  -  User          CLONE_NEWUSER         User and group IDs
  -  UTS          CLONE_NEWUTS           Hostname and NIS domain name
  
```

![img](https://img-blog.csdnimg.cn/20201012133948571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JvbmdfVG9h,size_16,color_FFFFFF,t_70)

为了在分布式的环境下进行通信和定位，容器必然需要一个独立的IP、端口、路由等等，自然就想到了网络的隔离。同时，你的容器还需要一个独立的主机名以便在网络中标识自己。想到网络，顺其自然就想到通信，也就想到了进程间通信的隔离。可能你也想到了权限的问题，对用户和用户组的隔离就实现了用户权限的隔离。最后，运行在容器中的应用需要有自己的PID,自然也需要与宿主机中的PID进行隔离。

### cgroups

Cgroups是control groups的缩写，最初由google的工程师提出，后来被整合进Linux内核。Cgroups是Linux内核提供的一种可以限制、记录、隔离进程组（process groups）所使用的物理资源（如：CPU、内存、IO等）的机制。对开发者来说，cgroups 有如下四个有趣的特点：

- cgroups 的 API 以一个伪文件系统的方式实现，即用户可以通过文件操作实现 cgroups 的组织管理。
- cgroups 的组织管理操作单元可以细粒度到线程级别，用户态代码也可以针对系统分配的资源创建和销毁 cgroups，从而实现资源再分配和管理。
- 所有资源管理的功能都以“subsystem（子系统）”的方式实现，接口统一。
- 子进程创建之初与其父进程处于同一个 cgroups 的控制组。

本质上来说，cgroups 是内核附加在程序上的一系列钩子（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。实现 cgroups 的主要目的是为不同用户层面的资源管理，提供一个统一化的接口。从单个进程的资源控制到操作系统层面的虚拟化。Cgroups 提供了以下四大功能:

- 资源限制（Resource Limitation）：cgroups 可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出 OOM（Out of Memory）。
- 优先级分配（Prioritization）：通过分配的 CPU 时间片数量及硬盘 IO 带宽大小，实际上就相当于控制了进程运行的优先级。
- 资源统计（Accounting）： cgroups 可以统计系统的资源使用量，如 CPU 使用时长、内存用量等等，这个功能非常适用于计费。
- 进程控制（Control）：cgroups 可以对进程组执行挂起、恢复等操作。
  Docker正是使用cgroup进行资源划分，每个容器都作为一个进程运行起来，每个业务容器都会有一个基础的pause容器也就是POD作为基础容器。pause容器提供了划分namespace的内容，并连通同一POD下的所有容器，共享网络资源。查看容器的PID，对应/proc/pid/下是该容器的运行资源。

![img](https://img-blog.csdnimg.cn/20201012134017849.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JvbmdfVG9h,size_16,color_FFFFFF,t_70)



## namespace

《深入剖析kubernetes》：用户运行在容器里的应用进程，跟宿主机上的其他进程一样，**都由宿主机操作系统统一管理**，只不过这些被隔离的进程拥有额外设置过的Namespace 参数。而docker 在这里扮演的角色，更多的是旁路式的辅助和管理工作。

linux内核对命名空间的支持完全隔离了工作环境中应用程序的视野。

### 来源

命名空间最初是用来解决命名唯一性问题的，即解决不同编码人员编写的代码模块在合并时可能出现的重名问题。

传统上，在Linux以及其他衍生的UNIX变体中，许多资源是全局管理的。这意味着进程之间彼此可能相互影响。偏偏有这样一些场景，比如一场“黑客马拉松”的比赛，组织者需要运行参赛者提供的代码，为了防止一些恶意的程序，必然要提供一套隔离的环境，一些提供在线持续集成服务的网站也有类似的需求。

我们不想让进程之间相互影响，就必须将它们隔离起来，最好都不知道对方的存在。而所谓的隔离，便是隔离他们使用的资源（比如），进而资源的管理也不在是全局的了。

### namespace 内核数据结构

[Namespaces in operation, part 1: namespaces overview](https://lwn.net/Articles/531114/) 是一个介绍 namespace 的系列文章，要点如下：

1. The purpose of each namespace is to wrap a particular global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. 对global system resource的封装
2. there will probably be further extensions to existing namespaces, such as the addition of namespace isolation for the kernel log. 将会有越来越多的namespace

namespace 简单说，就是进程的task_struct 以前都直接 引用资源id（各种资源或者是index，或者 是一个地址），现在是进程 task struct ==> nsproxy ==> 资源表(操作系统就是提供抽象，并将各种抽象封装为数据结构，外界可以引用)

[Linux内核的namespace机制分析](https://blog.csdn.net/xinxing__8185/article/details/51868685)

```
struct task_struct {	
    ……..		
    /* namespaces */		
    struct nsproxy *nsproxy;	
    …….
}
struct nsproxy {
        atomic_t count;	// nsproxy可以共享使用，count字段是该结构的引用计数
        struct uts_namespace *uts_ns;
        struct ipc_namespace *ipc_ns;
        struct mnt_namespace *mnt_ns;
        struct pid_namespace *pid_ns_for_children;
        struct net             *net_ns;
};
```

[What is the relation between `task_struct` and `pid_namespace`?](https://stackoverflow.com/questions/26779416/what-is-the-relation-between-task-struct-and-pid-namespace)

[Separation Anxiety: A Tutorial for Isolating Your System with Linux Namespaces](https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces) 该文章 用图的方式，解释了各个namespace 生效的机理，值得一读。其实要理解的比较通透，首先就得对 linux 进程、文件、网络这块了解的比较通透。**此外，虽说都是隔离，但他们隔离的方式不一样，比如root namespace是否可见，隔离的资源多少（比如pid只隔离了pid，mnt则隔离了root directory 和 挂载点，network 则隔离网卡、路由表、端口等所有网络资源），隔离后跨namespace如何交互**

### namespace 生效机制

![img](https://qiankunli.github.io/public/upload/linux/linux_namespace_object.png)

### pid namespace

进程是树结构的，每个namespace 理解的 根不一样，pid root namespace 最终提供完整视图

![img](https://qiankunli.github.io/public/upload/linux/pid_namespace.png)

### mount namespace

mount 也是有树的，每个namespace 理解的根 不一样, 挂载点目录彼此看不到. task_struct ==> nsproxy 包括 mnt_namespace。

```
struct mnt_namespace {
    atomic_t		count;
    struct vfsmount *	root;///当前namespace下的根文件系统
    struct list_head	list; ///当前namespace下的文件系统链表（vfsmount list）
    wait_queue_head_t poll;
    int event;
};
struct vfsmount {
    ...
    struct dentry *mnt_mountpoint;	/* dentry of mountpoint,挂载点目录 */
    struct dentry *mnt_root;	/* root of the mounted tree,文件系统根目录 */
    ...
}
```

只是单纯一个隔离的 mnt namespace 环境是不够的，还要”change root”，参见《自己动手写docker》P45

《阿里巴巴云原生实践15讲》chroot 的作用是“重定向进程及其子进程的根目录到一个文件系统 上的新位置”，使得该进程再也**看不到也没法接触到这个位置上层的“世界”**。所以这 个被隔离出来的新环境就有一个非常形象的名字，叫做 Chroot Jail。

### network namespace

network namespace 倒是没有根， 但docker 创建 veth pair，root namespace 一个，child namespace 一个。此外 为 root namespace 额外加 iptables 和 路由规则，为 各个ethxx 提供路由和数据转发，并提供跨network namesapce 通信。

## cgroups

### V1 和 V2

![img](https://qiankunli.github.io/public/upload/container/cgroup_v1.jpeg)

Cgroup v1 的一个整体结构，每一个子系统都是独立的，资源的限制只能在子系统中发生。比如pid可以分别属于 memory Cgroup 和 blkio Cgroup。但是在 blkio Cgroup 对进程 pid 做磁盘 I/O 做限制的时候，blkio 子系统是不会去关心 pid 用了哪些内存，这些内存是不是属于 Page Cache，而这些 Page Cache 的页面在刷入磁盘的时候，产生的 I/O 也不会被计算到进程 pid 上面。**Cgroup v2 相比 Cgroup v1 做的最大的变动就是一个进程属于一个控制组，而每个控制组里可以定义自己需要的多个子系统**。Cgroup v2 对进程 pid 的磁盘 I/O 做限制的时候，就可以考虑到进程 pid 写入到 Page Cache 内存的页面了，这样 buffered I/O 的磁盘限速就实现了。

![img](https://qiankunli.github.io/public/upload/container/cgroup_v2.jpeg)

### 整体实现（可能过时了）

对于CPU Cgroup的配置会影响一个进程的task_struct作为调度单元的scheduled_entity，并影响在CPU上的调度。对于内存 Cgroup的配置起作用在进程申请内存的时候，也即当出现缺页，调用handle_pte_fault进而调用do_anonymous_page的时候，会查看是否超过了配置，超过了就分配失败，OOM。

[使用cgroups控制进程cpu配额](http://www.pchou.info/linux/2017/06/24/cgroups-cpu-quota.html)

从操作上看：

1. 可以创建一个目录（比如叫cgroup-test），

    

   ```plaintext
   mount -t cgroup -o none cgroup-test ./cgroup-test
   ```

    

   cgroup-test 便是一个hierarchy了，一个hierarchy 默认自动创建很多文件 ```

   - cgroup.clone_children
   - cgroup.procs
   - notify_on_release
   - tasks ```

你为其创建一个子文件`cgroup-test/ cgroup-1`，则目录变成 `- cgroup.clone_children - cgroup.procs - notify_on_release - tasks - cgroup-1 - cgroup.clone_children - cgroup.procs - notify_on_release - tasks`

往task 中写进程号，则标记该进程 属于某个cgroup。

注意，mount时，`-o none` 为none。 若是 `mount -t cgroup -o cpu cgroup-test ./cgroup-test` 则表示为cgroup-test hierarchy 挂载 cpu 子系统

```go
- cgroup.event_control
- notify_on_release
- cgroup.procs
- tasks
- cpu.cfs_period_us
- cpu.rt_period_us
- cpu.shares
- cpu.cfs_quota_us
- cpu.rt_runtime_us
- cpu.stat
```

cpu 开头的都跟cpu 子系统有关。可以一次挂载多个子系统，比如`-o cpu,mem`

### 从右向左 ==> 和docker run放在一起看

![img](https://qiankunli.github.io/public/upload/linux/linux_cgroup_docker.png)

### 从左向右 ==> 从 task 结构开始找到 cgroup 结构

[Docker 背后的内核知识——cgroups 资源限制](https://www.infoq.cn/article/docker-kernel-knowledge-cgroups-resource-isolation/)

在图中使用的回环箭头，均表示可以通过该字段找到所有同类结构

![img](https://qiankunli.github.io/public/upload/linux/linux_task_cgroup.png)

### 从右向左 ==> 查看一个cgroup 有哪些task

![img](https://qiankunli.github.io/2016/12/02/docker_cgroup_namespace.html)(/public/upload/linux/linux_task_cgroup.png)

为什么要使用cg_cgroup_link结构体呢？因为 task 与 cgroup 之间是多对多的关系。熟悉数据库的读者很容易理解，在数据库中，如果两张表是多对多的关系，那么如果不加入第三张关系表，就必须为一个字段的不同添加许多行记录，导致大量冗余。通过从主表和副表各拿一个主键新建一张关系表，可以提高数据查询的灵活性和效率。

### 使cgroups 数据生效

我们知道， 任何内存申请都是从缺页中断开始的，`handle_pte_fault ==> do_anonymous_page ==> mem_cgroup_newpage_charge（不同linux版本方法名不同） ==> mem_cgroup_charge_common ==> __mem_cgroup_try_charge`

```go
static int __mem_cgroup_try_charge(struct mm_struct *mm,
    gfp_t gfp_mask,
    unsigned int nr_pages,
    struct mem_cgroup **ptr,
    bool oom){
    ...
    struct mem_cgroup *memcg = NULL;
    ...
    memcg = mem_cgroup_from_task(p);
    ...
}
mem_cgroup_from_task ==> mem_cgroup_from_css
struct mem_cgroup *mem_cgroup_from_task(struct task_struct *p){
    ...
    return mem_cgroup_from_css(task_subsys_state(p, mem_cgroup_subsys_id));
}
```

### 整体

![img](https://qiankunli.github.io/public/upload/linux/linux_cgroup_object.png)

在系统运行之初，内核的主函数就会对root cgroups和css_set进行初始化，每次 task 进行 fork/exit 时，都会附加（attach）/ 分离（detach）对应的css_set。

```
struct cgroup { 
    unsigned long flags; 
    atomic_t count; 
    struct list_head sibling; 
    struct list_head children; 
    struct cgroup *parent; 
    struct dentry *dentry; 
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT]; 
    struct cgroupfs_root *root;
    struct cgroup *top_cgroup; 
    struct list_head css_sets; 
    struct list_head release_list; 
    struct list_head pidlists;
    struct mutex pidlist_mutex; 
    struct rcu_head rcu_head; 
    struct list_head event_list; 
    spinlock_t event_list_lock; 
};
```

sibling,children 和 parent 三个嵌入的 list_head 负责将统一层级的 cgroup 连接成一棵 cgroup 树。