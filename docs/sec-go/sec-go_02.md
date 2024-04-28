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

```
// Line comment, everything after slashes ignored
/* General comment, can be in middle of line or span multiple lines */
```

# 类型

内置数据类型的命名相当直观。Go 带有一组具有不同位长度的整数和无符号整数类型。还有浮点数、布尔值和字符串，这应该不足为奇。

有一些类型，如符文，在其他语言中不常见。本节涵盖了所有不同的类型。

# 布尔

布尔类型表示真或假值。有些语言不提供`bool`类型，您必须使用整数或定义自己的枚举，但 Go 方便地预先声明了`bool`类型。`true`和`false`常量也是预定义的，并且以全小写形式使用。以下是创建布尔值的示例：

```
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

```
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

```

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

# Array

Arrays are made up of sequenced elements of a specific type. An array can be created for any data type. The length of an array cannot be changed and must be specified at the time of declaration. Arrays are seldom used directly, but are used mostly through the slice type covered in the next section. Arrays are always one-dimensional, but you can create an array of arrays to create multidimensional objects.

To create an array of 128 bytes, this syntax can be used:

```

var myByteArray [128]byte

```

Individual elements of an array can be accessed by its 0-based numeric index. For example, to get the fifth element from the byte array, the syntax is as follows:

```

singleByte := myByteArray[4]

```

# Slice

Slices use arrays as the underlying data type. The main advantage is that slices can be resized, unlike arrays. Think of slices as a viewing window in to an underlying array. The **capacity** refers to the size of the underlying array, and the maximum possible length of a slice. The **length** of a slice refers to its current length which can be resized.

Slices are created using the `make()` function. The `make()` function will create a slice of a certain type with a certain length and capacity. The `make()` function can be used two ways when creating a slice. With only two parameters, the length and capacity are the same. With three parameters, you can specify a maximum capacity larger than the length. Here are two of the `make()` function declarations:

```

make([]T, lengthAndCapacity)
make([]T, length, capacity)

```

A nil slice can be created with a capacity and length of 0\. There is no underlying array associated with a nil slice. Here is a short example program demonstrating how to create and inspect a slice:

```

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

You can also append to a slice using the built-in `append()` function.

Append can add one or more elements at a time. The underlying array will be resized if necessary. This means that the maximum capacity of a slice can be increased. When a slice increases its underlying capacity, creating a larger underlying array, it will create the array with some extra space. This means that if you surpass a slice's capacity by one, it might increase the array size by four. This is done so that the underlying array has room to grow to reduce the number of times the underlying array has to be resized, which may require moving memory around to accommodate the larger array. It could be expensive to resize an array every time just to add a single element. The slice mechanics will automatically determine the best size for resizing.

This code sample provides various examples of working with slices:

```

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

# Struct

In Go, a struct or data structure is a collection of variables. The variables can be of different types. We will look at an example of creating a custom struct type.

Go uses case-based scoping to declare a variable either `public` or `private`. Variables and methods that are capitalized are exported and accessible from other packages. Lowercase values are private and only accessible within the same package.

The following example creates a simple struct named `Person` and one named `Hacker`. The `Hacker` type has a `Person` type embedded within it. An instance of each type is then created and the information about them is printed to standard output:

```

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

You can create *private* variables by starting their name with a lowercase letter. I use quotation marks because private variables work slightly different than in other languages. The privacy works at the package level and not at the *class* or type level.

# Pointer

Go provides a pointer type that stores the memory location where data of a specific type is stored. Pointers can be used to pass a struct to a function by reference without creating a copy. This also allows a function to modify an object in-place.

There is no pointer arithmetic allowed in Go. Pointers are considered *safe* because Go does not even define the addition operator on the pointer type. They can only be used to reference an existing object.

This example demonstrates basic pointer usage. It first creates an integer, and then creates a pointer to the integer. It then prints out the data type of the pointer, the address stored in the pointer, and then the value of data being pointed at:

```

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

# Function

Functions are defined with the `func` keyword. Functions can have multiple parameters. All parameters are positional and there are no named parameters. Go supports variadic parameters allowing for an unknown number of parameters. Functions are first-class citizens in Go, and can be used anonymously and returned as a variable. Go also supports multiple return values from a function. The underscore can be used to ignore a return variable.

All of these examples are demonstrated in the following code source:

```

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

# Interface

Interfaces are a special type that define a collection of function signatures. You can think of an interface as saying, "a type must implement function X and function Y to satisfy this interface." If you create any type and implement the functions needed to satisfy the interface, your type can be used anywhere that the interface is expected. You don't have to specify that you are trying to satisfy an interface, the compiler will determine if it satisfies the requirements.

You can add as many other functions as you want to your custom type. The interface defines the functions that are required, but it does not mean that your type is limited to implementing only those functions.

