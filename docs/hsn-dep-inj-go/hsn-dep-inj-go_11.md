# 控制你的热情

在本章中，我们将研究**依赖注入**（**DI**）可能出错的一些方式。

作为程序员，我们对新工具或技术的热情有时会让我们失去理智。希望本章能帮助我们保持理智，避免麻烦。

重要的是要记住，DI 是一种工具，因此应该在方便和适合的时候进行选择性应用。

本章将涵盖以下主题：

+   DI 引起的损害

+   过早的未来保护

+   模拟 HTTP 请求

+   不必要的注入？

# 技术要求

您可能还会发现阅读和运行本章的完整代码版本很有用，这些代码可以在[`github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch11`](https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch11)上找到。

# DI 引起的损害

DI 引起的损害是指使用 DI 使代码更难理解、维护或以其他方式使用的情况。

# 长构造函数参数列表

长构造函数参数列表可能是由 DI 引起的代码损害中最常见和最经常抱怨的。虽然 DI 并非代码损害的根本原因，但它确实没有帮助。

考虑以下示例，它使用构造函数注入：

```go
func NewMyHandler(logger Logger, stats Instrumentation,
   parser Parser, formatter Formatter,
   limiter RateLimiter,
   cache Cache, db Datastore) *MyHandler {

   return &MyHandler{
      // code removed
   }
}

// MyHandler does something fantastic
type MyHandler struct {
   // code removed
}

func (m *MyHandler) ServeHTTP(response http.ResponseWriter, request *http.Request) {
   // code removed
}
```

构造函数参数太多了。这使得使用、测试和维护都变得困难。那么问题的原因是什么呢？实际上有三个不同的问题。

第一个，也许是最常见的，当第一次采用 DI 时，出现错误的抽象。考虑构造函数的最后两个参数是`Cache`和`Datastore`。假设`cache`用于`datastore`的前端，而不是用于缓存`MyHandler`的输出，那么这些应该合并为不同的抽象。`MyHandler`代码不需要深入了解数据存储的位置和方式；它只需要对它需要的内容进行规定。我们应该用更通用的抽象替换这两个输入值，如下面的代码所示：

```go
// Loader is responsible for loading the data
type Loader interface {
   Load(ID int) ([]byte, error)
}
```

顺便说一句，这也是另一个包/层的绝佳位置。

第二个问题与第一个类似，违反了单一责任原则。我们的`MyHandler`承担了太多责任。它目前正在解码请求，从数据存储和/或缓存加载数据，然后呈现响应。解决这个问题的最佳方法是考虑软件的层次结构。这是顶层，我们的 HTTP 处理程序；它需要理解和使用 HTTP。因此，我们应该寻找方法让它成为其主要（也许是唯一）责任。

第三个问题是横切关注点。我们的参数包括日志记录和仪表盘依赖项，这些依赖项可能会被大多数代码使用，并且很少在少数测试之外进行更改。我们有几种处理这个问题的选择；我们可以应用配置注入，从而将它们合并为一个依赖项，并将它们与我们可能拥有的任何配置合并。或者我们可以使用**即时**（**JIT**）注入来访问全局单例。

在这种情况下，我们决定使用配置注入。应用后，我们得到以下代码：

```go
func NewMyHandler(config Config,
   parser Parser, formatter Formatter,
   limiter RateLimiter,
   loader Loader) *MyHandler {

   return &MyHandler{
      // code removed
   }
}
```

我们仍然有五个参数，这比我们开始时要好得多，但仍然相当多。

我们可以通过组合进一步减少这个问题。首先，让我们看看我们之前示例的构造函数，如下面的代码所示：

```go
func NewMyHandler(config Config,
   parser Parser, formatter Formatter,
   limiter RateLimiter,
   loader Loader) *MyHandler {

   return &MyHandler{
      config:    config,
      parser:    parser,
      formatter: formatter,
      limiter:   limiter,
      loader:    loader,
   }
}
```

