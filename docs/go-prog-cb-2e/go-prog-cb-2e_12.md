# 第十二章：响应式编程和数据流

在本章中，我们将讨论 Go 中的响应式编程设计模式。响应式编程是一种专注于数据流和变化传播的编程概念。诸如 Kafka 之类的技术允许您快速生成或消费数据流。因此，这些技术彼此之间是自然契合的。在*将 Kafka 连接到 Goflow*配方中，我们将探讨将`kafka`消息队列与`goflow`结合起来，以展示使用这些技术的实际示例。本章还将探讨连接到 Kafka 并使用它来处理消息的各种方法。最后，本章将演示如何在 Go 中创建一个基本的`graphql`服务器。

在本章中，我们将涵盖以下配方：

+   使用 Goflow 进行数据流编程

+   使用 Sarama 与 Kafka

+   使用异步生产者与 Kafka

+   将 Kafka 连接到 Goflow

+   使用 Go 编写 GraphQL 服务器

# 技术要求

为了继续本章中的所有配方，根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.12.6 或更高版本，网址为[`golang.org/doc/install.`](https://golang.org/doc/install)

1.  打开一个终端或控制台应用程序，并创建并导航到一个名为`~/projects/go-programming-cookbook`的项目目录。所有代码都将从这个目录运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，或者选择从该目录工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 使用 Goflow 进行数据流编程

`github.com/trustmaster/goflow`包对于创建基于数据流的应用程序非常有用。它试图抽象概念，以便您可以编写组件并使用自定义网络将它们连接在一起。这个配方将重新创建第九章中讨论的应用程序，*测试 Go 代码*，但将使用`goflow`包来实现。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter12/goflow`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/goflow 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/goflow   
```

1.  从`~/projects/go-programming-cookbook-original/chapter12/goflow`复制测试，或者使用这个作为练习来编写一些您自己的代码！

1.  创建一个名为`components.go`的文件，内容如下：

```go
package goflow

import (
  "encoding/base64"
  "fmt"
)

// Encoder base64 encodes all input
type Encoder struct {
  Val <-chan string
  Res chan<- string
}

// Process does the encoding then pushes the result onto Res
func (e *Encoder) Process() {
  for val := range e.Val {
    encoded := base64.StdEncoding.EncodeToString([]byte(val))
    e.Res <- fmt.Sprintf("%s => %s", val, encoded)
  }
}

// Printer is a component for printing to stdout
type Printer struct {
  Line <-chan string
}

// Process Prints the current line received
func (p *Printer) Process() {
  for line := range p.Line {
    fmt.Println(line)
  }
}
```

1.  创建一个名为`network.go`的文件，内容如下：

```go
package goflow

import (
  "github.com/trustmaster/goflow"
)

// NewEncodingApp wires together the components
func NewEncodingApp() *goflow.Graph {
  e := goflow.NewGraph()

  // define component types
  e.Add("encoder", new(Encoder))
  e.Add("printer", new(Printer))

  // connect the components using channels
  e.Connect("encoder", "Res", "printer", "Line")

  // map the in channel to Val, which is
  // tied to OnVal function
  e.MapInPort("In", "encoder", "Val")

  return e
}
```

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "fmt"

  "github.com/PacktPublishing/
   Go-Programming-Cookbook-Second-Edition/chapter12/goflow"
  flow "github.com/trustmaster/goflow"
)

