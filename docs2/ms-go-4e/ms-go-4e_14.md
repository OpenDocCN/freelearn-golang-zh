# 14

# 效率和性能

每个故事都有一个反派。对于开发者来说，这个反派通常是时间。他们必须在给定的时间内编写代码，理想情况下，代码必须尽可能快地运行。大多数错误和缺陷都是与时间限制作斗争的结果，无论是现实的还是想象的！因此，本章旨在帮助你与时间的第二个方面作斗争：效率和性能。对于时间的第一个方面，你需要一个技术技能良好的好管理者。

本章的第一部分是关于使用基准函数对 Go 代码进行基准测试，这些基准函数用于衡量函数或整个程序的性能。认为某个函数的实现比另一个实现更快是不够的。我们需要能够证明这一点。

之后，我们将讨论 Go 如何管理内存，以及粗心的 Go 代码如何引入内存泄漏。在 Go 中，内存泄漏发生在不再需要的内存没有被正确释放时，导致程序内存使用量随时间增长。理解内存模型对于编写高效、正确和并发的 Go 程序至关重要。在实践中，当我们的代码使用大量内存时（这通常不是情况），我们需要格外注意代码，以获得更好的性能。

最后，我们将展示如何使用 Go 语言结合 eBPF。eBPF，即扩展伯克利包过滤器，是一种使 Linux 内核可编程的技术。它起源于对传统伯克利包过滤器（BPF）的扩展，BPF 是为内核内部数据包过滤而设计的。然而，eBPF 是一个更通用和灵活的框架，允许在内核空间中执行用户提供的程序，而无需修改内核本身。

我们将涵盖以下主题：

+   基准测试代码

+   缓冲与非缓冲文件 I/O

+   错误定义的基准函数

+   Go 内存管理

+   内存泄漏

+   与 eBPF 一起工作

下一个部分是关于基准测试 Go 代码，这有助于你确定你的代码中什么更快，什么更慢——这使得它成为开始搜索效率的完美起点。

# 基准测试代码

基准测试衡量函数或程序的性能，允许你比较不同的实现，并了解代码更改对性能的影响。利用这些信息，你可以轻松地揭示需要重写以改进性能的代码部分。不言而喻，除非你有非常充分的理由，否则你不应该在正在用于其他更重要目的的繁忙机器上对 Go 代码进行基准测试！否则，你可能会干扰基准测试过程，得到不准确的结果，更重要的是，你可能会在机器上产生性能问题。

大多数情况下，操作系统的负载在代码性能中起着关键作用。让我在这里讲一个故事：我为一个项目开发的一个 Java 工具在独立运行时执行了大量的计算，耗时 6,242 秒（大约 1.7 小时）。当四个相同的 Java 命令行工具实例在同一台 Linux 机器上运行时，大约需要一天时间！如果你这么想，一个接一个地运行它们会比同时运行它们快！

Go 在基准测试（以及测试）方面遵循某些约定。最重要的约定是基准函数的名称必须以 `Benchmark` 开头。在 `Benchmark` 词语之后，我们可以放置一个下划线或大写字母。因此，`BenchmarkFunctionName()` 和 `Benchmark_functionName()` 都是有效的基准函数，而 `Benchmarkfunctionname()` 则不是。按照惯例，这样的函数被放在以 `_test.go` 结尾的文件中。一旦基准测试正确，`go test` 子命令就会为你完成所有脏活，包括扫描所有 `*_test.go` 文件中的特殊函数，生成适当的临时 `main` 包，调用这些特殊函数，获取结果，并生成最终输出。

基准函数使用 `testing.B` 变量，而测试函数使用 `testing.T` 变量。这很容易记住。

从 Go 1.17 开始，我们可以使用 `shuffle` 参数（`go test -shuffle=on`）帮助打乱测试和基准的执行顺序。`shuffle` 参数接受一个值（这是随机数生成器的种子），当你想重新播放执行顺序时很有用。它的默认值是关闭。**这个功能背后的逻辑是，有时测试和基准的执行顺序会影响它们的结果**。

下一个子节展示了一个简单的基准测试场景，我们尝试优化切片初始化。

## 一个简单的基准测试场景

我们首先展示一个测试两个函数性能的场景，这两个函数执行相同的事情，但实现方式不同。我们希望能够初始化具有连续值的切片，从 0 开始，到预定义的值。所以，给定一个名为 `mySlice` 的切片，`mySlice[0]` 将具有 `0` 的值，`mySlice[1]` 将具有 `1` 的值，以此类推。相关代码可以在 `ch14/slices` 中找到，其中包含两个名为 `initialize.go` 和 `initialize_test.go` 的文件。`initialize.go` 的 Go 代码分为两部分。第一部分如下：

```go
package main
import (
    "fmt"
)
func InitSliceNew(n int) []int {
    s := make([]int, n)
    for i := 0; i < n; i++ {
    	s[i] = i
    }
    return s
} 
```

在之前的代码中，我们看到使用 `make()` 预分配所需内存空间的所需功能实现。

`initialize.go` 的第二部分如下：

```go
func InitSliceAppend(n int) []int {
    s := make([]int, 0)
    for i := 0; i < n; i++ {
    	s = append(s, i)
    }
    return s
}
func main() {
    fmt.Println(InitSliceNew(10))
    fmt.Println(InitSliceAppend(10))
} 
```

在 `InitSliceAppend()` 中，我们看到一个不同的实现，它从一个空切片开始，并使用多个 `append()` 调用来填充它。`main()` 函数的目的是天真地测试 `InitSliceNew()` 和 `InitSliceAppend()` 的功能。

基准测试函数的实现位于 `initialize_test.go` 中，如下所示：

```go
package main
import (
    "testing"
)
var t []int
func BenchmarkNew(b *testing.B) {
    for i := 0; i < b.N; i++ {
    	t = InitSliceNew(i)
    }
}
func BenchmarkAppend(b *testing.B) {
    for i := 0; i < b.N; i++ {
    	t = InitSliceAppend(i)
    }
} 
```

在这里，我们有两个基准测试函数，分别基准测试 `InitSliceNew()` 和 `InitSliceAppend()`。全局参数 `t` 用于防止 Go 通过忽略 `InitSliceNew()` 和 `InitSliceAppend()` 的返回值来优化 `for` 循环。

请记住，基准测试过程发生在 `for` 循环内部。这意味着，当需要时，我们可以在 `for` 循环外部声明新变量、打开网络连接等。

现在有一些关于基准测试的重要信息：**默认情况下，每个基准函数至少执行一秒钟**——这个持续时间也包括由基准函数调用的函数的执行时间。如果基准函数在不到一秒的时间内返回，`b.N` 的值会增加，并且该函数将再次以 `b.N` 的总次数运行。当 `b.N` 的值为 1 时，它变为 2，然后是 5，然后是 10，然后是 20，然后是 50，以此类推。这是因为函数越快，Go 需要运行它的次数就越多，以获得准确的结果。

