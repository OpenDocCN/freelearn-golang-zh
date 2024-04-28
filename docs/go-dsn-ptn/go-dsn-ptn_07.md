# 第七章。行为模式 - 访问者，状态，中介者和观察者设计模式

这是关于行为模式的最后一章，也是本书关于 Go 语言中常见的、众所周知的设计模式的部分的结束。

在本章中，我们将研究另外三种设计模式。当您想要从一组对象中抽象出一些功能时，访问者模式非常有用。

状态通常用于构建**有限状态机**（**FSM**），在本节中，我们将开发一个小的*猜数字*游戏。

最后，观察者模式通常用于事件驱动的架构，并且在微服务世界中再次获得了很多关注。

在本章之后，我们需要在深入并发和它带来的设计模式的优势（和复杂性）之前，对常见的设计模式感到非常舒适。

# 访问者设计模式

在下一个设计模式中，我们将把对象类型的一些逻辑委托给一个名为访问者的外部类型，该类型将访问我们的对象以对其执行操作。

## 描述

在访问者设计模式中，我们试图将与特定对象一起工作所需的逻辑与对象本身分离。因此，我们可以有许多不同的访问者对特定类型执行某些操作。

例如，想象一下我们有一个写入控制台的日志记录器。我们可以使记录器“可访问”，以便您可以在每个日志前添加任何文本。我们可以编写一个访问者模式，它将日期、时间和主机名添加到对象中存储的字段。

## 目标

在行为设计模式中，我们主要处理算法。访问者模式也不例外。我们试图实现的目标如下：

+   将某种类型的算法与其在其他类型中的实现分离

+   通过使用一些类型来提高其灵活性，几乎不需要任何逻辑，因此所有新功能都可以添加而不改变对象结构

+   修复会破坏类型中的开闭原则的结构或行为

您可能会想知道开闭原则是什么。在计算机科学中，开闭原则指出：*实体应该对扩展开放，但对修改关闭*。这个简单的状态有很多含义，可以构建更易于维护且不太容易出错的软件。访问者模式帮助我们将一些常常变化的算法从我们需要它“稳定”的类型委托给一个经常变化的外部类型，而不会影响我们的原始类型。

## 日志附加器

我们将开发一个简单的日志附加器作为访问者模式的示例。遵循我们在之前章节中的方法，我们将从一个极其简单的示例开始，以清楚地理解访问者设计模式的工作原理，然后再转向更复杂的示例。我们已经开发了类似的示例，但以稍微不同的方式修改文本。

对于这个特定的例子，我们将创建一个访问者，它会向“访问”的类型附加不同的信息。

## 验收标准

要有效地使用访问者设计模式，我们必须有两个角色--访问者和可访问者。`Visitor`是将在`Visitable`类型内执行的类型。因此，`Visitable`接口实现将算法分离到`Visitor`类型：

1.  我们需要两个消息记录器：`MessageA`和`MessageB`，它们将在消息之前分别打印带有`A：`或`B：`的消息。

1.  我们需要一个访问者能够修改要打印的消息。它将分别将文本“Visited A”或“Visited B”附加到它们。

## 单元测试

正如我们之前提到的，我们将需要`Visitor`和`Visitable`接口的角色。它们将是接口。我们还需要`MessageA`和`MessageB`结构：

```go
package visitor 

import ( 
  "io" 
  "os" 
  "fmt" 
) 

type MessageA struct { 
  Msg string 
  Output io.Writer 
} 

type MessageB struct { 
  Msg string 
  Output io.Writer 
} 

type Visitor interface { 
  VisitA(*MessageA) 
  VisitB(*MessageB) 
} 

type Visitable interface { 
  Accept(Visitor) 
} 

type MessageVisitor struct {} 

```

`MessageA` 和 `MessageB` 结构都有一个 `Msg` 字段来存储它们将要打印的文本。输出 `io.Writer` 将默认实现 `os.Stdout` 接口，或者一个新的 `io.Writer` 接口，就像我们将用来检查内容是否正确的接口一样。

`Visitor` 接口有一个 `Visit` 方法，分别用于 `Visitable` 接口的 `MessageA` 和 `MessageB` 类型。`Visitable` 接口有一个名为 `Accept(Visitor)` 的方法，将执行解耦的算法。

与以前的示例一样，我们将创建一个实现 `io.Writer` 包的类型，以便我们可以在测试中使用它：

```go
package visitor 

import "testing" 

type TestHelper struct { 
  Received string 
} 

func (t *TestHelper) Write(p []byte) (int, error) { 
  t.Received = string(p) 
  return len(p), nil 
} 

```

`TestHelper` 结构实现了 `io.Writer` 接口。它的功能非常简单；它将写入的字节存储在 `Received` 字段上。稍后我们可以检查 `Received` 的内容来测试是否符合我们的预期值。

我们将只编写一个测试，检查代码的整体正确性。在这个测试中，我们将编写两个子测试：一个用于 `MessageA`，一个用于 `MessageB` 类型：

```go
func Test_Overall(t *testing.T) { 
  testHelper := &TestHelper{} 
  visitor := &MessageVisitor{} 
  ... 
} 

```

我们将在每个消息类型的每个测试中使用一个 `TestHelper` 结构和一个 `MessageVisitor` 结构。首先，我们将测试 `MessageA` 类型：

```go
func Test_Overall(t *testing.T) { 
  testHelper := &TestHelper{} 
  visitor := &MessageVisitor{} 

  t.Run("MessageA test", func(t *testing.T){ 
    msg := MessageA{ 
      Msg: "Hello World", 
      Output: testHelper, 
    } 

    msg.Accept(visitor) 
    msg.Print() 

    expected := "A: Hello World (Visited A)" 
    if testHelper.Received !=  expected { 
      t.Errorf("Expected result was incorrect. %s != %s", 
      testHelper.Received, expected) 
    } 
  }) 
  ... 
} 

```

这是完整的第一个测试。我们创建了 `MessageA` 结构，为 `Msg` 字段赋予了值 `Hello World`，并为其传递了在测试开始时创建的 `TestHelper` 的指针。然后，我们执行它的 `Accept` 方法。在 `MessageA` 结构的 `Accept(Visitor)` 方法中，将执行 `VisitA(*MessageA)` 方法来改变 `Msg` 字段的内容（这就是为什么我们传递了 `VisitA` 方法的指针，没有指针内容将不会被持久化）。

为了测试 `Visitor` 类型在 `Accept` 方法中是否完成了其工作，我们必须稍后在 `MessageA` 类型上调用 `Print()` 方法。这样，`MessageA` 结构必须将 `Msg` 的内容写入提供的 `io.Writer` 接口（我们的 `TestHelper`）。

测试的最后一部分是检查。根据*验收标准 2*的描述，`MessageA` 类型的输出文本必须以文本 `A:` 为前缀，存储的消息和文本 `"(Visited)"` 为结尾。因此，对于 `MessageA` 类型，期望的文本必须是 `"A: Hello World (Visited)"`，这是我们在 `if` 部分进行的检查。

