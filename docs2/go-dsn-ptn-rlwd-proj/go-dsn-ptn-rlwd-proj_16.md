# 第二章. 创建型模式 - 单例、建造者、工厂、原型和抽象工厂设计模式

我们定义了两种类型的汽车——豪华型和家庭型。汽车工厂必须返回第一组我们将要介绍的设计模式是创建型模式。正如其名所示，它将创建对象的常见实践分组，因此对象创建对需要这些对象的用户来说更加封装。主要来说，创建型模式试图为用户提供现成的对象，而不是要求他们创建，这在某些情况下可能是复杂的，或者这可能会使你的代码与应该在接口中定义的功能的具体实现耦合。

# 单例设计模式 - 在整个程序中有一个唯一的类型实例

你是否曾经为软件工程师做过面试？有趣的是，当你问他们关于设计模式时，超过 80%的人会提到**单例**设计模式。这是为什么？也许是因为它是使用最广泛的设计模式之一，或者是最容易理解的一种。我们之所以从创建型设计模式开始，正是因为它易于理解。

## 描述

单例模式易于记忆。正如其名所示，它将为你提供一个对象的单一实例，并保证没有重复。

在第一次调用使用实例时，它会被创建，然后在整个应用程序中需要使用该特定行为的所有部分之间被重复使用。

你会在许多不同的场景中使用单例模式。例如：

+   当你想使用相同的连接到数据库进行每个查询时

+   当你打开到服务器的**安全外壳**（**SSH**）连接以执行一些任务，并且不想为每个任务重新打开连接时

+   如果你需要限制对某些变量或空间的访问，你可以使用单例作为进入该变量的门（我们将在接下来的章节中看到，在 Go 中使用通道实现这一点更为可行）

+   如果你需要限制对某些地方的调用次数，你可以创建一个单例实例在可接受的窗口内进行调用

可能性是无限的，我们只是提到了其中的一些。

## 目标

作为一般指南，我们认为当以下规则适用时，我们会使用单例模式：

+   我们需要一个单一、共享的特定类型的值。

+   我们需要在整个程序中限制某些类型的对象创建到单个单元。

## 示例 - 一个唯一的计数器

作为必须确保只有一个实例的对象的例子，我们将编写一个在程序执行期间记录被调用次数的计数器。它不应该在乎我们有多少个计数器实例，它们都必须*计数*相同的值，并且必须在实例之间保持一致。

## 要求和验收标准

编写所描述的单一计数器有一些要求和验收标准。它们如下：

+   当之前没有创建计数器时，将创建一个新的计数器，其值为 0

+   如果计数器已经创建，则返回包含实际计数的这个实例

+   如果我们调用`AddOne`方法，计数器必须增加 1

在我们的单元测试中，我们有一个包含三个测试的场景来检查。

## 首先编写单元测试

Go 语言实现这种模式的方式与你在像 Java 或 C++这样的纯面向对象语言中找到的略有不同，在这些语言中你有静态成员。在 Go 中，没有类似静态成员的东西，但我们有包作用域来达到类似的效果。

为了设置我们的项目，我们必须在`$GOPATH/src`目录内创建一个新的文件夹。我们之前提到的通用规则是，创建一个子文件夹，包含 VCS 提供者（如 GitHub）、用户名和项目名称。

例如，在我的情况下，我使用 GitHub 作为我的 VCS，我的用户名是*sayden*，所以我将创建路径`$GOPATH/src/github.com/sayden/go-design-patterns/creational/singleton`。路径中的`go-design-patterns`实例是项目名称，创建型子文件夹也将是我们的库名称，而 singleton 是这个特定包和子文件夹的名称：

```go
mkdir -p $GOPATH/src/github.com/sayden/go-design-patterns/creational/singleton 
cd $GOPATH/src/github.com/sayden/go-design-
patterns/creational/singleton

```

在 singleton 文件夹内创建一个名为`singleton.go`的新文件，以反映包的名称，并为`singleton`类型编写以下包声明：

```go
package singleton 

type Singleton interface { 
    AddOne() int 
} 

type singleton struct { 
    count int 
} 

var instance *singleton 

func GetInstance() Singleton { 
    return nil 
} 
func (s *singleton) AddOne() int { 
    return 0 
} 

```

由于我们在编写代码时遵循 TDD（测试驱动开发）方法，让我们编写使用我们刚刚声明的函数的测试。测试将通过遵循我们之前编写的验收标准来定义。按照测试文件的惯例，我们必须创建一个与要测试的文件同名的文件，后缀为`_test.go`。这两个文件都必须位于同一个文件夹中：

```go
package singleton 

import "testing" 

func TestGetInstance(t *testing.T) { 
   counter1 := GetInstance() 

   if counter1 == nil { 
         //Test of acceptance criteria 1 failed 
         t.Error("expected pointer to Singleton after calling GetInstance(), not nil") 
   } 

   expectedCounter := counter1 
} 

```

第一个测试检查的是显而易见但同样重要的事情，在复杂的应用程序中。当我们请求计数器的实例时，我们实际上收到了一些东西。我们必须把它看作是一个创建型模式--我们将对象的创建委托给一个未知的包，这个包可能在创建或检索对象时失败。我们还把当前的计数器存储在`expectedCounter`变量中，以便稍后进行比较：

```go
currentCount := counter1.AddOne() 
if currentCount != 1 { 
     t.Errorf("After calling for the first time to count, the count must be 1 but it is %d\n", currentCount) 
} 

```

现在我们利用 Go 的零初始化特性。记住，Go 中的整数类型不能为 nil，而且我们知道，这是对计数器的第一次调用，它是一个整数类型的变量，我们也知道它是零初始化的。所以，在第一次调用`AddOne()`函数后，计数器的值必须是 1。

检查第二个条件的测试证明了`expectedConnection`变量与我们后来请求返回的连接没有不同。如果它们不同，消息`Singleton 实例必须不同`将导致测试失败：

```go
counter2 := GetInstance() 
if counter2 != expectedCounter { 
    //Test 2 failed 
    t.Error("Expected same instance in counter2 but it got a different instance") 
} 

```

最后一个测试只是用第二个实例再次计数 1。前面的结果是 1，所以现在它必须给出 2：

```go
currentCount = counter2.AddOne() 
if currentCount != 2 { 
    t.Errorf("After calling 'AddOne' using the second counter, the current count must be 2 but was %d\n", currentCount) 
} 

```

为了完成我们的测试部分，我们必须做的最后一件事是执行测试以确保它们在实现之前失败。如果其中之一没有失败，这意味着我们做错了什么，我们必须重新考虑那个特定的测试。我们必须打开终端并导航到单例包的路径以执行：

```go
$ go test -v .
=== RUN   TestGetInstance
--- FAIL: TestGetInstance (0.00s)
 singleton_test.go:9: expected pointer to Singleton after calling GetInstance(), not nil
 singleton_test.go:15: After calling for the first time to count, the count must be 1 but it is 0
 singleton_test.go:27: After calling 'AddOne' using the second counter, the current count must be 2 but was 0
FAIL
exit status 1
FAIL

```

## 实现

最后，我们必须实现单例模式。正如我们之前提到的，在 Java 或 C++等语言中，我们通常会编写一个`static`方法和一个实例来检索单例实例。在 Go 语言中，我们没有`static`关键字，但我们可以通过使用包的作用域来实现相同的结果。首先，我们创建一个`struct`，它包含我们在程序执行期间想要保证是单例的对象：

```go
package creational 

type singleton struct{ 
    count int 
} 

var instance *singleton 

func GetInstance() *singleton { 
    if instance == nil { 
        instance = new(singleton) 
    }  
    return instance 
} 

func (s *singleton) AddOne() int { 
    s.count++ 
    return s.count 
} 

```

