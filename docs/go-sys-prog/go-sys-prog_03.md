# 高级 Go 特性

在上一章中，您学习了如何编译 Go 代码，如何从用户那里获取输入并在屏幕上打印输出，如何创建自己的 Go 函数，Go 支持的数据结构以及如何处理命令行参数。

本章将讨论许多有趣的事情，因此您最好为许多有趣且实用的 Go 代码做好准备，这些代码将帮助您执行许多不同但非常重要的任务，从错误处理开始，以避免一些常见的 Go 错误结束。如果您熟悉 Go，可以跳过您已经知道的内容，但请不要跳过建议的练习。

因此，本章将讨论一些高级的 Go 特性，包括：

+   错误处理

+   错误日志记录

+   模式匹配和正则表达式

+   反射

+   如何使用`strace(1)`和`dtrace(1)`工具来监视 Go 可执行文件的系统调用

+   如何检测不可达的 Go 代码

+   如何避免各种常见的 Go 错误

# Go 中的错误处理

错误经常发生，因此我们的工作是捕捉并处理它们，特别是在编写处理敏感系统信息和文件的代码时。好消息是，Go 有一种特殊的数据类型叫做`error`，可以帮助表示错误状态；如果`error`变量的值为`nil`，则没有错误情况。

正如您在上一章中开发的`addCLA.go`程序中看到的，您可以使用`_`字符忽略大多数 Go 函数返回的`error`变量：

```go
temp, _ := strconv.Atoi(arguments[i]) 
```

然而，这并不被认为是良好的做法，应该避免，特别是在系统软件和其他类型的关键软件（如服务器进程）上。

正如您将在第六章中看到的，*文件输入和输出*，即使是**文件结束**（**EOF**）也是一种错误类型，在从文件中没有剩余内容可读时返回。由于`EOF`在`io`包中定义，您可以按以下方式处理它：

```go
if err == io.EOF {

    // Do something 
} 
```

然而，学习如何开发返回`error`变量的函数以及如何处理它们是最重要的任务，下面将对此进行解释。

# 函数可以返回错误变量

Go 函数可以返回`error`变量，这意味着错误条件可以在函数内部、函数外部或者函数内外都可以处理；后一种情况并不经常发生。因此，本小节将开发一个返回错误消息的函数。相关的 Go 代码可以在`funErr.go`中找到，并将分为三部分呈现。

第一部分包含以下 Go 代码：

```go
package main 

import ( 
   "errors" 
   "fmt" 
   "log" 
) 

func division(x, y int) (int, error, error) { 
   if y == 0 { 
         return 0, nil, errors.New("Cannot divide by zero!") 
   } 
   if x%y != 0 { 
         remainder := errors.New("There is a remainder!") 
         return x / y, remainder, nil 
   } else { 
         return x / y, nil, nil 
   } 

} 
```

除了预期的前言之外，上述代码定义了一个名为`division()`的新函数，该函数返回一个整数和两个`error`变量。如果您还记得您的数学课，当您除两个整数时，除法运算并不总是完美的，这意味着您可能会得到一个不为零的余数。您在`funErr.go`中看到的`errors` Go 包中的`errors.New()`函数创建一个新的`error`变量，使用提供的字符串作为错误消息。

`funErr.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   result, rem, err := division(2, 2) 
   if err != nil { 
         log.Fatal(err) 
   } else { 
         fmt.Println("The result is", result) 
   } 

   if rem != nil { 
         fmt.Println(rem) 
   } 
```

将`error`变量与`nil`进行比较是 Go 中非常常见的做法，可以快速判断是否存在错误条件。

`funErr.go`的最后一部分如下：

```go
   result, rem, err = division(12, 5) 
   if err != nil { 
         log.Fatal(err) 
   } else { 
         fmt.Println("The result is", result) 
   } 

   if rem != nil { 
         fmt.Println(rem) 
   } 

   result, rem, err = division(2, 0) 
   if err != nil { 
         log.Fatal(err) 
   } else { 
         fmt.Println("The result is", result) 
   } 

   if rem != nil { 
         fmt.Println(rem) 
   } 
} 
```

本部分展示了两种错误条件。第一种是具有余数的整数除法，而第二种是无效的除法，因为您不能将一个数除以零。正如名称`log.Fatal()`所暗示的，这个日志函数应该仅用于关键错误，因为当调用时，它会自动终止您的程序。然而，正如您将在下一小节中看到的，存在其他更温和的方式来记录您的错误消息。

执行`funErr.go`会生成以下输出：

```go
$ go run funErr.go
The result is 1
The result is 2
There is a remainder!
2017/03/07 07:39:19 Cannot divide by zero!
exit status 1
```

最后一行是由`log.Fatal()`函数自动生成的，在终止程序之前。重要的是要理解，在调用`log.Fatal()`之后的任何 Go 代码都不会被执行。

# 关于错误记录