The most commonly used interface is the `error` interface. The `error` interface only requires a single function to be implemented, a function named `Error()` that returns a string with the error message. Here is the interface definition:

```
type error interface {
   Error() string
} 

```

This makes it very easy for you to implement your own error interfaces. This example creates a `customError` type and then implements the `Error()` function needed to satisfy the interface. Then, a sample function is created, which returns the custom error:

```

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

Other frequently used interfaces are the `Reader` and `Writer` interfaces. Each one only requires one function to be implemented in order to satisfy the interface requirements. The big benefit here is that you can create your own custom types that reads and writes data in some arbitrary way. The implementation details are not important to the interface. The interface won't care whether you are reading and writing to a hard disk, a network connection, storage in memory, or `/dev/null`. As long as you implement the function signatures that are required, you can use your type anywhere the interface is used. Here is the definition of the `Reader` and `Writer` interfaces:

```

type Reader interface {
   Read(p []byte) (n int, err error)
} 
 
type Writer interface {
   Write(p []byte) (n int, err error)
} 

```

# Map

A map is a hash table or dictionary that stores key and value pairs. The key and value can be any data types, including maps themselves, creating multiple dimensions.

The order is not guaranteed. You can iterate over a map multiple times and it might be different. Additionally, maps are not concurrent safe. If you must share a map between threads, use a mutex.

Here are some example map usages:

```

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

Channels are used to communicate between threads. Channels are **first-in, first-out** (**FIFO**) queues. You can push objects on to the queue and pull from the front asynchronously. Each channel can only support one data type. Channels are blocking by default, but can be made nonblocking with a `select` statement. Like slices and maps, channels must be initialized before use with the `make()` function.

