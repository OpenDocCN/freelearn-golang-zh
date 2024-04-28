# 处理文件系统

本章主要讲解与 Unix 文件系统的交互。在这里，我们将从基本的读写操作到更高级的缓冲操作，如标记扫描和文件监控，一切都会涉及。

Unix 中所有用户或系统的信息都存储为文件，因此为了与系统和用户数据交互，我们必须与文件系统交互。

在本章中，我们将看到执行读写操作的不同方式，以及每种方式更注重代码的简单性，应用程序的内存使用和性能，以及执行速度。

本章将涵盖以下主题：

+   文件路径操作

+   读取文件

+   写文件

+   其他文件系统操作

+   第三方包

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅第三章，*Go 概述*。

# 处理路径

Go 提供了一系列函数，可以操作与平台无关的文件路径，主要包含在`path/filepath`和`os`包中。

# 工作目录

每个进程都有一个与之关联的目录，称为**工作目录**，通常从父进程继承。这使得可以指定相对路径 - 不以根文件夹开头。在 Unix 和 macOS 上为`/`，在 Windows 上为`C:\`（或任何其他驱动器号）。

绝对/完整路径以根目录开头，并且表示文件系统中的相同位置，即`/usr/local`。

相对路径不以根目录开头，路径以当前工作目录开头，即`documents`。

操作系统将这些路径解释为相对于当前目录，因此它们的绝对版本是工作目录和相对路径的连接。让我们看一个例子：

```go
user:~ $ cd documents

user:~/documents $ cd ../videos

user:~/videos $
```

在这里，用户位于他们的家目录`~`。用户指定要切换到`documents`目录，`cd`命令会自动将工作目录作为前缀添加到它，并移动到`~/documents`。

在进入第二个命令之前，让我们介绍一下在所有操作系统的所有目录中都可用的两个特殊文件：

+   `.`: 点是指当前目录。如果它是路径的第一个元素，则是进程的工作目录，否则它指的是它前面的路径元素（例如，在`~/./documents`中，`.`指的是`~`）。

+   `..`: 双点是指当前目录的父目录，如果它是路径的第一个元素，或者是它前面的目录的父目录（例如，在`~/images/../documents`中，`..`指的是`~/images`的父目录，`~`）。

了解这一点，我们可以轻松推断出第二个路径首先加入`~/documents/../videos`，然后解析父元素`..`，得到最终路径`~/videos`。

# 获取和设置工作目录

我们可以使用`os`包的`func Getwd() (dir string, err error)`函数来找出表示当前工作目录的路径。

更改工作目录是使用同一包的另一个函数完成的，即`func Chdir(dir string) error`，如下面的代码所示：

```go
wd, err := os.Getwd()
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println("starting dir:", wd)

if err := os.Chdir("/"); err != nil {
    fmt.Println(err)
    return
}

