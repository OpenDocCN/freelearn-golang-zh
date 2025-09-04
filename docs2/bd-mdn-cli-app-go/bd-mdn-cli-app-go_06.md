

# 调用外部进程并处理错误和超时

许多命令行应用程序会与其他外部命令或 API 服务交互。本章将指导你如何调用这些外部进程，以及当它们发生时如何处理超时和其他错误。本章将从深入探讨`os/exec`包开始，该包包含了创建调用外部进程的命令所需的一切，它为你提供了创建和运行命令的多种选项。你将学习如何从标准输出和标准错误管道中检索数据，以及为类似用途创建额外的文件描述符。

另一个外部进程涉及调用外部 API 服务端点。`net/http`包被讨论，这是我们开始定义客户端的地方，然后创建它执行的请求。我们将讨论请求可以创建和执行的不同方式。

当调用任何类型的进程时，都可能发生超时和其他错误。我们将以探讨如何在我们的代码中捕获超时和错误的发生结束本章。重要的是要意识到这些事情可能发生，因此编写能够处理它们的代码很重要。错误发生时采取的具体操作取决于用例，因此我们只将讨论捕获这些情况的代码。总结来说，我们将涵盖以下主题：

+   调用外部进程

+   与 REST API 交互

+   处理预期的 – 超时和其他错误

# 技术要求

为了理解并运行本章中分享的示例，你需要一个 UNIX 操作系统。

你也可以在 GitHub 上找到代码示例，链接为[`github.com/PacktPublishing/Building-Modern-CLI-Applications-in-Go/tree/main/Chapter06`](https://github.com/PacktPublishing/Building-Modern-CLI-Applications-in-Go/tree/main/Chapter06)。

# 调用外部进程

在你的命令行应用程序中，你可能需要调用一些外部进程。有时，第三方工具提供了作为包装器的 Golang 库。例如，Go CV，[`gocv.io/`](https://gocv.io/)，是针对 OpenCV（一个开源计算机视觉库）提供的 Golang 包装器。然后是 GoFFmpeg，[`github.com/xfrr/goffmpeg`](https://github.com/xfrr/goffmpeg)，它是针对 FFmpeg（一个用于录制、转换和流式传输音频和视频文件的库）提供的包装器。通常，你需要安装底层工具，如 OpenCV 或 FFmpeg，然后库与它交互。调用这些外部进程意味着导入包装器包，并在你的代码中调用其方法。通常，当你深入研究代码时，你会发现这些库为 C 代码提供了包装器。

除了导入外部工具的包装器之外，你也可以使用`os/exec` Golang 库来调用外部应用程序。这是库的主要目的，在本节中，我们将深入探讨如何使用它来调用外部应用程序。

首先，让我们通过每个示例来回顾一下 `os/exec` 包中存在的变量、类型和函数。

## os/exec 包

通过深入研究 `exec` 包，你会发现它是对 `os.StartProcess` 方法的包装，这使得处理标准输入和标准输出的重映射、通过管道连接输入和输出以及处理其他修改变得更加容易。

为了清晰起见，重要的是要注意，这个包不会调用操作系统的 shell，因此不处理通常由 shell 处理的任务：展开全局模式、管道或重定向。如果需要展开全局模式，可以直接调用 shell 并确保转义值以使其安全，或者也可以使用路径或文件路径的 `Glob` 函数。要展开字符串中存在的任何环境变量，请使用 `os` 包的 `ExpandEnv` 函数。

在以下小节中，我们将开始讨论 `os/exec` 包中存在的不同变量、类型、函数和方法。

### 变量

`ErrNotFound` 是当在应用程序的 `$``PATH` 变量中找不到可执行文件时返回的错误变量。

### 类型

`Cmd` 是一个表示外部命令的结构体。定义这种类型的变量只是为了准备运行命令。一旦通过 `Run`、`Output` 或 `CombinedOutput` 方法运行了 `Cmd` 类型的变量，它就不能再被重用。这个 `Cmd` 结构体上还有几个我们可以详细说明的字段：

+   `Path string` 这是唯一必需的字段。它是要运行的命令的路径；如果路径是相对的，那么它将相对于存储在 `Dir` 字段中的值。

+   `Args []string` 这个字段包含命令的参数。`Args[0]` 表示命令。`Path` 和 `Args` 在运行命令时设置，但如果 `Args` 是 `nil` 或空的，则在执行期间只使用 `{Path}`。

+   `Env []string` `Env` 字段表示要运行的命令的环境。切片中的每个值都必须以下列格式：`"key=value"`。如果值为空或 `nil`，则命令使用当前环境。如果切片有重复的键值，则使用重复键的最后一个值。

+   `Dir string` `Dir` 字段表示命令的工作目录。如果没有设置，则使用当前目录。

+   `Stdin io.Reader` `Stdin` 字段指定了命令进程的标准输入。如果数据是 `nil`，则进程从 `os.DevNull`（空设备）读取。然而，如果标准输入是 `*os.File`，则内容将通过管道传输。在执行过程中，一个 goroutine 从标准输入读取，然后将该数据发送到命令。`Wait` 方法将在 goroutine 开始复制之前不会完成。如果它没有完成，那么可能是因为 **文件结束**（**EOF**）、读取或写入管道错误。

+   `Stdout io.Writer` `Stdout` 字段指定了命令进程的标准输出。如果标准输出是 `nil`，则进程连接到 `os.DevNull` 空设备。如果标准输出是 `*os.File`，则输出将发送到它。在执行过程中，一个 goroutine 从命令进程读取并发送数据到写入器。

+   `Stderr io.Writer` `Stderr` 字段指定了命令进程的标准错误输出。如果标准错误是 `nil`，则进程连接到 `os.DevNull` 空设备。如果标准错误是 `*os.File`，则错误输出将发送到它。在执行过程中，一个 goroutine 从命令进程读取并发送数据到写入器。

+   `ExtraFiles []*os.File` `ExtraFiles` 字段指定了命令进程继承的附加文件。它不包括标准输入、标准输出或标准错误，因此如果非空，条目 *x* 成为 *3+x* 文件描述符。此字段在 Windows 上不受支持。

+   `SysProcAttr *syscall.SysProcAttr` `SysProcAttr` 保留系统特定的属性，这些属性作为 `os.ProcAttr` 的 `Sys` 字段传递给 `os.StartProcess`。

+   `Process *os.Process` `Process` 字段在命令运行后持有底层进程。

+   `ProcessState *os.ProcessState` `ProcessState` 字段包含有关进程的信息。在调用等待或运行方法后变得可用。

### 方法

以下是在 `exec.Cmd` 对象上存在的方法：

+   `func (c *Cmd) CombinedOutput() ([]byte, error)` `CombinedOutput` 方法将标准输出和标准错误都返回到 1 字节字符串输出中。

+   `func (c *Cmd) Output ([]byte, error)` `Output` 方法仅返回标准输出。如果发生错误，它通常将是 `*ExitError` 类型，如果命令的标准错误 `c.Stderr` 是 `nil`，则 `Output` 将填充 `ExitError.Stderr`。

+   `func (c *Cmd) Run() error` `Run` 方法开始执行命令并等待其完成。如果没有问题复制标准输入、标准输出或标准错误，并且命令以零状态退出，则返回的错误将是 `nil`。如果命令以错误退出，它通常将是 `*ExitError` 类型，但也可能是其他错误类型。

+   `func (c *Cmd)` `Start() error`

+   `Start` 方法将启动执行命令而不会等待其完成。如果 `Start` 方法运行成功，那么 `c.Process` 字段将被设置。然后 `c.Wait` 字段将返回退出代码并在完成后释放资源。

+   `func (c* Cmd) StderrPipe() (io.ReadCloser, error)` `StderrPipe` 返回一个连接到命令标准错误的管道。不需要关闭管道，因为 `Wait` 方法将在命令退出时关闭管道。不要在所有从标准错误管道读取完成之前调用 `Wait` 方法。不要与 `Run` 方法一起使用此命令，原因相同。

+   `func (c* Cmd) StdinPipe() (io.WriteCloser, error`) `StdinPipe`返回一个连接到命令标准输入的管道。在`Wait`之后，管道将被关闭，并且命令将退出。然而，有时命令将不会运行，直到标准输入管道被关闭，因此您可以调用`Close`方法来提前关闭管道。

+   `func (c *Cmd) StdoutPipe() (io.ReadCloser, error`) `StdoutPipe`方法返回一个连接到命令标准输出的管道。不需要关闭管道，因为`Wait`将在命令退出时关闭管道。同样，不要在所有从标准输出管道读取完成之前调用`Wait`。不要使用此命令与`Run`方法，原因相同。