从`MyHandler`作为*基本处理程序*开始，我们可以定义一个包装我们基本处理程序的新处理程序，如下面的代码所示：

```go
type FancyFormatHandler struct {
   *MyHandler
}
```

现在我们可以按以下方式为我们的`FancyFormatHandler`定义一个新的构造函数：

```go
func NewFancyFormatHandler(config Config,
   parser Parser,
   limiter RateLimiter,
   loader Loader) *FancyFormatHandler {

   return &FancyFormatHandler{
      &MyHandler{
         config:    config,
         formatter: &FancyFormatter{},
         parser:    parser,
         limiter:   limiter,
         loader:    loader,
      },
   }
}
```

就像那样，我们少了一个参数。这里真正的魔力在于匿名组合；因为这样，对`FancyFormatHandler.ServeHTTP()`的任何调用实际上都会调用`MyHandler.ServeHTTP()`。在这种情况下，我们添加了一点代码，以改进我们用户的处理程序的用户体验。

# 注入一个对象时，配置就可以了

通常情况下，你的第一反应是注入一个依赖，这样你就可以在隔离环境中测试你的代码。然而，为了这样做，你不得不引入如此多的抽象和间接性，以至于代码量和复杂性呈指数增长。

这种情况的一个普遍发生是使用通用库来访问外部资源，比如网络资源、文件或数据库。例如，让我们使用我们样本服务的`data`包。如果我们想要抽象出对`sql`包的使用，我们可能会从定义一个接口开始，如下面的代码所示：

```go
type Connection interface {
   QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
   QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
   ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
}
```

然后我们意识到`QueryRowContext()`和`QueryContext()`分别返回`*sql.Row`和`*sql.Rows`。深入研究这些结构，我们发现没有办法从`sql`包的外部填充它们的内部状态。为了解决这个问题，我们不得不定义我们自己的`Row`和`Rows`接口，如下面的代码所示：

```go
type Row interface {
   Scan(dest ...interface{}) error
}

type Rows interface {
   Scan(dest ...interface{}) error
   Close() error
   Next() bool
}

type Result interface {
   LastInsertId() (int64, error)
   RowsAffected() (int64, error)
}
```

我们现在完全与`sql`包解耦，并且能够在我们的测试中模拟它。

但让我们停下来一分钟，考虑一下我们所处的位置：

+   我们引入了大约 60 行代码，但我们还没有为它们编写任何测试

+   我们无法在不使用实际数据库的情况下测试新代码，这意味着我们永远无法完全与数据库解耦

+   我们增加了另一层抽象和一些复杂性

现在，将这与本地安装数据库并确保其处于良好状态进行比较。这里也有复杂性，但可以说是一个微不足道的一次性成本，特别是当分摊到我们所工作的所有项目时。我们还必须创建和维护数据库中的表。这个最简单的选择是一个`SQL`脚本——一个也可以用来支持实时系统的脚本。

对于我们的样本服务，我们决定维护一个`SQL`文件和一个本地安装的数据库。由于这个决定，我们不需要模拟对数据库的调用，而只需要将数据库配置传递给我们的本地数据库。

这种情况经常出现，特别是在来自可信来源的低级包中，比如标准库。解决这个问题的关键是要实事求是。问问自己，我真的需要模拟这个吗？有没有一些配置我可以传递进去，从而减少工作量？

最终，我们必须确保我们从额外的工作、代码和复杂性中获得足够的回报来证明这种努力是值得的。

# 不必要的间接性

DI 被误用的另一种方式是引入有限（或没有）目的的抽象。类似于我们之前讨论的注入配置而不是对象，这种额外的间接性导致了额外的工作、代码和复杂性。

让我们看一个例子，你可以引入一个抽象来帮助测试，但实际上并不需要。

在标准的 HTTP 库中，有一个名为`http.ServeMux`的结构体。`ServeMux`用于构建 HTTP 路由器，即 URL 和 HTTP 处理程序之间的映射。一旦`ServeMux`配置好了，它就会被传递到 HTTP 服务器中，如下面的代码所示：

