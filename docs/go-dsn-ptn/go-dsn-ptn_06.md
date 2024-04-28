# 第六章：行为模式 - 模板、备忘录和解释器设计模式

在本章中，我们将看到接下来的三种行为设计模式。难度正在提高，因为现在我们将使用结构和创建模式的组合来更好地解决一些行为模式的目标。

我们将从模板设计模式开始，这是一种看起来非常类似于策略模式但提供更大灵活性的模式。备忘录设计模式在我们每天使用的 99%的应用程序中使用，以实现撤销功能和事务操作。最后，我们将编写一个逆波兰符号解释器来执行简单的数学运算。

让我们从模板设计模式开始。

# 模板设计模式

**模板**模式是那些广泛使用的模式之一，特别是在编写库和框架时非常有用。其思想是为用户提供一种在算法中执行代码的方式。

在本节中，我们将看到如何编写成语法 Go 模板模式，并查看一些智能使用的 Go 源代码。我们将编写一个三步算法，其中第二步委托给用户，而第一步和第三步则不是。算法中的第一步和第三步代表模板。

## 描述

与策略模式封装算法实现在不同的策略中不同，模板模式将尝试实现类似的功能，但只是算法的一部分。

模板设计模式允许用户在算法的一部分中编写代码，而其余部分由抽象执行。这在创建库以简化某些复杂任务或者某些算法的可重用性仅受其一部分影响时很常见。

例如，我们有一个长时间的 HTTP 请求事务。我们必须执行以下步骤：

1.  验证用户。

1.  授权他。

1.  从数据库中检索一些详细信息。

1.  进行一些修改。

1.  将详细信息发送回一个新的请求。

不要重复步骤 1 到 5 在用户的代码中，每次他需要修改数据库中的内容。相反，步骤 1、2、3 和 5 将被抽象成相同的算法，该算法接收一个接口，其中包含第五步需要完成事务的内容。它也不需要是一个接口，它可以是一个回调。

## 目标

模板设计模式关乎可重用性和将责任交给用户。因此，这种模式的目标如下：

+   将库的算法的一部分延迟给用户。

+   通过抽象代码的部分来提高可重用性，这些部分在执行之间是不常见的

## 示例 - 一个包含延迟步骤的简单算法

在我们的第一个示例中，我们将编写一个由三个步骤组成的算法，每个步骤都返回一个消息。第一步和第三步由模板控制，只有第二步延迟到用户。

## 需求和验收标准

模板模式必须做的事情的简要描述是定义一个算法的模板，其中包括三个步骤，将第二步的实现延迟到用户：

1.  算法中的每个步骤必须返回一个字符串。

1.  第一步是一个名为`first()`的方法，返回字符串`hello`。

1.  第三步是一个名为`third()`的方法，返回字符串`template`。

1.  第二步是用户想要返回的任何字符串，但它由`MessageRetriever`接口定义，该接口具有`Message() string`方法。

1.  算法由一个名为`ExecuteAlgorithm`的方法顺序执行，并返回每个步骤返回的字符串，这些字符串由一个空格连接在一起。

## 简单算法的单元测试

我们将专注于仅测试公共方法。这是一个非常常见的方法。总的来说，如果你的私有方法没有从公共方法的某个级别被调用，它们根本就没有被调用。我们需要两个接口，一个用于模板实现者，一个用于算法的抽象步骤：

```go
type MessageRetriever interface { 
  Message()string 
} 

type Template interface { 
   first() string 
   third() string 
   ExecuteAlgorithm(MessageRetriever) string 
} 

```

模板实现者将接受一个`MessageRetriever`接口作为其执行算法的一部分。我们需要一个实现这个接口的类型，称为`Template`，我们将称之为`TemplateImpl`：

```go
type TemplateImpl struct{} 

func (t *TemplateImpl) first() string { 
  return "" 
} 

func (t *TemplateImpl) third() string { 
  return "" 
} 

func (t *TemplateImpl) ExecuteAlgorithm(m MessageRetriever) string { 
  return "" 
} 

```

因此，我们的第一个测试检查了第四和第五个验收标准。我们将创建一个实现`MessageRetriever`接口的`TestStruct`类型，返回字符串`world`并嵌入模板，以便它可以调用`ExecuteAlgorithm`方法。它将充当模板和抽象：

```go
type TestStruct struct { 
  Template 
} 

func (m *TestStruct) Message() string { 
  return "world" 
} 

```

首先，我们将定义`TestStruct`类型。在这种情况下，我们需要延迟到我们的算法的部分将返回`world`文本。这是我们稍后在测试中要查找的字符串，进行类型检查“这个字符串中是否存在`world`这个词？”。

仔细看，`TestStruct`嵌入了一个名为`Template`的类型，它代表了我们算法的模板模式。

当我们实现`Message()`方法时，我们隐式地实现了`MessageRetriever`接口。所以现在我们可以将`TestStruct`类型用作`MessageRetriever`接口的指针：

```go
func TestTemplate_ExecuteAlgorithm(t *testing.T) { 
  t.Run("Using interfaces", func(t *testing.T){ 
    s := &TestStruct{} 
    res := s.ExecuteAlgorithm(s) 
   expected := "world" 

    if !strings.Contains(res, expected) { 
      t.Errorf("Expected string '%s' wasn't found on returned string\n", expected) 
    } 
  }) 
} 

```

在测试中，我们将使用刚刚创建的类型。当我们调用`ExecuteAlgorithm`方法时，我们需要传递`MessageRetriever`接口。由于`TestStruct`类型也实现了`MessageRetriever`接口，我们可以将其作为参数传递，但这当然不是强制的。

根据第五个验收标准中定义的`ExecuteAlgorithm`方法的结果，必须返回一个字符串，其中包含`first()`方法的返回值，`TestStruct`的返回值（`world`字符串）和`third()`方法的返回值，它们之间用一个空格分隔。我们的实现在第二个位置；这就是为什么我们检查了字符串`world`前后是否有空格。

因此，当调用`ExecuteAlgorithm`方法时返回的字符串不包含字符串`world`时，测试失败。

这已经足够使项目编译并运行应该失败的测试：

```go
go test -v . 
=== RUN   TestTemplate_ExecuteAlgorithm
=== RUN   TestTemplate_ExecuteAlgorithm/Using_interfaces
--- FAIL: TestTemplate_ExecuteAlgorithm (0.00s)
 --- FAIL: TestTemplate_ExecuteAlgorithm/Using_interfaces (0.00s)
 template_test.go:47: Expected string ' world ' was not found on returned string
FAIL
exit status 1
FAIL

```

是时候转向实现这种模式了。

## 实现模板模式

根据验收标准的定义，我们必须在`first()`方法中返回字符串`hello`，在`third()`方法中返回字符串`template`。这很容易实现：

