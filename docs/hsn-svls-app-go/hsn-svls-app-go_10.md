# 第十章：测试您的无服务器应用程序

本章将教您如何使用 AWS 无服务器应用程序模型在本地测试您的无服务器应用程序。我们还将介绍使用第三方工具进行 Go 单元测试和性能测试，以及如何使用 Lambda 本身执行测试工具。

# 技术要求

本章是第七章的后续内容，*实施 CI/CD 流水线*，因此建议先阅读该章节，以便轻松地跟随本章。此外，建议具有测试驱动开发实践经验。本章的代码包托管在 GitHub 上，网址为[`github.com/PacktPublishing/Hands-On-Serverless-Applications-with-Go`](https://github.com/PacktPublishing/Hands-On-Serverless-Applications-with-Go)。

# 单元测试

对 Lambda 函数进行单元测试意味着尽可能完全地（尽可能）从外部资源（如以下事件：DynamoDB、S3、Kinesis）中隔离测试函数处理程序。这些测试允许您在实际部署新更改到生产环境之前捕获错误，并维护源代码的质量、可靠性和安全性。

在我们编写第一个单元测试之前，了解一些关于 Golang 中测试的背景可能会有所帮助。要在 Go 中编写新的测试套件，文件名必须以`_test.go`结尾，并包含以`TestFUNCTIONNAME`前缀的函数。`Test`前缀有助于识别测试例程。以`_test`结尾的文件将在构建部署包时被排除，并且只有在发出`go test`命令时才会执行。此外，Go 自带了一个内置的`testing`包，其中包含许多辅助函数。但是，为了简单起见，我们将使用一个名为`testify`的第三方包，您可以使用以下命令安装：

```go
go get -u github.com/stretchr/testify
```

以下是我们在上一章中构建的 Lambda 函数的示例，用于列出 DynamoDB 表中的所有电影。以下代表我们要测试的代码：

```go
func findAll() (events.APIGatewayProxyResponse, error) {
  ...

  svc := dynamodb.New(cfg)
  req := svc.ScanRequest(&dynamodb.ScanInput{
    TableName: aws.String(os.Getenv("TABLE_NAME")),
  })
  res, err := req.Send()
  if err != nil {
    return events.APIGatewayProxyResponse{
      StatusCode: http.StatusInternalServerError,
      Body: "Error while scanning DynamoDB",
    }, nil
  }

  ...

  return events.APIGatewayProxyResponse{
    StatusCode: 200,
    Headers: map[string]string{
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    },
    Body: string(response),
  }, nil
}
```

为了充分覆盖代码，我们需要测试所有边缘情况。我们可以执行的测试示例包括：

+   测试在未分配给函数的 IAM 角色的情况下的行为。

+   使用分配给函数的 IAM 角色进行测试。

为了模拟 Lambda 函数在没有 IAM 角色的情况下运行，我们可以删除凭据文件或取消设置本地使用的 AWS 环境变量。然后，发出`aws s3 ls`命令以验证 AWS CLI 无法找到 AWS 凭据。如果看到以下消息，那么您应该可以继续：

```go
Unable to locate credentials. You can configure credentials by running "aws configure".
```

在名为`main_test.go`的文件中编写您的单元测试：

```go
package main

import (
  "net/http"
  "testing"

  "github.com/aws/aws-lambda-go/events"
  "github.com/stretchr/testify/assert"
)

func TestFindAll_WithoutIAMRole(t *testing.T) {
  expected := events.APIGatewayProxyResponse{
    StatusCode: http.StatusInternalServerError,
    Body: "Error while scanning DynamoDB",
  }
  response, err := findAll()
  assert.IsType(t, nil, err)
  assert.Equal(t, expected, response)
}
```

测试函数以`Test`关键字开头，后跟函数名称和我们要测试的行为。然后，它调用`findAll`处理程序并将实际结果与预期响应进行比较。然后，您可以按照以下步骤进行：

1.  使用以下命令启动测试。该命令将查找当前文件夹中的任何文件中的任何测试并运行它们。确保设置`TABLE_NAME`环境变量：

```go
TABLE_NAME=movies go test
```

太棒了！我们的测试有效，因为预期和实际响应体等于**扫描 DynamoDB 时出错**的值：

![](img/b4ba8220-279b-4562-92f4-29cdf384bd74.png)

1.  编写另一个测试函数，以验证如果在运行时将 IAM 角色分配给 Lambda 函数的处理程序行为：

```go
package main

import (
  "testing"

  "github.com/stretchr/testify/assert"
)

func TestFindAll_WithIAMRole(t *testing.T) {
  response, err := findAll()
  assert.IsType(t, nil, err)
  assert.NotNil(t, response.Body)
}
```

再次，测试应该通过，因为预期和实际响应体不为空：

![](img/e9ea1546-d3a4-4e18-8d47-de990ba34e8c.png)

您现在已经在 Go 中运行了一个单元测试；让我们为期望输入参数的 Lambda 函数编写另一个单元测试。让我们以`insert`方法为例。我们要测试的代码如下（完整代码可以在 GitHub 存储库中找到）：

```go
func insert(request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
  ...
  return events.APIGatewayProxyResponse{
    StatusCode: 200,
    Headers: map[string]string{
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    },
  }, nil
}
```

这种情况是输入参数的无效有效负载。函数应返回带有`Invalid payload`消息的`400`错误：

```go
func TestInsert_InvalidPayLoad(t *testing.T) {
  input := events.APIGatewayProxyRequest{
    Body: "{'name': 'avengers'}",
  }

  expected := events.APIGatewayProxyResponse{
    StatusCode: 400,
    Body: "Invalid payload",
  }
  response, _ := insert(input)
  assert.Equal(t, expected, response)
}
```

另一个用例是在给定有效负载的情况下，函数应将电影插入数据库并返回`200`成功代码：

```go
func TestInsert_ValidPayload(t *testing.T) {
  input := events.APIGatewayProxyRequest{
    Body: "{\"id\":\"40\", \"name\":\"Thor\", \"description\":\"Marvel movie\", \"cover\":\"poster url\"}",
  }
  expected := events.APIGatewayProxyResponse{
    StatusCode: 200,
    Headers: map[string]string{
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
    },
  }
  response, _ := insert(input)
  assert.Equal(t, expected, response)
}
```

两个测试应该成功通过。这次，我们将以代码覆盖模式运行`go test`命令，使用`-cover`标志：

```go
TABLE_NAME=movies go test -cover
```

我们有 78%的代码被单元测试覆盖：

![](img/13ecb74b-ad68-471c-a8ee-a3c87b2b656b.png)

如果您想深入了解测试覆盖了哪些语句，哪些没有，可以使用以下命令生成 HTML 覆盖报告：

```go
TABLE_NAME=movies go test -cover -coverprofile=coverage.out
go tool cover -html=coverage.out -o coverage.html
```

如果在浏览器中打开`coverage.html`，您可以看到单元测试未覆盖的语句：

![](img/d234993b-c1bf-4817-9348-fdd985d708ec.png)

您可以通过利用 Go 的接口来改进单元测试，以模拟 DynamoDB 调用。这允许您模拟 DynamoDB 的实现，而不是直接使用具体的服务客户端（例如，[`aws.amazon.com/blogs/developer/mocking-out-then-aws-sdk-for-go-for-unit-testing/`](https://aws.amazon.com/blogs/developer/mocking-out-then-aws-sdk-for-go-for-unit-testing/)）。

# 自动化单元测试

拥有单元测试是很好的。然而，没有自动化的单元测试是没有用的，因此您的 CI/CD 流水线应该有一个测试阶段，以执行对代码存储库提交的每个更改的单元测试。这种机制有许多好处，例如确保您的代码库处于无错误状态，并允许开发人员持续检测和修复集成问题，从而避免在发布日期上出现最后一分钟的混乱。以下是我们在前几章中构建的自动部署 Lambda 函数的流水线的示例：

```go
version: 2
jobs:
 build:
 docker:
 - image: golang:1.8

 working_directory: /go/src/github.com/mlabouardy/lambda-circleci

 environment:
 S3_BUCKET: movies-api-deployment-packages
 TABLE_NAME: movies
 AWS_REGION: us-east-1

 steps:
 - checkout

 - run:
 name: Install AWS CLI & Zip
 command: |
 apt-get update
 apt-get install -y zip python-pip python-dev
 pip install awscli

 - run:
 name: Test
 command: |
 go get -u github.com/golang/lint/golint
 go get -t ./...
 golint -set_exit_status
 go vet .
 go test .

 - run:
 name: Build
 command: |
 GOOS=linux go build -o main main.go
 zip $CIRCLE_SHA1.zip main

 - run:
 name: Push
 command: aws s3 cp $CIRCLE_SHA1.zip s3://$S3_BUCKET

 - run:
 name: Deploy
 command: |
 aws lambda update-function-code --function-name InsertMovie \
 --s3-bucket $S3_BUCKET \
 --s3-key $CIRCLE_SHA1.zip --region us-east-1
```

对 Lambda 函数源代码的所有更改都将触发新的构建，并重新执行单元测试：

![](img/a2bdf032-5bb6-4535-82e6-9df0a6af07ce.png)

如果单击“Test”阶段，您将看到详细的`go test`命令结果：

![](img/80ce3d49-f65b-4387-876c-b1992f325189.png)

# 集成测试

与单元测试不同，单元测试测试系统的一个单元，集成测试侧重于作为一个整体测试 Lambda 函数。那么，在不将它们部署到 AWS 的本地开发环境中如何测试 Lambda 函数呢？继续阅读以了解更多信息。

# RPC 通信

如果您阅读 AWS Lambda 的官方 Go 库（[`github.com/aws/aws-lambda-go`](https://github.com/aws/aws-lambda-go)）的底层代码，您会注意到基于 Go 的 Lambda 函数是使用`net/rpc`通过**TCP**调用的。每个 Go Lambda 函数都会在由`_LAMBDA_SERVER_PORT`环境变量定义的端口上启动服务器，并等待传入请求。为了与函数交互，使用了两个 RPC 方法：

+   `Ping`：用于检查函数是否仍然存活和运行

+   `Invoke`：用于执行请求

有了这些知识，我们可以模拟 Lambda 函数的执行，并执行集成测试或预部署测试，以减少将函数部署到 AWS 之前的等待时间。我们还可以在开发生命周期的早期阶段修复错误，然后再将新更改提交到代码存储库。

以下示例是一个简单的 Lambda 函数，用于计算给定数字的 Fibonacci 值。斐波那契数列是前两个数字的和。以下代码是使用递归实现的斐波那契数列：

```go
package main

import "github.com/aws/aws-lambda-go/lambda"

func fib(n int64) int64 {
  if n > 2 {
    return fib(n-1) + fib(n-2)
  }
  return 1
}

func handler(n int64) (int64, error) {
  return fib(n), nil
}

func main() {
  lambda.Start(handler)
}
```

Lambda 函数通过 TCP 监听端口，因此我们需要通过设置`_LAMBDA_SERVER_PORT`环境变量来定义端口：

```go
_LAMBDA_SERVER_PORT=3000 go run main.go
```

要调用函数，可以使用`net/rpc` go 包中的`invoke`方法，也可以安装一个将 RPC 通信抽象为单个方法的 Golang 库：

```go
go get -u github.com/djhworld/go-lambda-invoke 
```

然后，通过设置运行的端口和要计算其斐波那契数的数字来调用函数：

```go
package main

import (
  "fmt"
  "log"

  "github.com/djhworld/go-lambda-invoke/golambdainvoke"
)

func main() {
  response, err := golambdainvoke.Run(3000, 9)
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(string(response))
}
```

使用以下命令调用 Fibonacci Lambda 函数：

```go
go run client.go
```

结果，`fib(9)=34`如预期返回：

![](img/72d7a601-bd42-4d7f-a922-c34269b10d76.png)

另一种方法是使用`net/http`包构建 HTTP 服务器，模拟 Lambda 函数在 API Gateway 后面运行，并以与测试任何 HTTP 服务器相同的方式测试函数，以验证处理程序。

在下一节中，我们将看到如何使用 AWS 无服务器应用程序模型以更简单的方式在本地测试 Lambda 函数。

# 无服务器应用程序模型

**无服务器应用程序模型**（**SAM**）是一种在 AWS 中定义无服务器应用程序的方式。它是对**CloudFormation**的扩展，允许在模板文件中定义运行函数所需的所有资源。

请参阅第十四章，*基础设施即代码*，了解如何使用 SAM 从头开始构建无服务器应用程序的说明。

此外，AWS SAM 允许您创建一个开发环境，以便在本地测试、调试和部署函数。执行以下步骤：

1.  要开始，请使用`pip` Python 包管理器安装 SAM CLI：

```go
pip install aws-sam-cli
```

确保安装所有先决条件，并确保 Docker 引擎正在运行。有关更多详细信息，请查看官方文档[`docs.aws.amazon.com/lambda/latest/dg/sam-cli-requirements.html`](https://docs.aws.amazon.com/lambda/latest/dg/sam-cli-requirements.html)。

1.  安装后，运行`sam --version`。如果一切正常，它应该输出 SAM 版本（在撰写本书时为*v0.4.0*）。

1.  为 SAM CLI 创建`template.yml`，在其中我们将定义运行函数所需的运行时和资源：

```go
AWSTemplateFormatVersion : '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: List all movies.
Resources:
 FindAllMovies:
 Type: AWS::Serverless::Function
 Properties:
 Handler: main
 Runtime: go1.x
 Events:
 Vote:
 Type: Api
 Properties:
 Path: /movies
 Method: get
```

SAM 文件描述了运行时环境和包含代码的处理程序的名称，当调用时，Lambda 函数将执行该代码。此外，模板定义了将触发函数的事件；在本例中，它是 API Gateway 端点。

+   为 Linux 构建部署包：

```go
GOOS=linux go build -o main
```

+   在本地使用`sam local`命令运行函数：

```go
sam local start-api
```

HTTP 服务器将在端口`3000`上运行并侦听：

![](img/3a87d6d2-f4c5-4724-a0ca-f9773c53b1e1.png)

如果您导航到`http://localhost:3000/movies`，在返回响应之前可能需要几分钟，因为它需要获取一个 Docker 镜像：

![](img/7d01d552-40a9-4efc-88ea-0eb05322e4e0.png)

SAM 本地利用容器的强大功能在 Docker 容器中运行 Lambda 函数的代码。在前面的屏幕截图中，它正在从 DockerHub（一个镜像存储库）拉取`lambci/lambda:go1.x` Docker 镜像。您可以通过运行以下命令来列出机器上所有可用的镜像来确认：

```go
docker image ls
```

以下是前面命令的输出：

![](img/5d6f0c47-6cdd-4a0f-a9a0-44a9af77c2dc.png)

一旦拉取了镜像，将基于您的`deployment`包创建一个新的容器：

![](img/f0a93574-b2b8-468d-9eb0-eedc237b867c.png)

在浏览器中，将显示错误消息，因为我们忘记设置 DynamoDB 表的名称：

![](img/263599b2-995d-4e38-9864-95fc3466c5bb.png)

我们可以通过创建一个`env.json`文件来解决这个问题，如下所示：

```go
{
    "FindAllMovies" : {
        "TABLE_NAME" : "movies"
    }
}
```

使用`--env-var`参数运行`sam`命令：

```go
sam local start-api --env-vars env.json
```

您还可以在同一 SAM 模板文件中使用`Environment`属性声明环境变量。

这次，您应该在 DynamoDB `movies`表中拥有所有电影，并且函数应该按预期工作：

![](img/5bf58d20-54a7-442c-b137-d7269c0322d0.png)

# 负载测试

我们已经看到了如何使用基准测试工具，例如 Apache Benchmark，以及如何测试测试工具。在本节中，我们将看看如何使用 Lambda 本身作为**无服务器测试**测试平台。

这个想法很简单：我们将编写一个 Lambda 函数，该函数将调用我们想要测试的 Lambda 函数，并将其结果写入 DynamoDB 表进行报告。幸运的是，这里不需要编码，因为 Lambda 函数已经在蓝图部分中可用：

![](img/f89c5a72-804b-45f7-b944-0667cbc6abe0.png)

为函数命名并创建一个新的 IAM 角色，如下图所示：

![](img/444ab0b3-a64e-4ab6-b70f-69490d9a6ab7.png)

单击“创建函数”，函数应该被创建，并授予执行以下操作的权限：

+   将日志推送到 CloudWatch。

+   调用其他 Lambda 函数。

+   向 DynamoDB 表写入数据。

以下截图展示了前面任务完成后的情况：

![](img/5cf896bd-c523-4fb9-a3f0-f80760c8b124.png)

在启动负载测试之前，我们需要创建一个 DynamoDB 表，Lambda 将在其中记录测试的输出。该表必须具有`testId`的哈希键字符串和`iteration`的范围数字：

![](img/003994a1-840e-4eda-9522-c116044eb434.png)

创建后，使用以下 JSON 模式调用 Lambda 函数。它将异步调用给定函数 100 次。指定一个唯一的`event.testId`来区分每个单元测试运行：

```go
{
    "operation": "load",
    "iterations": 100,
    "function": "HarnessTestFindAllMovies",
    "event": {
      "operation": "unit",
      "function": "FindAllMovies",
      "resultsTable": "load-test-results",
      "testId": "id",
      "event": {
        "options": {
          "host": "https://51cxzthvma.execute-api.us-east-1.amazonaws.com",
          "path": "/production/movies",
          "method": "GET"
        }
      }
    }
  }
```

结果将记录在 JSON 模式中给出的 DynamoDB 表中：

![](img/1379afc6-c4e7-4fee-81a7-23ff50929bdb.png)

您可能需要修改函数的代码以保存其他信息，例如运行时间、资源使用情况和响应时间。

# 摘要

在本章中，我们学习了如何为 Lambda 函数编写单元测试，以覆盖函数的所有边缘情况。我们还学习了如何使用 AWS SAM 设置本地开发环境，以在本地测试和部署函数，以确保其行为在部署到 AWS Lambda 之前正常工作。

在下一章中，我们将介绍如何使用 AWS 托管的服务（如 CloudWatch 和 X-Ray）来排除故障和调试无服务器应用程序。

# 问题

1.  为`UpdateMovie` Lambda 函数编写一个单元测试。

1.  为`DeleteMovie` Lambda 函数编写一个单元测试。

1.  修改前几章提供的`Jenkinsfile`，以包括自动化单元测试的执行。

1.  修改`buildspec.yml`定义文件，以在将部署包推送到 S3 之前，包括执行单元测试的执行。

1.  为前几章实现的每个 Lambda 函数编写一个 SAM 模板文件。
