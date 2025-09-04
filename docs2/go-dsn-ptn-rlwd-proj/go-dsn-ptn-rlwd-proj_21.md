# 第七章。行为模式 - 访问者、状态、中介者和观察者设计模式

这是关于行为模式的最后一章，同时也结束了这本书关于 Go 语言中常见、知名设计模式的部分。

在本章中，我们将探讨三个更多设计模式。访问者模式在你想从一组对象中抽象出某些功能时非常有用。

状态通常用于构建**有限状态机**（**FSM**），在本节中，我们将开发一个小型的*猜数字*游戏。

最后，观察者模式在事件驱动架构中很常见，并且在微服务世界中再次获得了大量关注。

在本章之后，我们需要在深入并发及其在设计模式中的优势（以及复杂性）之前，对常见设计模式感到非常熟悉。

# 访问者设计模式

在下一个设计模式中，我们将将对象类型的某些逻辑委托给一个外部类型，称为访问者，它将访问我们的对象以执行操作。

## 描述

在访问者设计模式中，我们试图将处理特定对象所需的逻辑与对象本身分离。因此，我们可以有许多不同的访问者对特定类型执行某些操作。

例如，假设我们有一个写入控制台的日志记录器。我们可以使记录器“可访问”，这样你就可以在每个日志前添加任何文本。我们可以编写一个访问者模式，将日期、时间和主机名添加到对象中存储的字段。

## 目标

在行为设计模式中，我们主要处理算法。访问者模式也不例外。我们试图实现的目标如下：

+   将某些类型的算法与其在另一个类型中的实现分离

+   通过使用几乎没有任何逻辑的某些类型来提高它们的灵活性，这样所有新的功能都可以添加，而无需更改对象结构

+   为了修复一个会破坏类型开放/封闭原则的结构或行为

你可能想知道开放/封闭原则是什么。在计算机科学中，开放/封闭原则指出：*实体应该对扩展开放，但对修改封闭*。这种简单的状态有很多含义，允许构建更易于维护的软件，并且更不容易出错。访问者模式帮助我们将一些经常变化的算法从需要“稳定”的类型中委托给一个可以经常更改而不影响原始类型的外部类型。

## 日志追加器

我们将以一个简单的日志追加器为例，来开发一个访问者模式的示例。遵循我们在前几章中采用的方法，我们将从一个极其简单的例子开始，以便清楚地理解访问者设计模式是如何工作的，然后再转向一个更复杂的例子。我们之前也开发过类似的例子，修改文本，但方式略有不同。

对于这个特定的例子，我们将创建一个访问者，它将向它“访问”的类型追加不同的信息。

## 接受标准

为了有效地使用访问者设计模式，我们必须有两个角色——访问者和可访问者。`访问者`是将在`可访问者`类型内执行操作的类型。因此，`可访问者`接口实现有一个与`访问者`类型分离的算法：

1.  我们需要两个消息记录器：`MessageA`和`MessageB`，它们将在消息前分别打印`A:`或`B:`。

1.  我们需要一个能够修改要打印的消息的访问者。它将分别向它们追加文本“Visited A”或“Visited B”。

## 单元测试

正如我们之前提到的，我们需要为`访问者`和`可访问者`接口提供一个角色。它们将是接口。我们还需要`MessageA`和`MessageB`结构体：

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

`MessageA`和`MessageB`结构体类型都有一个`Msg`字段来存储它们将要打印的文本。默认情况下，输出`io.Writer`将实现`os.Stdout`接口，或者一个新的`io.Writer`接口，就像我们将用它来检查内容是否正确的那样。

`访问者`接口有一个`Visit`方法，对应于`可访问者`接口的`MessageA`和`MessageB`类型。`可访问者`接口有一个名为`Accept(Visitor)`的方法，它将执行解耦算法。

如前所述，我们将创建一个实现`io.Writer`包的类型，以便我们可以在测试中使用它：

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

`TestHelper`结构体实现了`io.Writer`接口。它的功能相当简单；它将写入的字节存储在`Received`字段上。稍后我们可以检查`Received`的内容以测试我们的预期值。

我们将只编写一个测试来检查代码的整体正确性。在这个测试中，我们将编写两个子测试：一个用于`MessageA`类型，一个用于`MessageB`类型：

```go
func Test_Overall(t *testing.T) { 
  testHelper := &TestHelper{} 
  visitor := &MessageVisitor{} 
  ... 
} 

```

我们将在每个消息类型的每个测试中使用一个`TestHelper`结构体和一个`MessageVisitor`结构体。首先，我们将测试`MessageA`类型：

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

这是完整的第一个测试。我们创建了`MessageA`结构体，给它`Msg`字段一个值`Hello World`，并提供了我们在测试开始时创建的`TestHelper`的指针。然后，我们执行它的`Accept`方法。在`MessageA`结构体上的`Accept(Visitor)`方法内部，执行了`VisitA(*MessageA)`方法来更改`Msg`字段的内容（这就是为什么我们传递了`VisitA`方法的指针，如果没有指针，内容将不会持久化）。

