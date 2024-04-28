# 第十三章：无服务器编程

本章将重点介绍无服务器架构以及如何在 Go 语言中使用它们。无服务器架构是指开发人员不管理后端服务器的架构。这包括 Amazon Lambda、Google App Engine 和 Firebase 等服务。这些服务允许您快速部署应用程序并在网络上存储数据。

本章中的所有示例都涉及到按使用计费的第三方服务；确保在使用完毕后进行清理。否则，可以将这些示例视为在这些平台上启动更大型应用程序的起步器。

在本章中，我们将涵盖以下内容：

+   使用 Apex 在 Lambda 上进行 Go 编程

+   Apex 无服务器日志和指标

+   使用 Go 的 Google App Engine

+   使用`firebase.google.com/go`与 Firebase 一起工作

# 使用 Apex 在 Lambda 上进行 Go 编程

Apex 是一个用于构建、部署和管理 AWS Lambda 函数的工具。它曾经提供了一个用于在代码中管理 Lambda 函数的 Go `shim`，但现在可以使用原生的 AWS 库([`github.com/aws/aws-lambda-go`](https://github.com/aws/aws-lambda-go))来完成这个任务。本教程将探讨如何创建 Go Lambda 函数并使用 Apex 部署它们。

# 准备工作

根据以下步骤配置您的环境：

1.  从[`golang.org/doc/install`](https://golang.org/doc/install)下载并安装 Go 1.12.6 或更高版本到您的操作系统上。

1.  从[`apex.run/#installation`](http://apex.run/#installation)安装 Apex。

1.  打开终端或控制台应用程序，并创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。本教程中涵盖的所有代码都将在此目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`。在这里，您可以选择从该目录中工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 如何做...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter13/lambda`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/lambda 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/lambda
```

1.  创建一个 Amazon 账户和一个可以编辑 Lambda 函数的 IAM 角色，可以从[`aws.amazon.com/lambda/`](https://aws.amazon.com/lambda/)完成。

1.  创建一个名为`~/.aws/credentials`的文件，内容如下，将您在 Amazon 控制台中设置的凭据复制进去：

```go
        [default]
        aws_access_key_id = xxxxxxxx
        aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxx
```

1.  创建一个环境变量来保存您想要的区域：

```go
        export AWS_REGION=us-west-2
```

1.  运行`apex init`命令并按照屏幕上的说明进行操作：

```go
$ apex init 

Enter the name of your project. It should be machine-friendly, as this is used to prefix your functions in Lambda.

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

1.  删除`lambda/functions/hello`目录。

1.  创建一个新的`lambda/functions/greeter1/main.go`文件，内容如下：

```go
package main

import (
  "context"
  "fmt"

  "github.com/aws/aws-lambda-go/lambda"
)

// Message is the input to the function and
// includes a Name
type Message struct {
  Name string `json:"name"`
}

// Response is sent back and contains a greeting
// string
type Response struct {
  Greeting string `json:"greeting"`
}

// HandleRequest will be called when the lambda function is invoked
// it takes a Message and returns a Response that contains a greeting
func HandleRequest(ctx context.Context, m Message) (Response, error) {
  return Response{Greeting: fmt.Sprintf("Hello, %s", m.Name)}, nil
}

func main() {
  lambda.Start(HandleRequest)
}
```

1.  创建一个新的`lambda/functions/greeter/main.go`文件，内容如下：

```go
package main

import (
  "context"
  "fmt"

  "github.com/aws/aws-lambda-go/lambda"
)

// Message is the input to the function and
// includes a FirstName and LastName
type Message struct {
  FirstName string `json:"first_name"`
  LastName string `json:"last_name"`
}

// Response is sent back and contains a greeting
// string
type Response struct {
  Greeting string `json:"greeting"`
}

// HandleRequest will be called when the lambda function is invoked
// it takes a Message and returns a Response that contains a greeting
// this greeting contains the first and last name specified
func HandleRequest(ctx context.Context, m Message) (Response, error) {
  return Response{Greeting: fmt.Sprintf("Hello, %s %s", m.FirstName, m.LastName)}, nil
}

func main() {
  lambda.Start(HandleRequest)
}
```

1.  部署它们：

```go
$ apex deploy 
• creating function env= function=greeter2
• creating function env= function=greeter1
• created alias current env= function=greeter2 version=4
• function created env= function=greeter2 name=go-cookbook_greeter2 version=1
• created alias current env= function=greeter1 version=5
• function created env= function=greeter1 name=go-cookbook_greeter1 version=1
```

1.  调用新部署的函数：

```go
$ echo '{"name": "Reader"}' | apex invoke greeter1 {"greeting":"Hello, Reader"}

$ echo '{"first_name": "Go", "last_name": "Coders"}' | apex invoke greeter2 {"greeting":"Hello, Go Coders"}
```

1.  查看日志：

```go
$ apex logs greeter2
apex logs greeter2
/aws/lambda/go-cookbook_greeter2 START RequestId: 7c0f9129-3830-11e7-8755-75aeb52a51b9 Version: 1
/aws/lambda/go-cookbook_greeter2 END RequestId: 7c0f9129-3830-11e7-8755-75aeb52a51b9
/aws/lambda/go-cookbook_greeter2 REPORT RequestId: 7c0f9129-3830-11e7-8755-75aeb52a51b9 Duration: 93.84 ms Billed Duration: 100 ms 
Memory Size: 128 MB Max Memory Used: 19 MB 
```

1.  清理已部署的服务：

```go
$ apex delete
The following will be deleted:

- greeter1 - greeter2

Are you sure? (yes/no) yes
• deleting env= function=greeter
• function deleted env= function=greeter
```

# 它是如何工作的...

AWS Lambda 使得无需维护服务器即可按需运行函数变得容易。Apex 提供了部署、版本控制和测试函数的功能，使您可以将它们发送到 Lambda。

Go 库([`github.com/aws/aws-lambda-go`](https://github.com/aws/aws-lambda-go))在 Lambda 中提供了原生的 Go 编译，并允许我们将 Go 代码部署为 Lambda 函数。这是通过定义一个处理程序、处理传入的请求有效负载并返回响应来实现的。目前，您定义的函数必须遵循这些规则：

+   处理程序必须是一个函数。

+   处理程序可能需要零到两个参数。

+   如果有两个参数，则第一个参数必须满足`context.Context`接口。

+   处理程序可能返回零到两个参数。

+   如果有两个返回值，则第二个参数必须是一个错误。

+   如果只有一个返回值，它必须是一个错误。

在这个配方中，我们定义了两个问候函数，一个接受全名，另一个将名字分成名和姓。如果我们修改了一个函数`greeter`，而不是创建两个，Apex 将部署新版本，并在所有先前的示例中调用`v2`而不是`v1`。也可以使用`apex rollback greeter`进行回滚。

# Apex 无服务器日志和指标

在使用 Lambda 等无服务器函数时，拥有可移植的结构化日志非常有价值。此外，您还可以将处理日志的早期配方与此配方结合使用。我们在第四章*Go 中的错误处理*中涵盖的配方同样相关。因为我们使用 Apex 来管理我们的 Lambda 函数，所以我们选择使用 Apex 记录器进行此配方。我们还将依赖 Apex 提供的指标，以及 AWS 控制台。早期的配方探讨了更复杂的日志记录和指标示例，这些仍然适用——Apex 记录器可以轻松配置为使用例如 Amazon Kinesis 或 Elasticsearch 来聚合日志。

# 准备工作

参考本章中*Go 编程在 Apex 上的 Lambda*配方的*准备工作*部分。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter13/logging`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/logging 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/logging
```

1.  创建一个可以编辑 Lambda 函数的 Amazon 帐户和 IAM 角色，可以在[`aws.amazon.com/lambda/`](https://aws.amazon.com/lambda/)上完成。

1.  创建一个`~/.aws/credentials`文件，其中包含以下内容，将您在 Amazon 控制台中设置的凭据复制过来：

```go
        [default]
        aws_access_key_id = xxxxxxxx
        aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxx
```

1.  创建一个环境变量来保存您想要的区域：

```go
        export AWS_REGION=us-west-2
```

1.  运行`apex init`命令并按照屏幕上的说明进行操作：

```go
$ apex init 

Enter the name of your project. It should be machine-friendly, as this is used to prefix your functions in Lambda.

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

1.  删除`lambda/functions/hello`目录。

1.  创建一个新的`lambda/functions/secret/main.go`文件，其中包含以下内容：

```go
package main

import (
  "context"
  "os"

  "github.com/apex/log"
  "github.com/apex/log/handlers/text"
  "github.com/aws/aws-lambda-go/lambda"
)

// Input takes in a secret
type Input struct {
  Secret string `json:"secret"`
}

// HandleRequest will be called when the Lambda function is invoked
// it takes an input and checks if it matches our super secret value
func HandleRequest(ctx context.Context, input Input) (string, error) {
  log.SetHandler(text.New(os.Stderr))

  log.WithField("secret", input.Secret).Info("secret guessed")

  if input.Secret == "klaatu barada nikto" {
    return "secret guessed!", nil
  }
  return "try again", nil
}

func main() {
  lambda.Start(HandleRequest)
}
```

1.  将其部署到指定的区域：

```go
$ apex deploy
• creating function env= function=secret
• created alias current env= function=secret version=1
• function created env= function=secret name=logging_secret version=1
```

1.  要调用它，请运行以下命令：

```go
$ echo '{"secret": "open sesame"}' | apex invoke secret
"try again"

$ echo '{"secret": "klaatu barada nikto"}' | apex invoke secret
"secret guessed!"
```

1.  检查日志：

```go
$ apex logs secret
/aws/lambda/logging_secret START RequestId: cfa6f655-3834-11e7-b99d-89998a7f39dd Version: 1
/aws/lambda/logging_secret INFO[0000] secret guessed secret=open sesame
/aws/lambda/logging_secret END RequestId: cfa6f655-3834-11e7-b99d-89998a7f39dd
/aws/lambda/logging_secret REPORT RequestId: cfa6f655-3834-11e7-b99d-89998a7f39dd Duration: 52.23 ms Billed Duration: 100 ms Memory Size: 128 MB Max Memory Used: 19 MB 
/aws/lambda/logging_secret START RequestId: d74ea688-3834-11e7-aa4e-d592c1fbc35f Version: 1
/aws/lambda/logging_secret INFO[0012] secret guessed secret=klaatu barada nikto
/aws/lambda/logging_secret END RequestId: d74ea688-3834-11e7-aa4e-d592c1fbc35f
/aws/lambda/logging_secret REPORT RequestId: d74ea688-3834-11e7-aa4e-d592c1fbc35f Duration: 7.43 ms Billed Duration: 100 ms 
Memory Size: 128 MB Max Memory Used: 19 MB 
```

1.  检查您的指标：

```go
$ apex metrics secret 

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

在这个配方中，我们创建了一个名为 secret 的新 Lambda 函数，它将根据您是否猜对了秘密短语来做出响应。该函数解析传入的 JSON 请求，使用`Stderr`进行一些日志记录，并返回一个响应。

使用函数几次后，我们可以看到我们的日志可以使用`apex logs`命令查看。此命令可以在单个 Lambda 函数或所有受管理的函数上运行。如果您正在链接 Apex 命令并希望观看许多服务的日志，这将非常有用。

此外，我们还向您展示了如何使用`apex metrics`命令收集有关应用程序的一般指标，包括成本和调用。您还可以在 Lambda 部分的 AWS 控制台中直接查看大量此信息。与其他配方一样，我们在最后尽力清理。

# 使用 Go 的 Google App Engine

App Engine 是谷歌的一个服务，可以快速部署 Web 应用程序。这些应用程序可以访问云存储和各种其他谷歌 API。总体思路是 App Engine 将根据负载轻松扩展，并简化与托管应用相关的任何操作管理。这个配方将展示如何创建并可选部署一个基本的 App Engine 应用程序。这个配方不会深入讨论设置谷歌云帐户、设置计费或清理实例的具体细节。作为最低要求，此配方需要访问 Google Cloud Datastore ([`cloud.google.com/datastore/docs/concepts/overview`](https://cloud.google.com/datastore/docs/concepts/overview))。

# 准备工作

根据这些步骤配置您的环境：

1.  从[`golang.org/doc/install`](https://golang.org/doc/install)下载并安装 Go 1.11.1 或更高版本到您的操作系统。

1.  从[`cloud.google.com/appengine/docs/flexible/go/quickstart`](https://cloud.google.com/appengine/docs/flexible/go/quickstart)下载 Google Cloud SDK。

1.  创建一个允许您执行数据存储访问并记录应用程序名称的应用程序。对于这个配方，我们将使用`go-cookbook`。

1.  安装`gcloud components install app-engine-go` Go app engine 组件。

1.  打开终端或控制台应用程序，并创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。本配方中涵盖的所有代码都将从此目录运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`。在这里，您可以选择从该目录中工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter13/appengine`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/appengine 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/appengine
```

1.  创建一个名为`app.yml`的文件，其中包含以下内容，将`go-cookbook`替换为您在*准备就绪*部分创建的应用程序名称：

```go
runtime: go112

manual_scaling:
  instances: 1

#[START env_variables]
env_variables:
  GCLOUD_DATASET_ID: go-cookbook
#[END env_variables]
```

1.  创建一个名为`message.go`的文件，其中包含以下内容：

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
            m := &amp;amp;Message{
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
            _, err := c.store.GetAll(ctx, q, &amp;amp;messages)
            return messages, err
        }
```

1.  创建一个名为`controller.go`的文件，其中包含以下内容：

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
                return
            }

            ctx := context.Background()

            // store the new message
            r.ParseForm()
            if message := r.FormValue("message"); message != "" {
                if err := c.storeMessage(ctx, message); err != nil {
                    log.Printf("could not store message: %v", err)
                    http.Error(w, "could not store 
                    message", 
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

1.  创建一个名为`main.go`的文件，其中包含以下内容：

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

            port := os.Getenv("PORT")
            if port == "" {
                port = "8080"
                log.Printf("Defaulting to port %s", port)
            }

            log.Printf("Listening on port %s", port)
            log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", port), nil))
        }
```

1.  运行`gcloud config set project go-cookbook`命令，其中`go-cookbook`是您在*准备就绪*部分创建的项目。

1.  运行`gcloud auth application-default login`命令，并按照说明操作。

1.  运行`export PORT=8080`命令。

1.  运行`export GCLOUD_DATASET_ID=go-cookbook`命令，其中`go-cookbook`是您在*准备就绪*部分创建的项目。

1.  运行`go build`命令。

1.  运行`./appengine`命令。

1.  导航到[`localhost:8080/?message=hello%20there`](http://localhost:8080/?message=hello%20there)。

1.  尝试几条消息(`?message=other`)。

1.  可选择使用`gcloud app deploy`将应用程序部署到您的实例。

1.  使用`gcloud app browse`导航到部署的应用程序。

1.  可选择清理您的`appengine`实例和数据存储在以下 URL：

+   [`console.cloud.google.com/datastore`](https://console.cloud.google.com/datastore)

+   [`console.cloud.google.com/appengine`](https://console.cloud.google.com/appengine)

1.  `go.mod`文件可能会更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果您复制或编写了自己的测试，请运行`go test`命令。确保所有测试都通过。

# 它是如何工作的...

一旦云 SDK 配置为指向您的应用程序并已经经过身份验证，GCloud 工具允许快速部署和配置，使本地应用程序能够访问 Google 服务。

在验证和设置端口之后，我们在`localhost`上运行应用程序，然后可以开始使用代码。该应用程序定义了一个可以从数据存储中存储和检索的消息对象。这演示了您可能如何隔离这种代码。您还可以使用存储/数据库接口，如前几章所示。

接下来，我们设置一个处理程序，尝试将消息插入数据存储，然后检索所有消息，在浏览器中显示它们。这创建了类似基本留言簿的东西。您可能会注意到消息并不总是立即出现。如果您在没有消息参数的情况下导航或发送另一条消息，它应该在重新加载时出现。

最后，请确保在不再使用它们时清理实例。

# 使用 firebase.google.com/go 使用 Firebase 进行工作

Firebase 是另一个谷歌云服务，它创建了一个可扩展、易于管理的数据库，可以支持身份验证，并且特别适用于移动应用程序。在这个示例中，我们将使用最新的 Firestore 作为我们的数据库后端。Firebase 服务提供的功能远远超出了本示例涵盖的范围，但我们只会关注存储和检索数据。我们还将研究如何为您的应用程序设置身份验证，并使用我们自己的自定义客户端封装 Firebase 客户端。

# 准备工作

根据以下步骤配置您的环境：

1.  从[`golang.org/doc/install`](https://golang.org/doc/install)下载并安装 Go 1.11.1 或更高版本到您的操作系统。

1.  在[`console.firebase.google.com/`](https://console.firebase.google.com/)创建一个 Firebase 帐户、项目和数据库。

此示例以测试模式运行，默认情况下不安全。

1.  通过访问[`console.firebase.google.com/project/go-cookbook/settings/serviceaccounts/adminsdk`](https://console.firebase.google.com/project/go-cookbook/settings/serviceaccounts/adminsdk)生成服务管理员令牌。在这里，`go-cookbook`将替换为您的项目名称。

1.  将下载的令牌移动到`/tmp/service_account.json`。

1.  打开终端或控制台应用程序，并创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。本示例中涵盖的所有代码都将从该目录运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`。在这里，您可以选择从该目录工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter13/firebase`的新目录，并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/firebase 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/firebase
```

1.  创建一个名为`client.go`的文件，内容如下：

```go
package firebase

import (
  "context"

  "cloud.google.com/go/firestore"
  "github.com/pkg/errors"
)

// Client Interface for mocking
type Client interface {
  Get(ctx context.Context, key string) (interface{}, error)
  Set(ctx context.Context, key string, value interface{}) error
  Close() error
}

// firestore.Client implements Close()
// we create Get and Set
type firebaseClient struct {
  *firestore.Client
  collection string
}

func (f *firebaseClient) Get(ctx context.Context, key string) (interface{}, error) {
  data, err := f.Collection(f.collection).Doc(key).Get(ctx)
  if err != nil {
    return nil, errors.Wrap(err, "get failed")
  }
  return data.Data(), nil
}

func (f *firebaseClient) Set(ctx context.Context, key string, value interface{}) error {
  set := make(map[string]interface{})
  set[key] = value
  _, err := f.Collection(f.collection).Doc(key).Set(ctx, set)
  return errors.Wrap(err, "set failed")
}
```

1.  创建一个名为`auth.go`的文件，内容如下：

```go
package firebase

import (
  "context"

  firebase "firebase.google.com/go"
  "github.com/pkg/errors"
  "google.golang.org/api/option"
)

// Authenticate grabs oauth scopes using a generated
// service_account.json file from
// https://console.firebase.google.com/project/go-cookbook/settings/serviceaccounts/adminsdk
func Authenticate(ctx context.Context, collection string) (Client, error) {

  opt := option.WithCredentialsFile("/tmp/service_account.json")
  app, err := firebase.NewApp(ctx, nil, opt)
  if err != nil {
    return nil, errors.Wrap(err, "error initializing app")
  }

  client, err := app.Firestore(ctx)
  if err != nil {
    return nil, errors.Wrap(err, "failed to intialize filestore")
  }
  return &amp;amp;firebaseClient{Client: client, collection: collection}, nil
}
```

1.  创建一个名为`example`的新目录并进入该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "context"
  "fmt"
  "log"

  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter13/firebase"
)

func main() {
  ctx := context.Background()
  c, err := firebase.Authenticate(ctx, "collection")
  if err != nil {
    log.Fatalf("error initializing client: %v", err)
  }
  defer c.Close()

  if err := c.Set(ctx, "key", []string{"val1", "val2"}); err != nil {
    log.Fatalf(err.Error())
  }

  res, err := c.Get(ctx, "key")
  if err != nil {
    log.Fatalf(err.Error())
  }
  fmt.Println(res)

  if err := c.Set(ctx, "key2", []string{"val3", "val4"}); err != nil {
    log.Fatalf(err.Error())
  }

  res, err = c.Get(ctx, "key2")
  if err != nil {
    log.Fatalf(err.Error())
  }
  fmt.Println(res)
}
```

1.  运行`go run main.go`。

1.  您也可以运行`go build ./example`。您应该会看到以下输出：

```go
$ go run main.go 
[val1 val2]
[val3 val4]
```

1.  `go.mod`文件可能已更新，顶级示例目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

Firebase 提供了方便的功能，让您可以使用凭据文件登录。登录后，我们可以存储任何类型的结构化、类似地图的对象。在这种情况下，我们存储`map[string]interface{}`。这些数据可以被多个客户端访问，包括 Web 和移动设备。

客户端代码将所有操作封装在一个接口中，以便进行测试。这是编写客户端代码时常见的模式，也用于其他示例中。在我们的情况下，我们创建了一个`Get`和`Set`函数，用于按键存储和检索值。我们还公开了`Close()`，以便使用客户端的代码可以延迟`close()`并在最后清理我们的连接。
