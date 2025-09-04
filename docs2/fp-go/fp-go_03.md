# 3

# 高阶函数

在本章中，我们将通过高阶函数来探讨函数组合的概念。这里我们将介绍多种新的概念，例如闭包、偏应用和函数柯里化。我们将查看一些实际例子和这些概念的实际应用场景。

首先，我们将从抽象的角度介绍函数组合的核心概念，然后我们将结合实际例子来应用这些概念。在这里我们所学到的所有内容都高度依赖于在*第二章*中介绍的概念，在那里我们学习了如何将函数视为一等公民。

在本章中，我们将涵盖以下内容：

+   高阶函数简介

+   闭包和变量作用域

+   偏应用

+   函数柯里化，或者如何将 n 元函数简化为一元函数

+   示例：

# 技术要求

本章的所有示例都可以在[`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter3`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter3)找到。对于这个例子，任何版本的 Go 语言都可以使用。

# 高阶函数简介

从本质上讲，高阶函数是指那些要么接受函数作为输入，要么返回函数作为输出的函数。回想一下上一章，这两者都是通过支持函数作为“一等公民”而成为可能的。尽管将它们称为“高阶函数”可能并不常见，但许多编程语言都默认支持这些函数。例如，在 Java 和 Python 中，`map`、`filter`和`reduce`函数都是高阶函数的例子。

让我们在 Go 语言中创建一个简单的例子。我们将有一个返回`hello,`的函数`A`，以及一个接受`A`作为输入参数的函数`B`。这是一个高阶函数，因为`A`函数被用作`B`函数的输入：

```go
func A() string {
     return "hello"
}
func B(a A) string {
     return A() + " world"
}
```

在这里需要指出的是，我们并不是简单地将`A`函数的结果传递给`B`函数——我们实际上是在执行`B`函数的过程中运行`A`函数。到目前为止，我所展示的内容与我们之前在*第二章*中看到的内容并没有本质的不同。实际上，一等函数通常是通过高阶函数的实现来展示的。

当它们变得有趣时，是你开始使用它们进行偏应用计算，或者使用它们来构建函数柯里化的时候，但在我们深入这些内容之前，让我们先看看闭包的概念。

# 闭包和变量作用域

闭包与特定编程语言中变量作用域的工作方式密切相关。为了完全理解它们是如何工作的以及它们如何变得有用，我们首先将快速回顾一下 Go 中变量作用域是如何工作的。接下来，我们将提醒自己匿名函数是如何工作的以及它们是什么。最后，我们将探讨在这个上下文中闭包是什么。这将为我们理解稍后章节中提到的部分应用和函数柯里化打下基础。

## Go 中的变量作用域

Go 中的变量作用域是通过所谓的 **词法作用域** 来实现的。这意味着变量在其创建的上下文中被识别并可用。在 Go 中，“块”用于界定代码中的位置。例如，看以下代码：

```go
package main
import "fmt"
// location 1
func main() {
     // location 2
     b := true
     if b {
          // location 3
          fmt.Println(b)
     }
}
```

这段代码中有三个作用域位置：

+   第一个位置，`location 1`，是包作用域。我们的主函数位于这个作用域级别。

+   下一个位置是在我们的 `main` 函数内部。这就是我们定义 `b` 布尔变量的地方。

+   第三个位置是在 `if` 语句内部。在 Go 以及许多其他语言中，块是由花括号定义的。

注意

通常情况下，在“更高位置”定义的变量在较低位置是可用的，但在较低位置定义的变量在周围较高位置是不可用的。在前面的例子中，我们的代码按预期工作，因为 `b` 可以从 `location 3` 内部访问，尽管它是在 `location 2` 中定义的。

到目前为止，对于经验丰富的 Go 程序员来说，这一切应该都表现得相当符合预期。让我们看看一些其他关于范围示例。在继续阅读之前，试着找出代码的输出：

范围示例 1：

```go
 func main() {
      {
           b := true
     }
     if b {
          fmt.Println("b is true")
     }
}
```

