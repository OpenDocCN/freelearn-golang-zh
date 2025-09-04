

# 高级主题

如果你正在阅读这一章——恭喜你，你已经到达了这本书的最后一部分！我们已经讨论了许多与微服务开发相关的话题，但还有一些重要的话题需要覆盖。本章涵盖的主题范围很广，从可观察性和调试到服务所有权和安全。你可能会在各个时间点发现这些主题很有用：其中一些在你有正在运行的服务处理生产流量时会有所帮助，而其他一些则在你服务仍在积极开发时会有所帮助。

在本章中，我们将涵盖以下主题：

+   分析 Go 服务

+   创建微服务仪表板

+   框架

+   存储微服务所有权数据

+   使用 JWT 保护微服务通信

让我们进入本章的第一部分，它涵盖了服务分析。

# 技术要求

要完成本章，你需要 Go 1.11 或更高版本。此外，你还需要以下工具：

+   **Graphviz**：[`graphviz.org`](https://graphviz.org)

+   **Docker**：[`www.docker.com`](https://www.docker.com)

你可以在 GitHub 上找到本章的代码示例：https://github.com/PacktPublishing/microservices-with-go/tree/main/Chapter13。

# 分析 Go 服务

在本节中，我们将回顾一种称为**分析**的技术，它涉及收集运行进程的实时性能数据，例如 Go 服务。分析是一种强大的技术，可以帮助你分析各种类型的服务性能数据：

+   **CPU 使用率**：哪些操作使用了最多的 CPU 功率，它们之间的 CPU 使用率分布是怎样的？

+   **堆分配**：哪些操作使用了堆（Go 应用程序中分配的动态内存）以及使用了多少内存？

+   **调用图**：服务函数的执行顺序是怎样的？

分析可以在不同的情况下帮助你：

+   **识别 CPU 密集型逻辑**：在某个时刻，你可能会注意到你的服务消耗了大部分的 CPU 功率。为了理解这个问题，你可以收集 CPU 分析——一个显示各种服务组件（如单个函数）CPU 使用的图表。消耗过多 CPU 功率的组件可能表明各种问题，如实现效率低下或代码错误。

+   **捕获服务的内存占用**：类似于高 CPU 消耗，你的服务可能使用了过多的内存（例如，为堆分配过多的数据），这可能导致服务偶尔因内存不足而崩溃。进行内存分析可以帮助你分析服务各个部分的内存使用情况，并找到意外占用内存过多的组件。

让我们通过使用 Go SDK 的一部分 pprof 工具来展示如何分析 Go 服务，为了可视化工具的结果，你需要安装 Graphviz 库：[`graphviz.org/`](https://graphviz.org/)。

我们将以我们在*第二章*中实现的元数据服务为例。打开 `metadata/cmd/main.go` 文件，并将 `flag` 包添加到 `imports` 块中。然后，在主函数的开始处，紧接在日志初始化之后，添加以下代码：

```go
simulateCPULoad := flag.Bool("simulatecpuload",
    false,"simulate CPU load for profiling")
    flag.Parse()
    if *simulateCPULoad {
        go heavyOperation()
    }
go func() {
    if err := http.ListenAndServe("localhost:6060", nil);
    err != nil {
        logger.Fatal("Failed to start profiler handler",
            zap.Error(err))
    }
}()
```

在我们刚刚添加的代码中，我们引入了一个额外的标志 `simulatecpuload`，它将允许我们模拟一个 CPU 密集型操作以进行分析。我们还启动了一个 HTTP 处理器，我们将使用它从命令行访问分析器数据。

现在，让我们向同一文件添加另一个函数，该函数将运行一个连续循环并执行一些 CPU 密集型操作。我们将生成随机的 1,024 字节数组并计算它们的 md5 哈希（你可以在其 Go 包的注释中阅读有关 md5 操作的内容，网址为 https://pkg.go.dev/crypto/md5）。我们选择这种逻辑是完全随机的：我们很容易选择任何其他会消耗 CPU 负载可见部分的操作。

将以下代码添加到我们刚刚更新的 `main.go` 文件中：

```go
func heavyOperation() {
    for {
        token := make([]byte, 1024)
        rand.Read(token)
        md5.New().Write(token)
    }
}
```

现在，我们已经准备好测试我们的分析逻辑。使用 `--simulatecpuload` 参数运行服务：

```go
go run *.go --simulatecpuload
```

现在，执行以下命令：

```go
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=5
```

命令应该需要 5 秒才能完成。如果执行成功，pprof 工具将运行，如下所示：

```go
Type: cpu
Time: Sep 13, 2022 at 5:37pm (+05)
Duration: 5.14s, Total samples = 4.42s (85.92%)
Entering interactive mode (type "help" for commands,
    "o" for options)
(pprof)
```

在工具的命令提示符中键入 `web` 并按 *Enter*。如果一切正常，你将被重定向到一个包含 CPU 配置文件图的浏览器窗口：

![图 13.1 – Go CPU 配置文件示例](img/Figure_13.1_B18865.jpg)

图 13.1 – Go CPU 配置文件示例

让我们遍历图中的数据，了解如何解释它。图中的每个节点都包含以下数据：

+   包名称

+   函数名称

+   运行时间和执行总时间

例如，`heavyOperation` 函数仅花费了 0.01 秒，但其中执行的所有操作（包括其内部的所有函数调用）共花费了 4.39 秒，占去大部分的运行时间。

如果你遍历这个图，你会看到子操作的时间分布。在我们的例子中，`heavyOperation` 执行了两个被 CPU 分析器记录的函数：`md5.Write` 和 `rand.Read`。`md5.Write` 函数总共花费了 2.78 秒，而 `rand.Read` 花费了 1.59 秒的执行时间。逐层分析，你可以找到 CPU 密集型函数。

当处理 CPU 分析器数据时，注意那些花费最多处理时间的函数。这些函数被表示为更大的矩形，以帮助你找到它们。如果你注意到某些函数的处理时间异常高，花些时间分析它们的代码，看看是否有任何机会进行优化。

现在，让我们通过另一个分析器数据的示例。这次，我们将捕获一个**堆分析** – 一个显示 Go 进程动态内存分配的分析。运行以下命令：

```go
go tool pprof http://localhost:6060/debug/pprof/heap
```

与前面的例子类似，成功执行此命令应运行 pprof 工具，在那里我们可以执行 web 命令。结果将包含以下图表：

![图 13.2 – Go 堆配置文件示例](img/Figure_13.2_B18865.jpg)

图 13.2 – Go 堆配置文件示例

此图与 CPU 配置文件类似。每个节点内的最后一行显示了函数使用的内存与进程分配的总堆内存之间的比率。

在我们的例子中，有三个高级操作正在消耗堆内存：

+   **api.serviceRegister**：一个通过 Consul API 注册服务的函数

+   `zap`库

+   **trace.init**：初始化跟踪逻辑

1.  观察堆配置文件数据，很容易找到分配意外大量堆内存的函数。与 CPU 配置文件图类似，堆配置文件显示具有最高堆分配的函数以更大的矩形显示，这使得可视化最消耗内存的函数更容易。

1.  我建议您使用 pprof 工具进行练习，并尝试它提供的其他操作。能够对 Go 应用程序进行性能分析是生产调试中非常宝贵的技能，这有助于您优化服务并解决不同的性能相关问题。以下是一些其他有用的 Go 服务性能分析技巧：

    +   您可以在不向代码添加任何额外逻辑的情况下对 Go 测试进行性能分析。使用带有`-cpuprofile`和`-memprofile`标志的`go test`命令将分别捕获您的逻辑的 CPU 和内存配置文件。

    +   pprof 工具的`top`命令是一种方便显示顶级内存消费者的方式。还有一个`top10`命令，它显示了前 10 个内存消费者。

    +   使用 pprof 工具的`goroutine`模式，您可以获取所有使用 goroutine 的配置文件，以及它们的堆栈跟踪。

现在我们已经介绍了 Go 性能分析的基础知识，让我们继续本章的下一个主题：服务仪表板化。

# 创建微服务仪表板

在前两章中，我们回顾了与服务指标一起工作的各种方法。在*第十一章*中，我们演示了如何收集服务指标，而在*第十二章*中，我们展示了如何使用 Prometheus 工具聚合和查询它们。在本节中，我们将描述另一种访问指标数据的方法，这可以帮助您探索指标并将它们作为图表绘制。我们将介绍的技术称为**仪表板化**，它对于可视化各种服务指标非常有用。

让我们提供一个仪表板的例子——一组代表不同指标的图表。以下图显示了包含一些系统级指标（如 goroutine 数量、Go 线程数量和分配的内存大小）的 Go 服务仪表板：

![图 13.3 – 使用 Grafana 工具的 Go 进程仪表板示例![图 13.3 – Go 进程仪表板示例](img/Figure_13.3_B18865.jpg)

图 13.3 – Grafana 工具的 Go 进程仪表板示例

仪表板有助于可视化各种类型的数据，例如时间序列数据集，使我们能够分析服务性能。以下是一些使用仪表板的其它用例：

+   **调试**：能够可视化各种服务性能指标有助于我们识别服务问题并注意系统活动中的任何异常

+   **数据相关性**：多个服务性能图表并排显示有助于我们找到相关事件，例如服务器错误增加或可用内存突然下降

为每个服务设置一个仪表板，以及一些跨越所有服务的仪表板，以获取一些高级系统性能数据，例如活动服务实例的数量、网络吞吐量等等，这是一个很好的实践。

让我们演示如何为我们在 *第十二章* 中收集的 Prometheus 数据设置一个示例仪表板。为此，我们将使用名为 Grafana 的开源工具，它内置了对各种类型的时间序列数据的支持，并提供了一个方便的用户界面来设置不同的仪表板。按照以下说明设置 Grafana 仪表板：

1.  执行以下命令以运行 Grafana Docker 镜像：

    ```go
    docker run -d -p 3000:3000 grafana/grafana-oss
    ```

此命令应获取并运行 Grafana 的开源版本（Grafana 还有一个企业版本，我们将在本章中不涉及）并暴露端口 `3000`，以便我们可以通过 HTTP 访问。

注意

与 Prometheus 类似，Grafana 也是用 Go 编写的，它是广泛用于软件开发行业的流行开源 Go 项目的另一个例子。

1.  执行上述命令后，在您的浏览器中打开 [`localhost:3000`](http://localhost:3000)。这将带您到 Grafana 登录页面。默认情况下，基于 Docker 的 Grafana 版本包括一个用户，其用户名和密码都是 `admin`，因此您可以使用这些凭据登录。

1.  从侧边菜单中选择 **配置**：

![图 13.4 – Grafana 数据源配置菜单](img/Figure_13.4_B18865.jpg)

图 13.4 – Grafana 数据源配置菜单

1.  在 **配置** 页面上，点击 **数据源** 菜单项，然后点击 **添加数据源** 并从可用数据源列表中选择 **Prometheus**。这样做将打开一个新页面，显示 Prometheus 设置。在 **HTTP** 部分，将 **URL** 设置为 [`host.docker.internal:9090`](http://host.docker.internal:9090)，如下截图所示：

![图 13.5 – Grafana Prometheus 数据源配置](img/Figure_13.5_B18865.jpg)

图 13.5 – Grafana Prometheus 数据源配置

1.  现在，您可以在页面底部点击 **保存并测试** 按钮，这将通知您操作是否成功。如果您一切都做得很好，Grafana 应该准备好显示来自 Prometheus 的指标。

1.  从侧边菜单中，点击**新建仪表板**：

![图 13.6 – Grafana 的“新建仪表板”菜单项用于创建仪表板

![img/Figure_13.6_B18865.jpg]

图 13.6 – Grafana 新建仪表板菜单项用于创建仪表板

1.  这应该打开一个空白的仪表板页面。

1.  在此仪表板页面上点击**添加新面板**按钮；您将被重定向到面板创建页面。

面板是 Grafana 仪表板的核心元素，其目的是可视化提供的数据集。为了说明如何使用它，让我们选择我们的 Prometheus 数据源以及它已经拥有的某些指标。在面板视图中，选择`process_open_fds`元素并选择它。现在，点击**运行查询**按钮；您应该看到以下视图：

![图 13.7 – Grafana 面板视图

![img/Figure_13.7_B18865.jpg]

图 13.7 – Grafana 面板视图

我们刚刚配置了仪表板面板以显示存储在 Prometheus 中的`process_open_fds`时间序列。图表上的每个数据点都显示了时间序列在不同时间点的值，显示在图表下方。在右侧面板中，您可以设置面板标题为**打开文件描述符计数**。现在，通过点击顶部菜单中提供的**应用**按钮保存仪表板。您将被重定向到仪表板页面。

在顶部菜单中，您将找到`go_gc_duration_seconds`指标，您将添加一个新的面板到仪表板，用于可视化 Prometheus 中的`go_gc_duration_seconds`时间序列。

生成的仪表板应该看起来像这样：

![图 13.8 – 示例 Grafana 仪表板

![img/Figure_13.8_B18865.jpg]

图 13.8 – 示例 Grafana 仪表板

我们刚刚创建了一个示例仪表板，它包含两个面板，用于显示一些现有的 Prometheus 指标。您可以使用相同的方法为您的服务创建任何仪表板，以及显示系统全局指标的高级仪表板，例如 API 请求总数、网络吞吐量或所有服务实例的总数。

让我们提供一些有用的指标示例，这些指标可以用于为单个服务设置仪表板。这包括我们提到的*第十二章*中的*四个黄金信号*：

+   **客户端错误率**：客户端错误（如无效或未经认证的请求）与对服务的所有请求之间的比率

+   **服务器错误率**：服务器错误（如数据库写入错误）与对服务的所有请求之间的比率

+   **API 吞吐量**：每秒/每分钟的 API 请求数量

+   **API 延迟**：API 请求处理延迟，通常以百分位数衡量，如 p90/p95/p99（您可以通过阅读这篇博客了解百分位数：https://www.elastic.co/blog/averages-can-dangerous-use-percentile）

+   **CPU 利用率**：CPU 当前的使用情况（100%表示所有 CPU 都处于完全负载状态）

+   **内存利用率**：所有服务实例中已用内存与总内存之间的比率

+   **网络吞吐量**：每秒/每分钟的网络读写流量总量

根据您的服务执行的操作（例如，数据库写入或读取、缓存使用、Kafka 消费或生产），您可能希望包含额外的面板，以帮助您可视化服务性能。确保您涵盖了服务的高级所有功能，这样您就可以在仪表板上直观地注意到任何服务故障。

我们在示例中使用的 Grafana 工具也支持许多不同的可视化选项，例如显示表格、热图、数值等。我们不会在本章中介绍这些功能，但您可以通过阅读官方文档来熟悉它们：https://grafana.com/docs/。利用 Grafana 的全部功能将帮助您为您的服务设置出色的仪表板，简化您的调试和性能分析。

现在，让我们进入下一节，我们将描述 Go 框架。

# 框架

在*第二章*中，我们讨论了 Go 项目结构的话题，以及一些组织 Go 代码的常见模式。我们描述的代码组织原则通常基于约定——书面协议或声明，用于定义命名和放置 Go 文件的特定规则。我们遵循的一些约定是由 Go 语言的作者提出的，而其他则是被各种 Go 库的作者广泛使用并提出的。

虽然约定在建立组织 Go 代码的共同原则方面发挥着重要作用，但还有其他方法可以强制执行特定的代码结构。其中一种方法就是使用框架，我们将在本节中介绍。

通常来说，**框架**是用于建立代码各种组件结构的工具。以下代码片段作为例子：

```go
package main
import (
    "fmt"
    "net/http"
)
func main() {
    http.HandleFunc("/echo",
        func(w http.ResponseWriter, _ *http.Request) {
            fmt.Fprintf(w, "Hi!")
        })
    if err := http.ListenAndServe(":8080", nil);
    err != nil {
        panic(err)
    }
}
```

在这里，我们正在注册一个 HTTP 处理函数，并让它处理`localhost:8080/echo`端点的 HTTP 请求。我们示例中的代码非常简单，但它做了大量的后台工作（您可以检查`net/http`包的源代码，以了解 HTTP 处理逻辑的内部部分是多么复杂），以启动 HTTP 服务器、接受所有传入的请求并通过执行我们提供的函数来响应它们。最重要的是，我们的代码允许我们通过遵循调用`http.HandleFunc`函数和向其传递处理函数的相同格式来添加额外的 HTTP 处理程序。我们示例中使用的`net/http`库为处理各种端点的 HTTP 调用建立了结构，充当了我们 Go 应用程序的框架。

`net/http`包的作者能够通过遵循名为`net/http`包的模式添加额外的 HTTP 端点处理器（由`http.HandleFunc`函数提供），该模式通过调用其其他组件（在我们的情况下，是作为`http.HandleFunc`参数提供的函数）来控制执行流程。在我们的例子中，当我们调用`http.ListenAndServe`函数时，`net/http`包就控制了 HTTP 处理器函数的执行：每当 HTTP 服务器收到传入的请求时，我们的函数会自动被调用。

IaC（基础设施即代码）是大多数框架的主要机制，它使它们能够为应用程序代码的各个部分建立基础。一般来说，大多数框架通过控制应用程序或其一部分，并处理一些路由操作来工作，例如资源管理（打开和关闭传入连接、读写文件等）、序列化和反序列化，以及更多。

使用 Go 框架的主要用例有哪些？我们可以列出一些最常见的用例：

+   **编写网络服务器**：类似于我们关于 HTTP 服务器的例子，可以有其他类型的网络服务器，它们使用不同的协议处理对不同端点的请求，例如 Apache Thrift 或 gRPC。

+   **异步事件处理**：有各种异步通信工具的库，如 Apache Kafka，它们通过传递各种类型事件（例如属于不同主题的 Kafka 消息）的处理函数来以 IoC（控制反转）方式组织代码，这些处理函数会在每次有新的未处理消息时自动调用。

需要注意的是，框架有一些显著的缺点：

+   **更难调试和理解执行流程**：除了控制执行流程外，框架还执行大量对开发者隐藏的背景工作。正因为如此，通常很难理解代码是如何执行的，以及调试各种问题，例如初始化错误（你可以在以下文章中找到更多关于此的信息：[`www.baeldung.com/cs/framework-vs-library`](https://www.baeldung.com/cs/framework-vs-library))。

+   **学习曲线更陡峭**：框架通常需要开发者对它们提供的逻辑和抽象有很好的理解。这要求开发者花费更多时间阅读相关文档或在实践中学习一些关键经验。

+   `reflect`（https://pkg.go.dev/reflect）。这些操作在程序执行时进行，使得捕捉各种类型的问题变得困难，例如接口实现不正确或无效的命名。

在决定使用特定的框架时，您应该做一些分析，比较它为您提供的优势与它带来的劣势，尤其是在长期内。许多开发者低估了框架给他们或他们组织中的其他开发者带来的复杂性：大多数框架都进行了一定程度的“魔法”操作，为应用开发者提供方便的代码结构。一般来说，您应该始终从一个更简单的选项（在我们的案例中，即不使用特定的框架）开始，并且只有在框架的好处超过其劣势时才决定使用框架。

现在我们已经讨论了框架的话题，接下来让我们进入下一节，我们将描述微服务所有权的不同方面。

# 存储微服务所有权数据

使用微服务架构的一个关键好处是能够分散其开发：每个服务可以由一个独立的团队开发和维护，并且团队可以分布在全球各地。虽然分布式开发模型有助于不同团队独立构建其系统的各个部分，但它也带来了一些新的挑战，例如服务所有权。

为了说明服务所有权的问题，想象一下您正在一家拥有数千个微服务的公司工作。有一天，您的公司的安全工程师发现，在大多数公司服务中使用的流行 Go 库中存在一个关键的安全漏洞。您如何与正确的团队沟通，找出谁将负责对每个服务进行更改？

有许多公司拥有数千个微服务。在这样的公司中，记住哪个团队和哪些开发者负责每个服务变得不可能。在这样的公司中，找到解决服务所有权问题的解决方案变得至关重要。

注意

当我们在讨论微服务的所有权问题时，同样的原则也适用于许多其他类型的科技资产，例如 Kafka 主题和数据库表。

我们如何定义服务所有权？

有许多不同的方法来做这件事，每种方法对于某些特定的用例来说都同等重要：

+   **问责制**：哪个人/实体对服务负责，谁可以充当其主要联系点或主要权威？

+   **支持**：谁将提供对服务的支持，例如服务中的 bug、功能请求或用户问题？

+   **值班**：目前谁负责该服务的值班？在紧急情况下我们可以联系谁？

如您所见，根据使用场景的不同，对“所有权”一词的解释有很多种。让我们看看定义每个角色的几种方法，从问责制开始：谁应该对服务负责，或者说谁应该对服务承担责任？

在大多数组织中，责任归咎于工程经理：每位工程经理都作为某个独特领域的责任个人。如果您在您的服务和负责它们的工程经理之间定义一个映射，您可以通过让他们轻松找到相关联系人点来解决服务责任问题，例如负责该服务的工程经理。

定义服务责任的另一种方法是将其与团队关联。然而，这可能会出现多个问题：

+   **共享责任并不总是有效**：如果您有多个对服务负责的人，那么在他们中间谁是最终权威就变得不清楚了。

+   **在许多组织中，团队是一个松散定义的概念**：除非您在公司中有一个单一、明确定义的团队注册，否则最好避免在您的系统中引用团队名称。

现在，让我们来讨论所有权的支持方面。理想情况下，每个服务都应该有一个机制来报告任何问题或错误。这样的机制可以采取以下形式之一：

+   **支持渠道**：用于留下支持请求的消息渠道的标识符或 URL，例如相关 Google 群组、Slack 频道或任何其他类似工具的链接。

+   **票据系统 URL**：允许您创建支持请求票据的系统/页面的 URL。开发者通常使用 Atlassian Jira 来完成这项任务。

如果您为所有服务提供此类元数据，您将显著简化用户支持：所有服务用户，如其他开发者，都将始终知道如何为他们请求支持或报告任何错误或其他问题。

让我们继续讨论待命的所有权元数据。解决这个问题的简单方法是将每个服务与其待命轮换关联起来。如果您使用 PagerDuty，您可以存储服务名称与其对应的 PagerDuty 轮换标识符之间的关系。

我们刚才描述的所有权元数据的示例如下：

```go
ownership:
    rating-service:
        accountable: example@somecompany.com
            support:
                slack: rating-service-support-group
                    oncall:
                        pagerduty_rotation:SOME_ROTATION_ID
```

我们的示例定义在 YAML 格式中，尽管将此数据存储在允许我们通过 API 查询或修改它的系统中可能更可取。这样，您可以自动提交新的所有权更改（例如，当人们离开公司并且您希望自动重新分配所有权时）。我还建议将所有权数据对所有服务强制为必填项。为了强制执行此操作，您可以建立一个服务创建流程，在开发者在提供新服务之前要求提供所有权数据。

现在我们已经讨论了服务所有权，接下来让我们进入下一节，我们将描述 Go 微服务安全的基础知识。

# 使用 JWT 保护微服务通信

在本节中，我们将回顾一些微服务安全的基本概念，例如身份验证和授权。您将学习如何使用流行的**JSON Web Token**（JWT）协议在 Go 中实现此类逻辑。

让我们从安全的一个主要方面开始：**认证**。认证是验证某人身份的过程，例如通过用户凭证。当你登录到某个系统，如 Gmail 时，你通常通过提供登录详细信息（用户名和密码）来完成认证过程。执行认证的系统通过将提供的数据与它存储的现有记录进行比较来进行验证。验证可以是一步或多步：某些类型的认证，如双因素认证，需要一些额外的操作，例如通过短信验证对电话号码的访问。

成功的认证通常会导致授予调用者访问某些资源的权限，例如用户数据（例如，Gmail 中的用户电子邮件）。此外，执行此认证的服务器可能还会向调用者提供一个安全令牌，该令牌可以在后续调用中使用以跳过验证过程。

另一种形式的访问控制，称为**授权**，涉及指定对各种资源的访问权限。授权通常是为了检查用户是否有权执行某些操作，例如查看特定的管理员页面。授权通常是通过使用在认证期间获得的安全令牌来执行的，如下面的图所示：

![图 13.9 – 提供令牌的授权请求](img/Figure_13.9_B18865.jpg)

图 13.9 – 提供令牌的授权请求

在微服务中实现认证和授权有许多不同的方法。其中最受欢迎的协议之一是 JWT，这是一个提议的互联网标准，用于创建可以包含关于调用者身份的任何数量事实的安全令牌，例如他们是否是管理员。让我们回顾一下该协议的基本知识，以帮助您了解如何在您的服务中使用它。

## JWT 基础

JWT 由执行认证或授权的组件生成。每个令牌由三部分组成：一个头部、一个有效载荷和一个签名。有效载荷是令牌的主要部分，它包含一组**声明**——关于调用者身份的陈述，例如用户标识符或系统中的角色。以下代码显示了一个令牌有效载荷的示例：

```go
{
    "name": "Alexander",
    "role": "admin",
    "iat": 1663880774
}
```

我们的示例有效载荷包含三个声明：用户的姓名、角色（在我们的示例中为`admin`）和令牌发行时间（`iat`是一个标准字段名，它是 JWT 协议的一部分）。这些声明可以在各种流程中使用——例如，当检查用户是否有`admin`角色以访问系统仪表板时。

作为防止修改的保护机制，每个令牌都包含一个**签名**——其有效载荷的加密函数、一个头部和一个称为**密钥**的特殊值，这个密钥只有认证服务器才知道。以下伪代码提供了一个令牌签名计算的示例：

```go
HMACSHA256(
    base64UrlEncode(header) + "." +
    base64UrlEncode(payload),
    secret,
)
```

用于创建令牌签名的算法在**令牌头**中定义。以下 JSON 记录提供了一个头部的示例：

```go
{
    "alg": "HS256",
    "typ": "JWT"
}
```

在我们的示例中，令牌使用`HMAC-SHA256`，这是一种常用的 JWT 签名加密算法。我们选择`HMAC-SHA256`主要是由于其流行度；如果您想了解其他签名算法，可以在本章的*进一步阅读*部分找到它们的概述。

生成的 JWT 是令牌的头、有效载荷和签名的串联，使用*Base64uri*协议编码。例如，以下值是一个 JWT，它通过结合我们的代码片段中的头部和有效载荷，并使用名为`our-secret`的密钥字符串签名而创建：

```go
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiQWxleGFuZGVyIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNjYzODgwNzc0fQ.FqogLyrV28wR5po6SMouJ7qs2Y3m6gmpaPg6MUthWpQ
```

为了练习 JWT 的创建，我建议使用位于[`jwt.io`](https://jwt.io)的 JWT 工具尝试编码任意的 JWT 并查看生成的令牌值。

现在我们已经讨论了 JWT 的高级细节，让我们继续本节的实际部分——使用 JWT 在 Go 微服务中实现基本的身份验证和授权。

## 使用 JWT 实现身份验证和授权

在本节中，我们将提供一些使用 Go 实现基本访问控制（通过身份验证和授权）的示例。

让我们从认证过程开始。一个基于凭证的简单认证流程可以总结如下：

+   初始化认证的客户端会调用一个指定的端点（例如，HTTPS POST `/auth`），同时提供用户凭证，如用户名和密码。

+   处理认证的服务器会验证凭证并执行以下两个动作之一：

    +   如果凭证无效，则返回错误（例如，带有`401`代码的 HTTP 错误）。

    +   返回一个包含 JWT 的成功响应，该 JWT 由服务器的密钥签名。

+   如果认证成功，客户端可以存储收到的令牌，以便在后续请求中使用。

让我们说明如何实现我们刚才描述的认证流程的服务器逻辑。在我们的 Go 代码中生成 JWT，我们将使用`https://github.com/golang-jwt/jwt`库。

以下代码提供了一个处理 HTTP 认证请求的示例。它执行凭证验证，如果验证通过，则返回一个包含已签名 JWT 的成功响应：

```go
const secret = "our-secret"
func Authenticate(w http.ResponseWriter, req *http.Request) {
    username := req.FormValue("username")
    password := req.FormValue("password")
    if !validCredentials(username, password) {
        http.Error(w, "invalid credentials", http.StatusUnauthorized)
        return
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "username": username,
        "iat": time.Now().Unix(),
    })
    tokenString, err := token.SignedString(secret)
    if err != nil {
        http.Error(w, "failed to create a token", http.StatusInternalServerError)
        return
    }
    fmt.Fprintf(w, tokenString)
}
func validCredentials(username, password string) bool {
    // Implement your credential verification here.
    return false
}
```

在前面的代码中，我们使用`jwt.NewWithClaims`函数创建了一个令牌。该令牌包括两个字段：

+   `username`：已认证用户的名称

+   `iat`：令牌创建时间

我们刚才创建的服务器代码正在使用密钥值来签名令牌。不知道密钥的情况下，任何修改令牌的尝试都是不可能的：令牌签名允许我们检查令牌是否正确。

现在，让我们说明客户端如何使用在成功认证后收到的令牌进行请求：

```go
func authorizationExample(
    token string, operationURL string) error {
        req, err := http.NewRequest(
            http.MethodPost, operationURL, nil)
            if err != nil {
                return err
            }
        req.Header.Set("Authorization", "Bearer "+token)
        resp, err := http.DefaultClient.Do(req)
        // Handle response.
    }
```

在我们的授权操作示例中，我们在使用带有 `Bearer` 前缀的令牌值的同时，向请求中添加了 `Authorization` 头。`Bearer` 前缀定义了一个 **携带令牌** – 一个意图赋予其携带者访问权限的令牌。

同时，我们也提供处理此类授权请求并验证提供的令牌是否正确的服务器处理器的逻辑：

```go
func AuthorizedOperationExample(w http.ResponseWriter,
    req *http.Request) {
        authHeaderValue := req.Header.Get("Authorization")
        const bearerPrefix = "Bearer "
        if !strings.HasPrefix(authHeaderValue,
            bearerPrefix) {
                http.Error(w,
                    "request does not contain an Authorization Bearer token", http.StatusUnauthorized)
                return
            }
        tokenString := strings.TrimPrefix(authHeaderValue,
            bearerPrefix)
        // Validate token.
        token, err := jwt.Parse(tokenString,
            func(token *jwt.Token) (interface{}, error) {
                if _, ok := token.Method.(
                    *jwt.SigningMethodHMAC); !ok {
                        return nil,
                        fmt.Errorf(
                            "unexpected signing method:
                                %v", token.Header["alg"])
            }
            return secret, nil
        })
        if err != nil {
            http.Error(w, "invalid token",
                http.StatusUnauthorized)
        }
        claims, ok := token.Claims.(jwt.MapClaims)
        if !ok || !token.Valid {
            http.Error(w, "invalid token",
                http.StatusUnauthorized)
            return
        }
        username := claims["username"]
        fmt.Fprintf(w, "Hello, "+username.(string))
    }
```

让我们描述一下提供的示例的一些亮点：

+   我们使用 `jwt.Parse` 函数来解析令牌并验证它。如果签名算法与我们之前使用的 `HMAC-SHA256` 不匹配，我们返回一个错误。

+   解析后的令牌包含 `Claims` 字段，它包含令牌有效载荷中的声明。

+   我们在我们的函数中使用令牌有效载荷中的 `username` 声明。一旦我们成功解析令牌并验证它是有效的，我们可以假设其有效载荷中的信息已经安全地传递给我们，并且可以信任它。

现在我们已经提供了使用 JWTs 进行 Go 认证和授权的示例，让我们列出一些使用 JWTs 保护微服务通信的最佳实践：

+   (`exp` JWT 声明字段) 以避免用户使用旧的授权记录的情况。通过在令牌有效载荷中设置过期时间，你可以将其与授权请求进行验证。例如，当用户以系统管理员身份进行认证时，你可以设置一个短的令牌过期时间（例如，几个小时），以避免前管理员仍然可以在系统中执行关键操作的情况。

+   (`ist` JWT 声明字段)，在许多实际情况下可能很有用。例如，如果你确定在某个时间点发生的安全漏洞，你可以通过使用令牌发行时间元数据来使在该时刻之前发行的所有的访问令牌失效。

+   **使用 HTTPS 而不是 HTTP 来使用 JWTs**：HTTPS 协议加密请求元数据，例如授权请求头，防止各种类型的网络安全攻击。这类安全攻击的一个例子是 *中间人攻击*，即某个第三方（例如试图获取用户访问令牌的黑客）捕获网络流量以从请求头中提取 JWTs。

+   **优先使用标准 JWT 声明字段而不是自定义字段**：当在 JWT 有效载荷中包含元数据时，请确保没有相同目的的标准字段。你可以在 https://en.wikipedia.org/wiki/JSON_Web_Token#Standard_fields 找到标准 JWT 声明字段列表。

[`jwt.io/`](https://jwt.io/) 网站包含一些关于使用 JWTs 的额外提示，以及一个用于编码和解码 JWTs 的在线工具，你可以使用它来调试你的服务通信。

# 摘要

通过回顾许多未在前几章中包含的微服务开发主题，我们完成了这本书的最后一章。你学习了如何分析 Go 服务，创建微服务仪表板以便监控其性能，定义和存储微服务所有权数据，以及使用 JWT 保护微服务通信。我希望你在这一章中找到了许多有用的提示，这将帮助你构建可扩展、高性能和安全的微服务。

Go 语言及其工具集不断进化。每天，开发者都会发布新的库和工具来解决我们在本书中描述的各种微服务开发问题。虽然本书为您提供了许多关于 Go 微服务开发的提示，但您应该不断改进您的技能，并使您的服务更简单、更易于维护。

我还想感谢您阅读这本书。希望您喜欢阅读它，并从中获得了许多有用的经验，这将帮助您掌握 Go 微服务开发的技艺。让您的 Go 微服务具有高性能、安全且易于维护！

# 进一步阅读

要了解更多关于本章所涉及的主题，请查看以下资源：

+   *Go 程序分析*: [`go.dev/blog/pprof`](https://go.dev/blog/pprof)

+   Grafana 文档: [`grafana.com/docs/`](https://grafana.com/docs/)

+   *Grafana 对 Prometheus 的支持*: [`prometheus.io/docs/visualization/grafana/`](https://prometheus.io/docs/visualization/grafana/)

+   Top Go 框架: [`github.com/mingrammer/go-web-framework-stars`](https://github.com/mingrammer/go-web-framework-stars)

+   JSON web token: [`jwt.io/`](https://jwt.io/)

+   *JSON Web Token (JWT) 签名算法* 概述: https://auth0.com/blog/json-web-token-signing-algorithms-overview/

我们建议您使用以下资源来了解与 Go 微服务开发相关的最新新闻：

+   **Go 博客**: [`go.dev/blog/`](https://go.dev/blog/)

+   **微服务架构**: https://microservices.io

+   **精选的 Go 软件列表**: [`github.com/avelino/awesome-go`](https://github.com/avelino/awesome-go)
