# 第二章。创建模式-单例、生成器、工厂、原型和抽象工厂设计模式

我们定义了两种类型的汽车-豪华和家庭。汽车工厂将返回我们将要涵盖的第一组设计模式是创建模式。顾名思义，它将常见的创建对象的实践分组在一起，因此对象的创建更多地封装在需要这些对象的用户之外。主要是，创建模式试图为用户提供可直接使用的对象，而不是要求它们的创建，这在某些情况下可能是复杂的，或者会将您的代码与应该在接口中定义的功能的具体实现耦合在一起。

# 单例设计模式-在整个程序中具有唯一实例的类型

你有没有为软件工程师做过面试？有趣的是，当你问他们关于设计模式时，超过 80%的人会提到**Singleton**设计模式。为什么呢？也许是因为它是最常用的设计模式之一，或者是最容易理解的设计模式之一。由于后一种原因，我们将从创建型设计模式开始我们的旅程。

## 描述

单例模式很容易记住。顾名思义，它将为您提供对象的单一实例，并保证没有重复。

在第一次调用实例时，它被创建，然后在应用程序中需要使用特定行为的所有部分之间重复使用。

您将在许多不同的情况下使用单例模式。例如：

+   当您想要使用相同的数据库连接来进行每次查询时

+   当您打开**安全外壳**（**SSH**）连接到服务器以执行一些任务，并且不想为每个任务重新打开连接时

+   如果需要限制对某个变量或空间的访问，可以使用 Singleton 作为访问该变量的门（在接下来的章节中，我们将看到在 Go 中使用通道更容易实现这一点）

+   如果需要限制对某些地方的调用次数，可以创建一个 Singleton 实例来在接受的窗口中进行调用

可能性是无穷无尽的，我们只是提到了其中一些。

## 目标

作为一般指南，我们考虑在以下规则适用时使用 Singleton 模式：

+   我们需要一个特定类型的单一共享值。

+   我们需要将某种类型的对象创建限制为整个程序中的单个单元。

## 示例-唯一计数器

作为我们必须确保只有一个实例的对象的示例，我们将编写一个计数器，它保存程序执行期间调用的次数。无论我们有多少计数器实例，它们都必须*计数*相同的值，并且在实例之间必须保持一致。

## 需求和验收标准

编写所述的单一计数器有一些要求和验收标准。它们如下：

+   当之前没有创建计数器时，将创建一个新的计数器，其值为 0

+   如果计数器已经被创建，返回持有实际计数的实例

+   如果我们调用方法`AddOne`，计数必须增加 1

我们有三个测试场景要在我们的单元测试中检查。

## 首先编写单元测试

Go 对这种模式的实现与您在 Java 或 C++等纯面向对象语言中找到的实现略有不同，那里有静态成员。在 Go 中，没有静态成员，但我们有包范围来提供类似的结果。

为了设置我们的项目，我们必须在我们的`$GOPATH/src`目录中创建一个新文件夹。正如我们在第一章中提到的一般规则，*准备...开始...Go！*，是创建一个带有 VCS 提供者（如 GitHub）、用户名和项目名称的子文件夹。

例如，在我的情况下，我使用 GitHub 作为我的 VCS，我的用户名是*sayden*，所以我将创建路径`$GOPATH/src/github.com/sayden/go-design-patterns/creational/singleton`。路径中的`go-design-patterns`是项目名称，creational 子文件夹也将是我们的库名称，singleton 是此特定包和子文件夹的名称：

```go
mkdir -p $GOPATH/src/github.com/sayden/go-design-patterns/creational/singleton 
cd $GOPATH/src/github.com/sayden/go-design-
patterns/creational/singleton

```

在 singleton 文件夹内创建一个名为`singleton.go`的新文件，以反映包的名称，并编写以下`singleton`类型的包声明：

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

由于我们在编写代码时遵循 TDD 方法，让我们编写使用我们刚刚声明的函数的测试。测试将根据我们之前编写的验收标准来定义。按照测试文件的惯例，我们必须创建一个与要测试的文件同名的文件，后缀为`_test.go`。两者必须驻留在同一个文件夹中：

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

第一个测试检查了复杂应用程序中显而易见但同样重要的事情。当我们请求计数器的实例时，我们实际上收到了一些东西。我们必须将其视为一个创建模式——我们将对象的创建委托给一个可能在对象的创建或检索中失败的未知包。我们还将当前计数器存储在`expectedCounter`变量中，以便稍后进行比较：

```go
currentCount := counter1.AddOne() 
if currentCount != 1 { 
     t.Errorf("After calling for the first time to count, the count must be 1 but it is %d\n", currentCount) 
} 

```

现在我们利用 Go 的零初始化特性。请记住，Go 中的整数类型不能为 nil，而且我们知道这是对计数器的第一次调用，它是一个整数类型的变量，我们也知道它是零初始化的。因此，在对`AddOne()`函数的第一次调用之后，计数的值必须为 1。