我们必须密切关注这段代码。在 Java 或 C++等语言中，变量实例在程序开始时会初始化为 NULL。在 Go 中，你可以将结构体指针初始化为`nil`，但你不能将结构体初始化为`nil`（NULL 的等价物）。因此，`var instance *singleton`这一行定义了一个指向 Singleton 类型结构体的指针，并命名为`instance`。

我们创建了一个`GetInstance`方法，该方法检查实例是否尚未初始化（`instance == nil`），并在`instance = new(singleton)`这一行中分配的空间中创建一个实例。记住，当我们使用`new`关键字时，我们正在创建一个指向括号中类型实例的指针。

`AddOne`方法将取变量实例的计数，加 1，并返回计数器的当前值。

让我们现在再次运行我们的单元测试：

```go
$ go test -v -run=GetInstance
=== RUN   TestGetInstance
--- PASS: TestGetInstance (0.00s)
PASS
ok

```

## 关于单例设计模式的简要介绍

我们已经看到了单例模式的一个非常简单的例子，部分应用于某些情况，即一个简单的计数器。只需记住，单例模式将赋予你在应用程序中拥有某个结构体唯一实例的能力，并且没有任何包可以创建这个结构体的任何克隆。

使用单例，你还可以隐藏创建对象的复杂性，如果它需要一些计算，以及每次需要该实例时都创建它的陷阱，如果它们都是相似的。所有这些代码编写、检查变量是否已存在以及存储的工作都被封装在单例中，如果你使用全局变量，你就不需要重复它。

在这里，我们学习的是单线程上下文中的经典单例实现。当我们到达关于并发的章节时，我们将看到并发单例实现，因为这种实现不是线程安全的！

# 构建者设计模式 - 重新使用算法来创建接口的多个实现

谈到 **创建型** 设计模式，有一个 **Builder** 设计模式看起来非常语义化。Builder 模式帮助我们构建复杂对象，而不直接实例化它们的 struct，或者编写它们所需的逻辑。想象一个可能拥有几十个字段的对象，这些字段本身也是更复杂的 struct。现在想象一下，你有许多具有这些特征的对象，并且可能还有更多。我们不想在只需要使用对象的包中编写创建所有这些对象的逻辑。

## 描述

实例创建可以像提供开闭花括号 `{}` 并将实例留为零值那样简单，也可以像需要执行一些 API 调用、检查状态并为其实例字段创建对象的对象那样复杂。你也可以有一个由许多对象组成的对象，这在 Go 中非常地道，因为它不支持继承。

同时，你还可以使用相同的技巧来创建许多类型的对象。例如，你将使用几乎相同的技巧来构建汽车，就像你构建公共汽车一样，只是它们的大小和座位数不同，那么我们为什么不重用构建过程呢？这就是建造者模式发挥作用的地方。

## 目标

建造者设计模式试图：

+   抽象复杂的创建，以便对象创建与对象使用者分离

+   通过填充字段和创建嵌入对象逐步创建对象

+   在许多对象之间重用对象创建算法

## 示例 - 车辆制造

建造者设计模式通常被描述为导演、几个 Builder 和他们所构建的产品之间的关系。继续我们的汽车示例，我们将创建一个车辆 Builder。创建车辆（产品）的过程（通常被称为算法）对于每一种类型的车辆或多或少是相同的——选择车辆类型、组装结构、放置轮子，以及放置座位。如果你这么想，你可以用这个描述来构建汽车和摩托车（两个 Builder），因此我们正在重用这个描述来在制造中创建汽车。在我们的示例中，导演由 `ManufacturingDirector` 类型表示。

## 需求和验收标准

就我们所描述的，我们必须处理一个类型为 `Car` 和 `Motorbike` 的 Builder 以及一个唯一的导演 `ManufacturingDirector` 来接收 builders 并构建产品。因此，一个 `Vehicle` Builder 示例的要求如下：

+   我必须有一个制造类型，它可以构建车辆所需的一切

+   当使用汽车 Builder 时，必须返回具有四个轮子、五个座位以及定义为 `Car` 的结构的 `VehicleProduct`

+   当使用摩托车 Builder 时，必须返回具有两个轮子、两个座位以及定义为 `Motorbike` 的结构的 `VehicleProduct`

+   由任何 `BuildProcess` Builder 构建的 `VehicleProduct` 必须允许修改

## 车辆构建器的单元测试

根据先前的验收标准，我们将创建一个导演变量，即 `ManufacturingDirector` 类型，以使用代表汽车和摩托车的产品构建器变量的构建过程。导演负责构建对象，但构建器是返回实际车辆的那部分。因此，我们的构建器声明将如下所示：

```go
package creational 

type BuildProcess interface { 
    SetWheels() BuildProcess 
    SetSeats() BuildProcess 
    SetStructure() BuildProcess 
    GetVehicle() VehicleProduct 
} 

```

此前定义的接口定义了构建车辆所需的步骤。每个构建器都必须实现此 `interface`，才能被制造使用。在每一个 `Set` 步骤中，我们返回相同的构建过程，因此我们可以在同一个语句中链接各种步骤，就像我们稍后将要看到的那样。最后，我们还需要一个 `GetVehicle` 方法来从构建器中检索 `Vehicle` 实例：

```go
type ManufacturingDirector struct {} 

func (f *ManufacturingDirector) Construct() { 
    //Implementation goes here 
} 

func (f *ManufacturingDirector) SetBuilder(b BuildProcess) { 
    //Implementation goes here 
} 

```

`ManufacturingDirector` 导演变量是负责接受构建器的。它有一个 `Construct` 方法，将使用存储在 `Manufacturing` 中的构建器，并重现所需的步骤。`SetBuilder` 方法将允许我们更改 `Manufacturing` 导演中正在使用的构建器：

```go
type VehicleProduct struct { 
    Wheels    int 
    Seats     int 
    Structure string 
} 

```

产品是我们使用制造时想要检索的最终对象。在这种情况下，一辆车由轮子、座位和结构组成：

```go
type CarBuilder struct {} 

func (c *CarBuilder) SetWheels() BuildProcess { 
    return nil 
} 

func (c *CarBuilder) SetSeats() BuildProcess { 
    return nil 
} 

func (c *CarBuilder) SetStructure() BuildProcess { 
    return nil 
} 

func (c *CarBuilder) Build() VehicleProduct { 
    return VehicleProduct{} 
} 

```

第一个构建器是 `Car` 构建器。它必须实现 `BuildProcess` 接口中定义的每个方法。这就是我们为这个特定的构建器设置信息的地方：

```go
type BikeBuilder struct {} 

func (b *BikeBuilder) SetWheels() BuildProcess { 
    return nil 
} 

func (b *BikeBuilder) SetSeats() BuildProcess { 
    return nil 
} 

func (b *BikeBuilder) SetStructure() BuildProcess { 
    return nil 
} 

func (b *BikeBuilder) Build() VehicleProduct { 
    return VehicleProduct{} 
} 

```

`Motorbike` 结构必须与 `Car` 结构相同，因为它们都是构建器实现，但请注意，构建每个的过程可能非常不同。有了这个对象声明，我们可以创建以下测试：

```go
package creational 

import "testing" 

func TestBuilderPattern(t *testing.T) { 
    manufacturingComplex := ManufacturingDirector{} 

    carBuilder := &CarBuilder{} 
    manufacturingComplex.SetBuilder(carBuilder) 
    manufacturingComplex.Construct() 

    car := carBuilder.Build() 

    //code continues here... 

```

