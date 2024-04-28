# 保护 Go Web 应用程序

在本章中，我们将涵盖以下内容：

+   使用 OpenSSL 创建私钥和 SSL 证书

+   将 HTTP 服务器移动到 HTTPS

+   定义 REST API 和路由

+   创建 JSON Web 令牌

+   使用 JSON Web 令牌保护 RESTful 服务

+   在 Go Web 应用程序中防止跨站点请求伪造

# 介绍

保护 Web 应用程序是本章中我们将学习的最重要的方面之一，除了创建应用程序。应用程序安全是一个非常广泛的主题，可以以超出本章范围的各种方式实现。

在本章中，我们将专注于如何将我们的 Go Web 应用程序从 HTTP 协议移动到 HTTPS，通常称为**HTTP + TLS** **(传输层安全)**，以及使用**JSON Web 令牌** **(JWTs)**保护 Go Web 应用程序 REST 端点，并保护我们的应用程序免受**跨站点请求伪造（CSRF）**攻击。

# 使用 OpenSSL 创建私钥和 SSL 证书

将运行在 HTTP 上的服务器移动到 HTTPS，我们首先要做的是获取 SSL 证书，这可能是自签名的，也可能是由受信任的证书颁发机构（如 Comodo、Symantec 或 GoDaddy）签名的证书。

要获得由受信任的证书颁发机构签名的 SSL 证书，我们必须向他们提供**证书签名请求**（**CSR**），主要包括密钥对的公钥和一些附加信息，而自签名证书是您可以自行签发的证书，用自己的私钥签名。

自签名证书可以用于加密数据，也可以用 CA 签名的证书，但用户将收到一个警告，说证书未被他们的计算机或浏览器信任。因此，您不应该在生产或公共服务器上使用它们。

在这个教程中，我们将学习如何创建私钥、证书签名请求和自签名证书。

# 准备工作…

本教程假设您的机器上已安装了`openssl`。要验证是否已安装，请执行以下命令：

```go
$ openssl
OpenSSL> exit
```

# 如何做…

1.  使用`openssl`执行以下命令生成私钥和证书签名请求：

```go
$ openssl req -newkey rsa:2048 -nodes -keyout domain.key -out domain.csr -subj "/C=IN/ST=Mumbai/L=Andheri East/O=Packt/CN=packtpub.com"
```

这将产生以下输出：

![](img/e4195e84-d3b8-4163-b65e-2562c8c6e5e5.png)

1.  通过执行以下命令生成证书并用刚创建的私钥签名：

```go
$ openssl req -key domain.key -new -x509 -days 365 -out domain.crt -subj "/C=IN/ST=Mumbai/L=Andheri East/O=Packt/CN=packtpub.com"
```

# 工作原理…

一旦命令成功执行，我们可以看到生成了`domain.key`、`domain.csr`和`domain.crt`，其中`domain.key`是用于签署 SSL 证书的 2,048 位 RSA 私钥，而`domain.crt`和`domain.csr`是证书签名请求，包含了密钥对的公钥和一些附加信息，这些信息在签署证书时被插入。

让我们了解我们执行的生成证书签名请求的命令：

+   `-newkey rsa:2048`选项创建一个新的证书请求和一个新的私钥，应该是使用 RSA 算法生成的 2,048 位私钥。

+   `-nodes`选项指定创建的私钥不会使用密码短语加密。

+   `-keyout domain.key`选项指定要将新创建的私钥写入的文件名。

+   `-out domain.csr`选项指定要写入的输出文件名，或者默认情况下为标准输出。

+   `-subj`选项用指定的数据替换输入请求的主题字段，并输出修改后的请求。如果我们不指定此选项，则必须通过`OpenSSL`回答 CSR 信息提示以完成该过程。

接下来，我们将了解我们执行的生成证书并用私钥签名的命令，如下所示：

```go
openssl req -key domain.key -new -x509 -days 365 -out domain.crt -subj "/C=IN/ST=Mumbai/L=Andheri East/O=Packt/CN=packtpub.com"
```

`-key`选项指定从中读取私钥的文件。`-x509`选项输出自签名证书而不是证书请求。`-days 365`选项指定证书的认证天数。默认值为 30 天。

# 将 HTTP 服务器移动到 HTTPS

