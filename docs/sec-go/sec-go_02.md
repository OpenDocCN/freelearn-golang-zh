# 第二章：Go 编程语言

在深入研究使用 Go 进行安全性的更复杂示例之前，建立坚实的基础非常重要。本章概述了 Go 编程语言，以便您具备后续示例所需的知识。

本章不是 Go 编程语言的详尽论述，但将为您提供主要功能的扎实概述。本章的目标是为您提供必要的信息，以便在以前从未使用过 Go 的情况下理解和遵循源代码。如果您已经熟悉 Go，本章应该是对您已经知道的内容的快速简单回顾，但也许您会学到一些新的信息。

本章专门涵盖以下主题：

+   Go 语言规范

+   Go 游乐场

+   Go 之旅

+   关键字

+   关于源代码的注释

+   注释

+   类型

+   控制结构

+   延迟

+   包

+   类

+   Goroutines

+   获取帮助和文档

# Go 语言规范

整个 Go 语言规范可以在[`golang.org/ref/spec`](https://golang.org/ref/spec)上找到。本章中的大部分信息来自规范，因为这是语言的真正文档。这里的其他信息是短小的示例、提示、最佳实践和我在使用 Go 期间学到的其他内容。

# Go 游乐场

Go 游乐场是一个网站，您可以在其中编写和执行 Go 代码，而无需安装任何东西。在游乐场中，[`play.golang.org`](https://play.golang.org)，您可以测试代码片段以探索语言，并尝试理解语言的工作原理。它还允许您通过创建存储代码片段的唯一 URL 来分享您的片段。通过游乐场分享代码可能比纯文本片段更有帮助，因为它允许读者实际执行代码并调整源代码，以便在对其工作原理有任何疑问时进行实验：

！[](img/d5514c8a-7253-4641-8b61-3e02ebcac15e.png)

上面的截图显示了在游乐场中运行的简单程序。顶部有按钮可以运行、格式化、添加导入语句和与他人共享代码。

# Go 之旅

Go 团队提供的另一个资源是*Go 之旅*。这个网站，[`tour.golang.org`](https://tour.golang.org)，建立在前一节提到的游乐场之上。这次旅行是我对这种语言的第一次介绍，当我完成它时，我感到有能力开始处理 Go 项目。它会逐步引导您了解语言，并提供工作代码示例，以便您可以运行和修改代码以熟悉语言。这是向新手介绍 Go 的实用方式。如果您根本没有使用过 Go，我鼓励您去看一看。

！[](img/155646d8-315b-4b13-aa31-c7be08feb713.png)

上面的截图显示了游览的第一页。在右侧，您将看到一个嵌入式的小游乐场，其中包含左侧显示的短课程相关的代码示例。每节课都有一个简短的代码示例，您可以运行和调整。

# 关键字

为了强调 Go 的简单性，这里列出了其 25 个关键字的详细说明。如果您熟悉其他编程语言，您可能已经了解其中大部分。关键字根据其用途分组在一起进行检查。

**数据类型**：

| `var` | 这定义了一个新变量 |
| --- | --- |
| `const` | 这定义一个不变的常量值 |
| `type` | 这定义了一个新数据类型 |
| `struct` | 这定义了一个包含多个变量的新结构化数据类型 |
| `map` | 这定义了一个新的映射或哈希变量 |
| `interface` | 这定义了一个新接口 |

**函数**：

| `func` | 这定义了一个新函数 |
| --- | --- |
| `return` | 这退出一个函数，可选地返回值 |

**包**：

| `import`  | 这在当前包中导入外部包 |
| --- | --- |
| `package` | 这指定文件属于哪个包 |

**程序流**：

| `if` | 如果条件为真，则使用此分支执行 |
| --- | --- |
| `else` | 如果条件不成立，则使用此分支 |
| `goto` | 这用于直接跳转到标签；它很少使用，也不鼓励使用 |

**Switch 语句**：

| `switch` | 这用于基于条件进行分支 |
| --- | --- |
| `case` | 这定义了`switch`语句的条件 |
| `default` | 这定义了当没有匹配的情况时的默认执行 |
| `fallthrough` | 这用于继续执行下一个 case |

**迭代**：

| `for` | `for`循环可以像在 C 中一样使用，其中提供三个表达式：初始化程序、条件和增量器。在 Go 中，没有`while`循环，`for`关键字承担了`for`和`while`的角色。如果传递一个表达式，条件，`for`循环可以像`while`循环一样使用。 |
| --- | --- |
| `range` | `range`关键字与`for`循环一起用于迭代 map 或 slice。 |
| `continue` | `continue`关键字将跳过当前循环中剩余的任何执行，并直接跳转到下一个迭代。 |
| `break` | `break`关键字将立即完全退出`for`循环，跳过任何剩余的迭代。 |

**并发**：

| `go` | Goroutines 是内置到语言中的轻量级线程。您只需在函数调用前面加上`go`关键字，Go 就会在单独的线程中执行该函数调用。 |
| --- | --- |
| `chan` | 为了在线程之间通信，使用通道。通道用于发送和接收特定数据类型。它们默认是阻塞的。 |
| `select` | `select`语句允许通道以非阻塞方式使用。 |

**便利**：

| `defer` | `defer`关键字是一个相对独特的关键字，在其他语言中我以前没有遇到过。它允许您指定在周围函数返回时稍后调用的函数。当您想要确保当前函数结束时执行某种清理操作，但不确定何时或何地它可能返回时，它非常有用。一个常见的用例是延迟文件关闭。 |
| --- | --- |

# 关于源代码的注释

Go 源代码文件应该有`.go`扩展名。Go 文件的源代码以 UTF-8 Unicode 编码。这意味着您可以在代码中使用任何 Unicode 字符，比如在字符串中硬编码日语字符。

分号在行尾是可选的，通常省略。只有在分隔单行上的多个语句或表达式时才需要分号。

Go 确实有一个代码格式化标准，可以通过在源代码文件上运行`go fmt`来轻松遵守。应该遵循代码格式化，但不像 Python 那样严格由编译器执行确切的格式化以正确执行。

# 注释

注释遵循 C++风格，允许双斜杠和斜杠星号包装样式：

```go
// Line comment, everything after slashes ignored
/* General comment, can be in middle of line or span multiple lines */
```

# 类型

内置数据类型的命名相当直观。Go 带有一组具有不同位长度的整数和无符号整数类型。还有浮点数、布尔值和字符串，这应该不足为奇。

有一些类型，如符文，在其他语言中不常见。本节涵盖了所有不同的类型。

# 布尔

布尔类型表示真或假值。有些语言不提供`bool`类型，您必须使用整数或定义自己的枚举，但 Go 方便地预先声明了`bool`类型。`true`和`false`常量也是预定义的，并且以全小写形式使用。以下是创建布尔值的示例：

```go
var customFlag bool = false  
```

`bool`类型并不是 Go 独有的，但关于布尔类型的一个有趣的小知识是，它是唯一以一个人命名的类型。乔治·布尔生于 1815 年，逝世于 1864 年，写了《思维的法则》，在其中描述了布尔代数，这是所有数字逻辑的基础。`bool`类型在 Go 中非常简单，但其名称背后的历史非常丰富。

# 数字

主要的数字数据类型是整数和浮点数。Go 还提供了复数类型、字节类型和符文。以下是 Go 中可用的数字数据类型。

# 通用数字

这些通用类型可以在您不特别关心数字是 32 位还是 64 位时使用。将自动使用最大可用大小，但将与 32 位和 64 位处理器兼容。

+   `uint`：这是一个 32 位或 64 位的无符号整数

+   `int`：这是一个带有与`uint`相同大小的有符号整数

+   `uintptr`：这是一个无符号整数，用于存储指针值

# 特定数字

这些数字类型指定了位长度以及它是否具有符号位来确定正负值。位长度将确定最大范围。有符号整数的范围会减少一个位，因为最后一位保留给了符号。

# 无符号整数

在没有数字的情况下使用`uint`通常会选择系统的最大大小，通常为 64 位。您还可以指定这四种特定的`uint`大小之一：

+   `uint8`：无符号 8 位整数（0 至 255）

+   `uint16`：无符号 16 位整数（0 至 65535）

+   `uint32`：无符号 32 位整数（0 至 4294967295）

+   `uint64`：无符号 64 位整数（0 至 18446744073709551615）

# 有符号整数

与无符号整数一样，您可以单独使用`int`来选择最佳默认大小，或者指定这四种特定的`int`大小之一：

+   `int8`：8 位整数（-128 至 127）

+   `int16`：16 位整数（-32768 至 32767）

+   `int32`：32 位整数（-2147483648 至 2147483647）

+   `int64`：64 位整数（-9223372036854775808 至 9223372036854775807）

# 浮点数

浮点类型没有通用类型，必须是以下两种选项之一：

+   `float32`：IEEE-754 32 位浮点数

+   `float64`：IEEE-754 64 位浮点数

# 其他数字类型

Go 还为高级数学应用提供了复数类型，以及一些别名以方便使用：

+   `complex64`：具有`float32`实部和虚部的复数

+   `complex128`：具有`float64`实部和虚部的复数

+   `byte`：`uint8`的别名

+   `rune`：`int32`的别名

您可以以十进制、八进制或十六进制格式定义数字。十进制或十进制数字不需要前缀。八进制或八进制数字应以零为前缀。十六进制或十六进制数字应以零和 x 为前缀。

您可以在[`en.wikipedia.org/wiki/Octal`](https://en.wikipedia.org/wiki/Octal)上了解更多八进制数字系统，十进制数字在[`en.wikipedia.org/wiki/Decimal`](https://en.wikipedia.org/wiki/Decimal)，十六进制数字在[`en.wikipedia.org/wiki/Hexadecimal`](https://en.wikipedia.org/wiki/Hexadecimal)。

请注意，数字被存储为整数，它们之间没有区别，除了它们在源代码中的格式化方式。在处理二进制数据时，八进制和十六进制可能很有用。以下是如何定义整数的简短示例：

```go
package main

import "fmt"

func main() {
   // Decimal for 15
   number0 := 15

   // Octal for 15
   number1 := 017 

   // Hexadecimal for 15
   number2 := 0x0F

   fmt.Println(number0, number1, number2)
} 
```

# 字符串

Go 还提供了`string`类型以及一个`strings`包，其中包含一套有用的函数，如`Contains()`，`Join()`，`Replace()`，`Split()`，`Trim()`和`ToUpper()`。此外还有一个专门用于将各种数据类型转换为字符串的`strconv`包。您可以在[`golang.org/pkg/strings/`](https://golang.org/pkg/strings/)上阅读有关`strings`包的更多信息，以及在[`golang.org/pkg/strconv/`](https://golang.org/pkg/strconv/)上阅读有关`strconv`包的更多信息。

双引号用于字符串。单引号仅用于单个字符或符文，而不是字符串。可以使用长形式或使用声明和分配运算符的短形式来定义字符串。您还可以使用`` ` ``（反引号）符号，用于封装跨多行的字符串。以下是字符串用法的简短示例:

```go

package main
import "fmt"
func main() {
   // 长形式分配
   var myText = "test string 1"
   // 短形式分配
   myText2 := "test string 2"
   // 多行字符串
   myText3 := `long string
   spans multiple
   lines`
   fmt.Println(myText)
   fmt.Println(myText2)
   fmt.Println(myText3)
}

```

# 数组

数组由特定类型的序列化元素组成。可以为任何数据类型创建一个数组。数组的长度是不可变的，必须在声明时指定。数组很少直接使用，而是在下一节中介绍的切片类型中大多数使用。数组始终是一维的，但可以创建一个数组的数组来创建多维对象。

要创建一个包含`128`个字节的数组，可以使用以下语法：

```go

var myByteArray [128]byte

```

数组的各个元素可以通过基于`0`的数字索引进行访问。例如，要获取字节数组的第五个元素，语法如下：

```go

singleByte := myByteArray[4]

```

# 切片

切片使用数组作为基础数据类型。主要优点是切片可以调整大小，而数组不行。将切片视为对基础数组的查看窗口。**容量**指的是基础数组的大小，以及切片的最大可能长度。切片的**长度**指当前长度，可以调整大小。

使用`make()`函数创建切片。`make()`函数将创建指定类型、长度和容量的切片。在创建切片时，`make()`函数可以有两种方式。只有两个参数时，长度和容量相同。有三个参数时，可以指定一个比长度大的最大容量。以下是两种`make()`函数声明：

```go

make([]T, lengthAndCapacity)
make([]T, length, capacity)

```

可以创建具有容量和长度为`0`的`nil`切片。`nil`切片没有关联的基础数组。以下是演示如何创建和检查切片的简短示例程序：

```go

package main
import "fmt"
func main() {
   // 创建一个 nil 切片
   var mySlice []byte
   // 创建长度为 8，最大容量为 128 的字节切片
   mySlice = make([]byte, 8, 128)
   // 切片的最大容量
   fmt.Println("Capacity:", cap(mySlice))
   // 切片的当前长度
   fmt.Println("Length:", len(mySlice))
}

```

也可以使用内置`append()`函数向切片追加元素。

`Append`可以一次添加一个或多个元素。必要时，基础数组将调整大小。这意味着切片的最大容量可以增加。当一个切片增加其基础容量时，创建一个更大的基础数组时，将创建具有一些额外空间的数组。这意味着如果超过一个切片的容量，可能会将数组大小增加四倍。这样做是为了使基础数组有空间增长，以减少重新调整大小基础数组的次数，这可能需要移动内存以容纳更大的数组。每次只需添加一个元素就重新调整大小数组可能会很昂贵。切片机制将自动确定最佳的调整大小。

以下代码示例提供了使用切片的各种示例：

```go

package main
import "fmt"
func main() {
   var mySlice []int // nil slice
   // 在 nil 切片上可以使用附加功能。
   // 由于 nil 切片的容量为零，并且具有
   // 没有基础数组，它将创建一个。
   mySlice = append(mySlice, 1, 2, 3, 4, 5)
   // 可以从切片中访问单个元素
   // 就像使用方括号运算符一样，就像数组一样。
   firstElement := mySlice[0]
   fmt.Println("First element:", firstElement)
   // 仅获取第二个和第三个元素，请使用：
   subset := mySlice[1:4]
   fmt.Println(subset)
   // 要获取切片的全部内容，除了
   // 第一个元素，使用：
   subset = mySlice[1:]
   fmt.Println(subset)
   // 要获取切片的全部内容，除了
   // 最后一个元素，使用：
   subset = mySlice[0 : len(mySlice)-1]
   fmt.Println(subset)
   // 要复制切片，请使用 copy()函数。
   // 如果您使用等号将一个切片分配给另一个切片，
   // 切片将指向相同的内存位置，
   // 更改一个会更改两个切片。
   slice1 := []int{1, 2, 3, 4}
   slice2 := make([]int, 4)
   // 在内存中创建一个唯一的副本
   copy(slice2, slice1)
   // 更改一个不应影响另一个
   slice2[3] = 99
   fmt.Println(slice1)
   fmt.Println(slice2)
}

```

# 结构体

在 Go 中，结构体或数据结构是一组变量。变量可以是不同类型的。我们将看一个创建自定义结构体类型的示例。

Go 使用基于大小写的作用域来声明变量为`public`或`private`。大写的变量和方法是公开的，可以从其他包中访问。小写的值是私有的，只能在同一包中访问。

以下示例创建了一个名为`Person`的简单结构体，以及一个名为`Hacker`的结构体。`Hacker`类型在其中嵌入了一个`Person`类型。然后分别创建了每种类型的实例，并将有关它们的信息打印到标准输出：

```go

package main
import "fmt"
func main() {
   // 定义一个 Person 类型。两个字段都是公共的
   type Person struct {
      Name string
      Age  int
   }
   // 创建一个 Person 对象并存储指向它的指针
   nanodano := &Person{Name: "NanoDano", Age: 99}
   fmt.Println(nanodano)
   // 结构也可以嵌入在其他结构中。
   // 这通过简单地存储
   // 另一个变量作为数据类型。
   type Hacker struct {
      Person           Person
      FavoriteLanguage string
   }
   fmt.Println(nanodano)
   hacker := &Hacker{
      Person:           *nanodano,
      FavoriteLanguage: "Go",
   }
   fmt.Println(hacker)
   fmt.Println(hacker.Person.Name)
   fmt.Println(hacker)
}

```

你可以通过将它们的名称以小写字母开头来创建*私有*变量。我用引号是因为私有变量与其他语言中的工作方式略有不同。隐私工作在包级别而不是*类*或类型级别。

# 指针

Go 提供了一个指针类型，用于存储特定类型数据的内存位置。指针可以被用来通过引用传递一个结构体给函数，而不需要创建副本。这也允许函数就地修改对象。

Go 不允许指针算术。指针被认为是*安全*的，因为 Go 甚至不定义指针类型上的加法运算符。它们只能用于引用现有对象。

这个示例演示了基本的指针用法。它首先创建一个整数，然后创建一个指向该整数的指针。然后打印指针的数据类型，指针中存储的地址，以及被指向的数据的值：

```go

package main
import (
   "fmt"
   "reflect"
)
func main() {
   myInt := 42
   intPointer := &myInt
   fmt.Println(reflect.TypeOf(intPointer))
   fmt.Println(intPointer)
   fmt.Println(*intPointer)
}

```

# 函数

使用`func`关键字定义函数。函数可以有多个参数。所有参数都是位置参数，没有命名参数。Go 支持可变参数，允许有未知数量的参数。在 Go 中，函数是一等公民，并且可以匿名使用并作为变量返回。Go 还支持从函数返回多个值。下划线可以用于忽略返回变量。

所有这些示例都在以下代码来源中演示：

```go

package main
import "fmt"
// 没有参数的函数
func sayHello() {
   fmt.Println("Hello.")
}
// 带有一个参数的函数
func greet(name string) {
   fmt.Printf("Hello, %s.\n", name)
}
// 具有相同类型的多个参数的函数
func greetCustom(name, greeting string) {
   fmt.Printf("%s, %s.\n", greeting, name)
}
// 变参参数，无限参数
func addAll(numbers ...int) int {
   sum := 0
   for _, number := range numbers {
      sum += number
   }
   return sum
}
// 具有多个返回值的函数
// 由括号封装的多个值
func checkStatus() (int, error) {
   return 200, nil
}
// 将类型定义为函数，以便可以使用
// 作为返回类型
type greeterFunc func(string)
// 生成并返回一个函数
func generateGreetFunc(greeting string) greeterFunc {
   return func(name string) {
      fmt.Printf("%s, %s.\n", greeting, name)
   }
}
func main() {
   sayHello()
   greet("NanoDano")
   greetCustom("NanoDano", "Hi")
   fmt.Println(addAll(4, 5, 2, 3, 9))
   russianGreet := generateGreetFunc("Привет")
   russianGreet("NanoDano")
   var stringToIntMap map[string]int
   fmt.Println(statusCode, err)
}

```

# 接口

接口是一种特殊类型，它定义了一系列函数签名。你可以把接口看作是在说，“一个类型必须实现函数 X 和函数 Y 来满足这个接口。” 如果你创建了任何类型并实现了满足接口所需的函数，那么你的类型可以在期望接口的任何地方使用。你不必指定你正在尝试满足一个接口，编译器将确定它是否满足要求。

你可以为你的自定义类型添加任意多的其他函数。接口定义了所需的函数，但这并不意味着你的类型仅限于实现这些函数。

最常用的接口是`error`接口。`error`接口只需要实现一个函数，即一个名为`Error()`的函数，该函数返回一个带有错误消息的字符串。以下是接口定义：

```go
type error interface {
   Error() string
} 

```

这使得你很容易实现自己的错误接口。这个示例创建了一个`customError`类型，然后实现了满足接口所需的`Error()`函数。然后，创建了一个示例函数，该函数返回自定义错误：

```go

package main

import "fmt"

// Define a custom type that will
// be used to satisfy the error interface
type customError struct {
   Message string
}

// Satisfy the error interface
// by implementing the Error() function
// which returns a string
func (e *customError) Error() string {
   return e.Message
}

// Sample function to demonstrate
// how to use the custom error
func testFunction() error {
   if true != false { // Mimic an error condition
      return &customError{"Something went wrong."}
   }
   return nil
}

func main() {
   err := testFunction()
   if err != nil {
      fmt.Println(err)
   }
} 
```

其他经常使用的接口是 `Reader` 和 `Writer` 接口。每个接口只需要实现一个函数以满足接口要求。这里的一个重大好处是你可以创建自己的自定义类型，以某种任意的方式读取和写入数据。接口不关心实现细节。接口不会在乎你是在读写硬盘、网络连接、内存中的存储还是 `/dev/null`。只要你实现了所需的函数签名，你就可以在任何使用接口的地方使用你的类型。下面是 `Reader` 和 `Writer` 接口的定义：

```go

type Reader interface {
   Read(p []byte) (n int, err error)
} 
 
type Writer interface {
   Write(p []byte) (n int, err error)
} 

```

# Map

Map 是一个存储键值对的哈希表或字典。键和值可以是任何数据类型，包括映射本身，从而创建多个维度。

顺序不受保证。你可以多次迭代一个映射，并且可能会不同。此外，映射不是并发安全的。如果必须在线程之间共享映射，请使用互斥锁。

这里是一些示例映射用法：

```go

package main

import (
   "fmt"
   "reflect"
)

func main() {
   // Nil maps will cause runtime panic if used 
   // without being initialized with make()
   var intToStringMap map[int]string
   var stringToIntMap map[string]int
   fmt.Println(reflect.TypeOf(intToStringMap))
   fmt.Println(reflect.TypeOf(stringToIntMap))

   // Initialize a map using make
   map1 := make(map[string]string)
   map1["Key Example"] = "Value Example"
   map1["Red"] = "FF0000"
   fmt.Println(map1)

   // Initialize a map with literal values
   map2 := map[int]bool{
      4:  false,
      6:  false,
      42: true,
   }

   // Access individual elements using the key
   fmt.Println(map1["Red"])
   fmt.Println(map2[42])
   // Use range to iterate through maps
   for key, value := range map2 {
      fmt.Printf("%d: %t\n", key, value)
   }

} 
```

# Channel

通道用于线程之间通信。通道是**先进先出**（**FIFO**）队列。你可以将对象推送到队列并异步从前端拉取。每个通道只能支持一个数据类型。通道默认是阻塞的，但可以通过 `select` 语句使其成为非阻塞。像切片和映射一样，通道必须在使用之前用 `make()` 函数初始化。

在 Go 中的格言是 *不要通过共享内存来通信；而是通过通信来共享内存*。在[`blog.golang.org/share-memory-by-communicating`](https://blog.golang.org/share-memory-by-communicating)上阅读更多关于这一哲学的内容。

下面是一个演示基本通道使用的示例程序：

```go

package main

import (
   "log"
   "time"
)

// Do some processing that takes a long time
// in a separate thread and signal when done
func process(doneChannel chan bool) {
   time.Sleep(time.Second * 3)
   doneChannel <- true
}

func main() {
   // Each channel can support one data type.
   // Can also use custom types
   var doneChannel chan bool

   // Channels are nil until initialized with make
   doneChannel = make(chan bool)

   // Kick off a lengthy process that will
   // signal when complete
   go process(doneChannel)

   // Get the first bool available in the channel
   // This is a blocking operation so execution
   // will not progress until value is received
   tempBool := <-doneChannel
   log.Println(tempBool)
   // or to simply ignore the value but still wait
   // <-doneChannel

   // Start another process thread to run in background
   // and signal when done
   go process(doneChannel)

   // Make channel non-blocking with select statement
   // This gives you the ability to continue executing
   // even if no message is waiting in the channel
   var readyToExit = false
   for !readyToExit {
      select {
      case done := <-doneChannel:
         log.Println("Done message received.", done)
         readyToExit = true
      default:
         log.Println("No done signal yet. Waiting.")
         time.Sleep(time.Millisecond * 500)
      }
   }
} 
```

# 控制结构

控制结构用于控制程序执行的流程。最常见的形式是 `if` 语句、`for` 循环和 `switch` 语句。Go 也支持 `goto` 语句，但应保留用于极端性能情况，不应经常使用。让我们简要地看一下这些以了解语法。

# if

`if` 语句有 `if`、`else if` 和 `else` 子句，就像大多数其他语言一样。 Go 的一个有趣特性是能够在条件之前放置语句，创建在 `if` 语句完成后被丢弃的临时变量。

这个示例演示了使用 `if` 语句的各种方式：

```go

package main

import (
   "fmt"
   "math/rand"
)

func main() {
   x := rand.Int()

   if x < 100 {
      fmt.Println("x is less than 100.")
   }

   if x < 1000 {
      fmt.Println("x is less than 1000.")
   } else if x < 10000 {
      fmt.Println("x is less than 10,000.")
   } else {
      fmt.Println("x is greater than 10,000")
   }

   fmt.Println("x:", x)

   // You can put a statement before the condition 
   // The variable scope of n is limited
   if n := rand.Int(); n > 1000 {
      fmt.Println("n is greater than 1000.")
      fmt.Println("n:", n)
   } else {
      fmt.Println("n is not greater than 1000.")
      fmt.Println("n:", n)
   }
   // n is no longer available past the if statement
```

# for

`for` 循环有三个组件，可以像在 C 或 Java 中一样使用 `for` 循环。Go 没有 `while` 循环，因为当与单个条件一起使用时，`for` 循环起到相同的作用。请参考以下示例以获得更多的清晰度：

```go

package main

import (
   "fmt"
)

func main() {
   // Basic for loop
   for i := 0; i < 3; i++ {
      fmt.Println("i:", i)
   }

   // For used as a while loop
   n := 5
   for n < 10 {
      fmt.Println(n)
      n++
   }
} 
```

# range

`range`关键字用于遍历切片、映射或其他数据结构。`range`关键字与`for`循环结合使用，对可迭代的数据结构进行操作。`range`关键字返回键和值变量。以下是使用`range`关键字的一些基本示例：

```go

package main

import "fmt"

func main() {
   intSlice := []int{2, 4, 6, 8}
   for key, value := range intSlice {
      fmt.Println(key, value)
   }

   myMap := map[string]string{
      "d": "Donut",
      "o": "Operator",
   }

   // Iterate over a map
   for key, value := range myMap {
      fmt.Println(key, value)
   }

   // Iterate but only utilize keys
   for key := range myMap {
      fmt.Println(key)
   }

   // Use underscore to ignore keys
   for _, value := range myMap {
      fmt.Println(value)
   }
} 
```

# switch、case、fallthrough 和 default

`switch`语句允许您根据变量的状态分支执行。它类似于 C 和其他语言中的`switch`语句。

默认情况下没有`fallthrough`。这意味着一旦到达一个情况的末尾，代码就会完全退出`switch`语句，除非提供了显式的`fallthrough`命令。如果没有匹配到任何情况，则可以提供一个`default`情况。

您可以在要切换的变量前放置一个语句，例如`if`语句。这会创建一个作用域限于`switch`语句的变量。

此示例演示了两个`switch`语句。第一个使用硬编码的值，并包含一个`default`情况。第二个`switch`语句使用了一种允许在第一行中包含语句的替代语法：

```go

package main

import (
   "fmt"
   "math/rand"
)

func main() {
   x := 42

   switch x {
   case 25:
      fmt.Println("X is 25")
   case 42:
      fmt.Println("X is the magical 42")
      // Fallthrough will continue to next case
      fallthrough
   case 100:
      fmt.Println("X is 100")
   case 1000:
      fmt.Println("X is 1000")
   default:
      fmt.Println("X is something else.")
   }

   // Like the if statement a statement
   // can be put in front of the switched variable
   switch r := rand.Int(); r {
   case r % 2:
      fmt.Println("Random number r is even.")
   default:
      fmt.Println("Random number r is odd.")
   }
   // r is no longer available after the switch statement
} 
```

# 跳转

Go 语言确实有`goto`语句，但很少使用。使用一个名称和一个冒号创建一个标签，然后使用`goto`关键字*跳转*到它。这是一个基本示例：

```go

package main

import "fmt"

func main() {

   goto customLabel

   // Will never get executed because
   // the goto statement will jump right
   // past this line
   fmt.Println("Hello")

   customLabel:
   fmt.Println("World")
} 
```

# 延迟

通过延迟一个函数，它会在当前函数退出时运行。这是一种方便的方式，可以确保一个函数在退出之前被执行，这对于清理或关闭文件很有用。这很方便，因为一个延迟的函数会在周围函数的任何退出处被执行，如果有多个返回位置的话。

常见用例是延迟调用关闭文件或数据库连接。在打开文件后，您可以延迟调用关闭。这将确保文件在函数退出时关闭，即使有多个返回语句，您也不能确定当前函数何时何地退出。

此示例演示了`defer`关键字的一个简单用例。它创建一个文件，然后延迟调用`file.Close()`：

```go

package main

import (
   "log"
   "os"
)

func main() {

   file, err := os.Create("test.txt")
   if err != nil {
      log.Fatal("Error creating file.")
   }
   defer file.Close()
   // It is important to defer after checking the errors.
   // You can't call Close() on a nil object
   // if the open failed.

   // ...perform some other actions here...

   // file.Close() will be called before final exit
} 
```

一定要正确检查和处理错误。如果使用空指针，则`defer`调用会导致恐慌。

还要明白延迟函数是在周围函数退出时运行的。如果在`for`循环中放置一个`defer`调用，它将不会在每个`for`循环迭代结束时被调用。

# 包

包只是目录。每个目录都是一个包。创建子目录会创建一个新包。没有子包会导致一个平坦的层次结构。子目录仅用于组织代码。

包应该存储在您的`$GOPATH`变量的`src`文件夹中。

包名应该与文件夹名匹配，或者命名为`main`。一个`main`包意味着它不打算被导入到另一个应用程序中，而是打算编译并作为程序运行。使用`import`关键字导入包。

你可以单独导入包：

```go

import "fmt"

```

或者，你可以通过用括号包裹多个包来一次性导入多个包：

```go

import (
   "fmt"
   "log"
) 
```

# 类

从技术上讲，Go 并没有类，但有几个微妙的区别使其不被称为面向对象的语言。概念上，我认为它是一种面向对象的编程语言，尽管仅支持最基本的面向对象语言特性。它不具备许多人们对面向对象编程所熟悉的所有特性，比如继承和多态性，而是用其他特性如嵌入类型和接口来替代。也许你可以把它称为一个*微类*系统，因为它是一个最简化实现，没有额外的特性或负担，这取决于你的角度。

本书中，术语*对象*和*类*可能会被用来说明一个概念，使用熟悉的术语，但请注意这些在 Go 中并不是正式术语。类型定义与操作该类型的函数结合起来类似于类，而对象是类型的一个实例。

# 继承

Go 中没有继承，但可以嵌入类型。这里有一个`Person`和`Doctor`类型的示例，`Doctor`类型嵌入了`Person`类型。与直接继承`Person`的行为不同，它将`Person`对象作为变量存储，从而带来了其预期的`Person`方法和属性：  

```go

package main

import (
   "fmt"
   "reflect"
)

type Person struct {
   Name string
   Age  int
} 

type Doctor struct {
   Person         Person
   Specialization string
}

func main() {
   nanodano := Person{
      Name: "NanoDano",
      Age:  99,
   } 

   drDano := Doctor{
      Person:         nanodano,
      Specialization: "Hacking",
   }

   fmt.Println(reflect.TypeOf(nanodano))
   fmt.Println(nanodano)
   fmt.Println(reflect.TypeOf(drDano))
   fmt.Println(drDano)
} 
```

# 多态性

Go 中没有多态性，但可以使用接口创建可以被多个类型使用的通用抽象。接口定义了一个或多个必须满足以兼容接口的方法声明。接口在本章的前面已经介绍过。

# 构造函数

Go 中没有构造函数，但有类似于初始化对象的工厂函数`New()`。你只需创建一个名为`New()`的函数，返回你的数据类型。下面是一个示例：

```go

package main

import "fmt"

type Person struct {
   Name string
}

func NewPerson() Person {
   return Person{
      Name: "Anonymous",
   }
}

func main() {
   p := NewPerson()
   fmt.Println(p)
} 
```

Go 中没有析构函数，因为一切都是由垃圾回收来处理，你不需要手动销毁对象。通过延迟（defer）一个函数调用来在当前函数结束时执行一些清理操作是最接近的方法。

# 方法

方法是属于特定类型的函数，使用点标记法来调用，例如：

```go

myObject.myMethod()

```

点符号标记在 C++和其他面向对象的语言中被广泛使用。 点符号标记和类系统源自于在 C 中使用的一个常见模式。 这个常见模式是定义一组函数，所有这些函数都操作一个特定的数据类型。 所有相关的函数都有相同的第一个参数，即要操作的数据。 由于这是一个如此常见的模式，Go 将其内置到语言中。 在 Go 函数定义中，不是将要操作的对象作为第一个参数传递，而是有一个特殊的位置来指定接收器。 接收器在函数名称之前的一对括号之间指定。 下一个示例演示了如何使用函数接收器。

与其编写一组大型函数，所有这些函数都将指针作为它们的第一个参数，不如编写具有特殊*接收器*的函数。 接收器可以是类型或类型的指针：

```go
package main

import "fmt"

type Person struct {
   Name string
}

// Person function receiver
func (p Person) PrintInfo() {
   fmt.Printf("Name: %s\n", p.Name)
}

// Person pointer receiver
// If you did not use the pointer receivers
// it would not modify the person object
// Try removing the asterisk here and seeing how the
// program changes behavior
func (p *Person) ChangeName(newName string) {
   p.Name = newName
}

func main() {
   nanodano := Person{Name: "NanoDano"}
   nanodano.PrintInfo()
   nanodano.ChangeName("Just Dano")
   nanodano.PrintInfo()
} 
```

在 Go 中，您不会将所有变量和方法封装在一个整体的大括号对中。 您定义一个类型，然后定义操作该类型的方法。 这使您可以在一个地方定义所有的结构体和数据类型，并在包的其他地方定义方法。 您还可以选择在一起定义类型和方法。 这非常简单直接，创建了状态（数据）和逻辑之间稍微清晰的区别。

# 运算符重载

Go 中没有运算符重载，因此您不能使用`+`号将两个结构体相加，但是您可以轻松地在类型上定义一个`Add()`函数，然后调用类似`dataSet1.Add(dataSet2)`的函数。 通过将语言中的操作符重载省略掉，我们可以放心地使用这些操作符，而不必担心由于在代码中的其他地方重载操作符行为而导致的意外行为。

# Goroutines

Goroutines 是内置到语言中的轻量级线程。 您只需在函数调用前加上`go`这个词，就可以让函数在一个线程中执行。 本书中还可以将 goroutines 称为线程。

Go 确实提供了互斥锁，但在大多数情况下可以避免使用，并且本书不会涵盖它们。 您可以在[`golang.org/pkg/sync/`](https://golang.org/pkg/sync/)上阅读有关互斥锁的更多信息。 通道应该用于在线程之间共享数据和通信。 本章前面已经介绍了通道。

注意，`log`包是可以并发安全使用的，但`fmt`包不是。 下面是使用 goroutines 的简短示例：

```go

package main

import (
   "log"
   "time"
)

func countDown() {
   for i := 5; i >= 0; i-- {
      log.Println(i)
      time.Sleep(time.Millisecond * 500)
   }
}

func main() {
   // Kick off a thread
   go countDown()

   // Since functions are first-class
   // you can write an anonymous function
   // for a goroutine
   go func() {
      time.Sleep(time.Second * 2)
      log.Println("Delayed greetings!")
   }()

   // Use channels to signal when complete
   // Or in this case just wait
   time.Sleep(time.Second * 4)
} 
```

# 获取帮助和文档

Go 同时具有在线和离线帮助文档。 离线文档是 Go 内置的，与在线托管的文档相同。 接下来的几节将引导您访问这两种形式的文档。

# 在线 Go 文档

在线文档可在[`golang.org/`](https://golang.org/) 上找到，其中包含所有正式文档、规范和帮助文件。语言文档专门位于[`golang.org/doc/`](https://golang.org/doc/)，标准库信息位于[`golang.org/pkg/`](https://golang.org/pkg/)。

# 离线 Go 文档

Go 还附带了离线文档，使用`godoc`命令行工具即可。您可以在命令行上使用它，或者让它运行一个 Web 服务器，在其中提供与[`golang.org/`](https://golang.org/) 相同的网站。将完整的网站文档本地可用是非常方便的。以下是几个示例，用于获取`fmt`包的文档。将`fmt`替换为您感兴趣的任何包：


```go

# 获取 fmt 包信息
godoc fmt
# 获取 fmt 包的源代码
godoc -src fmt
# 获取特定函数信息
godoc fmt Printf
# 获取函数的源代码
godoc -src fmt Printf
# 运行 HTTP 服务器以查看 HTML 文档
godoc -http = localhost：9999

```

HTTP 选项提供与[`golang.org/`](https://golang.org/)上可用的相同文档。

# 摘要

阅读完本章后，您应该对 Go 基础有基本的了解，例如关键字是什么，它们的作用是什么，以及有哪些基本数据类型可用。您还应该可以轻松创建函数和自定义数据类型。

目标不是记住所有先前的信息，而是了解语言中提供了哪些工具。如有必要，使用本章作为参考。您可以在[`golang.org/ref/spec`](https://golang.org/ref/spec)找到有关 Go 语言规范的更多信息。

在下一章中，我们将讨论在 Go 中处理文件的工作。我们将涵盖基础知识，如获取文件信息，查看文件是否存在，截断文件，检查权限以及创建新文件。我们还将涵盖读取器和写入器接口，以及多种读取和写入数据的方法。除此之外，我们还将涵盖诸如打包到 ZIP 或 TAR 文件以及使用 GZIP 压缩文件等内容。
