# 文件和目录

在上一章中，我们谈到了许多重要的主题，包括开发和使用 Go 包，Go 数据结构，算法和 GC。然而，直到现在，我们还没有开发任何实际的系统实用程序。这很快就会改变，因为从这一非常重要的章节开始，我们将开始学习如何使用 Go 来开发真正的系统实用程序，以便处理文件系统的各种类型的文件和目录。

您应该始终记住，Unix 将一切都视为文件，包括符号链接、目录、网络设备、网络套接字、整个硬盘驱动器、打印机和纯文本文件。本章的目的是说明 Go 标准库如何允许我们了解路径是否存在，以及如何搜索目录结构以检测我们想要的文件类型。此外，本章将通过 Go 代码作为证据证明，许多传统的 Unix 命令行实用程序在处理文件和目录时并不难实现。

在本章中，您将学习以下主题：

+   将帮助您操作目录和文件的 Go 包

+   使用`flag`包轻松处理命令行参数和选项

+   在 Go 中开发`which(1)`命令行实用程序的版本

+   在 Go 中开发`pwd(1)`命令行实用程序的版本

+   删除和重命名文件和目录

+   轻松遍历目录树

+   编写`find(1)`实用程序的版本

+   在另一个地方复制目录结构

# 有用的 Go 包

允许您将文件和目录视为实体的最重要的包是`os`包，在本章中我们将广泛使用它。如果您将文件视为带有内容的盒子，`os`包允许您移动它们，将它们放入废纸篓，更改它们的名称，访问它们，并决定您想要使用哪些文件，而`io`包，将在下一章中介绍，允许您操作盒子的内容，而不必太担心盒子本身！

`flag`包，您将很快看到，让您定义和处理自己的标志，并操作 Go 程序的命令行参数。

`filepath`包非常方便，因为它包括`filepath.Walk()`函数，允许您以简单的方式遍历整个目录结构。

# 重新审视命令行参数！

正如我们在第二章中所看到的，*使用 Go 编写程序*，使用`if`语句无法高效处理多个命令行参数和选项。解决这个问题的方法是使用`flag`包，这将在这里解释。

记住`flag`包是一个标准的 Go 包，您不必在其他地方搜索标志的功能非常重要。

# flag 包

`flag`包为我们解析命令行参数和选项做了脏活，因此无需编写复杂和令人困惑的 Go 代码。此外，它支持各种类型的参数，包括字符串、整数和布尔值，这样可以节省时间，因为您不必执行任何数据类型转换。

`usingFlag.go`程序演示了`flag`Go 包的使用，并将分为三个部分呈现。第一部分包含以下 Go 代码：

```go
package main 

import ( 
   "flag" 
   "fmt" 
) 
```

程序的最重要的 Go 代码在第二部分中，如下所示：

```go
func main() { 
   minusO := flag.Bool("o", false, "o") 
   minusC := flag.Bool("c", false, "c") 
   minusK := flag.Int("k", 0, "an int") 

   flag.Parse() 
```

在这部分，您可以看到如何定义您感兴趣的标志。在这里，您定义了`-o`、`-c`和`-k`。虽然前两个是布尔标志，但`-k`标志需要一个整数值，可以写成`-k=123`。

最后一部分包含以下 Go 代码：

```go
   fmt.Println("-o:", *minusO) 
   fmt.Println("-c:", *minusC) 
   fmt.Println("-K:", *minusK) 

   for index, val := range flag.Args() { 
         fmt.Println(index, ":", val) 
   } 
} 
```

在这部分中，您可以看到如何读取选项的值，这也允许您判断选项是否已设置。另外，`flag.Args()`允许您访问程序未使用的命令行参数。

`usingFlag.go`的使用和输出在以下输出中展示：

```go
$ go run usingFlag.go
-o: false
-c: false
-K: 0
$ go run usingFlag.go -o a b
-o: true
-c: false
-K: 0
0 : a
1 : b
```

但是，如果您忘记输入命令行选项（`-k`）的值，或者提供的值类型错误，您将收到以下消息，并且程序将终止：

```go
$ ./usingFlag -k
flag needs an argument: -k
Usage of ./usingFlag:
  -c  c
  -k int
      an int
  -o  o $ ./usingFlag -k=abc invalid value "abc" for flag -k: strconv.ParseInt: parsing "abc": invalid syntax
Usage of ./usingFlag:
  -c  c
  -k int
      an int
  -o  o
```

如果您不希望程序在出现解析错误时退出，可以使用`flag`包提供的`ErrorHandling`类型，它允许您通过`NewFlagSet()`函数更改`flag.Parse()`在错误时的行为。但是，在系统编程中，通常希望在一个或多个命令行选项出现错误时退出实用程序。

# 处理目录

目录允许您创建一个结构，并以便于您组织和搜索文件的方式存储文件。实际上，目录是文件系统上包含其他文件和目录列表的条目。这是通过**inode**的帮助发生的，inode 是保存有关文件和目录的信息的数据结构。

如下图所示，目录被实现为分配给 inode 的名称列表。因此，目录包含对自身、其父目录和其每个子目录的条目，其中其他内容可以是常规文件或其他目录：

您应该记住的是，inode 保存有关文件的元数据，而不是文件的实际数据。

![](img/e74853d3-8d25-49c3-a968-dc7713c53a72.png)

inode 的图形表示

# 关于符号链接

**符号链接**是指向文件或目录的指针，在访问时解析。符号链接，也称为**软链接**，不等同于它们所指向的文件或目录，并且允许指向无处，这有时可能会使事情复杂化。

