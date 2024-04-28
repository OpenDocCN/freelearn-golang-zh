# 第八章。日志和测试

在上一章中，我们讨论了将应用程序责任委托给可通过 API 访问的网络服务和由消息队列处理的进程内通信。

这种方法模仿了将大型单片应用程序分解为较小块的新兴趋势；因此，允许开发人员利用不同的语言、框架和设计。

我们列举了这种方法的一些优点和缺点；大多数优点涉及保持开发的敏捷和精益，同时防止可能导致整个应用程序崩溃和级联错误的灾难性错误，一个很大的缺点是每个单独组件的脆弱性。例如，如果我们的电子邮件微服务在大型应用程序中有糟糕的代码，错误会很快显现出来，因为它几乎肯定会直接对另一个组件产生可检测的影响。但通过将进程隔离为微服务的一部分，我们也隔离了它们的状态和状态。

这就是本章内容发挥作用的地方——在 Go 应用程序中进行测试和记录的能力是该语言设计的优势。通过在我们的应用程序中利用这些功能，它可以扩展到包括更多的微服务；因此，我们可以更好地跟踪系统中任何问题的齿轮，而不会给整个应用程序增加太多额外的复杂性。

在本章中，我们将涵盖以下主题：

+   引入 Go 中的日志记录

+   IO 日志记录

+   格式化你的输出

+   使用 panic 和致命错误

+   引入 Go 中的测试

# 引入 Go 中的日志记录

Go 提供了无数种方法来将输出显示到`stdout`，最常见的是`fmt`包的`Print`和`Println`。事实上，你可以完全放弃`fmt`包，只使用`print()`或`println()`。

在成熟的应用程序中，你不太可能看到太多这样的情况，因为仅仅显示输出而没有能力将其存储在某个地方以进行调试或后续分析是罕见的，也缺乏实用性。即使你只是向用户输出一些反馈，通常也有意义这样做，并保留将其保存到文件或其他地方的能力，这就是`log`包发挥作用的地方。本书中的大多数示例出于这个原因使用了`log.Println`而不是`fmt.Println`。如果你在某个时候选择用其他（或附加）`io.Writer`替换`stdout`，这种更改是微不足道的。

# 记录到 IO

到目前为止，我们一直在将日志记录到`stdout`，但你可以利用任何`io.Writer`来接收日志数据。事实上，如果你希望输出路由到多个地方，你可以使用多个`io.Writer`。

## 多个记录器

大多数成熟的应用程序将写入多个日志文件，以区分需要保留的各种类型的消息。

这种最常见的用例在 Web 服务器中找到。它们通常保留一个`access.log`和一个`error.log`文件，以允许分析所有成功的请求；然而，它们还保留了不同类型消息的单独记录。

在下面的示例中，我们修改了我们的日志记录概念，包括错误和警告。

```go
package main

import (
  "log"
  "os"
)
var (
  Warn   *log.Logger
  Error  *log.Logger
  Notice *log.Logger
)
func main() {
  warnFile, err := os.OpenFile("warnings.log", os.O_RDWR|os.O_APPEND, 0660)
  defer warnFile.Close()
  if err != nil {
    log.Fatal(err)
  }
  errorFile, err := os.OpenFile("error.log", os.O_RDWR|os.O_APPEND, 0660)
  defer errorFile.Close()
  if err != nil {
    log.Fatal(err)
  }

  Warn = log.New(warnFile, "WARNING: ", Log.LstdFlags
)

  Warn.Println("Messages written to a file called 'warnings.log' are likely to be ignored :(")

  Error = log.New(errorFile, "ERROR: ", log.Ldate|log.Ltime)
  Error.SetOutput(errorFile)
  Error.Println("Error messages, on the other hand, tend to catch attention!")
}
```

我们可以采用这种方法来存储各种信息。例如，如果我们想要存储注册错误，我们可以创建一个特定的注册错误记录器，并在遇到该过程中的错误时允许类似的方法。

```go
  res, err := database.Exec("INSERT INTO users SET user_name=?, user_guid=?, user_email=?, user_password=?", name, guid, email, passwordEnc)

  if err != nil {
    fmt.Fprintln(w, err.Error)
    RegError.Println("Could not complete registration:", err.Error)
  } else {
    http.Redirect(w, r, "/page/"+pageGUID, 301)
  }
```

# 格式化你的输出

在实例化新的`Logger`时，你可以传递一些有用的参数和/或辅助字符串，以帮助定义和澄清输出。每个日志条目都可以以一个字符串开头，这在审查多种类型的日志条目时可能会有所帮助。你还可以定义你希望在每个条目上的日期和时间格式。

