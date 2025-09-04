# 15

# 近期 Go 版本的变化

本章是关于最新 Go 版本引入的变化。

首先，我们将看到 Go 的随机数生成能力发生了哪些变化。更具体地说，我们将讨论 `rand.Seed()`。

本章的剩余部分将讨论 Go 1.21 和 Go 1.22，在撰写本文时，它们是最新版本的 Go。**我们不应忘记，编程语言也是程序员开发的软件的一部分**。因此，编程语言和编译器一直在不断改进，以提供新的功能、更好的代码生成、代码优化和更快的运行速度。我们通过讨论 Go 版本 1.21 和 1.22 中引入的最重要改进来结束本章。

我们将涵盖以下主题：

+   关于 `rand.Seed()`

+   Go 1.21 的新特性是什么？

+   Go 1.22 的新特性是什么？

第一部分讨论了 `rand.Seed()` 函数以及为什么从 Go 版本 1.20 开始不再需要使用它。

# 关于 `rand.Seed()`

截至 Go 1.20 版本，使用随机值调用 `rand.Seed()` 来初始化随机数生成器已经没有理由了。然而，使用 `rand.Seed()` 并不会破坏现有的代码。为了获得特定的数字序列，建议调用 `New(NewSource(seed))`。

这在 `ch15/randSeed.go` 中得到了说明——相关的 Go 代码如下：

```go
 src := rand.NewSource(seed)
    r := rand.New(src)
    for i := 0; i < times; i++ {
        fmt.Println(r.Uint64())
    } 
```

`rand.NewSource()` 调用基于给定的种子返回一个新的（伪）随机源。因此，如果使用相同的种子调用，它将返回相同的值序列。`rand.New()` 调用返回一个新的 `*rand.Rand` 变量，它是生成（伪）随机值的东西。由于调用了 `Uint64()`，我们正在生成无符号的 `int64` 值。

运行 `randSeed.go` 产生以下输出：

```go
$ go run randSeed.go 1
Using seed: 1
5577006791947779410
8674665223082153551
$ go run randSeed.go 1
Using seed: 1
5577006791947779410
8674665223082153551 
```

下一部分介绍了 Go 1.21 引入的变化。

# Go 1.21 的新特性是什么？

在本节中，我们将讨论 Go 1.21 带来的两个新特性：标准库中的 `sync.OnceFunc()` 函数和内置函数 `clear`，后者用于删除或清零映射、切片或 *类型参数* 类型的所有元素。

我们将从 `sync.OnceFunc()` 函数开始介绍。

## `sync.OnceFunc()` 函数

`sync.OnceFunc()` 函数是 `sync` 包的一个辅助函数。它的完整签名是 `func OnceFunc(f func()) func()`，这意味着它接受一个函数作为参数并返回另一个函数。更详细地说，`sync.OnceFunc()` 返回一个函数，该函数只调用一次函数 `f`——这里的重要细节是 *只调用一次*。

现在这可能看起来不太清楚，但保存为 `syncOnce.go` 的代码将有助于说明 `sync.OnceFunc()` 的使用。`syncOnce.go` 的代码分为两部分。第一部分如下：

```go
package main
import (
    "fmt"
"sync"
"time"
)
var x = 0
func initializeValue() {
    x = 5
} 
```

`initializeValue()` 函数用于初始化全局变量 `x` 的值。让我们确保 `initializeValue()` 只执行一次。

第二部分包含以下代码：

```go
func main() {
    function := sync.OnceFunc(initializeValue)
    for i := 0; i < 10; i++ {
        go function()
    }
    time.Sleep(time.Second)
    for i := 0; i < 10; i++ {
        x = x + 1
    }
    fmt.Printf("x = %d\n", x)
    for i := 0; i < 10; i++ {
        go function()
    }
    time.Sleep(time.Second)
    fmt.Printf("x = %d\n", x)
} 
```

`sync.OnceFunc(initializeValue)`调用用于确保`initializeValue()`只被执行一次，尽管`function()`多次执行。换句话说，我们确保`initializeValue()`只由第一个 goroutine 执行。

运行`syncOnce.go`会产生以下输出：

```go
$ go run syncOnce.go
x = 15
x = 15 
```

输出显示`x`变量的值只初始化了一次。这意味着`sync.OnceFunc()`可以用于初始化变量、连接或文件，并确保初始化过程只执行一次。

现在，是时候学习`clear`函数了。

## 清除函数

在本小节中，我们将介绍在处理 map 和数组时使用`clear`函数的用法。当用于 map 对象时，`clear()`会清除 map 对象的所有元素。当用于 slice 对象时，`clear()`会将 slice 的所有元素重置为其数据类型的零值，同时保持相同的 slice 长度和容量——这与 map 对象发生的情况完全不同。

相关程序的名称是`clr.go`——重要的 Go 代码如下：