一旦 Web 应用程序开发结束，我们很可能会将其部署到服务器上。在部署时，建议始终在公开暴露的服务器上使用 HTTPS 协议运行 Web 应用程序，而不是 HTTP。在本教程中，我们将学习如何在 Go 中实现这一点。

# 如何做…

1.  创建`https-server.go`，在其中我们将定义一个处理程序，该处理程序将仅为所有 HTTPS 请求向 HTTP 响应流写入 Hello World！，如下所示：

```go
package main
import 
(
  "fmt"
  "log"
  "net/http"
)
const 
(
  CONN_HOST = "localhost"
  CONN_PORT = "8443"
  HTTPS_CERTIFICATE = "domain.crt"
  DOMAIN_PRIVATE_KEY = "domain.key"
)
func helloWorld(w http.ResponseWriter, r *http.Request) 
{
  fmt.Fprintf(w, "Hello World!")
}
func main() 
{
  http.HandleFunc("/", helloWorld)
  err := http.ListenAndServeTLS(CONN_HOST+":"+CONN_PORT,
  HTTPS_CERTIFICATE, DOMAIN_PRIVATE_KEY, nil)
  if err != nil 
  {
    log.Fatal("error starting https server : ", err)
    return
  }
}
```

1.  使用以下命令运行程序：

```go
$ go run https-server.go
```

# 工作原理…

一旦我们运行程序，HTTPS 服务器将在本地监听端口`8443`上启动。

浏览`https://localhost:8443/`将从服务器获得 Hello World!作为响应：

![](img/b88936f9-4a85-446f-bcc1-41e96312da93.png)

此外，使用`curl`从命令行执行`GET`请求并传递`--insecure`标志将跳过证书验证，因为我们使用的是自签名证书：

```go
$ curl -X GET https://localhost:8443/ --insecure
 Hello World!
```

让我们了解我们编写的程序：

+   `const (CONN_HOST = "localhost" CONN_PORT = "8443" HTTPS_CERTIFICATE = "domain.crt" DOMAIN_PRIVATE_KEY = "domain.key")`：在这里，我们声明了四个常量—`CONN_HOST`的值为`localhost`，`CONN_PORT`的值为`8443`，`HTTPS_CERTIFICATE`的值为`domain.crt`或自签名证书，`DOMAIN_PRIVATE_KEY`的值为`domain.key`或我们在上一个教程中创建的私钥。

+   `func helloWorld(w http.ResponseWriter, r *http.Request) { fmt.Fprintf(w, "Hello World!") }`：这是一个 Go 函数，它以`ResponseWriter`和`Request`作为输入参数，并在 HTTP 响应流上写入`Hello World!`。

接下来，我们声明了`main()`，程序从这里开始执行。由于这个方法做了很多事情，让我们逐行理解它：

+   `http.HandleFunc("/", helloWorld)`: 在这里，我们使用`net/http`包的`HandleFunc`将`helloWorld`函数注册到 URL 模式`/`，这意味着每当我们访问 HTTPS URL 模式`/`时，`helloWorld`都会被执行，并将`(http.ResponseWriter, *http.Request)`作为输入传递给它。

+   `err := http.ListenAndServeTLS(CONN_HOST+":"+CONN_PORT, HTTPS_CERTIFICATE, DOMAIN_PRIVATE_KEY, nil)`: 在这里，我们调用`http.ListenAndServeTLS`来提供处理每个传入连接的 HTTPS 请求的请求。`ListenAndServeTLS`接受四个参数—服务器地址、SSL 证书、私钥和处理程序。在这里，我们将服务器地址传递为`localhost:8443`，我们的自签名证书、私钥和处理程序为`nil`，这意味着我们要求服务器使用`DefaultServeMux`作为处理程序。

+   `if err != nil { log.Fatal("error starting https server : ", err) return}`：在这里，我们检查启动服务器时是否有任何问题。如果有问题，则记录错误并以状态码 1 退出。

# 定义 REST API 和路由

在编写 RESTful API 时，很常见的是在允许用户访问之前对用户进行身份验证。身份验证用户的先决条件是创建 API 路由，我们将在本教程中介绍。

# 如何做…

1.  使用`go get`命令安装`github.com/gorilla/mux`和`github.com/gorilla/handlers`包，如下所示：

```go
$ go get github.com/gorilla/mux
$ go get github.com/gorilla/handlers
```

1.  创建`http-rest-api.go`，在其中我们将定义三个路由—`/status`、`/get-token`和`/employees`—以及它们的处理程序，如下所示：