为了测试`访问者`类型是否在`Accept`方法中完成了其工作，我们必须稍后在`MessageA`类型上调用`Print()`方法。这样，`MessageA`结构体必须将`Msg`字段的内容写入提供的`io.Writer`接口（我们的`TestHelper`）。

测试的最后部分是检查。根据 *验收标准 2* 的描述，`MessageA` 类型的输出文本必须以文本 `A:` 开头，存储的消息和文本 `"(Visited)"` 在末尾。所以，对于 `MessageA` 类型，预期的文本必须是 `"A: Hello World (Visited)"`，这就是我们在 `if` 部分所做的检查。

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

事实上，我们只是将类型从 `MessageA` 改为 `MessageB`，现在期望的文本是 `"B: Hello World (Visited B)"`。`Msg` 字段也是 `"Hello World"`，我们同样使用了 `TestHelper` 类型。

我们仍然缺少接口的正确实现来编译代码和运行测试。`MessageA` 和 `MessageB` 结构体必须实现 `Accept(Visitor)` 方法：

```go
func (m *MessageA) Accept(v Visitor) { 
  //Do nothing 
} 

func (m *MessageB) Accept(v Visitor) { 
  //Do nothing 
} 

```

我们需要实现 `Visitor` 接口上声明的 `VisitA(*MessageA)` 和 `VisitB(*MessageB)` 方法。`MessageVisitor` 接口是必须实现它们的类型：

```go
func (mf *MessageVisitor) VisitA(m *MessageA){ 
  //Do nothing 
} 
func (mf *MessageVisitor) VisitB(m *MessageB){ 
  //Do nothing 
} 

```

最后，我们将为每种消息类型创建一个 `Print()` 方法。这是我们用来测试每个类型 `Msg` 字段内容的工具：

```go
func (m *MessageA) Print(){ 
  //Do nothing 
} 

func (m *MessageB) Print(){ 
  //Do nothing 
} 

```

现在我们可以运行测试来真正检查它们是否已经失败：

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

测试的输出很清晰。预期的消息是不正确的，因为内容是空的。是时候创建实现啦。

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

其功能相当直接--`fmt.Sprintf` 方法返回一个格式化的字符串，包含 `m.Msg` 的实际内容、一个空格和消息 `Visited`。这个字符串将被存储在 `Msg` 字段，覆盖之前的内 容。

现在，我们将为必须执行相应访问者的每种消息类型开发 `Accept` 方法：

```go
func (m *MessageA) Accept(v Visitor) { 
  v.VisitA(m) 
} 

func (m *MessageB) Accept(v Visitor) { 
  v.VisitB(m) 
} 

```

这段小代码有一些含义。在两种情况下，我们都在使用 `Visitor`，在我们的例子中，它正好与 `MessageVisitor` 接口相同，但它们可能完全不同。关键是理解访问者模式在其 `Visit` 方法中执行算法，该算法处理 `Visitable` 对象。`Visitor` 可以做什么？在这个例子中，它改变了 `Visitable` 对象，但它也可以简单地从它那里获取信息。例如，我们可以有一个 `Person` 类型，有很多字段：姓名、姓氏、年龄、地址、城市、邮政编码等等。我们可以编写一个访问者来从一个人那里获取唯一的字符串（姓名和姓氏），一个访问者来获取应用程序不同部分的地址信息，等等。

最后，是 `Print()` 方法，它将帮助我们测试类型。我们之前提到，它默认必须打印到 `Stdout` 调用：

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

它首先检查 `Output` 字段的内容，以分配 `os.Stdout` 调用的输出，以防它是空的。在我们的测试中，我们在这里存储了一个指向我们的 `TestHelper` 类型的指针，所以这行代码在我们的测试中永远不会被执行。最后，每个消息类型都会将存储在 `Msg` 字段中的完整消息打印到 `Output` 字段。这是通过使用 `Fprintf` 方法完成的，该方法将 `io.Writer` 包作为第一个参数，将格式化文本作为后续参数。

我们现在的实现已经完成，我们可以再次运行测试，看看它们现在是否都通过了：

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

一切正常！访问者模式完美地完成了它的任务，并且在调用它们的 `Visit` 方法之后，消息内容被修改了。这里非常重要的一点是，我们可以为这两个结构体，`MessageA` 和 `MessageB`，添加更多功能，而不改变它们的类型。我们只需创建一个新的访问者类型，它可以在 `Visitable` 上做所有事情，例如，我们可以创建一个 `Visitor` 来添加一个打印 `Msg` 字段内容的方法：

```go
type MsgFieldVisitorPrinter struct {} 

func (mf *MsgFieldVisitorPrinter) VisitA(m *MessageA){ 
  fmt.Printf(m.Msg) 
} 
func (mf *MsgFieldVisitorPrinter) VisitB(m *MessageB){ 
  fmt.Printf(m.Msg) 
} 

```

