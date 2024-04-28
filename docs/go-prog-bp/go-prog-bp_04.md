# 第四章：用于查找域名的命令行工具

我们在前几章中构建的聊天应用程序已经准备好在互联网上大放异彩，但在邀请朋友加入对话之前，我们需要为其在互联网上找一个家。在邀请朋友加入对话之前，我们需要选择一个有效、引人注目且可用的域名，以便将其指向运行我们 Go 代码的服务器。我们将开发一些命令行工具，而不是在我们喜爱的域名提供商前面花费数小时尝试不同的名称，这些工具将帮助我们找到合适的域名。在这个过程中，我们将看到 Go 标准库如何允许我们与终端和其他正在执行的应用程序进行交互，以及探索一些构建命令行程序的模式和实践。

在本章中，您将学到：

+   如何使用尽可能少的代码文件构建完整的命令行应用程序

+   如何确保我们构建的工具可以使用标准流与其他工具组合

+   如何与简单的第三方 JSON RESTful API 进行交互

+   如何在 Go 代码中利用标准输入和输出管道

+   如何从流式源中逐行读取

+   如何构建 WHOIS 客户端来查找域信息

+   如何存储和使用敏感或部署特定信息的环境变量

# 命令行工具的管道设计

我们将构建一系列命令行工具，这些工具使用标准流（`stdin`和`stdout`）与用户和其他工具进行通信。每个工具将通过标准输入管道逐行接收输入，以某种方式处理它，然后通过标准输出管道逐行打印输出，以供下一个工具或用户使用。

默认情况下，标准输入连接到用户的键盘，标准输出打印到运行命令的终端；但是，可以使用重定向元字符进行重定向。可以通过将输出重定向到 Windows 上的`NUL`或 Unix 机器上的`/dev/null`来丢弃输出，也可以将其重定向到文件，这将导致输出保存到磁盘。或者，您可以使用`|`管道字符将一个程序的输出管道到另一个程序的输入；我们将利用这个特性来连接我们的各种工具。例如，您可以通过以下代码将一个程序的输出管道到终端中的另一个程序的输入：

```go
one | two
```

我们的工具将使用字符串行的形式进行操作，其中每行（由换行符分隔）代表一个字符串。当没有任何管道重定向时，我们将能够直接与程序进行交互，使用默认的输入和输出，这在测试和调试代码时将非常有用。

# 五个简单的程序

在本章中，我们将构建五个小程序，最后将它们组合在一起。程序的主要特点如下：

+   **Sprinkle**：该程序将添加一些适合网络的词语，以增加找到可用域名的机会

+   **Domainify**：该程序将确保单词适合作为域名，方法是删除不可接受的字符，用连字符替换空格，并在末尾添加适当的顶级域（如`.com`和`.net`）

+   **Coolify**：该程序将通过调整元音字母将无聊的普通单词变成 Web 2.0

+   **Synonyms**：该程序将使用第三方 API 查找同义词

+   **可用**：该程序将使用适当的 WHOIS 服务器检查域名是否可用

五个程序在一个章节中可能看起来很多，但不要忘记在 Go 中整个程序可以有多小。

## Sprinkle

我们的第一个程序通过添加一些糖词来增加找到可用名称的几率。许多公司使用这种方法来保持核心消息一致，同时又能够负担得起`.com`域名。例如，如果我们传入单词`chat`，它可能输出`chatapp`；或者，如果我们传入`talk`，我们可能得到`talk time`。

Go 的`math/rand`包允许我们摆脱计算机的可预测性，为我们的程序过程提供机会或机会，并使我们的解决方案感觉比实际更智能一些。

为了使我们的 Sprinkle 程序工作，我们将：

+   使用特殊常量定义转换数组，以指示原始单词将出现在哪里

+   使用`bufio`包从`stdin`扫描输入，并使用`fmt.Println`将输出写入`stdout`

+   使用`math/rand`包来随机选择要应用于单词的转换，比如在单词后添加"app"或在术语前添加"get"

### 提示

我们所有的程序都将驻留在`$GOPATH/src`目录中。例如，如果您的`GOPATH`是`~/Work/projects/go`，您将在`~/Work/projects/go/src`文件夹中创建您的程序文件夹。

在`$GOPATH/src`目录中，创建一个名为`sprinkle`的新文件夹，并添加一个包含以下代码的`main.go`文件：

```go
package main
import (
  "bufio"
  "fmt"
  "math/rand"
  "os"
  "strings"
  "time"
)
const otherWord = "*"
var transforms = []string{
  otherWord,
  otherWord,
  otherWord,
  otherWord,
  otherWord + "app",
  otherWord + "site",
  otherWord + "time",
  "get" + otherWord,
  "go" + otherWord,
  "lets " + otherWord,
}
func main() {
  rand.Seed(time.Now().UTC().UnixNano())
  s := bufio.NewScanner(os.Stdin)
  for s.Scan() {
    t := transforms[rand.Intn(len(transforms))]
    fmt.Println(strings.Replace(t, otherWord, s.Text(), -1))
  }
}
```

从现在开始，假定您将自行解决适当的`import`语句。如果需要帮助，请参考附录中提供的提示，*稳定的 Go 环境的良好实践*。

