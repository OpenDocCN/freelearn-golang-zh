# 附录 – “Go”前进

在本附录部分，我们将探讨一些在前面章节中没有解释的 Go 编程语言的概念。

以下主题被涵盖：

+   阻塞 goroutine

+   查找堆分配

# 阻塞 goroutine

阻塞 goroutine 可能很重要。最简单的例子是`main`函数。如果我们没有在`main`函数内部使用阻塞调用或循环，程序将终止并重新启动。在大多数情况下，我们不希望程序终止，因为我们可能希望等待任何可能触发代码中进一步操作的输入信号。

现在我们来看一下阻塞 goroutine 的一些可能性。在某些情况下，阻塞 goroutine 是必要的，以便获得时间让调度器在其他 goroutine 上工作。

## 从通道读取

阻塞 goroutine 的一种非常常见的方式是从通道读取。从通道读取将阻塞 goroutine，直到可以读取值。以下代码示例说明了这一点：

```go
func main() {
      blocker := make(chan bool, 1)
     <-blocker
     println("this gets never printed")
}
```

## 一个 select 语句

一个`select`语句让 goroutine 等待多个操作。其语法与`switch`语句的语法类似。以下代码示例实现了一个阻塞直到两个情况之一可以运行的`select`语句：

```go
func main() {
    resultChannel := make(chan bool)
    errChannel := make(chan error)
    select {
    case result := <-resultChannel:
        println(result)
    case err := <-errChannel:
        println(err.Error())
    }
}
```

注意

如果两个情况恰好同时准备好，`select`语句将随机选择一个情况。

我们有时会遇到这样的情况，即我们的`main`函数在其他 goroutine 等待处理传入消息时应该什么也不做。在这种情况下，我们可以使用一个空的`select`语句来无限期地阻塞。以下代码片段是一个这样的例子：

```go
func main() {
    select {}
    println("this gets never printed")
}
```

## 休眠是一个阻塞调用

在某些情况下，我们只想为调度器工作在另一个 goroutine 上争取一些时间。在这种情况下，我们可以使用`time.Sleep()`来短暂休眠，然后继续在当前的 goroutine 上工作。这可以像以下代码示例那样：

```go
func main() {
    for {
        println("do some work")
        time.Sleep(50 * time.Millisecond)
    }
}
```

我们已经学习了不同的方法来阻塞 goroutine，现在让我们学习一点关于分配的内容。

# 查找堆分配

TinyGo 编译器工具链试图以这种方式优化代码，即结果中不留下任何堆分配，但有些分配无法优化。是否有办法知道哪些分配？是的！我们可以禁用`build`和`flash`命令。

当 GC 被禁用时，编译过程将失败并抛出一个错误，该错误指向导致堆分配的代码行。让我们看看以下导致堆分配的代码示例：

```go
package main
var myString *string
func main() {
            value := "my value"
            myString = &value
}
```

在构建此程序时，我们将使用以下命令禁用 GC：

```go
tinygo build -o Appendix/allocations/hex --gc=none --target=arduino Appendix/allocations/main.go
```

这将引发以下错误：

```go
/tmp/tinygo589978451/main.o: In function `main.main':
/home/tobias/go/src/github.com/PacktPublishing/Creative-DIY-Microcontroller-Projects-with-TinyGo-and-WebAssembly/blob/master/Appendix/allocations/main.go:6: undefined reference to `runtime.alloc'
collect2: Error: ld returned 1 as End-Status 
error: failed to link /tmp/tinygo589978451/main: exit status 1
```

将值的指针存储在全局对象中会导致堆分配。我们如何改进程序以不分配堆内存？我们可以简单地在这里省略使用指针。查看以下示例：

```go
package main
var myString string
func main() {
            value := "my value"
            myString = value
}
```

我们现在可以尝试再次构建程序，使用以下命令：

```go
tinygo build -o Appendix/allocations/hex --gc=none --target=arduino Appendix/allocations/main.go
```

此命令将创建输出文件，并且不会抛出任何错误。

如果你想要检查哪些操作会导致**堆分配**，哪些不会，请查看以下链接：

[`tinygo.org/compiler-internals/heap-allocation/`](https://tinygo.org/compiler-internals/heap-allocation/)

如果你想要更好地理解**堆**，请查看以下链接：

[`medium.com/eureka-engineering/understanding-allocations-in-go-stack-heap-memory-9a2631b5035d`](https://medium.com/eureka-engineering/understanding-allocations-in-go-stack-heap-memory-9a2631b5035d)
