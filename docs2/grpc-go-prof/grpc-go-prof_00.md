# 前言

在高度互联的微服务世界中，gRPC 已经成为一项重要的通信技术。通过站在 Protobuf 的肩膀上并在通信过程中实现必备功能，gRPC 提供了可靠、高效且用户友好的 API。在这本书中，你将探索为什么是这样，如何编写这些 API，以及如何在生产环境中使用它们。总体目标是让你了解如何使用 gRPC，以及 gRPC 的工作原理。首先，你将了解在使用 gRPC 之前需要了解的网络和 Protobuf 概念。然后，你将看到如何编写单例和不同类型的流 API。最后，在本书的其余部分，你将学习 gRPC 的功能和如何创建生产级别的 API。

# 本书面向对象

*如果你是一名软件工程师或架构师，曾经为编写 API 而挣扎，或者你在寻找现有 API 的更高效替代方案，这本书就是为你准备的。*

# 本书涵盖内容

*第一章*, 《网络基础》，将教你关于 gRPC 背后的网络概念。

*第二章*, 《Protobuf 初学者指南》, 将帮助你理解 Protobuf 对于高效通信的重要性。

*第三章*, 《gRPC 简介》，将让你了解为什么 gRPC 比传统的 REST API 更高效。

*第四章*, 《设置项目》，将标志着你进入 gRPC 世界的开始。

*第五章*, 《gRPC 端点类型》，将描述如何编写单例、服务器流、客户端流和双向流 API。

*第六章*, 《设计有效的 API》，将概述设计 gRPC API 时的不同权衡。

*第七章*, 《开箱即用功能》，将介绍 gRPC 提供的主要开箱即用功能。

*第八章*, 《更多必备功能》，将解释社区项目如何使你的 API 更加强大和安全。

*第九章*, 《生产级 API》，将教你如何测试、调试和部署你的 API。

# 为了最大限度地利用本书

*你必须已经熟悉 Go 语言。尽管大多数时候你只需要基本的编码技能，但理解 Go 的并发概念将会很有帮助。*

| **本书涵盖的软件/硬件** | **操作系统要求** |
| --- | --- |
| Go 1.20.4 | Windows, macOS, 或 Linux |
| Protobuf 23.2 | Windows, macOS, 或 Linux |
| gRPC 1.55.0 | Windows, macOS, 或 Linux |
| Buf 1.15.1 | Windows, macOS, 或 Linux |
| Bazel 6.2.1 | Windows, macOS, 或 Linux |

**如果您正在使用本书的数字版，我们建议您亲自输入代码或从本书的 GitHub 仓库（下一节中提供链接）获取代码。这样做将帮助您避免与代码的复制和粘贴相关的任何潜在错误。**

# 下载示例代码文件

您可以从 GitHub 在[`github.com/PacktPublishing/gRPC-Go-for-Professionals`](https://github.com/PacktPublishing/gRPC-Go-for-Professionals)下载本书的示例代码文件。如果代码有更新，它将在 GitHub 仓库中更新。

我们还有其他来自我们丰富图书和视频目录的代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)找到。查看它们吧！

# 下载彩色图像

我们还提供包含本书中使用的截图和图表的彩色图像的 PDF 文件。您可以从这里下载：[`packt.link/LEms7`](https://packt.link/LEms7)。

# 使用的约定

本书使用了多种文本约定。

`文本中的代码`: 表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“使用插件，我们可以在 protoc 中使用`--validate_out`选项。”

代码块按照以下方式设置：

```go
message AddTaskRequest {
  string description = 1;
  google.protobuf.Timestamp due_date = 2;
}
```

当我们希望您注意代码块中的特定部分时，相关的行或项目将以粗体显示：

```go
proto/todo/v2
├── todo.pb.go
├── todo.pb.validate.go
├── todo.proto
└── todo_grpc.pb.go
```

任何命令行输入或输出都按照以下方式编写：

```go
$ bazel run //:gazelle-update-repos
```

小贴士或重要注意事项

看起来像这样。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**: 如果您对本书的任何方面有疑问，请通过 customercare@packtpub.com 给我们发邮件，并在邮件主题中提及书名。

**勘误表**: 尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果您在本书中发现错误，我们将不胜感激，如果您能向我们报告，我们将不胜感激。请访问[www.packtpub.com/support/errata](http://www.packtpub.com/support/errata)并填写表格。

**盗版**: 如果您在互联网上以任何形式发现我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过版权@packt.com 与我们联系，并提供材料的链接。

**如果您有兴趣成为作者**: 如果您在某个主题上具有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com)。

# 分享您的想法

一旦您阅读了《专业级 gRPC Go》，我们很乐意听到您的想法！请点击此处直接进入此书的亚马逊评论页面并分享您的反馈。

您的评论对我们和科技社区非常重要，并将帮助我们确保我们提供高质量的内容。

# 下载本书的免费 PDF 副本

感谢您购买本书！

您喜欢在路上阅读，但无法随身携带您的印刷书籍吗？您的电子书购买是否与您选择的设备不兼容？

别担心，现在每购买一本 Packt 图书，您都可以免费获得该书的 DRM 免费 PDF 版本。

在任何地方、任何设备上阅读。直接从您最喜欢的技术书籍中搜索、复制和粘贴代码到您的应用程序中。

优惠远不止于此，您还可以获得独家折扣、时事通讯和每天收件箱中的精彩免费内容。

按照以下简单步骤获取这些好处：

1.  扫描下面的二维码或访问以下链接![img/B19664_QR_Free_PDF.jpg](img/B19664_QR_Free_PDF.jpg)

https://packt.link/free-ebook/9781837638840

1.  提交您的购买证明

1.  就这样！我们将直接将您的免费 PDF 和其他优惠发送到您的电子邮件中