保存在`symbLink.go`中并分为两部分的以下 Go 代码允许您检查路径或文件是否是符号链接。第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "path/filepath" 
) 

func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide an argument!") 
         os.Exit(1) 
   } 
   filename := arguments[1] 
```

这里没有发生什么特别的事情：您只需要确保获得一个命令行参数，以便有东西可以测试。第二部分是以下 Go 代码：

```go
   fileinfo, err := os.Lstat(fil /etcename) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 

   if fileinfo.Mode()&os.ModeSymlink != 0 { 
         fmt.Println(filename, "is a symbolic link") 
         realpath, err := filepath.EvalSymlinks(filename) 
         if err == nil { 
               fmt.Println("Path:", realpath) 
         } 
   } 

}
```

`SymbLink.go`的前述代码比通常更加神秘，因为它使用了更低级的函数。确定路径是否为真实路径的技术涉及使用`os.Lstat()`函数，该函数提供有关文件或目录的信息，并在`os.Lstat()`调用的返回值上使用`Mode()`函数，以将结果与`os.ModeSymlink`常量进行比较，该常量是符号链接位。

此外，还存在`filepath.EvalSymlinks()`函数，允许您评估任何存在的符号链接并返回文件或目录的真实路径，这也在`symbLink.go`中使用。这可能会让您认为我们在为这样一个简单的任务使用大量的 Go 代码，这在一定程度上是正确的，但是当您开发系统软件时，您必须考虑所有可能性并保持谨慎。

执行`symbLink.go`，它只需要一个命令行参数，会生成以下输出：

```go
$ go run symbLink.go /etc
/etc is a symbolic link
Path: /private/etc
```

在本章的其余部分，您还将看到一些前面提到的 Go 代码作为更大程序的一部分。

# 实现 pwd(1)命令

当我开始考虑如何实现一个程序时，我的脑海中涌现了很多想法，有时决定要做什么变得太困难了！关键在于做一些事情，而不是等待，因为当您编写代码时，您将能够判断您所采取的方法是好还是不好，以及您是否应该尝试另一种方法。

`pwd(1)`命令行实用程序非常简单，但工作得很好。如果您编写大量 shell 脚本，您应该已经知道`pwd(1)`，因为当您想要获取与正在执行的脚本位于同一目录中的文件或目录的完整路径时，它非常方便。

`pwd.go`的 Go 代码将分为两部分，并且只支持`-P`命令行选项，该选项解析所有符号链接并打印物理当前工作目录。`pwd.go`的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "path/filepath" 
) 

func main() { 
   arguments := os.Args 

   pwd, err := os.Getwd() 
   if err == nil { 
         fmt.Println(pwd) 
   } else { 
         fmt.Println("Error:", err) 
   } 
```

第二部分如下：

```go
   if len(arguments) == 1 { 
         return 
   } 

   if arguments[1] != "-P" { 
         return 
   } 

   fileinfo, err := os.Lstat(pwd) 
   if fileinfo.Mode()&os.ModeSymlink != 0 { 
         realpath, err := filepath.EvalSymlinks(pwd) 
         if err == nil { 
               fmt.Println(realpath) 
         } 
   } 
} 
```

请注意，如果当前目录可以由多个路径描述，这可能发生在使用符号链接时，`os.Getwd()`可以返回其中任何一个。此外，如果给出了`-P`选项并且正在处理一个目录是符号链接，您需要重用`symbolLink.go`中找到的一些 Go 代码来发现物理当前工作目录。此外，不在`pwd.go`中使用`flag`包的原因是我发现代码现在的方式更简单。

执行`pwd.go`将生成以下输出：

```go
$ go run pwd.go
/Users/mtsouk/Desktop/goBook/ch/ch5/code
```

在 macOS 机器上，`/tmp`目录是一个符号链接，这可以帮助我们验证`pwd.go`是否按预期工作：

```go
$ go run pwd.go
/tmp
$ go run pwd.go -P
/tmp
/private/tmp
```

# 使用 Go 开发`which(1)`实用程序

`which(1)`实用程序搜索`PATH`环境变量的值，以找出可执行文件是否存在于`PATH`变量的一个目录中。以下输出显示了`which(1)`实用程序的工作方式：

```go
$ echo $PATH
/home/mtsouk/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
$ which ls
/home/mtsouk/bin/ls
code$ which -a ls
/home/mtsouk/bin/ls
/bin/ls
```

我们的 Unix 实用程序的实现将支持 macOS 版本的`which(1)`支持的两个命令行选项`-a`和`-s`，并借助`flag`包：Linux 版本的`which(1)`不支持`-s`选项。`-a`选项列出可执行文件的所有实例，而不仅仅是第一个，而`-s`返回`0`如果找到了可执行文件，否则返回`1`：这与使用`fmt`包打印`0`或`1`不同。

为了检查 Unix 命令行实用程序在 shell 中的返回值，您应该执行以下操作：

```go
$ which -s ls $ echo $?
0
```

请注意，`go run`会打印出非零的退出代码。

`which(1)`的 Go 代码将保存在`which.go`中，并将分为四个部分呈现。`which.go`的第一部分包含以下 Go 代码：

```go
package main 

import ( 
   "flag" 
   "fmt" 
   "os" 
   "strings" 
) 
```

需要`strings`包来分割读取`PATH`变量的内容。`which.go`的第二部分处理了`flag`包的使用：