检查第二个条件的测试证明了`expectedConnection`变量与我们稍后请求的返回连接没有不同。如果它们不同，消息`Singleton instances must be different`将导致测试失败：

```go
counter2 := GetInstance() 
if counter2 != expectedCounter { 
    //Test 2 failed 
    t.Error("Expected same instance in counter2 but it got a different instance") 
} 

```

最后的测试只是再次计数 1，使用第二个实例。之前的结果是 1，所以现在必须给我们 2：

```go
currentCount = counter2.AddOne() 
if currentCount != 2 { 
    t.Errorf("After calling 'AddOne' using the second counter, the current count must be 2 but was %d\n", currentCount) 
} 

```

完成测试部分的最后一件事是执行测试，以确保它们在实施之前失败。如果其中一个没有失败，那就意味着我们做错了什么，我们必须重新考虑那个特定的测试。我们必须打开终端并导航到 singleton 包的路径以执行：

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

最后，我们必须实现单例模式。正如我们之前提到的，通常会在诸如 Java 或 C++之类的语言中编写一个`static`方法和实例来检索单例实例。在 Go 中，我们没有关键字`static`，但我们可以通过使用包的作用域来实现相同的结果。首先，我们创建一个包含我们希望在程序执行期间保证为单例的对象的`struct`：

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

我们必须密切关注这段代码。在诸如 Java 或 C++之类的语言中，变量实例将在程序开始时初始化为 NULL。在 Go 中，您可以将指向结构的指针初始化为`nil`，但不能将结构初始化为`nil`（相当于 NULL）。因此，`var instance *singleton`行将指针定义为 nil 的 Singleton 类型结构，并命名为`instance`。

我们创建了一个`GetInstance`方法，检查实例是否已经初始化（`instance == nil`），并在`instance = new(singleton)`行中创建一个实例。记住，当我们使用关键字`new`时，我们正在创建指向括号内类型的实例的指针。

`AddOne`方法将获取变量实例的计数，将其增加 1，并返回计数器的当前值。

让我们现在再次运行我们的单元测试：

```go
$ go test -v -run=GetInstance
=== RUN   TestGetInstance
--- PASS: TestGetInstance (0.00s)
PASS
ok

```

## 关于单例设计模式的一些话

我们已经看到了单例模式的一个非常简单的例子，部分应用于某些情况，即一个简单的计数器。只需记住，单例模式将使您能够在应用程序中拥有某个结构的唯一实例，并且没有任何包可以创建此结构的任何克隆。

使用单例，您还隐藏了创建对象的复杂性，以防它需要一些计算，并且如果它们都是相似的话，每次需要一个实例时都要创建它的陷阱。所有这些代码编写、检查变量是否已经存在和存储都封装在单例中，如果使用全局变量，则无需在每个地方重复。

在这里，我们正在学习单线程上的经典单例实现。当我们到达有关并发的章节时，我们将看到并发单例实现，因为这种实现不是线程安全的！

# 建造者设计模式 - 重用算法来创建接口的多个实现

谈到**创建型**设计模式，拥有**建造者**设计模式似乎非常合理。建造者模式帮助我们构建复杂的对象，而不是直接实例化它们的结构，或编写它们所需的逻辑。想象一个对象可能有数十个字段，这些字段本身是更复杂的结构。现在想象一下，您有许多具有这些特征的对象，而且您可能还有更多。我们不想在只需要使用这些对象的包中编写创建所有这些对象的逻辑。

## 描述

实例创建可以简单到提供开放和关闭大括号`{}`并将实例留有零值，也可以复杂到需要进行一些 API 调用、检查状态并为其字段创建对象的对象。您还可以拥有由许多对象组成的对象，这在 Go 中非常惯用，因为它不支持继承。

同时，您可以使用相同的技术来创建许多类型的对象。例如，您将使用几乎相同的技术来构建汽车和构建公共汽车，只是它们的大小和座位数不同，那么为什么不重用构建过程呢？这就是建造者模式发挥作用的地方。

## 目标

建造者设计模式尝试：

+   抽象复杂的创建，使对象创建与对象用户分离

+   通过填充其字段和创建嵌入对象逐步创建对象

+   在许多对象之间重用对象创建算法

## 示例 - 车辆制造

建造者设计模式通常被描述为导演、几个建造者和他们构建的产品之间的关系。继续我们的汽车示例，我们将创建一个车辆建造者。创建车辆（产品）的过程（通常描述为算法）对于每种类型的车辆来说基本相同 - 选择车辆类型、组装结构、放置轮子和放置座位。如果您仔细想想，您可以使用此描述构建汽车和摩托车（两个建造者），因此我们正在重用描述来在制造中创建汽车。在我们的示例中，导演由`ManufacturingDirector`类型表示。

## 需求和验收标准