```go
func TestExample(t *testing.T) {
   router := http.NewServeMux()
   router.HandleFunc("/health", func(resp http.ResponseWriter, req *http.Request) {
      _, _ = resp.Write([]byte(`OK`))
   })

   // start a server
   address := ":8080"
   go func() {
      _ = http.ListenAndServe(address, router)
   }()

   // call the server
   resp, err := http.Get("http://:8080/health")
   require.NoError(t, err)

   // validate the response
   responseBody, err := ioutil.ReadAll(resp.Body)
   assert.Equal(t, []byte(`OK`), responseBody)
}
```

随着我们的服务扩展，我们需要确保添加更多的端点。为了防止 API 回归，我们决定添加一些测试来确保我们的路由器配置正确。由于我们熟悉 DI，我们可以立即介绍一个`ServerMux`的抽象，以便我们可以添加一个模拟实现。这在下面的例子中显示：

```go
type MyMux interface {
   Handle(pattern string, handler http.Handler)
   Handler(req *http.Request) (handler http.Handler, pattern string)
   ServeHTTP(resp http.ResponseWriter, req *http.Request)
}

// build HTTP handler routing
func buildRouter(mux MyMux) {
   mux.Handle("/get", &getEndpoint{})
   mux.Handle("/list", &listEndpoint{})
   mux.Handle("/save", &saveEndpoint{})
}
```

有了我们的抽象，我们可以定义一个模拟实现`MyMux`，并编写一个测试，如下面的例子所示：

```go
func TestBuildRouter(t *testing.T) {
   // build mock
   mockRouter := &MockMyMux{}
   mockRouter.On("Handle", "/get", &getEndpoint{}).Once()
   mockRouter.On("Handle", "/list", &listEndpoint{}).Once()
   mockRouter.On("Handle", "/save", &saveEndpoint{}).Once()

   // call function
   buildRouter(mockRouter)

   // assert expectations
   assert.True(t, mockRouter.AssertExpectations(t))
}
```

这一切看起来都很好。然而，问题在于这是不必要的。我们的目标是通过测试端点和 URL 之间的映射来防止意外的 API 回归。

我们的目标可以在不模拟`ServeMux`的情况下实现。首先，让我们回到我们引入`MyMux`接口之前的原始函数，就像下面的例子所示：

```go
// build HTTP handler routing
func buildRouter(mux *http.ServeMux) {
   mux.Handle("/get", &getEndpoint{})
   mux.Handle("/list", &listEndpoint{})
   mux.Handle("/save", &saveEndpoint{})
}
```

深入了解`ServeMux`，我们可以看到，如果我们调用`Handler(req *http.Request)`方法，它将返回配置到该 URL 的`http.Handler`。

因为我们知道我们将为每个端点执行一次，所以我们应该定义一个函数来做到这一点，就像下面的例子中所示：

```go
func extractHandler(router *http.ServeMux, path string) http.Handler {
   req, _ := http.NewRequest("GET", path, nil)
   handler, _ := router.Handler(req)
   return handler
}
```

有了我们的函数，我们现在可以构建一个测试，验证每个 URL 返回预期的处理程序，就像下面的例子中所示：

```go
func TestBuildRouter(t *testing.T) {
   router := http.NewServeMux()

   // call function
   buildRouter(router)

   // assertions
   assert.IsType(t, &getEndpoint{}, extractHandler(router, "/get"))
   assert.IsType(t, &listEndpoint{}, extractHandler(router, "/list"))
   assert.IsType(t, &saveEndpoint{}, extractHandler(router, "/save"))
}
```

在前面的例子中，您可能还注意到我们的`buildRouter()`函数和我们的测试非常相似。这让我们对测试的效果产生了疑问。

在这种情况下，更有效的做法是确保我们有 API 回归测试，验证不仅路由器的配置，还有输入和输出格式，就像我们在第十章的结尾所做的那样，*现成的注入*。