+   `func (c *Cmd) String() string` `String`方法返回一个人类可读的命令描述，`c`，用于调试目的。具体的输出可能在不同版本的 Go 发布之间有所不同。此外，不要将其用作 shell 的输入，因为它不适合这个目的。

+   `func (c *Cmd) Wait() error` `Wait`方法等待任何复制到标准输入、标准输出或标准错误完成，以及命令退出。要使用`Wait`方法，命令必须是由`Start`方法而不是`Run`方法启动的。如果没有复制管道的错误，并且进程以`0`退出状态码退出，则返回的错误将是`nil`。如果命令的`Stdin`、`Stdout`或`Stderr`字段未设置为`*os.File`，则`Wait`还确保相应的输入-输出循环过程也完成。

`Error`是一个结构体，表示当`LookPath`函数无法识别文件为可执行文件时返回的错误。我们将详细定义这个特定错误类型的几个字段和方法。

以下是在`Error`类型上存在的几种方法：

+   `func (e *Error) Unwrap() error` 如果返回的错误是错误链，则可以使用`Unwrap`方法来*展开*它并确定它是哪种错误。

`ExitError`是一个结构体，表示命令未成功退出时的错误。`*os.ProcessState`被嵌入到这个结构体中，因此所有值和字段也将对`ExitError`类型可用。最后，我们可以更详细地定义这种类型的几个字段：

+   `Stderr []byte` 此字段保存了如果没有从`Cmd.Output`方法收集的标准错误输出响应的集合。如果错误输出足够长，`Stderr`可能只包含前缀和后缀。中间将包含关于省略的字节数的文本。为了调试目的，如果您想包含整个错误消息，请重定向到`Cmd.Stderr`。

以下是在`ExitError`类型上存在的几种方法：

+   `func (e *ExitError) Error() string` `Error`方法返回表示退出错误的字符串。

### 函数

以下是在`os/exec`包中存在的函数：

+   `func LookPath(file string) (string, error)` `LookPath`函数检查文件是否是可执行的并且可以找到。如果文件是相对路径，则相对于当前目录。

+   `func Command(name string, arg ...string) *Cmd` `Command`函数返回一个只设置了路径和参数的`Cmd`结构体。如果`name`有路径分隔符，则使用`LookPath`函数来确认文件已找到且可执行。否则，直接使用`name`作为路径。此函数在 Windows 上的行为略有不同。例如，它将整个命令行作为一个单独的字符串执行，包括引号内的参数，然后处理自己的解析。

