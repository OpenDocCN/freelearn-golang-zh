# 编写干净且易于维护的 Go 代码的最佳实践

"任何傻瓜都能写出计算机能理解的代码。优秀的程序员写出人类能理解的代码。"

- Martin Fowler ^([8])

编写易于测试和维护的干净代码比乍看之下要困难得多。幸运的是，作为编程语言，Go 非常具有意见性，并自带一套最佳实践。

如果你查看一些学习 Go 的可用材料（例如，Effective Go ^([6]））或观看核心 Go 团队杰出成员（如 Rob Pike）的演讲，就会很明显，软件工程师在处理自己的 Go 项目时会被温和地 *引导* 应用这些原则。从我的视角和经验来看，这些最佳实践往往对代码质量指标有可衡量的积极影响，同时也有助于最小化技术债务的积累。

在本章中，我们将涵盖以下主题：

+   通过 Go 工程师的眼镜理解面向对象设计的 SOLID 原则

+   在包级别组织源代码

+   在 Go 中编写精简且易于维护的代码的有用技巧和工具

# 面向对象设计的 SOLID 原则

SOLID 原则本质上是一套规则，旨在帮助你编写干净且易于维护的面向对象代码。让我们回顾一下这些首字母代表什么：

+   **单一职责**

+   **开闭原则**

+   **里氏替换原则**

+   **接口隔离原则**

+   **依赖倒置原则**

但等等！Go 是面向对象的编程语言，还是一种带有一些语法糖的功能性编程语言？

与其他传统面向对象编程语言（如 C++ 或 Java）不同，Go 没有内置对类的支持。然而，它 *确实* 支持接口和结构体的概念。结构体允许你将对象定义为字段的集合和相关方法。尽管对象和接口可以组合在一起，但根据设计，没有对经典面向对象继承的支持。

考虑到这些观察结果，我们应该将 Go 作为一个 *基于对象* 的编程语言来引用，并且因此，以下原则仍然有效。让我们从 Go 软件工程师的角度更详细地审视每个原则。

# 单一职责

**单一职责原则**（SRP）是由 Robert Martin ^([23]）描述的，他是一位经验丰富的软件工程师，以 *Uncle Bob* 的昵称提供关于软件开发最佳实践的指导。SRP 表述如下：

"在任何设计良好的系统中，对象应该只具有单一职责。"

简而言之，对象实现应该专注于做好一件事情，并且以高效的方式进行。为了理解这个原则是如何工作的，让我们检查一段违反该原则的代码。在以下虚构的场景中，我们为 ACME 无人机公司工作，我们正在使用 Go 构建基于无人机的货物配送系统。

以下代码片段展示了我们为`Drone`类型定义一组方法的一个初始尝试：

```go
// NavigateTo applies any required changes to the drone's speed 
// vector so that its eventual position matches dst.
func (d *Drone) NavigateTo(dst Vec3) error { //... }

// Position returns the current drone position vector.
func (d *Drone) Position() Vec3 { //... }

// Position returns the current drone speed vector.
func (d *Drone) Speed() Vec3 { //... }

// DetectTargets captures an image of the drone's field of view (FoV) using
// the on-board camera and feeds it to a pre-trained SSD MobileNet V1 neural
// network to detect and classify interesting nearby targets. For more info
// on this model see: 
// https://github.com/tensorflow/models/tree/master/research/object_detection
func (d *Drone) DetectTargets() ([]*Target, error) { //... }
```

上述代码通过混淆两个不同的责任违反了 SRP：

+   驾驶无人机

+   在无人机附近检测目标

在某些情况下，这是一个有效、可行的解决方案。然而，引入的额外耦合使得实现更难维护和扩展。例如，如果我们想评估不同的神经网络模型进行物体识别，怎么办？如果我们想将相同的物体识别代码用于不同的`Drone`类型，怎么办？

那么，我们如何应用单一职责原则（SRP）来改进我们的设计呢？首先，在假设所有无人机都配备有摄像头的情况下，我们可以在`Drone`对象上公开一个方法来捕获并返回使用摄像头拍摄的照片。此时，你可能正在想：等等，图像捕获难道不是与导航不同的责任吗？答案是：这完全取决于视角！描述和分配对象的责任本身就是一门艺术，而且相当主观。相反，我们也可以反驳说，导航需要访问各种传感器数据源，而摄像头就是其中之一。从这个意义上说，所提出的重构并没有违反 SRP。

在第二个重构步骤中，我们可以将目标检测代码提取到一个独立的、独立的对象中，这样我们就可以在不修改`Drone`类型中的任何代码的情况下继续进行对象识别模型评估。实现的第二次迭代可能看起来像这样：

```go
// NavigateTo applies any required changes to the drone's speed vector 
// so that its eventual position matches dst.
func (d *Drone) NavigateTo(dst Vec3) error { //... }

// Position returns the current drone position vector.
func (d *Drone) Position() Vec3 { //... }

// Position returns the current drone speed vector.
func (d *Drone) Speed() Vec3 { //... }

// CaptureImage records and returns an image of the drone's field of 
// view using the on-board drone camera.
func (d *Drone) CaptureImage() (*image.RGBA, error) { //... }
```

在一个单独的文件中（可能也在不同的包中），我们将定义`MobileNet`类型，其中包含我们的目标检测器的实现：

```go
// MobileNet performs target detection for drones using the 
// SSD MobileNet V1 NN.
// For more info on this model see:
// https://github.com/tensorflow/models/tree/master/research/object_detection
type MobileNet {
 // various attributes...
}

// DetectTargets captures an image of the drone's field of view and feeds
// it to a neural network to detect and classify interesting nearby 
// targets.
func (mn *MobileNet) DetectTargets(d *drone.Drone) ([]*Target, error){
 //...
}
```

成功！我们已经将原始实现拆分成了两个独立的对象，每个对象都只有一个单一的责任。

# 开放/封闭原则

**开放/封闭**原则是由伯特兰·迈耶提出的^([24])，他提出了以下观点：

“一个软件模块应该对扩展开放，但对修改封闭。”

几乎所有的 Go 程序都会导入并使用来自众多其他包的类型，其中一些是 Go 标准库的一部分，而其他包则由第三方提供。任何将包导入其代码库的软件工程师都应该始终安全地假设该包导出的所有类型都遵循一个保证不可变的契约。换句话说，一个包不应该能够修改由 *其他* 包导出的类型的行怍。虽然一些编程语言允许这种修改（通过一种俗称为 *猴子补丁* 的技术），但 Go 设计者已经设置了安全机制来确保这种类型的修改是严格禁止的。否则，Go 程序将能够违反 *封闭* 原则，对部署到生产环境的代码产生不可预见的后果。

到目前为止，你可能想知道：封闭原则也适用于包 *内部* 的代码吗？此外，Go 是如何实现 *开放* 原则的？根据 Meyer 的定义，我们应该能够使用面向对象的原则，如继承或组合，以扩展现有代码并添加额外的功能，而无需修改原始代码单元。正如我们在本章开头讨论的那样，Go 不支持继承；这留下了 *组合* 作为扩展现有代码的唯一可行方法。

让我们通过一个简单的例子来考察这些原则是如何交织在一起的。在短暂的无人机设计经历之后，我们决定转换行业，专注于构建角色扮演游戏。在下面的代码中，你可以看到为即将推出的游戏定义的泛型 `Sword` 类型：

```go
type Sword struct {
 name string // Important tip for RPG players: always name your swords!
}
// Damage returns the damage dealt by this sword.
func (Sword) Damage() int {
 return 2
}

// String implements fmt.Stringer for the Sword type.
func (s Sword) String() string {
 return fmt.Sprintf(
 "%s is a sword that can deal %d points of damage to opponents",
 s.name, s.Damage(),
 )
}
```

我们的设计要求之一是我们需要支持魔法物品，例如，一把附魔剑。由于我们的附魔剑仅仅是一个通用的剑，造成不同的伤害量，我们将应用 *开放* 原则，并使用 *组合* 来创建一个新的类型，该类型嵌入 `Sword` 类型并覆盖 `Damage` 方法的实现：

```go
type EnchantedSword struct {
 // Embed the Sword type
 Sword
}

// Damage returns the damage dealt by the enchanted sword.
func (EnchantedSword) Damage() int {
 return 42
}
```

然而，如果没有编写一些基于表格的测试，我们的实现就不可能完整！我们将创建的第一个测试函数被命名为 `TestSwordDamage`，根据其名称，你可以猜到其目的是检查调用我们迄今为止定义的类型上的 `Damage` 是否产生预期的结果。以下是我们将如何以表格驱动的方式定义我们的期望：

```go
specs := []struct {
 sword interface {
 Damage() int
 }
 exp int
}{
 {
 sword: Sword{name: "Silver Saber"},
 exp:   2,
 }, 
 {
 sword: EnchantedSword{Sword{name: "Dragon's Greatsword"}},
 exp:   42,
 },
}
```

`TestSwordDamage` 的实现只是遍历定义的期望并验证每个期望是否得到满足：

```go
func TestSwordDamage(t *testing.T) {
 specs := ... // see above code snippet for the spec definitions

 for specIndex, spec := range specs {
 if got := spec.sword.Damage(); got != spec.exp {
 t.Errorf("[spec %d] expected to get damage %d; got %d", specIndex, spec.exp, got)
 }
 }
}
```

我们的第二个测试附带其自己的期望列表。这次的目标是确保我们之前定义的类型上的 `String` 方法的输出是正确的：

```go
specs := []struct {
 sword fmt.Stringer
 exp   string
}{
 {
 sword: Sword{name: "Silver Saber"},
 exp:   "Silver Saber is a sword that can deal 2 points of damage to opponents",
 }, 
 {
 sword: EnchantedSword{Sword{name: "Dragon's Greatsword"}},
 exp:   "Dragon's Greatsword is a sword that can deal 42 points of 
         damage to opponents",
 },
}
```

这是 `TestSwordToString` 的实现，它与 `TestSwordDamage` 几乎相同；这里没有惊喜：

```go
func TestSwordToString(t *testing.T) {
 specs := ... // see above code snippet for the spec definitions

 for specIndex, spec := range specs {
 if got := spec.sword.String(); got != spec.exp {
 t.Errorf("[spec %d] expected to get\n%q\ngot:\n%q", 
             specIndex, spec.exp, got)
 }
 }
}
```

现在，我们可以运行`go test`。然而，我们的测试中有一个失败了：

```go
$ go test -v
=== RUN   TestSwordDamage
--- PASS: TestSwordDamage (0.00s)
=== RUN   TestSwordToString
--- FAIL: TestSwordToString (0.00s)
        sword_test.go:55: [spec 1] expected to get
                "Dragon's Greatsword is a sword that can deal 42 points of
                 damage to opponents"
                got:
                "Dragon's Greatsword is a sword that can deal 2 points of 
                  damage to opponents"
```

那么，是什么导致了第二个测试失败？为了揭示失败测试背后的原因，我们需要深入挖掘 Go 方法在底层是如何工作的。Go 方法不过是调用一个以对象实例作为参数的**函数**（也称为接收器）的语法糖。在前面的代码片段中，`String`总是以“剑”接收器调用，因此对`Damage`方法的调用**总是**被调度到由“剑”类型定义的实现。

这是一个**封闭**原则在行动中的典型例子：“剑”**不知道**任何可能包含它的类型，并且它的方法集**不能**被它所嵌入的对象所改变。重要的是要指出，尽管“魔法剑”类型不能修改在嵌入的“剑”实例上定义的方法的实现，但它仍然可以**访问**和**修改**它定义的任何字段（如果两种类型都在同一个包中定义，包括私有字段）。

# 里氏替换

我们将要探索的 SOLID 的第三个原则是**里氏替换原则**（**LSP**）。它是由 Barbara Liskov 在 1987 年在**面向对象编程系统、语言和应用**（**OOPSLA**）会议上的主题演讲中提出的。LSP 的正式定义如下：

如果对于每个类型为`S`的对象`O1`，存在一个类型为`T`的对象`O2`，并且对于所有以`T`定义的程序`P`，当用`O1`替换`O2`时，程序`P`的行为保持不变，那么`S`是`T`的子类型。

用通俗的话来说，如果两种类型的展现行为完全遵循相同的契约，那么它们是**可替换的**，因此**调用者**无法区分它们。从纯面向对象的角度思考，这可能是抽象类和具体类的典型用例。正如我们在前面的部分提到的，Go 不支持类或继承的概念，而是依靠**接口**作为促进类型替换的手段。

Go 语言的一个有趣特性，至少对于来自 Java 或 C++ 背景的人来说是这样，就是 Go 接口是**隐式的**。每个 Go 类型定义了一个隐式的接口，该接口由它实现的所有方法组成。这个设计决策允许 Go 编译器在决定一个对象实例是否可以作为函数或方法参数的替代品时，执行一个**编译时**的**鸭子类型**变体（这个术语的正式名称是**结构化类型**）。

鸭子类型这个术语的根源在于一个古老的谚语，被称为**鸭子测试**：

“如果它看起来像一只鸭子，并且它像鸭子一样嘎嘎叫，那么它就是一只鸭子。”

实质上，当给定一个对象和一个接口时，如果该对象的方法集包含与接口定义的方法名称和签名匹配的方法，则该对象可以用作接口的替代。无需显式指出类型实现了哪些接口是一个非常方便的特性。它帮助我们解耦对象的定义（可能是外部或第三方包）与接口定义和/或使用的地方。

在以下代码片段中，我们定义了`Adder`接口和一个简单的名为`PrintSum`的函数，该函数使用满足此接口的任何类型将两个数字相加：

```go
package main

import "fmt"

// Adder is implemented by objects that can add two integers together.
type Adder interface {
 Add(int, int) int
}

func PrintSum(a, b int, adder Adder) {
 fmt.Printf("%d + %d = %d", a, b, adder.Add(a, b))
}
```

`adder`包包括`Int`类型，它满足`Adder`接口，还有一个名为`Double`的类型，它不满足；尽管它定义了一个名为`Add`的函数，但你将注意到*参数类型*是不同的：

```go
package adder

// Int adds two integer values.
type Int struct{}

// Add returns the sum a+b.
func (Int) Add(a, b int) int { return a + b }

// Double adds two double values.
type Double struct{}

// Add returns the sum a+b.
func (Double) Add(a, b float64) float64 { return a + b }
```

以下代码片段说明了编译时接口替换检查是如何工作的。我们可以安全地将`Int`实例传递给`PrintSum`，因为`Int`隐式满足`Adder`接口。然而，尝试将`Double`实例传递给`PrintSum`将触发编译时错误：

```go
package main

import "github.com/foo/adder"

func main() {
 PrintSum(1, 2, adder.Int{}) // prints: "1 + 2 = 3"

 // This line will trigger a compile-time error:
 //  cannot use adder.Double literal (type adder.Double) as type Adder 
 //  in argument to PrintSum: adder.Double does not implement Adder 
 //  (wrong type for Add method) 
 //      have Add(float64, float64) float64
 //      want Add(int, int) int
 PrintSum(1, 2, adder.Double{})
}
```

在对象要替换的类型在编译时未知的情况下，编译器将自动生成代码以在运行时执行检查：

```go
var placeholder interface{}

// Cast to io.Reader works; os.Stdin implements io.Reader
placeholder = os.Stdin
_ = placeholder.(io.Reader)

// Cast to io.Reader triggers a run-time panic:
// "panic: interface conversion: string is not io.Reader: missing method Read"
placeholder = "cast check"
_ = placeholder.(io.Reader)

// Cast to io.Reader fails and isReader is set to false
placeholder = "cast check"
if _, isReader := placeholder.(io.Reader); !isReader {
 fmt.Printf("%T does not implement io.Reader\n", placeholder)
}
```

当你不确定类型实例或`interface{}`在运行时是否可以转换为另一个类型或接口时，通常是一个好习惯使用双重返回值的类型转换操作符（前述代码样本中的*最后一个*情况）来避免程序执行时的潜在恐慌。

# 接口隔离

与 SRP 类似，**接口隔离原则**（ISP）也是由罗伯特·马丁提出的。根据这个原则，客户端不应该被迫依赖于它们不使用的接口。

这个原则非常重要，因为它构成了应用我们之前讨论的其他原则的基础。回到我们之前的 RPG 示例，假设我们已经给我们的`Sword`对象添加了一些更有趣的方法：

```go
// Sharpen increases the damage dealt by this sword using a whetstone.
func (Sword) Sharpen() {
 //...
}

// MakeBlunt decreases the damage dealt by this sword due to constant use.
func (Sword) MakeBlunt(){
 //...
}

// Drop places the sword on the ground allowing others to pick it up.
func (Sword) Drop(){
 //...
}
```

那么，我们如何在游戏中使用武器呢？显然，我们需要引入一些*怪物*供玩家攻击！这可能是一个`Attack`函数的潜在签名：

```go
// Attack deals damage to a monster using a sword.
func Attack(m *Monster, s *Sword) {
 //...
}
```

然而，前述定义存在一些问题。

*隐式*（参见上一节）的`Sword`接口非常*开放*，也就是说，它包含了一堆我们的`Attack`实现不需要的其他方法。实际上，实现`Attack`的软件工程师可能会倾向于包括一些额外的*业务逻辑*规则，这些规则依赖于那些方法的可用性：

+   在攻击了一定次数后使剑变钝

+   如果怪物使用一些特殊装甲，导致玩家丢弃剑

沿着这条道路走下去将违反 SRP，并可能使代码的单元测试变得更加困难。此外，提出的`Attack`定义与`Sword`类型或嵌入它的其他类型的对象产生了强烈的耦合。

这两个观察结果完全证明了以下著名的 Go 谚语的合理性，该谚语最初归功于 Rob Pike：

"接口越大，抽象越弱。"

虽然 Go 的隐式接口（见上一节）允许我们传递任何嵌入`Sword`（例如，可能是我们之前示例中的`EnchantedSword`）的类型，但我们的要求无疑会声明`Attack`必须能够与其他类型的武器一起工作，例如，投射物或魔法咒语。

另一方面，`Attack`期望其第一个参数是一个`Monster`实例。从逻辑上讲，玩家应该能够使用武器对非怪物实体造成伤害，例如，破坏螺栓固定的门或切断悬挂在天花板上的吊灯的绳子。此外，理想情况下，我们希望当怪物攻击玩家时能够重用相同的实现。

这些都是应用 ISP 的绝佳用例。假设我们的`Attack`实现只需要以下内容：

+   为了确定武器造成的伤害量

+   一种将伤害应用于特定**目标**的机制

基于上述观察，我们可以将`Attack`函数的签名更改为接受两个显式接口作为参数：

```go
// DamageReceiver is implemented by objects that can receive weapon damage.
type DamageReceiver interface {
 ApplyDamage(int)
}

// Damager is implemented by objects that can be used as weapons.
type Damager interface {
 Damage(int)
}

// Attack deals weapon damage to target.
func Attack(target DamageReceiver, weapon Damager) {
 //...
}
```

通过这个相当简单的改动，我们一举两得。首先，我们的代码更加抽象，通过提供我们自己的实现所需接口的测试类型，可以更容易地测试其行为。其次，我们的接口实际上是最小的；这一事实不仅使得 SRP 的应用成为可能，而且也暗示了一个更简单的实现。正如对 Go 标准库的快速浏览所证实的，单方法接口（例如，`io`包中的`Reader`和`Writer`接口）是 Go 作者之间相当普遍的惯用法。

# 依赖倒置

另一个由罗伯特·马丁（Robert Martin）提出的原理是**依赖倒置原则**（DIP）。它稍微有点冗长，定义如下：

"高级模块不应依赖于低级模块。两者都应依赖于抽象。抽象不应依赖于细节。细节应依赖于抽象。"

DIP 实质上总结了我们迄今为止讨论的所有其他原则。如果你已经将 SOLID 原则的其余部分应用于你的代码库，你会发现它已经符合前面的定义！

接口的引入和使用有助于解耦高级和低级模块。开放/封闭原则确保接口本身是不可变的，但这并不妨碍我们提出任何数量的替代实现（前述定义中的*细节*部分），以满足隐式或显式的接口。同时，LSP 保证我们可以在依赖既定抽象的同时，也拥有在编译时甚至运行时灵活替换底层实现的能力，而不用担心破坏我们的应用程序。

# 应用 SOLID 原则

如果你决定将这些原则应用到自己的项目中，你将在设计、连接和测试软件组件的方式上获得更大的灵活性，并且在未来扩展代码库时所需的时间会更少。

然而，有一点需要记住的是，*没有免费的午餐*。你在灵活性方面获得的收益，你会在代码库的增大中失去；这可能会对项目的复杂度指标产生不利影响。

在我看来，这种权衡并不一定是坏事。通过遵循测试代码的最佳实践（将在后续章节中详细探讨），你可以驯服任何潜在的代码复杂度增加。同时，在编写测试时遇到困难通常是代码可能违反一个或多个 SOLID 原则并需要重构的好迹象。

最后，我想强调的是，尽管我们是通过 Go 工程师的视角来分析 SOLID 原则，但原则本身具有更广泛的适用范围，也可以应用于系统设计。例如，在基于微服务的部署中，你应该旨在构建和部署具有单一目的（SRP）的服务，并通过明确定义的合同和边界进行通信（ISP）。

# 将代码组织成包

如前节所述，应用 SOLID 原则可以作为指导，将我们的代码库拆分成更小的包，其中每个包实现特定的功能，其接口在构建更大系统时作为连接包的粘合剂。

在本节中，我们将探讨命名包的 Go 语言惯用方法，以及你在编写依赖于复杂包依赖图的代码时可能遇到的一些常见潜在陷阱。

# Go 包的命名约定

假设你遇到一个名为 `server` 的包。根据前面的建议，这个名字好吗？嗯，*显然*，我们可以猜测它是一种服务器，但那是什么类型的服务器呢？是 HTTP 服务器，比如基于 TCP 实现基于文本的线协议的服务器，或者可能是一个基于 UDP 的在线游戏服务器？有些人可能会争论，这个包可能导出一种类型或函数，暗示了包的目的（例如，`NewHTTPServer`）。这确实消除了歧义，但也引入了一点重复：在这个特定的情况下，*server* 文字既出现在包名中，也出现在它暴露的函数中。正如我们将在 *使用 linter 提高代码质量指标* 部分中看到的那样，这种做法被认为是一种反模式，可能会导致 linter 警告。

Go 包名应简短、简洁，并为包的 *预期* 用户提供对其用途的明确指示。

通过浏览 Go 标准库的代码，我们可以找到许多这种清晰包命名哲学的特征性例子：

+   `net` 包提供了创建各种类型网络监听器的机制（TCP、UDP、Unix 域套接字等）。

+   `net/http` 包提供了许多功能，其中包括 HTTP 服务器实现：`http.Server` 类型名称在用途上相当明确。

尽管应该保持包名简短，但你应避免提出可能与其他代码中常用变量名冲突的包名。否则，包用户将不得不使用别名（即导入 *blah* path-to-package）来导入包。在这种情况下，通常最好（如果可能的话）缩写包名。Go 标准库中的典型例子包括 `fmt` 和 `bufio` 包。更具体地说，`bufio` 包之所以命名为如此，是为了避免与 `buf` 这个变量名发生冲突，你很可能在处理使用缓冲区的代码时会遇到这个变量名。

最后，与其他标准库通常附带实用库或具有通用名称（如 *common* 或 *util*）的包的编程语言相比，Go 对这种做法持 *反对* 的态度。这实际上是从 SOLID 原则的角度来看是有道理的，因为这些包更有可能违反 SRP，而恰当地命名的包则通过其名称强制内容具有逻辑边界。此外，随着发布的 Go 包数量随着时间的推移而增长，搜索和定位具有通用名称的包将变得越来越困难。

# 循环依赖

要使 Go 程序结构良好，其导入图必须是无环的；换句话说，它不能包含任何循环。任何违反此谓词的行为都会导致 Go 编译器发出错误。随着你构建的系统复杂性增加，最终遇到令人讨厌的“检测到导入循环”错误的概率也会增加。

通常，导入循环是软件解决方案设计中存在缺陷的迹象。幸运的是，在许多情况下，我们可以重构我们的代码，并绕过大多数导入循环。让我们更仔细地看看循环依赖通常发生的一些常见情况以及处理它们的策略。

# 通过隐式接口打破循环依赖

在以下虚构的场景中，我们为一家初创公司工作，该公司正在构建负责控制全自动仓库的软件。配备抓取臂和激光的自主机器人（可能出什么问题？）正忙于在仓库地板上移动，定位并从货架上取订单物品，并将它们放入随后被运送给客户的纸箱中。

这是仓库 `Robot` 的一个临时定义：

```go
package warehouse

import "context"

// Robot navigates the warehouse floor and fetches items for packing.
type Robot struct {
 // various fields
}

// AcquireRobot blocks until a Robot becomes available or until the 
// context expires.
func AcquireRobot(ctx context.Context) *Robot { //...  }

// Pack instructs the robot to pick up an item from its shelf and place 
// it into a box that will be shipped to the customer.
func (r *Robot) Pack(item *entity.Item, to *entity.Box) error { //...  }
```

在前面的代码片段中，`Item` 和 `Box` 类型位于一个名为 `entity` 的外部包中。一切顺利，直到有一天有人试图向 `Box` 类型引入一个新的辅助方法，不幸的是，这引入了一个导入循环：

```go
package entity

// Box contains a list of items that are shipped to the customer.
type Box struct {
 // various fields
}

// Pack qty items of type i into the box.
func (b *Box) Pack(i *Item, qty int) error {
 robot := warehouse.Acquire() // **compile error: import cycle detected**
 // ...
}
```

从技术上来说，这是一个糟糕的设计决策：箱子和物品实际上不应该知道机器人的存在。然而，为了这个论点，我们将忽略这个设计缺陷，并尝试使用 Go 对隐式接口的支持来解决这个问题。第一步是在 `entity` 包内定义一个 `Packer` 接口。其次，我们需要提供一个获取 `Packer` 实例的抽象，如下面的代码片段所示：

```go
package entity 

import "context"

// Packer is implemented by objects that can pack an Item into a Box.
type Packer interface {
 Pack(*Item, *Box) error
}

// AcquirePacker returns a Packer instance.
var AcquirePacker func(context.Context) Packer
```

在这两个机制到位的情况下，辅助方法可以在不需要导入 `warehouse` 包的情况下工作：*无需* 导入 `warehouse` 包。

```go
// Pack qty items of type i into the box.
func (b *Box) Pack(i *Item, qty int) error {
 p := AcquirePacker(context.Background())
 for j := 0; j < qty; j++ {
 if err := p.Pack(i, b); err != nil {
 return err 
 }
 }
 return nil
}
```

我们需要解决的最后一个难题是如何在不导入 `warehouse` 包的情况下初始化 `AcquirePacker`。我们唯一能这样做的方式是通过一个导入 `warehouse` 和 `entity` 包的 *第三个* 包：

```go
package main

import "github.com/achilleasa/logistics/entity"
import "github.com/achilleasa/logistics/warehouse"

func wireComponents() {
 entity.AcquirePacker = func(ctx context.Context) entity.Packer {
 return warehouse.AcquireRobot(ctx)
 }
}
```

在前面的代码片段中，`wireComponents` 函数确保 `warehouse` 和 `entity` 包被连接在一起，而不会触发任何循环依赖错误。

# 有时候，代码重复并不是一个坏主意！

你可能之前听说过 **不要重复自己** （**DRY**）原则。DRY 的主要思想是通过编写可重用的代码来避免代码重复，这些代码可以在需要的地方包含。但 DRY 是否 *总是* 一个好主意？

Go 包作为组织代码到模块化和可重用单元的不错抽象。但，一般来说，编写 Go 程序的良好实践是尽量保持你的导入依赖图浅而宽；考虑到这可能是 DRY 原则所倡导的“包含而非重复”的相反，这听起来可能有些反直觉。

当依赖图变得更深时，循环依赖的可能性也会增加，这次是由于**传递性**依赖，即你代码导入的包的依赖。在以下示例中，我们有三个包：`x`、`y`和`z`。

包`y`定义了一个名为`IsPrime`的辅助函数，正如你可能从其名称中猜测到的，它返回一个布尔值，指示其输入是否为素数。同一个包导入并使用来自包`z`的一些类型：

```go
package y

import "z"

func IsPrime(v uint64) bool {
 // ... 
}

// Other functions referencing types exported from package z
```

包`z`从包`x`导入了一些类型：

```go
package z

import "x"

// functions referencing types exported from package x
```

到目前为止，一切顺利。几天后，我们决定向包`x`添加一个新的辅助函数，名为`IsValid`。该函数需要进行素性测试，由于包`y`已经提供了`IsPrime`，我们决定遵循 DRY 原则，将`y`导入到我们的代码中，从而造成循环依赖：

```go
package x

import "y" // circular dependency: x imports y, y imports z and z imports x

func IsValid(v uint64) bool {
 return v != 0 && y.IsPrime(v)
}
```

在这种情况下，如果我们需要的包含包中的代码足够小，我们可以直接复制它（包括其测试）并避免触发循环依赖的额外导入。正如一句流行的 Go 谚语所说：

“一点复制胜过一点依赖。”

# 编写精简且易于维护的 Go 代码的技巧和工具

在接下来的章节中，我们将介绍一些技术、工具和最佳实践，这些可以帮助你编写更简洁、更易于测试的代码，同时也能帮助你从同事和代码审查员那里获得一些赞誉。

我们将要讨论的大部分主题都是特定于 Go 语言的，但其中的一些原则可以推广并应用于其他编程语言和软件工程领域。

# 优化函数实现以提高可读性

在我大学早期，我的计算机科学教授们会坚决主张保持函数块短小精悍。他们的建议如下：

“如果一个函数实现无法适应单个屏幕，那么它必须被拆分成更小的函数。”

请记住，这些指南的根源在于一个时代，那时人们通过“屏幕”来指代能够适应 80×25 字符终端的代码量！快进到今天，情况已经发生了变化：软件工程师可以访问高分辨率的显示器、编辑器和预装了广泛复杂分析和重构工具的定制 IDE。尽管如此，对于编写易于他人审查、扩展和维护的代码，这些建议依然同样重要。

在*单一职责*部分，我们讨论了 SRP 的优点。不出所料，同样的原则也适用于函数块，这是你在编码时需要牢记在心的事情。

通过将复杂函数分解为更小的函数，代码变得更加易于阅读和推理。这起初可能看起来并不重要，但想想这种情况：你几个月没有接触代码，然后需要重新深入其中，同时试图追踪一个错误。作为额外的好处，独立的逻辑块也更容易进行测试，特别是如果你遵循编写表格驱动测试的实践。

自然地，同样的方法也可以应用于现有代码。如果你发现自己正在导航一个包含深层嵌套的`if`/`else`块、重复的代码块或其实施处理几个看似不相关的关注点的长函数，那么应用一些即兴重构并提取任何潜在的独立逻辑块到单独的函数中将会是一个极好的机会。

此外，在创建新函数或将现有函数拆分为更小的函数时，一个好的主意是将函数排列得在它们定义的文件中按调用顺序出现，也就是说，如果`A()`调用`B()`和`C()`，那么`B()`和`C()`都必须出现在下面，但不一定紧接在`A()`之后。这使得其他工程师（或只是好奇想了解某物是如何工作的普通人）浏览现有代码变得更加容易。

每条规则都有例外，这条规则也不例外。除非编译器非常擅长内联函数，否则将业务逻辑分散到多个函数中有时会对性能造成影响。尽管在许多情况下性能损失微不足道，但当最终目标是生成包含紧密内循环或预期高频调用的代码时，将实现整齐地封装在单个函数中可能是一个好主意，以避免在调用函数时产生的额外 Go 运行时开销（例如，将参数推送到栈上，检查调用者是否有足够的栈空间，以及函数调用返回时从栈上弹出东西）。

代码的可读性和性能之间总是存在权衡。当处理复杂系统时，可读性通常更受欢迎，但最终，这取决于你和你的团队来确定哪种可读性和性能的混合最适合你的特定用例。

# 变量命名约定

关于 Go 程序中变量和类型名称的理想长度，存在持续的争论。一方面，有支持所有变量都应该有清晰且自我描述性名称的信仰者。这对于在 Java 生态系统编写过代码的人来说是一种相当常见的哲学。另一方面，我们有*简约主义者*，即那些主张使用较短标识符名称的人，他们认为较长的标识符太冗长。

Go 语言作者显然属于后一种阵营。以两个最受欢迎的 Go 接口`io.Reader`和`io.Writer`的定义为例：

```go
type Reader interface {
 Read(p []byte) (n int, err error)
}

type Writer interface {
 Write(p []byte) (n int, err error)
}
```

同样的短标识符模式在 Go 标准库代码库中被广泛使用。我认为，只要未来的工程师能够轻松理解它们在各自作用域内的用途，使用*较短*但仍然*描述性*的变量名是好事。

这种方法的常见例子是为嵌套循环命名索引变量，通常使用单个字母变量，如`i`、`j`等。然而，在以下代码片段中，索引变量被用来访问多维切片`s`的元素：

```go
for i := 0; i < len(s); i++ {
 for j := 0; j < len(s[i]); j++ {
 value := s[i][j]
 // ...
 }
}
```

如果有人不熟悉这段代码，被分配去审查包含前面循环的拉取请求，他们可能会发现自己难以弄清楚`s`是什么以及每个索引级别代表什么！由于缩写的变量名几乎不提供关于它们真正用途的信息，为了回答这些问题，审查者将不得不在代码库中四处寻找线索：查找`s`的类型，然后转到其定义，等等。现在，将前面的代码块与以下一个执行完全相同功能的代码块进行对比，它使用了*略微更长*的变量名。在我看来，第二种方法具有更高的信息量，同时避免了过于冗长：

```go
for dateIdx := 0; dateIdx < len(tickers); dateIdx++ {
 for stockIdx := 0; stockIdx < len(tickers[dateIdx]); stockIdx++ {
 value := tickers[dateIdx][stockIdx]
 // ...
 }
}
```

最后，每个工程师都有自己的首选变量命名方法和哲学。在决定采用哪种方法时，试着花几分钟考虑一下你的变量命名选择如何影响与你共同在共享代码库上协作的其他工程师。

# 高效使用 Go 接口

"接受接口，返回结构体。"

*- 杰克·林达穆德*

将代码组织成包背后的关键点是，通过提供一个干净、文档齐全的 API 界面，使代码可重用，并以无摩擦的方式供外部消费者使用。当编写接受具体类型作为参数的函数或方法时，我们对我们实现的有用性施加了一个人为的限制：它只能与特定类型的实例一起工作。

虽然这不一定总是问题，但在某些情况下，要求具体的类型实例可能会使测试变得复杂且缓慢，尤其是如果构建此类实例是一个昂贵的操作。以下摘录是关于一个收集并将性能指标发布到键值存储的系统的一部分。

键值存储的实现如下所示：

```go
package kv

// Store implements a key-value store which stores data to disk.
type Store struct { // ...  }

func Open(path string) (*Store, error) { // Open path, load and verify data, replay pending transactions etc.  }

// Put persists (key, value) to the store.
func (s *Store) Put(key string, value interface{}) error { // ...  }

// Get looks up the value associated with key.
func (s *Store) Get(key string) (interface{}, error) { // ...  }

// Close waits for any pending transactions to complete and then 
// cleanly shuts down the KV store.
func (s *Store) Close() error { // ...  }
```

在 `metrics` 包中，我们可以找到 `ReportMetrics` 函数的定义。它接收一个 `kv.Store` 实例作为参数，并将收集到的指标持久化到其中：

```go
package metrics

// ReportMetrics writes the collected metrics to a KV store instance.
func (c *Collector) ReportMetrics(s *kv.Store) error {
 // for each metric call s.Put(k, v)
}

// Observe records a value for a particular metric.
func (c *Collector) Observe(metric string, value interface{}) {
 // ...
}
```

根据之前关于 SOLID 原则的讨论，你应该已经发现了这段代码的问题：*它只与特定的键值存储实现一起工作*！如果我们想将指标发布到网络套接字、写入 CSV 文件，或者可能记录到控制台会怎样呢？

然而，这段代码还有一个问题：测试它需要相当多的努力。为了理解原因，让我们设身处地地考虑一下这个包的消费者。作为我们集成测试套件的一部分，我们想要确保所有收集到的指标实际上都写入了键值存储实例。

首先，我们的测试代码必须创建一个键值存储的实例。由于 `Open` 方法需要一个文件，而我们可能会同时运行多个测试，因此我们需要创建一个临时的唯一文件，并将其作为参数传递给 `Open`。当然，我们不应该在测试运行完成后留下临时文件，因此我们需要确保我们的测试会在自己完成后进行清理：

```go
// Generate a random file for the KV store
tmpfile, err := ioutil.TempFile("", "metrics")
if err != nil {
 t.Fatal(err)
}
defer func() { _ = os.Remove(tmpfile.Name()) }() // clean up when we are 
                                                 // done
_ = tmpfile.Close()

// Create KV store
s, err := kv.Open(tmpfile.Name())
if err != nil {
 t.Fatal(err)
}
defer func() { _ = s.Close() }()
```

这就带我们来到了测试的核心：创建一个指标收集器，用大量测量值填充它，将捕获的指标报告给键值存储，并验证一切是否已正确写入存储：

```go
c := metrics.NewCollector()
for i := 0; i < 100; i++ {
 c.Observe(fmt.Sprintf("metric_%d", i), i)
}

if err = c.ReportMetrics(s); err != nil {
 t.Fatal(err)
}

// Ensure that all metrics have been written to the store
// ...
}
```

这个测试相当直接，但设置它需要相当多的样板代码。此外，`kv.Open` 看起来是一个相当昂贵的调用；想象一下，如果我们的测试套件由数百个测试组成，每个测试都需要一个真实的 `kv.Store` 实例，那么涉及的额外开销有多大。另一方面，如果 `ReportMetrics` 接收一个接口作为参数，我们就可以在测试时传递一个内存模拟，同时保留将指标报告到满足该特定接口的任何目的地的灵活性。因此，我们可以通过引入一个接口来改进前面的代码：

```go
package metrics 

// Sink is implemented by objects that metrics can be reported to.
type Sink interface {
 Put(k string, v interface{}) error
}

// ReportMetrics writes the collected metrics to a Sink.
func (c *Collector) ReportMetrics(s Sink) error {
 // for each metric call s.Put(k, v)
}
```

这个小小的改动让测试变得轻而易举！我们可以单独测试 `kv.Store` 代码，并切换到内存存储来运行所有单元测试：

```go
func TestReportMetrics(t *testing.T) {
 // Use in-memory store defined inside the test package
 s := new(inMemStore)

 // Create collector and populate some metrics
 c := metrics.NewCollector()
 for i := 0; i < 100; i++ {
 c.Observe(fmt.Sprintf("metric_%d", i), i)
 }

 if err = c.ReportMetrics(s); err != nil {
 t.Fatal(err)
 }

 // Ensure that all metrics have been written to the store...
}
```

林达穆德给出的另一条建议是，我们应该始终尝试返回具体类型而不是接口。这条建议实际上是有道理的：作为一个包的消费者，如果我在调用创建类型`Foo`的函数，我可能对调用该类型特定的一个或多个方法感兴趣。如果`NewFoo`函数返回一个接口，客户端代码将不得不手动将其转换为`Foo`，以便调用`Foo`特定的方法；这会违背最初返回接口的目的。

还很重要的一点是指出，在大多数情况下，实现将创建一个具体类型的实例；我们最初选择返回接口的主要原因是为了确保我们的具体类型始终满足特定的接口。本质上，我们正在给代码添加一个编译时检查！然而，有更简单的方法引入这样的编译时检查，同时仍然保留构造函数返回具体实例的能力：

```go
package metrics

import "fmt"

// Compile-time checks for ensuring a type implements a particular 
// interface.
var (
 // Works but allocates a dummy Foo instance on the heap.
 _ fmt.Stringer = &Foo{}

 // Preferred way that does not allocate anything on the heap.
 _ fmt.Stringer = (*Foo)(nil)
)

type Foo struct { }

func (*Foo) String() string { return "Foo" }
```

上述代码片段概述了两种相当常见的方法，通过定义一对使用保留的*空白标识符*作为提示的全球变量来实现编译时检查，该提示告知编译器它们实际上没有被使用。

# 零值是你的朋友

Go 提供的一个很棒的功能是，每个类型在实例化时都会自动分配其零值。Go 及其标准库中的一些有趣示例如下：

+   Go 通道；空通道会无限期阻塞尝试从中读取的 goroutine

+   Go 切片的零值；这是一个可以添加内容的空切片

+   `sync.Mutex`类型，其零值表示互斥锁处于解锁状态

+   `bytes.Buffer`类型，其零值表示一个空缓冲区

通过在设计新类型时依赖零值，我们可以提供开箱即用的实现，无需显式调用构造函数或其他初始化方法。以下代码片段定义了一个简单、线程安全的映射：

```go
package main

import (
 "fmt"
 "sync"
)

// SyncMap implements a thread-safe map. The zero SyncMap value is ready 
// to use.
type SyncMap struct {
 mu   sync.RWMutex
 data map[string]interface{}
}
```

实际上用于存储`SyncMap`数据的 Go 映射实例将在我们尝试向映射中添加项时懒加载。在处理底层映射之前获取一个*写者*互斥锁确保映射的初始化和项的插入以原子方式发生：

```go
// Put inserts a key-value pair into the map.
func (sm *SyncMap) Put(key string, value interface{}) {
 sm.mu.Lock()
 defer sm.mu.Unlock()

 if sm.data == nil {
 sm.data = make(map[string]interface{})
 }

 sm.data[key] = value
}
```

查找实现相当直接。与`Put`实现的明显不同之处在于，`Get`在执行查找之前会获取一个*读者*互斥锁。使用读者/写者互斥锁提供了对多个读者的并发访问，同时只允许单个写者修改映射的内容：

```go
// Get returns the value associate by key and a boolean value indicating
// whether key is present in the map.
func (sm *SyncMap) Get(key string) (interface{}, bool) {
 sm.mu.RLock()
 defer sm.mu.RUnlock()

 if sm.data == nil {
 return nil, false
 }

 return sm.data[key]
}
```

与需要通过调用`make`显式初始化的内置 Go 映射类型不同，`SyncMap`的零值可以直接安全使用：

```go
func main() {
 var sm SyncMap // we are using the zero value of the map
 sm.Put("foo", "bar")
 fmt.Println(sm.Get("foo")) // Prints: bar true
}
```

更重要的是，我们可以将前面的`SyncMap`实现嵌入到遵循相同零值模式的其它类型中，以提供需要无初始化的复杂类型：

```go
type Foo struct {
 bar Bar
}

type Bar struct {
 sm SyncMap
}

func main() {
 var foo Foo // still using a zero value
 foo.bar.sm.Put("answer", 42) // storing into the embedded map also 
                                 // works.
}
```

在前面的代码片段中，`SyncMap`实例已经准备好可以使用，并且可以通过`Foo`实例直接访问，而无需编写任何额外的代码来设置它。

# 使用工具分析和操作 Go 程序

Go 程序本身很容易解析。事实上，Go 库提供了内置的包，可以将 Go 程序解析为**抽象语法树**（**ASTs**），这些树可以被遍历、修改，并转换回 Go 代码。让我们通过一个简单的例子来看看使用这些包有多容易。

首先，我们需要一个辅助函数将 Go 程序转换为 AST 表示。下面的`parse`函数正是这样做的：

```go
import (
 "fmt"
 "go/ast"
 "go/parser"
 "go/token"
)

// parse a Go program into an AST representation.
func parse(program string) (*token.FileSet, *ast.File, error) {
 fs := token.NewFileSet()
 tree, err := parser.ParseFile(fs, "example.go", program, 0)
 if err != nil {
 return nil, nil, err
 }
 return fs, tree, nil
}
```

`ast` 包提供了一些辅助函数，实现了访问者模式，并为 AST 中的每个节点调用用户定义的回调。对于这个特定的例子，我们将定义一个名为`inspectVariables`的函数，该函数将访问 AST 中的每个节点，寻找对应于标识符（包、常量、类型、变量、函数或标签）的节点。对于每个发现的标识符，该函数将检查其`Kind`属性，如果标识符代表变量，则打印其名称：

```go
// inspectVariables visits each AST node and prints any encountered Go variable.
func inspectVariables(fs *token.FileSet, tree *ast.File) {
 ast.Inspect(tree, func(n ast.Node) bool {
 ident, ok := n.(*ast.Ident)
 if !ok || ident.Obj == nil || ident.Obj.Kind != ast.Var {
 return true
 }

 fmt.Printf("%s:\tvariable %q\n", fs.Position(n.Pos()), ident)
 return true
 })
}
```

为了完成我们的示例，我们需要提供一个`main`函数，该函数将解析一个简单的程序，并在生成的 AST 上调用`inspectVariables`：

```go
func main() {
 fs, tree, err := parse(`
 package foo 

 var global = "foo"

 func main(){ x := 42 }
 `)

 if err != nil {
 fmt.Printf("ERROR: %v\n", err)
 return
 }

 inspectVariables(fs, tree)
}
```

运行前面的程序会产生以下输出：

```go
$ go run print_vars.go
example.go:4:7: variable "global"
example.go:6:16: variable "x"
```

Go 生态系统包含大量工具，这些工具建立在解析基础设施之上，并为软件工程师提供分析、代码修改和生成服务。在接下来的章节中，我们将考察一些这样的工具，它们可以使您的软件开发生活更加轻松。

# 注意格式化和导入（gofmt，goimports）

制表符还是空格？开括号前是否应该有换行符？所有工程师在需要为新项目或刚刚继承的项目选择特定的代码格式化风格时，最终都会面临这个困境。

源代码格式化风格一直是团队成员之间长期、有时激烈的争论的主题。与其它编程语言相反，Go 在设计上对特定的格式化风格有很强的意见，并附带工具来帮助强制执行该特定风格。这个设计决策是有意义的，因为 Go 最初是为了被成千上万的谷歌雇佣的工程师使用而创建的。在这个庞大的开发规模下，作者代码的一致性不仅仅是一种优雅，实际上对于确保代码可以在开发团队之间传递是至关重要的。

`gofmt` 工具^([11])作为标准 Go 发行版的一部分提供。您提供它一个或多个文件的路径，它可以执行以下任务：

+   按照推荐的标准格式化代码

+   简化代码 (`gofmt -s example.go`)

+   执行简单的重写 (`gofmt -r 'a[b:len(a)] -> a[len(a):b]' example.go`)

默认情况下，`gofmt` 将格式化后的程序输出到标准输出。然而，用户可以通过传递 `-w` 标志来强制 `gofmt` 将其输出写回到它刚刚处理的源文件。

`goimports` 工具^([12]) 是 `gofmt` 的一个直接替代品，可以通过运行 `go get golang.org/x/tools/cmd/goimports` 来安装。除了提供与 `gofmt` 输出匹配的代码格式化功能外，`goimports` 还管理 Go 的导入行：它可以填充缺失的导入并删除未被处理文件引用的导入。更重要的是，`goimports` 还确保包按字母顺序排序并分组，这取决于它们是否属于标准库或第三方包。

以下是一个展示了一些问题的 Go 程序示例：缺失的导入、未使用的导入、多余的空白字符和错误的缩进：

```go
package main

import (
 "net" // Mixed stdlib and third-party packages
 "github.com/achilleasa/kv"
 "fmt" // Unused package 
)

type Server struct {
 ctx    context.Context // missing referenced import
 socket net.Conn
store *kv.Store // Incorrectly indented field definition
}

func foo(){} // Redundant line-breaks above foo()
```

这些工具的典型用法是将它们作为你最喜欢的编辑器或 IDE 的后保存钩子来执行。如果我们对前面的代码片段运行 `goimports`，我们将得到一个整洁的格式化输出：

```go
package main

import (
 "context"
 "net"

 "github.com/achilleasa/kv"
)

type Server struct {
 ctx    context.Context
 socket net.Conn
 store *kv.Store
}

func foo() {}
```

如你所见，导入语句已经被清理和排序，缺失的包已经被导入，未使用的包已经被删除。除此之外，代码现在已经被正确缩进，并且所有多余的空白字符都被移除了。

# 在包之间重构代码（gorename, gomvpkg, fix）

有时，你可能会在函数中遇到一个具有奇怪或非描述性名称的变量，你非常想重命名它。执行此类重命名操作非常简单；只需选择函数块并运行查找和替换操作。简单得就像做饼一样！

但如果你想要重命名一个公共结构体字段或从你的包中导出的函数呢？这绝对不是一个简单任务，因为你需要追踪所有对要重命名的对象（列表可能还包括其他包）的引用，并将它们更新为使用新名称。这种重命名操作将我们带入代码重构的领域；幸运的是，我们有工具可以自动化这个繁琐的任务：`gorename`^([16])。可以通过运行 `go get golang.org/x/tools/cmd/gorenam*e*` 来安装它。

`gorename` 的一个有趣特性，除了它可以在包之间工作之外，还在于它是 *类型感知的*。由于它在应用任何重命名操作之前依赖于解析程序，因此它足够智能，可以区分名为 `Foo` 的函数和具有相同名称的结构体字段。此外，它还增加了一层额外的安全性，即它只会应用重命名操作，只要最终结果是能够无错误编译的代码片段。

有时，你可能需要重命名一个 Go 包，甚至将其移动到同一项目或不同项目中的不同位置。`gomvpkg` 工具 ^([15]) 可以在此方面提供帮助，同时还会追踪依赖于重命名/移动包的包，并更新它们的导入语句以指向新的包位置。可以通过运行 `go get golang.org/x/tools/cmd/gomvpkg` 来安装它。移动包就像运行以下命令一样简单：

```go
# Rename foo to bar and update all imports for packages depending on foo.
$ gomvpkg -from foo -to bar
```

自 2011 年首次稳定版 Go 版本发布以来，Go 标准库在多年中发生了很大变化。在引入新 API 的同时，其他 API 被弃用并最终被移除。在某些情况下，现有的 API 被修改，通常是非向后兼容的方式，而在其他情况下，外部或实验性包最终被接受纳入标准库。

一个相关的例子是 `context` 包。在 Go 1.7 之前，这个包位于 `golang.org/x/net/context`，并且许多 Go 程序都在积极使用它。但随着 Go 1.7 的发布，该包成为了标准库的一部分，并移动到了独立的 `context` 包。一旦工程师切换到上下文包的新导入路径，他们的代码就会立即与仍在使用旧导入路径的代码不兼容。因此，有人必须承担审查现有代码库并重写现有导入以指向上下文包新位置的艰巨任务！

Go 设计者预见到了这些问题，并创建了一个基于规则的工具来检测依赖于旧、已弃用的 API 或包的代码，并自动将其重写为使用新 API。这个工具被恰当地命名为 `fix` ([`golang.org/cmd/fix/`](https://golang.org/cmd/fix/))，它随着每个新的 Go 版本一起发布，每次切换到新版本的 Go 时都可以通过运行 `go tool fix $path` 来调用它。重要的是要指出，所有应用到的修复都是 *幂等的*；因此，可以安全地多次运行该工具，而不会使代码库损坏。

# 使用 linters 提高代码质量指标

Linters 是专门用于解析 Go 文件并尝试检测、标记和报告以下情况的静态分析工具：

+   代码不符合标准格式化风格指南；例如，它包含多余的空白，缩进不正确，或包含有拼写错误的注释

+   程序中可能存在逻辑错误；例如，变量声明 *覆盖* 前一个具有相同名称的变量声明，以错误的参数数量或与格式字符串类型不匹配的参数调用函数，如 `fmt.Printf`，将值赋给变量但实际上没有使用它们，没有检查函数调用返回的错误，等等

+   代码可能包含安全漏洞；例如，它包含硬编码的安全凭证或指向可能使用不安全的随机数源或密码学上损坏的哈希原语（DES、RC4、MD5 或 SHA1）进行 SQL 注入的位置

+   代码表现出高复杂度（例如，深度嵌套的 `if`/`else` 块）或包含不必要的类型转换、未使用的局部或全局变量，或从未被调用的代码路径

以下表格总结了您可以使用以检查您的程序并提高您编写的代码质量指标的最受欢迎的 Go 代码检查工具： 

| **类别** | **代码检查工具** | **描述** |
| --- | --- | --- |
| 逻辑错误 | `bodyclose` ^([2]) | 检查 `http.Response` 主体是否始终关闭。 |
| 逻辑错误 | `errcheck` ^([7]) | 识别未检查返回的错误的情况。 |
| 逻辑错误 | `gosumcheck` ^([19]) | 确保正确处理类型切换的所有可能情况。 |
| 逻辑错误 | `go vet` ([`golang.org/cmd/vet/`](https://golang.org/cmd/vet/)) ^([20]) | 报告可疑的结构，例如，使用错误的参数调用 `fmt.Printf`。 |
| 逻辑错误 | `ineffassign` ^([21]) | 检测未被使用的变量赋值。 |
| 代码异味 | `deadcode` ([`github.com/tsenart/deadcode`](https://github.com/tsenart/deadcode)) ^([4]) | 报告未使用的代码块。 |
| 代码异味 | `dupl` ([`github.com/mibk/dupl`](https://github.com/mibk/dupl)) ^([5]) | 报告可能重复的代码块。 |
| 代码异味 | `goconst` ([`github.com/jgautheron/goconst`](https://github.com/jgautheron/goconst)) ^([9]) | 标记可以替换为常量的重复字符串。 |
| 代码异味 | `structcheck` ([`gitlab.com/opennota/check`](https://gitlab.com/opennota/check)) ^([30]) | 识别未使用的结构体字段。 |
| 代码异味 | `unconvert` ([`github.com/mdempsky/unconvert`](https://github.com/mdempsky/unconvert)) ^([31]) | 检测不必要的类型转换。 |
| 代码异味 | `unparam` ([`github.com/mvdan/unparam`](https://github.com/mvdan/unparam)) ^([33]) | 检测未使用的函数参数。 |
| 代码异味 | `varcheck` ^([34]) | 检测未使用的变量和常量。 |
| 性能 | `aligncheck` ^([1]) | 识别由于填充而占用更多空间的低效打包的结构体。 |
| 性能 | `copyfighter` ([`github.com/jmhodges/copyfighter`](https://github.com/jmhodges/copyfighter)) ^([3]) | 报告通过值传递大型结构体的函数；这种模式触发内存分配并增加垃圾收集器的压力。 |
| 性能 | `prealloc` ^([26]) | 识别可以预先分配的切片声明。 |
| 复杂度 | `gocyclo` ^([10]) | 计算 Go 函数的圈复杂度。 |
| 复杂度 | `gosimple` ^([18]) | 报告可以简化的代码。 |
| 复杂度 | `splint` ^([29]) | 识别过长或接收过多参数的函数。 |
| 安全 | `gosec` ([`github.com/securego/gosec`](https://github.com/securego/gosec)) ^([17]) | 扫描源代码以查找潜在的安全问题。 |
| 安全 | `safesql` ([`github.com/stripe/safesql`](https://github.com/stripe/safesql))^( [28]) | 检查潜在的 SQL 注入点。 |
| 风格 | `gofmt -s` ([`golang.org/cmd/gofmt/`](https://golang.org/cmd/gofmt/)) ^([11]) | 确保文件格式符合 `gofmt` 规则。 |
| 风格 | `golint` ([`github.com/golang/lint`](https://github.com/golang/lint)) ^([14]) | 报告与《Effective Go》中概述的建议不符的样式偏差。 |
| 风格 | `misspell` ([`github.com/client9/misspell`](https://github.com/client9/misspell)) ^([25]) | 使用字典来识别注释中的拼写错误。 |
| 风格 | `unindent` ([`github.com/mvdan/unindent`](https://github.com/mvdan/unindent)) ^([32]) | 识别未正确缩进的代码。 |

在您的项目中使用上述 linters 伴随着一些需要注意的问题。首先，每个 linter 都使用自己的输出格式来报告检测到的问题。当您尝试将 linters 与您首选的编辑器或 IDE 工作流程集成时，缺乏标准化的报告问题方式成为一个问题（例如，跳转到检测到问题的代码位置）。其次，每个 linter 都不知道其他 linter 的存在。因此，每次运行时，每个 linter 都需要重新解析所有包。当您处理小型代码库时，这通常不是问题，但当您处理大型项目时，这会变得令人烦恼，因为所有 linters 的完整运行可能需要几分钟才能完成。

要解决上述问题，您可以使用**元 linter**（也称为**linter 输出聚合器**）工具，例如 `golangci-lint` ^([13])（现在已弃用的 gometalinter 的替代品）或 revive ^([27]）。这些工具旨在并行执行可配置的 linter 列表，标准化它们的输出，消除重复警告，甚至根据正则表达式抑制警告（当您的项目包含由其他工具自动生成的文件时，这是一个非常实用的功能）。更重要的是，它们还无缝集成到工程师使用的绝大多数编辑器中。调用这些元 linter 工具的一个简单方法是向您的项目 *makefile* 中添加一个目标：

```go
lint: 
    golangci-lint run \
      --no-config --issues-exit-code=0 --deadline=30m \
      --disable-all --enable=deadcode  --enable=gocyclo --enable=golint \
      --enable=varcheck --enable=structcheck --enable=errcheck \
      --enable=dupl --enable=ineffassign \
      --enable=unconvert --enable=goconst --enable=gosec
```

为 linting 创建一个 makefile 规则，使得将 linters 作为常规 CI 流程的一部分运行变得容易，并且直到 lint 错误得到解决，阻止拉取请求合并。同时，它还提供了在您在代码库上工作时本地运行 linters 的灵活性。

工程师在创建拉取请求之前跳过运行 linters 是很常见的情况，这需要额外的提交来处理 lint 错误。您可以通过利用大多数版本控制系统（Git 是一个例子）支持某种预提交或预推送钩子，并让您的 VCS 自动为您运行 linters 来避免这种情况。

# 摘要

在本章的第一节“面向对象设计的 SOLID 原则”中，我们深入探讨了每个 SOLID 原则及其如何应用于编写干净的 Go 代码：

+   **SRP**：根据目的将结构体和函数分组，并将它们组织到具有清晰逻辑边界的包中。

+   **开放/封闭原则**：使用简单类型的组合和嵌入来构建更复杂类型，同时仍然保留它们所包含类型的相同隐式接口。

+   **LSP**：通过使用接口而不是具体类型来定义包之间的合同，避免不必要的耦合。

+   **ISP**：确保您的函数或方法签名只依赖于它们需要的操作，而不需要更多；使用尽可能小的接口来描述函数/方法参数，并避免与具体类型的实现细节耦合。

+   **DIP**：在设计代码时使用适当的抽象级别，以解耦高级和低级模块，同时确保实现细节依赖于抽象而不是相反。

在本章的中间部分，我们讨论了将代码组织到包中的主题，确定了您应该避免的常见包命名陷阱，并讨论了导入循环的概念，包括其成因。然后，我们概述了缓解循环依赖问题的策略。

最后，我们讨论了一些有用的技巧和工具，您可以使用它们来帮助编写易于推理和您的软件工程同事审查和维护的干净代码。

随着您的 Go 项目规模的增长，您无疑会注意到代码库中包导入语句数量的增加。这是相当正常的，坦白说，如果您在创建包时应用 SOLID 原则，这是预期的。然而，导入数量的增加，尤其是如果它们是由您无法控制的第三方编写的，这也需要某种过程来确保即使外部依赖项发生变化，您的程序仍然可以按预期编译。这是下一章的主要内容。

# 问题

1.  SOLID 首字母缩略词代表什么？

1.  为什么以下代码片段违反了 SRP？您会如何重构它以确保它不违反 SRP？

```go
import (
 "crypto/ecdsa"
)

type Document struct { //... }

// Append adds a line to the end of the document.
func (d *Document) Append(line string) { //...  }

// Content returns the document contents as a string.
func (d *Document) Content() string { //... }

// Sign calculates a hash for the document contents, signs it with the
// provided private key and returns back the result.
func (d *Document) Sign(pk *ecdsa.PrivateKey) (string, error) { //... }
```

1.  ISP 背后的主要概念是什么？讨论您将如何应用它来改进以下函数签名：

```go
// write a set of lines to a file. 
func write(lines []string, f *os.File) error {
 //...
}
```

1.  解释为什么*util*被认为是一个不太理想的 Go 包名称。

1.  为什么导入循环是 Go 程序的问题？

1.  列举使用零值设计新 Go 类型的优点。

# 进一步阅读

1.  `aligncheck`: 识别效率低下的结构体打包。URL: [`gitlab.com/opennota/check`](https://gitlab.com/opennota/check).

1.  `bodyclose`: 一个静态分析工具，用于检查 `res.Body` 是否正确关闭。URL: [`github.com/timakin/bodyclose`](https://github.com/timakin/bodyclose).

1.  `copyfighter`: 静态分析 Go 代码并在通过值传递大结构体时报告函数。URL: [`github.com/jmhodges/copyfighter`](https://github.com/jmhodges/copyfighter).

1.  `deadcode`: 报告未使用的代码块。URL: [`github.com/tsenart/deadcode`](https://github.com/tsenart/deadcode).

1.  `dupl`: 报告潜在的代码块重复。URL: [`github.com/mibk/dupl`](https://github.com/mibk/dupl).

1.  Effective Go: 编写清晰、惯用的 Go 代码的技巧。

1.  `errcheck`: 确保返回的错误被检查。URL: [`github.com/kisielk/errcheck`](https://github.com/kisielk/errcheck).

1.  佛勒，马丁：*重构：现有代码的设计改进.* 波士顿，马萨诸塞州，美国：Addison-Wesley，1999 — ISBN 0-201-48567-2 ([`www.worldcat.org/title/refactoring-improving-the-design-of-existing-code/oclc/863697997`](https://www.worldcat.org/title/refactoring-improving-the-design-of-existing-code/oclc/863697997)).

1.  `goconst`: 标记可以替换为常量的重复字符串。URL: [`github.com/jgautheron/goconst`](https://github.com/jgautheron/goconst).

1.  `gocyclo`: 计算代码的圈复杂度。URL: [`github.com/alecthomas/gocyclo`](https://github.com/alecthomas/gocyclo).

1.  `gofmt`: 格式化 Go 程序或检查它们是否格式正确。URL: [`golang.org/cmd/gofmt/`](https://golang.org/cmd/gofmt/).

1.  `goimports`: 通过添加缺失的导入行和删除未引用的导入行来更新 Go 导入行。URL: [`godoc.org/golang.org/x/tools/cmd/goimports`](https://godoc.org/golang.org/x/tools/cmd/goimports).

1.  `golangci-lint`: 检查器运行器。URL: [`github.com/golangci/golangci-lint`](https://github.com/golangci/golangci-lint).

1.  `golint`: 报告 Go 程序中的样式问题。URL: [`github.com/golang/lint`](https://github.com/golang/lint).

1.  `gomvpkg`: 移动 Go 包并更新导入声明。URL: [`godoc.org/golang.org/x/tools/cmd/gomvpkg`](https://godoc.org/golang.org/x/tools/cmd/gomvpkg).

1.  `gorename`: 在 Go 源代码中执行精确的类型安全重命名。URL: [`godoc.org/golang.org/x/tools/cmd/gorename`](https://godoc.org/golang.org/x/tools/cmd/gorename).

1.  `gosec`: 扫描源代码以查找潜在的安全问题。URL: [`github.com/securego/gosec`](https://github.com/securego/gosec).

1.  `gosimple`: 报告可以潜在简化的代码。URL: [`github.com/dominikh/go-tools/tree/master/cmd/gosimple`](https://github.com/dominikh/go-tools/tree/master/cmd/gosimple).

1.  `gosumcheck`：确保类型切换中所有可能的类型都得到适当处理。URL：[`github.com/haya14busa/gosum`](https://github.com/haya14busa/gosum)。

1.  `go vet`：检查 Go 源代码并报告可疑结构，例如参数与格式字符串不匹配的`printf`调用或被遮蔽的变量。URL：[`golang.org/cmd/vet/`](https://golang.org/cmd/vet/)。

1.  `ineffassign`：检测未被使用的变量赋值。URL：[`github.com/gordonklaus/ineffassign`](https://github.com/gordonklaus/ineffassign)。

1.  Liskov, Barbara: 主题演讲 - 数据抽象和层次结构。在：《面向对象编程系统、语言和应用（增补）会议论文集》，OOPSLA '87。纽约，纽约，美国：ACM，1987 — ISBN 0-89791-266-7 ([`www.worldcat.org/title/oopsla-87-addendum-to-the-proceedings-object-oriented-programming-systems-languages-and-applications-october-4-8-1987-orlando-florida/oclc/220450625`](https://www.worldcat.org/title/oopsla-87-addendum-to-the-proceedings-object-oriented-programming-systems-languages-and-applications-october-4-8-1987-orlando-florida/oclc/220450625))，第 17-34 页。

1.  Martin, Robert C.：*《整洁架构：软件结构和设计的工匠指南》，罗伯特·C·马丁系列*。波士顿，马萨诸塞州：普伦蒂斯·霍尔，2017 — ISBN 978-0-13-449416-6 ([`www.worldcat.org/title/clean-architecture-a-craftsmans-guide-to-software-structure-and-design-first-edition/oclc/1105785924`](https://www.worldcat.org/title/clean-architecture-a-craftsmans-guide-to-software-structure-and-design-first-edition/oclc/1105785924))。

1.  Meyer, Bertrand：*面向对象软件构造*。第 1 版。上萨德尔河，新泽西州，美国：普伦蒂斯-霍尔公司，1988 — ISBN 0136290493 ([`www.worldcat.org/title/object-oriented-software-construction/oclc/1134860513`](https://www.worldcat.org/title/object-oriented-software-construction/oclc/1134860513))。

1.  `misspell`：检查源代码中的拼写错误。URL：[`github.com/client9/misspell`](https://github.com/client9/misspell)。

1.  `prealloc`：识别可以预先分配的切片声明。URL：[`github.com/alexkohler/prealloc`](https://github.com/alexkohler/prealloc)。

1.  `revive`：golint 的更严格、可配置、可扩展且美观的替代品。URL：[`github.com/mgechev/revive`](https://github.com/mgechev/revive)。

1.  `safesql`：检查代码中潜在的 SQL 注入点。URL：[`github.com/stripe/safesql`](https://github.com/stripe/safesql)。

1.  `splint`：识别过长或接收过多参数的函数。URL：[`github.com/stathat/splint`](https://github.com/stathat/splint)。

1.  `structcheck`：识别未使用的结构体字段。URL：[`gitlab.com/opennota/check`](https://gitlab.com/opennota/check)。

1.  `unconvert`：检测不必要的类型转换。URL：[`github.com/mdempsky/unconvert`](https://github.com/mdempsky/unconvert)。

1.  `unindent`: 识别错误缩进的代码。URL: [`github.com/mvdan/unindent`](https://github.com/mvdan/unindent).

1.  `unparam`: 检测未使用的函数参数。URL: [`github.com/mvdan/unparam`](https://github.com/mvdan/unparam).

1.  `varcheck`: 检测未使用的变量和常量。URL: [`github.com/opennota/check`](https://github.com/opennota/check).
