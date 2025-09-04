

# 构建网络服务

本章的核心主题是使用 `net/http` 包处理 HTTP——请记住，所有网络服务都需要网络服务器才能运行。此外，在本章中，我们将把统计应用程序转换为接受 HTTP 连接的网络应用程序，并创建一个命令行客户端来与之交互。在章节的最后部分，我们将学习如何超时 HTTP 连接。

更详细地说，本章涵盖了：

+   `net/http` 包

+   创建一个网络服务器

+   更新统计应用程序

+   开发网络客户端

+   为统计服务创建客户端

+   超时 HTTP 连接

# `net/http` 包

`net/http` 包提供了允许你开发网络服务器和客户端的函数。例如，客户端使用 `http.Get()` 和 `http.NewRequest()` 来发送 HTTP 请求，而 `http.ListenAndServe()` 则用于通过指定服务器监听的 IP 地址和 TCP 端口来启动网络服务器。此外，`http.HandleFunc()` 定义了支持的 URL 以及将要处理这些 URL 的函数。

接下来的三个小节描述了 `net/http` 包中的三个重要数据结构——在阅读本章时，你可以将这些描述作为参考。

## `http.Response` 类型

`http.Response` 结构体体现了 HTTP 请求的响应——一旦收到响应头，`http.Client` 和 `http.Transport` 都会返回 `http.Response` 值。其定义可以在 [`go.dev/src/net/http/response.go`](https://go.dev/src/net/http/response.go) 找到：

```go
type Response struct {
    Status     string // e.g. "200 OK"
    StatusCode int // e.g. 200
    Proto      string // e.g. "HTTP/1.0"
    ProtoMajor int // e.g. 1
    ProtoMinor int // e.g. 0
    Header Header
    Body io.ReadCloser 
    ContentLength int64
    TransferEncoding []string
    Close bool
    Uncompressed bool
    Trailer Header 
    Request *Request
    TLS *tls.ConnectionState
} 
```

你不必使用所有结构字段，但了解它们的存在是好的。然而，其中一些字段，如 `Status`、`StatusCode` 和 `Body`，比其他字段更重要。Go 源文件以及 `go doc http.Response` 的输出都包含了关于每个字段目的的更多信息，这同样适用于标准 Go 库中找到的大多数 `struct` 数据类型。

## `http.Request` 类型

`http.Request` 结构体代表了一个客户端构建的 HTTP 请求，以便发送或接收 HTTP 服务器。`http.Request` 的公共字段如下：

```go
type Request struct {
    Method string
    URL *url.URL
    Proto  string
    ProtoMajor int
    ProtoMinor int
    Header Header
    Body io.ReadCloser
    GetBody func() (io.ReadCloser, error)
    ContentLength int64
    TransferEncoding []string
    Close bool
    Host string
    Form url.Values
    PostForm url.Values
    MultipartForm *multipart.Form
    Trailer Header
    RemoteAddr string
    RequestURI string
    TLS *tls.ConnectionState
    Cancel <-chan struct{}
    Response *Response
} 
```

`Body` 字段包含请求的主体。在读取请求的主体之后，你可以调用 `GetBody()`，它返回主体的新副本——这是可选的。

现在让我们介绍 `http.Transport` 结构体。

## `http.Transport` 类型

`http.Transport` 的定义，它为你提供了更多对 HTTP 连接的控制，相当长且复杂：

```go
type Transport struct {
    Proxy func(*Request) (*url.URL, error)
    DialContext func(ctx context.Context, network, addr string) (net.Conn, error)
    Dial func(network, addr string) (net.Conn, error)
    DialTLSContext func(ctx context.Context, network, addr string) (net.Conn, error)
    DialTLS func(network, addr string) (net.Conn, error)
    TLSClientConfig *tls.Config
    TLSHandshakeTimeout time.Duration
    DisableKeepAlives bool
    DisableCompression bool
    MaxIdleConns int
    MaxIdleConnsPerHost int
    MaxConnsPerHost int
    IdleConnTimeout time.Duration
    ResponseHeaderTimeout time.Duration
    ExpectContinueTimeout time.Duration
    TLSNextProto map[string]func(authority string, c *tls.Conn) RoundTripper
    ProxyConnectHeader Header
    GetProxyConnectHeader func(ctx context.Context, proxyURL *url.URL, target string) (Header, error)
    MaxResponseHeaderBytes int64
    WriteBufferSize int
    ReadBufferSize int
    ForceAttemptHTTP2 bool
} 
```

请记住，与`http.Client`相比，`http.Transport`是低级别的。后者实现了一个高级 HTTP 客户端——每个`http.Client`都包含一个`Transport`字段。你不需要在所有程序中使用`http.Transport`，也不需要始终处理它的所有字段。要了解更多关于`DefaultTransport`的信息，请输入`go doc http.DefaultTransport`。

让我们现在学习如何开发一个 Web 服务器。

# 创建 Web 服务器

本节介绍了一个用 Go 开发的简单 Web 服务器，以更好地理解此类应用程序背后的原理。尽管用 Go 编写的 Web 服务器可以高效且安全地做很多事情，但如果你真正需要的是一个支持模块、多个网站和虚拟主机的强大 Web 服务器，那么你最好使用 Apache、Nginx 或 Caddy 这样的 Web 服务器，这些服务器是用 Go 编写的。这些强大的 Web 服务器通常位于 Go 应用服务器之前。

你可能会问，为什么所展示的 Web 服务器使用 HTTP 而不是安全的 HTTP（HTTPS）。这个问题的答案很简单：大多数 Go Web 服务器都是以 Docker 镜像的形式部署的，并且隐藏在提供安全 HTTP 操作部分的 Web 服务器后面，例如 Caddy 和 Nginx，它们使用适当的认证信息提供安全 HTTP 操作。在不了解如何以及将在哪个域名下部署应用程序的情况下，使用安全 HTTP 协议以及所需的认证信息是没有意义的。

这是在微服务和作为 Docker 镜像部署的常规 Web 应用中的一种常见做法。因此，这是一个在这种情况下常见的做法的设计决策。然而，你的需求可能不同。

`net/http`包提供了函数和数据类型，允许你开发强大的 Web 服务器和客户端。`http.Set()`和`http.Get()`方法可以用来发送 HTTP 和 HTTPS 请求，而`http.ListenAndServe()`用于创建 Web 服务器，给定用户指定的处理函数或处理传入请求的函数。由于大多数 Web 服务需要支持多个端点，你最终需要多个离散的函数来处理传入的请求，这也导致了你服务器的更好设计。

定义受支持的端点以及响应每个客户端请求的处理函数的最简单方法就是使用`http.HandleFunc()`，它可以被多次调用。

在这个快速且有些理论性的介绍之后，是时候开始讨论更实际的话题了，从实现一个简单的 Web 服务器开始，如`wwwServer.go`所示：

```go
package main
import (
    "fmt"
"net/http"
"os"
"time"
)
func myHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Serving: %s\n", r.URL.Path)
    fmt.Printf("Served: %s\n", r.Host)
} 
```

这是一个处理函数，它使用`w http.ResponseWriter`向客户端发送消息，`http.ResponseWriter`是一个实现`io.Writer`接口的接口，用于发送服务器响应。

```go
func timeHandler(w http.ResponseWriter, r *http.Request) {
    t := time.Now().Format(time.RFC1123)
    Body := "The current time is:"
    fmt.Fprintf(w, "<h1 align=\"center\">%s</h1>", Body)
    fmt.Fprintf(w, "<h2 align=\"center\">%s</h2>\n", t)
    fmt.Fprintf(w, "Serving: %s\n", r.URL.Path)
    fmt.Printf("Served time for: %s\n", r.Host)
} 
```

这是一个名为`timeHandler`的另一个处理器函数，它以 HTML 格式返回当前时间。所有的`fmt.Fprintf()`调用都将数据发送回 HTTP 客户端，而`fmt.Printf()`的输出则打印在 Web 服务器运行的终端上。`fmt.Fprintf()`的第一个参数是`w http.ResponseWriter`，它实现了`io.Writer`接口，因此可以接受用于写入的数据。

```go
func main() {
    PORT := ":8001" 
```

这是你定义你的 Web 服务器将要监听的端口号的地方。

```go
 arguments := os.Args
    if len(arguments) != 1 {
        PORT = ":" + arguments[1]
    }
    fmt.Println("Using port number: ", PORT) 
```

如果你使用端口号`0`，你将得到一个随机选择的可用端口号，这对于测试或当你不想自己指定端口号时非常方便。

如果你不想使用预定义的端口号（`8001`），那么你应该将你自己的端口号作为命令行参数提供给`wwwServer.go`。

```go
 http.HandleFunc("/time", timeHandler)
    http.HandleFunc("/", myHandler) 
```

因此，Web 服务器支持`/time`和`/`这两个 URL。`/`路径匹配所有其他处理器没有匹配的 URL。我们将`myHandler()`与`/`关联的事实使得`myHandler()`成为默认的处理器函数。

```go
 err := http.ListenAndServe(PORT, nil)
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
} 
```

`http.ListenAndServe()`调用使用预定义的端口号开始 HTTP 服务器。由于`PORT`字符串中没有给出主机名，Web 服务器将监听所有可用的网络接口。端口号和主机名应该用冒号（`:`）分隔，即使没有主机名，也应该有这个冒号——在这种情况下，服务器将监听所有可用的网络接口和所有支持的主机名。这就是为什么`PORT`的值是`:8001`而不是仅仅`8001`。

`net/http`包的一部分是`ServeMux`结构体（`go doc http.ServeMux`），它是一个 HTTP 请求多路复用器，它提供了一种与默认方式略有不同的定义处理器函数和端点的方法，默认方式在`wwwServer.go`中使用。所以如果我们不创建和配置我们自己的`ServeMux`变量，那么`http.HandleFunc()`将使用`DefaultServeMux`，即默认的`ServeMux`。因此，在这种情况下，我们将使用默认的 Go 路由器来实现 Web 服务——这就是为什么`http.ListenAndServe()`的第二个参数是`nil`。

运行`wwwServer.go`并使用`curl(1)`与之交互会产生以下输出：

```go
$ go run wwwServer.go
Using port number:  :8001
Served: localhost:8001
Served time for: localhost:8001
Served: localhost:8001 
```

注意，由于`wwwServer.go`不会自动终止，你需要自己停止它。

在`curl(1)`这一侧，交互看起来如下：

```go
$ curl localhost:8001
Serving: / 
```

在第一种情况下，我们访问了 Web 服务器的`/`路径，并由`myHandler()`提供服务。

```go
$ curl localhost:8001/time
<h1 align="center">The current time is:</h1><h2 align="center">Thu, 31 Aug 2023 22:37:37 EEST</h2>
Serving: /time 
```

在这种情况下，我们访问了`/time`，并从`timeHandler()`得到了 HTML 输出。

```go
$ curl localhost:8001/doesNotExist
Serving: /doesNotExist 
```

在最后这种情况中，我们访问了不存在的`/doesNotExist`路径。由于它不能与任何其他路径匹配，因此由默认处理器提供服务，即`myHandler()`函数。

下一个部分是关于将统计应用程序变成 Web 应用程序！

# 更新统计应用程序

这次，统计应用程序将作为一个网络服务运行。需要执行的两个主要任务是定义 API 以及端点，并实现 API。还有一个需要确定的任务是关于应用程序服务器与其客户端之间的数据交换。关于服务器与其客户端之间的数据交换存在许多方法。我们将讨论以下四种方法：

+   使用纯文本

+   使用 HTML

+   使用 JSON

+   使用结合纯文本和 JSON 数据的混合方法

由于在 *第十一章*，*使用 REST API 工作中* 探讨了 JSON，而 HTML 可能不是服务的最佳选择，因为您需要将数据与 HTML 标签分开并解析数据，我们将使用第一种方法。因此，服务将使用纯文本数据。我们首先定义支持统计应用程序操作的 API。

## 定义 API

API 支持以下 URL：

+   `/list`: 这会列出所有可用的条目。

+   `/insert/name/d1/d2/d3/.../`: 这将插入一个新的数据集。在本章的后面部分，我们将看到如何从包含用户数据和参数的 URL 中提取所需信息。关键点是数据集中元素的数量是可变的，因此 URL 将包含可变数量的值。

+   `/delete/name/`: 这是在数据集名称的基础上删除条目的操作。

+   `/search/name/`: 这是在数据集名称的基础上搜索条目的操作。

+   `/status`: 这是一个额外的 URL，它返回统计应用程序中的条目数量。

端点列表不遵循标准的 REST 规范——所有这些内容都将在 *第十一章*，*使用 REST API 工作中* 进行介绍。

这次，**我们不使用默认的 Go 路由器**，这意味着我们定义并配置自己的 `http.NewServeMux()` 变量。这改变了我们提供处理函数的方式：具有 `func(http.ResponseWriter, *http.Request)` 签名的处理函数必须转换为 `http.HandlerFunc` 类型，并由 `ServeMux` 类型及其 `Handle()` 方法使用。因此，当使用不同于默认 Go 路由器（`DefaultServeMux`）的其他 `ServeMux` 时，我们应该通过调用 `http.HandlerFunc()` 来显式进行此转换，这使得 `http.HandlerFunc` 类型充当一个适配器，允许使用具有所需签名的普通函数作为 HTTP 处理器。当使用默认的 Go 路由器时，这不是问题，因为 `http.HandleFunc()` 函数会自动进行此转换。然而，您也可以使用 `ServeMux` 类型的 `HandleFunc()` 方法来进行相同的隐式转换。

为了使事情更清晰，`http.HandlerFunc` 类型支持一个名为 `HandlerFunc()` 的方法——类型和方法都在 `http` 包中定义。同样命名的 `http.HandleFunc()` 函数（不带 `r`）与默认的 Go 路由器一起使用。

例如，对于 `/time` 端点和 `timeHandler()` 处理函数，你应该调用 `mux.Handle()` 如 `mux.Handle("/time", http.HandlerFunc(timeHandler))`。如果你使用 `http.HandleFunc()` 并且因此使用 `DefaultServeMux`，那么你应该调用 `http.HandleFunc("/time", timeHandler)`。

下一个子节的主题是 HTTP 端点的实现。

## 实现处理程序

统计应用的新版本将在 `~/go/src` 目录下创建：`~/go/src/github.com/mactsouk/mGo4th/ch09/server`。正如预期的那样，你还需要执行以下操作：

```go
$ cd ~/go/src/github.com/mactsouk/mGo4th/ch09/server
$ touch handlers.go
$ touch stats.go 
```

如果你使用本书的 GitHub 仓库，你不需要从头创建服务器，因为 Go 代码已经在那里了。

`stats.go` 文件包含定义 Web 服务器操作的代码。通常，处理程序被放在一个单独的外部包中，但为了简单起见，我们决定在同一包内创建一个名为 `handlers.go` 的单独文件来放置处理程序。包含所有与客户端服务相关的功能的 `handlers.go` 文件内容如下：

```go
package main
import (
    "fmt"
"log"
"net/http"
"strconv"
"strings"
) 
```

对于 `handlers.go` 所需的所有包都已导入，即使其中一些已经被 `stats.go` 导入。请注意，包的名称是 `main`，这与 `stats.go` 的情况相同。

```go
const PORT = ":1234" 
```

这是 HTTP 服务器监听的自定义端口号。

```go
func defaultHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host)
    w.WriteHeader(http.StatusOK)
    body := "Thanks for visiting!\n"
    fmt.Fprintf(w, "%s", body)
} 
```

这是默认处理程序，它为所有不匹配其他处理程序请求提供服务。接下来是删除条目的处理程序：

```go
func deleteHandler(w http.ResponseWriter, r *http.Request) {
    // Get dataset
    paramStr := strings.Split(r.URL.Path, "/")
    fmt.Println("Path:", paramStr)
    if len(paramStr) < 3 {
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintln(w, "Not found:", r.URL.Path)
        return
    } 
```

这是 `/delete` 路径的处理函数，它首先分割 URL 以读取所需信息。如果我们没有足够的参数，我们应该使用适当的 HTTP 状态码（在这种情况下是 `http.StatusNotFound`）向客户端发送错误消息。只要它有意义，你可以使用任何你想要的 HTTP 状态码。`WriteHeader()` 方法在写入响应体之前发送带有提供状态码的头部。

```go
 log.Println("Serving:", r.URL.Path, "from", r.Host) 
```

这是 HTTP 服务器向日志文件发送数据的地方——这主要发生在调试原因。

```go
 dataset := paramStr[2]
    err := deleteEntry(dataset)
    if err != nil {
        fmt.Println(err)
        Body := err.Error() + "\n"
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintf(w, "%s", Body)
        return
    } 
```

由于删除过程基于数据集名称，因此所需的所有内容只是一个有效的数据集名称。这是在分割提供的 URL 后读取参数的地方。如果 `deleteEntry()` 函数返回错误，那么我们将构建一个合适的响应并发送给客户端。

```go
 body := dataset + " deleted!\n"
    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, "%s", body)
} 
```

到这一点，我们知道删除操作已成功，因此我们也向客户端发送了适当的消息以及 `http.StatusOK` 状态码。输入 `go doc http.StatusOK` 查看代码列表。

接下来是 `listHandler()` 的实现：

```go
func listHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host)
    w.WriteHeader(http.StatusOK)
    body := list()
    fmt.Fprintf(w, "%s", body)
} 
```

在 `/list` 路径中使用的 `list()` 辅助函数不能失败。因此，在服务 `/list` 时，总是返回 `http.StatusOK`。然而，有时 `list()` 的返回值可以是空字符串。

接下来，我们实现 `statusHandler()`：

```go
func statusHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host)
    w.WriteHeader(http.StatusOK)
    body := fmt.Sprintf("Total entries: %d\n", len(data))
    fmt.Fprintf(w, "%s", body)
} 
```

之前的代码定义了 `/status` 的处理函数。它只是返回统计应用中找到的总条目数。它可以用来验证网络服务是否正常工作。

接下来，我们展示 `insertHandler()` 处理器的实现：

```go
func insertHandler(w http.ResponseWriter, r *http.Request) {
    paramStr := strings.Split(r.URL.Path, "/")
    fmt.Println("Path:", paramStr)
    if len(paramStr) < 4 {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintln(w, "Not enough arguments: "+r.URL.Path)
        return
    } 
```

和之前一样，我们需要分割给定的 URL 以提取信息。在这种情况下，我们需要至少四个元素，因为我们正在尝试将一个新的数据集插入到统计服务中。

```go
 dataset := paramStr[2]
    // These are string values
    dataStr := paramStr[3:]
    data := make([]float64, 0)
    for _, v := range dataStr {
        val, err := strconv.ParseFloat(v, 64)
        if err == nil {
            data = append(data, val)
        }
    } 
```

在之前的代码中，我们初始化了 `dataset` 变量并读取了具有可变长度的数据元素。在这种情况下，我们还需要将数据元素转换为 `float64` 值，因为它们是以文本形式读取的。

```go
 entry := process(dataset, data)
    err := insert(&entry)
    if err != nil {
        w.WriteHeader(http.StatusNotModified)
        Body := "Failed to add record\n"
        fmt.Fprintf(w, "%s", Body)
    } else {
        Body := "New record added successfully\n"
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "%s", Body)
    }
    log.Println("Serving:", r.URL.Path, "from", r.Host)
} 
```

这是 `/insert` 处理器的结束。`insertHandler()` 实现的最后一部分处理 `insert()` 的返回值。如果没有错误，则向客户端发送 `http.StatusOK`。相反，如果返回 `http.StatusNotModified`，则表示统计应用中没有变化。检查交互的状态码是客户端的工作，但向客户端发送适当的响应状态码是服务器的工作。

接下来，我们实现 `searchHandler()`：

```go
func searchHandler(w http.ResponseWriter, r *http.Request) {
    // Get Search value from URL
    paramStr := strings.Split(r.URL.Path, "/")
    fmt.Println("Path:", paramStr)
    if len(paramStr) < 3 {
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintln(w, "Not found: "+r.URL.Path)
        return
    }
    var body string
    dataset := paramStr[2] 
```

在这一点上，我们从 URL 中提取数据集名称，就像我们在 `/delete` 中做的那样。

```go
 t := search(dataset)
    if t == nil {
        w.WriteHeader(http.StatusNotFound)
        body = "Could not be found: " + dataset + "\n"
    } else {
        w.WriteHeader(http.StatusOK)
        body = fmt.Sprintf("%s %d %f %f\n", t.Name, t.Len, t.Mean, t.StdDev)
    }
    log.Println("Serving:", r.URL.Path, "from", r.Host)
    fmt.Fprintf(w, "%s", body)
} 
```

`handlers.go` 的最后一个函数在这里结束，它关于 `/search` 端点。`search()` 辅助函数检查给定的输入是否存在于数据记录中，并相应地执行操作。

此外，`main()` 函数的实现，可以在 `stats.go` 中找到，如下所示：

```go
func main() {
    err := readJSONFile(JSONFILE)
    if err != nil && err != io.EOF {
        fmt.Println("Error:", err)
        return
    }
    createIndex() 
```

`main()` 的这部分与统计应用的正确初始化有关。内部，数据以 JSON 格式存储。

```go
 mux := http.NewServeMux()
    s := &http.Server{
        Addr:         PORT,
        Handler:      mux,
        IdleTimeout:  10 * time.Second,
        ReadTimeout:  time.Second,
        WriteTimeout: time.Second,
    } 
```

在这里，我们将 HTTP 服务器的参数存储在 `http.Server` 结构中，并使用我们自己的 `http.NewServeMux()` 而不是默认的。

```go
 mux.Handle("/list", http.HandlerFunc(listHandler))
    mux.Handle("/insert/", http.HandlerFunc(insertHandler))
    mux.Handle("/insert", http.HandlerFunc(insertHandler))
    mux.Handle("/search", http.HandlerFunc(searchHandler))
    mux.Handle("/search/", http.HandlerFunc(searchHandler))
    mux.Handle("/delete/", http.HandlerFunc(deleteHandler))
    mux.Handle("/status", http.HandlerFunc(statusHandler))
    mux.Handle("/", http.HandlerFunc(defaultHandler)) 
```

这是支持的 URL 列表。请注意，尽管 `/search` 将会失败，因为它没有包含所需的数据，但 `/search` 和 `/search/` 都由同一个处理函数处理。另一方面，`/delete/` 的处理方式不同——这将在测试应用时变得明显。由于我们使用 `http.NewServeMux()` 而不是默认的 Go 路由器，因此在定义处理函数时需要使用 `http.HandlerFunc()`。

```go
 fmt.Println("Ready to serve at", PORT)
    err = s.ListenAndServe()
    if err != nil {
        fmt.Println(err)
        return
    }
} 
```

如本章前面所述，每个 `mux.Handle()` 调用都可以替换为等效的 `mux.HandleFunc()` 调用。因此，`mux.Handle("/list", http.HandlerFunc(listHandler))` 将变为 `mux.HandleFunc("/list", listHandler)`。这同样适用于所有其他的 `mux.Handle()` 调用。

`ListenAndServe()`方法使用在`http.Server`结构中先前定义的参数启动 HTTP 服务器。`stats.go`的其余部分包含与网络服务操作相关的辅助函数。请注意，尽可能频繁地保存和更新应用程序的内容非常重要，因为这是一个实时应用程序，如果它崩溃，你可能会丢失数据。

下一个命令允许你执行应用程序——你需要在`go run`中提供两个文件：

```go
$ go run stats.go handlers.go
Ready to serve at :1234
2023/08/31 17:10:10 Serving: /list from localhost:1234
Path: [ delete d1]
2023/08/31 17:10:20 Serving: /delete/d1 from localhost:1234
Path: [ delete d2]
2023/08/31 17:10:22 Serving: /delete/d2 from localhost:1234
Path: [ delete d1]
2023/08/31 17:10:23 Serving: /delete/d1 from localhost:1234
d1 cannot be found!
2023/08/31 17:11:01 Serving: /status from localhost:1234
Path: [ search d3]
2023/08/31 17:11:26 Serving: /search/d3 from localhost:1234
Path: [ search d2]
2023/08/31 17:11:29 Serving: /search/d2 from localhost:1234
Path: [ search d5]
2023/08/31 17:11:30 Serving: /search/d5 from localhost:1234
Path: [ search d4]
2023/08/31 17:11:32 Serving: /search/d4 from localhost:1234
Path: [ insert v1 1.0 2 3 4 5 ]
2023/08/31 17:16:23 Serving: /insert/v1/1.0/2/3/4/5/ from localhost:1234
Path: [ insert v1 1.0 2 3 4 5 ]
2023/08/31 17:16:34 Serving: /insert/v1/1.0/2/3/4/5/ from localhost:1234
Path: [ insert v2 1.0 2 3 4 5 -5 -3 ]
2023/08/31 17:17:21 Serving: /insert/v2/1.0/2/3/4/5/-5/-3/ from localhost:1234 
```

在客户端，即`curl(1)`，我们有以下交互：

```go
$ curl localhost:1234/list
d6    4    2.325000    1.080220
d4    5    2.860000    1.441666
d1    6    2.216667    1.949715
d2    9    1.000000    0.000000
d0    12    0.333333    0.942809 
```

这里，我们通过访问`/list`获取统计应用程序的所有条目：

```go
$ curl localhost:1234/delete/d1
d1 deleted! 
```

如果`d1`存在于现有数据集列表中，之前的命令将会工作。如果你的列表为空或`d1`不存在，你应该在删除它之前将其包含在内。

```go
$ curl localhost:1234/delete/d2
d2 deleted!
$ curl localhost:1234/delete/d1
d1 cannot be found! 
```

在上一部分，我们尝试删除了`d1`和`d2`数据集。再次尝试删除`d1`会失败。

```go
$ curl localhost:1234/status
Total entries: 3 
```

接下来，我们访问`/status`并得到预期的输出：

```go
$ curl localhost:1234/search/d3
Could not be found: d3
$ curl localhost:1234/search/d4
d4 5 2.860000 1.44166 
```

首先，我们搜索不存在的`d3`，然后搜索存在的`d4`。在后一种情况下，网络服务返回`d4`的数据。现在，让我们尝试访问`/delete`而不是`/delete/`：

```go
$ curl localhost:1234/delete
<a href="/delete/">Moved Permanently</a>. 
```

所展示的消息是由 Go 路由器生成的，并告诉我们应该尝试`/delete/`而不是`/delete`，因为`/delete`已被永久移动。这是我们没有在路由中明确定义`/delete`和`/delete/`时可能会得到的消息类型。

现在，让我们插入两个数据集：

```go
$ curl localhost:1234/insert/v1/1.0/2/3/4/5/
New record added successfully
$ curl localhost:1234/insert/v2/1.0/2/3/4/5/-5/-3/
New record added successfully 
```

如果`v1`和`v2`都不存在，之前的命令将会工作。如果我们尝试插入一个已存在的数据集，除了`304 – Not Modified`之外，我们不会收到任何响应。

所有的东西看起来都在正常工作。现在我们可以将统计网络服务上线，并通过多个 HTTP 请求与之交互，因为`http`包使用多个 goroutine 与客户端交互——在实践中，这意味着统计应用程序是并发运行的！然而，在其当前版本中，没有保护措施来防止数据竞争，如果我们尝试在完全相同的时间插入相同的数据集多次，可能会发生数据竞争。

在本章的后面部分，我们将为统计服务器创建一个命令行客户端。此外，*第十二章*，*代码测试和性能分析*展示了如何测试你的代码。

下一个部分将展示如何为服务器应用程序构建 Docker 镜像。

## 创建 Docker 镜像

本节展示了如何将 Go 应用程序转换为 Docker 镜像——我们将使用的是一种与外部世界交互的 HTTP 服务器。在我们的案例中，它将是刚刚开发的统计网络服务。

`buildDocker`的内容，其中包含创建 Docker 镜像的步骤，如下所示：

```go
FROM golang:alpine AS builder
# Install git.
# Git is required for fetching the dependencies.
RUN apk update && apk add --no-cache git
RUN mkdir $GOPATH/src/server
ADD ./stats.go $GOPATH/src/server
ADD ./handlers.go $GOPATH/src/server
WORKDIR $GOPATH/src/server
RUN go mod init
RUN go mod tidy
RUN mkdir /pro
RUN go build -o /pro/server stats.go handlers.go
FROM alpine:latest
RUN mkdir /pro
COPY --from=builder /pro/server /pro/server
EXPOSE 1234
WORKDIR /pro
CMD ["/pro/server"] 
```

之后，我们可以使用`buildDocker`文件来构建一个名为`goapp`的 Docker 镜像，如下所示：

```go
$ docker build -f buildDocker -t goapp .
. . .
Successfully built 56d0b84b0ab5
Successfully tagged goapp:latest 
```

`docker-compose.yml`的内容，它允许我们使用 Docker 镜像，如下所示：

```go
version: "3"
services:
goapp:
**image:****goapp**
**container_name:****goapp**
restart: always
ports:
- 1234:1234
networks:
- services
networks:
services:
driver: bridge 
```

在`docker-compose.yml`文件中重要的是使用在上一步骤中创建的`goapp`镜像名称。

拥有`docker-compose.yml`，我们可以这样使用它：

```go
$ docker-compose up
[+] Running 2/0
  Network server_services  Created  0.0s
  Container goapp          Created  0.0s
Attaching to goapp
goapp  | Ready to serve at :1234
goapp  | 2023/08/31 13:32:54 Serving: /status from think:1234 
```

之后，我们可以自由地使用`curl(1)`或其他类似工具与网络服务进行交互。完成后，我们可以使用*Ctrl* + *C*来停止 Docker 镜像的运行。

这个特定网络服务的主要缺点是，一旦你禁用 Docker 镜像，所有数据都会丢失——这个问题的解决方案很简单。你可以将数据存储在外部数据库中，或者将内部 Docker 数据文件链接到本地文件系统中的文件。实现这两种解决方案超出了本章的范围。

在了解 HTTP 服务器之后，下一节将展示如何开发 HTTP 客户端。

# 开发网络客户端

本节展示了如何开发 HTTP 客户端，从简单版本开始，然后继续到更高级的版本。在这个简单版本中，所有工作都是由`http.Get()`调用完成的，当你不希望处理大量选项和参数时，这非常方便。然而，这种类型的调用给你在过程中的灵活性很小。请注意，`http.Get()`返回一个`http.Response`值。所有这些都在`simpleClient.go`中得到了说明：

```go
package main
import (
    "fmt"
"io"
"net/http"
"os"
"path/filepath"
)
func main() {
    if len(os.Args) != 2 {
        fmt.Printf("Usage: %s URL\n", filepath.Base(os.Args[0]))
        return
    } 
```

`filepath.Base()`函数返回路径的最后一个元素。当以`os.Args[0]`作为其参数时，它返回可执行二进制文件名。

```go
 URL := os.Args[1]
    data, err := http.Get(URL) 
```

在前两个语句中，我们使用`http.Get()`获取 URL 及其数据，它返回一个`*http.Response`和一个`error`变量。`*http.Response`值包含所有信息，因此你不需要对`http.Get()`进行任何额外的调用。

```go
 if err != nil {
        fmt.Println(err)
        return
    }
    _, err = io.Copy(os.Stdout, data.Body) 
```

`io.Copy()`函数从`data.Body`读取器中读取，它包含服务器响应的主体，并将数据写入`os.Stdout`。由于`os.Stdout`始终是打开的，因此你不需要为写入而打开它。因此，所有数据都写入标准输出，这通常是终端窗口：

```go
 if err != nil {
        fmt.Println(err)
        return
    }
    data.Body.Close()
} 
```

最后，我们关闭`data.Body`读取器，以便使垃圾收集工作更容易。

使用`simpleClient.go`生成以下类型的输出，在这种情况下是缩略的：

```go
$ go run simpleClient.go https://www.golang.org
<!DOCTYPE html>
<html lang="en" data-theme="auto">
<head>
<link rel="preconnect" href="https://www.googletagmanager.com">
...
</body>
</html> 
```

虽然`simpleClient.go`负责验证给定的 URL 是否存在且可访问，但它对过程没有控制权。下一小节将介绍一个高级 HTTP 客户端，该客户端处理服务器响应。

## 使用 http.NewRequest()改进客户端

上一节的网络客户端相对简单，并且没有提供任何灵活性，在本小节中，你将学习如何在不使用`http.Get()`函数的情况下读取 URL，并且拥有更多选项。然而，额外的灵活性是有代价的，因为你必须编写更多的代码。

`wwwClient.go`的代码如下：

```go
package main
import (
    "fmt"
"net/http"
"net/http/httputil"
"net/url"
"os"
"path/filepath"
"strings"
"time"
)
func main() {
    if len(os.Args) != 2 {
        fmt.Printf("Usage: %s URL\n", filepath.Base(os.Args[0]))
        return
    } 
```

虽然使用`filepath.Base()`不是必需的，但它可以使你的输出更加专业。

```go
 URL, err := url.Parse(os.Args[1])
    if err != nil {
        fmt.Println("Error in parsing:", err)
        return
    } 
```

`url.Parse()` 函数将字符串解析为 URL 结构。这意味着如果给定的参数不是一个有效的 URL，`url.Parse()` 会注意到。像往常一样，我们需要检查 `error` 变量。

```go
 c := &http.Client{
        Timeout: 15 * time.Second,
    }
    request, err := http.NewRequest(http.MethodGet, URL.String(), nil)
    if err != nil {
        fmt.Println("Get:", err)
        return
    } 
```

`http.NewRequest()` 函数在提供方法、URL 和可选主体时返回一个 `http.Request` 对象。`http.MethodGet` 参数定义了我们想使用 `GET` HTTP 方法检索数据，而 `URL.String()` 返回 `http.URL` 变量的 `string` 值。

```go
 httpData, err := c.Do(request)
    if err != nil {
        fmt.Println("Error in Do():", err)
        return
    } 
```

`http.Do()` 函数使用 `http.Client` 发送 HTTP 请求（`http.Request`）并返回一个 `http.Response`。因此，`http.Do()` 以更详细的方式完成了 `http.Get()` 的工作：

```go
 fmt.Println("Status code:", httpData.Status) 
```

`httpData.Status` 存储响应的 HTTP 状态码——这很重要，因为它允许你了解请求实际上发生了什么。

```go
 header, _ := httputil.DumpResponse(httpData, false)
    fmt.Print(string(header)) 
```

`httputil.DumpResponse()` 函数在此处用于获取服务器的响应，主要用于调试目的。`httputil.DumpResponse()` 的第二个参数是一个布尔值，用于指定函数是否在输出中包含主体——在我们的例子中，它被设置为 `false`，这意味着响应主体不会被包含在输出中，只会打印头部。如果你想在服务器端做同样的事情，你应该使用 `httputil.DumpRequest()`。

```go
 contentType := httpData.Header.Get("Content-Type")
    characterSet := strings.SplitAfter(contentType, "charset=")
    if len(characterSet) > 1 {
        fmt.Println("Character Set:", characterSet[1])
    } 
```

在这里，我们通过搜索 `Content-Type` 的值来了解响应的字符集：

```go
 if httpData.ContentLength == -1 {
        fmt.Println("ContentLength is unknown!")
    } else {
        fmt.Println("ContentLength:", httpData.ContentLength)
    } 
```

接下来，我们尝试通过读取 `httpData.ContentLength` 来获取响应的内容长度。然而，如果该值未设置，我们会打印一条相关的消息：

```go
 length := 0
var buffer [1024]byte
    r := httpData.Body
    for {
        n, err := r.Read(buffer[0:])
        if err != nil {
            fmt.Println(err)
                break
        }
        length = length + n
    }
    fmt.Println("Calculated response data length:", length)
} 
```

在程序的最后一部分，我们使用一种技术来自行发现服务器 HTTP 响应的大小。如果我们想在屏幕上显示 HTML 输出，我们可以打印 `r` 缓冲区变量的内容。

使用 `wwwClient.go` 并访问 [`www.golang.org`](https://www.golang.org) 产生以下输出：

```go
$ go run wwwClient.go https://www.golang.org
Status code: 200 OK 
```

上面的输出是 `fmt.Println("Status code:", httpData.Status)` 的输出。

接下来，我们看到 `fmt.Print(string(header))` 语句的输出，其中包含 HTTP 服务器响应的头部数据：

```go
HTTP/2.0 200 OK
Cache-Control: private
Content-Security-Policy: connect-src 'self' www.google-analytics.com stats.g.doubleclick.net ; default-src 'self' ; font-src 'self' fonts.googleapis.com fonts.gstatic.com data: ; frame-ancestors 'self' ; frame-src 'self' www.google.com feedback.googleusercontent.com www.googletagmanager.com scone-pa.clients6.google.com www.youtube.com player.vimeo.com ; img-src 'self' www.google.com www.google-analytics.com ssl.gstatic.com www.gstatic.com gstatic.com data: * ; object-src 'none' ; script-src 'self' 'sha256-n6OdwTrm52KqKm6aHYgD0TFUdMgww4a0GQlIAVrMzck=' 'sha256-4ryYrf7Y5daLOBv0CpYtyBIcJPZkRD2eBPdfqsN3r1M=' 'sha256-sVKX08+SqOmnWhiySYk3xC7RDUgKyAkmbXV2GWts4fo=' www.google.com apis.google.com www.gstatic.com gstatic.com support.google.com www.googletagmanager.com www.google-analytics.com ssl.google-analytics.com tagmanager.google.com ; style-src 'self' 'unsafe-inline' fonts.googleapis.com feedback.googleusercontent.com www.gstatic.com gstatic.com tagmanager.google.com ;
Content-Type: text/html; charset=utf-8
Date: Fri, 01 Sep 2023 19:12:13 GMT
Server: Google Frontend
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Vary: Accept-Encoding
X-Cloud-Trace-Context: 63a0ba25023e0ff4d5b5ccb87ef286bc 
```

输出的最后一部分是关于交互的字符集（`utf-8`）和响应的内容长度（`61870`），这是通过以下代码计算得出的：

```go
Character Set: utf-8
ContentLength is unknown!
EOF
Calculated response data length: 61870 
```

现在我们来看一个同时获取多个地址的技术。

## 使用 `errGroup`

在本节中，我们将使用 `errGroup` 包来通过 `golang.org/x/sync/errgroup` 外部包并发地获取多个 URL，因此 `eGroup.go` 位于 `~/go/src/github.com/mactsouk/mGo4th/ch09/eGroup`。

`eGroup.go` 的代码分为两部分。第一部分如下：

```go
package main
import (
    "fmt"
"net/http"
"os"
"golang.org/x/sync/errgroup"
)
func main() {
    if len(os.Args) == 1 {
        fmt.Println("Not enough arguments!")
        return
    }
    g := new(errgroup.Group) 
```

我们使用 `errgroup.Group` 变量，它是一组工作在相同更大任务不同部分的 goroutines。一般来说，`errgroup` 包为工作在相同任务子任务的 goroutines 提供同步、错误传播和 `Context` 取消。

`eGroup.go` 的第二部分包含以下代码：

```go
 for _, url := range os.Args[1:] {
        url := url
        g.Go(func() error {
            resp, err := http.Get(url)
            if err != nil {
                return err
            }
            defer resp.Body.Close()
            fmt.Println(url, "is OK.")
            return nil
        })
    }
    err := g.Wait()
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Everything went fine!")
} 
```

在这部分，我们使用 `g.Go()` 以 goroutine 的形式调用所需的函数。此外，我们使用闭包变量为 `url` 变量，以确保每个 goroutine 正确处理所需的 URL。

如预期的那样，我们需要先运行以下两个命令：

```go
$ go mod init
$ go mod tidy 
```

在我的 macOS 机器上运行 `eGroup.go` 生成以下输出：

```go
$ go run eGroup.go https://golang.org https://www.mtsoukalos.eu/
https://www.mtsoukalos.eu/ is OK.
https://golang.org is OK.
Everything went fine! 
```

在我的 Arch Linux 机器上运行相同的命令会产生不同的输出：

```go
$ go run eGroup.go https://golang.org https://www.mtsoukalos.eu/
https://golang.org is OK.
Error: Get "https://www.mtsoukalos.eu/": tls: failed to verify certificate: x509: certificate signed by unknown authority 
```

下一个部分将展示如何为我们之前开发的统计网络服务创建一个命令行客户端。

# 创建统计服务的客户端

在这个子节中，我们创建一个与本章早期开发的统计网络服务交互的命令行实用程序。这个统计客户端版本将使用 `cobra` 包创建，并且正如预期的那样，它将位于 `~/go/src`：`~/go/src/github.com/mactsouk/mGo4th/ch09/client`。上一个目录包含客户端的最终版本。创建客户端的初始步骤如下：

```go
$ cd ~/go/src/github.com/mactsouk/mGo4th/ch09/client
$ go mod init
$ ~/go/bin/cobra init
$ ~/go/bin/cobra add search
$ ~/go/bin/cobra add insert
$ ~/go/bin/cobra add delete
$ ~/go/bin/cobra add status
$ ~/go/bin/cobra add list 
```

因此，我们有一个包含五个命令的命令行实用程序，分别命名为 `search`、`insert`、`delete`、`status` 和 `list`。之后，我们需要实现这些命令并定义它们的本地参数，以便与统计服务器交互。

现在，让我们看看命令的实现，从 `root.go` 文件的 `init()` 函数实现开始，因为这是定义全局命令行参数的地方：

```go
func init() {
    rootCmd.PersistentFlags().StringP("server", "S", "localhost", "Server")
    rootCmd.PersistentFlags().StringP("port", "P", "1234", "Port number")
    viper.BindPFlag("server", rootCmd.PersistentFlags().Lookup("server"))
    viper.BindPFlag("port", rootCmd.PersistentFlags().Lookup("port"))
} 
```

因此，我们定义了两个全局参数，分别命名为 `server` 和 `port`，它们分别是服务器的主机名和端口号。这两个参数都有一个别名，并由 `viper` 处理。

现在，让我们检查 `status` 命令在 `status.go` 中的实现：

```go
SERVER := viper.GetString("server")
PORT := viper.GetString("port") 
```

所有命令都会读取 `server` 和 `port` 命令行参数的值以获取有关服务器的信息，`status` 命令也不例外：

```go
// Create request
URL := "http://" + SERVER + ":" + PORT + "/status" 
```

之后，我们构建请求的完整 URL。

```go
data, err := http.Get(URL)
if err != nil {
    fmt.Println(err)
    return
} 
```

然后，我们使用 `http.Get()` 向服务器发送一个 `GET` 请求。

```go
// Check HTTP Status Code
if data.StatusCode != http.StatusOK {
    fmt.Println("Status code:", data.StatusCode)
    return
} 
```

之后，我们检查请求的 HTTP 状态码以确保一切正常。

```go
// Read data
responseData, err := io.ReadAll(data.Body)
if err != nil {
    fmt.Println(err)
    return
}
fmt.Print(string(responseData)) 
```

如果一切正常，我们读取服务器响应的全部内容，它是一个字节切片，并将其作为字符串打印到屏幕上。`list` 的实现几乎与 `status` 的实现相同。唯一的区别是，实现位于 `list.go` 中，并且完整的 URL 构建如下：

```go
URL := "http://" + SERVER + ":" + PORT + "/list" 
```

之后，让我们看看 `delete` 命令在 `delete.go` 中的实现：

```go
 SERVER := viper.GetString("server")
        PORT := viper.GetString("port")
        dataset, _ := cmd.Flags().GetString("dataset")
        if dataset == "" {
            fmt.Println("Number is empty!")
            return
        } 
```

除了读取`server`和`port`全局参数的值之外，我们还读取`dataset`参数的值。如果`dataset`没有值，则命令将返回此信息。

```go
 URL := "http://" + SERVER + ":" + PORT + "/delete/" + dataset 
```

再次强调，我们在连接到服务器之前构建了请求的完整 URL。

```go
 data, err := http.Get(URL)
        if err != nil {
            fmt.Println(err)
            return
        } 
```

之前的代码将客户端请求发送到服务器。

```go
 if data.StatusCode != http.StatusOK {
            fmt.Println("Status code:", data.StatusCode)
            return
        } 
```

如果服务器响应中存在错误，`delete`命令将打印 HTTP 错误并终止。

```go
 responseData, err := io.ReadAll(data.Body)
        if err != nil {
            fmt.Println(err)
            return
        }
        fmt.Print(string(responseData)) 
```

如果一切正常，服务器响应文本将打印在屏幕上。

`delete.go`中的`init()`函数包含了定义本地`dataset`命令行参数以获取要删除的数据集名称的定义。

接下来，让我们更深入地了解`search`命令及其在`search.go`中的实现方式。实现方式与`delete`相同，除了完整的请求 URL：

```go
URL := "http://" + SERVER + ":" + PORT + "/search/" + dataset 
```

`search`命令也支持`dataset`命令行参数，用于获取要搜索的数据集名称——这是在`search.go`的`init()`函数中定义的。

最后一个展示的命令是`insert`命令，它支持两个在`insert.go`中的`init()`函数中定义的本地命令行参数：

```go
 insertCmd.Flags().StringP("dataset", "d", "", "Dataset name")
    insertCmd.Flags().StringP("values", "v", "", "List of values") 
```

这两个参数是获取所需用户输入所必需的。然而，`values`参数的值预期是一个以逗号分隔的浮点数值列表——这是我们定义如何获取数据集所有元素的方式。

`insert`命令是通过以下代码实现的：

```go
 SERVER := viper.GetString("server")
    PORT := viper.GetString("port") 
```

首先，我们读取`server`和`port`全局参数。

```go
 dataset, _ := cmd.Flags().GetString("dataset")
        if dataset == "" {
            fmt.Println("Dataset is empty!")
            return
        }
        values, _ := cmd.Flags().GetString("values")
        if values == "" {
            fmt.Println("No data!")
            return
        } 
```

然后，我们获取两个本地命令行参数的值。如果其中任何一个参数值为空，则命令将返回而不向服务器发送请求。

```go
 VALS := strings.Split(values, ",")
        vSend := ""
for _, v := range VALS {
            _, err := strconv.ParseFloat(v, 64)
            if err == nil {
                vSend = vSend + "/" + v
            }
        } 
```

之前的代码非常重要，因为它检查给定数据集元素是否为有效的`float64`值，然后创建一个形如`/value1/value2/.../valueN/`的字符串。这个字符串值附加到包含服务器请求的 URL 的末尾。

```go
 URL := "http://" + SERVER + ":" + PORT + "/insert/"
        URL = URL + "/" + dataset + "/" + vSend + "/" 
```

在这里，我们为了可读性，分两步创建服务器请求。

```go
data, err := http.Get(URL)
if err != nil {
    fmt.Println("**", err)
    return
} 
```

然后，我们将请求发送到服务器。

```go
if data.StatusCode != http.StatusOK {
    fmt.Println("Status code:", data.StatusCode)
    return
} 
```

检查 HTTP 状态码总是一个好的做法。因此，如果服务器响应一切正常，我们继续读取数据。否则，我们打印状态码，并退出。

```go
responseData, err := io.ReadAll(data.Body)
if err != nil {
    fmt.Println("*", err)
    return
}
fmt.Print(string(responseData)) 
```

在读取存储在字节切片中的服务器响应体之后，我们使用`string(responseData)`将其作为字符串打印在屏幕上。

客户端应用程序生成以下类型的输出：

```go
$ go run main.go list
List of entries:
d6    4    2.325000    1.080220
d4    5    2.860000    1.441666
d2    9    1.000000    0.000000
v1    5    3.000000    1.414214
v2    7    1.000000    3.422614 
```

这是`list`命令的输出。

```go
$ go run main.go status
Total entries: 5 
```

`status`命令的输出告诉我们应用程序中的条目数量。

```go
$ go run main.go search -d v1
v1 5 3.000000 1.414214 
```

之前的输出显示了在成功找到数据集时使用`search`命令的情况。

```go
$ go run main.go search -d notThere
Status code: 404 
```

之前的输出显示了在找不到数据集时使用`search`命令的情况。

```go
$ go run main.go delete -d v1
v1 deleted! 
```

这是`delete`命令的输出。

```go
$ go run main.go insert -d n1 -v 1,2,3,-4,0,0
New record added successfully 
```

这是`insert`命令的操作。如果你尝试多次插入相同的数据集名称，服务器输出将是`状态码：304`。

下一个部分解释了如何超时 HTTP 连接。

# 超时 HTTP 连接

本节介绍了处理耗时过长的 HTTP 连接超时的技术，这些技术可以在服务器端或客户端工作。

## 使用 SetDeadline()

`SetDeadline()`函数由`net`用于设置网络连接的读写截止时间。由于`SetDeadline()`的工作方式，你需要在任何读写操作之前调用`SetDeadline()`。请注意，Go 使用截止时间来实现超时，因此你不需要在每次应用程序接收或发送数据时重置超时。

`SetDeadline()`的使用在`withDeadline.go`中得到了说明，特别是在`Timeout()`函数的实现中：

```go
var timeout = time.Duration(time.Second)
func Timeout(network, host string) (net.Conn, error) {
    conn, err := net.DialTimeout(network, host, timeout)
    if err != nil {
        return nil, err
    }
    conn.SetDeadline(time.Now().Add(timeout))
    return conn, nil
} 
```

`timeout`全局变量定义了在`SetDeadline()`调用中使用的超时时间。前面的函数在`main()`中的以下代码中使用：

```go
t := http.Transport{
    Dial: Timeout,
}
client := http.Client{
        Transport: &t,
} 
```

因此，`http.Transport`在`Dial`字段中使用`Timeout()`，而`http.Client`使用`http.Transport`。当你调用`client.Get()`方法并传入所需的 URL（此处未显示）时，由于`http.Transport`的定义，`Timeout`会被自动使用。所以，如果`Timeout`函数在收到服务器响应之前返回，我们就遇到了超时。

使用`withdeadline.go`会产生以下类型的输出：

```go
$ go run withDeadline.go http://www.golang.org
Timeout value: 1s
<!DOCTYPE html>
... 
```

调用成功，并在不到 1 秒内完成，所以没有超时。

```go
$ go run withDeadline.go http://localhost:80
Timeout value: 1s
Get "http://localhost:80": read tcp 127.0.0.1:52492->127.0.0.1:80: i/o timeout 
```

这次我们遇到了超时，因为服务器响应时间过长。

接下来，我们展示如何使用`context`包超时一个连接。

## 在客户端设置超时时间

本节介绍了一种在客户端超时耗时过长的网络连接的技术。因此，如果客户端在期望的时间内没有从服务器收到响应，它将关闭连接。`timeoutClient.go`源文件展示了这一技术。

```go
package main
import (
    "context"
"fmt"
"io"
"net/http"
"os"
"strconv"
"time"
)
var delay int = 5 
```

在前面的代码中，我们定义了一个名为`delay`的全局变量，用于存储延迟值。

```go
func main() {
    if len(os.Args) == 1 {
        fmt.Println("Need a URL and a delay!")
        os.Exit(1)
    }
    url := os.Args[1]
    if len(os.Args) == 3 {
      t, err := strconv.Atoi(os.Args[2])
      if err != nil {
         fmt.Println(err)
         return
      }
      delay = t
   }
    fmt.Println("Delay:", delay) 
```

由于 URL 已经是字符串值，因此直接读取，而延迟期则使用`strconv.Atoi()`转换为数值。

`main()`函数的其余实现如下：

```go
 ctx, cncl := context.WithTimeout(context.Background(), time.Second * time.Duration(delay))
    defer cncl()
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        fmt.Println(err)
        return
    }
    res, err := http.DefaultClient.Do(req.WithContext(ctx))
    if err != nil {
        fmt.Println(err)
        return
    }
    defer res.Body.Close()
    body, err := io.ReadAll(res.Body)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(string(body))
} 
```

首先，我们初始化`ctx`上下文，然后使用`http.NewRequestWithContext()`将此上下文与 HTTP 请求关联。如果超时时间超过，使用`WithTimeout()`创建的`context.Context`将会过期。

与`timeoutClient.go`一起工作并产生超时情况时，会生成以下类型的输出：

```go
$ go run timeoutClient.go http://localhost:1234 5
Delay: 5
Get "http://localhost:1234": context deadline exceeded 
```

下一个子部分展示了如何在服务器端超时一个 HTTP 请求。

## 在服务器端设置超时时间

本节介绍了一种超时网络连接的技术，这些连接在服务器端完成时间过长。这比客户端更重要，因为拥有太多打开连接的服务器可能无法处理额外的请求，除非一些已经打开的连接关闭。这通常有两个原因。第一个原因是软件错误，第二个原因是当服务器遭受 **拒绝服务**（**DoS**）攻击时！

`timeoutServer.go` 中的 `main()` 函数展示了这项技术：

```go
func main() {
    PORT := ":8001"
    arguments := os.Args
    if len(arguments) != 1 {
        PORT = ":" + arguments[1]
    }
    fmt.Println("Using port number: ", PORT)
    m := http.NewServeMux()
    srv := &http.Server{
        Addr:         PORT,
        Handler:      m,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
    } 
```

这就是定义超时时间的地方。请注意，您可以定义读取和写入过程的超时时间。`ReadTimeout` 字段的值指定了读取整个客户端请求（包括主体）允许的最大持续时间，而 `WriteTimeout` 字段的值指定了在超时发送客户端响应之前允许的最大时间。

```go
 m.HandleFunc("/time", timeHandler)
    m.HandleFunc("/", myHandler)
    err := srv.ListenAndServe()
    if err != nil {
        fmt.Println(err)
        return
    }
} 
```

除了 `http.Server` 定义中的参数外，其余代码与往常一样：它包含处理函数并调用 `ListenAndServe()` 来启动 HTTP 服务器。

使用 `timeoutServer.go` 不会生成任何输出。然而，如果客户端连接到它而没有发送任何请求，客户端连接将在 3 秒后结束。如果客户端接收服务器响应的时间超过 3 秒，也会发生同样的事情。

# 摘要

在本章中，我们学习了如何使用 HTTP，如何从 Go 代码创建 Docker 镜像，以及如何开发 HTTP 客户端和服务器。我们还把统计应用程序转换成了 Web 应用程序，并为它编写了一个命令行客户端。此外，我们还学习了如何超时 HTTP 连接。

我们现在已准备好开始开发强大且并发的 HTTP 应用程序——然而，我们还没有完成 HTTP 的学习。*第十一章*，*与 REST API 一起工作*，将连接这些点，并展示如何开发强大的 RESTful 服务器和客户端。

但首先，我们需要了解如何使用 TCP/IP、TCP、UDP 和 WebSocket，这些是下一章的主题。

# 练习

+   修改 `wwwClient.go` 以将 HTML 输出保存到外部文件。

+   在统计应用程序中使用 `sync.Mutex` 以避免竞争条件。

+   使用 goroutines 和 channels 实现 `ab(1)` 的简单版本。`ab(1)` 是 Apache HTTP 服务器基准测试工具。

# 其他资源

+   Caddy 服务器：[`caddyserver.com/`](https://caddyserver.com/)

+   Nginx 服务器：[`nginx.org/en/`](https://nginx.org/en/)

+   `net/http` 包：[`pkg.go.dev/net/http`](https://pkg.go.dev/net/http)

+   官方 Docker Go 镜像：[`hub.docker.com/_/golang/`](https://hub.docker.com/_/golang/ )

# 加入我们的 Discord 社区

加入我们的社区 Discord 空间，与作者和其他读者进行讨论：

[`discord.gg/FzuQbc8zd6`](https://discord.gg/FzuQbc8zd6 )

![](https://discord.gg/FzuQbc8zd6 )