if wd, err = os.Getwd(); err != nil {
    fmt.Println(err)
    return
}
fmt.Println("final dir:", wd)
```

# 路径操作

`filepath`包包含不到 20 个函数，与标准库的包相比数量较少，它用于操作路径。让我们快速看一下这些函数：

+   `func Abs(path string) (string, error)`: 返回传递的路径的绝对版本（如果它不是绝对的，则将其连接到当前工作目录），然后清理它。

+   `func Base(path string) string`: 给出路径的最后一个元素（基本路径）。例如，`path/to/some/file`返回`file`。请注意，如果路径为空，此函数将返回一个`*.*`（点）路径。

+   `func Clean(path string) string`: 通过应用一系列定义的规则返回路径的最短版本。它执行操作，如替换`.`和`..`，或删除尾部分隔符。

+   `func Dir(path string) string`: 获取不包含最后一个元素的路径。这通常返回元素的父目录。

+   `func EvalSymlinks(path string) (string, error)`: 在评估符号链接后返回路径。如果提供的路径也是相对的，并且不包含绝对路径的符号链接，则路径是相对的。

+   `func Ext(path string) string`: 获取路径的文件扩展名，即以路径的最后一个元素的最终点开始的后缀，如果没有点，则为空字符串（例如`docs/file.txt`返回`.txt`）。

+   `func FromSlash(path string) string`: 用操作系统路径分隔符替换路径中找到的所有`/`（斜杠）。如果操作系统是 Windows，则此函数不执行任何操作，并在 Unix 或 macOS 下执行替换。

+   `func Glob(pattern string) (matches []string, err error)`: 查找与指定模式匹配的所有文件。如果没有匹配的文件，则结果为`nil`。它不报告在路径探索过程中发生的任何错误。它与`Match`共享语法。

+   `func HasPrefix(p, prefix string) bool`: 此函数已弃用。

+   `func IsAbs(path string) bool`: 显示路径是否为绝对路径。

+   `func Join(elem ...string) string`: 通过使用文件路径分隔符连接多个路径元素来连接它们。请注意，这也在结果上调用`Clean`。

+   `func Match(pattern, name string) (matched bool, err error)`: 验证给定的名称是否与模式匹配，允许使用通配符字符`*`和`?`，以及使用方括号的组或字符序列。

+   `func Rel(basepath, targpath string) (string, error)`: 返回从基本路径到目标路径的相对路径，如果不可能则返回错误。此函数在结果上调用`Clean`。

+   `func Split(path string) (dir, file string)`: 使用最终的尾部斜杠将路径分成两部分。结果通常是输入路径的父路径和文件名。如果没有分隔符，`dir`将为空，文件将是路径本身。

+   `func SplitList(path string) []string`: 使用列表分隔符字符返回路径列表，Unix 和 macOS 中为`:`，Windows 中为`;`。

+   `func ToSlash(path string) string`: 执行与`FromSlash`函数执行的相反替换，将每个路径分隔符更改为`/`，在 Unix 和 macOS 上不执行任何操作，并在 Windows 上执行替换。

+   `func VolumeName(path string) string`: 在非 Windows 平台上不执行任何操作。它返回引用卷的路径组件。这对本地路径和网络资源都适用。

+   `func Walk(root string, walkFn WalkFunc)` `error`: 从根目录开始，此函数递归地遍历文件树，对树的每个条目执行遍历函数。如果遍历函数返回错误，则遍历停止，并返回该错误。该函数定义如下：

```go
type WalkFunc func(path string, info os.FileInfo, err error) error
```

在继续下一个示例之前，让我们介绍一个重要的变量：`os.Args`。此变量至少包含一个值，即调用当前进程的路径。这可以跟随在同一调用中指定的可能参数。

我们想要实现一个小应用程序，列出并计算目录中的文件数量。我们可以使用刚刚看到的一些工具来实现这一点。

在下面的代码中显示了列出和计数文件的示例：

```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
)

func main() {
    if len(os.Args) != 2 { // ensure path is specified
        fmt.Println("Please specify a path.")
        return
    }
    root, err := filepath.Abs(os.Args[1]) // get absolute path
    if err != nil {
        fmt.Println("Cannot get absolute path:", err)
        return
    }
    fmt.Println("Listing files in", root)
    var c struct {
        files int
        dirs int
    }
    filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        // walk the tree to count files and folders
        if info.IsDir() {
            c.dirs++
        } else {
            c.files++
        }
        fmt.Println("-", path)
        return nil
    })
    fmt.Printf("Total: %d files in %d directories", c.files, c.dirs)
}
```

# 从文件中读取

可以使用`io/ioutil`包中的辅助函数，以及`ReadFile`函数来获取文件的内容，该函数一次性打开、读取和关闭文件。这使用一个小缓冲区（512 字节）并将整个内容加载到内存中。如果文件大小非常大，未知，或者文件内容可以一次处理一部分，这不是一个好主意。

一次从磁盘读取一个巨大的文件意味着将所有文件内容复制到主内存中，这是有限的资源。这可能会导致内存不足，以及运行时错误。一次读取文件的一部分可以帮助读取大文件的内容，而不会导致大量内存使用。这是因为在读取下一块时将重用相同部分的内存。

一次性读取所有内容的示例如下所示：

```go
package main

import (
    "fmt"
    "io/ioutil"
    "os"
)