```go
package main
import 
(
  "encoding/json"
  "log"
  "net/http"
  "os"
  "github.com/gorilla/handlers"
  "github.com/gorilla/mux"
)
const 
(
  CONN_HOST = "localhost"
  CONN_PORT = "8080"
)
type Employee struct 
{
  Id int `json:"id"`
  FirstName string `json:"firstName"`
  LastName string `json:"lastName"`
}
type Employees []Employee
var employees []Employee
func init() 
{
  employees = Employees
  {
    Employee{Id: 1, FirstName: "Foo", LastName: "Bar"},
    Employee{Id: 2, FirstName: "Baz", LastName: "Qux"},
  }
}
func getStatus(w http.ResponseWriter, r *http.Request) 
{
  w.Write([]byte("API is up and running"))
}
func getEmployees(w http.ResponseWriter, r *http.Request) 
{
  json.NewEncoder(w).Encode(employees)
}
func getToken(w http.ResponseWriter, r *http.Request) 
{ 
  w.Write([]byte("Not Implemented"))
}
func main() 
{
  router := mux.NewRouter().StrictSlash(true)
  router.HandleFunc("/status", getStatus).Methods("GET")
  router.HandleFunc("/get-token", getToken).Methods("GET")
  router.HandleFunc("/employees", getEmployees).Methods("GET")
  err := http.ListenAndServe(CONN_HOST+":"+CONN_PORT,
  handlers.LoggingHandler(os.Stdout, router))
  if err != nil 
  {
    log.Fatal("error starting http server : ", err)
    return
  }
}
```

1.  使用以下命令运行程序：

```go
$ go run http-rest-api.go
```

# 工作原理…

一旦我们运行程序，HTTP 服务器将在本地监听端口`8080`上启动。

接下来，您可以从命令行执行`GET`请求，如下所示：

```go
$ curl -X GET http://localhost:8080/status
 API is up and running
```

这将给您 REST API 的状态。您可以从命令行执行`GET`请求，如下所示：

```go
$ curl -X GET http://localhost:8080/employees
 [{"id":1,"firstName":"Foo","lastName":"Bar"},{"id":2,"firstName":"Baz","lastName":"Qux"}]
```

这将给你一个所有员工的列表。我们可以尝试通过命令行获取访问令牌：

```go
$ curl -X GET http://localhost:8080/get-token
```

我们将从服务器获取“Not Implemented”消息。

让我们了解我们编写的程序：

+   `import ("encoding/json" "log" "net/http" "os" “github.com/gorilla/handlers" "github.com/gorilla/mux")`：在这里，我们导入了`github.com/gorilla/mux`来创建一个 Gorilla Mux 路由器，以及`github.com/gorilla/handlers`来创建一个 Gorilla 日志处理程序，以 Apache Common Log Format 记录 HTTP 请求。

+   `func getStatus(w http.ResponseWriter, r *http.Request) { w.Write([]byte("API is up and running"))}`：这是一个处理程序，它只是向 HTTP 响应流写入 API 正在运行。

+   `func getEmployees(w http.ResponseWriter, r *http.Request) { json.NewEncoder(w).Encode(employees)}`：这是一个处理程序，它将一个静态员工数组写入 HTTP 响应流。

+   这是一个处理程序，它只是向 HTTP 响应流写入“Not Implemented”。

+   然后，我们定义了`main()`，在其中我们使用`NewRouter()`处理程序创建了一个`gorilla/mux`路由器实例，对新路由的尾随斜杠行为设置为`true`，添加路由并向其注册处理程序，最后调用`http.ListenAndServe`来处理每个传入连接的 HTTP 请求，每个连接在单独的 Goroutine 中处理。`ListenAndServe`接受两个参数——服务器地址和处理程序。在这里，我们将服务器地址传递为`localhost:8080`，处理程序为 Gorilla `LoggingHandler`，它以 Apache Common Log Format 记录 HTTP 请求。

# 创建 JSON Web 令牌

要保护您的 REST API 或服务端点，您必须编写一个在 Go 中生成 JSON Web 令牌或`JWT`的处理程序。

在这个示例中，我们将使用`https://github.com/dgrijalva/jwt-go`来生成`JWT`，尽管您可以在 Go 中实现许多第三方库中提供的任何库，例如`https://github.com/square/go-jose`和`https://github.com/tarent/loginsrv`。

