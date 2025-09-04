# 并行和并发

本章将涵盖以下食谱：

+   使用通道和 select 语句

+   使用 sync.WaitGroup 执行异步操作

+   使用原子操作和互斥锁

+   使用上下文包

+   执行通道的状态管理

+   使用工作池设计模式

+   使用工作者创建管道

# 简介

本章将介绍工作池、异步操作等待组和 `context` 包的使用。并行和并发是 Go 语言最宣传和推广的特性之一。本章将提供一些有用的模式来帮助你入门，并帮助你理解这些特性。

Go 提供了使并行应用程序成为可能的原语。Goroutines 允许任何函数成为异步和并发的。Channels 允许应用程序与 goroutines 建立通信。Go 中的一条著名说法是 *不要通过共享内存来通信；相反，通过通信来共享内存*，出自 [`blog.golang.org/share-memory-by-communicating`](https://blog.golang.org/share-memory-by-communicating)。

# 使用通道和 select 语句

Go channels 与 goroutines 结合，是异步通信的一等公民。当使用 select 语句时，Channels 变得特别强大。这些语句允许 goroutine 同时智能地处理多个通道的请求。

# 准备工作

根据以下步骤配置你的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到你的操作系统上，并配置你的 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序。

1.  导航到 `GOPATH/src` 并创建一个项目目录，例如 `$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

# 如何做到...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建 `chapter9/channels` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter9/channels`](https://github.com/agtorre/go-cookbook/tree/master/chapter9/channels) 复制测试用例，或者使用这个练习来编写你自己的代码。

1.  创建一个名为 `sender.go` 的文件，内容如下：

```go
        package channels

        import "time"

        // Sender sends "tick"" on ch until done is
        // written to, then it sends "sender done."
        // and exits
        func Sender(ch chan string, done chan bool) {
            t := time.Tick(100 * time.Millisecond)
            for {
                select {
                    case <-done:
                        ch <- "sender done."
                        return
                    case <-t:
                        ch <- "tick"
                }
            }
        }

```

1.  创建一个名为 `printer.go` 的文件，内容如下：

```go
        package channels

        import (
            "context"
            "fmt"
            "time"
        )

        // Printer will print anything sent on the ch chan
        // and will print tock every 200 milliseconds
        // this will repeat forever until a context is
        // Done, i.e. timed out or cancelled
        func Printer(ctx context.Context, ch chan string) {
            t := time.Tick(200 * time.Millisecond)
            for {
                select {
                  case <-ctx.Done():
                      fmt.Println("printer done.")
                      return
                  case res := <-ch:
                      fmt.Println(res)
                  case <-t:
                      fmt.Println("tock")
                }
            }
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，内容如下，并确保将 `channels` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
            "context"
            "time"

            "github.com/agtorre/go-cookbook/chapter9/channels"
        )

        func main() {
            ch := make(chan string)
            done := make(chan bool)

            ctx := context.Background()
            ctx, cancel := context.WithCancel(ctx)
            defer cancel()

            go channels.Printer(ctx, ch)
            go channels.Sender(ch, done)

            time.Sleep(2 * time.Second)
            done <- true
            cancel()
            //sleep a bit extra so channels can clean up
            time.Sleep(1 * time.Second)
        }

```

1.  运行 `go run main.go`。

1.  你还可以运行以下命令：

```go
 go build ./example

```

现在，你应该看到以下输出：

```go
 $ go run main.go
 tick
 tock
 tick
 tick
 tock
 tick
 tick
 tock
 tick
 .
 .
 .
 sender done.
 printer done.

```

1.  如果你复制或编写了自己的测试用例，请向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方演示了两种启动工作进程的方法，该进程可以读取或写入通道，也可能两者都做。终止条件是一个`done`通道，或者使用`context`包。*使用上下文包*配方将更详细地介绍上下文。

使用`main`包将独立的函数连接起来；多亏了这一点，只要通道不共享，就可以设置多个对。此外，还可以有多个 goroutine 监听同一个通道，我们将在*使用工作池设计模式*的配方中探讨这一点。

最后，由于 goroutines 的异步性质，建立清理和终止条件可能很棘手；例如，一个常见的错误是以下操作：

```go
select{
    case <-time.Tick(200 * time.Millisecond):
    //this resets whenever any other 'lane' is chosen
}

```

通过在`select`语句中放置勾号，可以防止这种情况发生。在`select`语句中也没有简单的方法来优先处理流量。

# 使用 sync.WaitGroup 执行异步操作

有时，执行多个异步操作然后等待它们完成再继续是有用的。例如，如果操作需要从多个 API 中提取信息并汇总这些信息，那么异步地发出客户端请求可能会有所帮助。本章将探讨使用`sync.WaitGroup`来并行编排非依赖性任务。

# 准备工作

参考本章中*使用通道和选择语句*配方中的*准备工作*部分。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中创建

    `chapter9/waitgroup`目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter9/waitgroup`](https://github.com/agtorre/go-cookbook/tree/master/chapter9/waitgroup)复制测试或将其作为练习来编写一些自己的代码。

1.  创建一个名为`tasks.go`的文件，内容如下：

```go
        package waitgroup

        import (
            "fmt"
            "log"
            "net/http"
            "strings"
            "time"
        )

        // GetURL gets a url, and logs the time it took
        func GetURL(url string) (*http.Response, error) {
            start := time.Now()
            log.Printf("getting %s", url)
            resp, err := http.Get(url)
            log.Printf("completed getting %s in %s", url, 
            time.Since(start))
            return resp, err
        }

        // CrawlError is our custom error type
        // for aggregating errors
        type CrawlError struct {
            Errors []string
        }

        // Add adds another error
        func (c *CrawlError) Add(err error) {
            c.Errors = append(c.Errors, err.Error())
        }

        // Error implements the error interface
        func (c *CrawlError) Error() string {
            return fmt.Sprintf("All Errors: %s", strings.Join(c.Errors, 
            ","))
        }

        // Valid can be used to determine if
        // we should return this
        func (c *CrawlError) Valid() bool {
            return len(c.Errors) != 0
        }

```

1.  创建一个名为`process.go`的文件，内容如下：

```go
        package waitgroup

        import (
            "log"
            "sync"
            "time"
        )

        // Crawl collects responses from a list of urls
        // that are passed in. It waits for all requests
        // to complete before returning.
        func Crawl(sites []string) ([]int, error) {
            start := time.Now()
            log.Printf("starting crawling")
            wg := &sync.WaitGroup{}

            var resps []int
            cerr := &CrawlError{}
            for _, v := range sites {
                wg.Add(1)
                go func(v string) {
                    defer wg.Done()
                    resp, err := GetURL(v)
                    if err != nil {
                        cerr.Add(err)
                        return
                    }
                    resps = append(resps, resp.StatusCode)
                }(v)
            }
            wg.Wait()
            if cerr.Valid() {
                return resps, cerr
            }
            log.Printf("completed crawling in %s", time.Since(start))
            return resps, nil
        }

```

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个名为`main.go`的文件，内容如下。确保你修改`waitgroup`导入以使用你在步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter9/waitgroup"
        )

        func main() {
            sites := []string{
                "https://golang.org",
                "https://godoc.org",
                "https://www.google.com/search?q=golang",
            }

            resps, err := waitgroup.Crawl(sites)
            if err != nil {
                panic(err)
            }
            fmt.Println("Resps received:", resps)
        }

```

1.  运行`go run main.go`。

1.  你还可以运行以下命令：

```go
 go build ./example

```

你应该看到以下内容：

```go
 $ go run main.go
 2017/04/05 19:45:07 starting crawling
 2017/04/05 19:45:07 getting https://www.google.com/search?
      q=golang
 2017/04/05 19:45:07 getting https://golang.org
 2017/04/05 19:45:07 getting https://godoc.org
 2017/04/05 19:45:07 completed getting https://golang.org in 
      178.22407ms
 2017/04/05 19:45:07 completed getting https://godoc.org in 
      181.400873ms
 2017/04/05 19:45:07 completed getting 
      https://www.google.com/search?q=golang in 238.019327ms
 2017/04/05 19:45:07 completed crawling in 238.191791ms
 Resps received: [200 200 200]

```

1.  如果你复制了自己的测试或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方展示了如何在等待工作时使用`waitgroups`作为同步机制。本质上，`waitgroup.Wait()`将等待其内部计数器达到`0`。`waitgroup.Add(int)`方法将计数器增加输入的数量，而`waitgroup.Done()`将计数器减`1`。因此，在各个 goroutine 将`waitgroup`标记为`Done()`的同时，需要异步地`Wait()`。

在这个示例中，我们在发送每个 HTTP 请求之前增加计数，然后调用一个 defer `wg.Done()`方法，这样我们就可以在 goroutine 终止时减少计数。然后我们等待所有 goroutine 完成，然后再返回我们的聚合结果。

在实践中，最好使用通道来传递错误和响应。

在执行此类异步操作时，你应该考虑修改共享映射等线程安全。如果你记住这一点，`waitgroups`是等待任何类型异步操作的有用功能。

# 使用原子操作和互斥锁

在像 Go 这样的语言中，你拥有内置的异步操作和并行性，考虑线程安全变得很重要。例如，同时从多个 goroutine 访问映射是危险的。Go 在`sync`和`sync/atomic`包中提供了一些辅助工具，以确保某些事件只发生一次，或者 goroutine 可以在操作上序列化。

这个示例将展示如何使用这些包安全地修改映射，以及如何保持一个可以被多个 goroutine 安全访问的全局序数值。它还将展示`Once.Do`方法，该方法可以用来确保 Go 应用程序只执行一次某些操作，例如读取配置或初始化变量。

# 准备就绪

参考本章中“使用通道和 select 语句”食谱的“准备就绪”部分。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建`chapter9/atomic`目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter9/atomic`](https://github.com/agtorre/go-cookbook/tree/master/chapter9/atomic)复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为`map.go`的文件，内容如下：

```go
        package atomic

        import (
            "errors"
            "sync"
        )

        // SafeMap uses a mutex to allow
        // getting and setting in a thread-safe way
        type SafeMap struct {
            m map[string]string
            mu *sync.RWMutex
        }

        // NewSafeMap creates a SafeMap
        func NewSafeMap() SafeMap {
            return SafeMap{m: make(map[string]string), mu: 
            &sync.RWMutex{}}
        }

        // Set uses a write lock and sets the value given
        // a key
        func (t *SafeMap) Set(key, value string) {
            t.mu.Lock()
            defer t.mu.Unlock()

            t.m[key] = value
        }

        // Get uses a RW lock and gets the value if it exists,
        // otherwise an error is returned
        func (t *SafeMap) Get(key string) (string, error) {
            t.mu.RLock()
            defer t.mu.RUnlock()

            if v, ok := t.m[key]; ok {
                return v, nil
            }

            return "", errors.New("key not found")
        }

```

1.  创建一个名为`ordinal.go`的文件，内容如下：

```go
        package atomic

        import (
            "sync"
            "sync/atomic"
        )

        // Ordinal holds a global a value
        // and can only be initialized once
        type Ordinal struct {
            ordinal uint64
            once *sync.Once
        }

        // NewOrdinal returns ordinal with once
        // setup
        func NewOrdinal() *Ordinal {
            return &Ordinal{once: &sync.Once{}}
        }

        // Init sets the ordinal value
        // can only be done once
        func (o *Ordinal) Init(val uint64) {
            o.once.Do(func() {
                atomic.StoreUint64(&o.ordinal, val)
            })
        }

        // GetOrdinal will return the current
        // ordinal
        func (o *Ordinal) GetOrdinal() uint64 {
            return atomic.LoadUint64(&o.ordinal)
        }

        // Increment will increment the current
        // ordinal
        func (o *Ordinal) Increment() {
            atomic.AddUint64(&o.ordinal, 1)
        }

```

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个名为`main.go`的文件，内容如下，并确保你修改`atomic`导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"
            "sync"

            "github.com/agtorre/go-cookbook/chapter9/atomic"
        )

        func main() {
            o := atomic.NewOrdinal()
            m := atomic.NewSafeMap()
            o.Init(1123)
            fmt.Println("initial ordinal is:", o.GetOrdinal())
            wg := sync.WaitGroup{}
            for i := 0; i < 10; i++ {
                wg.Add(1)
                go func(i int) {
                    defer wg.Done()
                    m.Set(fmt.Sprint(i), "success")
                    o.Increment()
                }(i)
            }

            wg.Wait()
            for i := 0; i < 10; i++ {
                v, err := m.Get(fmt.Sprint(i))
                if err != nil || v != "success" {
                    panic(err)
                }
            }
            fmt.Println("final ordinal is:", o.GetOrdinal())
            fmt.Println("all keys found and marked as: 'success'")
        }

```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下内容：

```go
 $ go run main.go

 initial ordinal is: 1123
 final ordinal is: 1133
 all keys found and marked as: 'success'

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

对于我们的映射食谱，我们使用了`ReadWrite`互斥锁。这种互斥锁背后的想法是，任何数量的读者都可以获取读锁，但只有一个写者可以获取写锁。此外，当其他人（读者或写者）拥有锁时，写者无法获取锁。这很有用，因为与标准互斥锁相比，读取操作非常快且非阻塞。每次我们想要设置数据时，我们使用`Lock()`对象，每次我们想要读取数据时，我们使用`RLock()`。最终使用`Unlock()`或`RUnlock()`是至关重要的，这样您就不会使应用程序发生死锁。defer `Unlock()`对象可能很有用，但可能比手动调用`Unlock()`慢。 

当您想要将额外的操作与锁定值分组时，这种模式可能不够灵活。例如，在某些情况下，您可能想要锁定，执行一些额外的处理，然后才解锁。在设计时考虑这一点非常重要。

`sync/atomic`包被`Ordinal`用于获取和设置值。还有原子比较操作，如`atomic.CompareAndSwapUInt64()`，它们非常有价值。本食谱允许在`Ordinal`对象上仅调用一次 Init；否则，它只能原子性地增加。

我们循环创建 10 个 goroutine（与`sync.Waitgroup`同步）并显示序号正确增加了 10 次，以及我们映射中的每个键都适当地设置了。

# 使用上下文包

本书中的几个食谱都使用了`context`包。本食谱将探讨创建和管理上下文的基本知识。了解上下文的良好参考资料是[`blog.golang.org/context`](https://blog.golang.org/context)。自从这篇博客文章被撰写以来，上下文已从`net/context`移动到一个名为`context`的包。这仍然偶尔会在与 GRPC 等第三方库交互时引起问题。

本食谱将探讨为上下文设置和获取值、取消和超时。

# 准备就绪

参考本章中*使用通道和 select 语句*食谱的*准备就绪*部分。

# 如何操作...

这些步骤涵盖了编写和运行您的应用程序：

1.  在您的终端/控制台应用程序中，创建`chapter9/context`目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter9/context`](https://github.com/agtorre/go-cookbook/tree/master/chapter9/context)复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为`values.go`的文件，内容如下：

```go
        package context

        import "context"

        type key string

        const (
            timeoutKey key = "TimeoutKey"
            deadlineKey key = "DeadlineKey"
        )

        // Setup sets some values
        func Setup(ctx context.Context) context.Context {

            ctx = context.WithValue(ctx, timeoutKey,
            "timeout exceeded")
            ctx = context.WithValue(ctx, deadlineKey,
            "deadline exceeded")

            return ctx
        }

        // GetValue grabs a value given a key and
        // returns a string representation of the
        // value
        func GetValue(ctx context.Context, k key) string {

            if val, ok := ctx.Value(k).(string); ok {
                return val
            }
            return ""

        }

```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package context

        import (
            "context"
            "fmt"
            "math/rand"
            "time"
        )

        // Exec sets two random timers and prints
        // a different context value for whichever
        // fires first
        func Exec() {
            // a base context
            ctx := context.Background()
            ctx = Setup(ctx)

            rand.Seed(time.Now().UnixNano())

            timeoutCtx, cancel := context.WithTimeout(ctx, 
            (time.Duration(rand.Intn(2)) * time.Millisecond))
            defer cancel()

            deadlineCtx, cancel := context.WithDeadline(ctx, 
            time.Now().Add(time.Duration(rand.Intn(2))
            *time.Millisecond))
            defer cancel()

            for {
                select {
                    case <-timeoutCtx.Done():
                    fmt.Println(GetValue(ctx, timeoutKey))
                    return
                    case <-deadlineCtx.Done():
                        fmt.Println(GetValue(ctx, deadlineKey))
                        return
                }
            }
        }

```

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个名为`main.go`的文件，内容如下。确保您修改`context`导入以使用步骤 2 中设置的路径：

```go
        package main

            import "github.com/agtorre/go-cookbook/chapter9/context"

        func main() {
            context.Exec()
        }

```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
 go build ./example

```

您现在应该看到以下输出：

```go
 $ go run main.go
 timeout exceeded

 OR

 $ go run main.go
 deadline exceeded

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

当处理上下文值时，创建一个新的类型来表示键是个好主意。在这种情况下，我们创建了一个 `key` 类型，然后声明了一些相应的 `const` 值来表示我们所有的可能键。

在这个例子中，我们使用 `Setup()` 函数同时初始化所有的键值对。当修改上下文时，函数通常需要一个 `context` 参数并返回一个 `context` 值。所以签名通常看起来像这样：

```go
func ModifyContext(ctx context.Context) context.Context

```

有时，这些方法也会返回一个错误或 `cancel()` 函数，例如在 `context.WithCancel`、`context.WithTimeout` 和 `context.WithDeadline` 的情况下。所有子上下文都继承父上下文的属性。

在这个菜谱中，我们创建了两个子上下文，一个带有截止日期，一个带有超时。我们将这些设置为随机范围的超时，然后在接收到任何一个时终止。最后，我们根据给定的键提取一个值并打印它。

# 执行通道的状态管理

在 Go 中，通道可以是任何类型。结构体的通道允许你通过单个消息传递大量的状态。这个菜谱将探讨使用通道传递复杂的请求结构体并在复杂的响应结构体中返回它们的结果。

在下一道菜谱中，*使用工作池设计模式*，这个值变得更加明显，因为你可以创建能够执行各种任务的一般用途的工作者。

# 准备就绪

参考本章中 *准备就绪* 部分的 *使用通道和选择语句* 菜谱。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter9/state` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter9/state`](https://github.com/agtorre/go-cookbook/tree/master/chapter9/state) 复制测试或[将其用作练习来编写你自己的代码。](https://github.com/agtorre/go-cookbook/tree/master/chapter9/state)

1.  创建一个名为 `state.go` 的文件，内容如下：

```go
        package state

        type op string

        const (
            // Add values
            Add op = "add"
            // Subtract values
            Subtract = "sub"
            // Multiply values
            Multiply = "mult"
            // Divide values
            Divide = "div"
        )

        // WorkRequest perform an op
        // on two values
        type WorkRequest struct {
            Operation op
            Value1 int64
            Value2 int64
        }

        // WorkResponse returns the result
        // and any errors
        type WorkResponse struct {
            Wr *WorkRequest
            Result int64
            Err error
        }

```

1.  创建一个名为 `processor.go` 的文件，内容如下：

```go
        package state

        import "context"

        // Processor routes work to Process
        func Processor(ctx context.Context, in chan *WorkRequest, out 
        chan *WorkResponse) {
            for {
                select {
                    case <-ctx.Done():
                        return
                    case wr := <-in:
                        out <- Process(wr)
                }
            }
        }

```

1.  创建一个名为 `process.go` 的文件，内容如下：

```go
        package state

        import "errors"

        // Process switches on operation type
        // Then does work
        func Process(wr *WorkRequest) *WorkResponse {
            resp := WorkResponse{Wr: wr}

            switch wr.Operation {
                case Add:
                    resp.Result = wr.Value1 + wr.Value2
                case Subtract:
                    resp.Result = wr.Value1 - wr.Value2
                case Multiply:
                    resp.Result = wr.Value1 * wr.Value2
                case Divide:
                    if wr.Value2 == 0 {
                        resp.Err = errors.New("divide by 0")
                        break
                    }
                    resp.Result = wr.Value1 / wr.Value2
                    default:
                        resp.Err = errors.New("unsupported operation")
            }
            return &resp
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，内容如下。确保将 `state` 导入修改为你在步骤 2 中设置的路径：

```go
        package main

        import (
            "context"
            "fmt"

            "github.com/agtorre/go-cookbook/chapter9/state"
        )

        func main() {
            in := make(chan *state.WorkRequest, 10)
            out := make(chan *state.WorkResponse, 10)
            ctx := context.Background()
            ctx, cancel := context.WithCancel(ctx)
            defer cancel()

            go state.Processor(ctx, in, out)

            req := state.WorkRequest{state.Add, 3, 4}
            in <- &req

            req2 := state.WorkRequest{state.Subtract, 5, 2}
            in <- &req2

            req3 := state.WorkRequest{state.Multiply, 9, 9}
            in <- &req3

            req4 := state.WorkRequest{state.Divide, 8, 2}
            in <- &req4

            req5 := state.WorkRequest{state.Divide, 8, 0}
            in <- &req5

            for i := 0; i < 5; i++ {
                resp := <-out
                fmt.Printf("Request: %v; Result: %v, Error: %vn",
                resp.Wr, resp.Result, resp.Err)
            }
        }

```

1.  运行 `go run main.go`.

1.  你也可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下输出：

```go
 $ go run main.go
 Request: &{add 3 4}; Result: 7, Error: <nil>
 Request: &{sub 5 2}; Result: 3, Error: <nil>
 Request: &{mult 9 9}; Result: 81, Error: <nil>
 Request: &{div 8 2}; Result: 4, Error: <nil>
 Request: &{div 8 0}; Result: 0, Error: divide by 0

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

在这个菜谱中的 `Processor()` 函数是一个无限循环直到其上下文被取消的函数，无论是通过显式调用取消还是通过超时。它将所有工作调度到 `Process()`，它可以处理各种操作给出的不同函数。也有可能让这些情况中的每一个调度另一个函数以实现更模块化的代码。

最终，响应被返回到响应通道，我们在最后循环并打印所有结果。我们还演示了除以 `0` 的错误情况。

# 使用工作池设计模式

工作池设计模式是一种将长时间运行的 goroutines 作为工作者调度的模式。这些工作者可以通过多个通道或使用具有指定类型的具有状态请求结构来处理各种工作，正如前一个菜谱中描述的那样。

这个菜谱将创建具有状态的工作者，并演示如何协调和启动多个工作者，它们在同一个通道上并发处理请求。这些工作者将像在 Web 身份验证应用程序中的加密工作者一样。他们的目的是使用 `bcrypt` 包对纯文本字符串进行哈希处理，并将文本密码与哈希进行比较。

# 准备工作

参考本章中 *Using channels and the select statement* 菜谱的 *Getting ready* 部分。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter9/pool` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter9/pool`](https://github.com/agtorre/go-cookbook/tree/master/chapter9/pool) 复制测试或使用这个练习来编写你自己的代码。

1.  创建一个名为 `worker.go` 的文件，并包含以下内容：

```go
        package pool

        import (
            "context"
            "fmt"
        )

        // Dispatch creates numWorker workers, returns a cancel 
        // function channels for adding work and responses, 
        // cancel must be called
        func Dispatch(numWorker int) (context.CancelFunc, chan 
        WorkRequest, chan WorkResponse) {
            ctx := context.Background()
            ctx, cancel := context.WithCancel(ctx)
            in := make(chan WorkRequest, 10)
            out := make(chan WorkResponse, 10)

            for i := 0; i < numWorker; i++ {
                go Worker(ctx, i, in, out)
            }
            return cancel, in, out
        }

        // Worker loops forever and is part of the worker pool
        func Worker(ctx context.Context, id int, in chan WorkRequest, 
        out chan WorkResponse) {
            for {
                select {
                    case <-ctx.Done():
                        return
                    case wr := <-in:
                        fmt.Printf("worker id: %d, performing %s
                        workn", id, wr.Op)
                        out <- Process(wr)
                }
            }
        }

```

1.  创建一个名为 `work.go` 的文件，并包含以下内容：

```go
        package pool

        import "errors"

        type op string

        const (
            // Hash is the bcrypt work type
            Hash op = "encrypt"
            // Compare is bcrypt compare work
            Compare = "decrypt"
        )

        // WorkRequest is a worker req
        type WorkRequest struct {
            Op op
            Text []byte
            Compare []byte // optional
        }

        // WorkResponse is a worker resp
        type WorkResponse struct {
            Wr WorkRequest
            Result []byte
            Matched bool
            Err error
        }

        // Process dispatches work to the worker pool channel
        func Process(wr WorkRequest) WorkResponse {
            switch wr.Op {
            case Hash:
                return hashWork(wr)
            case Compare:
                return compareWork(wr)
            default:
                return WorkResponse{Err: errors.New("unsupported 
                operation")}
            }
        }

```

1.  创建一个名为 `crypto.go` 的文件，并包含以下内容：

```go
        package pool

        import "golang.org/x/crypto/bcrypt"

        func hashWork(wr WorkRequest) WorkResponse {
            val, err := bcrypt.GenerateFromPassword(wr.Text, 
            bcrypt.DefaultCost)
            return WorkResponse{
                Result: val,
                Err: err,
                Wr: wr,
            }
        }

        func compareWork(wr WorkRequest) WorkResponse {
            var matched bool
            err := bcrypt.CompareHashAndPassword(wr.Compare, wr.Text)
            if err == nil {
                matched = true
            }
            return WorkResponse{
                Matched: matched,
                Err: err,
                Wr: wr,
            }
        }

```

1.  创建一个名为 `example` 的新目录，并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容。确保你修改 `state` 导入以使用你在第 2 步中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter9/pool"
        )

        func main() {
            cancel, in, out := pool.Dispatch(10)
            defer cancel()

            for i := 0; i < 10; i++ {
                in <- pool.WorkRequest{Op: pool.Hash, Text: 
                []byte(fmt.Sprintf("messages %d", i))}
            }

            for i := 0; i < 10; i++ {
                res := <-out
                if res.Err != nil {
                    panic(res.Err)
                }
                in <- pool.WorkRequest{Op: pool.Compare, Text: 
                res.Wr.Text, Compare: res.Result}
            }

            for i := 0; i < 10; i++ {
                res := <-out
                if res.Err != nil {
                    panic(res.Err)
                }
                fmt.Printf("string: "%s"; matched: %vn", 
                string(res.Wr.Text), res.Matched)
            }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下内容：

```go
 $ go run main.go
 worker id: 9, performing encrypt work
 worker id: 5, performing encrypt work
 worker id: 2, performing encrypt work
 worker id: 8, performing encrypt work
 worker id: 6, performing encrypt work
 worker id: 1, performing encrypt work
 worker id: 0, performing encrypt work
 worker id: 4, performing encrypt work
 worker id: 3, performing encrypt work
 worker id: 7, performing encrypt work
 worker id: 2, performing decrypt work
 worker id: 6, performing decrypt work
 worker id: 8, performing decrypt work
 worker id: 1, performing decrypt work
 worker id: 0, performing decrypt work
 worker id: 9, performing decrypt work
 worker id: 3, performing decrypt work
 worker id: 4, performing decrypt work
 worker id: 7, performing decrypt work
 worker id: 5, performing decrypt work
 string: "messages 9"; matched: true
 string: "messages 3"; matched: true
 string: "messages 4"; matched: true
 string: "messages 0"; matched: true
 string: "messages 1"; matched: true
 string: "messages 8"; matched: true
 string: "messages 5"; matched: true
 string: "messages 7"; matched: true
 string: "messages 2"; matched: true
 string: "messages 6"; matched: true

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 工作原理...

这个菜谱使用 `Dispatch()` 方法在单个输入通道、输出通道以及连接到单个 `cancel()` 函数的通道上创建多个工作者。如果你想要为不同的目的创建不同的池，这可以用来。例如，你可以通过使用单独的池来创建 10 个加密和 20 个比较工作者。对于这个菜谱，我们使用一个单独的池，向工作者发送哈希请求，检索响应，然后向同一个池发送比较请求。因此，执行工作的工作者每次都会不同，但它们都能执行这两种类型的工作。

这种方法的优点是，它既允许并行处理，也可以控制最大并发量。限制 goroutine 的最大数量对于限制内存也很重要。我选择加密作为这个菜谱的例子，因为如果为每个新请求启动一个新的 goroutine，例如在 Web 服务中，加密代码可能会耗尽你的 CPU 或内存。

# 使用工作者创建管道

这个菜谱演示了创建工作池组并将它们连接起来形成管道。对于这个菜谱，我们连接了两个池，但这个模式可以用于更复杂的操作，类似于中间件。

工作池可以用来保持工作者相对简单，并进一步控制并发。例如，在并行其他操作的同时序列化日志可能很有用。这也可以用于为更昂贵的操作创建更小的池，这样就不会超载机器资源。

# 准备就绪

参考本章中 *准备就绪* 部分 *使用通道和 select 语句* 菜谱。

# 如何做到...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建名为 `chapter9/pipeline` 的目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter9/pipeline`](https://github.com/agtorre/go-cookbook/tree/master/chapter9/pipeline) 复制测试或将其作为练习编写你自己的代码。

1.  创建一个名为 `worker.go` 的文件，并包含以下内容：

```go
        package pipeline

        import "context"

        // Worker have one role
        // that is determined when
        // Work is called
        type Worker struct {
            in chan string
            out chan string
        }

        // Job is a job a worker can do
        type Job string

        const (
            // Print echo's all input to
            // stdout
            Print Job = "print"
            // Encode base64 encodes input
            Encode Job = "encode"
        )

        // Work is how to dispatch a worker, they are assigned
        // a job here
        func (w *Worker) Work(ctx context.Context, j Job) {
            switch j {
                case Print:
                    w.Print(ctx)
                case Encode:
                    w.Encode(ctx)
                default:
                    return
            }
        }

```

1.  创建一个名为 `print.go` 的文件，并包含以下内容：

```go
        package pipeline

        import (
            "context"
            "fmt"
        )

        // Print prints w.in and repalys it
        // on w.out
        func (w *Worker) Print(ctx context.Context) {
            for {
                select {
                    case <-ctx.Done():
                        return
                    case val := <-w.in:
                        fmt.Println(val)
                        w.out <- val
                }
            }
        }

```

1.  创建一个名为 `encode.go` 的文件，并包含以下内容：

```go
        package pipeline

        import (
            "context"
            "encoding/base64"
            "fmt"
        )

        // Encode takes plain text as int
        // and returns "string => <base64 string encoding>
        // as out
        func (w *Worker) Encode(ctx context.Context) {
            for {
                select {
                    case <-ctx.Done():
                        return
                    case val := <-w.in:
                        w.out <- fmt.Sprintf("%s => %s", val, 
                        base64.StdEncoding.EncodeToString([]byte(val)))
                }
            }
        }

```

1.  创建一个名为 `pipeline.go` 的文件，并包含以下内容：

```go
        package pipeline

        import "context"

        // NewPipeline initializes the workers and
        // connects them, it returns the input of the pipeline
        // and the final output
        func NewPipeline(ctx context.Context, numEncoders, numPrinters 
        int) (chan string, chan string) {
            inEncode := make(chan string, numEncoders)
            inPrint := make(chan string, numPrinters)
            outPrint := make(chan string, numPrinters)
            for i := 0; i < numEncoders; i++ {
                w := Worker{
                    in: inEncode,
                    out: inPrint,
                }
                go w.Work(ctx, Encode)
            }

            for i := 0; i < numPrinters; i++ {
                w := Worker{
                    in: inPrint,
                   out: outPrint,
                }
                go w.Work(ctx, Print)
            }
            return inEncode, outPrint
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容，并确保将 `state` 导入修改为你在步骤 2 中设置的路径：

```go
        package main

        import (
            "context"
            "fmt"

            "github.com/agtorre/go-cookbook/chapter9/pipeline"
        )

        func main() {
            ctx := context.Background()
            ctx, cancel := context.WithCancel(ctx)
            defer cancel()

            in, out := pipeline.NewPipeline(ctx, 10, 2)

            go func() {
                for i := 0; i < 20; i++ {
                    in <- fmt.Sprint("Message", i)
                }
            }()

            for i := 0; i < 20; i++ {
                <-out
            }
        }

```

1.  运行 `go run main.go`。

1.  你还可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下内容：

```go
 $ go run main.go
 Message3 => TWVzc2FnZTM=
 Message7 => TWVzc2FnZTc=
 Message8 => TWVzc2FnZTg=
 Message9 => TWVzc2FnZTk=
 Message5 => TWVzc2FnZTU=
 Message11 => TWVzc2FnZTEx
 Message10 => TWVzc2FnZTEw
 Message4 => TWVzc2FnZTQ=
 Message12 => TWVzc2FnZTEy
 Message6 => TWVzc2FnZTY=
 Message14 => TWVzc2FnZTE0
 Message13 => TWVzc2FnZTEz
 Message0 => TWVzc2FnZTA=
 Message15 => TWVzc2FnZTE1
 Message1 => TWVzc2FnZTE=
 Message17 => TWVzc2FnZTE3
 Message16 => TWVzc2FnZTE2
 Message19 => TWVzc2FnZTE5
 Message18 => TWVzc2FnZTE4
 Message2 => TWVzc2FnZTI=

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`main` 包创建了一个由 10 个编码器和两个打印机组成的管道。它在输入通道上排队 20 个字符串，并在输出通道上等待 20 个响应。如果消息到达输出通道，则表示它们已成功通过整个管道。

`NewPipeline` 函数用于连接池。它确保通道以适当的缓冲大小创建，并且某些池的输出通道连接到其他池的适当输入通道。还可以通过在每个工作器上使用输入通道和输出通道的数组、多个命名通道或通道映射来扩展管道。这将允许发送消息到每个步骤的记录器等操作。
