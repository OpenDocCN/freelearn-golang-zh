# 第一章：微服务简介

首先，我们将看看如何使用`net/http`包轻松创建一个具有单个端点的简单 Web 服务器。然后，我们将检查`encoding/json`包，看看 Go 如何使我们能够轻松地使用 JSON 对象进行请求和响应。最后，我们将查看路由和处理器的工作方式以及我们如何在这些处理器之间管理上下文。

# 使用 net/http 构建简单 Web 服务器

`net/http`包提供了我们编写 HTTP 客户端和服务器所需的所有功能。它使我们能够向使用 HTTP 协议的其他服务器发送请求，以及运行可以将请求路由到单独的 Go 函数、提供静态文件以及更多功能的 HTTP 服务器。

首先，我们应该提出一个问题，*没有简单 hello world 示例的技术书会完整吗？* 我认为没有，这正是我们将开始的地方。

在这个例子中，我们将创建一个具有单个端点的 HTTP 服务器，该端点返回由 JSON 标准表示的静态文本，这将介绍 HTTP 服务器和处理器的基本功能。然后，我们将修改此端点以接受编码为 JSON 的请求，并使用`encoding/json`包向客户端返回响应。我们还将通过添加第二个端点返回简单图像来检查路由的工作方式。

到本章结束时，你将基本掌握基本包及其如何快速高效地构建简单微服务的方法。

由于标准库中包含的 HTTP 包，使用 Go 构建 Web 服务器变得极其简单。

它拥有你管理路由、处理**传输层安全性**（**TLS**）（我们将在第八章中介绍），支持 HTTP/2 开箱即用，以及运行一个处理大量请求的非常高效的服务器的所有功能。

本章的源代码可以在 GitHub 上找到，网址为[`github.com/building-microservices-with-go/chapter1.git`](http://github.com/building-microservices-with-go/chapter1.git)，本章节和随后的章节将广泛引用这些示例，所以如果你还没有这样做，请在继续之前去克隆这个仓库。

让我们看看创建基本服务器的语法，然后我们可以更深入地了解这些包：

示例 1.0 `basic_http_example/basic_http_example.go`

```go
09 func main() { 
10  port := 8080 
11 
12  http.HandleFunc("/helloworld", helloWorldHandler) 
13 
14  log.Printf("Server starting on port %v\n", 8080) 
15  log.Fatal(http.ListenAndServe(fmt.Sprintf(":%v", port), nil)) 
16 } 
17 
18 func helloWorldHandler(w http.ResponseWriter, r *http.Request) { 
19   fmt.Fprint(w, "Hello World\n") 
20 } 

```

我们首先做的事情是在`http`包上调用`HandleFunc`方法。`HandleFunc`方法在`DefaultServeMux`处理器上创建一个`Handler`类型，将第一个参数中传入的路径映射到第二个参数中的函数：

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) 

```

在第**15**行，我们启动 HTTP 服务器，`ListenAndServe`接受两个参数，将服务器绑定到的 TCP 网络地址和用于路由请求的处理程序：

```go
func ListenAndServe(addr string, handler Handler) error 

```

在我们的例子中，我们传递了网络地址 `:8080"`，这意味着我们希望将服务器绑定到所有可用的 IP 地址上的端口 `8080`。

我们传递的第二个参数是 `nil`，这是因为我们正在使用 `DefaultServeMux` 处理器，它是通过我们的 `http.HandleFunc` 调用来设置的。在 第三章 *介绍 Docker* 中，你将看到当我们介绍更复杂的路由器时使用这个第二个参数，但现在我们可以忽略它。

如果 `ListenAndServe` 函数无法启动服务器，它将返回一个错误，最常见的原因可能是你正在尝试绑定服务器上已经使用的端口。在我们的例子中，我们将 `ListenAndServe` 的输出直接传递给 `log.Fatal(error)`，这是一个方便的函数，相当于调用 `fmt.Print(a ...interface{})` 然后调用 `os.Exit(1)`。由于 `ListenAndServe` 如果服务器启动正确则会阻塞，所以我们永远不会在成功启动时退出。

让我们快速运行并测试我们的新服务器：

```go
$ go run ./basic_http_example.go  

```

现在，你应该看到应用程序的输出：

```go
2016/07/30 01:08:21 Server starting on port 8080  

```

如果你没有看到前面的输出，而是看到以下内容？

```go
2016/07/19 03:51:11 listen tcp :8080: bind: address already in use exit status 1  

```

再看看 `ListenAndServe` 的签名和我们调用它的方式。还记得我们之前说的为什么我们使用 `log.Fatal` 吗？

如果你确实收到这个错误消息，这意味着你的电脑上已经运行了一个使用端口 `8080` 的应用程序，这可能是你程序的另一个实例，也可能是另一个应用程序。你可以通过检查正在运行的过程来确认没有其他实例在运行：

```go
$ ps -aux | grep 'go run'  

```

如果你看到另一个 `go run ./basic_http_example.go`，你可以简单地将其终止并重试。如果你没有其他实例在运行，那么你可能有一些其他软件绑定到了这个端口。尝试在行 **10** 上更改端口并重新启动你的程序。

要测试服务器，打开一个新的浏览器，输入 URI `http://127.0.0.1:8080/helloworld`，如果一切正常，你应该从服务器看到以下响应：

```go
Hello World 

```

恭喜你，这是进入微服务大师的第一步。现在我们有了第一个运行起来的程序，让我们更仔细地看看我们如何返回和接受 JSON。

# 读取和写入 JSON

感谢内置在标准库中的 `encoding/json` 包，它使得将 JSON 编码和解码为 Go 类型既快速又简单。它实现了简单的 `Marshal` 和 `Unmarshal` 函数；然而，如果需要的话，该包还提供了 `Encoder` 和 `Decoder` 类型，这让我们在读取和写入 JSON 数据流时有了更大的控制权。在本节中，我们将探讨这两种方法，但首先让我们看看将标准的 Go `struct` 转换为其对应的 JSON 字符串是多么简单。

# 将 Go 结构体序列化为 JSON

要编码 JSON 数据，`encoding/json` 包提供了 `Marshal` 函数，其签名如下：

```go
func Marshal(v interface{}) ([]byte, error) 

```

这个函数接受一个参数，其类型为 `interface`，因此在 Go 中，几乎任何你能想到的对象都可以，因为 `interface` 代表了任何类型。它返回一个元组 `([]byte, error)`，你会在 Go 中经常看到这种返回风格，一些语言实现了 try catch 方法，鼓励在操作无法执行时抛出错误，Go 建议使用 `(return type, error)` 模式，其中错误为 `nil` 表示操作成功。

