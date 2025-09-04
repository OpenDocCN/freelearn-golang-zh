

# 第八章：错误和恐慌

Go 的错误处理一直备受争议。那些来自具有异常处理语言背景（如 Java）的人往往讨厌它，而那些来自错误是函数返回值语言背景（如 C）的人则对此感到舒适。

在两者都有背景的情况下，我认为错误处理的显式性质迫使你在开发的每一步都考虑异常情况。错误生成、错误传递和错误处理需要与“正常路径”（即没有错误发生时）相同的纪律和审查。

如果你注意到了，我在处理错误的三个阶段中做了区分：

+   错误的检测和生成处理涉及检测异常情况并捕获诊断信息

+   错误的传递处理允许错误向上传播到堆栈，可选地用上下文信息装饰它们

+   错误处理涉及实际解决错误，这可能包括终止程序

在本章中，你将了解以下内容：

+   如何生成错误

+   如何通过使用上下文信息来注释它们来传递它们

+   如何处理错误

+   在项目中组织错误

+   处理恐慌

# 错误的返回和处理

这个菜谱展示了如何检测错误以及如何用额外的上下文信息包装错误。

## 如何做...

使用函数或方法的最后一个返回值作为错误：

```go
func DoesNotReturnError() {...}
func MayReturnError() error {...}
func MayReturnStringAndError() (string,error) {...}
```

如果函数或方法成功，它将返回`nil`错误。如果在函数或方法中检测到错误条件，则直接返回该错误或用包含上下文信息的另一个错误包装该错误：

```go
func LoadConfig(f string) (*Config, error) {
   file, err:=os.Open(f)
   if err!=nil {
      return nil, fmt.Errorf("file %s: %w", f,err)
   }
   defer file.Close()
   var cfg Config
   err = json.NewDecoder(file).Decode(&cfg)
   if err!=nil {
     return nil, fmt.Errorf("While unmarshaling %s: %w",f,err)
   }
   return &cfg, nil
}
```

小贴士

不要用`panic`来代替错误。`panic`应该用来表示潜在的 bug 或不可恢复的情况。错误用来表示上下文相关的情况，例如缺少文件或无效输入。

## 它是如何工作的...

Go 使用显式的错误检测和处理。这意味着没有隐式或隐藏的执行路径用于错误（如抛出异常）。Go 的错误仅仅是接口值，一个错误为`nil`被解释为没有错误。上述函数调用了一些可能返回错误的文件管理函数。当这种情况发生时（即，当函数返回非`nil`错误时），这个函数只是用额外信息包装那个错误并返回它。这些额外信息允许调用者，有时是程序的用户确定正确的行动方案。

# 包装错误以添加上下文信息

使用标准库`errors`包，你可以用包含额外上下文信息的另一个错误包装一个错误。此包还提供了设施和约定，让你检查错误树是否包含特定错误或从错误树中提取特定错误。

## 如何做...

使用`fmt.Errorf`向错误添加上下文信息。在下面的例子中，返回的错误将包含来自`os.Open`的错误，并且它还将包括文件名：

```go
file, err := os.Open(fileName)
if err!=nil {
   return fmt.Errorf("%w: While opening %s",err,fileName)
}
```

注意上面`fmt.Errorf`中使用`%w`动词。`%w`动词用于创建一个包装其参数的错误。如果我们使用`%v`或`%s`，返回的错误将包含原始错误的文本，但它不会包装它。

# 比较错误

当你用一个额外的信息包装错误时，新的错误值与原始错误不是同一类型或值。例如，如果文件未找到，`os.Open`可能会返回`os.ErrNotExist`，如果你用额外的信息（如文件名）包装这个错误，调用这个函数的调用者将需要一个方法来获取原始错误以正确处理它。这个菜谱展示了如何处理这样的包装错误值。

## 如何做到这一点...

检查是否有错误很简单：检查错误值是否为`nil`或不是：

```go
file, err := os.Open(fileName)
if err!=nil {
  // File could not be opened
}
```

使用`errors.Is`来检查错误是否是你期望的：

```go
file, err := os.Open(fileName)
if errors.Is(err,os.ErrNotExist) {
  // File does not exist
}
```

## 它是如何工作的...

`errors.Is(err,target error)`通过以下方式比较`err`是否等于`target`：

1.  它检查`err==target`。

1.  如果这还失败，它会检查`err`是否有`Is(error) bool`方法，通过调用`err.Is(target)`。

