

# 第十二章：进程

本章提供了如何运行外部程序、如何与它们交互以及如何优雅地终止进程的食谱。在处理外部进程时，以下是一些需要记住的关键点：

+   当你启动一个外部进程时，它与你的程序并发运行。

+   如果你需要与子进程通信，你必须使用进程间通信机制，例如管道。

+   当你运行一个子进程时，它的标准输入和标准输出流对父进程来说似乎是独立的并发流。你不能依赖从这些流中接收到的数据的顺序。

本节涵盖了以下主要食谱：

+   运行外部程序

+   向进程传递参数

+   使用管道处理子进程的输出

+   向子进程提供输入

+   更改子进程的环境变量

+   使用信号优雅地终止

# 运行外部程序

有许多用例，你想要执行外部程序以执行任务。通常，这是因为在你自己的程序中执行相同的任务是不可能的或不容易的。例如，你可能选择执行多个外部图像处理程序的实例来修改一组图像。另一个用例是当你想使用制造商提供的程序配置某些设备时。本食谱包括执行外部程序的几种方法。

## 如何做到...

使用 `exec.Command` 或 `exec.CommandContext` 从你的程序中运行另一个程序。如果不需要取消（终止）子进程或设置超时，则 `exec.Command` 是合适的。否则，使用 `exec.CommandContext`，并取消或超时上下文来终止子进程：

1.  使用程序名称及其参数创建 `exec.Command`（或 `exec.CommandContext`）对象：

    +   如果你需要在平台的可执行命令路径中搜索程序，不要包含任何路径分隔符

    +   如果你在程序名称中使用路径分隔符，它必须是相对于 `exec.Command.Dir` 的路径，或者如果 `exec.Command.Dir` 为空，它必须是相对于当前工作目录的路径

    +   如果你知道可执行文件的位置，请使用绝对路径

1.  准备输入和输出流以捕获程序输出，或通过标准输入流发送输入。

1.  启动程序。

1.  等待程序结束。

以下示例在 `sub/` 目录下使用 `go` 命令构建一个 Go 程序：

```go
// Run "go build" to build the subprocess in the "sub" directory
func buildProgram() {
    // Create a Command with the executable and its arguments
     cmd := exec.Command(
       "go", "build", "-o", "subprocess", ".")
    // Set the working directory
     cmd.Dir = "sub"
    // Collect the stdout and stderr as a combined output from the 
    // process
    // This will run the process, and wait for it to end
     output, err := cmd.CombinedOutput()
     if err != nil {
          panic(err)
     }
     // The build command will not print anything if successful. So if
     // there is any output, it is a failure.
     if len(output) > 0 {
          panic(string(output))
     }
}
```

上述示例将收集进程输出作为一个合并的字符串。程序的标准输出和标准错误将作为一个字符串返回，因此你无法识别输出字符串的哪些部分来自标准输出，哪些来自标准错误。确保你可以正确解析输出。

警告

进程的标准输出和标准错误流是独立的并发流。通常，没有可移植的方法来确定哪个流首先产生了输出。这可能会产生严重的影响。例如，假设您执行了一个程序，该程序在`stdout`上产生一系列行，但每当它检测到错误时，它会将类似于“`最后打印的行有问题`”的消息打印到标准错误。但是当您在程序中读取错误时，最后打印的行可能还没有到达您的程序。

以下程序演示了`exec.CommandContext`和管道的使用：

```go
// Run the program built by buildProgram function for 10ms, reading 
// from the output
// and error pipes concurrently
func runSubProcessStreamingOutputs() {
    // Create a context with timeout
     ctx, cancel := context.WithTimeout(context.Background(), 10*time.
     Millisecond)
     defer cancel()
    // Create the command that will timeout in 10ms
     cmd := exec.CommandContext(ctx, "sub/subprocess")
    // Pipe the output and error streams
     stdout, err := cmd.StdoutPipe()
     if err != nil {
          panic(err)
     }
     stderr, err := cmd.StderrPipe()
     if err != nil {
          panic(err)
     }
     // Read from stderr from a separate goroutine
     go func() {
          io.Copy(os.Stderr, stderr)
     }()
    // Start running the program
     err = cmd.Start()
     if err != nil {
          panic(err)
     }
    // Copy the stdout of the child program to our stdout
     io.Copy(os.Stdout, stdout)
    // Wait for the program to end
     err = cmd.Wait()
     if err != nil {
          fmt.Println(err)
     }
}
```

之前的例子利用了子进程的标准输出和标准错误输出。请注意，程序在程序开始之前就开始从`stderr`流读取。那个 goroutine 将阻塞，直到子进程输出错误或子进程终止，此时`stderr`管道将关闭，goroutine 将终止。从标准输出读取的部分在主 goroutine 中运行，在`cmd.Wait`之前。这种顺序很重要。如果子进程开始在`stdout`上产生输出，但父程序没有监听，子进程将阻塞。在此处调用`cmd.Wait`将创建死锁，但运行时无法检测到这一点，因为父程序依赖于子程序的行为。

您可以将相同的流分配给子进程的`stdout`和`stderr`，如下所示：

