

# 函数的三个常见类别

在前面的章节中，我们已经探讨了函数式编程的一些核心组件。我们讨论了如何编写既符合函数式编程又符合纯函数式编程的函数。

在本章中，我们将探讨一些利用这些概念的实际函数实现。我们将涵盖以下类别和主题：

+   我们将要探讨的第一个类别是基于谓词的函数

+   然后，我们将查看数据转换函数，这些函数保持我们数据结构（更多内容将在后面介绍）

+   最后，我们将探讨函数，这些函数将数据转换并减少信息到一个单一值

这并不是一个详尽的列表，但有了这三个类别，我们可以构建我们日常应用的大部分内容。

# 技术要求

对于本章，你可以使用任何 Go 1.18 或更高版本的 Go，因为我们将在一些后续示例中使用泛型。你可以在 GitHub 上找到所有代码，链接为 [`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter6`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter6)。

# 基于谓词的函数

我们将要探索的第一种函数类型是基于谓词的函数。函数体内的 `if` 语句。一个常见的用例是将一组数据过滤成符合特定条件的子集 - 例如，给定一个人员列表，返回所有年龄大于 18 岁的人。

首先，我们可以引入一个函数类型别名，它定义了谓词的类型签名：

```go
type Predicate[A any] func(A) bool
```

这个类型别名告诉我们，该函数接受一个类型为 `A` 的输入，它可以代表程序中的 `any` 类型，但需要返回一个 `bool` 值。这个类型使用了泛型，它是在 Go 1.18 中引入的。我们现在可以在任何期望谓词的地方使用这个类型。第一个使用谓词工作的函数是简单的 `Filter` 函数。

## 实现一个 Filter 函数

`Filter` 函数是函数式程序员工具箱中的基本工具。让我们假设我们没有可用的高阶函数，并且我们想要编写一个类似 `Filter` 的函数。为此，让我们假设我们有一组数字，并且我们想要过滤出所有大于 10 的数字。我们可以编写如下内容：

```go
func Filter(numbers []int) []int {
	out := []int{}
	for _, num := range numbers {
		if num > 10 {
			out = append(out, num)
		}
	}
	return out
}
```

这已经足够好了，但它不够灵活。在这种情况下，这个函数将始终只过滤出大于 10 的数字。我们可以通过调整函数的输入参数中的阈值值来使其更加灵活。通过微小的改动，我们得到以下函数：

```go
func Filter(numbers []int, threshold int) []int {
	out := []int{}
	for _, num := range numbers {
		if num > threshold {
			out = append(out, num)
		}
	}
	return out
}
```

这使我们有了更灵活的`Filter`函数。然而，正如我们所知，需求经常变化，用户几乎无限期地需要现有系统的新功能。我们函数的下一个需求是可选地过滤出“大于”或在某些情况下“小于”。思考一段时间后，你可能会意识到这可以作为一个函数实现（函数体在代码片段中被省略，因为它是一个微不足道的更改）：

```go
func FilterLargerThan(numbers []int, threshold int) []int {
..
}
func FilterSmallerThan(numbers []int, threshold int) []int {
..
}
```

当然，这会起作用——但工作永远不会停止。接下来，你必须实现一个可以过滤出大于给定值但小于另一个值的数字的函数。然后，我们的用户对奇数特别感兴趣，因此需要有一个可以找到所有奇数的过滤器。后来，用户要求你计算某个值出现的确切次数，因此你还需要一个过滤器来确保你的数字列表中恰好有一个特定的值。你明白我的意思了；我们可以创建一系列适合所有这些用例的函数，但这种方法听起来并不是最佳选择。

拥有一个支持高阶函数的语言的好处之一是我们可以减少重复的实现并抽象我们的算法。所有上述用例都适合在函数式编程语言中经常被称为`Filter`的函数。`Filter`函数的实现相当直接。它支持的基本操作是遍历一个容器，如切片，并对容器中包含的每个数据元素应用谓词函数。如果谓词函数返回`true`，我们将此数据元素追加到我们的输出中。如果不匹配，我们简单地丢弃不匹配的元素。

