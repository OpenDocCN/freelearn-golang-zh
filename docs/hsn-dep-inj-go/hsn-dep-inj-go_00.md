# 前言

你好！这本书旨在介绍如何在 Go 语言中进行依赖注入。也许你会惊讶地发现，在 Go 语言中有许多不同的方法可以应用依赖注入，在本书中，我们将讨论六种不同的方法，有时它们还可以相互补充。

依赖注入，像许多软件工程概念一样，很容易被误解，因此本文试图解决这个问题。它深入探讨了相关概念，如 SOLID 原则、代码异味和测试诱导的破坏，以便提供更广泛和更实用的视角。

《Go 语言依赖注入实战》的目标不仅是教会你如何应用依赖注入，还有何时、何地以及何时不应该应用。每种方法都有明确定义；我们讨论它的优缺点，以及何时最适合应用该方法。此外，每种方法都会使用重要的示例逐步应用。

尽管我非常喜欢依赖注入，但它并不总是适合所有情况。这本书还将帮助你发现应用依赖注入可能不是最佳选择的情况。

在介绍每种依赖注入方法时，我会请你停下来，退后一步，考虑以下问题。这种技术试图解决什么问题？在你应用这种方法后，你的代码会是什么样子？如果这些问题的答案不会很快出现，不要担心；到本书结束时，它们会出现的。

愉快的编码！

# 这本书适合谁

这本书适用于希望他们的代码易于阅读、测试和维护的开发人员。它适用于来自面向对象背景的开发人员，他们希望更多地了解 Go，以及相信高质量代码不仅仅是交付一个特定功能的开发人员。

毕竟，编写代码很容易。同样，让单个测试用例通过也很简单。创建代码，使得测试在添加额外功能的几个月或几年后仍然通过，这几乎是不可能的。

为了能够持续地以这个水平交付代码，我们需要很多巧妙的技巧。这本书希望不仅能够装备你这些技巧，还能够给你应用它们的智慧。

# 为了充分利用这本书

尽管依赖注入和本书中讨论的许多其他编程概念并不简单或直观，但本书在假定很少的知识的情况下介绍它们。

也就是说，我们假设以下内容：

+   你具有构建和测试 Go 代码的基本经验。

+   由于之前使用 Go 或面向对象的语言（如 Java 或 Scala）的经验，你对对象/类的概念感到舒适。

此外，至少对构建和使用基于 HTTP 的 REST API 有一定的了解会很有益。在第四章中，《ACME 注册服务简介》，我们将介绍一个示例 REST 服务，它将成为本书许多示例的基础。为了能够运行这个示例服务，你需要在开发环境中安装和配置 MySQL 数据库服务，并能够自定义提供的配置以匹配你的本地环境。本书提供的所有命令都是在 OSX 下开发和测试的，并且应该可以在任何基于 Linux 或 Unix 的系统上无需修改地工作。使用基于 Windows 的开发环境的开发人员需要在运行这些命令之前进行调整。

# 下载示例代码文件

你可以从[www.packt.com](http://www.packt.com)的账户中下载本书的示例代码文件。如果你在其他地方购买了这本书，你可以访问[www.packt.com/support](http://www.packt.com/support)并注册，以便直接通过电子邮件接收文件。

你可以通过以下步骤下载代码文件：

1.  登录或注册[www.packt.com](http://www.packt.com)。

1.  选择“支持”选项卡。

1.  单击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明进行操作。

下载文件后，请确保使用最新版本的解压缩软件解压缩文件夹：

+   WinRAR/7-Zip for Windows

+   Zipeg/iZip/UnRarX for Mac

+   7-Zip/PeaZip for Linux

该书的代码包也托管在 GitHub 上，网址为[**https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go**](https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go)。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还有其他代码包，来自我们丰富的图书和视频目录，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到。去看看吧！

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/Bookname_ColorImages.pdf`](http://www.packtpub.com/sites/default/files/downloads/Bookname_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。例如："将下载的`WebStorm-10*.dmg`磁盘映像文件挂载为系统中的另一个磁盘。"

代码块设置如下：

```go
html, body, #map {
 height: 100%; 
 margin: 0;
 padding: 0
}
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示：

```go
[default]
exten => s,1,Dial(Zap/1|30)
exten => s,2,Voicemail(u100)
exten => s,102,Voicemail(b100)
exten => i,1,Voicemail(s0)
```

任何命令行输入或输出都以以下方式编写：

```go
$ mkdir css
$ cd css
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会以这种方式出现在文本中。例如："从管理面板中选择系统信息。"

警告或重要提示会以这种方式出现。

提示和技巧会以这种方式出现。
