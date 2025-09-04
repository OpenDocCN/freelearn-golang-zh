# 2

# 将函数视为一等公民

如我们在上一章中确立的，我们函数式程序的核心部分将是函数。在本章中，我们将详细探讨为什么在将函数视为**一等公民**的语言中函数如此强大。Go 默认将函数作为一等公民，这意味着我们默认获得这种功能。越来越多的语言正在选择这种方法。在本章中，我们将看到这将如何允许我们创建有趣的构造，这将提高我们代码的可读性和可测试性。

具体来说，我们将涵盖以下主题：

+   首类函数的优点

+   为函数定义类型

+   将函数当作对象使用

+   匿名函数与命名函数的比较

+   在数据类型或结构体中存储函数

+   使用所有前面的内容创建一个函数分发器

# 技术要求

本章的所有示例都可以在[`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter2`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter2)找到。对于这个示例，任何 Go 版本都可以使用

# 首类函数的优点

在我们讨论“一等函数”之前，让我们首先定义在编程语言设计中称任何事物为“一等”的含义。当我们谈论“一等公民”时，我们指的是一个实体（对象、原始数据类型或函数），对于该实体，所有常见的语言操作都是可用的。这些操作包括赋值、将其传递给函数、从函数返回或将其存储在另一种数据类型（如映射）中。

看这个列表，我们可以看到所有这些操作通常都适用于我们在语言中定义的结构体。对象和原始数据类型可以在函数之间传递。它们通常作为函数的结果返回，并且我们确实将它们分配给变量。当我们说函数是一等公民时，你可以简单地将其视为将函数当作对象来对待。它们的等价性将帮助我们创建本书中的所有未来构造。这将提高*可测试性*，例如，通过允许我们模拟结构体的函数，以及提高*可读性*，例如，通过移除单个函数分发器的庞大 switch 语句。

# 为函数定义类型

Go 是一种静态类型语言。尽管如此，我们不必为每个赋值指定类型——类型在底层已经存在。实际上，编译器会为我们处理这一点。当我们用 Go 处理函数时，它们也会隐式地被分配一个类型。虽然以编译器的方式为函数定义类型是一项困难的任务，但我们可以使用函数别名概念来为我们的代码库增加类型安全性。

在本书的其余部分处理函数时，我们经常会使用**类型别名**。这将帮助编译器提供更易读的错误信息，并且通常使我们的代码更易读。然而，类型别名不仅适用于函数的上下文。它是 Go 的一个很棒的功能，但并不常用。这也是你在其他主流语言中不太容易找到的功能。所以，让我们深入了解类型别名是什么。

实质上，类型别名正是它所说的那样；它为类型创建了一个别名。这类似于在 Unix 系统中为命令创建别名的方式。它帮助我们创建一个具有与原始类型相同属性的新类型。我们可能想要这样做的一个原因是为了可读性，正如我们在创建函数别名时将看到的。另一个原因是当我们编写代码时，更清晰地传达我们的意图。例如，我们可以使用我们的类型系统将`CountryID`和`CityID`定义为`String`的别名。尽管这两种类型在底层都是字符串，但在代码中它们不能互换使用。因此，它们向读者传达了实际期望的值。

## 原始类型别名

在面向对象的语言中，一个常见的模式是 OO 语言变成了`Person`结构体，而我们想要在这个结构体上设置一个电话号码：

```go
type Person struct {
	name        string
	phonenumber string
}
func (p *Person) setPhoneNumber(s string) {
	p.phonenumber = s
}
```

在这个例子中，它受到了 Java 的很大影响，我们正在创建一个类似于“setter”的函数，该函数接受`phonenumber`作为字符串输入并相应地更新我们的对象。如果你使用的是提供函数类型提示的 IDE，它将告诉你`setPhoneNumberfunction`期望一个字符串，这意味着任何字符串都是有效的。现在，如果我们有一个类型别名，我们可以使这个提示更有用。

