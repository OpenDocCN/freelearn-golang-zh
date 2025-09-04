# 第八章. 文件系统备份

有许多提供文件系统备份功能的解决方案。这些包括从像 Dropbox、Box 和 Carbonite 这样的应用程序到像苹果的 Time Machine、Seagate 或网络附加存储产品等硬件解决方案，仅举几例。大多数消费级工具提供一些关键自动功能，以及一个应用程序或网站供您管理您的策略和内容。通常，特别是对于开发者来说，这些工具并不完全符合我们的需求。然而，多亏了 Go 的标准库（包括 `ioutil` 和 `os` 等包），我们拥有了构建一个完全符合我们需求的备份解决方案所需的一切。

对于我们的下一个项目，我们将为我们的源代码项目构建一个简单的文件系统备份，它将归档指定的文件夹，并在我们每次进行更改时保存它们的快照。更改可能是当我们调整文件并保存它时，当我们添加新文件和文件夹时，甚至当我们删除一个文件时。我们希望能够回到任何时间点去检索旧文件。

具体来说，在本章中，你将学习以下主题：

+   如何构建由包和命令行工具组成的项目的结构

+   在工具执行之间持久化简单数据的一种实用方法

+   `os` 包如何允许你与文件系统交互

+   如何在尊重 *Ctrl + C* 的情况下运行无限定时循环的代码

+   如何使用 `filepath.Walk` 遍历文件和文件夹

+   如何快速确定目录内容是否已更改

+   如何使用 `archive/zip` 包来压缩文件

+   如何构建关注命令行标志和常规参数组合的工具

# 解决方案设计

我们将首先列出我们解决方案的一些高级验收标准和我们想要采取的方法：

+   解决方案应在我们更改源代码项目时定期创建我们文件的快照

+   我们希望控制检查目录更改的间隔

+   代码项目主要是基于文本的，所以将目录压缩以生成存档将节省大量空间

+   我们将快速构建这个项目，同时密切关注我们可能希望在以后进行改进的地方

+   我们做出的任何实现决策都应易于修改，如果我们决定在未来更改我们的实现

+   我们将构建两个命令行工具：一个执行工作的后端守护程序和一个用户交互实用程序，它将允许我们列出、添加和从备份服务中删除路径

## 项目结构

在 Go 解决方案中，在单个项目中既有允许其他 Go 程序员使用你的功能的包，又有允许最终用户使用你的程序的命令行工具是很常见的。

正如我们在上一章中看到的，一个用于结构化此类项目的约定正在出现，即我们将包放在主项目文件夹中，将命令行工具放在名为 `cmd` 或 `cmds` 的子文件夹中（如果你有多个命令）。由于在 Go 中所有包都是平等的（无论目录树如何），你可以从命令子包中导入包，知道你永远不需要从项目包中导入命令（这是非法的，因为你不能有循环依赖）。这看起来可能像是一个不必要的抽象，但实际上这是一个相当常见的模式，可以在标准 Go 工具链的示例中看到，例如 `gofmt` 和 `goimports`。

例如，对于我们的项目，我们将编写一个名为 `backup` 的包和两个命令行工具：守护程序和用户交互工具。我们将以以下方式组织我们的项目：

```go
/backup - package 
/backup/cmds/backup - user interaction tool 
/backup/cmds/backupd - worker daemon 

```

### 小贴士

我们没有直接将代码放在 `cmd` 文件夹中（即使我们只有一个命令）的原因是，当 `go install` 构建项目时，它使用文件夹的名称作为命令名称，如果所有的工具都被称为 `cmd`，那就不会很有用。

# 备份包

我们首先将编写 `backup` 包，当我们编写相关的工具时，我们将成为该包的第一个客户。该包将负责决定目录是否已更改并需要备份，以及实际执行备份过程。

## 首先考虑明显的接口

在开始一个新的 Go 程序时，早期需要考虑的一件事是是否有任何接口显得突出。我们不想过度抽象或浪费太多时间在设计一些我们知道在编码开始时会改变的东西，但这并不意味着我们不应该寻找值得提取的明显概念。如果你不确定，这是完全可以接受的；你应该使用具体类型编写代码，并在实际解决问题后重新审视潜在的抽象。

然而，由于我们的代码将归档文件，`Archiver` 接口就凸显为一个候选者。

在你的 `GOPATH/src` 文件夹内创建一个新的文件夹，命名为 `backup`，并添加以下 `archiver.go` 代码：

```go
package backup 
type Archiver interface { 
  Archive(src, dest string) error 
} 

```

`Archiver` 接口将指定一个名为 `Archive` 的方法，该方法接受源路径和目标路径，并返回一个错误。该接口的实现将负责归档源文件夹并将其存储在目标路径中。

### 注意

在一开始就定义一个接口是一个很好的方法，可以将一些概念从我们的脑海中提取出来并放入代码中；只要我们记得简单接口的力量，这个接口就可以随着我们解决方案的演变而改变。此外，记住 `io` 包中的大多数 I/O 接口只公开一个方法。

从一开始，我们就提出了这样的观点：虽然我们将实现 ZIP 文件作为我们的归档格式，但我们很容易将其稍后与另一种 `Archiver` 格式交换。