`MessageB` 类型有一个非常相似的实现：

```go
  t.Run("MessageB test", func(t *testing.T){ 
    msg := MessageB { 
      Msg: "Hello World", 
      Output: testHelper, 
    } 

    msg.Accept(visitor) 
    msg.Print() 

    expected := "B: Hello World (Visited B)" 
    if testHelper.Received !=  expected { 
      t.Errorf("Expected result was incorrect. %s != %s", 
        testHelper.Received, expected) 
    } 
  }) 
} 

```

实际上，我们刚刚将类型从 `MessageA` 更改为 `MessageB`，现在期望的文本是 `"B: Hello World (Visited B)"`。`Msg` 字段也是 `"Hello World"`，我们还使用了 `TestHelper` 类型。

我们仍然缺少正确的接口实现来编译代码并运行测试。`MessageA` 和 `MessageB` 结构必须实现 `Accept(Visitor)` 方法：

```go
func (m *MessageA) Accept(v Visitor) { 
  //Do nothing 
} 

func (m *MessageB) Accept(v Visitor) { 
  //Do nothing 
} 

```

我们需要实现在 `Visitor` 接口上声明的 `VisitA(*MessageA)` 和 `VisitB(*MessageB)` 方法。`MessageVisitor` 接口是必须实现它们的类型：

```go
func (mf *MessageVisitor) VisitA(m *MessageA){ 
  //Do nothing 
} 
func (mf *MessageVisitor) VisitB(m *MessageB){ 
  //Do nothing 
} 

```

最后，我们将为每种消息类型创建一个 `Print()` 方法。这是我们将用来测试每种类型的 `Msg` 字段内容的方法：

```go
func (m *MessageA) Print(){ 
  //Do nothing 
} 

func (m *MessageB) Print(){ 
  //Do nothing 
} 

```

现在我们可以运行测试，真正检查它们是否已经失败：

```go
go test -v .
=== RUN   Test_Overall
=== RUN   Test_Overall/MessageA_test
=== RUN   Test_Overall/MessageB_test
--- FAIL: Test_Overall (0.00s)
 --- FAIL: Test_Overall/MessageA_test (0.00s)
 visitor_test.go:30: Expected result was incorrect.  != A: Hello World (Visited A)
 --- FAIL: Test_Overall/MessageB_test (0.00s)
 visitor_test.go:46: Expected result was incorrect.  != B: Hello World (Visited B)
FAIL
exit status 1
FAIL

```

测试的输出很清楚。期望的消息是不正确的，因为内容是空的。现在是创建实现的时候了。

## 访问者模式的实现

我们将开始完成 `VisitA(*MessageA)` 和 `VisitB(*MessageB)` 方法的实现：

```go
func (mf *MessageVisitor) VisitA(m *MessageA){ 
  m.Msg = fmt.Sprintf("%s %s", m.Msg, "(Visited A)") 
} 
func (mf *MessageVisitor) VisitB(m *MessageB){ 
  m.Msg = fmt.Sprintf("%s %s", m.Msg, "(Visited B)") 
} 

```

它的功能非常简单- `fmt.Sprintf` 方法返回一个格式化的字符串，其中包含 `m.Msg` 的实际内容、一个空格和消息 `Visited`。这个字符串将被存储在 `Msg` 字段上，覆盖先前的内容。

现在我们将为每种消息类型开发 `Accept` 方法，该方法必须执行相应的 Visitor：

```go
func (m *MessageA) Accept(v Visitor) { 
  v.VisitA(m) 
} 

func (m *MessageB) Accept(v Visitor) { 
  v.VisitB(m) 
} 

```

这段小代码有一些含义。在这两种情况下，我们都使用了一个`Visitor`，在我们的例子中，它与`MessageVisitor`接口完全相同，但它们可以完全不同。关键是要理解访问者模式在其`Visit`方法中执行处理`Visitable`对象的算法。`Visitor`可能在做什么？在这个例子中，它改变了`Visitable`对象，但它也可以简单地从中获取信息。例如，我们可以有一个`Person`类型，有很多字段：姓名、姓氏、年龄、地址、城市、邮政编码等等。我们可以编写一个访问者，仅从一个人中获取姓名和姓氏作为唯一的字符串，一个访问者从应用程序的不同部分获取地址信息，等等。

最后，有一个`Print()`方法，它将帮助我们测试这些类型。我们之前提到它必须默认打印到`Stdout`：

```go
func (m *MessageA) Print() { 
  if m.Output == nil { 
    m.Output = os.Stdout 
  } 

  fmt.Fprintf(m.Output, "A: %s", m.Msg) 
} 

func (m *MessageB) Print() { 
  if m.Output == nil { 
    m.Output = os.Stdout 
  } 
  fmt.Fprintf(m.Output, "B: %s", m.Msg) 
} 

```

首先检查`Output`字段的内容，以便在`os.Stdout`调用的输出为空时将其赋值。在我们的测试中，我们在那里存储了一个指向我们的`TestHelper`类型的指针，因此在我们的测试中永远不会执行这行。最后，每个消息类型都会将存储在`Msg`字段中的完整消息打印到`Output`字段。这是通过使用`Fprintf`方法完成的，该方法将`io.Writer`包作为第一个参数，要格式化的文本作为下一个参数。

我们的实现现在已经完成，我们可以再次运行测试，看看它们是否都通过了：

```go
go test -v .
=== RUN   Test_Overall
=== RUN   Test_Overall/MessageA_test
=== RUN   Test_Overall/MessageB_test
--- PASS: Test_Overall (0.00s)
 --- PASS: Test_Overall/MessageA_test (0.00s)
 --- PASS: Test_Overall/MessageB_test (0.00s)
PASS
ok

```

一切都很好！访问者模式已经完美地完成了它的工作，调用它们的`Visit`方法后，消息内容已经被改变。这里非常重要的一点是，我们可以为这两个结构体添加更多功能，`MessageA`和`MessageB`，而不改变它们的类型。我们只需创建一个新的访问者类型，对`Visitable`上的所有操作进行处理，例如，我们可以创建一个`Visitor`来添加一个打印`Msg`字段内容的方法：

```go
type MsgFieldVisitorPrinter struct {} 

func (mf *MsgFieldVisitorPrinter) VisitA(m *MessageA){ 
  fmt.Printf(m.Msg) 
} 
func (mf *MsgFieldVisitorPrinter) VisitB(m *MessageB){ 
  fmt.Printf(m.Msg) 
} 

```

我们刚刚为这两种类型添加了一些功能，而没有改变它们的内容！这就是访问者设计模式的威力。

## 另一个例子

我们将开发第二个例子，这个例子会更加复杂一些。在这种情况下，我们将模拟一个有几种产品的在线商店。产品将具有简单的类型，只有字段，我们将创建一对访问者来处理它们。

