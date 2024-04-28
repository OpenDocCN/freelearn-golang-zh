# 第六章：Go 包和程序

第五章, *Go 中的函数*涵盖了函数，这是代码组织的基本抽象级别，使代码可寻址和可重用。本章将继续讨论围绕 Go 包展开的抽象层次。正如将在这里详细介绍的那样，包是存储在源代码文件中的语言元素的逻辑分组，可以共享和重用，如下面的主题所涵盖的：

+   Go 包

+   创建包

+   构建包

+   包可见性

+   导入包

+   包初始化

+   创建程序

+   远程包

# Go 包

与其他语言类似，Go 源代码文件被分组为可编译和可共享的单元，称为包。但是，所有 Go 源文件必须属于一个包（没有默认包的概念）。这种严格的方法使得 Go 可以通过偏爱惯例而不是配置来保持其编译规则和包解析规则简单。让我们深入了解包的基础知识，它们的创建、使用和推荐做法。

## 理解 Go 包

在我们深入讨论包的创建和使用之前，至关重要的是从高层次上理解包的概念，以帮助引导后续的讨论。Go 包既是代码组织的物理单元，也是逻辑单元，用于封装可以重用的相关概念。按照惯例，存储在同一目录中的一组源文件被认为是同一个包的一部分。以下是一个简单的目录树示例，其中每个目录代表一个包，包含一些源代码：

```go
 foo
 ├── blat.go
 └── bazz
 ├── quux.go
 └── qux.go 

```

golang.fyi/ch06-foo

虽然不是必需的，但是建议按照惯例，在每个源文件中设置包的名称与文件所在目录的名称相匹配。例如，源文件`blat.go`被声明为`foo`包的一部分，因为它存储在名为`foo`的目录中，如下面的代码所示：

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

文件`quux.go`和`qux.go`都是`bazz`包的一部分，因为它们位于具有该名称的目录中，如下面的代码片段所示：

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

在讨论包时理解的另一个重要概念是*Go 工作区*。工作区只是一个任意的目录，用作在某些任务（如编译）期间解析包的命名空间。按照惯例，Go 工具期望工作区目录中有三个特定命名的子目录：`src`、`pkg`和`bin`。这些子目录分别存储 Go 源文件以及所有构建的包构件。

建立一个静态目录位置，将 Go 包放在一起具有以下优势：

+   简单设置，几乎没有配置

+   通过将代码搜索减少到已知位置来实现快速编译

+   工具可以轻松创建代码和包构件的源图

+   从源代码自动推断和解析传递依赖关系

+   项目设置可以是可移植的，并且易于分发

以下是我笔记本电脑上 Go 工作区的部分（和简化的）树状布局，其中突出显示了三个子目录`bin`、`pkg`和`src`：

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

示例工作区目录

+   `bin`：这是一个自动生成的目录，用于存储编译的 Go 可执行文件（也称为程序或命令）。当 Go 工具编译和安装可执行包时，它们被放置在此目录中。前面的示例工作区显示了两个列出的二进制文件`circ`和`golint`。建议将此目录添加到操作系统的`PATH`环境变量中，以使您的命令在本地可用。

+   `pkg`：这个目录也是自动生成的，用于存储构建的包构件。当 Go 工具构建和安装非可执行包时，它们被存储为对象文件（带有`.a`后缀）在子目录中，子目录的名称模式基于目标操作系统和架构。在示例工作区中，对象文件被放置在`linux_amd64`子目录下，这表明该目录中的对象文件是为运行在 64 位架构上的 Linux 操作系统编译的。

+   `src`：这是一个用户创建的目录，用于存储 Go 源代码文件。`src`下的每个子目录都映射到一个包。*src*是解析所有导入路径的根目录。Go 工具搜索该目录以解析代码中引用的包，这些引用在编译或其他依赖源路径的活动中。上图中的示例工作区显示了两个包：`github.com/golang/lint/golint/`和`github.com/vladimirvivien/learning-go/ch06/current`。

### 注意

