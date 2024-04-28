# 第十二章：处理我们的 REST 服务的身份验证

在本章中，我们将探讨 Go 中的身份验证模式。这些模式是基于会话的身份验证、JSON Web Tokens（JWT）和 Open Authentication 2（OAuth2）。我们将尝试利用 Gorilla 包的 sessions 库来创建基本会话。然后，我们将尝试进入高级 REST API 身份验证策略，比如使用无状态 JWT。最后，我们将看到如何实现我们自己的 OAuth2，并了解有哪些包可用来提供给我们现成的 OAuth2 实现。在上一章中，API 网关为我们实现了身份验证（使用插件）。如果 API 网关不在我们的架构中，我们如何保护我们的 API？你将在本章中找到答案。

在本章中，我们将涵盖以下主题：

+   认证工作原理

+   介绍 Postman，一个用于测试 API 的可视化客户端

+   Go 中基于会话的身份验证

+   引入 Redis 来存储用户会话

+   介绍 JSON Web Tokens（JWT）

+   OAuth2 架构和基础知识

# 获取代码

您可以在[`github.com/narenaryan/gorestful/tree/master/chapter12`](https://github.com/narenaryan/gorestful/tree/master/chapter12)获取本章的代码示例。由于示例程序不是包，读者需要按照 GOPATH 的方式创建项目文件。

# 认证工作原理

传统上，身份验证或简单身份验证以会话为中心的方式工作。请求服务器资源的客户端试图证明自己是任何给定资源的合适消费者。流程开始如下。客户端使用用户凭据向服务器发送身份验证请求。服务器接受这些凭据并将其与服务器上存储的凭据进行匹配。如果匹配成功，它会在响应中写入一个称为 cookie 的东西。这个 cookie 是一小段信息，传输到后续请求中。现代网站的用户界面（UI）是单页应用程序（SPA）。在那里，静态网页资产如 HTML、JS 是从 CDN 提供的，以渲染网页。从下一次开始，网页和应用服务器之间的通信只通过 REST API/Web 服务进行。

会话是记录用户在一定时间内的通信的一种好方法。会话通常存储在 cookie 中。以下图表可以总结认证（简称 auth）的整个过程：

![](img/a377d17c-27dc-4a16-b66c-fa5adbdbc5ed.png)

现在看看实际的方法。客户端（例如浏览器）向服务器的登录 API 发送请求。服务器尝试使用数据库检查这些凭据，如果凭据存在，就会在响应中写入一个 cookie，表示这个用户已经通过身份验证。cookie 是服务器在以后的时间点要消耗的消息。当客户端接收到响应时，它会在本地存储该 cookie。如果是 Web 浏览器是客户端，它会将其存储在 cookie 存储中。从下一次开始，客户端可以自由地通过显示 cookie 作为通行证来请求服务器的资源。当客户端决定终止会话时，它调用服务器上的注销 API。服务器在响应中销毁会话。这个过程继续进行。服务器还可以在 cookie 上设置过期时间，以便在没有活动的情况下，认证窗口在一定时间内有效。这就是所有网站的工作原理。

现在，我们将尝试使用 Gorilla kit 的`sessions`包来实现这样的系统。我们已经在最初的章节中看到了 Gorilla kit 如何提供 HTTP 路由。这个 sessions 包就是其中之一。我们需要首先使用以下命令安装这个包：

```go
go get github.com/gorilla/sessions
```

现在，我们可以使用以下语句创建一个新的会话：

```go
var store = sessions.NewCookieStore([]byte("secret_key"))
```

`secret_key`应该是 Gorilla sessions 用来加密会话 cookie 的密钥。如果我们将会话添加为普通文本，任何人都可以读取它。因此，服务器需要将消息加密为一个随机字符串。为此，它要求提供一个密钥。这个密钥可以是任何随机生成的字符串。将密钥保存在代码中并不是一个好主意，所以我们尝试将其存储为环境变量，并在代码中动态读取它。我们将看到如何实现这样一个系统。

# 基于会话的身份验证

在 GOPATH 中创建一个名为`simpleAuth`的项目，并添加一个名为`main.go`的文件，其中包含我们程序的逻辑：

```go
mkdir simpleAuth
touch main.py
```

在这个程序中，我们将看到如何使用 Gorilla sessions 包创建基于会话的身份验证。参考以下代码片段：

```go
package main
import (
    "log"
    "net/http"
    "os"
    "time"
    "github.com/gorilla/mux"
    "github.com/gorilla/sessions"
)
var store =
sessions.NewCookieStore([]byte(os.Getenv("SESSION_SECRET")))
var users = map[string]string{"naren": "passme", "admin": "password"}
// HealthcheckHandler returns the date and time
func HealthcheckHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    if (session.Values["authenticated"] != nil) && session.Values["authenticated"] != false {
        w.Write([]byte(time.Now().String()))
    } else {
        http.Error(w, "Forbidden", http.StatusForbidden)
    }
}
// LoginHandler validates the user credentials
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    err := r.ParseForm()
    if err != nil {
        http.Error(w, "Please pass the data as URL form encoded",
http.StatusBadRequest)
        return
    }
    username := r.PostForm.Get("username")
    password := r.PostForm.Get("password")
    if originalPassword, ok := users[username]; ok {
        if password == originalPassword {
            session.Values["authenticated"] = true
            session.Save(r, w)
        } else {
            http.Error(w, "Invalid Credentials", http.StatusUnauthorized)
            return
        }
    } else {
        http.Error(w, "User is not found", http.StatusNotFound)
        return
    }
    w.Write([]byte("Logged In successfully"))
}
// LogoutHandler removes the session
func LogoutHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    session.Values["authenticated"] = false
    session.Save(r, w)
    w.Write([]byte(""))
}
func main() {
    r := mux.NewRouter()
    r.HandleFunc("/login", LoginHandler)
    r.HandleFunc("/healthcheck", HealthcheckHandler)
    r.HandleFunc("/logout", LogoutHandler)
    http.Handle("/", r)
    srv := &http.Server{
        Handler: r,
        Addr: "127.0.0.1:8000",
        // Good practice: enforce timeouts for servers you create!
        WriteTimeout: 15 * time.Second,
        ReadTimeout: 15 * time.Second,
    }
    log.Fatal(srv.ListenAndServe())
}
```

这是一个 REST API，允许用户访问系统的健康状况（正常或异常）。为了进行身份验证，用户需要首先调用登录端点。该程序导入了两个名为 mux 和 sessions 的主要包，来自 Gorilla kit。Mux 用于将 HTTP 请求的 URL 端点链接到函数处理程序，sessions 用于在运行时创建新会话和验证现有会话。

在 Go 中，我们需要将会话存储在程序内存中。我们可以通过创建`CookieStore`来实现。这行明确告诉程序从名为`SESSION_SECRET`的环境变量中选择一个密钥来创建一个密钥。

```go
var store = sessions.NewCookieStore([]byte(os.Getenv("SESSION_SECRET")))
```

`sessions`有一个名为`NewCookieStore`的新函数，返回一个存储。我们需要使用这个存储来管理 cookie。我们可以通过这个语句获取一个 cookie 会话。如果会话不存在，它将返回一个空的会话：

```go
session, _ := store.Get(r, "session.id")
```

`session.id`是我们为会话指定的自定义名称。使用这个名称，服务器将在客户端响应中发送一个 cookie。`LoginHandler`尝试解析客户端提供的多部分表单数据。这一步在程序中是必不可少的：

```go
err := r.ParseForm()
```

这将使用解析后的键值对填充`r.PostForm`映射。该 API 需要用户名和密码进行身份验证。因此，我们对`username`和`password`感兴趣。一旦`LoginHandler`接收到数据，它会尝试与名为**users**的映射中的详细信息进行检查。在实际场景中，我们使用数据库来验证这些详细信息。为了简单起见，我们硬编码了值并尝试进行身份验证。如果用户名不存在，则返回一个资源未找到的错误。如果用户名存在但密码不正确，则返回一个`UnAuthorized`错误消息。如果一切顺利，通过设置 cookie 值返回一个 200 响应，如下所示：

```go
session.Values["authenticated"] = true
session.Save(r, w)
```

第一条语句将名为`"authenticated"`的 cookie 键设置为`true`。第二条语句实际上将会话保存在响应中。它以请求和响应写入器作为参数。如果我们删除这个语句，cookie 将不会产生任何效果。现在，来看看`HealthCheckHandler`，它最初与`LoginHandler`做同样的事情，如下所示：

```go
session, _ := store.Get(r, "session.id")
```

然后，它检查给定的请求是否具有名为`"authenticated"`的 cookie 键。如果该键存在且为 true，则表示之前服务器经过身份验证的用户。但是，如果该键不存在或`"authenticated"`值为`false`，则会话无效，因此返回一个`StatusForbidden`错误。

客户端应该有一种方式来使登录会话失效。它可以通过调用服务器的注销 API 来实现。该 API 只是将`"authenticated"`值设置为`false`。这告诉服务器客户端未经身份验证：

```go
session, _ := store.Get(r, "session.id")
session.Values["authenticated"] = false
session.Save(r, w)
```

通过这种方式，可以在任何编程语言中使用会话来实现简单的身份验证，包括 Go。

不要忘记添加这个语句，因为这是实际修改和保存 cookie 的语句：`session.Save(r, w)`。

现在，让我们看看这个程序的执行。与其使用 CURL，我们可以使用一个名为 Postman 的绝妙工具。其主要好处是它可以在包括 Microsoft Window 在内的所有平台上运行；不再需要 CURL 了。

错误代码可能意味着不同的事情。例如，当用户尝试在没有身份验证的情况下访问资源时，会发出 Forbidden（403）错误，而当给定资源在服务器上不存在时，会发出 Resource Not Found（404）错误。

# 介绍 Postman，一个用于测试 REST API 的工具

Postman 是一个很棒的工具，允许 Windows、macOS X 和 Linux 用户进行 HTTP API 请求。您可以在[`www.getpostman.com/`](https://www.getpostman.com/)下载它。

安装 Postman 后，在“输入请求 URL”文本框中输入 URL。选择请求类型（`GET`、`POST`等）。对于每个请求，我们可以有许多设置，如头部、`POST`主体和其他详细信息。请查阅 Postman 文档以获取更多详细信息。Postman 的基本用法很简单。请看下面的截图：

![](img/0e02ffe3-9719-4ff1-9d82-832c209ba65d.png)

构建器是我们可以添加/编辑请求的窗口。上面的截图显示了我们尝试发出请求的空构建器。运行上面的`simpleAuth`项目中的`main.go`，尝试调用健康检查 API，就像这样。单击发送按钮，您会看到响应被禁止：

![](img/42a5cf72-dea0-41d6-b3a2-917401543995.png)

这是因为我们还没有登录。Postman 在身份验证成功后会自动保存 cookie。现在，将方法类型从`GET`更改为 POST，URL 更改为`http://localhost:8000/login`，调用登录 API。我们还应该将 auth 详细信息作为多部分表单数据传递。它看起来像下面的截图：

![](img/cb63a0a0-deb2-444c-a841-d642d390a511.png)

现在，如果我们点击发送，它会进行身份验证并接收 cookie。它返回一条消息，说成功登录。我们还可以通过点击右侧“发送”和“保存”按钮下方的 Cookies 链接来检查 cookies。它会显示已保存的 cookie 列表，你会在那里找到一个名为`session.id`的 cookie，内容看起来像这样：

```go
session.id=MTUwODYzNDcwN3xEdi1CQkFFQ180SUFBUkFCRUFBQUpmLUNBQUVHYzNSeWFXNW5EQThBRFdGMWRHaGxiblJwWTJGMFpXUUVZbTl2YkFJQ0FBRT189iF-ruBQmyTdtAOaMR-Rr9lNtsf1OJgirBDkcBpdEa0=; path=/; domain=localhost; Expires=Tue Nov 21 2017 01:11:47 GMT+0530 (IST);
```

尝试再次调用健康检查 API，它会返回系统日期和时间：

```go
2017-10-22 06:54:36.464214959 +0530 IST
```

如果客户端向注销 API 发出`GET`请求：

```go
http://localhost:8000/logout
```

会话将被使无效，并且访问资源将被禁止，直到进行另一个登录请求。

# 使用 Redis 持久化客户端会话

到目前为止我们创建的会话都存储在程序内存中。这意味着如果程序崩溃或重新启动，所有已登录的会话都将丢失。客户端需要重新进行身份验证以获取新的会话 cookie。有时这可能会很烦人。为了将会话保存在某个地方，我们选择了**Redis**。Redis 是一个键值存储，非常快，因为它存在于主内存中。

Redis 服务器存储我们提供的任何键值对。它提供基本的数据类型，如字符串、列表、哈希、集合等。有关更多详细信息，请访问[`redis.io/topics/data-types`](https://redis.io/topics/data-types)。我们可以在 Ubuntu 16.04 上使用以下命令安装 Redis：

```go
sudo apt-get install redis-server
```

在 macOS X 上，我们可以这样说：

```go
brew install redis
```

对于 Windows，也可以在 Redis 网站上找到二进制文件。安装 Redis 后，可以使用以下命令启动 Redis 服务器：

```go
redis-server
```

它在默认端口`6379`上启动服务器。现在，我们可以使用 Redis CLI（命令行工具）在其中存储任何内容。打开一个新的终端，输入`redis-cli`。一旦启动了 shell，我们可以执行 Redis 命令将数据存储和检索到用户定义的类型变量中：

```go
[7:30:30] naren:~ $ redis-cli
127.0.0.1:6379> SET Foo  1
OK
127.0.0.1:6379> GET Foo
"1"
```

我们可以使用`SET` Redis 命令存储键值。它将值存储为字符串。如果我们尝试执行`GET`，它会返回字符串。我们有责任将它们转换为数字。Redis 为我们提供了方便的函数来操作这些键。例如，我们可以像这样递增一个键：

```go
127.0.0.1:6379> INCR Foo
(integer) 2
```

Redis 在内部将整数视为整数。如果尝试递增非数字字符串，Redis 会抛出错误：

```go
127.0.0.1:6379> SET name "redis"
OK
127.0.0.1:6379> INCR name
(error) ERR value is not an integer or out of range
```

为什么我们在这里讨论 Redis？因为我们正在展示 Redis 的工作原理，并介绍 Redis 服务器上的一些基本命令。我们将把项目从`simpleAuth`修改为`simpleAuthWithRedis`。

在该项目中，我们使用 Redis 而不是在程序内存中存储会话。即使程序崩溃，会话也不会丢失，因为它们保存在外部服务器中。谁为此编写了桥接逻辑？我们应该。幸运的是，我们有一个包来处理 Redis 和 Go 会话包之间的协调。

使用以下命令安装该包：

```go
go get gopkg.in/boj/redistore.v1
```

并创建一个带有一些修改的新程序。在这里，我们使用`redistore`包，而不是使用会话库。`redistore`有一个名为`NewRediStore`的函数，它以 Redis 配置作为参数以及秘钥。所有其他函数保持不变。现在，在`simpleAuthWithRedis`目录中添加一个`main.go`文件：

```go
package main
import (
    "log"
    "net/http"
    "os"
    "time"
    "github.com/gorilla/mux"
    redistore "gopkg.in/boj/redistore.v1"
)
var store, err = redistore.NewRediStore(10, "tcp", ":6379", "", []byte(os.Getenv("SESSION_SECRET")))
var users = map[string]string{"naren": "passme", "admin": "password"}
// HealthcheckHandler returns the date and time
func HealthcheckHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    if (session.Values["authenticated"] != nil) && session.Values["authenticated"] != false {
        w.Write([]byte(time.Now().String()))
    } else {
        http.Error(w, "Forbidden", http.StatusForbidden)
    }
}
// LoginHandler validates the user credentials
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    err := r.ParseForm()
    if err != nil {
        http.Error(w, "Please pass the data as URL form encoded", http.StatusBadRequest)
        return
    }
    username := r.PostForm.Get("username")
    password := r.PostForm.Get("password")
    if originalPassword, ok := users[username]; ok {
        if password == originalPassword {
            session.Values["authenticated"] = true
            session.Save(r, w)
        } else {
            http.Error(w, "Invalid Credentials", http.StatusUnauthorized)
            return
        }
    } else {
        http.Error(w, "User is not found", http.StatusNotFound)
        return
    }
    w.Write([]byte("Logged In successfully"))
}
// LogoutHandler removes the session
func LogoutHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    session.Options.MaxAge = -1
    session.Save(r, w)
    w.Write([]byte(""))
}
func main() {
    defer store.Close()
    r := mux.NewRouter()
    r.HandleFunc("/login", LoginHandler)
    r.HandleFunc("/healthcheck", HealthcheckHandler)
    r.HandleFunc("/logout", LogoutHandler)
    http.Handle("/", r)
    srv := &http.Server{
        Handler: r,
        Addr: "127.0.0.1:8000",
        // Good practice: enforce timeouts for servers you create!
        WriteTimeout: 15 * time.Second,
        ReadTimeout: 15 * time.Second,
    }
    log.Fatal(srv.ListenAndServe())
}
```

一个有趣的变化是，我们删除了会话，而不是将其值设置为`false`：

```go
  session.Options.MaxAge = -1
```

这个改进的程序与之前的程序完全相同，只是会话保存在 Redis 中。打开 Redis CLI 并输入以下命令以获取所有可用的键：

```go
[15:09:48] naren:~ $ redis-cli
127.0.0.1:6379> KEYS *
1) "session_VPJ54LWRE4DNTYCLEJWAUN5SDLVW6LN6MLB26W2OB4JDT26CR2GA"
127.0.0.1:6379>
```

那个冗长的`"session_VPJ54LWRE4DNTYCLEJWAUN5SDLVW6LN6MLB26W2OB4JDT26CR2GA"`是由`redistore`存储的键。如果我们删除该键，客户端将自动被禁止访问资源。现在停止运行程序并重新启动。您会看到会话没有丢失。通过这种方式，我们可以保存客户端会话。我们也可以在 SQLite 数据库上持久化会话。许多第三方包都是为了使这一点更容易而编写的。

**Redis**可以用作 Web 应用程序的缓存。它可以存储临时数据，如会话、频繁请求的用户内容等。通常与**memcached**进行比较。

# JSON Web Tokens（JWT）和 OAuth2 简介

以前的身份验证方式是明文用户名/密码和基于会话的。它有一个通过将它们保存在程序内存或 Redis/SQLite3 中来管理会话的限制。现代 REST API 实现了基于令牌的身份验证。在这里，令牌可以是服务器生成的任何字符串，允许客户端通过显示令牌来访问资源。在这里，令牌是以这样一种方式计算的，即客户端和服务器只知道如何编码/解码令牌。**JWT**试图通过使我们能够创建可以传递的令牌来解决这个问题。

每当客户端将认证详细信息传递给服务器时，服务器会生成一个令牌并将其传递回客户端。客户端将其保存在某种存储中，例如数据库或本地存储（在浏览器的情况下）。客户端使用该令牌向服务器定义的任何 API 请求资源：

![](img/976d6e2e-da56-43e7-9c9c-ad288472b2ac.png)

这些步骤可以更简要地总结如下：

1.  客户端将用户名/密码在`POST`请求中传递给登录 API。

1.  服务器验证详细信息，如果成功，生成 JWT 并返回，而不是创建 cookie。客户端有责任存储这个令牌。

1.  现在，客户端有了 JWT。它需要在后续的 REST API 调用中添加这个令牌，比如`GET`、`POST`、`PUT`和`DELETE`。

1.  服务器再次检查 JWT，如果成功解码，服务器通过查看作为令牌一部分提供的用户名发送数据。

JWT 确保数据来自正确的客户端。创建令牌的技术负责处理这个逻辑。JWT 利用基于秘钥的加密。

# JSON web token 格式

我们在前面的部分讨论的一切都围绕着 JWT 令牌。我们将在这里看到它真正的样子以及它是如何生成的。JWT 是在执行几个步骤后生成的字符串。它们如下：

1.  通过对标头 JSON 进行**Base64Url**编码来创建 JWT 标头。

1.  通过对有效负载 JSON 进行**Base64Url**编码来创建 JWT 有效负载。

1.  通过使用秘钥对附加的标头和有效负载进行加密来创建签名。

1.  JWT 字符串可以通过附加标头、有效负载和签名来获得。

标头是一个简单的 JSON 对象。在 Go 中，它看起来像以下代码片段：

```go
`{
  "alg": "HS256",
  "typ": "JWT"
}`
```

`"alg"`是用于创建签名的算法（HMAC 与 SHA-256）的简写形式。消息类型是`"JWT"`。这对所有标头都是通用的。算法可能会根据系统而变化。

有效负载看起来像这样：

```go
`{
  "sub": "1234567890",
  "username": "Indiana Jones",
  "admin": true
}`
```

有效负载对象中的键称为声明。声明是指定服务器某些特殊含义的键。有三种类型的声明：

+   公共声明

+   私有声明（更重要）

+   保留声明

# 保留声明

保留声明是由 JWT 标准定义的声明。它们是：

+   iat: 发行时间

+   iss: 发行者名称

+   sub: 主题文本

+   aud: 受众名称

+   exp: 过期时间

例如，服务器在生成令牌时可以在有效负载中设置一个`exp`声明。然后客户端使用该令牌来访问 API 资源。服务器每次验证令牌时。当过期时间过去时，服务器将不再验证令牌。客户端需要通过重新登录生成新的令牌。

# 私有声明

私有声明是用来识别一个令牌与另一个令牌的名称。它可以用于授权。授权是识别哪个客户端发出了请求的过程。多租户是在系统中有多个客户端。服务器可以在令牌的有效负载上设置一个名为`username`的私有声明。下次，服务器可以读取这个有效负载并获取用户名，然后使用该用户名来授权和自定义 API 响应。

`"username": "Indiana Jones"`是前面示例有效负载上的私有声明。**公共声明**类似于私有声明，但它们应该在 IANA JSON Web Token 注册表中注册为标准。我们限制了这些的使用。

可以通过执行以下操作来创建签名（这不是代码，只是一个示例）：

```go
signature = HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

这只是对使用秘钥的 Base64URL 编码的标头和有效负载执行加密算法。这个秘钥可以是任何字符串。它与我们在以前的 cookie 会话中使用的秘钥完全相似。这个秘钥通常保存在环境变量中，并加载到程序中。

现在我们附加编码的标头、编码的有效负载和签名以获得我们的令牌字符串：

```go
tokenString = base64UrlEncode(header) + "." + base64UrlEncode(payload) + "." + signature
```

这就是 JWT 令牌是如何生成的。我们在 Go 中要手动做所有这些事情吗？不。在 Go 或任何其他编程语言中，有一些可用的包来包装令牌的手动创建和验证。Go 有一个名为`jwt-go`的精彩、流行的包。我们将在下一节中创建一个使用`jwt-go`来签署 JWT 并验证它们的项目。可以使用以下命令安装该包：

```go
go get github.com/dgrijalva/jwt-go 
```

这是该项目的官方 GitHub 页面：[`github.com/dgrijalva/jwt-go`](https://github.com/dgrijalva/jwt-go)。该包提供了一些函数，允许我们创建令牌。还有许多其他具有不同附加功能的包。您可以在[`jwt.io/#libraries-io`](https://jwt.io/#libraries-io)上查看所有可用的包和支持的功能。

# 在 Go 中创建 JWT

`jwt-go`包有一个名为`NewWithClaims`的函数，它接受两个参数：

1.  签名方法如 HMAC256、RSA 等

1.  声明映射

例如，它看起来像以下代码片段：

```go
token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
    "username": "admin",
    "iat":time.Now().Unix(),
})
```

`jwt.SigningMethodHS256`是包中可用的加密算法。第二个参数是一个带有声明的映射，例如私有（这里是用户名）和保留（发行于）。现在我们可以使用`SignedString`函数在令牌上生成一个`tokenString`：

```go
tokenString, err := token.SignedString("my_secret_key")
```

然后应将此`tokenString`传回客户端。

# 在 Go 中读取 JWT

`jwt-go`还为我们提供了解析给定 JWT 字符串的 API。`Parse`函数接受字符串和密钥函数作为参数。`key`函数是一个自定义函数，用于验证算法是否正确。假设这是由前面的编码生成的示例令牌字符串：

```go
tokenString = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoiMTUwODc0MTU5MTQ2NiJ9.5m6KkuQFCgyaGS_xcVy4xWakwDgtAG3ILGGTBgYVBmE"

```

我们可以解析并获取原始的 JSON 使用：

```go
token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
    // key function
    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("Unexpected signing method: %v", token.Header["alg"])
    }
    return "my_secret_key", nil
})

if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
    // Use claims for authorization if token is valid
    fmt.Println(claims["username"], claims["iat"])
} else {
    fmt.Println(err)
}
```

`token.Claims`由一个名为`MapClaims`的映射实现。我们可以从该映射中获取原始的 JSON 键值对。

# OAuth 2 架构和基础知识

OAuth 2 是用于在不同系统之间创建身份验证模式的身份验证框架。在此，客户端不是向资源服务器发出请求，而是首先请求某个名为资源所有者的实体。资源所有者返回客户端的身份验证授权（如果凭据成功）。客户端现在将此身份验证授权发送到另一个名为身份验证服务器的实体。此身份验证服务器接受授权并返回访问令牌。此令牌是客户端访问 API 资源的关键。它需要使用此访问令牌向资源服务器发出 API 请求，并提供响应。在整个流程中，第二部分可以使用 JWT 完成。在此之前，让我们了解身份验证和授权之间的区别。

# 身份验证与授权

**身份验证**是识别客户端是否真实的过程。当服务器对客户端进行身份验证时，它会检查用户名/密码对并创建会话 cookie/JWT。

**授权**是在成功身份验证后区分一个客户端与另一个客户端的过程。在云服务中，客户端请求的资源需要通过检查资源是否属于该客户端而不是其他客户端来提供。不同客户端的权限和资源访问也不同。例如，管理员拥有资源的最高权限。普通用户的访问受到限制。

OAuth2 是用于对多个客户端进行身份验证的协议，而 JWT 是一种令牌格式。我们需要对 JWT 令牌进行编码/解码以实现 OAuth 2 的第二阶段（以下截图中的虚线）。

看一下以下图表：

![](img/e52ff092-f411-4ce0-b04b-536942867fa2.png)

在这个图表中，我们可以使用 JWT 实现虚线部分。身份验证发生在身份验证服务器级别，授权发生在资源服务器级别。

在下一节中，让我们编写一个程序，完成两件事：

1.  对客户端进行身份验证并返回 JWT 字符串。

1.  通过验证 JWT 授权客户端 API 请求。

创建一个名为`jwtauth`的目录并添加`main.go`：

```go
package main
import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "time"
    jwt "github.com/dgrijalva/jwt-go"
    "github.com/dgrijalva/jwt-go/request"
    "github.com/gorilla/mux"
)
var secretKey = []byte(os.Getenv("SESSION_SECRET"))
var users = map[string]string{"naren": "passme", "admin": "password"}
// Response is a representation of JSON response for JWT
type Response struct {
    Token string `json:"token"`
    Status string `json:"status"`
}
// HealthcheckHandler returns the date and time
func HealthcheckHandler(w http.ResponseWriter, r *http.Request) {
    tokenString, err := request.HeaderExtractor{"access_token"}.ExtractToken(r)
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        // Don't forget to validate the alg is what you expect:
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("Unexpected signing method: %v", token.Header["alg"])
        }
        // hmacSampleSecret is a []byte containing your secret, e.g. []byte("my_secret_key")
        return secretKey, nil
    })
    if err != nil {
        w.WriteHeader(http.StatusForbidden)
        w.Write([]byte("Access Denied; Please check the access token"))
        return
    }
    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        // If token is valid
        response := make(map[string]string)
        // response["user"] = claims["username"]
        response["time"] = time.Now().String()
        response["user"] = claims["username"].(string)
        responseJSON, _ := json.Marshal(response)
        w.Write(responseJSON)
    } else {
        w.WriteHeader(http.StatusForbidden)
        w.Write([]byte(err.Error()))
    }
}
// LoginHandler validates the user credentials
func getTokenHandler(w http.ResponseWriter, r *http.Request) {
    err := r.ParseForm()
    if err != nil {
        http.Error(w, "Please pass the data as URL form encoded", http.StatusBadRequest)
        return
    }
    username := r.PostForm.Get("username")
    password := r.PostForm.Get("password")
    if originalPassword, ok := users[username]; ok {
        if password == originalPassword {
            // Create a claims map
            claims := jwt.MapClaims{
                "username": username,
                "ExpiresAt": 15000,
                "IssuedAt": time.Now().Unix(),
            }
            token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
            tokenString, err := token.SignedString(secretKey)
            if err != nil {
                w.WriteHeader(http.StatusBadGateway)
                w.Write([]byte(err.Error()))
            }
            response := Response{Token: tokenString, Status: "success"}
            responseJSON, _ := json.Marshal(response)
            w.WriteHeader(http.StatusOK)
            w.Header().Set("Content-Type", "application/json")
            w.Write(responseJSON)
        } else {
            http.Error(w, "Invalid Credentials", http.StatusUnauthorized)
            return
        }
    } else {
        http.Error(w, "User is not found", http.StatusNotFound)
        return
    }
}
func main() {
    r := mux.NewRouter()
    r.HandleFunc("/getToken", getTokenHandler)
    r.HandleFunc("/healthcheck", HealthcheckHandler)
    http.Handle("/", r)
    srv := &http.Server{
        Handler: r,
        Addr: "127.0.0.1:8000",
        // Good practice: enforce timeouts for servers you create!
        WriteTimeout: 15 * time.Second,
        ReadTimeout: 15 * time.Second,
    }
    log.Fatal(srv.ListenAndServe())
}
```

这是一个非常冗长的程序。首先，我们导入`jwt-go`及其名为`request`的子包。我们为两个端点创建了一个 REST API；一个用于通过提供身份验证详细信息获取访问令牌，另一个用于获取授权用户的健康检查 API。

在**`getTokenHandler`**处理函数中，我们正在将用户名和密码与我们自定义定义的用户映射进行比较。这也可以是一个数据库。如果身份验证成功，我们将生成一个 JWT 字符串并将其发送回客户端。

在`HealthcheckHandler`中，我们从名为`access_token`的标头中获取访问令牌，并通过解析 JWT 字符串来验证它。谁编写验证逻辑？JWT 包本身。当创建新的 JWT 字符串时，它应该有一个名为`ExpiresAt`的声明。参考以下代码片段：

```go
      claims := jwt.MapClaims{
        "username": username,
        "ExpiresAt": 15000,
        "IssuedAt": time.Now().Unix(),
      } 
```

程序的内部验证逻辑查看`IssuedAt`和`ExpiresAt`声明，并尝试计算并查看给定的令牌是否已过期。如果是新鲜的，那么意味着令牌已验证。

现在，当令牌有效时，我们可以在`HealthCheckHandler`中读取有效载荷，解析作为 HTTP 请求标头的`access_token`字符串。`username`是我们为授权插入的自定义私有声明。因此，我们知道实际发送此请求的是谁。对于每个请求，不需要传递会话。每个 API 调用都是独立的且基于令牌的。信息已经编码在令牌中。

`token.Claims.(jwt.MapClaims)`返回一个值为接口而不是字符串的映射。为了将值转换为字符串，我们应该这样做`claims["username"].(string)`。

让我们通过 Postman 工具来看看这个程序是如何运行的：

![](img/41e2bbd3-3948-4935-856a-88236498565f.png)

这将返回一个包含 JWT 令牌的 JSON 字符串。将其复制到剪贴板。如果您尝试在不传递 JWT 令牌作为其中一个标头的情况下向健康检查 API 发出请求，您将收到此错误消息而不是 JSON：

```go
Access Denied; Please check the access token
```

现在，将该令牌复制回来，并进行`GET`请求，添加一个`access_token`标头，其值为令牌字符串。在 Postman 中，标头部分可用于添加标头和键值对。请参阅以下屏幕截图：

![](img/76d0689e-288d-49e2-a116-ca6449a14e69.png)

它将正确返回时间作为 API 响应的一部分。我们还可以看到这是哪个用户的 JWT 令牌。这证实了我们的 REST API 的授权部分。我们可以将令牌验证逻辑放在每个 API 处理程序中，也可以将其作为中间件，并将其应用于所有处理程序。请参阅第三章，*使用中间件和 RPC*，并修改前面的程序以具有验证 JWT 令牌的中间件。

基于令牌的认证通常不提供注销 API 或用于删除会话基础认证中提供的令牌的 API。只要 JWT 没有过期，服务器就会向客户端提供授权资源。一旦过期，客户端需要刷新令牌，也就是说，向服务器请求一个新令牌。

# 摘要

在本章中，我们介绍了认证的过程。我们看到了认证通常是如何工作的。认证可以分为两种类型：基于会话的认证和基于令牌的认证。基于会话的认证也被称为简单认证，客户端成功登录时会创建一个会话。该会话被保存在客户端并在每个请求中提供。这里有两种可能的情况。在第一种情况下，会话将保存在服务器的程序内存中。当应用程序重新启动时，这种会话将被清除。第二种情况是将会话 cookie 保存在 Redis 中。Redis 是一个可以作为任何 Web 应用程序缓存的内存数据库。Redis 支持存储一些数据类型，如字符串、列表、哈希等。我们探讨了一个名为`redistore`的包，它用于替换用于持久化会话 cookie 的内置会话包。

接下来，我们了解了 JWT。JWT 是执行一些步骤的输出的令牌字符串。首先，创建一个标头、有效载荷和签名。通过使用`base64URL`编码和应用诸如 HMAC 之类的加密算法，可以获得签名。在基于令牌的认证中，客户端需要一个 JWT 令牌来访问服务器资源。因此，最初，它请求服务器提供访问令牌（JWT 令牌）。一旦客户端获得此令牌，下次它使用 JWT 令牌进行 API 调用，并将服务器返回响应。

我们引入了 OAuth 2.0，一个认证框架。在 OAuth 2 中，客户端首先向资源所有者请求授权。一旦获得授权，它就会向认证服务器请求访问令牌。认证服务器会提供访问令牌，客户端可以用它来请求 API。我们用 JWT 实现了 OAuth 2 的第二步。

我们使用一个叫做 Postman 的工具来测试所有的 API。Postman 是一个很棒的工具，可以帮助我们在任何机器上快速测试我们的 API。CURL 只能在 Linux 和 macOS X 上使用。Postman 对于 Windows 来说是一个明智的选择，因为它拥有 CURL 的所有功能。

通过学习如何创建 HTTP 路由、中间件和处理程序，我们从第一章走了很长的路。然后我们将我们的应用程序与数据库连接起来，以存储资源数据。在掌握了基础知识之后，我们探索了微服务和 RPC 等性能调优方面。最后，我们学会了如何部署我们的 Web 服务，并使用认证来保护它们。