那么，让我们做一些修改，并使用类型别名`phoneNumber`：

```go
type phoneNumber string
type Person struct {
	name        string
	phonenumber phoneNumber
}
func (p *Person) setPhoneNumber(s phoneNumber) {
	p.phonenumber = s
}
```

通过这个修改，我们的类型现在更清楚地传达了我们的意图，并且没有创建一个新结构体来模拟电话号码的开销。我们可以这样做，因为电话号码本质上可以看作是一个字符串。

使用这个，因为类型别名等同于底层类型，就像使用一个真正的字符串一样简单：

```go
func main() {
	p := Person{
		name:        "John",
		phonenumber: "123",
	}
	fmt.Printf("%v\n", p)
}
```

好的，太棒了。所以，我们有一个名字，它只是一个字符串，还有一个`phonenumber`，它是一个`phoneNumber`类型，它等于一个字符串。那么，好处从哪里来呢？嗯，一部分是在传达意图中获得的。代码被比原作者更多的人阅读，所以我们的代码要尽可能清晰。另一部分是在错误信息中。使用类型别名，错误信息将明确告诉我们期望什么，而不仅仅是说期望一个字符串。让我们创建一个可以更新`name`和`phonenumber`的函数，并且首先使用`string`为两者：

```go
func (p *Person) update(name, phonenumber string) {
	p.name = name
	p.phonenumber = phonenumber
}
```

当我们尝试编译我们的代码时会发生什么？嗯，我们将得到以下错误：

```go
./prog.go:26:18: cannot use phonenumber (variable of type 
  string) as type phoneNumber in assignment
```

在这个简单的例子中，它并没有做什么。但随着你的代码库的扩展，这确保了所有开发者都在思考应该传递给函数的类型。这通过向函数传递无效数据来降低错误的风险。根据 IDE 的不同，还有一个额外的优点，即你的 IDE 也会显示签名。如果你有一个接受五种不同类型字符串的大函数，你的 IDE 可能只会显示*函数期望输入（string, string, string, string, string）*，没有任何明确的参数传递顺序。如果每个字符串都是不同的类型，这可能会变成*name, phonenumber, email, street, country*。特别是在像 Go 这样的语言中，单字母变量名经常被使用，这可以带来可读性的好处。

为了让我们的代码工作，我们只需要对函数签名进行一个小改动：

```go
func (p *Person) update(name string, phonenumber phoneNumber) {
	p.name = name
	p.phonenumber = phonenumber
}
```

这是一个简单的修复，仅仅是一个小的改动，但持续这样做可以让你的代码仅通过类型系统传达更多的意义。最终，类型的存在是为了向其他读者以及编译器传达意义。

让我们来看看类型别名的一个额外好处。让我们给我们的结构体添加一个带有其自己的类型别名的`age`字段：

```go
type age uint
type Person struct {
	name        string
	age         age
	phonenumber phoneNumber
}
```

在 Go 中，我们不能像对`uint`这样的原始类型那样，将函数附加到它们上。然而，当我们分配一个类型别名时，这种限制就消失了。因此，现在我们可以将函数附加到`age`类型上，这实际上就是将函数附加到`uint`上：

```go
func (a age) valid() bool {
	return a < 120
}
func isValidPerson(p Person) bool {
	return p.age.valid() && p.name != ""
}
```

在前面的代码中，我们正在创建一个`valid`函数，它绑定到`age`类型。现在，在其他函数中，我们可以使用熟悉的点符号在类型上调用`valid()`函数。这个例子可能有点微不足道，但它是一些在原始类型上无法工作的事情。

如果我们尝试将一个函数附加到一个原始类型上，我们将无法编译我们的程序：

```go
func (u uint) valid() bool {
	return u < 120
}
```

这会抛出以下错误：

```go
./prog.go:30:7: cannot define new methods on non-local type 
  uint
Go build failed.
```

