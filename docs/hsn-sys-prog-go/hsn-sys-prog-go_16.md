# 使用 sync 和 atomic 进行同步

本章将继续介绍 Go 并发，介绍`sync`和`atomic`包，这是另外两个用于协调 goroutine 同步的工具。这将使编写优雅且简单的代码成为可能，允许并发使用资源并管理 goroutine 的生命周期。`sync`包含高级同步原语，而`atomic`包含低级原语。

本章将涵盖以下主题：

+   锁

+   等待组

+   其他同步组件

+   `atomic`包

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅第三章，*Go 概述*。

# 同步原语

我们已经看到通道专注于 goroutine 之间的通信，现在我们将关注`sync`包提供的工具，其中包括用于 goroutine 之间同步的基本原语。我们将首先看到如何使用锁实现对同一资源的并发访问。

# 并发访问和锁

Go 提供了一个通用接口，用于可以被锁定和解锁的对象。锁定对象意味着控制它，而解锁则释放它供其他人使用。该接口为每个操作公开了一个方法。以下是代码中的示例：

```go
type Locker interface {
    Lock()
    Unlock()
}
```

# 互斥锁

锁的最简单实现是`sync.Mutex`。由于其方法具有指针接收器，因此不应通过值复制或传递。`Lock()`方法尝试控制互斥锁，如果可能的话，或者阻塞 goroutine 直到互斥锁可用。`Unlock()`方法释放互斥锁，如果在未锁定的情况下调用，则返回运行时错误。

以下是一个简单的示例，我们在其中使用锁启动一堆 goroutine，以查看哪个先执行：

```go
func main() {
    var m sync.Mutex
    done := make(chan struct{}, 10)
    for i := 0; i < cap(done); i++ {
        go func(i int, l sync.Locker) {
            l.Lock()
            defer l.Unlock()
            fmt.Println(i)
            time.Sleep(time.Millisecond * 10)
            done <- struct{}{}
        }(i, &m)
    }
    for i := 0; i < cap(done); i++ {
        <-done
    }
}
```

