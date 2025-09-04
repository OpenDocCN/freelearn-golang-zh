# 第六章。行为模式 - 模板、备忘录和解释器设计模式

在本章中，我们将看到接下来的三个行为设计模式。难度正在提高，因为我们现在将使用结构和创建模式的组合来更好地解决某些行为模式的目标。

我们将从模板设计模式开始，这个模式看起来与策略模式非常相似，但提供了更大的灵活性。备忘录设计模式被我们每天使用的 99%的应用程序用于实现撤销功能和事务操作。最后，我们将编写一个逆波兰表示法解释器来执行简单的数学运算。

让我们从模板设计模式开始。

# 模板设计模式

**模板**模式是那些广泛使用且非常有用的模式之一，尤其是在编写库和框架时。想法是提供一种方式，让用户在算法中执行代码。

在本节中，我们将了解如何编写地道的 Go 模板模式，并查看一些明智使用 Go 源代码的例子。我们将编写一个三步算法，其中第二步委托给用户，而第一步和第三步则不是。算法的第一步和第三步代表模板。

## 描述

当我们使用策略模式封装算法实现时，我们将尝试通过模板模式实现类似的效果，但只是算法的一部分。

模板设计模式允许用户编写算法的一部分，而其余部分由抽象执行。这在创建库以简化某些复杂任务或当算法的复用性仅由其中一部分构成时很常见。

例如，想象一下我们有一个长串的 HTTP 请求事务。我们必须执行以下步骤：

1.  验证用户。

1.  授权他。

1.  从数据库中检索一些详细信息。

1.  进行一些修改。

1.  在新的请求中发送这些详细信息。

在用户代码中每次需要修改数据库上的内容时重复步骤 1 到 5 是没有意义的。相反，步骤 1、2、3 和 5 将在接收带有第五步所需完成事务的接口的同一算法中抽象化。它也不需要是一个接口，它也可以是一个回调。

## 目标

模板设计模式全部关于复用性和赋予用户责任。因此，这个模式的目的是以下：

+   将库中算法的一部分委托给用户

+   通过抽象代码中不常见的部分来提高复用性

## 示例 - 具有延迟步骤的简单算法

在我们的第一个例子中，我们将编写一个由三步组成的算法，每一步都返回一个消息。第一步和第三步由模板控制，只有第二步被委托给用户。

## 需求和验收标准

模板模式的简要描述是定义一个算法三步模板，将第二步的实现推迟到用户：

1.  算法中的每一步都必须返回一个字符串。

1.  第一步是一个名为`first()`的方法，返回字符串`hello`。

1.  第三步是一个名为`third()`的方法，返回字符串`template`。

1.  第二步是用户想要返回的任何字符串，但它由具有`Message() string`方法的`MessageRetriever`接口定义。

1.  算法通过一个名为`ExecuteAlgorithm`的方法按顺序执行，并返回每个步骤返回的字符串，这些字符串通过空格连接成一个字符串。

## 简单算法的单元测试

我们将只关注测试公共方法。这是一个非常常见的方法。总的来说，如果你的私有方法没有被公共方法中的某个级别调用，那么它们根本就不会被调用。这里我们需要两个接口，一个用于模板实现者，一个用于算法的抽象步骤：

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

模板实现者将接受一个`MessageRetriever`接口作为其执行算法的一部分。我们需要一个实现此接口的类型，我们将其称为`Template`，我们将称之为`TemplateImpl`：

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

因此，我们的第一个测试检查第四和第五个验收标准。我们将创建一个实现`MessageRetriever`接口并返回字符串`world`的`TestStruct`类型，并嵌入模板，以便它可以调用`ExecuteAlgorithm`方法。它将充当模板和抽象：

```go
type TestStruct struct { 
  Template 
} 

func (m *TestStruct) Message() string { 
  return "world" 
} 

```

首先，我们将定义`TestStruct`类型。在这种情况下，算法推迟给我们的一部分将返回`world`文本。这是我们将在测试中稍后查找的字符串，进行“这个字符串中是否包含单词`world`”的检查。

仔细观察，`TestStruct`嵌入了一个名为`Template`的类型，它代表我们算法的模板模式。

当我们实现`Message()`方法时，我们隐式地实现了`MessageRetriever`接口。因此，现在我们可以使用`TestStruct`类型作为`MessageRetriever`接口的指针：

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

在测试中，我们将使用我们刚刚创建的类型。当我们调用`ExecuteAlgorithm`方法时，我们需要传递`MessageRetriever`接口。由于`TestStruct`类型也实现了`MessageRetriever`接口，我们可以将其作为参数传递，但这不是强制性的。

根据第五个验收标准定义的`ExecuteAlgorithm`方法的结果必须返回一个包含`first()`方法返回值、`TestStruct`（`world`字符串）返回值和`third()`方法返回值的字符串，这些值由空格分隔。我们的实现位于第二个位置；这就是为什么我们检查字符串`world`前面和后面是否有空格。