在 macOS M1 Max 笔记本上对代码进行基准测试会产生以下类型的输出——你的输出可能会有所不同：

```go
$ go test -bench=. *.go
goos: darwin
goarch: arm64
BenchmarkNew-10             255704         79712 ns/op
BenchmarkAppend-10           86847        143459 ns/op
PASS
ok    command-line-arguments    33.539s 
```

这里有两个重要的点。首先，`-bench` 参数的值指定了将要执行的基准测试函数。使用的 `.` 值是一个正则表达式，它匹配所有有效的基准测试函数。第二个要点是，如果你省略了 `-bench` 参数，则不会执行任何基准测试函数。

生成的输出显示 `InitSliceNew()` 比较快，因为 `InitSliceNew()` 执行了 `255704` 次，每次耗时 `79712 ns`，而 `InitSliceAppend()` 执行了 `86847` 次，每次耗时 `143459 ns`。这完全合理，因为 `InitSliceAppend()` 需要不断分配内存——这意味着切片的长度和容量都会改变，而 `InitSliceNew()` 只分配一次必要的内存。

理解`append()`函数的工作原理将有助于你理解结果。如果底层数组有足够的容量，那么结果切片的长度将增加附加元素的数量，但其容量保持不变。这意味着没有新的内存分配。然而，如果底层数组没有足够的容量，将创建一个新的数组，这意味着将分配新的内存空间，具有更大的容量。之后，切片被更新以引用新的数组，并相应地调整其长度和容量。

下一个小节展示了允许我们减少分配数量的基准测试技术。

## 基准测试内存分配的数量

在这个第二个基准测试场景中，我们将处理与函数操作期间发生的内存分配数量有关的一个性能问题。我们展示了同一程序的两种版本，以说明慢速版本和改进版本之间的差异。所有相关代码都可以在`ch14/alloc`目录下的两个目录中找到，分别命名为`base`和`improved`。

### 初始版本

被基准测试的函数的目的是将消息写入缓冲区。这个版本没有进行优化。这个第一版本的代码位于`ch14/alloc/base`，其中包含两个 Go 源代码文件。第一个文件名为`allocate.go`，包含以下代码：

```go
package allocate
import (
    "bytes"
)
func writeMessage(msg []byte) {
    b := new(bytes.Buffer)
    b.Write(msg)
} 
```

`writeMessage()`函数只是将给定的消息写入一个新的缓冲区（`bytes.Buffer`）。因为我们只关心其性能，所以我们不处理错误处理。

第二个文件，名为`allocate_test.go`，包含基准测试代码，包含以下代码：

```go
package allocate
import (
    "testing"
)
func BenchmarkWrite(b *testing.B) {
    msg := []byte("Mastering Go!")
    for i := 0; i < b.N; i++ {
        for k := 0; k < 50; k++ {
            writeMessage(msg)
        }
    }
} 
```

使用带有`-benchmem`命令行标志的基准测试代码，它还会显示内存分配，产生以下类型的输出：

```go
$ go test -bench=. -benchmem *.go
goos: darwin
goarch: arm64
BenchmarkWrite-10     1148637    1024 ns/op    3200 B/op    50 allocs/op
PASS
ok    command-line-arguments    2.256s 
```

每次执行基准测试函数都需要 50 次内存分配。这意味着在内存分配的数量上还有改进的空间。我们将在下一个小节中尝试减少它们。

### 提高内存分配的数量

在本小节中，我们介绍了三个不同的函数，它们都实现了将消息写入缓冲区的功能。然而，这次，缓冲区作为函数参数给出，而不是内部初始化。

改进版本的代码位于`ch14/alloc/improved`，包含两个 Go 源代码文件。第一个文件名为`improve.go`，包含以下代码：

```go
package allocate
import (
    "bytes"
"io"
)
func writeMessageBuffer(msg []byte, b bytes.Buffer) {
    b.Write(msg)
}
func writeMessageBufferPointer(msg []byte, b *bytes.Buffer) {
    b.Write(msg)
}
func writeMessageBufferWriter(msg []byte, b io.Writer) {
    b.Write(msg)
} 
```

我们这里有三个函数，它们都实现了将消息写入缓冲区的功能。然而，`writeMessageBuffer()`通过值传递缓冲区，而`writeMessageBufferPointer()`传递缓冲区变量的指针。最后，`writeMessageBufferWriter()`使用`io.Writer`接口变量，它也支持`bytes.Buffer`变量。

第二个文件命名为 `improve_test.go`，将分三部分进行展示。第一部分包含以下代码：

```go
package allocate
import (
    "bytes"
"testing"
)
func BenchmarkWBuf(b *testing.B) {
    msg := []byte("Mastering Go!")
    buffer := bytes.Buffer{}
    for i := 0; i < b.N; i++ {
        for k := 0; k < 50; k++ {
            writeMessageBuffer(msg, buffer)
        }
    }
} 
```

这是 `writeMessageBuffer()` 的基准测试函数。缓冲区只分配一次，并通过传递给相关函数在所有基准测试中使用。

第二部分如下：

```go
func BenchmarkWBufPointerNoReset(b *testing.B) {
    msg := []byte("Mastering Go!")
    buffer := new(bytes.Buffer)
    for i := 0; i < b.N; i++ {
        for k := 0; k < 50; k++ {
            writeMessageBufferPointer(msg, buffer)
        }
    }
} 
```

这是 `writeMessageBufferPointer()` 的基准测试函数。再次强调，所使用的缓冲区只分配一次，并在所有基准测试中共享。

`improve_test.go` 文件的最后一部分包含以下代码：

```go
func BenchmarkWBufPointerReset(b *testing.B) {
    msg := []byte("Mastering Go!")
    buffer := new(bytes.Buffer)
    for i := 0; i < b.N; i++ {
        for k := 0; k < 50; k++ {
            writeMessageBufferPointer(msg, buffer)
            buffer.Reset()
        }
    }
}
func BenchmarkWBufWriterReset(b *testing.B) {
    msg := []byte("Mastering Go!")
    buffer := new(bytes.Buffer)
    for i := 0; i < b.N; i++ {
        for k := 0; k < 50; k++ {
            writeMessageBufferWriter(msg, buffer)
            buffer.Reset()
        }
    }
} 
```

在这里，我们可以看到在两个基准测试函数中使用了 `buffer.Reset()`。`buffer.Reset()` 函数的目的是将缓冲区重置为无内容状态。`buffer.Reset()` 函数的结果与 `buffer.Truncate(0)` 相同。`Truncate(n)` 会丢弃缓冲区中除前 `n` 个未读字节之外的所有字节。我们使用 `buffer.Reset()` 是因为它可能提高性能。然而，这一点还有待观察。