func main() {

  net := goflow.NewEncodingApp()

  in := make(chan string)
  net.SetInPort("In", in)

  wait := flow.Run(net)

  for i := 0; i < 20; i++ {
    in <- fmt.Sprint("Message", i)
  }

  close(in)
  <-wait
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
Message6 => TWVzc2FnZTY=
Message5 => TWVzc2FnZTU=
Message1 => TWVzc2FnZTE=
Message0 => TWVzc2FnZTA=
Message4 => TWVzc2FnZTQ=
Message8 => TWVzc2FnZTg=
Message2 => TWVzc2FnZTI=
Message3 => TWVzc2FnZTM=
Message7 => TWVzc2FnZTc=
Message10 => TWVzc2FnZTEw
Message9 => TWVzc2FnZTk=
Message12 => TWVzc2FnZTEy
Message11 => TWVzc2FnZTEx
Message14 => TWVzc2FnZTE0
Message13 => TWVzc2FnZTEz
Message16 => TWVzc2FnZTE2
Message15 => TWVzc2FnZTE1
Message18 => TWVzc2FnZTE4
Message17 => TWVzc2FnZTE3
Message19 => TWVzc2FnZTE5
```

1.  `go.mod`文件可能会更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您已经复制或编写了自己的测试，请返回上一级目录并运行`go test`命令。确保所有测试都通过。

# 它是如何工作的...

`github.com/trustmaster/goflow`包的工作方式是定义一个网络/图，注册一些组件，然后将它们连接在一起。这可能会感觉有点容易出错，因为组件是用字符串描述的，但通常在运行时会在应用程序设置和功能正确运行之前就会出现错误。

在这个配方中，我们设置了两个组件，一个是对传入的字符串进行 Base64 编码，另一个是打印传递给它的任何内容。我们将它连接到在`main.go`中初始化的输入通道，任何传递到该通道的内容都将通过我们的管道流动。

这种方法的重点很大程度上在于忽略正在进行的内部工作。我们把一切都当作连接的黑匣子，并让`goflow`来处理剩下的事情。您可以看到，在这个配方中，完成这个任务流水线的代码是多么简洁，而且我们有更少的旋钮来控制工作人员的数量，等等。

# 使用 Sarama 与 Kafka

Kafka 是一个流行的分布式消息队列，具有许多用于构建分布式系统的高级功能。本配方将展示如何使用同步生产者向 Kafka 主题写入，并如何使用分区消费者消费相同的主题。本配方不会探讨 Kafka 的不同配置，因为这是一个超出本书范围的更广泛的主题，但我建议从[`kafka.apache.org/intro`](https://kafka.apache.org/intro)开始。

# 做好准备

根据以下步骤配置您的环境：

1.  参考本章开头的*技术要求*部分。

1.  按照[`www.tutorialspoint.com/apache_kafka/apache_kafka_installation_steps.htm`](https://www.tutorialspoint.com/apache_kafka/apache_kafka_installation_steps.htm)中提到的步骤安装 Kafka。

1.  或者，您也可以访问[`github.com/spotify/docker-kafka`](https://github.com/spotify/docker-kafka)。

# 如何做...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter12/synckafka`的新目录，并导航到该目录。

1.  运行此命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/synckafka 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/synckafka   
```

1.  从`~/projects/go-programming-cookbook-original/chapter12/synckafka`复制测试，或者使用这个作为练习来编写一些您自己的代码！

1.  确保 Kafka 在`localhost:9092`上运行正常。

1.  在名为`consumer`的目录中创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "log"

            sarama "github.com/Shopify/sarama"
        )

        func main() {
            consumer, err := 
            sarama.NewConsumer([]string{"localhost:9092"}, nil)
            if err != nil {
                panic(err)
            }
            defer consumer.Close()

            partitionConsumer, err := 

           consumer.ConsumePartition("example", 0, 
            sarama.OffsetNewest)
            if err != nil {
                panic(err)
            }
            defer partitionConsumer.Close()

            for {
                msg := <-partitionConsumer.Messages()
                log.Printf("Consumed message: \"%s\" at offset: %d\n", 
                msg.Value, msg.Offset)
            }
        }
```

1.  在名为`producer`的目录中创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (

           "fmt"
           "log"

            sarama "github.com/Shopify/sarama"
        )

        func sendMessage(producer sarama.SyncProducer, value string) {
            msg := &sarama.ProducerMessage{Topic: "example", Value: 
            sarama.StringEncoder(value)}
            partition, offset, err := producer.SendMessage(msg)
            if err != nil {

               log.Printf("FAILED to send message: %s\n", err)
                return
            }
            log.Printf("> message sent to partition %d at offset %d\n", 
            partition, offset)
        }

        func main() {
            producer, err := 
            sarama.NewSyncProducer([]string{"localhost:9092"}, nil)
            if err != nil {
                panic(err)
            }
            defer producer.Close()

            for i := 0; i < 10; i++ {
                sendMessage(producer, fmt.Sprintf("Message %d", i))
            }
        }
```

1.  导航到上一级目录。

1.  运行`go run ./consumer`。

1.  在与同一目录的另一个终端中运行`go run ./producer`。

1.  在生产者终端中，您应该看到以下内容：

```go
$ go run ./producer 
2017/05/07 11:50:38 > message sent to partition 0 at offset 0
2017/05/07 11:50:38 > message sent to partition 0 at offset 1
2017/05/07 11:50:38 > message sent to partition 0 at offset 2
2017/05/07 11:50:38 > message sent to partition 0 at offset 3
2017/05/07 11:50:38 > message sent to partition 0 at offset 4
2017/05/07 11:50:38 > message sent to partition 0 at offset 5
2017/05/07 11:50:38 > message sent to partition 0 at offset 6
2017/05/07 11:50:38 > message sent to partition 0 at offset 7
2017/05/07 11:50:38 > message sent to partition 0 at offset 8
2017/05/07 11:50:38 > message sent to partition 0 at offset 9
```

在消费者终端中，您应该看到以下内容：

```go
$ go run ./consumer
2017/05/07 11:50:38 Consumed message: "Message 0" at offset: 0
2017/05/07 11:50:38 Consumed message: "Message 1" at offset: 1
2017/05/07 11:50:38 Consumed message: "Message 2" at offset: 2
2017/05/07 11:50:38 Consumed message: "Message 3" at offset: 3
2017/05/07 11:50:38 Consumed message: "Message 4" at offset: 4
2017/05/07 11:50:38 Consumed message: "Message 5" at offset: 5
2017/05/07 11:50:38 Consumed message: "Message 6" at offset: 6
2017/05/07 11:50:38 Consumed message: "Message 7" at offset: 7
2017/05/07 11:50:38 Consumed message: "Message 8" at offset: 8
2017/05/07 11:50:38 Consumed message: "Message 9" at offset: 9
```

1.  `go.mod`文件可能已更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您已经复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 如何运作...

该配方演示了通过 Kafka 传递简单消息。更复杂的方法应该使用诸如`json`、`gob`、`protobuf`或其他的序列化格式。生产者可以通过`sendMessage`同步地向 Kafka 发送消息。这在 Kafka 集群宕机的情况下处理得不好，并且可能导致这些情况下的进程挂起。这对于诸如 Web 处理程序之类的应用程序来说很重要，因为它可能导致超时并且对 Kafka 集群有硬性依赖。

假设消息队列正确，我们的消费者将观察 Kafka 流并对结果进行处理。本章中的先前配方可能利用此流来进行一些额外的处理。

# 使用 Kafka 的异步生产者

在继续下一个任务之前，等待 Kafka 生产者完成通常是没有意义的。在这种情况下，您可以使用异步生产者。这些生产者在通道上接收 Sarama 消息，并具有返回成功/错误通道的方法，可以单独检查。

在本配方中，我们将创建一个 Go 例程，用于处理成功和失败的消息，同时允许处理程序排队发送消息，而不管结果如何。

# 做好准备

参考*Sarama 使用 Kafka*配方中的*做好准备*部分。

# 如何做...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter12/asynckafka`的新目录，并导航到该目录。

1.  运行此命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/asynckafka 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/asynckafka   
```

1.  从`~/projects/go-programming-cookbook-original/chapter12/asynckafka`复制测试，或者使用这个作为练习来编写一些您自己的代码！

1.  确保 Kafka 在`localhost:9092`上运行正常。

1.  从上一个配方中复制 consumer 目录。

1.  创建一个名为`producer`的目录并导航到该目录。

1.  创建一个名为`producer.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "log"

            sarama "github.com/Shopify/sarama"
        )

        // Process response grabs results and errors from a producer
        // asynchronously
        func ProcessResponse(producer sarama.AsyncProducer) {
            for {
                select {
                    case result := <-producer.Successes():
                    log.Printf("> message: \"%s\" sent to partition 
                    %d at offset %d\n", result.Value, 
                    result.Partition, result.Offset)
                    case err := <-producer.Errors():
                    log.Println("Failed to produce message", err)
                }
            }
        }
