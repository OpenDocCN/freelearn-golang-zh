# 处理我们的 REST 服务的认证

在本章中，我们将探讨**表示状态转移**（**REST**）API 认证模式。这些模式是基于会话的认证、**JSON Web Tokens**（**JWT**）和**开放认证 2**（**OAuth 2.0**）。我们将尝试利用 Gorilla 包的`sessions`库来创建基本会话。然后，我们将继续探讨高级 REST API 认证策略，如无状态 JWT。最后，我们将讨论 OAuth 2.0 认证模式以及 API 的安全方面。在前一章中，**亚马逊网络服务**（**AWS**）API 网关为我们处理了认证（使用**身份和访问管理**（**IAM**）角色）。如果没有 API 网关，我们如何保护我们的 API？您将在本章中找到答案。

在本章中，我们将涵盖以下主题：

+   简单认证是如何工作的

+   介绍 Postman，一个用于测试 REST API 的视觉客户端

+   使用 Redis 持久化客户端会话

+   介绍 JWT 和 OAuth 2.0

+   OAuth 2.0 工作流程中的 JWT

+   读者练习

+   API 的安全方面

# 技术要求

为了运行代码示例，以下软件应预先安装：

+   操作系统：Linux (Ubuntu 18.04)/Windows 10/Mac OS X >= 10.13

+   Go 稳定版本编译器 >= 1.13.5

+   Dep：Go >= 0.5.3 的依赖管理工具

+   Docker 版本 >= 18.09.2

