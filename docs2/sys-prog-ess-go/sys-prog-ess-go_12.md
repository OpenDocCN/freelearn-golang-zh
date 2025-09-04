# 12

# 分发你的应用程序

在本章中，我们将探讨使用模块、**持续集成**（**CI**）和发布策略分发 Go 应用程序的关键概念和实际应用。随着我们的进展，你将熟练使用 Go modules 进行依赖项管理，设置 CI 工作流程来自动化测试和构建，并掌握发布过程以无缝分发你的应用程序。

本章将涵盖以下关键主题：

+   Go Modules

+   CI

+   发布你的应用程序

在本章结束时，你将掌握精确管理依赖项、自动化测试和构建过程以尽早捕捉错误、以及高效打包和发布应用程序的知识。这些技能为维护易于管理、更新和随着团队规模扩展而无需增加额外官僚层的健壮软件项目提供了基础。

# 技术要求

本章中展示的所有代码都可以在我们的`git`仓库的`ch12`目录中找到。

# Go Modules

Go Modules，在依赖项混乱的海洋中的灯塔。好吧——处理 Go 中的依赖项可能并不像在公园里散步的星期天早晨那么简单。但把它看作是一次前往火星的任务——复杂，是的，但有了正确的工具（模块），潜在的回报是巨大的。

在 Go 1.11 中引入，Go Modules 从根本上重塑了 Golang 中包管理的格局，特别是在系统编程领域尤为重要。此功能提供了一套强大的系统来管理项目依赖项，封装了项目所依赖的外部包的特定版本。在核心上，Go Modules 通过利用模块缓存和定义好的依赖项集，实现了可重复构建，从而消除了臭名昭著的“在我的机器上工作”综合征。

Go Modules 解决了几个类别的问题，使得使用它的体验异常稳健。我可以强调其中三个：可靠的版本控制、可重复构建和管理依赖项冗余。让我在 Go Modules 引入之前和之后解释每个问题的主要变化。

让我们先看看可靠的版本控制：

+   **Before**: 在 Go 项目中指定依赖项的方式留下了解释的空间。可能并不完全清楚你请求的是哪个版本的包。由于这种歧义，当你添加依赖项时，你无法完全确定项目中将包含哪些代码。存在意外版本或甚至不同包被拉入的可能性。

+   **After**: 介绍了**语义版本控制**（**SemVer**）的概念，确保你知道依赖项更新包含的变化类型，减少不可预测的破坏。

现在，让我们转向可重复构建：

+   **Before**: 依赖于外部包仓库意味着如果依赖项发生变化或消失，构建可能会在以后失败。

+   **之后**：引入 Go 模块代理和供应商（在项目中存储依赖项副本）的能力，确保你的代码始终以相同的方式构建

最后，让我们看看如何管理依赖项膨胀：

+   **之前**：嵌套依赖项很容易失控，增加大小和复杂性

+   **之后**：Go 模块计算所需依赖项的最小集合，使你的项目保持精简

模块是一组相关的 Go 包。它作为“可版本化”和可交换的源代码单元。

模块有两个主要目标：维护依赖项的特定要求，并创建可重复构建。

让我们先想象你正在组织一个图书馆。这个图书馆将作为我们理解模块、版本控制和 SemVer 的类比。将模块想象成一个令人兴奋的书系。每个系列是一系列相关书籍的集合，分卷发布。就像《哈利·波特》系列一样，每本书都对更大的叙事做出贡献，创造出一个令人兴奋且连贯的故事。现在，想象这个系列中的每本书都列出了所有之前的卷，并指定了理解当前书籍所需的精确版本。这确保了无论你在哪里开始这个系列，你都能有一个一致的经历，就像模块通过记录精确的依赖项要求来确保一致的构建一样。将版本控制仓库想象成图书馆里一个组织良好的书架。每个书架包含一个完整的书系，整齐地排列，方便读者找到并跟随系列，不会感到困惑。

但它们之间是如何相互关联的呢？简单来说：**仓库**就像图书馆中专门为特定系列或集合设立的章节。每个**模块**代表这个章节中的一个书系。每个书系（模块）由单个书籍（包）组成。最后，每本书（包）包含章节（Go 源文件），所有这些都包含在书的封面（目录）中。

