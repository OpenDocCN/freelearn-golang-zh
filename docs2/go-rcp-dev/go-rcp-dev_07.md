# 7

# 并发

并发是 Go 语言的核心部分。与许多其他通过丰富的多线程库支持并发的语言不同，Go 提供了相对较少的语言原语来编写并发程序。

首先，强调并发**不是**并行。并发是关于你如何编写程序；并行是关于程序如何运行。一个并发程序指定了程序中哪些部分可以并行运行。根据实际执行情况，程序中的并发部分可能按顺序运行，使用时间共享并发运行，或者并行运行。一个正确的并发程序无论以何种方式运行都会产生相同的结果。

本章通过食谱介绍了 Go 并发原语的一些内容。在本章中，您将学习以下内容：

+   创建 goroutines

+   并行运行多个独立函数并等待它们结束

+   使用通道发送和接收数据

+   从多个 goroutine 向通道发送数据

+   使用通道收集并发计算的结果

+   使用 `select` 语句处理多个通道

+   取消一个 goroutine

+   使用非阻塞 `select` 检测取消

+   并发更新共享变量

# 使用 goroutine 进行并发操作

Goroutine 是一个与其他 goroutine 并行运行的函数。当程序启动时，Go 运行时会创建几个 goroutines。其中之一运行垃圾回收器。另一个运行 `main` 函数。随着程序的执行，它会根据需要创建更多的 goroutines。一个典型的 Go 程序可能有数千个并发运行的 goroutines。Go 运行时会将这些 goroutines 调度到操作系统线程。每个操作系统线程都会分配一定数量的 goroutines，并使用时间共享来运行它们。在任何给定时刻，活跃的 goroutine 数量可以与逻辑处理器的数量相同：

```go
Number of threads per core * Number of cores per CPU * Number of CPUs
```

## 创建 goroutines

Goroutines 是 Go 语言的一个基本组成部分。您可以使用 `go` 关键字创建 goroutines。

### 如何做到这一点...

使用 `go` 关键字后跟函数调用来创建 goroutines：

```go
func f() {
  // Do some work
}
func main() {
  go f()
  ...
}
```

当 `go f()` 被评估时，运行时会创建一个新的 goroutine 并调用 `f` 函数。运行 `main` 的 goroutine 也会继续运行。换句话说，当 `go` 关键字被评估时，程序执行会分为两个并发执行流——一个是原始执行流（在前面的例子中，运行 `main` 的流）和另一个运行 `go` 关键字后面的函数。

函数可以根据需要接受参数：

```go
func f(i int) {
  // Do some work
}
func main() {
  var x int
  go f(x)
  ...
}
```

函数的参数在 goroutine 开始之前被评估。也就是说，`main` goroutine 首先评估 `f` 的参数（在这种情况下，是 `x` 值），然后创建一个新的 goroutine 并运行 `f`。

使用闭包运行 goroutine 是一种常见的做法。它们提供了理解代码所需的上下文。它们还防止将许多变量作为参数传递给 goroutine：

```go
func main() {
  var x int
  var y int
  ...
  go func(i int) {
    if y > 0 {
      // Do some work
    }
  }(x)
  ...
}
```

在这里，`x` 作为参数传递给 goroutine，但 `y` 被捕获。

当使用 `go` 关键字运行的函数结束时，goroutine 将终止。

## 并发运行多个独立函数并等待它们完成

当你有多个不共享数据的独立函数时，你可以使用此食谱来并发运行它们。我们还将使用 `sync.WaitGroup` 来等待 goroutine 完成。

### 如何实现...

1.  创建一个 `sync.WaitGroup` 实例以等待 goroutine：

    ```go
    wg := sync.WaitGroup{}
    ```

    `sync.WaitGroup` 简单来说是一个线程安全的计数器。我们将为每个创建的 goroutine 使用 `wg.Add(1)`，并在每个 goroutine 结束时使用 `wg.Done()` 减去 1。然后我们可以等待等待组达到零，这表示所有 goroutine 的终止。

1.  对于每个将要并发运行的函数，执行以下操作：

    +   向等待组添加 1

    +   启动一个新的 goroutine

    +   调用 `defer wg.Done()` 确保你发出 goroutine 终止的信号

        ```go
        wg.Add(1)
        go func() {
          defer wg.Done()
          // Do work
        }()
        ```

小贴士

你可以不向等待组为每个 goroutine 添加 1，而只需添加 goroutine 的数量。例如，如果你知道你将创建 5 个 goroutine，你可以在创建第一个 goroutine 之前简单地做 `wg.Add(5)`。

