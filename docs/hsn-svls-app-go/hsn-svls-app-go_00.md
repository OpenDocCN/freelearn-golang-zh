# 前言

无服务器架构在技术社区中很受欢迎，其中 AWS Lambda 是很受欢迎的。Go 语言易于学习，易于使用，并且易于其他开发人员阅读，现在它已被誉为 AWS Lambda 支持的语言。本书是您设计无服务器 Go 应用程序并将其部署到 Lambda 的最佳指南。

本书从快速介绍无服务器架构及其优势开始，然后通过实际示例深入探讨 AWS Lambda。然后，您将学习如何在 Go 中使用 AWS 无服务器服务设计和构建一个可投入生产的应用程序，而无需预先投资基础设施。本书将帮助您学习如何扩展无服务器应用程序并在生产中处理分布式无服务器系统。然后，您还将学习如何记录和测试您的应用程序。

在学习的过程中，您还将发现如何设置 CI/CD 管道以自动化 Lambda 函数的部署过程。此外，您将学习如何使用 AWS CloudWatch 和 X-Ray 等服务实时监视和排除故障您的应用程序。本书还将教您如何扩展无服务器应用程序并使用 AWS Cognito 安全访问。

通过本书，您将掌握设计、构建和部署基于 Go 的 Lambda 应用程序到生产的技能。

# 这本书适合谁

这本书适合希望了解无服务器架构的 Gophers。假定具有 Go 编程知识。对于有兴趣在 Go 中构建无服务器应用程序的 DevOps 和解决方案架构师也会从本书中受益。

# 本书涵盖了什么

第一章《Go 无服务器》给出了关于无服务器是什么，它是如何工作的，它的特点是什么，为什么 AWS Lambda 是无服务器计算服务的先驱，以及为什么您应该使用 Go 构建无服务器应用程序的基础解释。

第二章《开始使用 AWS Lambda》提供了在 Go 运行时和开发环境旁边设置 AWS 环境的指南。

第三章《使用 Lambda 开发无服务器函数》描述了如何从头开始编写您的第一个基于 Go 的 Lambda 函数，以及如何从控制台手动调用它。

第四章《使用 API Gateway 设置 API 端点》说明了如何使用 API Gateway 在收到 HTTP 请求时触发 Lambda 函数，并构建一个由无服务器函数支持的统一事件驱动的 RESTful API。

第五章《使用 DynamoDB 管理数据持久性》展示了如何通过使用 DynamoDB 数据存储解决 Lambda 函数无状态问题来管理数据。

第六章《部署您的无服务器应用程序》介绍了在构建 AWS Lambda 中的无服务器函数时可以使用的高级 AWS CLI 命令和选项，以节省时间。它还展示了如何创建和维护 Lambda 函数的多个版本和发布。

第七章《实施 CI/CD 管道》展示了如何设置持续集成和持续部署管道，以自动化 Lambda 函数的部署过程。

第八章《扩展您的应用程序》介绍了自动缩放的工作原理，Lambda 如何在高峰服务使用期间处理流量需求而无需容量规划或定期缩放，以及如何使用并发预留来限制执行次数。

第九章《使用 S3 构建前端》说明了如何使用由无服务器函数支持的 REST 后端构建单页面应用程序。

第十章，*测试您的无服务器应用程序*，展示了如何使用 AWS 无服务器应用程序模型在本地测试无服务器应用程序。它还涵盖了 Go 单元测试和使用第三方工具进行性能测试，并展示了如何使用 Lambda 执行测试工具。

第十一章，*监控和故障排除*，进一步介绍了如何使用 CloudWatch 设置函数级别的监控，以及如何使用 AWS X-Ray 调试和故障排除 Lambda 函数，以便对应用程序进行异常行为检测。

第十二章，*保护您的无服务器应用程序*，致力于在 AWS Lambda 中遵循最佳实践和建议，使您的应用程序符合 AWS Well-Architected Framework 的要求，从而使其具有弹性和安全性。

第十三章，*设计成本效应应用程序*，还介绍了一些优化和减少无服务器应用程序计费的技巧，以及如何使用实时警报跟踪 Lambda 的成本和使用情况，以避免问题的出现。