这里的输出会是什么？正确答案是… *编译错误*。在这个例子中，我们定义 `b` 的作用域与 `if` 块的作用域不同。因此，在这个作用域级别我们没有访问 `b` 的权限。

现在，想想这里的输出会是什么：

范围示例 2：

```go
func main() {
     s := "hello"
     if true {
          s := "world"
          fmt.Println(s)
     }
     fmt.Println(s)
}
```

正确答案是 `world hello`。这可能会让人有些惊讶。你知道在 Go 中你不能在给定的作用域中重新声明一个变量，但在这个例子中，`if` 语句内部的范围与我们的 `main` 函数的范围不同。因此，在 `if` 函数内部声明新的 `s` 变量是有效的。请注意，当我们使用在 `if` 语句外部声明的 `s` 变量时，它保持未变。这可能会有些令人惊讶的行为。让我们在跳到第三个例子之前稍微修改一下我们的代码。

让我们尝试猜测以下示例的输出可能是什么：

范围示例 3：

```go
func main() {
     s := "hello"
     if true {
           s = "world"
           fmt.Println(s)
     }
     fmt.Println(s)
}
```

为了指出这个片段中的差异，我们将 `if` 语句中的第一行从以下内容更改：

```go
S := world
```

现在，情况如下：

```go
S = world
```

这个看似微小的差异产生了以下输出：`world world`。要理解这一点，请记住，当我们使用`:=`语法时，我们是在声明一个新的变量。当我们只写`=`时，我们是在重新声明一个现有的变量。在这个例子中，我们只是在更新`s`变量的内容。

现在，让我们对这个例子做一个最后的修改：

作用域示例 4：

```go
func main() {
      s := "hello"
      s := "world"
      fmt.Println(s)
}
```

如你所猜到的，这段代码无法编译。虽然 Go 允许我们声明具有相同名称的变量，但它只允许我们在它们不在同一块作用域内时这样做。这里的一个显著例外是当函数返回多个值时。例如，在下面的代码片段中，我们可以将错误值重新声明为`func1`和`func2`的返回值：

```go
func main() {
      str1, err := func1()
      if err != nil {
           panic(err)
      }
      str2, err := func2()
      if err != nil {
           panic(err)
      }
      fmt.Printf("%v %v\n", str1, str2)
}
func func1() (string, error) {
      return "", errors.New("error 1")
}
func func2() (string, error) {
      return "", errors.New("error 2")
}
```

在前面的代码片段中，尽管我们使用了`:=`语法，但`err`值仍然被重新声明。这在 Go 语言中很常见，因为错误值会从每个函数向上冒泡，最终到达处理多个错误的事件处理方法。

记住作用域的工作方式和花括号在界定块中的作用，以及记住引入新变量与简单重新声明现有变量之间的区别非常重要。这样，我们就有了足够的背景知识，可以深入探讨在函数内部使用函数时的变量作用域。

## 在函数中捕获变量上下文（闭包）

在上一章中，我们看到了每次遇到花括号时，都会引入一个新的变量作用域。这发生在我们声明一个函数、分支到`if`语句、引入`for`循环，或者在函数的任何地方放置花括号时，就像我们的第一个作用域示例一样。我们还看到在*第二章*中，我们可以在函数内部创建函数——正如你可能猜到的，这又创建了一个新的作用域。

在本章的剩余部分，我们将频繁使用匿名函数。请记住，匿名函数本质上是一个没有与标识符关联的函数声明。这是我们使用的通用模板：

```go
// location 1
func outerFunction() func() {
     // location 2
     fmt.Println("outer function")
     return func() {
           // location 3
           fmt.Println("inner function")
     }
}
```

在这个例子中，我还标记了三个变量作用域的位置。正如你所看到的，`位置 3`，它是匿名函数的一部分，其作用域比`位置 2`低。这是闭包能够工作的一个关键原因。定义一个新的函数并不会自动创建一个顶级作用域。当我们在一个函数内部定义另一个函数时，这个新函数的作用域比其引入的位置低。

此外，请注意`outerFunction`是一个高阶函数。尽管我们没有将函数作为输入，但我们返回一个函数作为输出。这是高阶函数的有效特征。