Go 提供了可以帮助您以各种方式记录错误消息的函数。您已经在`funErr.go`中看到了`log.Fatal()`，这是一种处理简单错误的相当残酷的方式。简单地说，您应该有充分的理由在代码中使用`log.Fatal()`。一般来说，应该使用`log.Fatal()`而不是`os.Exit()`函数，因为它允许您使用一个函数调用打印错误消息并退出程序。

Go 在`log`标准包中提供了更温和地根据情况行为的附加错误记录函数，包括`log.Printf()`、`log.Print()`、`log.Println()`、`log.Fatalf()`、`log.Fatalln()`、`log.Panic()`、`log.Panicln()`和`log.Panicf()`。请注意，记录函数对于调试目的可能会很有用，因此不要低估它们的作用。

`logging.go`程序使用以下 Go 代码说明了所提到的两个记录函数：

```go
package main 

import ( 
   "log" 
) 

func main() { 
   x := 1 
   log.Printf("log.Print() function: %d", x) 
   x = x + 1 
   log.Printf("log.Print() function: %d", x) 
   x = x + 1 
   log.Panicf("log.Panicf() function: %d", x) 
   x = x + 1 
   log.Printf("log.Print() function: %d", x) 
} 
```

正如您所看到的，`logging.go`不需要`fmt`包，因为它有自己的函数来打印输出。执行`logging.go`将产生以下输出：

```go
$ go run logging.go
2017/03/10 16:51:56 log.Print() function: 1
2017/03/10 16:51:56 log.Print() function: 2
2017/03/10 16:51:56 log.Panicf() function: 3
panic: log.Panicf() function: 3

goroutine 1 [running]:
log.Panicf(0x10b78d0, 0x19, 0xc42003df48, 0x1, 0x1)
      /usr/local/Cellar/go/1.8/libexec/src/log/log.go:329 +0xda
main.main()
      /Users/mtsouk/ch3/code/logging.go:14 +0x1af
exit status 2
```

尽管`log.Printf()`函数的工作方式与`fmt.Printf()`相同，但它会自动打印日志消息打印的日期和时间，就像`funErr.go`中的`log.Fatal()`函数一样。此外，`log.Panicf()`函数的工作方式与`log.Fatal()`类似--它们都会终止当前程序。但是，`log.Panicf()`会打印一些额外的信息，用于调试目的。

Go 还提供了`log/syslog`包，它是 Unix 机器上运行的系统日志服务的简单接口。第七章，*使用系统文件*，将更多地讨论`log/syslog`包。

# 重新审视 addCLA.go 程序

本小节将介绍在前一章中开发的`addCLA.go`程序的改进版本，以使其能够处理任何类型的用户输入。新程序将被称为`addCLAImproved.go`，但是，您将只看到`addCLAImproved.go`和`addCLA.go`之间的差异，使用`diff(1)`命令行实用程序：

```go
$ diff addCLAImproved.go addCLA.go
13,18c13,14
<           temp, err := strconv.Atoi(arguments[i])
<           if err == nil {
<                 sum = sum + temp
<           } else {
<                 fmt.Println("Ignoring", arguments[i])
<           }
---
>           temp, _ := strconv.Atoi(arguments[i])
>           sum = sum + temp
```

这个输出基本上告诉我们的是，在`addCLA.go`中找到的最后两行代码，以`>`字符开头，被`addCLAImproved.go`中以`<`字符开头的代码替换了。两个文件的剩余代码完全相同。

`diff(1)`实用程序逐行比较文本文件，是发现同一文件不同版本之间代码差异的一种方便方法。

执行`addCLAImproved.go`将生成以下类型的输出：

```go
$ go run addCLAImproved.go
Sum: 0
$ go run addCLAImproved.go 1 2 -3
Sum: 0
$ go run addCLAImproved.go 1 a 2 b 3.2 @
Ignoring a
Ignoring b
Ignoring 3.2
Ignoring @
Sum: 3
```

因此，新的改进版本按预期工作，表现可靠，并允许我们区分有效和无效的输入。

# 模式匹配和正则表达式

**模式匹配**在 Go 中扮演着关键角色，它是一种基于**正则表达式**的搜索字符串的技术，用于根据特定的搜索模式搜索一组字符。如果模式匹配成功，它允许您从字符串中提取所需的数据，或者替换或删除它。**语法**是形式语言中字符串的一组生成规则。生成规则描述如何根据语言的语法创建有效的字符串。语法不描述字符串的含义或在任何上下文中可以对其进行的操作，只描述其形式。重要的是要意识到语法是正则表达式的核心，因为没有它，您无法定义或使用正则表达式。

正则表达式和模式匹配并非万能良药，因此不应尝试使用正则表达式解决每个问题，因为它们并不适用于您可能遇到的每种问题。此外，它们可能会给您的软件引入不必要的复杂性。

负责 Go 模式匹配功能的 Go 包称为`regexp`，您可以在`regExp.go`中看到其运行情况。`regExp.go`的代码将分为四部分呈现。

第一部分是预期的序言：

```go
package main 

import ( 
   "fmt" 
   "regexp" 
) 
```

第二部分如下：