第十四章，*基础设施即代码*，介绍了诸如 Terraform 和 SAM 之类的工具，帮助您以自动化方式设计和部署 N-Tier 无服务器应用程序，以避免人为错误和可重复的任务。

# 要充分利用本书

本书适用于在 Linux、Mac OS X 或 Windows 下工作的任何人。您需要安装 Go 并拥有 AWS 账户。您还需要 Git 来克隆本书提供的源代码库。同样，您需要具备 Go、Bash 命令行和一些 Web 编程技能的基础知识。所有先决条件都在第二章中进行了描述，*开始使用 AWS Lambda*，并提供了确保您能轻松跟随本书的说明。

最后，请记住，本书并不意味着取代在线资源，而是旨在补充它们。因此，您显然需要互联网访问来完成阅读体验的某些部分，通过提供的链接。

# 下载示例代码文件

您可以从[www.packtpub.com](http://www.packtpub.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packtpub.com/support](http://www.packtpub.com/support)注册，直接将文件发送到您的邮箱。

您可以按照以下步骤下载代码文件：

1.  登录或注册[www.packtpub.com](http://www.packtpub.com/support)。

1.  选择“支持”选项卡。

1.  点击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明操作。

下载文件后，请确保使用最新版本的解压缩软件解压缩文件夹：

+   Windows 的 WinRAR/7-Zip

+   Mac 的 Zipeg/iZip/UnRarX

+   Linux 的 7-Zip/PeaZip

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Hands-On-Serverless-Applications-with-Go`](https://github.com/PacktPublishing/Hands-On-Serverless-Applications-with-Go)。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还有来自丰富书籍和视频目录的其他代码包，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。去看看吧！

# 下载彩色图片

我们还提供了一个 PDF 文件，其中包含了本书中使用的屏幕截图/图表的彩色图片。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/HandsOnServerlessApplicationswithGo_ColorImages.pdf`](http://www.packtpub.com/sites/default/files/downloads/HandsOnServerlessApplicationswithGo_ColorImages.pdf)。

# 使用的约定

在本书中，您会发现一些文本样式，用于区分不同类型的信息。以下是一些样式的示例及其含义的解释。

文本中的代码单词显示如下：“在工作空间中，使用`vim`创建一个`main.go`文件，内容如下。”

代码块设置如下：

```go
package main
import "fmt"

func main(){
  fmt.Println("Welcome to 'Hands-On serverless Applications with Go'")
}
```

任何命令行输入或输出都是这样写的：

```go
pip install awscli
```

**粗体**：表示一个新术语，一个重要词，或者您在屏幕上看到的词。例如，菜单或对话框中的单词会在文本中显示为这样。这里有一个例子：“在`Source`页面上，选择`GitHub`作为源提供者。”

警告或重要说明看起来像这样。

提示和技巧看起来像这样。

# 联系我们

我们的读者的反馈总是受欢迎的。

**一般反馈**：发送电子邮件至`feedback@packtpub.com`，并在消息主题中提及书名。如果您对本书的任何方面有疑问，请发送电子邮件至`questions@packtpub.com`与我们联系。

**勘误**：尽管我们已经尽一切努力确保内容的准确性，但错误是难免的。如果您在本书中发现了错误，我们将不胜感激地接受您的报告。请访问[www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择您的书，点击勘误提交表格链接，并输入详细信息。

**盗版**：如果您在互联网上发现我们作品的任何形式的非法副本，我们将不胜感激地接受您提供的位置地址或网站名称。请通过`copyright@packtpub.com`与我们联系，并提供材料的链接。

**如果您有兴趣成为作者**：如果您在某个专业领域有专长，并且有兴趣撰写或为一本书做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。阅读并使用本书后，为什么不在购买它的网站上留下评论呢？潜在的读者可以看到并使用您的客观意见来做出购买决策，我们在 Packt 可以了解您对我们产品的看法，我们的作者可以看到您对他们书籍的反馈。谢谢！

有关 Packt 的更多信息，请访问[packtpub.com](https://www.packtpub.com/)。