我们将从 `Manufacturing` 导演和 `Car` 构建器开始，以满足前两个验收标准。在先前的代码中，我们创建了一个 `Manufacturing` 导演，它将在测试期间负责创建每辆车。在创建 `Manufacturing` 导演后，我们创建了一个 `CarBuilder`，然后通过使用 `SetBuilder` 方法将其传递给制造。一旦 `Manufacturing` 导演知道现在要构建什么，我们就可以调用 `Construct` 方法来使用 `CarBuilder` 创建 `VehicleProduct`。最后，一旦我们有了汽车的所有部件，我们就在 `CarBuilder` 上调用 `GetVehicle` 方法来检索一个 `Car` 实例：

```go
if car.Wheels != 4 { 
    t.Errorf("Wheels on a car must be 4 and they were %d\n", car.Wheels) 
} 

if car.Structure != "Car" { 
    t.Errorf("Structure on a car must be 'Car' and was %s\n", car.Structure) 
} 

if car.Seats != 5 { 
    t.Errorf("Seats on a car must be 5 and they were %d\n", car.Seats) 
} 

```

我们编写了三个小测试来检查结果是否为汽车。我们检查了汽车有四个轮子，结构描述为 `Car`，座位数为五。我们有足够的数据来执行测试并确保它们失败，这样我们就可以认为它们是可靠的：

```go
$ go test -v -run=TestBuilder .
=== RUN   TestBuilderPattern
--- FAIL: TestBuilderPattern (0.00s)
 builder_test.go:15: Wheels on a car must be 4 and they were 0
 builder_test.go:19: Structure on a car must be 'Car' and was
 builder_test.go:23: Seats on a car must be 5 and they were 0
FAIL

```

完美！现在我们将为 `Motorbike` 构建器创建测试，以覆盖第三和第四个验收标准：

```go
bikeBuilder := &BikeBuilder{} 

manufacturingComplex.SetBuilder(bikeBuilder) 
manufacturingComplex.Construct() 

motorbike := bikeBuilder.GetVehicle() 
motorbike.Seats = 1 

if motorbike.Wheels != 2 { 
    t.Errorf("Wheels on a motorbike must be 2 and they were %d\n", motorbike.Wheels) 
} 

if motorbike.Structure != "Motorbike" { 
    t.Errorf("Structure on a motorbike must be 'Motorbike' and was %s\n", motorbike.Structure) 
} 

```

上述代码是汽车测试的延续。正如你所见，我们通过传递`Motorbike`构建者来重用之前创建的制造过程，现在创建摩托车。然后我们再次点击`construct`按钮来创建必要的部件，并调用构建者的`GetVehicle`方法来检索摩托车实例。

快速浏览一下，因为我们已经将这款特定摩托车的默认座位数更改为 1。我们在这里想要展示的是，即使有构建者，你也必须能够更改返回实例中的默认信息以适应某些特定需求。由于我们手动设置了车轮，所以我们将不会测试这个功能。

重新运行测试会触发预期的行为：

```go
$ go test -v -run=Builder .
=== RUN   TestBuilderPattern
--- FAIL: TestBuilderPattern (0.00s)
 builder_test.go:15: Wheels on a car must be 4 and they were 0
 builder_test.go:19: Structure on a car must be 'Car' and was
 builder_test.go:23: Seats on a car must be 5 and they were 0
 builder_test.go:35: Wheels on a motorbike must be 2 and they were 0
 builder_test.go:39: Structure on a motorbike must be 'Motorbike' and was
FAIL

```

## 实现

我们将开始实现制造。正如我们之前所说的（以及我们在单元测试中设置的），`Manufacturing`导演必须接受一个构建者并使用提供的构建者构建一个车辆。为了回忆，`BuildProcess`接口将定义构建任何车辆所需的共同步骤，并且`Manufacturing`导演必须接受构建者并与他们一起构建车辆：

```go
package creational 

type ManufacturingDirector struct { 
    builder BuildProcess 
} 

func (f *ManufacturingDirector) SetBuilder(b BuildProcess) { 
    f.builder = b 
} 

func (f *ManufacturingDirector) Construct() { 
    f.builder.SetSeats().SetStructure().SetWheels() 
} 

```

我们的`ManufacturingDirector`需要一个字段来存储正在使用的构建者；这个字段将被称为`builder`。`SetBuilder`方法将用提供的参数中的构建者替换存储的构建者。最后，更仔细地看看`Construct`方法。它接受存储的构建者并重现创建某种未知类型的完整车辆的`BuildProcess`方法。正如你所见，我们通过在每个调用中返回`BuildProcess`接口，将所有设置调用放在同一行，从而使代码更加紧凑：

### 小贴士

你是否意识到构建模式中的导演实体也是一个清晰的单例模式候选？在某些场景中，可能只有导演的一个实例是关键的，这就是为什么你只为构建者的导演创建单例模式。设计模式组合是一个非常常见且非常强大的技术！

```go
type CarBuilder struct { 
    v VehicleProduct 
} 

func (c *CarBuilder) SetWheels() BuildProcess { 
    c.v.Wheels = 4 
    return c 
} 

func (c *CarBuilder) SetSeats() BuildProcess { 
    c.v.Seats = 5 
    return c 
} 

func (c *CarBuilder) SetStructure() BuildProcess { 
    c.v.Structure = "Car" 
    return c 
} 

func (c *CarBuilder) GetVehicle() VehicleProduct { 
    return c.v 
} 

```

这里是我们的第一个构建者，`car`构建者。构建者需要存储一个`VehicleProduct`对象，在这里我们将其命名为`v`。然后我们设置汽车在业务中具有的特定需求——四个车轮、五个座位和一个定义为`Car`的结构。在`GetVehicle`方法中，我们只需返回构建者中存储的`VehicleProduct`，它必须已经被`ManufacturingDirector`类型构建。

```go
type BikeBuilder struct { 
    v VehicleProduct 
} 

func (b *BikeBuilder) SetWheels() BuildProcess { 
    b.v.Wheels = 2 
    return b 
} 

func (b *BikeBuilder) SetSeats() BuildProcess { 
    b.v.Seats = 2 
    return b 
} 

func (b *BikeBuilder) SetStructure() BuildProcess { 
    b.v.Structure = "Motorbike" 
    return b 
} 

func (b *BikeBuilder) GetVehicle() VehicleProduct { 
    return b.v 
} 

```

`Motorbike`构建者与`car`构建者相同。我们定义摩托车有两个车轮、两个座位和一个称为`Motorbike`的结构。它与`car`对象非常相似，但想象一下，你想要区分运动摩托车（只有一个座位）和巡航摩托车（有两个座位）。你可以简单地为运动摩托车创建一个新的结构，该结构实现了构建过程。

你可以看到这是一个重复的模式，但在`BuildProcess`接口的每个方法的范围内，你可以封装尽可能多的复杂性，这样用户就不需要了解对象创建的细节。

在定义了所有对象之后，让我们再次运行测试：

```go
=== RUN   TestBuilderPattern
--- PASS: TestBuilderPattern (0.00s)
PASS
ok  _/home/mcastro/pers/go-design-patterns/creational 0.001s

```

干得好！想想看，向`ManufacturingDirector`导演添加新车辆是多么容易，只需创建一个封装新车辆数据的新的类。例如，让我们添加一个`BusBuilder`结构体：

```go
type BusBuilder struct { 
    v VehicleProduct 
} 

func (b *BusBuilder) SetWheels() BuildProcess { 
    b.v.Wheels = 4*2 
    return b 
} 

func (b *BusBuilder) SetSeats() BuildProcess { 
    b.v.Seats = 30 
    return b 
} 

func (b *BusBuilder) SetStructure() BuildProcess { 
    b.v.Structure = "Bus" 
    return b 
} 

func (b *BusBuilder) GetVehicle() VehicleProduct { 
    return b.v 
} 

```