现在，让我们具体说明我们所说的闭包是什么。闭包是指**任何使用在外部函数中引入的变量来执行其工作的内部函数**。让我们通过一个例子来使这个概念更加具体。

在这个例子中，我们将创建一个函数，该函数创建一个问候函数。我们的外部函数将是确定要显示的问候信息的函数。内部函数将要求输入一个名字，并返回与名字结合的问候语：

```go
func main() {
     greetingFunc := createGreeting()
     response := greetingFunc("Ana")
     fmt.Println(response)
}
func createGreeting() func(string) string {
     s := "Hello "
     return func(name string) string {
          return s + name
     }
}
```

在前面的例子中，我们使用了闭包。匿名内部函数引用外部变量`s`来创建问候。这段代码的输出是`Hello Ana`。这里重要的是，尽管`s`变量在`createGreeting`函数结束时已经超出作用域，但变量内容实际上被捕获在内部函数中。因此，在我们`main`函数中调用`greetingFunc`之后，捕获被固定为`Hello`。在内部函数中捕获变量就是我们所说的闭包的含义。

我们可以通过将问候字符串作为`createGreeting`函数的输入参数来使这个函数更加灵活，从而得到以下结果：

```go
func createGreeting(greeting string) func(string) string {..}
```

这个小小的改变带我们进入了下一个主题的开始：偏应用。

# 偏应用

现在我们已经理解了闭包，我们可以开始考虑偏应用。名称“偏应用”非常明确地告诉我们发生了什么——它是一个部分应用的函数。这也许仍然有点晦涩。一个部分应用的函数正在接受一个接受*N*个参数的函数，并“固定”这些参数的一个子集。通过固定参数的一个子集，它们就变得固定不变，而其他输入参数仍然保持灵活。

这最好用一个例子来说明。让我们扩展我们在本章前一部分中构建的`createGreeting`函数：

```go
func createGreeting(greeting string) func(string) string {
     return func(name string) string {
          return greeting + name
     }
}
```

我们在这里所做的改变是将问候语作为输入传递给`createGreeting`函数。每次我们调用`createGreeting`时，实际上我们都在创建一个新的函数，该函数期望`name`作为输入，但`greeting`字符串是固定的。现在让我们创建几个这样的函数并使用它们来打印输出：

```go
func main() {
     firstGreeting := createGreeting("Well, hello there ")
     secondGreeting := createGreeting("Hola ")
     fmt.Println(firstGreeting("Remi"))
     fmt.Println(firstGreeting("Sean"))
     fmt.Println(secondGreeting("Ana"))
}
```

运行此函数的输出如下：

```go
Well, hello there Remi
Well, hello there Sean
Hola Ana
```

在这个例子中，我们将`firstGreeting`函数的第一个参数固定为`Well, hello there,`，而对于`secondGreeting`函数，我们将值固定为`Hola`。这是偏应用——当我们创建用于问候用户的函数时，这个函数的部分已经被应用。在这种情况下，`greeting`变量被固定，但你也可以固定函数参数的任何子集——这不仅仅局限于一个变量。

### 示例：DogSpawner

在这个例子中，我们将把到目前为止所学的一切结合起来。对于这个例子，我们将创建 `DogSpawner`。你可以想象这可以用于创建游戏或需要维护狗信息的其他应用程序的上下文中。就像我们之前的例子一样，我们将将其简化到最基本的部分，并且不会制作一个真正的游戏。然而，在这个例子中，我们将利用之前章节中学到的知识，并用干净的函数式代码将它们全部结合起来。

从一个高层次的角度来看，我们的应用程序应该支持多种品种的狗。品种应该是易于扩展的。我们还希望记录狗的性别并给狗起一个名字。在我们的例子中，假设你想要生成许多狗，那么就会有大量的类型和性别的重复。我们将利用部分应用来防止这些函数调用的重复性，并提高代码的可读性。

首先，我们将定义这个程序需要的类型。记得从第一章中，我们可以使用 `type` 系统来提供有关代码中发生情况的更多信息：