```

1.  创建一个名为`handler.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "net/http"

            sarama "github.com/Shopify/sarama"
        )

        // KafkaController allows us to attach a producer
        // to our handlers
        type KafkaController struct {
            producer sarama.AsyncProducer
        }

        // Handler grabs a message from a GET parama and
        // send it to the kafka queue asynchronously
        func (c *KafkaController) Handler(w http.ResponseWriter, r 
        *http.Request) {
            if err := r.ParseForm(); err != nil {
                w.WriteHeader(http.StatusBadRequest)
                return
            }

            msg := r.FormValue("msg")
            if msg == "" {
                w.WriteHeader(http.StatusBadRequest)
                w.Write([]byte("msg must be set"))
                return
            }
            c.producer.Input() <- &sarama.ProducerMessage{Topic: 
            "example", Key: nil, Value: 
            sarama.StringEncoder(msg)}
            w.WriteHeader(http.StatusOK)
        }
```

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"
            "net/http"

            sarama "github.com/Shopify/sarama"
        )

        func main() {
            config := sarama.NewConfig()
            config.Producer.Return.Successes = true
            config.Producer.Return.Errors = true
            producer, err := 
            sarama.NewAsyncProducer([]string{"localhost:9092"}, config)
            if err != nil {
                panic(err)
            }
            defer producer.AsyncClose()

            go ProcessResponse(producer)

            c := KafkaController{producer}
            http.HandleFunc("/", c.Handler)
            fmt.Println("Listening on port :3333")
            panic(http.ListenAndServe(":3333", nil))
        }
```