如果使用 `git`，版本将与仓库的标签相关联。

SemVer

SemVer (`Major.Minor.Patch`) 使用一个编号系统来指示软件更新中包含的更改类型（破坏性更改、新功能、错误修复）。完整的 SemVer 规范可以在 [`semver.org/`](https://semver.org/) 找到。

## 使用模块的常规操作

阳光明媚的日子的工作流程如下：

1.  根据需要将导入添加到 `.go` 文件中。

1.  `go build` 或 `go test` 命令将自动添加新的依赖项以满足导入（自动更新 `go.mod` 文件并下载新的依赖项）。

有时会需要选择依赖项的特定版本。在这种情况下，应使用 `go get` 命令。

`go get` 命令的格式是 `<module-name>@<version>`，如下面的示例所示：

```go
go get foo@v1.2.3
```

直接更改模块文件

如果需要，也可以直接更改`go.mod`文件。在任何情况下，都建议让`Go`命令更改文件。

让我们首先通过创建我们的第一个模块来探索如何使用它们。

### 创建新模块

让我们先为我们的项目创建一个新的 Go 模块。

第一步是设置我们的工作区：

```go
mkdir mybestappever
cd mybestappever
```

我们可以通过运行一个简单的命令来初始化我们的模块：

```go
go mod init github.com/yourusername/mybestappever.
```

此命令创建一个`go.mod`文件，该文件将跟踪您的模块依赖项。

假设我们有一个包含以下内容的新的`main.go`文件：

```go
package main
import (
    "context"
    "fmt"
    "time"
    "github.com/alexrios/timer/v2"
)
func main() {
sw := &timer.Stopwatch{}
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
if err := sw.Start(ctx); err != nil {
    fmt.Printf("Failed to start stopwatch: %v\n", err)
    return
}
time.Sleep(1 * time.Second)
sw.Stop()
elapsed, err := sw.Elapsed()
if err != nil {
    fmt.Printf("Failed to get elapsed time: %v\n", err)
    return
}
fmt.Printf("Elapsed time: %v\n", elapsed)
}
```

此外，假设我们有一个包含以下内容的`main_test.go`文件：

```go
package main
import (
    "context"
    "testing"
    "time"
    "github.com/alexrios/timer/v2"
)
func TestStopwatch(t *testing.T) {
    sw := &timer.Stopwatch{}
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    if err := sw.Start(ctx); err != nil {
        t.Fatalf("Failed to start stopwatch: %v", err)
    }
    time.Sleep(1 * time.Second)
    sw.Stop()
    elapsed, err := sw.Elapsed()
    if err != nil {
        t.Fatalf("Failed to get elapsed time: %v", err)
    }
    if elapsed < 1*time.Second || elapsed > 2*time.Second {
        t.Errorf("Expected elapsed time around 1 second, got %v", elapsed)
    }
}
```

使用`go test`命令执行测试。当您运行它时，Go 工具会自动解决任何新的依赖项，更新`go.mod`文件，并下载必要的模块。

如果您检查`go.mod`文件，您应该看到一个新行用于依赖项：

```go
require github.com/alexrios/timer/v2 v2.0.0
```

### 理解模块版本控制

Go 使用`go.mod`文件。

当您构建或测试 Go 模块时，MVS 根据您的模块及其依赖项的`go.mod`文件中指定的版本要求确定要使用的模块版本集。以下是 MVS 解决这些版本的方式：

+   `go.mod`文件，它指定了直接依赖项的版本。

+   收集直接依赖项及其所需版本的`go.mod`文件。此过程递归地对所有依赖项进行。

+   提及它的`go.mod`文件。所需最高版本被认为是满足所有版本要求的最小版本。

+   **最小化版本**：算法确保选择每个模块的最小版本，这意味着不会选择比必要的更高版本。这减少了引入意外更改或不兼容性的风险。

通过始终选择满足所有要求的最小版本，MVS 避免了不必要的升级，并减少了从依赖项更新中引入破坏性更改的风险。

要列出当前模块及其所有依赖项，您可以使用`go` `list`命令：

```go
go list -m all
```

输出将包括主模块及其依赖项：

```go
github.com/yourusername/mybestappever
github.com/alexrios/timer/v2 v2.0.0
```

嘿！文件夹中有一个新文件：`go.sum`。

`go.sum`文件包含特定模块版本的校验和。这确保了在构建过程中始终一致地使用相同的模块版本。以下是一个您可能在`go.sum`文件中看到的示例：

```go
github.com/alexrios/timer/v2 v2.0.0 h1:...
github.com/alexrios/timer/v2 v2.0.0/go.mod h1:...
```

### 更新依赖项

要更新依赖到最新版本，我们可以使用`go` `get`命令：

```go
go get github.com/alexrios/timer@latest
```

要更新到特定版本，请明确指定版本：

```go
go get github.com/yourusername/timer@v1.1.0
```

有时，您需要指定依赖项的确切版本。您可以直接在`go.mod`文件中这样做，或者使用之前显示的`go get`命令。这对于确保兼容性或需要特定版本的功能或错误修复非常有用。

### 语义导入版本控制

Go 模块遵循 SemVer，它使用版本号来传达关于发布稳定性和兼容性的信息。版本方案是 `v<MAJOR>.<MINOR>.<PATCH>`：

+   主版本表示不兼容的 API 更改

+   小版本以向后兼容的方式添加功能

+   补丁版本包括向后兼容的错误修复

例如，`v1.2.3` 表示主版本 1，次版本 2，补丁版本 3。

当一个模块达到版本 2 或更高时，必须在模块路径中包含主版本号。例如，`github.com/alexrios/timer` 的版本 2 被标识为 `github.com/alexrios/timer/v2`。

您可以使用以下命令随时触发依赖项验证：

```go
go mod tidy
```

Go 中的 `go mod tidy` 命令对于维护干净和准确的 `go.mod` 和 `go.sum` 文件至关重要。它扫描您的项目源代码以确定哪些依赖项被使用，添加缺失的依赖项，并从 `go.mod` 文件中删除未使用的依赖项。此外，它更新 `go.sum` 文件以确保所有依赖项的一致校验和。这个过程有助于使您的项目免于不必要的依赖项，降低安全风险，确保可重复构建，并使依赖项管理更加容易管理。定期使用 `go mod tidy` 确保您的 Go 项目依赖项是最新的，并且准确反映了代码库的要求。

`github.com/alexrios/timer` 库是公开的，但在您的公司中，您可能使用的是闭源库，通常称为私有库。让我们看看如何使用它们。

### 使用私有库

当与托管在私有仓库中的 Go 模块一起工作时，您需要一个确保安全且直接访问的设置，绕过公共 Go 模块代理。本节将指导您配置 Git 和 Go 环境，以便在 GitHub 上无缝使用私有 Go 模块。

首先，您需要导航到您的家目录并打开 `.gitconfig` 文件，添加以下配置：

```go
[url "ssh://git@github.com/"]
    insteadOf = https://github.com/
```

这些行告诉 Git 在遇到以 `https://github.com/` 开头的 GitHub URL 时自动使用 SSH (`ssh://git@github.com/`)。

一旦完成，我们现在可以为私有模块配置 Go 环境。

`GOPRIVATE` 环境变量阻止 Go 工具尝试从公共 Go 模块镜像或代理中获取列出的模块。相反，它直接从它们的源中获取，这对于私有模块是必要的。

您可以通过在终端中运行以下命令来为单个私有仓库设置 `GOPRIVATE`，用 `<org>` 和 `<project>` 替换您的 GitHub 组织和仓库名称：

```go
go env -w GOPRIVATE="github.com/<org>/<project>"
```

或者，为组织中的所有仓库设置 `GOPRIVATE`。如果您在同一个组织下工作，有多个私有仓库，使用通配符 (`*`) 来覆盖所有这些仓库会非常方便：

```go
go env -w GOPRIVATE="github.com/<org>/*"
```

这里有一些额外的提示：

+   使用 `go env GOPRIVATE` 来显示当前设置。

+   **使用 SSH 密钥与 GitHub**：确保您的 SSH 密钥已设置并添加到您的 GitHub 账户。这种设置允许您在不每次都输入用户名和密码的情况下推送和拉取您的仓库。

+   `ssh-add ~/.ssh/your-ssh-key`（将`your-ssh-key`替换为您的 SSH 密钥路径）。

您已配置环境以安全地与包含在私有 GitHub 仓库中的 Go 模块一起工作！这种设置通过自动化 Git 操作的认证来简化开发工作流程，并确保直接、安全地访问您的私有 Go 模块。

### 版本控制和 go install

最基本的方法是将您的 Go 程序代码托管在公共版本控制仓库中（如 GitHub、GitLab 等）。安装了 Go 的用户可以使用`go install`命令后跟您的仓库 URL 来获取、编译和安装您的程序。

在我们的模块根目录中，我们可以简单地执行以下命令：

```go
go install
```

这使得程序可以通过在终端中输入其名称从系统上的任何目录访问。要测试安装是否成功，请打开一个**新**的终端窗口并输入以下内容：

```go
hello
```

使用`go install`，您的编译程序将存储在`$GOPATH/bin`。

自 1.16 版本以来，`go install`命令可以从特定版本安装 Go 可执行文件。

为了使其更容易理解，让我们使用[`github.com/alexrios/endpoints`](https://github.com/alexrios/endpoints)仓库作为我们的目标。

此仓库有一个`v0.5.0`标签，因此要安装此特定版本，我们可以运行以下命令：

```go
go install github.com/alexrios/endpoints@v0.5.0
```

当您不知道或不想发现可执行文件的最新版本时，您可以直接使用`latest`：

```go
go install github.com/alexrios/endpoints@latest
```

注意，在 Go 生态系统内，您可以在不使用任何特殊工具或过程的情况下安装其他可执行文件。非常强大，不是吗？

但对于具有多个模块的项目怎么办？我们如何轻松地处理它们？快速回答：在 Go 1.18 版本之前，这根本不可能。

而不是介绍所有关于 Go 早期时代的民间传说和噩梦，让我们“回到未来”，并关注我们如何轻松地做到这一点。1.18 版本引入了模块工作空间的概念。

### 模块工作空间

Go 模块工作空间是一种将属于同一项目的多个 Go 模块分组的方法。这个特性是为了解决依赖关系管理的难题而引入的，允许开发者同时处理多个模块。这不仅仅是关于整洁。它从根本上改变了 Go 工具链解决依赖关系的方式。

Go 工作空间是一个包含唯一`go.work`文件的目录，该文件引用一个或多个`go.mod`文件，每个文件代表一个模块。这种设置使我们能够在不出现版本冲突的常规头痛中构建、测试和管理多个相互关联的模块。

在工作区内部，Go 编译器将它们视为同等模块，而不是依赖于每个模块的外部 `go.mod` 文件。它查看工作区的 `go.work` 文件，该文件列出了项目中的所有模块，确保所有人都能良好地协同工作。

换句话说，工作区为你的项目创建了一个自包含的生态系统。你在一个模块内所做的任何更改都会立即影响到其他模块。这简化了开发，尤其是在处理大型应用程序相互关联的组件时。

足够的讨论；让我们看看实际操作。考虑以下简单的工作区设置：

```go
go work init ./myproject
go work use ./moduleA
go work use ./moduleB
```

在此工作区内部，`go.work` 文件充当协调者。它确保当你运行 Go 命令时，所有引用的模块都被视为单个统一代码库的一部分。这在开发相互依赖的模块或当你想在提交到上游之前测试模块之间的本地更改时尤其有用。

在这种配置下，`moduleA` 和 `moduleB` 都是同一工作区的一部分，允许无缝的集成测试和开发。

实际影响深远。假设你有两个模块：`moduleA` 和 `moduleB`。`moduleA` 依赖于 `moduleB`。传统上，更新 `moduleB` 可能会变成版本锁定和向后兼容的噩梦。然而，使用工作区，你可以同时修改这两个模块并实时测试集成。

这里有一个简单的 `go.work` 文件示例：

```go
go 1.21
use (
    ./path/to/module-a
    ./path/to/module-b
)
```

模块工作区的最后一段冒险是要同步工作区内的修改。

每次我们更改模块的 `go.mod` 文件或从我们的工作区添加或删除模块时，我们都应该运行以下命令：

```go
go work sync
```

此命令将使我们远离以下问题：

+   `go.mod` 文件将与你的 `go.work` 文件不同步。这种差异可能导致混淆和潜在的错误。

+   `go build` 或 `go test`，根据 Go 如何解析依赖项，存在几种场景：

    +   `go.work` 文件已经本地缓存，Go 可能会使用缓存的版本而不是你在修改的 `go.mod` 中指定的版本。

    +   **构建可能会失败**：如果你的更改引入了不兼容的依赖项版本或所需的版本不可用，你的构建和测试可能会失败。

    +   **协作问题**：如果你与其他人一起在项目上工作，不一致的依赖项声明可能导致在不同机器上构建重现的问题。

如果你忘记了此命令，可能会导致调试意外构建失败所浪费的时间，这些失败是由不匹配的依赖项引起的。如果你忘记手动修改了 `go.mod` 文件，可能很难追踪问题的根本原因。从团队协作的角度来看，当不同开发者的环境具有不同的依赖项状态时，项目维护变得更加困难。

这种模块管理方法不仅仅是为了保持你的理智——它是在培养一个环境，在那里物流噩梦不会阻碍创新。所以，下次当你发现自己正在玩转 Go 模块时，记住，工作空间是你的朋友，而不仅仅是项目复杂性的另一层。

虽然我们的模块知识已经准备好接受测试，但我们希望确保一切运行顺畅，并且（希望如此）没有错误。这就是 CI 发挥作用的时候。

# CI

CI 就像是你的代码保姆。哈！如果你相信这一点，那你可能从未尝试过在同时与基础设施作斗争，后者比撒哈拉沙漠中的冰棍融化得还要快的同时，将一群未驯服的微服务整理成有序的状态。

让我们直面现实——CI 更像是在驯服多头海德拉：一个头喷出单元测试，另一个头吐出集成测试，而在混乱的某个地方，可能潜伏着构建管道和部署。

那么，CI 究竟是什么，就像时髦的年轻人所说的那样？在 Go 的世界里，尤其是在系统编程领域，CI 是将代码更改不断合并到共享仓库中，并对结果进行无情测试的艺术。这是关于尽早捕捉错误，确保新代码不会破坏整个系统，并自动化那些繁重的任务，否则我们都会对着键盘哭泣。

将 CI 视为你的自动化代码质量控制。这是你放置构建脚本和测试套件的地方，也许为了保险起见，还会加入一些静态分析和代码检查，然后将所有这些连接起来，以便每次有人提交更改时都运行。为什么你会问？嗯，因为没有什么能像反复破坏代码库并迫使你的队友修复它那样锻炼一个人的性格。

让我们实际一点。以下是一个使用 GitHub Actions 为你的 Go 项目设置基本 CI 的片段：

```go
name: Go CI on Commit
on:
  push:
jobs:
  test-and-dependencies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Go
        uses: actions/setup-go@v3
        with:
          go-version: '¹.21'
      - name: Get dependencies
        run: go mod download
      - name: Run tests
        run: go test -v ./...
```

## 缓存

利用缓存机制的区别在于，你可以看到你的构建过程像一辆生锈的蒸汽机车一样缓慢地前进，也可以见证高速列车般的效率。让我们看看我们如何能让那些依赖项下载成为过去式。

想象一下你的 CI 管道就像不知疲倦的工厂工人，而那些依赖项就是它需要来保持生产顺畅的原材料。每次构建过程启动时，它都必须重新获取所有依赖项，这会减慢速度，并在过程中制造噪音。缓存就像在你的工厂旁边建一个库存充足的仓库——下次你需要那些材料时，无需进行探险，只需快速去仓库一趟即可。

在 CI 中缓存 Go 依赖的关键在于理解两件事：

+   `~/go/pkg/mod`. 这是我们的大仓库。

+   **如何存储 CI 工作流程中的内容**：大多数 CI 系统，如 GitHub Actions，都有内置机制在工作流程运行之间缓存文件或目录。

让我们将这些概念结合起来。以下是您如何修改基本的 GitHub Actions 工作流程以缓存 Go 依赖项：

```go
name: Go CI on Commit
on:
  push:
jobs:
  test-and-dependencies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Go
        uses: actions/setup-go@v3
        with:
          go-version: '¹.21'
      - name: Cache Go modules
        uses: actions/cache@v3
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-
      - name: Get dependencies
        run: go mod download
      - name: Run tests
        run: go test -v ./...
```

魔法发生在“缓存 Go 依赖项”步骤中：

+   `actions/cache@v3`：这是可靠的 GitHub Actions 缓存工具。

+   `path`：我们告诉它缓存我们的 Go 模块目录。

+   `key`：这唯一地标识了您的缓存。请注意，它包括您的 `go.sum` 文件的哈希值；如果依赖项发生变化，将创建一个新的缓存。

+   `restore-keys`：在找不到确切密钥时提供备用方案。

将缓存想象成这样：您的 CI 管道每次运行都会留下一些面包屑。下次运行时，它会检查这些面包屑，如果找到，就会抓取预先打包的依赖项，而不是去下载新的。

## 静态分析

静态分析工具充当自动代码审查小组，不知疲倦地检查您的 Go 代码中可能存在的陷阱、偏离最佳实践的情况，甚至可能影响您的 Go 代码质量的细微风格不一致。这就像有一支细致的程序员团队一直在您身后审视，但没有那种尴尬的代码压迫感。

让我们将 Staticcheck ([`staticcheck.dev/`](https://staticcheck.dev/))，这位警惕的代码检查员，集成到您的 Go CI 工作流程中，以帮助您保持代码的纯净质量。

Staticcheck 超越了基本的代码检查，深入到识别潜在的 bug、低效的代码模式，甚至可能影响您的 Go 代码质量的细微风格问题。它是您的自动化代码侦探，不知疲倦地寻找可能被粗略检查遗漏的问题。

让我们利用 GitHub Actions 和 `dominikh/staticcheck-action` 动作来简化我们的工作流程集成。

虽然您可以直接在工作流程中安装和执行 Staticcheck，但使用预构建的 GitHub 动作提供了一些优势：

+   **简化设置**：该动作为您处理安装和执行细节，减少了工作流程配置的复杂性

+   **潜在缓存**：某些操作可能会自动缓存 Staticcheck 的结果以加快未来的运行速度

+   **社区驱动**：许多操作正在积极维护，确保与最新的 Go 和 Staticcheck 版本兼容

使用此动作，您的修改后工作流程将如下所示：

```go
name: Go CI
on: [push]
jobs:
  build-test-staticcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: dominikh/staticcheck-action@v1.2.0
        with:
          # Optionally specify Staticcheck version:
          # version: latest
```

关键点如下：

+   `dominikh/staticcheck-action`) 并可选地自定义其版本

+   `v1.2.0`) 确保在 CI 运行中保持一致的行为

staticcheck-action 配置

咨询动作的文档 ([`github.com/dominikh/staticcheck-action`](https://github.com/dominikh/staticcheck-action)) 以了解支持选项，例如指定 Staticcheck 版本。

# 发布您的应用程序

当然——你的单元测试是一件美丽的事物，你可能甚至正在与代码覆盖率极乐世界调情。但真正的考验，我的朋友，在于你必须将你创作的杰作打包并投入野外——这就是 GoReleaser([`goreleaser.com/`](https://goreleaser.com/))登场的时候，准备好将你的发布过程从令人尴尬的磨难转变为自动化的交响曲。

忘记为每个该死的操作系统构建二进制文件，或者痛苦地处理 tar 包和校验和。想象一下，你的发布烦恼就像一个和谐的编码会议一样神话般，绝对没有任何东西会出错。进入交叉编译、自动版本标记、Docker 镜像创建、Homebrew taps 的领域... GoReleaser 不仅仅是一个工具；它是你发布日理智的守护者。

本质上，GoReleaser 是你的个人发布管家。你描述你希望你的宝贵 Go 应用程序如何打包和分发，它以熟练的流水线效率处理所有琐碎的细节。需要为每个已知的操作系统提供预构建的二进制文件的新颖 GitHub 发布？没问题。想要将 Docker 镜像推送到你最喜欢的注册表？轻而易举。

```go
builds:
-   ldflags:
      - -s -w
      - -extldflags "-static"
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - windows
      - darwin
    goarch:
      - amd64
    mod_timestamp: '{{ .CommitTimestamp }}'
archives:
-   name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    wrap_in_directory: true
    format: binary
    format_overrides:
      - goos: windows
        format: zip
dockers:
  - image_templates:
      - "ghcr.io/alexrios/endpoints:{{ .Tag }}"
      - "ghcr.io/alexrios/endpoints:v{{ .Major }}"
      - "ghcr.io/alexrios/endpoints:v{{ .Major }}.{{ .Minor }}"
      - "ghcr.io/alexrios/endpoints:latest"
```

太好了！现在，我们希望在每次标记我们的仓库时都能使我们的二进制文件可用。为了实现这一点，我们应该使用一个新的工作流程与 GoReleaser 作业：

```go
name: GoReleaser
on:
  push:
    tags:
      - '*'
jobs:
  goreleaser:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: write
    steps:
      -
        name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
      -
        name: Set up Go
        uses: actions/setup-go@v2
        with:
          go-version: 1.21
      -
        name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v2
        with:
          version: latest
          args: release --rm-dist --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

我记得有一次，我花了整个下午手动编写发布说明，祈祷你没有错过一个关键的错误修复。有了 GoReleaser，我学会了嘲笑我过去的痛苦。它愉快地自动生成发布说明，发布推文公告，如果你请求的话，甚至可能烘焙庆祝的饼干（嗯，可能不是饼干）。

就像一位智慧的老将军检阅部队一样，我痛苦地意识到，一个稳固的 CI 设置的价值相当于“无 bug”代码的重量。在 CI 之前的史前时代，我们花费了数天，有时甚至数周，来解开在发布前才出现的巨大集成混乱。CI 是这种混乱的解药。

让我们像一场高风险的同步舞蹈一样思考 CI：小而频繁的更改持续集成是一种优雅的华尔兹。大而罕见的合并？嗯，那就像一个混乱的 mosh pit，没有人能从那里毫发无损地离开。

# 摘要

在本章中，我们探讨了分布式 Go 应用程序的方法和工具。我们专注于细致的依赖管理、测试和集成过程的自动化，以及高效的软件发布策略。我们从 Go 模块和工作空间开始，讨论了它们如何通过更好的依赖管理来增强项目的一致性和可靠性。然后，我们探讨了持续集成及其在保持高软件质量中的关键作用。最后，我们介绍了使用 GoReleaser 部署应用程序的基本知识，它通过自动化跨不同平台的打包和分发来简化发布过程。这些是构成你的综合项目基础的关键概念和工具。

当你进入下一章的综合项目时，你将有机会应用本书中学到的所有知识和技能。这个最终项目旨在巩固你在实际场景中的理解和熟练程度，挑战你从开始到结束使用最佳实践和工具实现一个完整的解决方案。 

综合项目将证明你的学习之旅和宝贵的工作，展示你有效地开发、管理、自动化和发布健壮应用程序的能力。我对这一点感到兴奋——你呢？

# 第五部分：超越基础

在本部分中，我们将深入研究构建分布式缓存的复杂性，并探讨必要的系统编程实践。你将学习如何设计、实现和优化分布式缓存系统，以及有效的编码实践和保持系统编程社区更新的策略。

本部分包含以下章节：

+   *第十三章*，*综合项目* - *分布式缓存*

+   *第十四章*，*有效的编码实践*

+   *第十五章*，*通过系统编程保持敏锐*
