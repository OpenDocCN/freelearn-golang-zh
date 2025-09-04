# 6

# 理解进程间通信中的管道

管道是进程间通信（IPC）的基本工具，它允许系统进程之间进行高效的数据传输。本章提供了对管道的全面理解，包括其功能以及在各种编程场景中的应用，特别是它们在 Go 中的使用。

到本章结束时，你将清楚地了解管道在 IPC 中的功能，它们在系统编程中的重要性，以及如何在 Go 中有效地实现它们。本章旨在使读者具备利用管道在编程项目中实现高效进程通信的知识。

在本章中，我们将涵盖以下主要主题：

+   IPC 中的管道是什么？

+   匿名管道的机制

+   导航命名管道（`Mkfifo()`）

+   最佳实践 - 使用管道的指南

+   开发日志处理工具

# 技术要求

我们将使用一些系统依赖来执行本章的示例。因此，请确保你有这些程序可用：

+   `grep`

+   `echo`

# IPC 中的管道是什么？

在系统编程中，我们可以将管道想象为内存中的管道，用于在两个或多个进程之间传输数据。这个管道遵循生产者-消费者模型：一个进程，即生产者，将数据注入管道，而另一个进程，即消费者，从这个流中读取数据。作为 IPC 的关键元素，管道建立了一个单向的信息流。这种设置确保数据始终朝一个方向移动 - 从管道的“写入端”到“读取端”。这种机制允许进程以流畅和高效的方式进行通信，就像水通过管道流动一样，一个进程将信息顺利传递给下一个进程。

管道被用于各种系统级编程任务。最常见的应用包括以下：

+   **命令行实用程序**：管道通常用于将一个命令行实用程序的输出连接到另一个实用程序的输入，从而创建强大的命令链

+   **数据流**：当数据需要从一个进程流到另一个进程时，管道提供了一个简单而有效的解决方案

+   **进程间数据交换**：管道促进了进程间的数据交换，这在许多多进程应用中是必不可少的

## 管道为什么重要？

管道允许模块化软件创建，其中不同的进程专门从事特定任务并高效地通信。它们通过允许进程之间直接通信而不需要中间存储，从而促进了系统资源的有效利用。此外，它们提供了一个简单而强大的数据交换接口，使得复杂操作更容易管理。

由于管道设计为允许数据单向移动，因此通常使用两个管道进行双向通信。它们在另一个进程读取数据之前对数据进行缓冲。这种机制在处理读者和作者操作速度不同的情况时特别有用。

到目前为止，你可能已经在挠头并问自己*它们是 Go 的类似通道的结构吗？*答案是*是的，在* *某种程度上*。

它们之间有相似之处：

+   **通信机制**：管道和通道主要用于通信。管道促进进程间通信（IPC），而通道用于 Go 程序中 goroutines 之间的通信。

+   **数据传输**：在基本层面上，管道和通道都用于传输数据。在管道中，数据从一个进程流向另一个进程，而在通道中，数据在 goroutines 之间传递。

+   **同步**：两者都提供了一定程度的同步。向满管道写入或从空管道读取将阻塞进程，直到管道被读取或写入。同样，在 Go 中将数据发送到满通道或从空通道接收数据将阻塞 goroutine，直到通道准备好更多数据。

+   **缓冲**：管道和通道都可以进行缓冲。缓冲管道在阻塞或溢出之前有一个定义的容量，同样，Go 通道可以创建带有容量，允许在不立即接收者准备好的情况下保持一定数量的值。

但更重要的是，它们之间也有不同之处：

+   **通信方向**：标准管道是单向的，这意味着它们只允许单向数据流。Go 中的通道默认是双向的，允许在同一个通道上发送和接收数据。

+   **在上下文中的易用性**：通道是 Go 的本地特性，在 Go 程序中提供集成和易用性，这是管道无法比拟的。作为一个系统级特性，管道在 Go 中使用时需要更多的设置和处理。

因此，在我们创建第一个使用管道的 Go 程序之前，请记住以下指南。

在以下场景中使用管道：

+   你必须促进不同进程之间的通信，可能涉及不同的编程语言

+   你的应用程序涉及需要相互通信的独立可执行文件

+   你在一个类 Unix 环境中工作，可以利用强大的 IPC 机制

当以下情况适用时，使用 Go 通道：

+   你正在开发 Go 的并发应用程序，需要在 goroutines 之间进行同步和通信