1.  等待 goroutine 结束：

    ```go
    wg.Wait()
    ```

    此调用将阻塞，直到 `wg` 达到零，即直到所有 goroutine 调用 `wg.Done()`。

1.  现在，你可以使用所有 goroutine 的结果。

    此食谱的关键细节是所有 goroutine 都是独立的，这意味着以下内容：

    每个 goroutine 编写的所有变量都仅由该 goroutine 使用，直到 `wg.Done()`。goroutine 可以读取共享变量，但不能写入它们。在 `wg.Done()` 之后，所有 goroutine 都将终止，它们所写入的变量可以被使用。

1.  没有 goroutine 依赖于另一个 goroutine 的结果。

在 `wg.Wait` 之前，你不应该尝试读取 goroutine 的结果。这是一个具有未定义行为的内存竞争。

当你与其他写入或读取同时写入共享变量时，会发生**内存竞争**。包含内存竞争的程序的结果是未定义的。

# 使用通道在 goroutine 之间进行通信

更多的时候，多个 goroutine 需要通信和协调以分配工作、管理状态和汇总计算结果。通道是这种情况下首选的机制。通道是一种带有可选固定大小缓冲区的同步机制。

小贴士：

以下示例展示了已关闭的通道。关闭通道是一种传达数据结束的方法。如果你不需要向接收者传达数据结束的信号，则不需要关闭通道。

## 使用通道发送和接收数据

如果有另一个 goroutine 正在等待从它接收数据，或者在有缓冲的 channel 中，channel 缓冲区中有空间，那么 goroutine 可以向 channel 发送数据。否则，goroutine 将被阻塞，直到它可以发送。

如果有另一个 goroutine 正在等待向它发送数据，或者在有缓冲的 channel 中，channel 缓冲区中有数据，那么 goroutine 可以从 channel 接收数据。否则，接收器将被阻塞，直到它可以接收数据。

### 如何做到这一点...

1.  创建一个传递数据的 channel 类型。以下示例创建了一个可以传递字符串的 channel。

    ```go
    ch := make(chan string)
    ```

1.  在 goroutine 中，将数据元素发送到 channel。当所有数据元素都发送完毕后，关闭 channel：

    ```go
    go func() {
      for _, str := range stringData {
         // Send the string to the channel. This will block until
         // another goroutine can receive from the channel.
         ch <- str
      }
      // Close the channel when done. This is the way to signal the
      // receiver goroutine that there is no more data available.
      close(ch)
    }()
    ```

1.  在另一个 goroutine 中从 channel 接收数据。在以下示例中，主 goroutine 从 channel 接收字符串并打印它们。`for`循环在 channel 关闭时结束：

    ```go
    for str := range ch {
      fmt.Println(str)
    }
    ```

## 从多个 goroutines 向 channel 发送数据

有时候，你有很多 goroutine 在处理问题的某个部分，当它们完成后，它们会通过 channel 发送结果。这种情况的一个问题是决定何时关闭 channel。这个方法展示了如何做到这一点。

### 如何做到这一点...

1.  创建一个传递数据的 result channel：

    ```go
    ch := make(chan string)
    ```

1.  创建监听 goroutine 和一个等待组，稍后等待其完成。这个 goroutine 将在其他 goroutine 开始发送数据时被阻塞：

    ```go
    // Allocate results
    results := make([]string,0)
    // WaitGroup will be used later to wait for the listener 
    // goroutine to end
    listenerWg := sync.WaitGroup{}
    listenerWg.Add(1)
    go func() {
      defer listenerWg.Done()
      // Collect results and store in a slice
      for str:=range ch {
        results=append(results,str)
      }
    }()
    ```

1.  创建一个等待组来跟踪将要写入结果 channel 的 goroutines。然后，创建向 channel 发送数据的 goroutines：

    ```go
    wg := sync.WaitGroup{}
    for _,input := range inputs {
      wg.Add(1)
      go func(data string) {
        defer wg.Done()
        ch <- processInput(data)
      }(input)
    }
    ```

1.  等待处理 goroutines 结束并关闭结果 channel：

    ```go
    // Wait for all goroutines to end
    wg.Wait()
    // Close the channel to signal end of data
    // This will signal the listener goroutine that no more data 
    // will be arriving via the channel
    close(ch)
    ```

1.  等待监听 goroutine 结束：

    ```go
    listenerWg.Wait()
    ```

现在你可以使用`results`切片。

