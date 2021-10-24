### 创建channel

- 声明的通道后需要使用`make`函数初始化之后才能使用。



### channel操作

通道有发送（send）、接收(receive）和关闭（close）三种操作。

发送和接收都使用`<-`符号。

现在我们先使用以下语句定义一个通道：

```go
ch := make(chan int)
复制代码
```

#### 发送

将一个值发送到通道中。

```go
ch <- 10 // 把10发送到ch中
复制代码
```

#### 接收

从一个通道中接收值。

```go
x := <- ch // 从ch中接收值并赋值给变量x
<-ch       // 从ch中接收值，忽略结果
复制代码
```

#### 关闭

我们通过调用内置的`close`函数来关闭通道。

```go
close(ch)
复制代码
```

关于关闭通道需要注意的事情是，只有在通知接收方goroutine所有的数据都发送完毕的时候才需要关闭通道。通道是可以被垃圾回收机制回收的，它和关闭文件是不一样的，在结束操作之后关闭文件是必须要做的，但关闭通道不是必须的。

关闭后的通道有以下特点：

1. 对一个关闭的通道再发送值就会导致panic。
2. **对一个关闭的通道进行接收会一直获取值直到通道为空**。
3. **对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。**
4. 关闭一个已经关闭的通道会导致panic。



### 无缓冲的通道

无缓冲通道上的发送操作会**阻塞**，直到另一个`goroutine`在该通道上执行接收操作，这时值才能发送成功，两个`goroutine`将继续执行。相反，如果接收操作先执行，接收方的goroutine将阻塞，直到另一个`goroutine`在该通道上发送一个值。

使用无缓冲通道进行通信将导致发送和接收的`goroutine`同步化。因此，无缓冲通道也被称为`同步通道`。



### 有缓冲的通道

解决上面问题的方法还有一种就是使用有缓冲区的通道。我们可以在使用make函数初始化通道的时候为其指定通道的容量，例如：

```go
func main() {
	ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
	ch <- 10
	fmt.Println("发送成功")
}
复制代码
```

只要通道的容量大于零，那么该通道就是有缓冲的通道，通道的容量表示通道中能存放元素的数量。就像你小区的快递柜只有那么个多格子，格子满了就装不下了，就阻塞了，等到别人取走一个快递员就能往里面放一个。

我们可以使用内置的`len`函数获取通道内元素的数量，使用`cap`函数获取通道的容量，虽然我们很少会这么做。



### for range从通道循环取值 (判断一个通道是否被关闭了呢)

当向通道中发送完数据时，我们可以通过`close`函数来关闭通道。

当通道被关闭时，再往该通道发送值会引发`panic`，从该通道取值的操作会先取完通道中的值，再然后取到的值一直都是对应类型的零值。那如何判断一个通道是否被关闭了呢？

我们来看下面这个例子：

```go
// channel 练习
func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)
	// 开启goroutine将0~100的数发送到ch1中
	go func() {
		for i := 0; i < 100; i++ {
			ch1 <- i
		}
		close(ch1)
	}()
	// 开启goroutine从ch1中接收值，并将该值的平方发送到ch2中
	go func() {
		for {
			i, ok := <-ch1 // 通道关闭后再取值ok=false
			if !ok {
				break
			}
			ch2 <- i * i
		}
		close(ch2)
	}()
	// 在主goroutine中从ch2中接收值打印
	for i := range ch2 { // 通道关闭后会退出for range循环
		fmt.Println(i)
	}
}
复制代码
```

从上面的例子中我们看到有两种方式在接收值的时候判断该通道是否被关闭，不过我们通常使用的是`for range`的方式。使用`for range`遍历通道，当通道被关闭的时候就会退出`for range`。







```go
package main
import (
	"fmt"
)
func main() {
	c := make(chan int, 10)
	c <- 1
	c <- 2
	c <- 3
	close(c) // 这里就关闭了channel,下面依然可以读取到channel里面残留的数据,直到数据处理完成后才读取到关闭消息
	// c <- 4	// 本行代码不可执行,向一个已关闭的通道压入数据时会造成panic

	// 所以证明了根据从channel读取到的第二个字段的值为true或者false时,并不能证明channel是否未关闭或已关闭
	// 总结
	// 1. 第二个字段的值为true时，channel可能没关闭，也可能已经关闭，不能证明什么
	// 2. 第二个字段为false时，可以证明channel中已没有残留数据且已关闭

	for {
		i, isOpen := <-c
		if !isOpen { // 若不加本判断,从一个已关闭的channel中读取数据会直接返回,且是默认值,造成死循环!!!!!!
			fmt.Println("channel 已关闭!")
			break
		}
		fmt.Printf("i=%v, isOpen=%v \n", i, isOpen)
	}
}
```

**结论：**

\1. go语言无法即时准确地判断channel是否关闭

\2. **从channel读取数据**

  **2.1 第二个字段为true时，channel可能没关闭，也可能已经关闭，不能证明什么**

  **2.2 第二个字段为false时，可以证明channel中已没有残留数据且已关闭**

\3. 若channel已关闭，那么从该channel中读取数据会直接返回，且是默认值，所以一定要判断第二个字段













# [go 优雅的检查channel关闭](https://www.cnblogs.com/-wenli/p/12350181.html)

```
原文作者：shitaibin``链接：https:``//www.jianshu.com/p/79d27f200bcf``來源：简书
```

goroutine作为Golang并发的核心，我们不仅要关注它们的创建和管理，当然还要关注如何合理的退出这些协程，不（合理）退出不然可能会造成阻塞、panic、程序行为异常、数据结果不正确等问题。这篇文章介绍，如何合理的退出goroutine，减少软件bug。
goroutine在退出方面，不像线程和进程，不能通过某种手段**强制**关闭它们，只能等待goroutine主动退出。但也无需为退出、关闭goroutine而烦恼，下面就介绍3种优雅退出goroutine的方法，只要采用这种最佳实践去设计，基本上就可以确保goroutine退出上不会有问题，尽情享用。

## 第一种：使用for-range退出

`for-range`是使用频率很高的结构，常用它来遍历数据，**`range`能够感知channel的关闭，当channel被发送数据的协程关闭时，range就会结束**，接着退出for循环。
它在并发中的使用场景是：当协程只从1个channel读取数据，然后进行处理，处理后协程退出。下面这个示例程序，当in通道被关闭时，协程可自动退出。

```go
go func(in <-chan int) {
    // Using for-range to exit goroutine
    // range has the ability to detect the close/end of a channel
    for x := range in {
        fmt.Printf("Process %d\n", x)
    }
}(inCh)
```

　　

## 第二种：使用,ok退出

`for-select`也是使用频率很高的结构，select提供了多路复用的能力，所以for-select可以让函数具有持续多路处理多个channel的能力。**但select没有感知channel的关闭，这引出了2个问题**：1）继续在关闭的通道上读，会读到通道传输数据类型的零值，2）继续在关闭的通道上写，将会panic。问题2可使用的原则是，通道只由发送方关闭，接收方不可关闭，即某个写通道只由使用该select的协程关闭，select中就不存在继续在关闭的通道上写数据的问题。

问题1可以使用`,ok`来检测通道的关闭，使用情况有2种。
第一种：**如果某个通道关闭后，需要退出协程，直接return即可**。示例代码中，该协程需要从in通道读数据，还需要定时打印已经处理的数量，有2件事要做，所有不能使用for-range，需要使用for-select，当in关闭时，`ok=false`，我们直接返回。

```go
go func() {
    // in for-select using ok to exit goroutine
    for {
        select {
        case x, ok := <-in:
            if !ok {
                return
            }
            fmt.Printf("Process %d\n", x)
            processedCnt++
        case <-t.C:
            fmt.Printf("Working, processedCnt = %d\n", processedCnt)
        }
    }
}()
```

第二种：如果**某个通道关闭了，不再处理该通道，而是继续处理其他case**，退出是等待所有的可读通道关闭。我们需要**使用select的一个特征：select不会在nil的通道上进行等待**。这种情况，把只读通道设置为nil即可解决。

 

```go
go func() {
    // in for-select using ok to exit goroutine
    for {
        select {
        case x, ok := <-in1:
            if !ok {
                in1 = nil
            }
            // Process
        case y, ok := <-in2:
            if !ok {
                in2 = nil
            }
            // Process
        case <-t.C:
            fmt.Printf("Working, processedCnt = %d\n", processedCnt)
        }
 
        // If both in channel are closed, goroutine exit
        if in1 == nil && in2 == nil {
            return
        }
    }
}()
```

　　

## 第三种：使用退出通道退出

**使用`,ok`来退出使用for-select协程，解决是当读入数据的通道关闭时，没数据读时程序的正常结束**。想想下面这2种场景，`,ok`还能适用吗？

1. 接收的协程要退出了，如果它直接退出，不告知发送协程，发送协程将阻塞。
2. 启动了一个工作协程处理数据，如何通知它退出？

**使用一个专门的通道，发送退出的信号，可以解决这类问题**。以第2个场景为例，协程入参包含一个停止通道`stopCh`，当`stopCh`被关闭，`case <-stopCh`会执行，直接返回即可。

当我启动了100个worker时，只要`main()`执行关闭stopCh，每一个worker都会都到信号，进而关闭。如果`main()`向stopCh发送100个数据，这种就低效了。

```go
func worker(stopCh <-chan struct{}) {
    go func() {
        defer fmt.Println("worker exit")
        // Using stop channel explicit exit
        for {
            select {
            case <-stopCh:
                fmt.Println("Recv stop signal")
                return
            case <-t.C:
                fmt.Println("Working .")
            }
        }
    }()
    return
}
```

## 最佳实践回顾

1. 发送协程主动关闭通道，接收协程不关闭通道。技巧：把接收方的通道入参声明为只读(`<-chan`)，如果接收协程关闭只读协程，编译时就会报错。
2. 协程处理1个通道，并且是读时，协程优先使用`for-range`，因为`range`可以关闭通道的关闭自动退出协程。
3. `,ok`可以处理多个读通道关闭，需要关闭当前使用`for-select`的协程。
4. 显式关闭通道`stopCh`可以处理主动通知协程退出的场景。