因此，如果调用`ExecuteAlgorithm`方法返回的字符串不包含字符串`world`，则测试失败。

这样就足以使项目编译并通过应该失败的测试：

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

是时候将这个模式实现到我们的代码中了。

## 实现模板模式

根据验收标准定义，我们必须在 `first()` 方法中返回字符串 `hello`，在 `third()` 方法中返回字符串 `template`。这相当容易实现：

```go
type Template struct{} 

func (t *Template) first() string { 
  return "hello" 
} 

func (t *Template) third() string { 
  return "template" 
} 

```

使用这个实现，我们应该覆盖 *第二个* 和 *第三个* 验收标准，并部分覆盖 *第一个* 标准（算法的每个步骤都必须返回一个字符串）。

为了覆盖 *第五* 验收标准，我们定义了一个接受 `MessageRetriever` 接口作为参数的 `ExecuteAlgorithm` 方法，并返回完整的算法：一个由 `first()`、`Message()` 字符串和 `third()` 方法返回的字符串连接而成的单个字符串：

```go
func (t *Template) ExecuteAlgorithm(m MessageRetriever) string { 
  return strings.Join([]string{t.first(), m.Message(), t.third()},  " ") 
} 

```

`strings.Join` 函数具有以下签名：

```go
func Join([]string,string) string 

```

它接受一个字符串数组，并将它们连接起来，在数组的每个项目之间放置第二个参数。在我们的情况下，我们动态创建一个字符串数组，并将其作为第一个参数传递。然后我们传递一个空格作为第二个参数。

使用这个实现，测试必须已经通过：

```go
go test -v . 
=== RUN   TestTemplate_ExecuteAlgorithm 
=== RUN   TestTemplate_ExecuteAlgorithm/Using_interfaces 
--- PASS: TestTemplate_ExecuteAlgorithm (0.00s) 
    --- PASS: TestTemplate_ExecuteAlgorithm/Using_interfaces (0.00s) 
PASS 
ok

```

测试通过了。测试已经检查了返回结果中是否包含字符串 `world`，这是 `hello world template` 消息。`hello` 文本是 `first()` 方法返回的字符串，`world` 字符串是由我们的 `MessageRetriever` 实现返回的，而 `template` 是 `third()` 方法返回的字符串。空格是由 Go 的 `strings.Join` 函数插入的。但任何使用 `TemplateImpl.ExecuteAlgorithm` 类型的操作都将始终在其结果中返回 "hello [something] template"。

## 匿名函数

这不是实现模板设计模式的唯一方法。我们还可以使用匿名函数将我们的实现提供给 `ExecuteAlgorithm` 方法。

让我们在之前的测试方法中编写一个测试，就在测试之后（用粗体标记）：

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

我们的新测试被称为 *使用匿名函数*。我们还已经将测试的检查提取到一个外部函数中，以便在这个测试中重用。我们把这个函数叫做 `expectedOrError`，因为它如果收到的期望值不是正确的，就会失败。

在我们的测试中，我们将创建一个名为 `AnonymousTemplate` 的类型，它替换了之前的 `Template` 类型。这个新类型的 `ExecuteAlgorithm` 方法接受 `func()` 方法 `string` 类型，我们可以在测试中直接实现它以返回字符串 `world`。

`AnonymousTemplate` 类型将具有以下结构：

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

与 `Template` 类型唯一的区别是，`ExecuteAlgorithm` 方法接受一个返回字符串的函数，而不是 `MessageRetriever` 接口。让我们运行新的测试：

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

正如你在测试执行的输出中可以看到的，错误是在 *使用匿名函数* 测试中抛出的，这正是我们预期的。现在我们将按照以下方式编写实现：

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

实现与`Template`类型中的实现相当相似。然而，现在我们传递了一个名为`f`的函数，我们将将其用作`Join`函数中使用的字符串数组中的第二个元素。由于`f`只是一个返回字符串的函数，我们唯一需要做的就是将其在适当的位置（数组的第二个位置）执行。

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

太棒了！现在我们知道了两种实现模板设计模式的方法。

## 如何避免对接口的修改

之前方法的缺点是现在我们有两个模板需要维护，我们可能会重复代码。如果我们不能改变我们使用的接口，我们该怎么办？我们的接口是`MessageRetriever`，但现在我们想使用一个匿名函数。

好吧，你还记得适配器设计模式吗？我们只需要创建一个`Adapter`类型，它接受一个`func() string`类型，并返回`MessageRetriever`接口的实现。我们将把这个类型称为`TemplateAdapter`：

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

如你所见，`TemplateAdapter`类型有一个名为`myFunc`的字段，其类型为`func() string`。我们还定义了适配器为私有，因为它不应该在没有在`myFunc`字段中定义函数的情况下使用。我们创建了一个名为`MessageRetrieverAdapter`的公共函数来实现这一点。我们的测试应该看起来大致如此：

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

