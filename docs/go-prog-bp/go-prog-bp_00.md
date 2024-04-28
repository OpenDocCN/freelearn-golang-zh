# 前言

我决定写《Go 编程蓝图》是因为我想驱散一个谣言，即相对年轻的 Go 语言和社区不适合快速编写和迭代软件。我有一个朋友，他可以在一个周末内用 Ruby on Rails 开发完整的应用程序，通过混合现有的宝石和库；Rails 作为一个平台已经以其快速开发而闻名。由于我在 Go 和不断增长的开源软件包中也做到了同样的事情，我想分享一些真实世界的例子，展示我们如何可以快速构建和发布表现出色的软件，从第一天起就准备好扩展，这是 Rails 无法与之竞争的。当然，大多数的可扩展性发生在语言之外，但是像 Go 内置的并发性这样的特性意味着即使在最基本的硬件上，你也可以获得一些非常令人印象深刻的结果，这让你在事情开始变得真实时就能提前开始。

这本书探讨了五个非常不同的项目，其中任何一个都可以成为一个真正的创业基础。无论是低延迟的聊天应用程序、域名建议工具、建立在 Twitter 上的社交投票和选举服务，还是由 Google Places 提供支持的随机夜生活生成器，每一章都涉及大多数使用 Go 编写的产品或服务需要解决的各种问题。我在书中提出的解决方案只是解决每个项目的许多方法之一，我鼓励你自己对我如何解决它们做出自己的判断。概念比代码本身更重要，但你希望能够从中学到一些技巧和窍门，可以加入到你的 Go 工具包中。

我写这本书的过程可能会很有趣，因为它代表了许多敏捷开发者采用的一些哲学。我开始给自己一个挑战，即在深入研究并编写第一个版本之前，先构建一个真正可部署的产品（尽管是一个简单的产品；如果你愿意，可以称之为最小可行产品）。一旦我让它运行起来，我会从头开始重写它。小说家和记者们多次说过写作的艺术就是重写；我发现这对软件也是真实的。第一次我们写代码时，我们真正做的只是了解问题以及可能解决问题的方式，并将一些想法从我们的脑海中记录到纸上（或文本编辑器中）。第二次写代码时，我们将应用我们的新知识来真正解决问题。如果你从未尝试过这样做，试一试吧——你可能会发现，就像我一样，你的代码质量会显著提高。这并不意味着第二次就是最后一次——软件是不断演进的，我们应该尽量保持它的成本低廉和可替换性，这样如果某些部分过时或开始妨碍我们，我们也不介意将其丢弃。

我所有的代码都遵循测试驱动开发（TDD）的实践，其中一些我们将在章节中一起完成，而一些你只会在最终代码中看到结果。即使在印刷版中没有包含，所有的测试代码都可以在本书的 GitHub 存储库中找到。

一旦我完成了我的测试驱动的第二个版本，我会开始撰写描述我做了什么以及为什么这样做的章节。在大多数情况下，我采取的迭代方法被省略在书中，因为这只会增加页面的调整和编辑，这可能会让读者感到沮丧。然而，在一些情况下，我们将一起进行迭代，以了解渐进改进和小迭代的过程（从简单开始，只在绝对必要时引入复杂性）如何应用于编写 Go 软件包和程序。

我在 2012 年从英国搬到美国，但这并不是为什么这些章节以美式英语撰写的原因；这是出版商的要求。我想这本书是针对美国读者的，或者可能是因为美式英语是计算机的标准语言（在英国的代码中，处理颜色的属性是不带 U 拼写的）。无论如何，我提前为任何跨大西洋的差错道歉；我知道程序员有多么苛刻。