# 服务定位器

首先，定义一下——服务定位器是围绕一个对象的软件设计模式，该对象充当所有依赖项的中央存储库，并能够按名称返回它们。您会发现这种模式在许多语言中使用，并且是一些 DI 框架和容器的核心。

在我们深入探讨为什么这是 DI 引起的损害之前，让我们看一个过于简化的服务定位器的例子：

```go
func NewServiceLocator() *ServiceLocator {
   return &ServiceLocator{
      deps: map[string]interface{}{},
   }
}

type ServiceLocator struct {
   deps map[string]interface{}
}

// Store or map a dependency to a key
func (s *ServiceLocator) Store(key string, dep interface{}) {
   s.deps[key] = dep
}

// Retrieve a dependency by key
func (s *ServiceLocator) Get(key string) interface{} {
   return s.deps[key]
}
```

为了使用我们的服务定位器，我们首先必须创建它，并将我们的依赖项与它们的名称进行映射，就像下面的例子所示：

```go
// build a service locator
locator := NewServiceLocator()

// load the dependency mappings
locator.Store("logger", &myLogger{})
locator.Store("converter", &myConverter{})
```

有了我们构建的服务定位器和设置的依赖项，我们现在可以传递它并根据需要提取依赖项，就像下面的代码所示：

```go
func useServiceLocator(locator *ServiceLocator) {
   // use the locators to get the logger
   logger := locator.Get("logger").(Logger)

   // use the logger
   logger.Info("Hello World!")
}
```

现在，如果我们想在测试期间*替换*日志记录器，那么我们只需要构建一个带有模拟日志记录器的新服务定位器，并将其传递给我们的函数。

那有什么问题呢？首先，我们的服务定位器现在是一个上帝对象（如第一章中提到的*永远不要停止追求更好*），我们可能最终会在各个地方传递它。只需要将一个对象传递到每个函数中听起来可能是一件好事，但这会导致第二个问题。

对象和它使用的依赖之间的关系现在完全对外部隐藏了。我们不再能够查看函数或结构定义并立即知道需要哪些依赖。

最后，我们在没有 Go 类型系统和编译器保护的情况下操作。在前面的例子中，下面的这行可能引起了你的注意：

```go
logger := locator.Get("logger").(Logger)
```

因为服务定位器接受并返回`interface{}`，每次我们需要访问一个依赖项，我们都需要转换为适当的类型。这种转换不仅使代码变得混乱，还可能在值缺失或类型错误时导致运行时崩溃。我们可以通过更多的代码解决这些问题，就像下面的例子所示：

```go
// use the locators to get the logger
loggerRetrieved := locator.Get("logger")
if loggerRetrieved == nil {
   return
}
logger, ok := loggerRetrieved.(Logger)
if !ok {
   return
}

// use the logger
logger.Info("Hello World!")
```

采用先前的方法，我们的应用程序将不再崩溃，但变得非常混乱。

# 过早的未来保护

有时，DI 的应用并不是错误的，而只是不必要的。这种常见的表现形式是过早的未来保护。过早的未来保护是指我们根据可能有一天会需要它的假设，向软件添加我们目前不需要的功能。正如你所期望的那样，这会导致不必要的工作和复杂性。

让我们借鉴我们的服务的例子来看一个例子。目前，我们有一个 Get 端点，如下面的代码所示：

