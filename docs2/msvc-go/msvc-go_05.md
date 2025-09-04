

# 同步通信

在本章中，我们将介绍微服务之间最常见的通信方式——同步通信。在 *第二章* 中，我们已经在我们的微服务中实现了通过 HTTP 协议进行通信并在 JSON 格式返回结果的逻辑。在 *第四章* 中，我们说明了 JSON 格式在数据大小方面并不是最有效的，并且有许多不同的格式为开发者提供了额外的优势，包括代码生成。在本章中，我们将向您展示如何使用 Protocol Buffers 定义服务 API 并为它们生成客户端和服务器代码。

到本章结束时，您将了解微服务之间同步通信的关键概念，并学会如何实现微服务的客户端和服务器。

您在本章中获得的知识将帮助您更好地组织客户端和服务器代码，生成序列化和通信的代码，并在您的微服务中使用它。在本章中，我们将涵盖以下主题：

+   同步通信简介

+   使用 Protocol Buffers 定义服务 API

+   实现网关和客户端

现在，让我们继续探讨同步通信的主要概念。

# 技术要求

要完成本章，您需要 Go 1.11、我们在上一章中安装的 Protocol Buffers 编译器以及一个 gRPC 插件。

您可以通过运行以下命令来安装 gRPC 插件：

```go
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest 
export PATH="$PATH:$(go env GOPATH)/bin" 
```