就我们所描述的而言，我们必须处理`Car`和`Motorbike`类型的建造者以及一个名为`ManufacturingDirector`的唯一导演，以接受建造者并构建产品。因此，`Vehicle`建造者示例的要求如下：

+   我必须有一个制造类型，可以构造车辆所需的一切

+   使用汽车建造者时，必须返回具有四个轮子、五个座位和结构定义为`Car`的`VehicleProduct`

+   使用摩托车建造者时，必须返回具有两个轮子、两个座位和结构定义为`Motorbike`的`VehicleProduct`

+   由任何`BuildProcess`建造者构建的`VehicleProduct`必须可以进行修改

## 车辆建造者的单元测试

根据先前的验收标准，我们将创建一个主管变量，即`ManufacturingDirector`类型，以使用产品建造者变量表示的汽车和摩托车的构建过程。主管负责构建对象，但建造者返回实际的车辆。因此，我们的建造者声明将如下所示：

```go
package creational 

type BuildProcess interface { 
    SetWheels() BuildProcess 
    SetSeats() BuildProcess 
    SetStructure() BuildProcess 
    GetVehicle() VehicleProduct 
} 

```

上述接口定义了构建车辆所需的步骤。如果要被制造使用，每个建造者都必须实现这个`interface`。在每个`Set`步骤上，我们返回相同的构建过程，因此我们可以在同一语句中链接各种步骤，正如我们将在后面看到的。最后，我们需要一个`GetVehicle`方法来从建造者中检索`Vehicle`实例：

```go
type ManufacturingDirector struct {} 

func (f *ManufacturingDirector) Construct() { 
    //Implementation goes here 
} 

func (f *ManufacturingDirector) SetBuilder(b BuildProcess) { 
    //Implementation goes here 
} 

```

`ManufacturingDirector`主管变量负责接受建造者。它有一个`Construct`方法，将使用存储在`Manufacturing`中的建造者，并复制所需的步骤。`SetBuilder`方法将允许我们更改在`Manufacturing`主管中使用的建造者：

```go
type VehicleProduct struct { 
    Wheels    int 
    Seats     int 
    Structure string 
} 

```

产品是我们在使用制造时要检索的最终对象。在这种情况下，车辆由车轮、座位和结构组成：

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

第一个建造者是`Car`建造者。它必须实现`BuildProcess`接口中定义的每个方法。这是我们为这个特定建造者设置信息的地方：

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

`Motorbike`结构必须与`Car`结构相同，因为它们都是建造者实现，但请记住，构建每个的过程可能会有很大的不同。有了这些对象的声明，我们可以创建以下测试：

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

我们将从`Manufacturing`主管和`Car`建造者开始，以满足前两个验收标准。在上述代码中，我们正在创建我们的`Manufacturing`主管，它将负责在测试期间创建每辆车辆。创建`Manufacturing`主管后，我们创建了一个`CarBuilder`，然后通过使用`SetBuilder`方法将其传递给制造。一旦`Manufacturing`主管知道现在它必须构建什么，我们就可以调用`Construct`方法来使用`CarBuilder`创建`VehicleProduct`。最后，一旦我们为我们的汽车准备好所有零件，我们就调用`CarBuilder`上的`GetVehicle`方法来检索`Car`实例：

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

我们编写了三个小测试来检查结果是否为汽车。我们检查汽车是否有四个车轮，结构是否具有描述`Car`，座位数为五。我们有足够的数据来执行测试，并确保它们失败，以便我们可以认为它们是可靠的：

```go
$ go test -v -run=TestBuilder .
=== RUN   TestBuilderPattern
--- FAIL: TestBuilderPattern (0.00s)
 builder_test.go:15: Wheels on a car must be 4 and they were 0
 builder_test.go:19: Structure on a car must be 'Car' and was
 builder_test.go:23: Seats on a car must be 5 and they were 0
FAIL

```

太好了！现在我们将为`Motorbike`建造者创建测试，以满足第三和第四个验收标准：

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

上述代码是对汽车测试的延续。正如您所看到的，我们重用了先前创建的制造来通过将`Motorbike`建造者传递给它来创建摩托车。然后我们再次点击`construct`按钮来创建必要的零件，并调用建造者的`GetVehicle`方法来检索摩托车实例。

快速浏览一下，因为我们已经将这辆摩托车的默认座位数更改为 1。我们想要展示的是，即使有了建造者，您也必须能够更改返回实例中的默认信息，以满足某些特定需求。由于我们手动设置了车轮，我们不会测试这个功能。

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

## 实施

我们将开始实施制造。正如我们之前所说（并且在我们的单元测试中设置），“制造”总监必须接受一个建造者并使用提供的建造者构建车辆。回想一下，`BuildProcess`接口将定义构建任何车辆所需的常见步骤，“制造”总监必须接受建造者并与他们一起构建车辆：

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

