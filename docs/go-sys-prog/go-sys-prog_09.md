# Goroutines - 基本特性

在上一章中，您学习了 Unix 信号处理，以及在 Go 中添加管道支持和创建图形图像。

这个非常重要的章节的主题是 goroutines。Go 使用 goroutines 和**通道**来以自己的方式编写并发应用程序，同时提供对传统并发技术的支持。Go 中的所有内容都使用 goroutines 执行；当程序开始执行时，其单个 goroutine 会自动调用`main()`函数，以开始程序的实际执行。

在本章中，我们将介绍 goroutines 的简单部分，并提供易于遵循的代码示例。然而，在接下来的第十章*，* *Goroutines - 高级特性*中，我们将讨论与 goroutines 和通道相关的更重要和高级的技术，因此，请确保在阅读下一章之前充分理解本章。

因此，本章将告诉您以下内容：

+   创建 goroutines

+   同步 goroutines

+   关于通道以及如何使用它们

+   读取和写入通道

+   创建和使用管道

+   更改`wc.go`实用程序的 Go 代码，以便在新实现中使用 goroutines

+   进一步改进`wc.go`的 goroutine 版本

# 关于 goroutines

**goroutine**是可以并发执行的最小 Go 实体。请注意，这里使用“最小”一词非常重要，因为 goroutines 不是自主实体。Goroutines 存在于 Unix 进程中的线程中。简单来说，进程可以是自主的并独立存在，而 goroutines 和线程都不行。因此，要创建 goroutine，您需要至少有一个带有线程的进程。好处是 goroutines 比线程轻，线程比进程轻。Go 中的所有内容都使用 goroutines 执行，这是合理的，因为 Go 是一种并发编程语言。正如您刚刚了解的那样，当 Go 程序开始执行时，它的单个 goroutine 调用`main()`函数，从而启动实际的程序执行。

您可以使用`go`关键字后跟函数名或匿名函数的完整定义来定义新的 goroutine。`go`关键字在新的 goroutine 中启动函数参数，并允许调用函数自行继续。

然而，正如您将看到的，您无法控制或做出任何关于 goroutines 将以何种顺序执行的假设，因为这取决于操作系统的调度程序以及操作系统的负载。

# 并发和并行

一个非常常见的误解是**并发**和**并行**指的是同一件事，这与事实相去甚远！并行是多个事物同时执行，而并发是一种构造组件的方式，使它们在可能的情况下可以独立执行。

只有在并发构建时，您才能安全地并行执行它们：当且如果您的操作系统和硬件允许。很久以前，Erlang 编程语言就已经做到了这一点，早在 CPU 拥有多个核心和计算机拥有大量 RAM 之前。

在有效的并发设计中，添加并发实体使整个系统运行更快，因为更多的事情可以并行运行。因此，期望的并行性来自于对问题的更好并发表达和实现。开发人员在系统设计阶段负责考虑并发，并从系统组件的潜在并行执行中受益。因此，开发人员不应该考虑并行性，而应该考虑将事物分解为独立组件，这些组件在组合时解决最初的问题。

即使在 Unix 机器上无法并行运行函数，有效的并发设计仍将改善程序的设计和可维护性。换句话说，并发比并行更好！

# 同步 Go 包

`sync` Go 包包含可以帮助您同步 goroutines 的函数；`sync`的最重要的函数是`sync.Add`、`sync.Done`和`sync.Wait`。对于每个程序员来说，同步 goroutines 是一项必不可少的任务。

请注意，goroutines 的同步与共享变量和共享状态无关。共享变量和共享状态与您希望用于执行并发交互的方法有关。

# 一个简单的例子

在这一小节中，我们将介绍一个简单的程序，它创建了两个 goroutines。示例程序的名称将是`aGoroutine.go`，将分为三个部分；第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "time" 
) 