看看我们调用`MessageRetrieverAdapter`方法的地方。我们传递了一个匿名函数作为参数，该函数定义为`func() string`。然后，我们重用了在第一次测试中定义的`Template`类型来传递`messageRetriever`变量。最后，我们再次使用`expectedOrError`方法进行检查。看看`MessageRetrieverAdapter`方法，它将返回一个具有 nil 值的函数。如果我们严格遵循测试驱动开发规则，我们必须先进行测试，并且测试必须在实现完成之前通过。这就是为什么我们在`MessageRetrieverAdapter`函数中返回 nil 的原因。

那么，让我们运行测试：

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

测试在代码的第*39*行失败，并且没有继续（再次，这取决于你如何编写代码，表示错误的行可能位于其他位置）。我们停止测试执行，因为我们调用`ExecuteAlgorithm`方法时需要有效的`MessageRetriever`接口。

对于我们的模板模式的适配器实现，我们将从`MessageRetrieverAdapter`方法开始：

```go
func MessageRetrieverAdapter(f func() string) MessageRetriever { 
  return &adapter{myFunc: f} 
} 

```

这很容易，对吧？你可能想知道如果我们为`f`参数传递`nil`值会发生什么。好吧，我们将通过调用`myFunc`函数来解决这个问题。

通过这个实现，`adapter`类型就完成了：

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

当调用`Message()`函数时，我们在调用之前检查`myFunc`函数中是否实际上存储了某些内容。如果没有存储任何内容，我们返回一个空字符串。

现在，我们使用适配器模式完成了对`Template`类型的第三次实现：

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

Go 源代码中的 `Sort` 包可以被认为是一个排序算法的模板实现。正如包本身所定义的，`Sort` 包为排序切片和用户定义的集合提供了原语。

在这里，我们还可以找到一个很好的例子，说明为什么 Go 的作者不担心实现泛型。对列表进行排序可能是其他语言中泛型使用的最佳例子。Go 处理这个问题的方式也非常优雅——它通过接口来处理这个问题：

```go
type Interface interface { 
  Len() int 
  Less(i, j int) bool 
  Swap(i, j int) 
} 

```

这是需要使用 `sort` 包进行排序的列表的接口。用 Go 的作者的话说：

> *"通常，类型是一个满足排序的集合。接口可以通过本包中的例程进行排序。这些方法要求集合的元素通过整数索引进行枚举。)*

换句话说，编写一个实现此 `Interface` 的类型，以便 `Sort` 包可以用于排序任何切片。排序算法是模板，我们必须定义如何在我们的切片中检索值。

如果我们查看 `sort` 包，我们还可以找到一个如何使用排序模板的例子，但我们将创建自己的例子：

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

首先，我们实现了一个非常简单的类型，它存储了一个 `int` 列表。这可以是任何类型的列表，通常是一些类型的结构体列表。然后我们通过定义 `Len`、`Swap` 和 `Less` 方法实现了 `sort.Interface` 接口。

最后，`main` 函数创建了一个 `MyList` 类型的无序列表：

```go
func main() { 
  var myList MyList = []int{6,4,2,8,1} 

  fmt.Println(myList) 
  sort.Sort(myList) 
  fmt.Println(myList) 
} 

```

我们打印了我们创建的列表（无序），然后对其进行排序（`sort.Sort` 方法实际上修改了我们的变量而不是返回一个新的列表，所以要注意！）。最后，我们再次打印结果列表。`main` 方法的控制台输出如下：

```go
go run sort_example.go 
[6 4 2 8 1]
[1 2 4 6 8]

```

`sort.Sort` 函数以透明的方式对我们的列表进行了排序。它编写了大量的代码，并将 `Len`、`Swap` 和 `Less` 方法委托给一个接口，就像我们在模板中委托给 `MessageRetriever` 接口一样。

## 总结模板设计模式

我们想重点介绍这个模式，因为它在开发库和框架时非常重要，并允许我们的库用户拥有很多灵活性和控制权。

我们再次看到，混合模式以提供用户灵活性是非常常见的，不仅是在行为上，也在结构上。当与并发应用程序一起工作时，这会非常有用，因为我们需要限制对代码部分访问以避免竞态条件。

# 备忘录设计模式

让我们现在看看一个名字很花哨的模式。如果我们查字典看看 *memento* 的意思，我们会找到以下描述：

> *"作为一个人或事件的提醒物。)*

这里，关键词是 **提醒**，因为我们将通过这个设计模式记住动作。

## 描述

备忘录的意义与它在设计模式中提供的功能非常相似。基本上，我们将有一个包含一些状态的类型，我们希望能够保存其状态的里程碑。保存有限数量的状态，我们可以在必要时恢复它们，用于各种任务——撤销操作、历史记录等。

备忘录设计模式通常有三个参与者（通常称为 **actors**）：