```go
type (
     Name          string
     Breed         int
     Gender        int
     NameToDogFunc func(Name) Dog
)
```

注意，我们可以使用一个 `type` 块，类似于我们如何使用 `var` 或 `const` 块。这可以防止我们不得不重复 `type Name string` 结构。在这个 `type` 块中，我们简单地将 `Name` 选择为 `string` 对象，`Breed` 和 `Gender` 选择为 `int` 对象，而 `NameToDogFunc` 是一个接受给定的 `Name` 并返回一个给定的 `Dog` 结果的函数。我们选择 `int` 对象作为 `Breed` 和 `Gender` 的原因是我们将使用 Go 的 `Enum` 定义等效物来构建它们。我们将继续填充这些枚举值：

```go
// define possible breeds
const (
     Bulldog Breed = iota
     Havanese
     Cavalier
     Poodle
)
// define possible genders
const (
     Male Gender = iota
     Female
)
```

如前例所示，默认的 `iota` 关键字与我们所定义的类型无缝配合。再次证明，我们的类型别名编译成底层类型，在这种情况下，是 `int` 类型，`iota` 就定义在这个类型上。你可以在本例中将两个 `const` 块合并为一个块，但在处理枚举时，每个块只服务于单一目的时代码更易于阅读。

在这些常量和类型就绪后，我们可以创建一个结构体来表示我们的 `Dog`：

```go
type Dog struct {
     Name   Name
     Breed  Breed
     Gender Gender
}
```

在这个结构体中有点重复，因为我们的变量名与类型相同。对于这个例子，我们可以保持它轻量级，不需要向我们的 `Dog` 添加更多信息。有了这个，我们就有了开始实现部分应用函数所需的一切，但在我们到达那里之前，让我们看看在没有部分应用函数的情况下如何创建 `Dog` 结构体：

```go
func createDogsWithoutPartialApplication() {
     bucky := Dog{
           Name:   "Bucky",
           Breed:  Havanese,
           Gender: Male,
     }
     rocky := Dog{
           Name:   "Rocky",
           Breed:  Havanese,
           Gender: Male,
     }
     tipsy := Dog{
           Name:   "Tipsy",
           Breed:  Poodle,
           Gender: Female,
     }
}
```

在前面的例子中，我们创建了三只狗。前两只都是雄性巴厘犬，因此我们不得不在那里重复`品种`和`性别`信息。这两只狗之间唯一独特的东西就是名字。现在，让我们创建一个函数，允许我们创建具有各种性别和品种组合的`DogSpawner`：

```go
func DogSpawner(breed Breed, gender Gender) NameToDogFunc {
     return func(n Name) Dog {
           return Dog {
                 Breed:  breed,
                 Gender: gender,
                 Name:   n,
           }
     }
}
```

前面的`DogSpawner`函数是一个接受`品种`和`性别`作为输入的函数。它返回一个新的函数`NameToDogFunc`，该函数接受`名字`作为输入并返回一个新的`Dog`结构体。因此，`DogSpawner`函数允许我们创建新的函数，其中狗的品种和性别已经部分应用，但名字仍然需要作为输入。

使用`DogSpawner`函数，我们可以创建两个新的函数，`maleHavaneseSpawner`和`femalePoodleSpawner`。这些函数将允许我们通过只提供狗的名字来创建雄性巴厘犬和雌性贵宾犬。让我们继续在包作用域的`var`块中创建两个新的函数：

```go
var (
     maleHavaneseSpawner = DogSpawner(Havanese, Male)
     femalePoodleSpawner = DogSpawner(Poodle, Female)
)
```

在这个定义之后，`maleHavaneseSpawner`和`femalePoodleSpawner`函数可以在该包的任何地方使用。你也可以将它们公开为任何使用该包的人都可以访问的公共函数。让我们在我们的`main`函数中演示这些函数如何被使用：

```go
func main() {
     bucky := maleHavaneseSpawner("bucky")
     rocky := maleHavaneseSpawner("rocky")
     tipsy := femalePoodleSpawner("tipsy")
     fmt.Printf("%v\n", bucky)
     fmt.Printf("%v\n", rocky)
     fmt.Printf("%v\n", tipsy)
}
```