```go
// GetHandler is the HTTP handler for the "Get Person" endpoint
type GetHandler struct {
   cfg    GetConfig
   getter GetModel
}

// ServeHTTP implements http.Handler
func (h *GetHandler) ServeHTTP(response http.ResponseWriter, request *http.Request) {
   // extract person id from request
   id, err := h.extractID(request)
   if err != nil {
      // output error
      response.WriteHeader(http.StatusBadRequest)
      return
   }

   // attempt get
   person, err := h.getter.Do(id)
   if err != nil {
      // not need to log here as we can expect other layers to do so
      response.WriteHeader(http.StatusNotFound)
      return
   }

   // happy path
   err = h.writeJSON(response, person)
   if err != nil {
      response.WriteHeader(http.StatusInternalServerError)
   }
}

// output the supplied person as JSON
func (h *GetHandler) writeJSON(writer io.Writer, person *get.Person) error {
   output := &getResponseFormat{
      ID:       person.ID,
      FullName: person.FullName,
      Phone:    person.Phone,
      Currency: person.Currency,
      Price:    person.Price,
   }

   return json.NewEncoder(writer).Encode(output)
}
```

这是一个简单的 REST 端点，返回 JSON。如果我们决定，有一天，我们可能想以不同的格式输出，我们可以将编码移到一个依赖项中，如下面的示例所示：

```go
// GetHandler is the HTTP handler for the "Get Person" endpoint
type GetHandler struct {
   cfg       GetConfig
   getter    GetModel
   formatter Formatter
}

// ServeHTTP implements http.Handler
func (h *GetHandler) ServeHTTP(response http.ResponseWriter, request *http.Request) {
   // no changes to this method
}

// output the supplied person
func (h *GetHandler) buildOutput(writer io.Writer, person *Person) error {
   output := &getResponseFormat{
      ID:       person.ID,
      FullName: person.FullName,
      Phone:    person.Phone,
      Currency: person.Currency,
      Price:    person.Price,
   }

   // build output payload
   payload, err := h.formatter.Marshal(output)
   if err != nil {
      return err
   }

   // write payload to response and return
   _, err = writer.Write(payload)
   return err
}
```

那段代码看起来合理。那么问题出在哪里呢？简单地说，这是我们不需要做的工作。

因此，这是我们不需要编写或维护的代码。在这个简单的例子中，我们的更改只增加了一点额外的复杂性，这是相对常见的。这种少量的额外复杂性在整个系统中的扩散会减慢我们的速度。

如果这真的成为一个实际要求，那么这绝对是交付功能的正确方式，但在那时，它是一个功能，因此是我们必须承担的负担。

# 模拟 HTTP 请求

在本章的前面，我们谈到了注入并不是所有问题的答案，在某些情况下，传递配置要高效得多，而且代码要少得多。这种情况经常发生在处理外部服务时，特别是在处理 HTTP 服务时，比如我们示例服务中的上游货币转换服务。

虽然可以模拟对外部服务的 HTTP 请求并使用模拟来彻底测试对外部服务的调用，但这并不是必要的。让我们通过使用我们示例服务的代码来比较模拟和配置的差异。

以下是我们示例服务的代码，调用外部货币转换服务：

```go
// Converter will convert the base price to the currency supplied
type Converter struct {
   cfg Config
}

// Exchange will perform the conversion
func (c *Converter) Exchange(ctx context.Context, basePrice float64, currency string) (float64, error) {
   // load rate from the external API
   response, err := c.loadRateFromServer(ctx, currency)
   if err != nil {
      return defaultPrice, err
   }

   // extract rate from response
   rate, err := c.extractRate(response, currency)
   if err != nil {
      return defaultPrice, err
   }

   // apply rate and round to 2 decimal places
   return math.Floor((basePrice/rate)*100) / 100, nil
}

// load rate from the external API
func (c *Converter) loadRateFromServer(ctx context.Context, currency string) (*http.Response, error) {
   // build the request
   url := fmt.Sprintf(urlFormat,
      c.cfg.ExchangeBaseURL(),
      c.cfg.ExchangeAPIKey(),
      currency)

   // perform request
   req, err := http.NewRequest("GET", url, nil)
   if err != nil {
      c.logger().Warn("[exchange] failed to create request. err: %s", err)
      return nil, err
   }

   // set latency budget for the upstream call
   subCtx, cancel := context.WithTimeout(ctx, 1*time.Second)
   defer cancel()

   // replace the default context with our custom one
   req = req.WithContext(subCtx)

   // perform the HTTP request
   response, err := http.DefaultClient.Do(req)
   if err != nil {
      c.logger().Warn("[exchange] failed to load. err: %s", err)
      return nil, err
   }

   if response.StatusCode != http.StatusOK {
      err = fmt.Errorf("request failed with code %d", response.StatusCode)
      c.logger().Warn("[exchange] %s", err)
      return nil, err
   }

   return response, nil
}

func (c *Converter) extractRate(response *http.Response, currency string) (float64, error) {
   defer func() {
      _ = response.Body.Close()
   }()

   // extract data from response
   data, err := c.extractResponse(response)
   if err != nil {
      return defaultPrice, err
   }

   // pull rate from response data
   rate, found := data.Quotes["USD"+currency]
   if !found {
      err = fmt.Errorf("response did not include expected currency '%s'", currency)
      c.logger().Error("[exchange] %s", err)
      return defaultPrice, err
   }

   // happy path
   return rate, nil
}
```