```go
// Run the build subprocess for 10 ms with combined output
func runSubProcessCombinedOutput() {
    // Create a context with timeout
     ctx, cancel := context.WithTimeout(context.Background(), 10*time.
     Millisecond)
     defer cancel()
    // Define the command with the context
     cmd := exec.CommandContext(ctx, "sub/subprocess")
    // Assign both stdout and stderr to the same stream. This is 
    // equivalent to calling CombinedOutput
     cmd.Stdout = os.Stdout
     cmd.Stderr = os.Stdout
    // Start the process
     err := cmd.Start()
     if err != nil {
          panic(err)
     }
    // Wait until it ends. The output will be printed to our stdout
     err = cmd.Wait()
     if err != nil {
          fmt.Println(err)
     }
}
```

前述方法与使用`CombinedOutput`运行子进程类似。将`cmd.Stdout`和`cmd.Stderr`分配到同一流具有将子进程的输出合并在一起的效果。

# 向进程传递参数

将参数传递给子进程的机制可能会令人困惑。Shell 环境会解析和展开进程参数。例如，一个`*.txt`参数会被替换为匹配该模式的文件名列表，并且每个文件名都成为一个单独的参数。本食谱讨论了如何正确地将此类参数传递给子进程。

有两种方法可以将参数传递给子进程。

## 扩展参数

第一种选择是手动执行 shell 参数处理。

### 如何操作...

要手动执行 shell 处理，请按照以下步骤进行：

1.  从参数中移除 shell 特定的引号，例如以下 shell 命令：

    +   `./prog "test` `directory"` shell 命令变为`cmd:=exec.Command("./prog","test directory")`。

    +   `./prog dir1 "long dir name" '"quoted name"'` Bash 命令变为`cmd:=exec.Command("./prog", "long dir name", "'\"quoted name\"'")`。注意 Bash 对引号的特定处理。

1.  扩展模式。`./prog *.txt`变为`cmd:=exec.Command("./prog",listFiles("*.txt")...)`，其中`listFiles`是一个返回文件名切片的函数。

小贴士

通过空格分隔的文件列表作为单个参数传递。也就是说，`cmd:=exec.Command("./prog","file1.txt file2.txt")`将向进程传递单个参数，即`file1.txt file2.txt`。

1.  替换环境变量。`/.prog $HOME`变为`cmd:=exec.Command("./prog", os.Getenv("HOME"))`。运行`cmd:=exec.Command("./prog", "$HOME")`将字符串`$HOME`传递给程序，而不是其环境中的值。

1.  最后，您必须手动处理管道。也就是说，对于`./prog >output.txt`的 shell 命令，您必须运行`cmd:=exec.Command("./prog")`，创建一个`output.txt`文件，并将`cmd.Stdout=outputFile`。

## 通过 shell 运行命令

第二种选择是通过 shell 运行程序。

### 如何做到这一点...

使用特定平台的 shell 及其语法来运行命令：

```go
var cmd *exec.Cmd
switch runtime.GOOS {
case "windows":
     cmd = exec.Command("cmd", "/C", "echo test>test.txt")
case "darwin": // Mac OS
     cmd = exec.Command("/bin/sh", "-c", "echo test>test.txt")
case "linux": // Linux system, assuming there is bash
     cmd = exec.Command("/bin/bash", "-c", "echo test>test.txt")
default: // Some other OS. Assume it has `sh`
     cmd = exec.Command("/bin/sh", "-c", "echo test>test.txt")
}
out, err := cmd.Output()
```

此示例为 Windows 平台选择`cmd`，Darwin（Mac）选择`/bin/sh`，Linux 选择`/bin/bash`，其他任何平台选择`/bin/sh`。传递给 shell 的命令包含一个重定向，由 shell 处理。命令的输出将被写入到`test.txt`文件中。

# 使用管道处理子进程的输出

请记住，进程的标准输出和标准错误流是并发流。如果子进程生成的输出可能是无界的，您可以在单独的 goroutine 中处理它。这个示例展示了如何做。

## 如何做到这一点...

关于管道的一些话。管道是 Go 通道的基于流的类似物。它是一个**先进先出**（**FIFO**）的通信机制，有两个端点：一个写入器和一个读取器。读取器端在写入器写入内容之前阻塞，写入器端在读取器读取内容之前阻塞。当您完成管道时，您关闭写入器端，这也会关闭管道的读取器端。这发生在子进程终止时。如果您关闭管道的读取器端然后写入它，程序将收到信号并可能终止。如果父程序在子程序之前终止，就会发生这种情况。

1.  创建命令，并获取其`StdoutPipe`：

    ```go
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
    defer cancel()
    cmd := exec.CommandContext(ctx, "sub/subprocess")
    pipe, err := cmd.StdoutPipe()
    if err != nil {
      panic(err)
    }
    ```

