# 第十三章：使用上下文进行协调

本章是关于相对较新的上下文包及其在并发编程中的使用。它是一个非常强大的工具，通过定义一个在标准库中的许多不同位置以及许多第三方包中使用的独特接口。

本章将涵盖以下主题：

+   理解上下文是什么

+   在标准库中研究其用法

+   创建使用上下文的包

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅第三章，*Go 概述*。

# 理解上下文

上下文是在 1.7 版本中进入标准库的相对较新的组件。它是用于 goroutine 之间同步的接口，最初由 Go 团队内部使用，最终成为语言的核心部分。

# 接口

该包中的主要实体是`Context`本身，它是一个接口。它只有四种方法：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

让我们在这里了解这四种方法：

+   `Deadline`：返回上下文应该被取消的时间，以及一个布尔值，当没有截止日期时为`false`

+   `Done`：返回一个只接收空结构的通道，用于信号上下文应该被取消

+   `Err`：当`done`通道打开时返回`nil`；否则返回上下文取消的原因

+   `Value`：返回与当前上下文中的键关联的值，如果该键没有值，则返回`nil`

与标准库的其他接口相比，上下文具有许多方法，通常只有一两个方法。其中三个方法密切相关：

+   `Deadline`是取消的时间

+   `Done`信号上下文已完成

+   `Err`返回取消的原因

最后一个方法`Value`返回与某个键关联的值。包的其余部分是一系列函数，允许您创建不同类型的上下文。让我们浏览包含在该包中的各种函数，并查看创建和装饰上下文的各种工具。

# 默认上下文

`TODO`和`Background`函数返回`context.Context`，无需任何输入参数。返回的值是一个空上下文，它们之间的区别只是语义上的。

# Background

`Background`是一个空上下文，不会被取消，没有截止日期，也不保存任何值。它主要由`main`函数用作根上下文或用于测试目的。以下是此上下文的一些示例代码：

```go
func main() {
    ctx := context.Background()
    done := ctx.Done()
    for i :=0; ;i++{
        select {
        case <-done:
            return
        case <-time.After(time.Second):
            fmt.Println("tick", i)
        }
    }
}
```

