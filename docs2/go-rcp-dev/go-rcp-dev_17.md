# 17

# 测试、基准测试和性能分析

为你的代码编写测试和基准测试将帮助你以多种方式。在开发过程中，测试确保你正在开发的功能正常工作，并且在开发工作中不会破坏现有功能。基准测试确保你的程序保持在某些资源和时间限制内。在开发完成后，相同的测试和基准测试将确保任何维护工作（错误修复、功能增强等）不会在现有功能中引入错误。因此，你应该将编写测试和基准测试视为核心开发活动，并一起开发你的程序及其测试。

测试应关注在一切正常时测试预期的行为（“正常路径测试”）以及当事情失败时的行为。它不应专注于测试所有可能的执行路径。旨在测试所有可能实现选择的测试很快就会比程序本身更难维护。你应该在实用性和测试覆盖率之间找到平衡。

本节展示了处理几个常见测试和基准测试场景的惯用方法。这些是本章涵盖的主题：

+   与单元测试一起工作

+   编写单元测试

+   运行单元测试

+   测试中的日志记录

+   跳过测试

+   测试 HTTP 服务器

+   测试 HTTP 处理器

+   检查测试覆盖率

+   基准测试

+   编写基准测试

+   编写具有不同输入大小的多个基准测试

+   运行基准测试

+   性能分析

# 与单元测试一起工作

我们将处理一个示例函数，该函数按升序或降序对`time.Time`值进行排序，如下所示：

```go
package sort
import (
  "sort"
  "time"
)
// Sort times in ascending or descending order
func SortTimes(input []time.Time, asc bool) []time.Time {
  output := make([]time.Time, len(input))
  copy(output, input)
  if asc {
    sort.Slice(output, func(i, j int) bool {
      return output[i].Before(output[j])
    })
    return output
  }
  sort.Slice(output, func(i, j int) bool {
    return output[j].Before(output[i])
  })
  return output
}
```

我们将使用 Go 构建系统和标准库提供的内置测试工具。为此，假设我们将前面的函数存储在一个名为`sort.go`的文件中。那么，这个函数的单元测试将在这个`sort.go`文件所在的目录中的`sort_test.go`文件中。Go 构建系统将识别以`_test.go`结尾的源文件为单元测试，并将它们排除在常规构建之外。

# 编写一个单元测试

单元测试理想情况下测试单个单元（一个函数、一组相互关联的函数或类型的成员函数）是否按预期行为。

## 如何做...

1.  创建具有`_test.go`后缀的单元测试文件。对于`sort.go`，我们创建`sort_test.go`。以`_test.go`结尾的文件将不会被常规构建包含：

    ```go
    package sort
    ```

小贴士

你也可以在以`_test`结尾的单独测试包中编写测试。在这个例子中，它变成了`package sort_test`。在单独的包中编写测试允许你从外部测试包中的函数，因为你将无法访问正在测试的包中的未导出名称。你将不得不导入正在测试的包。

1.  Go 测试系统将运行遵循`Test<Feature>(*testing.T)`模式的函数。声明一个符合此模式的测试函数，并编写一个测试该行为的单元测试：

    ```go
    func TestSortTimesAscending(t *testing.T) {
        // 2.a Prepare input data
        input := []time.Time{
            time.Date(2023, 2, 1, 12, 8, 37, 0, time.Local),
            time.Date(2021, 5, 6, 9, 48, 11, 0, time.Local),
            time.Date(2022, 11, 13, 17, 13, 54, 0, time.Local),
            time.Date(2022, 6, 23, 22, 29, 28, 0, time.Local),
            time.Date(2023, 3, 17, 4, 5, 9, 0, time.Local),
        }
        // 2.b Call the function under test
        output := SortTimes(input, true)
        // 2.c Make sure the output is what is expected
        for i := 1; i < len(output); i++ {
            if !output[i-1].Before(output[i]) {
                t.Error("Wrong order")
            }
        }
    }
    ```

1.  测试函数的布局通常遵循以下结构：

    +   准备输入数据和测试函数运行所需的任何必要环境

    +   使用必要的输入调用测试函数

    +   确保测试函数返回了正确的结果或表现如预期。

1.  如果测试检测到错误，使用 `t.Error` 系列函数通知测试系统测试失败。

# 运行单元测试

使用 Go 构建系统工具运行单元测试。

## 如何操作...

