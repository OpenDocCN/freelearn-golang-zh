# 服务器端编程

在本章中，我们将介绍以下食谱：

+   使用 Apex 在 Lambda 上进行 Go 编程

+   Apex 服务器端无服务器日志和指标

+   使用 Go 在 Google App Engine 上

+   使用 zabawaba99/firego 与 Firebase 一起工作

# 简介

本章将重点介绍无服务器架构以及使用 Go 语言。它还将探索应用引擎和 Firebase，这两个服务可以快速将应用程序和数据存储部署到网络上。

本章中的所有食谱都涉及第三方服务，这些服务按使用情况收费；确保你在使用完毕后清理。否则，将这些食谱视为在这些平台上启动更大应用程序的启动器。

# 使用 Apex 在 Lambda 上进行 Go 编程

Apex 是一个用于构建、部署和管理 AWS Lambda 函数的工具。它为 Go 提供了包装器（使用 `Node.js` 模拟器）。目前，没有这样的模拟器就无法在 Lambda 上运行原生 Go 代码。本食谱将探索创建 Go Lambda 函数并使用 Apex 部署它们。

# 准备工作

根据以下步骤配置你的环境：

1.  在你的操作系统上下载并安装 Go ([`golang.org/doc/install`](https://golang.org/doc/install)) 并配置你的 `GOPATH` 环境变量。

1.  打开一个终端/控制台应用程序。

1.  导航到你的 `GOPATH/src` 目录并创建一个项目目录，例如，**`$GOPATH/src/github.com/yourusername/customrepo`**。所有代码都将从这个目录运行和修改。

1.  可选地，使用以下步骤安装代码的最新测试版本：

    **`go get github.com/agtorre/go-cookbook/...`** 命令。

1.  从 [`apex.run/`](http://apex.run/) 安装 Apex。

1.  运行 **`go get github.com/apex/go-apex`** 命令。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter12/lambda` 目录并导航到它。

1.  创建一个 Amazon 账户和一个 IAM 角色，可以编辑 Lambda 函数，这可以从 [`aws.amazon.com/lambda/`](https://aws.amazon.com/lambda/) 完成。

1.  创建一个名为 `~/.aws/credentials` 的文件，内容如下，从你在 Amazon 控制台中设置的凭证中复制：

```go
        [example]
        aws_access_key_id = xxxxxxxx
        aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxx

```

1.  创建一个环境变量来保存你想要的区域：

```go
        export AWS_REGION=us-west-2

```

1.  运行 `apex init` 命令并遵循屏幕上的说明：

```go
 $ apex init 

 Enter the name of your project. It should be machine-friendly, 
 as this is used to prefix your functions in Lambda.

 Project name: go-cookbook

 Enter an optional description of your project.

 Project description: Demonstrating Apex with the Go Cookbook

 [+] creating IAM go-cookbook_lambda_function role
 [+] creating IAM go-cookbook_lambda_logs policy
 [+] attaching policy to lambda_function role.
 [+] creating ./project.json
 [+] creating ./functions

 Setup complete, deploy those functions!

 $ apex deploy

```

1.  删除 `lambda/functions/hello` 目录。

1.  创建一个名为 `lambda/functions/greeter/main.go` 的新文件，内容如下：

```go
        package main

        import (
            "encoding/json"
            "fmt"

            "github.com/apex/go-apex"
        )

        type message struct {
            Name string `json:"name"`
        }

        func main() {
            apex.HandleFunc(func(event json.RawMessage, ctx 
            *apex.Context) (interface{}, error) {
                var m message
                if err := json.Unmarshal(event, &m); err != nil {
                    return nil, err
                }

                resp := map[string]string{
                    "greeting": fmt.Sprintf("Hello, %s", m.Name),
                }

                return resp, nil
            })
        }

```

1.  要测试你的函数，你可以运行以下命令：

```go
 $ echo '{"event":{"name": "test"}}' | go run 
 functions/greeter1/main.go 

 {"value":{"greeting":"Hello, test"}}

```

1.  将其部署到指定的区域：

```go
 $apex deploy
 • creating function env= function=greeter
 • created alias current env= function=greeter version=1
 • function created env= function=greeter name=go-
 cookbook_greeter1 version=1

```

1.  要调用它，请运行以下命令：

```go
 $ echo '{"name": "test"}' | apex invoke greeter
 {"greeting":"Hello, test"}

```

1.  现在修改 `lambda/functions/greeter/main.go`：

```go
        package main

        import (
            "encoding/json"
            "fmt"

            "github.com/apex/go-apex"
        )

        type message struct {
            FirstName string `json:"first_name"`
            LastName string `json:"last_name"`
        }

        func main() {
            apex.HandleFunc(func(event json.RawMessage, ctx 
            *apex.Context) (interface{}, error) {
                var m message
                if err := json.Unmarshal(event, &m); err != nil {
                    return nil, err
                }

                resp := map[string]string{
                    "greeting": fmt.Sprintf("Hello, %s %s", 
                    m.FirstName, m.LastName),
                }

                return resp, nil
            })
        }

```

1.  重新部署，创建版本 2：

```go
 $ apex deploy 
 • creating function env= function=greeter
 • created alias current env= function=greeter version=2
 • function created env= function=greeter name=go-
 cookbook_greeter1 version=2

```

1.  调用新部署的函数：

```go
 $ echo '{"first_name": "Go", "last_name": "Coders"}' | apex 
      invoke greeter2
 {"greeting":"Hello, Go Coders"}

```

1.  查看日志：

```go
 $ apex logs greeter
 apex logs greeter
 /aws/lambda/go-cookbook_greeter START RequestId: 7c0f9129-3830-
 11e7-8755-75aeb52a51b9 Version: 2
 /aws/lambda/go-cookbook_greeter END RequestId: 7c0f9129-3830-
 11e7-8755-75aeb52a51b9
 /aws/lambda/go-cookbook_greeter REPORT RequestId: 7c0f9129-3830-
 11e7-8755-75aeb52a51b9 Duration: 93.84 ms Billed Duration: 100 ms 
 Memory Size: 128 MB Max Memory Used: 19 MB 

```

1.  清理已部署的服务：

```go
 $ apex delete
 The following will be deleted:

 - greeter

 Are you sure? (yes/no) yes
 • deleting env= function=greeter
 • function deleted env= function=greeter

```

# 它是如何工作的...

AWS Lambda 使你能够按需运行函数而无需维护服务器。Apex 提供了部署、版本控制和测试函数的设施，当你将它们发送到 Lambda 时。它还提供了一个允许我们执行任意 Go 代码的适配器。这是通过定义一个处理程序、处理传入的请求有效载荷并返回一个响应来实现的，这与标准网络处理程序的流程非常相似。

在这个菜谱中，我们最初接收一个名字输入并问候这个名字。后来，我们利用版本控制将名字拆分为首名和姓氏。也可以部署一个单独的功能。还可以使用 `apex rollback greeter` 进行回滚。

# Apex 无服务器日志和指标

当与 Lambda 等无服务器函数一起工作时，拥有可移植的、结构化的日志非常有价值。此外，你还可以将处理日志的早期菜谱与此菜谱结合。本章中涵盖的 第四章 中关于 Go 中的错误处理的菜谱同样相关。因为我们使用 Apex 来处理我们的 Lambda 函数，所以我们选择使用 Apex 日志记录器来完成这个菜谱。我们还将依赖 Apex 提供的指标以及 AWS 控制台。早期的菜谱探讨了更复杂的日志和指标示例，这些仍然适用——Apex 日志记录器可以轻松配置，以便使用类似 Amazon Kinesis 或 Elasticsearch 这样的工具聚合日志。

# 准备工作

根据以下步骤配置你的环境：

1.  参考本章中 *Go 在 Lambda 上使用 Apex 编程* 菜谱的 *准备工作* 部分。

1.  运行 **`go get github.com/apex/log`** 命令。

# 如何完成...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter12/logging` 目录并导航到它。

1.  创建一个可以编辑 Lambda 函数的 Amazon 账户和 IAM 角色，这可以在 [`aws.amazon.com/lambda/`](https://aws.amazon.com/lambda/) 完成。

1.  创建一个包含以下内容的 `~/.aws/credentials` 文件，从你在亚马逊控制台设置的内容中复制你的凭证：

```go
        [example]
        aws_access_key_id = xxxxxxxx
        aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxx

```

1.  创建一个环境变量来保存你期望的区域：

```go
        export AWS_REGION=us-west-2

```

1.  运行 `apex init` 命令并遵循屏幕上的说明：

```go
 $ apex init 

 Enter the name of your project. It should be machine-friendly, as 
 this is used to prefix your functions in Lambda.

 Project name: logging 

 Enter an optional description of your project.

 Project description: An example of apex logging and metrics

 [+] creating IAM logging_lambda_function role
 [+] creating IAM logging_lambda_logs policy
 [+] attaching policy to lambda_function role.
 [+] creating ./project.json
 [+] creating ./functions

 Setup complete, deploy those functions!

 $ apex deploy

```

1.  删除 `lambda/functions/hello` 目录。

1.  创建一个名为 `lambda/functions/secret/main.go` 的新文件，并包含以下内容：

```go
        package main

        import (
            "encoding/json"
            "os"

            "github.com/apex/go-apex"
            "github.com/apex/log"
            "github.com/apex/log/handlers/text"
        )

        // Input takes in a secret
        type Input struct {
            Secret string `json:"secret"`
        }

        func main() {
            apex.HandleFunc(func(event json.RawMessage, ctx 
            *apex.Context) (interface{}, error) {
                log.SetHandler(text.New(os.Stderr))

                var input Input
                if err := json.Unmarshal(event, &input); err != nil {
                    log.WithError(err).Error("failed to unmarshal key 
                    input")
                    return nil, err
                }
                log.WithField("secret", input.Secret).Info("secret 
                guessed")

                if input.Secret == "klaatu barada nikto" {
                    return "secret guessed!", nil
                }
                return "try again", nil
            })
        }

```

1.  将其部署到指定的区域：

```go
 $ apex deploy
 • creating function env= function=secret
 • created alias current env= function=secret version=1
 • function created env= function=secret name=logging_secret 
 version=1

```

1.  要调用它，请运行以下命令：

```go
 $ echo '{"secret": "open sesame"}' | apex invoke secret
 "try again"

 $ echo '{"secret": "open sesame"}' | apex invoke secret
 "secret guessed!"

```

1.  检查日志：

```go
 $ apex logs secret
 /aws/lambda/logging_secret START RequestId: cfa6f655-3834-11e7-
 b99d-89998a7f39dd Version: 1
 /aws/lambda/logging_secret INFO[0000] secret guessed secret=open 
 sesame
 /aws/lambda/logging_secret END RequestId: cfa6f655-3834-11e7-
 b99d-89998a7f39dd
 /aws/lambda/logging_secret REPORT RequestId: cfa6f655-3834-11e7-
 b99d-89998a7f39dd Duration: 52.23 ms Billed Duration: 100 ms 
 Memory Size: 128 MB Max Memory Used: 19 MB 
 /aws/lambda/logging_secret START RequestId: d74ea688-3834-11e7-
 aa4e-d592c1fbc35f Version: 1
 /aws/lambda/logging_secret INFO[0012] secret guessed 
 secret=klaatu barada nikto
 /aws/lambda/logging_secret END RequestId: d74ea688-3834-11e7-
 aa4e-d592c1fbc35f
 /aws/lambda/logging_secret REPORT RequestId: d74ea688-3834-11e7-
 aa4e-d592c1fbc35f Duration: 7.43 ms Billed Duration: 100 ms 
 Memory Size: 128 MB Max Memory Used: 19 MB 

```

1.  检查你的指标：

```go
 $ apex metrics secret !3445

 secret
 total cost: $0.00
 invocations: 0 ($0.00)
 duration: 0s ($0.00)
 throttles: 0
 errors: 0
 memory: 128

```

1.  清理已部署的服务：

```go
 $ apex delete
 Are you sure? (yes/no) yes
 • deleting env= function=secret
 • function deleted env= function=secret

```

# 它是如何工作的...

在这个菜谱中，我们创建了一个新的 Lambda 函数，名为 secret，它会响应你是否猜对了秘密短语。该函数解析传入的 JSON 请求，使用 `Stderr` 进行一些日志记录，并返回一个响应。

使用该功能几次后，我们发现可以使用 `apex logs` 命令查看日志。此命令可以在单个 lambda 函数或所有我们的管理函数上运行。如果您正在将 Apex 命令链接在一起并希望跨多个服务查看日志，这特别有用。

此外，我们展示了如何使用 apex metrics 命令收集有关您的应用程序的一般指标，包括成本和调用次数。您还可以在 AWS 控制台的 Lambda 部分直接查看大量此类信息。与其他菜谱一样，我们试图在结束时清理。

# 使用 Go 运行 Google App Engine

App Engine 是一个 Google 服务，它简化了快速部署 Web 应用程序。这些应用程序可以访问云存储和各种其他 Google API。一般思路是 App Engine 将会轻松地根据负载进行扩展，并简化与托管应用程序相关的任何操作管理。本菜谱将展示如何创建和可选部署一个基本的 App Engine 应用程序。本菜谱不会涉及设置 Google 云账户、设置计费或清理实例的细节。至少，需要访问 Google Cloud Datastore ([`cloud.google.com/datastore/docs/concepts/overview`](https://cloud.google.com/datastore/docs/concepts/overview)) 才能使本菜谱工作。

# 准备就绪

根据以下步骤配置您的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到您的操作系统上，并配置 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序。

1.  导航到您的 `GOPATH/src` 并创建一个项目目录，例如，`$GOPATH/src/github.com/yourusername/customrepo`。所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/...` 命令安装代码的最新测试版本。

1.  从 [`cloud.google.com/appengine/docs/flexible/go/quickstart`](https://cloud.google.com/appengine/docs/flexible/go/quickstart) 下载 Google Cloud SDK。

1.  创建一个允许部署和访问数据存储的应用程序，并记录应用程序名称。

1.  执行 `go get cloud.google.com/go/datastore` 命令。

1.  执行 `go get google.golang.org/appengine` 命令。

# 如何操作...

这些步骤涵盖了编写和运行您的应用程序：

1.  在您的终端/控制台应用程序中，创建 `chapter12/appengine` 目录并导航到它。

1.  从 `https://github.com/agtorre/go-cookbook/tree/master/chapter12/appengine` 复制测试用例，或者将其作为练习编写一些自己的测试用例。

1.  创建一个名为 `app.yml` 的文件，并包含以下内容，将 `go-cookbook` 替换为 *准备就绪* 部分中创建的应用程序的名称：

```go
        runtime: go
        env: flex

        #[START env_variables]
        env_variables:
            GCLOUD_DATASET_ID: go-cookbook
        #[END env_variables]

```

1.  创建一个名为 `message.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "context"
            "time"

            "cloud.google.com/go/datastore"
        )

        // Message is the object we store
        type Message struct {
            Timestamp time.Time
            Message string
        }

        func (c *Controller) storeMessage(ctx context.Context, message 
        string) error {
            m := &Message{
                Timestamp: time.Now(),
                Message: message,
            }

            k := datastore.IncompleteKey("Message", nil)
            _, err := c.store.Put(ctx, k, m)
            return err
        }

        func (c *Controller) queryMessages(ctx context.Context, limit 
        int) ([]*Message, error) {
            q := datastore.NewQuery("Message").
            Order("-Timestamp").
            Limit(limit)

            messages := make([]*Message, 0)
            _, err := c.store.GetAll(ctx, q, &messages)
            return messages, err
        }

```

1.  创建一个名为 `controller.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "context"
            "fmt"
            "log"
            "net/http"

            "cloud.google.com/go/datastore"
        )

        // Controller holds our storage and other
        // state
        type Controller struct {
            store *datastore.Client
        }

        func (c *Controller) handle(w http.ResponseWriter, r 
        *http.Request) {
            if r.Method != http.MethodGet {
                http.Error(w, "invalid method", 
                http.StatusMethodNotAllowed)
            }

            ctx := context.Background()

            // store the new message
            r.ParseForm()
            if message := r.FormValue("message"); message != "" {
                if err := c.storeMessage(ctx, message); err != nil {
                    log.Printf("could not store message: %v", err)
                    http.Error(w, fmt.Sprintf("could not store 
                    message"), 
                    http.StatusInternalServerError)
                    return
                }
            }

            // get the current messages and display them
            fmt.Fprintln(w, "Messages:")
            messages, err := c.queryMessages(ctx, 10)
            if err != nil {
                log.Printf("could not get messages: %v", err)
                http.Error(w, "could not get messages", 
                http.StatusInternalServerError)
                return
            }

            for _, message := range messages {
                fmt.Fprintln(w, message.Message)
            }
        }

```

1.  创建一个名为 `main.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "log"
            "net/http"
            "os"

            "cloud.google.com/go/datastore"
            "golang.org/x/net/context"
            "google.golang.org/appengine"
        )

        func main() {
            ctx := context.Background()
            log.SetOutput(os.Stderr)

            // Set this in app.yaml when running in production.
            projectID := os.Getenv("GCLOUD_DATASET_ID")

            datastoreClient, err := datastore.NewClient(ctx, projectID)
            if err != nil {
                log.Fatal(err)
            }

            c := Controller{datastoreClient}

            http.HandleFunc("/", c.handle)
            appengine.Main()
        }

```

1.  运行`gcloud config set project go-cookbook`命令，其中`go-cookbook`是你*准备工作*部分中创建的项目。

1.  运行`gcloud auth application-default login`命令并遵循指示。

1.  运行`export PORT=8080`命令。

1.  运行`export GCLOUD_DATASET_ID=go-cookbook`命令，其中`go-cookbook`是你*准备工作*部分中创建的项目。

1.  运行`go build`命令。

1.  运行`./example`命令。

1.  导航到[`localhost:8080/?message=hello%20there`](http://localhost:8080/?message=hello%20there)。

1.  尝试发送几条更多的消息（`?message=other`）

1.  可选地，使用`gcloud app deploy`将应用程序部署到你的实例上。

1.  使用`gcloud app browse`导航到已部署的应用程序。

1.  清理你的 appengine 实例和数据存储：

    +   [`console.cloud.google.com/datastore`](https://console.cloud.google.com/datastore)

    +   [`console.cloud.google.com/appengine`](https://console.cloud.google.com/appengine)

1.  如果你复制或编写了自己的测试，运行`go test`命令。确保所有测试都通过。

# 它是如何工作的...

一旦云 SDK 配置为指向你的应用程序并经过认证，GCloud 工具允许快速部署和配置，使本地应用程序能够访问 Google 服务。

在认证和设置端口后，我们在 localhost 上运行应用程序，然后我们可以开始使用代码。应用程序定义了一个可以存储和从数据存储中检索的消息对象。这展示了你可能如何隔离这类代码。你还可以使用如前几章所示的数据存储/数据库接口。

接下来，我们设置一个处理程序，尝试将消息插入到数据存储中，然后检索所有消息，并在浏览器中显示它们。这创建了一个类似基本留言簿的东西。你可能注意到消息并不总是立即出现。如果你没有消息参数进行导航或发送另一条消息，它应该在重新加载时出现。

最后，确保如果你不再使用它们，清理实例。

# 使用 zabawaba99/firego 与 Firebase 一起工作

Firebase 是另一个 Google 云服务，它创建了一个可扩展、易于管理的数据库，可以支持身份验证，并且与移动应用程序配合得非常好。该服务提供的功能远不止本食谱中涵盖的内容，但我们将探讨如何存储数据、读取数据、修改数据以及恢复数据。我们还将探讨如何为你的应用程序设置身份验证，并用我们自己的自定义客户端包装 Firebase 客户端。

# 准备工作

根据以下步骤配置你的环境：

1.  从[`golang.org/doc/installand`](https://golang.org/doc/installand)下载并安装 Go 到你的操作系统上，并配置你的**`GOPATH`**环境变量。

1.  打开一个终端/控制台应用程序。

1.  导航到你的 `GOPATH/src` 并创建一个项目目录，例如，`$GOPATH/src/github.com/yourusername/customrepo`。所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/...` 命令安装代码的最新测试版本。

1.  在 [`console.firebase.google.com/`](https://console.firebase.google.com/) 创建一个账户和数据库。

1.  从 [`console.firebase.google.com/project/go-cookbook/settings/serviceaccounts/adminsdk`](https://console.firebase.google.com/project/go-cookbook/settings/serviceaccounts/adminsdk) 生成一个服务管理员令牌。

1.  将下载的令牌移动到 `/tmp/service_account.json`。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序创建 `chapter12/firebase` 目录并进入它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter12/firebase`](https://github.com/agtorre/go-cookbook/tree/master/chapter12/firebase) 复制测试代码，或者将其作为练习编写一些自己的代码。

1.  创建一个名为 `client.go` 的文件，并包含以下内容：

```go
        package firebase

        import (
            "log"

            "gopkg.in/zabawaba99/firego.v1"
        )

        // Client Interface for mocking
        type Client interface {
            Get() (map[string]interface{}, error)
            Set(key string, value interface{}) error
        }
        type firebaseClient struct {
            *firego.Firebase
        }

        func (f *firebaseClient) Get() (map[string]interface{}, error) 
        {
            var v2 map[string]interface{}
            if err := f.Value(&v2); err != nil {
                log.Fatalf("error getting")
            }
            return v2, nil
        }

        func (f *firebaseClient) Set(key string, value interface{}) 
        error {
            v := map[string]interface{}{key: value}
            if err := f.Firebase.Set(v); err != nil {
                return err
            }
            return nil
        }

```

1.  创建一个名为 `auth.go` 的文件，并包含以下内容。调整 **[`go-cookbook.firebaseio.com`](https://go-cookbook.firebaseio.com)** 以匹配你的应用程序名称：

```go
        package firebase

        import (
            "io/ioutil"

            "golang.org/x/oauth2"
            "golang.org/x/oauth2/google"
            "gopkg.in/zabawaba99/firego.v1"
        )

        // Authenticate grabs oauth scopes using a generated
        // service_account.json file from
        // https://console.firebase.google.com/project/go-
        cookbook/settings/serviceaccounts/adminsdk
        func Authenticate() (Client, error) {
            d, err := ioutil.ReadFile("/tmp/service_account.json")
            if err != nil {
                return nil, err
            }

            conf, err := google.JWTConfigFromJSON(d, 
            "https://www.googleapis.com/auth/userinfo.email",
            "https://www.googleapis.com/auth/firebase.database")
            if err != nil {
                return nil, err
            }
            f := firego.New("https://go-cookbook.firebaseio.com", 
            conf.Client(oauth2.NoContext))
            return &firebaseClient{f}, err
        }

```

1.  创建一个名为 `example` 的新目录并进入它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容。确保你修改 `channels` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"
            "log"

            "github.com/agtorre/go-cookbook/chapter12/firebase"
        )

        func main() {
            f, err := firebase.Authenticate()
            if err != nil {
                log.Fatalf("error authenticating")
            }
            f.Set("key", []string{"val1", "val2"})
            res, _ := f.Get()
            fmt.Println(res)

            vals := res["key"].([]interface{})
            vals = append(vals, map[string][]string{"key2": 
            []string{"val3"}})
            f.Set("key", vals)
            res, _ = f.Get()
            fmt.Println(res)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行 `go build ./example`。

你现在应该看到以下输出：

```go
 $ go run main.go
 map[key:[val1 val2]]
 map[key:[val1 val2 map[key2:[val3]]]]

```

1.  如果你复制或编写了自己的测试，向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

Firebase 使用 OAuth2 进行身份验证。在这种情况下，我们下载了一个凭证文件，可以与适当的范围请求一起使用，以返回可能与 Firebase 数据库一起工作的令牌。我们可以存储任何类型的结构化类似映射的对象。在这种情况下，我们存储 `map[string]interface{}`。

客户端代码将所有操作封装在一个接口中，以便于测试。这在编写客户端代码时是一个常见的模式，在其他食谱中也被使用。