+   你需要一个简单且安全的方法来处理单个 Go 程序中的并发

+   你必须实现复杂的并发模式，例如扇入、扇出或工作池，Go 的通道和 goroutine 模型可以优雅地处理这些模式

在我们的开发流程中，我们习惯在终端上每次都使用管道。

如前所述，管道将一个命令的输出作为另一个命令的输入。以下是一个简单的`bash`示例：

```go
cat file.txt | grep "flower"
```

在这个命令中，`cat file.txt` 读取 `file.txt` 的内容，然后管道（`|`）将此内容作为输入传递给 `grep "flower"`，它搜索包含 `"flower"` 的行。

要在 Go 中复制整个步骤序列，我们需要读取文件内容然后处理这些内容以找到所需的字符串。

注意

我们不需要使用管道来实现相同的结果，因为 Go 不像 Unix-like 系统那样使用管道；我们通常使用 Go 的文件处理和字符串处理能力来读取和处理数据。

## Golang 中的管道

Go 的标准库提供了创建和管理管道所需的函数。`io.Pipe()` 函数通常用于创建同步的内存管道。当你只需要控制数据的这种流程控制，而不需要执行任何系统调用时，这个函数需要记住。

此外，为了使用操作系统管道，我们可以调用 `os.Pipe()` 函数。这个函数内部使用 `SYS_PIPE2` 系统调用，Go 的 `stdlib` 包为我们处理所有复杂性，返回一个连接的文件对。

在这两种情况下，数据都是通过标准写入操作写入管道的写入端，并通过标准读取操作从读取端读取。确保在数据传输过程中，如管道损坏或数据完整性问题，能够有效地管理。

# 匿名管道的机制

匿名管道是最基本的管道形式。它们用于在父进程和子进程之间进行通信。让我们看看我们如何复制前面提到的简单脚本：

```go
package main
import (
    "fmt"
    "os"
    "os/exec"
)
func main() {
    echoCmd := exec.Command("echo", "Hello, world!")
    grepCmd := exec.Command("grep","Hello")
    pipe, err := echoCmd.StdoutPipe()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error creating StdoutPipe for echoCmd: %v\n", err)
        return
    }
    if err := grepCmd.Start(); err != nil {
        fmt.Fprintf(os.Stderr, "Error starting grepCmd: %v\n", err)
        return
    }
    if err := echoCmd.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error running echoCmd: %v\n", err)
        return
    }
    if err := pipe.Close(); err != nil {
        fmt.Fprintf(os.Stderr, "Error closing pipe: %v\n", err)
        return
    }
    if err := grepCmd.Wait(); err != nil {
        fmt.Fprintf(os.Stderr, "Error waiting for grepCmd: %v\n", err)
        return
    }
}
```

这个程序手动创建了管道以进行进程间通信（IPC）。这是它的工作方式：

1.  创建一个 `echo` 命令及其输出管道：

    ```go
    ```

    echoCmd := exec.Command("echo", "Hello, world!")

    pipe, err := echoCmd.StdoutPipe()

    ```go
    ```

    这设置了一个用于 `"Hello, world!"` 的 `echo` 命令并为其标准输出创建了一个管道。

1.  创建一个 `grep` 命令并设置其标准输入：

    ```go
    ```

    grepCmd := exec.Command("grep", "-i", "HELLO")

    grepCmd.Stdin = pipe

    ```go
    ```

    `grep` 命令被设置为从 `echoCmd` 的输出管道读取。

1.  为 `grepCmd` 输出创建一个管道：

    ```go
    ```

    grepOut, err := grepCmd.StdoutPipe()

    ```go
    ```

    这创建了一个管道来捕获 `grepCmd` 的标准输出。

1.  启动 `grepCmd`：

    ```go
    ```

    if err := grepCmd.Start(); err != nil {

    // 处理错误

    }

    ```go
    ```

    这启动了 `grepCmd` 但不等待它完成。它准备好从其标准输入读取（连接到 `echoCmd` 的输出）。

1.  运行 `echoCmd`：

    ```go
    ```

    if err := echoCmd.Run(); err != nil {

    // 处理错误

    }

    ```go
    ```

    运行 `echoCmd` 将其输出发送到 `grepCmd`。

1.  读取并打印 `grepCmd` 的输出：

    ```go
    ```

    scanner := bufio.NewScanner(grepOut)

    for scanner.Scan() {

    fmt.Println(scanner.Text())

    }

    ```go
    ```

    这段代码逐行读取 `grepCmd` 的输出并打印出来。