1.  要运行当前包中的所有单元测试，输入以下内容：

    ```go
    go test
    PASS
    ok  github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort    0.001s
    ```

1.  要运行包中的所有单元测试，输入以下内容：

    ```go
    go test <packageName>
    go test ./<folder>
    go test github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort
    ```

    或者，您可以输入以下内容：

    ```go
    go test ./sorting
    ```

1.  要递归地运行模块中所有包的所有单元测试，输入以下内容：

    ```go
    go test ./...
    ```

    从模块的根目录执行此操作。

1.  要运行当前包中的单个测试，输入以下内容：

    ```go
    go test -run TestSortTimesAscending
    go test -run ^TestSortTimesAscending$
    ```

    在这里，`^` 表示正则表达式中的字符串开头符号，`$` 表示字符串结尾符号。

    例如，以下将运行所有以 `Ascending` 结尾的测试：

    ```go
    go test -run Ascending$
    ```

# 测试中的日志记录

通常，对于测试来说，额外的日志功能很有用，可以显示关键变量的状态，尤其是在发生失败时。默认情况下，如果测试通过，Go 测试执行器不会打印任何日志信息，但如果测试失败，则日志信息也会包含在输出中。

## 如何操作...

1.  使用 `testing.T.Log` 和 `testing.T.Logf` 函数在测试中记录日志信息：

    ```go
    func TestSortTimeAscending(t *testing.T) {
      ...
      t.Logf("Input: %v",input)
      output:=SortTimes(input,true)
      t.Logf("Output: %v", output)
    ```

1.  运行测试。如果测试通过，则不会打印日志信息。如果测试失败，则打印日志。

    要使用日志运行测试，使用 `-v` 标志：

    ```go
    $ go test -v
    === RUN   TestSortTimesAscending
        sort_test.go:17: Input: [2023-02-01 12:08:37 -0700 MST 2021-05-06 09:48:11 -0600 MDT 2022-11-13 17:13:54 -0700 MST 2022-06-23 22:29:28 -0600 MDT 2023-03-17 04:05:09 -0600 MDT]
        sort_test.go:19: Output: [2021-05-06 09:48:11 -0600 MDT 2022-06-23 22:29:28 -0600 MDT 2022-11-13 17:13:54 -0700 MST 2023-02-01 12:08:37 -0700 MST 2023-03-17 04:05:09 -0600 MDT]
    --- PASS: TestSortTimesAscending (0.00s)
    ```

# 跳过测试

您可以根据输入标志跳过某些测试。此功能允许您快速测试，其中只运行测试子集，以及全面测试，其中运行所有测试。

## 如何操作...

1.  检查 `testing.Short()` 标志以确定应从短测试运行中排除的测试：

    ```go
    func TestService(t *testing.T) {
      if testing.Short() {
        t.Skip("Service")
      }
      ...
    }
    ```

1.  使用 `test.short` 标志运行测试：

    ```go
    $ go test -test.short -v
    === RUN   TestService
        service_test.go:15: Service
    --- SKIP: TestService (0.00s)
    === RUN   TestHandler
    --- PASS: TestHandler (0.00s)
    PASS
    ```

# 测试 HTTP 服务器

`net/http/httptest` 包通过提供快速创建测试 HTTP 服务器的设施来补充 `testing` 包。

对于本节，假设我们通过将其转换为 HTTP 服务来扩展我们的排序函数，如下所示：

```go
package service
import (
    "encoding/json"
    "io"
    "net/http"
    "time"
    "github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/
    sorting/sort"
)
// Common handler function for parsing the input, sorting, and 
// preparing the output
func HandleSort(w http.ResponseWriter, req *http.Request, ascending bool) {
    var input []time.Time
    data, err := io.ReadAll(req.Body)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    if err := json.Unmarshal(data, &input); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    output := sort.SortTimes(input, ascending)
    data, err = json.Marshal(output)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    w.Write(data)
}
// Prepares a multiplexer that handles POST /sort/asc and POST /sort/
// desc endpoints
func GetServeMux() *http.ServeMux {
    mux := http.NewServeMux()
    mux.HandleFunc("POST /sort/asc", func(w http.ResponseWriter, req 
    *http.Request) {
        HandleSort(w, req, true)
    })
    mux.HandleFunc("POST /sort/desc", func(w http.ResponseWriter, req 
    *http.Request) {
        HandleSort(w, req, false)
    })
    return mux
}
```