在我们着手撰写测试之前，我们应该首先问自己，我们想要测试什么？以下是典型的测试场景：

+   **正常路径**：外部服务器返回数据，我们成功提取数据

+   **失败/慢请求**：外部服务器返回错误或在时间上没有响应

+   **错误响应**：外部服务器返回无效的 HTTP 响应代码，表示它有问题

+   **无效响应**：外部服务器返回我们不期望的格式的有效负载

我们将通过模拟 HTTP 请求来开始我们的比较。

# 使用 DI 模拟 HTTP 请求

如果我们要使用 DI 和模拟，那么最干净的选项是模拟 HTTP 请求，以便我们可以使其返回我们需要的任何响应。

为了实现这一点，我们需要做的第一件事是抽象构建和发送 HTTP 请求，如下面的代码所示：

```go
// Requester builds and sending HTTP requests
//go:generate mockery -name=Requester -case underscore -testonly -inpkg -note @generated
type Requester interface {
   doRequest(ctx context.Context, url string) (*http.Response, error)
}
```

您可以看到，我们还包括了一个*go generate*注释，它将为我们创建模拟实现。

然后我们可以更新我们的`Converter`以使用`Requester`抽象，如下面的示例所示：

```go
// NewConverter creates and initializes the converter
func NewConverter(cfg Config, requester Requester) *Converter {
   return &Converter{
      cfg:       cfg,
      requester: requester,
   }
}

// Converter will convert the base price to the currency supplied
type Converter struct {
   cfg       Config
   requester Requester
}

// load rate from the external API
func (c *Converter) loadRateFromServer(ctx context.Context, currency string) (*http.Response, error) {
   // build the request
   url := fmt.Sprintf(urlFormat,
      c.cfg.ExchangeBaseURL(),
      c.cfg.ExchangeAPIKey(),
      currency)

   // perform request
   response, err := c.requester.doRequest(ctx, url)
   if err != nil {
      c.logger().Warn("[exchange] failed to load. err: %s", err)
      return nil, err
   }

   if response.StatusCode != http.StatusOK {
      err = fmt.Errorf("request failed with code %d", response.StatusCode)
      c.logger().Warn("[exchange] %s", err)
      return nil, err
   }

   return response, nil
}
```

有了`requester`抽象，我们可以使用模拟实现进行测试，如下面的代码所示：

```go
func TestExchange_invalidResponse(t *testing.T) {
   // build response
   response := httptest.NewRecorder()
   _, err := response.WriteString(`invalid payload`)
   require.NoError(t, err)

   // configure mock
   mockRequester := &mockRequester{}
   mockRequester.On("doRequest", mock.Anything, mock.Anything).Return(response.Result(), nil).Once()

   // inputs
   ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
   defer cancel()

   basePrice := 12.34
   currency := "AUD"

   // perform call
   converter := &Converter{
      requester: mockRequester,
      cfg:       &testConfig{},
   }
   result, resultErr := converter.Exchange(ctx, basePrice, currency)

   // validate response
   assert.Equal(t, float64(0), result)
   assert.Error(t, resultErr)
   assert.True(t, mockRequester.AssertExpectations(t))
}
```