1.  等待 `grepCmd` 完成：

    ```go
    ```

    if err := grepCmd.Wait(); err != nil {

    // 处理错误

    }

    ```go
    ```

    最后，它等待 `grepCmd` 完成处理。

我们有更简单的方法达到相同的结果，如下例所示：

```go
package main
import (
    "fmt"
    "os"
    "os/exec"
)
func main() {
    // Create and run the echo command
    echoCmd := exec.Command("echo", "Hello, world!")
    // Capture the output of echoCmd
    echoOutput, err := echoCmd.Output()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error running echoCmd: %v\n", err)
        return
    }
    // Create the grep command with the output of echoCmd as its input
    grepCmd := exec.Command("grep", "Hello")
    grepCmd.Stdin = strings.NewReader(string(echoOutput))
    // Capture the output of grepCmd
    grepOutput, err := grepCmd.Output()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error running grepCmd: %v\n", err)
        return
    }
    // Print the output of grepCmd
    fmt.Printf("Output of grep: %s", grepOutput)
}
```

此程序使用 `Output()` 方法直接执行命令并捕获其输出。以下是逐步解释：

1.  创建一个 `echo` 命令：

    ```go
    ```

    echoCmd := exec.Command("echo", "Hello, world!")

    ```go
    ```

    这行代码创建了一个 `exec.Cmd` 结构体来表示 `"Hello, world!"` 的 `echo` 命令。

1.  运行 `echoCmd` 并捕获其输出：

    ```go
    ```

    echoOutput, err := echoCmd.Output()

    ```go
    ```

    `Output()` 方法运行 `echoCmd`，等待其完成，并捕获其标准输出。如果有错误（例如，如果命令不存在），它将被捕获在 `err` 中。

1.  创建一个 `grep` 命令：

    ```go
    ```

    grepCmd := exec.Command("grep", "Hello")

    ```go
    ```

    这为 `"HELLO"` 的 `grep -i` 命令创建另一个 `exec.Cmd` 结构体。`-i` 标志使搜索不区分大小写。

1.  为 `grepCmd` 设置标准输入：

    ```go
    ```

    grepCmd.Stdin = strings.NewReader(string(echoOutput))

    ```go
    ```

    `echoCmd` 的输出被用作 `grepCmd` 的标准输入。这模仿了在 shell 中的管道行为。

1.  运行 `grepCmd` 并捕获其输出：

    ```go
    ```

    grepOutput, err := grepCmd.Output()

    ```go
    ```

    这执行 `grepCmd` 并捕获其输出。如果 `grepCmd` 遇到错误（例如没有找到匹配项），它将被捕获在 `err` 中。

1.  打印 `grepCmd` 的输出：

    ```go
    ```

    fmt.Printf("Output of grep: %s", grepOutput)

    ```go
    ```

1.  作为最后一步，`grepCmd` 的输出被打印到控制台。

使用 `Output()` 方法的这种做法很方便。它在许多场景中都适用，尤其是在处理简单的命令执行时，只需要捕获命令的输出。

匿名管道有其局限性，因为只有在创建进程或其子进程仍然存活时，它们才对通信有用。此外，我们有一个单向的数据流。为了解决这些问题，我们可以使用命名管道。

# 导航命名管道（Mkfifo()）

与匿名管道不同，命名管道不仅限于活着的进程。它们可以在任何进程之间使用，并持久存在于文件系统中。

进程间通信（IPC）有时可能是一个抽象的概念，对于系统编程新手来说可能难以理解。让我们用一个简单、相关的类比来使这更容易理解：办公室环境中的“任务邮箱”。

想象你在一个办公室里，每个团队成员都有一组特定的任务。沟通和任务委派是这个办公室顺利运行的关键。团队成员如何高效地交换任务和信息？这就是“任务邮箱”概念发挥作用的地方。

在我们的类比中，任务邮箱是办公室中的一个特殊邮箱，团队成员会将任务放入其中供他人处理。一旦任务进入邮箱，指定的团队成员就可以取走、处理并继续处理下一个任务。这个系统确保了每次需要传递任务时，任务都能得到有效沟通和处理，团队成员之间无需直接互动。