## 使用 channels 收集并发计算的结果

通常，你有多达多个 goroutine 在处理问题的各个部分，你必须收集每个 goroutine 的结果来编译一个单一的结果对象。Channels 是完成这个任务的完美机制。

### 如何做到这一点...

1.  创建一个 channel 来收集计算的结果：

    ```go
    resultCh := make(chan int)
    ```

    在这个示例中，`resultCh` channel 是一个`int`值的 channel。也就是说，计算的结果将是整数。

1.  创建一个`sync.WaitGroup`实例来等待 goroutines：

    ```go
    wg := sync.WaitGroup{}
    ```

1.  在 goroutines 之间分配工作。每个 goroutine 都应该能够访问`resultCh`。将每个 goroutine 添加到等待组中，并确保在 goroutine 中调用`defer wg.Done()`。

1.  在 goroutine 中执行计算，并将结果发送到`resultCh`：

    ```go
    var inputs [][]int=[]int{...}
    ...
    for i:=range inputs {
      wg.Add(1)
      go func(data []int) {
         defer wg.Done()
         // Perform the computation
         // computeResult takes a []int, and returns int
         // Send the result to resultCh
         resultCh <- computeResult(data)
      }(inputs[i])
    }
    ```

1.  在这里，你必须做两件事：等待所有 goroutines 完成并从`resultCh`收集结果。你可以这样做两种方式：

    +   在等待 goroutines 并发结束的同时收集结果。也就是说，创建一个 goroutine 并等待 goroutines 结束。当所有 goroutines 完成后，关闭 channel：

        ```go
        go func() {
          // Wait for the goroutines to end
          wg.Wait()
          // When all goroutines are done, close the channel
          close(resultCh)
        }()
        // Create a slice to contain results of the computations
        results:=make([]int,0)
        // Collect the results from the `resultCh`
        // The for-loop will terminate when resultCh is closed
        for result:=range resultCh {
          results=append(results,result)
        }
        ```

    +   在等待 goroutine 结束的同时异步收集结果。当所有 goroutine 完成后，关闭通道。然而，当你关闭通道时，收集结果的 goroutine 可能仍在运行。我们必须等待该 goroutine 结束。为此，我们可以使用另一个等待组：

        ```go
        results:=make([]int,0)
        // Create a new wait group just for the result collection 
        // goroutine
        collectWg := sync.WaitGroup{}
        // Add the collection goroutine to the waitgroup
        collectWg.Add(1)
        go func() {
          // Announce the completion of this goroutine
          defer collectWg.Done()
          // Collect results. The for-loop will terminate when resultCh 
          // is closed.
          for result:= range resultCh {
            results=append(results,result)
          }
        }()
        // Wait for the goroutines to end.
        wg.Wait()
        // Close the channel so the result collection goroutine can 
        // finish
        close(resultCh)
        // Now wait for the result collection goroutine to finish
        collectWg.Wait()
        // results slice is ready
        ```

# 使用 `select` 语句处理多个通道

在任何给定时间，你只能从通道发送数据或接收数据。如果你正在与多个 goroutine（以及因此，多个并发事件）交互，你需要一个语言结构，它将允许你同时与多个通道交互。这个结构就是 `select` 语句。

本节展示了如何使用 `select`。

### 如何做到...

阻塞 `select` 语句从零个或多个分支中选择一个活跃的分支。每个分支是一个通道发送或接收事件。如果没有活跃的分支（也就是说，没有通道可以发送到或从接收），`select` 将被阻塞。

在下面的示例中，`select` 语句等待从两个通道中的一个接收数据。程序只从其中一个通道接收数据。如果两个通道都准备好了，其中一个通道将被随机选择。另一个通道将被留下未读取：

```go
ch1:=make(chan int)
ch2:=make(chan int)
go func() {
  ch1<-1
}()
go func() {
  ch2<-2
}()
select {
case data1:= <- ch1:
  fmt.Println("Read from channel 1: %v", data1)
case data2:= <- ch2:
  fmt.Println("Read from channel 2: %v", data2)
}
```

## 取消 goroutine

在 Go 中创建 goroutine 很简单且高效，但你也要确保你的 goroutine 最终能够结束。如果一个 goroutine 无意中被留下运行，它被称为“泄漏”的 goroutine。如果一个程序持续泄漏 goroutine，最终它可能会因为内存不足错误而崩溃。

一些 goroutine 执行有限数量的操作并自然终止，但有些会无限期地运行，直到接收到外部刺激。对于长时间运行的 goroutine 接收此类刺激的常见模式是使用一个 `done` 通道。