+   **Memento**：一个存储我们想要保存的类型。通常，我们不会直接存储业务类型，我们通过这个类型提供额外的抽象层。

+   **Originator**：一个负责创建备忘录并存储当前活动状态的类型。我们说过，Memento 类型封装了业务类型的状态，我们使用 originator 作为备忘录的创建者。

+   **Care Taker**：一个存储备忘录列表的类型，它可以有将它们存储在数据库中的逻辑，或者不存储超过指定数量的它们。

## 目标

备忘录完全关于随时间推移的一系列操作，比如撤销一个或两个操作，或者为某些应用程序提供某种事务性。

备忘录为许多任务提供了基础，但其主要目标可以定义为以下内容：

+   捕获对象状态而不修改对象本身

+   保存有限数量的状态，以便我们可以在以后检索它们

## 一个简单的字符串示例

我们将开发一个简单的示例，使用字符串作为我们想要保存的状态。这样，我们将在使用新示例使其更复杂之前，专注于常见的备忘录模式实现。

存储在 `State` 实例的字段中的字符串将被修改，我们将能够撤销在这个状态下进行的操作。

## 需求和验收标准

我们一直在谈论状态；总的来说，备忘录模式是关于存储和检索状态。我们的验收标准必须全部关于状态：

1.  我们需要存储有限数量的字符串类型的状态。

1.  我们需要一种方法来将当前存储的状态恢复到状态列表中的一个。

在这两个简单的要求下，我们已经开始为这个例子编写一些测试。

## 单元测试

如前所述，备忘录设计模式通常由三个参与者组成：状态、备忘录和 originator。因此，我们需要三个类型来表示这些参与者：

```go
type State struct { 
  Description string 
} 

```

`State` 类型是我们将在本例中使用的核心业务对象。它是我们想要跟踪的任何类型的对象：

```go
type memento struct { 
  state State 
} 

```

`memento` 类型有一个名为 `state` 的字段，表示 `State` 类型的单个值。我们的 `states` 将在这个类型中容器化，然后再存储到 `care taker` 类型中。你可能想知道为什么我们不直接存储 `State` 实例。基本上，因为这会将 `originator` 和 `careTaker` 与业务对象耦合起来，而我们希望尽可能减少耦合。它也将变得不那么灵活，正如我们在第二个示例中将会看到的：

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

`originator`类型也存储状态。`originator`结构体的对象将从 mementos 中获取状态，并使用它们存储的状态创建新的 mementos。

### 小贴士

原始对象和 Memento 模式之间的区别是什么？为什么我们不直接使用 Originator 模式的对象？嗯，如果 Memento 包含一个特定的状态，`originator`类型包含当前加载的状态。此外，保存某个状态可能像获取一些值那样简单，也可能像维护某些分布式应用程序的状态那样复杂。

`Originator`将有两个公共方法——`NewMemento()`方法和`ExtractAndStoreState(m memento)`方法。`NewMemento`方法将返回一个使用`originator`当前`State`值构建的新 Memento。`ExtractAndStoreState`方法将接受一个 Memento 的状态并将其存储在`Originator`的状态字段中：

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

`careTaker`类型存储了所有需要保存的状态的 Memento 列表。它还存储了一个`Add`方法，用于在列表中插入一个新的 Memento，以及一个 Memento 检索器，它接受 Memento 列表上的一个索引。

因此，让我们从`careTaker`类型的`Add`方法开始。`Add`方法必须接受一个`memento`对象，并将其添加到`careTaker`对象的 Mementos 列表中：

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

在我们测试的开始，我们为 memento 创建了两个基本演员——`originator`和`careTaker`。我们在`originator`上设置了一个初始状态，描述为`Idle`。

然后，我们通过调用`NewMemento`方法创建了第一个 Memento。这应该将当前`originator`的状态封装在一个`memento`类型中。我们的第一个检查非常简单——返回的 Memento 的状态描述必须类似于我们传递给`originator`的状态描述，即`Idle`描述。

检查我们的 Memento 的`Add`方法是否正确工作的最后一步是查看在添加一个项目后 Memento 列表是否已增长：

```go
  currentLen := len(careTaker.mementoList) 
  careTaker.Add(mem) 

  if len(careTaker.mementoList) != currentLen+1 { 
    t.Error("No new elements were added on the list") 
  } 

```

我们还必须测试`Memento(int) memento`方法。这个方法应该从`careTaker`列表中获取一个`memento`值。它接受你想要从列表中检索的索引，因此，像列表一样，我们必须检查它是否正确地处理了负数和越界值：

```go
func TestCareTaker_Memento(t *testing.T) { 
  originator := originator{} 
  careTaker := careTaker{} 

  originator.state = State{"Idle"} 
  careTaker.Add(originator.NewMemento()) 

```

我们必须像我们之前的测试那样开始——创建一个`originator`和`careTaker`对象，并将第一个 Memento 添加到`caretaker`中：