## 通过实现接口来测试接口

现在我们有了 `Archiver` 类型的接口，我们将实现一个使用 ZIP 文件格式的类型。

将以下 `struct` 定义添加到 `archiver.go`：

```go
type zipper struct{} 

```

我们不会导出这个类型，这可能会让你得出结论，即包外部的用户无法使用它。实际上，我们将为他们提供一个类型的实例，让他们可以使用，以避免他们必须担心创建和管理自己的类型。

添加以下导出实现：

```go
// Zip is an Archiver that zips and unzips files. 
var ZIP Archiver = (*zipper)(nil) 

ZIP of type Archiver, so from outside the package, it's pretty clear that we can use that variable wherever Archiver is needed if you want to zip things. Then, we assign it with nil cast to the type *zipper. We know that nil takes no memory, but since it's cast to a zipper pointer, and given that our zipper struct has no state, it's an appropriate way of solving a problem, which hides the complexity of code (and indeed the actual implementation) from outside users. There is no reason anybody outside of the package needs to know about our zipper type at all, which frees us up to change the internals without touching the externals at any time: the true power of interfaces.
```

这个技巧的另一个实用的副作用是，编译器现在将检查我们的 `zipper` 类型是否正确实现了 `Archiver` 接口，所以如果你尝试构建这段代码，你会得到编译器错误：

```go
./archiver.go:10: cannot use (*zipper)(nil) (type *zipper) as type 
    Archiver in assignment:
 *zipper does not implement Archiver (missing Archive method)

```

我们看到我们的 `zipper` 类型没有实现接口中规定的 `Archive` 方法。

### 提示

你也可以在测试代码中使用 `Archive` 方法来确保你的类型实现了它们应该实现的接口。如果你不需要使用这个变量，你可以总是用一个下划线来丢弃它，你仍然会得到编译器的帮助：

`var _ Interface = (*Implementation)(nil)`

为了让编译器满意，我们将为我们的 `zipper` 类型添加 `Archive` 方法的实现。

将以下代码添加到 `archiver.go`：

```go
func (z *zipper) Archive(src, dest string) error { 
  if err := os.MkdirAll(filepath.Dir(dest), 0777); err != nil { 
    return err 
  } 
  out, err := os.Create(dest) 
  if err != nil { 
    return err 
  } 
  defer out.Close() 
  w := zip.NewWriter(out) 
  defer w.Close() 
  return filepath.Walk(src, func(path string, info os.FileInfo, err error) 
  error { 
    if info.IsDir() { 
      return nil // skip 
    } 
    if err != nil { 
      return err 
    } 
    in, err := os.Open(path) 
    if err != nil { 
      return err 
    } 
    defer in.Close() 
    f, err := w.Create(path) 
    if err != nil { 
      return err 
    } 
    _, err = io.Copy(f, in) 
    if err != nil { 
      return err 
    } 
    return nil 
  }) 
} 

```

你还必须从 Go 标准库中导入 `archive/zip` 包。在我们的 `Archive` 方法中，我们采取以下步骤来准备写入 ZIP 文件：

+   使用 `os.MkdirAll` 确保目标目录存在。`0777` 代码表示你可能需要创建任何缺失目录的文件权限

+   使用 `os.Create` 创建一个新的文件，该文件由 `dest` 路径指定

+   如果文件创建没有错误，使用 `defer out.Close()` 延迟关闭文件

+   使用 `zip.NewWriter` 创建一个新的 `zip.Writer` 类型，该类型将写入我们刚刚创建的文件，并延迟关闭写入器

一旦我们有了准备好的 `zip.Writer` 类型，我们使用 `filepath.Walk` 函数遍历源目录 `src`。

`filepath.Walk` 函数接受两个参数：根路径和一个回调函数，该函数将在遍历文件系统时对每个遇到的项（文件和文件夹）进行调用。

### 提示

函数是 Go 中的第一类类型，这意味着你可以将它们用作参数类型，以及全局函数和方法。`filepath.Walk` 函数指定第二个参数类型为 `filepath.WalkFunc`，这是一个具有特定签名的函数。只要我们遵守签名（正确的输入和返回参数），我们就可以编写内联函数，而无需担心 `filepath.WalkFunc` 类型。

快速查看 Go 源代码告诉我们 `filepath.WalkFunc` 的签名与我们在 `func(path string, info os.FileInfo, err error) error` 中传递的函数匹配

`filepath.Walk`函数是递归的，所以它也会深入到子文件夹中。回调函数本身接受三个参数：文件的完整路径、描述文件或文件夹本身的`os.FileInfo`对象，以及一个错误（如果发生错误，它也会返回一个错误）。如果回调函数的任何调用返回错误（除了特殊的`SkipDir`错误值），则操作将被终止，`filepath.Walk`返回该错误。我们只是将这个错误传递给`Archive`的调用者，让他们来处理，因为我们已经无能为力了。

对于树中的每个项目，我们的代码采取以下步骤：