```go
func main() { 
   minusA := flag.Bool("a", false, "a") 
   minusS := flag.Bool("s", false, "s") 

   flag.Parse() 
   flags := flag.Args() 
   if len(flags) == 0 { 
         fmt.Println("Please provide an argument!") 
         os.Exit(1) 
   } 
   file := flags[0] 
   fountIt := false 
```

`which.go`的一个非常重要的部分是读取`PATH` shell 环境变量以分割并使用它的部分，这在这里的第三部分中呈现：

```go
   path := os.Getenv("PATH") 
   pathSlice := strings.Split(path, ":") 
   for _, directory := range pathSlice { 
         fullPath := directory + "/" + file 
```

这里的最后一条语句构造了我们正在搜索的文件的完整路径，就好像它存在于`PATH`变量的每个单独目录中，因为如果你有文件的完整路径，你就不必再去搜索它了！

`which.go`的最后一部分如下：

```go
         fileInfo, err := os.Stat(fullPath) 
         if err == nil { 
               mode := fileInfo.Mode() 
               if mode.IsRegular() { 
                     if mode&0111 != 0 { 
                           fountIt = true 
                           if *minusS == true { 
                                 os.Exit(0) 
                           } 
                           if *minusA == true {

                                 fmt.Println(fullPath) 
                           } else { 
                                 fmt.Println(fullPath) 
                                 os.Exit(0) 
                           } 
                     } 
               } 
         } 
   } 
   if fountIt == false { 
         os.Exit(1) 
   } 
} 
```

在这里，对`os.Stat()`的调用告诉我们正在寻找的文件是否实际存在。在成功的情况下，`mode.IsRegular()`函数检查文件是否是常规文件，因为我们不寻找目录或符号链接。但是，我们还没有完成！`which.go`程序执行了一个测试，以找出找到的文件是否确实是可执行文件：如果不是可执行文件，它将不会被打印。因此，`if mode&0111 != 0`语句使用二进制操作验证文件实际上是可执行文件。

接下来，如果`-s`标志设置为`*minusS == true`，那么`-a`标志就不太重要了，因为一旦找到匹配项，程序就会终止。

正如您所看到的，在`which.go`中涉及许多测试，这对于系统软件来说并不罕见。尽管如此，您应该始终检查所有可能性，以避免以后出现意外。好消息是，这些测试中的大多数将在`find(1)`实用程序的 Go 实现中稍后使用：通过编写小程序来测试一些功能，然后将它们全部组合成更大的程序，这是一个很好的实践，因为这样做可以更好地学习技术，并且可以更容易地检测愚蠢的错误。

执行`which.go`将产生以下输出：

```go
$ go run which.go ls
/home/mtsouk/bin/ls
$ go run which.go -s ls
$ echo $?
0
$ go run which.go -s ls123123
exit status 1
$ echo $?
1
$ go run which.go -a ls
/home/mtsouk/bin/ls
/bin/ls
```

# 打印文件或目录的权限位

借助`ls(1)`命令，您可以找出文件的权限：

```go
$ ls -l /bin/ls
-rwxr-xr-x  1 root  wheel  38624 Mar 23 01:57 /bin/ls
```

在本小节中，我们将展示如何使用 Go 打印文件或目录的权限：Go 代码将保存在`permissions.go`中，并将分为两部分呈现。第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
) 

func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide an argument!") 
         os.Exit(1) 
   } 

   file := arguments[1] 
```

第二部分包含重要的 Go 代码：

```go
   info, err := os.Stat(file) 
   if err != nil { 
         fmt.Println("Error:", err) 
         os.Exit(1) 
   } 
   mode := info.Mode() 
   fmt.Print(file, ": ", mode, "\n") 
} 
```

再次强调，大部分的 Go 代码用于处理命令行参数并确保您有一个！实际工作的 Go 代码主要是调用`os.Stat()`函数，该函数返回一个描述`os.Stat()`检查的文件或目录的`FileInfo`结构。通过`FileInfo`结构，您可以调用`Mode()`函数来发现文件的权限。

执行`permissions.go`会产生以下输出：

```go
$ go run permissions.go /bin/ls
/bin/ls: -rwxr-xr-x
$ go run permissions.go /usr
/usr: drwxr-xr-x
$ go run permissions.go /us
Error: stat /us: no such file or directory
exit status 1
```

# 在 Go 中处理文件

操作系统的一个极其重要的任务是处理文件，因为所有数据都存储在文件中。在本节中，我们将向您展示如何删除和重命名文件，在下一节*在 Go 中开发 find(1)*中，我们将教您如何搜索目录结构以找到所需的文件。

# 删除文件

在本节中，我们将说明如何使用`os.Remove()` Go 函数删除文件和目录。

在测试删除文件和目录的程序时，请格外小心并且要有常识！

`rm.go`文件是`rm(1)`工具的 Go 实现，说明了您如何在 Go 中删除文件。尽管`rm(1)`的核心功能已经存在，但缺少`rm(1)`的选项：尝试实现其中一些选项将是一个很好的练习。在实现`-f`和`-R`选项时要特别注意。

`rm.go`的 Go 代码如下：

```go
package main 
import ( 
   "fmt" 
   "os" 
) 

