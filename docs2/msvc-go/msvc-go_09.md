

# 单元和集成测试

测试是任何开发过程的重要组成部分。始终重要的是用自动化测试覆盖你的代码，确保所有重要的逻辑在所有代码更改上都能持续测试。编写好的测试通常有助于确保在整个开发过程中所做的任何更改都能保持代码的正常和可靠运行。

在微服务开发中，测试尤为重要，但它也给开发者带来了一些额外的挑战。仅仅测试每个服务是不够的——测试服务之间的集成也同样重要，确保每个服务都能与其他服务协同工作。

在本章中，我们将涵盖单元测试和集成测试，并说明如何将测试添加到我们在前几章中创建的微服务中。我们将涵盖以下主题：

+   Go 测试概述

+   单元测试

+   集成测试

+   测试最佳实践

你将学习如何在 Go 中编写单元和集成测试，如何使用模拟技术，以及如何组织微服务的测试代码。这些知识将帮助你构建更可靠的服务。

让我们继续概述 Go 测试工具和技术。

# 技术要求

要完成本章，你需要 Go 版本 1.11 或更高版本。

你可以在此处找到本章的 GitHub 代码：[`github.com/PacktPublishing/microservices-with-go/tree/main/Chapter09`](https://github.com/PacktPublishing/microservices-with-go/tree/main/Chapter09)。

# Go 测试概述

在本节中，我们将提供一个关于 Go 测试功能的概述。我们将涵盖为 Go 代码编写测试的基础，列出 Go SDK 提供的有用函数和库，并描述各种测试编写技术，这些技术将有助于你在微服务开发中。

首先，让我们来了解一下为 Go 应用程序编写测试的基础。

Go 语言内置了对编写自动化测试的支持，并提供了一个名为 `testing` 的包来实现这一目的。

Go 代码与其测试之间存在一种约定关系。如果你有一个名为 `example.go` 的文件，其测试将位于同一包中的名为 `example_test.go` 的文件中。使用 `_test` 文件名后缀可以让你区分被测试的代码及其测试，从而更容易地导航源代码。

Go 测试函数遵循这种约定名称格式，每个测试函数名称都以 `Test` 前缀开头：

```go
func TestXxx(t *testing.T)
```

在这些函数内部，你可以使用 `testing.T` 结构来报告测试失败或使用它提供的任何其他辅助函数。

让我们以这个测试为例：

```go
func TestAdd(t *testing.T) {
  a, b := 1, 2
  if got, want := Add(1, 2), 3; got != want {
    t.Errorf("Add(%v, %v) = %v, want %v", a, b, got, want)
  }
}
```

在前面的函数中，我们使用了 `testing.T` 来报告测试失败，如果 `Add` 函数提供了意外的输出。

在执行方面，我们可以运行以下命令：

```go
go test
```

该命令会执行目标目录中的每个测试，并打印输出，其中包含任何失败的测试或其他必要的数据的错误消息。

开发者可以自由选择测试的格式；然而，有一些常见的技巧，如**表驱动测试**，通常有助于优雅地组织测试代码。

表驱动测试是指将输入以表格或一系列行的形式存储的测试。让我们以这个例子为例：

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        a    int
        b    int
        want int
    }{
        {a: 1, b: 2, want: 3},
        {a: -1, b: -2, want: -3},
        {a: -3, b: 3, want: 0},
        {a: 0, b: 0, want: 0},
    }
    for _, tt := range tests {
        assert.Equal(t, tt.want, Add(tt.a, tt.b), fmt.Sprintf("Add(%v, %v)", tt.a, tt.b))
    }
}
```

在此代码中，我们使用我们的函数的测试用例初始化`tests`变量，然后遍历它。请注意，我们使用`github.com/stretchr/testify`库提供的`assert.Equal`函数来比较被测试函数的预期结果和实际结果。这个库提供了一组方便的函数，可以简化你的测试逻辑。如果不使用`assert`库，比较测试结果的代码将如下所示：

```go
        if got, want := Add(tt.a, tt.b), tt.want; got != want {
            t.Errorf ("Add(%v, %v) = %v, want %v", tt.a, tt.b, got, want)
        }