+   如果`info.IsDir`方法告诉我们该项是一个文件夹，我们就返回`nil`，实际上跳过了它。没有理由将文件夹添加到 ZIP 存档中，因为文件的路径会为我们编码这些信息。

+   如果传递了错误（通过第三个参数），这意味着在尝试访问文件信息时出现了问题。这种情况很少见，所以我们只是返回错误，该错误最终会被传递给`Archive`的调用者。作为`filepath.Walk`的实现者，你在这里不需要强制终止操作；你可以自由地做你自己的情况中合理的事情。

+   使用`os.Open`打开源文件进行读取，如果成功，则延迟其关闭。

+   在`ZipWriter`对象上调用`Create`方法，表示我们想要创建一个新的压缩文件，并给出文件的完整路径，包括它嵌套在内的目录。

+   使用`io.Copy`读取源文件的所有字节，并通过`ZipWriter`对象将它们写入我们之前打开的 ZIP 文件。

+   返回`nil`以表示没有错误。

本章不会涵盖单元测试或**测试驱动开发**（**TDD**）实践，但你可以自由编写测试来确保我们的实现确实做了它应该做的事情。

### 小贴士

由于我们正在编写一个包，花些时间注释一下到目前为止导出的部分。你可以使用`golint`来帮助你找到可能遗漏的任何内容。

## 文件系统是否已更改？

我们备份系统最大的问题之一是决定一个文件夹是否在跨平台、可预测和可靠的方式下发生了变化。毕竟，如果没有与上一次备份不同的内容，创建备份就没有意义。当我们思考这个问题时，会想到以下几点：我们是否只需检查顶级文件夹的最后修改日期？我们是否应该使用系统通知来告知我们关心的文件何时发生变化？这两种方法都存在问题，而且事实证明，这并不是一个简单的问题。

### 小贴士

