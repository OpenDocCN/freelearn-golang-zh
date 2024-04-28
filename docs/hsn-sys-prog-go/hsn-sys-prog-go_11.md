# 第八章：退出代码、信号和管道

本章将继续上一章，并演示父子进程之间的通信。特别是，本章将向您展示如何通过正确使用退出代码、自定义信号处理和连接进程与管道来管理通信。这些通信形式将用于使我们的应用程序能够有效地与操作系统和其他进程进行通信。

本章将涵盖以下主题：

+   返回退出代码

+   读取退出代码

+   拦截信号

+   发送信号

+   使用管道

+   使用其他流工具

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，您可以参考第三章，*Go 概述*。

# 使用退出代码

退出代码，或退出状态，是进程在退出时传递给其父进程的一个小整数。这是通知您应用程序执行结果的最简单方式。在第二章，*Unix 操作系统组件*中，我们简要提到了退出代码。现在我们将学习如何在应用程序中使用它们以及如何解释子进程的退出代码。

# 发送退出代码

退出代码是进程在终止后通知其父进程其状态的方式。为了从当前进程返回任何退出状态，有一个函数可以直接完成工作：`os.Exit`。

此函数接受一个参数，即整数，并表示将返回给父进程的退出代码。可以使用一个简单的程序进行验证，如下面的代码所示：

```go
package main

import (
   "fmt"
    "os"
)

func main() {
    fmt.Println("Hello, playground")
    os.Exit(1)
}
```

完整示例可在[`play.golang.org/p/-6GIY7EaVD_V`](https://play.golang.org/p/-6GIY7EaVD_V)找到。

当应用程序成功执行时，使用退出代码`0`。任何其他退出代码都表示在执行过程中可能发生的某种错误。当主函数完成时，它返回`0`；当恐慌未被恢复时，它返回`2`。

# Bash 中的退出代码

每次在 shell 中执行命令时，生成的退出代码都会存储在一个变量中。执行的最后一个命令的状态存储在`$?`变量中，可以如下打印：

```go
> echo  $? # will print 1

```

重要的是要注意，退出代码仅在使用`go build`或`go install`获得的二进制文件运行时才有效。如果使用`go run`，则对于任何不是`0`的代码，它将返回`1`。

# 退出值位大小

退出状态是一个 8 位整数；这意味着即使 Go 函数的参数是整数，返回的状态也将是传递值和`256`之间的模运算的结果。

让我们看看以下程序：

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    fmt.Println("Hello, playground")
    os.Exit(-1)
}
```

完整示例可在[`play.golang.org/p/vzwI1kDiGrP`](https://play.golang.org/p/vzwI1kDiGrP)找到。

即使函数参数为`-1`，这将具有退出状态`255`，因为`(-1)%256=255`。这是因为退出代码是一个 8 位数字（`0`、`255`）。

# 退出和延迟函数

关于此函数使用的一个重要注意事项是延迟函数不会被执行。

以下示例将没有输出：

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    defer fmt.Println("Hello, playground")
    os.Exit(0)
}
```

