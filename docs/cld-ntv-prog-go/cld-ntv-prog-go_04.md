# 第四章：使用消息队列的异步微服务架构

在过去的两章中，您学习了如何使用 Go 编程语言构建基于 REST 的微服务。REST 架构风格既简单又灵活，这使其成为许多用例的绝佳选择。然而，基于 HTTP 构建的 REST 架构中的所有通信都将遵循客户端/服务器模型，进行请求/回复事务。在某些用例中，这可能是有限制的，其他通信模型可能更适合。

在本章中，我们将介绍发布/订阅通信模型，以及您需要实现它的技术。通常，发布/订阅架构需要一个中央基础设施组件——消息代理。在开源世界中，有许多不同的消息代理实现；因此，在本章中，我们将介绍两种我们认为最重要的消息代理——**RabbitMQ**和**Apache Kafka**。两者都适用于特定的用例；您将学习如何设置这两种消息代理，如何连接您的 Go 应用程序，以及何时应该使用其中一种。

然后，我们将向您展示如何利用这些知识来扩展您在前几章中工作的事件管理微服务，以便在发生重要事件时发布事件。这使我们能够实现第二个微服务来监听这些事件。您还将了解通常与异步通信一起使用的高级架构模式，例如*事件协作*和*事件溯源*，以及如何（以及何时）在应用程序中使用它们。

在本章中，我们将涵盖以下主题：

+   发布/订阅架构模式

+   事件协作

+   事件溯源

+   使用 RabbitMQ 的 AMQP

+   Apache Kafka

# 发布/订阅模式

发布/订阅模式是一种通信模式，是请求/回复模式的替代方案。与客户端（发出请求）和服务器（回复该请求）不同，发布/订阅架构由发布者和订阅者组成。

每个发布者都可以发出消息。发布者实际上并不关心谁收到了这些消息。这是订阅者的问题；每个订阅者可以订阅某种类型的消息，并在发布者发布给定类型的消息时得到通知。反过来，每个订阅者并不关心消息实际来自哪里。

![](img/e967a901-db08-4ba1-a019-ab96d69ff7bc.png)

请求/回复和发布/订阅通信模式

实际上，许多发布/订阅架构都需要一个中央基础设施组件——消息代理。发布者在消息代理上发布消息，订阅者在消息代理上订阅消息。然后，代理的主要任务之一是将发布的消息路由到对它们感兴趣的订阅者。

通常，消息将被路由到基于主题的方式。这意味着每个发布者都为发布的消息指定一个主题（主题通常只是一个字符串标识符，例如`user.created`）。每个订阅者也将订阅特定的主题。通常，代理还允许订阅者使用通配符表达式（例如`user.*`）订阅整个主题集。

与请求/回复相比，发布/订阅模式带来了一些明显的优势：

+   发布者和订阅者之间的耦合非常松散。甚至它们彼此之间都不知道。

+   发布/订阅架构非常灵活。可以添加新的订阅者（因此扩展现有流程）而无需修改发布者。反之亦然；可以添加新的发布者而无需修改订阅者。

+   如果消息由消息代理路由，您还会获得弹性。通常，消息代理会将所有消息存储在队列中，直到它们被订阅者处理。如果订阅者变得不可用（例如由于故障或有意关闭），本应路由到该订阅者的消息将排队，直到订阅者再次可用。

+   通常，您还会在协议级别获得消息代理的某种可靠性保证。例如，RabbitMQ 通过要求每个订阅者确认接收到的消息来保证*可靠传递*。只有在消息被确认后，代理才会从队列中删除消息。如果订阅者应该失败（例如，由于断开连接），当消息已经被传递但尚未被确认时，消息将被放回消息队列中。如果另一个订阅者监听同一消息队列，消息可能会被路由到该订阅者；否则，它将保留在队列中，直到订阅者再次可用。

+   您可以轻松扩展。如果对于单个订阅者来说发布了太多消息以有效处理它们，您可以添加更多订阅者，并让消息代理负载平衡发送给这些订阅者的消息。

当然，引入消息代理这样的中心基础设施组件也带来了自己的风险。如果做得不对，您的消息代理可能会成为单点故障，导致整个应用程序在其失败时崩溃。在生产环境中引入消息代理时，您应该采取适当的措施来确保高可用性（通常是通过集群和自动故障转移）。

如果您的应用程序在云环境中运行，您还可以利用云提供商提供的托管消息排队和传递服务之一，例如 AWS 的**简单队列服务**（**SQS**）或 Azure 服务总线。

在本章中，您将学习如何使用两种最流行的开源消息代理——RabbitMQ 和 Apache Kafka。在第八章中，*AWS 第二部分-S3、SQS、API 网关和 DynamoDB*，您将了解有关 AWS SQS 的信息。

# 介绍预订服务

在这一部分，我们将首先使用 RabbitMQ 实现发布/订阅架构。为此，我们需要向我们的架构添加新的微服务——预订服务将处理事件的预订。其责任将包括确保事件不会被过度预订。为此，它将需要了解现有的事件和位置。为了实现这一点，我们将修改**EventService**，以便在创建位置或事件时发出事件（是的，术语很混乱——确保不要将*发生了某事的通知*类型的事件与*Metallica 在这里演出*类型的事件弄混）。**BookingService**然后可以监听这些事件，并在有人为这些事件之一预订票时自己发出事件。

![](img/5e8ba22f-e4f5-4afe-a62e-a02471ec46b7.png)

我们的微服务概述以及它们将发布和订阅的事件

# 事件协作

事件协作描述了一个与事件驱动的发布/订阅架构很好配合的架构原则。

考虑以下示例，使用常规的请求/响应通信模式——用户请求预订服务为某个事件预订门票。由于事件由另一个微服务（**EventService**）管理，因此**BookingService**需要从**EventService**请求有关事件及其位置的信息。只有这样**BookingService**才能检查是否还有座位可用，并将用户的预订保存在自己的数据库中。此交易所需的请求和响应如下图所示：

![](img/33a76190-6cbc-46dc-a92c-74f6f5cf3070.png)

请求和响应

现在，考虑在发布/订阅架构中相同的场景，其中**BookingService**和**EventService**使用事件进行集成：每当**EventService**中的数据发生变化时，它会发出一个事件（例如*创建了一个新位置*，*创建了一个新事件*，*更新了一个事件*等）。

现在，**BookingService**可以监听这些事件。它可以构建自己的所有当前存在的位置和事件的数据库。现在，如果用户请求为特定事件预订新的预订，**BookingService**可以简单地使用自己本地数据库中的数据，而无需从另一个服务请求此数据。请参考以下图表，以进一步说明这个原则：

![](img/42e36988-e61f-42cd-a54b-9e1133e416b3.png)

使用自己本地数据库中的 BookingService

这是事件协作架构的关键点。在前面的图表中，一个服务几乎永远不需要查询另一个服务的数据，因为它通过监听其他服务发出的事件已经知道了它需要知道的一切。

显然，这种架构模式与发布/订阅非常配合。在前面的例子中，**EventService**将是发布者，而**BookingService**（可能还有其他服务）将是订阅者。当然，人们可能会对这个原则必然导致两个服务存储冗余数据感到不安。然而，这并不一定是件坏事——因为每个服务都不断监听其他服务发出的事件，整个数据集最终可以保持一致。此外，这增加了系统的整体弹性；例如，如果事件服务突然发生故障，**BookingService**仍然可以正常运行，因为它不再依赖事件服务的工作。

# 使用 RabbitMQ 实现发布/订阅

在接下来的部分，您将学习如何实现基本的发布/订阅架构。为此，我们将看一下**高级消息队列协议**（**AMQP**）及其最流行的实现之一，RabbitMQ。

# 高级消息队列协议

在协议级别上，RabbitMQ 实现了 AMQP。在开始使用 RabbitMQ 之前，让我们先看一下 AMQP 的基本协议语义。