```go
type Template struct{} 

func (t *Template) first() string { 
  return "hello" 
} 

func (t *Template) third() string { 
  return "template" 
} 

```

通过这种实现，我们应该覆盖*第二*和*第三*个验收标准，并部分覆盖*第一个*标准（算法中的每个步骤必须返回一个字符串）。

为了满足*第五*个验收标准，我们定义了一个`ExecuteAlgorithm`方法，它接受`MessageRetriever`接口作为参数，并返回完整的算法：通过连接`first()`、`Message() string`和`third()`方法返回的字符串来完成：

```go
func (t *Template) ExecuteAlgorithm(m MessageRetriever) string { 
  return strings.Join([]string{t.first(), m.Message(), t.third()},  " ") 
} 

```

`strings.Join`函数具有以下签名：

```go
func Join([]string,string) string 

```

它接受一个字符串数组并将它们连接起来，将第二个参数放在数组中的每个项目之间。在我们的情况下，我们创建一个字符串数组，然后将其作为第一个参数传递。然后我们传递一个空格作为第二个参数。

通过这种实现，测试必须已经通过了：

```go
go test -v . 
=== RUN   TestTemplate_ExecuteAlgorithm 
=== RUN   TestTemplate_ExecuteAlgorithm/Using_interfaces 
--- PASS: TestTemplate_ExecuteAlgorithm (0.00s) 
    --- PASS: TestTemplate_ExecuteAlgorithm/Using_interfaces (0.00s) 
PASS 
ok

```

测试通过了。测试已经检查了返回结果中是否存在字符串`world`，这是`hello world template`消息。`hello`文本是`first()`方法返回的字符串，`world`字符串是我们的`MessageRetriever`实现返回的，`template`是`third()`方法返回的字符串。空格是由 Go 的`strings.Join`函数插入的。但是，任何对`TemplateImpl.ExecuteAlgorithm`类型的使用都将始终在其结果中返回“hello [something] template”。

## 匿名函数

这不是实现模板设计模式的唯一方法。我们还可以使用匿名函数将我们的实现传递给`ExecuteAlgorithm`方法。

让我们在之前使用的相同方法中编写一个测试，就在测试之后（用粗体标记）：

```go
func TestTemplate_ExecuteAlgorithm(t *testing.T) { 
  t.Run("Using interfaces", func(t *testing.T){ 
    s := &TestStruct{} 
    res := s.ExecuteAlgorithm(s) 

    expectedOrError(res, " world ", t) 
  }) 

 t.Run("Using anonymous functions", func(t *testing.T)
  {
 m := new(AnonymousTemplate)
 res := m.ExecuteAlgorithm(func() string {
 return "world"
 })
 expectedOrError(res, " world ", t)
 }) 
} 

func expectedOrError(res string, expected string, t *testing.T){ 
  if !strings.Contains(res, expected) { 
    t.Errorf("Expected string '%s' was not found on returned string\n", expected) 
  } 
} 

```

我们的新测试被称为*使用匿名函数*。我们还将测试中的检查提取到一个外部函数中，以便在这个测试中重用它。我们将这个函数称为`expectedOrError`，因为如果没有收到预期的值，它将失败并抛出错误。

在我们的测试中，我们将创建一个名为`AnonymousTemplate`的类型，用它来替换之前的`Template`类型。这个新类型的`ExecuteAlgorithm`方法接受`func() string`类型的方法，我们可以直接在测试中实现它来返回字符串`world`。

`AnonymousTemplate`类型将具有以下结构：

```go
type AnonymousTemplate struct{} 

func (a *AnonymousTemplate) first() string { 
  return "" 
} 

func (a *AnonymousTemplate) third() string { 
  return "" 
} 

func (a *AnonymousTemplate) ExecuteAlgorithm(f func() string) string { 
  return "" 
} 

```

与`Template`类型的唯一区别是`ExecuteAlgorithm`方法接受一个返回字符串的函数，而不是`MessageRetriever`接口。让我们运行新的测试：

```go
go test -v .
=== RUN   TestTemplate_ExecuteAlgorithm
=== RUN   TestTemplate_ExecuteAlgorithm/Using_interfaces
=== RUN   TestTemplate_ExecuteAlgorithm/Using_anonymous_functions
--- FAIL: TestTemplate_ExecuteAlgorithm (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_interfaces (0.00s)
 --- FAIL: TestTemplate_ExecuteAlgorithm/Using_anonymous_functions (0.00s)
 template_test.go:47: Expected string ' world ' was not found on returned string
FAIL
exit status 1
FAIL

```

正如你在测试执行的输出中所读到的，错误是在*使用匿名函数*测试中抛出的，这正是我们所期望的。现在我们将按照以下方式编写实现：

```go
type AnonymousTemplate struct{} 

func (a *AnonymousTemplate) first() string { 
  return "hello" 
} 

func (a *AnonymousTemplate) third() string { 
  return "template" 
} 

func (a *AnonymousTemplate) ExecuteAlgorithm(f func() string) string { 
  return strings.Join([]string{a.first(), f(), a.third()}, " ") 
} 

```

实现与`Template`类型中的实现非常相似。然而，现在我们传递了一个名为`f`的函数，我们将在`Join`函数中使用它作为字符串数组中的第二项。由于`f`只是一个返回字符串的函数，我们需要做的唯一事情就是在正确的位置执行它（数组中的第二个位置）。

再次运行测试：

```go
go test -v .
=== RUN   TestTemplate_ExecuteAlgorithm
=== RUN   TestTemplate_ExecuteAlgorithm/Using_interfaces
=== RUN   TestTemplate_ExecuteAlgorithm/Using_anonymous_functions
--- PASS: TestTemplate_ExecuteAlgorithm (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_interfaces (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_anonymous_functions (0.00s)
PASS
ok

```

太棒了！现在我们知道两种实现模板设计模式的方法。

## 如何避免对接口进行修改

之前方法的问题是现在我们有两个模板需要维护，我们可能会重复代码。在我们使用的接口不能更改的情况下，我们该怎么办？我们的接口是`MessageRetriever`，但现在我们想要使用一个匿名函数。

好吧，你还记得适配器设计模式吗？我们只需要创建一个`Adapter`类型，它接受一个`func() string`类型，返回一个`MessageRetriever`接口的实现。我们将称这个类型为`TemplateAdapter`：

```go
type TemplateAdapter struct { 
  myFunc func() string 
} 

func (a *TemplateAdapter) Message() string { 
  return "" 
} 

func MessageRetrieverAdapter(f func() string) MessageRetriever { 
  return nil 
} 

```