前面的代码代表了我们完整的 Sprinkle 程序。它定义了三件事：一个常量，一个变量，以及作为 Sprinkle 入口点的必需的`main`函数。`otherWord`常量字符串是一个有用的标记，允许我们指定原始单词应出现在我们可能的每个转换中的位置。它让我们编写诸如`otherWord+"extra"`的代码，这清楚地表明，在这种特殊情况下，我们想在原始单词的末尾添加单词 extra。

可能的转换存储在我们声明为字符串切片的`transforms`变量中。在前面的代码中，我们定义了一些不同的转换，比如在单词末尾添加`app`或在单词前添加`lets`。随意添加一些更多的转换；越有创意，越好。

在`main`函数中，我们首先使用当前时间作为随机种子。计算机实际上无法生成随机数，但更改随机算法的种子数字会产生它可以的幻觉。我们使用纳秒级的当前时间，因为每次运行程序时它都是不同的（前提是系统时钟在每次运行之前没有被重置）。

然后，我们创建一个`bufio.Scanner`对象（称为`bufio.NewScanner`），并告诉它从`os.Stdin`读取输入，表示标准输入流。由于我们总是要从标准输入读取并写入标准输出，这将是我们五个程序中的常见模式。

### 提示

`bufio.Scanner`对象实际上将`io.Reader`作为其输入源，因此我们可以在这里使用各种类型。如果您为此代码编写单元测试，可以为扫描器指定自己的`io.Reader`，从中读取，而无需担心模拟标准输入流的需要。

作为默认情况，扫描器允许我们逐个读取由定义的分隔符分隔的字节块，例如回车和换行符。我们可以为扫描器指定自己的分割函数，或者使用标准库中内置的选项之一。例如，有`bufio.ScanWords`可以通过在空格上断开而不是换行符上断开来扫描单个单词。由于我们的设计规定每行必须包含一个单词（或短语），默认的逐行设置是理想的。

对`Scan`方法的调用告诉扫描器读取输入的下一块字节（下一行），并返回一个`bool`值，指示它是否找到了任何内容。这就是我们能够将其用作`for`循环的条件的方式。只要有内容可以处理，`Scan`就会返回`true`，并执行`for`循环的主体，当`Scan`到达输入的末尾时，它返回`false`，循环就会被打破。已选择的字节存储在扫描器的`Bytes`方法中，我们使用的方便的`Text`方法将`[]byte`切片转换为字符串。

在`for`循环内（对于每行输入），我们使用`rand.Intn`从`transforms`切片中选择一个随机项，并使用`strings.Replace`将原始单词插入到`otherWord`字符串出现的位置。最后，我们使用`fmt.Println`将输出打印到默认标准输出流。

让我们构建我们的程序并玩耍一下：

```go

go build –o sprinkle

./sprinkle

```

一旦程序运行，由于我们没有输入任何内容，或者指定了一个来源来读取内容，我们将使用默认行为，从终端读取用户输入。输入`chat`并按回车。我们代码中的扫描器注意到单词末尾的换行符，并运行转换代码，输出结果。例如，如果您多次输入`chat`，您可能会看到类似的输出：

```go

chat

go chat

chat

lets chat

chat

chat app

```

Sprinkle 永远不会退出（意味着`Scan`方法永远不会返回`false`来中断循环），因为终端仍在运行；在正常执行中，输入管道将被生成输入的任何程序关闭。要停止程序，请按*Ctrl* + *C*。

在我们继续之前，让我们尝试运行 Sprinkle，指定一个不同的输入源，我们将使用`echo`命令生成一些内容，并使用管道字符将其输入到我们的 Sprinkle 程序中：

```go

echo "chat" | ./sprinkle

```

程序将随机转换单词，打印出来，然后退出，因为`echo`命令在终止和关闭管道之前只生成一行输入。

我们已经成功完成了我们的第一个程序，它有一个非常简单但有用的功能，我们将会看到。

### 练习-可配置的转换

作为额外的任务，不要像我们所做的那样将`transformations`数组硬编码，看看是否可以将其外部化到文本文件或数据库中。

## Domainify

从 Sprinkle 输出的一些单词包含空格和其他在域名中不允许的字符，因此我们将编写一个名为 Domainify 的程序，将一行文本转换为可接受的域段，并在末尾添加适当的**顶级域**（**TLD**）。在`sprinkle`文件夹旁边，创建一个名为`domainify`的新文件夹，并添加一个带有以下代码的`main.go`文件：

```go
package main
var tlds = []string{"com", "net"}
const allowedChars = "abcdefghijklmnopqrstuvwxyz0123456789_-"
func main() {
  rand.Seed(time.Now().UTC().UnixNano())
  s := bufio.NewScanner(os.Stdin)
  for s.Scan() {
    text := strings.ToLower(s.Text())
    var newText []rune
    for _, r := range text {
      if unicode.IsSpace(r) {
        r = '-'
      }
      if !strings.ContainsRune(allowedChars, r) {
        continue
      }
      newText = append(newText, r)
    }
    fmt.Println(string(newText) + "." +        
                tlds[rand.Intn(len(tlds))])
  }
}
```

您会注意到 Domainify 和 Sprinkle 程序之间的一些相似之处：我们使用`rand.Seed`设置随机种子，使用`NewScanner`方法包装`os.Stdin`读取器，并扫描每一行，直到没有更多的输入。

