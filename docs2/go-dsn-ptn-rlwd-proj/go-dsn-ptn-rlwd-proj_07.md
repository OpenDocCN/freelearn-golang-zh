# 第六章。Go 包和程序

第五章，*Go 中的函数*介绍了函数，这是代码组织抽象的基本层次，使得代码可寻址和可重用。本章继续向上攀登抽象的阶梯，以 Go 包为中心进行讨论。正如以下主题所详细介绍的，一个包是一组逻辑上分组存储在源代码文件中的语言元素，可以共享和重用。以下是一些相关主题：

+   Go 包

+   创建包

+   构建包

+   包可见性

+   导入包

+   包初始化

+   创建程序

+   远程包

# Go 包

与其他语言类似，Go 源代码文件被分组为可编译和可共享的单元，称为包。然而，所有 Go 源文件都必须属于一个包（没有默认包的概念）。这种严格的方法允许 Go 通过优先考虑惯例而不是配置来保持其编译规则和包解析规则简单。让我们深入探讨包的基本原理，包括它们的创建、使用和推荐实践。

## 理解 Go 包

在我们深入包的创建和使用之前，对包的概念有一个高层次的理解至关重要，这有助于引导后续的讨论。Go 包是代码组织的物理和逻辑单元，用于封装可重用的相关概念。按照惯例，存储在相同目录中的源文件组被认为是同一包的一部分。以下是一个简单的目录树示例，其中每个目录代表一个包含一些源代码的包：

```go
 foo
 ├── blat.go
 └── bazz
 ├── quux.go
 └── qux.go 

```

golang.fyi/ch06-foo

虽然不是必需的，但建议的惯例是在每个源文件中将包的名称设置为与文件所在目录的名称匹配。例如，源文件 `blat.go` 被声明为 `foo` 包的一部分，如下面的代码所示，因为它存储在名为 `foo` 的目录中：

```go
package foo 

import ( 
   "fmt" 
   "foo/bar/bazz" 
) 

func fooIt() { 
   fmt.Println("Foo!") 
   bazz.Qux() 
} 

```

golang.fyi/ch06-foo/foo/blat.go

文件 `quux.go` 和 `qux.go` 都属于包 `bazz`，因为它们位于名为该名称的目录中，如下面的代码片段所示：

|

```go
package bazz
import "fmt"
func Qux() {
  fmt.Println("bazz.Qux")
}
```

golang.fyi/ch06-foo/foo/bazz/quux.go |

```go
package bazz
import "fmt"
func Quux() {
  Qux()fmt.Println("gazz.Quux")
}
```

golang.fyi/ch06-foo/foo/bazz/qux.go |

## 工作区

在讨论包时，还有一个重要概念需要理解，那就是 *Go 工作区*。工作区只是一个任意目录，用作命名空间，用于在编译等特定任务中解析包。按照惯例，Go 工具期望在工作区目录中有三个特定命名的子目录：`src`、`pkg` 和 `bin`。这些子目录存储 Go 源文件以及所有构建的包工件。

建立一个静态目录位置，将 Go 包放在一起，具有以下优点：

+   几乎零配置的简单设置

+   通过减少代码搜索到已知位置来快速编译

+   工具可以轻松创建代码和包文件的源图

+   从源自动推断和解析传递依赖项

+   可以使项目设置便携且易于分发

以下是我笔记本电脑上 Go 工作空间的局部（简化）树布局，其中突出显示了三个子目录 `bin`、`pkg` 和 `src`：

|

```go
/home/vladimir/Go/   
├── bin   
│  ├── circ   
│  ├── golint   
│  ...   
├── pkg   
│  └── linux_amd64    
│    ├── github.com   
│    │  ├── golang   
│    │  │  └── lint.a   
│    │  └── vladimirvivien   
│    │    └── learning-go   
│    │      └── ch06   
│    │        ├── current.a   
│    ...       ...    
└── src   
  ├── github.com   
  │  ├── golang   
  │  │  └── lint   
  │  │    ├── golint   
  │  │    │  ├── golint.go   
  │  ...   ... ...   
  │  └── vladimirvivien   
  │    └── learning-go   
  │      ├── ch01   
  │      ...   
  │      ├── ch06   
  │      │  ├── current   
  │      │  │  ├── doc.go   
  │      │  │  └── lib.go   
  ...     ...      

```

|

示例工作空间目录