+   `func CommandContext(ctx context.Context, name string, arg ...string) *Cmd` 与`Command`函数类似，但接收上下文。如果上下文在命令完成之前执行，则将通过调用`os.Process.Kill`来杀死进程。

现在我们已经深入研究了`os/exec`包以及执行函数所需的 struct、函数和方法，让我们实际在代码中使用它们来执行外部函数。让我们使用`Cmd`结构体创建命令，同时使用`Command`和`CommandContext`函数。然后我们可以取一个示例命令，使用`Run`、`Output`或`CombinedOutput`方法之一来运行它。最后，我们将处理这些方法通常返回的一些错误。

注意

如果你想跟随即将出现的示例，在`Chapter-6`仓库中，安装必要的应用程序。在 Windows 上，使用`.\build-windows.p1` PowerShell 脚本。在 Darwin 上，使用`make install`命令。一旦应用程序安装完毕，运行`go run main.go`。

## 使用 Cmd 结构体创建命令

创建命令有几种不同的方式。第一种方式是在`exec`包中的`Cmd`结构体。

### 使用 Cmd 结构体

我们首先使用未设置的`Cmd`结构体定义`cmd`变量。以下代码位于`/examples/command.go`中的`CreateCommandUsingStruct`函数：

```go
cmd := exec.Cmd{}
```

每个字段都是单独设置的。路径是通过`filepath.Join`设置的，这在不同的操作系统上都是安全的：

```go
cmd.Path = filepath.Join(os.Getenv("GOPATH"), "bin", "uppercase")
```

每个字段都是单独设置的。`Args`字段包含命令名，位于`Args[0]`位置，后面跟其余要传递的参数：

```go
cmd.Args = []string{"uppercase", "hack the planet"}
```

以下三个文件描述符被设置 - `Stdin`、`Stdout`和`Stderr`：

```go
cmd.Stdin = os.Stdin // io.Reader
cmd.Stdout = os.Stdout // io.Writer
cmd.Stderr = os.Stderr // io.Writer
```

然而，有一个`writer`文件描述符被传递到`ExtraFiles`字段。这个特定的字段被命令进程继承。需要注意的是，如果不将 writer 传递到`ExtraFiles`，管道将无法工作，因为子进程必须获取 writer 才能写入它：

```go
reader, writer, err := os.Pipe()
if err != nil {
    panic(err)
}
cmd.ExtraFiles = []*os.File{writer}
if err := cmd.Start(); err != nil {
    panic(err)
}
```

在实际调用的命令中，`cmd/uppercase/uppercase.go` 中有代码，它接受命令名之后的第一个参数并将其转换为大写。然后新的大写文本被编码到管道或额外的文件描述符中：

```go
input := os.Args[1:]
output := strings.ToUpper(strings.Join(input, ""))
pipe := os.NewFile(uintptr(3), "pipe")
err := json.NewEncoder(pipe).Encode(output)
if err != nil {
    panic(err)
}
```

回到 `CreateCommandUsingStruct` 函数，现在可以通过管道的 `read` 文件描述符读取编码到管道中的值，然后使用以下代码输出：

```go
var data string
decoder := json.NewDecoder(reader)
if err := decoder.Decode(&data); err != nil {
    panic(err)
}
fmt.Println(data)
```

我们现在知道了一种使用 `Cmd` 结构体创建命令的方法。所有内容都可以在初始化命令的同时一次性定义，这取决于您的偏好。

### 使用 Command 函数

创建命令的另一种方式是使用 `exec.Command` 函数。以下代码位于 `/examples/command.go` 中的 `CreateCommandUsingCommandFunction`：

```go
cmd := exec.Command(filepath.Join(os.Getenv("GOPATH"), "bin", "uppercase"), "hello world")
reader, writer, err := os.Pipe()
if err != nil {
    panic(err)
}
```

`exec.Command` 函数将命令的文件路径作为第一个参数。可选地传递一个表示参数的字符串切片作为剩余参数。函数的其余部分相同。因为 `exec.Command` 不接受任何额外的参数，所以我们同样在原始变量初始化之外定义了 `ExtraFiles` 字段。

## 运行命令

现在我们知道了如何创建命令，有多种不同的方式来运行或开始运行一个命令。虽然这些方法在本节前面已经详细描述过，但我们现在将分享使用每个方法的示例。

### 使用 Run 方法

如前所述，`Run` 方法启动命令进程，然后等待其完成。此代码从 `main.go` 文件中调用，但可以在 `/examples/running.go` 中找到。在这个例子中，我们调用了一个名为 `lettercount` 的不同命令，该命令计算字符串中的字母数，然后打印出结果：

```go
cmd := exec.Command(filepath.Join(os.Getenv("GOPATH"), "bin", "lettercount"), "four")
cmd.Stdin = os.Stdin
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
var count int
```

再次，我们使用 `ExtraFiles` 字段传递一个额外的文件描述符以写入结果：

```go
reader, writer, err := os.Pipe()
if err != nil {
    panic(err)
}
cmd.ExtraFiles = []*os.File{writer}
if err := cmd.Run(); err != nil {
    panic(err)
}
if err := json.NewDecoder(reader).Decode(&count); err != nil {
    panic(err)
}
```

最终使用以下代码打印结果：

```go
fmt.Println("letter count: ", count)
```

### 使用 Start 命令

`Start` 方法类似于 `Run` 方法；然而，它不会等待进程完成。您可以在 `examples/running.go` 中找到使用 `Start` 命令的代码。大部分是相同的，但您将替换包含 `cmd.Run` 的代码块，如下所示：

```go
if err := cmd.Start(); err != nil {
    panic(err)
}
err = cmd.Wait()
if err != nil {
    panic(err)
}
```