仅此一项就让类型别名变得非常强大。这也意味着你现在可以扩展你代码库中不是由你创建的类型。你可能正在使用一个公开结构的第三方库，但你想要向它添加自己的功能。实现这一点的其中一种方法是通过创建类型别名并使用你自己的功能来扩展它。虽然深入这个例子超出了我们本章要探讨的范围，但可以简单地说，类型别名是一个强大的结构。

## 函数的类型别名

由于在 Go 中函数是一个*一等公民*，我们可以像处理任何其他数据类型一样处理它们。因此，就像我们可以为变量或结构体创建类型别名一样，我们也可以为函数创建类型别名。

我们为什么要这样做呢？我们代码的读者获得的主要好处将是它带来的清晰度和可读性。看看以下`filter`函数的代码：

```go
func filter(is []int, predicate func(int) bool) []int {
	out := []int{}
	for _, i := range is {
		if predicate(i) {
			out = append(out, i)
		}
	}
	return out
}
```

这个函数是使用函数作为一等公民的一个很好的例子。在这里，`predicate`函数是一个传递给`filter`函数的函数。它以我们通常传递对象的方式传递。

如果我们想要清理这个函数签名，我们可以引入一个类型别名并重写过滤器函数：

```go
type predicate func(int) bool
func filter(is []int, p predicate) []int {
	out := []int{}
	for _, i := range is {
		if p(i) {
			out = append(out, i)
		}
	}
	return out
}
```

在这里，你可以看到第二个参数现在接受`predicate`类型。编译器将把这个类型转换为`func(int) bool`，但我们可以只在代码库中写`predicate`。

引入类型别名的好处之一是，我们的错误消息变得更加易读。让我们想象一下，我们向`filter`传递了一个不遵循`predicate`类型声明的函数：

```go
	filter(ints, func(i int, s string) bool { return i > 2 })
```

没有类型别名，错误消息的读法如下：

```go
./prog.go:9:15: cannot use func(i int, s string) bool {…} 
(value of type func(i int, s string) bool) as type func(int) 
bool in argument to filter
```

这是一个错误消息，虽然非常明确，但读起来相当冗长。有了类型别名，消息将告诉我们期望的是哪种类型的函数：

```go
./prog.go:9:15: cannot use func(i int, s string) bool {…} 
(value of type func(i int, s string) bool) as type predicate in 
argument to filter
```

# 将函数作为对象使用

在前面的章节中，我们看到了如何创建类型别名来使我们的代码在处理函数时更加易读。在本节中，让我们简要地看看函数如何像对象一样使用。这就是*一等*的含义。

## 将函数传递给函数

我们可以将函数传递给函数，就像前面的过滤器函数那样：

```go
type predicate func(int) bool
func largerThanTwo(i int) bool {
	return i > 2
}
func filter(is []int, p predicate) []int {
	out := []int{}
	for _, i := range is {
		if p(i) {
			out = append(out, i)
		}
	}
	return out
}
func main() {
	ints := []int{1, 2, 3}
	filter(ints, largerThanTwo)
}
```

在这个例子中，我们创建了一个`largerThanTwo`函数，它遵循`predicate`类型别名。请注意，我们不必在任何地方指定这个函数遵循我们的`predicate`类型；编译器将在编译时解决这个问题，就像它对常规变量所做的那样。接下来，我们创建了一个`filter`函数，它期望一个`ints`切片以及一个`predicate`函数。在我们的`main`函数中，我们创建了一个`ints`切片，并使用`largerThanTwo`函数作为第二个参数调用`filter`函数。

## 内联函数定义

我们不需要在包作用域中创建像`largerThanTwo`这样的函数。我们可以像创建内联结构体一样创建内联函数：

```go
func main() {
	// functions in variables
	inlinePersonStruct := struct {
		name string
	}{
		name: "John",
	}
	ints := []int{1, 2, 3}
	inlineFunction := func(i int) bool { return i > 2 }
	filter(ints, inlineFunction)
}
```

