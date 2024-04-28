# 前言

感谢您购买本书。我们希望通过本书中的示例和项目，您能从 Go Web 开发新手变成一个能够承担面向生产的严肃项目的人。因此，本书在相对较高的水平上涉及了许多 Web 开发主题。在本书结束时，您应该能够实现一个非常简单的博客，包括显示、身份验证和评论，同时关注性能和安全性。

# 本书涵盖内容

第一章，“介绍和设置 Go”，通过向您展示如何设置环境和依赖项，以便您可以在 Go 中创建 Web 应用程序，开启了本书。

第二章，“服务和路由”，讨论了如何生成对某些 Web 端点做出反应的响应服务器。我们将探讨 net/http 之外的各种 URL 路由选项的优点。

第三章，“连接到数据”，实现数据库连接，开始获取要在我们的网站上呈现和操作的数据。

第四章，“使用模板”，涵盖了模板包，展示了我们如何向最终用户呈现和修改正在使用的数据。

第五章，“与 RESTful API 集成的前端”，详细介绍了如何创建一个基础 API 来驱动演示和功能。

第六章，“会话和 Cookie”，与我们的最终用户保持状态，从而使他们能够在页面之间保留信息，如身份验证。

第七章，“微服务和通信”，将一些功能拆分为微服务进行重新实现。本章将作为对微服务理念的轻微介绍。

第八章，“日志和测试”，讨论了成熟的应用程序将需要测试和广泛的日志记录来调试和捕获问题，以防它们进入生产环境。

第九章，“安全性”，将专注于 Web 开发的最佳实践，并审查 Go 在这一领域为开发人员提供的内容。

第十章，“缓存、代理和性能改进”，审查了确保没有瓶颈或其他可能对性能产生负面影响的最佳选项。

# 您需要为本书准备的内容

Go 在跨平台兼容性方面表现出色，因此任何运行标准 Linux 版本、OS X 或 Windows 的现代计算机都足以开始。您可以在[`golang.org/dl/`](https://golang.org/dl/)找到完整的要求列表。在本书中，我们使用至少 Go 1.5，但任何更新的版本都应该没问题。

# 本书适合对象

本书适用于 Go 新手开发人员，但具有构建 Web 应用程序和 API 的经验。如果您了解 HTTP 协议、RESTful 架构、通用模板和 HTML，那么您应该已经准备好接手本书中的项目了。

# 约定

在本书中，您会发现一些文本样式，用于区分不同类型的信息。以下是一些样式的示例及其含义解释。

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下：“例如，为了尽快开始，您可以在任何喜欢的地方创建一个简单的`hello.go`文件，并且编译没有问题。”

代码块设置如下：

```go
func Double(n int) int {

  if (n == 0) {
    return 0
  } else {
    return n * 2
  }
}
```

当我们希望引起您对代码块特定部分的注意时，相关行或项目会以粗体显示：

```go
routes := mux.NewRouter()
  routes.HandleFunc("/page/{guid:[0-9a-zA\\-]+}", ServePage)
  routes.HandleFunc("/", RedirIndex)
  routes.HandleFunc("/home", ServeIndex)
  http.Handle("/", routes)
```

任何命令行输入或输出都以以下方式编写：

```go
export PATH=$PATH:/usr/local/go/bin

```

**新术语**和**重要单词**以粗体显示。例如，屏幕上显示的单词，例如菜单或对话框中的单词，会以这样的方式出现在文本中：“第一次点击您的 URL 和端点时，您将看到**我们刚刚设置了值！**，如下面的屏幕截图所示。”

### 注意

警告或重要提示会以这样的方式显示在框中。

### 提示

技巧和窍门会以这样的方式显示。

# 读者反馈

我们非常欢迎读者的反馈。请让我们知道您对本书的看法——您喜欢或不喜欢什么。读者的反馈对我们很重要，因为它可以帮助我们开发您真正能从中受益的标题。

要向我们发送一般反馈，只需发送电子邮件至`<feedback@packtpub.com>`，并在消息主题中提及书名。

如果您在某个专题上有专业知识，并且有兴趣撰写或为书籍做出贡献，请参阅我们的作者指南，网址为[www.packtpub.com/authors](http://www.packtpub.com/authors)。

# 客户支持

现在您是 Packt 书籍的自豪所有者，我们有很多东西可以帮助您充分利用您的购买。

## 下载示例代码

您可以从您在[`www.packtpub.com`](http://www.packtpub.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[`www.packtpub.com/support`](http://www.packtpub.com/support)注册并直接将文件发送到您的电子邮件。

您可以按照以下步骤下载代码文件：

1.  使用您的电子邮件地址和密码登录或注册到我们的网站。

1.  将鼠标指针悬停在顶部的**支持**选项卡上。

1.  点击**代码下载和勘误**。

1.  在**搜索**框中输入书名。

1.  选择您要下载代码文件的书籍。

1.  从下拉菜单中选择您购买本书的地方。

1.  点击**代码下载**。

您也可以通过在 Packt Publishing 网站上的书籍页面上点击**代码文件**按钮来下载代码文件。可以通过在**搜索**框中输入书名来访问该页面。请注意，您需要登录到您的 Packt 帐户。

下载文件后，请确保使用以下最新版本的软件解压缩文件夹：

+   WinRAR / 7-Zip for Windows

+   Zipeg / iZip / UnRarX for Mac

+   7-Zip / PeaZip for Linux

## 下载本书的彩色图像

我们还为您提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。彩色图像将帮助您更好地理解输出中的变化。您可以从[`www.packtpub.com/sites/default/files/downloads/LearningGoWebDevelopment_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/LearningGoWebDevelopment_ColorImages.pdf)下载此文件。

## 勘误

尽管我们已经非常注意确保内容的准确性，但错误确实会发生。如果您在我们的书籍中发现错误——可能是文本或代码中的错误——我们将不胜感激，如果您能向我们报告。通过这样做，您可以帮助其他读者避免挫折，并帮助我们改进本书的后续版本。如果您发现任何勘误，请访问[`www.packtpub.com/submit-errata`](http://www.packtpub.com/submit-errata)，选择您的书籍，点击**勘误提交表**链接，并输入您的勘误详情。一旦您的勘误被验证，您的提交将被接受，并且勘误将被上传到我们的网站或添加到该标题的勘误部分下的任何现有勘误列表中。

要查看先前提交的勘误表，请访问[`www.packtpub.com/books/content/support`](https://www.packtpub.com/books/content/support)，并在搜索框中输入书名。所需信息将出现在**勘误表**部分下。

## 盗版

互联网上盗版受版权保护的材料是跨所有媒体持续存在的问题。在 Packt，我们非常重视版权和许可的保护。如果您在互联网上发现我们作品的任何非法副本，请立即向我们提供位置地址或网站名称，以便我们采取补救措施。

请通过`<copyright@packtpub.com>`与我们联系，并提供涉嫌盗版材料的链接。

我们感谢您帮助保护我们的作者和我们为您提供有价值的内容的能力。

## 问题

如果您对本书的任何方面有问题，可以通过`<questions@packtpub.com>`与我们联系，我们将尽力解决问题。