由于我们希望遵循实现这些函数的最佳实践，因此这些函数将是纯函数且不可变的。在我们的过滤器函数内部，原始切片永远不会被修改，其中的元素也不会被修改：

```go
func FilterA any []A {
	output := []A{}
	for _, element := range input {
		if pred(element) {
			output = append(output, element)
		}
	}
	return output
}
```

这个`Filter`实现是一个相当典型的实现，你会在许多函数式（和多范式）编程语言中找到。通过这种方式使用高阶函数，我们实际上可以使算法的一部分可配置。换句话说，我们抽象了我们的算法。使用`Filter`函数，`if`语句的实际谓词部分是可定制的。

注意，我们使用的是泛型。`Filter`不关心它正在处理的数据类型。任何可以存储在切片中的东西都可以传递给`Filter`函数。让我们通过创建我们之前讨论的一些函数来查看我们如何在实践中使用它。我们将从实现`LargerThan`和`SmallerThan`过滤器开始：

```go
func main() {
	input := []int{1, 1, 3, 5, 8, 13, 21, 34, 55}
	larger20 :=
          Filter(input, func(i int) bool { return i > 20 })
	smaller20 :=
          Filter(input, func(i int) bool { return i < 20 })
	fmt.Printf("%v\n%v\n", larger20, smaller20)
}
```

我们传递给`Filter`作为输入的函数有点冗长，因为在编写本文时，Go 还没有创建匿名函数的语法糖。注意，我们不需要为这个实现重复`Filter`函数的主体。

实现其他过滤器，如`大于 X 但小于 Y`或`过滤偶数`，同样容易实现。记住，我们每次只需要传递`if`语句的逻辑，列表的迭代由`Filter`函数本身处理：

```go
func main() {
	input := []int{1, 1, 3, 5, 8, 13, 21, 34, 55}
	larger10smaller20 := Filter(input, func(i int) bool {
		return i > 10 && i < 20
	})
	evenNumbers := Filter(input, func(i int) bool {
		return i%2 == 0
	})
	fmt.Printf("%v\n%v\n", larger10smaller20, evenNumbers)
}
```

通过使用泛型实现，我们的`Filter`函数可以与任何数据类型一起工作。让我们看看这个函数如何与我们在前面章节中使用过的`Dog`结构体一起工作。

记住，我们的`Dog`结构体有三个字段：`Name`、`Breed`和`Gender`：

```go
type Dog struct {
	Name   Name
	Breed  Breed
	Gender Gender
}
```

此代码片段省略了`Breed`和`Gender`的`const`声明以及类型别名。这些与*第三章*中的相同，完整的实现可以在 GitHub 上找到：[`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter3`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter3)。

由于我们在`Filter`函数的实现中使用了泛型，这将适用于任何数据类型，包括自定义结构体。因此，我们可以直接使用该函数，无需任何修改。让我们实现一个对所有哈瓦那犬品种的狗进行过滤的过滤器：

```go
func main() {
	dogs := []Dog{
		Dog{"Bucky", Havanese, Male},
		Dog{"Tipsy", Poodle, Female},
	}
	result := Filter(dogs, func(d Dog) bool {
		return d.Breed == Havanese
	})
	fmt.Printf("%v\n", result)
}
```

这就是全部内容。接下来，让我们看看一些使用谓词的其他函数。

## 任何或所有

确保某些元素或所有元素符合特定条件是很常见的。将这种需求抽象成高阶函数的使用场景与`Filter`函数相同。如果我们不进行抽象，就必须为每个使用场景实现单独的`All`和`Any`函数。虽然这些在多范式语言或面向对象语言中并不常见，但在纯函数式语言中仍然存在，并且非常有用。

### 寻找匹配项

首先要查看的函数是`Any`函数。有时，你可能想知道某个值是否存在于列表中，而不关心它出现的次数或实际上使用这些值。如果是这种情况，`Any`函数正是你所需要的。