对改进版本进行基准测试会产生以下类型的输出：

```go
$ go test -bench=. -benchmem *.go
goos: darwin
goarch: arm64
BenchmarkWBuf-10            1128193   1056 ns/op   3200 B/op 50 allocs/op
BenchmarkWBufPointerNoReset-10 4050562   337.1 ns/op  2120 B/op 0 allocs/op
BenchmarkWBufPointerReset-10   7993546   150.7 ns/op   0 B/op      0 allocs/op
BenchmarkWBufWriterReset-10    7851434   151.8 ns/op   0 B/op   0 allocs/op
PASS
ok      command-line-arguments    7.667s 
```

如 `BenchmarkWBuf()` 基准测试函数的结果所示，将缓冲区作为函数的参数使用并不会自动加快过程，即使我们在基准测试期间共享相同的缓冲区。然而，其他基准测试的情况并非如此。

使用指向缓冲区的指针可以避免在传递给函数之前复制缓冲区——这解释了 `BenchmarkWBufPointerNoReset()` 函数的结果，在该函数中没有额外的内存分配。然而，我们仍然需要每个操作使用 2,120 字节。

使用 `-benchmem` 命令行参数的输出包括两列额外的信息。第四列显示了在基准测试函数的每次执行中平均分配的内存量。第五列显示了用于分配第四列内存值的分配次数。

最后，发现在使用 `writeMessageBufferPointer()` 和 `writeMessageBufferWriter()` 调用 `buffer.Reset()` 后重置缓冲区可以加快处理速度。一个可能的解释是空缓冲区更容易处理。因此，当使用 `buffer.Reset()` 时，我们既有 0 次内存分配，也有 0 字节的操作。结果，`BenchmarkWBufPointerReset()` 和 `BenchmarkWBufWriterReset()` 分别需要每个操作 150.7 纳秒和 151.8 纳秒，这比 `BenchmarkWBuf()` 和 `BenchmarkWBufPointerNoReset()` 分别需要的 1,056 纳秒和 337.1 纳秒快得多。

使用 `buffer.Reset()` 可能更高效，原因可能包括以下之一：

+   **分配内存的重用**：当你调用 `buffer.Reset()` 时，`bytes.Buffer` 所使用的底层字节切片不会被释放。相反，它会被重用。缓冲区的长度被设置为零，使得现有的内存可用于写入新数据。

+   **减少分配开销**：创建一个新的缓冲区涉及到分配一个新的底层字节切片。这种分配伴随着开销，包括管理内存、更新内存分配器的数据结构，以及可能调用垃圾收集器。

+   **避免垃圾收集**：创建和丢弃许多小缓冲区可能导致垃圾收集器压力增加，尤其是在高频缓冲区创建的场景中。通过使用`Reset()`重用缓冲区，你可以减少短期存在的对象数量，从而可能减少对垃圾收集器的影响。

下一节的主题是基准测试缓冲写入。

# 缓冲与非缓冲文件 I/O

在本节中，我们将比较读写文件时的缓冲和非缓冲操作。

在本节中，我们将测试缓冲区的大小是否在写操作的性能中扮演关键角色。相关的代码可以在`ch14/io`中找到。除了相关文件外，该目录还包括一个`testdata`目录，它首次出现在第十三章，*模糊测试和可观察性*中，用于存储与测试过程相关的数据。

`table.go`的代码在此处未展示——请随意查看。`table_test.go`的代码如下：

```go
package table
import (
    "fmt"
"os"
"path"
"strconv"
"testing"
)
var ERR error
var countChars int
func benchmarkCreate(b *testing.B, buffer, filesize int) {
    filename := path.Join(os.TempDir(), strconv.Itoa(buffer))
    filename = filename + "-" + strconv.Itoa(filesize)
    var err error
for i := 0; i < b.N; i++ {
        err = Create(filename, buffer, filesize)
    }
    ERR = err 
```

将`Create()`的返回值存储在名为`err`的变量中，并在之后使用另一个名为`ERR`的全局变量，这个做法很巧妙。我们希望防止编译器执行任何可能排除我们想要测量的函数执行的优化，因为其结果从未被使用。

```go
 err = os.Remove(filename)
    if err != nil {
        fmt.Println(err)
    }
    ERR = err
} 
```

`benchmarkCreate()`的签名或名称都没有使其成为一个基准函数。这是一个辅助函数，允许你调用`Create()`，该函数在磁盘上创建一个新文件；其实现可以在`table.go`中找到，带有适当的参数。其实现是有效的，并且可以被基准函数使用。

```go
func BenchmarkBuffer4Create(b *testing.B) {
    benchmarkCreate(b, 4, 1000000)
}
func BenchmarkBuffer8Create(b *testing.B) {
    benchmarkCreate(b, 8, 1000000)
}
func BenchmarkBuffer16Create(b *testing.B) {
    benchmarkCreate(b, 16, 1000000)
} 
```

这些是三个正确定义的基准函数，它们都调用了`benchmarkCreate()`。基准函数需要一个`*testing.B`变量，并且不返回任何值。在这种情况下，函数名末尾的数字表示缓冲区的大小。

```go
func BenchmarkRead(b *testing.B) {
    buffers := []int{1, 16, 96}
    files := []string{"10.txt", "1000.txt", "5k.txt"} 
```

这是定义将要用于表格测试的数组结构的代码。这使我们免于实现（3x3=）9 个单独的基准函数。

```go
 for _, filename := range files {
        for _, bufSize := range buffers {
            name := fmt.Sprintf("%s-%d", filename, bufSize)
            b.Run(name, func(b *testing.B) {
                for i := 0; i < b.N; i++ {
                    t := CountChars("./testdata/"+filename, bufSize)
                    countChars = t
                }
            })
        }
    }
} 
```

`b.Run()`方法允许你在基准函数内运行一个或多个子基准测试，它接受两个参数。首先，子基准测试的名称，它将在屏幕上显示，其次，实现子基准测试的函数。这是使用表格测试运行多个基准测试并了解其参数的有效方式。只需记住为每个子基准测试定义一个合适的名称，因为这将显示在屏幕上。

运行基准测试会生成以下输出：

```go
$ go test -bench=. -benchmem *.go
goos: darwin
goarch: arm64
BenchmarkBuffer4Create-10     382740   2915 ns/op   384 B/op    5 allocs/op
BenchmarkBuffer8Create-10     444297   2400 ns/op   384 B/op    5 allocs/op
BenchmarkBuffer16Create-10    491230   2165 ns/op   384 B/op  5 allocs/op 
```

前三行分别是 `BenchmarkBuffer4Create()`、`BenchmarkBuffer8Create()` 和 `BenchmarkBuffer16Create()` 基准测试函数的结果，并显示了它们的性能。

