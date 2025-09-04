

# 命令行编程

概述

在本章中，我们将探讨从命令行进行编程。我们将看到 Go 是创建强大命令行工具和应用程序的绝佳选择，并讨论使用 Go 处理命令行的许多工具。

通过阅读本章，您将熟悉在 Go 中开发强大的命令行工具和应用程序。我们将从读取命令行参数和利用这些标志值来控制应用程序行为的基础知识开始，一瞥在应用程序内外处理大量数据的方法，并在过程中评估退出代码和最佳实践。然后，我们将进一步深入探讨优雅地处理中断、从我们的应用程序中启动外部命令以及使用 `go install` 的策略。最后，我们将学习如何创建**终端用户界面**（**TUIs**），这允许我们用 Go 编写强大且用户友好的命令行工具。

# 技术要求

对于本章，您需要 Go 版本 1.21 或更高。本章的代码可以在[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter13`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter13)找到。

# 简介

在上一章中，我们探讨了 Go 在处理时间数据时提供的强大结构。在本章中，我们将稍微转换一下方向，讨论 Go 在为应用程序创建强大界面方面有益的许多方法之一。

用户界面不总是必须是网络应用程序的前端网页。最终用户可以通过引人入胜的命令行界面以及使用**命令行****界面**（**CLI**）与软件进行交互。

Go 提供了许多允许我们为命令行编程的包。我们将查看其中的一些包，了解 Go 在创建强大命令行工具方面的位置，并了解一些当前在这个领域的努力。

# 读取参数

命令行参数是构建灵活和交互式命令行应用程序的基本方面。读取参数允许开发者使他们的应用程序更加动态和适应用户输入。命令行参数为用户提供了一种自定义程序行为的方式，而无需修改其源代码。通过从命令行捕获输入参数，开发者可以创建适应不同用例和场景的通用应用程序。

在 Go 语言中，`os` 包提供了一种直接访问这些参数的简单方法。`os.Args` 切片提供了一种方便的方式来访问命令行参数。这使得开发者能够检索有关文件路径、配置参数或与应用程序功能相关的任何其他输入信息。读取命令行参数的能力通过使应用程序更加交互性和用户友好来增强用户体验。

此外，命令行参数可以实现自动化和脚本化，使用户能够以编程方式传递输入。这种灵活性在需要以不同参数执行相同程序的场景中尤其有价值，使其成为脚本化和自动化任务的强大工具。

让我们深入了解读取命令行参数的过程，并通过一个简单的示例来展示，该示例使用个性化消息问候用户。

## 练习 13.01 – 使用作为参数传递的名称说“你好”

在这个练习中，我们将使用从命令行传递的参数来打印一个 `hello` 语句：

1.  导入 `fmt` 和 `os` 包：

    ```go
    package main
    import (
      "fmt"
      "os"
    )
    ```

1.  利用前面提到的 `args` 切片来捕获命令行参数：

    ```go
    func main() {
      args := os.Args
    ```

1.  对提供的参数数量进行验证，不包括提供的可执行文件名：

    ```go
      if len(args) < 2 {
        fmt.Println("Usage: go run main.go <name>")
        return
      }
    ```

1.  从提供的参数中提取名称：

    ```go
      name := args[1]
    ```

1.  显示个性化的问候信息：

    ```go
      greeting := fmt.Sprintf("Hello, %s! Welcome to the command line.", name)
      fmt.Println(greeting)
    }
    ```

1.  运行以下命令以执行代码：

    ```go
    go run main.go Sam
    ```

输出如下：

```go
Hello, Sam! Welcome to the command line.
```

通过这些，我们已经展示了使用 Go 语言捕获命令行参数的基本方法，并看到了以简单方式捕获输入数据的一些好处。虽然在这个例子中我们使用了 `os` 包，但其他包可以帮助实现读取应用程序提供的输入的相同目标，例如使用 `flags` 包。让我们看看标志在编程中对于命令行有多有用。

# 使用标志来控制行为

`flags` 包提供了一种比直接使用 `os` 包更高级和更结构化的方法来读取参数。标志简化了解析和处理命令行输入的过程，使得开发者更容易创建健壮且用户友好的命令行应用程序。

`flags` 包允许您定义带有相关类型和默认值的标志，从而清楚地表明用户应提供何种类型的输入。它还自动生成帮助信息，使您的程序更具自文档性。以下是 `flags` 包如何帮助读取和处理命令行参数的简要概述：

+   **定义标志**: 您可以定义标志，包括它们的类型和默认值。这提供了一种清晰且结构化的方式来指定预期的输入。

+   **解析标志**: 在定义标志之后，您可以解析命令行参数。这将使用用户提供的值初始化标志变量。

+   **访问标志值**：一旦你解析了传递给程序的标志值，你就可以通过变量访问定义的标志，并在整个应用程序中继续使用它们。

标志允许你自定义程序的行为，而无需修改源代码。例如，你可以创建允许你根据标志值切换行为的标志。你还可以使用基于某些标志值设置的基本条件逻辑。让我们完成一个练习，并利用`flags`包来有条件地说“你好”。

## 练习 13.02 – 使用标志有条件地说“你好”

在这个练习中，我们将使用`flags`包打印一个`hello`语句：

1.  导入`fmt`和`os`包：

    ```go
    package main
    import (
      "flag"
      "fmt"
    )
    ```

1.  为实用程序创建标志并设置默认值：

    ```go
    var (
      nameFlag = flag.String("name", "Sam", "Name of the person to say hello to")
      quietFlag = flag.Bool("quiet", false, "Toggle to be quiet when saying hello")
    )
    Parse the flags and conditionally say hello pending the value of the quiet flag:
    func main(){
      flag.Parse()
      if !*quietFlag {
        greeting := fmt.Sprintf("Hello, %s! Welcome to the command line.", *nameFlag)
        fmt.Println(greeting)
      }
    }
    ```

如果你运行`go run main.go`，你会收到以下输出。这是因为`quietFlag`默认为`false`，而`nameFlag`默认为`Sam`：

```go
Hello, Sam! Welcome to the command line.
```

然而，你可以为标志设置值。为此，你可以使用`nameFlag`并设置`quietFlag`的值。运行`go run main.go --name=Cassie –-quiet=false`的输出如下。这是因为`quietFlag`被设置为`false`，而`nameFlag`被设置为`Sam`：

```go
Hello, Cassie! Welcome to the command line.
```

或者，如果你使用`quietFlag`的值为`true`，则不会输出任何内容。所以，如果你运行`go run main.go --quiet=true`，那么你将看不到任何输出，因为我们已经使用了标志来控制程序预期的输出行为。

这段代码展示了如何使用标志来控制程序的行为。如果你在使用他人的命令行界面，那么你可以无缝地使用`help`标志来列出可用的定义标志。Go 语言中的`flag`包会根据程序中的标志自动生成帮助信息。要查看前面代码的可用帮助信息，你可以运行`go run main.go --help`。这将提供以下输出：

```go
Usage of /var/folders/qt/5jjdv1bj3h33t2rl40tpt56w0000gn/T/go-build1361710947/b001/exe/main:
-name string
Name of the person to say hello to (default "Sam")
-quiet
Toggle to be quiet when saying hello
```

通过使用`flags`包，你可以提高代码的可读性和可维护性，同时提供更友好的用户体验。它简化了处理各种类型输入的过程，并自动生成使用信息，使用户更容易理解和与命令行应用程序交互。现在，让我们看看如何将数据流进和流出应用程序。

# 在应用程序中流进和流出大量数据

在命令行应用程序中，为了性能和响应性，高效地处理大量数据至关重要。通常，命令行应用程序可能是更大数据处理管道中的一小部分。大多数人都不愿意坐下来逐个输入大量数据，比如数据集。

Go 语言允许你将数据流到你的应用程序中，这样你就可以分块处理信息，而不是一次性处理。这允许你有效地处理大量数据，减少内存开销，并为未来的可扩展性提供更好的支持。

当处理大量数据时，它们通常存储在文件中。这可以是从金融 CSV 文件、分析 Excel 文件到机器学习数据集。使用 Go 流式传输数据的几个主要优点包括：

+   **内存效率**：程序可以逐行读取和处理数据，减少内存消耗，因为您不必将整个数据读入内存

+   **实时分析**：用户可以观察处理数据结果的真实时分析

+   **交互式界面**：您可以通过增强命令行界面使其接受动态信息或在大数据处理时显示额外详细信息

数据机密性可能取决于您可能流式传输到命令行应用程序的数据类型。因此，可能采用不同的编码机制来隐藏文本，而不提供真实的安全性，或者作为保护数据的第一步。

*Rot13*，或旋转 13 个位置，是一种简单的字母替换密码，用字母表中该字母之后的第 13 个字母替换它。例如，字母 A 将变成 N，B 将变成 C，以此类推。它是一种对称密钥算法，通常用作加密的简单形式，以掩盖文本。此算法不提供显著的安全性，主要用于娱乐，通常不会在生产环境中用于保护数据。它也是完全可逆的，这意味着应用 Rot13 两次将产生相同的数据。这在发送和接收文本的环境中可能很有用，接收端可能知道或不知道数据是否已被编码。

让我们扩展我们新的 Rot13 知识，以便我们可以为一个命令行应用程序工作在有趣的流数据示例。

## 练习 13.03 – 使用管道、stdin 和 stdout 应用 Rot13 编码到文件

在这个练习中，我们将使用 Rot13 编码处理一些输入数据：

1.  导入所需的包：

    ```go
    package main
    import (
      "bufio"
      "fmt"
      "io"
      "os"
    )
    ```

1.  定义`rot13`函数，将 Rot13 编码应用于给定的字符串：

    ```go
    func rot13(s string) string {
      result := make([]byte, len(s))
      for i := 0; i < len(s); i++ {
        char := s[i]
        switch {
        case char >= 'a' && char <= 'z':
          result[i] = 'a' + (char-'a'+13)%26
        case char >= 'A' && char <= 'Z':
          result[i] = 'A' + (char-'A'+13)%26
        default:
          result[i] = char
        }
      }
      return string(result)
    }
    ```

1.  定义一个函数，从`stdin`读取数据，应用 Rot13 编码，并将输出写入`stdout`：

    ```go
    func processStdin() {
      reader := bufio.NewReader(os.Stdin)
      for {
        input, err := reader.ReadString('\n')
        if err == io.EOF {
          break
        } else if err != nil {
          fmt.Println("Error reading stdin:", err)
          return
        }
        encoded := rot13(input)
        fmt.Print(encoded)
      }
    }
    ```

1.  定义一个处理文件或用户输入的函数，应用 Rot13 编码，并将输出写入`stdout`：

    ```go
    func processFileOrInput() {
      var inputReader io.Reader
      // Check if a file path is provided
      if len(os.Args) > 1 {
        file, err := os.Open(os.Args[1])
        if err != nil {
          fmt.Println("Error opening file:", err)
          return
        }
        defer file.Close()
        inputReader = file
      } else {
        // No file provided, read user input
        fmt.Print("Enter text: ")
        inputReader = os.Stdin
      }
      // Process input and apply rot13 encoding
      scanner := bufio.NewScanner(inputReader)
      for scanner.Scan() {
        // Apply rot13 encoding to the input line
        encoded := rot13(scanner.Text())
        fmt.Println(encoded)
      }
      if err := scanner.Err(); err != nil {
        fmt.Println("Error reading input:", err)
      }
    }
    ```

1.  定义主函数：

    ```go
    func main() {
      // Check if data is available on stdin
      stat, _ := os.Stdin.Stat()
      if (stat.Mode() & os.ModeCharDevice) == 0 {
        // Data available on stdin, process it
        processStdin()
      } else {
        // No data on stdin, process file or user input
        processFileOrInput()
      }
    }
    ```

如果您运行`go run main.go`并输入一些文本，您将收到以下输出：

```go
Enter text: enjoy
rawbl
the
gur
book
obbx
```

要退出程序，您可以输入 Ctrl + C。此外，程序可以用于管道中，其中一条命令的输出成为命令行应用程序的输入，如果您使用 `cat data.txt` | `go run main.go`。`cat` 是一个可以将文件连接在一起的命令（对于 Windows，使用 `type` 命令进行连接）。如果您单独使用它，那么它提供了一个打印文件内容的简单方法。如果您声明一个 `data.txt` 文件，并使用以下命令将文件内容传递给命令行应用程序，那么您将看到类似的输出：

```go
cat data.txt | go run main.go
```

这是生成的输出：

```go
rawbl
gur
obbx
```

前面的练习演示了一些内容。首先，我们看到了如何使用 `bufio.NewReader` 逐行处理 `stdin` 数据，直到遇到文件结束错误。我们还看到了如何处理文件或输入数据并对其进行 Rot13 编码。最后，我们看到了如何使用相同的代码将大量数据通过管道输入程序进行编码。此代码展示了 Go 在命令行应用程序中流式传输大量数据的能力。程序必须通过 *Ctrl* + *C* 终止以中断从 `stdin` 的读取并退出程序。这为我们探索退出代码和最佳实践提供了完美的过渡，在我们探索中断之前，我们将更详细地探讨中断，我们将学习更多关于使用中断（如 *Ctrl* + *C*）来终止程序的知识。

# 退出代码和命令行最佳实践

确保适当的退出代码并遵循最佳实践对于无缝的用户体验至关重要。退出代码为命令行应用程序提供了一种向调用应用程序传达其状态的方式。一个定义良好的退出代码系统使用户和其他脚本能够理解应用程序是否成功执行或在运行时遇到了问题。

在 Go 中，`os` 包提供了一个使用 `os.Exit` 函数设置退出代码的直接方法。传统上，退出代码 0 表示成功，而任何非零代码表示错误。

例如，您可以检查上一个练习的状态代码并验证成功状态代码。为此，请在终端中运行 `echo $?`。`$?` 是一个特殊的 shell 变量，它保存了最后执行命令的退出状态，而 `echo` 命令将其打印出来。您将看到 0 退出代码的打印输出，表示成功执行状态，没有错误。您可以在程序中手动捕获错误并返回非零代码以表示错误。您甚至可以创建自定义退出代码，如下所示：

```go
const (
  ExitCodeSuccess = 0
  ExitCodeInvalidInput = 1
  ExitCodeFileNotFound = 2
)
```

这些可以通过 `os.Exit` 容易地使用，通过在希望退出的成功情况下放置 `os.Exit(ExitCodeSuccess)`，并在特定情况下使用其他错误代码来退出。

在使用适当的退出代码是重要的命令行最佳实践的同时，还有一些其他事项需要考虑：

+   **一致的日志记录**：使用有意义的消息来帮助故障排除。

+   **提供清晰的用法信息**：提供清晰简洁的用法信息，包括标志和参数。此外，一些包允许您提供示例命令。应该使用这些命令让其他人轻松了解如何使用命令。

+   **处理帮助和版本控制**：实现标志以显示帮助和版本信息。这有助于使您的应用程序更加用户友好，并提供一种确保他们使用最新版本的方法，通过检查版本信息。

+   **优雅终止**：应考虑退出代码并优雅地终止，确保按需执行适当的清理任务。

您在命令行应用程序最佳实践和退出代码考虑方面已经取得了很好的进展。然而，有时最终用户会提供中断来取消应用程序。让我们学习在这种情况下应该做什么和考虑什么。

# 通过观察中断来了解何时停止

在构建健壮的应用程序时，优雅地处理中断至关重要，确保软件能够适当地响应表示它应该停止或执行特定操作的信号。在 Go 中，实现这一点的标准方式是通过监控中断信号，允许应用程序以有序的方式关闭或清理资源。

优雅关闭或终止是计算机科学中一个重要的概念。不可预见的事件、服务器维护或外部因素可能需要您的应用程序优雅地停止。这可能包括释放资源、保存状态或通知已连接的客户端。优雅的关闭确保您的应用程序保持可靠和可预测，最大限度地减少数据损坏或丢失的风险。

在没有适当清理的情况下突然终止应用程序可能导致各种问题，如未完成的交易、资源泄露、数据损坏等。优雅的关闭通过提供完成正在进行中的任务和释放已获取资源的机会来减轻这些风险。

操作系统通过信号与运行中的进程进行通信。进程只是计算机上运行的程序。`os/signal` 包为在 Go 程序中方便地处理这些信号提供了方法。有常见的中断信号，如 `stdin` 和退出程序。

`signal.Notify` 函数允许您注册通道以接收指定的信号。这为在接收到中断信号时优雅地关闭您的应用程序奠定了基础。还有一些有效的关闭模式和最佳实践需要牢记，例如确保关闭网络连接、保存状态和向 goroutines 发送信号，以便在退出前完成其任务。此外，使用超时和 `context` 包增强了应用程序在关闭期间的响应性，防止其无限期地卡住。

优雅地处理中断信号是构建健壮且可靠的 Go 命令行应用程序的基本技能。通过遵循最佳实践和模式以确保优雅的终止，你可以确保你的软件即使在面对意外中断的情况下也能表现出可预测的行为。

你不仅可以用不同的中断优雅地停止程序，还可以从命令行应用程序中启动其他命令。

# 从你的应用程序启动其他命令

从你的 Go 应用程序中启动外部命令为你打开了与其他程序、进程和系统工具交互的机会。Go 中的`os/exec`包提供了启动和与外部进程交互的功能。你可以使用这个包运行基本命令，捕获它们的输出，并无缝地处理错误。它为更高级的命令执行场景提供了一个基础。

例如，`os/exec`包允许你通过配置工作目录、环境变量等属性来自定义命令的执行。你还可以通过来自原始命令行应用程序的标准输入流向子命令提供输入。

通过在命令行应用程序中运行其他命令，你可以将一些进程放在后台运行，允许应用程序继续执行，同时监控或与并行进程交互。你甚至可以与命令建立双向通信，使命令行应用程序和外部进程之间实现实时交互和数据交换。

当启动其他应用程序时，必须考虑到跨平台的问题。不同操作系统之间在 shell 行为和命令路径上存在差异。因此，当从命令行应用程序中执行子命令时，重要的是要考虑到无论最终用户使用哪种计算设备，都要保持一致和可靠的命令执行。幸运的是，`os/exec`包为 Go 中执行外部命令提供了一个跨平台解决方案，这使得在不同操作系统上编写代码变得更加容易。

既然我们已经讨论了从 Go 命令行应用程序中执行其他命令的有用性，让我们看看一个实际操作的例子。

## 练习 13.04 – 创建有时间限制的秒表

在这个练习中，我们将创建一个有时间限制的秒表，并从应用程序中启动另一个命令：

1.  导入必要的包：

    ```go
    package main
    import (
      "fmt"
      "os"
      "os/exec"
      "time"
    )
    ```

1.  在主函数中，设置秒表的计时限制，并允许用户输入来启动时钟：

    ```go
    func main() {
      timeLimit := 5 * time.Second
      fmt.Println("Press Enter to start the stopwatch...")
      _, err := fmt.Scanln() // Wait for user to press Enter
      if err != nil {
        fmt.Println("Error reading from stdin:", err)
        return
      }
      fmt.Println("Stopwatch started. Waiting for", timeLimit)
    ```

1.  等待时间限制，从命令行应用程序中执行其他命令，并关闭主函数：

    ```go
      time.Sleep(timeLimit)
      fmt.Println("Time's up! Executing the other command.")
      cmd := exec.Command("echo", "Hello")
      cmd.Stdout = os.Stdout
      cmd.Stderr = os.Stderr
      err = cmd.Run()
      if err != nil {
        fmt.Println("Error executing command:", err)
      }
    }
    ```

如果你运行`go run main.go`并在提示时按下 Enter 键来启动计时器，你将收到以下输出：

```go
Press Enter to start the stopwatch...
Stopwatch started. Waiting for 5s
Time's up! Executing the other command.
Hello
```

执行外部命令是一种强大的功能，允许你的 Go 应用程序与更广泛的环境交互。虽然这是一个简单的 echo 命令，但它展示了如果你扩展此代码以启动其他应用程序、并行或后台运行命令等，这一功能有多强大。

# 终端用户界面

Go 命令行编程的最新更新中包括了一些**终端用户界面**，简称**TUI**。在 Go 中创建一个 TUI 打开了构建交互式命令行应用的新世界。构建终端用户界面的过程中涉及一些基本概念：

+   **组件**：TUI 由各种组件组成，如按钮、输入字段和/或列表

+   **布局**：在结构化的布局中排列组件对于整洁直观的设计至关重要

+   **用户输入处理**：处理键盘事件中的用户输入是交互式界面的基本要素

一些 TUI 包提供了对事件处理的支持，例如鼠标事件和按键，基于输入数据的动态更新，或自定义 UI 组件的外观。有几个流行的 TUI 包可用。在接下来的练习中，我们将查看其中一个，我们将在此基础上构建本章之前的练习。

## 练习 13.05 – 为我们的 Rot13 管道创建包装器

在这个练习中，我们将为我们在 *练习 13.03* 中创建的 Rot13 管道创建一个 TUI 包装器。

1.  导入必要的包：

    ```go
    package main
    import (
      "bufio"
      "fmt"
      "io"
      "os"
      "strings"
      tea "github.com/charmbracelet/bubbletea"
    )
    ```

1.  表示 TUI 选择和模型具体信息：

    ```go
    var choices = []string{"File input", "Type in input"}
    type model struct {
      cursor int
      choice string
    }
    func (m model) Init() tea.Cmd {
      return nil
    }
    ```

1.  定义处理模型更新的函数：

    ```go
    func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
      switch msg := msg.(type) {
      case tea.KeyMsg:
        switch msg.String() {
        case "ctrl+c", "q", "esc":
          return m, tea.Quit
        case "enter":
          m.choice = choices[m.cursor]
          return m, tea.Quit
        case "down", "j":
          m.cursor++
          if m.cursor >= len(choices) {
            m.cursor = 0
          }
        case "up", "k":
          m.cursor--
          if m.cursor < 0 {
            m.cursor = len(choices) - 1
          }
        }
      }
      return m, nil
    }
    ```

1.  定义 TUI 视图：

    ```go
    func (m model) View() string {
      s := strings.Builder{}
      s.WriteString("Select if you would like to work with file input or type in input:\n\n")
      for i := 0; i < len(choices); i++ {
        if m.cursor == i {
          s.WriteString("(•) ")
        } else {
          s.WriteString("( ) ")
        }
        s.WriteString(choices[i])
        s.WriteString("\n")
      }
      s.WriteString("\n(press q to quit)\n")
      return s.String()
    }
    ```

1.  定义与之前相同的 Rot13 编码函数：

    ```go
    func rot13(s string) string {
      result := make([]byte, len(s))
      for i := 0; i < len(s); i++ {
        char := s[i]
        switch {
        case char >= 'a' && char <= 'z':
          result[i] = 'a' + (char-'a'+13)%26
        case char >= 'A' && char <= 'Z':
          result[i] = 'A' + (char-'A'+13)%26
        default:
          result[i] = char
        }
      }
      return string(result)
    }
    ```

1.  定义从 `stdin` 读取数据以应用 Rot13 编码的函数：

    ```go
    func processStdin() {
      reader := bufio.NewReader(os.Stdin)
      for {
        input, err := reader.ReadString('\n')
        if err == io.EOF {
          break
        } else if err != nil {
          fmt.Println("Error reading stdin:", err)
          return
        }
        encoded := rot13(input)
        fmt.Print(encoded)
      }
    }
    ```

1.  定义处理文件输入的修改后的函数：

    ```go
    func processFile(filename string) {
      var inputReader io.Reader
      file, err := os.Open(filename)
      if err != nil {
        fmt.Println("Error opening file:", err)
        return
      }
      defer file.Close()
      inputReader = file
      // Process input and apply rot13 encoding
      scanner := bufio.NewScanner(inputReader)
      for scanner.Scan() {
        encoded := rot13(scanner.Text())
        fmt.Println(encoded)
      }
      if err := scanner.Err(); err != nil {
        fmt.Println("Error reading input:", err)
      }
    }
    ```

1.  定义启动 TUI 的主函数：

    ```go
    func main() {
      p := tea.NewProgram(model{})
      m, err := p.Run()
      if err != nil {
        fmt.Println("Error running program:", err)
        os.Exit(1)
      }
      if m, ok := m.(model); ok && m.choice != "" {
        fmt.Printf("\n---\nYou chose %s!\n", m.choice)
      }
      if m, ok := m.(model); ok && m.choice != "" && m.choice == "File input" {
        processFile("data.txt")
      }
      if m, ok := m.(model); ok && m.choice != "" && m.choice == "Type in input" {
        processStdin()
      }
    }
    ```

如果你运行 `go run main.go` 并选择“文件输入”，你会收到以下输出：

```go
Select if you would like to work with file input or type in input:
(•) File input
( ) Type in input
(press q to quit)
---
You chose File input!
rawbl
gur
obbx
```

如果你运行 `go run main.go` 并选择“类型”输入，你会收到以下输出：

```go
Select if you would like to work with file input or type in input:
( ) File input
(•) Type in input
(press q to quit)
---
You chose Type in input!
enjoy
rawbl
the
gur
book
obbx
```

以下示例是在我们之前的练习基础上进行的扩展。在这里，我们提供了一个非常简单的包装器，用于 Rot13 编码练习的入口，提供了一个很好的用户界面，如果你打算使用默认的数据文件作为输入或提供自己的输入。这个 TUI 故意设计得简单，以展示在定义模型接口以便它与终端用户界面一起工作时，涉及的内容相当多。

现在，让我们看看使用 `go install` 消费其他人的命令行应用的样子。

# go install

你可以使用 `go install` 命令安装 Go 命令行应用。这个命令是 Go 工具链提供的一个强大工具，它可以在你的工作空间的 bin 目录中编译和安装 Go 应用程序。这允许你从任何终端窗口全局运行你的应用程序。要安装一个 Go 应用程序，你只需导航到项目的根目录并运行 `go install`。

此命令通过提供 `GOOS` 标志来考虑跨平台编译，您可以使用该标志指定要针对哪个操作系统，以及 `GOARCH` 标志，您可以使用该标志指定要针对的底层架构。

一个常见的 Go 语言包示例，您可以使用它来生成 Go 语言的命令行界面，是 `cobra` 包。这也是一个工具，如果您想进一步深入开发您的编程技能，可以使用它来快速开发基于 Cobra 的应用程序。此包提供了一个使用 `go install` 命令的简单示例：

```go
go install github.com/spf13/cobra-cli@latest
```

前面的命令安装了所有使用 Cobra CLI 所需的依赖项。因此，我的机器知道了这个工具，您现在可以轻松地使用您刚刚安装的命令行程序，如下所示：

```go
cobra-cli –help
```

Cobra 是一个用于 Go 语言的 CLI 库，它赋予了应用程序强大的功能。

它可以生成必要的文件，以快速创建 Cobra 应用程序：

```go
Usage:
 cobra-cli [command]
