# 第十章。高级并发和最佳实践

一旦您熟悉了 Go 中并发特性的基本和中级用法，您可能会发现您能够使用双向通道和标准并发工具处理大多数开发用例。

在第二章*理解并发模型*和第三章*制定并发策略*中，我们不仅看了 Go 的并发模型，还比较了其他语言的并发模型以及分布式模型的工作方式。在本章中，我们将涉及这些内容以及一些关于设计和管理并发应用程序的更高级别概念。

特别是，我们将研究 goroutine 及其相关通道的集中管理——通常情况下，您可能会发现 goroutine 是一种设置并忘记的命题；然而，在某些情况下，我们可能希望更精细地控制通道的状态。

我们还从高层次上看了测试和基准测试，但我们将研究一些更详细和复杂的测试方法。我们还将探讨一些关于 Google App Engine 的入门知识，这将使我们能够使用一些特定的测试工具。

最后，我们将涉及一些 Go 的一般最佳实践，这不仅适用于并发应用程序设计，而且适用于您将来在语言中的工作。

# 超越基础的通道使用

我们已经讨论了许多不同类型的通道实现——不同类型的通道（接口、函数、结构体和通道），并且涉及了缓冲和非缓冲通道的区别。然而，我们在设计和流程中仍然可以做很多事情。

从设计上讲，Go 希望您保持简单。这对您在使用 Go 时的 90%的工作来说是非常棒的。但是还有其他时候，您需要深入挖掘解决方案，或者需要通过保留开放的 goroutine 进程、通道等资源来节省资源。

在某些时候，您可能希望对 goroutine 的大小和状态进行一些手动控制，以及对正在运行或关闭的 goroutine 进行控制，因此我们将研究如何做到这一点。

同样重要的是，将您的 goroutine 设计与整个应用程序设计协同工作对于单元测试至关重要，这是我们将在本章中涉及的一个主题。

# 构建工作线程

在本书的前面，我们谈到了并发模式和一些关于工作线程的内容。甚至在上一章中，当我们构建日志系统时，我们也介绍了工作线程的概念。

说实话，“工作线程”是一个相当通用和模糊的概念，不仅在 Go 中，而且在一般的编程和开发中也是如此。在某些语言中，它是一个对象/实例化类，而在其他语言中它是一个并发的执行者。在函数式编程语言中，工作线程是传递给另一个函数的返回值。

如果我们回到前言，我们会看到我们确实将 Go gopher 用作工作线程的示例。简而言之，工作线程是一个比单个函数调用或编程动作更复杂的东西，它将执行一个或多个任务。

那么为什么我们现在要谈论它呢？当我们构建通道时，我们正在创建一个执行工作的机制。当我们有一个结构体或一个接口时，我们将方法和值组合在一个地方，然后使用该*对象*来执行工作，同时也作为存储有关该工作的信息的地方。

这在应用程序设计中特别有用，因为我们能够将应用程序功能的各个元素委托给独立和明确定义的工作线程。例如，考虑一个服务器 ping 应用程序，其中具体的部分以自包含、分隔的方式执行特定的任务。

我们将尝试通过 HTTP 包检查服务器的可用性，检查状态代码和错误，并在发现任何特定服务器存在问题时进行退避。您可能已经看出这将导致什么 - 这是负载平衡的最基本方法。但一个重要的设计考虑是我们管理通道的方式。

我们将有一个主通道，所有重要的全局事务都应该在这里累积和评估，但每个单独的服务器也将有自己的通道，用于处理只对该单独结构重要的任务。

以下代码中的设计可以被视为一个基本的管道，大致相当于我们在前几章中讨论的生产者/消费者模型：

```go
package main

import
(
  "fmt"
  "time"
  "net/http"
)

const INIT_DELAY = 3000
const MAX_DELAY = 60000
const MAX_RETRIES = 4
const DELAY_INCREMENT = 5000
```

前面的代码给出了应用程序的配置部分，设置了多久检查服务器、备份的最长时间和在完全放弃之前重试的最大次数。

`DELAY_INCREMENT`值表示每次发现问题时我们将为服务器检查过程添加多少时间。让我们看看如何在以下部分创建服务器：

```go
var Servers []Server

type Server struct {
  Name string
  URI string
  LastChecked time.Time
  Status bool
  StatusCode int
  Delay int
  Retries int
  Channel chan bool
}
```

现在，我们设计基本的服务器（使用以下代码），其中包含其当前状态、上次检查时间、检查之间的延迟、用于评估状态和建立新状态的自己的通道，以及更新的重试延迟：

```go
func (s *Server) checkServerStatus(sc chan *Server) {
  var previousStatus string

    if s.Status == true {
      previousStatus = "OK"
    }else {
      previousStatus = "down"
    }

    fmt.Println("Checking Server",s.Name)
    fmt.Println("\tServer was",previousStatus,"on last check at",s.LastChecked)

    response, err := http.Get(s.URI)
    if err != nil {
      fmt.Println("\tError: ",err)
      s.Status = false
      s.StatusCode = 0
    }else {
      fmt.Println(response.Status)
      s.StatusCode = response.StatusCode
      s.Status = true
    }

    s.LastChecked = time.Now()
    sc <- s
}
```