没有使用`Any`函数，同样的结果可以通过`Filter`函数以某种临时方式实现。你可能会写出如下内容：

```go
func main() {
	input := []int{1, 1, 3, 5, 8, 13, 21, 34, 55}
	filtered := Filter(input, func(i int) bool { return i == 
        55 })
	contains55 := len(filtered) > 0
	fmt.Printf("%v\n", contains55)
}
```

请注意，我将其分成多行是为了清晰起见，但在像 Python 和 Haskell 这样的非冗长语言中，这种过滤器仍然是一个很好的单行代码。在 Go 中，如果你决定这样做，我会对行长度稍微谨慎一些。

这种实现有一个主要缺陷。如果你有一个包含 1000 万个元素的非常大的列表怎么办？`Filter`函数将遍历列表中的每个元素。它始终以线性时间，`O(n)`运行。我们的`Any`函数可以做得更好，尽管我们仍然会以`O(n)` – 最坏情况时间运行。然而，在实践中，它可能更高效。

注意

如果我们知道我们只需要查找整数，那么比我们的 `Any` 实现更好的算法。然而，我们希望为任何类型的数据编写它，所以那些其他算法对于字符串或自定义结构体等数据类型将失败。

尽管理论上的最坏情况复杂度为线性时间，但获取一些性能的最简单方法是通过遍历切片直到第一个元素匹配我们的搜索。如果找到匹配项，我们返回 `true`。否则，我们在函数结束时返回 `false`：

```go
func AnyA any bool {
	for _, element := range input {
		if pred(element) {
			return true
		}
	}
	return false
}
```

### 寻找所有匹配项

`All` 匹配的实现与 `Any` 匹配类似，具有相同的抽象 `if` 语句实现的优点。`All` 的实现具有与 `Any` 实现类似的实际优点。一旦一个元素返回 `false`。否则，我们在函数结束时返回 `true`：

```go
func AllA any bool {
	for _, element := range input {
		if !pred(element) {
			return false
		}
	}
	return true
}
```

## 实现 DropWhile 和 TakeWhile

下面的两个实现仍然是基于谓词的，但它们不是返回单个 `true` 或 `false` 作为输出，而是用来操作切片。从这个意义上说，它们更接近原始的 `Filter` 实现，但不同之处在于它们截断列表的开始或尾部。

### TakeWhile 实现

`TakeWhile` 是一个函数，只要满足条件，就会从输入切片中取元素。一旦条件失败，就会返回包含列表开始直到失败谓词的结果：

```go
func TakeWhileA any []A {
	out := []A{}
	for _, element := range input {
		if pred(element) {
			out = append(out, element)
		} else {
			return out
		}
	}
	return out
}
```

在这个函数中，这正是所发生的事情。只要我们的谓词对每个后续元素都成立，这个元素就会被存储在我们的输出值中。一旦谓词失败一次，输出就会被返回。让我们用一个简单的包含连续数字的切片来演示这一点。我们的谓词将寻找奇数。因此，只要数字是奇数，它们就会被追加到输出切片中，但一旦我们遇到偶数，我们迄今为止收集到的内容就会被返回：

```go
func main() {
	ints := []int{1, 1, 2, 3, 5, 8, 13}
	result := TakeWhile(ints, func(i int) bool {
		return i%2 != 0
	})
	fmt.Printf("%v\n", result)
}
```

在这个例子中，输出结果是 `[1 1]`。注意这与普通的 `Filter` 函数不同——如果将这个相同的谓词给 `Filter` 函数，我们的输出将是 `[1 1 3 5 13]`。

实现 DropWhile

实现 `DropWhile` 是 `TakeWhile` 的对应函数。这个函数会在满足条件的情况下丢弃元素。因此，从第一个失败的谓词测试开始直到列表的末尾返回元素：

```go
func DropWhileA any []A {
	out := []A{}
	drop := true
	for _, element := range input {
		if !pred(element) {
			drop = false
		}
		if !drop {
			out = append(out, element)
		}
	}
	return out
}
```

