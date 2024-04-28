# 第三章：结构模式 - 组合，适配器和桥接设计模式

我们将开始我们的结构模式之旅。结构模式，顾名思义，帮助我们用常用的结构和关系来塑造我们的应用程序。

Go 语言本质上鼓励使用组合，几乎完全不使用继承。因此，我们一直在广泛使用**组合**设计模式，所以让我们从定义组合设计模式开始。

# 组合设计模式

组合设计模式倾向于组合（通常定义为*拥有*关系）而不是继承（*是*关系）。自上世纪九十年代以来，*组合优于继承*的方法一直是工程师之间讨论的话题。我们将学习如何使用*拥有*方法创建对象结构。总的来说，Go 没有继承，因为它不需要！

## 描述

在组合设计模式中，您将创建对象的层次结构和树。对象内部有不同的对象，具有它们自己的字段和方法。这种方法非常强大，解决了继承和多重继承的许多问题。例如，典型的继承问题是当您有一个实体从两个完全不同的类继承时，它们之间绝对没有关系。想象一个训练的运动员，和一个游泳的游泳者：

+   `Athlete`类有一个`Train()`方法

+   `Swimmer`类有一个`Swim()`方法

`Swimmer`类继承自`Athlete`类，因此它继承了其`Train`方法并声明了自己的`Swim`方法。您还可以有一个自行车手，也是一名运动员，并声明了一个`Ride`方法。

但现在想象一下，一种会吃东西的动物，比如一只也会叫的狗：

+   `Cyclist`类有一个`Ride()`方法

+   `Animal`类有`Eat()`，`Dog()`和`Bark()`方法

没有花哨的东西。您也可以有一条鱼是一种动物，是的，会游泳！那么，您如何解决呢？鱼不能是一个还会训练的游泳者。鱼不训练（据我所知！）。您可以创建一个带有`Swim`方法的`Swimmer`接口，并使游泳者运动员和鱼实现它。这将是最好的方法，但您仍然必须两次实现`swim`方法，因此代码的可重用性将受到影响。那么三项全能运动员呢？他们是游泳，跑步和骑车的运动员。通过多重继承，您可以有一种解决方案，但这很快就会变得复杂且难以维护。

## 目标

正如您可能已经想象的那样，组合的目标是避免这种层次结构混乱，其中应用程序的复杂性可能会增长太多，代码的清晰度受到影响。

## 游泳者和鱼

我们将以 Go 的方式解决运动员和游泳的鱼的问题。在 Go 中，我们可以使用两种类型的组合--**直接**组合和**嵌入**组合。我们将首先通过使用直接组合来解决这个问题，即在结构体内部拥有所需的一切。

## 需求和验收标准

要求与之前描述的要求相似。我们将有一个运动员和一个游泳者。我们还将有一个动物和一条鱼。`Swimmer`和`Fish`方法必须共享代码。运动员必须训练，动物必须吃：

+   我们必须有一个带有`Train`方法的`Athlete`结构

+   我们必须有一个带有`Swim`方法的`Swimmer`

+   我们必须有一个带有`Eat`方法的`Animal`结构

+   我们必须有一个带有`Swim`方法的`Fish`结构，该方法与`Swimmer`共享，而不会出现继承或层次结构问题

## 创建组合

组合设计模式是一种纯粹的结构模式，除了结构本身之外，没有太多需要测试的地方。在这种情况下，我们不会编写单元测试，而只是描述在 Go 中创建这些组合的方法。

首先，我们将从`Athlete`结构和其`Train`方法开始：

```go
type Athlete struct{} 

func (a *Athlete) Train() { 
  fmt.Println("Training") 
} 

```

前面的代码非常简单。它的`Train`方法打印单词`Training`和一个换行符。我们将创建一个具有`Athlete`结构的复合游泳者：

```go
type CompositeSwimmerA struct{ 
  MyAthlete Athlete 
  MySwim func() 
} 

```

`CompositeSwimmerA`类型有一个`Athlete`类型的`MyAthlete`字段。它还存储一个`func()`类型。请记住，在 Go 中，函数是一等公民，它们可以像任何变量一样作为参数、字段或参数使用。因此，`CompositeSwimmerA`有一个`MySwim`字段，其中存储了一个**闭包**，它不带参数并且不返回任何内容。我如何将函数分配给它呢？好吧，让我们创建一个与`func()`签名匹配的函数（无参数，无返回）：

```go
func Swim(){ 
  fmt.Println("Swimming!") 
} 

```

就是这样！`Swim()`函数不带参数并且不返回任何内容，因此它可以用作`CompositeSwimmerA`结构中的`MySwim`字段：

```go
swimmer := CompositeSwimmerA{ 
  MySwim: Swim, 
} 

swimmer.MyAthlete.Train() 
swimmer.MySwim() 

```

因为我们有一个名为`Swim()`的函数，我们可以将其分配给`MySwim`字段。请注意，`Swim`类型没有括号，这将执行其内容。这样我们就可以将整个函数复制到`MySwim`方法中。

但等等。我们还没有将运动员传递给`MyAthlete`字段，我们正在使用它！这将失败！让我们看看执行此片段时会发生什么：

```go
$ go run main.go
Training
Swimming!

```

这很奇怪，不是吗？实际上并不是，因为 Go 中的零初始化的性质。如果您没有将`Athlete`结构传递给`CompositeSwimmerA`类型，编译器将创建一个其值为零初始化的结构，也就是说，一个`Athlete`结构，其字段的值初始化为零。如果这看起来令人困惑，请查看第一章*准备...开始...跑！*来回顾零初始化。再次考虑`CompositeSwimmerA`结构代码：

```go
type CompositeSwimmerA struct{ 
  MyAthlete Athlete 
  MySwim    func() 
} 

```