1.  返回到上一级目录。

1.  运行`go run ./consumer`。

1.  在与同一目录的另一个终端中运行`go run ./producer`。

1.  在第三个终端中，运行以下命令：

```go
$ curl "http://localhost:3333/?msg=this"      
$ curl "http://localhost:3333/?msg=is" 
$ curl "http://localhost:3333/?msg=an" 
$ curl "http://localhost:3333/?msg=example" 
```

在生产者终端中，您应该看到以下内容：

```go
$ go run ./producer
Listening on port :3333
2017/05/07 13:52:54 > message: "this" sent to partition 0 at offset 0
2017/05/07 13:53:25 > message: "is" sent to partition 0 at offset 1
2017/05/07 13:53:27 > message: "an" sent to partition 0 at offset 2
2017/05/07 13:53:29 > message: "example" sent to partition 0 at offset 3
```

1.  在消费者终端中，您应该看到这个：

```go
$ go run ./consumer
2017/05/07 13:52:54 Consumed message: "this" at offset: 0
2017/05/07 13:53:25 Consumed message: "is" at offset: 1
2017/05/07 13:53:27 Consumed message: "an" at offset: 2
2017/05/07 13:53:29 Consumed message: "example" at offset: 3
```

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶级食谱目录中。

1.  如果您已经复制或编写了自己的测试，请返回到上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

我们在本章中的修改都是针对生产者的。这一次，我们创建了一个单独的 Go 例程来处理成功和错误。如果这些问题没有得到处理，您的应用程序将陷入死锁。接下来，我们将我们的生产者附加到一个处理程序上，并在收到消息时通过对处理程序的`GET`调用发出消息。

处理程序在发送消息后将立即返回成功，而不管其响应如何。如果这是不可接受的，应该改用同步方法。在我们的情况下，我们可以接受稍后分别处理成功和错误。

最后，我们用几条不同的消息`curl`我们的端点，您可以看到它们从处理程序流向我们在上一节中编写的 Kafka 消费者最终打印的地方。

# 将 Kafka 连接到 Goflow

这个食谱将把 Kafka 消费者与 Goflow 管道结合起来。当我们的消费者从 Kafka 接收消息时，它将对它们运行`strings.ToUpper()`，然后打印结果。这些自然配对，因为 Goflow 旨在操作传入流，这正是 Kafka 提供给我们的。

# 准备就绪

参考*使用 Sarama 与 Kafka*食谱*的*准备就绪*部分。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter12/kafkaflow`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/kafkaflow 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/kafkaflow   
```

1.  从`~/projects/go-programming-cookbook-original/chapter12/kafkaflow`复制测试，或者将其用作编写一些自己代码的练习！

1.  确保 Kafka 在`localhost:9092`上运行。

1.  创建一个名为`components.go`的文件，其中包含以下内容：

