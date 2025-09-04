

# 第六章：设计有效的 API

虽然 gRPC 性能良好，但很容易犯错误，这些错误在长期或大规模上可能会给你带来损失。在本章中，我们将探讨设计高效 gRPC API 的重要考虑因素。由于我们正在讨论 API 设计，这些考虑因素将与 Protobuf 相关联，因为正如你所知，我们在 Protobuf 中定义我们的类型和端点。

在本章中，我们将涵盖以下主题：

+   如何选择正确的整数类型

+   理解字段标签对序列化数据大小的影响

+   如何使用字段掩码解决过度获取问题

+   理解重复字段如何导致比预期更大的有效负载

# 技术要求

对于本章，你可以在附带的 GitHub 仓库中找到相关代码，该仓库位于名为`chapter6`的文件夹中（[`github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter6`](https://github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter6)）。

# 选择正确的整数类型

Protobuf 由于它的二进制格式以及它对整数的表示而性能优越。虽然某些类型，如字符串，是“原样”序列化并附加字段标签、类型和长度，但数字——尤其是整数——通常以比在计算机内存中布局的方式更少的位进行序列化。

然而，你可能已经注意到我说了“通常序列化”。这是因为如果你为你的数据选择了错误的整数类型，`varint`编码算法可能会将`int32`编码成 5 个字节或更多，而在内存中它只有 4 个字节。

让我们看看一个错误的整数类型选择的例子。假设我们想要编码值 268,435,456。我们可以通过使用 Go 标准库中的`unsafe.Sizeof`函数和由 Protobuf 提供的`proto.Marshal`函数来检查这个值在内存中和使用 Protobuf 序列化时的样子。最后，我们还将使用已知的`Int32Value`类型来包装该值，以便能够使用 Protobuf 进行序列化。

在编写主函数之前，让我们尝试创建一个名为`serializedSize`的泛型函数，该函数将返回整数在内存中的大小以及使用 Protobuf 序列化的相同整数的大小。

重要提示

这里展示的代码位于附带的 GitHub 仓库的`helpers`目录下。我们认为将 TODO API 和这类代码混合在一起没有意义，所以我们将其分开。

让我们先添加依赖项：

```go
$ go get –u google.golang.org/protobuf
$ go get –u golang.org/x/exp/constraints
```

第一是能够访问已知的`Int32Value`类型，第二是能够访问预定义的类型约束以用于泛型。

我们将使用泛型来接受任何类型的整数作为数据，并让我们指定一个包装消息，以便能够使用 Protobuf 序列化数据。我们将有以下函数：

```go
func serializedSizeD constraints.Integer, W
  protoreflect.ProtoMessage (uintptr,
    int) {
  //...
}
```

然后，我们可以简单地使用 Protobuf 库中的 `proto.Marshal` 函数来序列化包装器，并返回 `unsafe.Sizeof` 的结果和序列化数据的长度：

```go
func serializedSizeD constraints.Integer, W
  protoreflect.ProtoMessage (uintptr, int) {
  out, err := proto.Marshal(wrapper)

  if err != nil {
    log.Fatal(err)
  }
  return unsafe.Sizeof(data), len(out) - 1
}
```

之后，就简单了。我们只需从我们的 `main` 函数中调用该函数，传递一个包含值 `268,435,456` 的变量和一个 `Int32Value` 实例：

```go
import (
  "fmt"
  "unsafe"
  "google.golang.org/protobuf/proto"
  "google.golang.org/protobuf/reflect/protoreflect"
  "google.golang.org/protobuf/types/known/wrapperspb"
  "golang.org/x/exp/constraints"
)
//...

func main() {
  var data int32 = 268_435_456
  i32 := &wrapperspb.Int32Value{
    Value: data,
  }

  d, w := serializedSize(data, i32)
  fmt.Printf("in memory: %d\npb: %d\n", d, w)
}
```

如果我们运行这个程序，我们应该得到以下结果：

```go
$ go run integers.go
in memory: 4
pb: 5
```

现在，如果你仔细看了代码，你可能会认为 `len(out)` 后面的 `–1` 是在作弊。在 Protobuf 中，`Int32Value` 被序列化为 6 个字节。虽然你关于实际序列化大小是 6 个字节的事实是正确的，但前几个字节代表类型和字段标签。因此，为了使序列化数据的比较公平，我们移除了元数据，只比较数字本身。

你可能正在想，我们当前的 TODO API，它使用 `uint64` 作为 ID，也存在这个问题，你完全正确。你可以很容易地看到，通过将 `int32` 改为 `uint64`，将 `Int32Value` 改为 `UInt64Value`，并将我们的数据设置为 72,057,594,037,927,936：

```go
func main() {
  var data uint64 = 72_057_594_037_927_936
  ui64 := &wrapperspb.UInt64Value{
    Value: data,
  }
  d, w := serializedSize(data, ui64)
  fmt.Printf("in memory: %d\npb: %d\n", d, w)
}
```

使用前面的代码，我们会得到以下结果：

```go
$ go run integers.go
in memory: 8
pb: 9
```

这意味着在注册了大约 72 万亿个任务之后，我们会遇到这个问题。显然，对于我们的用例，我们使用 `uint64` 作为 `id` 是安全的，因为要出现这样的问题，我们需要地球上每个人创建 9000 万个任务（72 万亿 / 80 亿）。但这个问题在其他用例中可能更为严重，我们需要意识到我们 API 的局限性。

## 使用整数的替代方案

一个经常被引用，甚至被 Google 推荐的替代方案是使用字符串作为 ID。他们提到 2⁶⁴ (`int64`) 已经“不像以前那么大了。”在公司的背景下，这是可以理解的。他们必须处理大量的数据，以及比我们大多数人更大的数字。

然而，字符串相对于数字类型的优势不仅仅如此。最大的优势可能是你的 API 的演变。如果在某个时刻你需要存储更大的数字，你唯一的替代方案就是切换到字符串类型。但问题是，你之前使用的数字类型和字符串之间没有前后兼容性。因此，你将不得不在你的模式中添加一个新字段，使消息定义变得杂乱，并让开发者检查在与旧版/新版应用程序通信时 ID 是否被设置为字符串或数字。

字符串还提供了安全性，因为这些不能用于算术运算。这在一定程度上限制了聪明开发者，但也是一种好的限制，使他们不能对 ID 进行一些聪明的操作，最终导致数字溢出。ID 被有效地视为全局变量，不应该有人手动处理。

总之，对于某些用例，直接从字符串开始为 ID 编写可能是个好主意。如果你预计要扩展或简单地处理比整数限制更大的数字，字符串是解决方案。然而，在许多情况下，你可能只需要 `uint64`。只需了解你的需求并规划未来即可。

# 选择正确的字段标签

如你所知，字段标签与实际数据一起序列化，以便 Protobuf 知道将数据反序列化到哪个字段。由于这些标签以 `varint` 编码，标签越大，对序列化数据大小的冲击就越大。在本节中，让我们讨论你必须考虑的两个因素，以防止这些标签过多地影响你的有效载荷。

## 必需/可选

如果你意识到权衡，大字段标签可能没问题。处理大标签的一种常见方式是将它们视为用于可选字段。可选字段意味着它不太常填充数据，由于 Protobuf 不序列化未填充的字段，因此标签本身不会被序列化。然而，我们偶尔会填充这个字段，这将产生成本。

这种设计的优点是，在不创建大量消息以保持字段标签较小的情况下，将相关信息保持在一起。这将使代码更容易阅读，并让读者了解他们可以填充的可能字段。

然而，缺点是，如果你正在创建一个面向用户的 API，你可能会经常产生成本。这可能是由于用户不理解如何正确使用你的 API，或者简单地因为用户有特定的需求。这种情况也可能在公司环境中发生，但可以通过高级软件工程师或内部文档来缓解。

让我们看看大标签带来的负面影响的例子。为了举例，让我们假设我们有以下消息（`helpers/tags.proto`）：

```go
message Tags {
  int32 tag = 1;
  int32 tag2 = 16;
  int32 tag3 = 2048;
  int32 tag4 = 262_144;
  int32 tag5 = 33_554_432;
  int32 tag6 = 536_870_911;
}
```

注意，这些数字并非随机。如果你还记得，在 Protobuf 入门介绍中，我解释过标签是以 `varints` 编码的。这些数字是标签单独序列化所需额外一个字节的阈值。

现在，有了这个，我们将计算消息的大小，我们将逐步设置字段的值。我们将从一个空对象开始，然后设置 `tag`，然后 `tag2`，依此类推。请注意，我们将为所有字段设置相同的值（1）。这将显示仅序列化标签所需的额外开销。

在 `helpers/tags.go` 中，我们有以下内容：

```go
package main
import (
  "fmt"
  "log"
  "google.golang.org/protobuf/proto"
  "google.golang.org/protobuf/reflect/protoreflect"
  pb "github.com/PacktPublishing/gRPC-Go-for-Professionals/
    helpers/proto"
)
func serializedSizeM protoreflect.ProtoMessage int
{
  out, err := proto.Marshal(msg)
  if err != nil {
    log.Fatal(err)
  }
  return len(out)
}
func main() {
  t := &pb.Tags{}
  tags := []int{1, 16, 2048, 262_144, 33_554_432, 536_870_911}
  fields := []*int32{&t.Tag, &t.Tag2, &t.Tag3, &t.Tag4,
    &t.Tag5, &t.Tag6}
  sz := serializedSize(t)
  fmt.Printf("0 - %d\n", sz)
  for i, f := range fields {
    *f = 1
    sz := serializedSize(t)
    fmt.Printf("%d - %d\n", tags[i], sz-(i+1))
  }
}
```

我们重新使用了之前看到的`serializedSize`。我们通过取消对字段指针的引用来设置字段，我们使用新设置的字段计算`Tag`消息的大小，并打印结果。这个结果经过一点处理，只显示标签的字节。我们从大小中减去 i+1，因为 i 是零索引的（所以`+1`）。因此，实际上，我们从大小中减去了已设置的字段数量，这也是不包含标签序列化数据所需的大小（值为 1 的字节）。

最后，如果我们运行这个，我们将得到以下结果（美化后的）：

```go
$ go run tags.go
Tag             Bytes
----            ----
0               0
1               1
16              3
2048            6
262144          10
33554432        15
536870911       20
```

这告诉我们，每次我们通过一个阈值，我们的序列化数据就会多出一个字节的开销。一开始，我们有一个空消息，所以得到 0 字节，然后我们有一个标签 1，它序列化为 1 字节，之后标签 2 序列化为 2 字节，以此类推。我们可以查看两行之间的差异来得到开销。将`value`设置为标签为`2048`的字段而不是标签为`16`的字段的开销是 3 字节（6 - 3 字节）。

总结来说，我们需要保留较小的字段标签，用于最常使用或必需的字段。这是因为这些标签几乎总是会被序列化，而我们希望最小化标签序列化的影响。对于可选字段，我们可能会使用较大的标签来将相关字段放在一起，并且因此，我们可能会产生非重复的数据负载增加。

## 分割消息

通常，我们更喜欢分割消息以保持较小的对象和较少的字段，从而有较小的标签。这让我们能够将信息安排成实体，并理解给定信息所代表的内容。我们的`Task`消息就是一个例子。它将信息分组，我们可以在例如`UpdateTasksRequest`中重用这个实体，以接受一个功能齐全的`Task`作为请求。

然而，虽然能够将信息分离成实体很有趣，但这并不是免费的。你的负载会受到用户定义类型的使用的影響。让我们看看分割消息的例子以及它如何影响序列化数据的大小。这个例子表明，在分割消息时会有大小开销。为了展示这一点，我们将创建一个包含名称和一个名称包装器的消息。第一次检查大小，我们只会设置字符串，第二次我们只会设置包装器。这就是我所说的这样的消息：

```go
message ComplexName {
  string name = 1;
}
message Split {
  string name = 1;
  ComplexName complex_name = 2;
}
```

目前，我们不必担心这个示例的有用性。我们只是在尝试证明分割消息会有额外的开销。

然后，我们将编写一个`main`函数，该函数简单地首先设置`name`的值，然后计算大小并打印它。然后，我们将清除名称，设置`ComplexName.name`字段，计算大小并打印它。如果有开销，大小应该不同。在`helpers/split.go`中，我们有以下内容：

```go
package main
import (
  "fmt"
  "log"
  "google.golang.org/protobuf/proto"
  "google.golang.org/protobuf/reflect/protoreflect"
  pb "github.com/PacktPublishing/gRPC-Go-for-Professionals
    /helpers/proto"
)
func serializedSizeM protoreflect.ProtoMessage int
{
  out, err := proto.Marshal(msg)
  if err != nil {
    log.Fatal(err)
  }
  return len(out)
}
func main() {
  s := &pb.Split{Name: "Packt"}
  sz := serializedSize(s)
  fmt.Printf("With Name: %d\n", sz)
  s.Name = ""
  s.ComplexName = &pb.ComplexName{Name: "Packt"}
  sz = serializedSize(s)
  fmt.Printf("With ComplexName: %d\n", sz)
}
```

如果我们运行这个，我们应该得到：

```go
$ go run split.go
With Name: 7
With ComplexName: 9
```

实际上，这两个大小是不同的。但区别在哪里呢？区别在于用户定义的类型被序列化为长度限定类型。在我们的例子中，简单的名称会被序列化为 0a 05 50 61 63 6b 74。0a 是长度限定+标签 1 的线类型，其余的是字符。但对于复杂类型，我们有 12 07 0a 05 50 61 63 6b 74。我们识别了最后的 7 个字节，但前面还有两个字节。12 是`长度限定线类型+标签 2`，07 是后续字节的长度。

总之，我们再次面临权衡。消息中的标签越多，我们在有效载荷大小方面承担成本的可能性就越大。然而，我们越试图分割消息以保持标签小，我们也将承担更多的成本，因为数据将被序列化为长度限定数据。

## 改进`UpdateTasksRequest`

为了反思我们在上一节中学到的内容，我们将改进`UpdateTasksRequest`的序列化大小。这很重要，因为消息的使用上下文。这是一个客户端可能会发送 0 次或多次的消息，因为它用于客户端流式 RPC 端点。这意味着序列化数据中的任何开销都将乘以我们在线发送此消息的次数。

重要提示

以下代码位于附带的 GitHub 仓库中。你将在`proto/todo/v2`文件夹中找到新的 Protobuf 代码，并且`UpdateTasks`的服务器/客户端代码将被更新以反映这一变化。最后，需要注意的一点是我们不提供前后兼容性。`chapter6`中的服务器不能接收来自`chapter5`中的客户端的请求。需要更多的工作来实现这一点。

如果我们查看当前的消息，我们有以下内容：

```go
message UpdateTasksRequest {
  Task task = 1;
}
```

这正是我们想要描述的内容，但现在我们知道由于子消息的存在，将序列化一些额外的字节。为了解决这个问题，我们可以简单地复制我们允许用户更改的字段以及描述要更新哪个任务的 ID。这将给我们以下结果：

```go
message UpdateTasksRequest {
  uint64 id = 1;
  string description = 2;
  bool done = 3;
  google.protobuf.Timestamp due_date = 4;
}
```

这与`Task`消息的定义相同。

现在，你可能认为我们在重复自己，这样做是浪费的。然而，这样做有两个重要的好处：

+   我们不再需要为用户定义类型的序列化承担开销。在每次请求中，我们节省 2 个字节（标签+类型和长度）。

+   现在，我们对用户可能更新的字段有了更多的控制。如果我们不想让用户再更改`due_date`，我们只需将其从`UpdateTaskRequest`消息中删除并保留标签 4。

为了证明这在序列化数据大小方面更有效，我们可以暂时修改`server/impl.go`中的`UpdateTasks`函数，以`chapter5`和`chapter6`为例。为了计算有效负载的大小，我们可以使用我们之前使用的`proto.Marshal`并汇总总序列化大小。最后，我们可以在接收到 EOF 时在终端上打印结果。

在`chapter6`中，它看起来是这样的：

```go
func (s *server) UpdateTasks(stream
  pb.TodoService_UpdateTasksServer) error {
  totalLength := 0
  for {
    req, err := stream.Recv()
    if err == io.EOF {
      log.Println("TOTAL: ", totalLength)
      return stream.SendAndClose(&pb.UpdateTasksResponse{})
    }
    if err != nil {
      return err
    }
    out, _ := proto.Marshal(req)
    totalLength += len(out)
    s.d.updateTask(
      req.Id,
      req.Description,
      req.DueDate.AsTime(),
      req.Done,
    )
  }
}
```

对于`chapter5`，这导致网络中发送了 56 字节的请求，而对于`chapter6`，我们只发送了 50 字节。再次强调，这看起来微不足道，因为我们是在小规模上做的，但一旦我们收到流量，它将迅速累积并影响我们的成本。

# 采用 FieldMasks 以减少有效负载

在改进我们的`UpdateTasksRequest`消息之后，我们现在可以开始查看`FieldMasks`以进一步减少有效负载大小，但这次我们将专注于`ListTasksResponse`。

首先，让我们了解什么是`FieldMasks`。它指的是包含一系列路径的对象，告诉 Protobuf 包含哪些字段，并隐式地告诉它哪些不应该包含。以下是一个例子。假设我们有一个`Task`这样的消息：

```go
message Task {
  uint64 id = 1;
  string description = 2;
  bool done = 3;
  google.protobuf.Timestamp due_date = 4;
}
```

如果我们只想选择`id`和`done`字段，我们可以有一个简单的`FieldMask`，如下所示：

```go
mask {
  paths: "id"
  paths: "done"
}
```

然后，我们可以将这个掩码应用于`Task`的一个实例，并且它将只保留提到的字段的值。当我们进行类似于`GET`的操作，并且不想获取太多不必要的数据（过度获取）时，这很有趣。

我们的 TODO API 包含一个这样的用例：`ListTasks`。为什么？因为如果用户只想获取部分信息，他们将无法做到。选择部分数据可能对同步本地存储到后端等特性很有用。如果后端有 ID 1、2 和 3，而本地有 1、2、3、4 和 5，我们希望能够计算出需要上传的任务的增量。为此，我们只需要列出 ID，获取描述、完成日期和`due_date`值将是浪费的。

## 改进 ListTasksRequest

`ListTasksResponse`是一种服务器端流式 API。我们发送一个请求，然后得到 0 个或多个响应。这一点很重要，因为发送`FieldMask`并不是免费的。我们仍然需要在网络中传输字节。然而，在我们的情况下，使用掩码很有趣，因为我们只需发送一次，它就会应用于服务器返回的所有元素。

我们需要做的第一件事是声明这样的`FieldMask`。为此，我们导入`field_mask.proto`并在`ListTasksRequest`中添加一个字段：

```go
import "google/protobuf/field_mask.proto";
//...
message ListTasksRequest {
  google.protobuf.FieldMask mask = 1;
}
```

然后，我们可以转到服务器端并将该掩码应用于我们发送的所有响应。这是通过反射和一些样板代码完成的。我们需要做的第一件事是在服务器中添加一个依赖项，以处理切片并特别访问`Contains`函数：

```go
$ go get golang.org/x/exp/slices
```

之后，我们可以使用反射。我们将遍历给定消息的所有字段，如果其名称不在掩码路径中，我们将移除其值：

重要提示

以下代码是一个简单的实现，用于在消息中过滤字段，但这对于我们的用例来说已经足够了。在现实中，`FieldMasks` 有更多强大的功能，如过滤映射、列表和子消息。不幸的是，Protobuf 的 Go 实现没有提供像其他实现那样的此类实用工具，因此我们需要依赖编写自己的代码或使用社区项目。

```go
import (
  "google.golang.org/protobuf/proto"
  "google.golang.org/protobuf/reflect/protoreflect"
  "google.golang.org/protobuf/types/known/fieldmaskpb"
  "golang.org/x/exp/slices"
)
//...
func Filter(msg proto.Message, mask *fieldmaskpb.FieldMask) {
  if mask == nil || len(mask.Paths) == 0 {
    return
  }
  rft := msg.ProtoReflect()
  rft.Range(func(fd protoreflect.FieldDescriptor, _
    protoreflect.Value) bool {
    if !slices.Contains(mask.Paths, string(fd.Name())) {
      rft.Clear(fd)
    }
    return true
  })
}
```

这样，我们现在基本上可以在 `ListTasks` 实现中使用 `Filter` 来过滤将在 `ListTasksResponse` 中发送的 `Task` 对象：

```go
func (s *server) ListTasks(req *pb.ListTasksRequest, stream
  pb.TodoService_ListTasksServer) error {
  return s.d.getTasks(func(t interface{}) error {
    task := t.(*pb.Task)
    Filter(task, req.Mask)
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

注意，`Filter` 在计算 `Overdue` 之前被调用。这是因为如果我们没有在 `FieldMask` 中包含 `due_date`，我们假设用户不关心逾期。最终，逾期将是 `false`，未序列化，因此不会通过网络发送。

然后，我们需要看看如何在客户端使用它。在这个例子中，`printTasks` 将只打印 ID。我们将接收 `FieldMask` 作为 `printTasks` 的参数，并将其添加到 `ListTasksRequest` 中：

```go
func printTasks(c pb.TodoServiceClient, fm *fieldmaskpb
  .FieldMask) {
  req := &pb.ListTasksRequest{
    Mask: fm,
  }
  //...
}
```

最后，使用 `fieldmaskpb.New`，我们首先使用路径 `id` 创建一个 `FieldMask`。这个函数将检查 `id` 是否是我们提供的第一个参数中的消息中的有效路径。如果没有错误，我们可以在我们的 `ListTasksRequest` 实例中设置 `Mask` 字段：

```go
func main() {
  //...
  fm, err := fieldmaskpb.New(&pb.Task{}, "id")
  if err != nil {
    log.Fatalf("unexpected error: %v", err)
  }
  //...
  fmt.Println("--------LIST-------")
  printTasks(c, fm)
  fmt.Println("-------------------")
  //...
}
```

如果我们运行它，我们应该得到以下输出：

```go
--------LIST-------
id:1 overdue:false
id:2 overdue:false
id:3 overdue:false
-------------------
```

注意，`overdue` 仍然被打印为 `false`，但在这个例子中，我们可以忽略它，因为我们已经在 `printTasks` 函数中打印了逾期，逾期（bool）的默认值是 `false`。

# 谨防解压缩重复字段

最后一个考虑因素对我们 TODO API 没有帮助，但值得一提。在 Protobuf 中，我们有不同的方式来编码重复字段。我们有两种重复字段的压缩和解压缩方式。

## 压缩重复字段

为了理解，让我们看看一个压缩重复字段的例子。假设我们有以下消息：

```go
message RepeatedUInt32Values {
  repeated uint32 values = 1;
}
```

这是一个简单的 `uint32` 标量类型的列表。如果我们使用值 1、2 和 3 进行序列化，我们会得到以下结果：

```go
$ cat repeated_scalar.txt | protoc --encode=
 RepeatedUInt32Values proto/repeated.proto | hexdump -C
0a 03 01 02 03
00000005
```

前一个命令中的 `repeated_scalar.txt` 包含以下内容：

```go
values: 1
values: 2
values: 3
```

这是一个压缩重复字段的例子，因为它如何包装多个值。你可能会认为这是正常的，因为这是一个列表，但我们将稍后看到这并不总是正确的。

要理解“包装多个值”的含义，我们需要仔细看看 `hexdump` 展示的十六进制数。我们有 5 个字节：0a 03 01 02 03。正如我们所知，重复字段被序列化为长度限定类型。所以 0a 是类型（`varint`）和字段标签（1）的组合，03 表示列表中有三个元素，其余的是实际值。

## 解压缩重复字段

然而，重复字段的序列化数据并不总是那么紧凑。让我们看看一个未展开的重复字段的例子。假设我们为名为`values`的字段添加了`packed`选项，并将其值设置为`false`：

```go
message RepeatedUInt32Values {
  repeated uint32 values = 1 [packed = false];
}
```

现在，如果我们用相同的值运行相同的命令，我们应该得到以下结果：

```go
$ cat repeated_scalar.txt | protoc --encode=
RepeatedUInt32Values proto/repeated.proto | hexdump -C
08 01 08 02 08 03
00000006
```

我们可以看到，我们有了完全不同的数据序列化方式。这次，我们反复序列化`uint32`。在这里，08 代表类型（`varint`）和标签（1），你可以看到它出现了三次，因为我们有三个值。如果我们重复字段中有超过两个值，这实际上会为每个值添加一个字节。在我们的例子中，我们序列化整个为 6 字节，而不是之前的 5 字节。

现在，你可能认为你将不会使用`packed`选项，并且你应该始终有一个`packed`字段。对于作用于标量的重复字段来说，你会是对的，但对于更复杂的类型来说则不然。例如，字符串、字节和用户定义的类型将始终以未打包的形式序列化，而且无法避免这一点。

让我们用一个用户定义的类型为例。假设我们有以下 Protobuf 代码：

```go
message UserDefined {
  uint32 value = 1;
}
message RepeatedUserDefinedValues {
  repeated UserDefined values = 1;
}
```

现在，我们可以尝试运行以下命令：

```go
$ cat repeated_ud.txt | protoc --encode=
RepeatedUserDefinedValues proto/repeated.proto | hexdump -C
0a 02 08 01 0a 02 08 02 0a 02 08 03
0000000c
```

前一个命令中的`repeated_ud.txt`包含以下内容：

```go
values: {value: 1}
values: {value: 2}
values: {value: 3}
```

我们可以看到，我们现在有了本章早期与子消息相关的开销，以及我们的重复字段现在是未打包的。我们有 0a 和 02，它们对应于子消息本身，以及 08 + value，它对应于名为`value`的字段。正如你所看到的，这现在浪费了更多的字节。

现在，由于在复杂类型上这是不可避免的，所以说我们不应该在这样类型上使用重复字段是不正确的。这是一个非常有用的概念，应该谨慎使用，并且我们应该意识到它的成本。

# 摘要

在本章中，我们看到了在设计我们的 API 时需要考虑的主要因素。其中大部分与 Protobuf 有关，因为它是我们 API 的接口，并且负责序列化和反序列化。我们了解到选择正确的整数类型很重要，这可能会导致有效载荷大小的问题，而且在我们想要演进我们的 API 时也会引发问题。

之后，我们看到了选择正确的字段标签也很重要。这是因为标签会与数据一起序列化，并且它们被序列化为`varints`。所以标签越大，我们的有效载荷就越大。

然后，我们看到了如何利用`FieldMasks`来选择我们所需的数据，并避免过度获取问题。虽然这个概念在 gRPC Go 中并不是非常发达，但其他实现广泛使用了它。这显著减少了我们在网络中发送的有效载荷。

最后，我们看到了在使用 Protobuf 中的重复字段时需要小心。这是因为如果我们在一个复杂类型上使用它们，我们会浪费一些字节。然而，不应该因为这一点而避免使用重复字段。有时它们是正确的数据结构。在下一章中，我们将介绍如何使 API 调用既高效又安全。

# 测验

1.  为什么对于整数类型来说，并不总是使用 `varint` 编码更优？

    1.  没有理由，它们总是比固定整数更优

    1.  `varint` 编码将更大的数字序列化成更多的字节

    1.  `varint` 编码将较小的数字序列化成更多的字节

1.  我们如何得到消息序列化后的字节数？

    1.  `proto.Marshal +` `len`

    1.  `proto.Unmarshal +` `len`

    1.  `len`

1.  我们应该给经常填充的字段分配什么样的标签？

    1.  更大的标签

    1.  较小的标签

1.  将消息拆分以使用较小的标签的主要问题是什么？

    1.  我们有开销，因为子消息被序列化为长度分隔类型

    1.  没问题——这就是正确的方法

1.  什么是 `FieldMask`？

    1.  一组字段路径，告诉我们需要排除哪些数据

    1.  一组字段路径，告诉我们需要包含哪些数据

1.  何时重复字段以未打包的形式序列化？

    1.  当重复字段作用于标量类型时

    1.  只有当我们使用带有值 `false` 的打包选项时

    1.  当重复字段作用于复杂类型时

# 答案

1.  B

1.  A

1.  B

1.  A

1.  B

1.  C