+   `bin`：这是一个自动生成的目录，用于存储编译后的 Go 可执行文件（也称为程序或命令）。当 Go 工具编译和安装可执行包时，它们将被放置在这个目录中。前一个示例工作空间显示了两个二进制文件 `circ` 和 `golint`。建议将此目录添加到操作系统的 `PATH` 环境变量中，以便在本地使用你的命令。

+   `pkg`：此目录也是自动生成的，用于存储构建的包文件。当 Go 工具构建和安装非可执行包时，它们将作为具有 `.a` 后缀的对象文件存储在基于目标操作系统和架构命名的子目录中。在示例工作空间中，对象文件被放置在 `linux_amd64` 子目录下，这表明该目录中的对象文件是为在 64 位架构上运行的 Linux 操作系统编译的。

+   `src`：这是一个用户创建的目录，用于存储 Go 源代码文件。`src` 下的每个子目录都映射到一个包。*src* 是所有导入路径解析的根目录。Go 工具搜索该目录以解析在编译或其他依赖于源路径的活动期间引用的包。前一个示例工作空间显示了两个包：`github.com/golang/lint/golint/` 和 `github.com/vladimirvivien/learning-go/ch06/current`。

### 注意

你可能对工作空间示例中显示的包路径中的 `github.com` 前缀感到好奇。值得注意的是，对于包目录没有命名要求（参见 *命名包* 部分）。包可以有任何任意名称。然而，Go 推荐某些约定，这些约定有助于全局命名空间解析和包组织。

## 创建工作空间

创建工作空间就像设置一个名为 `GOPATH` 的操作系统环境变量，并将其分配给工作空间目录的根路径。例如，在一个 Linux 机器上，如果工作空间的根目录是 `/home/username/Go`，则工作空间将被设置为：

```go
$> export GOPATH=/home/username/Go 

```

在设置 `GOPATH` 环境变量时，可以指定存储包的多个位置。每个目录由一个操作系统依赖的路径分隔符字符（换句话说，Linux/Unix 为冒号，Windows 为分号）分隔，如下所示：

```go
$> export GOPATH=/home/myaccount/Go;/home/myaccount/poc/Go

```

当解析包名称时，Go 工具将搜索`GOPATH`中列出的所有位置。然而，Go 编译器将只将编译后的工件，如对象和二进制文件，存储在分配给`GOPATH`的第一个目录位置。

### 注意

通过简单地设置操作系统环境变量来配置工作空间具有巨大的优势。这使开发者能够在编译时动态地设置工作空间以满足某些工作流程要求。例如，开发者可能希望在合并之前测试一个未经验证的代码分支。他或她可能需要设置一个临时工作空间来构建该代码，如下所示（Linux）: `$> GOPATH=/temporary/go/workspace/path go build`

## 导入路径

在继续介绍设置和使用包的细节之前，还有一个最后的重要概念需要介绍，那就是*导入路径*的概念。在`$GOPATH/src`下的每个包的相对路径构成了一个全局标识符，称为包的`导入路径`。这意味着在给定的工作空间中，没有两个包可以具有相同的导入路径值。

让我们回到之前简化的目录树。例如，如果我们设置工作空间为某个任意的路径值，例如`GOPATH=/home/username/Go`：

```go
/home/username/Go
└── foo
 ├── ablt.go
 └── bazz
 ├── quux.go
 └── qux.go 

```

从上面所示的示例工作空间中，包的目录路径映射到它们各自的导入路径，如下表所示：

| **目录路径** | **导入路径** |
| --- | --- |
| `/home/username/Go/foo` |

```go
"foo"   

```

|

| `/home/username/Go/foo/bar` |
| --- |

```go
"foo/bar"   

```

|

| `/home/username/Go/foo/bar/bazz` |
| --- |

```go
"foo/bar/bazz"   

```

|

# 创建包

到目前为止，本章已经介绍了 Go 包的基本概念；现在是时候深入探讨包中 Go 代码的创建。Go 包的主要目的之一是将公共逻辑抽象出来并聚合到可共享的代码单元中。在本章的早期部分提到，目录中的一个 Go 源代码文件组被认为是包。虽然这在技术上是真的，但 Go 包的概念不仅仅是将一堆文件放入目录中。