让我们用与我们的 `TakeWhile` 函数相同的输入数据来测试这个实现：

```go
func main() {
	ints := []int{1, 1, 2, 3, 5, 8, 13}
	result := DropWhile(ints, func(i int) bool {
		return i%2 != 0
	})
	fmt.Printf("%v\n", result)
}
```

这个函数的输出结果是 `[2 3 5 8 13]`。因此，被丢弃的唯一元素是 `[1 1]`。如果你将 `TakeWhile` 和 `DropWhile` 的输出结合起来，给定相同的谓词，你将重新创建输入切片。

# Map/转换函数

我们将要探讨的下一类函数是`Map`函数。这些函数将转换函数应用于容器中的每个元素，改变元素甚至可能改变数据类型。这是函数式程序员工具箱中最强大的函数之一，因为它允许你根据给定的规则转换你的数据。

我们将探讨两种主要的实现。第一种实现是简单的`Map`函数，其中对每个元素执行操作，但转换前后数据类型保持不变——例如，乘以切片中的每个元素。这将改变值的内 容，但不会改变值的类型。`Map`的另一种实现是数据类型也可以改变。这将被实现为`FMap`，这是我们上一章在探讨 Monads 时引入的。

## 保持数据类型不变的转换

我们将要探讨的第一个转换函数是数据类型保持不变的那种。每当程序员遇到这个函数时，他们可以确信函数调用后的数据类型与传递给函数的数据类型相同。换句话说，如果函数被用于一个包含`Dog`类型元素的列表，那么这个函数的输出仍然是一个包含`Dog`元素的列表。不过，这些结构体（structs）字段的实际内容可能会有所不同（例如，名称属性可能会被更新）。

就像`Filter`实现一样，这些将纯函数式地实现。调用`Map`函数**永远**不应该对我们提供给函数作为输入的对象进行就地更改。

总体来说，实现`Map`函数很简单。我们将遍历我们的值切片并对每个值调用转换函数。本质上，我们使用`Map`函数所做的就是抽象实际的转换逻辑。核心算法是我们对切片的迭代，而不是具体的转换。这意味着我们再次构建了一个高阶函数：

```go
type MapFunc[A any] func(A) A
func MapA any []A {
	output := make([]A, len(input))
	for i, element := range input {
		output[i] = m(element)
	}
	return output
}
```

在这个例子中，我们的泛型类型签名告诉我们，在调用`MapFunc`时数据类型是保留的：

```go
type MapFunc[A any] func(A) A
```

给定`A`，我们将得到`A`。请注意，类型可以是任何类型，根据泛型合约。我们的`Map`实现不需要类型约束。让我们看看将切片中的每个元素乘以`2`的示例：

```go
func main() {
	ints := []int{1, 1, 2, 3, 5, 8, 13}
	result := Map(ints, func(i int) int {
		return i * 2
	})
	fmt.Printf("%v\n", result)
}
```

这个函数也可以与任何数据类型一起工作。让我们看看一个示例，我们将对列表中每只狗的名称进行转换。如果狗的性别是男性，我们将名称前缀设置为`Mr.`；如果性别是女性，我们将前缀设置为`Mrs.`：

```go
func dogMapDemo() {
        dogs := []Dog{
                Dog{"Bucky", Havanese, Male},
                Dog{"Tipsy", Poodle, Female},
        }
        result := Map(dogs, func(d Dog) Dog {
                if d.Gender == Male {
                        d.Name = "Mr. " + d.Name
                } else {
                        d.Name = "Mrs. " + d.Name
                }
                return d
        })
        fmt.Printf("%v\n", result)
}
```

运行此代码将产生以下输出：

```go
[{Mr. Bucky 1 0} {Mrs. Tipsy 3 1}]
```

重要的是要强调，这些更改是对数据副本进行的，而不是对原始`Dog`对象进行的。

### 从一到多的转换

