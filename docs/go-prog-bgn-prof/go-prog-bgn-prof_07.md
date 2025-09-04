

# 第七章：接口

概述

本章旨在展示 Go 中接口的实现。与其他语言相比，它相当简单，因为在 Go 中它是隐式实现的，而其他语言则需要显式实现接口。

在开始时，您将能够为应用程序定义和声明一个接口，并在您的应用程序中实现接口。本章将向您介绍使用鸭子类型和多态，接受接口，并返回结构体。

到本章结束时，您将学会如何使用类型断言来访问接口的底层具体值，并使用类型选择语句。

# 技术要求

对于本章，您需要 Go 版本 1.21 或更高版本。本章的代码可以在以下位置找到：[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter07`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter07)。

# 简介

在上一章中，我们讨论了 Go 中的错误处理。我们探讨了 Go 中的错误是什么；它是指实现了错误接口的任何东西。当时，我们没有调查接口是什么。在这一章中，我们将探讨接口是什么。

例如，您的经理要求您创建一个可以接受 JSON 数据的 API。数据包含有关各种员工的信息，例如他们的地址和他们在项目上的工作时间。数据需要解析到`employee`结构体中，这是一个相对简单的任务。然后您创建了一个名为`loadEmployee(s string)`的函数。该函数将接受一个格式为 JSON 的字符串，然后将该字符串解析为加载`employee`结构体。

您的经理对工作很满意；然而，他们还有另一个要求。客户端需要能够接受一个包含员工数据的 JSON 格式文件。要执行的功能与之前相同。您创建了一个名为`loadEmployeeFromFile(f *os.File)`的函数，该函数从文件中读取数据，解析数据，并加载`employee`结构体。

您的经理又有新的要求，即员工数据现在也应来自一个 HTTP 端点。您需要能够从 HTTP 请求中读取数据，因此您创建了一个名为`loadEmployeeFromHTTP(r *Request)`的函数。

编写的三个函数都具有共同的行为，即它们正在执行。它们都需要能够读取数据。底层类型可能不同（例如`string`、`os.File`或`http.Request`），但在所有情况下，行为或读取数据都是相同的。

`func loadEmployee(s string)`, `func loadEmployeeFromFile(f *os.File)`, 和 `func loadEmployeeFromHTTP(r *Request)` 这些函数都可以用接口 `func loadEmployee (r io.Reader)` 来替换。`io.Reader` 是一个接口，我们将在本章后面更深入地讨论它，但就目前而言，只需知道它可以用来解决给定的问题。

在本章中，我们将看到接口如何解决这样的问题；通过将正在执行的行为定义为接口类型，我们可以接受任何底层具体类型。如果现在这还不清楚，随着我们继续本章的学习，它将开始变得清晰。我们将讨论接口如何让我们执行鸭子类型和多态。我们将看到接受接口和返回结构体会减少耦合并增加函数在我们程序更多区域的使用。我们还将检查空接口，并讨论使用案例以充分利用它，包括类型断言和类型选择语句。

# 接口

接口是一组描述数据类型行为的函数。接口定义了实现该接口的类型必须满足的行为。行为描述了该类型可以做什么。几乎一切事物都表现出某些行为。例如，猫可以喵喵叫、行走、跳跃和咕噜咕噜。所有这些都是猫的行为。汽车可以启动、停止、转弯和加速。所有这些都是汽车的行为。同样，类型的行为被称为方法。

注意

官方文档提供的定义是：*Go 中的接口提供了一种指定对象行为的方式。（[`go.dev/doc/effective_go#interfaces_and_types`](https://go.dev/doc/effective_go#interfaces_and_types)*）

描述接口有几种方式：

+   方法签名集合是只有方法名称、参数、类型和返回类型的方法。这是 `Speaker{}` 接口方法签名集合的一个例子：

    ```go
    type Speaker interface {
        Speak(message string) string
        Greet() string
    }
    ```

+   为了满足接口，需要类型的方法蓝图。使用 `Speaker{}` 接口，蓝图（接口）指出，为了满足 `Speaker{}` 接口，类型必须有一个接受字符串并返回字符串的 `Speak()` 方法。它还必须有一个返回字符串的 `Greet()` 方法。

+   行为是接口类型必须表现出的。例如，`Reader{}` 接口有一个 `Read` 方法。它的行为是读取数据。以下代码来自 Go 标准库的 `Reader{}` 接口：

    ```go
    type Reader interface {
        Read(b []byte)(n int, err error)
    }
    ```

+   接口可以描述为没有实现细节。当定义一个接口，如 `Reader{}` 接口时，它只包含方法签名而没有实际的代码实现。提供代码或实现细节的责任在于实现接口的类型，而不是接口本身。

一个类型的行为统称为方法集，它是与该类型相关联的方法集合。方法集包括接口定义的方法名称，以及任何输入参数和返回类型。例如，一个类型可能表现出 `Read()`、`Write()` 和 `Save()` 等行为。这些行为共同构成了类型的方法集，为类型可以执行的动作或功能提供了清晰的定义。

重要的是要注意，选择这些行为和类型特性的原因应该被清楚地记录。了解一个类型具有特定行为的原因可以为设计决策提供上下文，并增强整体代码理解。

![图 7.1：接口元素的图形表示](img/B18621_07_01.jpg)

图 7.1：接口元素的图形表示

当谈论行为时，请注意，我们没有讨论实现细节。定义接口时省略了实现细节。重要的是要理解，在接口的声明中未指定或强制实施任何实现。我们创建的每个实现接口的类型都可以有自己的实现细节。具有名为 `Greeting()` 方法的接口可以由各种类型以不同的方式实现。一个 `person` 结构体类型可以以不同于 `animal` 结构体类型的方式实现 `Greeting()`。

接口关注类型必须表现的行为。接口的职责不是提供方法实现。那是实现接口的类型的工作。类型，通常是结构体，包含方法集的实现细节。现在我们已经对接口有了基本的了解，在下一个主题中，我们将探讨如何定义接口。

## 定义一个接口

定义一个接口涉及以下步骤：

![图 7.2：定义一个接口](img/B18621_07_02.jpg)

图 7.2：定义一个接口

下面是一个声明接口的例子：

```go
type Speaker interface {
    Speak() string
}
```

让我们看看这个声明的每个部分：

+   从 `type` 关键字开始，然后是名称，然后是 `interface` 关键字。

+   我们正在定义一个名为 `Speaker{}` 的接口类型。在 Go 中，使用 `er` 后缀命名接口是惯用的。如果是一个单方法接口，通常将接口名称命名为那个单一方法。

+   接下来，你定义方法集。定义一个接口类型指定属于它的方法。在这个接口中，我们声明了一个具有一个名为 `Speak()` 的方法的接口类型，它返回一个字符串。

+   `Speaker{}` 接口的方法集是 `Speak()`.

下面是一个在 Go 中经常使用的接口：

```go
// https://golang.org/pkg/io/#Reader
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

让我们看看这段代码的各个部分：

+   接口名称是 `Reader{}`

+   方法集是 `Read()`

+   `Read()` 方法的签名是 `(p []byte)(n int, err error)`

接口可以有多个方法作为其方法集。让我们看看 Go 包中使用的接口：

```go
// https://golang.org/pkg/os/#FileInfo
type FileInfo interface {
    Name() string // base name of the file
    Size() int64 // length in bytes for regular files; system-dependent for others
    Mode() FileMode // file mode bits
    ModTime() time.Time // modification time
    IsDir() bool // abbreviation for Mode().IsDir()
    Sys() interface{} // underlying data source (can return nil)
}
```

如您所见，`FileInfo{}` 有多个方法。

总结来说，接口是声明方法集的类型。与其他使用接口的语言类似，它们不实现方法集。实现细节不是定义接口的一部分。在下一个主题中，我们将探讨 Go 要求你能够实现接口的条件。

## 实现接口

其他编程语言中的接口是显式实现接口的。显式实现意味着编程语言直接且明确地声明该对象正在使用此接口。例如，这是在 Java 中：

```go
class Dog implements Pet
```

代码段明确指出 `Dog` 类将实现 `Pet` 接口。

在 Go 中，接口是隐式实现的。这意味着一个类型将通过拥有接口的所有方法和签名来实现接口。以下是一个示例：

```go
package main
import (
  "fmt"
)
type Speaker interface {
  Speak() string
}
type cat struct {
}
func main() {
  c := cat{}
  fmt.Println(c.Speak())
  c.Greeting()
}
func (c cat) Speak() string {
  return "Purr Meow"
}
func (c cat) Greeting() {
  fmt.Println("Meow,Meow!!!!mmmeeeeoooowwww")
}
```

让我们将这段代码分解成几个部分：

```go
type Speaker interface {
  Speak() string
}
```

我们正在定义一个 `Speaker{}` 接口。它有一个描述 `Speak()` 行为的方法。该方法返回一个字符串。为了实现 `Speaker{}` 接口，类型必须拥有接口声明中列出的方法。然后，我们创建一个名为 `cat` 的空结构体类型：

```go
type cat struct {
}
func (c cat) Speak() string {
  return "Purr Meow"
}
```

`cat` 类型有一个返回字符串的 `Speak()` 方法。这满足了 `Speaker{}` 接口。现在，`cat` 的实现者有责任提供 `cat` 类型 `Speak()` 方法的实现细节。

注意，没有显式声明 `cat` 实现了 `Speaker{}` 接口；它只是通过满足接口的要求来实现。

还要注意的是，`cat` 类型有一个名为 `Greeting()` 的方法。类型可以拥有不满足 `Speaker{}` 接口需求的方法。然而，`cat` 至少必须拥有满足接口所需的方法集。

输出将如下所示：

```go
Purr Meow
Meow,Meow!!!!mmmeeeeoooowwww
```

## 隐式实现接口的优势

隐式实现接口有一些优势。我们看到了当你创建一个接口时，你必须去每个类型并明确声明该类型实现了接口。在 Go 中，满足接口的类型被认为是实现了接口。没有像其他语言那样的 `implements` 关键字；你不需要声明一个类型实现了接口。在 Go 中，如果它有接口的方法集和签名，它就隐式地实现了接口。

当你更改接口的方法集时，在其他语言中，你必须去所有那些不满足接口的类型，并移除类型的显式声明。在 Go 中并非如此，因为它是隐式声明。

另一个优点是你可以使用接口来处理另一个包中的类型。这解耦了接口定义与其实现。我们将在 *第十章*，*包保持项目可管理* 中讨论包及其作用域。

让我们看看在主包中如何使用来自不同包的接口的例子。`Stringer` 接口是 Go 语言中的一个接口，它通过 Go 语言被几个包使用。一个例子是 `fmt` 包，它在打印值时用于格式化：

```go
type Stringer interface {
  String() string
}
```

`Stringer` 是一个可以描述自身为字符串的类型接口。接口名称通常遵循方法名称，但添加了 `er` 后缀：

```go
package main
import (
  "fmt"
)
type Speaker interface {
  Speak() string
}
type cat struct {
  name string
  age int
}
func main() {
  c := cat{name: "Oreo", age:9}
  fmt.Println(c.Speak())
  fmt.Println(c)
}
func (c cat) Speak() string {
  return "Purr Meow"
}
func (c cat) String() string {
  return fmt.Sprintf("%v (%v years old)", c.name, c.age)
}
```

让我们将这段代码分解成几个部分：

+   我们为我们的 `cat` 类型添加了一个 `String()` 方法。它返回 `name` 和 `age` 字段的数据。

+   当我们在 `main()` 中调用 `fmt.Println()` 方法，并传入 `cat` 作为参数时，`fmt.Println()` 会调用 `cat` 类型的 `String()` 方法。

+   我们的 `cat` 类型现在实现了两个接口：`Speaker{}` 接口和 `Stringer{}` 接口。它具有满足这两个接口所需的方法：

![图 7.3：类型可以实现多个接口](img/B18621_07_03.jpg)

图 7.3：类型可以实现多个接口

## 练习 7.01 – 实现接口

在这个练习中，我们将创建一个简单的程序，演示如何隐式实现接口。我们将有一个 `person` 结构体，它将隐式实现 `Speaker{}` 接口。`person` 结构体将包含 `name`、`age` 和 `isMarried` 作为其字段。程序将调用 `person` 结构体的 `Speak()` 方法，并显示一个显示 `person` 结构体 `name` 的消息。`person` 结构体还将通过拥有一个 `String()` 方法来满足 `Stringer{}` 接口的要求。你可能还记得，在之前的 *隐式实现接口的优势* 部分，`Stringer{}` 接口是 Go 语言中的一个接口。它可以在打印值时用于格式化。这就是我们在这个练习中将如何使用它来格式化 `person` 结构体字段打印的方式：

1.  创建一个新的文件，并将其保存为 `main.go`。

1.  我们将创建一个 `package` `main` 并在这个程序中使用 `fmt` 包：

    ```go
    package main
    import (
      "fmt"
    )
    ```

1.  创建一个名为 `Speaker{}` 的接口，其中包含一个名为 `Speak()` 的方法，该方法返回一个字符串：

    ```go
    type Speaker interface {
      Speak() string
    }
    ```

    我们已经创建了一个 `Speaker{}` 接口。任何想要实现我们的 `Speaker{}` 接口类型都必须有一个返回字符串的 `Speak()` 方法。

1.  创建我们的 `person` 结构体，包含 `name`、`age` 和 `isMarried` 作为其字段：

    ```go
    type person struct {
      name string
      age int
      isMarried bool
    }
    ```

    我们的 `person` 类型包含 `name`、`age` 和 `isMarried` 字段。我们将在 `main` 函数中使用返回字符串的 `Speak()` 方法来打印这些字段的内容。拥有一个 `Speak()` 方法将满足 `Speaker{}` 接口的要求。

1.  在`main()`函数中，我们将初始化一个`person`类型，打印`Speak()`方法，并打印`person`字段的值：

    ```go
    func main() {
      p := person{name: "Cailyn", age: 44, isMarried: false}
      fmt.Println(p.Speak())
      fmt.Println(p)
    }
    ```

1.  为`person`创建一个`String()`方法并返回一个字符串值。这将满足`Stringer{}`接口，现在它可以通过`fmt.Println()`方法被调用：

    ```go
    func (p person) String() string {
      return fmt.Sprintf("%v (%v years old).\nMarried status: %v ", p.name, p.age, p.isMarried)
    }
    ```

1.  为`person`创建一个返回字符串的`Speak()`方法。`person`类型有一个与`Speaker{}`接口的`Speak()`方法具有相同签名的`Speak()`方法。通过拥有返回字符串的`Speak()`方法，`person`类型满足`Speaker{}`接口。为了满足接口，你必须有与接口相同的方法和方法签名：

    ```go
    func (p person) Speak() string {
      return "Hi my name is: " + p.name
    }
    ```

1.  打开终端并导航到代码的目录。

1.  运行`go build main.go`。

1.  修正返回的错误，并确保你的代码与这里的代码片段匹配。

1.  通过在命令行中输入可执行文件名并使用`./main`命令来运行可执行文件。

你应该得到以下输出：

```go
Hi my name is Cailyn
Cailyn (44 years old).
Married status: false
```

在这个练习中，我们看到了如何隐式地实现接口是多么简单。在下一个主题中，我们将通过让不同的数据类型，如 structs，实现相同的接口，并将其传递给任何具有该接口类型参数的函数来在此基础上构建。我们将在下一个主题中更详细地介绍这是如何可能的，并看看为什么类型以各种形式出现是一个好处。

# 鸭子类型

我们基本上一直在做被称为鸭子类型（duck typing）的事情。鸭子类型是计算机编程中的一个测试：*如果它看起来像鸭子，游泳像鸭子，发出鸭子的叫声，那么它一定是一只鸭子。* 如果一个类型匹配一个接口，那么你可以在使用该接口的任何地方使用那个类型。鸭子类型是根据方法匹配类型，而不是预期的类型：

```go
type Speaker interface {
  Speak() string
}
```

任何匹配`Speak()`方法的都可以是`Speaker{}`接口。当我们实现一个接口时，我们实际上是通过拥有所需的方法集来符合该接口的：

```go
package main
import (
  "fmt"
)
type Speaker interface {
  Speak() string
}
type cat struct {
}
func main() {
  c := cat{}
  fmt.Println(c.Speak())
}
func (c cat) Speak() string {
  return "Purr Meow"
}
```

`cat`与`Speaker{}`接口的`Speak()`方法匹配，所以`cat`是`Speaker{}`：

```go
package main
import (
  "fmt"
)
type Speaker interface {
  Speak() string
}
type cat struct {
}
func main() {
  c := cat{}
  chatter(c)
}
func (c cat) Speak() string {
  return "Purr Meow"
}
func chatter(s Speaker) {
  fmt.Println(s.Speak())
}
```

让我们分部分检查这段代码：

+   在前面的代码中，我们声明了一个`cat`类型并为`cat`类型创建了一个名为`Speak()`的方法。这满足了`Speaker{}`接口所需的方法集。

+   我们创建了一个名为`chatter`的方法，它接受`Speaker{}`接口作为参数。

+   在`main()`函数中，我们能够将`cat`类型传递给`chatter`函数，它可以评估为`Speaker{}`接口。这满足了接口所需的方法集。

# 多态

多态是能够以各种形式出现的能力。例如，一个形状可以表现为正方形、圆形、矩形或任何其他形状：

![图 7.4：形状的多态示例](img/B18621_07_04.jpg)

图 7.4：形状的多态示例

Go 不像其他面向对象的语言那样进行子类化，因为 Go 没有类。面向对象编程中的子类化是从一个类继承到另一个类。通过子类化，你继承了另一个类的字段和方法。Go 通过嵌入 struct 和使用接口的多态提供了类似的行为。

使用多态的一个优点是它允许重用已经编写并测试过的方法。通过将`int`、`float`和`bool`等具体类型传递，代码被重用。然而，如果你的 API 接受一个接口，那么调用者可以添加所需的方法集以满足该接口，无论底层类型如何。这种可重用性是通过允许你的 API 接受接口来实现的。任何满足接口的类型都可以传递给 API。我们在前面的例子中已经看到了这种行为。现在是时候更仔细地看看`Speaker{}`接口了。

正如我们在前面的例子中所看到的，每个具体类型都可以实现一个或多个接口。回想一下，我们的`Speaker{}`接口可以被`dog`、`cat`或`person`类型实现：

![图 7.5：由多个类型实现的 Speaker 接口](img/B18621_07_05.jpg)

图 7.5：由多个类型实现的 Speaker 接口

当一个函数接受一个接口作为输入参数时，任何实现该接口的具体类型都可以作为参数传递。现在，通过能够将各种具体类型传递给具有接口类型输入参数的方法或函数，你已经实现了多态。

让我们看看一些渐进的例子，这将使我们能够展示如何在 Go 中实现多态：

```go
package main
import (
  "fmt"
)
type Speaker interface {
  Speak() string
}
type cat struct {
}
func main() {
  c := cat{}
  catSpeak(c)
}
func (c cat) Speak() string {
  return "Purr Meow"
}
func catSpeak(c cat) {
  fmt.Println(c.Speak())
}
```

让我们分部分检查这段代码：

+   `cat`满足`Speaker{}`接口。`main()`函数调用`catSpeak()`并传递一个`cat`类型的参数。

+   在`catSpeak()`内部，它打印出其`Speak()`方法的结果。

我们将实现一些代码，它接受一个具体类型（`cat`、`dog`或`person`）并满足`Speaker{}`接口类型。使用之前的编码模式，它将类似于以下代码片段：

```go
package main
import (
  "fmt"
)
type Speaker interface {
  Speak() string
}
type cat struct {
}
type dog struct {
}
type person struct {
  name string
}
func main() {
  c := cat{}
  d := dog{}
  p := person{name:"Heather"}
  catSpeak(c)
  dogSpeak(d)
  personSpeak(p)
}
func (c cat) Speak() string {
  return "Purr Meow"
}
func (d dog) Speak() string {
  return "Woof Woof"
}
func (p person) Speak() string {
  return "Hi my name is " + p.name +"."
}
func catSpeak(c cat) {
  fmt.Println(c.Speak())
}
func dogSpeak(d dog) {
  fmt.Println(d.Speak())
}
func personSpeak(p person) {
  fmt.Println(p.Speak())
}
```

让我们分部分看看这段代码：

```go
type cat struct {
}
type dog struct {
}
type person struct {
  name string
}
```

我们有三个具体类型（`cat`、`dog`和`person`）。`cat`和`dog`类型是空的 struct，而`person`struct 有一个`name`字段：

```go
func (c cat) Speak() string {
  return "Purr Meow"
}
func (d dog) Speak() string {
  return "Woof Woof"
}
func (p person) Speak() string {
  return "Hi my name is " + p.name +"."
}
```

我们的每个类型都隐式地实现了`Speaker{}`接口。每个具体类型都以与其他类型不同的方式实现它：

```go
func main() {
  c := cat{}
  d := dog{}
  p := person{name:"Heather"}
  catSpeak(c)
  dogSpeak(d)
  personSpeak(p)
}
```

在`main()`函数中，我们调用`catSpeak()`、`dogSpeak()`和`personSpeak()`来调用它们各自的`Speak()`方法。前面的代码有很多执行类似操作的重叠函数。我们可以重构这段代码，使其更简单、更容易阅读。我们将使用实现接口时获得的一些特性来提供更简洁的实现：

```go
package main
import (
  "fmt"
)
type Speaker interface {
  Speak() string
}
func saySomething(say ...Speaker) {
  for _, s := range say {
    fmt.Println(s.Speak())
  }
}
type cat struct {}
func (c cat) Speak() string {
  return "Purr Meow"
}
type dog struct {}
func (d dog) Speak() string {
  return "Woof Woof"
}
type person struct {
  name string
}
func (p person) Speak() string {
  return "Hi my name is " + p.name + "."
}
func main() {
  c := cat{}
  d := dog{}
  p := person{name: "Heather"}
  saySomething(c,d,p)
}
```

让我们分部分看看这段代码：

```go
func saySomething(say ...Speaker)
```

我们的`saySomething()`函数使用可变参数。如果你还记得，可变参数可以接受零个或多个该类型的参数。有关可变函数的更多信息，请参阅*第五章*，*减少、重用和回收*。参数类型是`Speaker`。接口可以用作输入参数：

```go
func saySomething(say ...Speaker) {
  for _, s := range say {
    fmt.Println(s.Speak())
  }
}
```

我们遍历`Speaker`的切片。对于每种`Speaker`类型，我们调用`Speak()`方法。在我们的代码中，我们将`cat`和`dog`结构体类型传递给`person`函数。该函数接受一个`Speaker{}`接口类型的参数。可以调用该接口的任何方法。对于这些具体类型，都会调用`Speak()`方法。

![图 7.6：实现 Speaker 接口的多种类型](img/B18621_07_06.jpg)

图 7.6：实现 Speaker 接口的多种类型

在`main()`函数中，我们将通过使用接口来演示多态：

```go
func main() {
  c := cat{}
  d := dog{}
  p := person{name: "Heather"}
  saySomething(c,d,p)
}
```

我们实现了每个具体类型，`cat`、`dog`和`person`。`cat`、`dog`和`person`类型都满足`Speaker{}`接口。由于它们匹配接口，因此可以在使用该接口的任何地方使用这些类型。正如你所见，这还包括能够将`cat`、`dog`和`person`类型传递给一个方法。

通过使用接口和多态，此代码比之前的代码片段更简洁。本章开头示例显示了一个满足`Speaker{}`接口并调用`Speak()`方法的单个具体类型。然后我们在运行示例中添加了几个更多具体类型（`cat`、`dog`和`person`），每个都分别调用它们自己的`Speak()`方法。我们在该示例中发现了冗余代码，并开始寻找更好的实现解决方案的方法。我们发现接口类型可以作为参数输入类型。通过鸭子类型和多态，我们的第三个和最后一个代码片段能够拥有一个函数，该函数会对满足`Speaker()`接口的每个类型调用`Speak()`方法。

练习 7.02 – 使用多态计算不同形状的面积

我们将实现一个程序，该程序将计算三角形、矩形和正方形的面积。该程序将使用一个接受`Shape`接口的单个函数。任何满足`Shape`接口的类型都可以作为函数的参数传递。该函数应打印出面积和形状的名称：

1.  使用你选择的 IDE。

1.  创建一个新文件，并将其保存为`main.go`。

1.  我们将有一个名为`main`的包，并且在这个程序中我们将使用`fmt`包：

    ```go
    package main
    import (
      "fmt"
    )
    ```

1.  创建`Shape{}`接口，它有两个方法集，`Area() float64`和`Name() string`：

    ```go
    type Shape interface {
      Area() float64
      Name() string
    }
    ```

1.  接下来，我们将创建`triangle`、`rectangle`和`square`结构体类型。这些类型将各自满足`Shape{}`接口。`triangle`、`rectangle`和`square`具有计算形状面积所需的适当字段：

    ```go
    type triangle struct {
      base float64
      height float64
    }
    type rectangle struct {
      length float64
      width float64
    }
    type square struct {
      side float64
    }
    ```

1.  我们为`triangle`结构体类型创建`Area()`和`Name()`方法。三角形的面积是`底 * 高/2`。`Name()`方法返回形状的名称：

    ```go
    func (t triangle) Area() float64 {
      return (t.base * t.height) / 2
    }
    func (t triangle) Name() string {
      return "triangle"
    }
    ```

1.  我们为`rectangle`结构体类型创建`Area()`和`Name()`方法。矩形的面积是`长度 * 宽度`。`Name()`方法返回形状的名称：

    ```go
    func (r rectangle) Area() float64 {
      return r.length * r.width
    }
    func (r rectangle) Name() string {
      return "rectangle"
    }
    ```

1.  我们为`square`结构体类型创建`Area()`和`Name()`方法。正方形的面积是`边长 * 边长`。`Name()`方法返回形状的名称：

    ```go
    func (s square) Area() float64 {
      return s.side * s.side
    }
    func (s square) Name() string {
      return "square"
    }
    ```

    现在，我们的每个形状（`triangle`、`rectangle`和`square`）都满足`Shape`接口，因为它们各自都有一个具有适当签名的`Area()`和`Name()`方法：

![图 7.7：Shape 类型的正方形、三角形和矩形面积](img/B18621_07_07.jpg)

图 7.7：Shape 类型的正方形、三角形和矩形面积

1.  我们现在将创建一个函数，该函数接受`Shape`接口作为可变参数。该函数将遍历`Shape`类型，并执行其每个`Name()`和`Area()`方法：

    ```go
    func printShapeDetails(shapes ...Shape) {
      for _, item := range shapes {
        fmt.Printf("The area of %s is: %.2f\n", item.Name(), item.Area())
      }
    }
    ```

1.  在`main()`函数内部，设置`triangle`、`rectangle`和`square`的字段。将这三个都传递给`printShapeDetail()`函数。因为它们各自都满足`Shape`接口，所以都可以传递：

    ```go
    func main() {
      t := triangle{base: 15.5, height: 20.1}
      r := rectangle{length: 20, width: 10}
      s := square{side: 10}
      printShapeDetails(t, r, s)
    }
    ```

1.  通过在命令行中运行`go build`来构建程序：

    ```go
    go build main.go
    ```

1.  修正返回的错误，并确保你的代码与这里的代码片段匹配。

1.  通过在命令行中输入可执行文件名并按*Enter*键来运行可执行文件：

    ```go
    ./main
    ```

你应该看到以下输出：

```go
The area of triangle is: 155.78
The area of rectangle is: 200.00
The area of square is: 100.00
```

在这个练习中，我们看到了接口为我们程序提供的灵活性和可重用代码。进一步地，我们将讨论通过接受接口和为我们的函数和方法返回结构体来增加代码重用性和降低耦合的方法。当我们使用接口作为 API 的输入参数时，我们是在声明一个类型需要满足该接口。当使用具体类型时，我们要求 API 的参数必须是该类型。例如，如果函数签名是`func greeting(msg string)`，我们知道传递的参数必须是一个字符串。具体类型可以被认为是非抽象类型（如`float64`、`int`、`string`等）；然而，接口可以被认为是抽象类型，因为你在满足接口类型的方法集。底层接口类型是一个具体类型，但底层类型不是需要传递到 API 中的类型。类型必须满足接口类型定义的方法集要求。

在未来，如果我们需要传递另一种类型，这意味着我们的 API 上游代码需要更改，或者如果我们的 API 的调用者需要更改其数据类型，它可能会要求我们更改我们的 API 以适应它。如果我们使用接口，这不是问题；我们的代码的调用者需要满足接口的方法集。然后，调用者可以更改底层类型，如果它符合接口要求的话。

接受接口并返回结构体

有一个 Go 谚语说，*接受接口，返回结构体*。这也可以表述为接受接口并返回具体类型。这个谚语是在谈论为你的 API（函数、方法等）接受接口，并返回结构体或具体类型。这个谚语遵循 Postel 法则，该法则指出，*对你所做的事情要保守，对你所接受的东西要宽容*。我们关注的是 *对你所接受的东西要宽容* 部分。通过接受接口，你增加了你的函数或方法的 API 的灵活性。通过这样做，你允许 API 的用户满足接口的要求，但不会强迫用户使用具体类型。如果我们的函数或方法只接受具体类型，那么我们就限制了函数的用户只能使用特定的实现。在本章中，我们将探讨前面提到的 Go 谚语，并了解为什么遵循它是一个好的设计模式。我们将看到，当我们查看代码示例时：

![图 7.8：接受接口的好处](img/B18621_07_08.jpg)

图 7.8：接受接口的好处

以下示例将说明接受接口与使用具体类型相比的好处。我们将有两个执行相同任务（解码 JSON）的函数，但它们的输入不同。其中一个函数比另一个函数更优越，我们将讨论为什么是这样。

看看下面的例子：

main.go

```go
package main
import (
    "encoding/json"
    "fmt"
    "io"
    "strings"
)
type Person struct {
    Name string `json:"name"`
    Age int `json:"age"`
}
```

完整的代码可在[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/main/Chapter07/Example01/main.go`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/main/Chapter07/Example01/main.go)找到。

