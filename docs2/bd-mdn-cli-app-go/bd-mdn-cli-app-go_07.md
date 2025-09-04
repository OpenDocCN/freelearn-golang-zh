

# 第七章：为不同平台开发

Go 语言之所以成为构建命令行应用程序的强大语言之一，主要原因是它很容易开发出可以在多台机器上运行的应用程序。Go 提供了几个包，允许开发者编写与计算机交互的代码，而无需考虑特定的操作系统。这些包包括 `os`、`time`、`path` 和 `runtime`。在第一部分，我们将讨论这些包中的一些常用函数，并提供一些简单的示例与解释相匹配。

为了进一步强调这些文件的重要性，我们将重新审视 `audiofile` 代码，并实现一些利用这些包中存在的一些方法的新功能。毕竟，通过使用新学到的函数和方法实现新功能，是学习最好的方式。

然后，我们将学习如何使用 `runtime` 库来检查应用程序正在运行的操作系统，然后使用该信息在代码之间切换。通过了解构建标签、它们是什么以及如何使用它们，我们将学习一种更干净的方法来在代码块之间切换，以实现一个可以在三个不同的操作系统上运行的新功能：Darwin、Windows 和 Linux。到本章结束时，当你构建应用程序时，你会更有信心，知道你编写的代码将无缝运行，不受平台限制。

在本章中，我们将涵盖以下关键主题：

+   用于平台无关功能的包

+   实现独立或平台特定代码

+   针对特定平台的构建标签

# 技术要求