`inlinePersonStruct`在这个代码中作为内联函数与内联结构体定义比较的例子。实际上，由于这个结构体在`main`函数的其余部分没有使用，所以代码不会编译。

## 匿名函数

我们还可以在需要的地方动态创建函数。这些被称为*匿名函数*，因为它们没有分配给它们的名字。继续使用我们的`filter`函数，一个`largerThanTwo`谓词的匿名函数版本看起来是这样的：

```go
func main() {
	filter([]int{1, 2, 3}, func(i int) bool { return i > 2 })
}
```

在前面的例子中，我们既创建了一个整数切片，也创建了内联的谓词函数。它们都没有命名。切片不能在该`main`函数的任何其他地方引用，函数也是如此。虽然这类函数定义会使我们的代码更加冗长，并可能阻碍可读性，但我们将看到它们在*第三章*和*第四章*中的应用。

## 从函数中返回函数

任何编程语言的核心概念之一是从函数中返回一个值。由于函数被当作一个普通对象来处理，我们可以从一个函数中返回一个函数。

在前面的例子中，我们的`largerThanTwo`谓词函数始终检查一个整数是否大于两个。现在，让我们创建一个可以生成此类谓词函数的函数：

```go
func createLargerThanPredicate(threshold int) predicate {
	return func(i int) bool {
		return i > threshold
	}
}
```

在这个例子中，我们创建了一个`createLargerThanPredicate`函数，它返回一个`predicate`。记住，类型`predicate`只是一个类型别名，代表一个接受整数作为输入并返回 bool 作为输出的函数。接下来，我们在函数体中定义我们返回的函数。

我们返回的函数遵循`predicate`的类型签名，如果`i`大于`threshold`则返回 true。请注意，`i`函数并没有传递给`createLargerThanPredicate`函数本身。我们是在内联定义的。当我们调用`createLargerThanPredicate`函数时，我们得到的不是谓词函数的结果，而是一个遵循内部签名的新的函数：

```go
func main() {
	ints := []int{1, 2, 3}
	largerThanTwo := createLargerThanPredicate(2)
	filter(ints, largerThanTwo)
}
```

在这里，在`main`函数中，我们首先调用`createLargerThanPredicate(2)`函数。这返回一个新的`func(i int) bool`函数。这里的`2`指的是`threshold`参数，而不是`i`参数。

在下一行，我们再次可以使用新创建的`largerThanTwo`函数调用`filter`函数。

当我们深入研究更高级的主题，如*传递风格*编程和函数柯里化时，从函数中返回函数将是一个核心概念。目前，主要的收获是这允许我们即时创建可定制的函数。例如，我们可以创建一系列具有各自阈值的“大于”谓词：

```go
 func main() {
	largerThanTwo := createLargerThanPredicate(2)
	largerThanFive := createLargerThanPredicate(5)
	largerThanHundred := createLargerThanPredicate(100)
}
```

注意，这个例子无法编译，因为我们没有在`main`块的其余部分使用这些函数。但这展示了我们如何基本上“生成”具有一个固定参数的函数。我们不必在函数块内创建这些函数，而是可以将它们移动到包特定的`var`块。

## 函数在 var 中

继续前面的例子，我们可以创建一系列可以在整个包中使用的函数：

```go
var (
	largerThanTwo     = createLargerThanPredicate(2)
	largerThanFive    = createLargerThanPredicate(5)
	largerThanHundred = createLargerThanPredicate(100)
)
```

这些“函数工厂”使我们能够在整个代码中创建一些自定义函数。这里需要注意的是，这将在`var`块内部工作，但如果我们将这些移动到`const`块，则无法编译：

```go
const (
	largerThanTwo      = createLargerThanPredicate(2)
	largerThanFive     = createLargerThanPredicate(5)
	largerThanHundred  = createLargerThanPredicate(100)
)
```

这将生成以下错误：