调用 `cmd.Wait` 方法非常重要，因为它释放了命令进程占用的资源。

### 使用 Output 命令

如方法名所示，`Output` 方法返回所有已通过标准输出管道传入的内容。将命令推送到标准输出管道的最常见方式是通过 `fmt` 包中的任何打印方法。为 `lettercount` 命令在 `main` 函数的末尾添加了一行：

```go
fmt.Printf("successfully counted the letters of \"%v\" as %d\n", input, len(runes))
```

在使用此 `Output` 方法的代码中，唯一的区别可以在 `examples/running.go` 文件下的 `OutputMethod` 函数中找到，即这一行代码：

```go
out, err := cmd.Output()
```

`out`变量是一个字节切片，稍后可以将其转换为字符串以打印输出。这个变量捕获标准输出，当函数运行时，显示的输出如下：

```go
output: successfully counted the letters of "four" as 4
```

### 使用 CombinedOutput 命令

如方法名所示，`CombinedOutput`方法返回标准输出和标准错误管道数据的组合输出。在`lettercount`命令的`main`函数末尾添加一行：

```go
fmt.Fprintln(os.Stderr, "this is where the errors go")
```

与上一个函数的调用和当前函数`CombinedOutputMethod`之间的唯一重大区别是这一行：

```go
CombinedOutput, err := cmd.CombinedOutput()
```

同样，它返回一个字节切片，但现在包含标准错误和标准输出的组合输出。

### 在 Windows 上执行命令

与示例并列的是以`_windows.go`结尾的类似文件。在先前的示例中，需要注意的主要事项是`ExtraFiles`在 Windows 上不受支持。这些针对 Windows 的特定和简单示例执行了一个指向`google.com`的外部`ping`命令。让我们看一下其中一个：

```go
func CreateCommandUsingCommandFunction() {
    cmd := exec.Command("cmd", "/C", "ping", "google.com")
    output, err := cmd.CombinedOutput()
    if err != nil {
        panic(err)
    }
    fmt.Println(string(output))
}
```

就像我们为 Darwin 编写的命令一样，我们可以使用`exec.Command`函数或结构体来创建命令，并调用`Run`、`Start`、`Wait`、`Output`和`CombinedOutput`等操作。

此外，对于分页，Linux 和 UNIX 机器上使用`less`，而 Windows 上使用`more`。让我们快速展示这段代码：

```go
func Pagination() {
    moreCmd := exec.Command("cmd", "/C", "more")
    moreCmd.Stdin = strings.NewReader(blob)
    moreCmd.Stdout = os.Stdout
    moreCmd.Stderr = os.Stderr
    err := moreCmd.Run()
    if err != nil {
        panic(err)
    }
}
var (
    blob = `
    …
    `
)
```

同样，我们可以使用`exec.Command`方法传递名称和所有参数。我们还把长文本传递到`moreCmd.Stdin`字段。

因此，`os/exec`包提供了创建和运行外部命令的不同方式。无论您是使用`exec.Command`方法创建一个快速命令，还是直接使用`exec.Cmd`结构体创建一个命令然后运行`Start`命令，您都有选择。最后，您可以分别或一起检索标准输出和错误输出。了解`os/exec`包将使您能够轻松地从 Go 命令行应用程序成功运行外部命令。

# 与 REST API 交互

通常，如果公司或用户已经创建了一个 API，命令行应用程序将向 REST API 或 gRPC 端点发送请求。让我们首先谈谈使用 REST API 端点。了解`net/http`包非常重要。这是一个相当大的包，包含许多类型、方法和函数，其中许多用于服务器端开发。在这种情况下，命令行应用程序将是 API 的客户端，因此我们不会详细讨论每个部分。不过，我们将从客户端的角度探讨一些基本用例。

## GET 请求

让我们回顾一下*第三章*中的代码，*构建音频元数据 CLI*。在`/cmd/cli/command/get.go`文件中 CLI 命令代码的`Run`命令中，有一个调用相应 API 请求端点的代码片段，使用的是`GET`方法：

```go
params := "id=" + url.QueryEscape(cmd.id)
path := fmt.Sprintf("http://localhost/request?%s", params)
payload := &bytes.Buffer{}
method := "GET"
client := cmd.client
```

注意，在前面的代码中，我们取了字段值`id`，它已经在`cmd`变量上设置，并将其作为参数传递到 HTTP 请求中。考虑要传递的标志和参数，这些参数将用作 HTTP 请求的参数。以下代码执行请求：

```go
req, err := http.NewRequest(method, path, payload)
if err != nil {
    return err
}
resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()
```

最后，响应被读取到一个字节字符串中并打印出来。在访问响应体之前，检查响应或体是否为`nil`。这可以让你避免一些未来的麻烦：

```go
b, err := io.ReadAll(resp.Body)
if err != nil {
    return err
}
fmt.Println(string(b))
return nil
```

然而，在现实中，会有更多的事情要做：

1.  如果返回`200` `OK`，则我们可以返回输出，因为它是一个成功的响应。否则，在下一节“处理预期的 – 超时和错误”中，我们将讨论如何处理其他响应。

1.  **记录响应**：如果我们认为响应不包含任何敏感数据，理想情况下我们会记录响应。这些详细信息可以写入日志文件或在详细模式下输出。

1.  **存储响应**：有时，响应可能会存储在本地数据库或缓存中。

1.  标头中的`Content-Type`设置为`application/json`，我们将 JSON 响应反序列化到结构体中。

目前，在 audiofile 应用程序中，我们将数据转换成如下`Audio`结构体：

```go
var audio Audio
If err := json.Unmarshal(b, &audio); err != nil {
    fmt.Println("error unmarshalling JSON response"
}
```

