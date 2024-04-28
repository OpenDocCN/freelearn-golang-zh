# 构建伪终端

本章将介绍伪终端应用程序。许多程序（如 SQL 或 SSH 客户端）都是构建为伪终端，因为它能够在终端内进行交互使用。这些类型的应用程序非常重要，因为它们允许我们在没有图形界面的环境中控制应用程序，例如通过**安全外壳**（**SSH**）连接到服务器时。本章将指导您创建一些此类应用程序。

本章将涵盖以下主题：

+   终端和伪终端

+   基本伪终端

+   高级伪终端

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，您可以参考[第三章]（602a92d5-25f7-46b8-83d4-10c6af1c6750.xhtml），*Go 概述*。

# 理解伪终端

伪终端或伪电传打字机是在终端或电传打字机下运行并模拟其行为的应用程序。这是一种非常方便的方式，可以在没有图形界面的终端内运行交互式软件。这是因为它使用终端本身来模拟一个终端。

# 从电传打字机开始

**电传打字机**（**TTY**）或**电传打印机**是通过串行端口控制的电机式打字机的名称。它连接到能够向设备发送信息以打印的计算机上。数据由一系列有限的符号组成，例如 ASCII 字符，具有固定的字体。这些设备作为早期计算机的用户界面，因此它们在某种意义上是现代屏幕的前身。

当屏幕取代打印机作为输出设备时，它们的内容以类似的方式组织：字符的二维矩阵。在早期阶段，它们被称为玻璃 TTY，字符显示仍然是显示本身的一部分，由其自己的逻辑电路控制。随着第一批视频显示卡的到来，计算机能够拥有一个不依赖硬件的界面。

作为操作系统的主要界面使用的仅文本控制台从 TTY 继承其名称，并被称为控制台。即使操作系统运行在现代操作系统上的图形环境中，用户仍然可以访问一定数量的虚拟控制台，这些控制台作为**命令行界面**（**CLI**）使用，通常称为 shell。

# 伪电传打字机

许多应用程序设计为在 shell 内工作，但其中一些是在模仿 shell 的行为。图形界面有一个专门用于执行 shell 的终端模拟器。这些类型的应用程序被称为**伪电传打字机**（**PTY**）。为了被视为 PTY，应用程序需要能够执行以下操作：

+   接受用户输入

+   将输入发送到控制台并接收输出

+   向用户显示此输出

已经有一些示例可用的 Linux 实用程序，其中最显著的是**screen**。这是一个伪终端应用程序，允许用户使用多个 shell 并对其进行控制。它可以打开和关闭新的 shell，并在所有打开的 shell 之间切换。它允许用户命名一个会话，因此，如果由于任何意外原因而被终止，用户可以恢复会话。

# 创建基本 PTY

我们将从创建输入管理器的简单版本的伪终端开始，然后创建命令选择器，最后创建命令执行。

# 输入管理

标准输入可用于接收用户命令。我们可以通过使用缓冲输入来读取行并打印它们。为了读取一行，有一个有用的命令`bufio.Scanner`，它已经提供了一个行读取器。代码将类似于以下代码片段：

```go
s := bufio.NewScanner(os.Stdin)
w := os.Stdout
fmt.Fprint(w, "Some welcome message\n")
for {
    s.Scan() // get next the token
    fmt.Fprint(w, "You wrote \"") 
    w.Write(s.Bytes())
    fmt.Fprintln(w, "\"\n") // writing back the text
}
```

由于此代码没有退出点，我们可以从创建第一个命令`exit`开始，该命令将终止 shell 执行。我们可以对代码进行一些小改动，使其正常工作，如下所示：

```go
s := bufio.NewScanner(os.Stdin)
w := os.Stdout
fmt.Fprint(w, "Some welcome message\n")
for {
    s.Scan() // get next the token
    msg := string(s.Bytes())
    if msg == "exit" {
        return
    }
    fmt.Fprintf (w, "You wrote %q\n", msg) // writing back the text
}
```

现在应用程序有了除`kill`命令之外的退出点。目前，除了`exit`命令之外，它并没有实现任何命令，而只是打印出您输入的任何内容。

# 选择器

为了能够正确解释命令，消息需要被分割成参数。这与操作系统应用于传递给进程的参数的逻辑相同。`strings.Split`函数通过指定空格作为第二个参数并将字符串分割成单词来实现这一点，如下面的代码所示：

