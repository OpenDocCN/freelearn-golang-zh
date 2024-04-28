# 第八章。文件系统备份

有许多解决方案提供文件系统备份功能。这些包括从应用程序（如 Dropbox、Box、Carbonite）到硬件解决方案（如苹果的 Time Machine、希捷或网络附加存储产品）等各种解决方案。大多数消费者工具提供一些关键的自动功能，以及一个应用程序或网站供您管理您的策略和内容。通常，特别是对于开发人员来说，这些工具并不能完全满足我们的需求。然而，由于 Go 的标准库（其中包括`ioutil`和`os`等包），我们有了构建备份解决方案所需的一切。

对于我们的最终项目，我们将为我们的源代码项目构建一个简单的文件系统备份，该备份将存档指定的文件夹并在每次更改时保存它们的快照。更改可能是当我们调整文件并保存它时，或者如果我们添加新文件和文件夹，甚至如果我们删除文件。我们希望能够回到任何时间点以检索旧文件。

具体来说，在本章中，您将学到：

+   如何构建由包和命令行工具组成的项目

+   在工具执行之间持久化简单数据的务实方法

+   `os`包如何允许您与文件系统交互

+   如何在无限定时循环中运行代码，同时尊重*Ctrl* + *C*

+   如何使用`filepath.Walk`来迭代文件和文件夹

+   如何快速确定目录的内容是否已更改

+   如何使用`archive/zip`包来压缩文件

+   如何构建关心命令行标志和普通参数组合的工具

# 解决方案设计

我们将首先列出一些高层次的解决方案验收标准以及我们想要采取的方法：

+   解决方案应该在我们对源代码项目进行更改时定期创建我们文件的快照

+   我们希望控制检查目录更改的间隔

+   代码项目主要是基于文本的，因此将目录压缩以生成存档将节省大量空间

+   我们将快速构建这个项目，同时密切关注我们可能希望以后进行改进的地方

+   如果我们决定将来更改我们的实现，我们所做的任何实现决策都应该很容易修改

+   我们将构建两个命令行工具，后台守护进程执行工作，用户交互工具让我们列出、添加和删除备份服务中的路径

## 项目结构

在 Go 解决方案中，通常在单个项目中，既有一个允许其他 Go 程序员使用您的功能的包，也有一个允许最终用户使用您的代码的命令行工具。

一种约定正在兴起，即通过在主项目文件夹中放置包，并在名为`cmd`或`cmds`的子文件夹中放置命令行工具。由于在 Go 中所有包（无论目录树如何）都是平等的，您可以从子包中导入主包，知道您永远不需要从主包中导入命令。这可能看起来像是一个不必要的抽象，但实际上是一个非常常见的模式，并且可以在标准的 Go 工具链中看到，例如`gofmt`和`goimports`。

例如，对于我们的项目，我们将编写一个名为`backup`的包，以及两个命令行工具：守护进程和用户交互工具。我们将按以下方式构建我们的项目：

```go
/backup - package
/backup/cmds/backup – user interaction tool
/backup/cmds/backupd – worker daemon
```

# 备份包

我们首先将编写`backup`包，我们将成为编写相关工具时的第一个客户。该包将负责决定目录是否已更改并需要备份，以及实际执行备份过程。

## 明显的接口？

在着手编写新的 Go 程序时，首先要考虑的是是否有任何接口吸引了你的注意。我们不希望在一开始就过度抽象或浪费太多时间设计我们知道在编码开始时会发生变化的东西，但这并不意味着我们不应该寻找值得提取的明显概念。由于我们的代码将对文件进行归档，`Archiver`接口显然是一个候选者。

在`GOPATH`中创建一个名为`backup`的新文件夹，并添加以下`archiver.go`代码：

```go
package backup

type Archiver interface {
  Archive(src, dest string) error
}
```

`Archiver`接口将指定一个名为`Archive`的方法，该方法接受源和目标路径，并返回一个错误。该接口的实现将负责对源文件夹进行归档，并将其存储在目标路径中。

### 注意

