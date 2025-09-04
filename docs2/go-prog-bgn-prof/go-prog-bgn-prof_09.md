

# 使用 Go 模块定义项目

概述

本章深入探讨了使用 Go 模块来结构和管理工作 Go 项目。我们将从介绍模块的概念及其在组织代码中的重要性开始。本章还将涵盖创建第一个模块，同时讨论必要的`go.mod`和`go.sum`文件。

此外，我们将介绍如何将第三方模块作为依赖项使用，并提供有效管理这些依赖项的见解。本章将通过练习和活动提供实践经验，使您能够开发出更加结构化和易于管理的 Go 项目，促进代码重用并简化开发过程。

# 技术要求

对于本章，您需要 Go 版本 1.21 或更高版本。本章的代码可以在以下位置找到：[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter09`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter09)。

# 简介

在上一章中，我们学习了使用 Go 包创建可维护、可重用和模块化软件的重要性。我们学习了包的结构、合适的包命名原则以及可执行包和非可执行包的区别。还讨论了可导出和不可导出代码的概念。

在本章中，我们将在此基础上扩展知识，并探讨使用 Go 模块来定义项目，提高我们的软件开发能力。我们将了解 Go 模块是什么，它们如何有帮助，甚至创建我们自己的模块。我们将了解与 Go 模块一起工作所需的不同文件，以维护项目依赖项的完整性，然后学习如何消费第三方模块并管理它们。最后，我们将探讨如何创建包含多个模块的项目，以及何时这样做是有用的。

# 什么是模块？

在 Go 编程的世界里，模块是一个基本概念，它是组织、版本控制和管理工作及其依赖项的基石。将其视为一个自包含、封装的单元，它简化了依赖项管理的复杂性，同时促进了代码的重用和维护性。

Go 模块代表了一组离散的 Go 包，所有这些包都整齐地打包在一个共同的、版本化的伞状结构下。这种隔离确保了您的代码库保持一致性和良好的结构，使其更容易共享、协作和维护。模块旨在让您控制项目的外部依赖项，并提供一种结构化的机制来版本控制和管理工作。

## 使用 Go 模块时的关键组件

与 Go 模块工作相关的一些关键组件。让我们看看一些有助于我们为项目进行 Go 依赖项管理的方面：

+   `go.mod` 文件：Go 模块的核心是 `go.mod` 文件。该文件作为您模块的蓝图，包含有关模块路径和版本的基本信息，以及其依赖项的详细列表。这个详细的映射确保所有必需的包都得到了明确定义，并且它们的特定版本被记录下来。

+   `go.sum` 文件：`go.sum` 文件与 `go.mod` 文件协同工作，是 Go 模块管理中的一个重要组成部分。它包含所有在 `go.mod` 中列出的依赖项的加密校验和，如 SHA-256 哈希。这些校验和作为安全措施，确保下载的依赖项未被篡改或损坏。

+   **版本控制**：Go 模块引入了一个强大的版本控制系统，在依赖项管理中发挥着关键作用。每个模块都被分配一个唯一的版本标识符，通常通过版本控制系统的标签或提交哈希实现。这种细致的方法确保您的项目始终使用已知和验证的依赖项集。已发布的模块使用语义版本控制模型发布版本号，您可以在他们的网站上找到更多关于此的信息：[`semver.org`](https://semver.org)。

这三个方面的 Go 模块帮助我们进行项目依赖项管理。使用 Go 模块，您不再需要手动跟踪和管理项目的依赖项。随着您导入包，它们会自动添加到 `go.mod` 文件中，并带有版本信息，简化了确保代码与为它设计的确切依赖项集兼容的过程。

## go.mod 文件

`go.mod` 文件是 Go 模块的主要配置文件。它包含以下信息：

+   `module mymodule` 指定了模块路径为 `mymodule`。

+   `go.mod` 文件列出了模块所需的依赖项，包括它们的模块路径和特定版本或版本范围。

+   **替换指令（可选）**：这些指令允许您指定某些依赖项的替换，这在测试或解决兼容性问题时可能很有用。

+   **排除指令（可选）**：这些指令允许您排除可能存在已知问题的特定版本的依赖项。

下面是一个简单的 `go.mod` 文件示例：

```go
module mymodule
require (
  github.com/some/dependency v1.2.3
  github.com/another/dependency v2.0.0
)
replace (
  github.com/dependency/v3 => github.com/dependency/v4
)
exclude (
  github.com/some/dependency v2.0.0
)
```

前面的代码展示了如何轻松读取 `go.mod` 文件，列出项目的依赖项，并在使用 `replace` 指令进行本地工作时进行调整，或者根据需要排除某些依赖项。

## go.sum 文件

`go.sum` 文件包含项目中使用的特定版本依赖项的校验和列表。这些校验和用于验证下载的包文件的完整性。

`go.sum` 文件由 Go 工具链自动生成和维护。它确保下载的包未被篡改，并且项目始终使用依赖项的正确版本。

下面是一个简化的 `go.sum` 文件示例：

```go
github.com/some/dependency v1.2.3 h1:abcdefg...
github.com/some/dependency v1.2.3/go.mod h1:hijklm...
github.com/another/dependency v2.0.0 h1:mnopqr...
github.com/another/dependency v2.0.0/go.mod h1:stuvwx...
```

在前一个示例中，`go.sum` 文件的内容展示了如何验证 Go 项目的下载包文件的完整性。这是一个非常简单的例子；然而，在实际中，`go.sum` 文件可能会变得相当大，这取决于项目可能拥有的依赖项的大小和数量。

# 模块是如何有帮助的？

Go 模块提供了许多增强 Go 开发体验的好处。让我们更深入地看看 Go 模块是如何有帮助的。

## 精确且简化的依赖项管理

Go 模块最显著的优点之一是它们能够提供对依赖项的精确控制。当你在 `go.mod` 文件中指定依赖项时，你可以定义所需的精确版本，这消除了与不那么严格的依赖项管理方法相关的猜测工作和潜在兼容性问题。

Go 模块简化了添加、更新和管理依赖项的过程。在过去，Go 开发者必须依赖于 `GOPATH` 和 `vendor` 目录，这可能导致版本冲突，并使依赖项管理变得具有挑战性。Go 模块用更直观和高效的方法取代了这些做法。

## 版本控制和可重复性

Go 模块引入了一个健壮的版本控制系统。每个模块都带有特定的版本标识符或提交哈希。这种细致的版本控制确保你的项目依赖于一致且已知的依赖项集合。它促进了可重复性，这意味着你和你的合作者可以轻松地重新创建相同的发展环境，减少“在我的机器上它工作”的问题。

## 改善协作

通过定义良好的模块，在 Go 项目上进行协作变得更加容易。模块为你的代码提供了清晰的边界，确保它保持一致性和自包含。这使得你更容易与他人分享你的工作，其他人也可以在不担心破坏现有功能的情况下为你的项目做出贡献。

## 依赖项安全性

Go 模块通过 `go.sum` 文件整合了安全措施。通过在所有项目依赖项中包含前面章节中提到的加密校验和，你可以看到这是如何保护下载包免受潜在篡改或损坏的。

## 在促进隔离和模块化的同时易于使用

很容易看到 Go 模块如何帮助我们的程序。模块通过易于理解、更新和跟踪项目依赖项，使开发团队更容易维护。随着项目的演变，很容易跟上外部包的变化。

Go 模块促进了隔离和模块化。它们还提供了一个自然机制来隔离你的项目与全局工作空间。这种隔离促进了模块化，让你能够专注于构建自包含、可重用且易于管理和共享的组件。这建立在 Go 的惯用特性之上，并促进了开发团队在 Go 项目中的最佳实践。

Go 模块在 Go 1.11 版本中正式引入，它们提供了一种更复杂、结构化和版本感知的方式来管理项目依赖。鼓励开发者迁移到 Go 模块以进行现代 Go 项目开发。

# 练习 09.01 – 创建和使用你的第一个模块

在这个练习中，我们将看到如何轻松地创建我们的第一个 Go 模块：

1.  创建一个名为`bookutil`的新目录并进入它：

    ```go
    mkdir bookutil
    cd bookutil
    ```

1.  初始化一个名为`bookutil`的 Go 模块：

    ```go
    go mod init bookutil
    ```

1.  验证`go.mod`是否在你的项目目录中创建，并且模块路径设置为`bookutil`。

注意

在运行`go mod init`之后不会创建`go.sum`文件。它将在你与模块交互并添加其依赖时生成和更新。

1.  现在，让我们在模块的项目目录内创建一个名为`author`的目录，以专注于通过创建与书籍章节相关的函数来创建一个 Go 包。

1.  在`author`目录内，创建一个名为`author.go`的文件来定义包和函数。

    这里是`author.go`的起始代码：

    ```go
    package author
    import "fmt"
    // Author represents an author of a book.
    type Author struct {
        Name string
        Contact string
    }
    ```

1.  现在，我们可以添加必要的函数来创建我们的作者并定义作者可以执行的操作：

    ```go
    func NewAuthor(name, contact string) *Author {
        return &Author{Name: name, Contact: contact}
    }
    func (a *Author) WriteChapter(chapterTitle string, content string) {
        fmt.Printf("Author %s is writing a chapter titled   '%s'\n", a.Name, chapterTitle)
        fmt.Println(content)
    }
    func (a *Author) ReviewChapter(chapterTitle string, content string) {
        fmt.Printf("Author %s is reviewing a chapter titled '%s'\n", a.Name, chapterTitle)
        fmt.Println(content)
    }
    func (a *Author) FinalizeChapter(chapterTitle string) {
        fmt.Printf("Author %s has finalized the chapter titled '%s'.\n", a.Name, chapterTitle)
    }
    ```

1.  定义了作者包后，我们可以在模块中创建一个 Go 文件来演示如何使用它。让我们将这个文件命名为`main.go`，放在我们的目录根目录下：

    ```go
    package main
    import "bookutil/author "
    func main() {
        // Create an author instance.
        authorInstance := author.NewAuthor("Jane Doe",   "jane@example.com")
        // Write and review a chapter.
        chapterTitle := "Introduction to Go Modules"
        chapterContent := "Go modules provide a structured way to manage dependencies and improve code maintainability."
        authorInstance.WriteChapter(chapterTitle, chapterContent)
        authorInstance.ReviewChapter(chapterTitle, "This chapter looks great, but let's add some more examples.")
        authorInstance.FinalizeChapter(chapterTitle)
    }
    ```

1.  在文件夹中保存文件并运行以下命令：

    ```go
    go run main.go
    ```

运行前面的代码会产生以下输出：

```go
Author John Doe is writing a chapter titled 'Introduction to Go Modules':
Go modules provide a structured way to manage dependencies and improve code maintainability.
Author John Doe is reviewing a chapter titled 'Introduction to Go Modules':
This chapter looks great, but let's add some more examples.
Author John Doe has finalized the chapter titled 'Introduction to Go Modules'.
```

在这个练习中，我们学习了如何创建 Go 模块并使用它来运行程序。

注意

你的 Go 模块不必与你的 Go 包同名，因为你可以有一个 Go 模块包含多个包，并且一个项目也可以有一个 Go 模块。根据项目的主要目的命名模块是一个好的实践。

在这种情况下，模块的主要目的是管理和处理书籍章节和作者，因此模块的名称反映了更广泛的环境。名称`bookutil`提供了灵活性，可以包括与书籍相关操作相关的多个包，包括`author`包。

此外，还有一些关于模块命名的最佳实践，如`<prefix>/<descriptive-text>`和`github.com/<project-name/>`，你可以在 Go 文档中了解更多信息：[`go.dev/doc/modules/managing-dependencies#naming_module`](https://go.dev/doc/modules/managing-dependencies#naming_module)。

现在你已经成功创建了一个名为`bookutil`的 Go 模块，其中包含一个专注于书籍章节的`author`包，让我们来探讨使用外部 Go 模块的重要性以及它们如何增强你的项目。

# 你应该在何时使用外部模块，为什么？

在 Go 开发中，利用外部模块是一种常见的做法，这可以给你的项目带来好处。外部模块，也称为第三方依赖，在合理使用时提供了许多优势。在本节中，我们将探讨何时使用外部模块以及它们被采用背后的有力理由。

你应该使用外部模块来完成以下操作：

+   提高代码的可重用性和效率

+   扩展项目功能

+   转移依赖项管理

+   通过开源社区促进协作开发

+   通过开源代码利用经过验证的可靠性、社区支持和文档

然而，始终要谨慎行事，选择与你的项目目标和长期可持续性计划相一致的依赖项和模块。

# 练习 09.02 – 在我们的模块中使用外部模块

有时，在代码中，你需要为提供给某物的身份提供一个唯一的标识符。这个唯一的标识符通常被称为**通用唯一标识符**（**UUID**）。Google 提供了一个包来创建这样的 UUID。让我们看看如何使用它：

1.  创建一个名为 `myuuidapp` 的新目录并进入它：

    ```go
    mkdir myuuidapp
    cd myuuidapp
    ```

1.  初始化一个名为 `myuuidapp` 的 Go 模块：

    ```go
    go mod init myuuidapp
    ```

1.  验证 `go.mod` 文件是否已创建在你的项目目录中，并将模块路径设置为 `myuuidapp`。

1.  添加一个 `main.go` 文件。

1.  在 `main.go` 中，将主包名添加到文件顶部：

    ```go
    package main
    ```

1.  现在，添加我们将在这个文件中使用的导入：

    ```go
    import (
        "fmt"
        "github.com/google/uuid"
    )
    ```

1.  创建 `main()` 函数：

    ```go
    func main() {
    ```

1.  使用外部模块包生成一个新的 UUID：

    ```go
        id := uuid.New()
    ```

1.  打印生成的 UUID：

    ```go
        fmt.Printf("Generated UUID: %s\n", id)
    ```

1.  关闭 `main()` 函数：

    ```go
    }
    ```

1.  保存文件，然后运行以下命令以获取外部依赖项，从而更新你的 `go.mod` 文件并包含依赖信息：

    ```go
    go get github.com/google/uuid
    ```

1.  验证我们的 `go.mod` 文件现在已更新为包含新的 `require` 行的包依赖项。根据它们的发布版本，你的包版本号可能不同：

    ```go
    require github.com/google/uuid v1.3.1
    ```

1.  验证我们的 `go.sum` 文件现在已更新为新的依赖项。再次提醒，根据它们的发布版本，你的包版本号可能不同：

    ```go
    github.com/google/uuid v1.3.1 h1:KjJaJ9iWZ3jOFZIf1Lqf4laDRCasjl0BCmnEGxkdLb4=
    github.com/google/uuid v1.3.1/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
    ```

1.  运行代码：

    ```go
    go run main.go
    ```

运行前面的代码会产生以下输出，包含一个随机的 UUID：

```go
Generated UUID: 7a533339-58b6-4396-b7f7-d0a50216bf88
```

通过这样，你已经学会了如何在你的模块中使用外部模块的包，通过使用 Google 的开源代码生成一个唯一的标识符。在这个例子中，我们相信 Google 有经过良好测试的代码，并且它符合我们为代码库设定的标准。如果我们想要升级或降级外部包的版本，那么这将被转移到我们的 Go 模块上。接下来，我们将通过查看在项目中何时使用多个模块来扩展我们对模块的理解。

注意

更多关于 UUID 模块和包的信息可以在 GitHub 上找到：[`github.com/google/uuid/tree/master`](https://github.com/google/uuid/tree/master)。

# 在项目中消耗多个模块

你可以在项目中消费多个 Go 模块。就像你之前看到的 Google 模块示例一样，你可以在项目中使用该模块，同时使用你可能需要的其他 Go 模块。

## 活动 9.01 – 消费多个模块

在这个活动中，我们将在我们的代码中使用多个 Go 模块：

1.  创建一个新的 UUID 并打印该 UUID。

1.  使用 `rsc.io/quote` 模块获取并打印一个随机引言。

你的输出应该看起来像这样，第二行有不同的 UUID 和不同的随机句子：

```go
Generated UUID: 3c986212-f12d-415e-8eb5-87f61a6cbfee
Random Quote: Do not communicate by sharing memory, share memory by communicating.
```

注意

该活动的解决方案可以在本章 GitHub 仓库文件夹中找到：[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter09/Activity09.01`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter09/Activity09.01).

# 在项目中定义多个模块

Go 模块系统旨在管理整个模块的依赖项和版本，而不是模块内的子集或子项目。然而，可能存在这样的情况，即你的主项目中有多个不同的组件或子项目，并且这些组件或子项目都有自己的依赖项和版本要求。在这种情况下，你可以以这种方式组织你的项目，即每个组件都是其自己的模块，与主项目模块分开。这些子模块可以作为独立的 Go 模块进行维护，每个子模块都有自己的 `go.mod` 文件。

例如，如果你有一个包含主组件和其他两个组件的项目，并且每个组件都有独特的依赖项，你可以这样组织你的项目：

```go
myproject/
├── mainmodule/
│   ├── main.go
│   ├── go.mod
│   ├── go.sum
│   ├── ...
├── secondmodule/
│   ├── othermain.go
│   ├── go.mod
│   ├── go.sum
│   ├── ...
├── thirdmodule/
│   ├── othermain.go
│   ├── go.mod
│   ├── go.sum
│   ├── ...
```

每个子组件/模块（即 `secondmodule` 和 `thirdmodule`）被视为一个独立的 Go 模块，具有自己的 `go.mod` 文件和依赖项。

在以下情况下创建子模块是有意义的：

+   **组件有不同的依赖项**：当项目中的不同组件有不同的依赖项集合时，创建子模块可以让你分别管理这些依赖项

+   **有单独的版本要求**：如果不同的组件需要同一依赖项的不同版本，使用子模块可以帮助更有效地管理这些版本冲突

+   **有组件可重用性**：当你打算在多个项目中重用组件时，将其作为单独的模块可以促进其在各种环境中的重用

+   **有可维护性**：子模块可以提高代码组织性和可维护性，因为每个组件都可以单独开发、测试和维护

虽然技术上可以在项目中创建子模块，但这不是常规做法，并且应该在需要为项目中的不同组件进行单独的依赖项管理、版本控制或代码组织时进行。每个子模块都应该有自己的 `go.mod` 文件，该文件定义了其特定的依赖项和版本要求。

# Go 工作空间

在 Go 1.18 中，发布了*Go 工作区*功能，这改善了在同一项目本地处理多个`Go`模块的体验。最初，当在同一项目中处理多个 Go 模块时，您需要手动为每个模块编辑 Go 模块文件，使用`replace`指令来使用您的本地更改。现在，使用 Go 工作区，我们可以定义一个`go.work`文件，指定使用我们的本地更改，而无需手动管理多个`go.mod`文件。这对于处理大型项目或跨多个存储库的项目尤其有用。

## 练习 09.03 – 使用工作区

在这个练习中，我们将回顾在处理需要替换依赖项以使用本地更改的多个 Go 模块的项目时的情况。然后我们将更新示例代码，使其使用 Go 工作区来展示改进：

1.  创建一个名为`printer`的新文件夹并添加一个`printer.go`文件。

1.  在`printer.go`中，将`printer`包名添加到文件顶部：

    ```go
    package printer
    ```

1.  现在，添加我们将在此文件中使用的导入：

    ```go
    import (
        "fmt"
        "github.com/google/uuid"
    )
    ```

1.  创建导出的`PrintNewUUID()`函数，返回一个字符串：

    ```go
    func PrintNewUUID() string {
    ```

1.  使用外部模块包生成新的 UUID：

    ```go
        id := uuid.New()
    ```

1.  创建并返回一个字符串以打印生成的 UUID：

    ```go
        return fmt.Sprintf("Generated UUID: %s\n", id)
    ```

1.  关闭`PrintNewUUID()`函数：

    ```go
    }
    ```

1.  创建一个 Go 模块并安装必要的依赖项：

    ```go
    go mod init github.com/sicoyle/printer
    go mod tidy
    ```

1.  回退一个文件夹并创建一个与`printer`文件夹并排的新文件夹，命名为`othermodule`，并添加一个`main.go`文件。

1.  在`main.go`中，将`main`包名添加到文件顶部：

    ```go
    package main
    ```

1.  现在，添加我们将在此文件中使用的导入：

    ```go
    import (
        "fmt"
        "github.com/sicoyle/printer"
    )
    ```

1.  创建`main()`函数：

    ```go
    func main() {
    ```

1.  使用我们在`printer`模块中定义的`PrintNewUUID()`函数：

    ```go
        msg := printer.PrintNewUUID()
    ```

1.  打印生成的 UUID 消息字符串：

    ```go
        fmt.Println(msg)
    ```

1.  关闭`main()`函数：

    ```go
    }
    ```

1.  初始化名为`othermodule`的 Go 模块：

    ```go
    go mod init othermodule
    ```

1.  添加模块的要求：

    ```go
    go mod tidy
    ```

1.  查看 Go 尝试检索模块依赖项时的错误消息；`printer`包仅包含未在 GitHub 上公开的本地更改：

    ```go
    go: finding module for package github.com/sicoyle/printer
    go: othermodule imports
          github.com/sicoyle/printer: cannot find module...
    ```

1.  在 Go 工作区之前解决此问题的旧方法包括在`othermodule`目录内编辑 Go 模块以替换内容：

    ```go
    go mod edit -replace github.com/sicoyle/printer=../printer
    ```

1.  验证`othermodule/go.mod`文件是否已更新以包含以下内容：

    ```go
    module othermodule
    go 1.21.0
    replace github.com/sicoyle/printer => ../printer
    ```

1.  现在，我们可以成功整理我们的依赖项：

    ```go
    go mod tidy
    ```

1.  运行代码：

    ```go
    go run main.go
    ```

1.  运行前面的代码，显示以下输出，包含一个随机 UUID：

    ```go
    Generated UUID: 5ff596a2-7c0e-41fe-b0b1-256b28a35b76
    ```

我们刚刚看到了在引入 Go 工作区之前的工作流程。现在，让我们看看这个新功能带来的变化。

1.  将`othermodule/go.mod`的全部内容替换为以下内容：

    ```go
    module othermodule
    go 1.21.0
    ```

1.  运行整理命令；您将看到错误找到`printer`模块：

    ```go
    go mod tidy
    ```

1.  在`printer`目录中运行以下命令以初始化 Go 工作区：

    ```go
    go work init
    ```

1.  在工作区中使用您的本地更改：

    ```go
    go work use ./printer
    ```

1.  运行以下代码：

    ```go
    go run othermodule/main.go
    ```

1.  运行前面的代码，显示以下输出，包含一个随机 UUID：

    ```go
    Generated UUID: 5ff596a2-7c0e-41fe-b0b1-256b28a35b76
    ```

这个练习演示了在引入 Go 工作空间功能之前和之后的流程。这个功能为开发者提供了一种更好地管理大型项目之间以及包含多个可能需要更新的 `go.mod` 文件的不同存储库中的本地更改的方法。

本章我们覆盖了很多内容。让我们回顾一下我们所学到的所有内容。

# 摘要

在本章中，我们探讨了 Go 模块的世界，从了解模块是什么以及它们如何提供结构化的项目组织和项目依赖管理开始。我们介绍了两个关键的模块文件——`go.mod` 和 `go.sum`——它们负责处理依赖关系。我们还深入探讨了外部模块，强调它们在扩展项目功能以及对其可维护性产生的影响。我们讨论了在单个项目中使用和消费多个模块，以及 Go 工作空间的概念，用于在项目目录内管理多个模块。动手练习和活动加深了我们的理解。

在下一章中，我们将通过介绍包如何帮助团队在迭代、重用和维护项目时使项目更易于管理来增强我们对模块的理解。
