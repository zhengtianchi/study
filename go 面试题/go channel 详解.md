# 什么是 CSP



**不要通过共享内存来通信，而要通过通信来实现内存共享。**

这就是 Go 的并发哲学，它依赖 CSP 模型，基于 channel 实现。

CSP 经常被认为是 Go 在并发编程上成功的关键因素。CSP 全称是 “Communicating Sequential Processes”，这也是 Tony Hoare 在 1978 年发表在 ACM 的一篇论文。论文里指出一门编程语言应该重视 input 和 output 的原语，尤其是并发编程的代码。

在那篇文章发表的时代，人们正在研究模块化编程的思想，该不该用 goto 语句在当时是最激烈的议题。彼时，面向对象编程的思想正在崛起，几乎没什么人关心并发编程。

在文章中，CSP 也是一门自定义的编程语言，作者定义了输入输出语句，用于 processes 间的通信（communicatiton）。processes 被认为是需要输入驱动，并且产生输出，供其他 processes 消费，processes 可以是进程、线程、甚至是代码块。输入命令是：!，用来向 processes 写入；输出是：?，用来从 processes 读出。这篇文章要讲的 channel 正是借鉴了这一设计。

Hoare 还提出了一个 -> 命令，如果 -> 左边的语句返回 false，那它右边的语句就不会执行。

通过这些输入输出命令，Hoare 证明了如果一门编程语言中把 processes 间的通信看得第一等重要，那么并发编程的问题就会变得简单。

Go 是第一个将 CSP 的这些思想引入，并且发扬光大的语言。仅管内存同步访问控制（原文是 memory access synchronization）在某些情况下大有用处，Go 里也有相应的 sync 包支持，但是这在大型程序很容易出错。

Go 一开始就把 CSP 的思想融入到语言的核心里，所以并发编程成为 Go 的一个独特的优势，而且很容易理解。

**大多数的编程语言的并发编程模型是基于线程和内存同步访问控制，Go 的并发编程的模型则用 goroutine 和 channel 来替代。**Goroutine 和线程类似，channel 和 mutex (用于内存同步访问控制)类似。

Goroutine 解放了程序员，让我们更能贴近业务去思考问题。而不用考虑各种像线程库、线程开销、线程调度等等这些繁琐的底层问题，goroutine 天生替你解决好了。

Channel 则天生就可以和其他 channel 组合。我们可以把收集各种子系统结果的 channel 输入到同一个 channel。**Channel 还可以和 select, cancel, timeout 结合起来**。而 mutex 就没有这些功能。

Go 的并发原则非常优秀，目标就是简单：尽量使用 channel；把 goroutine 当作免费的资源，随便用。







## 关闭一个 channel 的过程是怎样的



关闭某个 channel，会执行函数 `closechan`：

```go
func closechan(c *hchan) {
    // 关闭一个 nil channel，panic
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    // 上锁
    lock(&c.lock)
    // 如果 channel 已经关闭
    if c.closed != 0 {
        unlock(&c.lock)
        // panic
        panic(plainError("close of closed channel"))
    }

    // …………

    // 修改关闭状态
    c.closed = 1

    var glist *g

    // 将 channel 所有等待接收队列的里 sudog 释放
    for {
        // 从接收队列里出队一个 sudog
        sg := c.recvq.dequeue()
        // 出队完毕，跳出循环
        if sg == nil {
            break
        }

        // 如果 elem 不为空，说明此 receiver 未忽略接收数据
        // 给它赋一个相应类型的零值
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        // 取出 goroutine
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        // 相连，形成链表
        gp.schedlink.set(glist)
        glist = gp
    }

    // 将 channel 等待发送队列里的 sudog 释放
    // 如果存在，这些 goroutine 将会 panic
    for {
        // 从发送队列里出队一个 sudog
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }

        // 发送者会 panic
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        // 形成链表
        gp.schedlink.set(glist)
        glist = gp
    }
    // 解锁
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    // 遍历链表
    for glist != nil {
        // 取最后一个
        gp := glist
        // 向前走一步，下一个唤醒的 g
        glist = glist.schedlink.ptr()
        gp.schedlink = 0
        // 唤醒相应 goroutine
        goready(gp, 3)
    }
}
```



close 逻辑比较简单，对于一个 channel，recvq 和 sendq 中分别保存了阻塞的发送者和接收者。关闭 channel 后，对于等待接收者而言，会收到一个相应类型的零值。对于等待发送者，会直接 panic。所以，在不了解 channel 还有没有接收者的情况下，不能贸然关闭 channel。

close 函数先上一把大锁，接着把所有挂在这个 channel 上的 sender 和 receiver 全都连成一个 sudog 链表，再解锁。最后，再将所有的 sudog 全都唤醒。

唤醒之后，该干嘛干嘛。sender 会继续执行 chansend 函数里 goparkunlock 函数之后的代码，很不幸，检测到 channel 已经关闭了，panic。receiver 则比较幸运，进行一些扫尾工作后，返回。这里，selected 返回 true，而返回值 received 则要根据 channel 是否关闭，返回不同的值。如果 channel 关闭，received 为 false，否则为 true。这我们分析的这种情况下，received 返回 false。