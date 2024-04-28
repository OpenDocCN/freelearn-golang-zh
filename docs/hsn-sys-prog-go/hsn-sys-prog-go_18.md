# 实现并发模式

本章将介绍并发模式以及如何使用它们构建健壮的系统应用程序。我们已经看过了所有涉及并发的工具（goroutines 和通道，`sync`和`atomic`，以及上下文），现在我们将看一些常见的组合模式，以便我们可以在程序中使用它们。

本章将涵盖以下主题：

+   从生成器开始

+   通过管道进行排序

+   复用和解复用

+   其他模式

+   资源泄漏

# 技术要求

这一章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅[第三章]（602a92d5-25f7-46b8-83d4-10c6af1c6750.xhtml），*Go 概述*。

# 从生成器开始

生成器是一个每次调用时返回序列的下一个值的函数。使用生成器的最大优势是惰性创建序列的新值。在 Go 中，这可以用接口或通道来表示。当生成器与通道一起使用时，其中一个优点是它们以并发方式产生值，在单独的 goroutine 中，使主 goroutine 能够执行其他类型的操作。

可以用一个非常简单的接口来抽象化：

```go
type Generator interface {
    Next() interface{}
}

type GenInt64 interface {
    Next() int64
}
```

接口的返回类型将取决于用例，在我们的情况下是`int64`。它的基本实现可以是一个简单的计数器：

```go
type genInt64 int64

func (g *genInt64) Next() int64 {
    *g++
    return int64(*g)
}
```

这个实现不是线程安全的，所以如果我们尝试在 goroutine 中使用它，可能会丢失一些元素：

```go
func main() {
    var g genInt64
    for i := 0; i < 1000; i++ {
        go func(i int) {
            fmt.Println(i, g.Next())
        }(i)
    }
    time.Sleep(time.Second)
}
```

使生成器并发的一个简单方法是对整数执行原子操作。

这将使并发生成器线程安全，代码需要进行很少的更改：

```go
type genInt64 int64

func (g *genInt64) Next() int64 {
    return atomic.AddInt64((*int64)(g), 1)
}
```

这将避免应用程序中的竞争条件。但是，还有另一种可能的实现，但这需要使用通道。其思想是在 goroutine 中生成值，然后将其传递到共享通道中的下一个方法，如下例所示：

```go
type genInt64 struct {
    ch chan int64
}

func (g genInt64) Next() int64 {
    return <-g.ch
}

func NewGenInt64() genInt64 {
    g := genInt64{ch: make(chan int64)}
    go func() {
        for i := int64(0); ; i++ {
            g.ch <- i
        }
    }()
    return g
}
```

循环将永远继续，并且在生成器用户停止使用`Next`方法请求新值时，将在发送操作中阻塞。

代码之所以以这种方式结构化，是因为我们试图实现我们在开头定义的接口。我们也可以只返回一个通道并用它进行接收：

```go
func GenInt64() <-chan int64 {
 ch:= make(chan int64)
    go func() {
        for i := int64(0); ; i++ {
            ch <- i
        }
    }()
    return ch
}
```

直接使用通道的主要优势是可以将其包含在`select`语句中，以便在不同的通道操作之间进行选择。以下显示了两个不同生成器之间的`select`：

```go
func main() {
    ch1, ch2 := GenInt64(), GenInt64()
    for i := 0; i < 20; i++ {
        select {
        case v := <-ch1:
            fmt.Println("ch 1", v)
        case v := <-ch2:
            fmt.Println("ch 2", v)
        }
    }
}
```

# 避免泄漏

允许循环结束是个好主意，以避免 goroutine 和资源泄漏。其中一些问题如下：

+   当 goroutine 挂起而不返回时，内存中的空间仍然被使用，导致应用程序在内存中的大小增加。只有当 goroutine 返回或发生 panic 时，GC 才会收集 goroutine 和堆栈中定义的变量。