# 如何做…

1.  使用`go get`命令安装`github.com/dgrijalva/jwt-go`、`github.com/gorilla/mux`和`github.com/gorilla/handlers`包，如下所示：

```go
$ go get github.com/dgrijalva/jwt-go
$ go get github.com/gorilla/handlers
$ go get github.com/gorilla/mux
```

1.  创建`create-jwt.go`，在其中我们将定义`getToken`处理程序来生成`JWT`，如下所示：

```go
package main
import 
(
  "encoding/json"
  "log"
  "net/http"
  "os"
  "time"
  jwt "github.com/dgrijalva/jwt-go"
  "github.com/gorilla/handlers"
  "github.com/gorilla/mux"
)
const 
(
  CONN_HOST = "localhost"
  CONN_PORT = "8080"
  CLAIM_ISSUER = "Packt"
  CLAIM_EXPIRY_IN_HOURS = 24
)
type Employee struct 
{
  Id int `json:"id"`
  FirstName string `json:"firstName"`
  LastName string `json:"lastName"`
}
type Employees []Employee
var employees []Employee
func init() 
{
  employees = Employees
  {
    Employee{Id: 1, FirstName: "Foo", LastName: "Bar"},
    Employee{Id: 2, FirstName: "Baz", LastName: "Qux"},
  }
}
var signature = []byte("secret")
func getToken(w http.ResponseWriter, r *http.Request) 
{
  claims := &jwt.StandardClaims
  {
    ExpiresAt: time.Now().Add(time.Hour *
    CLAIM_EXPIRY_IN_HOURS).Unix(),
    Issuer: CLAIM_ISSUER,
  }
  token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
  tokenString, _ := token.SignedString(signature)
  w.Write([]byte(tokenString))
}
func getStatus(w http.ResponseWriter, r *http.Request) 
{
  w.Write([]byte("API is up and running"))
}
func getEmployees(w http.ResponseWriter, r *http.Request) 
{
  json.NewEncoder(w).Encode(employees)
}
func main() 
{
  muxRouter := mux.NewRouter().StrictSlash(true)
  muxRouter.HandleFunc("/status", getStatus).Methods("GET")
  muxRouter.HandleFunc("/get-token", getToken).Methods("GET")
  muxRouter.HandleFunc("/employees", getEmployees).Methods("GET")
  err := http.ListenAndServe(CONN_HOST+":"+CONN_PORT,
  handlers.LoggingHandler(os.Stdout, muxRouter))
  if err != nil 
  {
    log.Fatal("error starting http server : ", err)
    return
  }
}
```

1.  使用以下命令运行程序：

```go
$ go run create-jwt.go
```

# 它是如何工作的…

一旦我们运行程序，HTTP 服务器将在本地监听端口`8080`。

接下来，我们从命令行执行一个`GET`请求：

```go
$ curl -X GET http://localhost:8080/status
 API is up and running
```

它将给你 API 的状态。接下来，我们从命令行执行一个`GET`请求：

```go
$ curl -X GET http://localhost:8080/employees
 [{"id":1,"firstName":"Foo","lastName":"Bar"},{"id":2,"firstName":"Baz","lastName":"Qux"}]
```

它将给你一个所有员工的列表。接下来，让我们尝试通过命令行获取 REST API 的访问令牌：

```go
$ curl -X GET http://localhost:8080/get-token
```

它将给我们生成的 JWT 令牌：

```go
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MTM1MDY4ODEsImlzcyI6IlBhY2t0In0.95vuiR7lpWt4AIBDasBzOffL_Xv78_J9rcrKkeqSW08
```

接下来，浏览到`https://jwt.io/`，并将生成的令牌粘贴到 Encoded 部分，以查看其解码值，如下面的屏幕截图所示：

![](img/f3d84091-ec25-4d4a-a277-8a034edb5fd8.png)

让我们了解我们在这个示例中引入的更改：

+   `import ( "encoding/json" "log" "net/http" "os" "time" jwt "github.com/dgrijalva/jwt-go" "github.com/gorilla/handlers" "github.com/gorilla/mux")`：在这里，我们导入了一个额外的包——`github.com/dgrijalva/jwt-go`，它具有 JWT 的 Go 实现。

