

# 刷新并发和并行性

本章将探讨 Go 并发核心中的 goroutines。你将学习它们是如何工作的，区分并发和并行性，管理当前运行的 goroutines，处理数据竞争问题，使用通道进行通信，并使用`Channel`状态和信号来最大化其潜力。掌握这些概念对于编写高效且无错误的 Go 代码至关重要。

在本章中，我们将涵盖以下主要主题：

+   理解 goroutines

+   管理数据竞争

+   理解通道

+   交付保证

+   状态和信号

# 技术要求

你可以在这个章节的源代码中找到[`github.com/PacktPublishing/System-Programming-Essentials-with-Go/tree/main/ch2`](https://github.com/PacktPublishing/System-Programming-Essentials-with-Go/tree/main/ch2)。

# 理解 goroutines

Goroutines 是由 Go 调度器创建和调度以独立运行的函数。Go 调度器负责 goroutines 的管理和执行。

在幕后，我们有一个复杂的算法来使 goroutines 工作。幸运的是，在 Golang 中，我们可以使用`go`关键字以简单的方式实现这个高度复杂的操作。

注意

如果你习惯于具有`async`/`await`功能的语言，你可能已经习惯了事先决定你的函数。它将被并发使用来更改函数签名，以表示该函数可以被暂停/恢复。调用此函数也需要特殊的符号。当使用 goroutines 时，不需要更改函数签名。

在以下代码片段中，我们有一个主函数依次调用`say`函数，分别传递参数`"hello"`和`"world"`：

```go
func main() {
  say(«hello»)
  say(«world»)
}
```

`say`函数接收一个字符串作为参数，并迭代五次。对于每次迭代，我们让函数休眠 500 毫秒，并在打印`s`参数后立即执行：

```go
func say(s string) {
  for i := 1; i < 5; i++ {
     time.Sleep(500 * time.Millisecond)
     fmt.Println(s)
  }
}
```

当我们执行程序时，它应该打印以下输出：

```go
hello
hello
hello
hello
hello
world
world
world
world
world
```

现在，我们在`say`函数的第一次调用之前引入`go`关键字，以在我们的程序中引入并发：

```go
func main() {
  go say(«hello»)
  say(«world»)
}
```

输出应该在`hello`和`world`之间交替。

那么，如果我们为第二次函数调用创建一个 goroutine，我们也能达到相同的结果，对吗？

```go
func main() {
  say(«hello»)
  go say(«world»)
}
```

让我们看看程序的结果：

```go
hello
hello
hello
hello
```

等等！这里有什么不对劲。我们做错了什么？主函数和 goroutine 似乎不同步。

我们没有做错什么。这是预期的行为。当你仔细观察第一个程序时，goroutine 被触发，`say`的第二次调用在主函数的上下文中顺序执行。

换句话说，程序应该等待函数终止，以便达到`main`函数的末尾。对于第二个程序，我们有相反的行为。第一次调用是一个正常的函数调用，所以它按照预期打印了五次，但当第二个 goroutine 被触发时，主函数没有后续指令，所以程序终止。

虽然从程序运行的角度来看，行为是正确的，但这不是我们的意图。我们需要一种方法来在给`main`函数一个终止的机会之前，同步这个执行组中所有 goroutine 的`wait`。在这种情况下，我们可以利用 Go 的构造，即`sync`包中的`WaitGroup`。

## WaitGroup

如同其名，`WaitGroup`是 Go 标准库中的一种机制，允许我们等待一组 goroutine 直到它们显式完成。

没有特定的工厂函数来创建它们，因为它们的零值已经是一个有效的可用状态。由于`WaitGroup`已经被创建，我们需要控制我们正在等待多少个 goroutine。我们可以使用`Add()`方法来通知这个组。

我们如何通知组我们已经完成了一个任务？这再直观不过了。我们可以使用`Done()`方法来实现这一点。

在下面的示例中，我们引入`wait group`来使我们的程序按照预期输出消息：

```go
func main() {
  wg := sync.WaitGroup{}
  wg.Add(2)
  go say(«world», &wg)
  go say("hello", &wg)
  wg.Wait()
}
```

我们创建`WaitGroup`（`wg := sync.WaitGroup{}`）并声明有两个 goroutine 参与这个组（`wg.Add(2)`）。

在程序的最后一行，我们使用`Wait()`方法显式地保持执行，以避免程序终止。

要使我们的函数与`Waitgroup`交互，我们需要发送对这个组的引用。一旦我们有了它的引用，函数就可以延迟调用`Done()`，以确保每次函数完成时都能正确地为我们组发出信号。

这是新的`say`函数：

```go
func say(s string, wg *sync.WaitGroup) {
  defer wg.Done()
  for i := 0; i < 5; i++ {
     fmt.Println(s)
  }
}
```

我们不需要依赖`time.Sleep()`，所以这个版本没有它。

现在，我们可以控制我们的 goroutine 组。让我们处理并发编程中的一个核心担忧问题——状态。

## 改变共享状态

想象一个场景，两个勤奋的工人在繁忙的仓库里负责将物品装箱。每个工人将固定数量的物品装入包中，我们必须跟踪打包的总物品数量。

这个看似简单的任务，类似于并发编程，如果处理不当，很快就会变成一场噩梦。有了适当的同步，工作者可以避免有意干扰彼此的工作，导致结果不正确和不可预测的行为。这是一个经典的数据竞争示例，是并发编程中常见的挑战。

以下代码将向您展示一个类比，其中两个仓库工人面对打包项目时的数据竞争问题。我们首先展示没有适当同步的代码，以演示数据竞争问题。然后，我们将修改代码以解决问题，确保工人能够顺利且准确地合作。

让我们走进熙熙攘攘的仓库，亲眼见证并发挑战以及在这个例子中同步的重要性：

```go
package main
import (
     "fmt"
     "sync"
)
func main() {
     fmt.Println("Total Items Packed:", PackItems(0))
}
func PackItems(totalItems int) int {
     const workers = 2
     const itemsPerWorker = 1000
     var wg sync.WaitGroup
     itemsPacked := 0
     for i := 0; i < workers; i++ {
          wg.Add(1)
          go func(workerID int) {
               defer wg.Done()
               // Simulate the worker packing items into boxes.
               for j := 0; j < itemsPerWorker; j++ {
                      itemsPacked = totalItems
                    // Simulate packing an item.
                    itemsPacked++
               // Update the total items packed without proper synchronization.
               totalItems = itemsPacked
               }
          }(i)
     }
     // Wait for all workers to finish.
     wg.Wait()
     return totalItems
}
```

`main`函数首先调用`PackItems`函数，初始`totalItems`值为 0。

在`PackItems`函数中，定义了两个常量：

+   `workers`：工作 goroutine 的数量（设置为 2）

+   `itemsPerWorker`：每个工人应该打包进箱子的项目数量（设置为 1,000）

创建名为`wg`的`WaitGroup`，等待所有工作 goroutine 完成，然后返回最终的`totalItems`值。

一个循环运行`workers`次，其中每次迭代启动一个新的 goroutine 来模拟工人将项目打包进箱子。在 goroutine 内部，执行以下步骤：

1.  将一个工人 ID 作为参数传递给 goroutine。

1.  `defer wg.Done()`语句确保当 goroutine 退出时，等待组会递减。

1.  `itemsPacked`变量初始化为`totalItems`的当前值，以跟踪此工人打包的项目。

1.  一个循环运行`itemsPerWorker`次，模拟将项目打包进箱子的过程。然而，实际上并没有发生打包；循环只是递增`itemsPacked`变量。

1.  在内循环的最后一步，`totalItems`接收`itemsPacked`变量的修改后的值，该变量包含工人打包的项目数量。

1.  通过将`itemsPacked`值添加到`totalItems`变量中。

由于多个 goroutine 尝试在不适当的同步下并发修改`totalItems`，因此发生数据竞争，导致不可预测和不正确的结果。

### 非确定性结果

考虑这个替代的`main`函数：

```go
func main() {
     times := 0
     for {
          times++
          counter := PackItems(0)
          if counter != 2000 {
               log.Fatalf("it should be 2000 but found %d on execution %d", counter, times)
          }
     }
}
```

程序会不断运行`PackItems`函数，直到达到预期的 2,000 个结果。一旦发生这种情况，程序将显示函数返回的错误值以及达到该点所需的尝试次数。

由于 Go 调度器的非确定性，结果大多数时候是正确的。这段代码需要多次运行才能揭示其同步缺陷。

在单次执行中，我需要超过 16,000 次迭代：

```go
it should be 2000 but found 1170 on execution 16421
```

您的机会！

在您的机器上运行代码的实验。您的代码需要多少次迭代才能失败？

如果你正在使用个人电脑，可能有很多任务正在执行，但你的机器可能有很多未使用的资源。然而，如果你在具有容器的云环境中运行程序，重要的是要考虑集群中共享节点上的噪声量。这里的“噪声”是指在运行你的程序时在主机机器上执行的工作。它可能和你本地的实验一样空闲。然而，在成本效益高的场景中，每个核心和内存都得到充分利用，它很可能被充分利用。

这种对资源的持续竞争场景使得我们的调度器更倾向于选择其他工作负载，而不是仅仅继续运行我们的 goroutine。

在下面的示例中，我们调用 `runtime.Gosched` 函数来模拟噪声。想法是向 Go 调度器发出提示，“*嘿！也许现在是暂停我的好时机*”：

```go
for j := 0; j < itemsPerWorker; j++ {
    itemsPacked = totalItems
    runtime.Gosched() // emulating noise!
    itemsPacked++
    totalItems = itemsPacked
}
```

再次运行主函数，我们可以看到错误的结果比以前出现得更快。例如，在我的执行中，我只需要四次迭代：

```go
it should be 2000 but found 1507 on execution 4
```

不幸的是，代码仍然存在 bug。我们如何预见到这一点？到目前为止，你应该已经猜到了 Go 工具提供了答案，而且你又猜对了。我们可以在测试中管理数据竞争。

# 管理数据竞争

当多个 goroutines 并发访问共享数据或资源时，可能会发生“竞争条件”。正如我们所证明的，这种类型的并发 bug 可能导致不可预测和不受欢迎的行为。Go 测试工具有一个内置功能，称为**Go 竞争检测**，可以检测和识别 Go 代码中的竞争条件。

那么，让我们创建一个 `main_test.go` 文件，并添加一个简单的测试用例：

```go
package main
import (
     "testing"
)
func TestPackItems(t *testing.T) {
     totalItems := PackItems(2000)
     expectedTotal := 2000
     if totalItems != expectedTotal {
          t.Errorf("Expected total: %d, Actual total: %d", expectedTotal, totalItems)
     }
}
```

现在，让我们使用竞争检测器：

```go
go test -race
```

控制台中的结果可能如下所示：

```go
==================
WARNING: DATA RACE
Read at 0x00c00000e288 by goroutine 9:
  example1.PackItems.func1()
      /tmp/main.go:35 +0xa8
  example1.PackItems.func2()
      /tmp/main.go:45 +0x47
Previous write at 0x00c00000e288 by goroutine 8:
  example1.PackItems.func1()
      /tmp/main.go:39 +0xba
  example1.PackItems.func2()
      /tmp/main.go:45 +0x47
// Other lines omitted for brevity
```

初次看到输出时可能会感到有些令人畏惧，但最初最引人注目的信息是消息 `WARNING:` `DATA RACE`。

为了修复此代码中的同步问题，我们应该使用同步机制来保护对 `totalItems` 变量的访问。如果没有适当的同步，对共享数据的并发写入可能导致竞争条件和意外结果。

我们已经使用了 `sync` 包中的 `WaitGroup`。让我们探索更多的同步机制，以确保程序的正确性。

## 原子操作

在 Go 中，“atomic”这个术语并不涉及物理上操纵原子，就像在物理学或化学中那样，这让人感到心碎。在编程中拥有这种能力将是非常有趣的；相反，Go 中的原子操作专注于使用 `sync/atomic` 包同步和管理 goroutines 之间的并发。

Go 提供了原子操作来加载、存储、添加以及 `int32`、`int64`、`uint32`、`uint64`、`uintptr`、`float32` 和 `float64`。原子操作不能直接在任意数据结构上执行。

让我们使用原子包修改我们的程序。首先，我们应该导入它：

```go
import (
     "fmt"
     "sync"
     "sync/atomic"
)
```

我们将利用`AddInt32`函数而不是直接更新`totalItems`来保证同步：

```go
for j := 0; j < itemsPerWorker; j++ {
    atomic.AddInt32(&totalItems, int32(itemsPacked))
}
```

如果我们再次检查数据竞争，将不会报告任何问题。

当我们需要同步单个操作时，原子结构非常出色，但当我们想要同步一段代码块时，其他工具则更为合适，例如互斥锁（mutexes）。

## 互斥锁

哎，互斥锁！它们就像是派对上的保安，为 goroutines 提供保护。想象一下，有一群这些小小的 Go 生物试图围绕共享数据进行舞蹈。一切都很愉快，直到混乱爆发，你会有一个 goroutine 交通堵塞，数据到处溢出！

不要担心，因为互斥锁就像舞池管理员一样迅速介入，确保在任意时刻只有一个酷炫的 goroutine 能在关键部分进行操作。它们就像是并发的节奏守护者，确保每个人轮流进行，没有人会踩到别人的脚趾。

你可以通过声明一个`sync.Mutex`类型的变量来创建一个互斥锁。互斥锁允许我们使用`Lock()`和`Unlock()`方法来保护代码的关键部分。当一个 goroutine 调用`Lock()`时，它会获取互斥锁，而任何尝试调用`Lock()`的其他 goroutine 将会被阻塞，直到锁通过`Unlock()`被释放。

下面是我们程序使用互斥锁的代码：

```go
package main
import (
     "fmt"
     "sync"
)
func main() {
      m := sync.Mutex{}
     fmt.Println("Total Items Packed:", PackItems(&m, 0))
}
func PackItems(m *sync.Mutex, totalItems int) int {
     const workers = 2
     const itemsPerWorker = 1000
     var wg sync.WaitGroup
     for i := 0; i < workers; i++ {
          wg.Add(1)
          go func(workerID int) {
               defer wg.Done()
               for j := 0; j < itemsPerWorker; j++ {
                    m.Lock()
                    itemsPacked := totalItems
                   itemsPacked++
                      totalItems = itemsPacked
                    m.Unlock()
               }
          }(i)
     }
     // Wait for all workers to finish.
     wg.Wait()
     return totalItems
}
```

在这个例子中，我们锁定了一段处理共享状态更改的代码块，并在完成后解锁互斥锁。

如果互斥锁确保了共享状态的正确处理，你可以考虑两种选择：

+   你可以为每个关键行使用加锁和解锁。

+   你可以在函数开始时简单地加锁，并使用`defer`延迟解锁。

是的，你可以这样做！遗憾的是，这两种方法都存在一个缺陷。我们会无差别地引入延迟。为了说明这一点，让我们基准测试第二种方法与原始互斥锁的使用。

让我们创建一个使用多次加锁/解锁调用的函数的第二个版本，称为`MultiplePackItems`，其中除了函数名和内部循环之外，其他一切保持不变。

下面是内部循环的代码：

```go
for j := 0; j < itemsPerWorker; j++ {
    m.Lock()
    itemsPacked = totalItems
    m.Unlock()
    m.Lock()
    itemsPacked++
    m.Unlock()
    m.Lock()
    totalItems = itemsPacked
    m.Unlock()
}
```

让我们通过基准测试来查看这两种选项的性能：

```go
Benchmark-8                   36546             32629 ns/op
BenchmarkMultipleLocks-8      13243             91246 ns/op
```

多重锁的版本在每次操作所需的时间上大约比第一个版本慢**~64%**。

基准测试

我们将在*第六章*《分析性能》中详细讨论基准测试和其他性能测量技术。

这些例子显示了 goroutines 独立执行任务，没有相互协作。然而，在许多情况下，我们的任务需要交换信息或信号来做出决策，例如启动或停止一个过程。

当信息交换至关重要时，我们可以使用 Go 语言中的一个旗舰工具——通道（channel）。

# 理解通道的意义

欢迎来到通道狂欢节！

想象 Go 通道就像神奇的、巨型的管子，允许马戏团表演者（goroutines）在确保没有人掉球的情况下传递玩球（数据）。这字面意义上来说，确保没有人掉球。

## 如何使用通道

要使用通道，我们需要使用一个内置函数`make()`，来告知我们想要通过这个通道传递哪种类型的数据：

```go
 make(Chan T)
```

如果我们想要一个`string`类型的通道，我们应该声明以下内容：

```go
 make (chan string)
```

我们可以指定容量。具有容量的通道称为缓冲通道。现在我们不会深入讨论容量的问题。当我们没有指定容量时，我们创建一个无缓冲通道。

## 无缓冲通道

无缓冲通道是多个 goroutine 之间通信的一种方式，它需要遵守一个简单的规则——想要通过通道发送数据和想要接收数据的 goroutine 应该同时**准备好**。

将其想象成一个“信任跌落”练习。发送者和接收者必须完全信任对方，确保数据的安全，就像杂技演员信任他们的搭档在空中接住他们一样。

抽象？让我们通过示例来探索这个概念。

首先，让我们向一个没有接收者的通道发送信息：

```go
package main
func main() {
    c := make(chan string)
    c <- "message"
}
```

当我们执行时，控制台将打印出类似以下内容：

```go
fatal error: all goroutines are sleep – dead lock!
goroutine 1 [chan send]:
main.main()
```

让我们分析这个输出。

`all goroutines are sleep – deadlock!`是主要的错误信息。它告诉我们，我们程序中的所有 goroutine 都处于`sleep`状态，这意味着它们正在等待某些事件或资源变得可用。然而，由于它们都在等待并且无法取得任何进展，程序遇到了死锁情况。

`goroutine 1 [chan send]:`是消息的一部分，提供了关于遇到死锁的具体 goroutine 的额外信息。在这种情况下，它是`goroutine 1`，并且它参与了通道发送操作（`chan send`）。

这种死锁发生是因为执行被暂停，等待另一个 goroutine 接收信息，但没有人这么做。

死锁

死锁是一种条件，其中两个或多个进程或 goroutine 无法继续进行，因为它们都在等待永远不会发生的事情。

现在，我们可以尝试相反的情况；在下一个示例中，我们想要从一个没有发送者的通道接收信息：

```go
package main
func main() {
    c := make(chan string)
    fmt.Println(<- c )
}
```

控制台输出的内容非常相似，但现在错误信息是关于接收的：

```go
fatal error: all goroutines are sleep – dead lock!
goroutine 1 [chan receive]:
main.main()
```

现在，遵循规则就像同时发送和接收一样简单。所以，声明两者都将是足够的：

```go
package main
func main() {
    c := make(chan string)
    c <- "message" // Sending
    fmt.Println(<- c ) // Receiving
}
```

这是个好主意，但不幸的是，它不起作用，正如我们可以在以下输出中看到的那样：

```go
fatal error: all goroutines are sleep – dead lock!
goroutine 1 [chan send]:
main.main()
```

如果我们遵循规则，为什么它不起作用呢？

好吧，我们并没有完全遵循规则。规则指出，想要通过通道发送数据和想要接收数据的 goroutine 应该同时**准备好**。

需要注意的重要部分是最后的部分——**同时准备好**。

由于代码是按顺序逐行运行的，当我们尝试发送`c <- "message"`时，程序会等待接收者接收消息。我们需要让这两方同时发送和接收消息。我们可以使用我们的并发编程知识来实现这一点。

让我们在混合中使用 goroutine，使用马戏团类比。我们将引入一个函数`throwBalls`，它将期望抛出的球的颜色（`color`）和它应该接收这些抛出的通道（`balls`）：

```go
package main
import "fmt"
func main() {
    balls := make(chan string)
    go throwBalls("red", balls)
    fmt.Println(<-balls, "received!")
}
func throwBalls(color string, balls chan string) {
    fmt.Printf("throwing the %s ball\n", color)
    balls <- color
}
```

这里，我们有三个主要步骤：

1.  我们创建了一个无缓冲的字符串通道，名为`balls`。

1.  使用`throwBalls`函数内联启动 goroutine 来将“红色”发送到通道。

1.  主函数接收并打印从通道接收到的值。

这个示例的输出如下：

```go
throwing the red ball
red received!
```

我们做到了！我们成功地在 goroutine 之间使用通道传递了信息！

但是当我们再发送一个球时会发生什么？让我们用绿色球试一试：

```go
func main() {
    balls := make(chan string)
    go throwBalls("red", balls)
    go throwBalls("green", balls)
    fmt.Println(<-balls, "received!")
}
```

输出只显示接收到一个球。发生了什么？

```go
throwing the red ball
red received!
```

红色还是绿色？

由于我们启动了多个 goroutine，调度器将任意选择哪个应该首先执行。因此，你可以看到绿色或红色随机运行代码。

我们可以通过在通道中添加一个额外的`print`语句来解决这个问题：

```go
func main() {
    balls := make(chan string)
    go throwBalls("red", balls)
    go throwBalls("green", balls)
    fmt.Println(<-balls, "received!")
    fmt.Println(<-balls, "received!")
}
```

虽然它有效，但这不是最优雅的解决方案。如果我们有比发送者更多的接收者，我们可能会再次遇到死锁：

```go
func main() {
    balls := make(chan string)
    go throwBalls("red", balls)
    go throwBalls("green", balls)
    fmt.Println(<-balls, "received!")
    fmt.Println(<-balls, "received!")
    fmt.Println(<-balls, "received!")
}
```

最后的打印将永远等待，导致另一个死锁。

如果我们想让代码能够处理任意数量的球，我们就应该停止添加越来越多的行，并用`range`关键字替换它们。

### 遍历通道

遍历通过通道发送的值的机制是`range`关键字。

让我们更改代码以遍历通道值：

```go
func main() {
    balls := make(chan string)
    go throwBalls("red", balls)
    go throwBalls("green", balls)
    for color := range balls {
         fmt.Println(color, "received!")
    }
}
```

我们可以愉快地检查控制台以查看优雅地接收到的球，但是等等——所有的 goroutine 都在睡眠中！又死锁了？

当我们遍历通道并且`range`期望通道关闭以停止迭代时，会发生这个错误。

### 关闭通道

要关闭通道，我们需要调用内置的`close`函数，并传递通道：

```go
close(balls)
```

好的，我们现在可以保证通道已经关闭。让我们通过在发送者和`range`之间添加`close`调用来更改代码：

```go
go throwBalls("green", balls)
close(balls)
for color := range balls {
```

你可能已经注意到，如果`range`在通道关闭时停止，那么使用这段代码，一旦通道关闭，`range`将永远不会运行。

我们需要协调这一组任务，是的，你是对的——我们再次使用`WaitGroup`来帮助我们。这次，我们不想污染`throwBalls`签名以接收我们的`WaitGroup`，所以我们将创建内联匿名函数以使我们的函数不知道并发。此外，当我们有保证所有任务都完成时，我们想要关闭通道。我们通过`WaitGroup`的`Wait()`方法来推断这一点。

这里是我们的`main`函数：

```go
func main() {
    balls := make(chan string)
    wg := sync.WaitGroup{}
    wg.Add(2)
    go func() {
        defer wg.Done()
        throwBalls("red", balls)
    }()
    go func() {
        defer wg.Done()
        throwBalls("green", balls)
    }()
    go func() {
        wg.Wait()
        close(balls)
    }()
    for color := range balls {
        fmt.Println(color, "received!")
    }
}
```

呼吁！这次，输出显示正确：

```go
throwing the green ball
green received!
throwing the red ball
red received!
```

哇，这是一段旅程，对吧？但是等等！我们仍然需要探索缓冲通道！

## 缓冲通道

是时候进行类比了！

这些是小丑发挥作用的地方！想象一辆座位有限（容量）的小丑车。小丑（发送者）可以跳进跳出汽车，把杂技球（数据）扔进去。

我们想要创建一个带有缓冲通道的程序，模拟马戏团车之旅，其中小丑们试图进入一辆小丑车（一次最多三个小丑）并带着气球。司机控制汽车并管理小丑的乘坐，而小丑们试图进入。如果车满了，他们就会等待并打印一条消息。所有小丑都完成后，程序等待车司机完成，然后打印马戏团车之旅结束的消息。

如果一个小丑试图把太多的杂技球塞进车里，就像一辆装满了小丑和杂技球的汽车一样好笑，创造了一个滑稽的景象！

首先，让我们创建程序结构以接收我们的发送者和接收者：

```go
package main
import (
    "fmt"
    "sync"
    "time"
)
func main() {
    clownChannel := make(chan int, 3)
    clowns := 5
    // senders and receivers logic here!
    var wg sync.WaitGroup
    wg.Wait()
    fmt.Println("Circus car ride is over!")
}
```

这里是司机的 goroutine（接收者）：

```go
go func() {
        defer close(clownChannel)
        for clownID := range clownChannel {
            balloon := fmt.Sprintf("Balloon %d", clownID)
            fmt.Printf("Driver: Drove the car with %s inside\n", balloon)
            time.Sleep(time.Millisecond * 500)
            fmt.Printf("Driver: Clown finished with %s, the car is ready for more!\n", balloon)
        }
    }()
```

我们在骑手块下方添加小丑逻辑（发送者）：

```go
for clown := 1; clown <= clowns; clown++ {
    wg.Add(1)
    go func(clownID int) {
        defer wg.Done()
        balloon := fmt.Sprintf("Balloon %d", clownID)
        fmt.Printf("Clown %d: Hopped into the car with %s\n", clownID, balloon)
        select {
            case clownChannel <- clownID:
                fmt.Printf("Clown %d: Finished with %s\n", clownID, balloon)
            default:
                fmt.Printf("Clown %d: Oops, the car is full, can't fit %s!\n", clownID, balloon)
        }
    }(clown)
}
```

运行代码后，我们可以看到小丑们制造的所有麻烦：

```go
Clown 1: Hopped into the car with Balloon 1
Clown 1: Finished with Balloon 1
Driver: Drove the car with Balloon 1 inside
Clown 2: Hopped into the car with Balloon 2
Clown 2: Finished with Balloon 2
Clown 5: Hopped into the car with Balloon 5
Clown 5: Finished with Balloon 5
Clown 3: Hopped into the car with Balloon 3
Clown 3: Finished with Balloon 3
Clown 4: Hopped into the car with Balloon 4
Clown 4: Oops, the car is full, can't fit Balloon 4!
Circus car ride is over!
```

select

`select`语句允许我们在多个通信通道上等待，并选择第一个就绪的通道，从而有效地允许我们在通道上执行非阻塞操作。

当使用通道工作时，很容易陷入比较消息队列和通道的困境，但可能存在更好的理解方式。通道内部是环形缓冲区，当选择程序设计时，这些信息可能会令人困惑且无助于解决问题。通过优先理解信号和消息的保证交付，你将更好地配备高效地与通道一起工作。

# 交付的保证

缓冲通道和无缓冲通道之间的主要区别是交付的保证。

如我们之前所见，无缓冲通道始终保证交付，因为它们只有在接收者准备好时才发送消息。相反，缓冲通道不能保证消息交付，因为它们可以在同步步骤成为强制性的之前“缓冲”任意数量的消息。因此，读者可能无法从通道缓冲区中读取消息。

选择它们之间最显著的副作用是你可以为程序引入多少延迟。

## 延迟

在并发编程的上下文中，延迟指的是数据从发送者（goroutine）通过通道到达接收者（goroutine）所需的时间。

在 Go 通道中，延迟受多个因素的影响：

+   **缓冲**: 缓冲可以减少发送者和接收者不完全同步时的延迟。

+   **阻塞**: 无缓冲通道会阻塞发送者和接收者，直到他们准备好进行通信，这可能导致更高的延迟。缓冲通道允许发送者继续进行，而无需立即同步，这可能会降低延迟。

+   **Goroutine 调度**：通道通信中的延迟也取决于 Go 运行时如何调度 goroutine。例如，可用的 CPU 核心数量和调度算法等因素会影响 goroutine 的执行速度。

### 选择通道类型

作为一项经验法则，我们认为无缓冲的通道在以下场景下是一个不错的选择：

+   **保证交付**：提供一种保证，即发送的值被另一个 goroutine 接收。这在需要确保数据完整性和无数据丢失的场景中特别有用。

+   **一对一通信**：无缓冲通道最适合 goroutine 之间的一对一通信。

+   **负载均衡**：无缓冲通道可用于实现负载均衡模式，确保工作在 worker goroutine 之间均匀分布。

相反，缓冲通道提供以下功能：

+   **异步通信**：缓冲通道允许 goroutine 之间进行异步通信。在缓冲通道上发送数据时，如果通道的缓冲区有空间，发送者不会阻塞直到数据被接收。这可以在某些场景中提高吞吐量。

+   **减少竞争**：在存在多个发送者和接收者的场景中，使用缓冲通道可以减少竞争。例如，在生产者-消费者模式中，可以使用缓冲通道允许生产者继续生产，而无需等待消费者赶上。

+   **防止死锁**：缓冲通道可以通过允许一定程度的缓冲来帮助防止 goroutine 死锁，这在工作负载不可预测变化时可能很有用。

+   **批量处理**：缓冲通道可用于批量处理或管道，其中数据以一个速率产生，以另一个速率消费。

现在我们已经涵盖了延迟的关键方面以及它如何影响并发编程中的通道通信，让我们将重点转移到另一个关键方面——状态和信号。理解状态和信号的语义对于避免常见陷阱和做出明智的设计决策至关重要。

# 状态和信号

探索状态和信号的语义可以使你在避免更直接的错误或做出良好的设计选择方面领先一步。

## 状态

虽然 Go 通过通道简化了并发的采用，但也有一些特性和陷阱。

我们应该记住，通道有三个状态——nil、open（空、非空）和 closed。这些状态与我们能否以及如何使用通道，无论是从发送者还是接收者的角度来看，都有很强的关联。

当你想从通道中读取时考虑：

+   向一个`只写`通道读取会导致编译错误

+   如果通道是`nil`，则无限期地从它读取将阻塞你的 goroutine，直到它被初始化

+   在一个`开放`且`空`的通道中读取将阻塞，直到有数据可用

+   在一个`开放`且`非空`的通道中，读取将返回数据

+   如果通道是`关闭`的，读取它将返回其类型的默认值，并返回`false`以指示关闭

写入也有其细微之处：

+   向只读通道写入会导致编译错误

+   向`nil`通道写入会阻塞，直到它被初始化

+   向一个`打开`且`满`的通道写入会阻塞，直到有空间

+   在一个`打开`且`非满`的通道中，写入是成功的

+   向一个`关闭`的通道写入会导致`panic`

关闭通道取决于其状态：

+   关闭一个带有数据的`打开通道`允许读取直到耗尽，然后返回默认值。

+   关闭一个`打开空通道`会立即关闭它，并且读取也会返回默认值。

+   尝试关闭一个`已关闭的通道`会导致`panic`。

+   关闭只读通道会导致编译错误。

## 信号

在 goroutine 之间进行信号传递是频道的一个常见用例。你可以通过在它们之间发送信号或消息来使用频道协调和同步不同 goroutine 的执行。

这里有一个如何使用 Go 频道在两个 goroutine 之间进行信号传递的简单示例：

```go
package main
import (
    "fmt"
    "sync"
)
func main() {
    signalChannel := make(chan bool)
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("Goroutine 1 is waiting for a signal...")
        <-signalChannel
        fmt.Println("Goroutine 1 received the signal and is now doing something.")
    }()
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("Goroutine 2 is about to send a signal.")
        signalChannel <- true
        fmt.Println("Goroutine 2 sent the signal.")
    }()
    wg.Wait()
    fmt.Println("Both goroutines have finished.")
}
```

在这个片段中，我们创建了一个名为`signalChannel`的通道，用于在两个 goroutine 之间进行信号传递。`Goroutine 1`使用`<-signalChannel`在通道上等待信号，而`Goroutine 2`使用`signalChannel <- true`发送信号。

`sync.WaitGroup`确保我们在打印`"Both goroutines have finished."`之前等待两个 goroutine 都完成。

当你运行这个程序时，你会看到`Goroutine 1`等待`Goroutine 2`的信号，然后继续执行其任务。

Go 频道是同步和协调 goroutine 之间复杂交互的灵活方式。它们可以用来实现生产者-消费者或扇出/扇入的并发模式。

## 选择你的同步机制

频道总是答案吗？绝对不是！我们可以使用互斥锁或频道来解决相同的问题。我们如何选择？偏好实用主义。当互斥锁使你的解决方案易于阅读和维护时，不要犹豫，选择互斥锁！

如果你在它们之间选择有困难，这里有一个有偏见的指南。

当你需要做以下事情时使用频道：

+   传递数据的所有权

+   分配工作单元

+   以异步方式通信结果

当你处理以下内容时，请使用互斥锁：

+   缓存

+   共享状态

好的，让我们总结一下，回顾本章我们所学的内容。

# 摘要

在本章中，我们学习了 goroutine 的功能、它们的简单性和使用`WaitGroup`进行同步的重要性。我们还意识到了管理共享状态的困难，使用仓库类比来解释数据竞争。此外，我们介绍了 Go 的竞态检测工具来识别竞态条件，通信通道的重要性及其潜在的风险。

现在我们已经更新了并发知识，让我们在下一章中探索使用系统调用的操作系统的交互。

# 第二部分：与操作系统交互

在本部分，我们将使用 Go 语言深入探讨系统级编程概念。您将探索进程间通信（IPC）机制、系统事件处理、文件操作和 Unix 套接字。本节提供了实际示例和详细解释，以帮助您掌握构建健壮和高效系统级应用程序的知识和技能。

本部分包含以下章节：

+   *第三章*, *理解系统调用*

+   *第四章*, *文件和目录操作*

+   *第五章*, *处理系统事件*

+   *第六章*, *理解进程间通信中的管道*

+   *第七章*, *Unix 套接字*
