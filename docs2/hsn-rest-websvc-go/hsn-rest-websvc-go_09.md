# 异步 API 设计

在本章中，我们将讨论如何为客户设计异步 API。我们将探讨如队列任务和发布/订阅模式等策略。同步请求会在服务器上等待直到返回结果。另一方面，异步（**asynchronous**）请求会立即收到一个包含最终结果信息的响应。现实世界由许多同步和异步事件组成。

异步事件在浏览器中非常流行。异步 API 模仿了现代浏览器中事件循环的行为。在本章中，我们将查看请求类型之间的差异。我们还将用 Go 编写一些客户端，以消费异步 API。

在本章中，我们将涵盖以下主题：

+   理解同步/异步 API 请求

+   服务的扇入/扇出

+   使用队列延迟 API 作业

+   长运行任务设计

+   API 的缓存策略

+   事件驱动 API

# 技术要求

您需要安装以下软件来运行本章的代码示例：

+   操作系统：Linux (Ubuntu 18.04)/Windows 10/Mac OS X >=10.13

+   Go 稳定版本编译器 >= 1.13.5

+   Dep: Go >= 0.5.3 的依赖管理工具

+   Docker 版本 >= 18.09.2

您可以从[`github.com/PacktPublishing/Hands-On-Restful-Web-services-with-Go/tree/master/chapter9`](https://github.com/PacktPublishing/Hands-On-Restful-Web-services-with-Go/tree/master/chapter9)下载本章的代码。克隆代码并使用`chapter9`目录中的代码示例。

# 理解同步/异步 API 请求

同步请求是一个会阻塞服务器直到返回响应的 HTTP 请求。大多数网络服务都是以这种方式运行的。如今，随着分布式系统和松耦合的出现，API 请求也可以是异步的。换句话说，异步请求返回的信息可以用来获取进程的信息。这些服务器上的异步请求与服务器如何并发地为多个客户端执行作业密切相关。让我们看看同步请求的样子：

![图片](img/6ed7e24a-abe2-45fb-b25c-a4716b31faaf.png)

在此类请求中，Web 服务器执行所有操作并返回一个**即时响应**给**Web 客户端**/**移动客户端**。这种方法的缺点是，如果服务器渲染结果花费太多时间，客户端会被阻塞在服务器的操作上。

异步请求会立即返回一个响应，但不包含结果。它为查找请求操作的状态发出一个票据。客户端可以使用该票据（不同的响应）来检查操作的状态和最终结果：

![图片](img/841cf354-4327-437d-987d-36d99bca8471.png)

如前图所示，客户端正在向服务器发送请求，服务器向客户端返回响应。这个响应不是客户端可以立即消费的东西。长时间运行的任务/作业可以通过服务器异步执行。然后，客户端可以使用收到的响应来了解作业的状态。一旦作业完成，服务器可以通知客户端，或者客户端可以通过查看状态来轮询结果。到目前为止，我们只构建了一个同步 API。本章将详细讨论其实现。

在下一节中，我们将讨论 API 如何分化成多个或合并成一个调用。这些技术分别称为输出和输入。

# 服务输入/输出

让我们以一个电子商务网站与第三方支付网关集成的真实世界例子为例。在这里，网站使用支付网关的 API 弹出支付界面并输入安全凭证。同时，网站可能调用另一个名为分析（analytics）的 API 来记录支付尝试。将单个请求分解成多个请求的过程称为**输出**。在现实世界中，单个客户端请求可能涉及许多输出服务。

另一个例子是**MapReduce**。Map 是一个输入操作，而 Reduce 是一个输出操作。服务器可以将一条信息扇出到下一组服务（API），并忽略结果，或者等待从那些服务器返回的所有响应。如图所示，服务器正在将一个进入的请求多路复用到两个出去的请求：

![图片](img/cb751d20-f1e0-4b5e-8c27-a02390d0d5bc.png)

这是一个简单的输出过程。

**输入**是一种操作，其中两个或多个进入的请求汇聚成一个单一请求。这种场景是 API 如何从多个后端服务聚合结果，并即时返回给客户端。例如，考虑一个酒店价格聚合器或航班票务聚合器，它从各种数据提供商获取关于多个酒店或航班的请求信息，并将它们显示出来。以下图显示了输入操作如何结合多个请求，并准备一个最终响应，该响应被客户端消费：

![图片](img/8b9179a1-4dc9-4c6a-9213-33a5deb5649e.png)

客户端也可以是一个服务器，它为其他客户端提供服务。如图所示，左侧的手动服务器正在收集来自**酒店 A**、**酒店 B**和**航空公司 A**的响应，并为不同的客户端准备另一个响应。因此，输入和输出操作并不总是完全独立的。大多数情况下，它将是一个混合场景，其中输入和输出操作相互匹配。

请记住，对下一组服务器的输出操作也可以是异步的。对于输入请求可能不成立。输入操作有时被称为 API 调用。

在本节中，我们看到了扇入和扇出的工作方式。要使用这些技术，我们需要了解如何实现异步服务（API）。在下一节中，我们将尝试使用称为作业排队的机制来实现此类服务。

# 使用排队延迟 API 作业

在同步 API 中，阻塞代码在准备发送给客户端的响应中起着至关重要的作用。然而，在异步设计中，非阻塞是关键。队列和工作者可以帮助实现非阻塞代码。服务器可以并行运行多个工作者，他们可以耗尽队列的内容并对其进行处理。每当客户端通过异步 API 请求操作时，服务器可以将该请求放入作业队列，所有工作者都可以在他们轮到时选择任务。

这种方法可以将 API 服务器卸载，并专注于其业务逻辑，而不是在并行/独立任务（如发送电子邮件、联系第三方服务等）上阻塞。

排队的一些用例如下：

+   压缩图像并通过电子邮件发送最终结果

+   自动背压（将服务器负载限制在可预测的量）

为了详细解释这个概念，让我们制定一个示例并尝试实现它。

让我们开发一个可以执行两种不同类型作业的异步 API 服务器：

+   将给定信息记录到数据库

+   发送电子邮件

条件是它不应该阻塞其他操作。API 应该返回一个 Job ID 票据给客户端，客户端可以使用该信息来获取作业的运行信息。

在开始实施之前，我们应该了解一些启用排队到我们服务的基本知识。我们可以从头开始实现队列/工作者，但有许多优秀的开源排队系统可供选择，例如 RabbitMQ 或 ZeroMQ。

作为实施上述问题的一部分，我们将使用 RabbitMQ，因为它很受欢迎，Go 绑定也很成熟。

# 强大的消息队列 RabbitMQ

RabbitMQ 实现了一种名为 **高级消息队列协议** （**AMQP**）的消息协议。它使用它来支持工作者队列。它还支持许多其他数据交换模式，例如以下内容：

+   发布/订阅

+   主题/订阅

+   路由消息

+   **远程过程调用** （**RPC**）

在本节中，我们将重点关注 RabbitMQ 的消息功能。我们可以使用 Docker 在我们的系统上安装 RabbitMQ，如下所示：

```go
docker run --hostname rabbitmq-host --name rabbitmq-server -p 5672:5672 -p 15672:15672 rabbitmq:3
```

它使用给定的主机名 `rabbitmq-host` 和容器名 `rabbitmq-server` 启动 RabbitMQ 代理。我们使用 `rabbitmq:3` 作为我们服务的基础镜像。Docker 从 Docker Hub 拉取镜像并创建一个容器。您将看到类似以下内容的输出：

```go
 Starting broker...
2019-08-10 08:19:20.371 [info] <0.223.0>
 node           : rabbit@rabbitmq-host
 home dir       : /var/lib/rabbitmq
 config file(s) : /etc/rabbitmq/rabbitmq.conf
 cookie hash    : tUgaG2zTrSrf/yZv3KRV5Q==
 log(s)         : <stdout>
 database dir   : /var/lib/rabbitmq/mnesia/rabbit@rabbitmq-host

....
2019-08-10 08:19:20.873 [info] <0.497.0> started TCP listener on [::]:5672
```

RabbitMQ 使用默认端口 `5672` 进行操作。您可以使用 Docker 命令的初始设置来更改此端口。

前面的 RabbitMQ 代理在前台运行。然而，在生产中，您必须将其在后台运行。这意味着您需要将`-d`标志传递给 Docker 命令以在后台运行，如下所示：

`docker run -d --hostname rabbitmq-host --name rabbitmq-server -p 5672:5672 -p 15672:15672 rabbitmq:3`

默认情况下，如果我们启动容器时（`docker run ...`）不传递用户凭证，将为代理创建一个默认的`<guest:guest>`用户的凭证。您可以在任何时候重置它们或在启动容器时传递它们。您可以在[`hub.docker.com/_/rabbitmq`](https://hub.docker.com/_/rabbitmq)了解更多信息。

# 使用 Go 与 RabbitMQ 通信

现在，我们有一个消息代理（RabbitMQ）。在构建异步 API 之前，我们应该了解 Go 程序如何与消息代理通信并发送/接收消息。在这个过程中，我们将创建用于生产和消费的客户端。

首先，我们必须创建一个`connection`以连接到代理。如果连接成功，需要从连接中创建一个`Channel`。它具有在消息代理上执行操作的 API。然后，我们可以定义一个消息发送到的队列。最后，我们向队列发布一条消息。

我们使用一个名为`amqp`的开源 Go 包来与 RabbitMQ 一起工作。

让我们创建本章的第一个程序：

1.  为消息发送者创建一个如下所示的目录：

```go
mkdir -p $GOPATH/src/github.com/git-user/chapter9/basicSender
```

1.  使用`dep`工具安装`amqp`包：

```go
dep ensure --add "github.com/streadway/amqp"
```

这将在目录中创建`Gopkg.toml`和`Gopkg.lock`文件。

现在，我们准备就绪。我们将查看一个示例，该示例在 RabbitMQ 中创建一个队列并向其发送消息：

1.  首先，让我们在`main.go`内部导入必要的包/库。这些是`log`和`amqp`：

```go
package main

import (
    "log"

    "github.com/streadway/amqp"
)
```

1.  现在，我们需要一个处理程序来处理每一步将生成的错误。Go 的错误处理可能很混乱，这会妨碍可读性。为了有一个干净的代码结构，我们需要在单个位置处理错误：

```go
func handleError(err error, msg string) {
    if err != nil {
        log.Fatalf("%s: %s", msg, err)
    }
}
```

这个函数接收一个错误和一个消息，并将信息记录到`STDOUT`。

1.  现在，让我们编写发送和接收消息的逻辑。在程序的主块中，创建一个连接和通道。然后，使用包含用户凭证的连接字符串连接到 RabbitMQ。一旦连接成功，获取`Channel`对象以推送消息。代码如下所示：

```go
func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    handleError(err, "Dialing failed to RabbitMQ broker")
    defer conn.Close()

    channel, err := conn.Channel()
    handleError(err, "Fetching channel failed")
    defer channel.Close()
}
```

这是连接字符串：

```go
amqp://guest:guest@localhost:5672/
```

它由`protocol ://user:password@host:port`的详细信息组成，其中`host`、`port`、`user`和`password`是 RabbitMQ 服务器的凭证。

在生产环境中，你绝不应该使用 RabbitMQ 的默认凭证。请为所有敏感信息设置强密码，包括 – RabbitMQ。

1.  声明一个名为`test`的队列以发布消息：

```go
testQueue, err := channel.QueueDeclare(
    "test", // Name of the queue
    false,  // Message is persisted or not
    false,  // Delete message when unused
    false,  // Exclusive
    false,  // No Waiting time
    nil,    // Extra args
)

handleError(err, "Queue creation failed")
```

1.  现在，我们有一个队列。让我们准备一个`amqp`消息（RabbitMQ 消息）以将其推入队列。假设消息体是服务器时间的日志：

```go
serverTime := time.Now()
message := amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte(serverTime.String()),
}
```

1.  将前面的消息发布到预定义的队列，即`testQueue`：

```go
err = channel.Publish(
    "",             // exchange
    testQueue.Name, // routing key(Queue)
    false,          // mandatory
    false,          // immediate
    message,
)

handleError(err, "Failed to publish a message")
log.Println("Successfully published a message to the queue")
```

`Publish`方法将给定消息发布到 RabbitMQ 队列。

我们已经完成了发送器的创建。现在，如果我们运行这个程序，它会立即推送一个消息。

现在，让我们编写一个接收器（工作程序）来消费这些消息：

1.  逻辑是定义一个`Consumer`并接收消息。工作程序的代码基本上与之前相同：

```go
mkdir -p $GOPATH/src/github.com/git-user/chapter9/basicReceiver touch $GOPATH/src/github.com/git-user/chapter9/basicReceiver/
main.go
```

1.  使用连接字符串连接到 RabbitMQ，获取`Channel`，并为`testQueue`创建一个表示：

```go
package main

import (
    "log"

    "github.com/streadway/amqp"
)

func handleError(err error, msg string) {
    if err != nil {
        log.Fatalf("%s: %s", msg, err)
    }
}

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    handleError(err, "Dialing failed to RabbitMQ broker")
    defer conn.Close()

    channel, err := conn.Channel()
    handleError(err, "Fetching channel failed")
    defer channel.Close()

    testQueue, err := channel.QueueDeclare(
        "test", // Name of the queue
        false,  // Message is persisted or not
        false,  // Delete message when unused
        false,  // Exclusive
        false,  // No Waiting time
        nil,    // Extra args
    )

    handleError(err, "Queue creation failed")

}
```

1.  我们需要添加一些额外的功能来消费。在主部分的`handleError`之后，我们需要定义要消费的队列及其属性：

```go
messages, err := channel.Consume(
    testQueue.Name, // queue
    "",             // consumer
    true,           // auto-acknowledge
    false,          // exclusive
    false,          // no-local
    false,          // no-wait
    nil,            // args
)

handleError(err, "Failed to register a consumer")
```

1.  `messages`是一个可消费的，可以读取推送到`testQueue`的消息。让我们看看我们如何读取它：

```go
go func() {
    for message := range messages {
        log.Printf("Received a message from the queue: %s",
         message.Body)
    }
}()
```

这运行了一个 goroutine，它启动了一个函数，该函数运行一个无限循环以收集消息并处理它们。goroutine 是由 Go 的运行时引擎管理的轻量级线程。我们可以从一个函数中启动 goroutine。在这里，我们只是在我们的 goroutine 中将消息记录到`STDOUT`。如果我们不阻塞主程序，整个过程将结束，快速杀死 goroutine。

1.  让我们创建一个通道并从中读取以阻塞主程序：

```go
log.Println("Worker has started")
wait := make(chan bool)
<-wait
```

这样，我们就有一个工作程序。

1.  让我们运行这两个程序来看看它们是如何工作的。首先，运行工作程序。如果不存在，它会创建一个名为"text"的队列。然后，在终端上运行发送程序以向队列发送消息：

```go
go run $GOPATH/src/github.com/git-user/chapter9/basicReceiver/
main.go
2019/08/10 16:02:29 Worker has started 
```

1.  在另一个终端窗口（shell）中运行发送程序，将服务器时间日志推送到队列：

```go
go run $GOPATH/src/github.com/git-user/chapter9/basicSender/
main.go
2019/08/10 16:03:15 Successfully published a message to the queue 
```

1.  如果你检查第一个终端，你会看到以下消息：

```go
2019/08/10 16:03:15 Received a message from the queue: 2019-08-10 16:03:15.367476 +0200 CEST m=+0.014980319
```

这意味着工作程序能够成功从队列中检索消息。这个功能可以被 API 服务器利用，将长运行作业放入消息队列，并让专门的工人处理它们。

在掌握队列的基础知识和它如何帮助我们构建异步 API 之后，我们应该实现一个现实世界的问题。在下一节中，我们将定义问题陈述并尝试设计一个同时执行各种功能的长运行任务。

# 长运行任务设计

到目前为止，我们已经了解了队列的基础知识以及如何延迟作业。现在，我们将设计一个解决方案来解决有关异步 API 的问题。问题是，我们想要构建一个可以处理以下场景请求的系统：

+   服务器应该将信息作为一个操作保存到数据库中。

+   它应该发送电子邮件到指定的电子邮件地址。

+   它应该执行一个长运行的任务并将结果 POST 到一个回调。这被称为 web 钩子。

假设这三个操作是异步和长运行的。我们需要一个机制来促进具有以下特征的长运行过程：

+   客户端可以触发一个 API 并返回一个作业 ID。

+   作业以相应的消息格式推送到队列中。

+   工作者选择工作并开始执行。

+   最后，工作者将结果保存在各种端点上，并将状态发送到数据库。

以下图详细显示了前面的要求：

![](img/32f92cad-16d3-40c7-bb77-9dacb758c13f.png)

以下图显示了几个引人入胜的实体：

+   **API 服务器**

+   ****数据库服务器****

+   **队列**

+   **工作者**

**API 服务器**正在接受来自客户端的异步请求，并将这些工作推送到消息队列。然后，工作者选择这些工作并对它们执行一些操作。

**工作者 A** 将消息中的信息保存到数据库中。**工作者 B** 选择一个工作。在处理消息后，它将一些信息发布到作为请求一部分接收的回调。**工作者 C** 的任务是发送电子邮件。

为了简化，我们将模拟最终操作（数据库插入、发送电子邮件和回调）。我们这样做是为了专注于异步 API 设计而不是具体操作。

为了设计这个流程，我们需要重用相同的消息并使其适合所有用例。JSON 是存储关于作业信息的更好格式。

我们需要创建一些结构来保存关于工作的信息，如下所示：

+   **作业**：作业的全局存储

+   **日志**：专门针对作业 A 的信息

+   **回调**：专门针对作业 B 的信息

+   **邮件**：专门针对作业 C 的信息

A、B 和 C 是前面图中提到的工作者类型。

现在，让我们定义我们的项目。创建项目目录并开发前面架构中显示的每个部分是这个过程的一部分：

1.  创建项目：

```go
mkdir -p $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV1
```

我们将其命名为 `V1`（版本 1），因为我们第一次尝试实现异步性。在接下来的章节中，我们将添加更多功能，并推出更多版本。

1.  我们需要将我们的结构存储在 `models` 目录中。创建一个名为 `models` 的包，并添加一个新文件来存储前面的结构：

```go
mkdir -p $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV1
/models touch $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV1
/models/job.go 
```

1.  对于字段，我们有 `UUID` 来跟踪工作，`type` 来区分工作，以及针对各自工作的特定额外数据。我们使用 Google 的 `UUID` 包生成 `UUID` 字符串并设置作业 ID。`type` 可以是 "A"，"B" 或 "C"。`Log` 用于与时间相关的操作，因此需要一个时间字段。`callback` 需要一个回调 URL 来发布数据。`mail` 需要一个电子邮件地址来发送消息。结构文件包含以下结构：

```go
package models

import (
    "time"

    "github.com/google/uuid"
)

// Job represents UUID of a Job
type Job struct {
    ID      uuid.UUID   `json:"uuid"`
    Type      string      `json:"type"`
    ExtraData interface{} `json:"extra_data"`
}

// Worker-A data
type Log struct {
    ClientTime time.Time `json:"client_time"`
}

// CallBack data
type CallBack struct {
    CallBackURL string `json:"callback_url"`
}

// Mail data
type Mail struct {
    EmailAddress string `json:"email_address"`
}
```

之前文件中的重要字段是 `ExtraData`:

```go
    ExtraData interface{} `json:"extra_data"`
```

我们将其定义为接口，并使其成为 Log、Callback 和 Mail 的占位符。当我们发布消息时，我们实例化相应的结构。

1.  在主程序中，我们必须定义一些辅助函数和常量。我们将这些添加到项目的主文件中：

```go
touch $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV1
/main.go
```

1.  定义队列名称、HTTP 服务器运行的地址以及处理任何错误的错误处理器：

```go
const queueName string = "jobQueue"
const hostString string = "127.0.0.1:8000"

func handleError(err error, msg string) {
    if err != nil {
        log.Fatalf("%s: %s", msg, err)
    }
}
```

这样做是为了避免重复代码。

我们必须在我们的项目中再烘焙几个组件：

+   一个 HTTP 服务器

+   工作者

+   URL 处理器

`Handler`接收一个传入的请求并尝试创建一个即时作业 ID。一旦它成功将作业放入队列，它就会将作业 ID 返回给调用者。现在，已经启动并监听作业队列的工作者将选择这些任务并发地执行它们。

1.  为工作者创建一个文件：

```go
touch $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV1
/worker.go
```

`Workers`是一个`struct`，它包含对消息队列的连接。使用该连接，所有工作者从队列中读取：

```go
type Workers struct {
    conn *amqp.Connection
}
```

在某个时候，我们需要启动工作者。为此，我们需要定义一个运行方法来启动/引导工作者。工作者应该监听消息队列中的消息并消费它们。

1.  一旦有传入的消息，检查工作类型并将其委派给相应的函数，即`dbWork`、`callbackWork`和`emailWork`：

```go
func (w *Workers) run() {
    log.Printf("Workers are booted up and running")
    channel, err := w.conn.Channel()
    handleError(err, "Fetching channel failed")
    defer channel.Close()

    jobQueue, err := channel.QueueDeclare(
        queueName, // Name of the queue
        false,     // Message is persisted or not
        false,     // Delete message when unused
        false,     // Exclusive
        false,     // No Waiting time
        nil,       // Extra args
    )
    handleError(err, "Job queue fetch failed")

    messages, err := channel.Consume(
        jobQueue.Name, // queue
        "",            // consumer
        true,          // auto-acknowledge
        false,         // exclusive
        false,         // no-local
        false,         // no-wait
        nil,           // args
    )
    go func() {
        for message := range messages {

            job := models.Job{}
            err = json.Unmarshal(message.Body, &job)

            log.Printf("Workers received a message from the queue:
             %s", job)
            handleError(err, "Unable to load queue message")

            switch job.Type {
            case "A":
                w.dbWork(job)
            case "B":
                w.callbackWork(job)
            case "C":
                w.emailWork(job)
            }
        }
    }()
    defer w.conn.Close()
    wait := make(chan bool)
    <-wait // Run long-running worker
}
```

1.  函数结束时，我们关闭了通道并阻塞了工作者，因为 goroutines 在后台运行。

1.  现在，我们可以通过为三个函数`dbWork`、`callbackWork`和`emailWork`添加延迟来模拟实际工作者的工作。我们使用延迟来模拟后台工作并相应地记录消息。我们将在`workers`结构体上定义这些函数，以便函数紧密关联：

```go

func (w *Workers) dbWork(job models.Job) {
    result := job.ExtraData.(map[string]interface{})
    log.Printf("Worker %s: extracting data..., JOB: %s",
     job.Type, result)
    time.Sleep(2 * time.Second)
    log.Printf("Worker %s: saving data to database...,
     JOB: %s", job.Type, job.ID)
}

func (w *Workers) callbackWork(job models.Job) {
    log.Printf("Worker %s: performing some long running process...,
     JOB: %s", job.Type, job.ID)
    time.Sleep(10 * time.Second)
    log.Printf("Worker %s: posting the data back to the given
     callback..., JOB: %s", job.Type, job.ID)
}

func (w *Workers) emailWork(job models.Job) {
    log.Printf("Worker %s: sending the email..., JOB: %s",
     job.Type, job.ID)
    time.Sleep(2 * time.Second)
    log.Printf("Worker %s: sent the email successfully,
     JOB: %s", job.Type, job.ID)
}
```

这些工作者独立于主程序工作。它们监听消息队列并按照其类型处理传入的消息。通过这种方式，我们定义了端点/工作者。

1.  现在，是时候为我们的 HTTP 服务器定义一些端点了，这些端点接受 API 请求并将消息发布到队列。这些将放入一个名为`handlers.go`的新文件中：

```go
touch $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV1
/handlers.go
```

1.  我们的管理员也需要访问消息队列的连接，这是一个我们可以发布消息的通道。因此，最好为服务器定义一个结构体，并将处理程序定义为方法。让我们称它为`JobStruct`：

```go
// JobServer holds handler functions
type JobServer struct {
    Queue   amqp.Queue
    Channel *amqp.Channel
    Conn    *amqp.Connection
}
```

1.  我们应该在前面定义的结构体上附加一个名为`publish`的方法。所有处理程序都可以使用此方法将 JSON 体发布到消息队列。这与我们在介绍 RabbitMQ 通道时探索的逻辑类似：

```go
func (s *JobServer) publish(jsonBody []byte) error {
    message := amqp.Publishing{
        ContentType: "application/json",
        Body:        jsonBody,
    }
    err := s.Channel.Publish(
        "",        // exchange
        queueName, // routing key(Queue)
        false,     // mandatory
        false,     // immediate
        message,
    )

    handleError(err, "Error while generating JobID")
    return err
}
```

现在，让我们定义三个处理程序，它们将处理三种类型的工作，如下所示：

+   第一个处理程序为工作类型 A 创建一个任务——将客户端时间保存到数据库中。

+   第二个处理程序为工作类型 B 创建一个任务——在一段时间后回调 URL。

+   第三个处理程序为工作类型 C 创建一个任务——发送电子邮件。

对于第一个处理程序，我们从 HTTP 请求中获取一个名为`client_time`的查询参数，并将其用于在数据库中保存。我们使用`json`和`strconv`进行所需的数据转换。一旦我们有了工作者所需的所有信息，我们就可以组合 JSON 并将其发布到队列中：

```go
func (s *JobServer) asyncDBHandler(w http.ResponseWriter,
r *http.Request) {
    jobID, err := uuid.NewRandom()
    queryParams := r.URL.Query()
    // Ex: client_time: 1569174071
    unixTime, err := strconv.ParseInt(queryParams.Get("client_time"),
     10, 64)
    clientTime := time.Unix(unixTime, 0)
    handleError(err, "Error while converting client time")

    jsonBody, err := json.Marshal(models.Job{ID: jobID,
        Type:      "A",
        ExtraData: models.Log{ClientTime: clientTime},
    })
    handleError(err, "JSON body creation failed")

    if s.publish(jsonBody) == nil {
        w.WriteHeader(http.StatusOK)
        w.Header().Set("Content-Type", "application/json")
        w.Write(jsonBody)
    } else {
        w.WriteHeader(http.StatusInternalServerError)
    }
}
```

如您所见，此函数处理程序为处理`"A"`（数据库作业）的工作者组合了所需的消息。额外的信息在`ExtraData`字段中传递。正如我们之前提到的，接口可以适应任何新的结构体。因此，在运行时，我们在`ExtraData`中设置可以放入的内容。

其他两个处理程序看起来完全一样，只是`jsonBody`的组成不同。

处理程序 2 的 JSON 消息如下：

```go
    jsonBody, err := json.Marshal(models.Job{ID: jobID,
        Type:      "B",
        ExtraData: "", // Can be custom data, Ex: {"client_time":
                       // "2020-01-22T20:38:15+02:00"}
 })
```

处理程序 3 的 JSON 消息如下：

```go
    jsonBody, err := json.Marshal(models.Job{ID: jobID,
        Type:      "C",
        ExtraData: "", // Can be custom data, Ex: {"email_address":
                       // "packt@example.org"}
 })
```

接下来是主程序。我们必须使用我们的主逻辑将工人、处理程序和结构体粘合在一起。之前，我们添加了常量，但现在我们必须扩展这一点，以使工人和 API 活跃起来。

最后，我们应该将迄今为止构建的每一部分都粘合在一起。按照以下步骤操作：

1.  我们需要一个返回`JobServer`对象的函数。让我们将这个名为`getServer`的函数添加到`main.go`文件中。作业服务器持有连接和队列。代码如下：

```go
func getServer(name string) JobServer {
    /*
        Creates a server object and initiates
        the Channel and Queue details to publish messages
    */
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    handleError(err, "Dialing failed to RabbitMQ broker")

    channel, err := conn.Channel()
    handleError(err, "Fetching channel failed")

    jobQueue, err := channel.QueueDeclare(
        name,  // Name of the queue
        false, // Message is persisted or not
        false, // Delete message when unused
        false, // Exclusive
        false, // No Waiting time
        nil,   // Extra args
    )
    handleError(err, "Job queue creation failed")
    return JobServer{Conn: conn, Channel: channel, Queue: jobQueue}
}
```

使用这个服务器，我们可以将 URL 端点链接到处理程序函数。这些函数使用 RabbitMQ/消息队列的实例化连接属性。

1.  现在，通过调用前面的函数并传入`queueName`（我们将其定义为常量）来获取`JobServer`：

```go
func main() {
    jobServer := getServer(queueName)

    // Rest of the code goes here....
}
```

1.  接下来，我们应该启动工人。如果我们正常启动他们，他们将阻塞主执行线程。因此，我们必须将它们变成 goroutines：

```go
// Start Workers
go func(conn *amqp.Connection) {
    workerProcess := Workers{
        conn: jobServer.Conn,
    }
    workerProcess.run()
}(jobServer.Conn)
```

1.  为了接收客户端请求并使我们的应用程序成为可能，我们必须将处理程序附加到 URL。可以使用 Gorilla Mux 路由器来完成此操作。我们在第二章中详细讨论了它，处理我们的 REST 服务的路由。我们将重用那里使用的相同模式将路由附加到处理程序：

```go
router := mux.NewRouter()
// Attach handlers
router.HandleFunc("/job/database", jobServer.asyncDBHandler)
router.HandleFunc("/job/mail", jobServer.asyncMailHandler)
router.HandleFunc("/job/callback", jobServer.asyncCallbackHandler)