+   `const ( CONN_HOST = "localhost" CONN_PORT = "8080" CLAIM_ISSUER = "Packt" CLAIM_EXPIRY_IN_HOURS = 24 )`：在这里，我们引入了两个额外的常量——一个是`CLAIM_ISSUER`，用于标识发出 JWT 的主体，另一个是`CLAIM_EXPIRY_IN_HOURS`，用于标识 JWT 必须在到期时间之后多长时间内不被接受进行处理。

+   `var signature = []byte("secret")`：这是服务器保存的签名。使用这个签名，服务器将能够验证现有令牌并签发新令牌。

接下来，我们定义了一个`getToken`处理程序，在其中我们首先使用`JWT StandardClaims`处理程序准备了一个声明对象，然后使用`jwt NewWithClaims`处理程序生成了一个 JWT 令牌，并最终使用服务器签名对其进行签名，并将其写入 HTTP 响应流。

# 使用 JSON Web Token 保护 RESTful 服务

一旦我们有了 REST API 端点和 JWT 令牌生成处理程序，我们就可以轻松地使用 JWT 保护我们的端点，我们将在本教程中介绍。

# 如何做…

1.  使用`go get`命令安装`github.com/auth0/go-jwt-middleware`、`github.com/dgrijalva/jwt-go`、`github.com/gorilla/mux`和`github.com/gorilla/handlers`包，如下所示：

```go
$ go get github.com/auth0/go-jwt-middleware
$ go get github.com/dgrijalva/jwt-go
$ go get github.com/gorilla/handlers
$ go get github.com/gorilla/mux
```

1.  创建`http-rest-api-secured.go`，在其中我们将定义 JWT 中间件以检查 HTTP 请求中的 JWT，并将`/employees`路由包装在其中，如下所示：

```go
package main
import 
(
  "encoding/json"
  "log"
  "net/http"
  "os"
  "time"
  jwtmiddleware "github.com/auth0/go-jwt-middleware"
  jwt "github.com/dgrijalva/jwt-go"
  "github.com/gorilla/handlers"
  "github.com/gorilla/mux"
)
const 
(
  CONN_HOST = "localhost"
  CONN_PORT = "8080"
  CLAIM_ISSUER = "Packt"
  CLAIM_EXPIRY_IN_HOURS = 24
)
type Employee struct 
{
  Id int `json:"id"`
  FirstName string `json:"firstName"`
  LastName string `json:"lastName"`
}
type Employees []Employee
var employees []Employee
func init() 
{
  employees = Employees
  {
    Employee{Id: 1, FirstName: "Foo", LastName: "Bar"},
    Employee{Id: 2, FirstName: "Baz", LastName: "Qux"},
  }
}
var signature = []byte("secret")
var jwtMiddleware = jwtmiddleware.New
(
  jwtmiddleware.Options
  {
    ValidationKeyGetter: func(token *jwt.Token) (interface{}, error) 
    {
      return signature, nil
    },
    SigningMethod: jwt.SigningMethodHS256,
  }
)
func getToken(w http.ResponseWriter, r *http.Request) 
{
  claims := &jwt.StandardClaims
  {
    ExpiresAt: time.Now().Add(time.Hour *
    CLAIM_EXPIRY_IN_HOURS).Unix(),
    Issuer: CLAIM_ISSUER,
  }
  token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
  tokenString, _ := token.SignedString(signature)
  w.Write([]byte(tokenString))
}
func getStatus(w http.ResponseWriter, r *http.Request) 
{
  w.Write([]byte("API is up and running"))
}
func getEmployees(w http.ResponseWriter, r *http.Request) 
{
  json.NewEncoder(w).Encode(employees)
}
func main() 
{
  muxRouter := mux.NewRouter().StrictSlash(true)
  muxRouter.HandleFunc("/status", getStatus).Methods("GET")
  muxRouter.HandleFunc("/get-token", getToken).Methods("GET")
  muxRouter.Handle("/employees", jwtMiddleware.Handler
  (http.HandlerFunc(getEmployees))).Methods("GET")
  err := http.ListenAndServe(CONN_HOST+":"+CONN_PORT,
  handlers.LoggingHandler(os.Stdout, muxRouter))
  if err != nil 
  {
    log.Fatal("error starting http server : ", err)
    return
  }
}
```

1.  使用以下命令运行程序：

```go
$ go run http-rest-api-secured.go
```

# 它是如何工作的…