正如你所看到的，`TemplateAdapter`类型有一个名为`myFunc`的字段，它的类型是`func() string`。我们还将适配器定义为私有，因为在没有在`myFunc`字段中定义函数的情况下，它不应该被使用。我们创建了一个名为`MessageRetrieverAdapter`的公共函数来实现这一点。我们的测试应该看起来是这样的：

```go
t.Run("Using anonymous functions adapted to an interface", func(t *testing.T){ 
  messageRetriever := MessageRetrieverAdapter(func() string { 
    return "world" 
  }) 

  if messageRetriever == nil { 
    t.Fatal("Can not continue with a nil MessageRetriever") 
  } 

  template := Template{} 
  res := template.ExecuteAlgorithm(messageRetriever) 

  expectedOrError(res, " world ", t) 
}) 

```

看看我们调用`MessageRetrieverAdapter`方法的语句。我们传递了一个匿名函数作为参数，定义为`func() string`。然后，我们重用了我们第一个测试中定义的`Template`类型来传递`messageRetriever`变量。最后，我们再次使用`expectedOrError`方法进行检查。看一下`MessageRetrieverAdapter`方法，它将返回一个具有 nil 值的函数。如果严格遵循测试驱动开发规则，我们必须先进行测试，实现完成之前测试不能通过。这就是为什么我们在`MessageRetrieverAdapter`函数中返回 nil。

所以，让我们运行测试：

```go
go test -v .
=== RUN   TestTemplate_ExecuteAlgorithm
=== RUN   TestTemplate_ExecuteAlgorithm/Using_interfaces
=== RUN   TestTemplate_ExecuteAlgorithm/Using_anonymous_functions
=== RUN   TestTemplate_ExecuteAlgorithm/Using_anonymous_functions_adapted_to_an_interface
--- FAIL: TestTemplate_ExecuteAlgorithm (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_interfaces (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_anonymous_functions (0.00s)
 --- FAIL: TestTemplate_ExecuteAlgorithm/Using_anonymous_functions_adapted_to_an_interface (0.00s)
 template_test.go:39: Can not continue with a nil MessageRetriever
FAIL
exit status 1
FAIL

```

代码的*第 39 行*测试失败，而且没有继续执行（再次取决于你如何编写你的代码，表示错误的行可能在其他地方）。我们停止测试执行，因为当我们调用`ExecuteAlgorithm`方法时，我们将需要一个有效的`MessageRetriever`接口。

对于我们的 Template 模式的适配器实现，我们将从`MessageRetrieverAdapter`方法开始：

```go
func MessageRetrieverAdapter(f func() string) MessageRetriever { 
  return &adapter{myFunc: f} 
} 

```

很容易，对吧？你可能会想知道如果我们为`f`参数传递`nil`值会发生什么。好吧，我们将通过调用`myFunc`函数来解决这个问题。

`adapter`类型通过这个实现完成了：

```go
type adapter struct { 
  myFunc func() string 
} 

func (a *adapter) Message() string { 
  if a.myFunc != nil { 
    return a.myFunc() 
  } 

  return "" 
} 

```

在调用`Message()`函数时，我们检查在调用之前`myFunc`函数中是否实际存储了东西。如果没有存储任何东西，我们返回一个空字符串。

现在，我们使用适配器模式实现的`Template`类型的第三个实现已经完成：

```go
go test -v .
=== RUN   TestTemplate_ExecuteAlgorithm
=== RUN   TestTemplate_ExecuteAlgorithm/Using_interfaces
=== RUN   TestTemplate_ExecuteAlgorithm/Using_anonymous_functions
=== RUN   TestTemplate_ExecuteAlgorithm/Using_anonymous_functions_adapted_to_an_interface
--- PASS: TestTemplate_ExecuteAlgorithm (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_interfaces (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_anonymous_functions (0.00s)
 --- PASS: TestTemplate_ExecuteAlgorithm/Using_anonymous_functions_adapted_to_an_interface (0.00s)
PASS
ok

```

## 在 Go 的源代码中寻找模板模式

在 Go 的源代码中，`Sort`包可以被认为是一个排序算法的模板实现。正如包本身所定义的那样，`Sort`包提供了对切片和用户定义集合进行排序的基本操作。

在这里，我们还可以找到一个很好的例子，说明为什么 Go 的作者不担心实现泛型。在其他语言中，对列表进行排序可能是泛型使用的最佳例子。Go 处理这个问题的方式也非常优雅-它通过接口来处理这个问题：

```go
type Interface interface { 
  Len() int 
  Less(i, j int) bool 
  Swap(i, j int) 
} 

```

这是需要使用`sort`包进行排序的列表的接口。用 Go 的作者的话来说：

> *"一个类型，通常是一个集合，满足 sort.Interface 可以通过这个包中的例程进行排序。这些方法要求集合的元素通过整数索引进行枚举。"*

换句话说，编写一个实现这个`Interface`的类型，以便`Sort`包可以用来对任何切片进行排序。排序算法是模板，我们必须定义如何检索我们切片中的值。

如果我们查看`sort`包，我们还可以找到一个使用排序模板的示例，但我们将创建我们自己的示例：

```go
package main 

import ( 
  "sort" 
  "fmt" 
) 

type MyList []int 

func (m MyList) Len() int { 
  return len(m) 
} 

func (m MyList) Swap(i, j int) { 
  m[i], m[j] = m[j], m[i] 
} 

func (m MyList) Less(i, j int) bool { 
  return m[i] < m[j] 
} 

```

首先，我们做了一个非常简单的存储`int`列表的类型。这可以是任何类型的列表，通常是某种结构的列表。然后，我们通过定义`Len`、`Swap`和`Less`方法来实现`sort.Interface`接口。

最后，`main`函数创建了一个`MyList`类型的无序数字列表：

```go
func main() { 
  var myList MyList = []int{6,4,2,8,1} 

  fmt.Println(myList) 
  sort.Sort(myList) 
  fmt.Println(myList) 
} 

```

我们打印我们创建的列表（无序），然后对其进行排序（`sort.Sort`方法实际上修改了我们的变量，而不是返回一个新列表，所以要小心！）。最后，我们再次打印结果列表。这个`main`方法的控制台输出如下：

```go
go run sort_example.go 
[6 4 2 8 1]
[1 2 4 6 8]

```

`sort.Sort`函数以透明的方式对我们的列表进行了排序。它有很多代码编写，并将`Len`、`Swap`和`Less`方法委托给一个接口，就像我们在我们的模板中委托给`MessageRetriever`接口一样。

## 总结模板设计模式

我们希望在这个模式上放很多重点，因为在开发库和框架时非常重要，并且允许我们的库的用户有很大的灵活性和控制。

我们还看到了混合模式是非常常见的，以提供灵活性给用户，不仅在行为上，而且在结构上。当我们需要限制对我们代码的某些部分的访问以避免竞争时，这将非常有用。

# 备忘录设计模式