```go
  mem, err := careTaker.Memento(0) 
  if err != nil { 
    t.Fatal(err) 
  } 

  if mem.state.Description != "Idle" { 
    t.Error("Unexpected state") 
  } 

```

一旦我们在`careTaker`对象上有了第一个对象，我们就可以使用`careTaker.Memento(0)`来请求它。`Memento(int)`方法上的索引`0`检索切片上的第一个项目（记住切片从`0`开始）。不应该返回错误，因为我们已经向`caretaker`对象添加了一个值。

然后，在检索第一个 memento 之后，我们检查描述是否与测试开始时传递的描述匹配：

```go
  mem, err = careTaker.Memento(-1) 
  if err == nil { 
    t.Fatal("An error is expected when asking for a negative number but no error was found") 
  } 
} 

```

这个测试的最后一步涉及到使用负数来检索一些值。在这种情况下，必须返回一个错误，显示不能使用负数。当传递负数时，也有可能返回第一个索引，但在这里我们将返回一个错误。

最后要检查的函数是`ExtractAndStoreState`方法。此函数必须接受一个备忘录并提取其所有状态信息，将其设置在`Originator`对象中：

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

这个测试很简单。我们创建一个具有`Idle`状态的默认`originator`变量。然后，我们检索一个新的备忘录对象以供以后使用。我们将`originator`变量的状态更改为`Working`状态，以确保新状态将被写入。

最后，我们必须使用`idleMemento`变量调用`ExtractAndStoreState`方法。这应该将`originator`的状态恢复到`idleMemento`状态值，这是我们之前在最后一个`if`语句中检查过的。

现在是时候运行测试了：

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

由于三个测试失败，我们可以继续实施。

## 实现备忘录模式

如果你不做得太过分，备忘录模式的实现通常非常简单。三个参与者（`memento`、`originator`和`care taker`）在模式中具有非常明确的角色，它们的实现非常直接：

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

当调用`NewMemento`方法时，`Originator`对象需要返回 Memento 类型的新值。它还需要根据需要将`memento`对象存储在结构体的状态字段中，以便用于`ExtractAndStoreState`方法：

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

`careTaker`类型也很简单。当我们调用`Add`方法时，我们通过调用带有参数传递的值的`append`方法来覆盖`mementoList`字段。这创建了一个包含新值的新列表。

在调用`Memento`方法之前，我们必须做一些检查。在这种情况下，我们检查索引是否不在切片的范围之外，以及索引在`if`语句中是否不是负数，在这种情况下我们返回一个错误。如果一切顺利，它只返回指定的`memento`对象，没有错误。

### 小贴士

关于方法和函数命名约定的说明。你可能会发现有些人喜欢给方法如`Memento`起一个稍微描述性更强的名字。一个例子是使用像`MementoOrError`这样的名字，清楚地表明在调用此函数时返回两个对象，甚至`GetMementoOrError`方法。这可能是一种非常明确的命名方法，但这并不一定不好，但在 Go 的源代码中你不太可能找到它。

是时候检查测试结果了：

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

这已经足够达到 100%的覆盖率。虽然这远非一个完美的指标，但至少我们知道我们正在触及源代码的每一个角落，而且我们没有在测试中作弊以达到这个目标。

## 另一个使用命令和外观模式的例子

之前的例子很好，足够简单，可以理解 Memento 模式的功能。然而，它更常与命令模式和简单的外观模式一起使用。

策略是使用命令模式来封装一组不同类型的状态（那些实现`Command`接口的状态），并提供一个小型的外观来自动化在`caretaker`对象中的插入。

我们将要开发一个假设的音频混音器的简单示例。我们将使用相同的 Memento 模式来保存两种状态：`Volume`和`Mute`。`Volume`状态将是一个字节类型，而`Mute`状态是一个布尔类型。我们将使用两种完全不同的类型来展示这种方法的灵活性（及其缺点）。

作为旁注，我们还可以在每个`Command`接口上提供它们自己的序列化方法。这样，我们可以赋予保管者存储状态的能力，而不必真正知道存储的是什么。

我们的`Command`接口将有一个方法来返回其实现者的值。这很简单，我们音频混音器中每个想要撤销的命令都必须实现这个接口：

```go
type Command interface { 
  GetValue() interface{} 
} 

```

在这个界面中有些有趣的东西。`GetValue`方法返回一个值的接口。这也意味着这个方法的返回类型是...嗯...无类型的？其实不是，但它返回一个可以代表任何类型的接口，我们稍后如果想要使用其特定类型，需要将其类型转换。现在我们必须定义`Volume`和`Mute`类型并实现`Command`接口：

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

它们的实现都很简单。然而，`Mute`类型将在`GetValue()`方法上返回一个`bool`类型，而`Volume`将返回一个`byte`类型。

如同先前的例子，我们需要一个`Memento`类型来保存一个`Command`。换句话说，它将存储一个指向`Mute`或`Volume`类型的指针：