`checkServerStatus()`方法是我们应用程序的核心。我们在`main()`函数中通过`cycleServers()`循环将所有服务器传递到这个方法中，之后它就变得自我实现了。

如果我们的`Status`设置为`true`，我们将状态发送到控制台作为`OK`（否则为`down`），并使用`s.StatusCode`设置我们的`Server`状态代码，作为 HTTP 代码或者如果有网络或其他错误则为`0`。

最后，将`Server`的上次检查时间设置为`Now()`，并通过`serverChan`通道传递`Server`。在以下代码中，我们将演示如何循环遍历我们可用的服务器：

```go
func cycleServers(sc chan *Server) {

  for i := 0; i < len(Servers); i++ {
    Servers[i].Channel = make(chan bool)
    go Servers[i].updateDelay(sc)
    go Servers[i].checkServerStatus(sc)
  }

}
```

这是我们的初始循环，从主函数调用。它只是循环遍历我们可用的服务器，并初始化其监听 goroutine，以及发送第一个`checkServerStatus`请求。

这里值得注意两件事：首先，由`Server`调用的通道实际上永远不会死，而是应用程序将停止检查服务器。这对于这里的所有实际目的来说都是可以的，但是如果我们有成千上万台服务器要检查，我们会浪费资源，因为本质上相当于一个未关闭的通道和一个未被移除的映射元素。稍后，我们将讨论手动终止 goroutines 的概念，这是我们只能通过停止通信通道来抽象实现的。现在让我们来看一下控制服务器状态及其下一步的以下代码：

```go
func (s *Server) updateDelay(sc chan *Server) {
  for {
    select {
      case msg := <- s.Channel:

        if msg == false {
          s.Delay = s.Delay + DELAY_INCREMENT
          s.Retries++
          if s.Delay > MAX_DELAY {
            s.Delay = MAX_DELAY
          }

        }else {
          s.Delay = INIT_DELAY
        }
        newDuration := time.Duration(s.Delay)

        if s.Retries <= MAX_RETRIES {
          fmt.Println("\tWill check server again")
          time.Sleep(newDuration * time.Millisecond)
          s.checkServerStatus(sc)
        }else {
          fmt.Println("\tServer not reachable after",MAX_RETRIES,"retries")
        }

      default:
    }
  }
}
```

这是每个`Server`将监听其状态变化的地方，由`checkServerStatus()`报告。当任何给定的`Server`结构接收到通过我们的初始循环报告状态变化的消息时，它将评估该消息并相应地采取行动。

如果`Status`设置为`false`，我们知道服务器由于某种原因无法访问。然后`Server`引用本身将延迟到下次检查的时间。如果设置为`true`，则服务器是可访问的，延迟将被设置或重置为`INIT_DELAY`的默认重试值。

最后，在重新初始化`checkServerStatus()`方法之前，它会在 goroutine 上设置睡眠模式，并在`main()`函数中的初始 goroutine 循环中传递`serverChan`引用：

```go
func main() {

  endChan := make(chan bool)
  serverChan := make(chan *Server)

Servers = []Server{ {Name: "Google", URI: "http://www.google.com", Status: true, Delay: INIT_DELAY}, {Name: "Yahoo", URI: "http://www.yahoo.com", Status: true, Delay: INIT_DELAY}, {Name: "Bad Amazon", URI: "http://amazon.zom", Status: true, Delay: INIT_DELAY} }
```

在这里有一个快速的说明 - 在我们的`Servers`切片中，我们故意在最后一个元素中引入了一个拼写错误。您会注意到`amazon.zom`，这将在`checkServerStatus()`方法中引发一个 HTTP 错误。以下是循环遍历服务器以找到合适匹配的函数：

```go
  go cycleServers(serverChan)

  for {
    select {
      case currentServer := <- serverChan:
        currentServer.Channel <- false
      default:

    }
  }

  <- endChan

}
```

以下是包含拼写错误的输出示例：

```go
Checking Server Google
 Server was OK on last check at 0001-01-01 00:00:00 +0000 UTC
 200 OK
 Will check server again
Checking Server Yahoo
 Server was OK on last check at 0001-01-01 00:00:00 +0000 UTC
 200 OK
 Will check server again
Checking Server Amazon
 Server was OK on last check at 0001-01-01 00:00:00 +0000 UTC
 Error:  Get http://amazon.zom: dial tcp: GetAddrInfoW: No such host is known.
 Will check server again
Checking Server Google
 Server was OK on last check at 2014-04-23 12:49:45.6575639 -0400 EDT

```