```go
package kafkaflow

import (
  "fmt"
  "strings"

  flow "github.com/trustmaster/goflow"
)

// Upper upper cases the incoming
// stream
type Upper struct {
  Val <-chan string
  Res chan<- string
}

// Process loops over the input values and writes the upper
// case string version of them to Res
func (e *Upper) Process() {
  for val := range e.Val {
    e.Res <- strings.ToUpper(val)
  }
}

// Printer is a component for printing to stdout
type Printer struct {
  flow.Component
  Line <-chan string
}

// Process Prints the current line received
func (p *Printer) Process() {
  for line := range p.Line {
    fmt.Println(line)
  }
}
```

1.  创建一个名为`network.go`的文件，其中包含以下内容：

```go
package kafkaflow

import "github.com/trustmaster/goflow"

// NewUpperApp wires together the components
func NewUpperApp() *goflow.Graph {
  u := goflow.NewGraph()

  u.Add("upper", new(Upper))
  u.Add("printer", new(Printer))

  u.Connect("upper", "Res", "printer", "Line")
  u.MapInPort("In", "upper", "Val")

  return u
}
```

1.  在名为`consumer`的目录中创建一个名为`main.go`的文件，其中包含以下内容：

```go
package main

import (
  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/kafkaflow"
  sarama "github.com/Shopify/sarama"
  flow "github.com/trustmaster/goflow"
)

func main() {
  consumer, err := sarama.NewConsumer([]string{"localhost:9092"}, nil)
  if err != nil {
    panic(err)
  }
  defer consumer.Close()

  partitionConsumer, err := consumer.ConsumePartition("example", 0, sarama.OffsetNewest)
  if err != nil {
    panic(err)
  }
  defer partitionConsumer.Close()

  net := kafkaflow.NewUpperApp()

  in := make(chan string)
  net.SetInPort("In", in)

  wait := flow.Run(net)
  defer func() {
    close(in)
    <-wait
  }()

  for {
    msg := <-partitionConsumer.Messages()
    in <- string(msg.Value)
  }

}
```

1.  从*使用 Sarama 与 Kafka*食谱中复制`producer`目录。

1.  运行`go run ./consumer`。

1.  在与同一目录的另一个终端中运行`go run ./producer`。

1.  在生产者终端中，您现在应该看到以下内容：

```go
$ go run ./producer 
2017/05/07 18:24:12 > message "Message 0" sent to partition 0 at offset 0
2017/05/07 18:24:12 > message "Message 1" sent to partition 0 at offset 1
2017/05/07 18:24:12 > message "Message 2" sent to partition 0 at offset 2
2017/05/07 18:24:12 > message "Message 3" sent to partition 0 at offset 3
2017/05/07 18:24:12 > message "Message 4" sent to partition 0 at offset 4
2017/05/07 18:24:12 > message "Message 5" sent to partition 0 at offset 5
2017/05/07 18:24:12 > message "Message 6" sent to partition 0 at offset 6
2017/05/07 18:24:12 > message "Message 7" sent to partition 0 at offset 7
2017/05/07 18:24:12 > message "Message 8" sent to partition 0 at offset 8
2017/05/07 18:24:12 > message "Message 9" sent to partition 0 at offset 9
```

在消费者终端中，您应该看到以下内容：

```go
$ go run ./consumer
MESSAGE 0
MESSAGE 1
MESSAGE 2
MESSAGE 3
MESSAGE 4
MESSAGE 5
MESSAGE 6
MESSAGE 7
MESSAGE 8
MESSAGE 9
```

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶级食谱目录中。

1.  如果您已经复制或编写了自己的测试，请返回到上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个食谱结合了本章中以前食谱的想法。与以前的食谱一样，我们设置了 Kafka 消费者和生产者。这个食谱使用了*使用 Sarama 与 Kafka*食谱中的同步生产者，但也可以使用异步生产者。一旦收到消息，我们就像在*数据流编程的 Goflow*食谱中一样，在输入通道上排队。我们修改了这个食谱中的组件，将我们的传入字符串转换为大写，而不是 Base64 编码。我们重用打印组件，结果网络配置类似。

