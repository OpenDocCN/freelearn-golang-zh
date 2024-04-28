# 第十四章：性能改进、技巧和诀窍

在本章中，我们将专注于优化应用程序和发现瓶颈。这些都是一些可立即被现有应用程序使用的技巧。如果您或您的组织需要完全可重现的构建，许多这些食谱是必需的。当您想要对应用程序的性能进行基准测试时，它们也是有用的。最后一个食谱侧重于提高 HTTP 的速度；然而，重要的是要记住网络世界变化迅速，重要的是要及时了解最佳实践。例如，如果您需要 HTTP/2，自 Go 1.6 版本以来，可以使用内置的 Go `net/http`包。

在这一章中，我们将涵盖以下食谱：

+   使用 pprof 工具

+   基准测试和发现瓶颈

+   内存分配和堆管理

+   使用 fasthttprouter 和 fasthttp

# 技术要求

为了继续本章中的所有食谱，请根据以下步骤配置您的环境：

1.  在您的操作系统上从[`golang.org/doc/install`](https://golang.org/doc/install)下载并安装 Go 1.12.6 或更高版本。

1.  打开一个终端或控制台应用程序，并创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。所有代码将从该目录运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，并选择从该目录工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

1.  可选地，从[`www.graphviz.org/Home.php`](http://www.graphviz.org/Home.php)安装 Graphviz。

# 使用 pprof 工具

`pprof`工具允许 Go 应用程序收集和导出运行时分析数据。它还提供了用于从 Web 界面访问工具的 Webhook。本食谱将创建一个基本应用程序，验证`bcrypt`哈希密码与明文密码，然后对应用程序进行分析。

您可能希望在第十一章 *分布式系统*中涵盖`pprof`工具，以及其他指标和监控食谱。但它实际上被放在了本章，因为它将用于分析和改进程序，就像基准测试可以使用一样。因此，本食谱将主要关注于使用`pprof`来分析和改进应用程序的内存使用情况。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter14/pprof`的新目录，并导航到该目录。

1.  运行此命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/pprof 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/pprof   
```

1.  从`~/projects/go-programming-cookbook-original/chapter14/pprof`复制测试，或者使用这个作为练习来编写一些您自己的代码！

1.  创建一个名为`crypto`的目录并导航到该目录。

1.  创建一个名为`handler.go`的文件，内容如下：

```go
        package crypto

        import (
            "net/http"

            "golang.org/x/crypto/bcrypt"
        )

        // GuessHandler checks if ?message=password
        func GuessHandler(w http.ResponseWriter, r *http.Request) {
            if err := r.ParseForm(); err != nil{
               // if we can't parse the form
               // we'll assume it is malformed
               w.WriteHeader(http.StatusBadRequest)
               w.Write([]byte("error reading guess"))
               return
            }

            msg := r.FormValue("message")

            // "password"
            real := 
            []byte("$2a$10$2ovnPWuIjMx2S0HvCxP/mutzdsGhyt8rq/
            JqnJg/6OyC3B0APMGlK")

            if err := bcrypt.CompareHashAndPassword(real, []byte(msg)); 
            err != nil {
                w.WriteHeader(http.StatusBadRequest)
                w.Write([]byte("try again"))
                return
            }

            w.WriteHeader(http.StatusOK)
            w.Write([]byte("you got it"))
            return
        }
```

1.  导航到上一级目录。

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个`main.go`文件，内容如下：

```go
        package main

        import (
            "fmt"
            "log"
            "net/http"
            _ "net/http/pprof"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter14/pprof/crypto"
        )

        func main() {

            http.HandleFunc("/guess", crypto.GuessHandler)
            fmt.Println("server started at localhost:8080")
            log.Panic(http.ListenAndServe("localhost:8080", nil))
        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

现在您应该看到以下输出：

```go
$ go run main.go
server started at localhost:8080
```

1.  在一个单独的终端中，运行以下命令：

```go
$ go tool pprof http://localhost:8080/debug/pprof/profile
```

1.  这将启动一个 30 秒的计时器。

1.  在`pprof`运行时运行几个`curl`命令：

```go
$ curl "http://localhost:8080/guess?message=test"
try again

$curl "http://localhost:8080/guess?message=password" 
you got it

.
.
.
.

$curl "http://localhost:8080/guess?message=password" 
you got it  
```

1.  返回到`pprof`命令并等待其完成。

1.  从`pprof`提示符中运行`top10`命令：

```go
(pprof) top 10
930ms of 930ms total ( 100%)
Showing top 10 nodes out of 15 (cum >= 930ms)
flat flat% sum% cum cum%
870ms 93.55% 93.55% 870ms 93.55% 
golang.org/x/crypto/blowfish.encryptBlock
30ms 3.23% 96.77% 900ms 96.77% 
golang.org/x/crypto/blowfish.ExpandKey
30ms 3.23% 100% 30ms 3.23% runtime.memclrNoHeapPointers
0 0% 100% 930ms 100% github.com/agtorre/go-
cookbook/chapter13/pprof/crypto.GuessHandler
0 0% 100% 930ms 100% 
golang.org/x/crypto/bcrypt.CompareHashAndPassword
0 0% 100% 30ms 3.23% golang.org/x/crypto/bcrypt.base64Encode
0 0% 100% 930ms 100% golang.org/x/crypto/bcrypt.bcrypt
0 0% 100% 900ms 96.77% 
golang.org/x/crypto/bcrypt.expensiveBlowfishSetup
0 0% 100% 930ms 100% net/http.(*ServeMux).ServeHTTP
0 0% 100% 930ms 100% net/http.(*conn).serve
```

1.  如果您安装了 Graphviz 或支持的浏览器，请从`pprof`提示符中运行`web`命令。您应该会看到类似这样的东西，右侧有一长串红色框：

！[](img/7df536c6-2641-4619-97a3-e9ada9b6084d.png)

1.  `go.mod`文件可能会更新，`go.sum`文件现在应该存在于顶级食谱目录中。

1.  如果您已经复制或编写了自己的测试，请返回到上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`pprof`工具提供了关于应用程序的许多运行时信息。使用`net/pprof`包通常是最简单的配置方式，只需要在端口上进行监听并导入即可。

在我们的案例中，我们编写了一个处理程序，使用了一个非常计算密集的应用程序（`bcrypt`），以便演示在使用`pprof`进行分析时它们是如何出现的。这将快速地分离出在应用程序中创建瓶颈的代码块。

我们选择收集一个通用概要，导致`pprof`在 30 秒内轮询我们的应用程序端点。然后我们对端点生成流量，以帮助产生结果。当您尝试检查单个处理程序或代码分支时，这可能会有所帮助。

最后，我们查看了在 CPU 利用率方面排名前 10 的函数。还可以使用`pprof http://localhost:8080/debug/pprof/heap`命令查看内存/堆管理。`pprof`控制台中的`web`命令可用于查看 CPU/内存概要的可视化，并有助于突出更活跃的代码。

# 基准测试和查找瓶颈

使用基准测试来确定代码中的慢部分是另一种方法。基准测试可用于测试函数的平均性能，并且还可以并行运行基准测试。这在比较函数或对特定代码进行微优化时非常有用，特别是要查看在并发使用时函数实现的性能如何。在本示例中，我们将创建两个结构，两者都实现了原子计数器。第一个将使用`sync`包，另一个将使用`sync/atomic`。然后我们将对这两种解决方案进行基准测试。

# 如何操作...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter14/bench`的新目录，并导航到该目录。

1.  运行此命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/bench 
```

您应该会看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/bench   
```

1.  从`~/projects/go-programming-cookbook-original/chapter14/bench`复制测试，或者将其作为练习编写一些自己的代码！

请注意，复制的测试还包括本示例中稍后编写的基准测试。

1.  创建一个名为`lock.go`的文件，内容如下：

```go
        package bench

        import "sync"

        // Counter uses a sync.RWMutex to safely
        // modify a value
        type Counter struct {
            value int64
            mu *sync.RWMutex
        }

        // Add increments the counter
        func (c *Counter) Add(amount int64) {
            c.mu.Lock()
            c.value += amount
            c.mu.Unlock()
        }

        // Read returns the current counter amount
        func (c *Counter) Read() int64 {
            c.mu.RLock()
            defer c.mu.RUnlock()
            return c.value
        }
```

1.  创建一个名为`atomic.go`的文件，内容如下：

```go
        package bench

        import "sync/atomic"

        // AtomicCounter implements an atmoic lock
        // using the atomic package
        type AtomicCounter struct {
            value int64
        }

        // Add increments the counter
        func (c *AtomicCounter) Add(amount int64) {
            atomic.AddInt64(&c.value, amount)
        }

        // Read returns the current counter amount
        func (c *AtomicCounter) Read() int64 {
            var result int64
            result = atomic.LoadInt64(&c.value)
            return result
        }
```

1.  创建一个名为`lock_test.go`的文件，内容如下：

```go
        package bench

        import "testing"

        func BenchmarkCounterAdd(b *testing.B) {
            c := Counter{0, &sync.RWMutex{}}
            for n := 0; n < b.N; n++ {
                c.Add(1)
            }
        }

        func BenchmarkCounterRead(b *testing.B) {
            c := Counter{0, &sync.RWMutex{}}
            for n := 0; n < b.N; n++ {
                c.Read()
            }
        }

        func BenchmarkCounterAddRead(b *testing.B) {
            c := Counter{0, &sync.RWMutex{}}
            b.RunParallel(func(pb *testing.PB) {
                for pb.Next() {
                    c.Add(1)
                    c.Read()
                }
            })
        }
```

1.  创建一个名为`atomic_test.go`的文件，内容如下：

```go
        package bench

        import "testing"

        func BenchmarkAtomicCounterAdd(b *testing.B) {
            c := AtomicCounter{0}
            for n := 0; n < b.N; n++ {
                c.Add(1)
            }
        }

        func BenchmarkAtomicCounterRead(b *testing.B) {
            c := AtomicCounter{0}
            for n := 0; n < b.N; n++ {
                c.Read()
            }
        }

        func BenchmarkAtomicCounterAddRead(b *testing.B) {
            c := AtomicCounter{0}
            b.RunParallel(func(pb *testing.PB) {
                for pb.Next() {
                    c.Add(1)
                    c.Read()
                }
            })
        }
```

1.  运行`go test -bench .`命令，您将看到以下输出：

```go
$ go test -bench . 
BenchmarkAtomicCounterAdd-4 200000000 8.38 ns/op
BenchmarkAtomicCounterRead-4 1000000000 2.09 ns/op
BenchmarkAtomicCounterAddRead-4 50000000 24.5 ns/op
BenchmarkCounterAdd-4 50000000 34.8 ns/op
BenchmarkCounterRead-4 20000000 66.0 ns/op
BenchmarkCounterAddRead-4 10000000 146 ns/op
PASS
ok github.com/PacktPublishing/Go-Programming-Cookbook-Second-
Edition/chapter14/bench 10.919s
```

1.  如果您已经复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

本示例是比较代码的关键路径的一个示例。例如，有时您的应用程序必须经常执行某些功能，也许是每次调用。在这种情况下，我们编写了一个原子计数器，可以从多个 go 例程中添加或读取值。

第一个解决方案使用`RWMutex`和`Lock`或`RLock`对象进行写入和读取。第二个使用`atomic`包，它提供了相同的功能。我们使函数的签名相同，以便可以在稍作修改的情况下重用基准测试，并且两者都可以满足相同的`atomic`整数接口。

最后，我们为添加值和读取值编写了标准基准测试。然后，我们编写了一个并行基准测试，调用添加和读取函数。并行基准测试将创建大量的锁争用，因此我们预计会出现减速。也许出乎意料的是，`atomic`包明显优于`RWMutex`。

# 内存分配和堆管理

一些应用程序可以从优化中受益很多。例如，考虑路由器，我们将在以后的示例中进行讨论。幸运的是，工具基准测试套件提供了收集许多内存分配以及内存分配大小的标志。调整某些关键代码路径以最小化这两个属性可能会有所帮助。

这个教程将展示编写一个将字符串用空格粘合在一起的函数的两种方法，类似于`strings.Join("a", "b", "c")`。一种方法将使用连接，而另一种方法将使用`strings`包。然后我们将比较这两种方法之间的性能和内存分配。

# 如何做...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter14/tuning`的新目录，并导航到该目录。

1.  运行此命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/tuning 
```

应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/tuning   
```

1.  从`~/projects/go-programming-cookbook-original/chapter14/tuning`复制测试，或者将其作为练习编写一些您自己的代码！

请注意，复制的测试还包括稍后在本教程中编写的基准测试。

1.  创建一个名为`concat.go`的文件，其中包含以下内容：

```go
        package tuning

        func concat(vals ...string) string {
            finalVal := ""
            for i := 0; i < len(vals); i++ {
                finalVal += vals[i]
                if i != len(vals)-1 {
                    finalVal += " "
                }
            }
            return finalVal
        }
```

1.  创建一个名为`join.go`的文件，其中包含以下内容：

```go
        package tuning

        import "strings"

        func join(vals ...string) string {
            c := strings.Join(vals, " ")
            return c
        }
```

1.  创建一个名为`concat_test.go`的文件，其中包含以下内容：

```go
        package tuning

        import "testing"

        func Benchmark_concat(b *testing.B) {
            b.Run("one", func(b *testing.B) {
                one := []string{"1"}
                for i := 0; i < b.N; i++ {
                    concat(one...)
                }
            })
            b.Run("five", func(b *testing.B) {
                five := []string{"1", "2", "3", "4", "5"}
                for i := 0; i < b.N; i++ {
                    concat(five...)
                }
            })

            b.Run("ten", func(b *testing.B) {
                ten := []string{"1", "2", "3", "4", "5",
                "6", "7", "8", "9", "10"}
                for i := 0; i < b.N; i++ {
                    concat(ten...)
                }
            })
        }
```

1.  创建一个名为`join_test.go`的文件，其中包含以下内容：

```go
        package tuning

        import "testing"

        func Benchmark_join(b *testing.B) {
            b.Run("one", func(b *testing.B) {
                one := []string{"1"}
                for i := 0; i < b.N; i++ {
                    join(one...)
                }
            })
            b.Run("five", func(b *testing.B) {
                five := []string{"1", "2", "3", "4", "5"}
                for i := 0; i < b.N; i++ {
                    join(five...)
                }
            })

            b.Run("ten", func(b *testing.B) {
                ten := []string{"1", "2", "3", "4", "5",
                "6", "7", "8", "9", "10"}
                    for i := 0; i < b.N; i++ {
                        join(ten...)
                    }
            })
        }
```

1.  运行`GOMAXPROCS=1 go test -bench=. -benchmem -benchtime=1s`命令，您将看到以下输出：

```go
$ GOMAXPROCS=1 go test -bench=. -benchmem -benchtime=1s
Benchmark_concat/one 100000000 13.6 ns/op 0 B/op 0 allocs/op
Benchmark_concat/five 5000000 386 ns/op 48 B/op 8 allocs/op
Benchmark_concat/ten 2000000 992 ns/op 256 B/op 18 allocs/op
Benchmark_join/one 200000000 6.30 ns/op 0 B/op 0 allocs/op
Benchmark_join/five 10000000 124 ns/op 32 B/op 2 allocs/op
Benchmark_join/ten 10000000 183 ns/op 64 B/op 2 allocs/op
PASS
ok github.com/PacktPublishing/Go-Programming-Cookbook-Second-
Edition/chapter14/tuning 12.003s
```

1.  如果您已经复制或编写了自己的测试，请运行`go test`。确保所有测试都通过。

# 工作原理...

基准测试有助于调整应用程序并进行某些微优化，例如内存分配。在对带有输入的应用程序进行分配基准测试时，重要的是要尝试各种输入大小，以确定它是否会影响分配。我们编写了两个函数，`concat`和`join`。两者都将`variadic`字符串参数与空格连接在一起，因此参数（*a*，*b*，*c*）将返回字符串*a b c*。

`concat`方法仅通过字符串连接实现这一点。我们创建一个字符串，并在列表中和`for`循环中添加字符串和空格。我们在最后一个循环中省略添加空格。`join`函数使用内部的`Strings.Join`函数来更有效地完成这个任务。与您自己的函数相比，对标准库进行基准测试有助于更好地理解性能、简单性和功能性之间的权衡。

我们使用子基准测试来测试所有参数，这也与表驱动基准测试非常搭配。我们可以看到`concat`方法在单个长度输入的情况下比`join`方法产生了更多的分配。一个很好的练习是尝试使用可变长度的输入字符串以及一些参数来进行测试。

# 使用 fasthttprouter 和 fasthttp

尽管 Go 标准库提供了运行 HTTP 服务器所需的一切，但有时您需要进一步优化诸如路由和请求时间等内容。本教程将探讨一个加速请求处理的库，称为`fasthttp`（[`github.com/valyala/fasthttp`](https://github.com/valyala/fasthttp)），以及一个显著加速路由性能的路由器，称为`fasthttprouter`（[`github.com/buaazp/fasthttprouter`](https://github.com/buaazp/fasthttprouter)）。尽管`fasthttp`很快，但重要的是要注意它不支持 HTTP/2（[`github.com/valyala/fasthttp/issues/45`](https://github.com/valyala/fasthttp/issues/45)）。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter14/fastweb`的新目录，并导航到该目录。

1.  运行此命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/fastweb 
```

应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter14/fastweb   
```

1.  从`~/projects/go-programming-cookbook-original/chapter14/fastweb`复制测试，或者将其作为练习编写一些您自己的代码！

1.  创建一个名为`items.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "sync"
        )

        var items []string
        var mu *sync.RWMutex

        func init() {
            mu = &sync.RWMutex{}
        }

        // AddItem adds an item to our list
        // in a thread-safe way
        func AddItem(item string) {
            mu.Lock()
            items = append(items, item)
            mu.Unlock()
        }

        // ReadItems returns our list of items
        // in a thread-safe way
        func ReadItems() []string {
            mu.RLock()
            defer mu.RUnlock()
            return items
        }
```

1.  创建一个名为`handlers.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "encoding/json"

            "github.com/valyala/fasthttp"
        )

        // GetItems will return our items object
        func GetItems(ctx *fasthttp.RequestCtx) {
            enc := json.NewEncoder(ctx)
            items := ReadItems()
            enc.Encode(&items)
            ctx.SetStatusCode(fasthttp.StatusOK)
        }

        // AddItems modifies our array
        func AddItems(ctx *fasthttp.RequestCtx) {
            item, ok := ctx.UserValue("item").(string)
            if !ok {
                ctx.SetStatusCode(fasthttp.StatusBadRequest)
                return
            }

            AddItem(item)
            ctx.SetStatusCode(fasthttp.StatusOK)
        }
```

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"
            "log"

            "github.com/buaazp/fasthttprouter"
            "github.com/valyala/fasthttp"
        )

        func main() {
            router := fasthttprouter.New()
            router.GET("/item", GetItems)
            router.POST("/item/:item", AddItems)

            fmt.Println("server starting on localhost:8080")
            log.Fatal(fasthttp.ListenAndServe("localhost:8080", 
            router.Handler))
        }
```

1.  运行`go build`命令。

1.  运行`./fastweb`命令：

```go
$ ./fastweb
server starting on localhost:8080
```

1.  从单独的终端，使用一些`curl`命令进行测试：

```go
$ curl "http://localhost:8080/item/hi" -X POST 

$ curl "http://localhost:8080/item/how" -X POST 

$ curl "http://localhost:8080/item/are" -X POST 

$ curl "http://localhost:8080/item/you" -X POST 

$ curl "http://localhost:8080/item" -X GET 
["hi","how", "are", "you"]
```

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果您已经复制或编写了自己的测试，请运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`fasthttp`和`fasthttprouter`包可以大大加快 Web 请求的生命周期。这两个包在热代码路径上进行了大量优化，但不幸的是，需要重写处理程序以使用新的上下文对象，而不是传统的请求和响应写入器。

有许多框架采用了类似的路由方法，有些直接集成了`fasthttp`。这些项目在它们的`README`文件中保持最新的信息。

我们的配方实现了一个简单的`list`对象，我们可以通过一个端点进行附加，然后由另一个端点返回。这个配方的主要目的是演示如何处理参数，设置一个现在明确定义支持的方法的路由器，而不是通用的`Handle`和`HandleFunc`，并展示它们与标准处理程序有多么相似，但又有许多其他好处。