func namedFunction() { 
   time.Sleep(10000 * time.Microsecond) 
   fmt.Println("Printing from namedFunction!") 
} 
```

除了预期的`package`和`import`语句之外，您还可以看到一个名为`namedFunction()`的函数的实现，在打印屏幕上的消息之前会休眠一段时间。

`aGoroutine.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   fmt.Println("Chapter 09 - Goroutines.") 
   go namedFunction() 
```

在这里，您创建了一个执行`namedFunction()`函数的 goroutine。这个天真程序的最后部分如下：

```go
   go func() { 
         fmt.Println("An anonymous function!") 
   }() 

   time.Sleep(10000 * time.Microsecond) 
   fmt.Println("Exiting...") 
} 
```

在这里，您创建了另一个 goroutine，它执行一个包含单个`fmt.Println()`语句的匿名函数。

正如您所看到的，以这种方式运行的 goroutines 是完全隔离的，彼此之间无法交换任何类型的数据，这并不总是所期望的操作风格。

如果您忘记在`main()`函数中调用`time.Sleep()`函数，或者`time.Sleep()`睡眠了很短的时间，那么`main()`将会过早地结束，两个 goroutines 将没有足够的时间开始和完成它们的工作；结果，您将无法在屏幕上看到所有预期的输出！

执行`aGoroutine.go`将生成以下输出：

```go
$ go run aGoroutine.go
Chapter 09 - Goroutines.
Printing from namedFunction!
Exiting... 
```

# 创建多个 goroutines

这一小节将向您展示如何创建许多 goroutines 以及处理更多 goroutines 所带来的问题。程序的名称将是`moreGoroutines.go`，将分为三个部分。

`moreGoroutines.go`的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "time" 
) 
```

程序的第二部分包含以下 Go 代码：

```go
func main() { 
   fmt.Println("Chapter 09 - Goroutines.") 

   for i := 0; i < 10; i++ { 
         go func(x int) { 
               time.Sleep(10) 
               fmt.Printf("%d ", x) 
         }(i) 
   } 
```

这次，匿名函数接受一个名为`x`的参数，其值为变量`i`。使用变量`i`的`for`循环依次创建十个 goroutines。

程序的最后部分如下：

```go
   time.Sleep(10000) 
   fmt.Println("Exiting...") 
} 
```

再次，如果您将较小的值作为`time.Sleep()`的参数，当您执行程序时将会看到不同的结果。

执行`moreGoroutines.go`将生成一个有些奇怪的输出：

```go
$ go run moreGoroutines.go
Chapter 09 - Goroutines.
1 7 Exiting...
2 3
```

然而，当您多次执行`moreGoroutines.go`时，大惊喜来了：

```go
$ go run moreGoroutines.go
Chapter 09 - Goroutines.
Exiting...
$ go run moreGoroutines.go
Chapter 09 - Goroutines.
3 1 0 9 2 Exiting...
4 5 6 8 7
$ go run moreGoroutines.go
Chapter 09 - Goroutines.
2 0 1 8 7 3 6 5 Exiting...
4
```

正如您所看到的，程序的所有先前输出都与第一个不同！因此，输出不仅不协调，而且并不总是有足够的时间让所有 goroutines 执行；您无法确定 goroutines 将以何种顺序执行。然而，尽管您无法解决后一个问题，因为 goroutines 的执行顺序取决于开发人员无法控制的各种参数，下一小节将教您如何同步 goroutines 并为它们提供足够的时间完成，而无需调用`time.Sleep()`。

# 等待 goroutines 完成它们的工作