```go
func main() {
    m := map[string]int{"One": 1}
    m["Two"] = 2
    fmt.Println("Before clear:", m)
    clear(m)
    fmt.Println("After clear:", m)
    s := make([]int, 0, 10)
    for i := 0; i < 5; i++ {
        s = append(s, i)
    }
    fmt.Println("Before clear:", s, len(s), cap(s))
    clear(s)
    fmt.Println("After clear:", s, len(s), cap(s))
} 
```

在之前的代码中，我们创建了一个名为`m`的 map 变量和一个名为`s`的 slice 变量。在向它们中添加一些数据后，我们调用`clear()`函数。

运行`clr.go`会产生以下输出：

```go
$ go run clr.go
Before clear: map[One:1 Two:2]
After clear: map[]
Before clear: [0 1 2 3 4] 5 10
After clear: [0 0 0 0 0] 5 10 
```

那么，刚才发生了什么？在调用`clear()`之后，`m`是一个空的 map，而`s`是一个与之前相同长度和容量的 slice，其所有元素都重置为其数据类型的零值，即`int`。

下一节将介绍 Go 1.22 中引入的最重要变化。

# Go 1.22 的新特性是什么？

在完成本书的写作过程中，Go 1.22 版本正式发布。在本节中，我们将介绍 Go 1.22 版本中最有趣的新特性和改进。

+   循环变量中不再有共享。

+   缩小切片大小的函数（`Delete()`、`DeleteFunc()`、`Compact()`、`CompactFunc()`和`Replace()`）现在将新长度和旧长度之间的元素置零。

+   `math/rand`有一个更新版本，称为`math/rand/v2`。