首先，我们将开发接口。`ProductInfoRetriever` 类型有一个方法来获取产品的价格和名称。`Visitor` 接口，就像之前一样，有一个接受 `ProductInfoRetriever` 类型的 `Visit` 方法。最后，`Visitable` 接口完全相同；它有一个接受 `Visitor` 类型作为参数的 `Accept` 方法。

```go
type ProductInfoRetriever interface { 
  GetPrice() float32 
  GetName() string 
} 

type Visitor interface { 
  Visit(ProductInfoRetriever) 
} 

type Visitable interface { 
  Accept(Visitor) 
} 

```

在线商店的所有产品都必须实现`ProductInfoRetriever`类型。此外，大多数产品都将具有一些共同的字段，例如名称或价格（在`ProductInfoRetriever`接口中定义的字段）。我们创建了`Product`类型，实现了`ProductInfoRetriever`和`Visitable`接口，并将其嵌入到每个产品中：

```go
type Product struct { 
  Price float32 
  Name  string 
} 

func (p *Product) GetPrice() float32 { 
  return p.Price 
} 

func (p *Product) Accept(v Visitor) { 
  v.Visit(p) 
} 

func (p *Product) GetName() string { 
  return p.Name 
} 

```

现在我们有一个非常通用的`Product`类型，可以存储商店几乎任何产品的信息。例如，我们可以有一个`Rice`和一个`Pasta`产品：

```go
type Rice struct { 
  Product 
} 

type Pasta struct { 
  Product 
} 

```

每个都嵌入了`Product`类型。现在我们需要创建一对`Visitors`接口，一个用于计算所有产品的价格总和，一个用于打印每个产品的名称：

```go
type PriceVisitor struct { 
  Sum float32 
} 

func (pv *PriceVisitor) Visit(p ProductInfoRetriever) { 
  pv.Sum += p.GetPrice() 
} 

type NamePrinter struct { 
  ProductList string 
} 

func (n *NamePrinter) Visit(p ProductInfoRetriever) { 
  n.Names = fmt.Sprintf("%s\n%s", p.GetName(), n.ProductList) 
} 

```

`PriceVisitor`结构体获取`ProductInfoRetriever`类型的`Price`变量的值，作为参数传递，并将其添加到`Sum`字段。`NamePrinter`结构体存储`ProductInfoRetriever`类型的名称，作为参数传递，并将其附加到`ProductList`字段的新行上。

现在是`main`函数的时间：

```go
func main() { 
  products := make([]Visitable, 2) 
  products[0] = &Rice{ 
    Product: Product{ 
      Price: 32.0, 
      Name:  "Some rice", 
    }, 
  } 
  products[1] = &Pasta{ 
    Product: Product{ 
      Price: 40.0, 
      Name:  "Some pasta", 
    }, 
  } 

  //Print the sum of prices 
  priceVisitor := &PriceVisitor{} 

  for _, p := range products { 
    p.Accept(priceVisitor) 
  } 

  fmt.Printf("Total: %f\n", priceVisitor.Sum) 

  //Print the products list 
  nameVisitor := &NamePrinter{} 

  for _, p := range products { 
    p.Accept(nameVisitor) 
  } 

  fmt.Printf("\nProduct list:\n-------------\n%s",  nameVisitor.ProductList) 
} 

```

我们创建了两个`Visitable`对象的切片：一个`Rice`和一个`Pasta`类型，带有一些任意的名称。然后我们使用`PriceVisitor`实例作为参数对它们进行迭代。在`range for`之后，我们打印总价格。最后，我们使用`NamePrinter`重复这个操作，并打印结果的`ProductList`。这个`main`函数的输出如下：

```go
go run visitor.go
Total: 72.000000
Product list:
-------------
Some pasta
Some rice

```

好的，这是访问者模式的一个很好的例子，但是...如果产品有特殊的考虑呢？例如，如果我们需要在冰箱类型的总价格上加 20 呢？好的，让我们编写`Fridge`结构：

```go
type Fridge struct { 
  Product 
} 

```

这里的想法是只需重写`GetPrice()`方法，以返回产品的价格加 20：

```go
type Fridge struct { 
  Product 
} 

func (f *Fridge) GetPrice() float32 { 
  return f.Product.Price + 20 
} 

```

不幸的是，这对我们的例子来说还不够。`Fridge`结构不是`Visitable`类型。`Product`结构是`Visitable`类型，而`Fridge`结构嵌入了一个`Product`结构，但是正如我们在前几章中提到的，嵌入第二种类型的类型不能被视为后者的类型，即使它具有所有的字段和方法。解决方案是还要实现`Accept(Visitor)`方法，以便它可以被视为`Visitable`：

```go
type Fridge struct { 
  Product 
} 

func (f *Fridge) GetPrice() float32 { 
  return f.Product.Price + 20 
} 

func (f *Fridge) Accept(v Visitor) { 
  v.Visit(f) 
} 

```

让我们重写`main`函数以将这个新的`Fridge`产品添加到切片中：

```go
func main() { 
  products := make([]Visitable, 3) 
  products[0] = &Rice{ 
    Product: Product{ 
      Price: 32.0, 
      Name:  "Some rice", 
    }, 
  } 
  products[1] = &Pasta{ 
    Product: Product{ 
      Price: 40.0, 
      Name:  "Some pasta", 
    }, 
  } 
  products[2] = &Fridge{ 
    Product: Product{ 
      Price: 50, 
      Name:  "A fridge", 
    }, 
  } 
  ... 
} 

```

其他一切都保持不变。运行这个新的`main`函数会产生以下输出：

```go
$ go run visitor.go
Total: 142.000000
Product list:
-------------
A fridge
Some pasta
Some rice

```

如预期的那样，总价格现在更高了，输出了大米（32）、意大利面（40）和冰箱（50 的产品加上 20 的运输，所以是 70）的总和。我们可以不断地为这些产品添加访问者，但是想法很清楚——我们将一些算法解耦到访问者之外。

## 访问者来拯救！

我们已经看到了一个强大的抽象，可以向某些类型添加新的算法。然而，由于 Go 语言中缺乏重载，这种模式在某些方面可能有限（我们在第一个示例中已经看到了这一点，在那里我们不得不创建`VisitA`和`VisitB`的实现）。在第二个示例中，我们没有处理这个限制，因为我们使用了`Visitor`结构的`Visit`方法的接口，但我们只使用了一种类型的访问者（`ProductInfoRetriever`），如果我们为第二种类型实现了`Visit`方法，我们将会遇到相同的问题，这是原始*四人帮*设计模式的目标之一。

# 状态设计模式

状态模式与 FSM 直接相关。FSM，简单来说，是具有一个或多个状态并在它们之间移动以执行某些行为的东西。让我们看看状态模式如何帮助我们定义 FSM。

## 描述