现在让我们看一个有着花哨名字的模式。如果我们查字典查看*memento*的含义，我们会找到以下描述：

> *"作为对某人或事件的提醒而保留的对象。"*

在这里，关键词是**reminder**，因为我们将用这种设计模式来记住动作。

## 描述

备忘录的含义与它在设计模式中提供的功能非常相似。基本上，我们将有一个带有一些状态的类型，我们希望能够保存其状态的里程碑。保存有限数量的状态后，我们可以在需要时恢复它们，以执行各种任务-撤销操作、历史记录等。

备忘录设计模式通常有三个参与者（通常称为**演员**）：

+   **备忘录**：存储我们想要保存的类型的类型。通常，我们不会直接存储业务类型，而是通过这种类型提供额外的抽象层。

+   **发起者**：负责创建备忘录并存储当前活动状态的类型。我们说备忘录类型包装了业务类型的状态，我们使用发起者作为备忘录的创建者。

+   **Care Taker**：一种存储备忘录列表的类型，可以具有将它们存储在数据库中或不存储超过指定数量的逻辑。

## 目标

备忘录完全是关于随时间的一系列操作，比如撤消一两个操作或为某个应用程序提供某种事务性。

备忘录为许多任务提供了基础，但其主要目标可以定义如下：

+   捕获对象状态而不修改对象本身

+   保存有限数量的状态，以便以后可以检索它们。

## 一个简单的字符串示例

我们将使用字符串作为要保存的状态的简单示例。这样，我们将在使其变得更加复杂之前，专注于常见的备忘录模式实现。

存储在`State`实例的字段中的字符串将被修改，我们将能够撤消在此状态下进行的操作。

## 需求和验收标准

我们不断谈论状态；总之，备忘录模式是关于存储和检索状态的。我们的验收标准必须完全关于状态：

1.  我们需要存储有限数量的字符串类型的状态。

1.  我们需要一种方法来将当前存储的状态恢复到状态列表中的一个。

有了这两个简单的要求，我们就可以开始为这个示例编写一些测试了。

## 单元测试

如前所述，备忘录设计模式通常由三个角色组成：状态、备忘录和 originator。因此，我们将需要三种类型来表示这些角色：

```go
type State struct { 
  Description string 
} 

```

`State`类型是我们在本例中将使用的核心业务对象。它是我们想要跟踪的任何类型的对象：

```go
type memento struct { 
  state State 
} 

```

`memento`类型有一个名为`state`的字段，表示`State`类型的单个值。在将它们存储到`care taker`类型之前，我们的`states`将被封装在这种类型中。您可能会想知道为什么我们不直接存储`State`实例。基本上，因为这将使`originator`和`careTaker`与业务对象耦合在一起，而我们希望尽可能少地耦合。这也将更不灵活，正如我们将在第二个例子中看到的：

```go
type originator struct { 
  state State 
} 

func (o *originator) NewMemento() memento { 
  return memento{} 
} 

func (o *originator) ExtractAndStoreState(m memento) { 
  //Does nothing 
} 

```

`originator`类型还存储状态。`originator`结构的对象将从备忘录中获取状态，并使用其存储的状态创建新的备忘录。

### 提示

原始对象和备忘录模式之间有什么区别？为什么我们不直接使用 Originator 模式的对象？嗯，如果 Memento 包含特定状态，`originator`类型包含当前加载的状态。此外，保存某物的状态可能像获取某个值那样简单，也可能像维护某个分布式应用程序的状态那样复杂。

Originator 将有两个公共方法--`NewMemento()`方法和`ExtractAndStoreState(m memento)`方法。`NewMemento`方法将返回一个使用`originator`当前`State`值构建的新 Memento。`ExtractAndStoreState`方法将获取 Memento 的状态并将其存储在`Originator`的状态字段中：

```go
type careTaker struct { 
  mementoList []memento 
} 

func (c *careTaker) Add(m memento) { 
  //Does nothing 
} 

func (c *careTaker) Memento(i int) (memento, error) { 
  return memento{}, fmt.Errorf("Not implemented yet") 
} 

```

`careTaker`类型存储了我们需要保存的所有状态的 Memento 列表。它还存储了一个`Add`方法，用于在列表中插入新的 Memento，以及一个 Memento 检索器，它在 Memento 列表上获取一个索引。

所以让我们从`careTaker`类型的`Add`方法开始。`Add`方法必须获取一个`memento`对象，并将其添加到`careTaker`对象的 Mementos 列表中：

```go
func TestCareTaker_Add(t *testing.T) { 
  originator := originator{} 
  originator.state = State{Description:"Idle"} 

  careTaker := careTaker{} 
  mem := originator.NewMemento() 
  if mem.state.Description != "Idle" { 
    t.Error("Expected state was not found") 
  } 

```

在我们的测试开始时，我们为备忘录创建了两个基本的角色--`originator`和`careTaker`。我们在 originator 上设置了第一个状态，描述为`Idle`。

然后，我们调用`NewMemento`方法创建第一个 Memento。这应该将当前 originator 的状态封装在`memento`类型中。我们的第一个检查非常简单--返回的 Memento 的状态描述必须与我们传递给 originator 的状态描述相同，即`Idle`描述。

检查我们的 Memento 的`Add`方法是否正确工作的最后一步是看看在添加一个项目后 Memento 列表是否增长：

```go
  currentLen := len(careTaker.mementoList) 
  careTaker.Add(mem) 

  if len(careTaker.mementoList) != currentLen+1 { 
    t.Error("No new elements were added on the list") 
  } 

```

我们还必须测试`Memento(int) memento`方法。这应该从`careTaker`列表中获取一个`memento`值。它获取你想要从列表中检索的索引，所以像通常一样，我们必须检查它对负数和超出索引值的行为是否正确：

```go
func TestCareTaker_Memento(t *testing.T) { 
  originator := originator{} 
  careTaker := careTaker{} 

  originator.state = State{"Idle"} 
  careTaker.Add(originator.NewMemento()) 

```

我们必须像在之前的测试中一样开始--创建`originator`和`careTaker`对象，并将第一个 Memento 添加到`caretaker`中：

```go
  mem, err := careTaker.Memento(0) 
  if err != nil { 
    t.Fatal(err) 
  } 

  if mem.state.Description != "Idle" { 
    t.Error("Unexpected state") 
  } 

```

一旦我们在`careTaker`对象上有了第一个对象，我们就可以使用`careTaker.Memento(0)`来请求它。`Memento(int)`方法中的索引`0`检索切片上的第一个项目（记住切片从`0`开始）。不应该返回错误，因为我们已经向`caretaker`对象添加了一个值。

然后，在检索第一个 memento 之后，我们检查描述是否与测试开始时传递的描述匹配：