最终结果是，通过 Kafka 消费者接收的所有消息都被传输到我们基于流的工作流中进行操作。这使我们能够将我们的工作流组件进行模块化和可重用，并且我们可以在不同的配置中多次使用相同的组件。同样，我们将从任何写入 Kafka 的生产者接收流量，因此我们可以将生产者多路复用成单个数据流。

# 在 Go 中编写 GraphQL 服务器

GraphQL 是由 Facebook 创建的 REST 的替代品（[`graphql.org/`](http://graphql.org/)）。这项技术允许服务器实现和发布模式，然后客户端可以请求他们需要的信息，而不是理解和利用各种 API 端点。

对于这个示例，我们将创建一个代表一副扑克牌的`Graphql`模式。我们将公开一个名为 card 的资源，可以按花色和值进行过滤。或者，如果未指定参数，此模式可以返回牌组中的所有牌。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter12/graphql`的新目录，并导航到该目录。

1.  运行此命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/graphql 
```

应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter12/graphql   
```

1.  从`~/projects/go-programming-cookbook-original/chapter12/graphql`复制测试，或者将其用作练习来编写自己的代码！

1.  创建并导航到`cards`目录。

1.  创建一个名为`card.go`的文件，其中包含以下内容：

```go
        package cards

        // Card represents a standard playing
        // card
        type Card struct {
            Value string
            Suit string
        }

        var cards []Card

        func init() {
            cards = []Card{
                {"A", "Spades"}, {"2", "Spades"}, {"3", "Spades"},
                {"4", "Spades"}, {"5", "Spades"}, {"6", "Spades"},
                {"7", "Spades"}, {"8", "Spades"}, {"9", "Spades"},
                {"10", "Spades"}, {"J", "Spades"}, {"Q", "Spades"},
                {"K", "Spades"},
                {"A", "Hearts"}, {"2", "Hearts"}, {"3", "Hearts"},
                {"4", "Hearts"}, {"5", "Hearts"}, {"6", "Hearts"},
                {"7", "Hearts"}, {"8", "Hearts"}, {"9", "Hearts"},
                {"10", "Hearts"}, {"J", "Hearts"}, {"Q", "Hearts"},
                {"K", "Hearts"},
                {"A", "Clubs"}, {"2", "Clubs"}, {"3", "Clubs"},
                {"4", "Clubs"}, {"5", "Clubs"}, {"6", "Clubs"},
                {"7", "Clubs"}, {"8", "Clubs"}, {"9", "Clubs"},
                {"10", "Clubs"}, {"J", "Clubs"}, {"Q", "Clubs"},
                {"K", "Clubs"},
                {"A", "Diamonds"}, {"2", "Diamonds"}, {"3", 
                "Diamonds"},
                {"4", "Diamonds"}, {"5", "Diamonds"}, {"6", 
                "Diamonds"},
                {"7", "Diamonds"}, {"8", "Diamonds"}, {"9", 
                "Diamonds"},
                {"10", "Diamonds"}, {"J", "Diamonds"}, {"Q", 
                "Diamonds"},
                {"K", "Diamonds"},
            }
        }
```

1.  创建一个名为`type.go`的文件，其中包含以下内容：

```go
        package cards

        import "github.com/graphql-go/graphql"

        // CardType returns our card graphql object
        func CardType() *graphql.Object {
            cardType := graphql.NewObject(graphql.ObjectConfig{
                Name: "Card",
                Description: "A Playing Card",
                Fields: graphql.Fields{
                    "value": &graphql.Field{
                        Type: graphql.String,
                        Description: "Ace through King",
                        Resolve: func(p graphql.ResolveParams) 
                        (interface{}, error) {
                            if card, ok := p.Source.(Card); ok {
                                return card.Value, nil
                            }
                            return nil, nil
                        },
                    },
                    "suit": &graphql.Field{
                        Type: graphql.String,
                        Description: "Hearts, Diamonds, Clubs, Spades",
                        Resolve: func(p graphql.ResolveParams) 
                        (interface{}, error) {
                            if card, ok := p.Source.(Card); ok {
                                return card.Suit, nil
                            }
                            return nil, nil
                        },
                    },
                },
            })
            return cardType
        }
```

1.  创建一个名为`resolve.go`的文件，其中包含以下内容：

```go
        package cards

        import (
            "strings"

            "github.com/graphql-go/graphql"
        )

        // Resolve handles filtering cards
        // by suit and value
        func Resolve(p graphql.ResolveParams) (interface{}, error) {
            finalCards := []Card{}
            suit, suitOK := p.Args["suit"].(string)
            suit = strings.ToLower(suit)

            value, valueOK := p.Args["value"].(string)
            value = strings.ToLower(value)

            for _, card := range cards {
                if suitOK && suit != strings.ToLower(card.Suit) {
                    continue
                }
                if valueOK && value != strings.ToLower(card.Value) {
                    continue
                }

                finalCards = append(finalCards, card)
            }
            return finalCards, nil
        }
```

1.  创建一个名为`schema.go`的文件，其中包含以下内容：

```go
        package cards

        import "github.com/graphql-go/graphql"

        // Setup prepares and returns our card
        // schema
        func Setup() (graphql.Schema, error) {
            cardType := CardType()

            // Schema
            fields := graphql.Fields{
                "cards": &graphql.Field{
                    Type: graphql.NewList(cardType),
                    Args: graphql.FieldConfigArgument{
                        "suit": &graphql.ArgumentConfig{
                            Description: "Filter cards by card suit 
                            (hearts, clubs, diamonds, spades)",
                            Type: graphql.String,
                        },
                        "value": &graphql.ArgumentConfig{
                            Description: "Filter cards by card 
                            value (A-K)",
                            Type: graphql.String,
                        },
                    },
                    Resolve: Resolve,
                },
            }

            rootQuery := graphql.ObjectConfig{Name: "RootQuery", 
            Fields: fields}
            schemaConfig := graphql.SchemaConfig{Query: 
            graphql.NewObject(rootQuery)}
            schema, err := graphql.NewSchema(schemaConfig)

            return schema, err
        }
```

1.  导航回`graphql`目录。

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "encoding/json"
            "fmt"
            "log"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter12/graphql/cards"
            "github.com/graphql-go/graphql"
        )

        func main() {
            // grab our schema
            schema, err := cards.Setup()
            if err != nil {
                panic(err)
            }

            // Query
            query := `
            {
                cards(value: "A"){
                    value
                    suit
                }
            }`

            params := graphql.Params{Schema: schema, RequestString: 
            query}
            r := graphql.Do(params)
            if len(r.Errors) > 0 {
                log.Fatalf("failed to execute graphql operation, 
                errors: %+v", r.Errors)
            }
            rJSON, err := json.MarshalIndent(r, "", " ")
            if err != nil {
                panic(err)
            }
            fmt.Printf("%s \n", rJSON)
        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
{
 "data": {
 "cards": [
 {
 "suit": "Spades",
 "value": "A"
 },
 {
 "suit": "Hearts",
 "value": "A"
 },
 {
 "suit": "Clubs",
 "value": "A"
 },
 {
 "suit": "Diamonds",
 "value": "A"
 }
 ]
 }
} 
```

1.  测试一些额外的查询，例如以下内容：

+   `cards(suit: "Spades")`

+   `cards(value: "3", suit:"Diamonds")`

1.  `go.mod`文件可能已更新，并且`go.sum`文件现在应该存在于顶级示例目录中。

1.  如果您已经复制或编写了自己的测试，请返回上一个目录并运行`go test`。确保所有测试都通过。

# 工作原理...

`cards.go`文件定义了一个`card`对象，并在名为`cards`的全局变量中初始化了基本牌组。这种状态也可以保存在长期存储中，例如数据库中。然后，我们在`types.go`中定义了`CardType`，它允许`graphql`将卡对象解析为响应。接下来，我们进入`resolve.go`，在那里我们定义了如何按值和类型过滤卡片。这个`Resolve`函数将被最终的模式使用，该模式在`schema.go`中定义。

例如，您可以修改此示例中的`Resolve`函数，以从数据库中检索数据。最后，我们加载模式并对其运行查询。这是一个小修改，将我们的模式挂载到 REST 端点，但为了简洁起见，此示例只运行一个硬编码查询。有关`GraphQL`查询的更多信息，请访问[`graphql.org/learn/queries/`](http://graphql.org/learn/queries/)。