一个灯开关是 FSM 的一个常见例子。它有两种状态——开和关。一种状态可以转移到另一种状态，反之亦然。状态模式的工作方式类似。我们有一个`State`接口和我们想要实现的每个状态的实现。通常还有一个上下文，用于在状态之间保存交叉信息。

通过 FSM，我们可以通过将其范围分割为状态来实现非常复杂的行为。这样我们可以基于任何类型的输入来建模执行管道，或者创建对特定事件以指定方式做出响应的事件驱动软件。

## 目标

状态模式的主要目标是开发 FSM，如下所示：

+   当一些内部事物发生变化时，拥有一种可以改变自身行为的类型

+   可以通过添加更多状态并重新路由它们的输出状态轻松升级模型复杂的图形和管道

## 一个小猜数字游戏

我们将开发一个非常简单的使用 FSM 的游戏。这个游戏是一个猜数字游戏。想法很简单——我们将不得不猜出 0 到 10 之间的某个数字，我们只有几次尝试，否则就会输掉。

我们将让玩家选择难度级别，询问用户在失去之前有多少次尝试。然后，我们将要求玩家输入正确的数字，并在他们猜不中或尝试次数达到零时继续询问。

## 验收标准

对于这个简单的游戏，我们有五个验收标准，基本上描述了游戏的机制：

1.  游戏将询问玩家在失去游戏之前有多少次尝试。

1.  要猜的数字必须在 0 到 10 之间。

1.  每当玩家输入一个要猜的数字时，重试次数就会减少一个。

1.  如果重试次数达到零且数字仍然不正确，游戏结束，玩家输了。

1.  如果玩家猜中数字，玩家获胜。

## 状态模式的实现

单元测试的想法在状态模式中非常简单，因此我们将花更多时间详细解释如何使用它的机制，这比通常更复杂一些。

首先，我们需要一个接口来表示不同的状态和一个游戏上下文来存储状态之间的信息。对于这个游戏，上下文需要存储重试次数，用户是否已经赢得游戏，要猜的秘密数字和当前状态。状态将有一个`executeState`方法，该方法接受这些上下文之一，并在游戏结束时返回`true`，否则返回`false`：

```go
type GameState interface { 
  executeState(*GameContext) bool 
} 

type GameContext struct { 
  SecretNumber int 
  Retries int 
  Won bool 
  Next GameState 
} 

```

如*验收标准 1*中所述，玩家必须能够输入他们想要的重试次数。这将通过一个名为`StartState`的状态来实现。此外，`StartState`结构必须在玩家之前设置上下文的初始值：

```go
type StartState struct{} 
func(s *StartState) executeState(c *GameContext) bool { 
  c.Next = &AskState{} 

  rand.Seed(time.Now().UnixNano()) 
  c.SecretNumber = rand.Intn(10) 

  fmt.Println("Introduce a number a number of retries to set the difficulty:") 
  fmt.Fscanf(os.Stdin, "%d\n", &c.Retries) 

  return true 
} 

```

首先，`StartState`结构实现了`GameState`结构，因为它在其结构上具有`executeState(*Context)`方法，返回布尔类型。在这个状态的开始，它设置了执行完这个状态后唯一可能的状态--`AskState`状态。`AskState`结构尚未声明，但它将是我们询问玩家猜数字的状态。

在接下来的两行中，我们使用 Go 的`Rand`包生成一个随机数。在第一行中，我们用当前时刻返回的`int64`类型数字来喂入随机生成器，因此我们确保每次执行都有一个随机的喂入（如果你在这里放一个常数，随机生成器也会生成相同的数字）。`rand.Intn(int)`方法返回 0 到指定数字之间的整数，因此我们满足了*验收标准 2*。

接下来，我们设置一个消息询问要设置的重试次数，然后使用`fmt.Fscanf`方法，一个强大的函数，您可以向其传递一个`io.Reader`（控制台的标准输入）、一个格式（数字）和一个接口来存储读取器的内容，在这种情况下是上下文的`Retries`字段。

最后，我们返回`true`告诉引擎游戏必须继续。让我们看看我们在函数开头使用的`AskState`结构：

```go
type AskState struct {} 
func (a *AskState) executeState(c *GameContext) bool{ 
  fmt.Printf("Introduce a number between 0 and 10, you have %d tries left\n", c.Retries) 

  var n int 
  fmt.Fscanf(os.Stdin, "%d", &n) 
  c.Retries = c.Retries - 1 

  if n == c.SecretNumber { 
    c.Won = true 
    c.Next = &FinishState{} 
  } 

  if c.Retries == 0 { 
    c.Next = &FinishState{} 
  } 

  return true 
} 

```

`AskState`结构也实现了`GameState`状态，你可能已经猜到了。这个状态从一个向玩家的消息开始，要求他们插入一个新的数字。在接下来的三行中，我们创建一个本地变量来存储玩家将要输入的数字的内容。我们再次使用`fmt.Fscanf`方法，就像我们在`StartState`结构中做的那样，来捕获玩家的输入并将其存储在变量`n`中。然后，我们的计数器中的重试次数减少了一个，所以我们必须在上下文的`Retries`字段中减去一个。

然后，有两个检查：一个检查用户是否输入了正确的数字，如果是，则上下文字段`Won`设置为`true`，下一个状态设置为`FinishState`结构（尚未声明）。

第二个检查是控制重试次数是否已经达到零，如果是，则不会让玩家再次要求输入数字，并直接将玩家发送到`FinishState`结构。毕竟，我们必须再次告诉游戏引擎游戏必须继续，通过在`executeState`方法中返回`true`。

最后，我们定义了`FinishState`结构。它控制游戏的退出状态，检查上下文对象中`Won`字段的内容：

```go
type FinishState struct{} 
func(f *FinishState) executeState(c *GameContext) bool { 
  if c.Won { 
    println("Congrats, you won") 
  }  
  else { 
    println("You lose") 
  } 
  return false 
} 

```

`TheFinishState`结构也通过在其结构中具有`executeState`方法来实现`GameState`状态。这里的想法非常简单——如果玩家赢了（这个字段之前在`AskState`结构中设置），`FinishState`结构将打印消息`恭喜，你赢了`。如果玩家没有赢（记住布尔变量的零值是`false`），`FinishState`将打印消息`你输了`。

在这种情况下，游戏可以被认为已经结束，所以我们返回`false`来表示游戏不应该继续。

我们只需要`main`方法来玩我们的游戏。

```go
func main() { 
  start := StartState{} 
  game := GameContext{ 
    Next:&start, 
  } 
  for game.Next.executeState(&game) {} 
} 

```

嗯，是的，它不能再简单了。游戏必须从`start`方法开始，尽管在未来游戏需要更多初始化的情况下，它可以更抽象地放在外面，但在我们的情况下没问题。然后，我们创建一个上下文，将`Next`状态设置为指向`start`变量的指针。因此，在游戏中将执行的第一个状态将是`StartState`状态。

