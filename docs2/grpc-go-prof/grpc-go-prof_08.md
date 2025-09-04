# 8

# 更多重要功能

我们之前看到 gRPC 为我们提供了许多重要的开箱即用的功能，使我们的工作变得更简单。在本章中，我们将深入了解一些 gRPC 未提供但由社区提供的功能。它们通常建立在 gRPC 功能之上，以提供更多便利。它们还提供了一种实现最常见实践的方法，以保护和优化您的 API。

在本章中，我们将介绍以下主要内容：

+   验证请求消息

+   创建中间件

+   验证请求

+   跟踪 API 调用

+   应用速率限制

+   错误重试

到本章结束时，我们将了解中间件是什么以及它们的作用。我们将通过学习名为`protoc-gen-validate`和`go-grpc-middleware`的出色社区项目来实现这一点。

# 技术要求

对于本章，您可以在附带的 GitHub 仓库中找到相关代码，文件夹名为`chapter8`([`github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter8`](https://github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter8))。

# 验证请求

我们将要做的第一件事是减少检查请求消息某些属性的代码。我们将使用`protoc`的`protoc-gen-validate`插件，该插件帮助我们为某些消息生成验证代码。这对于检查任务描述长度和截止日期的使用场景非常有用。我们只需调用生成的`Validate()`函数，它就会告诉我们请求消息的要求是否得到满足。

我们要生成此代码的第一件事是安装插件。这是一个由 Buf 维护的插件，您可以这样获取它：

```go
$ go install github.com/envoyproxy/protoc-gen-validate
```

一旦我们有了这个，我们现在就可以使用 protoc 的`--validate_out`选项了。

现在，无论我们是手动使用 protoc 还是使用 Buf CLI，我们都需要从 GitHub 仓库复制`validate.proto`文件。此文件可以在以下位置找到：[`github.com/bufbuild/protoc-gen-validate/blob/main/validate/validate.proto`](https://github.com/bufbuild/protoc-gen-validate/blob/main/validate/validate.proto)。我们将将其复制到`validate`目录下的`proto`文件夹中：

```go
proto
└── validate
    └── validate.proto
```

现在，我们可以将此文件导入其他 proto 文件，并使用提供的验证规则作为字段选项。

让我们以`proto/todo/v2/todo.proto`中的`AddTaskRequest`为例。目前，我们有以下内容：

```go
message AddTaskRequest {
  string description = 1;
  google.protobuf.Timestamp due_date = 2;
}
```

如我们所知，每次我们尝试在服务器端添加一个`任务`时，我们都会检查描述是否为空，以及`due_date`是否大于`time.Now()`。

我们现在将把这个逻辑编码到我们的 `proto` 文件中。首先，我们需要做的是导入 `validate.proto` 文件。然后，我们将能够访问 `validate.rules` 字段选项，它包含多个类型的规则集。我们将处理 `string` 和 `Timestamp`，我们将使用 `min_len` 和 `gt_now` 字段。第一个描述了在调用 `Validate` 时字符串应该具有的最小长度，第二个告诉我们提供的 `Timestamp` 应该在未来：

```go
import "validate/validate.proto";
//...
message AddTaskRequest {
  string description = 1 [
    (validate.rules).string.min_len = 1
  ];
  google.protobuf.Timestamp due_date = 2 [
    (validate.rules).timestamp.gt_now = true
  ];
}
```

现在我们已经描述了这个逻辑，我们需要生成检查这个逻辑的代码。否则，这些选项毫无价值。为了生成此代码，我们将手动使用 protoc，然后我会向你展示如何使用 Buf 和 Bazel 来完成：

如前所述，我们可以使用插件在 `protoc` 中使用 `--validate_out` 选项。它看起来如下所示：

```go
$ protoc -Iproto --go_out=proto --go_opt=paths=
  source_relative --go-grpc_out=proto --go-grpc_opt=
    paths=source_relative --validate_out=
"lang=go,paths=source_relative:proto" proto/todo/v2/*.proto
```

注意到命令与我们过去运行的类似。我们只是添加了新的选项，并告诉它处理 Go 代码，并基于 `v2` 文件夹中的 proto 文件生成代码。

现在，在 Protobuf 和 gRPC 生成的代码之上，你应该在 `v2` 文件夹中有一个 `.pb.validate.go` 文件。它应该看起来像这样：

```go
proto/todo/v2
├── todo.pb.go
├── todo.pb.validate.go
├── todo.proto
└── todo_grpc.pb.go
```

在生成的文件中，你应该能够看到以下函数（以及其他函数）：

```go
// Validate checks the field values on Task with the rules
defined in the proto
// definition for this message. If any rules are violated,
the first error
// encountered is returned, or nil if there are no
violations.
func (m *Task) Validate() error {
  return m.validate(false)
}
```

这是我们要在服务器端的 `AddTask` 端点使用的函数。目前，我们有以下检查：

```go
func (s *server) AddTask(_ context.Context, in
*pb.AddTaskRequest) (*pb.AddTaskResponse, error) {
  if len(in.Description) == 0 {
    return nil, status.Error(
      codes.InvalidArgument,
      "expected a task description, got an empty string",
    )
  }
  if in.DueDate.AsTime().Before(time.Now().UTC()) {
    return nil, status.Error(
      codes.InvalidArgument,
      "expected a task due_date that is in the future",
    )
  }
  //...
}
```

让我们用 `Validate` 函数来替换它。我们只是简单地在 `in` 参数上调用该函数，如果它返回任何错误，我们将从函数返回错误，否则，我们将简单地继续我们的执行：

```go
func (s *server) AddTask(_ context.Context, in
*pb.AddTaskRequest) (*pb.AddTaskResponse, error) {
  if err := in.Validate(); err != nil {
    return nil, err
  }
  //...
}
```

如此简单，我们就避免了手动编写所有检查并尝试在不同端点保持错误信息一致。

我们现在可以进入 `main` 客户端，并逐个取消错误部分中的函数注释：

```go
func main() {
  //...
  fmt.Println("-------ERROR-------")
  addTask(c, "", dueDate)
  addTask(c, "not empty", time.Now().Add(-5*time.Second))
  fmt.Println("-------------------")
}
```

我们应该为第一个 `addTask` 获得以下错误：

```go
$ go run ./client 0.0.0.0:50051
-------ERROR-------
rpc error: code = Unknown desc = invalid AddTaskRequest
.Description: value length must be at least 1 runes
```

这是为第二个 `addTask` 添加的：

```go
$ go run ./client 0.0.0.0:50051
-------ERROR-------
rpc error: code = Unknown desc = invalid AddTaskRequest
.DueDate: value must be greater than now
```

注意到代码错误是 `Unknown`。截至本书编写时，`protoc-gen-validate` 似乎没有自定义错误代码。这可能在插件的 `v2` 版本中出现。然而，它为我们提供了一个简单的验证代码和清晰的错误信息。

## Buf

使用 Buf CLI 的 `protoc-gen-validate` 非常简单。我们将在我们的 YAML 文件中添加一些配置以生成代码。首先，我们需要在我们的 `buf.yaml` 文件中添加对 `protoc-gen-validate` 的依赖：

```go
version: v1
#...
deps:
- buf.build/envoyproxy/protoc-gen-validate
```

这告诉 Buf 在生成过程中需要 `protoc-gen-validate`。它将稍后自己找出如何拉取依赖。

之后，我们需要在 `buf.gen.yaml` 文件中配置插件：

```go
version: v1
plugins:
  #...
  - plugin: buf.build/bufbuild/validate-go
    out: proto
    opt: paths=source_relative
```

这些选项与我们之前手动输入的相同。现在，我们可以通过输入以下命令来常规生成：

```go
$ buf generate proto
```

你现在应该拥有与使用 `protoc` 命令获得相同的三个生成文件：`todo.pb.validate.go`、`todo_grpc.pb.go` 和 `todo.pb.go`。注意，在这种情况下，我们还为 `v1` 和 `validate.proto` 生成了代码。

## Bazel

和往常一样，我们首先需要在 `WORKSPACE.bazel` 文件中定义依赖项。我们将从 GitHub 获取 `protoc-gen-validate` 项目并加载其相关依赖项：

重要提示

下面的代码引用了一个名为 `PROTOC_GEN_VALIDATE_VERSION` 的变量。这个变量在 `chapter8` 文件夹中的 `versions.bzl` 文件中定义。我们在这里不包括它，以保持代码与版本无关。

```go
#...
git_repository(
  name = "com_envoyproxy_protoc_gen_validate",
  tag = PROTOC_GEN_VALIDATE_VERSIO**N**,
  remote = "https://github.com/bufbuild/protoc-gen-validate"
)

load("@com_envoyproxy_protoc_gen_validate//bazel:
  repositories.bzl", "pgv_dependencies")
load("@com_envoyproxy_protoc_gen_validate//
  :dependencies.bzl", "go_third_party")
pgv_dependencies()
# gazelle:repository_macro deps.bzl%go_third_party
go_third_party()
```

这样，我们现在需要更新 `deps.bzl` 文件中的依赖项。我们可以通过输入以下命令来完成：

```go
$ bazel run //:gazelle-update-repos
```

最后，我们需要生成代码并将其链接到我们现有的 `proto/todo/v2/BUILD.bazel` 中的 `todo go_library`。

首先需要添加对 `v2_proto proto_library` 中 `protoc-gen-validate` `validate.proto` 的依赖。这将允许 `todo.proto` 导入它：

```go
proto_library(
  name = "v2_proto",
  #…
  deps = [
    #...
    "@com_envoyproxy_protoc_gen_validate//
      validate:validate_proto",
  ],
)
```

然后，我们将用 `pgv_go_proto_library` 替换 `v2_go_proto go_proto_library` (`protoc-gen-validate` 库，以便生成的代码可以访问编译所需的任何 `protoc-gen-validate` 内部代码：

```go
load("@com_envoyproxy_protoc_gen_validate//bazel:pgv_proto_
  library.bzl", "pgv_go_proto_library")
//...
pgv_go_proto_library(
  name = "v2_go_proto",
  compilers = ["@io_bazel_rules_go//proto:go_grpc"],
  importpath = "github.com/PacktPublishing/gRPC-Go-for-Professionals/
    proto/todo/v2",
  proto = ":v2_proto",
  deps = ["@com_envoyproxy_protoc_gen_validate//
    validate:validate_go"],
)
```

最后，为了避免在下次运行 Gazelle 时对 `validate/validate.proto` 的模糊导入，我们将 `validate/validate.proto` 的导入（在 `proto/todo/v2/todo.proto` 中）映射到 `@com_envoyproxy_protoc_gen_validate//validate:validate_proto`（在 `protoc-gen-validate` 中定义）。在 `proto/todo/v2/BUILD.bazel` 文件顶部，我们可以添加以下 `Gazelle` 指令：

```go
# gazelle:resolve proto validate/validate.proto
  @com_envoyproxy_protoc_gen_validate//validate:
  validate_proto
```

现在我们已经用 `pgv_go_proto_library` 替换了旧的 `v2_go_proto`，依赖于这个库的代码将自动获得对生成的 `Validate` 函数的访问。

我们可以尝试运行服务器：

```go
$ bazel run //server:server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后运行带有取消注释的错误部分代码的客户端：

```go
$ bazel run //client:client 0.0.0.0:50051
-------ERROR-------
rpc error: code = Unknown desc = invalid AddTaskRequest
.Description: value length must be at least 1 runes
```

总结来说，我们看到了我们可以在 proto 文件中编码验证逻辑，并使用 `protoc-gen-validate` 自动生成验证代码。这简化了我们的代码，并在我们的 API 端点提供了一致的错误消息。

# 中间件 = 拦截器

在 gRPC 的上下文中，一个中间件是一个拦截器。它位于开发人员注册的代码和实际的 gRPC 框架之间。当 gRPC 从线路上接收到一些数据时，它首先将数据通过中间件传递，然后如果允许通过，数据将到达实际的端点处理器。

这些中间件通常用于保护端点免受恶意行为者的攻击或强制执行某些先决条件。一个保护 API 的例子是限制客户端的速率。这是在给定时间段内客户端可以发出的请求数量的限制，这是很重要的，因为它可以防止许多攻击，如暴力攻击、DoS 和 DDoS 攻击，以及网页抓取。为了强制执行某些先决条件，我们已经看到了一个例子，其中客户端在能够调用端点之前需要被认证。

在查看社区提供的中间件之前，我想提醒你，我们已经在*第七章*中创建了中间件。我们只是从未把它们称为中间件。实际上，我们创建了两个，如下所示：

```go
opts := []grpc.ServerOption{
  //...
  grpc.ChainUnaryInterceptor(unaryAuthInterceptor,
    unaryLogInterceptor),
  grpc.ChainStreamInterceptor(streamAuthInterceptor,
    streamLogInterceptor),
}
```

如果你还记得，这些中间件首先会检查是否存在一个值为`authd`的`auth_token`头，并且该头存在。然后，如果情况如此，它将在终端上记录 API 调用并继续执行我们为 API 端点编写的代码。

总结一下，中间件是一个可以根据某些条件中断执行的拦截器，它的作用是保护 API 端点。

# 验证请求

在本节和接下来的内容中，我们将简化我们目前拥有的中间件。首先，我们将从简化认证过程开始。我们在上一章中看到，我们可以轻松地创建一个检查头中认证令牌的拦截器。在本节中，我们将更进一步，使其更加简单。

重要提示

gRPC 支持通过 RBAC 策略重试请求的认证，而不需要第三方库。然而，配置相当冗长，并且没有很好的文档记录。如果你有兴趣尝试，可以查看以下示例：[`github.com/grpc/grpc-go/blob/master/examples/features/authz/README.md`](https://github.com/grpc/grpc-go/blob/master/examples/features/authz/README.md)。

在之前编写拦截器时，我们需要为单一拦截器创建以下函数：

```go
func unaryAuthInterceptor(ctx context.Context, req
  interface{}, info *grpc.UnaryServerInfo, handler
    grpc.UnaryHandler) (interface{}, error)
```

以及以下类似的流拦截器：

```go
func streamAuthInterceptor(srv interface{}, ss
  grpc.ServerStream, info *grpc.StreamServerInfo, handler
    grpc.StreamHandler) error
```

虽然这为我们提供了关于调用、上下文等信息，但也使得我们的代码变得简短，我们需要考虑如何在流拦截器和单一拦截器之间共享常见的业务逻辑。

通过我们将要添加的中间件，我们将专注于我们的逻辑，并且我们将在我们的 gRPC 服务器中像以前一样轻松地注册拦截器。这个中间件是 GitHub 仓库中名为 `go-grpc-middleware` 的认证中间件（[`github.com/grpc-ecosystem/go-grpc-middleware`](https://github.com/grpc-ecosystem/go-grpc-middleware)）。它将使我们摆脱为拦截器添加的复杂认证函数定义，并允许我们直接使用预定义的拦截器中的 `validateAuthToken` 函数进行注册。

要开始，我们将从我们的 `server` 文件夹中获取依赖项：

```go
$ go get github.com/grpc-ecosystem/go-grpc-middleware/v2/
  interceptors/auth
```

然后，我们将从 `server/interceptors.go` 文件中移除 `unaryAuthInterceptor` 和 `streamAuthInterceptor`。由于新的认证中间件将为我们处理一切，所以我们不再需要它们。

最后，我们将前往 `server/main.go`，在那里我们将用 `auth.UnaryServerInterceptor` 和 `auth.StreamServerInterceptor` 替换旧的拦截器。这两个拦截器接受一个 `AuthFunc`，它基本上代表了认证逻辑。在我们的情况下，我们将传递我们的 `validateAuthToken`。

`AuthFunc` 类型看起来是这样的：

```go
type AuthFunc func(ctx context.Context) (context.Context,
error)
```

因此，我们需要稍微修改 `validateAuthToken` 以返回一个上下文和/或一个错误。我们新的函数将如下所示：

```go
func validateAuthToken(ctx context.Context)
  (context.Context, error) {
  md, ok := metadata.FromIncomingContext(ctx)
  if !ok {
    return nil, status.Errorf(/*...*/)
  }
  if t, ok := md["auth_token"]; ok {
    switch {
    case len(t) != 1:
      return nil, status.Errorf(/*...*/)
    case t[0] != "authd":
      return nil, status.Errorf(/*...*/)
    }
  } else {
    return nil, status.Errorf(/*...*/)
  }
  return ctx, nil
}
```

这使我们能够在 gRPC 服务器中注册 `validateAuthToken`。我们新的主函数现在将如下所示：

```go
import (
  //...
  "github.com/grpc-ecosystem/go-grpc-middleware/v2/
    interceptors/auth"
)
func main() {
  //...
  opts := []grpc.ServerOption{
    //...
    grpc.ChainUnaryInterceptor
      (auth.UnaryServerInterceptor(validateAuthToken),
         unaryLogInterceptor),
    grpc.ChainStreamInterceptor(auth
      .StreamServerInterceptor(validateAuthToken),
        streamLogInterceptor),
  }
  //...
}
```

现在，我们应该能够运行服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

我们还运行了客户端：

```go
$ go run ./client 0.0.0.0:50051
```

我们没有收到任何错误，并且它的工作方式与之前相似。然而，为了测试中间件是否正常工作，我们可以临时修改客户端的拦截器以添加错误的认证头（`client/interceptors.go`）：

```go
const authTokenValue string = "notauthd"
```

如果我们重新运行客户端，我们应该得到以下错误：

```go
$ go run ./client 0.0.0.0:50051
--------ADD--------
rpc error: code = Unauthenticated desc = incorrect
  auth_token
```

这证明了我们的中间件按预期工作，并且我们可以仅依靠 `validAuthToken` 来进行认证检查。

## Bazel

为了使用 Bazel 运行它，我们需要更新我们的依赖项并将新的依赖项链接到 `server/BUILD.bazel` 中的 `server_lib` 目标。因此，我们首先运行 `gazelle-update-repos` 命令，这将获取 `go-grpc-middleware` 依赖项：

```go
$ bazel run //:gazelle-update-repos
```

一旦我们有了这个，我们现在可以让 `gazelle` 命令将 `go-grpc-middleware` 依赖项包含到目标中：

```go
$ bazel run //:gazelle
```

最后，我们将能够运行我们的服务器：

```go
$ bazel run //server:server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

并且带有错误 `auth` 令牌的客户端应该给出以下信息：

```go
$ bazel run //client:client 0.0.0.0:50051
--------ADD--------
rpc error: code = Unauthenticated desc = incorrect
  auth_token
```

总结来说，在本节中，我们看到了我们可以通过使用 `go-grpc-middleware` 包来简化认证拦截器。它让我们专注于实际的逻辑，而不是如何编写可以注册到 gRPC 的拦截器。

# 记录 API 调用

在本节中，让我们简化日志拦截器。这就像我们在上一节中所做的那样，但我们将使用另一个中间件：日志中间件。

虽然这个中间件与许多不同的日志记录器集成，但我们将使用它与 Golang 的默认 `log` 包。这样，集成您喜欢的日志记录器将变得容易。

重要提示

下一个命令仅在您没有获取 `go-grpc-middleware` 的上一个依赖项时才需要。如果您是按章节顺序阅读的，那么您应该不需要它。

要开始，让我们获取中间件的依赖项。在 `server` 文件夹中，我们将运行以下命令：

```go
$ go get github.com/grpc-ecosystem/go-grpc-middleware/v2/
  interceptors/logging
```

现在，我们可以开始创建我们的日志记录器。我们将通过定义一个返回 `loggerFunc` 的函数来创建它。这是一个具有以下签名的函数：

```go
func(ctx context.Context, lvl logging.Level, msg string,
  fields ...any)
```

我们已经知道上下文是什么，但所有其余的都是针对日志记录器的。级别是一个日志级别，如 `Debug`、`Info`、`Warning` 或 `Error`。这通常用于根据严重程度过滤日志。然后，消息只是一个由日志中间件生成的消息，如 `":started call"` 或 `":finished call"`。这有助于我们理解日志的上下文。最后，字段是我们需要打印有用日志的所有其他信息。在我们的情况下，我们将使用服务名称和方法名称。这将使我们能够创建如下日志：

```go
INFO :started call todo.v2.TodoService UpdateTasks
```

一件难以理解的事情是 `fields` 参数。这是因为它被表示为 `any` 的 `vararg`。实际上，我们可以将其转换为映射，以获取特定的字段名称，如 `grpc.service`、`grpc.method` 等。为此，我们可以简单地编写以下代码：

```go
f := make(map[string]any, len(fields)/2)
i := logging.Fields(fields).Iterator()
for i.Next() {
  k, v := i.At()
  f[k] = v
}
```

注意，我们正在创建一个长度为 `len(fields)/2` 的映射。这是因为 `fields` 参数中字段名称和它们的值是交错排列的。以下是一个例子：

```go
grpc.service todo.v2.TodoService    grpc.method ListTasks
```

你可以通过展开 `vararg` 来打印字段并亲自查看整个内容：

```go
log.Println(fields...)
```

现在我们有了这个知识，我们可以继续编写日志记录器。我们将创建一个名为 `logCalls` 的函数，它接受一个 `log.Logger`（来自 `golang` 标准库）作为参数，并返回一个 `logging.Logger`（来自日志中间件）。日志记录器的逻辑将是检查日志级别，将消息的级别添加到前面，然后我们将服务名称和方法名称添加到整个消息中：

```go
import (
  //...
 "github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/logging"
)
const grpcService = "grpc.service"
const grpcMethod = "grpc.method"
func logCalls(l *log.Logger) logging.Logger {
  return logging.LoggerFunc(func(_ context.Context, lvl
   logging.Level, msg string, fields ...any) {
    f := make(map[string]any, len(fields)/2)
    i := logging.Fields(fields).Iterator()
    for i.Next() {
      k, v := i.At()
      f[k] = v
    }
    switch lvl {
    case logging.LevelDebug:
      msg = fmt.Sprintf("DEBUG :%v", msg)
    case logging.LevelInfo:
      msg = fmt.Sprintf("INFO :%v", msg)
    case logging.LevelWarn:
      msg = fmt.Sprintf("WARN :%v", msg)
    case logging.LevelError:
      msg = fmt.Sprintf("ERROR :%v", msg)
    default:
      panic(fmt.Sprintf("unknown level %v", lvl))
    }
    l.Println(msg, f[grpcService], f[grpcMethod])
  })
}
```

现在，虽然这个方法总是准确的，因为我们可以在构建的映射中检索键，但这意味着每次调用这个拦截器时，我们都需要构建一个映射。这并不真的高效。我想在你看到高效的例子之前先展示完整的例子，这样你就能理解如何使用 `fields` 参数。

为了更高效，我们可以利用我们的服务和方法是始终位于索引 5 和 7 的这一事实。因此，我们将移除映射创建部分，我们将用 `5` 和 `7` 替换 `grpcService` 和 `grpcMethod`，并将访问 `fields` 的第 5 和第 7 个元素：

```go
const grpcService = 5
const grpcMethod = 7
func logCalls(l *log.Logger) logging.Logger {
  return logging.LoggerFunc(func(_ context.Context, lvl
    logging.Level, msg string, fields ...any) {
    // ...
    l.Println(msg, fields[grpcService], fields[grpcMethod])
  })
}
```

这要高效得多。现在，有一点需要提及的是，这不太安全。我们假设我们接收到的所有字段都将始终包含在相同索引处的`service`和`method`，并且我们的`fields`数组足够大。我们可以安全地假设，在撰写本文时，因为这些是始终按此顺序添加的常见字段。然而，如果库发生变化，你可能会尝试进行越界访问或获取不同的信息。请注意这一点。

最后一件我们需要做的事情是注册拦截器。这与我们之前所做的身份验证拦截器类似，但主要区别在于现在我们需要创建一个记录器并将其传递给`logCalls`函数。我们将使用 golang 的`log.Logger`，它在消息之前打印日期和时间。最后，我们将`logCalls`的结果传递给`logging.UnaryServerInterceptor`和`logging.StreamServerInterceptor`：

```go
import (
  //...
  "github.com/grpc-ecosystem/go-grpc-middleware/v2/
    interceptors/logging"
)
//...
func main() {
  //...
  logger := log.New(os.Stderr, "", log.Ldate|log.Ltime)
  opts := []grpc.ServerOption{
    //...
    grpc.ChainUnaryInterceptor(
      //...
      logging.UnaryServerInterceptor(logCalls(logger)),
    ),
    grpc.ChainStreamInterceptor(
      //...
      logging.StreamServerInterceptor(logCalls(logger)),
    ),
  }
  //...
}
```

之后，我们现在可以运行我们的服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

注意

在运行客户端之前，请确保将`client/interceptors.go`文件中`authTokenValue`的值替换为`authd`。

然后运行我们的客户端：

```go
$ go run ./client 0.0.0.0:50051
```

如果我们检查服务器正在运行的终端，我们应该会看到一些类似以下的消息：

```go
INFO :started call todo.v2.TodoService ListTasks
INFO :finished call todo.v2.TodoService ListTasks
```

总结来说，我们看到了，与身份验证中间件类似，我们只需在我们的 gRPC 服务器中添加一个记录器即可。我们还看到，通过将`fields varargs`转换为映射，我们可以访问比服务名和方法名更多的信息。最后，我们看到了一些字段在`vararg`中始终位于相同的位置，因此，我们不必为每个调用生成映射，可以直接通过索引访问信息。

# 跟踪 API 调用

在日志记录的基础上，它以开发者友好的方式简单地描述事件之外，你可能还需要获取可以被仪表板工具聚合的指标。这些指标可能包括每秒请求数、状态分布（`Ok`、`Internal`等），以及其他许多指标。在本节中，我们将使用 OpenTelemetry 和 Prometheus 对我们的代码进行仪表化，以便可以使用 Grafana 等工具创建仪表板。

首先要理解的是，我们将运行一个用于 Prometheus 指标的 HTTP 服务器。Prometheus 通过`/metrics`路由将指标暴露给外部工具，以便想要查询数据的工具可以了解所有可用的指标类型。

因此，为了创建这样的服务器，我们将获取 Prometheus 的 Go 库的依赖项。我们将通过获取对`go-grpc-middleware/providers/prometheus`的依赖来实现这一点。`Prometheus Go`库是这个库的传递依赖，我们仍然需要能够注册在`Prometheus`提供程序中定义的一些更多拦截器：

```go
$ go get github.com/grpc-ecosystem/go-grpc-middleware/
  providers/prometheus
```

现在，我们可以创建一个 HTTP 服务器，稍后它将被用来公开`/metrics`路由。我们将创建一个名为`newMetricsServer`的函数，它接受服务器运行地址：

重要提示

下面的代码解释了`server/main.go`文件中的每个部分。在这里显示整个文件可能会让人感到不知所措。因此，我们将遍历所有代码，你将能够在`main.go`中看到导入和整体结构。请注意，为了更好地解释，我们将在本节后面添加某些元素。如果你看到尚未展示的代码部分，请继续阅读，你将得到你所查看代码的解释。

```go
func newMetricsServer(httpAddr string) *http.Server {
  httpSrv := &http.Server{Addr: httpAddr}
  m := http.NewServeMux()
  httpSrv.Handler = m
  return httpSrv
}
```

现在我们有了 HTTP 服务器，我们将重构`main`函数，并将 gRPC 服务器的创建分离到另一个函数中：

```go
func newGrpcServer(lis net.Listener) (*grpc.Server, error) {
  creds, err := credentials.NewServerTLSFromFile
    ("./certs/server_cert.pem", "./certs/server_key.pem")
  if err != nil {
    return nil, err
  }
  logger := log.New(os.Stderr, "", log.Ldate|log.Ltime)
  opts := []grpc.ServerOption{
    //...
  }
  s := grpc.NewServer(opts...)
  pb.RegisterTodoServiceServer(s, &server{
    d: New(),
  })
  return s, nil
}
```

实际上没有发生任何变化；我们只是将创建过程分离成一个函数，以便我们可以在以后并行运行两个服务器。不过，在着手做那之前，我们的`main`函数也应该包含两个地址作为参数。第一个是为 gRPC 服务器准备的，另一个是为 HTTP 服务器准备的：

```go
func main() {
  args := os.Args[1:]
  if len(args) != 2 {
    log.Fatalln("usage: server [GRPC_IP_ADDR]
[METRICS_IP_ADDR]")
  }
  grpcAddr := args[0]
  httpAddr := args[1]
}
```

现在，我们可以处理同时运行两个服务器。我们将使用`errgroup`（[`pkg.go.dev/golang.org/x/sync/errgroup`](https://pkg.go.dev/golang.org/x/sync/errgroup)）包。它允许我们将多个 goroutines 添加到组中并等待它们。

我们首先需要为组创建一个上下文。我们将创建一个可取消的上下文，以便稍后我们可以释放服务器的资源：

```go
ctx := context.Background()
ctx, cancel := context.WithCancel(ctx)
defer cancel()
```

接下来，我们可以开始处理`SIGTERM`信号。这是因为当我们想要退出两个服务器时，我们将按*Ctrl* + *C*。这将发送`SIGTERM`信号，我们期望服务器能够优雅地关闭。为了处理这一点，我们将创建一个通道，当接收到`SIGTERM`信号时，该通道将被释放：

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
defer signal.Stop(quit)
```

之后，我们现在可以创建我们的两个服务器的组。我们首先将从我们创建的可取消上下文中创建组。然后，我们将使用`Go(func() error)`函数将 goroutines 添加到该组中。第一个 goroutine 将处理 gRPC 服务器的服务，第二个 goroutine 将处理 HTTP 服务器：

```go
lis, err := net.Listen("tcp", grpcAddr)
if err != nil {
  log.Fatalf("unexpected error: %v", err)
}
g, ctx := errgroup.WithContext(ctx)
grpcServer, err := newGrpcServer(lis)
if err != nil {
  log.Fatalf("unexpected error: %v", err)
}
g.Go(func() error {
  log.Printf("gRPC server listening at %s\n", grpcAddr)
  if err := grpcServer.Serve(lis); err != nil {
    log.Printf("failed to gRPC server: %v\n", err)
    return err
  }
  log.Println("gRPC server shutdown")
  return nil
})
metricsServer := newMetricsServer(httpAddr)
g.Go(func() error {
  log.Printf("metrics server listening at %s\n", httpAddr)
  if err := metricsServer.ListenAndServe(); err != nil &&
    err != http.ErrServerClosed {
    log.Printf("failed to serve metrics: %v\n", err)
    return err
  }
  log.Println("metrics server shutdown")
  return nil
})
```

现在我们有了组，我们可以等待上下文完成或等待`quit`通道接收事件：

```go
select {
  case <-quit:
    break
  case <-ctx.Done():
    break
}
```

一旦收到这些事件之一，我们将通过确保上下文完成（调用`cancel`函数）来启动资源的释放，最后，我们可以等待组完成我们注册的所有 goroutines：

```go
cancel()
timeoutCtx, timeoutCancel := context.WithTimeout(
  context.Background(),
  10*time.Second,
)
defer timeoutCancel()
log.Println("shutting down servers, please wait...")
grpcServer.GracefulStop()
metricsServer.Shutdown(timeoutCtx)
if err := g.Wait(); err != nil {
  log.Fatal(err)
}
```

最后，作为本节最终目标，我们需要添加跟踪功能。度量服务器将公开度量路由，而 gRPC 服务器将收集度量并将它们添加到 Prometheus 注册表中。这个注册表是一个收集器的集合。我们将一个或多个收集器注册到其中，然后注册表将收集不同的度量，最后，它将公开这些度量。

在创建注册表之前，我们首先使用`go-grpc-middleware/providers/prometheus`包中提供的`NewServerMetrics`函数创建一个收集器。然后，我们将实际创建注册表。最后，我们将注册收集器：

```go
srvMetrics := grpcprom.NewServerMetrics(
  grpcprom.WithServerHandlingTimeHistogram(
    grpcprom.WithHistogramBuckets([]float64{0.001, 0.01,
      0.1, 0.3, 0.6, 1, 3, 6, 9, 20, 30, 60, 90, 120}),
  ),
)
reg := prometheus.NewRegistry()
reg.MustRegister(srvMetrics)
```

注意，我们向`NewServerMetrics`传递了一个选项。这个选项将使我们能够根据调用延迟将调用放入不同的桶中。这基本上告诉我们有多少请求在 0.001 秒内被服务，0.01 秒，等等。

最后，我们将把注册表传递给 HTTP 服务器，以便它知道有哪些度量标准可用，并且我们将把收集器传递给我们的 gRPC 服务器，以便它可以将其推送到它：

```go
func newMetricsServer(httpAddr string, reg
  *prometheus.Registry) *http.Server {
  //...
  m.Handle("/metrics", promhttp.HandlerFor(reg,
    promhttp.HandlerOpts{}))
  //...
  return httpSrv
}
func newGrpcServer(lis net.Listener, srvMetrics
  *grpcprom.ServerMetrics) (*grpc.Server, error) {
  //...
  opts := []grpc.ServerOption{
    //...
    grpc.ChainUnaryInterceptor(
      otelgrpc.UnaryServerInterceptor(),
      srvMetrics.UnaryServerInterceptor(),
      //...
    ),
    grpc.ChainStreamInterceptor(
      otelgrpc.StreamServerInterceptor(),
      srvMetrics.StreamServerInterceptor(),
      //...
    ),
  }
  //...
}
func main() {
  //...
  grpcServer, err := newGrpcServer(lis, srvMetrics)
  //...
  metricsServer := newMetricsServer(httpAddr, reg)
  //...
}
```

注意，我们现在正在使用`opentelemetry`（`otelgrpc`）。这是一个工具，它允许我们自动从我们的 gRPC 服务器生成所有指标。然后，Prometheus 将选择那些由收集器（`srvMetrics`）选择的指标。最后，HTTP 服务器将能够公开这些指标。

要为 gRPC 获取 OpenTelemetry，我们只需要获取依赖项：

```go
$ go get go.opentelemetry.io/contrib/instrumentation/
  google.golang.org/grpc/otelgrpc
```

我们现在应该能够运行我们的服务器：

```go
$ go run ./server 0.0.0.0:50051 0.0.0.0:50052
metrics server listening at 0.0.0.0:50052
gRPC server listening at 0.0.0.0:50051
```

然后，我们可以针对`0.0.0.0:50051`地址运行我们的客户端：

```go
$ go run ./client 0.0.0.0:50051
```

在客户端调用全部服务后，我们可以查看如下指标：

```go
$ curl http://localhost:50052/metrics
```

你现在应该有如下所示的日志（简化以仅显示`AddTask`）：

```go
grpc_server_handled_total{grpc_code="OK",grpc_method="
  AddTask",grpc_service="todo.v2.TodoService",grpc_type=
    "unary"} 3
grpc_server_handling_seconds_bucket{grpc_method="AddTask",
  grpc_service="todo.v2.TodoService",grpc_type="unary",le=
    "0.001"} 3
grpc_server_handling_seconds_sum{grpc_method="AddTask",grpc
  _service="todo.v2.TodoService",grpc_type="unary"}
    0.000119291
grpc_server_msg_received_total{grpc_method="AddTask",grpc_
  service="todo.v2.TodoService",grpc_type="unary"} 3
grpc_server_msg_sent_total{grpc_method="AddTask",grpc_
  service="todo.v2.TodoService",grpc_type="unary"} 3
grpc_server_started_total{grpc_method="AddTask",grpc_
  service="todo.v2.TodoService",grpc_type="unary"} 3
```

这些指标意味着服务器接收了三个`AddTask`请求，在 0.001 秒内处理了它们（总计：0.000119291），并向客户端返回了三个响应。

显然，还有更多的事情要做来处理这些指标。然而，这可能需要一本书来详细说明。如果你对这个领域感兴趣，我鼓励你查看如何将 Prometheus 与 Grafana 等工具集成，以创建以更易于阅读的方式表示这些指标的仪表板。

## Bazel

我们需要更新依赖项，以便 Prometheus 和 OpenTelemetry 能够正常工作。为此，我们将运行`gazelle-update-repos`：

```go
$ bazel run //:gazelle-update-repos
```

然后，我们将运行`gazelle`以自动将依赖项链接到我们的代码：

```go
$ bazel run //:gazelle
```

最后，我们现在可以运行我们的服务器：

```go
$ bazel run //:server:server 0.0.0.0:50051 0.0.0.0:50052
metrics server listening at 0.0.0.0:50052
gRPC server listening at 0.0.0.0:50051
```

然后运行我们的客户端：

```go
$ bazel run //client:client 0.0.0.0:50051
```

总之，我们看到了如何通过使用 OpenTelemetry 和 Prometheus 从我们的 gRPC 服务器中获取指标。我们通过创建一个在`/metrics`路由上导出指标的第二个服务器，并通过使用 Prometheus 注册表和收集器，将 gRPC 服务器的指标交换到 HTTP 服务器中来实现这一点。

# 使用速率限制来保护 API

对于我们将要添加到服务器的最后一个拦截器，我们将使用一个速率限制器。更确切地说，我们将使用由`golang.org/x/time/rate`包提供的令牌桶速率限制器的实现。在本节中，我们不会深入探讨速率限制器是什么或如何构建一个——这超出了本书的范围，然而，你将看到如何在 gRPC 的上下文中使用一个速率限制器（一个现成的或自定义的）。

我们需要做的第一件事是获取速率限制器的依赖项：

```go
$ go get golang.org/x/time/rate
```

重要提示

下一个命令仅在您没有获取到 `go-grpc-middleware` 的上一个依赖项时才需要。如果您是按章节顺序进行的，那么您应该不需要它。

然后我们获取拦截器的依赖项：

```go
$ go get github.com/grpc-ecosystem/go-grpc-middleware/v2/
  interceptors/ratelimit
```

现在，我们将创建一个名为 `limit.go` 的文件，其中将包含我们的逻辑和 `rate.Limiter` 的包装器。我们创建这样的包装器是因为我们稍后将要使用的拦截器需要限流器实现一个名为 `Limit` 的函数，该函数接受一个上下文作为参数，而 `rate.Limiter` 没有这样的函数：

```go
package main
import (
  "context"
  "fmt"
  "golang.org/x/time/rate"
)
type simpleLimiter struct {
  limiter *rate.Limiter
}
func (l *simpleLimiter) Limit(_ context.Context) error {
  if !l.limiter.Allow() {
    return fmt.Errorf("reached Rate-Limiting %v", l
      .limiter.Limit())
  }
  return nil
}
```

注意我们只是检查速率限制器是否允许（或不允许）调用通过。如果不允许，我们返回一个错误，否则返回 `nil`。

最后要做的事情是在拦截器中注册 `simpleLimiter`。我们将创建一个类型为 `rate.Limiter` 的实例，每秒 2 个令牌（称为 `r`），突发大小为 4（称为 `b`）。如果您对这些参数不清楚，我们建议您阅读关于限流器的文档（[`pkg.go.dev/golang.org/x/time/rate#Limiter`](https://pkg.go.dev/golang.org/x/time/rate#Limiter)）：

```go
import (
  //...
  "github.com/grpc-ecosystem/go-grpc-middleware/v2/
    interceptors/ratelimit"
)
func newGrpcServer(lis net.Listener, srvMetrics
  *grpcprom.ServerMetrics) (*grpc.Server, error) {
  //...
  limiter := &simpleLimiter{
    limiter: rate.NewLimiter(2, 4),
  }
  opts := []grpc.ServerOption{
    //...
    grpc.ChainUnaryInterceptor(
      ratelimit.UnaryServerInterceptor(limiter),
      //...
    ),
    grpc.ChainStreamInterceptor(
      ratelimit.StreamServerInterceptor(limiter),
      //...
    ),
  }
  //...
}
```

就这些。现在我们已经为我们的 API 启用了速率限制。我们现在可以运行我们的服务器：

```go
$ go run ./server 0.0.0.0:50051 0.0.0.0:50052
metrics server listening at 0.0.0.0:50052
gRPC server listening at 0.0.0.0:50051
```

然后，我们可以尝试每秒执行超过两个调用。这不应该很难。事实上，您通常可以运行一次客户端，它应该会失败。但为了确保它失败，请多次运行客户端。在 Linux 和 Mac 上，您可以运行以下命令：

```go
$ for i in {1..10}; do go run ./client 0.0.0.0:50051; done
```

在 Windows（PowerShell）上，您可以运行以下命令：

```go
$ foreach ($item in 1..10) { go run ./client 0.0.0.0:50051 }
```

您应该会看到一些查询返回响应，然后，您应该能够很快看到以下消息：

```go
rpc error: code = ResourceExhausted desc =
  /todo.v2.TodoService/UpdateTasks is rejected by
    grpc_ratelimit middleware, please retry later. reached
      Rate-Limiting 2
```

显然，我们的速率非常低，这在生产中并不实用。我们选择这么低的速率是为了向您展示如何进行速率限制。在生产中，您将会有特定的业务需求需要遵循。您将不得不调整我们展示的代码以匹配这些需求。

## Bazel

为了使用 Bazel 运行此示例，我们需要更新仓库并运行 Gazelle 以将新的依赖项 (`golang.org/x/time/rate`) 导入到我们的库中：

```go
$ bazel run //:gazelle-update-repos
$ bazel run //:gazelle
```

之后，您应该能够像这样运行服务器：

```go
$ bazel run //server:server 0.0.0.0:50051 0.0.0.0:50052
```

总结来说，我们看到了我们可以在我们的 gRPC 服务器中集成速率限制器。`go-grpc-middleware` 的速率限制拦截器使得添加现成的实现或自定义实现变得容易。

# 重试调用

到目前为止，我们只专注于服务器端。现在让我们看看客户端的一个重要特性。这个特性是根据状态码重试失败的调用。这可能对网络不可靠的使用场景很有趣。如果我们得到一个 `Unavailable` 错误代码，我们将以指数级增加的等待时间进行重试。这是因为我们不希望过于频繁地重试并超载网络。

重要提示

gRPC 支持无需第三方库的重试。然而，配置相当冗长，并且文档不是很好。如果您有兴趣尝试，可以查看以下示例：[`github.com/grpc/grpc-go/blob/master/examples/features/retry/README.md`](https://github.com/grpc/grpc-go/blob/master/examples/features/retry/README.md)。

让我们获取我们需要的依赖项（`client`文件夹）：

```go
$ go get github.com/grpc-ecosystem/go-grpc-middleware/v2/
  interceptors/retry
```

然后，我们可以为重试定义一些选项。我们将定义重试的次数和错误代码。我们希望重试 3 次，使用指数退避（从 100 毫秒开始），错误代码为`Unavailable`：

```go
retryOpts := []retry.CallOption{
  retry.WithMax(3),
  retry.WithBackoff(retry.BackoffExponential(100 *
    time.Millisecond)),
  retry.WithCodes(codes.Unavailable),
}
```

然后，我们只需将这些选项传递给`retry`包提供的拦截器：

```go
import (
  //...
  "github.com/grpc-ecosystem/go-grpc-middleware/v2/
    interceptors/retry"
)
func main() {
  //...
  retryOpts := []retry.CallOption{
    //...
  }
  opts := []grpc.DialOption{
    //...
    grpc.WithChainUnaryInterceptor(
      retry.UnaryClientInterceptor(retryOpts...),
      //...
    ),
    grpc.WithChainStreamInterceptor(
      retry.StreamClientInterceptor(retryOpts...),
      //...
    ),
    //...
  }
  //...
}
```

重要提示

对于客户端流，无法进行重试。如果您尝试在这样一个 RPC 端点上进行重试，您将收到以下错误：`rpc error: code = Unimplemented desc = grpc_retry: cannot retry on ClientStreams, set grpc_retry.Disable()`。因此，添加`retry.StreamClientInterceptor`有一定的风险。我们只是想向您展示一些流也可以进行重试。

一旦我们有了这个，我们现在就遇到了一个问题。我们的 API 在本地运行，我们几乎不可能得到一个`Unavailable`错误。所以，为了测试和演示，我们将暂时让我们的`AddTask`直接返回这样的错误。在`server/impl.go`中，我们可以注释掉函数的其余部分，并添加以下内容：

```go
func (s *server) AddTask(_ context.Context, in
  *pb.AddTaskRequest) (*pb.AddTaskResponse, error) {
  return nil, status.Errorf(
    codes.Unavailable,
    "unexpected error: %s",
    "unavailable",
  )
}
```

现在，我们运行我们的服务器：

```go
$ go run ./server 0.0.0.0:50051 0.0.0.0:50052
metrics server listening at 0.0.0.0:50052
gRPC server listening at 0.0.0.0:50051
```

然后运行我们的客户端：

```go
$ go run ./client 0.0.0.0:50051
--------ADD--------
rpc error: code = Unavailable desc = unexpected error:
  unavailable
```

我们得到一个错误。虽然这看起来只执行了一个查询，但如果您回顾您的服务器，您应该能够看到以下内容：

```go
INFO :started call todo.v2.TodoService AddTask
WARN :finished call todo.v2.TodoService AddTask
INFO :started call todo.v2.TodoService AddTask
WARN :finished call todo.v2.TodoService AddTask
INFO :started call todo.v2.TodoService AddTask
WARN :finished call todo.v2.TodoService AddTask
```

这实际上是三个请求。

## Bazel

总是如此，您需要运行`gazelle-update-repos`和`gazelle`以获取新的依赖项并将它们链接到您的库：

```go
$ bazel run //:gazelle-update-repos
$ bazel run //:gazelle
```

现在您应该能够正确运行您的客户端：

```go
$ bazel run //client:client 0.0.0.0:50051
--------ADD--------
rpc error: code = Unavailable desc = unexpected error:
  unavailable
```

总结来说，在本节中，我们看到了可以根据某些条件进行重试，使用指数退避，并且持续一定时间。重试是一个重要的功能，因为网络通常不可靠，我们不希望用户每次出现问题都要手动重试。

# 摘要

在本章中，我们探讨了通过使用社区项目，如`protoc-gen-validate`或`go-grpc-middleware`，我们可以获得的关键功能。我们看到了我们可以在我们的 proto 文件中编码请求验证逻辑。这使得我们的代码更精简，并且在整个 API 端点之间提供错误消息的一致性。

然后，我们探讨了中间件是什么以及如何创建一个。我们从重构我们的身份验证和日志拦截器开始。我们看到了通过使用`go-grpc-middleware`，我们可以专注于拦截器的实际逻辑，并且有更少的样板代码要处理。

之后，我们看到我们可以从我们的 API 中公开跟踪数据。我们使用了 OpenTelemetry 和 Prometheus 从 gRPC API 收集数据，并通过 HTTP 服务器公开它。

然后，我们学习了如何在我们的 API 上应用速率限制。这有助于防止欺诈行为者或缺陷客户端过载我们的服务器。我们使用了令牌桶算法和一个现成的速率限制器实现来对我们的 API 应用限制。

最后，我们还看到我们可以通过与重试中间件一起工作在客户端使用拦截器。这让我们可以根据错误代码重试调用，最多重试次数，并且可选地使用指数退避。

在下一章中，我们将讨论 gRPC API 的开发生命周期，如何确保它们的正确性，如何调试它们，以及如何部署它们。

# 问答

1.  protoc-gen-validate 插件的目的是什么？

    1.  在`.proto`文件中提供检查逻辑

    1.  生成验证代码

    1.  两者都是

1.  `go-grpc-middleware`用于什么？

    1.  提供常用的拦截器

    1.  生成验证代码

1.  哪个中间件用于将事件显示为可读文本？

    1.  `tracing`

    1.  `auth`

    1.  `logging`

1.  哪个中间件用于限制每秒请求的数量？

    1.  `tracing`

    1.  `ratelimit`

    1.  `auth`

# 答案

1.  C

1.  A

1.  C

1.  B

# 挑战

+   通过使用日志中间件简化你在上一章中创建的客户端日志记录器。

+   检查`protoc-gen-validate`规则([`github.com/bufbuild/protoc-gen-validate/blob/main/README.md`](https://github.com/bufbuild/protoc-gen-validate/blob/main/README.md))，并简化你在上一章挑战中添加的错误处理。

+   检查[`github.com/grpc-ecosystem/go-grpc-middleware/tree/v2`](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/v2)中可用的其他中间件，并尝试实现一个。一个例子可以是选择器中间件。

+   基于服务器公开的指标创建一个简单的 Grafana 仪表板。一个例子可以是显示成功请求百分比的仪表板。