```go
  mem, err = careTaker.Memento(-1) 
  if err == nil { 
    t.Fatal("An error is expected when asking for a negative number but no error was found") 
  } 
} 

```

这个测试的最后一步涉及使用负数来检索一些值。在这种情况下，必须返回一个错误，显示不能使用负数。当传递负数时，也可能返回第一个索引，但在这里我们将返回一个错误。

要检查的最后一个函数是`ExtractAndStoreState`方法。这个函数必须获取一个 Memento 并提取它的所有状态信息，然后将其设置在`Originator`对象中：

```go
func TestOriginator_ExtractAndStoreState(t *testing.T) { 
  originator := originator{state:State{"Idle"}} 
  idleMemento := originator.NewMemento() 

  originator.ExtractAndStoreState(idleMemento) 
  if originator.state.Description != "Idle" { 
    t.Error("Unexpected state found") 
  } 
} 

```

这个测试很简单。我们用一个`Idle`状态创建一个默认的`originator`变量。然后，我们检索一个新的 Memento 对象以便以后使用。我们将`originator`变量的状态更改为`Working`状态，以确保新状态将被写入。

最后，我们必须使用`idleMemento`变量调用`ExtractAndStoreState`方法。这应该将原始对象的状态恢复到`idleMemento`状态的值，这是我们在最后的`if`语句中检查的。

现在是运行测试的时候了：

```go
go test -v . 
=== RUN   TestCareTaker_Add
--- FAIL: TestCareTaker_Add (0.00s)
 memento_test.go:13: Expected state was not found
 memento_test.go:20: No new elements were added on the list
=== RUN   TestCareTaker_Memento
--- FAIL: TestCareTaker_Memento (0.00s)
 memento_test.go:33: Not implemented yet
=== RUN   TestOriginator_ExtractAndStoreState
--- FAIL: TestOriginator_ExtractAndStoreState (0.00s)
 memento_test.go:54: Unexpected state found
FAIL
exit status 1
FAIL

```

因为三个测试失败，我们可以继续实现。

## 实现备忘录模式

如果你不要太疯狂，实现备忘录模式通常非常简单。模式中的三个角色（`memento`，`originator`和`care taker`）在模式中有非常明确定义的角色，它们的实现非常直接：

```go
type originator struct { 
  state State 
} 

func (o *originator) NewMemento() memento { 
  return memento{state: o.state} 
} 

func (o *originator) ExtractAndStoreState(m memento) { 
  o.state = m.state 
} 

```

当调用`NewMemento`方法时，`Originator`对象需要返回 Memento 类型的新值。它还需要根据`ExtractAndStoreState`方法的需要将`memento`对象的值存储在结构的状态字段中。

```go
type careTaker struct { 
  mementoList []memento 
} 

func (c *careTaker) Push(m memento) { 
  c.mementoList = append(c.mementoList, m) 
} 

func (c *careTaker) Memento(i int) (memento, error) { 
  if len(c.mementoList) < i || i < 0 { 
    return memento{}, fmt.Errorf("Index not found\n") 
  } 
  return c.mementoList[i], nil 
} 

```

`careTaker`类型也很简单。当我们调用`Add`方法时，通过调用带有传入参数的`append`方法来覆盖`mementoList`字段。这将创建一个包含新值的新列表。

在调用`Memento`方法时，我们必须先进行一些检查。在这种情况下，我们检查索引是否不超出切片的范围，并且在`if`语句中检查索引是否不是负数，如果是，则返回错误。如果一切顺利，它只返回指定的`memento`对象，没有错误。

### 提示

关于方法和函数命名约定的一点说明。你可能会发现一些人喜欢给方法取名字更具描述性，比如`Memento`。一个例子是使用一个名为`MementoOrError`的方法，清楚地显示在调用此函数时返回两个对象，甚至是`GetMementoOrError`方法。这可能是一种非常明确的命名方法，不一定是坏事，但你在 Go 的源代码中不会经常看到这种情况。

检查测试结果的时间到了：

```go
go test -v .
=== RUN   TestCareTaker_Add
--- PASS: TestCareTaker_Add (0.00s)
=== RUN   TestCareTaker_Memento
--- PASS: TestCareTaker_Memento (0.00s)
=== RUN   TestOriginator_ExtractAndStoreState
--- PASS: TestOriginator_ExtractAndStoreState (0.00s)
PASS
ok

```

这足以达到 100%的覆盖率。虽然这远非完美的度量标准，但至少我们知道我们正在触及源代码的每一个角落，而且我们没有在测试中作弊来实现它。

## 另一个使用命令和外观模式的例子

前面的例子足够好而简单，以理解 Memento 模式的功能。然而，它更常用于与命令模式和简单的 Facade 模式结合使用。

这个想法是使用命令模式来封装一组不同类型的状态（实现`Command`接口的那些状态），并提供一个小的外观来自动将其插入`caretaker`对象中。

我们将开发一个假想音频混音器的小例子。我们将使用相同的 Memento 模式来保存两种类型的状态：`Volume`和`Mute`。 `Volume`状态将是一个字节类型，`Mute`状态将是一个布尔类型。我们将使用两种完全不同的类型来展示这种方法的灵活性（以及它的缺点）。

顺便说一句，我们还可以在接口上为每个`Command`接口提供自己的序列化方法。这样，我们可以让 caretaker 在不真正知道正在存储什么的情况下将状态存储在某种存储中。

我们的`Command`接口将有一个方法来返回其实现者的值。这很简单，我们想要撤消的音频混音器中的每个命令都必须实现这个接口：

```go
type Command interface { 
  GetValue() interface{} 
} 

```

这个接口中有一些有趣的地方。`GetValue`方法返回一个值的接口。这也意味着这个方法的返回类型是...嗯...无类型的？不是真的，但它返回一个接口，可以是任何类型的表示，我们稍后需要对其进行类型转换，如果我们想使用其特定类型。现在我们必须定义`Volume`和`Mute`类型并实现`Command`接口：

```go
type Volume byte 

func (v Volume) GetValue() interface{} { 
  return v 
} 

type Mute bool 

func (m Mute) GetValue() interface{} { 
  return m 
} 

```

它们都是相当简单的实现。但是，`Mute`类型将在`GetValue()`方法上返回一个`bool`类型，`Volume`将返回一个`byte`类型。

与之前的例子一样，我们需要一个将保存`Command`的`Memento`类型。换句话说，它将存储指向`Mute`或`Volume`类型的指针：

```go
type Memento struct { 
  memento Command 
} 

```

`originator`类型的工作方式与之前的例子相同，但是使用`Command`关键字而不是`state`关键字：

```go
type originator struct { 
  Command Command 
} 

func (o *originator) NewMemento() Memento { 
  return Memento{memento: o.Command} 
} 

func (o *originator) ExtractAndStoreCommand(m Memento) { 
  o.Command = m.memento 
} 

```