我们将在本章的后面通过一些并发模式再次运行前面的代码，将其转换为更实用的东西。

# 实现 nil 通道阻塞

设计管道或生产者/消费者模型等东西的一个更大的问题是，在任何给定时间，任何给定 goroutine 的状态都有点像黑洞。

考虑以下循环，在该循环中，生产者通道创建一组任意的消费者通道，并期望每个通道只执行一项任务：

```go
package main

import (
  "fmt"
  "time"
)

const CONSUMERS = 5

func main() {

  Producer := make(chan (chan int))

  for i := 0; i < CONSUMERS; i++ {
    go func() {
      time.Sleep(1000 * time.Microsecond)
      conChan := make(chan int)

      go func() {
        for {
          select {
          case _,ok := <-conChan:
            if ok  {
              Producer <- conChan
            }else {
              return
            }
          default:
          }
        }
      }()

      conChan <- 1
      close(conChan)
    }()
  }
```

给定要生成的消费者的随机数量，我们为每个消费者附加一个通道，并通过该消费者的通道将消息传递到`Producer`上游。我们只发送一条消息（我们可以使用缓冲通道处理），但是在发送完消息后我们简单地关闭通道。

无论是在多线程应用程序、分布式应用程序还是高并发应用程序中，生产者-消费者模型的一个基本属性是数据能够以稳定、可靠的方式在队列/通道之间移动。这需要在生产者和消费者之间共享一些相互的知识。

与分布式（或多核）环境不同，我们确实具有对该安排两端状态的某种固有意识。接下来我们将看一下生产者消息的监听循环：

```go
  for {
    select {
    case consumer, ok := <-Producer:
      if ok == false {
        fmt.Println("Goroutine closed?")
        close(Producer)
      } else {
        log.Println(consumer)
        // consumer <- 1
      }
      fmt.Println("Got message from secondary channel")
    default:
    }
  }
}
```

主要问题是`Producer`通道之一对于任何给定的`Consumer`并不了解太多，包括它何时处于活动状态。如果我们取消注释`// consumer <- 1`行，我们将会得到一个恐慌，因为我们试图在关闭的通道上发送消息。

当消息通过次要 goroutine 的通道上游传递到`Producer`的通道时，我们会得到适当的接收，但无法检测下游 goroutine 何时关闭。

在许多情况下，知道 goroutine 何时终止并不重要，但是考虑一个应用程序，在完成一定数量的任务时生成新的 goroutines，有效地将一个任务分解成小任务。也许每个块都依赖于上一个块的总体完成情况，并且广播器必须在继续之前了解当前 goroutines 的状态。

## 使用 nil 通道

在 Go 的早期版本中，您可以在未初始化的、因此为 nil 或 0 值的通道之间进行通信而不会引发恐慌（尽管结果可能是不可预测的）。从 Go 版本 1 开始，跨 nil 通道的通信产生了一种一致但有时令人困惑的效果。

需要注意的是，在 select 开关内，单独在 nil 通道上进行传输仍会导致死锁和恐慌。这在使用全局通道并且从未正确初始化它们时最常见。以下是在 nil 通道上进行传输的示例：

```go
func main() {

  var channel chan int

    channel <- 1

  for {
    select {
      case <- channel:

      default:
    }
  }

}
```

由于通道设置为其`0`值（在这种情况下为 nil），它会永久阻塞，Go 编译器将会检测到这一点，至少在较新的版本中是如此。您还可以在`select`语句之外复制这一点，如下面的代码所示：

```go
  var done chan int
  defer close(done)
  defer log.Println("End of script")
  go func() {
    time.Sleep(time.Second * 5)
    done <- 1
  }()

  for {
    select {
      case <- done:
        log.Println("Got transmission")
        return
      default:
    }
  }
```

由于`select`语句中的默认值使主循环在等待通道上的通信时保持活动状态，所以前面的代码将永远阻塞，没有恐慌。然而，如果我们初始化通道，应用程序将如预期般运行。

在这两种边缘情况——关闭的通道和 nil 通道中，我们需要一种方法让主通道了解 goroutine 的状态。

# 使用 tomb 实现对 goroutines 的更精细控制

与许多这类问题一样——无论是小众还是常见的——存在第三方实用程序，可以抓住你的 goroutines。

Tomb 是一个库，它提供了与任何 goroutine 和通道一起使用的诊断功能——它可以告诉主通道另一个 goroutine 是死了还是快死了。

此外，它允许您明确地终止 goroutine，这比简单关闭它附加的通道更微妙。如前所述，关闭通道实际上是使 goroutine 失效，尽管它最终仍然可能是活动的。

您将找到一个简单的获取和抓取主体脚本，它接受 URL 结构的切片（带有状态和 URI），并尝试获取每个 URL 的 HTTP 响应并将其应用于结构。但是，我们不仅仅报告 goroutines 的信息，还可以向“主”结构的每个子 goroutine 发送“kill 消息”的能力。