在 Go 中，未处理的错误是坏事，尽管该语言实现了 `Panic` 和 `Recover`，这与其他语言的异常处理类似，但你应该使用这些函数的情况是相当不同的（参见 *The Go Programming Language*，Kernaghan）。在 Go 中，`panic` 函数会导致正常执行停止，并且执行 Go 线程中所有延迟调用的函数，然后程序会崩溃并显示日志消息。它通常用于指示代码中存在错误的意外错误，良好的健壮的 Go 代码将尝试处理这些运行时异常，并将详细的 `error` 对象返回给调用函数。

这种模式正是使用 `Marshal` 函数实现的。在 `Marshal` 无法从给定的对象创建 JSON 编码的字节数组的情况下，这可能是因为运行时崩溃，那么这个错误会被捕获，并返回一个详细说明问题的错误对象给调用者。

让我们尝试一下，基于我们现有的示例进行扩展，而不是简单地从我们的处理器中打印一个字符串，让我们创建一个简单的 `struct` 用于响应，并返回这个 `struct`。

示例 1.1 `reading_writing_json_1/reading_writing_json_1.go`

```go
10 type helloWorldResponse struct { 
11    Message string 
12 } 

```

在我们的处理器中，我们将创建这个对象的实例，设置消息，然后使用 `Marshal` 函数将其编码为字符串后再返回。

让我们看看它会是什么样子：

```go
23 func helloWorldHandler(w http.ResponseWriter, r *http.Request) { 
24   response := helloWorldResponse{Message: "HelloWorld"} 
25   data, err := json.Marshal(response) 
26   if err != nil { 
27     panic("Ooops") 
28   } 
29  
30   fmt.Fprint(w, string(data)) 
31 } 

```

现在，当我们再次运行我们的程序并刷新浏览器时，我们会看到以下输出以有效的 JSON 格式渲染：

```go
{"Message":"Hello World"} 

```

这很棒；然而，`Marshal` 的默认行为是取字段的字面名称并将其用作 JSON 输出中的字段。如果我想使用驼峰命名法，并希望看到 "message"，我们能否在 `helloWorldResponse` `struct` 中重命名字段？

不幸的是，我们不能这样做，因为在 Go 中，小写属性不是导出的，`Marshal` 会忽略这些属性，并且不会将它们包含在输出中。

虽然如此，`encoding/json` 包实现了 `struct` 字段属性，允许我们将属性输出更改为我们选择的任何内容。

示例 1.2 `reading_writing_json_2/reading_writing_json_2.go`

```go
10 type helloWorldResponse struct { 
11   Message string `json:"message"` 
12 } 

```

使用 `struct` 字段的标签，我们可以更好地控制输出格式。在前面的示例中，当我们对 `struct` 进行编码时，服务器的输出会是：

```go
{"message":"Hello World"} 

```

这正是我们想要的，但我们可以使用字段标签来进一步控制输出。我们可以转换对象类型，甚至如果我们需要的话，可以完全忽略一个字段：

```go
type helloWorldResponse struct {
// change the output field to be "message" 
   Message   string `json:"message"` 
   // do not output this field 
   Author  string `json:"-"` 
   // do not output the field if the value is empty 
   Date    string `json:",omitempty"` 
   // convert output to a string and rename "id" 
   Id    int    `json:"id, string"` 
} 

```

通道、复杂数据类型和函数不能被编码成 JSON；尝试编码这些类型将会导致`Marshal`函数返回一个`UnsupportedTypeError`。

它也不能表示循环数据结构；如果你的`struct`包含循环引用，那么`Marshal`将导致无限递归，这对网络请求来说绝不是好事。

如果我们想要以缩进格式导出我们的 JSON，我们可以使用`MarshallIndent`函数，这允许你传递一个额外的`string`参数来指定你想要的缩进。右对齐两个空格，不是制表符吗？

```go
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error) 

```

精明的读者可能已经注意到，我们在将我们的`struct`解码成字节数组，然后将它写入响应流中，这似乎并不特别高效，实际上确实如此。Go 提供了`Encoders`和`Decoders`，它们可以直接写入流，因为我们已经有了带有`ResponseWriter`的流，那么我们就这么做吧。

在我们这么做之前，我认为我们需要稍微看看一下`ResponseWriter`，看看那里发生了什么。

`ResponseWriter`是一个定义了三个方法的接口：

```go
// Returns the map of headers which will be sent by the 
// WriteHeader method. 
Header() 

// Writes the data to the connection. If WriteHeader has not 
// already been called then Write will call 
// WriteHeader(http.StatusOK). 
Write([]byte) (int, error) 

// Sends an HTTP response header with the status code. 
WriteHeader(int) 

```

如果我们有一个`ResponseWriter`接口，我们如何使用它与`fmt.Fprint(w io.Writer, a ...interface{})`方法一起使用？这个方法需要一个`Writer`接口作为参数，而我们有一个`ResponseWriter`接口。如果我们查看`Writer`的签名，我们可以看到它是：

```go
Write(p []byte) (n int, err error) 

```

因为`ResponseWriter`接口实现了这个方法，所以它也满足了`Writer`接口，因此任何实现了`ResponseWriter`的对象都可以传递给任何期望`Writer`的函数。

太棒了，Go 真是强大，但我们还没有回答我们的问题，*有没有更好的方法在返回之前不将数据序列化到一个临时对象中，直接发送到输出流？*

`encoding/json`包有一个名为`NewEncoder`的函数，这个函数返回一个`Encoder`对象，可以用来将 JSON 直接写入一个打开的写入器，猜猜看；我们就有这样一个：

```go
func NewEncoder(w io.Writer) *Encoder 

```

因此，我们不需要将`Marshal`的输出存储到字节数组中，我们可以直接将其写入 HTTP 响应。

示例 1.3 `reading_writing_json_3/reading_writing_json_3.go`：

```go
func helloWorldHandler(w http.ResponseWriter, r *http.Request) { 
    response := HelloWorldResponse{Message: "HelloWorld"} 
    encoder := json.NewEncoder(w) 
    encoder.Encode(&response) 
}                

```

我们将在后面的章节中讨论基准测试，但为了了解为什么这很重要，我们创建了一个简单的基准测试来检查这两种方法之间的差异，看看输出结果。

示例 1.4 `reading_writing_json_2/reading_writing_json_2.go`：

```go
$go test -v -run="none" -bench=. -benchtime="5s" -benchmem  BenchmarkHelloHandlerVariable-8  20000000  511 ns/op  248 B/op  5 allocs/op BenchmarkHelloHandlerEncoder-8  20000000  328 ns/op   24 B/op  2 allocs/op BenchmarkHelloHandlerEncoderReference-8  20000000  304 ns/op  8 B/op  1 allocs/op PASS ok  github.com/building-microservices-with-go/chapter1/reading_writing_json_2  24.109s 

```