现在我们有一个存储在`MySwim`字段中的函数指针。我们可以以相同的方式分配`Swim`函数，但需要多一步：

```go
localSwim := Swim 

swimmer := CompositeSwimmerA{ 
  MySwim: localSwim, 
} 

swimmer.MyAthlete.Train() 
swimmer.MySwim () 

```

首先，我们需要一个包含函数`Swim`的变量。这是因为函数没有地址，无法将其传递给`CompositeSwimmerA`类型。然后，为了在结构体内使用这个函数，我们必须进行两步调用。

那么我们的鱼问题呢？有了我们的`Swim`函数，这不再是问题。首先，我们创建`Animal`结构：

```go
type Animal struct{} 

func (r *Animal)Eat() { 
  println("Eating") 
} 

```

然后我们将创建一个嵌入`Animal`对象的`Shark`对象：

```go
type Shark struct{ 
  Animal 
  Swim func() 
} 

```

等一下！`Animal`类型的字段名在哪里？你有没有意识到我在上一段中使用了*embed*这个词？这是因为在 Go 中，您还可以将对象嵌入到对象中，使其看起来很像继承。也就是说，我们不必显式调用字段名来访问其字段和方法，因为它们将成为我们的一部分。因此，以下代码将是完全正常的：

```go
fish := Shark{ 
  Swim: Swim, 
} 

fish.Eat() 
fish.Swim() 

```

现在我们有一个`Animal`类型，它是零初始化并嵌入的。这就是为什么我可以调用`Animal`结构的`Eat`方法而不创建它或使用中间字段名。此片段的输出如下：

```go
$ go run main.go 
Eating 
Swimming!

```

最后，有第三种使用组合模式的方法。我们可以创建一个带有`Swim`方法的`Swimmer`接口和一个`SwimmerImpl`类型，将其嵌入到运动员游泳者中：

```go
type Swimmer interface { 
  Swim() 
} 
type Trainer interface { 
  Train() 
} 

type SwimmerImpl struct{} 
func (s *SwimmerImpl) Swim(){ 
  println("Swimming!") 
} 

type CompositeSwimmerB struct{ 
  Trainer 
  Swimmer 
} 

```

使用这种方法，您可以更明确地控制对象的创建。`Swimmer`字段被嵌入，但不会被零初始化，因为它是一个指向接口的指针。这种方法的正确使用将是以下方式：

```go
swimmer := CompositeSwimmerB{ 
  &Athlete{}, 
  &SwimmerImpl{}, 
} 

swimmer.Train() 
swimmer.Swim() 

```

`CompositeSwimmerB`的输出如下，如预期的那样：

```go
$ go run main.go
Training
Swimming!

```

哪种方法更好？嗯，我有个人偏好，不应被视为金科玉律。在我看来，*接口*方法是最好的，原因有很多，但主要是因为明确性。首先，您正在使用首选的接口而不是结构。其次，您不会将代码的部分留给编译器的零初始化特性。这是一个非常强大的功能，但必须小心使用，因为它可能导致运行时问题，而在使用接口时，您会在编译时发现这些问题。在不同的情况下，零初始化实际上会在运行时为您节省，事实上！但我尽可能多地使用接口，所以这实际上并不是一个选项。

## 二叉树组合

另一种非常常见的组合模式是在使用二叉树结构时。在二叉树中，您需要在字段中存储自身的实例：

```go
type Tree struct { 
  LeafValue int 
  Right     *Tree 
  Left      *Tree 
} 

```

这是一种递归组合，由于递归的性质，我们必须使用指针，以便编译器知道它必须为此结构保留多少内存。我们的`Tree`结构为每个实例存储了一个`LeafValue`对象，并在其`Right`和`Left`字段中存储了一个新的`Tree`。

有了这个结构，我们可以创建一个对象，就像这样：

```go
root := Tree{ 
  LeafValue: 0, 
  Right:&Tree{ 
    LeafValue: 5, 
    Right: &1Tree{ 6, nil, nil }, 
    Left: nil, 
  }, 
  Left:&Tree{ 4, nil, nil }, 
} 

```

我们可以这样打印其最深层分支的内容：

```go
fmt.Println(root.Right.Right.LeafValue) 

$ go run main.go 
6

```

## 组合模式与继承

在 Go 中使用组合设计模式时，必须非常小心，不要将其与继承混淆。例如，当您在`Son`结构中嵌入`Parent`结构时，就像以下示例中一样：

```go
type Parent struct { 
  SomeField int 
} 

type Son struct { 
  Parent 
} 

```

您不能认为`Son`结构也是`Parent`结构。这意味着您不能将`Son`结构的实例传递给期望`Parent`结构的函数，就像以下示例中一样：

```go
func GetParentField(p *Parent) int{ 
  fmt.Println(p.SomeField) 
} 

```

当您尝试将`Son`实例传递给`GetParentField`方法时，您将收到以下错误消息：

```go
cannot use son (type Son) as type Parent in argument to GetParentField

```

事实上，这是有很多道理的。这个问题的解决方案是什么？嗯，您可以简单地将`Son`结构与父结构组合起来，而不是嵌入，以便稍后可以访问`Parent`实例：

```go
type Son struct { 
  P Parent 
} 

```

所以现在你可以使用`P`字段将其传递给`GetParentField`方法：

```go
son := Son{} 
GetParentField(son.P) 

```

## 关于组合模式的最后几句话

在这一点上，您应该真的很熟悉使用组合设计模式。这是 Go 语言中非常惯用的特性，从纯面向对象的语言切换过来并不是非常痛苦的。组合设计模式使我们的结构可预测，但也允许我们创建大多数设计模式，正如我们将在后面的章节中看到的。

# 适配器设计模式