The saying in Go is *Do not communicate by sharing memory; instead, share memory by communicating*. Read more about this philosophy at [`blog.golang.org/share-memory-by-communicating`](https://blog.golang.org/share-memory-by-communicating).

Here is an example program that demonstrates basic channel usage:

```

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

# Control structures

Control structures are used to control the flow of program execution. The most common forms are the `if` statements, `for` loops, and `switch` statements. Go also supports the `goto` statement, but should be reserved for cases of extreme performance and not used regularly. Let's look briefly at each of these to understand the syntax.

# if

The `if` statement comes with the `if`, `else if`, and `else` clauses, just like most other languages. The one interesting feature that Go has is the ability to put a statement before the condition, creating temporary variables that are discarded after the `if` statement has completed.

This example demonstrates the various ways to use an `if` statement:

```

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

The `for` loop has three components, and can be used just like a `for` loop in C or Java. Go has no `while` loop because the `for` loop serves the same purpose when used with a single condition. Refer to the following example for more clarity:

```

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

The `range` keyword is used to iterate over a slice, map, or other data structure. The `range` keyword is used in combination with the `for` loop, to operate on an iterable data structure. The `range` keyword returns the key and value variables. Here are some basic examples of using the `range` keyword:

```

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

# switch, case, fallthrough, and default

The `switch` statement allows you to branch execution based on the state of a variable. It is similar to the `switch` statement in C and other languages.

There is no `fallthrough` by default. This means once the end of a case is reached, the code exits the `switch` statement completely unless an explicit `fallthrough` command is provided. A `default` case can be provided if none of the cases are matched.

You can put a statement in front of the variable to be switched, such as the `if` statement. This creates a variable whose scope is limited to the `switch` statement.

This example demonstrates two `switch` statements. The first one uses hardcoded values and includes a `default` case. The second `switch` statement uses an alternate syntax that allows for a statement in the first line:

```

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

# goto

Go does have a `goto` statement, but it is very rarely used. Create a label with a name and a colon, then *go to* it using the `goto` keyword. Here is a basic example:

```

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

# Defer

By deferring a function, it will run whenever the current function is exited. This is a convenient way to ensure that a function will get executed before exiting, which is useful for cleaning up or closing files. It is convenient because a deferred function will get executed no matter where the surrounding function exits if there are multiple return locations.

Common use cases are deferring calls to close a file or database connection. Right after opening a file, you can defer a call to close. This will ensure that a file is closed whenever the function is exited, even if there are multiple return statements and you can't be sure about when and where the current function will exit.

This example demonstrates a simple use case for the `defer` keyword. It creates a file and then defers a call to `file.Close()`:

```

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

Be sure to properly check and handle errors. The `defer` call will panic if using a nil pointer.

It is also important to understand that deferred functions are run when the surrounding function is exited. If you put a `defer` call inside a `for` loop, it will not get called at the end of each `for` loop iteration.

# Packages

Packages are just directories. Every directory is its own package. Creating subdirectories creates a new package. Having no subpackages leads to a flat hierarchy. Subdirectories are used just for organizing code.

Packages should be stored in the `src` folder of your `$GOPATH` variable.

A package name should match the folder name or be named `main`. A `main` package means that it is not intended to be imported into another application, but meant to compile and run as a program. Packages are imported using the `import` keyword.

You can import packages individually:

```

import "fmt"

```

Alternatively, you can import multiple packages at once by wrapping them with parenthesis:

```

import (
   "fmt"
   "log"
) 
```

# Classes

Go technically does not have classes, but there are only a few subtle distinctions that keep it from being called an object-oriented language. Conceptually, I do consider it an object-oriented programming language, though it only supports the most basic features of an object-oriented language. It does not come with all of the features many people have come to associate with object-oriented programming, such as inheritance and polymorphism, which are replaced with other features such as embedded types and interfaces. Perhaps you could call it a *microclass* system, because it is a minimalistic implementation with none of the extra features or baggage, depending on your perspective.

Throughout this book, the terms *object* and *class* may be used to illustrate a point using familiar terms, but be aware that these are not formal terms in Go. A type definition in combination with the functions that operate on that type are like the class, and the object is an instance of a type.

# Inheritance

There is no inheritance in Go, but you can embed types. Here is an example of a `Person` and `Doctor` types, which embeds the `Person` type. Instead of inheriting the behavior of `Person` directly, it stores the `Person` object as a variable, which brings with it all of its expected `Person` methods and attributes:

```

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

# Polymorphism

There is no polymorphism in Go, but you can use interfaces to create common abstraction that can be used by multiple types. Interfaces define one or more method declarations that must be satisfied to be compatible with the interface. Interfaces were covered earlier in this chapter.

# Constructors

There are no constructors in Go, but there are `New()` functions that act like factories initializing an object. You simply have to create a function named `New()` that returns your data type. Here is an example:

```

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

There are no deconstructors in Go, since everything is garbage collected and you do not manually destroy objects. Defer is the closest you can get by deferring a function call to perform some cleanup when the current function ends.

# Methods

Methods are functions that belong to a specific type, and are called using the dot notation, for example:

```

myObject.myMethod()

```

The dot notation is widely used in C++ and other object-oriented languages. The dot notation and the class system stemmed from a common pattern that was used in C. The common pattern is to define a set of functions that all operate on a specific data type. All of the related functions have the same first parameter, which is the data to be operated on. Since this is such a common pattern, Go built it into the language. Instead of passing the object to be manipulated as the first argument, there is a special place to designate the receiver in a Go function definition. The receiver is specified between a set of parenthesis before the function name. The next example demonstrates how to use function receivers.

Instead of writing a large set of functions that all took a pointer as their first parameter, you can write functions that have a special *receiver*. The receiver can either be a type or a pointer to a type:

```
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

In Go, you do not encapsulate all of the variables and methods inside a monolithic pair of braces. You define a type, and then define methods that operate on that type. This allows you to define all of your structs and data types in one place, and define the methods elsewhere in your package. You also have the option of defining a type and the methods right next to each other. It's pretty simple and straightforward, and it creates a slightly clearer distinction between the state (data) and the logic.

# Operator overloading

There is no operator overloading in Go, so you can't add to structs together with the `+` sign, but you can easily define an `Add()` function on the type and then call something like `dataSet1.Add(dataSet2)`. By omitting operator overloading from the language, we can confidently use the operators without worrying about unexpected behavior due to operator behavior being overloaded somewhere else in code without realizing it.

# Goroutines

Goroutines are lightweight threads built into the language. You simply have to put the word `go` in front of a function call to have the function execute in a thread. Goroutines may also be referred to as threads in this book.

Go does provide mutexes, but they are avoidable in most cases and will not be covered in this book. You can read more about mutexes in the `sync` package documentation at [`golang.org/pkg/sync/`](https://golang.org/pkg/sync/). Channels should be used instead for sharing data and communicating between threads. Channels were covered earlier in this chapter.

Note that the `log` package is safe to use concurrently, but the `fmt` package is not. Here is a short example of using goroutines:

```

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

# Getting help and documentation

Go has both online and offline help documentation. The offline documentation is built-in for Go and is the same documentation that is hosted online. These next sections will walk you through accessing both forms of documentation.

# Online Go documentation

The online documentation is available at [`golang.org/`](https://golang.org/), and has all the formal documentation, specifications, and help files. Language documentation specifically is at [`golang.org/doc/`](https://golang.org/doc/), and information about the standard library is at [`golang.org/pkg/`](https://golang.org/pkg/).

# Offline Go documentation

Go also comes with offline documentation with the `godoc` command-line tool. You can use it on the command line, or have it run a web server where it serves the same website that [`golang.org/`](https://golang.org/) hosts. It is quite handy to have the full website documentation available locally. Here are a few examples that get documentation for the `fmt` package. Replace `fmt` with whatever package you are interested in:

```

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