但如果响应体不是 JSON 格式，而是其他内容类型呢？在一个完美的世界里，我们会有一份 API 文档，它会告诉我们预期的内容，这样我们就可以相应地处理它。或者，你可以使用以下方法先检查确认类型：

```go
contentType := http.DetectContentType(b) // b are the bytes from reading the resp.Body
```

在互联网上快速搜索 HTTP 内容类型将返回一个长长的列表。在前面的示例中，音频公司可能已经决定返回一个`Content-Type`值为`audio/wave`。在这种情况下，我们既可以下载也可以流式传输结果。`net/http`包中定义了不同的 HTTP 方法类型作为常量：

+   `MethodGet`: 用于请求数据

+   `MethodPost`: 用于插入数据

+   `MethodPut`: 请求是幂等的，用于插入或更新整个资源

+   `MethodPatch`: 与`MethodPut`类似，但只发送部分数据以更新，而不修改整个资源

+   `MethodDelete`: 用于删除或移除数据

+   `MethodConnect`: 在与代理通信时使用，当 URI 以`https://`开头时

+   `MethodOptions`: 用于描述与目标之间的通信选项或允许的方法

+   `MethodTrace`: 通过提供目标路径上的消息回环用于调试

数据返回的方法类型和内容类型有很多可能性。在前面的`Get`示例中，我们使用客户端的`Do`方法调用该方法。另一种选择是使用`http.Get`方法。如果我们使用该方法，那么我们将使用以下代码来执行请求：

```go
resp, err := http.Get(path)
if err != nil {
    return err
}
defer resp.Body.Close()
```

类似地，与其使用`client.Do`方法进行 POST 操作或提交表单，不如使用特定的`http.Post`和`http.PostForm`方法。有时，一个方法更适合你所做的事情。在这个时候，重要的是要了解你的选项。

## 分页

假设请求返回了大量的数据。与其一次性接收所有数据而使客户端过载，通常分页是一个选项。调用中可以传递两个作为参数的字段：

+   `Limit`：要返回的对象数量

+   `Page`：返回多页结果的游标

我们可以内部定义这些，然后按照以下方式构建路径：

```go
path := fmt.Sprintf("http://localhost/request?limit=%d&page=%d", limit, page)
```

如果你使用外部 API，请确保使用适当的分页和用法参数构建它们的文档。这只是一个通用示例。实际上，还有几种其他实现分页的方法。你可以在循环中发送额外的请求，增加页面数，直到检索到所有数据。

然而，从命令行方面，你可以在分页后返回所有数据，也可以在 CLI 端处理分页。在从 HTTP `Get`请求收集了大量数据后，在客户端处理分页的一种方法是将数据管道化。这些数据可以被管道化到操作系统的分页命令中。对于 UNIX，`less`是分页命令。我们创建命令，然后将字符串输出管道化到`Stdin`管道。这段代码可以在`examples/pagination.go`文件中找到。类似于我们分享的其他示例，在创建命令时，我们创建一个管道并将写入器作为额外的文件描述符传递给命令，以便可以写出数据：

```go
pagesCmd := exec.Command(filepath.Join(os.Getenv("GOPATH"), "bin", "pages"))
reader, writer, err := os.Pipe()
if err != nil {
    panic(err)
}
pagesCmd.Stdin = os.Stdin
pagesCmd.Stdout = os.Stdout
pagesCmd.Stderr = os.Stderr
pagesCmd.ExtraFiles = []*os.File{writer}
if err := pagesCmd.Run(); err != nil {
    panic(err)
}
```

再次，从读取器中解码的数据被编码到`data` `string`变量中：

```go
var data string
decoder := json.NewDecoder(reader)
if err := decoder.Decode(&data); err != nil {
    panic(err)
}
```

然后，这个字符串被传递到`Strings.NewReader`方法中，并定义为`less` UNIX 命令的输入：

```go
lessCmd := exec.Command("/usr/bin/less")
lessCmd.Stdin = strings.NewReader(data)
lessCmd.Stdout = os.Stdout
err = lessCmd.Run()
if err != nil {
    panic(err)
}
```

当命令运行时，数据以页面的形式输出。然后用户可以按空格键继续到下一页，或者使用任何命令键来导航数据输出。

## 速率限制

经常在处理第三方 API 时，特定时间内可以处理多少请求是有限制的。这通常被称为**速率限制**。对于单个命令，你可能需要向 HTTP 端点发送多个请求，因此你可能更喜欢限制发送这些请求的频率。大多数公共 API 都会通知用户他们的速率限制，但有时你会意外地达到 API 的速率限制。我们将讨论如何限制你的请求以保持在限制内。

有一个有用的库`x/time/rate`，可以用来定义限制，即某事应该执行的频率，以及限制器，它控制过程在限制内执行。让我们使用一些示例代码，假设我们想要每五秒执行一次。

这个特定示例的代码位于`examples/limiting.go`文件中。再次强调，这只是一个示例，使用`runner`的方式有很多种。我们将只介绍基本用法。我们首先定义一个包含函数`Run`和`limiter`字段的`struct`，后者控制其运行频率。`Limit()`函数将使用`runner`结构体在速率限制内调用函数：

```go
type runner struct {
    Run func() bool
    limiter *rate.Limiter
}
func Limit() {
    thing := runner{}
    start := time.Now()
```

在将`thing`定义为`runner`实例之后，我们获取开始时间，然后定义`thing`的功能。如果调用在规定时间内允许，因为不超过限制，我们打印当前时间戳并返回一个`false`变量。当至少过去 30 秒时，我们退出函数：