1.  如果这还失败，它会检查`err`是否有`Unwrap() error`方法，并且`err.Unwrap()`不是`nil`，通过检查`err.Unwrap()`是否等于`target`。

1.  如果这还失败，它会检查`err`是否有`Unwrap() []error`方法，并且`target`等于那些切片元素中的任何一个。

这意味着如果你包装了一个错误，调用者仍然可以检查包装的错误是否发生，并相应地行事。

如果你使用`errors.New()`或`fmt.Errorf()`定义了一个错误，那么返回的错误接口包含一个指向对象的指针。在这种情况下，两个错误具有相同的字符串表示并不意味着它们是相等的。以下程序展示了这种情况：

```go
var e1 = errors.New("test")
var e2 = errors.New("test")
if e1 != e2 {
   fmt.Println("Errors are different!")
}
```

在上面，即使错误字符串相同，`e1`和`e2`都是指向不同对象的指针。程序将打印`Errors are different`。因此，声明如下错误是有效的：

```go
var (
  ErrNotFound = errors.New("Not found")
)
```

与`ErrNotFound`的比较将检查错误值是否是指向与`ErrNotFound`相同对象的指针。

# 结构化错误

一个**结构化错误**提供了在错误到达程序用户之前处理错误时可能至关重要的上下文信息。这个菜谱展示了如何使用这样的错误。

## 如何做到这一点...

1.  定义一个包含捕获错误情况的元数据的结构体。

1.  实现一个`Error() string`方法来使其成为一个`error`。

1.  如果错误可以包装其他错误，包括一个`error`或`[]error`来存储那些。

1.  可选地实现`Is(error) bool`方法来控制如何比较这个错误。

1.  可选地实现`Unwrap() error`或`Unwrap() []error`来返回包装的错误。

## 它是如何工作的...

任何实现了 `error` 接口（只包含一个方法，`Error() string`）的数据类型都可以用作错误。这意味着你可以创建包含详细错误信息的数据结构，稍后可以对其进行操作。所以，如果你需要几个数据字段来描述一个错误，而不是构建一个复杂的字符串并通过 `fmt.Errorf` 返回它，你可以使用一个结构体。

例如，假设你正在解析多行格式化的文本输入。向用户提供准确和有用的信息很重要；没有人会喜欢在没有显示错误位置的情况下收到 `Syntax error` 消息。因此，你声明这个错误结构：

```go
type ErrSyntax struct {
   Line int
   Col int
   Diag string
}
func (err ErrSyntax) Error() string {
  return fmt.Sprintf("Syntax error line: %d col: %d, %s", err.Line, 
  err.Col, err.Diag)
}
```

你现在可以生成有用的错误信息：

```go
func ParseInput(input string) error {
  ...
  if nextRune != ',' {
     return ErrSyntax {
        Line: line,
        Col: col,
        Diag: "Expected comma",
    }
  }
  ...
}
```

你可以使用这些错误信息向用户显示有用的消息或控制交互式响应，例如将光标定位到错误位置或突出显示错误附近的文本。

# 包装结构化错误

结构化错误可以通过包装它来装饰另一个错误，并添加额外的信息。这个配方展示了如何做到这一点。

## 如何做到这一点...

1.  在结构中保留一个错误成员变量（或错误切片）来存储根本原因。

1.  实现 `Unwrap() error`（或 `Unwrap() []error`）方法。

## 它是如何工作的...

你可以将根原因错误包装在一个结构化错误中。这允许你添加有关错误的额外结构化上下文信息：

```go
type ErrFile struct {
   Name string
   When string
   Err error
}
func (err ErrFile) Error() string {
   return fmt.Sprintf("%s: file %s, when %s", err.Err, err.Name, err.
   When)
}
func (err ErrFile) Unwrap() error { return err.Err }
func ReadConfigFile(name string) error {
  f, err:=os.Open(name)
  if err!=nil {
     return ErrFile {
        Name: name,
        Err:err,
        When: "opening configuration file",
     }
  }
  ...
}
```

注意，`Unwrap` 是必要的。如果没有它，以下代码将无法检测到错误是否源自 `os.ErrNotFound`：

```go
err:=ReadConfig("config.json")
if errors.Is(err,os.ErrNotFound) {
   // file not found
}
```

使用 `Unwrap` 方法，`errors.Is` 函数可以遍历封装的错误，并确定其中至少有一个是 `os.ErrNotFound`。

# 通过类型比较结构化错误

