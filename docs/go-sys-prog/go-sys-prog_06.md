# 第六章：文件输入和输出

在上一章中，我们谈到了将文件和目录作为实体进行操作，而不查看其内容。但是，在本章中，我们将采取不同的方法，查看文件的内容：您可能认为本章是本书中最重要的章节之一，因为**文件输入**和**文件输出**是任何操作系统的主要任务。

本章的主要目的是教授 Go 标准库如何允许我们打开文件，读取其内容，如果需要，对其进行处理，创建新文件，并将所需数据放入其中。读取和写入文件的两种主要方法是：使用`io`包和使用`bufio`包的函数。但是，这两个包的工作方式是相似的。

本章将告诉您以下内容：

+   打开文件进行写入和读取

+   使用`io`包进行文件输入和输出

+   使用`io.Writer`和`io.Reader`接口

+   使用`bufio`包进行缓冲输入和输出

+   在 Go 中复制文件

+   在 Go 中实现`wc(1)`实用程序的版本

+   在 Go 中开发`dd(1)`命令的版本

+   创建稀疏文件

+   字节切片在文件输入和输出中的重要性：字节切片首次在第二章中提到，*使用 Go 编写程序*

+   将结构化数据存储在文件中，并在以后读取它们

+   将制表符转换为空格字符，反之亦然

本章不会讨论向现有文件追加数据：您将不得不等到第七章，*使用系统文件*，以了解如何在不破坏现有数据的情况下将数据放在文件末尾。

# 关于文件输入和输出

文件输入和输出包括与读取文件数据和将所需数据写入文件有关的一切。没有一个操作系统不支持文件，因此也不支持文件输入和输出。

由于本章内容较多，我将停止讲话，开始向您展示将使事情更清晰的实际 Go 代码。因此，您将在本章中学到的第一件事是字节切片，在涉及文件输入和输出的应用程序中非常重要。

# 字节切片

字节切片是一种用于文件读写的切片。简单来说，它们是用作文件读写操作期间的缓冲区的字节切片。本节将介绍一个小的 Go 示例，其中使用字节切片进行文件写入和读取。正如您将在本章中看到的字节切片一样，请确保您理解所呈现的示例。相关的 Go 代码保存为`byteSlice.go`，将分为三个部分。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "io/ioutil" 
   "os" 
) 
```

`byteSlice.go`的第二部分如下：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Please provide a filename") 
         os.Exit(1) 
   } 
   filename := os.Args[1] 

   aByteSlice := []byte("Mihalis Tsoukalos!\n") 
   ioutil.WriteFile(filename, aByteSlice, 0644) 
```

在这里，您使用`aByteSlice`字节切片将一些文本保存到由`filename`变量标识的文件中。`byteSlice.go`的最后一部分是以下 Go 代码：

```go
   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
   defer f.Close() 

   anotherByteSlice := make([]byte, 100) 
   n, err := f.Read(anotherByteSlice) 
   fmt.Printf("Read %d bytes: %s", n, anotherByteSlice)

} 
```

在这里，您定义了另一个名为`anotherByteSlice`的字节切片，其中有`100`个位置，将用于从先前创建的文件中读取。请注意，`fmt.Printf()`中使用的`%s`强制`anotherByteSlice`作为字符串打印：使用`Println()`将产生完全不同的输出。

请注意，由于文件较小，`f.Read()`调用将向`anotherByteSlice`中放入较少的数据。

`anotherByteSlice`的大小表示在单次调用`Read()`或任何其他类似从文件读取数据的操作之后可以存储在其中的最大数据量。

执行`byteSlice.go`将生成以下输出：

```go
$ go run byteSlice.go usingByteSlices
Read 19 bytes: Mihalis Tsoukalos!
```

检查`usingByteSlices`文件的大小将验证是否已将正确的数据量写入其中：

```go
$ wc usingByteSlices
   1   2  19 usingByteSlices
```

# 关于二进制文件

在 Go 中，读取和写入二进制和纯文本文件没有区别。因此，在处理文件时，Go 不会对其格式做出任何假设。但是，Go 提供了一个名为 binary 的包，允许您在不同的编码之间进行转换，例如**小端**和**大端**。

`readBinary.go`文件简要说明了如何将整数转换为小端数和大端数，当您要处理的文件包含某些类型的数据时可能会有用；这主要发生在处理原始设备和原始数据包操作时：记住一切都是文件！`readBinary.go`的源代码将分两部分呈现。

第一部分如下：

```go
package main 

import ( 
   "bytes" 
   "encoding/binary" 
   "fmt" 
   "os" 
   "strconv" 
) 

func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Please provide an integer") 
         os.Exit(1) 
   } 
   aNumber, _ := strconv.ParseInt(os.Args[1], 10, 64) 
```

程序的这一部分没有什么特别之处。第二部分如下：

```go
   buf := new(bytes.Buffer) 
   err := binary.Write(buf, binary.LittleEndian, aNumber) 
   if err != nil { 
         fmt.Println("Little Endian:", err) 
   } 

   fmt.Printf("%d is %x in Little Endian\n", aNumber, buf) 
   buf.Reset() 
   err = binary.Write(buf, binary.BigEndian, aNumber)

   if err != nil { 
         fmt.Println("Big Endian:", err) 
   } 
   fmt.Printf("And %x in Big Endian\n", buf) 
} 
```

第二部分包含了所有重要的 Go 代码：转换是通过`binary.Write()`方法和适当的写入参数（`binary.LittleEndian`或`binary.BigEndian`）进行的。`bytes.Buffer`变量用于程序的`io.Reader`和`io.Writer`接口。最后，`buf.Reset()`语句重置缓冲区，以便之后用于存储大端数。

执行`readBinary.go`将生成以下输出：

```go
$ go run readBinary.go 1
1 is 0100000000000000 in Little Endian
And 0000000000000001 in Big Endian
```

您可以通过访问其文档页面[`golang.org/pkg/encoding/binary/`](https://golang.org/pkg/encoding/binary/)找到有关二进制包的更多信息。

# Go 中有用的 I/O 包

`io`包用于执行原始文件 I/O 操作，而`bufio`包用于执行缓冲 I/O。

在缓冲 I/O 中，操作系统在文件读写操作期间使用中间缓冲区，以减少文件系统调用的次数。因此，缓冲输入和输出更快更高效。

此外，您可以使用`fmt`包的一些函数将文本写入文件。请注意，`flag`包也将在本章以及所有需要支持命令行标志的后续章节中使用。

# io 包

`io`包提供了允许您向文件写入或从文件读取的函数。其用法将在`usingIO.go`文件中进行演示，该文件将分三部分呈现。程序的作用是从文件中读取`8`个字节并将它们写入标准输出。

Go 程序的序言是第一部分：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "os" 
) 
```

第二部分是以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Please provide a filename") 
         os.Exit(1) 
   } 

   filename := os.Args[1] 
   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Printf("error opening %s: %s", filename, err) 
         os.Exit(1) 
   } 
   defer f.Close() 
```

该程序还使用了方便的`defer`命令，它推迟了函数的执行，直到周围的函数返回。因此，在文件 I/O 操作中经常使用`defer`，因为它可以让您不必记住在完成文件处理或在使用`return`语句或`os.Exit()`离开函数时执行`Close()`调用。

程序的最后一部分如下：

```go
   buf := make([]byte, 8) 
   if _, err := io.ReadFull(f, buf); err != nil { 
         if err == io.EOF { 
               err = io.ErrUnexpectedEOF 
         } 
   } 
   io.WriteString(os.Stdout, string(buf)) 
   fmt.Println() 
} 
```

这里的`io.ReadFull()`函数从打开文件的读取器中读取数据，并将数据放入一个有 8 个位置的字节切片中。您还可以在这里看到使用`io.WriteString()`函数将数据打印到标准输出（`os.Stdout`），这也是一个文件。但是，这不是一个很常见的做法，因为您可以简单地使用`fmt.Println()`。