要创建自定义格式的日志，只需调用`New()`函数，并使用`io.Writer`，如下所示：

```go
package main

import (
  "log"
  "os"
)

var (
  Warn   *log.Logger
  Error  *log.Logger
  Notice *log.Logger
)

func main() {
  warnFile, err := os.OpenFile("warnings.log", os.O_RDWR|os.O_APPEND, 0660)
  defer warnFile.Close()
  if err != nil {
    log.Fatal(err)
  }
  Warn = log.New(warnFile, "WARNING: ", log.Ldate|log.Ltime)

  Warn.Println("Messages written to a file called 'warnings.log' are likely to be ignored :(")
  log.Println("Done!")
}
```

这不仅允许我们使用`log.Println`函数与我们的`stdout`，还允许我们在名为`warnings.log`的日志文件中存储更重要的消息。使用`os.O_RDWR|os.O_APPEND`常量允许我们写入文件并使用追加文件模式，这对于日志记录很有用。

# 使用 panic 和致命错误

除了简单地存储应用程序的消息之外，您还可以创建应用程序的 panic 和致命错误，这将阻止应用程序继续运行。这对于任何错误不会导致执行停止的用例至关重要，因为这可能会导致潜在的安全问题、数据丢失或任何其他意外后果。这些类型的机制通常被限制在最关键的错误上。

何时使用`panic()`方法并不总是清楚的，但在实践中，这应该被限制在不可恢复的错误上。不可恢复的错误通常意味着状态变得模糊或无法保证。

例如，对从数据库获取的记录进行操作，如果未能从数据库返回预期的结果，则可能被视为不可恢复的，因为未来的操作可能发生在过时或丢失的数据上。

在下面的例子中，我们可以实现一个 panic，我们无法创建一个新用户；这很重要，这样我们就不会尝试重定向或继续进行任何进一步的创建步骤：

```go
  if err != nil {
    fmt.Fprintln(w, err.Error)
    RegError.Println("Could not complete registration:", err.Error)
    panic("Error with registration,")
  } else {
    http.Redirect(w, r, "/page/"+pageGUID, 301)
  }
```

请注意，如果您想强制出现此错误，您可以在查询中故意制造一个 MySQL 错误：

```go
  res, err := database.Exec("INSERT INTENTIONAL_ERROR INTO users SET user_name=?, user_guid=?, user_email=?, user_password=?", name, guid, email, passwordEnc)
```

当触发此错误时，您将在相应的日志文件或`stdout`中找到它：

![使用 panic 和致命错误](img/B04294_08_01.jpg)

在上面的例子中，我们利用 panic 作为一个硬性停止，这将阻止进一步的执行，从而可能导致进一步的错误和/或数据不一致。如果不需要硬性停止，使用`recover()`函数允许您在问题得到解决或减轻后重新进入应用程序流程。

# 在 Go 中引入测试

Go 打包了大量出色的工具，用于确保您的代码干净、格式良好、没有竞争条件等。从`go vet`到`go fmt`，许多在其他语言中需要单独安装的辅助应用程序都作为 Go 的一部分打包了。

测试是软件开发的关键步骤。单元测试和测试驱动开发有助于发现对开发人员来说并不立即显而易见的错误。通常我们对应用程序太熟悉，以至于无法发现可能引发其他未发现的错误的可用性错误。

Go 的测试包允许对实际功能进行单元测试，同时确保所有依赖项（网络、文件系统位置）都可用；在不同的环境中进行测试可以让您在用户之前发现这些错误。

如果您已经在使用单元测试，Go 的实现将会非常熟悉和愉快：

```go
package example

func Square(x int) int {
  y := x * x
  return y
}
```

这保存为`example.go`。接下来，创建另一个 Go 文件，测试这个平方根功能，代码如下：

```go
package example

import (
  "testing"
)

func TestSquare(t *testing.T) {
  if v := Square(4); v != 16 {
    t.Error("expected", 16, "got", v)
  }
}
```

您可以通过进入目录并简单地输入`go test -v`来运行此测试。如预期的那样，给定我们的测试输入，这是通过的：

![在 Go 中引入测试](img/B04294_08_02.jpg)

这个例子显然是微不足道的，但为了演示如果您的测试失败会看到什么，让我们修改我们的`Square()`函数如下：

```go
func Square(x int) int {
  y := x
  return y
}
```

再次运行测试后，我们得到：