您可以在 [`github.com/PacktPublishing/microservices-with-go/tree/main/Chapter05`](https://github.com/PacktPublishing/microservices-with-go/tree/main/Chapter05) 找到本章的 GitHub 代码。

# 同步通信简介

在本节中，我们将介绍同步通信的基础知识，并介绍我们将用于微服务的 Protocol Buffers 的额外好处。

**同步通信**是网络应用（如微服务）之间交互的方式，其中服务通过**请求-响应模型**交换数据。该过程在以下图中展示：

![图 5.1 – 同步通信](img/Figure_5.1_B18865.jpg)

图 5.1 – 同步通信

有许多**协议**允许应用以这种方式进行通信。HTTP 是同步通信中最受欢迎的协议之一。在 *第二章* 中，我们已经在我们的微服务中实现了调用和处理 HTTP 请求的逻辑。

HTTP 协议允许您以不同的方式发送请求和响应数据：

+   `https://www.google.com/search?q=portugal` URL，`q=portugal` 是一个 URL 参数。

+   `User-Agent: Mozilla/5.0`.

+   **请求和响应体**：请求和响应可以包含包含任意数据的正文。例如，当客户端将文件上传到服务器时，文件内容通常作为请求正文发送。

当服务器由于错误无法处理客户端请求，或者由于网络问题未收到请求时，客户端会收到一个特定的响应，表明发生了错误。在 HTTP 协议的情况下，有两种类型的错误：

+   **客户端错误**：这种错误是由客户端引起的。此类错误的例子包括无效的请求参数（例如错误的用户名）、未经授权的访问以及访问未找到的资源（例如，不存在的网页）。

+   **服务器错误**：这种错误是由服务器引起的。这可能是一个应用程序错误或上游组件（例如数据库）的错误。

在*第二章*中，我们通过将结果数据作为 JSON 格式的 HTTP 响应体发送来实现我们的 API 处理器。我们通过使用 Go JSON 编码器实现了这一点：

```go
if err := json.NewEncoder(w).Encode(details); err != nil {
    log.Printf("Response encode error: %v\n", err)
}
```

如前一章所述，JSON 格式在数据大小方面并不是最优的。此外，它不提供有用的工具，例如由 Protocol Buffers 等格式提供的数据结构的跨语言代码生成工具。此外，通过 HTTP 发送请求并手动编码数据不是服务之间通信的唯一形式。有一些现有的**远程过程调用**（RPC）库和框架有助于在多个服务之间进行通信，并为应用程序开发者提供一些附加功能：

+   **客户端和服务器代码生成**：开发者可以生成连接到其他微服务并发送数据的客户端代码，以及生成接受传入请求的服务器代码。

+   **认证**：大多数 RPC 库和框架为跨服务请求提供认证选项，例如基于 TLS 和基于令牌的认证。

+   **上下文传播**：这是在请求中发送附加数据的能力，例如跟踪，我们将在*第十一章*中介绍。

+   **文档生成**：Thrift 可以为服务和数据结构生成 HTML 文档。

在下一节中，我们将介绍一些您可以在 Go 服务中使用并了解它们提供的功能的 RPC 库。

## Go RPC 框架和库

让我们回顾一些适用于 Go 开发者的流行 RPC 框架和库。

### Apache Thrift

我们已经在*第四章*中介绍了 Apache Thrift，并提到了其定义 RPC 服务的能力——由应用程序提供的一组函数，例如微服务。以下是一个 Thrift RPC 服务定义的示例：

```go
service MetadataService {
  Metadata get(1: string id)
}
```

服务的 Thrift 定义可以用来生成客户端和服务器代码。客户端代码将包括连接到服务实例的逻辑，以及向其发送请求、序列化和反序列化请求和响应结构。使用如 Apache Thrift 这样的库而不是手动进行 HTTP 请求的优势是能够为多种语言生成这样的代码：用 Go 编写的服务可以轻松与用 Java 编写的服务通信，而两者都将使用生成的代码进行通信，从而消除了实现序列化/反序列化逻辑的需求。此外，Thrift 允许我们为 RPC 服务生成文档。

### gRPC

**gRPC** 是由 Google 创建的一个 RPC 框架。gRPC 使用 HTTP/2 作为传输协议，并使用 Protocol Buffers 作为序列化格式。类似于 Apache Thrift，它提供了定义 RPC 服务并生成服务客户端和服务器代码的能力。除此之外，它还提供了一些额外功能，例如以下内容：

+   认证

+   上下文传播

+   文档生成

gRPC 的采用率比 Apache Thrift 高得多，它对流行的 Protocol Buffers 格式的支持使其非常适合微服务开发者。在本章中，我们将使用 gRPC 作为我们微服务之间同步通信的框架。在下一节中，我们将展示如何利用 Protocol Buffers 提供的功能来定义我们的服务 API。

# 使用 Protocol Buffers 定义服务 API

让我们演示如何使用 Protocol Buffers 格式定义服务 API，并使用 **proto** 编译器为与我们的每个服务进行通信生成客户端和服务器 gRPC 代码。这些知识将帮助您为使用行业中最受欢迎的通信工具之一，为您的微服务定义和实现 API 建立基础。

让我们从我们的元数据服务开始，并使用 Protocol Buffers 模式语言编写其 API 定义。

打开我们在上一章中创建的 `api/movie.proto` 文件，并向其中添加以下内容：

```go
service MetadataService {
    rpc GetMetadata(GetMetadataRequest) returns (GetMetadataResponse);
    rpc PutMetadata(PutMetadataRequest) returns (PutMetadataResponse);
}

message GetMetadataRequest {
    string movie_id = 1;
}

message GetMetadataResponse {
    Metadata metadata = 1;
}
```

我们刚刚添加的代码定义了我们的元数据服务和其 `GetMetadata` 端点。我们现在已经有了上一章中的 `Metadata` 结构，现在可以重用它。

让我们注意我们刚刚添加的代码的一些方面：

+   `GetMetadataRequest` 和 `GetMetadataResponse`。

+   **命名规范**：您应该为所有端点遵循一致的命名规则。我们将使用函数名作为所有请求和响应函数的前缀。

现在，让我们将评分服务的定义添加到同一个文件中：

```go
service RatingService {
    rpc GetAggregatedRating(GetAggregatedRatingRequest) returns (GetAggregatedRatingResponse);
    rpc PutRating(PutRatingRequest) returns (PutRatingResponse);
}

message GetAggregatedRatingRequest {
    string record_id = 1;
    int32 record_type = 2;
}

message GetAggregatedRatingResponse {
    double rating_value = 1;
}

message PutRatingRequest {
    string user_id = 1;
    string record_id = 2;
    int32 record_type = 3;
    int32 rating_value = 4;
}

message PutRatingResponse {
}
```

我们的服务评分有两个端点，我们以与元数据服务类似的方式定义了它们的请求和响应。

最后，让我们将电影服务的定义添加到同一个文件中：

```go
service MovieService {
    rpc GetMovieDetails(GetMovieDetailsRequest) returns (GetMovieDetailsResponse);
}

message GetMovieDetailsRequest {
    string movie_id = 1;
}

message GetMovieDetailsResponse {
    MovieDetails movie_details = 1;
}
```

现在我们的 `movie.proto` 文件包括了我们的结构定义和服务的 API 定义。我们准备好为新添加的服务定义生成代码。在应用程序的 `src` 目录中运行以下命令：

```go
protoc -I=api --go_out=. --go-grpc_out=. movie.proto  
```

前面的命令与我们之前用于生成数据结构代码的命令类似。然而，它还向编译器传递了一个 `--go-grpc_out` 标志。此标志告诉 Protocol Buffers 编译器以 gRPC 格式生成服务代码。

让我们看看我们的命令生成的输出编译后的代码。如果命令执行没有错误，你将在 `src/gen` 目录下找到一个 `movie_grpc.pb.go` 文件。该文件将包含为我们服务生成的 Go 代码。让我们看看生成的客户端代码：

```go
type MetadataServiceClient interface {
    GetMetadata(ctx context.Context, in *GetMetadataRequest, opts ...grpc.CallOption) (*GetMetadataResponse, error)
}

type metadataServiceClient struct {
    cc grpc.ClientConnInterface
}

func NewMetadataServiceClient(cc grpc.ClientConnInterface) MetadataServiceClient {
    return &metadataServiceClient{cc}
}
func (c *metadataServiceClient) GetMetadata(ctx context.Context, in *GetMetadataRequest, opts ...grpc.CallOption) (*GetMetadataResponse, error) {
    out := new(GetMetadataResponse)
    err := c.cc.Invoke(ctx, "/MetadataService/GetMetadata", in, out, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}
```

这生成的代码可以用于我们的应用程序中，从 Go 应用程序调用我们的 API。此外，我们可以为其他语言，如 Java，生成这样的客户端代码，向编译器命令中添加更多参数。这是一个可以节省我们大量时间的出色功能——在编写微服务应用程序时，我们不必编写调用我们服务的客户端逻辑，而可以使用生成的客户端并将它们插入到我们的应用程序中。

除了客户端代码外，Protocol Buffers 编译器还生成了可以用于处理请求的服务代码。在同一个 `movie_grpc.pb.go` 文件中，你会找到以下内容：

```go
type MetadataServiceServer interface {
    GetMetadata(context.Context, *GetMetadataRequest) (*GetMetadataResponse, error)
    mustEmbedUnimplementedMetadataServiceServer()
}
func RegisterMetadataServiceServer(s grpc.ServiceRegistrar, srv MetadataServiceServer) {
s.RegisterService(&MetadataService_ServiceDesc, srv)
}
```

我们将在我们的应用程序中使用我们刚刚看到的客户端和服务器代码。在下一节中，我们将修改我们的 API 处理程序以使用生成的代码，并使用 Protocol Buffers 格式处理请求。

# 实现网关和客户端

在本节中，我们将说明如何将生成的客户端和服务器 gRPC 代码插入到我们的微服务中。这将帮助我们切换它们之间的通信，从 JSON 序列化的 HTTP 到 Protocol Buffers gRPC 调用。

## 元数据服务

在*第二章*中，我们创建了我们的内部模型结构，例如元数据，而在*第四章*中，我们创建了它们的 Protocol Buffers 对应版本。然后，我们为我们的 Protocol Buffers 定义生成了代码。结果，我们有了我们模型结构的两个版本——内部版本，定义在 `metadata/pkg/model` 中，以及生成的版本，位于 `gen` 包中。

你可能会认为现在有两个类似的结构是多余的。虽然确实存在一定程度的冗余，因为这些重复定义，但这些结构实际上服务于不同的目的：

+   **内部模型**：您为应用程序手动创建的结构应在其代码库中跨用，例如存储库、控制器和其他逻辑。

+   **生成的模型**：由像 protoc 编译器这样的工具生成的结构，我们在前两章中使用过，应仅用于序列化。用例包括在服务之间传输数据或存储序列化数据。

你可能好奇为什么不建议在应用程序代码库中使用生成的结构。这里有多个原因，如下列所示：

+   **应用程序和序列化格式之间的不必要耦合**：如果你想要从一种序列化格式切换到另一种格式（例如，从 Thrift 切换到 Protocol Buffers），并且你的所有应用程序代码库都使用为先前序列化格式生成的结构，那么你需要重写不仅序列化代码，而且整个应用程序。

+   **生成的代码结构可能在不同版本之间有所不同**：虽然生成的结构的字段命名和高级结构在代码生成工具的不同版本之间通常是稳定的，但生成的代码的内部函数和结构可能从版本到版本有所不同。如果你的应用程序的任何部分使用了一些在代码生成器版本更新期间发生变化的生成函数，那么在代码生成器版本更新期间，你的应用程序可能会意外地崩溃。

+   **生成的代码通常更难使用**：在如 Protocol Buffers 这样的格式中，所有字段始终是可选的。在生成的代码中，这导致了许多可以具有 nil 值的字段。对于应用程序开发者来说，这意味着需要在所有应用程序中进行更多的 nil 检查，以防止可能的 panic。

由于这些原因，最佳实践是保留内部结构和生成的结构，并且仅使用生成的结构进行序列化。让我们通过以下示例说明如何实现这一点。

我们需要添加一些`metadata/pkg/model`目录，创建一个`mapper.go`文件，并将其内容添加如下：

```go
package model

import (
    "movieexample.com/gen"
)

// MetadataToProto converts a Metadata struct into a 
// generated proto counterpart.
func MetadataToProto(m *Metadata) *gen.Metadata {
    return &gen.Metadata{
        Id:          m.ID,
        Title:       m.Title,
        Description: m.Description,
        Director:    m.Director,
    }
}

// MetadataFromProto converts a generated proto counterpart 
// into a Metadata struct.
func MetadataFromProto(m *gen.Metadata) *Metadata {
    return &Metadata{
        ID:          m.Id,
        Title:       m.Title,
        Description: m.Description,
        Director:    m.Director,
    }
}
```

我们刚刚添加的代码将内部模型转换为生成的结构，然后再转换回来。在以下代码块中，我们将在服务器代码中使用它。

现在，让我们实现一个处理元数据服务的 gRPC 处理器，该处理器将处理对服务的客户端请求。在`metadata/internal/handler`包中，创建一个`grpc`目录并添加一个`grpc.go`文件：

```go
package grpc

import (
    "context"
    "errors"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "movieexample.com/gen"
    "movieexample.com/metadata/internal/controller"
    "movieexample.com/metadata/internal/repository"
    "movieexample.com/metadata/pkg/model"
)

// Handler defines a movie metadata gRPC handler.
type Handler struct {
    gen.UnimplementedMetadataServiceServer
    svc *controller.MetadataService
}

// New creates a new movie metadata gRPC handler.
func New(ctrl *metadata.Controller) *Handler {
    return &Handler{ctrl: ctrl}
}
```

让我们实现`GetMetadataByID`函数：

```go
// GetMetadataByID returns movie metadata by id.
func (h *Handler) GetMetadata(ctx context.Context, req *gen.GetMetadataRequest) (*gen.GetMetadataResponse, error) {
    if req == nil || req.MovieId == "" {
        return nil, status.Errorf(codes.InvalidArgument, "nil req or empty id")
    }
    m, err := h.svc.Get(ctx, req.MovieId)
    if err != nil && errors.Is(err, controller.ErrNotFound) {
        return nil, status.Errorf(codes.NotFound, err.Error())
    } else if err != nil {
        return nil, status.Errorf(codes.Internal, err.Error())
    }
    return &gen.GetMetadataResponse{Metadata: model.MetadataToProto(m)}, nil
}
```

让我们突出显示实现中的某些部分：

+   处理器嵌入生成的`gen.UnimplementedMetadataServiceServer`结构。这是由 Protocol Buffers 编译器强制执行未来兼容性所必需的。

+   我们的处理器以与生成的`MetadataServiceServer`接口中定义的完全相同的格式实现了`GetMetadata`函数。

+   我们正在使用`MetadataToProto`映射函数将我们的内部结构转换为生成的结构。

现在，我们已经准备好更新主文件并将其切换到 gRPC 处理器。更新`metadata/cmd/main.go`文件，更改其内容如下：

```go
package main

import (
    "context"
    "flag"
    "fmt"
    "log"
    "net"
    "time"

    "google.golang.org/grpc"
    "movieexample.com/gen"
    "movieexample.com/metadata/internal/controller"
    grpchandler "movieexample.com/metadata/internal/handler/grpc"
    "movieexample.com/metadata/internal/repository/memory"
    "movieexample.com/metadata/pkg/model"
)

func main() {
    log.Println("Starting the movie metadata service")
    repo := memory.New()
    svc := controller.New(repo)
    h := grpchandler.New(svc)
    lis, err := net.Listen("tcp", "localhost:8081")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    srv := grpc.NewServer()
    gen.RegisterMetadataServiceServer(srv, h)
    srv.Serve(lis)
}
```

更新的`main`函数说明了我们如何实例化我们的 gRPC 服务器并在其中监听请求。函数的其余部分与之前类似。

我们已经完成了对元数据服务的更改，现在可以继续进行评分服务的开发。

## 评分服务

让我们为评分服务创建一个 gRPC 处理器。在`rating/internal/handler`包中创建一个`grpc`目录，并向其中添加一个包含以下代码的`grpc.go`文件：

```go
package grpc

import (
    "context"
    "errors"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "movieexample.com/gen"
    "movieexample.com/rating/internal/controller"
    "movieexample.com/rating/pkg/model"
)

// Handler defines a gRPC rating API handler.
type Handler struct {
    gen.UnimplementedRatingServiceServer
    svc *controller.RatingService
}

// New creates a new movie metadata gRPC handler.
func New(svc *controller.RatingService) *Handler {
    return &Handler{ctrl: ctrl}
}
```

现在，让我们实现`GetAggregatedRating`端点：

```go
// GetAggregatedRating returns the aggregated rating for a 
// record.
func (h *Handler) GetAggregatedRating(ctx context.Context, req *gen.GetAggregatedRatingRequest) (*gen.GetAggregatedRatingResponse, error) {
    if req == nil || req.RecordId == "" || req.RecordType == "" {
        return nil, status.Errorf(codes.InvalidArgument, "nil req or empty id")
    }
    v, err := h.svc.GetAggregatedRating(ctx, model.RecordID(req.RecordId), model.RecordType(req.RecordType))
    if err != nil && errors.Is(err, controller.ErrNotFound) {
        return nil, status.Errorf(codes.NotFound, err.Error())
    } else if err != nil {
        return nil, status.Errorf(codes.Internal, err.Error())
    }
    return &gen.GetAggregatedRatingResponse{RatingValue: v}, nil
}
```

最后，让我们实现`PutRating`端点：

```go
// PutRating writes a rating for a given record.
func (h *Handler) PutRating(ctx context.Context, req *gen.PutRatingRequest) (*gen.PutRatingResponse, error) {
    if req == nil || req.RecordId == "" || req.UserId == "" {
        return nil, status.Errorf(codes.InvalidArgument, "nil req or empty user id or record id")
    }
    if err := h.svc.PutRating(ctx, model.RecordID(req.RecordId), model.RecordType(req.RecordType), &model.Rating{UserID: model.UserID(req.UserId), Value: model.RatingValue(req.RatingValue)}); err != nil {
        return nil, err
    }
    return &gen.PutRatingResponse{}, nil
}
```

现在，我们准备好更新我们的`rating/cmd/main.go`文件。用以下内容替换它：

```go
package main

import (
    "log"
    "net"

    "google.golang.org/grpc"
    "movieexample.com/gen"
    "movieexample.com/rating/internal/controller"
    grpchandler "movieexample.com/rating/internal/handler/grpc"
    "movieexample.com/rating/internal/repository/memory"
)

func main() {
    log.Println("Starting the rating service")
    repo := memory.New()
    svc := controller.New(repo)
    h := grpchandler.New(svc)
    lis, err := net.Listen("tcp", "localhost:8082")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    srv := grpc.NewServer()
    gen.RegisterRatingServiceServer(srv, h)
    srv.Serve(lis)
}
```

我们启动服务的方式与元数据服务类似。现在，我们准备好将电影服务链接到元数据和评分服务。

## 电影服务

在前面的示例中，我们创建了 gRPC 服务器来处理客户端请求。现在，让我们说明如何添加调用我们的服务器的逻辑。这将帮助我们通过 gRPC 在我们的微服务之间建立通信。

首先，让我们实现一个可以在我们的服务网关中重用的函数。创建`src/internal/grpcutil`目录，并向其中添加一个名为`grpcutil.go`的文件。向其中添加以下代码：

```go
package grpcutil
import (
    "context"
    "math/rand"
    "pkg/discovery"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "movieexample.com/pkg/discovery"
)
// ServiceConnection attempts to select a random service 
// instance and returns a gRPC connection to it.
func ServiceConnection(ctx context.Context, serviceName string, registry discovery.Registry) (*grpc.ClientConn, error) {
    addrs, err := registry.ServiceAddresses(ctx, serviceName)
    if err != nil {
        return nil, err
    }
    return grpc.Dial(addrs[rand.Intn(len(addrs))], grpc.WithTransportCredentials(insecure.NewCredentials()))
}
```

我们刚刚实现的函数将尝试使用提供的服务注册表随机选择目标服务的一个实例，然后为它创建一个 gRPC 连接。

现在，让我们为我们的元数据服务创建一个网关。在`movie/internal/gateway`包中，创建一个名为`metadata`的目录。在其内部，创建一个名为`grpc`的目录，并添加一个`metadata.go`文件，其中包含以下代码：

```go
package grpc

import (
    "context"
    "google.golang.org/grpc"
    "movieexample.com/gen"
    "movieexample.com/internal/grpcutil"
    "movieexample.com/metadata/pkg/model"
    "movieexample.com/pkg/discovery"
)

// Gateway defines a movie metadata gRPC gateway.
type Gateway struct {
    registry discovery.Registry
}

// New creates a new gRPC gateway for a movie metadata 
// service.
func New(registry discovery.Registry) *Gateway {
    return &Gateway{registry}
}
```

让我们实现从远程 gRPC 服务获取元数据的函数：

```go
// Get returns movie metadata by a movie id.
func (g *Gateway) Get(ctx context.Context, id string) (*model.Metadata, error) {
    conn, err := grpcutil.ServiceConnection(ctx, "metadata", g.registry)
    if err != nil {
        return nil, err
    }
    defer conn.Close()
    client := gen.NewMetadataServiceClient(conn)
    resp, err := client.GetMetadataByID(ctx, &gen.GetMetadataByIDRequest{MovieId: id})
    if err != nil {
        return nil, err
    }
    return model.MetadataFromProto(resp.Metadata), nil
}
```

让我们突出显示我们网关实现的一些细节：

+   我们使用`grpcutil.ServiceConnection`函数创建到我们的元数据服务的连接。

+   我们使用来自`gen`包生成的客户端代码创建一个客户端。

+   我们使用`MetadataFromProto`映射函数将生成的结构转换为内部结构。

现在，我们准备好为我们的评分服务创建一个网关。在`movie/internal/gateway`包内，创建一个`rating/grpc`目录，并向其中添加一个包含以下内容的`grpc.go`文件：

```go
package grpc

import (
    "context"
    "pkg/discovery"
    "rating/pkg/model"

    "google.golang.org/grpc"
    "movieexample.com/internal/grpcutil"
    "movieexample.com/gen"
)

// Gateway defines an gRPC gateway for a rating service.
type Gateway struct {
    registry discovery.Registry
}

// New creates a new gRPC gateway for a rating service.
func New(registry discovery.Registry) *Gateway {
    return &Gateway{registry}
}
```

添加`GetAggregatedRating`函数的实现：

```go
// GetAggregatedRating returns the aggregated rating for a 
// record or ErrNotFound if there are no ratings for it.
func (g *Gateway) GetAggregatedRating(ctx context.Context, recordID model.RecordID, recordType model.RecordType) (float64, error) {
    conn, err := grpcutil.ServiceConnection(ctx, "rating", g.registry)
    if err != nil {
        return 0, err
    }
    defer conn.Close()
    client := gen.NewRatingServiceClient(conn)
    resp, err := client.GetAggregatedRating(ctx, &gen.GetAggregatedRatingRequest{RecordId: string(recordID), RecordType: string(recordType)})
    if err != nil {
        return 0, err
    }
    return resp.RatingValue, nil
}
```

到目前为止，我们已经完成了大部分更改。最后一步是更新电影服务的`main`函数。将其更改为以下内容：

```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"
    "movieexample.com/gen"
    "movieexample.com/movie/internal/controller"
    metadatagateway "movieexample.com/movie/internal/gateway/metadata/grpc"
    ratinggateway "movieexample.com/movie/internal/gateway/rating/grpc"
    grpchandler "movieexample.com/movie/internal/handler/grpc"
"movieexample.com/pkg/discovery/static"
)
func main() {
    log.Println("Starting the movie service")
    registry := static.NewRegistry(map[string][]string{
        "metadata": {"localhost:8081"},
        "rating":   {"localhost:8082"},
        "movie":    {"localhost:8083"},
    })
    ctx := context.Background()
    if err := registry.Register(ctx, "movie", "localhost:8083"); err != nil {
        panic(err)
    }
    defer registry.Deregister(ctx, "movie")
    metadataGateway := metadatagateway.New(registry)
    ratingGateway := ratinggateway.New(registry)
    svc := controller.New(ratingGateway, metadataGateway)
    h := grpchandler.New(svc)
    lis, err := net.Listen("tcp", "localhost:8083")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    srv := grpc.NewServer()
    gen.RegisterMovieServiceServer(srv, h)
    srv.Serve(lis)
}
```

您可能已经注意到格式没有改变，我们只是更新了网关的导入，将它们从 HTTP 更改为 gRPC。

我们已经完成了对服务的更改。现在，服务可以使用 Protocol Buffers 序列化相互通信，并且您可以在每个`cmd`目录内使用`go run *.go`命令运行它们。

# 摘要

在本章中，我们介绍了同步通信的基础，并学习了如何使用 Protocol Buffers 格式使微服务相互通信。我们展示了如何使用 Protocol Buffers 架构语言定义我们的服务 API，并生成可在用 Go 和其他语言编写的微服务应用程序中重用的代码。

本章所获得的知识应有助于您使用 Protocol Buffers 和 gRPC 编写和维护现有服务。它还作为了如何为您的服务使用代码生成的一个示例。在下一章中，我们将继续探索不同的通信方式，通过介绍另一个模型，即异步通信。

# 进一步阅读

+   *gRPC*: [`grpc.io`](https://grpc.io)

+   *HTTP/2 详细概述*: [`web.dev/performance-http2`](https://web.dev/performance-http2)
