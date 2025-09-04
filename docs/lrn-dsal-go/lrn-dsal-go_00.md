# 前言

*使用 Go 学习数据结构和算法* 涵盖了计算机编程中的简单和高级概念。主要目标是选择正确的算法和数据结构来解决问题。本书解释了比较算法复杂性和数据结构的概念，这些概念涉及代码性能和效率。

Golang 在过去两年中一直是热门词汇，这一领域取得了巨大的进步。许多开发者和组织正在逐渐迁移到 Golang，采用其快速、轻量级和内置的并发功能。这意味着我们需要在这个不断发展的语言中有一个扎实的数据结构和算法基础。

# 本书面向对象

这本全面的书是为那些想要了解如何选择最佳数据结构和算法以帮助解决特定问题的开发者而编写的。一些基本的 Go 编程知识将是一个额外的优势。

这本书是为那些想要学习如何编写高效程序并使用适当的数据结构和算法的人而编写的。

# 为了充分利用这本书

我们假设的知识是关于矩阵、集合操作和统计概念等主题的基本编程语言和数学技能。读者应该具备根据流程图或指定算法编写伪代码的能力。编写功能性代码、测试、遵循指南以及在 Go 语言中构建复杂项目是我们假设的读者技能的先决条件。

# 下载示例代码文件

您可以从 [www.packt.com](http://www.packt.com) 的账户下载本书的示例代码文件。如果您在其他地方购买了这本书，您可以访问 [www.packt.com/support](http://www.packt.com/support) 并注册，以便将文件直接通过电子邮件发送给您。

您可以通过以下步骤下载代码文件：

1.  在 [www.packt.com](http://www.packt.com) 登录或注册。

1.  选择支持选项卡。

1.  点击代码下载和勘误表。

1.  在搜索框中输入书名，并遵循屏幕上的说明。

文件下载后，请确保使用最新版本的软件解压缩或提取文件夹：

+   WinRAR/7-Zip 用于 Windows

+   Zipeg/iZip/UnRarX 用于 Mac

+   7-Zip/PeaZip 用于 Linux

该书的代码包也托管在 GitHub 上，网址为 [`github.com/PacktPublishing/Learn-Data-Structures-and-Algorithms-with-Golang`](https://github.com/PacktPublishing/Learn-Data-Structures-and-Algorithms-with-Golang)。如果代码有更新，它将在现有的 GitHub 仓库中更新。

我们还有其他来自我们丰富的图书和视频目录的代码包，可在 [`github.com/PacktPublishing/`](https://github.com/PacktPublishing/) 获取。查看它们吧！

# 下载彩色图像

我们还提供了一份包含本书中使用的截图/图表彩色图像的 PDF 文件。您可以从这里下载：`www.packtpub.com/sites/default/files/downloads/9781789618501_ColorImages.pdf`。

# 使用的约定

本书使用了多种文本约定。

`CodeInText`: 表示文本中的代码词汇、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 昵称。以下是一个示例：“让我们在下一节中看看`len`函数。”

代码块设置为如下：

```go
//main package has examples shown
// in Hands-On Data Structures and algorithms with Go book
package main

// importing fmt package
import (
  "fmt"
)
// main method
func main() {
  fmt.Println("Hello World")
}
```

任何命令行输入或输出都写作如下：

```go
go build
./hello_world
```

**粗体**: 表示新术语、重要词汇或您在屏幕上看到的词汇。

警告或重要提示看起来像这样。

小贴士和技巧看起来像这样。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**: 如果您对本书的任何方面有疑问，请在邮件主题中提及书名，并通过`customercare@packtpub.com`发送邮件给我们。

**勘误**: 尽管我们已经尽一切努力确保内容的准确性，但错误仍然可能发生。如果您在这本书中发现了错误，我们将不胜感激，如果您能向我们报告。请访问[www.packt.com/submit-errata](http://www.packt.com/submit-errata)，选择您的书籍，点击勘误提交表单链接，并输入详细信息。

**盗版**: 如果您在互联网上以任何形式遇到我们作品的非法副本，如果您能提供位置地址或网站名称，我们将不胜感激。请通过`copyright@packt.com`与我们联系，并提供材料的链接。

**如果您有兴趣成为作者**: 如果您在某个主题上具有专业知识，并且您有兴趣撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦您阅读并使用了这本书，为什么不在您购买它的网站上留下评论呢？潜在读者可以查看并使用您的客观意见来做出购买决定，Packt 可以了解您对我们产品的看法，我们的作者也可以看到他们对书籍的反馈。谢谢！

有关 Packt 的更多信息，请访问[packt.com](http://www.packt.com/)。
