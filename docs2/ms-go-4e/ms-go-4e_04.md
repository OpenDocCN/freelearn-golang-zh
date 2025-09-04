

# Go 泛型

本章将介绍泛型以及如何使用新的语法来编写泛型函数和定义泛型数据类型。泛型编程是一种编程范式，它允许开发者使用一个或多个将在以后提供的数据类型来实现函数。

Go 的泛型支持是在 Go 1.18 中引入的，该版本于 2022 年 2 月正式发布。因此，Go 的泛型现在已经不再是新闻了！尽管如此，Go 社区仍在努力理解并充分利用泛型。事实上，大多数 Go 开发者已经在没有泛型帮助的情况下完成了他们的工作。

如果你觉得在这一学习旅程的这个阶段你对这一章不感兴趣，你可以自由地跳过它，稍后再回来。然而，即使你现在可能不感兴趣，我仍然建议你阅读它。

这引出了以下事实：**如果你不想使用 Go 泛型，对泛型的有用性有疑问，或者有其他的想法，你不必使用 Go 泛型**。毕竟，你仍然可以在不使用泛型的情况下编写出色、高效、可维护和正确的软件！此外，你可以使用泛型并支持大量数据类型，如果不是所有可用的数据类型，这并不意味着你应该这样做。始终支持所需的数据类型，不多也不少，但不要忘记关注你数据未来的发展以及支持在编写代码时未知的数据类型的可能性。

本章涵盖：

+   泛型简介

+   约束

+   使用泛型定义新数据类型

+   何时使用泛型

+   `cmp`包

+   `slices`包

+   `maps`包

# 泛型简介

泛型是一种功能，它允许你不必精确指定一个或多个函数参数的数据类型，主要是因为你希望使你的函数尽可能通用。换句话说，泛型允许函数处理多种数据类型，而无需编写任何特殊代码，就像空接口和一般接口的情况一样。接口在*第五章*，*反射与接口*中有详细说明。

在 Go 中使用接口时，你必须编写额外的代码来确定你正在处理的接口变量的数据类型，而泛型则不需要这样做。

让我从展示一个小型代码示例开始，该示例实现了一个函数，清楚地展示了泛型可以派上用场并帮助你避免编写大量代码的情况：

```go
func PrintSliceT any {
    for _, v := range s {
        fmt.Println(v)
    }
} 
```

那么，这里有什么呢？有一个名为`PrintSlice()`的函数，它接受任何数据类型的切片。这通过函数签名中使用`[]T`来表示，它指定了该函数接受一个切片，并结合`[T any]`部分指定所有数据类型都被接受并支持。`[T any]`部分告诉编译器数据类型`T`在执行时不会被确定，但它在编译时仍然会根据调用代码提供的类型来确定和强制执行。我们还可以使用多个（泛型）数据类型，使用`[T, U, W any]`表示法——之后我们应该在函数签名中使用`T`、`U`和`W`数据类型。

`any`关键字告诉编译器关于`T`的数据类型没有任何约束。我们将在稍后讨论约束——现在，我们只学习泛型的语法。

现在，想象一下为整数切片、字符串切片、浮点数切片、复数值切片等实现`PrintSlice()`功能的功能分别编写单独的函数。因此，我们发现了一个深刻的案例，使用泛型简化了代码和我们的编程工作。然而，并非所有情况都如此明显，我们应该非常小心地避免过度使用`any`。

## 嗨，泛型！

以下（`hw.go`）是一段使用泛型的代码，它可以帮助你在深入了解更高级的示例之前更好地理解它们：

```go
package main
import (
    "fmt"
)
func PrintSliceT any {
    for _, v := range s {
        fmt.Print(v, " ")
    }
    fmt.Println()
} 
```

`PrintSlice()`与我们在本章前面看到的函数类似。`PrintSlice()`在同一行中打印每个切片的元素，并在`fmt.Println()`的帮助下在末尾打印一个新行。

```go
func main() {
    PrintSlice([]int{1, 2, 3})
    PrintSlice([]string{"a", "b", "c"})
    PrintSlice([]float64{1.2, -2.33, 4.55})
} 
```

在这里，我们使用相同的`PrintSlice()`函数，但传入三种不同的数据类型：`int`、`string`和`float64`。Go 编译器不会对此提出异议。相反，它将像我们分别为每种数据类型编写了三个单独的函数一样执行代码。

因此，运行`hw.go`会产生以下输出：