现在，让我们将这个类比转换到我们的程序中。由于进程通常需要相互通信，就像办公室里的团队成员一样，这就是命名管道发挥作用的地方。它就像我们的任务邮箱，作为一个不同进程可以交换信息的渠道。一个进程可以将信息放入管道，另一个进程可以取出来进行处理。这是一种简单而有效的方法，可以促进进程间的通信。

为了使这个类比生动起来，让我们创建这个程序。我们将创建一个虚拟的“任务邮箱”（一个命名管道）并演示如何使用它在不同程序的部分之间传递消息（任务）。这个例子将说明命名管道的概念，并使抽象的 IPC 概念更加具体和易于理解。

首先，让我们处理创建我们的命名管道。我们需要验证命名管道是否存在：

```go
func namedPipeExists(pipePath string) bool {
   _, err := os.Stat(pipePath)
   if err == nil {
      return true // The named pipe exists.
   }
   if os.IsNotExist(err) {
      return false // The named pipe does not exist.
   }
   fmt.Println("Error checking named pipe:", err)
   return false
}
```

在我们的`main()`函数中，我们确保在不存在时创建一个命名管道。`Mkfifo()`函数在文件系统中创建一个命名管道：

```go
// Check if the mailbox exists
if !namedPipeExists(mailboxPath) {
   fmt.Println("The mailbox does not exist.")
   // Set up the mailbox (named pipe)
   fmt.Println("Creating the task mailbox...")
   if err := unix.Mkfifo(mailboxPath, 0666); err != nil {
      fmt.Println("Error setting up the task mailbox:", err)
      return
   }
}
```

一旦创建，使用`os.OpenFile`和`os.O_RDWR`打开管道进行读取。这样，发送的数据就是从管道中读取的：

```go
// Open the named pipe for read and write
mailbox, err := os.OpenFile(mailboxPath, os.O_RDWR, os.ModeNamedPipe)
if err != nil {
   fmt.Println("Error opening named pipe:", err)
}
defer mailbox.Close()
```

现在，我们的主要逻辑驻留在发送任务通过管道的一个 goroutine 中，而另一个 goroutine 读取它们。一旦我们使用扫描器，当发送者发送一个`"EOD"`（日终）字符串实例时，我们就停止读取新任务。为了同步这些 goroutine，我们使用`sync.WaitGroup`：

```go
wg := &sync.WaitGroup{}
wg.Add(2)
go func() {
   defer wg.Done()
   ReadTask(mailbox)
}()
go func() {
   defer wg.Done()
   i := 0
   for i < 10 {
      SendTask(mailbox, fmt.Sprintf(«Task %d\n», i))
      i++
   }
   // Close the mailbox
   SendTask(mailbox, "EOD\n")
   fmt.Println("All tasks sent.")
}()
wg.Wait()
```

在`writer.go`文件中的发送逻辑只是将数据推送到管道：

```go
func SendTask(pipe *os.File, data string) error {
   _, err := pipe.WriteString(data)
   if err != nil {
      return fmt.Errorf("error writing to named pipe: %v", err)
   }
   return nil
}
```

在`reader.go`文件中的`ReadTask()`函数负责接收任务：

```go
func ReadTask(pipe *os.File) error {
   fmt.Println("Reading tasks from the mailbox...")
   scanner := bufio.NewScanner(pipe)
   for scanner.Scan() {
      task := scanner.Text()
      fmt.Printf("Processing task: %s\n", task)
      if task == "EOD" {
         break
      }
   }
   if err := scanner.Err(); err != nil {
      return fmt.Errorf("error reading tasks from the mailbox: %v", err)
   }
   fmt.Println("All tasks processed.")
   return nil
}
```

运行我们的程序，我们应该看到以下输出：

```go
All tasks sent.
Reading tasks from the mailbox...
Processing task: Task 0
Processing task: Task 1
Processing task: Task 2
Processing task: Task 3
Processing task: Task 4
Processing task: Task 5
Processing task: Task 6
Processing task: Task 7
Processing task: Task 8
Processing task: Task 9
Processing task: EOD
All tasks processed.
```

使用命名管道有几个重要的特性。例如，它们可以在任何进程之间使用。它们独立于进程存在，可以在文件系统中找到。此外，尽管单个命名管道是单向的，但两个命名管道可以用于双向通信。

# 最佳实践 - 使用管道的指南

在探讨了使用管道进行进程间通信（IPC）的实际方面之后，讨论最佳实践和指南至关重要。遵循这些原则确保您的实现既高效又安全且易于维护。