`Map` 函数的一个变体是 `Flatmap` 函数。这个函数将映射一个 `Flatmap`。

我们将要使用的函数实现并不那么高效，但对于大多数目的来说足够好了。对于切片中的每个元素，我们将调用转换函数，该函数将我们的单个元素转换为一个元素切片。我们不会将这个中间结果作为切片的切片存储，而是立即将每个切片折叠并连续存储在内存中的单个元素：

```go
func FlatMapA any []A) []A {
	output := []A{}
	for _, element := range input {
		newElements := m(element)
		output = append(output, newElements…)
	}
	return output
}
```

让我们通过实现一个示例来演示这一点。对于切片中的每个整数 `N`，我们将将其转换为从 0 到 `N` 的所有整数的切片。最后，我们将这个结果作为连续切片返回：

```go
func main() {
        ints := []int{1, 2, 3}
        result := FlatMap(ints, func(n int) []int {
                out := []int{}
                for i := 0; i < n; i++ {
                        out = append(out, i)
                }
                return out
        })
        fmt.Printf("%v\n", result)
}
```

运行此代码的输出如下：

```go
[0 0 1 0 1 2]
```

这就是我们展示在图像中的内容。每个单独的元素都被转换为一个切片，然后这些切片被组合。对于输入切片中的每个元素，中间输出将如下所示：

```go
0: [0]
1: [0 1]
2: [0 1 2]
```

这个中间输出随后被组合成一个单一的切片。接下来，让我们看看在函数式编程语言中起着关键作用的函数的最后一类。

# 数据减少函数

我们将要查看的最后一组函数是 *reducer* 函数。这些函数将操作应用于元素容器，并从中导出一个单一值。结合本章前面看到的函数，我们可以组合我们的大多数应用程序。至少，就数据操作而言。在函数式编程中，这类函数有几个不同的名称。在 Haskell 中，你会找到名为 `Fold` 或 `Fold` 加后缀的函数，例如 `Foldr`，而在某些语言中它们被称为 `Reduce`。本书余下部分我们将使用 `Reduce` 术语。

我们将要查看的第一个函数就是简单的 `Reduce`。这个高阶函数将操作抽象为列表中的两个数据元素。然后它重复这个操作，累加结果，直到得到一个单一答案。就像 `Filter` 和 `Map` 函数一样，这些函数是纯函数，所以实际输入数据永远不会改变。

这个算法中的抽象函数是一个接受相同数据类型两个值的函数，并返回该数据类型的一个单一值。结果是通过对它们执行一些操作来实现的，这些操作是由函数的调用者提供的：

```go
type (
        reduceFunc[A any] func(a1, a2 A) A
)
```

这个函数最终将迭代地调用切片中的每个元素，存储中间结果，并将这些结果反馈回函数：

注意

这听起来像是递归，但在这个章节的实现中它不是递归的。我们将在下一章查看递归方法。

```go
func ReduceA any A {
	if len(input) == 0 {
		// return default zero
		return *new(A)
	}
	result := input[0]
	for _, element := range input[1:] {
		result = reducer(result, element)
	}
	return result
}
```

在这个例子中，我们也在处理我们的边缘情况。如果我们得到一个空切片，我们返回传递给我们的函数的类型的`default-nil`值。如果切片中只有一个项目，则无法执行任何操作，我们只需返回该值（通过不执行循环，因此基于`input[0]`立即返回结果）。

这些高阶函数抽象是如何将两个元素组合成一个答案的。一个可能的归约器是`sum reducer`，它将两个数字相加并返回结果。以下匿名函数是这个函数的一个示例：

```go
func(a1, a2 A) A { return a1 + a2 }
```

这是一个匿名函数，我们将将其传递给`Reduce`以执行所有元素的求和——但就目前的写法而言，这种方法有一个问题。`Reduce`函数是泛型的，可以接受`+`运算符，但并非为每个数据类型定义。为了解决这个问题，我们可以创建一个`Sum`函数，它内部调用归约器，但将类型签名收紧，只允许提供数字作为输入。