请记住，在 Go 1.22 中，标准库的 HTTP 路由功能得到了改进。在实践中，这意味着`net/http.ServeMux`使用的模式已经增强，可以接受方法和通配符。更多关于这个的信息可以在[`pkg.go.dev/net/http@master#ServeMux`](https://pkg.go.dev/net/http@master#ServeMux)找到。

我们将首先介绍`slices`包中的变化。

## 切片的变化

除了缩小切片大小的函数的变化之外，还增加了`slices.Concat()`，它**连接多个切片**。所有这些都在`sliceChanges.go`中展示。`main()`函数的代码分为两部分。

`sliceChanges.go` 的第一部分代码如下：

```go
func main() {
    s1 := []int{1, 2}
    s2 := []int{-1, -2}
    s3 := []int{10, 20}
    conCat := slices.Concat(s1, s2, s3)
    fmt.Println(conCat) 
```

在前面的代码中，我们使用 `slices.Concat()` 连接三个切片。

`sliceChanges.go` 的其余部分包含以下代码：

```go
 v1 := []int{-1, 1, 2, 3, 4}
    fmt.Println("v1:", v1)
    v2 := slices.Delete(v1, 1, 3)
    fmt.Println("v1:", v1)
    fmt.Println("v2:", v2)
} 
```

如前所述，`slices.Delete()` 将其参数指定的切片中删除的元素置零，并返回一个不包含删除切片元素的切片——因此 `v1` 的长度与之前相同，但 `v2` 的长度更小。

运行 `sliceChanges.go` 产生以下输出：

```go
$ go run sliceChanges.go
[1 2 -1 -2 10 20]
v1: [-1 1 2 3 4]
v1: [-1 3 4 0 0]
v2: [-1 3 4] 
```

第一行显示了连接切片 (`conCat`) 的内容。第二行包含 `v1` 的初始版本，而第三行显示了调用 `slices.Delete()` 后 `v1` 的内容。最后一行包含存储在 `v2` 切片中的 `slices.Delete()` 的返回值。

接下来，我们将查看 `for` 循环中的变化。

## `for` 循环中的变化

Go 1.22 在 `for` 循环中引入了一些变化，我们将使用 `changesForLoops.go` 在本小节中展示这些变化。`changesForLoops.go` 中 `main()` 函数的代码将分为两部分。第一部分如下：

```go
func main() {
    for x := range 5 {
        fmt.Print(" ", x)
    }
    fmt.Println() 
```

因此，从 Go 1.22 开始，`for` 循环可以遍历整数。

`changesForLoops.go` 的最后一部分如下：

```go
 values := []int{1, 2, 3, 4, 5}
    for _, val := range values {
        go func() {
            fmt.Printf("%d ", val)
        }()
    }
    time.Sleep(time.Second)
    fmt.Println()
} 
```

因此，从 Go 1.22 开始，每次执行 `for` 循环时，都会**分配一个新的变量**。这意味着不再共享循环变量，这意味着可以在 goroutine 内使用循环变量而无需担心竞态条件。

这也意味着 `ch08/goClosure.go` 在执行时不会遇到任何问题。然而，编写清晰的代码始终被视为一种良好的实践。

运行 `changesForLoops.go` 产生以下输出：

```go
$ go run changesForLoops.go
 0 1 2 3 4
5 3 4 1 2 
```

输出的第一行显示循环可以遍历整数。第二行输出验证了每个通过 `for` 循环创建的 goroutine 都使用不同的、独立的循环变量副本。

最后，我们展示了 `math/rand` 包更新版本的新功能。

## `math/rand/v2` 包

Go 1.22 引入了对 `math/rand` 包的更新，名为 `math/rand/v2`。此包的功能在 `randv2.go` 中展示，分为三个部分。`randv2.go` 的第一部分如下：

```go
package main
import (
    "fmt"
"math/rand/v2"
)
func Read(p []byte) (n int, err error) {
    for i := 0; i < len(p); {
        val := rand.Uint64()
        for j := 0; j < 8 && i < len(p); j++ {
            p[i] = byte(val & 0xff)
            val >>= 8
            i++
        }
    }
    return len(p), nil
} 
```

其中最重要的变化之一是 `math/rand` 中 `Read()` 方法的弃用。然而，使用 `Uint64()` 方法实现了一个自定义的 `Read()` 函数。

第二部分包含以下代码：

```go
func main() {
    str := make([]byte, 3)
    nChar, err := Read(str)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Printf("Read %d random bytes\n", nChar)
        fmt.Printf("The 3 random bytes are: %v\n", str)
    } 
```

在这部分中，我们调用之前实现的 `Read()` 来获取 3 个随机字节。

`randv2.go` 的最后一部分包含以下代码：

```go
 var max int = 100
    n := rand.N(max)
    fmt.Println("integer n =", n)
    var uMax uint = 100
    uN := rand.N(uMax)
    fmt.Println("unsigned int uN =", uN)
} 
```

这里引入了适用于任何整数类型的泛型函数。在之前的代码中，我们使用 `rand.N()` 获取 `int` 值以及 `uint` 值。`rand.N()` 的参数指定了它将要返回的值的类型。

`rand.N()` 也可以用于时间长度，因为 `time.Duration` 是基于 `int64` 的。

使用 Go 1.22 或更高版本运行 `randv2.go` 会产生以下类型的输出：

```go
$ go run randv2.go
Read 3 random bytes
The 3 random bytes are: [71 215 175]
integer n = 0
unsigned int uN = 2 
```

本小节总结了本章内容，这也是本书的最后一章！感谢您阅读整本书，感谢您选择我的书来学习 Go！

# 摘要

在本书的最后一章，我们介绍了 Go 1.21 和 Go 1.22 中引入的有趣且重要的变化，以便更清晰地了解 Go 语言是如何不断改进和演变的。

那么，Go 开发者的未来看起来如何？简而言之，看起来非常美好！你应该已经享受在 Go 中编程，随着语言的演变，你应该继续这样做。如果你想了解 Go 的最新和最伟大之处，你绝对应该访问 Go 团队的官方 GitHub 页面 [`github.com/golang`](https://github.com/golang)。

Go 帮助你创建出色的软件！所以，去创建出色的软件！记住，**当我们享受我们所做的事情时，我们最有效率**！

# 练习

尝试以下练习：

+   将 `./ch02/genPass.go` 中的 `rand.Seed()` 调用改为 `rand.New(rand.NewSource(seed))` 以进行必要的更改。

+   类似地，将 `./ch02/randomNumbers.go` 中的 `rand.Seed()` 调用替换为 `rand.New(rand.NewSource(seed))` 以进行必要的更改。

# 其他资源

+   罗布·派克——GopherConAU 2023 的“我们做对了什么，做错了什么”演讲：[`www.youtube.com/watch?v=yE5Tpp2BSGw`](https://www.youtube.com/watch?v=yE5Tpp2BSGw)

+   认识 Go 的作者：[`youtu.be/3yghHvvZQmA`](https://youtu.be/3yghHvvZQmA)

+   这是一段布莱恩·肯尼根采访肯·汤普森的视频——与 Go 语言无直接关系：[`youtu.be/EY6q5dv_B-o`](https://youtu.be/EY6q5dv_B-o)

+   布莱恩·肯尼根关于成功语言设计的讨论——与 Go 语言无直接关系：[`youtu.be/Sg4U4r_AgJU`](https://youtu.be/Sg4U4r_AgJU)

+   布莱恩·肯尼根：来自 Lex Fridman 播客的 UNIX、C、AWK、AMPL 和 Go 编程：[`youtu.be/O9upVbGSBFo`](https://youtu.be/O9upVbGSBFo)

+   Go 1.21 发布说明：[`go.dev/doc/go1.21`](https://go.dev/doc/go1.21)

+   Go 1.22 发布说明：[`go.dev/doc/go1.22`](https://go.dev/doc/go1.22)

**作者的话**

成为一名优秀的程序员很困难，但这是可以做到的。持续进步，谁知道呢——你可能会成名，甚至有关于你的电影！

感谢您阅读这本书。请随时与我联系，提出建议、问题或可能的其他书籍的想法！

*荣耀归主*

# 加入我们的 Discord 社区

加入我们社区的 Discord 空间，与作者和其他读者进行讨论：

[`discord.gg/FzuQbc8zd6`](https://discord.gg/FzuQbc8zd6)

![](https://discord.gg/FzuQbc8zd6)