执行`usingIO.go`会生成以下输出：

```go
$ go run usingIO.go usingByteSlices
Mihalis
```

# bufio 包

`bufio`包的函数允许您执行缓冲文件操作，这意味着尽管其操作看起来类似于`io`中找到的操作，但它们的工作方式略有不同。

`bufio`实际上的作用是将`io.Reader`或`io.Writer`对象包装成一个实现所需接口的新值，并为新值提供缓冲。`bufio`包的一个方便功能是它允许您轻松地逐行、逐词和逐字符读取文本文件。

再次，一个示例将尝试澄清事情：展示了`bufio`使用的 Go 文件的名称是`bufIO.go`，将分为四个部分呈现。

第一部分是预期的序文：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "os" 
) 
```

第二部分是以下内容：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Please provide a filename") 
         os.Exit(1) 
   } 

   filename := os.Args[1] 
```

在这里，你只需尝试获取要使用的文件的名称。

`bufIO.go`的第三部分包含以下 Go 代码：

```go
   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Printf("error opening %s: %s", filename, err) 
         os.Exit(1) 
   } 
   defer f.Close() 

   scanner := bufio.NewScanner(f) 
```

`bufio.NewScanner`的默认行为是逐行读取其输入，这意味着每次调用`Scan()`方法读取下一个标记时，都会返回一个新行。最后一部分是你实际调用`Scan()`方法以读取文件的全部内容的地方：

```go
   for scanner.Scan() { 
         line := scanner.Text() 

         if scanner.Err() != nil { 
               fmt.Printf("error reading file %s", err) 
               os.Exit(1) 
         } 
         fmt.Println(line) 
   } 
}

```

`Text()`方法将`Scan()`方法的最新标记作为字符串返回，这种情况下将是一行。但是，如果你在尝试逐行读取文件时遇到奇怪的结果，那很可能是你的文件如何结束一行的方式，这通常是来自 Windows 机器的文本文件的情况。

执行`bufIO.go`并将其输出提供给`wc(1)`可以帮助你验证`bufIO.go`是否按预期工作：

```go
$ go run bufIO.go inputFile | wc
      11      12      62
$ wc inputFile
      11      12      62 inputFile
```

# 文件 I/O 操作

现在你已经了解了`io`和`bufio`包的基础知识，是时候学习更详细的关于它们的用法以及它们如何帮助你处理文件的信息了。但首先，我们将讨论`fmt.Fprintf()`函数。

# 使用 fmt.Fprintf()向文件写入

使用`fmt.Fprintf()`函数允许你以类似于`fmt.Printf()`函数的方式向文件写入格式化文本。请注意，`fmt.Fprintf()`可以写入任何`io.Writer`接口，并且我们的文件将满足`io.Writer`接口。

展示了`fmt.Fprintf()`的 Go 代码可以在`fmtF.go`中找到，该文件将分为三个部分呈现。第一部分是预期的序文：

```go
package main 

import ( 
   "fmt" 
   "os" 
) 
```

第二部分包含以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Please provide a filename") 
         os.Exit(1) 
   } 

   filename := os.Args[1] 
   destination, err := os.Create(filename) 
   if err != nil { 
         fmt.Println("os.Create:", err) 
         os.Exit(1) 
   } 
   defer destination.Close() 
```

请注意，如果文件已经存在，`os.Create()`函数将截断该文件。

最后一部分是以下内容：

```go
   fmt.Fprintf(destination, "[%s]: ", filename) 
   fmt.Fprintf(destination, "Using fmt.Fprintf in %s\n", filename) 
} 
```

在这里，你可以使用`fmt.Fprintf()`将所需的文本数据写入由目标变量标识的文件，就像你使用`fmt.Printf()`方法一样。

执行`fmtF.go`将生成以下输出：

```go
$ go run fmtF.go test
$ cat test
[test]: Using fmt.Fprintf in test 
```

换句话说，你可以使用`fmt.Fprintf()`创建纯文本文件。

# 关于 io.Writer 和 io.Reader

`io.Writer`和`io.Reader`都是嵌入`io.Write()`和`io.Read()`方法的接口。`io.Writer`和`io.Reader`的使用将在`readerWriter.go`中进行说明，该文件将分为四个部分呈现。该程序计算其输入文件的字符数，并将字符数写入另一个文件：如果你处理的是每个字符占用多个字节的 Unicode 字符，你可能会考虑该程序正在读取字节。输出文件名为原始文件名加上`.Count`扩展名。

第一部分是以下内容：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "os" 
) 
```

第二部分是以下内容：

```go
func countChars(r io.Reader) int { 
   buf := make([]byte, 16) 
   total := 0 
   for { 
         n, err := r.Read(buf) 
         if err != nil && err != io.EOF { 
               return 0 
         } 
         if err == io.EOF { 
               break 
         } 
         total = total + n 
   } 
   return total 
} 
```

再次，在读取过程中使用了字节切片。`break`语句允许你退出`for`循环。第三部分是以下代码：

```go
func writeNumberOfChars(w io.Writer, x int) { 
   fmt.Fprintf(w, "%d\n", x) 
} 
```

在这里你可以看到如何使用`fmt.Fprintf()`向文件写入数字：我没有成功使用字节切片做同样的事情！另外，请注意，所呈现的代码使用`io.Writer`变量(`w`)向文件写入文本。

`readerWriter.go`的最后一部分包含以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Please provide a filename") 
         os.Exit(1) 
   } 

   filename := os.Args[1] 
   _, err := os.Stat(filename)

   if err != nil { 
         fmt.Printf("Error on file %s: %s\n", filename, err) 
         os.Exit(1) 
   } 

   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Println("Cannot open file:", err) 
         os.Exit(-1) 
   } 
   defer f.Close() 

   chars := countChars(f) 
   filename = filename + ".Count" 
   f, err = os.Create(filename) 
   if err != nil { 
         fmt.Println("os.Create:", err) 
         os.Exit(1) 
   } 
   defer f.Close() 
   writeNumberOfChars(f, chars) 
} 
```

执行`readerWriter.go`不会生成任何输出；因此，你需要检查其正确性，这在本例中是通过`wc(1)`的帮助来实现的：

```go
$ go run readerWriter.go /tmp/swtag.log
$ wc /tmp/swtag.log
     119     635    7780 /tmp/swtag.log
$ cat /tmp/swtag.log.Count
7780
```

# 查找行的第三列

现在您已经知道如何读取文件，是时候介绍您在第三章中看到的`readColumn.go`程序的修改版本了，*高级 Go 功能*。新版本也被命名为`readColumn.go`，但有两个主要改进。第一个是您可以将所需的列作为命令行参数提供，第二个是如果它获得多个命令行参数，它可以读取多个文件。

`readColumn.go`文件将分为三部分。`readColumn.go`的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "flag" 
   "fmt" 
   "io" 
   "os" 
   "strings" 
) 
```

`readColumn.go`的下一部分包含以下 Go 代码：

```go
func main() { 
   minusCOL := flag.Int("COL", 1, "Column") 
   flag.Parse() 
   flags := flag.Args() 

   if len(flags) == 0 { 
         fmt.Printf("usage: readColumn <file1> [<file2> [... <fileN]]\n") 
         os.Exit(1) 
   } 

   column := *minusCOL 

   if column < 0 { 
         fmt.Println("Invalid Column number!") 
         os.Exit(1) 
   } 
```

从`minusCOL`变量的定义中，您将了解到，如果用户不使用此标志，程序将打印它读取的每个文件的第一列的内容。

`readColumn.go`的最后部分如下：

```go
   for _, filename := range flags { 
         fmt.Println("\t\t", filename) 
         f, err := os.Open(filename) 
         if err != nil { 
               fmt.Printf("error opening file %s", err) 
               continue 
         } 
         defer f.Close() 

         r := bufio.NewReader(f)

         for { 
               line, err := r.ReadString('\n') 

               if err == io.EOF { 
                     break 
               } else if err != nil { 
                     fmt.Printf("error reading file %s", err) 
               } 

               data := strings.Fields(line) 
               if len(data) >= column { 
                     fmt.Println((data[column-1])) 
               } 
         } 
   } 
} 
```