+   如果文件保持打开状态，这可能会阻止其他进程对其执行操作。如果打开的文件数量达到操作系统强加的限制，进程将无法打开其他文件（或接受网络连接）。

这个问题的一个简单解决方案是始终使用`context.Context`，这样您就有了 goroutine 的明确定义的退出点：

```go
func NewGenInt64(ctx context.Context) genInt64 {
    g := genInt64{ch: make(chan int64)}
    go func() {
        for i := int64(0); ; i++ {
            select {
            case g.ch <- i:
                // do nothing
            case <-ctx.Done():
                close(g.ch)
                return
            }
        }
    }()
    return g
}
```

这可以用于生成值，直到需要它们并在不需要新值时取消上下文。相同的模式也可以应用于返回通道的版本。例如，我们可以直接使用`cancel`函数或在上下文上设置超时：

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*100)
    defer cancel()
    g := NewGenInt64(ctx)
    for i := range g.ch {
        go func(i int64) {
            fmt.Println(i, g.Next())
        }(i)
    }
    time.Sleep(time.Second)
}
```

生成器将生成数字，直到提供的上下文到期。此时，生成器将关闭通道。

# 通过管道进行排序

管道是一种结构化应用程序流程的方式，通过将主要执行分成可以使用某种通信手段相互交谈的阶段来实现。这可以是以下之一：

+   外部，比如网络连接或文件

+   应用程序内部，如 Go 的通道

第一个阶段通常被称为生产者，而最后一个通常被称为消费者。

Go 提供的并发工具集允许我们有效地使用多个 CPU，并通过阻塞输入或输出操作来优化它们的使用。通道特别适用于内部管道通信。它们可以由接收入站通道并返回出站通道的函数表示。基本结构看起来像这样：

```go
func stage(in <-chan interface{}) <-chan interface{} {
    var out = make(chan interface{})
    go func() {
        for v := range in {
            v = v.(int)+1 // some operation
            out <- v
        }
        close(out)
    }()
    return out
}
```

我们创建一个与输入通道相同类型的通道并返回它。在一个单独的 goroutine 中，我们从输入通道接收数据，对数据执行操作，然后将其发送到输出通道。

这种模式可以通过使用`context.Context`进一步改进，以便我们更好地控制应用程序流程。它看起来像以下代码：

```go
func stage(ctx context.Context, in <-chan interface{}) <-chan interface{} {
    var out = make(chan interface{})
    go func() {
        defer close(out)
        for v := range in {
            v = v.(int)+1 // some operation
            select {
                case out <- v:
                case <-ctx.Done():
                    return
            }
        }
    }()
    return out
}
```

在设计管道时，有一些通用规则应该遵循：

+   中间阶段将接收一个入站通道并返回另一个。

+   生产者不会接收任何通道，但会返回一个。

+   消费者将接收一个通道而不返回一个。

+   每个阶段在创建时都会关闭通道，当它发送完消息时。

+   每个阶段应该保持从输入通道接收，直到它关闭。

让我们创建一个简单的管道，使用特定字符串从读取器中过滤行并打印过滤后的行，突出显示搜索字符串。我们可以从第一个阶段开始 - 源 - 它在签名中不会接收任何通道，但会使用读取器扫描行。我们为了对早期退出请求做出反应（上下文取消）和使用`bufio`扫描器逐行读取。以下代码显示了这一点：

```go
func SourceLine(ctx context.Context, r io.ReadCloser) <-chan string {
    ch := make(chan string)
    go func() {
        defer func() { r.Close(); close(ch) }()
        s := bufio.NewScanner(r)
        for s.Scan() {
            select {
            case <-ctx.Done():
                return
            case ch <- s.Text():
            }
        }
    }()
    return ch
}
```

我们可以将剩余的操作分为两个阶段：过滤阶段和写入阶段。过滤阶段将简单地从源通道过滤到输出通道。我们仍然传递上下文，以避免在上下文已经完成的情况下发送额外的数据。这是文本过滤的实现：

```go
func TextFilter(ctx context.Context, src <-chan string, filter string) <-chan string {
    ch := make(chan string)
    go func() {
        defer close(ch)
        for v := range src {
            if !strings.Contains(v, filter) {
                continue
            }
            select {
            case <-ctx.Done():
                return
            case ch <- v:
            }
        }
    }()
    return ch
}
```

最后，我们有最终阶段，消费者，它将在写入器中打印输出，并且还将使用上下文进行早期退出：

```go
func Printer(ctx context.Context, src <-chan string, color int, highlight string, w io.Writer) {
    const close = "\x1b[39m"
    open := fmt.Sprintf("\x1b[%dm", color)
    for {
        select {
        case <-ctx.Done():
            return
        case v, ok := <-src:
            if !ok {
                return
            }
            i := strings.Index(v, highlight)
            if i == -1 {
                panic(v)
            }
            fmt.Fprint(w, v[:i], open, highlight, close, v[i+len(highlight):], "\n")
        }
    }
}
```

使用这个函数的方式如下：

```go
func main() {
    var search string
    ...
    ctx := context.Background()
    src := SourceLine(ctx, ioutil.NopCloser(strings.NewReader(sometext)))
    filter := TextFilter(ctx, src, search)
    Printer(ctx, filter, 31, search, os.Stdout)
}
```

通过这种方法，我们学会了如何将复杂的操作分解为由阶段执行的简单任务，并使用通道连接。

# 复用和解复用

现在我们熟悉了管道和阶段，我们可以介绍两个新概念：

+   **复用（多路复用）或扇出**：从一个通道接收并发送到多个通道

+   **解复用（解多路复用）或扇入**：从多个通道接收并通过一个通道发送

这种模式非常常见，可以让我们以不同的方式利用并发的力量。最明显的方式是从比其后续步骤更快的通道中分发数据，并创建多个此类步骤的实例来弥补速度差异。

# 扇出

复用的实现非常简单。同一个通道需要传递给不同的阶段，以便每个阶段都从中读取。

每个 goroutine 在运行时调度期间竞争资源，因此如果我们想保留更多的资源，我们可以为管道的某个阶段或应用程序中的某个操作使用多个 goroutine。

我们可以创建一个小应用程序，使用这种方法统计出现在一段文本中的单词的次数。让我们创建一个初始的生产者阶段，从写入器中读取并返回该行的单词切片：

```go
func SourceLineWords(ctx context.Context, r io.ReadCloser) <-chan []string {
    ch := make(chan []string)
    go func() {
        defer func() { r.Close(); close(ch) }()
        b := bytes.Buffer{}
        s := bufio.NewScanner(r)
        for s.Scan() {
            b.Reset()
            b.Write(s.Bytes())
            words := []string{}
            w := bufio.NewScanner(&b)
            w.Split(bufio.ScanWords)
            for w.Scan() {
                words = append(words, w.Text())
            }
            select {
            case <-ctx.Done():
                return
            case ch <- words:
            }
        }
    }()
    return ch
}
```

现在我们可以定义另一个阶段，用于计算这些单词的出现次数。我们将使用这个阶段进行扇出：

```go
func WordOccurrence(ctx context.Context, src <-chan []string) <-chan map[string]int {
    ch := make(chan map[string]int)
    go func() {
        defer close(ch)
        for v := range src {
            count := make(map[string]int)
            for _, s := range v {
                count[s]++
            }
            select {
            case <-ctx.Done():
                return
            case ch <- count:
            }
        }
    }()
    return ch
}
```

为了将第一阶段用作第二阶段的多个实例的来源，我们只需要使用相同的输入通道创建多个计数阶段：

```go
ctx, canc := context.WithCancel(context.Background())
defer canc()
src := SourceLineWords(ctx,   
    ioutil.NopCloser(strings.NewReader(cantoUno)))