完整示例可在此处找到：[`play.golang.org/p/y_3ip7sdPnx`](https://play.golang.org/p/y_3ip7sdPnx)。

我们可以看到，在示例的上下文中，循环无限进行，因为上下文从未完成。

# TODO

`TODO`是另一个空上下文，当上下文的范围不清楚或上下文的类型尚不可用时应使用。它的使用方式与`Background`完全相同。实际上，在底层，它们是相同的东西；区别只是语义上的。如果我们查看源代码，它们具有完全相同的定义：

```go
var (
    background = new(emptyCtx)
    todo = new(emptyCtx)
)
```

该代码的源代码可以在[`golang.org/pkg/context/?m=all#pkg-variables`](https://golang.org/pkg/context/?m=all#pkg-variables)找到。

可以使用包的其他函数来扩展这些基本上下文。它们将充当装饰器，并为它们添加更多功能。

# 取消、超时和截止日期

我们查看的上下文从未被取消，但该包提供了不同的选项来添加此功能。

# 取消

`context.WithCancel`装饰器函数获取一个上下文并返回另一个上下文和一个名为`cancel`的函数。返回的上下文将是具有不同`done`通道（标记当前上下文完成的通道）的上下文的副本，当父上下文完成或调用`cancel`函数时关闭该通道-无论哪个先发生。

在以下示例中，我们可以看到在调用`cancel`函数之前等待几秒钟，程序正确终止。`Err`的值是`context.Canceled`变量：

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    time.AfterFunc(time.Second*5, cancel)
    done := ctx.Done()
    for i := 0; ; i++ {
        select {
        case <-done:
            fmt.Println("exit", ctx.Err())
            return
        case <-time.After(time.Second):
            fmt.Println("tick", i)
        }
    }
}
```

完整示例在这里：[`play.golang.org/p/fNHLIZL8e0L`](https://play.golang.org/p/fNHLIZL8e0L)。

# 截止时间

`context.WithDeadline`是另一个装饰器，它将`time.Time`作为时间截止时间，并将其应用于另一个上下文。如果已经有截止时间并且早于提供的截止时间，则指定的截止时间将被忽略。如果在截止时间到达时`done`通道仍然打开，则会自动关闭它。

在以下示例中，我们将截止时间设置为现在的 5 秒后，并在 10 秒后调用`cancel`。截止时间在取消之前到达，`Err`返回`context.DeadlineExceeded`错误：

```go
func main() {
    ctx, cancel := context.WithDeadline(context.Background(), 
         time.Now().Add(5*time.Second))
    time.AfterFunc(time.Second*10, cancel)
    done := ctx.Done()
    for i := 0; ; i++ {
        select {
        case <-done:
            fmt.Println("exit", ctx.Err())
            return
        case <-time.After(time.Second):
            fmt.Println("tick", i)
        }
    }
}
```

完整示例在这里：[`play.golang.org/p/iyuOmd__CGH`](https://play.golang.org/p/iyuOmd__CGH)。

我们可以看到前面的示例的行为与预期完全一致。它将打印`tick`语句每秒几次，直到截止时间到达并返回错误。

# 超时

最后一个与取消相关的装饰器是`context.WithTimeout`，它允许您指定`time.Duration`以及上下文，并在超时时自动关闭`done`通道。

如果有截止时间活动，则新值仅在早于父级时应用。我们可以看一个几乎相同的示例，除了上下文定义之外，得到与截止时间示例相同的结果：

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(),5*time.Second)
    time.AfterFunc(time.Second*10, cancel)
    done := ctx.Done()
    for i := 0; ; i++ {
        select {
        case <-done:
            fmt.Println("exit", ctx.Err())
            return
        case <-time.After(time.Second):
            fmt.Println("tick", i)
        }
    }
}
```

完整示例在这里：[`play.golang.org/p/-Zp63_e0zYD`](https://play.golang.org/p/-Zp63_e0zYD)。

# 键和值

`context.WithValue`函数创建了一个父上下文的副本，其中给定的键与指定的值相关联。它的范围包含相对于单个请求的值，而在处理过程中不应该用于其他范围，例如可选的函数参数。

键应该是可以比较的东西，最好避免使用`string`值，因为使用上下文的两个不同包可能会覆盖彼此的值。建议使用用户定义的具体类型，如`struct{}`。

在这里，我们可以看到一个示例，我们使用空结构作为键，为每个 goroutine 添加不同的值：

```go
type key struct{}

type key struct{}

func main() {
    ctx, canc := context.WithCancel(context.Background())
    wg := sync.WaitGroup{}
    wg.Add(5)
    for i := 0; i < 5; i++ {
        go func(ctx context.Context) {
            v := ctx.Value(key{})
            fmt.Println("key", v)
            wg.Done()
            <-ctx.Done()
            fmt.Println(ctx.Err(), v)
        }(context.WithValue(ctx, key{}, i))
    }
    wg.Wait()
    canc()
    time.Sleep(time.Second)
}

```

完整示例在这里：[`play.golang.org/p/lM61u_QKEW1`](https://play.golang.org/p/lM61u_QKEW1)。

我们还可以看到取消父级会取消其他上下文。另一个有效的键类型可以是导出的指针值，即使底层数据相同也不会相同：

```go
type key *int

func main() {
    k := new(key)
    ctx, canc := context.WithCancel(context.Background())
    wg := sync.WaitGroup{}
    wg.Add(5)
    for i := 0; i < 5; i++ {
        go func(ctx context.Context) {
            v := ctx.Value(k)
            fmt.Println("key", v, ctx.Value(new(key)))
            wg.Done()
            <-ctx.Done()
            fmt.Println(ctx.Err(), v)
        }(context.WithValue(ctx, k, i))
    }
    wg.Wait()
    canc()
    time.Sleep(time.Second)
}
```

完整示例在这里：[`play.golang.org/p/05XJwWF0-0n`](https://play.golang.org/p/05XJwWF0-0n)。

我们可以看到，定义具有相同底层值的键指针不会返回预期的值。

# 标准库中的上下文

现在我们已经介绍了包的内容，我们将看看如何在标准包或应用程序中使用它们。上下文在标准包中的一些函数和方法中使用，主要是网络包。现在让我们来看看它们：

+   `http.Server`使用`Shutdown`方法，以便完全控制超时或取消操作。

+   `http.Request`允许您使用`WithContext`方法设置上下文。它还允许您使用`Context`获取当前上下文。

+   在`net`包中，`Listen`，`Dial`和`Lookup`有一个使用`Context`来控制截止时间和超时的版本。

+   在`database/sql`包中，上下文用于停止或超时许多不同的操作。

# HTTP 请求

在官方包引入之前，每个与 HTTP 相关的框架都使用自己的版本上下文来存储与 HTTP 请求相关的数据。这导致了碎片化，并且在不重写中间件或任何特定绑定代码的情况下无法重用处理程序和中间件。

# 传递作用域值

在`http.Request`中引入`context.Context`试图通过定义一个可以分配、恢复和在各种处理程序中使用的单一接口来解决这个问题。

缺点是上下文不会自动分配给请求，并且上下文值不能被回收利用。没有真正好的理由这样做，因为上下文应该存储特定于某个包或范围的数据，而包本身应该是唯一能够与它们交互的对象。

一个很好的模式是使用一个独特的未导出的密钥类型，结合辅助函数来获取或设置特定的值：

```go
type keyType struct{}

var key = &keyType{}

func WithKey(ctx context.Context, value string) context.Context {
    return context.WithValue(ctx, key, value)
}

func GetKey(ctx context.Context) (string, bool) {
    v := ctx.Value(key)
    if v == nil {
        return "", false
    }
    return v.(string), true
}
```

上下文请求是标准库中唯一存储在数据结构中的情况，使用`WithContext`方法存储，并使用`Context`方法访问。这样做是为了不破坏现有代码，并保持 Go 1 的兼容性承诺。

完整示例在此处可用：[`play.golang.org/p/W6gGp_InoMp`](https://play.golang.org/p/W6gGp_InoMp)。

# 请求取消

上下文的一个很好的用法是在使用`http.Client`执行 HTTP 请求时进行取消和超时处理，它会自动处理上下文中的中断。以下示例正是如此：

```go
func main() {
    const addr = "localhost:8080"
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(time.Second * 5)
    })
    go func() {
        if err := http.ListenAndServe(addr, nil); err != nil {
            log.Fatalln(err)
        }
    }()
    req, _ := http.NewRequest(http.MethodGet, "http://"+addr, nil)
    ctx, canc := context.WithTimeout(context.Background(), time.Second*2)
    defer canc()
    time.Sleep(time.Second)
    if _, err := http.DefaultClient.Do(req.WithContext(ctx)); err != nil {
        log.Fatalln(err)
    }
}
```

上下文取消方法也可以用于中断传递给客户端的当前 HTTP 请求。在调用不同的端点并返回收到的第一个结果的情况下，取消其他请求是一个好主意。

让我们创建一个应用程序，它在不同的搜索引擎上运行查询，并返回最快的结果，取消其他搜索。我们可以创建一个 Web 服务器，它有一个唯一的端点，在 0 到 10 秒内回复：

```go
const addr = "localhost:8080"
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    d := time.Second * time.Duration(rand.Intn(10))
    log.Println("wait", d)
    time.Sleep(d)
})
go func() {
    if err := http.ListenAndServe(addr, nil); err != nil {
        log.Fatalln(err)
    }
}()
```

我们可以为请求使用可取消的上下文，结合等待组将其与请求结束同步。每个 goroutine 将创建一个请求，并尝试使用通道发送结果。由于我们只对第一个感兴趣，我们将使用`sync.Once`来限制它：

```go
ctx, canc := context.WithCancel(context.Background())
ch, o, wg := make(chan int), sync.Once{}, sync.WaitGroup{}
wg.Add(10)
for i := 0; i < 10; i++ {
    go func(i int) {
        defer wg.Done()
        req, _ := http.NewRequest(http.MethodGet, "http://"+addr, nil)
        if _, err := http.DefaultClient.Do(req.WithContext(ctx)); err != nil {
            log.Println(i, err)
            return
        }
        o.Do(func() { ch <- i })
    }(i)
}
log.Println("received", <-ch)
canc()
log.Println("cancelling")
wg.Wait()
```

当此程序运行时，我们将看到其中一个请求成功完成并发送到通道，而其他请求要么被取消，要么被忽略。

# HTTP 服务器

`net/http`包中有几种上下文的用法，包括停止监听器或成为请求的一部分。

# 关闭

`http.Server`允许我们为关闭操作传递上下文。这使我们能够使用一些上下文的功能，如取消和超时。我们可以定义一个新的服务器及其`mux`和可取消的上下文：

```go
mux := http.NewServeMux()
server := http.Server{
    Addr: ":3000",
    Handler: mux,
}
ctx, canc := context.WithCancel(context.Background())
defer canc()
mux.HandleFunc("/shutdown", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("OK"))
    canc()
})
```

我们可以在一个单独的 goroutine 中启动服务器：

```go
go func() {
    if err := server.ListenAndServe(); err != nil {
        if err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }
}()
```

当调用关闭端点并调用取消函数时，上下文将完成。我们可以等待该事件，然后使用具有超时的另一个上下文调用关闭方法：

```go
select {
case <-ctx.Done():
    ctx, canc := context.WithTimeout(context.Background(), time.Second*5)
    defer canc()
    if err := server.Shutdown(ctx); err != nil {
        log.Fatalln("Shutdown:", err)
    } else {
        log.Println("Shutdown:", "ok")
    }
}
```

这将允许我们在超时内有效地终止服务器，之后将以错误终止。

# 传递值

服务器中上下文的另一个用法是在不同的 HTTP 处理程序之间传播值和取消。让我们看一个例子，每个请求都有一个整数类型的唯一密钥。我们将使用一对类似于使用整数的值的函数。生成新密钥将使用`atomic`完成：

```go
type keyType struct{}

var key = &keyType{}

var counter int32

func WithKey(ctx context.Context) context.Context {
    return context.WithValue(ctx, key, atomic.AddInt32(&counter, 1))
}

func GetKey(ctx context.Context) (int32, bool) {
    v := ctx.Value(key)
    if v == nil {
        return 0, false
    }
    return v.(int32), true
}
```

现在，我们可以定义另一个函数，它接受任何 HTTP 处理程序，并在必要时创建上下文，并将密钥添加到其中：

```go

func AssignKeyHandler(h http.Handler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        if ctx == nil {
            ctx = context.Background()
        }
        if _, ok := GetKey(ctx); !ok {
            ctx = WithKey(ctx)
        }
        h.ServeHTTP(w, r.WithContext(ctx))
    }
}
```

通过这样做，我们可以定义一个非常简单的处理程序，用于在特定根目录下提供文件。此函数将使用上下文中的键正确记录信息。它还将在尝试提供文件之前检查文件是否存在：

```go
func ReadFileHandler(root string) http.HandlerFunc {
    root = filepath.Clean(root)
    return func(w http.ResponseWriter, r *http.Request) {
        k, _ := GetKey(r.Context())
        path := filepath.Join(root, r.URL.Path)
        log.Printf("[%d] requesting path %s", k, path)
        if !strings.HasPrefix(path, root) {
            http.Error(w, "not found", http.StatusNotFound)
            log.Printf("[%d] unauthorized %s", k, path)
            return
        }
        if stat, err := os.Stat(path); err != nil || stat.IsDir() {
            http.Error(w, "not found", http.StatusNotFound)
            log.Printf("[%d] not found %s", k, path)
            return
        }
        http.ServeFile(w, r, path)
        log.Printf("[%d] ok: %s", k, path)
    }
}
```

我们可以将这些处理程序组合起来，以便从不同文件夹（如主目录用户或临时目录）提供内容：

```go
home, err := os.UserHomeDir()
if err != nil {
    log.Fatal(err)
}
tmp := os.TempDir()
mux := http.NewServeMux()
server := http.Server{
    Addr: ":3000",
    Handler: mux,
}

mux.Handle("/tmp/", http.StripPrefix("/tmp/", AssignKeyHandler(ReadFileHandler(tmp))))
mux.Handle("/home/", http.StripPrefix("/home/", AssignKeyHandler(ReadFileHandler(home))))
if err := server.ListenAndServe(); err != nil {
    if err != http.ErrServerClosed {
        log.Fatal(err)
    }
}
```

我们使用`http.StipPrefix`来删除路径的第一部分并获取相对路径，并将其传递给下面的处理程序。生成的服务器将使用上下文在处理程序之间传递键值——这允许我们创建另一个类似的处理程序，并使用`AssignKeyHandler`函数来包装处理程序，并使用`GetKey(r.Context())`来访问处理程序内部的键。

# TCP 拨号

网络包提供了与上下文相关的功能，比如在拨号或监听传入连接时取消拨号。它允许我们在拨号连接时使用上下文的超时和取消功能。

# 取消连接

为了测试在 TCP 连接中使用上下文的用法，我们可以创建一个带有 TCP 服务器的 goroutine，在开始监听之前等待一段时间：

```go
addr := os.Args[1]
go func() {
    time.Sleep(time.Second)
    listener, err := net.Listen("tcp", addr)
    if err != nil {
        log.Fatalln("Listener:", addr, err)
    }
    c, err := listener.Accept()
    if err != nil {
        log.Fatalln("Listener:", addr, err)
    }
    defer c.Close()
}()
```

我们可以使用一个比服务器等待时间更短的超时上下文。我们必须使用`net.Dialer`来在拨号操作中使用上下文：

```go
ctx, canc := context.WithTimeout(context.Background(),   
    time.Millisecond*100)
defer canc()
conn, err := (&net.Dialer{}).DialContext(ctx, "tcp", os.Args[1])
if err != nil {
    log.Fatalln("-> Connection:", err)
}
log.Println("-> Connection to", os.Args[1])
conn.Close()
```

该应用程序将尝试连接一小段时间，但最终在上下文过期时放弃，返回一个错误。

在想要从一系列端点建立单个连接的情况下，上下文取消将是一个完美的用例。所有连接尝试将共享相同的上下文，并且正确拨号的第一个连接将调用取消，停止其他尝试。我们将创建一个单个服务器，它正在监听我们将尝试拨打的地址之一：

```go
list := []string{
    "localhost:9090",
    "localhost:9091",
    "localhost:9092",
}
go func() {
    listener, err := net.Listen("tcp", list[0])
    if err != nil {
        log.Fatalln("Listener:", list[0], err)
    }
    time.Sleep(time.Second * 5)
    c, err := listener.Accept()
    if err != nil {
        log.Fatalln("Listener:", list[0], err)
    }
    defer c.Close()
}()
```

然后，我们可以尝试拨打所有三个地址，并在其中一个连接时立即取消上下文。我们将使用`WaitGroup`与 goroutines 的结束进行同步：

```go
ctx, canc := context.WithTimeout(context.Background(), time.Second*10)
defer canc()
wg := sync.WaitGroup{}
wg.Add(len(list))
for _, addr := range list {
    go func(addr string) {
        defer wg.Done()
        conn, err := (&net.Dialer{}).DialContext(ctx, "tcp", addr)
        if err != nil {
            log.Println("-> Connection:", err)
            return
        }
        log.Println("-> Connection to", addr, "cancelling context")
        canc()
        conn.Close()
    }(addr)
}
wg.Wait()
```

在此程序的输出中，我们将看到一个连接成功，然后是其他尝试的取消错误。

# 数据库操作

在本书中我们不会讨论`sql/database`包，但为了完整起见，值得一提的是它也使用了上下文。它的大部分操作都有相应的上下文对应，例如：

+   开始一个新的事务

+   执行查询

+   对数据库进行 ping

+   准备查询

这就是标准库中使用上下文的包的内容。接下来，我们将尝试使用上下文构建一个包，以允许该包的用户取消请求。

# 实验性包

实验包中一个值得注意的例子使用了上下文，我们已经看过了——信号量。现在我们对上下文的用途有了更好的理解，很明显为什么获取操作也需要一个上下文作为参数。

在创建应用程序时，我们可以提供带有超时或取消的上下文，并相应地采取行动：

```go
func main() {
    s := semaphore.NewWeighted(int64(5))
    ctx, canc := context.WithTimeout(context.Background(), time.Second)
    defer canc()
    wg := sync.WaitGroup{}
    wg.Add(20)
    for i := 0; i < 20; i++ {
        go func(i int) {
            defer wg.Done()
            if err := s.Acquire(ctx, 1); err != nil {
                fmt.Println(i, err)
                return
            }
            go func(i int) {
                fmt.Println(i)
                time.Sleep(time.Second / 2)
                s.Release(1)
            }(i)
        }(i)
    }
    wg.Wait()
}
```

运行此应用程序将显示，信号量在第一秒被获取，但之后上下文过期，所有剩余操作都失败了。

# 应用程序中的上下文

如果包或应用程序具有可能需要很长时间并且用户可以取消的操作，或者应该具有超时或截止日期等时间限制，那么`context.Context`是集成到其中的完美工具。

# 要避免的事情

尽管 Go 团队已经非常清楚地定义了上下文的范围，但开发人员一直以各种方式使用它——有些方式不太正统。让我们看看其中一些以及有哪些替代方案，而不是求助于上下文。

# 错误的键类型

避免的第一个做法是使用内置类型作为键。这是有问题的，因为它们可以被覆盖，因为具有相同内置值的两个接口被认为是相同的，如下例所示：

```go
func main() {
    var a interface{} = "request-id"
    var b interface{} = "request-id"
    fmt.Println(a == b)

    ctx := context.Background()
    ctx = context.WithValue(ctx, a, "a")
    ctx = context.WithValue(ctx, b, "b")
    fmt.Println(ctx.Value(a), ctx.Value(b))
}
```

完整的示例在这里可用：[`play.golang.org/p/2W3noYQP5eh`](https://play.golang.org/p/2W3noYQP5eh)。

第一个打印指令输出`true`，由于键是按值比较的，第二个赋值遮蔽了第一个，导致两个键的值相同。解决这个问题的一个潜在方法是使用空结构自定义类型，或者使用内置值的未导出指针。

# 传递参数

可能会发生这样的情况，你需要通过一系列函数调用长途跋涉。一个非常诱人的解决方案是使用上下文来存储该值，并且只在需要它的函数中调用它。通常不是一个好主意隐藏应该显式传递的必需参数。这会导致代码不够可读，因为它不会清楚地表明什么影响了某个函数的执行。

将函数传递到堆栈下仍然要好得多。如果参数列表变得太长，那么它可以被分组到一个或多个结构中，以便更易读。

让我们来看看以下函数：

```go
func SomeFunc(ctx context.Context, 
    name, surname string, age int, 
    resourceID string, resourceName string) {}
```

参数可以按以下方式分组：

```go
type User struct {
    Name string
    Surname string
    Age int
}

type Resource struct {
    ID string
    Name string
}

func SomeFunc(ctx context.Context, u User, r Resource) {}
```

# 可选参数

上下文应该用于传递可选参数，并且还用作一种类似于 Python `kwargs` 或 JavaScript `arguments` 的万能工具。将上下文用作行为的替代品可能会导致非常严重的问题，因为它可能导致变量的遮蔽，就像我们在`context.WithValue`的示例中看到的那样。

这种方法的另一个重大缺点是隐藏发生的事情，使代码更加晦涩。当涉及可选值时，更好的方法是使用指向结构参数的指针 - 这允许您完全避免传递结构与`nil`。

假设你有以下代码：

```go
// This function has two mandatory args and 4 optional ones
func SomeFunc(ctx context.Context, arg1, arg2 int, 
    opt1, opt2, opt3, opt4 string) {}
```

通过使用`Optional`，你会得到这样的东西：

```go
type Optional struct {
    Opt1 string
    Opt2 string
    Opt3 string
    Opt4 string
}

// This function has two mandatory args and 4 optional ones
func SomeFunc(ctx context.Context, arg1, arg2 int, o *Optional) {}
```

# 全局变量

一些全局变量可以存储在上下文中，以便它们可以通过一系列函数调用传递。这通常不是一个好的做法，因为全局变量在应用程序的每个点都可用，因此使用上下文来存储和调用它们是毫无意义的，而且是资源和性能的浪费。如果您的包有一些全局变量，您可以使用我们在第十二章中看到的 Singleton 模式，*使用 sync 和 atomic 进行同步*，允许从包或应用程序的任何点访问它们。

# 使用上下文构建服务

我们现在将专注于如何创建支持上下文使用的包。这将帮助我们整合到目前为止学到的有关并发性的知识。我们将尝试创建一个并发文件搜索，使用通道、goroutine、同步和上下文。

# 主接口和用法

包的签名将包括上下文、根文件夹、搜索项和一对可选参数：

+   **在内容中搜索**：将在文件内容中查找字符串，而不是名称

+   **排除列表**：不会搜索具有所选名称/名称的文件

该函数看起来可能是这样的：

```go
type Options struct {
    Contents bool
    Exclude []string
}

func FileSearch(ctx context.Context, root, term string, o *Options)
```

由于它应该是一个并发函数，返回类型可以是结果的通道，它可以是错误，也可以是文件中一系列匹配项。由于我们可以搜索内容的名称，后者可能有多个匹配项：

```go
type Result struct {
    Err error
    File string
    Matches []Match
}

type Match struct {
    Line int
    Text string
}
```

前一个函数将返回一个只接收的`Result`类型的通道：

```go
func FileSearch(ctx context.Context, root, term string, o *Options) <-chan Result
```

在这里，这个函数将继续从通道接收值，直到它被关闭：

```go
for r := range FileSearch(ctx, directory, searchTerm, options) {
    if r.Err != nil {
        fmt.Printf("%s - error: %s\n", r.File, r.Err)
        continue
    }
    if !options.Contents {
        fmt.Printf("%s - match\n", r.File)
        continue
    }
    fmt.Printf("%s - matches:\n", r.File)
    for _, m := range r.Matches {
        fmt.Printf("\t%d:%s\n", m.Line, m.Text)
    }
}
```

# 出口和入口点

结果通道应该由上下文的取消或搜索结束来关闭。由于通道不能被关闭两次，我们可以使用`sync.Once`来避免第二次关闭通道。为了跟踪正在运行的 goroutines，我们可以使用`sync.Waitgroup`：

```go
ch, wg, once := make(chan Result), sync.WaitGroup{}, sync.Once{}
go func() {
    wg.Wait()
    fmt.Println("* Search done *")
    once.Do(func() {
        close(ch)
    })
}()
go func() {
    <-ctx.Done()
    fmt.Println("* Context done *")
    once.Do(func() {
        close(ch)
    })
}()
```

我们可以为每个文件启动一个 goroutine，这样我们可以定义一个私有函数，作为入口点，然后递归地用于子目录：

```go
func fileSearch(ctx context.Context, ch chan<- Result, wg *sync.WaitGroup, file, term string, o *Options)
```

主要导出的函数将首先向等待组添加一个值。然后，启动私有函数，将其作为异步进程启动：

```go
wg.Add(1)
go fileSearch(ctx, ch, &wg, root, term, o)
```

每个`fileSearch`应该做的最后一件事是调用`WaitGroup.Done`来标记当前文件的结束。

# 排除列表

私有函数将在完成使用`Done`方法之前减少等待组计数器。此外，它应该首先检查文件名，以便如果在排除列表中，可以跳过它：

```go
defer wg.Done()
_, name := filepath.Split(file)
if o != nil {
    for _, e := range o.Exclude {
        if e == name {
            return
        }
    }
}
```

如果不是这种情况，我们可以使用`os.Stat`来检查当前文件的信息，并且如果不成功，向通道发送错误。由于我们不能冒险通过向关闭的通道发送数据来引发恐慌，我们可以检查上下文是否完成，如果没有，发送错误：

```go
info, err := os.Stat(file)
if err != nil {
    select {
    case <-ctx.Done():
        return
    default:
        ch <- Result{File: file, Err: err}
    }
    return
}
```

# 处理目录

接收到的信息将告诉我们文件是否是目录。如果是目录，我们可以获取文件列表并处理错误，就像我们之前使用`os.Stat`一样。然后，如果上下文尚未完成，我们可以启动另一系列搜索，每个文件一个。以下代码总结了这些操作：

```go
if info.IsDir() {
    files, err := ioutil.ReadDir(file)
    if err != nil {
        select {
        case <-ctx.Done():
            return
        default:
            ch <- Result{File: file, Err: err}
        }
        return
    }
    select {
    case <-ctx.Done():
    default:
        wg.Add(len(files))
        for _, f := range files {
            go fileSearch(ctx, ch, wg, filepath.Join(file, 
        f.Name()), term, o)
        }
    }
    return
}
```

# 检查文件名和内容

如果文件是常规文件而不是目录，我们可以比较文件名或其内容，具体取决于指定的选项。检查文件名非常容易：

```go
if o == nil || !o.Contents {
    if name == term {
        select {
        case <-ctx.Done():
        default:
            ch <- Result{File: file}
        }
    }
    return
}
```

如果我们正在搜索内容，我们应该打开文件：

```go
f, err := os.Open(file)
if err != nil {
    select {
    case <-ctx.Done():
    default:
        ch <- Result{File: file, Err: err}
    }
    return
}
defer f.Close()
```

然后，我们可以逐行读取文件以搜索所选的术语。如果在读取文件时上下文过期，我们将停止所有操作：

```go
scanner, matches, line := bufio.NewScanner(f), []Match{}, 1
for scanner.Scan() {
    select {
    case <-ctx.Done():
        break
    default:
        if text := scanner.Text(); strings.Contains(text, term) {
            matches = append(matches, Match{Line: line, Text: text})
        }
        line++
    }
}
```

最后，我们可以检查扫描器的错误。如果没有错误并且搜索有结果，我们可以将所有匹配项发送到输出通道：

```go
select {
case <-ctx.Done():
    break
default:
    if err := scanner.Err(); err != nil {
        ch <- Result{File: file, Err: err}
        return
    }
    if len(matches) != 0 {
        ch <- Result{File: file, Matches: matches}
    }
}
```

不到 200 行的代码中，我们创建了一个并发文件搜索函数，每个文件使用一个 goroutine。它利用通道发送结果和同步原语来协调操作。

# 总结

在本章中，我们看到了一个较新的包上下文的用途。我们看到`Context`是一个简单的接口，有四种方法，并且应该作为函数的第一个参数使用。它的主要作用是处理取消和截止日期，以同步并发操作，并为用户提供取消操作的功能。

我们看到了默认上下文`Background`和`TODO`不允许取消，但它们可以使用包的各种函数进行扩展，以添加超时或取消。我们还谈到了上下文在持有值方面的能力，以及应该小心使用这一点，以避免遮蔽和其他问题。

然后，我们深入研究了标准包，看看上下文已经被使用在哪里。这包括了请求的 HTTP 功能，它可以用于值、取消和超时，以及服务器关闭操作。我们还看到了 TCP 包如何允许我们以类似的方式使用它，并且列出了数据库包中允许我们使用上下文来取消它们的操作。

在使用上下文构建自己的功能之前，我们先了解了一些应该避免的用法，从使用错误类型的键到使用上下文传递应该在函数或方法签名中的值。然后，我们继续创建一个函数，用于搜索文件和内容，利用了我们从前三章学到的并发知识。

下一章将通过展示最常见的 Go 并发模式及其用法来结束本书的并发部分。这将使我们能够将迄今为止学到的关于并发的所有知识放在一些非常常见和有效的配置中。

# 问题

1.  在 Go 中上下文是什么？

1.  取消、截止时间和超时之间有什么区别？

1.  在使用上下文传递值时，有哪些最佳实践？

1.  哪些标准包已经使用了上下文？