最常用的结构模式之一是**适配器**模式。就像在现实生活中，您有插头适配器和螺栓适配器一样，在 Go 中，适配器将允许我们使用最初未为特定任务构建的东西。

## 描述

当接口过时且无法轻松或快速替换时，适配器模式非常有用。相反，您可以创建一个新接口来处理应用程序当前需求，该接口在底层使用旧接口的实现。

适配器还帮助我们在应用程序中保持*开闭原则*，使其更可预测。它们还允许我们编写使用一些无法修改的基础的代码。

### 注意

开闭原则首次由 Bertrand Meyer 在他的书《面向对象的软件构造》中提出。他指出代码应该对新功能开放，但对修改关闭。这是什么意思？嗯，这意味着一些事情。一方面，我们应该尝试编写可扩展的代码，而不仅仅是可工作的代码。同时，我们应该尽量不修改源代码（你的或其他人的），因为我们并不总是意识到这种修改的影响。只需记住，代码的可扩展性只能通过设计模式和面向接口的编程来实现。

## 目标

适配器设计模式将帮助您满足最初不兼容的代码部分的需求。这是在决定适配器模式是否适合您的问题时要牢记的关键点——最初不兼容但必须一起工作的两个接口是适配器模式的良好候选对象（但它们也可以使用外观模式，例如）。

## 使用不兼容的接口与适配器对象

对于我们的示例，我们将有一个旧的`Printer`接口和一个新的接口。新接口的用户不希望旧接口的签名，并且我们需要一个适配器，以便用户仍然可以在必要时使用旧的实现（例如与一些旧代码一起工作）。

## 需求和验收标准

有一个名为`LegacyPrinter`的旧接口和一个名为`ModernPrinter`的新接口，创建一个结构来实现`ModernPrinter`接口，并按照以下步骤使用`LegacyPrinter`接口：

1.  创建一个实现`ModernPrinter`接口的适配器对象。

1.  新的适配器对象必须包含`LegacyPrinter`接口的实例。

1.  在使用`ModernPrinter`时，它必须在后台调用`LegacyPrinter`接口，并在前面加上文本`Adapter`。

## 单元测试我们的打印机适配器

我们将首先编写旧代码，但不会测试它，因为我们应该想象它不是我们的代码：

```go
type LegacyPrinter interface { 
  Print(s string) string 
} 
type MyLegacyPrinter struct {} 

func (l *MyLegacyPrinter) Print(s string) (newMsg string) { 
  newMsg = fmt.Sprintf("Legacy Printer: %s\n", s) 
  println(newMsg) 
  return 
} 

```

名为`LegacyPrinter`的旧接口有一个接受字符串并返回消息的`Print`方法。我们的`MyLegacyPrinter`结构实现了`LegacyPrinter`接口，并通过在传递的字符串前加上文本`Legacy Printer:`来修改传递的字符串。在修改文本后，`MyLegacyPrinter`结构将文本打印到控制台，然后返回它。

现在我们将声明我们需要适配的新接口：

```go
type ModernPrinter interface { 
  PrintStored() string 
} 

```

在这种情况下，新的`PrintStored`方法不接受任何字符串作为参数，因为它必须提前存储在实现者中。我们将调用我们的适配器模式的`PrinterAdapter`接口：

```go
type PrinterAdapter struct{ 
  OldPrinter LegacyPrinter 
  Msg        string 
} 
func(p *PrinterAdapter) PrintStored() (newMsg string) { 
  return 
} 

```

如前所述，`PrinterAdapter`适配器必须有一个字段来存储要打印的字符串。它还必须有一个字段来存储`LegacyPrinter`适配器的实例。因此，让我们编写单元测试：

```go
func TestAdapter(t *testing.T){ 
  msg := "Hello World!" 

```

我们将使用消息`Hello World!`作为我们的适配器。当将此消息与`MyLegacyPrinter`结构的实例一起使用时，它会打印文本`Legacy Printer: Hello World!`：

```go
adapter := PrinterAdapter{OldPrinter: &MyLegacyPrinter{}, Msg: msg} 

```

我们创建了一个名为`adapter`的`PrinterAdapter`接口的实例。我们将`MyLegacyPrinter`结构的实例作为`LegacyPrinter`字段传递给`OldPrinter`。此外，我们在`Msg`字段中设置要打印的消息：

```go
returnedMsg := adapter.PrintStored() 

if returnedMsg != "Legacy Printer: Adapter: Hello World!\n" { 
  t.Errorf("Message didn't match: %s\n", returnedMsg) 
} 

```

然后我们使用了`ModernPrinter`接口的`PrintStored`方法；这个方法不接受任何参数，必须返回修改后的字符串。我们知道`MyLegacyPrinter`结构返回传递的字符串，并在前面加上文本`LegacyPrinter:`，适配器将在前面加上文本`Adapter:`。因此，最终我们必须有文本`Legacy Printer: Adapter: Hello World!\n`。

由于我们正在存储接口的实例，因此我们还必须检查我们处理指针为 nil 的情况。这是通过以下测试完成的：

```go
adapter = PrinterAdapter{OldPrinter: nil, Msg: msg} 
returnedMsg = adapter.PrintStored() 

if returnedMsg != "Hello World!" { 
  t.Errorf("Message didn't match: %s\n", returnedMsg) 
} 

```

如果我们没有传递`LegacyPrinter`接口的实例，适配器必须忽略其适配性质，简单地打印并返回原始消息。是时候运行我们的测试了；考虑以下内容：

```go
$ go test -v .
=== RUN   TestAdapter
--- FAIL: TestAdapter (0.00s)
 adapter_test.go:11: Message didn't match: 
 adapter_test.go:17: Message didn't match: 
FAIL
exit status 1
FAIL

```

## 实施

为了使我们的单个测试通过，我们必须重用存储在`PrinterAdapter`结构中的旧`MyLegacyPrinter`：