```go
args := strings.Split(string(s.Bytes()), " ")
cmd := args[0]
args = args[1:]
```

可以对`cmd`执行任何类型的检查，例如以下的`switch`语句：

```go
switch cmd {
case "exit":
    return
case "someCommand":
    someCommand(w, args)
case "anotherCommand":
    anotherCommand(w, args)
}
```

这允许用户通过定义一个函数并在`switch`语句中添加一个新的`case`来添加新的命令。

# 命令执行

现在一切都准备就绪，唯一剩下的就是定义各种命令将实际执行的操作。我们可以定义执行命令的函数类型以及“switch”的行为：

```go
var cmdFunc func(w io.Writer, args []string) (exit bool)
switch cmd {
case "exit":
    cmdFunc = exitCmd
}
if cmdFunc == nil {
    fmt.Fprintf(w, "%q not found\n", cmd)
    continue
}
if cmdFunc(w, args) { // execute and exit if true
    return
}
```

返回值告诉应用程序是否需要终止，并允许我们轻松定义我们的`exit`函数，而不需要它成为一个特殊情况：

```go
func exitCmd(w io.Writer, args []string) bool {
    fmt.Fprintf(w, "Goodbye! :)")
    return true
}
```

现在我们可以实现任何类型的命令，具体取决于我们应用程序的范围。让我们创建一个`shuffle`命令，它将使用`math`/`rand`包以随机顺序打印参数：

```go
func shuffle(w io.Writer, args ...string) bool {
    rand.Shuffle(len(args), func(i, j int) {
        args[i], args[j] = args[j], args[i]
    })
    for i := range args {
        if i > 0 {
            fmt.Fprint(w, " ")
        }
        fmt.Fprintf(w, "%s", args[i])
    }
    fmt.Fprintln(w)
    return false
}
```

我们可以通过创建一个“print”命令与文件系统和文件进行交互，该命令将在输出中显示文件的内容：

```go
func print(w io.Writer, args ...string) bool {
    if len(args) != 1 {
        fmt.Fprintln(w, "Please specify one file!")
        return false
    }
    f, err := os.Open(args[0])
    if err != nil {
        fmt.Fprintf(w, "Cannot open %s: %s\n", args[0], err)
    }
    defer f.Close()
    if _, err := io.Copy(w, f); err != nil {
        fmt.Fprintf(w, "Cannot print %s: %s\n", args[0], err)
    }
    fmt.Fprintln(w)
    return false
}
```

# 一些重构

伪终端应用程序的当前版本可以通过一些重构来改进。我们可以通过将命令定义为自定义类型，并添加描述其行为的一些方法来开始：

```go
type cmd struct {
    Name string // the command name
    Help string // a description string
    Action func(w io.Writer, args ...string) bool
}

func (c cmd) Match(s string) bool {
  return c.Name == s
}

func (c cmd) Run(w io.Writer, args ...string) bool {
  return c.Action(w, args...)
}
```

每个命令的所有信息都可以包含在一个结构中。我们还可以开始定义依赖其他命令的命令，比如帮助命令。如果我们在`var cmds []cmd`包中定义了一些命令的切片或映射，那么`help`命令将如下所示：

```go
help := cmd{
    Name: "help",
    Help: "Shows available commands",
    Action: func(w io.Writer, args ...string) bool {
        fmt.Fprintln(w, "Available commands:")
        for _, c := range cmds {
            fmt.Fprintf(w, " - %-15s %s\n", c.Name, c.Help)
        }
        return false
    },
}
```

选择正确命令的主循环的部分将略有不同；它需要在切片中找到匹配项并执行它：

```go
for i := range cmds {
    if !cmds[i].Match(args[0]) {
        continue
    }
    idx = i
    break
}
if idx == -1 {
    fmt.Fprintf(w, "%q not found. Use `help` for available commands\n", args[0])
    continue
}
if cmds[idx].Run(w, args[1:]...) {
    fmt.Fprintln(w)
    return
}
```

现在有一个`help`命令，显示了可用命令的列表，我们可以建议用户在每次指定不存在的命令时使用它——就像我们当前检查索引是否已从其默认值`-1`更改一样。

# 改进 PTY

现在我们已经看到如何创建一个基本的伪终端，我们将看到如何通过一些附加功能来改进它。

# 多行输入