就这些；你的`ManufacturingDirector`将准备好通过遵循 Builder 设计模式使用新产品。

## 总结 Builder 设计模式

Builder 设计模式通过使用导演使用的通用构建算法，帮助我们维护不可预测数量的产品。构建过程始终从产品的用户那里抽象出来。

同时，当我们源代码的新手需要向*流水线*添加新产品时，有一个定义好的构建模式很有帮助。`BuildProcess`接口指定了他必须遵守的规范才能成为可能的构建者之一。

然而，当你不确定算法是否将更加或更少稳定时，尽量避免使用 Builder 模式，因为任何对这个接口的微小更改都将影响所有构建者，如果你添加一个某些构建者需要而其他构建者不需要的新方法，可能会很尴尬。

# 工厂方法 - 委托不同类型支付的创建

工厂方法模式（或简称，Factory）可能是工业界第二广为人知和使用的设计模式。它的目的是将用户从需要为特定目的（如从网络服务或数据库检索某些值）实现的结构知识中抽象出来。用户只需要一个提供此值的接口。通过将此决策委托给工厂，这个工厂可以提供一个符合用户需求的接口。如果需要，它还可以简化对底层类型实现进行降级或升级的过程。

## 描述

当使用工厂方法设计模式时，我们获得了一个额外的封装层，这样我们的程序就可以在受控环境中增长。使用工厂方法，我们将创建对象族的任务委托给不同的包或对象，从而抽象出我们可能使用的对象池的知识。想象一下，如果你想通过旅行社组织假期，你不需要处理酒店和旅行，你只需告诉旅行社你感兴趣的目的地，以便他们为你提供所需的一切。旅行社代表了一个旅行工厂。

## 目标

在之前的描述之后，以下关于工厂方法设计模式的目标应该对你来说已经很清晰了：

+   将结构新实例的创建委托给程序的另一部分

+   在接口级别而不是具体实现级别上工作

+   将对象家族分组以获得家族对象创建器

## 示例 - 商店支付方法的工厂

对于我们的示例，我们将实现一个支付方法工厂，它将为我们提供在商店支付的不同方式。一开始，我们将有两种支付方式——现金和信用卡。我们还将有一个带有`Pay`方法的接口，任何想要用作支付方法的`struct`都必须实现此接口。

## 验收标准

使用前面的描述，验收标准的以下要求如下：

+   为每个支付方式提供一个名为`Pay`的通用方法

+   能够将支付方法的创建委托给工厂

+   能够通过仅向工厂方法添加来向库中添加更多支付方式

## 第一个单元测试

工厂方法有一个非常简单的结构；我们只需要确定我们存储了多少接口的实现，然后提供一个`GetPaymentMethod`方法，你可以传递一个支付类型作为参数：

```go
type PaymentMethod interface { 
    Pay(amount float32) string 
} 

```

前面的行定义了支付方法的接口。它们定义了在商店进行支付的方式。工厂方法将返回实现此接口的类型实例：

```go
const ( 
    Cash      = 1 
    DebitCard = 2 
) 

```

我们必须将工厂识别的支付方法定义为常量，这样我们就可以从包外部调用和检查可能的支付方法。

```go
func GetPaymentMethod(m int) (PaymentMethod, error) { 
    return nil, errors.New("Not implemented yet") 
} 

```

前面的代码是创建我们对象的函数。它返回一个指针，该指针必须有一个实现`PaymentMethod`接口的对象，如果请求一个未注册的方法，则返回错误。

```go
type CashPM struct{} 
type DebitCardPM struct{} 

func (c *CashPM) Pay(amount float32) string { 
    return "" 
} 

func (c *DebitCardPM) Pay(amount float32) string { 
    return "" 
} 

```

为了完成工厂的声明，我们创建了两种支付方式。正如你所见，`CashPM`和`DebitCardPM`结构体通过声明一个方法，`Pay(amount float32) string`来实现`PaymentMethod`接口。返回的字符串将包含有关支付的信息。

通过这个声明，我们将开始编写第一个验收标准的测试：拥有一个通用的方法来检索实现`PaymentMethod`接口的对象：

```go
package creational 

import ( 
    "strings" 
    "testing" 
) 

func TestCreatePaymentMethodCash(t *testing.T) { 
    payment, err := GetPaymentMethod(Cash) 
    if err != nil { 
        t.Fatal("A payment method of type 'Cash' must exist") 
    } 

    msg := payment.Pay(10.30) 
    if !strings.Contains(msg, "paid using cash") { 
        t.Error("The cash payment method message wasn't correct") 
    } 
    t.Log("LOG:", msg) 
} 

```

现在，我们将不得不将测试分散到几个测试函数中。`GetPaymentMethod`是一个用于检索支付方法的通用方法。我们使用在实现文件中定义的常量`Cash`（如果我们在这个包的作用域之外使用这个常量，我们将使用包的名字作为前缀，所以语法将是`creational.Cash`）。我们还检查在请求支付方式时是否收到了错误。注意，如果我们请求支付方式时收到错误，我们调用`t.Fatal`来停止测试的执行；如果我们像之前的测试一样只调用`t.Error`，那么在尝试访问 nil 对象的`Pay`方法时，我们会在下一行遇到问题，我们的测试将崩溃执行。我们继续使用接口的`Pay`方法，通过传递 10.30 作为金额。返回的消息将必须包含文本`paid using cash`。`t.Log(string)`方法是测试中的一个特殊方法。这个结构允许我们在运行测试时通过传递`-v`标志来写入一些日志。

```go
func TestGetPaymentMethodDebitCard(t *testing.T) { 
    payment, err = GetPaymentMethod(Debit9Card) 

    if err != nil { 
        t.Error("A payment method of type 'DebitCard' must exist")
    } 

    msg = payment.Pay(22.30) 

    if !strings.Contains(msg, "paid using debit card") { 
        t.Error("The debit card payment method message wasn't correct") 
    } 

    t.Log("LOG:", msg) 
}
```

我们使用借记卡方法重复相同的操作。我们要求使用常量`DebitCard`定义的支付方式，当使用借记卡支付时，返回的消息必须包含`paid using debit card`字符串。

```go

func TestGetPaymentMethodNonExistent(t *testing.T) { 
    payment, err = GetPaymentMethod(20) 

    if err == nil { 
        t.Error("A payment method with ID 20 must return an error") 
    } 
    t.Log("LOG:", err) 
}
```

最后，我们将测试请求一个不存在的支付方式的情况（用数字 20 表示，它在工厂中不匹配任何已识别的常量）。我们将检查在请求未知支付方式时是否返回了错误消息（任何错误）。

让我们检查是否所有测试都失败了：

```go
$ go test -v -run=GetPaymentMethod .
=== RUN   TestGetPaymentMethodCash
--- FAIL: TestGetPaymentMethodCash (0.00s)
 factory_test.go:11: A payment method of type 'Cash' must exist
=== RUN   TestGetPaymentMethodDebitCard
--- FAIL: TestGetPaymentMethodDebitCard (0.00s)
 factory_test.go:24: A payment method of type 'DebitCard' must exist
=== RUN   TestGetPaymentMethodNonExistent
--- PASS: TestGetPaymentMethodNonExistent (0.00s)
 factory_test.go:38: LOG: Not implemented yet
FAIL
exit status 1
FAIL

```

正如你在本例中看到的，我们只能看到返回`PaymentMethod`接口失败的测试。在这种情况下，我们只需实现代码的一部分，然后继续之前继续测试之前再次进行测试。

## 实现