count1, count2 := WordOccurrence(ctx, src), WordOccurrence(ctx, src)
```

# 扇入

Demuxing 有点复杂，因为我们不需要在一个 goroutine 中盲目地接收数据，而是需要同步一系列通道。避免竞争条件的一个好方法是创建另一个通道，所有来自各种输入通道的数据都将被接收到。我们还需要确保一旦所有通道都完成，这个合并通道就会关闭。我们还必须记住，如果上下文被取消，通道将被关闭。我们在这里使用`sync.Waitgroup`等待所有通道完成：

```go
wg := sync.WaitGroup{}
merge := make(chan map[string]int)
wg.Add(len(src))
go func() {
    wg.Wait()
    close(merge)
}()
```

问题在于我们有两种可能的触发器来关闭通道：常规传输结束和上下文取消。

我们必须确保如果上下文结束，不会向输出通道发送任何消息。在这里，我们正在从输入通道收集值并将它们发送到合并通道，但前提是上下文没有完成。我们这样做是为了避免将发送操作发送到关闭的通道，这将使我们的应用程序发生恐慌：

```go
for _, ch := range src {
    go func(ch <-chan map[string]int) {
        defer wg.Done()
        for v := range ch {
            select {
            case <-ctx.Done():    
                return
            case merge <- v:
            }
        }
    }(ch)
}
```

最后，我们可以专注于使用合并通道执行我们的最终字数的最后一个操作：

```go
count := make(map[string]int)
for {
    select {
    case <-ctx.Done():
        return count
    case c, ok := <-merge:
        if !ok {
            return count
        }
        for k, v := range c {
            count[k] += v
        }
    }
}
```

应用程序的`main`函数，在添加扇入后，将如下所示：

```go
func main() {
    ctx, canc := context.WithCancel(context.Background())
    defer canc()
    src := SourceLineWords(ctx, ioutil.NopCloser(strings.NewReader(cantoUno)))
    count1, count2 := WordOccurrence(ctx, src), WordOccurrence(ctx, src)
    final := MergeCounts(ctx, count1, count2)
    fmt.Println(final)
}
```

我们可以看到，扇入是应用程序最复杂和关键的部分。让我们回顾一下帮助构建一个没有恐慌或死锁的扇入函数的决定：

+   使用合并通道从各种输入中收集值。

+   使用`sync.WaitGroup`，计数器等于输入通道的数量。

+   在一个单独的 goroutine 中使用它，并等待它关闭通道。

+   对于每个输入通道，创建一个将值传输到合并通道的 goroutine。

+   确保只有在上下文没有完成的情况下才发送记录。

+   在退出这样的 goroutine 之前，使用等待组的`done`函数。

遵循上述步骤将允许我们使用简单的`range`从合并通道中获取值。在我们的示例中，我们还检查上下文是否完成，然后才从通道接收，以便允许 goroutine 提前退出。

# 生产者和消费者

通道允许我们轻松处理多个消费者从一个生产者接收数据的情况，反之亦然。

与单个生产者和一个消费者的情况一样，我们已经看到，这是非常直接的：

```go
func main() {
    // one producer
    var ch = make(chan int)
    go func() {
        for i := 0; i < 100; i++ {
            ch <- i
        }
        close(ch)
    }()
    // one consumer
    var done = make(chan struct{})
    go func() {
        for i := range ch {
            fmt.Println(i)
        }
        close(done)
    }()
    <-done
}
```

完整的示例在这里：[`play.golang.org/p/hNgehu62kjv`](https://play.golang.org/p/hNgehu62kjv)。

# 多个生产者（N * 1）

使用等待组可以轻松处理多个生产者或消费者的情况。在多个生产者的情况下，所有的 goroutine 都将共享同一个通道：

```go
// three producer
var ch = make(chan string)
wg := sync.WaitGroup{}
wg.Add(3)
for i := 0; i < 3; i++ {
    go func(n int) {
        for i := 0; i < 100; i++ {
            ch <- fmt.Sprintln(n, i)
        }
        wg.Done()
    }(i)
}
go func() {
    wg.Wait()
    close(ch)
}()
```

完整的示例在这里：[`play.golang.org/p/4DqWKntl6sS`](https://play.golang.org/p/4DqWKntl6sS)。

他们将使用`sync.WaitGroup`等待每个生产者完成后关闭通道。

# 多个消费者（1 * M）

相同的推理适用于多个消费者-它们都在不同的 goroutine 中从同一个通道接收：

```go
func main() {
    // three consumers
    wg := sync.WaitGroup{}
    wg.Add(3)
    var ch = make(chan string)

    for i := 0; i < 3; i++ {
        go func(n int) {
            for i := range ch {
                fmt.Println(n, i)
            }
            wg.Done()
        }(i)
    }

    // one producer
    go func() {
        for i := 0; i < 10; i++ {
            ch <- fmt.Sprintln("prod-", i)
        }
        close(ch)
    }()

    wg.Wait()
}
```

完整的示例在这里：[`play.golang.org/p/_SWtw54ITFn`](https://play.golang.org/p/_SWtw54ITFn)。

在这种情况下，`sync.WaitGroup`用于等待应用程序结束。

# 多个消费者和生产者（N*M）

最后的情况是我们有任意数量的生产者（`N`）和另一个任意数量的消费者（`M`）。

在这种情况下，我们需要两个等待组：一个用于生产者，另一个用于消费者：

```go
const (
    N = 3
    M = 5
)
wg1 := sync.WaitGroup{}
wg1.Add(N)
wg2 := sync.WaitGroup{}
wg2.Add(M)
var ch = make(chan string)
```

接下来是一系列生产者和消费者，每个都在自己的 goroutine 中：

```go
for i := 0; i < N; i++ {
    go func(n int) {
        for i := 0; i < 10; i++ {
            ch <- fmt.Sprintf("src-%d[%d]", n, i)
        }
        wg1.Done()
    }(i)
}