您可以从[`github.com/PacktPublishing/Hands-On-Restful-Web-services-with-Go/tree/master/chapter14`](https://github.com/PacktPublishing/Hands-On-Restful-Web-services-with-Go/tree/master/chapter14)下载本章的代码。克隆代码，并使用`chapter14`目录中的代码示例。

# 简单认证是如何工作的

传统上，认证或简单认证与会话一起工作。流程开始如下。客户端使用用户凭据向服务器发送认证请求。服务器获取这些凭据并将它们与服务器上存储的凭据进行匹配。如果匹配成功，它将在响应中写入一个称为**cookie**的东西。这个 cookie 是一小段信息，客户端会将其传输给所有后续请求。现代网站正在被设计成**单页应用程序**（**SPAs**）。在这些网站上，静态资源如 HTML 和 JavaScript 文件最初是从**内容分发网络**（**CDN**）提供的，以渲染网页。之后，网页与应用服务器之间的通信仅通过 REST API/网络服务进行。

**会话**是一种记录特定时间段内用户通信的好方法。会话是一个概念，其中认证信息存储在 cookie 中。以下图表解释了基于会话的基本认证过程中发生了什么：

![图片](img/474314d9-b218-4c16-ae15-8c72dcd32b49.png)

现在，让我们看看一个实际的方法。一个**客户端**（例如，浏览器）向**服务器**的**登录 API**发送请求。服务器会尝试与数据库核对那些凭证，如果凭证存在，就会在响应中写回一个**cookie**，表示该用户已认证。

**cookie**是一个在稍后时间由服务器消费的消息。当客户端收到响应时，它会将 cookie 本地存储。之后，客户端可以通过显示 cookie 作为通行证的关键来从服务器请求资源。

当客户端决定终止会话时，它会在服务器上调用注销 API。服务器在响应中销毁会话。对于每个登录/注销，这个过程都会重复。服务器还可以对 cookies 设置过期时间，这样在没有活动的情况下，认证窗口在一定时间内有效。这就是许多网站的工作方式。

现在，我们将尝试使用`gorilla/sessions`包实现这样一个系统。我们在第一章中已经学习了 gorilla/mux。首先，我们需要使用以下命令安装该包：

```go
go get github.com/gorilla/mux
go get github.com/gorilla/sessions
```

或者，我们可以通过使用 Dep 工具来完成这项工作，如下所示：

```go
dep init
dep ensure -add github.com/gorilla/mux
dep ensure -add github.com/gorilla/sessions
```

我们可以使用`sessions.NewCookieStore`方法从`sessions`包中创建一个新的会话，如下所示：

```go
var store = sessions.NewCookieStore([]byte("secret_key"))
```

那个`secret_key`应该是`gorilla/sessions`用来加密会话 cookie 的密钥。如果我们以普通文本添加会话，任何人都可以读取它。因此，服务器需要将消息加密为随机字符串。为此，它要求提供秘密密钥。这个秘密密钥可以是任何随机生成的字符串。

将秘密密钥保存在代码中不是一个好主意，所以我们尝试将其存储为环境变量，并在代码中即时读取。在下一节中，我们将查看一个会话认证的示例。

# 一个简单的认证示例

让我们构建一个安全的 API，只有登录后才能让客户端访问。在这个过程中，我们将定义三个端点：

+   `/login`

+   `/logout`

+   `/healthcheck`

`/healthcheck`是数据 API，但首先必须使用`/login`端点进行登录。我们的 API 应该拒绝所有未经认证的请求。创建一个名为`simpleAuth`的项目目录，如下所示：

```go
mkdir -p $GOPATH/src/github.com/git-user/chapter14/simpleAuth
touch $GOPATH/src/github.com/git-user/chapter14/simpleAuth/main.go
```

在程序中，我们可以看到如何使用 gorilla/sessions 包来启用基于会话的 API 端点认证。按照以下步骤操作：

1.  我们需要为我们的程序导入一些内容。主要的是`mux`和`sessions`，如下面的代码块所示：

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
```

1.  现在，创建一个 cookie 存储来存储写入的 cookie 信息。我们可以使用`sessions.NewCookieStore`方法来完成。它需要一个包含秘密密钥的字节序列。秘密密钥从`SESSION_SECRET`环境变量中获取，如下所示：

```go
var store = sessions.NewCookieStore([]byte(os.Getenv("SESSION_SECRET")))
```

您可以设置任何您想要的键到那个环境变量中。

1.  由于我们没有注册机制，让我们创建一些模拟的`用户名`和`密码`，如下所示：

```go
var users = map[string]string{"naren": "passme", "admin": "password"}
```

这些模拟的用户名/密码组合将与客户端请求进行核对。

1.  现在，添加一个设置 cookie 的登录处理程序。API 是一个带有在 POST 主体中提供的凭据的 `POST` 请求，如下所示：

```go
func LoginHandler(w http.ResponseWriter, r *http.Request) {
   ...
}
```

1.  在这个函数中，我们首先应该解析 POST 主体，并获取 `username` 和 `password`，如下所示：

```go
err := r.ParseForm()

if err != nil {
http.Error(w, "Please pass the data as URL form encoded",
 http.StatusBadRequest)
    return
}
username := r.PostForm.Get("username")
password := r.PostForm.Get("password")
```

1.  一旦我们收集了 `username` 和 `password`，我们的计划是使用模拟数据验证它们。如果凭据匹配，则将请求会话设置为已认证。否则，相应地显示错误消息，如下所示：

```go
if originalPassword, ok := users[username]; ok {
    session, _ := store.Get(r, "session.id")
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
```

这样就完成了登录处理程序。

1.  以类似的方式，让我们定义注销处理程序。注销处理程序接收一个进入的 `GET` 请求并将会话变量 authenticated 设置为 `false`，如下所示：

```go
// LogoutHandler removes the session
func LogoutHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    session.Values["authenticated"] = false
    session.Save(r, w)
    w.Write([]byte(""))
}
```

1.  如果您看到了注销处理程序的实现，那么我们已经修改了 `session` 对象以使客户端会话无效，如下所示：

```go
session, _ := store.Get(r, "session.id")
session.Values["authenticated"] = false
session.Save(r, w)
```

1.  以这种方式，可以使用任何编程语言中的客户端会话来实现简单的身份验证，包括 Go。

不要忘记在修改后保存 cookie。这个操作的代码是 `session.Save(r, w)`。

1.  现在，让我们定义我们的 `/healthcheck` 数据 API。这是一个发送系统时间的 API。如果客户端会话已认证，则返回响应。否则，返回 `403 Forbidden` 响应。请求中的 `session` 对象可用于有效性检查，如下所示：

```go
// HealthcheckHandler returns the date and time
func HealthcheckHandler(w http.ResponseWriter, r *http.Request) {
    session, _ := store.Get(r, "session.id")
    if (session.Values["authenticated"] != nil) && session.Values
    ["authenticated"] != false {
        w.Write([]byte(time.Now().String()))
    } else {
        http.Error(w, "Forbidden", http.StatusForbidden)
    }
}
```

1.  所有 API 处理程序都已就绪。我们必须编写 `main` 函数，在该函数中将 API 端点（路由）映射到前面的处理程序。我们为此使用 `mux` 路由器。然后，我们将此路由器传递到在 `http://localhost:8000` 上运行的 HTTP 服务器，如下所示：

```go
func main() {
    r := mux.NewRouter()
    r.HandleFunc("/login", LoginHandler)
    r.HandleFunc("/healthcheck", HealthcheckHandler)
    r.HandleFunc("/logout", LogoutHandler)
    http.Handle("/", r)

    srv := &http.Server{
        Handler: r,
        Addr:    "127.0.0.1:8000",
        // Good practice: enforce timeouts for servers you create!
        WriteTimeout: 15 * time.Second,
        ReadTimeout:  15 * time.Second,
    }
    log.Fatal(srv.ListenAndServe())
}
```

1.  这样就完成了受保护的应用程序。我们可以通过在 `simpleAuth` 根目录下输入以下代码来运行程序：

```go
go run main.go
```

这将在 `http://localhost:8000` 上启动服务器。

错误代码可以表示不同的含义。例如，当用户尝试未经认证访问资源时，会发出 `Forbidden (403)`，而当给定的资源在服务器上不存在时，会发出 `Resource Not Found (404)`。

在下一节中，我们将介绍一个新的查询 API 的工具，称为 Postman。Postman 工具有一个很好的 **用户界面**（**UI**），并且运行在所有平台上。我们将使用这个新工具测试 `simpleAuth` 示例。

# 介绍 Postman，一个用于测试 REST API 的可视化客户端

**Postman** 是一个优秀的 UI 客户端，允许 Windows、Mac OS X 和 Linux 用户进行 HTTP API 请求。您可以从这里下载它：[`www.getpostman.com/product/api-client`](https://www.getpostman.com/product/api-client)[.](https://www.getpostman.com/product/api-client) 我们将使用 Postman 工具而不是使用 `curl` 来发送 API 请求。我们将从上一节中的 `simpleAuth` 示例中选择。请看以下步骤：

1.  安装后，打开 Postman 工具，然后在“输入请求 URL”输入框中尝试一个 [www.example.org](http://www.example.org) URL。您可以从下拉菜单中选择请求类型（GET、POST 等）。对于每个请求，您都可以配置许多设置，如 Headers、POST 主体和其他细节，这些都可以在 URL 下作为菜单进行配置。玩转这些选项，并熟悉它们。

请查阅 Postman 文档以获取更多详细信息。它提供了多种重放 API 查询的选项。花些时间探索 [`learning.postman.com/getting-started/`](https://learning.postman.com/getting-started/) 中的功能。

看看下面的截图：

![图片](img/2c388dc6-8bf9-47fb-ad5f-b937ce2150ea.png)

构建器是顶级菜单项，我们可以在这里添加/编辑请求。前面的截图显示了空白的构建器，我们在这里尝试发送请求。

1.  接下来，运行前面 `simpleAuth` 项目中的 `main.go` 文件，并尝试调用 `/healthcheck` API。点击“发送”按钮。您将看到响应是 403 禁止，如下面的截图所示：

![图片](img/1cddf709-f83d-4317-96d9-f08741f7d390.png)

这是因为我们还没有登录。Postman 在身份验证成功后自动保存 cookie。

1.  现在，通过将方法类型从 GET 改为 POST，并将 URL 改为 http://localhost:8000/login，调用登录 API。我们应该以 `multipart/form-data` 的形式传递凭证。我们应该看到以下内容：

![图片](img/96b735c6-b1b4-45de-8a9a-d1ffba88ff29.png)

1.  点击蓝色“发送”按钮。这会向之前运行的服务器发送登录请求。它返回一条消息，表示登录成功。我们可以通过点击“Cookies”链接来检查 cookies，就在右侧的“保存”按钮旁边。

这显示了保存的 cookies 列表，您将在 localhost 网站找到一个名为 `session.id` 的 cookie。内容看起来像这样：

```go
session.id=MTU3NzI4NTk2NXxEdi1CQkFFQ180SUFBUkFCRUFBQUpmLUNBQUVHYzNSeWFXNW5EQThBRFdGMWRHaGxiblJwWTJGMFpXUUVZbTl2YkFJQ0FBRT18Be0S-fIy6T7U-hnASBnPxLU2gFJ0jnAdaKWI6X04GPo=; path=/; domain=localhost; Expires=Fri, 24 Jan 2020 14:59:25 GMT;
```

1.  再次尝试调用 `/healthcheck` API，该 API 返回系统日期和时间作为响应，如下所示：

```go
2019-12-25 16:00:03.501678 +0100 CET m=+169.811215440
```

假设客户端向 `logout` API 发送另一个 `GET` 请求，如下所示：

```go
http://localhost:8000/logout
```

如果发生这种情况，会话将被无效化，直到完成另一个登录请求，否则将禁止访问资源。

我们示例中使用的管理员和密码凭证仅用于说明目的，不应在生产环境中使用。

总是使用随机生成的强密码！

# 使用 Redis 持久化客户端会话

我们迄今为止创建的会话存储在程序内存中。这意味着如果程序崩溃或重启，所有已记录的会话都将丢失。它需要客户端再次登录以获取新的会话 cookie。这对审计 cookie 没有帮助。为了将会话存储在某个地方，我们可以使用 Redis。

我们在 第九章，*异步 API 设计* 中讨论了在 Docker 容器中运行 Redis。为了回顾，Redis 服务器存储键值对。它提供了基本的数据类型，如字符串、列表、散列、集合等。更多详情，请访问 [`redis.io/topics/data-types`](https://redis.io/topics/data-types)。

现在，是时候将我们的 Redis 知识付诸实践了。我们将修改我们的项目，从 `simpleAuth` 更改为 `simpleAuthWithRedis`。新的项目现在应该使用 Redis 作为会话存储。将前一个示例中的代码复制到新的项目中。

在 第九章，*异步 API 设计* 中，我们使用了 `go-redis` 包从 Go 程序与 Redis 服务器交互。在本章中，我们将介绍一个新的方便的包 `redistore.v1`，这样我们就可以在 Redis 中存储会话。

使用以下 `dep` 命令安装包：

```go
dep init
dep ensure -add gopkg.in/boj/redistore.v1
```

创建一个新的程序，进行一些修改。在这里，我们不会使用 `gorilla/sessions` 包，而是使用 `redistore` 包。`redistore` 包有一个名为 `NewRediStore` 的函数，它接受 Redis 配置作为其参数，以及密钥。它返回与 `gorilla/sessions` 相同的会话对象。

代码的其他部分保持不变。为了简洁，我们只需做出以下更改：

```go
package main
import (
    ...
    redistore "gopkg.in/boj/redistore.v1"
)

var store, err = redistore.NewRediStore(10, "tcp", ":6379", "", []byte(os.Getenv("SESSION_SECRET")))
```

在注销处理程序中，你会看到以下代码：

```go
session.Values["authenticated"] = false
```

将前面的代码更改为以下内容：

```go
  session.Options.MaxAge = -1
```

此步骤从 Redis 存储中删除密钥，相当于从客户端撤销认证。这个改进的程序与之前的程序非常相似，只是会话现在保存在 Redis 中。

运行新的程序，重复使用 Postman 工具中的 API 查询。一旦登录成功，在 Docker 容器中启动 `redis-cli`，如下所示：

```go
docker exec -i -t some-redis redis-cli
```

`some-redis` 是运行容器的 Redis 服务器名称。现在，在 shell 中输入 `KEYS *` 命令以查看新存储的会话，如下所示：

```go
127.0.0.1:6379> KEYS *
1) "session_VPJ54LWRE4DNTYCLEJWAUN5SDLVW6LN6MLB26W2OB4JDT26CR2GA"
127.0.0.1:6379>
```

长达 `session_VPJ54LWRE4DNTYCLEJWAUN5SDLVW6LN6MLB26W2OB4JDT26CR2GA`

`key` 是由 `redistore` 包存储的密钥。如果我们删除该密钥，客户端将无法访问 `/healthcheck` API。现在，停止运行程序并重新启动它。你会看到会话没有被丢失。这样，我们可以保存客户端会话。

Redis 可以作为你的 Web 应用程序的缓存服务。它可以存储临时数据，如会话、频繁请求的用户内容等。它通常与 memcached 相比。

在下一节中，我们将探讨一种新的认证方式，称为 JWT。它不同于会话，并且不会在客户端机器上存储任何 cookies。

# 介绍 JWT 和 OAuth2

现代的 REST API 实现基于令牌的认证。在这里，令牌可以是服务器生成的任何字符串，允许客户端通过生成令牌来访问资源。令牌的计算方式使得只有客户端和服务器知道如何编码/解码令牌。

之前的例子与基于会话的身份验证相关。这有一个限制，即通过在程序内存中保存会话或 Redis/SQLite3 来管理会话。JWT 采取不同的方法，创建了可以用于身份验证的令牌。

当**Client**将身份验证详细信息传递给**Server**时，服务器生成令牌并将其返回给**Client**。客户端将其保存在某种存储中，例如 AWS Secrets Manager、数据库或本地存储（在浏览器的情况下）。**Client**使用该令牌从服务器定义的任何 API 请求资源。这个过程可以在以下图中看到：

![图片](img/288b80ee-76aa-424c-a781-dd4c8cc7af5a.png)

前面图中所显示的步骤可以更简要地总结如下：

1.  **Client**在登录 API 的`POST`请求中传递**用户名/密码**。

1.  **Server**验证详细信息，如果成功，则生成**JWT**并将其返回而不是创建 cookie。存储此令牌是客户端的责任。

1.  现在，**Client**拥有了 JWT。它需要将其添加到头部部分以进行后续的 REST API 调用。

1.  **Server**检查来自头部的 JWT，如果成功解码，则服务器验证客户端。

JWT 确保数据是从正确的客户端发送的。创建令牌的技术负责处理这种逻辑。JWT 利用基于密钥的加密。

# JWT 格式

我们在前面章节中讨论的每一件事都是关于传递 JWT 令牌。在本节中，我们将看到 JWT 的样子以及它是如何生成的。生成 JWT 的高级步骤如下：

1.  通过对 JSON 头部进行`Base64Url`编码来创建一个`JWT`头部。

1.  通过对 JSON 有效载荷进行`Base64Url`编码来创建一个`JWT`有效载荷。

1.  通过使用密钥加密附加的头部和有效载荷来创建一个签名。

1.  通过附加 JWT 头部、JWT 有效载荷和签名可以获得 JWT 字符串。

头部是一个简单的 JSON 对象。它看起来像以下 Go 代码片段：

```go
{
  "alg": "HS256",
  "typ": "JWT"
}
```

`HS256`是用于创建签名的算法（HMAC 与 SHA-256）的简称。消息类型是`JWT`。这将是所有头部的常见格式。算法可能会根据系统而变化。

有效载荷看起来像这样：

```go
{
  "sub": "1234567890",
  "username": "Indiana Jones",
  "admin": true
}
```

有效载荷对象中的键称为**声明**。声明是一个键，它指定了对于服务器的一些特殊含义。有三种类型的声明：

+   保留声明

+   私有声明（更重要）

+   公共声明

我们将在接下来的章节中详细讨论这些声明。

# 保留声明

**保留声明**是由 JWT 标准定义的。它们如下：

+   **iat**: 发布时间

+   **iss**: 发布者名称

+   **sub**: 主题文本

+   **aud**: 受众名称

+   **exp**: 过期时间

例如，服务器在生成令牌时，可以在有效负载中设置一个 exp 声明。然后客户端使用该令牌来访问 API 资源。服务器每次都会验证令牌。当过期时间过去后，服务器将不再验证令牌。客户端需要再次登录以生成新的令牌。

# 私有声明

**私有声明**用于区分一个令牌与另一个令牌。它们可用于授权。**授权**是一个识别哪个客户端发出了请求的过程。**多租户**指的是多个客户端访问系统上的 API 的情况。服务器可以在令牌的有效负载上设置一个名为**用户名**的私有声明。下次，服务器可以读取这个有效负载并获取用户名，然后使用该用户名进行授权并定制 API 响应。这与 cookie 类似，但方式不同。

例如，`username`: `Indiana Jones`是以下有效负载中的私有声明：

```go
{
  "sub": "1234567890",
  "username": "Indiana Jones",
  "admin": true
}
```

**公共声明**与私有声明类似，但它们应该在 IANA JWT 注册表中注册，以使其成为标准。我们限制这些的使用。

可以通过执行此操作创建签名（这不是代码，只是一个说明）：

```go
signature = HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

这只是对 Base64Url 编码的头部和有效负载使用一个秘密进行加密算法。这个秘密可以是任何字符串。它与我们在上一个 cookie 会话中使用的秘密完全相同。这个秘密通常保存在环境变量中，并加载到程序中。

现在，我们将编码的头部、编码的有效负载和签名附加在一起，以获取我们的令牌字符串，如下所示：

```go
tokenString = base64UrlEncode(header) + "." + base64UrlEncode(payload) + "." + signature
```

这就是 JWT 令牌的生成方式。有几个 Go 包可以生成和验证 JWT。其中一个流行的包是`jwt-go`。在下一节中，我们将创建一个使用`jwt-go`来签名 JWT 并验证它的项目。我们可以使用以下`dep`命令安装此包：

```go
dep ensure -add github.com/dgrijalva/jwt-go 
```

这是项目的官方 GitHub 页面：[`github.com/dgrijalva/jwt-go`](https://github.com/dgrijalva/jwt-go)。

包提供了一些函数，允许我们创建令牌。还有许多其他具有不同附加功能的包。您可以在[`jwt.io/#libraries-io`](https://jwt.io/#libraries-io)查看所有可用的包和受支持的特性。

# 在 Go 中创建 JWT

`jwt-go`包有一个名为`NewWithClaims`的函数，它接受两个参数：

+   一种签名方法，如`HMAC256`、`RSA`等

+   声明映射

例如，它看起来像以下代码片段：

```go
token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
    "username": "admin",
    "iat":time.Now().Unix(),
})
```

`jwt.SigningMethodHS256`是包内可用的加密算法。第二个参数是一个包含声明，如私有（这里，用户名）和保留（发行 iat）的映射。现在，我们可以使用令牌上的`SignedString`函数生成`tokenString`，如下所示：

```go
tokenString, err := token.SignedString("my_secret_key")
```

然后，应该将此`tokenString`作为成功登录响应的一部分发送回客户端。

# 在 Go 中读取 JWT

`jwt-go` 还为我们提供了解析给定 JWT 字符串的 API。`Parse` 函数接受一个字符串和一个键函数作为参数。键函数是一个自定义函数，用于验证算法是否真实。让我们假设这是一个由前面的编码生成的示例令牌字符串：

```go
tokenString = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoiMTUwODc0MTU5MTQ2NiJ9.5m6KkuQFCgyaGS_xcVy4xWakwDgtAG3ILGGTBgYVBmE"

```

我们可以使用以下代码解析并获取原始 JSON：

```go
token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
    // key function
    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("Unexpected signing method: %v",
         token.Header["alg"])
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

`token.Claims` 是通过一个名为 **MapClaims** 的映射实现的。我们可以从该映射中获取原始 JSON 的键值对。

在下一节中，我们将讨论 OAuth 2.0 的工作流程以及认证和授权之间的区别。

# JWT 在 OAuth2.0 的工作流程中

**OAuth 2.0** 是一个用于在不同系统之间创建认证模式的认证框架。在这里，客户端不是向资源服务器发出请求，而是向一个称为 **资源所有者** 的实体发出初始请求。这个资源所有者会向客户端发送认证授权（如果凭证已验证）。客户端现在将这个认证授权发送给另一个称为 **认证服务器** 的实体。认证服务器接收授权并返回访问令牌。访问令牌是一个

# 认证与授权的比较

**认证** 是识别客户端是否真实的过程。当服务器认证客户端时，它会检查用户名/密码对并创建一个会话 `cookie/JWT`。

**授权** 是评估用户是否有权访问给定资源的过程。在云服务中，应该有一种机制来限制某些用户/角色的资源访问范围，而授权正是实现这一点的手段。

简而言之，认证决定了一个服务的客户是谁，而授权决定了客户在资源访问方面的界限。

OAuth 2.0 是一个用于认证多个客户端到服务的协议，而 JWT 是一种令牌格式。我们需要对 JWT 令牌进行编码/解码以实现 OAuth 2.0 的第二阶段（以下图中虚线部分）：

![](img/af069940-c17a-4f2e-81b5-ae551da3746b.png)

在前面的图中，虚线部分是客户端从 **认证服务器** 请求访问令牌的地方。这就是 JWT 生命周期开始的地方。

在下一节中，我们将提供一个开发练习供您完成。这个练习可以通过结合我们迄今为止学到的所有概念来完成。

# 练习

您能设计以下要求吗？

开发一个使用令牌认证的 `/healthcheck` API。其主要职责应如下：

+   认证客户端并返回 JWT 字符串

+   通过验证 JWT 授权客户端 API 请求

您应该使用上一节中关于 `jwt-go` 包的知识。您必须构建两个端点，如下所示：

+   `/getToken`

+   `/healthcheck`

第一个端点应该成功登录客户端并返回 JWT 令牌。然后客户端应使用带有令牌的第二个端点来接收成功响应。

开发完成后，最终的 API 测试场景应该看起来类似于以下内容：

1.  如果你向`/healthcheck` API 发出没有令牌的`GET`请求，你应该收到一个`访问被拒绝`的消息，如下所示：

```go
Access Denied; Please check the access token
```

1.  你应该能够通过`POST`请求从 API 服务器进行认证并获取 JWT 令牌，如下面的截图所示：

![图片](img/7b838296-b9e0-42b7-96c3-a7de42d20fb1.png)

1.  使用返回的 JWT，你现在可以成功地向`/healthcheck`发出带有 access_token 头的`GET`请求，如下所示：

![图片](img/b7d4a20e-60cf-4197-a395-45466f0e5769.png)

如果你能够实现前面的要求，你已经对 JWT 有了很好的理解。如果没有，不要担心。挑战的解决方案可以在`chapter14/jwtauth`项目目录中找到。

基于令牌的认证通常不提供退出 API 或删除基于会话认证提供的令牌的 API。只要 JWT 没有过期，服务器就会将授权资源提供给客户端。一旦它过期，客户端就需要刷新令牌——也就是说，请求服务器提供一个新的令牌。

在下一节中，我们将讨论一些确保 API 安全的技巧。

# API 的安全方面

每个开发的 REST API 都可以是公开的或受保护的。公开 API 对请求资源的客户端数量没有限制。但大多数商业 API 都是受保护的。那么，关于安全性，我们应该注意哪些重要事项？在以下列表中，我们将指出所有应该注意的因素，以确保 REST API 的安全：

+   总是使用 HTTPS 来提供 API 的**传输层安全性**（**TLS**）。

+   使用用户访问令牌来限制 API 的速率。

+   在 API 上设计各种认证和授权角色。

+   当客户端和服务器内部时，使用公钥/私钥加密来签名 JWT。

+   永远不要将用户凭证存储在明文文件中。

+   清理 URL 查询参数；使用 POST 体来处理传入的请求。

+   正如我们在上一章中提到的，使用 API 网关以获得更好的性能和保护。

+   使用云服务，如 AWS Secrets Manager 来存储密钥/密码。

大多数现代 API 都在云提供商上作为容器/**虚拟专用服务器**（**VPS**）运行。那里的安全性可以在两个级别上应用：

+   网络级别

+   应用程序级别

在云上开发和部署应用程序的 API 开发者应该意识到上述级别。了解安全漏洞的应对措施和对 API 进行修补是您将需要的核心安全技能。

此外，暴露的 REST API 是攻击如**拒绝服务**（**DoS**）攻击的最容易受害的目标。在防火墙服务后面部署 API 可以增加对这些攻击的安全性。在内部 API 的情况下，公司服务仅与其他内部服务通信，但来自不同的地理区域，VPN 是理想的选择。

安全性是其自身领域的猛兽，需要在 API 开发的各个方面进行仔细监控。

# 摘要

在本章中，我们介绍了身份验证的过程。我们看到了身份验证通常是如何工作的。身份验证可以是三种类型之一：基本身份验证、基于会话的身份验证或基于令牌的身份验证。基本身份验证要求每个 HTTP 请求都提供 `用户名` 和 `密码`。基于会话的身份验证使用保存的会话来验证客户端。

存储在程序内存中的会话一旦网络服务器崩溃/重启就会丢失。可以使用名为 `redistore` 的软件包与 Redis 一起使用，以帮助存储会话 cookie。

接下来，我们学习了 JWT（基于令牌的认证），客户端从服务器请求 JWT 令牌。一旦客户端拥有 JWT 令牌，它可以在请求 API 资源时将令牌传递到 HTTP 头部。

然后，我们介绍了 OAuth 2.0，这是一个身份验证框架。在那里，我们看到了客户端如何从资源所有者请求授权。一旦获得授权，它就会从身份验证服务器请求访问令牌。在从身份验证服务器获得访问令牌后，客户端可以使用该令牌来请求 API。

我们使用 curl 和 Postman 测试了所有我们的 API。Postman 是一款出色的工具，它可以帮助我们在任何机器上快速测试我们的 API，而 curl 则仅限于 Linux 和 Mac OS X。

通过学习如何创建 HTTP 路由、中间件和处理程序，我们从第一章走了很长的路。然后我们探索了 API 的各种 SQL 和 NoSQL 存储后端。在基础知识之后，我们探讨了性能调优方面，如异步设计、GraphQL API 和微服务。最后，我们学习了如何将我们的网络服务部署到云端，并通过启用身份验证来确保它们的安全。