func main() {
    if len(os.Args) != 2 {
        fmt.Println("Please specify a path.")
        return
    }
    b, err := ioutil.ReadFile(os.Args[1])
    if err != nil {
        fmt.Println("Error:", err)
    }
    fmt.Println(string(b))
}
```

# 读取器接口

对于所有从磁盘读取的操作，有一个至关重要的接口：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

它的工作非常简单 - 用已读取的内容填充给定的字节片，并返回已读取的字节数和错误（如果发生错误）。有一个特殊的错误变量由`io`包定义，称为`EOF`（**文件结束**），当没有更多输入可用时应返回它。

读取器使得可以以块的方式处理数据（大小由切片确定），如果同一切片用于后续操作，则由此产生的程序始终更加内存高效，因为它使用了相同的有限内存部分来分配切片。

# 文件结构

`os.File`类型满足读取器接口，并且是用于与文件内容交互的主要角色。获取用于读取目的的实例的最常见方法是使用`os.Open`函数。在使用完文件后记得关闭文件非常重要 - 对于短暂存在的程序可能不明显，但如果一个应用程序不断打开文件而不关闭已经完成的文件，应用程序将达到操作系统强加的打开文件限制并开始失败打开操作。

shell 提供了一些实用程序，如下所示：

+   获取打开文件的限制 - `ulimit -n`

+   另一个是检查某个进程打开了多少文件 - `lsof -p PID`

前面的示例仅打开文件以将其内容显示到标准输出，通过将所有内容加载到内存中来实现。可以很容易地通过刚才提到的工具进行优化。在下面的示例中，我们使用一个小缓冲区并在下一次读取之前打印其内容，使用小缓冲区以将内存使用量保持在最低。

使用字节数组作为缓冲区的示例如下所示：

```go
func main() {
    if len(os.Args) != 2 {
        fmt.Println("Please specify a file")
        return
    }
    f, err := os.Open(os.Args[1])
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer f.Close() // we ensure close to avoid leaks

    var (
        b = make([]byte, 16)
    )
    for n := 0; err == nil; {
        n, err = f.Read(b)
        if err == nil {
            fmt.Print(string(b[:n])) // only print what's been read
        }
    }
    if err != nil && err != io.EOF { // we expect an EOF
        fmt.Println("\n\nError:", err)
    }
}
```

如果一切正常，读取循环将继续执行读取操作，直到文件内容结束。在这种情况下，读取循环将返回一个`io.EOF`错误，表明没有更多内容可用。

# 使用缓冲区

**数据缓冲区**，或者只是一个缓冲区，是用于在数据移动时存储临时数据的一部分内存。字节缓冲区是在`bytes`包中实现的，并且由一个底层切片实现，该切片能够在需要存储的数据量不适合时进行扩展。

如果每次分配新缓冲区，旧缓冲区最终将被 GC 自行清理，这不是一个最佳解决方案。最好始终重用缓冲区而不是分配新的。这是因为它们使得可以重置切片同时保持容量不变（数组不会被 GC 清除或收集）。

缓冲区还提供了两个函数来显示其底层长度和容量。在下面的示例中，我们可以看到如何使用`Buffer.Reset`重用缓冲区以及如何跟踪其容量。

缓冲重用及其底层容量的示例如下所示：

```go
package main

import (
    "bytes"
    "fmt"
)

func main() {
    var b = bytes.NewBuffer(make([]byte, 26))
    var texts = []string{
        `As he came into the window`,
        `It was the sound of a crescendo
He came into her apartment`,
        `He left the bloodstains on the carpet`,
        `She ran underneath the table
He could see she was unable
So she ran into the bedroom
She was struck down, it was her doom`,
    }
    for i := range texts {
        b.Reset()
        b.WriteString(texts[i])
        fmt.Println("Length:", b.Len(), "\tCapacity:", b.Cap())
    }
}
```

# 窥视内容

在前面的示例中，我们固定了一定数量的字节，以便在打印之前存储内容。`bufio`包提供了一些功能，使得可以使用用户无法直接控制的底层缓冲区，并且可以执行一个非常重要的操作，名为*peek*。

**Peeking** 是在不推进阅读器光标的情况下读取内容的能力。在这里，在幕后，被窥视的数据存储在缓冲区中。每次读取操作都会检查这个缓冲区是否有数据，如果有，那么数据将被返回并从缓冲区中移除。这就像一个队列（先进先出）。

这个简单操作打开的可能性是无穷的，它们都源于窥视直到找到所需的数据序列，然后实际读取感兴趣的块。这个操作的最常见用途包括以下内容：

+   缓冲区从阅读器中读取，直到找到换行符（一次读取一行）。

+   直到找到空格为止（一次读取一个单词）。

允许应用程序实现这种行为的结构是`bufio.Scanner`。这使得可以定义分割函数是什么，并具有以下类型：

```go
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)
```

当返回错误时，此函数停止，否则它返回要在内容中前进的字节数，最终返回一个标记。包中实现的函数如下：

+   `ScanBytes`：字节标记

+   `ScanRunes`：符文标记

+   `ScanWord`：单词标记

+   `ScanLines`：行标记

我们可以实现一个文件阅读器，只需一个阅读器就可以计算行数。结果程序将尝试模拟 Unix 的`wc -l`命令的功能。

一个打印文件并计算行数的示例在以下代码中显示：

```go
func main() {
    if len(os.Args) != 2 {
        fmt.Println("Please specify a path.")
        return
    }
    f, err := os.Open(os.Args[1])
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer f.Close()
    r := bufio.NewReader(f) // wrapping the reader with a buffered one
    var rowCount int
    for err == nil {
        var b []byte
        for moar := true; err == nil && moar; {
            b, moar, err = r.ReadLine()
            if err == nil {
                fmt.Print(string(b))
            }
        }
        // each time moar is false, a line is completely read
        if err == nil {
            fmt.Println()
            rowCount++

        }
    }
    if err != nil && err != io.EOF {
        fmt.Println("\nError:", err)
        return
    }
    fmt.Println("\nRow count:", rowCount)
}
```

# Closer 和 seeker

还有另外两个与阅读器相关的接口：`io.Closer`和`io.Seeker`：

```go
type Closer interface {
        Close() error
}