```go
1 2 3
a b c
1.2 -2.33 4.55 
```

因此，每个切片都使用单个泛型函数按预期打印。

基于这些信息，让我们首先讨论泛型和约束。

# 约束

假设你有一个使用泛型乘以两个数值的函数。这个函数应该与所有数据类型一起工作吗？这个函数可以与所有数据类型一起工作吗？**你能乘以两个字符串或两个结构体吗？**避免这种问题的解决方案是使用约束。*类型约束*允许你指定你想要与之一起工作的数据类型列表，以避免逻辑错误和错误。

暂时忘记乘法，考虑一些更简单的事情。假设我们想要比较变量以检查它们是否相等——有没有办法告诉 Go 我们只想与可以比较的值一起工作？Go 1.18 带来了预定义的类型约束——其中之一被称为`comparable`，包括可以比较相等或不等的数据类型。

对于更多预定义的约束，你应该查看 `constraints` 包（[`pkg.go.dev/golang.org/x/exp/constraints`](https://pkg.go.dev/golang.org/x/exp/constraints)）。

`allowed.go` 的代码展示了 `comparable` 约束的使用。

```go
package main
import (
    "fmt"
)
func SameT comparable bool {
    // Or
// return a == b
if a == b {
        return true
    }
    return false
} 
```

`Same()` 函数使用预定义的 `comparable` 约束而不是 `any`。实际上，`comparable` 约束只是一个预定义的接口，它包括所有可以用 `==` 或 `!=` 比较的数据类型。

我们不需要编写任何额外的代码来检查我们的输入，因为函数签名确保我们只会处理可接受和功能性的数据类型。

```go
func main() {
    fmt.Println("4 = 3 is", Same(4,3))
    fmt.Println("aa = aa is", Same("aa","aa"))
    fmt.Println("4.1 = 4.15 is", Same(4.1,4.15))
} 
```

`main()` 函数三次调用 `Same()`，使用不同的数据类型，并打印其结果。

运行 `allowed.go` 产生以下输出：

```go
4 = 3 is false
aa = aa is true
4.1 = 4.15 is false 
```

由于只有 `Same("aa","aa")` 是 `true`，我们得到相应的输出。

如果你尝试运行一个类似于 `Same([]int{1,2},[]int{1,3})` 的语句，该语句尝试比较两个切片，编译器将生成以下错误信息：

```go
# command-line-arguments
./allowed.go:19:10: []int does not satisfy comparable 
```

这是因为我们无法直接比较两个切片——这种功能应该手动实现。请注意，你可以比较两个数组！

下一小节将展示如何创建你自己的约束。

## 创建约束

本小节提供了一个示例，其中我们定义了可以使用接口作为泛型函数参数传递的数据类型。`numeric.go` 的代码如下：

```go
package main
import (
    "fmt"
)
type Numeric interface {
    int | int8 | int16 | int32 | int64 | float64
} 
```

在这里，我们定义了一个新的接口 `Numeric`，它指定了支持的数据类型列表。只要你可以使用你将要实现的泛型函数，你就可以使用任何你想要的数据类型。在这种情况下，如果我们想添加 `string` 或 `uint` 到支持的数据类型列表中，这是有意义的。在这种情况下，将 `string` 添加到 `Numeric` 接口中没有任何意义。

```go
func AddT Numeric T {
    return a + b
} 
```

这是定义具有两个泛型参数的泛型函数的定义，这些参数使用 `Numeric` 约束。

```go
func main() {
    fmt.Println("4 + 3 =", Add(4,3))
    fmt.Println("4.1 + 3.2 =", Add(4.1,3.2))
} 
```

之前的代码是 `main()` 函数的实现，其中调用了 `Add()`。

运行 `numeric.go` 产生以下输出：

```go
4 + 3 = 7
4.1 + 3.2 = 7.3 
```

尽管如此，Go 的规则仍然适用。因此，如果你尝试调用 `Add(4.1,3)`，你将得到以下错误信息：

```go
# command-line-arguments
./numeric.go:19:15: default type int of 3 does not match inferred type float64 for T 
```

这种错误的理由是 `Add()` 函数期望两个相同数据类型的参数。然而，`4.1` 是 `float64` 类型，而 `3` 是 `int` 类型，所以它们不是同一数据类型。

我们还没有讨论的一个额外问题是约束。正如我们已经知道的，**Go 对不同数据类型有不同的处理，即使底层数据类型相同**。这意味着如果我们创建一个基于 `int` 的新数据类型（`type aType int`），它将不会由 `Numeric` 约束支持，因为这是未指定的。下一小节将展示如何处理这种情况并克服这一限制。

## 支持底层数据类型

使用超类型，我们正在添加对底层数据类型（真实的那个）的支持，而不是当前的数据类型，这可能是现有 Go 数据类型的别名。超类型由 `~` 运算符支持。`supertypes.go` 中的 `supertypes.go` 部分展示了超类型的使用。`supertypes.go` 中的代码第一部分如下：

```go
type AnotherInt int
type AllInts interface {
    ~int
} 
```

在之前的代码中，我们定义了一个名为 `AllInts` 的约束，它使用了一个超类型（`~int`）以及一个名为 `AnotherInt` 的新数据类型，实际上它是 `int`。`AllInts` 约束的定义允许 `AnotherInt` 由 `AllInts` 支持。

`supertypes.go` 的第二部分如下：

```go
func AddElementsT AllInts T {
    sum := T(0)
    for _, v := range s {
        sum = sum + v
    }
    return sum
} 
```

在这部分中，我们定义了一个泛型函数。该函数附带了一个约束，因为它只支持 `AllInts` 的切片。

`supertypes.go` 的最后一部分如下：

```go
func main() {
    s := []AnotherInt{0, 1, 2}
    fmt.Println(AddElements(s))
} 
```

在最后一部分，我们使用 `AnotherInt` 的切片作为参数调用 `AddElements()`——这种能力是通过在 `AllInts` 约束中使用超类型提供的。

运行 `supertypes.go` 产生以下输出：

```go
$ go run supertypes.go
3 
```

因此，**在类型约束中使用超类型允许 Go 处理实际的底层数据类型**。

## 支持任何类型的切片

在本小节中，我们将指定函数参数只能为任何数据类型的切片。`sliceConstraint.go` 中的相关代码如下：

```go
func f1[S interface{ ~[]E }, E interface{}](x S) int {
    return len(x)
}
func f2[S ~[]E, E interface{}](x S) int {
    return len(x)
}
func f3[**S** **~[]****E****,** **E****any**](x S) int {
    return len(x)
} 
```

所有三个泛型函数是等效的。使用 `~[]E` 指定底层数据类型应该是切片，即使它是由不同名称的类型。

`f1()` 函数是函数签名的长版本。`interface{ ~[]E }` 指定我们只想与任何数据类型的切片（`E interface{}`）一起工作。`f2()` 函数将 `interface{ ~[]E }` 替换为仅 `~[]E`，因为 Go 允许你在约束位置省略 `interface{}`。最后，`f3()` 函数将常用的 `interface{}` 替换为其预定义的等效项 `any`，这是我们之前已经看到过其作用的。我发现 `f3()` 的实现更简单，更容易理解。

下一节展示了在定义新数据类型时如何使用泛型。

# 使用泛型定义新数据类型

在本节中，我们将使用泛型创建一个新的数据类型，这在 `newDT.go` 中展示。`newDT.go` 的代码如下：

```go
package main
import (
    "fmt"
"errors"
)
type TreeLast[T any] []T 
```

之前的语句声明了一个名为 `TreeLast` 的新数据类型，它使用了泛型。

```go
func (t TreeLast[T]) replaceLast(element T) (TreeLast[T], error) {
    if len(t) == 0 {
        return t, errors.New("This is empty!")
    }

    t[len(t) - 1] = element
    return t, nil
} 
```

`replaceLast()` 是一个操作 `TreeLast` 变量的方法。除了函数签名外，没有其他内容显示泛型的使用。

```go
func main() {
    tempStr := TreeLast[string]{"aa", "bb"}
    fmt.Println(tempStr)
    tempStr.replaceLast("cc")
    fmt.Println(tempStr) 
```

在 `main()` 的第一部分中，我们使用 `aa` 和 `bb` 字符串值创建了一个 `TreeLast` 变量，然后我们通过调用 `replaceLast("cc")` 将 `bb` 值替换为 `cc`。

```go
 tempInt := TreeLast[int]{12, -3}
    fmt.Println(tempInt)
    tempInt.replaceLast(0)
    fmt.Println(tempInt)
} 
```

`main()` 的第二部分使用 `TreeLast` 变量（用 `int` 值填充）执行与第一部分类似的操作。因此，`TreeLast` 可以与 `string` 和 `int` 值一起工作而不会出现任何问题。

运行 `newDT.go` 产生以下输出：

```go
[aa bb]
[aa cc]
[12 -3]
[12 0] 
```

输出的前两行与 `TreeLast[string]` 变量相关，而输出的最后两行与 `TreeLast[int]` 变量相关。

下一个子部分是关于在 Go 结构中使用泛型。

## 在 Go 结构中使用泛型

在本节中，我们将实现一个使用泛型的链表——这是泛型使用简化事情的一个例子，因为它允许你一次性实现链表，同时能够处理多种数据类型。

`structures.go` 的代码如下：

```go
package main
import (
    "fmt"
)
type node[T any] struct {
    Data T
    next *node[T]
} 
```

节点结构使用泛型来支持可以存储所有类型数据的节点。这并不意味着节点的下一个字段可以指向另一个具有不同数据类型 `Data` 字段的节点。链表包含相同数据类型元素的规则仍然适用——这仅仅意味着，如果你想创建三个链表，一个用于存储 `string` 值，一个用于存储 `int` 值，第三个用于存储给定 `struct` 数据类型的 JSON 记录，你不需要为此编写任何额外的代码。

```go
type list[T any] struct {
    start *node[T]
} 
```

这是 `node` 节点构成的链表的根节点定义。`list` 和 `node` 必须共享相同的数据类型 `T`。然而，正如之前所述，这并不阻止你创建多个不同数据类型的链表。

如果你想要限制允许的数据类型列表，你仍然可以在 `node` 和 `list` 的定义中将 `any` 替换为约束。

```go
func (l *list[T]) add(data T) {
    n := node[T]{
        Data: data,
        next: nil,
    } 
```

`add()` 函数是泛型的，以便能够与所有类型的节点一起工作。除了 `add()` 的签名外，其余代码与泛型的使用无关。

```go
 if l.start == nil {
        l.start = &n
        return
    }

    if l.start.next == nil {
        l.start.next = &n
        return
    } 
```

这两个 `if` 块与向链表中添加新节点有关。第一个 `if` 块是当列表为空时的情况，而第二个 `if` 块是当我们处理当前列表的最后一个节点时的情况。

```go
 temp := l.start
    l.start = l.start.next
    l.add(data)
    l.start = temp
} 
```

`add()` 的最后一部分与在列表中添加新节点时定义节点之间的适当关联有关。

```go
func main() {
    var myList list[int] 
```

首先，我们在 `main()` 中定义一个整数值的链表，这是我们将要处理的链表。

```go
 fmt.Println(myList) 
```

`myList` 的初始值是 `nil`，因为列表为空且不包含任何节点。

```go
 myList.add(12)
    myList.add(9)
    myList.add(3)
    myList.add(9) 
```

在本部分，我们向链表中添加了四个元素。

```go
 // Print all elements
    cur := myList.start
    for {
        fmt.Println("*", cur)
        if cur == nil {
            break
        }
        cur= cur.next
    }
} 
```

`main()` 的最后一部分是关于通过使用指向列表中下一个节点的 `next` 字段来遍历列表并打印所有元素。

运行 `structures.go` 产生以下输出：

```go
{<nil>}
* &{12 0x14000096240}
* &{9 0x14000096260}
* &{3 0x14000096290}
* &{9 <nil>}
* <nil> 
```

让我们更详细地讨论一下输出。第一行显示空列表的值为`nil`。列表的第一个节点包含一个值为`12`和内存地址（`0x14000096240`），该地址指向第二个节点。这个过程一直持续到我们达到最后一个节点，它包含值为`9`的值，在这个链表中出现了两次，并指向`nil`，因为它是最后的节点。因此，泛型使得链表能够与多种数据类型一起工作。

接下来的三个部分介绍了三个使用泛型的包——您可以自由地查看它们的实现细节（见*附加资源*部分）。

# The cmp package

`cmp`包在 Go 1.21 中成为标准 Go 库的一部分，包含用于比较有序值的类型和函数。之所以在`slices`和`maps`包之前介绍它，是因为它被其他两个包使用。请记住，在其当前版本中，`cmp`包很简单，但它可能会在未来通过更多功能得到丰富。

在底层，`cmp`、`slices`和`maps`包使用泛型和约束，这是在本章中介绍它们的主要原因。因此，泛型可以用来创建可以与多种数据类型一起工作的包。

`cmpPackage.go`中的重要代码可以在`main()`函数中找到。

```go
func main() {
    fmt.Println(cmp.Compare(5, 4))
    fmt.Println(cmp.Compare(4, 5))
    fmt.Println(cmp.Less(4, 5.1))
} 
```

在这里，`cmp.Compare(x, y)`比较两个值，当`x < y`时返回`-1`，当`x=y`时返回`0`，当`x > y`时返回`1`。`cmp.Compare(x, y)`返回一个`int`值。另一方面，`cmp.Less(x, y)`返回一个`bool`值，当`x < y`时设置为`true`，否则为`false`。

注意，在最后一个语句中，我们正在比较一个整数值和一个浮点值。然而，`cmp`包足够聪明，可以将`int`值转换为`float64`值，并比较这两个值！

运行`cmpPackage.go`产生以下输出：

```go
$ go run cmpPackage.go
1
-1
true 
```

`cmp.Compare(5, 4)`的输出是`1`，`cmp.Compare(4, 5)`的输出是`-1`，而`cmp.Less(4, 5)`的输出是`true`。

# The slices package

`slices`包自 Go 1.21 以来一直是标准 Go 库的一部分，并为任何数据类型的切片提供了函数。在我们继续讨论`slices`包之前，让我们谈谈*浅拷贝*和*深拷贝*功能，包括它们之间的区别。

## 浅拷贝和深拷贝

一个*浅拷贝*会创建一个新的变量，然后将其赋值为原始变量版本中找到的所有值。如果我们谈论的是映射，那么这个过程会使用普通赋值来分配所有键和值。

*深度复制* 首先创建一个新的变量，然后插入原始变量中找到的所有值。然而，每个值都必须递归地复制——如果我们谈论的是字符串，这可能不是问题，但如果我们谈论的是结构体、结构体的引用或指针，这可能会成为一个问题。在这个过程中，可能会创建无限循环。关键词在这里是 *递归*——这意味着我们需要遍历所有值（如果我们谈论的是切片或映射）或字段（如果我们谈论的是结构体），并找出需要递归复制的部分。

因此，浅复制和深度复制之间的主要区别在于，在深度复制中，**实际值是递归复制的**，而在浅复制中，**我们使用普通赋值来分配原始值**。

我们现在可以继续介绍 `slices` 包的功能。`slicesPackage.go` 中 `main()` 实现的第一部分如下：

```go
func main() {
    s1 := []int{1, 2, -1, -2}
    s2 := slices.Clone(s1)
    s3 := slices.Clone(s1[2:])
    fmt.Println(s1[2], s2[2], s3[0])
    s1[2] = 0
    s1[3] = 0
    fmt.Println(s1[2], s2[2], s3[0]) 
```

`slices.Clone()` 函数返回给定切片的浅复制——元素通过赋值复制。在执行 `s2 := slices.Clone(s1)` 调用后，`s1` 和 `s2` 相等，但它们的元素具有各自的内存空间。

第二部分如下：

```go
 s1 = slices.Compact(s1)
    fmt.Println("s1 (compact):", s1)
    fmt.Println(slices.Contains(s1, 2), slices.Contains(s1, -2))
    s4 := make([]int, 10, 100)
    fmt.Println("Len:", len(s4), "Cap:", cap(s4))
    s4 = slices.Clip(s4)
    fmt.Println("Len:", len(s4), "Cap:", cap(s4)) 
```

`slices.Compact()` 函数将连续出现的相等元素替换为单个副本。因此，`-1 -1 -1` 将变成 `-1`，而 `-1 0 -1` 则不会改变。一般来说，`slices.Compact()` 在排序切片上工作得最好。

`slices.Contains()` 函数报告给定值是否存在于切片中。

`slices.Clip()` 函数从切片中移除未使用的容量。简单来说，切片的容量将等于切片的长度。当容量远大于切片长度时，这可以为您节省大量内存。

最后的部分包含以下代码：

```go
 fmt.Println("Min", slices.Min(s1), "Max:", slices.Max(s1))
    // Replace s2[1] and s2[2]
    s2 = slices.Replace(s2, 1, 3, 100, 200)
    fmt.Println("s2 (replaced):", s2)
    slices.Sort(s2)
    fmt.Println("s2 (sorted):", s2)
} 
```

`slices.Min()` 和 `slices.Max()` 函数分别返回切片中的最小值和最大值。

`slices.Replace()` 函数用提供的值替换给定范围内的元素，在这个例子中是 `s2[1:3]`，这些值在这个例子中是 `100` 和 `200`，并返回修改后的切片。最后，`slices.Sort()` 以升序对任何有序类型的值进行排序。

运行 `slicesPackage.go` 产生以下输出：

```go
$ go run slicesPackage.go
-1 -1 -1
0 -1 -1
s1 (compact): [1 2 0]
true false
Len: 10 Cap: 100
Len: 10 Cap: 10
Min: 0 Max: 2
s2 (replaced): [1 100 200 -2]
s2 (sorted): [-2 1 100 200] 
```

您可以在切片的容量中看到 `slices.Clip()` 的效果，以及在 `s2` 切片的值中看到 `slices.Replace()` 的效果。

下一个部分将介绍 `maps` 包。

# `maps` 包

`maps` 包自 Go 1.21 版本以来一直是标准 Go 库的一部分，并为任何类型的映射提供了函数——其用法在 `mapsPackage.go` 中得到了说明。

`mapsPackage.go` 程序使用了两个辅助函数，定义如下：

```go
func delete(k string, v int) bool {
    return v%2 != 0
}
func equal(v1 int, v2 float64) bool {
    return float64(v1) == v2
} 
```

`delete()` 函数的目的是定义要从映射中删除哪些键值对——这个函数作为参数调用 `maps.DeleteFunc()`。当前实现对所有奇数值返回 `true`。这意味着所有奇数值及其键都将被删除。`delete()` 的第一个参数具有映射键的数据类型，而第二个参数具有映射值的数据类型。

`equal()` 函数的目的是定义两个映射的值如何定义相等。在这种情况下，我们想要比较 `int` 值和 `float64` 值。为了使其合法，我们需要将 `int` 值转换为 `float64` 值，这发生在 `equal()` 内部。

让我们继续实现 `main()` 函数。在 `mapsPackage.go` 中找到的 `main()` 函数的第一部分如下：

```go
func main() {
    m := map[string]int{
        "one": 1, "two": 2,
        "three": 3, "four": 4,
    }
    maps.DeleteFunc(m, delete)
    fmt.Println(m) 
```

在之前的代码中，我们定义了一个名为 `m` 的映射，并调用 `maps.DeleteFunc()` 来删除其中的一些元素。

第二部分如下：

```go
 n := maps.Clone(m)
    if maps.Equal(m, n) {
        fmt.Println("Equal!")
    } else {
        fmt.Println("Not equal!")
    }
    n["three"] = 3
    n["two"] = 22
    fmt.Println("Before n:", n, "m:", m)
    maps.Copy(m, n)
    fmt.Println("After n:", n, "m:", m) 
```

`maps.Clone()` 函数返回其参数的浅拷贝。之后，我们调用 `maps.Equal()` 来确保 `maps.Clone()` 正如预期那样工作。

`maps.Copy(dst, src)` 函数将 `src` 中的所有键值对复制到 `dst` 中。当 `src` 中的键已存在于 `dst` 中时，则 `dst` 中的值将被 `src` 中相应键的值覆盖。在我们的程序中，我们将 `n` 复制到 `m` 映射中。

最后一部分如下：

```go
 t := map[string]int{
        "one": 1, "two": 2,
        "three": 3, "four": 4,
    }
    mFloat := map[string]float64{
        "one": 1.00, "two": 2.00,
        "three": 3.00, "four": 4.00,
    }
    eq := maps.EqualFunc(t, mFloat, equal)
    fmt.Println("Is t equal to mFloat?", eq)
} 
```

在最后一部分，我们通过创建两个映射来测试 `maps.EqualFunc()` 的操作，一个使用 `int` 值，另一个使用 `float64` 值，并根据我们之前创建的 `equal()` 函数进行比较。换句话说，`maps.EqualFunc()` 的目的是通过比较它们的函数参数来确定两个映射是否包含相同的键值对。

运行 `mapsPackage.go` 产生以下输出：

```go
$ go run mapsPackage.go
map[four:4 two:2]
Equal!
Before n: map[four:4 three:3 two:22] m: map[four:4 two:2]
After n: map[four:4 three:3 two:22] m: map[four:4 three:3 two:22]
Is t equal to mFloat? true 
```

`maps.DeleteFunc(m, delete)` 语句删除所有值是奇数的键值对，使 `m` 只保留偶数值。此外，对 `maps.Equal()` 的调用返回 `true`，并在屏幕上显示 `Equal!` 消息。`maps.Copy(m, n)` 语句将 `m["two"]` 的值更改为 `22`，并将 `three` 键添加到 `m` 中，其值为 `3`，因为在调用 `maps.Copy()` 之前 `m` 中不存在该键。

# 何时使用泛型

泛型并非万能，不能取代良好的、准确的和理性的程序设计。因此，在考虑使用泛型解决问题时，以下是一些需要记住的原则和个人建议：

+   当创建需要与多种数据类型一起工作的代码时，可能会使用泛型。

+   当接口和反射的实现使代码比必要的更复杂、更难以理解时，应该使用泛型。

+   此外，当预期未来要支持更多数据类型时，可能会使用泛型。

+   再次强调，在编码时使用任何东西的目标是代码的简洁性和易于维护，而不是炫耀你的编码能力。

+   最后，当开发者对泛型感到舒适时，可以使用泛型。没有 Go 规则强制使用泛型。

本节总结了本章内容。请注意，为了使用 `cmp`、`slices` 和 `maps` 包，你需要 Go 版本 1.21 或更高版本。

# 摘要

本章介绍了泛型，并解释了泛型发明的理由。此外，它还介绍了 Go 泛型的语法以及如果你不小心使用泛型可能会出现的一些问题。

当 Go 社区仍在努力探索如何使用泛型时，有两点是重要的：首先，如果你不想使用泛型或者对它们感到不舒服，你不必使用泛型；其次，当你正确使用泛型时，你将需要为支持多种数据类型编写更少的代码。

虽然泛型函数更灵活，但使用泛型的代码通常比使用预定义静态数据类型的代码运行得更慢。因此，你为灵活性付出的代价是执行速度。同样，使用泛型的 Go 代码的编译时间也比不使用泛型的等效代码长。一旦 Go 社区开始在现实场景中使用泛型，泛型提供最高生产力的案例将变得更加明显。最终，编程是关于理解你决策的成本。只有在这种情况下，你才能称自己为程序员。因此，理解使用泛型而不是接口、反射或其他技术的成本是很重要的。

下一章将介绍类型方法，这些方法是附加到数据类型上的函数、反射和接口。所有这些都将使我们能够进一步改进统计应用程序。此外，下一章将比较泛型与接口和反射，因为它们在使用上有重叠。

# 练习

尝试解决以下练习：

+   在 `structures.go` 中创建一个 `PrintMe()` 方法，用于打印链表的所有元素。

+   Go 1.21 版本附带了一个名为 `clear` 的新函数，该函数用于清除映射和切片。对于映射，它会删除所有条目，而对于切片，它会将所有现有值置零。尝试使用它来了解它是如何工作的。

+   使用泛型实现 `delete()` 和 `search()` 功能，针对 `structures.go` 中找到的链表。

+   使用 `structures.go` 中找到的代码，使用泛型实现一个双链表。

# 其他资源

+   为什么使用泛型？[`blog.golang.org/why-generics`](https://blog.golang.org/why-generics)

+   泛型简介：[`go.dev/blog/intro-generics`](https://go.dev/blog/intro-generics)

+   泛型的下一步：[`blog.golang.org/generics-next-step`](https://blog.golang.org/generics-next-step)

+   为 Go 添加泛型的提案：[`blog.golang.org/generics-proposal`](https://blog.golang.org/generics-proposal)

+   所有可比较的类型：[`go.dev/blog/comparable`](https://go.dev/blog/comparable)

+   `constraints` 包：[`pkg.go.dev/golang.org/x/exp/constraints`](https://pkg.go.dev/golang.org/x/exp/constraints)

+   `cmp` 包：[`pkg.go.dev/cmp`](https://pkg.go.dev/cmp)

+   `slices` 包：[`pkg.go.dev/slices`](https://pkg.go.dev/slices)

+   `maps` 包：[`pkg.go.dev/maps`](https://pkg.go.dev/maps)

+   `slices` 包的官方提案（其他 Go 功能也存在类似提案）：[`github.com/golang/go/issues/45955`](https://github.com/golang/go/issues/45955)

# 留下评论！

喜欢这本书吗？通过留下亚马逊评论来帮助像你这样的读者。扫描下面的二维码，获取你选择的免费电子书。

![](img/Review_QR_Code.png)