```go
type PrinterAdapter struct{ 
  OldPrinter LegacyPrinter 
  Msg        string 
} 

func(p *PrinterAdapter) PrintStored() (newMsg string) { 
  if p.OldPrinter != nil { 
    newMsg = fmt.Sprintf("Adapter: %s", p.Msg) 
    newMsg = p.OldPrinter.Print(newMsg) 
  } 
  else { 
    newMsg = p.Msg 
  } 
return 
} 

```

在`PrintStored`方法中，我们检查是否实际上有一个`LegacyPrinter`的实例。在这种情况下，我们将存储的消息和`Adapter`前缀组合成一个新的字符串，以便将其存储在返回变量（称为`newMsg`）中。然后我们使用指向`MyLegacyPrinter`结构的指针来使用`LegacyPrinter`接口打印组合的消息。

如果在`OldPrinter`字段中没有存储`LegacyPrinter`实例，我们只需将存储的消息分配给返回变量`newMsg`并返回该方法。这应该足以通过我们的测试：

```go
$ go test -v .
=== RUN   TestAdapter
Legacy Printer: Adapter: Hello World!
--- PASS: TestAdapter (0.00s)
PASS
ok

```

完美！现在我们可以通过使用这个`Adapter`来继续使用旧的`LegacyPrinter`接口，同时我们可以为将来的实现使用`ModernPrinter`接口。只要记住，适配器模式理想上只提供使用旧的`LegacyPrinter`的方法，而不提供其他任何东西。这样，它的范围将更加封装和在将来更易于维护。

## Go 源代码中适配器模式的示例

您可以在 Go 语言源代码的许多地方找到适配器实现。著名的`http.Handler`接口有一个非常有趣的适配器实现。在 Go 中，一个非常简单的`Hello World`服务器通常是这样做的：

```go
package main 

import ( 
    "fmt" 
    "log" 
    "net/http" 
) 
type MyServer struct{ 
  Msg string 
} 
func (m *MyServer) ServeHTTP(w http.ResponseWriter,r *http.Request){ 
  fmt.Fprintf(w, "Hello, World") 
} 

func main() { 
  server := &MyServer{ 
  Msg:"Hello, World", 
} 

http.Handle("/", server)  
log.Fatal(http.ListenAndServe(":8080", nil)) 
} 

```

HTTP 包有一个名为`Handle`的函数（类似于 Java 中的`static`方法），它接受两个参数--一个表示路由的字符串和一个`Handler`接口。`Handler`接口如下：

```go
type Handler interface { 
  ServeHTTP(ResponseWriter, *Request) 
} 

```

我们需要实现一个`ServeHTTP`方法，HTTP 连接的服务器端将使用它来执行其上下文。但是还有一个`HandlerFunc`函数，允许您定义一些端点行为：

```go
func main() { 
  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) { 
    fmt.Fprintf(w, "Hello, World") 
  }) 

  log.Fatal(http.ListenAndServe(":8080", nil)) 
} 

```

`HandleFunc`函数实际上是使用函数直接作为`ServeHTTP`实现的适配器的一部分。再慢慢读一遍最后一句--你能猜出它是如何实现的吗？

```go
type HandlerFunc func(ResponseWriter, *Request) 

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { 
  f(w, r) 
} 

```

我们可以定义一个与定义结构相同的函数类型。我们使这个函数类型实现`ServeHTTP`方法。最后，从`ServeHTTP`函数中，我们调用接收器本身`f(w, r)`。

你必须考虑 Go 的隐式接口实现。当我们定义一个像`func(ResponseWriter, *Request)`这样的函数时，它会被隐式地识别为`HandlerFunc`。而且因为`HandleFunc`函数实现了`Handler`接口，我们的函数也隐式地实现了`Handler`接口。这听起来很熟悉吗？如果*A = B*，*B = C*，那么*A = C*。隐式实现为 Go 提供了很多灵活性和功能，但你也必须小心，因为你不知道一个方法或函数是否实现了可能引起不良行为的某个接口。

我们可以在 Go 的源代码中找到更多的示例。`io`包使用管道的示例非常有力。在 Linux 中，管道是一种流机制，它将输入的内容输出为输出的其他内容。`io`包有两个接口，它们在 Go 的源代码中随处可见--`io.Reader`和`io.Writer`接口：

```go
type Reader interface { 
  Read(p []byte) (n int, err error) 
} 

type Writer interface { 
  Write(p []byte) (n int, err error) 
} 

```

我们到处都使用`io.Reader`，例如，当您使用`os.OpenFile`打开文件时，它返回一个文件，实际上实现了`io.Reader`接口。这有什么用呢？想象一下，您编写了一个`Counter`结构，从您提供的数字开始计数到零：

```go
type Counter struct {} 
func (f *Counter) Count(n uint64) uint64 { 
  if n == 0 { 
    println(strconv.Itoa(0)) 
    return 0 
  } 

  cur := n 
  println(strconv.FormatUint(cur, 10)) 
  return f.Count(n - 1) 
} 

```

如果您向这个小片段提供数字 3，它将打印以下内容：

```go
3
2
1

```

嗯，不是很令人印象深刻！如果我想要写入文件而不是打印呢？我们也可以实现这种方法。如果我想要打印到文件和控制台呢？嗯，我们也可以实现这种方法。我们必须通过使用`io.Writer`接口将其模块化一些：

```go
type Counter struct { 
  Writer io.Writer 
} 
func (f *Counter) Count(n uint64) uint64 { 
  if n == 0 { 
    f.Writer.Write([]byte(strconv.Itoa(0) + "\n")) 
    return 0 
  } 

  cur := n 
  f.Writer.Write([]byte(strconv.FormatUint(cur, 10) + "\n")) 
  return f.Count(n - 1) 
}

```