而`caretaker`对象几乎相同，但这次我们将使用堆栈而不是简单列表，并且我们将存储命令而不是状态：

```go
type careTaker struct { 
  mementoList []Memento 
} 

func (c *careTaker) Add(m Memento) { 
  c.mementoList = append(c.mementoList, m) 
} 

func (c *careTaker) Pop() Memento { 
  if len(c.mementoStack) > 0 { 
    tempMemento := c.mementoStack[len(c.mementoStack)-1] 
    c.mementoStack = c.mementoStack[0:len(c.mementoStack)-1] 
    return tempMemento 
  } 

  return Memento{} 
} 

```

然而，我们的`Memento`列表被替换为`Pop`方法。它也返回一个`memento`对象，但它将它们作为堆栈返回（最后进入，最先出）。因此，我们取堆栈上的最后一个元素并将其存储在`tempMemento`变量中。然后，我们用不包含下一行上的最后一个元素的新版本替换堆栈。最后，我们返回`tempMemento`变量。

到目前为止，一切看起来几乎与之前的例子一样。我们还谈到了通过使用 Facade 模式自动化一些任务，所以让我们来做吧。这将被称为`MementoFacade`类型，并将具有`SaveSettings`和`RestoreSettings`方法。`SaveSettings`方法接受一个`Command`，将其存储在内部 originator 中，并保存在内部的`careTaker`字段中。`RestoreSettings`方法执行相反的流程-恢复`careTaker`的索引并返回`Memento`对象中的`Command`：

```go
type MementoFacade struct { 
  originator originator 
  careTaker  careTaker 
} 

func (m *MementoFacade) SaveSettings(s Command) { 
  m.originator.Command = s 
  m.careTaker.Add(m.originator.NewMemento()) 
} 

func (m *MementoFacade) RestoreSettings(i int) Command { 
  m.originator.ExtractAndStoreCommand(m.careTaker.Memento(i)) 
  return m.originator.Command 
} 

```

我们的 Facade 模式将保存 originator 和 care taker 的内容，并提供这两个易于使用的方法来保存和恢复设置。

那么，我们如何使用这个？

```go
func main(){ 
  m := MementoFacade{} 

  m.SaveSettings(Volume(4)) 
  m.SaveSettings(Mute(false)) 

```

首先，我们使用 Facade 模式获取一个变量。零值初始化将给我们零值的`originator`和`caretaker`对象。它们没有任何意外的字段，因此一切都将正确初始化（例如，如果它们中的任何一个有一个指针，它将被初始化为`nil`，如第一章中提到的*零初始化*部分，*准备...开始...跑！*）。

我们使用`Volume(4)`创建了一个`Volume`值，是的，我们使用了括号。`Volume`类型没有像结构体那样的内部字段，因此我们不能使用大括号来设置其值。设置它的方法是使用括号（或者创建指向`Volume`类型的指针，然后设置指向空间的值）。我们还使用外观模式保存了一个`Mute`类型的值。

我们不知道这里返回了什么类型的`Command`，因此我们需要进行类型断言。我们将编写一个小函数来帮助我们进行检查类型并打印适当的值：

```go
func assertAndPrint(c Command){ 
  switch cast := c.(type) { 
  case Volume: 
    fmt.Printf("Volume:\t%d\n", cast) 
  case Mute: 
    fmt.Printf("Mute:\t%t\n", cast) 
  } 
} 

```

`assertAndPrint`方法接受一个`Command`类型并将其转换为两种可能的类型-`Volume`或`Mute`。在每种情况下，它都会向控制台打印一条消息。现在我们可以继续并完成`main`函数，它将如下所示：

```go
func main() { 
  m := MementoFacade{} 

  m.SaveSettings(Volume(4)) 
  m.SaveSettings(Mute(false)) 

 assertAndPrint(m.RestoreSettings(0))
 assertAndPrint(m.RestoreSettings(1)) 
} 

```

粗体部分显示了`main`函数中的新更改。我们从`careTaker`对象中取出索引 0 并将其传递给新函数，索引`1`也是一样。运行这个小程序，我们应该在控制台上得到`Volume`和`Mute`的值：

```go
$ go run memento_command.go
Mute:   false
Volume: 4

```

太好了！在这个小例子中，我们结合了三种不同的设计模式，以便继续舒适地使用各种模式。请记住，我们也可以将`Volume`和`Mute`状态的创建抽象为工厂模式，所以这并不是我们停下来的地方。

## 关于备忘录模式的最后一句话

通过备忘录模式，我们学会了一种强大的方式来创建可撤销的操作，这在编写 UI 应用程序时非常有用，也在开发事务操作时非常有用。无论如何，情况都是一样的：您需要一个`Memento`，一个`Originator`和一个`caretaker`角色。

### 提示

**事务操作**是一组必须全部完成或失败的原子操作。换句话说，如果您有一个由五个操作组成的事务，只有其中一个失败，事务就无法完成，其他四个操作所做的每个修改都必须被撤消。

# 解释器设计模式

现在我们将深入研究一个相当复杂的模式。事实上，**解释器**模式被广泛用于解决业务案例，其中有一个语言执行常见操作的需求。让我们看看我们所说的语言是什么意思。

## 描述

我们可以谈论的最著名的解释器可能是 SQL。它被定义为用于管理关系数据库中保存的数据的特定编程语言。SQL 非常复杂和庞大，但总的来说，它是一组单词和操作符，允许我们执行插入、选择或删除等操作。

另一个典型的例子是音乐符号。它本身就是一种语言，解释器是懂得音符与乐器上的表示之间连接的音乐家。

在计算机科学中，为各种原因设计一个小语言可能是有用的：重复的任务，非开发人员的高级语言，或者**接口定义语言**（**IDL**）如**Protocol buffers**或**Apache Thrift**。

## 目标

设计一个新的语言，无论大小，都可能是一项耗时的任务，因此在投入时间和资源编写其解释器之前，明确目标非常重要：

+   提供一些范围内非常常见操作的语法（比如播放音符）。

+   拥有一个中间语言来在两个系统之间转换操作。例如，生成 3D 打印所需的**Gcode**的应用程序。

+   简化某些操作的使用，使用更易于使用的语法。

SQL 允许使用关系数据库的非常易于使用的语法（也可能变得非常复杂），但其思想是不需要编写自己的函数来进行插入和搜索。

## 例如 - 逆波兰表达式计算器

解释器的一个非常典型的例子是创建一个逆波兰表示法计算器。对于那些不知道波兰表示法是什么的人来说，它是一种数学表示法，用于进行操作，其中你首先写下你的操作（求和），然后是值（3 4），因此*+ 3 4*等同于更常见的*3 + 4*，其结果将是*7*。因此，对于逆波兰表示法，你首先放置值，然后是操作，因此*3 4 +*也将是*7*。