完整示例可在[`play.golang.org/p/2zbczc_ckgb`](https://play.golang.org/p/2zbczc_ckgb)找到。

# 恐慌和退出代码

如果应用程序因未恢复的恐慌而终止，则延迟函数将被执行，但退出代码将为`2`：

```go
package main

import (
    "fmt"
)

func main() {
    defer fmt.Println("Hello, playground")
    panic("panic")
}
```

完整示例可在[`play.golang.org/p/mjOMb0KsM3e`](https://play.golang.org/p/mjOMb0KsM3e)找到。

# 退出代码和 goroutines

如果`os.Exit`函数发生在 goroutine 中，所有 goroutine（包括主 goroutine）将立即终止，而不执行任何延迟调用，如下所示：

```go
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    go func() {
        defer fmt.Println("go end (deferred)")
        fmt.Println("go start")
        os.Exit(1)
    }()
    fmt.Println("main end (deferred)")
    fmt.Println("main start")
    time.Sleep(time.Second)
    fmt.Println("main end")
}
```

完整的示例可在[`play.golang.org/p/JVEB5MTcEoa`](https://play.golang.org/p/JVEB5MTcEoa)找到。

使用`os.Exit`时需要小心，因为所有延迟操作都不会被执行，这可能导致资源泄漏或错误，比如不刷新缓冲区和未将所有内容写入文件。

# 读取子进程退出码

我们在上一章中探讨了如何创建子进程。Go 使您可以轻松检查子进程的退出码，但这并不简单，因为`exec.Cmd`结构中有一个`os.ProcessState`属性的字段。

`os.ProcessState`属性有一个`Sys`方法，返回一个接口。在 Unix 中，它的值是一个`syscall.WaitStatus`结构，可以使用`ExitCode`方法访问退出码。下面的代码演示了这一点：

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func exitStatus(state *os.ProcessState) int {
    status, ok := state.Sys().(syscall.WaitStatus)
    if !ok {
        return -1
    }
    return status.ExitStatus()
}

func main() {
    cmd := exec.Command("ls", "__a__")
    if err := cmd.Run(); err != nil {
        if status := exitStatus(cmd.ProcessState); status == -1 {
            fmt.Println(err)
        } else {
            fmt.Println("Status:", status)
        }
    }
}
```

如果无法访问命令变量，则返回的错误是`exec.ExitError`，它包装了`os.ProcessState`属性，如下所示：

```go
func processState(e error) *os.ProcessState {
    err, ok := e.(*exec.ExitError)
    if !ok {
        return nil
    }
    return err.ProcessState
}
```

我们可以看到获取退出码并不简单，需要进行一些类型转换。

# 处理信号

信号是 Unix 操作系统提供的另一种进程间通信工具。它们是可以从一个进程发送到另一个进程的整数值，使我们的应用程序能够与父进程以外的更多进程通信。通过这样做，应用程序能够解释传入的信号，并且还可以向其他进程发送信号。

# 处理传入信号

Go 应用程序的正常行为是处理一些传入信号，包括`SIGHUP`，`SIGINT`和`SIGABRT`，然后终止应用程序。我们可以用自定义行为替换这个标准行为，拦截所有或部分信号并相应地处理。

# 信号包

使用`os/signal`包可以实现自定义行为，该包公开了必要的函数。

例如，如果应用程序不需要拦截信号，`signal.Ignore`函数允许将信号添加到被忽略的列表中。`signal.Ignored`函数也允许验证某个信号是否被忽略。

为了使用通道拦截信号，可以使用核心函数`signal.Notify`。这使得可以指定一个通道，并选择应该发送到该通道的信号。然后应用程序可以在任何 goroutine 中使用该通道来处理具有自定义行为的信号。请注意，如果未指定信号，则该通道将接收发送到应用程序的所有信号，如下所示：

```go
signal.Notify(ch, signalList...)
```

`signal.Stop`函数用于停止从特定通道接收信号，而`signal.Reset`函数停止拦截一个或多个信号到所有通道。为了重置所有信号，`Reset`不需要传递任何参数。

# 优雅关闭

应用程序在等待任务完成并清除所有资源后终止时执行优雅关闭。使用自定义信号处理是一个很好的实践，因为它给我们释放仍然打开的资源的时间。在关闭之前，我们可以执行任何其他应该在退出应用程序之前完成的任务；例如，保存当前状态。

现在我们知道退出码是如何工作的，我们可以介绍`log`包。从现在开始，将使用它来将语句打印到标准输出，而不是`fmt`。这使得可以执行`Print`语句和`Fatal`语句，后者相当于打印并执行`os.Exit(1)`。`log`包还允许用户定义日志标志，以打印日期、时间和/或文件/行。

我们可以从一个非常基本的例子开始，处理所有信号如下：

```go
package main

import (
    "log"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    log.Println("Start application...")
    c := make(chan os.Signal)
    signal.Notify(c)
    s := <-c
    log.Println("Exit with signal:", s)
}
```

为了测试这个应用程序，您可以使用两个不同的终端。 首先，您可以在第一个终端中启动应用程序，并使用另一个终端执行`ps`命令来查找应用程序的 PID，以便使用`kill`命令向其发送信号。

第二种方法只使用一个终端，在后台启动应用程序。 这将在屏幕上显示 PID，并将在`kill`命令中使用，如下所示：

```go
$ go build -o "signal" ch8/signal/base/base.go

$ ./signal &
[1] 265
[Log] Start application...

$ kill -6 265
[Log] Exit with signal: aborted
```

请注意，如果您使用的是 macOS，您将收到`abort trap`信号名称。

# 退出清理和资源释放

更实际和常见的干净关闭的例子是资源清理。 在使用退出语句时，延迟函数（例如`bufio.Writer`结构的`Flush`）不会被执行。 这可能会导致信息丢失，如下例所示：

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "os"
    "time"
)

func main() {
    f, err := os.OpenFile("file.txt", os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    w := bufio.NewWriter(f)
    defer w.Flush()
    for i := 0; i < 3; i++ {
        fmt.Fprintln(w, "hello")
        log.Println(i)
        time.Sleep(time.Second)
    }
}
```

如果在应用程序完成之前向该应用程序发送了`TERM`信号，则文件将被创建和截断，但刷新将永远不会被执行，导致一个空文件。

这可能是预期的行为，但这很少发生。 最好在信号处理部分进行任何清理，如下例所示：

```go
func main() {
    c := make(chan os.Signal, syscall.SIGTERM)
    signal.Notify(c)
    f, err := os.OpenFile("file.txt", os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    w := bufio.NewWriter(f)
    go func() {
        <-c
        w.Flush()
        os.Exit(0)
    }()
    for i := 0; i < 3; i++ {
        fmt.Fprintln(w, "hello")
        log.Println(i)
        time.Sleep(time.Second)
    }
}
```

在这种情况下，我们将使用 goroutine 与信号通道结合，以在退出之前刷新写入器。 这将确保将缓冲区中写入的任何内容持久保存到文件中。

# 配置重新加载

信号不仅可以用于终止应用程序。 应用程序可以对每个信号做出不同的反应，以便可以用于执行不同的功能，从而可以控制应用程序流程。

下一个示例将在文本文件中存储一些设置。 设置将以其字符串版本存储为`time.Duration`类型。 持续时间是一个`int64`值，其字符串版本以人类可读的格式存储，例如`2m10s`，它还具有许多有用的方法。 这在`time`包的不同函数中使用。

应用程序将以取决于当前设置值的频率执行某个操作。 信号的可能操作包括以下内容：

+   `SIGHUP (1)`: 这会从设置文件中加载间隔。

+   `SIGTERM (2)`: 这会保存当前的间隔值，并退出应用程序。

+   `SIGQUIT (6)`: 这会退出而不保存。

+   `SIGUSR1 (10)`: 这会将间隔加倍。

+   `SIGUSR2 (11)`: 这会将间隔减半。

+   `SIGALRM (14)`: 这会保存当前的间隔值。

使用`signal.Notify`函数捕获这些信号，该函数用于所有不同的信号。 从通道接收到的值需要一个条件语句，即类型开关，以允许应用程序根据值执行不同的操作：

```go
func main() {
    c := make(chan os.Signal, 1)
    d := time.Second * 4
    signal.Notify(c,
        syscall.SIGHUP, syscall.SIGINT, syscall.SIGQUIT,
        syscall.SIGUSR1, syscall.SIGUSR2, syscall.SIGALRM)
    // initial load
    if err := handleSignal(syscall.SIGHUP, &d); err != nil && 
        !os.IsNotExist(err) {
            log.Fatal(err)
    }

    for {
        select {
        case s := <-c:
            if err := handleSignal(s, &d); err != nil {
                log.Printf("Error handling %s: %s", s, err)
                continue
            }
        default:
            time.Sleep(d)
            log.Println("After", d, "Executing action!")
        }
    }
}
```

`handleSignal`函数将包含信号中的`switch`语句：

```go
func handleSignal(s os.Signal, d *time.Duration) error {
    switch s {
    case syscall.SIGHUP:
        return loadSettings(d)
    case syscall.SIGALRM:
        return saveSettings(d)
    case syscall.SIGINT:
        if err := saveSettings(d); err != nil {
            log.Println("Cannot save:", err)
            os.Exit(1)
        }
        fallthrough
    case syscall.SIGQUIT:
        os.Exit(0)
    case syscall.SIGUSR1:
        changeSettings(d, (*d)*2)
        return nil
    case syscall.SIGUSR2:
        changeSettings(d, (*d)/2)
        return nil
    }
    return nil
}
```

以下描述了将在信号处理函数中实现的不同行为：

+   更改值只会使用持续指针来存储新值。

+   加载将尝试扫描文件的内容（如果存在）作为持续时间并更改设置值。

+   保存将持续时间写入文件，并使用其字符串格式。 以下代码描述了这一点：

```go

func changeSettings(d *time.Duration, v time.Duration) {
    *d = v
    log.Println("Changed", v)
}

func loadSettings(d *time.Duration) error {
    b, err := ioutil.ReadFile(cfgPath)
    if err != nil {
        return err
    }
    var v time.Duration
    if v, err = time.ParseDuration(string(b)); err != nil {
        return err
    }
    *d = v
    log.Println("Loaded", v)
    return nil
}

func saveSettings(d *time.Duration) error {
    f, err := os.OpenFile(cfgPath,   
        os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644)
            if err != nil {
                return err
            }
        defer f.Close()

    if _, err = fmt.Fprint(f, d); err != nil {
        return err
    }
    log.Println("Saved", *d)
    return nil
```

我们将在`init`函数中获取用户主目录的路径，并将其用于组成`settings`文件的路径，如下所示：

```go
var cfgPath string

func init() {
    u, err := user.Current()
    if err != nil {
        log.Fatalln("user:", err)
    }
    cfgPath = filepath.Join(u.HomeDir, ".multi")
}
```

我们可以在一个终端中启动应用程序，并使用另一个终端发送信号，如下所示：

| **终端 1** | **终端 2** |
| --- | --- |

|

```go
$ go run ch08/signal/multi/multi.go
Loaded 1s
After 1s Executing action!

Changed 2s
After 2s Executing action!

Changed 4s
After 4s Executing action!

Changed 2s
After 2s Executing action!

Saved 1s

$
```

|

```go
 $ kill -SIGUSR1 $(pgrep multi)

$ kill -SIGUSR1 $(pgrep multi)

$ kill -SIGUSR2 $(pgrep multi)

$ kill -SIGINT $(pgrep multi)

```

|

在左列中，我们可以看到应用程序的输出； 在右列中，我们可以看到我们启动的命令。 为了获取正在运行的应用程序的 PID，我们使用`pgrep`命令并嵌套在`kill`中。 

# 向其他进程发送信号

在了解了如何处理传入信号的方式之后，让我们看看如何以编程方式向其他进程发送信号。`os.Process`结构是我们唯一需要的工具——其`Signal`方法使得向项目发送信号成为可能。就是这么简单！

较不简单的部分是获取进程。有两种用例，如下：

+   进程是一个子进程，我们已经通过`os.StartProcess`或`exec.Command`结构获得了进程值。

+   进程已经存在，但我们没有它，因此需要使用其 PID 搜索它。

第一个用例更简单，因为我们已经将进程作为变量或作为`exec.Cmd`变量的属性，并且可以直接调用该方法。

另一个用例需要使用`os.FindProcess`方法通过 PID 搜索进程，如下：

```go
p, err := os.FindProcess(pid)
if err != nil {
    panic(err)
}
```

一旦我们有了`os.Process`，我们可以使用其`Signal`方法向其发送特定信号，如下：

```go
if err = p.Signal(syscall.SIGTERM); err != nil {
    panic(err)
}
```

我们将发送给进程的信号类型取决于目标进程和我们想要建议的行为，例如中断或终止。

# 连接流

在 Go 中，流是一种抽象，可以将任何类型的通信或数据流视为一系列读取器和写入器。我们已经学会了流是 Go 的重要组成部分。现在我们将学习如何使用我们已经了解的有关输入和输出的知识来控制与进程相关的流——输入、输出和错误。

# 管道

管道是连接输入和输出的同步方式之一，允许进程进行通信。

# 匿名管道

使用 shell 时，可以将不同的命令链接成一个序列，使一个命令的输出成为下一个命令的输入。例如，考虑以下命令：

```go
cat book_list.txt | grep "Game" | wc -l
```

在这里，我们正在显示一个文件，使用前面的命令来过滤包含特定字符串的行，并最终使用过滤后的输出来计算行数。

在应用程序内创建进程时，可以在 Go 中以编程方式完成此操作。

`io.Pipe`函数返回一个连接的读取器/写入器对；写入管道写入的任何内容都将被管道读取器读取。写操作是阻塞的，这意味着所有写入的数据都必须在执行新的写操作之前被读取。

我们已经看到`exec.Cmd`允许其输出和输入使用通用流，这使我们可以使用`io.Pipe`函数返回的值将一个进程连接到另一个进程。

首先，我们定义三个命令，如下：

+   `cat`索引为`0`

+   `grep`索引为`1`

+   `wc`索引为`2`

然后，我们可以定义我们需要的两个管道，如下所示：

```go
r1, w1 := io.Pipe()
r2, w2 := io.Pipe()

var cmds = []*exec.Cmd{
   exec.Command("cat", "book_list.txt"),
   exec.Command("grep", "Game"),
   exec.Command("wc", "-l"),
}
```

接下来，我们连接输入和输出流。我们连接`cat`（命令`0`）的输出和`grep`（命令`1`）的输入，然后对`grep`的输出和`wc`的输入进行相同的操作：

```go
cmds[1].Stdin, cmds[0].Stdout = r1, w1
cmds[2].Stdin, cmds[1].Stdout = r2, w2
cmds[2].Stdout = os.Stdout
```

然后，我们启动我们的命令，如下：

```go
for i := range cmds {
    if err := cmds[i].Start(); err != nil {
        log.Fatalln("Start", i, err)
    }
}
```

我们等到每个命令执行结束，然后关闭相应的管道写入器；否则，下一个命令的读取器将挂起。为了简化操作，每个管道写入器都是切片中的一个元素，并且每个写入器的索引与其链接的命令的索引相同。最后一个是`nil`，因为最后一个命令没有通过管道链接：

```go
for i, closer := range []io.Closer{w1, w2, nil} {
    if err := cmds[i].Wait(); err != nil {
        log.Fatalln("Wait", i, err)
    }
    if closer == nil {
        continue
    }
    if err := closer.Close(); err != nil {
        log.Fatalln("Close", i, err)
    }
}
```

`io`包还提供了其他工具，可以帮助简化一些操作。

# 标准输入和输出管道

`io.MultiWriter`函数使得可以将相同的内容写入多个读取器。当需要自动将命令的输出广播到一系列不同的命令时，这将非常有用。

假设我们想要做之前做过的事情（即在文件中查找单词），但是要查找不同的单词。我们可以使用`MultiWriter`函数将输出复制到一系列`grep`命令，每个命令都将连接到自己的`wc`命令。

在本例中，我们将使用`exec.Command`的两个辅助方法：

+   `Cmd.StdinPipe`：这返回一个`PipeWriter`结构，将连接到命令的标准输入。

+   `Cmd.StdoutPipe`：这返回一个`PipeReader`结构，将连接到命令的标准输出。

让我们首先定义一个搜索项列表：一个用于命令的元组（`grep`和`wc`），一个用于连接到第一个命令的写入器，一个用于每个命令链的最终输出：

```go
var (
    words = []string{"Game", "Feast", "Dragons", "of"}
    cmds = make([][2]*exec.Cmd, len(words))
    writers = make([]io.Writer, len(words))
    buffers = make([]bytes.Buffer, len(words))
    err error
)
```

现在让我们定义命令及其连接——每个`grep`命令将在一侧使用`MultiWriter`函数与`cat`连接，并在另一侧连接到`wc`命令的输入：

```go
for i := range words {
    cmds[i][0] = exec.Command("grep", words[i])
    if writers[i], err = cmds[i][0].StdinPipe(); err != nil {
        log.Fatal("in pipe", i, err)
    }
    cmds[i][1] = exec.Command("wc", "-l")
    if cmds[i][1].Stdin, err = cmds[i][0].StdoutPipe(); err != nil {
        log.Fatal("in pipe", i, err)
    }
    cmds[i][1].Stdout = &buffers[i]
}

cat := exec.Command("cat", "book_list.txt")
cat.Stdout = io.MultiWriter(writers...)
```

我们可以运行主要的`cat`命令，当它完成时，我们可以关闭第一组写入管道，这样`grep`命令就可以终止，如下所示：

```go
for i := range cmds {
    if err := writers[i].(io.Closer).Close(); err != nil {
        log.Fatalln("close 0", i, err)
    }
}

for i := range cmds {
    if err := cmds[i][0].Wait(); err != nil {
        log.Fatalln("grep wait", i, err)
    }
}
```

然后我们可以等待另一个命令完成并显示结果，如下所示：

```go
for i := range cmds {
    if err := cmds[i][1].Wait(); err != nil {
        log.Fatalln("wc wait", i, err)
    }
    count := bytes.TrimSpace(buffers[i].Bytes())
    log.Printf("%10q %s entries", cmds[i][0].Args[1], count)
}
```

请注意，当使用`StdinPipe`方法时，生成的写入器必须关闭，但使用`StdoutPipe`方法则不需要。

# 总结

在本章中，我们学习了如何使用三个主要功能处理进程之间的通信：退出代码、信号和管道。

退出代码是 0 到 255 之间的 8 位值，由进程返回给其父进程。退出代码为`0`表示应用程序执行成功。在 Go 中很容易返回退出代码，但使用`os.Exit`函数会忽略延迟函数的执行。当发生 panic 时，所有延迟函数都会执行，返回的代码是`2`。从子进程获取退出代码相对复杂，因为它取决于操作系统；然而，在 Unix 系统中，可以使用一系列类型断言来实现。

信号用于与任何进程进行通信。它们是 6 位值，介于 1 和 64 之间，通过系统调用从一个进程发送到另一个进程。可以使用通道和`signal.Notify`函数来接收信号。使用`Process.Signal`方法很容易发送信号。

管道是一组同步连接的输入和输出流。它们用于将一个进程的输入连接到另一个进程的输出。我们看到了如何连接多个命令，就像终端一样，并学习了如何使用`io.MultiReader`将一个命令的输出广播到多个命令。

在下一章中，我们将深入研究网络编程，从 TCP 一直到 HTTP 服务器。

# 问题

1.  退出代码是什么？谁会使用它？

1.  当应用程序发生 panic 时会发生什么？返回哪个退出代码？

1.  当接收到所有信号时，Go 应用程序的默认行为是什么？

1.  如何拦截信号并决定应用程序的行为？

1.  你能向其他进程发送信号吗？如果可以，怎么做？

1.  管道是什么，为什么重要？
