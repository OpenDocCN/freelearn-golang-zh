# 第九章：部署

说到底，当您准备启动您的 Web 服务或 API 时，总会有一些需要考虑的事项，从代码存储库到分段，到实时环境，到停止、启动和更新策略。

部署编译应用程序总是比部署解释应用程序更加复杂。幸运的是，Go 被设计为一种非常现代的编译语言。这意味着，人们已经付出了大量的思考，以解决传统上困扰 C 或 C++构建的服务器和服务的问题。

考虑到这一点，在本章中，我们将探讨一些可用于轻松部署和更新应用程序的工具和策略，以最小化停机时间。

我们还将研究一些可以减少我们的 Web 服务内部负载的方法，例如将图像存储和消息传递作为部署策略的一部分。

在本章结束时，您应该掌握一些特定于 Go 的和一般的技巧，可以最大限度地减少部署 API 和 Web 服务时常见的烦恼，特别是那些需要频繁更新并需要最少停机时间的服务。

在本章中，我们将探讨：

+   应用程序设计和结构

+   云端部署选项和策略

+   利用消息系统

+   将图像托管与我们的 API 服务器分离，并将其连接到基于云的 CDN

# 项目结构

尽管应用程序的设计和基础设施是机构和个人偏好的问题，但您计划其架构的方式可能会对您用于将应用程序部署到云端或任何生产环境中的方法产生真正的影响。

让我们快速回顾一下我们应用程序的结构，记住除非我们打算为大规模跨平台使用而生产我们的应用程序，否则我们不需要包对象：

```go
bin/
  api # Our API binary

pkg/

src/
  github.com/
    nkozyra/
    api/
      /api/api.go
        /interface/interface.go
        /password/password.go
        /pseudoauth/pseudoauth.go
        /services/services.go
        /specification/specification.go
        /v1/v1.go
        /v2/v2.go
```

我们的应用程序的结构可能会引人注目，具体取决于我们如何将其部署到云端。

如果在部署之前有一个处理构建、依赖管理和推送到实时服务器的传输过程，那么这个结构就不相关了，因为源代码和 Go 包依赖可以被二进制文件所取代。

然而，在整个项目被推送到每个应用服务器或服务器或 NFS/文件服务器的情况下，结构仍然是必不可少的。此外，正如前面所指出的，任何需要考虑跨平台分发的地方，都应该保留 Go 项目的整个结构。

即使这并非至关重要，如果构建机器（或机器）与目标机器不完全相同，这会影响您构建二进制文件的过程，尽管这并不排除仅处理该二进制文件。

在一个示例 GitHub 存储库中，如果存在任何开放目录访问，可能还需要对非二进制代码进行混淆，类似于我们的`interface.go`应用程序。

# 使用进程控制来保持您的 API 运行

处理版本控制和开发流程的方法超出了本书的范围，但在为 Web 构建和部署编译代码时，一个相当常见的问题是安装和重新启动这些进程的过程。

管理更新的方式，同时最大限度地减少或消除停机时间对于实时应用程序至关重要。

对于脚本语言和依赖外部 Web 服务器通过 Web 公开应用程序的语言来说，这个过程很容易。脚本要么监听更改并重新启动其内部 Web 服务，要么在未缓存时进行解释，并且更改立即生效。

对于长时间运行的二进制文件，这个过程变得更加复杂，不仅涉及更新和部署我们的应用程序，还涉及确保我们的应用程序处于活动状态，如果服务停止，不需要手动干预。

幸运的是，有几种简单的方法来处理这个问题。第一种是自动维护的严格进程管理。第二种是一个特定于 Go 的工具。让我们首先看看进程管理器以及它们如何与 Go Web 服务一起工作。

## 使用监督者

对于*nix 服务器来说，这里有几个大的解决方案，从非常简单到更复杂和细粒度的解决方案。它们的操作方式没有太大的区别，因此我们将简要地介绍如何使用 Supervisor 来管理我们的 Web 服务。

### 注意