```go
func main() { 
match, _ := regexp.MatchString("Mihalis", "Mihalis Tsoukalos") 
   fmt.Println(match) 
   match, _ = regexp.MatchString("Tsoukalos", "Mihalis tsoukalos") 
   fmt.Println(match) 
```

`regexp.MatchString()`的两次调用都尝试在给定的字符串（第二个参数）中查找静态字符串（第一个参数）。

第三部分包含一行 Go 代码，但至关重要：

```go
   parse, err := regexp.Compile("[Mm]ihalis") 
```

`regexp.Compile()`函数读取提供的正则表达式并尝试解析它。如果成功解析正则表达式，则`regexp.Compile()`返回`regexp.Regexp`变量类型的值，您随后可以使用它。`regexp.Compile()`函数中的`[Mm]`表达式表示您要查找的内容可以以大写`M`或小写`m`开头。`[`和`]`都是特殊字符，不是正则表达式的一部分。因此，提供的语法是天真的，只匹配单词`Mihalis`和`mihalis`。

最后一部分使用存储在`parse`变量中的先前正则表达式：

```go
   if err != nil { 
         fmt.Printf("Error compiling RE: %s\n", err) 
   } else { 
         fmt.Println(parse.MatchString("Mihalis Tsoukalos")) 
         fmt.Println(parse.MatchString("mihalis Tsoukalos")) 
         fmt.Println(parse.MatchString("M ihalis Tsoukalos")) 
         fmt.Println(parse.ReplaceAllString("mihalis Mihalis", "MIHALIS")) 
   } 
} 
```

运行`regExp.go`会生成以下输出：

```go
$ go run regExp.go
true
false
true
true
false
MIHALIS MIHALIS
```

因此，对`regexp.MatchString()`的第一次调用是匹配的，但第二次调用不是，因为模式匹配是区分大小写的，`Tsoukalos`与`tsoukalos`不匹配。最后的`parse.ReplaceAllString()`函数搜索给定的字符串（`"mihalis Mihalis"`）并用其第二个参数（`"MIHALIS"`）替换每个匹配项。

本节的其余部分将使用静态文本呈现各种示例，因为您还不知道如何读取文本文件。但是，由于静态文本将存储在数组中并逐行处理，因此所呈现的代码可以轻松修改以支持从外部文本文件获取输入。

# 打印行的给定列的所有值

这是一个非常常见的情景，因为您经常需要从结构化文本文件的给定列中获取所有数据，以便随后进行分析。将呈现`readColumn.go`的代码，该代码将在两部分中呈现，打印第三列中的值。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "strings" 
) 

func main() { 
   var s [3]string 
   s[0] = "1 2 3" 
   s[1] = "11 12 13 14 15 16" 
   s[2] = "-1 2 -3 -4 -5 6" 
```

在这里，您导入所需的 Go 包并使用包含三个元素的数组定义了一个包含三行的字符串。

第二部分包含以下 Go 代码：

```go
   column := 2 

   for i := 0; i < len(s); i++ { 
         data := strings.Fields(s[i]) 
         if len(data) >= column { 
               fmt.Println((data[column-1])) 
         } 
   } 
} 
```

首先，您定义您感兴趣的列。然后，您开始迭代存储在数组中的字符串。这类似于逐行读取文本文件。`for`循环内的 Go 代码拆分输入行的字段，将它们存储在`data`数组中，验证所需列的值是否存在，并在屏幕上打印它。所有繁重的工作都由方便的`strings.Fields()`函数完成，该函数根据空格字符拆分字符串，如`unicode.IsSpace()`中定义的，并返回一个字符串切片。虽然`readColumn.go`没有使用`regexp.Compile()`函数，但其实现逻辑仍然基于正则表达式的原则，使用了`strings.Fields()`。

要记住的一件重要的事情是，您永远不应信任您的数据。简而言之，始终验证您期望获取的数据是否存在。

执行`readColumn.go`将生成以下类型的输出：

```go
$ go run readColumn.go
2
12
2
```

第六章，*文件输入和输出*，将展示`readColumn.go`的改进版本，您可以将其用作起点，以便修改所示示例的其余部分。

# 创建摘要

在本节中，我们将开发一个程序，它将添加多行文本中给定列的所有值。为了使事情更有趣，列号将作为程序的参数给出。本小节的程序与上一小节的`readColumn.go`的主要区别在于，您需要将每个值转换为整数。

将开发的程序的名称是`summary.go`，可以分为三部分。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "strconv" 
   "strings" 
) 

func main() { 
   var s [3]string 
   s[0] = "1 b 3" 
   s[1] = "11 a 1 14 1 1" 
   s[2] = "-1 2 -3 -4 -5" 
```

第二部分包含以下 Go 代码：

```go
   arguments := os.Args 
   column, err := strconv.Atoi(arguments[1]) 
   if err != nil { 
         fmt.Println("Error reading argument") 
         os.Exit(-1) 
   } 
   if column == 0 { 
         fmt.Println("Invalid column") 
         os.Exit(1) 
   } 
```