```go
type Memento struct { 
  memento Command 
} 

```

`originator`类型在先前的例子中是这样工作的，但这次使用的是`Command`关键字而不是`state`关键字：

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

`caretaker`对象几乎和之前一样，但这次我们将使用一个栈而不是简单的列表，并且我们将存储一个命令而不是状态：

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

然而，我们的`Memento`列表被一个`Pop`方法所取代。它也返回一个`memento`对象，但它将以栈的形式返回它们（后进先出）。因此，我们从栈中取出最后一个元素并将其存储在`tempMemento`变量中。然后，在下一行我们将栈替换为一个不包含最后一个元素的新版本。最后，我们返回`tempMemento`变量。

到目前为止，一切看起来几乎和上一个例子一样。我们也讨论了通过使用门面模式来自动化一些任务，所以让我们来做吧。这将被称为`MementoFacade`类型，并将具有`SaveSettings`和`RestoreSettings`方法。`SaveSettings`方法接受一个`Command`，将其存储在一个内部发起者中，并保存在一个内部的`careTaker`字段中。`RestoreSettings`方法执行相反的操作——恢复`careTaker`的索引并返回`Memento`对象内的`Command`：

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

我们的门面模式将保存发起者和保管者的内容，并提供这两个易于使用的方法来保存和恢复设置。

那么，我们该如何使用这个方法呢？

```go
func main(){ 
  m := MementoFacade{} 

  m.SaveSettings(Volume(4)) 
  m.SaveSettings(Mute(false)) 

```

首先，我们使用门面模式获取一个变量。零值初始化将给我们零值的`originator`和`caretaker`对象。它们没有任何意外的字段，所以一切都将正确初始化（如果它们中有任何指针，例如，它将被初始化为`nil`，如第一章中*零初始化*部分所述），*准备... 稳定... 开始!*）。

我们使用`Volume(4)`创建一个`Volume`值，是的，我们使用了括号。`Volume`类型没有像结构体那样的内部字段，所以我们不能使用花括号来设置其值。设置它的方法是使用括号（或者创建一个指向类型`Volume`的指针，然后设置指向空间的值）。我们还使用门面模式保存了一个`Mute`类型的值。

我们不知道返回的是哪种`Command`类型，所以我们需要做一个类型断言。我们将创建一个小的函数来帮助我们完成这项工作，该函数检查类型并打印适当的值：

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

`assertAndPrint`方法接受一个`Command`类型，并将其转换为两种可能类型——`Volume`或`Mute`。在每种情况下，它都会在控制台上打印一条带有个性化信息的消息。现在我们可以继续并完成`main`函数，它将看起来像这样：

```go
func main() { 
  m := MementoFacade{} 

  m.SaveSettings(Volume(4)) 
  m.SaveSettings(Mute(false)) 

 assertAndPrint(m.RestoreSettings(0))
 assertAndPrint(m.RestoreSettings(1)) 
} 

```

粗体部分显示了`main`函数中的新更改。我们从`careTaker`对象中取出索引 0 并传递给新函数，同样也传递了索引`1`。运行这个小程序，我们应该在控制台上得到`Volume`和`Mute`的值：

```go
$ go run memento_command.go
Mute:   false
Volume: 4

```

太棒了！在这个小例子中，我们结合了三种不同的设计模式，以便更舒适地使用各种模式。记住，我们也可以将`Volume`和`Mute`状态的创建抽象为工厂模式，所以这并不是我们停止的地方。

## 最后关于备忘录模式的讨论

通过备忘录模式，我们学习了一种创建不可逆操作的有效方法，这在编写 UI 应用程序时非常有用，而且在开发事务性操作时也同样有用。无论如何，情况都是一样的：你需要一个`备忘录`，一个`发起者`和一个`保管者`角色。

### 小贴士

**事务操作**是一组原子操作，必须全部完成或失败。换句话说，如果你有一个由五个操作组成的事务，只要其中一个操作失败，事务就无法完成，其他四个操作所做的所有修改都必须撤销。

# 解释器设计模式

现在我们将深入探讨一个相当复杂的模式。**解释器模式**实际上被广泛用于解决需要有一种语言来执行常见操作的业务案例。让我们看看我们所说的“语言”是什么意思。

## 描述

我们可以谈论的最著名的解释者可能是 SQL。它被定义为用于管理关系数据库中存储数据的专用编程语言。SQL 非常复杂且庞大，但总的来说，它是一组单词和运算符，允许我们执行插入、选择或删除等操作。

另一个典型的例子是音乐记谱法。它本身是一种语言，解释者是知道音符与其所演奏乐器上表示之间关系的音乐家。

在计算机科学中，出于各种原因，设计一个小语言可能是有用的：重复性任务、为非开发者提供高级语言，或者**接口定义语言（IDL**）如**协议缓冲区**或**Apache Thrift**。

## 目标