```go
BenchmarkRead/10.txt-1-10     146298    8180 ns/op   168 B/op   6 allocs/op
BenchmarkRead/10.txt-16-10    197534    6024 ns/op   200 B/op   6 allocs/op
BenchmarkRead/10.txt-96-10    197245    6148 ns/op   440 B/op   6 allocs/op
BenchmarkRead/1000.txt-1-10   4382        268204 ns/op  168 B/op   6 allocs/op
BenchmarkRead/1000.txt-16-10  32732    36684 ns/op   200 B/op   6 allocs/op
BenchmarkRead/1000.txt-96-10  105078   11337 ns/op   440 B/op      6 allocs/op
BenchmarkRead/5k.txt-1-10     912       1308924 ns/op  168 B/op	  6 allocs/op
BenchmarkRead/5k.txt-16-10    7413        159638 ns/op  200 B/op	  6 allocs/op
BenchmarkRead/5k.txt-96-10    36471    32841 ns/op   440 B/op      6 allocs/op 
```

之前的结果来自表格测试的 9 个子基准测试。

```go
PASS
ok      command-line-arguments    24.518s 
```

那么，这个输出告诉我们什么呢？首先，每个基准测试函数末尾的 `-10` 表示用于其执行的 goroutine 数量，这实际上是 `GOMAXPROCS` 环境变量的值。同样，你还可以看到 `GOOS` 和 `GOARCH` 的值，它们显示了在生成的输出中机器的操作系统和架构。输出中的第二列显示了相关函数执行的次数。执行速度较快的函数比执行速度较慢的函数执行次数更多。例如，`BenchmarkBuffer4Create()` 执行了 `382740` 次，而 `BenchmarkBuffer16Create()` 执行了 `491230` 次，因为它更快！输出中的第三列显示了每次运行的平均时间，并以每基准测试函数执行纳秒（`ns/op`）为单位进行衡量。第三列的值越高，基准测试函数就越慢。**第三列的值较大表明该函数可能需要优化**。

到目前为止，我们已经学习了如何创建基准测试函数来测试我们自己的函数的性能，以便更好地理解可能需要优化的潜在瓶颈。你可能会问，我们需要多久创建一次基准测试函数？答案是简单的。当某些东西运行得比预期慢，或者当你想要在两种或多种实现之间进行选择时。

下一个子节将展示如何比较基准测试结果。

## `benchstat` 工具

现在想象一下，你有一些基准测试数据，并且想要将其与另一台计算机或不同配置产生的结果进行比较。`benchstat` 工具在这里可以帮助你。该工具位于 `https://pkg.go.dev/golang.org/x/perf/cmd/benchstat` 包中，可以使用 `go install golang.org/x/perf/cmd/benchstat@latest` 命令下载。Go 将所有二进制文件放在 `~/go/bin` 目录下，`benchstat` 也不例外。