```go
    thing.Run = func() bool {
        if thing.limiter.Allow() {
            fmt.Println(time.Now()) // or call request
            return false
        }
        if time.Since(start) > 30*time.Second {
            return true
        }
        return false
    }
```

我们为`thing`定义了限制器。我们使用了一个自定义变量，我们将在稍后详细讨论。简单来说，`NewLimiter`方法接受两个变量。第一个参数是限制，每五秒一个事件，第二个参数允许最多一个令牌的突发：

```go
    thing.limiter = rate.NewLimiter(forEvery(1, 5*time.
    Second),     1)
```

对于那些不熟悉限制和突发之间区别的人来说，突发定义了 API 可以处理的并发请求数量。速率限制是每单位时间内允许的请求数量。

接下来，在`for`循环内部，我们调用`Run`函数，并且只有在它返回`true`时才退出循环，这通常是在 30 秒后：

```go
    for {
        if thing.Run() {
            break
        }
    }
}
```

如前所述，返回速率限制的`forEvery`函数被传递到`NewLimiter`方法中。它简单地调用`rate.Every`方法，该方法接受事件之间的最小时间间隔并将其转换为限制：

```go
func forEvery(eventCount int, duration time.Duration) rate.Limit {
    return rate.Every(duration / time.Duration(eventCount))
}
```

我们运行这段代码，时间戳每五秒输出一次。注意，它们每五秒输出一次：

```go
2022-09-11 18:45:44.356917 -0700 PDT m=+0.000891459
2022-09-11 18:45:49.356877 -0700 PDT m=+5.000891042
2022-09-11 18:45:54.356837 -0700 PDT m=+10.000891084
2022-09-11 18:45:59.356797 -0700 PDT m=+15.000891084
2022-09-11 18:46:04.356757 -0700 PDT m=+20.000891167
2022-09-11 18:46:09.356718 -0700 PDT m=+25.000891167
```

处理限制请求还有其他方法，例如在循环内部调用的代码之后使用`time.Sleep(d Duration)`方法。我建议使用`rate`包，因为它不仅适用于限制执行，还适用于处理突发情况。它具有更多功能，可以在发送请求到外部 API 时用于更复杂的情况。

你现在已经学会了如何向外部 API 发送请求以及如何处理响应，当收到成功的响应时，如何转换和分页结果。此外，由于速率限制对于 API 来说是常见的，我们已经讨论了如何实现这一点。由于本节只处理了成功的情况，让我们在下一节中考虑如何处理失败的情况。

# 处理预期的 – 超时和错误

当构建一个调用外部命令或向外部 API 发送 HTTP 请求的 CLI 时，使用用户传入的数据，预期意外情况是一个好主意。在理想的世界里，您可以防止不良数据。我相信您熟悉短语*垃圾输入，垃圾输出*。您可以创建测试，以确保您的代码覆盖了尽可能多的坏情况。然而，超时和错误是会发生的。这是软件的本质，当您在开发和生产中遇到它们时，您可以修改您的代码来处理新情况。

## 外部命令进程的超时

让我们先讨论在调用外部命令时如何处理超时。超时代码位于`examples/timeout.go`文件中。以下是一个完整的方法，它调用了`timeout`命令。如果您查看位于`cmd/timeout/timeout.go`中的`timeout`命令代码，您会看到它包含一个基本的无限循环。此命令将超时，但我们需要使用以下代码来处理超时：

```go
func Timeout() {
    errChan := make(chan error, 1)
    cmd := exec.Command(filepath.Join(os.Getenv("GOPATH"), 
           "bin", "timeout"))
    if err := cmd.Start(); err != nil {
        panic(err)
    }
    go func() {
        errChan <- cmd.Wait()
    }()
    select {
        case <-time.After(time.Second * 10):
            fmt.Println("timeout command timed out")
            return
        case err := <-errChan:
            if err != nil {
                fmt.Println("timeout error:", err)
            }
    }
}
```

我们首先定义一个错误通道，`errChan`，它将接收来自`cmd.Wait()`方法的任何错误。然后定义命令`cmd`，接下来调用`cmd`的`Start`方法来启动外部进程。在 Go 函数中，我们使用`cmd.Wait()`方法等待命令返回。`errChan`仅在命令退出并且标准输入和标准错误复制完成后才会接收错误值。在下面的`select`块中，我们等待从两个不同的通道接收。第一种情况等待 10 秒后返回的时间。第二种情况等待命令完成并接收错误值。此代码使我们能够优雅地处理任何超时问题。

## 外部命令进程的错误或恐慌

首先，让我们定义错误和恐慌之间的区别。错误发生在应用程序可以被恢复但处于异常状态时。如果发生恐慌，则表示发生了意外情况。例如，我们尝试访问`nil`指针上的字段或尝试访问超出数组索引范围的索引。我们可以从处理错误开始。

`os/exec`包中存在一些错误：

+   `exec.ErrDot`：当命令的文件路径在当前目录`"."`中无法解析时发生的错误，因此命名为`ErrDot`

+   `exec.ErrNotFound`：当可执行文件在定义的文件路径中无法解析时发生的错误

您可以检查类型以独特地处理每个错误。

### 处理命令路径找不到时的错误

以下代码位于`examples/error.go`文件中的`HandlingDoesNotExistErrors`函数中：

```go
cmd := exec.Command("doesnotexist", "arg1")
if errors.Is(cmd.Err, exec.ErrDot) {
    fmt.Println("path lookup resolved to a local directory")
}
if err := cmd.Run(); err != nil {
    if errors.Is(err, exec.ErrNotFound) {
        fmt.Println("executable failed to resolve")
    }
}
```

