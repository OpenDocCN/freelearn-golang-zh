# 第二章：开始使用 AWS Lambda

本章提供了在 Go 运行时和开发环境中设置 AWS 环境的指南。您将了解强大的 AWS CLI，它将使部署无服务器应用程序更加高效，并极大地提高您的生产力。

此外，您将获得一组关于如何选择您的 Go **集成开发环境**（**IDE**）的提示和建议。

# 技术要求

在继续安装和配置 AWS 和 Go 环境之前，建议您在笔记本电脑（Windows、Linux 或 macOS X）上跟着本章进行操作，预先安装 Python 2 版本 2.6.5+或 Python 3 版本 3.3+，并设置好 AWS 账户，以便您可以轻松执行给定的命令。本章的代码包托管在 GitHub 上，网址为[`github.com/PacktPublishing/Hands-On-serverless-Applications-with-Go`](https://github.com/PacktPublishing/Hands-On-Serverless-Applications-with-Go)。

# 设置 AWS 环境

本节将指导您如何安装和配置 AWS 命令行。CLI 是一个可靠且必不可少的工具，将在接下来的章节中介绍；它将通过自动化 Lambda 函数的部署和配置为我们节省大量时间。

# AWS 命令行

AWS CLI 是一个强大的工具，可从终端会话中管理您的 AWS 服务和资源。它是建立在 AWS API 之上的，因此通过 AWS 管理控制台可以完成的所有操作都可以通过 CLI 完成；这使它成为一个方便的工具，可以用来通过脚本自动化和控制您的 AWS 基础架构。后面的章节将提供有关使用 CLI 管理 Lambda 函数和创建其他围绕 Lambda 的 AWS 服务的信息。

让我们来看一下 AWS CLI 的安装过程；您可以在*AWS 管理控制台*部分找到有关其配置和测试的信息。

# 安装 AWS CLI

要开始，请打开一个新的终端会话，然后使用`pip` Python 软件包管理器来安装`awscli`的最新稳定版本：

```go
pip install awscli
```

如果您已安装 CLI，则建议出于安全目的升级到最新版本：

```go
pip install --upgrade awscli
```

Windows 用户也可以使用 MSI 安装程序([`s3.amazonaws.com/aws-cli/AWSCLI64.msi`](https://s3.amazonaws.com/aws-cli/AWSCLI64.msi)或[`s3.amazonaws.com/aws-cli/AWSCLI32.msi`](https://s3.amazonaws.com/aws-cli/AWSCLI32.msi)*)*，无需安装 Python。

安装完成后，您需要将 AWS 二进制路径添加到`PATH`环境变量中，方法如下：

+   对于 Windows，按 Windows 键，然后键入环境变量。在环境变量窗口中，突出显示**系统变量**部分中的`PATH`变量。编辑它并通过在最后一个路径后面放置一个分号来添加路径，输入安装 CLI 二进制文件的文件夹的完整路径。

+   对于 Linux、Mac 或任何 Unix 系统，请打开您的 shell 配置文件（`.bash_profile`、`.profile`或`.bash_login`），并将以下行添加到文件的末尾：

```go
export PATH=~/.local/bin:$PATH
```

最后，将配置文件加载到当前会话中：

```go
source ~/.bash_profile
```

通过打开一个新的终端会话并输入以下命令来验证 CLI 是否正确安装：

```go
aws --version
```

您应该能够看到 AWS CLI 的版本；在我的情况下，安装了 1.14.60 版本：

![](img/ec6f5ad5-48fa-455f-8bd9-d3f3ecb385b0.png)

让我们来测试一下，并以法兰克福地区的 Lambda 函数为例进行列出：

```go
aws lambda list-functions --region eu-central-1
```

上一个命令将显示以下输出：

![](img/a9f5051f-6af0-4e26-8073-ac6fc7e97ad1.png)

在使用 CLI 时，通常需要您的 AWS 凭证来对 AWS 服务进行身份验证。有多种方法可以配置 AWS 凭证：

+   **环境凭证**：`AWS_ACCESS_KEY_ID`和`AWS_SECRET_KEY`变量。

+   **共享凭证文件**：`~/.aws/credentials`文件。

+   **IAM 角色**：如果您在 EC2 实例中使用 CLI，则这些角色可以避免在生产中管理凭据文件的需要。

在下一节中，我将向您展示如何使用**AWS 身份和访问管理**（**IAM**）服务为 CLI 创建新用户。

# AWS 管理控制台

IAM 是一个允许您管理用户、组以及他们对 AWS 服务的访问级别的服务。

强烈建议您除了进行结算任务外，不要使用 AWS 根帐户执行任何任务，因为它具有创建和删除 IAM 用户、更改结算、关闭帐户以及在 AWS 帐户上执行所有其他操作的最终权限。因此，我们将创建一个新的 IAM 用户，并根据*最小权限原则*授予其访问正确 AWS 资源所需的权限。在这种情况下，用户将完全访问 AWS Lambda 服务：

1.  使用您的 AWS 电子邮件地址和密码登录 AWS 管理控制台（[`console.aws.amazon.com/console/home`](https://console.aws.amazon.com/console/home)）。

1.  从**安全、身份和合规性**部分打开**IAM**控制台：

![](img/0c79c822-802f-4e49-95c5-338317005bb3.png)

1.  从导航窗格中，选择用户，然后单击“添加用户”按钮，为用户设置名称，并选择编程访问（如果您希望同一用户访问控制台，则还要选择 AWS 管理控制台访问）：

![](img/8cb3470b-5390-4319-bb8e-6f9ed5b00332.png)

1.  在“设置权限”部分，将 AWSLambdaFullAccess 策略分配给用户：

![](img/a04d00d8-8f99-4eb3-ac17-5de986f12b32.png)

1.  在最后一页，您应该看到用户的 AWS 凭据：

![](img/767571a7-8d64-4bec-be8b-048f47b7922a.png)

确保将访问密钥保存在安全位置，因为您将无法再次看到它们：

![](img/d6919348-6e5b-4968-a63d-09a3426d3e09.png)

# 配置

我们的 IAM 用户已创建。让我们使用`aws configure`命令提供访问密钥和秘密密钥以及默认区域：

![](img/8cd29640-c603-455c-803a-90d51132785c.png)

CLI 将在本地文件`~/.aws/credentials`（或在 Windows 上的`%UserProfile%\.aws/credentials`）中存储在前述命令中指定的凭据，内容如下：

```go
[default]
aws_access_key_id = AKIAJYZMNSSSMS4EKH6Q
aws_secret_access_key = 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
region=eu-central-1
```

# 测试

就是这样；尝试以下命令，如果您有任何 Lambda 函数，您应该能够看到它们被列出：

![](img/fc8c86f5-0db3-4547-8da4-cbb7db791d1f.png)

默认输出为 JSON。您可以通过添加`--output`选项（支持的值：*json*、*table*、*text*）来更改命令的输出格式。以下是以表格格式显示的结果：

![](img/07bcdb7f-2e39-4fce-b308-c7bdd5de8aca.png)

此外，您可以使用`--query`选项从此 JSON 文档中提取输出元素。例如，要输出函数名称属性，可以使用以下命令：

```go
aws lambda list-functions --query Functions[].FunctionName
```

输出应该类似于以下内容：

![](img/bacc9024-c0f8-4e4b-a722-0b2c670ee374.png)

可以使用`jq`这样的工具来操作 JSON。它使我们能够针对 CLI 返回的 JSON 进行过滤、映射、计数和执行其他高级 JSON 处理：

```go
aws lambda list-functions | jq '.Functions[].FunctionName'
```

控制台将显示以下输出：

![](img/65fd55d9-0655-478f-a6be-6e856aad12b7.png)

# 设置 Go 环境

本节将指导您如何在多个平台上下载和安装 Go，如何构建一个简单的 Hello World 应用程序，以及如何使用 IDE 加快 Go 开发速度。在此过程中，您将熟悉编写 Go 函数所需的 Go 命令。

# 运行时环境

从 Go 下载页面（[`golang.org/dl/`](https://golang.org/dl/)）下载适合您操作系统和架构的适当软件包：

+   **对于 macOS X：**下载`goVersion.darwin.amd64.pkg`文件，并按照安装提示进行操作。您可能需要重新启动任何打开的终端会话以使更改生效。

+   **对于 Windows**：下载 MSI 安装程序并按照向导进行操作。安装程序将为您设置环境变量。

+   **对于 Linux**：打开一个新的终端会话，并键入以下命令（在撰写本文时，当前版本为 1.10）：

```go
curl https://golang.org/doc/install?download=go1.10.1.linux-amd64.tar.gz -O /tmp/go1.10.tar.gz
tar -C /usr/local -xzf /tmp/go1.10.tar.gz
```

前面的命令将使用`curl`下载最新的 Go 包。然后，它将使用`tar`来解压该包。接下来，通过将以下行添加到您的 shell 的配置脚本中，将`/usr/local/go/bin`添加到`PATH`环境变量中：

```go
export PATH=$PATH:/usr/local/go/bin
```

如果您将 Go 安装在自定义目录中，而不是`/usr/local`，您必须设置`GOROOT`环境变量，指向安装目录：

```go
export GOROOT=PATH/go
export PATH=$PATH:$GOROOT/bin
```

然后您需要重新加载用户配置文件以应用更改：

```go
$ source ~/.bash_profile
```

现在 Go 已经正确安装，并且已经为您的计算机设置了路径，让我们来测试一下。创建一个工作区，我们将在整本书中构建我们的无服务器应用程序：

```go
mkdir -p $HOME/go/src
```

Go 源代码位于工作区中；默认情况下应该是`$HOME/go`。如果您想使用不同的目录，您需要设置`GOPATH`环境变量。

要验证 Go 工作区是否正确配置，您可以运行`go env`命令：

![](img/e73dbc4f-41ce-49e3-acc5-37b55ef37028.png)

如果设置了`GOPATH`变量，您就可以开始了。在工作区内，使用`vim`创建一个`main.go`文件，内容如下：

```go
package main
import "fmt"

func main(){
  fmt.Println("Welcome to 'Hands-On serverless Applications with Go'")
}
```

使用以下命令编译文件：

```go
go run main.go
```

如果成功运行，文件将显示“欢迎来到'使用 Go 进行无服务器应用'”，这表明 Go 正在正确编译文件。

Go 是一种编译语言，因此您可以使用以下命令为应用程序生成单个二进制文件：

```go
go build -o app main.go
```

如果您想为特定的操作系统和架构构建可执行文件，可以覆盖`GOOS`和`GOARCH`参数：

```go
GOOS=linux GOARCH=amd64 go build -o app main.go
```

使用 vim 文本编辑器编辑 Go 并不是最佳选择；因此，在下一节中，我将向您展示如何使用 VSCode 作为 Go 编辑器，以增强您的开发生产力/体验。

# 开发环境

拥有一个 IDE 可以提高您的开发速度，并节省大量时间，这些时间可以用于调试和搜索正确的语法。此外，您可以轻松导航和搜索 Lambda 函数代码。

但我们应该使用哪一个呢？有许多解决方案；这些解决方案可以分为三个主要类别：

+   **IDE**：GoLand，Eclipse，Komodo

+   **编辑器**：Atom，VSCode，Sublime Text

+   **基于云的 IDE**：Cloud9，Codeanywhere，CodeEnvy

Go 生态系统提供了各种编辑器和 IDE；确保您尝试它们，找到最适合您的那个。

我选择了 Visual Studio Code（VS Code），因为它符合我的所有标准：

+   开源

+   支持多种语言

+   插件驱动工具

+   强大的社区和支持

VSCode 对 Go 开发有很好的支持，包括开箱即用的语法高亮显示，内置的 GIT 集成，所有 Go 工具的集成以及 Delve 调试器。

除了对 Go 的本机支持外，开源社区还构建了一些有用和强大的插件，您可以从 VSCode Marketplace 安装：

![](img/fb1dfa15-617b-41cc-abb5-404d208f4fce.png)

VSCode 也是跨平台的，因此您可以在 Mac、Linux 或 Windows 上使用它。使用 Visual Studio Code，您可以通过一系列可用的插件扩展功能，这些插件带来了许多强大和稳健的功能，例如以下内容：

+   **自动完成**：在编写 Go 文件时，您可以看到 IntelliSense 提供了建议的完成：

![](img/0bbd3ee9-32bc-44e2-bda7-87c539cb1e96.png)

+   **签名帮助**：悬停在任何变量、函数或结构上都会给出有关该项的信息，例如文档、签名、预期输入和输出参数。例如，以下屏幕截图显示了有关`Println`的信息，该信息是从悬停在`main.go`文件上获得的：

![](img/ab28ea9a-b336-4d88-b0a9-76e02fa4ffc7.png)

+   **代码格式化**：它会在保存时自动格式化您的 Go 源代码，使用**gofmt**工具，使您的代码更易于编写、阅读和维护。

+   **集成调试器**：您可以设置断点和条件断点，并查看每个帧中的堆栈跟踪和本地和全局变量。

+   **自动导入 Go 包**：它会在保存时自动导入所需的 Go 包。

+   **测试运行器**：它允许您运行、停止和重新启动单元测试以及集成测试。

我期待着 JetBrains 发布的 GoLand 的稳定版本：它看起来是一个非常有前途的 Go IDE，我很期待它的发展。

就是这样！您已经准备好开始在 Go 中构建和部署无服务器应用程序。

# 摘要

在本章中，我们学习了如何安装、配置和使用 AWS CLI。当涉及管理 AWS 服务和自动部署 Lambda 函数时，这个工具将非常有帮助。然后，我们介绍了如何创建用户并从 IAM 生成 AWS 凭据，以获取最少必要的权限。这样，如果您的访问密钥落入错误的手中，造成的危害将是有限的。此外，我们学习了如何设置 Go 环境，逐步在多个平台（Windows、macOS X 和 Linux）上安装 Go，并编译了我们的第一个 Go 中的 Hello World 应用程序。在此过程中，我们介绍了 Go 中最重要的命令，这将帮助您轻松地跟随后面的章节。

在下一章中，我们将终于动手编写我们的第一个 Go 中的 Lambda 函数。

# 问题

1.  AWS CLI 不支持哪种格式？

+   JSON

+   表

+   XML

+   文本

1.  是否建议使用 AWS 根帐户进行日常与 AWS 的交互？如果是，为什么？

1.  您需要设置哪些环境变量才能使用 AWS CLI？

1.  如何使用带有命名配置文件的 AWS CLI？

1.  解释 GOPATH 环境变量。

1.  哪个命令行命令编译 Go 程序？

+   `go build`

+   `go run`

+   `go fmt`

+   `go doc`

1.  什么是 Go 工作区？
