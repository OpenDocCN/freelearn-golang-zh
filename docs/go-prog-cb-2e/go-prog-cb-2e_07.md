# 第七章：Web 客户端和 API

使用 API 并编写 Web 客户端可能是一件棘手的事情。不同的 API 具有不同类型的授权、认证和协议。我们将探索`http.Client`结构对象，使用 OAuth2 客户端和长期令牌存储，并最后使用 GRPC 和额外的 REST 接口。

在本章结束时，您应该知道如何与第三方或内部 API 进行交互，并且对于常见操作（如对 API 的异步请求）有一些模式。

在本章中，我们将涵盖以下的步骤：

+   初始化、存储和传递 http.Client 结构

+   为 REST API 编写客户端

+   执行并行和异步客户端请求

+   使用 OAuth2 客户端

+   实现 OAuth2 令牌存储接口

+   在添加功能和函数组合中包装客户端

+   理解 GRPC 客户端

+   使用 twitchtv/twirp 进行 RPC

# 技术要求

为了继续本章中的所有示例，根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.12.6 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  打开终端或控制台应用程序，创建一个项目目录，例如`~/projects/go-programming-cookbook`，并导航到该目录。所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，并选择从该目录工作，而不是手动输入示例，如下所示：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 初始化、存储和传递 http.Client 结构

Go 的`net/http`包为处理 HTTP API 公开了一个灵活的`http.Client`结构。这个结构具有单独的传输功能，使得对请求进行短路、修改每个客户端操作的标头以及处理任何 REST 操作相对简单。创建客户端是一个非常常见的操作，这个示例将从工作和创建一个`http.Client`对象的基础知识开始。

# 如何做...

这些步骤涵盖了编写和运行应用程序的步骤：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/client`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/client 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/client 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/client`复制测试，或者自己编写一些代码来练习！

1.  创建一个名为`client.go`的文件，内容如下：

```go
        package client

        import (
            "crypto/tls"
            "net/http"
        )

        // Setup configures our client and redefines
        // the global DefaultClient
        func Setup(isSecure, nop bool) *http.Client {
            c := http.DefaultClient

            // Sometimes for testing, we want to
            // turn off SSL verification
            if !isSecure {
                c.Transport = &http.Transport{
                TLSClientConfig: &tls.Config{
                    InsecureSkipVerify: false,
                },
            }
        }
        if nop {
            c.Transport = &NopTransport{}
        }
        http.DefaultClient = c
        return c
        }

        // NopTransport is a No-Op Transport
        type NopTransport struct {
        }

        // RoundTrip Implements RoundTripper interface
        func (n *NopTransport) RoundTrip(*http.Request) 
        (*http.Response, error) {
            // note this is an unitialized Response
            // if you're looking at headers etc
            return &http.Response{StatusCode: http.StatusTeapot}, nil
        }
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package client

        import (
            "fmt"
            "net/http"
        )

        // DoOps takes a client, then fetches
        // google.com
        func DoOps(c *http.Client) error {
            resp, err := c.Get("http://www.google.com")
            if err != nil {
                return err
            }
            fmt.Println("results of DoOps:", resp.StatusCode)

            return nil
        }

        // DefaultGetGolang uses the default client
        // to get golang.org
        func DefaultGetGolang() error {
            resp, err := http.Get("https://www.golang.org")
            if err != nil {
                return err
            }
            fmt.Println("results of DefaultGetGolang:", 
            resp.StatusCode)
            return nil
        }
```

1.  创建一个名为`store.go`的文件，内容如下：

```go
        package client

        import (
            "fmt"
            "net/http"
        )

        // Controller embeds an http.Client
        // and uses it internally
        type Controller struct {
            *http.Client
        }

        // DoOps with a controller object
        func (c *Controller) DoOps() error {
            resp, err := c.Client.Get("http://www.google.com")
            if err != nil {
                return err
            }
            fmt.Println("results of client.DoOps", resp.StatusCode)
            return nil
        }
```

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter7/client"

        func main() {
            // secure and op!
            cli := client.Setup(true, false)

            if err := client.DefaultGetGolang(); err != nil {
                panic(err)
            }

            if err := client.DoOps(cli); err != nil {
                panic(err)
            }

            c := client.Controller{Client: cli}
            if err := c.DoOps(); err != nil {
                panic(err)
            }

            // secure and noop
            // also modifies default
            client.Setup(true, true)

            if err := client.DefaultGetGolang(); err != nil {
                panic(err)
            }
        }
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

现在您应该看到以下输出：