设计一种新语言，无论大小，都可能是一项耗时的工作，因此在投入时间和资源编写解释器之前，明确目标非常重要：

+   为某些范围内的非常常见的操作提供语法（例如演奏音符）。

+   有一个中间语言来在两个系统之间翻译动作。例如，生成用于 3D 打印的 **Gcode** 所需的应用程序。

+   简化某些操作的使用，使其语法更易于使用。

SQL 允许使用非常易于使用的语法（也可能变得非常复杂）来使用关系数据库，但理念是不需要编写自己的函数来进行插入和搜索操作。

## 示例 - 一个波兰表示法计算器

解释器的一个非常典型的例子是创建一个逆波兰表示法计算器。对于那些不知道波兰表示法的人来说，它是一种数学记法，用于进行操作，你首先写下操作（加法）然后是值（3 4），所以 *+ 3 4* 等同于更常见的 *3 + 4*，其结果将是 *7*。因此，对于逆波兰表示法，你首先放置值然后是操作，所以 *3 4 +* 也会是 *7*。

## 计算器的验收标准

对于我们的计算器，我们应该通过的验收标准如下：

1.  创建一种语言，允许进行常见的算术运算（加法、减法、乘法和除法）。语法是 `sum` 用于加法，`mul` 用于乘法，`sub` 用于减法，`div` 用于除法。

1.  必须使用逆波兰表示法来完成。

1.  用户必须能够连续编写他们想要的任意数量的操作。

1.  操作必须从左到右执行。

因此，`3 4 sum 2 sub`的表示法与`(3 + 4) - 2`相同，结果将是*5*。

## 一些操作的单元测试

在这种情况下，我们只有一个公共方法`Calculate`，它接受一个以字符串形式定义的操作，并返回一个值或错误：

```go
func Calculate(o string) (int, error) { 
  return 0, fmt.Errorf("Not implemented yet") 
} 

```

因此，我们将发送一个类似`"3 4 +"`的字符串到`Calculate`方法，并且它应该返回*7, nil*。另外两个测试将检查正确的实现：

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

首先，我们将使用作为示例的操作。`3 4 sum 2 sub`的表示法是我们语言的一部分，我们在`Calculate`函数中使用它。如果返回错误，则测试失败。最后，结果必须等于`5`，我们在最后一行检查它。下一个测试检查其他运算符在稍微复杂一些的操作上的实现：

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

在这里，我们使用更长的操作重复了前面的过程，即`(((5 - 3) * 8) + 4) / 5`的表示法，它等于*4*。从左到右，它将是以下这样：

```go
(((5 - 3) * 8) + 4) / 5
 ((2 * 8) + 4) / 5
 (16 + 4) / 5
 20 / 5
 4

```

测试当然必须失败！

```go
$ go test -v .
 interpreter_test.go:9: Not implemented yet
 interpreter_test.go:13: Expected result not found: 4 != 0
 interpreter_test.go:19: Not implemented yet
 interpreter_test.go:23: Expected result not found: 5 != 0
exit status 1
FAIL

```

## 实现

这次实现将比测试更长。首先，我们将定义我们的可能运算符在常量中：

```go
const ( 
  SUM = "sum" 
  SUB = "sub" 
  MUL = "mul" 
  DIV = "div" 
) 

```

解释器模式通常使用抽象语法树来实现，这通常是通过使用栈来实现的。我们在本书中已经创建过栈，所以这应该对读者来说已经很熟悉了：

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

我们有两种方法--`Push`方法用于将元素添加到栈顶，以及`Pop`方法用于移除元素并返回它们。如果你认为这一行`*p = (*p)[:length-1]`有点晦涩，我们将对其进行解释。

存储在`p`方向上的值将被`p (*p)`方向上的实际值覆盖，但只取数组的前一个元素（`(:length-1)`）。

现在，我们将逐步进行`Calculate`函数，根据需要创建更多函数：

```go
func Calculate(o string) (int, error) { 
  stack := polishNotationStack{} 
  operators := strings.Split(o, " ") 

```

我们需要做的第一件事是创建栈，并从传入的操作中获取所有不同的符号（在这种情况下，我们并没有检查它是否为空）。我们通过空格分割传入的字符串操作，以获取一个漂亮的符号切片（值和运算符）。

接下来，我们将使用 range 遍历每个符号，但我们需要一个函数来知道传入的符号是值还是运算符：

```go
func isOperator(o string) bool { 
  if o == SUM || o == SUB || o == MUL || o == DIV { 
    return true 
  } 

  return false 
} 

```

如果传入的符号是我们在常量中定义的任何一个，则传入的符号是一个运算符：

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

如果它是一个运算符，我们认为我们已经通过了两个值，所以我们需要从栈中取出这两个值。第一个取出的值将是最右边的，第二个是最左边的（记住，在减法和除法中，操作数的顺序很重要）。然后，我们需要一个函数来获取我们想要执行的操作：

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

