# 第十章：并行和并发

本章中的示例涵盖了工作池、异步操作的等待组以及`context`包的使用。并行和并发是 Go 语言最广告和推广的特性之一。本章将提供一些有用的模式，帮助您入门并了解这些特性。

Go 提供了使并行应用程序成为可能的原语。Goroutines 允许任何函数变成异步和并发的。通道允许应用程序与 Goroutines 建立通信。Go 语言中有一句著名的话是：“*不要通过共享内存进行通信；相反，通过通信共享内存*”，出自[`blog.golang.org/share-memory-by-communicating`](https://blog.golang.org/share-memory-by-communicating)。

在本章中，我们将涵盖以下示例：

+   使用通道和 select 语句

+   使用 sync.WaitGroup 执行异步操作

+   使用原子操作和互斥锁

+   使用上下文包

+   执行通道的状态管理

+   使用工作池设计模式

+   使用工作进程创建管道

# 技术要求

为了继续本章中的所有示例，请按照以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.12.6 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  打开终端或控制台应用程序，并创建并转到一个项目目录，例如`~/projects/go-programming-cookbook`。所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，（可选）从该目录中工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 使用通道和 select 语句

Go 通道与 Goroutines 结合使用，是异步通信的一等公民。当我们使用 select 语句时，通道变得特别强大。这些语句允许 Goroutine 智能地处理来自多个通道的请求。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter10/channels`的新目录，并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/channels 
```

您应该看到一个名为`go.mod`的文件，其中包含以下代码：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/channels    
```

1.  复制`~/projects/go-programming-cookbook-original/chapter10/channels`中的测试，或者利用这个机会编写一些您自己的代码！

1.  创建一个名为`sender.go`的文件，内容如下：

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

1.  创建一个名为`printer.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录，并转到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "context"
            "time"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter10/channels"
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
            time.Sleep(3 * time.Second)
        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您现在应该看到以下输出，但打印顺序可能会有所不同：

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

1.  `go.mod`文件可能会被更新，顶级示例目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

此示例演示了启动读取或写入通道的工作进程的两种方法，并且可能同时执行两者。当写入`done`通道或通过调用取消函数或超时取消`context`时，工作进程将终止。*使用上下文包*示例将更详细地介绍`context`包。

`main`包用于将各个函数连接在一起；由于这一点，可以设置多个成对，只要通道不共享。除此之外，可以有多个 Goroutines 监听同一个通道，我们将在*使用工作池设计模式*示例中探讨。

最后，由于 Goroutines 的异步性质，建立清理和终止条件可能会很棘手；例如，一个常见的错误是执行以下操作：

```go
select{
    case <-time.Tick(200 * time.Millisecond):
    //this resets whenever any other 'lane' is chosen
}
```

通过将`Tick`放在`select`语句中，可以防止这种情况发生。在`select`语句中也没有简单的方法来优先处理流量。

# 使用 sync.WaitGroup 执行异步操作

有时，异步执行一些操作并等待它们完成是有用的。例如，如果一个操作需要从多个 API 中提取信息并聚合该信息，那么将这些客户端请求异步化将会很有帮助。这个示例将探讨如何使用`sync.WaitGroup`来编排并行的非依赖任务。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter10/waitgroup`的新目录，并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/waitgroup 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/waitgroup    
```

1.  从`~/projects/go-programming-cookbook-original/chapter10/waitgroup`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`tasks.go`的文件，其中包含以下内容：

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

        // Present can be used to determine if
        // we should return this
        func (c *CrawlError) Present() bool {
            return len(c.Errors) != 0
        }
```

1.  创建一个名为`process.go`的文件，其中包含以下内容：

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
            // we encountered a crawl error
            if cerr.Present() {
                return resps, cerr
            }
            log.Printf("completed crawling in %s", time.Since(start))
            return resps, nil
        }
```

1.  创建一个名为`example`的新目录，并转到该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter10/waitgroup"
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

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

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

1.  `go.mod`文件可能会更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

这个示例向您展示了如何在等待工作时使用`waitgroups`作为同步机制。实质上，`waitgroup.Wait()`将等待其内部计数器达到`0`。`waitgroup.Add(int)`方法将按输入的数量递增计数器，`waitgroup.Done()`将递减计数器`1`。因此，必须异步`Wait()`，而各种 Goroutines 标记`waitgroup`为`Done()`。

在这个示例中，我们在分派每个 HTTP 请求之前递增，然后调用 defer `wg.Done()`方法，这样我们就可以在 Goroutine 终止时递减。然后我们等待所有 Goroutines 完成，然后返回我们聚合的结果。

实际上，最好使用通道来传递错误和响应。

在执行此类异步操作时，您应该考虑诸如修改共享映射之类的事物的线程安全性。如果您记住这一点，`waitgroups`是等待任何类型的异步操作的有用功能。

# 使用原子操作和互斥

在诸如 Go 之类的语言中，您可以构建异步操作和并行性，考虑诸如线程安全之类的事情变得很重要。例如，同时从多个 Goroutines 访问映射是危险的。Go 在`sync`和`sync/atomic`包中提供了许多辅助工具，以确保某些事件仅发生一次，或者 Goroutines 可以在操作上进行序列化。

这个示例将演示使用这些包来安全地修改具有各种 Goroutines 的映射，并保持可以被多个 Goroutines 安全访问的全局序数值。它还将展示`Once.Do`方法，该方法可用于确保 Go 应用程序只执行一次某些操作，例如读取配置文件或初始化变量。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter10/atomic`的新目录，并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/atomic 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/atomic    
```

1.  从`~/projects/go-programming-cookbook-original/chapter10/atomic`复制测试，或者利用这个机会编写一些自己的代码！

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

1.  创建一个名为`example`的新目录并进入。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "sync"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter10/atomic"
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
$ go build $ ./example
```

现在你应该看到以下输出：

```go
$ go run main.go
initial ordinal is: 1123
final ordinal is: 1133
all keys found and marked as: 'success'
```

1.  `go.mod`文件可能已更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果你复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

对于我们的 map 配方，我们使用了`ReadWrite`互斥锁。这个互斥锁的思想是任意数量的读取者可以获取读取锁，但只有一个写入者可以获取写入锁。此外，当其他人（读取者或写入者）拥有锁时，写入者不能获取锁。这很有用，因为读取非常快速且非阻塞，与标准互斥锁相比。每当我们想要设置数据时，我们使用`Lock()`对象，每当我们想要读取数据时，我们使用`RLock()`。关键是你最终要使用`Unlock()`或`RUnlock()`，这样你就不会使你的应用程序死锁。延迟的`Unlock()`对象可能很有用，但可能比手动调用`Unlock()`慢。

当你想要将额外的操作与锁定的值分组时，这种模式可能不够灵活。例如，在某些情况下，你可能想要锁定，进行一些额外的处理，只有在完成这些处理后才解锁。对于你的设计来说，考虑这一点是很重要的。

`sync/atmoic`包被`Ordinal`用来获取和设置值。还有原子比较操作，比如`atomic.CompareAndSwapUInt64()`，非常有价值。这个配方允许只能在`Ordinal`对象上调用`Init`一次；否则，它只能被原子地递增。

我们循环创建 10 个 Goroutines（与`sync.Waitgroup`同步），并展示序数正确递增了 10 次，我们的 map 中的每个键都被适当地设置。

# 使用上下文包

本书中的几个配方都使用了`context`包。这个配方将探讨创建和管理上下文的基础知识。理解上下文的一个很好的参考是[`blog.golang.org/context`](https://blog.golang.org/context)。自从写这篇博客以来，上下文已经从`net/context`移动到一个叫做`context`的包中。这在与 GRPC 等第三方库交互时仍然偶尔会引起问题。

这个配方将探讨为上下文设置和获取值，取消和超时。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter10/context`的新目录并进入。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/context 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/context    
```

1.  从`~/projects/go-programming-cookbook-original/chapter10/context`复制测试，或者利用这个机会编写一些自己的代码！

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

1.  创建一个名为`example`的新目录并进入。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

            import "github.com/PacktPublishing/
                    Go-Programming-Cookbook-Second-Edition/
                    chapter10/context"

        func main() {
            context.Exec()
        }
```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

现在你应该看到以下输出：

```go
$ go run main.go
timeout exceeded
      OR
$ go run main.go
deadline exceeded
```

1.  `go.mod`文件可能已更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果你复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

当使用上下文值时，最好创建一个新类型来表示键。在这种情况下，我们创建了一个`key`类型，然后声明了一些对应的`const`值来表示所有可能的键。

在这种情况下，我们使用`Setup()`函数同时初始化所有的键/值对。在修改上下文时，函数通常需要一个`context`参数并返回一个`context`值。因此，签名通常如下所示：

```go
func ModifyContext(ctx context.Context) context.Context
```

有时，这些方法还会返回错误或`cancel()`函数，例如`context.WithCancel`、`context.WithTimeout`和`context.WithDeadline`的情况。所有子上下文都继承父上下文的属性。

在这个示例中，我们创建了两个子上下文，一个带有截止日期，一个带有超时。我们将这些超时设置为随机范围，然后在接收到任何一个超时时终止。最后，我们提取了给定键的值并打印出来。

# 执行通道的状态管理

在 Go 中，通道可以是任何类型。结构体通道允许您通过单个消息传递大量状态。本示例将探讨使用通道传递复杂请求结构并在复杂响应结构中返回它们的结果。

在下一个示例中，*使用工作池设计模式*，这种价值变得更加明显，因为您可以创建能够执行各种任务的通用工作程序。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter10/state`的新目录并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/state 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/state    
```

1.  复制`~/projects/go-programming-cookbook-original/chapter10/state`中的测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`state.go`的文件，内容如下：

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

1.  创建一个名为`processor.go`的文件，内容如下：

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

1.  创建一个名为`process.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录并进入该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "context"
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter10/state"
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

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

现在应该看到以下输出：

```go
$ go run main.go
Request: &{add 3 4}; Result: 7, Error: <nil>
Request: &{sub 5 2}; Result: 3, Error: <nil>
Request: &{mult 9 9}; Result: 81, Error: <nil>
Request: &{div 8 2}; Result: 4, Error: <nil>
Request: &{div 8 0}; Result: 0, Error: divide by 0
```

1.  `go.mod`文件可能会被更新，顶级示例目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

本示例中的`Processor()`函数是一个循环函数，直到其上下文被取消为止，可以通过显式调用取消或超时来取消。它将所有工作分派给`Process()`，当给定各种操作时，它可以处理不同的函数。也可以让每个这些情况分派另一个函数，以获得更模块化的代码。

最终，响应被返回到响应通道，并且我们在最后循环打印所有结果。我们还演示了`divide by 0`示例中的错误情况。

# 使用工作池设计模式

工作池设计模式是一种将长时间运行的 Goroutines 作为工作程序分派的模式。这些工作程序可以使用多个通道处理各种工作，也可以使用描述类型的有状态请求结构，如前面的示例所述。本示例将创建有状态的工作程序，并演示如何协调和启动多个工作程序，它们都在同一个通道上并发处理请求。这些工作程序将是`crypto`工作程序，就像在 Web 身份验证应用程序中一样。它们的目的将是使用`bcrypt`包对明文字符串进行哈希处理，并将文本密码与哈希进行比较。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter10/pool`的新目录并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/pool 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/pool    
```

1.  复制`~/projects/go-programming-cookbook-original/chapter10/pool`中的测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`worker.go`的文件，其中包含以下内容：

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

1.  创建一个名为`work.go`的文件，其中包含以下内容：

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

1.  创建一个名为`crypto.go`的文件，其中包含以下内容：

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

1.  创建一个名为`example`的新目录，并进入该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter10/pool"
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

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

现在你应该看到以下输出：

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

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶层示例目录中。

1.  如果你复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

这个示例使用`Dispatch()`方法在单个输入通道、输出通道和连接到单个`cancel()`函数上创建多个工作人员。如果你想为不同的目的创建不同的池，这个方法就可以使用。例如，你可以通过使用单独的池创建 10 个`crypto`和 20 个`compare`工作人员。对于这个示例，我们使用一个单一的池，将哈希请求发送给工作人员，检索响应，然后将`compare`请求发送到同一个池中。因此，执行工作的工作人员每次都会不同，但它们都能执行任何类型的工作。

这种方法的优点是，这两种请求都允许并行处理，并且还可以控制最大并发数。限制 Goroutines 的最大数量对于限制内存也很重要。我选择了`crypto`作为这个示例，因为`crypto`是一个很好的例子，它可以通过为每个新请求启动一个新的 Goroutine 来压倒你的 CPU 或内存；例如，在一个 web 服务中。

# 使用工作人员创建管道

这个示例演示了创建工作池组并将它们连接在一起形成一个管道。对于这个示例，我们将两个池连接在一起，但这种模式可以用于更复杂的操作，类似于中间件。

工作池对于保持工作人员相对简单并进一步控制并发非常有用。例如，将日志串行化，同时并行化其他操作可能很有用。对于更昂贵的操作，拥有一个较小的池也可能很有用，这样你就不会过载机器资源。

# 操作步骤如下...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter10/pipeline`的新目录，并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/pipeline 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter10/pipeline    
```

1.  复制`~/projects/go-programming-cookbook-original/chapter10/pipeline`中的测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`worker.go`的文件，其中包含以下内容：

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

1.  创建一个名为`print.go`的文件，其中包含以下内容：

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

1.  创建一个名为`encode.go`的文件，其中包含以下内容：

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

1.  创建一个名为`pipeline.go`的文件，其中包含以下内容：

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

1.  创建一个名为`example`的新目录，并进入该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "context"
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter10/pipeline"
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

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

现在你应该看到以下输出：

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

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶层示例目录中。

1.  如果你复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

`main`包创建了一个包含 10 个编码器和 2 个打印机的管道。它在输入通道上排队了 20 个字符串，并等待在输出通道上获得 20 个响应。如果消息到达输出通道，表示它们已经成功通过整个管道。

`NewPipeline` 函数用于连接管道。它确保通道以适当的缓冲区大小创建，并且一些池的输出通道连接到其他池的适当输入通道。还可以通过在每个工作器上使用输入通道数组和输出通道数组，多个命名通道，或通道映射来扩展管道。这将允许诸如在每个步骤发送消息到记录器之类的操作。