现在我们在`Writer`字段中提供了一个`io.Writer`。这样，我们可以像这样创建计数器：`c := Counter{os.Stdout}`，我们将得到一个控制台`Writer`。但等一下，我们还没有解决我们想要将计数带到许多`Writer`控制台的问题。但是我们可以编写一个新的`Adapter`，其中包含一个`io.Writer`，并使用`Pipe()`连接读取器和写入器，我们可以在相反的极端进行读取。这样，您可以解决这两个不兼容的接口`Reader`和`Writer`可以一起使用的问题。

实际上，我们不需要编写适配器--Go 的`io`库在`io.Pipe()`中为我们提供了一个适配器。管道将允许我们将`Reader`转换为`Writer`接口。`io.Pipe()`方法将为我们提供一个`Writer`（管道的入口）和一个`Reader`（出口）供我们使用。因此，让我们创建一个管道，并将提供的写入器分配给前面示例的`Counter`：

```go
pipeReader, pipeWriter := io.Pipe() 
defer pw.Close() 
defer pr.Close() 

counter := Counter{ 
  Writer: pipeWriter, 
} 

```

现在我们有了一个`Reader`接口，之前我们有了一个`Writer`。我们在哪里可以使用`Reader`？`io.TeeReader`函数帮助我们将数据流从`Reader`接口复制到`Writer`接口，并返回一个新的`Reader`，您仍然可以使用它将数据流再次传输到第二个写入器。因此，我们将从相同的读取器流式传输数据到两个写入器--`file`和`Stdout`。

```go
tee := io.TeeReader(pipeReader, file) 

```

现在我们知道我们正在写入一个文件，我们已经传递给`TeeReader`函数。我们仍然需要打印到控制台。`io.Copy`适配器可以像`TeeReader`一样使用--它接受一个读取器并将其内容写入写入器：

```go
go func(){ 
  io.Copy(os.Stdout, tee) 
}() 

```

我们必须在不同的 Go 例程中启动`Copy`函数，以便并发执行写入操作，并且一个读/写不会阻塞另一个读/写。让我们修改`counter`变量，使其再次计数到 5：

```go
counter.Count(5) 

```

通过对代码进行这种修改，我们得到了以下输出：

```go
$ go run counter.go
5
4
3
2
1
0

```

好的，计数已经打印在控制台上。文件呢？

```go
$ cat /tmp/pipe
5
4
3
2
1
0

```

太棒了！通过使用 Go 原生库中提供的`io.Pipe()`适配器，我们已经将计数器与其输出解耦，并将`Writer`接口适配为`Reader`接口。

## Go 源代码告诉我们有关适配器模式的信息

通过适配器设计模式，您已经学会了一种快速实现应用程序中开/闭原则的方法。与其修改旧的源代码（在某些情况下可能不可能），不如创建一种使用新签名的旧功能的方法。

# 桥梁设计模式

**桥梁**模式是从原始*四人帮*书中得到的定义略微神秘的设计。它将抽象与其实现解耦，以便两者可以独立变化。这种神秘的解释只是意味着您甚至可以解耦最基本的功能形式：将对象与其功能解耦。

## 描述

桥梁模式试图像通常的设计模式一样解耦事物。它将抽象（对象）与其实现（对象执行的操作）解耦。这样，我们可以随心所欲地更改对象的操作。它还允许我们更改抽象对象，同时重用相同的实现。

## 目标

桥梁模式的目标是为经常更改的结构带来灵活性。通过了解方法的输入和输出，它允许我们在不太了解代码的情况下进行更改，并为双方留下更容易修改的自由。

## 每个打印机和每种打印方式都有两种。

对于我们的示例，我们将转到控制台打印机抽象以保持简单。我们将有两个实现。第一个将写入控制台。在上一节中了解了`io.Writer`接口后，我们将使第二个写入`io.Writer`接口，以提供更多灵活性。我们还将有两个抽象对象使用这些实现——一个`Normal`对象，它将以直接的方式使用每个实现，以及一个`Packt`实现，它将在打印消息中附加句子`Message from Packt:`。

在本节的结尾，我们将有两个抽象对象，它们有两种不同的功能实现。因此，实际上，我们将有 2²种可能的对象功能组合。

## 要求和验收标准

正如我们之前提到的，我们将有两个对象（`Packt`和`Normal`打印机）和两个实现（`PrinterImpl1`和`PrinterImpl2`），我们将使用桥接设计模式将它们连接起来。更多或更少，我们将有以下要求和验收标准：

+   一个接受要打印的消息的`PrinterAPI`

+   一个简单地将消息打印到控制台的 API 实现

+   一个将消息打印到`io.Writer`接口的 API 实现

+   一个`Printer`抽象，具有实现打印类型的`Print`方法

+   一个`normal`打印机对象，它将实现`Printer`和`PrinterAPI`接口

+   `normal`打印机将直接将消息转发到实现

+   一个`Packt`打印机，它将实现`Printer`抽象和`PrinterAPI`接口

+   `Packt`打印机将在所有打印中附加消息`Message from Packt:`

## 单元测试桥接模式

让我们从*验收标准 1*开始，即`PrinterAPI`接口。该接口的实现者必须提供一个`PrintMessage(string)`方法，该方法将打印作为参数传递的消息：

```go
type PrinterAPI interface { 
  PrintMessage(string) error 
} 

```

我们将通过前一个 API 的实现转到*验收标准 2*：

```go
type PrinterImpl1 struct{} 

func (p *PrinterImpl1) PrintMessage(msg string) error { 
  return errors.New("Not implemented yet") 
} 

```

我们的`PrinterImpl1`是一种通过提供`PrintMessage`方法的实现来实现`PrinterAPI`接口的类型。`PrintMessage`方法尚未实现，并返回错误。这足以编写我们的第一个单元测试来覆盖`PrinterImpl1`：

