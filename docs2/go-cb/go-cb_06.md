# Web 客户端和 API

在本章中，我们将涵盖以下食谱：

+   初始化、存储和传递 http.Client 结构体

+   编写 REST API 的客户端

+   执行并行和异步客户端请求

+   利用 OAuth2 客户端

+   实现 OAuth2 令牌存储接口

+   在客户端添加功能并进行函数组合

+   理解 GRPC 客户端

# 简介

与 API 和编写 Web 客户端可能是一个棘手的话题。不同的 API 有不同的授权、认证和协议类型。我们将探索 `http.Client` 结构体对象，与 OAuth2 客户端和长期令牌存储一起工作，并以额外的 REST 接口结束 GRPC。

到本章结束时，你应该对如何与第三方或内部 API 进行接口交互以及一些常见操作的模式有所了解，例如对 API 的异步请求。

# 初始化、存储和传递 http.Client 结构体

Go 的 `net/http` 包提供了一个灵活的 `http.Client` 结构体，用于与 HTTP API 一起工作。这个结构体有独立的传输功能，相对简单就可以绕过请求，为每个客户端操作修改头信息，并处理任何 REST 操作。创建客户端是一个非常常见的操作，这个食谱将从创建 `http.Client` 对象的基本操作开始。

# 准备工作

根据以下步骤配置你的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到你的操作系统上，并配置你的 `GOPATH` 环境变量。

1.  打开一个终端/控制台应用程序。

1.  导航到 `GOPATH/src` 并创建一个项目目录。例如，`$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装最新测试版本的代码。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter6/client` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter6/client`](https://github.com/agtorre/go-cookbook/tree/master/chapter6/client) 复制测试或将其作为练习编写一些你自己的代码。

1.  创建一个名为 `client.go` 的文件，内容如下：

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

1.  创建一个名为 `exec.go` 的文件，内容如下：

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

1.  创建一个名为 `store.go` 的文件，内容如下：

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

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，内容如下。确保你修改 `client` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter6/client"

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

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下输出：

```go
 $ go run main.go
 results of DefaultGetGolang: 200
 results of DoOps: 200
 results of client.DoOps 200
 results of DefaultGetGolang: 418

```

1.  如果你复制或编写了自己的测试，请进入上一级目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`net/http` 包公开了一个 `DefaultClient` 包变量，该变量用于内部操作，如 `Do`、`GET`、`POST` 等。我们的 `Setup()` 函数返回一个客户端并将默认客户端设置为相同。在设置客户端时，大部分的修改都会发生在传输中，它只需要实现 `RoundTripper` 接口。

本示例提供了一个始终返回 418 状态码的无操作往返器的示例。你可以想象这可能在测试中很有用。它还展示了如何将客户端作为函数参数传递，将其用作结构参数，以及使用默认客户端处理请求。

# 为 REST API 编写客户端

为 REST API 编写客户端不仅可以帮助你更好地理解相关的 API，而且为你提供了所有未来使用该 API 的应用程序的有用工具。这将探讨客户端的结构化以及展示一些你可以立即利用的策略。

对于这个客户端，我们假设身份验证由基本身份验证处理，但也可以通过端点检索令牌等。为了简单起见，我们假设我们的 API 公开了一个端点 `GetGoogle()`，它返回从对 [`www.google.com`](https://www.google.com) 进行 `GET` 请求返回的状态码。

# 准备就绪

请参阅 *准备就绪* 部分，该部分在 *初始化、存储和传递 http.Client 结构体* 菜谱中。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter6/rest` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter6/rest`](https://github.com/agtorre/go-cookbook/tree/master/chapter6/rest) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `client.go` 的文件，内容如下：

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

1.  创建一个名为 `transport.go` 的文件，内容如下：

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

1.  创建一个名为 `exec.go` 的文件，内容如下：

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

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，内容如下。确保你修改 `rest` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter6/rest"

        func main() {
            if err := rest.Exec(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`。

1.  你还可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下输出：

```go
 $ go run main.go
 Result of GetGoogle: 200

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

此代码演示了如何使用 `Transport` 接口隐藏诸如身份验证、令牌刷新等逻辑。它还演示了通过方法公开 API 调用。如果我们正在针对类似用户 API 的东西进行实现，我们预计会有像以下这样的方法：

```go
type API interface{
  GetUsers() (Users, error)
  CreateUser(User) error
  UpdateUser(User) error
  DeleteUser(User)
}