前面的代码没有做任何您以前没有见过的事情。`for`循环用于处理所有命令行参数。但是，如果由于某种原因文件无法打开，程序将不会停止执行，而是会继续处理其余的文件（如果存在）。但是，程序期望其输入文件以换行符结尾，如果输入文件以不同的方式结束，您可能会看到奇怪的结果。

执行`readColumn.go`会生成以下输出，为了节省一些书籍空间，输出进行了缩写：

```go
$ go run readColumn.go -COL=3 pF.data isThereAFile up.data
            pF.data
            isThereAFile
error opening file open isThereAFile: no such file or directory
            up.data
0.05
0.05
0.05
0.05
0.05
0.05
```

在这种情况下，没有名为`isThereAFile`的文件，`pF.data`文件也没有第三列。但是，程序尽力打印了它能够打印的内容！

# 在 Go 中复制文件

每个操作系统都允许您复制文件，因为这是非常重要和必要的操作。现在您知道如何读取文件，本节将向您展示如何在 Go 中复制文件！

# 复制文件有多种方法！

大多数编程语言提供了多种创建文件副本的方法，Go 也不例外。由开发人员决定要实现哪种方法。

*有多种方法可以做到*这一规则几乎适用于本书中实现的所有内容，但是文件复制是这一规则的最典型的例子，因为您可以逐行、逐字节或一次性复制文件！但是，这一规则不适用于 Go 喜欢格式化其代码的方式！

# 复制文本文件

处理文本文件的复制没有特殊的意义，除非你想要检查或修改它们的内容。因此，这里介绍的三种技术不会区分纯文本和二进制文件的复制。

第七章*，处理系统文件*，将讨论文件权限，因为有时您希望使用您选择的文件权限创建新文件。

# 使用 io.Copy

本小节将介绍一种使用`io.Copy()`函数复制文件的技术。`io.Copy()`函数的特殊之处在于它在过程中不提供任何灵活性。程序的名称将是`notGoodCP.go`，将分为三部分呈现。请注意，`notGoodCP.go`的更合适的文件名可能是`copyEntireFileAtOnce.go`或`copyByReadingInputFileAllAtOnce.go`！

`notGoodCP.go`的 Go 代码的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "os" 
) 
```

第二部分如下：

```go
func Copy(src, dst string) (int64, error) { 
   sourceFileStat, err := os.Stat(src) 
   if err != nil { 
         return 0, err 
   } 

   if !sourceFileStat.Mode().IsRegular() { 
         return 0, fmt.Errorf("%s is not a regular file", src) 
   } 

   source, err := os.Open(src) 
   if err != nil { 
         return 0, err 
   } 
   defer source.Close() 

   destination, err := os.Create(dst) 
   if err != nil { 
         return 0, err 
   } 
   defer destination.Close() 
   nBytes, err := io.Copy(destination, source) 
   return nBytes, err 

}

```

在这里，我们定义了自己的函数，该函数使用`io.Copy()`来复制文件。`Copy()`函数在尝试复制文件之前会检查源文件是否是常规文件，这是非常合理的。

`main()`函数的最后部分是实现：

```go
func main() { 
   if len(os.Args) != 3 { 
         fmt.Println("Please provide two command line arguments!") 
         os.Exit(1) 
   } 

   sourceFile := os.Args[1] 
   destinationFile := os.Args[2] 
   nBytes, err := Copy(sourceFile, destinationFile) 

   if err != nil { 
         fmt.Printf("The copy operation failed %q\n", err) 
   } else { 
         fmt.Printf("Copied %d bytes!\n", nBytes) 
   } 
} 
```

测试文件是否是另一个文件的精确副本的最佳工具是`diff(1)`实用程序，它也适用于二进制文件。您可以通过阅读其主页了解有关`diff(1)`的更多信息。

执行`notGoodCP.go`将生成以下结果：

```go
$ go run notGoodCP.go testFile aCopy
Copied 871 bytes!
$ diff aCopy testFile
$ wc testFile aCopy
      51     127     871 testFile
      51     127     871 aCopy
     102     254    1742 total
```

# 一次性读取文件！

本节中的技术将使用`ioutil.WriteFile()`和`ioutil.ReadFile()`函数。请注意，`ioutil.ReadFile()`没有实现`io.Reader`接口，因此有一定的限制。

这一部分的 Go 代码名为`readAll.go`，将分为三部分呈现。

第一部分包含以下 Go 代码：

```go
package main 

import ( 
   "fmt" 
   "io/ioutil" 
   "os" 
) 
```

第二部分如下：

```go
func main() { 
   if len(os.Args) != 3 { 
         fmt.Println("Please provide two command line arguments!") 
         os.Exit(1) 
   } 

   sourceFile := os.Args[1] 
   destinationFile := os.Args[2] 
```

最后一部分如下：

```go
   input, err := ioutil.ReadFile(sourceFile) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 

   err = ioutil.WriteFile(destinationFile, input, 0644) 
   if err != nil { 
         fmt.Println("Error creating the new file", destinationFile) 
         fmt.Println(err) 
         os.Exit(1) 
   } 
} 
```

请注意，`ioutil.ReadFile()`函数读取整个文件，当你想要复制大文件时可能不是高效的。同样，`ioutil.WriteFile()`函数将所有给定的数据写入由其第一个参数标识的文件。

执行`readAll.go`将生成以下输出：

```go
$ go run readAll.go testFile aCopy
$ diff aCopy testFile
$ ls -l testFile aCopy
-rw-r--r--  1 mtsouk  staff  871 May  3 21:07 aCopy
-rw-r--r--@ 1 mtsouk  staff  871 May  3 21:04 testFile
$ go run readAll.go doesNotExist aCopy
open doesNotExist: no such file or directory
exit status 1
```

# 一个更好的文件复制程序

本节将介绍一个使用更传统方法的程序，其中使用缓冲区进行读取和复制到新文件。

尽管传统的 Unix 命令行实用程序在没有错误时是静默的，但在自己的工具中打印一些信息，比如读取的字节数，也不是坏事。然而，正确的做法是遵循 Unix 的方式。

存在两个主要原因使`cp.go`比`notGoodCP.go`更好。第一个是开发者可以更多地控制这个过程，但需要编写更多的 Go 代码；第二个是`cp.go`允许你定义缓冲区的大小，这是复制操作中最重要的参数。

`cp.go`的代码将分为五部分呈现。第一部分是预期的序文，以及一个保存读取缓冲区大小的全局变量：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "os" 
   "path/filepath" 
   "strconv" 
) 

var BUFFERSIZE int64 
```

第二部分如下：

```go
func Copy(src, dst string, BUFFERSIZE int64) error { 
   sourceFileStat, err := os.Stat(src) 
   if err != nil { 
         return err 
   } 

   if !sourceFileStat.Mode().IsRegular() { 
         return fmt.Errorf("%s is not a regular file.", src) 
   } 

   source, err := os.Open(src) 
   if err != nil { 
         return err 
   } 
   defer source.Close() 
```

如你所见，缓冲区的大小作为参数传递给了`Copy()`函数。另外两个命令行参数是输入文件名和输出文件名。

第三部分包含了`Copy()`函数的剩余 Go 代码：

```go
   _, err = os.Stat(dst) 
   if err == nil { 
         return fmt.Errorf("File %s already exists.", dst) 
   } 

   destination, err := os.Create(dst) 
   if err != nil { 
         return err 
   } 
   defer destination.Close() 

   if err != nil { 
         panic(err) 
   } 

   buf := make([]byte, BUFFERSIZE) 
   for { 
         n, err := source.Read(buf) 
         if err != nil && err != io.EOF { 
               return err 
         } 
         if n == 0 { 
               break 
         } 

         if _, err := destination.Write(buf[:n]); err != nil { 
               return err 
         } 
   } 
   return err 
} 
```