`main`函数的最后一行有很多东西。我们创建了一个循环，里面没有任何语句。和任何循环一样，在条件不满足后它会继续循环。我们使用的条件是`GameStates`结构的返回值，在游戏未结束时为`true`。

所以，思路很简单：我们在上下文中执行状态，将上下文的指针传递给它。每个状态都返回`true`，直到游戏结束，`FinishState`结构将返回`false`。所以我们的循环将继续循环，等待`FinishState`结构发送的`false`条件来结束应用程序。

让我们再玩一次：

```go
go run state.go
Introduce a number a number of retries to set the difficulty:
5
Introduce a number between 0 and 10, you have 5 tries left
8
Introduce a number between 0 and 10, you have 4 tries left
2
Introduce a number between 0 and 10, you have 3 tries left
1
Introduce a number between 0 and 10, you have 2 tries left
3
Introduce a number between 0 and 10, you have 1 tries left
4
You lose

```

我们输了！我们把重试次数设为 5。然后我们继续插入数字，试图猜出秘密数字。我们输入了 8、2、1、3 和 4，但都不对。我甚至不知道正确的数字是多少；让我们来修复这个！

去到`FinishState`结构的定义并且改变那一行写着`You lose`的地方，用以下内容替换它：

```go
fmt.Printf("You lose. The correct number was: %d\n", c.SecretNumber) 

```

现在它会显示正确的数字。让我们再玩一次：

```go
go run state.go
Introduce a number a number of retries to set the difficulty:
3
Introduce a number between 0 and 10, you have 3 tries left
6
Introduce a number between 0 and 10, you have 2 tries left
2
Introduce a number between 0 and 10, you have 1 tries left
1
You lose. The correct number was: 9

```

这次我们把难度加大了，只设置了三次尝试……但我们又输了。我输入了 6、2 和 1，但正确的数字是 9。最后一次尝试：

```go
go run state.go
Introduce a number a number of retries to set the difficulty:
5
Introduce a number between 0 and 10, you have 5 tries left
3
Introduce a number between 0 and 10, you have 4 tries left
4
Introduce a number between 0 and 10, you have 3 tries left
5
Introduce a number between 0 and 10, you have 2 tries left
6
Congrats, you won

```

太好了！这次我们降低了难度，允许最多五次尝试，我们赢了！我们甚至还有一次尝试剩下，但我们在第四次尝试后猜中了数字，输入了 3、4、5。正确的数字是 6，这是我的第四次尝试。

## 一个赢的状态和一个输的状态

你是否意识到我们可以有一个赢和一个输的状态，而不是直接在`FinishState`结构中打印消息？这样我们可以，例如，在赢的部分检查一些假设的得分板，看看我们是否创造了记录。让我们重构我们的游戏。首先我们需要一个`WinState`和一个`LoseState`结构：

```go
type WinState struct{} 

func (w *WinState) executeState(c *GameContext) bool { 
  println("Congrats, you won") 

  return false 
} 

type LoseState struct{} 

func (l *LoseState) executeState(c *GameContext) bool { 
  fmt.Printf("You lose. The correct number was: %d\n", c.SecretNumber) 
  return false 
} 

```

这两个新状态没有什么新东西。它们包含了之前在`FinishState`状态中的相同消息，顺便说一句，必须修改为使用这些新状态：

```go
func (f *FinishState) executeState(c *GameContext) bool { 
  if c.Won { 
    c.Next = &WinState{} 
  } else { 
    c.Next = &LoseState{} 
  } 
  return true 
} 

```

现在，结束状态不再打印任何东西，而是将其委托给链中的下一个状态——如果用户赢了，则是`WinState`结构，如果没有，则是`LoseState`结构。记住，游戏现在不会在`FinishState`结构上结束，我们必须返回`true`而不是`false`来通知引擎必须继续执行链中的状态。

## 使用状态模式构建的游戏

你现在可能会想，你可以用新状态无限扩展这个游戏，这是真的。状态模式的威力不仅在于创建复杂的有限状态机的能力，还在于通过添加新状态和修改一些旧状态指向新状态而不影响有限状态机的其余部分来改进它的灵活性。

# 中介者设计模式

让我们继续使用中介者模式。顾名思义，它是一种将处于两种类型之间以交换信息的模式。但是，为什么我们会想要这种行为呢？让我们仔细看一下。

## 描述

任何设计模式的关键目标之一是避免对象之间的紧密耦合。这可以通过多种方式实现，正如我们已经看到的。

但是当应用程序增长很多时，特别有效的一种方法是中介者模式。中介者模式是一个很好的例子，它是每个程序员通常在不太考虑的情况下使用的模式。

中介者模式将充当两个对象之间交换通信的类型。这样，通信的对象不需要彼此了解，可以更自由地进行更改。维护对象提供什么信息的模式是中介者。

## 目标

如前所述，中介者模式的主要目标是松散耦合和封装。目标是：

+   为了提供两个必须相互通信的对象之间的松散耦合

+   通过将这些需求传递给中介者模式，减少特定类型的依赖量

## 一个计算器

对于中介者模式，我们将开发一个非常简单的算术计算器。你可能认为计算器如此简单，不需要任何模式。但我们会看到这并不完全正确。

我们的计算器只会执行两个非常简单的操作：求和和减法。

## 验收标准

谈论验收标准来定义一个计算器听起来相当有趣，但无论如何我们都要做：

1.  定义一个名为`Sum`的操作，它接受一个数字并将其加到另一个数字。

1.  定义一个名为`Subtract`的操作，它接受一个数字并将其减去另一个数字。

嗯，我不知道你怎么想，但在这个*复杂*的标准之后，我真的需要休息。那么为什么我们要这么定义呢？耐心点，你很快就会得到答案。

## 实现

我们必须直接跳到实现，因为我们无法测试求和是否正确（嗯，我们可以，但那样就是在测试 Go 是否写得正确！）。我们可以测试是否符合验收标准，但对于我们的例子来说有点过度了。

那么让我们从实现必要的类型开始：

```go
package main 

type One struct{} 
type Two struct{} 
type Three struct{} 
type Four struct{} 
type Five struct{} 
type Six struct{} 
type Seven struct{} 
type Eight struct{} 
type Nine struct{} 
type Zero struct{} 

```

嗯...这看起来相当尴尬。我们在 Go 中已经有数字类型来执行这些操作，我们不需要为每个数字都定义一个类型！

但让我们再继续一下这种疯狂的方法。让我们实现`One`结构：

```go
type One struct{} 

func (o *One) OnePlus(n interface{}) interface{} { 
  switch n.(type) { 
  case One: 
    return &Two{} 
  case Two: 
    return &Three{} 
  case Three: 
    return &Four{} 
  case Four: 
    return &Five{} 
  case Five: 
    return &Six{} 
  case Six: 
    return &Seven{} 
  case Seven: 
    return &Eight{} 
  case Eight: 
    return &Nine{} 
  case Nine: 
    return [2]interface{}{&One{}, &Zero{}} 
  default: 
    return fmt.Errorf("Number not found") 
  } 
} 

```

