

# 告诉 UNIX 系统做什么

本章是关于 Go 语言系统编程的。系统编程涉及与文件和目录、进程控制、信号处理、网络编程、系统文件、配置文件以及文件**输入和输出**（**I/O**）的工作。如果你还记得*第一章*，*Go 语言快速入门*，使用 Linux 系统编写系统工具的原因是，通常，Go 软件是在 Docker 环境中执行的——Docker 镜像使用 Linux 操作系统，这意味着你可能需要考虑 Linux 操作系统来开发你的工具。然而，由于 Go 代码的可移植性，大多数系统工具在 Windows 机器上无需任何更改或仅做少量修改即可工作。要记住的关键思想是 Go 使得系统编程更加可移植。此外，在本章中，我们将借助`cobra`包来改进统计应用程序。

如前所述，从 Go 1.16 版本开始，`GO111MODULE`环境变量默认设置为`on`——这影响了不属于 Go 标准库的 Go 包的使用。在实践中，这意味着你必须将你的代码放在`~/go/src`下。

本章涵盖：

+   `stdin`, `stdout`, 和 `stderr`

+   UNIX 进程

+   文件 I/O

+   读取纯文本文件

+   向文件写入

+   与 JSON 一起工作

+   `viper`包

+   `cobra`包

+   重要的 Go 特性

+   更新统计应用程序

# stdin, stdout, 和 stderr

每个 UNIX 操作系统都会为它的进程始终打开三个文件。记住，UNIX 将一切视为文件，即使是打印机或鼠标。UNIX 使用文件描述符，即正整数值，作为访问打开文件的内部表示，这比使用长路径要美观得多。因此，默认情况下，所有 UNIX 系统都支持三个特殊的标准文件名：`/dev/stdin`、`/dev/stdout`和`/dev/stderr`，它们也可以分别使用文件描述符`0`、`1`和`2`来访问。这三个文件描述符也分别被称为标准输入、标准输出和标准错误。此外，文件描述符`0`在 macOS 机器上可以访问为`/dev/fd/0`，在 Debian Linux 机器上可以访问为`/dev/fd/0`和`/dev/pts/0`。

Go 使用`os.Stdin`来访问标准输入，`os.Stdout`来访问标准输出，以及`os.Stderr`来访问标准错误。虽然你仍然可以使用`/dev/stdin`、`/dev/stdout`和`/dev/stderr`或相关的文件描述符值来访问相同的设备，但坚持使用`os.Stdin`、`os.Stdout`和`os.Stderr`会更佳、更安全、更可移植。

# UNIX 进程

由于 Go 服务器、工具和 Docker 镜像主要在 Linux 上执行，了解 Linux 进程和线程是有好处的。

严格来说，进程是一个包含指令、用户数据和系统数据部分以及其他在运行时获得的资源类型的执行环境。另一方面，程序是一个包含指令和数据二进制文件，这些指令和数据用于初始化进程的指令和数据部分。每个运行的 UNIX 进程都有一个唯一的无符号整数标识符，称为进程 ID。

存在三种进程类别：用户进程、守护进程和内核进程。用户进程在用户空间运行，通常没有特殊访问权限。守护进程是可以在用户空间找到的程序，可以在后台运行而不需要终端。内核进程仅在内核空间执行，可以完全访问所有内核数据结构。

使用 C 语言创建新进程的方式涉及调用 `fork(2)` 系统调用。`fork(2)` 的返回值允许程序员区分父进程和子进程。虽然你可以使用 `exec` 包在 Go 中创建新的进程，但 Go 不允许你控制线程——Go 提供了 goroutines，用户可以在由 Go 运行时创建和处理的线程之上创建 goroutines，这部分由操作系统控制。

现在，我们需要学习如何在 Go 中读取和写入文件。

# 文件 I/O

本节讨论 Go 中的文件 I/O，包括使用 `io.Reader` 和 `io.Writer` 接口、缓冲和非缓冲 I/O，以及 `bufio` 包。