在这个`main`函数中，我们可以看到如何利用部分应用函数。我们本可以创建一个创建狗的函数，例如`newDog(n Name, b Breed, g Gender) Dog{}`，但这仍然会导致在创建我们的狗时有很多重复，如下所示：

```go
func main() {
     createDog("bucky", Havanese, Male)
     createDog("rocky", Havanese, Male)
     createDog("tipsy", Poodle, Female)
     createDog("keeno", Cavalier, Male)
}
```

尽管只有三个参数时仍然相当易读，但更多的参数将显著降低可读性。我们将在讨论了函数柯里化之后，在本章的最后示例中展示这一点。

# 函数柯里化，或如何将多元函数简化为一元函数

函数柯里化常被误认为是部分应用。正如你将看到的，函数柯里化和部分应用是相关但不同的概念。当我们谈论函数柯里化时，我们是在谈论将接受单个参数的函数转换为一个序列的函数，其中每个函数恰好接受一个参数。在伪代码中，我们所做的是将如下函数转换为一个由三个函数组成的序列：

```go
func F(a,b,c): int {}
```

第一个函数`(Fa)`接受`a`参数作为输入并返回一个新的函数`(Fb)`作为输出。`(Fb)`接受`b`作为输入并返回一个`(Fc)`函数。`(Fc)`是最终函数，它接受`c`作为输入并返回一个`int`对象作为输出：

```go
func Fa(a): Fb(b)
func Fb(b): Fc(c)
func Fc(c): int
```

这是通过再次利用第一公民和高阶函数的概念来实现的。我们将能够通过从函数中返回一个函数来实现这种转换。我们将从这一核心特性中获得更可组合的函数。就我们的目的而言，你可以将这视为单参数的部分应用。

这里要注意的一点是，在其他编程语言如 Haskell 中，函数柯里化比在我们的 Go 示例中扮演着更重要的角色。Haskell（以 Haskell Curry 的名字命名），将每个函数转换为一个柯里化函数。编译器会处理这件事，所以作为用户你通常不会意识到这一点。Go 编译器不做这样的事情，但我们仍然可以手动以这种方式创建函数。在我们深入更大的端到端示例之前，让我们快速看看如何将之前的伪代码转换为功能性的 Go 代码。

没有柯里化，我们的函数将看起来像这样：

```go
func threeSum(a, b, c int) int {
     return a + b + c
}
```

现在，有了柯里化，相同的例子将转化为这样：

```go
func threeSumCurried(a int) func(int) func(int) int {
     return func(b int) func(int) int {
          return func(c int) int {
               return a + b + c
          }
     }
}
```

当在`main`函数中调用它们时，这些返回相同的结果。注意`main`函数中两次调用之间的语法差异：

```go
func main() {
     fmt.Println(threeSum(10, 20, 30))
     fmt.Println(threeSumCurried(10)(20)(30))
}
```

不言而喻，这个函数的柯里化版本比非柯里化版本更难阅读和理解。这回到了我在第一章提到的事情——你应该在合理的地方利用函数式概念。对于这个简单的例子，这样做并不合理，但它确实展示了我们试图做到的事情。函数柯里化的真正力量只有在我们也决定将其与部分应用结合以创建灵活函数时才会派上用场。为了展示这是如何工作的，让我们深入一个例子。

## 示例：柯里化函数

在这个例子中，我们将扩展我们构建的`DogSpawner`示例的功能，以演示部分应用。如果我们查看该应用程序的`main`函数中的主要`DogSpawner`代码，我们可以看出我们几乎使用了一个一元函数：

```go
func DogSpawner(breed Breed, gender Gender) NameToDogFunc {
     // implementation
}
```

这让我们接近了，但还不够。为了成为一个正确的柯里化函数，`DogSpawner`只能接受一个参数。本质上，我们将创建一个由三个函数组成的序列，这些函数依次接受参数来创建`Dog`，`DogSpawner(Breed)(Gender)(Name)`。如果我们用 Go 实现这个函数，我们得到以下代码：

