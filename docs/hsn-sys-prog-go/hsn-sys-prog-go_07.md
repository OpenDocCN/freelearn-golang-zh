# 处理流

本章涉及数据流，将输入和输出接口扩展到文件系统之外，并介绍如何实现自定义读取器和写入器以满足任何目的。

它还专注于输入和输出实用程序的缺失部分，以多种不同的方式将它们组合在一起，目标是完全控制传入和传出的数据。

本章将涵盖以下主题：

+   流

+   自定义读取器

+   自定义写入器

+   实用程序

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅第三章，*Go 概述*。

# 流

写入器和读取器不仅仅用于文件；它们是抽象数据流的接口，这些流通常被称为**流**，是大多数应用程序的重要组成部分。

# 输入和读取器

如果应用程序无法控制数据流，并且将等待错误来结束流程，则传入的数据流被视为`io.Reader`接口，在最佳情况下会收到`io.EOF`值，这是一个特殊的错误，表示没有更多内容可读取，否则会收到其他错误。另一种选择是读取器也能够终止流。在这种情况下，正确的表示是`io.ReadCloser`接口。

除了`os.File`，标准包中还有几个读取器的实现。

# 字节读取器

`bytes`包含一个有用的结构，它将字节切片视为`io.Reader`接口，并实现了许多更多的 I/O 接口：

+   `io.Reader`：这可以作为常规读取器

+   `io.ReaderAt`：这使得可以从特定位置开始读取

+   `io.WriterTo`：这使得可以在偏移量处写入内容

+   `io.Seeker`：这可以自由移动读取器的光标

+   `io.ByteScanner`：这可以为每个字节执行读取操作

+   `io.RuneScanner`：这可以对由多个字节组成的字符执行相同的操作

符文和字节之间的区别可以通过以下示例来澄清，其中我们有一个由一个符文组成的字符串`⌘`，它由三个字节`e28c98`表示：

```go
func main() {
    const a = `⌘`

    fmt.Printf("plain string: %s\n", a)
    fmt.Printf("quoted string: %q\n",a)

    fmt.Printf("hex bytes: ")
    for i := 0; i < len(a); i++ {
        fmt.Printf("%x ", a[i])
    }
    fmt.Printf("\n")
}
```