`getOperationFunc` 函数返回一个接受两个参数的函数，该函数返回一个整数。我们检查传入的操作符，并返回一个执行指定操作的匿名函数。所以，现在我们的 `for range` 继续这样：

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

`mathFunc` 变量由函数返回。我们立即使用它来对从栈中取出的左右值执行操作，并将结果存储在一个名为 `res` 的新变量中。最后，我们需要将这个新值推入栈中，以便稍后继续操作。

现在，当传入的符号是一个值时，这里是实现：

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

每次我们得到一个符号时，我们需要将其推入栈中。我们必须将字符串符号解析为可用的 `int` 类型。这通常通过使用 `strconv` 包的 `Atoi` 函数来完成。`Atoi` 函数接受一个字符串，并从中返回一个整数或错误。如果一切顺利，值将被推入栈中。

在 `range` 语句的末尾，只需存储一个值，所以我们只需返回它，函数就完成了：

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

太好了！我们刚刚以非常简单和容易的方式创建了一个逆波兰表达式解释器（我们仍然缺少解析器，但这又是另一个故事）。

## 解释器设计模式带来的复杂性

在这个例子中，我们没有使用任何接口。这并不完全符合在更面向对象的语言中定义的解释器设计模式。然而，这个例子是理解语言目标和下一个更复杂层次的最简单例子，并且并不适合初学者用户。

使用更复杂的例子，我们可能需要定义一个包含更多自身类型、一个值或什么都没有的类型。使用解析器，你可以创建这个抽象语法树以供以后解释。

同样的例子，通过使用接口实现，将在以下描述部分中展示。

## 再次使用接口实现解释器模式

我们将要使用的主要接口称为 `Interpreter` 接口。该接口有一个 `Read()` 方法，每个符号（值或操作符）都必须实现：

```go
type Interpreter interface { 
  Read() int 
} 

```

我们将只实现操作符的加法和减法以及一个名为 `Value` 的数字类型：

```go
type value int 

func (v *value) Read() int { 
  return int(*v) 
} 

```

`Value` 是一个 `int` 类型，当实现 `Read` 方法时，只返回其值：

```go
type operationSum struct { 
  Left  Interpreter 
  Right Interpreter 
} 

func (a *operationSum) Read() int { 
  return a.Left.Read() + a.Right.Read() 
} 

```

`operationSum` 结构体具有 `Left` 和 `Right` 字段，其 `Read` 方法返回它们各自的 `Read` 方法的和。`operationSubtract` 结构体与之相同，但执行减法操作：

```go
type operationSubtract struct { 
  Left  Interpreter 
  Right Interpreter 
} 

func (s *operationSubtract) Read() int { 
  return s.Left.Read() - s.Right.Read() 
} 

```

我们还需要一个工厂模式来创建操作符；我们将称之为 `operatorFactory` 方法。现在的不同之处在于它不仅接受符号，还接受从栈中取出的 `Left` 和 `Right` 值：

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

正如我们刚才提到的，我们还需要一个栈。我们可以通过更改其类型来重用之前的例子：

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

现在栈使用解释器指针而不是`int`，但其功能相同。最后，我们的`main`方法也类似于之前的示例：

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

和之前一样，我们首先检查符号是运算符还是值。当它是值时，我们将其推入栈中。

当符号是运算符时，我们也从栈中取出左右值，使用当前运算符和从栈中取出的左右值调用工厂模式。一旦我们有了运算符类型，我们只需要调用它的`Read`方法，将返回的值推送到栈中。

最后，只留下一个示例在栈上，所以我们打印它：

```go
$ go run interpreter.go
5

```

## 解释器模式的威力

这种模式非常强大，但必须谨慎使用。创建一种语言，它会在其用户和提供的功能之间产生强烈的耦合。人们可能会陷入试图创建过于灵活的语言的错误，这种语言使用和维护起来极其复杂。此外，人们可能会创建一个相当小而有用的语言，但有时它不能正确解释，这可能会给用户带来麻烦。

在我们的例子中，我们省略了大量的错误检查，以便专注于解释器的实现。然而，你需要大量的错误检查和详细的错误输出，以帮助用户纠正其语法错误。所以，编写你的语言时尽情享受乐趣，但要对你的用户友好。

# 摘要

本章讨论了三个极其强大的模式，在使用它们的生产代码之前需要大量练习。通过模拟典型的生产问题来对这些模式进行一些练习是一个非常好的主意：

+   创建一个简单的 REST 服务器，重用大部分错误检查和连接功能，以提供一个易于使用的接口来练习模板模式。

+   创建一个小型库，可以写入不同的数据库，但只有在所有写入都正常的情况下，或者删除新创建的写入以练习 Memento 为例。

+   编写你自己的语言，以便练习像机器人通常那样回答简单问题，这样你就可以练习一点解释器模式。

这个想法是练习编码并重新阅读任何部分，直到你对每个模式感到舒适。