func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide an argument!") 
         os.Exit(1) 
   } 

   file := arguments[1] 
   err := os.Remove(file) 
   if err != nil { 
         fmt.Println(err) 
         return 
   } 
} 
```

如果`rm.go`在没有任何问题的情况下执行，将不会产生任何输出，这符合 Unix 哲学。因此，有趣的是观察当您尝试删除的文件不存在时可以获得的错误消息：当您没有必要的权限删除它时以及当目录不为空时：

```go
$ go run rm.go 123
remove 123: no such file or directory
$ ls -l /tmp/AlTest1.err
-rw-r--r--  1 root  wheel  1278 Apr 17 20:13 /tmp/AlTest1.err
$ go run rm.go /tmp/AlTest1.err
remove /tmp/AlTest1.err: permission denied
$ go run rm.go test
remove test: directory not empty
```

# 重命名和移动文件

在本小节中，我们将向您展示如何使用 Go 代码重命名和移动文件：Go 代码将保存为`rename.go`。尽管相同的代码可以用于重命名或移动目录，但`rename.go`只允许处理文件。

在执行一些无法轻易撤消的操作时，例如覆盖文件时，您应该格外小心，也许通知用户目标文件已经存在，以避免不愉快的意外。尽管传统的`mv(1)`实用程序的默认操作会自动覆盖目标文件（如果存在），但我认为这并不是很安全。因此，默认情况下，`rename.go`不会覆盖目标文件。

在开发系统软件时，您必须处理所有细节，否则这些细节将在最不经意的时候显露为错误！广泛的测试将使您能够找到您错过的细节并加以纠正。

`rename.go`的代码将分为四部分呈现。第一部分包括预期的序言以及处理`flag`包设置的 Go 代码：

```go
package main 

import ( 
   "flag" 
   "fmt" 
   "os" 
   "path/filepath" 
) 

func main() { 
   minusOverwrite := flag.Bool("overwrite", false, "overwrite") 

   flag.Parse() 
   flags := flag.Args() 

   if len(flags) < 2 { 
         fmt.Println("Please provide two arguments!") 
         os.Exit(1) 
   } 
```

第二部分包含以下 Go 代码：

```go
   source := flags[0] 
   destination := flags[1] 
   fileInfo, err := os.Stat(source) 
   if err == nil { 
         mode := fileInfo.Mode() 
         if mode.IsRegular() == false { 
               fmt.Println("Sorry, we only support regular files as source!") 
               os.Exit(1) 
         } 
   } else { 
         fmt.Println("Error reading:", source) 
         os.Exit(1) 
   } 
```

这部分确保源文件存在，是一个普通文件，并且不是一个目录或者其他类似网络套接字或管道的东西。再次，你在`which.go`中看到的`os.Stat()`的技巧在这里被使用了。

`rename.go`的第三部分如下：

```go
   newDestination := destination 
   destInfo, err := os.Stat(destination) 
   if err == nil { 
         mode := destInfo.Mode() 
         if mode.IsDir() { 
               justTheName := filepath.Base(source) 
               newDestination = destination + "/" + justTheName 
         } 
   } 
```

这里还有另一个棘手的地方；你需要考虑源文件是普通文件而目标是目录的情况，这是通过`newDestination`变量的帮助实现的。

另一个你应该考虑的特殊情况是，当源文件以包含绝对或相对路径的格式给出时，比如`./aDir/aFile`。在这种情况下，当目标是一个目录时，你应该获取路径的基本名称，即跟在最后一个`/`字符后面的内容，在这种情况下是`aFile`，并将其添加到目标目录中，以正确构造`newDestination`变量。这是通过`filepath.Base()`函数的帮助实现的，它返回路径的最后一个元素。

最后，`rename.go`的最后部分包含以下 Go 代码：

```go
   destination = newDestination 
   destInfo, err = os.Stat(destination) 
   if err == nil { 
         if *minusOverwrite == false { 
               fmt.Println("Destination file already exists!") 
               os.Exit(1) 
         } 
   } 

   err = os.Rename(source, destination) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
} 
```

`rename.go`最重要的 Go 代码与识别目标文件是否存在有关。再次，这是通过`os.Stat()`函数的支持实现的。如果`os.Stat()`返回一个错误消息，这意味着目标文件不存在；因此，你可以调用`os.Rename()`。如果`os.Stat()`返回`nil`，这意味着`os.Stat()`调用成功，并且目标文件存在。在这种情况下，你应该检查`overwrite`标志的值，以查看是否允许覆盖目标文件。

当一切正常时，你可以自由地调用`os.Rename()`并执行所需的任务！

如果`rename.go`被正确执行，它将不会产生任何输出。然而，如果有问题，`rename.go`将生成一些输出：

```go
$ touch newFILE
$ ./rename newFILE regExpFind.go
Destination file already exists!
$ ./rename -overwrite newFILE regExpFind.go
$
```

# 在 Go 中开发 find(1)

这一部分将教你开发一个简化版本的`find(1)`命令行实用程序所需的必要知识。开发的版本将不支持`find(1)`支持的所有命令行选项，但它将有足够的选项来真正有用。

在接下来的子章节中，你将看到整个过程分为小步骤。因此，第一个子章节将向你展示访问给定目录树中的所有文件和目录的 Go 方式。

# 遍历目录树

`find(1)`最重要的任务是能够访问从给定目录开始的所有文件和子目录。因此，这一部分将在 Go 中实现这个任务。`traverse.go`的 Go 代码将分为三部分呈现。第一部分是预期的序言：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "path/filepath" 
) 
```

第二部分是关于实现一个名为`walkFunction()`的函数，该函数将用作 Go 函数`filepath.Walk()`的参数：

```go
func walkFunction(path string, info os.FileInfo, err error) error { 
   _, err = os.Stat(path) 
   if err != nil { 
         return err 
   } 

   fmt.Println(path) 
   return nil 
} 
```

再次，`os.Stat()`函数被使用是因为成功的`os.Stat()`函数调用意味着我们正在处理实际存在的东西（文件、目录、管道等）！

