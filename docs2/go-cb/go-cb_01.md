# I/O 和文件系统

在本章中，将涵盖以下食谱：

+   使用常见的 I/O 接口

+   使用 bytes 和 strings 包

+   使用目录和文件工作

+   使用 CSV 格式工作

+   使用临时文件工作

+   使用 text/template 和 HTML/templates 工作与文本

# 简介

Go 为基本和复杂的 I/O 提供了出色的支持。本章中的食谱将探索常见的 Go 接口以处理 I/O，并展示如何使用它们。Go 标准库经常使用这些接口，这些接口将在本书的食谱中广泛使用。

您将学习如何在内存和流的形式中处理数据。您将看到处理文件和目录的示例，以及处理 CSV 格式的示例。临时文件食谱讨论了一种无需处理名称冲突等开销即可处理文件的方法。最后，我们将探索 Go 标准模板，包括纯文本和 HTML。

这些食谱应该为使用接口表示和修改数据奠定基础，并帮助您以抽象和灵活的方式思考数据。

# 使用常见的 I/O 接口

Go 提供了标准库中使用的多个 I/O 接口。尽可能使用这些接口，而不是直接传递结构体或其他类型，这是一个最佳实践。在本食谱中，我们将探索两个强大的接口：`io.Reader` 和 `io.Writer` 接口。这些接口在标准库中广泛使用，了解如何使用它们将使您成为更好的 Go 开发者。

`Reader` 和 `Writer` 接口看起来像这样：

```go
type Reader interface {
        Read(p []byte) (n int, err error)
}

type Writer interface {
        Write(p []byte) (n int, err error)
}

```

Go 还使得组合接口变得容易。例如，看看以下代码：

```go
type Seeker interface {
        Seek(offset int64, whence int) (int64, error)
}

type ReadSeeker interface {
        Reader
        Seeker
}

```

食谱还将探索一个名为 `Pipe()` 的 `io` 函数：

```go
func Pipe() (*PipeReader, *PipeWriter)

```

本书剩余部分将使用这些接口。

# 准备工作