```go
$ go run main.go
results of DefaultGetGolang: 200
results of DoOps: 200
results of client.DoOps 200
results of DefaultGetGolang: 418
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`net/http`包公开了一个`DefaultClient`包变量，该变量被以下内部操作使用：`Do`、`GET`、`POST`等。我们的`Setup()`函数返回一个客户端，并将默认客户端设置为相同的客户端。在设置客户端时，大部分修改将发生在传输中，传输只需要实现`RoundTripper`接口。

这个示例提供了一个总是返回 418 状态码的无操作往返器的示例。您可以想象这对于测试可能有多么有用。它还演示了将客户端作为函数参数传递，将它们用作结构参数，并使用默认客户端来处理请求。

# 为 REST API 编写客户端

为 REST API 编写客户端不仅有助于更好地理解相关的 API，还将为所有将来使用该 API 的应用程序提供一个有用的工具。这个配方将探讨构建客户端的结构，并展示一些您可以立即利用的策略。

对于这个客户端，我们将假设认证是由基本认证处理的，但也应该可以命中一个端点来检索令牌等。为了简单起见，我们假设我们的 API 公开了一个端点`GetGoogle()`，它返回从[`www.google.com`](https://www.google.com)进行`GET`请求返回的状态码。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/rest`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/rest 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/rest 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/rest`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`client.go`的文件，内容如下：

```go
        package rest

        import "net/http"

        // APIClient is our custom client
        type APIClient struct {
            *http.Client
        }

        // NewAPIClient constructor initializes the client with our
        // custom Transport
        func NewAPIClient(username, password string) *APIClient {
            t := http.Transport{}
            return &APIClient{
                Client: &http.Client{
                    Transport: &APITransport{
                        Transport: &t,
                        username: username,
                        password: password,
                    },
                },
            }
        }

        // GetGoogle is an API Call - we abstract away
        // the REST aspects
        func (c *APIClient) GetGoogle() (int, error) {
            resp, err := c.Get("http://www.google.com")
            if err != nil {
                return 0, err
            }
            return resp.StatusCode, nil
        }
```

1.  创建一个名为`transport.go`的文件，内容如下：

```go
        package rest

        import "net/http"

        // APITransport does a SetBasicAuth
        // for every request
        type APITransport struct {
            *http.Transport
            username, password string
        }

        // RoundTrip does the basic auth before deferring to the
        // default transport
        func (t *APITransport) RoundTrip(req *http.Request) 
        (*http.Response, error) {
            req.SetBasicAuth(t.username, t.password)
            return t.Transport.RoundTrip(req)
        }
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package rest

        import "fmt"

        // Exec creates an API Client and uses its
        // GetGoogle method, then prints the result
        func Exec() error {
            c := NewAPIClient("username", "password")

            StatusCode, err := c.GetGoogle()
            if err != nil {
                return err
            }
            fmt.Println("Result of GetGoogle:", StatusCode)
            return nil
        }
```

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter7/rest"

        func main() {
            if err := rest.Exec(); err != nil {
                panic(err)
            }
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
Result of GetGoogle: 200
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这段代码演示了如何隐藏诸如认证和使用`Transport`接口执行令牌刷新等逻辑。它还演示了如何通过方法公开 API 调用。如果我们正在针对诸如用户 API 之类的东西进行实现，我们期望有以下方法：

```go
type API interface{
  GetUsers() (Users, error)
  CreateUser(User) error
  UpdateUser(User) error
  DeleteUser(User)
}
```

如果您阅读了第五章*关于数据库和存储的所有内容*，这可能看起来与名为*执行数据库事务接口*的配方相似。通过接口进行组合，特别是像`RoundTripper`接口这样的常见接口，为编写 API 提供了很大的灵活性。此外，编写一个顶层接口并传递接口而不是直接传递给客户端可能是有用的。在下一个配方中，我们将更详细地探讨这一点，因为我们将探讨编写 OAuth2 客户端。

# 执行并行和异步客户端请求

在 Go 中并行执行客户端请求相对简单。在下一个配方中，我们将使用客户端使用 Go 缓冲通道检索多个 URL。响应和错误都将发送到一个单独的通道，任何有权访问客户端的人都可以立即访问。

在这个配方的情况下，创建客户端，读取通道，处理响应和错误都将在`main.go`文件中完成。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/async`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/async 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/async 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/async`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`config.go`的文件，内容如下：

```go
        package async

        import "net/http"

        // NewClient creates a new client and 
        // sets its appropriate channels
        func NewClient(client *http.Client, bufferSize int) *Client {
            respch := make(chan *http.Response, bufferSize)
            errch := make(chan error, bufferSize)
            return &Client{
                Client: client,
                Resp: respch,
                Err: errch,
            }
        }

        // Client stores a client and has two channels to aggregate
        // responses and errors
        type Client struct {
            *http.Client
            Resp chan *http.Response
            Err chan error
        }

        // AsyncGet performs a Get then returns
        // the resp/error to the appropriate channel
        func (c *Client) AsyncGet(url string) {
            resp, err := c.Get(url)
            if err != nil {
                c.Err <- err
                return
            }
            c.Resp <- resp
        }
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package async

        // FetchAll grabs a list of urls
        func FetchAll(urls []string, c *Client) {
            for _, url := range urls {
                go c.AsyncGet(url)
            }
        }