在这个例子中，我们将运行脚本 10 秒，如果任何 goroutine 在分配的时间内未能完成其工作，它将响应说由于主结构发送的 kill 命令而无法获取 URL 的主体：

```go
package main

import (
  "fmt"
  "io/ioutil"
  "launchpad.net/tomb"
  "net/http"
  "strconv"
  "sync"
  "time"
)

var URLS []URL

type GoTomb struct {
  tomb tomb.Tomb
}
```

这是创建所有生成的 goroutines 的父结构或主结构所需的最低必要结构。`tomb.Tomb`结构只是一个互斥体，两个通道（一个用于死亡和一个用于垂死），以及一个原因错误结构。`URL`结构的结构如下代码所示：

```go
type URL struct {
  Status bool
  URI    string
  Body   string
}
```

我们的`URL`结构相当基本——`Status`默认设置为`false`，当主体被检索时设置为`true`。它包括`URI`变量——这是 URL 的引用——以及用于存储检索数据的`Body`变量。以下函数允许我们对`GoTomb`结构执行“kill”：

```go
func (gt GoTomb) Kill() {

  gt.tomb.Kill(nil)

}
```

前面的方法在我们的`GoTomb`结构上调用了`tomb.Kill`。在这里，我们将唯一的参数设置为`nil`，但这很容易改为一个更具描述性的错误，比如`errors.New("Time to die, goroutine")`。在这里，我们将展示`GoTomb`结构的监听器：

```go
func (gt *GoTomb) TombListen(i int) {

  for {
    select {
    case <-gt.tomb.Dying():
      fmt.Println("Got kill command from tomb!")
      if URLS[i].Status == false {
        fmt.Println("Never got data for", URLS[i].URI)
      }
      return
    }
  }
}
```

我们调用附加到我们的`GoTomb`的`TombListen`，它设置一个监听`Dying()`通道的选择，如下面的代码所示：

```go
func (gt *GoTomb) Fetch() {
  for i := range URLS {
    go gt.TombListen(i)

    go func(ii int) {

      timeDelay := 5 * ii
      fmt.Println("Waiting ", strconv.FormatInt(int64(timeDelay), 10), " seconds to get", URLS[ii].URI)
      time.Sleep(time.Duration(timeDelay) * time.Second)
      response, _ := http.Get(URLS[ii].URI)
      URLS[ii].Status = true
      fmt.Println("Got body for ", URLS[ii].URI)
      responseBody, _ := ioutil.ReadAll(response.Body)
      URLS[ii].Body = string(responseBody)
    }(i)
  }
}
```

当我们调用`Fetch()`时，我们还将 tomb 设置为`TombListen()`，它接收所有生成的 goroutines 的“主”消息。我们故意设置一个很长的等待时间，以确保我们最后几次尝试`Fetch()`在`Kill()`命令之后进行。最后，我们的`main()`函数，处理整体设置：

```go
func main() {

  done := make(chan int)

  URLS = []URL{{Status: false, URI: "http://www.google.com", Body: ""}, {Status: false, URI: "http://www.amazon.com", Body: ""}, {Status: false, URI: "http://www.ubuntu.com", Body: ""}}

  var MasterChannel GoTomb
  MasterChannel.Fetch()

  go func() {

    time.Sleep(10 * time.Second)
    MasterChannel.Kill()
    done <- 1
  }()

  for {
    select {
    case <-done:
      fmt.Println("")
      return
    default:
    }
  }
}
```

通过将`time.Sleep`设置为`10`秒，然后杀死我们的 goroutines，我们保证在被杀死之前，`Fetch()`之间的 5 秒延迟会阻止我们的最后几个 goroutines 成功完成。

### 提示