```go
func TestPrintAPI1(t *testing.T){ 
  api1 := PrinterImpl1{} 

  err := api1.PrintMessage("Hello") 
  if err != nil { 
    t.Errorf("Error trying to use the API1 implementation: Message: %s\n", err.Error()) 
  } 
} 

```

在我们的测试中，我们创建了一个`PrinterImpl1`类型的实例来覆盖`PrintAPI1`。然后我们使用它的`PrintMessage`方法将消息`Hello`打印到控制台。由于我们还没有实现，它必须返回错误字符串`Not implemented yet`：

```go
$ go test -v -run=TestPrintAPI1 . 
=== RUN   TestPrintAPI1 
--- FAIL: TestPrintAPI1 (0.00s) 
        bridge_test.go:14: Error trying to use the API1 implementation: Message: Not implemented yet 
FAIL 
exit status 1 
FAIL    _/C_/Users/mario/Desktop/go-design-patterns/structural/bridge/traditional

```

好的。现在我们必须编写第二个 API 测试，它将使用`io.Writer`接口：

```go
type PrinterImpl2 struct{ 
  Writer io.Writer 
} 

func (d *PrinterImpl2) PrintMessage(msg string) error { 
  return errors.New("Not implemented yet") 
} 

```

正如你所看到的，我们的`PrinterImpl2`结构存储了一个`io.Writer`实现。此外，我们的`PrintMessage`方法遵循了`PrinterAPI`接口。

现在我们熟悉了`io.Writer`接口，我们将创建一个测试对象来实现这个接口，并将写入它的任何内容存储在一个本地字段中。这将帮助我们检查通过写入器发送的内容：

```go
type TestWriter struct { 
  Msg string 
} 

func (t *TestWriter) Write(p []byte) (n int, err error) { 
  n = len(p) 
  if n > 0 { 
    t.Msg = string(p) 
    return n, nil 
  } 
  err = errors.New("Content received on Writer was empty") 
  return 
} 

```

在我们的测试对象中，我们在将其写入本地字段之前检查内容是否为空。如果为空，我们返回错误，如果不为空，我们将`p`的内容写入`Msg`字段。我们将在以下测试中使用这个小结构来测试第二个 API：

```go
func TestPrintAPI2(t *testing.T){ 
  api2 := PrinterImpl2{} 

  err := api2.PrintMessage("Hello") 
  if err != nil { 
    expectedErrorMessage := "You need to pass an io.Writer to PrinterImpl2" 
    if !strings.Contains(err.Error(), expectedErrorMessage) { 
      t.Errorf("Error message was not correct.\n 
      Actual: %s\nExpected: %s\n", err.Error(), expectedErrorMessage) 
    } 
  } 

```

让我们在这里停顿一下。我们在前面的代码的第一行创建了一个名为`api2`的`PrinterImpl2`实例。我们故意没有传递任何`io.Writer`实例，所以我们首先检查我们是否真的收到了错误。然后我们尝试使用它的`PrintMessage`方法，但我们必须得到一个错误，因为它在`Writer`字段中没有存储任何`io.Writer`实例。错误必须是`You need to pass an io.Writer to PrinterImpl2`，我们隐式检查错误的内容。让我们继续测试：

```go
  testWriter := TestWriter{} 
  api2 = PrinterImpl2{ 
    Writer: &testWriter, 
  } 

  expectedMessage := "Hello" 
  err = api2.PrintMessage(expectedMessage) 
  if err != nil { 
    t.Errorf("Error trying to use the API2 implementation: %s\n", err.Error()) 
  } 

  if testWriter.Msg !=  expectedMessage { 
    t.Fatalf("API2 did not write correctly on the io.Writer. \n  Actual: %s\nExpected: %s\n", testWriter.Msg, expectedMessage) 
  } 
} 

```

对于这个单元测试的第二部分，我们使用`TestWriter`对象的一个实例作为`io.Writer`接口，`testWriter`。我们将消息`Hello`传递给`api2`，并检查是否收到任何错误。然后，我们检查`testWriter.Msg`字段的内容--请记住，我们已经编写了一个`io.Writer`接口，它会将传递给其`Write`方法的任何字节存储在`Msg`字段中。如果一切正确，消息应该包含单词`Hello`。

这些就是我们对`PrinterImpl2`的测试。由于我们还没有任何实现，所以在运行这个测试时应该会得到一些错误。

```go
$ go test -v -run=TestPrintAPI2 .
=== RUN   TestPrintAPI2
--- FAIL: TestPrintAPI2 (0.00s)
bridge_test.go:39: Error message was not correct.
Actual: Not implemented yet
Expected: You need to pass an io.Writer to PrinterImpl2
bridge_test.go:52: Error trying to use the API2 implementation: Not 
implemented yet
bridge_test.go:57: API2 did not write correctly on the io.Writer.
Actual:
Expected: Hello
FAIL
exit status 1
FAIL

```

至少有一个测试通过了--检查在使用`PrintMessage`时是否返回了错误消息（任何错误）。其他一切都失败了，这在这个阶段是预期的。

现在我们需要一个打印机抽象，用于可以使用`PrinterAPI`实现者的对象。我们将定义这个为`PrinterAbstraction`接口，其中包含一个`Print`方法。这涵盖了*验收标准 4*：

```go
type PrinterAbstraction interface { 
  Print() error 
} 

```

对于*验收标准 5*，我们需要一个普通打印机。`Printer`抽象将需要一个字段来存储`PrinterAPI`。因此，我们的`NormalPrinter`可能如下所示：

```go
type NormalPrinter struct { 
  Msg     string 
  Printer PrinterAPI 
} 

func (c *NormalPrinter) Print() error { 
  return errors.New("Not implemented yet") 
} 

```