httpServer := &http.Server{
    Handler:      router,
    Addr:         hostString,
    WriteTimeout: 15 * time.Second,
    ReadTimeout:  15 * time.Second,
}

// Run HTTP server
log.Fatal(httpServer.ListenAndServe())
```

这将启动一个 HTTP 服务器并将请求路由到我们之前定义的 URL。正如你可能已经注意到的，我们正在使用作业服务器的处理程序作为端点。

1.  最后但同样重要的是，我们应该安全地关闭连接和通道：

```go
// Cleanup resources
defer jobServer.Channel.Close()
defer jobServer.Conn.Close()
```

这就完成了我们的示例。让我们从项目根目录（`longRunningTask`）构建 Go 项目并查看输出：

确保你的 RabbitMQ 服务器没有关闭。我们的作业服务器使用 RabbitMQ 作为消息队列。

1.  运行`go build`命令：

```go
go build .
```

1.  这将生成一个与项目同名的可执行文件，即`longRunningTaskV1`。我们可以这样启动我们的 HTTP 服务器：

```go
./longRunningTaskV1
2019/09/22 20:36:06 Workers are booted up and running
```

1.  服务器现在正在端口`8000`上运行。向服务器发送几个`curl` `GET`请求：

```go
> curl -X GET http://localhost:8000/job/database\?client_time\=1569177495
{"uuid":"9dfbc374-a046-4b29-b6f8-5414a277aaa2","type":"A","extra_data":{"client_time":"2019-09-22T20:38:15+02:00"}}