您可能会对工作区示例中显示的包路径中的`github.com`前缀感到疑惑。值得注意的是，包目录没有命名要求（请参阅*命名包*部分）。包可以有任意的名称。但是，Go 建议遵循一些约定，这有助于全局命名空间解析和包组织。

## 创建工作区

创建工作区就像设置一个名为`GOPATH`的操作系统环境一样简单，并将其分配给工作区目录的根路径。例如，在 Linux 机器上，工作区的根目录为`/home/username/Go`，工作区将被设置为：

```go
$> export GOPATH=/home/username/Go 

```

在设置`GOPATH`环境变量时，可以指定存储包的多个位置。每个目录由操作系统相关的路径分隔符分隔（换句话说，Linux/Unix 使用冒号，Windows 使用分号），如下所示：

```go
$> export GOPATH=/home/myaccount/Go;/home/myaccount/poc/Go

```

当解析包名称时，Go 工具将搜索`GOPATH`中列出的所有位置。然而，Go 编译器只会将编译后的文件，如对象和二进制文件，存储在分配给`GOPATH`的第一个目录位置中。

### 注意

通过简单设置操作系统环境变量来配置工作区具有巨大的优势。它使开发人员能够在编译时动态设置工作区，以满足某些工作流程要求。例如，开发人员可能希望在合并代码之前测试未经验证的代码分支。他或她可能希望设置一个临时工作区来构建该代码，方法如下（Linux）：`$> GOPATH=/temporary/go/workspace/path go build`

## 导入路径

在继续设置和使用包的详细信息之前，最后一个重要概念要涵盖的是*导入路径*的概念。每个包的相对路径，位于工作区路径`$GOPATH/src`下，构成了一个全局标识符，称为包的`导入路径`。这意味着在给定的工作区中，没有两个包可以具有相同的导入路径值。

让我们回到之前的简化目录树。例如，如果我们将工作区设置为某个任意路径值，如`GOPATH=/home/username/Go`：

```go
/home/username/Go
└── foo
 ├── ablt.go
 └── bazz
 ├── quux.go
 └── qux.go 

```

从上面示例的工作区中，包的目录路径映射到它们各自的导入路径，如下表所示：

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

到目前为止，本章已经涵盖了 Go 包的基本概念；现在是时候深入了解并查看包含在包中的 Go 代码的创建。Go 包的主要目的之一是将常见逻辑抽象出来并聚合到可共享的代码单元中。在本章的前面提到，一个目录中的一组 Go 源文件被认为是一个包。虽然这在技术上是正确的，但是关于 Go 包的概念还不仅仅是将一堆文件放在一个目录中。

