# 第七章：处理进程和守护进程

本章将介绍如何使用 Go 标准库处理当前进程的属性，以及如何更改它们。我们还将重点介绍如何创建子进程，并概述`os/exec`包。

最后，我们将解释守护进程是什么，它们具有什么属性，以及如何使用标准库创建它们。

本章将涵盖以下主题：

+   理解进程

+   子进程

+   从守护进程开始

+   创建服务

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，您可以参考第三章，*Go 概述*。

# 理解进程

我们已经看到了 Unix 操作系统中进程的重要性，现在我们将看看如何获取有关当前进程的信息以及如何创建和处理子进程。

# 当前进程

Go 标准库允许我们获取有关当前进程的信息。这是通过使用`os`包中提供的一系列函数来完成的。

# 标准输入

程序可能想要知道的第一件事是它的标识符和父标识符，即 PID 和 PPID。这实际上非常简单 - `os.Getpid()`和`os.Getppid()`函数都返回一个整数值，其中包含这两个标识符，如下面的代码所示：

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    fmt.Println("Current PID:", os.Getpid())
    fmt.Println("Current Parent PID:", os.Getppid())
}
```

完整示例可在[`play.golang.org/p/ng0m9y4LcD5`](https://play.golang.org/p/ng0m9y4LcD5)找到。

# 用户和组 ID

另一个有用的信息是当前用户和进程所属的组。一个典型的用例可能是将它们与特定文件的权限进行比较。

`os`包提供以下功能：

+   `os.Getuid()`: 返回进程所有者的用户 ID

+   `os.Getgid()`: 返回进程所有者的组 ID

+   `os.Getgroups()`: 返回进程所有者的附加组 ID

我们可以看到这三个函数返回它们的数字形式的 ID：

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    fmt.Println("User ID:", os.Getuid())
    fmt.Println("Group ID:", os.Getgid())
    groups, err := os.Getgroups()
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("Group IDs:", groups)
}
```