可以改进的第一件事是参数和间距之间的关系，通过添加对带引号字符串的支持。这可以通过具有自定义分割函数的`bufio.Scanner`来实现，该函数的行为类似于`bufio.ScanWords`，除了它知道引号的存在。以下代码演示了这一点：

```go
func ScanArgs(data []byte, atEOF bool) (advance int, token []byte, err error) {
    // first space
    start, first := 0, rune(0)
    for width := 0; start < len(data); start += width {
        first, width = utf8.DecodeRune(data[start:])
        if !unicode.IsSpace(first) {
            break
        }
    }
    // skip quote
    if isQuote(first) {
        start++
    }
```

该函数有一个跳过空格并找到第一个非空格字符的第一个块；如果该字符是引号，则跳过它。然后，它查找终止参数的第一个字符，对于普通参数是空格，对于其他参数是相应的引号：

```go
    // loop until arg end character
    for width, i := 0, start; i < len(data); i += width {
        var r rune
        r, width = utf8.DecodeRune(data[i:])
        if ok := isQuote(first); !ok && unicode.IsSpace(r) || ok  
            && r == first {
                return i + width, data[start:i], nil
        }
    }
```

如果在引用上下文中达到文件结尾，则返回部分字符串；否则，不跳过引号并请求更多数据：

```go
    // token from EOF
    if atEOF && len(data) > start {
        return len(data), data[start:], nil
    }
    if isQuote(first) {
        start--
    }
    return start, nil, nil
}
```