其他一些值得注意的进程管理器如下：

+   Upstart: [`upstart.ubuntu.com/`](http://upstart.ubuntu.com/)

+   Monit: [`mmonit.com/monit/`](http://mmonit.com/monit/)

+   Runit: [`smarden.org/runit/`](http://smarden.org/runit/)

这些直接监督初始化守护进程监控进程管理器的基本原则是监听运行的应用程序，如果没有根据一组配置的规则尝试重新启动它们。

值得指出的是，这些系统没有真正的分布式方法，允许您以聚合方式管理多个服务器的进程，因此通常需要依靠负载均衡器和网络监控来获取此类反馈。

在 Supervisor 的情况下，安装完成后，我们只需要一个简单的配置文件，通常可以通过导航到*nix 发行版上的`/etc/supervisor/conf.d/`来找到。以下是我们应用程序的一个示例文件：

```go
[program:socialnetwork]
command=/var/app/api
autostart=true
autorestart=true
stderr_logfile=/var/log/api.log
stdout_logfile=/var/log/api.log
```

虽然您可以变得更加复杂，例如，将多个应用程序组合在一起以允许同步重启，这对升级非常有用，但这就是您需要保持我们长时间运行的 API 的全部内容。

当需要更新时，比如从 GIT 到暂存再到线上，可以手动触发一个重新启动服务的进程，也可以通过命令以编程方式触发，比如以下命令：

```go
supervisorctl restart program:socialnetwork
```

这不仅可以使您的应用程序保持运行，还可以强制执行一个完整的更新过程，将您的代码推送到线上并触发进程的重新启动。这确保了最小的停机时间。

## 使用 Manners 创建更加优雅的服务器

虽然替代进程管理器在自己的工作中表现得很好，但它们在应用程序内部缺乏一些控制。例如，简单地杀死或重新启动 Web 服务器几乎肯定会中断任何活动的请求。

单独使用 Manners 时，缺少一些像**goagain**这样的进程的监听控制，它是一个将 TCP 监听器聚合到 goroutines 中，并允许通过 SIGUSR1/SIGUSR2 进程间自定义信号进行外部控制的库。

但是，您可以将两者结合使用来创建这样的进程。或者，您可以直接编写内部监听器，因为对于优雅地重新启动 Web 服务器的目的，goagain 可能会有点过度。

使用 Manners 作为`net/http`的替代/包装器的示例将如下所示：

```go
package main

import
(
  "github.com/braintree/manners"
  "net/http"
  "os"
  "os/signal"
)

var Server *GracefulServer

func SignalListener() {
  sC := make(chan os.signal, 1)
  signal.Notify(sC, syscall.SIGUSR1, syscall.SIGUSR2)
  s := <- sC
  Server.Shutdown <- true
}
```

在 goroutine 中运行并阻塞的通道监听 SIGUSR1 或 SIGUSR2 时，当接收到这样的信号时，我们将布尔值传递给`Server.Shutdown`通道。

```go
func Init(allowedDomains []string) {
  for _, domain := range allowedDomains {
    PermittedDomains = append(PermittedDomains, domain)
  }
  Routes = mux.NewRouter()
  Routes.HandleFunc("/interface", APIInterface).Methods("GET", "POST", "PUT", "UPDATE")
  Routes.HandleFunc("/api/user",UserLogin).Methods("POST","GET")
  ...
}
```

这只是我们在`api.go`中`Init()`函数的重新处理。这注册了我们需要 Manners 包装的 Gorilla 路由器。

```go
func main() {

  go func() {
    SignalListener()
  }()
  Server = manners.NewServer()
  Server.ListenAndServe(HTTPport, Routes)
}
```

在`main()`函数中，我们不仅启动`http.ListenAndServe()`函数，还使用 Manners 服务器。

这将防止在发送关闭信号时断开开放的连接。

### 注意

+   您可以使用`go get github.com/braintree/manners`来安装 Manners。

+   您可以在[`github.com/braintree/manners`](https://github.com/braintree/manners)了解更多关于 Manners 的信息。

+   您可以使用`go get github.com/rcrowley/goagain`来安装 goagain。

+   您可以在[`github.com/rcrowley/goagain`](https://github.com/rcrowley/goagain)了解更多关于 goagain 的信息。

# 使用 Docker 部署

在过去几年里，几乎没有什么服务器端产品能像 Docker 在技术世界中引起如此大的轰动。

Docker 创建了类似于易于部署、预配置的虚拟机，与 VirtualBox、VMWare 等传统虚拟机软件相比，对主机的影响要小得多。

它能够以比虚拟机更少的整体重量来实现这一点，通过利用 Linux 容器，这允许用户空间被包含，同时保留对操作系统本身的许多访问权限。这样一来，每个虚拟机就不需要成为操作系统和应用程序的完整镜像了。

为了在 Go 中使用，这通常是一个很好的选择，特别是如果我们为多个目标处理器创建构建，并希望轻松部署 Docker 容器到任何一个或所有这些处理器。更好的是，现在设置方面基本上是开箱即用的，因为 Docker 已经创建了语言堆栈，并在其中包含了 Go。

尽管在其核心，Docker 本质上只是一个典型 Linux 发行版镜像的抽象，但使用它可以使升级和快速配置变得轻而易举，甚至可能提供额外的安全性好处。最后一点取决于您的应用程序及其依赖关系。

Docker 使用非常简单的配置文件，使用语言堆栈，您可以轻松创建一个容器，可以启动并具有我们 API 所需的一切。

看看这个 Docker 文件示例，看看我们如何为我们的社交网络网络服务获取所有必要的包：

```go
FROM golang:1.3.1-onbuild

RUN go install github.com/go-sql-driver/mysql
RUN go install github.com/gorilla/mux
RUN go install github.com/gorilla/sessions
RUN go install github.com/nkozyra/api/password
RUN go install github.com/nkozyra/api/pseudoauth
RUN go install github.com/nkozyra/api/services
RUN go install github.com/nkozyra/api/specification
RUN go install github.com/nkozyra/api/api

EXPOSE 80 443
```

然后可以使用简单的命令构建和运行该文件：

```go
docker build -t api .
docker run --name api-running api -it --rm
```

您可以看到，至少在最低限度下，这将极大地加快 Go 更新过程，跨多个实例（或在这种情况下是容器）。

完整的 Docker 基础镜像也适用于 Google 云平台。如果您使用或希望测试 Google Cloud，这对于快速部署最新版本的 Go 非常有用。

# 在云环境中部署

对于那些还记得满屋子都是物理单用途服务器、毁灭性的硬件故障和极其缓慢的重建和备份时间的人来说，云托管的出现很可能是一大福音。

如今，一个完整的架构通常可以很快地从模板构建，自动扩展和监控也比以往更容易。现在，市场上也有很多参与者，从谷歌、微软和亚马逊到专注于简单、节俭和易用性的小公司，如 Linode 和 Digital Ocean。

每个网络服务都有自己的功能集和缺点，但大多数都共享一个非常常见的工作流程。为了探索 Golang 本身可能通过 API 提供的其他功能，我们将看看亚马逊云服务。

### 注意

请注意，类似的工具也适用于 Go 的其他云平台。甚至微软的平台 Azure 也有一个专为 Go 编写的客户端库。

## 亚马逊云服务

与前述的许多云服务一样，部署到亚马逊云服务或 AWS 基本上与部署到任何标准物理服务器的基础设施没有太大区别。

不过，AWS 有一些区别。首先是它提供的服务范围。亚马逊不仅仅处理静态虚拟服务器。它还处理一系列支持服务，如 DNS、电子邮件和短信服务（通过他们的 SNS 服务）、长期存储等等。

尽管迄今为止已经说了很多，但请注意，许多备选云服务提供类似的功能，可能与以下示例提供的功能类似。

### 使用 Go 直接与 AWS 进行接口

虽然一些云服务确实提供了某种形式的 API 与其服务配套，但没有一个像亚马逊云服务那样强大。

AWS API 提供了对其环境中的每一个可能操作的直接访问，从添加实例、分配 IP 地址、添加 DNS 条目等等。

正如您所期望的那样，直接与此 API 进行接口可以打开许多可能性，因为它涉及自动化应用程序的健康以及管理更新和错误修复。

要直接与 AWS 进行接口，我们将使用`goamz`包启动我们的应用程序：

```go
package main
import (
    "launchpad.net/goamz/aws"
    "launchpad.net/goamz/ec2"
)
```

### 提示

要获取运行此示例所需的两个依赖项，请运行`go get launchpad.net/goamz/aws`命令和`go get launchpad.net/goamz/ec2`命令。

您可以在[`godoc.org/launchpad.net/goamz`](http://godoc.org/launchpad.net/goamz)找到有关此的其他文档。`goamz`包还包括 Amazon S3 存储服务的包，以及 Amazon 的 SNS 服务和简单数据库服务的一些额外实验性包。

基于镜像启动一个新实例很简单。也许对于习惯于手动部署或通过受控、自动化或自动缩放过程部署的人来说，这太简单了。

```go
    AWSAuth, err := aws.EnvAuth()
    if err != nil {
        fmt.Println(err.Error())
    }
    instance := ec2.New(AWSAuth, aws.USEast)
    instanceOptions := ec2.RunInstances({
        ImageId:      "ami-9eaa1cf6",
        InstanceType: "t2.micro",
    })
```

在这种情况下，`ami-9eaa1cf6`指的是 Ubuntu Server 14.04。

在我们的下一节中，拥有与亚马逊 API 的接口将是重要的，我们将把图像数据从我们的关系数据库中移出，并放入 CDN 中。

# 处理二进制数据和 CDN

您可能还记得在第三章中，*路由和引导*，我们看了如何以 BLOB 格式将二进制数据，特别是图像数据，存储在我们应用程序的数据库中。

当时，我们以一种非常基础的方式处理这个问题，只是简单地将二进制图像数据放入某种存储系统中。

Amazon S3 是 AWS 内容分发/交付网络方面的一部分，它基于桶的概念来收集数据，每个桶都有自己的访问控制权限。需要注意的是，AWS 还提供了一个名为 Cloudfront 的真正 CDN，但 S3 可以用作存储服务。

让我们首先看一下使用`goamz`包在给定存储桶中列出最多 100 个项目：

### 提示

在代码中用您的凭据替换-----------。

```go
package main

import
(
  "fmt"
    "launchpad.net/goamz/aws"
    "launchpad.net/goamz/s3"
)

func main() {
  Auth := aws.Auth { AccessKey: `-----------`, SecretKey: `-----------`, }
  AWSConnection := s3.New(Auth, aws.USEast)

  Bucket := AWSConnection.Bucket("social-images")

    bucketList, err := Bucket.List("", "", "", 100)
    fmt.Println(AWSConnection,Bucket,bucketList,err)  
    if err != nil {
        fmt.Println(err.Error())
    }
    for _, item := range bucketList.Contents {
        fmt.Println(item.Key)
    }
}
```

在我们的社交网络示例中，我们将其作为`/api/user/:id:`端点的一部分处理。

```go
 func UsersUpdate(w http.ResponseWriter, r *http.Request) {
  Response := UpdateResponse{}
  params := mux.Vars(r)
  uid := params["id"]
  email := r.FormValue("email")
  img, _, err := r.FormFile("user_image")
  if err != nil {
    fmt.Println("Image error:")
    fmt.Println(err.Error())
```

返回上传，而不是检查错误并继续尝试处理图像，或者我们继续前进。我们将在这里展示如何处理空值：

```go
  }
  imageData, ierr := ioutil.ReadAll(img)
  if err != nil {
    fmt.Println("Error reading image:")
    fmt.Println(err.Error())
```

在这一点上，我们已经尝试读取图像并提取数据——如果我们不能，我们通过`fmt.Println`或`log.Println`打印响应并跳过剩余的步骤，但不要惊慌，因为我们可以以其他方式继续编辑。

```go
  } else {
    mimeType, _, mimerr := mime.ParseMediaType(string(imageData))
    if mimerr != nil {
      fmt.Println("Error detecting mime:")
      fmt.Println(mimerr.Error())
    } else {
      Auth := aws.Auth { AccessKey: `-----------`, SecretKey: `-----------`, }
      AWSConnection := s3.New(Auth, aws.USEast)
      Bucket := AWSConnection.Bucket("social-images")
      berr := Bucket.Put("FILENAME-HERE", imageData, "", "READ")
      if berr != nil {
        fmt.Println("Error saving to bucket:")
        fmt.Println(berr.Error())
      }
    }
  }
```

在第三章中，*路由和引导*，我们接受了表单中上传的数据，将其转换为 Base64 编码的字符串，并保存在我们的数据库中。

由于我们现在要直接保存图像数据，我们可以跳过这最后一步。我们可以从我们请求中的`FormFile`函数中读取任何内容，并将整个数据发送到我们的 S3 存储桶，如下所示：

```go
    f, _, err := r.FormFile("image1")
    if err != nil {
      fmt.Println(err.Error())
    }
    fileData,_ := ioutil.ReadAll(f)
```

对于这个图像，我们需要确保有一个唯一的标识符，避免竞争条件。

## 检查文件上传的存在

`FormFile()`函数实际上在底层调用`ParseMultipartForm()`，并为文件、文件头和标准错误返回默认值（如果不存在）。

## 使用 net/smtp 发送电子邮件

将我们的 API 和社交网络与辅助工具解耦是一个好主意，可以在我们的系统中创建特定性感，减少这些系统之间的冲突，并为每个系统提供更适当的系统和维护规则。

我们可以很容易地为我们的电子邮件系统配备一个套接字客户端，使系统能够直接监听来自我们 API 的消息。实际上，这只需要几行代码就可以实现：

```go
package main

import
(
  "encoding/json"
  "fmt"
  "net"
)

const
(
  port = ":9000"
)

type Message struct {
  Title string `json:"title"`
  Body string `json:"body"`
  To string `json:"recipient"`
  From string `json:"sender"`
}

func (m Message) Send() {

}
func main() {

  emailQueue,_ := net.Listen("tcp",port)
  for {
    conn, err := emailQueue.Accept()
    if err != nil {

    }
    var message []byte
    var NewEmail Message
    fmt.Fscan(conn,message)
    json.Unmarshal(message,NewEmail)
    NewEmail.Send()
  }

}
```

让我们来看一下实际的发送函数，它将把我们 API 中注册过程中的消息发送到电子邮件服务器：

```go
func (m Message) Send() {
  mailServer := "mail.example.com"
  mailServerQualified := mailServer + ":25"
  mailAuth := smtp.PlainAuth(
        "",
        "[email]",
        "[password]",
        mailServer,
      )
  recip := mail.Address("Nathan Kozyra","nkozyra@gmail.com")
  body := m.Body

  mailHeaders := make(map[string] string)
  mailHeaders["From"] = m.From
  mailHeaders["To"] = recip.toString()
  mailHeaders["Subject"] = m.Title
  mailHeaders["Content-Type"] = "text/plain; charset=\"utf-8\""
  mailHeaders["Content-Transfer-Encoding"] = "base64"
  fullEmailHeader := ""
  for k, v := range mailHeaders {
    fullEmailHeader += base64.StdEncoding.EncodeToString([]byte(body))
  }

  err := smtp.SendMail( mailServerQualified, mailAuth, m.From, m.To, []byte(fullEmailHeader))
  if err != nil {
    fmt.Println("could not send email")
    fmt.Println(err.Error())
  }
}
```

虽然这个系统可以很好地工作，因为我们可以监听 TCP 并接收告诉我们要发送什么和发送到什么地址的消息，但它本身并不特别容错。

我们可以通过使用消息队列系统轻松解决这个问题，接下来我们将使用 RabbitMQ 来看一下。

# RabbitMQ with Go

Web 设计的一个方面，特别与 API 相关，但几乎是任何 Web 堆栈的一部分，是服务器和其他系统之间的消息传递的概念。

它通常被称为**高级消息队列协议**或**AMQP**。它可以成为 API/web 服务的重要组成部分，因为它允许否则分离的服务相互通信，而无需使用另一个 API。

通过消息传递，我们在这里谈论的是可以或应该在发生重要事件时在不同的系统之间共享的通用事物被传递给相关的接收者。

再举个类比，就像手机上的推送通知。当后台应用程序有要通知您的事情时，它会生成警报并通过消息传递系统传递。

以下图表是该系统的基本表示。发送者（S），在我们的情况下是 API，将消息添加到堆栈，然后接收者（R）或电子邮件发送过程将检索这些消息：

![RabbitMQ with Go](img/1304OS_09_01.jpg)

我们认为这些过程对 API 特别重要，因为通常有机构希望将 API 与基础设施的其余部分隔离开来。尽管这样做是为了防止 API 资源影响现场站点或允许两个不同的应用程序安全地在相同的数据上运行，但也可以用于允许一个服务接受多个请求，同时允许第二个服务或系统根据资源的允许情况进行处理。

这还为用不同编程语言编写的应用程序提供了非常基本的数据粘合剂。

在我们的 Web 服务中，我们可以使用 AMQP 解决方案告诉我们的电子邮件系统在成功注册后生成欢迎电子邮件。这使我们的核心 API 不必担心这样做，而是可以专注于我们系统的核心。

我们可以通过制定标准消息和标题并将其传递为 JSON 来形式化系统 A 和系统 B 之间的请求的多种方式之一：

```go
type EmailMessage struct {
  Recipient string `json:"to"`
  Sender string `json:"from"`
  Title string `json:"title"`
  Body string `json:"body"`
  SendTime time.Time `json:"sendtime"`
  ContentType string `json:"content-type"`
}
```

以这种方式接收电子邮件，而不是通过开放的 TCP 连接，使我们能够保护消息的完整性。在我们之前的例子中，由于故障、崩溃或关闭而丢失的任何消息将永远丢失。

消息队列，另一方面，就像具有可配置耐久性级别的邮箱一样运作，这使我们能够决定消息应该如何保存，何时过期，以及哪些进程或用户应该访问它们。

在这种情况下，我们使用一个文字消息，作为一个包的一部分交付，将通过队列被我们的邮件服务摄取。在发生灾难性故障的情况下，消息仍将存在，供我们的 SMTP 服务器处理。

另一个重要特性是它能够向消息发起者发送“收据”。在这种情况下，电子邮件系统会告诉 API 或 Web 服务，电子邮件消息已成功从队列中被电子邮件进程取走。

这是在我们简单的 TCP 过程中复制的一些东西。我们必须构建的故障保护和应急措施的数量将使其成为一个非常沉重的独立产品。

幸运的是，在 Go 中集成消息队列是相当简单的：

```go
func Listen() {

  qConn, err := amqp.Dial("amqp://user:pass@domain:port/")
  if err != nil {
    log.Fatal(err)
  }
```

这只是我们与 RabbitMQ 服务器的连接。如果检测到连接出现任何错误，我们将停止该过程。

```go
  qC,err := qConn.Channel()
  if err != nil {
    log.Fatal(err)
  }

  queue, err := qC.QueueDeclare("messages", false, false, false, false, nil)
  if err != nil {
    log.Fatal(err)
  }
```

这里队列的名称有点像 memcache 键或数据库名称一样任意。关键是确保发送和接收机制搜索相同的队列名称：

```go
  messages, err := qC.Consume( queue.Name, "", true, false, false, false, nil)
  waitChan := make(chan int)
  go func() {
    for m := range messages {
      var tmpM Message
      json.Unmarshal(d.Body,tmpM)
      log.Println(tmpM.Title,"message received")
      tmpM.Send()
    }
```

在我们的循环中，我们监听消息并在接收到消息时调用`Send()`方法。在这种情况下，我们传递的是 JSON，然后将其解组为`Message`结构，但这种格式完全取决于您：

```go
  }()

  <- waitChan

}
```

而且，在我们的`main()`函数中，我们需要确保用调用 AMQP 监听器的`Listen()`函数替换我们的无限 TCP 监听器：

```go
func main() {

  Listen()
```

现在，我们有能力从消息队列中接收消息（在电子邮件意义上），这意味着我们只需要在我们的 Web 服务中包含这个功能即可。

在我们讨论的示例用法中，新注册的用户将收到一封电子邮件，提示激活账户。这通常是为了防止使用虚假电子邮件地址进行注册。这并不是一个完全可靠的安全机制，但它确保我们的应用程序可以与拥有真实电子邮件地址的人进行通信。

发送到队列也很容易。

考虑到我们在两个独立应用程序之间共享凭据，将这些内容正式化为一个单独的包是有意义的：

```go
package emailQueue

import
(
  "fmt"
  "log"
  "github.com/streadway/amqp"
)

const
(
  QueueCredentials = "amqp://user:pass@host:port/"
  QueueName = "email"
)

func Listen() {

}

func Send(Recipient string, EmailSubject string, EmailBody string) {

}
```

通过这种方式，我们的 API 和我们的监听器都可以导入我们的`emailQueue`包并共享这些凭据。在我们的`api.go`文件中，添加以下代码：

```go
func UserCreate(w http.ResponseWriter, r *http.Request) {

  ...

  q, err := Database.Exec("INSERT INTO users set user_nickname=?, user_first=?, user_last=?, user_email=?, user_password=?, user_salt=?",NewUser.Name,NewUser.First, NewUser.Last,NewUser.Email,hash,salt)
  if err != nil {
    errorMessage, errorCode := dbErrorParse(err.Error())
    fmt.Println(errorMessage)
    error, httpCode, msg := ErrorMessages(errorCode)
    Response.Error = msg
        Response.ErrorCode = error
    http.Error(w, "Conflict", httpCode)
  } else {

    emailQueue.Send(NewUser.Email,"Welcome to the Social Network","Thanks for joining the Social Network!  Your personal data will help us become billionaires!")

  }
```

在我们的`e-mail.go`进程中：

```go
emailQueue.Listen()
```

### 注意

AMQP 是一个更通用的消息传递接口，具有 RabbitMQ 扩展。您可以在[`github.com/streadway/amqp`](https://github.com/streadway/amqp)上阅读更多信息。

有关 Grab Rabbit Hole 的更多信息，请访问[`github.com/michaelklishin/rabbit-hole`](https://github.com/michaelklishin/rabbit-hole)，或者可以使用`go get github.com/michaelklishin/rabbit-hole`命令进行下载。

# 摘要

通过将 API 的逻辑与我们托管的环境和辅助支持服务分开，我们可以减少功能蔓延和由于非必要功能而导致的崩溃的机会。

在本章中，我们将图像托管从数据库中移到云端，并将原始图像数据和结果引用存储到 S3，这是一个经常用作 CDN 的服务。然后，我们使用 RabbitMQ 演示了如何在部署中利用消息传递。

在这一点上，您应该掌握了将这些服务卸载以及更好地了解部署、更新和优雅重启的可用策略。

在下一章中，我们将开始完成社交网络的最终必要要求，并通过这样做，探索增加我们的 Web 服务的速度、可靠性和整体性能的一些方法。

我们还将引入一个次要服务，允许我们在 SPA 界面内进行社交网络聊天，并扩展我们的图像到 CDN 工作流程，以允许用户创建图库。我们将研究如何通过界面和 API 直接最大化图像呈现和获取的方式。