`GetServeMux` 函数准备一个请求多路复用器，该复用器处理 `POST /sort/asc` 和 `POST /sort/desc` HTTP 端点，分别用于升序和降序排序请求。输入是时间值的 JSON 数组。处理程序返回排序后的 JSON 数组。

## 如何操作...

1.  使用包含对测试服务器支持的 `net/http/httptest` 包：

    ```go
    import (
      "net/http/httptest"
      "testing"
      ...
    )
    ```

1.  在测试函数中，创建一个处理程序或多路复用器，并使用它来创建测试服务器。确保测试结束时服务器关闭 -- 使用 `defer server.Close()`：

    ```go
    func TestService(t *testing.T) {
      mux := GetServeMux()
      server := httptest.NewServer(mux)
      defer server.Close()
    ```

1.  使用 `server.URL` 调用服务器。这是由 `httptest.NewServer` 函数初始化的，使用未分配的本地端口。在以下示例中，我们向服务器发送无效输入以验证服务器是否返回错误：

    ```go
    rsp, err := http.Post(server.URL+"/sort/asc", "application/json", strings.NewReader("test"))
    if err != nil {
      t.Error(err)
      return
    }
    // Must return http error
    if rsp.StatusCode/100 == 2 {
      t.Errorf("Error was expected")
      return
    }
    ```

    注意，`http.Post` 函数不会返回错误。`http.Post` 的错误意味着 `POST` 操作失败。在这种情况下，`POST` 操作是成功的，但返回了 HTTP 错误状态。

1.  你可以向服务器发出多个调用以测试不同的输入并检查输出：

    ```go
    data, err := json.Marshal([]time.Time{
      time.Date(2023, 2, 1, 12, 8, 37, 0, time.Local),
      time.Date(2021, 5, 6, 9, 48, 11, 0, time.Local),
      time.Date(2022, 11, 13, 17, 13, 54, 0, time.Local),
      time.Date(2022, 6, 23, 22, 29, 28, 0, time.Local),
      time.Date(2023, 3, 17, 4, 5, 9, 0, time.Local),
    ))
    if err != nil {
      t.Error(err)
      return
    }
    rsp, err = http.Post(server.URL+"/sort/asc", "application/json", bytes.NewReader(data))
    if err != nil {
      t.Error(err)
      return
    }
    defer rsp.Body.Close()
    if rsp.StatusCode != 200 {
      t.Errorf("Expected status code 200, got %d", rsp.StatusCode)
      return
    }
    var output []time.Time
    if err := json.NewDecoder(rsp.Body).Decode(&output); err != nil {
      t.Error(err)
      return
    }
    for i := 1; i < len(output); i++ {
      if !output[i-1].Before(output[i]) {
        t.Errorf("Wrong order")
      }
    }
    ```

# 测试 HTTP 处理器

`net/http/httptest` 包还包含 `ResponseRecorder`，它可以作为 `http.ResponseWriter` 用于 HTTP 处理器，以测试单个处理器而不创建服务器。

## 如何做...

1.  创建 `ResponseRecorder`：

    ```go
    func TestHandler(t *testing.T) {
      w := httptest.NewRecorder()
    ```

1.  调用处理器，传递响应记录器而不是 `http.ResponseWriter`：

    ```go
    data, err := json.Marshal([]time.Time{
      time.Date(2023, 2, 1, 12, 8, 37, 0, time.Local),
      time.Date(2021, 5, 6, 9, 48, 11, 0, time.Local),
      time.Date(2022, 11, 13, 17, 13, 54, 0, time.Local),
      time.Date(2022, 6, 23, 22, 29, 28, 0, time.Local),
      time.Date(2023, 3, 17, 4, 5, 9, 0, time.Local),
    })
    if err != nil {
      t.Error(err)
      return
    }
    req, _ := http.NewRequest("POST", "localhost/sort/asc", bytes.NewReader(data))
    req.Header.Set("Content-Type", "application/json")
    HandleSort(w, req, true)
    ```

1.  响应记录器存储由处理器构建的 HTTP 响应。验证响应是否正确：

    ```go
    if w.Result().StatusCode != 200 {
      t.Errorf("Expecting HTTP 200, got %d", w.Result().StatusCode)
      return
    }
    var output []time.Time
    if err := json.NewDecoder(w.Result().Body).Decode(&output); err != nil {
      t.Error(err)
      return
    }
    for i := 1; i < len(output); i++ {
      if !output[i-1].Before(output[i]) {
        t.Errorf("Wrong order")
      }
    }
    ```