完整的示例可在[`play.golang.org/p/gVZOufSmlq1`](https://play.golang.org/p/gVZOufSmlq1)找到。

还有`bytes.Buffer`，它在`bytes.Reader`的基础上添加了写入功能，并且可以访问底层切片或将内容作为字符串获取。

`Buffer.String`方法将字节转换为字符串，在 Go 中进行此类转换是通过复制字节来完成的，因为字符串是不可变的。这意味着对缓冲区的任何更改都将在复制后进行，不会传播到字符串。

# 字符串读取器

`strings`包含另一个与`io.Reader`接口非常相似的结构，称为`strings.Reader`。它的工作方式与第一个完全相同，但底层值是字符串而不是字节切片。

在处理需要读取的字符串时，使用字符串而不是字节读取器的主要优势之一是避免在初始化时复制数据。这种微妙的差异有助于提高性能和内存使用，因为它减少了分配并需要**垃圾回收器**（**GC**）清理副本。

# 定义读取器

任何 Go 应用程序都可以定义`io.Reader`接口的自定义实现。在实现接口时的一个很好的一般规则是接受接口并返回具体类型，避免不必要的抽象。

让我们看一个实际的例子。我们想要实现一个自定义读取器，它从另一个读取器中获取内容并将其转换为大写；例如，我们可以称之为`AngryReader`：

```go
func NewAngryReader(r io.Reader) *AngryReader {
    return &AngryReader{r: r}
}

type AngryReader struct {
    r io.Reader
}

func (a *AngryReader) Read(b []byte) (int, error) {
    n, err := a.r.Read(b)
    for r, i, w := rune(0), 0, 0; i < n; i += w {
        // read a rune
        r, w = utf8.DecodeRune(b[i:])
        // skip if not a letter
        if !unicode.IsLetter(r) {
            continue
        }
        // uppercase version of the rune
        ru := unicode.ToUpper(r)
        // encode the rune and expect same length
        if wu := utf8.EncodeRune(b[i:], ru); w != wu {
            return n, fmt.Errorf("%c->%c, size mismatch %d->%d", r, ru, w, wu)
        }
    }
    return n, err
}
```

这是一个非常直接的例子，使用`unicode`和`unicode/utf8`来实现其目标：

+   `utf8.DecodeRune`用于获取第一个符文及其宽度是读取的切片的一部分

+   `unicode.IsLetter`确定符文是否为字母

+   `unicode.ToUpper`将文本转换为大写

+   `ut8.EncodeLetter`将新字母写入必要的字节

+   字母及其大写版本应该具有相同的宽度

完整示例可在[`play.golang.org/p/PhdSsbzXcbE`](https://play.golang.org/p/PhdSsbzXcbE)找到。

# 输出和写入器

适用于传入流的推理也适用于传出流。我们有`io.Writer`接口，应用程序只能发送数据，还有`io.WriteCloser`接口，它还能关闭连接。

# 字节写入器

我们已经看到`bytes`包提供了`Buffer`，它具有读取和写入功能。这实现了`ByteReader`接口的所有方法，以及一个以上的`Writer`接口：

+   `io.Writer`：这可以作为常规写入器

+   `io.WriterAt`：这使得可以从某个位置开始写入

+   io.ByteWriter：这使得可以写入单个字节

`bytes.Buffer`是一个非常灵活的结构，因为它既适用于`Writer`和`ByteWriter`，如果重复使用，它的`Reset`和`Truncate`方法效果最佳。与其让 GC 回收已使用的缓冲区并创建一个新的缓冲区，不如重置现有的缓冲区，保留缓冲区的底层数组，并将切片长度设置为`0`。

在前一章中，我们看到了缓冲区使用的一个很好的例子：

```go
    bookList := []book{
        {Author: grr, Title: "A Game of Thrones", Year: 1996},
        {Author: grr, Title: "A Clash of Kings", Year: 1998},
        {Author: grr, Title: "A Storm of Swords", Year: 2000},
        {Author: grr, Title: "A Feast for Crows", Year: 2005},
        {Author: grr, Title: "A Dance with Dragons", Year: 2011},
        {Author: grr, Title: "The Winds of Winter"},
        {Author: grr, Title: "A Dream of Spring"},
    }
    b := bytes.NewBuffer(make([]byte, 0, 16))
    for _, v := range bookList {
        // prints a msg formatted with arguments to writer
        fmt.Fprintf(b, "%s - %s", v.Title, v.Author)
        if v.Year > 0 { // we do not print the year if it's not there
            fmt.Fprintf(b, " (%d)", v.Year)
        }
        b.WriteRune('\n')
        if _, err := b.WriteTo(dst); true { // copies bytes, drains buffer
            fmt.Println("Error:", err)
            return
        }
    }
```

缓冲区不适用于组合字符串值。因此，当调用`String`方法时，字节会被转换为不可变的字符串，与切片不同。以这种方式创建的新字符串是使用当前切片的副本制作的，对切片的更改不会影响字符串。这既不是限制也不是特性；这是一个属性，如果使用不正确可能会导致错误。以下是重置缓冲区并使用`String`方法的效果示例：

```go
package main

import (
    "bytes"
    "fmt"
)

func main() {
    b := bytes.NewBuffer(nil)
    b.WriteString("One")
    s1 := b.String()
    b.WriteString("Two")
    s2 := b.String()
    b.Reset()
    b.WriteString("Hey!")    // does not change s1 or s2
    s3 := b.String()
    fmt.Println(s1, s2, s3)  // prints "One OneTwo Hey!"
}
```

完整示例可在[`play.golang.org/p/zBjGPMC4sfF`](https://play.golang.org/p/zBjGPMC4sfF)找到

# 字符串写入器

字节缓冲区执行字节的复制以生成一个字符串。这就是为什么在 1.10 版本中，`strings.Builder`首次亮相。它共享缓冲区的所有与写入相关的方法，并且不允许通过`Bytes`方法访问底层切片。获取最终字符串的唯一方法是使用`String`方法，它在底层使用`unsafe`包将切片转换为字符串而不复制底层数据。

这样做的主要后果是这个结构强烈地不鼓励复制——因为复制的切片的底层数组指向相同的数组，并且在副本中写入会影响另一个。结果的操作会导致恐慌：

```go
package main

import (
    "strings"
)

func main() {
    b := strings.Builder{}
    b.WriteString("One")
    c := b
    c.WriteString("Hey!") // panic: strings: illegal use of non-zero Builder copied by value
}
```

# 定义一个写入器

任何写入器的自定义实现都可以在应用程序中定义。一个非常常见的情况是装饰器，它是一个包装另一个写入器并改变或扩展原始写入器功能的写入器。至于读取器，最好有一个接受另一个写入器并可能包装它以使其与许多标准库结构兼容的构造函数，例如以下内容：

+   `*os.File`

+   `*bytes.Buffer`

+   `*strings.Builder`

让我们来看一个真实的用例——我们想要生成一些带有每个单词中混淆字母的文本，以测试何时开始变得无法阅读。我们将创建一个可配置的写入器，在将其写入目标写入器之前混淆字母，并创建一个接受文件并创建其混淆版本的二进制文件。我们将使用`math/rand`包来随机化混淆。

让我们定义我们的结构及其构造函数。这将接受另一个写入器、一个随机数生成器和一个混淆的`chance`：

```go
func NewScrambleWriter(w io.Writer, r *rand.Rand, chance float64) *ScrambleWriter {
    return &ScrambleWriter{w: w, r: r, c: chance}
}

type ScrambleWriter struct {
    w io.Writer
    r *rand.Rand
    c float64
}
```

`Write`方法需要执行字节而不是字母，并打乱字母的顺序。它将迭代符文，使用我们之前看到的`ut8.DecodeRune`函数，打印出任何不是字母的内容，并堆叠它可以找到的所有字母序列：

```go
func (s *ScrambleWriter) Write(b []byte) (n int, err error) {
    var runes = make([]rune, 0, 10)
    for r, i, w := rune(0), 0, 0; i < len(b); i += w {
        r, w = utf8.DecodeRune(b[i:])
        if unicode.IsLetter(r) {
            runes = append(runes, r)
            continue
        }
        v, err := s.shambleWrite(runes, r)
        if err != nil {
            return n, err
        }
        n += v
        runes = runes[:0]
    }
    if len(runes) != 0 {
        v, err := s.shambleWrite(runes, 0)
        if err != nil {
            return n, err
        }
        n += v
    }
    return
}
```

当序列结束时，它将由`shambleWrite`方法处理，该方法将有效地执行一个混乱并写入混乱的符文：

```go
func (s *ScrambleWriter) shambleWrite(runes []rune, sep rune) (n int, err error) {
    //scramble after first letter
    for i := 1; i < len(runes)-1; i++ {
        if s.r.Float64() > s.c {
            continue
        }
        j := s.r.Intn(len(runes)-1) + 1
        runes[i], runes[j] = runes[j], runes[i]
    }
    if sep!= 0 {
        runes = append(runes, sep)
    }
    var b = make([]byte, 10)
    for _, r := range runes {
        v, err := s.w.Write(b[:utf8.EncodeRune(b, r)])
        if err != nil {
            return n, err
        }
        n += v
    }
    return
}
```

完整示例可在[`play.golang.org/p/0Xez--6P7nj`](https://play.golang.org/p/0Xez--6P7nj)中找到。

# 内置实用程序

`io`和`io/ioutil`包中有许多其他函数，可以帮助管理读取器、写入器等。了解所有可用的工具将帮助您避免编写不必要的代码，并指导您在使用最佳工具时进行操作。

# 从一个流复制到另一个流

`io`包中有三个主要函数，可以实现从写入器到读取器的数据传输。这是一个非常常见的场景；例如，您可以将从打开的文件中读取的内容写入到另一个打开的文件中，或者将缓冲区中的内容排空并将其内容写入标准输出。

我们已经看到如何在文件上使用`io.Copy`函数来模拟第四章*，与文件系统一起工作*中`cp`命令的行为。这种行为可以扩展到任何读取器和写入器的实现，从缓冲区到网络连接。

如果写入器也是`io.WriterTo`接口，复制将调用`WriteTo`方法。如果不是，它将使用固定大小的缓冲区（32 KB）进行一系列写入。如果操作以`io.EOF`值结束，则不会返回错误。一个常见的情况是`bytes.Buffer`结构，它能够将其内容写入另一个写入器，并且将相应地行事。或者，如果目标是`io.ReaderFrom`接口，则执行`ReadFrom`方法。

如果接口是一个简单的`io.Writer`接口，这个方法将使用一个临时缓冲区，之后将被清除。为了避免在垃圾回收上浪费计算资源，并且可能重用相同的缓冲区，还有另一个函数——`io.CopyBuffer`函数。这有一个额外的参数，只有在这个额外的参数是`nil`时才会分配一个新的缓冲区。

最后一个函数是`io.CopyN`，它的工作原理与`io.Copy`完全相同，但可以指定要写入到额外参数的字节数限制。如果读取器也是`io.Seeker`，则可以有用地写入部分内容——seeker 首先将光标移动到正确的偏移量，然后写入一定数量的字节。

让我们举一个一次复制`n`个字节的例子：

```go
func CopyNOffset(dst io.Writer, src io.ReadSeeker, offset, length int64) (int64, error) {
  if _, err := src.Seek(offset, io.SeekStart); err != nil {
    return 0, err
  }
  return io.CopyN(dst, src, length)
}
```

完整示例可在[`play.golang.org/p/8wCqGXp5mSZ`](https://play.golang.org/p/8wCqGXp5mSZ)中找到。

# 连接的读取器和写入器

`io.Pipe`函数创建一对连接的读取器和写入器。这意味着发送到写入器的任何内容都将从读取器接收到。如果仍有上次操作的挂起数据，写入操作将被阻塞；只有在读取器完成消耗已发送的内容后，新操作才会结束。

这对于非并发应用程序来说并不是一个重要的工具，非并发应用程序更有可能使用通道等并发工具，但是当读取器和写入器在不同的 goroutine 上执行时，这可以是一个很好的同步机制，就像下面的程序一样：

```go
    pr, pw := io.Pipe()
    go func(w io.WriteCloser) {
        for _, s := range []string{"a string", "another string", 
           "last one"} {
                fmt.Printf("-> writing %q\n", s)
                fmt.Fprint(w, s)
        }
        w.Close()
    }(pw)
    var err error
    for n, b := 0, make([]byte, 100); err == nil; {
        fmt.Println("<- waiting...")
        n, err = pr.Read(b)
        if err == nil {
            fmt.Printf("<- received %q\n", string(b[:n]))
        }
    }
    if err != nil && err != io.EOF {
        fmt.Println("error:", err)
    }
```

完整示例可在[`play.golang.org/p/0YpRK25wFw_c`](https://play.golang.org/p/0YpRK25wFw_c)中找到。

# 扩展读取器

当涉及到传入流时，标准库中有很多函数可用于改进读取器的功能。其中一个最简单的例子是`ioutil.NopCloser`，它接受一个读取器并返回`io.ReadCloser`，什么也不做。如果一个函数负责释放资源，但使用的读取器不是`io.Closer`（比如`bytes.Buffer`），这就很有用。

有两个工具可以限制读取的字节数。`ReadAtLeast`函数定义了要读取的最小字节数。只有在没有要读取的字节时才会返回`EOF`；否则，如果在`EOF`之前读取了较少的字节数，将返回`ErrUnexpectedEOF`。如果字节缓冲区比请求的字节数要短，这是没有意义的，将会返回`ErrShortBuffer`。在读取错误的情况下，函数会设法至少读取所需数量的字节，并且会丢弃该错误。

然后是`ReadFull`，它预期填充缓冲区，否则将返回`ErrUnexpectedEOF`。

另一个约束函数是`LimitReader`。这个函数是一个装饰器，它接收一个读取器并返回另一个读取器，一旦读取到所需的字节，就会返回`EOF`。这可以用于预览实际读取器的内容，就像下面的例子一样：

```go
s := strings.NewReader(`Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s, when an unknown printer took a galley of type and scrambled it to make a type specimen book. It has survived not only five centuries, but also the leap into electronic typesetting, remaining essentially unchanged.`)
    io.Copy(os.Stdout, io.LimitReader(s, 25)) // will print "Lorem Ipsum is simply dum"
```

完整的示例可在[`play.golang.org/p/LllOdWg9uyU`](https://play.golang.org/p/LllOdWg9uyU)找到。

更多的读取器可以使用`MultiReader`函数组合成一个序列，将依次读取每个部分，直到达到`EOF`，然后跳转到下一个。

一个读取器和一个写入器可以连接起来，以便来自读取器的任何内容都会被复制到写入器，这与`io.Pipe`的相反情况相反。这是通过`io.TeeReader`完成的。

让我们尝试使用它来创建一个在文件系统中充当搜索引擎的写入器，只打印出与所请求的查询匹配的行。我们想要一个执行以下操作的程序：

+   从参数中读取目录路径和要搜索的字符串

+   获取所选路径中的文件列表

+   读取每个文件，并将包含所选字符串的行传递给另一个写入器

+   另一个写入器将注入颜色字符以突出显示字符串，并将其内容复制到标准输出

让我们从颜色注入开始。在 Unix shell 中，可以通过以下序列获得彩色输出：

+   `\xbb1`: 一个转义字符

+   `[`: 一个开放的括号

+   `39`: 一个数字

+   `m`: 字母*m*

数字确定了背景和前景颜色。对于本例，我们将使用`31`（红色）和`39`（默认）。

我们正在创建一个写入器，它将打印出匹配的行并突出显示文本：

```go
type queryWriter struct {
    Query []byte
    io.Writer
}

func (q queryWriter) Write(b []byte) (n int, err error) {
    lines := bytes.Split(b, []byte{'\n'})
    l := len(q.Query)
    for _, b := range lines {
        i := bytes.Index(b, q.Query)
        if i == -1 {
            continue
        }
        for _, s := range [][]byte{
            b[:i], // what's before the match
            []byte("\x1b[31m"), //star red color
            b[i : i+l], // match
            []byte("\x1b[39m"), // default color
            b[i+l:], // whatever is left
        } {
            v, err := q.Writer.Write(s)
            n += v
            if err != nil {
                return 0, err
            }
        }
        fmt.Fprintln(q.Writer)
    }
    return len(b), nil
}
```

这将与打开文件一起使用`TeeReader`，以便读取文件将写入`queryWriter`：

```go
func main() {
    if len(os.Args) < 3 {
        fmt.Println("Please specify a path and a search string.")
        return
    }
    root, err := filepath.Abs(os.Args[1]) // get absolute path
    if err != nil {
        fmt.Println("Cannot get absolute path:", err)
        return
    }
    q := []byte(strings.Join(os.Args[2:], " "))
    fmt.Printf("Searching for %q in %s...\n", query, root)
    err = filepath.Walk(root, func(path string, info os.FileInfo,   
        err error) error {
            if info.IsDir() {
                return nil
            }
            fmt.Println(path)
            f, err := os.Open(path)
            if err != nil {
                return err
            }
        defer f.Close()

        _, err = ioutil.ReadAll(io.TeeReader(f, queryWriter{q, os.Stdout}))
        return err
    })
    if err != nil {
        fmt.Println(err)
    }
}
```

正如你所看到的，无需写入；从文件中读取会自动写入连接到标准输出的查询写入器。

# 写入器和装饰器

有大量的工具可用于增强、装饰和使用读取器，但对于写入器却不适用。

还有`io.WriteString`函数，它可以防止将字符串转换为字节。首先，它会检查写入器是否支持字符串写入，尝试将其转换为`io.stringWriter`，这是一个只有`WriteString`方法的未导出接口，然后如果成功，写入字符串，否则将其转换为字节。

有`io.MultiWriter`函数，它创建一个写入器，将信息复制到一系列其他写入器中，这些写入器在创建时接收。一个实际的例子是在将内容写入标准输出的同时显示它，就像下面的例子一样：

```go
    r := strings.NewReader("let's read this message\n")
    b := bytes.NewBuffer(nil)
    w := io.MultiWriter(b, os.Stdout)
    io.Copy(w, r) // prints to the standard output
    fmt.Println(b.String()) // buffer also contains string now
```

完整的示例可在[`play.golang.org/p/ZWDF2vCDfsM`](https://play.golang.org/p/ZWDF2vCDfsM)找到。

还有一个有用的变量，`ioutil.Discard`，它是一个写入器，写入到`/dev/null`，一个空设备。这意味着写入到这个变量会忽略数据。

# 总结

在本章中，我们介绍了流的概念，用于描述数据的传入和传出流。我们看到读取器接口表示接收到的数据，而写入器则是发送的数据。

我们比较了标准包中可用的不同读取器。在上一章中我们看了文件，在这一章中我们将字节和字符串读取器加入到列表中。我们学会了如何使用示例实现自定义读取器，并且看到设计一个读取器建立在另一个读取器之上总是一个好主意。

然后，我们专注于写入器。我们发现如果正确打开，文件也是写入器，并且标准包中有几个写入器，包括字节缓冲区和字符串构建器。我们还实现了一个自定义写入器，并看到如何使用`utf8`包处理字节和符文。

最后，我们探索了`io`和`ioutil`中剩余的功能，分析了用于复制数据和连接读取器和写入器的各种工具。我们还看到了用于改进或更改读取器和写入器功能的装饰器。

在下一章中，我们将讨论伪终端应用程序，并利用所有这些知识来构建其中一些。

# 问题

1.  什么是流？

1.  哪些接口抽象了传入流？

1.  哪些接口代表传出流？

1.  何时应该使用字节读取器？何时应该使用字符串读取器？

1.  字符串构建器和字节缓冲区之间有什么区别？

1.  读者和写入者的实现为什么要接受一个接口作为输入？

1.  管道与`TeeReader`有什么不同？