## 高效数据处理

在高效数据处理的环境中，尤其是在最小化传输中的数据时，采用了两种关键策略：分块和压缩。

分块涉及将大型数据集分解成更小、更易于管理的部分。分块的主要优势是防止管道缓冲区溢出，这可能导致数据传输中的瓶颈。通过分段数据，每个块可以依次处理和传输，确保数据流更加顺畅和高效。这种技术在数据流或实时处理数据的情况下特别有用。

### 示例 - 分块数据

在此代码片段中，思想是写端按块大小发送数据，而读端接收相同的数据。

写端代码如下所示：

```go
func writeInChunks(pipe *os.File, data []byte, chunkSize int) error {
   for i := 0; i < len(data); i += chunkSize {
      end := i + chunkSize
      if end > len(data) {
         end = len(data)
      }
      chunk := data[i:end]
      _, err := pipe.Write(data[i:end])
      if err != nil {
         return err
      }
      writer.Flush() // Ensure chunk is written
   }
   return nil
}
```

在读端，代码如下所示：

```go
// Read chunks from the named pipe
    for {
        chunk, err := reader.ReadBytes('\n') // Assuming chunks are newline-separated
        if err != nil {
            if err == io.EOF {
                break // End of file reached
            }
            panic(err)
        }
        fmt.Printf("Received chunk: %s\n", string(chunk))
    }
```

压缩是在发送数据之前减小数据大小的过程。当数据高度可压缩时，例如文本文件或某些类型的图像和视频文件，这尤其有益。通过压缩数据，需要传输的信息量显著减少，从而加快传输速度，并可能降低带宽使用。然而，考虑压缩和解压缩数据的计算开销以及数据本身的性质（某些数据可能不易压缩）是很重要的。

### 示例 – 压缩数据

对于压缩，您可以使用`compress/gzip`之类的库来压缩和解压缩数据。

在这些代码片段中，写端压缩数据，而读端解压缩数据以读取。

在以下代码片段中，我们正在压缩并发送数据：

```go
   // Create a gzip writer on top of the named pipe
     gzipWriter := gzip.NewWriter(fifo)
  // Example data to compress and write
    data := []byte("Some data to be compressed and written to the pipe")
    // Write compressed data to the named pipe
    if _, err := gzipWriter.Write(data); err != nil {
        panic(err)
    }
    gzipWriter.Flush() // Ensure data is written
```

因此，对于阅读，我们还需要解压缩数据：

```go
// Create a gzip reader
gzipReader, err := gzip.NewReader(fifo)
if err != nil {
   // handler errors
}
defer gzipReader.Close()
// Read and decompress data from the named pipe
var buf bytes.Buffer
io.Copy(&buf, gzipReader)
```

## 错误处理和资源管理

我们必须处理错误并妥善保存资源，以创建可维护和健壮的软件。让我们探讨如何接近这两个维度的健壮性。

### 强健的错误处理

在管道操作后始终检查错误。这包括读取、写入和关闭操作。此外，为读取/写入操作实现超时以避免死锁。

#### 示例 – 使用超时读取管道

在此代码片段中，我们有用于利用上下文超时读取管道的样板代码：

```go
timeout := time.After(5 * time.Second)
done := make(chan bool)
go func() {
    _, err := pipe.Read(buffer)
    // Handle read operation and error
    done <- true
}()
select {
case <-timeout:
    // Handle timeout, e.g., close pipe, log error
case <-done:
    // Read operation completed
}
```

### 正确的资源管理

确保在使用后正确关闭管道。在 Go 中使用`defer`关闭文件描述符。

在以下代码片段中，我们可以观察到我们可以避免资源泄漏：

```go
pipeReader, pipeWriter, _ := os.Pipe()
defer pipeReader.Close()
defer pipeWriter.Close()
// Perform pipe operations
```

处理泄漏

监控任何资源泄漏。如果管道保持打开状态，可能会导致文件描述符耗尽。

## 安全考虑

当我们传输敏感数据时，我们应该在通过管道发送之前对其进行加密。在通过管道接收数据后，我们需要确保数据的验证，尤其是在程序的关键部分使用时。

在创建命名管道时，我们还需要谨慎处理权限。仅限制对受信任用户的访问。此外，使用随机或不可预测的名称来防止针对命名管道的域名抢注攻击。

#### 示例 – 保护命名管道创建

在以下代码片段中，管道名称接收一个随机因子并限制对管道所有者的访问：