为了帮助说明第一个包的创建，我们将使用在[github.com/vladimirvivien/learning-go/ch06](https://github.com/vladimirvivien/learning-go/ch06)中找到的示例源代码。该目录中的代码定义了一组函数，用于使用*欧姆定律*计算电值。以下显示了组成示例包的目录布局（假设它们保存在某个工作空间目录`$GOPATH/src`中）：

|

```go
github.com/vladimirvivien/learning-go/ch06   
├── current   
│  ├── curr.go   
│  └── doc.go   
├── power   
│  ├── doc.go   
│  ├── ir   
│  │  └── power.go   
│  ├── powlib.go   
│  └── vr   
│    └── power.go   
├── resistor   
│  ├── doc.go   
│  ├── lib.go   
│  ├── res_equivalence.go   
│  ├── res.go   
│  └── res_power.go   
└── volt   
  ├── doc.go   
  └── volt.go   

```

|

Ohm 定律示例的包布局

在之前的树状结构中，每个目录都包含一个或多个 Go 源代码文件，这些文件定义并实现了将被安排到包中并使其可重用的函数和其他源代码元素。以下表格总结了从先前工作空间布局中提取的导入路径和包信息：

| **导入路径** | **包** |
| --- | --- |
| "github.com/vladimirvivien/learning-go/ch06/**current**" | `current` |
| "github.com/vladimirvivien/learning-go/ch06/**power**" | `power` |
| "github.com/vladimirvivien/learning-go/ch06/**power/ir**" | `ir` |
| "github.com/vladimirvivien/learning-go/ch06/**power/vr**" | `vr` |
| "github.com/vladimirvivien/learning-go/ch06/**resistor**" | `resistor` |
| "github.com/vladimirvivien/learning-go/ch06/**volt**" | `volt` |

虽然没有命名要求，但给包目录命名以反映它们各自的目的还是很有道理的。从上一个表中可以看出，示例中的每个包都被命名为代表一个电学概念，例如电流、功率、电阻和电压。*包命名*部分将进一步详细介绍包命名约定。

## 声明包

Go 源文件必须声明自己属于一个包。这是通过使用`package`子句来完成的，它是 Go 源文件中的第一个合法语句。声明的包由`package`关键字后跟一个名称标识符组成。以下显示了`volt`包中的源文件`volt.go`：

```go
package volt 

func V(i, r float64) float64 { 
   return i * r 
} 

func Vser(volts ...float64) (Vtotal float64) { 
   for _, v := range volts { 
         Vtotal = Vtotal + v 
   } 
   return 
} 

func Vpi(p, i float64) float64 { 
   return p / i 
} 

```

golang.fyi/ch06/volt/volt.go

源文件中的包标识符可以设置为任何任意值。与 Java 不同，包的名称并不反映源文件所在的目录结构。虽然对包名称没有要求，但将包标识符命名为与文件所在的目录相同的名称是一种公认的约定。在我们之前的源代码列表中，包被声明为标识符`volt`，因为文件存储在 *volt* 目录中。

## 多文件包

包的逻辑内容（源代码元素，如类型、函数、变量和常量）可以物理地跨越多个 Go 源文件。包目录可以包含一个或多个 Go 源文件。例如，在以下示例中，包`resistor`被不必要地分割成几个 Go 源文件，以说明这一点：

|

```go
package resistor   

func recip(val float64) float64 {   
   return 1 / val   
}   

```

golang.fyi/ch06/resistor/lib.go |

|

```go
  package resistor   

func Rser(resists ...float64) (Rtotal float64) {   
   for _, r := range resists {   
         Rtotal = Rtotal + r   
   }   
   return   
}   

func Rpara(resists ...float64) (Rtotal float64) {   
   for _, r := range resists {   
         Rtotal = Rtotal + recip(r)   
   }   
   return   
}   

```

golang.fyi/ch06/resistor/res_equivalance.go |

|

```go
package resistor   

func R(v, i float64) float64 {   
   return v / i   
}   

```

golang.fyi/ch06/resistor/res.go |

|

```go
package resistor   

func Rvp(v, p float64) float64 {   
   return (v * v) / p   
}   

```

golang.fyi/ch06/resistor/res_power.go |

每个包中的文件都必须有一个与相同名称标识符（在这种情况下为`resistor`）的包声明。Go 编译器会将所有源文件中的所有元素缝合在一起，形成一个可以在单个作用域内被其他包使用的单一逻辑单元。

需要指出的是，如果给定目录中的所有源文件中的包声明不相同，则编译将失败。这是可以理解的，因为编译器期望目录中的所有文件都属于同一个包。

## 包命名

如前所述，Go 期望工作区中的每个包都有一个唯一的完全限定导入路径。你的程序可以有任意多的包，你的包结构可以在工作区中尽可能深。然而，Go 的惯用规则规定了包的命名和组织的一些**规则**，以便于创建和使用包。

### 使用全局唯一命名空间

首先，在全局范围内完全限定你的包的导入路径是一个好主意，特别是如果你打算与他人共享代码。考虑以一个唯一标识你或你组织的命名空间方案开始你的导入路径名称。例如，公司*Acme, Inc.*可能选择以`acme.com/apps`开始所有他们的 Go 包名称。因此，一个包的完全限定导入路径将是`"acme.com/apps/foo/bar"`。

### 注意

在本章的后面部分，我们将看到如何在使用 GitHub 等源代码仓库服务时使用包导入路径。

### 为路径添加上下文

接下来，当你为你的包制定命名方案时，使用包的路径为你的包名添加上下文。名称中的上下文应从通用开始，从左到右变得越来越具体。例如，让我们参考前面示例中的 power 包的导入路径。功率值的计算被分为三个子包，如下所示：

+   `github.com/vladimirvivien/learning-go/ch06/**power**`

+   `github.com/vladimirvivien/learning-go/ch06/**power/ir**`

+   `github.com/vladimirvivien/learning-go/ch06/**power/vr**`

父路径`power`包含具有更广泛上下文的包成员。子包`ir`和`vr`包含具有更具体、上下文更窄的成员。这种命名模式在 Go 中广泛使用，包括以下内置包：

+   `crypto/md5`

+   `net/http`

+   `net/http/httputil`

+   `reflect`

注意一个包深度为一个是一个完全合法的包名（参见`reflect`），只要它能捕捉到其上下文和所做事情的本质。再次强调，保持简单。避免在命名空间内嵌套包超过三层。如果你是一个习惯于长嵌套包名的 Java 开发者，这种诱惑将特别强烈。

### 使用短名称

当审查内置 Go 包的名称时，你会注意到与其他语言相比，名称的简短。在 Go 中，一个包被认为是一组实现一组紧密相关功能的代码集合。因此，你的包的导入路径应该是简洁的，并反映它们的功能，而不要过长。我们的示例源代码通过使用如 volt、power、resistance、current 等简短名称来命名包目录来体现这一点。在它们各自的上下文中，每个目录名称都确切地说明了包的功能。

短名称规则在 Go 的内置包中得到了严格的执行。例如，以下是从 Go 的内置包中的一些包名：`log`、`http`、`xml`和`zip`。每个名称都能清楚地识别包的用途。

### 注意

简短的包名在大型代码库中具有减少按键次数的优势。然而，拥有简短且通用的包名也有其缺点，即容易发生导入路径冲突。在大型项目（或开源库的开发者）中，他们可能会在代码中使用相同的流行名称（换句话说，`log`、`util`、`db`等等）。正如我们在本章后面将要看到的，这可以通过使用`named`导入路径来处理。

# 构建包

Go 工具通过应用某些约定和合理的默认值来降低编译代码的复杂性。尽管 Go 的构建工具的全面讨论超出了本节（或本章）的范围，但了解`build`和`install`工具的目的和使用方法是很有用的。一般来说，构建和安装工具的使用方法如下：

*$> go build [<package import path>]* 

`import path`可以显式提供或完全省略。`build`工具接受以完全限定或相对路径表示的`import path`。给定一个正确设置的工作区，以下都是编译前面示例中的包`volt`的等效方式：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go
$> go build ./ch06/volt 
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go build ./volt 
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06/volt
$> go build . 
$> cd $GOPATH/src/ 
$> go build github.com/vladimirvivien/learning-go/ch06/current /volt

```

上述`go build`命令将编译在目录`volt`中找到的所有 Go 源文件及其依赖项。此外，还可以使用附加到导入路径的通配符参数构建给定目录中的所有包和子包，如下所示：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go build ./...

```

以下将构建在目录`$GOPATH/src/github.com/vladimirvivien/learning-go/ch06`中找到的所有包和子包。

## 安装包

默认情况下，构建命令将结果输出到由工具生成的临时目录中，该目录在构建过程完成后会丢失。要实际生成可用的工件，必须使用`install`工具来保留编译对象文件的副本。

`install`工具与构建工具具有完全相同的语义：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go install ./volt

```

除了编译代码外，它还会将结果保存并输出到工作区位置`$GOPATH/pkg`，如下所示：

```go
$GOPATH/pkg/linux_amd64/github.com/vladimirvivien/learning-go/
└── ch06
 └── volt.a

```

生成的对象文件（具有`.a`扩展名）允许包在工作区中重用并与其他包链接。在本章后面，我们将探讨如何编译可执行程序。

# 包可见性

无论声明的源文件数量有多少，所有在包级别声明的源代码元素（类型、变量、常量和函数）都共享一个公共作用域。因此，编译器不会允许在包的整个范围内重复声明元素标识符。让我们使用以下代码片段来说明这一点，假设这两个源文件都是同一包`$GOPATH/src/foo`的一部分：

|

```go
package foo   

var (   
  bar int = 12   
)   

func qux () {   
  bar += bar   
}   

```

foo/file1.go

```go
package foo   

var bar struct{   
  x, y int   
}   

func quux() {   
  bar = bar * bar   
}   

```

foo/file2.go

非法变量标识符重新声明

虽然它们位于两个不同的文件中，但在 Go 语言中，具有标识符 `bar` 的变量声明是非法的。由于这两个文件是同一包的一部分，因此这两个标识符具有相同的范围，因此会发生冲突。

对于函数标识符也是如此。Go 语言不支持在同一作用域内重载函数名。因此，无论函数的签名如何，使用超过一次的函数标识符都是非法的。如果我们假设以下代码出现在同一包内的两个不同的源文件中，以下片段将是非法的：

|

```go
package foo   

var (   
  bar int = 12   
)   

func qux () {   
  bar += bar   
}   

```

foo/file1.go

```go
package foo   

var (   
  fooVal int = 12   
)   

func qux (inc int) int {   
  return fooVal += inc   
}   

```

foo/file1.go

非法函数标识符重新声明

在前面的代码片段中，函数名标识符 `qux` 被使用了两次。即使两个函数有不同的签名，编译器也会失败编译。唯一修复的方法是更改名称。

## 包成员可见性

包的有用之处在于它能够将其源元素暴露给其他包。控制包元素的可见性很简单，遵循以下规则：*大写标识符会自动导出*。这意味着任何具有大写标识符的类型、变量、常量或函数都会自动从声明它的包外部可见。

参考前面描述的欧姆定律示例，以下展示了从 `resistor` 包（位于 [github.com/vladimirvivien/learning-go/ch06/resistor](https://github.com/vladimirvivien/learning-go/ch06/resistor)）中的这一功能：

| **代码** | **描述** |
| --- | --- |

|

```go
package resistor   

func R(v, i float64) float64 {   
   return v / i   
}   

```

| 函数 `R` 会自动导出，可以从其他包中访问：`resistor.R()` |
| --- |

|

```go
package resistor   

func recip(val float64) float64 {   
   return 1 / val   
}   

```

| 函数标识符 `recip` 全部为小写，因此不会被导出。尽管在它自己的作用域内是可访问的，但该函数在其他包中是不可见的。 |
| --- |

值得重申的是，同一包内的成员总是对彼此可见。在 Go 语言中，没有像其他语言中那样的复杂可见性结构，如私有、友元、默认等。这使开发者能够专注于正在实现的解决方案，而不是建模可见性层次。

# 导入包

在这个阶段，你应该已经很好地理解了什么是包，它有什么作用，以及如何创建一个包。现在，让我们看看如何使用包来导入和重用其成员。正如你将在其他几种语言中发现的那样，关键字 `import` 用于从外部包导入源代码元素。它允许导入的源访问在导入的包中找到的导出元素（参见本章前面提到的 *包作用域和可见性* 部分）。导入子句的一般格式如下：

*import [包名标识符] "<导入路径>"*

注意，导入路径必须用双引号括起来。`import` 语句还支持一个可选的包标识符，可以用来显式命名导入的包（稍后讨论）。`import` 语句也可以写成导入块，如下所示。这在有两个或更多导入包列表的情况下很有用：

*import (*

*[包名称标识符] "<导入路径>"*

*)*

以下源代码片段显示了之前介绍的欧姆定律示例中的导入声明块：

```go
import ( 
   "flag" 
   "fmt" 
   "os" 

   "github.com/vladimirvivien/learning-go/ch06/current" 
   "github.com/vladimirvivien/learning-go/ch06/power" 
   "github.com/vladimirvivien/learning-go/ch06/power/ir" 
   "github.com/vladimirvivien/learning-go/ch06/power/vr" 
      "github.com/vladimirvivien/learning-go/ch06/volt" 
) 

```

golang.fyi/ch06/main.go

通常省略导入包的名称标识符，如上所示。Go 将导入路径的最后一个目录的名称作为导入包的名称标识符，如下表所示的一些包所示：

| **导入路径** | **包名** |
| --- | --- |
| `flag` | `flag` |
| `github.com/vladimirvivien/learning-go/ch06/current` | `current` |
| `github.com/vladimirvivien/learning-go/ch06/power/ir` | `ir` |
| `github.com/vladimirvivien/learning-go/ch06/volt` | `volt` |

```go
volt.V() is invoked from imported package "github.com/vladimirvivien/learning-go/ch06/volt":
```

```go
... 
import "github.com/vladimirvivien/learning-go/ch06/volt" 
func main() { 
   ... 
   switch op { 
   case "V", "v": 
         val := volt.V(i, r) 
  ... 
} 

```

golang.fyi/ch06/main.go

## 指定包标识符

```go
import res "github.com/vladimirvivien/learning-go/ch06/resistor"
```

按照前面描述的格式，名称标识符放在导入路径之前，如前面的片段所示。命名包可以用作缩短或自定义包名称的一种方式。例如，在一个包含大量特定包使用的源文件中，这可以是一个减少按键的好功能。

给包命名也是避免给定源文件中包标识符冲突的一种方法。可以想象导入两个或更多具有不同导入路径但解析到相同包名的包。例如，您可能需要使用来自不同库的两个不同的日志系统来记录信息，如下面的代码片段所示：

```go
package foo 
import ( 
   flog "github.com/woom/bat/logger" 
   hlog "foo/bar/util/logger" 
) 

func main() { 
   flog.Info("Programm started") 
   err := doSomething() 
   if err != nil { 
     hlog.SubmitError("Error - unable to do something") 
   } 
} 

"logger" by default. To resolve this, at least one of the imported packages must be assigned a name identifier to resolve the name clash. In the previous example, both import paths were named with a meaningful name to help with code comprehension.
```

## 点标识符

```go
SubmitError from the logger package, the package name is omitted:
```

```go
package foo 

import ( 
   . "foo/bar/util/logger" 
) 

func main() { 
   err := doSomething() 
   if err != nil { 
     SubmitError("Error - unable to do something") 
   } 
} 

```

虽然这个特性可以帮助减少重复的按键，但并不鼓励这种做法。通过合并包的作用域，更容易遇到标识符冲突。

## 空标识符

```go
fmt; however, it never uses it in the subsequent source code:
```

```go
package foo 
import ( 
   _ "fmt" 
   "foo/bar/util/logger" 
) 

func main() { 
   err := doSomething() 
   if err != nil { 
     logger.Submit("Error - unable to do something") 
   } 
} 

```

空标识符的一个常见用法是加载包以产生副作用。这依赖于当导入包时的初始化顺序（参见以下 *包初始化* 部分）。使用空标识符会导致即使无法引用其任何成员，导入的包也会被初始化。这在需要静默运行某些初始化序列的上下文中被使用。

# 包初始化

```go
foo will be a, y, b, and x:
```

```go
package foo 
var x = a + b(a) 
var a = 2 
var b = func(i int) int {return y * i} 
var y = 3 

```

Go 还使用一个名为 `init` 的特殊函数，它不接受任何参数也不返回任何结果值。它用于封装在导入包时调用的自定义初始化逻辑。例如，以下源代码显示了在 `resistor` 包中使用 `init` 函数来初始化函数变量 `Rpi`：

```go
package resistor 

var Rpi func(float64, float64) float64 

func init() { 
   Rpi = func(p, i float64) float64 { 
         return p / (i * i) 
   } 
} 

func Rvp(v, p float64) float64 { 
   return (v * v) / p 
} 

```

golang.fyi/ch06/resistor/res_power.go

在前面的代码中，`init` 函数在包级变量初始化之后被调用。因此，`init` 函数中的代码可以安全地依赖于声明的变量值处于稳定状态。`init` 函数具有以下特殊之处：

+   一个包可以定义多个 `init` 函数

+   你不能在运行时直接访问声明的 `init` 函数

+   它们按照在源文件中出现的词法顺序执行

+   `init` 函数是将逻辑注入在执行任何其他函数或方法之前执行的包的一种很好的方式。

# 创建程序

到目前为止，在本书中，你已经学习了如何创建和打包 Go 代码作为可重用的包。然而，一个包不能作为一个独立程序执行。要创建一个程序（也称为命令），你需要将一个包作为一个执行入口点进行定义，如下所示：

+   声明（至少一个）源文件为名为 `main` 的特殊包的一部分

+   声明一个函数名 `main()` 作为程序的入口点

`main` 函数不接受任何参数也不返回任何值。以下显示了用于欧姆定律示例的 `main` 包的缩略源代码（从前面）。它使用来自 Go 标准库的 `flag` 包来解析格式为 `flag` 的程序参数：

```go
package main 
import ( 
   "flag" 
   "fmt" 
   "os" 

   "github.com/vladimirvivien/learning-go/ch06/current" 
   "github.com/vladimirvivien/learning-go/ch06/power" 
   "github.com/vladimirvivien/learning-go/ch06/power/ir" 
   "github.com/vladimirvivien/learning-go/ch06/power/vr" 
   res "github.com/vladimirvivien/learning-go/ch06/resistor" 
   "github.com/vladimirvivien/learning-go/ch06/volt" 
) 

var ( 
   op string 
   v float64 
   r float64 
   i float64 
   p float64 

   usage = "Usage: ./circ <command> [arguments]\n" + 
     "Valid command { V | Vpi | R | Rvp | I | Ivp |"+  
    "P | Pir | Pvr }" 
) 

func init() { 
   flag.Float64Var(&v, "v", 0.0, "Voltage value (volt)") 
   flag.Float64Var(&r, "r", 0.0, "Resistance value (ohms)") 
   flag.Float64Var(&i, "i", 0.0, "Current value (amp)") 
   flag.Float64Var(&p, "p", 0.0, "Electrical power (watt)") 
   flag.StringVar(&op, "op", "V", "Command - one of { V | Vpi |"+   
    " R | Rvp | I | Ivp | P | Pir | Pvr }") 
} 

func main() { 
   flag.Parse() 
   // execute operation 
   switch op { 
   case "V", "v": 
    val := volt.V(i, r) 
    fmt.Printf("V = %0.2f * %0.2f = %0.2f volts\n", i, r, val) 
   case "Vpi", "vpi": 
   val := volt.Vpi(p, i) 
    fmt.Printf("Vpi = %0.2f / %0.2f = %0.2f volts\n", p, i, val) 
   case "R", "r": 
   val := res.R(v, i)) 
    fmt.Printf("R = %0.2f / %0.2f = %0.2f Ohms\n", v, i, val) 
   case "I", "i": 
   val := current.I(v, r)) 
    fmt.Printf("I = %0.2f / %0.2f = %0.2f amps\n", v, r, val) 
   ... 
   default: 
         fmt.Println(usage) 
         os.Exit(1) 
   } 
} 