```

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/chapter7/async"
        )

        func main() {
            urls := []string{
                "https://www.google.com",
                "https://golang.org",
                "https://www.github.com",
            }
            c := async.NewClient(http.DefaultClient, len(urls))
            async.FetchAll(urls, c)

            for i := 0; i < len(urls); i++ {
                select {
                    case resp := <-c.Resp:
                    fmt.Printf("Status received for %s: %d\n", 
                    resp.Request.URL, resp.StatusCode)
                    case err := <-c.Err:
                   fmt.Printf("Error received: %s\n", err)
                }
            }
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
Status received for https://www.google.com: 200
Status received for https://golang.org: 200
Status received for https://github.com/: 200
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

本配方创建了一个处理请求的框架，以一种`async`方式使用单个客户端。它将尝试尽快检索您指定的尽可能多的 URL。在许多情况下，您可能希望进一步限制这一点，例如使用工作池。在客户端之外处理这些`async` Go 例程并为特定的存储或检索接口处理这些也是有意义的。

本配方还探讨了使用 case 语句在多个通道上进行切换。由于获取是异步执行的，必须有一些机制等待它们完成。在这种情况下，只有当主函数读取与原始列表中的 URL 数量相同的响应和错误时，程序才会终止。在这种情况下，还需要考虑应用程序是否应该超时，或者是否有其他方法可以提前取消其操作。

# 利用 OAuth2 客户端

OAuth2 是一种与 API 通信的相对常见的协议。`golang.org/x/oauth2`包提供了一个非常灵活的客户端，用于处理 OAuth2。它有子包指定各种提供程序的端点，如 Facebook、Google 和 GitHub。

本配方将演示如何创建一个新的 GitHub OAuth2 客户端以及一些基本用法。

# 准备工作

完成本章开头“技术要求”部分提到的初始设置步骤后，继续以下步骤：

1.  在[`github.com/settings/applications/new`](https://github.com/settings/applications/new)上配置 OAuth 客户端。

1.  使用您的客户端 ID 和密钥设置环境变量：

+   `export GITHUB_CLIENT="your_client"`

+   `export GITHUB_SECRET="your_secret"`

1.  在[`developer.github.com/v3/`](https://developer.github.com/v3/)上查看 GitHub API 文档。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/oauthcli`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/oauthcli 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/oauthcli 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/oauthcli`复制测试，或者将其作为练习编写自己的代码！

1.  创建一个名为`config.go`的文件，内容如下：

```go
        package oauthcli

        import (
            "context"
            "fmt"
            "os"

            "golang.org/x/oauth2"
            "golang.org/x/oauth2/github"
        )

        // Setup return an oauth2Config configured to talk
        // to github, you need environment variables set
        // for your id and secret
        func Setup() *oauth2.Config {
            return &oauth2.Config{
                ClientID: os.Getenv("GITHUB_CLIENT"),
                ClientSecret: os.Getenv("GITHUB_SECRET"),
                Scopes: []string{"repo", "user"},
                Endpoint: github.Endpoint,
            }
        }

        // GetToken retrieves a github oauth2 token
        func GetToken(ctx context.Context, conf *oauth2.Config) 
        (*oauth2.Token, error) {
            url := conf.AuthCodeURL("state")
            fmt.Printf("Type the following url into your browser and 
            follow the directions on screen: %v\n", url)
            fmt.Println("Paste the code returned in the redirect URL 
            and hit Enter:")

            var code string
            if _, err := fmt.Scan(&code); err != nil {
                return nil, err
            }
            return conf.Exchange(ctx, code)
        }
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package oauthcli

        import (
            "fmt"
            "net/http"
        )

        // GetUsers uses an initialized oauth2 client to get
        // information about a user
        func GetUser(client *http.Client) error {
            url := fmt.Sprintf("https://api.github.com/user")

            resp, err := client.Get(url)
            if err != nil {
                return err
            }
            defer resp.Body.Close()
            fmt.Println("Status Code from", url, ":", resp.StatusCode)
            io.Copy(os.Stdout, resp.Body)
            return nil
        }
```

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个`main.go`文件，内容如下：

```go
        package main

        import (
            "context"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter7/oauthcli"
        )

        func main() {
            ctx := context.Background()
            conf := oauthcli.Setup()

            tok, err := oauthcli.GetToken(ctx, conf)
            if err != nil {
                panic(err)
            }
            client := conf.Client(ctx, tok)

            if err := oauthcli.GetUser(client); err != nil {
                panic(err)
            }

        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

现在应该看到以下输出：

```go
$ go run main.go
Visit the URL for the auth dialog: 
https://github.com/login/oauth/authorize?
access_type=offline&client_id=
<your_id>&response_type=code&scope=repo+user&state=state
Paste the code returned in the redirect URL and hit Enter:
<your_code>
Status Code from https://api.github.com/user: 200
{<json_payload>}
```

1.  `go.mod`文件可能会更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

标准的 OAuth2 流程是基于重定向的，并以服务器重定向到您指定的端点结束。然后您的服务器负责抓取代码并将其交换为令牌。本配方通过允许我们使用诸如`https://localhost`或`https://a-domain-you-own`之类的 URL 绕过了这一要求，手动复制/粘贴代码，然后按*Enter*。令牌交换后，客户端将根据需要智能地刷新令牌。

重要的是要注意，我们没有以任何方式存储令牌。如果程序崩溃，必须重新交换令牌。还需要注意的是，除非刷新令牌过期、丢失或损坏，否则只需要显式检索一次令牌。一旦客户端配置完成，只要在 OAuth2 流程期间请求了适当的范围，它就应该能够执行所有典型的 HTTP 操作。本配方请求了`"repo"`和`"user"`范围，但可以根据需要添加更多或更少。

# 实现 OAuth2 令牌存储接口

在上一个配方中，我们为客户端检索了一个令牌并执行了 API 请求。这种方法的缺点是我们没有长期存储令牌。例如，在 HTTP 服务器中，我们希望在请求之间对令牌进行一致的存储。

这个配方将探讨修改 OAuth2 客户端以在请求之间存储令牌，并使用密钥根据需要检索它。为了简单起见，这个密钥将是一个文件，但也可以是数据库、Redis 等。

# 准备工作

参考*准备工作*部分中*利用 OAuth2 客户端*配方。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/oauthstore`的新目录，并切换到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/oauthstore 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/oauthstore 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/oauthstore`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`config.go`的文件，内容如下：

```go
        package oauthstore

        import (
            "context"
            "net/http"

            "golang.org/x/oauth2"
        )

        // Config wraps the default oauth2.Config
        // and adds our storage
        type Config struct {
            *oauth2.Config
            Storage
        }

        // Exchange stores a token after retrieval
        func (c *Config) Exchange(ctx context.Context, code string)     
        (*oauth2.Token, error) {
            token, err := c.Config.Exchange(ctx, code)
            if err != nil {
                return nil, err
            }
            if err := c.Storage.SetToken(token); err != nil {
                return nil, err
            }
            return token, nil
        }

        // TokenSource can be passed a token which
        // is stored, or when a new one is retrieved,
        // that's stored
        func (c *Config) TokenSource(ctx context.Context, t 
        *oauth2.Token) oauth2.TokenSource {
            return StorageTokenSource(ctx, c, t)
        }

        // Client is attached to our TokenSource
        func (c *Config) Client(ctx context.Context, t *oauth2.Token) 
        *http.Client {
            return oauth2.NewClient(ctx, c.TokenSource(ctx, t))
        }
```

1.  创建一个名为`tokensource.go`的文件，内容如下：

```go
        package oauthstore

        import (
            "context"

            "golang.org/x/oauth2"
        )

        type storageTokenSource struct {
            *Config
            oauth2.TokenSource
        }

        // Token satisfies the TokenSource interface
        func (s *storageTokenSource) Token() (*oauth2.Token, error) {
            if token, err := s.Config.Storage.GetToken(); err == nil && 
            token.Valid() {
                return token, err
            }
            token, err := s.TokenSource.Token()
            if err != nil {
                return token, err
            }
            if err := s.Config.Storage.SetToken(token); err != nil {
                return nil, err
            }
            return token, nil
        }

        // StorageTokenSource will be used by out configs TokenSource
        // function
        func StorageTokenSource(ctx context.Context, c *Config, t 
        *oauth2.Token) oauth2.TokenSource {
            if t == nil || !t.Valid() {
                if tok, err := c.Storage.GetToken(); err == nil {
                   t = tok
                }
            }
            ts := c.Config.TokenSource(ctx, t)
            return &storageTokenSource{c, ts}
        }
```

1.  创建一个名为`storage.go`的文件，内容如下：

```go
        package oauthstore

        import (
            "context"
            "fmt"

            "golang.org/x/oauth2"
        )

        // Storage is our generic storage interface
        type Storage interface {
            GetToken() (*oauth2.Token, error)
            SetToken(*oauth2.Token) error
        }

        // GetToken retrieves a github oauth2 token
        func GetToken(ctx context.Context, conf Config) (*oauth2.Token, 
        error) {
            token, err := conf.Storage.GetToken()
            if err == nil && token.Valid() {
                return token, err
            }
            url := conf.AuthCodeURL("state")
            fmt.Printf("Type the following url into your browser and 
            follow the directions on screen: %v\n", url)
            fmt.Println("Paste the code returned in the redirect URL 
            and hit Enter:")

            var code string
            if _, err := fmt.Scan(&code); err != nil {
                return nil, err
            }
            return conf.Exchange(ctx, code)
        }
```

1.  创建一个名为`filestorage.go`的文件，内容如下：

```go
        package oauthstore

        import (
            "encoding/json"
            "errors"
            "os"
            "sync"

            "golang.org/x/oauth2"
        )

        // FileStorage satisfies our storage interface
        type FileStorage struct {
            Path string
            mu sync.RWMutex
        }

        // GetToken retrieves a token from a file
        func (f *FileStorage) GetToken() (*oauth2.Token, error) {
            f.mu.RLock()
            defer f.mu.RUnlock()
            in, err := os.Open(f.Path)
            if err != nil {
                return nil, err
            }
            defer in.Close()
            var t *oauth2.Token
            data := json.NewDecoder(in)
            return t, data.Decode(&t)
        }

        // SetToken creates, truncates, then stores a token
        // in a file
        func (f *FileStorage) SetToken(t *oauth2.Token) error {
            if t == nil || !t.Valid() {
                return errors.New("bad token")
            }

            f.mu.Lock()
            defer f.mu.Unlock()
            out, err := os.OpenFile(f.Path, 
            os.O_RDWR|os.O_CREATE|os.O_TRUNC, 0755)
            if err != nil {
                return err
            }
            defer out.Close()
            data, err := json.Marshal(&t)
            if err != nil {
                return err
            }

            _, err = out.Write(data)
            return err
        }
```

1.  创建一个名为`example`的新目录并切换到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "context"
            "io"
            "os"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter7/oauthstore"

            "golang.org/x/oauth2"
            "golang.org/x/oauth2/github"
        )

        func main() {
            conf := oauthstore.Config{
                Config: &oauth2.Config{
                    ClientID: os.Getenv("GITHUB_CLIENT"),
                    ClientSecret: os.Getenv("GITHUB_SECRET"),
                    Scopes: []string{"repo", "user"},
                    Endpoint: github.Endpoint,
                },
                Storage: &oauthstore.FileStorage{Path: "token.txt"},
            }
            ctx := context.Background()
            token, err := oauthstore.GetToken(ctx, conf)
            if err != nil {
                panic(err)
            }

            cli := conf.Client(ctx, token)
            resp, err := cli.Get("https://api.github.com/user")
            if err != nil {
                panic(err)
            }
            defer resp.Body.Close()
            io.Copy(os.Stdout, resp.Body)
        }
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您现在应该看到以下输出：

```go
$ go run main.go
Visit the URL for the auth dialog: 
https://github.com/login/oauth/authorize?
access_type=offline&client_id=
<your_id>&response_type=code&scope=repo+user&state=state
Paste the code returned in the redirect URL and hit Enter:
<your_code>
{<json_payload>}

$ go run main.go
{<json_payload>}
```

1.  `go.mod`文件可能已更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方负责将令牌的内容存储和检索到文件中。如果是第一次运行，它必须执行整个代码交换，但后续运行将重用访问令牌，并且如果有一个可用，它将使用刷新令牌进行刷新。

目前在这段代码中没有办法区分用户/令牌，但可以通过 cookie 作为文件名的密钥或数据库中的一行来实现。让我们来看看这段代码的功能：

+   `config.go`文件包装了标准的 OAuth2 配置。对于涉及检索令牌的每个方法，我们首先检查本地存储中是否有有效的令牌。如果没有，我们使用标准配置检索一个，然后存储它。

+   `tokensource.go`文件实现了我们自定义的`TokenSource`接口，与`Config`配对。与`Config`类似，我们总是首先尝试从文件中检索我们的令牌；如果失败，我们将使用新令牌设置它。

+   `storage.go`文件是`Config`和`TokenSource`使用的`storage`接口。它只定义了两种方法，我们还包括了一个辅助函数来启动 OAuth2 基于代码的流程，类似于我们在上一个配方中所做的，但如果已经存在一个有效令牌的文件，它将被使用。

+   `filestorage.go`文件实现了`storage`接口。当我们存储一个新令牌时，我们首先截断文件并写入`token`结构的 JSON 表示。否则，我们解码文件并返回`token`。

# 在客户端中添加功能和函数组合

2015 年，Tomás Senart 就如何使用接口包装`http.Client`结构并利用中间件和函数组合进行了出色的演讲。您可以在[`github.com/gophercon/2015-talks`](https://github.com/gophercon/2015-talks)找到更多信息。这个配方借鉴了他的想法，并演示了在`http.Client`结构的`Transport`接口上执行相同操作的示例，类似于我们之前的配方*为 REST API 编写客户端*。

以下教程将为标准的`http.Client`结构实现日志记录和基本 auth 中间件。它还包括一个`decorate`函数，可以在需要时与各种中间件一起使用。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/decorator`的新目录，并进入此目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/decorator 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/decorator 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/decorator`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`config.go`的文件，内容如下：

```go
        package decorator

        import (
            "log"
            "net/http"
            "os"
        )

        // Setup initializes our ClientInterface
        func Setup() *http.Client {
            c := http.Client{}

            t := Decorate(&http.Transport{},
                Logger(log.New(os.Stdout, "", 0)),
                BasicAuth("username", "password"),
            )
            c.Transport = t
            return &c
        }
```

1.  创建一个名为`decorator.go`的文件，内容如下：

```go
        package decorator

        import "net/http"

        // TransportFunc implements the RountTripper interface
        type TransportFunc func(*http.Request) (*http.Response, error)

        // RoundTrip just calls the original function
        func (tf TransportFunc) RoundTrip(r *http.Request) 
        (*http.Response, error) {
            return tf(r)
        }

        // Decorator is a convenience function to represent our
        // middleware inner function
        type Decorator func(http.RoundTripper) http.RoundTripper

        // Decorate is a helper to wrap all the middleware
        func Decorate(t http.RoundTripper, rts ...Decorator) 
        http.RoundTripper {
            decorated := t
            for _, rt := range rts {
                decorated = rt(decorated)
            }
            return decorated
        }
```

1.  创建一个名为`middleware.go`的文件，内容如下：

```go
        package decorator

        import (
            "log"
            "net/http"
            "time"
        )

        // Logger is one of our 'middleware' decorators
        func Logger(l *log.Logger) Decorator {
            return func(c http.RoundTripper) http.RoundTripper {
                return TransportFunc(func(r *http.Request) 
                (*http.Response, error) {
                   start := time.Now()
                   l.Printf("started request to %s at %s", r.URL,     
                   start.Format("2006-01-02 15:04:05"))
                   resp, err := c.RoundTrip(r)
                   l.Printf("completed request to %s in %s", r.URL, 
                   time.Since(start))
                   return resp, err
                })
            }
        }

        // BasicAuth is another of our 'middleware' decorators
        func BasicAuth(username, password string) Decorator {
            return func(c http.RoundTripper) http.RoundTripper {
                return TransportFunc(func(r *http.Request) 
                (*http.Response, error) {
                    r.SetBasicAuth(username, password)
                    resp, err := c.RoundTrip(r)
                    return resp, err
                })
            }
        }
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package decorator

        import "fmt"

        // Exec creates a client, calls google.com
        // then prints the response
        func Exec() error {
            c := Setup()

            resp, err := c.Get("https://www.google.com")
            if err != nil {
                return err
            }
            fmt.Println("Response code:", resp.StatusCode)
            return nil
        }
```

1.  创建一个名为`example`的新目录，并进入。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter7/decorator"

        func main() {
            if err := decorator.Exec(); err != nil {
                panic(err)
            }
        }
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您现在应该看到以下输出：

```go
$ go run main.go
started request to https://www.google.com at 2017-01-01 13:38:42
completed request to https://www.google.com in 194.013054ms
Response code: 200
```

1.  如果您复制或编写了自己的测试，请返回到上一级目录并运行`go test`。确保所有测试都通过了。

# 工作原理...

这个教程利用了闭包作为一等公民和接口。实现这一点的主要技巧是让一个函数实现一个接口。这使我们能够用一个函数实现的接口来包装一个结构体实现的接口。

`middleware.go`文件包含两个示例客户端中间件函数。这些可以扩展为包含其他中间件，比如更复杂的 auth 和 metrics。这个教程也可以与前一个教程结合起来，生成一个可以通过其他中间件扩展的 OAuth2 客户端。

`Decorator`函数是一个方便的函数，允许以下操作：

```go
Decorate(RoundTripper, Middleware1, Middleware2, etc)

vs

var t RoundTripper
t = Middleware1(t)
t = Middleware2(t)
etc
```

与包装客户端相比，这种方法的优势在于我们可以保持接口的稀疏性。如果您想要一个功能齐全的客户端，您还需要实现`GET`、`POST`和`PostForm`等方法。

# 理解 GRPC 客户端

GRPC 是一个高性能的 RPC 框架，使用协议缓冲区([`developers.google.com/protocol-buffers`](https://developers.google.com/protocol-buffers))和 HTTP/2([`http2.github.io`](https://http2.github.io))构建。在 Go 中创建一个 GRPC 客户端涉及到与 Go HTTP 客户端相同的许多复杂性。为了演示基本客户端的使用，最容易的方法是同时实现一个服务器。这个教程将创建一个`greeter`服务，它接受一个问候和一个名字，并返回句子`<greeting> <name>!`。此外，服务器可以指定是否感叹`!`或不是`.`(句号)。

这个教程不会探讨 GRPC 的一些细节，比如流式传输；但是，它有望作为创建一个非常基本的服务器和客户端的介绍。

# 准备就绪

在本章开头的*技术要求*部分完成初始设置步骤后，安装 GRPC ([`grpc.io/docs/quickstart/go/`](https://grpc.io/docs/quickstart/go/)) 并运行以下命令：

+   `go get -u github.com/golang/protobuf/{proto,protoc-gen-go}`

+   `go get -u google.golang.org/grpc`

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/grpc`的新目录，并进入此目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/grpc 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/grpc 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/grpc`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`greeter`的目录并进入。

1.  创建一个名为`greeter.proto`的文件，内容如下：

```go
        syntax = "proto3";

        package greeter;

        service GreeterService{
            rpc Greet(GreetRequest) returns (GreetResponse) {}
        }

        message GreetRequest {
            string greeting = 1;
            string name = 2;
        }

        message GreetResponse{
            string response = 1;
        }
```

1.  返回到`grpc`目录。

1.  运行以下命令：

```go
$ protoc --go_out=plugins=grpc:. greeter/greeter.proto
```

1.  创建一个名为`server`的新目录，并进入该目录。

1.  创建一个名为`greeter.go`的文件，内容如下。确保修改`greeter`导入，使用你在第 3 步设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter7/grpc/greeter"
            "golang.org/x/net/context"
        )

        // Greeter implements the interface
        // generated by protoc
        type Greeter struct {
            Exclaim bool
        }

        // Greet implements grpc Greet
        func (g *Greeter) Greet(ctx context.Context, r 
        *greeter.GreetRequest) (*greeter.GreetResponse, error) {
            msg := fmt.Sprintf("%s %s", r.GetGreeting(), r.GetName())
            if g.Exclaim {
                msg += "!"
            } else {
                msg += "."
            }
            return &greeter.GreetResponse{Response: msg}, nil
        }
```

1.  创建一个名为`server.go`的文件，内容如下。确保修改`greeter`导入，使用你在第 3 步设置的路径：

```go
        package main

        import (
            "fmt"
            "net"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter7/grpc/greeter"
            "google.golang.org/grpc"
        )

        func main() {
            grpcServer := grpc.NewServer()
            greeter.RegisterGreeterServiceServer(grpcServer, 
            &Greeter{Exclaim: true})
            lis, err := net.Listen("tcp", ":4444")
            if err != nil {
                panic(err)
            }
            fmt.Println("Listening on port :4444")
            grpcServer.Serve(lis)
        }
```

1.  返回到`grpc`目录。

1.  创建一个名为`client`的新目录，并进入该目录。

1.  创建一个名为`client.go`的文件，内容如下。确保修改`greeter`导入，使用你在第 3 步设置的路径：

```go
        package main

        import (
            "context"
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter7/grpc/greeter"
            "google.golang.org/grpc"
        )

        func main() {
            conn, err := grpc.Dial(":4444", grpc.WithInsecure())
            if err != nil {
                panic(err)
            }
            defer conn.Close()

            client := greeter.NewGreeterServiceClient(conn)

            ctx := context.Background()
            req := greeter.GreetRequest{Greeting: "Hello", Name: 
            "Reader"}
            resp, err := client.Greet(ctx, &req)
            if err != nil {
                panic(err)
            }
            fmt.Println(resp)

            req.Greeting = "Goodbye"
            resp, err = client.Greet(ctx, &req)
            if err != nil {
                panic(err)
            }
            fmt.Println(resp)
        }
```

1.  返回到`grpc`目录。

1.  运行`go run ./server`，你会看到以下输出：

```go
$ go run ./server
Listening on port :4444
```

1.  在另一个终端中，从`grpc`目录运行`go run ./client`，你会看到以下输出：

```go
$ go run ./client
response:"Hello Reader!" 
response:"Goodbye Reader!"
```

1.  `go.mod`文件可能会被更新，顶级示例目录中现在应该存在`go.sum`文件。

1.  如果你复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

GRPC 服务器设置为监听端口`4444`。一旦客户端连接，它就可以向服务器发送请求并接收响应。请求、响应和支持的方法的结构由我们在第 4 步创建的`.proto`文件所决定。在实践中，当集成到 GRPC 服务器时，它们应该提供`.proto`文件，该文件可以用于自动生成客户端。

除了客户端，`protoc`命令还会为服务器生成存根，所需的一切就是填写实现细节。生成的 Go 代码还具有 JSON 标记，相同的结构可以重用于 JSON REST 服务。我们的代码设置了一个不安全的客户端。要安全地处理 GRPC，你需要使用 SSL 证书。

# 使用 twitchtv/twirp 进行 RPC

`twitchtv/twirp` RPC 框架提供了许多 GRPC 的优点，包括使用协议缓冲区（[`developers.google.com/protocol-buffers`](https://developers.google.com/protocol-buffers)）构建模型，并允许通过 HTTP 1.1 进行通信。它还可以使用 JSON 进行通信，因此可以使用`curl`命令与`twirp` RPC 服务进行通信。这个示例将实现与之前 GRPC 部分相同的`greeter`。该服务接受一个问候和一个名字，并返回句子`<greeting> <name>!`。此外，服务器可以指定是否感叹`!`或不感叹`.`。

这个示例不会探索`twitchtv/twirp`的其他功能，主要关注基本的客户端-服务器通信。有关支持的更多信息，请访问他们的 GitHub 页面（[`github.com/twitchtv/twirp`](https://github.com/twitchtv/twirp)）。

# 准备就绪

完成本章开头*技术要求*部分提到的初始设置步骤后，安装 twirp [`twitchtv.github.io/twirp/docs/install.html`](https://twitchtv.github.io/twirp/docs/install.html)，并运行以下命令：

+   `go get -u github.com/golang/protobuf/{proto,protoc-gen-go}`

+   `go get github.com/twitchtv/twirp/protoc-gen-twirp`

# 如何操作...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter7/twirp`的新目录，并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/twirp 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/twirp 
```

1.  从`~/projects/go-programming-cookbook-original/chapter7/twirp`复制测试，或者将其作为练习编写一些自己的代码！

1.  创建一个名为`rpc/greeter`的目录，并进入该目录。

1.  创建一个名为`greeter.proto`的文件，内容如下：

```go
        syntax = "proto3";

        package greeter;

        service GreeterService{
            rpc Greet(GreetRequest) returns (GreetResponse) {}
        }

        message GreetRequest {
            string greeting = 1;
            string name = 2;
        }

        message GreetResponse{
            string response = 1;
        }
```

1.  返回到`twirp`目录。

1.  运行以下命令：

```go
$ protoc --proto_path=$GOPATH/src:. --twirp_out=. --go_out=. ./rpc/greeter/greeter.proto
```

1.  创建一个名为`server`的新目录，并进入该目录。

1.  创建一个名为`greeter.go`的文件，内容如下。确保修改`greeter`导入，使用你在第 3 步设置的路径：

```go
package main

import (
  "context"
  "fmt"

  "github.com/PacktPublishing/
   Go-Programming-Cookbook-Second-Edition/
   chapter7/twirp/rpc/greeter"
)

// Greeter implements the interface
// generated by protoc
type Greeter struct {
  Exclaim bool
}

// Greet implements twirp Greet
func (g *Greeter) Greet(ctx context.Context, r *greeter.GreetRequest) (*greeter.GreetResponse, error) {
  msg := fmt.Sprintf("%s %s", r.GetGreeting(), r.GetName())
  if g.Exclaim {
    msg += "!"
  } else {
    msg += "."
  }
  return &greeter.GreetResponse{Response: msg}, nil
}
```

1.  创建一个名为`server.go`的文件，内容如下。确保修改`greeter`导入以使用您在第 3 步设置的路径：

```go
package main

import (
  "fmt"
  "net/http"

  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/twirp/rpc/greeter"
)

func main() {
  server := &Greeter{}
  twirpHandler := greeter.NewGreeterServiceServer(server, nil)

  fmt.Println("Listening on port :4444")
  http.ListenAndServe(":4444", twirpHandler)
}
```

1.  导航回到`twirp`目录的上一级目录。

1.  创建一个名为`client`的新目录并导航到该目录。

1.  创建一个名为`client.go`的文件，内容如下。确保修改`greeter`导入以使用您在第 3 步设置的路径：

```go
package main

import (
  "context"
  "fmt"
  "net/http"

  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter7/twirp/rpc/greeter"
)

func main() {
  // you can put in a custom client for tighter controls on timeouts etc.
  client := greeter.NewGreeterServiceProtobufClient("http://localhost:4444", &http.Client{})

  ctx := context.Background()
  req := greeter.GreetRequest{Greeting: "Hello", Name: "Reader"}
  resp, err := client.Greet(ctx, &req)
  if err != nil {
    panic(err)
  }
  fmt.Println(resp)

  req.Greeting = "Goodbye"
  resp, err = client.Greet(ctx, &req)
  if err != nil {
    panic(err)
  }
  fmt.Println(resp)
}
```

1.  导航回到`twirp`目录的上一级目录。

1.  运行`go run ./server`，您将看到以下输出：

```go
$ go run ./server
Listening on port :4444
```

1.  在另一个终端中，从`twirp`目录运行`go run ./client`。您应该会看到以下输出：

```go
$ go run ./client
response:"Hello Reader." 
response:"Goodbye Reader."
```

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶层配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

我们设置了`twitchtv/twirp` RPC 服务器监听端口`4444`。与 GRPC 一样，`protoc`可以用于为许多语言生成客户端，并且例如生成 Swagger ([`swagger.io/`](https://swagger.io/))文档。

与 GRPC 一样，我们首先将我们的模型定义为`.proto`文件，生成 Go 绑定，最后实现生成的接口。由于使用了`.proto`文件，只要您不依赖于任何框架的更高级功能，代码在 GRPC 和`twitchtv/twirp`之间相对可移植。

此外，因为`twitchtv/twirp`服务器支持 HTTP 1.1，我们可以使用`curl`进行如下操作：

```go
$ curl --request "POST" \ 
 --location "http://localhost:4444/twirp/greeter.GreeterService/Greet" \
 --header "Content-Type:application/json" \
 --data '{"greeting": "Greetings to", "name":"you"}' 

{"response":"Greetings to you."}
```