### 如何做到...

1.  创建一个数据类型为空的 `done` 通道：

    ```go
    done:=make(chan struct{})
    ```

1.  创建一个通道为 goroutine 提供输入：

    ```go
    input := make(chan int)
    ```

1.  创建如下所示的 goroutine：

    ```go
    go func() {
      for {
        select {
          case data:= <- input:
            // Process data
          case <-done:
            // Done signal. Terminate
            return
         }
      }
    }()
    ```

要取消 goroutine(s)，只需关闭 `done` 通道：

```go
close(done)
```

这将启用所有监听 `done` 通道的 goroutine 中的 `case <-done` 分支，并且它们将终止。

## 使用非阻塞 `select` 检测取消

非阻塞 `select` 有一个 `default` 分支。当 `select` 语句运行时，它会检查所有可用的分支，如果没有一个可用，则选择 `default` 分支。这允许 `select` 在不阻塞的情况下继续。

### 如何做到...

1.  创建一个数据类型为空的 `done` 通道：

    ```go
    done:=make(chan struct{})
    ```

1.  创建如下所示的 goroutine：

    ```go
    go func() {
      for {
        select {
          case <-done:
            // Done signal. Terminate
            return
           default:
             // Done signal is not sent. Continue
         }
         // Do work
      }
    }()
    ```

要取消 goroutine(s)，只需关闭 `done` 通道。

```go
close(done)
```

# 共享内存

最著名的 Go 惯用语句之一是：“不要通过共享内存来通信，要通过通信来共享内存。”通道是通过通信来共享内存的。通过共享内存进行通信是通过多个 goroutine 中的共享变量来完成的。尽管不鼓励这样做，但有许多情况下共享内存比通道更有意义。如果至少有一个 goroutine 更新了一个被其他 goroutine 读取的共享变量，你必须确保没有内存竞争。

当一个 goroutine 在另一个 goroutine 读取或写入变量的同时更新变量时，会发生内存竞争。当这种情况发生时，无法保证其他 goroutine 会看到该变量的更新。这种情况的一个著名例子是`busy-wait`循环：

```go
func main() {
  done:=false
  go func() {
    // Wait while done==false
    for !done {}
    fmt.Println("Done is true now")
  }()
  done=true
  // Wait indefinitely
  select{}
}
```

这个程序有一个内存竞争。`done=true`的赋值与`for !done`循环并发。这意味着，尽管主 goroutine 运行`done=true`，但读取`done`的 goroutine 可能永远看不到这个更新，无限期地停留在`for`循环中。

## 并发更新共享变量

Go 内存模型保证在一个 goroutine 中，变量写入的效果只对写入之后执行的指令可见。也就是说，如果你更新一个共享变量，你必须使用特殊的工具来确保这个更新对其他 goroutine 可见。确保这一点的简单方法是使用互斥锁。互斥锁代表“互斥排他”。互斥锁是一个你可以用来确保以下内容的工具：

+   任何给定时间只有一个 goroutine 更新一个变量

+   一旦更新完成并且互斥锁被释放，所有 goroutine 都可以看到这个更新

在这个菜谱中，我们展示了如何做到这一点。

### 如何做到这一点...

更新共享变量的程序部分是一个“临界区”。你使用互斥锁来确保只有一个 goroutine 可以进入其临界区。

声明一个互斥锁来保护临界区：

```go
// cacheMutex will be used to protect access to cache
var cacheMutex sync.Mutex
var cache map[string]any = map[string]any{}
```

互斥锁保护一组共享变量。例如，如果你有 goroutine 更新一个单个整数，你为更新该整数的临界区声明一个互斥锁。每次读取或写入该整数值时，你必须使用相同的互斥锁。

在更新共享变量时，首先锁定互斥锁。然后执行更新并解锁互斥锁：

```go
cacheMutex.Lock()
cache[key]=value
cacheMutex.Unlock()
```

使用这种模式，如果有多个 goroutine 尝试更新`cache`，它们将在`cacheMutex.Lock()`处排队，并且只允许一个进行更新。当那个 goroutine 执行更新时，它将调用`cacheMutex.Unlock()`，这将允许一个等待的 goroutine 获取锁并再次更新缓存。

在读取共享变量时，首先锁定互斥锁。然后执行读取，然后解锁互斥锁：

```go
cacheMutex.Lock()
cachedValue, cached := cache[key]
cacheMutex.Unlock()
if cached {
  // Value found in cache
}
```