```

golang.fyi/ch06/main.go

以下列表显示了 `main` 包的源代码以及当程序运行时执行的 `main` 函数的实现。欧姆定律程序接受命令行参数，指定要执行哪种电操作（请参阅以下 *访问程序参数* 部分）。`init` 函数用于初始化程序标志值的解析。`main` 函数被设置为一个大的 switch 语句块，根据选定的标志选择要执行的正确操作。

## 访问程序参数

当程序执行时，Go 运行时通过包变量 `os.Args` 将所有命令行参数作为一个切片提供。例如，当以下程序执行时，它打印出传递给程序的所有命令行参数：

```go
package main 
import ( 
   "fmt" 
   "os" 
) 

func main() { 
   for _, arg := range os.Args { 
         fmt.Println(arg) 
   } 
} 

```

golang.fyi/ch06-args/hello.go

以下是在使用所示参数调用程序时的输出：

```go
$> go run hello.go hello world how are you?
/var/folders/.../exe/hello
hello
world
how
are
you?

```

注意，命令行参数 `"hello world how are you?"` 放在程序名称之后，被分割成一个空格分隔的字符串。`os.Args` 切片中的位置 0 保存了程序二进制路径的完全限定名称。切片的其余部分分别存储字符串中的每个项。

Go 标准库中的 `flag` 包内部使用此机制来提供结构化命令行参数的处理，这些参数被称为标志。在前面列出的欧姆定律示例中，`flag` 包用于解析以下源代码片段中列出的几个标志（从前面的完整列表中提取）：

```go
var ( 
   op string 
   v float64 
   r float64 
   i float64 
   p float64 
) 