```go
./prog.go:8:23: createLargerThanPredicate(2) (value of type 
predicate) is not constant
./prog.go:9:23: createLargerThanPredicate(5) (value of type 
predicate) is not constant
./prog.go:10:23: createLargerThanHundred(100) (value of type 
predicate) is not constant
```

我们的函数从包的角度来看不被认为是“常量”。

## 数据结构内部的函数

到目前为止，我们创建了一大堆函数，这些函数要么是在顶层 `var` 块中定义的，要么是在函数内联定义的。如果我们想在应用程序的运行时内存中某个地方存储我们的函数怎么办？

好吧，就像我们可以在运行时内存中存储原始类型和结构体一样，我们也可以在那里存储函数。

让我们从将我们的 `largerThan` 谓词存储在数组中开始。我们将谓词声明移回 `var` 块，并在 `main` 函数中传递给 `filter` 函数：

```go
var (
	largerThanTwo     = createLargerThanPredicate(2)
	largerThanFive    = createLargerThanPredicate(5)
	largerThanHundred = createLargerThanPredicate(100)
)
func main() {
	ints := []int{1, 2, 3, 6, 101}
	predicates := []predicate{largerThanTwo, largerThanFive, 
        largerThanHundred}
	for _, predicate := range predicates {
		fmt.Printf("%v\n", filter(ints, predicate))
	}
}
```

在前面的例子中，我们创建了一个“谓词切片”。类型将是 `[]predicate`，作为声明的一部分，我们还将我们之前创建的三个谓词推送到这个切片中。在这行代码之后，切片包含对三个函数的引用：`largerThanTwo`、`largerThanFive` 和 `largerThanHundred`。

一旦我们创建了切片，我们就可以像任何常规切片一样迭代它。当我们写 `for _, predicate := range predicates` 时，`predicate` 的值将依次取我们存储在切片中的每个函数的值。因此，当我们为每个后续迭代打印过滤函数的输出时，我们得到以下内容：

```go
[3 6 101]
[6 101]
[101]
```

在第一次迭代中，`predicate` 指的是 `largerThanTwofunction`；在第二次迭代中，它变为 `largerThanFive`，最后变为 `largerThanHundred`。

同样，我们可以在映射中存储函数：

```go
func main() {
	ints := []int{1, 2, 3, 6, 101}
	dispatcher := map[string]predicate{
		"2": largerThanTwo,
		"5": largerThanFive,
	}
	fmt.Printf("%v\n", filter(ints, dispatcher["2"]))
}
```

在这个例子中，我们创建了一个存储谓词并关联谓词函数与字符串作为键的映射。然后我们可以调用 `filter` 函数并要求映射返回与 `"2"` 键关联的函数。这返回以下内容：

```go
[3 6 101]
```

这种模式非常强大，我们将在本章后面的 *示例 1* 中探讨。

在我们深入那个例子之前，让我们看看如何在结构体内部存储函数。

## 结构体内部的函数

到现在为止，我们可以在任何可以使用数据类型的地方使用函数，让我们看看这在结构体中是如何体现的。让我们创建一个名为 `ConstraintChecker` 的结构体，该结构体用于检查一个值是否介于两个值之间。

让我们从定义我们的结构体开始。`ConstraintChecker` 结构体有两个字段。每个字段都是类型为 `predicate` 的函数。第一个函数是 `largerThan`，第二个是 `smallerThan`。这些是输入数字应该位于其间的边界：

```go
type ConstraintChecker struct {
	largerThan  predicate
	smallerThan predicate
}
```

接下来，我们为这个结构体创建一个方法。`check` 方法接受一个整数输入，并将其分别传递给 `largerThan` 和 `smallerThan` 函数。由于这两个谓词函数都返回一个布尔值，我们只需检查输入在这两个函数中返回的值是否为真：

```go
func (c ConstraintChecker) check(input int) bool {
	return c.largerThan(input) && c.smallerThan(input)
}
```