![在 Go 中引入测试](img/B04294_08_03.jpg)

对命令行应用程序进行命令行测试与与 Web 交互是不同的。我们的应用程序包括标准的 HTML 端点以及 API 端点；测试它需要比我们之前使用的方法更多的细微差别。

幸运的是，Go 还包括一个专门用于测试 HTTP 应用程序结果的包，`net/http/httptest`。

与前面的例子不同，`httptest`让我们评估从我们的各个函数返回的一些元数据，这些函数在 HTTP 版本的单元测试中充当处理程序。

那么，让我们来看一种简单的评估我们的 HTTP 服务器可能产生的内容的方法，通过生成一个快速端点，它简单地返回一年中的日期。

首先，我们将向我们的 API 添加另一个端点。让我们将这个处理程序示例分离成自己的应用程序，以便隔离其影响：

```go
package main

import (
  "fmt"
  "net/http"
  "time"
)

func testHandler(w http.ResponseWriter, r *http.Request) {
  t := time.Now()
  fmt.Fprintln(w, t.YearDay())
}

func main() {
  http.HandleFunc("/test", testHandler)
  http.ListenAndServe(":8080", nil)
}
```

这将简单地通过 HTTP 端点`/test`返回一年中的日期（1-366）。那么我们如何测试这个呢？

首先，我们需要一个专门用于测试的新文件。当涉及到需要达到多少测试覆盖率时，这通常对开发人员或组织很有帮助，理想情况下，我们希望覆盖每个端点和方法，以获得相当全面的覆盖。在这个例子中，我们将确保我们的一个 API 端点返回一个正确的状态码，以及一个`GET`请求返回我们在开发中期望看到的内容：

```go
package main

import (
  "io/ioutil"
  "net/http"
  "net/http/httptest"
  "testing"
)

func TestHandler(t *testing.T) {
  res := httptest.NewRecorder()
  path := "http://localhost:4000/test"
  o, err := http.NewRequest("GET", path, nil)
  http.DefaultServeMux.ServeHTTP(res, req)
  response, err := ioutil.ReadAll(res.Body)
  if string(response) != "115" || err != nil {
    t.Errorf("Expected [], got %s", string(response))
  }
}
```

现在，我们可以通过确保我们的端点通过（200）或失败（404）并返回我们期望的文本来在我们的实际应用程序中实现这一点。我们还可以自动添加新内容并对其进行验证，通过这些示例后，您应该有能力承担这一任务。

鉴于我们有一个 hello-world 端点，让我们编写一个快速测试，验证我们从端点得到的响应，并看看我们如何在`test.go`文件中获得一个正确的响应：

```go
package main

import (
  "net/http"
  "net/http/httptest"
  "testing"
)

func TestHelloWorld(t *testing.T) {

  req, err := http.NewRequest("GET", "/page/hello-world", nil)
  if err != nil {
    t.Fatal("Creating 'GET /page/hello-world' request failed!")
  }
  rec := httptest.NewRecorder()
  Router().ServeHTTP(rec, req)
}
```

在这里，我们可以测试我们是否得到了我们期望的状态码，尽管它很简单，但这并不一定是一个微不足道的测试。实际上，我们可能还会创建一个应该失败的测试，以及另一个检查我们是否得到了我们期望的 HTTP 响应的测试。但这为更复杂的测试套件，比如健全性测试或部署测试，奠定了基础。例如，我们可能会生成仅供开发使用的页面，从模板生成 HTML 内容，并检查输出以确保我们的页面访问和模板解析按照我们的期望工作。

### 注意

在[`golang.org/pkg/net/http/httptest/`](https://golang.org/pkg/net/http/httptest/)上阅读有关使用 http 和 httptest 包进行测试的更多信息

# 总结

简单地构建一个应用程序甚至不到一半的战斗，作为开发人员进行用户测试引入了测试策略中的巨大差距。测试覆盖率是一种关键武器，当我们发现错误之前，它可以帮助我们找到错误。

幸运的是，Go 提供了实现自动化单元测试所需的所有工具，以及支持它所需的日志记录架构。

在本章中，我们看了日志记录器和测试选项。通过为不同的消息生成多个记录器，我们能够将由内部应用程序故障引起的警告与错误分开。

然后，我们使用测试和`httptest`包来进行单元测试，自动检查我们的应用程序并通过测试潜在的破坏性更改来保持其当前状态。

在第九章*安全*中，我们将更彻底地研究实施安全性；从更好的 TLS/SSL 到防止注入和中间人和跨站点请求伪造攻击。