查看位于 [`fsnotify.org`](https://fsnotify.org)（项目源：[`github.com/fsnotify`](https://github.com/fsnotify)）的 `fsnotify` 项目。作者们正在尝试构建一个跨平台的包，用于订阅文件系统事件。在撰写本文时，该项目仍处于起步阶段，并且不是本章的可行选项，但将来，它可能会成为文件系统事件的行业标准解决方案。

我们将生成一个由我们在考虑是否发生变化时关心的所有信息组成的 MD5 哈希值。

查看 `os.FileInfo` 类型，我们可以看到我们可以了解关于文件或文件夹的大量信息：

```go
type FileInfo interface { 
  Name() string       // base name of the file 
  Size() int64        // length in bytes for regular files;  
                         system-dependent for others 
  Mode() FileMode     // file mode bits 
  ModTime() time.Time // modification time 
  IsDir() bool        // abbreviation for Mode().IsDir() 
  Sys() interface{}   // underlying data source (can return nil) 
} 

```

为了确保我们了解文件夹中任何文件的多种更改，哈希值将由文件名和路径（如果它们重命名了一个文件，哈希值将不同）组成，大小（如果文件更改大小，它显然不同），最后修改日期，项目是文件还是文件夹，以及文件模式位。即使我们不会存档文件夹，我们仍然关心它们的名称和文件夹的树状结构。

创建一个名为 `dirhash.go` 的新文件，并添加以下函数：

```go
package backup 
import ( 
  "crypto/md5" 
  "fmt" 
  "io" 
  "os" 
  "path/filepath" 
) 
func DirHash(path string) (string, error) { 
  hash := md5.New() 
  err := filepath.Walk(path, func(path string, info os.FileInfo, err error) 
  error { 
    if err != nil { 
      return err 
    } 
    io.WriteString(hash, path) 
    fmt.Fprintf(hash, "%v", info.IsDir()) 
    fmt.Fprintf(hash, "%v", info.ModTime()) 
    fmt.Fprintf(hash, "%v", info.Mode()) 
    fmt.Fprintf(hash, "%v", info.Name()) 
    fmt.Fprintf(hash, "%v", info.Size()) 
    return nil 
  }) 
  if err != nil { 
    return "", err 
  } 
  return fmt.Sprintf("%x", hash.Sum(nil)), nil 
} 

```

我们首先创建一个新的 `hash.Hash` 函数，该函数知道如何计算 MD5 值，然后再使用 `filepath.Walk` 再次遍历指定路径目录内的所有文件和文件夹。对于每个项目，假设没有错误发生，我们使用 `io.WriteString` 将差异信息写入哈希生成器，这允许我们将字符串写入 `io.Writer`，同时使用 `fmt.Fprintf` 执行相同的操作，但同时也暴露了格式化功能，使我们能够使用 `%v` 格式说明符为每个项目生成默认值格式。

一旦每个文件都已被处理，并且假设没有错误发生，我们随后使用 `fmt.Sprintf` 生成结果字符串。`hash.Hash` 中的 `Sum` 方法通过附加指定的值来计算最终的哈希值。在我们的情况下，我们不想附加任何内容，因为我们已经添加了我们关心的所有信息，所以我们只传递 `nil`。`%x` 格式说明符表示我们希望值以十六进制（基数为 16）和小写字母的形式表示。这是表示 MD5 哈希的常用方式。

## 检查更改并启动备份

现在我们有了对文件夹进行哈希处理和执行备份的能力，我们将这两个功能结合在一个新的类型 `Monitor` 中。`Monitor` 类型将包含路径及其相关哈希值的映射，对任何 `Archiver` 类型（当然，我们现在将使用 `backup.ZIP`）的引用，以及表示存档位置的字符串。

创建一个名为 `monitor.go` 的新文件，并添加以下定义：

```go
type Monitor struct { 
  Paths       map[string]string 
  Archiver    Archiver 
  Destination string 
} 

```

为了触发更改检查，我们将添加以下 `Now` 方法：

```go
func (m *Monitor) Now() (int, error) { 
  var counter int 
  for path, lastHash := range m.Paths { 
    newHash, err := DirHash(path) 
    if err != nil { 
      return counter, err 
    } 
    if newHash != lastHash { 
      err := m.act(path) 
      if err != nil { 
        return counter, err 
      } 
      m.Paths[path] = newHash // update the hash 
      counter++ 
    } 
  } 
  return counter, nil 
} 

```

`Now`方法遍历映射中的每个路径，并生成该文件夹的最新哈希值。如果哈希值与映射中上一次检查生成的哈希值不匹配，则认为它已更改，需要再次备份。我们在调用尚未编写的`act`方法之前这样做，然后使用这个新哈希值更新映射中的哈希值。

为了给用户一个关于他们调用`Now`时发生了什么的概述，我们还在维护一个计数器，每次我们备份一个文件夹时都会增加这个计数器。我们将在稍后使用这个计数器来让我们的最终用户了解系统正在做什么，而不会用信息轰炸他们：

```go
m.act undefined (type *Monitor has no field or method act) 

```

编译器再次帮助我们并提醒我们尚未添加`act`方法：

```go
func (m *Monitor) act(path string) error { 
  dirname := filepath.Base(path) 
  filename := fmt.Sprintf("%d.zip", time.Now().UnixNano()) 
  return m.Archiver.Archive(path, filepath.Join(m.Destination,  dirname, filename)) 
} 

```

由于我们在 ZIP `Archiver`类型中已经完成了繁重的工作，我们在这里只需要生成一个文件名，决定归档将放在哪里，然后调用`Archive`方法。

### 小贴士

如果`Archive`方法返回错误，`act`方法和`Now`方法将分别返回它。这种将错误向上传递的机制在 Go 中非常常见，允许你处理可以采取一些有用措施来恢复的情况，或者将问题推迟给其他人。

前面代码中的`act`方法使用`time.Now().UnixNano()`生成时间戳文件名，并硬编码了`.zip`扩展名。

### 硬编码在短时间内是可以接受的

就像我们现在这样硬编码文件扩展名在开始时是可以接受的，但如果你这么想，我们在某种程度上混合了关注点。如果我们更改`Archiver`实现以使用 RAR 或我们制作的压缩格式，`.zip`扩展名就不再合适了。

### 小贴士

在继续阅读之前，思考一下你可能会采取哪些步骤来避免这种硬编码。文件名扩展的决定在哪里？你需要做出哪些更改才能避免硬编码？

文件名扩展的决定可能应该在`Archiver`接口中，因为它知道将要进行的归档类型。因此，我们可以添加一个`Ext()`字符串方法，并从我们的`act`方法中访问它。但我们可以通过允许`Archiver`作者指定整个文件名格式而不是仅指定扩展名来添加一些额外的功能，而无需做太多额外的工作。

在`archiver.go`中，更新`Archiver`接口定义：

```go
type Archiver interface { 
  DestFmt() string 
  Archive(src, dest string) error 
} 

```

我们的`zipper`类型现在需要实现以下功能：

```go
func (z *zipper) DestFmt() string { 
  return "%d.zip" 
} 

```

现在我们可以要求`act`方法从`Archiver`接口获取整个格式字符串，更新`act`方法：

```go
func (m *Monitor) act(path string) error { 
  dirname := filepath.Base(path) 
  filename := fmt.Sprintf(m.Archiver.DestFmt(), time.Now().UnixNano()) 
  return m.Archiver.Archive(path, filepath.Join(m.Destination, dirname, 
  filename)) 
} 

```

# 用户命令行工具

我们将构建的两个工具中的第一个允许用户为备份守护程序工具（我们稍后将编写）添加、列出和删除路径。你可以公开一个 Web 界面，甚至使用桌面用户界面集成的绑定包，但我们将保持简单，并构建自己的命令行工具。

在 `backup` 文件夹内创建一个名为 `cmds` 的新文件夹，并在其中创建另一个 `backup` 文件夹，这样你就有 `backup/cmds/backup`。

在我们的新 `backup` 文件夹内，将以下代码添加到 `main.go` 文件中：

```go
func main() { 
  var fatalErr error 
  defer func() { 
    if fatalErr != nil { 
      flag.PrintDefaults() 
      log.Fatalln(fatalErr) 
    } 
  }() 
  var ( 
    dbpath = flag.String("db", "./backupdata", "path to database directory") 
  ) 
  flag.Parse() 
  args := flag.Args() 
  if len(args) < 1 { 
    fatalErr = errors.New("invalid usage; must specify command") 
    return 
  } 
} 

```

我们首先定义了 `fatalErr` 变量，并延迟执行检查以确保该值是 `nil`。如果不是，它将打印错误信息以及标志默认值，并以非零状态码退出。然后我们定义了一个名为 `db` 的标志，它期望在解析标志和获取剩余参数以及确保至少有一个参数之前，提供 `filedb` 数据库目录的路径。

## 持久化小数据

为了跟踪我们生成的路径和哈希值，我们需要一种数据存储机制，理想情况下即使在停止和启动程序时也能正常工作。这里有各种各样的选择：从文本文件到完整的水平可扩展数据库解决方案。Go 的简洁性原则告诉我们，在我们的小型备份程序中内置数据库依赖并不是一个好主意；相反，我们应该考虑以最简单的方式解决这个问题。

`github.com/matryer/filedb` 包正是针对这类问题的一个实验性解决方案。它允许你以非常简单、无模式的数据库方式与文件系统交互。其设计灵感来源于 `mgo` 等包，并且适用于数据查询需求非常简单的情况。在 `filedb` 中，数据库是一个文件夹，而集合是一个文件，其中每一行代表一个不同的记录。当然，随着 `filedb` 项目的不断发展，这一切都可能发生变化，但希望接口不会改变。

### 注意

在 Go 项目中添加此类依赖项应非常谨慎，因为随着时间的推移，依赖项可能会过时、超出其初始范围或在某些情况下完全消失。虽然听起来有些反直觉，但你应该考虑将几个文件复制并粘贴到你的项目中是否比依赖外部依赖项更好。或者，考虑通过将整个包复制到命令的 `vendor` 文件夹中来维护依赖项。这类似于存储依赖项的快照，你知道它对你的工具是有效的。

将以下代码添加到 `main` 函数的末尾：

```go
db, err := filedb.Dial(*dbpath) 
if err != nil { 
  fatalErr = err 
  return 
} 
defer db.Close() 
col, err := db.C("paths") 
if err != nil { 
  fatalErr = err 
  return 
} 

```

在这里，我们使用 `filedb.Dial` 函数连接到 `filedb` 数据库。实际上，这里并没有发生太多事情，除了指定数据库的位置，因为没有真正的数据库服务器需要连接（尽管这可能在将来发生变化，这就是为什么接口中存在这样的规定）。如果连接成功，我们延迟关闭数据库。关闭数据库实际上会做一些事情，因为可能还有需要清理的打开的文件。

按照与 `mgo` 的模式，接下来我们使用 `C` 方法指定一个集合，并将其引用保存在 `col` 变量中。如果在任何点上发生错误，我们将它分配给 `fatalErr` 变量并返回。

要存储数据，我们将定义一个名为`path`的类型，该类型将存储完整路径和最后一个哈希值，并使用 JSON 编码将此存储在我们的`filedb`数据库中。在`main`函数上方添加以下`struct`定义：

```go
type path struct { 
  Path string 
  Hash string 
} 

```

## 解析参数

当我们调用`flag.Args`（而不是`os.Args`）时，我们接收一个不包括标志的参数切片。这允许我们在同一工具中混合标志参数和非标志参数。

我们希望我们的工具能够以下方式使用：

+   要添加路径：

```go
backup -db=/path/to/db add {path} [paths...]

```

+   要删除路径：

```go
backup -db=/path/to/db remove {path} [paths...]

```

+   要列出所有路径：

```go
backup -db=/path/to/db list

```

要实现这一点，因为我们已经处理了标志，所以我们必须检查第一个（非标志）参数。

在`main`函数中添加以下代码：

```go
switch strings.ToLower(args[0]) { 
case "list": 
case "add": 
case "remove": 
} 

```

在这里，我们简单地根据设置成小写的第一个参数进行切换（如果用户输入`backup LIST`，我们仍然希望它能够工作）。

### 列出路径

要列出数据库中的路径，我们将使用路径的`col`变量的`ForEach`方法。在`list`情况中添加以下代码：

```go
var path path 
col.ForEach(func(i int, data []byte) bool { 
  err := json.Unmarshal(data, &path) 
  if err != nil { 
    fatalErr = err 
    return true 
  } 
  fmt.Printf("= %s\n", path) 
  return false 
}) 

```

我们向`ForEach`传递一个回调函数，该函数将为该集合中的每个项目被调用。然后我们将其从 JSON 反序列化为我们的`path`类型，并使用`fmt.Printf`打印出来。根据`filedb`接口，我们返回`false`，这告诉我们返回`true`将停止迭代，而我们想确保我们列出所有路径。

#### 为您自己的类型提供字符串表示

如果你以这种方式在 Go 中打印结构体，使用`%s`格式说明符，你可能会得到一些混乱的结果，这些结果难以阅读。然而，如果类型实现了`String()`字符串方法，它将被使用，我们可以利用这个方法来控制打印的内容。在路径结构体下方添加以下方法：

```go
func (p path) String() string { 
  return fmt.Sprintf("%s [%s]", p.Path, p.Hash) 
} 

```

这告诉`path`类型如何将其自身表示为字符串。

### 添加路径

要添加路径或多个路径，我们将遍历剩余的参数并调用每个的`InsertJSON`方法。在`add`情况中添加以下代码：

```go
if len(args[1:]) == 0 { 
  fatalErr = errors.New("must specify path to add") 
  return 
} 
for _, p := range args[1:] { 
  path := &path{Path: p, Hash: "Not yet archived"} 
  if err := col.InsertJSON(path); err != nil { 
    fatalErr = err 
    return 
  } 
  fmt.Printf("+ %s\n", path) 
} 

```

如果用户没有指定任何其他参数，例如，如果他们只是调用`backup add`而没有输入任何路径，我们将返回一个致命错误。否则，我们完成工作并打印出路径字符串（以`+`符号为前缀），以指示它已成功添加。默认情况下，我们将哈希设置为`尚未存档`字符串字面量，这是一个无效的哈希，但同时也让用户知道它尚未存档，并通知我们的代码（因为文件夹的哈希永远不会等于该字符串）。

### 删除路径

要删除路径或多个路径，我们使用路径集合的`RemoveEach`方法。在`remove`情况中添加以下代码：

```go
var path path 
col.RemoveEach(func(i int, data []byte) (bool, bool) { 
  err := json.Unmarshal(data, &path) 
  if err != nil { 
    fatalErr = err 
    return false, true 
  } 
  for _, p := range args[1:] { 
    if path.Path == p { 
      fmt.Printf("- %s\n", path) 
      return true, false 
    } 
  } 
  return false, false 
}) 

```

我们提供给`RemoveEach`的回调函数期望我们返回两个 bool 类型：第一个指示项目是否应该被删除，第二个指示我们是否应该停止迭代。

## 使用我们的新工具

我们已经完成了简单的 `backup` 命令行工具。让我们看看它的实际运行情况。在 `backup/cmds/backup` 内创建一个名为 `backupdata` 的文件夹；这将成为 `filedb` 数据库。

在终端中通过导航到 `main.go` 文件并运行以下命令来构建工具：

```go
go build -o backup

```

如果一切顺利，我们现在可以添加一个路径：

```go
./backup -db=./backupdata add ./test ./test2

```

你应该看到预期的输出：

```go
+ ./test [Not yet archived]
+ ./test2 [Not yet archived]

```

现在让我们添加另一个路径：

```go
./backup -db=./backupdata add ./test3

```

你现在应该看到完整的列表：

```go
./backup -db=./backupdata list

```

我们的程序应该产生以下结果：

```go
= ./test [Not yet archived]
= ./test2 [Not yet archived]
= ./test3 [Not yet archived]

```

让我们移除 `test3` 以确保 `remove` 功能正常工作：

```go
./backup -db=./backupdata remove ./test3
./backup -db=./backupdata list

```

这将带我们回到这里：

```go
+ ./test [Not yet archived]
+ ./test2 [Not yet archived]

```

现在，我们能够以一种对我们用例有意义的方式与 `filedb` 数据库进行交互。接下来，我们构建一个守护程序，它将实际使用我们的 `backup` 包来完成工作。

# 守护程序备份工具

我们将称为 `backupd` 的 `backup` 工具将负责定期检查 `filedb` 数据库中列出的路径，对文件夹进行哈希处理以查看是否有任何变化，并使用 `backup` 包实际执行需要归档的文件夹的归档工作。

在 `backup/cmds/backup` 文件夹旁边创建一个名为 `backupd` 的新文件夹，然后让我们直接进入处理致命错误和标志：

```go
func main() { 
  var fatalErr error 
  defer func() { 
    if fatalErr != nil { 
      log.Fatalln(fatalErr) 
    } 
  }() 
  var ( 
    interval = flag.Duration("interval", 10 * time.Second, "interval between 
    checks") 
    archive  = flag.String("archive", "archive", "path to archive location") 
    dbpath   = flag.String("db", "./db", "path to filedb database") 
  ) 
  flag.Parse() 
} 

```

你现在应该非常熟悉这种代码了。在指定三个标志：`interval`、`archive` 和 `db` 之前，我们推迟处理致命错误。`interval` 标志表示检查文件夹是否发生变化之间的秒数，`archive` 标志是 ZIP 文件将要存放的归档位置的路径，而 `db` 标志是与 `backup` 命令交互的相同 `filedb` 数据库的路径。通常的 `flag.Parse` 调用设置变量并验证我们是否准备好继续。

为了检查文件夹的哈希值，我们需要一个我们之前编写的 `Monitor` 实例。将以下代码添加到 `main` 函数中：

```go
m := &backup.Monitor{ 
  Destination: *archive, 
  Archiver:    backup.ZIP, 
  Paths:       make(map[string]string), 
} 

```

在这里，我们使用 `archive` 值作为 `Destination` 类型创建 `backup.Monitor`。我们将使用 `backup.ZIP` 归档器并创建一个映射，以便它能够内部存储路径和哈希值。在守护程序启动时，我们希望从数据库中加载路径，这样它就不会在不必要的情况下进行归档，因为我们停止和启动某些操作。

将以下代码添加到 `main` 函数中：

```go
db, err := filedb.Dial(*dbpath) 
if err != nil { 
  fatalErr = err 
  return 
} 
defer db.Close() 
col, err := db.C("paths") 
if err != nil { 
  fatalErr = err 
  return 
} 

```

你之前也见过这段代码；它连接到数据库并创建一个对象，使我们能够与 `paths` 集合交互。如果任何操作失败，我们设置 `fatalErr` 并返回。

## 重复的结构

由于我们将使用与我们在用户命令行工具程序中使用的相同路径结构，因此我们需要为这个程序也包含一个定义。在 `main` 函数上方插入以下结构：

```go
type path struct { 
  Path string 
  Hash string 
} 

LastChecked field to our backupd program so that we can add rules where each folder only gets archived once an hour at most. Our backup program doesn't care about this and will chug along perfectly happy with its view into what fields constitute a path structure.
```

## 缓存数据

现在，我们可以查询所有现有的路径并更新`Paths`映射，这是一种有用的技术，可以增加程序的速度，尤其是在处理缓慢或断开的数据存储时。通过将数据加载到缓存中（在我们的例子中是`Paths`映射），我们可以在需要信息时以闪电般的速度访问它，而无需每次都咨询文件。

将以下代码添加到`main`函数的主体中：

```go
var path path 
col.ForEach(func(_ int, data []byte) bool { 
  if err := json.Unmarshal(data, &path); err != nil { 
    fatalErr = err 
    return true 
  } 
  m.Paths[path.Path] = path.Hash 
  return false // carry on 
}) 
if fatalErr != nil { 
  return 
} 
if len(m.Paths) < 1 { 
  fatalErr = errors.New("no paths - use backup tool to add at least one") 
  return 
} 

```

再次使用`ForEach`方法允许我们遍历数据库中的所有路径。我们将 JSON 字节反序列化到我们在其他程序中使用的相同的`path`结构中，并在`Paths`映射中设置值。假设一切顺利，我们进行最后的检查以确保至少有一条路径，如果没有，我们返回错误。

### 注意

我们程序的一个限制是，一旦它开始运行，它将不会动态添加路径。守护进程需要重新启动。如果你觉得这很麻烦，你总是可以构建一个机制，定期更新`Paths`映射或使用其他类型的配置管理。

## 无限循环

我们接下来需要做的事情是立即对哈希进行检查，看看在进入无限定时循环之前是否需要存档。在这个无限定时循环中，我们会以规定的、规律的间隔再次执行检查。

无限循环听起来像是一个糟糕的想法；实际上，对某些人来说，它听起来像是一个错误。然而，既然我们在这里讨论的是程序内的无限循环，并且由于无限循环可以通过简单的`break`命令轻松中断，所以它们并不像听起来那么戏剧化。当我们把无限循环与没有默认情况的`select`语句混合时，我们能够以可管理的方式运行代码，而不会在等待某事发生时消耗 CPU 周期。执行将被阻塞，直到两个通道之一接收数据。

在 Go 语言中，编写一个无限循环就像运行以下代码一样简单：

```go
for {} 

```

大括号内的指令会不断地被执行，速度取决于运行代码的机器。再次强调，除非你小心地处理你要求它执行的任务，否则这听起来像是一个糟糕的计划。在我们的例子中，我们立即在两个通道上启动一个`select`情况，它会安全地阻塞，直到其中一个通道有有趣的东西要说。

添加以下代码：

```go
check(m, col) 
signalChan := make(chan os.Signal, 1) 
signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM) 
for { 
  select { 
  case <-time.After(*interval): 
    check(m, col) 
  case <-signalChan: 
    // stop 
    fmt.Println() 
    log.Printf("Stopping...") 
    return 
  } 
} 

```

当然，作为负责任的程序员，我们关心当用户终止我们的程序时会发生什么。因此，在调用`check`方法（该方法尚未存在）之后，我们创建一个信号通道，并使用`signal.Notify`请求将终止信号发送到通道，而不是自动处理。在我们的无限`for`循环中，我们选择两种可能性：要么`timer`通道发送消息，要么终止信号通道发送消息。如果是`timer`通道的消息，我们再次调用`check`；如果是`signalChan`，我们就开始终止程序；否则，我们将循环回并等待。

`time.After`函数返回一个通道，该通道将在指定时间过后发送一个信号（实际上，是当前时间）。由于我们使用`flag.Duration`，我们可以直接将这个（通过`*`解引用）作为`time.Duration`参数传递给函数。使用`flag.Duration`还意味着用户可以用人类可读的方式指定时间长度，例如`10s`代表 10 秒或`1m`代表一分钟。

最后，我们从主函数返回，导致延迟语句执行，例如关闭数据库连接。

## 更新 filedb 记录

剩下的就是我们实现`check`函数，该函数应该调用`Monitor`类型的`Now`方法，并在有新哈希值的情况下更新数据库。

在`main`函数下方，添加以下代码：

```go
func check(m *backup.Monitor, col *filedb.C) { 
  log.Println("Checking...") 
  counter, err := m.Now() 
  if err != nil { 
    log.Fatalln("failed to backup:", err) 
  } 
  if counter > 0 { 
    log.Printf("  Archived %d directories\n", counter) 
    // update hashes 
    var path path 
    col.SelectEach(func(_ int, data []byte) (bool, []byte, bool) { 
      if err := json.Unmarshal(data, &path); err != nil { 
        log.Println("failed to unmarshal data (skipping):", err) 
        return true, data, false 
      } 
      path.Hash, _ = m.Paths[path.Path] 
      newdata, err := json.Marshal(&path) 
      if err != nil { 
        log.Println("failed to marshal data (skipping):", err) 
        return true, data, false 
      } 
      return true, newdata, false 
    }) 
  } else { 
    log.Println("  No changes") 
  } 
} 

```

`check`函数首先告诉用户正在执行检查，然后立即调用`Now`。如果`Monitor`类型为我们做了任何工作，即询问是否存档了任何文件，我们将它们输出给用户，并继续用新值更新数据库。`SelectEach`方法允许我们通过返回替换字节来更改集合中的每个记录。因此，我们解包字节以获取路径结构，更新哈希值，并返回打包的字节。这确保了下次我们启动`backupd`进程时，它将使用正确的哈希值。

# 测试我们的解决方案

让我们看看我们的两个程序是否能够很好地一起工作。你可能需要为这个打开两个终端窗口，因为我们将会运行两个程序。

我们已经将一些路径添加到数据库中，所以让我们使用`backup`来看看：

```go
./backup -db="./backupdata" list

```

你应该看到两个测试文件夹；如果没有，请参考*添加路径*部分：

```go
= ./test [Not yet archived]
= ./test2 [Not yet archived]

```

在另一个窗口中，导航到`backupd`文件夹，并创建我们的两个测试文件夹，分别命名为`test`和`test2`。

使用通常的方法构建`backupd`：

```go
go build -o backupd

```

假设一切顺利，我们现在可以开始备份过程，确保将`db`路径指向与`backup`程序相同的路径，并指定我们想要使用一个名为`archive`的新文件夹来存储 ZIP 文件。为了测试目的，让我们指定一个`5`秒的间隔以节省时间：

```go
./backupd -db="../backup/backupdata/" -archive="./archive" - interval=5s

```

立即，`backupd`应该检查文件夹，计算哈希值，并注意它们是不同的（与`尚未存档`不同），并为两个文件夹启动存档过程。它将打印出以下输出：

```go
Checking...
Archived 2 directories

```

打开在`backup/cmds/backupd`中新建的`archive`文件夹，并注意它创建了两个子文件夹：`test`和`test2`。在这些文件夹内是空文件夹的压缩存档版本。你可以随意解压一个看看；到目前为止，没有什么特别激动人心的。

同时，回到终端窗口，`backupd`正在再次检查文件夹是否有变化：

```go
Checking...
 No changes
Checking...
 No changes

```

在你最喜欢的文本编辑器中，在`test2`文件夹内创建一个新的文本文件，包含单词`test`，并将其保存为`one.txt`。几秒钟后，你会看到`backupd`已经注意到了新文件，并在`archive/test2`文件夹内创建了一个新的快照。

当然，由于时间不同，它有一个不同的文件名，但如果你解压它，你会注意到它确实创建了一个文件夹的压缩存档版本。

通过以下操作尝试解决方案：

+   修改`one.txt`文件的内容

+   将一个文件添加到`test`文件夹中

+   删除一个文件

# 摘要

在本章中，我们成功地为你的代码项目构建了一个非常简单的备份系统。你可以看到扩展或修改这些程序的行为是多么简单。你可以继续解决的问题的范围是无限的。

与我们在上一节中使用的本地存档目标文件夹不同，想象一下挂载一个网络存储设备并使用它。突然之间，你有了这些重要文件的异地（或至少非本地）备份。你可以轻松地将 Dropbox 文件夹设置为存档目标，这意味着你不仅可以访问快照，而且一份副本也存储在云端，甚至可以与其他用户共享。

将`Archiver`接口扩展以支持`Restore`操作（这将仅使用`encoding/zip`包解压文件）允许你构建可以查看存档并访问单个文件更改的工具，就像 Mac 上的 Time Machine 允许你做的那样。对文件进行索引可以让你在整个代码历史中完成完整的搜索，就像 GitHub 做的那样。

由于文件名是时间戳，你可以将备份的旧存档转移到不太活跃的存储介质，或者将更改总结成每日存档。

显然，备份软件已经存在，经过良好的测试，并且在全球范围内使用，因此专注于解决尚未解决的问题可能是一个明智的选择。但是，当编写小程序来完成这些事情只需要如此少的努力时，它通常值得去做，因为它给你带来的控制力。当你编写代码时，你可以得到你想要的确切结果，而不需要妥协，这取决于每个人自己做出决定。

具体来说，在本章中，我们探讨了 Go 的标准库如何使与文件系统的交互变得简单：打开文件进行读取、创建新文件和创建目录。`os`包与`io`包中的强大类型混合，进一步与`encoding/zip`和其他功能结合，清楚地展示了如何将 Go 接口组合起来以提供非常强大的结果。