我们将从`GetPaymentMethod`方法开始。它必须接收一个整数，该整数与同一文件中定义的某个常量匹配，以便知道它应该返回哪种实现。

```go
package creational 

import ( 
    "errors" 
    "fmt" 
) 

type PaymentMethod interface { 
    Pay(amount float32) string 
} 

const ( 
    Cash      = 1 
    DebitCard = 2 
) 

type CashPM struct{} 
type DebitCardPM struct{} 

func GetPaymentMethod(m int) (PaymentMethod, error) { 
    switch m { 
        case Cash: 
        return new(CashPM), nil 
        case DebitCard: 
        return new(DebitCardPM), nil 
        default: 
        return nil, errors.New(fmt.Sprintf("Payment method %d not recognized\n", m)) 
    } 
} 

```

我们使用普通的 switch 来检查参数`m`（方法）的内容。如果它与已知的任何方法（现金或借记卡）匹配，它将返回它们的新实例。否则，它将返回 nil 和一个错误，表明未识别支付方式。现在我们可以再次运行我们的测试来检查单元测试的第二部分：

```go
$go test -v -run=GetPaymentMethod .
=== RUN   TestGetPaymentMethodCash
--- FAIL: TestGetPaymentMethodCash (0.00s)
 factory_test.go:16: The cash payment method message wasn't correct
 factory_test.go:18: LOG:
=== RUN   TestGetPaymentMethodDebitCard
--- FAIL: TestGetPaymentMethodDebitCard (0.00s)
 factory_test.go:28: The debit card payment method message wasn't correct
 factory_test.go:30: LOG:
=== RUN   TestGetPaymentMethodNonExistent
--- PASS: TestGetPaymentMethodNonExistent (0.00s)
 factory_test.go:38: LOG: Payment method 20 not recognized
FAIL
exit status 1
FAIL

```

现在，我们没有收到找不到支付方式类型错误的消息。相反，当它尝试使用它覆盖的任何方法时，我们收到一个`message not correct`错误。我们还去掉了在请求未知支付方式时返回的`Not implemented`消息。现在让我们实现结构体：

```go
type CashPM struct{} 
type DebitCardPM struct{} 

func (c *CashPM) Pay(amount float32) string { 
     return fmt.Sprintf("%0.2f paid using cash\n", amount) 
} 

func (c *DebitCardPM) Pay(amount float32) string { 
     return fmt.Sprintf("%#0.2f paid using debit card\n", amount) 
} 

```

我们只是获取数量，并以格式良好的消息打印出来。使用这种实现方式，现在所有的测试都将通过：

```go
$ go test -v -run=GetPaymentMethod .
=== RUN   TestGetPaymentMethodCash
--- PASS: TestGetPaymentMethodCash (0.00s)
 factory_test.go:18: LOG: 10.30 paid using cash
=== RUN   TestGetPaymentMethodDebitCard
--- PASS: TestGetPaymentMethodDebitCard (0.00s)
 factory_test.go:30: LOG: 22.30 paid using debit card
=== RUN   TestGetPaymentMethodNonExistent
--- PASS: TestGetPaymentMethodNonExistent (0.00s)
 factory_test.go:38: LOG: Payment method 20 not recognized
PASS
ok

```

你看到 `LOG` 消息了吗？它们不是错误，我们只是打印出在使用测试包时接收到的信息。除非你传递 `-v` 标志给测试命令，否则可以省略这些消息：

```go
$ go test -run=GetPaymentMethod .
ok

```

## 将借记卡方法升级到新平台

现在想象一下，由于某种原因，你的 `DebitCard` 支付方法已经改变，你需要一个新的结构体来适应它。为了实现这个场景，你只需要在用户请求 `DebitCard` 支付方法时创建新的结构体并替换旧的即可：

```go
type CreditCardPM struct {} 
 func (d *CreditCardPM) Pay(amount float32) string { 
   return fmt.Sprintf("%#0.2f paid using new credit card implementation\n", amount) 
} 

```

这是我们的新类型，它将替换 `DebitCardPM` 结构体。`CreditCardPM` 实现了与借记卡相同的 `PaymentMethod` 接口。我们没有删除之前的，以防将来需要它。唯一的区别在于返回的消息，现在包含了关于新类型的信息。我们还需要修改获取支付方法的方法：

```go
func GetPaymentMethod(m int) (PaymentMethod, error) { 
    switch m { 
        case Cash: 
        return new(CashPM), nil 
        case DebitCard: 
        return new(CreditCardPM), nil 
        default: 
        return nil, errors.New(fmt.Sprintf("Payment method %d not recognized\n", m)) 
   } 
} 

```

唯一的修改是在创建新借记卡的那一行，现在它指向新创建的结构体。让我们运行测试看看一切是否仍然正确：

```go
$ go test -v -run=GetPaymentMethod .
=== RUN   TestGetPaymentMethodCash
--- PASS: TestGetPaymentMethodCash (0.00s)
 factory_test.go:18: LOG: 10.30 paid using cash
=== RUN   TestGetPaymentMethodDebitCard
--- FAIL: TestGetPaymentMethodDebitCard (0.00s)
 factory_test.go:28: The debit card payment method message wasn't correct
 factory_test.go:30: LOG: 22.30 paid using new debit card implementation
=== RUN   TestGetPaymentMethodNonExistent
--- PASS: TestGetPaymentMethodNonExistent (0.00s)
 factory_test.go:38: LOG: Payment method 20 not recognized
FAIL
exit status 1
FAIL

```

哎呀！出问题了。使用信用卡支付时预期的消息与返回的消息不匹配。这意味着我们的代码不正确吗？一般来说，是的，你不应该修改你的测试来让程序工作。在定义测试时，你也应该意识到不要定义得过多，因为你可能会在测试中产生一些代码中没有的耦合。由于消息限制，我们有几种语法上正确的消息可能性，所以我们将它改为以下内容：

```go
return fmt.Sprintf("%#0.2f paid using debit card (new)\n", amount) 

```

我们现在再次运行测试：

```go
$ go test -v -run=GetPaymentMethod .
=== RUN   TestGetPaymentMethodCash
--- PASS: TestGetPaymentMethodCash (0.00s)
 factory_test.go:18: LOG: 10.30 paid using cash
=== RUN   TestGetPaymentMethodDebitCard
--- PASS: TestGetPaymentMethodDebitCard (0.00s)
 factory_test.go:30: LOG: 22.30 paid using debit card (new)
=== RUN   TestGetPaymentMethodNonExistent
--- PASS: TestGetPaymentMethodNonExistent (0.00s)
 factory_test.go:38: LOG: Payment method 20 not recognized
PASS
ok

```

一切又恢复正常了。这只是一个如何编写良好单元测试的小例子。当我们想要检查借记卡支付方法返回的消息是否包含“使用借记卡支付”字符串时，我们可能过于严格了，最好分别检查这些单词或者为返回的消息定义更好的格式。

## 我们关于工厂方法学到的东西

使用工厂方法模式，我们已经学会了如何将对象家族分组，使得它们的实现超出了我们的范围。我们还学会了在需要升级已使用结构体的实现时应该做什么。最后，我们看到了如果你不想将自己绑定到与测试直接无关的某些实现上，测试必须谨慎编写。

# 抽象工厂 - 工厂之工厂

在了解了工厂设计模式后，我们在这个案例中将相关的支付方法家族分组，有人可能会想——如果我以更结构化的家族层次结构来分组对象家族会怎样？

## 描述