前面的代码读取您感兴趣的列的索引。如果要使`summary.go`更好，可以检查`column`变量中的负值，并打印适当的错误消息。

`summary.go`的最后一部分如下：

```go
   sum := 0 
   for i := 0; i < len(s); i++ { 
         data := strings.Fields(s[i]) 
         if len(data) >= column { 
               temp, err := strconv.Atoi(data[column-1]) 
               if err == nil { 
                     sum = sum + temp 
               } else { 
                     fmt.Printf("Invalid argument: %s\n", data[column-1]) 
               } 
         } else { 
               fmt.Println("Invalid column!") 
         } 
   } 
   fmt.Printf("Sum: %d\n", sum) 
} 
```

正如您所看到的，`summary.go`中的大部分 Go 代码都是关于处理异常和潜在错误。`summary.go`的核心功能是用几行 Go 代码实现的。

执行`summary.go`将给出以下输出：

```go
$ go run summary.go 0
Invalid column
exit status 1
$ go run summary.go 2
Invalid argument: b
Invalid argument: a
Sum: 2
$ go run summary.go 1
Sum: 11
```

# 查找出现次数

一个非常常见的编程问题是找出 IP 地址在日志文件中出现的次数。因此，本小节中的示例将向您展示如何使用方便的映射结构来做到这一点。`occurrences.go`程序将分为三部分呈现。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "strings" 
) 

func main() { 

   var s [3]string 
   s[0] = "1 b 3 1 a a b" 
   s[1] = "11 a 1 1 1 1 a a" 
   s[2] = "-1 b 1 -4 a 1" 
```

第二部分如下：

```go
   counts := make(map[string]int) 

   for i := 0; i < len(s); i++ { 
         data := strings.Fields(s[i]) 
         for _, word := range data { 
               _, ok := counts[word] 
               if ok { 
                     counts[word] = counts[word] + 1 
               } else { 
                     counts[word] = 1 
               } 
         } 
   } 
```

在这里，我们使用上一章的知识创建了一个名为`counts`的映射，并使用两个`for`循环将所需的数据填充到其中。

最后一部分非常小，因为它只是打印`counts`映射的内容：

```go
   for key, _ := range counts {

         fmt.Printf("%s -> %d \n", key, counts[key]) 
   } 
} 
```

执行`occurrences.go`并使用`sort(1)`命令行实用程序对`occurrences.go`的输出进行排序将生成以下类型的输出：

```go
$ go run occurrences.go | sort -n -r -t\  -k3,3
1 -> 8
a -> 6
b -> 3
3 -> 1
11 -> 1
-4 -> 1
-1 -> 1
```

正如你所看到的，传统的 Unix 工具仍然很有用。

# 查找和替换

本小节中的示例将搜索提供的文本，查找给定字符串的两种变体，并用另一个字符串替换它。程序将被命名为`findReplace.go`，实际上将使用 Go 正则表达式。在这种情况下使用`regexp.Compile()`函数的主要原因是它极大地简化了事情，并允许您只访问文本一次。

`findReplace.go`程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "regexp" 
) 
```

下一部分如下：

```go
func main() { 

   var s [3]string 
   s[0] = "1 b 3" 
   s[1] = "11 a B 14 1 1" 
   s[2] = "b 2 -3 B -5" 

   parse, err := regexp.Compile("[bB]")

   if err != nil { 
         fmt.Printf("Error compiling RE: %s\n", err) 
         os.Exit(-1) 
   } 
```

前面的 Go 代码将找到大写`B`或小写`b`（`[bB]`）的每个出现。请注意，还有`regexp.MustCompile()`，它的工作方式类似于`regexp.Compile()`。但是，`regexp.MustCompile()`不会返回一个`error`变量；如果给定的表达式错误并且无法解析，它会直接 panic。因此，`regexp.Compile()`是一个更好的选择。

最后一部分如下：

```go
   for i := 0; i < len(s); i++ { 
         temp := parse.ReplaceAllString(s[i], "C") 
         fmt.Println(temp) 
   } 
} 
```

在这里，您可以使用`parse.ReplaceAllString()`将每个匹配项替换为大写的 C。

执行`findReplace.go`将生成预期的输出：

```go
$ go run findReplace.go
1 C 3
11 a C 14 1 1
C 2 -3 C -5
```

`awk(1)`和`sed(1)`命令行工具可以更轻松地完成大部分以前的任务，但`sed(1)`和`awk(1)`不是通用的编程语言。

# 反射

反射是 Go 的一个高级特性，它允许您动态了解任意对象的类型以及有关其结构的信息。您应该回忆起第二章中的`dataStructures.go`程序，*在 Go 中编写程序*，它使用反射来查找数据结构的字段以及每个字段的类型。所有这些都是在`reflect` Go 包和`reflect.TypeOf()`函数的帮助下完成的，该函数返回一个`Type`变量。

反射在`reflection.go` Go 程序中得到了展示，将分为四部分呈现。

