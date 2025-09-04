# 第十一章：响应式编程和数据流

在本章中，我们将涵盖以下配方：

+   Goflow 用于数据流编程

+   使用 RxGo 进行响应式编程

+   使用 Sarama 与 Kafka

+   使用 Kafka 的异步生产者

+   将 Kafka 连接到 Goflow

+   在 Go 中编写 GraphQL 服务器

# 简介

本章将讨论 Go 中的响应式编程设计模式。响应式编程是一种编程概念，它关注数据流和变化的传播（[`en.wikipedia.org/wiki/Reactive_programming`](https://en.wikipedia.org/wiki/Reactive_programming)）。像 Kafka 这样的技术允许你快速产生或消费数据流。因此，这些技术彼此之间是自然匹配的。在 *将 Kafka 连接到 Goflow* 的配方中，我们将探讨将 `kafka` 消息队列与 `goflow` 结合起来，以展示使用这些技术的实际示例。本章还将探讨与 Kafka 连接的各种方式，并使用它来处理消息。最后，本章将演示如何在 Go 中创建一个基本的 `graphql` 服务器。

# Goflow 用于数据流编程

`github.com/trustmaster/goflow` 包对于创建基于数据流的应用程序很有用。它试图抽象概念，以便您可以使用自定义网络编写组件并将它们连接起来。这个配方将重新创建第八章 Testing 中讨论的应用程序，但它将使用 `goflow` 包来完成。

# 准备工作

根据以下步骤配置您的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到您的操作系统上，并配置您的 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序。

1.  导航到您的 `GOPATH/src` 并创建一个项目目录，例如，`$GOPATH/src/github.com/yourusername/customrepo`。所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

1.  运行 `go get github.com/trustmaster/goflow` 命令。

# 如何做到这一点...

这些步骤涵盖了编写和运行您的应用程序：

1.  在您的终端/控制台应用程序中创建 `chapter11/goflow` 目录并进入它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter11/goflow`](https://github.com/agtorre/go-cookbook/tree/master/chapter11/goflow) 复制测试或将其用作练习来编写您自己的测试。

1.  创建一个名为 `components.go` 的文件，内容如下：

```go
        package goflow

        import (
            "encoding/base64"
            "fmt"
            flow "github.com/trustmaster/goflow"
        )

        // Encoder base64 encodes all input
        type Encoder struct {
            flow.Component
            Val <-chan string
            Res chan<- string
        }

        // OnVal does the encoding then pushes the result onto Re
        func (e *Encoder) OnVal(val string) {
            encoded := base64.StdEncoding.EncodeToString([]byte(val))
            e.Res <- fmt.Sprintf("%s => %s", val, encoded)
        }

        // Printer is a component for printing to stdout
        type Printer struct {
            flow.Component
            Line <-chan string
        }

        // OnLine Prints the current line received
        func (p *Printer) OnLine(line string) {
            fmt.Println(line)
        }

```

1.  创建一个名为 `network.go` 的文件，内容如下：

```go
        package goflow

        import flow "github.com/trustmaster/goflow"

        // EncodingApp creates a flow-based
        // pipeline to encode and print the
        // result
        type EncodingApp struct {
            flow.Graph
        }

        // NewEncodingApp wires together the components
        func NewEncodingApp() *EncodingApp {
            e := &EncodingApp{}
            e.InitGraphState()

            // define component types
            e.Add(&Encoder{}, "encoder")
            e.Add(&Printer{}, "printer")

            // connect the components using channels
            e.Connect("encoder", "Res", "printer", "Line")

            // map the in channel to Val, which is
            // tied to OnVal function
            e.MapInPort("In", "encoder", "Val")

            return e
        }

```

1.  创建一个名为 `example` 的新目录并进入它。

1.  创建一个名为 `main.go` 的文件，内容如下。确保将 `goflow` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter11/goflow"
            flow "github.com/trustmaster/goflow"
        )

        func main() {

            net := goflow.NewEncodingApp()

            in := make(chan string)
            net.SetInPort("In", in)

            flow.RunNet(net)

            for i := 0; i < 20; i++ {
                in <- fmt.Sprint("Message", i)
            }

            close(in)
            <-net.Wait()
        }

```

1.  运行 `go run main.go`。

1.  您还可以运行以下命令：

```go
 go build ./example

```

您现在应该看到以下输出：

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

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`github.com/trustmaster/goflow` 包通过定义一个网络/图、注册一些组件，然后将它们连接起来来工作。由于这些是用字符串描述的，所以这可能会感觉有点容易出错，但通常在运行时设置正确之前，这会在早期失败。

在这个食谱中，我们设置了两个组件，一个用于对传入的字符串进行 base64 编码，另一个用于打印传递给它的任何内容。我们将其连接到一个在 `main.go` 中初始化的输入通道，任何传递到该通道的内容都将通过我们的管道流动。

这种方法的重点很多是忽略正在发生的事情的内部。我们像对待一个连接的黑盒一样对待一切，让 `goflow` 做其余的工作。您可以从这个食谱中看到完成这个任务管道所需的代码有多小，以及我们控制工人数量的旋钮更少，等等。

# 使用 RxGo 进行响应式编程

ReactiveX ([`reactivex.io/`](http://reactivex.io/)) 是一个用于使用可观察流进行编程的 API。RxGo ([github.com/reactivex/rxgo](http://github.com/reactivex/rxgo)) 是一个库，用于在 Go 中支持这种模式。它帮助您将应用程序视为一个大的事件流，当这些事件发生时，它会以不同的方式做出响应。这个食谱将创建一个使用这种方法处理不同葡萄酒的应用程序。理想情况下，这种方法可以与葡萄酒数据或葡萄酒 API 相关联，并汇总有关葡萄酒的信息。

# 准备就绪

根据以下步骤配置您的环境：

1.  参考本章中 *Goflow for dataflow programming* 食谱的 *准备就绪* 部分。

1.  运行 `go get github.com/reactivex/rxgo` 命令。

# 如何做...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建 `chapter11/reactive` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter11/reactive`](https://github.com/agtorre/go-cookbook/tree/master/chapter11/reactive) 复制测试或将其作为练习编写一些自己的测试。

1.  创建一个名为 `wine.go` 的文件，内容如下：

```go
        package reactive

        // Wine represents a bottle
        // of wine and is our
        // input stream
        type Wine struct {
            Name string
            Age int
            Rating float64 // 1-5
        }

        // GetWine returns an array of wines,
        // ages, and ratings
        func GetWine() interface{} {
            // some example wines
            w := []interface{}{
                Wine{"Merlot", 2011, 3.0},
                Wine{"Cabernet", 2010, 3.0},
                Wine{"Chardonnay", 2010, 4.0},
                Wine{"Pinot Grigio", 2009, 4.5},
            }
            return w
        }

        // Results holds a list of results by age
        type Results map[int]Result

        // Result is used for aggregation
        type Result struct {
            SumRating float64
            NumSamples int
        }

```

1.  创建一个名为 `exec.go` 的文件，内容如下：

```go
        package reactive

        import (
            "github.com/reactivex/rxgo/iterable"
            "github.com/reactivex/rxgo/observable"
            "github.com/reactivex/rxgo/observer"
            "github.com/reactivex/rxgo/subscription"
        )

        // Exec connects rxgo and returns
        // our results side-effect + a subscription
        // channel to block on at the end
        func Exec() (Results, <-chan subscription.Subscription) {
            results := make(Results)
            watcher := observer.Observer{
                NextHandler: func(item interface{}) {
                    wine, ok := item.(Wine)
                    if ok {
                        result := results[wine.Age]
                        result.SumRating += wine.Rating
                        result.NumSamples++
                        results[wine.Age] = result
                    }
                },
            }
            wine := GetWine()
            it, _ := iterable.New(wine)

            source := observable.From(it)
            sub := source.Subscribe(watcher)

            return results, sub
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，内容如下。确保您将 `reactive` 导入修改为在步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter11/reactive"
        )

        func main() {
            results, sub := reactive.Exec()

            // wait for the channel to emit a Subscription
            <-sub

            // process results
            for key, val := range results {
                fmt.Printf("Age: %d, Sample Size: %d, Average Rating: 
                %.2f\n", key, val.NumSamples, 
                val.SumRating/float64(val.NumSamples))
            }
        }

```

1.  运行 `go run main.go`。

1.  您还可以运行以下命令：

```go
 go build ./example

```

您现在应该看到以下内容：

```go
 $ go run main.go
 Age: 2011, Sample Size: 1, Average Rating: 3.00
 Age: 2010, Sample Size: 2, Average Rating: 3.50
 Age: 2009, Sample Size: 1, Average Rating: 4.50

```

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

RxGo 通过抽象源流（可以是数组或通道）来工作，允许你聚合流，并最终创建处理事件的观察者。这些可以处理错误或数据。RxGo 使用 `interface{}` 类型作为其参数，这样你可以传递任意值。因此，你必须使用反射来将传入的数据转换为正确的类型。如果你需要在观察者上返回错误，这可能很棘手。此外，增加的反射可能在性能上造成开销。

最后，你必须修改一些共享状态，无论是全局的还是局部闭包内的，这些状态将在最后使用。在我们的例子中，我们有一个 `Results` 类型，它是一个键为年份、值为总分和样本数量的映射。这使我们能够发出关于每年的平均值。如果我们使用酒名而不是类型，我们也可以按类型进行聚合。这个库还处于早期阶段。在许多方面，你可以使用基本的 Go 通道达到相同的效果。这有助于说明这些想法如何转化为 Go。

# 使用 Sarama 与 Kafka

Kafka 是一个流行的分布式消息队列，具有许多用于构建分布式系统的先进功能。本菜谱将展示如何使用同步生产者写入 Kafka 主题，以及如何使用分区消费者消费相同的主题。本菜谱不会探讨 Kafka 的不同配置，因为这是一个更广泛的话题，但我建议从 [`kafka.apache.org/intro`](https://kafka.apache.org/intro) 开始。

# 准备工作

根据以下步骤配置你的环境：

1.  参考本章中 *Goflow for dataflow programming* 菜单的 *Getting ready* 部分。

1.  使用以下步骤安装 Kafka：[`www.tutorialspoint.com/apache_kafka/apache_kafka_installation_steps.htm`](https://www.tutorialspoint.com/apache_kafka/apache_kafka_installation_steps.htm)。

1.  或者，你也可以访问 [`github.com/spotify/docker-kafka`](https://github.com/spotify/docker-kafka)。

1.  运行 `go get gopkg.in/Shopify/sarama.v1` 命令。

# 如何做到这一点...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端/控制台应用程序中创建 `chapter11/synckafka` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter11/synckafka`](https://github.com/agtorre/go-cookbook/tree/master/chapter11/synckafka) 复制测试或将其作为练习来编写一些自己的测试。

1.  确保 Kafka 在 `localhost:9092` 上运行。

1.  在名为 `consumer` 的目录中创建一个名为 `main.go` 的文件，内容如下：

```go
        package main

        import (
            "log"

            sarama "gopkg.in/Shopify/sarama.v1"
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

1.  在名为 `producer` 的目录中创建一个名为 `main.go` 的文件，内容如下：

```go
        package main

        import (

           "fmt"
           "log"

            sarama "gopkg.in/Shopify/sarama.v1"
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

1.  运行 `go run consumer/main.go`。

1.  在另一个终端中运行 `go run producer/main.go`。

1.  在生产者终端中，你应该看到以下内容：

```go
 $ go run producer/main.go 
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

1.  在消费者终端中，你应该看到以下内容：

```go
 $ go run consumer/main.go 
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

1.  如果你复制或编写了自己的测试，向上导航一个目录并运行 `go test`。确保所有测试通过。

# 它是如何工作的...

这个食谱演示了通过 Kafka 传递简单消息。更复杂的方法应使用如 `json`、`gob`、`protobuf` 或其他序列化格式。生产者可以通过 `sendMessage` 同步地将消息发送到 Kafka。这并不很好地处理 Kafka 集群宕机的情况，可能会导致进程挂起。对于像网络处理器这样的应用程序来说，这是一个重要的考虑因素，因为它可能导致超时和对 Kafka 集群的硬依赖。

假设消息队列正确无误，我们的消费者将观察 Kafka 流并处理结果。本章前面的食谱可能已经使用此流进行一些额外的处理。

# 使用 Kafka 的异步生产者

在进行下一项任务之前，通常不需要等待 Kafka 生产者完成。在这种情况下，你可以使用异步生产者。这些生产者从 Sarama 消息通道接收消息，并具有返回可以单独检查的成功/错误通道的方法。

在这个食谱中，我们将创建一个 go 线程来处理成功和失败的消息，同时允许处理器在结果未知的情况下排队发送消息。

# 准备阶段

参考使用 Sarama 与 Kafka 的 *准备阶段* 部分。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter11/asyncsarama` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter11/asyncsarama`](https://github.com/agtorre/go-cookbook/tree/master/chapter11/asyncsarama) 复制测试或将其作为练习来编写你自己的测试。

1.  确保 Kafka 在 `localhost:9092` 上运行。

1.  从上一个食谱复制消费者目录。

1.  创建一个名为 `producer` 的目录并导航到它。

1.  创建一个名为 `producer.go` 的文件：

```go
        package main

        import (
            "log"

            sarama "gopkg.in/Shopify/sarama.v1"
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

1.  创建一个名为 `handler.go` 的文件：

```go
        package main

        import (
            "net/http"

            sarama "gopkg.in/Shopify/sarama.v1"
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
            sarama.StringEncoder(r.FormValue("msg"))}
            w.WriteHeader(http.StatusOK)
        }

```

1.  创建一个名为 `main.go` 的文件：

```go
        package main

        import (
            "fmt"
            "net/http"

            sarama "gopkg.in/Shopify/sarama.v1"
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

1.  运行 `go build` 命令。

1.  向上导航一个目录。

1.  运行 `go run consumer/main.go`。

1.  在同一目录下的另一个终端中，运行 `./producer/producer`。

1.  在第三个终端中，运行以下命令：

```go
 $ curl "http://localhost:3333/?msg=this" 
 $ curl "http://localhost:3333/?msg=is" 
 $ curl "http://localhost:3333/?msg=an" 
 $ curl "http://localhost:3333/?msg=example" 

```

在生产者终端中，你应该看到以下内容：

```go
 $ ./producer/producer 
 Listening on port :3333
 2017/05/07 13:52:54 > message: "this" sent to partition 0 at 
 offset 0
 2017/05/07 13:53:25 > message: "is" sent to partition 0 at offset 
 1
 2017/05/07 13:53:27 > message: "an" sent to partition 0 at offset 
 2
 2017/05/07 13:53:29 > message: "example" sent to partition 0 at 
 offset 3

```

1.  在消费者终端中，你应该看到以下内容：

```go
 $ go run consumer/main.go 
 2017/05/07 13:52:54 Consumed message: "this" at offset: 0
 2017/05/07 13:53:25 Consumed message: "is" at offset: 1
 2017/05/07 13:53:27 Consumed message: "an" at offset: 2
 2017/05/07 13:53:29 Consumed message: "example" at offset: 3

```

1.  如果你复制或编写了自己的测试，向上导航一个目录并运行 `go test`。确保所有测试通过。

# 它是如何工作的...

本章的所有修改都是针对生产者进行的。这次，我们创建了一个单独的 go 线程来处理成功和错误。如果这些没有被处理，你的应用程序将发生死锁。接下来，我们将我们的生产者附加到处理器，并在处理器通过 `GET` 调收到的消息时在其上发出消息。

无论响应如何，处理程序在发送消息后会立即返回成功。如果这不可接受，应使用同步方法。在我们的情况下，我们接受稍后分别处理成功和错误。

最后，我们使用几个不同的消息 curl 我们的端点，您可以看到它们从处理程序流向我们之前章节中编写的 Kafka 消费者最终打印的地方。

# 将 Kafka 连接到 Goflow

这个配方将结合一个 Kafka 消费者和一个 Goflow 管道。随着我们的消费者从 Kafka 接收消息，它将对它们运行`strings.ToUpper()`，然后打印结果。这些自然地配对，因为 Goflow 被设计为在传入的流上操作，这正是 Kafka 为我们提供的。

# 准备就绪

参考配方*使用 Sarama 的 Kafka*中的*准备就绪*部分*.*。

# 如何做到这一点...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建`chapter11/kafkaflow`目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter11/kafkaflow`](https://github.com/agtorre/go-cookbook/tree/master/chapter11/kafkaflow)复制测试或将其用作练习来编写一些自己的测试。

1.  确保 Kafka 在`localhost:9092`上运行。

1.  创建一个名为`components.go`的文件，内容如下：

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
            flow.Component
            Val <-chan string
            Res chan<- string
        }

        // OnVal does the encoding then pushes the result onto Re
        func (e *Upper) OnVal(val string) {
            e.Res <- strings.ToUpper(val)
        }

        // Printer is a component for printing to stdout
        type Printer struct {
            flow.Component
            Line <-chan string
        }

        // OnLine Prints the current line received
        func (p *Printer) OnLine(line string) {
            fmt.Println(line)
        }

```

1.  创建一个名为`network.go`的文件，内容如下：

```go
        package kafkaflow

        import flow "github.com/trustmaster/goflow"

        // UpperApp creates a flow-based
        // pipeline to upper case and print the
        // result
        type UpperApp struct {
            flow.Graph
        }

        // NewUpperApp wires together the compoents
        func NewUpperApp() *UpperApp {
            u := &UpperApp{}
            u.InitGraphState()

            u.Add(&Upper{}, "upper")
            u.Add(&Printer{}, "printer")

            u.Connect("upper", "Res", "printer", "Line")
            u.MapInPort("In", "upper", "Val")

            return u
        }

```

1.  在名为`consumer`的目录中创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "github.com/agtorre/go-cookbook/chapter11/kafkaflow"
            flow "github.com/trustmaster/goflow"
            sarama "gopkg.in/Shopify/sarama.v1"
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

            net := kafkaflow.NewUpperApp()

            in := make(chan string)
            net.SetInPort("In", in)

            flow.RunNet(net)
            defer func() {
                close(in)
                <-net.Wait()
            }()

            for {
                msg := <-partitionConsumer.Messages()
                in <- string(msg.Value)
            }
        }

```

1.  从*使用 Sarama 的 Kafka*配方复制消费者目录。

1.  运行`go run consumer/main.go`。

1.  在另一个终端中运行`go run producer/main.go`。

1.  在生产者终端中，您现在应该看到以下内容：

```go
 $ go run producer/main.go 
 go run producer/main.go !3300
 2017/05/07 18:24:12 > message "Message 0" sent to partition 0 at 
 offset 0
 2017/05/07 18:24:12 > message "Message 1" sent to partition 0 at 
 offset 1
 2017/05/07 18:24:12 > message "Message 2" sent to partition 0 at 
 offset 2
 2017/05/07 18:24:12 > message "Message 3" sent to partition 0 at 
 offset 3
 2017/05/07 18:24:12 > message "Message 4" sent to partition 0 at 
 offset 4
 2017/05/07 18:24:12 > message "Message 5" sent to partition 0 at 
 offset 5
 2017/05/07 18:24:12 > message "Message 6" sent to partition 0 at 
 offset 6
 2017/05/07 18:24:12 > message "Message 7" sent to partition 0 at 
 offset 7
 2017/05/07 18:24:12 > message "Message 8" sent to partition 0 at 
 offset 8
 2017/05/07 18:24:12 > message "Message 9" sent to partition 0 at 
 offset 9

```

在消费者终端中，您应该看到以下内容：

```go
 $ go run consumer/main.go 
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

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方结合了本章中先前配方中的想法。像之前的配方一样，我们设置了一个 Kafka 消费者和生产者。这个配方使用了*使用 Sarama 的 Kafka*配方中的同步生产者，但也可以使用异步生产者。一旦收到消息，我们就像在*Goflow 数据流编程*配方中做的那样，将其入队到输入通道中。我们修改了这个配方中的组件，将传入的字符串转换为大写，而不是进行 base64 编码。我们重用了打印组件，并且最终的网络配置相似。

最终结果是，通过 Kafka 消费者接收的所有消息都被传输到我们的基于流的作业管道中进行操作。这使我们能够将管道组件模块化和可重用，我们可以在不同的配置中使用相同的组件多次。同样，我们将接收任何写入 Kafka 的生产者的流量，因此我们可以将生产者多路复用到单个数据流中。

# 使用 Go 编写 GraphQL 服务器

GraphQL 是 Facebook 创建的 REST 的替代品 ([`graphql.org/`](http://graphql.org/))。这项技术允许服务器实现并发布一个模式，然后客户端可以请求他们所需的信息，而不是理解和利用各种 API 端点。

对于这个菜谱，我们将创建一个 `Graphql` 模式，它代表一副扑克牌。我们将公开一个资源卡片，可以根据花色和值进行过滤。如果没有指定参数，它还可以返回牌组中的所有卡片。

# 准备就绪

根据以下步骤配置你的环境：

1.  参考本章中 *Goflow for dataflow programming* 菜单的 *准备就绪* 部分。

1.  运行 `go get github.com/graphql-go/graphql` 命令。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中创建 `chapter11/graphql` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter11/graphql`](https://github.com/agtorre/go-cookbook/tree/master/chapter11/graphql) 复制测试或将其作为练习编写一些自己的测试。

1.  创建并导航到 `cards` 目录。

1.  创建一个名为 `card.go` 的文件，内容如下：

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

1.  创建一个名为 `type.go` 的文件：

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

1.  创建一个名为 `resolve.go` 的文件：

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

1.  创建一个名为 `schema.go` 的文件：

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

1.  返回到 `graphql` 目录。

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，内容如下。确保你修改 `cards` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "encoding/json"
            "fmt"
            "log"

            "github.com/agtorre/go-cookbook/chapter11/graphql/cards"
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
            }
 `
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

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

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

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`cards.go` 文件定义了一个 `card` 对象，并在名为 `cards` 的全局变量中初始化基础牌组。这种状态也可以存储在长期存储中，如数据库。然后我们在 `types.go` 中定义 `CardType`，允许 `graphql` 将卡片对象解析为响应。接下来，我们跳转到 `resolve.go`，在那里我们定义如何根据值和类型过滤卡片。这个 `Resolve` 函数将被最终的模式使用，该模式在 `schema.go` 中定义。

例如，你可能需要修改这个菜谱中的 `Resolve` 函数以从数据库中检索数据。最后，我们加载模式并对它运行查询。将我们的模式挂载到 REST 端点是一个小的修改，但为了简洁，这个菜谱只是运行了一个硬编码的查询。有关 `GraphQL` 查询的更多信息，请访问 [`graphql.org/learn/queries/`](http://graphql.org/learn/queries/)。