> curl -X GET http://localhost:8000/job/callback
{"uuid":"ac297c92-74ec-4fcb-b3e6-6dfb96eb45e0","type":"B","extra_data":""}

> curl -X GET http://localhost:8000/job/mail
{"uuid":"4ed59a6f-24d8-4179-8432-fe4adcdd4f51","type":"C","extra_data":""}
```

1.  服务器不会阻塞请求，而是快速返回任务的工作 ID。让我们看看服务器日志：

```go
2019/09/22 20:39:56 Workers received a message from the queue: {9dfbc374-a046-4b29-b6f8-5414a277aaa2 A map[client_time:2019-09-22T20:38:15+02:00]}
2019/09/22 20:39:56 Worker A: extracting data..., JOB: map[client_time:2019-09-22T20:38:15+02:00]
2019/09/22 20:39:58 Worker A: saving data to database..., JOB: 9dfbc374-a046-4b29-b6f8-5414a277aaa2
2019/09/22 20:40:29 Workers received a message from the queue: {ac297c92-74ec-4fcb-b3e6-6dfb96eb45e0 B }
2019/09/22 20:40:29 Worker B: performing some long running process..., JOB: ac297c92-74ec-4fcb-b3e6-6dfb96eb45e0
2019/09/22 20:40:39 Worker B: posting the data back to the given callback..., JOB: ac297c92-74ec-4fcb-b3e6-6dfb96eb45e0
2019/09/22 20:40:39 Workers received a message from the queue: {4ed59a6f-24d8-4179-8432-fe4adcdd4f51 C }
2019/09/22 20:40:39 Worker C: sending the email..., JOB: 4ed59a6f-24d8-4179-8432-fe4adcdd4f51
2019/09/22 20:40:41 Worker C: sent the email successfully, JOB: 4ed59a6f-24d8-4179-8432-fe4adcdd4f51
```

发送邮件有 2 秒的延迟，但客户端不会在该决策上阻塞。这就是异步 API 按设计工作的方式。

在设计异步 API 之前，总是要提前准备一个设计。由于对于给定的问题没有银弹，你必须探索各种架构，例如消息队列。

好的，但客户端如何检索作业的状态，无论是已启动、进行中还是已完成？为了启用这个功能，我们必须将作业的状态存储在某个地方。这可以是一个数据库或临时缓存。现代应用程序通过轮询 API 来获取作业状态，进行大量的读取操作。Redis 是这类问题的良好缓存解决方案。我们可以通过 Redis 扩展这个示例来解决`查找作业 ID 的状态`的问题。

在下一节中，我们将介绍 Redis，包括安装 Redis 并将其链接到 Go 程序。之后，我们将构建一个具有作业状态的长时间运行任务的扩展版本。

# API 的缓存策略

Redis 是一个优秀的开源缓存解决方案，用于缓存高读取配置/信息。它是一个键/值对存储，由于其内存存储，读取速度更快。一个键/值对存储的例子是一个媒体网站，其中一些文章会在主页上固定显示几个小时。

而不是让每个读者都去数据库中检索记录，媒体机构可以使用 Redis 来存储文章内容。这是 Redis 的许多应用之一。

作业状态是临时信息，一旦作业完成并将状态记录到日志存储中，就变得无关紧要。因此，Redis 是实现作业状态缓存的最好选择。我们计划做以下事情：

+   写入作业状态

+   读取作业状态

这两个操作都是由我们的作业服务器执行的，但时间不同。状态可以有以下三种形式：

+   已启动

+   进行中

+   完成

Redis 提供了一组丰富的数据结构来临时存储信息。其中，我们使用简单的`Key:String`来存储作业状态。作业 ID 可以是键，状态是其值。用简单的符号表示，它看起来像这样：

```go
{ '4ed59a6f-24d8-4179-8432-fe4adcdd4f51': 'In Progress'
```

我们可以很容易地使用 Docker 运行 Redis 实例。只需运行一个 Redis 容器并公开端口`6379`：

```go
> docker run --name some-redis -p 6379:6379 -d redis
```

上述命令在本地主机的`6379`端口上运行 Redis 服务器。进程将以守护进程的方式运行，带有`-d`选项。要检查哪个容器是 Redis 的，你可以简单地运行以下 Docker 命令：

```go
> docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                             NAMES
2f0b2b457ed7        redis               "docker-entrypoint.s…"   8 minutes ago       Up 8 minutes        0.0.0.0:6379->6379/tcp                                                            some-redis
c3b2a0a0295d        rabbitmq:3          "docker-entrypoint.s…"   6 weeks ago         Up 11 hours         4369/tcp, 0.0.0.0:5672->5672/tcp, 5671/tcp, 25672/tcp, 0.0.0.0:15672->15672/tcp   rabbitmq-server
```

`redis-cli`是一个可以用来快速检查 Redis 服务器的工具。你不需要单独安装它。你的 Redis Docker 实例已经内置了它。你只需要对 Redis 容器执行一个命令，就像这样：

```go
> docker exec -i -t some-redis redis-cli
127.0.0.1:6379>
```

你可以使用以下命令获取 Redis 中存储的所有可用键：

```go
 127.0.0.1:6379> KEYS *
```

你也可以在 CLI 中设置一个带有过期日期的键值，就像这样：

```go
127.0.0.1:6379> SET topic async
OK
```

上述命令将一个名为`topic`的键设置为`async`。服务器在成功插入后会返回`OK`。CLI 易于使用，但在大多数情况下，你也可以从应用程序中访问 Redis。在下一节中，我们将学习如何从 Go 程序中这样做。

# go-redis，用于与 Redis 通信的 Go 客户端

有一个广泛使用的类型安全的 Go 客户端，用于与 Redis 服务器通信，称为`go-redis`。我们可以通过创建一个类似于 RabbitMQ 的客户端来连接到 Redis 服务器。让我们看看执行此操作的步骤：

1.  首先，我们需要创建一个名为`redisIntro`的简单项目，以展示其基本用法：

```go
mkdir -p $GOPATH/src/github.com/git-user/chapter9/redisIntro
```

1.  使用`dep`工具从项目的根目录初始化必要的依赖项并安装`go-redis`包：

```go
dep init
dep ensure --add "github.com/go-redis/redis"
```

1.  现在，创建一个小的客户端，调用默认的`PING`命令并返回`PONG`：

```go
touch $GOPATH/src/github.com/git-user/chapter9/redisIntro/main.go
```

1.  可以使用`go-redis`包中的`redis.NewClient`方法创建一个客户端：

```go
package main

import (
    "fmt"

    "github.com/go-redis/redis"
)

func main() {
    client := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password set
        DB:       0,  // use default DB
    })

    pong, _ := client.Ping().Result() // Ignoring error
    fmt.Println(pong)
}
```

1.  Redis 客户端可以在服务器上执行命令。其中一个命令是`PING`。`SET`或`GET`命令的工作方式相同。现在，让我们运行程序：

```go
> go run $GOPATH/src/github.com/git-user/chapter9/redisIntro/
main.go
PONG
```

这会打印出`PONG`消息，这是 Redis 服务器给出的响应。有了这个，我们可以确认我们的程序已经成功连接到服务器，并且查询工作正常。

# 使用 Redis 进行作业状态缓存

现在我们已经介绍了 Redis 和 Go 的 Redis 客户端，让我们快速添加作业状态功能到我们之前设计的异步 API 中。我们需要对那个项目进行以下更改：

1.  添加一个新的路由以从客户端收集作业 ID

1.  添加一个新的处理程序以从 Redis 获取作业状态

1.  每当有人添加一个新的作业时，我们需要在作业生命周期的每个阶段将状态写入 Redis

我们将创建一个新的项目来实现这个功能。它的结构与我们之前创建的`longRunningTaskV1`类似。代码和文件都是相同的。你可以克隆该项目并将其重命名为`longRunningTaskV2`。

让我们看看实现，包括依赖项安装和我们必须对先前项目进行的修改。我们不会展示完整的代码，以避免任何冗余：

1.  为了确保你有所有必要的依赖项，运行`dep`命令：

```go
dep ensure
```

1.  将 Redis 包添加到缓存中，以便存储/检索作业的状态：

```go
dep ensure --add github.com/go-redis/redis
```

第一个更改是添加一个路由以访问作业状态。状态可以是以下三者之一：

+   `STARTED`

+   `IN PROGRESS`

+   `DONE`

1.  让我们向`JobServer`结构体添加一个名为`redisClient`的新属性。它存储到已启动并运行的 Redis 容器的客户端连接：

```go
vi $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV2
/handlers.go
```

1.  添加 Redis 包并修改其结构：

```go
import (
    ...
    "github.com/go-redis/redis"
)