第一个是 Go 程序的序言，代码如下：

```go
package main 

import ( 
   "fmt" 
   "reflect" 
) 
```

第二部分如下：

```go
func main() { 

   type t1 int 
   type t2 int 

   x1 := t1(1) 
   x2 := t2(1) 
   x3 := 1 
```

在这里，您创建了两种新类型，名为`t1`和`t2`，它们都是`int`，以及三个变量，名为`x1`、`x2`和`x3`。

第三部分包含以下 Go 代码：

```go
   st1 := reflect.ValueOf(&x1).Elem() 
   st2 := reflect.ValueOf(&x2).Elem() 
   st3 := reflect.ValueOf(&x3).Elem() 

   typeOfX1 := st1.Type() 
   typeOfX2 := st2.Type() 
   typeOfX3 := st3.Type() 

   fmt.Printf("X1 Type: %s\n", typeOfX1) 
   fmt.Printf("X2 Type: %s\n", typeOfX2) 
   fmt.Printf("X3 Type: %s\n", typeOfX3) 
```

在这里，您可以使用`reflect.ValueOf()`和`Type()`找到`x1`、`x2`和`x3`变量的类型。

`reflection.go`的最后一部分涉及`struct`变量：

```go
   type aStructure struct { 
         X    uint 
         Y    float64 
         Text string 
   } 

   x4 := aStructure{123, 3.14, "A Structure"} 
   st4 := reflect.ValueOf(&x4).Elem() 
   typeOfX4 := st4.Type() 

   fmt.Printf("X4 Type: %s\n", typeOfX4) 
   fmt.Printf("The fields of %s are:\n", typeOfX4) 

   for i := 0; i < st4.NumField(); i++ { 
         fmt.Printf("%d: Field name: %s ", i, typeOfX4.Field(i).Name) 
         fmt.Printf("Type: %s ", st4.Field(i).Type()) 
         fmt.Printf("and Value: %v\n", st4.Field(i).Interface()) 
   } 
} 
```

Go 中存在一些管理反射的规则，但讨论它们超出了本书的范围。您应该记住的是，您的程序可以使用反射来检查自己的结构，这是一种非常强大的能力。

执行`reflection.go`打印以下输出：

```go
$ go run reflection.go
X1 Type: main.t1
X2 Type: main.t2
X3 Type: int
X4 Type: main.aStructure
The fields of main.aStructure are:
0: Field name: X Type: uint and Value: 123
1: Field name: Y Type: float64 and Value: 3.14
2: Field name: Text Type: string and Value: A Structure
```

输出的前两行显示，Go 不认为类型`t1`和`t2`相等，尽管`t1`和`t2`都是`int`类型的别名。

旧习惯难改！

尽管 Go 试图成为一种安全的编程语言，但有时它被迫忘记安全性，并允许程序员做任何他/她想做的事情。

# 从 Go 调用 C 代码

Go 允许您调用 C 代码，因为有时执行某些任务的唯一方法，例如与硬件设备或数据库服务器通信，是使用 C。然而，如果您发现自己在同一个项目中多次使用此功能，您可能需要重新考虑您的方法和编程语言的选择。

在本书的范围之外更多地讨论 Go 中的这一功能。您应该记住的是，您很可能永远不需要从 Go 程序中调用 C 代码。然而，如果您希望探索这一 Go 功能，可以首先访问[cgo 工具的文档](https://golang.org/cmd/cgo/)，并查看[`github.com/golang/go/blob/master/misc/cgo/gmp/gmp.go`](https://github.com/golang/go/blob/master/misc/cgo/gmp/gmp.go)中的代码。

# 不安全的代码

不安全的代码是绕过 Go 的类型安全和内存安全的 Go 代码，需要使用`unsafe`包。您很可能永远不需要在 Go 程序中使用不安全的代码，但如果出于某种奇怪的原因您确实需要使用它，那可能与指针有关。

对于您的程序来说，使用不安全的代码可能是危险的，因此只有在绝对必要时才使用它。如果您不完全确定需要它，那么就不要使用它。

本小节中的示例代码保存为`unsafe.go`，将分两部分呈现。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "unsafe" 
) 