这足以编写`Print（）`方法的单元测试：

```go
func TestNormalPrinter_Print(t *testing.T) { 
  expectedMessage := "Hello io.Writer" 

  normal := NormalPrinter{ 
    Msg:expectedMessage, 
    Printer: &PrinterImpl1{}, 
  } 

  err := normal.Print() 
  if err != nil { 
    t.Errorf(err.Error()) 
  } 
} 

```

测试的第一部分检查了在使用`PrinterImpl1 PrinterAPI`接口时，`Print（）`方法尚未实现。我们将在这个测试中使用的消息是`Hello io.Writer`。使用`PrinterImpl1`时，我们没有简单的方法来检查消息的内容，因为我们直接打印到控制台。在这种情况下，检查是视觉的，所以我们可以检查*验收标准 6*：

```go
  testWriter := TestWriter{} 
  normal = NormalPrinter{ 
    Msg: expectedMessage, 
    Printer: &PrinterImpl2{ 
      Writer:&testWriter, 
    }, 
  } 

  err = normal.Print() 
  if err != nil { 
    t.Error(err.Error()) 
  } 

  if testWriter.Msg != expectedMessage { 
    t.Errorf("The expected message on the io.Writer doesn't match actual.\n  Actual: %s\nExpected: %s\n", testWriter.Msg, expectedMessage) 
  } 
} 

```

`NormalPrinter`测试的第二部分使用`PrinterImpl2`，这需要一个`io.Writer`接口的实现者。我们在这里重用我们的`TestWriter`结构来检查消息的内容。简而言之，我们希望一个接受`string`类型的`Msg`和`PrinterAPI`类型的`Printer`的`NormalPrinter`结构。在这一点上，如果我使用`Print`方法，我不应该收到任何错误，并且`TestWriter`上的`Msg`字段必须包含我们在初始化`NormalPrinter`时传递给它的消息。

让我们运行测试：

```go
$ go test -v -run=TestNormalPrinter_Print .
=== RUN   TestNormalPrinter_Print
--- FAIL: TestNormalPrinter_Print (0.00s)
 bridge_test.go:72: Not implemented yet
 bridge_test.go:85: Not implemented yet
 bridge_test.go:89: The expected message on the io.Writer doesn't match actual.
 Actual:
 Expected: Hello io.Writer
FAIL
exit status 1
FAIL

```

有一个技巧可以快速检查单元测试的有效性--我们调用`t.Error`或`t.Errorf`的次数必须与控制台上的错误消息数量以及它们产生的行数相匹配。在前面的测试结果中，有三个错误分别在*第 72 行*、*第 85 行*和*第 89 行*，这恰好与我们编写的检查相匹配。

我们的`PacktPrinter`结构在这一点上将与`NormalPrinter`的定义非常相似：

```go
type PacktPrinter struct { 
  Msg     string 
  Printer PrinterAPI 
} 

func (c *PacktPrinter) Print() error { 
  return errors.New("Not implemented yet") 
} 

```

这涵盖了*验收标准 7*。我们几乎可以复制并粘贴以前的测试内容，只需做一些更改：

```go
func TestPacktPrinter_Print(t *testing.T) { 
  passedMessage := "Hello io.Writer" 
  expectedMessage := "Message from Packt: Hello io.Writer" 

  packt := PacktPrinter{ 
    Msg:passedMessage, 
    Printer: &PrinterImpl1{}, 
  } 

  err := packt.Print() 
  if err != nil { 
    t.Errorf(err.Error()) 
  } 

  testWriter := TestWriter{} 
  packt = PacktPrinter{ 
    Msg: passedMessage, 
    Printer:&PrinterImpl2{ 
      Writer:&testWriter, 
    }, 
  } 

  err = packt.Print() 
  if err != nil { 
    t.Error(err.Error()) 
  } 

  if testWriter.Msg != expectedMessage { 
    t.Errorf("The expected message on the io.Writer doesn't match actual.\n  Actual: %s\nExpected: %s\n", testWriter.Msg,expectedMessage) 
  } 
} 

```

我们在这里做了什么改变？现在我们有了`passedMessage`，它代表了我们传递给`PackPrinter`的消息。我们还有一个预期的消息，其中包含了来自`Packt`的带前缀的消息。如果您还记得*验收标准 8*，这个抽象必须给传递给它的任何消息加上`Message from Packt：`的前缀，并且同时，它必须能够使用`PrinterAPI`接口的任何实现。

第二个改变是，我们实际上创建了`PacktPrinter`结构，而不是`NormalPrinter`结构；其他一切都是一样的：

```go
$ go test -v -run=TestPacktPrinter_Print .
=== RUN   TestPacktPrinter_Print
--- FAIL: TestPacktPrinter_Print (0.00s)
 bridge_test.go:104: Not implemented yet
 bridge_test.go:117: Not implemented yet
 bridge_test.go:121: The expected message on the io.Writer d
oesn't match actual.
 Actual:
 Expected: Message from Packt: Hello io.Writer
FAIL
exit status 1
FAIL

```

三个检查，三个错误。所有测试都已覆盖，我们终于可以继续实施了。

## 实施

我们将按照创建测试的顺序开始实现，首先是`PrinterImpl1`的定义：

```go
type PrinterImpl1 struct{} 
func (d *PrinterImpl1) PrintMessage(msg string) error { 
  fmt.Printf("%s\n", msg) 
  return nil 
} 

```

我们的第一个 API 接收消息`msg`并将其打印到控制台。在空字符串的情况下，将不会打印任何内容。这足以通过第一个测试：

```go
$ go test -v -run=TestPrintAPI1 .
=== RUN   TestPrintAPI1
Hello
--- PASS: TestPrintAPI1 (0.00s)
PASS
ok

```