```go
func DogSpawnerCurry(breed Breed) func(Gender) NameToDogFunc {
     return func(gender Gender) NameToDogFunc {
            return func(name Name) Dog {
                   return Dog{
                          Breed:  breed,
                          Gender: gender,
                       Name:   name,
               }
          }
     }
}
```

读取这个的方式是`DogSpawnerCurry`是一个接受`breed`作为输入的函数。它返回一个接受`gender`作为输入的函数，而这个函数反过来又返回一个接受`name`作为输入的函数，该函数返回`Dog`。这有点复杂，但你会习惯的。这也是类型别名派上用场的地方。如果没有类型别名，这将更加冗长，这会阻碍阅读并使编写时更容易出错：

```go
func DogSpawnerCurry(breed Breed) func(Gender) func(Name) Dog {
     return func(gender Gender) func(Name) Dog{
          return func(name Name) Dog {
               return Dog{
                    Breed:  breed,
                    Gender: gender,
                    Name:   name,
               }
          }
     }
}
```

现在我们已经涵盖了本章的三个主要主题，让我们看看一些进一步的例子来展示这些技术。

# 示例：服务器构造函数

在这个第一个例子中，我们将利用到目前为止所学到的知识来创建数据类型的灵活构造函数。我们还将看到如何创建具有我们选择的默认值的构造函数。

在我们的设置中，`Server` 结构体是一个简单的结构体，具有固定数量的最大连接数、传输类型和名称。我们不会构建实际的 Web 服务器，而是仅用少量开销来展示这些概念。我们在这个示例中想要关注的是核心思想，您可以在任何合适的地方应用这些思想。我们的服务器只有三个可配置的参数，但您可以想象，当有更多参数需要配置时，这种好处更为明显。

和往常一样，我们将从定义应用程序的自定义类型开始。为了保持轻量级，我定义了两个——`TransportType`，它是一个用作枚举的 `int` 类型，以及 `func(options) options` 的类型别名。让我们也为 `TransportType` 设置一些值：

```go
type (
     ServerOptions func(options) options
     TransportType int
)
const (
     UDP TransportType = iota
     TCP
)
```

现在我们有了这个，让我们将结构体放到位——我们将用作 `Server` 和 `options` 的两个结构体：

```go
type Server struct {
     options
}
type options struct {
     MaxConnection int
     TransportType TransportType
     Name          string
}
```

在此示例中，我们未为嵌入的字段声明新名称就嵌入了 `options`。在 Go 中，这可以通过简单地写出您想要嵌入的结构体的类型来实现。这样做时，`Server` 结构体会包含 `options` 结构体具有的所有字段。这是在 Go 中建模对象组合的一种方式。

这可能看起来有点奇特，需要进一步调查。在更典型的设置中，您可能会在 `Server` 结构体中包含我们放在 `options` 结构体内部的变量。使用 `options` 结构体并将其嵌入 `Server` 的主要原因是为了将其用作我们希望用户提供的服务器配置。我们不希望用户提供不包含在此结构体中的数据，例如 `isAlive` 标志。这清楚地分离了关注点，并允许我们在其之上构建更高阶的函数和部分应用层。

下一步是为我们创建一种通过多次函数调用配置 `options` 结构体的方式。对于 `options` 结构体内部的每个变量，我们创建一个高阶函数。这些函数接收要配置的参数，并返回一个新的函数，`ServerOptions`：

```go
func MaxConnection(n int) ServerOptions {
     return func(o options) options {
     o.MaxConnection = n
          return o
     }
}
func ServerName(n string) ServerOptions {
     return func(o options) options {
          o.Name = n
          return o
     }
}
func Transport(t TransportType) ServerOptions {
        return func(o options) options {
                o.TransportType = t
                return o
        }
}
```

如您在前三个函数（`MaxConnection`、`ServerName` 和 `TransportType`）中看到的，我们正在使用闭包来构建这个配置。每个函数接收一个 `options` 类型的结构体，更改相应的变量，并返回应用更改后的相同 `options` 结构体。*请注意，这些函数只更改它们对应的变量，结构体中的其他所有内容都保持不变*。