一旦我们运行程序，HTTP 服务器将在本地监听端口`8080`。

接下来，我们从命令行执行`GET`请求，如下所示：

```go
$ curl -X GET http://localhost:8080/status
 API is up and running
```

它将向我们显示 API 的状态。接下来，我们从命令行执行`GET`请求，如下所示：

```go
$ curl -X GET http://localhost:8080/employees
 Required authorization token not found
```

它将向我们显示 JWT 未在请求中找到的消息。因此，要获取所有员工的列表，我们必须获取 API 的访问令牌，可以通过执行以下命令获取：

```go
$ curl -X GET http://localhost:8080/get-token

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MTM1MTI2NTksImlzcyI6IlBhY2t0In0.2r_q_82erdOmt862ofluiMGr3O5x5_c0_sMyW7Pi5XE
```

现在，再次调用员工 API，将 JWT 作为 HTTP`Authorization`请求头传递，如下所示：

```go
$ curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1MTM1MTI2NTksImlzcyI6IlBhY2t0In0.2r_q_82erdOmt862ofluiMGr3O5x5_c0_sMyW7Pi5XE" http://localhost:8080/employees
```

它将为您提供所有员工的列表，如下所示：

```go
[{"id":1,"firstName":"Foo","lastName":"Bar"},{"id":2,"firstName":"Baz","lastName":"Qux"}]
```

让我们了解本教程中引入的更改：

1.  使用`import（"encoding/json" "log" "net/http" "os" "time" jwtmiddleware "github.com/auth0/go-jwt-middleware" jwt "github.com/dgrijalva/jwt-go" "github.com/gorilla/handlers" "github.com/gorilla/mux"）`，我们导入了一个额外的包，`github.com/auth0/go-jwt-middleware`，别名为`jwtmiddleware`，它在 HTTP 请求中检查 JWT。

1.  然后，我们构建了一个新的安全实例`jwtmiddleware`，将`SigningMethod`设置为`HS256`，并将`ValidationKeyGetter`选项设置为一个返回用于验证 JWT 的密钥的 Go 函数。在这里，服务器签名被用作验证 JWT 的密钥。

1.  最后，我们在`main()`中使用`jwtmiddleware`处理程序包装了`/employees`路由，这意味着对于每个 URL 模式为`/employees`的请求，我们在提供响应之前检查并验证 JWT。

# 在 Go Web 应用程序中防止跨站点请求伪造

从恶意网站、电子邮件、博客、即时消息或程序攻击受信任的站点，用户当前已经认证，以防止不必要的操作，这是一种常见的做法。我们经常称之为跨站点请求伪造。

在 Go 中实现跨站点请求伪造非常容易，使用 Gorilla CSRF 包，我们将在本教程中介绍。

# 如何做…

1.  使用`go get`命令安装`github.com/gorilla/csrf`和`github.com/gorilla/mux`包，如下所示：

```go
$ go get github.com/gorilla/csrf
$ go get github.com/gorilla/mux
```

1.  创建`sign-up.html`，其中包含名称和电子邮件输入文本字段，以及一个在提交 HTML 表单时调用的操作，如下所示：

```go
<html>
  <head>
    <title>Sign Up!</title>
  </head>
  <body>
    <form method="POST" action="/post" accept-charset="UTF-8">
      <input type="text" name="name">
      <input type="text" name="email">
      {{ .csrfField }}
      <input type="submit" value="Sign up!">
    </form>
  </body>
</html>
```

1.  创建`prevent-csrf.go`，在其中创建一个`signUp`处理程序，用于呈现注册 HTML 表单，以及一个`post`处理程序，每当提交 HTML 表单并且请求具有有效的 CSRF 令牌时执行，如下所示：