这里没有什么特别的：你只需不断调用源文件的`Read()`，直到达到输入文件的末尾。每次读取内容时，你都要调用目标文件的`Write()`来保存到输出文件。`buf[:n]`的表示法允许你从`buf`切片中读取前`n`个字符。

第四部分包含以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 4 { 
         fmt.Printf("usage: %s source destination BUFFERSIZE\n", 
filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   source := os.Args[1] 
   destination := os.Args[2] 
   BUFFERSIZE, _ = strconv.ParseInt(os.Args[3], 10, 64) 
```

`filepath.Base()`用于获取可执行文件的名称。

最后一部分如下：

```go
   fmt.Printf("Copying %s to %s\n", source, destination) 
   err := Copy(source, destination, BUFFERSIZE) 
   if err != nil { 
         fmt.Printf("File copying failed: %q\n", err) 
   } 
}

```

执行`cp.go`将生成以下输出：

```go
$ go run cp.go inputFile aCopy 2048
Copying inputFile to aCopy
$ diff inputFile aCopy
```

如果`copy`操作出现问题，你将得到一个描述性的错误消息。

因此，如果程序找不到输入文件，它将打印以下内容：

```go
$ go run cp.go A /tmp/myCP 1024
Copying A to /tmp/myCP
File copying failed: "stat A: no such file or directory"
```

如果程序无法读取输入文件，你将得到以下消息：

```go
$ go run cp.go inputFile /tmp/myCP 1024
Copying inputFile to /tmp/myCP
File copying failed: "open inputFile: permission denied"
```

如果程序无法创建输出文件，它将打印以下错误消息：

```go
$ go run cp.go inputFile /usr/myCP 1024
Copying inputFile to /usr/myCP
File copying failed: "open /usr/myCP: operation not permitted"
```

如果目标文件已经存在，你将得到以下输出：

```go
$ go run cp.go inputFile outputFile 1024
Copying inputFile to outputFile
File copying failed: "File outputFile already exists."
```

# 文件复制操作的基准测试

在文件操作中使用的缓冲区的大小真的很重要，它会影响你的系统工具的性能，特别是当你处理非常大的文件时。

尽管开发可靠的软件应该是你的主要关注点，但你不应该忘记让你的系统软件快速高效！

因此，本节将尝试通过使用不同的缓冲区大小执行`cp.go`，并将其性能与`readAll.go`、`notGoodCP.go`以及`cp(1)`进行比较，以查看缓冲区大小如何影响文件复制操作。

在旧的 Unix 时代，当 Unix 机器上的 RAM 数量太小时，不建议使用大缓冲区。然而，如今，使用大小为`100 MB`的缓冲区并不被认为是不好的做法，特别是当你事先知道你要复制大量的大文件，比如数据库服务器的数据文件。

我们将在测试中使用三个不同大小的文件：这三个文件将使用`dd(1)`实用程序生成，如下所示：

```go
$dd if=/dev/urandom of=100MB count=100000 bs=1024
100000+0 records in
100000+0 records out
102400000 bytes transferred in 6.800277 secs (15058210 bytes/sec)
$ dd if=/dev/urandom of=1GB count=1000000 bs=1024
1000000+0 records in
1000000+0 records out
1024000000 bytes transferred in 68.887482 secs (14864820 bytes/sec)
$ dd if=/dev/urandom of=5GB count=5000000 bs=1024
5000000+0 records in
5000000+0 records out
5120000000 bytes transferred in 339.357738 secs (15087324 bytes/sec)
$ ls -l 100MB 1GB 5GB
-rw-r--r--  1 mtsouk  staff   102400000 May  4 10:30 100MB
-rw-r--r--  1 mtsouk  staff  1024000000 May  4 10:32 1GB
-rw-r--r--  1 mtsouk  staff  5120000000 May  4 10:38 5GB
```

第一个文件大小为`100 MB`，第二个文件大小为`1 GB`，第三个文件大小为`5 GB`。

现在，是时候使用`time(1)`实用程序进行实际测试了。首先，我们将测试`notGoodCP.go`和`readAll.go`的性能：

```go
$ time ./notGoodCP 100MB copy
Copied 102400000 bytes!

real  0m0.153s
user  0m0.003s
sys   0m0.084s
$ time ./notGoodCP 1GB copy
Copied 1024000000 bytes!

real  0m1.461s
user  0m0.029s
sys   0m0.833s
$ time ./notGoodCP 5GB copy
Copied 5120000000 bytes!

real  0m12.193s
user  0m0.161s
sys   0m5.251s
$ time ./readAll 100MB copy

real  0m0.249s
user  0m0.003s
sys   0m0.138s
$ time ./readAll 1GB copy

real  0m3.117s
user  0m0.639s
sys   0m1.644s
$ time ./readAll 5GB copy

real  0m28.918s
user  0m8.106s
sys   0m21.364s
```

现在，您将看到`cp.go`程序使用四种不同的缓冲区大小`16`，`1024`，`1048576`和`1073741824`的结果。首先，让我们复制`100 MB`文件：

```go
$ time ./cp 100MB copy 16
Copying 100MB to copy

real  0m13.240s
user  0m2.699s
sys   0m10.530s
$ time ./cp 100MB copy 1024
Copying 100MB to copy

real  0m0.386s
user  0m0.053s
sys   0m0.303s
$ time ./cp 100MB copy 1048576
Copying 100MB to copy

real  0m0.135s
user  0m0.001s
sys   0m0.075s
$ time ./cp 100MB copy 1073741824
Copying 100MB to copy

real  0m0.390s
user  0m0.011s
sys   0m0.136s
```

然后，我们将复制`1 GB`文件：

```go
$ time ./cp 1GB copy 16
Copying 1GB to copy

real  2m10.054s
user  0m26.497s
sys   1m43.411s
$ time ./cp 1GB copy 1024
Copying 1GB to copy

real  0m3.520s
user  0m0.533s
sys   0m2.944s
$ time ./cp 1GB copy 1048576
Copying 1GB to copy

real  0m1.431s
user  0m0.006s
sys   0m0.749s
$ time ./cp 1GB copy 1073741824
Copying 1GB to copy

real  0m2.033s
user  0m0.012s
sys   0m1.310s
```

接下来，我们将复制`5 GB`文件：

```go
$ time ./cp 5GB copy 16
Copying 5GB to copy

real  10m41.551s
user  2m11.695s
sys   8m29.248s
$ time ./cp 5GB copy 1024
Copying 5GB to copy

real  0m16.558s
user  0m2.415s
sys   0m13.597s
$ time ./cp 5GB copy 1048576
Copying 5GB to copy

real  0m7.172s
user  0m0.028s
sys   0m3.734s
$ time ./cp 5GB copy 1073741824
Copying 5GB to copy

real  0m8.612s
user  0m0.011s
sys   0m4.536s
```

最后，让我们展示 macOS Sierra 附带的`cp(1)`实用程序的结果：

```go
$ time cp 100MB copy

real  0m0.274s
user  0m0.002s
sys   0m0.105s
$ time cp 1GB copy

real  0m2.735s
user  0m0.003s
sys   0m1.014s
$ time cp 5GB copy

real  0m12.199s
user  0m0.012s
sys   0m5.050s
```

下图显示了`time(1)`实用程序输出的实际字段值的图表，显示了所有上述结果：

![](img/a8e66124-879d-4896-b67f-28ae698552a5.png)

各种复制实用程序的基准测试结果

从结果中可以看出，`cp(1)`实用程序表现得相当不错。但是，`cp.go`更加灵活，因为它允许您定义缓冲区的大小。另一方面，如果您使用缓冲区大小较小（16 字节）的`cp.go`，那么整个过程将完全失败！此外，有趣的是`readAll.go`在处理相对较小的文件时表现相当不错，只有在复制`5 GB`文件时才会变慢，对于这样一个小程序来说并不糟糕：您可以将`readAll.go`视为一个快速而粗糙的解决方案！

# 在 Go 中开发 wc(1)

`wc.go`程序的代码的主要思想是，您可以逐行读取文本文件，直到没有内容可读为止。对于每行读取的内容，您可以找出它包含的字符数和单词数。由于需要逐行读取输入，因此最好使用`bufio`而不是普通的`io`，因为它可以简化代码。但是，尝试使用`io`自己实现`wc.go`将是一个非常有教育意义的练习。

但首先，您将看到`wc(1)`实用程序生成以下输出：

```go
$ wc wc.go cp.go
      68     160    1231 wc.go
      45     112     755 cp.go
     113     272    1986 total
```

因此，如果`wc(1)`必须处理多个文件，它会自动生成摘要信息。

在第九章中，*Goroutines - 基本特性*，您将学习如何使用 Go routines 创建`wc.go`的版本。但是，两个版本的核心功能将完全相同！

# 计算单词

代码实现中最棘手的部分是单词计数，它使用了正则表达式：

```go
r := regexp.MustCompile("[^\\s]+") 
for range r.FindAllString(line, -1) { 
numberOfWords++ 
} 
```

在这里，提供的正则表达式根据空白字符分隔行的单词，以便之后对它们进行计数！

# wc.go 代码！

在这个小介绍之后，是时候看看`wc.go`的 Go 代码了，它将分为五个部分呈现。第一部分是预期的序言：

```go
package main 

import ( 
   "bufio" 
   "flag" 
   "fmt" 
   "io" 
   "os" 
   "regexp" 
) 
```

第二部分是`countLines()`函数的实现，其中包括程序的核心功能。请注意，`countLines()`的命名可能不太合适，因为`countLines()`还会计算文件的单词和字符数：

```go
func countLines(filename string) (int, int, int) { 
   var err error 
   var numberOfLines int 
   var numberOfCharacters int 
   var numberOfWords int 
   numberOfLines = 0

   numberOfCharacters = 0 
   numberOfWords = 0 

   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Printf("error opening file %s", err) 
         os.Exit(1) 
   } 
   defer f.Close() 

   r := bufio.NewReader(f) 
   for { 
         line, err := r.ReadString('\n') 

         if err == io.EOF { 
               break 
         } else if err != nil { 
               fmt.Printf("error reading file %s", err)
                break 
         } 

         numberOfLines++ 
         r := regexp.MustCompile("[^\\s]+") 
         for range r.FindAllString(line, -1) { 
               numberOfWords++ 
         } 
         numberOfCharacters += len(line) 
   } 

   return numberOfLines, numberOfWords, numberOfCharacters 
} 
```

这里有很多有趣的事情。首先，您可以看到前一节中呈现的 Go 代码，用于计算每行的单词。计算行数很容易，因为每次`bufio`读取器读取新行时，`numberOfLines`变量的值就会增加一。`ReadString()`函数告诉程序读取输入直到第一个`'\n'`的出现：多次调用`ReadString()`意味着您正在逐行读取文件。

接下来，您可以看到`countLines()`函数返回三个整数值。最后，计算字符数是通过`len()`函数实现的，该函数返回给定字符串中的字符数，在这种情况下是读取的行。`for`循环在获得`io.EOF`错误消息时终止，这表示从输入文件中没有剩余内容可读取。

`wc.go`的第三部分从`main()`函数的实现开始，其中还包括`flag`包的配置：

```go
func main() { 
   minusC := flag.Bool("c", false, "Characters") 
   minusW := flag.Bool("w", false, "Words") 
   minusL := flag.Bool("l", false, "Lines") 

   flag.Parse() 
   flags := flag.Args() 

   if len(flags) == 0 { 
         fmt.Printf("usage: wc <file1> [<file2> [... <fileN]]\n") 
         os.Exit(1) 
   } 

   totalLines := 0 
   totalWords := 0 
   totalCharacters := 0 
   printAll := false 

   for _, filename := range flag.Args() { 
```

最后的`for`语句用于处理程序给定的所有输入文件。`wc.go`程序支持三个标志：`-c`标志用于打印字符计数，`-w`标志用于打印单词计数，`-l`标志用于打印行计数。

第四部分是以下内容：

```go
         numberOfLines, numberOfWords, numberOfCharacters := countLines(filename) 

         totalLines = totalLines + numberOfLines 
         totalWords = totalWords + numberOfWords 
         totalCharacters = totalCharacters + numberOfCharacters 

         if (*minusC && *minusW && *minusL) || (!*minusC && !*minusW && !*minusL) { 
               fmt.Printf("%d", numberOfLines) 
               fmt.Printf("\t%d", numberOfWords) 
               fmt.Printf("\t%d", numberOfCharacters) 
               fmt.Printf("\t%s\n", filename) 
               printAll = true 
               continue 
         } 

         if *minusL { 
               fmt.Printf("%d", numberOfLines) 
         } 

         if *minusW { 
               fmt.Printf("\t%d", numberOfWords) 
         } 

         if *minusC { 
               fmt.Printf("\t%d", numberOfCharacters) 
         } 

         fmt.Printf("\t%s\n", filename) 
   } 
```

这部分涉及根据命令行标志在每个文件基础上打印信息。正如您所看到的，这里的大部分 Go 代码是用于根据命令行标志处理输出。

最后一部分是以下内容：

```go
   if (len(flags) != 1) && printAll { 
         fmt.Printf("%d", totalLines) 
         fmt.Printf("\t%d", totalWords) 
         fmt.Printf("\t%d", totalCharacters) 
         fmt.Println("\ttotal") 
return 
   } 

   if (len(flags) != 1) && *minusL { 
         fmt.Printf("%d", totalLines) 
   } 

   if (len(flags) != 1) && *minusW { 
         fmt.Printf("\t%d", totalWords) 
   } 

   if (len(flags) != 1) && *minusC { 
         fmt.Printf("\t%d", totalCharacters) 
   } 

   if len(flags) != 1 { 
         fmt.Printf("\ttotal\n") 
   } 
} 
```

这是您根据程序的标志打印的总行数、单词数和字符数。再次强调，这里的大部分 Go 代码是用于根据命令行标志修改输出。

执行`wc.go`将生成以下输出：

```go
$ go build wc.go
$ ls -l wc
-rwxr-xr-x  1 mtsouk  staff  2264384 Apr 29 21:10 wc
$ ./wc wc.go sparse.go notGoodCP.go
120   280   2319  wc.go
44    98    697   sparse.go
27    61    418   notGoodCP.go
191   439   3434  total
$ ./wc -l wc.go sparse.go
120   wc.go
44    sparse.go
164   total
$ ./wc -w -l wc.go sparse.go
120   280   wc.go
44    98    sparse.go
164   378   total
```

这里有一个微妙的地方：使用 Go 源文件作为`go run wc.go`命令的命令行参数将失败。这将发生是因为编译器将尝试编译 Go 源文件，而不是将它们视为`go run wc.go`命令的命令行参数。以下输出证明了这一点：

```go
$ go run wc.go sparse.go
# command-line-arguments
./sparse.go:11: main redeclared in this block
      previous declaration at ./wc.go:49