for i := 0; i < M; i++ {
    go func(n int) {
        for i := range ch {
            fmt.Printf("cons-%d, msg %q\n", n, i)
        }
        wg2.Done()
    }(i)
}
```

最后一步是等待`WaitGroup`生产者完成工作，以关闭通道。

然后，我们可以等待消费者通道，让所有消息都被消费者处理：

```go
wg1.Wait()
close(ch)
wg2.Wait()
```

# 其他模式

到目前为止，我们已经看过了可以使用的最常见的并发模式。现在，我们将专注于一些不太常见但值得一提的模式。

# 错误组

`sync.WaitGroup`的强大之处在于它允许我们等待同时运行的 goroutines 完成它们的工作。我们已经看过了如何共享上下文可以让我们在正确使用时为 goroutines 提供早期退出。第一个并发操作，比如从通道发送或接收，位于`select`块中，与上下文完成通道一起：

```go
func main() {
    ctx, canc := context.WithTimeout(context.Background(), time.Second)
    defer canc()
    wg := sync.WaitGroup{}
    wg.Add(10)
    var ch = make(chan int)
    for i := 0; i < 10; i++ {
        go func(ctx context.Context, i int) {
            defer wg.Done()
            d := time.Duration(rand.Intn(2000)) * time.Millisecond
            time.Sleep(d)
            select {
            case <-ctx.Done():
                fmt.Println(i, "early exit after", d)
                return
            case ch <- i:
                fmt.Println(i, "normal exit after", d)
            }
        }(ctx, i)
    }
    go func() {
        wg.Wait()
        close(ch)
    }()
    for range ch {
    }
}
```

实验性的`golang.org/x/sync/errgroup`包提供了对这种情况的改进。

内置的 goroutines 始终是`func()`类型，但这个包允许我们并发执行`func() error`并返回从各种 goroutines 接收到的第一个错误。

在启动更多 goroutines 并接收第一个错误的情况下，这非常有用。`errgroup.Group`类型可以用作零值，其`Do`方法以`func() error`作为参数并并发启动函数。

`Wait`方法要么等待所有函数成功完成并返回`nil`，要么返回来自任何函数的第一个错误。

让我们创建一个定义 URL 访问者的示例，即一个获取 URL 字符串并返回`func() error`的函数，用于发起调用：

```go
func visitor(url string) func() error {
    return func() (err error) {
        s := time.Now()
        defer func() {
            log.Println(url, time.Since(s), err)
        }()
        var resp *http.Response
        if resp, err = http.Get(url); err != nil {
            return
        }
        return resp.Body.Close()
    }
}
```

我们可以直接使用`Go`方法并等待。这将返回由无效 URL 引起的错误：

```go
func main() {
    eg := errgroup.Group{}
    var urlList = []string{
        "http://www.golang.org/",
        "http://invalidwebsite.hey/",
        "http://www.google.com/",
    }
    for _, url := range urlList {
        eg.Go(visitor(url))
    }
    if err := eg.Wait(); err != nil {
        log.Fatalln("Error:", err)
    }
}
```

错误组还允许我们使用`WithContext`函数创建一个组以及上下文。当收到第一个错误时，此上下文将被取消。上下文的取消使`Wait`方法能够立即返回，但也允许在函数的 goroutines 中进行早期退出。

我们可以创建一个类似的`func() error`创建者，它会将值发送到通道直到上下文关闭。我们将引入一个小概率（1%）引发错误：

```go
func sender(ctx context.Context, ch chan<- string, n int) func() error {
    return func() (err error) {
        for i := 0; ; i++ {
            if rand.Intn(100) == 42 {
                return errors.New("the answer")
            }
            select {
            case ch <- fmt.Sprintf("[%d]%d", n, i):
            case <-ctx.Done():
                return nil
            }
        }
    }
}
```

我们将使用专用函数生成一个错误组和一个上下文，并使用它来启动函数的多个实例。在等待组时，我们将在一个单独的 goroutine 中接收到它。等待结束后，我们将确保没有更多的值被发送到通道（这将导致恐慌），通过额外等待一秒钟：

```go
func main() {
    eg, ctx := errgroup.WithContext(context.Background())
    ch := make(chan string)
    for i := 0; i < 10; i++ {
        eg.Go(sender(ctx, ch, i))
    }
    go func() {
        for s := range ch {
            log.Println(s)
        }
    }()
    if err := eg.Wait(); err != nil {
        log.Println("Error:", err)
    }
    close(ch)
    log.Println("waiting...")
    time.Sleep(time.Second)
}
```

正如预期的那样，由于上下文中的`select`语句，应用程序运行顺利，不会发生恐慌。

# 泄漏桶

我们在前几章中看到了如何使用 ticker 构建速率限制器：通过使用`time.Ticker`强制客户端等待轮到自己被服务。还有另一种对服务和库进行速率限制的方法，称为**泄漏桶**。这个名字让人联想到一个有几个孔的桶。如果你在往里面加水，就要小心不要把太多水放进去，否则它会溢出。在添加更多水之前，你需要等待水位下降 - 这种速度取决于桶的大小和孔的数量。通过以下类比，我们可以很容易地理解这种并发模式的作用：

+   通过孔洞流出的水代表已完成的请求。

+   从桶中溢出的水代表被丢弃的请求。

桶将由两个属性定义：

+   **速率**：如果请求频率较低，则每个时间段的理想请求量。

+   **容量**：在资源暂时变得无响应之前，可以同时完成的请求数量。

桶具有最大容量，因此当请求的频率高于指定的速率时，该容量开始下降，就像当您放入太多水时，桶开始溢出一样。如果频率为零或低于速率，则桶将缓慢恢复其容量，因此水将被缓慢排出。

漏桶的数据结构将具有容量和可用请求的计数器。该计数器在创建时将与容量相同，并且每次执行请求时都会减少。速率指定了状态需要多久重置到容量的频率：

```go
type bucket struct {
    capacity uint64
    status uint64
}
```

创建新的桶时，我们还应该注意状态重置。我们可以使用 goroutine 和上下文来正确终止它。我们可以使用速率创建一个 ticker，然后使用这些 ticks 来重置状态。我们需要使用 atomic 包来确保它是线程安全的：

```go
func newBucket(ctx context.Context, cap uint64, rate time.Duration) *bucket {
    b := bucket{capacity: cap, status: cap}
    go func() {
        t := time.NewTicker(rate)
        for {
            select {
            case <-t.C:
                atomic.StoreUint64(&b.status, b.capacity)
            case <-ctx.Done():
                t.Stop()
                return
            }
        }
    }()
    return &b
}
```

当我们向桶中添加内容时，我们可以检查状态并相应地采取行动：

+   如果状态为`0`，我们无法添加任何内容。

+   如果要添加的数量高于可用性，我们将添加我们可以的内容。

+   否则，我们将添加完整的数量：

```go
func (b *bucket) Add(n uint64) uint64 {
    for {
        r := atomic.LoadUint64(&b.status)
        if r == 0 {
            return 0
        }
        if n > r {
            n = r
        }
        if !atomic.CompareAndSwapUint64(&b.status, r, r-n) {
            continue
        }
        return n
    }
}
```

我们使用循环尝试原子交换操作，直到成功为止，以确保我们在进行**比较和交换**（**CAS**）时得到的内容不会在进行**加载**操作时发生变化。

桶可以用于尝试向桶中添加随机数量并记录其结果的客户端：

```go
type client struct {
    name string
    max int
    b *bucket
    sleep time.Duration
}

