# 第三章：Go 概述

本章将概述 Go 语言及其基本功能。我们将简要解释语言及其特性，并在接下来的章节中详细阐述。这将帮助我们更好地理解 Go，同时使用其所有功能和应用。

本章将涵盖以下主题：

+   语言的特点

+   包和导入

+   基本类型、接口和用户定义类型

+   变量和函数

+   流程控制

+   内置函数

+   并发模型

+   内存管理

# 技术要求

从本章开始，您需要在计算机上安装 Go。按照以下步骤进行操作：

1.  从[`golang.org/dl/`](https://golang.org/dl/)下载 Go 的最新版本。

1.  使用`tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz`进行提取。

1.  使用`export PATH=$PATH:/usr/local/go/bin`将其添加到`PATH`中。

1.  确保使用`go version`安装了 Go。

1.  在您的`.profile`中添加 export 语句以自动添加。

1.  如果要使用不同的目录来存放代码，也可以更改`GOPATH`变量（默认为`~/go`）。

我还建议安装 Visual Studio Code（https://code.visualstudio.com/）及其 vscode-go（https://github.com/Microsoft/vscode-go）扩展，其中包含一个助手，可安装所有需要改进 Go 开发体验的工具。

# 语言特性

Go 是一种具有出色并发原语和大部分自动化内存系统的现代服务器语言。一些人认为它是 C 的继任者，并且在许多场景中都能做到这一点，因为它的性能良好，具有广泛的标准库，并且有一个提供许多第三方库的伟大社区，这些库涵盖、扩展和改进了其功能。

# Go 的历史

Go 是在 2007 年创建的，旨在解决 Google 的工程问题，并于 2009 年公开宣布，2012 年达到 1.0 版本。主要版本仍然相同（版本 1），而次要版本（版本 1.1、1.2 等）随着其功能一起增长。这样做是为了保持 Go 对所有主要版本的兼容性承诺。2018 年提出了两个新功能（泛型和错误处理）的草案，这些功能可能会包含在 2.0 版本中。

Go 背后的大脑如下：

+   **Robert Griesemer**：谷歌研究员，参与了许多项目，包括 V8 JavaScript 引擎和设计，以及 Sawzall 的实现。

+   **Rob Pike**：Unix 团队成员，Plan 9 和 Inferno 操作系统开发团队成员，Limbo 编程语言设计团队成员。

+   **Ken Thompson**：计算机科学的先驱，原始 Unix 的设计者，B 语言的发明者（C 的前身）。Ken 还是 Plan 9 操作系统的创造者和早期开发者之一。

# 优势和劣势

Go 是一种非常有主见的语言；有些人喜欢它，有些人讨厌它，主要是由于其一些设计选择。以下是一些未受好评的功能：

+   冗长的错误处理

+   缺乏泛型

+   缺少依赖和版本管理

前两点将在下一个主要版本中解决，而后者首先由社区（godep、glide 和 govendor）和 Google 自己（dep 用于依赖项）以及 gopkg.in（http://labix.org/gopkg.in）在版本管理方面解决。

语言的优势是无数的：

+   这是一种静态类型的语言，具有静态类型检查等带来的所有优势。

+   它不需要集成开发环境（IDE），即使它支持许多 IDE。

+   标准库非常强大，对许多项目来说可能是唯一的依赖项。

+   它具有并发原语（通道和 goroutines），隐藏了编写高效且安全的异步代码的最困难部分。

+   它配备了一个格式化工具`gofmt`，统一了 Go 代码的格式，使其他人的代码看起来非常熟悉。

+   它生成没有依赖的二进制文件，使部署快速简便。

+   它是极简主义的，关键字很少，代码非常容易阅读和理解。

+   它是鸭子类型的，具有隐式接口定义（*如果它走起来像鸭子，游泳像鸭子，嘎嘎叫像鸭子，那么它可能就是鸭子*）。这在测试系统的特定功能时非常方便，因为它可以被模拟。

+   它是跨平台的，这意味着它能够为与托管平台不同的架构和操作系统生成二进制文件。

+   有大量的第三方包，因此在功能方面留下了很少。托管在公共存储库上的每个包都是可索引和可搜索的。

# 命名空间

现在，让我们看看 Go 代码是如何组织的。`GOPATH`环境变量决定了代码的位置。里面有三个子目录：

+   `src`包含所有源代码。

+   `pkg`包含已编译的包，分为架构/操作系统。

+   `bin`包含编译后的二进制文件。

源文件夹下的路径对应包的名称（`$GOPATH/src/my/package/name`将是`my/package/name`）。

`go get`命令使得可以使用它来获取和编译包。`go get`调用`http://package_name?go-get=1`，如果找到`go-import`元标签，就会使用它来获取包。该标签应包含包名称、使用的 VCS 和存储库 URL，所有这些都用空格分隔。让我们看一个例子：

```go
 <meta name="go-import" content="package-name vcs repository-url">
```

`go get`下载一个包后，它会尝试对其他无法递归解析的包执行相同的操作，直到所有必要的源代码都可用。

每个文件都以`package`定义开头，即`package package_name`，需要对目录中的所有文件保持一致。如果包生成一个二进制文件，那么包就是`main`。

# 导入和导出符号

包声明后是一系列`import`语句，指定所需的包。

导入未使用的包（除非它们被忽略）是编译错误，这就是为什么 Go 格式化工具`gofmt`会删除未使用的包。还有一些实验性或社区工具，如 goimports ([`godoc.org/golang.org/x/tools/cmd/goimports`](https://godoc.org/golang.org/x/tools/cmd/goimports))或 goreturns ([`github.com/sqs/goreturns`](https://github.com/sqs/goreturns))，也会向 Go 文件添加丢失的导入。避免循环依赖非常重要，因为它们将无法编译。

由于不允许循环依赖，包需要与其他语言设计不同。为了打破循环依赖，最好的做法是从一个包中导出功能，或者用接口替换依赖关系。

Go 将所有符号可见性减少到一个二进制模型 - 导出和未导出 - 不像许多其他语言有中间级别。对于每个包，所有以大写字母开头的符号都是导出的，而其他所有内容只能在包内部使用。导出的值也可以被其他包使用，而未导出的值只能在包内部使用。

一个例外是，如果包路径中的一个元素是`internal`（例如`my/package/internal/pdf`），这将限制它及其子包只能被附近的包导入（例如`my/package`）。如果有很多未导出的符号，并且希望将它们分解为子包，同时阻止其他包使用它，这将非常有用，基本上是私有子包。看一下以下内部包的列表：

+   `my/package/internal`

+   `my/package/internal/numbers`

+   `my/package/internal/strings`

这些只能被`my/package`使用，不能被任何其他包导入，包括`my`。

导入可以有不同的形式。标准导入形式是完整的包名称：

```go
import "math/rand"
...
rand.Intn
```

命名导入将包名称替换为自定义名称，引用包时必须使用该名称：

```go
import r "math/rand"
...
r.Intn
```

相同的包导入使符号可用，而无需命名空间：

```go
import . "math/rand"
...
Intn
```

忽略的导入用于导入包，而无需使用它们。这使得可以在不在代码中引用包的情况下执行包的`init`函数：

```go
import _ math/rand  
// still executes the rand.init function
```

# 类型系统

Go 类型系统定义了一系列基本类型，包括字节，字符串和缓冲区，复合类型如切片或映射，以及应用程序定义的自定义类型。

# 基本类型

这些是 Go 的基本类型：

| **类别** | **类型** |
| --- | --- |
| 字符串 | `string` |
| 布尔值 | `bool` |
| 整数 | `int`，`int8`，`int16`，`int32`和`int64` |
| 无符号整数 | `uint`，`uint8`，`uint16`，`uint32`和`uint64` |
| 整数指针 | `uinptr` |
| 浮点数 | `float32`和`float64` |
| 复数 | `complex64`和`complex128` |

`int`，`uint`和`uiptr`的位数取决于架构（例如，x86 为 32 位，x86_64 为 64 位）。

# 复合类型

除了基本类型外，还有其他类型，称为复合类型。它们如下：

| **类型** | **描述** | **示例** |
| --- | --- | --- |
| 指针 | 变量在内存中的地址 | `*int` |
| 数组 | 具有固定长度的相同类型元素的容器 | `[2]int` |
| 切片 | 数组的连续段 | `[]int` |
| 映射 | 字典或关联数组 | `map[int]int` |
| 结构体 | 可以具有不同类型字段的集合 | `struct{ value int }` |
| 函数 | 具有相同参数和输出的一组函数 | `func(int, int) int` |
| 通道 | 用于通信相同类型元素的管道 | `chan int` |
| 接口 | 一组特定的方法，具有支持它们的基础值 | `interface{}` |

空接口`interface{}`是一个通用类型，可以包含任何值。由于此接口没有要求（方法），因此可以满足任何值。

接口，指针，切片，函数，通道和映射可以具有空值，在 Go 中用`nil`表示：

+   指针是不言自明的；它们不指向任何变量地址。

+   接口的基础值可以为空。

+   其他指针类型，如切片或通道，可以为空。

# 自定义定义的类型

包可以通过使用`type defined definition`表达式定义自己的类型，其中定义是共享定义内存表示的类型。自定义类型可以由基本类型定义：

```go
type Message string    // custom string
type Counter int       // custom integer
type Number float32    // custom float
type Success bool      // custom boolean
```

它们也可以由切片，映射或指针等复合类型定义：

```go
type StringDuo [2]string            // custom array   
type News chan string               // custom channel
type Score map[string]int           // custom map
type IntPtr *int                    // custom pointer
type Transform func(string) string  // custom function
type Result struct {                // custom struct
    A, B int
}
```

它们也可以与其他自定义类型结合使用：

```go
type Broadcast Message // custom Message
type Timer Counter     // custom Counter
type News chan Message // custom channel of custom type Message
```

自定义类型的主要用途是定义方法并使类型特定于范围，例如定义名为`Message`的`string`类型。

接口定义的工作方式不同。可以通过指定一系列不同的方法来定义它们，例如：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

它们也可以是其他接口的组合：

```go
type ReadCloser interface {
    Reader 
    Closer 
}
```

或者，它们可以是两个接口的组合：

```go
type ReadCloser interface {
    Reader        // composition
    Close() error // method
}
```

# 变量和函数

既然我们已经看过了类型，我们将看看如何在语言中实例化不同的类型。我们将首先了解变量和常量的工作原理，然后再讨论函数和方法。

# 处理变量

变量表示映射到连续内存部分的内容。它们具有定义内存扩展量的类型，以及指定内存中内容的值。类型可以是基本类型，复合类型或自定义类型，其值可以通过声明初始化为零值，也可以通过赋值初始化为其他值。

# 声明

使用 `var` 关键字和指定名称和类型来声明变量的零值；例如 `var a int`。对于来自其他语言（如 Java）的人来说，这可能有些反直觉，因为类型和名称的顺序是颠倒的，但实际上更易读。

`var a int` 的例子描述了一个带有名称 `a` 的变量（`var`），它是一个整数（`int`）。这种表达式创建了一个新的变量，其选定类型的零值为：

| **类型** | **零值** |
| --- | --- |
| 数值类型（`int`、`uint` 和 `float` 类型） | `0` |
| 字符串（`string` 类型） | `""` |
| 布尔值 | `false` |
| 指针、接口、切片、映射和通道 | `nil` |

初始化变量的另一种方式是通过赋值，可以有推断类型或特定类型。通过以下方式实现推断类型：

+   变量名称，后跟 `:=` 运算符和值（例如 `a := 1`），也称为**短声明**。

+   `var` 关键字，后跟名称、`=` 运算符和一个值（例如 `var a = 1`）。

请注意，这两种方法几乎是等效的，也是多余的，但 Go 团队决定保留它们两者，以遵守 Go 1 的兼容性承诺。就短声明而言，主要区别在于类型不能被指定，而是由值推断出来。

具有特定类型的赋值是通过声明进行的，后跟等号和值。如果声明的类型是接口，或者推断的类型是不正确的，这将非常有用。以下是一些示例：

```go
var a = 1             // this will be an int
b := 1                // this is equivalent
var c int64 = 1       // this will be an int64

var d interface{} = 1 // this is declared as an interface{}
```

有些类型需要使用内置函数才能正确初始化：

+   `new` 可以创建某种类型的指针，同时为底层变量分配一些空间。

+   `make` 初始化切片、映射和通道：

+   切片需要一个额外的参数用于大小，以及一个可选的参数用于底层数组的容量。

+   映射可以有一个参数用于其初始容量。

+   通道也可以有一个参数用于其容量。它们不同于映射，映射不能改变。

使用内置函数初始化类型的方法如下：

```go
a := new(int)                   // pointer of a new in variable
sliceEmpty := make([]int, 0)    // slice of int of size 0, and a capacity of 0
sliceCap := make([]int, 0, 10)  // slice of int of size 0, and a capacity of 10
map1 = make(map[string]int)     // map with default capacity
map2 = make(map[string]int, 10) // map with a capacity of 10
ch1 = make(chan int)            // channel with no capacity (unbuffered)
ch2 = make(chan int, 10)        // channel with capacity of 10 (buffered)
```

# 操作

我们已经看到了赋值操作，使用 `=` 运算符给变量赋予一个新值。让我们再看看一些运算符：

+   有比较运算符 `==` 和 `!=`，用于比较两个值并返回一个布尔值。

+   有一些数学运算可以在相同类型的所有数字变量上执行，即 `+`、`-`、`*` 和 `/`。求和运算也用于连接字符串。`++` 和 `--` 是用于将数字增加或减少一的简写形式。`+=`、`-=`、`*=` 和 `/=` 在等号之前执行操作，并将其赋给左边的变量。这四种运算产生与所涉及的变量相同类型的值；还有其他一些特定于数字的比较运算符：`<`、`<=`、`>` 和 `>=`。

+   有些运算仅适用于整数，并产生其他整数：`%`、`&`、`|`、`^`、`&^`、`<<` 和 `>>`。

+   还有一些仅适用于布尔值的运算符，产生另一个布尔值：`&&`、`||` 和 `!`。

+   一个运算符仅适用于通道，即 `<-`，用于从通道接收值或向通道发送值。

+   对于所有非指针变量，也可以使用 `&`，即**引用运算符**，来获取可以分配给指针变量的变量地址。`*` 运算符使得可以对指针执行**解引用操作**，并获取其指示的变量的值：

| **运算符** | **名称** | **描述** | **示例** |
| --- | --- | --- | --- |
| `=` | 赋值 | 将值赋给变量 | `a = 10` |
| `:=` | 声明和赋值 | 声明一个变量并给它赋值 | `a := 0` |
| `==` | 等于 | 比较两个变量，如果它们相同则返回布尔值 | `a == b` |
| `!=` | 不等于 | 比较两个变量，如果它们不同则返回布尔值 | `a != b` |
| `+` | 加 | 相同数值类型之间的求和 | `a + b` |
| `-` | 减 | 相同数值类型之间的差异 | `a - b` |
| `*` | 乘 | 相同数值类型的乘法 | `a * b` |
| `/` | 除 | 相同数值类型之间的除法 | `a / b` |
| `%` | 取模 | 相同数值类型的除法后余数 | `a % b` |
| `&` | 和 | 按位与 | `a & b` |
| `&^` | 位清除 | 位清除 | `a &^ b` |
| `<<` | 左移 | 位向左移动 | `a << b` |
| `>>` | 右移 | 位向右移动 | `a >> b` |
| `&&` | 和 | 布尔与 | `a && b` |
| ` | | ` | 或 | 布尔或 | `a | | b` |
| `!` | 非 | 布尔非 | `!a` |
| `<-` | 接收 | 从通道接收 | `<-a` |
| `->` | 发送 | 发送到通道 | `a <- b` |
| `&` | 引用 | 返回变量的指针 | `&a` |
| `*` | 解引用 | 返回指针的内容 | `*a` |

# 转换

将一种类型转换为另一种类型的操作称为**转换**，对于接口和具体类型的工作方式略有不同：

+   接口可以转换为实现它的具体类型。此转换可以返回第二个值（布尔值），并显示转换是否成功。如果省略布尔变量，则应用程序将在转换失败时出现恐慌。

+   具体类型之间可以发生类型转换，这些类型具有相同的内存结构，或者可以在数值类型之间发生转换：

```go
type N [2]int                // User defined type
var n = N{1,2}
var m [2]int = [2]int(N)     // since N is a [2]int this casting is possible

var a = 3.14                 // this is a float64
var b int = int(a)           // numerical types can be casted, in this case a will be rounded to 3

var i interface{} = "hello"  // a new empty interface that contains a string
x, ok := i.(int)             // ok will be false
y := i.(int)                 // this will panic
z, ok := i.(string)          // ok will be true
```

有一种特殊的条件运算符用于转换，称为**类型开关**，它允许应用程序一次尝试多次转换。以下是使用`interface{}`检查底层值的示例：

```go
func main() {
    var a interface{} = 10
    switch a.(type) {
    case int:
        fmt.Println("a is an int")
    case string:
        fmt.Println("a is a string")
    }
}
```

# 作用域

变量具有作用域或可见性，这也与其生命周期相关。这可以是以下之一：

+   **包**: 变量在所有包中可见；如果变量被导出，它也可以从其他包中可见。

+   **函数**: 变量在声明它的函数内可见。

+   **控制**: 变量在定义它的块内可见。

可见性下降，从包到块。由于块可以嵌套，外部块对内部块的变量没有可见性。

同一作用域中的两个变量不能具有相同的名称，但内部作用域的变量可以重用标识符。当这种情况发生时，外部变量在内部作用域中不可见 - 这称为**遮蔽**，需要牢记以避免出现难以识别的问题，例如以下情况：

```go
// this exists in the outside block
var err error
// this exists only in this block, shadows the outer err
if err := errors.New("Doh!"); err != 
    fmt.Println(err)           // this not is changing the outer err
}
fmt.Println(err)               // outer err has not been changed
```

# 常量

Go 的变量没有不可变性，但定义了另一种不可变值的类型称为常量。这由`const`关键字定义（而不是`var`），它们是不能改变的值。这些值可以是基本类型和自定义类型，如下所示：

+   数值（整数，`float`）

+   复杂

+   字符串

+   布尔

指定的值在分配给变量时没有类型。数值类型和基于字符串的类型都会自动转换，如下面的代码所示：

```go
const PiApprox = 3.14

var PiInt int = PiApprox // 3, converted to integer
var Pi float64 = PiApprox // is a float

type MyString string

const Greeting = "Hello!"

var s1 string = Greeting   // is a string
var s2 MyString = Greeting // string is converted to MyString
```

数值常量对数学运算非常有用，因为它们只是常规数字，所以可以与任何数值变量类型一起使用。

# 函数和方法

Go 中的函数由`func`关键字标识，后跟标识符、可能的参数和返回值。Go 中的函数可以一次返回多个值。参数和返回类型的组合称为**签名**，如下面的代码所示：

```go
func simpleFunc()
func funcReturn() (a, b int)
func funcArgs(a, b int)
func funcArgsReturns(a, b int) error
```

括号中的部分是函数体，`return`语句可以在其中用于提前中断函数。如果函数返回值，则`return`语句必须返回相同类型的值。

`return`值可以在签名中命名；它们是零值变量，如果`return`语句没有指定其他值，那么这些值就是返回的值：

```go
func foo(a int) int {        // no variable for returned type
    if a > 100 {
        return 100
    }
    return a
}

func bar(a int) (b int) {    // variable for returned type
    if a > 100 {
        b = 100
        return               // same as return b
    }
    return a
}
```

在 Go 中，函数是一级类型，它们也可以被分配给变量，每个签名代表不同的类型。它们也可以是匿名的；在这种情况下，它们被称为**闭包**。一旦一个变量被初始化为一个函数，相同的变量可以被重新分配为具有相同签名的另一个函数。以下是将闭包分配给变量的示例：

```go
var a = func(item string) error { 
    if item != "elixir" {
        return errors.New("Gimme elixir!")
    }
    return nil 
}
```

由接口声明的函数称为方法，它们可以由自定义类型实现。方法的实现看起来像一个函数，唯一的区别是名称前面有一个实现类型的单个参数。这只是一种语法糖——方法定义在幕后创建一个函数，它接受一个额外的参数，即实现方法的类型。

这种语法使得可以为不同类型定义相同的方法，每个方法将作为函数声明的命名空间。通过这种方式，可以以两种不同的方式调用方法，如下面的代码所示：

```go
type A int

func (a A) Foo() {}

func main() {
    A{}.Foo()  // Call the method on an instance of the type
    A.Foo(A{}) // Call the method on the type and passing an instance as argument  
}
```

重要的是要注意，类型及其指针共享相同的命名空间，因此同一个方法只能为其中一个实现。同一个方法不能同时为类型和其指针定义，因为对于类型和其指针声明两次方法将产生编译错误（方法重复声明）。方法不能为接口定义，只能为具体类型定义，但接口可以用于复合类型，包括函数参数和返回值，如下面的示例所示：

```go
// use error interface with chan
type ErrChan chan error
// use error interface in a map
type Result map[string]error

type Printer interface{
    Print()
}
// use the interface as argument
func CallPrint(p Printer) {
    p.Print()
}
```

内置包已经定义了一个接口，它在标准库中和所有在线可用的包中都被使用——`error`接口：

```go
type error interface {
    Error() string
}
```

这意味着任何类型都可以使用`Error() string`方法作为错误，并且每个包都可以根据自己的需要定义其错误类型。这可以用来简洁地携带有关错误的信息。在这个例子中，我们定义了`ErrKey`，它指定了未找到`string`键。除了键之外，我们不需要任何其他东西来表示我们的错误，如下面的代码所示：

```go
type ErrKey string

func (e Errkey) Error() string {
    returm fmt.Errorf("key %q not found", e)
}
```

# 值和指针

在 Go 中，一切都是按值传递的，所以当函数或方法被调用时，变量的副本会被放在堆栈中。这意味着对值所做的更改不会反映在被调用的函数之外。即使切片、映射和其他引用类型也是按值传递的，但由于它们的内部结构包含指针，它们的行为就像是按引用传递一样。如果为一种类型定义了一个方法，则不能为其指针定义该方法，反之亦然。下面的示例已经用来检查值只在方法内部更新，而这种更改不会反映在`main`函数中：

```go
package main

import (
    "fmt"
)

type A int

func (a A) Foo() {
    a++
    fmt.Println("foo", a)
}

func main() {
    var a A
    fmt.Println("before", a) // 0
    a.Foo() // 1
    fmt.Println("after", a) // 0
}
```

为了改变原始变量，参数必须是指向变量本身的指针——指针将被复制，但它将引用相同的内存区域，从而可以改变其值。请注意，分配另一个值指针，而不是其内容，不会改变原始指针所引用的内容，因为它是一个副本。

如果我们为类型而不是其指针使用方法，我们将看不到更改在方法外传播。

在下面的示例中，我们使用了值接收器。这使得`Birthday`方法中的`User`值成为`main`中的`User`值的副本：

```go
type User struct {
    Name string
    Age int
}

func (u User) Birthday() {
    u.Age++
    fmt.Println(u.Name, "turns", u.Age)
}

func main() {
    u := User{Name: "Pietro", Age: 30}
    fmt.Println(u.Name, "is now", u.Age)
    u.Birthday()
    fmt.Println(u.Name, "is now", u.Age)
}
```

完整的示例可在[`play.golang.org/p/hnUldHLkFJY`](https://play.golang.org/p/hnUldHLkFJY)中找到。

由于更改是应用于副本的，原始值保持不变，正如我们从第二个打印语句中所看到的。如果我们想要更改原始对象中的值，我们必须使用指针接收器，这样被复制的对象将是指针，更改将被应用于底层值：

```go
func (u *User) Birthday() {
    u.Age++
    fmt.Println(u.Name, "turns", u.Age)
}
```

完整示例可在[`play.golang.org/p/JvnaQL9R7U5`](https://play.golang.org/p/JvnaQL9R7U5)中找到。

我们可以看到使用指针接收器允许我们更改底层值，并且我们可以更改`struct`的一个字段或替换整个`struct`本身，如下面的代码所示：

```go
func (u *User) Birthday() {
    *u = User{Name: u.Name, Age: u.Age + 1}
   fmt.Println(u.Name, "turns", u.Age)
}
```

完整示例可在[`play.golang.org/p/3ugBEZqAood`](https://play.golang.org/p/3ugBEZqAood)中找到。

如果我们尝试更改指针的值而不是底层值，我们将编辑一个与在`main`中创建的对象无关的新对象，并且更改不会传播：

```go
func (u *User) Birthday() {
    u = &User{Name: u.Name, Age: u.Age + 1}
    fmt.Println(u.Name, "turns", u.Age)
}
```

完整示例可在[`play.golang.org/p/m8u2clKTqEU`](https://play.golang.org/p/m8u2clKTqEU)中找到。

Go 中的一些类型是自动按引用传递的。这是因为这些类型在内部被定义为包含指针的结构。这创建了一个类型列表，以及它们的内部定义：

| **类型** | **内部定义** |
| --- | --- |
| `map` |

```go
struct {
    m *internalHashtable
}
```

|

| `slice` |
| --- |

```go
struct {
    array *internalArray 
    len int
    cap int
}
```

|

| `channel` |
| --- |

```go
struct {
    c *internalChannel
}
```

|

# 理解流控制

为了控制应用程序的流程，Go 提供了不同的工具 - 一些语句如`if`/`else`，`switch`和`for`用于顺序场景，而`go`和`select`等其他语句用于并发场景。

# 条件

`if`语句验证一个二进制条件，并在条件为`true`时执行`if`块内的代码。当存在`else`块时，当条件为`false`时执行。该语句还允许在条件之前进行短声明，用`；`分隔。这个条件可以与`else if`语句链接，如下面的代码所示：

```go
if r := a%10; r != 0 { // if with short declaration
    if r > 5 {         // if without declaration 
        a -= r
    } else if r < 5 {  // else if statement
        a += 10 - r 
    }
} else {               // else statement
    a /= 10
}
```

另一个条件语句是`switch`。这允许短声明，就像`if`一样，然后是一个表达式。这样的表达式的值可以是任何类型（不仅仅是布尔类型），并且它与一系列`case`语句进行比较，每个`case`语句后面都跟着一段代码。第一个与表达式匹配的语句，如果`switch`和`case`条件相等，将执行其块。

如果在中断块的执行中存在`break`语句，但有一个`fallthrough`，则执行以下`case`块内的代码。一个称为`default`的特殊情况可以用来在没有满足条件的情况下执行其代码，如下面的代码所示：

```go
switch tier {                        // switch statement
case 1:                              // case statement
    fmt.Println("T-shirt")
    if age < 18{
        break                        // exits the switch block
    }
    fallthrough                      // executes the next case
case 2:
    fmt.Println("Mug")
    fallthrough                      // executes the next case 
case 3:
    fmt.Println("Sticker pack")    
default:                             // executed if no case is satisfied
    fmt.Println("no reward")
}
```

# 循环

`for`语句是 Go 中唯一的循环语句。这要求您指定三个表达式，用`；`分隔：

+   对现有变量进行短声明或赋值

+   在每次迭代之前验证的条件

+   在迭代结束时执行的操作

所有这些语句都是可选的，没有条件意味着它总是`true`。`break`语句中断循环的执行，而`continue`跳过当前迭代并继续下一个：

```go
for {                    // infinite loop
    if condition {
        break            // exit the loop
    }
}

for i < 0 {              // loop with condition
    if condition {
        continue         // skip current iteration and execute next    
    }
}

for i:=0; i < 10; i++ {  // loop with declaration, condition and operation 
}
```

当`switch`和`for`的组合嵌套时，`continue`和`break`语句将引用内部流控制语句。

外部循环或条件可以使用`name:`表达式进行标记，其中名称是其标识符，`loop`和`continue`都可以在后面跟着名称，以指定在哪里进行干预，如下面的代码所示：

```go
label:
    for i := a; i<a+2; i++ {
        switch i%3 {
        case 0:
            fmt.Println("divisible by 3")
            break label                          // this break the outer for loop
        default:
            fmt.Println("not divisible by 3")
        }
    }
```

# 探索内置函数

我们已经列出了一些用于初始化一些变量的内置函数，即`make`和`new`。现在，让我们逐个查看每个函数并了解它们的作用：

+   `func append(slice []Type, elems ...Type) []Type`: 此函数将元素追加到切片的末尾。如果底层数组已满，则在追加之前将内容重新分配到一个更大的切片中。

+   `func cap(v Type) int`: 返回数组、或者如果参数是切片则返回底层数组的元素数量。

+   `func close(c chan<- Type)`: 关闭一个通道。

+   `func complex(r, i FloatType) ComplexType`: 给定两个浮点数，返回一个复数。

+   `func copy(dst, src []Type) int`: 从一个切片复制元素到另一个切片。

+   `func delete(m map[Type]Type1, key Type)`: 从映射中删除一个条目。

+   `func imag(c ComplexType) FloatType`: 返回复数的虚部。

+   `func len(v Type) int`: 返回数组、切片、映射、字符串或通道的长度。

+   `func make(t Type, size ...IntegerType) Type`: 创建一个新的切片、映射或通道。

+   `func new(Type) *Type`: 返回指向指定类型变量的指针，并初始化为零值。

+   `func panic(v interface{})`: 停止当前 goroutine 的执行，并且如果没有被拦截，整个程序也会停止。

+   `func print(args ...Type)`: 将参数写入标准错误。

+   `func println(args ...Type)`: 将参数写入标准错误，并在末尾添加一个新行。

+   `func real(c ComplexType) FloatType`: 返回复数的实部。

+   `func recover() interface{}`: 停止 panic 序列并捕获 panic 值。

# 延迟、panic 和 recover

一个隐藏了很多复杂性但使得执行许多操作变得容易的非常重要的关键字是`defer`。这个关键字应用于函数、方法或闭包的执行，并使得它之前的函数在函数返回之前执行。一个常见且非常有用的用法是关闭资源。在成功打开资源后，延迟的关闭语句将确保它被执行，而不受退出点的影响，如下面的代码所示：

```go
f, err := os.Open("config.txt")
if err != nil {
    return err
}
defer f.Close() // it will be closed anyways

// do operation on f
```

在函数的生命周期内，所有延迟语句都被添加到一个列表中，并在退出之前按相反的顺序执行，从最后一个`defer`到第一个。

即使发生 panic，这些语句也会被执行，这就是为什么带有`recover`调用的延迟函数可以用于拦截相应 goroutine 中的 panic 并避免否则会终止应用程序的 panic。除了手动调用`panic`函数外，还有一组操作会引发 panic，包括以下操作：

+   访问负数或不存在的数组/切片索引（索引超出范围）

+   将整数除以`0`

+   向关闭的通道发送数据

+   对`nil`指针进行解引用（`nil`指针）

+   使用递归函数调用填充堆栈（堆栈溢出）

Panic 应该用于不可恢复的错误，这就是为什么在 Go 中错误只是值。恢复 panic 应该只是尝试在退出应用程序之前对该错误进行处理。如果发生了意外问题，那是因为它没有被正确处理或者缺少了一些检查。这代表了一个需要处理的严重问题，程序需要改变，这就是为什么它应该被拦截和解除。

# 并发模型

并发对于 Go 来说是如此核心，以至于它的两个基本工具只是关键字——`chan`和`go`。这是一种非常巧妙的方式，它隐藏了一个设计良好且实现简单易懂的并发模型的复杂性。

# 理解通道和 goroutine

通道是用于通信的，这就是为什么 Go 的口号是：

“不要通过共享内存来通信，而是通过通信来共享内存。”

通道用于共享数据，通常连接应用程序中的两个或多个执行线程，这使得可以发送和接收数据而不必担心数据安全性。Go 具有由运行时而不是操作系统管理的轻量级线程的实现，它们之间进行通信的最佳方式是通过使用通道。

创建一个新的 goroutine 非常简单 - 只需要使用`go`运算符，后面跟着一个函数执行。这包括方法调用和闭包。如果函数有任何参数，它们将在例程开始之前被评估。一旦开始，如果不使用通道，就无法保证来自外部作用域的变量更改会被同步：

```go
a := myType{}
go doSomething(a)     // function call
go func() {           // closure call
    // ...
}()                   // note that the closure is executed
go a.someMethod()     // method call
```

我们已经看到如何使用`make`函数创建一个新的通道。如果通道是无缓冲的（`0`容量），向通道发送数据是一个阻塞操作，它会等待另一个 goroutine 从同一个通道接收数据以解锁它。容量显示了通道在进行下一个发送操作之前能够容纳多少消息：

```go
unbuf := make(chan int)    // unbuffered channel
buf := make(chan int, 3)   // channel with size 3
```

为了向通道发送数据，我们可以使用`<-`运算符。如果通道在运算符的左边，那么这是一个发送操作，如果在右边，那么这是一个接收操作。从通道接收到的值可以被赋给一个变量，如下所示：

```go
var ch = make(chan int)
go func() {
    b := <-ch        // receive and assign
    fmt.Println(b)
}()
ch <- 10             // send to channel
```

使用`close()`函数可以关闭一个通道。这个操作意味着不能再向通道发送更多的值。这通常是发送者的责任。向关闭的通道发送数据会导致`panic`，这就是为什么应该由接收者来完成。此外，当从通道接收数据时，可以在赋值中指定第二个布尔变量。如果通道仍然打开，这将为真，以便接收者知道通道何时已关闭。

```go
var ch = make(chan int)
go func() {
    b, ok := <-ch        // channel open, ok is true
    b, ok = <-ch         // channel closed, ok is false
    b <- ch              // channel close, b will be a zero value
}()
ch <- 10                 // send to channel
close(ch)                // close the channel
```

有一个特殊的控制语句叫做`select`，它的工作方式与`switch`完全相同，但只能在通道上进行操作：

```go
var ch1 = make(chan int)
var ch2 = make(chan int)
go func() { ch1 <- 10 }
go func() { <-ch2 }
switch {            // the first operation that completes is selected
case a := <-ch1:
    fmt.Println(a)
case ch2 <- 20:
    fmt.Println(b)    
}
```

# 理解内存管理

Go 是垃圾收集的；它以计算成本管理自己的内存。编写高效的应用程序需要了解其内存模型和内部工作，以减少垃圾收集器的工作并提高总体性能。

# 栈和堆

内存被分为两个主要区域 - 栈和堆。应用程序入口函数（`main`）有一个栈，每个 goroutine 都有一个栈，它们存储在堆中。**栈**就像其名字所暗示的那样，是一个随着每个函数调用而增长的内存部分，在函数返回时会收缩。**堆**由一系列动态分配的内存区域组成，它们的生命周期不像栈中的项目那样事先定义；堆空间可以随时分配和释放。

所有超出定义它们的函数生存期的变量都存储在堆中，比如返回的指针。编译器使用一个叫做**逃逸分析**的过程来检查哪些变量进入堆中。可以使用`go tool compile -m`命令来验证这一点。

栈中的变量随着函数的执行而来而去。让我们看一个栈如何工作的实际例子：

```go
func main() {
    var a, b = 0, 1
    f1(a,b)
    f2(a)
}

func f1(a, b int) {
    c := a + b
    f2(c)
}

func f2(c int) {
    print(c)
}
```

我们有`main`函数调用一个名为`f1`的函数，它调用另一个名为`f2`的函数。然后，同一个函数直接被`main`调用。

当`main`函数开始时，栈会随着被使用的变量而增长。在内存中，这看起来像下表，每一列代表栈的伪状态，表示栈随时间变化的方式，从左到右：

| `main`调用 | `f1`调用 | `f2`调用 | `f2`返回 | `f1`返回 | `f2`调用 | `f2`返回 | `main`返回 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `main()` | `main()` | `main()` | `main()` | `main()` | `main()` | `main()` | // 空 |
| `a = 0` | `a = 0` | `a = 0` | `a = 0` | `a = 0` | `a = 0` | `a = 0` |  |
| `b = 1` | `b = 1` | `b = 1` | `b = 1` | `b = 1` | `b = 1` | `b = 1` |  |
|  | `f1()` | `f1()` | `f1()` |  | `f2()` |  |  |
|  | `a = 0` | `a = 0` | `a = 0` |  | `c = 0` |  |  |
|  | `b = 1` | `b = 1` | `b = 1` |  |  |  |  |
|  | `c = 1` | `c = 1` | `c = 1` |  |  |  |  |
|  |  | `f2()` |  |  |  |  |  |
|  |  | `c = 1` |  |  |  |  |  |

当调用`f1`时，堆栈再次增长，通过将`a`和`b`变量复制到新部分并添加新变量`c`来实现。`f2`也是同样的情况。当`f2`返回时，堆栈通过摆脱函数及其变量来缩小，这就是`f1`完成时发生的情况。当直接调用`f2`时，它会通过回收用于`f1`的相同内存部分来再次增长。

垃圾收集器负责清理堆中未引用的值，因此避免在其中存储数据是降低**垃圾收集器**（**GC**）工作量的好方法，这会在 GC 运行时导致应用程序性能略微下降。

# Go 中 GC 的历史

GC 负责释放堆中任何栈中未引用的区域。这最初是用 C 编写的，并且具有*停止世界*行为。程序会在一小段时间内停止，释放内存，然后恢复其流程。

Go 1.4 开始将运行时，包括垃圾收集器，转换为 Go。将这些部分翻译成 Go 为更容易的优化奠定了基础，这在 1.5 版本中已经开始，其中 GC 变得更快，并且可以与其他 goroutine 并发运行。

从那时起，该过程进行了大量的优化和改进，成功将 GC 时间减少了几个数量级。

# 构建和编译程序

现在我们已经快速概述了所有语言特性和功能，我们可以专注于如何运行和构建我们的应用程序。

# 安装

在 Go 中，有不同的命令来构建软件包和应用程序。第一个是`go install`，后面跟着路径或软件包名称，它将在`$GOPATH`内的`pkg`目录中创建软件包的编译版本。

所有编译的软件包都按操作系统和架构组织，这些都存储在`$GOOS`和`$GOARCH`环境变量中。可以使用`go env`命令查看这些设置，以及其他信息，比如编译标志：

```go
$ go env
GOARCH="amd64"
...
GOOS="linux"
GOPATH="/home/user/go"
...
GOROOT="/usr/lib/go-1.12"
...
```

对于当前的架构和操作系统，所有编译的软件包将被放置在`$GOOS_$GOARCH`子目录中：

```go
$ ls /home/user/go/pkg/
linux_amd64
```

如果软件包名称是`main`并且包含一个`main`函数，该命令将生成一个可执行的二进制文件，该文件将存储在`$GOPATH/bin`中。如果软件包已经安装并且源文件没有更改，它将不会被重新编译，这将在第一次编译后显著加快构建时间。

# 构建

也可以使用`go build`命令在特定位置构建二进制文件。可以使用`-o`标志定义特定的输出文件，否则将在工作目录中构建，使用软件包名称作为二进制文件名：

```go
# building the current package in the working directory
$ go build . 

# building the current package in a specific location
$ go build . -o "/usr/bin/mybinary"
```

执行`go build`命令时，参数可以是以下之一：

+   一个软件包作为相对路径（比如`go build .`用于当前软件包或`go build ../name`）

+   一个软件包作为绝对路径（`go build some/package`）将在`$GOPATH`中查找

+   一个特定的 Go 源文件（`go build main.go`）

后一种情况允许您构建一个位于`$GOPATH`之外的文件，并且将忽略同一目录中的任何其他源文件。

# 运行

还有第三个命令，它类似于构建，但也运行二进制文件。它使用`build`命令使用临时目录作为输出创建二进制文件，并即时执行二进制文件：

```go
$ go run main.go
main output line 1
main output line 2

$
```

当您对源代码进行更改时，可以使用运行而不是构建或安装。如果代码相同，最好是构建一次，然后多次执行。

# 摘要

在本章中，我们看了一些 Go 的历史以及它当前的优缺点。在了解了命名空间后，我们探讨了包系统和导入的工作方式，以及基本、复合和用户定义类型的类型系统。

我们通过查看变量的声明和初始化方式，允许类型之间的操作，如何将变量转换为其他类型，以及如何查看接口的基础类型，来重点关注变量。我们看到了作用域和屏蔽的工作方式，以及常量和变量之间的区别。之后，我们进入了函数，它是一种一等类型，以及每个签名代表不同类型的方式。然后，我们了解了方法实际上是伪装成函数并附加到允许自定义类型满足接口的类型。

此外，我们学习了如何使用诸如`if`、`for`和`switch`之类的语句来控制应用程序流程。我们分析了各种控制语句和循环语句之间的区别，并查看了每个内置函数的作用。然后，我们看到了基本并发是如何通过通道和 goroutine 工作的。最后，我们对 Go 的内部内存分配方式有了一些了解，以及其垃圾收集器的历史和性能，以及如何构建、安装和运行 Go 二进制文件。

在下一章中，我们将看到如何通过与文件系统交互将其中一些内容付诸实践。

# 问题

1.  导出符号和未导出符号之间有什么区别？

1.  自定义类型为什么重要？

1.  短声明的主要限制是什么？

1.  什么是作用域，它如何影响变量屏蔽？

1.  如何访问一个方法？

1.  解释一下一系列`if`/`else`和`switch`之间的区别。

1.  在典型的用例中，通常谁负责关闭通道？

1.  什么是逃逸分析？