type Seeker interface {
        Seek(offset int64, whence int) (int64, error)
}
```

这些通常与`io.Reader`结合使用，得到的接口如下：

```go
type ReadCloser interface {
        Reader
        Closer
}

type ReadSeeker interface {
        Reader
        Seeker
}
```

`Close`方法确保资源被释放并避免泄漏，而`Seek`方法使得可以移动当前对象（例如`Writer`）的光标到文件的起始/结束位置，或者从当前位置移动。

`os.File` 结构实现了这个方法，以满足所有列出的接口。在操作结束时关闭文件是可能的，或者根据你想要实现的目标移动当前光标。

# 写入文件

正如我们在阅读中所看到的，写入文件有不同的方式，每种方式都有其自身的缺点和优势。例如，在`ioutil`包中，我们有另一个名为`WriteFile`的函数，它允许我们在一行中执行整个操作。这包括打开文件，写入其内容，然后关闭文件。

一个一次性写入文件所有内容的示例在以下代码中显示：

```go
package main

import (
    "fmt"
    "io/ioutil"
    "os"
)

func main() {
    if len(os.Args) != 3 {
        fmt.Println("Please specify a path and some content")
        return
    }
    // the second argument, the content, needs to be casted to a byte slice
    if err := ioutil.WriteFile(os.Args[1], []byte(os.Args[2]), 0644); err != nil {
        fmt.Println("Error:", err)
    }
}
```

这个示例一次性写入所有内容。这要求我们使用一个字节切片在内存中分配所有内容。如果内容太大，内存使用可能会成为操作系统的问题，这可能会终止我们的应用程序的进程。

如果内容的大小不是很大，而且应用程序的生命周期很短，那么如果内容加载到内存中并用单个操作写入，这不是问题。这对于长期运行的应用程序来说不是最佳实践，这些应用程序对许多不同的文件进行读取和写入。它们必须在内存中分配所有内容，而该内存将在某个时刻被 GC 释放 - 这个操作不是免费的，这意味着它在内存使用和性能方面有缺点。

# Writer 接口

对于写入也适用于阅读的相同原则 - 在`io`包中有一个确定写入行为的接口，如下所示：

```go
type Writer interface {
        Write(p []byte) (n int, err error)
}
```

`io.Writer`接口定义了一个方法，给定一个字节片，返回已写入多少字节以及是否有任何错误。写入器使得可以一次写入一块数据，而无需一次性拥有所有数据。`os.File`结构也恰好是一个写入器，并且可以以这种方式使用。

我们可以使用字节片作为缓冲区逐段写入信息。在下面的示例中，我们将尝试将从上一节读取的内容与写入相结合，使用`io.Seeker`的能力在写入之前反转其内容。

下面的代码示例显示了反转文件内容的示例：

```go
// Let's omit argument check and file opening, we obtain src and dst
cur, err := src.Seek(0, os.SEEK_END) // Let's go to the end of the file
if err != nil {
    fmt.Println("Error:", err)
    return
}
b := make([]byte, 16)
```

在移动到文件末尾并定义字节缓冲区后，我们进入一个循环，该循环在文件中稍微向后移动，然后读取其中的一部分，如下面的代码所示：

```go