我们的“制造总监”需要一个字段来存储正在使用的建造者；这个字段将被称为“建造者”。 `SetBuilder`方法将用参数中提供的建造者替换存储的建造者。最后，仔细看一下“构造”方法。它接受已存储的建造者并重现将创建某种未知类型的完整车辆的`BuildProcess`方法。正如您所看到的，我们已经在同一行中使用了所有设置调用，这要归功于在每个调用上返回`BuildProcess`接口。这样代码更加紧凑：

### 提示

您是否意识到建造者模式中的总监实体也是单例模式的明显候选者？在某些情况下，只有总监的一个实例可用可能非常关键，这就是您将为建造者的总监创建单例模式的地方。设计模式组合是一种非常常见且非常强大的技术！

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

这是我们的第一个建造者，“汽车”建造者。一个建造者将需要存储一个名为`v`的`VehicleProduct`对象。然后我们设置了我们业务中汽车的特定需求-四个轮子，五个座位和一个名为“汽车”的结构。在`GetVehicle`方法中，我们只返回建造者内部存储的`VehicleProduct`，这个产品必须已经由“制造总监”类型构建。

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

“摩托车”建造者与“汽车”建造者相同。我们定义了摩托车有两个轮子，两个座位和一个名为“摩托车”的结构。它与“汽车”对象非常相似，但想象一下，您想要区分运动摩托车（只有一个座位）和巡航摩托车（有两个座位）。您可以简单地为运动摩托车创建一个实现构建过程的新结构。

您可以看到这是一个重复的模式，但在`BuildProcess`接口的每个方法的范围内，您可以封装尽可能多的复杂性，以便用户不需要了解有关对象创建的详细信息。

有了所有对象的定义，让我们再次运行测试：

```go
=== RUN   TestBuilderPattern
--- PASS: TestBuilderPattern (0.00s)
PASS
ok  _/home/mcastro/pers/go-design-patterns/creational 0.001s

```

干得好！想象一下向“制造总监”添加新车辆有多么容易，只需创建一个封装新车辆数据的新类。例如，让我们添加一个`BusBuilder`结构：

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

就是这样；通过遵循建造者设计模式，您的“制造总监”将准备好使用新产品。

## 封装建造者设计模式

建造者设计模式通过使用总监使用的通用构建算法来帮助我们维护不可预测数量的产品。构建过程始终从产品的用户中抽象出来。

同时，当我们的源代码新手需要向*管道*添加新产品时，拥有定义的构建模式会有所帮助。`BuildProcess`接口指定了他必须遵守以成为可能的建造者的一部分。

但是，当您不完全确定算法是否会更加稳定时，请尽量避免使用建造者模式，因为此接口的任何小更改都将影响到所有建造者，如果您添加一种新方法，有些建造者需要它，而其他建造者则不需要，这可能会很尴尬。

# 工厂方法-委托创建不同类型的付款

工厂方法模式（或简称工厂）可能是行业中第二为人熟知和使用的设计模式。它的目的是将用户与他需要为特定目的实现的结构体的知识抽象出来，比如从网络服务或数据库中检索一些值。用户只需要一个提供这个值的接口。通过将这个决定委托给工厂，这个工厂可以提供适合用户需求的接口。如果需要，它还可以简化底层类型的实现的降级或升级过程。

## 描述

使用工厂方法设计模式时，我们获得了一个额外的封装层，以便我们的程序可以在受控环境中增长。通过工厂方法，我们将对象族的创建委托给不同的包或对象，以使我们抽象出我们可以使用的可能对象池的知识。想象一下，您想要使用旅行社组织您的假期。您不需要处理酒店和旅行，只需告诉旅行社您感兴趣的目的地，他们将为您提供一切所需。旅行社代表了旅行的工厂。

## 目标

在前面的描述之后，工厂方法设计模式的以下目标必须对您清晰：

+   将新实例的结构的创建委托给程序的不同部分

+   在接口级别工作，而不是使用具体的实现

+   将对象族分组以获得一个对象族创建者

## 示例-商店的支付方法工厂

对于我们的例子，我们将实现一个支付方法工厂，它将为我们提供在商店支付的不同方式。一开始，我们将有两种支付方式--现金和信用卡。我们还将有一个带有`Pay`方法的接口，每个想要用作支付方法的结构体都必须实现。

## 验收标准

使用前述描述，验收标准的要求如下：

+   拥有一个称为`Pay`的每种支付方法的通用方法

+   为了能够将支付方法的创建委托给工厂

+   能够通过将其添加到工厂方法来将更多的支付方法添加到库中

## 第一个单元测试

工厂方法有一个非常简单的结构；我们只需要确定我们存储了多少个接口的实现，然后提供一个`GetPaymentMethod`方法，您可以将支付类型作为参数传递：

```go
type PaymentMethod interface { 
    Pay(amount float32) string 
} 

```