根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go，并在 [`golang.org/doc/install`](https://golang.org/doc/install) 配置 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序，导航到您的 `GOPATH/src` 目录，并创建一个项目目录，例如 `$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用以下命令安装最新测试版本的代码：

```go
 go get github.com/agtorre/go-cookbook/

```

# 如何做到...

这些步骤涵盖了编写和运行您的应用程序：

1.  在您的终端/控制台应用程序中，创建一个名为 `chapter1/interfaces` 的新目录。

1.  导航到该目录。

从 [`github.com/agtorre/go-cookbook/tree/master/chapter1/interfaces`](https://github.com/agtorre/go-cookbook/tree/master/chapter1/interfaces) 复制测试，或者将其作为练习编写一些您自己的代码。

1.  创建一个名为 `interfaces.go` 的文件，内容如下：

```go
        package interfaces

        import (
                "fmt"
                "io"
                "os"
        )

        // Copy copies data from in to out first directly,
        // then using a buffer. It also writes to stdout
        func Copy(in io.ReadSeeker, out io.Writer) error {
                // we write to out, but also Stdout
                w := io.MultiWriter(out, os.Stdout)

                // a standard copy, this can be dangerous if there's a 
                // lot of data in in
                if _, err := io.Copy(w, in); err != nil {
                    return err
                }

                in.Seek(0, 0)

                // buffered write using 64 byte chunks
                buf := make([]byte, 64)
                if _, err := io.CopyBuffer(w, in, buf); err != nil {
                    return err
                }

                // lets print a new line
                fmt.Println()

                return nil
        }

```

1.  创建一个名为 `pipes.go` 的文件，内容如下：

```go
        package interfaces

        import (
                "io"
                "os"
        )

        // PipeExample helps give some more examples of using io  
        //interfaces
        func PipeExample() error {
                // the pipe reader and pipe writer implement
                // io.Reader and io.Writer
                r, w := io.Pipe()

                // this needs to be run in a separate go routine
                // as it will block waiting for the reader
                // close at the end for cleanup
                go func() {
                    // for now we'll write something basic,
                    // this could also be used to encode json
                    // base64 encode, etc.
                    w.Write([]byte("testn"))
                    w.Close()
                }()

                if _, err := io.Copy(os.Stdout, r); err != nil {
                    return err
                }
                return nil
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个 `main.go` 文件，并包含以下内容，并确保你修改导入的接口以使用步骤 2 中设置的路径：

```go
        package main

        import (
             "bytes"
             "fmt"

             "github.com/agtorre/go-cookbook/chapter1/interfaces"
        )

        func main() {
                in := bytes.NewReader([]byte("example"))
                out := &bytes.Buffer{}
                fmt.Print("stdout on Copy = ")
                if err := interfaces.Copy(in, out); err != nil {
                        panic(err)
                }

                fmt.Println("out bytes buffer =", out.String())

                fmt.Print("stdout on PipeExample = ")
                if err := interfaces.PipeExample(); err != nil {
                        panic(err)
                }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行这些：

```go
 go build      ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 stdout on Copy = exampleexample
 out bytes buffer = exampleexample
 stdout on PipeExample = test

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`，并确保所有测试通过。

# 它是如何工作的...

`Copy()` 函数在接口之间进行复制，并将它们视为流。将数据视为流有许多实际用途，尤其是在处理网络流量或文件系统时。`Copy()` 函数还创建了一个多写入器，它结合了两个写入流，并使用 `ReadSeeker` 对它们进行两次写入。如果你使用 `Reader` 接口而不是看到 `exampleexample`，那么即使复制到 `MultiWriter` 接口两次，你也只会看到 `example`。还有一个缓冲写入的示例，如果你的流不适合内存，你可能需要使用它。

`PipeReader` 和 `PipeWriter` 结构体实现了 `io.Reader` 和 `io.Writer` 接口。它们是连接的，创建了一个内存管道。管道的主要目的是从流中读取，同时从同一流向不同的源写入。本质上，它将两个流合并成一个管道。

Go 接口是一个干净的抽象，用于封装执行常见操作的数据。这在进行 I/O 操作时变得明显，因此 `io` 包是学习接口组合的绝佳资源。`pipe` 包通常未被充分利用，但在连接输入和输出流时提供了极大的灵活性，并且具有线程安全性。

# 使用 `bytes` 和 `strings` 包

`bytes` 和 `string` 包提供了一些有用的辅助函数，用于处理字符串和字节类型之间的转换。它们允许创建与许多通用 I/O 接口一起工作的缓冲区。

# 准备工作

请参考 *准备工作* 部分的步骤，在 *使用通用 I/O 接口* 菜谱*中*。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建一个名为 `chapter1/bytestrings` 的新目录。

1.  导航到该目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter1/bytesstrings`](https://github.com/agtorre/go-cookbook/tree/master/chapter1/bytesstrings) 复制测试，或者将其用作练习来编写你自己的代码！

1.  创建一个名为 `buffer.go` 的文件，并包含以下内容：

```go
        package bytestrings

        import (
                "bytes"
                "io"
                "io/ioutil"
        )

        // Buffer demonstrates some tricks for initializing bytes    
        //Buffers
        // These buffers implement an io.Reader interface
        func Buffer(rawString string) *bytes.Buffer {

                // we'll start with a string encoded into raw bytes
                rawBytes := []byte(rawString)

                // there are a number of ways to create a buffer from 
                // the raw bytes or from the original string
                var b = new(bytes.Buffer)
                b.Write(rawBytes)

                // alternatively
                b = bytes.NewBuffer(rawBytes)

                // and avoiding the intial byte array altogether
                b = bytes.NewBufferString(rawString)

                return b
        }

        // ToString is an example of taking an io.Reader and consuming 
        // it all, then returning a string
        func toString(r io.Reader) (string, error) {
                b, err := ioutil.ReadAll(r)
                if err != nil {
                    return "", err
                }
                return string(b), nil
        }

```

1.  创建一个名为 `bytes.go` 的文件，并包含以下内容：

```go
        package bytestrings

        import (
                "bufio"
                "bytes"
                "fmt"
        )

        // WorkWithBuffer will make use of the buffer created by the
        // Buffer function
        func WorkWithBuffer() error {
                rawString := "it's easy to encode unicode into a byte 
                              array"

                b := Buffer(rawString)

                // we can quickly convert a buffer back into byes with
                // b.Bytes() or a string with b.String()
                fmt.Println(b.String())

                // because this is an io Reader we can make use of  
                // generic io reader functions such as
                s, err := toString(b)
                if err != nil {
                    return err
                }
                fmt.Println(s)

                // we can also take our bytes and create a bytes reader
                // these readers implement io.Reader, io.ReaderAt, 
                // io.WriterTo, io.Seeker, io.ByteScanner, and 
                // io.RuneScanner interfaces
                reader := bytes.NewReader([]byte(rawString))

                // we can also plug it into a scanner that allows 
                // buffered reading and tokenzation
                scanner := bufio.NewScanner(reader)
                scanner.Split(bufio.ScanWords)

                // iterate over all of the scan events
                for scanner.Scan() {
                    fmt.Print(scanner.Text())
                }

                return nil
        }

```

1.  创建一个名为 `string.go` 的文件，并包含以下内容：

```go
        package bytestrings

        import (
                "fmt"
                "io"
                "os"
                "strings"
        )

        // SearchString shows a number of methods
        // for searching a string
        func SearchString() {
                s := "this is a test"

                // returns true because s contains
                // the word this
                fmt.Println(strings.Contains(s, "this"))

                // returns true because s contains the letter a
                // would also match if it contained b or c
                fmt.Println(strings.ContainsAny(s, "abc"))

                // returns true because s starts with this
                fmt.Println(strings.HasPrefix(s, "this"))

                // returns true because s ends with this
                fmt.Println(strings.HasSuffix(s, "test"))
                }

        // ModifyString modifies a string in a number of ways
        func ModifyString() {
                s := "simple string"

                // prints [simple string]
                fmt.Println(strings.Split(s, " "))

                // prints "Simple String"
                fmt.Println(strings.Title(s))

                // prints "simple string"; all trailing and
                // leading white space is removed
                s = " simple string "
                fmt.Println(strings.TrimSpace(s))
        }

        // StringReader demonstrates how to create
        // an io.Reader interface quickly with a string
        func StringReader() {
                s := "simple stringn"
                r := strings.NewReader(s)

                // prints s on Stdout
                io.Copy(os.Stdout, r)
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个 `main.go` 文件，并包含以下内容，并确保你修改导入的接口以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter1/bytestrings"

        func main() {
                err := bytestrings.WorkWithBuffer()
                if err != nil {
                        panic(err)
                }

                // each of these print to stdout
                bytestrings.SearchString()
                bytestrings.ModifyString()
                bytestrings.StringReader() 
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行这些：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 it's easy to encode unicode into a byte array ??
 it's easy to encode unicode into a byte array ??
 it'seasytoencodeunicodeintoabytearray??true
 true
 true
 true
 [simple string]
 Simple String
 simple string
 simple string

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`，并确保所有测试通过。

# 它是如何工作的...

当与数据一起工作时，字节库提供了一些方便的函数。例如，一个缓冲区在处理流处理库或方法时比字节数组更灵活。一旦创建了缓冲区，它就可以用来满足 `io.Reader` 接口，这样您就可以利用 `ioutil` 函数来操作数据。对于流式应用程序，您可能希望使用缓冲区和扫描器。`bufio` 包在这些情况下非常有用。有时，使用数组或切片对于较小的数据集或当您的机器上有大量内存时更为合适。

Go 提供了在基本类型之间进行接口转换的很多灵活性--在字符串和字节之间进行转换相对简单。当与字符串一起工作时，`strings` 包提供了一些方便的函数来处理、搜索和操作字符串。在某些情况下，一个好的正则表达式可能是合适的，但大多数时候，`strings` 和 `strconv` 包就足够了。`strings` 包允许您使字符串看起来像标题，将其分割成数组，或删除空白。它还提供了一个自己的 `Reader` 接口，可以用作 `bytes` 包读取器类型的替代。

# 与目录和文件一起工作

在您在平台之间切换时（例如 Windows 和 Linux），与目录和文件一起工作可能会很困难。Go 提供了跨平台支持，在 `os` 和 `ioutils` 包中处理文件和目录。我们已经看到了 `ioutils` 的例子，但现在我们将探讨如何以另一种方式使用它们！

# 准备就绪

参考在 *准备就绪* 部分的步骤，在 *使用通用 I/O 接口* 菜谱中。

# 如何操作...

这些步骤涵盖了编写和运行您的应用程序：

1.  在您的终端/控制台应用程序中，创建一个名为 `chapter1/filedirs` 的新目录。

1.  导航到该目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter1/filedirs`](https://github.com/agtorre/go-cookbook/tree/master/chapter1/filedirs) 复制测试，或者将其作为练习编写一些自己的代码！

1.  创建一个名为 `dirs.go` 的文件，内容如下：

```go
        package filedirs

        import (
                "errors"
                "io"
                "os"
        )

        // Operate manipulates files and directories
        func Operate() error {
                // this 0777 is similar to what you'd see with chown
                // on a command line this will create a director 
                // /tmp/example, you may also use an absolute path 
                // instead of a relative one
                if err := os.Mkdir("example_dir", os.FileMode(0755)); 
                err !=  nil {
                        return err
                }

                // go to the /tmp directory
                if err := os.Chdir("example_dir"); err != nil {
                        return err
                }

                // f is a generic file object
                // it also implements multiple interfaces
                // and can be used as a reader or writer
                // if the correct bits are set when opening
                f, err := os.Create("test.txt")
                if err != nil {
                        return err
                }

                // we write a known-length value to the file and 
                // validate that it wrote correctly
                value := []byte("hellon")
                count, err := f.Write(value)
                if err != nil {
                        return err
                }
                if count != len(value) {
                        return errors.New("incorrect length returned 
                        from write")
                }

                if err := f.Close(); err != nil {
                        return err
                }

                // read the file
                f, err = os.Open("test.txt")
                if err != nil {
                        return err
                }

                io.Copy(os.Stdout, f)

                if err := f.Close(); err != nil {
                        return err
                }

                // go to the /tmp directory
                if err := os.Chdir(".."); err != nil {
                        return err
                }

                // cleanup, os.RemoveAll can be dangerous if you
                // point at the wrong directory, use user input,
                // and especially if you run as root
                if err := os.RemoveAll("example_dir"); err != nil {
                        return err
                }

                return nil
        }

```

1.  创建一个名为 `bytes.go` 的文件，内容如下：

```go
        package filedirs

        import (
                "bytes"
                "io"
                "os"
                "strings"
        )

        // Capitalizer opens a file, reads the contents,
        // then writes those contents to a second file
                func Capitalizer(f1 *os.File, f2 *os.File) error {
                if _, err := f1.Seek(0, 0); err != nil {
                        return err
                }

                var tmp = new(bytes.Buffer)

                if _, err := io.Copy(tmp, f1); err != nil {
                        return err
                }

                s := strings.ToUpper(tmp.String())

                if _, err := io.Copy(f2, strings.NewReader(s)); err != 
                nil {
                        return err
                }
                return nil
        }

        // CapitalizerExample creates two files, writes to one
        //then calls Capitalizer() on both
        func CapitalizerExample() error {
                f1, err := os.Create("file1.txt")
                if err != nil {
                        return err
                }

                if _, err := f1.Write([]byte(`this file contains a 
                number of words and new lines`)); err != nil {
                        return err
                }

                f2, err := os.Create("file2.txt")
                if err != nil {
                        return err
                }

                if err := Capitalizer(f1, f2); err != nil {
                        return err
                }

                if err := os.Remove("file1.txt"); err != nil {
                        return err
                }

                if err := os.Remove("file2.txt"); err != nil {
                        return err
                }

                return nil
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个包含以下内容的 `main.go` 文件，并确保您修改 `filedirs` 包导入以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter1/filedirs"

        func main() {
                if err := filedirs.Operate(); err != nil {
                        panic(err)
                }

                if err := filedirs.CapitalizerExample(); err != nil {
                        panic(err)
                }
        }

```

1.  运行 `go run main.go`。

1.  您还可以运行以下命令：

```go
 go build ./example

```

您应该看到以下输出：

```go
 $ go run main.go 
 hello

```

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行 `go test`，并确保所有测试都通过。

# 它是如何工作的...

如果你熟悉 Unix 中的文件，Go 的 `os` 库应该非常熟悉。你可以基本上执行所有常见的操作--获取文件属性、收集具有不同权限的文件、创建和修改目录和文件。我们执行了一系列对目录和文件的操纵，并在之后进行了清理。

与文件对象一起工作非常类似于内存流。文件还直接提供了一些便利函数，例如 `Chown`、`Stat` 和 `Truncate`。最简单的方法是利用文件。在所有之前的菜谱中，我们必须小心清理我们的程序。

在构建后端应用程序时，与文件一起工作是件非常常见的事情。文件可用于配置、密钥、临时存储等。Go 使用 `os` 包封装 OS 系统调用，并允许相同的函数在 Windows 或 Unix 上运行。

一旦你的文件被打开并存储在 `File` 结构体中，它就可以很容易地传递到之前讨论的许多接口中。所有之前处理缓冲区和内存数据流的示例都可以直接替换为文件对象。这可能对将所有日志同时写入 `stderr` 和文件的情况很有用。

# 使用 CSV 格式

CSV 是一种常见的格式来操作数据。例如，导入或导出 CSV 文件到 Excel 是很常见的。Go 的 `CSV` 包在数据接口上操作，因此将数据写入缓冲区、stdout、文件或套接字变得很容易。本节中的示例将展示一些将数据输入和输出到 CSV 格式的常见方法。

# 准备工作

参考在 *使用通用 I/O 接口* 菜单中的 *准备工作* 部分的步骤*。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建一个名为 `chapter1/csvformat` 的新目录。

1.  导航到该目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter1/csvformat`](https://github.com/agtorre/go-cookbook/tree/master/chapter1/csvformat) 复制测试，或者将其作为练习编写一些你自己的代码！

1.  创建一个名为 `read_csv.go` 的文件，并包含以下内容：

```go
        package csvformat

        import (
                "bytes"
                "encoding/csv"
                "fmt"
                "io"
                "strconv"
        )

        // Movie will hold our parsed CSV
        type Movie struct {
                Title string
                Director string
                Year int
        }

        // ReadCSV gives shows some examples of processing CSV
        // that is passed in as an io.Reader
        func ReadCSV(b io.Reader) ([]Movie, error) {

                r := csv.NewReader(b)

                // These are some optional configuration options
                r.Comma = ';'
                r.Comment = '-'

                var movies []Movie

                // grab and ignore the header for now
                // we may also wanna use this for a dictionary key or
                // some other form of lookup
                _, err := r.Read()
                if err != nil && err != io.EOF {
                        return nil, err
                }

                // loop until it's all processed
                for {
                        record, err := r.Read()
                        if err == io.EOF {
                                break
                        } else if err != nil {
                                return nil, err
                        }

                        year, err := strconv.ParseInt(record[2], 10, 
                        64)
                        if err != nil {
                                return nil, err
                        }

                        m := Movie{record[0], record[1], int(year)}
                        movies = append(movies, m)
                }
                return movies, nil
        }

        // AddMoviesFromText uses the CSV parser with a string
        func AddMoviesFromText() error {
                // this is an example of us taking a string, converting
                // it into a buffer, and reading it 
                // with the csv package
                in := `
                - first our headers
                movie title;director;year released

                - then some data
                Guardians of the Galaxy Vol. 2;James Gunn;2017
                Star Wars: Episode VIII;Rian Johnson;2017
                `

                b := bytes.NewBufferString(in)
                m, err := ReadCSV(b)
                if err != nil {
                        return err
                }
                fmt.Printf("%#vn", m)
                return nil
        }

```

1.  创建一个名为 `write_csv.go` 的文件，并包含以下内容：

```go
        package csvformat

        import (
                "bytes"
                "encoding/csv"
                "io"
                "os"
        )

        // A Book has an Author and Title
        type Book struct {
                Author string
                Title string
        }

        // Books is a named type for an array of books
        type Books []Book

        // ToCSV takes a set of Books and writes to an io.Writer
        // it returns any errors
        func (books *Books) ToCSV(w io.Writer) error {
                n := csv.NewWriter(w)
                err := n.Write([]string{"Author", "Title"})
                if err != nil {
                        return err
                }
                for _, book := range *books {
                        err := n.Write([]string{book.Author, 
                        book.Title})
                        if err != nil {
                                return err
                        }
                }

                n.Flush()
                return n.Error()
        }

        // WriteCSVOutput initializes a set of books
        // and writes the to os.Stdout
        func WriteCSVOutput() error {
                b := Books{
                        Book{
                                Author: "F Scott Fitzgerald",
                                Title: "The Great Gatsby",
                        },
                        Book{
                                Author: "J D Salinger",
                                Title: "The Catcher in the Rye",
                        },
                }

                return b.ToCSV(os.Stdout)
        }

        // WriteCSVBuffer returns a buffer csv for
        // a set of books
        func WriteCSVBuffer() (*bytes.Buffer, error) {
                b := Books{
                        Book{
                                Author: "F Scott Fitzgerald",
                                Title: "The Great Gatsby",
                        },
                        Book{
                                Author: "J D Salinger",
                                Title: "The Catcher in the Rye",
                        },
                }

                w := &bytes.Buffer{}
                err := b.ToCSV(w)
                return w, err
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并确保你将 `csvformat` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
                "fmt"

                "github.com/agtorre/go-cookbook/chapter1/csvformat"
        )

        func main() {
                if err := csvformat.AddMoviesFromText(); err != nil {
                        panic(err)
                }

                if err := csvformat.WriteCSVOutput(); err != nil {
                        panic(err)
                }

                buffer, err := csvformat.WriteCSVBuffer()
                if err != nil {
                        panic(err)
                }

                fmt.Println("Buffer = ", buffer.String())
        }

```

1.  运行 `go run main.go`。

1.  你还可以运行以下命令：

```go
 go build
 ./example

```

你应该看到以下输出：

```go
 $ go run main.go 
 []csvformat.Movie{csvformat.Movie{Title:"Guardians of the 
        Galaxy Vol. 2", Director:"James Gunn", Year:2017},         
        csvformat.Movie{Title:"Star Wars: Episode VIII", Director:"Rian 
        Johnson", Year:2017}}
 Author,Title
 F Scott Fitzgerald,The Great Gatsby
 J D Salinger,The Catcher in the Rye
 Buffer = Author,Title
 F Scott Fitzgerald,The Great Gatsby
 J D Salinger,The Catcher in the Rye

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`，并确保所有测试都通过。

# 工作原理...

为了探索读取 CSV 格式，我们首先将我们的数据表示为一个结构体。在 Go 中，将数据格式化为结构体非常有用，因为它使得诸如序列化和编码等操作相对简单。我们的读取示例使用电影作为数据类型。该函数接受一个 `io.Reader` 接口作为输入，该接口包含我们的 CSV 数据。这可能是一个文件或一个缓冲区。然后我们使用这些数据来创建和填充一个 `Movie` 结构体，包括将年份转换为整数。我们还向 CSV 解析器添加了选项，使用 `;` 作为分隔符，`-` 作为注释行。

接下来，我们探索相同的概念，但方向相反。小说用标题和作者来表示。我们初始化一个小说数组，然后将特定的小说以 CSV 格式写入一个 `io.Writer` 接口。同样，这可以是一个文件、stdout 或一个缓冲区。

`CSV` 包是为什么你想要在 Go 中将数据流视为实现常见接口的绝佳例子。通过简单的单行调整，我们可以轻松更改数据源和目标，并且可以轻松地操作 CSV 数据，而无需使用过多的内存或时间。例如，可以一次从数据流中读取一条记录，并一次以修改后的格式写入另一个单独的流中。这样做不会产生显著的内存或处理器使用。

在以后探索数据管道和工作者池时，你会看到这些想法如何结合，以及如何并行处理这些流。

# 使用临时文件

我们已经创建并使用文件处理了多个示例。我们也必须手动处理清理、名称冲突等问题。临时文件和目录是处理这些情况更快、更简单的方法。

# 准备就绪

参考在 *使用通用 I/O 接口* 菜单中的 *准备就绪* 部分的步骤*。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建一个名为 `chapter1/tempfiles` 的新目录。

1.  导航到这个目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter1/tempfiles`](https://github.com/agtorre/go-cookbook/tree/master/chapter1/tempfiles) 复制测试，或者使用这个练习来编写你自己的代码！

1.  创建一个名为 `temp_files.go` 的文件，内容如下：

```go
        package tempfiles

        import (
                "fmt"
                "io/ioutil"
                "os"
        )

        // WorkWithTemp will give some basic patterns for working
        // with temporary files and directories
        func WorkWithTemp() error {
                // If you need a temporary place to store files with 
                // the same name ie. template1-10.html a temp directory 
                //  is a good way to approach it, the first argument 
                // being blank means it will use create the directory                
                // in the location returned by 
                // os.TempDir()
                t, err := ioutil.TempDir("", "tmp")
                if err != nil {
                        return err
                }

                // This will delete everything inside the temp file 
                // when this function exits if you want to do this 
                //  later, be sure to return the directory name to the 
                // calling function
                defer os.RemoveAll(t)

                // the directory must exist to create the tempfile
                // created. t is an *os.File object.
                tf, err := ioutil.TempFile(t, "tmp")
                if err != nil {
                        return err
                }

                fmt.Println(tf.Name())

                // normally we'd delete the temporary file here, but 
                // because we're placing it in a temp directory, it 
                // gets cleaned up by the earlier defer

                return nil
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个 `main.go` 文件，内容如下，并确保你修改导入的 tempfiles 以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter1/tempfiles"

        func main() {
                if err := tempfiles.WorkWithTemp(); err != nil {
                        panic(err)
                }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到（使用不同的路径）以下输出：

```go
 $ go run main.go 
 /var/folders/kd/ygq5l_0d1xq1lzk_c7htft900000gn/T
        /tmp764135258/tmp588787953

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`，并确保所有测试通过。

# 它是如何工作的...

可以使用 `ioutil` 包创建和利用临时文件和目录。虽然你仍然必须自己删除文件，但 `RemoveAll` 是一种约定，它只需一行额外的代码就会为你完成这项工作。

在编写测试时，强烈建议使用临时文件。对于构建工件等其他事情也很有用。Go 的 `ioutil` 包默认会尝试尊重操作系统偏好，但如果需要，它允许你回退到其他目录。

# 使用 text/template 和 HTML/templates 进行工作

Go 为模板提供了丰富的支持。嵌套模板、导入函数、表示变量、遍历数据等操作都非常简单。如果你需要比 CSV 写入器更复杂的功能，模板可能是一个很好的解决方案。

模板的另一个应用是用于网站。当我们想要将服务器端数据渲染到客户端时，模板非常适合。最初，Go 模板可能看起来有些令人困惑。本章将探讨与模板一起工作、收集目录中的模板以及处理 HTML 模板。

# 准备就绪

参考在 *使用常见 I/O 接口* 菜单中的 *准备就绪* 部分的步骤*。

# 如何实现...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建一个名为 `chapter1/templates` 的新目录。

1.  导航到这个目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter1/templates`](https://github.com/agtorre/go-cookbook/tree/master/chapter1/templates) 复制测试，或者将其作为练习编写一些自己的测试！

1.  创建一个名为 `templates.go` 的文件，内容如下：

```go
        package templates

        import (
                "os"
                "strings"
                "text/template"
        )

        const sampleTemplate = `
                This template demonstrates printing a {{ .Variable | 
                printf "%#v" }}.

                {{if .Condition}}
                If condition is set, we'll print this
                {{else}}
                Otherwise, we'll print this instead
                {{end}}

                Next we'll iterate over an array of strings:
                {{range $index, $item := .Items}}
                {{$index}}: {{$item}}
                {{end}}

                We can also easily import other functions like 
                strings.Split
                then immediately used the array created as a result:
                {{ range $index, $item := split .Words ","}}
                {{$index}}: {{$item}}
                {{end}}

                Blocks are a way to embed templates into one another
                {{ block "block_example" .}}
                No Block defined!
                {{end}}

                {{/*
                This is a way
                to insert a multi-line comment
                */}}
`

        const secondTemplate = `
                {{ define "block_example" }}
                {{.OtherVariable}}
                {{end}}
`        

        // RunTemplate initializes a template and demonstrates a 
        // variety of template helper functions
        func RunTemplate() error {
                data := struct {
                        Condition bool
                        Variable string
                        Items []string
                        Words string
                        OtherVariable string
                }{
                        Condition: true,
                        Variable: "variable",
                        Items: []string{"item1", "item2", "item3"},
                        Words: 
                        "another_item1,another_item2,another_item3",
                        OtherVariable: "I'm defined in a second 
                        template!",
                }

                funcmap := template.FuncMap{
                        "split": strings.Split,
                }

                // these can also be chained
                t := template.New("example")
                t = t.Funcs(funcmap)

                // We could use Must instead to panic on error
                // template.Must(t.Parse(sampleTemplate))
                t, err := t.Parse(sampleTemplate)
                if err != nil {
                        return err
                }

                // to demonstrate blocks we'll create another template
                // by cloning the first template, then parsing a second
                t2, err := t.Clone()
                if err != nil {
                        return err
                }

                t2, err = t2.Parse(secondTemplate)
                if err != nil {
                        return err
                }

                // write the template to stdout and populate it
                // with data
                err = t2.Execute(os.Stdout, &data)
                if err != nil {
                        return err
                }

                return nil
        }

```

1.  创建一个名为 `template_files.go` 的文件，内容如下：

```go
        package templates

        import (
                "io/ioutil"
                "os"
                "path/filepath"
                "text/template"
        )

        //CreateTemplate will create a template file that contains data
        func CreateTemplate(path string, data string) error {
                return ioutil.WriteFile(path, []byte(data), 
                os.FileMode(0755))
        }

        // InitTemplates sets up templates from a directory
        func InitTemplates() error {
                tempdir, err := ioutil.TempDir("", "temp")
                if err != nil {
                        return err
                }
                defer os.RemoveAll(tempdir)

                err = CreateTemplate(filepath.Join(tempdir, "t1.tmpl"), 
                `Template 1! {{ .Var1 }}
                {{ block "template2" .}} {{end}}
                {{ block "template3" .}} {{end}}
                `)
                if err != nil {
                        return err
                }

                err = CreateTemplate(filepath.Join(tempdir, "t2.tmpl"), 
                `{{ define "template2"}}Template 2! {{ .Var2 }}{{end}}
                `)
                if err != nil {
                        return err
                }

                err = CreateTemplate(filepath.Join(tempdir, "t3.tmpl"), 
                `{{ define "template3"}}Template 3! {{ .Var3 }}{{end}}
                `)
                if err != nil {
                        return err
                }

                pattern := filepath.Join(tempdir, "*.tmpl")

                // Parse glob will combine all the files that match 
                // glob and combine them into a single template
                tmpl, err := template.ParseGlob(pattern)
                if err != nil {
                        return err
                }

                // Execute can also work with a map instead
                // of a struct
                tmpl.Execute(os.Stdout, map[string]string{
                        "Var1": "Var1!!",
                        "Var2": "Var2!!",
                        "Var3": "Var3!!",
                 })

                 return nil
        }

```

1.  创建一个名为 `html_templates.go` 的文件，内容如下：

```go
        package templates

        import (
                "fmt"
                "html/template"
                "os"
        )

        // HTMLDifferences highlights some of the differences
        // between html/template and text/template
        func HTMLDifferences() error {
                t := template.New("html")
                t, err := t.Parse("<h1>Hello! {{.Name}}</h1>n")
                if err != nil {
                        return err
         }

                // html/template auto-escapes unsafe operations like 
                // javascript injection this is contextually aware and 
                // will behave differently
                // depending on where a variable is rendered
                err = t.Execute(os.Stdout, map[string]string{"Name": "                 <script>alert('Can you see me?')</script>"})
                if err != nil {
                        return err
                }

                // you can also manually call the escapers
                fmt.Println(template.JSEscaper(`example         
                <example@example.com>`))
                fmt.Println(template.HTMLEscaper(`example 
                <example@example.com>`))
                fmt.Println(template.URLQueryEscaper(`example 
                <example@example.com>`))

                return nil
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并确保你修改了导入的临时文件以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter1/templates"

        func main() {
                if err := templates.RunTemplate(); err != nil {
                        panic(err)
                }

                if err := templates.InitTemplates(); err != nil {
                        panic(err)
                }

                if err := templates.HTMLDifferences(); err != nil {
                        panic(err)
                }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行这些：

```go
 go build ./example

```

你应该看到以下输出（路径可能不同）：

```go
 $ go run main.go 

 This template demonstrates printing a "variable".

 If condition is set, we'll print this

 Next we'll iterate over an array of strings:

 0: item1

 1: item2

 2: item3

 We can also easily import other functions like strings.Split
 then immediately used the array created as a result:

 0: another_item1

 1: another_item2

 2: another_item3

 Blocks are a way to embed templates into one another

 I'm defined in a second template!

 Template 1! Var1!!
 Template 2! Var2!!
 Template 3! Var3!!
 <h1>Hello! &lt;script&gt;alert('Can you see 
         me?')&lt;/script&gt;</h1>
 example x3Cexample@example.comx3E
 example &lt;example@example.com&gt;
 example+%3Cexample%40example.com%3E

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`，并确保所有测试都通过。

# 工作原理...

Go 有两个模板包--`text/template` 和 `html/template`。这两个包共享功能并提供了各种函数。通常，使用 `html/template` 来渲染网站，对于其他所有内容则使用 text/html。模板是纯文本，但可以在花括号块中使用变量和函数。

模板包还提供了方便的方法来处理文件。示例在临时目录中创建了一些模板，然后使用一行代码读取它们。

`html/template` 包是 `text/template` 包的包装器。所有的模板示例都直接使用 `html/template` 包，无需修改，只需更改导入语句。HTML 模板提供了额外的上下文感知安全性。这防止了诸如 JavaScript 注入等问题。

模板包提供了您期望的现代模板库所具备的功能。组合模板、添加应用程序逻辑以及确保将结果输出到 HTML 和 JavaScript 时的安全性都很简单。