我们只是为这两种类型添加了一些功能，而没有改变它们的内部内容！这就是访问者设计模式的力量。

## 另一个示例

我们将开发第二个示例，这个示例稍微复杂一些。在这种情况下，我们将模拟一个在线商店，其中包含一些产品。产品将具有普通类型，只有字段，我们将创建几个访问者来处理这些产品。

首先，我们将开发接口。`ProductInfoRetriever` 类型有一个方法可以获取产品的价格和名称。`Visitor` 接口，就像之前一样，有一个 `Visit` 方法，它接受 `ProductInfoRetriever` 类型。最后，`Visitable` 接口完全相同；它有一个 `Accept` 方法，该方法接受一个 `Visitor` 类型作为参数：

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

在线商店的所有产品都必须实现 `ProductInfoRetriever` 类型。此外，大多数产品将有一些公共字段，例如名称或价格（在 `ProductInfoRetriever` 接口中定义的）。我们创建了 `Product` 类型，实现了 `ProductInfoRetriever` 和 `Visitable` 接口，并将其嵌入到每个产品中：

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

现在我们有一个非常通用的 `Product` 类型，它可以存储商店几乎任何产品的信息。例如，我们可能有一个 `Rice` 和 `Pasta` 产品：

```go
type Rice struct { 
  Product 
} 

type Pasta struct { 
  Product 
} 

```

每个都嵌入了 `Product` 类型。现在我们需要创建几个 `Visitors` 接口，一个用于计算所有产品的价格总和，另一个用于打印每个产品的名称：

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

`PriceVisitor` 结构体接受作为参数传递的 `ProductInfoRetriever` 类型的 `Price` 变量的值，并将其添加到 `Sum` 字段。`NamePrinter` 结构体存储作为参数传递的 `ProductInfoRetriever` 类型的名称，并将其追加到新的 `ProductList` 字段行。

现在是 `main` 函数的时间：

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

我们创建了一个包含两个`Visitable`对象的切片：一个`Rice`和一个`Pasta`类型的对象，具有一些任意的名称。然后我们使用`PriceVisitor`实例作为参数对它们中的每一个进行迭代。在 for 循环结束后，我们打印出总价。最后，我们使用`NamePrinter`重复此操作并打印出结果`ProductList`。这个`main`函数的输出如下：

```go
go run visitor.go
Total: 72.000000
Product list:
-------------
Some pasta
Some rice

```

好的，这是一个访问者模式的良好示例，但是……如果对产品有特殊考虑怎么办？例如，如果我们需要将 20 加到冰箱类型的总价上怎么办？好的，让我们编写`Fridge`结构：

```go
type Fridge struct { 
  Product 
} 

```

这里的想法是只是重写`GetPrice()`方法以返回产品的价格加上 20：

```go
type Fridge struct { 
  Product 
} 

func (f *Fridge) GetPrice() float32 { 
  return f.Product.Price + 20 
} 

```

不幸的是，这对我们的示例还不够。`Fridge`结构不是`Visitable`类型。`Product`结构是`Visitable`类型，而`Fridge`结构包含一个`Product`结构体，但正如我们在前面的章节中提到的，嵌套第二个类型的类型不能被认为是后者类型，即使它具有所有字段和方法。解决方案是实现`Accept(Visitor)`方法，使其可以被认为是`Visitable`：

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

让我们重写`main`函数，以添加这个新的`Fridge`产品到切片中：

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

其他一切继续相同。运行这个新的`main`函数会产生以下输出：

```go
$ go run visitor.go
Total: 142.000000
Product list:
-------------
A fridge
Some pasta
Some rice

```

如预期的那样，总价现在更高了，输出的是大米（32）、意大利面（40）和冰箱（产品 50 加上运输 20，所以 70）的总和。我们可以永远向这些产品添加访问者，但理念是清晰的——我们将一些算法从类型中解耦到了访问者中。

## 访问者来拯救！

我们已经看到了一个强大的抽象，可以将新算法添加到某些类型中。然而，由于 Go 中缺少重载，这个模式在某些方面可能有限制（我们在第一个示例中看到了这一点，当时我们必须创建`VisitA`和`VisitB`实现）。在第二个示例中，我们没有处理这个限制，因为我们使用了`Visitor`结构体的`Visit`方法接口，但我们只使用了一种类型的访问者（`ProductInfoRetriever`），如果我们为第二种类型实现`Visit`方法，我们也会遇到同样的问题，这是原始*四人帮*设计模式的一个目标。

# 状态设计模式

状态模式与 FSM 直接相关。在非常简单的术语中，FSM 是具有一个或多个状态并在它们之间移动以执行某些行为的东西。让我们看看状态模式如何帮助我们定义 FSM。

## 描述

开关灯是有限状态机（FSM）的一个常见示例。它有两个状态——开和关。一个状态可以转换到另一个状态，反之亦然。状态模式的工作方式与此类似。我们有一个`State`接口和每个我们想要实现的状态的实现。通常还有一个上下文，它持有状态之间的跨信息。

