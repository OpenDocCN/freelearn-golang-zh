# 第七章：Go 应用程序的微服务

本章将涵盖以下内容：

+   与 Web 处理程序、请求和 ResponseWriter 一起工作

+   使用结构体和闭包进行有状态处理程序

+   验证 Go 结构体和用户输入

+   渲染和内容协商

+   实现和使用中间件

+   构建 reverse proxy 应用程序

+   将 GRPC 导出为 JSON API

# 简介

默认情况下，Go 是编写 Web 应用的绝佳选择。内置的 `net/http` 包与 `html/template` 等包结合使用，可以轻松地构建功能齐全的现代 Web 应用程序。它如此简单，以至于鼓励为甚至基本的长运行应用程序启动 Web 界面。尽管标准库功能齐全，但仍有许多第三方 Web 包，从路由到全栈框架，包括以下内容：

+   [`github.com/urfave/negroni`](https://github.com/urfave/negroni)

+   [`github.com/gin-gonic/gin`](https://github.com/gin-gonic/gin)

+   [`github.com/labstack/echo`](https://github.com/labstack/echo)

+   [`www.gorillatoolkit.org/`](http://www.gorillatoolkit.org/)

+   [`github.com/julienschmidt/httprouter`](https://github.com/julienschmidt/httprouter)

本章中的食谱将侧重于您在处理程序工作时可能遇到的基本任务，当导航响应和请求对象时，以及处理中间件等概念。

# 与 Web 处理程序、请求和 ResponseWriter 一起工作

Go 定义了 `HandlerFuncs` 和 `Handler` 接口，具有以下签名：

```go
// HandlerFunc implements the Handler interface
type HandlerFunc func(http.ResponseWriter, *http.Request)

type Handler interface {
    ServeHTTP(http.ResponseWriter, *http.Request)
}

```

默认情况下，`net/http` 包广泛使用这些类型。例如，可以将路由附加到 `Handler` 或 `HandlerFunc` 接口。本食谱将探讨创建 `Handler` 接口，监听本地端口，并在处理 `http.Request` 后对 `http.ResponseWriter` 接口执行一些操作。这应被视为 Go Web 应用程序和 RESTful API 的基础。

# 准备工作

根据以下步骤配置您的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 在您的操作系统上下载和安装 Go，并配置您的 `GOPATH` 环境变量。

1.  打开一个终端/控制台应用程序。

1.  导航到您的 `GOPATH/src` 并创建一个项目目录，例如 `$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

1.  从 [`curl.haxx.se/download.html`](https://curl.haxx.se/download.html) 安装 curl 命令。

# 如何操作...

这些步骤涵盖了编写和运行应用程序：

1.  在您的终端/控制台应用程序中，创建并导航到 `chapter7/handlers` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter7/handlers`](https://github.com/agtorre/go-cookbook/tree/master/chapter7/handlers) 复制测试或将其作为练习来编写一些自己的代码。

1.  创建一个名为 `get.go` 的文件，并包含以下内容：

```go
        package handlers

        import (
            "fmt"
            "net/http"
        )

        // HelloHandler takes a GET parameter "name" and responds
        // with Hello <name>! in plaintext
        func HelloHandler(w http.ResponseWriter, r *http.Request) {
            w.Header().Set("Content-Type", "text/plain")
            if r.Method != http.MethodGet {
                w.WriteHeader(http.StatusMethodNotAllowed)
                return
            }
            name := r.URL.Query().Get("name")

            w.WriteHeader(http.StatusOK)
            w.Write([]byte(fmt.Sprintf("Hello %s!", name)))
        }

```

1.  创建一个名为 `post.go` 的文件，并包含以下内容：

```go
        package handlers

        import (
            "encoding/json"
            "net/http"
        )

        // GreetingResponse is the JSON Response that
        // GreetingHandler returns
        type GreetingResponse struct {
            Payload struct {
                Greeting string `json:"greeting,omitempty"`
                Name string `json:"name,omitempty"`
                Error string `json:"error,omitempty"`
            } `json:"payload"`
            Successful bool `json:"successful"`
        }

        // GreetingHandler returns a GreetingResponse which either has 
        // errors or a useful payload
        func GreetingHandler(w http.ResponseWriter, r *http.Request) {
            w.Header().Set("Content-Type", "application/json")
            if r.Method != http.MethodPost {
                w.WriteHeader(http.StatusMethodNotAllowed)
                return
            }
            var gr GreetingResponse
            if err := r.ParseForm(); err != nil {
                gr.Payload.Error = "bad request"
                if payload, err := json.Marshal(gr); err == nil {
                    w.Write(payload)
                }
            }
            name := r.FormValue("name")
            greeting := r.FormValue("greeting")

            w.WriteHeader(http.StatusOK)
            gr.Successful = true
            gr.Payload.Name = name
            gr.Payload.Greeting = greeting
            if payload, err := json.Marshal(gr); err == nil {
               w.Write(payload)
            }
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `handlers` 导入修改为你在第二步中设置的路径：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter7/handlers"
        )

        func main() {
            http.HandleFunc("/name", handlers.HelloHandler)
            http.HandleFunc("/greeting", handlers.GreetingHandler)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run main.go`.

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Listening on port :3333

```

1.  在另一个终端中，运行以下命令：

```go
 curl "http://localhost:3333/name?name=Reader" -X GET      curl "http://localhost:3333/greeting" -X POST -d 
      'name=Reader;greeting=Goodbye'

```

你将看到以下输出：

```go
 $curl "http://localhost:3333/name?name=Reader" -X GET 
 Hello Reader!

 $curl "http://localhost:3333/greeting" -X POST -d   
      'name=Reader;greeting=Goodbye' 
 {"payload":
      {"greeting":"Goodbye","name":"Reader"},"successful":true}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

对于这个菜谱，我们设置了两个处理器。第一个处理器期望一个带有名为 `name` 的 `GET` 参数的 `GET` 请求。当我们使用 curl 访问它时，它返回纯文本字符串 `Hello <name>!`

第二个处理器期望一个带有 `PostForm` 请求的 `POST` 方法。如果你使用了一个标准的 HTML 表单而没有任何 AJAX 调用，你将得到这个结果。或者，我们也可以从请求体中解析 JSON。我建议你也尝试这个练习。最后，处理器发送一个 JSON 格式的响应并设置所有适当的头信息。

尽管所有这些都被明确写出，但有许多方法可以使代码更简洁，如下所示：

+   使用 [`github.com/unrolled/render`](https://github.com/unrolled/render) 来处理响应

+   使用本章 *与 Web 处理器、请求和 ResponseWriters 协同工作* 菜单中提到的各种 Web 框架来解析路由参数，限制路由到特定的 HTTP 动词，处理优雅的关闭，等等

# 使用结构和闭包进行有状态的处理

由于 HTTP 处理器函数的签名稀疏，可能看起来很难向处理器添加状态。例如，有各种方法来包含数据库连接。实现这一目标的两种方法是通过闭包传递状态，这对于单个处理器的灵活性很有用，或者通过使用结构体。

这个菜谱将演示这两者。我们将使用一个结构控制器来存储存储接口，并创建两个由外部函数修改的单个处理器路由。

# 准备工作

参考本章 *准备工作* 部分的 *与 Web 处理器、请求和 ResponseWriters 协同工作* 菜单中的步骤。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序，创建并导航到 `chapter7/rest` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter7/controllers`](https://github.com/agtorre/go-cookbook/tree/master/chapter7/controllers) 复制测试或将其作为练习来编写一些自己的。

1.  创建一个名为 `controller.go` 的文件，并包含以下内容：

```go
        package controllers

        // Controller passes state to our handlers
        type Controller struct {
            storage Storage
        }

        // New is a Controller 'constructor'
        func New(storage Storage) *Controller {
            return &Controller{
                storage: storage,
            }
        }

        // Payload is our common response
        type Payload struct {
            Value string `json:"value"`
        }

```

1.  创建一个名为 `storage.go` 的文件，并包含以下内容：

```go
        package controllers

        // Storage Interface Supports Get and Put
        // of a single value
        type Storage interface {
            Get() string
            Put(string)
        }

        // MemStorage implements Storage
        type MemStorage struct {
            value string
        }

        // Get our in-memory value
        func (m *MemStorage) Get() string {
            return m.value
        }

        // Put our in-memory value
        func (m *MemStorage) Put(s string) {
            m.value = s
        }

```

1.  创建一个名为 `post.go` 的文件，并包含以下内容：

```go
        package controllers

        import (
            "encoding/json"
            "net/http"
        )

        // SetValue modifies the underlying storage of the controller 
        // object
        func (c *Controller) SetValue(w http.ResponseWriter, r 
        *http.Request) {
            if r.Method != http.MethodPost {
                w.WriteHeader(http.StatusMethodNotAllowed)
                return
            }
            if err := r.ParseForm(); err != nil {
                w.WriteHeader(http.StatusInternalServerError)
                return
            }
            value := r.FormValue("value")
            c.storage.Put(value)
            w.WriteHeader(http.StatusOK)
            p := Payload{Value: value}
            if payload, err := json.Marshal(p); err == nil {
                w.Write(payload)
            }

        }

```

1.  创建一个名为 `get.go` 的文件，并包含以下内容：

```go
        package controllers

        import (
            "encoding/json"
            "net/http"
        )

        // GetValue is a closure that wraps a HandlerFunc, if 
        // UseDefault is true value will always be "default" else it'll 
        // be whatever is stored in storage
        func (c *Controller) GetValue(UseDefault bool) http.HandlerFunc 
        {
            return func(w http.ResponseWriter, r *http.Request) {
                w.Header().Set("Content-Type", "application/json")
                if r.Method != http.MethodGet {
                    w.WriteHeader(http.StatusMethodNotAllowed)
                    return
                }
                value := "default"
                if !UseDefault {
                    value = c.storage.Get()
                }
                w.WriteHeader(http.StatusOK)
                p := Payload{Value: value}
                if payload, err := json.Marshal(p); err == nil {
                    w.Write(payload)
                }
            }
        }

```

1.  创建一个名为 `example` 的新目录，并导航到该目录。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `controllers` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter7/controllers"
        )

        func main() {
            storage := controllers.MemStorage{}
            c := controllers.New(&storage)
            http.HandleFunc("/get", c.GetValue(false))
            http.HandleFunc("/get/default", c.GetValue(true))
            http.HandleFunc("/set", c.SetValue)

            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Listening on port :3333

```

1.  在另一个终端中，运行以下命令：

```go
 curl "http://localhost:3333/set -X POST -d "value=value" 
 curl "http://localhost:3333/get -X GET 
 curl "http://localhost:3333/get/default -X GET

```

你将看到以下输出：

```go
 $curl "http://localhost:3333/set -X POST -d "value=value"
 {"value":"value"}

 $curl "http://localhost:3333/get -X GET 
 {"value":"value"}

 $curl "http://localhost:3333/get/default -X GET 
 {"value":"default"}

```

1.  如果你复制或编写了自己的测试，请向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

这些策略之所以有效，是因为 Go 允许方法满足诸如 `http.HandlerFunc` 这样的类型函数。通过使用结构体，我们可以在 `main.go` 中注入各种组件，这可能包括数据库连接、日志记录等。在这个配方中，我们插入了一个 `Storage` 接口。所有连接到控制器的处理程序都可以使用其方法和属性。

`GetValue` 方法没有 `http.HandlerFunc` 签名，而是返回它。这就是我们如何使用闭包来注入状态。在 `main.go` 中，我们定义了两个路由，一个将 `UseDefault` 设置为 `false`，另一个设置为 `true`。这可以在定义跨越多个路由的函数时使用，或者在使用结构体时，你的处理程序感觉过于繁琐。

# 验证 Go 结构体和用户输入

网页验证可能是一个难题。这个配方将探讨使用闭包来支持轻松模拟验证函数，并允许在初始化控制器结构体时（如前一个配方所述）进行灵活的验证类型。

我们将在结构体上执行此验证，但不探讨如何填充结构体。我们可以假设数据将通过解析 JSON 有效负载、显式地从表单输入填充或其他方法来填充。

# 准备工作

参考在 *Working with web handlers, requests, and ResponseWriters* 配方的 *Getting ready* 部分中给出的步骤。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter7/validation` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter7/validation`](https://github.com/agtorre/go-cookbook/tree/master/chapter7/validation) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `controller.go` 的文件，并包含以下内容：

```go
        package validation

        // Controller holds our validation functions
        type Controller struct {
            ValidatePayload func(p *Payload) error
        }

        // New initializes a controller with our
        // local validation, it can be overwritten
        func New() *Controller {
            return &Controller{
                ValidatePayload: ValidatePayload,
            }
        }

```

1.  创建一个名为 `validate.go` 的文件，并包含以下内容：

```go
        package validation

        import "errors"

        // Verror is an error that occurs
        // during validation, we can
        // return this to a user
        type Verror struct {
            error
        }

        // Payload is the value we
        // process
        type Payload struct {
            Name string `json:"name"`
            Age int `json:"age"`
        }

        // ValidatePayload is 1 implementation of
        // the closure in our controller
        func ValidatePayload(p *Payload) error {
            if p.Name == "" {
                return Verror{errors.New("name is required")}
            }

            if p.Age <= 0 || p.Age >= 120 {
                return Verror{errors.New("age is required and must be a 
                value greater than 0 and less than 120")}
            }
            return nil
        }

```

1.  创建一个名为 `process.go` 的文件，并包含以下内容：

```go
        package validation

        import (
            "encoding/json"
            "fmt"
            "net/http"
        )

        // Process is a handler that validates a post payload
        func (c *Controller) Process(w http.ResponseWriter, r 
        *http.Request) {
            if r.Method != http.MethodPost {
                w.WriteHeader(http.StatusMethodNotAllowed)
                return
            }

            decoder := json.NewDecoder(r.Body)
            defer r.Body.Close()
            var p Payload

            if err := decoder.Decode(&p); err != nil {
                fmt.Println(err)
                w.WriteHeader(http.StatusBadRequest)
                return
            }

            if err := c.ValidatePayload(&p); err != nil {
                switch err.(type) {
                case Verror:
                    w.WriteHeader(http.StatusBadRequest)
                    // pass the Verror along
                    w.Write([]byte(err.Error()))
                    return
                default:
                    w.WriteHeader(http.StatusInternalServerError)
                    return
                }
            }
        }

```

1.  创建一个名为 `example` 的新目录，并导航到该目录。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `validation` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter7/validation"
        )

        func main() {
            c := validation.New()
            http.HandleFunc("/", c.Process)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Listening on port :3333

```

1.  在另一个终端中运行以下命令：

```go
 curl "http://localhost:3333/-X POST -d '{}' curl "http://localhost:3333/-X POST -d '{"name":"test"}'      curl "http://localhost:3333/-X POST -d '{"name":"test",
      "age": 5}'  

```

你应该看到以下输出：

```go
 $curl "http://localhost:3333/-X POST -d '{}'
      name is required

 $curl "http://localhost:3333/-X POST -d '{"name":"test"}'
 age is required and must be a value greater than 0 and 
      less than 120

 $curl "http://localhost:3333/-X POST -d '{"name":"test",
      "age": 5}' -v

 <lots of output, should contain a 200 OK status code>

```

1.  如果你复制或编写了自己的测试用例，向上移动一个目录并运行 `go test`。确保所有测试通过。

# 它是如何工作的...

我们通过向我们的控制器结构体传递闭包来处理验证。对于控制器可能需要验证的任何输入，我们都需要这些闭包之一。这种方法的优势在于我们可以在运行时模拟和替换验证函数，从而使测试变得简单得多。此外，我们不受单一函数签名的限制，我们可以传递诸如数据库连接之类的信息给验证函数。

此配方还展示了返回一个名为 `Verror` 的类型化错误。此类型包含可以显示给用户的验证错误消息。这种方法的一个缺点是它不能同时处理多个验证消息。这可以通过修改 `Verror` 类型来实现，允许更多的状态，例如，通过包括一个映射，以便在从我们的 `ValidatePayload` 函数返回之前存储多个验证错误。

# 渲染和内容协商

Web 处理器可以返回各种内容类型，例如，它们可以返回 JSON、纯文本、图像等。在与 API 通信时，通常可以指定和接受内容类型，以明确你将传递数据的格式以及你希望接收的数据格式。

此配方将探讨使用 unrolled/render 和自定义函数来协商内容类型并相应地响应。

# 准备工作

根据以下步骤配置你的环境：

1.  参考配方 *Working with web handlers, requests, and ResponseWriters* 中的 *Getting ready* 部分的步骤。

1.  运行 `go get github.com/unrolled/render` 命令。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建并导航到 `chapter7/negotiate` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter7/negotiate`](https://github.com/agtorre/go-cookbook/tree/master/chapter7/negotiate) 复制测试用例，或者将此作为练习来编写你自己的代码。

1.  创建一个名为 `negotiate.go` 的文件，并包含以下内容：

```go
        package negotiate

        import (
            "net/http"

            "github.com/unrolled/render"
        )

        // Negotiator wraps render and does
        // some switching on ContentType
        type Negotiator struct {
            ContentType string
            *render.Render
        }

        // GetNegotiator takes a request, and figures
        // out the ContentType from the Content-Type header
        func GetNegotiator(r *http.Request) *Negotiator {
            contentType := r.Header.Get("Content-Type")

            return &Negotiator{
                ContentType: contentType,
                Render: render.New(),
            }
        }

```

1.  创建一个名为 `respond.go` 的文件，并包含以下内容：

```go
        package negotiate

        import "io"
        import "github.com/unrolled/render"

        // Respond switches on Content Type to determine
        // the response
        func (n *Negotiator) Respond(w io.Writer, status int, v 
        interface{}) {
            switch n.ContentType {
                case render.ContentJSON:
                    n.Render.JSON(w, status, v)
                case render.ContentXML:
                    n.Render.XML(w, status, v)
                default:
                    n.Render.JSON(w, status, v)
                }
        }

```

1.  创建一个名为 `handler.go` 的文件，并包含以下内容：

```go
        package negotiate

        import (
            "encoding/xml"
            "net/http"
        )

        // Payload defines it's layout in xml and json
        type Payload struct {
            XMLName xml.Name `xml:"payload" json:"-"`
            Status string `xml:"status" json:"status"`
        }

        // Handler gets a negotiator using the request,
        // then renders a Payload
        func Handler(w http.ResponseWriter, r *http.Request) {
            n := GetNegotiator(r)

            n.Respond(w, http.StatusOK, &Payload{Status:       
            "Successful!"})
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将协商导入修改为使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter7/negotiate"
        )

        func main() {
            http.HandleFunc("/", negotiate.Handler)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Listening on port :3333

```

1.  在另一个终端中运行以下命令：

```go
 curl "http://localhost:3333 -H "Content-Type: text/xml" curl "http://localhost:3333 -H "Content-Type: application/json"

```

你将看到以下输出：

```go
 $curl "http://localhost:3333 -H "Content-Type: text/xml"
      <payload><status>Successful!</status></payload> 
 $curl "http://localhost:3333 -H "Content-Type: application/json"
 {"status":"Successful!"}

```

1.  如果你复制或编写了自己的测试用例，向上移动一个目录并运行 `go test`。确保所有测试通过。

# 它是如何工作的...

`github.com/unrolled/render` 包为此菜谱做了大量工作。如果你需要处理 HTML 模板等，你可以输入大量的其他选项。此菜谱可用于在处理网络处理程序时自动协商，如在此处通过传递各种内容类型标题或直接操作结构体所示。

可以将类似的模式应用于接受标题，但请注意，这个标题通常包含多个值，并且你的代码必须考虑到这一点。

# 实现和使用中间件

Go 中的处理程序中间件是一个被广泛探讨的领域。有各种处理中间件的包。此菜谱将从头开始创建中间件并实现一个 `ApplyMiddleware` 函数来链接一系列中间件。

它还将探索在请求上下文对象中设置值并在稍后使用中间件检索它们。所有这些都将使用一个非常基本的处理程序来完成，以帮助演示如何将中间件逻辑与你的处理程序解耦。

# 准备工作

请参考 *与网络处理程序、请求和 ResponseWriters 一起工作* 菜谱中的 *准备工作* 部分给出的步骤。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter7/middleware` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter7/middleware`](https://github.com/agtorre/go-cookbook/tree/master/chapter7/middleware) 复制测试或将其用作练习来编写一些你自己的测试。

1.  创建一个名为 `middleware.go` 的文件，并包含以下内容：

```go
        package middleware

        import (
            "log"
            "net/http"
            "time"
        )

        // Middleware is what all middleware functions will return
        type Middleware func(http.HandlerFunc) http.HandlerFunc

        // ApplyMiddleware will apply all middleware, the last 
        // arguments will be the
        // outer wrap for context passing purposes
        func ApplyMiddleware(h http.HandlerFunc, middleware 
        ...Middleware) http.HandlerFunc {
            applied := h
            for _, m := range middleware {
                applied = m(applied)
            }
            return applied
        }

        // Logger logs requests, this will use an id passed in via
        // SetID()
        func Logger(l *log.Logger) Middleware {
            return func(next http.HandlerFunc) http.HandlerFunc {
                return func(w http.ResponseWriter, r *http.Request) {
                    start := time.Now()
                    l.Printf("started request to %s with id %s", r.URL, 
                    GetID(r.Context()))
                    next(w, r)
                    l.Printf("completed request to %s with id %s in
                    %s", r.URL, GetID(r.Context()), time.Since(start))
                }
            }
        }

```

1.  创建一个名为 `context.go` 的文件，并包含以下内容：

```go
        package middleware

        import (
            "context"
            "net/http"
            "strconv"
        )

        // ContextID is our type to retrieve our context
        // objects
        type ContextID int

        // ID is the only ID we've defined
        const ID ContextID = 0

        // SetID updates context with the id then
        // increments it
        func SetID(start int64) Middleware {
            return func(next http.HandlerFunc) http.HandlerFunc {
                return func(w http.ResponseWriter, r *http.Request) {
                    ctx := context.WithValue(r.Context(), ID, 
                    strconv.FormatInt(start, 10))
                    start++
                    r = r.WithContext(ctx)
                    next(w, r)
                }
            }
        }

        // GetID grabs an ID from a context if set
        // otherwise it returns an empty string
        func GetID(ctx context.Context) string {
            if val, ok := ctx.Value(ID).(string); ok {
                return val
            }
            return ""
        }

```

1.  创建一个名为 `handler.go` 的文件，并包含以下内容：

```go
        package middleware

        import (
            "net/http"
        )

        // Handler is very basic
        func Handler(w http.ResponseWriter, r *http.Request) {
            w.WriteHeader(http.StatusOK)
            w.Write([]byte("success"))
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `middleware` 导入修改为你在第 2 步中设置的路径：

```go
        package main

        import (
            "fmt"
            "log"
            "net/http"
            "os"

            "github.com/agtorre/go-cookbook/chapter7/middleware"
        )

        func main() {
            // We apply from bottom up
            h := middleware.ApplyMiddleware(
            middleware.Handler,
            middleware.Logger(log.New(os.Stdout, "", 0)),
            middleware.SetID(100),
            ) 
            http.HandleFunc("/", h)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Listening on port :3333

```

1.  在另一个终端中，多次运行以下 curl 命令：

```go
 curl "http://localhost:3333

```

你将看到以下输出：

```go
 $curl "http://localhost:3333
 success

 $curl "http://localhost:3333
 success

 $curl "http://localhost:3333
 success

```

1.  在原始的 `main.go` 中，你应该看到以下：

```go
 Listening on port :3333
 started request to / with id 100
 completed request to / with id 100 in 52.284µs
 started request to / with id 101
 completed request to / with id 101 in 40.273µs
 started request to / with id 102

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

中间件可以用于执行简单的操作，如日志记录、指标收集和分析。它还可以用于在每个请求上动态填充变量。例如，我们可以从请求中收集 X-Header 来设置 ID 或生成 ID，就像我们在本菜谱中所做的那样。另一种 ID 策略可能是为每个请求生成一个 UUID--这使我们能够轻松地将日志消息关联起来，并在涉及多个微服务构建响应的情况下跟踪你的请求。

当处理上下文值时，考虑中间件的顺序很重要。通常，最好不要让中间件相互依赖。例如，在本食谱中，可能最好在日志中间件本身生成 UUID。然而，本食谱应作为分层中间件和初始化它们的 `main.go` 的指南。

# 构建反向代理应用程序

在本食谱中，我们将开发一个反向代理应用程序。其想法是，通过在浏览器中点击 `http://localhost:3333`，所有流量都将转发到可配置的主机，响应将转发到你的浏览器。最终结果应该是通过我们的代理应用程序在浏览器中渲染的 [`www.golang.org`](https://www.golang.org)。

这可以与端口转发和 ssh 隧道结合使用，以便通过中间服务器安全地访问网站。本食谱将从零开始构建反向代理，但此功能也由 `net/http/httputil` 包提供。使用此包，可以通过 `Director func(*http.Request)` 修改传入的请求，并通过 `ModifyResponse func(*http.Response) error` 修改传出的响应。此外，还支持缓冲响应。

# 准备就绪

参考文档中“准备就绪”部分的步骤。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter7/proxy` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter7/proxy`](https://github.com/agtorre/go-cookbook/tree/master/chapter7/proxy) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `proxy.go` 的文件，并包含以下内容：

```go
        package proxy

        import (
            "log"
            "net/http"
        )

        // Proxy holds our configured client
        // and BaseURL to proxy to
        type Proxy struct {
            Client *http.Client
            BaseURL string
        }

        // ServeHTTP means that proxy implments the Handler interface
        // It manipulates the request, forwards it to BaseURL, then 
        // returns the response
        func (p *Proxy) ServeHTTP(w http.ResponseWriter, r 
        *http.Request) {
            if err := p.ProcessRequest(r); err != nil {
                log.Printf("error occurred during process request: %s", 
                err.Error())
                w.WriteHeader(http.StatusBadRequest)
                return
            }

            resp, err := p.Client.Do(r)
            if err != nil {
                log.Printf("error occurred during client operation: 
                %s", err.Error())
                w.WriteHeader(http.StatusInternalServerError)
                return
            }
            defer resp.Body.Close()
            CopyResponse(w, resp)
        }

```

1.  创建一个名为 `process.go` 的文件，并包含以下内容：

```go
        package proxy

        import (
            "bytes"
            "net/http"
            "net/url"
        )

        // ProcessRequest modifies the request in accordnance
        // with Proxy settings
        func (p *Proxy) ProcessRequest(r *http.Request) error {
            proxyURLRaw := p.BaseURL + r.URL.String()

            proxyURL, err := url.Parse(proxyURLRaw)
            if err != nil {
                return err
            }
            r.URL = proxyURL
            r.Host = proxyURL.Host
            r.RequestURI = ""
            return nil
        }

        // CopyResponse takes the client response and writes everything
        // to the ResponseWriter in the original handler
        func CopyResponse(w http.ResponseWriter, resp *http.Response) {
            var out bytes.Buffer
            out.ReadFrom(resp.Body)

            for key, values := range resp.Header {
                for _, value := range values {
                w.Header().Add(key, value)
                }
            }

            w.WriteHeader(resp.StatusCode)
            w.Write(out.Bytes())
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `proxy` 导入修改为使用你在第 2 步中设置的路径：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter7/proxy"
        )

        func main() {
            p := &proxy.Proxy{
                Client: http.DefaultClient,
                BaseURL: "https://www.golang.org",
            }
            http.Handle("/", p)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Listening on port :3333

```

1.  使用浏览器导航到 `localhost:3333/`。你应该能看到 [`golang.org/`](https://golang.org/) 网站被渲染出来！

1.  如果你复制或编写了自己的测试，请进入上一级目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

Go 请求和响应对象在客户端和处理器之间大部分是可共享的。此代码通过满足 `Handler` 接口的 `Proxy` 结构体获取请求。`main.go` 文件使用 `Handle` 而不是其他地方使用的 `HandleFunc`。一旦请求可用，它就被修改为为请求添加 `Proxy.BaseURL` 前缀，然后客户端将其分派。最后，响应被复制回 `ResponseWriter` 接口。这包括所有头部、正文和状态。

我们还可以添加一些额外的功能，例如基本身份验证、令牌管理等等，如果需要的话。这对于令牌管理很有用，其中代理管理 JavaScript 或其他客户端应用程序的会话。

# 将 GRPC 作为 JSON API 导出

在第六章 *理解 GRPC 客户端* 的配方中，*Web 客户端和 API*，我们编写了一个基本的 GRPC 服务器和客户端。这个配方将在此基础上扩展，通过将常见的 RPC 函数放在一个包中，并将它们包装在 GRPC 服务器和标准网络处理器中。当你的 API 想要支持这两种类型的客户端，但又不想为常见功能重复代码时，这可能很有用。

# 准备就绪

根据以下步骤配置你的环境：

1.  请参考 *与网络处理器、请求和 ResponseWriters 一起工作* 配方中的 *准备就绪* 部分的步骤。

1.  从 [`github.com/grpc/grpc/blob/master/INSTALL.md`](https://github.com/grpc/grpc/blob/master/INSTALL.md) 安装 GRPC。

1.  运行 `go get github.com/golang/protobuf/proto` 命令。

1.  运行 `go get github.com/golang/protobuf/protoc-gen-go` 命令。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter7/grpcjson` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter7/grpcjson`](https://github.com/agtorre/go-cookbook/tree/master/chapter7/grpcjson) 复制测试，或者将其作为练习编写一些自己的代码。

1.  创建一个名为 `keyvalue` 的新目录，并导航到它。

1.  创建一个名为 `keyvalue.proto` 的文件，并包含以下内容：

```go
        syntax = "proto3";

        package keyvalue;

        service KeyValue{
            rpc Set(SetKeyValueRequest) returns (KeyValueResponse){}
            rpc Get(GetKeyValueRequest) returns (KeyValueResponse){}
        }

        message SetKeyValueRequest {
            string key = 1;
            string value = 2;
        }

        message GetKeyValueRequest{
            string key = 1;
        }

        message KeyValueResponse{
            string success = 1;
            string value = 2;
        }

```

1.  运行以下命令：

```go
 protoc --go_out=plugins=grpc:. keyvalue.proto

```

1.  向上移动一个目录。

1.  创建一个名为 `internal` 的新目录。

1.  创建一个名为 `internal/keyvalue.go` 的文件，并包含以下内容：

```go
        package internal

        import (
            "golang.org/x/net/context"
            "sync"

            "github.com/agtorre/go-cookbook/chapter7/grpcjson/keyvalue"
            "google.golang.org/grpc"
            "google.golang.org/grpc/codes"
        )

        // KeyValue is a struct that holds a map
        type KeyValue struct {
            mutex sync.RWMutex
            m map[string]string
        }

        // NewKeyValue initializes the map and controller
        func NewKeyValue() *KeyValue {
            return &KeyValue{
                m: make(map[string]string),
            }
        }

        // Set sets a value to a key, then returns the value
        func (k *KeyValue) Set(ctx context.Context, r 
        *keyvalue.SetKeyValueRequest) (*keyvalue.KeyValueResponse, 
        error) {
            k.mutex.Lock()
            k.m[r.GetKey()] = r.GetValue()
            k.mutex.Unlock()
            return &keyvalue.KeyValueResponse{Value: r.GetValue()}, nil
        }

        // Get gets a value given a key, or say not found if 
        // it doesn't exist
        func (k *KeyValue) Get(ctx context.Context, r 
        *keyvalue.GetKeyValueRequest) (*keyvalue.KeyValueResponse, 
        error) {
            k.mutex.RLock()
            defer k.mutex.RUnlock()
            val, ok := k.m[r.GetKey()]
            if !ok {
                return nil, grpc.Errorf(codes.NotFound, "key not set")
            }
            return &keyvalue.KeyValueResponse{Value: val}, nil
        }

```

1.  创建一个名为 `grpc` 的新目录。

1.  创建一个名为 `grpc/main.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "fmt"
            "net"

            "github.com/agtorre/go-cookbook/chapter7/grpcjson/internal"
            "github.com/agtorre/go-cookbook/chapter7/grpcjson/keyvalue"
            "google.golang.org/grpc"
        )

        func main() {
            grpcServer := grpc.NewServer()
            keyvalue.RegisterKeyValueServer(grpcServer, 
            internal.NewKeyValue())
            lis, err := net.Listen("tcp", ":4444")
            if err != nil {
                panic(err)
            }
            fmt.Println("Listening on port :4444")
            grpcServer.Serve(lis)
        }

```

1.  创建一个名为 `http` 的新目录。

1.  创建一个名为 `http/set.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "encoding/json"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter7/grpcjson/internal"
            "github.com/agtorre/go-cookbook/chapter7/grpcjson/keyvalue"
            "github.com/apex/log"
        )

        // Controller holds an internal KeyValueObject
        type Controller struct {
            *internal.KeyValue
        }

        // SetHandler wraps or GRPC Set
        func (c *Controller) SetHandler(w http.ResponseWriter, r 
        *http.Request) {
            var kv keyvalue.SetKeyValueRequest

            decoder := json.NewDecoder(r.Body)
            if err := decoder.Decode(&kv); err != nil {
                log.Errorf("failed to decode: %s", err.Error())
                w.WriteHeader(http.StatusBadRequest)
                return
            }

            gresp, err := c.Set(r.Context(), &kv)
            if err != nil {
                log.Errorf("failed to set: %s", err.Error())
                w.WriteHeader(http.StatusInternalServerError)
                return
            }

            resp, err := json.Marshal(gresp)
            if err != nil {
                log.Errorf("failed to marshal: %s", err.Error())
                w.WriteHeader(http.StatusInternalServerError)
                return
            }
            w.WriteHeader(http.StatusOK)
            w.Write(resp)
        }

```

1.  创建一个名为 `http/get.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "encoding/json"
            "net/http"

            "google.golang.org/grpc"
            "google.golang.org/grpc/codes"

            "github.com/agtorre/go-cookbook/chapter7/grpcjson/keyvalue"
            "github.com/apex/log"
        )

        // GetHandler wraps our RPC Get call
        func (c *Controller) GetHandler(w http.ResponseWriter, r 
        *http.Request) {
            key := r.URL.Query().Get("key")
            kv := keyvalue.GetKeyValueRequest{Key: key}

            gresp, err := c.Get(r.Context(), &kv)
            if err != nil {
                if grpc.Code(err) == codes.NotFound {
                    w.WriteHeader(http.StatusNotFound)
                    return
                }
                log.Errorf("failed to get: %s", err.Error())
                w.WriteHeader(http.StatusInternalServerError)
                return
            }

            w.WriteHeader(http.StatusOK)
            resp, err := json.Marshal(gresp)
            if err != nil {
                log.Errorf("failed to marshal: %s", err.Error())
                w.WriteHeader(http.StatusInternalServerError)
                return
            }
            w.Write(resp)
        }

```

1.  创建一个名为 `http/main.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter7/grpcjson/internal"
        )

        func main() {
            c := Controller{KeyValue: internal.NewKeyValue()}
            http.HandleFunc("/set", c.SetHandler)
            http.HandleFunc("/get", c.GetHandler)

            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run http/*.go` 命令。

你应该看到以下输出：

```go
 $ go run http/*.go
 Listening on port :3333

```

1.  在另一个终端中运行以下命令：

```go
 curl "http://localhost:3333/set" -d '{"key":"test", 
      "value":"123"}' -v curl "http://localhost:3333/get?key=badtest" -v curl "http://localhost:3333/get?key=test" -v

```

你应该看到以下输出：

```go
 $curl "http://localhost:3333/set" -d '{"key":"test", 
      "value":"123"}' -v
 {"value":"123"}

 $curl "http://localhost:3333/get?key=badtest" -v  
      'name=Reader;greeting=Goodbye' 
 <should return a 404>

 $curl "http://localhost:3333/get?key=test" -v 
      'name=Reader;greeting=Goodbye' 
 {"value":"123"}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

虽然这个配方省略了客户端，但你可以在第六章的*理解 gRPC 客户端*配方中复制这些步骤，*Web 客户端和 API*，你应该会看到与我们使用 curl 看到的结果完全相同。`http`和`grpc`目录都使用了相同的内部包。在这个包中，我们必须小心返回适当的 gRPC 错误代码，并将这些错误代码正确映射到我们的 HTTP 响应。在这种情况下，我们使用`codes.NotFound`，将其映射到`http.StatusNotFound`。如果你必须处理多个错误，使用`switch`语句可能比使用`if…else`语句更有意义。

你可能还会注意到，gRPC 签名通常非常一致。它们接收一个请求并返回一个可选的响应和一个错误。如果你的 gRPC 调用足够重复，你可以创建一个通用的处理程序适配器，而且这也似乎非常适合代码生成，你最终可能会在像`github.com/goadesign/goa`这样的包中看到类似的东西。