记住，由于 Go 中有多种数字数据类型，我们希望能够为所有这些使用`Sum`函数。这可以通过为我们的泛型函数创建一个自定义类型约束来实现。我们还将考虑将`Number`的类型别名视为有效——这可以通过在每个类型前添加`~`前缀来实现：

```go
type Number interface {
        ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uint |
                ~int8 | ~int16 | ~int32 | ~int64 | ~int |
                ~float32 | ~float64
}
```

接下来，我们可以将此类型用作泛型函数（如`Sum`函数）的类型约束：

```go
func SumA Number A {
	return Reduce(input, func(a1, a2 A) A { return a1 + a2 })
}
```

就这样——现在，我们可以使用这个函数来返回数字切片的求和，其中数字是 Go 中我们定义在约束中的任何当前支持的类似数字的数据类型：

```go
func main{
        ints := []int{1, 2, 3, 4}
        result := Sum(ints)
        fmt.Printf("%v\n", result)
}
```

这个函数的输出是`10`。实际上，我们的归约器执行了`1 + 2 + 3 + 4`的求和。有了归约器，我们因此可以将这些操作抽象为列表。添加一个执行每个元素乘法的类似函数与编写求和函数一样容易：

```go
func ProductA Number A {
        return Reduce(input, func(a1, a2 A) A { return a1 * a2 })
}
```

这个实现的工作方式与`Sum`函数相同。

在 Haskell 和其他函数式语言中，提供了一些不同的归约器实现，每个实现都略微改变了核心算法。你将找到以下内容：

+   从列表的开始到结束迭代的归约器

+   从列表的末尾到开始迭代的归约器

+   从列表的第一个元素而不是默认值开始的归约器

+   从具有默认值开始并从列表的末尾到开始迭代的归约器