这一小节将向您演示正确的方法来创建一个等待其 goroutines 完成工作的调用函数。程序的名称将是`waitGR.go`，将分为四个部分；第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "sync" 
) 
```

除了`time`包的缺失和`sync`包的添加之外，这里没有什么特别的。

第二部分包含以下 Go 代码：

```go
func main() { 
   fmt.Println("Waiting for Goroutines!") 

   var waitGroup sync.WaitGroup 
   waitGroup.Add(10) 
```

在这里，您创建了一个新变量，类型为`sync.WaitGroup`，它等待一组 goroutines 完成。属于该组的 goroutines 的数量由一个或多个对`sync.Add()`函数的调用定义。

在 Go 语句之前调用`sync.Add()`以防止竞争条件是很重要的。

另外，`sync.Add(10)`的调用告诉我们的程序，我们将等待十个 goroutines 完成。

程序的第三部分如下：

```go
   var i int64 
   for i = 0; i < 10; i++ { 

         go func(x int64) { 
               defer waitGroup.Done() 
               fmt.Printf("%d ", x) 
         }(i) 
   } 
```

在这里，您可以使用`for`循环创建所需数量的 goroutines，但也可以使用多个顺序的 Go 语句。当每个 goroutine 完成其工作时，将执行`sync.Done()`函数：在函数定义之后立即使用`defer`关键字告诉匿名函数在完成之前自动调用`sync.Done()`。

`waitGR.go`的最后一部分如下：

```go
   waitGroup.Wait() 
   fmt.Println("\nExiting...") 
} 
```

这里的好处是不需要调用`time.Sleep()`，因为`sync.Wait()`会为我们做必要的等待。

再次应该注意的是，您不应该对 goroutines 的执行顺序做任何假设，这也由以下输出验证：

```go
$ go run waitGR.go
Waiting for Goroutines!
9 0 5 6 7 8 2 1 3 4
Exiting...
$ go run waitGR.go
Waiting for Goroutines!
9 0 5 6 7 8 3 1 2 4
Exiting...
$ go run waitGR.go
Waiting for Goroutines!
9 5 6 7 8 1 0 2 3 4
Exiting...
```

如果您调用`waitGroup.Add()`的次数超过所需次数，当执行`waitGR.go`时，将收到以下错误消息：

```go
Waiting for Goroutines!
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0xc42000e28c)
      /usr/local/Cellar/go/1.8.3/libexec/src/runtime/sema.go:47 +0x34
sync.(*WaitGroup).Wait(0xc42000e280)
      /usr/local/Cellar/go/1.8.3/libexec/src/sync/waitgroup.go:131 +0x7a
main.main()
      /Users/mtsouk/ch/ch9/code/waitGR.go:22 +0x13c
exit status 2
9 0 1 2 6 7 8 3 4 5
```

这是因为当您告诉程序通过调用`sync.Add(1)` n+1 次来等待 n+1 个 goroutines 时，您的程序不能只有 n 个 goroutines（或更少）！简单地说，这将使`sync.Wait()`无限期地等待一个或多个 goroutines 调用`sync.Done()`而没有任何运气，这显然是一个死锁的情况，阻止您的程序完成。

# 创建动态数量的 goroutines

这次，将作为命令行参数给出要创建的 goroutines 的数量：程序的名称将是`dynamicGR.go`，并将分为四个部分。

`dynamicGR.go`的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "path/filepath" 
   "strconv" 
   "sync" 
) 
```

`dynamicGR.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Printf("usage: %s integer\n",filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   numGR, _ := strconv.ParseInt(os.Args[1], 10, 64) 
   fmt.Printf("Going to create %d goroutines.\n", numGR) 
   var waitGroup sync.WaitGroup 

   var i int64 
   for i = 0; i < numGR; i++ { 
         waitGroup.Add(1) 
```

正如您所看到的，`waitGroup.Add(1)`语句是在创建新的 goroutine 之前调用的。

`dynamicGR.go`的 Go 代码的第三部分如下：

```go
         go func(x int64) { 
               defer waitGroup.Done() 
               fmt.Printf(" %d ", x) 
         }(i) 
   } 
```

在前面的部分中，创建了每个简单的 goroutine。

程序的最后一部分如下：

```go
   waitGroup.Wait() 
   fmt.Println("\nExiting...") 
} 
```

在这里，您只需告诉程序使用`waitGroup.Wait()`语句等待所有 goroutines 完成。

执行`dynamicGR.go`需要一个整数参数，这是您想要创建的 goroutines 的数量：