在支持 `try`-`catch` 块的语言中，你通常根据错误类型来捕获错误。你可以依靠 `errors.Is` 来模拟相同的功能。

## 如何做到这一点...

在你的错误类型中实现 `Is(error) bool` 方法来定义你关心的等价类型。

## 它是如何工作的...

你可能记得，`errors.Is(err,target)` 函数首先检查 `err = target`，如果失败，则检查 `err.Is(target)`，前提是 `err` 实现了 `Is(error) bool` 方法。因此，你可以使用 `Is(error) bool` 方法来调整如何比较你的自定义错误类型。如果没有 `Is(error) bool` 方法，`errors.Is` 将使用 `==` 进行比较，即使两个错误的类型相同，如果它们的内容不同，比较也会失败。以下示例允许你检查给定的错误是否在错误树中包含 `ErrSyntax`：

```go
type ErrSyntax struct {
   Line int
   Col int
   Err error
}
func (err ErrSyntax) Error() string {...}
func (err ErrSyntax) Is(e error) bool {
  _,ok:=e.(ErrSyntax)
  return ok
}
```

现在，你可以测试一个错误是否是语法错误：

```go
err:=Parse(input)
if errors.Is(err,ErrSyntax{}) {
   // err is a syntax error
}
```

# 从错误树中提取特定错误

## 如何做到这一点...

使用 `errors.As` 函数遍历错误树，找到特定错误，并提取它。

## 它是如何工作的...

与 `errors.Is` 函数类似，`errors.As(err error, target any) bool` 会遍历 `err` 的错误树，直到找到一个可以赋值给 `target` 的错误。这是通过以下方式完成的：

1.  它检查`target`指向的值是否可以分配给`err`指向的值。

1.  如果这失败了，它会检查`err`是否有`As(error) bool`方法，通过调用`err.As(target)`。如果它返回`true`，那么就找到了一个错误。

1.  如果没有，它会检查`err`是否有`Unwrap() error`方法，并且`err.Unwrap()`不是`nil`，然后向下遍历树。

1.  否则，它会检查`err`是否有`Unwrap() []error`方法，并且如果它返回一个非空切片，它会为这些切片中的每一个向下遍历树，直到找到匹配项。

换句话说，`errors.As`将可以分配给`target`的错误复制到`target`中。

以下示例可以用来从一个错误树中提取`ErrSyntax`的实例：

```go
func (err ErrSyntax) As(target any) bool {
   if tgt, ok:=target.(*ErrSyntax); ok {
      *tgt=err
      return true
   }
   return false
}
func main() {
  ...
  err:=Parse(in)
  var syntaxError ErrSyntax
  if errors.As(err,&syntaxError) {
    // syntaxError has a copy of the ErrSyntax
  }
  ...
}
```

注意这里指针的使用。错误结构体被用作值，而你想要一个错误结构体的副本，所以你传递一个指向它的指针：一个`ErrSyntax`实例可以被复制到一个`*ErrSyntax`实例中。如果你的程序使用`*ErrSyntax`作为错误值，你需要通过声明`var syntaxError *ErrSyntax`并将`&syntaxError`传递以复制指针到双指针指向的内存位置。

# 处理恐慌

通常，**恐慌**是一个不可恢复的情况，例如资源耗尽或违反了不变量（即错误）。一些恐慌，如内存不足或除以零，将由运行时（或由硬件引发并作为恐慌传递给程序）引发。当你检测到错误时，你应该在程序中引发恐慌。但是，你如何决定一个情况是否是错误并且你应该恐慌呢？

通常情况下，外部输入（用户输入、API 提交的数据或从文件读取的数据）不应该引起恐慌。这种情况下应该检测并返回给用户有意义的错误。在这种情况下，恐慌可能是指定为你程序中常量字符串的正则表达式编译失败。输入不是可以通过重新运行程序并使用不同输入来修复的东西；它只是一个错误。

如果没有使用`recover`处理恐慌，程序将通过打印诊断输出（包括恐慌的原因和活动 goroutine 的堆栈）来终止。

# 在必要时恐慌

大多数时候，决定是否恐慌或返回错误不是一个容易的决定。这个方法提供了一些指导方针，使这个决定更容易。

## 如何做到这一点...

有两种情况下你可以恐慌。如果以下任何一个情况成立，就恐慌：

+   违反了不变量

+   程序无法在当前状态下继续

不变量是在程序中不能被违反的条件。因此，如果你检测到它被违反，而不是返回一个错误，你应该恐慌。

