# 第九章：Web 应用程序

Go 在标准库中有一个强大的 HTTP 包。`net/http`包的文档位于[`golang.org/pkg/net/http/`](https://golang.org/pkg/net/http/)，包含了 HTTP 和 HTTPS 的实用工具。起初，我建议你远离社区的 HTTP 框架，坚持使用 Go 标准库。标准的 HTTP 包包括了用于监听、路由和模板的函数。内置的 HTTP 服务器具有生产质量，并直接绑定到端口，消除了需要单独的 httpd，如 Apache、IIS 或 nginx。然而，通常会看到 nginx 监听公共端口`80`，并将所有请求反向代理到监听本地端口而不是`80`的 Go 服务器。

在本章中，我们涵盖了运行 HTTP 服务器的基础知识，使用 HTTPS，设置安全的 cookies，以及转义输出。我们还介绍了如何使用 Negroni 中间件包，并实现用于记录、添加安全的 HTTP 头和提供静态文件的自定义中间件。Negroni 采用了 Go 的成熟方法，并鼓励使用标准库`net/http`处理程序。它非常轻量级，并建立在现有的 Go 结构之上。此外，还提到了与运行 Web 应用程序相关的其他最佳实践。

还提供了 HTTP 客户端的示例。从进行基本的 HTTP 请求开始，我们继续进行 HTTPS 请求，并使用客户端证书进行身份验证和代理路由流量。

在本章中，我们将涵盖以下主题：

+   HTTP 服务器

+   简单的 HTTP 服务器

+   TLS 加密的 HTTP（HTTPS）

+   使用安全的 cookies

+   HTML 转义输出

+   Negroni 中间件

+   记录请求

+   添加安全的 HTTP 头

+   提供静态文件

+   其他最佳实践

+   跨站请求伪造（CSRF）令牌

+   防止用户枚举和滥用

+   避免本地和远程文件包含漏洞

+   HTTP 客户端

+   进行基本的 HTTP 请求

+   使用客户端 SSL 证书

+   使用代理

+   使用系统代理

+   使用 HTTP 代理

+   使用 SOCKS5 代理（Tor）

# HTTP 服务器

HTTP 是建立在 TCP 层之上的应用程序协议。概念相对简单；你可以使用纯文本来构造一个请求。在第一行，你将提供方法，比如`GET`或`POST`，以及路径和你遵循的 HTTP 版本。之后，你将提供一系列键值对来描述你的请求。通常，你需要提供一个`Host`值，以便服务器知道你正在请求哪个网站。一个简单的 HTTP 请求可能是这样的：

```go
GET /archive HTTP/1.1
Host: www.devdungeon.com  
```

不过，你不需要担心 HTTP 规范中的所有细节。Go 提供了一个`net/http`包，其中包含了几个工具，可以轻松地创建生产就绪的 Web 服务器，包括对 HTTP/2.0 的支持，Go 1.6 及更新版本。本节涵盖了与运行和保护 HTTP 服务器相关的主题。

# 简单的 HTTP 服务器

在这个例子中，一个 HTTP 服务器演示了使用标准库创建一个监听服务器是多么简单。目前还没有路由或多路复用。在这种情况下，通过服务器提供了一个特定的目录。`http.FileServer()`内置了目录列表，所以如果你对`/`发出 HTTP 请求，它将列出目录中可用的文件：

```go
package main

import (
   "fmt"
   "log"
   "net/http"
   "os"
)

func printUsage() {
   fmt.Println(os.Args[0] + ` - Serve a directory via HTTP

URL should include protocol IP or hostname and port separated by colon.

Usage:
  ` + os.Args[0] + ` <listenUrl> <directory>

Example:
  ` + os.Args[0] + ` localhost:8080 .
  ` + os.Args[0] + ` 0.0.0.0:9999 /home/nanodano
`)
}

func checkArgs() (string, string) {
   if len(os.Args) != 3 {
      printUsage()
      os.Exit(1)
   }
   return os.Args[1], os.Args[2]
}

func main() {
   listenUrl, directoryPath := checkArgs()
   err := http.ListenAndServe(listenUrl,      
     http.FileServer(http.Dir(directoryPath)))
   if err != nil {
      log.Fatal("Error running server. ", err)
   }
}
```

下一个示例显示了如何路由路径并创建一个处理传入请求的函数。这个示例不接受任何命令行参数，因为它本身并不是一个很有用的程序，但你可以将其用作基本模板：

```go
package main

import (
   "fmt"
   "net/http"
   "log"
)

func indexHandler(writer http.ResponseWriter, request *http.Request) {
   // Write the contents of the response body to the writer interface
   // Request object contains information about and from the client
   fmt.Fprintf(writer, "You requested: " + request.URL.Path)
}

func main() {
   http.HandleFunc("/", indexHandler)
   err := http.ListenAndServe("localhost:8080", nil)
   if err != nil {
      log.Fatal("Error creating server. ", err)
   }
}
```

# HTTP 基本认证

HTTP 基本认证通过取用户名和密码，用冒号分隔符组合它们，并使用 base64 进行编码来实现。用户名和密码通常可以作为 URL 的一部分传递，例如：`http://<username>:<password>@www.example.com`。在底层，实际发生的是用户名和密码被组合、编码，并作为 HTTP 头传递。

如果您使用这种身份验证方法，请记住它是不加密的。在传输过程中，用户名和密码没有任何保护。您始终希望在传输层上使用加密，这意味着添加 TLS/SSL。

如今，HTTP 基本身份验证并不常用，但它很容易实现。更常见的方法是在应用程序中构建或使用自己的身份验证层，例如将用户名和密码与一个充满了盐和哈希密码的用户数据库进行比较。

有关创建需要 HTTP 基本身份验证的 HTTP 服务器的客户端示例，请参阅第八章 *暴力破解*。Go 标准库仅提供了 HTTP 基本身份验证的客户端方法。它不提供服务器端检查基本身份验证的方法。

我不建议您在服务器上实现 HTTP 基本身份验证。如果需要对客户端进行身份验证，请使用 TLS 证书。

# 使用 HTTPS

在第六章 *密码学*中，我们向您介绍了生成密钥并创建自签名证书所需的步骤。我们还为您提供了如何运行 TCP 套接字级别的 TLS 服务器的示例。本节将演示如何创建一个 TLS 加密的 HTTP 服务器或 HTTPS 服务器。

TLS 是 SSL 的更新版本，Go 有一个很好地支持它的标准包。您需要一个使用该密钥生成的私钥和签名证书。您可以使用自签名证书或由公认的证书颁发机构签名的证书。从历史上看，由受信任的机构签名的 SSL 证书总是需要花钱的，但[`letsencrypt.org/`](https://letsencrypt.org/)改变了这一局面，他们开始提供由广泛信任的机构签名的免费和自动化证书。

如果您需要一个证书（`cert.pem`）的示例，请参考第六章 *密码学*中的创建自签名证书的示例。

以下代码演示了如何运行一个提供单个网页的 HTTPS 服务器的最基本示例。有关各种 HTTP 蜜罐示例和更多 HTTP 服务器参考代码，请参考第十章 *网络爬虫*中的示例。在源代码中初始化 HTTPS 服务器后，您可以像处理 HTTP 服务器对象一样处理它。请注意，这与 HTTP 服务器之间的唯一区别是您需要调用`http.ListenAndServeTLS()`而不是`http.ListenAndServe()`。此外，您必须为服务器提供证书和密钥：

```go
package main

import (
   "fmt"
   "net/http"
   "log"
)

func indexHandler(writer http.ResponseWriter, request *http.Request) {
   fmt.Fprintf(writer, "You requested: "+request.URL.Path)
}

func main() {
   http.HandleFunc("/", indexHandler)
   err := http.ListenAndServeTLS( 
      "localhost:8181", 
      "cert.pem", 
      "privateKey.pem", 
      nil, 
   )
   if err != nil {
      log.Fatal("Error creating server. ", err)
   }
}
```

# 创建安全 cookie

Cookies 本身不应包含用户无法查看的敏感信息。攻击者可以针对 cookie 进行攻击，试图收集私人信息。最常见的目标是会话 cookie。如果会话 cookie 受到损害，攻击者可以使用该 cookie 冒充用户，服务器将允许这种行为。

`HttpOnly`标志要求浏览器阻止 JavaScript 访问 cookie，以防止跨站脚本攻击。只有在进行 HTTP 请求时才会发送 cookie。如果确实需要通过 JavaScript 访问 cookie，只需创建一个与会话 cookie 不同的 cookie。

`Secure`标志要求浏览器仅在 TLS/SSL 加密下传输 cookie。这可以防止通过嗅探公共未加密的 Wi-Fi 网络或中间人连接进行的会话劫持尝试。一些网站只会在登录页面上使用 SSL 来保护您的密码，但之后的每次连接都是通过普通 HTTP 进行的，会话 cookie 可以在传输过程中被窃取，或者在缺少`HttpOnly`标志的情况下，可能会被 JavaScript 窃取。

创建会话令牌时，请确保使用加密安全的伪随机数生成器生成它。会话令牌的长度应至少为 128 位。请参阅第六章，*密码学*，了解生成安全随机字节的示例。

以下示例创建了一个简单的 HTTP 服务器，只有一个函数`indexHandler()`。该函数使用推荐的安全设置创建一个 cookie，然后在打印响应正文并返回之前调用`http.SetCookie()`：

```go
package main

import (
   "fmt"
   "net/http"
   "log"
   "time"
)

func indexHandler(writer http.ResponseWriter, request *http.Request) {
   secureSessionCookie := http.Cookie {
      Name: "SessionID",
      Value: "<secure32ByteToken>",
      Domain: "yourdomain.com",
      Path: "/",
      Expires: time.Now().Add(60 * time.Minute),
      HttpOnly: true, // Prevents JavaScript from accessing
      Secure: true, // Requires HTTPS
   }   
   // Write cookie header to response
   http.SetCookie(writer, &secureSessionCookie)   
   fmt.Fprintln(writer, "Cookie has been set.")
}

func main() {
   http.HandleFunc("/", indexHandler)
   err := http.ListenAndServe("localhost:8080", nil)
   if err != nil {
      log.Fatal("Error creating server. ", err)
   }
}
```

# HTML 转义输出

Go 语言有一个标准函数用于转义字符串，防止 HTML 字符被渲染。

在输出用户接收到的任何数据到响应输出时，始终对其进行转义，以防止跨站脚本攻击。无论用户提供的数据来自 URL 查询、POST 值、用户代理标头、表单、cookie 还是数据库，都适用这一规则。以下代码片段给出了转义字符串的示例：

```go
package main

import (
   "fmt"
   "html"
)

func main() {
   rawString := `<script>alert("Test");</script>`
   safeString := html.EscapeString(rawString)

   fmt.Println("Unescaped: " + rawString)
   fmt.Println("Escaped: " + safeString)
}
```

# Negroni 中间件

中间件是指可以绑定到请求/响应流程并在传递给下一个中间件并最终返回给客户端之前采取行动或进行修改的函数。

中间件是按顺序在每个请求上运行的一系列函数。您可以向此链中添加更多函数。我们将看一些实际的例子，比如列入黑名单的 IP 地址、添加日志记录和添加授权检查。

中间件的顺序很重要。例如，我们可能希望先放日志记录中间件，然后是 IP 黑名单中间件。我们希望 IP 黑名单模块首先运行，或者至少在开始附近运行，这样其他中间件不会浪费资源处理一个将被拒绝的请求。您可以在将请求和响应传递给下一个中间件处理程序之前操纵它们。

您可能还想构建自定义中间件来进行分析、日志记录、列入黑名单的 IP 地址、注入标头，或拒绝某些用户代理，比如`curl`、`python`或`go`。

这些示例使用了 Negroni 包。在编译和运行这些示例之前，您需要`go get`该包。这些示例调用了`http.ListenAndServe()`，但您也可以很容易地修改它们以使用`http.ListenAndServeTLS()`来使用 TLS：

```go
go get github.com/urfave/negroni 
```

以下示例创建了一个`customMiddlewareHandler()`函数，我们将告诉`negroniHandler`接口使用它。自定义中间件只是简单地记录传入的请求 URL 和用户代理，但您可以做任何您喜欢的事情，包括修改请求再返回给客户端：

```go
package main

import (
   "fmt"
   "log"
   "net/http"

   "github.com/urfave/negroni"
)

// Custom middleware handler logs user agent
func customMiddlewareHandler(rw http.ResponseWriter, 
   r *http.Request, 
   next http.HandlerFunc, 
) {
   log.Println("Incoming request: " + r.URL.Path)
   log.Println("User agent: " + r.UserAgent())

   next(rw, r) // Pass on to next middleware handler
}

// Return response to client
func indexHandler(writer http.ResponseWriter, request *http.Request) {
   fmt.Fprintf(writer, "You requested: " + request.URL.Path)
}

func main() {
   multiplexer := http.NewServeMux()
   multiplexer.HandleFunc("/", indexHandler)

   negroniHandler := negroni.New()
   negroniHandler.Use(negroni.HandlerFunc(customMiddlewareHandler))
   negroniHandler.UseHandler(multiplexer)

   http.ListenAndServe("localhost:3000", negroniHandler)
}
```

# 记录请求

由于日志记录是如此常见的任务，Negroni 附带了一个日志记录中间件，您可以使用，如下例所示：

```go
package main

import (
   "fmt"
   "net/http"

   "github.com/urfave/negroni"
)

// Return response to client
func indexHandler(writer http.ResponseWriter, request *http.Request) {
   fmt.Fprintf(writer, "You requested: " + request.URL.Path)
}

func main() {
   multiplexer := http.NewServeMux()
   multiplexer.HandleFunc("/", indexHandler)

   negroniHandler := negroni.New()
   negroniHandler.Use(negroni.NewLogger()) // Negroni's default logger
   negroniHandler.UseHandler(multiplexer)

   http.ListenAndServe("localhost:3000", negroniHandler)
}
```

# 添加安全的 HTTP 标头

利用 Negroni 包，我们可以轻松地创建自己的中间件来注入一组 HTTP 标头，以帮助提高安全性。您需要评估每个标头，看看它是否适合您的应用程序。此外，并非每个浏览器都支持这些标头中的每一个。这是一个很好的基线，可以根据需要进行修改。

此示例中使用了以下标头：

| **标头** | **描述** |
| --- | --- |
| `Content-Security-Policy` | 这定义了哪些脚本或远程主机是受信任的，并能够提供可执行的 JavaScript |
| `X-Frame-Options` | 这定义了是否可以使用框架和 iframe，以及允许出现在框架中的域 |
| `X-XSS-Protection` | 这告诉浏览器在检测到跨站脚本攻击时停止加载；如果定义了良好的`Content-Security-Policy`标头，则基本上是不必要的 |
| `Strict-Transport-Security` | 这告诉浏览器只使用 HTTPS，而不是 HTTP |
| `X-Content-Type-Options` | 这告诉浏览器使用服务器提供的 MIME 类型，而不是基于 MIME 嗅探的猜测进行修改 |

客户端的网络浏览器最终决定是否使用或忽略这些标头。如果浏览器不知道如何应用标头值，它们就无法保证任何安全性。

这个例子创建了一个名为`addSecureHeaders()`的函数，它被用作额外的中间件处理程序，以在返回给客户端之前修改响应。根据你的应用程序需要调整标头：

```go
package main

import (
   "fmt"
   "net/http"

   "github.com/urfave/negroni"
)

// Custom middleware handler logs user agent
func addSecureHeaders(rw http.ResponseWriter, r *http.Request, 
   next http.HandlerFunc) {
   rw.Header().Add("Content-Security-Policy", "default-src 'self'")
   rw.Header().Add("X-Frame-Options", "SAMEORIGIN")
   rw.Header().Add("X-XSS-Protection", "1; mode=block")
   rw.Header().Add("Strict-Transport-Security", 
      "max-age=10000, includeSubdomains; preload")
   rw.Header().Add("X-Content-Type-Options", "nosniff")

   next(rw, r) // Pass on to next middleware handler
}

// Return response to client
func indexHandler(writer http.ResponseWriter, request *http.Request) {
   fmt.Fprintf(writer, "You requested: " + request.URL.Path)
}

func main() {
   multiplexer := http.NewServeMux()
   multiplexer.HandleFunc("/", indexHandler)

   negroniHandler := negroni.New()

   // Set up as many middleware functions as you need, in order
   negroniHandler.Use(negroni.HandlerFunc(addSecureHeaders))
   negroniHandler.Use(negroni.NewLogger())
   negroniHandler.UseHandler(multiplexer)

   http.ListenAndServe("localhost:3000", negroniHandler)
}
```

# 提供静态文件

另一个常见的 Web 服务器任务是提供静态文件。值得一提的是 Negroni 中间件处理程序用于提供静态文件。只需添加一个额外的`Use()`调用，并将`negroni.NewStatic()`传递给它。确保你的静态文件目录只包含客户端应该访问的文件。在大多数情况下，静态文件目录包含客户端的 CSS 和 JavaScript 文件。不要放置数据库备份、配置文件、SSH 密钥、Git 存储库、开发文件或任何客户端不应该访问的内容。像这样添加静态文件中间件：

```go
negroniHandler.Use(negroni.NewStatic(http.Dir("/path/to/static/files")))  
```

# 其他最佳实践

在创建 Web 应用程序时，还有一些其他值得考虑的事项。虽然它们不是 Go 特有的，但在开发时考虑这些最佳实践是值得的。

# CSRF 令牌

**跨站请求伪造**，或**CSRF**，令牌是一种试图阻止一个网站代表你对另一个网站采取行动的方式。

CSRF 是一种常见的攻击方式，受害者会访问一个嵌入了恶意代码的网站，试图向不同的网站发出请求。例如，一个恶意的行为者嵌入了 JavaScript，试图向每个银行网站发出 POST 请求，尝试将 1000 美元转账到攻击者的银行账户。如果受害者在其中一个银行有活动会话，并且该银行没有实施 CSRF 令牌，那么银行的网站可能会接受并处理该请求。

即使在受信任的网站上，也有可能成为 CSRF 攻击的受害者，如果受信任的网站容易受到反射或存储型跨站脚本攻击。自 2007 年以来，CSRF 一直是*OWASP 十大*中的一部分，并且在 2017 年仍然如此。

Go 提供了一个`xsrftoken`包，你可以在[`godoc.org/golang.org/x/net/xsrftoken`](https://godoc.org/golang.org/x/net/xsrftoken)上了解更多信息。它提供了一个`Generate()`函数来创建令牌，以及一个`Valid()`函数来验证令牌。你可以使用他们的实现，也可以选择开发适合自己需求的实现。

要实现 CSRF 令牌，创建一个 16 字节的随机令牌，并将其存储在与用户会话关联的服务器上。你可以使用任何你喜欢的后端来存储令牌，无论是在内存中、数据库中还是在文件中。将 CSRF 令牌嵌入表单作为隐藏字段。在服务器端处理表单时，验证 CSRF 令牌是否存在并与用户匹配。在使用后销毁令牌。不要重复使用相同的令牌。

在前面的章节中已经介绍了实现 CSRF 令牌的各种要求：

+   生成令牌：在第六章中，*密码学*，名为*密码学安全伪随机数生成器（CSPRNG）*的部分提供了生成随机数、字符串和字节的示例。

+   创建、提供和处理 HTML 表单：在第九章中，*Web 应用程序*，名为*HTTP 服务器*的部分提供了创建安全 Web 服务器的信息，而第十二章，*社会工程*，有一个名为*HTTP POST 表单登录蜜罐*的部分，其中有一个处理 POST 请求的示例。

+   将令牌存储在文件中：在第三章中，*文件操作*，名为*将字节写入文件*的部分提供了将数据存储在文件中的示例。

+   在数据库中存储令牌：在第八章中，*暴力破解*，标题为*暴力破解数据库登录*的部分提供了连接到各种数据库类型的蓝图。

# 防止用户枚举和滥用

这里需要记住的重要事项如下：

+   不要让人们弄清楚谁有帐户

+   不要让某人通过您的电子邮件服务器向用户发送垃圾邮件

+   不要让人们通过暴力尝试弄清楚谁已注册

让我们详细说明一下实际例子。

# 注册

当有人尝试注册电子邮件地址时，不要向 Web 客户端用户提供有关帐户是否已注册的任何反馈。相反，向该地址发送一封电子邮件，并简单地向 Web 用户显示一条消息，内容是“已向提供的地址发送了一封电子邮件”。

如果他们从未注册过，一切都是正常的。如果他们已经注册，网页用户不会收到电子邮件已注册的通知。相反，将向用户的地址发送一封电子邮件，通知他们该电子邮件已经注册。这将提醒他们已经有一个帐户，他们可以使用密码重置工具，或者让他们知道有可疑的情况，可能有人在做一些恶意的事情。

要小心，不要让攻击者反复尝试登录过程并向真实用户的电子邮件发送大量邮件。

# 登录

不要向网页用户提供关于电子邮件是否存在的反馈。您不希望某人能够尝试使用电子邮件地址登录并通过返回的错误消息了解该地址是否有帐户。例如，攻击者可以尝试使用一系列电子邮件地址登录，如果 Web 服务器对某些电子邮件返回“密码不匹配”，对其他电子邮件返回“该电子邮件未注册”，他们可以确定哪些电子邮件已在您的服务中注册。

# 重置密码

避免允许电子邮件垃圾邮件。限制发送的电子邮件数量，以便攻击者无法通过多次提交忘记密码表单来向用户发送垃圾邮件。

创建重置令牌时，请确保它具有良好的熵，以便无法猜测。不要仅基于时间和用户 ID 创建令牌，因为这样太容易被猜测和暴力破解，熵不足。对于令牌，您应该使用至少 16-32 个随机字节以获得足够的熵。参考第六章，*密码学*，了解生成密码学安全随机字节的示例。

此外，将令牌设置为在短时间后过期。从一小时到一天不等的时间段都是不错的选择，这取决于您的应用程序。一次只允许一个重置令牌，并在使用后销毁令牌，以防止重放和再次使用。

# 用户配置文件

与登录页面类似，如果您有用户配置文件页面，请小心允许用户名枚举。例如，如果有人访问`/users/JohnDoe`，然后访问`/users/JaneDoe`，一个返回`404 Not Found`错误，另一个返回`401 Access Denied`错误，攻击者可以推断一个帐户实际上存在，而另一个不存在。

# 防止 LFI 和 RFI 滥用

**本地文件包含**（**LFI**）和**远程文件包含**（**RFI**）是*OWASP 十大*漏洞之一。它们指的是从本地文件系统或远程主机加载未经意的文件的危险，或者加载预期的文件但带有污染数据。远程文件包含是危险的，因为如果不采取预防措施，用户可能会从恶意服务器提供远程文件。

如果用户未经任何消毒就指定了文件名，则不要从本地文件系统打开文件。考虑一个示例，Web 服务器在请求时返回一个文件。用户可能能够使用这样的 URL 请求包含敏感系统信息的文件，例如`/etc/passwd`。

```go
http://localhost/displayFile?filename=/etc/passwd  
```

如果 Web 服务器处理方式如下（伪代码）：

```go
file = os.Open(request.GET['filename'])
return file.ReadAll()
```

您不能简单地通过在特定目录前面添加来修复它，就像这样：

```go
os.Open('/path/to/mydir/' + GET['filename']).
```

这还不够，因为攻击者可以使用目录遍历返回到文件系统的根目录，就像这样：

```go
http://localhost/displayFile?filename=../../../etc/passwd   
```

务必检查任何文件包含中的目录遍历攻击。

# 受污染的文件

如果攻击者发现了 LFI，或者您提供了一个用于查看日志文件的 Web 界面，您需要确保即使日志被污染，也不会执行任何代码。

攻击者可能会通过对服务采取某些操作来污染您的日志并插入恶意代码。任何生成的日志都必须被视为已加载或显示的服务。

例如，Web 服务器日志可能会通过向实际上是代码的 URL 发出 HTTP 请求而被污染。您的日志将显示`404 Not Found`错误并记录所请求的 URL，实际上是代码。如果它是 PHP 服务器或另一种脚本语言，这将打开潜在的代码执行，但是，对于 Go 来说，最坏的情况将是 JavaScript 注入，这对用户仍然可能是危险的。想象一种情况，一个 Web 应用程序有一个 HTTP 日志查看器，它从磁盘加载日志文件。如果攻击者向`yourwebsite.com/<script>alert("test");</script>`发出请求，那么您的 HTML 日志查看器可能实际上会渲染该代码，如果没有适当地转义或清理。

# HTTP 客户端

如今，发出 HTTP 请求是许多应用程序的核心部分。作为一个友好的网络语言，Go 包含了`net/http`包中用于发出 HTTP 请求的几个工具。

# 基本的 HTTP 请求

这个例子使用了`net/http`标准库包中的`http.Get()`函数。它将把整个响应主体读取到一个名为`body`的变量中，然后将其打印到标准输出：

```go
package main

import (
   "fmt"
   "io/ioutil"
   "log"
   "net/http"
)

func main() {
   // Make basic HTTP GET request
   response, err := http.Get("http://www.example.com")
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   // Read body from response
   body, err := ioutil.ReadAll(response.Body)
   response.Body.Close()
   if err != nil {
      log.Fatal("Error reading response. ", err)
   }

   fmt.Printf("%s\n", body)
}
```

# 使用客户端 SSL 证书

如果远程 HTTPS 服务器具有严格的身份验证并需要受信任的客户端证书，您可以通过在`http.Transport`对象中设置`TLSClientConfig`变量来指定证书文件，该对象由`http.Client`用于发出 GET 请求。

这个例子发出了一个类似于上一个例子的 HTTP GET 请求，但它没有使用`net/http`包提供的默认 HTTP 客户端。它创建了一个自定义的`http.Client`并配置它以使用客户端证书的 TLS。如果您需要证书或私钥，请参考第六章，“密码学”，以获取生成密钥和自签名证书的示例：

```go
package main

import (
   "crypto/tls"
   "log"
   "net/http"
)

func main() {
   // Load cert
   cert, err := tls.LoadX509KeyPair("cert.pem", "privKey.pem")
   if err != nil {
      log.Fatal(err)
   }

   // Configure TLS client
   tlsConfig := &tls.Config{
      Certificates: []tls.Certificate{cert},
   }
   tlsConfig.BuildNameToCertificate()
   transport := &http.Transport{ 
      TLSClientConfig: tlsConfig, 
   }
   client := &http.Client{Transport: transport}

   // Use client to make request.
   // Ignoring response, just verifying connection accepted.
   _, err = client.Get("https://example.com")
   if err != nil {
      log.Println("Error making request. ", err)
   }
}
```

# 使用代理

正向代理可以用于许多用途，包括查看 HTTP 流量、调试应用程序、逆向工程 API、操纵标头，还可以潜在地用于增加您对目标服务器的匿名性。但是，请注意，许多代理服务器仍然使用`X-Forwarded-For`头来转发您的原始 IP。

您可以使用环境变量设置代理，也可以在请求中明确设置代理。Go HTTP 客户端支持 HTTP、HTTPS 和 SOCKS5 代理，比如 Tor。

# 使用系统代理

如果通过环境变量设置了系统的 HTTP(S)代理，Go 的默认 HTTP 客户端将会遵守。Go 使用`HTTP_PROXY`、`HTTPS_PROXY`和`NO_PROXY`环境变量。小写版本也是有效的。您可以在运行进程之前设置环境变量，或者在 Go 中设置环境变量：

```go
os.Setenv("HTTP_PROXY", "proxyIp:proxyPort")  
```

配置环境变量后，使用默认的 Go HTTP 客户端进行的任何 HTTP 请求都将遵守代理设置。在[`golang.org/pkg/net/http/#ProxyFromEnvironment`](https://golang.org/pkg/net/http/#ProxyFromEnvironment)上阅读更多关于默认代理设置的信息。

# 使用特定的 HTTP 代理

要显式设置代理 URL，忽略环境变量，请在由`http.Client`使用的自定义`http.Transport`对象中设置`ProxyURL`变量。以下示例创建了自定义`http.Transport`并指定了`proxyUrlString`。该示例仅具有代理的占位符值，必须替换为有效的代理。然后创建并配置了`http.Client`以使用带有代理的自定义传输：

```go
package main

import (
   "io/ioutil"
   "log"
   "net/http"
   "net/url"
   "time"
)

func main() {
   proxyUrlString := "http://<proxyIp>:<proxyPort>"
   proxyUrl, err := url.Parse(proxyUrlString)
   if err != nil {
      log.Fatal("Error parsing URL. ", err)
   }

   // Set up a custom HTTP transport for client
   customTransport := &http.Transport{ 
      Proxy: http.ProxyURL(proxyUrl), 
   }
   httpClient := &http.Client{ 
      Transport: customTransport, 
      Timeout:   time.Second * 5, 
   }

   // Make request
   response, err := httpClient.Get("http://www.example.com")
   if err != nil {
      log.Fatal("Error making GET request. ", err)
   }
   defer response.Body.Close()

   // Read and print response from server
   body, err := ioutil.ReadAll(response.Body)
   if err != nil {
      log.Fatal("Error reading body of response. ", err)
   }
   log.Println(string(body))
}
```

# 使用 SOCKS5 代理（Tor）

Tor 是一项旨在保护您隐私的匿名服务。除非您充分了解所有影响，否则不要使用 Tor。在[`www.torproject.org`](https://www.torproject.org)上阅读有关 Tor 的更多信息。此示例演示了在进行请求时如何使用 Tor，但这同样适用于其他 SOCKS5 代理。

要使用 SOCKS5 代理，唯一需要修改的是代理的 URL 字符串。不要使用 HTTP 协议，而是使用`socks5://`协议前缀。

默认的 Tor 端口是`9050`，或者在使用 Tor 浏览器捆绑包时是`9150`。以下示例将执行对`check.torproject.org`的 GET 请求，这将让您知道是否正确地通过 Tor 网络进行路由：

```go
package main

import (
   "io/ioutil"
   "log"
   "net/http"
   "net/url"
   "time"
)

// The Tor proxy server must already be running and listening
func main() {
   targetUrl := "https://check.torproject.org"
   torProxy := "socks5://localhost:9050" // 9150 w/ Tor Browser

   // Parse Tor proxy URL string to a URL type
   torProxyUrl, err := url.Parse(torProxy)
   if err != nil {
      log.Fatal("Error parsing Tor proxy URL:", torProxy, ". ", err)
   }

   // Set up a custom HTTP transport for the client   
   torTransport := &http.Transport{Proxy: http.ProxyURL(torProxyUrl)}
   client := &http.Client{
      Transport: torTransport,
      Timeout: time.Second * 5
   }

   // Make request
   response, err := client.Get(targetUrl)
   if err != nil {
      log.Fatal("Error making GET request. ", err)
   }
   defer response.Body.Close()

   // Read response
   body, err := ioutil.ReadAll(response.Body)
   if err != nil {
      log.Fatal("Error reading body of response. ", err)
   }
   log.Println(string(body))
}
```

# 摘要

在本章中，我们介绍了使用 Go 编写 Web 服务器的基础知识。您现在应该可以轻松创建基本的 HTTP 和 HTTPS 服务器。此外，您应该了解中间件的概念，并知道如何使用 Negroni 包来实现预构建和自定义中间件。

我们还介绍了在尝试保护 Web 服务器时的一些最佳实践。您应该了解 CSRF 攻击是什么，以及如何防止它。您应该能够解释本地和远程文件包含以及风险是什么。

标准库中的 Web 服务器具有生产质量，并且具有创建生产就绪 Web 应用程序所需的一切。还有许多其他用于 Web 应用程序的框架，例如 Gorilla、Revel 和 Martini，但是，最终，您将不得不评估每个框架提供的功能，并查看它们是否符合您的项目需求。

我们还介绍了标准库提供的 HTTP 客户端功能。您应该知道如何进行基本的 HTTP 请求和使用客户端证书进行身份验证的请求。您应该了解在进行请求时如何使用 HTTP 代理。

在下一章中，我们将探讨网络爬虫，以从 HTML 格式的网站中提取信息。我们将从基本技术开始，例如字符串匹配和正则表达式，并探讨用于处理 HTML DOM 的`goquery`包。我们还将介绍如何使用 cookie 在登录会话中爬取。还讨论了指纹识别 Web 应用程序以识别框架。我们还将介绍使用广度优先和深度优先方法爬取网络。
