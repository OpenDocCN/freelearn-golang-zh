# 第十一章：处理通道和 goroutines

本章将涵盖使用 Go 进行并发编程，使用其基本内置功能、通道和 goroutines。并发描述了在同一时间段内执行应用程序的不同部分的能力。

使软件并发可以成为构建系统应用程序的强大工具，因为一些操作可以在其他操作尚未结束时开始。

本章将涵盖以下主题：

+   理解 goroutines

+   探索通道

+   优势使用

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅第三章，*Go 概述*。

# 理解 goroutines

Go 是一种以并发为中心的语言，以至于两个主要特性——通道和 goroutines——都是内置包的一部分。我们现在将看到它们是如何工作以及它们的基本功能是什么，首先是 goroutines，它使得可以并发执行应用程序的部分。

# 比较线程和 goroutines

Goroutines 是用于并发的原语之一，但它们与线程有何不同？让我们在这里阅读它们的每一个。

# 线程

当前操作系统是为具有每个 CPU 多个核心的现代架构构建的，或者使用超线程等技术，允许单个核心支持多个线程。线程是可以由操作系统调度程序管理的进程的一部分，可以将它们分配给特定的核心/CPU。与进程一样，线程携带有关应用程序执行的信息，但是这些信息的大小小于进程。这包括程序中的当前指令，当前执行的堆栈以及所需的变量。

操作系统已经负责进程之间的上下文切换；它保存旧进程信息并加载新进程信息。这被称为**进程上下文切换**，这是一个非常昂贵的操作，甚至比进程执行更昂贵。

为了从一个线程跳转到另一个线程，可以在线程之间执行相同的操作。这被称为**线程上下文切换**，它也是一个繁重的操作，即使它不像进程切换那样繁重，因为线程携带的信息比进程少。

# Goroutines

线程在内存中有最小大小；通常，它的大小是以 MB 为单位的（Linux 为 2MB）。最小大小对新线程的应用程序创建设置了一些限制——如果每个线程至少有几 MB，那么 1,000 个线程将占用至少几 GB 的内存。Go 解决这些问题的方式是通过使用类似线程的构造，但这是由语言运行时而不是操作系统处理的。goroutine 在内存中的大小是三个数量级（每个 goroutine 为 2KB），这意味着 1,000 个 goroutines 的最小内存使用量与单个线程的内存使用量相当。

这是通过定义 goroutines 内部保留的数据来实现的，使用一个称为`g`的数据结构来描述 goroutine 信息，例如堆栈和状态。这是`runtime`包中的一个未导出的数据类型，并且可以在 Go 源代码中找到。Go 使用来自相同包的另一个数据结构来跟踪操作系统，称为`m`。用于执行 goroutine 的逻辑处理器存储在`p`结构中。这可以在 Go `runtime`包文档中进行验证：