前面的行定义了支付方法的接口。它们定义了在商店支付的方式。工厂方法将返回实现此接口的类型的实例：

```go
const ( 
    Cash      = 1 
    DebitCard = 2 
) 

```

我们必须将工厂的已识别支付方法定义为常量，以便我们可以从包外部调用和检查可能的支付方法。

```go
func GetPaymentMethod(m int) (PaymentMethod, error) { 
    return nil, errors.New("Not implemented yet") 
} 

```

前面的代码是将为我们创建对象的函数。它返回一个指针，必须有一个实现`PaymentMethod`接口的对象，并且如果要求一个未注册的方法，则返回一个错误。

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

为了完成工厂的声明，我们创建了两种支付方法。正如您所看到的，`CashPM`和`DebitCardPM`结构体通过声明一个`Pay(amount float32) string`方法来实现`PaymentMethod`接口。返回的字符串将包含有关支付的信息。

有了这个声明，我们将从编写第一个验收标准的测试开始：拥有一个通用方法来检索实现`PaymentMethod`接口的对象：

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

现在我们将把测试分成几个测试函数。`GetPaymentMethod`是一个常用的检索支付方式的方法。我们使用常量`Cash`，我们已经在实现文件中定义了它（如果我们在包的范围之外使用这个常量，我们将使用包的名称作为前缀来调用它，所以语法将是`creational.Cash`）。我们还检查在请求支付方式时是否没有收到错误。请注意，如果我们在请求支付方式时收到错误，我们将调用`t.Fatal`来停止测试的执行；如果我们像之前的测试一样只调用`t.Error`，那么当我们尝试访问 nil 对象的`Pay`方法时，我们将在下一行中遇到问题，我们的测试将崩溃执行。我们继续通过将 10.30 作为金额传递给接口的`Pay`方法来使用接口。返回的消息将包含文本`paid using cash`。`t.Log(string)`方法是测试中的一个特殊方法。这个结构允许我们在运行测试时写一些日志，如果我们传递了`-v`标志。

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

我们用相同的方法重复相同的操作。我们请求使用常量`DebitCard`定义的支付方式，当使用借记卡支付时，返回的消息必须包含`paid using debit card`字符串。

```go

func TestGetPaymentMethodNonExistent(t *testing.T) { 
    payment, err = GetPaymentMethod(20) 

    if err == nil { 
        t.Error("A payment method with ID 20 must return an error") 
    } 
    t.Log("LOG:", err) 
}
```

最后，我们将测试请求一个不存在的支付方式的情况（用数字 20 表示，它与工厂中的任何已识别的常量都不匹配）。当请求未知的支付方式时，我们将检查是否返回了错误消息（任何错误消息）。

让我们检查一下所有的测试是否都失败了：

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

正如您在这个例子中所看到的，我们只能看到返回`PaymentMethod`接口的测试失败。在这种情况下，我们将不得不实现代码的一部分，然后再次进行测试，然后才能继续。

## 实施

我们将从`GetPaymentMethod`方法开始。它必须接收一个与同一文件中定义的常量匹配的整数，以知道应该返回哪种实现。

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

我们使用一个普通的 switch 来检查参数`m`（方法）的内容。如果它匹配任何已知的方法--现金或借记卡，它将返回它们的新实例。否则，它将返回一个 nil 和一个指示支付方式未被识别的错误。现在我们可以再次运行我们的测试，以检查单元测试的第二部分：

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

现在我们不会再收到找不到支付方式类型的错误，而是在尝试使用它所涵盖的任何方法时，会收到“消息不正确”的错误。当我们请求一个未知的支付方式时，我们也摆脱了“未实现”的消息。现在让我们实现结构体：

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

我们只是得到金额，以一个格式良好的消息打印出来。有了这个实现，现在所有的测试都会通过：

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

您看到了`LOG`：消息吗？它们不是错误，我们只是打印一些在使用被测试的包时收到的信息。除非您将`-v`标志传递给测试命令，否则可以省略这些消息：

```go
$ go test -run=GetPaymentMethod .
ok

```

## 将借记卡方法升级到新平台

现在想象一下，由于某种原因，您的`DebitCard`支付方式已经更改，您需要一个新的结构。为了实现这种情况，您只需要创建新的结构，并在用户请求`DebitCard`支付方式时替换旧的结构。

```go
type CreditCardPM struct {} 
 func (d *CreditCardPM) Pay(amount float32) string { 
   return fmt.Sprintf("%#0.2f paid using new credit card implementation\n", amount) 
} 

```

这是我们将替换`DebitCardPM`结构的新类型。`CreditCardPM`实现了与借记卡相同的`PaymentMethod`接口。我们没有删除以后可能需要的旧结构。唯一的区别在于返回的消息现在包含了关于新类型的信息。我们还必须修改检索支付方式的方法：

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

