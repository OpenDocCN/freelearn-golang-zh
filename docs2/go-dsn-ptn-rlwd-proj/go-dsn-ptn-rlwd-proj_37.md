# 第十章. 使用 Go kit 框架的 Go 微服务

**微服务**是离散的组件，它们协同工作，为更大的应用程序提供功能性和业务逻辑，通常通过网络协议（如 HTTP/2 或某些其他二进制传输）进行通信，并分布到许多物理机器上。每个组件与其他组件隔离，它们接受定义良好的输入并产生定义良好的输出。同一服务的多个实例可以在多个服务器上运行，并且可以在它们之间进行负载均衡。如果设计得当，单个实例的失败不会导致整个系统崩溃，并且在运行时可以启动新的实例以帮助处理负载峰值。

Go kit（参考[`gokit.io`](https://gokit.io)）是由 Peter Bourgon（Twitter 上的`@peterbourgon`）创立的，用于构建具有微服务架构的应用程序的分布式编程工具包，目前由一群 Gophers 在开源社区中维护。它旨在解决构建此类系统时许多基础（有时可能有些枯燥）方面的问题，同时鼓励良好的设计模式，让您能够专注于构成您产品或服务的业务逻辑。

Go kit 并不试图从头解决每个问题；相反，它集成了许多流行的相关服务来解决**SOA**（**面向服务的架构**）问题，例如服务发现、度量、监控、日志记录、负载均衡、断路器以及许多其他正确运行大规模微服务的重要方面。当我们使用 Go kit 手动构建服务时，您会注意到我们将编写大量的模板或框架代码，以便使一切正常工作。

对于小型产品和服务，以及小型开发团队，您可能会决定直接暴露一个简单的 JSON 端点更容易，但 Go kit 在大型团队中表现尤为出色，用于构建具有许多不同服务的大量系统，每个服务在架构中运行数十或数百次。具有一致的日志记录、仪表化、分布式跟踪，并且每个组件都与下一个相似，这意味着运行和维护此类系统变得显著更容易。

> “Go kit 的最终目的是在服务内部鼓励良好的设计实践：SOLID 设计、领域驱动设计或六边形架构等。它并不是教条地遵循任何一种，而是试图使良好的设计/软件工程变得可行。” ——Peter Bourgon

在本章中，我们将构建一些解决各种安全挑战的微服务（在一个名为`vault`的项目中）——在此基础上，我们可以构建更多的功能。业务逻辑将保持非常简单，这样我们就可以专注于学习构建微服务系统的原则。

### 注意

作为技术选择，有一些 Go kit 的替代方案；它们大多数有类似的方法，但优先级、语法和模式不同。在开始项目之前，请确保您已经考虑了其他选项，但本章中学到的原则将适用于所有情况。

具体来说，在本章中，您将学习：

+   如何使用 Go kit 手动编码一个微服务

+   gRPC 是什么以及如何使用它来构建服务器和客户端

+   如何使用 Google 的协议缓冲区和相关工具以高效二进制格式描述服务和进行通信

+   Go kit 中的端点如何允许我们编写单个服务实现，并通过多种传输协议将其公开

+   Go kit 包含的子包如何帮助我们解决许多常见问题

+   中间件如何让我们在不触及实现本身的情况下包装端点以适应其行为

+   如何将方法调用描述为请求和响应消息

+   如何对我们的服务进行速率限制以保护免受流量激增的影响

+   一些其他的 Go 语言惯用技巧和技巧

本章中的一些代码行跨越了许多行；它们是以溢出的内容在下一行右对齐的方式编写的，如下例所示：

```go
func veryLongFunctionWithLotsOfArguments(one string, two int, three
 http.Handler, four string) (bool, error) { 
  log.Println("first line of the function") 
} 

```

前面的代码片段中的前三行应该写在一行中。不用担心；Go 编译器会足够友好地指出您是否出错。

# 介绍 gRPC

当涉及到我们的服务如何相互通信以及客户端如何与服务通信时，有许多选项，Go kit 并不关心（更确切地说，它并不介意——它足够关心，以至于提供了许多流行机制的实现）。实际上，我们能够为我们的用户提供多个选项，并让他们决定他们想要使用哪一个。我们将添加对熟悉的 JSON over HTTP 的支持，但我们也将引入一个新的 API 技术选择。

gRPC，即 Google 的**远程过程调用**，是一个开源机制，用于通过网络调用远程运行的代码。它使用 HTTP/2 进行传输，并使用协议缓冲区来表示构成服务和消息的数据。

RPC 服务与 RESTful 网络服务不同，因为您不是使用定义良好的 HTTP 标准来更改数据（就像您使用 REST 那样——使用`POST`创建某物，使用`PUT`更新某物，使用`DELETE`删除某物等），而是触发一个远程函数或方法，传递预期的参数，并得到一个或多个数据响应。

为了突出差异，想象一下我们正在创建一个新用户。在一个 RESTful 的世界里，我们可以发出如下请求：

```go
POST /users 
{ 
  "name": "Mat", 
  "twitter": "@matryer" 
} 

```

我们可能会得到如下响应：

```go
201 Created 
{ 
  "id": 1, 
  "name": "Mat", 
  "twitter": "@matryer" 
} 

```

RESTful 调用表示对资源状态的查询或更改。在 RPC 世界中，我们会使用生成的代码来代替，以便进行二进制序列化的过程调用，这些调用在 Go 中感觉更像正常的方法或函数。

与 RESTful 服务和 gPRC 服务之间唯一的另一个关键区别是，gPRC 而不是 JSON 或 XML，使用一种称为**协议缓冲区**的特殊格式。

# 协议缓冲区

协议缓冲区（在代码中称为`protobuf`）是一种非常小且编码和解码非常快的二进制序列化格式。您使用声明性迷你语言以抽象方式描述数据结构，并生成源代码（多种语言），以便用户轻松读写数据。

你可以把协议缓冲区看作是 XML 的现代替代品，只不过数据结构的定义与内容分开，而内容是以二进制格式而不是文本格式。

当你查看真实示例时，可以清楚地看到其好处。如果我们想在 XML 中表示一个有名字的人，我们可以这样写：

```go
<person> 
  <name>MAT</name> 
</person> 

```

这大约占用 30 个字节（不包括空白）。让我们看看它在 JSON 中的样子：

```go
{"name":"MAT"} 

```

现在我们已经缩减到 14 个字节，但结构仍然内嵌在内容中（名称字段与值一起展开）。

在协议缓冲区中，等效内容只需 5 个字节。以下表格显示了每个字节，以及 XML 和 JSON 表示的前五个字节，以供比较。**描述**行解释了**内容**行中字节的含义：

| **字节** | **1** | **2** | **3** | **4** | **5** |
| --- | --- | --- | --- | --- | --- |
| **内容** | 0a | 03 | 4d | 61 | 72 |
| **描述** | 类型（字符串） | 长度（3） | M | A | T |
| **XML** | < | p | e | r | s |
| **JSON** | { | " | n | a | m |

结构定义存在于一个特殊的`.proto`文件中，与数据分开。

仍然有许多情况下，XML 或 JSON 比协议缓冲区更适合，而在决定使用的数据格式时，文件大小并不是唯一的衡量标准，但对于固定模式结构和远程过程调用，或者对于真正大规模运行的应用程序，它是一个因合理原因而流行的选择。

## 安装协议缓冲区

有一些工具可以编译并生成协议缓冲区的源代码，您可以从项目的 GitHub 主页[`github.com/google/protobuf/releases`](https://github.com/google/protobuf/releases)获取。一旦下载了文件，解压它并将 bin 文件夹中的`protoc`文件放置在您的机器上的一个适当文件夹中：一个在您的`$PATH`环境变量中提到的文件夹。

一旦 protoc 命令就绪，我们需要添加一个插件，这将允许我们使用 Go 代码。在终端中执行以下命令：

```go
go get -u github.com/golang/protobuf/{proto,protoc-gen-go}

```

这将安装两个我们将要使用的包。

## 协议缓冲区语言

为了定义我们的数据结构，我们将使用协议缓冲区语言的第三个版本，称为`proto3`。

在你的`$GOPATH`中创建一个名为`vault`的新文件夹，并在其中创建一个名为`pb`的子文件夹。`pb`包将存放我们的协议缓冲区定义和生成的源代码。

我们将定义一个名为`Vault`的服务，它有两个方法，`Hash`和`Validate`：

| **方法** | **描述** |
| --- | --- |
| `Hash` | 为给定的密码生成一个安全的哈希值。可以存储哈希值而不是存储明文密码。 |
| `Validate` | 给定一个密码和之前生成的哈希值，`Validate`方法将检查密码是否正确。 |

每个服务调用都有一个请求和响应对，我们也将定义这些。在`pb`中，将以下代码插入一个名为`vault.proto`的新文件中：

```go
syntax = "proto3"; 
package pb; 
service Vault { 
  rpc Hash(HashRequest) returns (HashResponse) {} 
  rpc Validate(ValidateRequest) returns (ValidateResponse) {} 
} 
message HashRequest { 
  string password = 1; 
} 
message HashResponse { 
  string hash = 1; 
  string err = 2; 
} 
message ValidateRequest { 
  string password = 1; 
  string hash = 2; 
} 
message ValidateResponse { 
  bool valid = 1; 
} 

```

### 提示

为了节省纸张，已经删除了垂直空白，但如果你认为在各个块之间添加空格可以提高可读性，你可以这样做。

我们在文件中首先指定的是使用`proto3`语法，以及生成源代码的包名为`pb`。

`service`块定义了`Vault`以及在其下方定义的两个方法——`HashRequest`、`HashResponse`、`ValidateRequest`和`ValidateResponse`消息。服务块中以`rpc`开头的行表示我们的服务由两个远程过程调用组成：`Hash`和`Validate`。

消息内部字段采用以下格式：

```go
type name = position; 

```

`type`是一个描述标量值类型的字符串，例如`string`、`bool`、`double`、`float`、`int32`、`int64`等。`name`是一个人类可读的字符串，用于描述字段，例如`hash`和`password`。位置是一个整数，表示该字段在数据流中的位置。这很重要，因为内容是字节流，将内容与定义对齐对于能够使用该格式至关重要。此外，如果我们稍后添加（甚至重命名）字段（协议缓冲区的一个关键设计特性），我们可以在不破坏期望以特定顺序包含某些字段的组件的情况下这样做；它们将继续无改动地工作，忽略新数据，并透明地传递它。

### 提示

有关支持类型的完整列表以及对该语言的深入探讨，请查看[`developers.google.com/protocol-buffers/docs/proto3`](https://developers.google.com/protocol-buffers/docs/proto3)上的文档。

注意，每个方法调用都有一个相关的请求和响应对。这些是当远程方法被调用时通过网络发送的消息。

由于哈希方法只接受一个密码字符串参数，因此`HashRequest`对象包含一个密码字符串字段。像正常的 Go 函数一样，响应可能包含一个错误，这就是为什么`HashResponse`和`ValidateResponse`都有两个字段。在 proto3 中，没有像 Go 中那样的专用`error`接口，所以我们打算将错误转换为字符串。

## 生成 Go 代码

Go 无法理解 proto3 代码，但幸运的是，我们之前安装的协议缓冲编译器和 Go 插件可以将它翻译成 Go 可以理解的东西：Go 代码。

在终端中，导航到`pb`文件夹，并运行以下命令：

```go
protoc vault.proto --go_out=plugins=grpc:.

```

这将生成一个名为`vault.pb.go`的新文件。打开该文件并检查其内容。它为我们做了很多工作，包括定义消息，甚至为我们创建了`VaultClient`和`VaultServer`类型，这将允许我们分别消费和公开服务。

### 提示

如果你对生成的其余代码（文件描述符看起来特别有趣）感兴趣，你可以自由地解码。现在，我们将相信它工作正常，并使用`pb`包来构建我们的服务实现。

# 构建服务

最后，无论我们的架构中正在进行什么其他黑暗魔法，它最终都会归结为调用某个 Go 方法，执行一些工作，并返回一个结果。所以接下来我们要做的是定义和实现 Vault 服务本身。

在`vault`文件夹内，向一个新创建的`service.go`文件中添加以下代码：

```go
// Service provides password hashing capabilities. 
type Service interface { 
  Hash(ctx context.Context, password string) (string,
    error) 
  Validate(ctx context.Context, password, hash string)
    (bool, error) 
} 

```

此接口定义了服务。

### 提示

你可能会认为`VaultService`比仅仅`Service`更好，但请记住，由于这是一个 Go 包，它将在外部被视为`vault.Service`，这听起来很顺耳。

我们定义了两个方法：`Hash`和`Validate`。每个方法都将`context.Context`作为第一个参数，然后是正常的`string`参数。响应也是正常的 Go 类型：`string`、`bool`和`error`。

### 提示

一些库可能仍然需要旧的上下文依赖项，即`golang.org/x/net/context`，而不是 Go 1.7 首次提供的`context`包。注意错误关于混合使用的问题，并确保你导入的是正确的。

设计微服务的一部分是注意状态存储的位置。即使你将在单个文件中实现服务的各种方法，并且可以访问全局变量，你也绝不应该使用它们来存储每个请求或甚至每个服务的状态。重要的是要记住，每个服务可能会在多个物理机器上多次运行，每个机器都无法访问其他机器的全局变量。

在这个精神下，我们将使用一个空的`struct`来实现我们的服务，这实际上是一个整洁的 Go 惯用技巧，可以将方法组合在一起，以便在不存储对象本身中的任何状态的情况下实现接口。向`service.go`添加以下`struct`：

```go
type vaultService struct{} 

```

### 提示

如果实现确实需要任何依赖项（例如数据库连接或配置对象），你可以将它们存储在结构体中，并在函数体中使用方法接收器。

## 从测试开始

在可能的情况下，首先编写测试代码有许多优点，通常最终会提高代码的质量和可维护性。我们将编写一个单元测试，该测试将使用我们新的服务来散列并验证密码。

创建一个名为`service_test.go`的新文件，并添加以下代码：

```go
package vault 
import ( 
  "testing" 
  "golang.org/x/net/context" 
) 
func TestHasherService(t *testing.T) { 
  srv := NewService() 
  ctx := context.Background() 
  h, err := srv.Hash(ctx, "password") 
  if err != nil { 
    t.Errorf("Hash: %s", err) 
  } 
  ok, err := srv.Validate(ctx, "password", h) 
  if err != nil { 
    t.Errorf("Valid: %s", err) 
  } 
  if !ok { 
    t.Error("expected true from Valid") 
  } 
  ok, err = srv.Validate(ctx, "wrong password", h) 
  if err != nil { 
    t.Errorf("Valid: %s", err) 
  } 
  if ok { 
    t.Error("expected false from Valid") 
  } 
} 

```

我们将通过`NewService`方法创建一个新的服务，然后使用它来调用`Hash`和`Validate`方法。我们甚至测试了一个不愉快的案例，即我们输入了错误的密码，并确保`Validate`返回`false`——否则，它将非常不安全。

## Go 语言中的构造函数

在其他面向对象的语言中，**构造函数**是一种特殊的函数，用于创建类的实例。它执行任何初始化并接受所需的参数，例如依赖项等。在这些语言中，通常只有一种创建对象的方式，但它往往具有奇怪的语法或依赖于命名约定（例如，函数名称与类名相同）。

Go 语言没有构造函数；它更简单，只有函数，并且由于函数可以返回参数，构造函数将只是一个全局函数，它返回一个可用的结构体实例。Go 语言的简单哲学驱使语言设计者做出这类决策；而不是强迫人们学习关于构建对象的新概念，开发者只需要学习函数的工作方式，他们就可以使用函数构建构造函数。

即使我们在构建一个对象的过程中没有进行任何特殊的工作（例如初始化字段、验证依赖关系等），有时添加一个构建函数也是值得的。在我们的情况下，我们不想通过暴露`vaultService`类型来膨胀 API，因为我们已经暴露了我们的`Service`接口类型，并且将其隐藏在构造函数中是一种实现这一点的不错方式。

在`vaultService`结构定义下方，添加`NewService`函数：

```go
// NewService makes a new Service. 
func NewService() Service { 
  return vaultService{} 
} 

```

这不仅阻止了我们暴露内部结构，而且如果将来我们需要对`vaultService`进行更多工作以准备其使用，我们也可以在不更改 API 的情况下完成，因此不需要我们的包的用户在他们的端进行任何更改，这对于 API 设计来说是一个巨大的胜利。

## 使用 bcrypt 散列和验证密码

我们将在服务中实现的第一个方法是`Hash`。它将接受一个密码并生成一个散列。然后可以将生成的散列（以及密码）传递给稍后要调用的`Validate`方法，该方法将确认或否认密码是否正确。

### 小贴士

要了解更多关于在应用程序中正确存储密码的方法，请查看 Coda Hale 关于该主题的博客文章，链接为[`codahale.com/how-to-safely-store-a-password/`](https://codahale.com/how-to-safely-store-a-password/)。

我们服务的目的是确保密码永远不会需要存储在数据库中，因为如果有人能够未经授权访问数据库，那将是一个安全风险。相反，您可以生成一个单向哈希（无法解码），它可以安全地存储，并且当用户尝试进行身份验证时，您可以执行检查以查看密码是否生成相同的哈希。如果哈希匹配，则密码相同；否则，它们不相同。

`bcrypt` 包提供了以安全可靠的方式为我们完成这项工作的方法。

向 `service.go` 添加 `Hash` 方法：

```go
func (vaultService) Hash(ctx context.Context, password
 string) (string, error) { 
  hash, err :=
    bcrypt.GenerateFromPassword([]byte(password),
    bcrypt.DefaultCost) 
  if err != nil { 
    return "", err 
  } 
  return string(hash), nil 
} 

```

确保您导入适当的 `bcrypt` 包（尝试 `golang.org/x/crypto/bcrypt`）。我们本质上是在包装 `GenerateFromPassword` 函数以生成哈希，然后在没有错误发生的情况下返回它。

注意，`Hash` 方法中的接收器只是 `(vaultService)`；我们没有捕获变量，因为我们无法在空 `struct` 上存储状态。

接下来，让我们添加 `Validate` 方法：

```go
func (vaultService) Validate(ctx context.Context,
  password, hash string) (bool, error) { 
  err := bcrypt.CompareHashAndPassword([]byte(hash),
    []byte(password)) 
  if err != nil { 
    return false, nil 
  } 
  return true, nil 
} 

```

与 `Hash` 类似，我们正在调用 `bcrypt.CompareHashAndPassword` 以安全方式确定密码是否正确。如果返回错误，则表示有问题，我们返回 `false` 表示。否则，当密码有效时，我们返回 `true`。

# 使用请求和响应模拟方法调用

由于我们的服务将通过各种传输协议公开，我们需要一种方式来模拟服务内外部的请求和响应。我们将通过为服务将接受或返回的每种消息类型添加一个 `struct` 来实现这一点。

为了让某人能够调用 `Hash` 方法并接收哈希密码作为响应，我们需要将以下两个结构添加到 `service.go`：

```go
type hashRequest struct { 
  Password string `json:"password"` 
} 
type hashResponse struct { 
  Hash string `json:"hash"` 
  Err  string `json:"err,omitempty"` 
} 

```

`hashRequest` 类型包含一个字段，即密码，而 `hashResponse` 包含生成的哈希以及一个 `Err` 字符串字段，以防出现错误。

### 小贴士

要模拟远程方法调用，您实际上是为传入参数创建一个 `struct`，并为返回参数创建一个 `struct`。

在继续之前，看看您是否可以为 `Validate` 方法模拟相同的请求/响应对。查看 `Service` 接口中的签名，检查它接受的参数，并考虑它需要做出什么样的响应。

我们将添加一个辅助方法（类型为 Go kit 的 `http.DecodeRequestFunc`），它将能够将 `http.Request` 的 JSON 主体解码到 `service.go`：

```go
func decodeHashRequest(ctx context.Context, r
 *http.Request) (interface{}, error) { 
  var req hashRequest 
  err := json.NewDecoder(r.Body).Decode(&req) 
  if err != nil { 
    return nil, err 
  } 
  return req, nil 
} 

```

`decodeHashRequest` 的签名由 Go kit 决定，因为它将稍后代表我们解码 HTTP 请求。在这个函数中，我们只是使用 `json.Decoder` 将 JSON 解码到我们的 `hashRequest` 类型中。

接下来，我们将为 `Validate` 方法添加请求和响应结构以及一个解码辅助函数：

```go
type validateRequest struct { 
  Password string `json:"password"` 
  Hash     string `json:"hash"` 
} 
type validateResponse struct { 
  Valid bool   `json:"valid"` 
  Err   string `json:"err,omitempty"` 
} 
func decodeValidateRequest(ctx context.Context, 
 r *http.Request) (interface{}, error) { 
  var req validateRequest 
  err := json.NewDecoder(r.Body).Decode(&req) 
  if err != nil { 
    return nil, err 
  } 
  return req, nil 
} 

```

在这里，`validateRequest` 结构体同时接受 `Password` 和 `Hash` 字符串，因为签名有两个输入参数，并返回一个包含名为 `Valid` 或 `Err` 的 `bool` 数据类型的响应。

我们需要做的最后一件事是编码响应。在这种情况下，我们可以编写一个单独的方法来编码 `hashResponse` 和 `validateResponse` 对象。

将以下代码添加到 `service.go` 中：

```go
func encodeResponse(ctx context.Context, 
  w http.ResponseWriter, response interface{})
error { 
  return json.NewEncoder(w).Encode(response) 
} 

```

我们的 `encodeResponse` 方法只是让 `json.Encoder` 帮我们完成工作。再次注意，签名是通用的，因为 `response` 类型是 `interface{}`；这是因为它是 Go kit 用于解码到 `http.ResponseWriter` 的机制。

## Go kit 中的端点

端点是 Go kit 中的一个特殊函数类型，它代表单个 RPC 方法。定义在 `endpoint` 包中：

```go
type Endpoint func(ctx context.Context, request
  interface{})  
(response interface{}, err error) 

```

端点函数接受 `context.Context` 和 `request`，并返回 `response` 或 `error`。`request` 和 `response` 类型是 `interface{}`，这告诉我们，在构建端点时，处理实际类型的责任在于实现代码。

端点很强大，因为，就像 `http.Handler`（和 `http.HandlerFunc`）一样，你可以用通用中间件包装它们，以解决在构建微服务时出现的各种常见问题：日志记录、跟踪、速率限制、错误处理等等。

Go kit 解决了在多种协议上传输的问题，并使用端点作为从它们的代码跳转到我们的代码的通用方式。例如，gRPC 服务器将在端口上监听，并在接收到适当的消息时调用相应的 `Endpoint` 函数。多亏了 Go kit，这一切对我们来说都是透明的，因为我们只需要用 Go 代码处理我们的 `Service` 接口。

### 为服务方法创建端点

为了将我们的服务方法转换为 `endpoint.Endpoint` 函数，我们将编写一个处理传入的 `hashRequest`、调用 `Hash` 服务方法，并根据响应构建和返回适当的 `hashResponse` 对象的函数。

将 `MakeHashEndpoint` 函数添加到 `service.go` 中：

```go
func MakeHashEndpoint(srv Service) endpoint.Endpoint { 
  return func(ctx context.Context, request interface{})
  (interface{}, error) { 
    req := request.(hashRequest) 
    v, err := srv.Hash(ctx, req.Password) 
    if err != nil { 
      return hashResponse{v, err.Error()}, nil 
    } 
    return hashResponse{v, ""}, nil 
  } 
} 

```

这个函数接受 `Service` 作为参数，这意味着我们可以从我们的 `Service` 接口的任何实现中生成端点。然后我们使用类型断言来指定请求参数实际上应该是 `hashRequest` 类型。我们调用 `Hash` 方法，传入上下文和 `Password`，这些是从 `hashRequest` 获取的。如果一切顺利，我们使用从 `Hash` 方法返回的值构建 `hashResponse` 并返回它。

让我们对 `Validate` 方法也做同样的事情：

```go
func MakeValidateEndpoint(srv Service) endpoint.Endpoint { 
  return func(ctx context.Context, request interface{})
  (interface{}, error) { 
    req := request.(validateRequest) 
    v, err := srv.Validate(ctx, req.Password, req.Hash) 
    if err != nil { 
      return validateResponse{false, err.Error()}, nil 
    } 
    return validateResponse{v, ""}, nil 
  } 
} 

```

这里，我们做的是同样的事情：获取请求并使用它来调用方法，然后再构建响应。请注意，我们从不会从 `Endpoint` 函数返回错误。

### 不同的错误级别

在 Go kit 中，主要有两种错误类型：传输错误（网络故障、超时、断开连接等）和业务逻辑错误（请求和响应的基础设施执行成功，但逻辑或数据中存在问题）。

如果`Hash`方法返回错误，我们不会将其作为第二个参数返回；相反，我们将构建`hashResponse`，其中包含错误字符串（可通过`Error`方法访问）。这是因为从端点返回的错误旨在指示传输错误，也许 Go kit 将通过某些中间件配置为重试调用几次。如果我们的服务方法返回错误，则被视为业务逻辑错误，并且对于相同的输入可能会始终返回相同的错误，因此不值得重试。这就是为什么我们将错误包装到响应中，并将其返回给客户端，以便他们可以处理它。

### 将端点包装到服务实现中

在 Go kit 中处理端点时，另一个非常有用的技巧是编写我们`vault.Service`接口的实现，它只是对底层端点进行必要的调用。

将以下结构体添加到`service.go`中：

```go
type Endpoints struct { 
  HashEndpoint     endpoint.Endpoint 
  ValidateEndpoint endpoint.Endpoint 
} 

```

为了实现`vault.Service`接口，我们将在我们的`Endpoints`结构体中添加两个方法，这些方法将构建一个请求对象，发送请求，并将生成的响应对象解析为要返回的正常参数。

添加以下`Hash`方法：

```go
func (e Endpoints) Hash(ctx context.Context, password
  string) (string, error) { 
  req := hashRequest{Password: password} 
  resp, err := e.HashEndpoint(ctx, req) 
  if err != nil { 
    return "", err 
  } 
  hashResp := resp.(hashResponse) 
  if hashResp.Err != "" { 
    return "", errors.New(hashResp.Err) 
  } 
  return hashResp.Hash, nil 
} 

```

我们使用`hashRequest`调用`HashEndpoint`，我们使用密码参数在将一般响应缓存到`hashResponse`并从中返回哈希值或错误之前创建它。

我们将对`Validate`方法做同样的事情：

```go
func (e Endpoints) Validate(ctx context.Context, password,
 hash string) (bool, error) { 
  req := validateRequest{Password: password, Hash: hash} 
  resp, err := e.ValidateEndpoint(ctx, req) 
  if err != nil { 
    return false, err 
  } 
  validateResp := resp.(validateResponse) 
  if validateResp.Err != "" { 
    return false, errors.New(validateResp.Err) 
  } 
  return validateResp.Valid, nil 
} 

```

这两个方法将使我们能够将我们创建的端点视为正常的 Go 方法；这对于我们在本章后面实际消费服务时非常有用。

# Go kit 中的 HTTP 服务器

当我们为我们的端点创建一个 HTTP 服务器以进行哈希和验证时，Go kit 的真实价值才显现出来。

创建一个名为`server_http.go`的新文件，并添加以下代码：

```go
package vault 
import ( 
  "net/http" 
  httptransport "github.com/go-kit/kit/transport/http" 
  "golang.org/x/net/context" 
) 
func NewHTTPServer(ctx context.Context, endpoints
 Endpoints) http.Handler { 
  m := http.NewServeMux() 
  m.Handle("/hash", httptransport.NewServer( 
    ctx, 
    endpoints.HashEndpoint, 
    decodeHashRequest, 
    encodeResponse, 
  )) 
  m.Handle("/validate", httptransport.NewServer( 
    ctx, 
    endpoints.ValidateEndpoint, 
    decodeValidateRequest, 
    encodeResponse, 
  )) 
  return m 
} 

```

我们正在导入`github.com/go-kit/kit/transport/http`包，并且（由于我们还在导入`net/http`包）告诉 Go 我们将显式地引用此包为`httptransport`。

我们正在使用标准库中的 `NewServeMux` 函数来构建 `http.Handler` 接口，并进行简单的路由，将 `/hash` 和 `/validate` 路径映射。我们获取 `Endpoints` 对象，因为我们想让我们的 HTTP 服务器提供这些端点，包括我们稍后可能添加的任何中间件。调用 `httptransport.NewServer` 是让 Go kit 为每个端点提供 HTTP 处理器的方法。像大多数函数一样，我们传入 `context.Context` 作为第一个参数，这将形成每个请求的基本上下文。我们还传入端点以及我们之前编写的解码和编码函数，以便服务器知道如何反序列化和序列化 JSON 消息。

# Go kit 中的 gRPC 服务器

使用 Go kit 添加 gRPC 服务器几乎和添加 JSON/HTTP 服务器一样简单，就像我们在上一节中所做的那样。在我们的生成代码（在 `pb` 文件夹中），我们得到了以下 `pb.VaultServer` 类型：

```go
type VaultServer interface { 
  Hash(context.Context, *HashRequest)
    (*HashResponse, error) 
  Validate(context.Context, *ValidateRequest)
    (*ValidateResponse, error) 
} 

```

这个类型与我们自己的 `Service` 接口非常相似，只是它接受生成请求和响应类而不是原始参数。

我们将首先定义一个将实现前面接口的类型。将以下代码添加到一个名为 `server_grpc.go` 的新文件中：

```go
package vault 
import ( 
  "golang.org/x/net/context" 
  grpctransport "github.com/go-kit/kit/transport/grpc" 
) 
type grpcServer struct { 
  hash     grpctransport.Handler 
  validate grpctransport.Handler 
} 
func (s *grpcServer) Hash(ctx context.Context,
 r *pb.HashRequest) (*pb.HashResponse, error) { 
  _, resp, err := s.hash.ServeGRPC(ctx, r) 
  if err != nil { 
    return nil, err 
  } 
  return resp.(*pb.HashResponse), nil 
} 
func (s *grpcServer) Validate(ctx context.Context,
 r *pb.ValidateRequest) (*pb.ValidateResponse, error) { 
  _, resp, err := s.validate.ServeGRPC(ctx, r) 
  if err != nil { 
    return nil, err 
  } 
  return resp.(*pb.ValidateResponse), nil 
} 

```

注意，你需要导入 `github.com/go-kit/kit/transport/grpc` 作为 `grpctransport`，以及生成的 `pb` 包。

`grpcServer` 结构体包含每个服务端点的字段，这次是 `grpctransport.Handler` 类型。然后，我们实现接口的方法，在适当的处理器上调用 `ServeGRPC` 方法。这个方法实际上会通过首先解码请求，调用适当的端点函数，获取响应，然后编码并发送回请求客户端来处理请求。

## 从协议缓冲类型转换为我们的类型

你会注意到我们正在使用 `pb` 包中的请求和响应对象，但请记住，我们自己的端点使用我们在 `service.go` 中早期添加的结构。我们将需要一个针对每种类型的方法来将其转换为我们的类型。

### 小贴士

接下来会有很多重复的输入；如果你愿意，可以从 GitHub 仓库 [`github.com/matryer/goblueprints`](https://github.com/matryer/goblueprints) 复制粘贴以节省你的手指。我们正在手动编码，因为这很重要，要理解构成服务的所有部分。 

在 `server_grpc.go` 中添加以下函数：

```go
func EncodeGRPCHashRequest(ctx context.Context,
  r interface{}) (interface{}, error) { 
  req := r.(hashRequest) 
  return &pb.HashRequest{Password: req.Password}, nil 
} 

```

这个函数是 Go kit 定义的 `EncodeRequestFunc` 函数，它用于将我们的 `hashRequest` 类型转换为可以与客户端通信的协议缓冲类型。它使用 `interface{}` 类型，因为它很通用，但在这个案例中，我们可以确信类型，因此我们将传入的请求转换为 `hashRequest`（我们的类型）然后使用适当的字段构建一个新的 `pb.HashRequest` 对象。

我们将为此进行编码和解码请求和响应，包括 hash 和 validate 端点的编码和解码。将以下代码添加到`server_grpc.go`中：

```go
func DecodeGRPCHashRequest(ctx context.Context,
 r interface{}) (interface{}, error) { 
  req := r.(*pb.HashRequest) 
  return hashRequest{Password: req.Password}, nil 
} 
func EncodeGRPCHashResponse(ctx context.Context,
 r interface{}) (interface{}, error) { 
  res := r.(hashResponse) 
  return &pb.HashResponse{Hash: res.Hash, Err: res.Err},
    nil 
} 
func DecodeGRPCHashResponse(ctx context.Context,
 r interface{}) (interface{}, error) { 
  res := r.(*pb.HashResponse) 
  return hashResponse{Hash: res.Hash, Err: res.Err}, nil 
} 
func EncodeGRPCValidateRequest(ctx context.Context,
 r interface{}) (interface{}, error) { 
  req := r.(validateRequest) 
  return &pb.ValidateRequest{Password: req.Password,
    Hash: req.Hash}, nil 
} 
func DecodeGRPCValidateRequest(ctx context.Context,
 r interface{}) (interface{}, error) { 
  req := r.(*pb.ValidateRequest) 
  return validateRequest{Password: req.Password,
    Hash: req.Hash}, nil 
} 
func EncodeGRPCValidateResponse(ctx context.Context,
 r interface{}) (interface{}, error) { 
  res := r.(validateResponse) 
  return &pb.ValidateResponse{Valid: res.Valid}, nil 
} 
func DecodeGRPCValidateResponse(ctx context.Context,
 r interface{}) (interface{}, error) { 
  res := r.(*pb.ValidateResponse) 
  return validateResponse{Valid: res.Valid}, nil 
} 

```

如您所见，为了使事物正常工作，需要进行大量的模板代码编写。

### 小贴士

代码生成（此处未涉及）在这里会有很大的应用，因为代码非常可预测且具有自我相似性。

为了使我们的 gRPC 服务器正常工作，最后要做的事情是提供一个辅助函数来创建我们的`grpcServer`结构体实例。在`grpcServer`结构体下面，添加以下代码：

```go
func NewGRPCServer(ctx context.Context, endpoints
 Endpoints) pb.VaultServer { 
  return &grpcServer{ 
    hash: grpctransport.NewServer( 
      ctx, 
      endpoints.HashEndpoint, 
      DecodeGRPCHashRequest, 
      EncodeGRPCHashResponse, 
    ), 
    validate: grpctransport.NewServer( 
      ctx, 
      endpoints.ValidateEndpoint, 
      DecodeGRPCValidateRequest, 
      EncodeGRPCValidateResponse, 
    ), 
  } 
} 

```

就像我们的 HTTP 服务器一样，我们接收一个基本上下文和通过 gRPC 服务器公开的实际`Endpoints`实现。我们创建并返回一个新的`grpcServer`类型实例，通过调用`grpctransport.NewServer`来设置`hash`和`validate`的处理程序。我们使用我们的`endpoint.Endpoint`函数来处理服务，并告诉服务使用我们哪些编码/解码函数来处理每种情况。

# 创建服务器命令

到目前为止，我们所有的服务代码都位于`vault`包内部。我们现在将使用这个包来创建一个新的工具，以暴露服务器功能。

在`vault`中创建一个新的文件夹名为`cmd`，并在其中创建另一个名为`vaultd`的文件夹。我们将把命令代码放在`vaultd`文件夹中，因为尽管代码将在`main`包中，但工具的名称默认为`vaultd`。如果我们只是将命令放在`cmd`文件夹中，工具将被构建成一个名为`cmd`的二进制文件，这会很令人困惑。

### 注意

在 Go 项目中，如果包的主要用途是导入到其他程序中（如 Go kit），则根级文件应组成包，并将具有适当的包名（不是`main`）。如果主要目的是命令行工具，如 Drop 命令（[`github.com/matryer/drop`](https://github.com/matryer/drop)），则根文件将在`main`包中。

这种做法的合理性在于可用性；当导入一个包时，您希望用户必须输入的字符串尽可能短。同样，当使用`go install`时，您希望路径既短又简洁。

我们将要构建的工具（后缀为`d`，表示它是守护程序或后台任务）将启动我们的 gRPC 和 JSON/HTTP 服务器。每个服务器将在自己的 goroutine 中运行，我们将捕获来自服务器的任何终止信号或错误，这将导致我们的程序终止。

在 Go kit 中，主函数最终会变得相当大，这是有意为之；有一个函数包含您整个微服务的全部内容；从那里，您可以深入了解细节，但它提供了每个组件的直观视图。

我们将在`vaultd`文件夹中的新`main.go`文件中逐步构建`main`函数，首先是一个相当大的导入列表：

```go
import ( 
  "flag" 
  "fmt" 
  "log" 
  "net" 
  "net/http" 
  "os" 
  "os/signal" 
  "syscall" 
  "your/path/to/vault" 
  "your/path/to/vault/pb" 
  "golang.org/x/net/context" 
  "google.golang.org/grpc" 
) 

```

应将 `your/path/to` 前缀替换为从 `$GOPATH` 到你的项目的实际路由。请注意上下文导入；在 Go kit 转移到 Go 1.7 的时候，你可能只需要输入 context 而不是这里列出的导入。最后，Google 的 `grpc` 包为我们提供了在网络上公开 gRPC 功能所需的一切。

现在，我们将组合我们的 `main` 函数；记住，从这一部分开始的全部内容都放在 `main` 函数体内：

```go
func main() { 
  var ( 
    httpAddr = flag.String("http", ":8080",
      "http listen address") 
    gRPCAddr = flag.String("grpc", ":8081",
      "gRPC listen address") 
  ) 
  flag.Parse() 
  ctx := context.Background() 
  srv := vault.NewService() 
  errChan := make(chan error) 

```

我们使用标志来允许操作团队决定在网络上公开服务时将监听哪些端点，但为 JSON/HTTP 服务器提供合理的默认值 `:8080`，为 gRPC 服务器提供 `:8081`。

然后，我们使用 `context.Background()` 函数创建一个新的上下文，该函数返回一个非空、空的上下文，没有指定取消或截止日期，也不包含任何值，非常适合我们所有服务的基上下文。请求和中间件可以自由地从该上下文中创建新的上下文对象，以便添加请求范围的数据或截止日期。

接下来，我们使用 `NewService` 构造函数为我们创建一个新的 `Service` 类型，并创建一个零缓冲通道，该通道可以接收错误（如果发生错误）。

我们现在将添加代码来捕获终止信号（如 *Ctrl + C*）并将错误发送到 `errChan`：

```go
  go func() { 
    c := make(chan os.Signal, 1) 
    signal.Notify(c, syscall.SIGINT, syscall.SIGTERM) 
    errChan <- fmt.Errorf("%s", <-c) 
  }() 

```

在这里，在一个新的 goroutine 中，我们要求 `signal.Notify` 通知我们何时接收到 `SIGINT` 或 `SIGTERM` 信号。当这种情况发生时，信号将通过 `c` 通道发送，此时我们将它格式化为字符串（调用其 `String()` 方法），并将其转换为错误，然后将其发送到 `errChan`，从而导致程序终止。

## 使用 Go kit 端点

是时候创建一个我们可以传递给服务器的端点实例了。将以下代码添加到主函数体中：

```go
  hashEndpoint := vault.MakeHashEndpoint(srv) 
  validateEndpoint := vault.MakeValidateEndpoint(srv) 
  endpoints := vault.Endpoints{ 
    HashEndpoint:     hashEndpoint, 
    ValidateEndpoint: validateEndpoint, 
  } 

```

我们将字段分配给端点辅助函数的输出，对于哈希和验证方法都是如此。我们为两者传递相同的服务，因此 `endpoints` 变量实际上是我们 `srv` 服务的包装器。

### 小贴士

你可能会想通过完全删除变量的赋值来整理这段代码，直接将辅助函数的返回值设置到结构体初始化的字段中，但当我们稍后添加中间件时，你会感谢这种方法的。

我们现在可以使用这些端点来启动我们的 JSON/HTTP 和 gRPC 服务器。

## 运行 HTTP 服务器

现在，我们将添加一个 goroutine 到主函数体中，用于创建和运行 JSON/HTTP 服务器：

```go
  // HTTP transport 
  go func() { 
    log.Println("http:", *httpAddr) 
    handler := vault.NewHTTPServer(ctx, endpoints) 
    errChan <- http.ListenAndServe(*httpAddr, handler) 
  }() 

```

Go kit 在我们的包代码中已经为我们完成了所有繁重的工作，所以我们只需调用 `NewHTTPServer` 函数，传递背景上下文和希望公开的服务端点，然后在调用标准库的 `http.ListenAndServe` 之前，该函数在指定的 `httpAddr` 中公开处理器功能。如果发生错误，我们将它发送到错误通道。

## 运行 gRPC 服务器

为了运行 gRPC 服务器，还需要做一些额外的工作，但仍然相当简单。我们必须创建一个低级别的 TCP 网络监听器，并在其上提供 gRPC 服务器。将以下代码添加到主函数主体中：

```go
  go func() { 
    listener, err := net.Listen("tcp", *gRPCAddr) 
    if err != nil { 
      errChan <- err 
      return 
    } 
    log.Println("grpc:", *gRPCAddr) 
    handler := vault.NewGRPCServer(ctx, endpoints) 
    gRPCServer := grpc.NewServer() 
    pb.RegisterVaultServer(gRPCServer, handler) 
    errChan <- gRPCServer.Serve(listener) 
  }() 

```

我们在指定的 `gRPCAddr` 端点创建 TCP 监听器，并将任何错误发送到 `errChan` 错误通道。我们使用 `vault.NewGRPCServer` 创建处理器，再次传递背景上下文和我们公开的 `Endpoints` 实例。

### 提示

注意到 JSON/HTTP 服务器和 gRPC 服务器实际上公开的是相同的服务——字面上是同一个实例。

然后，我们使用 Google 的 `grpc` 包创建一个新的 gRPC 服务器，并通过 `RegisterVaultServer` 函数使用我们自己的生成的 `pb` 包进行注册。

### 注意

`RegisterVaultService` 函数只是在我们自己的 `grpcServer` 上调用 `RegisterService`，但隐藏了自动生成的服务描述的内部细节。如果你查看 `vault.pb.go` 并搜索 `RegisterVaultServer` 函数，你会看到它引用了类似 `&_Vault_serviceDesc` 的内容，这是服务的描述。你可以随意挖掘生成的代码；元数据特别有趣，但本书不涉及这部分内容。

然后，我们要求服务器自己 `Serve`，如果发生错误，将错误信息发送到同一个错误通道。

### 提示

这章不涉及，但建议每个服务都应提供**传输层安全性**（**TLS**），特别是处理密码的服务。

## 防止主函数立即终止

如果我们在这里关闭了主函数，它将立即退出并终止所有服务器。这是因为我们正在做的所有防止这种情况发生的事情都在它自己的 goroutine 中。为了防止这种情况，我们需要一种方法在函数末尾阻塞，等待程序收到终止信号。

由于我们使用 `errChan` 错误通道来处理错误，这是一个完美的候选者。我们可以监听这个通道，在没有任何内容发送下来时，它会阻塞并允许其他 goroutine 执行它们的工作。如果出现问题（或收到终止信号），`<-errChan` 调用将解除阻塞并退出，所有 goroutine 都将停止。

在主函数的底部，添加最后的语句和结束块：

```go
  log.Fatalln(<-errChan) 
} 

```

当发生错误时，我们只是记录它并以非零代码退出。

## 通过 HTTP 消费服务

现在我们已经连接好了一切，我们可以使用 `curl` 命令或任何允许我们发送 JSON/HTTP 请求的工具来测试 HTTP 服务器。

在终端中，让我们首先运行我们的服务器。转到 `vault/cmd/vaultd` 文件夹并启动程序：

```go
go run main.go

```

服务器启动后，你将看到如下内容：

```go
http: :8080
grpc: :8081

```

现在，打开另一个终端，使用 `curl` 发出以下 HTTP 请求：

```go
curl -XPOST -d '{"password":"hernandez"}'
    http://localhost:8080/hash

```

我们正在向散列端点发送一个包含我们想要散列的密码的 JSON 体的 POST 请求。然后，我们得到如下内容：

```go
 {"hash":"$2a$10$IXYT10DuK3Hu.
      NZQsyNafF1tyxe5QkYZKM5by/5Ren"} 

```

### 小贴士

在这个例子中的散列值不会与你的匹配——有多个可接受的散列值，而且无法知道你会得到哪一个。确保复制并粘贴你的实际散列值（双引号内的所有内容）。

生成的散列值是我们根据指定的密码存储在数据存储中的值。然后，当用户再次尝试登录时，我们将使用他们输入的密码以及这个散列值向验证端点发送请求：

```go
curl -XPOST -d
     '{"password":"hernandez",
       "hash":"PASTE_YOUR_HASH_HERE"}'
     http://localhost:8080/validate

```

通过复制和粘贴正确的散列值并输入相同的 `hernandez` 密码来发送此请求，你将看到以下结果：

```go
{"valid":true}

```

现在，更改密码（这相当于用户输入错误）并将看到以下内容：

```go
{"valid":false}

```

你可以看到，我们 vault 服务的 JSON/HTTP 微服务暴露是完整且正在工作的。

接下来，我们将探讨如何消费 gRPC 版本。

# 构建 gRPC 客户端

与 JSON/HTTP 服务不同，gRPC 服务并不容易供人类交互。它们实际上是作为机器到机器的协议而设计的，因此如果我们想使用它们，我们必须编写一个程序。

为了帮助我们做到这一点，我们首先将在我们的 vault 服务内部添加一个新的包，名为 `vault/client/grpc`。它将提供一个对象，该对象在从 Google 的 `grpc` 包获取的 gRPC 客户端连接对象的基础上执行适当的调用、编码和解码，所有这些都在我们自己的 `vault.Service` 接口背后隐藏。因此，我们将能够将这个对象用作我们接口的另一个实现。

在 vault 中创建新的文件夹，以便你有 `vault/client/grpc` 的路径。如果你愿意，可以想象添加其他客户端，因此这似乎是一个建立良好模式的合适选择。

将以下代码添加到一个新的 `client.go` 文件中：

```go
func New(conn *grpc.ClientConn) vault.Service { 
  var hashEndpoint = grpctransport.NewClient( 
    conn, "Vault", "Hash", 
    vault.EncodeGRPCHashRequest, 
    vault.DecodeGRPCHashResponse, 
    pb.HashResponse{}, 
  ).Endpoint() 
  var validateEndpoint = grpctransport.NewClient( 
    conn, "Vault", "Validate", 
    vault.EncodeGRPCValidateRequest, 
    vault.DecodeGRPCValidateResponse, 
    pb.ValidateResponse{}, 
  ).Endpoint() 
  return vault.Endpoints{ 
    HashEndpoint:     hashEndpoint, 
    ValidateEndpoint: validateEndpoint, 
  } 
} 

```

`grpctransport` 包引用的是 `github.com/go-kit/kit/transport/grpc`。现在这可能会让你感到熟悉；我们正在根据指定的连接创建两个新的端点，这次明确指定了 `Vault` 服务名称和端点名称 `Hash` 和 `Validate`。我们从前端的 vault 包中传递适当的编码器和解码器以及空响应对象，然后将它们都包装在我们的 `vault.Endpoints` 结构中，这是我们添加的结构——它实现了 `vault.Service` 接口，该接口为我们触发了指定的端点。

## 消费服务的命令行工具

在本节中，我们将编写一个命令行工具（或 CLI-命令行界面），它将允许我们通过 gRPC 协议与我们的服务进行通信。如果我们用 Go 编写另一个服务，我们将以与编写 CLI 工具时相同的方式使用 vault 客户端包。

我们的工具将允许你在命令行中以流畅的方式访问服务，通过用空格分隔命令和参数，这样我们就可以像这样哈希密码：

```go
vaultcli hash MyPassword

```

我们将能够使用如下哈希来验证密码：

```go
vaultcli hash MyPassword HASH_GOES_HERE

```

在`cmd`文件夹中，创建一个名为`vaultcli`的新文件夹。添加一个`main.go`文件并插入以下主函数：

```go
func main() { 
  var ( 
    grpcAddr = flag.String("addr", ":8081",
     "gRPC address") 
  ) 
  flag.Parse() 
  ctx := context.Background() 
  conn, err := grpc.Dial(*grpcAddr, grpc.WithInsecure(),  
  grpc.WithTimeout(1*time.Second)) 
  if err != nil { 
    log.Fatalln("gRPC dial:", err) 
  } 
  defer conn.Close() 
  vaultService := grpcclient.New(conn) 
  args := flag.Args() 
  var cmd string 
  cmd, args = pop(args) 
  switch cmd { 
  case "hash": 
    var password string 
    password, args = pop(args) 
    hash(ctx, vaultService, password) 
  case "validate": 
    var password, hash string 
    password, args = pop(args) 
    hash, args = pop(args) 
    validate(ctx, vaultService, password, hash) 
  default: 
    log.Fatalln("unknown command", cmd) 
  } 
} 

```

确保你将`vault/client/grpc`包导入为`grpcclient`，将`google.golang.org/grpc`导入为`grpc`。你还需要导入`vault`包。

在调用 gRPC 端点以建立连接之前，我们像往常一样解析标志并获取背景上下文。如果一切顺利，我们将延迟关闭连接并使用该连接创建我们的 vault 服务客户端。记住，这个对象实现了我们的`vault.Service`接口，因此我们可以像调用正常 Go 方法一样调用这些方法，而无需担心通信是通过网络协议进行的。

然后，我们开始解析命令行参数，以决定采取哪种执行流程。

## 在 CLI 中解析参数

在命令行工具中解析参数非常常见，在 Go 中有一个整洁的惯用方法来做这件事。所有参数都可通过`os.Args`切片获得，或者如果你使用标志，则通过`flags.Args()`方法（该方法获取不带标志的参数）。我们想要从切片（从开始处）移除每个参数并按顺序消费它们，这将帮助我们决定通过程序采取哪种执行流程。我们将添加一个名为`pop`的辅助函数，它将返回第一个项目，并返回移除了第一个项目的切片。

我们将编写一个快速单元测试来确保我们的`pop`函数按预期工作。如果你想要尝试自己编写`pop`函数，那么一旦测试就绪，你应该这样做。记住，你可以通过在终端中导航到相应的文件夹并执行以下命令来运行测试：

```go
go test

```

在`vaultcli`内部创建一个名为`main_test.go`的新文件，并添加以下测试函数：

```go
func TestPop(t *testing.T) { 
  args := []string{"one", "two", "three"} 
  var s string 
  s, args = pop(args) 
  if s != "one" { 
    t.Errorf("unexpected "%s"", s) 
  } 
  s, args = pop(args) 
  if s != "two" { 
    t.Errorf("unexpected "%s"", s) 
  } 
  s, args = pop(args) 
  if s != "three" { 
    t.Errorf("unexpected "%s"", s) 
  } 
  s, args = pop(args) 
  if s != "" { 
    t.Errorf("unexpected "%s"", s) 
  } 
} 

```

我们期望每次调用`pop`都会返回切片中的下一个项目，并且在切片为空时返回空参数。

在`main.go`的底部添加`pop`函数：

```go
func pop(s []string) (string, []string) { 
  if len(s) == 0 { 
    return "", s 
  } 
  return s[0], s[1:] 
} 

```

## 通过提取 case 体来保持良好的视线

我们唯一剩下要做的事情是实现前面 switch 语句中提到的哈希和验证方法。

我们本可以将此代码嵌入到 switch 语句本身中，但这样会使主函数难以阅读，并且会隐藏不同缩进级别上的 happy path 执行，这是我们应尽量避免的。

相反，将 switch 语句中的情况跳转到专用函数中是一个好习惯，该函数接受它需要的任何参数。在主函数下方，添加以下哈希和验证函数：

```go
func hash(ctx context.Context, service vault.Service, 
  password string) { 
  h, err := service.Hash(ctx, password) 
  if err != nil { 
    log.Fatalln(err.Error()) 
  } 
  fmt.Println(h) 
} 
func validate(ctx context.Context, service vault.Service,  
  password, hash string) { 
  valid, err := service.Validate(ctx, password, hash) 
  if err != nil { 
    log.Fatalln(err.Error()) 
  } 
  if !valid { 
    fmt.Println("invalid") 
    os.Exit(1) 
  } 
  fmt.Println("valid") 
} 

```

这些函数只是简单地调用服务上的相应方法，并根据结果将结果记录或打印到控制台。如果验证方法返回 false，程序将以退出代码 1 退出，因为非零值表示错误。

## 从 Go 源代码安装工具

要安装此工具，我们只需在终端中导航到`vaultcli`文件夹，并输入以下命令：

```go
go install

```

假设没有错误，该包将被构建并部署到`$GOPATH/bin`文件夹，该文件夹应该已经列在你的`$PATH`环境变量中。这意味着工具已经准备好像终端中的正常命令一样使用。

部署的二进制文件名称将与文件夹名称匹配，这就是为什么即使在只构建单个命令的情况下，我们也在`cmd`文件夹内有一个额外的文件夹。

一旦安装了命令，我们就可以使用它来测试 gRPC 服务器。

前往`cmd/vaultd`并启动服务器（如果它还没有运行），只需输入以下命令：

```go
go run main.go

```

在另一个终端中，通过输入以下命令来哈希密码：

```go
vaultcli hash blanca

```

注意，哈希值被返回。现在让我们验证这个哈希值：

```go
vaultcli validate blanca PASTE_HASH_HERE

```

### 小贴士

哈希值可能包含会干扰您的终端的特殊字符，因此如果需要，您应该用引号转义字符串。

在 Mac 上，用`$'PASTE_HASH_HERE'`格式化参数以正确转义它。

在 Windows 上，尝试用感叹号包围参数：`!PASTE_HASH_HERE!`。

如果您输入了正确的密码，您会注意到您看到了单词`valid`；否则，您会看到`invalid`。

# 使用服务中间件进行速率限制

现在我们已经构建了一个完整的服务，我们将看到如何轻松地向我们的端点添加中间件，以扩展服务而不需要触及实际的实现。

在现实世界的服务中，限制它将尝试处理的请求数量是合理的，这样服务就不会过载。这可能发生在进程需要的内存比可用内存多的情况下，或者如果我们注意到性能下降，那么它可能消耗了太多的 CPU。在微服务架构中，解决这些问题的策略是添加另一个节点并分散负载，这意味着我们希望每个单独的实例都受到速率限制。

由于我们提供了客户端，我们应该在那里添加速率限制，这将防止过多的请求进入网络。但是，如果许多客户端同时尝试访问相同的服务，添加到服务器上的速率限制也是合理的。幸运的是，Go kit 中的端点既用于客户端也用于服务器，因此我们可以使用相同的代码在这两个地方添加中间件。

我们将添加一个基于 **Token Bucket** 的速率限制器，你可以在 [`en.wikipedia.org/wiki/Token_bucket`](https://en.wikipedia.org/wiki/Token_bucket) 上了解更多信息。Juju 团队已经编写了一个 Go 实现供我们使用，通过导入 `github.com/juju/ratelimit`，Go kit 也为这个实现提供了中间件，这将为我们节省大量时间和精力。

通用思路是我们有一个令牌桶，每个请求都需要一个令牌来完成其工作。如果没有令牌在桶中，我们就达到了限制，请求无法完成。桶在特定的时间间隔内填充。

导入 `github.com/juju/ratelimit` 并在我们创建 `hashEndpoint` 之前插入以下代码：

```go
rlbucket := ratelimit.NewBucket(1*time.Second, 5) 

```

`NewBucket` 函数创建一个新的速率限制桶，以每秒一个令牌的速度填充，最多五个令牌。这些数字对我们来说相当愚蠢，但我们希望在开发过程中能够手动达到我们的限制。

由于 Go kit 的 `ratelimit` 包与 Juju 的包同名，我们需要用不同的名称来导入它：

```go
import ratelimitkit "github.com/go-kit/kit/ratelimit"  

```

## Go kit 中的中间件

Go kit 中的端点中间件通过 `endpoint.Middleware` 函数类型指定：

```go
type Middleware func(Endpoint) Endpoint 

```

一段中间件仅仅是一个接收 `Endpoint` 并返回 `Endpoint` 的函数。记住，`Endpoint` 也是一个函数：

```go
type Endpoint func(ctx context.Context, request
  interface{}) (response interface{}, err error) 

```

这有点令人困惑，但它们与为我们 `http.HandlerFunc` 构建的包装器相同。一个中间件函数返回一个 `Endpoint` 函数，它在调用被包装的 `Endpoint` 之前和/或之后执行某些操作。传递给返回 `Middleware` 的函数的参数被闭包，这意味着它们可以通过闭包（无需在其他地方存储状态）在内部代码中使用。

我们将使用 Go kit 的 `ratelimit` 包中的 `NewTokenBucketLimiter` 中间件，如果我们查看代码，我们将看到它如何使用闭包并返回函数来在传递执行给 `next` 端点之前调用令牌桶的 `TakeAvailable` 方法：

```go
func NewTokenBucketLimiter(tb *ratelimit.Bucket)
  endpoint.Middleware { 
  return func(next endpoint.Endpoint) endpoint.Endpoint { 
    return func(ctx context.Context, request interface{})
    (interface{}, error) { 
      if tb.TakeAvailable(1) == 0 { 
        return nil, ErrLimited 
      } 
      return next(ctx, request) 
    } 
  } 
} 

```

在 Go kit 中出现了一种模式，即先获取端点，然后立即在其后放置所有中间件适配器。返回的函数在调用时接收端点，并且相同的变量会被覆盖为结果。

对于一个简单的例子，考虑以下代码：

```go
e := getEndpoint(srv) 
{ 
  e = getSomeMiddleware()(e) 
  e = getLoggingMiddleware(logger)(e) 
  e = getAnotherMiddleware(something)(e) 
} 

```

现在，我们将为此端点执行此操作；更新主函数内的代码以添加速率限制中间件：

```go
  hashEndpoint := vault.MakeHashEndpoint(srv) 
  { 
    hashEndpoint = ratelimitkit.NewTokenBucketLimiter
     (rlbucket)(hashEndpoint) 
  } 
  validateEndpoint := vault.MakeValidateEndpoint(srv) 
  { 
    validateEndpoint = ratelimitkit.NewTokenBucketLimiter
     (rlbucket)(validateEndpoint) 
  } 
  endpoints := vault.Endpoints{ 
    HashEndpoint:     hashEndpoint, 
    ValidateEndpoint: validateEndpoint, 
  } 

```

这里没有太多需要更改的；我们只是在将 `hashEndpoint` 和 `validateEndpoint` 变量分配给 `vault.Endpoints` 结构体之前更新它们。

## 手动测试速率限制器

为了查看我们的速率限制器是否工作，并且由于我们设置了如此低的阈值，我们可以仅使用我们的命令行工具进行测试。

首先，通过在运行服务器的终端窗口中按*Ctrl + C*来重启服务器（以便运行新代码）。这个信号将被我们的代码捕获，并将错误发送到`errChan`，导致程序退出。一旦它已经终止，重新启动它：

```go
go run main.go

```

现在，在另一个窗口中，让我们来哈希一些密码：

```go
vaultcli hash bourgon

```

重复这个命令几次——在大多数终端中，你可以按上箭头键并回车。你会注意到前几个请求是成功的，因为它们在限制范围内，但如果你稍微激进一些，在一秒内发出超过五个请求，你会注意到我们得到了错误：

```go
$ vaultcli hash bourgon
$2a$10$q3NTkjG0YFZhTG6gBU2WpenFmNzdN74oX0MDSTryiAqRXJ7RVw9sy
$ vaultcli hash bourgon
$2a$10$CdEEtxSDUyJEIFaykbMMl.EikxvV5921gs/./7If6VOdh2x0Q1oLW
$ vaultcli hash bourgon
$2a$10$1DSqQJJGCmVOptwIx6rrSOZwLlOhjHNC83OPVE8SdQ9q73Li5x2le
$ vaultcli hash bourgon
Invoke: rpc error: code = 2 desc = rate limit exceeded
$ vaultcli hash bourgon
Invoke: rpc error: code = 2 desc = rate limit exceeded
$ vaultcli hash bourgon
Invoke: rpc error: code = 2 desc = rate limit exceeded
$ vaultcli hash bourgon
$2a$10$kriTDXdyT6J4IrqZLwgBde663nLhoG3innhCNuf8H2nHf7kxnmSza

```

这表明我们的速率限制器正在工作。我们会在令牌桶重新填满之前看到错误，然后我们的请求再次得到满足。

## 优雅的速率限制

与其返回错误（这通常是一个相当严厉的回应），我们可能更希望服务器只是保留我们的请求，并在能够处理时完成它——这就是所谓的节流。对于这种情况，Go kit 提供了`NewTokenBucketThrottler`中间件。

更新中间件代码，使用这个中间件函数：

```go
  hashEndpoint := vault.MakeHashEndpoint(srv) 
  { 
    hashEndpoint = ratelimitkit.NewTokenBucketThrottler(rlbucket,
     time.Sleep)(hashEndpoint) 
  } 
  validateEndpoint := vault.MakeValidateEndpoint(srv) 
  { 
    validateEndpoint = ratelimitkit.NewTokenBucketThrottler(rlbucket,
      time.Sleep)(validateEndpoint) 
  } 
  endpoints := vault.Endpoints{ 
    HashEndpoint:     hashEndpoint, 
    ValidateEndpoint: validateEndpoint, 
  } 

```

`NewTokenBucketThrottler`的第一个参数与之前的端点相同，但现在我们添加了一个`time.Sleep`的第二个参数。

### 注意

Go kit 允许我们通过指定在需要延迟时应该发生什么来定制行为。在我们的例子中，我们传递了`time.Sleep`，这是一个将请求执行暂停指定时间的函数。如果你想做不同的事情，可以在这里编写自己的函数，但现在这个方法就足够了。

现在重复之前的测试，但这次，请注意我们永远不会得到错误——相反，终端会在请求可以满足之前挂起一秒钟。

# 摘要

通过构建一个真实的微服务示例，我们在本章中涵盖了大量的内容。没有代码生成，涉及的工作量很大，但对于大型团队和大型微服务架构来说，这些投资是值得的，因为你可以构建出构成系统的自相似、离散组件。

我们学习了 gRPC 和协议缓冲如何为我们提供客户端和服务器之间的高效传输通信。使用`proto3`语言，我们定义了我们的服务，包括消息，并使用工具生成了一个 Go 包，为我们提供了客户端和服务器代码。

我们探讨了 Go kit 的基本原理以及我们如何使用端点来描述我们的服务方法。当涉及到构建 HTTP 和 gRPC 服务器时，我们让 Go kit 为我们做繁重的工作，通过利用项目中包含的包。我们看到了中间件函数如何让我们轻松地将端点适应，例如，限制服务器需要处理的流量量。

我们还学习了 Go 语言中的构造函数，这是一种解析传入命令行参数的巧妙技巧，以及如何使用`bcrypt`包来哈希和验证密码，这是一个明智的方法，可以帮助我们避免存储密码。

构建微服务还有很多内容，建议您访问 Go kit 网站[`gokit.io`](https://gokit.io)或加入 gophers.slack.com 上的`#go-kit`频道进行更多了解。

现在我们已经构建了我们的 Vault 服务，我们需要考虑我们的选项以便将其部署到野外。在下一章中，我们将我们的微服务打包成 Docker 容器，并部署到 Digital Ocean 的云平台。