// JobServer holds handler functions
type JobServer struct {
    Queue       amqp.Queue
    Channel     *amqp.Channel
    Conn        *amqp.Connection
    redisClient *redis.Client
}
```

1.  现在，添加一个处理函数，该函数接受一个`UUID`作为参数，并通过从 Redis 获取作业状态来构造响应：

```go
...
func (s *JobServer) statusHandler(w http.ResponseWriter,
r *http.Request) {
    queryParams := r.URL.Query()
    // fetch UUID from query
    uuid := queryParams.Get("uuid")
    w.Header().Set("Content-Type", "application/json")
    jobStatus := s.redisClient.Get(uuid)
    status := map[string]string{"uuid": uuid, "status":
     jobStatus.Val()}
    response, err := json.Marshal(status)
    handleError(err, "Cannot create response for client")
    w.Write(response)
}
```

此处理程序使用 Redis 客户端的`Get`函数从 Redis 服务器获取键的值。在此之后，将 HTTP JSON 响应发送回客户端。

1.  现在，更改`main.go`文件并添加一个新的路由：

```go

import (
    ...
    "github.com/go-redis/redis" // Add redis import
)

func main() {
    jobServer := getServer(queueName)

    // Create a client and attach to job server
    jobServer.redisClient = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password set
        DB:       0,  // use default DB
    })

    ...
    router := mux.NewRouter()
    // Attach handlers
    router.HandleFunc("/job/database", jobServer.asyncDBHandler)
    router.HandleFunc("/job/mail", jobServer.asyncMailHandler)
    router.HandleFunc("/job/callback",
     jobServer.asyncCallbackHandler)

    // Add a new route here
    router.HandleFunc("/job/status", jobServer.statusHandler)
}
```

这导入了 Redis 包并创建了一个新的 Redis 客户端。它还添加了一个新的路由来收集客户端的 UUID 作业字符串。新的路由是`"job/status"`，我们将新创建的处理程序`statusHandler`附加到它。

1.  我们可以通过 API 调用获取工作的状态，但待办功能是在每次执行新工作项时在 Redis 中写入工作状态。为此，我们必须稍微修改我们的工作函数。我们将修改的文件如下：

```go
vi $GOPATH/src/github.com/git-user/chapter9/longRunningTaskV2
/worker.go
```

在这里，我们应该修改工作结构体，使其保持一个额外的 Redis 连接，以便可以在缓存中写入工作的状态。我们的计划是将工作 ID 作为键，将状态作为值。

1.  将 Redis 包添加到导入`NewClient`：

```go
import (
    ...
    "github.com/go-redis/redis"
}
```

1.  修改`Worker`结构体以添加`redisClient`。这将保持对 Redis 服务器的新连接：

```go
// Workers do the job. It holds connections