使用有限状态机（FSM），我们可以通过在状态之间分割它们的范围来实现非常复杂的行为。这样，我们可以根据任何类型的输入来建模执行管道，或者创建响应特定事件的特定方式的基于事件的软件。

## 目标

国家模式的主要目标是为了开发有限状态机（FSM），具体如下：

+   要有一个类型，当某些内部事物发生变化时改变其自身的行为

+   通过添加更多状态并重新路由它们的输出状态，可以轻松升级复杂的图和管道模型

## 一个简单的猜数字游戏

我们将开发一个非常简单的游戏，该游戏使用有限状态机（FSM）。这个游戏是一个数字猜测游戏。想法很简单——我们将在 0 到 10 之间猜测一个数字，我们只有几次尝试，否则就会失败。

我们将让玩家通过询问他们在游戏失败前有多少次尝试机会来选择难度级别。然后，我们将询问玩家正确的数字，如果他们猜不对或者尝试次数达到零，我们将继续询问。

## 验收标准

对于这个简单的游戏，我们有五个验收标准，基本上描述了游戏的机制：

1.  游戏将询问玩家在游戏失败前将有多少次尝试机会。

1.  要猜测的数字必须在 0 到 10 之间。

1.  每次玩家输入一个猜测数字时，尝试次数就会减少一次。

1.  如果尝试次数达到零而数字仍然不正确，游戏结束，玩家失败。

1.  如果玩家猜对了数字，玩家获胜。

## 状态模式的实现

在状态模式中，单元测试的想法非常直接，因此我们将花更多的时间详细解释如何使用它，这比通常要复杂一些。

首先，我们需要一个接口来表示不同的状态，以及一个游戏上下文来存储状态之间的信息。对于这个游戏，上下文需要存储重试次数、用户是否获胜、要猜测的秘密数字以及当前状态。状态将有一个 `executeState` 方法，它接受这些上下文之一，如果游戏结束则返回 `true`，否则返回 `false`：

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

如 *验收标准 1* 所述，玩家必须能够输入他们想要的尝试次数。这将通过一个名为 `StartState` 的状态来实现。此外，`StartState` 结构体必须在玩家之前准备游戏，将上下文设置为初始值：

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

首先，`StartState` 结构体实现了 `GameState` 结构体，因为它在其结构体上有一个布尔类型的 `executeState(*Context)` 方法。在这个状态开始时，它设置执行此状态后唯一可能的状态——`AskState` 状态。`AskState` 结构体尚未声明，但将是我们询问玩家猜测数字的状态。

在接下来的两行中，我们使用 Go 的 `Rand` 包来生成随机数。在第一行，我们将随机生成器与当前时刻返回的 `int64` 类型数字相结合，以确保每次执行时都能提供随机的输入（如果你在这里放置一个常数，随机化器也会生成相同的数字）。`rand.Intn(int)` 方法返回一个介于零和指定数字之间的整数，因此在这里我们涵盖了*接受标准 2*。

接下来，一个请求设置重试次数的消息位于 `fmt.Fscanf` 方法之前，这是一个强大的函数，你可以向它传递一个 `io.Reader`（控制台的标准输入）、一个格式（数字）和一个接口来存储读取器的内容，在这种情况下，是上下文的 `Retries` 字段。

最后，我们返回 `true` 来告诉引擎游戏必须继续。让我们看看我们一开始在函数中使用的 `AskState` 结构体：

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

`AskState` 结构体也实现了 `GameState` 状态，正如你可能已经猜到的。这个状态从给玩家的消息开始，要求他们插入一个新的数字。在接下来的三行中，我们创建一个局部变量来存储玩家将要输入的数字的内容。我们再次使用了 `fmt.Fscanf` 方法，就像我们在 `StartState` 结构体中所做的那样，来捕获玩家的输入并将其存储在变量 `n` 中。然后，我们的计数器中减少了一个重试，所以我们必须从 `Retries` 字段表示的重试次数中减去一个。

然后，有两个检查：一个检查用户是否输入了正确的数字，如果是这样，则将上下文字段的 `Won` 设置为 `true`，并将下一个状态设置为 `FinishState` 结构体（尚未声明）。

第二个检查是控制重试次数是否未达到零，如果是这样，它不会让玩家再次请求数字，并将玩家直接发送到 `FinishState` 结构体。毕竟，我们必须再次告诉游戏引擎游戏必须继续，通过在 `executeState` 方法中返回 `true`。

最后，我们定义 `FinishState` 结构体。它控制游戏的退出状态，检查上下文对象中 `Won` 字段的内容：

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

`TheFinishState` 结构体通过在其结构体中拥有 `executeState` 方法来实现 `GameState` 状态。这里的想法非常简单--如果玩家已经赢了（这个字段在 `AskState` 结构体中之前已经设置），则 `FinishState` 结构体会打印消息 `恭喜，你赢了`。如果玩家没有赢（记住布尔变量的零值是 `false`），则 `FinishState` 会打印消息 `你输了`。