1.  创建一个新的 goroutine 并从子进程的 stdout 读取。在这个 goroutine 中处理子进程的输出：

    ```go
    // Read from the pipe in a separate goroutine
    go func() {
      // Filter lines that contain "0"
      scanner := bufio.NewScanner(pipe)
      for scanner.Scan() {
        line := scanner.Text()
        if strings.Contains(line, "0")  {
          fmt.Printf("Filtered line: %s\n", line)
        }
      }
      if err := scanner.Err(); err != nil {
        fmt.Println("Scanner error: %v", err)
      }
    }()
    ```

1.  启动进程：

    ```go
    err = cmd.Start()
    if err != nil {
      panic(err)
    }
    ```

1.  等待进程结束：

    ```go
    err = cmd.Wait()
    if err != nil {
      fmt.Println(err)
    }
    ```

# 向子进程提供输入

您可以使用两种方法向子进程提供输入：将`cmd.Stdin`设置为流或使用`cmd.StdinPipe`获取发送输入到子进程的写入器。

## 如何做到这一点...

1.  创建命令：

    ```go
    // Run grep and search for a word
    cmd := exec.Command("grep", word)
    ```

1.  通过设置`Stdin`流为进程提供输入：

    ```go
    // Open a file
    input, err := os.Open("input.txt")
    if err != nil {
      panic(err)
    }
    cmd.Stdin = input
    ```

1.  运行程序并等待其结束：

    ```go
    if err = cmd.Start(); err != nil {
      panic(err)
    }
    if err = cmd.Wait(); err != nil {
      panic(err)
    }
    ```

    或者，您可以使用管道提供流式输入。

1.  创建命令：

    ```go
    // Run grep and search for a word
    cmd := exec.Command("grep", word)
    ```

1.  获取输入管道：

    ```go
    input, err:=cmd.StdinPipe()
    if err!=nil {
      panic(err)
    }
    ```

1.  通过管道将输入发送到程序。完成后，关闭管道：

    ```go
    go func() {
      // Defer close the pipe
      defer input.Close()
      // Open a file
      file, err := os.Open("input.txt")
      if err != nil {
        panic(err)
      }
      defer file.Close()
      io.Copy(input,file)
    }()
    ```

1.  运行程序并等待其结束：

    ```go
    if err = cmd.Start(); err != nil {
      panic(err)
    }
    if err = cmd.Wait(); err != nil {
      panic(err)
    }
    ```

# 修改子进程的环境变量

环境变量是与进程相关联的键值对。它们对于传递特定于环境的信息非常有用，例如当前用户的家目录、可执行搜索路径、配置选项等。在容器化部署中，环境变量是传递程序所需凭证的便捷方式。

进程的环境变量由其父进程提供，但一旦进程开始，就会为子进程分配提供的环境变量的副本。因此，父进程在子进程开始运行后不能更改其子进程的环境变量。

## 如何操作...

+   当启动子进程时，要使用与当前进程相同的环境变量，请将 `Command.Env` 设置为 `nil`。这将复制当前进程的环境变量到子进程中。

+   要使用额外的环境变量启动子进程，将这些新变量追加到当前进程变量中：

    ```go
    // Run the server
    cmd:=exec.Command("./server")
    // Copy current process environment variables
    cmd.Env=os.Environ()
    // Append new environment variables
    // Set the authentication key as an environment variable
    // of the current process
    cmd.Env=append(cmd.Env,fmt.Sprintf("AUTH_KEY=%s", authkey))
    // Start the server process. Parent process environment is copied to
    cmd.Start()
    ```

# 使用信号进行优雅终止

要优雅地终止程序，你应该执行以下操作：

+   不再接受新的请求

+   完成已接受但未完成的任何请求

+   允许一定的时间让任何长时间运行的过程完成，如果它们在给定时间内无法完成，则终止它们

在基于云的服务开发中，优雅终止尤为重要，因为大多数云服务都是短暂的，并且经常被新的实例所取代。这个配方展示了如何实现它。

## 如何操作...

1.  处理中断和终止信号。中断信号（`SIGINT`）通常由用户发起（例如，通过按下 *Ctrl* + *C*），而终止信号（`SIGTERM`）通常由宿主操作系统发起，或者在容器化环境中，由容器编排系统发起。

1.  禁止接受任何新的请求。

1.  等待现有请求完成，并设置超时

1.  终止进程。

下面的例子展示了如何操作。这是一个简单的 HTTP 回显服务器。当程序启动时，它创建一个 goroutine，监听响应 `SIGINT` 和 `SIGTERM` 信号的通道。当接收到这些信号中的任何一个时，它会关闭服务器（首先禁止单新请求的接受，然后等待现有请求完成，直到超时），然后终止程序：

```go
func main() {
  // Create a simple HTTP echo service
  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    io.Copy(w, r.Body)
  })
  server := &http.Server{Addr: ":8080"}
  // Listen for SIGINT and SIGTERM signals
  // Terminate the server with the signal
  sigTerm := make(chan os.Signal, 1)
  signal.Notify(sigTerm, syscall.SIGINT, syscall.SIGTERM)
  go func() {
    <-sigTerm
    // 5 second timeout for the server to shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.
    Second)
    defer cancel()
    server.Shutdown(ctx)
  }()
  // Start the server. When the server shuts down, program will end
  server.ListenAndServe()
}
```