type Workers struct {
    conn        *amqp.Connection
    redisClient *redis.Client
}
```

1.  在`run`函数中，创建一个具体的 Redis 客户端连接：

```go
func (w *Workers) run() {
    ...
    // Create a new connection
    w.redisClient = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password set
        DB:       0,  // use default DB
    })
    ...
}
```

1.  修改工作函数以添加状态消息。例如，让我们看看`dbWork`：

```go
...
func (w *Workers) dbWork(job models.Job) {
    result := job.ExtraData.(map[string]interface{})
    w.redisClient.Set(job.ID.String(), "STARTED", 0)
    log.Printf("Worker %s: extracting data..., JOB: %s",
     job.Type, result)
    w.redisClient.Set(job.ID.String(), "IN PROGRESS", 0)
    time.Sleep(2 * time.Second)
    log.Printf("Worker %s: saving data to database..., JOB: %s",
     job.Type, job.ID)
    w.redisClient.Set(job.ID.String(), "DONE", 0)
}
...
```

我们正在将消息写入 Redis 键，其中消息作为值。当过程转移到下一阶段时，相同的键被覆盖。这是通过调用 Redis 的`redisClient.Set()`函数实现的。

如果你想知道为什么提供了第三个参数，那是因为它是 Redis 服务器上键的过期时间。我们也可以设置一个只存在一段时间的关键。现在，我们希望持久化我们的键，所以过期设置为`zero`，这意味着在 Redis 中没有过期。

我们可以将相同的过程应用于其他两个工作函数，即`callbackWork`和`emailWork`。

现在，是时候测试我们的新特性了：

1.  构建`longRunningTaskV2`项目并使用 curl 调用一个工作项。现在，使用我们添加的新端点找到该工作的状态：

```go
go build .