AMQP 消息代理管理两种基本资源——**交换**和**队列**。每个发布者将其消息发布到一个交换中。每个订阅者消费一个队列。AMQP 代理负责将发布到交换中的消息放入相应的队列中。消息发布到交换后的去向取决于**交换类型**和称为**绑定**的路由规则。AMQP 有三种不同类型的交换：

+   **Direct exchanges**: 消息以给定的主题（在 AMQP 中称为**路由键**）发布，这是一个简单的字符串值。可以定义直接交换和队列之间的绑定，以精确匹配该主题。

+   **Fanout exchanges**: 消息通过绑定连接到扇出交换机的所有队列。消息可以有路由键，但会被忽略。每个绑定的队列将接收发布在扇出交换机中的所有消息。

+   **主题交换**：这与直接交换类似。但是，现在队列是使用消息的路由键必须匹配的模式绑定到交换。主题交换通常假定路由键使用句点字符`'.'`进行分段。例如，您的路由键可以遵循`"<entityname>.<state-change>.<location>"`模式（例如，`"event.created.europe"`）。现在可以创建包含通配符的队列绑定，使用`'*'`或`'#'`字符。`*`将匹配任何单个路由键段，而`#`将匹配任意数量的段。因此，对于前面的示例，有效的绑定可能如下：

+   `event.created.europe`（显然）

+   `event.created.*`（每当在世界的任何地方创建事件时都会收到通知）

+   `event.#`（每当在世界的任何地方对事件进行任何更改时都会收到通知）

+   `event.*.europe`（每当在欧洲对事件进行任何更改时都会收到通知）

下一个图表显示了一个可能的示例交换和队列拓扑结构。在这种情况下，我们有一个发布消息的服务**EventService**。我们有两个队列，消息将被路由到这两个队列中。第一个队列**evts_booking**将接收与事件的任何更改相关的所有消息。第二个队列**evts_search**将只接收关于新事件创建的消息。请注意，**evts_booking**队列有两个订阅者。当两个或更多订阅者订阅同一个队列时，消息代理将轮流将消息分发给其中一个订阅者。

![](img/cc668950-bec2-43b7-beb1-5ca489f7e80d.png)

消息代理将消息轮流显示给其中一个订阅者

重要的是要注意，整个 AMQP 拓扑（即所有交换和队列以及它们如何相互绑定）不是由代理定义的，而是由发布者和消费者自己定义的。AMQP 指定了客户端可以使用的几种方法来声明它们需要的交换和队列。例如，发布者通常会使用`exchange.declare`方法来断言它想要发布的交换实际上存在（如果之前不存在，代理将创建它）。另一方面，订阅者可能会使用`queue.declare`和`queue.bind`方法来声明它想要订阅的队列，并将其绑定到一个交换。