抽象工厂设计模式是一个新的分组层，用于实现更大的（更复杂的）组合对象，它通过其接口使用。将对象分组在家族中以及将家族分组在一起的想法是拥有可以互换且更容易增长的庞大工厂。在开发的早期阶段，与工厂和抽象工厂一起工作也比等待所有具体实现完成再开始你的代码要容易。此外，除非你知道你的对象库存对于特定字段将非常大，并且可以很容易地分组到家族中，否则你不会从一开始就编写抽象工厂。

## 目标

当你的对象数量增长到如此之多，以至于创建一个独特的点来获取它们似乎成为获得运行时对象创建灵活性的唯一方式时，将相关对象家族分组是非常方便的。以下关于抽象工厂方法的目标必须对你清晰明了：

+   为返回所有工厂的公共接口的工厂方法提供一层新的封装

+   将常见的工厂组合成一个*超级工厂*（也称为工厂的工厂）

## 再来一次，车辆工厂的例子？

对于我们的例子，我们将重用我们在构建器设计模式中创建的工厂。我们希望通过使用不同的方法来展示解决相同问题的相似性，以便你可以看到每种方法的优点和缺点。这将展示 Go 中隐式接口的力量，因为我们几乎不需要做任何修改。最后，我们将创建一个新的工厂来创建运输订单。

## 接受标准

以下是用`Vehicle`对象的工厂方法使用时的接受标准：

+   我们必须使用由抽象工厂返回的工厂来检索一个`Vehicle`对象。

+   车辆必须是一个具体实现`Motorbike`或`Car`的实例，该实例实现了两个接口（`Vehicle`和`Car`或`Vehicle`和`Motorbike`）。

## 单元测试

这将是一个很长的例子，所以请注意。我们将有以下实体：

+   **Vehicle**：我们工厂中所有对象必须实现的接口：

    +   **Motorbike**：一个用于运动型（一个座位）和巡航型（两个座位）摩托车的接口。

    +   **Car**：一个用于豪华型（带四个车门）和家庭型（带五个车门）汽车的接口。

+   **VehicleFactory**：一个接口（抽象工厂），用于检索实现`VehicleFactory`方法的工厂：

    +   **Motorbike** 工厂：一个实现`VehicleFactory`接口的工厂，用于返回实现`Vehicle`和`Motorbike`接口的车辆。

    +   **Car** 工厂：另一个实现`VehicleFactory`接口的工厂，用于返回实现`Vehicle`和`Car`接口的车辆。

为了清晰起见，我们将每个实体分开到不同的文件中。我们将从`Vehicle`接口开始，它将位于`vehicle.go`文件中：

```go
package abstract_factory 

type Vehicle interface { 
    NumWheels() int 
    NumSeats() int 
} 

```

`Car` 和 `Motorbike` 接口将分别位于 `car.go` 和 `motorbike.go` 文件中：

```go
// Package abstract_factory file: car.go 
package abstract_factory 

type Car interface { 
    NumDoors() int 
} 
// Package abstract_factory file: motorbike.go 
package abstract_factory 

type Motorbike interface { 
    GetMotorbikeType() int 
} 

```

我们还有一个最后的接口，每个工厂都必须实现。这将在 `vehicle_factory.go` 文件中：

```go
package abstract_factory 

type VehicleFactory interface { 
    NewVehicle(v int) (Vehicle, error) 
} 

```

因此，现在我们将声明汽车工厂。它必须实现之前定义的 `VehicleFactory` 接口以返回 `Vehicles` 实例：

```go
const ( 
    LuxuryCarType = 1 
    FamilyCarType = 2 
) 

type CarFactory struct{} 
func (c *CarFactory) NewVehicle(v int) (Vehicle, error) { 
    switch v { 
        case LuxuryCarType: 
        return new(LuxuryCar), nil 
        case FamilyCarType: 
        return new(FamilyCar), nil 
        default: 
        return nil, errors.New(fmt.Sprintf("Vehicle of type %d not recognized\n", v)) 
    } 
} 

```

我们已定义了两种类型的汽车——豪华型和家庭型。汽车工厂必须返回实现 `Car` 和 `Vehicle` 接口的汽车，因此我们需要两个具体的实现：

```go
//luxury_car.go 
package abstract_factory 

type LuxuryCar struct{} 

func (*LuxuryCar) NumDoors() int { 
    return 4 
} 
func (*LuxuryCar) NumWheels() int { 
    return 4 
} 
func (*LuxuryCar) NumSeats() int { 
    return 5 
} 

package abstract_factory 

type FamilyCar struct{} 

func (*FamilyCar) NumDoors() int { 
    return 5 
} 
func (*FamilyCar) NumWheels() int { 
    return 4 
} 
func (*FamilyCar) NumSeats() int { 
    return 5 
} 

```

关于汽车的部分就到这里。现在我们需要摩托车工厂，它和汽车工厂一样，必须实现 `VehicleFactory` 接口：

```go
const ( 
    SportMotorbikeType = 1 
    CruiseMotorbikeType = 2 
) 

type MotorbikeFactory struct{} 

func (m *MotorbikeFactory) Build(v int) (Vehicle, error) { 
    switch v { 
        case SportMotorbikeType: 
        return new(SportMotorbike), nil 
        case CruiseMotorbikeType: 
        return new(CruiseMotorbike), nil 
        default: 
        return nil, errors.New(fmt.Sprintf("Vehicle of type %d not recognized\n", v)) 
    } 
} 

```

对于摩托车工厂，我们已使用 `const` 关键字定义了两种类型的摩托车：`SportMotorbikeType` 和 `CruiseMotorbikeType`。我们将通过 `Build` 方法中的 `v` 参数来切换，以确定应返回哪种类型。让我们编写两个具体的摩托车：

```go
//sport_motorbike.go 
package abstract_factory 

type SportMotorbike struct{} 

func (s *SportMotorbike) NumWheels() int { 
    return 2 
} 
func (s *SportMotorbike) NumSeats() int { 
    return 1 
} 
func (s *SportMotorbike) GetMotorbikeType() int { 
    return SportMotorbikeType 
} 

//cruise_motorbike.go 
package abstract_factory 

type CruiseMotorbike struct{} 

func (c *CruiseMotorbike) NumWheels() int { 
    return 2 
} 
func (c *CruiseMotorbike) NumSeats() int { 
    return 2 
} 
func (c *CruiseMotorbike) GetMotorbikeType() int { 
    return CruiseMotorbikeType 
} 

```

为了完成，我们需要抽象工厂本身，我们将将其放入之前创建的 `vehicle_factory.go` 文件中：

```go
package abstract_factory 

import ( 
    "fmt" 
    "errors" 
) 

type VehicleFactory interface { 
    Build(v int) (Vehicle, error) 
} 

const ( 
    CarFactoryType = 1 
    MotorbikeFactoryType = 2 
) 

func BuildFactory(f int) (VehicleFactory, error) { 
    switch f { 
        default: 
        return nil, errors.New(fmt.Sprintf("Factory with id %d not recognized\n", f)) 
    } 
}
```

我们将编写足够的测试以进行可靠的检查，因为本书的范围不涵盖 100%的语句。对于读者来说，完成这些测试将是一个很好的练习。首先，一个摩托车工厂测试：

```go
package abstract_factory 

import "testing" 

func TestMotorbikeFactory(t *testing.T) { 
    motorbikeF, err := BuildFactory(MotorbikeFactoryType) 
    if err != nil { 
        t.Fatal(err) 
    } 

    motorbikeVehicle, err := motorbikeF.Build(SportMotorbikeType) 
    if err != nil { 
        t.Fatal(err) 
    } 

    t.Logf("Motorbike vehicle has %d wheels\n", motorbikeVehicle.NumWheels()) 

    sportBike, ok := motorbikeVehicle.(Motorbike) 
    if !ok { 
        t.Fatal("Struct assertion has failed") 
    } 
    t.Logf("Sport motorbike has type %d\n", sportBike.GetMotorbikeType()) 
} 

```

