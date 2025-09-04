

# 理解系统调用

在本章中，你将开始一段探索系统调用世界的旅程，这些基本接口将用户级程序与操作系统内核连接起来。通过相关的类比和现实世界的平行关系，我们将揭示软件执行的复杂舞蹈，强调内核、用户模式和内核模式的关键作用。

理解系统调用及其与操作系统的交互对于任何希望构建高效和健壮应用的软件开发者至关重要。在本书的更广泛背景下，本章为后续关于高级操作系统交互和系统级编程的讨论奠定了基础。此外，在现实世界的背景下，掌握这些概念使开发者能够优化软件性能，在系统级别解决问题，并充分利用操作系统的能力。

在本章中，我们将涵盖以下主要主题：

+   系统调用简介

+   `syscall` 包

+   深入了解 `os` 和 `x/sys` 包

+   每日系统调用

+   开发和测试**命令行界面**（CLI）程序

到本章结束时，你不仅将掌握系统调用的理论基础，还将通过使用 Go 构建 CLI 应用程序获得实践经验。

# 技术要求

您可以在[`github.com/PacktPublishing/System-Programming-Essentials-with-Go/tree/main/ch3`](https://github.com/PacktPublishing/System-Programming-Essentials-with-Go/tree/main/ch3)找到本章的源代码。

# 系统调用简介

系统调用，通常称为“syscalls”，是操作系统接口的基本组成部分。它们是由操作系统内核提供的低级函数，允许用户级进程请求内核的服务。

如果你对这个概念还不熟悉，一些类比可以使理解更加容易。让我们将这个想法与旅行联系起来。

用户模式与内核模式

处理器（或 CPU）有两种操作模式：用户模式和内核模式（也称为管理模式或特权模式）。这些模式决定了程序对系统资源的访问和控制级别。用户模式受限，不允许直接访问某些关键系统资源，而内核模式具有更多权限，可以访问这些资源。权限已授予，请谨慎行事

当涉及到系统调用时，内核扮演着严格的边境控制官员的角色。系统调用就像我们需要穿越软件执行多样化景观的护照。将内核想象成一个高度设防的国际边境检查站。正如旅行者需要获得进入外国土地的许可一样，我们的进程需要获得访问内核资源的批准。系统调用作为护照，允许我们穿越用户和内核空间之间的边界。

## 服务目录和标识

就像一本旅行指南一样，内核通过系统调用**应用程序编程接口**（**API**）提供了一整套服务。这些服务从创建新进程到处理**输入和输出**（**I/O**）操作，如外国国家的便利设施和景点。

一个数字代码唯一地标识每个系统调用，就像你的护照号码。然而，这个编号系统在日常使用中是隐藏的。相反，我们通过它们的名称与系统调用互动，就像旅行者通过当地名称而不是代码来识别服务和地标。例如，一个`open`系统调用可能在内核的内部系统调用表中被标识为数字 5。然而，作为程序员，我们通过其名称“`open`”来引用这个系统调用，就像旅行者通过当地名称而不是 GPS 坐标来识别地点。

系统调用表的位置取决于操作系统和架构，但如果你对这些标识符感兴趣，可以访问以下链接中的定制表格：

[`filippo.io/linux-syscall-table/`](https://filippo.io/linux-syscall-table/)

## 信息交换

系统调用不是单向交易；它们涉及仔细的信息交换。每个系统调用都带有参数，这些参数规定了哪些数据需要在用户空间（你的进程域）和内核空间（内核的领域）之间传输。想象一下，这是一个协调良好的跨境对话，信息在双方之间无缝传递。

当你使用`write()`系统调用来将数据保存到文件时，你不仅传递数据，还传递有关写入位置的信息（例如，文件描述符和数据缓冲区）。这种数据交换就像跨境对话，数据在用户和内核空间之间无缝移动。

# `syscall`包

我们观察到一种一致的趋势：我们需要的绝大多数功能都可以在标准库中轻松访问，这是其全面性和实用性的证明。然而，在这个模式中有一个显著的例外——`syscall`包。

该包一直是跨各种架构和操作系统与系统调用和常量接口的基础。然而，随着时间的推移，出现了几个问题，导致其被弃用：

+   `syscall`就像你承诺要清理的……某个时候会清理的过度拥挤的衣柜。

+   **测试限制**：该包的大部分内容缺乏明确的测试。此外，由于包的设计，跨平台测试不可行。

+   `syscall`包创下了记录，成为标准库中最少维护、测试和记录的包之一。

+   `syscall`包，每个系统都有其独特的变体，对于开发者来说就像是一个谜团中的谜团。虽然人们希望有清晰的指导，但`godoc`工具只提供了一个简要的预览，非常类似于电影预告片，只展示了亮点。这种针对其原生环境的精选显示，加上普遍缺乏文档，使得理解和有效使用这个包成为一项具有挑战性的任务。

+   `syscall`包常常感觉像是在进行无休止的追逐，就像追逐独角兽一样。操作系统随着不断的进化，提出了超出了 Go 团队控制范围的挑战。例如，FreeBSD 的变化影响了这个包的兼容性。

为了解决这些担忧，Go 团队提出了以下建议：

+   `syscall`包将被冻结，这意味着不会对其做任何进一步的修改。这包括即使引用的操作系统中有所变化，也不会更新它。

+   `x/sys` ([`pkg.go.dev/golang.org/x/sys`](https://pkg.go.dev/golang.org/x/sys))是为了替换`syscall`包而创建的。这个新包更容易维护、文档化和用于跨平台开发。

+   废弃：虽然`syscall`包将继续存在并运行，但所有新的公共开发都转移到了`x/sys`。`syscall`包的文档将指导用户转向这个新的存储库。

从本质上讲，虽然`syscall`包在一段时间内发挥了作用，但它带来的维护、文档和兼容性方面的挑战，使得它不得不被废弃，转而采用更结构化和可维护的`x/sys`方法。

关于这个决定的更多信息，Rob Pike 有一篇帖子解释了这一决定([`go.googlesource.com/proposal/+/refs/heads/master/design/freeze-syscall.md`](https://go.googlesource.com/proposal/+/refs/heads/master/design/freeze-syscall.md))。

# 深入了解 os 和 x/sys 包

正如我们在 Go 文档中关于`x/sys`包所看到的那样：

“*`x/sys`的主要用途是在其他提供更便携接口到系统的包内部，例如“os”、“time”和“net”*。”

*如果可能的话，使用那些包而不是这个包。关于这个包中函数和数据类型的详细信息，请参阅相应操作系统的手册。这些调用返回 err == nil 表示成功；否则 err 是描述失败的操作系统错误。在大多数系统中，这个错误有* *type syscall.Errno.*”

## x/sys 包 – 底层系统调用

Go 语言中的`x/sys`包提供了对底层系统调用的访问。它通常在需要直接与操作系统交互或进行特定平台操作时使用。使用`x/sys`时需要谨慎，因为不当的使用可能导致系统不稳定或安全问题。

要使用这个包，你应该使用 Go 工具下载它：

```go
go get -u golang.org/x/sys
```

让我们来探讨这个包能提供什么。

### 系统调用

这里有一些系统调用调用和常量：

+   `unix.Syscall()`: 使用参数调用特定的系统调用

+   `unix.Syscall6()`: 与`Syscall()`类似，但用于具有六个参数的系统调用

+   `unix.SYS_*`: 表示各种系统调用的常量（例如，`unix.SYS_READ`, `unix.SYS_WRITE`）

例如，下面两个代码片段会产生相同的结果，打印出`"Hello World!"`。

使用`fmt`包，你可以得到以下输出：

```go
fmt.Println("Hello World!")
```

通过使用`x/sys`包，你可以得到以下内容：

```go
unix.Syscall(unix.SYS_WRITE, 1,
  uintptr(unsafe.Pointer(&[]byte("Hello, World!")[0])),
  uintptr(len("Hello, World!")),
 )
```

如果我们决定使用低级抽象而不是`fmt`包，事情可能会变得非常复杂。

我们可以通过类别继续探索包 API。

### 文件操作

这些函数让我们可以与普通文件交互：

+   `unix.Create()`: 创建新文件

+   `unix.Unlink()`: 删除文件

+   `unix.Mkdir()`, `unix.Rmdir()`, 和 `unix.Link()`: 创建和删除目录和链接

+   `unix.Getdents()`: 获取目录条目

### 信号

这里有两个与 OS 信号交互的函数示例：

+   `unix.Kill()`: 向进程发送终止信号

+   `unix.SIGINT`: 中断信号（通常称为*Ctrl* + *C*）

### 用户和组管理

我们可以使用以下调用管理用户和组：

+   `syscall.Setuid()`, `syscall.Setgid()`, `syscall.Setgroups()`: 设置用户和组 ID

### 系统信息

我们可以使用`Sysinfo()`函数分析一些关于内存和交换使用以及平均负载的统计数据：

+   `syscall.Sysinfo()`: 获取系统信息

### 文件描述符

虽然这不是日常任务，但我们也可以直接与文件描述符交互：

+   `unix.FcntlInt()`: 对文件描述符执行各种操作

+   `unix.Dup2()`: 复制文件描述符

### 内存映射文件

Mmap 是内存映射文件的缩写。它提供了一种机制，可以在不依赖系统调用的前提下读写文件。当使用`Mmap()`时，操作系统会为程序分配一段虚拟地址空间，该空间直接“映射”到相应的文件部分。如果程序访问地址空间的那部分数据，它将检索存储在文件相关部分的数据：

+   `syscall.Mmap()`: 将文件或设备映射到内存

## 操作系统功能

Go 语言中的`os`包提供了一组丰富的函数，用于与操作系统交互。它分为几个子包，每个子包都专注于 OS 功能的一个特定方面。

以下是一些文件和目录操作：

+   `os.Create()`: 创建或打开文件以写入

+   `os.Mkdir()` 和 `os.MkdirAll()`: 创建目录

+   `os.Remove()` 和 `os.RemoveAll()`: 删除文件和目录

+   `os.Stat()`: 获取文件或目录信息（元数据）

+   `os.IsExist()`, `os.IsNotExist()`, 和 `os.IsPermission()`: 检查文件/目录存在或权限错误

+   `os.Open()`: 以读取方式打开文件

+   `os.Rename()`: 重命名或移动文件

+   `os.Truncate()`: 调整文件大小

+   `os.Getwd()`: 获取当前工作目录

+   `os.Chdir()`: 更改当前工作目录

+   `os.Args`: 命令行参数

+   `os.Getenv()`: 获取环境变量

+   `os.Setenv()`: 设置环境变量

以下是与进程和信号相关的内容：

+   `os.Getpid()`: 获取当前进程 ID

+   `os.Getppid()`: 获取父进程 ID

+   `os.Getuid()`和`os.Getgid()`: 获取用户和组 ID

+   `os.Geteuid()`和`os.Getegid()`: 获取有效用户和组 ID

+   `os.StartProcess()`: 启动新进程

+   `os.Exit()`: 退出当前进程

+   `os.Signal`: 表示信号（例如，`SIGINT`，`SIGTERM`）

+   `os/signal.Notify()`: 在接收到信号时通知

`os`包允许你创建和操作进程。你可以启动新进程，获取当前进程的信息，并操作其属性：

```go
package main
import (
     "fmt"
     "os"
     "os/exec"
)
func main() {
     // Start a new process
     cmd := exec.Command("ls", "-l")
     cmd.Stdout = os.Stdout
     cmd.Stderr = os.Stderr
     err := cmd.Run()
     if err != nil {
          fmt.Println(err)
          return
     }
     // Get the current process ID
     pid := os.Getpid()
     fmt.Println("Current process ID:", pid)
}
```

该程序的主要部分如下：

+   `exec.Command("ls", "-l")`: 这将创建一个新的命令来运行带有`-l`标志的`ls`命令。

+   `cmd.Stdout = os.Stdout`: 这将`ls`命令的标准输出重定向到主程序的标准输出。

+   `cmd.Stderr = os.Stderr`: 类似地，这将`ls`命令的标准错误重定向到主程序的标准错误。

+   `err := cmd.Run()`: 这将运行`ls`命令。如果在执行过程中出现错误，它将被存储在`err`变量中。

+   `os.Getpid()`: 这将检索当前进程的进程 ID。

虽然`os`包提供了许多系统相关任务的底层接口，但`syscall`(和`x/sys`)包允许你直接进行更底层的系统调用。这在你需要对系统资源进行精细控制时非常有用。

## 可移植性

虽然`x/sys`是进行系统调用的首选包，但你必须明确选择 Unix 和 Windows。与操作系统交互的推荐方式是使用`os`包。当你将程序构建到特定的操作系统和架构时，编译器将执行繁重的工作以使用适当的系统调用版本。

例如，在 Windows 中，你需要调用具有以下签名的函数：

```go
SetEnvironmentVariable(name *uint16, value *uint16) (err error)
```

对于基于 Unix 的系统，签名甚至没有相同的名称，正如我们将在下一个片段中看到的那样：

```go
Setenv(key, value string) error
```

为了避免这种“签名俄罗斯方块”，我们可以使用具有相同语义的`os`包中的函数：

```go
Setenv(key, value string) error
```

（是的！签名与 Unix 版本相同。）

注意

`syscall`([`pkg.go.dev/syscall`](https://pkg.go.dev/syscall))的主要用途是在提供更可移植系统接口的其他包内部，例如`os`、`time`和`net`。

从现在开始，我们利用`os`包，只有在特殊情况下才会直接调用`x/sys`包。

### 最佳实践

作为使用 Go 中的`os`和`x/sys`包的系统程序员，请考虑以下最佳实践：

+   对于大多数任务，使用`os`包，因为它提供了一个更安全和更可移植的接口

+   仅在需要精细控制系统调用的情况下保留`x/sys`包

+   使用 `x/sys` 包时，请注意平台特定的常量和类型，以确保跨平台兼容性

+   认真处理系统调用和 `os` 包函数返回的错误，以维护应用程序的可靠性

+   在不同的操作系统上测试你的系统级代码，以验证其在各种环境中的行为

让我们看看我们如何追踪我们在终端上日常执行的命令中发生的事情。

# 每日系统调用

在我们的程序中，每次都会发生几个系统调用。我们可以使用 `strace` 工具追踪这些调用。

## 跟踪系统调用

`strace` 工具可能不是所有 Linux 发行版都预安装的，但在大多数官方仓库中都有。以下是如何在一些主要发行版上安装它的方法。

Debian（使用 APT）：运行以下命令：

```go
apt-get install strace -y
```

**Red Hat 家族（使用 DNF 和 YUM）**

+   当使用 `yum` 时，运行以下命令：

    ```go
    yum install strace
    ```

+   当使用 `dnf` 时，运行以下命令：

    ```go
    dnf install strace
    ```

**Arch Linux（使用 Pacman）**：运行以下命令：

```go
pacman -S strace
```

### 基本 strace 使用

使用 `strace` 的基本方法是调用 `strace` 实用程序，后跟程序名称；例如：

```go
strace ls
```

这将生成一个输出，显示系统调用、它们的参数和返回值。例如，`execve` 系统调用 ([`man7.org/linux/man-pages/man2/execve.2.html`](https://man7.org/linux/man-pages/man2/execve.2.html)) 可能看起来像这样：

```go
execve("/usr/bin/ls", ["ls"], 0x7ffdee76b2a0 /* 71 vars */) = 0
```

## 跟踪特定系统调用

如果你只想跟踪特定的系统调用，请使用 `-e` 标志后跟系统调用名称。例如，要跟踪 `ls` 命令的 `execve` 系统调用，请运行以下命令：

```go
strace -e execve ls
```

现在，我们可以使用我们的工具集中的新工具来追踪程序中的系统调用。考虑以下简单的 `main.go` 文件：

```go
package main
import "unix"
func main() {
     unix.Write(1, []byte{"Hello, World!"})
}
```

这个程序需要通过向标准输出写入数据与硬件设备交互，即我们的控制台。为了访问控制台并执行此操作，程序需要从内核获得权限。这种权限是通过系统调用获得的，例如请求访问特定功能，如向控制台发送消息，这允许你的程序利用控制台的资源。

`unix.Write` 函数正在使用两个参数被调用：

+   第一个参数是 `1`，它是 Unix-like 系统中标准输出（`stdout`）的文件描述符。这意味着程序将数据写入程序运行的控制台或终端。

+   第二个参数是 `[]byte{"Hello, World!"}`，它是一个包含 `"Hello,` `World!"` 字符串的字节切片。

我们构建的程序将二进制文件命名为 `app`：

```go
go build -o app main.go
```

我们随后使用 `strace` 工具运行，过滤 `write` 系统调用：

```go
strace -e write ./app 2>&1
```

你应该看到以下输出作为结果：

```go
write(1, "Hello, World!", 13Hello, World!)           = 13
```

现在，是时候探索一个与操作系统交互的程序了。让我们制作并测试我们的第一个 CLI 应用程序。

# 开发和测试 CLI 程序

CLI 应用程序是软件开发、系统管理和自动化中的必备工具。在创建 CLI 应用程序时，与`stdin`（标准输入）、`stderr`（标准错误）和`stdout`（标准输出）的交互在确保其有效性和用户友好性方面起着至关重要的作用。

在本节中，我们将探讨为什么这些标准流是 CLI（命令行界面）开发不可或缺的组成部分。

## 标准流

`stdin`、`stderr`和`stdout`的概念深深植根于 Unix 哲学的“*一切皆文件*。”（我们将在第四章文件和目录操作中进一步探讨这一点。）这些标准化流为 CLI 应用程序与用户和其他进程之间的通信提供了一种一致的方式。用户已经习惯了 CLI 工具以某种方式工作，遵守这些约定增强了应用程序的可预测性和用户友好性。

CLI 应用程序最强大的特性之一是它们能够通过管道（详见第六章管道）无缝协作。在类 Unix 系统中，您可以链式连接多个 CLI 工具，每个工具处理前一个工具的`stdout`中的数据。这种模式允许高效地处理数据并实现复杂任务的自动化。当您的应用程序与`stdout`交互时，它成为这些管道中的宝贵构建块，使用户能够轻松创建复杂的流程。

### 输入灵活性

通过利用`stdin`，您的 CLI 应用程序可以接受来自各种来源的输入。用户可以通过键盘进行交互式输入，或者将其他进程的数据直接通过管道输入到您的工具中。此外，您的应用程序还可以从文件中读取输入，使用户能够处理存储在不同格式和位置的各类数据。这种灵活性使得您的应用程序能够适应广泛的用例场景。

### 输出灵活性

同样，通过使用`stdout`，您的 CLI 应用程序可以以易于重定向、保存到文件或用作其他进程输入的格式提供输出。这种适应性确保了用户能够以多种方式利用工具的输出，从而提高工作效率和灵活性。

### 错误处理

`stderr`专门设计用于错误消息。将错误消息与常规程序输出分开简化了用户的错误检测和处理。当您的应用程序遇到问题时，`stderr`提供了一个专门的通道来传达错误信息。这种分离使得用户能够迅速识别和解决问题。

### 跨平台兼容性

`stdin`、`stderr`和`stdout`的美丽之处在于它们的平台无关性。这些流在不同的操作系统和环境之间保持一致。因此，我们的命令行应用程序可以保持可移植性和兼容性，确保它们在各种系统上无需修改即可可靠地运行。

### 测试和调试

通过遵循使用`stderr`进行错误输出的惯例，可以使测试和调试更加直接。用户可以轻松地单独捕获和分析错误消息，与程序的正常输出分开。这种分离有助于在开发和生产环境中定位和解决问题。

### 日志记录

许多命令行界面（CLI）应用程序使用`stderr`记录错误消息。这种做法使用户能够有效地监控应用程序的行为和解决问题。适当的日志记录增强了应用程序的可维护性，并有助于其整体健壮性。

### 用户体验

在使用`stdin`、`stderr`和`stdout`时保持一致性有助于提升用户体验。用户熟悉这些流，并期望命令行应用程序以标准方式运行。这种熟悉性降低了新用户的学习曲线，并提高了整体用户满意度。

### 遵守惯例

在软件开发和脚本编写社区中，许多最佳实践和既定惯例都假设使用`stdin`、`stderr`和`stdout`。遵守这些惯例使得你的命令行应用程序更容易集成到现有的工作流程和实践中，为开发者和用户节省时间和精力。

## 文件描述符

你是否曾好奇过，你的电脑是如何在不费吹灰之力地管理所有这些打开的文件、网络连接和设备？好吧，有一个鲜为人知的秘密让一切运行得如此顺畅：文件描述符。这些不起眼的数字 ID 是电脑处理文件、目录、设备等背后的无名英雄。

从正式的角度来说，文件描述符是操作系统用来唯一标识和管理打开的文件、套接字、管道和其他 I/O 资源的抽象表示或数字标识符。它是程序引用打开资源的一种方式。

文件描述符可以表示不同类型的资源：

+   常规文件：这些是包含数据的磁盘文件

+   目录：磁盘上目录的表示

+   字符设备：提供对使用字符流工作的设备（如键盘和串行端口）的访问

+   块设备：用于访问块设备，如硬盘

+   套接字：用于进程间的网络通信

+   管道：用于进程间通信（IPC）

当 shell 启动一个进程时，它通常继承三个打开的文件描述符。描述符 0 代表标准输入，为进程提供输入的文件。描述符 1 代表标准输出，进程写入输出的文件。描述符 2 代表标准错误，进程写入错误消息和有关异常条件的通知的文件。这些描述符通常连接到交互式 shell 或程序中的终端。在`os`包中，`stdin`、`stdout`和`stderr`是打开的文件，分别指向标准输入、输出和错误描述符（[`cs.opensource.google/go/go/+/refs/tags/go1.21.1:src/os/file.go;l=64`](https://cs.opensource.google/go/go/+/refs/tags/go1.21.1:src/os/file.go;l=64)）。

总结来说，`stdin`、`stderr`和`stdout`对于开发有效、用户友好且可互操作的 CLI 应用程序至关重要。这些标准化流提供了一种灵活、灵活且可靠的输入、输出和错误处理方式。通过采用这些流，我们的 CLI 应用程序对用户更加易于访问和有价值，增强了他们自动化任务、处理数据和高效实现目标的能力。

## 创建 CLI 应用程序

让我们按照标准流的最佳实践创建并测试我们的第一个命令行界面（CLI）应用程序。

此程序将捕获所有给出的参数（从现在起称为单词）。当单词长度为偶数时，它将发送到`stdout`；否则，它将发送到`stderr`：

```go
words := os.Args[1:]
if len(words) == 0 {
    fmt.Fprintln(os.Stderr, "No words provided.")
    os.Exit(1)
}
```

第一行检索传递给程序的命令行参数，不包括程序名称本身。程序名称总是`os.Args`切片中的第一个元素（`os.Args[0]`），因此通过使用`[1:]`切片，它获取程序之后的全部参数。

条件检查`words`切片的长度是否为零，这意味着在程序名称之后没有提供任何命令行参数。如果没有提供参数，它将使用`fmt.Fprintln(os.Stderr, "No words provided.")`将一个`"No words provided."`错误信息打印到标准错误流。

然后它使用非零退出代码（`os.Exit(1)`）退出程序。在类 Unix 操作系统中，退出代码为 0 通常表示成功，而非零退出代码表示错误。在这种情况下，程序表示它遇到了由于缺少命令行参数而导致的错误：

```go
for _, w := range words {
    if len(w)%2 == 0 {
        fmt.Fprintf(os.Stdout, "word %s is even\n", w)
    } else {
        fmt.Fprintf(os.Stderr, "word %s is odd\n", w)
    }
}
```

此代码遍历`words`切片中的每个单词，检查其长度是偶数还是奇数，然后相应地将消息打印到标准输出或标准错误。

`main.go`文件将如下所示：

```go
package main
import (
     "fmt"
     "os"
)
func main() {
     words := os.Args[1:]
     if len(words) == 0 {
          fmt.Fprintln(os.Stderr, "No words provided.")
          os.Exit(1)
     }
     for _, w := range words {
          if len(w)%2 == 0 {
               fmt.Fprintf(os.Stdout, "word %s is even\n", w)
          } else {
               fmt.Fprintf(os.Stderr, "word %s is odd\n", w)
          }
     }
}
```

要查看我们的程序运行效果，我们应该传递参数，如下一个示例所示：

```go
go run main.go alex golang error
```

要查看哪些单词被打印到`stdout`（标准输出）和哪些被打印到`stderr`（标准错误），您可以在终端中使用重定向：

```go
go run main.go word1 word2 word3 > stdout.txt 2> stderr.txt
```

在运行前面的命令后，您可以检查`stdout.txt`和`stderr.txt`的内容，以查看哪些单词被打印到每个流：

```go
cat stdout.txt
cat stderr.txt
```

长度为偶数的单词将位于`stdout.txt`中，而长度为奇数的单词将位于`stderr.txt`中。

## 重定向和标准流

记住`stdout`是文件描述符 1，而`stderr`是文件描述符 2 吗？现在，它们将结合在一起。

当我们使用`> stdout.txt`时，我们使用 shell 重定向运算符。它将命令的标准输出（`stdout`）重定向到运算符左侧的文件。由于`stdout`是标准输出，通常省略数字 1，但`2>`不是这样。它专门重定向标准错误（`stderr`）。

注意

`stdout.txt`和`stderr.txt`文件是`go run`命令的标准输出和标准错误将被写入的地方。如果这些文件中的任何一个不存在，它将被创建；如果它已经存在，它将被覆盖。

## 使其可测试

我们不希望在终端中执行程序以确保程序在每次小变化后仍然工作。在这方面，我们想要添加自动化测试。让我们重构代码以编写测试。

### 移动核心理念

将检查单词长度和打印结果的核心理念移动到名为`app`的单独函数中。这使得代码更加组织化，并且更容易测试：

```go
func app(words []string) {
     for _, w := range words {
          if len(w)%2 == 0 {
               fmt.Fprintf(os.Stdout, "word %s is even\n", w)
          } else {
               fmt.Fprintf(os.Stderr, "word %s is odd\n", w)
          }
     }
}
```

### 引入灵活的配置

添加一个`CliConfig`结构体来保存 CLI 的配置值。这为未来的修改提供了灵活性。目前，我们感兴趣的是使标准流易于更改以进行测试：

```go
type CliConfig struct {
     ErrStream, OutStream io.Writer
}
func app(words []string, cfg CliConfig) {
     for _, w := range words {
          if len(w)%2 == 0 {
               fmt.Fprintf(cfg.OutStream, "word %s is even\n", w)
          } else {
               fmt.Fprintf(cfg.ErrStream, "word %s is odd\n", w)
          }
     }
}
```

### 功能选项

功能选项是 Go 中的一个设计模式，它允许灵活且干净地配置对象。当对象有许多可选配置时，它特别有用。

这种模式提供了几个好处：

+   可读性：无需记住参数的顺序，就可以清楚地知道正在设置哪些选项。

+   可扩展性：你可以轻松地添加新选项，而无需更改现有的函数签名或调用。

+   安全性：你可以确保对象在构造后始终处于有效状态。你可以在构造函数中轻松提供默认值。如果没有提供选项，则使用默认值。

在我们的程序中，我们有两个可选配置：`outStream`和`errStream`。

你可以使用功能选项而不是使用带有多个参数的构造函数或配置结构体：

```go
type Option func(*CliConfig) error
func WithErrStream(errStream io.Writer) Option {
     return func(c *CliConfig) error {
          c.ErrStream = errStream
          return nil
     }
}
func WithOutStream(outStream io.Writer) Option {
     return func(c *CliConfig) error {
          c.OutStream = outStream
          return nil
     }
}
```

现在，你可以为`CliConfig`结构体提供一个接受这些选项的构造函数：

```go
func NewCliConfig(opts ...Option) (CliConfig, error) {
     c := CliConfig{
          ErrStream: os.Stderr,
          OutStream: os.Stdout,
     }
     for _, opt := range opts {
          if err := opt(&c); err != nil {
               return CliConfig{}, err
          }
     }
     return c, nil
}
```

在前面的设置中，创建新的`CliConfig`结构体变得直观且易于阅读：

```go
NewCliConfig(WithOutStream(&var1),WithErrStream(&var2))
NewCliConfig(WithOutStream(&var1))
NewCliConfig(WithErrStream(&var2))
```

### 更新主函数

我们可以修改`main`函数以使用新的`CliConfig`结构体和`app`函数，并处理`NewCliConfig`的潜在错误：

```go
func main() {
     words := os.Args[1:]
     if len(words) == 0 {
          fmt.Fprintln(os.Stderr, "No words provided.")
          os.Exit(1)
     }
     cfg, err := NewCliConfig()
     if err != nil {
          fmt.Fprintf(os.Stderr, "Error creating config: %v\n", err)
          os.Exit(1)
     }
     app(words, cfg)
}
```

### 测试

让我们来看看我们的测试函数，并检查我们用它实现了什么：

```go
package main
import (
    "bytes"
    "strings"
    "testing"
)
func TestMainProgram(t *testing.T) {
    var stdoutBuf, stderrBuf bytes.Buffer
    config, err := NewCliConfig(WithOutStream(&stdoutBuf), WithErrStream(&stderrBuf))
    if err != nil {
        t.Fatal("Error creating config:", err)
    }
    app([]string{"main", "alex", "golang", "error"}, config)
    output := stdoutBuf.String()
    if len(output) == 0 {
        t.Fatal("Expected output, got nothing")
    }
    if !strings.Contains(output, "word alex is even") {
        t.Fatal("Expected output does not contain 'word alex is even'")
    }
    if !strings.Contains(output, "word golang is even") {
        t.Fatal("Expected output does not contain 'word golang is even'")
    }
    errors := stderrBuf.String()
    if len(errors) == 0 {
        t.Fatal("Expected errors, got nothing")
    }
    if !strings.Contains(errors, "word error is odd") {
        t.Fatal("Expected errors does not contain 'word error is odd'")
    }
}
```

让我们分解这个测试的关键组件和步骤：

1.  `TestMainProgram`函数是检查`app`函数行为的测试函数。

1.  创建了两个 `bytes.Buffer` 变量，`stdoutBuf` 和 `stderrBuf`。这些缓冲区将分别捕获程序的标准输出和标准错误流。这允许你在测试中捕获和检查程序的输出和错误消息。

1.  调用 `NewCliConfig` 函数创建一个具有自定义输出和错误流的 `CliConfig` 配置。使用 `WithOutStream` 和 `WithErrStream` 选项将输出和错误流分别设置为 `stdoutBuf` 和 `stderrBuf` 缓冲区。这样做是为了捕获程序的输出和错误，并在测试中进行检查。

1.  `app` 函数被调用，并传入一个单词列表作为输入，同时提供自定义的 `CliConfig` 结构体作为配置。在这种情况下，单词 `"main"`、`"alex"`、`"golang"` 和 `"error"` 作为参数传递，以模拟程序的行为。

测试随后检查程序输出和错误的各个方面：

1.  它检查 `stdoutBuf` 中是否有捕获的输出。如果没有输出，则测试失败。

1.  它检查预期的输出消息，例如 `"word alex is even"` 和 `"word golang is even"`，是否包含在捕获的输出中。如果任何预期的输出消息缺失，则测试失败。

1.  它检查 `stderrBuf` 中是否有捕获的错误。如果没有错误，则测试失败。

1.  它检查预期的错误消息，即 `"word error is odd"`，是否包含在捕获的错误中。如果预期的错误消息缺失，则测试失败。

我们可以使用 `go test` 命令运行测试，并将显示类似的输出：

```go
=== RUN   TestMainProgram
--- PASS: TestMainProgram (0.00s)
PASS
```

总结来说，这个单元测试验证了当给定的单词集合特定时，`app` 函数是否正确地产生了预期的输出和错误消息。它使用 `bytes.Buffer` 捕获程序的输出和错误，检查预期消息的存在，并在预期输出或错误消息缺失时报告测试失败。这个测试有助于确保 `app` 函数在不同场景下的行为符合预期，避免了使用终端进行手动测试。

我们现在可以使用我们的程序与其他 Linux 工具一起使用：

```go
go build -o cli-app main.go
ls -l | xargs app | grep even
```

最后一条命令列出当前目录的内容，将列表中的每一行作为 `app` 命令的参数传递，然后过滤 `app` 命令的输出，只显示包含单词 “`even`” 的行。

在我们继续前进之前，有一个关于本章关键概念的摘要将是有帮助的。

# 摘要

从旅行的类比中，我们看到了系统调用如何充当护照，使进程能够在软件执行的广阔领域中导航。我们区分了用户模式和内核模式，强调了与每种模式相关的特权和限制。本章还揭示了 Go 中`syscall`包面临的挑战，导致其最终被更易于维护的`x/sys`包所取代。此外，在本章中，我们成功构建了一个 CLI 应用程序，利用了 Go 的`os`和`x/sys`包的力量。我们亲眼见证了系统调用如何被集成到实际软件解决方案中，从而实现与操作系统的直接交互。随着你继续前进，请记住所强调的最佳实践和所获得的技能，确保在 Go 中进行安全、高效的系统级编程和健壮的 CLI 工具创建。

在下一章中，我们将探索 Go 中的文件和目录操作的世界，这对于任何与文件系统打交道的开发者来说都是一项至关重要的技能集。我们的主要重点是识别不安全的权限、确定目录大小和定位重复文件。对于所有与文件系统打交道的开发者来说，这些技术具有重大意义，因为它们在维护数据完整性和保障软件应用中的安全性方面发挥着至关重要的作用。
