# 前言

Go 编程语言已被开发者迅速采用，用于构建 Web 应用程序。凭借其令人印象深刻的性能和易于开发的特点，Go 语言得到了各种开源框架的支持，用于构建可扩展和高性能的 Web 服务和应用程序。本书是一本全面的指南，涵盖了使用 Go 语言进行全栈开发的各个方面。随着你在本书中的学习进展，你将逐步从头开始构建一个在线乐器商店应用程序。

这本内容清晰、示例丰富的书籍从对 Go 开发的实际接触开始，涵盖了 Go 的构建块以及 Go 的强大并发特性。然后，我们将探索流行的 React 框架，并使用它来构建应用程序的前端。从那里，你将使用 Gin 框架构建 RESTful Web API，Gin 是一个非常强大且流行的 Go 框架，用于 Web API。之后，我们将深入探讨重要的软件后端概念，例如使用 **对象关系映射**（**ORM**）连接数据库、为你的服务设计路由、保护你的服务，甚至使用流行的 Stripe API 来处理信用卡支付。我们还将介绍如何在生产环境中高效地测试和基准测试你的应用程序。在结论章节中，我们将通过学习 GopherJS 来介绍纯 Go 中的同构开发。

在阅读完本书之后，你将自信地承担使用 Go 语言构建全栈 Web 应用程序的任务。

# 这本书面向谁

本书将吸引那些计划开始使用 Go 语言构建功能齐全的全栈 Web 应用程序的开发者。本书假定读者具备 Go 语言和 JavaScript 的基本知识。也期望读者对 HTML 和 CSS 有一定的了解。本书的目标读者是那些希望转向 Go 语言的 Web 开发者。

# 本书涵盖的内容

第一章，*欢迎来到全栈 Go*，介绍了本书涵盖的主题，以及本书将构建的应用程序架构概述。本章还为我们提供了对本书内容的预期一瞥。

第二章，*Go 语言构建块*，介绍了 Go 语言的构建块，以及如何利用它们构建简单应用程序。它从介绍 Go 的变量、条件语句、循环和函数的语法开始。然后，它涵盖了如何在 Go 中构建数据结构，以及如何将方法附加到它们上。最后，它通过学习如何编写可以描述我们程序行为的接口来结束。

第三章，*Go 并发*，涵盖了 Go 语言的并发特性。它涵盖了 Go 的并发原语，如 goroutines、channels 和 select 语句，然后转向涵盖一些对生产并发软件重要的概念，如锁和等待组。

第四章，*使用 React.js 的前端开发*，涵盖了极受欢迎的 React.js 框架的构建块。它首先查看 React 组件，这是 React 框架的基础。从那里，它涵盖了用于向组件传递数据的 props。最后，它描述了处理状态和使用开发者工具。

第五章，*构建 GoMusic 的前端*，使用上一章获得的知识构建 GoMusic 的前端。它构建了 GoMusic 商店所需的 React 组件，并利用 React 的开发者工具来调试前端。本章将涵盖大部分前端代码。

第六章，*使用 Gin 框架的 Go 语言 RESTful Web APIs*，为您介绍了 RESTful Web APIs 和 Gin 框架。接着深入探讨了 Gin 的关键构建块，并解释了如何使用它开始编写 Web APIs。它还涵盖了如何在 Gin 中执行 HTTP 请求路由以及如何分组 HTTP 请求路由。

第七章，*使用 Gin 和 React 的高级 Web Go 应用*，讨论了关于 Gin Web 框架和我们的后端 Web API 的更高级主题。它涵盖了重要的实用主题，例如如何通过使用中间件扩展功能、如何实现用户认证、如何附加日志以及如何向我们的模型绑定添加验证。它还涵盖了 ORM 的概念，以及如何通过 Go ORM 将我们的 Web API 后端连接到 MySQL 数据库。最后，它将涵盖一些剩余的 React 代码，然后我们将讨论如何将 React 应用程序连接到我们的 Go 后端。本章还涵盖了如何构建和部署 React 应用程序到生产环境。