在这种情况下，游戏可以被认为是结束了，所以我们返回 `false` 来表示游戏必须不继续。

我们只需要 `main` 方法来玩我们的游戏：

```go
func main() { 
  start := StartState{} 
  game := GameContext{ 
    Next:&start, 
  } 
  for game.Next.executeState(&game) {} 
} 

```

好吧，是的，这不能更简单了。游戏必须从`start`方法开始，尽管将来如果游戏需要更多的初始化，它可以在外面进一步抽象化，但就我们目前的情况来看，这是可以的。然后，我们创建一个上下文，我们将`Next`状态设置为指向`start`变量的指针。所以游戏将首先执行的是`StartState`状态。

`main`函数的最后一行有很多东西只是在那里。我们创建了一个循环，循环体内没有任何语句。就像任何循环一样，当条件不满足时，它会一直循环。我们使用的是`GameStates`结构的返回值，只要游戏没有结束，就返回`true`。

所以，这个想法很简单：我们在上下文中执行状态，传递上下文的指针给它。每个状态都会返回`true`，直到游戏结束，`FinishState`结构将返回`false`。所以我们的 for 循环会一直循环，等待`FinishState`结构发送的`false`条件来结束应用程序。

让我们玩一次：

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

我们输了！我们将重试次数设置为 5。然后我们继续输入数字，试图猜出秘密数字。我们输入了 8、2、1、3 和 4，但都不是。我甚至不知道正确的数字是什么；让我们修复这个问题！

前往`FinishState`结构的定义，并更改显示“你输了”的那一行，将其替换为以下内容：

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

这次我们让它变得更难一些，只设置了三次尝试...我们又输了。我输入了 6、2 和 1，但正确的数字是 9。最后一次尝试：

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

太棒了！这次我们降低了难度，允许最多尝试五次，我们赢了！我们甚至还有一次额外的尝试，但在第四次尝试后，我们输入了 3、4、5，猜出了数字。正确的数字是 6，这是我第四次尝试。

## 胜利状态和失败状态

你意识到我们可以有一个胜利状态和一个失败状态，而不是直接在`FinishState`结构中打印消息吗？这样我们就可以，例如，检查胜利部分的一些假设的分数板，看看我们是否创下了记录。让我们重构我们的游戏。首先我们需要一个`WinState`和一个`LoseState`结构：

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

这两个新状态没有新内容。它们包含之前在`FinishState`状态中已有的相同信息，顺便说一句，必须修改以使用这些新状态：

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

现在，完成状态不会打印任何内容，而是将这个任务委托给链中的下一个状态——如果用户赢了，就是`WinState`结构，如果没有赢，就是`LoseState`结构。记住，现在游戏不会在`FinishState`结构中结束，我们必须返回`true`来通知引擎它必须继续执行链中的状态。

## 使用状态模式构建的游戏

你现在可能认为你可以通过添加新的状态来无限扩展这个游戏，这是真的。状态模式的强大之处不仅在于能够创建复杂的有限状态机（FSM），而且在于它具有足够的灵活性，可以通过添加新状态和修改一些旧状态来指向新状态，而不会影响整个 FSM 的其余部分。

# 中介者设计模式

让我们继续讨论中介者模式。正如其名称所暗示的，这是一种位于两种类型之间以交换信息的模式。但是，我们为什么想要这种行为呢？让我们详细看看。

## 描述

任何设计模式的关键目标之一是避免对象之间的紧密耦合。这可以通过许多方式实现，正如我们之前所看到的。

但当应用程序规模很大时，这是一种特别有效的方法，即中介者模式。中介者模式是程序员经常使用而很少思考的模式的一个完美例子。

中介者模式将充当负责在两个对象之间交换通信的类型。这样，通信对象不需要相互了解，可以更自由地改变。维护哪些对象提供什么信息的模式是中介者。

## 目标

如前所述，中介者模式的主要目标是关于松散耦合和封装。目标是：

+   为了在必须相互通信的两个对象之间提供松散耦合

+   通过将这些需求传递给中介者模式，将特定类型的依赖性减少到最小

## 计算器

对于中介者模式，我们将开发一个极其简单的算术计算器。你可能认为计算器如此简单，不需要任何模式。但我们会看到这并不完全正确。

我们的计算器只会执行两种非常简单的操作：求和和减法。

## 接受标准

说到用接受标准来定义计算器，听起来相当有趣，但让我们这样做：

1.  定义一个名为 `Sum` 的操作，它接受一个数字并将其添加到另一个数字上。

1.  定义一个名为 `Subtract` 的操作，它接受一个数字并将其从另一个数字中减去。

嗯，我不知道你是否和我一样，我真的需要在这 *复杂* 的标准之后休息一下。那么我们为什么要定义这么多呢？耐心点，你很快就会得到答案。

## 实现