for step, r, w := int64(16), 0, 0; cur != 0; {
    if cur < step { // ensure cursor is 0 at max
        b, step = b[:cur], cur
    }
    cur = cur - step
    _, err = src.Seek(cur, os.SEEK_SET) // go backwards
    if err != nil {
        break
    }
    if r, err = src.Read(b); err != nil || r != len(b) {
        if err == nil { // all buffer should be read
            err = fmt.Errorf("read: expected %d bytes, got %d", len(b), r)
        }
        break
    }
```

然后，我们将内容反转并将其写入目标，如下面的代码所示：

```go
    for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
        switch { // Swap (\r\n) so they get back in place
        case b[i] == '\r' && b[i+1] == '\n':
            b[i], b[i+1] = b[i+1], b[i]
        case j != len(b)-1 && b[j-1] == '\r' && b[j] == '\n':
            b[j], b[j-1] = b[j-1], b[j]
        }
        b[i], b[j] = b[j], b[i] // swap bytes
    }
    if w, err = dst.Write(b); err != nil || w != len(b) {
        if err != nil {
            err = fmt.Errorf("write: expected %d bytes, got %d", len(b), w)
        }
    }
}
if err != nil && err != io.EOF { // we expect an EOF
    fmt.Println("\n\nError:", err)
}
```

# 缓冲区和格式

在前一节中，我们看到`bytes.Buffer`可以用于临时存储数据，并且通过附加底层切片来处理自己的增长。`fmt`包广泛使用缓冲区来执行其操作；由于依赖原因，这些不是字节包中的缓冲区。这种方法是 Go 的谚语之一：

“少量复制胜过少量依赖。”

如果必须导入一个包来使用一个函数或类型，那么应该考虑将必要的代码复制到自己的包中。如果一个包包含的内容远远超出了你的需求，复制可以减少最终二进制文件的大小。您还可以自定义代码并根据自己的需求进行调整。

缓冲区的另一个用途是在写入之前组成消息。让我们编写一些代码，以便我们可以使用缓冲区格式化书籍列表：

```go
const grr = "G.R.R. Martin"

type book struct {
    Author, Title string
    Year int
}

func main() {
    dst, err := os.OpenFile("book_list.txt", os.O_CREATE|os.O_WRONLY, 0666)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer dst.Close()
    bookList := []book{
        {Author: grr, Title: "A Game of Thrones", Year: 1996},
        {Author: grr, Title: "A Clash of Kings", Year: 1998},
        {Author: grr, Title: "A Storm of Swords", Year: 2000},
        {Author: grr, Title: "A Feast for Crows", Year: 2005},
        {Author: grr, Title: "A Dance with Dragons", Year: 2011},
        // if year is omitted it defaulting to zero value
        {Author: grr, Title: "The Winds of Winter"},
        {Author: grr, Title: "A Dream of Spring"},
    }
    b := bytes.NewBuffer(make([]byte, 0, 16))
    for _, v := range bookList {
        // prints a msg formatted with arguments to writer
        fmt.Fprintf(b, "%s - %s", v.Title, v.Author)
        if v.Year > 0 { 
            // we do not print the year if it's not there
            fmt.Fprintf(b, " (%d)", v.Year)
        }
        b.WriteRune('\n')
        if _, err := b.WriteTo(dst); true { // copies bytes, drains buffer
            fmt.Println("Error:", err)
            return
        }
    }
}
```

缓冲区用于组成书籍描述，如果不存在年份，则会被省略。在处理字节时，这是非常高效的，如果每次都重用缓冲区，效果会更好。如果此类操作的输出应该是一个字符串，则`strings`包中有一个非常相似的结构称为`Builder`，它具有相同的写入方法，但也有一些不同之处，例如以下内容：

+   `String()`方法使用`unsafe`包将字节转换为字符串，而不是复制它们。

+   不允许复制`strings.Builder`，然后对副本进行写入，因为这会导致`panic`。

# 高效写入

每次执行`os.File`方法，即`Write`，这将转换为系统调用，这是一个带有一些开销的操作。一般来说，通过一次写入更多的数据来减少在此类调用上花费的时间是一个好主意，从而最小化操作的数量。

`bufio.Writer`结构是一个包装另一个写入器（如`os.File`）的写入器，并且仅在缓冲区满时执行写入操作。这使得可以使用`Flush`方法执行强制写入，通常保留到写入过程的结束。使用缓冲区的一个良好模式如下：

```go
  var w io.WriteCloser
  // initialise writer
  defer w.Close()
  b := bufio.NewWriter(w)
  defer b.Flush()
  // write operations