$ go run wc.go wc.go
package main: case-insensitive file name collision:
"wc.go" and "wc.go"
$ go run wc.go cp.go sparse.go
# command-line-arguments
./cp.go:35: main redeclared in this block
      previous declaration at ./wc.go:49
./sparse.go:11: main redeclared in this block
      previous declaration at ./cp.go:35
```

此外，尝试在 Go 版本 1.3.3 的 Linux 系统上执行`wc.go`将失败，并显示以下错误消息：

```go
$ go version
go version go1.3.3 linux/amd64
$ go run wc.go
# command-line-arguments
./wc.go:40: syntax error: unexpected range, expecting {
./wc.go:46: non-declaration statement outside function body
./wc.go:47: syntax error: unexpected }
```

# 比较`wc.go`和`wc(1)`的性能

在这一小节中，我们将比较我们版本的`wc(1)`与 macOS Sierra 10.12.6 自带的`wc(1)`的性能。首先，我们将执行`wc.go`：

```go
$ file wc
wc: Mach-O 64-bit executable x86_64
$ time ./wc *.data
672320      3361604     9413057     connections.data
269123      807369      4157790     diskSpace.data
672040      1344080     8376070     memory.data
1344533     2689066     5378132     pageFaults.data
269465      792715      4068250     uptime.data
3227481     8994834     31393299    total