在前面的示例中，我们的模拟请求者返回了一个无效的响应，而不是调用外部服务。通过这样做，我们可以确保我们的代码在发生这种情况时表现得恰当。

为了覆盖其他典型的测试场景，我们只需要复制这个测试，并更改模拟的响应和期望。

现在让我们将基于模拟的测试与基于配置的等效测试进行比较。

# 使用配置模拟 HTTP 请求

我们可以在不进行任何代码更改的情况下测试`Converter`。第一步是定义一个返回我们需要的响应的 HTTP 服务器。在下面的示例中，服务器返回的与前一节中的模拟相同：

```go
server := httptest.NewServer(http.HandlerFunc(func(response http.ResponseWriter, request *http.Request) {
   payload := []byte(`invalid payload`)
   response.Write(payload)
}))
```

然后我们从测试服务器获取 URL，并将其作为配置传递给`Converter`，如下面的示例所示：

```go
cfg := &testConfig{
   baseURL: server.URL,
   apiKey:  "",
}

converter := NewConverter(cfg)
```

现在，下面的示例显示了我们如何执行 HTTP 调用并验证响应，就像我们在模拟版本中所做的那样：

```go
result, resultErr := converter.Exchange(ctx, basePrice, currency)

// validate response
assert.Equal(t, float64(0), result)
assert.Error(t, resultErr)
```

通过这种方法，我们可以实现与基于模拟的版本相同的测试场景覆盖率，但代码和复杂性要少得多。或许更重要的是，我们不会因为额外的构造函数参数而导致测试引起的损害。

# 不必要的注入

到目前为止，您可能会想，“有时使用 DI 并不是最佳选择，但我怎么知道呢？”为此，我想再给您提供一个自我调查。

当您不确定如何继续，或者在进行潜在的大规模重构之前，首先快速浏览一下我的 DI 调查：

+   依赖是否是环境问题（比如日志记录）？

环境依赖是必要的，但往往会污染函数的用户体验，特别是构造函数。注入它们是合适的，但您应该更倾向于使用较不显眼的 DI 方法，比如即时注入或配置注入。

+   在重构期间是否有测试来保护我们？

在对测试覆盖率较低的现有代码应用 DI 时，添加一些猴子补丁将是您可以进行的最小更改，因此也是风险最小的更改。一旦测试就位，它将受到保护，即使这些更改意味着删除猴子补丁。

+   依赖的存在是否具有信息性？

依赖的存在告诉用户有关结构体的什么？如果答案不多或没有，那么依赖可以合并到任何配置注入中。同样，如果依赖在这个结构体的范围之外不存在，那么您可以使用即时注入来管理它。

+   你将有多少个依赖的实现？

如果答案是多于一个，那么注入依赖是正确的选择。如果答案是一个，那么您需要深入一点。依赖是否会发生变化？如果它从未发生过变化，那么注入它就是一种浪费，而且很可能增加了不必要的复杂性。

+   依赖是否在测试之外发生过变化？

如果它只在测试期间更改，那么这是一个很好的即时注入的候选项，毕竟，我们希望避免测试引起的损害。

+   依赖是否需要在每次执行时更改？

如果答案是肯定的，那么你应该使用方法注入。在可能的情况下，尽量避免向结构体添加任何决定要使用哪个依赖的逻辑（例如`switch`语句）。相反，确保您要么注入依赖并使用它，要么注入一个包含决定依赖的逻辑的工厂或定位器对象。这将确保您的结构体不会受到任何与单一职责相关的问题的影响。它还有助于我们避免在添加新的依赖实现时进行大规模的手术式变更。

+   依赖是否稳定？