```

`defer`语句在返回当前函数之前以相反的顺序执行，因此第一个`Flush`确保将缓冲区中的任何内容写入，然后`Close`实际关闭文件。如果两个操作以相反的顺序执行，flush 将尝试写入一个关闭的文件，返回错误，并且无法写入最后一块信息。

# 文件模式

我们看到`os.OpenFile`函数使得可以选择如何使用文件模式打开文件，文件模式是一个`uint32`，其中每个位都有特定含义（类似于 Unix 文件和文件夹权限）。`os`包提供了一系列值，每个值都指定一种模式，正确的组合方式是使用`|`（按位或）。

下面的代码显示了可用的代码，并且直接从 Go 的源代码中获取：

```go
// Exactly one of O_RDONLY, O_WRONLY, or O_RDWR must be specified.
O_RDONLY int = syscall.O_RDONLY // open the file read-only.
O_WRONLY int = syscall.O_WRONLY // open the file write-only.
O_RDWR int = syscall.O_RDWR // open the file read-write.
// The remaining values may be or'ed in to control behavior.
O_APPEND int = syscall.O_APPEND // append data to the file when writing.
O_CREATE int = syscall.O_CREAT // create a new file if none exists.
O_EXCL int = syscall.O_EXCL // used with O_CREATE, file must not exist.
O_SYNC int = syscall.O_SYNC // open for synchronous I/O.
O_TRUNC int = syscall.O_TRUNC // if possible, truncate file when opened.
```

前三个表示允许的操作（读、写或两者），其他的如下：

+   `O_APPEND`: 在每次写入之前，文件偏移量被定位在文件末尾。

+   `O_CREATE`: 可以创建文件（如果文件不存在）。

+   `O_EXCL`: 如果与创建一起使用，如果文件已经存在，则失败（独占创建）。

+   `O_SYNC`: 执行读/写操作并验证其完成。

+   `O_TRUNC`: 如果文件存在，其大小将被截断为`0`。

# 其他操作

读和写不是文件上可以执行的唯一操作。在下一节中，我们将看看如何使用`os`包来执行它们。

# 创建

要创建一个空文件，可以调用一个名为`Create`的辅助函数，它以`0666`权限打开一个新文件，并在文件不存在时将其截断。或者，我们可以使用`OpenFile`与`O_CREATE|O_TRUNCATE`模式来指定自定义权限，如下面的代码所示：

```go
package main

import "os"

func main() {
    f, err := os.Create("file.txt")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    f.Close()
}
```

# 截断

要截断文件内容到一定尺寸，并且如果文件较小则保持不变，可以使用`os.Truncate`方法。其用法非常简单，如下面的代码所示：

```go
package main

import "os"

func main() {
    // let's keep thing under 4kB
    if err := os.Truncate("file.txt", 4096); err != nil {
        fmt.Println("Error:", err)
    }
}
```

# 删除

为了删除文件，还有另一个简单的函数，称为`os.Remove`，如下面的代码所示：

```go
package main

import "os"

func main() {
    if err := os.Remove("file.txt"); err != nil {
        fmt.Println("Error:", err)
    }
}
```

# 移动

`os.Rename`函数可以更改文件名和/或其目录。请注意，如果目标文件已经存在，此操作将替换目标文件。

更改文件名或其目录的代码如下：

```go
import "os"