不要忘记，在调用`filepath.Walk()`和调用执行`walkFunction()`之间，活跃和繁忙的文件系统中可能会发生许多事情，这是调用`os.Stat()`的主要原因。

代码的最后部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Not enough arguments!") 
         os.Exit(1) 
   } 

   Path := arguments[1] 
   err := filepath.Walk(Path, walkFunction) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
} 
```

所有这些繁琐的工作都是由`filepath.Walk()`函数自动完成的，借助于之前定义的`walkFunction()`函数。`filepath.Walk()`函数接受两个参数：一个目录的路径和它将使用的遍历函数。

执行`traverse.go`将生成以下类型的输出：

```go
$ go run traverse.go ~/code/C/cUNL
/home/mtsouk/code/C/cUNL
/home/mtsouk/code/C/cUNL/gpp
/home/mtsouk/code/C/cUNL/gpp.c
/home/mtsouk/code/C/cUNL/sizeofint
/home/mtsouk/code/C/cUNL/sizeofint.c
/home/mtsouk/code/C/cUNL/speed
/home/mtsouk/code/C/cUNL/speed.c
/home/mtsouk/code/C/cUNL/swap
/home/mtsouk/code/C/cUNL/swap.c
```

正如你所看到的，`traverse.go`的代码相当天真，因为它除其他事情外，无法区分目录、文件和符号链接。然而，它完成了访问给定目录树下的每个文件和目录的繁琐工作，这是`find(1)`实用程序的基本功能。

# 仅访问目录！

虽然能够访问所有内容是很好的，但有时您只想访问目录而不是文件。因此，在本小节中，我们将修改`traverse.go`以仍然访问所有内容，但只打印目录名称。新程序的名称将是`traverseDir.go`。需要更改的是`traverse.go`的唯一部分是`walkFunction()`的定义：

```go
func walkFunction(path string, info os.FileInfo, err error) error { 
   fileInfo, err := os.Stat(path) 
   if err != nil { 
         return err 
   } 

   mode := fileInfo.Mode() 
   if mode.IsDir() { 
         fmt.Println(path) 
   } 
   return nil 
} 
```

如您所见，您需要使用`os.Stat()`函数调用返回的信息来检查您是否正在处理目录。如果您有一个目录，那么打印其路径，您就完成了。

执行`traverseDir.go`将生成以下输出：

```go
$ go run traverseDir.go ~/code
/home/mtsouk/code
/home/mtsouk/code/C
/home/mtsouk/code/C/cUNL
/home/mtsouk/code/C/example
/home/mtsouk/code/C/sysProg
/home/mtsouk/code/C/system
/home/mtsouk/code/Haskell
/home/mtsouk/code/aLink
/home/mtsouk/code/perl
/home/mtsouk/code/python  
```

# find(1)的第一个版本

本节中的 Go 代码保存为`find.go`，将分为三部分呈现。正如您将看到的，`find.go`使用了在`traverse.go`中找到的大量代码，这是您逐步开发程序时获得的主要好处。

`find.go`的第一部分是预期的序言：

```go
package main 

import ( 
   "flag" 
   "fmt" 
   "os" 
   "path/filepath" 
) 
```

由于我们已经知道将来会改进`find.go`，因此即使这是`find.go`的第一个版本并且没有任何标志，这里也使用了`flag`包！

Go 代码的第二部分包含了`walkFunction()`的实现：

```go
func walkFunction(path string, info os.FileInfo, err error) error { 

   fileInfo, err := os.Stat(path) 
   if err != nil { 
         return err 
   } 

   mode := fileInfo.Mode() 
   if mode.IsDir() || mode.IsRegular() { 
         fmt.Println(path) 
   } 
   return nil 
} 
```

从`walkFunction()`的实现中，您可以轻松理解`find.go`只打印常规文件和目录，没有其他内容。这是一个问题吗？不是，如果这是您想要的。一般来说，这不是好的。尽管如此，尽管存在一些限制，但拥有一个能够工作的东西的第一个版本是一个很好的起点！下一个版本将被命名为`improvedFind.go`，将通过向其添加各种命令行选项来改进`find.go`。

`find.go`的最后一部分包含实现`main()`函数的代码：

```go
func main() { 
   flag.Parse() 
   flags := flag.Args() 

   if len(flags) == 0 { 
         fmt.Println("Not enough arguments!") 
         os.Exit(1) 
   } 

   Path := flags[0]

   err := filepath.Walk(Path, walkFunction) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
} 
```

执行`find.go`将创建以下输出：

```go
$ go run find.go ~/code/C/cUNL
/home/mtsouk/code/C/cUNL
/home/mtsouk/code/C/cUNL/gpp
/home/mtsouk/code/C/cUNL/gpp.c
/home/mtsouk/code/C/cUNL/sizeofint
/home/mtsouk/code/C/cUNL/sizeofint.c
/home/mtsouk/code/C/cUNL/speed
/home/mtsouk/code/C/cUNL/speed.c
/home/mtsouk/code/C/cUNL/swap
/home/mtsouk/code/C/cUNL/swap.c
```

# 添加一些命令行选项

本小节将尝试改进您之前创建的`find(1)`的 Go 版本。请记住，这是开发真实程序使用的过程，因为您不会在程序的第一个版本中实现每个可能的命令行选项。

新版本的 Go 代码将保存为`improvedFind.go`。新版本将能够忽略符号链接：只有在使用适当的命令行选项运行`improvedFind.go`时，才会打印符号链接。为此，我们将使用`symbolLink.go`的一些 Go 代码。

`improvedFind.go`程序是一个真正的系统工具，您可以在自己的 Unix 机器上使用。

支持的标志将是以下内容：

+   -s：这是用于打印套接字文件的

+   -p：这是用于打印管道的

+   -sl：这是用于打印符号链接的

+   -d：这是用于打印目录的

+   -f：这是用于打印文件的

正如您将看到的，大部分新的 Go 代码是为了支持添加到程序中的标志。此外，默认情况下，`improvedFind.go`打印每种类型的文件或目录，并且您可以组合任何前述标志以打印您想要的文件类型。

除了在实现`main()`函数中进行各种更改以支持所有这些标志之外，大部分其余更改将发生在`walkFunction()`函数的代码中。此外，`walkFunction()`函数将在`main()`函数内部定义，这是为了避免使用全局变量。

`improvedFind.go`的第一部分如下：

```go
package main 