好吧，我就说到这里。这个实现有什么问题？这完全疯狂！为了进行求和而使每种可能的数字操作都变得太过了！特别是当我们有多于一位数时。

嗯，信不信由你，这就是今天许多软件通常设计的方式。一个对象使用两个或三个对象的小应用程序会增长，最终使用数十个对象。仅仅因为它隐藏在某些疯狂的地方，所以要简单地添加或删除应用程序中的类型变得非常困难。

那么在这个计算器中我们能做什么？使用一个中介者类型来解放所有情况：

```go
func Sum(a, b interface{}) interface{}{ 
  switch a := a.(type) { 
    case One: 
    switch b.(type) { 
      case One: 
        return &Two{} 
      case Two: 
        return &Three{} 
      default: 
        return fmt.Errorf("Number not found") 
    } 
    case Two: 
    switch b.(type) { 
      case One: 
        return &Three{} 
      case Two: 
        return &Four{} 
      default: 
      return fmt.Errorf("Number not found") 

    } 
    case int: 
    switch b := b.(type) { 
      case One: 
        return &Three{} 
      case Two: 
        return &Four{} 
      case int: 
        return a + b 
      default: 
      return fmt.Errorf("Number not found") 

    } 
    default: 
    return fmt.Errorf("Number not found") 
  } 
} 

```

我们只开发了一对数字来简化。`Sum`函数充当两个数字之间的中介者。首先它检查名为`a`的第一个数字的类型。然后，对于第一个数字的每种类型，它检查名为`b`的第二个数字的类型，并返回结果类型。

虽然解决方案现在看起来仍然非常疯狂，但唯一知道计算器中所有可能数字的是`Sum`函数。但仔细看，你会发现我们为`int`类型添加了一个类型情况。我们有`One`、`Two`和`int`情况。在`int`情况下，我们还有另一个`int`情况用于`b`数字。我们在这里做什么？如果两种类型都是`int`情况，我们可以返回它们的和。

你认为这样会有效吗？让我们写一个简单的`main`函数：

```go
func main(){ 
  fmt.Printf("%#v\n", Sum(One{}, Two{})) 
  fmt.Printf("%d\n", Sum(1,2)) 
} 

```

我们打印类型`One`和类型`Two`的总和。通过使用`"%#v"`格式，我们要求打印有关类型的信息。函数中的第二行使用`int`类型，并且我们还打印结果。这在控制台上产生以下输出：

```go
$go run mediator.go
&main.Three{}
7

```

不是很令人印象深刻，对吧？但是让我们思考一下。通过使用中介者模式，我们已经能够重构最初的计算器，在那里我们必须为每种类型定义每个操作，转换为中介者模式的`Sum`函数。

好处在于，由于中介者模式的存在，我们已经能够开始将整数作为计算器的值使用。我们刚刚通过添加两个整数定义了最简单的示例，但我们也可以使用整数和`type`来做同样的事情：

```go
  case One: 
    switch b := b.(type) { 
    case One: 
      return &Two{} 
    case Two: 
      return &Three{} 
    case int: 
      return b+1 
    default: 
      return fmt.Errorf("Number not found") 
    } 

```

通过这个小修改，我们现在可以使用类型`One`和`int`作为数字`b`。如果我们继续在中介者模式上工作，我们可以在类型之间实现很大的灵活性，而无需实现它们之间的每种可能操作，从而产生紧密耦合。

我们将在主函数中添加一个新的`Sum`方法，以查看其运行情况：

```go
func main(){ 
  fmt.Printf("%#v\n", Sum(One{}, Two{})) 
  fmt.Printf("%d\n", Sum(1,2)) 
 fmt.Printf("%d\n", Sum(One{},2)) 
} 
$go run mediator.go&main.Three{}33

```

很好。中介者模式负责了解可能的类型并返回最适合我们情况的类型，即整数。现在我们可以继续扩展这个`Sum`函数，直到完全摆脱使用我们定义的数值类型。

## 使用中介者解耦两种类型

我们进行了一个颠覆性的示例，试图超越传统思维，深入思考中介者模式。应用程序中实体之间的紧密耦合可能在未来变得非常复杂，并且如果需要进行更复杂的重构，则可能更加困难。

只需记住，中介者模式的作用是作为两种不相互了解的类型之间的管理类型，以便您可以获取其中一种类型而不影响另一种类型，并以更轻松和便捷的方式替换类型。

# 观察者设计模式

我们将用我最喜欢的*四人帮*设计模式之一结束，即观察者模式，也称为发布/订阅或发布/监听器。通过状态模式，我们定义了我们的第一个事件驱动架构，但是通过观察者模式，我们将真正达到一个新的抽象层次。

## 描述

观察者模式背后的思想很简单--订阅某个事件，该事件将触发许多订阅类型上的某些行为。为什么这么有趣？因为我们将一个事件与其可能的处理程序解耦。

例如，想象一个登录按钮。我们可以编写代码，当用户点击按钮时，按钮颜色会改变，执行一个操作，并在后台执行表单检查。但是通过观察者模式，更改颜色的类型将订阅按钮点击事件。检查表单的类型和执行操作的类型也将订阅此事件。

## 目标

观察者模式特别有用，可以在一个事件上触发多个操作。当您事先不知道有多少操作会在事件之后执行，或者有可能操作的数量将来会增加时，它也特别有用。总之，执行以下操作：

+   提供一个事件驱动的架构，其中一个事件可以触发一个或多个操作

+   将执行的操作与触发它们的事件解耦

+   提供触发相同操作的多个事件

## 通知者

我们将开发最简单的应用程序，以充分理解观察者模式的根源。我们将创建一个`Publisher`结构，它是触发事件的结构，因此必须接受新的观察者，并在必要时删除它们。当触发`Publisher`结构时，它必须通知所有观察者有关关联数据的新事件。

## 验收标准

需求必须告诉我们有一些类型会触发一个或多个操作的某种方法：

1.  我们必须有一个带有`NotifyObservers`方法的发布者，该方法接受消息作为参数并触发订阅的每个观察者上的`Notify`方法。

1.  我们必须有一个方法向发布者添加新的订阅者。

1.  我们必须有一个方法从发布者中删除新的订阅者。

## 单元测试

也许你已经意识到，我们的要求几乎完全定义了`Publisher`类型。这是因为观察者执行的操作对观察者模式来说是无关紧要的。它应该只执行一个动作，即`Notify`方法，在这种情况下，一个或多个类型将实现。因此，让我们为此模式定义唯一的接口：

```go
type Observer interface { 
  Notify(string) 
} 

```

