

## 1. goroutine 里面 panic 了会怎么样



在Go语言中，我们通常会用到panic和recover来抛出错误和捕获错误，这一对操作在单协程环境下我们正常用就好了，并不会踩到什么坑。但是在多协程并发环境下，我们常常会碰到以下两个问题。假设我们现在有2个协程，我们叫它们协程A和B好了：

- 如果协程A发生了panic，协程B是否会因为协程A的panic而挂掉？
- 如果协程A发生了panic，协程B是否能用recover捕获到协程A的panic？

答案分别是：会、不能。



**哪个协程发生了panic，我们就需要在哪个协程recover**



## 2. Go 中 defer 机制，可以返回数据吗

### defer概述

`defer` 是`golang` 中独有的流程控制语句，用于延迟指定语句的运行时机，只能运行于函数的内部，且当他所属函数运行完之后它才会被调用。例如：

```
func deferTest(){
    defer fmt.Println("HelloDefer")
    fmt.Println("HelloWorld")
}
```

它会先打印出`HelloWorld` ，然后再打印出`HelloDefer` 。

一个函数中如果有多个`defer` ，运行顺序和函数中的调用顺序相反，因为它们都是被写在了栈中：

```go
func deferTest(){
    defer fmt.Println("HelloDefer1")
    defer fmt.Println("HelloDefer2")
    fmt.Println("HelloWorld")
}
```

运行结果：

```go
fmt.Println("HelloDefer2")
fmt.Println("HelloDefer1")
fmt.Println("HelloWorld")
```

### defer和return

在包含有`return` 语句的函数中**，`defer` 的运行顺序位于`return` 之后，但是`defer` 所运行的代码片段会生效**：

```go
func main(){
    fmt.Println(deferReturn)
}
func deferReturn() int{
    i := 1
    defer func(){
        fmt.Println("Defer")
        i += 1
    }()
    return func()int{
        fmt.Println("Return")
        return i
    }()
}
```

运行结果：

```
Return
Defer
1
```

这里很明显就能看到`defer` 是在`return` 之后运行的！但是有一个问题是`defer` 里执行了语句`i +=1` ，按照这个逻辑的话返回的`i` 值应该是`2` 而不是`1` 。这个问题是由于`return` 的运行机制导致的：`return` 在返回一个对象时，如果返回类型不是指针或者引用类型，那么`return` 返回的就不会是这个对象本身，而是这个对象的副本。

我们可以验证这一个观点：

```go
func main(){
    ...
    fmt.Println("main:    ", x, &x)
}
func deferReturn() int{
    ...
    defer ...{
        fmt.Println("Defer:    ", i, &i)
        ...
    }()
    return ...{
        fmt.Println("Return:    ", i, &i)
        ...
    }()
}
```

程序的输出为：

```
Return:     1 0xc042008238
Defer:     1 0xc042008238
main:     1 0xc042008230  //main函数中的i的地址和deferReturn()中的i的地址是不一样的
```

如果把函数的返回值改成指针类型，这时候的main函数中的返回值就会和函数体内的一致：

```
func main(){
    x := deferReturn()
    fmt.Println("main:    ", x, *x)
}
func deferReturn()*int{
    i := 1
    p := &i
    defer func() {
        *p += 1
        fmt.Println("defer:    ", p, *p)
    }()
    return func() *int{
        fmt.Println("Return:    ", p, *p)
        return p
    }()
}
```

结果：

```
Return:     0xc0420361d0 1
defer:     0xc0420361d0 2
main:     0xc0420361d0 2
```

### defer与return的执行顺序

首先看个例子：

```go
package main

import (
    "fmt"
)

func main() {
    ret := test()
    fmt.Println("test return:", ret)
}

func test() ( int) {
    var i int

    defer func() {
        i++        //defer里面对i增1
        fmt.Println("test defer, i = ", i)
    }()

    return i
}
```

执行结果为：

```subunit
test defer, i =  1
test return: 0
```

test函数的返回值为0，defer里面的i++操作好像对返回值并没有什么影响。
这是否表示“return i”执行结束以后才执行defer呢？
非也！再看下面的例子：

```go
package main

import (
    "fmt"
)

func main() {
    ret := test()
    fmt.Println("test return:", ret)
}

//返回值改为命名返回值
func test() (i int) {
    //var i int

    defer func() {
        i++
        fmt.Println("test defer, i = ", i)
    }()

    return i
}
```

执行结果为：

```subunit
test defer, i =  1
test return: 1
```

这次test函数的返回值变成了1，defer里面的“i++"修改了返回值。所以defer的执行时机应该是return之后，且返回值返回给调用方之前。
至于第一个例子中test函数返回值不是1的原因，还涉及到函数匿名返回值与命名返回值的差异，以后再单独分析。

#### 结论

1. defer的执行顺序为：后defer的先执行。
2. :heavy_check_mark: **defer的执行顺序在return之后，但是在返回值返回给调用方之前，所以使用defer可以达到修改返回值的目的**。





## 3. go里面goroutine创建数量有限制吗？



- 有，P本地队列有数量限制，不允许超过 256 个
- 过多会占用大量的CPU和内存，并且会导致主进程奔溃





## 4. select可以用于什么

**在多个通道上进行读或写操作，让函数可以处理多个事情，但1次只处理1个。以下特性也都必须熟记于心：**

1. 每次执行select，都会只执行其中1个case或者执行default语句。
2. 当没有case或者default可以执行时，select则阻塞，等待直到有1个case可以执行。
3. 当有多个case可以执行时，则随机选择1个case执行。
4. `case`后面跟的必须是读或者写通道的操作，否则编译出错。



## 5. go map实现

#### sync.Map实现原理，适用的场景





## 6. Go GC算法，三色标记法描述



## 7. Go内存模型(tcmalloc)



## 8. Go channel

- 在不能更改channel状态的情况下，没有简单普遍的方式来检查channel是否已经关闭了
- 关闭已经关闭的channel会导致panic，所以在closer(关闭者)不知道channel是否已经关闭的情况下去关闭channel是很危险的
- 发送值到已经关闭的channel会导致panic，所以如果sender(发送者)在不知道channel是否已经关闭的情况下去向channel发送值是很危险的



## 9. 判断 channel 是否已经被关闭

1. **从channel读取数据 ** (OK法) 

-  **第二个字段为true时，channel可能没关闭，也可能已经关闭，不能证明什么**
- **第二个字段为false时，可以证明channel中已没有残留数据且已关闭**
-  **若channel已关闭 并且通道为空，那么从该channel中读取数据会直接返回，且是默认值，所以一定要判断第二个字段**

2. `for-range`是使用频率很高的结构，常用它来遍历数据，**`range`能够感知channel的关闭，当channel被发送数据的协程关闭时，range就会结束**，接着退出for循环。



## 10 关闭后的通道有以下特点：

1. 对一个关闭的通道再发送值就会导致panic。
2. **对一个关闭的通道进行接收会一直获取值直到通道为空**。
3. **对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。**
4. 关闭一个已经关闭的通道会导致panic。



## 11 go select介绍

A. select机制用来处理异步IO问题

B. select机制最大的一条限制就是每个case语句里必须是一个IO操作