使用 `Encoder` 而不是将数据序列化到字节数组中几乎快了 50%。我们在这里处理的是纳秒，所以时间可能看起来无关紧要，但事实并非如此；这仅仅是两行代码。如果你在代码的其他部分也有这种低效的情况，那么你的应用程序将运行得更慢，你需要更多的硬件来满足负载，这将花费你金钱。这两种方法之间的差异并没有什么巧妙之处，我们所做的只是理解标准包的工作原理，并为我们自己的需求选择了正确的选项，这并不是性能调优，这是理解框架。

# 将 JSON 解码到 Go 结构体

现在我们已经学会了如何将 JSON 发送回客户端，如果我们需要在返回输出之前读取输入怎么办？我们可以使用 URL 参数，我们将在下一章中看到这一点，但通常你将需要更复杂的数据结构，这些结构涉及服务以接受 JSON 作为 HTTP `POST` 请求的一部分。

将我们在上一节中学到的类似技术应用于编写 JSON，读取 JSON 也很容易。要将 JSON 解码到 `struct` 字段，`encoding/json` 包为我们提供了 `Unmarshal` 函数：

```go
func Unmarshal(data []byte, v interface{}) error 

```

`Unmarshal` 函数与 `Marshal` 函数的作用相反；它根据需要分配映射、切片和指针。传入的对象键使用 `struct` 字段名或其标签进行匹配，并将进行不区分大小写的匹配；然而，精确匹配是首选。与 `Marshal` 类似，`Unmarshal` 只会设置导出的 `struct` 字段，即以大写字母开头的字段。

我们首先添加一个新的 `struct` 字段来表示请求，而 `Unmarshal` 可以将 JSON 解码到 `interface{}` 中，这将是一个 `map[string]interface{} // 用于 JSON 对象类型` 或 `[]interface{} // 用于 JSON 数组`，具体取决于我们的 JSON 是对象还是数组。

在我看来，如果我们明确地说明我们期望的请求内容，这将使我们的代码对读者来说更加清晰。我们还可以通过在需要使用数据时不必手动转换数据来节省我们的工作。

记住两点：

+   你不是为编译器编写代码，你是为人类编写代码以便理解

+   你阅读代码的时间将比编写代码的时间多

考虑到这两点，我们创建了一个简单的 `struct` 来表示我们的请求，它看起来是这样的：

示例 1.5 `reading_writing_json_4/reading_writing_json_4.go`:

```go
14 type helloWorldRequest struct { 
15   Name string `json:"name"` 
16 } 

```

再次，我们将使用 `struct` 字段标签，因为我们可以让 `Unmarshal` 进行不区分大小写的匹配，所以 `{"name": "World}` 会正确地解码到 `struct` 中，就像 `{"Name": "World"}` 一样，当我们指定一个标签时，我们是在明确请求的形式，这是好事。在速度和性能方面，它也大约快了 10%，记住，性能很重要。

