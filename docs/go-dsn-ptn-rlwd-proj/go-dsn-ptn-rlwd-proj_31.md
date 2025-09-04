# 第六章。通过 RESTful 数据 Web 服务 API 公开数据和功能

在上一章中，我们构建了一个从 Twitter 读取推文、统计标签投票并将结果存储在 MongoDB 数据库中的服务。我们还使用了 MongoDB shell 来添加投票并查看投票结果。如果我们是唯一使用我们解决方案的人，这种方法是可以的，但如果我们发布了我们的项目并期望用户直接连接到我们的 MongoDB 实例以使用我们构建的服务，那就疯了。

因此，在本章中，我们将构建一个 RESTful 数据服务，通过该服务将公开数据和功能。我们还将构建一个简单的网站来消费新的 API。用户可以使用我们的网站创建和监控投票，或者在他们自己的应用上构建基于我们发布的 Web 服务。

### 小提示

本章中的代码依赖于第五章中的代码，*构建分布式系统和灵活数据处理*，因此建议您首先完成该章节，特别是因为它涵盖了本章代码运行的环境设置。

具体来说，您将学习：

+   如何通过包装`http.HandlerFunc`类型为我们提供 HTTP 请求的简单但强大的执行管道

+   如何使用`context`包安全地在 HTTP 处理器之间共享数据

+   负责公开数据的处理器的编写最佳实践

+   在现在允许我们编写最简单的实现的同时，留出空间在以后改进它们而不改变接口的小抽象

+   通过向我们的项目中添加简单的辅助函数和类型，将防止我们（或至少推迟）对外部包的依赖

# RESTful API 设计

要使 API 被认为是 RESTful 的，它必须遵守一些原则，这些原则与 Web 背后的原始概念保持一致，并且大多数开发者都已经知道。这种做法可以确保我们不会在我们的 API 中构建任何奇怪或不寻常的东西，同时也可以让我们的用户在消费它时有一个先发优势，因为他们已经熟悉其概念。

一些重要的 RESTful 设计概念包括：

+   HTTP 方法描述了要采取的操作类型；例如，`GET`方法将始终读取数据，而`POST`请求将创建某些内容

+   数据被表达为资源集合

+   动作被表达为数据的变化

+   URL 用于引用特定数据

+   HTTP 头用于描述进入和离开服务器的表示类型

下表显示了我们将支持的 HTTP 方法和 URL，包括简短描述和预期如何使用调用的示例用例。

| **请求** | **描述** | **用例** |
| --- | --- | --- |
| `GET /polls` | 读取所有投票 | 向用户展示投票列表 |
| `GET /polls/{id}` | 读取投票 | 显示特定投票的详细信息或结果 |
| `POST /polls` | 创建投票 | 创建一个新的投票 |
| `DELETE /polls/{id}` | 删除投票 | 删除特定的投票 |

`{id}` 占位符表示在路径中唯一 ID 的位置将放在哪里。

# 处理程序之间的数据共享

有时，我们需要在中间件和处理程序之间共享一个状态。Go 1.7 将 `context` 包引入了标准库，它为我们提供了共享基本请求范围数据的方式。

每个 `http.Request` 方法都附带一个可通过 `request.Context()` 方法访问的 `context.Context` 对象，我们可以从中创建新的上下文对象。然后我们可以调用 `request.WithContext()` 来获取一个（便宜）浅拷贝的 `http.Request` 方法，它使用我们的新 `Context` 对象。

要添加一个值，我们可以通过 `context.WithValue` 方法创建一个新的上下文（基于请求中现有的上下文）：

```go
ctx := context.WithValue(r.Context(), "key", "value") 

```

### 小贴士

虽然技术上可以使用这种方法存储任何类型的数据，但建议只存储简单的原始类型，如字符串和整数，并且不要用它来注入处理程序可能需要的依赖项或指向其他对象的指针。在本章的后面部分，我们将探讨访问依赖项的模式，例如数据库连接。

在中间件代码中，当我们传递执行到包装处理程序时，我们可以使用我们的新 `ctx` 对象：

```go
Handler.ServeHTTP(w, r.WithContext(ctx)) 

```