然后我们将文本转换为小写，并构建一个名为`newText`的`rune`类型的新切片。`rune`类型仅包含出现在`allowedChars`字符串中的字符，`strings.ContainsRune`让我们知道。如果`rune`是一个空格，我们通过调用`unicode.IsSpace`来确定，我们将其替换为连字符，这在域名中是可以接受的做法。

### 注意

在字符串上进行范围循环会返回每个字符的索引和`rune`类型，这是一个表示字符本身的数值（具体是`int32`）。有关符文、字符和字符串的更多信息，请参阅[`blog.golang.org/strings`](http://blog.golang.org/strings)。

最后，我们将`newText`从`[]rune`切片转换为字符串，并在打印之前在末尾添加`.com`或`.net`。

构建并运行 Domainify：

```go

go build –o domainify

./domainify

```

输入一些选项，看看`domainify`的反应如何：

+   `Monkey`

+   `Hello Domainify`

+   `"What's up?"`

+   `One (two) three!`

例如，`One (two) three!`可能产生`one-two-three.com`。

现在我们将组合 Sprinkle 和 Domainify 以使它们一起工作。在您的终端中，导航到`sprinkle`和`domainify`的父文件夹（可能是`$GOPATH/src`），并运行以下命令：

```go

./sprinkle/sprinkle | ./domainify/domainify

```

在这里，我们运行了 Sprinkle 程序并将输出导入 Domainify 程序。默认情况下，`sprinkle`使用终端作为输入，`domanify`输出到终端。再次尝试多次输入`chat`，注意输出与之前 Sprinkle 输出的类似，只是现在这些单词适合作为域名。正是这种程序之间的管道传输使我们能够组合命令行工具。

### 练习-使顶级域名可配置

仅支持`.com`和`.net`顶级域名相当受限。作为额外的任务，看看是否可以通过命令行标志接受 TLD 列表。

## Coolify

通常，像`chat`这样的常见单词的域名已经被占用，一个常见的解决方案是对单词中的元音进行处理。例如，我们可能删除`a`得到`cht`（实际上更不太可能可用），或者添加一个`a`得到`chaat`。虽然这显然对酷度没有实际影响，但它已经成为一种流行的，尽管略显过时的方式来获得仍然听起来像原始单词的域名。

我们的第三个程序 Coolify 将允许我们处理通过输入的单词的元音，并将修改后的版本写入输出。

在`sprinkle`和`domainify`旁边创建一个名为`coolify`的新文件夹，并创建带有以下代码的`main.go`代码文件：

```go
package main
const (
  duplicateVowel bool   = true
  removeVowel    bool   = false
) 
func randBool() bool {
  return rand.Intn(2) == 0
}
func main() {
  rand.Seed(time.Now().UTC().UnixNano())
  s := bufio.NewScanner(os.Stdin)
  for s.Scan() {
    word := []byte(s.Text())
    if randBool() {
      var vI int = -1
      for i, char := range word {
        switch char {
        case 'a', 'e', 'i', 'o', 'u', 'A', 'E', 'I', 'O', 'U':
          if randBool() {
            vI = i
          }
        }
      }
      if vI >= 0 {
        switch randBool() {
        case duplicateVowel:
          word = append(word[:vI+1], word[vI:]...)
        case removeVowel:
          word = append(word[:vI], word[vI+1:]...)
        }
      }
    }
    fmt.Println(string(word))
  }
}
```

虽然前面的 Coolify 代码看起来与 Sprinkle 和 Domainify 的代码非常相似，但它稍微复杂一些。在代码的顶部，我们声明了两个常量，`duplicateVowel`和`removeVowel`，这有助于使 Coolify 代码更易读。`switch`语句决定我们是复制还是删除元音。此外，使用这些常量，我们能够非常清楚地表达我们的意图，而不仅仅使用`true`或`false`。

然后我们定义`randBool`辅助函数，它只是通过要求`rand`包生成一个随机数，然后检查该数字是否为零来随机返回`true`或`false`。它将是`0`或`1`，因此它有 50/50 的机会成为`true`。

Coolify 的`main`函数的开始方式与 Sprinkle 和 Domainify 的`main`函数相同——通过设置`rand.Seed`方法并在执行循环体之前创建标准输入流的扫描器来执行每行输入的循环体。我们首先调用`randBool`来决定是否要改变一个单词，因此 Coolify 只会影响通过其中的一半单词。

然后我们遍历字符串中的每个符文，并寻找元音。如果我们的`randBool`方法返回`true`，我们将元音字符的索引保留在`vI`变量中。如果不是，我们将继续在字符串中寻找另一个元音，这样我们就可以随机选择单词中的元音，而不总是修改相同的元音。

一旦我们选择了一个元音，我们再次使用`randBool`来随机决定要采取什么行动。

### 注意

这就是有用的常量发挥作用的地方；考虑以下备用的 switch 语句：

```go
switch randBool() {
case true:
  word = append(word[:vI+1], word[vI:]...)
case false:
  word = append(word[:vI], word[vI+1:]...)
}
```

在上述代码片段中，很难判断发生了什么，因为`true`和`false`没有表达任何上下文。另一方面，使用`duplicateVowel`和`removeVowel`告诉任何阅读代码的人我们通过`randBool`的结果的意图。

切片后面的三个点使每个项目作为单独的参数传递给`append`函数。这是一种将一个切片附加到另一个切片的成语方式。在`switch`情况下，我们对切片进行一些操作，以便复制元音或完全删除它。我们重新切片我们的`[]byte`切片，并使用`append`函数构建一个由原始单词的部分组成的新单词。以下图表显示了我们在代码中访问字符串的哪些部分：

![Coolify](img/Image00010.jpg)

如果我们以`blueprints`作为示例单词的值，并假设我们的代码选择第一个`e`字符作为元音（所以`vI`是`3`），我们可以看到单词的每个新切片在这个表中代表什么：

| 代码 | 值 | 描述 |
| --- | --- | --- |
| `word[:vI+1]` | `blue` | 描述了从单词切片的开头到所选元音的切片。`+1`是必需的，因为冒号后面的值不包括指定的索引；它切片直到该值。 |
| `word[vI:]` | `eprints` | 描述了从所选元音开始并包括切片到切片的末尾。 |
| `word[:vI]` | `blu` | 描述了从单词切片的开头到所选元音之前的切片。 |
| `word[vI+1:]` | `prints` | 描述了从所选元音后的项目到切片的末尾。 |

修改单词后，我们使用`fmt.Println`将其打印出来。

让我们构建 Coolify 并玩一下，看看它能做什么：

```go

go build –o coolify

./coolify

```

当 Coolify 运行时，尝试输入`blueprints`，看看它会做出什么样的修改：

```go

blueprnts

bleprints

bluepriints

blueprnts

blueprints

bluprints

```

让我们看看 Coolify 如何与 Sprinkle 和 Domainify 一起玩，通过将它们的名称添加到我们的管道链中。在终端中，使用`cd`命令返回到父文件夹，并运行以下命令：

```go

./coolify/coolify | ./sprinkle/sprinkle | ./domainify/domainify

```

首先，我们将用额外的部分来调整一个单词，通过调整元音字母使其更酷，最后将其转换为有效的域名。尝试输入一些单词，看看我们的代码会做出什么建议。

## 同义词

到目前为止，我们的程序只修改了单词，但要真正使我们的解决方案生动起来，我们需要能够集成一个提供单词同义词的第三方 API。这使我们能够在保留原始含义的同时建议不同的域名。与 Sprinkle 和 Domainify 不同，同义词将为每个给定的单词写出多个响应。我们将这三个程序连接在一起的架构意味着这不是问题；事实上，我们甚至不必担心，因为这三个程序都能够从输入源中读取多行。

[bighughlabs.com](http://bighughlabs.com)的 Big Hugh Thesaurus 有一个非常干净简单的 API，允许我们进行一次 HTTP `GET`请求来查找同义词。

### 提示

如果将来我们使用的 API 发生变化或消失（毕竟，这是互联网！），您可以在[`github.com/matryer/goblueprints`](https://github.com/matryer/goblueprints)找到一些选项。

在使用 Big Hugh Thesaurus 之前，您需要一个 API 密钥，您可以通过在[`words.bighugelabs.com/`](http://words.bighugelabs.com/)注册该服务来获取。

### 使用环境变量进行配置

您的 API 密钥是一项敏感的配置信息，您不希望与他人分享。我们可以将其存储为代码中的`const`，但这不仅意味着我们不能在不分享密钥的情况下分享我们的代码（尤其是如果您喜欢开源项目），而且，也许更重要的是，如果密钥过期或者您想使用其他密钥，您将不得不重新编译您的项目。

更好的解决方案是使用环境变量来存储密钥，因为这样可以让您在需要时轻松更改它。您还可以为不同的部署设置不同的密钥；也许您在开发或测试中有一个密钥，而在生产中有另一个密钥。这样，您可以为代码的特定执行设置一个特定的密钥，这样您可以轻松地在不必更改系统级设置的情况下切换密钥。无论如何，不同的操作系统以类似的方式处理环境变量，因此如果您正在编写跨平台代码，它们是一个完美的选择。

创建一个名为`BHT_APIKEY`的新环境变量，并将您的 API 密钥设置为其值。

### 注意

对于运行 bash shell 的计算机，您可以修改您的`~/.bashrc`文件或类似文件，包括`export`命令，例如：

```go
export BHT_APIKEY=abc123def456ghi789jkl
```

在 Windows 计算机上，您可以转到计算机的属性并在**高级**部分中查找**环境变量**。

### 消费 web API

在 Web 浏览器中请求[`words.bighugelabs.com/apisample.php?v=2&format=json`](http://words.bighugelabs.com/apisample.php?v=2&format=json)会显示我们在查找单词 love 的同义词时 JSON 响应数据的结构。

```go
{
  "noun":{
    "syn":[
      "passion",
      "beloved",
      "dear"
    ]
  },
  "verb":{
    "syn":[
      "love",
      "roll in the hay",
      "make out"
    ],
    "ant":[
      "hate"
    ]
  }
}
```

真正的 API 返回的实际单词比这里打印的要多得多，但结构才是重要的。它表示一个对象，其中键描述了单词类型（动词、名词等），值是包含在`syn`或`ant`（分别表示同义词和反义词）上的字符串数组的对象；这就是我们感兴趣的同义词。

要将这个 JSON 字符串数据转换成我们在代码中可以使用的东西，我们必须使用`encoding/json`包中的功能将其解码为我们自己的结构。因为我们正在编写的东西可能在我们项目的范围之外有用，所以我们将在一个可重用的包中消费 API，而不是直接在我们的程序代码中。在`$GOPATH/src`中的其他程序文件夹旁边创建一个名为`thesaurus`的新文件夹，并将以下代码插入到一个新的`bighugh.go`文件中：

```go
package thesaurus
import (
  "encoding/json"
  "errors"
  "net/http"
)
type BigHugh struct {
  APIKey string
}
type synonyms struct {
  Noun *words `json:"noun"`
  Verb *words `json:"verb"`
}
type words struct {
  Syn []string `json:"syn"`
}
func (b *BigHugh) Synonyms(term string) ([]string, error) {
  var syns []string
  response, err := http.Get("http://words.bighugelabs.com/api/2/" + b.APIKey + "/" + term + "/json")
  if err != nil {
    return syns, errors.New("bighugh: Failed when looking for synonyms for \"" + term + "\"" + err.Error())
  }
  var data synonyms
  defer response.Body.Close()
  if err := json.NewDecoder(response.Body).Decode(&data); err != nil {
    return syns, err
  }
  syns = append(syns, data.Noun.Syn...)
  syns = append(syns, data.Verb.Syn...)
  return syns, nil
}
```

在上述代码中，我们定义的`BigHugh`类型包含必要的 API 密钥，并提供了`Synonyms`方法，该方法将负责访问端点、解析响应并返回结果。这段代码最有趣的部分是`synonyms`和`words`结构。它们用 Go 术语描述了 JSON 响应格式，即包含名词和动词对象的对象，这些对象又包含一个名为`Syn`的字符串切片。标签（在每个字段定义后面的反引号中的字符串）告诉`encoding/json`包将哪些字段映射到哪些变量；这是必需的，因为我们给它们赋予了不同的名称。

### 提示

通常，JSON 键具有小写名称，但我们必须在我们的结构中使用大写名称，以便`encoding/json`包知道这些字段存在。如果我们不这样做，包将简单地忽略这些字段。但是，类型本身（`synonyms`和`words`）不需要被导出。

`Synonyms`方法接受一个`term`参数，并使用`http.Get`向 API 端点发出 web 请求，其中 URL 不仅包含 API 密钥值，还包含`term`值本身。如果由于某种原因 web 请求失败，我们将调用`log.Fatalln`，它会将错误写入标准错误流并以非零退出代码（实际上是`1`的退出代码）退出程序，表示发生了错误。

如果 web 请求成功，我们将响应主体（另一个`io.Reader`）传递给`json.NewDecoder`方法，并要求它将字节解码为我们的`synonyms`类型的`data`变量。我们推迟关闭响应主体，以便在使用 Go 的内置`append`函数将`noun`和`verb`的同义词连接到我们然后返回的`syns`切片之前保持内存清洁。

虽然我们已经实现了`BigHugh`词库，但这并不是唯一的选择，我们可以通过为我们的包添加`Thesaurus`接口来表达这一点。在`thesaurus`文件夹中，创建一个名为`thesaurus.go`的新文件，并将以下接口定义添加到文件中：

```go
package thesaurus
type Thesaurus interface {
  Synonyms(term string) ([]string, error)
}
```

这个简单的接口只是描述了一个接受`term`字符串并返回包含同义词的字符串切片或错误（如果出现问题）的方法。我们的`BigHugh`结构已经实现了这个接口，但现在其他用户可以为其他服务添加可互换的实现，比如[Dictionary.com](http://Dictionary.com)或 Merriam-Webster 在线服务。

接下来我们将在一个程序中使用这个新的包。通过在终端中返回到`$GOPATH/src`，创建一个名为`synonyms`的新文件夹，并将以下代码插入到一个新的`main.go`文件中，然后将该文件放入该文件夹中：

```go
func main() {
  apiKey := os.Getenv("BHT_APIKEY")
  thesaurus := &thesaurus.BigHugh{APIKey: apiKey}
  s := bufio.NewScanner(os.Stdin)
  for s.Scan() {
    word := s.Text()
    syns, err := thesaurus.Synonyms(word)
    if err != nil {
      log.Fatalln("Failed when looking for synonyms for \""+word+"\"", err)
    }
    if len(syns) == 0 {
      log.Fatalln("Couldn't find any synonyms for \"" + word + "\"")
    }
    for _, syn := range syns {
      fmt.Println(syn)
    }
  }
}
```

当你再次管理你的导入时，你将编写一个完整的程序，能够通过集成 Big Huge Thesaurus API 来查找单词的同义词。

在前面的代码中，我们的`main`函数首先要做的事情是通过`os.Getenv`调用获取`BHT_APIKEY`环境变量的值。为了使你的代码更加健壮，你可能需要再次检查以确保这个值被正确设置，并在没有设置时报告错误。现在，我们将假设一切都配置正确。

接下来，前面的代码开始看起来有点熟悉，因为它再次从`os.Stdin`扫描每一行输入，并调用`Synonyms`方法来获取替换词列表。

让我们构建一个程序，看看当我们输入单词`chat`时，API 返回了什么样的同义词：

```go

go build –o synonyms

./synonyms

chat

confab

confabulation

schmooze

New World chat

Old World chat

conversation

thrush

wood warbler

chew the fat

shoot the breeze

chitchat

chatter

```

你得到的结果很可能与我们在这里列出的结果不同，因为我们正在使用实时 API，但这里重要的一点是，当我们将一个词或术语作为程序的输入时，它会返回一个同义词列表作为输出，每行一个。

### 提示

尝试以不同的顺序将你的程序链接在一起，看看你得到什么结果。无论如何，我们将在本章后面一起做这件事。

### 获取域名建议

通过组合我们在本章中迄今为止构建的四个程序，我们已经有了一个有用的工具来建议域名。现在我们所要做的就是运行这些程序，同时以适当的方式将输出导入输入。在终端中，导航到父文件夹并运行以下单行命令：

```go

./synonyms/synonyms | ./sprinkle/sprinkle | ./coolify/coolify | ./domainify/domainify

```

因为`synonyms`程序在我们的列表中排在第一位，它将接收来自终端的输入（无论用户决定输入什么）。同样，因为`domainify`是链中的最后一个，它将把输出打印到终端供用户查看。在每一步，单词行将通过其他程序进行传输，使它们有机会发挥魔力。

输入一些单词来看一些域名建议，例如，如果你输入`chat`并回车，你可能会看到：

```go

getcnfab.com

confabulationtim.com

getschmoozee.net

schmosee.com

neew-world-chatsite.net

oold-world-chatsite.com

conversatin.net

new-world-warblersit.com

gothrush.net

lets-wood-wrbler.com

chw-the-fat.com

```

你得到的建议数量实际上取决于同义词的数量，因为它是唯一一个生成比我们给它的输出更多行的程序。

我们仍然没有解决我们最大的问题——我们不知道建议的域名是否真的可用，所以我们仍然需要坐下来，把它们每一个输入到一个网站中。在下一节中，我们将解决这个问题。

## 可用

我们的最终程序 Available 将连接到 WHOIS 服务器，询问传入的域名的详细信息——当然，如果没有返回任何详细信息，我们可以安全地假设该域名可以购买。不幸的是，WHOIS 规范（参见[`tools.ietf.org/html/rfc3912`](http://tools.ietf.org/html/rfc3912)）非常简单，没有提供关于当你询问域名的详细信息时，WHOIS 服务器应该如何回复的信息。这意味着以编程方式解析响应变得非常混乱。为了暂时解决这个问题，我们将只集成一个我们可以确定在响应中有“无匹配”（No match）的单个 WHOIS 服务器，当它没有该域名的记录时。

### 注意

一个更健壮的解决方案可能是使用具有明确定义结构的 WHOIS 接口来获取详细信息，也许在域名不存在的情况下提供错误消息，针对不同的 WHOIS 服务器有不同的实现。正如你所能想象的，这是一个相当大的项目；非常适合开源项目。

在`$GOPATH/src`目录旁边创建一个名为`available`的新文件夹，并在其中添加一个名为`main.go`的文件，其中包含以下函数代码：

```go
func exists(domain string) (bool, error) {
  const whoisServer string = "com.whois-servers.net"
  conn, err := net.Dial("tcp", whoisServer+":43")
  if err != nil {
    return false, err
  }
  defer conn.Close()
  conn.Write([]byte(domain + "\r\n"))
  scanner := bufio.NewScanner(conn)
  for scanner.Scan() {
    if strings.Contains(strings.ToLower(scanner.Text()), "no match") {
      return false, nil
    }
  }
  return true, nil
}
```

`exists`函数通过打开到指定`whoisServer`实例的端口`43`的连接来实现 WHOIS 规范中的一点内容，使用`net.Dial`进行调用。然后我们推迟关闭连接，这意味着无论函数如何退出（成功或出现错误，甚至是恐慌），都将在连接`conn`上调用`Close()`。连接打开后，我们只需写入域名，然后跟着`\r\n`（回车和换行字符）。这就是规范告诉我们的全部内容，所以从现在开始我们就要自己动手了。

基本上，我们正在寻找响应中是否提到了“无匹配”的内容，这就是我们决定域名是否存在的方式（在这种情况下，`exists`实际上只是询问 WHOIS 服务器是否有我们指定的域名的记录）。我们使用我们喜欢的`bufio.Scanner`方法来帮助我们迭代响应中的行。将连接传递给`NewScanner`是可行的，因为`net.Conn`实际上也是一个`io.Reader`。我们使用`strings.ToLower`，这样我们就不必担心大小写敏感性，使用`strings.Contains`来查看任何行是否包含“无匹配”文本。如果是，我们返回`false`（因为域名不存在），否则我们返回`true`。

`com.whois-servers.net` WHOIS 服务支持`.com`和`.net`的域名，这就是为什么 Domainify 程序只添加这些类型的域名。如果你使用的服务器对更广泛的域名提供了 WHOIS 信息，你可以添加对其他顶级域的支持。

让我们添加一个`main`函数，使用我们的`exists`函数来检查传入的域名是否可用。以下代码中的勾号和叉号符号是可选的——如果你的终端不支持它们，你可以自由地用简单的`Yes`和`No`字符串替换它们。

将以下代码添加到`main.go`中：

```go
var marks = map[bool]string{true: "✔", false: "×"}
func main() {
  s := bufio.NewScanner(os.Stdin)
  for s.Scan() {
    domain := s.Text()
    fmt.Print(domain, " ")
    exist, err := exists(domain)
    if err != nil {
      log.Fatalln(err)
    }
    fmt.Println(marks[!exist])
    time.Sleep(1 * time.Second)
  }
}
```

在`main`函数的前面代码中，我们只是迭代通过`os.Stdin`传入的每一行，用`fmt.Print`打印出域名（但不是`fmt.Println`，因为我们不想要换行），调用我们的`exists`函数来查看域名是否存在，然后用`fmt.Println`打印出结果（因为我们*确实*希望在最后有一个换行）。

最后，我们使用`time.Sleep`告诉进程在 1 秒内什么都不做，以确保我们对 WHOIS 服务器轻松一些。

### 提示

大多数 WHOIS 服务器都会以各种方式限制，以防止你占用过多资源。因此，减慢速度是确保我们不会惹恼远程服务器的明智方式。

考虑一下这对单元测试意味着什么。如果一个单元测试实际上是在向远程 WHOIS 服务器发出真实请求，每次测试运行时，您都会在您的 IP 地址上累积统计数据。一个更好的方法是对 WHOIS 服务器进行存根，以模拟真实的响应。

在前面代码的顶部的`marks`映射是将`exists`的布尔响应映射到人类可读的文本的一种好方法，这样我们只需使用`fmt.Println(marks[!exist])`在一行中打印响应。我们说不存在是因为我们的程序正在检查域名是否可用（逻辑上与是否存在于 WHOIS 服务器中相反）。

### 注意

我们可以在我们的代码中愉快地使用检查和叉字符，因为所有的 Go 代码文件都符合 UTF-8 标准——实际上获得这些字符的最好方法是在网上搜索它们，然后使用复制和粘贴将它们带入代码；否则，还有一些依赖于平台的方法来获得这样的特殊字符。

修复`main.go`文件的`import`语句后，我们可以尝试运行 Available，看看域名是否可用：

```go

go build –o available

./available

```

一旦 Available 正在运行，输入一些域名：

```go

packtpub.com

packtpub.com 

×

google.com

google.com 

×

madeupdomain1897238746234.net

madeupdomain1897238746234.net 

✔
```

正如你所看到的，对于显然不可用的域名，我们得到了一个小叉号，但是当我们使用随机数字编造一个域名时，我们发现它确实是可用的。

# 读累了记得休息一会哦~

**公众号：古德猫宁李**

+   电子书搜索下载

+   书单分享

+   书友学习交流

**网站：**[沉金书屋 https://www.chenjin5.com](https://www.chenjin5.com)

+   电子书搜索下载

+   电子书打包资源分享

+   学习资源分享

# 组合所有五个程序

现在我们已经完成了我们的所有五个程序，是时候把它们全部放在一起，这样我们就可以使用我们的工具为我们的聊天应用程序找到一个可用的域名。这样做的最简单方法是使用我们在本章中一直在使用的技术：在终端中使用管道连接输出和输入。

在终端中，导航到这五个程序的父文件夹，并运行以下单行代码：

```go

./synonyms/synonyms | ./sprinkle/sprinkle | ./coolify/coolify | ./domainify/domainify | ./available/available

```

程序运行后，输入一个起始词，看它如何生成建议，然后再检查它们的可用性。

例如，输入`chat`可能会导致程序执行以下操作：

1.  单词`chat`进入`synonyms`，然后出来一系列的同义词：

+   `confab`

+   `confabulation`

+   `schmooze`

1.  同义词流入`sprinkle`，在那里它们会被增加上网友好的前缀和后缀，比如：

+   `confabapp`

+   `goconfabulation`

+   `schmooze time`

1.  这些新词汇流入`coolify`，其中元音可能会被调整：

+   `confabaapp`

+   `goconfabulatioon`

+   `schmoooze time`

1.  修改后的词汇流入`domainify`，在那里它们被转换成有效的域名：

+   `confabaapp.com`

+   `goconfabulatioon.net`

+   `schmooze-time.com`

1.  最后，域名流入`available`，在那里它们被检查是否已经被某人注册了：

+   `confabaapp.com` ×

+   `goconfabulatioon.net` ✔

+   `schmooze-time.com` ✔

## 一款程序统治所有

通过将程序连接在一起来运行我们的解决方案是一种优雅的架构，但它并没有一个非常优雅的界面。具体来说，每当我们想要运行我们的解决方案时，我们都必须输入一个长长的混乱的行，其中每个程序都被列在一起，用管道字符分隔。在本节中，我们将编写一个 Go 程序，使用`os/exec`包来运行每个子程序，同时按照我们的设计将一个程序的输出传递到下一个程序的输入。

在其他五个程序旁边创建一个名为`domainfinder`的新文件夹，并在其中创建另一个名为`lib`的新文件夹。`lib`文件夹是我们将保存子程序构建的地方，但我们不想每次进行更改时都复制和粘贴它们。相反，我们将编写一个脚本，用于构建子程序并将二进制文件复制到`lib`文件夹中。

在 Unix 机器上创建一个名为`build.sh`的新文件，或者在 Windows 上创建一个名为`build.bat`的文件，并插入以下代码：

```go
#!/bin/bash
echo Building domainfinder...
go build -o domainfinder
echo Building synonyms...
cd ../synonyms
go build -o ../domainfinder/lib/synonyms
echo Building available...
cd ../available
go build -o ../domainfinder/lib/available
cd ../build
echo Building sprinkle...
cd ../sprinkle
go build -o ../domainfinder/lib/sprinkle
cd ../build
echo Building coolify...
cd ../coolify
go build -o ../domainfinder/lib/coolify
cd ../build
echo Building domainify...
cd ../domainify
go build -o ../domainfinder/lib/domainify
cd ../build
echo Done.
```

前面的脚本只是构建了我们所有的子程序（包括我们尚未编写的`domainfinder`），告诉`go build`将它们放在我们的`lib`文件夹中。确保通过执行`chmod +x build.sh`或类似的操作赋予新脚本执行权限。从终端运行此脚本，并查看`lib`文件夹，确保它确实将我们的子程序的二进制文件放在那里。

### 提示

现在不要担心`no buildable Go source files`错误，这只是 Go 告诉我们`domainfinder`程序没有任何`.go`文件可供构建。

在`domainfinder`内创建一个名为`main.go`的新文件，并在文件中插入以下代码：

```go
package main
var cmdChain = []*exec.Cmd{
  exec.Command("lib/synonyms"),
  exec.Command("lib/sprinkle"),
  exec.Command("lib/coolify"),
  exec.Command("lib/domainify"),
  exec.Command("lib/available"),
}
func main() {

  cmdChain[0].Stdin = os.Stdin
  cmdChain[len(cmdChain)-1].Stdout = os.Stdout

  for i := 0; i < len(cmdChain)-1; i++ {
    thisCmd := cmdChain[i]
    nextCmd := cmdChain[i+1]
    stdout, err := thisCmd.StdoutPipe()
    if err != nil {
      log.Fatalln(err)
    }
    nextCmd.Stdin = stdout
  }

  for _, cmd := range cmdChain {
    if err := cmd.Start(); err != nil {
      log.Fatalln(err)
    } else {
      defer cmd.Process.Kill()
    }
  }

  for _, cmd := range cmdChain {
    if err := cmd.Wait(); err != nil {
      log.Fatalln(err)
    }
  }

}
```

`os/exec`包为我们提供了一切我们需要从 Go 程序内部运行外部程序或命令的东西。首先，我们的`cmdChain`切片按照我们想要将它们连接在一起的顺序包含了`*exec.Cmd`命令。

在`main`函数的顶部，我们将第一个程序的`Stdin`（标准输入流）绑定到此程序的`os.Stdin`流，将最后一个程序的`Stdout`（标准输出流）绑定到此程序的`os.Stdout`流。这意味着，就像以前一样，我们将通过标准输入流接收输入，并将输出写入标准输出流。

我们的下一个代码块是通过迭代每个项目并将其`Stdin`设置为其前一个程序的`Stdout`来将子程序连接在一起的地方。

以下表格显示了每个程序，以及它从哪里获取输入，以及它的输出去哪里：

| 程序 | 输入（Stdin） | 输出（Stdout） |
| --- | --- | --- |
| `synonyms` | 与`domainfinder`相同的`Stdin` | `sprinkle` |
| `sprinkle` | `synonyms` | `coolify` |
| `coolify` | `sprinkle` | `domainify` |
| `domainify` | `coolify` | `available` |
| `available` | `domainify` | 与`domainfinder`相同的`Stdout` |

然后我们迭代每个命令调用`Start`方法，该方法在后台运行程序（与`Run`方法相反，后者将阻塞我们的代码，直到子程序退出——这当然是不好的，因为我们必须同时运行五个程序）。如果出现任何问题，我们将使用`log.Fatalln`退出，但如果程序成功启动，我们将推迟调用杀死进程。这有助于确保子程序在我们的`main`函数退出时退出，这将是`domainfinder`程序结束时。

一旦所有程序都在运行，我们就会再次迭代每个命令，并等待其完成。这是为了确保`domainfinder`不会提前退出并过早终止所有子程序。

再次运行`build.sh`或`build.bat`脚本，并注意`domainfinder`程序具有与我们之前看到的相同行为，但界面更加优雅。

# 总结

在这一章中，我们学习了五个小的命令行程序如何在组合在一起时产生强大的结果，同时保持模块化。我们避免了紧密耦合我们的程序，因此它们仍然可以单独使用。例如，我们可以使用我们的可用程序来检查手动输入的域名是否可用，或者我们可以将我们的`synonyms`程序仅用作命令行同义词词典。

我们学习了如何使用标准流来构建这些类型的程序的不同流，以及如何重定向标准输入和标准输出让我们非常容易地玩弄不同的流。

我们学习了在 Go 中消耗 JSON RESTful API web 服务是多么简单，当我们需要从 Big Hugh Thesaurus 获取同义词时。一开始我们保持简单，通过内联编码来编写代码，后来重构代码将`Thesaurus`类型抽象成自己的包，可以共享。当我们打开到 WHOIS 服务器的连接并通过原始 TCP 写入数据时，我们还使用了非 HTTP API。

我们看到了`math/rand`包如何通过允许我们在代码中使用伪随机数和决策，为我们带来了一些变化和不可预测性，这意味着每次运行程序时，我们都会得到不同的结果。

最后，我们构建了我们的`domainfinder`超级程序，将所有子程序组合在一起，为我们的解决方案提供了简单、干净和优雅的界面。