`benchstat` 工具取代了 `benchcmp` 工具，后者可以在 [`pkg.go.dev/golang.org/x/tools/cmd/benchcmp`](https://pkg.go.dev/golang.org/x/tools/cmd/benchcmp) 找到。

因此，想象一下，我们有两个针对 `table_test.go` 的基准测试结果保存在 `r1.txt` 和 `r2.txt` 中——你应该删除 `go test` 输出中所有不包含基准测试结果的行，这样就会留下所有以 `Benchmark` 开头的行。你可以这样使用 `benchstat`：

```go
$ ~/go/bin/benchstat r1.txt r2.txt
                   │     r1.txt     │                 r2.txt            │
                   │     sec/op     │    sec/op     vs base             │
Buffer4Create-8      10472.0n ± ∞ ¹   830.8n ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Buffer8Create-8       6884.0n ± ∞ ¹   798.9n ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Buffer16Create-8      5010.0n ± ∞ ¹   770.5n ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/10.txt-1-8       14.955µ ± ∞ ¹   3.987µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/10.txt-16-8      12.172µ ± ∞ ¹   2.583µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/10.txt-96-8      11.925µ ± ∞ ¹   2.612µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/1000.txt-1-8      381.3µ ± ∞ ¹   175.8µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/1000.txt-16-8     54.05µ ± ∞ ¹   22.68µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/1000.txt-96-8    19.115µ ± ∞ ¹   6.225µ ± ∞ ¹     ~ (p=1.333 n=1+2) ²
Read/5k.txt-1-8       1812.5µ ± ∞ ¹   895.7µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/5k.txt-16-8       221.8µ ± ∞ ¹   107.7µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
Read/5k.txt-96-8       51.53µ ± ∞ ¹   21.52µ ± ∞ ¹     ~ (p=0.667 n=1+2) ²
geomean                36.91µ         9.717µ        -73.68%
¹ need >= 6 samples for confidence interval at level 0.95
² need >= 4 samples to detect a difference at alpha level 0.05 
```

你可以通过将生成的输出重定向到文件来保存基准测试的结果。例如，你可以运行 `go test -bench=. > output.txt`。

如果最后一列的值是 `~`，就像这里发生的情况一样，这意味着结果没有发生显著变化。之前的输出显示两个结果之间没有差异。关于 `benchstat` 的更多讨论超出了本书的范围。输入 `benchstat -h` 以了解支持的参数。

下一个部分涉及到一个敏感的主题，即错误定义的基准函数。

# 错误定义的基准函数

在定义基准函数时，你应该非常小心，因为你可能会错误地定义它们。看看以下基准函数的 Go 代码：

```go
func BenchmarkFiboI(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = fibo1(i)
    }
} 
```

`BenchmarkFibo()` 函数具有有效的名称和正确的签名。坏消息是，这个基准函数在逻辑上是错误的，并且不会产生任何结果。原因是，正如之前描述的那样，随着 `b.N` 值的增长，基准函数的运行时间也会因为 `for` 循环而增加。这个事实阻止了 `BenchmarkFiboI()` 收敛到一个稳定的数字，从而阻止了函数完成，因此没有返回任何结果。出于类似的原因，下一个基准函数也是错误实现的：

```go
func BenchmarkfiboII(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = fibo1(b.N)
    }
} 
```

另一方面，以下两个基准函数的实现并没有什么问题：

```go
func BenchmarkFiboIV(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = fibo1(10)
    }
}
func BenchmarkFiboIII(b *testing.B) {
    _ = fibo1(b.N)
} 
```

正确的基准函数是识别你代码中瓶颈的工具，你应该将其放入自己的项目中，尤其是在处理文件 I/O 或 CPU 密集型操作时——正如我写这篇文章时，我已经等待了三天，等待一个 Python 程序完成其操作以测试一个数学算法的暴力方法的性能。

基准测试就到这里。下一个部分将讨论 Go 处理内存的方式。

# Go 内存管理

本节的主题是 Go 内存管理。我们将首先陈述一个你应该已经熟悉的事实：Go 为了简单性和使用 **垃圾回收器**（**GC**）而牺牲了内存管理的可见性和完全控制。尽管 GC 操作会给程序的速度带来开销，但它使我们免于手动处理内存，这是一个巨大的优势，并使我们免于许多错误。

在程序执行过程中存在两种类型的内存分配：*动态分配*和*自动分配*。自动分配是指编译器在程序开始执行之前可以推断其生命周期的分配。例如，所有局部变量、函数的返回参数和函数参数都有确定的生存期，这意味着它们可以被编译器自动分配。所有其他分配都是动态进行的，这包括必须在函数作用域之外可用的数据。

我们继续讨论 Go 的内存管理，通过介绍堆和栈来展开，因为大多数的内存分配都发生在这里。

## 堆和栈

*堆*是编程语言存储全局变量的地方——堆是垃圾回收发生的地方。*栈*是编程语言存储函数使用的临时变量的地方——每个函数都有自己的栈。由于 goroutines 位于用户空间，Go 运行时负责管理它们的操作规则。此外，**每个 goroutine 都有自己的栈**，而堆是“共享”的。

动态分配发生在堆上，而自动分配存储在栈上。Go 编译器执行一个称为*逃逸分析*的过程，以确定是否需要在堆上分配内存或者应该保持在栈上。

在 C++中，当你使用`new`运算符创建新变量时，你知道这些变量将进入堆。在 Go 和`new()`、`make()`函数的使用中并非如此。在 Go 中，编译器根据变量的大小和逃逸分析的结果来决定新变量将被放置的位置。这也是为什么我们可以从 Go 函数中返回局部变量的指针。尽管我们在这本书中没有频繁地看到`new()`，但请记住，`new()`返回指向初始化内存的指针。

如果你想知道 Go 程序中的变量在哪里分配，你可以使用`go run`的`-m` `gc`标志。这在`allocate.go`中得到了说明——这是一个常规程序，无需修改即可显示额外的输出，因为所有细节都由 Go 处理。

```go
package main
import "fmt"
const VAT = 24
type Item struct {
    Description string
    Value       float64
}
func Value(price float64) float64 {
    total := price + price*VAT/100
return total
}
func main() {
    t := Item{Description: "Keyboard", Value: 100}
    t.Value = Value(t.Value)
    fmt.Println(t)
    tP := &Item{}
    *&tP.Description = "Mouse"
    *&tP.Value = 100
    fmt.Println(tP)
} 
```

运行`allocate.go`生成下一个输出——这个输出是使用`-gcflags '-m'`的结果，它修改了生成的可执行文件。你不应该使用`-gcflags`标志创建用于生产的可执行二进制文件。

```go
$ go run -gcflags '-m' allocate.go
# command-line-arguments
./allocate.go:12:6: can inline Value
./allocate.go:19:17: inlining call to Value
./allocate.go:20:13: inlining call to fmt.Println
./allocate.go:25:13: inlining call to fmt.Println
./allocate.go:20:13: ... argument does not escape
./allocate.go:20:14: t escapes to heap
./allocate.go:22:8: &Item{} escapes to heap
./allocate.go:25:13: ... argument does not escape
{Keyboard 124}
&{Mouse 100} 
```

`t escapes to heap`这条消息的意思是`t`逃出了函数。简单来说，这意味着`t`在函数外部被使用，并且没有局部作用域（因为它被传递出了函数）。然而，这并不一定意味着变量已经移动到了堆上。在其他情况下，你可能会看到`moved to heap`的消息。这条消息表明编译器决定将一个变量移动到堆上，因为它可能在函数外部被使用。`does not escape`这条消息表示相关的参数没有逃逸到堆上。

理想情况下，我们应该编写算法以使用栈而不是堆，但这是不可能的，因为栈不能分配太大的对象，也不能存储比函数生命周期更长的变量。所以，这取决于 Go 编译器来决定。

输出的最后两行是由两个`fmt.Println()`语句生成的输出。

如果你想得到更详细的输出，可以使用`-m`两次：

```go
$ go run -gcflags '-m -m' allocate.go
# command-line-arguments
./allocate.go:12:6: can inline Value with cost 13 as: func(float64) float64 { total := price + price * VAT / 100; return total }
./allocate.go:17:6: cannot inline main: function too complex: cost 199 exceeds budget 80
./allocate.go:19:17: inlining call to Value
./allocate.go:20:13: inlining call to fmt.Println
./allocate.go:25:13: inlining call to fmt.Println
./allocate.go:22:8: &Item{} escapes to heap:
./allocate.go:22:8:   flow: tP = &{storage for &Item{}}:
./allocate.go:22:8:     from &Item{} (spill) at ./allocate.go:22:8
./allocate.go:22:8:     from tP := &Item{} (assign) at ./allocate.go:22:5
./allocate.go:22:8:   flow: {storage for ... argument} = tP:
./allocate.go:22:8:     from tP (interface-converted) at ./allocate.go:25:14
./allocate.go:22:8:     from ... argument (slice-literal-element) at ./allocate.go:25:13
./allocate.go:22:8:   flow: fmt.a = &{storage for ... argument}:
./allocate.go:22:8:     from ... argument (spill) at ./allocate.go:25:13
./allocate.go:22:8:     from fmt.a := ... argument (assign-pair) at ./allocate.go:25:13
./allocate.go:22:8:   flow: {heap} = *fmt.a:
./allocate.go:22:8:     from fmt.Fprintln(os.Stdout, fmt.a...) (call parameter) at ./allocate.go:25:13
./allocate.go:20:14: t escapes to heap:
./allocate.go:20:14:   flow: {storage for ... argument} = &{storage for t}:
./allocate.go:20:14:     from t (spill) at ./allocate.go:20:14
./allocate.go:20:14:     from ... argument (slice-literal-element) at ./allocate.go:20:13
./allocate.go:20:14:   flow: fmt.a = &{storage for ... argument}:
./allocate.go:20:14:     from ... argument (spill) at ./allocate.go:20:13
./allocate.go:20:14:     from fmt.a := ... argument (assign-pair) at ./allocate.go:20:13
./allocate.go:20:14:   flow: {heap} = *fmt.a:
./allocate.go:20:14:     from fmt.Fprintln(os.Stdout, fmt.a...) (call parameter) at ./allocate.go:20:13
./allocate.go:20:13: ... argument does not escape
./allocate.go:20:14: t escapes to heap
./allocate.go:22:8: &Item{} escapes to heap
./allocate.go:25:13: ... argument does not escape
{Keyboard 124}
&{Mouse 100} 
```

虽然信息更详细，但我发现这个输出太拥挤了。通常，使用`-m`一次就能揭示程序堆和栈背后的情况。

你应该记住的是，堆通常是存储最大内存量的地方。在实践中，这意味着**测量堆大小通常足以理解和计算 Go 进程的内存使用情况**。因此，Go 垃圾回收器的大部分时间都用于处理堆，这意味着当我们想要优化程序的内存使用时，堆是首先要分析的部分。

下一个小节将讨论 Go 内存模型的主要元素。

## Go 内存模型的主要元素

在本节中，我们将讨论 Go 内存模型的主要元素，以便更好地理解幕后发生的事情。

Go 内存模型与以下主要元素一起工作：

+   **程序代码**：当进程即将运行时，操作系统会将程序代码进行内存映射，因此 Go 无法控制这部分。这类数据是只读的。

+   **全局数据**：全局数据也被操作系统以只读状态进行内存映射。

+   **未初始化的数据**：未初始化的数据由操作系统存储在匿名页面中。我们所说的未初始化数据，指的是包的全局变量等数据。尽管在程序开始之前我们可能不知道它们的值，但我们知道程序开始执行时我们需要为它们分配内存。这种内存空间一旦分配就不再释放。因此，垃圾回收器无法控制这部分内存。

+   **堆**：如本章前面所述，这是用于动态分配的堆。

+   **栈**：这些是用于自动分配的栈。

你不需要了解 Go 内存模型所有这些组件的详细信息。你需要记住的是，当我们有意或无意地将对象放入堆中而不让垃圾回收器释放它们，从而不释放它们各自的内存空间时，就会产生问题。我们将在稍后看到与切片和映射相关的内存泄漏案例。

还存在一个名为*Go 分配器*的内部 Go 组件，用于执行内存分配。它可以为 Go 对象动态分配内存块，以使 Go 对象能够正常工作，并且它被优化以防止内存碎片化和锁定。Go 分配器由 Go 团队实现和维护，因此其操作细节可能会发生变化。

下一节将讨论与未正确释放内存空间有关的内存泄漏。

# 内存泄漏

在接下来的小节中，我们将讨论**切片和映射中的内存泄漏**。当分配内存后没有完全释放时，就会发生内存泄漏。

我们将从由错误使用切片引起的内存泄漏开始。

## 切片和内存泄漏

在本小节中，我们将展示使用切片并产生内存泄漏的代码，然后说明避免此类内存泄漏的方法。切片内存泄漏的一个常见场景是在切片不再需要时仍然持有对较大底层数组的引用。这阻止 GC 回收与数组相关的内存。

`slicesLeaks.go`中的代码如下：

```go
package main
import (
    "fmt"
"time"
)
func createSlice() []int {
    return make([]int, 1000000)
}
func getValue(s []int) []int {
    val := s[:3]
    return val
}
func main() {
    for i := 0; i < 15; i++ {
        message := createSlice()
        val := getValue(message)
        fmt.Print(len(val), " ")
        time.Sleep(10 * time.Millisecond)
    }
} 
```

`createSlice()`函数创建了一个具有大型底层数组的切片，这意味着它需要大量的内存。`getValue()`函数取其输入切片的前五个元素，并将这些元素作为切片返回。然而，它这样做的同时还引用了原始输入切片，这意味着该输入切片不能被 GC 释放。是的，这是一个问题！

使用一些额外的命令行参数运行`slicesLeaks.go`会产生以下输出：

```go
$ go run -gcflags '-m -l' slicesLeaks.go
# command-line-arguments
./slicesLeaks.go:9:13: make([]int, 1000000) escapes to heap
./slicesLeaks.go:12:15: leaking param: s to result ~r0 level=0
./slicesLeaks.go:21:12: ... argument does not escape
./slicesLeaks.go:21:16: len(val) escapes to heap
./slicesLeaks.go:21:23: " " escapes to heap
3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 
```

输出表明存在泄漏参数。*泄漏参数*意味着这个函数在返回后以某种方式保持了其参数的存活状态——这就是内存泄漏发生的地方。然而，这并不意味着它被移动到栈上，因为大多数*泄漏参数*都是在堆上分配的。

`slicesNoLeaks.go`中`getValue()`函数的实现是`createSlice()`函数的一个改进版本：

```go
func getValue(s []int) []int {
    returnVal := make([]int, 3)
    copy(returnVal, s)
    return returnVal
} 
```

这次我们创建了要返回的切片部分的副本，这意味着函数不再引用初始切片。因此，GC 将被允许释放其内存。

运行`slicesNoLeaks.go`会产生以下输出：

```go
$ go run -gcflags '-m -l' slicesNoLeaks.go
# command-line-arguments
./slicesNoLeaks.go:9:13: make([]int, 1000000) escapes to heap
./slicesNoLeaks.go:12:15: s does not escape
./slicesNoLeaks.go:13:19: make([]int, 3) escapes to heap
./slicesNoLeaks.go:22:12: ... argument does not escape
./slicesNoLeaks.go:22:16: len(val) escapes to heap
./slicesNoLeaks.go:22:23: " " escapes to heap
3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 
```

因此，我们没有收到关于泄漏参数的消息，这意味着问题已经解决。

接下来，我们将讨论内存泄漏和映射。

## 映射和内存泄漏

本小节讨论由映射（maps）引入的内存泄漏，如`mapsLeaks.go`所示。`mapsLeaks.go`中的代码如下：

```go
package main
import (
    "fmt"
"runtime"
)
func printAlloc() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("%d KB\n", m.Alloc/1024)
}
func main() {
    n := 2000000
    m := make(map[int][128]byte)
    printAlloc()
    for i := 0; i < n; i++ {
    	m[i] = [128]byte{}
    }
    printAlloc()
    for i := 0; i < n; i++ {
    	delete(m, i)
    }
    runtime.GC()
    printAlloc()
    runtime.KeepAlive(m)
    m = nil
    runtime.GC()
    printAlloc()
} 
```

`printAlloc()`是一个用于打印内存信息的辅助函数，而`runtime.KeepAlive(m)`语句保持对`m`的引用，这样映射就不会被垃圾回收。

运行`mapsLeaks.go`会产生以下输出：

```go
$ go run -gcflags '-m -l' mapsLeaks.go
# command-line-arguments
./mapsLeaks.go:11:12: ... argument does not escape
./mapsLeaks.go:11:31: m.Alloc / 1024 escapes to heap
./mapsLeaks.go:16:11: make(map[int][128]byte) does not escape
111 KB
927931 KB
600767 KB
119 KB 
```

`make(map[int][128]byte)`语句仅分配了 111 KB 的内存。然而，当我们填充映射时，它分配了 927,931 KB 的内存。之后，我们删除映射的所有元素，并期望使用的内存会缩小。然而，空映射需要 600,767 KB 的内存！这是因为按设计，映射中的桶（buckets）数量不能缩小。因此，当我们删除所有映射元素时，我们并没有减少现有桶的数量；我们只是将桶中的槽位清零。

然而，使用`m = nil`允许 GC 释放之前由`m`占用的内存，现在只分配了 119 KB 的内存。因此，将`nil`值赋予未使用的对象是一种良好的实践。

最后，我们将介绍一种可以减少内存分配的技术。

## 内存预分配

**内存预分配**是指在数据结构需要之前为其预留内存空间的行为。尽管预分配内存不是万能的，但在某些情况下，它可以避免频繁的内存分配和释放，从而提高性能并减少内存碎片。

当你对所需的容量或大小有良好的估计，你预计会有大量的插入或追加操作，或者当你想要减少内存重新分配并提高性能时，考虑预分配是很重要的。然而，当处理大量数据时，预分配更有意义。

`preallocate.go` 中 `main()` 函数的实现分为两部分。第一部分包含以下代码：

```go
func main() {
    mySlice := make([]int, 0, 100)
    for i := 0; i < 100; i++ {
    	mySlice = append(mySlice, i)
    }
    fmt.Println(mySlice) 
```

在这个例子中，使用 `make()` 函数创建了一个长度为 0、容量为 100 的切片。这为切片预先分配了内存，当元素被追加时，切片可以增长而不需要重复重新分配，这可以加快处理速度。

第二部分如下：

```go
 myMap := make(map[string]int, 10)
    for i := 0; i < 10; i++ {
    	key := fmt.Sprintf("k%d", i)
    	myMap[key] = i
    }
    fmt.Println(myMap)
} 
```

与之前一样，通过提供初始容量，我们减少了在元素添加时频繁调整映射大小的可能性，从而提高了内存使用效率。

下一节将讨论从 Go 语言中使用 eBPF 的方法——因为**eBPF 仅在 Linux 系统上可用**，所以提供的代码只能在 Linux 机器上执行。

# 使用 eBPF

BPF 代表伯克利包过滤器，而 eBPF 代表扩展 BPF。BPF 最初于 1992 年推出，旨在提高数据包捕获工具的性能。2013 年，Alexei Starovoitov 对 BPF 进行了重大重写，该重写于 2014 年被包含在 Linux 内核中，并取代了 BPF。通过这次重写，现在被称为 eBPF 的 BPF 变得更加灵活，可以用于除网络数据包捕获之外的各种任务。

eBPF 软件可以用 BCC、`bpftrace` 或使用 LLVM 编程。LLVM 编译器可以使用支持的编程语言（如 C 或 LLVM 中间表示）将 BPF 程序编译成 BPF 字节码。由于两种方式都因为使用了低级代码而难以编程，因此使用 BCC 或 `bpftrace` 可以使开发人员的工作更加简单。

## 什么是 eBPF？

由于 eBPF 具有众多功能，很难精确描述它能够做什么。相比之下，描述我们如何使用 eBPF 要容易得多。eBPF 可以用于三个主要领域：网络、安全和可观察性。本节重点介绍 eBPF 的可观察性功能（跟踪）。

你可以将 eBPF 视为位于 Linux 内核内部的虚拟机，它可以执行 eBPF 命令，即定制的 BPF 代码。因此，eBPF 使得 Linux 内核可编程，帮助你解决实际问题。请注意，eBPF（以及所有编程语言）本身并不能解决问题。eBPF 只提供了解决问题的工具！eBPF 程序由 Linux 内核的 eBPF 运行时执行。

更详细地说，eBPF 的关键特性和方面包括以下内容：

+   **可编程性**：eBPF 允许用户将小型程序写入并加载到内核中，这些程序可以附加到内核代码中的各种钩子或入口点。这些程序在受限的虚拟机环境中运行，确保安全性和安全性。

+   **内核内执行**：eBPF 程序以安全的方式在内核中执行，使得直接在内核空间执行高效且开销低的操作成为可能。

+   **动态附加点**：eBPF 程序可以附加到内核中的各种钩子或附加点，允许开发者动态地扩展和自定义内核行为。例如，包括网络、跟踪和安全相关的钩子。

+   **可观察性和跟踪**：eBPF 因其允许开发者对内核进行仪器化以收集有关系统行为、性能和交互的见解而被广泛用于可观察性和跟踪目的。像 `bpftrace` 和 `perf` 这样的工具使用 eBPF 提供高级跟踪功能。

+   **网络**：eBPF 在网络中被广泛用于诸如数据包过滤、流量监控和负载均衡等任务。它使得创建高效且可定制的网络解决方案成为可能，而无需修改内核。

+   **性能分析**：eBPF 提供了一个强大的性能分析和分析框架。它允许开发者和管理员收集有关系统性能的详细信息，而不会产生显著的开销。

与传统性能工具相比，eBPF 的主要优势在于其高效性、生产安全性以及它是 Linux 内核的一部分。在实践中，这意味着我们可以使用 eBPF 而无需向 Linux 内核添加或加载任何其他组件。

## 关于可观察性和 eBPF

大多数 Linux 应用程序都在用户空间执行，这是一个没有太多特权的层。尽管使用用户空间更安全、更安全，但它有限制，需要使用系统调用来请求内核访问特权资源。即使是简单的命令在执行时也会使用大量的系统调用。在实践中，这意味着如果我们能够观察我们应用程序的系统调用，我们可以更多地了解它们的行为和操作方式。

当事情按预期运行且我们应用程序的性能良好时，我们通常不太关心性能和执行的系统调用。但当事情出错时，我们迫切需要了解更多关于我们应用程序的操作。在 Linux 内核中放置特殊代码或开发模块以了解我们应用程序的操作是一项困难的任务，可能需要很长时间。这就是可观察性和 eBPF 发挥作用的地方。eBPF、其语言及其工具允许我们动态地看到幕后发生的事情，而无需更改整个 Linux 操作系统。

与 eBPF 通信你所需要的一切是一个支持 `libbpf` 的编程语言（[`github.com/libbpf/libbpf`](https://github.com/libbpf/libbpf)）。除了 C 语言之外，Go 还提供了对 `libbpf` 库的支持（[`github.com/aquasecurity/libbpfgo`](https://github.com/aquasecurity/libbpfgo)）。

下一个子部分将展示如何在 Go 中创建一个 eBPF 工具。

## 在 Go 中创建 eBPF 工具

由于 `gobpf` 是一个外部 Go 包，并且默认情况下，所有最新的 Go 版本都使用模块，所有源代码都应该放在 `~/go/src` 下的某个位置。所提供的工具通过跟踪 `getuid(2)` 系统调用来记录每个用户的用户 ID，并为每个用户 ID 记录计数。

`uid.go` 工具的代码将被分为四个部分进行展示。第一部分包含以下代码：

```go
package main
import (
    "encoding/binary"
"flag"
"fmt"
"os"
"os/signal"
    bpf "github.com/iovisor/gobpf/bcc"
)
import "C"
const source string = `
#include <uapi/linux/ptrace.h>
BPF_HASH(counts);
int count(struct pt_regs *ctx) {
    if (!PT_REGS_PARM1(ctx))
        return 0;
    u64 *pointer;
    u64 times = 0;
    u64 uid;
    uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    pointer = counts.lookup(&uid);
        if (pointer !=0)
            times = *pointer;
    times++;
        counts.update(&uid, &times);
    return 0;
}
` 
```

如果你熟悉 C 编程语言，你应该能认出 `source` 变量 **包含 C 代码**——这是与 Linux 内核通信以获取所需信息的代码。然而，这段代码是从 Go 程序中调用的。

工具的第二部分如下：

```go
func main() {
    pid := flag.Int("pid", -1, "attach to pid, default is all processes")
    flag.Parse()
    m := bpf.NewModule(source, []string{})
    defer m.Close() 
```

在本节的第二部分，我们定义了一个名为 `pid` 的命令行参数，并初始化了一个名为 `m` 的新 eBPF 模块。

工具的第三部分包含以下代码：

```go
 Uprobe, err := m.LoadUprobe("count")
    if err != nil {
        fmt.Fprintf(os.Stderr, "Failed to load uprobe count: %s\n", err)
        return
    }
    err = m.AttachUprobe("c", "getuid", Uprobe, *pid)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Failed to attach uprobe to getuid: %s\n", err)
        return
    }
    table := bpf.NewTable(m.TableId("counts"), m)
    fmt.Println("Tracing getuid()... Press Ctrl-C to end.") 
```

`m.LoadUprobe("count")` 语句加载了 `count()` 函数。通过 `m.AttachUprobe()` 调用启动了对探针的处理。`AttachUprobe()` 方法表示我们想要使用 `Uprobe` 跟踪 `getuid(2)` 系统调用。`bpf.NewTable()` 语句使我们能够访问在 C 代码中定义的 `counts` 哈希表。记住，eBPF 程序是用 C 代码编写的，这些代码存储在一个 `string` 变量中。

工具的最后部分包含以下代码：

```go
 sig := make(chan os.Signal, 1)
    signal.Notify(sig, os.Interrupt)
    <-sig
    fmt.Printf("%s\t%s\n", "User ID", "COUNT")
    for it := table.Iter(); it.Next(); {
        k := binary.LittleEndian.Uint64(it.Key())
        v := binary.LittleEndian.Uint64(it.Leaf())
        fmt.Printf("%d\t\t%d\n", k, v)
    }
} 
```

之前的代码使用了通道和 UNIX 信号处理来阻塞程序。一旦按下 *Ctrl* + *C*，`sig` 通道将解除对程序的阻塞，并借助 `table` 变量打印所需的信息。由于 `table` 变量中的数据是二进制格式，我们需要使用两个 `binary.LittleEndian.Uint64()` 调用来解码它。

为了执行程序，你需要安装 C 编译器和 BPF 库，这取决于你的 Linux 变体。请参考你的 Linux 变体文档了解如何安装 eBPF 的说明。如果你在运行程序时遇到任何问题，请在相关论坛中提问。

运行 `uid.go` 将生成以下类型的输出：

```go
$ go run uid.go
Tracing getuid()... Press Ctrl-C to end.
User ID    COUNT
979        4
0          3 
```

你可以将 `uid.go` 中的代码用作模板，在编写自己的 Go eBPF 工具时使用。

下一个部分将讨论 `rand.Seed()` 函数以及为什么从 Go 版本 1.20 开始不需要使用它。

# 摘要

在本书的这一章中，我们介绍了与基准测试、性能和效率相关的各种高级 Go 语言主题。请记住，基准测试结果可能会受到各种因素的影响，例如硬件、编译器优化和工作负载。在考虑基准测试运行的具体条件时，**仔细和理性地解释结果**是很重要的。

在这一章中，我们了解到 Go 具有自动内存管理，这意味着语言运行时为您处理内存分配和释放。Go 内存管理的主要组件是垃圾回收、自动内存分配和运行时调度器。

本章还介绍了一种非常强大的技术，即 eBPF。如果你在使用 Linux 机器，那么你绝对应该更多地了解 eBPF 以及如何使用 Go 来使用它。由于其多功能性和在 Linux 内核中解决广泛用例的能力，eBPF 框架已经获得了流行。当与 eBPF 一起工作时，你应该首先像系统管理员那样思考，而不是像程序员那样。简单来说，先尝试现有的 eBPF 工具，而不是编写自己的。然而，如果你有一个现有 eBPF 工具无法解决的问题，那么你可能需要开始像开发者一样行事。

下一章将介绍 Go 1.21 和 Go 1.22 以及它们引入的变化。

# 练习

尝试以下练习：

+   创建三种不同的函数实现，用于复制二进制文件，并对它们进行基准测试以找到最快的版本。你能解释为什么这个函数更快吗？

+   编写一个不使用`buffer.Reset()`的`BenchmarkWBufWriterReset()`版本，并看看它的性能如何。

+   这是一个非常困难的任务：在 Go 中创建一个机器学习库。请记住，在幕后，机器学习使用统计和矩阵运算。

# 其他资源

+   gobpf: [`github.com/iovisor/gobpf`](https://github.com/iovisor/gobpf)

+   最小的 Go 二进制文件: [`totallygamerjet.hashnode.dev/the-smallest-go-binary-5kb`](https://totallygamerjet.hashnode.dev/the-smallest-go-binary-5kb)

+   使用 Go 执行跟踪（gotraceui）: [`gotraceui.dev/`](https://gotraceui.dev/)

+   如何使用 Grafana Pyroscope 在 Go 中调试内存泄漏: [`grafana.com/blog/2023/04/19/how-to-troubleshoot-memory-leaks-in-go-with-grafana-pyroscope/`](https://grafana.com/blog/2023/04/19/how-to-troubleshoot-memory-leaks-in-go-with-grafana-pyroscope/)

+   *这里几字节，那里几字节，很快你就在谈论真正的内存了*：[`dave.cheney.net/2021/01/05/a-few-bytes-here-a-few-there-pretty-soon-youre-talking-real-memory`](https://dave.cheney.net/2021/01/05/a-few-bytes-here-a-few-there-pretty-soon-youre-talking-real-memory)

# 加入我们的 Discord 社区

加入我们的社区 Discord 空间，与作者和其他读者进行讨论：

[加入我们的 Discord 服务器](https://discord.gg/FzuQbc8zd6)

![](https://discord.gg/FzuQbc8zd6 )