func (c client) Run(ctx context.Context, start time.Time) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            n := 1 + rand.Intn(c.max-1)
            time.Sleep(c.sleep)
            e := time.Since(start).Seconds()
            a := c.b.Add(uint64(n))
            log.Printf("%s tries to take %d after %.02fs, takes  
                %d", c.name, n, e, a)
        }
    }
}
```

我们可以同时使用更多客户端，以便并发访问资源将产生以下结果：

+   一些 goroutine 将向桶中添加他们期望的内容。

+   一个 goroutine 最终将通过添加与剩余容量相等的数量来填充桶，即使他们试图添加的数量更高。

+   其他 goroutine 在容量重置之前将无法向桶中添加内容：

```go
func main() {
    ctx, canc := context.WithTimeout(context.Background(), time.Second)
    defer canc()
    start := time.Now()
    b := newBucket(ctx, 10, time.Second/5)
    t := time.Second / 10
    for i := 0; i < 5; i++ {
        c := client{
            name: fmt.Sprint(i),
            b: b,
            sleep: t,
            max: 5,
        }
        go c.Run(ctx, start)
    }
    <-ctx.Done()
}
```

# 排序

在具有多个 goroutine 的并发场景中，我们可能需要在 goroutine 之间进行同步，例如在每个 goroutine 发送后需要等待轮次的情况下。

这种情况的一个用例可能是一个基于轮次的应用程序，其中不同的 goroutine 正在向同一个通道发送消息，并且每个 goroutine 都必须等到所有其他 goroutine 完成后才能再次发送消息。

可以使用主 goroutine 和发送者之间的私有通道来获得此场景的非常简单的实现。我们可以定义一个非常简单的结构，其中包含消息和`Wait`通道。它将有两种方法-一种用于标记交易已完成，另一种等待这样的信号-当它在下面使用通道时。以下方法显示了这一点：

```go
type msg struct {
    value string
    done chan struct{}
}