```go
$ go run dynamicGR.go 15
Going to create 15 goroutines.
 0  2  4  1  3  5  14  10  8  9  12  11  6  13  7
Exiting...
$ go run dynamicGR.go 15
Going to create 15 goroutines.
 5  3  14  4  10  6  7  11  8  9  12  2  13  1  0
Exiting...
$ go run dynamicGR.go 15
Going to create 15 goroutines.
 4  2  3  6  5  10  9  7  0  12  11  1  14  13  8
Exiting...
```

可以想象，您想要创建的 goroutines 越多，输出就会越多样化，因为没有办法控制程序的 goroutines 执行顺序。

# 关于通道

**通道**，简单地说，是一种通信机制，允许 goroutines 交换数据。但是，这里存在一些规则。首先，每个通道允许特定数据类型的交换，这也称为通道的**元素类型**，其次，为了使通道正常运行，您需要使用一些 Go 代码来接收通过通道发送的内容。

您应该使用`chan`关键字声明一个新的通道，并且可以使用`close()`函数关闭一个通道。此外，由于每个通道都有自己的类型，开发人员应该定义它。

最后，一个非常重要的细节：当您将通道用作函数参数时，可以指定其方向，即它将用于写入还是读取。在我看来，如果您事先知道通道的目的，请使用此功能，因为它将使您的程序更健壮，更安全：否则，只需不定义通道函数参数的目的。因此，如果您声明通道函数参数仅用于读取，并尝试向其写入，您将收到一个错误消息，这很可能会使您免受讨厌的错误。

当你尝试从写通道中读取时，你将得到以下类似的错误消息：

```go
# command-line-arguments
./writeChannel.go:13: invalid operation: <-c (receive from send-only type chan<- int)
```

# 向通道写入

在本小节中，你将学习如何向通道写入。所呈现的程序将被称为`writeChannel.go`，并分为三个部分。

第一部分包含了预期的序言：

```go
package main 

import ( 
   "fmt" 
   "time" 
) 
```

正如你所理解的，使用通道不需要任何额外的 Go 包。

`writeChannel.go`的第二部分如下：

```go
func writeChannel(c chan<- int, x int) { 
   fmt.Println(x) 
   c <- x 
   close(c) 
   fmt.Println(x) 
} 
```

尽管`writeChannel()`函数向通道写入数据，但由于当前没有人从程序中读取通道，数据将丢失。

程序的最后一部分包含以下 Go 代码：

```go
func main() { 
   c := make(chan int) 
   go writeChannel(c, 10) 
   time.Sleep(2 * time.Second) 
} 
```

在这里，你可以看到使用`chan`关键字定义了一个名为`c`的通道变量，用于`int`数据。

执行`writeChannel.go`将创建以下输出：

```go
 $ go run writeChannel.go
 10
```

这不是你期望看到的！这个意外的输出的原因是第二个`fmt.Println(x)`语句没有被执行。原因很简单：`c <- x`语句阻塞了`writeChannel()`函数的其余部分的执行，因为没有人从`c`通道中读取。

# 从通道中读取

