# 前言

Go 编程语言是在 Google 开发的，用于解决他们在为其基础设施开发软件时遇到的问题。他们需要一种静态类型的语言，不会减慢开发人员的速度，可以立即编译和执行，利用多核处理器，并使跨分布式系统的工作变得轻松。

《使用 Go 进行分布式计算》的使命是使并发和并行推理变得轻松，并为读者提供设计和实现此类程序的信心。我们将首先深入探讨 goroutines 和 channels 背后的核心概念，这是 Go 语言构建的两个基本概念。接下来，我们将使用 Go 和 Go 标准库设计和构建一个分布式搜索引擎。

## 这本书是为谁准备的

这本书适用于熟悉 Golang 语法并对基本 Go 开发有一定了解的开发人员。如果您经历过 Web 应用程序产品周期，将会更有优势，尽管这并非必需。

## 本书涵盖的内容

第一章《Go 的开发环境》涵盖了开始使用 Go 和本书其余部分所需的一系列主题和概念。其中一些主题包括 Docker 和 Go 中的测试。

第二章《理解 Goroutines》介绍了并发和并行主题，然后深入探讨了 goroutines 的实现细节、Go 的运行时调度器等。

第三章《Channels and Messages》首先解释了控制并行性的复杂性，然后介绍了使用不同类型的通道来控制并行性的策略。

第四章《RESTful Web》提供了开始在 Go 中设计和构建 REST API 所需的所有上下文和知识。我们还将讨论使用不同可用方法与 REST API 服务器进行交互。

第五章《介绍 Goophr》开始讨论分布式搜索引擎的含义，使用 OpenAPI 规范描述 REST API，并描述搜索引擎组件的责任。最后，我们将描述项目结构。

第六章《Goophr Concierge》深入介绍了 Goophr 的第一个组件，详细描述了该组件应该如何工作。借助架构和逻辑流程图，进一步强化了这些概念。最后，我们将看看如何实现和测试该组件。

第七章《Goophr 图书管理员》详细介绍了负责维护搜索词索引的组件。我们还将讨论如何搜索给定的词语以及如何对搜索结果进行排序等。最后，我们将看看如何实现和测试该组件。

第八章《部署 Goophr》将前三章中实现的所有内容汇集起来，并在本地系统上启动应用程序。然后，我们将通过 REST API 添加一些文档并对其进行搜索，以测试我们的设计。

第九章《Web 规模架构的基础》是一个广泛而复杂的主题介绍，讨论如何设计和扩展系统以满足 Web 规模的需求。我们将从单个运行在单个服务器上的单体实例开始，并将其扩展到跨越多个区域，具有冗余保障以确保服务永远不会中断等。

## 充分利用本书

+   本书中的材料旨在实现动手操作。在整本书中，我们都在努力提供所有相关信息，以便读者可以选择自己尝试解决问题，然后再参考书中提供的解决方案。

+   书中的代码除了标准库外没有任何 Go 依赖。这样做是为了确保书中提供的代码示例永远不会改变，也让我们能够探索标准库。

+   书中的源代码应放置在`$GOPATH/src/distributed-go`目录下。给出的示例源代码将位于`$GOPATH/src/distributed-go/chapterX`文件夹中，其中`X`代表章节编号。

+   从[`golang.org/`](https://golang.org/)和[`www.docker.com/community-edition`](https://www.docker.com/community-edition)网站下载并安装 Go 和 Docker

### 下载示例代码文件

您可以从[`www.packtpub.com`](http://www.packtpub.com/)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，可以访问[`www.packtpub.com/support`](http://www.packtpub.com/support)并注册，文件将直接发送到您的邮箱。

您可以按照以下步骤下载代码文件：

1.  在[`www.packtpub.com`](http://www.packtpub.com/support)登录或注册。

1.  选择“支持”选项卡。

1.  点击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明操作。

下载文件后，请确保使用以下最新版本解压或提取文件夹：

+   WinRAR / 7-Zip for Windows

+   Zipeg / iZip / UnRarX for Mac

+   7-Zip / PeaZip for Linux

本书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Distributed-Computing-with-Go`](https://github.com/PacktPublishing/Distributed-Computing-with-Go)。如果代码有更新，将在现有的 GitHub 存储库中进行更新。

我们还有其他代码包来自我们丰富的图书和视频目录，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到。快去看看吧！

### 下载彩色图片

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图片。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/DistributedComputingwithGo_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/DistributedComputingwithGo_ColorImages.pdf)。

### 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。例如，“现在我们已经准备好所有的代码，让我们使用`Dockerfile`文件构建 Docker 镜像。”

代码块设置如下：

```go
// addInt.go 

package main 

func addInt(numbers ...int) int { 
    sum := 0 
    for _, num := range numbers { 
        sum += num 
    } 
    return sum 
} 
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目会以粗体显示：

```go
// addInt.go 

package main 

func addInt(numbers ...int) int { 
    sum := 0 
    for _, num := range numbers { 
        sum += num 
    } 
    return sum 
} 
```

任何命令行输入或输出都将按以下方式编写：

```go
$ cd docker
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词，例如在菜单或对话框中，也会在文本中出现。例如，“从**管理**面板中选择**系统信息**。”

警告或重要提示会这样出现。

提示和技巧会这样出现。