值得探索上下文包的文档[`golang.org/pkg/context/`](https://golang.org/pkg/context/)，以了解它提供了哪些其他功能。

我们将使用这项技术来允许我们的处理程序访问一个在其他地方提取和验证的 API 密钥。

## 上下文键

在上下文对象中设置值需要我们使用一个键，虽然值参数是 `interface{}` 类型可能看起来很明显，这意味着我们可以（但不一定应该）存储我们喜欢的任何东西，但了解键的类型可能会让你感到惊讶：

```go
func WithValue(parent Context, key, val interface{}) Context 

```

键也是一个 `interface{}`。这意味着我们不仅限于使用字符串作为键，这对于考虑不同代码可能试图在相同上下文中使用相同名称设置值是有好处的，这可能会引起问题。

相反，一个更稳定的键值模式正在 Go 社区中浮现（并且已经在标准库的一些地方使用）。我们将创建一个简单的（私有）`struct`来存储我们的键，并添加一个辅助方法来从上下文中获取值。

在新的 `api` 文件夹内添加必要的最小 `main.go` 文件：

```go
package main 
func main(){} 

```

添加一个名为 `contextKey` 的新类型：

```go
type contextKey struct { 
  name string 
} 

```

这个结构只包含键的名称，但即使两个键的 `name` 字段相同，指向它的指针也将保持唯一。接下来，我们将添加一个键来存储我们的 API 密钥值：

```go
var contextKeyAPIKey = &contextKey{"api-key"} 

```

将相关变量组合在一起并使用共同的前缀是一种良好的实践；在我们的情况下，我们可以从`contextKey`前缀开始命名所有上下文键类型。在这里，我们创建了一个名为`contextKeyAPIKey`的键，它是一个指向`contextKey`类型的指针，名称设置为`api-key`。

接下来，我们将编写一个辅助函数，它将根据上下文提取密钥：

```go
func APIKey(ctx context.Context) (string, bool) {
 key, ok := ctx.Value(contextKeyAPIKey).(string)
 return key, ok
}

```

函数接收`context.Context`并返回 API 密钥字符串以及一个`ok`布尔值，表示密钥是否成功获取并转换为字符串。如果密钥缺失或类型错误，第二个返回参数将为 false，但我们的代码不会崩溃。

注意，`contextKey`和`contextKeyAPIKey`是内部的（它们以小写字母开头），但`APIKey`将被导出。在`main`包中，这并不重要，但如果你正在编写一个包，了解你存储和从上下文中提取数据的方式的复杂性对用户来说是件好事。

# 包装处理函数

我们将利用在 Go 中构建服务和网站时学习到的最有价值的模式之一，我们已经在第二章中稍微探索过，即*添加用户账户*：包装处理函数。我们已经看到如何包装`http.Handler`类型，在主处理函数执行前后运行代码，现在我们将应用相同的技巧到`http.HandlerFunc`函数替代品上。

## API 密钥

大多数 Web API 要求客户端为其应用程序注册一个 API 密钥，并要求他们在每次请求时发送该密钥。这些密钥有许多用途，从简单地识别请求来自哪个应用程序到解决某些应用程序只能根据用户允许的内容执行有限操作的情况下的授权问题。虽然我们实际上不需要为我们的应用程序实现 API 密钥，但我们要求客户端提供一个，这将允许我们稍后添加实现，同时保持接口不变。

我们将在`main.go`的底部添加第一个`HandlerFunc`包装函数，命名为`withAPIKey`：

```go
func withAPIKey(fn http.HandlerFunc) http.HandlerFunc { 
  return func(w http.ResponseWriter, r *http.Request) { 
    key := r.URL.Query().Get("key") 
    if !isValidAPIKey(key) { 
      respondErr(w, r, http.StatusUnauthorized, "invalid
       API key") 
      return 
    } 
    ctx := context.WithValue(r.Context(),
     contextKeyAPIKey, key) 
    fn(w, r.WithContext(ctx)) 
  } 
} 

```

如您所见，我们的`withAPIKey`函数既接受一个`http.HandlerFunc`类型作为参数，也返回一个；这就是我们在这个上下文中所说的包装。`withAPIKey`函数依赖于我们尚未编写的许多其他函数，但您可以清楚地看到正在发生的事情。我们的函数立即返回一个新的`http.HandlerFunc`类型，通过调用`isValidAPIKey`来检查`key`查询参数。如果密钥被认为无效（通过返回`false`），我们则响应一个`invalid API key`错误；否则，我们将密钥放入上下文并调用下一个处理器。要使用此包装器，我们只需将一个`http.HandlerFunc`类型传递给此函数，以便启用`key`参数检查。由于它也返回一个`http.HandlerFunc`类型，因此结果可以传递给其他包装器或直接传递给`http.HandleFunc`函数，以便将其注册为特定路径模式的处理器。

让我们接下来添加我们的`isValidAPIKey`函数：

```go
func isValidAPIKey(key string) bool { 
  return key == "abc123" 
} 

```

目前，我们只是将 API 密钥硬编码为`abc123`；任何其他内容都将返回`false`，因此被视为无效。稍后，我们可以修改这个函数，以便通过咨询配置文件或数据库来检查密钥的真实性，而不会影响我们使用`isValidAPIKey`方法或`withAPIKey`包装器的方式。

## 跨源资源共享

同源安全策略规定，在 Web 浏览器中，AJAX 请求仅允许来自同一域的服务，这将使我们的 API 相当有限，因为我们不一定会在所有使用我们的 Web 服务的网站上托管。**CORS**（**跨源资源共享**）技术绕过了同源策略，使我们能够构建一个能够为托管在其他域上的网站提供服务的服务。为此，我们只需在响应中设置`Access-Control-Allow-Origin`头为`*`。既然我们将在创建投票调用中使用`Location`头，我们也将允许客户端访问此头，这可以通过在`Access-Control-Expose-Headers`头中列出它来实现。将以下代码添加到`main.go`：

```go
func withCORS(fn http.HandlerFunc) http.HandlerFunc { 
  return func(w http.ResponseWriter, r *http.Request) { 
    w.Header().Set("Access-Control-Allow-Origin", "*") 
    w.Header().Set("Access-Control-Expose-Headers",
     "Location") 
    fn(w, r) 
  } 
} 

```

这是迄今为止最简单的包装函数；它只是在`ResponseWriter`类型上设置适当的头，并调用指定的`http.HandlerFunc`类型。

### 小贴士

在本章中，我们明确处理 CORS（跨源资源共享），以便我们能够确切地了解正在发生的情况；对于真正的生产代码，你应该考虑使用开源解决方案，例如[`github.com/fasterness/cors`](https://github.com/fasterness/cors)。

# 依赖注入

现在我们可以确信请求有一个有效的 API 密钥并且是 CORS 兼容的，我们必须考虑处理器将如何连接到数据库。一个选项是让每个处理器自己拨号连接，但这不是非常**DRY**（不要重复自己），并且留下了可能产生错误代码的空间，例如忘记在完成数据库会话后关闭数据库会话的代码。这也意味着，如果我们想改变我们连接数据库的方式（也许我们想使用域名而不是硬编码的 IP 地址），我们可能需要在许多地方修改我们的代码，而不是一个地方。

相反，我们将创建一个新类型，它封装了处理器的所有依赖，并在 `main.go` 中使用数据库连接来构建它。

创建一个名为 `Server` 的新类型：

```go
// Server is the API server. 
type Server struct { 
  db *mgo.Session 
} 

```

我们的处理函数将是这个服务的方法，这样它们就能访问数据库会话。

# 响应

任何 API 的一个重要部分是使用状态码、数据、错误以及有时是头部来响应请求——`net/http` 包使得所有这些都非常容易实现。我们有一个选项，对于小型项目或者大型项目的早期阶段来说，仍然是最佳选项，那就是直接在处理器内部构建响应代码。

然而，随着处理器的数量增加，我们最终会重复大量的代码，并在我们的项目中到处散布表示决策。一个更可扩展的方法是将响应代码抽象成辅助函数。

对于我们 API 的第一个版本，我们将只使用 JSON，但我们希望有灵活性，以便在需要时添加其他表示。

创建一个名为 `respond.go` 的新文件，并添加以下代码：

```go
func decodeBody(r *http.Request, v interface{}) error { 
  defer r.Body.Close() 
  return json.NewDecoder(r.Body).Decode(v) 
} 
func encodeBody(w http.ResponseWriter, r *http.Request, v  interface{}) error { 
  return json.NewEncoder(w).Encode(v) 
} 

```

这两个函数分别抽象了从 `Request` 和 `ResponseWriter` 对象中解码和编码数据。解码器还会关闭请求体，这是推荐的。尽管我们在这里没有添加很多功能，但这意味着我们不需要在我们的代码的其他地方提及 JSON，如果我们决定添加对其他表示的支持或切换到二进制协议，我们只需要修改这两个函数。

接下来，我们将添加一些额外的辅助函数，这将使响应变得更加容易。在 `respond.go` 中添加以下代码：

```go
func respond(w http.ResponseWriter, r *http.Request, 
 status int, data interface{}) { 
  w.WriteHeader(status) 
  if data != nil { 
    encodeBody(w, r, data) 
  } 
} 

```

这个函数使得使用我们的 `encodeBody` 辅助函数将状态码和一些数据写入 `ResponseWriter` 对象变得非常容易。

处理错误是另一个值得抽象的重要方面。添加以下 `respondErr` 辅助函数：

```go
func respondErr(w http.ResponseWriter, r *http.Request, 
 status int, args ...interface{}) { 
  respond(w, r, status, map[string]interface{}{ 
    "error": map[string]interface{}{ 
      "message": fmt.Sprint(args...), 
    }, 
  }) 
} 

```

这个方法提供了一个类似于 `respond` 函数的接口，但写入的数据将被包裹在一个 `error` 对象中，以便清楚地表明出了问题。最后，我们可以添加一个专门用于生成正确消息的 HTTP 错误特定辅助函数，该函数使用 Go 标准库中的 `http.StatusText` 函数：

```go
func respondHTTPErr(w http.ResponseWriter, r *http.Request, status int) { 
  respondErr(w, r, status, http.StatusText(status)) 
} 

```

注意，这些函数都是“狗粮”，这意味着它们相互使用（就像吃自己的狗粮一样），这对于我们希望在只有一个地方进行实际响应（如果或更可能地，当我们需要做出更改时）来说很重要。

# 理解请求

`http.Request` 对象为我们提供了访问底层 HTTP 请求可能需要的每一块信息的途径；因此，浏览一下 `net/http` 文档以真正了解其强大功能是值得的。以下是一些示例，但不仅限于以下内容：

+   URL、路径和查询字符串

+   HTTP 方法

+   Cookies

+   文件

+   表单值

+   请求者的引用和用户代理

+   基本认证详情

+   请求体

+   标头信息

它没有解决一些问题，我们需要自己解决或寻找外部包来帮助我们。URL 路径解析就是这样一个问题——虽然我们可以通过 `http.Request` 类型中的 `URL.Path` 字段以字符串的形式访问路径（如 `/people/1/books/2`），但没有简单的方法可以提取路径中编码的数据，如 `1` 的人 ID 或 `2` 的书 ID。

### 备注

一些项目很好地解决了这个问题，例如 Goweb 或 Gorillz 的 `mux` 包。它们允许你映射包含占位符的路径模式，然后从原始字符串中提取这些值并将其提供给你的代码。例如，你可以映射一个模式 `/users/{userID}/comments/{commentID}`，这将映射路径，如 `/users/1/comments/2`。在你的处理程序代码中，你可以通过大括号内的名称来获取值，而不是自己解析路径。

由于我们的需求很简单，我们将构建一个简单的路径解析实用工具；如果需要，我们总是可以使用不同的包，但这意味着要向我们的项目中添加一个依赖。

创建一个名为 `path.go` 的新文件，并插入以下代码：

```go
package main 
import ( 
  "strings" 
) 
const PathSeparator = "/" 
type Path struct { 
  Path string 
  ID   string 
} 
func NewPath(p string) *Path { 
  var id string 
  p = strings.Trim(p, PathSeparator) 
  s := strings.Split(p, PathSeparator) 
  if len(s) > 1 { 
    id = s[len(s)-1] 
    p = strings.Join(s[:len(s)-1], PathSeparator) 
  } 
  return &Path{Path: p, ID: id} 
} 
func (p *Path) HasID() bool { 
  return len(p.ID) > 0 
} 

```

这个简单的解析器提供了一个 `NewPath` 函数，它解析指定的路径字符串，并返回一个 `Path` 类型的新的实例。使用 `strings.Trim` 去除前导和尾随斜杠，并使用 `PathSeparator` 常量（即一个正斜杠）将剩余的路径分割（使用 `strings.Split`）。如果有多个段（`len(s) > 1`），最后一个被认为是 ID。我们使用 `s[len(s)-1]` 重新切片字符串以选择 ID，并使用 `s[:len(s)-1]` 选择路径的其余部分。同样，我们使用 `PathSeparator` 常量重新连接路径段，以形成一个不包含 ID 的单个字符串。

这支持任何 `collection/id` 对，这正是我们 API 所需要的。以下表格显示了给定原始路径字符串的 `Path` 类型的状态：

| **原始路径字符串** | **路径** | **ID** | **HasID** |
| --- | --- | --- | --- |
| `/` | `/` | `nil` | `false` |
| `/people/` | `people` | `nil` | `false` |
| `/people/1/` | `people` | `1` | `true` |

# 使用一个函数提供我们的 API 服务

一个网络服务不过是一个简单的 Go 程序，它绑定到特定的 HTTP 地址和端口，并处理请求，因此我们可以使用我们所有的命令行工具编写知识和技巧。

### 小贴士

我们还希望确保我们的`main`函数尽可能简单和谦逊，这始终是编码的目标，尤其是在 Go 中。

在编写我们的`main`函数之前，让我们看看我们的 API 程序的一些设计目标：

+   我们应该能够指定 API 监听的 HTTP 地址和端口以及 MongoDB 实例的地址，而无需重新编译程序（通过命令行标志）

    我们希望程序在终止时能够优雅地关闭，允许正在处理的请求（在发送终止信号给我们的程序时仍在处理的请求）完成

+   我们希望程序能够正确地记录状态更新并报告错误

在`main.go`文件顶部，将`main`函数占位符替换为以下代码：

```go
func main() { 
  var ( 
    addr  = flag.String("addr", ":8080", "endpoint
     address") 
    mongo = flag.String("mongo", "localhost", "mongodb
     address") 
  ) 
  log.Println("Dialing mongo", *mongo) 
  db, err := mgo.Dial(*mongo) 
  if err != nil { 
    log.Fatalln("failed to connect to mongo:", err) 
  } 
  defer db.Close() 
  s := &Server{ 
    db: db, 
  } 
  mux := http.NewServeMux() 
  mux.HandleFunc("/polls/",
   withCORS(withAPIKey(s.handlePolls))) 
  log.Println("Starting web server on", *addr) 
  http.ListenAndServe(":8080", mux) 
  log.Println("Stopping...") 
} 

```

这个函数就是我们的 API`main`函数的全部内容。我们首先指定两个命令行标志，`addr`和`mongo`，并设置一些合理的默认值，然后要求`flag`包解析它们。然后我们尝试在指定的地址拨号到 MongoDB 数据库。如果我们不成功，我们通过调用`log.Fatalln`来中止。假设数据库正在运行并且我们能够连接，我们在延迟关闭连接之前将引用存储在`db`变量中。这确保了我们的程序在结束时能够正确地断开连接并清理。

### 小贴士

我们创建我们的服务器并指定数据库依赖项。我们调用我们的服务器`s`，有些人认为这是一种不好的做法，因为它很难阅读只引用单个字母变量的代码并知道它是什么。然而，由于这个变量的作用域很小，我们可以确信它的使用将非常接近其定义，从而消除了混淆的可能性。

然后，我们创建一个新的`http.ServeMux`对象，这是 Go 标准库提供的请求多路复用器，并为以`/polls/`路径开始的请求注册一个单一的处理程序。请注意，`handlePolls`处理程序是我们服务器上的一个方法，这就是它将如何访问数据库。

## 使用处理程序函数包装器

当我们在`ServeMux`处理程序上调用`HandleFunc`时，我们就在使用处理程序函数包装器，如下所示：

```go
withCORS(withAPIKey(handlePolls)) 

```

由于每个函数都接受一个`http.HandlerFunc`类型作为参数，并返回一个，因此我们能够通过嵌套函数调用链式执行，就像我们之前所做的那样。因此，当接收到路径前缀为`/polls/`的请求时，程序将采取以下执行路径：

1.  调用了`withCORS`函数，该函数设置了适当的头信息。

1.  接下来调用 `withAPIKey` 函数，该函数检查请求中的 API 密钥，如果无效则终止，否则调用下一个处理器函数。

1.  然后调用 `handlePolls` 函数，该函数可能使用 `respond.go` 中的辅助函数向客户端写入响应。

1.  执行回到 `withAPIKey`，然后退出。

1.  执行最终回到 `withCORS`，然后退出。

# 处理端点

最后一个拼图是 `handlePolls` 函数，它将使用辅助函数来理解传入的请求、访问数据库并生成一个有意义的响应，该响应将被发送回客户端。我们还需要对上一章中我们正在处理的投票数据进行建模。

创建一个名为 `polls.go` 的新文件并添加以下代码：

```go
package main 
import "gopkg.in/mgo.v2/bson" 
type poll struct { 
  ID      bson.ObjectId  `bson:"_id" json:"id"` 
  Title   string         `json:"title"` 
  Options []string       `json:"options"` 
  Results map[string]int `json:"results,omitempty"` 
  APIKey  string         `json:"apikey"` 
} 

```

在这里，我们定义了一个名为 `poll` 的结构体，它有五个字段，这些字段反过来描述了我们在上一章中编写的代码创建和维护的投票。我们还添加了 `APIKey` 字段，您可能不会在现实世界中这样做，但它将允许我们演示如何从上下文中提取 API 密钥。每个字段都有一个标签（`ID` 的情况有两个），这允许我们提供一些额外的元数据。

## 使用标签向结构体添加元数据

标签只是跟在结构体类型字段定义同一行的字符串。我们使用黑色引号字符来表示字面字符串，这意味着我们可以在标签字符串内部自由使用双引号。`reflect` 包允许我们提取与任何键关联的值；在我们的例子中，`bson` 和 `json` 都是键的例子，它们是每个由空格字符分隔的键/值对。`encoding/json` 和 `gopkg.in/mgo.v2/bson` 包允许您使用标签来指定用于编码和解码的字段名称（以及一些其他属性），而不是从字段名称本身推断值。我们使用 BSON 与 MongoDB 数据库通信，使用 JSON 与客户端通信，因此我们可以实际指定同一 `struct` 类型的不同视图。例如，考虑 ID 字段：

```go
ID bson.ObjectId `bson:"_id" json:"id"` 

```

Go 中字段的名称是 `ID`，JSON 字段是 `id`，BSON 字段是 `_id`，这是 MongoDB 中使用的特殊标识符字段。

## 使用单个处理器执行许多操作

由于我们的简单路径解析解决方案只关心路径，因此当查看客户端正在进行的 RESTful 操作类型时，我们必须做一些额外的工作。具体来说，我们需要考虑 HTTP 方法，以便我们知道如何处理请求。例如，对 `/polls/` 路径的 `GET` 调用应读取投票，而 `POST` 调用将创建一个新的投票。一些框架通过允许您根据路径以外的更多内容映射处理器来解决此问题，例如 HTTP 方法或请求中存在特定头。由于我们的案例非常简单，我们将使用简单的 `switch` 语句。在 `polls.go` 中添加 `handlePolls` 函数：

```go
func (s *Server) handlePolls(w http.ResponseWriter,
 r *http.Request) { 
  switch r.Method { 
    case "GET": 
    s.handlePollsGet(w, r) 
    return 
    case "POST": 
    s.handlePollsPost(w, r) 
    return 
    case "DELETE": 
    s.handlePollsDelete(w, r) 
    return 
  } 
  // not found 
  respondHTTPErr(w, r, http.StatusNotFound) 
} 

```

我们根据 HTTP 方法进行分支，并根据它是`GET`、`POST`还是`DELETE`来分支我们的代码。如果 HTTP 方法不是这些，我们只响应一个`404 http.StatusNotFound`错误。为了使此代码编译，你可以在`handlePolls`处理程序下方添加以下函数存根：

```go
func (s *Server) handlePollsGet(w http.ResponseWriter,
 r *http.Request) { 
  respondErr(w, r, http.StatusInternalServerError,
   errors.New("not    
  implemented")) 
} 
func (s *Server) handlePollsPost(w http.ResponseWriter,
 r *http.Request) { 
  respondErr(w, r, http.StatusInternalServerError,
   errors.New("not   
   implemented")) 
} 
func (s *Server) handlePollsDelete(w http.ResponseWriter,
  r *http.Request) { 
  respondErr(w, r, http.StatusInternalServerError,
   errors.New("not  
   implemented")) 
} 

```

### 小贴士

在本节中，我们学习了如何手动解析请求的元素（HTTP 方法）并在代码中做出决策。这对于简单情况来说很好，但值得看看像 Gorilla 的`mux`包这样的包，它们提供了一些更强大的解决这些问题的方法。然而，将外部依赖降到最低是编写良好且封装良好的 Go 代码的核心原则。

### 读取投票

现在是时候实现我们的网络服务功能了。添加以下代码：

```go
func (s *Server) handlePollsGet(w http.ResponseWriter,
 r *http.Request) { 
  session := s.db.Copy() 
  defer session.Close() 
  c := session.DB("ballots").C("polls") 
  var q *mgo.Query 
  p := NewPath(r.URL.Path) 
  if p.HasID() { 
    // get specific poll 
    q = c.FindId(bson.ObjectIdHex(p.ID)) 
  } else { 
    // get all polls 
    q = c.Find(nil) 
  } 
  var result []*poll 
  if err := q.All(&result); err != nil { 
    respondErr(w, r, http.StatusInternalServerError, err) 
    return 
  } 
  respond(w, r, http.StatusOK, &result) 
} 

```

在我们的每个子处理函数中，我们首先创建数据库会话的副本，这将允许我们与 MongoDB 交互。然后我们使用`mgo`创建一个指向数据库中`polls`集合的对象——如果你还记得，这就是我们的投票所在。

我们通过解析路径构建一个`mgo.Query`对象。如果存在 ID，我们在`polls`集合上使用`FindId`方法；否则，我们将`nil`传递给`Find`方法，这表示我们想要选择所有投票。我们使用`ObjectIdHex`方法将 ID 从字符串转换为`bson.ObjectId`类型，这样我们就可以用它们的数值（十六进制）标识符来引用投票。

由于`All`方法期望生成一个`poll`对象的集合，我们定义结果为`[]*poll`或指向 poll 类型的指针切片。在查询上调用`All`方法将导致`mgo`使用其与 MongoDB 的连接来读取所有投票并填充`result`对象。

### 注意

对于小规模，例如少量投票，这种方法是可行的，但随着投票的增加，我们需要考虑一种更复杂的方法。我们可以通过在查询上使用`Iter`方法迭代它们，并使用`Limit`和`Skip`方法来分页结果，这样我们就不试图将太多数据加载到内存中，或者一次向用户提供太多信息。

现在我们已经添加了一些功能，让我们第一次尝试我们的 API。如果你正在使用我们在上一章中设置的相同 MongoDB 实例，你应该已经在`polls`集合中有些数据了；为了确保我们的 API 能够正常工作，你应该确保数据库中至少有两个投票。

如果你需要向数据库添加其他投票，在终端中运行`mongo`命令以打开一个数据库外壳，这将允许你与 MongoDB 交互。然后，输入以下命令以添加一些测试投票：

```go
> use ballots
switched to db ballots
> db.polls.insert({"title":"Test  poll","options":
     ["one","two","three"]})
> db.polls.insert({"title":"Test poll  two","options":
     ["four","five","six"]})

```

在终端中，导航到你的`api`文件夹，构建并运行项目：

```go
go build -o api
./api

```

现在，通过在浏览器中导航到`http://localhost:8080/polls/?key=abc123`来对该`/polls/`端点发起一个`GET`请求；请记住包括末尾的反斜杠。结果将是一个包含 JSON 格式的投票数组的数组。

从投票列表中复制一个 ID，并将其插入到浏览器中的`?`字符之前，以访问特定投票的数据，例如，`http://localhost:8080/polls/5415b060a02cd4adb487c3ae?key=abc123`。请注意，它不会返回所有投票，而只返回一个。

### 小贴士

通过移除或更改密钥参数来测试 API 密钥功能，以查看错误看起来像什么。

你可能也注意到，尽管我们只返回一个投票，但这个投票值仍然嵌套在一个数组中。这是出于两个原因的故意设计决策：第一个也是最重要的原因是嵌套使得 API 用户编写代码来消费数据变得更加容易。如果用户总是期望一个 JSON 数组，他们可以编写描述这种期望的强类型，而不是为单个投票和投票集合使用不同的类型。作为 API 设计者，这是你的决定。第二个原因是，我们让对象嵌套在数组中使得 API 代码更加简单，这允许我们只更改`mgo.Query`对象，而让其他代码保持不变。

### 创建投票

客户应该能够向`/polls/`发送一个`POST`请求以创建投票。让我们在`POST`情况下添加以下代码：

```go
func (s *Server) handlePollsPost(w http.ResponseWriter, 
 r *http.Request) { 
  session := s.db.Copy() 
  defer session.Close() 
  c := session.DB("ballots").C("polls") 
  var p poll 
  if err := decodeBody(r, &p); err != nil { 
    respondErr(w, r, http.StatusBadRequest, "failed to
     read poll from request", err) 
    return 
  } 
  apikey, ok := APIKey(r.Context()) 
  if ok { 
    p.APIKey = apikey 
  } 
  p.ID = bson.NewObjectId() 
  if err := c.Insert(p); err != nil { 
    respondErr(w, r, http.StatusInternalServerError,
     "failed to insert 
    poll", err) 
    return 
  } 
  w.Header().Set("Location", "polls/"+p.ID.Hex()) 
  respond(w, r, http.StatusCreated, nil) 
} 

```

在我们获取到数据库会话的副本之后，我们尝试解码请求的正文，根据 RESTful 原则，该正文应该包含客户端想要创建的投票对象的表示。如果发生错误，我们使用`respondErr`辅助函数将错误写入用户并立即退出函数。然后我们为投票生成一个新的唯一 ID，并使用`mgo`包的`Insert`方法将其发送到数据库。然后我们设置响应的`Location`头并使用`201 http.StatusCreated`消息响应，指向可以访问新创建的投票的 URL。一些 API 返回对象而不是提供链接；没有具体的标准，所以这取决于你作为设计者的决定。

### 删除投票

我们将要包含在我们 API 中的最后一块功能是能够删除投票。通过向投票的 URL（例如`/polls/5415b060a02cd4adb487c3ae`）发送一个使用`DELETE` HTTP 方法的请求，我们希望能够从数据库中删除该投票并返回一个`200 成功`响应：

```go
func (s *Server) handlePollsDelete(w http.ResponseWriter, 
  r *http.Request) { 
  session := s.db.Copy() 
  defer session.Close() 
  c := session.DB("ballots").C("polls") 
  p := NewPath(r.URL.Path) 
  if !p.HasID() { 
    respondErr(w, r, http.StatusMethodNotAllowed,
      "Cannot delete all polls.") 
    return 
  } 
  if err := c.RemoveId(bson.ObjectIdHex(p.ID)); err != nil { 
    respondErr(w, r, http.StatusInternalServerError,
     "failed to delete poll", err) 
    return 
  } 
  respond(w, r, http.StatusOK, nil) // ok 
} 

```

与`GET`情况类似，我们解析路径，但这次如果路径不包含 ID，我们将返回一个错误。目前我们不希望人们能够通过一个请求删除所有投票，因此我们使用合适的`StatusMethodNotAllowed`代码。然后，使用我们在前一个案例中使用的相同集合，我们调用`RemoveId`，将路径中的 ID 转换为`bson.ObjectId`类型后传递。假设一切顺利，我们将以没有主体的`http.StatusOK`消息响应。

### CORS 支持

为了让我们的`DELETE`功能能够在 CORS 下工作，我们必须做一些额外的工作来支持 CORS 浏览器处理某些 HTTP 方法（如`DELETE`）的方式。实际上，CORS 浏览器会发送一个预检请求（带有`OPTIONS` HTTP 方法），请求允许发送`DELETE`请求（列在`Access-Control-Request-Method`请求头中），并且 API 必须适当地响应，以便请求能够工作。在`switch`语句中添加另一个针对`OPTIONS`的情况：

```go
case "OPTIONS": 
  w.Header().Add("Access-Control-Allow-Methods", "DELETE") 
  respond(w, r, http.StatusOK, nil) 
  return 

```

如果浏览器请求发送`DELETE`请求的权限，API 将通过设置`Access-Control-Allow-Methods`头为`DELETE`来响应，从而覆盖我们在`withCORS`包装处理程序中设置的默认`*`值。在现实世界中，`Access-Control-Allow-Methods`头的值将根据请求做出相应改变，但由于我们只支持`DELETE`这一种情况，我们可以现在将其硬编码。

### 注意

CORS 的详细信息超出了本书的范围，但如果您打算构建真正可访问的 Web 服务和 API，建议您在网上研究具体细节。前往[`enable-cors.org/`](http://enable-cors.org/)开始。

## 使用 curl 测试我们的 API

Curl 是一个命令行工具，它允许我们向我们的服务发送 HTTP 请求，这样我们就可以像真正的应用程序或客户端一样访问它，消费该服务。

### 注意

Windows 用户默认没有 curl 的访问权限，需要寻找替代方案。查看[`curl.haxx.se/dlwiz/?type=bin`](http://curl.haxx.se/dlwiz/?type=bin)或在网上搜索 Windows curl 的替代方案。

在终端中，让我们通过我们的 API 读取数据库中的所有投票。导航到您的`api`文件夹，构建并运行项目，并确保 MongoDB 正在运行：

```go
go build -o api
./api

```

我们接下来执行以下步骤：

1.  输入以下使用`-X`标志的`curl`命令，表示我们想要向指定的 URL 发送一个`GET`请求：

    ```go
    curl -X GET http://localhost:8080/polls/?
             key=abc123

    ```

1.  在按下*Enter*键后，输出将被打印出来：

    ```go
    [{"id":"541727b08ea48e5e5d5bb189","title":"Best
              Beatle?",
              "options": ["john","paul","george","ringo"]},
            {"id":"541728728ea48e5e5d5bb18a","title":"Favorite
              language?",
              "options": ["go","java","javascript","ruby"]}]

    ```

1.  虽然看起来不太美观，但您可以看到 API 返回了您的数据库中的投票。发出以下命令来创建一个新的投票：

    ```go
    curl --data '{"title":"test","options":
             ["one","two","three"]}'
             -X POST http://localhost:8080/polls/?key=abc123

    ```

1.  再次获取列表以查看包含的新投票：

    ```go
    curl -X GET http://localhost:8080/polls/?
             key=abc123

    ```

1.  复制并粘贴其中一个 ID，并调整 URL 以具体指向该投票：

    ```go
    curl -X GET
              http://localhost:8080/polls/541727b08ea48e5e5d5bb189?
              key=abc123
    [{"id":"541727b08ea48e5e5d5bb189",","title":"Best  Beatle?",
              "options": ["john","paul","george","ringo"]}]

    ```

1.  现在我们只看到选定的投票。让我们发送一个`DELETE`请求来删除这个投票：

    ```go
    curl -X DELETE  
              http://localhost:8080/polls/541727b08ea48e5e5d5bb189? 
              key=abc123

    ```

1.  现在我们再次获取所有投票时，我们会看到披头士的投票已经消失了：

    ```go
    curl -X GET http://localhost:8080/polls/?key=abc123
    [{"id":"541728728ea48e5e5d5bb18a","title":"Favorite    
              language?","options":["go","java","javascript","ruby"]}]

    ```

既然我们知道我们的 API 按预期工作，现在是时候构建一个能够正确消费 API 的东西了。

# 消费该 API 的 Web 客户端

我们将构建一个超简单的 Web 客户端，该客户端消费通过我们的 API 公开的能力和数据，使用户能够与我们在上一章以及本章早期构建的投票系统进行交互。我们的客户端将由三个网页组成：

+   一个显示所有投票的`index.html`页面

+   一个显示特定投票结果的`view.html`页面

+   一个允许用户创建新投票的`new.html`页面

在`api`文件夹旁边创建一个名为`web`的新文件夹，并将以下内容添加到`main.go`文件中：

```go
package main 
import ( 
  "flag" 
  "log" 
  "net/http" 
) 
func main() { 
  var addr = flag.String("addr", ":8081", "website address") 
  flag.Parse() 
  mux := http.NewServeMux() 
  mux.Handle("/", http.StripPrefix("/",  
    http.FileServer(http.Dir("public")))) 
  log.Println("Serving website at:", *addr) 
  http.ListenAndServe(*addr, mux) 
} 

```

这几行 Go 代码真正凸显了语言和 Go 标准库的美丽。它们代表了一个完整、高度可扩展的静态网站托管程序。该程序接受一个`addr`标志，并使用熟悉的`http.ServeMux`类型从名为`public`的文件夹中提供静态文件。

### 小贴士

在构建 UI 的同时构建接下来的几页，需要编写大量的 HTML 和 JavaScript 代码。由于这不是 Go 代码，如果你不想全部输入，请随意前往这本书的 GitHub 仓库[`github.com/matryer/goblueprints`](https://github.com/matryer/goblueprints)并复制粘贴。你也可以根据需要包含 Bootstrap 和 jQuery 库的最新版本，但可能与后续版本存在实现差异。