本小节将通过允许你从通道中读取来改进`writeChannel.go`的 Go 代码。所呈现的程序将被称为`readChannel.go`，并分为四个部分呈现。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "time" 
) 
```

`readChannel.go`的第二部分包含以下 Go 代码：

```go
func writeChannel(c chan<- int, x int) { 
   fmt.Println(x) 
   c <- x 
   close(c) 
   fmt.Println(x) 
} 
```

再次注意，如果没有人收集写入通道的数据，发送数据的函数将在等待有人读取其数据时停滞。然而，在第十章*，* *Goroutines - Advanced Features*中，你将看到这个问题的一个非常好的解决方案。

第三部分包含以下 Go 代码：

```go
func main() { 
   c := make(chan int) 
   go writeChannel(c, 10) 
   time.Sleep(2 * time.Second) 
   fmt.Println("Read:", <-c) 
   time.Sleep(2 * time.Second) 
```

在这里，`fmt.Println()`函数中的`<-c`语句用于从通道中读取单个值：相同的语句也可以用于将通道的值存储到变量中。然而，如果你不存储从通道中读取的值，它将会丢失。

`readChannel.go`的最后一部分如下：

```go
   _, ok := <-c 
   if ok { 
         fmt.Println("Channel is open!") 
   } else { 
         fmt.Println("Channel is closed!") 
   } 
} 
```

在这里，你看到了一种技术，可以让你知道你想要从中读取的通道是否已关闭。然而，如果通道是打开的，所呈现的 Go 代码将因为在赋值中使用了`_`字符而丢弃通道的读取值。

执行`readChannel.go`将创建以下输出：

```go
$ go run readChannel.go
10
Read: 10
10
Channel is closed!
$ go run readChannel.go
10
10
Read: 10
Channel is closed!
```

# 解释 h1s.go

在第八章*，* *Processes and Signals*中，你看到了 Go 如何使用许多示例处理 Unix 信号，包括`h1s.go`。然而，现在你更了解 goroutines 和通道，是时候更详细地解释一下`h1s.go`的 Go 代码了。

正如你已经知道的，`h1s.go`使用通道和 goroutines，现在应该清楚了，作为 goroutine 执行的匿名函数使用无限的`for`循环从`sigs`通道读取。这意味着每次有我们感兴趣的信号时，goroutine 都会从`sigs`通道中读取并处理它。

# 管道

Go 程序很少使用单个通道。一个非常常见的使用多个通道的技术称为**pipeline**。因此，pipeline 是一种连接 goroutines 的方法，使得一个 goroutine 的输出成为另一个 goroutine 的输入，借助通道。使用 pipeline 的好处如下：

+   使用 pipeline 的好处之一是程序中有一个恒定的流动，因为没有人等待所有事情都完成才开始执行程序的 goroutines 和通道

+   此外，你使用的变量更少，因此占用的内存空间也更少，因为你不必保存所有东西。

+   最后，使用管道简化了程序的设计并提高了可维护性

`pipelines.go`的代码将以五个部分呈现；第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "path/filepath" 
   "strconv" 
) 
```

第二部分包含以下 Go 代码：

```go
func genNumbers(min, max int64, out chan<- int64) { 

   var i int64 
   for i = min; i <= max; i++ { 
         out <- i 
   } 
   close(out) 
} 
```

在这里，您定义了一个函数，它接受三个参数：两个整数和一个输出通道。输出通道将用于写入将在另一个函数中读取的数据：这就是创建管道的方式。

程序的第三部分如下：

```go
func findSquares(out chan<- int64, in <-chan int64) { 
   for x := range in { 
         out <- x * x 
   } 
   close(out) 
} 
```

这次，函数接受两个都是通道的参数。但是，`out`是一个输出通道，而`in`是一个用于读取数据的输入通道。

第四部分包含另一个函数的定义：

```go
func calcSum(in <-chan int64) { 
   var sum int64 
   sum = 0 
   for x2 := range in { 
         sum = sum + x2 
   } 
   fmt.Printf("The sum of squares is %d\n", sum) 
} 
```

`pipelines.go`的最后一个函数只接受一个用于读取数据的通道作为参数。

`pipelines.go`的最后一部分是`main()`函数的实现：

```go
func main() { 
   if len(os.Args) != 3 { 
         fmt.Printf("usage: %s n1 n2\n", filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 
   n1, _ := strconv.ParseInt(os.Args[1], 10, 64) 
   n2, _ := strconv.ParseInt(os.Args[2], 10, 64) 

   if n1 > n2 { 
         fmt.Printf("%d should be smaller than %d\n", n1, n2) 
         os.Exit(10) 
   } 

   naturals := make(chan int64) 
   squares := make(chan int64) 
   go genNumbers(n1, n2, naturals) 
   go findSquares(squares, naturals) 
   calcSum(squares) 
} 
```