## 计算器的验收标准

对于我们的计算器，我们应该通过的验收标准如下：

1.  创建一种允许进行常见算术运算（求和、减法、乘法和除法）的语言。语法是`sum`表示求和，`mul`表示乘法，`sub`表示减法，`div`表示除法。

1.  必须使用逆波兰表示法。

1.  用户必须能够连续写入任意多个操作。

1.  操作必须从左到右执行。

因此，`3 4 sum 2 sub`表示的是*(3 + 4) - 2*，结果将是*5*。

## 一些操作的单元测试

在这种情况下，我们只有一个名为`Calculate`的公共方法，它接受一个操作及其值定义为字符串，并将返回一个值或一个错误：

```go
func Calculate(o string) (int, error) { 
  return 0, fmt.Errorf("Not implemented yet") 
} 

```

因此，我们将发送一个字符串，如`"3 4 +"`到`Calculate`方法，它应该返回*7, nil*。另外两个测试将检查正确的实现：

```go
func TestCalculate(t *testing.T) { 
  tempOperation = "3 4 sum 2 sub" 
  res, err = Calculate(tempOperation) 
  if err != nil { 
    t.Error(err) 
  } 

  if res != 5 { 
    t.Errorf("Expected result not found: %d != %d\n", 5, res) 
  } 

```

首先，我们将进行我们作为示例使用的操作。`3 4 sum 2 sub`表示是我们语言的一部分，我们在`Calculate`函数中使用它。如果返回错误，则测试失败。最后，结果必须等于`5`，我们在最后几行进行检查。下一个测试检查了稍微复杂一些的操作的其余操作符：

```go
  tempOperation := "5 3 sub 8 mul 4 sum 5 div" 
  res, err := Calculate(tempOperation) 
  if err != nil { 
    t.Error(err) 
  } 

  if res != 4 { 
    t.Errorf("Expected result not found: %d != %d\n", 4, res) 
  } 
} 

```

在这里，我们使用了一个更长的操作重复了前面的过程，即*(((5 - 3) * 8) + 4) / 5*，它等于*4*。从左到右，它将如下进行：

```go
(((5 - 3) * 8) + 4) / 5
 ((2 * 8) + 4) / 5
 (16 + 4) / 5
 20 / 5
 4

```

当然，测试必须失败！

```go
$ go test -v .
 interpreter_test.go:9: Not implemented yet
 interpreter_test.go:13: Expected result not found: 4 != 0
 interpreter_test.go:19: Not implemented yet
 interpreter_test.go:23: Expected result not found: 5 != 0
exit status 1
FAIL

```

## 实施

这次实现的时间要比测试长。首先，我们将在常量中定义可能的运算符：

```go
const ( 
  SUM = "sum" 
  SUB = "sub" 
  MUL = "mul" 
  DIV = "div" 
) 

```

解释器模式通常使用抽象语法树来实现，这通常使用堆栈来实现。我们在本书中之前已经创建了堆栈，所以这对读者来说应该已经很熟悉了：

```go
type polishNotationStack []int 

func (p *polishNotationStack) Push(s int) { 
  *p = append(*p, s) 
} 

func (p *polishNotationStack) Pop() int { 
  length := len(*p) 

  if length > 0 { 
    temp := (*p)[length-1] 
    *p = (*p)[:length-1] 
    return temp 
  } 

  return 0 
} 

```

我们有两种方法——`Push`方法用于将元素添加到堆栈顶部，`Pop`方法用于移除元素并返回它们。如果你认为`*p = (*p)[:length-1]`这一行有点神秘，我们会解释它。

存储在`p`方向的值将被覆盖为`p`方向的实际值（*p*），但只取数组的开始到倒数第二个元素(:length-1)。

因此，现在我们将逐步进行`Calculate`函数，根据需要创建更多的函数：

```go
func Calculate(o string) (int, error) { 
  stack := polishNotationStack{} 
  operators := strings.Split(o, " ") 

```

我们需要做的前两件事是创建堆栈并从传入的操作中获取所有不同的符号（在这种情况下，我们没有检查它是否为空）。我们通过空格拆分传入的字符串操作，以获得一个漂亮的符号（值和运算符）切片。

接下来，我们将使用 range 迭代每个符号，但我们需要一个函数来知道传入的符号是值还是运算符：

```go
func isOperator(o string) bool { 
  if o == SUM || o == SUB || o == MUL || o == DIV { 
    return true 
  } 

  return false 
} 

```

如果传入的符号是我们常量中定义的任何一个，那么传入的符号就是一个运算符。

```go
func Calculate(o string) (int, error) { 
  stack := polishNotationStack{} 
  operators := strings.Split(o, " ") 

for _, operatorString := range operators {
 if isOperator(operatorString) {
 right := stack.Pop()
 left := stack.Pop()
 } 
  else 
  {
 //Is a value
 } 
}

```

如果它是一个运算符，我们认为我们已经传递了两个值，所以我们要做的就是从堆栈中取出这两个值。取出的第一个值将是最右边的，第二个值将是最左边的（请记住，在减法和除法中，操作数的顺序很重要）。然后，我们需要一些函数来获取我们想要执行的操作：

```go
func getOperationFunc(o string) func(a, b int) int { 
  switch o { 
  case SUM: 
    return func(a, b int) int { 
      return a + b 
    } 
  case SUB: 
    return func(a, b int) int { 
      return a - b 
    } 
  case MUL: 
    return func(a, b int) int { 
      return a * b 
    } 
  case DIV: 
    return func(a, b int) int { 
      return a / b 
    } 
  } 
  return nil 
} 

```

`getOperationFunc`函数返回一个返回整数的两参数函数。我们检查传入的运算符，然后返回执行指定操作的匿名函数。因此，现在我们的`for range`继续如下：

```go
func Calculate(o string) (int, error) { 
  stack := polishNotationStack{} 
  operators := strings.Split(o, " ") 

for _, operatorString := range operators { 
  if isOperator(operatorString) { 
      right := stack.Pop() 
      left := stack.Pop() 
 mathFunc := getOperationFunc(operatorString)
 res := mathFunc(left, right)
 stack.Push(res) 
    } else { 
      //Is a value 
    } 
} 

```

`mathFunc`变量由该函数返回。我们立即使用它对从堆栈中取出的左值和右值执行操作，并将其结果存储在一个名为`res`的新变量中。最后，我们需要将这个新值推送到堆栈中，以便稍后继续操作。

现在，当传入的符号是一个值时，这是实现：