```

如果您已经阅读了第五章，*所有关于数据库和存储*，这可能与配方相似。这种通过接口，尤其是像`RoundTripper`接口这样的通用接口，为编写 API 提供了很多灵活性。此外，编写一个顶级接口，就像我们之前做的那样，并传递接口而不是直接传递客户端，可能也很有用。我们将在下一个配方中进一步探讨，当我们探索编写 OAuth2 客户端时。

# 执行并行和异步客户端请求

在 Go 中并行执行客户端请求相对简单。在下面的配方中，我们将使用客户端通过 Go 缓冲通道检索多个 URL。响应和错误都将进入一个单独的通道，任何人都可以通过访问客户端轻松访问。

在本配方的情况下，客户端的创建、读取通道以及响应和错误的处理都将全部在`main.go`文件中完成。

# 准备工作

参考本章中*初始化、存储和传递 http.Client 结构体*配方中的*准备工作*部分。

# 如何操作...

这些步骤涵盖了编写和运行您的应用程序：

1.  在您的终端/控制台应用程序中，创建名为`chapter6/async`的目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter6/async`](https://github.com/agtorre/go-cookbook/tree/master/chapter6/async)复制测试或将其作为练习编写一些自己的代码。

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

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个名为`main.go`的文件，内容如下。确保您修改`client`导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"
            "net/http"

            "github.com/agtorre/go-cookbook/chapter6/async"
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
 go build ./example

```

您应该看到以下输出：

```go
 $ go run main.go
 Status received for https://www.google.com: 200
 Status received for https://golang.org: 200
 Status received for https://github.com/: 200

```

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 工作原理...

此配方创建了一个框架，用于使用单个客户端以扇出异步方式处理请求。它将尽可能快地检索您指定的尽可能多的 URL。在许多情况下，您可能还想进一步限制，例如使用工作池。也可能有在客户端外部处理这些异步 Go 协程的必要，特别是对于特定的存储或检索接口。

此配方还探讨了使用 case 语句在多个通道上进行切换。我们处理了锁定问题，因为我们知道我们将收到多少响应，并且只有在收到所有响应后才会完成。如果我们可以接受丢弃某些响应，另一个选项可能是超时。

# 使用 OAuth2 客户端

OAuth2 是与 API 通信的相对常见的协议。`golang.org/x/oauth2`包提供了一个相当灵活的客户端，用于与 OAuth2 一起工作。它有子包，指定了各种提供者的端点，例如 Facebook、Google 和 GitHub。

本教程将演示如何创建一个新的 GitHub OAuth2 客户端及其基本用法。

# 准备工作

根据以下步骤配置你的环境：

1.  参考关于初始化、存储和传递 http.Client 结构的*准备工作*部分。

1.  运行`go get golang.org/x/oauth2`命令。

1.  在[`github.com/settings/applications/new`](https://github.com/settings/applications/new)配置 OAuth 客户端。

1.  使用你的客户端 ID 和密钥设置环境变量：

    1.  `export GITHUB_CLIENT="your_client"`

    1.  `export GITHUB_SECRET="your_secret"`

1.  在[`developer.github.com/v3/`](https://developer.github.com/v3/)上复习 GitHub API 文档。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到`chapter6/client`目录。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter6/oauthcli`](https://github.com/agtorre/go-cookbook/tree/master/chapter6/oauthcli)复制测试或将其作为练习编写一些自己的代码。

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
        func GetUsers(client *http.Client) error {
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

1.  创建一个名为`example`的新目录并进入它。

1.  创建一个`main.go`文件，内容如下。确保修改`client`导入以使用你在步骤 2 中设置的路径：

```go
        package main

        import (
            "context"

            "github.com/agtorre/go-cookbook/chapter6/oauthcli"
        )

        func main() {
            ctx := context.Background()
            conf := oauthcli.Setup()

            tok, err := oauthcli.GetToken(ctx, conf)
            if err != nil {
                panic(err)
            }
            client := conf.Client(ctx, tok)

            if err := oauthcli.GetUsers(client); err != nil {
                panic(err)
            }

        }

```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下输出：

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

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

标准的 OAuth2 流程是基于重定向的，并以服务器将重定向到您指定的端点结束。然后，您的服务器负责抓取代码并将其交换为令牌。本教程通过允许我们使用类似`https://localhost`或`https://a-domain-you-own`的 URL，手动复制/粘贴代码，然后按回车键来绕过这一要求。一旦令牌被交换，客户端将根据需要智能地刷新令牌。

重要的是要注意，我们不会以任何方式存储令牌。如果程序崩溃，它必须重新交换令牌。还需要注意的是，除非刷新令牌过期、丢失或损坏，否则我们只需要明确检索令牌一次。一旦客户端配置完成，它应该能够执行所有典型的 HTTP 操作，这些操作针对它授权的 API，并且具有适当的权限范围。

# 实现 OAuth2 令牌存储接口

在之前的配方中，我们检索了客户端的令牌并执行了 API 请求。这种方法的一个缺点是我们没有令牌的长期存储。例如，在一个 HTTP 服务器中，我们希望在请求之间保持令牌的一致存储。

此配方将探讨修改 OAuth2 客户端以在请求之间存储令牌并使用密钥随意检索它们。为了简化，这个密钥将是一个文件，但它也可以是数据库、Redis 等。

# 准备工作

参考配方 *Making use of OAuth2 clients* 中的 *Getting ready* 部分。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter6/client` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter6/oauthstore`](https://github.com/agtorre/go-cookbook/tree/master/chapter6/oauthstore) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `config.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `tokensource.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `storage.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `filestorage.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `example` 的新目录，并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容。确保你将 `oauthstore` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
            "context"
            "io"
            "os"

            "github.com/agtorre/go-cookbook/chapter6/oauthstore"

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

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下输出：

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

1.  如果你复制或编写了自己的测试，向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

此配方负责将令牌的内容存储和检索到文件中。如果是首次运行，它必须执行整个代码交换，但后续运行将重用访问令牌，如果可用，它将使用刷新令牌进行刷新。

在此代码中目前没有区分用户/令牌的方法，但可以通过将 cookies 作为文件名或数据库中的一行作为键来实现。让我们看看这段代码是如何工作的：

+   `config.go` 文件包装了标准的 OAuth2 配置。对于涉及检索令牌的每个方法，我们首先检查本地存储中是否有有效的令牌。如果没有，我们使用标准配置检索一个，然后存储它。

+   `tokensource.go` 文件实现了我们的自定义 `TokenSource` 接口，它与 `Config` 配对。类似于 `Config`，我们首先尝试从文件中检索我们的令牌，如果没有，则使用新的令牌设置。

+   `storage.go` 文件是 `Config` 和 `TokenSource` 使用的 `storage` 接口。它只定义了两个方法，我们还包含了一个辅助函数来引导 OAuth2 基于代码的流程，类似于我们在之前的配方中所做的，但如果存在包含有效令牌的文件，它将使用该文件。

+   `filestorage.go`文件实现了`storage`接口。当我们存储一个新的令牌时，我们首先截断文件并写入`token`结构的 JSON 表示。否则，我们解码文件并返回`token`。

# 包装客户端以添加功能并进行函数组合

在 2015 年，Tomás Senart 做了一次关于使用接口包装`http.Client`结构体的精彩演讲，这使得你可以利用中间件和函数组合。你可以在[`github.com/gophercon/2015-talks`](https://github.com/gophercon/2015-talks)了解更多信息。这个食谱借鉴了他的想法，并演示了如何将类似我们早期食谱“编写 REST API 客户端”的方式应用到`http.Client`结构的`Transport`接口上。

这个食谱将实现一个用于标准`http.Client`结构体的日志记录和基本身份验证中间件。它还包括一个装饰函数，当需要使用大量中间件时可以使用。

# 准备工作

参考本章中“准备就绪”部分的“初始化、存储和传递 http.Client 结构体”食谱。

# 如何实现...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建`chapter6/decorator`目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter6/decorator`](https://github.com/agtorre/go-cookbook/tree/master/chapter6/decorator)复制测试代码，或者将其作为练习来编写你自己的代码。

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

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个`main.go`文件，内容如下。确保将`decorator`导入修改为你在第二步中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter6/decorator"

        func main() {
            if err := decorator.Exec(); err != nil {
                panic(err)
            }
        }

```

1.  运行`go run main.go`。

1.  你还可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下输出：

```go
 $ go run main.go
 started request to https://www.google.com at 2017-01-01 13:38:42
 completed request to https://www.google.com in 194.013054ms
 Response code: 200

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个食谱利用了闭包作为一等公民和接口。允许这样做的主要技巧是有一个函数实现了接口。这允许我们用函数实现的接口来包装由结构体实现的接口。

`middleware.go`文件包含两个示例客户端中间件函数。这些可以扩展以包含额外的中间件，例如更复杂的身份验证、度量等。这个食谱也可以与之前的食谱结合，以生成可以由额外中间件扩展的 OAuth2 客户端。

`Decorator` 函数是一个便利函数，它允许以下操作：

```go
Decorate(RoundTripper, Middleware1, Middleware2, etc)

vs

var t RoundTripper
t = Middleware1(t)
t = Middleware2(t)
etc

```

与包装客户端相比，这种方法的优势在于我们可以保持接口稀疏。如果你想有一个功能齐全的客户端，你还需要实现 `GET`、`POST` 和 `PostForm` 等方法。

# 理解 GRPC 客户端

GRPC 是一个使用协议缓冲区 ([`developers.google.com/protocol-buffers`](https://developers.google.com/protocol-buffers)) 和 HTTP/2 ([`http2.github.io`](https://http2.github.io)) 构建的高性能 RPC 框架。在 Go 中创建 GRPC 客户端与使用 Go HTTP 客户端有很多相似之处。为了演示基本的客户端使用，也最容易实现一个服务器。这个示例将创建一个 `greeter` 服务，它接受一个问候语和一个名字，并返回句子 `<greeting> <name>!`。此外，服务器可以指定是否要感叹号 `!` 或不使用 `.`。

这个示例不会探讨 GRPC 的某些细节，例如流式传输，但希望它能作为创建一个非常基本的客户端和服务器的一个介绍。

# 准备就绪

根据以下步骤配置你的环境：

1.  参考本章中 *初始化、存储和传递 http.Client 结构体* 菜单的 *准备就绪* 部分。

1.  在 [`github.com/grpc/grpc/blob/master/INSTALL.md`](https://github.com/grpc/grpc/blob/master/INSTALL.md) 安装 GRPC。

1.  运行 `go get github.com/golang/protobuf/proto` 命令。

1.  运行 `go get github.com/golang/protobuf/protoc-gen-go` 命令。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter6/grpc` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter6/grpc`](https://github.com/agtorre/go-cookbook/tree/master/chapter6/grpc) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `greeter` 的新目录并进入它。

1.  创建一个名为 `greeter.proto` 的文件，内容如下：

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

1.  运行以下命令：

```go
 protoc --go_out=plugins=grpc:. greeter.proto

```

1.  向上移动一个目录。

1.  创建一个名为 `server` 的新目录并进入它。

1.  创建一个名为 `server.go` 的文件，内容如下。确保你修改 `greeter` 导入以使用第 3 步中设置的路径：

```go
        package main

        import (
            "fmt"
            "net"

            "github.com/agtorre/go-cookbook/chapter6/grpc/greeter"
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

1.  创建一个名为 `greeter.go` 的文件，内容如下。确保你修改 `greeter` 导入以使用第 3 步中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter6/grpc/greeter"
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

1.  向上移动一个目录。

1.  创建一个名为 `client` 的新目录并进入它。

1.  创建一个名为 `client.go` 的文件，内容如下。确保你修改 `greeter` 导入以使用第 3 步中设置的路径：

```go
        package main

        import (
            "context"
            "fmt"

            "github.com/agtorre/go-cookbook/chapter6/grpc/greeter"
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

1.  向上移动一个目录。

1.  运行 `go run server/server.go server/greeter.go`，你将看到以下输出：

```go
 $ go run server/server.go server/greeter.go
 Listening on port :4444

```

1.  在另一个终端中，从 `grpc` 目录运行 `go run client/client.go`，你将看到以下输出：

```go
 $ go run client/client.go
 response:"Hello Reader!" 
 response:"Goodbye Reader!"

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

GRPC 服务器被设置为监听端口 `4444`。一旦客户端连接，它就可以向服务器发送请求并接收响应。请求、响应的结构以及支持的方法由我们在第 4 步中创建的 `.proto` 文件决定。在实际应用中，当与 GRPC 服务器集成时，它们应该提供 `.proto` 文件，该文件可以用于自动生成客户端。

除了客户端之外，`protoc` 命令还会为服务器生成存根，所需做的只是填写实现细节。生成的 Go 代码还包含 JSON 标签，相同的结构体可以用于 JSON REST 服务。我们的代码设置了一个不安全的客户端。为了安全地处理 GRPC，你需要使用 SSL 证书。