real  0m17.467s
user  0m22.164s
sys   0m3.885s
```

然后，我们将执行 macOS 版本的`wc(1)`来处理相同的文件：

```go
$ file `which wc`
/usr/bin/wc: Mach-O 64-bit executable x86_64
$ time wc *.data
672320 3361604 9413057 connections.data
269123  807369 4157790 diskSpace.data
672040 1344080 8376070 memory.data
1344533 2689066 5378132 pageFaults.data
269465  792715 4068250 uptime.data
3227481 8994834 31393299 total

real  0m0.086s
user  0m0.076s
sys   0m0.007s
```

让我们先看看好消息；这两个实用程序生成了完全相同的输出，这意味着我们的`wc(1)`的 Go 版本效果很好，可以处理大型文本文件！

现在，坏消息是，`wc.go`很慢！`wc(1)`处理所有五个文件不到一秒，而`wc.go`执行相同的任务几乎需要 18 秒！

在开发任何类型的软件时，无论在任何平台上，使用任何编程语言，一般的想法是，您应该在尝试优化之前先尝试拥有一个没有错误的工作版本，而不是反过来！

# 逐个字符读取文本文件

尽管不需要逐个字符读取文本文件来开发`wc(1)`实用程序，但了解如何在 Go 中实现它是很好的。文件的名称将是`charByChar.go`，并将分为四部分呈现。

第一部分是以下 Go 代码：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "io/ioutil" 
   "os" 
   "strings" 
) 
```

尽管`charByChar.go`的 Go 代码行数不多，但它需要大量的 Go 标准包，这是一个天真的迹象，表明它实现的任务并不是微不足道的。第二部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Not enough arguments!") 
         os.Exit(1) 
   } 
   input := arguments[1] 
```

第三部分是以下内容：

```go
   buf, err := ioutil.ReadFile(input) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
```

最后一部分包含以下 Go 代码：

```go
   in := string(buf) 
   s := bufio.NewScanner(strings.NewReader(in)) 
   s.Split(bufio.ScanRunes) 

   for s.Scan() { 
         fmt.Print(s.Text()) 
   } 
} 
```

在这里，`ScanRunes`是一个分割函数，它将每个字符（rune）作为一个标记返回。然后，调用`Scan()`允许我们逐个处理每个字符。还有`ScanWords`和`ScanLines`用于分别获取单词和行。如果您在程序的最后一条语句中使用`fmt.Println(s.Text())`而不是`fmt.Print(s.Text())`，那么每个字符将被单独打印在自己的行上，程序的任务也将更加明显。

执行`charByChar.go`将生成以下输出：

```go
$ go run charByChar.go test
package main
...
```

`wc(1)`命令可以通过比较输入文件和`charByChar.go`生成的输出来验证`charByChar.go`的 Go 代码的正确性：

```go
$ go run charByChar.go test | wc
      32      54     439
$ wc test
      32      54     439 test
```

# 进行一些文件编辑！

本节将介绍一个将文件中的制表符转换为空格字符，反之亦然的 Go 程序！这通常是文本编辑器的工作，但了解如何自行执行此操作也是很好的。

代码将保存在`tabSpace.go`中，并将分为四部分呈现。

请注意，`tabSpace.go`按行读取文本文件，但您也可以开发一个按字符读取文本文件的版本。

在当前的实现中，所有工作都是通过正则表达式、模式匹配和搜索替换操作完成的。

第一部分是预期的序言：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "io" 
   "os" 
   "path/filepath" 
   "strings" 
) 
```

第二部分包含以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 3 { 
         fmt.Printf("Usage: %s [-t|-s] filename!\n", filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 
   convertTabs := false 
   convertSpaces := false 
   newLine := "" 

   option := os.Args[1] 
   filename := os.Args[2] 
   if option == "-t" { 
         convertTabs = true 
   } else if option == "-s" { 
         convertSpaces = true 
   } else { 
         fmt.Println("Unknown option!") 
         os.Exit(1) 
   } 
```

第三部分包含以下 Go 代码：

```go
   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Printf("error opening %s: %s", filename, err) 
         os.Exit(1) 
   } 
   defer f.Close() 

   r := bufio.NewReader(f) 
   for { 
         line, err := r.ReadString('\n') 

         if err == io.EOF { 
               break 
         } else if err != nil { 
               fmt.Printf("error reading file %s", err) 
               os.Exit(1) 
         } 
```

最后一部分如下：

```go
         if convertTabs == true { 
               newLine = strings.Replace(line, "\t", "    ", -1) 
         } else if convertSpaces == true { 
               newLine = strings.Replace(line, "    ", "\t", -1) 
         } 

         fmt.Print(newLine) 
   } 
} 
```

这部分是使用适当的`strings.Replace()`调用发生魔术的地方。在当前的实现中，每个制表符都被四个空格字符替换，反之亦然，但您可以通过修改 Go 代码来更改这一点。

再次，`tabSpace.go`的很大一部分涉及错误处理，因为当您尝试打开文件进行读取时，可能会发生许多奇怪的事情！

根据 Unix 哲学，`tabSpace.go`的输出将打印在屏幕上，并且不会保存在新的文本文件中。使用`tabSpace.go`与`wc(1)`可以证明其正确性：

```go
$ go run tabSpace.go -t cp.go > convert
$ wc convert cp.go
    76     192    1517 convert 
      76     192    1286 cp.go
     152     384    2803 total
$ go run tabSpace.go -s convert | wc
      76     192    1286
```

# 进程间通信

**进程间通信**（**IPC**），简单地说，就是允许 Unix 进程相互通信。存在各种技术允许进程和程序相互通信。在 Unix 系统中使用最广泛的技术是管道，自早期 Unix 以来就存在。第八章，*进程和信号*，将更多地讨论如何在 Go 中实现 Unix 管道。另一种 IPC 形式是 Unix 域套接字，也将在第八章，*进程和信号*中讨论。

第十二章，*网络编程*，将讨论另一种进程间通信形式，即网络套接字。共享内存也存在，但 Go 反对使用共享内存作为通信手段。第九章，*Goroutines - 基本特性*，和第十章，*Goroutines - 高级特性*，将展示允许 goroutines 与其他 goroutines 通信并共享和交换数据的各种技术。

# Go 中的稀疏文件

使用`os.Seek()`函数创建的大文件可能存在空洞，并且占用的磁盘块比没有空洞的相同大小的文件少；这样的文件称为稀疏文件。本节将开发一个创建稀疏文件的程序。

`sparse.go`的 Go 代码将分为三部分呈现。第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "log" 
   "os" 
   "path/filepath" 
   "strconv" 
) 
```

`sparse.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 3 { 
         fmt.Printf("usage: %s SIZE filename\n", filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   SIZE, _ := strconv.ParseInt(os.Args[1], 10, 64) 
   filename := os.Args[2] 

   _, err := os.Stat(filename) 
   if err == nil { 
         fmt.Printf("File %s already exists.\n", filename) 
         os.Exit(1) 
   } 
```

`strconv.ParseInt()`函数用于将定义稀疏文件大小的命令行参数从其字符串值转换为其整数值。此外，`os.Stat()`调用确保您不会意外覆盖现有文件。

