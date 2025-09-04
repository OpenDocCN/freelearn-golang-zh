# 前言

我与这本书的联系一直是一段难忘的旅程。三年前，当 Packt 联系我为他们写一本书时，我并不确定我能否首先实现这个目标。但是，在家人、朋友和工作中几位优秀的导师的支持下，我成功地做到了。我总是有解释事物的习惯，并想将这种习惯转化为更重要的形式——一本书。这种愿望最终演变成了一本名为《使用 Go 构建 RESTful Web 服务》的书，于 2017 年出版。回顾过去，这根本不是一个坏主意。

除了全职的软件开发工作外，我还是一个开源博客作者。我写的内容是我工作中所学到的。每个月，我都会开发许多功能，修复许多错误，并审查许多合并请求。我将所有这些经验转化为文章。这本书是许多这些经验的宝贵集合。你可以问我是什么促使我写这本书？那就是分享我所知道的强烈愿望。软件工程是一项硬技能，它一直是一门实践学科，与学术研究不同。这驱使我写了《动手实践 Go 的 RESTful Web 服务》，这是我的第一本书的超级续集。

在这个信息技术时代，产品通过**应用程序编程接口**（**APIs**）相互交流。在过去十年中，新一代 Web 语言（如 Python、JavaScript（Node.js）和 Go）的兴起，展示了与传统 Web 开发（如 ASP.NET 和 Java 的 Spring）不同的方法。特别是，Go 编程语言在企业和原型领域之间找到了一个完美的平衡点。我们可以将 Go 同时与“Python 是原型”和“Java 是企业”进行比较。一些最好的开源工具，如 Docker、Terraform 和 Kubernetes，都是用 Go 编写的。谷歌大量使用它来为其内部服务。你可以在[`github.com/golang/go/wiki/GoUsers`](https://github.com/golang/go/wiki/GoUsers)上看到使用 Go 的公司列表。

由于代码简洁、严格的类型检查和对并发的支持，Go 是编写现代 Web 服务器的更好语言。一个中级 Go 开发者可以通过了解如何使用 Go 创建 RESTful 服务而受益匪浅。这本书试图让读者对 Web 服务开发感到舒适。记住，这是一本实践指南。

行业专家建议，不久的将来，Python 可能会进一步进入数据科学领域，这可能会在 Web 开发领域造成真空。Go 拥有填补这一空白的全部资格。从单体到微服务的范式转变，以及对于强大 API 接口的需求，可能会使 Go 在解释型语言之上占据更高的地位。

尽管这本书不是一本食谱书，但它为读者在阅读过程中的旅程提供了许多技巧和窍门。这本书是为希望使用 Go 开发 RESTful Web 服务和 API 的软件开发人员和 Web 开发人员而写的。它还将帮助对使用 Go 进行 Web 开发感兴趣的 Python 和 Node.js 开发者。

我希望您喜欢这本书，并且它能帮助您的事业达到新的高度！

# 本书面向的对象

这本书是为任何熟悉 Go 语言并希望学习 REST API 开发的 Go 开发者而写的。即使是资深工程师也会喜欢这本书，因为它讨论了许多前沿概念，例如构建微服务、使用 GraphQL 开发 API、使用协议缓冲区、异步 API 设计和基础设施即代码。

已经熟悉 REST 概念并从其他平台（如 Python 和 Ruby）进入 Go 世界的开发者，阅读这本书也会受益匪浅。

# 本书涵盖的内容

第一章，*开始 REST API 开发*，讨论了 REST 架构和动词的基本原理。

第二章，*处理我们的 REST 服务的路由*，描述了如何定义 REST API 的基本路由和处理函数。

第三章，*与中间件和 RPC 一起工作*，涵盖了与中间件处理程序和基本 RPC 一起工作的内容。

第四章，*使用流行的 Go 框架简化 RESTful 服务*，展示了使用几个开源框架快速原型设计 REST API。

第五章，*使用 MongoDB 和 Go 创建 REST API*，解释了如何将 MongoDB 用作 REST API 的后端存储。

第六章，*与协议缓冲区和 gRPC 一起工作*，展示了如何使用协议缓冲区和 gRPC 通过 HTTP/JSON 获得性能提升。

第七章，*与 PostgreSQL、JSON 和 Go 一起工作*，解释了如何将 PostgreSQL 用作后端存储并利用 JSON 存储来创建 REST API。

第八章，*在 Go 中构建 REST API 客户端*，介绍了构建客户端软件和 API 测试工具的技术。

第九章，*异步 API 设计*，展示了通过利用异步设计模式来扩展 API 的技术。

第十章，*GraphQL 和 Go*，讨论了与 REST 不同的 API 查询语言。

第十一章，*使用微服务扩展我们的 REST API*，介绍了使用 Go Micro 构建微服务。

第十二章，*为部署容器化 REST 服务*，展示了如何为 API 部署准备容器化生态系统。

第十三章，*在亚马逊网络服务上部署 REST 服务*，展示了如何使用基础设施即代码将容器化生态系统部署到 AWS 云。

第十四章，*为我们的 REST 服务处理身份验证*，讨论了使用简单身份验证和**JSON Web Tokens**（JWT）来保护 API。

# 为了充分利用本书

对于本书，您需要一个安装了 Linux（Ubuntu 18.04）、macOS X >=10.13 或 Windows 的笔记本电脑/PC。我们将使用 Go 1.13.x 作为编译器的版本，并将安装许多第三方包，因此需要一个有效的互联网连接。

我们还将在最后几章中使用 Docker 来解释 API 网关的概念。建议使用 Docker 的最新稳定版本。如果 Windows 用户在本地 Go 安装或使用 CURL 进行任何示例时遇到问题，请使用 Docker Desktop for Windows 并运行 Ubuntu 容器来测试您的代码示例；有关更多详细信息，请参阅[`www.docker.com/docker-windows`](https://www.docker.com/docker-windows)。

在深入本书之前，请先在[`tour.golang.org/welcome/1`](https://tour.golang.org/welcome/1)刷新您的语言基础知识。

尽管这些是基本要求，但我们在需要时将引导您完成安装。

# 下载示例代码文件

您可以从[www.packt.com](http://www.packt.com/)的账户下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packtpub.com/support](https://www.packtpub.com/support)并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在[www.packt.com](http://www.packt.com/)上登录或注册。

1.  选择“支持”标签。

1.  点击“代码下载”。

1.  在搜索框中输入本书的名称，并遵循屏幕上的说明。

下载文件后，请确保您使用最新版本的以下软件解压缩或提取文件夹：

+   Windows 的 WinRAR/7-Zip

+   Mac 的 Zipeg/iZip/UnRarX

+   Linux 的 7-Zip/PeaZip

本书代码包也托管在 GitHub 上，地址为**[`github.com/PacktPublishing/Hands-On-Restful-Web-services-with-Go`](https://github.com/PacktPublishing/Hands-On-Restful-Web-services-with-Go)**。如果代码有更新，它将在现有的 GitHub 仓库中更新。

我们还有其他来自我们丰富的图书和视频目录的代码包可供选择，请访问**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**。查看它们！

# 下载彩色图像

我们还提供了一个包含本书中使用的截图/图表彩色图像的 PDF 文件。您可以从这里下载：[`static.packt-cdn.com/downloads/9781838643577_ColorImages.pdf`](https://static.packt-cdn.com/downloads/9781838643577_ColorImages.pdf)。

# 使用的约定

本书使用了多种文本约定。

`CodeInText`: 表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“将前面的程序命名为`basicHandler.go`。”

代码块如下设置：

```go
{
 "ID": 1,
 "DriverName": "Menaka",
}
```

当我们希望您注意代码块中的特定部分时，相关的行或项目将以粗体显示：

```go
{
 "ID": 1,
 "DriverName": "Menaka",
}
```

任何命令行输入或输出都如下所示：

```go
> go run customMux.go
```

**粗体**: 表示新术语、重要单词或您在屏幕上看到的单词。例如，菜单或对话框中的单词在文本中如下所示。以下是一个示例：“它返回一条消息说已成功登录。”

警告或重要注意事项如下所示。

技巧和窍门如下所示。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**: 如果您对本书的任何方面有疑问，请在邮件主题中提及书名，并通过`customercare@packtpub.com`给我们发送电子邮件。

**勘误**: 尽管我们已经尽最大努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，如果您能向我们报告，我们将不胜感激。请访问 [www.packtpub.com/support/errata](https://www.packtpub.com/support/errata)，选择您的书籍，点击勘误提交表单链接，并输入详细信息。

**盗版**: 如果您在互联网上以任何形式发现我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过`copyright@packt.com`与我们联系，并提供材料的链接。

**如果您有兴趣成为作者**: 如果您在某个领域有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问 [authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦您阅读并使用过这本书，为何不在您购买它的网站上留下评论呢？潜在读者可以查看并使用您的客观意见来做出购买决定，Packt 公司可以了解您对我们产品的看法，我们的作者也可以看到他们对书籍的反馈。谢谢！

有关 Packt 的更多信息，请访问 [packt.com](http://www.packt.com/)。