Available Commands:
 add Add a command to a Cobra Application
completion Generate the autocompletion script for the specified shell
help Help about any command
init Initialize a Cobra Application
Flags:
-a, --author string author name for copyright attribution (default "YOUR NAME")
--config string config file (default is $HOME/.cobra.yaml)
-h, --help help for cobra-cli
-l, --license string name of license for the project
--viper use Viper for configuration
Use "cobra-cli [command] --help" for more information about a command.
```

有了这些，您就知道了如何安装他人的命令行应用程序。现在，让我们总结一下本章所学的内容。

# 摘要

在本章中，我们研究了通过命令行进行编程的各种方法。我们揭示了 Go 语言如何成为创建命令行应用程序的绝佳选择，以及如何使用原生 Go 工具链。

从 `os` 和 `flag` 包开始，我们探讨了如何从命令行读取应用程序的参数。然后，我们研究了用于控制程序行为的标志，并探讨了如何通过在应用程序中流式传输大量数据来阐明 Go 语言中的命令行应用程序如何成为更大程序管道的一部分。

我们还探讨了如何优雅地处理 CLI 关闭过程，包括讨论退出码和中断，以及在我们的命令行应用程序中调用其他命令。我们通过查看终端 UI，将 CLI 提升到下一个层次，并使用原生 Go 工具链安装其他 CLI 来结束这一章。

在下一章中，我们将使用 Go 语言查看文件和系统。虽然我们在本章中已经涉及了这一点，以便为我们的 CLI 应用程序读取输入数据，但我们将在下一章深入探讨读取和写入文件。
