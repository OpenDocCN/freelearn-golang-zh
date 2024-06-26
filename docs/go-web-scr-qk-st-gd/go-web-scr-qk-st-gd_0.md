# 前言

互联网是一个充满有趣信息和见解的地方，等待被获取。就像金块一样，这些零散的数据片段可以被收集、过滤、组合和精炼，从而产生极具价值的产品。凭借正确的知识、技能和一点创造力，您可以构建一个能够支撑数十亿美元公司的网络爬虫。为了支持这一点，您需要使用最适合工作的工具，首先是一种专为速度、简单和安全性而构建的编程语言。

Go 编程语言结合了前辈的最佳理念和前沿思想，摒弃了不必要的废话，产生了一套锋利的工具和清晰的架构。通过 Go 标准库和开源贡献者的项目，您拥有构建任何规模的网络爬虫所需的一切。

# 这本书适合谁

这本书适合有一点编码经验的人，对如何构建快速高效的网络爬虫感兴趣。

# 本书涵盖的内容

第一章《介绍网络爬虫和 Go》解释了什么是网络爬虫，以及如何安装 Go 编程语言和工具。

第二章《请求/响应周期》概述了 HTTP 请求和响应的结构，并解释了如何使用 Go 进行制作和处理。

第三章《网络爬虫礼仪》解释了如何构建一个遵循最佳实践和推荐的网络爬虫，以有效地爬取网络，同时尊重他人。

第四章《解析 HTML》展示了如何使用各种工具从 HTML 页面中解析信息。

第五章《网络爬虫导航》演示了有效浏览网站的最佳方法。

第六章《保护您的网络爬虫》解释了如何使用各种工具安全、可靠地浏览互联网。

第七章《并发爬取》介绍了 Go 并发模型，并解释了如何构建高效的网络爬虫。

第八章《100 倍速爬取》提供了构建大规模网络爬虫的蓝图，并提供了一些来自开源社区的示例。

# 为了充分利用本书

为了充分利用本书，您应该熟悉您的终端或命令提示符，确保您有良好的互联网连接，并阅读每一章，即使您认为您已经知道了。本书的读者应该以开放的心态思考网络爬虫应该如何行动，并学习当前的最佳实践和适当的礼仪。本书还专注于 Go 编程语言，涵盖了安装、基本命令、标准库和包管理，因此对 Go 的一些熟悉将有所帮助，因为本书对语言进行了广泛的涵盖，只深入到了进行网络爬取所需的深度。为了能够运行本书中的大部分代码，读者应该熟悉他们的终端或命令提示符，以便运行示例等其他任务。

# 下载示例代码文件

您可以从[www.packt.com](http://www.packt.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packt.com/support](http://www.packt.com/support)并注册，以便直接通过电子邮件接收文件。

您可以按照以下步骤下载代码文件：

1.  在[www.packt.com](http://www.packt.com)登录或注册

1.  选择“支持”选项卡

1.  点击“代码下载和勘误”

1.  在搜索框中输入书名，然后按照屏幕上的说明操作

一旦文件下载完成，请确保使用以下最新版本的软件解压缩或提取文件夹：

+   WinRAR/7-Zip 适用于 Windows

+   Zipeg/iZip/UnRarX 适用于 Mac

+   7-Zip/PeaZip 适用于 Linux

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Go-Web-Scraping-Quick-Start-Guide`](https://github.com/PacktPublishing/Go-Web-Scraping-Quick-Start-Guide)。如果代码有更新，将在现有的 GitHub 存储库中更新。

我们还有其他代码包，来自我们丰富的书籍和视频目录，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。去看看吧！

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。例如："这是使用`net/http`包的默认 HTTP 客户端请求`index.html`资源。"

代码块设置如下：

```go
POST /login HTTP/1.1
Host: myprotectedsite.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 38

username=myuser&password=supersecretpw
```

任何命令行输入或输出都以以下方式书写：

```go
go run main.go
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会在文本中以这种方式出现。例如："在这种情况下，您将收到 500 内部服务器错误的状态码。"

警告或重要说明显示为这样。

提示和技巧显示为这样。