我们使用包方法 `BuildFactory` 来检索摩托车工厂（在参数中传递 `MotorbikeFactory` ID），并检查是否有任何错误。然后，已经有了摩托车工厂，我们请求一个 `SportMotorbikeType` 类型的车辆并再次检查错误。有了返回的车辆，我们可以请求车辆接口的方法（`NumWheels` 和 `NumSeats`）。我们知道它是一辆摩托车，但如果我们不使用类型断言，我们无法请求摩托车的类型。我们在车辆上使用类型断言来检索代码行 `sportBike, found := motorbikeVehicle.(Motorbike)` 中 `motorbikeVehicle` 表示的摩托车，我们必须检查我们收到的类型是否正确。

最后，现在我们有一个摩托车实例，我们可以通过使用 `GetMotorbikeType` 方法来请求自行车类型。现在我们将以相同的方式编写一个测试来检查汽车工厂：

```go
func TestCarFactory(t *testing.T) { 
    carF, err := BuildFactory(CarFactoryType) 
    if err != nil { 
        t.Fatal(err) 
    } 

    carVehicle, err := carF.Build(LuxuryCarType) 
    if err != nil { 
        t.Fatal(err) 
    } 

    t.Logf("Car vehicle has %d seats\n", carVehicle.NumWheels()) 

    luxuryCar, ok := carVehicle.(Car) 
    if !ok { 
        t.Fatal("Struct assertion has failed") 
    } 
    t.Logf("Luxury car has %d doors.\n", luxuryCar.NumDoors()) 
} 

```

再次，我们使用 `BuildFactory` 方法通过使用参数中的 `CarFactoryType` 来检索 `Car` 工厂。我们希望这个工厂返回一个 `Luxury` 类型的汽车，以便返回一个 `vehicle` 实例。我们再次进行类型断言，以便指向一个汽车实例，这样我们就可以使用 `NumDoors` 方法来请求车门数量。

让我们运行单元测试：

```go
go test -v -run=Factory .
=== RUN   TestMotorbikeFactory
--- FAIL: TestMotorbikeFactory (0.00s)
 vehicle_factory_test.go:8: Factory with id 2 not recognized
=== RUN   TestCarFactory
--- FAIL: TestCarFactory (0.00s)
 vehicle_factory_test.go:28: Factory with id 1 not recognized
FAIL
exit status 1
FAIL 

```

完成。它不能识别任何工厂，因为它们的实现尚未完成。

## 实现

为了简洁起见，每个工厂的实现已经完成。它们与工厂方法非常相似，唯一的区别在于在工厂方法中，我们不使用工厂方法的实例，因为我们直接使用包函数。`vehicle`工厂的实现如下：

```go
func BuildFactory(f int) (VehicleFactory, error) { 
    switch f { 
        case CarFactoryType: 
        return new(CarFactory), nil 
        case MotorbikeFactoryType: 
        return new(MotorbikeFactory), nil 
        default: 
        return nil, errors.New(fmt.Sprintf("Factory with id %d not recognized\n", f)) 
    } 
} 

```

就像在任何工厂一样，我们在不同的工厂可能性之间切换，以返回所需的那个。因为我们已经实现了所有具体的车辆，所以测试也必须运行：

```go
go test -v -run=Factory -cover .
=== RUN   TestMotorbikeFactory
--- PASS: TestMotorbikeFactory (0.00s)
 vehicle_factory_test.go:16: Motorbike vehicle has 2 wheels
 vehicle_factory_test.go:22: Sport motorbike has type 1
=== RUN   TestCarFactory
--- PASS: TestCarFactory (0.00s)
 vehicle_factory_test.go:36: Car vehicle has 4 seats
 vehicle_factory_test.go:42: Luxury car has 4 doors.
PASS
coverage: 45.8% of statements
ok

```

所有这些都通过了。仔细观察并注意，我们在运行测试时使用了`-cover`标志来返回包的覆盖率百分比：45.8%。这告诉我们，45.8%的行被我们所写的测试覆盖，但还有 54.2%没有被测试覆盖。这是因为我们没有用测试覆盖巡航摩托车和家庭汽车。如果你编写这些测试，结果应该会上升到大约 70.8%。

### 小贴士

类型断言在其他语言中也称为**类型转换**。当你有一个接口实例，本质上是一个指向结构的指针时，你只能访问接口方法。使用类型断言，你可以告诉编译器指向的结构类型，这样你就可以访问整个结构的字段和方法。

## 关于抽象工厂方法的一些说明

我们已经学会了如何编写一个工厂的工厂，它为我们提供了一个非常通用的车辆类型对象。这种模式在许多应用程序和库中都很常见，例如跨平台 GUI 库。想象一下，有一个按钮，一个通用对象，以及一个按钮工厂，它为你提供了一个 Microsoft Windows 按钮的工厂，同时你还有一个 Mac OS X 按钮的工厂。你不想处理每个平台的实现细节，你只想实现按钮引发的一些特定行为的动作。

此外，我们已经看到了使用两种不同的解决方案来处理相同问题时存在的差异--抽象工厂和建造者模式。正如你所看到的，在建造者模式中，我们有一个无结构的对象列表（同一工厂中的汽车和摩托车）。我们还鼓励在建造者模式中重用构建算法。在抽象工厂中，我们有一个非常结构的车辆列表（摩托车工厂和汽车工厂）。我们也没有将汽车和摩托车的创建混合在一起，这为创建过程提供了更多的灵活性。抽象工厂和建造者模式都可以解决相同的问题，但你的特定需求将帮助你找到应该导致你选择一个解决方案而不是另一个的细微差异。

# 原型设计模式

本章我们将看到的最后一个模式是**原型**模式。像所有创建型模式一样，这在创建对象时也很有用，原型模式周围通常会有更多的模式。

当使用建造者模式时，我们处理重复的构建算法，并且通过工厂我们简化了许多类型对象的创建；使用原型模式时，我们将使用某种类型的已创建实例来克隆它，并完成每个上下文的特定需求。让我们详细看看。

## 描述

原型模式的目的是在编译时创建一个对象或一组对象，但在运行时可以克隆任意多次。例如，这可以作为新注册用户网页的默认模板或某些服务的默认定价计划。与建造者模式的关键区别在于，对象是为用户克隆的，而不是在运行时构建的。您还可以构建一个类似缓存的解决方案，使用原型存储信息。

## 目标

原型设计模式的主要目标是避免重复创建对象。想象一个由数十个字段和嵌入类型组成的默认对象。我们不希望在每次使用该对象时都编写该类型所需的所有内容，尤其是如果我们可以通过创建具有不同*基础*的实例来搞砸它：

+   维护一组将被克隆以创建新实例的对象

+   提供某种类型的默认值，以便在此基础上开始工作

+   释放复杂对象初始化的 CPU 资源，以占用更多内存资源

## 示例

我们将构建一个虚构的定制衬衫店的组件，该店将有一些带有默认颜色和价格的衬衫。每件衬衫还将有一个**库存单位（SKU）**，这是一个用于识别存储在特定位置的系统的标识符，它需要更新。

## 验收标准

为了实现示例中描述的内容，我们将使用衬衫的原型。每次我们需要一件新衬衫时，我们将使用这个原型，克隆它并与之工作。特别是，这些是使用原型模式设计方法在此示例中的验收标准：

+   拥有一个衬衫克隆对象和接口，可以请求不同类型的衬衫（白色、黑色和蓝色，价格分别为 15.00、16.00 和 17.00 美元）

