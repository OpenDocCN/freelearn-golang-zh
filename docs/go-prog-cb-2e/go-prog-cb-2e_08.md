# 第八章：Go 应用程序的微服务

Go 是一个编写 Web 应用程序的绝佳选择。内置的`net/http`包结合`html/template`等包，可以实现现代完整功能的 Web 应用程序。它如此简单，以至于它鼓励为管理甚至是基本的长期运行的应用程序启动 Web 界面。尽管标准库功能齐全，但仍然有大量第三方 Web 包，涵盖从路由到全栈框架的各种功能，包括以下内容：

+   [`github.com/urfave/negroni`](https://github.com/urfave/negroni)

+   [`github.com/gin-gonic/gin`](https://github.com/gin-gonic/gin)

+   [`github.com/labstack/echo`](https://github.com/labstack/echo)

+   [`www.gorillatoolkit.org/`](http://www.gorillatoolkit.org/)

+   [`github.com/julienschmidt/httprouter`](https://github.com/julienschmidt/httprouter)

本章的食谱将侧重于处理程序、响应和请求对象以及处理中间件等概念时可能遇到的基本任务。

在本章中，将涵盖以下食谱：

+   处理 web 处理程序、请求和 ResponseWriter 实例

+   使用结构和闭包进行有状态处理程序

+   验证 Go 结构和用户输入的输入

+   渲染和内容协商

+   实现和使用中间件

+   构建一个反向代理应用程序

+   将 GRPC 导出为 JSON API

# 技术要求

为了继续本章中的所有食谱，根据以下步骤配置您的环境：

1.  从[`golang.org/doc/install`](https://golang.org/doc/install)下载并安装 Go 1.12.6 或更高版本到您的操作系统上。

1.  打开终端或控制台应用程序；创建一个项目目录，例如`~/projects/go-programming-cookbook`，并导航到该目录。所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，或者选择从该目录工作，而不是手动输入示例，如下所示：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

1.  从[`curl.haxx.se/download.html`](https://curl.haxx.se/download.html)安装`curl`命令。

# 处理 web 处理程序、请求和 ResponseWriter 实例

Go 定义了具有以下签名的`HandlerFunc`和`Handler`接口：

```go
// HandlerFunc implements the Handler interface
type HandlerFunc func(http.ResponseWriter, *http.Request)

type Handler interface {
    ServeHTTP(http.ResponseWriter, *http.Request)
}
```

默认情况下，`net/http`包广泛使用这些类型。例如，路由可以附加到`Handler`或`HandlerFunc`接口。本教程将探讨创建`Handler`接口，监听本地端口，并在处理`http.Request`后对`http.ResponseWriter`接口执行一些操作。这应该被视为 Go Web 应用程序和 RESTful API 的基础。

# 如何做...

以下步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter8/handlers`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/handlers 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/handlers    
```

1.  从`~/projects/go-programming-cookbook-original/chapter8/handlers`复制测试，或者使用这个作为练习来编写一些自己的代码！

1.  创建一个名为`get.go`的文件，内容如下：

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

1.  创建一个名为`post.go`的文件，内容如下：

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
                }  else if err != nil {
                    w.WriteHeader(http.StatusInternalServerError)
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

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             $ chapter8/handlers"
        )

        func main() {
            http.HandleFunc("/name", handlers.HelloHandler)
            http.HandleFunc("/greeting", handlers.GreetingHandler)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
Listening on port :3333
```

1.  在一个单独的终端中，运行以下命令：

```go
$ curl "http://localhost:3333/name?name=Reader" -X GET $ curl "http://localhost:3333/greeting" -X POST -d  
 'name=Reader;greeting=Goodbye'
```

您应该看到以下输出：

```go
$ curl "http://localhost:3333/name?name=Reader" -X GET 
Hello Reader!

$ curl "http://localhost:3333/greeting" -X POST -d 'name=Reader;greeting=Goodbye' 
{"payload":{"greeting":"Goodbye","name":"Reader"},"successful":true}
```

1.  `go.mod`文件可能会被更新，顶级食谱目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

对于这个示例，我们设置了两个处理程序。第一个处理程序期望使用名为`name`的`GET`参数的`GET`请求。当我们使用`curl`时，它返回纯文本字符串`Hello <name>!`。

第二个处理程序期望使用`PostForm`请求的`POST`方法。这是如果您使用标准 HTML 表单而没有任何 AJAX 调用时会得到的结果。或者，我们可以从请求体中解析 JSON。这通常使用`json.Decoder`来完成。我建议您也尝试这个练习。最后，处理程序发送一个 JSON 格式的响应并设置所有适当的标头。

尽管所有这些都是明确写出的，但有许多方法可以使代码更简洁，包括以下方法：

+   使用[`github.com/unrolled/render`](https://github.com/unrolled/render)来处理响应

+   使用本章中提到的各种 Web 框架来解析路由参数，限制路由到特定的 HTTP 动词，处理优雅的关闭等

# 使用结构和闭包进行有状态的处理程序

由于 HTTP 处理程序函数的签名稀疏，向处理程序添加状态可能会显得棘手。例如，有多种方法可以包含数据库连接。实现这一点的两种方法是通过闭包传递状态，这对于在单个处理程序上实现灵活性非常有用，或者使用结构。

这个示例将演示两者。我们将使用一个`struct`控制器来存储一个存储接口，并创建两个由外部函数修改的单个处理程序的路由。

# 如何做...

以下步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter8/controllers`的新目录，并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/controllers 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/controllers    
```

1.  从`~/projects/go-programming-cookbook-original/chapter8/controllers`复制测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`controller.go`的文件，内容如下：

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

1.  创建一个名为`storage.go`的文件，内容如下：

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

1.  创建一个名为`post.go`的文件，内容如下：

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
            } else if err != nil {
                w.WriteHeader(http.StatusInternalServerError)
            }

        }
```

1.  创建一个名为`get.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录并进入该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/controllers"
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

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
Listening on port :3333
```

1.  在另一个终端中，运行以下命令：

```go
$ curl "http://localhost:3333/set" -X POST -d "value=value" 
$ curl "http://localhost:3333/get" -X GET 
$ curl "http://localhost:3333/get/default" -X GET
```

您应该看到以下输出：

```go
$ curl "http://localhost:3333/set" -X POST -d "value=value"
{"value":"value"}

$ curl "http://localhost:3333/get" -X GET 
{"value":"value"}

$ curl "http://localhost:3333/get/default" -X GET 
{"value":"default"}
```

1.  `go.mod`文件可能会被更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这些策略有效是因为 Go 允许方法满足诸如`http.HandlerFunc`之类的类型化函数。通过使用结构，我们可以在`main.go`中注入各种部分，其中包括数据库连接，日志记录等。在这个示例中，我们插入了一个`Storage`接口。连接到控制器的所有处理程序都可以使用它的方法和属性。

`GetValue`方法没有`http.HandlerFunc`签名，而是返回一个。这就是我们可以使用闭包来注入状态的方式。在`main.go`中，我们定义了两个路由，一个将`UseDefault`设置为`false`，另一个将其设置为`true`。这可以在定义跨多个路由的函数时使用，或者在使用结构时，您的处理程序感觉太繁琐时使用。

# 验证 Go 结构和用户输入的输入

Web 验证可能会有问题。这个示例将探讨使用闭包来支持验证函数的易于模拟，并在初始化控制器结构时允许对验证类型的灵活性，正如前面的示例所描述的那样。

我们将对一个结构执行此验证，但不探讨如何填充这个结构。我们可以假设数据是通过解析 JSON 有效载荷、明确从表单输入中填充或其他方法填充的。

# 如何做...

以下步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter8/validation`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/validation 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/validation    
```

1.  从`~/projects/go-programming-cookbook-original/chapter8/validation`复制测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`controller.go`的文件，内容如下：

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

1.  创建一个名为`validate.go`的文件，内容如下：

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

1.  创建一个名为`process.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/validation"
        )

        func main() {
            c := validation.New()
            http.HandleFunc("/", c.Process)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }
```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

你应该看到以下输出：

```go
$ go run main.go
Listening on port :3333
```

1.  在另一个终端中，运行以下命令：

```go
$ curl "http://localhost:3333/" -X POST -d '{}' $ curl "http://localhost:3333/" -X POST -d '{"name":"test"}' $ curl "http://localhost:3333/" -X POST -d '{"name":"test",
  "age": 5}' -v
```

你应该看到以下输出：

```go
$ curl "http://localhost:3333/" -X POST -d '{}'
name is required

$ curl "http://localhost:3333/" -X POST -d '{"name":"test"}'
age is required and must be a value greater than 0 and 
less than 120

$ curl "http://localhost:3333/" -X POST -d '{"name":"test",
"age": 5}' -v

<lots of output, should contain a 200 OK status code>
```

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶层食谱目录中。

1.  如果你复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

我们通过向我们的控制器结构传递一个闭包来处理验证。对于控制器可能需要验证的任何输入，我们都需要一个这样的闭包。这种方法的优势在于我们可以在运行时模拟和替换验证函数，因此测试变得更简单。此外，我们不受限于单个函数签名，可以传递诸如数据库连接之类的东西给我们的验证函数。

这个食谱展示的另一件事是返回一个名为`Verror`的类型错误。这种类型保存了可以显示给用户的验证错误消息。这种方法的一个缺点是它不能一次处理多个验证消息。通过修改`Verror`类型以允许更多状态，例如通过包含一个映射，来容纳多个验证错误，然后从我们的`ValidatePayload`函数返回，这是可能的。

# 渲染和内容协商

Web 处理程序可以返回各种内容类型；例如，它们可以返回 JSON、纯文本、图像等。在与 API 通信时，通常可以指定和接受内容类型，以澄清你将以什么格式传递数据，以及你想要接收什么数据。

这个食谱将探讨使用`unrolled/render`和一个自定义函数来协商内容类型并相应地做出响应。

# 如何做...

以下步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter8/negotiate`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/negotiate 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/negotiate    
```

1.  复制来自`~/projects/go-programming-cookbook-original/chapter8/negotiate`的测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`negotiate.go`的文件，内容如下：

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

1.  创建一个名为`respond.go`的文件，内容如下：

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

1.  创建一个名为`handler.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/negotiate"
        )

        func main() {
            http.HandleFunc("/", negotiate.Handler)
            fmt.Println("Listening on port :3333")
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }
```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

你应该看到以下输出：

```go
$ go run main.go
Listening on port :3333
```

1.  在另一个终端中，运行以下命令：

```go
$ curl "http://localhost:3333" -H "Content-Type: text/xml" $ curl "http://localhost:3333" -H "Content-Type: application/json"
```

你应该看到以下输出：

```go
$ curl "http://localhost:3333" -H "Content-Type: text/xml"
<payload><status>Successful!</status></payload> 
$ curl "http://localhost:3333" -H "Content-Type: application/json"
{"status":"Successful!"}
```

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶层食谱目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

`github.com/unrolled/render`包为这个教程做了大量的工作。如果需要处理 HTML 模板等，还有大量其他选项可以输入。这个教程可以用于在通过传递各种内容类型标头来自动协商工作时，或者通过直接操作结构来演示如何解耦中间件逻辑和处理程序。

类似的模式可以应用于接受标头，但要注意这些标头通常包含多个值，您的代码将不得不考虑到这一点。

# 实现和使用中间件

Go 中用于处理程序的中间件是一个被广泛探索的领域。有各种各样的包用于处理中间件。这个教程将从头开始创建中间件，并实现一个`ApplyMiddleware`函数来链接一系列中间件。

它还将探讨在请求上下文对象中设置值并稍后使用中间件检索它们。这将通过一个非常基本的处理程序来完成，以帮助演示如何将中间件逻辑与处理程序解耦。

# 如何做...

以下步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter8/middleware`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/middleware 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/middleware    
```

1.  复制`~/projects/go-programming-cookbook-original/chapter8/middleware`中的测试，或者将其用作编写一些自己代码的练习！

1.  创建一个名为`middleware.go`的文件，其中包含以下内容：

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

1.  创建一个名为`context.go`的文件，其中包含以下内容：

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

1.  创建一个名为`handler.go`的文件，其中包含以下内容：

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

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"
            "log"
            "net/http"
            "os"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/middleware"
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

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
Listening on port :3333
```

1.  在另一个终端中，运行以下`curl`命令多次：

```go
$ curl http://localhost:3333
```

您应该看到以下输出：

```go
$ curl http://localhost:3333
success

$ curl http://localhost:3333
success

$ curl http://localhost:3333
success
```

1.  在原始的`main.go`中，您应该看到以下内容：

```go
Listening on port :3333
started request to / with id 100
completed request to / with id 100 in 52.284µs
started request to / with id 101
completed request to / with id 101 in 40.273µs
started request to / with id 102
```

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶级教程目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

中间件可以用于执行简单的操作，比如日志记录、度量收集和分析。中间件也可以用于在每个请求上动态填充变量。例如，可以收集请求中的 X-header 来设置一个 ID 或生成一个 ID，就像我们在这个教程中所做的那样。另一种 ID 策略可能是为每个请求生成一个**通用唯一标识符**（**UUID**）—这样我们就可以轻松地将日志消息关联在一起，并跟踪您的请求穿越不同的应用程序，如果多个微服务参与构建响应的话。

在处理上下文值时，考虑中间件的顺序是很重要的。通常，最好不要让中间件相互依赖。例如，在这个教程中，最好在日志中间件本身生成 UUID。然而，这个教程应该作为分层中间件和在`main.go`中初始化它们的指南。

# 构建反向代理应用程序

在这个教程中，我们将开发一个反向代理应用程序。这个想法是，通过在浏览器中访问`http://localhost:3333`，所有流量将被转发到一个可配置的主机，并且响应将被转发到您的浏览器。最终结果应该是通过我们的代理应用程序在浏览器中呈现[`www.golang.org`](https://www.golang.org)。

这可以与端口转发和 SSH 隧道结合使用，以便通过中间服务器安全地访问网站。这个配方将从头开始构建一个反向代理，但这个功能也由`net/http/httputil`包提供。使用这个包，传入的请求可以通过`Director func(*http.Request)`进行修改，传出的响应可以通过`ModifyResponse func(*http.Response) error`进行修改。此外，还支持对响应进行缓冲。

# 如何做...

以下步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter8/proxy`的新目录，并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/proxy 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/proxy    
```

1.  从`~/projects/go-programming-cookbook-original/chapter8/proxy`复制测试，或者将其作为练习编写一些自己的代码！

1.  创建一个名为`proxy.go`的文件，内容如下：

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

        // ServeHTTP means that proxy implements the Handler interface
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

1.  创建一个名为`process.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录并进入。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/proxy"
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

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
Listening on port :3333
```

1.  将浏览器导航到`localhost:3333/`。您应该看到[`golang.org/`](https://golang.org/)网站呈现出来！

1.  `go.mod`文件可能已更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

Go 请求和响应对象在客户端和处理程序之间大部分是可共享的。这段代码使用了一个满足`Handler`接口的`Proxy`结构获取的请求。`main.go`文件使用了`Handle`而不是其他地方使用的`HandleFunc`。一旦请求可用，它就被修改为在请求中添加`Proxy.BaseURL`，然后客户端进行分发。最后，响应被复制回`ResponseWriter`接口。这包括所有标头、正文和状态。

如果需要，我们还可以添加一些额外的功能，比如基本的`auth`请求，令牌管理等。这对于代理管理 JavaScript 或其他客户端应用程序的会话非常有用。

# 将 GRPC 导出为 JSON API

在第七章*Web Clients and APIs*的*理解 GRPC 客户端*配方中，我们编写了一个基本的 GRPC 服务器和客户端。这个配方将扩展这个想法，将常见的 RPC 函数放在一个包中，并将它们包装在一个 GRPC 服务器和一个标准的 Web 处理程序中。当您的 API 希望支持两种类型的客户端，但又不想为常见功能复制代码时，这将非常有用。

# 准备工作

根据以下步骤配置您的环境：

1.  参考本章开头的*技术要求*部分中给出的步骤。

1.  安装 GRPC ([`grpc.io/docs/quickstart/go/`](https://grpc.io/docs/quickstart/go/))并运行以下命令：

+   `go get -u github.com/golang/protobuf/{proto,protoc-gen-go}`

+   `go get -u google.golang.org/grpc`

# 如何做...

以下步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter8/grpcjson`的新目录，并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/grpcjson 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter8/grpcjson    
```

1.  从`~/projects/go-programming-cookbook-original/chapter8/grpcjson`复制测试，或者将其作为练习编写一些自己的代码！

1.  创建一个名为`keyvalue`的新目录并进入。

1.  创建一个名为`keyvalue.proto`的文件，内容如下：

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
$ protoc --go_out=plugins=grpc:. keyvalue.proto
```

1.  返回上一级目录。

1.  创建一个名为`internal`的新目录。

1.  创建一个名为`internal/keyvalue.go`的文件，内容如下：

```go
        package internal

        import (
            "golang.org/x/net/context"
            "sync"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/grpcjson/keyvalue"
            "google.golang.org/grpc"
            "google.golang.org/grpc/codes"
        )

        // KeyValue is a struct that holds a map
        type KeyValue struct {
            mutex sync.RWMutex
            m map[string]string
        }

        // NewKeyValue initializes the KeyValue struct and its map
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

1.  创建一个名为`grpc`的新目录。

1.  创建一个名为`grpc/main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/grpcjson/internal"
            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/grpcjson/keyvalue"
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

1.  创建一个名为`http`的新目录。

1.  创建一个名为`http/set.go`的文件，内容如下：

```go
        package main

        import (
            "encoding/json"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/grpcjson/internal"
            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/grpcjson/keyvalue"
            "github.com/apex/log"
        )

        // Controller holds an internal KeyValueObject
        type Controller struct {
            *internal.KeyValue
        }

        // SetHandler wraps our GRPC Set
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

1.  创建一个名为`http/get.go`的文件，内容如下：

```go
        package main

        import (
            "encoding/json"
            "net/http"

            "google.golang.org/grpc"
            "google.golang.org/grpc/codes"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/grpcjson/keyvalue"
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

1.  创建一个名为`http/main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter8/grpcjson/internal"
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

1.  运行`go run ./http`命令。您应该会看到以下输出：

```go
$ go run ./http
Listening on port :3333
```

1.  在单独的终端中，运行以下命令：

```go
$ curl "http://localhost:3333/set" -d '{"key":"test", 
 "value":"123"}' -v $ curl "http://localhost:3333/get?key=badtest" -v $ curl "http://localhost:3333/get?key=test" -v
```

您应该会看到以下输出：

```go
$ curl "http://localhost:3333/set" -d '{"key":"test", 
"value":"123"}' -v
{"value":"123"}

$ curl "http://localhost:3333/get?key=badtest" -v 
<should return a 404>

$ curl "http://localhost:3333/get?key=test" -v 
{"value":"123"}
```

1.  `go.mod`文件可能已更新，`go.sum`文件现在应该存在于顶层配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

尽管这个配方省略了客户端，但您可以复制第七章中*理解 GRPC 客户端*配方中的步骤，并且您应该会看到与我们的 curls 看到的相同的结果。`http`和`grpc`目录都使用相同的内部包。在这个包中，我们必须小心返回适当的 GRPC 错误代码，并将这些错误代码正确映射到我们的 HTTP 响应中。在这种情况下，我们使用`codes.NotFound`，将其映射到`http.StatusNotFound`。如果您需要处理多个错误，使用`switch`语句可能比`if...else`语句更合适。

您可能还注意到的另一件事是，GRPC 签名通常非常一致。它们接受一个请求并返回一个可选的响应和一个错误。如果您的 GRPC 调用足够重复，并且似乎很适合代码生成，那么可能可以创建一个通用的处理程序`shim`；最终您可能会看到类似`goadesign/goa`这样的包。
