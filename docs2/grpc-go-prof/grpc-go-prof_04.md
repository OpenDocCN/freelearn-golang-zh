# 4

# 设置项目

正如章节标题所暗示的，我们将从头开始搭建一个 gRPC 项目。我们首先将创建我们的 Protobuf 模式，因为我们正在进行模式驱动开发。一旦模式创建完成，我们将生成 Go 代码。最后，我们将编写服务器和客户端的模板，以便我们可以在本书的后续部分重用它们。

在本章中，我们将涵盖以下主要主题：

+   常见的 gRPC 项目架构

+   从模式生成 Go 代码

+   编写可重用的服务器/客户端模板

# 前提条件

我假设你已经从上一章安装了 protoc。如果没有，现在是安装它的正确时机，因为没有它，你将无法从本章中获得太多好处。

在本章中，我将展示设置 gRPC 项目的常见方法。我将使用 protoc、Buf 和 Bazel。因此，根据你感兴趣的，你可能需要下载相应的工具。Buf 是 protoc 的一个抽象层，使我们能够更容易地运行 protoc 命令。在此基础上，它还提供了诸如 linting 和检测破坏性更改等功能。你可以从这里下载 Buf：[`docs.buf.build/installation`](https://docs.buf.build/installation)。我还会使用 Bazel 来自动从 Protobuf 生成 Go 代码以及服务器和客户端的二进制文件。如果你有兴趣使用它，你可以查看安装文档（[`github.com/bazelbuild/bazelisk#installation`](https://github.com/bazelbuild/bazelisk#installation)）。

最后，你可以在配套 GitHub 仓库的`chapter4`文件夹中找到本章的代码（[`github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter4`](https://github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter4)）。

# 创建.proto 文件定义

由于本章的目标是编写一个可以用于后续项目的模板，我们将创建一个虚拟的 proto 文件，这样我们可以测试我们的构建系统是否正常工作。这个虚拟的 proto 文件将包含一个消息和一个服务，因为我们想测试 Protobuf 和 gRPC 的代码生成。

被称为`DummyMessage`的消息将定义如下：

```go
message DummyMessage {}
```

被称为`DummyService`的服务将定义如下：

```go
service DummyService {}
```

现在，因为我们计划生成 Golang 代码，我们仍然需要定义一个名为`go_package`的选项，并将其值设置为 Go 模块名称与包含 proto 文件的子文件夹名称的连接。这个选项很重要，因为它允许我们定义生成代码应该所在的包。在我们的情况下，项目架构如下：

```go
.
├── client
│   └── go.mod
├── go.work
├── proto
│   ├── dummy
│   │   └── v1
│   │       └── dummy.proto
│   └── go.mod
└── server
    └── go.mod
```

我们有一个 monorepo（Go 工作空间），包含三个子模块：`client`、`proto`和`server`。我们通过进入每个文件夹并运行以下命令来创建每个子模块：

```go
$ go mod init github.com/github.com/PacktPublishing/
gRPC-Go-for-Professionals/
$FOLDER_NAME
```

`$FOLDER_NAME`应替换为你当前所在的文件夹名称（`client`、`proto`或`server`）。

为了使过程更快一些，我们可以创建一个命令，该命令将列出 `root` 目录中的文件夹并执行 `go` 命令。为此，您可以使用以下 UNIX（Linux/macOS）命令：

```go
$ find . -maxdepth 1 -type d -not -path . -execdir sh -c "pushd {}; go
mod init 'github.com/PacktPublishing/
gRPC-Go-for-Professionals/{}';
popd" ";"
```

如果您使用的是 Windows，您可以使用 PowerShell 运行以下命令：

```go
$ Get-ChildItem . -Name -Directory | ForEach-Object { Push-
Location $_; go mod init "github.com/PacktPublishing/
gRPC-Go-for-Professionals/$_" ;
Pop-Location }
```

之后，我们可以创建工作空间文件。我们这样做是通过进入项目的根目录（`chapter4`）并运行以下命令：

```go
$ go work init client server proto
```

现在我们有以下 `go.work`：

```go
go 1.20
use (
  ./client
  ./proto
  ./server
)
```

每个子模块都有以下 `go.mod`：

```go
module $MODULE_NAME/$SUBMODULE_NAME
go 1.20
```

在这里，我们将用 URL 替换 `$MODULE_NAME`，例如 `github.com/PacktPublishing/gRPC-Go-for-Professionals`，并用文件夹包含文件的相应名称替换 `$SUBMODULE_NAME`。在客户端的 `go.mod` 的情况下，我们将有如下内容：

```go
module github.com/PacktPublishing/gRPC-Go-for-Professionals/client
go 1.20
```

最后，我们可以通过在我们的文件中添加以下行来完成 `dummy.proto`：

```go
option go_package = "github.com/PacktPublishing/
gRPC-Go-for-Professionals/
proto/dummy/v1";
```

现在我们有以下 `dummy.proto`：

```go
syntax = "proto3";
option go_package = "github.com/PacktPublishing/
gRPC-Go-for-Professionals/proto/dummy/v1";
message DummyMessage {}
service DummyService {}
```

这就是我们测试从 `dummy.proto` 生成代码所需的所有内容。

# 生成 Go 代码

为了在生成代码所需的工具方面保持公正，我将从底层到高层介绍三种不同的工具。我们将首先看看如何使用 protoc 手动生成代码。然后，因为我们不想总是编写冗长的命令行，我们将看看如何使用 Buf 使这一生成过程变得更简单。最后，我们将看看如何使用 Bazel 将代码生成集成到我们的构建过程中。

重要提示

在本节中，我将展示编译您的 proto 文件的基本方法。大多数情况下，这些命令可以满足您的需求，但有时您可能需要查看每个工具的文档。对于 protoc，您可以通过运行 `protoc --help` 来获取选项列表。对于 Buf，您可以访问在线文档：[`docs.buf.build/installation`](https://docs.buf.build/installation)。对于 Bazel，您也可以在[`bazel.build/reference/be/protocol-buffer`](https://bazel.build/reference/be/protocol-buffer)找到在线文档。

## Protoc

使用 protoc 是从 proto 文件中手动生成代码的方法。如果您只处理几个 proto 文件且文件之间（导入）没有很多依赖关系，这种方法可能很合适。否则，正如我们将看到的，这将会相当痛苦。

然而，我仍然认为我们应该稍微学习一下 protoc 命令，以便我们了解我们可以用它做什么。此外，高级工具基于 protoc，这将帮助我们理解其不同的功能。

使用我们上一节中的 `dummy.proto` 文件，我们可以在根目录（`chapter4`）中运行 protoc，如下所示：

```go
$ protoc --go_out=. \
         --go_opt=module=github.com/PacktPublishing/
                 gRPC-Go-for-Professionals \
         --go-grpc_out=. \
         --go-grpc_opt=module=github.com/PacktPublishing/
                  gRPC-Go-for-Professionals \
         proto/dummy/v1/dummy.proto
```

现在，这可能会看起来有点吓人，实际上，这不是您能写的最短的命令之一。当我们谈到 Buf 时，我会向您展示一个更紧凑的命令。但首先，让我们将前面的命令分解成几个部分。

在讨论`--go_out`和`--go-grpc_out`之前，让我们看看`--go_opt=module`和`--go-grpc_opt=module`。这些选项在告诉 protoc 关于要被`proto`文件中`go_package`选项传递的值移除的公共模块。假设我们有以下内容：

```go
option go_package = "github.com/PacktPublishing/
gRPC-Go-for-Professionals/proto/dummy/v1";
```

然后，`-` `-go_opt=module=github.com/PacktPublishing/gRPC-Go-for-Professionals`将会从我们的`go_package`中移除`module=`后面的值，因此现在我们只有`/proto/dummy/v1`。

现在我们已经理解了这一点，我们可以进入`--go_out`和`--go-grpc_out`。这两个选项告诉 protoc 在哪里生成 Go 代码。在我们的例子中，看起来我们正在告诉 protoc 在根级别生成我们的代码，但实际上，因为它与前面两个选项结合在一起，它将在 proto 文件旁边生成代码。这是由于包的移除，导致 protoc 在`/``proto/dummy/v1`包中生成代码。

现在，你可以看到一直编写这种命令可能会多么痛苦。大多数人不会这样做。他们要么编写一个脚本来自动完成这项工作，要么使用其他工具，例如 Buf。

## Buf

对于 Buf，我们需要进行一些额外的设置来生成代码。在项目的根目录（`chapter4`）中，我们将创建一个 Buf 模块。为了做到这一点，我们可以简单地运行以下命令：

```go
$ buf mod init
```

这将创建一个名为`buf.yaml`的文件。这个文件是设置项目级别选项的地方，例如代码检查或跟踪破坏性更改。这些内容超出了本书的范围，但如果你对这个工具感兴趣，可以查看其文档（[`buf.build/docs/tutorials/getting-started-with-buf-cli/`](https://buf.build/docs/tutorials/getting-started-with-buf-cli/))）。

一旦我们有了这些，我们需要编写生成配置。在一个名为`buf.gen.yaml`的文件中，我们将有如下内容：

```go
version: v1
plugins:
  - plugin: go
    out: proto
    opt: paths=source_relative
  - plugin: go-grpc
    out: proto
    opt: paths=source_relative
```

在这里，我们正在定义 Go 插件在 Protobuf 和 gRPC 中的使用。对于每一个，我们都在说我们希望生成代码在`proto`目录下，并且我们使用另一个`--go_opt`和`--go-grpc_opt`选项来配置 protoc，其值为`paths=source_relative`。当这个设置被启用时，生成的代码将被放置在输入文件（`dummy.proto`）相同的目录中。因此，最终，Buf 正在运行类似于我们在 protoc 部分所做的事情。它正在运行以下命令：

```go
$ protoc --go_out=. \
         --go_opt=paths=source_relative \
         --go-grpc_out=. \
         --go-grpc_opt=paths=source_relative \
         proto/dummy/v1/dummy.proto
```

为了使用 Buf 运行生成，我们只需要运行以下命令（在`chapter4`目录下）：

```go
$ buf generate proto
```

使用 Buf 在中型或大型项目中相当普遍。它有助于自动化代码生成，并且易于开始使用。然而，你可能已经注意到你需要分步生成代码然后构建你的 Go 应用程序。Bazel 将帮助我们一步完成所有这些。

## Bazel

重要提示

在本节中，我将使用名为 `GO_VERSION`、`RULES_GO_VERSION`、`RULES_GO_SHA256`、`GAZELLE_VERSION`、`GAZELLE_SHA256` 和 `PROTO_VERSION` 的变量。我们没有在本节中包含这些变量，以确保书籍易于更新。您可以在 `chapter4` 文件夹中的 `versions.bzl` 文件中找到这些版本（[`github.com/PacktPublishing/gRPC-Go-for-Professionals/tree/main/chapter4`](https://github.com/PacktPublishing/Implementing-gRPC-in-Golang-Microservice/tree/main/chapter4)）。

Bazel 的设置稍微复杂一些，但值得付出努力。一旦您的构建系统启动并运行，您将能够通过一条命令构建整个应用程序（生成和构建）以及/或运行它。

在 Bazel 中，我们首先在根级别定义一个名为 `WORKSPACE.bazel` 的文件。在这个文件中，我们定义了我们项目的所有依赖项。在我们的例子中，我们依赖于 Protobuf 和 Go。除此之外，我们还将添加一个对 Gazelle 的依赖，这将帮助我们创建生成代码所需的 `BUILD.bazel` 文件。

因此，在 `WORKSPACE.bazel` 中，在定义其他任何内容之前，我们将定义我们的工作区名称，导入我们的版本变量，并导入一些用于克隆 Git 仓库和下载存档的实用工具：

```go
workspace(name = "github_com_packtpublishing_grpc_go_for_
professionals")
load("//:versions.bzl",
  "GO_VERSION",
  "RULES_GO_VERSION",
  "RULES_GO_SHA256",
  "GAZELLE_VERSION",
  "GAZELLE_SHA256",
  "PROTO_VERSION"
)
load("@bazel_tools//tools/build_defs/repo:http.bzl",
  "http_archive")
load("@bazel_tools//tools/build_defs/repo:git.bzl",
  "git_repository")
```

然后，我们将定义 Gazelle 的依赖项：

```go
http_archive(
  name = "bazel_gazelle",
  sha256 = GAZELLE_SHA256,
  urls = [
    "https://mirror.bazel.build/github.com/bazelbuild/
    bazel-gazelle/releases/download/%s/bazel-gazelle-
    %s.tar.gz" % (GAZELLE_VERSION, GAZELLE_VERSION),
    "https://github.com/bazelbuild/bazel-gazelle/
    releases/download/%s/bazel-gazelle-%s.tar.gz" %
    (GAZELLE_VERSION, GAZELLE_VERSION),
  ],
)
```

接着，我们需要拉取构建 Go 二进制文件、应用程序等的依赖项：

```go
http_archive(
  name = "io_bazel_rules_go",
  sha256 = RULES_GO_SHA256,
  urls = [
    "https://mirror.bazel.build/github.com/bazelbuild/
    rules_go/releases/download/%s/rules_go-%s.zip" %
    (RULES_GO_VERSION, RULES_GO_VERSION),
    "https://github.com/bazelbuild/rules_go/releases/
    download/%s/rules_go-%s.zip" % (RULES_GO_VERSION,
    RULES_GO_VERSION),
  ],
)
```

现在我们有了这个，我们可以拉取 `rules_go` 的依赖项，设置构建 Go 项目的工具链，并告诉 Gazelle 我们 `WORKSPACE.bazel` 文件的位置：

```go
load("@io_bazel_rules_go//go:deps.bzl",
  "go_register_toolchains", "go_rules_dependencies")
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")
go_rules_dependencies()
go_register_toolchains(version = GO_VERSION)
gazelle_dependencies(go_repository_default_config =
  "//:WORKSPACE.bazel")
```

最后，我们拉取 Protobuf 的依赖并加载其依赖项：

```go
git_repository(
  name = "com_google_protobuf",
  tag = PROTO_VERSION,
  remote = "https://github.com/protocolbuffers/protobuf"
)
load("@com_google_protobuf//:protobuf_deps.bzl",
  "protobuf_deps")
protobuf_deps()
```

我们的 `WORKSPACE.bazel` 文件已经完成。

现在，让我们转到根级别的 `BUILD.bazel` 文件。在这个文件中，我们将定义运行 Gazelle 的命令，并将让 Gazelle 知道 Go 模块名称，以及我们希望它不要考虑 `proto` 目录中的 Go 文件。我们这样做是因为否则，Gazelle 会认为 `proto` 目录中的 Go 文件也应该有自己的 Bazel 目标文件，这可能会在以后造成问题：

```go
load("@bazel_gazelle//:def.bzl", "gazelle")
# gazelle:exclude proto/**/*.go
# gazelle:prefix github.com/PacktPublishing/gRPC-Go-for-Professionals
gazelle(name = "gazelle")
```

注意

有了这些，我们现在可以运行以下命令：

```go
$ bazel run //:gazelle
```

如果您通过 Bazelisk 安装了 Bazel，每次您运行 bazel 命令时，Bazel 都会尝试获取其最新版本。为了避免这种情况，您可以创建一个名为 .bazelversion 的文件，包含您当前安装的 bazel 版本。您可以通过输入以下命令找到版本：

**bazel --version**

一个例子可以在 chapter4 文件夹中找到。

在拉取依赖并编译完成后，您应该能够在 `proto/dummy/v1` 目录中看到一个生成的 `BUILD.bazel` 文件。此文件最重要的部分是以下 `go_library`：

```go
go_library(
  name = "dummy",
  embed = [":v1_go_proto"],
  importpath = "github.com/PacktPublishing/gRPC-Go-for-
    Professionals/proto/dummy/v1",
  visibility = ["//visibility:public"],
)
```

之后，我们将使用这个库并将其链接到我们的二进制文件中。它包含我们开始所需的全部生成代码。

# 服务器模板

我们的建设系统已经准备好了。现在我们可以专注于代码。但在进入之前，让我们定义我们想要的东西。在本节中，我们想要构建一个 gRPC 服务器的模板，我们可以将其用于后续章节，甚至用于书外的后续项目。为此，有一些事情我们想要避免：

+   实现细节，如服务实现

+   特定的连接选项

+   将 IP 地址设置为常量

我们可以通过不再关心生成的代码来解决这些问题。它只是为了测试我们的构建系统。然后，我们将默认使用不安全的连接进行测试。最后，我们将 IP 地址作为程序的参数：

让我们一步一步来做：

1.  我们首先需要将 gRPC 依赖项添加到`server/go.mod`中。因此，在`server`目录中，我们可以输入以下命令：

    ```go
    $ go get google.golang.org/grpc
    ```

1.  然后，我们将获取程序传递的第一个参数，如果没有传递参数，则返回一个用法信息：

    ```go
    args := os.Args[1:]
    ```

    ```go
    if len(args) == 0 {
    ```

    ```go
      log.Fatalln("usage: server [IP_ADDR]")
    ```

    ```go
    }
    ```

    ```go
    addr := args[0]
    ```

1.  之后，我们需要监听传入的连接。我们可以使用 Go 提供的`net.Listen`来实现。这个监听器在程序结束时需要被关闭。这可能是当用户终止它或服务器失败时。显然，如果在构造监听器期间发生错误，我们只想让程序失败并通知用户：

    ```go
    lis, err := net.Listen("tcp", addr)
    ```

    ```go
    if err != nil {
    ```

    ```go
      log.Fatalf("failed to listen: %v\n", err)
    ```

    ```go
    }
    ```

    ```go
    defer func(lis net.Listener) {
    ```

    ```go
      if err := lis.Close(); err != nil {
    ```

    ```go
        log.Fatalf("unexpected error: %v", err)
    ```

    ```go
      }
    ```

    ```go
    }(lis)
    ```

    ```go
    log.Printf("listening at %s\n", addr)
    ```

1.  现在，有了所有这些，我们可以开始创建一个`grpc.Server`。我们首先需要定义一些连接选项。由于这是一个未来项目的模板，我们将保持选项为空。有了这个`grpc.ServerOption`对象数组，我们可以创建一个新的 gRPC 服务器。这是我们稍后用于注册端点的服务器。之后，我们将在某个时候关闭服务器，因此我们使用`defer`语句来实现。最后，我们在创建的`grpc.Server`上调用名为`Serve`的函数。它接受监听器作为参数。它可能会失败，所以如果有错误，我们将将其返回给客户端：

    ```go
    opts := []grpc.ServerOption{}
    ```

    ```go
    s := grpc.NewServer(opts...)
    ```

    ```go
    //registration of endpoints
    ```

    ```go
    defer s.Stop()
    ```

    ```go
    if err := s.Serve(lis); err != nil {
    ```

    ```go
      log.Fatalf("failed to serve: %v\n", err)
    ```

    ```go
    }
    ```

最后，我们有以下`main`函数（`server`/`main.go`）：

```go
package main
import (
  "log"
  "net"
  "os"
  "google.golang.org/grpc"
)
func main() {
  args := os.Args[1:]
  if len(args) == 0 {
    log.Fatalln("usage: server [IP_ADDR]")
  }
  addr := args[0]
  lis, err := net.Listen("tcp", addr)
  if err != nil {
    log.Fatalf("failed to listen: %v\n", err)
  }
  defer func(lis net.Listener) {
    if err := lis.Close(); err != nil {
      log.Fatalf("unexpected error: %v", err)
    }
  }(lis)
  log.Printf("listening at %s\n", addr)
  opts := []grpc.ServerOption{}
  s := grpc.NewServer(opts...)
  //registration of endpoints
  defer s.Stop()
  if err := s.Serve(lis); err != nil {
    log.Fatalf("failed to serve: %v\n", err)
  }
}
```

现在，我们可以通过在`server/main.go`上运行`go run`命令来运行我们的服务器。我们可以使用*Ctrl* + *C*来终止执行：

```go
$ go run server/main.go 0.0.0.0:50051
listening at 0.0.0.0:50051
```

## Bazel

如果您想使用 Bazel，您需要几个额外的步骤。第一步是更新根目录中的`BUILD.bazel`。在那里，我们将使用一个 Gazelle 命令来检测我们项目所需的所有依赖项，并将它们放入一个名为`deps.bzl`的文件中。因此，在`gazelle`命令之后，我们只需添加以下内容：

```go
gazelle(
  name = "gazelle-update-repos",
  args = [
    "-from_file=go.work",
    "-to_macro=deps.bzl%go_dependencies",
    "-prune",
  ],
  command = "update-repos",
)
```

现在，我们可以运行以下命令：

```go
$ bazel run //:gazelle-update-repos
```

在检测完`server`模块的所有依赖项后，它将创建一个`deps.bzl`文件并将其链接到我们的`WORKSPACE.bazel`中。您应该在工作空间文件中包含以下行：

```go
load("//:deps.bzl", "go_dependencies")
# gazelle:repository_macro deps.bzl%go_dependencies
go_dependencies()
```

最后，我们可以重新运行`gazelle`命令以确保它为我们的服务器创建`BUILD.bazel`文件。我们运行以下命令：

```go
$ bazel run //:gazelle
```

然后，我们在`server`目录中获取我们的`BUILD.bazel`文件。需要注意的是，在这个文件中，我们可以看到 Bazel 将 gRPC 链接到了`server_lib`。我们应该有类似以下的内容：

```go
go_library(
  name = "server_lib",
  srcs = ["main.go"],
  deps = [
    "@org_golang_google_grpc//:go_default_library",
  ],
  #...
)
```

我们现在可以用与`go` `run`命令相同的方式运行我们的服务器：

注意

这个命令将拉取 Protobuf 并构建它。最近的 Protobuf 版本需要用 C++14 或更高版本构建。你可以告诉 bazel 在名为`.bazelrc`的文件中自动指定构建 Protobuf 的 C++版本。为了保持本章的版本独立性，我们建议你检查 GitHub 仓库中的 chapter4 目录下的`.bazelrc`文件。你可以将文件复制粘贴到你的项目文件夹中。

```go
$ bazel run //server:server 0.0.0.0:50051
listening at 0.0.0.0:50051
```

因此，服务器部分我们已经完成。这是一个简单的模板，将使我们能够在下一章中轻松地创建新的服务器。它正在监听指定的端口并等待一些请求。现在，为了使执行此类请求变得容易，让我们为客户端创建一个模板。

# 客户端模板

现在我们来编写客户端模板。这将会非常类似于编写服务器模板，但不同的是，我们不是在 IP 和端口上创建监听器，而是要调用`grpc.Dial`函数，并将连接选项传递给它。

再次强调，我们不会硬编码要连接的地址。我们将将其作为一个参数来处理：

```go
args := os.Args[1:]
if len(args) == 0 {
  log.Fatalln("usage: client [IP_ADDR]")
}
addr := args[0]
```

之后，我们将创建一个`DialOption`实例，为了使这个模板通用，我们将使用`insecure.NewCredentials()`函数与服务器建立一个不安全的连接。不过，不用担心；我们稍后会讨论如何建立安全连接：

```go
opts := []grpc.DialOption{
  grpc.WithTransportCredentials(insecure.NewCredentials()),
}
```

最后，我们只需调用`grpc.Dial`函数来创建一个`grpc.ClientConn`对象。这是我们稍后需要用来调用 API 端点的对象。最后，这是一个连接对象，所以在我们客户端的生命周期结束时，我们将关闭它：

```go
conn, err := grpc.Dial(addr, opts...)
if err != nil {
  log.Fatalf("did not connect: %v", err)
}
defer func(conn *grpc.ClientConn) {
  if err := conn.Close(); err != nil {
    log.Fatalf("unexpected error: %v", err)
  }
}(conn)
```

对于客户端来说，这就差不多了。完整的代码如下（`client/main.go`）：

```go
package main
import (
  "log"
  "os"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials/insecure"
)
func main() {
  args := os.Args[1:]
  if len(args) == 0 {
    log.Fatalln("usage: client [IP_ADDR]")
  }
  addr := args[0]
  opts := []grpc.DialOption{
    grpc.WithTransportCredentials(insecure.NewCredentials()),
  }
  conn, err := grpc.Dial(addr, opts...)
  if err != nil {
    log.Fatalf("did not connect: %v", err)
  }
  defer func(conn *grpc.ClientConn) {
    if err := conn.Close(); err != nil {
      log.Fatalf("unexpected error: %v", err)
    }
  }(conn)
}
```

显然，目前它什么都没做；然而，我们可以通过先运行我们的服务器来测试它：

```go
$ go run server/main.go 0.0.0.0:50051
```

然后，我们运行我们的客户端：

```go
$ go run client/main.go 0.0.0.0:50051
```

服务器应该无限期地等待，而客户端应该在终端上无错误地返回。如果是这种情况，你就准备好编写一些 gRPC API 端点。

## Bazel

这次，Bazel 的设置不会像服务器那样长。这主要是因为我们已经有了一个`deps.bzl`文件，我们可以为客户端重用它。我们只需要使用 Gazelle 生成我们的`BUILD.bazel`文件，然后我们就完成了：

```go
$ bazel run //:gazelle
```

我们现在应该在`client`目录中有一个`BUILD.bazel`文件。需要注意的是，在这个文件中，我们可以看到 Bazel 将 gRPC 链接到了`client_lib`。我们应该有类似以下的内容：

```go
go_library(
  name = "client_lib",
  srcs = ["main.go"],
  deps = [
    "@org_golang_google_grpc//:go_default_library",
    "@org_golang_google_grpc//credentials/insecure",
  ],
  #...
)
```

我们现在可以用与`go` `run`命令相同的方式运行我们的客户端：

```go
$ bazel run //client:client 0.0.0.0:50051
```

现在我们已经有了服务器和客户端。到目前为止，它们什么也不做，但这正是预期的目的。在这本书的稍后部分，通过简单地复制它们，我们就能专注于最重要的部分，即 API。但在做任何那之前，让我们快速看一下服务器和客户端设置的一些最重要的选项。

# 服务器和拨号选项

我们简要提到了 `ServerOption` 和 `DialOption`，使用了 `grpc.ServerOption` 对象和 `grpc.WithTransportCredentials` 函数。然而，还有很多其他选项可以选择。为了可读性，我不会详细介绍每一个，但我想要展示一些你可能需要使用的主要选项。所有 `ServerOptions` 都可以在 `grpc-go` 仓库的 `server.go` 文件中找到（[`github.com/grpc/grpc-go/blob/master/server.go`](https://github.com/grpc/grpc-go/blob/master/server.go)），而 `DialOptions` 在 `dialoptions.go` 文件中（[`github.com/grpc/grpc-go/blob/master/dialoptions.go`](https://github.com/grpc/grpc-go/blob/master/dialoptions.go)）。

## grpc.Creds

这是一个选项，在服务器和客户端双方，当我们谈论保护 API 时都会使用。目前，我们看到我们可以使用 `grpc.WithTransportCredentials` 与 `insecure.NewCredentials` 的结果一起调用，这给我们提供了一个不安全的连接。这意味着请求和响应都没有加密；任何人都可以拦截这些消息并读取它们。

`grpc.Creds` 允许我们提供一个 `TransportCredentials` 对象实例，这是一个适用于所有受支持传输安全协议的通用接口，例如 TLS 和 SSL。如果我们有一个名为 `server.crt` 的证书文件和一个名为 `server.pem` 的密钥文件，我们可以创建以下 `ServerOption`：

```go
certFile := "server.crt"
keyFile := "server.pem"
creds, err := credentials.NewServerTLSFromFile(certFile, keyFile)
if err != nil {
  log.Fatalf("failed loading certificates: %v\n", err)
}
opts = append(opts, grpc.Creds(creds))
```

同样，在客户端，我们会有一个 `DialOptions` 来与服务器通信：

```go
certFile := "ca.crt"
creds, err := credentials.NewClientTLSFromFile(
  certFile, "")
if err != nil {
  log.Fatalf("error while loading CA trust certificate:
    %v\n", err)
}
opts = append(opts, grpc.WithTransportCredentials(creds))
```

到目前为止，你不必太担心这一点。正如我提到的，我们稍后会使用它，并看到如何获取证书。

## grpc.*Interceptor

如果你不太熟悉拦截器，这些是在处理请求（服务器端）之前或之后（客户端）调用的代码片段。通常的目标是向请求添加一些额外信息，但它也可以用来记录请求或拒绝某些请求，例如，如果它们没有正确的头信息。

我们将在稍后看到如何定义拦截器，但想象一下，我们有一段代码记录我们的请求，另一段代码检查授权头是否已设置。我们可以在服务器端像这样链式调用这些拦截器：

```go
 opts = append(opts, grpc.ChainUnaryInterceptor
  (LogInterceptor(), CheckHeaderInterceptor()))
```

注意，这些拦截器的顺序很重要，因为它们将按照 `grpc.ChainUnaryInterceptor` 函数中提供的顺序被调用。

对于客户端，我们可以有相同类型的日志拦截器，以及另一个添加带有缓存令牌值的授权头部的拦截器。这将给出以下内容：

```go
opts = append(opts, grpc.WithChainUnaryInterceptor
  (LogInterceptor(), AddHeaderInterceptor()))
```

最后，请注意，你可以使用其他函数来添加这些拦截器。以下是一些其他函数：

+   使用 `WithUnaryInterceptor` 来设置一个单一 RPC 拦截器

+   使用 `WithStreamInterceptor` 来设置一个流 RPC 拦截器

+   使用 `WithChainStreamInterceptor` 来链式连接多个流 RPC 拦截器

我们看到了两个重要的选项，这两个选项可以在服务器和客户端两侧进行配置。通过使用凭证，我们可以保护通信参与者之间的通信安全，通过使用拦截器，我们可以在发送或接收请求之前运行任意代码。显然，我们只看到了两个选项，而每一侧都有更多选项。如果你对查看所有这些选项感兴趣，我邀请你访问本节开头链接的 GitHub 仓库。

# 摘要

在本章中，我们为未来的服务器和客户端创建了模板。目标是编写样板代码并设置我们的构建，以便我们可以生成代码并运行我们的 Go 应用程序。

我们看到我们可以手动使用 protoc 生成 Go 代码并将其与我们的应用程序一起使用。然后我们看到我们可以通过使用 Buf 生成代码来使这个过程更加顺畅。最后，我们看到我们可以使用 Bazel 在单步中生成我们的代码并运行我们的应用程序。

最后，我们看到了我们可以使用多个 `ServerOptions` 和 `DialOptions` 来调整服务器和客户端。我们主要看了 `grpc.Creds` 和拦截器，但还有更多选项可以在 `grpc-go` 仓库中检查。

在下一章中，我们将看到如何编写 gRPC 提供的每种类型的 API。我们将从单一 API 开始，然后检查服务器和客户端流 API，最后，我们将了解如何编写双向流端点。

# 测验

1.  手动使用 protoc 的优势是什么？

    1.  无需设置；你只需要安装 protoc。

    1.  更短的生成命令。

    1.  我们可以同时生成 Go 代码并运行应用程序。

1.  使用 Buf 的优势是什么？

    1.  无需设置；你只需要安装 protoc。

    1.  更短的生成命令。

    1.  我们可以同时生成 Go 代码并运行应用程序。

1.  使用 Bazel 的优势是什么？

    1.  无需设置；你只需要安装 protoc。

    1.  更短的生成命令。

    1.  我们可以同时生成 Go 代码并运行应用程序。

1.  什么是拦截器？

    1.  一段外部代码，它拦截通信的有效负载

    1.  在服务器处理程序中运行的代码片段

    1.  在处理或发送请求之前或之后运行的代码片段

# 答案

1.  A

1.  B

1.  C

1.  C