要访问与请求一起发送的 JSON，我们需要查看传递给我们的处理器的 `http.Request` 对象。以下列表并没有显示请求上的所有方法，只是我们立即要处理的方法，对于完整的文档，我建议查看 [`godoc.org/net/http#Request`](https://godoc.org/net/http#Request) 上的文档：

```go
type Requests struct { 
... 
  // Method specifies the HTTP method (GET, POST, PUT, etc.). 
  Method string 

// Header contains the request header fields received by the server. The type Header is a link to map[string] []string.  
Header Header 

// Body is the request's body. 
Body io.ReadCloser 
... 
} 

```

请求中发送的 JSON 数据可以在 `Body` 字段中访问。`Body` 实现了 `io.ReadCloser` 接口作为流，并且不会返回 `[]byte` 或 `string`。如果我们需要获取正文中的数据，我们可以简单地将其读取到一个字节数组中，如下面的示例所示：

```go
30 body, err := ioutil.ReadAll(r.Body) 
31 if err != nil { 
32     http.Error(w, "Bad request", http.StatusBadRequest) 
33     return   
34 } 

```

这里有一些我们需要记住的事情。我们没有调用 `Body.Close()`，如果我们用客户端进行调用，我们就需要这样做，因为它是不会自动关闭的；然而，当在 `ServeHTTP` 处理器中使用时，服务器会自动关闭请求流。

为了了解所有这些是如何在我们的处理器中工作的，我们可以查看以下处理器：

```go
28 func helloWorldHandler(w http.ResponseWriter, r *http.Request) { 
29  
30   body, err := ioutil.ReadAll(r.Body) 
31   if err != nil { 
32     http.Error(w, "Bad request", http.StatusBadRequest) 
33             return 
34   } 
35  
36   var request helloWorldRequest 
37   err = json.Unmarshal(body, &request) 
38   if err != nil { 
39     http.Error(w, "Bad request", http.StatusBadRequest) 
40             return 
41   } 
42  
43  response := helloWorldResponse{Message: "Hello " + request.Name} 
44  
45   encoder := json.NewEncoder(w) 
46   encoder.Encode(response) 
47 } 

```

让我们运行这个示例，看看它是如何工作的。为了测试，我们可以简单地使用 `curl` 命令向运行中的服务器发送请求。如果你觉得使用 GUI 工具（如 Postman，它适用于 Google Chrome 浏览器）比使用 Postman 更舒服，它们也可以正常工作，或者你可以自由使用你喜欢的工具：

```go
$ curl localhost:8080/helloworld -d '{"name":"Nic"}'  

```

你应该看到以下响应：

```go
{"message":"Hello Nic"} 

```

如果你在请求中不包括正文，你认为会发生什么？

```go
$ curl localhost:8080/helloworld  

```

如果你猜对了，你会得到 `HTTP 状态 400 错误请求`，那么你就能赢得奖品：

```go
func Error(w ResponseWriter, error string, code int) 

```

错误会以给定的消息和状态码回复请求。一旦我们发送了这些，我们就需要返回停止函数的进一步执行，因为这个操作不会自动关闭 `ResponseWriter` 接口并返回到调用函数。

在你认为你已经完成之前，尝试一下，看看你是否能提高处理器的性能。想想我们在序列化 JSON 时讨论的事情。

明白了？

好吧，如果不是这样，这里就是答案，我们只是在使用 `Decoder`，它是我们在写入 JSON 时使用的 `Encoder` 的相反。它有即时 33% 的性能提升，代码也更少。

示例 1.6 `reading_writing_json_5/reading_writing_json_5.go`：

```go
27 func helloWorldHandler(w http.ResponseWriter, r *http.Request) { 
28 
29   var request HelloWorldRequest 
30   decoder := json.NewDecoder(r.Body) 
31  
32   err := decoder.Decode(&request) 
33   if err != nil { 
34     http.Error(w, "Bad request", http.StatusBadRequest) 
35             return 
36   } 
37  
38   response := HelloWorldResponse{Message: "Hello " + request.Name} 
39  
40   encoder := json.NewEncoder(w) 
41   encoder.Encode(response) 
42 } 

```

现在我们可以看到使用 Go 编码和解码 JSON 是多么简单，我建议现在花五分钟时间花些时间研究 `encoding/json` 包的文档 ([`golang.org/pkg/encoding/json/`](https://golang.org/pkg/encoding/json/))，因为你可以用这个包做很多事情。

# 在 `net/http` 中的路由

即使是一个简单的微服务也需要能够根据请求的路径或方法将请求路由到不同的处理程序。在 Go 中，这由 `DefaultServeMux` 方法处理，它是一个 `ServerMux` 实例。在本章的早期，我们简要介绍了当将 nil 传递给 `ListenAndServe` 函数的处理程序参数时，将使用 `DefaultServeMux` 方法。当我们调用 `http.HandleFunc("/helloworld", helloWorldHandler)` 包函数时，我们实际上只是间接调用 `http.DefaultServerMux.HandleFunc(…)`。

Go HTTP 服务器没有特定的路由器，而是将实现 `http.Handler` 接口的对象作为顶级函数传递给 `Listen()` 函数。当请求进入服务器时，此处理程序的 `ServeHTTP` 方法被调用，并负责执行或委托任何工作。为了便于处理多个路由，HTTP 包有一个特殊对象 `ServerMux`，它实现了 `http.Handler` 接口。

向 `ServerMux` 处理程序添加处理程序有两个函数：

```go
func HandlerFunc(pattern string, handler func(ResponseWriter, *Request)) 
func Handle(pattern string, handler Handler) 

```

`HandleFunc` 函数是一个便利函数，它创建了一个处理程序，该处理程序的 `ServeHTTP` 方法会调用一个带有 `func(ResponseWriter, *Request)` 签名的普通函数，该函数作为参数传递。

`Handle` 函数要求你传递两个参数，你想要注册的处理程序的模式以及实现 `Handler` 接口的对象：

```go
type Handler interface { 
  ServeHTTP(ResponseWriter, *Request) 
} 

```

# 路径

我们已经解释了 `ServeMux` 负责将传入请求路由到已注册的处理程序的方式，然而路由匹配的方式可能相当令人困惑。`ServeMux` 处理程序有一个非常简单的路由模型，它不支持通配符或正则表达式，使用 `ServeMux` 时，你必须明确注册的路径。

你可以注册固定根路径，例如 `/images/cat.jpg`，或者根子树，例如 `/images/`。根子树中的尾部斜杠很重要，因为任何以 `/images/` 开头的请求，例如 `/images/happy_cat.jpg`，都将被路由到与 `/images/` 关联的处理程序。

如果我们将路径 `/images/` 注册到处理程序 foo，并且用户向我们的服务发送到 `/images` 的请求（注意没有尾部斜杠），那么 `ServerMux` 将将请求转发到 `/images/` 处理程序，并附加一个尾部斜杠。

如果我们还将路径 `/images`（注意没有尾部斜杠）注册到处理程序 bar，并且用户请求 `/images`，则此请求将被导向 bar；然而，`/images/` 或 `/images/cat.jpg` 将被导向 foo：

```go
http.Handle("/images/", newFooHandler())
http.Handle("/images/persian/", newBarHandler())
http.Handle("/images", newBuzzHandler())
/images                  => Buzz
/images/                 => Foo
/images/cat              => Foo
/images/cat.jpg          => Foo
/images/persian/cat.jpg  => Bar

```

较长的路径始终优先于较短的路径，因此可以有一个显式路由指向不同的处理程序，以捕获所有路由。

我们还可以指定主机名，例如可以注册路径 `search.google.com/`，那么 `/ServerMux` 将将任何请求转发到 `http://search.google.com` 和 `http://www.google.com`，并转发到它们各自的处理程序。

如果你习惯了基于框架的应用程序开发方法，例如使用 Ruby on Rails 或 ExpressJS，你可能会觉得这个路由器非常简单，确实如此，记住我们不是使用框架，而是 Go 的标准包，我们的目的是始终提供一个可以在此基础上构建的基础。在非常简单的情况下，`ServeMux` 方法已经足够好了，事实上我个人根本不使用其他任何东西。然而，每个人的需求都是不同的，标准包的美丽和简洁性使得构建自己的路由变得极其简单，因为所需的所有东西只是一个实现了 `Handler` 接口的对象。快速在谷歌上搜索会找到一些非常好的第三方路由器，但我的建议是，在决定选择第三方包之前，首先学习 `ServeMux` 的限制，这将大大有助于你的决策过程，因为你将知道你试图解决的问题。

# 方便处理程序

`net/http` 包实现了几个创建不同类型方便处理程序的方法，让我们来检查这些。

# FileServer

`FileServer` 函数返回一个处理程序，它使用文件系统的内容来服务 HTTP 请求。这可以用来服务静态文件，如图像或其他存储在文件系统上的内容：

```go
func FileServer(root FileSystem) Handler 

```

看看下面的代码：

```go
http.Handle("/images", http.FileServer(http.Dir("./images")))

```

这允许我们将文件系统路径 `./images` 的内容映射到服务器路由 `/images`，`Dir` 实现了一个仅限于特定目录树的文件系统，`FileServer` 方法使用它来提供资产服务。

# NotFoundHandler

`NotFoundHandler` 函数返回一个简单的请求处理程序，对每个请求回复一个 `404 页面未找到` 的回复：

```go
func NotFoundHandler() Handler 

```

# RedirectHandler

`RedirectHandler` 函数返回一个请求处理程序，它将接收到的每个请求重定向到给定的 URI，并使用给定的状态码。提供的代码应在 3xx 范围内，通常是 `StatusMovedPermanently`、`StatusFound` 或 `StatusSeeOther`：

```go
func RedirectHandler(url string, code int) Handler 

```

# StripPrefix

`StripPrefix` 函数返回一个处理程序，它通过从请求 URL 的路径中移除给定的前缀来服务 HTTP 请求，然后调用 `h` 处理程序。如果路径不存在，则 `StripPrefix` 将回复一个 HTTP 404 未找到错误：

```go
func StripPrefix(prefix string, h Handler) Handler 

```

# TimeoutHandler

`TimeoutHandler` 函数返回一个 `Handler` 接口，它以给定的时间限制运行 `h`。当我们研究第六章（74445ff8-eb01-4a2f-a910-0551e7d85a5f.xhtml）中常见的模式，即 *微服务框架* 时，我们将看到这如何有助于避免服务中的级联故障：

```go
func TimeoutHandler(h Handler, dt time.Duration, msg string) Handler 

```

新的处理程序调用 `h.ServeHTTP` 来处理每个请求，但如果调用运行时间超过了其时间限制，处理程序将回复一个包含给定消息 `(msg)` 的 `503 服务不可用` 响应：

最后两个处理器特别有趣，因为它们实际上是在链式处理。这是一个我们将在后面的章节中更深入探讨的技术，它允许你练习编写干净的代码，同时也允许你保持代码的 DRY（Don't Repeat Yourself）。

我可能直接从 Go 文档中提取了这些处理器的多数描述，你可能已经阅读过这些内容，因为你已经阅读过文档，对吧？在 Go 中，文档非常出色，并且强烈鼓励（甚至强制）为你的包编写文档，如果你使用标准包中包含的`golint`命令，那么它将报告你的代码中不符合标准的部分。我强烈建议在使用某个包时花点时间浏览标准文档，这不仅可以帮助你了解正确的用法，你可能会发现更好的方法。你肯定会接触到良好的实践和风格，甚至可能在你继续工作的那一天（Stack Overflow 停止工作，整个行业停滞不前）有所帮助。

# 静态文件处理器

虽然在这本书中我们主要会处理 API，但通过添加一个二级端点来观察默认路由器和路径的工作方式是有益的说明。

作为一个小练习，尝试修改`reading_writing_json_5/reading_writing_json_5.go`中的代码，添加一个端点`/cat`，它返回 URI 中指定的猫的图片。为了给你一点提示，你需要使用`net/http`包中的`FileServer`函数，你的 URI 将类似于`http://localhost:8080/cat/cat.jpg`。

第一次尝试就成功了，还是你忘记添加`StripPrefix`处理器了？

示例 1.7 `reading_writing_json_6/reading_writing_json_6.go`:

```go
21 cathandler := http.FileServer(http.Dir("./images")) 
22 http.Handle("/cat/", http.StripPrefix("/cat/", cathandler)) 

```

在前面的示例中，我们向路径`/cat/`注册了一个`StripPrefix`处理器。如果我们没有这样做，那么`FileServer`处理器就会在我们的`images/cat`目录中寻找我们的图片。也值得提醒一下关于`/cat`和`/cat/`作为路径的区别。如果我们注册我们的路径为`/cat`，那么我们不会匹配`/cat/cat.jpg`。如果我们注册我们的路径为`/cat/`，我们将匹配`/cat`和`/cat/whatever`。

# 创建处理器

现在我们将通过展示如何创建`Handler`而不是仅仅使用`HandleFunc`来结束这里的示例。我们将把为我们的`helloworld`端点执行请求验证的代码和返回响应的代码分别放入单独的处理程序中，以说明如何链式处理。

示例 1.8 `chapter1/reading_writing_json_7.go`:

```go
31 type validationHandler struct { 
32   next http.Handler 
33 } 
34  
35 func newValidationHandler(next http.Handler) http.Handler { 
36   return validationHandler{next: next} 
37 } 

```

创建我们自己的 `Handler` 时，我们首先需要定义一个 `struct` 字段，该字段将实现 `Handlers` 接口中的方法。由于在这个例子中，我们将要链式连接处理程序，第一个处理程序，也就是我们的验证处理程序，需要有一个对链中下一个处理程序的引用，因为它有调用 `ServeHTTP` 或返回响应的责任。

为了方便，我们添加了一个返回新处理程序的函数；然而，我们也可以直接设置下一个字段。但是，这种方法是更好的形式，因为它使我们的代码更容易阅读，并且当我们需要通过一个创建函数传递复杂依赖项到处理程序时，它可以使事情保持整洁一些：

```go
37 func (h validationHandler) ServeHTTP(rw http.ResponseWriter, r  
*http.Request) {
38   var request helloWorldRequest
39   decoder := json.NewDecoder(r.Body)
40
41   err := decoder.Decode(&request)
42   if err != nil {
43     http.Error(rw, "Bad request", http.StatusBadRequest)
44     return
45   }
46
47   h.next.ServeHTTP(rw, r)
48 } 

```

之前的代码块展示了我们如何实现 `ServeHTTP` 方法。这里需要注意的唯一有趣的事情是从第 **44** 行开始的语句。如果解码请求返回错误，我们将向响应写入 500 错误，处理链在这里会停止。只有当没有错误返回时，我们才会调用链中的下一个处理程序，我们通过调用其 `ServeHTTP` 方法来完成此操作。为了传递从请求中解码出的名称，我们只是设置了一个变量：

```go
53 type helloWorldHandler struct{} 
54  
55 func newHelloWorldHandler() http.Handler { 
56   return helloWorldHandler{} 
57 } 
58  
59 func (h helloWorldHandler) ServeHTTP(rw http.ResponseWriter, r *http.Request) { 
60   response := helloWorldResponse{Message: "Hello " + name} 
61  
62   encoder := json.NewEncoder(rw) 
63   encoder.Encode(response) 
64 } 

```

写入响应的 `helloWorldHandler` 类型与我们使用简单函数时看起来并没有太大区别。如果你将其与 *示例 1.6* 进行比较，你会发现我们真正做的只是移除了请求解码。

现在我首先要提到的是，这段代码纯粹是为了说明你可以如何做某事，而不是你应该这样做。在这个简单的情况下，将请求验证和响应发送拆分为两个处理程序增加了许多不必要的复杂性，并且并没有真正使我们的代码更加简洁。然而，这项技术是有用的。当我们稍后在章节中检查身份验证时，你会看到这个模式，因为它允许我们将身份验证逻辑集中化并在处理程序之间共享。

# Context

之前模式的问题是没有办法在不破坏 `http.Handler` 接口的情况下，将验证过的请求从一个处理程序传递到下一个处理程序，但是猜猜看，Go 已经为我们解决了这个问题。`context` 包在 Go 1.7 之前被列为实验性包，最终被纳入标准包。`Context` 类型实现了一种安全的方法来访问请求范围内的数据，该数据可以由多个 Go 线程安全地同时使用。让我们快速看一下这个包，然后更新我们的示例以查看其使用情况。

# 背景

`Background` 方法返回一个没有任何值的空上下文；它通常用于主函数和顶级 `Context`：

```go
func Background() Context 

```

# WithCancel

`WithCancel` 方法返回一个带有取消函数的父上下文副本，调用取消函数会释放与上下文关联的资源，应该在 `Context` 类型的操作完成后立即调用：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) 

```

# WithDeadline

`WithDeadline` 方法返回父上下文的副本，该副本在当前时间大于截止日期后过期。此时，上下文的 `Done` 通道关闭，与之关联的资源被释放。它还返回一个 `CancelFunc` 方法，允许手动取消上下文：

```go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) 

```

# `WithTimeout`

`WithTimeout` 方法与 `WithDeadline` 类似，但你需要传递一个持续时间，`Context` 类型应该存在。一旦这个持续时间过去，`Done` 通道关闭，与上下文关联的资源被释放：

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) 

```

# `WithValue`

`WithValue` 方法返回父 `Context` 的一个副本，其中 `val` 值与键相关联。`Context` 值非常适合用于请求作用域的数据：

```go
func WithValue(parent Context, key interface{}, val interface{}) Context 

```

为什么不尝试修改 *示例 1.7* 来实现一个请求作用域的上下文。关键可能就在上一句中；每个请求都需要自己的上下文。

# 使用上下文

你可能觉得这相当痛苦，尤其是如果你来自像 Rails 或 Spring 这样的框架背景。编写这类代码并不是你想要花费时间的事情，构建应用程序功能要重要得多。然而，有一点需要注意，Ruby 或 Java 的基础包中都没有更高级的功能。幸运的是，自从 Go 存在的七年里，许多优秀的人已经做到了这一点，当我们查看第三章 介绍 Docker 中的框架时，我们会发现所有这些复杂性都已被一些出色的开源作者处理好了。

除了将上下文引入 Go 的主要发布版本 1.7 以外，还对 `http.Request` 结构体进行了重要更新，我们还有以下新增功能：

```go
func (r *Request) Context() context.Context

```

`Context()` 方法让我们能够访问一个 `context.Context` 结构体，它始终不为空，因为它在请求最初创建时被填充。对于入站请求，`http.Server` 自动管理上下文的生命周期，当客户端连接关闭时自动取消上下文。对于出站请求，`Context` 通过以下方式控制取消：这意味着如果我们取消 `Context()` 方法，我们可以取消出站请求。这一概念在以下示例中得到了说明：

```go
70 func fetchGoogle(t *testing.T) {
71   r, _ := http.NewRequest("GET", "https://google.com", nil)
72
73   timeoutRequest, cancelFunc := context.WithTimeout(r.Context(), 1*time.Millisecond)
74   defer cancelFunc()
75
76   r = r.WithContext(timeoutRequest)
77
78   _, err := http.DefaultClient.Do(r)
79   if err != nil {
80     fmt.Println("Error:", err)
81   }
82 }

```

在第 **74** 行，我们正在从请求中的原始上下文创建一个超时上下文，与自动取消上下文的入站请求不同，我们必须在出站请求中手动执行此步骤。

第 **77** 行实现了添加到 `http.Request` 对象的两个新上下文方法中的第二个：

```go
func (r *Request) WithContext(ctx context.Context) *Request

```

`WithContext` 对象返回原始请求的一个浅拷贝，其上下文已更改为给定的 `ctx` 上下文。

当我们执行这个函数时，我们会发现 1 毫秒后请求将因错误而完成：

```go
Error: Get https://google.com: context deadline exceeded

```

上下文在请求有机会完成之前就超时了，而`do`方法立即返回。这是一个用于出站连接的绝佳技术，多亏了 Go 1.7 中的变化，现在实现起来非常简单。

我们的入站连接怎么办？让我们看看我们如何更新之前的示例。示例 1.9 更新了我们的示例，展示了我们如何利用`context`包实现 Go 协程安全访问对象。完整的示例可以在`reading_writing_json_8/reading_writing_json_8.go`中找到，但我们需要的所有修改都在我们处理器的两个`ServeHTTP`方法中：

```go
41 func (h validationHandler) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
42   var request helloWorldRequest
43   decoder := json.NewDecoder(r.Body)
44
45   err := decoder.Decode(&request)
46   if err != nil {
47     http.Error(rw, "Bad request", http.StatusBadRequest)
48     return
49   }
50
51   c := context.WithValue(r.Context(), validationContextKey("name"), request.Name)
52   r = r.WithContext(c)
53
54   h.next.ServeHTTP(rw, r)
55 }

```

如果我们快速查看我们的`validationHandler`，你会看到当我们收到一个有效的请求时，我们为这个请求创建一个新的上下文，然后将请求中的`Name`字段的值设置到上下文中。你也可能想知道第**51**行发生了什么。当你向上下文中添加一个项目，比如使用`WithValue`调用时，该方法返回前一个上下文的副本，为了节省一点时间并增加一点混淆，我们持有上下文的指针，因此为了将这个副本传递给`WithValue`，我们必须取消引用它。为了更新我们的指针，我们还必须将返回的值设置到指针引用的值，因此我们再次需要取消引用它。我们还需要关注这个方法调用中的键，我们使用`validationContextKey`，这是一个显式声明的字符串类型：

```go
13 type validationContextKey string

```

我们之所以不直接使用简单的字符串，是因为上下文经常跨越包，如果我们只使用字符串，那么我们可能会遇到键冲突，其中一个受我们控制的包正在写入`name`键，而另一个不受我们控制的包也在使用上下文并写入名为`name`的键，在这种情况下，第二个包会意外地覆盖你的上下文值。通过声明包级别的类型`validationContextKey`并使用它，我们可以确保避免这些冲突：

```go
64 func (h helloWorldHandler) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
65   name := r.Context().Value(validationContextKey("name")).(string)
66   response := helloWorldResponse{Message: "Hello " + name}
67
68   encoder := json.NewEncoder(rw)
69   encoder.Encode(response)
70 }

```

要检索值，我们只需获取上下文，然后调用`Value`方法将其转换为字符串。

# Go 标准库中的 RPC

如预期的那样，Go 标准库对 RPC 的支持非常出色，直接开箱即用。让我们看看一些示例，了解我们如何使用它。

# 简单的 RPC 示例

在这个简单的示例中，我们将看到如何使用标准 RPC 包创建一个客户端和服务器，它们使用共享接口通过 RPC 进行通信。我们将遵循我们在学习`net/http`包时运行的典型 Hello World 示例，看看构建基于 RPC 的 API 在 Go 中有多简单：

`rpc/server/server.go`:

```go
34 type HelloWorldHandler struct{} 
35  
36 func (h *HelloWorldHandler) HelloWorld(args *contract.HelloWorldRequest, reply *contract.HelloWorldResponse) error { 
37   reply.Message = ""Hello "" + args.Name 
38   return nil 
39 } 

```

就像我们在使用标准库创建 RPC API 的例子中一样，我们也将定义一个处理器。这个处理器与`http.Handler`的区别在于它不需要符合一个接口；只要我们有一个带有方法的`struct`字段，我们就可以将其注册到 RPC 服务器上：

```go
func Register(rcvr interface{}) error 

```

`Register`函数，位于`rpc`包中，将给定接口中的方法发布到默认服务器，并允许客户端连接到服务时调用它们。方法名称使用具体类型的名称，所以在我们这个实例中，如果我的客户端想要调用`HelloWorld`方法，我们将使用`HelloWorldHandler.HelloWorld`来访问它。如果我们不想使用具体类型的名称，我们可以使用`RegisterName`函数将其注册为不同的名称，该函数使用提供的名称：

```go
func RegisterName(name string, rcvr interface{}) error 

```

这将使我能够将`struct`字段的名称保持为对我的代码有意义的任何名称；然而，对于我的客户端合同，我可能决定使用不同的名称，例如`Greet`：

```go
19 func StartServer() { 
20   helloWorld := &HelloWorldHandler{} 
21   rpc.Register(helloWorld) 
22  
23   l, err := net.Listen("("tcp",", fmt.Sprintf(":%(":%v",", port)) 
24   if err != nil { 
25     log.Fatal(fmt.Sprintf("("Unable to listen on given port: %s",", err)) 
26   } 
27  
28   for { 
29     conn, _ := l.Accept() 
30     go rpc.ServeConn(conn) 
31   } 
32 } 

```

在`StartServer`函数中，我们首先创建我们处理器的新的实例，然后将其注册到默认 RPC 服务器。

与`net/http`的便利性不同，我们可以直接使用`ListenAndServe`创建服务器，当我们使用 RPC 时，我们需要做一些更多的人工工作。在**23**行，我们使用给定的协议创建一个套接字，并将其绑定到 IP 地址和端口。这使我们能够特别选择我们希望服务器使用的协议，`tcp`、`tcp4`、`tcp6`、`unix`或`unixpacket`：

```go
func Listen(net, laddr string) (Listener, error) 

```

`Listen()`函数返回一个实现了`Listener`接口的实例：

```go
type Listener interface { 
  // Accept waits for and returns the next connection to the listener. 
  Accept() (Conn, error) 

  // Close closes the listener. 
  // Any blocked Accept operations will be unblocked and return errors. 
  Close() error 

  // Addr returns the listener's network address. 
  Addr() Addr 
} 

```

为了接收连接，我们必须在监听器上调用`Accept`方法。如果你查看**29**行，你会看到我们有一个无限循环，这是因为与`ListenAndServe`不同，它会阻塞所有连接，而在 RPC 服务器中，我们逐个处理每个连接，一旦我们处理了第一个连接，我们就需要再次调用`Accept`来处理后续的连接，否则应用程序会退出。`Accept`是一个阻塞方法，所以如果没有客户端当前尝试连接到服务，则`Accept`将阻塞，直到有一个客户端尝试连接。一旦我们收到一个连接，我们就需要再次调用`Accept`方法来处理下一个连接。如果你查看我们的示例代码中的**30**行，你会看到我们正在调用`ServeConn`方法：

```go
func ServeConn(conn io.ReadWriteCloser) 

```

`ServeConn`方法在给定的连接上运行`DefaultServer`方法，并且会阻塞直到客户端完成。在我们的示例中，我们在运行服务器之前使用 go 语句，这样我们就可以立即处理下一个等待的连接，而不会阻塞第一个客户端关闭其连接。

在通信协议方面，`ServeConn`使用`gob`线格式[`golang.org/pkg/encoding/gob/`](https://golang.org/pkg/encoding/gob/)，当我们查看 JSON-RPC 时，我们将看到如何使用不同的编码。

`gob`格式是专门设计来促进 Go 到 Go 之间的通信，并且围绕着一个更容易使用且可能比协议缓冲区等更高效的思路进行设计，但这会牺牲跨语言通信的能力。

使用 gob，源和目标值以及类型不需要完全对应，当你发送`struct`时，如果源中有一个字段但接收到的`struct`中没有，那么解码器将忽略这个字段，并且处理将继续而不会出错。如果目标中有一个字段在源中不存在，那么解码器同样会忽略这个字段，并成功处理消息的其余部分。虽然这似乎是一个小的优势，但与旧版本的 RPC 消息（如 JMI）相比，这是一个巨大的进步，因为 JMI 要求客户端和服务器上必须存在完全相同的接口。JMI 的这种不灵活性在两个代码库之间引入了紧密耦合，并在需要部署应用程序更新时带来了无尽的复杂性。

为了向我们的客户提出请求，我们不能再简单地使用 curl，因为我们不再使用 HTTP 协议，消息格式也不再是 JSON。如果我们查看`rpc/client/client.go`中的示例，我们可以看到如何实现一个连接客户端：

```go
13 func CreateClient() *rpc.Client {
14   client, err := rpc.Dial("tcp", fmt.Sprintf("localhost:%v", port))
15   if err != nil {
16     log.Fatal("dialing:", err)
17   }
18
19   return client
20 }

```

之前的代码块显示了我们需要如何设置`rpc.Client`，在**14**行，我们首先需要使用`rpc`包中的`Dial()`函数创建客户端本身：

```go
func Dial(network, address string) (*Client, error)

```

然后，我们使用这个返回的连接向服务器发送请求：

```go
22 func PerformRequest(client *rpc.Client) 
contract.HelloWorldResponse {
23   args := &contract.HelloWorldRequest{Name: "World"}
24   var reply contract.HelloWorldResponse
25
26   err := client.Call("HelloWorldHandler.HelloWorld", args, &reply)
27   if err != nil {
28     log.Fatal("error:", err)
29   }
30
31   return reply
32 }

```

在**26**行，我们正在使用客户端上的`Call()`方法在服务器上调用命名函数：

```go
func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error

```

`Call`是一个阻塞函数，它等待服务器发送回复，假设没有错误，将响应写入我们传递给方法的`HelloWorldResponse`引用中，如果在处理请求时发生错误，则返回错误，并可以相应地处理。

# 通过 HTTP 的 RPC

在你需要使用 HTTP 作为传输协议的情况下，`rpc`包可以通过调用`HandleHTTP`方法来提供这种支持。

`HandleHTTP`方法在你的应用程序中设置两个端点：

```go
const ( 
  // Defaults used by HandleHTTP 
  DefaultRPCPath   = "/_"/_goRPC_"_" 
  DefaultDebugPath = "/"/debug/rpc"" 
) 

```

如果你将浏览器指向`DefaultDebugPath`，你可以看到已注册端点的详细信息，有两点需要注意：

+   这并不意味着你可以轻松地从网页浏览器与你的 API 进行通信。消息仍然是`gob`编码的，因此你需要编写 JavaScript 中的 gob 编码器和解码器，我实际上不确定这是否可能。包的明确意图绝不是支持这种功能，因此我不会建议这样做，基于 JSON 或 JSON-RPC 的消息更适合这个用例。

+   调试端点不会为你提供 API 的自动生成文档。输出相当基础，意图似乎是让你可以跟踪对端点的调用次数。

话虽如此，可能存在需要使用 HTTP 的原因，可能是你的网络不允许使用任何其他协议，或者可能你有一个无法处理纯 TCP 连接的负载均衡器。我们还可以利用 HTTP 标头和其他元数据，这些在纯 TCP 请求中是不可用的。

`rpc_http/server/server.go`

```go
22 func StartServer() { 
23   helloWorld := &HelloWorldHandler{} 
24   rpc.Register(helloWorld) 
25   rpc.HandleHTTP() 
26  
27   l, err := net.Listen("("tcp",", fmt.Sprintf(":%(":%v",", port)) 
28   if err != nil { 
29       log.Fatal(fmt.Sprintf("("Unable to listen on given port: %s",", err)) 
30   } 
31  
32   log.Printf("("Server starting on port %v\n",", port) 
33  
34   http.Serve(l, nil) 
35 } 

```

如果我们查看前面的示例中的第 **25** 行，我们可以看到我们调用的是 `rpc.HandleHTTP` 方法，这是使用 HTTP 与 RPC 一起使用的要求，因为它会将我们之前提到的 HTTP 处理程序与 `DefaultServer` 方法注册。然后我们调用 `http.Serve` 方法，并传递我们在第 **27** 行创建的监听器，我们将第二个参数设置为 `nil`，因为我们希望使用 `DefaultServer` 方法。这正是我们在之前的示例中查看 RESTful 端点时所查看的方法。

# 通过 HTTP 的 JSON-RPC

在这个最后的例子中，我们将查看 `net/rpc/jsonrpc` 包，它提供了一个内置的编解码器，用于将数据序列化和反序列化为 JSON-RPC 标准。我们还将查看如何通过 HTTP 发送这些响应，虽然你可能想知道为什么不直接使用 REST，在某种程度上，我会同意你的观点，这是一个有趣的例子，可以展示我们如何扩展标准框架。

`StartServer` 方法中包含的内容我们之前都见过，这是标准的 `rpc` 服务器设置，主要区别在于第 **42** 行，在那里我们不是启动 RPC 服务器，而是启动一个 `http` 服务器，并将监听器及其处理程序传递给它：

`rpc_http_json/server/server.go`

```go
33 func StartServer() { 
34  helloWorld := new(HelloWorldHandler) 
35  rpc.Register(helloWorld) 
36 
37  l, err := net.Listen("("tcp",", fmt.Sprintf(":%(":%v",", port)) 
38  if err != nil { 
39    log.Fatal(fmt.Sprintf("("Unable to listen on given port: %s",", err)) 
40  } 
41 
42 http.Serve(l, http.HandlerFunc(httpHandler)) 
43 } 

```

我们传递给服务器的处理程序是魔法发生的地方：

```go
45 func httpHandler(w http.ResponseWriter, r *http.Request) { 
46   serverCodec := jsonrpc.NewServerCodec(&HttpConn{in: r.Body, out: w}) 
47   err := rpc.ServeRequest(serverCodec) 
48   if err != nil { 
49     log.Printf("("Error while serving JSON request: %v",", err) 
50   http.Error(w, ""Error while serving JSON request, details have been logged.",.", 500) 
51   return 
52   } 
53 } 

```

在第 **46** 行，我们调用 `jsonrpc.NewServerCodec` 函数，并传递一个实现 `io.ReadWriteCloser` 类型的类型。`NewServerCodec` 方法返回一个实现 `rpc.ClientCodec` 类型的类型，它具有以下方法：

```go
type ClientCodec interface { 
  // WriteRequest must be safe for concurrent use by multiple goroutines. 
  WriteRequest(*Request, interface{}) error 
  ReadResponseHeader(*Response) error 
  ReadResponseBody(interface{}) error 

  Close() error 
} 

```

`ClientCodec` 类型实现了 RPC 请求的写入和 RPC 响应的读取。为了将请求写入连接，客户端调用 `WriteRequest` 方法。为了读取响应，客户端必须成对地调用 `ReadResponseHeader` 和 `ReadResponseBody`。一旦读取了主体，客户端就有责任调用 `Close` 方法来关闭连接。如果将 nil 接口传递给 `ReadResponseBody`，则应读取并丢弃响应的主体：

```go
17 type HttpConn struct { 
18   in  io.Reader 
19   out io.Writer 
20 } 
21 
22 func (c *HttpConn) Read(p []byte) (n int, err error)  { return c.in.Read(p) } 
23 func (c *HttpConn) Write(d []byte) (n int, err error) { return c.out.Write(d) } 
24 func (c *HttpConn) Close() error                      { return nil } 

```

`NewServerCodec` 方法要求我们传递一个实现了 `ReadWriteCloser` 接口类型的参数。由于在我们的 `httpHandler` 方法中并没有传递给我们这样的类型作为参数，因此我们定义了自己的类型 `HttpConn`，它封装了 `http.Request` 的主体，实现了 `io.Reader` 接口，以及 `ResponseWriter` 方法，实现了 `io.Writer` 接口。然后我们可以编写自己的方法，代理对读取器和写入器的调用，创建一个具有正确接口的类型。

对于我们关于标准库中 RPC 的简短介绍，这就结束了；我们将在第三章“介绍 Docker”中更深入地探讨一些框架，看看这些框架是如何被用来构建生产级微服务的。

# 摘要

这章的内容就到这里，我们刚刚用 Go 语言编写了我们的第一个微服务，并且只使用了标准库，你现在应该已经对标准库的强大功能有了认识，它为我们提供了编写基于 RESTful 和 RPC 的微服务所需的大部分功能。我们还探讨了使用 `encoding/json` 包进行数据编码和解码的方法，以及如何通过使用 `gobs` 创建轻量级消息。

随着你阅读这本书的进程，你将看到许多奇妙的开源软件包是如何建立在这些基础之上，使得 Go 语言成为微服务开发的绝佳选择，并且到书的结尾，你将拥有成功使用 Go 语言构建微服务所需的所有知识。
