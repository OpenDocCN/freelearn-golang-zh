# 第二章：在 Go 中编写程序

本章将讨论许多重要、有趣和实用的 Go 主题，这将帮助您更加高效。我认为从编译和运行上一章的`hw.go`程序的 Go 代码开始本章是一个不错的主意。然后，您将学习如何处理 Go 可以使用的环境变量，如何处理 Go 程序的命令行参数，以及如何在屏幕上打印输出并从用户那里获取输入。最后，您将了解如何在 Go 中定义函数，学习极其重要的`defer`关键字，查看 Go 提供的数据结构，并了解 Go 接口，然后再查看生成随机数的代码。

因此，在本章中，您将熟悉许多 Go 概念，包括以下内容：

+   编译您的 Go 程序

+   Go 环境变量

+   使用传递给 Go 程序的命令行参数

+   获取用户输入并在屏幕上打印输出

+   Go 函数和`defer`关键字

+   Go 数据结构和接口

+   生成随机数

# 编译 Go 代码

只要包名是`main`并且其中有`main()`函数，Go 就不在乎一个独立程序的源文件的名称。这是因为`main()`函数是程序执行的起点。这也意味着在单个项目的文件中不能有多个`main()`函数。

有两种运行 Go 程序的方式：

+   第一个是`go run`，只是执行 Go 代码而不生成任何新文件，只会生成一些临时文件，之后会被删除

+   第二种方式，`go build`，编译代码，生成可执行文件，并等待您运行可执行文件