func (m *msg) Wait() {
    <-m.done
}

func (m *msg) Done() {
    m.done <- struct{}{}
}
```

我们可以使用生成器创建消息源。我们可以使用`send`操作进行随机延迟。每次发送后，我们等待通过调用`Done`方法获得的信号。我们始终使用上下文来确保一切都不会泄漏：

```go
func send(ctx context.Context, v string) <-chan msg {
    ch := make(chan msg)
    go func() {
        done := make(chan struct{})
        for i := 0; ; i++ {
            time.Sleep(time.Duration(float64(time.Second/2) * rand.Float64()))
            m := msg{fmt.Sprintf("%s msg-%d", v, i), done}
            select {
            case <-ctx.Done():
                close(ch)
                return
            case ch <- m:
                m.Wait()
            }
        }
    }()
    return ch
}
```

我们可以使用 fan-in 将所有通道放入一个单一的通道中：

```go

func merge(ctx context.Context, sources ...<-chan msg) <-chan msg {
    ch := make(chan msg)
    go func() {
        <-ctx.Done()
        close(ch)
    }()
    for i := range sources {
        go func(i int) {
            for {
                select {
                case v := <-sources[i]:
                    select {
                    case <-ctx.Done():
                        return
                    case ch <- v:
                    }
                }
            }
        }(i)
    }
    return ch
}
```

主应用程序将从合并的通道接收，直到它关闭。当它从每个通道接收到一个消息时，通道将被阻塞，等待主 goroutine 调用`Done`方法信号。

这种特定的配置将允许主 goroutine 仅从每个通道接收一个消息。当消息计数达到 goroutine 数量时，我们可以从主 goroutine 调用`Done`并重置列表，以便其他 goroutine 将被解锁并能够再次发送消息：

```go
func main() {
    ctx, canc := context.WithTimeout(context.Background(), time.Second)
    defer canc()
    sources := make([]<-chan msg, 5)
    for i := range sources {
        sources[i] = send(ctx, fmt.Sprint("src-", i))
    }
    msgs := make([]msg, 0, len(sources))
    start := time.Now()
    for v := range merge(ctx, sources...) {
        msgs = append(msgs, v)
        log.Println(v.value, time.Since(start))
        if len(msgs) == len(sources) {
            log.Println("*** done ***")
            for _, m := range msgs {
                m.Done()
            }
            msgs = msgs[:0]
            start = time.Now()
        }
    }
}
```

运行应用程序将导致所有 goroutine 向主 goroutine 发送一条消息。每个 goroutine 都将等待其他人发送消息。然后，他们将开始再次发送消息。这导致消息按轮次发送，正如预期的那样。

# 总结

在本章中，我们研究了一些特定的并发模式，用于我们的应用程序。我们了解到生成器是返回通道的函数，并且还向这些通道提供数据，并在没有更多数据时关闭它们。我们还看到我们可以使用上下文来允许生成器提前退出。

接下来，我们专注于管道，这是使用通道进行通信的执行阶段。它们可以是源，不需要任何输入；目的地，不返回通道；或者中间的，接收通道作为输入并返回一个作为输出。

另一个模式是多路复用和分解复用，它包括将一个通道传播到不同的 goroutine，并将多个通道合并成一个。它通常被称为*扇出扇入*，它允许我们在一组数据上并发执行不同的操作。

最后，我们学习了如何实现一个更好的速率限制器称为**漏桶**，它限制了在特定时间内的请求数。我们还看了顺序模式，它使用私有通道向所有发送 goroutine 发出信号，告诉它们何时可以再次发送数据。

在下一章中，我们将介绍在*顺序*部分中提出的两个额外主题中的第一个。在这里，我们将演示如何使用反射来构建适应任何用户提供的类型的通用代码。

# 问题

1.  生成器是什么？它的责任是什么？

1.  你如何描述一个管道？

1.  什么类型的阶段获得一个通道并返回一个通道？

1.  扇入和扇出之间有什么区别？