func init() { 
   flag.Float64Var(&v, "v", 0.0, "Voltage value (volt)") 
   flag.Float64Var(&r, "r", 0.0, "Resistance value (ohms)") 
   flag.Float64Var(&i, "i", 0.0, "Current value (amp)") 
   flag.Float64Var(&p, "p", 0.0, "Electrical power (watt)") 
   flag.StringVar(&op, "op", "V", "Command - one of { V | Vpi |"+   
    " R | Rvp | I | Ivp | P | Pir | Pvr }") 
} 
func main(){ 
  flag.Parse() 
  ... 
} 

init used to parse and initialize expected flags "v", "i", "p", and "op" (at runtime, each flag is prefixed with a minus sign). The initialization functions in package flag sets up the expected type, the default value, a flag description, and where to store the parsed value for the flag. The flag package also supports the special flag "help", used to provide helpful hints about each flag.
```

在`main`函数中，`flag.Parse()`用于启动解析任何作为命令行提供的标志的过程。例如，为了计算一个 12 伏特和 300 欧姆的电路的电流，程序需要三个标志并产生以下输出：

```go
$> go run main.go -op I -v 12 -r 300
I = 12.00 / 300.00 = 0.04 amps

```

## 构建和安装程序

构建和安装 Go 程序遵循与构建常规包完全相同的程序（如在前面的*构建和安装包*部分中讨论的那样）。当你构建可执行 Go 程序源文件时，编译器将通过传递链接`main`包中声明的所有依赖项来生成一个可执行的二进制文件。默认情况下，构建工具将输出二进制文件命名为与 Go 程序源文件所在的目录相同的名字。

例如，在欧姆定律的示例中，位于`github.com/vladimirvivien/learning-go/ch06`目录中的`main.go`文件被声明为`main`包的一部分。程序可以按照以下方式构建：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go build .

```