现在我们已经创建了结构体和我们的方法，让我们看看我们如何使用这个结构体：

```go
func main() {
	checker := ConstraintChecker{
		largerThan:  createLargerThanPredicate(2),
		smallerThan: func(i int) bool { return i < 10 },
	}
	fmt.Printf("%v\n", checker.check(5))
}
```

在我们的主函数中，我们首先实例化函数。请注意，我们可以通过提供现有函数（如我们为`largerThan`所做的那样）以及使用匿名函数（如`smallerThan`字段的情况）来创建`ConstraintChecker`结构体。

这展示了结构体可以存储函数，以及这些函数如何被当作结构体中的任何其他字段来对待。本质上，我们可以将绑定到结构体的每个函数视为结构体的一个**字段**函数。将函数作为字段传递与绑定相比有一些优势，我们将在本章的*示例 2*中更详细地探讨。

主要区别在于，绑定到函数的函数本质上是不变的——实现不会改变。而传递给字段的函数则完全灵活。实际的实现对我们这个结构体来说是未知的。我们将在*示例 2*中更详细地探讨这是如何允许我们为测试模拟函数的。

# 示例 1 – 映射调度器

这些一等函数类型使得“映射调度器模式”成为可能。这是一种模式，其中我们使用“键到函数”的映射。

## 创建一个简单的计算器

对于这个第一个例子，让我们构建一个真正简单的计算器。这只是为了演示基于特定输入值调度函数的想法。在这种情况下，我们将构建一个计算器，它接受两个整数作为输入，一个操作，并将这个操作的结果返回给用户。对于这个第一个例子，我们只支持加法、减法、乘法和除法操作。

首先，让我们定义支持的基本函数：

```go
func add(a, b int) int {
	return a + b
}
func sub(a, b int) int {
	return a - b
}
func mult(a, b int) int {
	return a + b
}
func div(a, b int) int {
	if b == 0 {
		panic("divide by zero")
	}
	return a / b
}
```

到目前为止，这些都是相当标准的东西。我们的计算器支持一些函数。在大多数情况下，结果会立即返回，但对于除法函数，我们会快速检查以确保我们不是除以零，否则会恐慌。在实际应用中，我们会尽量避免使用`panic`操作，但在这个例子中，它实际上没有任何影响。在这个例子中，没有用户因为恐慌而受到伤害！

接下来，让我们看看如何实现`calculate`函数，它接受两个数字和所需的操作。我们将首先不考虑函数作为一等公民，而是使用`switch`语句来决定调度哪个操作：

```go
func calculate(a, b int, operation string) int {
	switch operation {
	case "+":
		return add(a, b)
	case "-":
		return sub(a, b)
	case "*":
		return mult(a, b)
	case "/":
		return div(a, b)
	default:
		panic("operation not supported")
	}
}
```

`switch`语句的每个分支都会对我们的数字执行所需的操作并返回结果。如果选项耗尽且没有匹配输入，我们会恐慌。每次我们向计算器添加一个新函数时，我们都需要通过另一个分支扩展这个函数。随着时间的推移，这可能不是最可读的选项。所以，让我们看看本章到目前为止学到的替代方案。

首先，让我们介绍这类函数的类型：

```go
type calculateFunc func(int, int) int
```

接下来，让我们创建一个映射，我们可以将用户的字符串输入绑定到计算器函数：

```go
var (
	operations = map[string]calculateFunc{
		"+": add,
		"-": sub,
		"*": mult,
		"/": div,
	}
)
```

这个映射被称为 `operations`。映射的键是用户将提供的输入，即我们在计算器中支持的运算。我们将每个输入绑定到特定的函数调用。

现在，如果我们想实现实际的 `calculate` 函数，我们只需在我们的映射中查找键并调用相应的函数。如果请求的操作与我们的映射中的键不匹配，我们会陷入恐慌。这与基于 switch 的方法的默认分支类似：

