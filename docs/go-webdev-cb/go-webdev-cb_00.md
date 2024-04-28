# 前言

Go 是一种设计用于扩展并支持语言级并发性的开源编程语言，这使得开发人员可以轻松编写大型并发的 Web 应用程序。

从创建 Web 应用程序到在 AWS 上部署，这将是一个学习 Go Web 开发的一站式指南。无论您是新手程序员还是专业开发人员，本书都将帮助您快速掌握 Go Web 开发。

本书将专注于在 Go 中编写模块化代码，并包含深入的信息性配方，逐步构建基础。您将学习如何创建服务器、处理 HTML 表单、会话和错误处理、SQL 和 NoSQL 数据库、Beego、创建和保护 RESTful Web 服务、创建、单元测试和调试 WebSockets，以及创建 Go Docker 容器并在 AWS 上部署它们等概念和配方。

通过本书，您将能够将您在 Go 中学到的新技能应用于在任何领域创建和探索 Web 应用程序。

# 本书适合人群

本书适用于希望使用 Go 编写大型并发 Web 应用程序的开发人员。对 Go 有一定了解的读者会发现本书最有益。

# 本书内容

第一章《在 Go 中创建您的第一个服务器》解释了如何编写和与 HTTP 和 TCP 服务器交互，使用 GZIP 压缩优化服务器响应，并在 Go Web 应用程序中实现路由和日志记录。

第二章《处理模板、静态文件和 HTML 表单》介绍了如何创建 HTML 模板；从文件系统中提供静态资源；创建、读取和验证 HTML 表单；以及为 Go Web 应用程序实现简单的用户身份验证。

第三章《在 Go 中处理会话、错误和缓存》探讨了实现 HTTP 会话、HTTP cookie、错误处理和缓存，以及使用 Redis 管理 HTTP 会话，这对于在多个数据中心部署的 Web 应用程序是必需的。

第四章《在 Go 中编写和消费 RESTful Web 服务》解释了如何编写 RESTful Web 服务、对其进行版本控制，并创建 AngularJS 与 TypeScript 2、ReactJS 和 VueJS 客户端来消费它们。

第五章《使用 SQL 和 NoSQL 数据库》介绍了在 Go Web 应用程序中使用 MySQL 和 MongoDB 数据库实现 CRUD 操作。

第六章《使用微服务工具包 Go 编写微服务》专注于使用协议缓冲区编写和处理微服务，使用微服务发现客户端（如 Consul），使用 Go Micro 编写微服务，并通过命令行和 Web 仪表板与它们进行交互，以及实现 API 网关模式以通过 HTTP 协议访问微服务。

第七章《在 Go 中使用 WebSocket》介绍了如何编写 WebSocket 服务器及其客户端，以及如何使用 GoLand IDE 编写单元测试并进行调试。

第八章《使用 Go Web 应用程序框架-Beego》介绍了设置 Beego 项目架构，编写控制器、视图和过滤器，实现与 Redis 支持的缓存，以及使用 Nginx 监控和部署 Beego 应用程序。

第九章《使用 Go 和 Docker》介绍了如何编写 Docker 镜像、创建 Docker 容器、用户定义的 Docker 网络、使用 Docker Registry，并运行与另一个 Docker 容器链接的 Go Web 应用程序 Docker 容器。

第十章，*保护 Go Web 应用程序*，演示了使用 OpenSSL 创建服务器证书和私钥，将 HTTP 服务器转移到 HTTPS，使用 JSON Web Token（JWT）保护 RESTful API，并防止 Go Web 应用程序中的跨站点请求伪造。

第十一章，*将 Go Web 应用程序和 Docker 容器部署到 AWS*，讨论了设置 EC2 实例，交互以及在其上运行 Go Web 应用程序和 Go Docker 容器。

# 充分利用本书

读者应具备 Go 的基本知识，并在计算机上安装 Go 以执行说明和代码。

# 下载示例代码文件

您可以从[www.packtpub.com](http://www.packtpub.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，可以访问[www.packtpub.com/support](http://www.packtpub.com/support)并注册，以便将文件直接发送到您的邮箱。

您可以按照以下步骤下载代码文件：

1.  登录或注册[www.packtpub.com](http://www.packtpub.com/support)。

1.  选择“支持”选项卡。

1.  单击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明操作。

下载文件后，请确保使用最新版本的以下软件解压缩或提取文件夹：

+   WinRAR/7-Zip for Windows

+   Zipeg/iZip/UnRarX for Mac

+   7-Zip/PeaZip for Linux

本书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Go-Web-Development-Cookbook`](https://github.com/PacktPublishing/Go-Web-Development-Cookbook)。我们还有来自丰富书籍和视频目录的其他代码包，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。快去看看吧！

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/GoWebDevelopmentCookbook_ColorImages.pdf`](http://www.packtpub.com/sites/default/files/downloads/GoWebDevelopmentCookbook_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。例如：“GZIP 压缩意味着从服务器以`.gzip`格式向客户端发送响应，而不是发送纯文本响应。”

代码块设置如下：

```go
for 
{
  conn, err := listener.Accept()
  if err != nil 
  {
    log.Fatal("Error accepting: ", err.Error())
  }
  log.Println(conn)
}
```

任何命令行输入或输出都以以下方式编写：

```go
$ go get github.com/gorilla/handlers
$ go get github.com/gorilla/mux
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会在文本中出现。例如：“AngularJS 客户端页面具有 HTML 表单，其中显示如下的 Id、FirstName 和 LastName 字段。”

警告或重要说明看起来像这样。

提示和技巧看起来像这样。

# 章节

在本书中，您会经常看到几个标题（*准备工作*，*如何做*，*工作原理*，*更多内容*和*另请参阅*）。

为了清晰地说明如何完成配方，使用以下各节：

# 准备工作

本节告诉您该配方中可以期望的内容，并描述了为该配方设置任何软件或任何先决设置所需的步骤。

# 如何做…

本节包含遵循该配方所需的步骤。

# 工作原理…

本节通常包括对前一节发生的事情的详细解释。

# 更多内容…

本节包含有关该配方的其他信息，以使您对该配方更加了解。

# 另请参阅

本节提供了有关该配方的其他有用信息的链接。