有多个实现 AMQP 的开源消息代理。其中最流行的之一（也是我们在本章中将要使用的）是 RabbitMQ 代理，这是一个由**Pivotal**开发并在**Mozilla Public License**下提供的开源 AMQP 代理。其他实现 AMQP 的消息代理包括**Apache QPID**（[`qpid.apache.org`](https://qpid.apache.org)）和**Apache ActiveMQ**（[`activemq.apache.org`](http://activemq.apache.org)）。

虽然在这个例子中我们将使用 RabbitMQ，但本章中编写的代码应该适用于所有类型的 AMQP 实现。

# 使用 Docker 快速启动 RabbitMQ

在构建我们的发布/订阅架构之前，您需要在开发环境中设置一个正在运行的 RabbitMQ 消息代理。使用官方的 Docker 镜像是开始使用 RabbitMQ 的最简单方法。

对于本例，我们将假设您的本地机器上已经安装了 Docker。请查看官方安装说明，了解如何在您的操作系统上安装 Docker：[`docs.docker.com/engine/installation`](https://docs.docker.com/engine/installation)。

您可以使用以下命令在命令行上启动一个新的 RabbitMQ 代理：

```go
$ docker run --detach \ 
    --name rabbitmq \ 
    -p 5672:5672 \ 
    -p 15672:15672 \ 
    rabbitmq:3-management 
```

上述命令将在您的机器上创建一个名为`rabbitmq`的新容器。为此，Docker 将使用`rabbitmq:3-management`镜像。该镜像包含了 RabbitMQ 3 的最新版本（在撰写本文时为 3.6.6）和管理 UI。`-p 5672:5672`标志将指示 Docker 将 TCP 端口`5672`（这是 AMQP 的 IANA 分配的端口号）映射到您的`localhost`地址。`-p 15672:15672`标志将对管理用户界面执行相同的操作。

启动容器后，您将能够在浏览器中打开到`amqp://localhost:5672`的 AMQP 连接，并在`http://localhost:15672`中打开管理 UI。

当您在 Windows 上使用 Docker 时，您需要用本地 Docker 虚拟机的 IP 地址替换 localhost。您可以使用以下命令在命令行上确定此 IP 地址：`$ docker-machine ip default`。

无论您是使用 docker-machine 还是本地 Docker 安装，RabbitMQ 用户界面应该看起来与以下截图非常相似：

![](img/6075ceff-4af4-4e70-bfa2-de38dd0501c0.png)

RabbitMQ 的管理用户界面

在浏览器中打开管理界面（`http://localhost:15672`或您的 docker-machine IP 地址）。RabbitMQ 镜像提供了一个默认的 guest 用户，其密码也是`guest`。在生产中运行 RabbitMQ 时，这当然是您应该更改的第一件事。对于开发目的，这样做就可以了。

# 高级 RabbitMQ 设置

上一节中描述的基于 Docker 的设置可以让您快速入门，并且（经过一些调整）也适用于生产设置。如果您不想为消息代理使用 Docker，您还可以从软件包存储库在大多数常见的 Linux 发行版上安装 RabbitMQ。例如，在 Ubuntu 和 Debian 上，您可以使用以下命令安装 RabbitMQ：

```go
$ echo 'deb http://www.rabbitmq.com/debian/ testing main' | \ 
    sudo tee /etc/apt/sources.list.d/rabbitmq.list 
$ wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc | \ 
    sudo apt-key add - 
$ apt-get update 
$ apt-get install -y rabbitmq-server 
```

类似的命令也适用于**CentOS**和**RHEL**：

```go
$ rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc 
$ yum install rabbitmq-server-3.6.6-1.noarch.rpm 
```

对于生产设置，您可能希望考虑设置 RabbitMQ 作为集群，以确保高可用性。请查看官方文档[`www.rabbitmq.com/clustering.html`](http://www.rabbitmq.com/clustering.html)了解如何设置 RabbitMQ 集群的更多信息。

# 使用 Go 连接 RabbitMQ

要连接到 RabbitMQ 代理（或者说是任何 AMQP 代理），我们建议您使用`github.com/streadway/amqp`库（这是事实上的标准 Go 库，用于 AMQP）。让我们从安装库开始：

```go
$ go get -u github.com/streadway/amqp
```

然后，您可以通过将库导入到您的代码中来开始。使用`amqp.Dial`方法打开一个新连接：

```go
import "github.com/streadway/amqp" 

func main() { 
  connection, err := amqp.Dial("amqp://guest:guest@localhost:5672") 
  if err != nil { 
    panic("could not establish AMQP connection: " + err.Error()) 
  } 

  defer connection.Close() 
} 
```

在这种情况下，`"amqp://guest:guest@localhost:5672"`是您的 AMQP 代理的 URL。请注意，用户凭据嵌入到 URL 中。`amqp.Dial`方法在成功时返回连接对象，否则返回`nil`和错误（与 Go 中一样，请确保您实际上检查了此错误）。此外，在不再需要连接时不要忘记使用`Close()`方法关闭连接。

当然，通常不建议将连接详细信息（更不用说凭据）硬编码到您的应用程序中。记住您学到的关于十二要素应用程序的知识，让我们引入一个环境变量`AMQP_URL`，我们可以使用它来动态配置 AMQP 代理：

```go
import "github.com/streadway/amqp" 
import "os" 

func main() { 
  amqpURL := os.Getenv("AMQP_URL"); 
  if amqpURL == "" { 
    amqpURL = "amqp://guest:guest@localhost:5672" 
  } 

  connection, err := amqp.Dial(amqpURL) 
  // ... 
} 
```

在 AMQP 中，大多数操作不是直接在连接上进行的，而是在通道上进行的。通道用于在一个实际的 TCP 连接上*多路复用*多个虚拟连接。

通道本身不是线程安全的。在 Go 中，我们需要记住这一点，并注意不要从多个 goroutine 访问同一个通道。但是，使用多个通道，每个通道只被一个线程访问，是完全安全的。因此，当有疑问时，最好创建一个新通道。

继续在现有连接上创建一个新通道：

```go
connection, err := amqp.Dial(amqpURL) 
if err != nil { 
  panic("could not establish AMQP connection: " + err.Error()) 
} 

channel, err := connection.Channel() 
if err != nil { 
  panic("could not open channel: " + err.Error()) 
} 
```

现在我们可以使用这个通道对象进行一些实际的 AMQP 操作，例如发布消息和订阅消息。

# 发布和订阅 AMQP 消息

在深入研究 MyEvents 微服务架构之前，让我们看一下我们可以使用的基本 AMQP 方法。为此，我们将首先构建一个小的示例程序，该程序能够向交换发布消息。

打开通道后，消息发布者应声明要发布消息的交换。为此，您可以在通道对象上使用`ExchangeDeclare()`方法：

```go
err = channel.ExchangeDeclare("events", "topic", true, false, false, false, nil) 
if err != nil { 
  panic(err) 
} 
```

正如您所看到的，`ExchangeDeclare`需要相当多的参数。这些如下所示：

+   交换名称

+   交换类型（请记住 AMQP 知道`direct`，`fanout`和`topic`交换）

+   `durable`标志将导致交换在代理重新启动时保持声明状态

+   `autoDelete`标志将导致交换在声明它的通道关闭时被删除

+   `internal`标志将阻止发布者将消息发布到此队列中

+   `noWait`标志将指示`ExchangeDeclare`方法不等待来自代理的成功响应

+   `args`参数可能包含具有附加配置参数的映射

声明交换后，您现在可以发布一条消息。为此，您可以使用通道的`Publish()`方法。发出的消息将是您需要首先实例化的`amqp.Publishing`结构的实例：

```go
message := amqp.Publishing { 
  Body: []byte("Hello World"),
} 
```

然后，使用`Publish()`方法发布您的消息：

```go
err = channel.Publish("events", "some-routing-key", false, false, message) 
if err != nil { 
  panic("error while publishing message: " + err.Error()) 
} 
```

`Publish()`方法接受以下参数：

+   要发布到的交换的名称

+   消息的路由键

+   `mandatory`标志将指示代理确保消息实际上被路由到至少一个队列中

+   `immediate`标志将指示代理确保消息实际上被传递给至少一个订阅者

+   `msg`参数包含要发布的实际消息

对于发布/订阅架构，发布者不需要知道谁订阅其发布的消息，显然`mandatory`和`immediate`标志不适用，因此在此示例（以及所有后续示例）中，我们将它们设置为 false。

您现在可以运行此程序，它将连接到您的本地 AMQP 代理，声明一个交换，并发布一条消息。当然，这条消息不会被路由到任何地方并消失。为了实际处理它，您将需要一个订阅者。

继续创建第二个 Go 程序，其中您连接到 AMQP 代理并创建一个新的通道，就像在前一节中一样。但是，现在，不是声明一个交换并发布一条消息，让我们声明一个队列并将其绑定到该交换：

```go
_, err = channel.QueueDeclare("my_queue", true, false, false, false, nil) 
if err != nil { 
  panic("error while declaring the queue: " + err.Error()) 
} 

err = channel.QueueBind("my_queue", "#", "events", false, nil) 
if err != nil { 
  panic("error while binding the queue: " + err.Error())
} 
```

声明并绑定队列后，您现在可以开始消费此队列。为此，请使用通道的`Consume()`函数：

```go
msgs, err := channel.Consume("my_queue", "", false, false, false, false, nil) 
if err != nil { 
  panic("error while consuming the queue: " + err.Error()) 
} 
```

`Consume()`方法接受以下参数：

+   要消耗的队列的名称。

+   唯一标识此消费者的字符串。当留空（就像在这种情况下）时，将自动生成唯一标识符。

+   当设置`autoAck`标志时，接收到的消息将自动确认。当未设置时，您需要在处理接收到的消息后显式确认消息，使用接收到的消息的`Ack()`方法（请参阅以下代码示例）。

+   当设置`exclusive`标志时，此消费者将是唯一被允许消费此队列的消费者。当未设置时，其他消费者可能会监听同一队列。

+   `noLocal`标志指示代理不应将在同一通道上发布的消息传递给此消费者。

+   `noWait`标志指示库不等待来自代理的确认。

+   `args`参数可能包含具有附加配置参数的映射。

在这个例子中，`msgs`将是一个通道（这次是一个实际的 Go 通道，而不是一个 AMQP 通道）的`amqp.Delivery`结构。为了从队列中接收消息，我们可以简单地从该通道中读取值。如果要连续读取消息，最简单的方法是使用`range`循环：

```go
for msg := range msgs { 
  fmt.Println("message received: " + string(msg.Body)) 
  msg.Ack(false) 
} 
```

请注意，在前面的代码中，我们使用`msg.Ack`函数显式确认消息。这是必要的，因为我们之前将`Consume()`函数的`autoAck`参数设置为 false。

显式确认消息具有重要目的——如果您的消费者在接收和确认消息之间由于任何原因失败，消息将被放回队列，然后重新传递给另一个消费者（或者如果没有其他消费者，则留在队列中）。因此，消费者应该只在完成处理消息时确认消息。如果消息在消费者实际处理之前就被确认（这就是`autoAck`参数会导致的情况），然后消费者意外死机，消息将永远丢失。因此，显式确认消息是使系统具有弹性和容错性的重要步骤。

# 构建事件发射器

在前面的例子中，我们使用 AMQP 通道从发布者向订阅者发送简单的字符串消息。为了使用 AMQP 构建实际的发布/订阅架构，我们需要传输更复杂的带有结构化数据的消息。一般来说，每个 AMQP 消息只是一串字节。为了提交结构化数据，我们可以使用序列化格式，比如 JSON 或 XML。此外，由于 AMQP 不限于 ASCII 消息，我们还可以使用二进制序列化协议，比如`MessagePack`或`ProtocolBuffers`。

无论您决定使用哪种序列化格式，您都需要确保发布者和订阅者都了解序列化格式和消息的实际内部结构。

关于序列化格式，我们将在本章中选择简单的 JSON 序列化格式。它被广泛采用；使用 Go 标准库和其他编程语言（这一点很重要——尽管在本书中我们专门致力于 Go，但在微服务架构中，有许多不同的应用运行时是很常见的）轻松地进行序列化和反序列化消息。

我们还需要确保发布者和订阅者都知道消息的结构。例如，一个`LocationCreated`事件可能有一个`name`属性和一个`address`属性。为了解决这个问题，我们将引入一个共享库，其中包含所有可能事件的结构定义，以及 JSON（反）序列化的说明。然后，这个库可以在发布者和所有订阅者之间共享。

首先在您的 GOPATH 中创建`todo.com/myevents/contracts`目录。我们将描述的第一种事件类型是`EventCreatedEvent`事件。当创建新事件时，此消息将由事件服务发布。让我们在新创建的包的`event_created.go`文件中将此事件定义为一个结构：

```go
package contracts 

import "time" 

type EventCreatedEvent struct { 
  ID         string    `json:"id"` 
  Name       string    `json:"id"` 
  LocationID string    `json:"id"` 
  Start      time.Time `json:"start_time"` 
  End        time.Time `json:"end_time"` 
} 
```

此外，我们需要为每个事件生成一个主题名称（在 RabbitMQ 中，主题名称也将用作消息的路由键）。为此，请向您新定义的结构添加一个新方法——`EventName()`：

```go
func (e *EventCreatedEvent) EventName() string { 
  return "event.created" 
} 
```

我们现在可以使用 Go 接口来定义一个通用的事件类型。这种类型可以用来强制每种事件类型实际上都实现了一个`EventName()`方法。由于事件发布者和事件订阅者以后也将被用于多个服务，我们将事件接口代码放入`todo.com/myevents/lib/msgqueue`包中。首先创建包目录和一个新文件`event.go`：

```go
package msgqueue 

type Event interface { 
  EventName() string 
} 
```

当然，我们的示例应用程序使用的事件不仅仅是`EventCreatedEvent`。例如，我们还有一个`LocationCreatedEvent`和一个`EventBookedEvent`。由于在打印中显示它们的所有实现会相当重复，我们希望在本章的示例文件中查看它们。

让我们现在继续构建一个事件发射器，它可以实际将这些消息发布到 AMQP 代理。由于我们将在本章的后面部分探索其他消息代理，因此我们将首先定义任何事件发射器应该满足的接口。为此，在之前创建的`msgqueue`包中创建一个`emitter.go`文件，内容如下：

```go
package msgqueue 

type EventEmitter interface { 
  Emit(event Event) error 
} 
```

此接口描述了所有事件发射器实现需要满足的方法（实际上只有一个方法）。让我们继续创建一个`todo.com/myevents/lib/msgqueue/amqp`子包，其中包含一个`emitter.go`文件。该文件将包含`AMQPEventEmitter`的结构定义。

考虑以下代码示例：

```go
package amqp 

import "github.com/streadway/amqp" 

type amqpEventEmitter struct { 
  connection *amqp.Connection 
} 
```

请注意`amqpEventEmitter`类型声明为包私有，因为它使用小写名称声明。这将阻止用户直接实例化`amqpEventEmitter`类型。为了正确实例化，我们将提供一个构造方法。

接下来，让我们添加一个`setup`方法，我们可以用来声明此发布者将要发布到的交换机：

```go
func (a *amqpEventEmitter) setup() error {
   channel, err := a.connection.Channel()
   if err != nil {
     return err
   }

   defer channel.Close() 

  return channel.ExchangeDeclare("events", "topic", true, false, false, false, nil) 
 } 
```

您可能想知道为什么我们在此方法中创建了一个新的 AMQP 通道，并在声明交换机后立即关闭它。毕竟，我们可以在以后重用相同的通道来发布消息。我们稍后会解决这个问题。

继续添加一个构造函数`NewAMQPEventEmitter`，用于构建此结构的新实例：

```go
func NewAMQPEventEmitter(conn *amqp.Connection) (EventEmitter, error) { 
  emitter := &amqpEventEmitter{ 
    connection: conn, 
  } 

  err := emitter.setup()
   if err != nil { 
    return nil, err 
  } 

  return emitter, nil 
} 
```

现在，到`amqpEventEmitter`事件的实际核心——`Emit`方法。首先，我们需要将作为参数传递给方法的事件转换为 JSON 文档：

```go
import "encoding/json"

 // ...

 func (a *amqpEventEmitter) Emit(event Event) error { 
  jsonDoc, err := json.Marshal(event) 
  if err != nil { 
    return err 
  } 
} 
```

接下来，我们可以创建一个新的 AMQP 通道，并将我们的消息发布到事件交换机中：

```go
func (a *amqpEventEmitter) Emit(event Event) error { 
  // ... 

  chan, err := a.connection.Channel(); 
  if err != nil { 
    return err 
  } 

  defer chan.Close() 

  msg := amqp.Publishing{ 
    Headers:     amqpTable{"x-event-name": event.EventName()}, 
    Body:        jsonDoc, 
    ContentType: "application/json", 
  } 

  return chan.Publish( 
    "events", 
    event.EventName(), 
    false, 
    false, 
    msg 
  ) 
} 
```

请注意，我们使用`amqp.Publishing`的`Headers`字段来将事件名称添加到特殊的消息头中。这将使我们更容易实现事件监听器。

还要注意，在此代码中，我们为每个发布的消息创建了一个新通道。虽然理论上可以重用相同的通道来发布多个消息，但我们需要记住，单个 AMQP 通道不是线程安全的。这意味着从多个 go 协程调用事件发射器的`Emit()`方法可能会导致奇怪和不可预测的结果。这正是 AMQP 通道的问题所在；使用多个通道，多个线程可以使用相同的 AMQP 连接。

接下来，我们可以将新的事件发射器集成到您已经在第二章和第三章中构建的现有事件服务中。首先，在`ServiceConfig`结构中添加一个 AMQP 代理的配置选项：

```go
type ServiceConfig struct { 
  // ... 
  AMQPMessageBroker string `json:"amqp_message_broker"` 
} 
```

这使您可以通过 JSON 配置文件指定 AMQP 代理。在`ExtractConfiguration()`函数中，我们还可以添加一个备用选项，如果设置了环境变量，则可以从中提取此值：

```go
func ExtractConfiguration(filename string) ServiceConfig { 
  // ... 

  json.NewDecoder(file).Decode(&conf) 
  if broker := os.Getenv("AMQP_URL"); broker != "" { 
    conf.AMQPMessageBroker = broker 
  } 

  return conf 
} 
```

现在，我们可以在事件服务的`main`函数中使用此配置选项来构造一个新的事件发射器：

```go
package main 

// ... 
import "github.com/streadway/amqp" 
import msgqueue_amqp "todo.com/myevents/lib/msgqueue/amqp" 

func main() { 
  // ... 

  config := configuration.ExtractConfiguration(*confPath) 
  conn, err := amqp.Dial(config.AMQPMessageBroker) 
  if err != nil { 
    panic(err) 
  } 

  emitter, err := msgqueue_amqp.NewAMQPEventEmitter(conn) 
  if err != nil { 
    panic(err) 
  } 

  // ... 
} 
```

现在，我们可以将此事件发射器传递给`rest.ServeAPI`函数，然后再传递给`newEventHandler`函数：

```go
func ServeAPI(endpoint string, dbHandler persistence.DatabaseHandler, eventEmitter msgqueue.EventEmitter) error { 
  handler := newEventHandler(dbHandler, eventEmitter) 
  // ... 
} 
```

然后，事件发射器可以作为`eventServiceHandler`结构的字段存储：

```go
type eventServiceHandler struct { 
  dbhandler persistence.DatabaseHandler 
  eventEmitter msgqueue.EventEmitter 
} 

func newEventHandler(dbhandler persistence.DatabaseHandler, eventEmitter msgqueue.EventEmitter) *eventServiceHandler { 
  return &eventServiceHandler{ 
    dbhandler: dbhandler, 
    eventEmitter: eventEmitter, 
  } 
} 
```

现在，`eventServiceHandler`持有对事件发射器的引用，您可以在实际的 REST 处理程序中使用它。例如，通过 API 创建新事件时，您可以发出`EventCreatedEvent`。为此，请修改`eventServiceHandler`的`newEventHandler`方法如下：

```go
func (eh *eventServiceHandler) newEventHandler(w http.ResponseWriter, r *http.Request) { 
  id, err := eh.dbhandler.AddEvent(event) 
  if err != nil { 
    // ... 
  } 

  msg := contracts.EventCreatedEvent{ 
    ID: hex.EncodeToString(id), 
    Name: event.Name, 
    LocationID: event.Location.ID, 
    Start: time.Unix(event.StartDate, 0), 
    End: time.Unix(event.EndDate, 0), 
  } 
  eh.eventEmitter.emit(&msg) 

  // ... 
} 
```

# 构建事件订阅者

现在我们可以使用`EventEmitter`在`RabbitMQ`代理上发布事件，我们还需要一种方法来监听这些事件。这将是我们将在本节中构建的`EventListener`的目的。

与之前一样，让我们首先定义所有事件监听器（AMQP 事件监听器是其中之一）应该满足的接口。为此，在`todo.com/myevents/lib/msgqueue`包中创建`listener.go`文件：

```go
package msgqueue 

type EventListener interface { 
  Listen(eventNames ...string) (<-chan Event, <-chan error, error) 
} 
```

这个接口看起来与事件发射器的接口有很大不同。这是因为对事件发射器的每次调用`Emit()`方法只是立即发布一条消息。然而，事件监听器通常会长时间处于活动状态，并且需要在接收到消息时立即做出反应。这反映在我们的`Listen()`方法的设计中：首先，它将接受事件监听器应该监听的事件名称列表。然后返回两个 Go 通道：第一个将用于流式传输事件监听器接收到的任何事件。第二个将包含接收这些事件时发生的任何错误。

首先通过在`todo.com/myevents/lib/msgqueue/amqp`包中创建一个新的`listener.go`文件来构建 AMQP 实现：

```go
package amqp 

import "github.com/streadway/amqp" 

type amqpEventListener struct { 
  connection *amqp.Connection 
  queue      string 
} 
```

类似于事件发射器，继续添加一个`setup`方法。在这个方法中，我们需要声明监听器将要消费的 AMQP 队列：

```go
func (a *ampqEventListener) setup() error { 
  channel, err := a.connection.Channel() 
  if err != nil { 
    return nil 
  } 

  defer channel.Close() 

  _, err := channel.QueueDeclare(a.queue, true, false, false, false, nil) 
  return err 
} 
```

请注意，监听器将要消费的队列的名称可以使用`amqpEventListener`结构的`queue`字段进行配置。这是因为以后，多个服务将使用事件监听器来监听它们的事件，并且每个服务都需要自己的 AMQP 队列。

您可能已经注意到，我们尚未将新声明的队列绑定到事件交换机。这是因为我们还不知道我们实际上需要监听哪些事件（记住`Listen`方法的`events`参数吗？）。

最后，让我们添加一个构造函数来创建新的 AMQP 事件监听器：

```go
func NewAMQPEventListener(conn *amqp.Connection, queue string) (msgqueue.EventListener, error) { 
  listener := &amqpEventListener{ 
    connection: conn, 
    queue:      queue, 
  } 

  err := listener.setup() 
  if err != nil { 
    return nil, err 
  } 

  return listener, nil 
} 
```

有了构建新的 AMQP 事件监听器的可能性，让我们实现实际的`Listen()`方法。首先要做的是使用`eventNames`参数并相应地绑定事件队列：

```go
func (a *amqpEventListener) Listen(eventNames ...string) (<-chan msgqueue.Event, <-chan error, error) { 
  channel, err := a.connection.Channel() 
  if err != nil { 
    return nil, nil, err 
  } 

  defer channel.Close() 

  for _, eventName := range eventNames { 
    if err := channel.QueueBind(a.queue, eventName, "events", false, nil); err != nil { 
      return nil, nil, err 
    } 
  } 
} 
```

接下来，我们可以使用通道的`Consume()`方法从队列中接收消息：

```go
func (a *amqpEventListener) Listen(eventNames ...string) (<-chan msgqueue.Event, <-chan error, error) { 
  // ... 

  msgs, err := channel.Consume(a.queue, "", false, false, false, false, nil) 
  if err != nil { 
    return nil, nil, err 
  } 
} 
```

`msgs`变量现在持有`amqp.Delivery`结构的通道。然而，我们的事件监听器应该返回一个`msgqueue.Event`的通道。这可以通过在我们自己的 goroutine 中消费`msgs`通道，构建相应的事件结构，然后将这些事件发布到我们从这个函数返回的另一个通道中来解决：

```go
func (a *amqpEventListener) Listen(eventNames ...string) (<-chan msgqueue.Event, <-chan error, error) { 
  // ... 

  events := make(chan msgqueue.Event) 
  errors := make(errors) 

  go func() { 
    for msg := range msgs { 
      // todo: Map message to actual event struct 
    } 
  }() 

  return events, errors, nil 
} 
```

现在棘手的部分在于内部 goroutine 中。在这里，我们需要将原始的 AMQP 消息映射到实际的事件结构之一（如之前定义的`EventCreatedEvent`）。

还记得 EventEmitter 在发布事件时向 AMQP 消息添加了额外的`x-event-name`头部吗？现在我们可以使用这个来将这些消息映射回它们各自的结构类型。让我们从 AMQP 消息头中提取事件名称开始：

以下所有代码都放在`Listen`方法的内部`range`循环中。

```go
rawEventName, ok := msg.Headers["x-event-name"] 
if !ok { 
  errors <- fmt.Errorf("msg did not contain x-event-name header") 
  msg.Nack(false) 
  continue 
} 

eventName, ok := rawEventName.(string) 
if !ok { 
  errors <- fmt.Errorf( 
    "x-event-name header is not string, but %t", 
    rawEventName 
  ) 
  msg.Nack(false) 
  continue 
} 
```

前面的代码尝试从 AMQP 消息中读取`x-event-name`头部。由于`msg.Headers`属性基本上是一个`map[string]interface{}`，我们需要一些映射索引和类型断言，直到我们实际使用事件名称。如果接收到不包含所需头部的消息，将向错误通道写入错误。此外，消息将被 nack'ed（简称否定确认），表示经纪人无法成功处理该消息。

知道事件名称后，我们可以使用简单的 switch/case 结构从这个名称创建一个新的事件结构：

```go
var event msgqueue.Event 

switch eventName { 
  case "event.created": 
    event = new(contracts.EventCreatedEvent) 
  default: 
    errors <- fmt.Errorf("event type %s is unknown", eventName) 
    continue 
} 

err := json.Unmarshal(msg.Body, event) 
if err != nil { 
  errors <- err 
  continue 
} 

events <- event 
```

# 构建预订服务

现在我们有了一个事件监听器，我们可以使用它来实现预订服务。它的一般架构将遵循事件服务的架构，因此我们不会过多地详细介绍这个问题。

首先创建一个新的包`todo.com/myevents/bookingservice`，并创建一个新的`main.go`文件：

```go
package main 

import "github.com/streadway/amqp" 
import "todo.com/myevents/lib/configuration" 
import msgqueue_amqp "todo.com/myevents/lib/msgqueue/amqp" 
import "flag" 

func main() { 
  confPath := flag.String("config", "./configuration/config.json", "path to config file") 
  flag.Parse() 
  config := configuration.ExtractConfiguration(*confPath) 

  dblayer, err := dblayer.NewPersistenceLayer(config.Databasetype, config.DBConnection) 
  if err != nil { 
    panic(err) 
  } 

  conn, err := amqp.Dial(config.AMQPMessageBroker) 
  if err != nil { 
    panic(err) 
  } 

  eventListener, err := msgqueue_amqp.NewAMQPEventListener(conn) 
  if err != nil { 
    panic(err) 
  } 
} 
```

这将使用数据库连接和工作事件监听器设置预订服务。现在我们可以使用这个事件监听器来监听事件服务发出的事件。为此，添加一个新的子包`todo.com/myevents/bookingservice/listener`并创建一个新的`event_listener.go\`文件：

```go
package listener 

import "log" 
import "todo.com/myevents/lib/msgqueue" 
import "todo.com/myevents/lib/persistence" 
import "gopkg.in/mgo.v2/bson" 

type EventProcessor struct { 
  EventListener msgqueue.EventListener 
  Database      persistence.DatabaseHandler 
} 

func (p *EventProcessor) ProcessEvents() error { 
  log.Println("Listening to events...") 

  received, errors, err := p.EventListener.Listen("event.created") 
  if err != nil { 
    return err 
  } 

  for { 
    select { 
      case evt := <-received: 
        p.handleEvent(evt) 
      case err = <-errors: 
        log.Printf("received error while processing msg: %s", err) 
    } 
  } 
} 
```

在`ProcessEvents()`函数中，我们调用事件监听器的`Listen`函数来监听新创建的事件。`Listen`函数返回两个通道，一个用于接收消息，一个用于监听期间发生的错误。然后我们将使用一个无限运行的 for 循环和一个 select 语句同时从这两个通道中读取。接收到的事件将传递给`handleEvent`函数（我们仍然需要编写），接收到的错误将简单地打印到标准输出。

让我们继续使用`handleEvent`函数：

```go
func (p *EventProcessor) handleEvent(event msgqueue.Event) { 
  switch e := event.(type) { 
    case *contracts.EventCreatedEvent: 
      log.Printf("event %s created: %s", e.ID, e) 
      p.Database.AddEvent(persistence.Event{ID: bson.ObjectId(e.ID)}) 
    case *contracts.LocationCreatedEvent: 
      log.Printf("location %s created: %s", e.ID, e) 
      p.Database.AddLocation(persistence.Location{ID: bson.ObjectId(e.ID)}) 
    default: 
      log.Printf("unknown event: %t", e) 
  } 
} 
```

这个函数使用类型开关来确定传入事件的实际类型。目前，我们的事件监听器通过将`EventCreated`和`LocationCreated`两个事件存储在它们的本地数据库中来处理这两个事件。

在这个例子中，我们使用了一个共享库`todo.com/myevents/lib/persistence`来管理数据库访问。这仅仅是为了方便。在真实的微服务架构中，各个微服务通常使用完全独立的持久化层，可能构建在完全不同的技术栈上。

在我们的`main.go`文件中，现在可以实例化`EventProcessor`并调用`ProcessEvents()`函数：

```go
func main() { 
  // ... 

  eventListener, err := msgqueue_amqp.NewAMQPEventListener(conn) 
  if err != nil { 
    panic(err) 
  } 

  processor := &listener.EventProcessor{eventListener, dblayer} 
  processor.ProcessEvents() 
} 
```

除了监听事件，预订服务还需要实现自己的 REST API，用户可以用来预订指定事件的门票。这将遵循您已经在第二章和第三章中学到的相同原则，*使用 Rest API 构建微服务*和*保护微服务*。因此，我们将避免详细解释预订服务的 REST API，并只描述要点。您可以在本章的代码示例中找到 REST 服务的完整实现。

在`main.go`文件中，我们需要将`processor.ProcessEvents()`调用移到自己的 go-routine 中。否则，它会阻塞，程序永远不会达到`ServeAPI`方法调用：

```go
func main() { 
  // ... 

  processor := &listener.EventProcessor{eventListener, dblayer} 
  go processor.ProcessEvents() 

  rest.ServeAPI(config.RestfulEndpoint, dbhandler, eventEmitter) 
} 
```

最后，我们将转向实际的请求处理程序。它在`/events/{eventID}/bookings`注册为 POST 请求；它会检查当前为该事件放置了多少预订，并且事件的位置是否仍然有容量可以再预订一次。在这种情况下，它将创建并持久化一个新的预订，并发出一个`EventBooked`事件。查看示例文件以查看完整的实现。

# 事件溯源

使用异步消息传递构建应用程序为应用一些高级架构模式打开了大门，其中之一将在本节中学习。

在使用消息传递、发布/订阅和事件协作时，整个系统状态的每一次变化都以一个事件的形式反映出来，该事件由参与服务中的一个发出。通常，这些服务中的每一个都有自己的数据库，保持对系统状态的自己视图（至少是所需的），并通过不断监听其他服务发布的事件来保持最新。

然而，系统状态的每一次变化都由一个已发布的事件表示，这提供了一个有趣的机会。想象一下，有人记录并保存了任何人发布的每一个事件到事件日志中。理论上（也在实践中），你可以使用这个事件日志来重建整个系统状态，而不必依赖任何其他类型的数据库。

举个例子，考虑以下（小）事件日志：

1.  上午 8:00—用户＃1 名称为爱丽丝被创建

1.  上午 9:00—用户＃2 名称为鲍勃被创建

1.  下午 1:00—用户＃1 被删除

1.  下午 3:00—用户＃2 将名称更改为塞德里克

通过重放这些事件，很容易重建系统在一天结束时的状态——有一个名为塞德里克的用户。然而，还有更多。由于每个事件都有时间戳，你可以重建应用在任何给定时间点的状态（例如，在上午 10:00，你的应用有两个用户，爱丽丝和鲍勃）。

除了点对点恢复，事件溯源还为你提供了系统中发生的一切的完整审计日志。审计日志通常在许多情况下是一个实际的要求，但也使得在出现错误时更容易调试系统。拥有完整的事件日志可以让你在确切的时间点复制系统的状态，然后逐步重放事件，以实际重现特定的错误。

此外，事件日志使各个服务不那么依赖其本地数据库。在极端情况下，你可以完全放弃数据库，并让每个服务在启动时从事件日志中内存重建其整个查询模型。

# 使用 Apache Kafka 实现发布/订阅和事件溯源

在本章的其余部分，我们不会构建自己的事件溯源系统。之前，我们使用 RabbitMQ 来实现服务之间的消息传递。然而，RabbitMQ 只处理消息分发，因此如果你需要一个包含所有事件的事件日志，你需要自己实现它，监听所有事件并持久化它们。你还需要自己处理事件重放。

Apache Kafka 是一个分布式消息代理，还附带一个集成的事务日志。它最初是由 LinkedIn 构建的，并作为 Apache 许可下的开源产品提供。

在前面的部分中，我们已经使用 AMQP 连接构建了`EventEmitter`和`EventListener`接口的实现。在本节中，我们将使用 Kafka 来实现相同的接口。

# 使用 Docker 快速开始 Kafka

与 RabbitMQ 相反，Apache Kafka 设置起来要复杂一些。Kafka 本身需要一个工作的 Zookeeper 设置，以进行领导者选举、管理集群状态和持久化集群范围的配置数据。然而，为了开发目的，我们可以使用`spotify/kafka`镜像。该镜像带有内置的 Zookeeper 安装，可以快速轻松地设置。

就像之前的 RabbitMQ 图像一样，使用`docker run`命令快速开始：

```go
$ docker run -d --name kafka -p 9092:9092 spotify/kafka
```

这将启动一个单节点 Kafka 实例，并将其绑定到本地主机的 TCP 端口`9092`。

# Apache Kafka 的基本原则

Kafka 提供了一个发布/订阅消息代理，但不是基于 AMQP，因此使用不同的术语。

Kafka 中的第一个基本概念是主题。主题类似于订阅者可以写入的类别或事件名称。它包含了曾经发布到该主题的所有消息的完整日志。每个主题分为可配置数量的分区。当发布新消息时，需要包含一个分区键。代理使用分区键来决定消息应写入主题的哪个分区。

![](img/228af662-564a-48ba-90ca-d1c11e87a40f.png)

每个 Kafka 主题由可配置数量的分区组成；每个发布的消息都有一个分区键，用于决定消息应保存到哪个分区

Kafka 代理保证在每个分区内，消息的顺序与发布时的顺序相同。对于每个主题，消息将保留一段可配置的保留期。然而，当事务日志变得更大时，代理的性能并不会显著下降。因此，完全可以使用无限的保留期操作 Kafka，并以此方式将其用作事件日志。当然，您需要考虑所需的磁盘存储将成比例增长。幸运的是，Kafka 对水平扩展支持得相当好。

从每个主题，任意数量的订阅者（在 Kafka 行话中称为 *消费者*）可以读取消息，任意数量的发布者（*生产者*）可以写入消息。每个消费者都可以定义从事件日志的哪个偏移量开始消费。例如，一个刚初始化的只在内存中操作的消费者可以从头（偏移量 = `0`）读取整个事件日志以重建其整个查询模型。另一个只有本地数据库并且只需要在某个时间点之后发生的新事件的消费者可以从稍后的时间点开始读取事件日志。

每个消费者都是消费者组的成员。在给定主题中发布的消息将发布到每个组的一个消费者。这可以用来实现发布/订阅通信，类似于我们已经使用 AMQP 构建的内容。以下图表说明了使用 AMQP 和 Kafka 架构的发布/订阅中的不同术语和角色。在这两种情况下，发布在交换/主题中的每条消息都将路由到每个消费者。

![](img/27552bd3-67e3-4bf0-9a6b-7a99d8bdef10.png)

使用 AMQP（1）和 Apache Kafka（2）进行发布/订阅；在交换/主题中发布的每条消息都会路由到每个订阅者

在 AMQP 中，也可以有多个订阅者监听同一个队列。在这种情况下，传入的消息将不会路由到所有订阅者，而是路由到其中一个已连接的订阅者。这可以用来在不同的订阅者实例之间构建某种负载均衡。

在 Kafka 中，可以通过将多个订阅者实例放入同一消费者组来实现相同的功能。然而，在 Kafka 中，每个订阅者被分配到一个固定的（可能是多个）分区。因此，可以并行消费主题的消费者数量受到主题分区数量的限制。以下图表说明了这个例子：

![](img/00d574d2-a4b6-4166-a6c2-9b88b89e846a.png)

使用 AMQP（1）和 Apache Kafka（2）进行负载均衡；在交换/主题中发布的每条消息都会路由到已连接的订阅者之一

如果决定在同一消费者组中有多个消费者订阅主题的同一分区，代理将简单地将该分区中的所有消息分派给最后连接的消费者。

# 使用 Go 连接到 Kafka

在本章的前几节中，我们连接到 AMQP 代理时使用了事实上的标准库 `github.com/streadway/amqp`。连接到 Kafka 代理时，可用的 Go 库之间存在更多的多样性。在撰写本书时，Go 中最受欢迎的 Kafka 客户端库如下：

1.  `github.com/Shopify/sarama` 提供完整的协议支持，是纯 Go 实现的。它是根据 MIT 许可证授权的。它是维护活跃的。

1.  `github.com/elodina/go_kafka_client` 也是纯 Go 实现的。它提供的功能比 `Shopify` 库更多，但似乎维护活跃度较低。它是根据 Apache 许可证授权的。

1.  `github.com/confluentinc/confluent-kafka-go`为`librdkafka` C 库提供了一个 Go 包装器（这意味着您需要在系统上安装`librdkafka`才能使此库工作）。据说它比`Shopify`库更快，因为它依赖于高度优化的 C 库。不过，出于同样的原因，它可能难以构建。它正在积极维护，尽管其社区似乎比`Shopify`库小。

在本章中，我们将使用`github.com/Shopify/sarama`库。通过`go get`安装它：

```go
$ go get github.com/Shopify/sarama
```

在前面的部分中，我们已经在`todo.com/myevents/lib/msgqueue`包中定义了`EventEmitter`和`EventListener`接口。在本节中，我们将为这两个接口添加替代实现。在深入之前，让我们快速看一下如何使用`sarama`库来连接到 Kafka 代理。

无论您是打算发布还是消费消息，您都需要首先实例化`sarama.Client`结构。为此，您可以使用`sarama.NewClient`函数。要实例化一个新客户端，您需要 Kafka 代理地址的列表（记住，Kafka 设计为在集群中运行，因此您实际上可以同时连接到许多集群代理）和一个配置对象。创建配置对象的最简单方法是使用`sarama.NewConfig`函数：

```go
import "github.com/Shopify/sarama" 

func main() { 
  config := sarama.NewConfig() 
  brokers := []string{"localhost:9092"} 
  client, err := sarama.NewClient(brokers, config) 

  if err != nil { 
    panic(err) 
  } 
} 
```

当然，在开发设置中，将`localhost`作为单个代理工作正常。对于生产设置，代理列表应从环境中读取：

```go
func main() { 
  brokerList := os.Getenv("KAFKA_BROKERS") 
  if brokerList == "" { 
    brokerList = "localhost:9092" 
  } 

  brokers := strings.Split(brokerList, ",") 
  config := sarama.NewConfig() 

  client, err := sarama.NewClient(brokers, config) 
  // ... 
} 
```

您可以使用`config`对象来微调 Kafka 连接的各种参数。对于大多数情况，默认设置就可以了。

# 使用 Kafka 发布消息

Sarama 库提供了两种发布消息的实现——`sarama.SyncProducer`和`sarama.AsyncProducer`。

`AsyncProducer`提供了一个异步接口，使用 Go 通道来发布消息并检查这些操作的成功。它允许高吞吐量的消息，但如果您只想发出单个消息，使用起来有点笨重。因此，`SyncProducer`提供了一个更简单的接口，它接受一个用于生产的消息，并在从代理接收到消息已成功发布到事件日志的确认之前阻塞。

您可以使用`sarama.NewSyncProducerFromClient`和`sarama.NewAsyncProducerFromClient`函数实例化一个新的生产者。在我们的示例中，我们将使用`SyncProducer`，您可以按以下方式创建：

```go
producer, err := sarama.NewSyncProducerFromClient(client) 
if err != nil { 
  panic(err) 
} 
```

让我们继续使用`SyncProducer`来创建我们的`EventEmitter`接口的 Kafka 实现。首先创建`todo.com/myevents/lib/msgqueue/kafka`包和该包中的`emitter.go`文件：

```go
package kafka 

type kafkaEventEmitter struct { 
  producer sarama.SyncProducer 
} 
```

继续添加一个构造函数来实例化这个结构：

```go
func NewKafkaEventEmitter(client sarama.Client) (msgqueue.EventEmitter, error) { 
  producer, err := sarama.NewSyncProducerFromClient(client) 
  if err != nil { 
    return nil, err 
  } 

  emitter := &kafkaEventEmitter{ 
    producer: producer, 
  } 

  return emitter, nil 
} 
```

为了发送消息，您需要构建`sarama.ProducerMessage`结构的实例。为此，您需要主题（在我们的情况下，由`msgqueue.Event`的`EventName()`方法提供）和实际的消息正文。正文需要作为`sarama.Encoder`接口的实现提供。您可以使用`sarama.ByteEncoder`和`sarama.StringEncoder`类型，将字节数组或字符串简单地强制转换为`Encoder`实现：

```go
func (e *kafkaEventEmitter) Emit(event msgqueue.Event) error { 
  jsonBody, err := json.Marshal(event) 
  if err != nil { 
    return err 
  } 

  msg := &sarama.ProducerMessage{ 
    Topic: event.EventName(), 
    Value: sarama.ByteEncoder(jsonBody), 
  } 

  _, _, err = e.producer.SendMessage(msg) 
  return err 
} 
```

在此代码示例中，关键是生产者的`SendMessage()`方法。请注意，我们实际上忽略了此方法的一些返回值。前两个返回值返回了消息写入的分区号和消息在事件日志中的偏移量。

前面的代码是有效的，但有一个致命的缺陷：它为每种事件类型创建了一个新的 Kafka 主题。虽然订阅者完全可以同时消费多个主题，但无法保证处理顺序。这可能导致生产者按顺序短时间内依次发出`位置#1 创建`和`位置#1 更新`，而订阅者按照不同的顺序接收它们。

为了解决这个问题，我们需要做两件事：

+   所有消息必须发布在同一个主题上。这意味着我们需要另一种方法在消息中存储实际的事件名称。

+   每条消息必须公开一个分区键。我们可以使用消息的分区键来确保涉及相同实体的消息（即相同事件，相同用户）存储在事件日志的单个分区中，并且按顺序路由到相同的消费者。

让我们从分区键开始。还记得`todo.com/myevents/lib/msgqueue`包中的`Event`接口吗？它看起来是这样的：

```go
package msgqueue 

type Event interface { 
  EventName() string 
} 
```

继续添加一个新的`PartitionKey()`方法到这个接口：

```go
package msgqueue 

type Event interface { 
  PartitionKey() string 
  EventName() string 
} 
```

接下来，我们可以修改之前定义的现有事件结构（例如`EventCreatedEvent`）来实现这个`PartitionKey()`方法：

```go
func (e *EventCreatedEvent) PartitionKey() string { 
  return e.ID 
} 
```

现在，让我们回到`kafkaEventEmitter`。我们现在可以在将消息发布到 Kafka 时使用每个事件的`PartitionKey()`方法。现在，我们只需要在事件旁边发送事件名称。为了解决这个问题，我们将在`todo.com/myevents/lib/msgqueue/kafka`包的新文件`payload.go`中定义这个事件：

```go
package kafka 

type messageEnvelope struct { 
  EventName string      `json:"eventName"` 
  Payload   interface{} `json:"payload"` 
} 
```

现在，我们可以调整`kafkaEventEmitter`，首先构造`messageEnvelope`结构的实例，然后对其进行 JSON 序列化：

```go
func (e *kafkaEventEmitter) Emit(event msgqueue.Event) error { 
  envelope := messageEnvelope{event.EventName(), event} 
  jsonBody, err := json.Marshal(&envelope) 
  // ... 
```

# 从 Kafka 消费消息

从 Kafka 代理服务器消费消息比在 AMQP 中更复杂一些。您已经了解到 Kafka 主题可能由许多分区组成，每个消费者可以消费一个或多个（最多全部）这些分区。Kafka 架构允许通过将主题分成更多分区并让一个消费者订阅每个分区来进行水平扩展。

这意味着每个订阅者都需要知道主题的哪些分区存在，以及它应该消费其中的哪些。我们在本节中介绍的一些库（尤其是 Confluent 库）实际上支持自动订阅者分区和自动组平衡。`sarama`库不提供此功能，因此我们的`EventListener`将需要手动选择要消费的分区。

对于我们的示例，我们将实现`EventListener`，以便默认情况下监听主题的所有可用分区。我们将添加一个特殊属性，用于明确指定要监听的分区。

在`todo.com/myevents/lib/msgqueue/kafka`包中创建一个新文件`listener.go`：

```go
package kafka 

import "github.com/Shopify/sarama" 
import "todo.com/myevents/lib/msgqueue" 

type kafkaEventListener struct { 
  consumer   sarama.Consumer 
  partitions []int32 
} 
```

继续为这个结构体添加一个构造函数：

```go
func NewKafkaEventListener(client sarama.Client, partitions []int32) (msgqueue.EventListener, error) { 
  consumer, err := sarama.NewConsumerFromClient(client) 
  if err != nil { 
    return nil, err 
  } 

  listener := &kafkaEventListener{ 
    consumer: consumer, 
    partitions: partitions, 
  } 

  return listener, nil 
} 
```

`kafkaEventListener`的`Listen()`方法遵循与我们在上一节中实现的`amqpEventListener`相同的接口：

```go
func (k *kafkaEventListener) Listen(events ...string) (<-chan msgqueue.Event, <-chan error, error) { 
  var err error 

  topic := "events" 
  results := make(chan msgqueue.Event) 
  errors := make(chan error) 
} 
```

首先要做的是确定应该消费哪些主题分区。我们将假设当`NewKafkaEventListener`方法传递了一个空切片时，监听器应该监听所有分区：

```go
func (k *kafkaEventListener) Listen(events ...string) (<-chan msgqueue.Event, <-chan error, error) { 
  var err error 

  topic := "events" 
  results := make(chan msgqueue.Event) 
  errors := make(chan error) 

  partitions := k.partitions 
  if len(partitions) == 0 { 
    partitions, err = k.consumer.partitions(topic) 
    if err != nil { 
      return nil, nil, err 
    } 
  } 

  log.Printf("topic %s has partitions: %v", topic, partitions) 
} 
```

Sarama 消费者只能消费一个分区。如果我们想要消费多个分区，我们需要启动多个消费者。为了保持`EventListener`的接口，我们将在`Listen()`方法中启动多个消费者，每个消费者在自己的 goroutine 中运行，然后让它们都写入同一个结果通道：

```go
func (k *kafkaEventListener) Listen(events ...string) (<-chan msgqueue.Event, <-chan error, error) { 
  // ... 

  log.Printf("topic %s has partitions: %v", topic, partitions) 

  for _, partitions := range partitions { 
    con, err := k.consumer.ConsumePartition(topic, partition, 0) 
    if err != nil { 
      return nil, nil, err 
    } 

    go func() { 
      for msg := range con.Messages() { 

      } 
    }() 
  } 
} 
```

注意在第一个 for 循环内启动的 goroutines。其中每个都包含一个内部 for 循环，遍历给定分区中接收到的所有消息。现在我们可以对传入的消息进行 JSON 解码，并重建适当的事件类型。

以下所有代码示例都放置在`kafkaEventListener`的`Listen()`方法的内部 for 循环中。

```go
for msg := range con.Messages() { 
  body := messageEnvelope{} 
  err := json.Unmarshal(msg.Value, &body) 
  if err != nil { 
    errors <- fmt.Errorf("could not JSON-decode message: %s", err) 
    continue 
  } 
} 
```

现在我们有一个新问题。我们已经将事件主体解组为`messageEnvelope`结构。这包含了事件名称和实际事件主体。然而，事件主体只是被定义为`interface{}`。理想情况下，我们需要将这个`interface{}`类型转换回正确的事件类型（例如，`contracts.EventCreatedEvent`），这取决于事件名称。为此，我们可以使用`github.com/mitchellh/mapstructure`包，您可以通过 go get 安装：

```go
$ go get -u github.com/mitchellh/mapstructure
```

`mapstructure`库的工作方式类似于`encoding/json`库，只是它不接受`[]byte`输入变量，而是通用的`interface{}`输入值。这允许您接受未知结构的 JSON 输入（通过在`interface{}`值上调用`json.Unmarshal`），然后将已解码的未知结构类型映射到已知的结构类型：

```go
for msg := range con.Messages() { 
  body := messageEnvelope{} 
  err := json.Unmarshal(msg.Value, &body) 
  if err != nil { 
    errors <- fmt.Errorf("could not JSON-decode message: %s", err) 
    continue 
  } 

  var event msgqueue.Event 
  switch body.EventName { 
    case "event.created": 
      event = contracts.EventCreatedEvent{} 
    case "location.created": 
      event = contracts.LocationCreatedEvent{} 
    default: 
      errors <- fmt.Errorf("unknown event type: %s", body.EventName) 
      continue 
  } 

  cfg := mapstructure.DecoderConfig{ 
    Result: event, 
    TagName: "json", 
  } 
  err = mapstructure.NewDecoder(&cfg).Decode(body.Payload) 
  if err != nil { 
    errors <- fmt.Errorf("could not map event %s: %s", body.EventName, err) 
  } 
} 
```

在实际解码之前创建的`mapstructure.DecoderConfig`结构中的`TagName`属性指示`mapstructure`库尊重事件合同中已经存在的``json:"..."``注释。

成功解码消息后，可以将其发布到结果通道中：

```go
for msg := range con.Messages() { 
  // ...   
  err = mapstructure.NewDecoder(&cfg).Decode(body.Payload) 
  if err != nil { 
    errors <- fmt.Errorf("could not map event %s: %s", body.EventName, err) 
  } 

  results <- event 
} 
```

我们的 Kafka 事件监听器现在已经完全可用。由于它实现了`msgqueue.EventListener`接口，您可以将其用作现有 AMQP 事件监听器的即插即用替代品。

然而，有一个警告。当启动时，我们当前的 Kafka 事件监听器总是从事件日志的开头开始消费。仔细看一下前面代码示例中的`ConsumePartition`调用——它的第三个参数（在我们的例子中是`0`）描述了消费者应该开始消费的事件日志中的偏移量。

使用`0`作为偏移量将指示事件监听器从头开始读取整个事件日志。如果您想要使用 Kafka 实现事件溯源，这是理想的解决方案。如果您只想将 Kafka 用作消息代理，您的服务将需要记住从事件日志中读取的最后一条消息的偏移量。当您的服务重新启动时，您可以从上次已知的位置继续消费。

# 总结

在本章中，您学习了如何使用消息队列（如 RabbitMQ 和 Apache Kafka）集成多个服务进行异步通信。您还了解了事件协作和事件溯源等架构模式，这有助于您构建适合云部署的可扩展和弹性的应用程序。

本章中我们使用的技术与任何特定的云提供商无关。您可以轻松地在任何云基础设施或您自己的服务器上部署自己的 RabbitMQ 或 Kafka 基础设施。在第八章中，*AWS 第二部分-S3、SQS、API 网关和 DynamoDB*，我们将再次关注消息队列，这次特别关注 AWS 提供给您的托管消息解决方案。