我们必须直接跳到实现，因为我们无法测试求和是否正确（嗯，我们可以，但我们将测试 Go 是否正确编写！）。我们可以测试是否通过了接受标准，但这对于我们这个例子来说有点过度。

因此，让我们首先实现必要的类型：

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

嗯...这看起来相当尴尬。我们已经在 Go 中有了执行这些操作的数值类型，我们不需要为每个数字创建一个类型！

但让我们先继续这种疯狂的方法。让我们实现 `One` 结构体：

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

好吧，我就说到这里。这个实现有什么问题？这完全疯狂！为了使数字之间所有可能的操作都成为可能的求和操作，这是过度杀鸡用牛刀！尤其是当我们有多个数字的时候。

好吧，信不信由你，这就是今天许多软件通常是如何设计的。一个小型应用程序，其中对象使用两个或三个对象开始，最终会使用几十个。简单地添加或删除应用程序中的一个类型变得绝对痛苦，因为它隐藏在这些疯狂之中。

那么，在这个计算器中我们能做什么呢？使用一个中介类型，释放所有情况：

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

我们只是开发了一对数字来简化事情。`Sum` 函数充当两个数字之间的中介。首先，它检查名为 `a` 的第一个数字的类型。然后，对于第一个数字的每种类型，它检查名为 `b` 的第二个数字的类型，并返回结果类型。

虽然解决方案现在看起来仍然非常疯狂，但唯一了解计算器中所有可能数字的是 `Sum` 函数。但仔细看看，你会发现我们为 `int` 类型添加了一个类型情况。我们有 `One`、`Two` 和 `int` 的情况。在 `int` 情况内部，我们还有一个 `int` 情况用于数字 `b`。我们在这里做什么？如果两种类型都是 `int` 情况，我们可以返回它们的和。

你认为这会工作吗？让我们写一个简单的 `main` 函数：

```go
func main(){ 
  fmt.Printf("%#v\n", Sum(One{}, Two{})) 
  fmt.Printf("%d\n", Sum(1,2)) 
} 

```

我们打印类型 `One` 和类型 `Two` 的和。通过使用 `"%#v"` 格式，我们要求打印有关类型的详细信息。函数的第二行使用 `int` 类型，我们也打印了结果。这在控制台产生了以下输出：

```go
$go run mediator.go
&main.Three{}
7

```

并不太令人印象深刻，对吧？但让我们思考一下。通过使用中介者模式，我们已经能够重构最初的计算器，其中我们必须为每个类型定义每个操作，到中介者模式，即 `Sum` 函数。

好事是，多亏了中介者模式，我们能够开始使用整数作为计算器的值。我们只是通过添加两个整数定义了最简单的例子，但我们可以用整数和 `type` 做同样的事情：

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

通过这个小小的修改，我们现在可以使用类型 `One` 和 `int` 作为数字 `b`。如果我们继续在这个中介者模式上工作，我们可以在不实现它们之间所有可能的操作的情况下，实现类型之间的大量灵活性，从而避免产生紧密耦合。

我们将在主函数中添加一个新的 `Sum` 方法来观察这个动作：

```go
func main(){ 
  fmt.Printf("%#v\n", Sum(One{}, Two{})) 
  fmt.Printf("%d\n", Sum(1,2)) 
 fmt.Printf("%d\n", Sum(One{},2)) 
} 
$go run mediator.go&main.Three{}33

```

很好。中介者模式负责了解所有可能的类型，并返回最适合我们情况的类型，即整数。现在我们可以继续扩展这个 `Sum` 函数，直到我们完全摆脱使用我们定义的数值类型。

## 使用中介者解耦两个类型

我们已经进行了一个颠覆性的示例，试图跳出思维定势，深入思考中介者模式。应用程序中实体的紧密耦合在未来可能会变得非常复杂，如果需要，允许更困难的重构。

只需记住，中介者模式的作用是在两种彼此不了解的类型之间充当管理类型，这样你就可以在不影响另一种类型的情况下替换一种类型，或者以更简单、更方便的方式替换一种类型。

# 观察者设计模式

我们将完成常见的*四大家族*设计模式，以我最喜欢的模式——观察者模式，也称为发布/订阅或发布/监听器模式。使用状态模式，我们定义了第一个事件驱动架构，但使用观察者模式，我们将真正达到一个新的抽象层次。

## 描述

观察者模式背后的思想很简单——订阅某些事件，这些事件将在许多订阅类型上触发某些行为。这为什么如此有趣？因为我们解耦了事件与其可能的处理器。

例如，想象一个登录按钮。我们可以编写代码，当用户点击按钮时，按钮颜色改变，执行一个动作，并在后台执行表单检查。但使用观察者模式，改变颜色的类型将订阅按钮点击的事件。检查表单的类型和执行动作的类型也将订阅此事件。

## 目标

观察者模式特别适用于实现由一个事件触发的多个动作。它也特别适用于你事先不知道事件之后将执行多少动作，或者有可能会在不久的将来增加动作数量的可能性。总结如下：