+   `type g`: [golang.org/pkg/runtime/?m=all#m](https://golang.org/pkg/runtime/?m=all#g)

+   `type m`: [golang.org/pkg/runtime/?m=all#g](https://golang.org/pkg/runtime/?m=all#m)

+   `type p`: [golang.org/pkg/runtime/?m=all#p](https://golang.org/pkg/runtime/?m=all#p)

这三个实体的交互如下——对于每个 goroutine，都会创建一个新的`g`，`g`被排入`p`，每个`p`都会尝试获取`m`来执行`g`中的代码。有一些操作会阻塞执行，例如这些：

+   内置同步（通道和`sync`包）

+   阻塞的系统调用，例如文件操作

+   网络操作

当这些类型的操作发生时，运行时会将`p`从`m`中分离出来，并使用（或创建，如果尚不存在）另一个专用的`m`来执行阻塞操作。执行此类操作后，线程变为空闲状态。

# 新的 goroutine

Goroutines 是 Go 如何在简单接口后隐藏复杂性的最佳示例之一。在编写应用程序以启动 goroutine 时，所需的只是执行一个以`go`关键字开头的函数：

```go
func main() {
    go fmt.Println("Hello, playground")
}
```

完整的示例可在[`play.golang.org/p/3gPGZkJtJYv`](https://play.golang.org/p/3gPGZkJtJYv)找到。

如果我们运行上一个示例的应用程序，我们会发现它不会产生任何输出。为什么？在 Go 中，应用程序在主 goroutine 终止时终止，看起来是这种情况。发生的情况是，Go 语句创建具有相应`runtime.g`的 goroutine，但这必须由 Go 调度程序接管，而这并没有发生，因为程序在 goroutine 实例化后立即终止。

使用`time.Sleep`函数让主 goroutine 等待（即使是一纳秒！）足以让调度程序挑选出 goroutine 并执行其代码。这在以下代码中显示：

```go
func main() {
    go fmt.Println("Hello, playground")
    time.Sleep(time.Nanosecond)
}
```

完整的示例可在[`play.golang.org/p/2u125pTclv6`](https://play.golang.org/p/2u125pTclv6)找到。

我们已经看到 Go 方法也算作函数，这就是为什么它们可以像普通函数一样与`go`语句并发执行：

```go
type a struct{}

func (a) Method() { fmt.Println("Hello, playground") }

func main() {
    go a{}.Method()
    time.Sleep(time.Nanosecond)
}
```

完整的示例可在[`play.golang.org/p/RUhgfRAPa2b`](https://play.golang.org/p/RUhgfRAPa2b)找到。

闭包是匿名函数，因此它们也可以被使用，这实际上是一个非常常见的做法：

```go
func main() {
    go func() {
        fmt.Println("Hello, playground")
    }()
    time.Sleep(time.Nanosecond)
}

```

完整的示例可在[`play.golang.org/p/a-JvOVwAwUV`](https://play.golang.org/p/a-JvOVwAwUV)找到。

# 多个 goroutines

在多个 goroutine 中组织代码可以帮助将工作分配给处理器，并具有许多其他优势，我们将在接下来的章节中看到。由于它们如此轻量级，我们可以使用循环非常容易地创建多个 goroutine：

```go
func main() {
    for i := 0; i < 10; i++ {
        go fmt.Println(i)
    }
    time.Sleep(time.Nanosecond)
}
```

完整的示例可在[`play.golang.org/p/Jaljd1padeX`](https://play.golang.org/p/Jaljd1padeX)找到。

这个示例并行打印从`0`到`9`的数字列表，使用并发的 goroutines 而不是在单个 goroutine 中顺序执行相同的操作。

# 参数评估

如果我们稍微改变这个示例，使用没有参数的闭包，我们将看到一个非常不同的结果：

```go
func main() {
    for i := 0; i < 10; i++ {
         go func() { fmt.Println(i) }()
    }
    time.Sleep(time.Nanosecond)
}
```

完整的示例可在[`play.golang.org/p/RV54AsYY-2y`](https://play.golang.org/p/RV54AsYY-2y)找到。

如果我们运行此程序，我们会看到 Go 编译器在循环中发出警告：`循环变量 i 被函数文字捕获`。

循环中的变量被引用在我们定义的函数中——goroutines 的创建循环比 goroutines 的执行更快，结果是循环在单个 goroutine 启动之前就完成了，导致在最后一次迭代后打印循环变量的值。

为了避免捕获循环变量的错误，最好将相同的变量作为参数传递给闭包。 goroutine 函数的参数在创建时进行评估，这意味着对该变量的更改不会在 goroutine 内部反映出来，除非您传递对值的引用，例如指针，映射，切片，通道或函数。我们可以通过运行以下示例来看到这种差异：

```go
func main() {
    var a int
    // passing value
    go func(v int) { fmt.Println(v) }(a)

    // passing pointer
    go func(v *int) { fmt.Println(*v) }(&a)

    a = 42
    time.Sleep(time.Nanosecond)
}
```

完整的示例可在[`play.golang.org/p/r1dtBiTUMaw`](https://play.golang.org/p/r1dtBiTUMaw)找到。

按值传递参数不受程序的最后赋值的影响，而传递指针类型意味着对指针内容的更改将被 goroutine 看到。

# 同步

Goroutine 允许代码并发执行，但值之间的同步不能保证。我们可以看看在尝试并发使用变量时会发生什么，例如下面的例子：

```go
func main() {
    var i int
    go func(i *int) {
        for j := 0; j < 20; j++ {
            time.Sleep(time.Millisecond)
            fmt.Println(*i, j)
        }
    }(&i)
    for i = 0; i < 20; i++ {
        time.Sleep(time.Millisecond)
        fmt.Println(i)
    }
}
```

我们有一个整数变量，在主例程中更改——在每次操作之间进行毫秒暂停——并在更改后打印值。

在另一个 goroutine 中，有一个类似的循环（使用另一个变量）和另一个`print`语句来比较这两个值。考虑到暂停是相同的，我们期望看到相同的值，但事实并非如此。我们看到有时两个 goroutine 不同步。

更改不会立即反映，因为内存不会立即同步。我们将在下一章中学习如何确保数据同步。

# 探索通道

通道是 Go 和其他几种编程语言中独有的概念。通道是非常强大的工具，可以简单地实现不同 goroutine 之间的同步，这是解决前面例子中提出的问题的一种方法。

# 属性和操作

通道是 Go 中的一种内置类型，类型为数组、切片和映射。它以`chan type`的形式呈现，并通过`make`函数进行初始化。

# 容量和大小

除了通过通道传输的类型之外，通道还具有另一个属性：它的`容量`。这代表了通道在进行任何新的发送尝试之前可以容纳的项目数量，从而导致阻塞操作。通道的容量在创建时决定，其默认值为`0`：

```go
// channel with implicit zero capacity
var a = make(chan int)

// channel with explicit zero capacity
var a = make(chan int, 0)

// channel with explicit capacity
var a = make(chan int, 10)
```

通道的容量在创建后无法更改，并且可以随时使用内置的`cap`函数进行读取：

```go
func main() {
    var (
        a = make(chan int, 0)
        b = make(chan int, 5)
    )

    fmt.Println("a is", cap(a))
    fmt.Println("b is", cap(b))
}
```

完整示例可在[`play.golang.org/p/Yhz4bTxm5L8`](https://play.golang.org/p/Yhz4bTxm5L8)中找到。

`len`函数在通道上使用时，告诉我们通道中保存的元素数量：

```go
func main() {
    var (
        a = make(chan int, 5)
    )
    for i := 0; i < 5; i++ {
        a <- i
        fmt.Println("a is", len(a), "/", cap(a))
    }
}
```

完整示例可在[`play.golang.org/p/zJCL5VGmMsC`](https://play.golang.org/p/zJCL5VGmMsC)中找到。

从前面的例子中，我们可以看到通道容量保持为`5`，并且随着每个元素的增加而增加。

# 阻塞操作

如果通道已满或其容量为`0`，则操作将被阻塞。如果我们采用最后一个例子，填充通道并尝试执行另一个发送操作，我们的应用程序将被卡住。

```go
func main() {
    var (
        a = make(chan int, 5)
    )
    for i := 0; i < 5; i++ {
        a <- i
        fmt.Println("a is", len(a), "/", cap(a))
    }
    a <- 0 // Blocking
}
```

完整示例可在[`play.golang.org/p/uSfm5zWN8-x`](https://play.golang.org/p/uSfm5zWN8-x)中找到。

当所有 goroutine 都被锁定时（在这种特定情况下，我们只有主 goroutine），Go 运行时会引发死锁，这是一个终止应用程序执行的致命错误：

```go
fatal error: all goroutines are asleep - deadlock!
```

这种情况可能发生在接收或发送操作中，这是应用程序设计错误的症状。让我们看下面的例子：

```go
func main() {
    var a = make(chan int)
    a <- 10
    fmt.Println(<-a)
}
```

在前面的例子中，有`a <- 10`发送操作和匹配的`<-a`接收操作，但仍然导致死锁。然而，我们创建的通道没有容量，因此第一个发送操作将被阻塞。我们可以通过两种方式进行干预：

+   **通过增加容量**：这是一个非常简单的解决方案，涉及使用`make(chan int, 1)`初始化通道。只有在接收者数量是已知的情况下才能发挥最佳作用；如果它高于容量，则问题会再次出现。

+   **通过使操作并发进行**：这是一个更好的方法，因为它使用通道来实现并发。

让我们尝试使用第二种方法使前面的例子工作：

```go
func main() {
    var a = make(chan int)
    go func() {
        a <- 10
    }()
    fmt.Println(<-a)
}
```

现在，我们可以看到这里没有死锁，程序正确打印了值。使用容量方法也可以使其工作，但它将根据我们发送单个消息的事实进行调整，而另一种方法将允许我们通过通道发送任意数量的消息，并从另一侧相应地接收它们：

```go
func main() {
    const max = 10
    var a = make(chan int)

    go func() {
        for i := 0; i < max; i++ {
            a <- i
        }
    }()
    for i := 0; i < max; i++ {
        fmt.Println(<-a)
    }
}
```

完整示例可在[`play.golang.org/p/RKcojupCruB`](https://play.golang.org/p/RKcojupCruB)找到。

现在我们有一个常量来存储执行的操作次数，但有一种更好更惯用的方法可以让接收方知道没有更多的消息。我们将在下一章关于同步的内容中介绍这个。

# 关闭通道

处理发送方和接收方之间同步结束的最佳方法是`close`操作。这个函数通常由发送方执行，因为接收方可以使用第二个变量验证通道是否仍然打开：

```go
value, ok := <-ch
```

第二个接收方是一个布尔值，如果通道仍然打开，则为`true`，否则为`false`。当在`close`通道上执行接收操作时，第二个接收到的变量将具有`false`值，第一个变量将具有通道类型的`0`值，如下所示：

+   数字为`0`

+   布尔值为`false`

+   字符串为`""`

+   对于切片、映射或指针，使用`nil`

可以使用`close`函数重写发送多条消息的示例，而无需事先知道将发送多少条消息：

```go
func main() {
    const max = 10
    var a = make(chan int)

    go func() {
        for i := 0; i < max; i++ {
            a <- i
        }
        close(a)
    }()
    for {
        v, ok := <-a
        if !ok {
            break
        }
        fmt.Println(v)
    }
}
```

完整示例可在[`play.golang.org/p/GUzgG4kf5ta`](https://play.golang.org/p/GUzgG4kf5ta)找到。

有一种更简洁和优雅的方法可以接收来自通道的消息，直到它被关闭：通过使用我们用于迭代映射、数组和切片的相同关键字。这是通过`range`完成的：

```go
for v := range a {
    fmt.Println(v)
}
```

# 单向通道

处理通道变量时的另一种可能性是指定它们是仅用于发送还是仅用于接收数据。这由`<-`箭头指示，如果仅用于接收，则将在`chan`之前，如果仅用于发送，则将在其后：

```go
func main() {
    var a = make(chan int)
    s, r := (chan<- int)(a), (<-chan int)(a)
    fmt.Printf("%T - %T", s, r)
}
```

完整示例可在[`play.golang.org/p/ZgEPZ99PLJv`](https://play.golang.org/p/ZgEPZ99PLJv)找到。

通道已经是指针了，因此将其中一个转换为其只发送或只接收版本将返回相同的通道，但将减少可以在其上执行的操作数量。通道的类型如下：

+   只发送通道，`chan<-`，允许您发送项目，关闭通道，并防止您发送数据，从而导致编译错误。

+   只接收通道，`<-chan`，允许您接收数据，任何发送或关闭操作都将导致编译错误。

当函数参数是发送/接收通道时，转换是隐式的，这是一个好习惯，因为它可以防止接收方关闭通道等错误。我们可以采用另一个示例，并利用单向通道进行一些重构。

我们还可以创建一个用于发送值的函数，该函数使用只发送通道：

```go
func send(ch chan<- int, max int) {
    for i := 0; i < max; i++ {
        ch <- i
    }
    close(ch)
}
```

对于接收，使用只接收通道：

```go
func receive(ch <-chan int) {
    for v := range ch{
        fmt.Println(v)
    }
}
```

然后，使用相同的通道，它将自动转换为单向版本：

```go
func main() {
    var a = make(chan int)

    go send(a, 10)

    receive(a)
}
```

完整示例可在[`play.golang.org/p/pPuqpfnq8jJ`](https://play.golang.org/p/pPuqpfnq8jJ)找到。

# 等待接收方

在上一节中，我们看到的大多数示例都是在 goroutine 中完成的发送操作，并且在主 goroutine 中完成了接收操作。可能情况是所有操作都由 goroutine 处理，那么我们如何将主操作与其他操作同步？

一个典型的技术是使用另一个通道，用于唯一的目的是信号一个 goroutine 已经完成了其工作。接收 goroutine 知道通过关闭通信通道没有更多的消息可获取，并在完成操作后关闭与主 goroutine 共享的另一个通道。`main`函数可以在退出之前等待通道关闭。

用于此范围的典型通道除了打开或关闭之外不携带任何其他信息，因此通常是`chan struct{}`通道。这是因为空数据结构在内存中没有大小。我们可以通过对先前示例进行一些更改来看到这种模式的实际应用，从接收函数开始：

```go
func receive(ch <-chan int, done chan<- struct{}) {
    for v := range ch {
        fmt.Println(v)
    }
    close(done)
}
```

接收函数得到了额外的参数——通道。这用于表示发送方已经完成，并且`main`函数将使用该通道等待接收方完成其任务：

```go
func main() {
    a := make(chan int)
    go send(a, 10)
    done := make(chan struct{})
    go receive(a, done)
    <-done
}
```

完整示例可在[`play.golang.org/p/thPflJsnKj4`](https://play.golang.org/p/thPflJsnKj4)找到。

# 特殊值

通道在几种情况下的行为不同。我们现在将看看当通道设置为其零值`nil`时会发生什么，或者当它已经关闭时会发生什么。

# nil 通道

我们之前已经讨论过通道在 Go 中属于指针类型，因此它们的默认值是`nil`。但是当您从`nil`通道发送或接收时会发生什么？

如果我们创建一个非常简单的应用程序，尝试向空通道发送数据，我们会遇到死锁：

```go
func main() {
    var a chan int
    a <- 1
}
```

完整示例可在[`play.golang.org/p/KHJ4rvxh7TM`](https://play.golang.org/p/KHJ4rvxh7TM)找到。

如果我们对接收操作进行相同的操作，我们会得到死锁的相同结果：

```go
func main() {
    var a chan int
    <-a
}
```

完整示例可在[`play.golang.org/p/gIjhy7aMxiR`](https://play.golang.org/p/gIjhy7aMxiR)找到。

最后要检查的是`close`函数在`nil`通道上的行为。它会导致`close of nil channel`的明确值的恐慌：

```go
func main() {
    var a chan int
    close(a)
}
```

完整示例可在[`play.golang.org/p/5RjdcYUHLSL`](https://play.golang.org/p/5RjdcYUHLSL)找到。

总之，我们已经看到`nil`通道的发送和接收是阻塞操作，并且`close`会导致恐慌。

# 关闭通道

我们已经知道从关闭的通道接收会返回通道类型的零值，第二个布尔值为`false`。但是如果我们在关闭通道后尝试发送一些东西会发生什么？让我们通过以下代码来找出：

```go
func main() {
    a := make(chan int)
    close(a)
    a <- 1
}
```

完整示例可在[`play.golang.org/p/_l_xZt1ZojT`](https://play.golang.org/p/_l_xZt1ZojT)找到。

如果我们在关闭后尝试发送数据，将返回一个非常特定的恐慌：`在关闭的通道上发送`。当我们尝试关闭已经关闭的通道时，类似的事情会发生：

```go
func main() {
    a := make(chan int)
    close(a)
    close(a)
}
```

完整示例可在[`play.golang.org/p/GHK7ERt1XQf`](https://play.golang.org/p/GHK7ERt1XQf)找到。

这个示例将导致特定值的恐慌——`关闭已关闭的通道`。

# 管理多个操作

有许多情况下，多个 goroutine 正在执行它们的代码并通过通道进行通信。典型的情况是等待其中一个通道的发送或接收操作被执行。

当您操作多个通道时，Go 使得可以使用一个特殊的关键字来执行类似于`switch`的通道操作。这是通过`select`语句完成的，后面跟着一系列`case`语句和一个可选的`default` case。

我们可以看到一个快速示例，我们在 goroutine 中从一个通道接收值，并在另一个 goroutine 中向另一个通道发送值。在这些示例中，主 goroutine 使用`select`语句与两个通道进行交互，从第一个接收，然后发送到第二个：

```go
func main() {
    ch1, ch2 := make(chan int), make(chan int)
    a, b := 2, 10
    go func() { <-ch1 }()
    go func() { ch2 <- a }()
    select {
    case ch1 <- b:
        fmt.Println("ch1 got a", b)
    case v := <-ch2:
        fmt.Println("ch2 got a", v)
    }
}
```

完整示例可在[`play.golang.org/p/_8P1Edxe3o4`](https://play.golang.org/p/_8P1Edxe3o4)找到。

在 playground 中运行此程序时，我们可以看到从第二个通道的接收操作总是最先完成。如果我们改变 goroutine 的执行顺序，我们会得到相反的结果。最后执行的操作是首先接收的。这是因为 playground 是一个在安全环境中运行和执行 Go 代码的网络服务，并且进行了一些优化以使此操作具有确定性。

# 默认子句

如果我们在上一个示例中添加一个默认情况，应用程序执行的结果将会非常不同，特别是如果我们改变`select`：

```go
select {
case v := <-ch2:
    fmt.Println("ch2 got a", v)
case ch1 <- b:
    fmt.Println("ch1 got a", b)
default:
    fmt.Println("too slow")
}
```

完整的示例可在[`play.golang.org/p/F1aE7ImBNFk`](https://play.golang.org/p/F1aE7ImBNFk)找到。

`select`语句将始终选择`default`语句。这是因为当执行`select`语句时，调度程序尚未选择 goroutine。如果我们在`select`切换之前添加一个非常小的暂停（使用`time.Sleep`），我们将使调度程序至少选择一个 goroutine，然后我们将执行两个操作中的一个：

```go
func main() {
    ch1, ch2 := make(chan int), make(chan int)
    a, b := 2, 10
    for i := 0; i < 10; i++ {
        go func() { <-ch1 }()
        go func() { ch2 <- a }()
        time.Sleep(time.Nanosecond)
        select {
        case ch1 <- b:
            fmt.Println("ch1 got a", b)
        case v := <-ch2:
            fmt.Println("ch2 got a", v)
        default:
            fmt.Println("too slow")
        }
    }
}
```

完整的示例可在[`play.golang.org/p/-aXc3FN6qDj`](https://play.golang.org/p/-aXc3FN6qDj)找到。

在这种情况下，我们将有一组混合的操作被执行，具体取决于哪个操作被 Go 调度程序选中。

# 定时器和滴答器

`time`包提供了一些工具，使得可以编排 goroutines 和 channels——定时器和滴答器。

# 定时器

可以替换`select`语句中的`default`子句的实用程序是`time.Timer`类型。这包含一个只接收通道，在其构造期间使用`time.NewTimer`指定持续时间后将返回一个`time.Time`值：

```go
func main() {
    ch1, ch2 := make(chan int), make(chan int)
    a, b := 2, 10
    go func() { <-ch1 }()
    go func() { ch2 <- a }()
    t := time.NewTimer(time.Nanosecond)
    select {
    case ch1 <- b:
        fmt.Println("ch1 got a", b)
    case v := <-ch2:
        fmt.Println("ch2 got a", v)
    case <-t.C:
        fmt.Println("too slow")
    }
}
```

完整的示例可在[`play.golang.org/p/vCAff1kI4yA`](https://play.golang.org/p/vCAff1kI4yA)找到。

定时器公开一个只读通道，因此无法关闭它。使用`time.NewTimer`创建时，它会在指定的持续时间之前等待在通道中触发一个值。

`Timer.Stop`方法将尝试避免通过通道发送数据并返回是否成功。如果尝试停止定时器后返回`false`，我们仍然需要在能够再次使用通道之前从通道中接收值。

`Timer.Reset`使用给定的持续时间重新启动定时器，并与`Stop`一样返回一个布尔值。这个值要么是`true`要么是`false`：

+   当定时器处于活动状态时为`true`

+   当定时器被触发或停止时为`false`

我们将使用一个实际的示例来测试这些功能：

```go
t := time.NewTimer(time.Millisecond)
time.Sleep(time.Millisecond / 2)
if !t.Stop() {
    panic("it should not fire")
}
select {
case <-t.C:
    panic("not fired")
default:
    fmt.Println("not fired")
}
```

我们正在创建一个新的`1ms`定时器。在这里，我们等待`0.5ms`，然后成功停止它：

```go
if t.Reset(time.Millisecond) {
    panic("timer should not be active")
}
time.Sleep(time.Millisecond)
if t.Stop() {
    panic("it should fire")
}
select {
case <-t.C:
    fmt.Println("fired")
default:
    panic("not fired")
}
```

完整的示例可在[`play.golang.org/p/ddL_fP1UBVv`](https://play.golang.org/p/ddL_fP1UBVv)找到。

然后，我们将定时器重置为`1ms`并等待它触发，以查看`Stop`是否返回`false`并且通道是否被排空。

# AfterFunc

使用`time.Timer`的一个非常有用的实用程序是`time.AfterFunc`函数，它返回一个定时器，当定时器触发时将在其自己的 goroutine 中执行传递的函数：

```go
func main() {
    time.AfterFunc(time.Millisecond, func() {
        fmt.Println("Hello 1!")
    })
    t := time.AfterFunc(time.Millisecond*5, func() {
        fmt.Println("Hello 2!")
    })
    if !t.Stop() {
        panic("should not fire")
    }
    time.Sleep(time.Millisecond * 10)
}
```

完整的示例可在[`play.golang.org/p/77HIIdlRlZ1`](https://play.golang.org/p/77HIIdlRlZ1)找到。

在上一个示例中，我们为两个不同的闭包定义了两个定时器，并停止其中一个，让另一个触发。

# 滴答声

`time.Ticker`类似于`time.Timer`，但其通道以持续时间相等的规则间隔提供更多的元素。它们在创建时使用`time.NewTicker`指定。这使得可以使用`Ticker.Stop`方法停止滴答器的触发：

```go
func main() {
    tick := time.NewTicker(time.Millisecond)
    stop := time.NewTimer(time.Millisecond * 10)
    for {
        select {
        case a := <-tick.C:
            fmt.Println(a)
        case <-stop.C:
            tick.Stop()
        case <-time.After(time.Millisecond):
            return
        }
    }
}
```

完整的示例可在[`play.golang.org/p/8w8I7zIGe-_j`](https://play.golang.org/p/8w8I7zIGe-_j)找到。

在这个例子中，我们还使用了`time.After`——一个从匿名`time.Timer`返回通道的函数。当不需要停止计时器时，可以使用它。还有另一个函数`time.Tick`，它返回匿名`time.Ticker`的通道。这两个函数都会返回一个应用程序无法控制的通道，这个通道最终会被垃圾收集器回收。

这就结束了对通道的概述，从它们的属性和基本用法到一些更高级的并发示例。我们还检查了一些特殊情况以及如何同步多个通道。

# 将通道和 goroutines 结合

现在我们知道了 Go 并发的基本工具和属性，我们可以使用它们来为我们的应用程序构建更好的工具。我们将看到一些利用通道和 goroutines 解决实际问题的示例。

# 速率限制器

一个典型的场景是有一个 Web API 在一定时间内对调用次数有一定限制。这种类型的 API 如果超过阈值，将会暂时阻止使用，使其在一段时间内无法使用。在为 API 创建客户端时，我们需要意识到这一点，并确保我们的应用程序不会过度使用它。

这是一个非常好的场景，我们可以使用`time.Ticker`来定义调用之间的间隔。在这个例子中，我们将创建一个客户端，用于 Google Maps 的地理编码服务，该服务在 24 小时内有 10 万次请求的限制。让我们从定义客户端开始：

```go
type Client struct {
    client *http.Client
    tick *time.Ticker
}
```

客户端由一个 HTTP 客户端组成，它将调用地图，一个 ticker 将帮助防止超过速率限制，并需要一个 API 密钥用于与服务进行身份验证。我们可以为我们的用例定义一个自定义的`Transport`结构，它将在请求中注入密钥，如下所示：

```go
type apiTransport struct {
    http.RoundTripper
    key string
}

func (a apiTransport) RoundTrip(r *http.Request) (*http.Response, error) {
    q := r.URL.Query()
    q.Set("key", a.key)
    r.URL.RawQuery = q.Encode()
    return a.RoundTripper.RoundTrip(r)
}
```

这是一个很好的例子，说明了 Go 接口如何允许扩展自己的行为。我们正在定义一个实现`http.RoundTripper`接口的类型，并且还有一个是相同接口的实例属性。实现在执行底层传输之前将 API 密钥注入请求。这种类型允许我们定义一个帮助函数，创建一个新的客户端，我们在这里使用我们定义的新传输和默认传输一起：

```go
func NewClient(tick time.Duration, key string) *Client {
    return &Client{
        client: &http.Client{
            Transport: apiTransport{http.DefaultTransport, key},
        },
        tick: time.NewTicker(tick),
    }
}
```

地图地理编码 API 返回由各种部分组成的一系列地址。这可以在[`developers.google.com/maps/documentation/geocoding/intro#GeocodingResponses`](https://developers.google.com/maps/documentation/geocoding/intro#GeocodingResponses)找到。

结果以 JSON 格式编码，因此我们需要一个可以接收它的数据结构：

```go
type Result struct {
    AddressComponents []struct {
        LongName string `json:"long_name"`
        ShortName string `json:"short_name"`
        Types []string `json:"types"`
    } `json:"address_components"`
    FormattedAddress string `json:"formatted_address"`
    Geometry struct {
        Location struct {
            Lat float64 `json:"lat"`
            Lng float64 `json:"lng"`
        } `json:"location"`
        // more fields
    } `json:"geometry"`
    PlaceID string `json:"place_id"`
    // more fields
}
```

我们可以使用这个结构来执行反向地理编码操作——通过使用相应的端点从坐标获取位置。在执行 HTTP 请求之前，我们等待 ticker，记得`defer`关闭 body 的闭包：

```go
    const url = "https://maps.googleapis.com/maps/api/geocode/json?latlng=%v,%v"
    <-c.tick.C
    resp, err := c.client.Get(fmt.Sprintf(url, lat, lng))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
```

然后，我们可以解码结果，使用我们已经定义的`Result`类型的数据结构，并检查`status`字符串：

```go
    var v struct {
        Results []Result `json:"results"`
        Status string `json:"status"`
    }
    // get the result
    if err := json.NewDecoder(resp.Body).Decode(&v); err != nil {
        return nil, err
    }
    switch v.Status {
    case "OK":
        return v.Results, nil
    case "ZERO_RESULTS":
        return nil, nil
    default:
        return nil, fmt.Errorf("status: %q", v.Status)
    }
}
```

最后，我们可以使用客户端对一系列坐标进行地理编码，期望请求之间至少相隔`860ms`：

```go
c := NewClient(24*time.Hour/100000, os.Getenv("MAPS_APIKEY"))
start := time.Now()
for _, l := range [][2]float64{
    {40.4216448, -3.6904040},
    {40.4163111, -3.7047328},
    {40.4123388, -3.7096724},
    {40.4145150, -3.7064412},
} {
    locs, err := c.ReverseGeocode(l[0], l[1])
    e := time.Since(start)
    if err != nil {
        log.Println(e, l, err)
        continue
    }
    // just print the first location
    if len(locs) != 0 {
        locs = locs[:1]
    }
    log.Println(e, l, locs)
}
```

# 工作者

前面的例子是一个使用`time.Ticker`通道来限制请求速率的 Google Maps 客户端。速率限制对于 API 密钥是有意义的。假设我们有来自不同账户的更多 API 密钥，那么我们可能可以执行更多的请求。

一个非常典型的并发方法是工作池。在这里，你有一系列的客户端可以被选中来处理输入，应用程序的不同部分可以请求使用这些客户端，在完成后将客户端返回。

我们可以创建多个共享相同通道的客户端，其中请求是坐标，响应是服务的响应。由于响应通道是唯一的，我们可以定义一个自定义类型，其中包含所有需要的通道信息：

```go
type result struct {
    Loc [2]float64
    Result []maps.Result
    Error error
}
```

下一步是创建通道-我们将从环境变量中读取一个逗号分隔的值列表。我们将创建一个用于请求的通道和一个用于响应的通道。这两个通道的容量等于工作人员的数量，在这种情况下，但即使通道是无缓冲的，这也可以工作。由于我们只是使用通道，我们将需要另一个通道“完成”，它表示工作人员是否已完成其最后一项工作：

```go
keys := strings.Split(os.Getenv("MAPS_APIKEYS"), ",")
requests := make(chan [2]float64, len(keys))
results := make(chan result, len(keys))
done := make(chan struct{})
```

现在，我们将为每个密钥创建一个 goroutine，在其中定义一个客户端，该客户端在请求通道上提供数据，执行请求，并将结果发送到专用通道。当请求通道关闭时，goroutine 将退出范围并向“完成”通道发送消息，如下面的代码所示：

```go
for i := range keys {
    go func(id int) {
        log.Printf("Starting worker %d with API key %q", id, keys[id])
        client := maps.NewClient(maps.DailyCap, keys[id])
        for j := range requests {
            var r = result{Loc: j}
            log.Printf("w[%d] working on %v", id, j)
            r.Result, r.Error = client.ReverseGeocode(j[0], j[1])
            results <- r
        }
        done <- struct{}{}
    }(i)
}
```

位置可以按顺序发送到另一个 goroutine 中的请求通道：

```go
go func() {
    for _, l := range [][2]float64{
        {40.4216448, -3.6904040},
        {40.4163111, -3.7047328},
        {40.4123388, -3.7096724},
        {40.4145150, -3.7064412},
    } {
        requests <- l
    }
    close(requests)
}()
```

我们可以统计我们收到的完成信号的数量，并在所有工作人员完成时关闭结果通道：

```go
go func() {
    count := 0
    for range done {
        if count++; count == len(keys) {
            break
        }
    }
    close(results)
}()
```

该通道用于计算有多少工作人员已完成，一旦所有工作人员都已完成，它将关闭结果通道。这将允许我们只需循环遍历它以获取结果：

```go
for r := range results {
    log.Printf("received %v", r)
}
```

使用通道只是等待所有 goroutine 完成的一种方式，我们将在下一章中使用`sync`包看到更多惯用的方法。

# 工作人员池

通道可以用作资源池，允许我们按需请求它们。在以下示例中，我们将创建一个小应用程序，该应用程序将查找在网络中哪些地址是有效的，使用来自`github.com/tatsushid/go-fastping`包的第三方客户端。

该池将有两种方法，一种用于获取新客户端，另一种用于将客户端返回到池中。`Get`方法将尝试从通道中获取现有客户端，如果不可用，则返回一个新客户端。`Put`方法将尝试将客户端放回通道，否则将丢弃它：

```go
const wait = time.Millisecond * 250

type pingPool chan *fastping.Pinger

func (p pingPool) Get() *fastping.Pinger {
    select {
    case v := <-p:
        return v
    case <-time.After(wait):
        return fastping.NewPinger()
    }
}

func (p pingPool) Put(v *fastping.Pinger) {
    select {
    case p <- v:
    case <-time.After(wait):
    }
    return
}
```

客户端将需要指定需要扫描的网络，因此它需要一个从`net.Interfaces`函数开始的可用网络列表，然后遍历接口及其地址：

```go
ifaces, err := net.Interfaces()
if err != nil {
    return nil, err
}
for _, iface := range ifaces {
    // ...
    addrs, err := iface.Addrs()
    // ...
    for _, addr := range addrs {
        var ip net.IP
        switch v := addr.(type) {
        case *net.IPNet:
            ip = v.IP
        case *net.IPAddr:
            ip = v.IP
        }
        // ...
        if ip = ip.To4(); ip != nil {
            result = append(result, ip)
        }
    }
}
```

我们可以接受命令行参数以在接口之间进行选择，并且当参数不存在或错误时，我们可以向用户显示接口列表以进行选择：

```go
if len(os.Args) != 2 {
    help(ifaces)
}
i, err := strconv.Atoi(os.Args[1])
if err != nil {
    log.Fatalln(err)
}
if i < 0 || i > len(ifaces) {
    help(ifaces)
}
```

`help`函数只是一个接口 IP 的打印：

```go
func help(ifaces []net.IP) {
    log.Println("please specify a valid network interface number")
    for i, f := range ifaces {
        mask, _ := f.DefaultMask().Size()
        fmt.Printf("%d - %s/%v\n", i, f, mask)
    }
    os.Exit(0)
}
```

下一步是获取需要检查的 IP 范围：

```go
m := ifaces[i].DefaultMask()
ip := ifaces[i].Mask(m)
log.Printf("Lookup in %s", ip)
```

现在我们有了 IP，我们可以创建一个函数来获取同一网络中的其他 IP。在 Go 中，IP 是一个字节切片，因此我们将替换最低有效位以获得最终地址。由于 IP 是一个切片，其值将被每个操作覆盖（切片是指针）。我们将更新原始 IP 的副本-因为切片是指向相同数组的指针-以避免覆盖：

```go
func makeIP(ip net.IP, i int) net.IP {
    addr := make(net.IP, len(ip))
    copy(addr, ip)
    b := new(big.Int)
    b.SetInt64(int64(i))
    v := b.Bytes()
    copy(addr[len(addr)-len(v):], v)
    return addr
}
```

然后，我们将需要一个用于结果的通道和另一个用于跟踪 goroutine 的通道；对于每个 IP，我们需要检查是否可以为每个地址启动 goroutine。我们将使用 10 个客户端的池，在每个 goroutine 中-我们将为每个客户端请求，然后将它们返回到池中。所有有效的 IP 将通过结果通道发送：

```go
done := make(chan struct{})
address := make(chan net.IP)
ones, bits := m.Size()
pool := make(pingPool, 10)
for i := 0; i < 1<<(uint(bits-ones)); i++ {
    go func(i int) {
        p := pool.Get()
        defer func() {
            pool.Put(p)
            done <- struct{}{}
        }()
        p.AddIPAddr(&net.IPAddr{IP: makeIP(ip, i)})
        p.OnRecv = func(a *net.IPAddr, _ time.Duration) { address <- a.IP }
        p.Run()
    }(i)
}
```

每次一个例程完成时，我们都会在“完成”通道中发送一个值，以便在退出应用程序之前统计接收到的“完成”信号的数量。这将是结果循环：

```go
i = 0
for {
    select {
    case ip := <-address:
        log.Printf("Found %s", ip)
    case <-done:
        if i >= bits-ones {
            return
        }
        i++
    }
}
```

循环将继续，直到通道中的计数达到 goroutine 的数量。这结束了一起使用通道和 goroutine 的更复杂的示例。

# 信号量

信号量是用于解决并发问题的工具。它们具有一定数量的可用配额，用于限制对资源的访问；此外，各种线程可以从中请求一个或多个配额，然后在完成后释放它们。如果可用配额的数量为 1，则意味着信号量一次只支持一个访问，类似于互斥锁的行为。如果配额大于 1，则我们指的是最常见的类型——加权信号量。

在 Go 中，可以使用容量等于配额的通道来实现信号量，其中您向通道发送一条消息以获取配额，并从中接收一条消息以释放配额：

```go
type sem chan struct{}

func (s sem) Acquire() {
    s <- struct{}{}
}

func (s sem) Relase() {
    <-s
}
```

前面的代码向我们展示了如何使用几行代码在通道中实现信号量。以下是如何使用它的示例：

```go
func main() {
    s := make(sem, 5)
    for i := 0; i < 10; i++ {
        go func(i int) {
            s.Acquire()
            fmt.Println(i, "start")
            time.Sleep(time.Second)
            fmt.Println(i, "end")
            s.Relase()
        }(i)
    }
    time.Sleep(time.Second * 3)
}
```

完整示例可在[`play.golang.org/p/BR5GN2QopjQ`](https://play.golang.org/p/BR5GN2QopjQ)中找到。

我们可以从前面的示例中看到，程序在第一轮获取时为一些请求提供服务，而在第二轮获取时为其他请求提供服务，不允许同时执行超过五次。

# 总结

在本章中，我们讨论了 Go 并发中的两个主要角色——goroutines 和通道。我们首先解释了线程是什么，线程和 goroutines 之间的区别，以及它们为什么如此方便。线程很重，需要一个 CPU 核心，而 goroutines 很轻，不绑定到核心。我们看到了一个新的 goroutine 可以通过在函数前加上`go`关键字来轻松启动，并且可以一次启动一系列不同的 goroutines。我们看到了并发函数的参数在创建 goroutine 时进行评估，而不是在实际开始时进行。我们还看到，如果没有额外的工具，很难保持不同的 goroutines 同步。

然后，我们介绍了通道，用于在不同的 goroutines 之间共享信息，并解决我们之前提到的同步问题。我们看到 goroutines 有一个最大容量和一个大小——它目前持有多少元素。大小不能超过容量，当额外的元素发送到一个满的通道时，该操作会阻塞，直到从通道中删除一个元素。从一个空通道接收也是一个阻塞操作。

我们看到了如何使用`close`函数关闭通道，这个操作应该在发送数据的同一个 goroutine 中完成，以及在特殊情况下（如`nil`或关闭的通道）操作的行为。我们介绍了`select`语句来选择并发通道操作并控制应用程序流程。然后，我们介绍了与`time`包相关的并发工具——定时器和计时器。

最后，我们展示了一些真实世界的例子，包括一个速率限制的 Google Maps 客户端和一个工具，可以同时 ping 网络中的所有地址。

在下一章中，我们将研究一些同步原语，这些原语将允许更好地处理 goroutines 和内存，使用更清晰和简单的代码。

# 问题

1.  什么是线程，谁负责它？

1.  为什么 goroutines 与线程不同？

1.  在启动 goroutine 时何时评估参数？

1.  缓冲和非缓冲通道有什么区别？

1.  为什么单向通道有用？

1.  当在`nil`或关闭的通道上进行操作时会发生什么？

1.  计时器和定时器用于什么？
