

# 第七章：开箱即用的功能

由于编写生产就绪的 API 比发送请求和接收响应要复杂得多，因此 gRPC 比我们看到的简单通信模式提供了更多功能。在本章中，我们将看到我们可以使用的一些最重要的功能，以便使我们的 API 更加健壮、高效和安全。

在本章中，我们将涵盖以下主题：

+   处理错误、取消和截止日期

+   发送 HTTP 头

+   在线加密数据

+   使用拦截器提供额外的逻辑

+   调度到不同服务器的请求

到本章结束时，我们将了解到使用 gRPC 时直接提供的最重要的功能。

# 技术要求

对于本章，你将在附带的 GitHub 仓库中找到名为`chapter7`的文件夹中的相关代码([`github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter7`](https://github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter7))。

在上一节中，我将使用 Kubernetes 来展示客户端负载均衡。我假设你已经安装了 Docker 并且有一个 Kubernetes 集群。你可以以任何你想要的方式完成这件事，但我提供了一个 Kind ([`kind.sigs.k8s.io/`](https://kind.sigs.k8s.io/)) 配置，以便轻松且本地地启动一个集群。这个配置位于`chapter7`的`k8s`文件夹中，在名为`kind.yaml`的文件中。一旦安装了 Kind，你可以这样使用它：

```go
$ kind create cluster --config k8s/kind.yaml
```

你可以通过运行以下命令来丢弃它：

```go
$ kind delete cluster
```

# 处理错误

到目前为止，我们还没有讨论可能出现在业务逻辑内部或外部的潜在错误。这对于一个生产就绪的 API 来说显然不是很好，因此我们将看到如何解决这些问题。在本节中，我们将集中精力解决名为`AddTask`的 RPC 端点。

在开始编码之前，我们需要了解 gRPC 中错误的工作方式，但这不应该很难，因为它们与我们习惯的 REST API 非常相似。

错误是通过一个名为`Status`的包装结构返回的。这个结构可以以多种方式构建，但本节中我们感兴趣的是以下几种：

```go
func Error(c codes.Code, msg string) error
func Errorf(c codes.Code, format string, a ...interface{}) error
```

它们都接受一个错误消息和一个错误代码。让我们专注于代码，因为消息只是描述错误的字符串。状态代码是预定义的代码，它们在不同的 gRPC 实现中是一致的。它与 HTTP 代码（如`404`和`500`）类似，但主要区别是它们有更描述性的名称，并且比 HTTP 中的代码少得多（总共 16 个）。

要查看所有这些代码，你可以访问 gRPC Go 文档（[`pkg.go.dev/google.golang.org/grpc/codes#Code`](https://pkg.go.dev/google.golang.org/grpc/codes#Code)）。它包含了每个错误的良好解释，并且比 HTTP 代码更不模糊，所以不要害怕。不过，对于本节，我们感兴趣的两种常见错误是：

+   `InvalidArgument`

+   `Internal`

第一个表示客户端指定了一个不适用于端点正常工作的参数。第二个表示系统的一个预期属性已损坏。

`InvalidArgument`非常适合验证输入。我们将在`AddTask`中使用它来确保`Task`的描述不为空（没有描述的任务是无用的），并且已指定截止日期且不在过去。注意，我们使截止日期成为必需的，但如果你希望使其可选，我们只需检查请求中的`DueDate`属性是否为`nil`并相应地处理：

```go
import (
  //...
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/status"
}
func (s *server) AddTask(_ context.Context, in *pb.AddTaskRequest)
(*pb.AddTaskResponse, error) {
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

这些检查将确保我们的数据库中只有有用的任务，并且我们的截止日期在将来。

最后，我们还有一个可能来自`addTask`函数的错误，这将是数据库传递的错误。我们可以进行广泛的检查，根据每个数据库错误创建更精确的错误代码，但在这个例子中，为了简单起见，我们只是简单地说任何数据库错误都是`Internal`错误。

我们将从`addTask`函数获取潜在的错误，并执行与我们对`InvalidArgument`所做类似的操作，但这次将是一个`Internal`代码，我们将使用`Errorf`函数来传递错误的详细信息：

```go
func (s *server) AddTask(_ context.Context, in *pb.AddTaskRequest) (*pb.AddTaskResponse, error) {
  //...
  id, err := s.d.addTask(in.Description,
  in.DueDate.AsTime())
  if err != nil {
    return nil, status.Errorf(
      codes.Internal,
      "unexpected error: %s",
      err.Error(),
    )
  }
  //...
}
```

现在，我们已经完成了服务器端。我们可以切换到客户端 - 如果你之前没有注意到，我们已经在`addTask`中“处理”错误了。我们有以下几行：

```go
res, err := c.AddTask(context.Background(), req)
if err != nil {
  panic(err)
}
```

当然，客户端可能会进行更复杂的错误处理或甚至恢复，但我们的目标现在是要确保我们的服务器错误被正确地传递给客户端。为了测试`InvalidArgument`错误，我们可以简单地尝试添加一个没有描述的`Task`。在`main`的末尾，我们可以添加以下内容：

```go
import (
  //...
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/status"
)
func main() {
  //...
  fmt.Println("-------ERROR-------")
  addTask(c, "", dueDate)
  fmt.Println("-------------------")
}
```

然后，我们运行我们的服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

而我们的客户端应该返回预期的错误：

```go
$ go run ./client 0.0.0.0:50051
-------ERROR-------
panic: rpc error: code = InvalidArgument desc = expected a task description, got an empty string
```

然后，我们可以通过提供一个过去时间的`Time`实例来检查截止日期错误：

```go
fmt.Println("-------ERROR-------")
// addTask(c, "", dueDate)
addTask(c, "not empty", time.Now().Add(-5 * time.Second))
fmt.Println("-------------------")
```

我们应该得到以下结果：

```go
$ go run ./client 0.0.0.0:50051
-------ERROR-------
panic: rpc error: code = InvalidArgument desc = expected a task due_
date that is in the future
```

最后，我们不会显示`Internal`错误，因为这会使我们在内存数据库中创建一个假错误，但请理解它将返回以下内容：

```go
$ go run ./client 0.0.0.0:50051
-------ERROR-------
panic: rpc error: code = Internal desc = unexpected error: <AN_ERROR_
MESSAGE>
```

在完成本节之前，了解我们如何检查错误的类型并相应地采取行动也很重要。我们基本上会恐慌，但会有更易读的消息。例如，想象以下代码的情况：

```go
rpc error: code = InvalidArgument desc = expected a task due_date that
is in the future
```

相反，我们将打印以下内容：

```go
InvalidArgument: expected a task due_date that is in the future
```

为了做到这一点，我们将修改`addTask`，使其在出现错误时尝试使用`FromError`函数将其转换为状态——如果转换正确完成，我们将打印错误代码和错误消息；如果没有转换为状态，我们将像以前一样直接 panic：

```go
func addTask(c pb.TodoServiceClient, description string, dueDate time.
  Time) uint64 {
  //...
  res, err := c.AddTask(context.Background(), req)

  if err != nil {
    if s, ok := status.FromError(err); ok {
      switch s.Code() {
      case codes.InvalidArgument, codes.Internal:
        log.Fatalf("%s: %s", s.Code(), s.Message())
      default:
        log.Fatal(s)
      }
    } else {
      panic(err)
    }
  }
  //...
}
```

现在，在运行了之前定义的错误之一后，我们可以得到以下结果：

```go
$ go run ./client 0.0.0.0:50051
-------ERROR-------
InvalidArgument: expected a task due_date that is in the future
```

## Bazel

重要提示

这里展示的命令每次更新与 gRPC 相关的导入时都需要使用。为了简化，我们只在本章中展示一次，并假设你将在其他部分中能够做到这一点。

由于我们将在本章中添加更多依赖项，我们需要更新我们的`BUILD`文件。如果我们现在尝试使用 Bazel 运行服务器，我们会得到一个错误，显示以下内容：

```go
No dependencies were provided.
Check that imports in Go sources match importpath attributes in deps.
```

为了解决这个问题，我们只需运行`gazelle`命令，如下所示：

```go
$ bazel run //:gazelle
```

然后，我们就能正确地运行服务器和客户端：

```go
$ bazel run //server:server 0.0.0.0:50051
listening at 0.0.0.0:50051
$ bazel run //client:client 0.0.0.0:50051
```

总结一下，我们看到了我们可以在服务器端使用`status`包中的`Error`和`Errorf`函数创建一个错误。我们有多个错误代码可供选择。我们只看到了两个，但它们是常见的。最后，在客户端，我们看到了我们可以根据错误代码相应地采取行动，通过将 Go 错误转换为状态并根据状态码编写条件。

# 取消调用

当你想根据某些条件停止一个调用或中断一个长时间运行的流时，gRPC 为你提供了可以在任何时候执行的取消函数。

如果你之前在任何分布式系统代码或 API 中使用过 Go，你可能见过一个名为`context`的类型。这是提供请求范围信息和在 API 的参与者之间传递信号的习惯用法，这也是 gRPC 的一个重要组成部分。

如果你没有注意到，到目前为止，我们每次发起请求时都使用了`context.Background()`。在 Go 文档中，这被描述为返回“*一个非空、空的上下文。它永远不会取消，没有值，也没有截止日期*。”正如你可以猜到的，仅此不足以用于生产就绪的 API，以下是一些原因：

+   如果用户想要提前终止请求怎么办？

+   如果 API 调用永远不会返回怎么办？

+   如果我们需要服务器知道全局值（例如，一个认证令牌）怎么办？

在本节中，让我们专注于第一个问题，在接下来的两个部分中，我们将回答其他问题。

为了获得取消调用的能力，我们将使用`context`包中的`WithCancel`函数（[`pkg.go.dev/context#WithCancel`](https://pkg.go.dev/context#WithCancel)）。这个函数将返回构建的上下文和一个`cancel`函数，我们可以执行它来中断使用上下文进行的调用。所以，现在，我们不仅使用`context.Background()`，我们将创建一个上下文，如下所示：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

注意，在函数末尾调用 `cancel` 函数是很重要的，以释放与上下文相关的资源。为了确保函数被调用，我们可以使用 `defer`。然而，这并不意味着我们不能在函数结束之前调用该函数。

作为例子，我们将创建一个虚构的需求。它是虚构的，因为我们将改进/删除在本节中将要编写的代码。虚构的需求是在我们收到逾期 `Task` 后取消 `ListTasks` 调用。我们可以同意，从功能的角度来看，这没有意义，但无论如何，本节的目标是尝试取消一个调用。

要实现这样的功能，我们将使用 `WithCancel` 函数创建上下文，将此上下文传递给 `ListTasks` API 端点，最后，我们将在读取循环中添加另一个 `if`，检查是否存在逾期任务。如果确实如此，我们将调用 `cancel` 函数：

```go
func printTasks(c pb.TodoServiceClient) {
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()
  //...
  stream, err := c.ListTasks(ctx, req)
  //...
  for {
    //...
    if res.Overdue {
      log.Printf("CANCEL called")
      cancel()
    }
    fmt.Println(res.Task.String(), "overdue: ",
    res.Overdue)
  }
}
```

注意，我们可能会选择断开连接而不是直接调用 `cancel` 函数。然后，`defer cancel()` 将触发，服务器将停止工作。然而，我决定直接调用 `cancel` 并让客户端循环运行，因为我想要展示，当我们取消调用时，我们会收到一个错误。

现在，我们需要意识到 `cancel` 需要时间在网络中传播，因此服务器可能会在我们不知道的情况下继续运行。为了检查服务器发送的内容，我们将在将其发送到客户端之前，在终端上简单地打印出 `Task`：

```go
func (s *server) ListTasks(req *pb.ListTasksRequest, stream
  pb.TodoService_ListTasksServer) error {
  return s.d.getTasks(func(t interface{}) error {
    //...
    log.Println(task)
    overdue := //...
    err := stream.Send(&pb.ListTasksResponse{
      //...
    })
    return err
  })
}
```

最后，我想提到，我们不需要在客户端的 `main` 函数中添加任何代码，这是因为我们已经看到，当我们运行当前的 `main` 代码时，我们会遇到逾期任务。第一个逾期任务应该出现在 `update` 部分：

```go
fmt.Println("-------UPDATE------")
updateTasks(c, []*pb.UpdateTasksRequest{
  {Id: id1, Description: "A better name for the task"},
  //...
}...)
printTasks(c, nil)
fmt.Println("-------------------")
```

这是因为，如果你记得，我们更新了 `Id` 值和那个 `Task` 的描述，并将 `Task` 的其余属性设置为默认值。这意味着 `Done` 将设置为 `false`，`DueDate` 将设置为空的时间对象。

现在，我们可以这样运行服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们运行客户端：

重要提示

在运行以下代码之前，请确保你已经注释掉了会导致恐慌的函数调用。这包括我们在上一节中添加的两个 `addTask`。

```go
$ go run ./client 0.0.0.0:50051
```

你应该在客户端注意到一切运行正常，即使 `update` 部分包含以下消息，也没有任何内容被取消：

```go
CANCEL called.
```

原因是服务器不知道调用已被取消。为了解决这个问题，我们可以使服务器能够感知取消。

要做到这一点，我们需要检查上下文的 `Done` 通道。当取消传播到服务器时，此通道将被关闭，在取消示例中，上下文将有一个等于 `context.Canceled` 的错误。当我们有这个事件时，我们知道服务器需要返回并有效地停止处理剩余的请求：

```go
func (s *server) ListTasks(req *pb.ListTasksRequest, stream
  pb.TodoService_ListTasksServer) error {
  ctx := stream.Context()
  return s.d.getTasks(func(t interface{}) error {
    select {
    case <-ctx.Done():
      switch ctx.Err() {
      case context.Canceled:
        log.Printf("request canceled: %s", ctx.Err())
      default:
      }
      return ctx.Err()
    /// TODO: replace following case by 'default:' on production APIs.
    case <-time.After(1 * time.Millisecond):
    }
    //...
  })
}
```

在运行此代码之前，有几个需要注意的事项。

第一件事是，当处理`stream`时，我们可以通过使用生成的流类型（在我们的案例中是`pb.TodoService_ListTasksServer`）中可用的`Context()`函数来获取上下文。

其次，请注意，我们故意在每个闭包调用中暂停 1 毫秒。在生产环境中不会发生这种情况；我们会有一个默认分支。这样做是为了让服务器有时间注意到取消操作。请注意，这个数字是任意的；这是我能在我的机器上注意到取消错误的最小时间。你可能需要将其设置得更大，或者你可以将其设置得更小。

现在，我们可以这样运行服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们可以再次运行客户端：

```go
$ go run ./client 0.0.0.0:50051
//...
CANCEL called
id:1 description:"A better name for the task" due_date:{} overdue:  true
unexpected error: rpc error: code = Canceled desc = context canceled
```

最后，你应该注意，在服务器端，你会收到以下信息：

```go
request canceled: context canceled
```

总结来说，我们看到了如何使用`context.WithCancel()`创建一个可取消的上下文。我们还看到这个函数返回一个`cancel`函数，我们需要在作用域结束时调用它来释放上下文关联的资源，但我们也可以根据某些条件提前调用它。最后，我们看到了如何让服务器知道取消操作，这样它就不会执行超过所需的工作。

# 指定截止日期

在处理异步通信时，截止日期是最重要的事情。这是因为通话可能因为网络或其他问题而永远无法返回。这就是为什么谷歌建议我们为每个 RPC 调用设置一个截止日期。幸运的是，对我们来说，这就像取消一个调用一样简单。

在客户端，我们首先需要做的是创建一个上下文。这类似于`WithCancel`函数，但这次，我们将使用`WithTimeout`。它接受一个父上下文，就像`WithCancel`一样，但除此之外，它还接受一个`Time`实例，表示我们愿意等待服务器回答的最大时间。

在`printTasks`中的`WithCancel`代替，我们现在将使用以下上下文：

```go
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Millisecond)
defer cancel()
```

显然，1 毫秒的超时时间对于让服务器回答来说太低了，但我们故意这样做是为了得到一个`DeadlineExceeded`错误。在现实场景中，我们需要根据为服务设定的要求来设置超时。这非常依赖于你的用例和服务的任务，所以你需要进行实验并跟踪服务器响应的平均时间。

这就是我们在 gRPC 中设置截止日期所需的一切。我们现在可以运行我们的服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们可以运行客户端：

```go
$ go run ./client 0.0.0.0:50051
//...
unexpected error: rpc error: code = DeadlineExceeded desc = context
  deadline exceeded
```

我们可以看到，第一个`ListTasks`正如预期的那样失败了。

现在，虽然这已经是你设置截止日期所需的一切，但你也可以让服务器知道`DeadlineExceeded`错误。即使从技术上讲这已经完成，因为我们当`Done`通道关闭时返回`ctx.Err()`，我们仍然想打印一条消息，说明截止日期已经超过。

要做到这一点，这就像`Canceled`错误一样，但这次，我们将在`ctx.Err()`的`switch`上添加一个`DeadlineExceeded`分支：

```go
func (s *server) ListTasks(req *pb.ListTasksRequest, stream
  pb.TodoService_ListTasksServer) error {
  ctx := stream.Context()
  return s.d.getTasks(func(t interface{}) error {
    select {
    case <-ctx.Done():
      switch ctx.Err() {
      //...
      case context.DeadlineExceeded:
        log.Printf("request deadline exceeded: %s",
        ctx.Err())
      }
      return ctx.Err()
    //...
  }
  //...
}
```

如果我们重新运行服务器和客户端，现在应该在运行服务器的终端上看到以下消息：

```go
request deadline exceeded: context deadline exceeded
```

总结来说，我们看到了，类似于`WithCancel`，我们可以使用`WithTimeout`来为调用创建一个截止日期。建议始终设置一个截止日期，因为我们可能永远也收不到服务器的回答。最后，我们还看到了如何使服务器具有截止日期意识，以便它不会过度工作。

# 发送元数据

基于上下文构建的另一个特性是可以通过调用传递元数据。在 gRPC 中，这些元数据可以是 HTTP 头或 HTTP 尾迹。它们都是键值对列表，用于多种目的，例如传递身份验证令牌和数字签名、数据完整性等。在本节中，我们将主要关注通过头传递元数据。尾迹是发送在消息之后而不是之前的头。开发者较少使用，但 gRPC 使用它来实现流接口。无论如何，如果你感兴趣，可以查看`grpc.SetTrailer`函数（[`pkg.go.dev/google.golang.org/grpc#SetTrailer`](https://pkg.go.dev/google.golang.org/grpc#SetTrailer)）。

我们的用例是将身份验证令牌传递给`UpdateTasks` RPC 端点，并在检查之后，我们将决定是更新任务还是返回一个`Unauthenticated`错误。显然，我们不会处理如何生成`auth`令牌，因为这是一个实现细节，但我们将简单地使用`authd`作为正确的令牌，其他所有内容都将被视为不正确。

让我们从服务器端开始。服务器正在接收上下文中的数据；因此，我们将使用 gRPC 中`metadata`包的`FromIncomingContext`函数。这将返回一个映射和是否有元数据。在`UpdateTasks`中，我们可以做以下操作：

```go
func (s *server) UpdateTasks(stream pb.TodoService_UpdateTasksServer) error {
  ctx := stream.Context()
  md, _ := metadata.FromIncomingContext(ctx)
  //...
}
```

我们现在可以检查是否提供了身份验证令牌。为此，这是一个简单的 Golang 映射使用。我们尝试访问`md`映射中的`auth_token`元素。它将返回值和一个布尔值，表示键是否在映射中。如果是，我们将检查只有一个值，并且这个值等于`"authd"`。如果不是，我们将返回一个`Unauthenticated`错误：

```go
func (s *server) UpdateTasks(stream pb.TodoService_UpdateTasksServer) error {
  ctx := stream.Context()
  md, _ := metadata.FromIncomingContext(ctx)
  if t, ok := md["auth_token"]; ok {
    switch {
    case len(t) != 1:
      return status.Errorf(
        codes.InvalidArgument,
        "auth_token should contain only 1 value",
      )
    case t[0] != "authd":
      return status.Errorf(
        codes.Unauthenticated,
        "incorrect auth_token",
      )
    }
  } else {
    return status.Errorf(
      codes.Unauthenticated,
      "failed to get auth_token",
    )
  }
  //...
}
```

对于服务器来说，这就是全部了；如果没有元数据，如果有元数据但没有`auth_token`，如果有多个值的`auth_token`，以及如果`auth_token`的值与`"authd"`不同，我们将返回一个错误。

现在我们可以转到客户端发送适当的头信息。这可以通过`metadata`包中的另一个函数`AppendToOutgoingContext`来完成。我们知道在调用端点之前我们已经创建了一个上下文，所以我们只需将`auth_token`附加到它。这就像以下这样简单：

```go
func updateTasks(c pb.TodoServiceClient, reqs ...*pb.
  UpdateTasksRequest) {
  ctx := context.Background()
  ctx = metadata.AppendToOutgoingContext(ctx, "auth_token", "authd")
  stream, err := c.UpdateTasks(ctx)
  //...
}
```

我们通过包含键值对的新的上下文来覆盖我们创建的上下文。请注意，键值对可以交错。这意味着我们可以有如下情况：

```go
metadata.AppendToOutgoingContext(ctx, K1, V1, K2, V2, ...)
```

在这里，`K`代表键，`V`代表值。

现在我们可以运行服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后我们运行客户端：

```go
$ go run ./client 0.0.0.0:50051
```

一切都应该顺利。然而，如果你为`auth_token`设置了不同于`"authd"`的值，你应该得到以下消息：

```go
unexpected error: rpc error: code = Unauthenticated desc = incorrect
auth_token
```

如果你没有设置`auth_token`头信息，你会看到以下内容：

```go
unexpected error: rpc error: code = Unauthenticated desc = failed to
get auth_token
```

假设你为`auth_token`设置了多个值，如下所示：

```go
ctx = metadata.AppendToOutgoingContext(ctx, "auth_token", "authd", 
"auth_token", "authd")
```

你应该得到以下错误：

```go
unexpected error: rpc error: code = InvalidArgument desc = auth_token
should contain only 1 value
```

总结来说，我们看到了如何使用`metadata.FromIncomingContext`函数从上下文中获取元数据，以及当我们这样做时可能出现的所有可能的错误。我们还看到了如何通过使用`metadata.AppendToOutgoingContext`函数将键值对附加到上下文中来实际从客户端发送元数据。

# 使用拦截器的外部逻辑

虽然某些头信息可能只适用于一个端点，但大多数情况下，我们希望能够在不同的端点之间应用相同的逻辑。在`auth_token`头信息的情况下，如果我们有多个路由，只有在用户登录时才能调用，我们不希望重复我们在上一节中做的所有检查。这会使代码膨胀；它不可维护；并且可能会在开发者寻找端点核心时分散他们的注意力。这就是为什么我们将使用身份验证拦截器。我们将提取那个身份验证逻辑，它将在 API 中的每次调用之前被调用。

我们的拦截器将被命名为`authInterceptor`。服务器端的拦截器将简单地执行我们在上一节中做的所有检查，如果一切顺利，将启动端点的执行。否则，拦截器将返回错误，端点将不会被调用。

要定义服务器端拦截器，我们有两种可能性。第一种是在我们使用单一 RPC 端点（例如，`AddTasks`）时使用。拦截器函数将如下所示：

```go
func unaryInterceptor(ctx context.Context, req interface{}, info
*grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error)
```

然后我们有了在流上工作的拦截器。它们看起来如下所示：

```go
func streamInterceptor(srv interface{}, ss grpc.ServerStream, info
*grpc.StreamServerInfo, handler grpc.StreamHandler) error
```

它们看起来非常相似。主要区别是参数的类型。现在，我们不会使用所有参数来处理我们的用例，所以我鼓励您查看文档（[`pkg.go.dev/google.golang.org/grpc`](https://pkg.go.dev/google.golang.org/grpc)）中的`UnaryServerInterceptor`和`StreamServerInterceptor`，并尝试使用它们。

让我们从 `AddTasks` 将会使用的单一拦截器开始。我们首先将检查提取到一个函数中，该函数将在拦截器之间共享。在一个名为 `interceptors.go` 的文件中，我们可以编写以下内容：

```go
import (
  "context"
  "google.golang.org/grpc"
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/metadata"
  "google.golang.org/grpc/status"
)
const authTokenKey string = "auth_token"
const authTokenValue string = "authd"
func validateAuthToken(ctx context.Context) error {
  md, _ := metadata.FromIncomingContext(ctx)
  if t, ok := md[authTokenKey]; ok {
    switch {
    case len(t) != 1:
      return status.Errorf(
        codes.InvalidArgument,
        fmt.Sprintf("%s should contain only 1 value", authTokenKey),
     )
    case t[0] != authTokenValue:
      return status.Errorf(
        codes.Unauthenticated,
        fmt.Sprintf("incorrect %s", authTokenKey),
      )
    }
  } else {
    return status.Errorf(
      codes.Unauthenticated,
      fmt.Sprintf("failed to get %s", authTokenKey),
    )
  }
  return nil
}
```

这与我们直接在 `UpdateTasks` 中所做的没有区别。但现在，编写我们的拦截器很简单。我们只需调用 `validateAuthToken` 函数并检查错误。如果有错误，我们将直接返回它。如果没有错误，我们将调用 `handler` 函数，这实际上会调用端点：

```go
func unaryAuthInterceptor(ctx context.Context, req interface{}, info
*grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
  if err := validateAuthToken(ctx); err != nil {
    return nil, err
  }
  return handler(ctx, req)
}
```

我们可以对流做同样的操作。唯一会改变的是处理器的参数以及我们如何获取上下文：

```go
func streamAuthInterceptor(srv interface{}, ss grpc.ServerStream, info
*grpc.StreamServerInfo, handler grpc.StreamHandler) error {
  if err := validateAuthToken(ss.Context()); err != nil {
    return err
  }
  return handler(srv, ss)
}
```

现在，你可能认为我们有函数，但没有人来调用它们。你完全正确。我们需要注册这些拦截器，以便我们的服务器知道它们的存在。这是在 `server/main.go` 中完成的，在那里我们可以将拦截器作为选项添加到 gRPC 服务器中。目前，我们创建服务器的方式如下：

```go
var opts []grpc.ServerOption
s := grpc.NewServer(opts...)
```

要添加拦截器，我们只需将它们添加到 `opts` 变量中：

```go
opts := []grpc.ServerOption{
  grpc.UnaryInterceptor(unaryAuthInterceptor),
  grpc.StreamInterceptor(streamAuthInterceptor),
}
```

我们现在可以运行服务器：

重要提示

在运行服务器之前，您可以在 `server/impl.go` 中的 `UpdateTasks` 函数中删除整个认证逻辑。由于拦截器将自动认证请求，这不再需要。

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们可以运行客户端：

```go
$ go run ./client 0.0.0.0:50051
--------ADD--------
rpc error: code = Unauthenticated desc = failed to get auth_token
exit status 1
```

如预期的那样，我们得到了一个错误，因为我们从未在客户端的 `addTask` 函数中添加 `auth_token` 标题。

显然，我们不想手动将标题添加到所有的调用中。我们将创建一个客户端拦截器，在发送请求之前为我们添加它。在客户端，我们有两种定义拦截器的方式。对于单一调用，我们有以下内容：

```go
func unaryInterceptor(ctx context.Context, method string, req
interface{}, reply interface{}, cc *grpc.ClientConn, invoker grpc.
UnaryInvoker, opts ...grpc.CallOption) error
```

对于流，我们有以下内容：

```go
func streamInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error)
```

如您所见，这一侧有更多的参数。而且，尽管大多数参数对我们用例来说并不重要，但我鼓励您查看 `UnaryClientInterceptor` 和 `StreamClientInterceptor` 的文档 ([`pkg.go.dev/google.golang.org/grpc`](https://pkg.go.dev/google.golang.org/grpc))，并尝试使用它们。

在客户端拦截器中，我们将简单地创建一个新的上下文，并在调用端点之前附加元数据。我们甚至不需要创建一个单独的函数来共享逻辑，因为这就像调用我们之前看到的 `AppendToOutgoingContext` 函数一样简单。

在 `client/interceptors.go` 中，我们可以编写以下内容：

```go
import (
  "context"
  "google.golang.org/grpc"
  "google.golang.org/grpc/metadata"
)
const authTokenKey string = "auth_token"
const authTokenValue string = "authd"
func unaryAuthInterceptor(ctx context.Context, method string, req,
  reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption) error {
  ctx = metadata.AppendToOutgoingContext(ctx, authTokenKey,
      authTokenValue)
  err := invoker(ctx, method, req, reply, cc, opts...)
  return err
}
func streamAuthInterceptor(ctx context.Context, desc *grpc.StreamDesc,
  cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts     ...grpc.CallOption) (grpc.ClientStream, error) {
  ctx = metadata.AppendToOutgoingContext(ctx, authTokenKey,      authTokenValue)
  s, err := streamer(ctx, desc, cc, method, opts...)
  if err != nil {
    return nil, err
  }
  return s, nil
}
```

最后，就像在服务器中一样，我们还需要注册这些拦截器。这次，这些拦截器将通过将 `DialOptions` 添加到我们在 `main` 中使用的 `Dial` 函数来注册。目前，你应该有如下内容：

```go
opts := []grpc.DialOption{
  grpc.WithTransportCredentials(insecure.NewCredentials()),
}
```

我们现在可以添加拦截器如下：

```go
opts := []grpc.DialOption{
  //...
  grpc.WithUnaryInterceptor(unaryAuthInterceptor),
  grpc.WithStreamInterceptor(streamAuthInterceptor),
}
```

当它们被注册后，我们可以运行我们的服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们可以运行客户端：

重要提示

在运行客户端之前，你可以在`client/main.go`文件中的`updateTask`中删除对`AppendToOutgoingContext`的调用。由于拦截器会自动执行，所以这不再需要。

```go
$ go run ./client 0.0.0.0:50051
```

现在所有的调用都应该没有错误地通过。

总结来说，在本节中，我们看到了我们可以在服务器和客户端的两侧编写单一和流拦截器。这些拦截器的目标是自动在多个端点之间执行一些重复性工作。在我们的例子中，我们自动化了`auth_token`头部的添加和检查以进行身份验证。

# 压缩有效载荷

虽然 Protobuf 将数据序列化为二进制格式，这比文本数据涉及更小的有效载荷，但我们可以在二进制数据上应用压缩。gRPC 为我们提供了 gzip Compressor ([`pkg.go.dev/google.golang.org/grpc/encoding/gzip`](https://pkg.go.dev/google.golang.org/grpc/encoding/gzip))，并且对于更高级的使用场景，允许我们编写自己的 Compressor ([`pkg.go.dev/google.golang.org/grpc/encoding`](https://pkg.go.dev/google.golang.org/grpc/encoding))。

在深入探讨如何使用 gzip Compressor 之前，重要的是要理解无损压缩可能会导致更大的有效载荷大小。如果你的有效载荷不包含重复数据，这正是 gzip 检测并压缩的数据，你将发送比所需的更多字节。所以，你需要对典型的有效载荷进行实验，看看 gzip 如何影响其大小。

为了展示一个例子，我在`helpers`文件夹中包含了一个名为`gzip.go`的文件，该文件包含一个名为`compressedSize`的辅助函数。这个函数返回序列化数据的原始大小及其经过 gzip 压缩后的大小：

```go
func compressedSizeM protoreflect.ProtoMessage (int, int) {
  var b bytes.Buffer
  gz := gzip.NewWriter(&b)
  out, err:= proto.Marshal(msg)
  if err != nil {
    log.Fatal(err)
  }
  if _, err := gz.Write(out); err != nil {
    log.Fatal(err)
  }
  if err := gz.Close(); err != nil {
    log.Fatal(err)
  }
  return len(out), len(b.Bytes())
}
```

由于这是一个通用函数，我们可以用它来处理任何消息。我们可以从一个不适合压缩的消息开始：`Int32Value`。所以，在文件的`main`函数中，我们将创建一个`Int32Value`实例，通过`compressedSize`函数传递它，并将打印原始大小和新大小：

```go
func main() {
  var data int32 = 268_435_456
  i32 := &wrapperspb.Int32Value{
    Value: data,
  }
  o, c := compressedSize(i32)
  fmt.Printf("original: %d\ncompressed: %d\n", o, c)
}
```

如果我们运行这个程序，我们应该得到以下结果：

```go
$ go run gzip.go
original: 6
compressed: 30
```

压缩后的有效载荷是原始大小的五倍。这绝对是在生产环境中需要避免的事情。显然，大多数时候，我们不会发送如此简单的消息，所以让我们看看一个更具体的例子。我们将使用本书中定义的`Task`消息：

```go
syntax = "proto3";
package todo;
import "google/protobuf/timestamp.proto";
option go_package = "github.com/PacktPublishing/gRPC-Go-for-Professionals/helpers/proto";
message Task {
  uint64 id = 1;
  string description = 2;
  bool done = 3;
  google.protobuf.Timestamp due_date = 4;
}
```

然后，我们可以使用以下命令来编译它：

```go
$ protoc --go_out=. \
         --go_opt=module=github.com/PacktPublishing/
                 gRPC-Go-for-Professionals/helpers \
         proto/todo.proto
```

然后，我们现在可以创建一个`Task`实例，并将`compressedSize`函数传递给它，以查看压缩的结果：

```go
func main() {
  task := &pb.Task{
    Id: 1,
    Description: "This is a task",
    DueDate: timestamppb.New(time.Now().Add(5 * 24 *
    time.Hour)),
  }
  o, c := compressedSize(task)
  fmt.Printf("original: %d\ncompressed: %d\n", o, c)
}
```

如果我们运行它，我们应该得到以下大小：

```go
$ go run gzip.go
original: 32
compressed: 57
```

这比之前的例子要好，但仍然不够高效，因为我们发送的字节比所需的更多。所以，在之前我们看到的情况下，使用 gzip 压缩是没有意义的。

最后，让我们看看压缩何时有用。假设我们的大部分`Task`实例都有很长的描述。例如，我们可能像这样：

```go
task := &pb.Task{
  //...
  Description: `This is a task that is quite long and requires a lot
  of work.
  We are not sure we can finish it even after 5 days.
  Some planning will be needed and a meeting is required.`,
  //...
}
```

然后，运行`compressedSize`函数将给出以下大小：

```go
$ go run gzip.go
original: 192
compressed: 183
```

这里的教训是，在 gRPC 中启用 gzip 压缩之前，我们需要了解我们的数据。现在，让我们看看如何启用它。

在服务器端（`server/main.go`），这就像添加以下导入一样简单：

```go
_ "google.golang.org/grpc/encoding/gzip"
```

注意，我们在它前面添加一个下划线，以避免编译器错误，错误信息是我们在不使用导入的情况下操作。

那就是服务器端的全部内容。在客户端，代码稍微多一点，但这也很简单。我们可以通过添加`DialOption`来为所有 RPC 端点启用压缩，或者我们可以通过添加`CallOption`来为单个端点启用压缩（[`pkg.go.dev/google.golang.org/grpc#CallOption`](https://pkg.go.dev/google.golang.org/grpc#CallOption)）。

对于第一个选项，我们可以简单地添加以下内容：

```go
opts := []grpc.DialOption{
  //...
  grpc.WithDefaultCallOptions(grpc.UseCompres
  sor(gzip.Name))
}
```

gzip 添加了与服务器中相同的作用，但没有前面的下划线。

而对于按调用添加压缩，我们可以添加`CallOption`。如果我们想将 gzip 压缩添加到`AddTask`调用中，我们会得到以下内容：

```go
res, err := c.AddTask(context.Background(), req, grpc.UseCompressor(gzip.Name))
```

总结一下，我们看到了总是添加压缩不是一个好主意，我们应该在测试我们的数据后再添加它。然后，我们看到了如何在服务器和客户端注册 gzip 压缩器。最后，我们看到了我们可以全局或按调用启用压缩。

# 保护连接

到目前为止，我们还没有使我们的连接安全——我们使用了不安全的凭证。在 gRPC 中，我们可以使用 TLS、mTLS 和 ATLS 连接。第一个使用单向认证，客户端可以验证服务器的身份。第二个是双向通信，服务器验证客户端的身份，客户端验证服务器的。最后，ATLS 类似于 TLS，但设计和优化了 Google 的使用。

如果你正在处理较小规模的通信或与 Google Cloud 合作，那么 mTLS 和 ATLS 都值得探索。如果你对 mTLS 感兴趣，你应该检查`grpc-go` GitHub 仓库中的 mTLS 文件夹：[`github.com/grpc/grpc-go/tree/master/examples/features/encryption/mTLS`](https://github.com/grpc/grpc-go/tree/master/examples/features/encryption/mTLS)。如果你想使用 ATLS，查看这个链接：[`grpc.io/docs/languages/go/alts/`](https://grpc.io/docs/languages/go/alts/)。然而，在我们的情况下，我们将看到最常用的加密形式，即 TLS。

要这样做，我们需要创建一些自签名的证书。显然，在生产环境中，这些证书将自动通过类似 Let’s Encrypt 的工具创建。然而，一旦这些证书可用，整体设置是相同的。

现在，为了简化，我们将从`grpc-go`存储库中的示例下载这些证书。这些证书也可以在`chapter7`文件夹下的`certs`目录中找到。我们首先需要获取服务器证书及其密钥：

```go
$ curl https://raw.githubusercontent.com/grpc/grpc-go/master/examples/
data/x509/server_cert.pem --output server_cert.pem
$ curl https://raw.githubusercontent.com/grpc/grpc-go/master/examples/
data/x509/server_key.pem --output server_key.pem
```

然后我们需要获取**证书颁发机构**（**CA**）证书：

```go
$ curl https://raw.githubusercontent.com/grpc/grpc-go/master/examples/
data/x509/ca_cert.pem --output ca_cert.pem
```

现在，我们可以从服务器开始。我们将添加凭证作为`ServerOption`，因为我们希望所有调用都加密。为了创建凭证，我们可以使用 gRPC 的`credential`包中的`NewServerTLSFromFile`函数。它读取两个文件，服务器证书和服务器密钥：

```go
func main() {
  //...
  creds, err := credentials.NewServerTLSFrom
  File("./certs/server_cert.pem", "./certs/server_key.pem")
  if err != nil {
    log.Fatalf("failed to create credentials: %v", err)
  }
}
```

一旦创建，我们就可以使用`grpc.Creds`函数，该函数创建一个`ServerOption`：

```go
opts := []grpc.ServerOption{
  grpc.Creds(creds),
  //...
}
```

让我们看看现在尝试运行服务器会发生什么：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们运行客户端：

```go
$ go run ./client 0.0.0.0:50051
--------ADD--------
rpc error: code = Unavailable desc = connection error: desc = "error
reading server preface: EOF"
```

我们得到一个错误，基本上告诉我们客户端无法连接到服务器。为了解决这个问题，我们需要转到客户端并创建凭证的`DialOption`。

这次，我们将使用`NewClientTLSFromFile`函数，该函数接受 CA 证书。为了测试目的，我们将添加主机 URL 作为第二个参数（证书域是*.test.example.com）。

```go
creds, err := credentials.NewClientTLSFromFile("./certs/ca_cert.pem",
  "x.test.example.com")
if err != nil {
  log.Fatalf("failed to load credentials: %v", err)
}
```

要添加凭证，我们使用一个名为 W`ithTransportCredentials`的函数，该函数创建一个`DialOption`。

```go
opts := []grpc.DialOption{
  grpc.WithTransportCredentials(creds),
  //grpc.WithTransportCredentials(insecure.NewCredentials())
  //...
}
```

注意，我们移除了不安全的凭证，因为我们现在想要加密通信。

让我们现在重新运行服务器：

```go
$ go run ./server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

然后，我们对客户端做同样的操作：

```go
$ go run ./client 0.0.0.0:50051
```

一切顺利——我们应该通过所有之前通过的调用，但现在我们的通信是安全的。

## Bazel

为了使用 Bazel 运行我们在这个部分编写的代码，我们需要在`BUILD`文件中包含证书文件。这可以通过导出它们并将它们作为`data`添加到`server_lib`和`client_lib`目标中来实现。

要导出文件，我们需要在`certs`文件夹中创建一个`BUILD.bazel`文件，该文件包含以下内容：

```go
exports_files([
  "server_cert.pem",
  "server_key.pem",
  "ca_cert.pem"
])
```

然后，在服务器的`BUILD`文件中，我们现在可以添加对`server_cert`和`server_key`的依赖，如下所示（在`server/BUILD.bazel`中）：

```go
go_library(
  name = "server_lib",
  //...
  data = [
    "//certs:server_cert.pem",
    "//certs:server_key.pem",
  ],
  //...
)
```

最后，我们可以在客户端添加对`ca_cert`的依赖，如下所示（在`client/BUILD.bazel`中）：

```go
go_library(
  name = "client_lib",
  //...
  data = [
    "//certs:ca_cert.pem",
  ],
  //...
)
```

你现在应该能够像前几章中展示的那样，使用 Bazel 正确运行服务器和客户端。

总结一下，我们了解到创建服务器端连接需要服务器证书和服务器密钥文件，而在客户端则需要 CA 证书。我们还使用了自签名证书，但在生产环境中，这些证书应由我们生成。最后，我们看到了如何创建`ServerOption`和`DialOption`以在 gRPC 中启用 TLS。

# 使用负载均衡分发请求

通常来说，负载均衡是一个复杂的话题。有许多种实现它的方法。gRPC 默认提供客户端负载均衡。相比于旁路或代理负载均衡，这并不是一个特别受欢迎的选择，因为它需要“知道”所有服务器的地址，并在客户端拥有复杂的逻辑，但它有一个优点，那就是可以直接与服务器通信，从而实现低延迟通信。如果你想了解更多关于如何为你的用例选择正确的[负载均衡方法，请查看以下文档](https://grpc.io/blog/grpc-load-balancing/)：[`grpc.io/blog/grpc-load-balancing/`](https://grpc.io/blog/grpc-load-balancing/)。

为了看到客户端负载均衡的力量，我们将部署我们的服务器在 Kubernetes 上的三个实例，并让客户端在这三个实例之间进行负载均衡。我事先创建了 Docker 镜像，这样我们就不必在这里经历所有这些步骤。如果你对检查 Docker 文件感兴趣，你可以在`server`和`client`文件夹中看到它们。它们有详细的文档。此外，我还将镜像上传到了 Docker Hub，这样我们就可以轻松地拉取它们（[`hub.docker.com/r/clementjean/grpc-go-packt-book/tags`](https://hub.docker.com/r/clementjean/grpc-go-packt-book/tags)）。

在部署服务器和客户端之前，让我们看看在代码方面我们需要做哪些更改。在服务器端，我们将简单地打印出我们收到的每个请求。这是通过一个类似于以下拦截器来完成的（在`server/interceptors.go`中）:

```go
func unaryLogInterceptor(ctx context.Context, req interface{}, info
*grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
  log.Println(info.FullMethod, "called")
  return handler(ctx, req)
}
func streamLogInterceptor(srv interface{}, ss grpc.ServerStream, info
  *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
  log.Println(info.FullMethod, "called")
  return handler(srv, ss)
}
```

这只是简单地打印出被调用的方法并继续执行。

之后，这些拦截器需要在拦截器链中注册。这是因为我们已经有了一个认证拦截器，而 gRPC 只接受一个`grpc.UnaryInterceptor`和`grpc.StreamInterceptor`的调用。现在，我们可以在`server`/`main.go`中将相同类型（单一或流）的两个拦截器合并：

```go
opts := []grpc.ServerOption{
  //...
  grpc.ChainUnaryInterceptor(unaryAuthInterceptor,
  unaryLogInterceptor),
  grpc.ChainStreamInterceptor(streamAuthInterceptor,
  streamLogInterceptor),
}
```

服务器端的介绍就到这里。现在让我们专注于客户端。我们将使用`grpc.WithDefaultServiceConfig`函数添加一个`DialOption`。这个函数需要一个 JSON 字符串作为参数，它代表服务及其方法的全局客户端配置。如果你对深入了解[配置感兴趣，可以查看以下文档](https://github.com/grpc/grpc/blob/master/doc/service_config.md)：[`github.com/grpc/grpc/blob/master/doc/service_config.md`](https://github.com/grpc/grpc/blob/master/doc/service_config.md)。

对于我们来说，配置将会很简单；我们只需说明我们的客户端应该使用`round_robin`负载均衡策略。默认策略被称为`pick_first`。这意味着客户端将尝试连接到所有可用的地址（由 DNS 解析），一旦它能够连接到一个地址，它将把所有请求发送到该地址。`round_robin`则不同。它将尝试连接到所有可用的地址。然后，它将依次将请求转发到每个服务器。

要设置 `round_robin` 负载均衡，我们只需要在 `client/main.go` 中添加一个 `DialOption`，如下所示：

```go
opts := []grpc.DialOption{
  //...
  grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":
  [{"round_robin":{}}]}`),
}
```

最后，有一点需要注意，负载均衡只与 DNS 方案一起工作。这意味着我们将改变运行客户端的方式。之前，我们有以下内容：

```go
$ go run ./client 0.0.0.0:50051
```

现在，我们需要在前面添加 `dns:///` 方案，如下所示：

```go
$ go run ./client dns:///$HOSTNAME:50051
```

现在，我们准备讨论部署我们的应用程序。让我们开始部署服务器。我们首先需要的是一个无头服务。这是通过将 `ClusterIP` 设置为 `None` 来实现的，这允许客户端通过 DNS 找到所有服务器实例。每个服务器实例都将有自己的 DNS A 记录，该记录指示实例的 IP。在此基础上，我们将向服务器公开端口 `50051` 并使选择器等于 `todo-server`，这样所有具有该选择器的 Pods 都将被公开。

目前，在 `k8s/server.yaml` 中，我们有以下内容：

```go
apiVersion: v1
kind: Service
metadata:
  name: todo-server
spec:
  clusterIP: None
ports:
  - name: grpc
    port: 50051
  selector:
    app: todo-server
```

之后，我们将创建一个包含 3 个实例的 Deployment。我们将确保这些 Deployment 有正确的标签，以便服务能够找到它们，并且我们将公开端口 `50051`。

我们现在可以在服务之后添加以下内容：

```go
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-server
  labels:
    app: todo-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: todo-server
  template:
    metadata:
      labels:
        app: todo-server
    spec:
      containers:
      - name: todo-server
        image: clementjean/grpc-go-packt-book:server
        ports:
        - name: grpc
          containerPort: 50051
```

我们现在可以通过以下命令部署服务器实例：

```go
$ kubectl apply -f k8s/server.yaml
```

稍后，我们应该有以下的 Pods（名称可能不同）：

```go
$ kubectl get pods
NAME                           READY   STATUS
todo-server-85cf594fb6-tkqm9   1/1     Running
todo-server-85cf594fb6-vff6q   1/1     Running
todo-server-85cf594fb6-w4s6l   1/1     Running
```

接下来，我们需要为客户端创建一个 Pod。通常，如果客户端不是一个微服务，我们就不需要将其部署到 Kubernetes 中。然而，由于我们的客户端是一个简单的 Go 应用程序，将其部署到容器中与我们的服务器实例通信会更容易一些。

在 `k8s/client.yaml` 中，我们有以下简单的 Pod：

```go
apiVersion: v1
kind: Pod
metadata:
  name: todo-client
spec:
  containers:
  - name: todo-client
    image: clementjean/grpc-go-packt-book:client
  restartPolicy: Never
```

我们现在可以通过以下命令运行客户端：

```go
$ kubectl apply -f k8s/client.yaml
```

几秒钟后，我们应该得到类似的输出（或者错误而不是完成）：

```go
$ kubectl get pods
NAME                           READY   STATUS
todo-client                    0/1     Completed
```

现在，最重要的是看到负载均衡的实际效果。为了做到这一点，我们将对每个服务器名称执行一个 `kubectl logs` 命令：

```go
$ kubectl logs todo-server-85cf594fb6-tkqm9
listening at 0.0.0.0:50051
/todo.v2.TodoService/UpdateTasks called
/todo.v2.TodoService/ListTasks called

$ kubectl logs todo-server-85cf594fb6-vff6q
listening at 0.0.0.0:50051
/todo.v2.TodoService/DeleteTasks called

$ kubectl logs todo-server-85cf594fb6-w4s6l
listening at 0.0.0.0:50051
/todo.v2.TodoService/AddTask called
/todo.v2.TodoService/AddTask called
/todo.v2.TodoService/AddTask called
/todo.v2.TodoService/ListTasks called
/todo.v2.TodoService/ListTasks called
```

现在，你可能会有不同的结果，但你应该能够看到负载被分配到不同的实例上。还有一点需要注意，因为我们没有使用真实的数据库，所以 `todo-client` 的日志可能是不正确的。这是因为我们可能在服务器 1 上有一个 `Task`，并要求列出服务器 2 的 `Task`，而服务器 2 并不知道我们想要的 `Task`。在生产环境中，我们会使用真实的数据库，这种情况不应该发生。

总结来说，我们看到了默认的负载均衡策略是 `pick_first`，它尝试按顺序连接到所有可用的地址，直到找到一个可到达的地址，并将所有请求发送给它。然后，我们使用了一个 `round_robin` 负载均衡策略，它依次将请求发送到每个服务器。最后，我们看到了在 gRPC 代码中设置客户端负载均衡是简单的。其余的配置主要是 DevOps 的工作。

# 摘要

在本章中，我们看到了使用 gRPC 时默认获得的关键特性。我们了解到我们通过错误代码和消息返回错误。与 HTTP 相比，gRPC 中的错误代码要少得多，这使得它们更不容易产生歧义。

之后，我们看到了我们可以使用上下文来使调用可取消并指定截止日期。这些特性对于确保在服务器端返回之前如果出现问题，我们的客户端不会无限期地等待，以及进行可靠调用非常重要。

使用上下文和中间件，我们还看到了我们可以发送元数据并使用它们来验证请求。在我们的案例中，我们每次请求时都会检查认证令牌。在客户端，我们看到了中间件可以自动为我们添加元数据。这对于在服务和/或端点之间共享的元数据特别有用。

然后，我们看到了如何加密网络通信。我们使用了 TLS，因为这是最常见的加密方式。我们看到了一旦我们有了证书，我们就可以简单地创建一个`ServerOption`和一个`DialOption`，让服务器和客户端知道如何相互理解。

之后，我们看到了如何压缩数据包。最重要的是，我们看到了何时这可能是有用的，何时则不是。

最后，我们使用了客户端负载均衡，采用`round_robin`策略，将请求分配到我们服务器的不同实例。

在下一章中，我们将看到更多与本章类似的重要特性。我们将介绍中间件的概念，并了解如何使用不同类型的中间件来使我们的 API 更加稳固。

# 测验

1.  上下文用于什么？

    1.  在客户端和服务器之间传递元数据

    1.  使调用可取消

    1.  指定超时

    1.  所有上述内容

1.  中间件用于什么？

    1.  在端点之间共享逻辑

    1.  拦截恶意数据

1.  使用 gRPC 中压缩的潜在问题是什么？

    1.  没有问题

    1.  存在数据包损坏的可能性

    1.  存在数据包变大的可能性

# 答案

1.  D

1.  A

1.  C

# 挑战

+   在服务器端实现更多错误。一个例子可能是处理来自`updateTask`和`deleteTasks`的错误，它们正在与数据库通信。

+   由于截止日期可以节省时间和资源，因此指定它们很重要。确保所有调用到我们的客户端都有 200 毫秒的截止日期。

+   创建一个客户端中间件，用于记录客户端发送的请求。