当`main.go`源文件被构建时，构建工具将生成一个名为`ch06`的二进制文件，因为程序的源代码位于一个同名目录中。你可以使用输出标志`-o`来控制二进制文件的名字。在以下示例中，构建工具创建了一个名为`ohms`的二进制文件。

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go build -o ohms

```

最后，使用 Go 的`install`命令安装 Go 程序的方式与使用 Go 安装常规包的方式完全相同：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go install .

```

当使用 Go 的`install`命令安装程序时，如果需要，它将被构建，其生成的二进制文件将被保存在`$GOPAHT/bin`目录中。将工作区`bin`目录添加到你的操作系统的`$PATH`环境变量中，将使你的 Go 程序可用于执行。

### 注意

Go 生成的程序是静态链接的二进制文件。它们运行时不需要满足任何额外的依赖。然而，Go 编译的二进制文件包含了 Go 运行时。这是一组处理垃圾回收、类型信息、反射、goroutine 调度和 panic 管理的操作。虽然一个类似的 C 程序可能要小得多，但 Go 的运行时附带了一些使 Go 变得有趣的工具。

# 远程包

Go 附带的一个工具允许程序员直接从远程源代码仓库检索包。默认情况下，Go 可以轻松地与以下版本控制系统集成：

+   Git (`git`, [`git-scm.com/`](http://git-scm.com/))

+   Mercurial (`hg`, [`www.mercurial-scm.org/`](https://www.mercurial-scm.org/))

+   Subversion (`svn`, [`subversion.apache.org/`](http://subversion.apache.org/))

+   Bazaar (`bzr`, [`bazaar.canonical.com/`](http://bazaar.canonical.com/))

### 注意

为了让 Go 从远程仓库拉取包源代码，您必须在操作系统的执行路径上安装该版本控制系统的客户端作为命令。在底层，Go 启动客户端与源代码仓库服务器进行交互。

`get` 命令行工具允许程序员使用完全限定的项目路径作为包的导入路径来检索远程包。一旦下载了包，就可以将其导入到本地源文件中使用。例如，如果您想包含前一个片段中 Ohm 定律示例中的一个包，您可以从命令行发出以下命令：

```go
$> go get github.com/vladimirvivien/learning-go/ch06/volt

```

`go get` 工具会下载指定的导入路径以及所有引用的依赖项。然后，该工具将在 `$GOPATH/pkg` 中构建和安装包的工件。如果 `import` 路径恰好是一个程序，go get 还会在 `$GOPATH/bin` 中生成二进制文件以及 `$GOPATH/pkg` 中引用的任何包。

# 摘要

本章深入探讨了源代码组织和包的概念。读者了解了 Go 工作空间和导入路径。读者还介绍了包的创建以及如何导入包以实现代码重用。本章介绍了诸如导入成员的可见性和包初始化等机制。章节的最后部分讨论了从打包代码创建可执行 Go 程序所需的步骤。

这是一个内容丰富的章节，理应如此，以公正地对待 Go 中包创建和管理这样广泛的主题。下一章将回到 Go 类型讨论，详细介绍了复合类型，如数组、切片、结构体和映射。