在检查命令类型时，使用`errors.Is`方法，而不是检查`cmd.Err == exec.ErrDot`，因为错误不是直接返回的。`errors.Is`方法检查错误链中是否存在特定错误类型的任何发生。

### 处理其他错误

此外，在`examples/error.go`文件中，还处理了命令进程本身抛出的错误。第二种方法`HandlingOtherMethods`将命令的标准错误设置为我们可以稍后使用的缓冲区。让我们看看代码：

```go
cmd := exec.Command(filepath.Join(os.Getenv("GOPATH"), "bin", "error"))
var out bytes.Buffer
var stderr bytes.Buffer
cmd.Stdout = &out
cmd.Stderr = &stderr
if err := cmd.Run(); err != nil {
    fmt.Println(fmt.Sprint(err) + ": " + stderr.String())
    return
}
fmt.Println(out.String())
```

当遇到错误时，我们不仅打印错误，`退出状态 1`，还打印任何已通过标准错误管道管道的数据，这应该使用户能够获得更多关于错误发生原因的详细信息。

为了进一步了解这段代码的工作原理，让我们看看`cmd/error/error.go`文件中存在的错误命令实现：

```go
func main() {
    if len(os.Args) != 0 { // not passing in any arguments in this example throws an error
        fmt.Fprintf(os.Stderr, "missing arguments\n")
        os.Exit(1)
    }
    fmt.Println("executing command with no errors")
}
```

由于我们没有将任何参数传递给命令函数，在检查`os.Args`的长度后，我们将退出原因打印到标准错误管道。这是一种非常简单但有效处理错误的方法。在调用外部进程时，我们只是返回错误，但正如我们可能都经历过的一样，错误信息可能有点晦涩。在后面的章节中，我们将讨论如何将这些错误信息重写为更易读的格式，并提供一些示例。

在*第四章* *构建 CLIs 的流行框架*中，我们讨论了在 Cobra 命令结构中`RunE`函数的使用，这允许我们在命令运行时返回一个错误值。如果你在`RunE`方法中调用外部进程，那么你可以捕获并返回错误给用户，当然，在将其重写为更易读的格式之后！

对于恐慌（panic），处理方式与错误不同，但提供一个从恐慌中优雅恢复的方法是良好的编程实践。你可以在`examples/panic.go`文件中的`Panic`方法中看到这段代码的初始化。这个方法调用位于`cmd/panic/panic.go`中的`panic`命令。这个命令简单地引发恐慌然后恢复。它将恐慌信息返回到标准错误管道，打印堆栈，并以非零退出代码退出：

```go
defer func() {
    if panicMessage := recover(); panicMessage != nil {
        fmt.Fprintf(os.Stderr, "(panic) : %v\n", panicMessage)
        debug.PrintStack()
        os.Exit(1)
    }
}()
panic("help!")
```

在运行此命令的一侧，我们像处理任何其他错误一样处理它，通过捕获错误并打印到标准错误管道中的数据。

## HTTP 请求中的超时和其他错误

类似地，你发送请求到外部 API 服务器时也可能遇到错误。为了清楚起见，超时也被视为错误。此示例的代码位于`examples/http.go`中，其中包含两个函数：

+   `HTTPTimeout()`

+   `HTTPError()`

在深入研究之前的方法之前，让我们谈谈为了使这些方法正确执行，需要运行的代码。

`cmd/api/`文件夹包含定义处理程序和本地启动 HTTP 服务器的代码。`mux.HandleFunc`方法定义请求模式并将其与`handler`函数匹配。服务器通过其地址定义，运行在 localhost，端口`8080`，以及`Handler`，`mux`。最后，在定义的服务器上调用`server.ListenAndServe()`方法：

```go
func main() {
    mux := http.NewServeMux()
    server := &http.Server{
        Addr: ":8080",
        Handler: mux,
    }
    mux.HandleFunc("/timeout", timeoutHandler)
    mux.HandleFunc("/error", errorHandler)
    err := server.ListenAndServe()
    if err != nil {
        fmt.Println("error starting api: ", err)
        os.Exit(1)
    }
}
```

超时处理程序被简单地定义。它等待两秒钟后通过使用`time.After(time.Second*2)`通道发送响应：

```go
func timeoutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("got /timeout request")
    <-time.After(time.Second * 2)
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("this took a long time"))
}
```

错误处理程序返回状态码`http.StatusInternalServerError`：

```go
func errorHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("got /error request")
    w.WriteHeader(http.StatusInternalServerError)
    w.Write([]byte("internal service error"))
}
```

在另一个终端中，在存储库的根目录下运行`make install`命令以启动 API 服务器。现在，让我们看看调用每个端点的代码，并展示我们如何处理它。让我们首先讨论第一种错误类型——超时：

+   `HTTPTimeout`：在`examples/http.go`文件中存在`HTTPTimeout`方法。让我们一起走过这段代码：

    +   首先，我们使用`http.Client`结构体**定义客户端**，指定超时为一秒。记住，由于 API 上的超时处理程序在两秒后返回响应，因此请求肯定会超时：

```go
client := http.Client{
    Timeout: 1 * time.Second,
}
```

+   接下来，我们**定义请求**：一个对`/timeout`端点的`GET`方法。我们传递一个空的主体：

```go
body := &bytes.Buffer{}
req, err := http.NewRequest(http.MethodGet, "http://localhost:8080/timeout", body)
if err != nil {
    panic(err)
}
```

+   客户端`Do`方法使用请求变量作为参数被调用。我们等待服务器在一秒内响应，如果没有，则返回错误。客户端`Do`方法返回的错误将是`*url.Error`类型。您可以访问此错误类型的不同字段，但在以下代码中，我们检查错误的`Timeout`方法是否返回`true`。在这个语句中，我们可以按自己的意愿行事。我们可以暂时返回错误，我们可以退避并重试，或者我们可以退出。这取决于您的具体用例：