# 检查测试覆盖率

测试覆盖率报告显示哪些源代码行被测试覆盖。

## 如何做...

1.  要快速获取覆盖率结果，使用 `cover` 标志运行测试：

    ```go
    $ go test -cover
    PASS
    coverage: 76.2% of statements
    ```

1.  要将测试覆盖率配置文件写入到单独的文件中，以便你可以获取详细的报告，给测试运行指定一个覆盖率配置文件名：

    ```go
    $ go test -coverprofile=cover.out
    PASS
    coverage: 76.2% of statements
    $ go tool cover -html=cover.out
    ```

    此命令打开浏览器并允许你看到哪些行被测试覆盖。

# 基准测试

单元测试检查正确性，而基准测试检查性能和内存使用。

# 编写基准测试

与单元测试类似，基准测试存储在 `_test.go` 文件中，但这些函数以 `Benchmark` 开头而不是 `Test`。基准测试给定一个数字 `N`，其中你重复相同的操作 `N` 次同时运行时测量性能。

## 如何做...

1.  在 `_test.go` 文件中创建一个基准测试函数。以下示例在 `sort_test.go` 文件中：

    ```go
    func BenchmarkSortAscending(b *testing.B) {
    ```

1.  在基准测试循环之前进行设置，否则你将基准测试设置代码以及实际算法：

    ```go
    input := []time.Time{
      time.Date(2023, 2, 1, 12, 8, 37, 0, time.Local),
      time.Date(2021, 5, 6, 9, 48, 11, 0, time.Local),
      time.Date(2022, 11, 13, 17, 13, 54, 0, time.Local),
      time.Date(2022, 6, 23, 22, 29, 28, 0, time.Local),
      time.Date(2023, 3, 17, 4, 5, 9, 0, time.Local),
    }
    ```

1.  编写一个迭代 `b.N` 次的 `for` 循环并执行将被基准测试的操作：

    ```go
    for i := 0; i < b.N; i++ {
      SortTimes(input, true)
    }
    ```

小贴士

在基准测试循环中避免记录或打印数据。

# 编写具有不同输入大小的多个基准测试

你通常想看到你的算法在不同输入大小下的行为。Go 测试框架只提供了基准测试应该运行多少次，而不是使用什么输入大小。使用以下模式来练习不同的输入大小。

## 如何做...

1.  定义一个未导出的参数化基准测试函数，该函数接受输入大小信息或不同大小的输入。以下示例获取项目数量和排序方向作为参数，并在执行基准测试之前创建一个给定大小的随机打乱输入切片：

    ```go
    func benchmarkSort(b *testing.B, nItems int, asc bool) {
        input := make([]time.Time, nItems)
        t := time.Now().UnixNano()
        for i := 0; i < nItems; i++ {
            input[i] = time.Unix(0, t-int64(i))
        }
        rand.Shuffle(len(input), func(i, j int) { input[i], input[j] 
        = input[j], input[i] })
        for i := 0; i < b.N; i++ {
            SortTimes(input, asc)
        }
    }
    ```

1.  通过调用公共基准测试并使用不同的值来定义导出的基准测试函数：

    ```go
    func BenchmarkSort1000Ascending(b *testing.B)  { benchmarkSort(b, 1000, true) }
    func BenchmarkSort100Ascending(b *testing.B)   { benchmarkSort(b, 100, true) }
    func BenchmarkSort10Ascending(b *testing.B)    { benchmarkSort(b, 10, true) }
    func BenchmarkSort1000Descending(b *testing.B) { benchmarkSort(b, 1000, false) }
    func BenchmarkSort100Descending(b *testing.B)  { benchmarkSort(b, 100, false) }
    func BenchmarkSort10Descending(b *testing.B)   { benchmarkSort(b, 10, false) }
    ```

# 运行基准测试

Go 工具在运行基准测试之前运行单元测试——对失败的代码进行基准测试没有意义。

## 如何操作...

1.  使用`go test -bench=<regexp>`工具。要运行所有基准测试，请使用以下命令：

    ```go
    go test -bench=.
    ```

1.  如果你想运行基准测试的子集，请输入一个基准正则表达式。以下命令仅运行名称中包含`1000`的基准测试：

    ```go
    go test -bench=1000
    goos: linux
    goarch: amd64
    pkg: github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort
    cpu: AMD Ryzen 5 7530U with Radeon Graphics
    BenchmarkSort1000Ascending-12             9753        105997 ns/op
    BenchmarkSort1000Descending-12             9813        105192 ns/op
    PASS
    ```