func main() { 
   var value int64 = 5

   var p1 = &value 
   var p2 = (*int32)(unsafe.Pointer(p1)) 
```

首先创建一个名为`value`的新`int64`变量。然后，创建一个指向它的指针命名为`p1`。接下来，创建另一个指针指向`p1`。然而，指向`p1`的`p2`指针是指向`int64`变量的指针，尽管`p1`指向`int64`变量。尽管这违反了 Go 的规则，但`unsafe.Pointer()`函数使这成为可能。

第二部分如下：

```go
   fmt.Println("*p1: ", *p1) 
   fmt.Println("*p2: ", *p2) 
   *p1 = 312121321321213212 
   fmt.Println(value) 
   fmt.Println("*p2: ", *p2) 
   *p1 = 31212132 
   fmt.Println(value) 
   fmt.Println("*p2: ", *p2) 
} 
```

执行`unsafe.go`将创建以下输出：

```go
$ go run unsafe.go
*p1:  5
*p2:  5
312121321321213212
*p2:  606940444
31212132
*p2:  31212132
```

输出显示了不安全指针有多危险。当`value`变量的值适合于`int32`内存空间（`5`和`31212132`）时，`p2`运行正常并显示正确的结果。然而，当`value`变量持有一个不适合`int32`内存空间的值（`312121321321213212`）时，`p2`显示了错误的结果（`606940444`），而没有提供警告或错误消息。

# 将 Go 与其他编程语言进行比较

Go 并不完美，但其他编程语言也不完美。本节将简要讨论其他编程语言，并将它们与 Go 进行比较，以便让您更好地了解您的选择。因此，可以与 Go 进行比较的编程语言列表包括：

+   **C**：C 是开发系统软件最流行的编程语言，因为每个 Unix 操作系统的可移植部分都是用 C 编写的。然而，它也有一些关键缺点，包括 C 指针，它们很棒也很快，但可能导致难以检测的错误和内存泄漏。此外，C 不提供垃圾回收；在 C 创建时，垃圾回收是一种可能会减慢计算机速度的奢侈品。然而，如今的计算机非常快，垃圾回收不再拖慢速度。此外，与其他系统编程语言相比，C 程序需要更多的代码来开发给定的任务。最后，C 是一种不支持现代编程范式的旧编程语言，比如面向对象和函数式编程。

+   **C++**：如前所述，我不再喜欢 C++。如果你认为应该使用 C++，那么你可能想考虑使用 C。然而，C++相对于 Go 的主要优势在于，如果需要，C++可以像 C 一样使用。然而，无论是 C 还是 C++都不支持并发编程。

+   **Rust**：Rust 是一种新的系统编程语言，试图避免由不安全代码引起的不愉快的错误。目前，Rust 的语法变化太快，但这将在不久的将来结束。如果出于某种原因你不喜欢 Go，你应该尝试 Rust。

+   **Swift**：在目前的状态下，Swift 更适合开发 macOS 系统的系统软件。然而，我相信在不久的将来，Swift 将在 Linux 机器上更受欢迎，所以你应该留意它。

+   **Python**：Python 是一种脚本语言，这是它的主要缺点。这是因为通常情况下，你不希望将系统软件的源代码公开给所有人。

+   **Perl**：关于 Python 所说的也适用于 Perl。然而，这两种编程语言都有大量的模块，可以让你的生活变得更轻松，你的代码变得更简洁。

如果你问我的意见，我认为 Go 是一种现代、可移植、成熟和安全的编程语言，用于编写系统软件。在寻找其他选择之前，你应该先尝试 Go。然而，如果你是一名 Go 程序员，想尝试其他东西，我建议你选择 Rust 或 Swift。然而，如果你需要编写可靠的并发程序，Go 应该是你的首选。

如果你无法在 Go 和 Rust 之间做出选择，那就试试 C。学习系统编程的基础知识比你选择的编程语言更重要。

尽管它们有缺点，但请记住，所有脚本编程语言都非常适合编写原型，并且它们的优势在于可以为软件创建图形界面。然而，使用脚本语言交付系统软件很少被接受，除非有一个真正的好理由这样做。

# 分析软件

有时程序因某种未知原因失败或性能不佳，你希望找出原因，而不必重写代码并添加大量的调试语句。因此，本节将讨论`strace(1)`和`dtrace(1)`，它们允许你在 Unix 机器上执行程序时看到幕后发生了什么。虽然这两个工具都可以与`go run`命令一起使用，但如果你首先使用`go build`创建可执行文件并使用该文件，你将获得更少的无关输出。这主要是因为`go run`在实际运行 Go 代码之前会生成临时文件，而你想调试的是实际程序，而不是用于构建程序的编译器。

请记住，尽管`dtrace(1)`比`strace(1)`更强大，并且有自己的编程语言，但`strace(1)`更适用于观察程序所做的系统调用。

# 使用 strace(1)命令行实用程序

`strace(1)`命令行实用程序允许您跟踪系统调用和信号。由于 Mac 机器上没有`strace(1)`，因此本节将使用 Linux 机器来展示`strace(1)`。但是，正如您将在稍后看到的那样，macOS 机器有`dtrace(1)`命令行实用程序，可以做更多的事情。

程序名称后面的数字指的是其页面所属的手册部分。尽管大多数名称只能找到一次，这意味着不必放置部分编号，但是有些名称可能位于多个部分，因为它们具有多重含义，例如`crontab(1)`和`crontab(5)`。因此，如果尝试检索此类页面而没有明确指定部分编号，将会得到手册中具有最小部分编号的条目。

要对`strace(1)`生成的输出有一个良好的感觉，请查看以下图，其中`strace(1)`用于检查`addCLAImproved.go`的可执行文件：

![](img/6c0c5c81-3946-433a-bc90-4dafd085d3a0.png)

在 Linux 机器上使用 strace(1)命令

`strace(1)`输出的真正有趣的部分是以下行，这在前面的图中看不到：

```go
$ strace ./addCLAImproved 1 2 2>&1 | grep write
write(1, "Sum: 3\n", 7Sum: 3
```

我们使用`grep(1)`命令行实用程序提取包含我们感兴趣的 C 系统调用的行，这种情况下是`write(2)`。这是因为我们已经知道`write(2)`用于打印输出。因此，您了解到在这种情况下，单个`write(2)` C 系统调用用于在屏幕上打印所有输出；它的第一个参数是文件描述符，第二个参数是要打印的文本。

请注意，您可能希望使用`strace(1)`的`-f`选项，以便还跟踪在程序执行期间可能创建的任何子进程。

请记住，还存在`write(2)`的另外两个变体，名为`pwrite(2)`和`writev(2)`，它们提供与`write(2)`相同的核心功能，但方式略有不同。

前一个命令的以下变体需要更多对`write(2)`的调用，因为它生成更多的输出：

```go
$ strace ./addCLAImproved 1 a b 2>&1 | grep write
write(1, "Ignoring a\n", 11Ignoring a
write(1, "Ignoring b\n", 11Ignoring b
write(1, "Sum: 1\n", 7Sum: 1
```

Unix 使用文件描述符作为访问其所有文件的内部表示，这些文件描述符是正整数值。默认情况下，所有 Unix 系统都支持三个特殊和标准的文件名：`/dev/stdin`、`/dev/stdout`和`/dev/stderr`。它们也可以使用文件描述符 0、1 和 2 进行访问。这三个文件描述符也分别称为标准输入、标准输出和标准错误。此外，文件描述符 0 可以在 Mac 机器上作为`/dev/fd/0`进行访问，在 Debian Linux 机器上可以作为`/dev/pts/0`进行访问，因为 Unix 中的一切都是文件。

因此，需要在命令的末尾放置`2>&1`的原因是将所有输出，从标准错误（文件描述符 2）重定向到标准输出（文件描述符 1），以便能够使用`grep(1)`命令进行搜索，该命令仅搜索标准输出。请注意，存在许多`grep(1)`的变体，包括`zegrep(1)`、`fgrep(1)`和`fgrep(1)`，当它们需要处理大型或巨大的文本文件时，可能会更快地工作。

您在这里看到的是，即使您在 Go 中编写，生成的可执行文件也使用 C 系统调用和函数，因为除了使用机器语言外，C 是与 Unix 内核通信的唯一方式。

# DTrace 实用程序

尽管在 FreeBSD 上工作的调试实用程序，如`strace(1)`和`truss(1)`，可以跟踪进程产生的系统调用，但它们可能会很慢，因此不适合解决繁忙的 Unix 系统上的性能问题。另一个名为`dtrace(1)`的工具使用**DTrace**设施，允许您在系统范围内看到幕后发生的事情，而无需修改或重新编译任何内容。它还允许您在生产系统上工作，并动态地观察运行的程序或服务器进程，而不会引入大量开销。

本小节将使用`dtruss(1)`命令行实用程序，它只是一个`dtrace(1)`脚本，显示进程的系统调用。当在 macOS 机器上检查`addCLAImproved.go`可执行文件时，`dtruss(1)`生成的输出看起来与以下截图中看到的类似：

![](img/f596ddbd-3b87-454d-8eda-478318fd1014.png)

在 macOS 机器上使用 dtruss(1)命令

再次，输出的以下部分验证了在 Unix 机器上，最终一切都被转换成 C 系统调用和函数，因为这是与 Unix 内核通信的唯一方式。您可以显示对`write(2)`系统调用的所有调用如下：

```go
$ sudo dtruss -c ./addCLAImproved 2000 2>&1 | grep write
```

然而，这一次你会得到大量的输出，因为 macOS 可执行文件多次使用`write(2)`而不是只使用一次来打印相同的输出。

开始意识到并非所有的 Unix 系统都以相同的方式工作，尽管它们有许多相似之处，这是很奇妙的。但这也意味着你不应该对 Unix 系统在幕后的工作方式做任何假设。

真正有趣的是以下命令的输出的最后部分：

```go
$ sudo dtruss -c ./addCLAImproved 2000
CALL                                        COUNT
__pthread_sigmask                               1
exit                                            1
getpid                                          1
ioctl                                           1
issetugid                                       1
read                                            1
thread_selfid                                   1
ulock_wake                                      1
bsdthread_register                              2
close                                           2
csops                                           2
open                                            2
select                                          2
sysctl                                          3
mmap                                            7
mprotect                                        8
stat64                                         41
write                                          83
```

你得到这个输出的原因是`-c`选项告诉`dtruss(1)`统计所有系统调用并打印它们的摘要，这种情况下显示`write(2)`被调用了 83 次，`stat64(2)`被调用了 41 次。

`dtrace(1)`实用程序比`strace(1)`更强大，并且有自己的编程语言，但学习起来更困难。此外，尽管有 Linux 版本的`dtrace(1)`，但在 Linux 系统上，`strace(1)`更加成熟，以更简单的方式跟踪系统调用。

您可以通过阅读 Brendan Gregg 和 Jim Mauro 的*DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X, and FreeBSD*以及访问[`dtrace.org/`](http://dtrace.org/)了解更多关于`dtrace(1)`实用程序的信息。

# 在 macOS 上禁用系统完整性保护

第一次尝试在 Mac OS X 机器上运行`dtrace(1)`和`dtruss(1)`可能会遇到麻烦，并收到以下错误消息：

```go
$ sudo dtruss ./addCLAImproved 1 2 2>&1 | grep -i write
dtrace: error on enabled probe ID 2132 (ID 156: syscall::write:return): invalid kernel access in action #12 at DIF offset 92
```

在这种情况下，你可能需要禁用 DTrace 的限制，但仍然保持系统完整性保护对其他所有内容有效。您可以通过访问[`support.apple.com/en-us/HT204899`](https://support.apple.com/en-us/HT204899)了解更多关于系统完整性保护的信息。

# 无法到达的代码

无法到达的代码是永远不会被执行的代码，是一种逻辑错误。由于 Go 编译器本身无法捕捉这种逻辑错误，因此您需要使用`go tool vet`命令来帮助。

你不应该将无法到达的代码与从未被有意执行的代码混淆，比如不需要的函数的代码，因此在程序中从未被调用。

本节的示例代码保存为`cannotReach.go`，可以分为两部分。

第一部分包含以下 Go 代码：

```go
package main 

import ( 
   "fmt" 
) 

func x() int {

   return -1 
   fmt.Println("Exiting x()") 
   return -1 
} 

func y() int { 
   return -1 
   fmt.Println("Exiting y()") 
   return -1 
} 
```

第二部分如下：

```go
func main() { 
   fmt.Println(x()) 
   fmt.Println("Exiting program...") 
} 
```

正如你所看到的，无法到达的代码在第一部分。`x()`和`y()`函数都有无法到达的代码，因为它们的`return`语句放错了位置。然而，我们还没有完成，因为我们将让`go tool vet`工具发现无法到达的代码。这个过程很简单，包括执行以下命令：

```go
$ go tool vet cannotReach.go
cannotReach.go:9: unreachable code
cannotReach.go:14: unreachable code

```

此外，您可以看到`go tool vet`即使周围的函数根本不会被执行，也会检测到无法到达的代码，就像`y()`一样。

# 避免常见的 Go 错误

本节将简要讨论一些常见的 Go 错误，以便您在程序中避免它们：

+   如果在 Go 函数中出现错误，要么记录下来，要么返回错误；除非你有一个非常好的理由，否则不要两者都做。

+   Go 接口定义行为，而不是数据和数据结构。

+   使用`io.Reader`和`io.Writer`接口，因为它们使您的代码更具可扩展性。

+   确保只在需要时将变量的指针传递给函数。其余时间，只传递变量的值。

+   错误变量不是字符串；它们是`error`值。

+   如果你害怕犯错，你很可能最终什么有用的事情都不会做。所以尽量多实验。

以下是可以应用于每种编程语言的一般建议：

+   在小型和独立的 Go 程序中测试您的 Go 代码和函数，以确保它们表现出您认为应该有的行为方式。

+   如果你不太了解 Go 的某个特性，在第一次使用之前先进行测试，特别是如果你正在开发系统实用程序。

+   不要在生产机器上测试系统软件

+   在将系统软件部署到生产机器上时，要在生产机器不忙的时候进行，并确保您有备份计划

# 练习

1.  查找并访问`log`包的文档页面。

1.  使用`strace(1)`来检查上一章中的`hw.go`。

1.  如果您使用 Mac，尝试使用`dtruss(1)`检查`hw.go`可执行文件。

1.  编写一个从用户那里获取输入并使用`strace(1)`或`dtruss(1)`检查其可执行文件的程序。

1.  访问 Rust 的网站[`www.rust-lang.org/`](https://www.rust-lang.org/)。

1.  访问 Swift 的网站[`swift.org/`](https://swift.org/)。

1.  访问`io`包的文档页面[`golang.org/pkg/io/`](https://golang.org/pkg/io/)。

1.  自己使用`diff(1)`命令行实用程序，以便更好地学习如何解释其输出。

1.  访问并阅读`write(2)`的主页。

1.  访问`grep(1)`的主页。

1.  通过检查自己的结构来自己玩反射。

1.  编写一个改进版本的`occurrences.go`，它只会显示高于已知数值阈值的频率，该阈值将作为命令行参数给出。

# 总结

本章教会了您一些高级的 Go 特性，包括错误处理、模式匹配和正则表达式、反射和不安全的代码。还讨论了`strace(1)`和`dtrace(1)`工具。

下一章将涵盖许多有趣的内容，包括使用最新 Go 版本（1.8）中提供的新`sort.slice()` Go 函数，以及大 O 符号、排序算法、Go 包和垃圾回收。