+   本章的代码文件可在以下链接获取：[`github.com/PacktPublishing/Building-Modern-CLI-Applications-in-Go/tree/main/Chapter07`](https://github.com/PacktPublishing/Building-Modern-CLI-Applications-in-Go/tree/main/Chapter07).

# 用于平台无关功能的包

当你构建 `os`、`time` 和 `path` 时。另一个有用的包是 `runtime` 包，它有助于检测应用程序正在运行的操作系统，以及其他事情。我们将通过一些简单的示例来回顾这些包，以展示如何应用一些可用的方法。

## `os` 包

`os` 包是你的首选包。我们在上一章讨论了调用外部命令；现在我们将从更高层次讨论这个问题，并专注于某些组中的命令：环境、文件和进程操作。

### 环境操作

如其名所示，`os` 包包含提供关于应用程序运行环境信息的函数，以及为未来的方法调用更改环境。以下是一些常见操作的工作目录：

+   `func Chdir(dir string) error`: 这将更改当前工作目录

+   `func Getwd() (dir string, err error)`: 这将获取当前工作目录

环境操作也有以下内容：

+   `func Environ() []string`: 列出环境键和值

+   `func Getenv(key string) string`: 通过键获取环境变量

+   `func Setenv(key, value string) error`: 通过键和值设置环境变量

+   `func Unsetenv(key string) error`: 通过键取消设置环境变量

+   `func Clearenv()`: 清除环境变量

+   `func ExpandEnv(s string) string`: 将字符串中环境变量键的值展开为其值

代码*第七章*位于 GitHub 的`environment.go`文件中，我们在这里提供了一些示例代码，展示了如何使用这些操作：

```go
func environment() {
    dir, err := os.Getwd()
    if err != nil {
        fmt.Println("error getting working directory:", err)
    }
    fmt.Println("retrieved working directory: ", dir)
    fmt.Println("setting WORKING_DIR to", dir)
    err = os.Setenv("WORKING_DIR", dir)
    if err != nil {
        fmt.Println("error setting working directory:", err)
    }
    fmt.Println(os.ExpandEnv("WORKING_DIR=${WORKING_DIR}"))
    fmt.Println("unsetting WORKING_DIR")
    err = os.Unsetenv("WORKING_DIR")
    if err != nil {
        fmt.Println("error unsetting working directory:", err)
    }
    fmt.Println(os.ExpandEnv("WORKING_DIR=${WORKING_DIR}"))
    fmt.Printf("There are %d environment variables:\n", len(os.
        Environ()))
    for _, envar := range os.Environ() {
        fmt.Println("\t", envar)
    }
}
```

简要描述前面的代码，我们首先获取工作目录，然后将其设置为`WORKING_DIR`环境变量。为了显示变化，我们使用`os.ExpandEnv`来打印键值对。然后我们取消设置`WORKING_DIR`环境变量。同样，我们再次使用`os.ExpandEnv`来打印键值对。如果环境变量未设置，`os.ExpandEnv`变量将打印空字符串。最后，我们打印出环境变量的数量，然后遍历所有变量来打印它们。运行前面的代码将产生以下输出：

```go
retrieved working directory:  /Users/mmontagnino/Code/src/github.com/marianina8/Chapter-7
setting WORKING_DIR to /Users/mmontagnino/Code/src/github.com/marianina8/Chapter-7
WORKING_DIR=/Users/mmontagnino/Code/src/github.com/marianina8/Chapter-7
There are 44 environment variables.
key=WORKING_DIR, value=/Users/mmontagnino/Code/src/github.com/marianina8/Chapter-7
unsetting WORKING_DIR
WORKING_DIR=
```

如果你在你的机器上运行此代码而不是 Linux、Unix 或 Windows，结果输出将相似。自己试试看。

关于运行以下示例的说明

要运行*第七章*的示例，你首先需要运行安装命令将 sleep 命令安装到你的 GOPATH。在类 Unix 系统中，运行`make install`命令后跟`make run`命令。在 Linux 系统中，运行`./build-linux.sh`脚本后跟`./run-linux.sh`脚本。在 Windows 上，运行`.\build-windows.ps1`后跟`.\run-windows.ps1`PowerShell 脚本。

### 文件操作

`os`包还提供了适用于不同操作系统的广泛文件操作，这些操作可以通用。许多函数和方法可以应用于文件，所以我不一一列举名称，而是将功能分组，并列举一些：

+   以下可用于更改文件、目录和链接的权限和所有者：

    +   `func Chmod(name string, mode FileMode) error`

    +   `func Chown(name string uid, gid int) error`

    +   `func Lchown(name string uid, gid int) error`

+   以下可用于创建管道、文件、目录和链接：

    +   `func Pipe() (r *File, w *File, err error)`

    +   `func Create(name string) (*File, error)`

    +   `func Mkdir(name string, perm FileMode) error`

    +   `func Link(oldname, newname string) error`

+   以下用于从文件、目录和链接中读取：

    +   `func ReadFile(name string) ([]byte, error)`

    +   `func ReadDir(name string) ([]DirEntry, error)`

    +   `func Readlink(name string) (string, error)`

+   以下操作用于检索特定用户的数据：

    +   `func UserCacheDir() (string, error)`

    +   `func UserConfigDir() (string, error)`

    +   `func UserHomeDir() (string, error)`

+   以下用于写入文件：

    +   `func (f *File) Write(b []byte) (n int, err error)`

    +   `func (f *File) WriteString(s string) (n int, err error)`

    +   `func WriteFile(name string, data []byte, perm FileMode) error`

+   以下用于文件比较：

    +   `func SameFile(fi1, fi2 FileInfo) bool`

在 GitHub 上*第七章*的代码中有一个`file.go`文件，其中包含一些使用这些操作的示例代码。在该文件中，有多个函数，第一个是`func createFiles() error`，它处理创建三个文件以供操作：

```go
func createFiles() error {
    filename1 := "file1"
    filename2 := "file2"
    filename3 := "file3"
    f1, err := os.Create(filename1)
    if err != nil {
        return fmt.Errorf("error creating %s: %v\n", filename1, 
          err)
    }
    defer f1.Close()
    f1.WriteString("abc")
    f2, err := os.Create(filename2)
    if err != nil {
        return fmt.Errorf("error creating %s: %v\n", filename2, 
          err)
    }
    defer f2.Close()
    f2.WriteString("123")
    f3, err := os.Create(filename3)
    if err != nil {
        return fmt.Errorf("error creating %s: %v", filename3, 
          err)
    }
    defer f3.Close()
    f3.WriteString("xyz")
    return nil
}
```

`os.Create`方法允许在不同的操作系统上无缝地创建文件。下一个函数`file()`利用这些文件来展示如何使用存在于`os`包中的方法。`file()`函数主要获取或更改当前工作目录，并运行不同的函数，包括以下内容：

+   `func createExamplesDir() (string, error)`: 这个函数在用户主目录中创建一个`examples`目录

+   `func printFiles(dir string) error`: 这个函数打印出由`dir string`表示的目录下的文件/目录

+   `func sameFileCheck(f1, f2 string) error`: 这个函数检查由`f1`和`f2`字符串表示的两个文件是否是同一个文件

让我们先展示`file()`函数，以了解整体情况：

```go
originalWorkingDir, err := os.Getwd()
if err != nil {
    fmt.Println("getting working directory: ", err)
}
fmt.Println("working directory: ", originalWorkingDir)
examplesDir, err := createExamplesDir()
if err != nil {
    fmt.Println("creating examples directory: ", err)
}
err = os.Chdir(examplesDir)
if err != nil {
    fmt.Println("changing directory error:", err)
}
fmt.Println("changed working directory: ", examplesDir)
workingDir, err := os.Getwd()
if err != nil {
    fmt.Println("getting working directory: ", err)
}
fmt.Println("working directory: ", workingDir)
createFiles()
err = printFiles(workingDir)
if err != nil {
    fmt.Printf("Error printing files in %s\n", workingDir)
}
err = os.Chdir(originalWorkingDir)
if err != nil {
    fmt.Println("changing directory error: ", err)
}
fmt.Println("working directory: ", workingDir)
symlink := filepath.Join(originalWorkingDir, "examplesLink")
err = os.Symlink(examplesDir, symlink)
if err != nil {
    fmt.Println("error creating symlink: ", err)
}
fmt.Printf("created symlink, %s, to %s\n", symlink, examplesDir)
err = printFiles(symlink)
if err != nil {
    fmt.Printf("Error printing files in %s\n", workingDir)
}
file := filepath.Join(examplesDir, "file1")
linkedFile := filepath.Join(symlink, "file1")
err = sameFileCheck(file, linkedFile)
if err != nil {
    fmt.Println("unable to do same file check: ", err)
}
// cleanup
err = os.Remove(symlink)
if err != nil {
    fmt.Println("removing symlink error: ", err)
}
err = os.RemoveAll(examplesDir)
if err != nil {
    fmt.Println("removing directory error: ", err)
}
```

让我们回顾一下前面的代码。首先，我们获取当前工作目录并打印出来。然后，我们调用`createExamplesDir()`函数并进入该目录。

我们在更改工作目录后获取当前工作目录，以确保它现在是`examplesDir`值。接下来，我们调用`createFiles()`函数在`examplesDir`文件夹内创建这三个文件，并调用`printFiles()`函数列出`examplesDir`工作目录中的文件。

我们将工作目录改回原始工作目录，并在主目录下创建一个指向`examplesDir`文件夹的`symlink`。我们打印出`symlink`下的文件，以确认它们是相同的。

然后，我们从`examplesDir`中取出`file0`和从`symlink`中取出的`file0`，在`sameFileCheck`函数中进行比较，以确保它们相等。

最后，我们运行一些清理函数来删除`symlink`和`examplesDir`文件夹。

`file()`函数利用了`os`包中许多可用的方法，从获取工作目录到更改它，创建`symlink`，以及删除文件和目录。展示单独的函数调用代码将展示`os`包的更多用途。首先，让我们展示`createExamplesDir`的代码：

```go
func createExamplesDir() (string, error) {
    homeDir, err := os.UserHomeDir()
    if err != nil {
        return "", fmt.Errorf("getting user's home directory: 
          %v\n", err)
    }
    fmt.Println("home directory: ", homeDir)
    examplesDir := filepath.Join(homeDir, "examples")
    err = os.Mkdir(examplesDir, os.FileMode(int(0777)))
    if err != nil {
        return "", fmt.Errorf("making directory error: %v\n", 
          err)
    }
    fmt.Println("created: ", examplesDir)
    return examplesDir, nil
}
```

前面的代码在获取用户主目录时使用了`os.UserHomeDir`方法，然后使用`os.Mkdir`方法创建了一个新文件夹。下一个函数`printFiles`从`os.ReadDir`方法获取要打印的文件：

```go
func printFiles(dir string) error {
    files, err := os.ReadDir(dir)
    if err != nil {
        return fmt.Errorf("read directory error: %s\n", err)
    }
    fmt.Printf("files in %s:\n", dir)
    for i, file := range files {
        fmt.Printf(" %v %v\n", i, file.Name())
    }
    return nil
}
```

最后，`sameFileCheck`接受两个由字符串表示的文件，`f1`和`f2`。要获取每个文件的文件信息，我们将在文件字符串上调用`os.Lstat`方法。`os.SameFile`接受此文件信息并返回一个`boolean`值来表示结果——如果文件相同则返回`true`，否则返回`false`：

```go
func sameFileCheck(f1, f2 string) error {
    fileInfo0, err := os.Lstat(f1)
    if err != nil {
        return fmt.Errorf("getting fileinfo: %v", err)
    }
    fileInfo0Linked, err := os.Lstat(f2)
    if err != nil {
        return fmt.Errorf("getting fileinfo: %v", err)
    }
    isSameFile := os.SameFile(fileInfo0, fileInfo0Linked)
    if isSameFile {
        fmt.Printf("%s and %s are the same file.\n", fileInfo0.
            Name(), fileInfo0Linked.Name())
    } else {
    fmt.Printf("%s and %s are NOT the same file.\n", fileInfo0.
        Name(), fileInfo0Linked.Name())
    }
    return nil
}
```

这总结了使用`os`包中与文件操作相关的方法的代码示例。接下来，我们将讨论一些与机器上运行的进程相关的操作。

### 进程操作

当调用外部命令时，我们可以使用`os`包，我们可以对进程执行操作，发送进程信号，或等待进程完成，然后接收一个包含有关已完成的进程信息的进程状态。在*第七章*代码中，我们有一个`process()`函数，它利用以下方法对进程和进程状态进行操作：

+   `func Getegid() int`: 这返回调用者的有效组 ID。注意，Windows 不支持此功能，组 ID 的概念仅适用于 Unix-like 或 Linux 系统。例如，在 Windows 上这将返回`-1`。

+   `func Geteuid() int`: 这返回调用者的有效用户 ID。注意，Windows 不支持此功能，用户 ID 的概念仅适用于 Unix-like 或 Linux 系统。例如，在 Windows 上这将返回`-1`。

+   `func Getpid() int`: 这获取调用者的进程 ID。

+   `func FindProcess(pid int) (*Process, error)`: 这返回与`pid`关联的进程。

+   `func (p *Process) Wait() (*ProcessState, error)`: 当进程完成时返回进程状态。

+   `func (p *ProcessState) Exited() bool`: 如果进程已退出，则返回`true`。

+   `func (p *ProcessState) Success() bool`: 如果进程成功退出，则返回`true`。

+   `func (p *ProcessState) ExitCode() int`: 这返回进程的退出代码。

+   `func (p *ProcessState) String() string`: 这以字符串格式返回进程状态。

以下代码如下，并开始于几个打印行语句，这些语句返回调用者的有效组、用户和进程 ID。接下来，定义了一个`cmd`睡眠命令。启动命令，并从`cmd`值中，我们得到 pid：

```go
func process() {
    fmt.Println("Caller group id:", os.Getegid())
    fmt.Println("Caller user id:", os.Geteuid())
    fmt.Println("Process id of caller", os.Getpid())
    cmd := exec.Command(filepath.Join(os.Getenv("GOPATH"), 
           "bin", "sleep"))
    fmt.Println("running sleep for 1 second...")
    if err := cmd.Start(); err != nil {
        panic(err)
    }
    fmt.Println("Process id of sleep", cmd.Process.Pid)
    this, err := os.FindProcess(cmd.Process.Pid)
    if err != nil {
        fmt.Println("unable to find process with id: ", cmd.
            Process.Pid)
    }
    processState, err := this.Wait()
    if err != nil {
        panic(err)
    }
    if processState.Exited() && processState.Success() {
        fmt.Println("Sleep process ran successfully with exit 
            code: ", processState.ExitCode())
    } else {
        fmt.Println("Sleep process failed with exit code: ", 
            processState.ExitCode())
    }
    fmt.Println(processState.String())
}
```

从进程的 pid，然后我们可以使用`os.FindProcess`方法找到进程。我们调用进程的`Wait()`方法以获取`os.ProcessState`。这个`Wait()`方法，就像`cmd.Wait()`方法一样，等待进程完成。一旦完成，就返回进程状态。我们可以使用`Exited()`方法检查进程状态是否已退出，以及是否成功使用`Success()`方法。如果是这样，我们将打印进程运行成功，以及我们从`ExitCode()`方法获取的退出代码。最后，我们可以使用`String()`方法干净地打印进程状态。

## 时间包

操作系统通过两种不同类型的内部时钟提供对时间的访问：

+   **墙钟**：这用于告知时间，并且由于与 **网络时间协议**（**NTP**）的时钟同步，可能会出现变化

+   **单调时钟**：这用于测量时间，并且不受时钟同步变化的影响

要更具体地说明这些变化，如果墙钟注意到它比 NTP 移动得更快或更慢，它将调整其时钟速率。单调时钟不会调整。在测量持续时间时，使用单调时钟非常重要。幸运的是，在 Go 中，`Time` 结构体包含墙钟和单调时钟，我们不需要指定使用哪一个。在 *第七章* 的代码中，有一个 `timer.go` 文件，展示了如何获取当前时间和持续时间，无论操作系统如何：

```go
func timer() {
    start := time.Now()
    fmt.Println("start time: ", start)
    time.Sleep(1 * time.Second)
    elapsed := time.Until(start)
    fmt.Println("elapsed time: ", elapsed)
}
```

当运行以下代码时，你们将看到类似的输出：

```go
start time:  2022-09-24 23:47:38.964133 -0700 PDT m=+0.000657043
elapsed time:  -1.002107875s
```

此外，你们中的许多人也看到过有一个 `time.Now().Unix()` 方法。它返回自 Unix 纪元（即自 1970 年 1 月 1 日 UTC 以来经过的时间）的纪元时间。这些方法在它们运行的操作系统和架构上都将以类似的方式工作。

## 路径包

当为不同的操作系统开发命令行应用程序时，你们很可能会不得不处理文件或目录路径名。为了在不同操作系统中适当地处理这些路径，你们需要使用 `path` 包。因为这个包不处理像我们在前面的例子中使用的那样带有驱动器字母或反斜杠的 Windows 路径，我们将使用 `path/filepath` 包。

`path/filepath` 包根据操作系统使用正斜杠或反斜杠。为了好玩，在 *第七章* 的 `walking.go` 文件中，我使用了 `filepath` 包来遍历一个目录。让我们看看代码：

```go
func walking() {
    workingDir, err := os.Getwd()
    if err != nil {
        panic(err)
    }
    dir1 := filepath.Join(workingDir, "dir1")
    filepath.WalkDir(dir1, func(path string, d fs.DirEntry, err 
      error) error {
        if !d.IsDir() {
            contents, err := os.ReadFile(path)
            if err != nil {
                return err
            }
            fmt.Printf("%s -> %s\n", d.Name(), 
                string(contents))
        }
        return nil
    })
}
```

我们使用 `os.Getwd()` 获取当前工作目录。然后使用 `filepath.Join` 方法为 `dir1` 目录创建一个可用于任何操作系统的路径。最后，我们使用 `filepath.WalkDir` 遍历目录，并打印出文件名及其内容。

## 运行时包

在本节中要讨论的最后一个包是 `runtime` 包。它之所以被提及，是因为它被用来轻松确定代码运行的操作系统，并因此执行代码块，但你可以从 `runtime` 系统中获得大量信息：

+   `GOOS`: 这返回运行应用程序的目标操作系统

+   `GOARCH:` 这返回运行应用程序的目标架构

+   `func GOROOT() string`: 这返回 Go 树的根目录

+   `Compiler`: 这返回构建二进制的编译器工具链的名称

+   `func NumCPU() int`: 这返回当前进程可用的逻辑 CPU 数量

+   `func NumGoroutine() int`: 这返回当前存在的 goroutine 数量

+   `func Version() string`: 这返回 Go 树的版本字符串

此包将为您提供足够的信息来理解`runtime`环境。在`checkRuntime.go`文件中的*第七章*代码中是`checkRuntime`函数，它将这些方法付诸实践：

```go
func checkRuntime() {
    fmt.Println("Operating System:", runtime.GOOS)
    fmt.Println("Architecture:", runtime.GOARCH)
    fmt.Println("Go Root:", runtime.GOROOT())
    fmt.Println("Compiler:", runtime.Compiler)
    fmt.Println("No. of CPU:", runtime.NumCPU())
    fmt.Println("No. of Goroutines:", runtime.NumGoroutine())
    fmt.Println("Version:", runtime.Version())
    debug.PrintStack()
}
```

运行代码将提供类似于以下输出的结果：

```go
Operating System: darwin
Architecture: amd64
Go Root: /usr/local/go
Compiler: gc
No. of CPU: 10
No. of Goroutines: 1
Version: go1.19
goroutine 1 [running]:
runtime/debug.Stack()
        /usr/local/go/src/runtime/debug/stack.go:24 +0x65
runtime/debug.PrintStack()
        /usr/local/go/src/runtime/debug/stack.go:16 +0x19
main.checkRuntime()
        /Users/mmontagnino/Code/src/github.com/marianina8/Chapter-7/checkRuntime.go:17 +0x372
main.main()
        /Users/mmontagnino/Code/src/github.com/marianina8/Chapter-7/main.go:9 +0x34
```

现在我们已经学习了构建跨多个操作系统和架构的命令行应用程序所需的一些包，在下一节中，我们将回到之前章节中的`audiofile` CLI，并实现一些新的功能，展示我们在这部分学到的方法和函数如何发挥作用。

# 实现独立或平台特定代码

学习的最佳方式是将所学知识付诸实践。在本节中，我们将重新审视`audiofile` CLI 以实现一些新的命令。在我们将要实现的新功能代码中，重点将放在`os`和`path`/`filepath`包的使用上。

## 平台无关代码

现在我们将为`audiofile` CLI 实现一些新的功能，这些功能将独立于操作系统运行：

+   `Delete`：通过 ID 删除存储的元数据

+   `Search`：在存储的元数据中搜索特定的搜索字符串

这些新功能命令的创建是从 cobra-CLI 开始的；然而，特定平台的代码被隔离在`storage/flatfile.go`文件中，这是存储接口的平面文件存储。

首先，让我们展示`Delete`方法：

```go
func (f FlatFile) Delete(id string) error {
    dirname, err := os.UserHomeDir()
    if err != nil {
        return err
    }
    audioIDFilePath := filepath.Join(dirname, "audiofile", id)
    err = os.RemoveAll(audioIDFilePath)
    if err != nil {
        return err
    }
    return nil
}
```

平面文件存储存储在用户主目录下的`audiofile`目录中。然后，随着每个新的音频文件和匹配的元数据的添加，它们被存储在其唯一的标识符 ID 中。从`os`包中，我们使用`os.UserHomeDir()`来获取用户的主目录，然后使用`filepath.Join`方法创建删除与 ID 相关的所有元数据和文件的所需路径，这些路径独立于操作系统。确保您在平面文件存储中存储了一些音频文件。如果没有，添加一些文件。例如，使用`audio/beatdoctor.mp3`文件，并使用以下命令上传：

```go
./bin/audiofile upload --filename audio/beatdoctor.mp3
```

成功上传后返回 ID：

```go
Uploading audio/beatdoctor.mp3 ...
Audiofile ID:  a5d9ab11-6f5f-4da0-9307-a3b609b0a6ba
```

您可以通过运行`list`命令来确保数据已被添加：

```go
./bin/audiofile list
```

返回了`audiofile`元数据，因此我们已经检查了它在存储中的存在：

```go
    {
        "Id": "a5d9ab11-6f5f-4da0-9307-a3b609b0a6ba",
        "Path": "/Users/mmontagnino/audiofile/a5d9ab11-6f5f-4da0-9307-a3b609b0a6ba/beatdoctor.mp3",
        "Metadata": {
            "tags": {
                "title": "Shot In The Dark",
                "album": "Best Bytes Volume 4",
                "artist": "Beat Doctor",
                "album_artist": "Toucan Music (Various Artists)",
                "composer": "",
                "genre": "Electro House",
                "year": 0,
                "lyrics": "",
                "comment": "URL: http://freemusicarchive.org/music/Beat_Doctor/Best_Bytes_Volume_4/09_beat_doctor_shot_in_the_dark\r\nComments: http://freemusicarchive.org/\r\nCurator: Toucan Music\r\nCopyright: Attribution-NonCommercial 3.0 International: http://creativecommons.org/licenses/by-nc/3.0/"
            },
            "transcript": ""
        },
        "Status": "Complete",
        "Error": null
    },
```

现在，我们可以删除它：

```go
./bin/audiofile delete --id a5d9ab11-6f5f-4da0-9307-a3b609b0a6ba
success
```

然后通过尝试通过 ID 获取音频来确认它已被删除：

```go
./bin/audiofile get --id a5d9ab11-6f5f-4da0-9307-a3b609b0a6ba
Error: unexpected response: 500 Internal Server Error
Usage:
  audiofile get [flags]
Flags:
  -h, --help        help for get
      --id string   audiofile id
unexpected response: 500 Internal Server Error%
```

看起来发生了一个意外的错误，我们没有正确实现当搜索已删除文件的元数据时如何处理这种情况。我们需要修改`services/metadata/handler_getbyid.go`文件。在第 20 行，当我们调用`GetById`方法并处理错误时，在确认错误与找不到文件夹有关后，让我们返回`200`而不是`500`。这并不一定是用户正在搜索一个不存在的 ID 的错误：

```go
audio, err := m.Storage.GetByID(id)
if err != nil {
    if strings.Contains(err.Error(), "not found") ||     strings.Contains(err.Error(), "no such file or directory") {
        io.WriteString(res, "id not found")
        res.WriteHeader(200)
        return
    }
    res.WriteHeader(500)
    return
}
```

让我们再试一次：

```go
./bin/audiofile get --id a5d9ab11-6f5f-4da0-9307-a3b609b0a6ba
id not found
```

现在好多了！现在让我们实现搜索功能。实现再次被隔离到`storage/flatfile.go`文件中，你将在其中找到`Search`方法：

```go
func (f FlatFile) Search(searchFor string) ([]*models.Audio, error) {
    dirname, err := os.UserHomeDir()
    if err != nil {
        return nil, err
    }
    audioFilePath := filepath.Join(dirname, "audiofile")
    matchingAudio := []*models.Audio{}
    err = filepath.WalkDir(audioFilePath, func(path string, 
          d fs.DirEntry, err error) error {
        if d.Name() == "metadata.json" {
            contents, err := os.ReadFile(path)
            if err != nil {
                return err
            }
            if strings.Contains(strings.
               ToLower(string(contents)), strings.
               ToLower(searchFor)) {
                data := models.Audio{}
                err = json.Unmarshal(contents, &data)
                if err != nil {
                    return err
                }
                matchingAudio = append(matchingAudio, &data)
            }
        }
        return nil
    })
    return matchingAudio, err
}
```

如同存储中大多数方法一样，我们首先使用`os.UserHomeDir()`方法获取用户的家目录，然后再次使用`filepath.Join`来获取根`audiofile`路径目录，我们将从这里开始遍历。调用`filepath.WalkDir`方法从`audioFilePath`开始。我们检查每个`metadata.json`文件，看`searchFor`字符串是否存在于内容中。该方法返回一个`*models.Audio`切片，如果`searchFor`字符串在内容中找到，音频将被追加到稍后返回的切片中。

让我们用以下命令尝试一下，看看是否返回了预期的元数据：

```go
./bin/audiofile search --value "Beat Doctor"
```

现在我们已经创建了一些新的命令来展示如何在实际例子中使用`os`包和`path/filepath`包，让我们尝试编写一些可以在特定操作系统上运行的代码。

## 平台特定代码

假设你的命令行应用程序需要操作系统上存在的外部应用程序，但所需的应用程序在不同操作系统之间可能不同。对于`audiofile`命令行应用程序，假设我们想要创建一个命令来通过命令行播放音频文件。每个操作系统都需要使用不同的命令来播放音频，如下所示：

+   macOS: `afplay <filepath>`

+   Windows: `start <filepath>`

+   Linux: `aplay <filepath>`

再次，我们使用 Cobra-CLI 来创建新的`play`命令。让我们看看每个操作系统播放音频文件需要调用的不同函数。首先是 macOS 的代码：

```go
func darwinPlay(audiofilePath string) {
    cmd := exec.Command("afplay", audiofilePath)
    if err := cmd.Start(); err != nil {
       panic(err)
    }
    fmt.Println("enjoy the music!")
    err := cmd.Wait()
    if err != nil {
       panic(err)
    }
}
```

我们创建了一个命令来使用`afplay`可执行文件并传入`audiofilePath`。接下来是 Windows 的代码：

```go
func windowsPlay(audiofilePath string) {
    cmd := exec.Command("cmd", "/C", "start", audiofilePath)
    if err := cmd.Start(); err != nil {
        return err
    }
    fmt.Println("enjoy the music!")
    err := cmd.Wait()
    if err != nil {
        return err
    }
}
```

这是一个非常类似的功能，除了它使用 Windows 中的`start`可执行文件来播放音频。接下来是 Linux 的代码：

```go
func linuxPlay(audiofilePath string) {
    cmd := exec.Command("aplay", audiofilePath)
    if err := cmd.Start(); err != nil {
        panic(err)
    }
    fmt.Println("enjoy the music!")
    err := cmd.Wait()
    if err != nil {
        panic(err)
    }
}
```

再次，代码几乎完全相同，只是调用播放音频的应用程序不同。在另一种情况下，此代码可能需要针对操作系统更具体，需要不同的参数，甚至需要操作系统特定的完整路径。无论如何，我们已准备好在`play`命令的`RunE`字段中使用这些函数。完整的`play`命令如下：

```go
var playCmd = &cobra.Command{
    Use: "play",
    Short: "Play audio file by id",
    RunE: func(cmd *cobra.Command, args []string) error {
        b, err := getAudioByID(cmd)
        if err != nil {
            return err
        }
        audio := models.Audio{}
        err = json.Unmarshal(b, &audio)
        if err != nil {
            return err
        }
        switch runtime.GOOS {
        case "darwin":
            darwinPlay(audio.Path)
            return nil
        case "windows":
            windowsPlay(audio.Path)
            return nil
        case "linux":
            linuxPlay(audio.Path)
            return nil
        default:
            fmt.Println(`Your operating system isn't supported 
                for playing music yet.
                Feel free to implement your additional use 
                case!`)
        }
        return nil
    },
}
```

这段代码的重要部分是我们为`runtime.GOOS`值创建了一个 switch case，它告诉我们应用程序正在运行在哪个操作系统上。根据操作系统，会调用不同的方法来启动一个进程来播放音频文件。让我们重新编译并尝试使用存储的音频文件 ID 之一来测试播放方法：

```go
./bin/audiofile play --id bf22c5c4-9761-4b47-aab0-47e93d1114c8
enjoy the music!
```

本章的最后部分将向我们展示如果我们想的话，如何使用构建标签来实现这一点。

# 针对平台的构建标签

构建标签，或构建约束，可用于许多目的，但在这个部分，我们将讨论如何使用构建标签来识别在为特定操作系统构建时应该包含哪些文件。构建标签位于文件顶部的注释中：

```go
//go:build
```

在运行 `go build` 时，构建标签作为标志传入。一个文件上可能有多个标签，并且它们遵循以下注释语法：

```go
//go:build [tags]
```

每个标签之间由一个空格分隔。假设我们想表明这个文件只会在 Darwin 操作系统的构建中包含，那么我们就可以将其添加到文件的顶部：

```go
//go:build darwin
```

然后在构建应用程序时，我们会使用类似以下的内容：

```go
go build –tags darwin
```

这只是一个关于如何使用构建标签来限制特定于操作系统的文件的快速概述。在我们进入实现之前，让我们更详细地讨论一下 `build` 包。

## 构建包

`build` 包收集有关 Go 包的信息。在 *第七章* 代码仓库中，有一个 `buildChecks.go` 文件，该文件使用 `build` 包来获取当前包的信息。让我们看看这段代码能提供哪些信息：

```go
func buildChecks() {
    ctx := build.Context{}
    p1, err := ctx.Import(".", ".", build.AllowBinary)
    if err != nil {
        fmt.Println("err: ", err)
    }
    fmt.Println("Dir:", p1.Dir)
    fmt.Println("Package name: ", p1.Name)
    fmt.Println("AllTags: ", p1.AllTags)
    fmt.Println("GoFiles: ", p1.GoFiles)
    fmt.Println("Imports: ", p1.Imports)
    fmt.Println("isCommand: ", p1.IsCommand())
    fmt.Println("IsLocalImport: ", build.IsLocalImport("."))
    fmt.Println(ctx)
}
```

我们首先创建 `context` 变量，然后调用 `Import` 方法。`Import` 方法在文档中的定义如下：

```go
func (ctxt *Context) Import(path string, srcDir string, mode ImportMode) (*Package, error)
```

它返回由 `path` 和 `srcDir` 源目录参数命名的 Go 包的详细信息。在这种情况下，`main` 包从包中返回，然后我们可以检查所有变量和方法，以获取有关包的更多信息。在本地运行此方法将返回类似以下内容：

```go
Dir: .
Package name:  main
AllTags:  [buildChecks]
GoFiles:  [checkRuntime.go environment.go file.go main.go process.go timer.go walking.go]
Imports:  [fmt io/fs os os/exec path/filepath runtime runtime/debug strings time]
isCommand/main package:  true
IsLocalImport:  true
```

我们检查的大多数值都是不言自明的。`AllTags` 返回存在于 `main` 包中的所有标签。`GoFiles` 返回包含在 `main` 包中的所有文件。`Imports` 包含包中存在的所有唯一导入。`IsCommand()` 如果包被视为要安装的命令，或者它是主包，则返回 `true`。最后，`IsLocalImport` 方法检查导入文件是否为本地文件。这是一个有趣的小细节，可以让你对 `build` 包可能提供的功能更加感兴趣。

## 构建标签

现在我们对 `build` 包有了更多了解，让我们用它来完成本章的主要目的，为特定操作系统构建包。构建标签应该有意命名，因为我们使用它们来完成特定目的，所以我们可以按操作系统命名每个构建标签：

```go
//go:build darwin
//go:build linux
//go:build windows
```

让我们回顾一下音频文件代码。记得在 `play` 命令中，我们检查 `runtime` 操作系统，然后调用一个特定方法。让我们使用构建标签重写这段代码。

### 音频文件中的示例

让我们先简化命令的代码如下：

```go
var playCmd = &cobra.Command{
    Use: "play",
    Short: "Play audio file by id",
    Long: `Play audio file by id`,
    RunE: func(cmd *cobra.Command, args []string) error {
        b, err := getAudioByID(cmd)
        if err != nil {
            return err
        }
        audio := models.Audio{}
        err = json.Unmarshal(b, &audio)
        if err != nil {
            return err
        }
        return play(audio.Path)
    },
}
```

通过删除操作系统切换语句和实现每个操作系统播放功能的三个函数，我们极大地简化了代码。相反，我们创建了三个新的文件：`play_darwin.go`、`play_windows.go` 和 `play_linux.go`。在每个文件中都有一个针对每个操作系统的构建标签。以 Darwin 文件 `play_darwin.go` 为例：

```go
//go:build darwin
package cmd
import (
    "fmt"
    "os/exec"
)
func play(audiofilePath string) error {
    cmd := exec.Command("afplay", audiofilePath)
    if err := cmd.Start(); err != nil {
        return err
    }
    fmt.Println("enjoy the music!")
    err := cmd.Wait()
    if err != nil {
        return err
    }
    return nil
}
```

注意到 `play` 函数已被重命名为与 `play.go` 中 `play` 命令中调用的函数相匹配。由于只有一个文件被包含在构建中，因此不会混淆哪个 `play` 函数被调用。我们确保在 `make` 文件中只调用一个，这是我们目前运行应用程序的方式。在 `Makefile` 中，我指定了一个专门为 Darwin 构建的命令：

```go
build-darwin:
    go build -tags darwin -o bin/audiofile main.go
    chmod +x bin/audiofile
```

为 Windows 和 Linux 创建了一个包含 `play` 函数的 Go 文件。在构建应用程序时，每个操作系统的特定标签也需要类似地传递给 `-tags` 标志。在后面的章节中，我们将讨论交叉编译，这是下一步。但在我们这样做之前，让我们通过回顾一个列表来结束这一章，这个列表列出了在为多个平台开发时需要记住的操作系统级别的差异。

## 操作系统级别的差异

由于您将为主要的操作系统构建应用程序，了解它们之间的差异以及需要注意的事项非常重要。让我们从以下列表开始深入了解：

+   在 Windows 中，反斜杠（`\`）用作目录分隔符，而 Linux 和 Unix 使用正斜杠（`/`）。

+   **权限**：

    +   类 Unix 系统使用文件模式来管理权限，其中权限分配给文件和目录。

    +   Windows 使用 **访问控制列表**（**ACL**）来管理权限，其中权限以更灵活和细粒度的方式分配给文件或目录中的特定用户或组。

    +   通常，在开发任何命令行应用程序时，无论它将在哪个操作系统上运行，仔细考虑用户和组权限都是一个好习惯。*   Go 中的 `exec` 包提供了一种方便的方式，可以在终端中以相同的方式运行命令。然而，需要注意的是，命令及其参数必须以正确的格式传递给每个操作系统。*   在 Windows 上，您需要指定文件扩展名（例如，`.exe`、`.bat` 等）以运行可执行文件。*   **环境变量**：

    +   环境变量可以用来配置您的应用程序，但它们的名称和值在 Windows 和 Linux/Unix 之间可能不同。

    +   在 Windows 上，环境变量名称不区分大小写，而在 Linux/Unix 上，它们是区分大小写的。*   `\r`) 后跟一个换行符 (`\n`)，而 Linux/Unix 只使用换行符 (`\n`)。*   `os/signal` 包提供了一种处理发送到您应用程序的信号的方法。然而，此包在 Windows 上不受支持。*   要以跨平台的方式处理信号，您可以使用 `os/exec` 包代替。*   `os.Stdin` 属性，而在 Linux/Unix 上，您可以使用 `os.Stdin` 或 `bufio` 包来读取用户输入。*   `go-colorable` 提供了一种平台无关的方式来处理控制台颜色。*   `os.Stdin`、`os.Stdout` 和 `os.Stderr` 在 Windows 和 Linux/Unix 之间可能表现不同。在两个平台上测试您的代码以确保它按预期工作是很重要的。

这些是在 Go 中为不同操作系统开发命令行应用程序时需要注意的一些差异。在各个平台上彻底测试您的应用程序以确保它按预期工作是很重要的。

# 摘要

您的应用程序支持的操作系统越多，事情就会变得越复杂。希望您掌握了开发平台无关应用程序的一些支持包的知识，您将对自己的应用程序能够在不同的操作系统上以类似的方式运行而感到自信。此外，通过检查 `runtime` 操作系统，甚至使用构建标签将代码分离成不同的操作系统特定文件，您至少有几种定义代码组织方式的选择。这一章可能比必要的更深入，但希望它能给您带来灵感。

为多个操作系统构建将扩展您命令行应用程序的使用范围。您不仅能够接触到 Linux 或 Unix 用户，还能接触到 Darwin 和 Windows 用户。如果您想扩大用户群，那么构建支持更多操作系统的应用程序是一种简单的方法。

在下一章，*第八章*，*为人类构建与为机器构建*，我们将学习如何构建一个根据接收者是谁（机器或人类）输出 CLI 的命令行界面。我们还将学习如何构建清晰的语言结构，并为与社区中其他 CLI 保持一致而命名命令。

# 问题

1.  操作系统中存在两种不同的时钟，是哪一个？Go 中的 `time.Time` 结构体存储哪一个时钟，或者两者都存储？计算持续时间应该使用哪一个？

1.  哪个包常量可以用来确定 `runtime` 操作系统？

1.  在 Go 文件中，构建标签注释设置在哪里——顶部、底部还是定义函数之上？

# 答案

1.  墙上时钟和单调时钟。`time.Time` 结构体存储了两个时间值。计算持续时间时应使用单调时钟值。

1.  `runtime.GOOS`

1.  在 Go 文件的顶部第一行。

# 进一步阅读

+   访问在线文档，了解在[`pkg.go.dev/`](https://pkg.go.dev/)中讨论的包。

# 第三部分：交互性和同理心驱动的设计

本部分主要介绍如何从最终用户的角度出发，开发一个更用户友好的命令行界面（CLI）。它涵盖了诸如为人类而非机器构建、使用 ASCII 艺术提高信息密度、确保标志名称和参数的一致性等主题。本节还强调了同理心在 CLI 开发中的重要性，包括以用户友好的方式重写错误、提供详细的日志记录、创建手册页和用法示例。此外，还讨论了通过提示和终端仪表板进行交互的好处，并提供了如何使用 Termdash 库构建用户提示和仪表板的示例。

本部分包含以下章节：

+   *第八章*，*为人类而非机器构建*

+   *第九章*，*开发的同理心方面*

+   *第十章*，*使用提示和终端仪表板进行交互*