对于 tomb 包，请访问[`godoc.org/launchpad.net/tomb`](http://godoc.org/launchpad.net/tomb)并使用`go get launchpad.net/tomb`命令进行安装。

# 使用通道超时

关于通道和`select`循环的一个相当关键的点，我们还没有特别仔细地检查过，那就是在一定的超时之后终止`select`循环的能力和通常的必要性。

到目前为止，我们编写的许多应用程序都是长时间运行或永久运行的，但有时我们会希望对 goroutines 的运行时间设置有限的时间限制。

我们迄今为止使用的`for { select { } }`开关要么永久存在（带有默认情况），要么等待从一个或多个情况中退出。

有两种管理基于间隔的任务的方法——都作为时间包的一部分，这并不奇怪。

`time.Ticker`结构允许在指定的时间段之后执行任何给定的操作。它提供了一个 C，一个阻塞通道，可以用来检测在那段时间之后发送的活动；参考以下代码：

```go
package main

import (
  "log"
  "time"
)

func main() {

  timeout := time.NewTimer(5 * time.Second)
  defer log.Println("Timed out!")

  for {
    select {
    case <-timeout.C:
      return
    default:
    }
  }

}
```

我们可以扩展这个方法，在一定时间后结束通道和并发执行。看一下以下修改：

```go
package main

import (
  "fmt"
  "time"
)

func main() {

  myChan := make(chan int)

  go func() {
    time.Sleep(6 * time.Second)
    myChan <- 1
  }()

  for {
    select {
      case <-time.After(5 * time.Second):
        fmt.Println("This took too long!")
        return
      case <-myChan:
        fmt.Println("Too little, too late")
    }
  }
}
```

# 使用并发模式构建负载均衡器

当我们在本章前面构建我们的服务器 ping 应用程序时，很容易想象将其带到一个更可用和有价值的空间。

对于负载均衡器的健康检查，ping 服务器通常是第一步。正如 Go 提供了一个可用的开箱即用的 Web 服务器解决方案一样，它还提供了一个非常干净的`Proxy`和`ReverseProxy`结构和方法，使得创建负载均衡器变得非常简单。

当然，循环负载均衡器将需要大量的后台工作，特别是在更改请求之间的`ReverseProxy`位置时进行检查和重新检查。我们将使用每个请求触发的 goroutines 来处理这些。

最后，请注意我们在配置底部有一些虚拟 URL——将这些更改为生产 URL 应立即将运行此服务器的服务器转换为工作负载均衡器。让我们看一下应用程序的主要设置：

```go
package main

import (
  "fmt"
  "log"
  "net/http"
  "net/http/httputil"
  "net/url"
  "strconv"
  "time"
)

const MAX_SERVER_FAILURES = 10
const DEFAULT_TIMEOUT_SECONDS = 5
const MAX_TIMEOUT_SECONDS = 60
const TIMEOUT_INCREMENT = 5
const MAX_RETRIES = 5
```

在前面的代码中，我们定义了我们的常量，就像我们之前做的那样。我们有一个`MAX_RETRIES`，它限制了我们可以有多少次失败，`MAX_TIMEOUT_SECONDS`，它定义了我们在再次尝试之前等待的最长时间，以及我们的`TIMEOUT_INCREMENT`用于在失败之间更改该值。接下来，让我们看一下我们的`Server`结构的基本构造：

```go
type Server struct {
  Name        string
  Failures    int
  InService   bool
  Status      bool
  StatusCode  int
  Addr        string
  Timeout     int
  LastChecked time.Time
  Recheck     chan bool
}
```

正如我们在前面的代码中看到的，我们有一个通用的`Server`结构，用于维护当前状态、最后的状态代码以及上次检查服务器的时间的信息。

请注意，我们还有一个`Recheck`通道，触发延迟尝试检查`Server`是否再次可用。通过该通道传递的每个布尔值都将从可用池中删除服务器，或者重新宣布其仍在服务中：

```go
func (s *Server) serverListen(serverChan chan bool) {
  for {
    select {
    case msg := <-s.Recheck:
      var statusText string
      if msg == false {
        statusText = "NOT in service"
        s.Failures++
        s.Timeout = s.Timeout + TIMEOUT_INCREMENT
        if s.Timeout > MAX_TIMEOUT_SECONDS {
          s.Timeout = MAX_TIMEOUT_SECONDS
        }
      } else {
        if ServersAvailable == false {
          ServersAvailable = true
          serverChan <- true
        }
        statusText = "in service"
        s.Timeout = DEFAULT_TIMEOUT_SECONDS
      }

      if s.Failures >= MAX_SERVER_FAILURES {
        s.InService = false
        fmt.Println("\tServer", s.Name, "failed too many times.")
      } else {
        timeString := strconv.FormatInt(int64(s.Timeout), 10)
        fmt.Println("\tServer", s.Name, statusText, "will check again in", timeString, "seconds")
        s.InService = true
        time.Sleep(time.Second * time.Duration(s.Timeout))
        go s.checkStatus()
      }

    }
  }
}
```

这是一个实例化的方法，它在每个服务器上监听在任何给定时间服务器的可用性的消息。在运行 goroutine 的同时，我们保持一个永久监听通道，以便从“checkStatus（）”接收布尔响应。如果服务器可用，则下一个延迟设置为默认值；否则，将`TIMEOUT_INCREMENT`添加到延迟中。如果服务器失败次数太多，通过将其`InService`属性设置为`false`并不再调用“checkStatus（）”方法来将其从轮换中取出。接下来让我们看一下检查`Server`当前状态的方法：

```go
func (s *Server) checkStatus() {
  previousStatus := "Unknown"
  if s.Status == true {
    previousStatus = "OK"
  } else {
    previousStatus = "down"
  }
  fmt.Println("Checking Server", s.Name)
  fmt.Println("\tServer was", previousStatus, "on last check at", s.LastChecked)
  response, err := http.Get(s.Addr)
  if err != nil {
    fmt.Println("\tError: ", err)
    s.Status = false
    s.StatusCode = 0
  } else {
    s.StatusCode = response.StatusCode
    s.Status = true
  }

  s.LastChecked = time.Now()
  s.Recheck <- s.Status
}
```

我们的“checkStatus（）”方法应该看起来很熟悉，基于服务器 ping 示例。我们寻找服务器；如果它可用，我们向我们的`Recheck`通道传递`true`；否则传递`false`，如下面的代码所示：

```go
func healthCheck(sc chan bool) {
  fmt.Println("Running initial health check")
  for i := range Servers {
    Servers[i].Recheck = make(chan bool)
    go Servers[i].serverListen(sc)
    go Servers[i].checkStatus()
  }
}
```

我们的`healthCheck`函数只是启动每个服务器检查（和重新检查）其状态的循环。它只运行一次，并通过`make`语句初始化`Recheck`通道：

```go
func roundRobin() Server {
  var AvailableServer Server

  if nextServerIndex > (len(Servers) - 1) {
    nextServerIndex = 0
  }

  if Servers[nextServerIndex].InService == true {
    AvailableServer = Servers[nextServerIndex]
  } else {
    serverReady := false
    for serverReady == false {
      for i := range Servers {
        if Servers[i].InService == true {
          AvailableServer = Servers[i]
          serverReady = true
        }
      }

    }
  }
  nextServerIndex++
  return AvailableServer
}
```

`roundRobin`函数首先检查队列中的下一个可用`Server`——如果该服务器不可用，它将循环查找剩余的服务器，以找到第一个可用的`Server`。如果它循环遍历所有服务器，它将重置为`0`。让我们看一下全局配置变量：

```go
var Servers []Server
var nextServerIndex int
var ServersAvailable bool
var ServerChan chan bool
var Proxy *httputil.ReverseProxy
var ResetProxy chan bool
```

这些是我们的全局变量——我们的`Servers`切片的`Server`结构，用于递增下一个要返回的`Server`的`nextServerIndex`变量，`ServersAvailable`和`ServerChan`，它们在可用服务器可用后才启动负载均衡器，然后是我们的`Proxy`变量，告诉我们的`http`处理程序去哪里。这需要一个`ReverseProxy`方法，我们现在将在以下代码中查看：

```go
func handler(p *httputil.ReverseProxy) func(http.ResponseWriter, *http.Request) {
  Proxy = setProxy()
  return func(w http.ResponseWriter, r *http.Request) {

    r.URL.Path = "/"

    p.ServeHTTP(w, r)

  }
}
```

请注意，我们在这里操作的是`ReverseProxy`结构，这与我们之前对提供网页服务的尝试不同。我们的下一个函数执行循环负载均衡，并获取我们的下一个可用服务器：

```go
func setProxy() *httputil.ReverseProxy {

  nextServer := roundRobin()
  nextURL, _ := url.Parse(nextServer.Addr)
  log.Println("Next proxy source:", nextServer.Addr)
  prox := httputil.NewSingleHostReverseProxy(nextURL)

  return prox
}
```

`setProxy`函数在每个请求之后被调用，你可以在我们的处理程序中看到它是第一行。接下来，我们有一个通用的监听函数，用于监听我们将要进行反向代理的请求：

```go
func startListening() {
  http.HandleFunc("/index.html", handler(Proxy))
  _ = http.ListenAndServe(":8080", nil)

}

func main() {
  nextServerIndex = 0
  ServersAvailable = false
  ServerChan := make(chan bool)
  done := make(chan bool)

  fmt.Println("Starting load balancer")
  Servers = []Server{{Name: "Web Server 01", Addr: "http://www.google.com", Status: false, InService: false}, {Name: "Web Server 02", Addr: "http://www.amazon.com", Status: false, InService: false}, {Name: "Web Server 03", Addr: "http://www.apple.zom", Status: false, InService: false}}

  go healthCheck(ServerChan)

  for {
    select {
    case <-ServerChan:
      Proxy = setProxy()
      startListening()
      return

    }
  }

  <-done
}
```

通过这个应用程序，我们拥有一个简单但可扩展的负载均衡器，它与 Go 中的常见核心组件一起工作。其并发特性使其保持精简和快速，我们只使用标准的 Go 编写了非常少量的代码。

# 选择单向和双向通道

为了简单起见，我们设计了大部分应用程序和示例代码都使用双向通道，但当然任何通道都可以设置为单向。这基本上将通道转换为“只读”或“只写”通道。

如果您想知道为什么在不节省任何资源或保证问题的情况下限制通道的方向，原因归结为代码的简单性和限制恐慌的潜力。

到目前为止，我们知道在关闭的通道上发送数据会导致恐慌，因此如果我们有一个只写通道，我们在野外永远不会意外遇到这个问题。许多情况下也可以通过 `WaitGroups` 来减轻这种情况，但在这种情况下，这就像用锤子敲钉子。考虑以下循环：

```go
const TOTAL_RANDOMS = 100

func concurrentNumbers(ch chan int) {
  for i := 0; i < TOTAL_RANDOMS; i++ {
    ch <- i
  }
}

func main() {

  ch := make(chan int)

  go concurrentNumbers(ch)

  for {
    select {
      case num := <- ch:
        fmt.Println(num)
        if num == 98 {
          close(ch)
        }
      default:
    }
  }
}
```

由于我们在 goroutine 完成之前突然关闭了 `ch` 通道的一个数字，任何对它的写入都会导致运行时错误。

在这种情况下，我们正在调用一个只读命令，但它在 `select` 循环中。我们可以通过只允许在单向通道上发送特定操作来更安全地进行这个操作。这个应用程序将始终在通道被过早关闭的情况下工作，比 `TOTAL_RANDOMS` 常量少一个。

## 使用只接收或只发送的通道

当我们限制通道的方向或读/写能力时，我们还减少了如果我们的一个或多个进程无意中在这样的通道上发送时关闭通道死锁的可能性。

因此，对于问题“何时适合使用单向通道？”的简短答案是“只要可能”。

不要强迫这个问题，但如果您可以将通道设置为只读或只写，可能会在将来避免问题。

# 使用不确定的通道类型

一个经常有用的技巧，我们还没有解决的是，能够拥有有效的无类型通道的能力。

如果您想知道为什么这可能有用，简短的答案是简洁的代码和应用程序设计节俭。通常这是一种不鼓励的策略，但您可能会发现它在某些时候很有用，特别是当您需要通过单个通道传达一个或多个不同的概念时。以下是一个不确定通道类型的示例：

```go
package main

import (

  "fmt"
  "time"
)

func main() {

  acceptingChannel := make(chan interface{})

  go func() {

    acceptingChannel <- "A text message"
    time.Sleep(3 * time.Second)
    acceptingChannel <- false
  }()

  for {
    select {
      case msg := <- acceptingChannel:
        switch typ := msg.(type) {
          case string:
            fmt.Println("Got text message",typ)
          case bool:
            fmt.Println("Got boolean message",typ)
            if typ == false {
              return
            }
          default:
          fmt.Println("Some other type of message")
        }

      default:

    }

  }

  <- acceptingChannel
}
```

# 使用 Go 进行单元测试

与您可能有的许多基本和中级开发和部署要求一样，Go 自带了一个用于处理单元测试的内置应用程序。

测试的基本前提是您创建您的包，然后创建一个测试包来针对初始应用程序运行。以下是一个非常基本的示例：

```go
mathematics.go
package mathematics

func Square(x int) int {

  return x * 3
}
mathematics_test.go
package mathematics

import
(
  "testing"
)

func Test_Square_1(t *testing.T) {
  if Square(2) != 4 {
    t.Error("Square function failed one test")
  }
}
```

在该子目录中进行简单的 Go 测试将给您所需的响应。虽然这显然很简单，而且故意有缺陷，但您可能会看到如何轻松地拆分您的代码并逐步测试它。这足以在开箱即用的情况下进行非常基本的单元测试。

然后，对此进行更正将相当简单——相同的测试将通过以下代码：

```go
func Square(x int) int {

  return x * x
}
```

测试包有一定的局限性；然而，它提供了基本的通过/失败，没有断言的能力。有两个第三方包可以在这方面提供帮助，我们将在以下部分进行探讨。

## GoCheck

**GoCheck** 主要通过增加断言和验证来扩展基本测试包。您还将获得一些基本的基准测试实用程序，它的工作方式比您需要使用 Go 进行工程设计的任何东西更基本。

### 提示

有关 GoCheck 的更多详细信息，请访问 [`labix.org/gocheck`](http://labix.org/gocheck) 并使用 `go get gopkg.in/check.v1` 进行安装。

## Ginkgo 和 Gomega

这使得测试可以像单元测试一样细粒度，但也扩展了我们处理应用程序使用的方式，以详细和明确的行为。

如果 BDD 是您或您的组织感兴趣的内容，这是一个实现更深入单元测试的成熟包。

### 提示

有关 Ginkgo 的更多信息，请访问[`github.com/onsi/ginkgo`](https://github.com/onsi/ginkgo)并使用`go get github.com/onsi/ginkgo/ginkgo`进行安装。

有关依赖性的更多信息，请参阅`go get github.com/onsi/gomega`。

# 使用 Google App Engine

如果您对 Google App Engine 不熟悉，简短的版本是它是一个云环境，允许简单构建和部署**平台即服务**（**PaaS**）解决方案。

与许多类似的解决方案相比，Google App Engine 允许您以非常简单和直接的方式构建和测试应用程序。Google App Engine 允许您使用 Python、Java、PHP 和当然，Go 来编写和部署。

在很大程度上，Google App Engine 提供了一个标准的 Go 安装，使得可以轻松地与`http`软件包配合使用。但它还为您提供了一些独特于 Google App Engine 的值得注意的额外软件包：

| 软件包 | 描述 |
| --- | --- |
| `appengine/memcache` | 这提供了一个分布式的内存缓存安装，是 Google App Engine 独有的 |
| `appengine/mail` | 这允许您通过类似 SMTP 的平台发送电子邮件 |
| `appengine/log` | 鉴于您的存储可能更短暂，它将日志的云版本正式化 |
| `appengine/user` | 这打开了身份和 OAuth 功能 |
| `appengine/search` | 这使您的应用程序可以通过数据存储库获得 Google 搜索的功能 |
| `appengine/xmpp` | 这提供了类似 Google Chat 的功能 |
| `appengine/urlfetch` | 这是一个爬虫功能 |
| `appengine/aetest` | 这扩展了 Google App Engine 的单元测试 |

尽管对于 Google App Engine 来说，Go 仍然被认为是测试版，但您可以期待，如果有人能够在云环境中成功部署它，那就是 Google。

# 利用最佳实践

当涉及到最佳实践时，Go 的美妙之处在于，即使您并不一定做对了一切，Go 要么会警告您，要么会为您提供必要的工具来修复它。

如果您尝试包含代码但不使用它，或者尝试初始化变量但不使用它，Go 会阻止您。如果您想清理代码的格式，Go 可以使用`go fmt`来实现。

## 构建您的代码

从头开始构建软件包时，最简单的事情之一就是以符合惯例的方式构建代码目录。新软件包的标准看起来可能是以下代码：

```go
/projects/
  thisproject/
    bin/
    pkg/
    src/
      package/
        mypackage.go
```

设置您的 Go 代码就像这样不仅对您自己的组织有帮助，还可以更轻松地分发您的软件包。

## 记录您的代码

对于在企业或协作编码环境中工作过的人来说，文档是神圣的。正如您可能记得的那样，使用`godoc`命令可以让您快速获取有关软件包的信息，无论是在命令行还是通过特设的本地主机服务器。以下是您可以使用`godoc`的两种基本方式：

| 使用 godoc | 描述 |
| --- | --- |
| `godoc fmt` | 这将`fmt`文档显示在屏幕上 |
| `godoc -http=:3000` | 这将在端口`:3030`上托管文档 |

Go 使得记录您的代码变得非常容易，而且您绝对应该这样做。只需在每个标识符（软件包、类型或函数）上方添加单行注释，您将把它附加到上下文文档中，如下面的代码所示：

```go
// A demo documentation package
package documentation

// The documentation struct object
// Chapter int represents a document's chapter
// Content represents the text of the documentation
type Documentation struct {
  Chapter int
  Content string
}

//  Display() outputs the content of any given Document by chapter
func (d Documentation) Display() {

}
```

安装后，这将允许任何人在您的软件包上运行`godoc`文档，并获得您愿意提供的详细信息。

您经常会在 Go 核心代码中看到更健壮的示例，值得审查以比较您的文档风格与 Google 和 Go 社区的风格。

## 通过 go get 使您的代码可用

假设您已经按照之前列出的组织技术保持了代码的一致性，那么通过代码存储库和主机使您的代码可用应该是轻而易举的。

使用 GitHub 作为标准，这是我们设计第三方应用程序的方式：

1.  确保您遵循先前的结构格式。

1.  将源文件保存在它们将在远程存储的目录结构下。换句话说，预期本地结构将反映远程结构。

1.  显然，只提交您希望在远程存储库中共享的文件。

假设您的存储库是公开的，任何人都应该能够获取（`go get`）并安装（`go install`）您的包。

## 避免在您的包中使用并发

最后一个可能看起来有些不合时宜的观点是，如果您正在构建将被导入的单独包，请尽量避免包含并发代码。

这不是一个死板的规则，但考虑到潜在的使用情况，这是有道理的——除非您的包绝对需要，并发，否则让主应用程序处理并发。这样做将防止许多隐藏的、难以调试的行为，这可能会使您的库不那么吸引人。

# 总结

我真诚地希望您能够通过本书探索、理解并利用 Go 强大的并发能力。

我们已经讨论了很多内容，从最基本的、不涉及通道的并发 goroutines 到复杂的通道类型、并行性和分布式计算，并且在每一步都带来了一些示例代码。

到目前为止，您应该已经完全具备了以高度并发、快速和无错误的方式构建任何您心中所需的代码的能力。除此之外，您应该能够产生格式良好、结构合理、有文档的代码，可以被您、您的组织或其他人用来实现最佳利用并发的代码。

并发本身是一个模糊的概念；对不同的人（以及多种语言）来说，它的含义略有不同，但核心目标始终是快速、高效、可靠的代码，可以为任何应用程序提供性能提升。

掌握了 Go 中并发实现以及其内部工作原理的全面理解，我希望您在语言不断发展和成长的过程中继续您的 Go 之旅，并呼吁您考虑在 Go 项目本身的发展中做出贡献。