在这里，`main()`函数首先读取其两个命令行参数并创建必要的通道变量（`naturals`和`squares`）。然后，它调用管道的函数：请注意，通道的最后一个函数不会作为 goroutine 执行。

以下图显示了`pipelines.go`中使用的管道的图形表示，以说明特定管道的工作方式：

![](img/e6d2874d-12a1-4441-b5a1-f2af5c0056fe.png)

pipelines.go 中使用的管道结构的图形表示

运行`pipelines.go`将生成以下输出：

```go
$ go run pipelines.go
usage: pipelines n1 n2
exit status 1
$ go run pipelines.go 3 2
3 should be smaller than 2
exit status 10
$ go run pipelines.go 3 20
The sum of squares is 2865
$ go run pipelines.go 1 20
The sum of squares is 2870
$ go run pipelines.go 20 20
The sum of squares is 400
```

# wc.go 的更好版本

正如我们在第六章中讨论的，在本章中，您将学习如何创建一个使用 goroutines 的`wc.go`的版本。新实用程序的名称将是`dWC.go`，将分为四个部分。请注意，`dWC.go`的当前版本将每个命令行参数都视为一个文件。

实用程序的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "io" 
   "os" 
   "path/filepath" 
   "regexp" 
   "sync" 
) 
```

第二部分包含以下 Go 代码：

```go
func count(filename string) { 
   var err error 
   var numberOfLines int = 0 
   var numberOfCharacters int = 0 
   var numberOfWords int = 0 

   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Printf("%s\n", err) 
         return 
   } 
   defer f.Close() 

   r := bufio.NewReader(f) 
   for { 
         line, err := r.ReadString('\n') 

         if err == io.EOF { 
               break 
         } else if err != nil { 
               fmt.Printf("error reading file %s\n", err) 
         } 
         numberOfLines++ 
         r := regexp.MustCompile("[^\\s]+") 
         for range r.FindAllString(line, -1) { 
               numberOfWords++ 
         } 
         numberOfCharacters += len(line) 
   } 

   fmt.Printf("\t%d\t", numberOfLines) 
   fmt.Printf("%d\t", numberOfWords) 
   fmt.Printf("%d\t", numberOfCharacters) 
   fmt.Printf("%s\n", filename) 
} 
```

`count()`函数完成所有处理，而不向`main()`函数返回任何信息：它只是打印其输入文件的行数、单词数和字符数，然后退出。尽管`count()`函数的当前实现完成了所需的工作，但这并不是设计程序的正确方式，因为无法控制程序的输出。

实用程序的第三部分如下：

```go
func main() { 
   if len(os.Args) == 1 { 
         fmt.Printf("usage: %s <file1> [<file2> [... <fileN]]\n", 
               filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 
```

`dWC.go`的最后一部分如下：

```go
   var waitGroup sync.WaitGroup 
   for _, filename := range os.Args[1:] { 
         waitGroup.Add(1) 
         go func(filename string) { 
               count(filename) 
               defer waitGroup.Done() 
         }(filename) 
   } 
   waitGroup.Wait() 
} 
```

正如您所看到的，每个输入文件都由不同的 goroutine 处理。如预期的那样，您无法对输入文件的处理顺序做出任何假设。

执行`dWC.go`将生成以下输出：

```go
$ go run dWC.go /tmp/swtag.log /tmp/swtag.log doesnotExist
open doesnotExist: no such file or directory
          48    275   3571  /tmp/swtag.log
          48    275   3571  /tmp/swtag.log

```

在这里，您可以看到，尽管`doesnotExist`文件名是最后一个命令行参数，但它是`dWC.go`输出中的第一个命令行参数！

尽管`dWC.go`使用了 goroutines，但其中并没有巧妙之处，因为 goroutines 在没有相互通信和执行任何其他任务的情况下运行。此外，输出可能会混乱，因为无法保证`count()`函数的`fmt.Printf()`语句不会被中断。

因此，即将呈现的部分以及将在第十章中呈现的一些技术，即*Goroutines - 高级特性*，将改进`dWC.go`。

# 计算总数

`dWC.go`的当前版本无法计算总数，可以通过使用`awk`处理`dWC.go`的输出来轻松解决：

```go
$ go run dWC.go /tmp/swtag.log /tmp/swtag.log | awk '{sum1+=$1; sum2+=$2; sum3+=$3} END {print "\t", sum1, "\t", sum2, "\t", sum3}'
       96    550   7142

```

然而，这离完美和优雅还有很大差距！

`dWC.go`的当前版本无法计算总数的主要原因是其 goroutines 无法相互通信。这可以通过通道和管道的帮助轻松解决。新版本的`dWC.go`将被称为`dWCtotal.go`，将分为五个部分呈现。

`dWCtotal.go`的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "io" 
   "os" 
   "path/filepath" 
   "regexp" 
) 

type File struct { 
   Filename   string 
   Lines      int 
   Words      int 
   Characters int 
   Error      error 
} 
```

在这里，定义了一个新的`struct`类型。新结构称为`File`，有四个字段和一个额外的字段用于保存错误消息。这是管道循环多个值的正确方式。有人可能会认为`File`结构的更好名称应该是`Counts`、`Results`、`FileCounts`或`FileResults`。

程序的第二部分如下：

```go
func process(files []string, out chan<- File) { 
   for _, filename := range files { 
         var fileToProcess File 
         fileToProcess.Filename = filename 
         fileToProcess.Lines = 0 
         fileToProcess.Words = 0 
         fileToProcess.Characters = 0 
         out <- fileToProcess 
   } 
   close(out) 
} 
```

`process()`函数的更好名称应该是`beginProcess()`或`processResults()`。您可以尝试在整个`dWCtotal.go`程序中自行进行更改。

`dWCtotal.go`的第三部分包含以下 Go 代码：

```go
func count(in <-chan File, out chan<- File) { 
   for y := range in { 
         filename := y.Filename 
         f, err := os.Open(filename) 
         if err != nil { 
               y.Error = err 
               out <- y 
               continue 
         } 
         defer f.Close() 
         r := bufio.NewReader(f) 
         for { 
               line, err := r.ReadString('\n') 
               if err == io.EOF { 
                     break 
               } else if err != nil { 
                     fmt.Printf("error reading file %s", err) 
                     y.Error = err 
                     out <- y 
                     continue 
               } 
               y.Lines = y.Lines + 1 
               r := regexp.MustCompile("[^\\s]+") 
               for range r.FindAllString(line, -1) { 
                     y.Words = y.Words + 1 
               } 
               y.Characters = y.Characters + len(line) 
         } 
         out <- y 
   } 
   close(out) 
} 
```

尽管`count()`函数仍然计算计数，但它不会打印它们。它只是使用`File`类型的`struct`变量将行数、单词数、字符数以及文件名发送到另一个通道。

这里有一个非常重要的细节，就是`count()`函数的最后一条语句：为了正确结束管道，您应该关闭所有涉及的通道，从第一个开始。否则，程序的执行将以类似以下的错误消息失败：

```go
fatal error: all goroutines are asleep - deadlock!
```

然而，就关闭管道的管道而言，您还应该注意不要过早关闭通道，特别是在管道中存在分支时。

程序的第四部分包含以下 Go 代码：

```go
func calculate(in <-chan File) { 
   var totalWords int = 0 
   var totalLines int = 0 
   var totalChars int = 0 
   for x := range in { 
         totalWords = totalWords + x.Words 
         totalLines = totalLines + x.Lines 
         totalChars = totalChars + x.Characters 
         if x.Error == nil { 
               fmt.Printf("\t%d\t", x.Lines) 
               fmt.Printf("%d\t", x.Words) 
               fmt.Printf("%d\t", x.Characters) 
               fmt.Printf("%s\n", x.Filename) 
         } 
   } 

   fmt.Printf("\t%d\t", totalLines) 
   fmt.Printf("%d\t", totalWords) 
   fmt.Printf("%d\ttotal\n", totalChars) 
} 
```

这里没有什么特别的：`calculate()`函数负责打印程序的输出。

`dWCtotal.go`的最后部分如下：

```go
func main() { 
   if len(os.Args) == 1 { 
         fmt.Printf("usage: %s <file1> [<file2> [... <fileN]]\n", 
               filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   files := make(chan File)
   values := make(chan File) 

   go process(os.Args[1:], files) 
   go count(files, values) 
   calculate(values) 
} 
```

由于`files`通道仅用于传递文件名，它本可以是一个`string`通道，而不是一个`File`通道。但是，这样代码更一致。

现在`dWCtotal.go`即使只处理一个文件也会自动生成总数：

```go
$ go run dWCtotal.go /tmp/swtag.log
      48    275   3571  /tmp/swtag.log
      48    275   3571  total
$ go run dWCtotal.go /tmp/swtag.log /tmp/swtag.log doesNotExist
      48    275   3571  /tmp/swtag.log
      48    275   3571  /tmp/swtag.log
      96    550   7142  total
```

请注意，`dWCtotal.go`和`dWC.go`都实现了相同的核心功能，即计算文件的单词、字符和行数：不同之处在于信息处理的方式不同，因为`dWCtotal.go`使用了管道而不是孤立的 goroutines。

第十章*，* *Goroutines - Advanced Features*，将使用其他技术来实现`dWCtotal.go`的功能。

# 进行一些基准测试

在本节中，我们将比较第六章*，* *文件输入和输出*中的`wc.go`与`wc(1)`、`dWC.go`和`dWCtotal.go`的性能。为了使结果更准确，所有三个实用程序将处理相对较大的文件：

```go
$ wc /tmp/*.data
  712804 3564024 9979897 /tmp/connections.data
  285316  855948 4400685 /tmp/diskSpace.data
  712523 1425046 8916670 /tmp/memory.data
 1425500 2851000 5702000 /tmp/pageFaults.data
  285658  840622 4313833 /tmp/uptime.data
 3421801 9536640 33313085 total

```

因此，`time(1)`实用程序将测量以下命令：

```go
$ time wc /tmp/*.data /tmp/*.data
$ time wc /tmp/uptime.data /tmp/pageFaults.data
$ time ./dWC /tmp/*.data /tmp/*.data
$ time ./dWC /tmp/uptime.data /tmp/pageFaults.data
$ time ./dWCtotal /tmp/*.data /tmp/*.data
$ time ./dWCtotal /tmp/uptime.data /tmp/pageFaults.data
$ time ./wc /tmp/uptime.data /tmp/pageFaults.data
$ time ./wc /tmp/*.data /tmp/*.data
```

以下图显示了使用`time(1)`实用程序测量上述命令时的实际领域的图形表示：

![](img/de995173-8729-4a8b-8960-775d6436074c.png)

绘制`time(1)`实用程序的实际领域

原始的`wc(1)`实用程序是迄今为止最快的。此外，`dWC.go`比`dWCtotal.go`和`wc.go`都要快。除了`dWC.go`，其余两个 Go 版本的性能相同。

# 练习

1.  创建一个管道，读取文本文件，找到给定单词的出现次数，并计算所有文件中该单词的总出现次数。

1.  尝试让`dWCtotal.go`更快。

1.  创建一个简单的 Go 程序，使用通道进行乒乓球比赛。您应该使用命令行参数定义乒乓球的总数。

# 总结

在本章中，我们讨论了创建和同步 goroutines，以及创建和使用管道和通道，以使 goroutines 能够相互通信。此外，我们开发了两个使用 goroutines 处理其输入文件的`wc(1)`实用程序的版本。

在继续下一章之前，请确保您充分理解本章的概念，因为在下一章中，我们将讨论与 goroutines 和通道相关的更高级特性，包括共享内存、缓冲通道、`select`关键字、`GOMAXPROCS`环境变量和信号通道。