完整示例可在[`play.golang.org/p/EqmonEEc_ZI`](https://play.golang.org/p/EqmonEEc_ZI)找到。

为了获取用户和组的名称，`os/user`包中有一些辅助函数。这些函数（名称相当自明）如下：

+   `func LookupGroupId(gid string) (*Group, error)`

+   `func LookupId(uid string) (*User, error)`

即使用户 ID 是一个整数，它需要一个字符串作为参数，因此需要进行转换。最简单的方法是使用`strconv`包，它提供了一系列实用程序，用于将字符串转换为其他基本数据类型，反之亦然。

我们可以在以下示例中看到它们的作用：

```go
package main

import (
    "fmt"
    "os"
    "os/user"
    "strconv"
)

func main() {
    uid := os.Getuid()
    u, err := user.LookupId(strconv.Itoa(uid))
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("User: %s (uid %d)\n", u.Username, uid)
    gid := os.Getgid()
    group, err := user.LookupGroupId(strconv.Itoa(gid))
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("Group: %s (uid %d)\n", group.Name, uid)
}
```

完整示例可在[`play.golang.org/p/C6EWF2c50DT`](https://play.golang.org/p/C6EWF2c50DT)找到。

# 工作目录

进程可以提供给我们的另一个非常有用的信息是工作目录，以便我们可以更改它。在第四章，*与文件系统一起工作*中，我们了解了可以使用的工具 - `os.Getwd`和`os.Chdir`。

在以下实际示例中，我们将看看如何使用这些函数来操作工作目录：

1.  首先，我们将获取当前工作目录，并使用它获取二进制文件的路径。

1.  然后，我们将工作目录与另一个路径连接起来，并使用它创建一个目录。

1.  最后，我们将使用刚创建的目录的路径来更改当前工作目录。

查看以下代码：

```go
// obtain working directory
wd, err := os.Getwd()
if err != nil {
    fmt.Println("Error:", err)
    return
}
fmt.Println("Working Directory:", wd)
fmt.Println("Application:", filepath.Join(wd, os.Args[0]))

// create a new directory
d := filepath.Join(wd, "test")
if err := os.Mkdir(d, 0755); err != nil {
    fmt.Println("Error:", err)
    return
}
fmt.Println("Created", d)

// change the current directory
if err := os.Chdir(d); err != nil {
    fmt.Println("Error:", err)
    return
}
fmt.Println("New Working Directory:", d)
```

完整示例可在[`play.golang.org/p/UXAer5nGBtm`](https://play.golang.org/p/UXAer5nGBtm)找到。

# 子进程

Go 应用程序可以与操作系统交互，创建其他进程。`os`的另一个子包提供了创建和运行新进程的功能。在`os/exec`包中，有一个`Cmd`类型，表示命令执行：

```go
type Cmd struct {
    Path string // command to run.
    Args []string // command line arguments (including command)
    Env []string // environment of the process
    Dir string // working directory
    Stdin io.Reader // standard input`
    Stdout io.Writer // standard output
    Stderr io.Writer // standard error
    ExtraFiles []*os.File // additional open files
    SysProcAttr *syscall.SysProcAttr // os specific attributes
    Process *os.Process // underlying process
    ProcessState *os.ProcessState // information on exited processte
}
```

创建新命令的最简单方法是使用`exec.Command`函数，它接受可执行路径和一系列参数。让我们看一个简单的例子，使用`echo`命令和一些参数：

```go
package main

import (
    "fmt"
    "os/exec"
)

func main() {
    cmd := exec.Command("echo", "A", "sample", "command")
    fmt.Println(cmd.Path, cmd.Args[1:]) // echo [A sample command]
}
```

完整的示例可在[`play.golang.org/p/dBIAUteJbxI`](https://play.golang.org/p/dBIAUteJbxI)找到。

一个非常重要的细节是标准输入、输出和错误的性质-它们都是我们已经熟悉的接口：

+   输入是一个`io.Reader`，可以是`bytes.Reader`、`bytes.Buffer`、`strings.Reader`、`os.File`或任何其他实现。

+   输出和错误都是`io.Writer`，也可以是`os.File`或`bytes.Buffer`，也可以是`strings.Builder`或任何其他的写入器实现。

根据父应用程序的需求，有不同的启动进程的方式：

+   `Cmd.Run`：执行命令，并返回一个错误，如果子进程正确执行，则为`nil`。

+   `Cmd.Start`：异步执行命令，并让父进程继续其流程。为了等待子进程完成执行，还有另一种方法`Cmd.Wait`。

+   `Cmd.Output`：执行命令并返回其标准输出，如果`Stderr`未定义但标准错误产生了输出，则返回错误。

+   `Cmd.CombinedOutput`：执行命令并返回标准错误和输出的组合，当需要检查或保存子进程的整个输出-标准输出加标准错误时非常有用。

# 访问子属性

一旦命令开始执行，同步或异步，底层的`os.Process`就会被填充，可以看到它的 PID，就像下面的例子中所示的那样：

```go
package main

import (
    "fmt"
    "os/exec"
)

func main() {
    cmd := exec.Command("ls", "-l")
    if err := cmd.Start(); err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("Cmd: ", cmd.Args[0])
    fmt.Println("Args:", cmd.Args[1:])
    fmt.Println("PID: ", cmd.Process.Pid)
    cmd.Wait()
}
```

# 标准输入

标准输入可以用来从应用程序向子进程发送一些数据。可以使用缓冲区来存储数据，并让命令读取它，就像下面的例子中所示的那样：

```go
package main

import (
    "bytes"
    "fmt"
    "os"
    "os/exec"
)

func main() {
    b := bytes.NewBuffer(nil)
    cmd := exec.Command("cat")
    cmd.Stdin = b
    cmd.Stdout = os.Stdout
    fmt.Fprintf(b, "Hello World! I'm using this memory address: %p", b)
    if err := cmd.Start(); err != nil {
        fmt.Println(err)
        return
    }
    cmd.Wait()
}
```

# 从守护进程开始

在 Unix 中，所有在后台运行的程序都被称为**守护进程**。它们通常以字母*d*结尾，比如`sshd`或`syslogd`，并提供操作系统的许多功能。

# 操作系统支持

在 macOS、Unix 和 Linux 中，如果一个进程在其父进程生命周期结束后仍然存在，那么它就是一个守护进程，这是因为父进程终止执行后，子进程的父进程会变成`init`进程，一个没有父进程的特殊守护进程，PID 为 1，它随着操作系统的启动和终止而启动和终止。在进一步讨论之前，让我们介绍两个非常重要的概念- *会话* 和 *进程组*：

+   进程组是一组共享信号处理的进程。该组的第一个进程称为**组长**。有一个 Unix 系统调用`setpgid`，可以改变进程的组，但有一些限制。进程可以在`exec`系统调用执行之前改变自己的进程组，或者改变其一个子进程的组。当进程组改变时，会话组也需要相应地改变，目标组的领导者也是如此。

+   会话是一组进程组，允许我们对进程组和其他操作施加一系列限制。会话不允许进程组迁移到另一个会话，并且阻止进程在不同会话中创建进程组。`setsid`系统调用允许我们改变进程会话到一个新的会话，如果进程不是进程组领导者。此外，第一个进程组 ID 设置为会话 ID。如果这个 ID 与正在运行的进程的 ID 相同，那么该进程被称为**会话领导者**。

现在我们已经解释了这两个属性，我们可以看看创建守护进程所需的标准操作，通常包括以下操作：

+   清理环境以删除不必要的变量。

+   创建一个 fork，以便主进程可以正常终止进程。

+   使用`setsid`系统调用，完成以下三个步骤：

1.  从 fork 的进程中删除 PPID，以便它被`init`进程接管

1.  为 fork 创建一个新的会话，这将成为会话领导者

1.  将进程设置为组领导者

+   fork 的当前目录设置为根目录，以避免使用其他目录，并且父进程打开的所有文件都被关闭（如果需要，子进程将打开它们）。

+   将标准输入设置为`/dev/null`，并使用一些日志文件作为标准输出和错误。

+   可选地，fork 可以再次 fork，然后退出。第一个 fork 将成为组领导者，第二个将具有相同的组，允许我们有另一个不是组领导者的 fork。

这对基于 Unix 的操作系统有效，尽管 Windows 也支持永久后台进程，称为**服务**。服务可以在启动时自动启动，也可以使用名为**服务控制管理器**（**SCM**）的可视应用程序手动启动和停止。它们还可以通过常规提示中的`sc`命令以及 PowerShell 中的`Start-Service`和`Stop-Service` cmdlet 来进行控制。

# 守护进程的操作

现在我们了解了守护进程是什么以及它是如何工作的，我们可以尝试使用 Go 标准库来创建一个。Go 应用程序是多线程的，不允许直接调用`fork`系统调用。

我们已经学会了`os/exec`包中的`Cmd.Start`方法允许我们异步启动一个进程。第二步是使用`release`方法关闭当前进程的所有资源。

以下示例向我们展示了如何做到这一点：

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "time"
)

var pid = os.Getpid()

func main() {
    fmt.Printf("[%d] Start\n", pid)
    fmt.Printf("[%d] PPID: %d\n", pid, os.Getppid())
    defer fmt.Printf("[%d] Exit\n\n", pid)
    if len(os.Args) != 1 {
        runDaemon()
        return
    }
    if err := forkProcess(); err != nil {
        fmt.Printf("[%d] Fork error: %s\n", pid, err)
        return
    }
    if err := releaseResources(); err != nil {
        fmt.Printf("[%d] Release error: %s\n", pid, err)
        return
    }
}
```

让我们看看`forkProcess`函数的作用，创建另一个进程，并启动它：

1.  首先，进程的工作目录被设置为根目录，并且输出和错误流被设置为标准流：

```go
func forkProcess() error {
    cmd := exec.Command(os.Args[0], "daemon")
    cmd.Stdout, cmd.Stderr, cmd.Dir = os.Stdout, os.Stderr, "/"
    return cmd.Start()
}
```

1.  然后，我们可以释放资源 - 首先，我们需要找到当前进程。然后，我们可以调用`os.Process`方法`Release`，以确保主进程释放其资源：

```go
func releaseResources() error {
    p, err := os.FindProcess(pid)
    if err != nil {
        return err
    }
    return p.Release()
}
```

1.  `main`函数将包含守护逻辑，在这个例子中非常简单 - 它将每隔几秒打印正在运行的内容。

```go
func runDaemon() {
    for {
        fmt.Printf("[%d] Daemon mode\n", pid)
        time.Sleep(time.Second * 10)
    }
}
```

# 服务

我们已经看到了从引导到操作系统关闭的第一个进程被称为`init`或`init.d`，因为它是一个守护进程。这个进程负责处理其他守护进程，并将其配置存储在`/etc/init.d`目录中。

每个 Linux 发行版都使用自己的守护进程控制过程版本，例如 Chrome OS 中的`upstart`或 Arch Linux 中的`systemd`。它们都有相同的目的并且行为类似。

每个守护进程都有一个控制脚本或应用程序，驻留在`/etc/init.d`中，并且应该能够解释一系列命令作为第一个参数，例如`status`，`start`，`stop`和`restart`。在大多数情况下，`init.d`文件是一个脚本，根据参数执行开关并相应地行为。

# 创建一个服务

一些应用程序能够自动处理它们的服务文件，这就是我们将逐步尝试实现的内容。让我们从一个`init.d`脚本开始：

```go
#!/bin/sh

"/path/to/mydaemon" $1
```

这是一个将第一个参数传递给守护程序的示例脚本。二进制文件的路径将取决于文件的位置。这需要在运行时定义：

```go
// ErrSudo is an error that suggest to execute the command as super user
// It will be used with the functions that fail because of permissions
var ErrSudo error

var (
    bin string
    cmd string
)

func init() {
    p, err := filepath.Abs(filepath.Dir(os.Args[0]))
    if err != nil {
        panic(err)
    }
    bin = p
    if len(os.Args) != 1 {
        cmd = os.Args[1]
    }
    ErrSudo = fmt.Errorf("try `sudo %s %s`", bin, cmd)
}
```

`main`函数将处理不同的命令，如下所示：

```go
func main() {
    var err error
    switch cmd {
    case "run":
        err = runApp()
    case "install":
        err = installApp()
    case "uninstall":
        err = uninstallApp()
    case "status":
        err = statusApp()
    case "start":
        err = startApp()
    case "stop":
        err = stopApp()
    default:
        helpApp()
    }
    if err != nil {
        fmt.Println(cmd, "error:", err)
    }
}
```

我们如何确保我们的应用程序正在运行？一个非常可靠的策略是使用`PID`文件，这是一个包含正在运行进程的当前 PID 的文本文件。让我们定义一些辅助函数来实现这一点：

```go
const (
    varDir = "/var/mydaemon/"
    pidFile = "mydaemon.pid"
)

func writePid(pid int) (err error) {
    f, err := os.OpenFile(filepath.Join(varDir, pidFile), os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer f.Close()
    if _, err = fmt.Fprintf(f, "%d", pid); err != nil {
        return err
    }
    return nil
}

func getPid() (pid int, err error) {
    b, err := ioutil.ReadFile(filepath.Join(varDir, pidFile))
    if err != nil {
        return 0, err
    }
    if pid, err = strconv.Atoi(string(b)); err != nil {
        return 0, fmt.Errorf("Invalid PID value: %s", string(b))
    }
    return pid, nil
}
```

`install`和`uninstall`函数将负责添加或删除位于`/etc/init.d/mydaemon`的服务文件，并要求我们以 root 权限启动应用程序，因为文件的位置：

```go
const initdFile = "/etc/init.d/mydaemon"

func installApp() error {
    _, err := os.Stat(initdFile)
    if err == nil {
        return errors.New("Already installed")
    }
    f, err := os.OpenFile(initdFile, os.O_CREATE|os.O_WRONLY, 0755)
    if err != nil {
        if !os.IsPermission(err) {
            return err
        }
        return ErrSudo
    }
    defer f.Close()
    if _, err = fmt.Fprintf(f, "#!/bin/sh\n\"%s\" $1", bin); err != nil {
        return err
    }
    fmt.Println("Daemon", bin, "installed")
    return nil
}

func uninstallApp() error {
    _, err := os.Stat(initdFile)
    if err != nil && os.IsNotExist(err) {
        return errors.New("not installed")
    }
    if err = os.Remove(initdFile); err != nil {
        if err != nil {
            if !os.IsPermission(err) {
                return err
            }
       return ErrSudo
        }
    }
    fmt.Println("Daemon", bin, "removed")
    return err
}
```

创建文件后，我们可以使用`mydaemon install`命令将应用程序安装为服务，并使用`mydaemon uninstall`命令将其删除。

守护程序安装完成后，我们可以使用`sudo service mydaemon [start|stop|status]`来控制守护程序。现在，我们只需要实现这些操作：

+   `status`将查找`pid`文件，读取它，并向进程发送信号以检查它是否正在运行。

+   `start`将使用`run`命令运行应用程序，并写入`pid`文件。

+   `stop`将获取`pid`文件，找到进程，杀死它，然后删除`pid`文件。

让我们看看`status`命令是如何实现的。请注意，在 Unix 中不存在`0`信号，并且不会触发操作系统或应用程序的操作，但如果进程没有运行，操作将失败。这告诉我们进程是否存活：

```go
func statusApp() (err error) {
    var pid int
    defer func() {
        if pid == 0 {
            fmt.Println("status: not active")
            return
        }
        fmt.Println("status: active - pid", pid)
    }()
    pid, err = getPid()
    if err != nil {
        if os.IsNotExist(err) {
            return nil
        }
        return err
    }
    p, err := os.FindProcess(pid)
    if err != nil {
        return nil
    }
    if err = p.Signal(syscall.Signal(0)); err != nil {
        fmt.Println(pid, "not found - removing PID file...")
        os.Remove(filepath.Join(varDir, pidFile))
        pid = 0
    }
    return nil
}
```

在`start`命令中，我们将按照*操作系统支持*部分中介绍的步骤创建守护程序：

1.  使用文件进行标准输出和输入

1.  将工作目录设置为根目录

1.  异步启动命令

除了这些操作，`start`命令还将进程的 PID 值保存在特定文件中，用于查看进程是否存活：

```go
func startApp() (err error) {
    const perm = os.O_CREATE | os.O_APPEND | os.O_WRONLY
    if err = os.MkdirAll(varDir, 0755); err != nil {
        if !os.IsPermission(err) {
            return err
        }
        return ErrSudo
    }
    cmd := exec.Command(bin, "run")
    cmd.Stdout, err = os.OpenFile(filepath.Join(varDir, outFile),  
        perm, 0644)
            if err != nil {
                 return err
            }
    cmd.Stderr, err = os.OpenFile(filepath.Join(varDir, errFile), 
        perm, 0644)
            if err != nil {
                return err
           }
    cmd.Dir = "/"
    if err = cmd.Start(); err != nil {
        return err
    }
    if err := writePid(cmd.Process.Pid); err != nil {
        if err := cmd.Process.Kill(); err != nil {
            fmt.Println("Cannot kill process", cmd.Process.Pid, err)
        }
        return err
    }
    fmt.Println("Started with PID", cmd.Process.Pid)
    return nil
}
```

最后，`stopApp`将终止由 PID 文件标识的进程（如果存在）：

```go
func stopApp() (err error) {
    pid, err := getPid()
    if err != nil {
        if os.IsNotExist(err) {
            return nil
        }
        return err
    }
    p, err := os.FindProcess(pid)
    if err != nil {
        return nil
    }
    if err = p.Signal(os.Kill); err != nil {
        return err
    }
    if err := os.Remove(filepath.Join(varDir, pidFile)); err != nil {
        return err
    }
    fmt.Println("Stopped PID", pid)
    return nil
}
```

现在，应用程序控制所需的所有部分都已经准备就绪，唯一缺少的是主应用程序部分，它应该是一个循环，以便守护程序保持活动状态：

```go
func runApp() error {
    fmt.Println("RUN")
    for {
        time.Sleep(time.Second)
    }
    return nil
}
```

在这个例子中，它只是在循环迭代之间固定时间睡眠。这通常是在主循环中一个好主意，因为一个空的`for`循环会无缘无故地使用大量资源。假设你的应用程序在`for`循环中检查某个条件。如果满足条件，不断检查这个条件会消耗大量资源。添加几毫秒的空闲睡眠可以帮助减少 90-95%的空闲 CPU 消耗，因此在设计守护程序时请记住这一点！

# 第三方包

到目前为止，我们已经看到了如何使用`init.d`服务从头开始实现守护程序。我们的实现非常简单和有限。它可以改进，但已经有许多包提供了相同的功能。它们支持不同的提供者，如`init.d`和`systemd`，其中一些还可以在 Windows 等非 Unix 操作系统上工作。

其中一个更有名的包（在 GitHub 上有 1000 多个星）是`kardianos/service`，它支持所有主要平台 - Linux、macOS 和 Windows。

它定义了一个表示守护程序的主接口，并具有两种方法 - 一种用于启动守护程序，另一种用于停止它。两者都是非阻塞的：

```go
type Interface interface {
    // Start provides a place to initiate the service. The service doesn't not
    // signal a completed start until after this function returns, so the
    // Start function must not take more than a few seconds at most.
    Start(s Service) error

    // Stop provides a place to clean up program execution before it is terminated.
    // It should not take more than a few seconds to execute.
    // Stop should not call os.Exit directly in the function.
    Stop(s Service) error
}
```

该包已经提供了一些用例，从简单到更复杂的用例，在示例（[`github.com/kardianos/service/tree/master/example`](https://github.com/kardianos/service/tree/master/example)）目录中。最佳实践是使用主活动循环启动一个 goroutine。`Start`方法可用于打开和准备必要的资源，而`Stop`应该用于释放它们，以及其他延迟活动，如缓冲区刷新。

一些其他包只与 Unix 系统兼容，比如`takama/daemon`（[`github.com/takama/daemon`](https://github.com/takama/daemon)），它的工作方式类似。它也提供了一些使用示例。

# 总结

在本章中，我们回顾了如何获取与当前进程相关的信息，如 PID 和 PPID，UID 和 GID，以及工作目录。然后，我们看到了`os/exec`包如何允许我们创建子进程，以及如何读取它们的属性，类似于当前进程。

接下来，我们看了一下守护程序是什么，以及各种操作系统如何支持它们。我们验证了使用`os/exec`的`Cmd.Run`来执行一个超出其父进程生存期的进程是多么简单。

然后，我们通过 Unix 提供的自动化守护程序管理系统，逐步创建了一个能够通过`service`运行的应用程序。

在下一章中，我们将通过查看如何使用退出代码以及如何管理和发送信号来提高我们对子进程的控制。

# 问题

1.  Go 应用程序中有哪些关于当前进程的信息可用？

1.  如何创建一个子进程？

1.  如何确保子进程能够生存其父进程？

1.  你能访问子属性吗？你如何使用它们？

1.  Linux 中的守护程序是什么，它们是如何处理的？