```go
func Calculate(o string) (int, error) { 
  stack := polishNotationStack{} 
  operators := strings.Split(o, " ") 

for _, operatorString := range operators { 
    if isOperator(operatorString) { 
      right := stack.Pop() 
      left := stack.Pop() 
      mathFunc := getOperationFunc(operatorString) 
      res := mathFunc(left, right) 
      stack.Push(res) 
    } else { 
 val, err := strconv.Atoi(operatorString)
 if err != nil {
 return 0, err
 }
 stack.Push(val) 
    } 
  } 

```

每次我们得到一个符号时，我们需要将其推送到堆栈中。我们必须将字符串符号解析为可用的`int`类型。这通常使用`strconv`包来完成，通过使用其`Atoi`函数。`Atoi`函数接受一个字符串并返回一个整数或一个错误。如果一切顺利，该值将被推送到堆栈中。

在`range`语句结束时，只需存储一个值，所以我们只需要返回它，函数就完成了。

```go
func Calculate(o string) (int, error) { 
  stack := polishNotationStack{} 
  operators := strings.Split(o, " ") 

for _, operatorString := range operators { 
    if isOperator(operatorString) { 
      right := stack.Pop() 
      left := stack.Pop() 
      mathFunc := getOperationFunc(operatorString) 
      res := mathFunc(left, right) 
      stack.Push(res) 
    } else { 
      val, err := strconv.Atoi(operatorString) 
      if err != nil { 
        return 0, err 
      } 

      stack.Push(val) 
    } 
  } 
 return int(stack.Pop()), nil
}

```

是时候再次运行测试了：

```go
$ go test -v .
ok

```

太棒了！我们刚刚以一种非常简单和容易的方式创建了一个逆波兰表示法解释器（我们仍然缺少解析器，但那是另一回事）。

## 解释器设计模式的复杂性

在这个例子中，我们没有使用任何接口。这并不完全符合更面向对象的语言中定义的解释器设计模式。然而，这个例子是最简单的例子，可以理解语言的目标，下一个级别不可避免地会更复杂，不适合初学者用户。

在更复杂的例子中，我们将不得不定义一个包含更多类型的类型，一个值或无类型。通过解析器，您可以创建这个抽象语法树以后进行解释。

使用接口完成相同的示例，如下所述。

## 再次使用接口的解释器模式

我们将使用的主要接口称为`Interpreter`接口。该接口具有一个`Read()`方法，每个符号（值或运算符）都必须实现：

```go
type Interpreter interface { 
  Read() int 
} 

```

我们将仅实现运算符的求和和减法，以及称为`Value`的数字类型：

```go
type value int 

func (v *value) Read() int { 
  return int(*v) 
} 

```

`Value`是一个`int`类型，当实现`Read`方法时，只返回其值：

```go
type operationSum struct { 
  Left  Interpreter 
  Right Interpreter 
} 

func (a *operationSum) Read() int { 
  return a.Left.Read() + a.Right.Read() 
} 

```

`operationSum`结构具有`Left`和`Right`字段，其`Read`方法返回其各自的`Read`方法的和。`operationSubtract`结构也是一样，但是减去：

```go
type operationSubtract struct { 
  Left  Interpreter 
  Right Interpreter 
} 

func (s *operationSubtract) Read() int { 
  return s.Left.Read() - s.Right.Read() 
} 

```

我们还需要一个工厂模式来创建运算符；我们将称之为`operatorFactory`方法。现在的区别在于它不仅接受符号，还接受从堆栈中取出的`Left`和`Right`值：

```go
func operatorFactory(o string, left, right Interpreter) Interpreter { 
  switch o { 
  case SUM: 
    return &operationSum{ 
      Left: left, 
      Right: right, 
    } 
  case SUB: 
    return &operationSubtract{ 
      Left: left, 
      Right: right, 
    } 
  } 

  return nil 
} 

```

正如我们刚才提到的，我们还需要一个堆栈。我们可以通过更改其类型来重用前面示例中的堆栈：

```go
type polishNotationStack []Interpreter 

func (p *polishNotationStack) Push(s Interpreter) { 
  *p = append(*p, s) 
} 

func (p *polishNotationStack) Pop() Interpreter { 
  length := len(*p) 

  if length > 0 { 
    temp := (*p)[length-1] 
    *p = (*p)[:length-1] 
    return temp 
  } 

  return nil 
} 

```

现在堆栈使用解释器指针而不是`int`，但其功能是相同的。最后，我们的`main`方法看起来也与之前的示例类似：

```go
func main() { 
  stack := polishNotationStack{} 
  operators := strings.Split("3 4 sum 2 sub", " ") 

  for _, operatorString := range operators { 
    if operatorString == SUM || operatorString == SUB { 
      right := stack.Pop() 
      left := stack.Pop() 
      mathFunc := operatorFactory(operatorString, left, right) 
      res := value(mathFunc.Read()) 
      stack.Push(&res) 
    } else { 
      val, err := strconv.Atoi(operatorString) 
      if err != nil { 
        panic(err) 
      } 

      temp := value(val) 
      stack.Push(&temp) 
    } 
  } 

  println(int(stack.Pop().Read())) 
} 

```

与以前一样，我们首先检查符号是运算符还是值。当它是一个值时，它将其推送到堆栈中。

当符号是一个运算符时，我们还从堆栈中取出右值和左值，我们使用当前运算符和刚刚从堆栈中取出的左值和右值调用工厂模式。一旦我们有了运算符类型，我们只需要调用其`Read`方法将返回的值也推送到堆栈中。

最后，只剩下一个例子必须留在堆栈上，所以我们打印它：

```go
$ go run interpreter.go
5

```

## 解释器模式的力量

这种模式非常强大，但也必须谨慎使用。创建一种语言会在其用户和提供的功能之间产生强大的耦合。人们可能会犯错误，试图创建一种过于灵活、难以使用和维护的语言。另外，人们可能会创建一种相当小而有用的语言，有时解释不正确，这可能会给用户带来痛苦。

在我们的例子中，我们省略了相当多的错误检查，以便专注于解释器的实现。然而，你需要相当多的错误检查和错误时冗长的输出，以帮助用户纠正其语法错误。所以，写你的语言时要开心，但要对用户友好。

# 总结

本章介绍了三种非常强大的模式，在将它们用于生产代码之前，需要进行大量的练习。通过模拟典型的生产问题来进行一些练习是一个非常好的主意：

+   创建一个简单的 REST 服务器，重用大部分错误检查和连接功能，提供一个易于使用的接口来练习模板模式

+   制作一个小型库，可以写入不同的数据库，但前提是所有写入都没有问题，或者删除新创建的写入以练习备忘录，例如

+   编写你自己的语言，做一些简单的事情，比如回答简单的问题，就像机器人通常做的那样，这样你就可以练习一下解释器模式

这个想法是练习编码并重新阅读任何部分，直到你对每个模式都感到舒适。