```go
package main
import 
(
  "fmt"
  "html/template"
  "log"
  "net/http"
  "github.com/gorilla/csrf"
  "github.com/gorilla/mux"
)
const 
(
  CONN_HOST = "localhost"
  CONN_PORT = "8443"
  HTTPS_CERTIFICATE = "domain.crt"
  DOMAIN_PRIVATE_KEY = "domain.key"
)
var AUTH_KEY = []byte("authentication-key")
func signUp(w http.ResponseWriter, r *http.Request) 
{
  parsedTemplate, _ := template.ParseFiles("sign-up.html")
  err := parsedTemplate.Execute
  (
    w, map[string]interface{}
    {
      csrf.TemplateTag: csrf.TemplateField(r),
    }
  )
  if err != nil 
  {
    log.Printf("Error occurred while executing the 
    template : ", err)
    return
  }
}
func post(w http.ResponseWriter, r *http.Request) 
{
  err := r.ParseForm()
  if err != nil 
  {
    log.Print("error occurred while parsing form ", err)
  }
  name := r.FormValue("name")
  fmt.Fprintf(w, "Hi %s", name)
}
func main() 
{
  muxRouter := mux.NewRouter().StrictSlash(true)
  muxRouter.HandleFunc("/signup", signUp)
  muxRouter.HandleFunc("/post", post)
  http.ListenAndServeTLS(CONN_HOST+":"+CONN_PORT, 
  HTTPS_CERTIFICATE, DOMAIN_PRIVATE_KEY, csrf.Protect
  (AUTH_KEY)(muxRouter))
}
```

1.  使用以下命令运行程序：

```go
$ go run prevent-csrf.go
```

# 它是如何工作的…

一旦我们运行程序，HTTP 服务器将在本地监听端口`8443`。

接下来，从命令行执行`POST`请求，如下所示：

```go
$ curl -X POST --data "name=Foo&email=aggarwalarpit.89@gmail.com" https://localhost:8443/post --insecure
```

它将向您显示`Forbidden - CSRF token invalid`消息作为服务器的响应，并禁止您提交 HTML 表单，因为服务器在请求中找不到有效的 CSRF 令牌：

![](img/9154e672-74b9-424e-8e39-62b5d6d5a274.png)

因此，要提交表单，首先我们必须注册，通过执行以下命令生成有效的 CSRF 令牌：

```go
$ curl -i -X GET https://localhost:8443/signup --insecure
```

这将给你一个 HTTP `X-CSRF-Token`，如下面的截图所示：

![](img/6a93d469-bc90-479c-90f4-948a2e81f13f.png)

现在，您必须将其作为 HTTP `X-CSRF-Token`请求头和 HTTP cookie 一起传递，以提交 HTML 表单，如下所示：

```go
$ curl -X POST --data "name=Foo&email=aggarwalarpit.89@gmail.com" -H "X-CSRF-Token: M9gqV7rRcXERvSJVRSYprcMzwtFmjEHKXRm6C8cDC4EjTLIt4OiNzVrHfYNB12nEx280rrKs8fqOgvfcJgQiFA==" --cookie "_gorilla_csrf=MTUyMzQzMjg0OXxJa1ZLVTFsbGJHODFMMHg0VEdWc0wxZENVRVpCWVZGU1l6bHVMMVZKVEVGM01EVjBUakVyUlVoTFdsVTlJZ289fJI5dumuyObaHVp97GN_CiZBCCpnbO0wlIwgSgvHL7-C;" https://localhost:8443/post --insecure

Hi Foo
```

让我们了解一下我们编写的程序：

+   `const (CONN_HOST = "localhost" CONN_PORT = "8443" HTTPS_CERTIFICATE = "domain.crt" DOMAIN_PRIVATE_KEY = "domain.key")`：在这里，我们声明了四个常量 - `CONN_HOST`的值为`localhost`，`CONN_PORT`的值为`8443`，`HTTPS_CERTIFICATE`的值为`domain.crt`或自签名证书，以及`DOMAIN_PRIVATE_KEY`的值为`domain.key`或我们在上一个示例中创建的私钥。

+   `var AUTH_KEY = []byte("authentication-key")`：这是用于生成 CSRF 令牌的身份验证密钥。

+   `signUp`：这是一个处理程序，解析`sign-up.html`并在表单中用 CSRF 令牌替换`{{ .csrfField }}`提供一个`<input>`字段。

+   `post`：这是一个处理程序，解析提交的表单，获取名称输入字段的值，并将其写入 HTTP 响应流。

最后，我们定义了`main()`，在这里我们使用`NewRouter()`处理程序创建了一个`gorilla/mux`路由器实例，对于新路由的尾随斜杠行为设置为`true`，注册了`/signup`路由与`signUp`处理程序以及`/post`路由与`post`处理程序，并调用了`http.ListenAndServeTLS`，将处理程序传递为`csrf.Protect(AUTH_KEY)(muxRouter)`，这样可以确保所有没有有效令牌的`POST`请求都会返回`HTTP 403 Forbidden`。