最后一部分是发生操作的地方：

```go
   fd, err := os.Create(filename) 
   if err != nil { 
         log.Fatal("Failed to create output") 
   } 

   _, err = fd.Seek(SIZE-1, 0) 
   if err != nil { 
         fmt.Println(err) 
         log.Fatal("Failed to seek") 
   } 

   _, err = fd.Write([]byte{0}) 
   if err != nil { 
         fmt.Println(err) 
         log.Fatal("Write operation failed") 
   } 

   err = fd.Close() 
   if err != nil { 
         fmt.Println(err) 
         log.Fatal("Failed to close file") 
   } 
} 
```

首先，您尝试使用`os.Create()`创建所需的稀疏文件。然后，您调用`fd.Seek()`以使文件变大而不添加实际数据。最后，您使用`fd.Write()`向其写入一个字节。由于您没有其他事情要做，因此调用`fd.Close()`，完成操作。

执行`sparse.go`会生成以下输出：

```go
$ go run sparse.go 1000 test
$ go run sparse.go 1000 test
File test already exists.
exit status 1
```

您如何判断文件是否是稀疏文件？您将在一会儿学到这一点，但首先让我们创建一些文件：

```go
$ go run sparse.go 100000 testSparse $ dd if=/dev/urandom  bs=1 count=100000 of=noSparseDD
100000+0 records in
100000+0 records out
100000 bytes (100 kB) copied, 0.152511 s, 656 kB/s
$ dd if=/dev/urandom seek=100000 bs=1 count=0 of=sparseDD
0+0 records in
0+0 records out
0 bytes (0 B) copied, 0.000159399 s, 0.0 kB/s
$ ls -l noSparseDD sparseDD testSparse
-rw-r--r-- 1 mtsouk mtsouk 100000 Apr 29 21:43 noSparseDD
-rw-r--r-- 1 mtsouk mtsouk 100000 Apr 29 21:43 sparseDD
-rw-r--r-- 1 mtsouk mtsouk 100000 Apr 29 21:40 testSparse
```

请注意，某些 Unix 变体将不会创建稀疏文件：我想到的第一个这样的 Unix 变体是使用 HFS 文件系统的 macOS。因此，为了获得更好的结果，您可以在 Linux 机器上执行所有这些命令。

那么，您如何判断这三个文件中的任何一个是否是稀疏文件？`ls(1)`实用程序的`-s`标志显示文件实际使用的文件系统块数。因此，`ls -ls`命令的输出允许您检测是否正在处理稀疏文件：

```go
$ ls -ls noSparseDD sparseDD testSparse
104 -rw-r--r-- 1 mtsouk mtsouk 100000 Apr 29 21:43 noSparseDD
      0 -rw-r--r-- 1 mtsouk mtsouk 100000 Apr 29 21:43 sparseDD
      8 -rw-r--r-- 1 mtsouk mtsouk 100000 Apr 29 21:40 testSparse
```

现在看输出的第一列。使用`dd(1)`实用程序生成的`noSparseDD`文件不是稀疏文件。`sparseDD`文件是使用`dd(1)`实用程序生成的稀疏文件。最后，`testSparse`也是使用`sparse.go`创建的稀疏文件。

# 读取和写入数据记录

本节将教你如何处理写入和读取数据记录。记录与其他类型的文本数据的不同之处在于记录具有特定数量的字段的给定结构：可以将其视为关系数据库中的表中的一行。实际上，如果你想在 Go 中开发自己的数据库服务器，记录可以非常有用来在表中存储数据！

`records.go`的 Go 代码将以 CSV 格式保存数据，并将分为四个部分呈现。第一部分包含以下 Go 代码：

```go
package main 

import ( 
   "encoding/csv" 
   "fmt" 
   "os" 
) 
```

因此，这是你必须声明你要以 CSV 格式读取或写入数据的地方。第二部分如下：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Need just one filename!") 
         os.Exit(-1) 
   } 

   filename := os.Args[1] 
   _, err := os.Stat(filename) 
   if err == nil { 
         fmt.Printf("File %s already exists.\n", filename) 
         os.Exit(1) 
   } 
```

程序的第三部分如下：

```go
   output, err := os.Create(filename) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(-1) 
   } 
   defer output.Close() 

   inputData := [][]string{{"M", "T", "I."}, {"D", "T", "I."}, 
{"M", "T", "D."}, {"V", "T", "D."}, {"A", "T", "D."}} 
   writer := csv.NewWriter(output) 
   for _, record := range inputData { 
         err := writer.Write(record) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(-1) 
         } 
   } 
   writer.Flush() 
```

你应该熟悉这部分的操作；与本章迄今为止所见的最大不同之处在于写入器来自`csv`包。

`records.go`的最后一部分包含以下 Go 代码：

```go
   f, err := os.Open(filename) 
   if err != nil { 
         fmt.Println(err) 
         return 
   } 
   defer f.Close() 

   reader := csv.NewReader(f) 
   reader.FieldsPerRecord = -1 
   allRecords, err := reader.ReadAll() 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 

   for _, rec := range allRecords { 
         fmt.Printf("%s:%s:%s\n", rec[0], rec[1], rec[2]) 
   } 
} 
```

`reader`一次性读取整个文件，以使整个操作更快。但是，如果你处理大型数据文件，你可能需要每次读取文件的较小部分，直到你读取完整个文件。使用的`reader`来自`csv`包。

执行`records.go`将创建以下输出：

```go
$ go run records.go recordsDataFile
M:T:I. 
D:T:I.
M:T:D.
V:T:D.
A:T:D.
$ ls -l recordsDataFile
-rw-r--r--  1 mtsouk  staff  35 May  2 19:20 recordsDataFile
```

名为`recordsDataFile`的 CSV 文件包含以下数据：

```go
$ cat recordsDataFile
M,T,I.
D,T,I.
M,T,D.
V,T,D.
A,T,D.
```

# Go 中的文件锁定

有时，你不希望同一进程的任何其他子进程更改文件甚至访问文件，因为你正在更改其数据，你不希望其他进程读取不完整或不一致的数据。尽管你将在第九章和第十章中学到更多关于文件锁定和 go 例程的知识，*Goroutines - 基本特性*和*Goroutines - 高级特性*，但本章将呈现一个简单的 Go 示例，没有详细的解释，以便让你了解事情是如何工作的：你应该等到第九章和第十章中学到更多。

所呈现的技术将使用`Mutex`，这是一种通用的同步机制。`Mutex`锁将允许我们从同一 Go 进程中锁定文件。因此，这种技术与使用`flock(2)`系统调用无关。

存在各种文件锁定技术。其中之一是通过创建一个额外的文件来表示另一个程序或进程正在使用给定的资源。所呈现的技术更适用于使用多个 go 例程的程序。

写入的文件锁定技术将在`fileLocking.go`中进行演示，该文件将分为四个部分呈现。第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "math/rand" 
   "os" 
   "sync" 
   "time" 
) 

var mu sync.Mutex 

func random(min, max int) int { 
   return rand.Intn(max-min) + min 
} 
```

第二部分如下：

```go
func writeDataToFile(i int, file *os.File, w *sync.WaitGroup) { 
   mu.Lock() 
   time.Sleep(time.Duration(random(10, 1000)) * time.Millisecond) 
   fmt.Fprintf(file, "From %d, writing %d\n", i, 2*i) 
   fmt.Printf("Wrote from %d\n", i) 
   w.Done() 
mu.Unlock() 
} 
```

使用`mu.Lock()`语句锁定文件，使用`mu.Unlock()`语句解锁文件。

第三部分如下：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Println("Please provide one command line argument!") 
         os.Exit(-1) 
   } 

   filename := os.Args[1] 
   number := 3 

   file, err := os.OpenFile(filename, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0644) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
