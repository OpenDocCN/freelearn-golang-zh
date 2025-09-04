

# 与 REST API 一起工作

本章的主题是使用 Go 编程语言开发 RESTful 服务器和客户端，Go 在许多领域都表现出色。

REST 是*表示状态转移*的缩写，主要是为设计网络服务提供了一种标准化的高效方式，使客户端能够访问数据并使用提供的服务器功能。

RESTful 服务通常使用 JSON 格式来交换信息，这得到了 Go 的良好支持。REST 与任何操作系统或系统架构无关，也不是一种协议；然而，为了实现 RESTful 服务，我们需要使用 HTTP 或 HTTPS 等协议，因为 REST 是基于 HTTP(S)协议的 API 约定。

虽然我们已经熟悉了所展示的大多数 Go 代码，但 REST 背后的思想和 Go 代码实现它们的方式将会是新的。RESTful 服务器开发的中心是定义适当的 Go 结构和执行必要的序列化和反序列化操作，以支持客户端和服务器之间 JSON 数据的交换。

这确实是一个非常重要且实用的章节，涵盖了：

+   REST 简介

+   开发 RESTful 服务器和客户端

+   创建功能性的 RESTful 服务器

+   创建 RESTful 客户端

# REST 简介

大多数现代网络应用程序通过公开它们的 API，并允许客户端使用这些 API 与它们进行交互和通信来工作。尽管 REST 与 HTTP 没有直接关联，但大多数网络服务使用 HTTP 作为其底层协议。此外，尽管 REST 可以与任何数据格式一起工作，但通常 REST 意味着**通过 HTTP 的 JSON**，因为在大多数情况下，RESTful 服务中的数据交换是以 JSON 格式进行的。也有时候数据是以纯文本格式交换的，通常当交换的数据很简单且没有实际需要 JSON 记录时。由于 RESTful 服务的工作方式，它应该有一个遵循以下原则的架构：

+   客户端-服务器设计

+   无状态实现（每次交互不依赖于之前的交互）

+   可缓存

+   统一接口

+   分层系统

根据 HTTP 协议，我们可以在 HTTP 服务器上执行以下操作：

+   `POST`：这用于创建新资源。

+   `GET`：这用于读取（获取）现有资源。

+   `PUT`：这用于更新现有资源。按照惯例，一个`PUT`请求应该包含现有资源的完整和更新版本。

+   `DELETE`：这用于删除现有资源。

+   `PATCH`：这用于更新现有资源。一个`PATCH`请求只包含对现有资源的修改。

这里重要的是，你做的每一件事，尤其是当你做的是非同寻常的事情时，都必须有很好的文档记录。作为一个参考，请记住，Go 支持的 HTTP 方法被定义为`net/http`包中的常量：

```go
const (
    MethodGet     = "GET"
    MethodHead    = "HEAD"
    MethodPost    = "POST"
    MethodPut     = "PUT"
    MethodPatch   = "PATCH" // RFC 5789
    MethodDelete  = "DELETE"
    MethodConnect = "CONNECT"
    MethodOptions = "OPTIONS"
    MethodTrace   = "TRACE"
) 
```

对于每个客户端请求返回的 HTTP 状态码也存在一些约定。以下是最流行的 HTTP 状态码及其含义：

+   `200` 表示一切顺利，指定的操作已成功执行。

+   `201` 表示所需的资源已创建。

+   `202` 表示请求已被接受，目前正在处理中。这通常用于动作需要太长时间才能完成的情况。

+   `301` 表示请求的资源已永久移动——新的 URI 应该包含在响应中。这在 RESTful 服务中很少使用，因为 API 版本控制被用来代替。

+   `400` 表示请求有误，你应该在再次发送之前更改你的初始请求。

+   `401` 表示客户端尝试未经授权访问受保护的请求。

+   `403` 表示客户端即使已经正确授权，也没有访问资源的必要权限。在 UNIX 术语中，`403` 表示当前用户没有执行该操作的必要权限。

+   `404` 表示未找到资源。

+   `405` 表示客户端使用了资源类型不允许的方法。

+   `500` 表示内部服务器错误——这通常表明服务器出现故障。