func main() {
    if err := os.Rename("file.txt", "../file.txt"); err != nil {
        fmt.Println("Error:", err)
    }
}
```

# 复制

没有唯一的函数可以复制文件，但可以使用`io.Copy`函数轻松实现。下面的示例显示了如何使用它从一个文件复制到另一个文件：

```go
func CopyFile(from, to string) (int64, error) {
    src, err := os.Open(from)
    if err != nil {
        return 0, err
    }
    defer src.Close()
    dst, err := os.OpenFile(to, os.O_WRONLY|os.O_CREATE, 0644)
    if err != nil {
        return 0, err
    }
    defer dst.Close()  
    return io.Copy(dst, src)
}
```

# 统计

`os`包提供了`FileInfo`接口，返回文件的元数据，如下面的代码所示：

```go
type FileInfo interface {
        Name() string // base name of the file
        Size() int64 // length in bytes for regular files; system-dependent for others
        Mode() FileMode // file mode bits
        ModTime() time.Time // modification time
        IsDir() bool // abbreviation for Mode().IsDir()
        Sys() interface{} // underlying data source (can return nil)
}
```

`os.Stat`函数返回指定路径文件的信息。

# 更改属性

为了与文件系统交互并更改这些属性，有三个函数可用：

+   `func Chmod(name string, mode FileMode) error`: 更改文件的权限

+   `func Chown(name string, uid, gid int) error`: 更改文件的所有者和组

+   `func Chtimes(name string, atime time.Time, mtime time.Time) error`: 更改文件的访问和修改时间

# 第三方包

社区提供了许多可以完成各种任务的包。我们将在本节中快速浏览其中一些。

# 虚拟文件系统

在 Go 中，文件是一个具体类型的结构体，没有围绕它们的抽象，而文件的信息由`os.FileInfo`表示，它是一个接口。这有点不一致，已经有许多尝试创建一个完整和一致的文件系统抽象，通常称为*虚拟文件系统*。

最常用的两个包如下：

+   `vfs`: [github.com/blang/vfs](https://github.com/blang/vfs)

+   `afero`: [github.com/spf13/afero](https://github.com/spf13/afero)

即使它们是分开开发的，它们都做同样的事情——它们定义了一个具有`os.File`所有方法的接口，然后定义了一个实现`os`包中可用的函数的接口，比如创建、打开和删除文件等。

它们提供了基于标准包实现的`os.File`版本，但也有一个使用模拟文件系统的数据结构的内存版本。这对于为任何包构建测试非常有用。

# 文件系统事件

Go 在`golang.org/x/`包中有一些实验性功能，这些功能位于 Go 的 GitHub 处理程序下（[`github.com/golang/`](https://github.com/golang/)）。`golang.org/x/sys`包是其中之一，包括一个专门用于 Unix 系统事件的子包。这已被用于构建一个在 Go 的文件功能中缺失的功能，可以非常有用 - 观察某个路径上的文件事件，如创建、删除和更新。

最著名的两个实现如下：

+   `notify`：[github.com/rjeczalik/notify](https://github.com/rjeczalik/notify)

+   `fsnotify`：[github.com/fsnotify/fsnotify](https://github.com/fsnotify/fsnotify)

这两个包都公开了一个函数，允许创建观察者。观察者是包含负责传递文件事件的通道的结构。它们还公开了另一个负责终止/关闭观察者和底层通道的函数。

# 总结

在本章中，我们概述了如何在 Go 中执行文件操作。为了定位文件，`filepath`包提供了广泛的函数数组。这些函数可以帮助您执行各种操作，从组合路径到从中提取元素。

我们还看了如何使用各种方法读取操作，从位于`io/ioutil`包中的最简单和内存效率较低的方法到需要`io.Writer`实现来读取固定大小的字节块的方法。在`bufio`包中实现的查看内容的能力的重要性，允许进行一整套操作，如读取单词或读取行，当找到一个标记时停止读取操作。有其他对文件非常有用的接口；例如，`io.Closer`确保资源被释放，`io.Seeker`用于在不需要实际读取文件和丢弃输出的情况下移动读取光标。

将字节切片写入文件可以通过不同的方式实现 - `io/ioutil`包可以通过函数调用实现，而对于更复杂或更节省内存的操作，可以使用`io.Writer`接口。这使得可以一次写入一个字节切片，并且可以被`fmt`包用于打印格式化数据。缓冲写入用于减少在磁盘上的实际写入量。这是通过一个收集内容的缓冲区来实现的，然后每次缓冲区满时将其传输到磁盘上。

最后，我们看到了如何在文件系统上执行其他文件操作（创建、删除、复制/移动和更改文件属性），并查看了一些与文件系统相关的第三方包，即虚拟文件系统抽象和文件系统事件通知。

下一章将讨论流，并将重点放在与文件系统无关的所有读取器和写入器的实例上。

# 问题

1.  绝对路径和相对路径有什么区别？

1.  如何获取或更改当前工作目录？

1.  使用`ioutil.ReadAll`的优点和缺点是什么？

1.  为什么缓冲对于读取操作很重要？

1.  何时应该使用`ioutil.WriteFile`？

1.  使用允许查看的缓冲读取器时有哪些操作可用？

1.  何时最好使用字节缓冲区读取内容？

1.  缓冲区如何用于写入？使用它们有什么优势？