`Observer`接口有一个`Notify`方法，它接受一个`string`类型，其中包含要传播的消息。它不需要返回任何东西，但是当调用`Publisher`结构的`publish`方法时，我们可以返回一个错误，以便检查是否已经到达了所有观察者。

为了测试所有的验收标准，我们只需要一个名为`Publisher`的结构，其中包含三种方法：

```go
type Publisher struct { 
  ObserversList []Observer 
} 

func (s *Publisher) AddObserver(o Observer) {} 

func (s *Publisher) RemoveObserver(o Observer) {} 

func (s *Publisher) NotifyObservers(m string) {} 

```

`Publisher`结构将订阅的观察者列表存储在名为`ObserversList`的切片字段中。然后它具有接受标准的三种方法--`AddObserver`方法用于向发布者订阅新的观察者，`RemoveObserver`方法用于取消订阅观察者，以及`NotifyObservers`方法，其中包含一个作为我们想要在所有观察者之间传播的消息的字符串。

有了这三种方法，我们必须设置一个根测试来配置`Publisher`和三个子测试来测试每种方法。我们还需要定义一个实现`Observer`接口的测试类型结构。这个结构将被称为`TestObserver`：

```go
type TestObserver struct { 
  ID      int 
  Message string 
} 
func (p *TestObserver) Notify(m string) { 
  fmt.Printf("Observer %d: message '%s' received \n", p.ID, m) 
  p.Message = m 
} 

```

`TestObserver`结构通过在其结构中定义`Notify(string)`方法来实现观察者模式。在这种情况下，它打印接收到的消息以及自己的观察者 ID。然后，它将消息存储在其`Message`字段中。这使我们可以稍后检查`Message`字段的内容是否符合预期。请记住，也可以通过传递`testing.T`指针和预期消息并在`TestObserver`结构内部进行检查来完成。

现在我们可以设置`Publisher`结构来执行这三个测试。我们将创建`TestObserver`结构的三个实例：

```go
func TestSubject(t *testing.T) { 
  testObserver1 := &TestObserver{1, ""} 
  testObserver2 := &TestObserver{2, ""} 
  testObserver3 := &TestObserver{3, ""} 
  publisher := Publisher{} 

```

我们为每个观察者分配了不同的 ID，以便稍后可以看到它们每个人都打印了预期的消息。然后，我们通过在`Publisher`结构上调用`AddObserver`方法来添加观察者。

让我们编写一个`AddObserver`测试，它必须将新的观察者添加到`Publisher`结构的`ObserversList`字段中：

```go
  t.Run("AddObserver", func(t *testing.T) { 
    publisher.AddObserver(testObserver1) 
    publisher.AddObserver(testObserver2) 
    publisher.AddObserver(testObserver3) 

    if len(publisher.ObserversList) != 3 { 
      t.Fail() 
    } 
  }) 

```

我们已经向`Publisher`结构添加了三个观察者，因此切片的长度必须为 3。如果不是 3，测试将失败。

`RemoveObserver`测试将获取 ID 为 2 的观察者并将其从列表中删除：

```go
  t.Run("RemoveObserver", func(t *testing.T) { 
    publisher.RemoveObserver(testObserver2) 

    if len(publisher.ObserversList) != 2 { 
      t.Errorf("The size of the observer list is not the " + 
        "expected. 3 != %d\n", len(publisher.ObserversList)) 
    } 

    for _, observer := range publisher.ObserversList { 
      testObserver, ok := observer.(TestObserver) 
      if !ok {  
        t.Fail() 
      } 

      if testObserver.ID == 2 { 
        t.Fail() 
      } 
    } 
  }) 

```

删除第二个观察者后，`Publisher`结构的长度现在必须为 2。我们还检查剩下的观察者中没有一个的`ID`为 2，因为它必须被移除。

测试的最后一个方法是`Notify`方法。使用`Notify`方法时，所有`TestObserver`结构的实例都必须将它们的`Message`字段从空更改为传递的消息（在本例中为`Hello World!`）。首先，我们将检查在调用`NotifyObservers`测试之前所有的`Message`字段是否实际上都是空的：

```go
t.Run("Notify", func(t *testing.T) { 
    for _, observer := range publisher.ObserversList { 
      printObserver, ok := observer.(*TestObserver) 
      if !ok { 
        t.Fail() 
        break 
      } 

      if printObserver.Message != "" { 
        t.Errorf("The observer's Message field weren't " + "  empty: %s\n", printObserver.Message) 
      } 
    } 

```

使用`for`语句，我们正在迭代`publisher`实例中的`ObserversList`字段。我们需要将指针从观察者转换为`TestObserver`结构的指针，并检查转换是否已正确完成。然后，我们检查`Message`字段实际上是否为空。

下一步是创建要发送的消息--在本例中，它将是`"Hello World!"`，然后将此消息传递给`NotifyObservers`方法，以通知列表上的每个观察者（目前只有观察者 1 和 3）：

```go
    ... 
    message := "Hello World!" 
    publisher.NotifyObservers(message) 

    for _, observer := range publisher.ObserversList { 
      printObserver, ok := observer.(*TestObserver) 
      if !ok { 
        t.Fail() 
        break 
      } 

      if printObserver.Message != message { 
        t.Errorf("Expected message on observer %d was " + 
          "not expected: '%s' != '%s'\n", printObserver.ID, 
          printObserver.Message, message) 
      } 
    } 
  }) 
} 

```

调用`NotifyObservers`方法后，`ObserversList`字段中的每个`TestObserver`测试必须在其`Message`字段中存储`"Hello World!"`消息。同样，我们使用`for`循环来遍历`ObserversList`字段中的每个观察者，并将每个类型转换为`TestObserver`测试（请记住，`TestObserver`结构没有任何字段，因为它是一个接口）。我们可以通过向`Observer`实例添加一个新的`Message()`方法并在`TestObserver`结构中实现它来避免类型转换，以返回`Message`字段的内容。这两种方法都是有效的。一旦我们将类型转换为`TestObserver`方法调用`printObserver`变量作为局部变量，我们检查`ObserversList`结构中的每个实例是否在其`Message`字段中存储了字符串`"Hello World!"`。

是时候运行测试了，必须全部失败以检查它们在后续实现中的有效性：

```go
go test -v  
=== RUN   TestSubject 
=== RUN   TestSubject/AddObserver 
=== RUN   TestSubject/RemoveObserver 
=== RUN   TestSubject/Notify 
--- FAIL: TestSubject (0.00s) 
    --- FAIL: TestSubject/AddObserver (0.00s) 
    --- FAIL: TestSubject/RemoveObserver (0.00s) 
        observer_test.go:40: The size of the observer list is not the expected. 3 != 0 
    --- PASS: TestSubject/Notify (0.00s) 
FAIL 
exit status 1 
FAIL

```

有些地方不如预期。如果我们还没有实现函数，`Notify`方法是如何通过测试的？再看一下`Notify`方法的测试。测试遍历`ObserversList`结构，并且每个`Fail`调用都在此`for`循环内。如果列表为空，它将不会进行迭代，因此不会执行任何`Fail`调用。