import ( 
   "flag" 
   "fmt" 
   "os" 
   "path/filepath" 
) 

func main() { 

   minusS := flag.Bool("s", false, "Sockets") 
   minusP := flag.Bool("p", false, "Pipes") 
   minusSL := flag.Bool("sl", false, "Symbolic Links") 
   minusD := flag.Bool("d", false, "Directories") 
   minusF := flag.Bool("f", false, "Files") 

   flag.Parse() 
   flags := flag.Args() 

   printAll := false 
   if *minusS && *minusP && *minusSL && *minusD && *minusF { 
         printAll = true 
   } 

   if !(*minusS || *minusP || *minusSL || *minusD || *minusF) { 
         printAll = true 
   } 

   if len(flags) == 0 { 
         fmt.Println("Not enough arguments!") 
         os.Exit(1) 
   } 

   Path := flags[0] 
```

因此，如果所有标志都未设置，程序将打印所有内容，这由第一个`if`语句处理。同样，如果所有标志都设置了，程序也将打印所有内容。因此，需要一个名为`printAll`的新布尔变量。

`improvedFind.go`的第二部分包含以下 Go 代码，主要是`walkFunction`变量的定义，实际上是一个函数：

```go
   walkFunction := func(path string, info os.FileInfo, err error) error { 
         fileInfo, err := os.Stat(path) 
         if err != nil { 
               return err 
         } 

         if printAll { 
               fmt.Println(path) 
               return nil 
         } 

         mode := fileInfo.Mode() 
         if mode.IsRegular() && *minusF { 
               fmt.Println(path) 
               return nil 
         } 

         if mode.IsDir() && *minusD { 
               fmt.Println(path) 
               return nil 
         } 

         fileInfo, _ = os.Lstat(path)

         if fileInfo.Mode()&os.ModeSymlink != 0 { 
               if *minusSL { 
                     fmt.Println(path) 
                     return nil 
               } 
         } 

         if fileInfo.Mode()&os.ModeNamedPipe != 0 { 
               if *minusP { 
                     fmt.Println(path) 
                     return nil 
               } 
         } 

         if fileInfo.Mode()&os.ModeSocket != 0 { 
               if *minusS { 
                     fmt.Println(path) 
                     return nil 
               } 
         } 

         return nil 
   } 
```

在这里，好处是一旦找到匹配并打印文件，你就不必访问`if`语句的其余部分，这是将`minusF`检查放在第一位，`minusD`检查放在第二位的主要原因。调用`os.Lstat()`用于找出我们是否正在处理符号链接。这是因为`os.Stat()`会跟随符号链接并返回有关链接引用的文件的信息，而`os.Lstat()`不会这样做：`stat(2)`和`lstat(2)`也是如此。

你应该对`improvedFind.go`的最后部分非常熟悉：

```go
   err := filepath.Walk(Path, walkFunction) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
} 
```

执行`improvedFind.go`生成以下输出，这是`find.go`输出的增强版本：

```go
$ go run improvedFind.go -d ~/code/C
/home/mtsouk/code/C
/home/mtsouk/code/C/cUNL
/home/mtsouk/code/C/example
/home/mtsouk/code/C/sysProg
/home/mtsouk/code/C/system
$ go run improvedFind.go -sl ~/code
/home/mtsouk/code/aLink
```

# 从查找输出中排除文件名

有时你不需要显示`find(1)`的输出中的所有内容。因此，在这一小节中，你将学习一种技术，允许你根据文件名手动排除`improvedFind.go`的输出中的文件。

请注意，该程序的这个版本不支持正则表达式，只会排除文件名的精确匹配。

因此，`improvedFind.go`的改进版本将被命名为`excludeFind.go`。`diff(1)`实用程序的输出可以揭示`improvedFind.go`和`excludeFind.go`之间的代码差异：

```go
$ diff excludeFind.go improvedFind.go
10,19d9
< func excludeNames(name string, exclude string) bool {`
<     if exclude == "" {
<           return false
<     }
<     if filepath.Base(name) == exclude {
<           return true
<     }
<     return false
< }
<
27d16
<     minusX := flag.String("x", "", "Files")
54,57d42
<           if excludeNames(path, *minusX) {
<                 return nil
<           }
<
```

最重要的变化是引入了一个名为`excludeNames()`的新的 Go 函数，处理文件名的排除以及`-x`标志的添加，用于设置要从输出中排除的文件名。所有的工作都由文件路径完成。`Base()`函数找到路径的最后一部分，即使路径不是文件而是目录，也会将其与`-x`标志的值进行比较。

请注意，`excludeNames()`函数的更合适的名称可能是`isExcluded()`或类似的，因为`-x`选项接受单个值。

使用`excludeFind.go`执行并不带`-x`标志的命令将证明新的 Go 代码实际上是有效的。

```go
$ go run excludeFind.go -x=dT.py ~/code/python
/home/mtsouk/code/python
/home/mtsouk/code/python/dataFile.txt
/home/mtsouk/code/python/python
$ go run excludeFind.go ~/code/python
/home/mtsouk/code/python
/home/mtsouk/code/python/dT.py
/home/mtsouk/code/python/dataFile.txt
/home/mtsouk/code/python/python
```

# 从查找输出中排除文件扩展名

文件扩展名是最后一个点（`.`）字符之后的文件名的一部分。因此，`image.png`文件的文件扩展名是 png，这适用于文件和目录。

再次，为了实现这个功能，你需要一个单独的命令行选项，后面跟着你想要排除的文件扩展名：新的标志将被命名为`-ext`。这个`find(1)`实用程序的版本将基于`excludeFind.go`的代码，并将被命名为`finalFind.go`。你们中的一些人可能会说，这个选项更合适的名称应该是`-xext`，你们是对的！

再次，`diff(1)`实用程序将帮助我们发现`excludeFind.go`和`finalFind.go`之间的代码差异：新功能是在名为`excludeExtensions()`的 Go 函数中实现的，这使得理解更加容易。

```go
$ diff finalFind.go excludeFind.go
8d7
<     "strings"
21,34d19
< func excludeExtensions(name string, extension string) bool {
<     if extension == "" {
<           return false
<     }
<     basename := filepath.Base(name)
<     s := strings.Split(basename, ".")
<     length := len(s)
<     basenameExtension := s[length-1]
<     if basenameExtension == extension {
<           return true
<     }
<     return false
< }
<
43d27
<     minusEXT := flag.String("ext", "", "Extensions")
74,77d57
<           if excludeExtensions(path, *minusEXT) {
<                 return nil
<           }
< 
```

由于我们正在寻找路径中最后一个点后的字符串，我们使用`strings.Split()`根据路径中包含的点字符来分割路径。然后，我们取`strings.Split()`的返回值的最后一部分，并将其与使用`-ext`标志给定的扩展名进行比较。因此，这里没有什么特别的，只是一些字符串操作代码。再次强调，`excludeExtensions()`更合适的名称应该是`isExcludedExtension()`。

执行`finalFind.go`将生成以下输出：

```go
$ go run finalFind.go -ext=py ~/code/python
/home/mtsouk/code/python
/home/mtsouk/code/python/dataFile.txt
/home/mtsouk/code/python/python
$ go run finalFind.go ~/code/python
/home/mtsouk/code/python
/home/mtsouk/code/python/dT.py
/home/mtsouk/code/python/dataFile.txt
/home/mtsouk/code/python/python
```

# 使用正则表达式

这一部分将说明如何在`finalFind.go`中添加对正则表达式的支持：工具的最新版本的名称将是`regExpFind.go`。新的标志将被称为`-re`，它将需要一个字符串值：与此字符串值匹配的任何内容都将包含在输出中，除非它被另一个命令行选项排除。此外，由于标志提供的灵活性，我们不需要删除任何以前的选项来添加另一个选项！

再次，`diff(1)`命令将告诉我们`regExpFind.go`和`finalFind.go`之间的代码差异：

```go
$ diff regExpFind.go finalFind.go
8d7
<     "regexp"
36,44d34
< func regularExpression(path, regExp string) bool {
<     if regExp == "" {
<           return true
<     }
<     r, _ := regexp.Compile(regExp)
<     matched := r.MatchString(path)
<     return matched
< }
<
54d43
<     minusRE := flag.String("re", "", "Regular Expression")
71a61
>
75,78d64
<           if regularExpression(path, *minusRE) == false {
<                 return nil
<           }
< 
```

在第七章*,* *处理系统文件*中，我们将更多地讨论 Go 中的模式匹配和正则表达式：现在，理解`regexp.Compile()`创建正则表达式，`MatchString()`尝试在`regularExpression()`函数中进行匹配就足够了。

执行`regExpFind.go`将生成以下输出：

```go
$ go run regExpFind.go -re=anotherPackage /Users/mtsouk/go
/Users/mtsouk/go/pkg/darwin_amd64/anotherPackage.a
/Users/mtsouk/go/src/anotherPackage
/Users/mtsouk/go/src/anotherPackage/anotherPackage.go
$ go run regExpFind.go -ext=go -re=anotherPackage /Users/mtsouk/go
/Users/mtsouk/go/pkg/darwin_amd64/anotherPackage.a
/Users/mtsouk/go/src/anotherPackage 
```

可以使用以下命令验证先前的输出：

```go
$ go run regExpFind.go /Users/mtsouk/go | grep anotherPackage
/Users/mtsouk/go/pkg/darwin_amd64/anotherPackage.a
/Users/mtsouk/go/src/anotherPackage
/Users/mtsouk/go/src/anotherPackage/anotherPackage.go
```

# 创建目录结构的副本

凭借您在前几节中获得的知识，我们现在将开发一个 Go 程序，该程序在另一个目录中创建目录结构的副本：这意味着输入目录中的任何文件都不会复制到目标目录，只会复制目录。当您想要将有用的文件从一个目录结构保存到其他位置并保持相同的目录结构时，或者当您想要手动备份文件系统时，这可能会很方便。

由于您只对目录感兴趣，因此`cpStructure.go`的代码基于本章前面看到的`traverseDir.go`的代码：再次，为了学习目的而开发的小程序帮助您实现更大的程序！此外，`test`选项将显示程序的操作，而不会实际创建任何目录。

`cpStructure.go`的代码将分为四部分呈现。第一部分如下：

```go
package main 

import ( 
   "flag" 
   "fmt" 
   "os" 
   "path/filepath" 
   "strings" 
) 
```

这里没有什么特别的，只是预期的序言。第二部分如下：

```go
func main() { 
   minusTEST := flag.Bool("test", false, "Test run!") 

   flag.Parse() 
   flags := flag.Args() 

   if len(flags) == 0 || len(flags) == 1 { 
         fmt.Println("Not enough arguments!") 
         os.Exit(1) 
   } 

   Path := flags[0] 
   NewPath := flags[1] 

   permissions := os.ModePerm 
   _, err := os.Stat(NewPath) 
   if os.IsNotExist(err) { 
         os.MkdirAll(NewPath, permissions) 
   } else { 
         fmt.Println(NewPath, "already exists - quitting...") 
         os.Exit(1) 
   } 
```

`cpStructure.go`程序要求预先不存在目标目录，以避免后续不必要的意外和错误。

第三部分包含`walkFunction`变量的代码：

```go
   walkFunction := func(currentPath string, info os.FileInfo, err error) error { 
         fileInfo, _ := os.Lstat(currentPath) 
         if fileInfo.Mode()&os.ModeSymlink != 0 { 
               fmt.Println("Skipping", currentPath) 
               return nil 
         } 

         fileInfo, err = os.Stat(currentPath) 
         if err != nil { 
               fmt.Println("*", err) 
               return err 
         } 

         mode := fileInfo.Mode() 
         if mode.IsDir() { 
               tempPath := strings.Replace(currentPath, Path, "", 1) 
               pathToCreate := NewPath + "/" + filepath.Base(Path) + tempPath 

               if *minusTEST { 
                     fmt.Println(":", pathToCreate) 
                     return nil 
               } 

               _, err := os.Stat(pathToCreate) 
               if os.IsNotExist(err) { 
                     os.MkdirAll(pathToCreate, permissions) 
               } else { 
                     fmt.Println("Did not create", pathToCreate, ":", err) 
               } 
         } 
         return nil 
   } 
```

在这里，第一个`if`语句确保我们将处理符号链接，因为符号链接可能很危险并且会造成问题：始终尝试处理特殊情况以避免问题和令人讨厌的错误。

`os.IsNotExist()`函数允许您确保您要创建的目录尚不存在。因此，如果目录不存在，您可以使用`os.MkdirAll()`创建它。`os.MkdirAll()`函数创建包括所有必要父目录的目录路径，这对开发人员来说更简单。

然而，`walkFunction`变量的代码必须处理的最棘手的部分是删除源路径的不必要部分并正确构造新路径。程序中使用的`strings.Replace()`函数用其第二个参数(`Path`)替换其第一个参数(`currentPath`)中可以找到的出现，用其第三个参数(`""`)替换其最后一个参数(`1`)的次数。如果最后一个参数是负数，这里不是这种情况，那么将没有限制地进行替换。在这种情况下，它会从`currentPath`变量（正在检查的目录）中删除`Path`变量的值，这是源目录。

程序的最后部分如下：

```go
   err = filepath.Walk(Path, walkFunction) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(1) 
   } 
} 
```

执行`cpStructure.go`将生成以下输出：

```go
$ go run cpStructure.go ~/code /tmp/newCode
Skipping /home/mtsouk/code/aLink
$ ls -l /home/mtsouk/code/aLink
lrwxrwxrwx 1 mtsouk mtsouk 14 Apr 21 18:10 /home/mtsouk/code/aLink -> /usr/local/bin 
```

以下图显示了前述示例中使用的源目录和目标目录结构的图形表示：

![](img/ecf2e299-f496-48f4-9622-d1225bd52ad4.png)

两个目录结构及其文件的图形表示

# 练习

1.  阅读[`golang.org/pkg/os/`](https://golang.org/pkg/os/)上的`os`包的文档页面。

1.  访问[`golang.org/pkg/path/filepath/`](https://golang.org/pkg/path/filepath/)了解更多关于`filepath.Walk()`函数的信息。

1.  更改`rm.go`的代码以支持多个命令行参数，然后尝试实现`rm(1)`实用程序的`-v`命令行选项。

1.  对`which.go`的 Go 代码进行必要的更改，以支持多个命令行参数。

1.  开始在 Go 中实现`ls(1)`实用程序的版本。不要一次性尝试支持每个`ls(1)`选项。

1.  修改`traverseDir.go`的代码，以便只打印常规文件。

1.  查看`find(1)`的手册页面，并尝试在`regExpFind.go`中添加对其某些选项的支持。

# 摘要

在本章中，我们讨论了许多内容，包括使用`flag`标准包，允许您使用目录和文件以及遍历目录结构的 Go 函数，并且我们开发了各种 Unix 命令行实用程序的 Go 版本，包括`pwd(1)`、`which(1)`、`rm(1)`和`find(1)`。

在下一章中，我们将继续讨论文件操作，但这次您将学习如何在 Go 中读取文件和写入文件：正如您将看到的，有许多方法可以做到这一点。虽然这给了您灵活性，但它也要求您能够选择尽可能高效地完成工作的正确技术！因此，您将首先学习更多关于`io`包以及`bufio`包，到本章结束时，您将拥有`wc(1)`和`dd(1)`实用程序的 Go 版本！