第八章，*测试和基准测试您的 Web API*，讨论了如何测试和基准测试 Go 应用程序。它将帮助我们了解可以使用测试包使用的类型和方法，以创建可以与代码集成的单元测试。然后它将专注于学习如何基准测试我们的代码以检查其性能。

第九章，*使用 GopherJS 介绍同构 Go*，涵盖了 GopherJS，这是一个非常流行的开源项目，它帮助将 Go 代码转换为 JavaScript，从而实际上允许我们用 Go 编写前端代码。如果你想在 Go 而不是 JavaScript 中编写前端，GopherJS 是这样做的方法。本章将讨论 GopherJS 的基础知识，然后介绍一些示例和用例。我们还将讨论如何通过在 GopherJS 中构建一个简单的 React 应用程序来将 GopherJS 与 React.js 结合使用。

第十章，*从这里开始去哪里？*提供了从这里继续学习旅程的建议。它讨论了云原生架构、容器和 React Native 移动应用开发。

# 为了充分利用这本书

要最大限度地从这本书中获益，最好的方法是逐章阅读。尝试在章节中练习代码示例。大多数章节都包含一个概述章节中代码所需工具和软件的部分。每个章节的代码将在 GitHub 上提供。

# 下载示例代码文件

你可以从[www.packt.com](http://www.packt.com)的账户下载这本书的示例代码文件。如果你在其他地方购买了这本书，你可以访问[www.packt.com/support](http://www.packt.com/support)并注册，以便将文件直接通过电子邮件发送给你。

你可以通过以下步骤下载代码文件：

1.  在[www.packt.com](http://www.packt.com)登录或注册。

1.  选择“支持”选项卡。

1.  点击“代码下载与勘误”。

1.  在搜索框中输入书名，并遵循屏幕上的说明。

一旦文件下载完成，请确保使用最新版本的以下软件解压缩或提取文件夹：

+   Windows 版的 WinRAR/7-Zip

+   Mac 版的 Zipeg/iZip/UnRarX

+   Linux 版的 7-Zip/PeaZip

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go`](https://github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go)。如果代码有更新，它将在现有的 GitHub 仓库中更新。

我们还有其他来自我们丰富的图书和视频目录的代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)找到。查看它们吧！

# 下载彩色图像

我们还提供了一份包含本书中使用的截图/图表的彩色图像的 PDF 文件。你可以从这里下载：[`www.packtpub.com/sites/default/files/downloads/9781789130751_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/9781789130751_ColorImages.pdf)。

# 使用的约定

在本书中使用了多种文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 账号。以下是一个示例：“我们还将介绍一些特定于 Go 的概念，例如 slice、`panic` 和 `defer`。”

代码块设置如下：

```go
package mypackage
```

当我们希望您注意代码块中的特定部分时，相关的行或项目将以粗体显示：

```go
type Student struct{
    Person
    studentId int
}

func (s Student) GetStudentID()int{
    return s.studentId
}
```

任何命令行输入或输出都应如下编写：

```go
go install
```

**粗体**：表示新术语、重要单词或您在屏幕上看到的单词。例如，菜单或对话框中的单词在文本中显示如下。以下是一个示例：“这被称为**类型推断**，因为您从提供的值中推断变量类型。”

警告或重要注意事项看起来像这样。

小贴士和技巧看起来像这样。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**：如果您对本书的任何方面有疑问，请在邮件主题中提及书名，并将邮件发送至 `customercare@packtpub.com`。

**勘误**：尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，我们将不胜感激，如果您能向我们报告，请访问 [www.packt.com/submit-errata](http://www.packt.com/submit-errata)，选择您的书，点击勘误提交表单链接，并输入详细信息。

**盗版**：如果您在互联网上以任何形式发现我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过 `copyright@packt.com` 与我们联系，并提供材料的链接。

**如果您有兴趣成为作者**：如果您在某个领域有专业知识，并且您有兴趣撰写或为本书做出贡献，请访问 [authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦您阅读并使用了本书，为何不在购买该书的网站上留下评论呢？潜在读者可以查看并使用您的客观意见来做出购买决定，Packt 可以了解您对我们产品的看法，我们的作者也可以看到他们对本书的反馈。谢谢！

如需了解 Packt 的更多信息，请访问 [packt.com](http://www.packt.com/)。