唯一的修改是在创建新的借记卡的那一行，现在指向新创建的结构。让我们运行测试，看看一切是否仍然正确：

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

哦，糟糕！出了些问题。使用信用卡支付时返回的预期消息与返回的消息不匹配。这是否意味着我们的代码不正确？一般来说，是的，你不应该修改你的测试来使你的程序工作。在定义测试时，你还应该注意不要定义得太多，因为这样你可能会在测试中实现一些你的代码中没有的耦合。由于消息限制，我们有一些语法上正确的消息可能性，所以我们将把它改为以下内容：

```go
return fmt.Sprintf("%#0.2f paid using debit card (new)\n", amount) 

```

现在我们再次运行测试：

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

一切又恢复正常了。这只是一个写好单元测试的小例子。当我们想要检查使用借记卡支付方法返回的消息是否包含“使用借记卡支付”字符串时，我们可能有点过于严格，最好分别检查这些单词，或者定义一个更好的格式来返回消息。

## 我们从工厂方法中学到了什么

通过工厂方法模式，我们已经学会了如何将对象家族分组，使其实现在我们的范围之外。我们还学会了在需要升级已使用结构的实现时该怎么做。最后，我们已经看到，如果你不想将自己与与测试无关的某些实现绑定在一起，那么测试必须小心编写。

# 抽象工厂 - 工厂的工厂

在学习了工厂设计模式之后，我们将一个相关对象家族（在我们的例子中是支付方法）进行了分组，人们很快就会想到——如果我将对象家族分组到一个更有结构的家族层次结构中会怎样？

## 描述

抽象工厂设计模式是一种新的分组层，用于实现一个更大（和更复杂）的复合对象，通过它的接口来使用。将对象分组到家族中并将家族分组的想法是拥有可以互换并且更容易扩展的大工厂。在开发的早期阶段，使用工厂和抽象工厂比等到所有具体实现都完成后再开始编写代码要更容易。此外，除非你知道你的特定领域的对象库将非常庞大并且可以轻松地分组到家族中，否则你不会从一开始就编写抽象工厂。

## 目标

当你的对象数量增长到需要创建一个唯一的点来获取它们时，将相关的对象组合在一起是非常方便的，这样可以获得运行时对象创建的灵活性。抽象工厂方法的以下目标必须对你清晰明了：

+   为返回所有工厂的通用接口提供新的封装层

+   将常见工厂组合成一个*超级工厂*（也称为工厂的工厂）

## 车辆工厂的例子，又来了？

对于我们的例子，我们将重用我们在生成器设计模式中创建的工厂。我们想展示使用不同方法解决相同问题的相似之处，以便你可以看到每种方法的优势和劣势。这将向你展示 Go 中隐式接口的强大之处，因为我们几乎不用改动任何东西。最后，我们将创建一个新的工厂来创建装运订单。

## 验收标准

以下是使用“车辆”对象工厂方法的验收标准：

+   我们必须使用抽象工厂返回的工厂来检索“车辆”对象。

+   车辆必须是“摩托车”或“汽车”的具体实现，它实现了两个接口（“车辆”和“汽车”或“车辆”和“摩托车”）。

## 单元测试

这将是一个很长的例子，所以请注意。我们将有以下实体：

+   **车辆**：我们工厂中的所有对象必须实现的接口：

+   **摩托车**：一种摩托车的接口，类型为运动（一座）和巡航（两座）。

+   **汽车**：用于豪华车（四门）和家庭车（五门）的汽车接口。

+   **VehicleFactory**：一个接口（抽象工厂），用于检索实现`VehicleFactory`方法的工厂：

+   **摩托车**工厂：一个实现`VehicleFactory`接口的工厂，返回实现`Vehicle`和`Motorbike`接口的车辆。

+   **汽车**工厂：另一个实现`VehicleFactory`接口的工厂，返回实现`Vehicle`和`Car`接口的车辆。

为了清晰起见，我们将每个实体分开放在不同的文件中。我们将从`Vehicle`接口开始，它将在`vehicle.go`文件中：

```go
package abstract_factory 

type Vehicle interface { 
    NumWheels() int 
    NumSeats() int 
} 

```

`Car`和`Motorbike`接口将分别放在`car.go`和`motorbike.go`文件中：

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

我们有一个最后的接口，每个工厂都必须实现这个接口。这将在`vehicle_factory.go`文件中：

```go
package abstract_factory 

type VehicleFactory interface { 
    NewVehicle(v int) (Vehicle, error) 
} 

```

所以，现在我们要声明汽车工厂。它必须实现之前定义的`VehicleFactory`接口，以返回`Vehicles`实例：

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

我们定义了两种类型的汽车--豪华车和家庭车。`car`工厂将返回实现`Car`和`Vehicle`接口的汽车，因此我们需要两种具体的实现：

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

汽车完成了。现在我们需要摩托车工厂，它必须像汽车工厂一样实现`VehicleFactory`接口：

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