```go
func calculateWithMap(a, b int, opString string) int {
	if operation, ok := operations[opString]; ok {
		return operation(a, b)
	}
	panic("operation not supported")
}
```

这样，我们可以用映射调度器替换 `Switch` 语句。同时记住，映射查找通常是常数时间完成的，所以这种函数调度器的实现相当高效。它确实需要我们使用更多内存来绑定键到函数，但这可以忽略不计。使用这种方法，添加新操作只需在我们的映射中添加一个新条目，而不是扩展 `switch` 语句。

使用匿名函数，我们还可以在行内定义调度函数。例如，这是我们将如何扩展映射以包含位移函数的方式：

```go
var (
	operations = map[string]calculateFunc{
		"+": add,
		"-": sub,
		"*": mult,
		"/": div,
		"<<": func(a, b int) int { return a << b },
		">>": func(a, b int) int { return a >> b },
	 }
)
```

以这种方式，我们可以为匿名函数创建一个映射调度器。但这可能会变得难以阅读，所以在应用时请使用最佳判断。

# 示例 2 – 测试时模拟函数

在以下示例中，我们将通过使用本章学到的内容来模拟函数。我们将构建和测试的应用程序是一个简单的待办事项应用程序。这个待办事项应用程序简单地允许用户向待办事项添加文本，或覆盖所有内容。

我们不会使用实际的数据库，所以我们将想象这个数据库存在，并使用文件系统和程序参数。我们的目标将是创建测试此应用程序，其中我们可以模拟数据库交互。为了实现这一点，我们将使用函数作为一等公民和类型别名以提高代码可读性。

完整的示例可以在 GitHub 上找到：[`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter2/Examples/TestingExample`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter2/Examples/TestingExample)

让我们从设置我们的主要结构体开始。我们需要两个结构体：`Todo` 和 `Db`。`Todo` 结构体表示待办事项，它将包含一段文本。该结构体还包含对 `Db` 结构体的引用：

```go
type Todo struct {
	Text string
	Db   *Db
}
func NewTodo() Todo {
	return Todo{
		Text: "",
		Db:   NewDB(),
	}
}
```

在这个例子中，我们还创建了一个“构造函数”，以确保用户得到一个正确初始化的对象。

我们将为这个结构体添加两个绑定函数：`Write` 和 `Append`。`Write` 函数将覆盖 `Text` 字段的内容，而 `Append` 函数将内容添加到现有字段的现有内容中。让我们还假设对这些函数的调用只能由授权用户进行。因此，我们首先进行数据库调用，以确定用户是否有权执行此操作：

```go
func (t *Todo) Write(s string){
	if t.Db.IsAuthorized() {
		t.Text = s
	} else {
		panic("user not authorized to write")
	}
}
func (t *Todo) Append(s string) {
	if t.Db.IsAuthorized() {
		t.Text += s
	} else {
		panic("user not authorized to append")
	}
}
```

在此基础上，让我们看看模拟数据库。因为我们希望在稍后编写的测试中能够模拟数据库的函数，我们将利用一等函数的概念。首先，我们将创建一个`Db`结构体。因为我们只是在假装连接到一个真实的数据库，所以我们不会麻烦地设置连接并让一个实际的数据库在某处运行：

```go
type authorizationFunc func() bool
type Db struct {
	AuthorizationFn authorizationFunc
}
```

这是`Db`的结构体定义。记住，函数可以作为结构体的字段存储。这正是这里发生的事情，我们的`Db`结构体包含一个名为`AuthorizationFn`的单个字段。这是一个指向类型`authorizationFunc`的函数的引用。记住，这只是一个类型别名。编译器实际上会期望一个具有`func() bool`签名的函数。因此，我们期望一个不接受任何输入参数并返回 bool 值的函数。

现在，让我们创建这样一个授权函数。由于这个示例是自包含的，我们不对使用实际数据库的开销感兴趣。对于这个例子，假设如果程序参数包含作为程序第一个参数的`admin`字符串，则用户被授权：