反向归约器（从列表的末尾迭代到开始）留作读者独立探索的练习，但它们的完整代码可以在 GitHub 上找到：[`github.com/PacktPublishing/Functional-Programming-in-Go./blob/main/Chapter6/pkg/reducers.go`](https://github.com/PacktPublishing/Functional-Programming-in-Go./blob/main/Chapter6/pkg/reducers.go)。然而，我们将探讨具有起始值的归约器。

提供不同的起始值将允许我们编写一个函数，例如“将所有数字相乘，然后最终乘以二”。我们可以通过修改我们的`Reducer`函数来实现这一点：

```go
func ReduceWithStartA any A {
        if len(input) == 0 {
                return startValue
        }
        if len(input) == 1 {
                return reducer(startValue, input[0])
        }
        result := reducer(startValue, input[0])
        for _, element := range input[1:] {
                result = reducer(result, element)
        }
        return result
}
```

我们正在处理与原始`Reduce`函数类似的边缘情况，但一个关键的区别是我们始终有一个默认值返回。我们可以在切片为空时返回它，或者在切片恰好包含一个元素时返回起始值与切片中第一个元素的组合。

在下一个示例代码中，我们将使用逗号将字符串连接起来，但为了展示我们新的`ReduceWithStart`函数，我们将提供一个起始值`first`：

```go
func main() {
        words := []string{"hello", "world", "universe"}
        result := ReduceWithStart(words, "first", func(s1, s2 
            string) string {
                return s1 + ", " + s2
        })
        fmt.Printf("%v\n", result)
}
```

如果我们运行此代码，我们将得到以下输出：

```go
first, hello, world, universe
```

在这些函数到位后，让我们看看一个示例，其中我们将结合使用所有三类函数。

# 示例 – 处理机场数据

在这个例子中，我们将结合本章中的函数来分析机场数据。在我们能够使用我们创建的函数之前，我们需要做一些工作。在 GitHub 上，你可以在[`github.com/PacktPublishing/Functional-Programming-in-Go./blob/main/Chapter6/resources/airlines.json`](https://github.com/PacktPublishing/Functional-Programming-in-Go./blob/main/Chapter6/resources/airlines.json)找到`.json`提取文件。

以下片段是数据集的模板：

```go
  {
    "Airport": {
      "Code": string,
      "Name": string
    },
    "Statistics": {
      "Flights": {
        "Cancelled": number,
        "Delayed": number,
        "On Time": number,
        "Total": number
      },
      "Minutes Delayed": {
        "Carrier": number,
        "Late Aircraft": number,
        "Security": number,
        "Total": number,
        "Weather": number
      }
    }
  }
```

要处理这些数据，我们将重新创建`.json`结构作为 Go 中的结构体。我们可以使用内置的`.json`标签和反序列化器来在内存中读取这些数据。我们用于处理这些数据的 Go 结构体如下所示：

```go
type Entry struct {
	Airport struct {
		Code string `json:"Code"`
		Name string `json:"Name"`
	} `json:"Airport"`
	Statistics struct {
		Flights struct {
			Cancelled int `json:"Cancelled"`
			Delayed   int `json:"Delayed"`
			OnTime    int `json:"On Time"`
			Total     int `json:"Total"`
		} `json:"Flights"`
		MinutesDelayed struct {
			Carrier                int `json:"Carrier"`
			LateAircraft           int `json:"Late 
                                        Aircraft"`
			Security               int `json:"Security"`
			Weather                int `json:"Weather"`
		} `json:"Minutes Delayed"`
	} `json:"Statistics"`
}
```

这段文字稍微有些冗长，但它只是文件第一项内容的复制。在此之后，我们需要编写一些代码来将文件内容作为条目读入内存：

```go
func getEntries() []Entry {
        bytes, err := ioutil.ReadFile("./resources/airlines.
            json")
        if err != nil {
                panic(err)
        }
        var entries []Entry
        err = json.Unmarshal(bytes, &entries)
        if err != nil {
                panic(err)
        }
        return entries
}
```

与前几章一样，我们在代码中使用`panic`。这是不被鼓励的，但出于演示目的，这是可以的。此代码将读取我们的资源文件，根据我们创建的结构体将其解析为`json`，并作为切片返回。

现在，为了演示我们创建的函数，这是我们的问题陈述：**编写一个返回西雅图机场（机场** **代码：SEA**）总延误小时的函数。

根据这个问题陈述，我们可以看到有三个动作需要执行：

1.  通过机场代码 SEA 过滤数据。

1.  将`MinutesDelayed`字段转换为小时。

1.  汇总所有小时数。

步骤 2 和步骤 3 的顺序可以颠倒，但这样，它遵循我们在本章中介绍这些函数的结构：

```go
func main() {
	entries := getEntries()
	SEA := Filter(entries, func(e Entry) bool {
		return e.Airport.Code == "SEA"
	})
	WeatherDelayHours := FMap(SEA, func(e Entry) int {
		return e.Statistics.MinutesDelayed.Weather / 60
	})
	totalWeatherDelay := Sum(WeatherDelayHours)
	fmt.Printf("%v\n", totalWeatherDelay)
}
```

我们就这样开始了。我们使用本章中看到的三种函数实现了我们的用例。正如你所见，每次我们调用一个函数时，我们都会将结果存储在一个新的切片中。因此，原始数据永远不会丢失，如果我们选择这样做，我们仍然可以使用它来处理函数的其他部分。

# 摘要

在本章中，我们看到了三种类型的函数，这些函数将帮助我们以函数式的方式构建程序。首先，我们看到了基于谓词的函数，这些函数可以将我们的数据过滤成满足特定要求的子集，或者告诉我们数据集是否完全或部分符合某个条件。接下来，我们看到了数据如何以函数式的方式改变，以及数据类型保证保持不变的数据转换方式，以及那些也在改变类型本身的函数。

最后，我们研究了 reducer 函数，这些函数将一系列元素减少到一个单一值。我们在机场数据示例中展示了这三种类型函数的组合方式。

在下一章中，我们将深入探讨递归，看看它在函数式编程中的作用，以及编写递归函数在 Go 中的性能影响。