对于摩托车工厂，我们还使用`const`关键字定义了两种摩托车类型：`SportMotorbikeType`和`CruiseMotorbikeType`。我们将在`Build`方法中切换`v`参数，以知道应返回哪种类型。让我们写两种具体的摩托车：

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

最后，我们需要抽象工厂本身，我们将把它放在之前创建的`vehicle_factory.go`文件中。

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

我们将编写足够的测试来进行可靠的检查，因为本书的范围并不涵盖 100%的语句。这将是一个很好的练习，让读者完成这些测试。首先是`motorbike`工厂的测试：

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

我们使用包方法`BuildFactory`来检索摩托车工厂（在参数中传递`MotorbikeFactory` ID），并检查是否有任何错误。然后，已经有了摩托车工厂，我们要求一个`SportMotorbikeType`类型的车辆，并再次检查错误。通过返回的车辆，我们可以询问车辆接口的方法（`NumWheels`和`NumSeats`）。我们知道它是一辆摩托车，但是我们不能在不使用类型断言的情况下询问摩托车的类型。我们使用类型断言在车辆上检索摩托车，代码行`sportBike, found := motorbikeVehicle.(Motorbike)`，我们必须检查我们收到的类型是否正确。

最后，现在我们有了摩托车实例，我们可以使用`GetMotorbikeType`方法询问摩托车类型。现在我们要编写一个检查汽车工厂的测试：

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

同样，我们使用`BuildFactory`方法通过参数中的`CarFactoryType`来检索`Car`工厂。使用这个工厂，我们想要一个`Luxury`类型的汽车，以便返回一个`vehicle`实例。我们再次进行类型断言，指向汽车实例，以便我们可以使用`NumDoors`方法询问门的数量。

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

完成。它无法识别任何工厂，因为它们的实现还没有完成。

## 实施

出于简洁起见，每个工厂的实施已经完成。它们与工厂方法非常相似，唯一的区别是在工厂方法中，我们不使用工厂方法的实例，因为我们直接使用包函数。`vehicle`工厂的实现如下：

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

就像在任何工厂中，我们在工厂可能性之间切换，以返回被要求的那个。由于我们已经实现了所有具体的车辆，测试也必须运行：

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

所有测试都通过了。仔细观察并注意，我们在运行测试时使用了`-cover`标志，以返回包的覆盖率百分比：45.8%。这告诉我们的是，45.8%的代码行被我们编写的测试覆盖，但仍有 54.2%没有被测试覆盖。这是因为我们没有用测试覆盖游轮摩托车和家庭汽车。如果你编写这些测试，结果应该会上升到大约 70.8%。

### 提示

类型断言在其他语言中也被称为**转换**。当你有一个接口实例时，它本质上是一个指向结构体的指针，你只能访问接口方法。通过类型断言，你可以告诉编译器指向的结构体的类型，这样你就可以访问整个结构体的字段和方法。

## 关于抽象工厂方法的几行

我们已经学会了如何编写一个工厂的工厂，它为我们提供了一个非常通用的车辆类型对象。这种模式通常用于许多应用程序和库，比如跨平台 GUI 库。想象一个按钮，一个通用对象，以及一个为 Microsoft Windows 按钮提供工厂的按钮工厂，同时你还有另一个为 Mac OS X 按钮提供工厂。你不想处理每个平台的实现细节，而只想为某些特定行为引发的行为实现操作。

此外，我们已经看到了使用两种不同解决方案（抽象工厂和生成器模式）来解决同一个问题时的差异。如你所见，使用生成器模式时，我们有一个无结构的对象列表（在同一个工厂中有汽车和摩托车）。此外，我们鼓励在生成器模式中重用构建算法。在抽象工厂中，我们有一个非常结构化的车辆列表（摩托车工厂和汽车工厂）。我们也没有混合创建汽车和摩托车，提供了更多的灵活性在创建过程中。抽象工厂和生成器模式都可以解决同样的问题，但你的特定需求将帮助你找到应该采用哪种解决方案的细微差别。

# 原型设计模式

我们将在本章中看到的最后一个模式是**原型**模式。像所有创建模式一样，当创建对象时，这也非常方便，而且很常见的是原型模式被更多的模式所包围。

在使用生成器模式时，我们处理重复的构建算法；而在工厂模式中，我们简化了许多类型对象的创建；而在原型模式中，我们将使用某种类型的已创建实例进行克隆，并根据每个上下文的特定需求进行完善。让我们详细看一下。

## 描述

原型模式的目的是在编译时已经创建了一个对象或一组对象，但你可以在运行时克隆它们任意多次。例如，作为刚刚在您的网页上注册的用户的默认模板，或者某项服务中的默认定价计划。与生成器模式的关键区别在于，对象是为用户克隆的，而不是在运行时构建它们。你还可以构建类似缓存的解决方案，使用原型存储信息。