提前定义一个接口是将一些概念从我们的头脑中转移到代码中的好方法；这并不意味着随着我们解决方案的演变，这个接口就不能改变，只要我们记住简单接口的力量。还要记住，`io`包中的大多数 I/O 接口只公开一个方法。

从一开始，我们就已经说明了，虽然我们将实现 ZIP 文件作为我们的存档格式，但以后我们可以很容易地用其他类型的`Archiver`格式来替换它。

## 实现 ZIP

现在我们有了`Archiver`类型的接口，我们将实现一个使用 ZIP 文件格式的接口。

将以下`struct`定义添加到`archiver.go`：

```go
type zipper struct{}
```

我们不打算导出这种类型，这可能会让你得出结论，包外的用户将无法使用它。实际上，我们将为他们提供该类型的一个实例供他们使用，以免他们担心创建和管理自己的类型。

添加以下导出的实现：

```go
// Zip is an Archiver that zips and unzips files.
var ZIP Archiver = (*zipper)(nil)
```

这段有趣的 Go 代码实际上是一种非常有趣的方式，可以向编译器暴露意图，而不使用任何内存（确切地说是 0 字节）。我们定义了一个名为`ZIP`的变量，类型为`Archiver`，因此从包外部很清楚，我们可以在需要`Archiver`的任何地方使用该变量——如果你想要压缩文件。然后我们将其赋值为`nil`，转换为`*zipper`类型。我们知道`nil`不占用内存，但由于它被转换为`zipper`指针，并且考虑到我们的`zipper`结构没有字段，这是解决问题的一种合适方式，它隐藏了代码的复杂性（实际实现）对外部用户。包外部没有任何理由需要知道我们的`zipper`类型，这使我们可以随时更改内部而不触及外部；这就是接口的真正力量。

这个技巧的另一个方便之处是，编译器现在将检查我们的`zipper`类型是否正确实现了`Archiver`接口，如果你尝试构建这段代码，你将会得到一个编译器错误：

```go

./archiver.go:10: cannot use (*zipper)(nil) (type *zipper) as type Archiver in assignment:

 *zipper does not implement Archiver (missing Archive method)

```

我们看到我们的`zipper`类型没有实现接口中规定的`Archive`方法。

### 注意

你也可以在测试代码中使用`Archive`方法来确保你的类型实现了它们应该实现的接口。如果你不需要使用这个变量，你可以使用下划线将其丢弃，你仍然会得到编译器的帮助：

```go
var _ Interface = (*Implementation)(nil)
```

为了让编译器满意，我们将为我们的`zipper`类型添加`Archive`方法的实现。

