# Go语言处理运行时错误

Go语言的错误处理思想及设计包含以下特征：

- 一个可能造成错误的函数，需要返回值中返回一个错误接口（error），如果调用是成功的，错误接口将返回 nil，否则返回错误。
- 在函数调用后需要检查错误，如果发生错误，则进行必要的错误处理。


Go语言没有类似 [Java](http://c.biancheng.net/java/) 或 .NET 中的异常处理机制，虽然可以使用 defer、panic、recover 模拟，但官方并不主张这样做，Go语言的设计者认为其他语言的异常机制已被过度使用，上层逻辑需要为函数发生的异常付出太多的资源，同时，如果函数使用者觉得错误处理很麻烦而忽略错误，那么程序将在不可预知的时刻崩溃。

Go语言希望开发者将错误处理视为正常开发必须实现的环节，正确地处理每一个可能发生错误的函数，同时，**Go语言使用返回值返回错误的机制**，也能大幅降低编译器、运行时处理错误的复杂度，让开发者真正地掌握错误的处理。

## net 包中的例子

net.Dial() 是Go语言系统包 net 即中的一个函数，一般用于创建一个 Socket 连接。

net.Dial 拥有两个返回值，即 Conn 和 error，这个函数是阻塞的，因此在 Socket 操作后，会返回 Conn 连接对象和 error，如果发生错误，error 会告知错误的类型，Conn 会返回空。

根据Go语言的错误处理机制，Conn 是其重要的返回值，因此，为这个函数增加一个错误返回，类似为 error，参见下面的代码：

```go
func Dial(network, address string) (Conn, error) {
    var d Dialer
    return d.Dial(network, address)
}
```

在 io 包中的 Writer 接口也拥有错误返回，代码如下：

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

io 包中还有 Closer 接口，只有一个错误返回，代码如下：

```go
type Closer interface {
    Close() error
}
```

## 错误接口的定义格式

error 是 Go 系统声明的接口类型，代码如下：

```go
type error interface {
    Error() string
}
```

所有符合 Error()string 格式的方法，都能实现错误接口，Error() 方法返回错误的具体描述，使用者可以通过这个字符串知道发生了什么错误。

## 自定义一个错误

返回错误前，需要定义会产生哪些可能的错误，在Go语言中，使用 errors 包进行错误的定义，格式如下：

```go
var err = errors.New("this is an error")
```

错误字符串由于相对固定，一般在包作用域声明，应尽量减少在使用时直接使用 errors.New 返回。

#### 1) errors 包

Go语言的 errors 中对 New 的定义非常简单，代码如下：

```go
// 创建错误对象
func New(text string) error {
    return &errorString{text}
}

// 错误字符串
type errorString struct {
    s string
}

// 返回发生何种错误
func (e *errorString) Error() string {
    return e.s
}
```

代码说明如下：

- 第 2 行，将 errorString 结构体实例化，并赋值错误描述的成员。
- 第 7 行，声明 errorString 结构体，拥有一个成员，描述错误内容。
- 第 12 行，实现 error 接口的 Error() 方法，该方法返回成员中的错误描述。

#### 2) 在代码中使用错误定义

下面的代码会定义一个除法函数，当除数为 0 时，返回一个预定义的除数为 0 的错误。

```go
package main
import (
    "errors"
    "fmt"
)
// 定义除数为0的错误
var errDivisionByZero = errors.New("division by zero")
func div(dividend, divisor int) (int, error) {
    // 判断除数为0的情况并返回
    if divisor == 0 {
        return 0, errDivisionByZero
    }
    // 正常计算，返回空错误
    return dividend / divisor, nil
}
func main() {
    fmt.Println(div(1, 0))
}
```

代码输出如下：

```go
0 division by zero
```

代码说明：

- 第 9 行，预定义除数为 0 的错误。
- 第 11 行，声明除法函数，输入被除数和除数，返回商和错误。
- 第 14 行，在除法计算中，如果除数为 0，计算结果为无穷大，为了避免这种情况，对除数进行判断，并返回商为 0 和除数为 0 的错误对象。
- 第 19 行，进行正常的除法计算，没有发生错误时，错误对象返回 nil。

## 示例：在解析中使用自定义错误

使用 errors.New 定义的错误字符串的错误类型是无法提供丰富的错误信息的，那么，如果需要携带错误信息返回，就需要借助自定义结构体实现错误接口。

下面代码将实现一个解析错误（ParseError），这种错误包含两个内容，分别是文件名和行号，解析错误的结构还实现了 error 接口的 Error() 方法，返回错误描述时，就需要将文件名和行号返回。

```go
package main
import (
    "fmt"
)
// 声明一个解析错误
type ParseError struct {
    Filename string // 文件名
    Line     int    // 行号
}
// 实现error接口，返回错误描述
func (e *ParseError) Error() string {
    return fmt.Sprintf("%s:%d", e.Filename, e.Line)
}
// 创建一些解析错误
func newParseError(filename string, line int) error {
    return &ParseError{filename, line}
}
func main() {
    var e error
    // 创建一个错误实例，包含文件名和行号
    e = newParseError("main.go", 1)
    // 通过error接口查看错误描述
    fmt.Println(e.Error())
    // 根据错误接口具体的类型，获取详细错误信息
    switch detail := e.(type) {
    case *ParseError: // 这是一个解析错误
        fmt.Printf("Filename: %s Line: %d\n", detail.Filename, detail.Line)
    default: // 其他类型的错误
        fmt.Println("other error")
    }
}
```

代码输出如下：

```
main.go:1
Filename: main.go Line: 1
```

代码说明如下：

- 第 8 行，声明了一个解析错误的结构体，解析错误包含有 2 个成员，分别是文件名和行号。
- 第 14 行，实现了错误接口，将成员的文件名和行号格式化为字符串返回。
- 第 19 行，根据给定的文件名和行号创建一个错误实例。
- 第 25 行，声明一个错误接口类型。
- 第 27 行，创建一个实例，这个错误接口内部是 *ParserError 类型，携带有文件名 main.go 和行号 1。
- 第 30 行，调用 Error() 方法，通过第 15 行返回错误的详细信息。
- 第 33 行，通过错误断言，取出发生错误的详细类型。
- 第 34 行，通过分析这个错误的类型，得知错误类型为 *ParserError，此时可以获取到详细的错误信息。
- 第 36 行，如果不是我们能够处理的错误类型，会打印出其他错误做出其他的处理。


错误对象都要实现 error 接口的 Error() 方法，这样，所有的错误都可以获得字符串的描述，如果想进一步知道错误的详细信息，可以通过类型断言，将错误对象转为具体的错误类型进行错误详细信息的获取。



# Go语言宕机（panic）——程序终止运行



Go语言的类型系统会在编译时捕获很多错误，但有些错误只能在运行时检查，如数组访问越界、空指针引用等，这些运行时错误会引起宕机。

宕机不是一件很好的事情，可能造成体验停止、服务中断，就像没有人希望在取钱时遇到 ATM 机蓝屏一样，但是，如果在损失发生时，程序没有因为宕机而停止，那么用户将会付出更大的代价，这种代价可以是金钱、时间甚至生命，因此，宕机有时也是一种合理的止损方法。

一般而言，**当宕机发生时，程序会中断运行，并立即执行在该 goroutine（可以先理解成线程）中被延迟的函数（defer 机制）**，随后，程序崩溃并输出日志信息，日志信息包括 panic value 和函数调用的堆栈跟踪信息，panic value 通常是某种错误信息。

对于每个 goroutine，日志信息中都会有与之相对的，发生 panic 时的函数调用堆栈跟踪信息，通常，我们不需要再次运行程序去定位问题，日志信息已经提供了足够的诊断依据，因此，在我们填写问题报告时，一般会将宕机和日志信息一并记录。

虽然Go语言的 panic 机制类似于其他语言的异常，但 panic 的适用场景有一些不同，由于 panic 会引起程序的崩溃，因此 panic 一般用于严重错误，如程序内部的逻辑不一致。任何崩溃都表明了我们的代码中可能存在漏洞，所以对于大部分漏洞，我们应该使用Go语言提供的错误机制，而不是 panic。

## 手动触发宕机

Go语言可以在程序中手动触发宕机，让程序崩溃，这样开发者可以及时地发现错误，同时减少可能的损失。

Go语言程序在宕机时，会将堆栈和 goroutine 信息输出到控制台，所以宕机也可以方便地知晓发生错误的位置，那么我们要如何触发宕机呢，示例代码如下所示：

```go
package main
func main() {
    panic("crash")
}
```

代码运行崩溃并输出如下：

```
panic: crash

goroutine 1 [running]:
main.main()
  D:/code/main.go:4 +0x40
exit status 2
```

以上代码中只用了一个内建的函数 panic() 就可以造成崩溃，panic() 的声明如下：

```
func panic(v interface{})    //panic() 的参数可以是任意类型的。
```

## 在运行依赖的必备资源缺失时主动触发宕机

regexp 是Go语言的正则表达式包，正则表达式需要编译后才能使用，而且编译必须是成功的，表示正则表达式可用。

编译正则表达式函数有两种，具体如下：

#### 1) func Compile(expr string) (*Regexp, error)

编译正则表达式，发生错误时返回编译错误同时返回 Regexp 为 nil，该函数适用于在编译错误时获得编译错误进行处理，同时继续后续执行的环境。

#### 2) func MustCompile(str string) *Regexp

当编译正则表达式发生错误时，使用 panic 触发宕机，该函数适用于直接使用正则表达式而无须处理正则表达式错误的情况。

MustCompile 的代码如下：

```go
func MustCompile(str string) *Regexp {
    regexp, error := Compile(str)
    if error != nil {
        panic(`regexp: Compile(` + quote(str) + `): ` + error.Error())
    }
    return regexp
}
```

代码说明如下：

- 第 1 行，编译正则表达式函数入口，输入包含正则表达式的字符串，返回正则表达式对象。
- 第 2 行，Compile() 是编译正则表达式的入口函数，该函数返回编译好的正则表达式对象和错误。
- 第 3 和第 4 行判断如果有错，则使用 panic() 触发宕机。
- 第 6 行，没有错误时返回正则表达式对象。


手动宕机进行报错的方式不是一种偷懒的方式，反而能迅速报错，终止程序继续运行，防止更大的错误产生，不过，如果任何错误都使用宕机处理，也不是一种良好的设计习惯，因此应根据需要来决定是否使用宕机进行报错。

## 在宕机时触发延迟执行语句

当 panic() 触发的宕机发生时，panic() 后面的代码将不会被运行，但是在 panic() 函数前面已经运行过的 defer 语句依然会在宕机发生时发生作用，参考下面代码：

```go
package main
import "fmt"
func main() {
    defer fmt.Println("宕机后要做的事情1")
    defer fmt.Println("宕机后要做的事情2")
    panic("宕机")
}
```

代码输出如下：

```
宕机后要做的事情2
宕机后要做的事情1
panic: 宕机
goroutine 1 [running]:
main.main()
  D:/code/main.go:8 +0xf8
exit status 2
```

对代码的说明：

- 第 6 行和第 7 行使用 defer 语句延迟了 2 个语句。
- 第 8 行发生宕机。


宕机前，defer 语句会被优先执行，由于第 7 行的 defer 后执行，因此会在宕机前，这个 defer 会优先处理，随后才是第 6 行的 defer 对应的语句，这个特性可以用来在宕机发生前进行宕机信息处理。



# Go语言宕机恢复（recover）——防止程序崩溃

Recover 是一个Go语言的内建函数，可以让进入宕机流程中的 goroutine 恢复过来，**recover 仅在延迟函数 defer 中有效**，在正常的执行过程中，调用 recover 会返回 nil 并且没有其他任何效果，如果当前的 goroutine 陷入恐慌，调用 recover 可以捕获到 panic 的输入值，并且恢复正常的执行。

通常来说，不应该对进入 panic 宕机的程序做任何处理，但有时，需要我们可以从宕机中恢复，至少我们可以在程序崩溃前，做一些操作，举个例子，当 web 服务器遇到不可预料的严重问题时，在崩溃前应该将所有的连接关闭，如果不做任何处理，会使得客户端一直处于等待状态，如果 web 服务器还在开发阶段，服务器甚至可以将异常信息反馈到客户端，帮助调试。

#### 提示

在其他语言里，宕机往往以异常的形式存在，底层抛出异常，上层逻辑通过 try/catch 机制捕获异常，没有被捕获的严重异常会导致宕机，捕获的异常可以被忽略，让代码继续运行。

Go语言没有异常系统，**其使用 panic 触发宕机类似于其他语言的抛出异常，recover 的宕机恢复机制就对应其他语言中的 try/catch 机制。**

## 让程序在崩溃时继续执行

下面的代码实现了 ProtectRun() 函数，该函数传入一个匿名函数或闭包后的执行函数，当传入函数以任何形式发生 panic 崩溃后，可以将崩溃发生的错误打印出来，同时允许后面的代码继续运行，不会造成整个进程的崩溃。

保护运行函数：

```go
package main

import (
    "fmt"
    "runtime"
)

// 崩溃时需要传递的上下文信息
type panicContext struct {
    function string // 所在函数
}

// 保护方式允许一个函数
func ProtectRun(entry func()) {
    // 延迟处理的函数
    defer func() {
        // 发生宕机时，获取panic传递的上下文并打印
        err := recover()
        switch err.(type) {
        case runtime.Error: // 运行时错误
            fmt.Println("runtime error:", err)
        default: // 非运行时错误
            fmt.Println("error:", err)
        }
    }()
    entry()
}

func main() {
    fmt.Println("运行前")
    // 允许一段手动触发的错误
    ProtectRun(func() {
        fmt.Println("手动宕机前")
        // 使用panic传递上下文
        panic(&panicContext{
            "手动触发panic",
        })
        fmt.Println("手动宕机后")  // panic 后面的不会被执行
    })
    // 故意造成空指针访问错误
    ProtectRun(func() {
        fmt.Println("赋值宕机前")
        var a *int
        *a = 1
        fmt.Println("赋值宕机后")   // panic 后面的不会被执行
    })
    fmt.Println("运行后")
}
```

代码输出结果：

```
运行前
手动宕机前
error: &{手动触发panic}

赋值宕机前
runtime error: runtime error: invalid memory address or nil pointer dereference
运行后
```

对代码的说明：

- 第 9 行声明描述错误的结构体，保存执行错误的函数。
- 第 17 行使用 defer 将闭包延迟执行，当 panic 触发崩溃时，ProtectRun() 函数将结束运行，此时 defer 后的闭包将会发生调用。
- 第 20 行，recover() 获取到 panic 传入的参数。
- 第 22 行，使用 switch 对 err 变量进行类型断言。
- 第 23 行，如果错误是有 Runtime 层抛出的运行时错误，如空指针访问、除数为 0 等情况，打印运行时错误。
- 第 25 行，其他错误，打印传递过来的错误数据。
- 第 44 行，使用 panic 手动触发一个错误，并将一个结构体附带信息传递过去，此时，recover 就会获取到这个结构体信息，并打印出来。
- 第 57 行，模拟代码中空指针赋值造成的错误，此时会由 **Runtime 层抛出错误**，被 ProtectRun() 函数的 recover() 函数捕获到。

## panic 和 recover 的关系

panic 和 recover 的组合有如下特性：

- 有 panic 没 recover，程序宕机。
- 有 panic 也有 recover，程序不会宕机，执行完对应的 defer 后，**从宕机点退出当前函数后继续执行。**

#### 提示

虽然 panic/recover 能模拟其他语言的异常机制，但并不建议在编写普通函数时也经常性使用这种特性。

在 panic 触发的 defer 函数内，可以继续调用 panic，进一步将错误外抛，直到程序整体崩溃。

如果想在捕获错误时设置当前函数的返回值，可以对返回值使用命名返回值方式直接进行设置。