```go
pipePath := "/tmp/my_secure_pipe_" + randomString(10)
syscall.Mkfifo(pipePath, 0600) // Restricts access to the owner only
```

域名抢注攻击

在域名抢注攻击中，攻击者创建一个预期将被合法应用程序或服务使用的命名管道。这种攻击通常针对动态创建命名管道进行 IPC 的应用程序或服务，但它们没有充分验证管道创建者的身份。

## 性能优化

根据应用程序的需求调整缓冲区大小。较小的缓冲区可以减少内存使用，而较大的缓冲区可以提高吞吐量。

以下实践对于实现良好的性能至关重要：使用非阻塞 I/O 操作来提高性能，尤其是在需要高响应性的应用程序中。

通过遵循这些最佳实践，您可以确保在 Go 中使用命名管道不仅有效，而且安全且易于维护。命名管道是系统编程中的强大工具，通过仔细考虑这些指南，您可以充分利用它们的潜力来构建健壮且高效的应用程序。随着您在 Go 和系统编程技能的提升，请记住这些实践以提升代码质量。

# 开发日志处理工具

在介绍了 IPC 中管道的基础知识和在 Go 中使用它们的最佳实践之后，让我们探索更多高级主题。我们将探讨一个管道可以有效地利用的场景，并看看 Go 的并发模型如何补充这些用例。本节旨在为您提供利用管道进行复杂系统编程任务的实用见解。

在下一个示例中，我们将开发一个简单的实时日志处理工具。这个工具将从文件中读取日志数据（模拟另一个进程写入的日志文件），处理日志条目（例如，基于严重性进行过滤），然后将结果输出到控制台。

首先，我们创建了一个`filterLogs()`函数，它从读取器读取日志，过滤它们，并将它们写入写入器：

```go
func filterLogs(reader io.Reader, writer io.Writer) {
   scanner := bufio.NewScanner(reader)
   for scanner.Scan() {
      logEntry := scanner.Text()
      if strings.Contains(logEntry, "ERROR") {
         writer.Write([]byte(logEntry + "\n"))
      }
   }
}
```

注意，该函数从读取器（我们的命名管道）读取，仅过滤包含`"ERROR"`的日志条目，并将它们写入写入器（我们发送到标准输出）：

```go
func main() {
   // Create a named pipe (simulating a log file)
   pipePath := "/tmp/my_log_pipe"
   if err := os.RemoveAll(pipePath); err != nil {
      panic(err)
   }
   if err := os.Mkfifo(pipePath, 0600); err != nil {
      panic(err)
   }
   defer os.RemoveAll(pipePath)
   // Open the named pipe for reading
   pipeFile, err := os.OpenFile(pipePath, os.O_RDONLY|os.O_CREATE, os.ModeNamedPipe)
   if err != nil {
      panic(err)
   }
   defer pipeFile.Close()
   // Start a goroutine to simulate log writing
   go func() {
      writer, err := os.OpenFile(pipePath, os.O_WRONLY, os.ModeNamedPipe)
      if err != nil {
         panic(err)
      }
      defer writer.Close()
      for {
         writer.WriteString("INFO: All systems operational\n")
         writer.WriteString("ERROR: An error occurred\n")
         time.Sleep(1 * time.Second)
      }
   }()
   // Process the logs
   filterLogs(pipeFile, os.Stdout)
}
```

在`main()`函数中，创建了一个命名管道来模拟日志文件。这个管道作为日志数据的来源。管道被打开用于读取。同时，启动了一个 goroutine 来模拟将日志条目写入这个管道，包括`"INFO"`和`"ERROR"`消息。调用`filterLogs()`函数来处理传入的日志数据。它过滤并输出错误消息。

虽然简单，但这段代码展示了在 Go 中使用管道进行实时日志处理的实际应用。它展示了如何设置一个用于连续数据处理的管道，模拟了系统监控和日志分析工具中的常见场景。

# 摘要

在我们结束本章之前，让我们回顾一下我们在系统编程中关于 IPC 获得的关键见解和知识，特别是在 Go 的上下文中。

我们探讨了它们在促进进程间数据交换中的基本作用，强调了它们在系统级编程中的重要性。这些管道有广泛的应用，包括命令行工具、数据流和进程间数据交换。我们还比较了管道和通道，突出了它们在用法上的差异。

在接下来的章节中，我们将应用所获得的知识来创建自动化。
