# 前言

最初，基于 SOAP 的 Web 服务因 XML 而变得更受欢迎。然后，自 2012 年以来，REST 加快了步伐，并完全取代了 SOAP。新一代的 Web 语言，如 Python、JavaScript（Node.js）和 Go，展示了与传统的 ASP.NET 和 Spring 等相比，不同的 Web 开发方法。自本十年以来，由于其速度和直观性，Go 变得越来越受欢迎。少量冗长的代码、严格的类型检查和对并发的支持使 Go 成为编写任何 Web 后端的更好选择。一些最好的工具，如 Docker 和 Kubernetes，都是用 Go 编写的。谷歌在日常活动中大量使用 Go。您可以在[`github.com/golang/go/wiki/GoUsers`](https://github.com/golang/go/wiki/GoUsers)上看到使用 Go 的公司列表。

对于任何互联网公司，Web 开发部门至关重要。公司积累的数据需要以 API 或 Web 服务的形式提供给客户。各种客户端（浏览器、移动应用程序和服务器）每天都会使用 API。REST 是一种定义资源消耗形式的架构模式。

Go 是一个更好的编写 Web 服务器的语言。作为中级 Go 开发人员，了解如何使用语言中提供的构造创建 RESTful 服务是其责任。一旦掌握了基础知识，开发人员应该学习其他内容，如测试、优化和部署服务。本书旨在使读者能够舒适地开发 Web 服务。

专家认为，在不久的将来，随着 Python 进入数据科学领域并与 R 竞争，Go 可能会成为与 NodeJS 竞争的 Web 开发领域的唯一选择语言。本书不是一本食谱。然而，在您的旅程中，它提供了许多技巧和窍门。通过本书，读者最终将能够通过大量示例舒适地进行 REST API 开发。他们还将了解到最新的实践，如协议缓冲区/gRPC/API 网关，这将使他们的知识提升到下一个水平。

# 本书涵盖内容

第一章，“开始 REST API 开发”，讨论了 REST 架构和动词的基本原理。

第二章，“为我们的 REST 服务处理路由”，描述了如何为我们的 API 添加路由。

第三章，“使用中间件和 RPC”，讲述了如何使用中间件处理程序和基本的 RPC。

第四章，“使用流行的 Go 框架简化 RESTful 服务”，介绍了使用框架进行快速原型设计 API。

第五章，“使用 MongoDB 和 Go 创建 REST API”，解释了如何将 MongoDB 用作我们 API 的数据库。

第六章，“使用协议缓冲区和 gRPC”，展示了如何使用协议缓冲区和 gRPC 来获得比 HTTP/JSON 更高的性能提升。

第七章，“使用 PostgreSQL、JSON 和 Go”，解释了使用 PostgreSQL 和 JSON 存储创建 API 的好处。

第八章，“在 Go 中构建 REST API 客户端和单元测试”，介绍了在 Go 中构建客户端软件和使用单元测试进行 API 测试的技术。

第九章，“使用微服务扩展我们的 REST API”，讲述了如何使用 Go Kit 将我们的 API 服务拆分为微服务。

第十章，“部署我们的 REST 服务”，展示了如何使用 Nginx 部署服务，并使用 supervisord 进行监控。

第十一章，“使用 API 网关监控和度量 REST API”，解释了如何通过在 API 网关后添加多个 API 来使我们的服务达到生产级别。

第十二章，“为我们的 REST 服务处理身份验证”，讨论了如何使用基本身份验证和 JSON Web Tokens（JWT）保护我们的 API。

# 本书所需内容

对于这本书，您需要一台安装了 Linux（Ubuntu 16.04）、macOS X 或 Windows 的笔记本电脑/个人电脑。我们将使用 Go 1.8+作为我们的编译器版本，并安装许多第三方软件包，因此需要一个可用的互联网连接。

我们还将在最后的章节中使用 Docker 来解释 API 网关的概念。建议使用 Docker V17.0+。如果 Windows 用户在本书中的任何示例中遇到原生 Go 安装的问题，请使用 Docker for Windows 并运行 Ubuntu 容器，这样会更灵活；有关更多详细信息，请参阅[`www.docker.com/docker-windows`](https://www.docker.com/docker-windows)。

在深入阅读本书之前，请在[`tour.golang.org/welcome/1`](https://tour.golang.org/welcome/1)上复习您的语言基础知识。

尽管这些是基本要求，但我们将在必要时为您安装指导。

# 这本书适合谁

这本书适用于所有熟悉 Go 语言并希望学习 REST API 开发的开发人员。即使是资深工程师也可以享受这本书，因为它涵盖了许多尖端概念，如微服务、协议缓冲区和 gRPC。

已经熟悉 REST 概念并从其他平台（如 Python 和 Ruby）进入 Go 世界的开发人员也可以受益匪浅。

# 约定

在本书中，您将找到许多文本样式，用于区分不同类型的信息。以下是一些这些样式的示例及其含义的解释。

文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下："将前面的程序命名为`basicHandler.go`。"

代码块设置如下：

```go
{
 "ID": 1,
 "DriverName": "Menaka",
 "OperatingStatus": true
 }
```

任何命令行输入或输出都以以下形式编写：

```go
go run customMux.go
```

**新术语**和**重要单词**以粗体显示。您在屏幕上看到的单词，例如菜单或对话框中的单词，会在文本中以这种方式出现："它返回消息，说成功登录。"

警告或重要说明会以这样的形式出现在一个框中。

提示和技巧会出现在这样的形式。