```go
func argsAuthorization() bool {
	user := os.Args[1]
	// super secure authorization layer
	// in a real application, this would be a database call
	if user == "admin" {
		return true
	}
	return false
}
```

注意这个函数与类型`authorizationFunc`的函数签名相匹配。因此，它可以存储在我们的`Db`结构体的`authorizationFn`字段中。接下来，让我们为我们的`Db`创建一个构造函数类型函数，这样我们就可以为用户提供一个正确初始化的结构体：

```go
func NewDB() *Db {
	return &Db{
		AuthorizationFn: argsAuthorization,
	}
}
```

注意我们是如何将`argsAuthorization`函数传递给`AuthorizationFn`字段的。因此，每次我们创建数据库时，我们都可以更改`AuthorizationFn`的实现以匹配我们的用例。我们将利用这一点来进行单元测试，但你也可以利用这一点来提供不同的授权实现，从而提高我们结构体的可重用性。

在这里引入一个方便的结构是创建一个绑定到`Db`对象的函数，该函数将调用内部授权函数：

```go
func (d *Db) IsAuthorized() bool {
	return d.AuthorizationFn()
}
```

这是一个简单的质量改进。这样，我们可以在`IsAuthorized`中添加代码，无论选择哪种实现方式，它都会运行。我们可以在那里添加日志进行调试、收集指标、处理潜在的异常等等。在我们的情况下，我们将保持它为一个简单的对`AuthorizationFn`的函数调用。

在此基础上，让我们现在考虑测试我们的代码。如果不模拟`IsAuthorized`函数，我们的测试将无法通过`Write`和`Append`测试，因为只有授权用户才能调用这些函数。我们的测试运行不应该依赖于“外部世界”来成功。单元测试应该在隔离状态下运行，而不关心真实的底层系统（在这种情况下，程序参数，但在实际场景中，实际的数据库）。

那么，我们如何解决这个问题呢？我们将通过创建一个包含我们自己的`AuthorizationFn`的`Db`结构体来模拟`authorizationFn`的实现：

```go
func TestTodoWrite(t *testing.T) {
	todo := pkg.Todo{
		Db: &pkg.Db{
			AuthorizationF: func() bool { return true },
		},
	}
	todo.Write("hello")
	if todo.Text != "hello" {
		t.Errorf("Expected 'hello' but got %v\n", todo.Text)
	}
	todo.Append(" world")
	if todo.Text != "hello world" {
		t.Errorf("Expected 'hello world' but got %v\n", 
          todo.Text)
	}
}
```

注意在这个测试的设置中，我们手动构建了一个`Todo`结构体，而不是调用构造函数类型的`newTodo()`函数。我们还在手动构建`Db`。这是为了避免在单元测试中运行默认实现。我们不是使用代码中找到的现有函数，而是提供了一个自定义的授权函数。我们的自定义函数简单地对每次调用`IsAuthorized`返回 true。这是我们测试用例中期望的行为，因为我们想测试`Todo`结构体的功能，而不是`Db`的功能。使用这种模式，我们可以模拟实现的核心部分。我们还得到了额外的好处，即我们的结构体本身变得更加灵活，因为实现现在可以在运行时进行替换。

# 摘要

在本章中，我们探讨了作为 Go 开发者，一等函数是什么以及它们为我们打开了哪些类型的用例。我们探讨了函数与对象之间的等价性，例如它们如何被实例化、作为参数传递、存储在其他数据结构中，以及从其他函数返回。

我们还学习了如何使用类型别名来创建更易读的代码，并提供更清晰的错误消息。我们看到了这些如何应用于函数以及结构体和原始数据类型的常规数据类型。

在示例中，我们看到了如何创建一个可读的函数分发器，以及如何利用一等函数创建函数的模拟。在下一章中，我们将使用本章学到的知识来构建高阶函数。