预期输出如下：

```go
{Joe 18}
{Jane 21}
```

让我们检查这段代码的每一部分。我们将在接下来的章节中讨论代码的一些部分。这段代码将一些数据解码到一个结构体中。为此使用了两个函数，`loadPerson2()` 和 `loadPerson()`：

```go
func loadPerson2(s string) (Person, error) {
    var p Person
    err := json.NewDecoder(strings.NewReader(s)).Decode(&p)
    return p, err
}
```

`loadPerson2()` 函数接受一个具体的字符串参数并返回一个结构体。返回结构体的方式符合“接受接口，返回结构体”的一半原则。然而，它的接受范围非常有限，并不开放。这限制了函数的使用范围，只能用于狭窄的实现。唯一可以传递的是字符串。当然，在某些情况下这可能可以接受，但在其他情况下可能会出现问题。例如，如果你的函数或方法应该只接受特定的数据类型，那么你可能不想接受接口：

```go
func loadPerson(r io.Reader) (Person, error) {
    var p Person
    err := json.NewDecoder(r).Decode(&p)    return p, err
}
```

在这个函数中，我们接受 `io.Reader{}` 接口。`io.Reader{}` ([`pkg.go.dev/io#Reader`](https://pkg.go.dev/io#Reader)) 和 `io.Writer{}` ([`pkg.go.dev/io#Writer`](https://pkg.go.dev/io#Writer)) 接口是 Go 包中最常用的接口之一。`json.NewDecoder` 接受任何满足 `io.Reader{}` 接口的对象。调用者只需确保他们传递的对象满足 `io.Reader{}` 接口：

```go
p, err := loadPerson(strings.NewReader(s))
```

`strings.NewReader` 返回一个具有 `Read(b []byte) (n int, err error)` 方法的 `Reader` 类型，该方法满足 `io.Reader{}` 接口。它可以传递给我们的 `loadPerson()` 函数。你可能认为每个函数仍然在执行其预期功能。你会是对的，但假设调用者不再传递字符串，或者另一个调用者将传递包含 JSON 数据的文件：

```go
f, err := os.Open("data.json")
if err != nil {
  fmt.Println(err)
}
```

我们的 `loadPerson2()` 函数将无法工作；然而，我们的 `loadPerson()` 数据将工作，因为 `os.Open()` 的返回类型满足 `io.Reader{}` 接口。

假设，例如，数据将通过 HTTP 端点传入。我们将从 `*http.Request` 获取数据。再次强调，`loadPerson2()` 函数不是一个好的选择。我们将从 `request.Body` 获取数据，它恰好实现了 `io.Reader{}` 接口。

你可能想知道接口是否适合输入参数。如果是这样，为什么我们还要返回它们呢？如果你返回一个接口，这会给用户增加不必要的难度。用户将不得不查找接口，然后找到方法集和方法集的签名：

```go
func someFunc() Speaker{} {
  // code
}
```

你需要查看 `Speaker{}` 接口的定义，然后花时间查看实现代码，所有这些对于函数的用户来说都是不必要的。如果函数的返回类型需要接口，函数的用户可以为该具体类型创建接口并在他们的代码中使用它。

当你开始遵循这个 Go 谚语时，检查 Go 标准包中是否有接口。这将增加你的函数可以提供的不同实现的数量。我们的函数用户可以通过使用 Go 标准包中的 `io.Reader{}` 接口，使用 `strings.newReader`、`http.Request.Body` 和 `os.File` 等各种实现，就像我们的代码示例一样。

## 空接口

空接口是一个没有方法集和行为的接口。空接口没有指定任何方法：

```go
interface{}
```

这是一个简单但复杂的概念，需要我们理解。如您所知，接口是隐式实现的；没有`implements`关键字。由于空接口没有指定任何方法，这意味着 Go 中的每个类型都自动实现了空接口。所有类型都满足空接口。

在下面的代码片段中，我们将演示如何使用空接口。我们还将看到接受空接口的函数如何允许传递任何类型到该函数：

main.go

```go
package main
import "fmt"
type Speaker interface {
    Speak() string
}
type cat struct {
    name string
}
```

完整的代码可在[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/01d1c9d340172a55335add4ad7adc285b7a51fe4/Chapter07/Example02/main.go`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/01d1c9d340172a55335add4ad7adc285b7a51fe4/Chapter07/Example02/main.go)找到。

预期的输出如下：

```go
({oreo}, main.cat)
({oreo}, main.cat)
(99, int)
(false, bool)
(test, string)
```

让我们分部分评估代码：

```go
func emptyDetails(s interface{}) {
  fmt.Printf("(%v, %T)\n", i, i)
}
```

函数接受一个空的`interface{}`。由于所有类型都实现了空接口，因此可以将任何类型传递给该函数。它将打印值和具体类型。`%v`动词打印值，`%T`动词打印具体类型：

```go
func main() {
  c := cat{name: "oreo"}
  i := 99
  b := false
  str := "test"
  catDetails(c)
  emptyDetails(c)
  emptyDetails(i)
  emptyDetails(b)
  emptyDetails(str)
}
```

我们传递了`cat`类型、`integer`、`bool`和`string`。`emptyDetails()`函数将打印它们中的每一个：

![图 7.9：猫类型实现了空接口 interface{}和 Speaker 接口](img/B18621_07_09.jpg)

图 7.9：猫类型实现了空接口 interface{}和 Speaker 接口

`cat`类型隐式实现了空`interface{}`和`Speaker{}`接口。

现在我们对空接口有了基本的了解，我们将探讨在即将到来的主题中它们的各种用法，包括以下内容：

+   类型切换

+   类型断言

+   Go 包的示例

## 类型断言和 switch

类型断言提供了访问接口具体类型的方法。请记住，`interface{}`可以是任何值：

```go
package main
import (
  "fmt"
)
func main() {
  var str interface{} = "some string"
  var i interface{} = 42
  var b interface{} = true
  fmt.Println(str)
  fmt.Println(i)
  fmt.Println(b)
}
```

类型断言的输出将如下所示：

```go
some string
42
true
```

在每个变量声明的实例中，每个变量都被声明为一个空接口，但`str`的具体值是一个字符串，`i`是整数，`b`是布尔值。

当存在空的 `interface{}` 类型时，有时了解底层具体类型是有益的。例如，您可能需要根据该类型执行数据操作。如果该类型是字符串，您将执行与整数值不同的数据修改和验证。当您消费未知模式的 JSON 数据时，这也适用。在摄入过程中，该 JSON 中的值可能是已知的。我们需要将数据转换为 `map[string]interface{}` 并根据其底层类型或结构执行各种数据按摩或转换。本章后面的活动将向我们展示如何执行此类操作。我们可以使用 `strconv` 包执行类型转换：

```go
package main
import (
  "fmt"
  "strconv"
)
func main() {
  var str interface{} = "some string"
  var i interface{} = 42
  fmt.Println(strconv.Atoi(i))
}
```

![图 7.10：需要类型断言时的错误](img/B18621_07_10.jpg)

图 7.10：需要类型断言时的错误

因此，看起来我们不能使用类型转换，因为类型不兼容类型转换。我们需要使用类型断言：

```go
v := s.(T)
```

上一条语句表示它断言接口值 `s` 是类型 `T`，并将 `v` 的底层值赋给它：

![图 7.11：类型断言流程](img/B18621_07_11.jpg)

图 7.11：类型断言流程

考虑以下代码片段：

```go
package main
import (
  "fmt"
  "strings"
)
func main() {
  var str interface{} = "some string"
  v := str.(string)
  fmt.Println(strings.Title(v))
}
```

让我们再次检查前面的代码：

+   上一段代码断言 `str` 是 `string` 类型，并将其赋值给变量 `v`

+   由于 `v` 是 `string` 类型，它将以标题大小写打印它

结果如下：

```go
Some String
```

当断言与预期类型匹配时，这是很好的。那么，如果 `s` 不是类型 `T` 会发生什么？让我们看看：

```go
package main
import (
  "fmt"
  "strings"
)
func main() {
  var str interface{} = 49
  v := str.(string)
  fmt.Println(strings.Title(v))
}
```

让我们检查前面的代码：

+   `str{}` 是一个空接口，具体类型是 `int`

+   类型断言正在检查 `str` 是否是字符串类型，但在这种情况下，它不是，所以代码将引发恐慌

+   结果如下：

![图 7.12：失败的类型断言](img/B18621_07_12.jpg)

图 7.12：失败的类型断言

不希望抛出恐慌。然而，Go 有一种方法来检查 `str` 是否是字符串：

```go
package main
import (
  "fmt"
)
func main() {
  var str interface{} = "the book club"
  v, isValid := str.(int)
  fmt.Println(v, isValid)
}
```

让我们再次检查前面的代码：

+   类型断言返回两个值，底层值和布尔值。

+   `isValid` 被赋值为 `bool` 类型的返回值。如果它返回 `true`，则表示 `str` 是 `int` 类型。这意味着断言是正确的。我们可以使用返回的布尔值来确定对 `str` 可以采取哪些操作。

+   当断言失败时，它将返回 `false`。返回值将是您试图断言的零值。它也不会引发恐慌。

有时会不知道空接口的具体类型。这就是您将使用类型选择的情况。类型选择可以执行多种类型的断言；它类似于常规的 `switch` 语句。它有 `case` 和 `default` 子句。区别在于类型选择语句评估的是类型而不是值。

这里是一个基本的语法结构：

```go
switch v := i.(type) {
case S:
  // code to act upon the type S
}
```

让我们检查前面的代码：

```go
i.(type)
```

语法类似于类型断言 `i.(int)`，除了在我们的例子中指定的类型 `int` 被替换为 `type` 关键字。被断言的类型 `i` 被分配给 `v`；然后，它被与每个 `case` 语句进行比较：

```go
case S:
```

在 `switch` 类型中，语句用于评估类型。在常规切换中，它们用于评估值。在这里，它用于评估 `S` 的类型。

现在我们已经对类型选择语句有了基本理解，让我们看看一个使用我们刚刚评估的语法的例子：

main.go

```go
func typeExample(i []interface{}) {
    for _, x := range i {
    switch v := x.(type) {
        case int:
            fmt.Printf("%v is int\n", v)
        case string:
            fmt.Printf("%v is a string\n", v)
        case bool:
            fmt.Printf("a bool %v\n", v)
        default:
            fmt.Printf("Unknown type %T\n", v)
        }
    }
}
```

完整的代码可以在[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/main/Chapter07/Example03/main.go`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/main/Chapter07/Example03/main.go)找到。

现在我们将分块探索代码：

```go
func main() {
  c := cat{name: "oreo"}
  i := []interface{}{42, "The book club", true, c}
  typeExample(i)
}
```

在 `main()` 函数中，我们初始化一个变量 `i` 为接口的切片。在这个切片中，我们有 `int`、`string`、`bool` 和 `cat` 类型：

```go
func typeExample(i []interface{})
```

该函数接受一个接口的切片：

```go
  for _, x := range i {
    switch v := x.(type) {
      case int:
        fmt.Printf("%v is int\n", v)
      case string:
        fmt.Printf("%v is a string\n", v)
      case bool:
        fmt.Printf("a bool %v\n", v)
      default:
        fmt.Printf("Unknown type %T\n", v)
    }
  }
```

`for` 循环遍历接口的切片。切片中的第一个值是 `42`。`switch` 情况断言切片值 `42` 是 `int` 类型。`case int` 语句将评估为 `true` 并打印出 `42` 是 `int`。当 `for` 循环遍历到 `cat` 类型的最后一个值时，`switch` 语句将不会在其情况评估中找到该类型。由于在 `case` 语句中没有检查 `cat` 类型，所以默认将执行其 `print` 语句。以下是代码执行的结果：

```go
42 is int
The book club is string
a bool true
Unknown type main.cat
```

## 练习 7.03 – 分析空接口{}数据

在这个练习中，我们得到了一个映射。映射的键是一个字符串，其值是一个空的 `interface{}`。映射的值包含存储在映射值部分中的不同类型的数据。我们的任务是确定每个键的值类型。我们将编写一个程序来分析 `map[string]interface{}` 的数据。理解数据的值可以是任何类型。我们需要编写逻辑来捕获我们不想查找的类型。我们将把信息存储在一个结构体切片中，该切片将包含键名、数据和数据类型：

1.  创建一个名为 `main.go` 的新文件。

1.  在文件内部，我们将有一个 `main` 包，并需要导入 `fmt` 包：

    ```go
    package main
    import (
      "fmt"
    )
    ```

1.  我们将创建一个名为 `record` 的结构体，它将存储来自 `map[string]interface{}` 的键、值类型和数据。这个结构体用于存储我们对映射进行的分析。`key` 字段是映射键的名称。`valueType` 字段存储映射中作为值存储的数据类型。`data` 字段存储我们正在分析的数据。它是一个空的 `interface{}`，因为映射中可能有各种类型的数据：

    ```go
    type record struct {
      key string
      valueType string
      data interface{}
    }
    ```

1.  我们将创建一个 `person` 结构体，它将被添加到我们的 `map[string]interface{}` 中：

    ```go
    type person struct {
      lastName string
      age int
      isMarried bool
    }
    ```

1.  我们将创建一个`animal`结构体，并将其添加到我们的`map[string]interface{}`中：

    ```go
    type animal struct {
      name string
      category string
    }
    ```

1.  创建一个`newRecord()`函数。`key`参数将是我们的映射键。该函数还接受`interface{}`作为输入参数。`i`将是传递给函数的键的映射值。它将返回一个`record`类型：

    ```go
    func newRecord(key string, i interface{}) record {
    ```

1.  在`newRecord()`函数内部，我们初始化`record{}`并将其赋值给`r`变量。然后，我们将`r.key`赋值给键输入参数。

1.  `switch`语句将`i`的类型赋值给`v`变量。`v`变量的类型将与一系列`case`语句进行比较。如果一个类型在某个`case`语句中评估为`true`，那么`valueType`记录将被分配给该类型，同时`v`的值将被分配给`r.data`，然后返回`record`类型：

    ```go
      r := record{}
      r.key = key
      switch v := i.(type) {
      case int:
        r.valueType = "int"
        r.data = v  case bool:
        r.valueType = "bool"
        r.data = v  case string:
        r.valueType = "string"
        r.data = v  case person:
        r.valueType = "person"
    ```

1.  需要`r.data = vA default`语句来配合`switch`语句。如果`v`的类型在`case`语句中没有评估为`true`，那么将执行`default`。`record.valueType`将被标记为`unknown`：

    ```go
      default:
        r.valueType = "unknown"
        r.data = v  }
        return r
    }
    ```

1.  在`main()`函数内部，我们将初始化我们的映射。映射被初始化为一个字符串作为键，一个空接口作为值。然后，我们将`a`赋值给一个`animal`结构体字面量，将`p`赋值给一个`person`结构体字面量。接着，我们开始向映射中添加各种键值对：

    ```go
    func main() {
      m := make(map[string]interface{})
      a := animal{name: "oreo", category: "cat"}
      p := person{lastName: "Doe", isMarried: false, age: 19}
      m["person"] = p
      m["animal"] = a
      m["age"] = 54
      m["isMarried"] = true
      m["lastName"] = "Smith"
    ```

1.  接下来，我们初始化一个`record`切片。我们遍历映射，并将记录添加到`rs`中：

    ```go
      rs := []record{}
      for k, v := range m {
        r := newRecord(k, v)
        rs = append(rs, r)
      }
    ```

1.  现在，打印出记录的字段值。我们遍历记录的切片并打印每个记录值：

    ```go
      for _, v := range rs {
        fmt.Println("Key: ", v.key)
        fmt.Println("Data: ", v.data)
        fmt.Println("Type: ", v.valueType)
        fmt.Println()
      }
    }
    ```

遍历映射可能会产生不同的输出顺序。预期输出的一个例子如下：

![图 7.13：练习的输出](img/B18621_07_13.jpg)

图 7.13：练习的输出

这个练习展示了 Go 语言识别空接口底层类型的能力。正如您从结果中可以看到，我们的类型选择器能够识别每种类型，除了`animal`键的值。它的类型被标记为`unknown`。它甚至能够识别`person`结构体类型，并且数据具有结构体的字段值。

## 活动 7.01 – 计算薪酬和绩效评估

在这个活动中，我们将计算经理和开发者的年度薪酬。我们将打印出开发者和经理的名字以及他们全年的薪酬。开发者的薪酬将基于时薪。开发者类型还将跟踪他们一年中工作的小时数。开发者类型还将包括他们的评估。评估将需要是一个字符串键的集合。这些字符串是开发者正在被评估的类别，例如工作质量、团队合作和沟通。

这个活动的目的是通过调用一个接受接口的单个函数`payDetails()`来演示接口的多态性。这个`payDetails()`函数将打印开发者和经理的薪酬信息。

以下步骤可以帮助你找到解决方案：

1.  创建一个具有`Id`、`FirstName`和`LastName`字段的`Employee`类型。

1.  创建一个具有以下字段的`Developer`类型：`Employee`类型的`Individual`和`HourlyRate`、`HoursWorkedInYear`以及`map[string]interface{}`类型的`Review`。

1.  创建一个具有以下字段的`Manager`类型：`Employee`类型的`Individual`、`Salary`和`CommissionRate`。

1.  创建一个具有`Pay()`方法返回字符串和`float64`的`Payer`接口。

1.  `Developer`类型应该通过返回`Developer`名称和根据`Developer.HourlyRate *` `Developer.HoursWorkInYear`的计算返回开发者年度工资来实现`Payer{}`接口。

1.  `Manager`类型应该通过返回`Manager`名称和根据`Manager.Salary` + (`Manager.Salary *` `Manager.CommissionRate`)的计算返回`Manager`年度工资来实现`Payer{}`接口。

1.  添加一个名为`payDetails`（`p Payer`）的函数，该函数接受一个`Payer`接口并打印`fullName`和从`Pay()`方法返回的工资。

1.  我们现在需要计算一个开发者的评价等级。`Review`是通过`map[string]interface{}`获得的。该映射的键是一个字符串；它是开发者被评价的内容，例如工作质量、团队合作和技能。

1.  映射中的空`interface{}`是必需的，因为一些经理将评价作为字符串给出，而另一些则作为数字给出。以下是字符串到整数的映射：

    ```go
    "Excellent" – 5
    "Good" – 4
    "Fair" – 3
    "Poor" – 2
    "Unsatisfactory" – 1
    ```

1.  我们需要将绩效评价值计算为`float`类型。它是映射`interface{}`的总和除以映射的长度。考虑到评价可以是字符串或整数，因此你需要能够接受这两种类型并将它们转换为浮点数。

预期的输出如下：

```go
Eric Davis got a review rating of 2.80
Eric Davis got paid 84000.00 for the year
Mr. Boss got paid 160500.00 for the year
```

注意

该活动的解决方案可以在本章节的 GitHub 仓库文件夹中找到：[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter07/Activity7.01`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter07/Activity7.01)。

在这个活动中，我们看到了使用空接口的好处，它可以接受任何类型的数据。然后我们使用了类型断言和类型选择语句来根据空接口的底层具体类型执行某些任务。

## any

`interface{}`的`any`关键字。使用`any`类型定义，Go 已经替换了所有对空接口的引用。然而，需要注意的是，它们是可互换的，是类型别名。

# 摘要

本章在介绍接口时提出了一些基本和高级主题。我们了解到 Go 对接口的实现与其他语言有一些相似之处；例如，接口不包含它所表示的行为的实现细节，接口是方法的蓝图。实现接口的不同类型可以在它们的实现细节上有所不同。然而，Go 在实现接口方面与其他语言不同。我们了解到实现是隐式完成的，而不是像其他语言那样显式完成。

这意味着 Go 不支持子类化；因此，为了实现多态性，它使用接口。它允许接口类型以不同的形式出现，例如，`Shape` 接口可以表现为矩形、正方形或圆形。

我们还讨论了接受接口并返回结构体的设计模式。我们演示了这种模式允许其他调用者有更广泛的使用。我们考察了空接口，并看到了在不知道传递的类型或可能传递多个不同类型到您的 API 时如何使用它。尽管在运行时我们不知道类型，但我们向您展示了如何使用类型断言和类型切换来确定类型。我们还看到了有关 `any` 关键字作为空接口类型别名的更新。对这些各种工具的知识和实践将帮助您构建健壮和流畅的程序。

在下一章中，我们将探讨更多关于 Go 1.18 的泛型更新，以及这如何允许开发者为多种类型的变量使用代码！