您可以在测试输出的第二行中看到`Hello`消息，就在`RUN`消息之后。

`PrinterImpl2`结构也不是很复杂。不同之处在于，我们将在`io.Writer`接口上写入，而不是打印到控制台，这必须存储在结构中。

```go
type PrinterImpl2 struct { 
  Writer io.Writer 
} 

func (d *PrinterImpl2) PrintMessage(msg string) error { 
  if d.Writer == nil { 
    return errors.New("You need to pass an io.Writer to PrinterImpl2") 
  } 

  fmt.Fprintf(d.Writer, "%s", msg) 
  return nil 
} 

```

根据我们的测试，我们首先检查了`Writer`字段的内容，并返回了预期的错误消息`**You need to pass an io.Writer to PrinterImpl2**`，如果没有存储任何内容。这是我们稍后将在测试中检查的消息。然后，`fmt.Fprintf`方法将`io.Writer`接口作为第一个字段，并将格式化的消息作为其余部分，因此我们只需将`msg`参数的内容转发给提供的`io.Writer`：

```go
$ go test -v -run=TestPrintAPI2 .
=== RUN   TestPrintAPI2
--- PASS: TestPrintAPI2 (0.00s)
PASS
ok

```

现在我们将继续使用普通打印机。这个打印机必须简单地将消息转发给存储在`PrinterAPI`接口中的`Printer`，而不做任何修改。在我们的测试中，我们使用了两种`PrinterAPI`的实现--一种打印到控制台，一种写入到`io.Writer`接口：

```go
type NormalPrinter struct { 
  Msg     string 
  Printer PrinterAPI 
} 

func (c *NormalPrinter) Print() error { 
  c.Printer.PrintMessage(c.Msg) 
  return nil 
}
```

我们返回 nil，因为没有发生错误。这应该足以通过单元测试：

```go
$ go test -v -run=TestNormalPrinter_Print . 
=== RUN   TestNormalPrinter_Print 
Hello io.Writer 
--- PASS: TestNormalPrinter_Print (0.00s) 
PASS 
ok

```

在前面的输出中，您可以看到`PrinterImpl1`结构写入`stdout`的`Hello io.Writer`消息。我们可以认为这个检查已经通过了：

最后，`PackPrinter`方法类似于`NormalPrinter`，但只是在每条消息前加上文本`Message from Packt:`：

```go
type PacktPrinter struct { 
  Msg     string 
  Printer PrinterAPI 
} 

func (c *PacktPrinter) Print() error { 
  c.Printer.PrintMessage(fmt.Sprintf("Message from Packt: %s", c.Msg)) 
  return nil 
} 

```

就像`NormalPrinter`方法一样，我们接受了`Msg`字符串和`PrinterAPI`实现，存储在`Printer`字段中。然后，我们使用`fmt.Sprintf`方法来组合一个新的字符串，其中包含文本`Message from Packt:`和提供的消息。我们取得组合的文本，并将其传递给存储在`PacktPrinter`结构的`Printer`字段中的`PrinterAPI`的`PrintMessage`方法：

```go
$ go test -v -run=TestPacktPrinter_Print .
=== RUN   TestPacktPrinter_Print
Message from Packt: Hello io.Writer
--- PASS: TestPacktPrinter_Print (0.00s)
PASS
ok

```

同样，您可以看到使用`PrinterImpl1`写入`stdout`的结果，文本为`Message from Packt: Hello io.Writer`。这最后的测试应该覆盖桥接模式中的所有代码。正如您之前所见，您可以使用`-cover`标志来检查覆盖率：

```go
$ go test -cover .
ok      
2.622s  coverage: 100.0% of statements

```

哇！100%的覆盖率-看起来不错。然而，这并不意味着代码是完美的。我们还没有检查消息的内容是否为空，也许这是应该避免的，但这不是我们的要求的一部分，这也是一个重要的观点。仅仅因为某个功能不在需求或验收标准中，并不意味着它不应该被覆盖。

## 使用桥接模式重用一切

通过桥接模式，我们学会了如何将对象及其实现与`PrintMessage`方法解耦。这样，我们可以重用其抽象以及其实现。我们可以随意交换打印机抽象以及打印机 API，而不影响用户代码。

我们还尽量保持事情尽可能简单，但我相信您已经意识到，所有`PrinterAPI`接口的实现都可以使用工厂来创建。这将是非常自然的，您可能会发现许多实现都遵循了这种方法。然而，我们不应该陷入过度设计，而应该分析每个问题，以精确地设计其需求，并找到创建可重用、可维护和*可读*源代码的最佳方式。可读的代码通常被遗忘，但如果没有人能够理解和维护它，那么强大而不耦合的源代码就是无用的。这就像十世纪的书籍一样--它可能是一部宝贵的故事，但如果我们难以理解它的语法，那就会非常令人沮丧。

# 总结

在本章中，我们已经看到了组合的力量，以及 Go 语言如何利用它的本质。我们已经看到适配器模式可以帮助我们通过在两个不兼容的接口之间使用“适配器”对象来使它们一起工作。同时，我们在 Go 语言的源代码中看到了一些真实的例子，语言的创建者使用了这种设计模式来改进标准库中某个特定部分的可能性。最后，我们已经看到了桥接模式及其可能性，允许我们在对象和它们的实现之间创建可完全重用的交换结构。

此外，在整个章节中，我们一直在使用组合设计模式，不仅仅是在解释它时。我们之前提到过它，但设计模式经常彼此使用。我们使用纯粹的组合而不是嵌入来增加可读性，但是，正如你已经学到的，根据需要可以互换使用两者。在接下来的章节中，我们将继续使用组合模式，因为它是构建 Go 编程语言中关系的基础。