让我们通过在`Notify`测试的开头添加一个小的非空列表检查来解决这个问题：

```go
  if len(publisher.ObserversList) == 0 { 
      t.Errorf("The list is empty. Nothing to test\n") 
  } 

```

我们将重新运行测试，看看`TestSubject/Notify`方法是否已经失败：

```go
go test -v
=== RUN   TestSubject
=== RUN   TestSubject/AddObserver
=== RUN   TestSubject/RemoveObserver
=== RUN   TestSubject/Notify
--- FAIL: TestSubject (0.00s)
 --- FAIL: TestSubject/AddObserver (0.00s)
 --- FAIL: TestSubject/RemoveObserver (0.00s)
 observer_test.go:40: The size of the observer list is not the expected. 3 != 0
 --- FAIL: TestSubject/Notify (0.00s)
 observer_test.go:58: The list is empty. Nothing to test
FAIL
exit status 1
FAIL

```

很好，它们全部失败了，现在我们对测试有了一些保证。我们可以继续实现。

## 实施

我们的实现只是定义`AddObserver`、`RemoveObserver`和`NotifyObservers`方法：

```go
func (s *Publisher) AddObserver(o Observer) { 
  s.ObserversList = append(s.ObserversList, o) 
} 

```

`AddObserver`方法通过将指针附加到当前指针列表来将`Observer`实例添加到`ObserversList`结构中。这很容易。`AddObserver`测试现在必须通过（但其他测试不通过，否则我们可能做错了什么）：

```go
go test -v
=== RUN   TestSubject
=== RUN   TestSubject/AddObserver
=== RUN   TestSubject/RemoveObserver
=== RUN   TestSubject/Notify
--- FAIL: TestSubject (0.00s)
 --- PASS: TestSubject/AddObserver (0.00s)
 --- FAIL: TestSubject/RemoveObserver (0.00s)
 observer_test.go:40: The size of the observer list is not the expected. 3 != 3
 --- FAIL: TestSubject/Notify (0.00s)
 observer_test.go:87: Expected message on observer 1 was not expected: 'default' != 'Hello World!'
 observer_test.go:87: Expected message on observer 2 was not expected: 'default' != 'Hello World!'
 observer_test.go:87: Expected message on observer 3 was not expected: 'default' != 'Hello World!'
FAIL
exit status 1
FAIL

```

很好。只有`AddObserver`方法通过了测试，所以我们现在可以继续进行`RemoveObserver`方法：

```go
func (s *Publisher) RemoveObserver(o Observer) { 
  var indexToRemove int 

  for i, observer := range s.ObserversList { 
    if observer == o { 
      indexToRemove = i 
      break 
    } 
  } 

  s.ObserversList = append(s.ObserversList[:indexToRemove], s.ObserversList[indexToRemove+1:]...) 
} 

```

`RemoveObserver`方法将遍历`ObserversList`结构中的每个元素，将`Observer`对象的`o`变量与列表中存储的对象进行比较。如果找到匹配项，它将保存在本地变量`indexToRemove`中，并停止迭代。在 Go 中删除切片的索引有点棘手：

1.  首先，我们需要使用切片索引来返回一个新的切片，其中包含从切片开头到我们想要移除的索引（不包括）的每个对象。

1.  然后，我们从要删除的索引（不包括）到切片中的最后一个对象获取另一个切片

1.  最后，我们将前两个新切片合并成一个新的切片（使用`append`函数）

例如，在一个从 1 到 10 的列表中，我们想要移除数字 5，我们必须创建一个新的切片，将从 1 到 4 的切片和从 6 到 10 的切片连接起来。

这个索引移除是使用`append`函数完成的，因为我们实际上是将两个列表连接在一起。仔细看一下`append`函数第二个参数末尾的三个点。`append`函数将一个元素（第二个参数）添加到一个切片（第一个参数），但我们想要添加整个列表。这可以通过使用三个点来实现，它们的作用类似于*继续添加元素，直到完成第二个数组*。

好的，现在让我们运行这个测试：

```go
go test -v           
=== RUN   TestSubject 
=== RUN   TestSubject/AddObserver 
=== RUN   TestSubject/RemoveObserver 
=== RUN   TestSubject/Notify 
--- FAIL: TestSubject (0.00s) 
    --- PASS: TestSubject/AddObserver (0.00s) 
    --- PASS: TestSubject/RemoveObserver (0.00s) 
    --- FAIL: TestSubject/Notify (0.00s) 
        observer_test.go:87: Expected message on observer 1 was not expected: 'default' != 'Hello World!' 
        observer_test.go:87: Expected message on observer 3 was not expected: 'default' != 'Hello World!' 
FAIL 
exit status 1 
FAIL 

```

我们继续在正确的路径上。`RemoveObserver`测试已经修复，而没有修复其他任何东西。现在我们必须通过定义`NotifyObservers`方法来完成我们的实现：

```go
func (s *Publisher) NotifyObservers(m string) { 
  fmt.Printf("Publisher received message '%s' to notify observers\n", m) 
  for _, observer := range s.ObserversList { 
    observer.Notify(m) 
  } 
} 

```

`NotifyObservers`方法非常简单，因为它在控制台上打印一条消息，宣布特定消息将传递给“观察者”。之后，我们使用 for 循环遍历`ObserversList`结构，并通过传递参数`m`执行每个`Notify(string)`方法。执行完毕后，所有观察者必须在其`Message`字段中存储消息`Hello World!`。让我们通过运行测试来看看这是否成立：

```go
go test -v 
=== RUN   TestSubject 
=== RUN   TestSubject/AddObserver 
=== RUN   TestSubject/RemoveObserver 
=== RUN   TestSubject/Notify 
Publisher received message 'Hello World!' to notify observers 
Observer 1: message 'Hello World!' received  
Observer 3: message 'Hello World!' received  
--- PASS: TestSubject (0.00s) 
    --- PASS: TestSubject/AddObserver (0.00s) 
    --- PASS: TestSubject/RemoveObserver (0.00s) 
    --- PASS: TestSubject/Notify (0.00s) 
PASS 
ok

```

太棒了！我们还可以在控制台上看到“发布者”和“观察者”类型的输出。 “发布者”结构打印以下消息：

```go
hey! I have received the message  'Hello World!' and I'm going to pass the same message to the observers 
```

之后，所有观察者按如下方式打印各自的消息：

```go
hey, I'm observer 1 and I have received the message 'Hello World!'
```

第三个观察者也是如此。

## 总结

我们已经利用状态模式和观察者模式解锁了事件驱动架构的力量。现在，您可以在应用程序中真正执行异步算法和操作，以响应系统中的事件。

观察者模式通常用于 UI。Android 编程中充满了观察者模式，以便 Android SDK 可以将操作委托给创建应用程序的程序员。