`io/ioutil` 包（[`pkg.go.dev/io/ioutil`](https://pkg.go.dev/io/ioutil)）自 Go 版本 1.16 起已被弃用。使用 `io/ioutil` 功能的现有 Go 代码将继续工作，但最好停止使用该包。

## io.Reader 和 io.Writer 接口

本小节介绍了流行的 `io.Reader` 和 `io.Writer` 接口的定义，因为这两个接口是 Go 中文件 I/O 的基础——前者允许你从文件中读取，而后者允许你向文件写入。`io.Reader` 接口的定义如下：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
} 
```

当我们要使我们的数据类型满足 `io.Reader` 接口时应该重新审视的此定义，告诉我们以下信息：

+   `Reader` 接口需要实现一个方法。

+   `Read()` 方法接收一个字节切片作为输入，该切片将被填充至其长度。

+   `Read()` 方法返回读取的字节数以及一个 `error` 变量。

`io.Writer` 接口的定义如下：

```go
type Writer interface {
    Write(p []byte) (n int, err error)
} 
```

之前的定义，当我们要使我们的数据类型满足 `io.Writer` 接口并将数据写入文件时应该重新审视，揭示了以下信息：

+   该接口需要实现一个方法。

+   `Write()` 方法接收一个字节切片，其中包含要写入的数据。

+   `Write()` 方法返回写入的字节数和一个 `error` 变量。

## 使用和误用 io.Reader 和 io.Writer

下面的代码展示了如何使用 `io.Reader` 和 `io.Writer` 来处理 **自定义数据类型**，在这个例子中，是两个名为 `S1` 和 `S2` 的 Go 结构体。

对于 `S1` 结构体，展示的代码实现了两个接口，分别用于从终端读取用户数据以及将数据打印到终端。尽管这有些冗余，因为我们已经有了 `fmt.Scanln()` 和 `fmt.Printf()`，但这是一个很好的练习，展示了这两个接口的多样性和灵活性。在不同的情境下，你可以使用 `io.Writer` 来写入日志服务，保留第二个数据备份，或者满足你需求的任何其他东西。然而，这也是接口允许你做疯狂或，如果你愿意，不寻常的事情的一个例子。开发者需要使用适当的 Go 概念和特性来创建所需的功能！

`Read()` 方法使用 `fmt.Scanln()` 从终端获取用户输入，而 `Write()` 方法使用 `fmt.Printf()` 将其缓冲区参数的内容打印出结构体 `F1` 字段值的次数！

对于 `S2` 结构体，展示的代码仅实现了 `io.Reader` 接口，以传统方式。`Read()` 方法读取 `S2` 结构体的文本字段，它是一个字节切片。当没有更多内容可读取时，`Read()` 方法返回预期的 `io.EOF` 错误，实际上这并不是一个错误，而是一个预期的状态。除了 `Read()` 方法外，还存在两个辅助方法，名为 `eof()`，它声明没有更多内容可读取，以及 `readByte()`，它逐字节读取 `S2` 结构体的文本字段。`Read()` 方法完成后，用作缓冲区的 `S2` 结构体的文本字段将被清空。

使用这个实现，`S2` 的 `io.Reader` 可以以传统方式读取，在这种情况下，是使用 `bufio.NewReader()` 和多次 `Read()` 调用——`Read()` 调用的次数取决于使用的缓冲区大小，在这个例子中，是一个有两个数据位置的字节切片。

输入以下代码并将其保存为 `ioInterface.go`：

```go
package main
import (
    "bufio"
"fmt"
"io"
) 
```

之前的部分展示了我们正在使用 `io` 和 `bufio` 包来处理文件。

```go
type S1 struct {
    F1 int
    F2 string
}
type S2 struct {
    F1   S1
    text []byte
} 
```

这是我们将要工作的两个结构体。

```go
// Using pointer to S1 for changes to be persistent
func (s *S1) Read(p []byte) (n int, err error) {
    fmt.Print("Give me your name: ")
    fmt.Scanln(&p)
    s.F2 = string(p)
    return len(p), nil
} 
```

在前面的代码中，我们实现了 `S1` 的 `io.Reader` 接口。

```go
func (s *S1) Write(p []byte) (n int, err error) {
    if s.F1 < 0 {
        return -1, nil
    }
    for i := 0; i < s.F1; i++ {
        fmt.Printf("%s ", p)
    }
    fmt.Println()
    return s.F1, nil
} 
```

之前的方法实现了 `S1` 的 `io.Writer` 接口。

```go
func (s S2) eof() bool {
    return len(s.text) == 0
}
func (s *S2) readByte() byte {
    // this function assumes that eof() check was done before
    temp := s.text[0]
    s.text = s.text[1:]
    return temp
} 
```

之前的功能是标准库中 `bytes.Buffer.ReadByte` 的实现。

```go
func (s *S2) Read(p []byte) (n int, err error) {
    if s.eof() {
        err = io.EOF
        return 0, err
    }
    l := len(p)
    if l > 0 {
        for n < l {
            p[n] = s.readByte()
            n++
            if s.eof() {
                s.text = s.text[0:0]
                break
            }
        }
    }
    return n, nil
} 
```

之前的代码从给定的缓冲区读取，直到它为空。当所有数据都被读取后，相关的结构体字段将被清空。之前的方法实现了 `S2` 的 `io.Reader`。然而，`Read()` 操作是由 `eof()` 和 `readByte()` 支持的，这两个也是用户自定义的。

记住，Go 允许你给函数的返回值命名；在这种情况下，没有额外参数的`return`语句会自动返回函数签名中按顺序出现的每个命名返回变量的当前值。`Read()`方法可以使用该功能，但通常，裸返回被认为是不良实践。

```go
func main() {
    s1var := S1{4, "Hello"}
    fmt.Println(s1var) 
```

我们初始化一个名为`s1var`的`S1`变量。

```go
 buf := make([]byte, 2)
    _, err := s1var.Read(buf) 
```

上行代码使用两个字节的缓冲区读取`s1var`变量。但是，该代码块并没有达到预期的效果，因为`Read()`方法的实现是从终端获取值——在这里我们误用了 Go 接口！

```go
 if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("Read:", s1var.F2)
    _, _ = s1var.Write([]byte("Hello There!")) 
```

在上一行中，我们调用`s1var`的`Write()`方法来写入字节数组的内容。

```go
 s2var := S2{F1: s1var, text: []byte("Hello world!!")} 
```

在之前的代码中，我们初始化了一个名为`s2var`的`S2`变量。

```go
 // Read s2var.text
    r := bufio.NewReader(&s2var) 
```

我们现在为`s2var`创建一个读取器。

```go
 for {
        n, err := r.Read(buf)
        if err == io.EOF {
            break 
```

我们会一直从`s2var`读取，直到出现`io.EOF`条件。

```go
 } else if err != nil {
            fmt.Println("*", err)
            break
        }
        fmt.Println("**", n, string(buf[:n]))
    }
} 
```

运行`ioInterface.go`会产生以下输出：

```go
$ go run ioInterface.go
{4 Hello} 
```

输出的第一行显示了`s1var`变量的内容。

```go
Give me your name: Mike
Calling the Read() method of the s1var variable.
Read: Mike
Hello There! Hello There! Hello There! Hello There!
The previous line is the output of s1var.Write([]byte("Hello There!")).
** 2 He
** 2 ll
** 2 o 
** 2 wo
** 2 rl
** 2 d!
** 1 ! 
```

输出的最后部分展示了使用大小为二的缓冲区的读取过程。下一节将讨论带缓冲和不带缓冲的操作。

## 带缓冲和不带缓冲的文件 I/O

*带缓冲的文件 I/O* 发生在有缓冲区临时存储数据，在读取数据或写入数据之前。因此，你不会逐字节读取文件，而是可以一次性读取多个字节。你将数据放入缓冲区，并等待有人以期望的方式读取它。

*不带缓冲的文件 I/O* 发生在没有缓冲区在读取或写入之前临时存储数据的情况——这可能会影响程序的性能。

你可能接下来会问，如何决定何时使用带缓冲的文件 I/O 和何时使用不带缓冲的文件 I/O。当处理关键数据时，不带缓冲的文件 I/O 通常是一个更好的选择，因为带缓冲的读取可能会导致数据过时，而带缓冲的写入可能在计算机电源中断时导致数据丢失。然而，大多数情况下，这个问题没有明确的答案。这意味着你可以使用任何使你的任务更容易实现的方法。然而，请记住，带缓冲的读取器也可以通过减少从文件或套接字读取所需的系统调用来提高性能，因此程序员的选择可能会对性能产生实际影响。

此外，还有`bufio`包。正如其名所示，`bufio`是关于带缓冲的 I/O。内部，`bufio`包实现了`io.Reader`和`io.Writer`接口，它将它们包装起来以创建`bufio.Reader`和`bufio.Writer`类型。`bufio`包在处理纯文本文件时非常流行，你将在下一节中看到它的实际应用。

# 读取文本文件

在本节中，你将学习如何读取纯文本文件，以及如何使用提供获取随机数方式的`/dev/random` UNIX 设备。

## 逐行读取文本文件

逐行读取文件的函数位于`byLine.go`中，并命名为`lineByLine()`。逐行读取文本文件的技巧也用于逐字读取纯文本文件，以及逐字符读取纯文本文件，因为通常你按行处理纯文本文件。所提供的实用程序打印它所读取的每一行，这使得它成为`cat(1)`实用程序的简化版本。

首先，你使用`bufio.NewReader()`调用为所需的文件创建一个新的读取器。然后，你使用该读取器与`bufio.ReadString()`一起按行读取输入文件。技巧是通过`bufio.ReadString()`的参数实现的，该参数告诉`bufio.ReadString()`在找到该字符之前继续读取。当该参数是换行符（`\n`）时，不断调用`bufio.ReadString()`会导致按行读取输入文件。

`lineByLine()`的实现如下：

```go
func lineByLine(file string) error {
    f, err := os.Open(file)
    if err != nil {
        return err
    }
    defer f.Close()
    r := bufio.NewReader(f) 
```

在确保你可以使用`os.Open()`打开给定的文件进行读取后，你使用`bufio.NewReader()`创建一个新的读取器。

```go
 for {
        line, err := r.ReadString('\n') 
```

`bufio.ReadString()`返回两个值：读取的字符串和一个错误变量。

```go
 if err == io.EOF {
            if len(line) != 0 {
                    fmt.Println(line)
            }
            break
        }
        if err != nil {
            fmt.Printf("error reading file %s", err)
            return err
        }
        fmt.Print(line) 
```

使用`fmt.Print()`而不是`fmt.Println()`来打印输入行表明换行符包含在每个输入行中。

```go
 }
    return nil
} 
```

运行`byLine.go`会生成以下类型的输出：

```go
$ go run byLine.go ~/csv.data
Dimitris,Tsoukalos,2101112223,1600665563
Mihalis,Tsoukalos,2109416471,1600665563
Jane,Doe,0800123456,1608559903 
```

之前的输出显示了使用`byLine.go`逐行显示的`~/csv.data`（使用你自己的纯文本文件）的内容。下一小节将展示如何逐字读取纯文本文件。

## 逐字读取文本文件

逐字读取纯文本文件是你在文件上想要执行的最有用的函数之一，因为你通常希望按单词处理文件——这在本小节中通过`byWord.go`中的代码进行说明。所需的功能在`wordByWord()`函数中实现。`wordByWord()`函数使用**正则表达式**来分隔输入文件每一行中找到的单词。`regexp.MustCompile("[^\\s]+")`语句中定义的正则表达式表示我们使用空白字符来分隔一个单词与另一个单词。

`wordByWord()`函数的实现如下：

```go
func wordByWord(file string) error {
    f, err := os.Open(file)
    if err != nil {
        return err
    }
    defer f.Close()
    r := bufio.NewReader(f)
    re := regexp.MustCompile("[^\\s]+")
    for { 
```

这是我们定义用于将行拆分为单词的正则表达式的地方。

```go
 line, err := r.ReadString('\n')
        if err == io.EOF {
            if len(line) != 0 {
                words := re.FindAllString(line, -1)
                for i := 0; i < len(words); i++ {
                    fmt.Println(words[i])
                }
            }
            break 
```

这是程序的难点部分。如果我们遇到一个不以换行符结束的行的文件末尾，我们也必须处理它，但之后，我们必须退出`for`循环，因为从该文件中再也没有什么可读的内容了。之前的代码处理了这一点。

```go
 } else if err != nil {
            fmt.Printf("error reading file %s", err)
            return err
        } 
```

在这个程序部分，我们处理可能出现的潜在错误条件，这些条件可能会阻止我们读取文件。

```go
 words := re.FindAllString(line, -1) 
```

这是我们将正则表达式应用于`line`变量以将其拆分为字段的地方，当读取的行以换行符结束。

```go
 for i := 0; i < len(words); i++ {
            fmt.Println(words[i])
        } 
```

这个`for`循环只是打印`words`切片的字段。如果你想知道输入行中找到的单词数量，你只需找到`len(words)`调用的值。

```go
 }
    return nil
} 
```

运行`byWord.go`会产生以下类型的输出：

```go
$ go run byWord.go ~/csv.data
Dimitris,Tsoukalos,2101112223,1600665563
Mihalis,Tsoukalos,2109416471,1600665563
Jane,Doe,0800123456,1608559903 
```

由于`~/csv.data`不包含任何空白字符，每一行都被视为一个单独的单词！

## 逐字符读取文本文件

在本小节中，你将学习如何逐字符读取文本文件，除非你想开发一个文本编辑器，否则这是一个罕见的需求。你读取每一行，并使用带有范围的`for`循环将其分割，这会返回两个值。你丢弃第一个值，它是`line`变量中当前字符的位置，而使用第二个值。然而，这个值是一个 rune，这意味着你必须使用`string()`将其转换为字符。

`charByChar()`的实现如下：

```go
func charByChar(file string) error {
  f, err := os.Open(file)
  if err != nil {
    return err
  }
  defer f.Close()
  r := bufio.NewReader(f)
  for {
    line, err := r.ReadString('\n')
    if err == io.EOF {
      if **len****(line) !=** **0** {
        for _, x := range line {
          fmt.Println(string(x))
        } 
```

再次提醒，我们应该特别注意那些不以换行符结尾的行。捕获此类行的条件是，我们已经到达了正在读取的文件的末尾，并且我们仍然有文本需要处理。

```go
 }
      break
    } else if err != nil {
      fmt.Printf("error reading file %s", err)
      return err
    }
    for _, x := range line {
      fmt.Println(string(x))
    }
  }
  return nil
} 
```

注意，由于`fmt.Println(string(x))`语句，每个字符都会打印在一行上，这意味着程序的输出将会很大。如果你想要更紧凑的输出，你应该使用`fmt.Print()`函数。

运行`byCharacter.go`并使用`head(1)`进行过滤，不带任何参数，会产生以下类型的输出：

```go
$ go run byCharacter.go ~/csv.data | head
D
...
,
T 
```

不带任何参数使用`head(1)`实用程序将输出限制为仅 10 行。输入`man head`以了解更多关于`head(1)`实用程序的信息。

下一个部分是关于从`/dev/random`读取的内容，它是一个 UNIX 系统文件。

## 从`/dev/random`读取

在本小节中，你将学习如何从`/dev/random`系统设备读取。`/dev/random`系统设备的目的生成随机数据，你可能用它来测试你的程序，或者在这种情况下，作为随机数生成器的种子。从`/dev/random`获取数据可能有点棘手，这也是我们在这里特别讨论的主要原因。

`devRandom.go`的代码如下：

```go
package main
import (
    "encoding/binary"
"fmt"
"os"
) 
```

你需要`encoding/binary`，因为你从`/dev/random`读取二进制数据，并将其转换为整数值。

```go
func main() {
    f, err := os.Open("/dev/random")
    defer f.Close()
    if err != nil {
        fmt.Println(err)
        return
    }
    var seed int64
    binary.Read(f, binary.LittleEndian, &seed)
    fmt.Println("Seed:", seed)
} 
```

有两种表示形式称为*小端*和*大端*，它们与内部表示的字节序相关。在我们的情况下，我们使用小端。端序与不同计算系统如何排序多个信息字节的方式有关。

一个关于字节序的实际情况的例子是不同语言以不同方式读取文本：欧洲语言通常是从左到右读取，而阿拉伯文本是从右到左读取的。

在大端表示法中，字节是从左到右读取的，而小端表示法是从右到左读取字节。对于需要 4 个字节来存储的 `0x01234567` 值，大端表示法是 `01 | 23 | 45 | 67`，而小端表示法是 `67 | 45 | 23 | 01`。

运行 `devRandom.go` 会产生以下类型的输出：

```go
$ go run devRandom.go
Seed: 422907465220227415 
```

这意味着 `/dev/random` 设备是一个获取随机数据的好地方，包括随机数生成器的种子值。

## 从文件中读取特定数量的数据

本小节教你如何从文件中读取特定数量的数据。当你想查看文件的一小部分时，这个实用程序会很有用。作为命令行参数给出的数值指定了将要用于读取的缓冲区的大小。`readSize.go` 中的最重要的代码是 `readSize()` 函数的实现：

```go
func readSize(f *os.File, size int) []byte {
    buffer := make([]byte, size)
    n, err := f.Read(buffer) 
```

所有的魔法都在 `buffer` 变量的定义中发生，因为这是我们定义它可以保存的最大数据量的地方。因此，每次我们调用 `readSize()`，该函数将从 `f` 中最多读取 `size` 个字符。

```go
 // io.EOF is a special case and is treated as such
if err == io.EOF {
        return nil
    }
    if err != nil {
        fmt.Println(err)
        return nil
    }
    return buffer[0:n]
} 
```

剩余的代码是关于错误条件；`io.EOF` 是一个特殊且预期的条件，应该单独处理，并将读取的字符作为字节切片返回给调用函数。

运行 `readSize.go` 会产生以下类型的输出。

```go
$ go run readSize.go 12 readSize.go
package main 
```

在这种情况下，我们由于 `12` 参数而从 `readSize.go` 本身读取了 12 个字符。现在我们已经知道如何读取文件，是时候学习如何向文件写入数据了。

# 向文件写入

到目前为止，我们已经看到了读取文件的方法。本小节展示了如何以四种不同的方式将数据写入文件，以及如何向现有文件追加数据。`writeFile.go` 的代码如下：

```go
package main
import (
    "bufio"
"fmt"
"io"
"os"
)
func main() {
    buffer := []byte("Data to write\n")
    f1, err := os.Create("/tmp/f1.txt") 
```

`os.Create()` 返回一个与文件路径关联的 `*os.File` 值。请注意，如果文件已经存在，`os.Create()` 将截断它。

```go
 if err != nil {
        fmt.Println("Cannot create file", err)
        return
    }
    defer f1.Close()
    fmt.Fprintf(f1, string(buffer)) 
```

`fmt.Fprintf()` 函数需要一个 `string` 变量，它可以帮助你使用你想要的格式将数据写入自己的文件。唯一的要求是拥有一个 `io.Writer` 来写入。在这种情况下，一个有效的 `*os.File` 变量，它满足 `io.Writer` 接口，就可以完成这项工作。

```go
 f2, err := os.Create("/tmp/f2.txt")
    if err != nil {
        fmt.Println("Cannot create file", err)
        return
    }
    defer f2.Close()
    n, err := f2.WriteString(string(buffer)) 
```

`WriteString()` 将字符串的内容写入一个有效的 `*os.File` 变量。

```go
 fmt.Printf("wrote %d bytes\n", n)
    f3, err := os.Create("/tmp/f3.txt") 
```

在这里，我们创建一个临时文件。

Go 还提供了 `os.CreateTemp()` 来创建临时文件。输入 `go doc os.CreateTemp` 来了解更多信息。

```go
 if err != nil {
        fmt.Println(err)
        return
    }
    w := bufio.NewWriter(f3) 
```

此函数返回一个 `bufio.Writer`，它满足 `io.Writer` 接口。

```go
 n, err = w.WriteString(string(buffer))
    fmt.Printf("wrote %d bytes\n", n)
    w.Flush()
    f := "/tmp/f4.txt"
    f4, err := os.Create(f)
    if err != nil {
        fmt.Println(err)
        return
    }
    defer f4.Close()
    for i := 0; i < 5; i++ {
        n, err = io.WriteString(f4, string(buffer))
        if err != nil {
            fmt.Println(err)
            return
        }
        fmt.Printf("wrote %d bytes\n", n)
    }
    // Append to a file
    f4, err = os.OpenFile(f, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644) 
```

`os.OpenFile()` 提供了一种更好的方式来创建或打开文件进行写入。`os.O_APPEND` 表示如果文件已经存在，你应该向其追加而不是截断它。`os.O_CREATE` 表示如果文件不存在，应该创建它。最后，`os.O_WRONLY` 表示程序应该只为写入打开文件。

```go
 if err != nil {
        fmt.Println(err)
        return
    }
    defer f4.Close()
    // Write() needs a byte slice
    n, err = f4.Write([]byte("Put some more data at the end.\n")) 
```

`Write()` 方法从字节切片获取输入，这是 Go 的写入方式。所有之前的技术都使用了字符串，这不是最佳方式，尤其是在处理二进制数据时。然而，使用字符串而不是字节切片更为实用，因为操纵 `string` 值比操纵字节切片的元素更方便，尤其是在处理 Unicode 字符时。另一方面，使用 `string` 值会增加分配，并可能导致大量的垃圾回收压力。

```go
 if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("wrote %d bytes\n", n)
} 
```

运行 `writeFile.go` 会在磁盘上生成一些关于写入字节的输出信息。有趣的是查看在 `/tmp` 文件夹中创建的文件：

```go
$ ls -l /tmp/f?.txt
-rw-r--r--@ 1 mtsouk  wheel   14 Aug  5 11:30 /tmp/f1.txt
-rw-r--r--@ 1 mtsouk  wheel   14 Aug  5 11:30 /tmp/f2.txt
-rw-r--r--@ 1 mtsouk  wheel   14 Aug  5 11:30 /tmp/f3.txt
-rw-r--r--@ 1 mtsouk  wheel  101 Aug  5 11:30 /tmp/f4.txt 
```

之前的输出显示，在 `f1.txt`、`f2.txt` 和 `f3.txt` 中写入了相同数量的信息（14 字节），这意味着所展示的写入技术是等效的。

下一个部分将展示如何在 Go 中处理 JSON 数据。

# 处理 JSON

Go 标准库包括 `encoding/json`，用于处理 JSON 数据。此外，Go 允许你通过标签在 Go 结构中添加对 JSON 字段的支持，这是结构和 JSON 子节点的主题。标签控制 JSON 记录与 Go 结构之间的编码和解码。但首先，我们应该谈谈 JSON 记录的 *序列化* 和 *反序列化*。

## 使用 Marshal() 和 Unmarshal()

使用 Go 结构处理 JSON 数据时，序列化和反序列化是重要的步骤。序列化是将 Go 结构转换为 JSON 记录的过程。你通常希望这样做，以便通过计算机网络传输 JSON 数据或将其保存到磁盘上。反序列化是将作为字节切片给出的 JSON 记录转换为 Go 结构的过程。你通常希望在通过计算机网络接收 JSON 数据或从磁盘文件加载 JSON 数据时这样做。

将 JSON 记录转换为 Go 结构，反之亦然时最常见的错误是没有将 Go 结构的必需字段导出，也就是说，它们的首字母大写。当你遇到序列化和反序列化问题时，应从那里开始调试过程。

`encodeDecode.go` 中的代码展示了使用硬编码数据简化过程的 JSON 记录的序列化和反序列化：

```go
package main
import (
    "encoding/json"
"fmt"
)
type UseAll struct {
    Name    string `json:"username"`
    Surname string `json:"surname"`
    Year    int `json:"created"`
} 
```

之前元数据告诉我们，`UseAll` 结构的 `Name` 字段在 JSON 记录中翻译为 `username`，反之亦然；`Surname` 字段翻译为 `surname`，反之亦然；`Year` 结构字段在 JSON 记录中翻译为 `created`，反之亦然。这些信息与 JSON 数据的序列化和反序列化有关。除此之外，你可以像使用常规 Go 结构一样处理和使用 `UseAll`。

```go
func main() {
    useall := UseAll{Name: "Mike", Surname: "Tsoukalos", Year: 2023}
    // Encoding JSON data: Convert Structure to JSON record with fields
    t, err := json.Marshal(&useall) 
```

`json.Marshal()` 函数需要一个指向结构变量的指针——其实际数据类型是一个空接口变量——并返回一个包含编码信息的字节切片和一个 `error` 变量。

```go
 if err != nil {
        fmt.Println(err)
    } else {
        fmt.Printf("Value %s\n", t)
    }
    // Decoding JSON data given as a string
    str := `{"username": "M.", "surname": "Ts", "created":2024}` 
```

JSON 数据通常以字符串的形式出现。

```go
 // Convert string into a byte slice
    jsonRecord := []byte(str) 
```

然而，由于`json.Unmarshal()`需要一个字节切片，你需要在将其传递给`json.Unmarshal()`之前将那个`string`转换为字节切片。

```go
 // Create a structure variable to store the result
    temp := UseAll{}
    err = json.Unmarshal(jsonRecord, &temp) 
```

`json.Unmarshal()`函数需要一个包含 JSON 记录的字节切片和一个指向将存储 JSON 记录的 Go 结构体变量的指针，返回一个`error`变量。

```go
 if err != nil {
        fmt.Println(err)
    } else {
        fmt.Printf("Data type: %T with value %v\n", temp, temp)
    }
} 
```

运行`encodeDecode.go`会产生以下输出：

```go
$ go run encodeDecode.go
Value {"username":"Mike","surname":"Tsoukalos","created":2023}
Data type: main.UseAll with value {M. Ts 2024} 
```

下一小节将更详细地说明如何在 Go 结构体中定义 JSON 标签。

## 结构体和 JSON

假设你有一个 Go 结构体，你想将其转换为 JSON 记录而不包括任何空字段——以下代码展示了如何使用`omitempty`来执行此任务：

```go
// Ignoring empty fields in JSON
type NoEmpty struct {
    Name    string `json:"username"`
    Surname string `json:"surname"`
    Year    int `json:"creationyear,omitempty"`
} 
```

现在，假设你有一些敏感数据存储在 Go 结构体的某些字段中，你不想将其包含在 JSON 记录中。你可以通过在所需的`json:`结构体标签中包含特殊值`-`来实现这一点。以下代码片段展示了如何做到这一点：

```go
// Removing private fields and ignoring empty fields
type Password struct {
    Name     string `json:"username"`
    Surname  string `json:"surname,omitempty"`
    Year     int `json:"creationyear,omitempty"`
    Pass     string `json:"-"`
} 
```

因此，在将`Password`结构体转换为 JSON 记录时使用`json.Marshal()`函数时，`Pass`字段将被忽略。

这两种技术已在`tagsJSON.go`中展示。运行`tagsJSON.go`会产生以下输出：

```go
$ go run tagsJSON.go
noEmptyVar decoded with value {username":"Mihalis","surname":""}
password decoded with value {"username":"Mihalis"} 
```

对于输出结果的第一行，我们有以下内容：将`noEmpty`值转换为名为`noEmptyVar`的`NoEmpty`结构体变量的值是`NoEmpty{Name: "Mihalis"}`。`noEmpty`结构体对于`Surname`和`Year`字段具有默认值。然而，由于它们没有被特别定义，`json.Marshal()`忽略了带有`omitempty`标签的`Year`字段，但没有忽略具有空`string`值但没有`omitempty`标签的`Surname`字段。

对于输出结果的第二行，`password`变量的值为`Password{Name: "Mihalis", Pass: "myPassword"}`。当`password`变量被转换为 JSON 记录时，`Pass`字段不包括在输出中。由于`omitempty`标签，`Password`结构体的剩余两个字段`Surname`和`Year`被省略。所以，剩下的就是`username`字段及其值。

到目前为止，我们已经看到了如何处理单个 JSON 记录。但是当我们有多个记录需要处理时会发生什么？我们是否必须逐个处理它们？下一小节将回答这些问题以及更多问题！

## 以流的形式读取和写入 JSON 数据

假设你有一个代表 JSON 记录的 Go 结构体切片，你想处理这些记录。你应该逐个处理记录吗？虽然可以这样做，但这看起来效率如何？好消息是 Go 支持以流的形式处理多个 JSON 记录而不是单个记录。本小节将教授如何使用包含以下两个函数的`JSONstreams.go`实用程序来完成这项任务：

```go
// DeSerialize decodes a serialized slice with JSON records
func DeSerialize(e *json.Decoder, slice interface{}) error {
    return e.Decode(slice)
} 
```

`DeSerialize()` 函数用于读取以 JSON 记录形式输入的数据，对其进行解码，并将其放入切片中。该函数将切片写入，该切片为 `interface{}` 数据类型，并作为参数给出，并从 `*json.Decoder` 参数的缓冲区获取输入。`*json.Decoder` 参数及其缓冲区在 `main()` 函数中定义，以避免每次都分配它，从而失去使用此类型带来的性能提升和效率。同样适用于以下 `*json.Encoder` 的使用：

```go
// Serialize serializes a slice with JSON records
func Serialize(e *json.Encoder, slice interface{}) error {
    return e.Encode(slice)
} 
```

`Serialize()` 函数接受两个参数，一个 `*json.Encoder` 和任何数据类型的切片，因此使用了 `interface{}`。该函数处理切片并将输出写入 `json.Encoder` 的缓冲区——这个缓冲区在创建编码器时作为参数传递。

由于使用了 `interface{}`，`Serialize()` 和 `DeSerialize()` 函数都可以与任何类型的 JSON 记录一起工作。

你可以用 `err := json.NewEncoder(buf).Encode(DataRecords)` 和 `err := json.NewEncoder(buf).Encode(DataRecords)` 调用分别替换 `Serialize()` 和 `DeSerialize()`。我个人更喜欢使用单独的函数，但你的口味可能不同。

`JSONstreams.go` 工具生成随机数据。运行 `JSONstreams.go` 会生成以下输出：

```go
$ go run JSONstreams.go
After Serialize:[{"key":"RESZD","value":63},{"key":"XUEYA","value":13}]
After DeSerialize:
0 {RESZD 63}
1 {XUEYA 13} 
```

在 `main()` 中生成的结构体输入切片被序列化，如输出第一行所示。之后，它被反序列化为原始的结构体切片。

## 美化打印 JSON 记录

本小节说明了如何美化打印 JSON 记录，这意味着以令人愉快和可读的格式打印 JSON 记录，而无需了解持有 JSON 记录的 Go 结构的内部结构。由于存在两种读取 JSON 记录的方式，分别是个别读取和流式读取，因此存在两种美化打印 JSON 数据的方式：作为单个 JSON 记录和作为流。因此，我们将实现两个单独的函数，分别命名为 `prettyPrint()` 和 `JSONstream()`。

`prettyPrint()` 函数的实现如下：

```go
func PrettyPrint(v interface{}) (err error) {
    b, err := json.MarshalIndent(v, "", "\t")
    if err == nil {
        fmt.Println(string(b))
    }
    return err
} 
```

所有工作都是由 `json.MarshalIndent()` 完成的，它应用缩进来格式化输出。

虽然 `json.MarshalIndent()` 和 `json.Marshal()` 都产生 JSON 文本结果（字节切片），但只有 `json.MarshalIndent()` 允许应用可定制的缩进，而 `json.Marshal()` 生成更紧凑的输出。

对于美化打印 JSON 数据流，你应该使用 `JSONstream()` 函数：

```go
func JSONstream(data interface{}) (string, error) {
  buffer := new(bytes.Buffer)
  encoder := json.NewEncoder(buffer)
  encoder.SetIndent("", "\t") 
```

`json.NewEncoder()` 函数返回一个新的编码器，该编码器将写入作为 `json.NewEncoder()` 参数传递的写入器。编码器将 JSON 值写入输出流。类似于 `json.MarshalIndent()`，`SetIndent()` 方法允许你将可定制的缩进应用于流。

```go
 err := encoder.Encode(data)
  if err != nil {
    return "", err
  }
  return buffer.String(), nil
} 
```

在我们完成配置编码器后，我们可以自由地使用 `Encode()` 处理 JSON 流。

这两个函数在 `prettyPrint.go` 中得到了说明，它使用随机数据生成 JSON 记录。运行 `prettyPrint.go` 会产生以下类型的输出：

```go
Last record: {YJOML 63}
{
    "key": "YJOML",
    "value": 63
}
[
    {
        "key": "HXNIG",
        "value": 79
    },
    {
        "key": "YJOML",
        "value": 63
    }
] 
```

之前的输出显示了单个 JSON 记录的格式化输出，然后是包含两个 JSON 记录的切片的格式化输出——所有 JSON 记录都表示为 Go 结构。

现在，我们将处理一些完全不同的事情，那就是开发一个强大的命令行工具——Go 在这方面真的很擅长。

# `viper` 包

*标志* 是特殊格式的字符串，它们被传递到程序中以控制其行为。如果您想支持多个标志和选项，自己处理标志可能会变得非常令人沮丧。Go 提供了 `flag` 包来处理命令行选项、参数和标志。尽管 `flag` 可以做很多事情，但它并不像其他外部 Go 包那样强大。因此，如果您正在开发简单的 UNIX 系统命令行工具，您可能会发现 `flag` 包非常有趣和有用。但您不是在阅读这本书来创建简单的命令行工具！因此，我将跳过 `flag` 包，向您介绍一个名为 `viper` 的外部包，它是一个功能强大的 Go 包，支持大量选项。`viper` 使用 `pflag` 包而不是 `flag`，这在代码中也有所说明。

所有 `viper` 项目都遵循一个模式。首先，您初始化 `viper`，然后定义您感兴趣的部分。之后，您获取这些元素并按顺序读取它们的值以使用它们。所需的值可以直接获取，这发生在您使用标准 Go 库中的 `flag` 包时，或者间接使用配置文件。当使用 JSON、YAML、TOML、HCL 或 Java 属性格式的格式化配置文件时，`viper` 会为您解析所有内容，这可以节省您编写和调试大量 Go 代码的时间。`viper` 还允许您从 Go 结构中提取和保存值。然而，这要求 Go 结构的字段与配置文件的键匹配。

`viper` 的主页位于 GitHub 上 ([`github.com/spf13/viper`](https://github.com/spf13/viper)). 请注意，您并不需要在使用工具时强制使用 `viper` 的所有功能，只需使用您需要的功能即可。然而，如果您的命令行工具需要太多的命令行参数和标志，那么使用配置文件会更好。

## 使用命令行标志

第一个示例展示了如何编写一个简单的工具，该工具接受两个作为命令行参数的值，并在屏幕上打印它们以供验证。这意味着我们需要为这两个参数准备两个命令行标志。

相关代码在 `~/go/src/github.com/mactsouk/mGo4th/ch07/useViper` 中。你应该将 `mGo4th` 替换为本书实际的 GitHub 仓库名称，或者将其重命名为 `mGo4th`。一般来说，短目录名更方便。

之后，你必须转到 `~/go/src/github.com/mactsouk/mGo4th/ch07/useViper` 目录并运行以下命令：

```go
$ go mod init
$ go mod tidy 
```

请记住，在 `useViper.go` 准备好并包含所有必需的外部包时，应执行前面的两个命令。本书的 GitHub 仓库包含所有程序的最终版本。

`useViper.go` 的实现如下：

```go
package main
import (
    "fmt"
"github.com/spf13/pflag"
"github.com/spf13/viper"
) 
```

我们需要导入 `pflag` 和 `viper` 包，因为我们将要使用它们的功能。

```go
func aliasNormalizeFunc(f *pflag.FlagSet, n string) pflag.NormalizedName {
    switch n {
    case "pass":
        n = "password"
break
case "ps":
        n = "password"
break
    }
    return pflag.NormalizedName(n)
} 
```

`aliasNormalizeFunc()` 函数用于为标志创建额外的别名，在这种情况下，为 `--password` 标志创建别名。根据现有代码，`--password` 标志可以通过 `--pass` 或 `–ps` 访问。

```go
func main() {
    pflag.StringP("name", "n", "Mike", "Name parameter") 
```

在前面的代码中，我们创建了一个名为 `name` 的新标志，也可以通过 `-n` 访问。它的默认值是 `Mike`，其描述，在实用程序的用法中显示，是 `Name 参数`。

```go
 pflag.StringP("password", "p", "hardToGuess", "Password")
    pflag.CommandLine.SetNormalizeFunc(aliasNormalizeFunc) 
```

我们创建了一个名为 `password` 的另一个标志，也可以通过 `-p` 访问，默认值为 `hardToGuess` 并带有描述。此外，我们注册了一个规范化函数来生成 `password` 标志的别名。

```go
 pflag.Parse()
    viper.BindPFlags(pflag.CommandLine) 
```

在定义所有命令行标志之后，应使用 `pflag.Parse()` 调用。它的目的是将命令行标志解析到预定义的标志中。

此外，`viper.BindPFlags()` 调用使所有标志对 `viper` 包可用。严格来说，我们说 `viper.BindPFlags()` 调用将现有的 `pflag` 标志集（`pflag.FlagSet`）绑定到 `viper`。

```go
 name := viper.GetString("name")
    password := viper.GetString("password") 
```

前面的命令显示你可以使用 `viper.GetString()` 读取两个 `string` 命令行标志的值。

```go
 fmt.Println(name, password)
    // Reading an Environment variable
    viper.BindEnv("GOMAXPROCS")
    val := viper.Get("GOMAXPROCS")
    if val != nil {
        fmt.Println("GOMAXPROCS:", val)
    } 
```

`viper` 包也可以与环境变量一起工作。我们首先需要调用 `viper.BindEnv()` 来告诉 `viper` 我们对哪个环境变量感兴趣，然后我们可以通过调用 `viper.Get()` 来读取它的值。如果 `GOMAXPROCS` 还未设置，这意味着它的值是 `nil`，则 `fmt.Println()` 调用将不会执行。

```go
 // Setting an Environment variable
    viper.Set("GOMAXPROCS", 16)
    val = viper.Get("GOMAXPROCS")
    fmt.Println("GOMAXPROCS:", val)
} 
```

同样，我们可以使用 `viper.Set()` 改变环境变量的值。

好事是 `viper` 自动提供用法信息：

```go
$ go build useViper.go
$ ./useViper.go --help
Usage of ./useViper:
  -n, --name string       Name parameter (default "Mike")
  -p, --password string   Password (default "hardToGuess")
pflag: help requested
exit status 2 
```

不带任何命令行参数使用 `useViper.go` 会产生以下类型的输出：

```go
$ go run useViper.go
Mike hardToGuess
GOMAXPROCS: 16 
```

然而，如果我们为命令行标志提供值，输出将会略有不同。

```go
$ go run useViper.go -n mtsouk -p d1ff1cultPAssw0rd
mtsouk d1ff1cultPAssw0rd
GOMAXPROCS: 16 
```

在第二种情况下，我们使用了命令行标志的快捷方式，因为这样做更快。

下一个子节讨论了在 `viper` 中使用 JSON 文件存储配置信息。

## 读取 JSON 配置文件

`viper` 包可以读取 JSON 文件以获取其配置，本小节将说明如何操作。使用文本文件来存储配置细节在编写需要大量数据和设置的复杂应用程序时非常有帮助。这可以在 `jsonViper.go` 中看到。再次强调，我们需要将 `jsonViper.go` 放在 `~/go/src` 目录下，就像之前做的那样：`~/go/src/github.com/mactsouk/mGo4th/ch07/jsonViper`。`jsonViper.go` 的代码如下：

```go
package main
import (
    "encoding/json"
"fmt"
"os"
"github.com/spf13/viper"
)
type ConfigStructure struct {
    MacPass     string `mapstructure:"macos"`
    LinuxPass   string `mapstructure:"linux"`
    WindowsPass string `mapstructure:"windows"`
    PostHost    string `mapstructure:"postgres"`
    MySQLHost   string `mapstructure:"mysql"`
    MongoHost   string `mapstructure:"mongodb"`
} 
```

这里有一个重要的观点：尽管我们使用 JSON 文件来存储配置，但 Go 结构使用 `mapstructure` 而不是 `json` 作为 JSON 配置文件字段的类型。

```go
var **CONFIG** = ".config.json"
func main() {
    if len(os.Args) == 1 {
        fmt.Println("Using default file", CONFIG)
    } else {
        CONFIG = os.Args[1]
    }
    viper.SetConfigType("json")
    viper.SetConfigFile(CONFIG)
    fmt.Printf("Using config: %s\n", viper.ConfigFileUsed())
    viper.ReadInConfig() 
```

之前的四个语句声明我们正在使用一个 JSON 文件，让 `viper` 知道默认配置文件的路径，打印所使用的配置文件，并读取和解析该配置文件。

请记住，`viper` 并不会检查配置文件实际上是否存在且可读。如果文件找不到或无法读取，`viper.ReadInConfig()` 就像它正在处理一个空配置文件一样处理。

```go
 if viper.IsSet("macos") {
        fmt.Println("macos:", viper.Get("macos"))
    } else {
        fmt.Println("macos not set!")
    } 
```

`viper.IsSet()` 调用检查配置中是否存在名为 `macos` 的键。如果已设置，它将使用 `viper.Get("macos")` 读取其值并在屏幕上打印。

```go
 if viper.IsSet("active") {
        value := viper.GetBool("active")
        if value {
            postgres := viper.Get("postgres")
            mysql := viper.Get("mysql")
            mongo := viper.Get("mongodb")
            fmt.Println("P:", postgres, "My:", mysql, "Mo:", mongo)
        }
    } else {
        fmt.Println("active is not set!")
    } 
```

在上述代码中，我们在读取值之前检查 `active` 键是否存在。如果其值等于 `true`，则从另外三个键（名为 `postgres`、`mysql` 和 `mongodb`）中读取值。

由于活动键应该持有布尔值，我们使用 `viper.GetBool()` 来读取它。

```go
 if !viper.IsSet("DoesNotExist") {
        fmt.Println("DoesNotExist is not set!")
    } 
```

如预期的那样，尝试读取一个不存在的键会失败。

```go
 var t ConfigStructure
    err := viper.Unmarshal(&t)
    if err != nil {
        fmt.Println(err)
        return
    } 
```

`viper.Unmarshal()` 调用允许您将 JSON 配置文件中的信息放入正确定义的 Go 结构中——这是可选的但很方便。

```go
 PrettyPrint(t)
} 
```

在本章前面已经介绍了 `PrettyPrint()` 函数的实现。

现在，您需要下载 `jsonViper.go` 的依赖项：

```go
$ go mod init
$ go mod tidy 
```

当前目录的内容如下：

```go
$ ls -l
total 120
-rw-r--r--@ 1 mtsouk  staff    745 Aug 21 18:21 go.mod
-rw-r--r--@ 1 mtsouk  staff  48357 Aug 21 18:21 go.sum
-rw-r--r--@ 1 mtsouk  staff   1418 Aug  3 07:51 jsonViper.go
-rw-r--r--@ 1 mtsouk  staff    188 Aug 21 18:20 myConfig.json 
```

用于测试的 `myConfig.json` 文件内容如下：

```go
{
"macos": "pass_macos",
"linux": "pass_linux",
"windows": "pass_windows",
"active": true,
"postgres": "machine1",
"mysql": "machine2",
"mongodb": "machine3"
} 
```

在前面的 JSON 文件上运行 `jsonViper.go` 产生以下输出：

```go
$ go run jsonViper.go myConfig.json
Using config: myConfig.json
macos: pass_macos
P: machine1 My: machine2 Mo: machine3
DoesNotExist is not set!
{
  "MacPass": "pass_macos",
  "LinuxPass": "pass_linux",
  "WindowsPass": "pass_windows",
  "PostHost": "machine1",
  "MySQLHost": "machine2",
  "MongoHost": "machine3"
} 
```

之前的输出是由 `jsonViper.go` 在解析 `myConfig.json` 并尝试查找所需信息时生成的。

下一节将讨论一个用于创建强大和专业命令行工具的 Go 包，例如 `docker` 和 `kubectl`。

# cobra 包

`cobra` 是一个非常实用且流行的 Go 包，它允许您使用命令、子命令和别名来开发命令行实用程序。如果您曾经使用过 `hugo`、`docker` 或 `kubectl`，您将立即意识到 `cobra` 包的作用，因为这些工具都是使用 `cobra` 开发的。命令可以有一个或多个别名，这在您想取悦业余和经验丰富的用户时非常有用。`cobra` 还支持持久标志和局部标志，分别是适用于所有命令的标志和仅适用于给定命令的标志。此外，默认情况下，`cobra` 使用 `viper` 来解析其命令行参数。

所有 Cobra 项目都遵循相同的发展模式。您使用 Cobra 命令行工具，创建命令，然后对生成的 Go 源代码文件进行所需的更改以实现所需的功能。根据您工具的复杂度，您可能需要对创建的文件进行大量更改。尽管 `cobra` 节省了您大量时间，但您仍然需要编写实现每个命令所需功能的代码。

为了正确下载 `cobra` 二进制文件，您需要采取一些额外的步骤：

```go
$ GO111MODULE=on go install github.com/spf13/cobra-cli@latest 
```

之前的命令下载了 `cobra-cli` 二进制文件——这是 `cobra` 可执行二进制文件的新名称。您不需要了解所有支持的环境变量，例如 `GO111MODULE`，但有时它们可以帮助您解决与 Go 安装相关的复杂问题。因此，如果您想了解您当前的 Go 环境，您可以使用 `go env` 命令。

由于我更喜欢使用较短的实用程序名称，我将 `cobra-cli` 重命名为 `cobra`。如果您不知道如何做到这一点或者您更喜欢使用 `cobra-cli`，请在所有命令中将 `~/go/bin/cobra` 替换为 `~/go/bin/cobra-cli`。

为了本节的目的，我们需要在 `ch07` 目录下创建一个单独的目录。正如本书多次提到的，如果您将代码放在 `~/go/src` 内的某个地方，一切都会变得容易得多；确切的位置取决于您，但使用类似 `~/go/src/github.com/mactsouk/mGo4th/ch07/go-cobra` 这样的结构会更好，其中 `mGo4th` 是您保存本书源代码文件的目录名称。假设您将使用上述目录，您需要执行以下命令（**如果您已经下载了本书的源代码，您不需要做任何事情，因为所有内容都将在那里**）：

```go
$ cd ~/go/src/github.com/mactsouk/mGo4th/ch07/
$ mkdir go-cobra # only required if the directory is not there
$ cd go-cobra
$ go mod init
go: creating new go.mod: module github.com/mactsouk/mGo4th/ch07/go-cobra
$ ~/go/bin/cobra init
Using config file: /Users/mtsouk/.cobra.yaml
Your Cobra application is ready at
/Users/mtsouk/go/src/github.com/mactsouk/mGo4th/ch07/go-cobra
$ go mod tidy
go: finding module for package github.com/spf13/viper
go: finding module for package github.com/spf13/cobra
go: downloading github.com/spf13/cobra v1.7.0
...
go: downloading github.com/rogpeppe/go-internal v1.9.0
go: downloading github.com/kr/text v0.2.0 
```

所有以 `go:` 开头的输出行都与 Go 模块相关，并且只会出现一次。如果您尝试执行当前为空的实用程序，您将得到以下输出：

```go
$ go run main.go
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application. 
```

最后几行是 `cobra` 项目的默认消息。我们稍后将对该消息进行修改。现在，您已经准备好开始使用 `cobra` 工具，并为我们正在开发的命令行实用程序添加命令。

## 一个包含三个命令的工具

本节说明了 `cobra add` 命令的使用，该命令用于向现有的 `cobra` 项目添加新命令。命令的名称是 `one`、`two` 和 `three`：

```go
$ ~/go/bin/cobra add one
Using config file: /Users/mtsouk/.cobra.yaml
one created at /Users/mtsouk/go/src/github.com/mactsouk/go-cobra
$ ~/go/bin/cobra add two 
$ ~/go/bin/cobra add three 
```

之前的命令在 `cmd` 文件夹中创建了三个新文件，分别命名为 `one.go`、`two.go` 和 `three.go`，它们是三个命令的初始天真实现。

你通常应该做的第一件事是从 `root.go` 中删除任何不需要的代码，并更改工具和每个命令的消息，如 `Short` 和 `Long` 字段中所述。然而，如果你想，你也可以保持源文件不变。

下一个子节通过添加命令行标志到命令中丰富了工具的功能。

## 添加命令行标志

我们将创建两个全局命令行标志和一个附加到给定命令（`two`）但不受其他两个命令支持的命令行标志。全局命令行标志定义在 `./cmd/root.go` 文件中。我们将定义两个名为 `directory` 的全局标志，它是一个字符串，以及名为 `depth` 的无符号整数。这两个全局标志都在 `./cmd/root.go` 的 `init()` 函数中定义：

```go
rootCmd.PersistentFlags().StringP("directory", "d", "/tmp", "Path")
rootCmd.PersistentFlags().Uint("depth", 2, "Depth of search")
viper.BindPFlag("directory", rootCmd.PersistentFlags().Lookup("directory"))
viper.BindPFlag("depth", rootCmd.PersistentFlags().Lookup("depth")) 
```

我们使用 `rootCmd.PersistentFlags()` 来定义全局标志，然后是标志的数据类型。第一个标志的名称是 `directory`，其快捷键是 `d`，而第二个标志的名称是 `depth`，没有快捷键——如果你想为它添加快捷键，你应该使用 `UintP()` 方法，因为 `depth` 参数是一个无符号整数。定义了两个标志后，我们通过调用 `viper.BindPFlag()` 将它们的控制权传递给 `viper`。第一个标志是 `string` 类型，而第二个是 `uint` 值。由于它们都在 `cobra` 项目中可用，我们调用 `viper.GetString("directory")` 来获取 `directory` 标志的值，并调用 `viper.GetUint("depth")` 来获取 `depth` 标志的值。这不是读取标志值并使用它的唯一方法。当我们在更新统计应用程序时，你将看到另一种方法。

最后，我们通过在 `./cmd/two.go` 文件的 `init()` 函数中添加下一行来添加一个仅对 `two` 命令可用的命令行标志：

```go
twoCmd.Flags().StringP("username", "u", "Mike", "Username") 
```

标志的名称是 `username`，其快捷键是 `u`。由于这是一个仅对 `two` 命令可用的本地标志，我们只能在 `./cmd/two.go` 文件中通过调用 `cmd.Flags().GetString("username")` 来获取其值。

下一个子节为现有命令创建了命令别名。

## 创建命令别名

在本节中，我们通过为现有命令创建别名来继续构建前一个子节中的代码。这意味着命令 `one`、`two` 和 `three` 分别也可以通过 `cmd1`、`cmd2` 和 `cmd3` 访问。

为了做到这一点，你需要为每个命令的 `cobra.Command` 结构添加一个名为 `Aliases` 的额外字段。`Aliases` 字段的类型是 *字符串切片*。因此，对于 `one` 命令，`./cmd/one.go` 中 `cobra.Command` 结构的开始部分将如下所示：

```go
var oneCmd = &cobra.Command{
    Use:     "one",
    Aliases: []string{"cmd1"},
    Short:   "Command one", 
```

你应该对 `./cmd/two.go` 和 `./cmd/three.go` 进行类似的修改。请记住，`one` 命令的 **内部名称** 是 `oneCmd`，并且继续如此——其他命令有类似的对内部名称。

如果你意外地将 `cmd1` 别名，或任何其他别名，放入多个命令中，Go 编译器不会抱怨。然而，只有它的第一次出现会被执行。

下一小节通过为 `one` 和 `two` 命令添加子命令来丰富这个实用程序。

## 创建子命令

本小节说明了如何为名为 `three` 的命令创建两个子命令。这两个子命令的名称将是 `list` 和 `delete`。使用 `cobra` 工具创建它们的方式如下：

```go
$ ~/go/bin/cobra add list -p 'threeCmd'
Using config file: /Users/mtsouk/.cobra.yaml
list created at /Users/mtsouk/go/src/github.com/mactsouk/mGo4th/ch07/go-cobra
$ ~/go/bin/cobra add delete -p 'threeCmd'
Using config file: /Users/mtsouk/.cobra.yaml
delete created at /Users/mtsouk/go/src/github.com/mactsouk/mGo4th/ch07/go-cobra 
```

之前的命令在 `./cmd` 中创建了两个新文件，分别命名为 `delete.go` 和 `list.go`。`-p` 标志后面跟着你想关联子命令的命令的内部名称。`three` 命令的内部名称是 `threeCmd`。你可以验证这两个命令与 `three` 命令相关联，如下所示（显示每个命令的默认消息）：

```go
$ go run main.go three delete
delete called
$ go run main.go three list
list called 
```

如果你运行 `go run main.go two list`，Go 会将 `list` 视为 `two` 的命令行参数，并且不会执行 `./cmd/list.go` 中的代码。`go-cobra` 项目的最终版本具有以下结构和包含以下文件，这些文件是由 `tree(1)` 工具生成的：

```go
$ tree
.
├── LICENSE
├── cmd
│   ├── delete.go
│   ├── list.go
│   ├── one.go
│   ├── root.go
│   ├── three.go
│   └── two.go
├── go.mod
├── go.sum
└── main.go
2 directories, 10 files 
```

到目前为止，你可能想知道当你想为两个不同的命令创建具有相同名称的两个子命令时会发生什么。在这种情况下，你首先创建第一个子命令，并在创建第二个子命令之前重命名其文件。

在最后一节中也展示了 `cobra` 包的使用，其中我们彻底更新了统计应用程序。下一节讨论了随着 Go 版本 1.16 带来的一些重要新增功能。

# 重要的 Go 功能

Go 1.16 带来了一些新功能，包括在 Go 可执行文件中嵌入文件以及引入了 `os.ReadDir()` 函数、`os.DirEntry` 类型以及 `io/fs` 包。

由于这些功能与系统编程相关，它们被包含并在本章中进行了探讨。我们首先介绍将文件嵌入到 Go 可执行文件中的方法。

## 嵌入文件

本节介绍了一个功能，允许你 **将静态资源嵌入到 Go 可执行文件中**。允许保留嵌入文件的数据类型有 `string`、`[]byte` 和 `embed.FS`。这意味着一个 Go 可执行文件可以包含一个你执行 Go 可执行文件时不需要手动下载的文件！所提供的实用程序嵌入了两份不同的文件，它可以根据给定的命令行参数检索它们。

以下代码，保存为 `embedFiles.go`，说明了这个新的 Go 功能：

```go
package main
import (
    _ "embed"
"fmt"
"os"
) 
```

您需要 `embed` 包来在您的 Go 可执行文件中嵌入任何文件。由于 `embed` 包不是直接使用的，您需要在它前面放置 `_`，这样 Go 编译器就不会报错。

```go
//go:embed static/image.png
var f1 []byte 
```

您需要在一行开头使用 `//go:embed`，这是一个 Go 注释但以特殊方式处理，后面跟要嵌入的文件路径。在这种情况下，我们嵌入 `static/image.png`，这是一个二进制文件。下一行应定义将要存储嵌入文件数据的变量，在这种情况下，是一个名为 `f1` 的字节切片。使用字节切片推荐用于二进制文件，因为我们将直接使用该字节切片来保存该二进制文件。

```go
//go:embed static/textfile
var f2 string 
```

在这种情况下，我们将 `static/textfile` 的内容保存到名为 `f2` 的字符串变量中。

```go
func writeToFile(s []byte, path string) error {
    fd, err := os.OpenFile(path, os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer fd.Close()
    n, err := fd.Write(s)
    if err != nil {
        return err
    }
    fmt.Printf("wrote %d bytes\n", n)
    return nil
} 
```

`writeToFile()` 函数用于将字节切片存储到文件中，并且是一个可以在其他情况下使用的辅助函数。

```go
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Print select 1|2")
        return
    }
    fmt.Println("f1:", len(f1), "f2:", len(f2)) 
```

此语句打印 `f1` 和 `f2` 变量的长度，以确保它们代表嵌入文件的尺寸。

```go
 switch arguments[1] {
    case "1":
        filename := "/tmp/temporary.png"
        err := writeToFile(f1, filename)
        if err != nil {
            fmt.Println(err)
            return
        }
    case "2":
        fmt.Print(f2)
    default:
        fmt.Println("Not a valid option!")
    }
} 
```

`switch` 块负责向用户返回所需的文件——在 `static/textfile` 的情况下，文件内容被打印到屏幕上。对于二进制文件，我们决定将其存储为 `/tmp/temporary.png`。

这次，我们将编译 `embedFiles.go` 以使事情更现实，因为它是一个包含嵌入文件的可执行二进制文件。我们使用 `go build embedFiles.go` 构建二进制文件。运行 `embedFiles` 产生以下类型的输出：

```go
$ ./embedFiles 2
f1: 75072 f2: 14
Data to write
$ ./embedFiles 1
f1: 75072 f2: 14
wrote 75072 bytes 
```

以下输出验证了 `temporary.png` 位于正确的路径（`/tmp/temporary.png`）：

```go
$ ls -l /tmp/temporary.png 
-rw-r--r--  1 mtsouk  wheel  75072 Feb 25 15:20 /tmp/temporary.png 
```

使用嵌入功能，我们可以创建一个嵌入其自身源代码并在执行时打印到屏幕上的实用程序！这是一种有趣的嵌入文件的方式。`printSource.go` 的源代码如下：

```go
package main
import (
    _ "embed"
"fmt"
)
//go:embed printSource.go
var src string
func main() {
    fmt.Print(src)
} 
```

如前所述，被嵌入的文件在 `//go:embed` 行中定义。运行 `printSource.go` 将上述代码打印到屏幕上。

## ReadDir 和 DirEntry

本节讨论 `os.ReadDir()` 和 `os.DirEntry`。然而，它首先讨论了 `io/ioutil` 包的弃用——`io/ioutil` 包的功能已转移到其他包。因此，我们有以下内容：

+   `os.ReadDir()` 是一个新函数，返回 `[]DirEntry`。这意味着它不能直接替换返回 `[]FileInfo` 的 `ioutil.ReadDir()`。尽管 `os.ReadDir()` 和 `os.DirEntry` 都没有提供任何新功能，但它们使事情更快更简单，这很重要。

+   `os.ReadFile()` 函数直接替换了 `ioutil.ReadFile()`。

+   `os.WriteFile()` 函数可以直接替换 `ioutil.WriteFile()`。

+   同样，`os.MkdirTemp()`可以替换`ioutil.TempDir()`而不做任何更改。然而，由于`os.TempDir()`的名称已经被占用，新的函数名称不同。

+   `os.CreateTemp()`函数与`ioutil.TempFile()`相同。尽管`os.TempFile()`的名称没有被占用，但 Go 团队决定将其命名为`os.CreateTemp()`，以便与`os.MkdirTemp()`保持一致。

+   `os.ReadDir()`和`os.DirEntry`可以在`io/fs`包中找到，作为`fs.ReadDir()`和`fs.DirEntry`，以与`io/fs`中找到的文件系统接口一起工作。

`ReadDirEntry.go`实用程序展示了`os.ReadDir()`的使用。此外，我们将在下一节中看到`fs.DirEntry`与`fs.WalkDir()`结合使用的情况——`io/fs`只支持`WalkDir()`，它默认使用`DirEntry`。`fs.WalkDir()`和`filepath.WalkDir()`都使用`DirEntry`而不是`FileInfo`。这意味着，为了在遍历目录树时看到任何性能提升，您需要将`filepath.Walk()`调用更改为`filepath.WalkDir()`调用。

所提供的实用程序使用`os.ReadDir()`来计算目录树的大小，并借助以下函数：

```go
func GetSize(path string) (int64, error) {
    contents, err := os.ReadDir(path)
    if err != nil {
        return -1, err
    }
    var total int64
for _, entry := range contents {
        // Visit directory entries
if entry.IsDir() { 
```

如果`entry.IsDir()`的返回值是`true`，那么我们处理一个目录，这意味着我们需要继续深入挖掘。

```go
 temp, err := GetSize(filepath.Join(path, entry.Name()))
            if err != nil {
                return -1, err
            }
            total += temp
            // Get size of each non-directory entry
        } else { 
```

如果我们处理一个文件，我们只需要获取它的大小。这涉及到调用`Info()`来获取关于文件的一般信息，然后调用`Size()`来获取它的大小：

```go
 info, err := entry.Info()
            if err != nil {
                return -1, err
            }
            // Returns an int64 value
            total += info.Size()
        }
    }
    return total, nil
} 
```

运行`ReadDirEntry.go`会产生以下输出，这表明实用程序按预期工作：

```go
$ go run ReadDirEntry.go /usr/bin
Total Size: 240527817 
```

最后，请记住，`ReadDir`和`DirEntry`都是从 Python 编程语言中复制的。

下一节介绍了`io/fs`包。

## `io/fs`包

本节说明了`io/fs`包的功能，该包也在 Go 1.16 中引入。由于`io/fs`提供了一种独特类型的函数，因此我们开始本节，解释它能够做什么。简单来说，`io/fs`提供了一个名为`FS`的只读文件系统接口。请注意，`embed.FS`实现了`fs.FS`接口，这意味着`embed.FS`可以利用`io/fs`包提供的一些功能。**这意味着您的应用程序可以创建自己的内部文件系统并处理其文件**。

下面的代码示例，保存为`ioFS.go`，通过将`./static`文件夹中的所有文件放入其中来创建一个文件系统。`ioFS.go`支持以下功能：列出所有文件，搜索文件名，以及使用`list()`、`search()`和`extract()`分别提取文件。我们首先展示`list()`的实现：

```go
func list(f embed.FS) error {
    return fs.WalkDir(f, ".", walkFunction)
} 
```

请记住，`fs.WalkDir()`与常规文件系统以及`embed.FS`文件系统一起工作。您可以通过运行`go doc fs.WalkDirFunc`来了解更多关于`walkFunction()`签名的信息。

在这里，我们从文件系统的给定目录开始，访问其内容。文件系统存储在 `f` 中，根目录定义为 `"."`。之后，所有魔法都在 `walkFunction()` 函数中发生，该函数的实现如下：

```go
func walkFunction(path string, d fs.DirEntry, err error) error {
    if err != nil {
        return err
    }
    fmt.Printf("Path=%q, isDir=%v\n", path, d.IsDir())
    return nil
} 
```

`walkFunction()` 函数以期望的方式处理给定根目录中的每个条目。请注意，`walkFunc()` 是由 `fs.WalkDir()` **自动调用**的，以访问每个文件或目录。

然后，我们展示 `extract()` 函数的实现：

```go
func extract(f embed.FS, filepath string) ([]byte, error) {
    s, err := fs.ReadFile(f, filepath)
    if err != nil {
        return nil, err
    }
    return s, nil
} 
```

使用 `ReadFile()` 函数从 `embed.FS` 文件系统检索一个文件，该文件通过其文件路径标识，并以字节切片的形式返回，这是 `extract()` 函数的返回值。

最后，我们有 `search()` 函数的实现，该函数基于 `walkSearch()`：

```go
func walkSearch(path string, d fs.DirEntry, err error) error {
    if err != nil {
        return err
    }
    if d.Name() == searchString { 
```

`searchString` 是一个全局变量，用于存储搜索字符串。当找到匹配项时，匹配的路径会打印在屏幕上。

```go
 fileInfo, err := fs.Stat(f, path)
        if err != nil {
            return err
        }
        fmt.Println("Found", path, "with size", fileInfo.Size())
        return nil
    } 
```

在打印匹配项之前，我们调用 `fs.Stat()` 来获取更多关于它的详细信息：

```go
 return nil
} 
```

`main()` 函数特别调用这三个函数。运行 `ioFS.go` 产生以下类型的输出：

```go
$ go run ioFS.go
Path=".", isDir=true
Path="static", isDir=true
Path="static/file.txt", isDir=false
Path="static/image.png", isDir=false
Path="static/textfile", isDir=false
Found static/file.txt with size 14
wrote 14 bytes 
```

初始时，实用程序列出文件系统中的所有文件（以 `Path` 开头的行）。然后，它验证 `static/file.txt` 是否可以在文件系统中找到。最后，它验证将 14 个字节写入新文件是否成功，因为所有 14 个字节都已写入。

因此，Go 版本 1.16 引入了重要的功能。

在下一节中，我们将改进统计应用程序。

# 更新统计应用程序

在本节中，我们将更改统计应用程序存储其数据所使用的格式。这次，统计应用程序将使用 JSON 格式。此外，它使用 `cobra` 包来实现支持的命令。

然而，在继续统计应用程序之前，我们将更多地了解 `slog` 包。

## The slog 包

Go 1.21 中将 `log/slog` 包添加到标准库中，以改进原始的 `log` 包。您可以在 [`pkg.go.dev/log/slog`](https://pkg.go.dev/log/slog) 找到更多关于它的信息。将其包含在本章中的主要原因是它能够以 JSON 格式创建日志条目，这在您想要进一步处理日志条目时非常有用。

将展示 `useSLog.go` 代码，该代码说明了 `log/slog` 包的使用，分为三个部分。第一部分如下：

```go
package main
import (
    "fmt"
"log/slog"
"os"
)
func main() {
    slog.Error("This is an ERROR message")
    slog.Debug("This is a DEBUG message")
    slog.Info("This ia an INFO message")
    slog.Warn("This is a WARNING message") 
```

首先，我们需要导入 `log/slog` 以使用 Go 标准库中的 `slog` 包。当使用默认日志记录器时，我们可以使用 `Error()`、`Debug()`、`Info()` 和 `Warn()` 发送消息，这是使用该功能的最简单方式。

第二部分包含以下代码：

```go
 logLevel := &slog.LevelVar{}
    fmt.Println("Log level:", logLevel)
    // Text Handler
    opts := &slog.HandlerOptions{
        Level: logLevel,
    }
    handler := slog.NewTextHandler(os.Stdout, opts)
    logger := slog.New(handler)
    logLevel.Set(slog.LevelDebug)
    logger.Debug("This is a DEBUG message") 
```

在这个程序的部分，我们使用 `&slog.LevelVar{}` 获取当前的日志级别，并将其更改为 `Debug` 级别，以便也能获取使用 `logger.Debug()` 发布的日志条目。这是通过 `logLevel.Set(slog.LevelDebug)` 语句实现的。

`useSLog.go` 的最后一部分包含以下代码：

```go
 // JSON Handler
    logJSON := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    logJSON.Error("ERROR message in JSON")
} 
```

所示的代码创建了一个写入 JSON 记录的日志记录器。

运行 `useSLog.go` 产生以下输出：

```go
$ go run useSLog.go
2023/08/22 21:49:18 ERROR This is an ERROR message
2023/08/22 21:49:18 INFO This ia an INFO message
2023/08/22 21:49:18 WARN This is a WARNING message 
```

之前的输出来自 `main()` 的前四个语句。`slog.Debug()` 语句没有生成输出，因为默认情况下不会打印 `DEBUG` 级别。

```go
Log level: LevelVar(INFO)
time=2023-08-22T21:49:18.474+03:00 level=DEBUG msg="This is a DEBUG message" 
```

之前的输出显示，如果我们提高日志级别，我们可以打印出 `DEBUG` 消息——默认的日志级别是 `INFO`。

```go
{"time":"2023-08-22T21:49:18.474392+03:00","level":"ERROR","msg":"ERROR message in JSON"} 
```

输出的最后一行显示了以 JSON 格式的日志信息。如果我们想将日志条目存储在常规数据库或时间序列数据库中以便进一步处理、可视化或数据分析，这将非常有用。

## 将日志发送到 io.Discard

本小节介绍了一个涉及使用 `io.Discard` 发送日志条目的技巧——`io.Discard` 会丢弃所有 `Write()` 调用而不做任何操作！尽管我们将此技巧应用于日志文件，但它也可以用于涉及写入数据的其他情况。`discard.go` 中 `main()` 函数的实现如下：

```go
func main() {
    if len(os.Args) == 1 {
        log.Println("Enabling logging!")
        log.SetOutput(os.Stderr)
    } else {
        log.SetOutput(os.Stderr)
        log.Println("Disabling logging!")
        **log.SetOutput(io.Discard)**
        log.Println("NOT GOING TO GET THAT!")
    }
} 
```

启用或禁用写入的条件很简单：当存在单个命令行参数时，启用日志记录；否则，禁用。禁用日志的语句是 `log.SetOutput(io.Discard)`。然而，在禁用日志之前，我们打印一条日志条目说明这一点。

运行 `discard.go` 产生以下输出：

```go
$ go run discard.go
2023/08/22 21:35:17 Enabling logging!
$ go run discard.go 1
2023/08/22 21:35:21 Disabling logging! 
```

在第二次程序执行中，`log.Println("NOT GOING TO GET THAT!")` 语句没有产生输出，因为它被发送到了 `io.Discard`。

在考虑所有这些信息后，让我们在 `cobra` 的帮助下继续实现统计应用程序。

## 使用 cobra

首先，我们需要创建一个地方来托管统计应用程序的 `cobra` 版本。在这个阶段，你有两个选择：要么创建一个单独的 GitHub 仓库，要么将必要的文件放在 `~/go/src` 目录下。本小节将遵循后者。因此，所有相关的代码都将位于 `~/go/src/github.com/mactsouk/mGo4th/ch07/stats`。

项目已经存在于 `stats` 目录中。如果你想要自己创建它，这些步骤是有意义的。

首先，我们需要创建并进入相关的目录：

```go
$ cd ~/go/src/github.com/mactsouk/mGo4th/ch07/stats 
```

之后，我们应该声明我们想要使用 Go 模块：

```go
$ go mod init
go: creating new go.mod: module github.com/mactsouk/mGo4th/ch07/stats 
```

之后，我们需要运行 `cobra init` 命令：

```go
$ ~/go/bin/cobra init
Using config file: /Users/mtsouk/.cobra.yaml
Your Cobra application is ready at
/Users/mtsouk/go/src/github.com/mactsouk/mGo4th/ch07/stats 
```

然后，我们可以执行 `go mod tidy`：

```go
$ go mod tidy 
```

然后，我们应该使用 `cobra`（或 `cobra-cli`）二进制文件创建应用程序的结构。**一旦我们有了结构，就很容易知道我们需要实现什么**。应用程序的结构基于支持的命令和功能：

```go
$ ~/go/bin/cobra add list
$ ~/go/bin/cobra add delete
$ ~/go/bin/cobra add insert
$ ~/go/bin/cobra add search 
```

在这一点上，执行 `go run main.go` 将会下载任何缺失的包并生成默认的 `cobra` 输出。

我们需要创建一个命令行标志来启用和禁用日志记录。我们将使用 `log/slog` 包。该标志名为 `--log`，它将是一个布尔变量。位于 `root.go` 中的相关语句如下：

```go
rootCmd.PersistentFlags().BoolVarP(&disableLogging, "log", "l", false, "Logging information") 
```

前面的声明由一个全局变量支持，定义如下：

```go
var disableLogging bool 
```

这是在使用命令行标志方面与我们之前在 `go-cobra` 项目中所做的方法不同的一个方法。

因此，`disableLogging` 全局变量的值保存了 `--log` 标志的值。尽管 `disableLogging` 变量是全局的，但你必须在每个命令中定义**一个单独的日志变量**。

下一个子节讨论了 JSON 数据的存储和加载。

## 存储和加载 JSON 数据

`saveJSONFile()` 辅助函数的这个功能在 `./cmd/root.go` 中实现，使用以下函数：

```go
func saveJSONFile(filepath string) error {
    f, err := os.Create(filepath)
    if err != nil {
        return err
    }
    defer f.Close()
    err = Serialize(&data, f)
    return err
} 
```

基本上，我们只需要使用 `Serialize()` 将结构体切片序列化，并将结果保存到文件中。接下来，我们需要能够从该文件中加载 JSON 数据。加载功能也在 `./cmd/root.go` 中实现，使用 `readJSONFile()` 辅助函数：

```go
func readJSONFile(filepath string) error {
    _, err := os.Stat(filepath)
    if err != nil {
        return err
    }
    f, err := os.Open(filepath)
    if err != nil {
        return err
    }
    defer f.Close()
    err = DeSerialize(&data, f)
    if err != nil {
        return err
    }
    return nil
} 
```

我们所需要做的就是读取包含 JSON 数据的数据文件，并通过反序列化将其数据放入结构体切片中。

现在，我们将讨论 `list` 和 `insert` 命令的实现。其他两个命令（`delete` 和 `search`）有类似的实现。

## 实现列表命令

`./cmd/list.go` 中的重要代码在 `list()` 函数的实现中：

```go
func list() {
    sort.Sort(DFslice(data))
    text, err := PrettyPrintJSONstream(data)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(text) 
```

`list` 的核心功能包含在上面的代码中，它使用 `PrettyPrintJSONstream(data)` 对 `data` 切片进行排序并格式化打印 JSON 记录。

```go
 logger = slog.New(slog.NewJSONHandler(os.Stderr, nil))
    if disableLogging == false {
        logger = slog.New(slog.NewJSONHandler(io.Discard, nil))
    }
    slog.SetDefault(logger)
    s := fmt.Sprintf("%d records in total.", len(data))
    logger.Info(s)
} 
```

前面的代码根据 `disableLogging` 的值处理日志，该值基于 `--log` 标志。

## 实现插入命令

`insert` 命令的实现如下：

```go
var insertCmd = &cobra.Command{
    Use:   "insert",
    Short: "Insert command",
    Long: `The insert command reads a datafile and stores
    its data into the application in JSON format.`,
    Run: func(cmd *cobra.Command, args []string) {
        logger = slog.New(slog.NewJSONHandler(os.Stderr, nil))
        // Work with logger
if disableLogging == false {
            logger = slog.New(slog.NewJSONHandler(io.Discard, nil))
        }
        slog.SetDefault(logger) 
```

首先，我们根据 `disableLogging` 的值为 `insert` 命令定义一个单独的日志记录器。

```go
 if file == "" {
            logger.Info("Need a file to read!")
            return
        }
        _, ok := index[file]
        if ok {
            fmt.Println("Found key:", file)
            delete(index, file)
        }
        // Now, delete it from data
if ok {
            for i, k := range data {
                if k.Filename == file {
                    data = slices.Delete(data, i, i+1)
                    break
                }
            }
        } 
```

然后，前面的代码确保 `file` 变量（包含数据的文件路径）不为空。另外，如果 `file` 是 `index` 映射的一个键，这意味着我们之前已经处理过该文件——**我们假设没有两个数据集有相同的文件名，因为对我们来说，文件名是唯一标识数据集的东西**。在这种情况下，我们从 `data` 切片和 `index` 映射中删除它，并再次处理它。这与 `update` 功能类似，该功能应用程序不支持。

```go
 err := ProcessFile(file)
        if err != nil {
            s := fmt.Sprintf("Error processing: %s", err)
            logger.Warn(s)
        }
        err = saveJSONFile(JSONFILE)
        if err != nil {
            s := fmt.Sprintf("Error saving data: %s", err)
            logger.Info(s)
        }
    },
} 
```

`insert` 命令实现的最后部分是处理给定的文件，使用 `ProcessFile()`，并使用 `saveJSONFile()` 保存 `data` 切片的更新版本。

# 摘要

本章介绍了使用环境变量、命令行参数、读取和写入纯文本文件、遍历文件系统、处理 JSON 数据以及使用`cobra`创建强大的命令行实用工具。这是本书最重要的章节之一，因为不与操作系统以及文件系统交互，不读取和保存数据，就无法创建任何真正的实用工具。

下一章将介绍 Go 语言中的并发，主要主题包括 goroutines、channels 以及安全的数据共享。我们还将讨论 UNIX 信号处理，因为 Go 使用 channels 和 goroutines 来完成这个目的。

# 练习

+   使用`byCharacter.go`、`byLine.go`和`byWord.go`的功能，创建`wc(1)` UNIX 实用工具的简化版本。

+   使用`viper`包处理命令行选项，创建`wc(1)` UNIX 实用工具的完整版本。

+   使用`cobra`包，通过命令而不是命令行选项创建`wc(1)` UNIX 实用工具的完整版本。

+   Go 提供了`bufio.Scanner`来逐行读取文件。尝试使用`bufio.Scanner`重写`byLine.go`。

+   Go 语言中的`bufio.Scanner`设计为逐行读取输入，将其分割成标记。如果你需要逐字符读取文件，一个常见的方法是结合使用`bufio.NewReader`和`Read()`或`ReadRune()`。以这种方式实现`byCharacter.go`的功能。

+   将`ioFS.go`作为`cobra`项目。

+   更新`cobra`项目中的统计应用程序，以便也将数据集的归一化版本存储在`data.json`中。

+   `byLine.go`实用工具使用`ReadString('\n')`读取输入文件。修改代码以使用`Scanner` ([`pkg.go.dev/bufio#Scanner`](https://pkg.go.dev/bufio#Scanner))进行读取。

+   类似地，`byWord.go`使用`ReadString('\n')`读取输入文件。修改代码以使用`Scanner`代替。

# 其他资源

+   `viper`包：[`github.com/spf13/viper`](https://github.com/spf13/viper)

+   `cobra`包：[`github.com/spf13/cobra`](https://github.com/spf13/cobra)

+   `encoding/json`的文档：[`pkg.go.dev/encoding/json`](https://pkg.go.dev/encoding/json)

+   `io/fs`的文档：[`pkg.go.dev/io/fs`](https://pkg.go.dev/io/fs)

+   Go `slog`包：[`gopherguides.com/articles/golang-slog-package`](http://gopherguides.com/articles/golang-slog-package)

+   使用`slog`在 Go 中进行日志记录的全面指南：[`betterstack.com/community/guides/logging/logging-in-go/`](https://betterstack.com/community/guides/logging/logging-in-go/)

+   端序：[`en.wikipedia.org/wiki/Endianness`](https://en.wikipedia.org/wiki/Endianness)

# 加入我们的 Discord 社区

加入我们的社区 Discord 空间，与作者和其他读者进行讨论：

[`discord.gg/FzuQbc8zd6`](https://discord.gg/FzuQbc8zd6)

![](https://discord.gg/FzuQbc8zd6)