为了帮助说明我们的第一个包的创建，我们将利用在[github.com/vladimirvivien/learning-go/ch06](https://github.com/vladimirvivien/learning-go/ch06)中找到的示例源代码。该目录中的代码定义了一组函数，用于使用*欧姆定律*计算电气值。以下显示了组成示例包的目录布局（假设它们保存在某个工作区目录`$GOPATH/src`中）：

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

Ohm's Law 示例的包布局

在上述目录中，每个目录都包含一个或多个 Go 源代码文件，用于定义和实现函数以及其他源代码元素，这些元素将被组织成包并可重复使用。以下表格总结了从前面的工作区布局中提取的导入路径和包信息：

| **导入路径** | **包** |
| --- | --- |
| "github.com/vladimirvivien/learning-go/ch06/**current**" | `current` |
| "github.com/vladimirvivien/learning-go/ch06/**power**" | `power` |
| "github.com/vladimirvivien/learning-go/ch06/**power/ir**" | `ir` |
| "github.com/vladimirvivien/learning-go/ch06/**power/vr**" | `vr` |
| "github.com/vladimirvivien/learning-go/ch06/**resistor**" | `resistor` |
| "github.com/vladimirvivien/learning-go/ch06/**volt**" | `volt` |

虽然没有命名要求，但是将包目录命名为反映其各自目的的名称是明智的。从前面的表格中，每个示例中的包都被命名为代表电气概念的名称，例如 current、power、resistor 和 volt。*包命名*部分将详细介绍包命名约定。

## 声明包

Go 源文件必须声明自己属于一个包。这是使用`package`子句完成的，作为 Go 源文件中的第一个合法语句。声明的包由`package`关键字后跟一个名称标识符组成。以下显示了`volt`包中的源文件`volt.go`：

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

源文件中的包标识符可以设置为任意值。与 Java 不同，包的名称不反映源文件所在的目录结构。虽然对于包名称没有要求，但是将包标识符命名为与文件所在目录相同的约定是被接受的。在我们之前的源代码清单中，包被声明为标识符`volt`，因为该文件存储在*volt*目录中。

## 多文件包

一个包的逻辑内容（源代码元素，如类型、函数、变量和常量）可以在多个 Go 源文件中物理扩展。一个包目录可以包含一个或多个 Go 源文件。例如，在下面的示例中，包`resistor`被不必要地分割成几个 Go 源文件，以说明这一点：

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

包中的每个文件必须具有相同的名称标识符的包声明（在本例中为`resistor`）。Go 编译器将从所有源文件中的所有元素中拼接出一个逻辑单元，形成一个可以被其他包使用的单一范围内的逻辑单元。

需要指出的是，如果给定目录中所有源文件的包声明不相同，编译将失败。这是可以理解的，因为编译器期望目录中的所有文件都属于同一个包。

## 命名包

如前所述，Go 期望工作区中的每个包都有一个唯一的完全限定的导入路径。您的程序可以拥有任意多的包，您的包结构可以在工作区中深入到您喜欢的程度。然而，惯用的 Go 规定了一些关于包的命名和组织的**规则**，以使创建和使用包变得简单。

### 使用全局唯一的命名空间

首先，在全局上下文中，完全限定您的包的导入路径是一个好主意，特别是如果您计划与他人共享您的代码。考虑以唯一标识您或您的组织的命名空间方案开始您的导入路径的名称。例如，公司*Acme, Inc.*可能选择以`acme.com/apps`开头命名他们所有的 Go 包名称。因此，一个包的完全限定导入路径将是`"acme.com/apps/foo/bar"`。

### 注意

在本章的后面，我们将看到如何在集成 Go 与 GitHub 等源代码存储库服务时使用包导入路径。

### 为路径添加上下文

接下来，当您为您的包设计一个命名方案时，使用包的路径为您的包名称添加上下文。名称中的上下文应该从左到右开始通用，然后变得更具体。例如，让我们参考电源包的导入路径（来自之前的示例）。电源值的计算分为三个子包，如下所示：

+   `github.com/vladimirvivien/learning-go/ch06/**power**`

+   `github.com/vladimirvivien/learning-go/ch06/**power/ir**`

+   `github.com/vladimirvivien/learning-go/ch06/**power/vr**`

父路径`power`包含具有更广泛上下文的包成员。子包`ir`和`vr`包含更具体的成员，具有更窄的上下文。这种命名模式在 Go 中被广泛使用，包括内置包，如以下所示：

+   `crypto/md5`

+   `net/http`

+   `net/http/httputil`

+   `reflect`

请注意，一个包深度为一是一个完全合法的包名称（参见`reflect`），只要它能捕捉上下文和它所做的本质。同样，保持简单。避免在您的命名空间内将您的包嵌套超过三层的诱惑。如果您是一个习惯于长嵌套包名称的 Java 开发人员，这种诱惑将特别强烈。

### 使用简短的名称

当审查内置的 Go 包名称时，您会注意到一个事实，即与其他语言相比，名称的简洁性。在 Go 中，包被认为是实现一组紧密相关功能的代码集合。因此，您的包的导入路径应该简洁，并反映出它们的功能，而不会过长。我们的示例源代码通过使用诸如 volt、power、resistance、current 等简短名称来命名包目录，充分体现了这一点。在各自的上下文中，每个目录名称都准确说明了包的功能。

在 Go 的内置包中严格遵守了简短名称规则。例如，以下是 Go 内置包中的几个包名称：`log`、`http`、`xml`和`zip`。每个名称都能够清楚地识别包的目的。

### 注意

短包名称有助于减少在较大代码库中的击键次数。然而，拥有短而通用的包名称也有一个缺点，即容易发生导入路径冲突，即在大型项目中的开发人员（或开源库的开发人员）可能最终在他们的代码中使用相同的流行名称（换句话说，`log`、`util`、`db`等）。正如我们将在本章后面看到的那样，这可以通过使用`命名`导入路径来处理。

# 构建包

通过应用某些约定和合理的默认值，Go 工具减少了编译代码的复杂性。虽然完整讨论 Go 的构建工具超出了本节（或本章）的范围，但了解`build`和`install`工具的目的和用法是有用的。一般来说，使用构建和安装工具的方式如下：

*$> go build [<package import path>]*

`import path`可以明确提供或完全省略。`build`工具接受`import path`，可以表示为完全限定或相对路径。在正确设置的工作区中，以下是从前面的示例中编译包`volt`的等效方式：

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

上面的`go build`命令将编译在目录`volt`中找到的所有 Go 源文件及其依赖项。此外，还可以使用通配符参数构建给定目录中的所有包和子包，如下所示：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go build ./...

```

前面的内容将构建在目录`$GOPATH/src/github.com/vladimirvivien/learning-go/ch06`中找到的所有包和子包。

## 安装一个包

默认情况下，构建命令将其结果输出到一个工具生成的临时目录中，在构建过程完成后会丢失。要实际生成可用的构件，必须使用`install`工具来保留已编译的对象文件的副本。

`install`工具与构建工具具有完全相同的语义：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go install ./volt

```

除了编译代码，它还将结果保存并输出到工作区位置`$GOPATH/pkg`，如下所示：

```go
$GOPATH/pkg/linux_amd64/github.com/vladimirvivien/learning-go/
└── ch06
 └── volt.a

```

生成的对象文件（带有`.a`扩展名）允许包在工作区中被重用和链接到其他包中。在本章的后面，我们将讨论如何编译可执行程序。

# 包可见性

无论声明为包的一部分的源文件数量如何，所有在包级别声明的源代码元素（类型、变量、常量和函数）都共享一个公共作用域。因此，编译器不允许在整个包中重新声明元素标识符超过一次。让我们使用以下代码片段来说明这一点，假设两个源文件都是同一个包`$GOPATH/src/foo`的一部分：

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

foo/file1.go |

```go
package foo   

var bar struct{   
  x, y int   
}   

func quux() {   
  bar = bar * bar   
}   

```

foo/file2.go |

非法的变量标识符重新声明

尽管它们在两个不同的文件中，但在 Go 中使用标识符`bar`声明变量是非法的。由于这些文件是同一个包的一部分，两个标识符具有相同的作用域，因此会发生冲突。

函数标识符也是如此。Go 不支持在相同作用域内重载函数名称。因此，无论函数的签名如何，使用函数标识符超过一次都是非法的。如果我们假设以下代码出现在同一包内的两个不同源文件中，则以下代码片段将是非法的：

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

foo/file1.go |

```go
package foo   

var (   
  fooVal int = 12   
)   

func qux (inc int) int {   
  return fooVal += inc   
}   

```

foo/file1.go |

非法的函数标识符重新声明

在前面的代码片段中，函数名标识符`qux`被使用了两次。即使这两个函数具有不同的签名，编译器也会失败。唯一的解决方法是更改名称。

## 包成员可见性

包的有用性在于其能够将其源元素暴露给其他包。控制包元素的可见性很简单，遵循这个规则：*大写标识符会自动导出*。这意味着任何具有大写标识符的类型、变量、常量或函数都会自动从声明它的包之外可见。

参考之前描述的欧姆定律示例，以下说明了来自包`resistor`（位于[github.com/vladimirvivien/learning-go/ch06/resistor](https://github.com/vladimirvivien/learning-go/ch06/resistor)）的功能：

| **代码** | **描述** |
| --- | --- |

|

```go
package resistor   

func R(v, i float64) float64 {   
   return v / i   
}   

```

| 函数`R`自动导出，并且可以从其他包中访问：`resistor.R()` |
| --- |

|

```go
package resistor   

func recip(val float64) float64 {   
   return 1 / val   
}   

```

| 函数标识符`recip`全部小写，因此未导出。虽然在其自己的范围内可访问，但该函数将无法从其他包中可见。 |
| --- |

值得重申的是，同一个包内的成员始终对彼此可见。在 Go 中，没有复杂的可见性结构，比如私有、友元、默认等，这使得开发人员可以专注于正在实现的解决方案，而不是对可见性层次进行建模。

# 导入包

到目前为止，您应该对包是什么，它的作用以及如何创建包有了很好的理解。现在，让我们看看如何使用包来导入和重用其成员。正如您在其他几种语言中所发现的那样，关键字`import`用于从外部包中导入源代码元素。它允许导入源访问导入包中的导出元素（请参阅本章前面的*包范围和可见性*部分）。导入子句的一般格式如下：

*import [包名称标识符] "<导入路径>"*

请注意，导入路径必须用双引号括起来。`import`语句还支持可选的包标识符，可用于显式命名导入的包（稍后讨论）。导入语句也可以写成导入块的形式，如下所示。这在列出两个或更多导入包的情况下很有用：

*import (*

*[包名称标识符] "<导入路径>"*

*)*

以下源代码片段显示了先前介绍的欧姆定律示例中的导入声明块：

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

通常省略导入包的名称标识符，如上所示。然后，Go 将导入路径的最后一个目录的名称作为导入包的名称标识符，如下表所示，对于某些包：

| **导入路径** | **包名称** |
| --- | --- |
| `flag` | `flag` |
| `github.com/vladimirvivien/learning-go/ch06/current` | `current` |
| `github.com/vladimirvivien/learning-go/ch06/power/ir` | `ir` |
| `github.com/vladimirvivien/learning-go/ch06/volt` | `volt` |

点符号用于访问导入包的导出成员。例如，在下面的源代码片段中，从导入包`"github.com/vladimirvivien/learning-go/ch06/volt"`调用了方法`volt.V()`：

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

如前所述，`import`声明可以显式为导入声明一个名称标识符，如下面的导入片段所示：

```go
import res "github.com/vladimirvivien/learning-go/ch06/resistor"
```

按照前面描述的格式，名称标识符放在导入路径之前，如前面的片段所示。命名包可以用作缩短或自定义包名称的一种方式。例如，在一个大型源文件中，有大量使用某个包的情况下，这可以是一个很好的功能，可以减少按键次数。

给包分配一个名称也是避免给定源文件中的包标识符冲突的一种方式。可以想象导入两个或更多的包，具有不同的导入路径，解析为相同的包名称。例如，您可能需要使用来自不同库的两个不同日志系统记录信息，如下面的代码片段所示：

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

```

如前面的片段所示，两个日志包默认都将解析为名称标识符`"logger"`。为了解决这个问题，至少其中一个导入的包必须分配一个名称标识符来解决名称冲突。在上面的例子中，两个导入路径都被命名为有意义的名称，以帮助代码理解。

## 点标识符

一个包可以选择将点（句号）分配为它的标识符。当一个`import`语句使用点标识符（`.`）作为导入路径时，它会导致导入包的成员与导入包的作用域合并。因此，导入的成员可以在不添加额外限定符的情况下被引用。因此，如果在以下源代码片段中使用点标识符导入了包`logger`，那么在访问 logger 包的导出成员函数`SubmitError`时，包名被省略了：

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

虽然这个特性可以帮助减少重复的按键，但这并不是一种鼓励的做法。通过合并包的作用域，更有可能遇到标识符冲突。

## 空白标识符

当导入一个包时，要求在导入的代码中至少引用其成员之一。如果未能这样做，将导致编译错误。虽然这个特性有助于简化包依赖关系的解析，但在开发代码的早期阶段，这可能会很麻烦。

使用空白标识符（类似于变量声明）会导致编译器绕过此要求。例如，以下代码片段导入了内置包`fmt`；但是，在随后的源代码中从未使用过它：

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

空白标识符的一个常见用法是为了加载包的副作用。这依赖于包在导入时的初始化顺序（请参阅下面的*包初始化*部分）。使用空白标识符将导致导入的包在没有引用其成员的情况下被初始化。这在需要在不引起注意的情况下运行某些初始化序列的代码中使用。

# 包初始化

当导入一个包时，它会在其成员准备好被使用之前经历一系列的初始化序列。包级变量的初始化是使用依赖分析来进行的，依赖于词法作用域解析，这意味着变量是基于它们的声明顺序和它们相互引用的解析来初始化的。例如，在以下代码片段中，包`foo`中的解析变量声明顺序将是`a`、`y`、`b`和`x`：

```go
package foo 
var x = a + b(a) 
var a = 2 
var b = func(i int) int {return y * i} 
var y = 3 

```

Go 还使用了一个名为`init`的特殊函数，它不接受任何参数，也不返回任何结果值。它用于封装在导入包时调用的自定义初始化逻辑。例如，以下源代码显示了在`resistor`包中使用的`init`函数来初始化函数变量`Rpi`：

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

在前面的代码中，`init`函数在包级变量初始化之后被调用。因此，`init`函数中的代码可以安全地依赖于声明的变量值处于稳定状态。`init`函数在以下方面是特殊的：

+   一个包可以定义多个`init`函数

+   您不能直接在运行时访问声明的`init`函数

+   它们按照它们在每个源文件中出现的词法顺序执行

+   `init`函数是将逻辑注入到在任何其他函数或方法之前执行的包中的一种很好的方法。

# 创建程序

到目前为止，在本书中，您已经学会了如何创建和捆绑 Go 代码作为可重用的包。但是，包本身不能作为独立的程序执行。要创建一个程序（也称为命令），您需要取一个包，并定义一个执行入口，如下所示：

+   声明（至少一个）源文件作为名为`main`的特殊包的一部分

+   声明一个名为`main()`的函数作为程序的入口点

函数`main`不接受任何参数，也不返回任何值。以下是`main`包的缩写源代码，用于之前的 Ohm 定律示例中。它使用了 Go 标准库中的`flag`包来解析格式为`flag`的程序参数：

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

前面的清单显示了`main`包的源代码以及`main`函数的实现，当程序运行时将执行该函数。Ohm's Law 程序接受指定要执行的电气操作的命令行参数（请参阅下面的*访问程序参数*部分）。`init`函数用于初始化程序标志值的解析。`main`函数设置为一个大的开关语句块，以根据所选的标志选择要执行的适当操作。

## 访问程序参数

当程序被执行时，Go 运行时将所有命令行参数作为一个切片通过包变量`os.Args`提供。例如，当执行以下程序时，它会打印传递给程序的所有命令行参数：

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

当使用显示的参数调用程序时，以下是程序的输出：

```go
$> go run hello.go hello world how are you?
/var/folders/.../exe/hello
hello
world
how
are
you?

```

请注意，程序名称后面放置的命令行参数`"hello world how are you?"`被拆分为一个以空格分隔的字符串。切片`os.Args`中的位置 0 保存了程序二进制路径的完全限定名称。切片的其余部分分别存储了字符串中的每个项目。

Go 标准库中的`flag`包在内部使用此机制来提供已知为标志的结构化命令行参数的处理。在前面列出的 Ohm's Law 示例中，`flag`包用于解析几个标志，如以下源代码片段中所示（从前面的完整清单中提取）：

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

```

代码片段显示了`init`函数用于解析和初始化预期的标志`"v"`、`"i"`、`"p"`和`"op"`（在运行时，每个标志都以减号开头）。`flag`包中的初始化函数设置了预期的类型、默认值、标志描述以及用于存储标志解析值的位置。`flag`包还支持特殊标志"help"，用于提供有关每个标志的有用提示。

`flag.Parse()`在`main`函数中用于开始解析作为命令行提供的任何标志的过程。例如，要计算具有 12 伏特和 300 欧姆的电路的电流，程序需要三个标志，并产生如下输出：

```go
$> go run main.go -op I -v 12 -r 300
I = 12.00 / 300.00 = 0.04 amps

```

## 构建和安装程序

构建和安装 Go 程序遵循与构建常规包相同的程序（如在*构建和安装包*部分中讨论的）。当您构建可执行的 Go 程序的源文件时，编译器将通过传递链接`main`包中声明的所有依赖项来生成可执行的二进制文件。构建工具将默认使用与包含 Go 程序源文件的目录相同的名称命名输出二进制文件。

例如，在 Ohm's Law 示例中，位于目录`github.com/vladimirvivien/learning-go/ch06`中的文件`main.go`被声明为`main`包的一部分。程序可以按以下方式构建：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go build .

```

当构建`main.go`源文件时，构建工具将生成一个名为`ch06`的二进制文件，因为程序的源代码位于具有该名称的目录中。您可以使用输出标志`-o`来控制二进制文件的名称。在以下示例中，构建工具将创建一个名为`ohms`的二进制文件。

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go build -o ohms

```

最后，安装 Go 程序的方法与使用 Go `install`命令安装常规包的方法完全相同：

```go
$> cd $GOPATH/src/github.com/vladimirvivien/learning-go/ch06
$> go install .

```

使用 Go install 命令安装程序时，如果需要，将构建该程序，并将生成的二进制文件保存在`$GOPAHT/bin`目录中。将工作区`bin`目录添加到您的操作系统的`$PATH`环境变量中，将使您的 Go 程序可供执行。

### 注意

Go 生成的程序是静态链接的二进制文件。它们不需要满足任何额外的依赖关系就可以运行。但是，Go 编译的二进制文件包括 Go 运行时。这是一组处理功能的操作，如垃圾回收、类型信息、反射、goroutines 调度和 panic 管理。虽然可比的 C 程序会小上几个数量级，但 Go 的运行时带有使 Go 变得愉快的工具。

# 远程软件包

Go 附带的工具之一允许程序员直接从远程源代码存储库检索软件包。默认情况下，Go 可以轻松支持与以下版本控制系统的集成：

+   Git（`git`，[`git-scm.com/`](http://git-scm.com/)）

+   Mercurial（`hg`，[`www.mercurial-scm.org/`](https://www.mercurial-scm.org/)）

+   Subversion（`svn`，[`subversion.apache.org/`](http://subversion.apache.org/)）

+   Bazaar（`bzr`，[`bazaar.canonical.com/`](http://bazaar.canonical.com/)）

### 注意

为了让 Go 从远程存储库中拉取软件包源代码，您必须在操作系统的执行路径上安装该版本控制系统的客户端作为命令。在幕后，Go 启动客户端与源代码存储库服务器进行交互。

`get`命令行工具允许程序员使用完全合格的项目路径作为软件包的导入路径来检索远程软件包。一旦软件包被下载，就可以在本地源文件中导入以供使用。例如，如果您想要包含前面片段中 Ohm's Law 示例中的一个软件包，您可以从命令行发出以下命令：

```go
$> go get github.com/vladimirvivien/learning-go/ch06/volt

```

`go get`工具将下载指定的导入路径以及所有引用的依赖项。然后，该工具将在`$GOPATH/pkg`中构建和安装软件包工件。如果`import`路径恰好是一个程序，go get 还将在`$GOPATH/bin`中生成二进制文件，以及在`$GOPATH/pkg`中引用的任何软件包。

# 总结

本章详细介绍了源代码组织和软件包的概念。读者了解了 Go 工作区和导入路径。读者还了解了如何创建软件包以及如何导入软件包以实现代码的可重用性。本章介绍了诸如导入成员的可见性和软件包初始化之类的机制。本章的最后部分讨论了从打包代码创建可执行 Go 程序所需的步骤。

这是一个冗长的章节，理所当然地对 Go 中软件包创建和管理这样一个广泛的主题进行了公正的处理。下一章将详细讨论复合类型，如数组、切片、结构和映射，回到 Go 类型讨论。