以下示例来自我编写的一个图形库。一个图包含节点和边，由`*Graph`结构管理。`Graph.NewEdge`方法在两个节点之间创建一条新边。这两个节点必须属于与`NewEdge`方法接收者相同的图，因此如果情况不是这样，应该恐慌，如下所示：

```go
func (g *Graph) NewEdge(from,to *Node) *Edge {
  if from.graph!=g {
     panic("from node is not in graph")
  }
  if to.graph!=g {
     panic("to node is not in graph")
  }
   ...
}
```

在上面，从这个方法返回错误没有任何好处。这显然是一个调用者没有意识到的错误，如果程序被允许继续，`Graph`对象的完整性将被违反，从而产生难以发现的错误。最好的做法是恐慌。

第二种情况是一个无法继续的情况。例如，假设你正在编写一个 Web 应用程序，并从文件系统中加载 HTML 模板。如果此类模板的编译失败，程序无法继续。你应该恐慌。

# 从恐慌中恢复

未处理的恐慌将终止程序。通常，这是唯一正确的做法。然而，有些情况下，你希望失败导致错误的原因，记录它，并继续。例如，一个处理多个并发请求的服务器不会因为其中一个请求恐慌而终止。这个配方展示了你如何从恐慌中恢复。

## 如何操作...

在`defer`函数中使用`recover`语句：

```go
func main() {
  defer func() {
     if r:=recover(); r != nil {
        // deal with the panic
     }
  }()
  ...
}
```

## 它是如何工作的...

当程序恐慌时，在所有延迟块执行完毕后，恐慌的函数将返回。该 goroutine 的堆栈将一个接一个地展开函数，通过运行它们的`deferred`语句进行清理，直到达到 goroutine 的开始，或者其中一个延迟函数调用了`recover`。如果没有恢复恐慌，程序将通过打印诊断信息和堆栈信息而崩溃。如果恢复了恐慌，`recover()`函数将返回传递给`panic`的任何参数，这可以是任何值。

因此，如果你从恐慌中恢复过来，你应该检查恢复的值是否是一个可以用来提供更多有用信息的错误。

# 在恢复中更改返回值

当你从恐慌中恢复时，你通常想返回某种类型的错误来描述发生了什么。这个配方展示了你如何做到这一点。

## 如何操作...

当从恐慌中恢复函数的返回值时，使用命名返回值。

## 它是如何工作的...

**命名返回值**允许你访问和设置函数的返回值。如下所示，你可以使用命名返回值来更改函数的返回值：

```go
func process() (err error) {
  defer func() {
     r:=recover()
     if e, ok:=r.(error); ok {
         err = e
     }
```

# 捕获恐慌的堆栈跟踪

当检测到恐慌时打印或记录堆栈跟踪是识别运行时问题的关键工具。这个配方展示了你如何将堆栈跟踪添加到你的日志消息中。

## 如何操作...

使用带有`recover`的`debug.Stack`函数：

```go
import "runtime/debug"
import "fmt"
func main() {
    defer func() {
        if r := recover(); r != nil {
            stackTrace := string(debug.Stack())
            // Work with stackTrace
            fmt.Println(stackTrace)
        }
    }()
    f()
}
func f() {
   var i *int
   *i=0
}
```

当在恢复函数内部时，`debug.Stack` 函数将返回正在恢复的 panic 的堆栈，而不是调用它的堆栈。因此，如果你能记录这个信息或打印它，它将显示 panic 源的确切位置。

警告

以这种方式获取堆栈是一个昂贵的操作。请谨慎使用，并且仅在必要时使用。

前面的程序将打印以下内容：

```go
goroutine 1 [running]:
runtime/debug.Stack()
     /usr/local/go-faketime/src/runtime/debug/stack.go:24 +0x5e
main.main.func1()
     /tmp/sandbox381445105/prog.go:13 +0x25
panic({0x48bbc0?, 0x5287c0?})
     /usr/local/go-faketime/src/runtime/panic.go:770 +0x132
main.f(...)
     /tmp/sandbox381445105/prog.go:23
main.main()
     /tmp/sandbox381445105/prog.go:18 +0x2e
```

这里：

+   `prog.go:13` 是调用 `debug.Stack()` 的位置

+   `prog.go:23` 是执行 `*i=0` 的位置

+   `prog.go:18` 是调用 `f()` 的位置

正如你所见，堆栈精确地指出了错误的所在位置 (`prog.go:23`)。