```

表驱动测试通过将测试用例和执行实际检查的逻辑分离，有助于减少测试的重复性。通常，当你需要针对定义的目标状态执行大量类似检查时，这些测试是良好的实践，如我们的示例所示。

表驱动格式还有助于提高测试代码的可读性，使得查看和比较相同函数的不同测试用例变得更加容易。这种格式在 Go 测试中相当常见；然而，你始终可以根据你的用例组织测试代码。

现在，让我们回顾 Go 内置测试库提供的基本功能。

### 子测试

Go 测试库的一个有趣特性是能够创建**子测试**——在另一个测试中执行的测试。子测试的好处之一是能够单独执行它们，以及在长时间运行的测试中并行执行它们，并以更细粒度的方式结构化测试输出。

子测试是通过调用`testing`库的`Run`函数创建的：

```go
func (t *T) Run(name string, f func(t *T)) bool
```

当使用`Run`函数时，你需要传递测试用例的名称和要执行的功能，Go 将负责单独执行每个测试用例。以下是一个使用`Run`函数的测试示例：

```go
func TestProcess(t *testing.T) {
  t.Run("test case 1", func(t *testing.T) {
    // Test case 1 logic.
  })
  t.Run("test case 2", func(t *testing.T) {
    // Test case 2 logic.
  })
}
```

在前面的例子中，我们通过两次调用`Run`函数创建了两个子测试，一次为每个子测试。

为了对子测试有更精细的控制，你可以使用以下选项：

+   当使用带有`-v`参数的`go test`命令运行测试时，每个子测试（无论是通过还是失败）都可以在输出中单独显示

+   你可以使用`go test`命令的`-run`参数运行单个测试用例

使用`Run`函数还有另一个有趣的优点。让我们想象一下，你有一个名为`Process`的函数，它需要几秒钟才能完成。如果你有一个包含大量测试用例的表格测试，并且按顺序执行它们，整个测试的执行可能需要很长时间。在这种情况下，你可以通过调用`t.Parallel()`函数让 Go 测试运行器以并行模式执行测试。以下是一个示例：

```go
func TestProcess(t *testing.T) {
    tests := []struct {
        name  string
        input string
           want  string
    }{
        {name: "empty", input: "", want: ""},
        {name: "dog", input: "animal that barks", want: "dog"},
        {name: "cat", input: "animal that meows", want: "cat"},
    }
    for _, tt := range tests {
        input := tt.input
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            assert.Equal(t, tt.want, Process(input), fmt.Sprintf("Process(%v)", input))
        })
    }
}
```

在我们的例子中，我们为每个测试用例调用 `t.Run` 函数，传递测试用例名称和要执行的功能。然后，我们调用 `t.Parallel()` 使每个测试用例并行执行。这种优化将显著减少我们的 `Process` 函数运行缓慢时的执行时间。

### 跳过

假设你希望在计算机上的每次更改后执行你的 Go 测试，但你有一些运行缓慢的测试，需要很长时间才能运行。在这种情况下，你将想要找到一种方法在特定条件下跳过运行测试。Go 测试库对此有内置支持——`Skip` 函数。让我们以这个测试函数为例：

```go
func TestProcess(t *testing.T) {
  if os.Getenv("RUNTIME_ENV") == "development" {
    t.Skip("Skipping a test in development environment")
  }
  ...
}
```

在前面的代码中，如果存在具有 `development` 值的 `RUNTIME_ENV` 运行时环境变量，则会跳过测试执行。请注意，我们在 `t.Skip` 调用中也提供了跳过的原因，以便在测试执行时记录。

跳过功能可以特别有用，用于绕过执行长时间运行的测试，例如执行缓慢的 I/O 操作或进行大量数据处理。为此，Go 测试库提供了一种将特定标志 `-test.short` 传递给 `go test` 命令的能力：

```go
go test -test.short
```

使用 `-test.short` 标志，你可以让 Go 测试运行器知道你想要以 **简短模式** 运行测试——在这种模式下，只有特殊的短测试被执行。你可以在所有长时间运行的测试中添加以下逻辑来排除它们在简短模式下的执行：

```go
func TestLongRunningProcess(t *testing.T) {
  if testing.Short() {
    t.Skip("Skipping a test in short mode")
  }
  ...
}
```

在前面的例子中，当将 `-test.short` 标志传递给 `test` 命令时，会跳过测试。

当一些测试用例比其他测试用例慢得多，并且你需要非常频繁地运行测试时，使用简短测试模式是有用的。跳过慢速测试并减少它们的执行频率可以显著提高你的开发速度，并使你的开发体验变得更好。

你可以通过查看 `testing` 包的官方文档来熟悉其他 Go 测试功能：[`pkg.go.dev/testing`](https://pkg.go.dev/testing)。我们现在将进入下一节，重点关注为我们的微服务实现单元测试的细节。

# 单元测试

我们已经涵盖了为 Go 应用程序自动化测试的许多有用功能，现在我们准备说明如何在我们的微服务代码中使用它们。首先，我们将从 **单元测试** 开始——对单个代码单元的测试，例如结构和单个函数。

让我们以元数据服务控制器为例，通过实现单元测试的过程。目前，我们的控制器文件看起来像这样：

```go
package metadata
import (
    "context"
    "movieexample.com/metadata/pkg/model"
)
type metadataRepository interface {
    Get(ctx context.Context, id string) (*model.Metadata, error)
}
// Controller defines a metadata service controller.
Type Controller struct {
    repo metadataRepository
}
// New creates a metadata service controller.
Func New(repo metadataRepository) *Controller {
    return &Controller{repo}
}
// Get returns movie metadata by id.
Func (c *Controller) Get(ctx context.Context, id string) (*model.Metadata, error) {
    return c.repo.Get(ctx, id)
}
```

让我们列出我们希望在代码中测试的内容：

+   当仓库返回 `ErrNotFound` 时进行的 `Get` 调用

+   当仓库返回除 `ErrNotFound` 之外的错误时的 `Get` 调用

+   当仓库返回元数据和没有错误时的 `Get` 调用

到目前为止，我们有三个测试案例需要实现。所有测试案例都需要对元数据仓库进行操作，我们需要模拟它从三个不同的响应。我们如何在测试中模拟元数据仓库的响应呢？让我们探索一种强大的技术，它允许我们通过测试代码实现这一点。

## 模拟

从组件模拟响应的技术称为**模拟**。模拟通常用于测试以模拟各种场景，例如返回特定的结果或错误。在 Go 代码中，有多种使用模拟的方法。第一种是手动实现组件的*假*版本，称为**模拟**。让我们以我们的元数据仓库为例说明如何实现这些模拟。我们的元数据仓库接口定义如下：

```go
type metadataRepository interface {
    Get(ctx context.Context, id string) (*model.Metadata, error)
}
```

这个接口的模拟实现可能看起来像这样：

```go
type mockMetadataRepository struct {
    returnRes *model.Metadata
    returnErr error
}
func (m *mockMetadataRepository) setReturnValues(res *model.Metadata, err error) {
    m.returnRes = res
    m.returnErr = err
}
func (m *mockMetadataRepository) Get(ctx context.Context, id string) (*model.Metadata, error) {
    return m.returnRes, m.returnErr
}
```

在我们的元数据仓库示例模拟中，我们通过提供`setReturnValues`函数允许在即将到来的`Get`函数调用中返回设置值。模拟可以用来测试我们的控制器如下：

```go
m := mockMetadataRepository{}
m.setReturnValues(nil, repository.ErrNotFound)
c := New(m)
res, err := c.Get(context.Background(), "some-id")
// Check res, err.
```

手动实现模拟是测试测试包范围之外的各个组件调用的相对简单的方法。这种方法的不利之处在于，你需要自己编写模拟代码，并在任何接口更改时更新其代码。

使用模拟的另一种方法是使用生成模拟代码的库。这类库的一个例子是[`github.com/golang/mock`](https://github.com/golang/mock)，它包含一个名为`mockgen`的模拟生成工具。你可以通过运行以下命令来安装它：

```go
go install github.com/golang/mock/mockgen
```

然后，可以使用`mockgen`工具如下所示：

```go
mockgen -source=foo.go [options]
```

让我们通过以下命令从我们项目的`src`目录生成我们元数据仓库的模拟代码：

```go
mockgen -package=repository -source=metadata/internal/controller/metadata/controller.go
```

你应该得到模拟源文件的 内容作为输出。内容将类似于以下内容：

```go
// MockmetadataRepository is a mock of metadataRepository 
// interface
type MockmetadataRepository struct {
    ctrl     *gomock.Controller
    recorder *MockmetadataRepositoryMockRecorder
}
// NewMockmetadataRepository creates a new mock instance
func NewMockmetadataRepository(ctrl *gomock.Controller) *MockmetadataRepository {
    mock := &MockmetadataRepository{ctrl: ctrl}
    mock.recorder = &MockmetadataRepositoryMockRecorder{mock}
    return mock
}
// EXPECT returns an object that allows the caller to indicate // expected use
func (m *MockmetadataRepository) EXPECT() *MockmetadataRepositoryMockRecorder {
    return m.recorder
}
// Get mocks base method.
func (m *MockmetadataRepository) Get(ctx context.Context, id string) (*model.Metadata, error) {
    ret := m.ctrl.Call(m, "Get", ctx, id)
    ret0, _ := ret[0].(*model.Metadata)
    ret1, _ := ret[1].(error)
    return ret0, ret1
}
```

生成的模拟代码实现了我们的接口，并允许我们以下方式设置对`Get`函数的预期响应：

```go
ctrl := gomock.NewController(t)
defer ctrl.Finish()
m := NewMockmetadataRepository(gomock.NewController())
ctx := context.Background()
id := "some-id"
m.EXPECT().Get(ctx, id).Return(nil, repository.ErrNotFound)
```

由`gomock`库生成的模拟代码提供了一些我们在手动创建的模拟版本中没有实现的有用功能。其中之一是使用`Times`函数设置目标函数应被调用的预期次数的能力：

```go
m.EXPECT().Get(ctx, id).Return(nil, repository.ErrNotFound).Times(1)
```

在前面的例子中，我们将`Get`函数被调用的次数限制为一次。`gomock`库在测试执行结束时验证这些约束，并报告函数是否被调用不同次数。当你想确保目标函数在测试中确实被调用时，这种机制非常有用。

到目前为止，我们已经展示了两种不同的使用模拟的方法，你可能想知道哪种方法是首选的。让我们比较这两种方法来找出答案。

手动实现模拟的好处是可以在不使用任何外部库的情况下进行，例如`gomock`。然而，这种方法的缺点如下：

+   手动实现模拟需要花费时间

+   对模拟接口的任何更改都需要手动更新模拟代码

+   实现由如`gomock`之类的库提供的额外功能（如调用计数验证）更困难

使用如`gomock`之类的库提供模拟代码将具有以下好处：

+   当所有模拟以相同方式生成时，代码一致性更高

+   无需编写样板代码

+   扩展的模拟功能集

在我们的比较中，自动模拟代码生成似乎提供了更多的优势，因此我们将遵循基于`gomock`的自动模拟生成方法。在下一节中，我们将展示如何为我们服务实现这一点。

## 实现单元测试

我们将展示如何使用生成的`gomock`代码实现控制器单元测试。首先，我们需要在我们的仓库中找到一个合适的地方来放置生成的代码。我们已经有了一个名为`gen`的目录，它被服务共享。我们可以创建一个名为`mock`的子目录，我们可以用它来存储各种生成的模拟。再次运行元数据仓库的模拟生成命令：

```go
mockgen -package=repository -source=metadata/internal/controller/metadata/controller.go
```

将其输出复制到名为`gen/mock/metadata/repository/repository.go`的文件中。现在，让我们为我们的元数据服务控制器添加一个测试。在其目录中创建一个名为`controller_test.go`的文件，并添加以下代码：

```go
package metadata
import (
    "context"
    "errors"
    "testing"
    "github.com/golang/mock/gomock"
    "github.com/stretchr/testify/assert"
    gen "movieexample.com/gen/mock/metadata/repository"
    "movieexample.com/metadata/internal/repository"
    "movieexample.com/metadata/pkg/model"
)
```

然后，添加以下代码，包含以表格格式组织的测试用例：

```go
func TestController(t *testing.T) {
    tests := []struct {
        name       string
        expRepoRes *model.Metadata
        expRepoErr error
        wantRes    *model.Metadata
        wantErr    error
    }{
        {
            name:       "not found",
            expRepoErr: repository.ErrNotFound,
            wantErr:    ErrNotFound,
        },
        {
            name:       "unexpected error",
            expRepoErr: errors.New("unexpected error"),
            wantErr:    errors.New("unexpected error"),
        },
        {
            name:       "success",
            expRepoRes: &model.Metadata{},
            wantRes:    &model.Metadata{},
        },
    }
```

最后，添加执行我们的测试的代码：

```go
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()
            repoMock := gen.NewMockmetadataRepository(ctrl)
            c := New(repoMock)
            ctx := context.Background()
            id := "id"
            repoMock.EXPECT().Get(ctx, id).Return(tt.expRepoRes, tt.expRepoErr)
            res, err := c.Get(ctx, id)
            assert.Equal(t, tt.wantRes, res, tt.name)
            assert.Equal(t, tt.wantErr, err, tt.name)
        })
    }
}
```

我们刚刚添加的代码实现了针对我们的`Get`函数的三个不同测试用例，使用了生成的仓库模拟。我们通过调用`EXPECT`函数并传递期望的值，让模拟返回特定的值。我们以表格驱动的方式组织了我们的测试，这在章节中已经描述过。

要运行测试，请使用常规命令：

```go
go test
```

如果你一切操作都正确，测试的输出应该包含`ok`。恭喜你，我们刚刚实现了单元测试并展示了如何使用模拟！我们将让你自己实现微服务的剩余测试——这将是一项相当多的工作，但这是确保代码始终经过测试和可靠的绝佳投资。

在下一节中，我们将要处理另一种类型的测试——集成测试。了解为什么以及如何为你的微服务编写集成测试，除了常规的单元测试，将帮助你编写更稳定的代码，并确保所有服务在集成时能良好工作。

# 集成测试

**集成测试**是自动化的测试，用于验证服务各个单元与自身之间的集成正确性。在本节中，您将学习如何编写集成测试以及如何在其中构建逻辑，同时获得一些有用的提示，这将帮助您在未来编写自己的集成测试。

与测试单个代码片段（如函数和结构）的单元测试不同，集成测试有助于确保各个代码片段的组合仍然能够良好地协同工作。

让我们以我们的评级服务为例，提供一个集成测试的例子。我们的服务集成测试将实例化服务实例及其客户端，并确保客户端请求会产生预期的结果。如您所记，我们的评级服务提供了两个 API 端点：

+   `PutRating`：将评级写入数据库

+   `GetAggregatedRating`：检索提供的记录（如电影）的评级并返回聚合值

我们对评级服务的集成测试可能包含以下调用序列：

+   使用`PutRating`端点写入一些数据

+   使用`GetAggregatedRating`端点验证数据

+   使用`PutRating`端点写入新的数据

+   调用`GetAggregatedRating`端点并检查聚合值是否反映了最新的评级更新

在微服务开发中，集成测试通常测试单个服务或它们的组合——开发者可以编写针对任意数量服务的测试。

与通常与被测试代码一起存在并可以访问一些内部函数、结构、常量和变量的单元测试不同，集成测试通常将测试的组件视为**黑盒**。黑盒是逻辑块，其实现细节未知，只能通过公开的 API 或用户界面访问。这种测试方式被称为**黑盒测试**——使用公共接口（如 API）测试系统，而不是调用单个内部函数或访问系统的内部组件。

微服务集成测试通常通过实例化服务实例并执行请求来执行，这些请求可以通过调用服务 API 或通过异步事件（如果系统以异步方式处理请求）进行。集成测试的结构通常遵循类似的模式：

+   **设置测试**：实例化被测试的组件和任何可以访问其接口的客户端

+   **执行测试操作并验证结果的正确性**：运行任意数量的操作，并将测试系统（如微服务）的输出与预期值进行比较

+   **清理测试**：通过清理设置中实例化的组件，优雅地终止测试，如有必要关闭任何客户端

为了说明如何编写集成测试，让我们从上一章中的三个微服务——元数据、电影和评分服务——中取三个。为了设置我们的测试，我们需要实例化六个组件——每个微服务的服务器和客户端。为了更容易运行测试，我们可以使用服务注册表和存储库的内存实现来实例化服务器。

在编写测试之前，通常很有帮助的是写下要测试的操作集以及每个步骤的预期输出。让我们写下我们的集成测试计划：

1.  使用元数据服务 API（`PutMetadata`端点）为示例电影编写元数据，并检查操作没有返回任何错误。

1.  使用元数据服务 API（`GetMetadata`端点）检索相同电影的元数据，并检查它是否与我们之前提交的记录匹配。

1.  使用电影服务 API（`GetMovieDetails`端点）获取我们示例电影的详细信息（应仅包含元数据），并确保结果与之前提交的数据匹配。

1.  使用评分服务 API（`PutRating`端点）为我们的示例电影写入第一个评分，并检查操作没有返回任何错误。

1.  使用评分服务 API（`GetAggregatedRating`端点）检索我们电影的初始聚合评分，并检查该值是否与我们之前步骤中提交的值匹配。

1.  使用评分服务 API 为我们的示例电影写入第二个评分，并检查操作没有返回任何错误。

1.  使用评分服务 API 检索我们电影的最新聚合评分，并检查该值反映了最后一个评分。

1.  获取我们示例电影的详细信息，并检查结果是否包含更新的评分。

有这样的计划使得编写集成测试的代码更容易，并带我们来到最后一步——实际实现它：

1.  创建一个名为`test/integration`的目录，并添加一个名为`main.go`的文件，其中包含以下代码：

    ```go
    package main
    ```

    ```go
    import (
    ```

    ```go
        "context"
    ```

    ```go
        "log"
    ```

    ```go
        "net"
    ```

    ```go
        "github.com/google/go-cmp/cmp"
    ```

    ```go
        "github.com/google/go-cmp/cmp/cmpopts"
    ```

    ```go
        "google.golang.org/grpc"
    ```

    ```go
        "movieexample.com/gen"
    ```

    ```go
        metadatatest "movieexample.com/metadata/pkg/testutil"
    ```

    ```go
        movietest "movieexample.com/movie/pkg/testutil"
    ```

    ```go
        "movieexample.com/pkg/discovery"
    ```

    ```go
        "movieexample.com/pkg/discovery/memory"
    ```

    ```go
        ratingtest "movieexample.com/rating/pkg/testutil"
    ```

    ```go
        "google.golang.org/grpc/credentials/insecure"
    ```

    ```go
    )
    ```

1.  让我们在文件中添加一些具有服务名称和地址的常量，我们可以在测试中稍后使用：

    ```go
    const (
    ```

    ```go
        metadataServiceName = "metadata"
    ```

    ```go
        ratingServiceName   = "rating"
    ```

    ```go
        movieServiceName    = "movie"
    ```

    ```go
        metadataServiceAddr = "localhost:8081"
    ```

    ```go
        ratingServiceAddr   = "localhost:8082"
    ```

    ```go
        movieServiceAddr    = "localhost:8083"
    ```

    ```go
    )
    ```

1.  下一步是实现设置代码以实例化我们的服务服务器：

    ```go
    func main() {
    ```

    ```go
        log.Println("Starting the integration test")
    ```

    ```go
        ctx := context.Background()
    ```

    ```go
        registry := memory.NewRegistry()
    ```

    ```go
        log.Println("Setting up service handlers and clients")
    ```

    ```go
        metadataSrv := startMetadataService(ctx, registry)
    ```

    ```go
        defer metadataSrv.GracefulStop()
    ```

    ```go
        ratingSrv := startRatingService(ctx, registry)
    ```

    ```go
        defer ratingSrv.GracefulStop()
    ```

    ```go
        movieSrv := startMovieService(ctx, registry)
    ```

    ```go
        defer movieSrv.GracefulStop()
    ```

注意对每个服务器的`GracefulStop`函数的`defer`调用——这段代码是我们测试的拆除逻辑的一部分，用于优雅地终止所有服务器。

1.  现在，让我们设置我们的服务测试客户端：

    ```go
        opts := grpc.WithTransportCredentials(insecure.NewCredentials())
    ```

    ```go
        metadataConn, err := grpc.Dial(metadataServiceAddr, opts)
    ```

    ```go
        if err != nil {
    ```

    ```go
            panic(err)
    ```

    ```go
        }
    ```

    ```go
        defer metadataConn.Close()
    ```

    ```go
        metadataClient := gen.NewMetadataServiceClient(metadataConn)
    ```

    ```go
        ratingConn, err := grpc.Dial(ratingServiceAddr, opts)
    ```

    ```go
        if err != nil {
    ```

    ```go
            panic(err)
    ```

    ```go
        }
    ```

    ```go
        defer ratingConn.Close()
    ```

    ```go
        ratingClient := gen.NewRatingServiceClient(ratingConn)
    ```

    ```go
        movieConn, err := grpc.Dial(movieServiceAddr, opts)
    ```

    ```go
        if err != nil {
    ```

    ```go
            panic(err)
    ```

    ```go
        }
    ```

    ```go
        defer movieConn.Close()
    ```

    ```go
        movieClient := gen.NewMovieServiceClient(movieConn)
    ```

现在，我们准备好实现测试命令的序列。第一步是测试、编写和读取元数据服务的操作：

```go
    log.Println("Saving test metadata via metadata service")
    m := &gen.Metadata{
        Id:          "the-movie",
        Title:       "The Movie",
        Description: "The Movie, the one and only",
        Director:    "Mr. D",
    }
    if _, err := metadataClient.PutMetadata(ctx, &gen.PutMetadataRequest{Metadata: m}); err != nil {
        log.Fatalf("put metadata: %v", err)
    }
    log.Println("Retrieving test metadata via metadata service")
    getMetadataResp, err := metadataClient.GetMetadata(ctx, &gen.GetMetadataRequest{MovieId: m.Id})
    if err != nil {
        log.Fatalf("get metadata: %v", err)
    }
    if diff := cmp.Diff(getMetadataResp.Metadata, m, cmpopts.IgnoreUnexported(gen.Metadata{})); diff != "" {
        log.Fatalf("get metadata after put mismatch: %v", diff)
    }
```

你可能注意到我们在调用`cmp.Diff`函数时使用了`cmpopts.IgnoreUnexported(gen.Metadata{})`选项——这告诉`cmp`库忽略`gen.Metadata`结构中的未导出字段。我们添加此选项是因为由 Protocol Buffers 代码生成器生成的`gen.Metadata`结构包括一些我们希望在比较中忽略的私有字段。

我们序列中的下一个测试将是检索电影详情并检查元数据是否与我们之前提交的记录匹配：

```go
    log.Println("Getting movie details via movie service")
    wantMovieDetails := &gen.MovieDetails{
        Metadata: m,
    }
    getMovieDetailsResp, err := movieClient.GetMovieDetails(ctx, &gen.GetMovieDetailsRequest{MovieId: m.Id})
    if err != nil {
        log.Fatalf("get movie details: %v", err)
    }
    if diff := cmp.Diff(getMovieDetailsResp.MovieDetails, wantMovieDetails, cmpopts.IgnoreUnexported(gen.MovieDetails{}, gen.Metadata{})); diff != "" {
        log.Fatalf("get movie details after put mismatch: %v", err)
    }
```

现在，我们准备好测试评分服务了。

让我们实现两个测试——一个用于写入评分，另一个用于检索初始聚合值，它应该与第一个评分匹配：

```go
    log.Println("Saving first rating via rating service")
    const userID = "user0"
    const recordTypeMovie = "movie"
    firstRating := int32(5)
    if _, err = ratingClient.PutRating(ctx, &gen.PutRatingRequest{
        UserId:      userID,
        RecordId:    m.Id,
        RecordType:  recordTypeMovie,
        RatingValue: firstRating,
    }); err != nil {
        log.Fatalf("put rating: %v", err)
    }
    log.Println("Retrieving initial aggregated rating via rating service")
    getAggregatedRatingResp, err := ratingClient.GetAggregatedRating(ctx, &gen.GetAggregatedRatingRequest{
        RecordId:   m.Id,
        RecordType: recordTypeMovie,
    })
    if err != nil {
        log.Fatalf("get aggreggated rating: %v", err)
    }
    if got, want := getAggregatedRatingResp.RatingValue, float64(5); got != want {
        log.Fatalf("rating mismatch: got %v want %v", got, want)
    }
```

测试的下一部分将是提交第二个评分并检查聚合值是否已更改：

```go
    log.Println("Saving second rating via rating service")
    secondRating := int32(1)
    if _, err = ratingClient.PutRating(ctx, &gen.PutRatingRequest{
        UserId:      userID,
        RecordId:    m.Id,
        RecordType:  recordTypeMovie,
        RatingValue: secondRating,
    }); err != nil {
        log.Fatalf("put rating: %v", err)
    }
    log.Println("Saving new aggregated rating via rating service")
    getAggregatedRatingResp, err = ratingClient.GetAggregatedRating(ctx, &gen.GetAggregatedRatingRequest{
        RecordId:   m.Id,
        RecordType: recordTypeMovie,
    })
    if err != nil {
        log.Fatalf("get aggreggated rating: %v", err)
    }
    wantRating := float64((firstRating + secondRating) / 2)
    if got, want := getAggregatedRatingResp.RatingValue, wantRating; got != want {
        log.Fatalf("rating mismatch: got %v want %v", got, want)
    }
```

我们几乎完成了`main`函数的实现——让我们实现最后的检查：

```go
    log.Println("Getting updated movie details via movie service")
    getMovieDetailsResp, err = movieClient.GetMovieDetails(ctx, &gen.GetMovieDetailsRequest{MovieId: m.Id})
    if err != nil {
        log.Fatalf("get movie details: %v", err)
    }
    wantMovieDetails.Rating = wantRating
    if diff := cmp.Diff(getMovieDetailsResp.MovieDetails, wantMovieDetails, cmpopts.IgnoreUnexported(gen.MovieDetails{}, gen.Metadata{})); diff != "" {
        log.Fatalf("get movie details after update mismatch: %v", err)
    }
    log.Println("Integration test execution successful")
}
```

我们的集成测试几乎准备好了。让我们在`main`函数下方添加初始化我们服务服务器的函数。首先，添加创建元数据服务服务器的函数：

```go
func startMetadataService(ctx context.Context, registry discovery.Registry) *grpc.Server {
    log.Println("Starting metadata service on " + metadataServiceAddr)
    h := metadatatest.NewTestMetadataGRPCServer()
    l, err := net.Listen("tcp", metadataServiceAddr)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    srv := grpc.NewServer()
    gen.RegisterMetadataServiceServer(srv, h)
    go func() {
        if err := srv.Serve(l); err != nil {
            panic(err)
        }
    }()
    id := discovery.GenerateInstanceID(metadataServiceName)
    if err := registry.Register(ctx, id, metadataServiceName, metadataServiceAddr); err != nil {
        panic(err)
    }
    return srv
}
```

你可能注意到我们在 goroutine 内部调用了`srv.Serve`函数——这样它就不会阻塞执行，并允许我们立即从函数返回。

让我们在同一文件中添加与评分服务服务器类似的实现：

```go
func startRatingService(ctx context.Context, registry discovery.Registry) *grpc.Server {
    log.Println("Starting rating service on " + ratingServiceAddr)
    h := ratingtest.NewTestRatingGRPCServer()
    l, err := net.Listen("tcp", ratingServiceAddr)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    srv := grpc.NewServer()
    gen.RegisterRatingServiceServer(srv, h)
    go func() {
        if err := srv.Serve(l); err != nil {
            panic(err)
        }
    }()
    id := discovery.GenerateInstanceID(ratingServiceName)
    if err := registry.Register(ctx, id, ratingServiceName, ratingServiceAddr); err != nil {
        panic(err)
    }
    return srv
}
```

最后，让我们添加一个初始化电影服务器的函数：

```go
func startMovieService(ctx context.Context, registry discovery.Registry) *grpc.Server {
    log.Println("Starting movie service on " + movieServiceAddr)
    h := movietest.NewTestMovieGRPCServer(registry)
    l, err := net.Listen("tcp", movieServiceAddr)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    srv := grpc.NewServer()
    gen.RegisterMovieServiceServer(srv, h)
    go func() {
        if err := srv.Serve(l); err != nil {
            panic(err)
        }
    }()
    id := discovery.GenerateInstanceID(movieServiceName)
    if err := registry.Register(ctx, id, movieServiceName, movieServiceAddr); err != nil {
        panic(err)
    }
    return srv
}
```

我们的集成测试已经准备好了！你可以通过执行以下命令来运行它：

```go
go run test/integration/*.go
```

如果一切正确，你应该看到以下输出：

```go
2022/07/16 16:20:46 Starting the integration test
2022/07/16 16:20:46 Setting up service handlers and clients
2022/07/16 16:20:46 Starting metadata service on localhost:8081
2022/07/16 16:20:46 Starting rating service on localhost:8082
2022/07/16 16:20:46 Starting movie service on localhost:8083
2022/07/16 16:20:46 Saving test metadata via metadata service
2022/07/16 16:20:46 Retrieving test metadata via metadata service
2022/07/16 16:20:46 Getting movie details via movie service
2022/07/16 16:20:46 Saving first rating via rating service
2022/07/16 16:20:46 Retrieving initial aggregated rating via rating service
2022/07/16 16:20:46 Saving second rating via rating service
2022/07/16 16:20:46 Saving new aggregated rating via rating service
2022/07/16 16:20:46 Getting updated movie details via movie service
2022/07/16 16:20:46 Integration test execution successful
```

如你所注意到的，我们的集成测试的结构与之前定义的测试操作序列精确匹配。我们将集成测试实现为一个可执行命令，并添加了足够的日志消息以帮助您进行调试——如果任何步骤失败，因此更容易理解失败发生在哪个步骤以及哪些操作先于该步骤。

重要的是要注意，我们在集成测试中使用了元数据和评分存储库的内存版本。另一种方法是设置一个将数据存储在某些持久数据库（如 MySQL）中的集成测试。然而，在集成测试中使用现有的持久数据库存在一些挑战：

+   集成测试数据不应干扰用户数据。否则，它可能对现有服务用户产生意外影响。

+   理想情况下，测试执行后应该清理测试数据，以免数据库被不必要的临时数据填满。

为了避免与现有用户数据发生干扰，我建议在非生产环境中运行集成测试，例如在预发布环境中。此外，我建议始终为你的测试记录生成随机标识符，以确保单个测试执行不会相互影响。例如，你可以使用[github.com/google/uuid](http://github.com/google/uuid)库通过`uuid.New()`函数生成新的标识符。最后，我建议在可能的情况下，始终在每个使用持久数据存储的集成测试结束时包含清理代码，以清理创建的记录。

现在，问题是我们在什么时候应该编写集成测试。这始终取决于你；然而，我确实有一些一般性的建议：

+   **测试关键流程**：确保你测试整个流程，例如用户注册和登录

+   **测试关键端点**：执行对你用户提供的最关键端点的测试

此外，你可能有一些在每次代码更改后执行的集成测试。像 Jenkins 这样的系统提供了这些功能，并允许你将任何自定义逻辑插入到代码的每次更新中。本书不会涵盖 Jenkins 的设置，但你可以在官方网站（https://www.jenkins.io）上熟悉其文档。

如我们所展示的如何编写单元测试和集成测试，让我们继续到本书的下一部分，描述一些 Go 测试的最佳实践。

# 测试最佳实践

在本节中，我们将列出一些额外的有用测试技巧，这些技巧将帮助你提高测试质量。

## 使用有帮助的消息

编写测试最重要的方面之一是在错误日志中提供足够的信息，以便容易理解到底发生了什么错误，以及哪个测试用例触发了失败。考虑以下测试用例代码：

```go
if got, want := Process(tt.in), tt.want; got != want {
  t.Errorf("Result mismatch")
}
```

错误日志没有包括从被测试的函数接收到的预期和实际值，这使得理解函数返回的内容以及它与预期值的不同变得更加困难。

更好的日志行应该是这样的：

```go
t.Errorf("got %v, want %v", got, want)
```

这条日志行包含了函数预期的和实际返回的值，并在调试测试时为你提供了更多的上下文。

重要提示

注意，在我们的测试日志中，首先记录实际值，然后是预期值。这种顺序是由 Go 团队推荐的测试中记录值的传统方式，并且被所有库和包遵循。在你的日志中遵循相同的顺序以保持一致性。

一个更好的错误消息如下：

```go
t.Errorf("YourFunc(%v) = %v, want %v", tt.in, got, want)
```

这个错误日志消息包括一些额外的信息——被调用的函数以及传递给它的输入参数。

为了标准化测试用例的代码，你可以使用 `github.com/stretchr/testify` 库。以下示例说明了如何比较预期值和实际值，并记录正在测试的函数名称以及传递给它的参数：

```go
assert.Equal(t, want, got, fmt.Sprintf("YourFunc(%v)", tt.in))
```

`github.com/stretchr/testify` 库的 assert 包会打印测试结果的预期值和实际值，并提供关于测试用例的详细信息（在我们的例子中是 `fmt.Sprintf` 的结果）。

## 避免在日志中使用 `Fatal`

内置的 Go 测试库包含了用于记录错误的多个函数，包括 `Error`、`Errorf`、`Fatal` 和 `Fatalf`。后两个函数会打印日志并中断测试的执行。考虑以下测试代码：

```go
if err := Process(tt.in); err != nil {
  t.Fatalf("Process(%v): %v, want nil", err)
} 
```

调用 `Fatalf` 函数会中断测试执行。中断测试执行通常不是最佳选择，因为它会导致执行的测试较少。执行较少的测试会使开发者对剩余的失败测试用例的信息更少。对于许多开发者来说，修复一个错误并重新运行所有测试可能是一个次优体验，并且尽可能继续测试执行通常更好。

以下示例可以重写如下：

```go
if err := Process(tt.in); err != nil {
  t.Errorf("Process(%v): %v, want nil", err)
} 
```

如果你在一个循环中使用此代码，你可以在 `Errorf` 调用之后添加 `continue` 以继续下一个测试用例。

## 使用 cmp 库进行比较

假设你有一个测试，它比较我们在 *第二章* 中定义的 `Metadata` 结构：

```go
want := &model.Metadata{ID: "123", Title: "Some title"}
id := "123"
if got := GetMetadata(ctx, "123"); got != want {
  t.Errorf("GetMetadata(%v): %v, want %v", id, got, want)
}
```

此代码对于结构引用将不起作用——在我们的代码中，`want` 变量持有 `model.Metadata` 结构的指针，因此即使这些结构具有相同的字段值，`!=` 操作符也会返回 `true`，前提是这些结构是分别创建的。

在 Go 中，可以使用 `reflect.DeepEqual` 函数比较结构指针：

```go
if !reflect.DeepEqual(GetMetadata(ctx, "123"), want); {
  t.Errorf("GetMetadata(%v): %v, want %v", id, *got, *want)
}
```

然而，测试的输出可能不易阅读。考虑一下，`Metadata` 结构内部有很多字段——如果只有一个字段不同，你需要扫描两个结构以找到差异。有一个方便的库可以简化测试中的比较，称为 `cmp`（https://pkg.go.dev/github.com/google/go-cmp/cmp）。

`cmp` 库允许你以与 `reflect.DeepEqual` 相同的方式比较任意的 Go 结构，但它还提供了可读性强的输出。以下是一个使用该函数的示例：

```go
if diff := cmp.Diff(want, got); diff != "" {
  t.Errorf("GetMetadata(%v): mismatch (-want +got):\n%s", tt.in, diff)
}
```

如果结构不匹配，`diff` 变量将是一个非空字符串，包括它们之间差异的可打印表示。以下是一个此类输出的示例：

```go
GetMetadata(123) mismatch (-want +got):
  model.Metadata{
      ID:      "123",
-     Tiitle: s"Title",
+     IPAddress: s"The Title",
  }
```

注意 `cmp` 库如何使用 `–` 和 `+` 前缀突出显示两个结构之间的差异。现在，阅读测试输出并注意结构之间的差异变得容易——这种优化将在调试过程中为你节省大量时间。

这总结了我们的 Go 测试最佳实践简编——您可以通过阅读*进一步阅读*部分中提到的文档来找到更多提示。请确保熟悉官方推荐和`testing`包的注释，以了解如何以传统方式编写测试并利用内置的 Go 测试库提供的所有功能。

# 摘要

在本章中，我们介绍了与 Go 测试相关的多个主题，包括 Go 测试库的常见特性和编写单元和集成测试的基础。您已经学会了如何向您的微服务添加测试，如何在各种情况下优化测试执行，创建测试模拟，并通过遵循最佳测试实践来最大化测试质量。您从阅读本章中获得的知识应该有助于提高您测试逻辑的效率，并提高您微服务的可靠性。

在下一章中，我们将转向一个新的主题，该主题将涵盖服务可靠性的主要方面，并描述各种使您的服务能够应对各种类型故障的技术。

# 进一步阅读

+   Golang 测试注释：[`github.com/golang/go/wiki/TestComments`](https://github.com/golang/go/wiki/TestComments)

+   Golang 测试包文档：[`pkg.go.dev/testing`](https://pkg.go.dev/testing)

+   *使用子测试和子基准测试*：[`go.dev/blog/subtests`](https://go.dev/blog/subtests)

# 第三部分：维护

本部分涵盖了 Go 微服务开发的一些高级主题，例如可靠性、可观察性、警报、所有权和安全。您将学习如何处理不同类型的微服务相关问题，如何收集和分析服务性能数据，如何设置自动服务事件警报，以及如何确保微服务之间的通信安全。本部分包括许多最佳实践和示例，这些示例将帮助您将新获得的知识应用到您的微服务中。

这包括以下章节：

+   *第十章*，可靠性概述

+   *第十一章*，收集服务遥测数据

+   *第十二章*，设置服务警报

+   *第十三章*，高级主题