任何问题、改进、建议或辩论（我喜欢 Go 社区以及核心团队和语言本身的主张）都是非常欢迎的。这些可能最好在专门设置的书籍 GitHub 问题中进行，网址为[`github.com/matryer/goblueprints`](https://github.com/matryer/goblueprints)，以便每个人都可以参与。

最后，如果有人基于这些项目创建了一家初创公司，或者在其他地方利用了它们，我会感到非常兴奋。我很想听听这方面的消息；你可以在 Twitter 上@matryer 给我发消息，让我知道情况。

# 本书内容包括

第一章 ，*使用 Web 套接字的聊天应用程序*，展示了如何构建一个完整的 Web 应用程序，允许多人在其 Web 浏览器中进行实时对话。我们看到 net/http 包如何让我们提供 HTML 页面，并与客户端的浏览器建立 Web 套接字连接。

第二章 ，*添加身份验证*，展示了如何向我们的聊天应用程序添加 OAuth，以便我们可以跟踪谁说了什么，但让他们可以使用 Google、Facebook 或 GitHub 登录。

第三章 ，*实现个人资料图片的三种方式*，解释了如何向聊天应用程序添加个人资料图片，可以从身份验证服务、[Gravitar.com](http://Gravitar.com)网站获取，或者允许用户从硬盘上传自己的图片。

第四章 ，*用命令行工具查找域名*，探讨了在 Go 中构建命令行工具的简易性，并将这些技能应用于解决为我们的聊天应用程序找到完美域名的问题。它还探讨了 Go 语言如何轻松利用标准输入和标准输出管道来生成一些非常强大的可组合工具。

第五章 ，*构建分布式系统并处理灵活数据*，解释了如何通过 NSQ 和 MongoDB 构建高度可扩展的 Twitter 投票和计票引擎，为民主的未来做准备。

第六章 ，*通过 RESTful 数据 Web 服务 API 公开数据和功能*，介绍了如何通过 JSON Web 服务公开我们在第五章 中构建的功能，具体来说，是如何通过包装 http.HandlerFunc 函数来实现强大的管道模式。

第七章 ，*随机推荐 Web 服务*，展示了如何使用 Google Places API 来生成基于位置的随机推荐 API，这是探索任何地区的一种有趣方式。它还探讨了保持内部数据结构私有的重要性，控制对相同数据的公共视图，以及如何在 Go 中实现枚举器。

第八章，*文件系统备份*，帮助我们构建一个简单但功能强大的文件系统备份工具，用于我们的代码项目，并探索使用 Go 标准库中的 os 包与文件系统进行交互。它还探讨了 Go 的接口如何允许简单的抽象产生强大的结果。

附录，*稳定的 Go 环境的良好实践*，教会我们如何从头开始在新机器上安装 Go，并讨论了我们可能拥有的一些环境选项以及它们将来可能产生的影响。我们还将考虑协作如何影响我们的一些决定，以及开源我们的包可能产生的影响。

# 本书所需内容

要编译和运行本书中的代码，您需要一台能够运行支持 Go 工具集的操作系统的计算机，可以在[`golang.org/doc/install#requirements`](https://golang.org/doc/install#requirements)找到支持的操作系统列表。

附录，*稳定的 Go 环境的良好实践*，提供了一些有用的提示，包括如何安装 Go 并设置开发环境，以及如何使用 GOPATH 环境变量。

# 本书适合对象

本书适用于所有 Go 程序员——从想通过构建真实项目来探索该语言的初学者到对如何以有趣的方式应用该语言感兴趣的专家 gophers。

# 读累了记得休息一会哦~

**公众号：古德猫宁李**

+   电子书搜索下载

+   书单分享

+   书友学习交流

**网站：**[沉金书屋 https://www.chenjin5.com](https://www.chenjin5.com)

+   电子书搜索下载

+   电子书打包资源分享

+   学习资源分享

# 约定

在本书中，您会发现一些文本样式，用于区分不同类型的信息。以下是一些这些样式的示例及其含义的解释。

文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 用户名显示如下："我们可以使用`import`关键字从其他包中使用功能，之前我们使用`go get`来下载它们。"

代码块设置如下：

```go
package meander
type Cost int8
const (
  _ Cost = iota
  Cost1
  Cost2
  Cost3
  Cost4
  Cost5
)
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目会以粗体显示：

```go
package meander
type Cost int8
const (

_ Cost = iota

  Cost1
  Cost2
  Cost3
  Cost4
  Cost5
)
```

任何命令行输入或输出都会以以下方式书写：

```go

go build -o project && ./project

```

**新术语**和**重要单词**以粗体显示。您在屏幕上看到的单词，比如菜单或对话框中的单词，会以这样的方式出现在文本中："一旦安装了 Xcode，您就打开**首选项**，然后导航到**下载**部分。

### 注意

警告或重要提示会以这样的方式出现在一个框中。

### 提示

提示和技巧会以这种方式出现。