+   提供一个事件驱动架构，其中一个事件可以触发一个或多个动作

+   解耦执行的动作与触发它们的动作

+   提供多个触发相同动作的事件

## 通知者

我们将开发一个尽可能简单的应用程序，以充分理解观察者模式的根源。我们将创建一个`Publisher`结构体，它是触发事件的那个，因此它必须接受新的观察者，并在必要时移除它们。当`Publisher`结构体被触发时，它必须通知所有订阅的新事件及其相关数据。

## 接受标准

需求必须告诉我们有一个类型可以触发一个或多个动作：

1.  我们必须有一个带有`NotifyObservers`方法的发布者，该方法接受一个消息作为参数，并在每个订阅的观察者上触发一个`Notify`方法。

1.  我们必须有一种方法可以向发布者添加新的订阅者。

1.  我们必须有一种方法来从发布者中移除新的订阅者。

## 单元测试

也许你已经意识到，我们定义的要求几乎完全针对 `Publisher` 类型。这是因为观察者的动作对于观察者模式来说是不相关的。它应该简单地执行一个动作，在这种情况下是 `Notify` 方法，这个动作由一个或多个类型实现。所以让我们只为这个模式定义这个接口：

```go
type Observer interface { 
  Notify(string) 
} 

```

`Observer` 接口有一个 `Notify` 方法，它接受一个 `string` 类型的参数，该参数将包含要传播的消息。它不需要返回任何内容，但如果我们想检查在调用 `Publisher` 结构的 `publish` 方法时是否所有观察者都已到达，我们可以返回一个错误。

要测试所有验收标准，我们只需要一个名为 `Publisher` 的结构，它有三个方法：

```go
type Publisher struct { 
  ObserversList []Observer 
} 

func (s *Publisher) AddObserver(o Observer) {} 

func (s *Publisher) RemoveObserver(o Observer) {} 

func (s *Publisher) NotifyObservers(m string) {} 

```

`Publisher` 结构将订阅的观察者列表存储在一个名为 `ObserversList` 的切片字段中。然后它有在验收标准中提到的三个方法——`AddObserver` 方法用于将新的观察者订阅到发布者，`RemoveObserver` 方法用于取消订阅观察者，以及 `NotifyObservers` 方法，该方法使用一个字符串作为我们希望在所有观察者之间传播的消息。

使用这三种方法，我们必须设置一个根测试来配置 `Publisher` 和三个子测试来测试每个方法。我们还需要定义一个实现 `Observer` 接口的测试类型结构。这个结构将被命名为 `TestObserver`：

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

`TestObserver` 结构通过在其结构中定义一个 `Notify(string)` 方法来实现观察者模式。在这种情况下，它将接收到的消息与其自己的观察者 ID 一起打印出来。然后，它将消息存储在其 `Message` 字段中。这允许我们稍后检查 `Message` 字段的内容是否如预期。记住，这也可以通过传递 `testing.T` 指针和预期的消息，并在 `TestObserver` 结构内进行检查来实现。

现在，我们可以设置 `Publisher` 结构来执行三个测试。我们将创建三个 `TestObserver` 结构的实例：

```go
func TestSubject(t *testing.T) { 
  testObserver1 := &TestObserver{1, ""} 
  testObserver2 := &TestObserver{2, ""} 
  testObserver3 := &TestObserver{3, ""} 
  publisher := Publisher{} 

```

我们为每个观察者分配了不同的 ID，这样我们就可以在以后看到每个观察者都打印了预期的消息。然后，我们通过在 `Publisher` 结构上调用 `AddObserver` 方法来添加观察者。

让我们编写一个 `AddObserver` 测试，它必须将一个新的观察者添加到 `Publisher` 结构的 `ObserversList` 字段：

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

我们已经向 `Publisher` 结构添加了三个观察者，因此切片的长度必须是 3。如果不是 3，则测试将失败。

`RemoveObserver` 测试将获取 ID 为 2 的观察者并将其从列表中移除：

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

在移除第二个观察者后，`Publisher` 结构的长度现在必须是 2。我们还检查剩下的观察者中没有 ID 为 2 的，因为它必须被移除。

最后要测试的方法是`Notify`方法。当使用`Notify`方法时，`TestObserver`结构体的所有实例都必须将它们的`Message`字段从空更改为传递的消息（在这种情况下是`Hello World!`）。首先，我们将在调用`NotifyObservers`测试之前检查所有`Message`字段实际上是否为空：

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

使用`for`语句，我们正在遍历`ObserversList`字段以在`publisher`实例中切片。我们需要将观察者指针强制转换为`TestObserver`结构体的指针，并检查转换是否正确完成。然后，我们检查`Message`字段实际上是否为空。

下一步是创建要发送的消息--在这种情况下，它将是`"Hello World!"`，然后将此消息传递给`NotifyObservers`方法以通知列表上的每个观察者（目前只有观察者 1 和 3）：

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