## 目标

原型设计模式的主要目标是避免重复的对象创建。想象一个由数十个字段和嵌入类型组成的默认对象。我们不想每次使用对象时都写这个类型所需的一切，尤其是如果我们可以通过创建具有不同*基础*的实例来搞砸它：

+   维护一组对象，这些对象将被克隆以创建新实例

+   提供某种类型的默认值以便开始在其上进行工作

+   释放复杂对象初始化的 CPU，以占用更多内存资源

## 例子

我们将构建一个想象中的定制衬衫商店的小组件，其中将有一些默认颜色和价格的衬衫。每件衬衫还将有一个**库存保留单位（SKU）**，用于识别存储在特定位置的物品，需要进行更新。

## 验收标准

为了实现示例中描述的内容，我们将使用衬衫的原型。每次我们需要一件新的衬衫，我们将取原型，克隆它并使用它。特别是，这些是在此示例中使用原型模式设计方法的验收标准：

+   为了拥有一个衬衫克隆器对象和接口，以请求不同类型的衬衫（白色、黑色和蓝色分别为 15.00、16.00 和 17.00 美元）

+   当您要求一件白色衬衫时，必须制作白色衬衫的克隆，并且新实例必须与原始实例不同

+   创建的对象的 SKU 不应影响新对象的创建

+   一个信息方法必须给我所有可用的实例字段信息，包括更新后的 SKU

## 单元测试

首先，我们需要一个`ShirtCloner`接口和一个实现它的对象。此外，我们需要一个名为`GetShirtsCloner`的包级函数来检索克隆器的新实例：

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

现在我们需要一个对象结构来克隆，它实现了一个接口来检索其字段的信息。我们将称该对象为`Shirt`和`ItemInfoGetter`接口：

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

### 提示

您是否意识到我们定义的名为`ShirtColor`的类型只是一个`byte`类型？也许您想知道为什么我们没有简单地使用`byte`类型。我们可以，但这样我们创建了一个易于阅读的结构，如果需要，我们可以在将来升级一些方法。例如，我们可以编写一个`String()`方法，返回字符串格式的颜色（类型 1 为`White`，类型 2 为`Black`，类型 3 为`Blue`）。

有了这段代码，我们现在可以编写我们的第一个测试：

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

我们将涵盖我们场景的第一种情况，我们需要一个克隆器对象，可以用来请求不同颜色的衬衫。

对于第二种情况，我们将获取原始对象（因为我们可以访问它，所以我们在包的范围内），并将其与我们的`shirt1`实例进行比较。

```go
if item1 == whitePrototype { 
    t.Error("item1 cannot be equal to the white prototype"); 
} 

```

现在，对于第三种情况。首先，我们将`item1`类型断言为衬衫，以便我们可以设置 SKU。我们将创建第二件衬衫，也是白色，我们也将对其进行类型断言，以检查 SKU 是否不同：

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

我们将打印两件衬衫的内存位置，因此我们在更物理层面上做出这种断言：

```go
t.Logf("LOG: The memory positions of the shirts are different %p != %p \n\n", &shirt1, &shirt2) 

```

最后，我们运行测试，以便检查它是否失败：

```go
go test -run=TestClone . 
--- FAIL: TestClone (0.00s) 
prototype_test.go:10: Not implemented yet 
FAIL 
FAIL

```

我们必须在这里停下来，以免测试在尝试使用`GetShirtsCloner`函数返回的空对象时出现恐慌。

## 实施

我们将从`GetClone`方法开始。这个方法应该返回指定类型的物品，我们有三种类型：白色、黑色和蓝色：

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

现在我们有了三个原型可以操作，我们可以实现`GetClone(s int)`方法：

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

`Shirt`结构还需要一个`GetInfo`实现来打印实例的内容。

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

最后，让我们运行测试，看看现在是否一切正常：

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

在日志中（运行测试时记得设置`-v`标志），您可以检查`shirt1`和`shirt2`的 SKU 是否不同。此外，我们可以看到两个对象的内存位置。请注意，您的计算机上显示的位置可能会有所不同。

## 关于原型设计模式的学习

原型模式是构建缓存和默认对象的强大工具。您可能也意识到了一些模式可能有些重叠，但它们之间有一些细微差别，使它们在某些情况下更合适，在其他情况下则不太合适。

# 总结

我们已经看到了软件行业中常用的五种主要的创建型设计模式。它们的目的是为了将用户从对象的创建中抽象出来，以应对复杂性或可维护性的需求。自上世纪 90 年代以来，它们已经成为成千上万个应用程序和库的基础，而今天我们使用的大多数软件在内部都有许多这些创建型模式。

值得一提的是，这些模式并不是无缝的。在更高级的章节中，我们将会看到如何在 Go 中进行并发编程，以及如何使用并发方法来创建一些更为关键的设计模式。