./longRunningTaskV2
```

1.  创建一个新的工作项，如下所示：

```go
curl -X GET http://localhost:8000/job/database\?client_time\=1569177495
```

1.  这将返回以下 JSON，其中包含工作详情：

```go
{"uuid":"07050695-ce75-4ae8-99d3-2ab068cafe9d","type":"A","extra_data":{"client_time":"2019-09-23T00:08:15+05:30"}}
```

1.  现在，我们可以使用`uuid`找到工作的状态：

```go
curl -X GET http://localhost:8000/job/status\?uuid\=07050695-ce75-4ae8-99d3-2ab068cafe9d
```

1.  这将返回以下状态：

```go
{"uuid":"07050695-ce75-4ae8-99d3-2ab068cafe9d","status":"DONE"}
```

这条消息根据客户端何时调用 API 而变化。但它是一种向客户端透明地提供异步工作状态的方法。

还有一种 API 类别实现了异步行为。这被称为事件驱动 API。服务器和客户端可以监听广播的事件，而不是明确请求它们。这种方法与传统异步实现不同。我们将在下一节中查看这一点。

# 事件驱动 API

我们之前解释的策略是请求/响应协议的实例，其中客户端通过 API 调用执行工作。还有许多其他类似架构，例如事件驱动 API，其中系统生成一系列事件，其他系统可以监听并从中接收更新。为了客户端接收事件，他们应该订阅。

这与某些语言中的回调类似，例如 JavaScript，其中事件循环持续运行并收集事件。这种方法适用于非阻塞客户端和服务器。

一个简单的例子包括客户端将 HTTP 端点注册到 API。每当有有用的信息可用时，服务器可以触发 API 作为事件。以下是一些实际例子：

+   一个向订阅客户端（例如，手机）发送一系列事件的气象站

+   亚马逊的**简单通知服务（SNS**）向端点发布消息

+   一个注册到 API 的 Slack webhook 以获取事件；例如，代码管道失败

实现事件驱动架构的一些协议如下：

+   发布/订阅

+   WebSocket 通信

+   Webhooks/ REST hooks

+   服务器推送（SSE）

根据实际用例，这些协议被用于不同的地方。我们将在第十一章使用微服务扩展我们的 REST API 中简要讨论发布/订阅。在那里，我们将学习如何构建事件驱动系统，从另一方消费事件，以及更多。

# 摘要

在本章中，我们介绍了异步 API。首先，我们解释了同步 API 和异步 API 之间的关键区别。然后，我们学习了多个 API 请求如何导致服务的扇入或扇出。

之后，我们介绍了一个名为 RabbitMQ 的队列系统。队列可以保存作业并允许服务器处理它们。我们学习了如何创建队列并将作业写入其中。我们还创建了一些可以从队列中提取作业并处理它们的 RabbitMQ 客户端。

我们还设计了一个具有多个工作者和队列的长运行任务。工作者始终监听队列并接受工作。我们定义了三种类型的工作者：数据库（DB）、电子邮件（Email）和回调（Callback）。

Redis 是一个存储键/值对的内存数据库。我们可以将其用作缓存来存储作业状态。我们扩展了我们的长运行任务，通过在 Redis 中存储作业状态来添加状态信息。

最后，我们介绍了事件驱动 API，并了解到，使用发布/订阅和 WebSocket，我们可以在客户端和服务器之间设置事件驱动的数据交换。

在下一章中，我们将探讨 GraphQL 的基础知识，以及如何在 Go 中编写 GraphQL 客户端和服务器示例。