如果你想了解更多关于 HTTP 协议的信息，你应该访问 RFC 7231，网址为 [`datatracker.ietf.org/doc/html/rfc7231`](https://datatracker.ietf.org/doc/html/rfc7231)。

现在，让我给你讲一个个人故事。几年前，我正在为一个我正在工作的项目开发一个小型的 RESTful 客户端。客户端连接到指定的服务器以获取用户名列表。对于每个用户名，我必须通过访问另一个端点来获取登录和登出时间。

从我的个人经验中，我可以告诉你，大部分的 Go 代码并不是关于与 RESTful 服务器交互，而是关于处理数据，将其转换为所需的格式，并将其存储在数据库中——我需要执行的两大棘手任务是获取 UNIX 纪元格式的日期和时间，以及从该纪元时间中截断关于分钟和秒的信息，以及确保具有相同数据的记录尚未存储在该数据库中后，将新记录插入数据库表。因此，预期你将要编写的代码的大部分将关于面对服务的逻辑，这不仅适用于 RESTful 服务，也适用于所有服务。

本章的第一部分包含有关编程 RESTful 服务器和客户端的通用但基本的信息。

# 开发 RESTful 服务器和客户端

本节将使用 Go 标准库的功能开发一个 RESTful 服务器及其客户端，以了解幕后实际是如何工作的。服务器功能在以下端点列表中描述：

+   `/add`：此端点是用于向服务器添加新条目。

+   `/delete`：此端点用于删除现有条目。

+   `/get`：此端点是用于获取已存在条目的信息。

+   `/time`：此端点返回当前日期和时间，主要用于测试 RESTful 服务器的操作。

+   `/`：此端点用于处理不匹配任何其他端点的任何请求。

定义端点的另一种替代和更专业的方法如下：

+   `/users/` 使用`GET`方法：获取所有用户的列表。

+   `/users/:id` 使用`GET`方法：获取具有给定 ID 值的用户信息。

+   `/users/:id` 使用`DELETE`方法：删除具有给定 ID 的用户。

+   `/users/` 使用`POST`方法：创建新用户。

+   `/users/:id` 使用`PATCH`或`PUT`方法：更新具有给定 ID 值的用户。

替代方法的实现留给读者作为练习——鉴于处理器的 Go 代码将相同，并且你只需重新定义我们指定端点处理的部分，这应该不难实现。

下一个子节将介绍 RESTful 服务器的实现。

## 一个 RESTful 服务器

提出的实现目的是为了理解幕后的事情是如何运作的，因为 REST 服务的原则保持不变。

每个处理器函数背后的逻辑很简单。每个函数读取用户输入，并在处理任何数据之前决定给定的输入和 HTTP 方法是否是所需的。

每个客户交互的原则也很简单：服务器应向客户端发送适当的错误消息和 HTTP 状态码，以便每个人都知道真正发生了什么。最后，所有内容都应被记录下来，以便用一种共同的语言进行沟通。

服务器代码，保存为`rServer.go`，如下所示：

```go
package main
import (
    "encoding/json"
"fmt"
"io"
"log"
"net/http"
"os"
"time"
)
type User struct {
    Username string `json:"user"`
    Password string `json:"password"`
} 
```

这是一个存储用户数据的结构，因此必须使用 JSON 标签，因为 JSON 数据格式与我们使用的 Go 语言中的不同。

```go
var user User 
```

`user`全局变量持有当前交互的用户数据——这是`/add`、`/get`和`/delete`端点及其简单实现的输入。由于此全局变量在整个程序中共享，我们的代码不是并发安全的，这对于用作概念证明的 RESTful 服务器来说是完全可以接受的。

```go
// PORT is where the web server listens to
var PORT = ":1234" 
```

RESTful 服务器只是一个 HTTP 服务器，因此我们需要定义服务器监听的 TCP 端口号。

```go
// DATA is the map that holds User records
var DATA = make(map[string]string) 
```

上述代码定义了一个名为`DATA`的全局变量，它包含服务的数据。

```go
func defaultHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host)
    w.WriteHeader(http.StatusNotFound)
    body := "Thanks for visiting!\n"
    fmt.Fprintf(w, "%s", body)
} 
```

这是服务的默认处理器。在生产服务器上，默认处理器可能会打印有关服务器操作以及可用端点的说明。

```go
func timeHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host)
    t := time.Now().Format(time.RFC1123)
    body := "The current time is: " + t + "\n"
    fmt.Fprintf(w, "%s", body)
} 
```

`timeHandler()` 是另一个简单的处理器，它返回当前的日期和时间——这样的简单处理器通常用于测试服务器的健康状态，并且在生产版本中通常会被移除。一个具有类似目的的非常流行的端点是 `/health`，它通常存在于现代 REST API 中，其目的是在即使在生产环境中也提供服务器的健康状态。

```go
func addHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host, r.Method)
    if r.Method != http.MethodPost {
        fmt.Fprintf(w, "%s\n", "Method not allowed!")
        http.Error(w, "Error:", http.StatusMethodNotAllowed)
        return
    } 
```

这是您第一次看到 `http.Error()` 函数。`http.Error()` 函数向客户端请求发送回复，包括指定的错误消息，该消息应为纯文本，以及所需的 HTTP 状态码。您仍然需要使用 `fmt.Fprintf()` 语句编写要发送回客户端的数据。然而，`http.Error()` 需要最后调用，因为我们不应该在调用 `http.Error()` 之后对 `w` 执行任何更多的写入操作。

```go
 d, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Error:", http.StatusBadRequest)
        return
    } 
```

我们尝试一次性从客户端读取所有数据，使用 `io.ReadAll()`，并通过检查 `io.ReadAll(r.Body)` 返回的 `d` 变量的值来确保我们读取数据时没有错误。

```go
 err = json.Unmarshal(d, &user)
    if err != nil {
        log.Println(err)
        http.Error(w, "Error:", http.StatusBadRequest)
        return
    } 
```

在从客户端读取数据后，我们将其放入 `user` 全局变量——尽管使用全局变量不被视为最佳实践，但我个人更喜欢在处理相对较小的 Go 源代码文件时使用全局变量来存储重要的设置或需要共享的数据。数据存储的位置以及如何处理数据由服务器决定。没有关于如何解释数据的规则。因此，**客户端应按照服务器的意愿与服务器通信**。

```go
 if user.Username == "" {
        http.Error(w, "Error:", http.StatusBadRequest)
        return
    }
    DATA[user.Username] = user.Password
    log.Println(DATA)
    w.WriteHeader(http.StatusCreated)
} 
```

如果给定的 `Username` 字段不为空，则将新的结构添加到 `DATA` 映射中。对于这个示例服务器，没有实现数据持久化——每次重新启动 RESTful 服务器时，`DATA` 映射都会从头开始初始化。

如果 `username` 字段的值为空，则我们无法将其添加到 `DATA` 映射中，并且操作会以 `http.StatusBadRequest` 状态码失败。

```go
func getHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host, r.Method)
    if r.Method != http.MethodGet {
        http.Error(w, "Error:", http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "%s\n", "Method not allowed!")
        return
    } 
```

对于 `/get` 端点，我们需要使用 `http.MethodGet`，因此我们必须确保满足这个条件（`if r.Method != http.MethodGet`）。

```go
 d, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "ReadAll - Error", http.StatusBadRequest)
        return
    } 
```

之后，我们仍然需要确保我们可以无问题地从客户端请求中读取数据。

```go
 err = json.Unmarshal(d, &user)
    if err != nil {
        log.Println(err)
        http.Error(w, "Unmarshal - Error", http.StatusBadRequest)
        return
    }
    fmt.Println(user) 
```

然后，我们使用客户端数据并将其放入 `User` 结构（`user` 全局变量）。再次提醒，使用全局变量来存储数据是个人偏好，对于较小的源代码文件来说效果很好，但对于较大的程序应该避免使用。

```go
 _, ok := DATA[user.Username]
    if ok && user.Username != "" {
        log.Println("Found!")
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "%s\n", d) 
```

如果找到了所需的用户记录，我们使用存储在 `d` 变量中的数据将其发送回客户端——记住，`d` 在 `io.ReadAll(r.Body)` 调用中初始化，并且已经包含了一个序列化的 JSON 记录。

```go
 } else {
        log.Println("Not found!")
        w.WriteHeader(http.StatusNotFound)
        http.Error(w, "Map - Resource not found!", http.StatusNotFound)
    }
    return
} 
```

否则，我们通知客户所需的记录未找到，并返回 `http.StatusNotFound`。

```go
func deleteHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving:", r.URL.Path, "from", r.Host, r.Method)
    if r.Method != http.MethodDelete {
        fmt.Fprintf(w, "%s\n", "Method not allowed!")
        http.Error(w, "Error:", http.StatusMethodNotAllowed)
        return
    } 
```

当删除资源时，`DELETE` HTTP 方法看起来是一个合理的选择，因此有`r.Method != http.MethodDelete`的检查。

```go
 d, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "ReadAll - Error", http.StatusBadRequest)
        return
    } 
```

再次，我们读取客户端输入并将其存储在`d`变量中。

```go
 err = json.Unmarshal(d, &user)
    if err != nil {
        log.Println(err)
        http.Error(w, "Unmarshal - Error", http.StatusBadRequest)
        return
    }
    log.Println(user) 
```

在删除资源时保留额外的日志信息被认为是一种良好的做法。

```go
 _, ok := DATA[user.Username]
    if ok && user.Username != "" {
        if user.Password == DATA[user.Username] { 
```

对于删除过程，我们在删除相关条目之前确保提供的用户名和密码值与`DATA`映射中存在的值相同。

```go
 delete(DATA, user.Username)
            w.WriteHeader(http.StatusOK)
            fmt.Fprintf(w, "%s\n", d)
            log.Println(DATA)
        }
    } else {
        log.Println("User", user.Username, "Not found!")
        w.WriteHeader(http.StatusNotFound)
        http.Error(w, "Resource not found!", http.StatusNotFound)
    }
    return
}
func main() {
    arguments := os.Args
    if len(arguments) != 1 {
        PORT = ":" + arguments[1]
    } 
```

之前的代码展示了一种在拥有默认值的同时定义 Web 服务器 TCP 端口号的技术。因此，如果没有命令行参数，将使用默认值。否则，将使用作为命令行参数给出的值。

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

上述代码块包含了 Web 服务器的详细信息及选项。

```go
 mux.Handle("/time", http.HandlerFunc(timeHandler))
    mux.Handle("/add", http.HandlerFunc(addHandler))
    mux.Handle("/get", http.HandlerFunc(getHandler))
    mux.Handle("/delete", http.HandlerFunc(deleteHandler))
    mux.Handle("/", http.HandlerFunc(defaultHandler)) 
```

之前的代码定义了 Web 服务器的端点——在这里没有特别之处，因为 RESTful 服务器在幕后实现了一个 HTTP 服务器。

```go
 fmt.Println("Ready to serve at", PORT)
    err := s.ListenAndServe()
    if err != nil {
        fmt.Println(err)
        return
    }
} 
```

最后一步是使用预定义的选项运行 Web 服务器，这是常见的做法。之后，我们使用`curl(1)`实用程序测试 RESTful 服务器，这在没有客户端且想要测试 RESTful 服务器操作时非常方便——好处是`curl(1)`可以发送和接收 JSON 数据。

当与 RESTful 服务器一起工作时，我们需要在`curl(1)`中添加`-H 'Content-Type: application/json'`来指定我们将使用 JSON 格式进行操作。`-d`选项用于向服务器传递数据，相当于`--data`选项，而`-v`选项在需要更多详细信息来理解正在发生的事情时会产生更详细的输出。

```go
$ curl localhost:1234/
Thanks for visiting! 
```

与 RESTful 服务器第一次交互是为了确保服务器按预期工作。接下来的交互是为服务器添加新用户——用户的详细信息在`{"user": "mtsouk", "password" : "admin"}` JSON 记录中：

```go
$ curl -H 'Content-Type: application/json' -d '{"user": "mtsouk", "password" : "admin"}' http://localhost:1234/add -v
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 1234 (#0) 
```

之前的输出显示`curl(1)`已成功连接到服务器（`localhost`），并使用所需的 TCP 端口（`1234`）。

```go
> POST /add HTTP/1.1
> Host: localhost:1234
> User-Agent: curl/7.64.1
> Accept: */*
> Content-Type: application/json
> Content-Length: 40 
```

之前的输出显示`curl(1)`将使用`POST`方法发送数据，数据长度为 40 字节。

```go
>
< HTTP/1.1 200 OK
< Date: Sat, 28 Oct 2023 19:56:45 GMT
< Content-Length: 0 
```

之前的输出告诉我们数据已发送，并且服务器响应的主体为`0`字节。

```go
<
* Connection #0 to host localhost left intact 
```

输出的最后部分告诉我们，在向服务器发送数据后，连接被关闭。

如果我们尝试添加相同的用户，RESTful 服务器不会抱怨：

```go
$ curl -H 'Content-Type: application/json' -d '{"user": "mtsouk", "password" : "admin"}' http://localhost:1234/add 
```

虽然这种行为可能并不完美，但如果进行了文档记录，那么它是好的。在生产服务器上这是不允许的，但在实验时是可以接受的。因此，我们在这里偏离了标准做法，你不应该在生产环境中这样做。

```go
$ curl -H 'Content-Type: application/json' -d '{"user": "mihalis", "password" : "admin"}' http://localhost:1234/add 
```

使用上述命令，我们添加了另一个用户，其详细信息由`{"user": "mihalis", "password" : "admin"}`指定。

```go
$ curl -H -d '{"user": "admin"}' http://localhost:1234/add
curl: (3) URL using bad/illegal format or missing URL
Error:
Method not allowed! 
```

前面的输出显示了一个错误的交互，其中 `-H` 后面没有跟值。尽管请求已发送到服务器，但由于 `/add` 不使用默认的 HTTP 方法，它被拒绝。

```go
$ curl -H 'Content-Type: application/json' -d '{"user": "admin", "password": "admin"}' http://localhost:1234/get
Error:
Method not allowed! 
```

这次，curl 命令是正确的，但使用的 HTTP 方法设置不正确。因此，请求没有被服务。

```go
$ curl -X GET -H 'Content-Type: application/json' -d '{"user": "admin", "password" : "admin"}' http://localhost:1234/get
Map - Resource not found!
$ curl -X GET -H 'Content-Type: application/json' -d '{"user": "mtsouk", "password" : "admin"}' http://localhost:1234/get
{"user": "mtsouk", "password" : "admin"} 
```

前两个交互使用 `/get` 来获取现有用户的信息。然而，只找到了第二个用户。

```go
$ curl -H 'Content-Type: application/json' -d '{"user": "mtsouk", "password" : "admin"}' http://localhost:1234/delete -X DELETE
{"user": "mtsouk", "password" : "admin"} 
```

最后一次交互成功删除了由 `{"user": "mtsouk", "password" : "admin"}` 指定的用户。

服务器进程为所有之前的交互生成的输出将如下所示：

```go
$ go run rServer.go
Ready to serve at :1234
2023/10/28 22:56:36 Serving: / from localhost:1234
2023/10/28 22:56:45 Serving: /add from localhost:1234 POST
2023/10/28 22:56:45 map[mtsouk:admin]
2023/10/28 22:57:44 Serving: /add from localhost:1234 POST
2023/10/28 22:57:44 map[mtsouk:admin]
2023/10/28 22:59:29 Serving: /add from localhost:1234 POST
2023/10/28 22:59:29 map[mihalis:admin mtsouk:admin]
2023/10/28 22:59:47 Serving: /add from localhost:1234 GET
2023/10/28 23:00:08 Serving: /get from localhost:1234 POST
2023/10/28 23:00:17 Serving: /get from localhost:1234 GET
{admin admin}
2023/10/28 23:00:17 Not found!
2023/10/28 23:00:32 Serving: /get from localhost:1234 GET
{mtsouk admin}
2023/10/28 23:00:32 Found!
2023/10/28 23:00:45 Serving: /delete from localhost:1234 DELETE
2023/10/28 23:00:45 {mtsouk admin}
2023/10/28 23:00:45 map[mihalis:admin]
2023/10/28 23:00:45 After: map[mihalis:admin] 
```

到目前为止，我们已经有一个可以工作的 RESTful 服务器，它已经通过 `curl(1)` 工具进行了测试。下一节是关于为 RESTful 服务器开发命令行客户端。

## 一个 RESTful 客户端

本小节说明了之前开发的 RESTful 服务器的客户端开发。然而，在这种情况下，客户端充当测试程序，尝试 RESTful 服务器的功能——在本章的后面，你将学习如何使用 cobra 库编写正确的客户端。因此，可以在 `rClient.go` 中找到客户端的代码，如下所示：

```go
package main
import (
    "bytes"
"encoding/json"
"fmt"
"io"
"net/http"
"os"
"time"
)
type User struct {
    Username string `json:"user"`
    Password string `json:"password"`
} 
```

这种相同的结构在服务器实现中也可以找到，并用于数据交换。

```go
var u1 = User{"admin", "admin"}
var u2 = User{"tsoukalos", "pass"}
var u3 = User{"", "pass"} 
```

在这里，我们预先定义了三个将要用于测试的 `User` 变量。

```go
const addEndPoint = "/add"
const getEndPoint = "/get"
const deleteEndPoint = "/delete"
const timeEndPoint = "/time" 
```

前面的常量定义了将要使用的端点。

```go
func deleteEndpoint(server string, user User) int {
    userMarshall, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error in req: ", err)
        return http.StatusInternalServerError
    }
    u := bytes.NewReader(userMarshall)
    req, err := http.NewRequest(http.MethodDelete, server+deleteEndPoint, u) 
```

我们准备了一个请求，将使用 `DELETE` HTTP 方法访问 `/delete`。

```go
 if err != nil {
        fmt.Println("Error in req: ", err)
        return http.StatusBadRequest
    }
    req.Header.Set("Content-Type", "application/json") 
```

这是指定在与服务器交互时使用 JSON 数据的正确方式。

```go
 c := &http.Client{
        Timeout: 15 * time.Second,
    }
    resp, err := c.Do(req) 
```

然后，我们使用 `Do()` 方法并设置 15 秒的超时时间来发送请求并等待服务器响应。

```go
 if err != nil {
        fmt.Println("Error:", err)
    }
    defer resp.Body.Close()
    if resp == nil {
        return http.StatusBadRequest
    }
    data, err := io.ReadAll(resp.Body)
    fmt.Print("/delete returned: ", string(data)) 
```

在这里放置 `fmt.Print()` 的原因是，即使交互中存在错误，我们也想了解服务器的响应。

```go
 if err != nil {
        fmt.Println("Error:", err)
    }
    return resp.StatusCode
} 
```

`resp.StatusCode` 的值指定了 `/delete` 的响应 HTTP 状态码。

```go
func getEndpoint(server string, user User) int {
    userMarshall, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error in unmarshalling: ", err)
        return http.StatusBadRequest
    }
    u := bytes.NewReader(userMarshall)
    req, err := http.NewRequest(http.MethodGet, server+getEndPoint, u) 
```

前面的代码将使用 `GET` HTTP 方法访问 `/get`。

```go
 if err != nil {
        fmt.Println("Error in req: ", err)
        return http.StatusBadRequest
    }
    req.Header.Set("Content-Type", "application/json") 
```

我们使用 `Header.Set()` 指定我们将使用 JSON 格式与服务器交互。

```go
 c := &http.Client{
        Timeout: 15 * time.Second,
    } 
```

前面的语句为 HTTP 客户端定义了一个超时时间，以防服务器响应过于繁忙。

```go
 resp, err := c.Do(req)
    if err != nil {
        fmt.Println("Error:", err)
    }
    defer resp.Body.Close()
    if resp == nil {
        return resp.StatusCode
    } 
```

前面的代码使用 `c.Do(req)` 将客户端请求发送到服务器，并将服务器响应保存到 `resp` 中，将错误值保存到 `err` 中。如果 `resp` 的值为 `nil`，则表示服务器响应为空，这是一个错误条件。

```go
 data, err := io.ReadAll(resp.Body)
    fmt.Print("/get returned: ", string(data))
    if err != nil {
        fmt.Println("Error:", err)
    }
    return resp.StatusCode
} 
```

`resp.StatusCode` 的值，由 RESTful 服务器指定和传递，决定了交互在 HTTP 意义上（逻辑上）是否成功。

```go
func addEndpoint(server string, user User) int {
    userMarshall, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error in unmarshalling: ", err)
        return http.StatusBadRequest
    }
    u := bytes.NewReader(userMarshall)
    req, err := http.NewRequest("POST", server+addEndPoint, u) 
```

我们将使用 `POST` HTTP 方法访问 `/add`。我们可以使用 `http.MethodPost` 而不是 `POST`。如本章前面所述，http 中存在用于剩余 HTTP 方法的相关全局变量（`http.MethodGet`、`http.MethodDelete`、`http.MethodPut` 等），并且建议我们使用它们以提高可移植性。

```go
 if err != nil {
        fmt.Println("Error in req: ", err)
        return http.StatusBadRequest
    }
    req.Header.Set("Content-Type", "application/json") 
```

如前所述，我们指定我们将使用 JSON 格式与服务器交互。

```go
 c := &http.Client{
        Timeout: 15 * time.Second,
    } 
```

再次强调，我们为客户端定义了一个超时时间，以防服务器响应过于繁忙。

```go
 resp, err := c.Do(req)
     if resp == nil || (resp.StatusCode == http.StatusNotFound) {
        return resp.StatusCode
    }
    defer resp.Body.Close()
    return resp.StatusCode
} 
```

`addEndpoint()` 函数用于使用 `POST` 方法测试 `/add` 端点。

```go
func timeEndpoint(server string) (int, string) {
    req, err := http.NewRequest(http.MethodPost, server+timeEndPoint, nil) 
```

我们将使用 `POST` HTTP 方法访问 `/time` 端点。

```go
 if err != nil {
        fmt.Println("Error in req: ", err)
        return http.StatusBadRequest, ""
    }
    c := &http.Client{
        Timeout: 15 * time.Second,
    } 
```

如前所述，我们为客户端定义了一个超时时间，以防服务器响应过于繁忙。

```go
 resp, err := c.Do(req)
    if resp == nil || (resp.StatusCode == http.StatusNotFound) {
        return resp.StatusCode, ""
    }
    defer resp.Body.Close()
    data, _ := io.ReadAll(resp.Body)
    return resp.StatusCode, string(data)
} 
```

`timeEndpoint()` 函数用于测试 `/time` 端点——请注意，此端点不需要从客户端获取任何数据，因此客户端请求为空。服务器将返回一个包含服务器当前时间和日期的字符串。

```go
func slashEndpoint(server, URL string) (int, string) {
    req, err := http.NewRequest(MethodPost, server+URL, nil) 
```

我们将使用 `POST` HTTP 方法访问 `/`。

```go
 if err != nil {
        fmt.Println("Error in req: ", err)
        return http.StatusBadRequest, ""
    }
    c := &http.Client{
        Timeout: 15 * time.Second,
    } 
```

在服务器响应延迟的情况下，在客户端设置超时时间被认为是一种良好的实践。

```go
 resp, err := c.Do(req)
    if resp == nil {
        return resp.StatusCode, ""
    }
    defer resp.Body.Close()
    data, _ := io.ReadAll(resp.Body)
    return resp.StatusCode, string(data)
} 
```

`slashEndpoint()` 函数用于测试服务器中的默认端点——请注意，此端点不需要从客户端获取任何数据。

下一步是实现 `main()` 函数，它使用所有前面的函数来访问 RESTful 服务器端点：

```go
func main() {
    if len(os.Args) != 2 {
        fmt.Println("Wrong number of arguments!")
        fmt.Println("Need: Server URL")
        return
    }
    server := os.Args[1] 
```

`server` 变量包含将要使用的服务器地址和端口号。

```go
 fmt.Println("/add")
    httpCode := addEndpoint(server, u1)
    if HTTPcode != http.StatusOK {
        fmt.Println("u1 Return code:", httpCode)
    } else {
        fmt.Println("u1 Data added:", u1, httpCode)
    }
    httpCode = addEndpoint(server, u2)
    if httpCode != http.StatusOK {
        fmt.Println("u2 Return code:", httpCode)
    } else {
        fmt.Println("u2 Data added:", u2, httpCode)
    }
    httpCode = addEndpoint(server, u3)
    if httpCode != http.StatusOK {
        fmt.Println("u3 Return code:", httpCode)
    } else {
        fmt.Println("u3 Data added:", u3, httpCode)
    } 
```

所有的前一部分代码都是用于使用各种类型的数据测试 `/add` 端点。

```go
 fmt.Println("/get")
    httpCode = getEndpoint(server, u1)
    fmt.Println("/get u1 return code:", httpCode)
    httpCode = getEndpoint(server, u2)
    fmt.Println("/get u2 return code:", httpCode)
    httpCode = getEndpoint(server, u3)
    fmt.Println("/get u3 return code:", httpCode) 
```

所有的前一部分代码都是用于使用各种类型的输入测试 `/get` 端点。我们只测试返回代码，因为 HTTP 状态码指定了操作的成功或失败。

```go
 fmt.Println("/delete")
    httpCode = deleteEndpoint(server, u1)
    fmt.Println("/delete u1 return code:", httpCode)
    httpCode = deleteEndpoint(server, u1)
    fmt.Println("/delete u1 return code:", httpCode)
    httpCode = deleteEndpoint(server, u2)
    fmt.Println("/delete u2 return code:", httpCode)
    httpCode = deleteEndpoint(server, u3)
    fmt.Println("/delete u3 return code:", httpCode) 
```

所有的前一部分代码都是用于使用各种类型的输入测试 `/delete` 端点。再次，我们打印交互的 HTTP 状态码，因为 HTTP 状态码的值指定了客户端请求的成功或失败。

```go
 fmt.Println("/time")
    httpCode, myTime := timeEndpoint(server)
    fmt.Print("/time returned: ", httpCode, " ", myTime)
    time.Sleep(time.Second)
    httpCode, myTime = timeEndpoint(server)
    fmt.Print("/time returned: ", httpCode, " ", myTime) 
```

前面的代码测试了 `/time` 端点——它打印了 HTTP 状态码以及服务器响应的其余部分。

```go
 fmt.Println("/")
    URL := "/"
    httpCode, response := slashEndpoint(server, URL)
    fmt.Print("/ returned: ", httpCode, " with response: ", response)
    fmt.Println("/what")
    URL = "/what"
    httpCode, response = slashEndpoint(server, URL)
    fmt.Print(URL, " returned: ", httpCode, " with response: ", response)
} 
```

程序的最后部分尝试连接到一个不存在的端点，以验证默认处理函数的正确操作。

运行 `rClient.go` 并与 `rServer.go` 交互会产生以下类型的输出：

```go
$ go run rClient.go http://localhost:1234
/add
u1 Data added: {admin admin} 200
u2 Data added: {tsoukalos pass} 200
u3 Return code: 400 
```

前一部分与 `/add` 端点的测试相关。前两个用户成功添加，而第三个用户（`var u3 = User{"", "pass"}`）没有添加，因为它不包含所有必需的信息。

```go
/get
/get returned: {"user":"admin","password":"admin"}
/get u1 return code: 200
/get returned: {"user":"tsoukalos","password":"pass"}
/get u2 return code: 200
/get returned: Map - Resource not found!
/get u3 return code: 404 
```

前一部分与 `/get` 端点的测试相关。用户名为 `admin` 和 `tsoukalos` 的前两个用户的数据成功返回，而存储在 `u3` 变量中的用户未找到。

```go
/delete
/delete returned: {"user":"admin","password":"admin"}
/delete u1 return code: 200
/delete returned: Delete - Resource not found!
/delete u1 return code: 404
/delete returned: {"user":"tsoukalos","password":"pass"}
/delete u2 return code: 200
/delete returned: Delete - Resource not found!
/delete u3 return code: 404 
```

之前的输出与`/delete`端点的测试相关。`admin`和`tsoukalos`用户已被删除。然而，尝试第二次删除`admin`失败了。

```go
/time
/time returned: 200 The current time is: Sat, 28 Oct 2023 23:03:39 EEST
/time returned: 200 The current time is: Sat, 28 Oct 2023 23:03:40 EEST 
```

同样，前一部分与`/time`端点的测试相关。

```go
/
/ returned: 404 with response: Thanks for visiting!
/what
/what returned: 404 swith response: Thanks for visiting! 
```

输出的最后一部分与默认处理器的操作相关。

重要的是要意识到`rClient.go`可以成功与支持相同端点的每个 RESTful 服务器进行交互，而无需了解 RESTful 服务器的实现细节。

到目前为止，RESTful 服务器和客户端可以相互交互。然而，它们都没有执行真正的任务。下一节将展示如何使用`gorilla/mux`和数据库后端来存储数据开发一个真实的 RESTful 服务器。

# 创建功能性的 RESTful 服务器

本节说明了在给定 REST API 的情况下如何使用 Go 开发 RESTful 服务器。所提供的 RESTful 服务与第九章中创建的统计应用程序之间最大的区别是，RESTful 服务使用 JSON 消息与其客户端进行交互，而统计应用程序通过纯文本消息进行交互和工作。

如果你打算使用`net/http`来实现 RESTful 服务器，请在考虑你的服务器需求之前不要这样做！此实现使用`gorilla/mux`包，这是一个更好的选择，因为它支持子路由——更多关于这一点在*使用 gorilla/mux*子节中。然而，`net/http`仍然非常强大，并且对于许多 REST 需求可能很有用。

RESTful 服务器的目的是实现一个登录/认证系统。登录系统的目的是跟踪已登录的用户以及他们的权限。该系统附带一个名为`admin`的默认管理员用户——默认密码也是`admin`，你应该更改它。应用程序将其数据存储在 SQLite3 数据库中，这意味着如果你重新启动它，现有用户列表将从该数据库读取，而不会丢失。

## REST API

应用程序的 API 有助于你实现你心中的功能。然而，这是一项客户端的工作，而不是服务器的工作。服务器的工作是通过通过适当定义和实现的 REST API 支持简单但完全工作的功能，尽可能多地促进其客户端的工作。在尝试开发和使用 RESTful 服务器之前，请确保你理解这一点。

我们将定义将要使用的端点、将要返回的 HTTP 代码以及允许的方法或方法。基于 REST API 创建用于生产的 RESTful 服务器是一项严肃的工作，不应轻率对待。创建原型来测试和验证你的想法和设计将使你在长期内节省大量时间。始终从原型开始。

支持的端点以及支持的 HTTP 方法和参数如下：

+   `/`: 这用于捕获和提供所有不匹配的内容。此端点与所有 HTTP 方法兼容。

+   `/getall`: 这用于获取数据库的全部内容。使用此功能需要具有管理权限的用户。此端点可能返回多个 JSON 记录，并支持`GET` HTTP 方法。

+   `/getid/username`: 这用于获取通过用户名识别的用户 ID，该 ID 被传递到端点。此命令应由具有管理权限的用户发出，并支持`GET` HTTP 方法。

+   `/username/ID`: 这用于根据所使用的 HTTP 方法删除或获取 ID 等于`ID`的用户的信息。因此，将要执行的实际操作取决于所使用的 HTTP 方法。`DELETE`方法删除用户，而`GET`方法返回用户信息。此端点应由具有管理权限的用户发出。

+   `/logged`: 这用于获取所有已登录用户的列表。此端点可能返回多个 JSON 记录，并需要使用`GET` HTTP 方法。

+   `/update`: 这用于更新用户的用户名、密码或管理员状态——数据库中用户的 ID 保持不变。此端点仅支持`PUT` HTTP 方法，并且对用户的搜索基于用户名。

+   `/login`: 这用于根据用户名和密码将用户登录到系统中。此端点支持`POST` HTTP 方法。

+   `/logout`: 这用于根据用户名和密码注销用户。此端点支持`POST` HTTP 方法。

+   `/add`: 这用于将新用户添加到数据库中。此端点支持`POST` HTTP 方法，并由具有管理权限的用户发出。

+   `/time`: 这是一个主要用于测试目的的端点。它是唯一一个不与 JSON 数据兼容、不需要有效账户且与所有 HTTP 方法兼容的端点。

现在，让我们讨论`gorilla/mux`包的功能和功能。

## 使用 gorilla/mux

`gorilla/mux`包([`github.com/gorilla/mux`](https://github.com/gorilla/mux))是默认 Go 路由器的流行且强大的替代品，允许您将传入请求与相应的处理程序匹配。尽管默认 Go 路由器(`http.ServeMux`)和`mux.Router`（`gorilla/mux`路由器）之间存在许多差异，但主要区别在于`mux.Router`在匹配路由与处理函数时支持多个条件。这意味着您可以用更少的代码处理一些选项，例如使用的 HTTP 方法。

让我们先通过一些匹配示例开始——默认 Go 路由器不支持此功能：

+   `r.HandleFunc("/url", UrlHandlerFunction)`: 前一个命令会在每次访问 `/url` 时调用 `UrlHandlerFunction` 函数。

+   `r.HandleFunc("/url", UrlHandlerFunction).Methods(http.MethodPut)`: 这个例子展示了如何告诉 Gorilla 匹配特定的 HTTP 方法（在这个例子中是 `PUT`，它通过使用 `http.MethodPut` 定义），这可以节省你手动编写代码来执行此操作。

+   `mux.NotFoundHandler = http.HandlerFunc(handlers.DefaultHandler)`: 使用 Gorilla，匹配任何其他路径都不匹配的内容的正确方法是通过使用 `mux.NotFoundHandler`。

+   `mux.MethodNotAllowedHandler = notAllowed`: 如果一个方法对于现有路由不被允许，它将通过 `MethodNotAllowedHandler` 来处理。这是 `gorilla/mux` 的特定功能。

+   `s.HandleFunc("/users/{id:[0-9]+}"), HandlerFunction)`: 这个最后的例子表明，你可以使用名称（`id`）和模式在路径中定义一个变量——Gorilla 会为你进行匹配！如果没有正则表达式，那么匹配将从路径中的第一个斜杠开始到下一个斜杠之间的任何内容。

现在，让我们来谈谈 `gorilla/mux` 的另一个功能，即子路由器。

## 子路由器的使用

服务器实现使用了子路由器。一个 *子路由器* 是一个嵌套路由，只有当父路由与子路由器的参数匹配时，才会检查它是否有潜在的匹配。好处是父路由可以包含所有在子路由器下定义的路径的共同条件，包括主机、路径前缀，以及在我们的案例中，HTTP 请求方法。因此，我们的子路由器是根据后续端点的公共请求方法来划分的。这不仅优化了请求匹配，还使得代码结构更容易理解。

例如，用于 `DELETE` HTTP 方法的子路由器就像以下这样简单：

```go
deleteMux := mux.Methods(http.MethodDelete).Subrouter()
deleteMux.HandleFunc("/username/{id:[0-9]+}", handlers.DeleteHandler) 
```

第一条语句用于定义子路由器的公共特性，在这个例子中是 `http.MethodDelete` HTTP 方法，而剩下的语句，在这个例子中是 `deleteMux.HandleFunc(...)`，用于定义支持的路径。

是的，`gorilla/mux` 可能比默认的 Go 路由器更难使用，但到目前为止，你应该已经理解了 `gorilla/mux` 包在处理 HTTP 服务时的好处。

下一个子节简要介绍了 Gin 框架，它是对 `gorilla/mux` 包的替代方案。

## Gin HTTP 框架

Gin 是一个用 Go 编写的开源 Web 框架，可以帮助你编写强大的 HTTP 服务。Gin 的 GitHub 仓库可以在`https://github.com/gin-gonic/gin`找到。Gin 使用`httprouter`作为其 HTTP 路由器，因为`httprouter`针对高性能和低内存使用进行了优化。`httprouter`是一个 HTTP 路由器，就像`net/http`包的默认`mux`或更高级的`gorilla/mux`。你可以在[`github.com/julienschmidt/httprouter`](https://github.com/julienschmidt/httprouter)和[`pkg.go.dev/github.com/julienschmidt/httprouter`](https://pkg.go.dev/github.com/julienschmidt/httprouter)了解更多关于它的信息。

## Gin 与 Gorilla 的比较

Gin 与`gorilla/mux`之间最大的区别在于`gorilla/mux`仅仅是一个 HTTP 路由器，没有其他功能，而 Gin 可以做到`gorilla/mux`所能做到的，包括 JSON 序列化和反序列化、验证、自定义响应写入等。简单来说，Gin 能做的事情比`gorilla/mux`多得多。在实践中，你可以将 Gin 视为一个比`gorilla/mux`更高级的框架，具有更多功能。

因此，这里的问题是你应该选择哪一个。从`gorilla/mux`开始尝试并不坏。如果`gorilla/mux`不能完成你的工作，那么你肯定需要使用 Gin。简单来说，如果性能至关重要，如果你需要中间件支持，或者如果你想要最小化和快速的路由，那么请使用 Gin。

## 与数据库交互

在本小节中，我们将向您展示我们如何与支持 RESTful 服务器功能的 SQLite 数据库进行交互。相关文件是`./server/restdb.go`。

RESTful 服务器本身对 SQLite 数据库一无所知。所有相关功能都保存在`restdb.go`文件中，这意味着如果你更改数据库，处理函数不需要知道这一点。

数据库名称、数据库表和`admin`用户是通过下一个`create_db.sql` SQL 文件创建的：

```go
DROP TABLE IF EXISTS users;
CREATE TABLE users (
    UserID INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    password TEXT NOT NULL,
    lastlogin INTEGER,
    admin INTEGER,
    active INTEGER
);
INSERT INTO users (username, password, lastlogin, admin, active) VALUES ('admin', 'admin', 1620922454, 1, 1); 
```

你可以使用`create_db.sql`如下：

```go
$ sqlite3 REST.db
SQLite version 3.39.5 2022-10-14 20:58:05
Enter ".help" for usage hints.
sqlite> .read create_db.sql 
```

我们可以验证`users`表已创建并包含所需的条目，如下所示：

```go
$ sqlite3 REST.db
SQLite version 3.39.5 2022-10-14 20:58:05
Enter ".help" for usage hints.
sqlite> .schema
CREATE TABLE users (
    UserID INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    password TEXT NOT NULL,
    lastlogin INTEGER,
    admin INTEGER,
    active INTEGER
);
sqlite> select * from users;
1|admin|admin|1620922454|1|1 
```

我们将介绍最重要的数据库相关函数，从`OpenConnection()`开始：

```go
func OpenConnection() *sql.DB {
    db, err := sql.Open("sqlite3", Filename)
    if err != nil {
        fmt.Println("Error connecting:", err)
        return nil
    }
    return db
} 
```

由于我们需要始终与 SQLite3 进行交互，我们创建了一个辅助函数，该函数返回一个`*sql.DB`变量，这是一个打开的 SQLite3 连接。`Filename`是一个全局变量，指定了将要使用的 SQLite3 数据库文件。

接下来，我们介绍`DeleteUser()`函数：

```go
func DeleteUser(ID int) bool {
    db := OpenConnection()
    if db == nil {
        log.Println("Cannot connect to SQLite3!")
        return false
    }
    defer db.Close() 
```

上一段代码展示了我们如何使用`OpenConnection()`来获取数据库连接并进行操作。

```go
 t := FindUserID(ID)
    if t.ID == 0 {
        log.Println("User", ID, "does not exist.")
        return false
    } 
```

在这里，我们使用`FindUserID()`辅助函数来确保具有给定用户 ID 的用户存在于数据库中。如果用户不存在，函数将停止并返回`false`。

```go
 stmt, err := db.Prepare("DELETE FROM users WHERE UserID = $1")
    if err != nil {
        log.Println("DeleteUser:", err)
        return false
    } 
```

这是删除用户的实际 SQL 语句。我们使用 `Prepare()` 来构建所需的 SQL 语句，然后使用 `Exec()` 执行。`Prepare()` 中的 `$1` 表示将在 `Exec()` 中给出的参数。如果我们想有更多参数，我们应该将它们命名为 `$2`、`$3` 等。

```go
 _, err = stmt.Exec(ID)
    if err != nil {
        log.Println("DeleteUser:", err)
        return false
    }
    return true
} 
```

这就是 `DeleteUser()` 函数的实现结束的地方。执行 `stmt.Exec(ID)` 语句会从数据库中删除用户。

下一个展示的 `ListAllUsers()` 函数返回一个 `User` 元素切片，它包含在 RESTful 服务器中找到的所有用户：

```go
func ListAllUsers() []User {
    db := OpenConnection()
    if db == nil {
        fmt.Println("Cannot connect to SQLite3!")
        return []User{}
    }
    defer db.Close()
    rows, err := db.Query("SELECT * FROM users \n")
    if err != nil {
        log.Println(err)
        return []User{}
    } 
```

由于 `SELECT` 查询不需要参数，我们使用 `Query()` 来运行它，而不是 `Prepare()` 和 `Exec()`。请注意，这很可能是一个返回多条记录的查询。

```go
 all := []User{}
    var c1 int
var c2, c3 string
var c4 int64
var c5, c6 int
for rows.Next() {
        err = rows.Scan(&c1, &c2, &c3, &c4, &c5, &c6)
        if err != nil {
            log.Println(err)
            return []User{}
        } 
```

这是我们从 SQL 查询返回的单个记录中读取值的方式。首先，我们为每个返回值定义多个变量，然后将它们的指针传递给 `Scan()`。只要数据库中有新的结果，`rows.Next()` 方法就会继续返回记录。

```go
 temp := User{c1, c2, c3, c4, c5, c6}
        all = append(all, temp)
    }
    log.Println("All:", all)
    return all
} 
```

因此，正如之前提到的，`ListAllUsers()` 从 `User` 结构体切片中返回。

最后，我们将展示 `IsUserValid()` 的实现：

```go
func IsUserValid(u User) bool {
    db := OpenConnection()
    if db == nil {
        fmt.Println("Cannot connect to SQLite3!")
        return false
    }
    defer db.Close() 
```

这是一个常见的模式：我们调用 `OpenConnection()` 并等待获取一个连接来使用，然后再继续。

```go
 rows, err := db.Query("SELECT * FROM users WHERE username = $1 \n", u.Username)
    if err != nil {
        log.Println(err)
        return false
    } 
```

在这里，我们直接将参数传递给 `Query()`，而没有使用 `Prepare()` 和 `Exec()`。

```go
 temp := User{}
    var c1 int
var c2, c3 string
var c4 int64
var c5, c6 int 
```

接下来，我们创建所需的参数以保留 SQL 查询的返回值。

```go
 // If there exist multiple users with the same username,
// we will get the FIRST ONE only.
for rows.Next() {
        err = rows.Scan(&c1, &c2, &c3, &c4, &c5, &c6)
        if err != nil {
            log.Println(err)
            return false
        }
        temp = User{c1, c2, c3, c4, c5, c6}
    } 
```

再次强调，`for` 循环会一直运行，直到 `rows.Next()` 返回新的记录。

```go
 if u.Username == temp.Username && u.Password == temp.Password {
        return true
    } 
```

这是一个重要的观点：不仅给定的用户必须存在，而且给定的密码必须与数据库中存储的给定用户的密码相同，才能使用户有效。

```go
 return false
} 
```

你可以自己查看 `restdb.go` 的其余源代码。大多数函数与这里展示的类似。`restdb.go` 的代码将被用于实现接下来展示的 RESTful 服务器。

## 实现 RESTful 服务器

现在，我们准备开始解释 RESTful 服务器的实现。服务器代码分为三个属于 `main` 包的文件。因此，除了 `restdb.go` 之外，我们还有 `main.go` 和 `handlers.go`。

这样做的主要原因是不必处理巨大的源代码文件，并且从逻辑上分离服务器的功能。

`main.go` 中最重要的部分属于 `main()` 函数，如下所示：

```go
 rMux.NotFoundHandler = http.HandlerFunc(DefaultHandler) 
```

因此，我们定义了默认的处理函数。尽管这不是必需的，但拥有这样一个处理函数是一种良好的实践。

```go
 notAllowed := notAllowedHandler{}
    rMux.MethodNotAllowedHandler = notAllowed 
```

当你尝试使用不支持的 HTTP 方法访问端点时，会执行 `MethodNotAllowedHandler` 处理器。该处理器的实际实现位于 `handlers.go` 文件中。

```go
 rMux.HandleFunc("/time", TimeHandler) 
```

`/time` 端点支持所有 HTTP 方法，因此它不属于任何子路由器。

```go
 // Define Handler Functions
// Register GET
    getMux := rMux.Methods(http.MethodGet).Subrouter()
    getMux.HandleFunc("/getall", GetAllHandler)
    getMux.HandleFunc("/getid/{username}", GetIDHandler)
    getMux.HandleFunc("/logged", LoggedUsersHandler)
    getMux.HandleFunc("/username/{id:[0-9]+}", GetUserDataHandler) 
```

首先，我们定义了一个用于 `GET` HTTP 方法的子路由器，以及支持的端点。记住，`gorilla/mux` 负责确保只有 `GET` 请求将通过 `getMux` 子路由器提供服务。

```go
 // Register PUT
// Update User
    putMux := rMux.Methods(http.MethodPut).Subrouter()
    putMux.HandleFunc("/update", UpdateHandler) 
```

之后，我们定义了一个用于 `PUT` 请求的子路由器。

```go
 // Register POST
// Add User + Login + Logout
    postMux := rMux.Methods(http.MethodPost).Subrouter()
    postMux.HandleFunc("/add", AddHandler)
    postMux.HandleFunc("/login", LoginHandler)
    postMux.HandleFunc("/logout", LogoutHandler) 
```

然后，我们定义了用于 `POST` 请求的子路由器。

```go
 // Register DELETE
// Delete User
    deleteMux := rMux.Methods(http.MethodDelete).Subrouter()
    deleteMux.HandleFunc("/username/{id:[0-9]+}", DeleteHandler) 
```

最后一个子路由器用于 `DELETE` HTTP 方法。`gorilla/mux` 中的代码负责根据客户端请求的详细信息选择正确的子路由器。

```go
 go func() {
        log.Println("Listening to", PORT)
        err := s.ListenAndServe()
        if err != nil {
            log.Printf("Error starting server: %s\n", err)
            return
        }
    }() 
```

HTTP 服务器作为 goroutine 执行，因为程序支持信号处理——有关更多详细信息，请参阅 *第八章*，*Go 并发*。

```go
 sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, os.Interrupt)
    sig := <-sigs
    log.Println("Quitting after signal:", sig)
    time.Sleep(5 * time.Second)
    s.Shutdown(nil) 
```

最后，我们添加了信号处理，以便优雅地终止 HTTP 服务器。`sig := <-sigs` 语句防止 `main()` 函数在没有接收到 `os.Interrupt` 信号的情况下退出。

`handlers.go` 文件包含了处理函数的实现，也是 `main` 包的一部分——其最重要的部分如下：

```go
// AddHandler is for adding a new user
func AddHandler(rw http.ResponseWriter, r *http.Request) {
    log.Println("AddHandler Serving:", r.URL.Path, "from", r.Host)
    d, err := io.ReadAll(r.Body)
    if err != nil {
        rw.WriteHeader(http.StatusBadRequest)
        log.Println(err)
        return
    } 
```

此处理程序用于 `/add` 端点。服务器使用 `io.ReadAll()` 读取客户端输入，并确保 `io.ReadAll()` 调用成功。

```go
 if len(d) == 0 {
        rw.WriteHeader(http.StatusBadRequest)
        log.Println("No input!")
        return
    } 
```

然后，代码确保客户端请求的正文不为空。

```go
 // We read two structures as an array:
// 1\. The user issuing the command
// 2\. The user to be added
    users := []User{}
    err = json.Unmarshal(d, &users)
    if err != nil {
        log.Println(err)
        rw.WriteHeader(http.StatusBadRequest)
        return
    } 
```

由于 `/add` 端点需要两个 `User` 结构体，之前的代码使用 `json.Unmarshal()` 将它们放入 `[]User` 变量中——这意味着客户端应该使用数组发送这两个 JSON 记录。

```go
 log.Println(users)
    if !IsUserAdmin(users[0]) {
        log.Println("Issued by non-admin user:", users[0].Username)
        rw.WriteHeader(http.StatusBadRequest)
        return
    } 
```

如果执行命令的用户没有管理权限，则请求失败。`IsUserAdmin()` 在 `restdb.go` 中实现，因为它与数据库中存储的数据有关。

```go
 result := InsertUser(users[1])
    if !result {
        rw.WriteHeader(http.StatusBadRequest)
    }
} 
```

否则，`InsertUser()` 将所需用户插入到数据库中。

最后，我们展示了 `/getall` 端点的处理程序。

```go
// GetAllHandler is for getting all data from the user database
func GetAllHandler(rw http.ResponseWriter, r *http.Request) {
    log.Println("GetAllHandler Serving:", r.URL.Path, "from", r.Host)
    d, err := io.ReadAll(r.Body)
    if err != nil {
        rw.WriteHeader(http.StatusBadRequest)
        log.Println(err)
        return
    } 
```

再次强调，我们使用 `io.ReadAll(r.Body)` 从客户端读取数据，并通过检查 `err` 变量确保整个过程没有错误。

```go
 if len(d) == 0 {
        rw.WriteHeader(http.StatusBadRequest)
        log.Println("No input!")
        return
    }
    user := User{}
    err = json.Unmarshal(d, &user)
    if err != nil {
        log.Println(err)
        rw.WriteHeader(http.StatusBadRequest)
        return
    } 
```

在这里，我们将客户端数据放入 `User` 变量中。`/getall` 端点需要一个单独的 `User` 记录作为输入。

```go
 if !IsUserAdmin(user) {
        log.Println("User", user, "is not an admin!")
        rw.WriteHeader(http.StatusBadRequest)
        return
    } 
```

只有管理员用户可以访问 `/getall` 并获取所有用户的列表，因此使用了 `IsUserAdmin()`。

```go
 err = SliceToJSON(ListAllUsers(), rw)
    if err != nil {
        log.Println(err)
        rw.WriteHeader(http.StatusBadRequest)
        return
    }
} 
```

代码的最后部分是关于从数据库获取所需数据，并使用 `SliceToJSON(ListAllUsers(), rw)` 调用将其发送到客户端。

随意将每个处理程序放入单独的 Go 文件中。一般思路是，如果你有很多处理函数，为每个处理函数使用单独的文件是一种良好的实践。除此之外，它还允许多个开发者同时在不打扰彼此的情况下工作多个处理函数。

在开发合适的命令行客户端之前，使用 `curl(1)` 测试 RESTful 服务器是一个好主意。

## 测试 RESTful 服务器

本小节展示了如何使用`curl(1)`实用程序测试 RESTful 服务器。您应该尽可能多地测试 RESTful 服务器，以查找错误或不受欢迎的行为。由于我们使用三个文件来实现服务器，我们需要以`go run main.go restdb.go handlers.go`的方式运行它。我们首先测试`/time`处理器，它适用于所有 HTTP 方法：

```go
$ curl localhost:1234/time
The current time is: Mon, 30 Oct 2023 19:38:21 EET 
```

接下来，我们测试默认处理器：

```go
$ curl localhost:1234/
/ is not supported. Thanks for visiting!
$ curl localhost:1234/doesNotExist
/doesNotExist is not supported. Thanks for visiting! 
```

最后，我们看看如果我们使用不支持 HTTP 方法与支持端点会发生什么——在这种情况下，仅支持`GET`的`/getall`端点：

```go
$ curl -s -X PUT -H 'Content-Type: application/json' localhost:1234/getall
Method not allowed! 
```

虽然`/getall`端点需要有效的用户才能操作，但我们使用的不受该端点支持的 HTTP 方法具有优先权，因此调用失败，这是正确的失败原因。

在测试过程中，重要的是要查看 RESTful 服务器的输出以及它生成的日志条目。并非所有信息都可以发送回客户端，但服务器进程允许打印任何我们想要的内容。这可以非常有助于调试像我们的 RESTful 服务器这样的服务器进程。

下一小节测试所有支持`GET` HTTP 方法的处理器。

## 测试 GET 处理器

首先，我们测试`/getall`端点——您的输出可能取决于 SQLite 数据库的内容：

```go
$ curl -s -X GET -H 'Content-Type: application/json' -d '{"username": "admin", "password" : "justChanged"}' localhost:1234/getall
[{"id":1,"username":"admin","password":"justChanged","lastlogin":1620922454,"admin":1,"active":1},{"id":2,"username":"","password":"admin","lastlogin":0,"admin":0,"active":0},{"id":3,"username":"mihalis","password":"admin","lastlogin":0,"admin":0,"active":0},{"id":4,"username":"newUser","password":"aPass","lastlogin":0,"admin":0,"active":0}] 
```

之前的输出是数据库中找到的所有现有用户的列表，以 JSON 格式呈现。您始终可以使用`jq(1)`实用程序处理生成的输出，以获得更美观的输出。

然后，我们测试`/logged`端点：

```go
$ curl -X GET -H 'Content-Type: application/json' -d '{"username": "admin", "password" : "justChanged"}' localhost:1234/logged
[{"id":1,"username":"admin","password":"justChanged","lastlogin":1620922454,"admin":1,"active":1}] 
```

然后，我们测试`/username/{id}`端点：

```go
$ curl -X GET -H 'Content-Type: application/json' -d '{"username": "admin", "password" : "justChanged"}' localhost:1234/username/3
{"id":3,"username":"mihalis","password":"admin","lastlogin":0,"admin":0,"active":0} 
```

最后，我们测试`/getid/{username}`端点：

```go
$ curl -X GET -H 'Content-Type: application/json' -d '{"username": "admin", "password" : "justChanged"}' localhost:1234/getid/mihalis
{"id":3,"username":"mihalis","password":"admin","lastlogin":0,"admin":0,"active":0} 
```

因此，用户`mihalis`的用户 ID 为`3`。

到目前为止，我们可以获取现有用户列表和已登录用户列表，并获取特定用户的信息——所有这些端点都使用`GET`方法。下一小节将测试所有支持`POST`方法的处理器。

## 测试 POST 处理器

首先，我们通过添加没有管理员权限的`packt`用户来测试`/add`端点：

```go
$ curl -X POST -H 'Content-Type: application/json' -d '[{"username": "admin", "password" : "justChanged", "admin":1}, {"username": "packt", "password" : "admin", "admin":0} ]' localhost:1234/add 
```

之前的调用向服务器传递了一个包含两个 JSON 记录的数组。第二个记录包含了`packt`用户的详细信息。该命令由`admin`用户发出，该用户由第一个 JSON 记录中的数据指定。

如果我们尝试多次添加相同的用户名，过程将会失败——这可以通过在`curl(1)`命令中使用`-v`来揭示。相关的错误信息将是`HTTP/1.1 400 Bad Request`。

此外，如果我们尝试使用非管理员用户的凭据添加新用户，服务器将生成`由非管理员用户发出的命令：packt`消息。

接下来，我们测试`/login`端点：

```go
$ curl -X POST -H 'Content-Type: application/json' -d '{"username": "packt", "password" : "admin"}' localhost:1234/login 
```

之前的命令用于登录`packt`用户。

最后，我们测试`/logout`端点：

```go
$ curl -X POST -H 'Content-Type: application/json' -d '{"username": "packt", "password" : "admin"}' localhost:1234/logout 
```

之前的命令用于注销`packt`用户。您可以使用`/logged`端点来验证前两个交互的结果。

现在，让我们测试唯一支持`PUT` HTTP 方法的端点。

## 测试 PUT 处理器

首先，我们按照以下方式测试`/update`端点：

```go
$ curl -X PUT -H 'Content-Type: application/json' -d '[{"username": "admin", "password" : "admin", "admin":1}, {"username": "admin", "password" : "justChanged", "admin":1} ]' localhost:1234/update 
```

上一条命令将`admin`用户的密码从`admin`更改为`justChanged`。

然后，我们尝试使用非管理员用户（`packt`）的凭据更改用户密码：

```go
$ curl -X PUT -H 'Content-Type: application/json' -d '[{"Username":"packt","Password":"admin"}, {"username": "admin", "password" : "justChanged", "admin":1} ]' localhost:1234/update 
```

生成的日志消息是`Command issued by non-admin user: packt`。

我们可能会考虑这样一个事实，即非管理员用户甚至无法更改自己的密码，这是一个缺陷——可能如此，但这是 RESTful 服务器实现的方式。其理念是非管理员用户不应直接发出危险命令。此外，这个缺陷可以很容易地修复，如下所示：一般来说，普通用户不会以这种方式与服务器交互，而是会提供一个用于此目的的 Web 界面。之后，管理员用户可以将用户请求发送到服务器。因此，这可以以不同的方式实现，更加安全，并且不会给普通用户不必要的权限。

最后，我们将测试`DELETE` HTTP 方法。

## 测试 DELETE 处理器

对于`DELETE` HTTP 方法，我们需要测试`/username/{id}`端点。由于此端点不返回任何输出，使用`curl(1)`中的`-v`选项将揭示返回的 HTTP 状态码：

```go
$ curl -X DELETE -H 'Content-Type: application/json' -d '{"username": "admin", "password" : "justChanged"}' localhost:1234/username/4 -v 
```

`HTTP/1.1 200 OK`状态码验证用户已被成功删除。如果我们尝试再次删除同一用户，请求将失败，并返回消息`HTTP/1.1 404 Not Found`。

到目前为止，我们知道 RESTful 服务器按预期工作。然而，`curl(1)`在日常使用 RESTful 服务器时远非完美。下一节将展示如何为 RESTful 服务器开发命令行客户端。

# 创建 RESTful 客户端

创建 RESTful 客户端比编写服务器程序更容易，主要是因为你不需要在客户端与数据库交互。客户端需要做的唯一事情是向服务器发送正确数量和类型的数据，并接收并解释服务器的响应。完整的 RESTful 客户端实现可以在 GitHub 仓库的书籍`./ch11/client`中找到。

支持的第一级 cobra 命令如下：

+   `list`: 此命令访问`/getall`端点并返回所有可用用户的列表。

+   `time`: 此命令用于访问`/time`端点。

+   `update`: 此命令用于更新用户记录——用户 ID 不能更改。

+   `logged`: 此命令列出所有已登录用户。

+   `delete`: 此命令删除现有用户。

+   `login`: 此命令用于登录用户。

+   `logout`: 此命令用于注销用户。

+   `add`: 此命令用于将新用户添加到系统中。

+   `getid`: 此命令返回由用户名标识的用户 ID。

+   `search`: 此命令显示有关给定用户（通过其 ID 识别）的信息。

我们即将展示的客户端比使用 `curl(1)` 工具工作要好得多，因为它可以处理接收到的信息，更重要的是，它可以解释 HTTP 返回码并在发送到服务器之前预处理数据。你付出的代价是开发调试 RESTful 客户端所需额外的时间。

存在两个命令行标志用于传递执行命令的用户名和密码：`username` 和 `password`。正如你将在它们的实现中看到的那样，它们分别有 `-u` 和 `-p` 快捷键。此外，由于包含用户信息的 JSON 记录字段数量较少，所有字段都将使用 `data` 标志或 `-d` 快捷键提供——这是在 `./cmd/root.go` 中实现的。每个命令将只读取所需的标志和输入 JSON 记录的所需字段——这是在每个命令的源代码文件中实现的。最后，当这有意义时，工具将返回 JSON 记录，或者与访问的端点相关的文本消息。现在，让我们继续客户端的结构和命令的实现。

## 创建命令行客户端的结构

本小节使用 cobra 工具来创建命令行工具的结构。但首先，我们将使用 Go 模块创建一个合适的 cobra 项目：

```go
$ cd ~/go/src/github.com/mactsouk/mGo4th/ch11
$ mkdir client
$ cd client
$ go mod init
$ ~/go/bin/cobra init
$ go mod tidy
$ go run main.go 
```

你不需要执行最后一个命令，但它确保到目前为止一切正常。之后，我们准备通过运行以下 cobra 命令来定义工具将支持的命令：

```go
$ ~/go/bin/cobra add add
$ ~/go/bin/cobra add delete
$ ~/go/bin/cobra add list
$ ~/go/bin/cobra add logged
$ ~/go/bin/cobra add login
$ ~/go/bin/cobra add logout
$ ~/go/bin/cobra add search
$ ~/go/bin/cobra add getid
$ ~/go/bin/cobra add time
$ ~/go/bin/cobra add update 
```

现在我们有了所需的结构，我们可以开始实现命令，也许可以移除由 cobra 插入的一些注释，这是下一个小节的主题。

## 实现 RESTful 客户端命令

由于展示整个代码没有意义，我们将展示一些命令中最具代表性的代码，从 `root.go` 开始，这是定义下一个全局变量的地方：

```go
var SERVER string
var PORT string
var data string
var username string
var password string 
```

这些全局变量持有工具的命令行选项值，并且可以在工具代码的任何地方访问。

```go
type User struct {
    ID        int `json:"id"`
    Username  string `json:"username"`
    Password  string `json:"password"`
    LastLogin int64 `json:"lastlogin"`
    Admin     int `json:"admin"`
    Active    int `json:"active"`
} 
```

我们定义了 `User` 结构，用于发送和接收数据。

```go
func init() {
    rootCmd.PersistentFlags().StringVarP(&username, "username", "u", "username", "The username")
    rootCmd.PersistentFlags().StringVarP(&password, "password", "p", "admin", "The password")
    rootCmd.PersistentFlags().StringVarP(&data, "data", "d", "{}", "JSON Record")
    rootCmd.PersistentFlags().StringVarP(&SERVER, "server", "s", "http://localhost", "RESTful server hostname")
    rootCmd.PersistentFlags().StringVarP(&PORT, "port", "P", ":1234", "Port of RESTful Server")
} 
```

我们展示了 `init()` 函数的实现，该函数包含了命令行选项的定义。命令行标志的值会自动存储在作为 `rootCmd.PersistentFlags().StringVarP()` 第一个参数传递的变量中。因此，具有 `-u` 别名的 `username` 标志将它的值存储在 `username` 全局变量中。

下一个部分是实现 `list` 命令，如 `list.go` 中所示：

```go
var listCmd = &cobra.Command{
    Use:   "list",
    Short: "List all available users",
    Long:  `The list command lists all available users.`, 
```

这一部分是关于显示在命令中的帮助信息。尽管它们是可选的，但有一个准确的命令描述是很好的。我们继续实际的实现：

```go
 Run: func(cmd *cobra.Command, args []string) {
        endpoint := "/getall"
        user := User{Username: username, Password: password} 
```

首先，我们构造一个名为`user`的`User`变量，该变量包含发出命令的用户的用户名和密码——`user`变量将被传递到服务器。

```go
 // bytes.Buffer is both a Reader and a Writer
        buf := new(bytes.Buffer)
        err := user.ToJSON(buf)
        if err != nil {
            fmt.Println("JSON:", err)

            os.Exit(1)
        } 
```

在将`user`变量传输到 RESTful 服务器之前，我们需要对其进行编码，这就是`ToJSON()`方法的目的。`ToJSON()`方法的实现可以在`root.go`中找到。

```go
 req, err := http.NewRequest(http.MethodGet,
                               SERVER+PORT+endpoint, buf)
        if err != nil {
            fmt.Println("GetAll – Error in req: ", err)
            return
        }
        req.Header.Set("Content-Type", "application/json") 
```

在这里，我们使用全局变量`SERVER`和`PORT`以及端点创建请求，使用所需的 HTTP 方法（`http.MethodGet`），并声明我们将使用`Header.Set()`语句发送 JSON 数据。

```go
 c := &http.Client{
            Timeout: 15 * time.Second,
        }
        resp, err := c.Do(req)
        if err != nil {
            fmt.Println("Do:", err)
            return
        } 
```

之后，我们使用`Do()`将我们的数据发送到服务器，并获取服务器的响应。

```go
 if resp.StatusCode != http.StatusOK {
            fmt.Println(resp)
            return
        } 
```

如果响应的状态码不是`http.StatusOK`，则请求失败。

```go
 users := []User{}
        SliceFromJSON(&users, resp.Body)
        if err != nil {
            fmt.Println(err)
            return
        }
        data, err := PrettyJSON(users)
        if err != nil {
            fmt.Println(err)
            return
        }
        fmt.Print(data)
    },
} 
```

如果状态码是`http.StatusOK`，那么我们准备读取一个`User`变量的切片。由于这些变量持有 JSON 记录，我们需要使用定义在`root.go`中的`SliceFromJSON()`对其进行解码。

最后是`add`命令的代码，如`add.go`中所示。`add`和`list`之间的区别在于`add`命令需要向 RESTful 服务器发送两个 JSON 记录；第一个记录包含发出命令的用户的数据，第二个记录包含即将被添加到系统中的用户的数据。`username`和`password`标志持有第一个记录的`Username`和`Password`字段的数据，而`data`命令行标志持有第二个记录的数据。

```go
var addCmd = &cobra.Command{
    Use:   "add",
    Short: "Add a new user",
    Long:  `Add a new user to the system.`,
    Run: func(cmd *cobra.Command, args []string) {
        endpoint := "/add"
        u1 := User{Username: username, Password: password} 
```

如前所述，我们获取发出命令的用户信息并将其放入一个结构体中。

```go
 // Convert data string to User Structure
var u2 User
        err := json.Unmarshal([]byte(data), &u2)
        if err != nil {
            fmt.Println("Unmarshal:", err)

            os.Exit(1)
        } 
```

由于`data`命令行标志持有`string`值，我们需要将那个字符串值转换为`User`结构——这就是`json.Unmarshal()`调用的目的。

```go
 users := []User{}
        users = append(users, u1)
        users = append(users, u2) 
```

然后，我们创建一个将要发送到服务器的`User`变量切片。你在这个切片中放置结构的顺序很重要：首先是发出命令的用户，然后是即将创建的用户的数据。这是由 RESTful 服务器决定的。

```go
 buf := new(bytes.Buffer)
        err = SliceToJSON(users, buf)
        if err != nil {
            fmt.Println("JSON:", err)
            return
        } 
```

然后，我们在发送到 RESTful 服务器通过 HTTP 请求之前对那个切片进行编码。

```go
 req, err := http.NewRequest(http.MethodPost,
                                    SERVER+PORT+endpoint, buf)
        if err != nil {
            fmt.Println("GetAll – Error in req: ", err)
            return
        }
        req.Header.Set("Content-Type", "application/json")
        c := &http.Client{
            Timeout: 15 * time.Second,
        }
        resp, err := c.Do(req)
        if err != nil {
            fmt.Println("Do:", err)
            return
        } 
```

我们准备请求并将其发送到服务器。服务器负责解码提供的数据并相应地执行，在这种情况下，通过向系统中添加新用户。客户端只需使用适当的 HTTP 方法（`http.MethodPost`）访问正确的端点，并检查返回的 HTTP 状态码。

```go
 if resp.StatusCode != http.StatusOK {
            fmt.Println("Status code:", resp.Status)
        }
        fmt.Println("User", u2.Username, "added.")
        }
    },
} 
```

`add`命令不会向客户端返回任何数据——我们感兴趣的是 HTTP 状态码，因为这决定了命令的成功或失败。

其余的命令有类似的实现，这里没有展示。请随意查看`cmd`目录中的 Go 源代码文件。

## 使用 RESTful 客户端

现在，我们将使用命令行工具与 RESTful 服务器交互。这种类型的工具可以用于管理 RESTful 服务器、创建自动化任务以及执行 CI/CD 任务。为了简化，客户端和服务器位于同一台机器上，我们主要使用默认用户（`admin`）——这使得展示的命令更短。此外，我们执行 `go build -o rest-cli` 来创建一个二进制可执行文件，以避免始终使用 `go run main.go`。

首先，我们从服务器获取时间：

```go
$ ./rest-cli time
The current time is: Wed, 01 Nov 2023 07:37:49 EET 
```

接下来，我们列出所有用户。由于输出取决于数据库的内容，我们只打印输出的一部分。请注意，`list` 命令需要由具有管理员权限的用户执行：

```go
$ ./rest-cli list -u admin -p admin
[
    {
        "id": 3,
        "username": "mihalis",
        "password": "admin",
        "lastlogin": 0,
        "admin": 0,
        "active": 0
    } 
```

请记住，你应该使用 `admin` 用户的激活密码来确保之前的命令正确执行。在我的情况下，激活密码也是 `admin`，但这取决于你当前数据库的状态。

接下来，我们测试使用无效密码发出的 `logged` 命令：

```go
$ ./rest-cli logged -u admin -p notPass
&{400 Bad Request 400 HTTP/1.1 1 1 map[Content-Length:[0] Date:[Wed, 01 Nov 2023 05:39:38 GMT]] 0x14000204020 0 [] false false map[] 0x14000132c00 <nil>} 
```

如预期的那样，命令失败了——这个输出用于调试目的。在确保命令按预期工作后，你可能想打印一个更合适的错误信息。

之后，我们测试 `add` 命令：

```go
$ ./rest-cli add -u admin -p admin --data '{"Username":"newUser", "Password":"aPass"}'
User newUser added. 
```

再次尝试添加相同的用户将会失败：

```go
$ ./rest-cli add -u admin -p admin --data '{"Username":"newUser", "Password":"aPass"}'
Status code: 400 Bad Request 
```

接下来，我们将删除 `newUser`——但首先，我们需要找到 `newUser` 的用户 ID：

```go
$ ./rest-cli getid -u admin -p admin --data '{"Username":"newUser"}'
User newUser has ID: 4
$ ./rest-cli delete -u admin -p admin --data '{"ID":4}'
User with ID 4 deleted. 
```

随意继续测试 RESTful 客户端，并告诉我你是否发现了任何错误！

## 与多个 REST API 版本一起工作

REST API 可以随时间改变和演变。关于如何实现 REST API 版本化的方法有很多，包括以下几种：

+   使用自定义 HTTP 头（`version-used`）来定义使用的版本

+   为每个版本使用不同的子域名（`v1.servername` 和 `v2.servername`）

+   使用 `Accept` 和 `Content-Type` 头的组合——这种方法基于内容协商

+   为每个版本使用不同的路径（如果 RESTful 服务器支持两个 REST API 版本，则为 `/v1` 和 `/v2`）

+   使用查询参数来引用所需的版本（`..../endpoint?version=v1` 或 `..../endpoint?v=1`）

关于如何实现 REST API 版本化，没有正确答案。使用对你和你的用户来说更自然的方法。重要的是要保持一致并在所有地方使用相同的方法。我个人更喜欢使用 `/v1/...` 来支持版本 1 的端点，以及 `/v2/...` 来支持版本 2 的端点，依此类推。

我们 RESTful 服务器和客户端的开发到此结束。通过本章所提供的内容，你可以创建强大的 RESTful 服务！

# 摘要

Go 被广泛用于开发 RESTful 客户端和服务器，本章展示了如何在 Go 中编写专业的 RESTful 客户端和服务器。虽然你可以使用标准 Go 库开发 RESTful 服务器，但这可能是一项非常繁琐的任务。本章使用的`gorilla/mux`等外部包和 Gin 可以通过提供高级功能来节省你的时间，这些功能如果使用标准 Go 库实现，则需要大量的代码。

记住，定义一个合适的 REST API 并实现相应的服务器和客户端是一个耗时且需要小幅度调整和修改的过程。

一个高效且富有成效的 RESTful 服务背后是正确定义的 JSON 记录和 HTTP 端点，它们支持所需的操作。考虑到这两项内容，Go 代码应该提供服务器和客户端之间交换 JSON 记录的支持。

下一章将介绍代码测试、性能分析、交叉编译和创建示例函数。我们将编写测试本章开发的 HTTP 处理器的代码。

# 练习

+   尝试让`rServer.go`使用 Gin 而不是`net/http`。更新后的`rServer.go`是否仍然与`rClient.go`兼容？为什么？这是好事吗？

+   将`server/restdb.go`文件修改为支持 PostgreSQL 而不是 SQLite。

+   将`server/restdb.go`文件修改为支持 MySQL 而不是 SQLite。

+   将`server/handlers.go`中的处理函数放入单独的文件中。

# 其他资源

+   你可以在[`github.com/gorilla/mux`](https://github.com/gorilla/mux)和[`www.gorillatoolkit.org/pkg/mux`](https://www.gorillatoolkit.org/pkg/mux)了解更多关于`gorilla/mux`的信息。

+   `go-querystring`库用于将 Go 结构编码成 URL 查询参数：[`github.com/google/go-querystring`](https://github.com/google/go-querystring)。

+   教程：使用 Go 和 Gin 开发 RESTful API：[`go.dev/doc/tutorial/web-service-gin`](https://go.dev/doc/tutorial/web-service-gin)。

+   如果你想验证 JSON 输入，可以查看 Go 的`validator`包，链接为[`github.com/go-playground/validator`](https://github.com/go-playground/validator)。

+   当处理 JSON 记录时，你可能觉得`jq(1)`命令行工具非常实用：[`stedolan.github.io/jq/`](https://stedolan.github.io/jq/)和[`jqplay.org/`](https://jqplay.org/)。

# 留下评论！

喜欢这本书吗？通过留下亚马逊评论来帮助像你这样的读者。扫描下面的二维码以获取你选择的免费电子书。

![二维码](img/Review_QR_Code.png)