完整示例可在以下链接找到：[`play.golang.org/p/resVh7LImLf`](https://play.golang.org/p/resVh7LImLf)

我们使用通道来在作业完成时向主 goroutine 发出信号，并退出应用程序。让我们创建一个外部计数器，并使用 goroutine 并发地增加它。

在不同 goroutine 上执行的操作不是线程安全的，如下例所示：

```go
done := make(chan struct{}, 10000)
var a = 0
for i := 0; i < cap(done); i++ {
    go func(i int) {
        if i%2 == 0 {
            a++
        } else {
            a--
        }
        done <- struct{}{}
    }(i)
}
for i := 0; i < cap(done); i++ {
    <-done
}
fmt.Println(a)
```

我们期望得到 5000 加一和 5000 减一，最后一条指令打印出 0。然而，每次运行应用程序时，我们得到的值都不同。这是因为这种操作不是线程安全的，因此两个或更多的操作可能同时发生，最后一个操作会覆盖其他操作。这种现象被称为**竞争条件**；也就是说，多个操作试图写入相同的结果。

这意味着没有任何同步，结果是不可预测的；如果我们检查前面的示例并使用锁来避免竞争条件，我们将得到整数的值为零，这是我们期望的结果：

```go
m := sync.Mutex{}
for i := 0; i < cap(done); i++ {
    go func(l sync.Locker, i int) {
        l.Lock()
        defer l.Unlock()
        if i%2 == 0 {
            a++
        } else {
            a--
        }
        done <- struct{}{}
    }(&m, i)
    fmt.Println(a)
}
```

一个非常常见的做法是在数据结构中嵌入一个互斥锁，以表示要锁定的容器。之前的计数器变量可以表示如下：

```go
type counter struct {
    m     sync.Mutex
    value int
}
```

计数器执行的操作可以是已经在主要操作之前进行了锁定并在之后进行了解锁的方法，如下面的代码块所示：

```go
func (c *counter) Incr(){
    c.m.Lock()
    c.value++
    c.m.Unlock()
}

func (c *counter) Decr(){
    c.m.Lock()
    c.value--
    c.m.Unlock()
}

func (c *counter) Value() int {
    c.m.Lock()
    a := c.value
    c.m.Unlock()
    return a
}
```

这将简化 goroutine 循环，使代码更清晰：

```go
var a = counter{}
for i := 0; i < cap(done); i++ {
    go func(i int) {
        if i%2 == 0 {
            a.Incr()
        } else {
            a.Decr()
        }
        done <- struct{}{}
    }(i)
}
// ...
fmt.Println(a.Value())
```

# RWMutex

竞争条件的问题是由并发写入引起的，而不是由读取操作引起的。实现 locker 接口的另一个数据结构`sync.RWMutex`，旨在支持这两种操作，具有独特的写锁和与读锁互斥。这意味着互斥锁可以被单个写锁或一个或多个读锁锁定。当读者锁定互斥锁时，其他试图锁定它的读者不会被阻塞。它们通常被称为共享-独占锁。这允许读操作同时发生，而不会有等待时间。

使用 locker 接口的`Lock`和`Unlock`方法执行写锁操作。使用另外两种方法执行读取操作：`RLock`和`RUnlock`。还有另一种方法`RLocker`，它返回一个用于读取操作的 locker。

我们可以通过创建一个字符串的并发列表来快速演示它们的用法：

```go
type list struct {
    m sync.RWMutex
    value []string
}
```

我们可以迭代切片以查找所选值，并在读取时使用读锁来延迟写入：

```go
func (l *list) contains(v string) bool {
    for _, s := range l.value {
        if s == v {
            return true
        }
    }
    return false
}

func (l *list) Contains(v string) bool {
    l.m.RLock()
    found := l.contains(v)
    l.m.RUnlock()
    return found
}
```

在添加新元素时，我们可以使用写锁：

```go
func (l *list) Add(v string) bool {
    l.m.Lock()
    defer l.m.Unlock()
    if l.contains(v) {
        return false
    }
    l.value = append(l.value, v)
    return true
}
```

然后我们可以尝试使用多个 goroutines 在列表上执行相同的操作：

```go
var src = []string{
    "Ryu", "Ken", "E. Honda", "Guile",
    "Chun-Li", "Blanka", "Zangief", "Dhalsim",
}
var l list
for i := 0; i < 10; i++ {
    go func(i int) {
        for _, s := range src {
            go func(s string) {
                if !l.Contains(s) {
                    if l.Add(s) {
                        fmt.Println(i, "add", s)
                    } else {
                        fmt.Println(i, "too slow", s)
                    }
                }
            }(s)
        }
    }(i)
}
time.Sleep(500 * time.Millisecond)
```

首先我们检查名称是否包含在锁中，然后尝试添加元素。这会导致多个例程尝试添加新元素，但由于写锁是排他的，只有一个会成功。

# 写入饥饿

在设计应用程序时，这种类型的互斥锁并不总是显而易见的选择，因为在读锁的数量更多而写锁的数量较少的情况下，互斥锁将在第一个读锁之后接受更多的读锁，让写入操作等待没有活动的读锁的时刻。这是一种被称为**写入饥饿**的现象。

为了验证这一点，我们可以定义一个类型，其中包含写入和读取操作，这需要一些时间，如下面的代码所示：

```go
type counter struct {
    m sync.RWMutex
    value int
}

func (c *counter) Write(i int) {
    c.m.Lock()
    time.Sleep(time.Millisecond * 100)
    c.value = i
    c.m.Unlock()
}

func (c *counter) Value() int {
    c.m.RLock()
    time.Sleep(time.Millisecond * 100)
    a := c.value
    c.m.RUnlock()
    return a
}
```

我们可以尝试在单独的 goroutines 中以相同的节奏执行写入和读取操作，使用低于方法执行时间的持续时间（50 毫秒与 100 毫秒）。我们还将检查它们在锁定状态下花费了多少时间：

```go
var c counter
t1 := time.NewTicker(time.Millisecond * 50)
time.AfterFunc(time.Second*2, t1.Stop)
for {
    select {
    case <-t1.C:
        go func() {
            t := time.Now()
            c.Value()
            fmt.Println("val", time.Since(t))
        }()
        go func() {
            t := time.Now()
            c.Write(0)
            fmt.Println("inc", time.Since(t))
        }()
    case <-time.After(time.Millisecond * 200):
        return
    }
}
```

如果我们执行应用程序，我们会发现对于每个写入操作，都会执行多次读取，并且每次调用都会花费比上一次更多的时间，等待锁。这对于读取操作并不成立，因为它可以同时进行，所以一旦读者成功锁定资源，所有其他等待的读者也会这样做。将`RWMutex`替换为`Mutex`将使两种操作具有相同的优先级，就像前面的例子一样。

# 锁定陷阱

在锁定和解锁互斥锁时必须小心，以避免应用程序中的意外行为和死锁。参考以下代码片段：

```go
for condition {
    mu.Lock()
    defer mu.Unlock()
    action()
}
```

这段代码乍一看似乎没问题，但它将不可避免地阻塞 goroutine。这是因为`defer`语句不是在每次循环迭代结束时执行，而是在函数返回时执行。因此，第一次尝试将锁定而不释放，第二次尝试将保持锁定状态。

稍微重构一下可以帮助解决这个问题，如下面的代码片段所示：

```go
for condition {
    func() {
        mu.Lock()
        defer mu.Unlock()
        action()
    }()
}
```

我们可以使用闭包来确保即使`action`发生恐慌，也会执行延迟的`Unlock`。

如果在互斥锁上执行的操作不会引起恐慌，可以考虑放弃延迟，只在执行操作后使用它，如下所示：

```go
for condition {
    mu.Lock()
    action()
    mu.Unlock()
}
```

`defer`是有成本的，因此最好在不必要时避免使用它，例如在进行简单的变量读取或赋值时。

# 同步 goroutines

到目前为止，为了等待 goroutines 完成，我们使用了一个空结构的通道，并通过通道发送一个值作为最后一个操作，如下所示：

```go
ch := make(chan struct{})
for i := 0; i < n; n++ {
    go func() {
        // do something
        ch <- struct{}{}
    }()
}
for i := 0; i < n; n++ {
    <-ch
}
```

这种策略有效，但不是实现任务的首选方式。从语义上讲不正确，因为我们使用通道，而通道是用于通信的工具，用于发送空数据。这种用例是关于同步而不是通信。这就是为什么有`sync.WaitGroup`数据结构，它涵盖了这种情况。它有一个主要状态，称为计数器，表示等待的元素数量：

```go
type WaitGroup struct {
    noCopy noCopy
    state1 [3]uint32
}
```

`noCopy`字段防止结构通过`panic`按值复制。状态是由三个`int32`组成的数组，但只使用第一个和最后一个条目；剩下的一个用于编译器优化。

`WaitGroup`提供了三种方法来实现相同的结果：

+   `Add`：使用给定值更改计数器的值，该值也可以是负数。如果计数器小于零，应用程序将会 panic。

+   `Done`：这是`Add`的简写，参数为`-1`。通常在 goroutine 完成其工作时调用，将计数器减 1。

+   `Wait`：此操作会阻塞当前 goroutine，直到计数器达到零。

使用等待组可以使代码更清晰和可读，如下例所示：

```go
func main() {
    wg := sync.WaitGroup{}
    wg.Add(10)
    for i := 1; i <= 10; i++ {
        go func(a int) {
            for i := 1; i <= 10; i++ {
                fmt.Printf("%dx%d=%d\n", a, i, a*i)
            }
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

对于等待组，我们正在添加一个等于 goroutines 的`delta`，我们将在之前启动。在每个单独的 goroutine 中，我们使用`Done`方法来减少计数。如果 goroutines 的数量未知，则可以在启动每个 goroutine 之前执行`Add`操作（参数为`1`），如下所示：

```go
func main() {
    wg := sync.WaitGroup{}
    for i := 1; rand.Intn(10) != 0; i++ {
        wg.Add(1)
        go func(a int) {
            for i := 1; i <= 10; i++ {
                fmt.Printf("%dx%d=%d\n", a, i, a*i)
            }
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

在前面的示例中，我们每次`for`循环迭代有 10%的机会完成，因此在启动 goroutine 之前我们会向组中添加一个。

一个非常常见的错误是在 goroutine 内部添加值，这通常会导致在没有执行任何 goroutine 的情况下过早退出。这是因为应用程序在创建 goroutines 并执行`Wait`函数之前开始并添加它们自己的增量，如下例所示：

```go
func main() {
    wg := sync.WaitGroup{}
    for i := 1; i < 10; i++ {
        go func(a int) {
            wg.Add(1)
            for i := 1; i <= 10; i++ {
                fmt.Printf("%dx%d=%d\n", a, i, a*i)
            }
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

此应用程序不会打印任何内容，因为它在任何 goroutine 启动和调用`Add`方法之前到达`Wait`语句。

# Go 中的单例

单例模式是软件开发中常用的策略。这涉及将某种类型的实例数量限制为一个，并在整个应用程序中使用相同的实例。该概念的一个非常简单的实现可能是以下代码：

```go
type obj struct {}

var instance *obj

func Get() *obj{
    if instance == nil {
        instance = &obj{}
    }
    return instance
}
```

这在连续的情况下是完全可以的，但在并发的情况下，就像许多 Go 应用程序一样，这是不安全的，并且可能会产生竞争条件。

通过添加一个锁，可以使前面的示例线程安全，从而避免任何竞争条件，如下所示：

```go
type obj struct {}

var (
    instance *obj
    lock     sync.Mutex
)

func Get() *obj{
    lock.Lock()
    defer lock.Unlock()
    if instance == nil {
        instance = &obj{}
    }
    return instance
}
```

这是安全的，但速度较慢，因为`Mutex`将在每次请求实例时进行同步。

实现此模式的最佳解决方案是使用`sync.Once`结构，如下例所示，它负责使用`Mutex`和`atomic`读取一次执行函数（我们将在本章的第二部分中看到）：

```go
type obj struct {}

var (
    instance *obj
    once     sync.Once
)

func Get() *obj{
    once.Do(func(){
        instance = &obj{}
    })
    return instance
}
```

结果代码是惯用的和清晰的，与互斥解决方案相比性能更好。由于操作只会执行一次，我们还可以摆脱在先前示例中对实例进行的`nil`检查。

# 一次和重置

`sync.Once`函数用于执行另一个函数一次，不再执行。有一个非常有用的第三方库，允许我们使用`Reset`方法重置单例的状态。

包的源代码可以在以下位置找到：[github.com/matryer/resync](https://github.com/matryer/resync)。

典型用途包括一些需要在特定错误上再次执行的初始化，例如获取 API 密钥或在连接中断时重新拨号。

# 资源回收

我们已经看到如何在上一章中使用具有工作池的缓冲通道来实现资源回收。将有两种方法如下：

+   一个`Get`方法，尝试从通道接收消息或返回一个新实例。

+   一个`Put`方法，尝试将实例返回到通道或丢弃它。

这是一个使用通道实现的简单池的实现：

```go
type A struct{}

type Pool chan *A

func (p Pool) Get() *A {
    select {
    case a := <-p:
        return a
    default:
        return new(A)
    }
}

func (p Pool) Put(a *A) {
    select {
    case p <- a:
    default:
    }
}
```

我们可以使用`sync.Pool`结构来改进这一点，它实现了一个线程安全的对象集，可以保存或检索。唯一需要定义的是当创建一个新对象时池的行为：

```go
type Pool struct {
    // New optionally specifies a function to generate
    // a value when Get would otherwise return nil.
    // It may not be changed concurrently with calls to Get.
    New func() interface{}
    // contains filtered or unexported fields
}
```

池提供两种方法：`Get`和`Put`。这些方法从池中返回对象（或创建新对象），并将对象放回池中。由于`Get`方法返回一个`interface{}`，因此需要将值转换为特定类型才能正确使用。我们已经广泛讨论了缓冲区回收，在以下示例中，我们将尝试使用`sync.Pool`来实现缓冲区回收。

我们需要定义池和函数来获取和释放新的缓冲区。我们的缓冲区将具有 4 KB 的初始容量，并且`Put`函数将确保在将其放回池之前重置缓冲区，如以下代码示例所示：

```go
var pool = sync.Pool{
    New: func() interface{} {
        return bytes.NewBuffer(make([]byte, 0, 4096))
    },
}

func Get() *bytes.Buffer {
    return pool.Get().(*bytes.Buffer)
}

func Put(b *bytes.Buffer) {
    b.Reset()
    pool.Put(b)
}
```

现在我们将创建一系列 goroutine，它们将使用`WaitGroup`来在完成时发出信号，并将执行以下操作：

+   等待一定时间（1-5 秒）。

+   获取一个缓冲区。

+   在缓冲区上写入信息。

+   将内容复制到标准输出。

+   释放缓冲区。

我们将使用等于`1`秒的睡眠时间，每`4`次循环增加一秒，最多达到`5`秒：

```go
start := time.Now()
wg := sync.WaitGroup{}
wg.Add(20)
for i := 0; i < 20; i++ {
    go func(v int) {
        time.Sleep(time.Second * time.Duration(1+v/4))
        b := Get()
        defer func() {
            Put(b)
            wg.Done()
        }()
        fmt.Fprintf(b, "Goroutine %2d using %p, after %.0fs\n", v, b, time.Since(start).Seconds())
        fmt.Printf("%s", b.Bytes())
    }(i)
}
wg.Wait()
```

打印的信息还包含缓冲区内存地址。这将帮助我们确认缓冲区始终相同，没有创建新的缓冲区。

# 切片回收问题

对于具有基础字节片的数据结构，例如`bytes.Buffer`，在与`sync.Pool`或类似的回收机制结合使用时，我们应该小心。让我们改变先前的示例，收集缓冲区的字节而不是将它们打印到标准输出。以下是此示例的示例代码：

```go
var (
    list = make([][]byte, 20)
    m sync.Mutex
)
for i := 0; i < 20; i++ {
    go func(v int) {
        time.Sleep(time.Second * time.Duration(1+v/4))
        b := Get()
        defer func() {
            Put(b)
            wg.Done()
        }()
        fmt.Fprintf(b, "Goroutine %2d using %p, after %.0fs\n", v, b, time.Since(start).Seconds())
        m.Lock()
        list[v] = b.Bytes()
        m.Unlock()
    }(i)
}
wg.Wait()
```

那么，当我们打印字节片段列表时会发生什么？我们可以在以下示例中看到这一点：

```go

for i := range list {
    fmt.Printf("%d - %s", i, list[i])
}
```

由于缓冲区正在重用相同的基础切片，并且在每次新使用时覆盖内容，因此我们得到了意外的结果。

通常解决此问题的方法是执行字节的副本，而不仅仅是分配它们：

```go
m.Lock()
list[v] = make([]byte, b.Len())
copy(list[v], b.Bytes())
m.Unlock()
```

# 条件

在并发编程中，条件变量是一个同步机制，其中包含等待相同条件进行验证的线程。在 Go 中，这意味着有一些 goroutine 在等待某些事情发生。我们已经使用单个 goroutine 等待通道的实现，如以下示例所示：

```go
ch := make(chan struct{})
go func() {
    // do something
    ch <- struct{}{}
}()
go func() {
    // wait for condition
    <-ch
    // do something else
}
```

这种方法仅限于单个 goroutine，但可以改进为支持更多侦听器，从发送消息切换到关闭通道：

```go
go func() {
    // do something
    close(ch)
}()
for i := 0; i < n; i++ {
    go func() {
        // wait for condition
        <-ch
        // do something else
    }()
}
```

关闭通道适用于多个侦听器，但在关闭后不允许它们进一步使用通道。

`sync.Cond`类型是一个工具，可以更好地处理所有这些行为。它在实现中使用锁，并公开三种方法：

+   `Broadcast`：这会唤醒等待条件的所有 goroutine。

+   `Signal`：如果至少有一个条件，则唤醒等待条件的单个 goroutine。

+   `Wait`：这会解锁锁定器，暂停 goroutine 的执行，稍后恢复执行并再次锁定它，等待`Broadcast`或`Signal`。

这不是必需的，但可以在持有锁时执行`Broadcast`和`Signal`操作，在调用之前锁定它，之后释放它。`Wait`方法要求在调用之前持有锁，并在使用条件后解锁。

让我们创建一个并发应用程序，该应用程序使用`sync.Cond`来协调更多的 goroutines。我们将从命令行获得提示，每条记录将被写入一系列文件。我们将有一个主结构来保存所有数据：

```go
type record struct {
    sync.Mutex
    buf string
    cond *sync.Cond
    writers []io.Writer
}
```

我们将监视的条件是`buf`字段的更改。在`Run`方法中，`record`结构将启动多个 goroutines，每个写入者一个。每个 goroutine 将等待条件触发，并将写入其文件：

```go
func (r *record) Run() {
    for i := range r.writers {
        go func(i int) {
            for {
                r.Lock()
                r.cond.Wait()
                fmt.Fprintf(r.writers[i], "%s\n", r.buf)
                r.Unlock()
            }
        }(i)
    }
}
```

我们可以看到，在使用`Wait`之前锁定条件，并在使用我们条件引用的值之后解锁它。主函数将根据提供的命令行参数创建一个记录和一系列文件：

```go
// let's make sure we have at least a file argument
if len(os.Args) < 2 {
    log.Fatal("Please specify at least a file")
}
r := record{
    writers: make([]io.Writer, len(os.Args)-1),
}
r.cond = sync.NewCond(&r)
for i, v := range os.Args[1:] {
    f, err := os.Create(v)
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    r.writers[i] = f
}
r.Run()
```

然后我们将使用`bufio.Scanner`读取行并广播`buf`字段的更改。我们还将接受特殊值`\q`作为退出命令：

```go
scanner := bufio.NewScanner(os.Stdin)
for {
    fmt.Printf(":> ")
    if !scanner.Scan() {
        break
    }
    r.Lock()
    r.buf = scanner.Text()
    r.Unlock()
    switch {
    case r.buf == `\q`:
        return
    default:
        r.cond.Broadcast()
    }
}
```

我们可以看到，在持有锁时更改`buf`，然后调用`Broadcast`唤醒等待条件的所有 goroutines。

# 同步地图

Go 中的内置地图不是线程安全的，因此尝试从不同的 goroutines 进行写入可能会导致运行时错误：`concurrent map writes`。我们可以使用一个简单的程序来验证这一点，该程序尝试并发进行更改：

```go
func main() {
    var m = map[int]int{}
    wg := sync.WaitGroup{}
    wg.Add(10)
    for i := 0; i < 10; i++ {
        go func(i int) {
            m[i%5]++
            fmt.Println(m)
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

在写入时进行读取也会导致运行时错误，即`concurrent map iteration and map write`，我们可以通过运行以下示例来看到这一点：

```go
func main() {
    var m = map[int]int{}
    var done = make(chan struct{})
    go func() {
        for i := 0; i < 100; i++ {
            time.Sleep(time.Nanosecond)
            m[i]++
        }
        close(done)
    }()
    for {
        time.Sleep(time.Nanosecond)
        fmt.Println(len(m), m)
        select {
        case <-done:
            return
        default:
        }
    }
}
```

有时，尝试迭代地图（如`Print`语句所做的那样）可能会导致恐慌，例如`index out of range`，因为内部切片可能已经在其他地方分配了。

使地图并发的一个非常简单的策略是将其与`sync.Mutex`或`sync.RWMutex`配对。这样可以在执行操作时锁定地图：

```go
type m struct {
    sync.Mutex
    m map[int]int
}
```

我们使用地图来获取或设置值，例如以下示例：

```go
func (m *m) Get(key int) int {
    m.Lock()
    a := m.m[key]
    m.Unlock()
    return a
}

func (m *m) Put(key, value int) {
    m.Lock()
    m.m[key] = value
    m.Unlock()
}
```

我们还可以传递一个接受键值对并对每个元组执行的函数，同时锁定地图：

```go
func (m *m) Range(f func(k, v int)) {
    m.Lock()
    for k, v := range m.m {
        f(k, v)
    }
    m.Unlock()
}
```

Go 1.9 引入了一个名为`sync.Map`的结构，它正是这样做的。它是一个非常通用的`map[interface{}]interface{}`，可以使用以下方法执行线程安全操作：

+   `Load`：从地图中获取给定键的值。

+   `Store`：为给定的键在地图中设置一个值。

+   `Delete`：从地图中删除给定键的条目。

+   `LoadOrStore`：返回键的值（如果存在）或存储的值。

+   `Range`：调用一个函数，该函数针对地图中的每个键值对返回一个布尔值。如果返回`false`，则迭代停止。

我们可以在以下代码片段中看到这是如何工作的，我们尝试同时进行多次写入：

```go
func main() {
    var m = sync.Map{}
    var wg = sync.WaitGroup{}
    wg.Add(1000)
    for i := 0; i < 1000; i++ {
        go func(i int) {
            m.LoadOrStore(i, i)
            wg.Done()
        }(i)
    }
    wg.Wait()
    i := 0
    m.Range(func(k, v interface{}) bool {
        i++
        return true
    })
   fmt.Println(i)
}
```

与具有常规`Map`的版本不同，此应用程序不会崩溃并执行所有操作。

# 信号量

在上一章中，我们看到可以使用通道创建加权信号量。在实验性的`sync`包中有更好的实现。可以在[golang.org/x/sync/semaphore](https://godoc.org/golang.org/x/sync/semaphore)找到。

这种实现使得可以创建一个新的信号量，使用`semaphore.NewWeighted`指定权重。

可以使用`Acquire`方法获取配额，指定要获取的配额数量。这些可以使用`Release`方法释放，如以下示例所示：

```go
func main() {
    s := semaphore.NewWeighted(int64(10))
    ctx := context.Background()
    for i := 0; i < 20; i++ {
        if err := s.Acquire(ctx, 1); err != nil {
            log.Fatal(err)
        }
        go func(i int) {
            fmt.Println(i)
            s.Release(1)
        }(i)
    }
    time.Sleep(time.Second)
}
```

获取配额除了数字之外还需要另一个参数，即`context.Context`。这是 Go 中可用的另一个并发工具，我们将在下一章中看到如何使用它。

# 原子操作

`sync`包提供了同步原语，在底层使用整数和指针的线程安全操作。我们可以在另一个名为`sync/atomic`的包中找到这些功能，该包可用于创建特定于用户用例的工具，具有更好的性能和更少的内存使用。

# 整数操作

有一系列针对不同类型整数的指针的函数：

+   `int32`

+   `int64`

+   `uint32`

+   `uint64`

+   `uintptr`

这包括表示指针的特定类型的整数，`uintptr`。这些类型可用的操作如下：

+   `Load`：从指针中检索整数值

+   `Store`：将整数值存储在指针中

+   `Add`：将指定的增量添加到指针值

+   `Swap`：将新值存储在指针中并返回旧值

+   `CompareAndSwap`：仅当新值与指定值相同时才将其交换

# 点击器

这个函数对于非常容易定义线程安全的组件非常有帮助。一个非常明显的例子可能是一个简单的整数计数器，它使用`Add`来改变计数器，`Load`来检索当前值，`Store`来重置它：

```go
type clicker int32

func (c *clicker) Click() int32 {
    return atomic.AddInt32((*int32)(c), 1)
}

func (c *clicker) Reset() {
    atomic.StoreInt32((*int32)(c), 0)
}

func (c *clicker) Value() int32 {
    return atomic.LoadInt32((*int32)(c))
}
```

我们可以在一个简单的程序中看到它的运行情况，该程序尝试同时读取、写入和重置计数器。

我们定义`clicker`和`WaitGroup`，并将正确数量的元素添加到等待组中，如下所示：

```go
c := clicker(0)
wg := sync.WaitGroup{}
// 2*iteration + reset at 5
wg.Add(21)
```

我们可以启动一堆不同操作的 goroutines，比如：10 次读取，10 次添加和一次重置：

```go
for i := 0; i < 10; i++ {
    go func() {
        c.Click()
        fmt.Println("click")
        wg.Done()
    }()
    go func() {
        fmt.Println("load", c.Value())
        wg.Done()
    }()
    if i == 0 || i%5 != 0 {
        continue
    }
    go func() {
        c.Reset()
        fmt.Println("reset")
        wg.Done()
    }()
}
wg.Wait()
```

我们将看到点击器按照预期的方式执行并发求和而没有竞争条件。

# 线程安全的浮点数

`atomic`包仅提供整数的原语，但由于`float32`和`float64`存储在与`int32`和`int64`相同的数据结构中，我们使用它们来创建原子浮点值。

诀窍是使用`math.Floatbits`函数将浮点数表示为无符号整数，以及使用`math.Floatfrombits`函数将无符号整数转换为浮点数。让我们看看这如何在`float64`中工作：

```go
type f64 uint64

func uf(u uint64) (f float64) { return math.Float64frombits(u) }
func fu(f float64) (u uint64) { return math.Float64bits(f) }

func newF64(f float64) *f64 {
    v := f64(fu(f))
    return &v
}

func (f *f64) Load() float64 {
  return uf(atomic.LoadUint64((*uint64)(f)))
}

func (f *f64) Store(s float64) {
  atomic.StoreUint64((*uint64)(f), fu(s))
}
```

创建`Add`函数有点复杂。我们需要使用`Load`获取值，然后比较和交换。由于这个操作可能失败，因为加载是一个`atomic`操作，**比较和交换**（**CAS**）是另一个，我们在循环中不断尝试直到成功：

```go
func (f *f64) Add(s float64) float64 {
    for {
        old := f.Load()
        new := old + s
        if f.CompareAndSwap(old, new) {
            return new
        }
    }
}

func (f *f64) CompareAndSwap(old, new float64) bool {
    return atomic.CompareAndSwapUint64((*uint64)(f), fu(old), fu(new))
}
```

# 线程安全的布尔值

我们也可以使用`int32`来表示布尔值。我们可以使用整数`0`作为`false`，`1`作为`true`，创建一个线程安全的布尔条件：

```go
type cond int32

func (c *cond) Set(v bool) {
    a := int32(0)
    if v {
        a++
    }
    atomic.StoreInt32((*int32)(c), a)
}

func (c *cond) Value() bool {
    return atomic.LoadInt32((*int32)(c)) != 0
}
```

这将允许我们将`cond`类型用作线程安全的布尔值。

# 指针操作

Go 中的指针变量存储在`intptr`变量中，这些整数足够大以容纳内存地址。`atomic`包使得可以对其他整数类型执行相同的操作。有一个允许不安全指针操作的包，它提供了`unsafe.Pointer`类型，用于原子操作。

在下面的示例中，我们定义了两个整数变量及其相关的整数指针。然后我们执行第一个指针与第二个指针的交换：

```go
v1, v2 := 10, 100
p1, p2 := &v1, &v2
log.Printf("P1: %v, P2: %v", *p1, *p2)
atomic.SwapPointer((*unsafe.Pointer)(unsafe.Pointer(&p1)), unsafe.Pointer(p2))
log.Printf("P1: %v, P2: %v", *p1, *p2)
v1 = -10
log.Printf("P1: %v, P2: %v", *p1, *p2)
v2 = 3
log.Printf("P1: %v, P2: %v", *p1, *p2)
```

交换后，两个指针现在都指向第二个变量；对第一个值的任何更改都不会影响指针。更改第二个变量会改变指针所指的值。

# 值

我们可以使用的最简单的工具是`atomic.Value`。它保存`interface{}`，并且可以通过线程安全地读取和写入它。它公开了两种方法，`Store`和`Load`，这使得设置或检索值成为可能。正如其他线程安全工具一样，`sync.Value`在第一次使用后不能被复制。

我们可以尝试有许多 goroutines 来设置和读取相同的值。每次加载操作都会获取最新存储的值，并且并发时不会出现错误：

```go
func main() {
    var (
        v atomic.Value
        wg sync.WaitGroup
    )
    wg.Add(20)
    for i := 0; i < 10; i++ {
        go func(i int) {
            fmt.Println("load", v.Load())
            wg.Done()
        }(i)
        go func(i int) {
            v.Store(i)
            fmt.Println("store", i)
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

这是一个非常通用的容器；它可以用于任何类型的变量，变量类型应该从一个变为另一个。如果具体类型发生变化，它将使方法恐慌；同样的情况也适用于`nil`空接口。

# 底层

`sync.Value`类型将其数据存储在一个非公开的接口中，如源代码所示：

```go
type Value struct {
    v interface{}
}
```

它使用`unsafe`包的一种类型来将该结构转换为另一个具有与接口相同的数据结构：

```go
type ifaceWords struct {
    typ unsafe.Pointer
    data unsafe.Pointer
}
```

具有完全相同内存布局的两种类型可以以这种方式转换，跳过 Go 的类型安全性。这使得可以使用指针进行 `atomic` 操作，并执行线程安全的 `Store` 和 `Load` 操作。

为了写入值获取锁，`atomic.Value` 使用与类型中的 `unsafe.Pointer(^uintptr(0))` 值（即 `0xffffffff`）进行比较和交换操作；它改变值并用正确的值替换类型。

同样，加载操作会循环，直到类型不同于 `0xffffffff`，然后尝试读取值。

使用这种方法，`atomic.Value` 能够使用其他 `atomic` 操作存储和加载任何值。

# 总结

在本章中，我们看到了 Go 标准包中用于同步的工具。它们位于两个包中：`sync`，提供诸如互斥锁之类的高级工具，以及 `sync/atomic`，执行低级操作。

首先，我们看到了如何使用锁同步数据。我们看到了如何使用 `sync.Mutex` 来锁定资源，而不管操作类型如何，并使用 `sync.RWMutex` 允许并发读取和阻塞写入。我们应该小心使用第二个，因为连续读取可能会延迟写入。

接下来，我们看到了如何跟踪正在运行的操作，以便等待一系列 goroutine 的结束，使用 `sync.WaitGroup`。这充当当前 goroutine 的线程安全计数器，并使得可以使用 `Wait` 方法将当前 goroutine 置于休眠状态，直到计数达到零。

此外，我们检查了 `sync.Once` 结构，用于执行功能一次，例如允许实现线程安全的单例。然后我们使用 `sync.Pool` 在可能的情况下重用实例而不是创建新实例。池需要的唯一东西是返回新实例的函数。

`sync.Condition` 结构表示特定条件并使用锁来改变它，允许 goroutine 等待改变。这可以使用 `Signal` 传递给单个 goroutine，或者使用 `Broadcast` 传递给所有 goroutine。该包还提供了 `sync.Map` 的线程安全版本。

最后，我们检查了 `atomic` 的功能，这些功能主要是整数线程安全操作：加载、保存、添加、交换和 CAS。我们还看到了 `atomic.Value`，它使得可以并发更改接口的值，并且在第一次更改后不允许更改类型。

下一章将介绍 Go 并发中引入的最新元素：`Context`，这是一个处理截止日期、取消等的接口。

# 问题

1.  什么是竞争条件？

1.  当您尝试并发执行地图的读取和写入操作时会发生什么？

1.  `Mutex` 和 `RWMutex` 之间有什么区别？

1.  等待组有什么用？

1.  `Once` 的主要用途是什么？

1.  您如何使用 `Pool`？

1.  使用原子操作的优势是什么？