完整的示例可在以下链接找到：[`play.golang.org/p/CodJjcpzlLx`](https://play.golang.org/p/CodJjcpzlLx)。

现在我们可以使用这个作为解析参数的行，同时使用如下定义的辅助结构`argsScanner`：

```go
type argsScanner []string

func (a *argsScanner) Reset() { *a = (*a)[0:0] }

func (a *argsScanner) Parse(r io.Reader) (extra string) {
    s := bufio.NewScanner(r)
    s.Split(ScanArgs)
    for s.Scan() {
        *a = append(*a, s.Text())
    }
    if len(*a) == 0 {
        return ""
    }
    lastArg := (*a)[len(*a)-1]
    if !isQuote(rune(lastArg[0])) {
        return ""
    }
    *a = (*a)[:len(*a)-1]
    return lastArg + "\n"
}
```

通过更改循环的工作方式，这个自定义切片将允许我们接收带引号和引号之间的新行的行：

```go
func main() {
 s := bufio.NewScanner(os.Stdin)
 w := os.Stdout
 a := argsScanner{}
 b := bytes.Buffer{}
 for {
        // prompt message 
        a.Reset()
        b.Reset()
        for {
            s.Scan()
            b.Write(s.Bytes())
            extra := a.Parse(&b)
            if extra == "" {
                break
            }
            b.WriteString(extra)
        }
        // a contains the split arguments
    }
}
```

# 为伪终端提供颜色支持

伪终端可以通过提供彩色输出来改进。我们已经看到，在 Unix 中有可以改变背景和前景颜色的转义序列。让我们首先定义一个自定义类型：

```go
type color int

func (c color) Start(w io.Writer) {
    fmt.Fprintf(w, "\x1b[%dm", c)
}

func (c color) End(w io.Writer) {
    fmt.Fprintf(w, "\x1b[%dm", Reset)
}

func (c color) Sprintf(w io.Writer, format string, args ...interface{}) {
    c.Start(w)
    fmt.Fprintf(w, format, args...)
    c.End(w)
}

// List of colors
const (
    Reset color = 0
    Red color = 31
    Green color = 32
    Yellow color = 33
    Blue color = 34
    Magenta color = 35
    Cyan color = 36
    White color = 37
)
```

这种新类型可以用于增强具有彩色输出的命令。例如，让我们使用交替颜色来区分字符串，现在我们支持带有空格的参数的`shuffle`命令：

```go
func shuffle(w io.Writer, args ...string) bool {
    rand.Shuffle(len(args), func(i, j int) {
        args[i], args[j] = args[j], args[i]
    })
    for i := range args {
        if i > 0 {
            fmt.Fprint(w, " ")
        }
        var f func(w io.Writer, format string, args ...interface{})
        if i%2 == 0 {
            f = Red.Fprintf
        } else {
            f = Green.Fprintf
        }
        f(w, "%s", args[i])
    }
    fmt.Fprintln(w)
    return false
}
```

# 建议命令

当指定的命令不存在时，我们可以建议一些类似的命令。为了这样做，我们可以使用 Levenshtein 距离公式，通过计算从一个字符串到另一个字符串所需的删除、插入和替换来衡量字符串之间的相似性。

在下面的代码中，我们将使用`agnivade/levenshtein`包，这将通过`go get`命令获得：

```go
go get github.com/agnivade/levenshtein/...
```

然后，我们定义一个新函数，当现有命令没有匹配时调用：

```go
func commandNotFound(w io.Writer, cmd string) {
    var list []string
    for _, c := range cmds {
        d := levenshtein.ComputeDistance(c.Name, cmd)
        if d < 3 {
            list = append(list, c.Name)
        }
    }
    fmt.Fprintf(w, "Command %q not found.", cmd)
    if len(list) == 0 {
        return
    }
    fmt.Fprint(w, " Maybe you meant: ")
    for i := range list {
        if i > 0 {
            fmt.Fprint(w, ", ")
        }
        fmt.Fprintf(w, "%s", list[i])
    }
}
```

# 可扩展命令

我们伪终端的当前限制是其可扩展性。如果需要添加新命令，需要直接添加到主包中。我们可以考虑一种方法，将命令与主包分离，并允许其他用户使用其命令扩展功能：

1.  第一步是创建一个导出的命令。让我们使用一个接口来定义一个命令，以便用户可以实现自己的命令：

```go
// Command represents a terminal command
type Command interface {
    GetName() string
    GetHelp() string
    Run(input io.Reader, output io.Writer, args ...string) (exit bool)
}
```

1.  现在我们可以指定一系列命令和一个函数，让其他包添加其他命令：

```go
// ErrDuplicateCommand is returned when two commands have the same name
var ErrDuplicateCommand = errors.New("Duplicate command")

var commands []Command

// Register adds the Command to the command list
func Register(command Command) error {
    name := command.GetName()
    for i, c := range commands {
        // unique commands in alphabetical order
        switch strings.Compare(c.GetName(), name) {
        case 0:
            return ErrDuplicateCommand
        case 1:
            commands = append(commands, nil)
            copy(commands[i+1:], commands[i:])
            commands[i] = command
            return nil
        case -1:
            continue
        }
    }
    commands = append(commands, command)
    return nil
}
```

1.  我们可以提供一个命令的基本实现，以执行简单的功能：

```go
// Base is a basic Command that runs a closure
type Base struct {
    Name, Help string
    Action func(input io.Reader, output io.Writer, args ...string) bool
}

func (b Base) String() string { return b.Name }

// GetName returns the Name
func (b Base) GetName() string { return b.Name }

// GetHelp returns the Help
func (b Base) GetHelp() string { return b.Help }

// Run calls the closure
func (b Base) Run(input io.Reader, output io.Writer, args ...string) bool {
    return b.Action(input, output, args...)
}
```

1.  我们可以提供一个函数，将命令与名称匹配：

```go
// GetCommand returns the command with the given name
func GetCommand(name string) Command {
    for _, c := range commands {
        if c.GetName() == name {
            return c
        }
    }
    return suggest
}
```

1.  我们可以使用前面示例中的逻辑，使此函数返回建议的命令，其定义如下：

```go
var suggest = Base{
    Action: func(in io.Reader, w io.Writer, args ...string) bool {
        var list []string
        for _, c := range commands {
            name := c.GetName()
            d := levenshtein.ComputeDistance(name, args[0])
            if d < 3 {
                list = append(list, name)
            }
        }
        fmt.Fprintf(w, "Command %q not found.", args[0])
        if len(list) == 0 {
            return false
        }
        fmt.Fprint(w, " Maybe you meant: ")
        for i := range list {
            if i > 0 {
                fmt.Fprint(w, ", ")
            }
            fmt.Fprintf(w, "%s", list[i])
        }
        return false
    },
}
```

1.  现在我们可以在`exit`和`help`包中注册一些命令。只有`help`可以在这里定义，因为命令列表是私有的：

```go
func init() {
    Register(Base{Name: "help", Help: "...", Action: helpAction})
    Register(Base{Name: "exit", Help: "...", Action: exitAction})
}

func helpAction(in io.Reader, w io.Writer, args ...string) bool {
    fmt.Fprintln(w, "Available commands:")
    for _, c := range commands {
        n := c.GetName()
        fmt.Fprintf(w, " - %-15s %s\n", n, c.GetHelp())
    }
    return false
}

func exitAction(in io.Reader, w io.Writer, args ...string) bool {
    fmt.Fprintf(w, "Goodbye! :)\n")
    return true
}
```

这种方法将允许用户使用`commandBase`结构来创建一个简单的命令，或者嵌入它或使用自定义结构，如果他们的命令需要它（比如带有状态的命令）：

```go
// Embedded unnamed field (inherits method)
type MyCmd struct {
    Base
    MyField string
}

// custom implementation
type MyImpl struct{}

func (MyImpl) GetName() string { return "myimpl" }
func (MyImpl) GetHelp() string { return "help string"}
func (MyImpl) Run(input io.Reader, output io.Writer, args ...string) bool {
    // do something
    return true
}
```

`MyCmd`结构和`MyImpl`结构之间的区别在于一个可以用作另一个命令的装饰器，而第二个是不同的实现，因此它不能与另一个命令交互。

# 带状态的命令

到目前为止，我们已经创建了没有内部状态的命令。但是有些命令可以保持内部状态并相应地改变其行为。状态可以限制在会话本身，也可以跨多个会话共享。最明显的例子是终端中的命令历史，其中执行的所有命令都被存储并在会话之间保留。

# 易失状态

最容易实现的是一个不持久的状态，当应用程序退出时会丢失。我们所需要做的就是创建一个自定义数据结构，托管状态并满足命令接口。方法将属于类型的指针，否则它们将无法修改数据。

在下面的示例中，我们将创建一个非常基本的内存存储，它作为一个堆栈（先进后出）与参数一起工作。让我们从推送和弹出功能开始：

```go
type Stack struct {
    data []string
}

func (s *Stack) push(values ...string) {
    s.data = append(s.data, values...)
}

func (s *Stack) pop() (string, bool) {
    if len(s.data) == 0 {
        return "", false
    }
    v := s.data[len(s.data)-1]
    s.data = s.data[:len(s.data)-1]
    return v, true
}
```

堆栈中存储的字符串表示命令的状态。现在，我们需要实现命令接口的方法——我们可以从最简单的开始：

```go
func (s *Stack) GetName() string {
    return "stack"
}

func (s *Stack) GetHelp() string {
    return "a stack-like memory storage"
}
```

现在我们需要决定它在内部是如何工作的。将有两个子命令：

+   `push`，后跟一个或多个参数，将推送到堆栈。

+   `pop`将取出堆栈的顶部元素，不需要任何参数。

让我们定义一个辅助方法`isValid`，检查参数是否有效：

```go
func (s *Stack) isValid(cmd string, args []string) bool {
    switch cmd {
    case "pop":
        return len(args) == 0
    case "push":
        return len(args) > 0
    default:
        return false
    }
}
```

现在，我们可以实现命令执行方法，它将使用有效性检查。如果通过了这一点，它将执行所选的命令或显示帮助消息：

```go
func (s *Stack) Run(r io.Reader, w io.Writer, args ...string) (exit bool) {
    if l := len(args); l < 2 || !s.isValid(args[1], args[2:]) {
        fmt.Fprintf(w, "Use `stack push <something>` or `stack pop`\n")
        return false
    }
    if args[1] == "push" {
        s.push(args[2:]...)
        return false
    }
    if v, ok := s.pop(); !ok {
        fmt.Fprintf(w, "Empty!\n")
    } else {
        fmt.Fprintf(w, "Got: `%s`\n", v)
    }
    return false
}
```

# 持久状态

下一步是在会话之间持久化状态，这需要在应用程序启动时执行一些操作，并在应用程序结束时执行另一些操作。这些新行为可以与命令接口的一些更改集成：

```go
type Command interface {
    Startup() error
    Shutdown() error
    GetName() string
    GetHelp() string
    Run(r io.Reader, w io.Writer, args ...string) (exit bool)
}
```

`Startup()`方法负责在应用程序启动时加载状态，`Shutdown()`方法需要在`exit`之前将当前状态保存到磁盘。我们可以使用这些方法更新`Base`结构；但是，这不会做任何事情，因为没有状态：

```go
// Startup does nothing
func (b Base) Startup() error { return nil }

// Shutdown does nothing
func (b Base) Shutdown() error { return nil }
```

命令列表没有被导出；它是未导出的变量`commands`。我们可以添加两个函数，这些函数将与这样一个列表进行交互，并确保我们在所有可用的命令上执行这些方法，`Startup`和`Shutdown`：

```go
// Shutdown executes shutdown for all commands
func Shutdown(w io.Writer) {
    for _, c := range commands {
        if err := c.Shutdown(); err != nil {
            fmt.Fprintf(w, "%s: shutdown error: %s", c.GetName(), err)
        }
    }
}

// Startup executes Startup for all commands
func Startup(w io.Writer) {
    for _, c := range commands {
        if err := c.Startup(); err != nil {
            fmt.Fprintf(w, "%s: startup error: %s", c.GetName(), err)
        }
    }
}
```

最后一步是在主循环开始之前在主应用程序中使用这些函数：

```go
func main() {
    s, w, a, b := bufio.NewScanner(os.Stdin), os.Stdout, args{}, bytes.Buffer{}
    command.Startup(w)
    defer command.Shutdown(w) // this is executed before returning
    fmt.Fprint(w, "** Welcome to PseudoTerm! **\nPlease enter a command.\n")
    for {
        // main loop
    }
}
```

# 升级 Stack 命令

我们希望之前定义的`Stack`命令能够在会话之间保存其状态。最简单的解决方案是将堆栈的内容保存为文本文件，每行一个元素。我们可以使用 OS/user 包将此文件对每个用户设置为唯一，并将其放置在用户的`home`目录中：

```go
func (s *Stack) getPath() (string, error) {
    u, err := user.Current()
    if err != nil {
        return "", err
    }
    return filepath.Join(u.HomeDir, ".stack"), nil
}
```

让我们开始写作；我们将创建并截断文件（使用`TRUNC`标志将其大小设置为`0`），并写入以下行：

```go
func (s *Stack) Shutdown(w io.Writer) error {
    path, err := s.getPath()
    if err != nil {
        return err
    }
    f, err := os.OpenFile(path, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0600)
    if err != nil {
        return err
    }
    defer f.Close()
    for _, v := range s.data {
        if _, err := fmt.Fprintln(f, v); err != nil {
            return err
        }
    }
    return nil
}
```

在关闭期间使用的方法将逐行读取文件，并将元素添加到堆栈中。我们可以使用`bufio.Scanner`，就像我们在之前的章节中看到的那样，轻松地做到这一点：

```go
func (s *Stack) Startup(w io.Writer) error {
    path, err := s.getPath()
    if err != nil {
        return err
    }
    f, err := os.Open(path)
    if err != nil {
        if os.IsNotExist(err) {
            return nil
        }
        return err
    }
    defer f.Close()
    s.data = s.data[:0]
    scanner := bufio.NewScanner(f)
    for scanner.Scan() {
        s.push(string(scanner.Bytes()))
    }
    return nil
}
```

# 总结

在本章中，我们通过一些术语，以便理解为什么现代终端应用程序存在以及它们是如何发展的。

然后，我们专注于如何实现基本的伪终端。第一步是创建一个处理输入管理的循环，然后需要创建一个命令选择器，最后是一个执行器。选择器可以在包中选择一系列定义的函数，并且我们创建了一个特殊的命令来退出应用程序。通过一些重构，我们从函数转变为包含名称和操作的结构体。

我们看到了如何以各种方式改进应用程序。首先，我们创建了对多行输入的支持（使用自定义的分割函数来支持带引号的字符串，以及换行符）。然后，我们创建了一些工具来为我们的函数添加有色输出，并在之前定义的某个命令中使用它们。当用户指定一个不存在的命令时，我们还使用 Levenshtein 距离来建议类似的命令。

最后，我们将命令与主应用程序分离，并创建了一种从外部注册新命令的方式。我们使用了接口，因为这允许更好的扩展和定制，以及接口的基本实现。

在下一章中，我们将开始讨论进程属性和子进程。

# 问题

1.  什么是终端，什么是伪终端？

1.  伪终端应该能够做什么？

1.  我们使用了哪些 Go 工具来模拟终端？

1.  我的应用程序如何从标准输入获取指令？

1.  使用接口来实现命令有什么优势？

1.  Levenshtein 距离是什么？为什么在伪终端中有用？