```

最后一部分是以下 Go 代码：

```go
   var w *sync.WaitGroup = new(sync.WaitGroup) 
   w.Add(number) 

   for r := 0; r < number; r++ { 
         go writeDataToFile(r, file, w) 
   } 

   w.Wait() 
} 
```

执行`fileLocking.go`将创建以下输出：

```go
$ go run fileLocking.go 123
Wrote from 0
Wrote from 2
Wrote from 1
$ cat /tmp/swtag.log
From 0, writing 0
From 2, writing 4
From 1, writing 2
```

正确的`fileLocking.go`在`writeDataToFile()`函数的末尾调用了`mu.Unlock()`，这允许所有 goroutines 使用该文件。如果你从`writeDataToFile()`函数中删除对`mu.Unlock()`的调用，并执行`fileLocking.go`，你将得到以下输出：

```go
$ go run fileLocking.go 123
Wrote from 2
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0xc42001024c)
      /usr/local/Cellar/go/1.8.1/libexec/src/runtime/sema.go:47 +0x34
sync.(*WaitGroup).Wait(0xc420010240)
      /usr/local/Cellar/go/1.8.1/libexec/src/sync/waitgroup.go:131 +0x7a
main.main()
     /Users/mtsouk/Desktop/goBook/ch/ch6/code/fileLocking.go:47 +0x172

goroutine 5 [semacquire]:
sync.runtime_SemacquireMutex(0x112dcbc)
     /usr/local/Cellar/go/1.8.1/libexec/src/runtime/sema.go:62 +0x34
sync.(*Mutex).Lock(0x112dcb8)
      /usr/local/Cellar/go/1.8.1/libexec/src/sync/mutex.go:87 +0x9d
main.writeDataToFile(0x0, 0xc42000c028, 0xc420010240)
      /Users/mtsouk/Desktop/goBook/ch/ch6/code/fileLocking.go:18 +0x3f
created by main.main
      /Users/mtsouk/Desktop/goBook/ch/ch6/code/fileLocking.go:44 +0x151

goroutine 6 [semacquire]:
sync.runtime_SemacquireMutex(0x112dcbc)
      /usr/local/Cellar/go/1.8.1/libexec/src/runtime/sema.go:62 +0x34
sync.(*Mutex).Lock(0x112dcb8)
      /usr/local/Cellar/go/1.8.1/libexec/src/sync/mutex.go:87 +0x9d
main.writeDataToFile(0x1, 0xc42000c028, 0xc420010240)
      /Users/mtsouk/Desktop/goBook/ch/ch6/code/fileLocking.go:18 +0x3f
created by main.main
    /Users/mtsouk/Desktop/goBook/ch/ch6/code/fileLocking.go:44 +0x151 exit status 2
$ cat 123
From 2, writing 4
```

得到这个输出的原因是，除了第一个 goroutine 能够执行`mu.Lock()`语句之外，其他的都无法获得`Mutex`。因此，它们无法写入文件，这意味着它们永远无法完成工作并永远等待，这就是 Go 生成上述错误消息的原因。

如果您对这个例子不完全理解，您应该等到第九章，“Goroutines - Basic Features”和第十章，“Goroutines - Advanced Features”。

# `dd`实用程序的简化版 Go 版本

`dd(1)`工具可以做很多事情，但本节将实现其功能的一小部分。我们的`dd(1)`版本将包括对两个命令行标志的支持：一个用于指定以字节为单位的块大小（`-bs`），另一个用于指定将要写入的块的总数（`-count`）。将这两个值相乘将给出生成文件的大小（以字节为单位）。

Go 代码保存为`ddGo.go`，将分为四部分呈现给您。第一部分是预期的序言：

```go
package main 

import ( 
   "flag" 
   "fmt" 
   "math/rand" 
   "os" 
   "time" 
) 
```

第二部分包含两个函数的 Go 代码：

```go
func random(min, max int) int { 
   return rand.Intn(max-min) + min 
} 

func createBytes(buf *[]byte, count int) { 
   if count == 0 { 
         return 
   } 
   for i := 0; i < count; i++ { 
         intByte := byte(random(0, 9)) 
         *buf = append(*buf, intByte) 
   } 
} 
```

第一个函数是用于获取随机数的，第二个函数是用于创建一个带有所需大小的随机数填充的字节片。

`ddGo.go`的第三部分如下：

```go
func main() { 
   minusBS := flag.Int("bs", 0, "Block Size") 
   minusCOUNT := flag.Int("count", 0, "Counter") 
   flag.Parse() 
   flags := flag.Args() 

   if len(flags) == 0 { 
         fmt.Println("Not enough arguments!") 
         os.Exit(-1) 
   } 

   if *minusBS < 0 || *minusCOUNT < 0 { 
         fmt.Println("Count or/and Byte Size < 0!") 
         os.Exit(-1) 
   } 

   filename := flags[0] 
   rand.Seed(time.Now().Unix()) 

   _, err := os.Stat(filename) 
   if err == nil { 
         fmt.Printf("File %s already exists.\n", filename) 
         os.Exit(1) 
   } 

   destination, err := os.Create(filename) 
   if err != nil { 
         fmt.Println("os.Create:", err) 
         os.Exit(1) 
   } 
```

在这里，您主要处理程序的命令行参数。

最后一部分如下：

```go
   buf := make([]byte, *minusBS) 
   buf = nil 
   for i := 0; i < *minusCOUNT; i++ { 
         createBytes(&buf, *minusBS) 
         if _, err := destination.Write(buf); err != nil { 
               fmt.Println(err) 
               os.Exit(-1) 
         } 
         buf = nil 
   } 
} 
```

每次调用`createBytes()`时清空`buf`字节片的原因是，您不希望`buf`字节片每次调用`createBytes()`函数时都变得越来越大。这是因为`append()`函数会在不触及现有数据的情况下在切片的末尾添加数据。

在我写的`ddGo.go`的第一个版本中，我忘记在每次调用`createBytes()`之前清空`buf`字节片。因此，生成的文件比预期的要大！我花了一段时间和几个`fmt.Println(buf)`语句才找到这种意外行为的原因！

执行`ddGo.go`将快速生成您想要的文件：

```go
$ time go run ddGo.go -bs=10000 -count=5000 test3

real  0m1.655s
user  0m1.576s
sys   0m0.104s
$ ls -l test3
-rw-r--r--  1 mtsouk  staff  50000000 May  6 15:27 test3
```

此外，使用随机数使得生成的文件大小彼此不同：

```go
$ go run ddGo.go -bs=100 -count=50 test1
$ go run ddGo.go -bs=100 -count=50 test2
$ ls -l test1 test2
-rw-r--r--  1 mtsouk  staff  5000 May  6 15:26 test1
-rw-r--r--  1 mtsouk  staff  5000 May  6 15:26 test2
$ diff test1 test2
Binary files test1 and test2 differ
```

# 练习

1.  访问[`golang.org/pkg/bufio/`](https://golang.org/pkg/bufio/)上的`bufio`包的文档页面。

1.  访问[`golang.org/pkg/io/`](https://golang.org/pkg/io/)上的`io`包的文档。

1.  尝试使`wc.go`更快。

1.  实现`tabSpace.go`的功能，但尝试逐个字符而不是逐行读取输入文本文件。

1.  更改`tabSpace.go`的代码，以便能够将替换制表符的空格数作为命令行参数。

1.  了解有关小端和大端表示的更多信息。

# 总结

在本章中，我们讨论了 Go 中的文件输入和输出。在其他事情中，我们开发了`wc(1)`，`dd(1)`和`cp(1)` Unix 命令行实用程序的 Go 版本，同时更多地了解了 Go 标准库的`io`和`bufio`包，它们允许您从文件中读取和写入。

在下一章中，我们将讨论另一个重要主题，即 Go 如何处理 Unix 机器的系统文件的方式。此外，您将学习如何读取和更改 Unix 文件权限，以及如何找到文件的所有者和组。此外，我们将讨论日志文件以及如何使用模式匹配从日志文件中获取所需的信息。