+   当您要求一件白色衬衫时，必须制作白色衬衫的克隆，并且新实例必须与原始实例不同

+   创建的对象的 SKU 不应影响新对象创建

+   一个信息方法必须给我所有在实例字段上可用的信息，包括更新的 SKU

## 单元测试

首先，我们需要一个`ShirtCloner`接口以及一个实现该接口的对象。此外，我们还需要一个包级别的函数`GetShirtsCloner`来获取克隆器的新实例：

```go
type ShirtCloner interface { 
    GetClone(s int) (ItemInfoGetter, error) 
} 

const ( 
    White = 1 
    Black = 2 
    Blue  = 3 
) 

func GetShirtsCloner() ShirtCloner { 
    return nil 
} 

type ShirtsCache struct {} 
func (s *ShirtsCache)GetClone(s int) (ItemInfoGetter, error) { 
    return nil, errors.New("Not implemented yet") 
} 

```

现在我们需要一个对象结构来克隆，该结构实现了一个用于检索其字段信息的接口。我们将把这个对象称为`Shirt`，以及`ItemInfoGetter`接口：

```go
type ItemInfoGetter interface { 
    GetInfo() string 
} 

type ShirtColor byte 

type Shirt struct { 
    Price float32 
    SKU   string 
    Color ShirtColor 
} 
func (s *Shirt) GetInfo()string { 
    return "" 
} 

func GetShirtsCloner() ShirtCloner { 
    return nil 
} 

var whitePrototype *Shirt = &Shirt{ 
    Price: 15.00, 
    SKU:   "empty", 
    Color: White, 
} 

func (i *Shirt) GetPrice() float32 { 
    return i.Price 
} 

```

### 小贴士

你是否意识到我们定义的`ShirtColor`类型实际上只是一个`byte`类型？也许你在想为什么我们没有简单地使用`byte`类型。我们可以这样做，但这样我们创建了一个易于阅读的结构体，如果需要的话，我们可以在未来通过添加一些方法来升级它。例如，我们可以编写一个`String()`方法，该方法以字符串格式返回颜色（类型 1 为`White`，类型 2 为`Black`，类型 3 为`Blue`）。

使用这段代码，我们目前已经可以编写我们的第一个测试：

```go
func TestClone(t *testing.T) { 
    shirtCache := GetShirtsCloner() 
    if shirtCache == nil { 
        t.Fatal("Received cache was nil") 
    } 

    item1, err := shirtCache.GetClone(White) 
    if err != nil { 
        t.Error(err) 
} 

//more code continues here... 

```

我们将涵盖我们场景的第一个案例，其中我们需要一个克隆对象，我们可以用它来请求不同的衬衫颜色。

对于第二种情况，我们将取原始对象（因为我们处于包的作用域内，所以我们可以访问它），并将其与我们的`shirt1`实例进行比较。

```go
if item1 == whitePrototype { 
    t.Error("item1 cannot be equal to the white prototype"); 
} 

```

现在，对于第三种情况。首先，我们将`item1`断言为衬衫类型，这样我们就可以设置 SKU。我们将创建第二件衬衫，也是白色的，并且我们也将其断言为检查 SKU 是否不同：

```go
shirt1, ok := item1.(*Shirt) 
if !ok { 
    t.Fatal("Type assertion for shirt1 couldn't be done successfully") 
} 
shirt1.SKU = "abbcc" 

item2, err := shirtCache.GetClone(White) 
if err != nil { 
    t.Fatal(err) 
} 

shirt2, ok := item2.(*Shirt) 
if !ok { 
    t.Fatal("Type assertion for shirt1 couldn't be done successfully") 
} 

if shirt1.SKU == shirt2.SKU { 
    t.Error("SKU's of shirt1 and shirt2 must be different") 
} 

if shirt1 == shirt2 { 
    t.Error("Shirt 1 cannot be equal to Shirt 2") 
} 

```

最后，对于第四种情况，我们记录第一件和第二件衬衫的信息：

```go
t.Logf("LOG: %s", shirt1.GetInfo()) 
t.Logf("LOG: %s", shirt2.GetInfo()) 

```

我们将打印两件衬衫的内存位置，因此我们在更物理的层面上进行这个断言：

```go
t.Logf("LOG: The memory positions of the shirts are different %p != %p \n\n", &shirt1, &shirt2) 

```

最后，我们运行测试以检查它是否失败：

```go
go test -run=TestClone . 
--- FAIL: TestClone (0.00s) 
prototype_test.go:10: Not implemented yet 
FAIL 
FAIL

```

我们必须在这里停止，这样测试就不会在尝试使用由`GetShirtsCloner`函数返回的 nil 对象时崩溃。

## 实现

我们将从`GetClone`方法开始。这个方法应该返回指定类型的项，我们有三种类型：白色、黑色和蓝色：

```go
var whitePrototype *Shirt = &Shirt{ 
    Price: 15.00, 
    SKU:   "empty", 
    Color: White, 
} 

var blackPrototype *Shirt = &Shirt{ 
    Price: 16.00, 
    SKU:   "empty", 
    Color: Black, 
} 

var bluePrototype *Shirt = &Shirt{ 
    Price: 17.00, 
    SKU:   "empty", 
    Color: Blue, 
} 

```

因此，现在我们已经有了三个原型可以工作，我们可以实现`GetClone(s int)`方法：

```go
type ShirtsCache struct {} 
func (s *ShirtsCache)GetClone(s int) (ItemInfoGetter, error) { 
    switch m { 
        case White: 
            newItem := *whitePrototype 
            return &newItem, nil 
        case Black: 
            newItem := *blackPrototype 
            return &newItem, nil 
        case Blue: 
            newItem := *bluePrototype 
            return &newItem, nil 
        default: 
            return nil, errors.New("Shirt model not recognized") 
    } 
} 

```

`Shirt`结构体还需要一个`GetInfo`实现来打印实例的内容。

```go
type ShirtColor byte 

type Shirt struct { 
    Price float32 
    SKU   string 
    Color ShirtColor 
} 

func (s *Shirt) GetInfo() string { 
    return fmt.Sprintf("Shirt with SKU '%s' and Color id %d that costs %f\n", s.SKU, s.Color, s.Price) 
} 

```

最后，让我们运行测试以查看一切是否正常工作：

```go
go test -run=TestClone -v . 
=== RUN   TestClone 
--- PASS: TestClone (0.00s) 
prototype_test.go:41: LOG: Shirt with SKU 'abbcc' and Color id 1 that costs 15.000000 
prototype_test.go:42: LOG: Shirt with SKU 'empty' and Color id 1 that costs 15.000000 
prototype_test.go:44: LOG: The memory positions of the shirts are different 0xc42002c038 != 0xc42002c040  

PASS 
ok

```

在日志中（记住在运行测试时设置`-v`标志），你可以检查`shirt1`和`shirt2`具有不同的 SKU。我们还可以看到两个对象的内存位置。请注意，你电脑上显示的位置可能会不同。

## 我们关于原型设计模式学到的内容

原型模式是构建缓存和默认对象的有力工具。你可能也已经意识到，某些模式之间可能存在一些重叠，但它们之间的小差异使得它们在某些情况下更为合适，而在其他情况下则不太合适。

# 摘要

我们已经看到了在软件行业中常用的五种主要创建型设计模式。它们的目的在于从用户的角度抽象出对象的创建，以简化复杂性或提高可维护性。自 1990 年代以来，它们一直是数千个应用程序和库的基础，我们今天使用的许多软件都包含这些创建型模式。

值得注意的是，这些模式不是线程安全的。在更高级的章节中，我们将看到 Go 中的并发编程，以及如何使用并发方法创建一些更关键的设计模式。