在调用`NotifyObservers`方法之后，`ObserversList`字段中的每个`TestObserver`都必须在它们的`Message`字段中存储消息`"Hello World!"`。再次强调，我们使用一个`for`循环来遍历`ObserversList`字段中的每个观察者，并将每个观察者强制转换为`TestObserver`测试（记住，`TestObserver`结构体没有字段，因为它是一个接口）。我们可以通过向`Observer`实例添加一个新的`Message()`方法并在`TestObserver`结构体中实现它来返回`Message`字段的值来避免强制类型转换。这两种方法都是有效的。一旦我们将类型转换为名为`printObserver`的局部变量，我们就检查`ObserversList`结构体中的每个实例是否在它们的`Message`字段中存储了字符串`"Hello World!"`。

是时候运行必须全部失败的测试，以检查它们在后续实现中的有效性：

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

有些事情没有按预期工作。如果我们还没有实现该函数，`Notify`方法是如何通过测试的？再次查看`Notify`方法的测试。测试遍历`ObserversList`结构，每个`Fail`调用都在这个`for`循环内部。如果列表为空，它不会遍历，因此不会执行任何`Fail`调用。

让我们通过在`Notify`测试的开始处添加一个小的不为空列表检查来解决这个问题：

```go
  if len(publisher.ObserversList) == 0 { 
      t.Errorf("The list is empty. Nothing to test\n") 
  } 

```

我们将重新运行测试，以查看`TestSubject/Notify`方法是否已经失败：

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

很好，它们全部都失败了，现在我们对测试有了一些保证。我们可以继续实现。

## 实现

我们的实现只是定义了`AddObserver`、`RemoveObserver`和`NotifyObservers`方法：

```go
func (s *Publisher) AddObserver(o Observer) { 
  s.ObserversList = append(s.ObserversList, o) 
} 

```

`AddObserver`方法通过将指针附加到当前指针列表来将`Observer`实例添加到`ObserversList`结构体中。这个操作非常简单。现在`AddObserver`测试应该通过（但其他测试没有通过，我们可能做错了什么）：

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

很好。仅仅是`AddObserver`方法通过了测试，所以我们现在可以继续到`RemoveObserver`方法：

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

`RemoveObserver` 方法将遍历 `ObserversList` 结构中的每个元素，比较 `Observer` 对象的 `o` 变量与列表中存储的变量。如果找到匹配项，它将索引保存在局部变量 `indexToRemove` 中，并停止迭代。在 Go 中在切片上删除索引的方式有点棘手：

1.  首先，我们需要使用切片索引来返回一个新的切片，包含从切片开始到要删除的索引（不包括）之间的每个对象。

1.  然后，我们从要删除的索引（不包括）到最后一个对象获取另一个切片。

1.  最后，我们将前两个新切片连接到一个新的切片中（即 `append` 函数）

例如，在一个从 1 到 10 的列表中，我们想要删除数字 5，我们必须创建一个新的切片，将一个从 1 到 4 的切片和一个从 6 到 10 的切片连接起来。

这个索引删除操作再次使用 `append` 函数完成，因为我们实际上是在将两个列表连接起来。只需仔细看看 `append` 函数第二个参数末尾的三个点。`append` 函数将一个元素（第二个参数）添加到一个切片（第一个参数）中，但我们要添加一个整个列表。这可以通过使用三个点来实现，这相当于 *继续添加元素，直到完成第二个数组*。

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

我们继续沿着正确的道路前进。`RemoveObserver` 测试已经修复，而没有修复其他任何内容。现在我们必须通过定义 `NotifyObservers` 方法来完成我们的实现：

```go
func (s *Publisher) NotifyObservers(m string) { 
  fmt.Printf("Publisher received message '%s' to notify observers\n", m) 
  for _, observer := range s.ObserversList { 
    observer.Notify(m) 
  } 
} 

```

`NotifyObservers` 方法相当简单，因为它向控制台打印一条消息，宣布要将特定消息传递给 `Observers`。之后，我们使用一个 for 循环遍历 `ObserversList` 结构，并通过传递参数 `m` 执行每个 `Notify(string)` 方法。执行此操作后，所有观察者必须在他们的 `Message` 字段中存储消息 `Hello World!`。让我们通过运行测试来验证这一点：

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

太棒了！我们还可以在控制台上看到 `Publisher` 和 `Observer` 类型的输出。`Publisher` 结构打印以下消息：

```go
hey! I have received the message  'Hello World!' and I'm going to pass the same message to the observers 
```

在此之后，所有观察者将按照以下方式打印各自的消息：

```go
hey, I'm observer 1 and I have received the message 'Hello World!'
```

第三位观察者也是如此。

## 摘要

我们已经通过状态模式和观察者模式解锁了事件驱动架构的力量。现在你可以在你的应用程序中真正执行异步算法和操作，这些算法和操作响应你系统中的事件。

观察者模式在 UI 中被广泛使用。Android 编程充满了观察者模式，这样 Android SDK 就可以将程序员创建应用程序时要执行的操作委托给它们。