# 分析

分析器通过采样运行中的程序来查找在特定函数中花费的时间。你可以分析基准测试，创建配置文件，然后检查该配置文件以找到程序中的瓶颈。

## 如何操作…

要获取 CPU 配置文件并进行分析，请按照以下步骤操作：

1.  使用`cpuprofile`标志运行基准测试：

    ```go
    $ go test -bench=1000Ascending --cpuprofile=profile
    goos: linux
    goarch: amd64
    pkg: github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort
    cpu: AMD Ryzen 5 7530U with Radeon Graphics
    BenchmarkSort1000Ascending-12           10000        106509 ns/op
    ```

1.  使用配置文件启动`pprof`工具：

    ```go
    $ go tool pprof profile
    File: sort.test
    Type: cpu
    ```

1.  使用`topN`命令查看配置文件中的前`N`个样本：

    ```go
    (pprof) top5
    Showing nodes accounting for 780ms, 71.56% of 1090ms total
    Showing top 5 nodes out of 47
          flat  flat%   sum%        cum   cum%
         250ms 22.94% 22.94%      360ms 33.03%  github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort.SortTimes.func1
         230ms 21.10% 44.04%      620ms 56.88%  sort.partition_func
         120ms 11.01% 55.05%      120ms 11.01%  runtime.memmove
          90ms  8.26% 63.30%      340ms 31.19%  internal/
          reflectlite.Swapper.func9
          90ms  8.26% 71.56%      230ms 21.10%  internal/
          reflectlite.typedmemmove
    ```

    这表明大部分时间都花在比较两个时间值的匿名函数中。`flat`列显示了在函数中花费的时间，不包括由该函数调用的函数所花费的时间。`cum`（累积）包括在函数中花费的时间，定义为函数返回的时间点减去函数开始运行的时间点。也就是说，累积值包括该函数调用的函数所花费的时间。例如`sort.partition_func`运行了`620ms`，但其中只有`230ms`是在`sort.partition_func`中花费的，其余时间是在`sort.partition_func`调用的函数中花费的。

1.  使用`web`命令查看调用图和每个函数花费的时间的视觉表示。

要获取内存配置文件并进行分析，请按照以下步骤操作：

1.  使用`memprofile`标志运行基准测试：

    ```go
    $ go test -bench=1000Ascending --memprofile=mem
    goos: linux
    goarch: amd64
    pkg: github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort
    cpu: AMD Ryzen 5 7530U with Radeon Graphics
    BenchmarkSort1000Ascending-12           10000        106509 ns/op
    ```

1.  使用配置文件启动`pprof`工具：

    ```go
    $ go tool pprof mem
    File: sort.test
    Type: alloc_space
    ```

1.  使用`topN`命令查看配置文件中的前`N`个样本：

    ```go
    pprof) top5
    Showing nodes accounting for 493.37MB, 99.90% of 493.87MB total
    Dropped 2 nodes (cum <= 2.47MB)
          flat  flat%   sum%        cum   cum%
      492.86MB 99.80% 99.80%   493.36MB 99.90%  github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort.SortTimes
        0.51MB   0.1% 99.90%   493.87MB   100%  github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort.benchmarkSort
             0     0% 99.90%   493.87MB   100%  github.com/PacktPublishing/Go-Recipes-for-Developers/src/chp17/sorting/sort.BenchmarkSort1000Ascending
             0     0% 99.90%   493.87MB   100%  testing.(*B).launch
             0     0% 99.90%   493.87MB   100%  testing.(*B).runN
    ```

    与 CPU 配置文件输出类似，此表显示了每个函数分配了多少内存。同样，`flat`指的是仅在该函数中分配的内存，而`cum`指的是在该函数及其调用的任何函数中分配的内存。在这里，你可以看到`sort.SortTimes`是分配最多内存的函数。这是因为它首先创建切片的副本，然后对其进行排序。

1.  使用`web`命令查看内存分配的视觉表示。

## 参见

+   Go 程序分析的权威指南可在[`go.dev/blog/pprof`](https://go.dev/blog/pprof)找到

+   `pprof`的 README 文件解释了节点和边的表示：[`github.com/google/pprof/blob/main/doc/README.md`](https://github.com/google/pprof/blob/main/doc/README.md)
