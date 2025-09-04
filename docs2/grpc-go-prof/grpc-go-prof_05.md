

# gRPC 端点的类型

在本章中，我们将看到您可以编写的不同类型的 gRPC 端点。对于每个端点，我们将了解我们正在讨论的通信类型的背后理念，并在我们的 Protobuf 服务中定义一个 RPC 端点。最后，我们将实现该端点并编写一个客户端来消费该端点。本章的最终目标是实现一个 TODO API，它将允许我们创建、更新、删除和列出我们的 TODO 列表中的任务。

本章我们将涵盖以下主要主题：

+   您可以编写的四种 RPC 端点类型

+   何时使用每个端点

+   如何实现端点的逻辑

+   如何消费 gRPC 端点

# 技术要求

您可以在本书的配套仓库中找到本章的代码，该仓库位于[`github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter5`](https://github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter5)。

# 使用模板

注意

如果您想与 github 仓库相同的架构一起工作，本节是必要的。继续从上一章的代码工作是完全可行的。

如果您还记得，上一章的目标是创建一个模板，我们可以用它来创建一个新的 gRPC 项目。既然我们现在要开始这样一个项目，我们需要将`chapter4`文件夹的内容复制到`chapter5`文件夹中。为此，只需运行以下命令：

```go
$ mkdir chapter5
$ cp -R chapter4/* chapter5
```

在我们的情况下，我们传递`–R`是因为我们想要递归地复制`chapter4`文件夹中的所有文件。

最后，我们还可以稍微清理一下模板。我们可以在`proto`文件夹中删除虚拟目录，因为这个目录只是为了测试我们的代码生成。然后，如果你为`chapter4`使用了`bazel`，我们可以删除所有以`bazel-`前缀开始的构建文件夹。为此，我们只需简单地按照以下方式删除它们：

```go
$ cd chapter5
$ rm -rf proto/dummy
$ rm -rf bazel-*
```

我们现在可以开始使用这个模板，并查看我们可以使用 gRPC 编写的不同 API 端点。

# 一元 API

重要注意事项

在底层协议方面，正如我们在*第一章*中提到的，*网络基础*，一元 API 使用客户端的`Send Header`、`Send Message`和`Half-Close`，以及服务器端的`Send Message`和`Send Trailer`。如果您需要对这些操作进行复习，我建议您快速查看本书*第一章*中的*RPC 操作*部分。这将帮助您在调用此 API 端点时了解正在发生的事情。

你可以编写的最简单且最熟悉的 API 端点是单参数端点。这些大致对应于`GET`、`POST`以及其他你可能已经在 REST API 中使用过的 HTTP 动词。你发送一个请求，然后得到一个响应。通常，这些端点将是你最常使用的，用于表示一个资源的处理。例如，如果你编写一个登录方法，你只需要发送`LoginRequest`并接收`LoginResponse`。

对于本节，我们将编写一个名为`AddTask`的 RPC 端点。正如其名称所暗示的，这个端点将在列表中创建一个新的任务。因此，在能够做到这一点之前，我们需要定义什么是*任务*。

任务是一个包含以下属性的对象：

+   `id`：一个标识该任务的数字

+   `description`：实际要完成的任务以及用户所阅读的内容

+   `done`：任务是否已经完成

+   `due_date`：当这个任务到期时

如果我们将这些翻译成 Protobuf 代码，我们可能会有以下内容（位于`chapter5/proto/todo/v1`目录下的`todo.proto`文件）：

```go
syntax = "proto3";
package todo.v1;
import "google/protobuf/timestamp.proto";
option go_package = "github.com/PacktPublishing/
  gRPC-Go-for-Professionals/proto/todo/v1";
message Task {
  uint64 id = 1;
  string description = 2;
  bool done = 3;
  google.protobuf.Timestamp due_date = 4;
}
```

注意这里使用了`Timestamp`类型。这是一个在`google.protobuf`包下提供的知名类型。我们使用这个类型来表示任务应该在未来的某个时间点完成。我们可以重写我们自己的`Date`类型或使用在`googleapis`存储库中定义的`Date`类型（[`github.com/googleapis/googleapis/blob/master/google/type/date.proto`](https://github.com/googleapis/googleapis/blob/master/google/type/date.proto)），但`Timestamp`对于这个 API 来说已经足够了。

现在我们有了任务，我们可以考虑我们的 RPC 端点。我们希望我们的端点能够接收一个任务，该任务应被插入到列表中，并返回该任务的标识符给客户端：

```go
message AddTaskRequest {
  string description = 1;
  google.protobuf.Timestamp due_date = 2;
}
message AddTaskResponse {
  uint64 id = 1;
}
service TodoService {
  rpc AddTask(AddTaskRequest) returns (AddTaskResponse);
}
```

在这里需要注意的一个重要事项是，我们不是使用`Task`消息作为`AddTask`端点的参数，而是在端点所需的信息周围创建了一个包装器（`AddTaskRequest`）。不多也不少。我们本来可以直接使用消息，但这可能会导致客户端在网络上发送不必要的额外数据（例如，设置 ID，这可能会被服务器忽略）。此外，对于我们 API 的将来版本，我们可以在`AddTaskRequest`中添加更多字段，而不会影响`Task`消息。我们有效地解耦了请求/响应的实际数据表示。

## 代码生成

重要提示

如果你现在还不确定如何从 proto 文件生成代码，我强烈建议你查看*第四章*，在那里我们介绍了三种生成代码的方法。在本章中，我们将手动生成一切，但你可以在 GitHub 存储库的`chapter5`目录中找到如何使用 Buf 和 Bazel 完成相同操作的方法。

现在我们已经为我们的 API 端点创建了接口，我们希望能够实现其背后的逻辑。为此，第一步是生成一些 Go 代码。为了做到这一点，我们将使用`protoc`和`paths`选项的`source_relative`值。所以，知道我们的`todo.proto`文件位于`proto/todo/v1/`下，我们可以运行以下命令：

```go
$ protoc --go_out=. \
         --go_opt=paths=source_relative \
         --go-grpc_out=. \
         --go-grpc_opt=paths=source_relative \
           proto/todo/v1/*.proto
```

运行之后，你应该有一个`proto/todo/v1/`目录，如下所示：

```go
proto/todo/v1/
├── todo.pb.go
├── todo.proto
└── todo_grpc.pb.go
```

这就是我们开始所需的所有内容。

## 检查生成的代码

在生成的代码中，我们有两个文件——Protobuf 生成的代码和 gRPC 代码。Protobuf 生成的代码在一个名为`todo.pb.go`的文件中。如果我们检查这个文件，我们可以看到的最重要的事情是以下代码（这是简化的）：

```go
type Task struct {
  Id uint64
  Description string
  Done bool
  DueDate *timestamppb.Timestamp
}
type AddTaskRequest struct {
  Description string
  DueDate *timestamppb.Timestamp
}
type AddTaskResponse struct {
  Id uint64
}
```

这意味着我们现在可以在 Go 代码中创建`Task`、`TaskRequest`和`TaskResponse`实例，这正是我们在本章后面将要做的。

对于 gRPC 生成的代码（`todo_grpc.pb.go`），为客户端和服务器都生成了接口。它们应该看起来像以下这样：

```go
type TodoServiceClient interface {
  AddTask(ctx context.Context, in *AddTaskRequest, opts
    ...grpc.CallOption) (*AddTaskResponse, error)
}
type TodoServiceServer interface {
  AddTask(context.Context, *AddTaskRequest)
    (*AddTaskResponse, error)
  mustEmbedUnimplementedTodoServiceServer()
}
```

它们看起来很相似，但服务器端的`AddTask`是我们唯一需要实现逻辑的。客户端的`AddTask`基本上是生成调用我们服务器上的 API 端点的请求，并返回它接收到的响应。

## 注册服务

为了让 gRPC 知道如何处理某个请求，我们需要注册服务的实现。为了注册这样的服务，我们将调用一个生成的函数，该函数存在于`todo_grpc.pb.go`中。在我们的例子中，这个函数叫做`RegisterTodoServiceServer`，其函数签名如下：

```go
func RegisterTodoServiceServer(s grpc.ServiceRegistrar, srv
  TodoServiceServer)
```

它接受`grpc.ServiceRegistrar`接口，这是`grpc.Server`实现的接口，以及`TodoServiceServer`接口，这是我们之前看到的接口。这个函数将把框架提供的通用 gRPC 服务器与我们的端点实现链接起来，以便框架知道如何处理请求。

因此，首先要做的是创建我们的服务器。我们首先将创建一个嵌入`UnimplementedTodoServiceServer`的结构体，这是一个生成的结构体，其中包含端点的默认实现。在我们的例子中，默认实现如下：

```go
func (UnimplementedTodoServiceServer) AddTask
  (context.Context, *AddTaskRequest) (*AddTaskResponse,
    error) {
  return nil, status.Errorf(codes.Unimplemented, "method
    AddTask not implemented")
}
```

如果我们没有在我们的服务器中实现`AddTask`，这个端点将被调用，并且每次调用它时都会返回一个错误。目前这看起来似乎没有用，因为这除了返回错误之外什么也不做，但事实是这是一个安全网，我们将在讨论 API 演变时看到原因。

接下来，我们的服务器将包含对我们数据库的引用。你可以根据你熟悉的任何数据库进行适配，但在这个例子中，我们将使用接口来抽象数据库，因为这将让我们专注于 gRPC，而不是其他技术，例如 MongoDB。

因此，我们的服务器类型（`server/server.go`）将看起来像这样：

```go
package main
import (
  pb "github.com/PacktPublishing/gRPC-Go-for-Professionals/proto/    todo/v1"
)
type server struct {
  d db
  pb.UnimplementedTodoServiceServer
}
```

现在，让我们看看 `db` 接口的样子。我们首先将有一个名为 `addTask` 的函数，它接受一个描述和一个 `dueDate` 值，并返回创建的任务的 `id` 值或错误。现在，需要注意的是，这个数据库接口应该与生成代码解耦。这再次是因为我们 API 的演变，因为如果我们改变端点或 `Request`/`Response` 对象，我们就必须更改我们的接口和所有实现。在这里，接口与生成代码独立。在 `server/db.go` 中，我们现在可以编写以下内容：

```go
package main
import "time"
type db interface {
  addTask(description string, dueDate time.Time) (uint64,
    error)
}
```

这个接口将允许我们对伪造的数据库实现进行测试，并为非发布环境实现内存数据库。

最后一步是实现内存数据库。我们将有一个常规数组 `Task` 来存储我们的待办事项，`addTask` 将简单地向该数组追加一个任务并返回当前任务的 ID。在名为 `server/in_memory.go` 的文件中，我们可以添加以下内容：

```go
package main
import (
  "time"
  pb "github.com/PacktPublishing/gRPC-Go-for-Professionals/
    proto/todo/v1"
  "google.golang.org/protobuf/types/known/timestamppb"
)
type inMemoryDb struct {
  tasks []*pb.Task
}
func New() db {
  return &inMemoryDb{}
}
func (d *inMemoryDb) addTask(description string, dueDate
  time.Time) (uint64, error) {
  nextId := uint64(len(d.tasks) + 1)
  task := &pb.Task{
    Id: nextId,
    Description: description,
    DueDate: timestamppb.New(dueDate),
  }

  d.tasks = append(d.tasks, task)
  return nextId, nil
}
```

关于这个实现，有几个需要注意的事项。首先，这可能是显而易见的，但这不是一个最优的“数据库”，并且仅用于开发目的。其次，不深入细节，我们可以使用 Golang 构建标签在编译时选择我们想要运行的数据库。例如，如果我们有 `inMemoryDb` 和 `mongoDb` 实现，我们可以在每个文件的顶部添加 `go:build` 标签。对于 `in_memory.go`，我们可以有如下内容：

```go
//go:build in_memory_db
//...
type inMemoryDb struct
func New() db
```

对于 `mongodb.go`，我们可以有如下内容：

```go
//go:build mongodb
//...
type mongoDb struct
func New() db
```

这将允许我们在编译时选择我们想要使用的 `New` 函数，从而创建 `inMemoryDb` 或 `mongoDb` 实例。

最后，你可能已经注意到，我们在实现这个“数据库”时使用了生成的代码。由于这是一个作为开发环境使用的“数据库”，这个实现是否与我们的生成代码耦合并不重要。最重要的是，不要将 `db` 接口与其耦合，这样你就可以使用任何数据库，甚至无需处理生成代码。

现在，我们终于准备好注册我们的服务器类型了。为此，我们只需进入我们的 `server/main.go main` 函数并添加以下行：

```go
import pb "github.com/PacktPublishing/gRPC-Go-for-Professionals/
    proto/todo/v1"
//...
s := grpc.NewServer(opts...)
pb.RegisterTodoServiceServer(s, &server{
  d: New(),
})
defer s.Stop()
```

这意味着我们现在已经将名为 `s` 的 gRPC 服务器链接到了创建的服务器实例。请注意，这里的 `New` 函数是我们定义在 `in_memory.go` 文件中的函数。

## 实现 AddTask

对于实现，我们将创建一个名为 `server/impl.go` 的文件，该文件将包含所有端点的实现。请注意，这纯粹是为了方便，你也可以为每个 RPC 端点创建一个文件。

现在你可能还记得，我们服务器的生成接口要求我们实现以下函数：

```go
AddTask(context.Context, *AddTaskRequest)
  (*AddTaskResponse, error)
```

因此，我们只需将这个函数添加到我们的服务器类型中，通过编写以服务器实例名称和服务器类型命名的函数前缀即可：

```go
func (s *server) AddTask(_ context.Context, in
  *pb.AddTaskRequest) (*pb.AddTaskResponse, error) {
}
```

最后，我们可以实现这个函数。这将调用我们的 `db` 接口中的 `addTask` 函数，由于目前这个函数从不返回错误（现在是这样），我们将获取给定的 ID 并将其作为 `AddTaskResponse` 返回：

```go
package main
import (
  "context"
  pb "github.com/PacktPublishing/gRPC-Go-for-Professionals/proto/    todo/v1"
)
func (s *server) AddTask(_ context.Context, in
  *pb.AddTaskRequest) (*pb.AddTaskResponse, error) {
  id, _ := s.d.addTask(in.Description, in.DueDate.AsTime())
  return &pb.AddTaskResponse{Id: id}, nil
}
```

注意，`AsTime` 是由 `google.golang.org/protobuf/types/known/timestamppb` 包提供的一个函数，它返回一个 Golang `time.Time` 对象。`timestamppb` 包是一组函数，允许我们操作 `google.protobuf.Timestamp` 对象，并在我们的 Go 代码中以惯用的方式使用它。

目前，你可能觉得这太简单了，但请记住，我们刚刚开始我们的 API。在本书的后面部分，我们将进行错误处理，并了解如何拒绝不正确的参数。

## 从客户端调用 AddTask

最后，让我们看看如何从 Go 客户端代码中调用端点。这很简单，因为我们已经在上一章中创建了样板代码。

我们将创建一个名为 `AddTask` 的函数，该函数将调用我们在服务器上注册的 API 端点。为此，我们需要传递一个 `TodoServiceClient` 实例、任务的描述和截止日期。我们稍后会创建客户端实例，但请注意 `TodoServiceClient` 是我们在检查生成的代码时看到的接口。在 `client/main.go` 中，我们可以添加以下内容：

```go
import  (
  //...
  google.golang.org/protobuf/types/known/timestamppb
  pb "github.com/PacktPublishing/gRPC-Go-for-Professionals/
  proto/todo/v1"
  //...
)
func addTask(c pb.TodoServiceClient, description string,
  dueDate time.Time) uint64 {
}
```

之后，使用参数，我们只需构建一个新的 `AddTaskRequest` 实例并将其发送到服务器。

```go
func addTask(c pb.TodoServiceClient, description string,
  dueDate time.Time) uint64 {
  req := &pb.AddTaskRequest{
    Description: description,
    DueDate: timestamppb.New(dueDate),
  }
  res, err := c.AddTask(context.Background(), req)
  //...
}
```

最后，我们将从我们的 API 调用中接收到 `AddTaskResponse` 或错误。如果有错误，我们将在屏幕上记录它，如果没有错误，我们记录并返回 ID：

```go
func addTask(c pb.TodoServiceClient, description string,
  dueDate time.Time) uint64 {
  //...
  if err != nil {
    panic(err)
  }
  fmt.Printf("added task: %d\n", res.Id)
  return res.Id
}
```

要调用此函数，我们需要使用一个名为 `NewTodoServiceClient` 的生成函数，我们将一个连接传递给它，它返回一个新的 `TodoServiceClient` 实例。然后，我们只需简单地将以下几行代码添加到客户端的 `main` 函数末尾：

```go
conn, err := grpc.Dial(addr, opts...)
if err != nil {
  log.Fatalf("did not connect: %v", err)
}
c := pb.NewTodoServiceClient(conn)
fmt.Println("--------ADD--------")
dueDate := time.Now().Add(5 * time.Second)
addTask(c, "This is a task", dueDate)
fmt.Println("-------------------")
defer func(conn *grpc.ClientConn) {
  /*...*/}(conn)
```

注意，这里我们正在添加一个五秒截止日期和描述为 `This is a task` 的任务。这只是一个示例，我鼓励你自己尝试使用不同的值进行更多调用。

现在，我们基本上可以运行我们的服务器和客户端，看看它们是如何交互的。要运行服务器，使用这个 `go` `run` 命令：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，在另一个终端中，我们可以以类似的方式运行客户端：

```go
$ go run ./client 0.0.0.0:50051
--------ADD--------
added task: 1
-------------------
```

最后，要停止服务器，你只需在运行它的终端中按 *Ctrl + C* 即可。

因此，我们可以看到我们已经在服务器上注册了一个服务实现，我们的客户端正在正确地发送请求并返回响应。我们所做的是一个简单的 Unary API 端点示例。

## Bazel

重要注意事项

在本节中，我们将看到如何使用 Bazel 运行应用程序。然而，由于如果每个部分都有这样的说明就会变得重复，我想提醒您，我们不会每次都走这些步骤。对于每个部分，您都可以运行下面的`bazel run`命令（对于服务器和客户端），而`gazelle`命令仅适用于本节。

到目前为止，您的 Bazel BUILD 文件可能已经过时。为了同步它们，我们可以简单地运行`gazelle`命令，它将更新所有依赖项和文件以进行编译：

```go
$ bazel run //:gazelle
```

之后，我们可以通过运行以下命令轻松运行服务器和客户端来执行服务器：

```go
$ bazel run //server:server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

同时，使用以下命令运行客户端：

```go
$ bazel run //client:client 0.0.0.0:50051
--------ADD--------
added task: 1
-------------------
```

这是我们的服务器和客户端工作正常的一个另一个例子。我们可以使用`go run`和`bazel run`命令来运行它们。我们现在对 Unary API 有信心；让我们转向服务器流式 API。

# 服务器流式 API

重要提示

在底层协议方面，服务器流式 API 使用客户端的`Send Header`、`Send Message`和`Half-Close`，以及服务器端的多个`Send Message`和`Send Trailer`。

现在，我们知道如何注册服务、与“数据库”交互以及运行客户端和服务器后，一切都会更快。我们将主要关注 API 端点本身。在我们的例子中，我们将创建一个`ListTasks`端点，正如其名称所暗示的，它将列出数据库中所有可用的任务。

我们将要做的其中一件事是，对于每个列出的任务，我们将返回该任务是否已过期。这样做主要是为了让您了解如何在`response`对象中提供有关某个对象的更多信息。

因此，在`todo.proto`文件中，我们将添加一个名为`ListTasks`的 RPC 端点，它将接受`ListTasksRequest`并返回一个`ListTasksResponse`流。这就是服务器流式 API。我们接收一个请求并返回零个或多个响应：

```go
message ListTasksRequest {
}
message ListTasksResponse {
  Task task = 1;
  bool overdue = 2;
}
service TodoService {
  //...
  rpc ListTasks(ListTasksRequest) returns (stream ListTasksResponse);
}
```

注意，这次我们发送一个空对象作为请求，并获取多个`Task`及其是否过期。我们可以通过发送我们想要列出的任务 ID 范围（分页）来使请求更加智能，但为了简洁起见，我们选择使其简单。

## 数据库的演变

在能够实现`ListTasks`端点之前，我们需要一种方法来访问我们的 TODO 列表中的所有元素。再次强调，我们不希望将`db`接口绑定到我们的生成代码，因此我们有几种选择：

+   我们创建了一些抽象来遍历任务。这对于我们的内存数据库可能很好，但例如与 Postgres 一起使用时会如何呢？

+   我们将我们的接口绑定到一个已经存在的抽象，例如数据库的游标。这稍微好一些，但我们仍然耦合了我们的接口。

+   我们只是将迭代留给我们的`db`接口实现，并将一个用户提供的函数应用于所有行。这样，我们就不会耦合到任何其他组件。

因此，我们将迭代留给我们的接口实现。这意味着我们将要添加到`inMemoryDb`的新函数将遍历所有任务并将一个作为参数提供的函数应用于每一个：

```go
type db interface {
  //...
  getTasks(f func(interface{}) error) error
}
```

如你所见，作为参数传递的函数本身将获得一个`interface{}`作为参数。这并不是类型安全的；然而，我们将确保在稍后处理时运行时接收一个`Task`。

然后，对于内存实现，我们有以下内容：

```go
func (d *inMemoryDb) getTasks(f func(interface{}) error)
  error {
  for _, task := range d.tasks {
    if err := f(task); err != nil {
      return err
    }
  }
  return nil
}
```

这里有一个需要注意的地方。我们将使错误对进程致命。如果用户提供的函数返回一个错误，`getTasks`函数将返回一个错误。

那就是数据库的全部内容；我们现在可以从它获取所有任务并对任务应用某种逻辑。让我们实现`ListTasks`。

## 实现 ListTasks

为了实现端点，让我们从 proto 文件生成 Go 代码，并取我们需要实现的函数签名。我们只需运行以下命令：

```go
$ protoc --go_out=. \
         --go_opt=paths=source_relative \
         --go-grpc_out=. \
         --go-grpc_opt=paths=source_relative \
         proto/todo/v1/*.proto
```

如果我们查看`proto/todo/v1/todo_grpc.pb.go`，我们现在可以看到`TodoServiceServer`接口有一个额外的函数：

```go
type TodoServiceServer interface {
  //...
  ListTasks(*ListTasksRequest, TodoService_ListTasksServer)
    error
  //...
}
```

如我们所见，签名与从`AddTask`函数得到的签名略有不同。我们现在只返回一个错误或`nil`，并且我们将`TodoService_ListTasksServer`作为一个参数。

如果你深入研究生成的代码，你会看到`TodoService_ListTasksServer`被定义为以下内容：

```go
type TodoService_ListTasksServer interface {
  Send(*ListTasksResponse) error
  grpc.ServerStream
}
```

这是一个可以发送`ListTasksResponse`对象的流。

现在我们知道了这一点，让我们在我们的代码中实现这个函数。我们可以进入服务器下的`impl.go`文件，并将`ListTasks`函数的签名复制粘贴到`TodoServiceServer`中：

```go
func (s *server) ListTasks(req *pb.ListTasksRequest, stream
  pb.TodoService_ListTasksServer) error
```

显然，我们添加了服务器类型来指定我们正在为我们的服务器实现`ListTasks`，并且我们命名了参数。`req`是我们从客户端接收的请求，`stream`是我们将用来发送多个答案的对象。

然后，我们函数的逻辑再次简单明了。我们将遍历所有任务，确保我们处理的是`Task`对象，并且对于这些任务中的每一个，我们将“计算”逾期情况，通过检查这些任务是否完成（对于完成的任务没有逾期）以及`due_date`字段是否在当前时间之前。总之，我们将只创建包含这些信息的`ListTasksResponse`并发送给客户端（`server/impl.go`）：

```go
func (s *server) ListTasks(req *pb.ListTasksRequest, stream
  pb.TodoService_ListTasksServer) error {
  return s.d.getTasks(func(t interface{}) error {
    task := t.(*pb.Task)
    overdue := task.DueDate != nil && !task.Done &&
      task.DueDate.AsTime().Before(time.Now().UTC())
    err := stream.Send(&pb.ListTasksResponse{
      Task: task,
      Overdue: overdue,
    })
    return err
  })
}
```

有一个需要注意的地方是，`AsTime`函数将创建一个 UTC 时区的时间，所以当你与它比较时间时，也需要它也在 UTC 时区。这就是为什么我们有`time.Now().UTC()`而不是简单地`time.Now()`。

显然，这个函数可能会失败（例如，如果名为`t`的变量不是一个`Task`怎么办？）但现阶段，我们不必过于担心错误处理。我们稍后会看到。现在，让我们从客户端调用这个端点。

## 从客户端调用 ListTasks

要从客户端调用`ListTasks` API 端点，我们需要了解如何消费服务器流式 RPC 端点。要做到这一点，我们检查方法签名或接口名称中的生成函数`TodoServiceClient`。它应该看起来像以下这样：

```go
ListTasks(ctx context.Context, in *ListTasksRequest, opts
...grpc.CallOption) (TodoService_ListTasksClient, error)
```

我们可以看到我们需要传递一个上下文和一个请求，还有一些可选的调用选项。然后，我们还可以看到我们将得到一个`TodoService_ListTasksClient`或一个错误。`TodoService_ListTasksClient`类型与我们之前在`ListTasks`端点中处理的流非常相似。不过，主要的区别是，我们不再有一个名为`Send`的函数，我们现在有一个名为`Recv`的函数。以下是`TodoService_ListTasksClient`的定义：

```go
type TodoService_ListTasksClient interface {
  Recv() (*ListTasksResponse, error)
  grpc.ClientStream
}
```

因此，我们将如何处理这个流呢？我们将遍历从`Recv`获取的所有响应，然后某个时刻，服务器会说：“我完成了。”这发生在我们得到一个等于`io.EOF`的错误时。

我们可以在`client/main.go`中创建一个名为`printTasks`的函数，该函数将反复调用`Recv`并检查我们是否完成或有错误，如果没有这种情况，它将在终端上打印我们的`Task`对象的字符串表示：

```go
func printTasks(c pb.TodoServiceClient) {
  req := &pb.ListTasksRequest{}
  stream, err := c.ListTasks(context.Background(), req)
  if err != nil {
    log.Fatalf("unexpected error: %v", err)
  }
  for {
    res, err := stream.Recv()
    if err == io.EOF {
      break
    }
    if err != nil {
      log.Fatalf("unexpected error: %v", err)
    }
    fmt.Println(res.Task.String(), "overdue: ",
       res.Overdue)
  }
}
```

一旦我们有了这个，我们就可以在`main`函数中`addTask`调用之后调用该函数：

```go
fmt.Println("--------ADD--------")
//...
fmt.Println("--------LIST-------")
printTasks(c)
fmt.Println("-------------------")
```

我们现在可以使用`go run`先运行我们的服务器，然后是我们的客户端。所以，在项目的根目录下，我们可以运行以下命令：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们运行我们的客户端：

```go
$ go run ./client 0.0.0.0:50051
//...
--------LIST-------
id:1 description:"This is a task" due_date:
  {seconds:1680158076 nanos:574914000} overdue: false
-------------------
```

这正如预期的那样工作。现在，我鼓励你尝试自己添加更多任务，尝试不同的值，并在添加所有任务后使用`printTasks`函数。这应该有助于你熟悉 API。

现在我们能够添加任务并列出所有任务，如果我们可以更新已经存在的任务那就太好了。这可能对标记任务为完成和更改截止日期很有用。我们将通过客户端流式 API 来测试这一点。

# 客户端流式 API

重要提示

在底层协议方面，客户端流式 API 使用客户端的`Send Header`，然后是多个`Send Message`和一个`Half-Close`，以及服务器端的`Send Message`和`Send Trailer`。

对于客户端流式 API 端点，我们可以发送零个或多个请求并得到一个响应。这是一个重要的概念，尤其是在实时上传数据时。一个例子可能是我们在前端点击一个编辑按钮，这会触发一个编辑会话，并且我们实时地发布每个编辑。显然，由于我们不是在处理这样复杂的前端，我们只会关注使 API 与这种功能兼容。

要定义一个客户端流式 API，我们只需在参数子句中写入 `stream` 关键字而不是 `return`。之前，对于我们的服务器流式，我们有以下内容：

```go
rpc ListTasks(ListTasksRequest) returns (stream ListTasksResponse);
```

现在，我们将有以下的 `UpdateTasks`：

```go
message UpdateTasksRequest {
  Task task = 1;
}
message UpdateTasksResponse {
}
service TodoService {
  //...
  rpc UpdateTasks(stream UpdateTasksRequest) returns
    (UpdateTasksResponse);
}
```

注意，在这种情况下，我们使用的是完整的 Task 消息作为请求，而不是像 `AddTask` 中的那样分开的字段。这并不是一个错误，我们将在第六章中进一步讨论这个问题。

这实际上意味着客户端发送多个请求，服务器返回一个响应。我们现在又向实现端点迈进了一步。然而，在这样做之前，让我们先谈谈数据库。

## 数据库的演变

在考虑实现 `UpdateTasks` 之前，我们需要看看我们如何与数据库交互。首先考虑的是，对于给定的任务，可以更新哪些信息。在我们的案例中，我们不希望客户端能够更新 ID；这是由数据库处理的一个细节。然而，对于所有其他信息，我们希望让用户能够更新它。当任务完成时，我们需要能够将 `done` 设置为 `true`。当 `Task` 描述需要更新时，客户端应该能够在数据库中更改它。最后，当截止日期更改时，客户端也应该能够更新它。

了解这些后，我们可以在数据库中定义 `updateTask` 的函数签名。它将接受任务 ID 和所有可以更改的信息作为参数，并返回一个错误或 `nil`：

```go
type db interface {
  //...
  updateTask(id uint64, description string, dueDate
    time.Time, done bool) error
}
```

再次强调，传递这么多参数看起来有点多，但我们不希望将此接口与任何生成的代码耦合。如果我们以后需要添加更多信息或删除一些，这就像更新此接口和更新实现一样简单。

现在，为了实现这一点，我们将进入 `in_memory.go` 文件。该函数将简单地遍历数据库中的所有任务，如果任何 `Task` 的 ID 与参数中传入的 ID 相同，我们将逐个更新所有字段：

```go
func (d *inMemoryDb) updateTask(id uint64, description
  string, dueDate time.Time, done bool) error {
  for i, task := range d.tasks {
    if task.Id == id {
      t := d.tasks[i]
      t.Description = description
      t.DueDate = timestamppb.New(dueDate)
      t.Done = done
      return nil
    }
  }
  return fmt.Errorf("task with id %d not found", id)
}
```

这意味着每次我们收到请求时，我们都会遍历所有任务。这并不高效，尤其是在数据库变得更大时。然而，我们也不是在处理真实的数据库，所以对于这本书中的用例来说应该足够好了。

## 实现 UpdateTasks

要实现端点，让我们从 proto 文件生成 Go 代码，并获取我们需要实现的函数签名。我们只需运行以下命令：

```go
$ protoc --go_out=. \
         --go_opt=paths=source_relative \
         --go-grpc_out=. \
         --go-grpc_opt=paths=source_relative \
         proto/todo/v1/*.proto
```

如果我们查看 `proto/todo/v1/todo_grpc.pb.go`，我们现在可以看到 `TodoServiceServer` 接口有一个额外的函数：

```go
type TodoServiceServer interface {
  //...
  UpdateTasks(TodoService_UpdateTasksServer) error
  //...
}
```

如我们所见，签名变更类似于 `ListTasks`；然而，这次我们甚至不处理请求。我们只是处理 `TodoService_UpdateTasksServer` 类型的流。如果我们检查 `TodoService_UpdateTasksServer` 类型的定义，我们会看到以下内容：

```go
type TodoService_UpdateTasksServer interface {
  SendAndClose(*UpdateTasksResponse) error
  Recv() (*UpdateTasksRequest, error)
  grpc.ServerStream
}
```

我们已经熟悉了`Recv`函数。它让我们获取一个对象，但现在我们还有一个`SendAndClose`函数。这个函数让我们告诉客户端我们在服务器端已完成。这用于在客户端发送`io.EOF`时关闭流。

带着这些知识，我们可以实现我们的端点。我们将反复在流上调用`Recv`函数，如果我们收到`io.EOF`，我们将使用`SendAndClose`函数；否则，我们将调用我们数据库上的`updateTask`函数：

```go
func (s *server) UpdateTasks(stream pb.TodoService
  _UpdateTasksServer) error {
  for {
    req, err := stream.Recv()
    if err == io.EOF {
      return stream.SendAndClose(&pb.UpdateTasksResponse{})
    }
    if err != nil {
      return err
    }
    s.d.updateTask(
      req.Task.Id,
      req.Task.Description,
      req.Task.DueDate.AsTime(),
      req.Task.Done,
    )
  }
}
```

现在我们应该能够触发这个 API 端点以实时更改一组`Task`。现在让我们看看客户端如何调用端点。

## 从客户端调用 UpdateTasks

这次，因为我们正在处理客户端流，我们将与服务器流相反。客户端将反复调用`Send`，服务器将反复调用`Recv`。最后，客户端将调用定义在生成的代码中的`CloseAndRecv`函数。

如果我们查看客户端的`UpdateTasks`生成的代码，我们将在`TodoServiceClient`类型中看到以下签名：

```go
UpdateTasks(ctx context.Context, opts ...grpc.CallOption)
  (TodoService_UpdateTasksClient, error)
```

注意现在`UpdateTasks`函数不再接受任何`request`参数，但它将返回一个`TodoService_UpdateTasksClient`类型的流。正如提到的，这个类型将包含两个函数：`Send`和`CloseAndRecv`。如果我们查看生成的代码，我们将看到以下内容：

```go
type TodoService_UpdateTasksClient interface {
  Send(*UpdateTasksRequest) error
  CloseAndRecv() (*UpdateTasksResponse, error)
  grpc.ClientStream
}
```

`Send`发送`UpdateTasksRequest`，而`CloseAndRecv`将通知服务器它已完成发送请求，并请求`UpdateTasksResponse`。

现在我们已经理解了，我们可以在`client/main.go`文件中实现`UpdateTasks`函数，我们将从 gRPC 客户端调用`UpdateTasks`函数。这将返回一个流，然后我们将通过它发送给定的任务。一旦我们遍历了所有需要更新的任务，我们将调用`CloseAndRecv`函数：

```go
func updateTasks(c pb.TodoServiceClient, reqs
...*pb.UpdateTasksRequest) {
  stream, err := c.UpdateTasks(context.Background())
  if err != nil {
    log.Fatalf("unexpected error: %v", err)
  }
  for _, req := range reqs {
    err := stream.Send(req)
    if err != nil {
      return
    }
    if err != nil {
      log.Fatalf("unexpected error: %v", err)
    }
    if req.Task != nil {
      fmt.Printf("updated task with id: %d\n", req.Task.Id)
    }
  }
  if _, err = stream.CloseAndRecv(); err != nil {
    log.Fatalf("unexpected error: %v", err)
  }
}
```

现在，正如你在`updateTasks`中看到的，我们需要传递零个或多个`UpdateTasksRequest`作为参数。为了获取创建任务实例并填充`UpdateTasksRequest.task`字段所需的 ID，我们将使用`addTasks`记录之前创建的任务的 ID。因此，以前在`main`中我们有以下内容：

```go
addTask(c, "This is a task", dueDate)
```

我们现在将得到以下类似的内容：

```go
id1 := addTask(c, "This is a task", dueDate)
id2 := addTask(c, "This is another task", dueDate)
id3 := addTask(c, "And yet another task", dueDate)
```

现在，我们可以创建一个`UpdateTasksRequest`数组，就像这样：

```go
[]*pb.UpdateTasksRequest{
  {Task: &pb.Task{Id: id1, Description: "A better name for
    the task"}},
  {Task: &pb.Task{Id: id2, DueDate: timestamppb.New
    (dueDate.Add(5 * time.Hour))}},
  {Task: &pb.Task{Id: id3, Done: true}},
}
```

这意味着具有`id1`的`Task`对象将被更新为具有新的描述，具有`id2`的`Task`对象将被更新为具有新的`due_date`值，最后，最后一个将被标记为`done`。

我们现在可以将客户端传递给`updateTasks`，并通过使用`…`操作符将这个数组作为可变参数展开。在`main`函数中，我们现在可以添加以下内容：

```go
fmt.Println("-------UPDATE------")
updateTasks(c, []*pb.UpdateTasksRequest{
  {Task: &pb.Task{Id: id1, Description: "A better name for
    the task"}},
  {Task: &pb.Task{Id: id2, DueDate: timestamppb.New
    (dueDate.Add(5 * time.Hour))}},
  {Task: &pb.Task{Id: id3, Done: true}},
}...)
printTasks(c)
fmt.Println("-------------------")
```

我们现在可以以类似前几节的方式运行它。我们使用`go run`首先运行服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后我们运行客户端来调用 API 端点：

```go
$ go run ./client 0.0.0.0:50051
//...
-------UPDATE------
updated task with id: 1
updated task with id: 2
updated task with id: 3
id:1  description:"A better name for the task"  due_date:{}
id:2  due_date:{seconds:1680267768  nanos:127075000}
id:3  done:true  due_date:{}
-------------------
```

在继续之前，这里有一个重要的事情需要注意。你可能会对任务在更新时丢失一些信息的事实感到惊讶。这是因为 Protobuf 会在字段未设置时使用默认值——这意味着如果客户端发送一个只有`done`等于`true`的`Task`对象，描述将被反序列化为空字符串，而`due_date`将是一个空的`google.protobuf.Timestamp`。目前，这非常低效，因为我们需要重新发送所有信息来更新单个字段。在本书的后面，我们将讨论如何解决这个问题。现在，我们可以根据它们的 ID 实时更新多个任务。现在，让我们转向最后一种可用的 API 类型：双向流。

# 双向流式 API

重要注意事项

就底层协议而言，双向流式 API 使用客户端的`Send Header`，然后是服务器和/或客户端的多个`Send Message`，客户端的`Half-Close`，最后是服务器的`Send Trailer`。

在双向流式 API 中，目标是让客户端发送零个或多个请求，并让服务器发送零个或多个响应。我们将使用这一点来模拟一个类似于`updateTasks`的功能，但在其中，我们将在每次删除后直接从`DeleteTasks`端点 API 获得反馈，而不是等待所有删除操作完成。

在继续实施之前，需要明确的一点是关于为什么不将`DeleteTasks`设计为服务器流式 API 或`updateTasks`设计为双向流式 API 的问题。这两个任务之间的区别在于它们的“破坏性”程度。我们可以在发送请求到服务器之前直接在客户端进行更新。如果存在任何错误，我们只需查看客户端上的列表，并根据修改时间将其与服务器上的列表同步。对于删除操作，这会稍微复杂一些。我们可以在客户端保留已删除的行，稍后由垃圾回收器回收，或者我们需要在其他地方存储信息以与服务器同步。这会带来更多的开销。

因此，我们将发送多个`DeleteTasksRequest`，并且对于每一个，我们将得到确认它已被删除。如果发生错误，我们仍然可以确信错误之前的任务已在服务器上删除。我们的 RPC 和消息如下所示：

```go
message DeleteTasksRequest {
  uint64 id = 1;
}
message DeleteTasksResponse {
}
service TodoService {
  //...
  rpc DeleteTasks(stream DeleteTasksRequest) returns
    (stream DeleteTasksResponse);
}
```

我们反复发送我们想要删除的`Task`的 ID，如果我们收到`DeleteTasksResponse`，这意味着任务已被删除。否则，我们会得到一个错误。

现在，在深入实施之前，让我们看看我们的数据库接口。

## 数据库的演变

因为我们想要在数据库中删除一个 `Task` 对象，我们将需要一个 `deleteTask` 函数。这个函数将接受要删除的 `Task` 对象的 ID，对其执行操作，并返回一个错误或 nil。我们可以在 `server/db.go` 中添加以下函数：

```go
type db interface {
  //...
  deleteTask(id uint64) error
}
```

实现起来很像 `updateTask`。然而，当我们找到具有正确 ID 的任务时，我们不会更新信息，而是使用 `go slice` 技巧来删除它。在 `server/in_memory.go` 中，我们现在有以下内容：

```go
func (d *inMemoryDb) deleteTask(id uint64) error {
  for i, task := range d.tasks {
    if task.Id == id {
      d.tasks = append(d.tasks[:i], d.tasks[i+1:]...)
      return nil
    }
  }
  return fmt.Errorf("task with id %d not found", id)
}
```

切片技巧将当前任务之后的元素追加到前一个任务中。这实际上覆盖了数组中的当前任务，从而删除了它。有了这个，我们现在可以准备实现 `DeleteTasks` 端点。

## 实现删除任务

在实现实际端点之前，我们需要了解我们将如何处理生成的代码。因此，让我们使用以下命令生成代码：

```go
$ protoc --go_out=. \
         --go_opt=paths=source_relative \
         --go-grpc_out=. \
         --go-grpc_opt=paths=source_relative \
         proto/todo/v1/*.proto
```

如果我们检查 `proto/todo/v1` 文件夹中的 `todo_grpc.pb.go`，我们应该在 `TodoServiceServer` 中添加以下函数：

```go
DeleteTasks(TodoService_DeleteTasksServer) error
```

这与 `UpdateTasks` 函数类似，因为我们得到一个流，我们返回一个错误或 nil。然而，我们没有 `Send` 和 `SendAndClose` 函数，我们现在有 `Send` 和 `Recv`。`TodoService_DeleteTasksServer` 定义如下：

```go
type TodoService_DeleteTasksServer interface {
  Send(*DeleteTasksResponse) error
  Recv() (*DeleteTasksRequest, error)
  grpc.ServerStream
}
```

这意味着在我们的情况下，我们可以调用 `Recv` 来获取 `DeleteTasksRequest`，然后为每个请求发送一个 `DeleteTasksResponse`。最后，因为我们正在处理流，我们仍然需要检查错误和 `io.EOF`。当我们得到 `io.EOF` 时，我们只需使用 `return` 结束函数：

```go
func (s *server) DeleteTasks(stream
  pb.TodoService_DeleteTasksServer) error {
  for {
    req, err := stream.Recv()
    if err == io.EOF {
      return nil
    }
    if err != nil {
      return err
    }
    s.d.deleteTask(req.Id)
    stream.Send(&pb.DeleteTasksResponse{})
  }
}
```

这里需要注意的一点是 `stream.Send` 调用。尽管这很简单，但这区分了客户端流和双向流。如果我们没有这个调用，我们实际上会从客户端发送多个请求，最终服务器会返回 `nil` 来关闭流。这将与 `UpdateTasks` 完全相同。但因为 `Send` 调用，我们现在在每次删除后都有直接的反馈。

## 从客户端调用 UpdateTasks

现在我们有了我们的端点，我们可以从客户端调用它。在我们这样做之前，我们需要查看 `TodoServiceClient` 生成的代码。我们现在应该有以下函数：

```go
DeleteTasks(ctx context.Context, opts ...grpc.CallOption)
  (TodoService_DeleteTasksClient, error)
```

再次强调，这与我们在 `ListTasks` 和 `UpdateTasks` 函数中看到的内容类似，因为它返回一个我们可以与之交互的字符串。然而，正如你所猜到的，我们现在可以使用 `Send` 和 `Recv`。`TodoService_DeleteTasksClient` 看起来是这样的：

```go
type TodoService_DeleteTasksClient interface {
  Send(*DeleteTasksRequest) error
  Recv() (*DeleteTasksResponse, error)
  grpc.ClientStream
}
```

使用这个生成的代码和底层的 gRPC 框架，我们现在可以发送多个 `DeleteTasksRequest` 并获取多个 `DeleteTasksResponse`。

现在，我们将在 `client/main.go` 中创建一个新的函数，该函数将接受 `DeleteTasksRequest` 的可变参数。然后，我们将创建一个通道，它将帮助我们等待接收和发送的整个过程的完成。如果我们没有这样做，我们会在完成之前从函数中返回。这个通道将在一个 goroutine 中使用，该 goroutine 将在后台使用 `Recv`。一旦在这个 goroutine 中接收到 `io.EOF`，我们将关闭这个通道。最后，我们将遍历所有请求并发送它们，一旦完成，我们将等待通道关闭。

这可能现在看起来有点抽象，但想想客户需要完成的任务。它需要同时使用 `Recv` 和 `Send`；因此，我们需要一些简单的并发代码：

```go
func deleteTasks(c pb.TodoServiceClient, reqs
...*pb.DeleteTasksRequest) {
  stream, err := c.DeleteTasks(context.Background())
  if err != nil {
    log.Fatalf("unexpected error: %v", err)
  }
  waitc := make(chan struct{})
  go func() {
    for {
      _, err := stream.Recv()
      if err == io.EOF {
        close(waitc)
        break
      }
      if err != nil {
        log.Fatalf("error while receiving: %v\n", err)
      }
      log.Println("deleted tasks")
    }
  }()
  for _, req := range reqs {
    if err := stream.Send(req); err != nil {
      return
    }
  }
  if err := stream.CloseSend(); err != nil {
    return
  }
  <-waitc
}
```

最后，在运行服务器和客户端之前，让我们在 `main` 函数中调用该函数。我们将删除使用 `addTasks` 创建的所有任务，并通过尝试打印所有任务来证明没有更多任务：

```go
fmt.Println("-------DELETE------")
deleteTasks(c, []*pb.DeleteTasksRequest{
  {Id: id1},
  {Id: id2},
  {Id: id3},
}...)
printTasks(c)
fmt.Println("-------------------")
```

通过那样，我们可以先运行服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后我们可以运行我们的客户端：

```go
$ go run ./client 0.0.0.0:50051
//...
-------DELETE------
2023/03/31 18:54:21 deleted tasks
2023/03/31 18:54:21 deleted tasks
2023/03/31 18:54:21 deleted tasks
-------------------
```

注意这里，与客户端流只有一个响应不同，我们有三个响应（三个已删除的任务）。这是因为我们对每个请求都得到一个响应。我们实际上实现了双向流。

我们在这里实现了双向流，这使得我们可以获取我们发送给服务器的每个请求的反馈。有了这个，我们可以确保在不需要等待服务器响应或错误的情况下更新客户端的资源。这对于需要实时更新的用例来说很有趣。

# 摘要

在本章中，我们看到了我们可以编写的不同类型的 gRPC API。我们看到了我们可以创建与我们在 REST API 中熟悉的类似 API 端点。这些端点被称为单一端点。然后，我们看到了我们可以创建服务器流 API，让服务器返回多个响应。同样，我们看到了客户端可以通过客户端流返回多个请求。最后，我们看到了我们可以“混合”服务器和客户端流以获得双向流。

我们当前的端点很简单，没有处理对生产级 API 至关重要的许多情况。

在下一章中，我们将开始看到我们可以在 API 层面上进行哪些改进。这将让我们首先关注 API 的可用性，然后再深入到生产级 API 的所有方面。

# 问答

1.  当你想发送一个请求并接收一个响应时，你应该使用哪种类型的 API 端点？

    1.  双向流

    1.  客户端流

    1.  单一

1.  当你想发送零个或多个请求并接收一个响应时，你应该使用哪种类型的 API 端点？

    1.  服务器流

    1.  双向流

    1.  客户端流

1.  当你想发送一个请求并接收零个或多个响应时，你应该使用哪种类型的 API 端点？

    1.  服务器流

    1.  客户端流

    1.  双向流

1.  当你想发送零个或多个请求并接收零个或多个响应时，应该使用哪种 API 端点？

    1.  客户端流

    1.  双向流

    1.  服务器流

# 答案

1.  C

1.  C

1.  A

1.  B