稳定的依赖是已经存在的，不太可能改变（或以向后兼容的方式改变），并且不太可能被替换的东西。这方面的很好的例子是标准库和良好管理、很少更改的公共包。如果依赖是稳定的，那么为了解耦而注入它的价值就不那么大，因为代码没有改变，可以信任。

您可能希望注入一个稳定的依赖，以便测试您如何使用它，就像我们之前看到的 SQL 包和 HTTP 客户端的例子一样。然而，为了避免测试引起的损害和不必要的复杂性，我们应该要么采用即时注入，以避免污染用户体验，要么完全避免注入。

+   这个结构体将有一个还是多个用途？

如果结构体只有一个用途，那么对于代码的灵活性和可扩展性的压力就很低。因此，我们可以更倾向于少注入，更具体地实现；至少在我们的情况发生变化之前是这样。另一方面，在许多地方使用的代码将承受更大的变化压力，并且可以说更希望具有更大的灵活性，以便在更多情况下更有用。在这些情况下，您将希望更倾向于注入，以给用户更多的灵活性。只是要小心，不要注入太多，以至于函数的用户体验变得糟糕。

对于共享代码，您还应该更加努力地将代码与尽可能多的外部（不稳定的）依赖解耦。当用户采用您的代码时，他们可能不想采用您的所有依赖项。

+   **这段代码是否包装了依赖项？**

如果我们包装一个包以使其用户体验更方便，以隔离我们免受该包中的更改影响，那么注入该包是不必要的。我们编写的代码与其包装的代码紧密耦合，因此引入抽象并没有取得显著成效。

+   **应用 DI 会让代码变得更好吗？**

当然，这是非常主观的，但也可能是最关键的问题。抽象是有用的，但它也增加了间接性和复杂性。

解耦很重要，但并非总是必要的。包和层之间的解耦比包内对象之间的解耦更重要。

通过经验和重复，您会发现许多这些问题会变得自然而然，因为您会在何时应用 DI 以及使用哪种方法方面形成直觉。

与此同时，以下表格可能会有所帮助：

| ** 方法** | ** 理想用于：** |
| --- | --- |
| Monkey patching |

+   依赖于单例的代码

+   当前没有测试或现有依赖注入的代码

+   解耦包而不对依赖包做任何更改

|

| 构造函数注入 |
| --- |

+   需要的依赖

+   必须在调用任何方法之前准备好的依赖项

+   被对象的大多数或所有方法使用的依赖

+   在请求之间不会改变的依赖

+   有多个实现的依赖项

|

| 方法注入 |
| --- |

+   与函数、框架和共享库一起使用

+   请求范围的依赖

+   无状态对象

+   在请求中提供上下文或数据的依赖，因此预计在调用之间会有所变化

|

| 配置注入 |
| --- |

+   替换构造函数或方法注入以改善代码的用户体验

|

| JIT 注入 |
| --- |

+   替换本来应该注入到构造函数中的依赖项，并且只有一个生产实现。

+   在对象和全局单例或环境依赖之间提供一层间接或抽象。特别是当我们想在测试期间替换全局单例时

+   允许用户可选地提供依赖项

|

| 现成的注入 |
| --- |

+   减少采用构造函数注入的成本

+   减少创建依赖项顺序的复杂性

|

# 总结

在本章中，我们研究了不必要或不正确地应用 DI 的影响。我们还讨论了一些情况，在这些情况下，采用 DI 并不是最佳选择。

然后，我们用列出了 10 个问题来帮助您确定 DI 是否适用于您当前的用例。

在下一章中，我们将总结我们对 DI 的研究，回顾我们在整本书中讨论过的所有内容。特别是，我们将对比我们样本服务的当前状态和原始状态。我们还将简要介绍如何使用 DI 启动新服务。

# 问题

1.  你最常见到的 DI 引起的损害形式是什么？

1.  为什么重要的是不要盲目地一直应用 DI？

1.  采用 Google Wire 等框架是否可以消除 DI 引起的所有损害形式？