```go
resp, err := client.Do(req)
if err != nil {
    urlErr := err.(*url.Error)
    if urlErr.Timeout() {
        fmt.Println("timeout: ", err)
        return
    }
}
defer resp.Body.Close()
```

当此方法执行时，输出如下：

```go
timeout:  Get "http://localhost:8080/timeout": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

超时只是其中一种错误，但您可能会遇到许多其他错误。由于客户端`Do`方法在`net/url`包中返回特定的错误类型，让我们来讨论一下。在`net/url`包中存在`url.Error`类型定义：

```go
type Error struct {
    Op  string // Operation
    URL string // URL
    Err error // Error
}
```

错误包含`Timeout()`方法，当请求超时时返回`true`，并且需要注意的是，当响应状态不是`200 OK`时，错误不会被设置。然而，状态码指示错误响应。错误响应可以分为两个不同的类别：

+   (`400`到`499`)表示客户端发生错误。一些例子包括`Bad Request (400)`、`Unauthorized (401)`和`Not Found (404)`。

+   (`500`到`599`)表示服务器端发生错误。一些常见的例子包括`Internal Server Error (500)`、`Bad Gateway (502)`和`Service Unavailable (503)`。

`HTTPErrors`：如何在`examples/http.go`文件中的`HTTPErrors`方法中处理这种情况的示例代码存在。同样，在执行此代码之前确保 API 服务器正在运行非常重要：

+   方法中的代码首先通过调用对`/error`端点的`GET`请求开始：

```go
resp, err := http.Get("http://localhost:8080/error")
```

+   如果错误不是`nil`，那么我们将它转换为`url.Error`类型以访问其中的字段和方法。例如，我们检查`urlError`是否是超时或临时网络错误。如果不是这两种情况，那么我们可以输出我们所知道的所有关于错误的信息到标准输出。这些附加信息可以帮助我们确定下一步要采取的措施：

```go
if err != nil {
    urlErr := err.(*url.Error)
    if urlErr.Timeout() {
         // a timeout is a type of error
        fmt.Println("timeout: ", err)
        return
    }
    if urlErr.Temporary() {
        // a temporary network error, retry later
        fmt.Println("temporary: ", err)
        return
    }
    fmt.Printf("operation: %s, url: %s, error: %s\n", urlErr.
        Op,        urlErr.URL, urlErr.Error())
    return
}
```

+   由于状态码错误响应不被视为 Go 语言错误，响应体可能包含一些有用的信息。如果它不是`nil`，那么我们可以读取状态码：

```go
if resp != nil {
    defer resp.Body.Close()
```

+   我们最初检查`StatusCode`是否不等于`http.StatusOK`。从那里，我们可以检查特定的错误消息并采取适当的行动。在这个例子中，我们只检查了三种不同类型的错误响应，但你可以检查对你所做的事情有意义的任何类型：

```go
if resp.StatusCode != http.StatusOK {
        // action for when status code is not okay
        switch resp.StatusCode {
        case http.StatusBadRequest:
            fmt.Printf("bad request: %v\n", resp.Status)
        case http.StatusInternalServerError:
            fmt.Printf("internal service error: %v\n", resp.
                Status)
        default:
            fmt.Printf("unexpected status code: %v\n", resp.
                StatusCode)
        }
    }
```

+   最后，客户端或服务器错误状态并不一定意味着响应体是`nil`。如果其中包含任何有用的信息，我们可以输出响应体：

```go
    data, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("err:", err)
    }
    fmt.Println("response body:", string(data))
}
```

这就结束了处理 HTTP 超时和其他错误的章节。尽管示例很简单，但它们为你提供了处理超时、临时网络和其他错误所必需的信息和指导。

# 摘要

在本章中，你深入了解了`os/exec`包。这包括学习创建命令的不同方式：使用`command`结构体或`Command`方法。我们不仅创建了命令，还向它们传递文件描述符以接收信息。我们学习了使用`Run`或`Start`方法运行命令的不同方式，以及从标准输出、标准错误类型和其他文件描述符检索数据的多重方式。

在本章中，我们还讨论了`net/http`和`net/url`包，当创建对外部 API 服务器的 HTTP 请求时，熟悉这些包非常重要。几个示例教会了我们如何使用`http.Client`上的方法创建请求，包括`Do`、`Get`、`Post`和`PostForm`。

学习如何构建健壮的代码非常重要，而优雅地处理错误是这个过程的一部分。我们需要知道如何首先捕获错误，因此我们讨论了在运行外部进程或向外部 API 服务器发送请求时可能发生的某些常见错误的检测方法。捕获和处理其他错误使我们确信我们的代码在它们发生时能够采取适当的行动。最后，我们现在知道如何在响应不正常时检查不同的状态码。

在学习了本章的所有信息后，我们现在应该更有信心构建一个与外部命令交互或向外部 API 发送请求的 CLI。在下一章，我们将学习如何编写可以在多个不同的架构和操作系统上运行的代码。

# 问题

1.  在`time`包中，我们使用什么方法通过通道接收特定持续时间后的时间？

1.  `http.Client`的`Do`方法返回的错误类型是什么？

1.  当 HTTP 请求收到一个状态码不是`StatusOK`的响应时，请求返回的错误是否被填充？

# 答案

1.  `time.After(d Duration) <-chan Time`

1.  `*url.Error`

1.  否

# 进一步阅读

+   请访问`net/http`的在线文档[在此](https://pkg.go.dev/net/http)，以及`net/url`的文档[在此](https://pkg.go.dev/net/url)