现在我们有了这个，我们就准备好了一切，可以开始构建我们的服务器。对于我们的构造器，我们将编写一个函数，它接受一个可变参数列表`ServerOptions`作为输入。请记住，这些输入实际上是其他函数。我们的构造器是一个高阶函数，它接受函数作为输入，并返回服务器作为输出。因此，当我们遍历我们的`ServerOptions`时，我们得到一系列可以调用的函数。我们将创建一个默认的`options`结构体，并将其传递给这些函数：

```go
func NewServer(os ...ServerOptions) Server {
     opts := options{}
     for _, option := range os {
          opts = option(opts)
     }
     return Server{
          options: opts,
          isAlive: true,
     }
}
```

在这里的代码中，您可以看到我们的`Server`是如何基于`options`结构体最终构建的。我们还设置了`isAlive`标志为`true`，因为这不是用户可以输入的内容。

太好了，我们已经准备好了一切，可以开始创建服务器了——那么我们该如何着手呢？嗯，我们的构造器与您可能见过的其他构造器略有不同。我们不是以原语或结构体等变量作为输入，而是将函数作为输入传递。让我们在`main`函数中演示如何调用这个构造器：

```go
func main() {
     server := NewServer(MaxConnection(10), ServerName("MyFirstServer"))
     fmt.Printf("%+v\n", server)
}
```

如您所知，我们在构造器内部调用了`MaxConnection(10)`函数。这个函数的输出不仅仅是结构体；输出是`function(options) options`。当运行这段代码时，我们得到以下输出：

```go
{options:{MaxConnection:10 TransportType:0 Name:MyFirstServer} 
  isAlive:true}
```

太好了——现在，我们有一个相当灵活的构造器。如果您注意输出，我们会得到`TransportType: 0`作为输出，即使我们没有在我们的`options`结构体中配置这个。这是因为 Go 为其原始类型使用了一个合理的默认零值。我们当前构造器设置允许我们做的事情之一是，通过仅对我们的代码进行少量修改，就可以创建我们自行设置的默认值。让我们更新`NewServer`函数，使用`TCP`（`TransportType: 1`）作为默认值：

```go
func NewServer(os ...ServerOptions) Server {
        opts := options{
                TransportType: TCP,
        }
        for _, option := range os {
                opts = option(opts)
        }
        return Server{
                options: opts,
                isAlive: true,
        }
}
```

在这个例子中，我们唯一做的改变是在`options`的初始化中添加了`TransportType: TCP`。现在，如果我们再次运行相同的`main`代码，我们得到以下输出：

```go
{options:{MaxConnection:10 TransportType:1 Name:MyFirstServer} 
  isAlive:true}
```

这就是当用户没有提供任何内容时创建我们自己的默认值有多简单。正如这个例子所示，我们可以轻松地使用函数式编程的概念来构建灵活的函数，如构造器，并实现 Go 中本不存在的功能。在一些语言中，例如 Python，当用户没有提供时，您可以为一个函数设置默认值。现在，我们可以使用我们的服务器`options`结构体来做同样的事情。

# 摘要

在本章中，我们涵盖了三个主题：闭包、偏应用和柯里化。通过使用闭包，我们学习了如何在内外函数之间共享变量上下文。这使得我们能够构建灵活的应用程序，例如最后的“构造函数”示例。接下来，我们学习了如何使用偏应用函数将某些参数固定到多元函数中。这展示了我们如何为函数创建默认配置，例如在我们示例中创建的`HavaneseSpawner`选项。最后，我们了解了函数柯里化及其与偏应用的关系。我们展示了如何通过将每个函数转换为单参数函数调用来扩展我们的偏应用示例。这三种技术使我们能够创建更多可组合和可重用的函数。

到目前为止，我们并没有关注函数的纯度，对系统的状态处理得有些草率。在下一章中，我们将讨论函数纯度的含义，如何封装副作用，以及这对编写可测试代码带来的好处。