将以下代码添加到`archiver.go`：

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
  return filepath.Walk(src, func(path string, info os.FileInfo, err error) error {
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
    io.Copy(f, in)
    return nil
  })
}
```

你还需要从 Go 标准库中导入`archive/zip`包。在我们的`Archive`方法中，我们采取以下步骤来准备写入 ZIP 文件：

+   使用`os.MkdirAll`确保目标目录存在。`0777`代码表示用于创建任何缺失目录的文件权限。

+   使用`os.Create`根据`dest`路径创建一个新文件。

+   如果文件创建没有错误，使用`defer out.Close()`延迟关闭文件。

+   使用`zip.NewWriter`创建一个新的`zip.Writer`类型，它将写入我们刚刚创建的文件，并延迟关闭写入器。

一旦我们准备好一个`zip.Writer`类型，我们使用`filepath.Walk`函数来迭代源目录`src`。

`filepath.Walk`函数接受两个参数：根路径和回调函数`func`，用于在遍历文件系统时遇到的每个项目（文件和文件夹）进行调用。`filepath.Walk`函数是递归的，因此它也会深入到子文件夹中。回调函数本身接受三个参数：文件的完整路径，描述文件或文件夹本身的`os.FileInfo`对象，以及错误（如果发生错误，它也会返回错误）。如果对回调函数的任何调用导致返回错误，则操作将被中止，并且`filepath.Walk`将返回该错误。我们只需将其传递给`Archive`的调用者，并让他们担心，因为我们无法做更多事情。

对于树中的每个项目，我们的代码采取以下步骤：

+   如果`info.IsDir`方法告诉我们该项目是一个文件夹，我们只需返回`nil`，有效地跳过它。没有理由将文件夹添加到 ZIP 存档中，因为文件的路径将为我们编码该信息。

+   如果传入错误（通过第三个参数），这意味着在尝试访问有关文件的信息时出现了问题。这是不常见的，所以我们只需返回错误，最终将其传递给`Archive`的调用者。

+   使用`os.Open`打开源文件进行读取，如果成功则延迟关闭。

+   在`ZipWriter`对象上调用`Create`，表示我们要创建一个新的压缩文件，并给出文件的完整路径，其中包括它所嵌套的目录。

+   使用`io.Copy`从源文件读取所有字节，并通过`ZipWriter`对象将它们写入我们之前打开的 ZIP 文件。

+   返回`nil`表示没有错误。

本章不涉及单元测试或**测试驱动开发**（**TDD**）实践，但请随意编写一个测试来确保我们的实现达到预期的效果。

### 提示

由于我们正在编写一个包，花一些时间注释到目前为止导出的部分。您可以使用`golint`来帮助您找到可能遗漏的任何导出部分。

## 文件系统是否发生了更改？

我们的备份系统面临的最大问题之一是如何以跨平台、可预测和可靠的方式确定文件夹是否发生了更改。当我们考虑这个问题时，有几件事情值得一提：我们应该只检查顶层文件夹的上次修改日期吗？我们应该使用系统通知来通知我们关心的文件何时发生更改吗？这两种方法都存在问题，事实证明这并不是一个微不足道的问题。

相反，我们将生成一个由我们关心的所有信息组成的 MD5 哈希，以确定某些内容是否发生了更改。

查看`os.FileInfo`类型，我们可以看到关于文件的许多信息：

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

为了确保我们能够意识到文件夹中任何文件的各种更改，哈希将由文件名和路径（因此如果它们重命名文件，哈希将不同）、大小（如果文件大小发生变化，显然是不同的）、上次修改日期、项目是文件还是文件夹以及文件模式位组成。尽管我们不会存档文件夹，但我们仍然关心它们的名称和文件夹的树结构。

创建一个名为`dirhash.go`的新文件，并添加以下函数：

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
  err := filepath.Walk(path, func(path string, info os.FileInfo, err error) error {
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

我们首先创建一个知道如何计算 MD5 的新`hash.Hash`，然后使用`filepath.Walk`来遍历指定路径目录中的所有文件和文件夹。对于每个项目，假设没有错误，我们使用`io.WriteString`将差异信息写入哈希生成器，这让我们可以将字符串写入`io.Writer`，以及`fmt.Fprintf`，它同时暴露了格式化功能，允许我们使用`%v`格式动词生成每个项目的默认值格式。

一旦每个文件都被处理，假设没有发生错误，我们就使用`fmt.Sprintf`生成结果字符串。`hash.Hash`上的`Sum`方法计算具有附加指定值的最终哈希值。在我们的情况下，我们不想附加任何东西，因为我们已经添加了所有我们关心的信息，所以我们只传递`nil`。`%x`格式动词表示我们希望该值以十六进制（基数 16）的小写字母表示。这是表示 MD5 哈希的通常方式。

## 检查更改并启动备份

现在我们有了哈希文件夹的能力，并且可以执行备份，我们将把这两者放在一个名为`Monitor`的新类型中。`Monitor`类型将具有一个路径映射及其关联的哈希值，任何`Archiver`类型的引用（当然，我们现在将使用`backup.ZIP`），以及一个表示存档位置的目标字符串。

创建一个名为`monitor.go`的新文件，并添加以下定义：

```go
type Monitor struct {
  Paths       map[string]string
  Archiver    Archiver
  Destination string
}
```

为了触发更改检查，我们将添加以下`Now`方法：

```go
func (m *Monitor) Now() (int, error) {
  var counter int
  for path, lastHash := range m.Paths {
    newHash, err := DirHash(path)
    if err != nil {
      return 0, err
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

`Now`方法遍历映射中的每个路径，并生成该文件夹的最新哈希值。如果哈希值与映射中的哈希值不匹配（上次检查时生成的哈希值），则认为它已更改，并需要再次备份。在调用尚未编写的`act`方法之前，我们会这样做，然后使用这个新的哈希值更新映射中的哈希值。

为了给我们的用户一个高层次的指示，当他们调用`Now`时发生了什么，我们还维护一个计数器，每次备份一个文件夹时我们会增加这个计数器。我们稍后将使用这个计数器来让我们的最终用户了解系统正在做什么，而不是用信息轰炸他们。

```go
m.act undefined (type *Monitor has no field or method act)
```

编译器再次帮助我们，并提醒我们还没有添加`act`方法：

```go
func (m *Monitor) act(path string) error {
  dirname := filepath.Base(path)
  filename := fmt.Sprintf("%d.zip", time.Now().UnixNano())
  return m.Archiver.Archive(path, filepath.Join(m.Destination, dirname, filename))
}
```

因为我们在我们的 ZIP `Archiver`类型中已经做了大部分工作，所以我们在这里所要做的就是生成一个文件名，决定存档的位置，并调用`Archive`方法。

### 提示

如果`Archive`方法返回一个错误，`act`方法和`Now`方法将分别返回它。在 Go 中，这种将错误传递到链条上的机制非常常见，它允许你处理你可以做一些有用的恢复的情况，或者将问题推迟给其他人。

上述代码中的`act`方法使用`time.Now().UnixNano()`生成时间戳文件名，并硬编码`.zip`扩展名。

### 硬编码在短时间内是可以的

像我们这样硬编码文件扩展名在开始时是可以的，但是如果你仔细想想，我们在这里混合了一些关注点。如果我们改变`Archiver`的实现以使用 RAR 或我们自己制作的压缩格式，`.zip`扩展名将不再合适。

### 提示

在继续阅读之前，想想你可能会采取哪些步骤来避免硬编码。文件扩展名决策在哪里？为了正确避免硬编码，你需要做哪些改变？

文件扩展名决定的正确位置可能在`Archiver`接口中，因为它知道将要进行的归档类型。所以我们可以添加一个`Ext()`字符串方法，并从我们的`act`方法中访问它。但是我们可以通过允许`Archiver`作者指定整个文件名格式，而不仅仅是扩展名，来增加一点额外的功能而不需要太多额外的工作。

回到`archiver.go`，更新`Archiver`接口定义：

```go
type Archiver interface {

DestFmt() string

  Archive(src, dest string) error
}
```

我们的`zipper`类型现在需要实现这个：

```go
func (z *zipper) DestFmt() string {
  return "%d.zip"
}
```

现在我们可以要求我们的`act`方法从`Archiver`接口获取整个格式字符串，更新`act`方法：

```go
func (m *Monitor) act(path string) error {
  dirname := filepath.Base(path)
  filename := fmt.Sprintf(m.Archiver.DestFmt(), time.Now().UnixNano())
  return m.Archiver.Archive(path, filepath.Join(m.Destination, dirname, filename))
}
```

# 用户命令行工具

我们将构建的两个工具中的第一个允许用户为备份守护程序工具（稍后我们将编写）添加、列出和删除路径。你可以暴露一个 web 界面，或者甚至使用桌面用户界面集成的绑定包，但我们将保持简单，构建一个命令行工具。

在`backup`文件夹内创建一个名为`cmds`的新文件夹，并在其中创建另一个`backup`文件夹。

### 提示

将命令的文件夹和命令二进制本身命名为相同的名称是一个好的做法。

在我们的新`backup`文件夹中，将以下代码添加到`main.go`：

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

我们首先定义我们的`fatalErr`变量，并推迟检查该值是否为`nil`的函数。如果不是，它将打印错误以及标志默认值，并以非零状态代码退出。然后我们定义一个名为`db`的标志，它期望`filedb`数据库目录的路径，然后解析标志并获取剩余的参数，并确保至少有一个。

## 持久化小数据

为了跟踪路径和我们生成的哈希，我们需要一种数据存储机制，最好是在我们停止和启动程序时仍然有效。我们在这里有很多选择：从文本文件到完全水平可扩展的数据库解决方案。Go 的简单原则告诉我们，将数据库依赖性构建到我们的小型备份程序中并不是一个好主意；相反，我们应该问问我们如何能以最简单的方式解决这个问题？

`github.com/matryer/filedb`包是这种问题的实验性解决方案。它允许您与文件系统交互，就好像它是一个非常简单的无模式数据库。它从`mgo`等包中获取设计灵感，并且可以在数据查询需求非常简单的情况下使用。在`filedb`中，数据库是一个文件夹，集合是一个文件，其中每一行代表不同的记录。当然，随着`filedb`项目的发展，这一切都可能会发生变化，但接口希望不会变。

将以下代码添加到`main`函数的末尾：

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

在这里，我们使用`filedb.Dial`函数连接到`filedb`数据库。实际上，在这里并没有发生太多事情，除了指定数据库的位置，因为没有真正的数据库服务器可以连接（尽管这可能会在未来发生变化，这就是接口中存在这些规定的原因）。如果成功，我们推迟关闭数据库。关闭数据库确实会做一些事情，因为可能需要清理的文件可能是打开的。

按照`mgo`模式，接下来我们使用`C`方法指定一个集合，并将其引用保存在`col`变量中。如果在任何时候发生错误，我们将把它赋给`fatalErr`变量并返回。

为了存储数据，我们将定义一个名为`path`的类型，它将存储完整路径和最后一个哈希值，并使用 JSON 编码将其存储在我们的`filedb`数据库中。在`main`函数之前添加以下`struct`定义：

```go
type path struct {
  Path string
  Hash string
}
```

## 解析参数

当我们调用`flag.Args`（而不是`os.Args`）时，我们会收到一个不包括标志的参数切片。这允许我们在同一个工具中混合标志参数和非标志参数。

我们希望我们的工具能够以以下方式使用：

+   添加路径：

```go

backup -db=/path/to/db add {path} [paths...]

```

+   删除路径：

```go

backup -db=/path/to/db remove {path} [paths...]

```

+   列出所有路径：

```go

backup -db=/path/to/db list

```

为了实现这一点，因为我们已经处理了标志，我们必须检查第一个（非标志）参数。

将以下代码添加到`main`函数：

```go
switch strings.ToLower(args[0]) {
case "list":
case "add":
case "remove":
}
```

在这里，我们只需切换到第一个参数，然后将其设置为小写（如果用户输入`backup LIST`，我们仍希望它能正常工作）。

### 列出路径

要列出数据库中的路径，我们将在路径的`col`变量上使用`ForEach`方法。在列表情况下添加以下代码：

```go
var path path
col.ForEach(func(i int, data []byte) bool {
  err := json.Unmarshal(data, &path)
  if err != nil {
    fatalErr = err
    return false
  }
  fmt.Printf("= %s\n", path)
  return false
})
```

我们向`ForEach`传递一个回调函数，该函数将为该集合中的每个项目调用。然后我们将其从 JSON 解封到我们的`path`类型，并使用`fmt.Printf`将其打印出来。我们根据`filedb`接口返回`false`，这告诉我们返回`true`将停止迭代，我们要确保列出它们所有。

#### 自定义类型的字符串表示

如果以这种方式在 Go 中打印结构体，使用`%s`格式动词，你可能会得到一些混乱的结果，这些结果对用户来说很难阅读。但是，如果该类型实现了`String()`字符串方法，那么将使用该方法，我们可以使用它来控制打印的内容。在路径结构体下面，添加以下方法：

```go
func (p path) String() string {
  return fmt.Sprintf("%s [%s]", p.Path, p.Hash)
}
```

这告诉`path`类型应该如何表示自己。

### 添加路径

要添加一个或多个路径，我们将遍历剩余的参数并为每个参数调用`InsertJSON`方法。在`add`情况下添加以下代码：

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

如果用户没有指定任何其他参数，比如他们只是调用`backup add`而没有输入任何路径，我们将返回一个致命错误。否则，我们将完成工作并打印出路径字符串（前缀为`+`符号）以指示成功添加。默认情况下，我们将哈希设置为`Not yet archived`字符串字面量-这是一个无效的哈希，但它具有双重目的，既让用户知道它尚未被归档，又向我们的代码指示这一点（因为文件夹的哈希永远不会等于该字符串）。

### 删除路径

要删除一个或多个路径，我们使用路径的集合的`RemoveEach`方法。在`remove`情况下添加以下代码：

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

我们提供给`RemoveEach`的回调函数期望我们返回两个布尔类型：第一个指示是否应删除该项，第二个指示我们是否应停止迭代。

## 使用我们的新工具

我们已经完成了我们简单的`backup`命令行工具。让我们看看它的运行情况。在`backup/cmds/backup`内创建一个名为`backupdata`的文件夹；这将成为`filedb`数据库。

通过导航到`main.go`文件并运行终端中的以下命令来构建工具：

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

现在你应该看到完整的列表：

```go

./backup -db=./backupdata list

```

我们的程序应该产生：

```go

= ./test [Not yet archived]

= ./test2 [Not yet archived]

= ./test3 [Not yet archived]

```

让我们删除`test3`以确保删除功能正常：

```go

./backup -db=./backupdata remove ./test3

./backup -db=./backupdata list

```

这将把我们带回到：

```go

+ ./test [Not yet archived]

+ ./test2 [Not yet archived]

```

我们现在能够以符合我们用例的方式与`filedb`数据库进行交互。接下来，我们构建将实际使用我们的`backup`包执行工作的守护程序。

# 守护进程备份工具

`backup`工具，我们将其称为`backupd`，将负责定期检查`filedb`数据库中列出的路径，对文件夹进行哈希处理以查看是否有任何更改，并使用`backup`包来执行需要的文件夹的归档。

在`backup/cmds/backup`文件夹旁边创建一个名为`backupd`的新文件夹，让我们立即处理致命错误和标志：

```go
func main() {
  var fatalErr error
  defer func() {
    if fatalErr != nil {
      log.Fatalln(fatalErr)
    }
  }()
  var (
    interval = flag.Int("interval", 10, "interval between checks (seconds)")
    archive  = flag.String("archive", "archive", "path to archive location")
    dbpath   = flag.String("db", "./db", "path to filedb database")
  )
  flag.Parse()
}
```

你现在一定很习惯看到这种代码了。在指定三个标志之前，我们推迟处理致命错误：`interval`，`archive`和`db`。`interval`标志表示检查文件夹是否更改之间的秒数，`archive`标志是 ZIP 文件将存储的存档位置的路径，`db`标志是与`backup`命令交互的相同`filedb`数据库的路径。通常调用`flag.Parse`设置变量并验证我们是否准备好继续。

为了检查文件夹的哈希值，我们将需要我们之前编写的`Monitor`的一个实例。将以下代码附加到`main`函数：

```go
m := &backup.Monitor{
  Destination: *archive,
  Archiver:    backup.ZIP,
  Paths:       make(map[string]string),
}
```

在这里，我们使用`archive`值作为`Destination`类型创建了一个`backup.Monitor`方法。我们将使用`backup.ZIP`归档程序，并创建一个准备好在其中存储路径和哈希的映射。在守护程序开始时，我们希望从数据库加载路径，以便在停止和启动时不会不必要地进行归档。

将以下代码添加到`main`函数中：

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

你以前也见过这段代码；它拨号数据库并创建一个允许我们与`paths`集合交互的对象。如果出现任何问题，我们设置`fatalErr`并返回。

## 重复的结构

由于我们将使用与用户命令行工具程序中相同的路径结构，因此我们也需要为该程序包含一个定义。在`main`函数之前插入以下结构：

```go
type path struct {
  Path string
  Hash string
}
```

面向对象的程序员们毫无疑问现在正在对页面尖叫，要求这个共享的片段只存在于一个地方，而不是在两个程序中重复。我敦促你抵制这种早期抽象的冲动。这四行代码几乎不能证明我们的代码需要一个新的包和依赖，因此它们可以在两个程序中很容易地存在，而几乎没有额外开销。还要考虑到我们可能想要在我们的`backupd`程序中添加一个`LastChecked`字段，这样我们就可以添加规则，每个文件夹最多每小时归档一次。我们的`backup`程序不关心这一点，它将继续快乐地查看哪些字段构成了一个路径。

## 缓存数据

我们现在可以查询所有现有的路径并更新`Paths`映射，这是一种增加程序速度的有用技术，特别是在数据存储缓慢或断开连接的情况下。通过将数据加载到缓存中（在我们的情况下是`Paths`映射），我们可以以闪电般的速度访问它，而无需每次需要信息时都要查阅文件。

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

再次使用`ForEach`方法使我们能够遍历数据库中的所有路径。我们将 JSON 字节解组成与我们在其他程序中使用的相同路径结构，并在`Paths`映射中设置值。假设没有出现问题，我们最后检查以确保至少有一个路径，如果没有，则返回错误。

### 注意

我们程序的一个限制是一旦启动，它将无法动态添加路径。守护程序需要重新启动。如果这让你烦恼，你可以随时构建一个定期更新`Paths`映射的机制。

## 无限循环

接下来，我们需要立即对哈希进行检查，看看是否需要进行归档，然后进入一个无限定时循环，在其中以指定的间隔定期进行检查。

无限循环听起来像一个坏主意；实际上，对于一些人来说，它听起来像一个 bug。然而，由于我们正在谈论这个程序内部的一个无限循环，并且由于无限循环可以很容易地通过简单的`break`命令打破，它们并不像听起来那么戏剧性。

在 Go 中，编写无限循环就像这样简单：

```go
for {}
```

大括号内的指令会一遍又一遍地执行，尽可能快地运行代码的机器。再次听起来像一个坏计划，除非你仔细考虑你要求它做什么。在我们的情况下，我们立即启动了一个`select` case，它会安全地阻塞，直到其中一个通道有有趣的事情要说。

添加以下代码：

```go
check(m, col)
signalChan := make(chan os.Signal, 1)
signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)
for {
  select {
  case <-time.After(time.Duration(*interval) * time.Second):
    check(m, col)
  case <-signalChan:
    // stop
    fmt.Println()
    log.Printf("Stopping...")
    goto stop
  }
}
stop:
```

当然，作为负责任的程序员，我们关心用户终止我们的程序时会发生什么。因此，在调用尚不存在的`check`方法之后，我们创建一个信号通道，并使用`signal.Notify`要求将终止信号发送到通道，而不是自动处理。在我们无限的`for`循环中，我们选择两种可能性：要么`timer`通道发送消息，要么终止信号通道发送消息。如果是`timer`通道消息，我们再次调用`check`，否则我们终止程序。

`time.After`函数返回一个通道，在指定的时间过去后发送一个信号（实际上是当前时间）。有些令人困惑的`time.Duration(*interval) * time.Second`代码只是指示在发送信号之前要等待的时间量；第一个`*`字符是解引用运算符，因为`flag.Int`方法表示指向 int 的指针，而不是 int 本身。第二个`*`字符将间隔值乘以`time.Second`，从而得到与指定间隔相等的值（以秒为单位）。将`*interval int`转换为`time.Duration`是必需的，以便编译器知道我们正在处理数字。

在前面的代码片段中，我们通过使用`goto`语句来回顾一下内存中的短暂旅程，以跳出 switch 并阻止循环。我们可以完全不使用`goto`语句，只需在接收到终止信号时返回，但是这里讨论的模式允许我们在`for`循环之后运行非延迟代码，如果我们希望的话。

## 更新 filedb 记录

现在剩下的就是实现`check`函数，该函数应该调用`Monitor`类型的`Now`方法，并在有任何新的哈希值时更新数据库。

在`main`函数下面，添加以下代码：

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

`check`函数首先告诉用户正在进行检查，然后立即调用`Now`。如果`Monitor`类型为我们做了任何工作，即询问它是否归档了任何文件，我们将输出它们给用户，并继续使用新值更新数据库。`SelectEach`方法允许我们更改集合中的每个记录，如果我们愿意的话，通过返回替换的字节。因此，我们`Unmarshal`字节以获取路径结构，更新哈希值并返回编组的字节。这确保下次我们启动`backupd`进程时，它将使用正确的哈希值进行操作。

# 测试我们的解决方案

让我们看看我们的两个程序是否能很好地配合，以及它们对我们的`backup`包内部代码产生了什么影响。您可能希望为此打开两个终端窗口，因为我们将运行两个程序。

我们已经向数据库中添加了一些路径，所以让我们使用`backup`来查看它们：

```go

./backup -db="./backupdata" list

```

你应该看到这两个测试文件夹；如果没有，可以参考*添加路径*部分。

```go

= ./test [Not yet archived]

= ./test2 [Not yet archived]

```

在另一个窗口中，导航到`backupd`文件夹并创建我们的两个测试文件夹，名为`test`和`test2`。

使用通常的方法构建`backupd`：

```go

go build -o backupd

```

假设一切顺利，我们现在可以开始备份过程，确保将`db`路径指向与`backup`程序相同的路径，并指定我们要使用一个名为`archive`的新文件夹来存储 ZIP 文件。为了测试目的，让我们指定一个间隔为`5`秒以节省时间：

```go

./backupd -db="../backup/backupdata/" -archive="./archive" -interval=5

```

立即，`backupd`应该检查文件夹，计算哈希值，注意到它们是不同的（`尚未归档`），并启动两个文件夹的归档过程。它将打印输出告诉我们这一点：

```go

Checking...

Archived 2 directories

```

打开`backup/cmds/backupd`内新创建的`archive`文件夹，并注意它已经创建了两个子文件夹：`test`和`test2`。在这些文件夹中是空文件夹的压缩归档版本。随意解压一个并查看；到目前为止并不是很令人兴奋。

与此同时，在终端窗口中，`backupd`一直在检查文件夹是否有变化：

```go

Checking...

 No changes

Checking...

 No changes

```

在您喜欢的文本编辑器中，在`test2`文件夹中创建一个包含单词`test`的新文本文件，并将其保存为`one.txt`。几秒钟后，您会发现`backupd`已经注意到了新文件，并在`archive/test2`文件夹中创建了另一个快照。

当然，它的文件名不同，因为时间不同，但是如果您解压缩它，您会注意到它确实创建了文件夹的压缩存档版本。

通过执行以下操作来尝试解决方案：

+   更改`one.txt`文件的内容

+   将文件添加到`test`文件夹中

+   删除文件

# 摘要

在本章中，我们成功地为您的代码项目构建了一个非常强大和灵活的备份系统。您可以看到扩展或修改这些程序行为有多么简单。您可以解决的潜在问题范围是无限的。

与上一节不同，我们不是将本地存档目标文件夹，而是想象挂载网络存储设备并使用该设备。突然间，您就可以对这些重要文件进行离站（或至少是离机）备份。您可以轻松地将 Dropbox 文件夹设置为存档目标，这意味着不仅您自己可以访问快照，而且副本存储在云中，甚至可以与其他用户共享。

扩展`Archiver`接口以支持`Restore`操作（只需使用`encoding/zip`包解压文件）允许您构建可以查看存档内部并访问单个文件更改的工具，就像 Time Machine 允许您做的那样。索引文件使您可以在整个代码历史记录中进行全面搜索，就像 GitHub 一样。

由于文件名是时间戳，您可以将旧存档备份到不太活跃的存储介质，或者将更改总结为每日转储。

显然，备份软件已经存在，经过充分测试，并且在全球范围内得到使用，专注于解决尚未解决的问题可能是一个明智的举措。但是，当写小程序几乎不费吹灰之力时，通常值得去做，因为它给予您控制权。当您编写代码时，您可以得到完全符合您要求的结果，而不需要妥协，这取决于每个人做出的决定。

具体来说，在本章中，我们探讨了 Go 标准库如何轻松地与文件系统交互：打开文件进行读取，创建新文件和创建目录。与`io`包中的强大类型混合在一起的`os`包，再加上像`encoding/zip`等功能，清楚地展示了极其简单的 Go 接口如何组合以产生非常强大的结果。