本书是在使用 Homebrew ([`brew.sh/`](https://brew.sh/))版本的 Go 的 Apple Mac OS Sierra 系统上编写的。但是，只要您有一个相对较新的 Go 版本，您应该不会在大多数 Linux 和 FreeBSD 系统上编译和运行所提供的 Go 代码时遇到困难。

因此，第一种方式如下：

```go
$ go run hw.go
Hello World!  
```

上述方式允许 Go 用作脚本语言。以下是第二种方式：

```go
$ go build hw.go
$ file hw
hw: Mach-O 64-bit executable x86_64
```

生成的可执行文件以 Go 源文件的名称命名，这比`a.out`要好得多，后者是 C 编译器生成的可执行文件的默认文件名。

如果您的代码中有错误，比如在调用 Go 函数时拼错了 Go 包名，您将会得到以下类型的错误消息：

```go
$ go run hw.go
# command-line-arguments
./hw.go:3: imported and not used: "fmt"
./hw.go:7: undefined: mt in mt.Println
```

如果您意外地拼错了`main()`函数，您将会得到以下错误消息，因为独立的 Go 程序的执行是从`main()`函数开始的：

```go
$ go run hw.go
# command-line-arguments
runtime.main_main f: relocation target main.main not defined
runtime.main_main f: undefined: "main.main"
```

最后，我想向您展示一个错误消息，它将让您对 Go 的格式规则有一个很好的了解：

```go
$ cat hw.gocat 
package main

import "fmt"

func main()
{
      fmt.Println("Hello World!")
}
$ go run hw.go
# command-line-arguments
./hw.go:6: syntax error: unexpected semicolon or newline before {

```

前面的错误消息告诉我们，Go 更喜欢以一种特定的方式放置大括号，这与大多数编程语言（如 Perl、C 和 C++）不同。这一开始可能看起来令人沮丧，但它可以节省您一行额外的代码，并使您的程序更易读。请注意，前面的代码使用了*Allman 格式样式*，而 Go 不接受这种格式。

对于这个错误的官方解释是，Go 在许多情况下要求使用分号作为语句终止符，并且编译器会在它认为必要时自动插入所需的分号，这种情况是在非空行的末尾。因此，将开括号（`{`）放在自己的一行上会让 Go 编译器在前一行末尾加上一个分号，从而产生错误消息。

如果您认为`gofmt`工具可以帮您避免类似的错误，您将会感到失望：

```go
$ gofmt hw.go
hw.go:6:1: expected declaration, found '{'

```

正如您在以下输出中所看到的，Go 编译器还有另一条规则：

```go
$ go run afile.go
# command-line-arguments
./afile.go:4: imported and not used: "net"
```

这意味着你不应该在程序中导入包而不实际使用它们。虽然这可能是一个无害的警告消息，但你的 Go 程序将无法编译。请记住，类似的警告和错误消息是你遗漏了某些东西的一个很好的指示，你应该尝试纠正它们。如果你对警告和错误采取相同的态度，你将创建更高质量的代码。

# 检查可执行文件的大小

因此，在成功编译`hw.go`之后，你可能想要检查生成的可执行文件的大小：

```go
$ ls -l hw
-rwxr-xr-x  1 mtsouk  staff  1628192 Feb  9 22:29 hw
$ file hw
hw: Mach-O 64-bit executable x86_64  
```

在 Linux 机器上编译相同的 Go 程序将创建以下文件：

```go
$ go versiongo 
go version go1.3.3 linux/amd64
$ go build hw.go
$ ls -l hw
-rwxr-xr-x 1 mtsouk mtsouk 1823712 Feb 18 17:35 hw
$ file hw
hw: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped

```

为了更好地了解 Go 可执行文件的大小，考虑一下，同样的程序用 C 编写的可执行文件大约为 8432 字节！

因此，你可能会问为什么一个如此小的程序会生成一个如此庞大的可执行文件？主要原因是 Go 可执行文件是静态构建的，这意味着它们不需要外部库来运行。使用`strip(1)`命令可以使生成的可执行文件稍微变小，但不要期望奇迹发生：

```go
$ strip hw
$ ls -l hw
-rwxr-xr-x  1 mtsouk  staff  1540096 Feb 18 17:41 hw
```

前面的过程与 Go 本身无关，因为`strip(1)`是一个 Unix 命令，它删除或修改文件的符号表，从而减小它们的大小。Go 可以自行执行`strip(1)`命令的工作并创建更小的可执行文件，但这种方法并不总是有效：

```go
$ ls -l hw
-rwxr-xr-x 1 mtsouk mtsouk 1823712 Feb 18 17:35 hw
$ CGO_ENABLED=0 go build -ldflags "-s" -a hw.go
$ ls -l hw
-rwxr-xr-x 1 mtsouk mtsouk 1328032 Feb 18 17:44 hw
$ file hw
hw: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

上述输出来自 Linux 机器；当在 macOS 机器上使用相同的编译命令时，对可执行文件的大小不会有任何影响。

# Go 环境变量

`go tool`可以使用许多专门用于 Go 的 Unix shell 环境变量，包括`GOROOT`、`GOHOME`、`GOBIN`和`GOPATH`。最重要的 Go 环境变量是`GOPATH`，它指定了你的工作空间的位置。通常，这是你在开发 Go 代码时需要定义的唯一环境变量；它与项目文件的组织方式有关。这意味着每个项目将被组织成三个主要目录，名为`src`、`pkg`和`bin`。然而，包括我在内的许多人更喜欢不使用`GOPATH`，而是手动组织他们的项目文件。

因此，如果你是 shell 变量的忠实粉丝，你可以将所有这些定义放在`.bashrc`或`.profile`中，这意味着这些环境变量将在每次登录到 Unix 机器时都处于活动状态。如果你没有使用 Bash shell（默认的 Linux 和 macOS shell），那么你可能需要使用另一个启动文件。查看你喜欢的 Unix shell 的文档，找出要使用哪个文件。

下面的截图显示了以下命令的部分输出，该命令显示了 Go 使用的所有环境变量：

```go
$ go help environment
```

![](img/87508a77-cb59-4e6f-ac23-d4a412f73ebc.png)

“go help environment”命令的输出

你可以通过执行下一个命令并将`NAME`替换为你感兴趣的环境变量来找到关于特定环境变量的额外信息：

```go
$ go env NAME  
```

所有这些环境变量与实际的 Go 代码或程序的执行无关，但它们可能会影响开发环境；因此，如果在尝试编译 Go 程序时遇到任何奇怪的行为，检查你正在使用的环境变量。

# 使用命令行参数

命令行参数允许你的程序获取输入，比如你想要处理的文件的名称，而不必编写程序的不同版本。因此，如果你无法处理传递给它的命令行参数，你将无法创建任何有用的系统软件。

这里有一个天真的 Go 程序，名为`cla.go`，它打印出所有的命令行参数，包括可执行文件的名称：

```go
package main 

import "fmt" 
import "os" 

func main() { 
   arguments := os.Args 
   for i := 0; i < len(arguments); i++ { 
         fmt.Println(arguments[i]) 
   } 
} 
```

正如您所看到的，Go 需要一个名为`os`的额外包，以便读取存储在`os.Args`数组中的程序的命令行参数。如果您不喜欢有多个导入语句，您可以将两个导入语句重写如下，我觉得这样更容易阅读：

```go
import ( 
   "fmt" 
   "os" 
)
```

当您使用单个导入块导入所有包时，`gofmt`实用程序会按字母顺序排列包名。

`cla.go`的 Go 代码很简单，它将所有命令行参数存储在一个数组中，并使用`for`循环进行打印。正如您将在接下来的章节中看到的，`os`包可以做更多的事情。如果您熟悉 C 语言，您应该知道在 C 中，命令行参数会自动传递给程序，而您无需包含任何额外的头文件来读取它们。Go 使用了一种不同的方法，这样可以给您更多的控制，但需要稍微更多的代码。

在构建后执行`cla.go`将创建以下类型的输出：

```go
$ ./cla 1 2 three
./cla
1
2
three
```

# 找到命令行参数的总和

现在，让我们尝试一些不同和棘手的事情：您将尝试找到给定给 Go 程序的命令行参数的总和。因此，您将把命令行参数视为数字。尽管主要思想保持不变，但实现完全不同，因为您将不得不将命令行参数转换为数字。Go 程序的名称将是`addCLA.go`，它可以分为两部分。

第一部分是程序的序言：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "strconv" 
) 
```

您需要`fmt`包来打印输出和`os`包来读取命令行参数。由于命令行参数存储为字符串，您还需要`srtconv`包将其转换为整数。

第二部分是`main()`函数的实现：

```go
func main() { 
   arguments := os.Args 
   sum := 0 
   for i := 1; i < len(arguments); i++ { 
         temp, _ := strconv.Atoi(arguments[i]) 
         sum = sum + temp 
   } 
   fmt.Println("Sum:", sum) 
} 
```

`strconv.Atoi()`函数返回两个值：第一个是整数，前提是转换成功，第二个是错误变量。

请注意，大多数 Go 函数都会返回一个错误变量，这个错误变量应该始终被检查，特别是在生产软件中。

如果您不使用`strconv.Atoi()`函数，那么您将会遇到两个问题：

+   第一个问题是程序将尝试使用字符串执行加法，这是数学运算。

+   第二个问题是您将无法判断命令行参数是否是有效的整数，这可以通过检查`strconv.Atoi()`的返回值来完成

因此，`strconv.Atoi()`不仅可以完成所需的工作，而且还可以告诉我们给定参数是否是有效的整数，这同样重要，因为它允许我们以不同的方式处理不合适的参数。

`addCLA.go`中的另一个关键的 Go 代码是忽略`strconv.Atoi()`函数的错误变量的值，使用模式匹配。`_`字符在 Go 模式匹配术语中表示“匹配所有”，但不要将其保存在任何变量中。

Go 支持四种不同大小的有符号和无符号整数，分别命名为 int8、int16、int32、int64、uint8、uint16、uint32 和 uint64。然而，Go 还有`int`和`uint`，它们是当前平台上最有效的有符号和无符号整数。因此，当有疑问时，请使用`int`或`uint`。

使用正确类型的命令行参数执行`addCLA.go`将创建以下输出：

```go
$ go run addCLA.go 1 2 -1 -3
Sum: -1
$ go run addCLA.go
Sum: 0
```

`addCLA.go`的好处是，如果没有参数，它不会崩溃，而无需您担心。然而，看到程序如何处理错误输入会更有趣，因为您永远不能假设会得到正确类型的输入：

```go
$ go run addCLA.go !
Sum: 0
$ go run addCLA.go ! -@
Sum: 0
$ go run addCLA.go ! -@ 1 2
Sum: 3
```

正如您所看到的，如果程序得到错误类型的输入，它不会崩溃，并且不会在其计算中包含错误的输入。这里的一个主要问题是`addCLA.go`不会打印任何警告消息，以让用户知道它们的某些输入被忽略。这种危险的代码会创建不稳定的可执行文件，当给出错误类型的输入时可能会产生安全问题。因此，这里的一般建议是，您永远不应该期望或依赖 Go 编译器，或任何其他编译器或程序，来处理这些事情，因为这是您的工作。

第三章《高级 Go 功能》将更详细地讨论 Go 中的错误处理，并将介绍上一个程序的更好和更安全的版本。目前，我们都应该高兴地证明我们的程序不会因任何输入而崩溃。

尽管这并不是一个完美的情况，但如果您知道您的程序对某些特定类型的输入不起作用，那也不是那么糟糕。糟糕的是，当开发人员不知道存在某些类型的输入可能会导致程序失败时，因为您无法纠正您不相信或认为是错误的东西。

尽管处理命令行参数看起来很容易，但如果您的命令行实用程序支持大量选项和参数，它可能会变得非常复杂。第五章《文件和目录》将更多地讨论使用`flag`标准 Go 包处理命令行选项、参数和参数。

# 用户输入和输出

根据 Unix 哲学，当程序成功完成其工作时，它不会生成任何输出。然而，出于许多原因，并非所有程序都能成功完成，并且它们需要通过打印适当的消息来通知用户其问题。此外，一些系统工具需要从用户那里获取输入，以决定如何处理可能出现的情况。

Go 用户输入和输出的英雄是`fmt`包，本节将向您展示如何通过从最简单的任务开始来执行这两个任务。

了解有关`fmt`包的更多信息的最佳位置是其文档页面，该页面可以在[`golang.org/pkg/fmt/`](https://golang.org/pkg/fmt/)找到。

# 获取用户输入

除了使用命令行参数来获取用户输入（这是系统编程中的首选方法），还有其他方法可以要求用户输入。

当使用`-i`选项时，两个示例是`rm(1)`和`mv(1)`命令：

```go
$ touch aFile
$ rm -i aFile
remove aFile? y
$ touch aFile
$ touch ../aFile
$ mv -i ../aFile .
overwrite ./aFile? (y/n [n]) y
```

因此，本节将向您展示如何在您的 Go 代码中模仿先前的行为，使您的程序能够理解`-i`参数，而不实际实现`rm(1)`或`mv(1)`的功能。

用于获取用户输入的最简单函数称为`fmt.Scanln()`，并读取整行。其他用于获取用户输入的函数包括`fmt.Scan()`、`fmt.Scanf()`、`fmt.Sscanf()`、`fmt.Sscanln()`和`fmt.Sscan()`。

然而，在 Go 中存在一种更高级的方式来从用户那里获取输入；它涉及使用`bufio`包。然而，使用`bufio`包从用户那里获取简单的响应有点过度。

`parameter.go`的 Go 代码如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "strings" 
) 

func main() { 
   arguments := os.Args 
   minusI := false 
   for i := 0; i < len(arguments); i++ { 
         if strings.Compare(arguments[i], "-i") == 0 { 
               minusI = true 
               break 
         } 
   } 

   if minusI { 
         fmt.Println("Got the -i parameter!") 
         fmt.Print("y/n: ") 
         var answer string 
         fmt.Scanln(&answer) 
         fmt.Println("You entered:", answer) 
   } else { 
         fmt.Println("The -i parameter is not set") 
   } 
} 
```

所呈现的代码并不特别聪明。它只是使用`for`循环访问所有命令行参数，并检查当前参数是否等于`-i`字符串。一旦它通过`strings.Compare()`函数找到匹配，它就会将`minusI`变量的值从 false 更改为 true。然后，因为它不需要再查找，它使用`break`语句退出`for`循环。如果给出了`-i`参数，带有`if`语句的块将要求用户使用`fmt.Scanln()`函数输入`y`或`n`。

请注意，`fmt.Scanln()` 函数使用了指向 `answer` 变量的指针。由于 Go 通过值传递变量，我们必须在这里使用指针引用，以便将用户输入保存到 `answer` 变量中。一般来说，从用户读取数据的函数往往是这样工作的。

执行 `parameter.go` 会产生以下类型的输出：

```go
$ go run parameter.go
The -i parameter is not set
$ go run parameter.go -i
Got the -i parameter!
y/n: y
You entered: y
```

# 打印输出

在 Go 中打印东西的最简单方法是使用 `fmt.Println()` 和 `fmt.Printf()` 函数。`fmt.Printf()` 函数与 C 的 `printf(3)` 函数有许多相似之处。你也可以使用 `fmt.Print()` 函数来代替 `fmt.Println()`。

`fmt.Print()` 和 `fmt.Println()` 之间的主要区别是，后者每次调用时自动打印一个换行符。`fmt.Println()` 和 `fmt.Printf()` 之间的最大区别是，后者需要为它将打印的每样东西提供一个格式说明符，就像 C 的 `printf(3)` 函数一样。这意味着你可以更好地控制你在做什么，但你需要写更多的代码。Go 将这些说明符称为**动词**，你可以在 [`golang.org/pkg/fmt/`](https://golang.org/pkg/fmt/) 找到更多关于支持的动词的信息。

# Go 函数

函数是每种编程语言的重要元素，因为它们允许你将大型程序分解为更小更易管理的部分，但它们必须尽可能独立，并且只能完成一项任务。因此，如果你发现自己编写了多个任务的函数，可能需要考虑编写多个函数。然而，Go 不会拒绝编译长、复杂或者做多个任务的函数。

一个安全的指示，你需要创建一个新函数的时候是，当你发现自己在程序中多次使用相同的 Go 代码。同样，一个安全的指示，你需要将一些函数放在一个模块中的时候是，当你发现自己在大多数程序中一直使用相同的函数。

最受欢迎的 Go 函数是 `main()`，它可以在每个独立的 Go 程序中找到。如果你看一下 `main()` 函数的定义，你很快就会意识到 Go 中的函数声明以 `func` 关键字开头。

一般来说，你必须尽量编写少于 20-30 行 Go 代码的函数。拥有更小的函数的一个好的副作用是，它们可以更容易地进行优化，因为你可以清楚地找出瓶颈在哪里。

# 给 Go 函数的返回值命名

与 C 不同，Go 允许你给函数的返回值命名。此外，当这样的函数有一个没有参数的返回语句时，函数会自动返回每个命名返回值的当前值。请注意，这样的函数按照它们在函数定义中声明的顺序返回它们的值。

给返回值命名是一个非常方便的 Go 特性，可以帮助你避免各种类型的错误，所以要使用它。

我的个人建议是：给你的函数的返回值命名，除非有非常好的理由不这样做。

# 匿名函数

匿名函数可以在一行内定义，无需名称，它们通常用于实现需要少量代码的事物。在 Go 中，一个函数可以返回一个匿名函数，或者将一个匿名函数作为其参数之一。此外，匿名函数可以附加到 Go 变量上。

对于匿名函数来说，最好的做法是有一个小的实现和局部使用。如果一个匿名函数没有局部使用，那么你可能需要考虑将其变成一个常规函数。

当匿名函数适合一项任务时，它非常方便，可以让你的生活更轻松；只是不要在程序中没有充分理由的情况下使用太多匿名函数。

# 说明 Go 函数

本小节将展示使用`functions.go`程序的 Go 代码来演示前面类型的函数的示例。程序的第一部分包含了预期的序言和`unnamedMinMax()`函数的实现：

```go
package main 

import ( 
   "fmt" 
) 

func unnamedMinMax(x, y int) (int, int) { 
   if x > y { 
         min := y 
         max := x 
         return min, max 
   } else { 
         min := x 
         max := y 
         return min, max 
   } 
} 
```

`unnamedMinMax()`函数是一个常规函数，它以两个整数作为输入，分别命名为`x`和`y`。它使用`return`语句返回两个整数。

`functions.go`的下一部分定义了另一个函数，但这次使用了命名返回值，它们被称为`min`和`max`：

```go
func minMax(x, y int) (min, max int) { 
   if x > y { 
         min = y 
         max = x 
   } else { 
         min = x 
         max = y 
   } 
   return min, max 
} 
```

下一个函数是`minMax()`的改进版本，因为你不必显式定义返回语句的返回变量：

```go
func namedMinMax(x, y int) (min, max int) { 
   if x > y { 
         min = y 
         max = x 
   } else { 
         min = x 
         max = y 
   } 
   return 
} 
```

然而，你可以通过查看`namedMinMax()`函数的定义轻松地发现将返回哪些值。`namedMinMax()`函数将以此顺序返回`min`和`max`的当前值。

下一个函数展示了如何对两个整数进行排序，而不必使用临时变量：

```go
func sort(x, y int) (int, int) { 
   if x > y { 
         return x, y 
   } else { 
         return y, x 
   } 
} 
```

前面的代码还展示了 Go 函数可以返回多个值的便利之处。`functions.go`的最后一部分包含了`main()`函数；这可以分为两部分来解释。

第一部分涉及匿名函数：

```go
 func main() {
   y := 4 
   square := func(s int) int { 
         return s * s 
   } 
   fmt.Println("The square of", y, "is", square(y)) 

   square = func(s int) int { 
         return s + s 
   } 
   fmt.Println("The square of", y, "is", square(y)) 
```

在这里，你定义了两个匿名函数：第一个计算给定整数的平方，而第二个则是给定整数的两倍。重要的是，它们都分配给了同一个变量，这是完全错误的，也是一种危险的做法。因此，不正确使用匿名函数可能会产生严重的错误，所以要格外小心，不要将同一个变量分配给不同的匿名函数。

请注意，即使将函数分配给变量，它仍然被视为匿名函数。

`main()`的第二部分使用了一些已定义的函数：

```go
   fmt.Println(minMax(15, 6)) 
   fmt.Println(namedMinMax(15, 6)) 
   min, max := namedMinMax(12, -1) 
   fmt.Println(min, max) 
} 
```

有趣的是，你可以使用两个变量在一个语句中获取`namedMinMax()`函数的两个返回值。

执行`functions.go`生成以下输出：

```go
$ go run functions.go
The square of 4 is 16
The square of 4 is 8
6 15
6 15
-1 12
```

下一部分展示了更多匿名函数与`defer`关键字结合的例子。

# defer 关键字

`defer`关键字推迟了函数的执行，直到包围函数返回，并且在文件 I/O 操作中被广泛使用。这是因为它可以让你不必记住何时关闭打开的文件。

展示`defer`的 Go 代码文件名为`defer.go`，包含四个主要部分。

第一部分是预期的序言，以及`a1()`函数的定义：

```go
package main 

import ( 
   "fmt" 
) 

func a1() { 
   for i := 0; i < 3; i++ { 
         defer fmt.Print(i, " ") 
   } 
} 
```

在前面的例子中，`defer`关键字与简单的`fmt.Print()`语句一起使用。

第二部分是`a2()`函数的定义：

```go
func a2() { 
   for i := 0; i < 3; i++ { 
         defer func() { fmt.Print(i, " ") }() 
   } 
} 
```

在`defer`关键字之后，有一个未附加到变量的匿名函数，这意味着在`for`循环终止后，匿名函数将自动消失。所呈现的匿名函数不带参数，但在`fmt.Print()`语句中使用了`i`局部变量。

下一部分定义了`a3()`函数，并包含以下 Go 代码：

```go
func a3() { 
   for i := 0; i < 3; i++ { 
         defer func(n int) { fmt.Print(n, " ") }(i) 
   } 
} 
```

这次，匿名函数需要一个名为`n`的整数参数，并从变量`i`中取其值。

`defer.go`的最后一部分是`main()`函数的实现：

```go
func main() { 
   a1() 
   fmt.Println() 
   a2() 
   fmt.Println() 
   a3() 
   fmt.Println() 
} 
```

执行`defer.go`将打印以下内容，这可能会让你感到惊讶：

```go
$ go run defer.go
2 1 0
3 3 3
2 1 0
```

因此，现在是时候通过检查`a1()`、`a2()`和`a3()`执行其代码的方式来解释`defer.go`的输出。输出的第一行验证了在包围函数返回后，延迟函数以**后进先出**（**LIFO**）的顺序执行。`a1()`中的`for`循环延迟了一个使用`i`变量当前值的函数调用。结果，所有数字都以相反的顺序打印，因为`i`的最后使用值是`2`。`a2()`函数比较棘手，因为由于`defer`，函数体在`for`循环结束后被评估，而它仍在引用局部`i`变量，这时对于所有评估的函数体来说，`i`变量的值都等于`3`。结果，`a2()`打印数字`3`三次。简而言之，您有三个使用变量的最后值的函数调用，因为这是传递给函数的内容。但是，`a3()`函数不是这种情况，因为`i`的当前值作为参数传递给延迟的函数，这是由`a3()`函数定义末尾的`(i)`代码决定的。因此，每次执行延迟的函数时，它都有一个不同的`i`值要处理。

由于使用`defer`可能会很复杂，您应该编写自己的示例，并在执行实际的 Go 代码之前尝试猜测它们的输出，以确保您的程序表现如预期。尝试能够判断函数参数何时被评估以及函数体何时实际执行。

您将在第六章 *文件输入和输出*中再次看到`defer`关键字的作用。

# 在函数中使用指针变量

**指针**是内存地址，以提高速度为代价，但代码难以调试且容易出现错误。C 程序员对此了解更多。在 Go 函数中使用指针变量的示例在`pointers.go`文件中进行了说明，可以分为两个主要部分。第一部分包含两个函数的定义和一个名为`complex`的新结构。

```go
func withPointer(x *int) { 
   *x = *x * *x 
} 

type complex struct { 
   x, y int 
} 

func newComplex(x, y int) *complex { 
   return &complex{x, y} 
} 
```

第二部分说明了在`main()`函数中使用先前定义的内容：

```go
func main() { 
   x := -2 
   withPointer(&x) 
   fmt.Println(x) 

   w := newComplex(4, -5) 
   fmt.Println(*w) 
   fmt.Println(w) 
} 
```

由于`withPointer()`函数使用指针变量，您不需要返回任何值，因为对传递给函数的变量的任何更改都会自动存储在传递的变量中。请注意，您需要在变量名前面加上`&`，以便将其作为指针而不是作为值传递。`complex`结构有两个成员，名为`x`和`y`，它们都是整数变量。

另一方面，`newComplex()`函数返回了一个指向先前在`pointers.go`中定义的`complex`结构的指针，需要存储在一个变量中。为了打印`newComplex()`函数返回的复杂变量的内容，您需要在其前面加上一个`*`字符。

执行`pointers.go`会生成以下输出：

```go
$ go run pointers.go
4
{4 -5}
&{4 -5} 
```

我不建议业余程序员在使用库所需之外使用指针，因为它们可能会引起问题。然而，随着经验的增加，您可能希望尝试使用指针，并根据您尝试解决的问题决定是否使用它们。

# Go 数据结构

Go 带有许多方便的**数据结构**，可以帮助您存储自己的数据，包括数组、切片和映射。您应该能够在任何数据结构上执行的最重要的任务是以某种方式访问其所有元素。第二个重要任务是在知道其索引或键后直接访问特定元素。最后两个同样重要的任务是向数据结构中插入元素和删除元素。一旦您知道如何执行这四个任务，您将完全控制数据结构。

# 数组

由于其速度快，并且几乎所有编程语言都支持，数组是最受欢迎的数据结构。您可以在 Go 中声明数组如下：

```go
myArray := [4]int{1, 2, 4, -4} 
```

如果您希望声明具有两个或三个维度的数组，可以使用以下表示法：

```go
twoD := [3][3]int{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}} 
threeD := [2][2][2]int{{{1, 2}, {3, 4}}, {{5, 6}, {7, 8}}} 
```

数组每个维度的第一个元素的索引是 0，每个维度的第二个元素的索引是 1，依此类推。可以轻松地访问、赋值或打印前三个数组中的单个元素。

```go
myArray[0] 
twoD[1][2] = 15 
threeD[0][1][1] = -1

```

访问数组所有元素的最常见方法是使用`len()`函数找到其大小，然后使用`for`循环。然而，还有更酷的方法可以访问数组的所有元素，这涉及在`for`循环中使用`range`关键字，并允许您绕过`len()`函数的使用，当您必须处理两个或更多维数组时，这是非常方便的。

这个小节中的所有代码都保存在`arrays.go`中，你应该自己看一下。运行`arrays.go`会生成以下输出：

```go
$ go run arrays.go
1 2 4 -4
0 2 -2 6 7 8
1 2 3 4 5 15 7 8 9
[[1 2] [3 -1]] [[5 6] [7 8]]
```

现在让我们尝试通过尝试访问一些奇怪的数组元素来破坏事物，比如访问一个不存在的索引号的元素或者访问一个负索引号的元素，使用以下名为`breakMe.go`的 Go 程序：

```go
package main 

import "fmt" 

func main() { 
   myArray := [4]int{1, 2, 4, -4} 
   threeD := [2][2][2]int{{{1, 2}, {3, 4}}, {{5, 6}, {7, 8}}} 
   fmt.Println("myArray[-1]:", myArray[-1])
   fmt.Println("myArray[10]:", myArray[10]) 
   fmt.Println("threeD[-1][20][0]:", threeD[-1][20][0]) 
} 
```

执行`breakMe.go`将生成以下输出：

```go
$ go run breakMe.go
# command-line-arguments
./breakMe.go:8: invalid array index -1 (index must be non-negative)
./breakMe.go:9: invalid array index 10 (out of bounds for 4-element array)
./breakMe.go:10: invalid array index -1 (index must be non-negative)
./breakMe.go:10: invalid array index 20 (out of bounds for 2-element array)
```

Go 认为可以检测到的编译器问题是编译器错误，因为这有助于开发工作流程，这就是为什么要打印`breakMe.go`的所有越界数组访问错误的原因。

尝试破坏事物是一个非常有教育意义的过程，你应该一直尝试。简而言之，知道某些事情不起作用的时候同样有用，就像知道什么时候起作用一样有用。

尽管 Go 数组很简单，但存在许多严重的缺点：

+   首先，一旦定义了数组，就不能改变其大小，这意味着 Go 数组不是动态的。简而言之，如果您想要在没有空间的现有数组中包含额外的元素，您将需要创建一个更大的数组，并将所有元素从旧数组复制到新数组中。

+   其次，当你将数组传递给函数时，实际上是传递了数组的副本，这意味着你在函数内部对数组所做的任何更改在函数结束后都会丢失。

+   最后，将大数组传递给函数可能会非常慢，主要是因为 Go 必须创建数组的第二个副本。解决所有这些问题的方法是使用切片。

# 切片

在许多编程语言中，你不会找到**切片**的概念，尽管它既聪明又方便。切片与数组有许多相似之处，并且允许您克服数组的缺点。

切片有容量和长度属性，它们并不总是相同的。切片的长度与具有相同数量元素的数组的长度相同，并且可以使用`len()`函数找到。切片的容量是为该特定切片分配的当前空间，并可以使用`cap()`函数找到。由于切片的大小是动态的，如果切片的空间不足，Go 会自动将其当前长度加倍以为更多元素腾出空间。

切片作为引用传递给函数，你在函数内部对切片所做的任何修改在函数结束后都不会丢失。此外，将大切片传递给函数比传递相同数组要快得多，因为 Go 不必复制切片，它只会传递切片变量的内存地址。

这个小节的代码保存在`slices.go`中，可以分为三个主要部分。

第一部分是序言以及定义两个以`slice`作为输入的函数：

```go
package main 

import ( 
   "fmt" 
) 

func change(x []int) { 
   x[3] = -2 
} 

func printSlice(x []int) { 
   for _, number := range x {

         fmt.Printf("%d ", number) 
   } 
   fmt.Println() 
} 
```

请注意，当您在切片上使用`range`时，您会在其迭代中得到一对值。第一个是索引号，第二个是元素的值。当您只对存储的元素感兴趣时，您可以忽略索引号，就像`printSlice()`函数一样。

`change()`函数只更改输入切片的第四个元素，而`printSlice()`是一个实用函数，用于打印其切片输入变量的内容。在这里，您还可以看到使用`fmt.Printf()`函数打印整数。

第二部分创建了一个名为`aSlice`的新切片，并使用第一部分中看到的`change()`函数对其进行更改：

```go
func main() { 
   aSlice := []int{-1, 4, 5, 0, 7, 9} 
   fmt.Printf("Before change: ") 
   printSlice(aSlice) 
   change(aSlice) 
   fmt.Printf("After change: ") 
   printSlice(aSlice) 
```

尽管您定义填充切片的方式与定义数组的方式有一些相似之处，但最大的区别在于您不必声明切片将具有的元素数量。

最后一部分说明了 Go 切片的容量属性以及`make()`函数：

```go
   fmt.Printf("Before. Cap: %d, length: %d\n", cap(aSlice), len(aSlice)) 
   aSlice = append(aSlice, -100) 
   fmt.Printf("After. Cap: %d, length: %d\n", cap(aSlice), len(aSlice)) 
   printSlice(aSlice) 
   anotherSlice := make([]int, 4) 
   fmt.Printf("A new slice with 4 elements: ") 
   printSlice(anotherSlice) 
} 
```

`make()`函数会自动将切片的元素初始化为该类型的零值，可以通过`printSlice`（`anotherSlice`）语句的输出进行验证。请注意，使用`make()`函数创建切片时需要指定元素的数量。

执行`slices.go`生成以下输出：

```go
$ go run slices.go 
Before change: -1 4 5 0 7 9 
After change: -1 4 5 -2 7 9 
Before. Cap: 6, length: 6 
After. Cap: 12, length: 7 
-1 4 5 -2 7 9 -100 
A new slice with 4 elements: 0 0 0 0 
```

从输出的第三行可以看出，切片的容量和长度在定义时是相同的。但是，使用`append()`向切片添加新元素后，其长度从`6`变为`7`，但其容量翻倍，从`6`变为`12`。将切片的容量翻倍的主要优势是性能更好，因为 Go 不必一直分配内存空间。

您可以从现有数组的元素创建一个切片，并使用`copy()`函数将现有切片复制到另一个切片。这两个操作都有一些棘手的地方，您应该进行实验。

第六章，*文件输入和输出*，将讨论一种特殊类型的切片，称为字节切片，可用于文件 I/O 操作。

# 映射

Go 中的 Map 数据类型等同于其他编程语言中的哈希表。映射的主要优势是它们可以使用几乎任何数据类型作为其索引，这种情况下称为**key**。要将数据类型用作键，它必须是可比较的。

因此，让我们看一个示例 Go 程序，名为`maps.go`，我们将用它进行说明。`maps.go`的第一部分包含您期望的 Go 代码前言：

```go
package main 

import ( 
   "fmt" 
) 

func main() { 

```

然后，您可以定义一个新的空映射，其中字符串作为键，整数作为值，如下所示：

```go
   aMap := make(map[string]int) 
```

之后，您可以向`aMap`映射添加新的键值对，如下所示：

```go
   aMap["Mon"] = 0 
   aMap["Tue"] = 1 
   aMap["Wed"] = 2 
   aMap["Thu"] = 3 
   aMap["Fri"] = 4 
   aMap["Sat"] = 5 
   aMap["Sun"] = 6 
```

然后，您可以获取现有键的值：

```go
   fmt.Printf("Sunday is the %dth day of the week.\n", aMap["Sun"]) 

```

然而，您可以对现有`map`执行的最重要的操作在以下 Go 代码中进行了说明：

```go
   _, ok := aMap["Tuesday"] 
   if ok { 
         fmt.Printf("The Tuesday key exists!\n") 
   } else { 
         fmt.Printf("The Tuesday key does not exist!\n") 
   } 
```

上述 Go 代码的作用是利用 Go 的错误处理能力，以验证映射的键在尝试获取其值之前是否已存在。这是尝试获取`map`键的值的正确和安全方式，因为要求一个不存在的`key`的值将导致返回零。这样就无法确定结果是零，是因为您请求的`key`不存在，还是因为相应键的元素实际上具有零值。

以下 Go 代码显示了如何遍历现有映射的所有键：

```go
   count := 0 
   for key, _ := range aMap { 
         count++ 
         fmt.Printf("%s ", key) 
   } 
   fmt.Printf("\n") 
   fmt.Printf("The aMap has %d elements\n", count) 
```

如果您对访问映射的键和值没有兴趣，只想计算其对数，那么您可以使用前面`for`循环的下一个更简单的变体：

```go
   count = 0 
   delete(aMap, "Fri") 
   for _, _ = range aMap { 
         count++ 
   } 
   fmt.Printf("The aMap has now %d elements\n", count) 
```

`main()`函数的最后一部分包含以下 Go 代码，用于说明定义和初始化映射的另一种方式：

```go
   anotherMap := map[string]int{ 
         "One":   1, 
         "Two":   2, 
         "Three": 3, 
         "Four":  4, 
   } 
   anotherMap["Five"] = 5 
   count = 0 
   for _, _ = range anotherMap { 
         count++ 
   } 
   fmt.Printf("anotherMap has %d elements\n", count) 
} 
```

但是，除了不同的初始化之外，所有其他`map`操作都完全相同。执行`maps.go`生成以下输出：

```go
$ go run maps.go
Sunday is the 6th day of the week.
The Tuesday key does not exist!
Wed Thu Fri Sat Sun Mon Tue
The aMap has 7 elements
The aMap has now 6 elements
anotherMap has 5 elements
```

映射是一种非常方便的数据结构，当开发系统软件时，您很有可能会需要它们。

# 将数组转换为地图

这个小节将执行一个实际的操作，即在不提前知道`array`大小的情况下将数组转换为地图。`array2map.go`的 Go 代码可以分为三个主要部分。第一部分是标准的 Go 代码，包括所需的包和`main()`函数的开始：

```go
package main 

import ( 
   "fmt" 
   "strconv" 
) 

func main() { 
```

实现核心功能的第二部分如下：

```go
anArray := [4]int{1, -2, 14, 0} 
aMap := make(map[string]int) 

length := len(anArray) 
for i := 0; i < length; i++ { 
   fmt.Printf("%s ", strconv.Itoa(i)) 
   aMap[strconv.Itoa(i)] = anArray[i] 
} 
```

首先定义`array`变量和将要使用的`map`变量。`for`循环用于访问所有数组元素并将它们添加到`map`中。`strconv.Itoa()`函数将`array`的索引号转换为字符串。

请记住，如果你知道地图的所有键都将是连续的正整数，你可能会考虑使用数组或切片而不是地图。实际上，即使键不是连续的，数组和切片也比地图更便宜，所以你最终可能会得到一个稀疏矩阵。

最后一部分仅用于打印生成的地图的内容，使用了`for`循环的预期范围形式：

```go
for key, value := range aMap {
    fmt.Printf("%s: %d\n", key, value) 
   } 
} 
```

正如您可以轻松猜到的那样，开发逆操作并不总是可能的，因为`map`是比`array`更丰富的数据结构。但是，使用更强大的数据结构所付出的代价是时间，因为数组操作通常更快。

# 结构

尽管数组、切片和地图都非常有用，但它们不能在同一个位置保存多个值。当您需要对各种类型的变量进行分组并创建一个新的方便类型时，可以使用结构--结构的各个元素称为字段。

这个小节的代码保存为`dataStructures.go`，可以分为三部分。第一部分包含序言和一个名为`message`的新结构的定义：

```go
package main 

import ( 
   "fmt" 
   "reflect" 
) 

func main() { 

   type message struct {
         X     int 
         Y     int 
         Label string 
   } 
```

消息结构有三个字段，名为`X`、`Y`和`Label`。请注意，结构通常在程序开头和`main()`函数之外定义。

第二部分使用消息结构定义了两个名为`p1`和`p2`的新消息变量，然后使用反射获取有关消息结构的`p1`和`p2`变量的信息：

```go
   p1 := message{23, 12, "A Message"} 
   p2 := message{} 
   p2.Label = "Message 2" 

   s1 := reflect.ValueOf(&p1).Elem() 
   s2 := reflect.ValueOf(&p2).Elem() 
   fmt.Println("S2= ", s2) 
```

最后一部分展示了如何使用`for`循环和`Type()`函数打印结构的所有字段而不知道它们的名称：

```go
   typeOfT := s1.Type() 
   fmt.Println("P1=", p1) 
   fmt.Println("P2=", p2) 

   for i := 0; i < s1.NumField(); i++ {
         f := s1.Field(i)

         fmt.Printf("%d: %s ", i, typeOfT.Field(i).Name) 
         fmt.Printf("%s = %v\n", f.Type(), f.Interface()) 
   } 

} 
```

运行`dataStructures.go`将生成以下类型的输出：

```go
$ go run dataStructures.go
S2=  {0 0 Message 2}
P1= {23 12 A Message}
P2= {0 0 Message 2}
0: X int = 23
1: Y int = 12
2: Label string = A Message
```

如果`struct`定义的字段名称以小写字母开头（`x`而不是`X`），上一个程序将失败，并显示以下错误消息：

```go
panic: reflect.Value.Interface: cannot return value obtained from unexported field or method

```

这是因为小写字段不会被导出；因此，它们不能被`reflect.Value.Interface()`方法使用。您将在下一章中了解更多关于`reflection`的内容。

# 接口

接口是 Go 的高级功能，这意味着如果您对 Go 不太熟悉，可能不希望在程序中使用它们。但是，在开发大型 Go 程序时，接口可能非常实用，这是本书讨论接口的主要原因。

但首先，我将讨论方法，这些是带有特殊接收器参数的函数。您将方法声明为普通函数，并在函数名称之前添加一个额外的参数。这个特殊的参数将函数连接到该额外参数的类型。因此，该参数被称为方法的接收器。您一会儿会看到这样的函数。

简而言之，接口是定义一组需要实现的函数的抽象类型，以便将类型视为接口的实例。当这种情况发生时，我们说该类型满足此接口。因此，接口是两种东西--一组方法和一种类型--它用于定义类型的行为。

让我们用一个例子来描述接口的主要优势。想象一下，你有一个名为 ATYPE 的类型和一个适用于 ATYPE 类型的接口。接受一个 ATYPE 变量的任何函数都可以接受实现了 ATYPE 接口的任何其他变量。

`interfaces.go`的 Go 代码可以分为三部分。第一部分如下所示：

```go
package main 

import ( 
   "fmt" 
) 

type coordinates interface { 
   xaxis() int 
   yaxis() int 
} 

type point2D struct { 
   X int 
   Y int 
} 
```

在这一部分中，你定义了一个名为 coordinates 的接口和一个名为`point2D`的新结构。接口有两个函数，名为`xaxis()`和`yaxis()`。坐标接口的定义表示，如果要转换为坐标接口，必须实现这两个函数。

重要的是注意，接口除了接口本身不声明任何其他特定类型。另一方面，接口的两个函数应声明它们返回值的类型。

第二部分包含以下 Go 代码：

```go
func (s point2D) xaxis() int { 
   return s.X 
} 

func (s point2D) yaxis() int { 
   return s.Y 
} 

func findCoordinates(a coordinates) { 
   fmt.Println("X:", a.xaxis(), "Y:", a.yaxis()) 
} 

type coordinate int 

func (s coordinate) xaxis() int { 
   return int(s) 
} 

func (s coordinate) yaxis() int { 
   return 0 
} 
```

在第二部分中，首先为`point2D`类型实现坐标接口的两个函数。然后开发一个名为`findCoordinates()`的函数，该函数接受一个实现坐标接口的变量。`findCoordinates()`函数只是使用简单的`fmt.Println()`函数调用打印点的两个坐标。然后，定义一个名为 coordinate 的新类型，用于属于*x*轴的点。最后，为 coordinate 类型实现坐标接口。

在编写`interfaces.go`代码时，我认为`coordinates`和`coordinate`这两个名称还不错。在写完上一段之后，我意识到`coordinate`类型本可以改名为`xpoint`以提高可读性。我保留了`coordinates`和`coordinate`这两个名称，以指出每个人都会犯错误，你使用的变量和类型名称必须明智选择。

最后一部分包含以下 Go 代码：

```go
func main() { 

   x := point2D{X: -1, Y: 12}
   fmt.Println(x) 
   findCoordinates(x) 

   y := coordinate(10) 
   findCoordinates(y) 
} 
```

在这一部分中，首先创建一个`point2D`变量，并使用`findCoordinates()`函数打印其坐标，然后创建一个名为`y`的坐标变量，它保存一个单一的坐标值。最后，使用与打印`point2D`变量相同的`findCoordinates()`函数打印`y`变量。

尽管 Go 不是一种面向对象的编程语言，但我将在这里使用一些面向对象的术语。因此，在面向对象的术语中，这意味着`point2D`和`coordinate`类型都是坐标对象。但是，它们都不是*只是*`coordinate`对象。

执行`interfaces.go`会创建以下输出：

```go
$ go run interfaces.go
{-1 12}
X: -1 Y: 12
X: 10 Y: 0
```

我认为在开发系统软件时，Go 接口并不是必需的，但它们是一个方便的 Go 特性，可以使系统应用程序的开发更易读和更简单，所以不要犹豫使用它们。

# 创建随机数

作为一个实际的编程示例，本节将讨论在 Go 中创建随机数。随机数有许多用途，包括生成良好的密码以及创建具有随机数据的文件，这些文件可用于测试其他应用程序。但是，请记住，通常编程语言生成伪随机数，这些数近似于真随机数生成器的属性。

Go 使用`math/rand`包生成随机数，并需要一个种子来开始生成随机数。种子用于初始化整个过程，非常重要，因为如果始终使用相同的种子开始，将始终得到相同的随机数序列。

`random.go`程序有三个主要部分。第一部分是程序的序言：

```go
package main 

import ( 
   "fmt" 
   "math/rand" 
   "os" 
   "strconv" 
   "time" 
) 
```

第二部分是定义`random()`函数，每次调用该函数都会返回一个随机数，使用`rand.Intn()` Go 函数：

```go
func random(min, max int) int { 
   return rand.Intn(max-min) + min 
} 
```

`random()` 函数的两个参数定义了生成的随机数的下限和上限。`random.go` 的最后部分是 `main()` 函数的实现，主要用于调用 `random()` 函数：

```go
func main() { 
   MIN := 0 
   MAX := 0 
   TOTAL := 0 
   if len(os.Args) > 3 { 
         MIN, _ = strconv.Atoi(os.Args[1]) 
         MAX, _ = strconv.Atoi(os.Args[2]) 
         TOTAL, _ = strconv.Atoi(os.Args[3]) 
   } else { 
         fmt.Println("Usage:", os.Args[0], "MIX MAX TOTAL") 
         os.Exit(-1) 
   } 

   rand.Seed(time.Now().Unix()) 
   for i := 0; i < TOTAL; i++ { 
         myrand := random(MIN, MAX) 
         fmt.Print(myrand) 
         fmt.Print(" ") 
   } 
   fmt.Println() 
} 
```

`main()` 函数的一个重要部分涉及处理命令行参数作为整数，并在没有获得正确数量的命令行参数时打印描述性错误消息。这是本书中我们将遵循的标准做法。`random.go` 程序使用 Unix 纪元时间作为随机数生成器的种子，通过调用 `time.Now().Unix()` 函数。要记住的重要事情是，你不必多次调用 `rand.Seed()`。最后，`random.go` 不检查 `strconv.Atoi()` 返回的错误变量以节省书本空间，而不是因为它不必要。

执行 `random.go` 会生成以下类型的输出：

```go
$ go run random.go 12 32 20
29 27 20 23 22 28 13 16 22 26 12 29 22 30 15 19 26 24 20 29

```

如果你希望在 Go 中生成更安全的随机数，你应该使用 `crypto/rand` 包，它实现了一个密码学安全的伪随机数生成器。你可以通过访问其文档页面 [`golang.org/pkg/crypto/rand/`](https://golang.org/pkg/crypto/rand/) 获取有关 `crypto/rand` 包的更多信息。

如果你真的对随机数感兴趣，那么随机数理论的权威参考书是 Donald Knuth 的《计算机编程艺术》第二卷。

# 练习

1.  浏览 Go 文档网站：[`golang.org/doc/`](https://golang.org/doc/)。

1.  编写一个 Go 程序，它会一直读取整数，直到你输入数字 0 为止，然后打印输入中的最小和最大整数。

1.  编写与之前相同的 Go 程序，但这次，你将使用命令行参数获取输入。你认为哪个版本更好？为什么？

1.  编写一个支持两个命令行选项（`-i` 和 `-k`）的 Go 程序，使用 if 语句可以随机顺序。现在将你的程序更改为支持三个命令行参数。正如你将看到的，后一个程序的复杂性太大，无法使用 if 语句处理。

1.  如果映射的索引是自然数，是否有任何情况下使用映射而不是数组是明智且有效的？

1.  尝试将 `array2map.go` 的功能放入一个单独的函数中。

1.  尝试在 Go 中开发自己的随机数生成器，它仍然使用当前时间作为种子，但不使用 `math/rand` 包。

1.  学习如何从现有数组创建切片。当你对切片进行更改时会发生什么？

1.  使用 `copy()` 函数复制现有切片。当目标切片小于源切片时会发生什么？当目标切片大于源切片时会发生什么？

1.  尝试编写一个支持 3D 空间中的点的接口。然后，使用这个接口来支持位于 x 轴上的点。

# 总结

在本章中，你学到了很多东西，包括获取用户输入和处理命令行参数。你熟悉了基本的 Go 结构，并创建了一个生成随机数的 Go 程序。尝试做提供的练习，如果在某些练习中失败，不要灰心。

下一章将讨论许多高级的 Go 特性，包括错误处理、模式匹配、正则表达式、反射、不安全代码、从 Go 调用 C 代码以及 `strace(1)` 命令行实用程序。我将把 Go 与其他编程语言进行比较，并给出实用建议，以避免一些常见的 Go 陷阱。
