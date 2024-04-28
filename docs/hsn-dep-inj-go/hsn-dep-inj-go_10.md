# 现成注入

在本节的最后一章中，我们将使用框架来进行**依赖注入**（**DI**）。选择与您首选风格相匹配的 DI 框架可以显著地简化您的生活。即使您不喜欢使用框架，研究它的实现方式和方法也可能会有所帮助，并帮助您找到改进您首选实现的方法。

虽然有许多可用的框架，包括 Facebook 的 Inject（[`github.com/facebookgo/inject`](https://github.com/facebookgo/inject)）和 Uber 的 Dig（[`godoc.org/go.uber.org/dig`](https://godoc.org/go.uber.org/dig)），但对于我们的示例服务，我们将使用 Google 的 Go Cloud Wire（[`github.com/google/go-cloud/tree/master/wire`](https://github.com/google/go-cloud/tree/master/wire)）。

本章将涵盖以下主题：

+   使用 Wire 进行现成的注入

+   现成注入的优点

+   应用现成的注入

+   现成注入的缺点

# 技术要求

熟悉我们在第四章中介绍的服务代码将是有益的，*ACME 注册服务简介*。本章还假设您已经阅读了第六章，*构造函数注入的依赖注入*。

您可能还会发现阅读和运行本章的完整代码版本对您有用，该代码版本可在[`github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch10`](https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch10)上找到。

获取代码并配置示例服务的说明在此处的 README 中可用：[`github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/`](https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/)。

您可以在`ch10/acme`中找到我们的服务代码，并已应用本章的更改。

# 使用 Wire 进行现成的注入

Go Cloud 项目是一个旨在使应用程序开发人员能够轻松在任何组合的云提供商上部署云应用程序的倡议。该项目的重要部分是基于代码生成的依赖注入工具**Wire**。

Wire 非常适合我们的示例服务，因为它提倡显式实例化，并且不鼓励使用全局变量；正如我们在之前的章节中尝试实现的那样。此外，Wire 使用代码生成来避免由于运行时反射而导致的性能损失或代码复杂性。

对我们来说，Wire 最有用的方面可能是其简单性。一旦我们理解了一些简单的概念，我们需要编写的代码和生成的代码就会相当简单。

# 引入提供者

文档将提供者定义如下：

“*可以生成值的函数*。”

对于我们的目的，我们可以换一种方式说，提供者返回一个依赖项的实例。

提供者可以采用的最简单形式是*简单的无参数函数*，如下面的代码所示：

```go
// Provider
func ProvideFetcher() *Fetcher {
   return &Fetcher{}
}

// Object being "provided"
type Fetcher struct {
}

func (f *Fetcher) GoFetch() (string, error) {
```

```go
return "", errors.New("not implemented yet")
}
```

提供者还可以通过具有以下参数的方式指示它们需要注入依赖项：

```go
func ProvideFetcher(cache *Cache) *Fetcher {
   return &Fetcher{
      cache: cache,
   }
}
```

此提供者的依赖项（参数）必须由其他提供者提供。

提供者还可以通过返回错误来指示可能无法初始化，如下面的代码所示：

```go
func ProvideCache() (*Cache, error) {
   cache := &Cache{}

   err := cache.Start()
   if err != nil {
      return nil, err
   }

   return cache, nil
}
```

重要的是要注意，当提供者返回错误时，使用提供的依赖项的任何注入器也必须返回错误。

# 理解注入器

Wire 中的第二个概念是注入器。注入器是魔术发生的地方。它们是我们（开发人员）定义的函数，Wire 将其用作代码生成的基础。

例如，如果我们想要一个函数，可以创建我们服务的 REST 服务器的实例，包括初始化和注入所有必需的依赖关系，我们可以通过以下函数实现：

```go
func initializeServer() (*rest.Server, error) {
 wire.Build(wireSet)
 return nil, nil
}
```

这可能对于这样一个简单的函数来说感觉很大，尤其是因为它似乎没有做任何事情（即 `返回 nil, nil`）。但这就是我们需要写的全部；代码生成器将把它转换成以下内容：

```go
func initializeServer() (*rest.Server, error) {
   configConfig, err := config.Load()
   if err != nil {
      return nil, err
   }
   getter := get.NewGetter(configConfig)
   lister := list.NewLister(configConfig)
   converter := exchange.NewConverter(configConfig)
   registerer := register.NewRegisterer(configConfig, converter)
   server := rest.New(configConfig, getter, lister, registerer)
   return server, nil
}
```

我们将在 *应用* 部分更详细地讨论这一点，但现在有三个上述函数的特点要记住。首先，生成器不关心函数的实现，除了函数必须包含一个 `wire.Build(wireSet)` 调用。其次，函数必须返回我们计划使用的具体类型。最后，如果我们依赖于任何返回错误的提供者，那么注入器也必须返回一个错误。

# 采用提供者集

在使用 Wire 时，我们需要了解的最后一个概念是提供者集。提供者集提供了一种将提供者分组的方法，在编写注入器时可以很有帮助。它们的使用是可选的；例如，之前我们使用了一个名为 `wireSet` 的提供者集，如下面的代码所示：

```go
func initializeServer() (*rest.Server, error) {
   wire.Build(wireSet)
   return nil, nil
}
```

然而，我们可以像下面的代码所示，单独传递所有的提供者：

```go
func initializeServer() (*rest.Server, error) {
   wire.Build(
      // *config.Config
      config.Load,

      // *exchange.Converter
      wire.Bind(new(exchange.Config), &config.Config{}),
      exchange.NewConverter,

      // *get.Getter
      wire.Bind(new(get.Config), &config.Config{}),
      get.NewGetter,

      // *list.Lister
      wire.Bind(new(list.Config), &config.Config{}),
      list.NewLister,

      // *register.Registerer
      wire.Bind(new(register.Config), &config.Config{}),
      wire.Bind(new(register.Exchanger), &exchange.Converter{}),
      register.NewRegisterer,

      // *rest.Server
      wire.Bind(new(rest.Config), &config.Config{}),
      wire.Bind(new(rest.GetModel), &get.Getter{}),
      wire.Bind(new(rest.ListModel), &list.Lister{}),
      wire.Bind(new(rest.RegisterModel), &register.Registerer{}),
      rest.New,
   )

   return nil, nil
}
```

遗憾的是，前面的例子并不是虚构的。它来自我们的小例子服务。

正如你所期望的，Wire 中还有很多更多的功能，但在这一点上，我们已经涵盖了足够让我们开始的内容。

# 现成注入的优势

虽然到目前为止在本章中我们一直在讨论 Wire，但我想花点时间讨论现成注入的优势。在评估工具或框架时，审视它可能具有的优势、劣势和对代码的影响是至关重要的。

现成注入的一些可能优势包括以下。

**减少样板代码**—将构造函数注入应用到程序后，`main()` 函数通常会因对象的实例化而变得臃肿。随着项目的增长，`main()` 也会增长。虽然这不会影响程序的性能，但维护起来会变得不方便。

许多依赖注入框架的目标要么是删除这些代码，要么是将其移动到其他地方。正如我们将看到的，这是在采用 Google Wire 之前我们示例服务的 `main()`：

```go
func main() {
   // bind stop channel to context
   ctx := context.Background()

   // build the exchanger
   exchanger := exchange.NewConverter(config.App)

   // build model layer
   getModel := get.NewGetter(config.App)
   listModel := list.NewLister(config.App)
   registerModel := register.NewRegisterer(config.App, exchanger)

   // start REST server
   server := rest.New(config.App, getModel, listModel, registerModel)
   server.Listen(ctx.Done())
}
```

这是在采用 Google Wire 之后的 `main()`：

```go
func main() {
   // bind stop channel to context
   ctx := context.Background()

   // start REST server
   server, err := initializeServer()
   if err != nil {
      os.Exit(-1)
   }

   server.Listen(ctx.Done())
}

```

所有相关的对象创建都被简化为这样：

```go
func initializeServer() (*rest.Server, error) {
   wire.Build(wireSet)
   return nil, nil
}
```

因为 Wire 是一个代码生成器，实际上我们最终会得到更多的代码，但其中更少的代码是由我们编写或维护的。同样，如果我们使用另一个名为 **Dig** 的流行 DI 框架，`main()` 将变成这样：

```go
func main() {
   // bind stop channel to context
   ctx := context.Background()

   // build DIG container
   container := BuildContainer()

   // start REST server
   err := container.Invoke(func(server *rest.Server) {
      server.Listen(ctx.Done())
   })

   if err != nil {
      os.Exit(-1)
   }
}
```

正如你所看到的，我们在代码上获得了类似的减少。

**自动实例化顺序**—与前面的观点类似，随着项目的增长，依赖项必须创建的顺序复杂性也会增加。因此，现成注入框架提供的许多 *魔法* 都集中在消除这种复杂性上。在 Wire 和 Dig 的两种情况下，提供者明确定义它们的直接依赖关系，并忽略它们的依赖项的任何要求。

考虑以下示例。假设我们有一个像这样的 HTTP 处理程序：

```go
func NewGetPersonHandler(model *GetPersonModel) *GetPersonHandler {
   return &GetPersonHandler{
      model: model,
   }
}

type GetPersonHandler struct {
   model *GetPersonModel
}

func (g *GetPersonHandler) ServeHTTP(response http.ResponseWriter, request *http.Request) {
   response.WriteHeader(http.StatusInternalServerError)
   response.Write([]byte(`not implemented yet`))
}
```

正如你所看到的，处理程序依赖于一个模型，看起来像下面的代码所示：

```go
func NewGetPersonModel(db *sql.DB) *GetPersonModel {
   return &GetPersonModel{
      db: db,
   }
}

type GetPersonModel struct {
   db *sql.DB
}

func (g *GetPersonModel) LoadByID(ID int) (*Person, error) {
   return nil, errors.New("not implemented yet")
}

type Person struct {
   Name string
}
```

这个模型依赖于 `*sql.DB`。然而，当我们为我们的处理程序定义提供者时，它只定义了它需要 `*GetPersonModel`，并不知道 `*sql.DB`，就像这样：

```go
func ProvideHandler(model *GetPersonModel) *GetPersonHandler {
   return &GetPersonHandler{
      model: model,
   }
}
```

与创建数据库、将其注入模型，然后将模型注入处理程序的替代方案相比，这样做更简单，无论是在编写还是在维护上。

**有人已经为你考虑过了**——也许一个好的 DI 框架可以提供的最不明显但最重要的优势是其创建者的知识。创建和维护一个框架的行为绝对不是一个微不足道的练习，它教给了它的作者比大多数程序员需要知道的更多关于 DI 的知识。这种知识通常会导致框架中出现微妙但有用的特性。例如，在 Dig 框架中，默认情况下，所有依赖关系都是单例的。这种设计选择导致了性能和资源使用的改进，以及更可预测的依赖关系生命周期。

# 应用现成的注入

正如我在前一节中提到的，通过采用 Wire，我们希望在`main()`中看到代码和复杂性显著减少。我们也希望能够基本上忘记依赖关系的实例化顺序，让框架来为我们处理。

# 采用 Google Wire

然而，我们需要做的第一件事是整理好我们的房子。大多数，如果不是全部，我们要让 Wire 处理的对象都使用我们的`*config.Config`对象，目前它存在为全局单例，如下面的代码所示：

```go
// App is the application config
var App *Config

// Load returns the config loaded from environment
func init() {
   filename, found := os.LookupEnv(DefaultEnvVar)
   if !found {
      logging.L.Error("failed to locate file specified by %s", DefaultEnvVar)
      return
   }

   _ = load(filename)
}

func load(filename string) error {
   App = &Config{}
   bytes, err := ioutil.ReadFile(filename)
   if err != nil {
      logging.L.Error("failed to read config file. err: %s", err)
      return err
   }

   err = json.Unmarshal(bytes, App)
   if err != nil {
      logging.L.Error("failed to parse config file. err : %s", err)
      return err
   }

   return nil
}
```

为了将其改为 Wire 可以使用的形式，我们需要删除全局实例，并将配置加载更改为一个函数，而不是由`init()`触发。

快速查看我们的全局单例的用法后，可以看到只有`main()`和`config`包中的一些测试引用了这个单例。由于我们之前的所有工作，这个改变将会非常简单。重构后的配置加载器如下：

```go
// Load returns the config loaded from environment
func Load() (*Config, error) {
   filename, found := os.LookupEnv(DefaultEnvVar)
   if !found {
      err := fmt.Errorf("failed to locate file specified by %s", DefaultEnvVar)
      logging.L.Error(err.Error())
      return nil, err
   }

   cfg, err := load(filename)
   if err != nil {
      logging.L.Error("failed to load config with err %s", err)
      return nil, err
   }

   return cfg, nil
}
```

这是我们更新后的`main()`：

```go
func main() {
   // bind stop channel to context
   ctx := context.Background()

   // load config
   cfg, err := config.Load(config.DefaultEnvVar)
   if err != nil {
      os.Exit(-1)
   }

   // build the exchanger
   exchanger := exchange.NewConverter(cfg)

   // build model layer
   getModel := get.NewGetter(cfg)
   listModel := list.NewLister(cfg)
   registerModel := register.NewRegisterer(cfg, exchanger)

   // start REST server
   server := rest.New(cfg, getModel, listModel, registerModel)
   server.Listen(ctx.Done())
}
```

现在我们已经移除了配置全局变量，我们准备开始采用 Google Wire。

我们将首先添加一个新文件；我们将其命名为`wire.go`。它可以被称为任何东西，但我们需要一个单独的文件，因为我们将使用 Go 构建标签来将我们在这个文件中编写的代码与 Wire 生成的版本分开。

如果你不熟悉构建标签，在 Go 中它们是文件顶部的注释，在`package`语句之前，形式如下：

```go
//+build myTag

package main

```

这些标签告诉编译器何时包含或不包含文件在编译期间。例如，前面提到的标签告诉编译器仅在触发构建时包含此文件，就像这样：

```go
$ go build -tags myTag
```

我们还可以使用构建标签来做相反的事情，使一个文件只在未指定标签时包含，就像这样：

```go
//+build !myTag

package main

```

回到`wire.go`，在这个文件中，我们将定义一个用于配置的注入器，它使用我们的配置加载器作为提供者，如下所示：

```go
//+build wireinject

package main

import (
   "github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/ch10/acme/internal/config"
   "github.com/google/go-cloud/wire"
)

// The build tag makes sure the stub is not built in the final build.

func initializeConfig() (*config.Config, error) {
   wire.Build(config.Load)
   return nil, nil
}
```

让我们更详细地解释一下注入器。函数签名定义了一个返回`*config.Config`实例或错误的函数，这与之前的`config.Load()`是一样的。

函数的第一行调用了`wire.Build()`并提供了我们的提供者，第二行返回了`nil, nil`。事实上，它返回什么并不重要，只要它是有效的 Go 代码。Wire 中的代码生成器将读取函数签名和`wire.Build()`调用。

接下来，我们打开一个终端，并在包含我们的`wire.go`文件的目录中运行`wire`。Wire 将为我们创建一个名为`wire_gen.go`的新文件，其内容如下所示：

```go
// Code generated by Wire. DO NOT EDIT.

//go:generate wire
//+build !wireinject

package main

import (
   "github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/ch10/acme/internal/config"
)

// Injectors from wire.go:

func initializeConfig() (*config.Config, error) {
   configConfig, err := config.Load()
   if err != nil {
      return nil, err
   }
   return configConfig, nil
}
```

你会注意到这个文件也有一个构建标签，但它与我们之前写的相反。Wire 已经复制了我们的`initializeConfig()`方法，并为我们*填写了所有的细节*。

到目前为止，代码非常简单，很可能与我们自己编写的代码非常相似。你可能会觉得到目前为止我们并没有真正获得太多。我同意。当我们将其余的对象转换过来时，Wire 将为我们处理的代码和复杂性将会显著增加。

为了完成这一系列的更改，我们更新`main()`以使用我们的`initializeConfig()`函数，如下所示：

```go
func main() {
   // bind stop channel to context
   ctx := context.Background()

   // load config
   cfg, err := initializeConfig()
   if err != nil {
      os.Exit(-1)
   }

   // build the exchanger
   exchanger := exchange.NewConverter(cfg)

   // build model layer
   getModel := get.NewGetter(cfg)
   listModel := list.NewLister(cfg)
   registerModel := register.NewRegisterer(cfg, exchanger)

   // start REST server
   server := rest.New(cfg, getModel, listModel, registerModel)
   server.Listen(ctx.Done())
}
```

处理配置后，我们可以继续下一个对象，`*exchange.Converter`。在先前的示例中，我们没有使用提供程序集，而是直接将我们的提供程序传递给`wire.Build()`调用。我们即将添加另一个提供程序，所以现在是时候更加有条理了。因此，我们将在`main.go`中添加一个私有全局变量，并将我们的`Config`和`Converter`提供程序添加到其中，如下所示：

```go
// List of wire enabled objects
var wireSet = wire.NewSet(
   // *config.Config
   config.Load,

   // *exchange.Converter
   wire.Bind(new(exchange.Config), &config.Config{}),
   exchange.NewConverter,
)
```

正如您所看到的，我还添加了一个`wire.Bind()`调用。Wire 要求我们定义或映射满足接口的具体类型，以便在注入期间满足它们。`*exchange.Converter`的构造函数如下所示：

```go
// NewConverter creates and initializes the converter
func NewConverter(cfg Config) *Converter {
   return &Converter{
      cfg: cfg,
   }
}
```

您可能还记得，这个构造函数使用配置注入和本地定义的`Config`接口。但是，我们注入的实际配置对象是`*config.Config`。我们的`wire.Bind()`调用告诉 Wire，在需要`exchange.Config`接口时使用`*config.Config`。

有了我们的提供程序集，我们现在可以更新我们的配置注入器，并添加一个`Converter`的注入器，如下所示：

```go
func initializeConfig() (*config.Config, error) {
   wire.Build(wireSet)
   return nil, nil
}

func initializeExchanger() (*exchange.Converter, error) {
   wire.Build(wireSet)
   return nil, nil
}
```

重要的是要注意，虽然`exchange.NewConverter()`不会返回错误，但我们的注入器必须。这是因为我们依赖于返回错误的配置提供程序。这可能听起来很麻烦，但不用担心，Wire 可以帮助我们做到这一点。

继续我们的对象列表，我们需要对我们的模型层做同样的事情。注入器是完全可预测的，几乎与`*exchange.Converter`完全相同，提供程序集的更改也是如此。

请注意，`main()`和更改后的提供程序集如下所示：

```go
func main() {
   // bind stop channel to context
   ctx := context.Background()

   // load config
   cfg, err := initializeConfig()
   if err != nil {
      os.Exit(-1)
   }

   // build model layer
   getModel, _ := initializeGetter()
   listModel, _ := initializeLister()
   registerModel, _ := initializeRegisterer()

   // start REST server
   server := rest.New(cfg, getModel, listModel, registerModel)
   server.Listen(ctx.Done())
}

// List of wire enabled objects
var wireSet = wire.NewSet(
   // *config.Config
   config.Load,

   // *exchange.Converter
   wire.Bind(new(exchange.Config), &config.Config{}),
   exchange.NewConverter,

   // *get.Getter
   wire.Bind(new(get.Config), &config.Config{}),
   get.NewGetter,

   // *list.Lister
   wire.Bind(new(list.Config), &config.Config{}),
   list.NewLister,

   // *register.Registerer
   wire.Bind(new(register.Config), &config.Config{}),
   wire.Bind(new(register.Exchanger), &exchange.Converter{}),
   register.NewRegisterer,
)
```

有几件重要的事情。首先，我们的提供程序集变得相当长。这可能没关系，因为我们所做的唯一更改是添加更多的提供程序和绑定语句。

其次，我们不再调用`initializeExchanger()`，我们实际上已经删除了该注入器。我们不再需要这个的原因是 Wire 正在为我们处理对模型层的注入。

最后，为了简洁起见，我忽略了可能从模型层注入器返回的错误。这是一个不好的做法，但不用担心，我们将在下一组更改后很快删除这些行。

快速运行 Wire 和我们的测试以确保一切仍然按预期工作后，我们准备继续进行最后一个对象，即 REST 服务器。

首先，我们对提供程序集进行了以下可能可预测的添加：

```go
// List of wire enabled objects
var wireSet = wire.NewSet(
   // lines omitted

   // *rest.Server
   wire.Bind(new(rest.Config), &config.Config{}),
   wire.Bind(new(rest.GetModel), &get.Getter{}),
   wire.Bind(new(rest.ListModel), &list.Lister{}),
   wire.Bind(new(rest.RegisterModel), &register.Registerer{}),
   rest.New,
)
```

之后，我们在`wire.go`中为我们的 REST 服务器定义注入器，如下所示：

```go
func initializeServer() (*rest.Server, error) {
   wire.Build(wireSet)
   return nil, nil
}
```

现在，我们可以更新`main()`，只调用 REST 服务器注入器，如下所示：

```go
func main() {
   // bind stop channel to context
   ctx := context.Background()

   // start REST server
   server, err := initializeServer()
   if err != nil {
      os.Exit(-1)
   }

   server.Listen(ctx.Done())
}
```

完成后，我们可以删除除`initializeServer()`之外的所有注入器，然后运行 Wire，完成！

现在可能是检查 Wire 为我们生成的代码的好时机：

```go
func initializeServer() (*rest.Server, error) {
   configConfig, err := config.Load()
   if err != nil {
      return nil, err
   }
   getter := get.NewGetter(configConfig)
   lister := list.NewLister(configConfig)
   converter := exchange.NewConverter(configConfig)
   registerer := register.NewRegisterer(configConfig, converter)
   server := rest.New(configConfig, getter, lister, registerer)
   return server, nil
}
```

这看起来熟悉吗？这与我们采用 wire 之前的`main()`非常相似。

鉴于我们的代码已经在使用构造函数注入，并且我们的服务相当小，很容易感觉我们为了获得最小的收益而做了很多工作。如果我们从一开始就采用 Wire，肯定不会有这种感觉。在我们的特定情况下，好处更多是长期的。现在 Wire 正在处理构造函数注入以及与实例化和实例化顺序相关的所有复杂性，我们的服务的所有扩展将会更加简单，而且更不容易出现人为错误。

# API 回归测试

完成 Wire 转换后，我们如何确保我们的服务仍然按我们的期望工作？

我们唯一的即时选择是运行应用程序并尝试。这个选择现在可能还可以，但我不喜欢它作为长期选择，所以让我们看看是否可以添加一些自动化测试。

我们应该问自己的第一个问题是*我们在测试什么？*我们不应该需要测试 Wire 本身，我们可以相信工具的作者会这样做。其他方面可能出现什么问题？

一个典型的答案可能是我们使用 Wire。如果我们配置错误 Wire，它将无法生成，所以这个问题已经解决了。这让我们只剩下了应用本身。

为了测试应用程序，我们需要运行它，然后进行 HTTP 调用，并验证响应是否符合我们的预期。

我们需要考虑的第一件事是如何启动应用程序，也许更重要的是，如何以一种可以同时运行多个测试的方式来做到这一点。

目前，我们的配置（数据库连接、HTTP 端口等）是硬编码在磁盘上的一个文件中的。我们可以使用它，但它包括一个固定的 HTTP 服务器端口。另一方面，在我们的测试中硬编码数据库凭据要糟糕得多。

让我们采取一个折中的方法。首先，让我们加载标准的`config`文件：

```go
// load the standard config (from the ENV)
cfg, err := config.Load()
require.NoError(t, err)
```

现在，让我们找一个空闲的 TCP 端口来绑定我们的服务器。我们可以使用端口`0`，并允许系统自动分配一个，就像下面的代码所示：

```go
func getFreePort() (string, error) {
   for attempt := 0; attempt <= 10; attempt++ {
      addr := net.JoinHostPort("", "0")
      listener, err := net.Listen("tcp", addr)
      if err != nil {
         continue
      }

      port, err := getPort(listener.Addr())
      if err != nil {
         continue
      }

      // close/free the port
      tcpListener := listener.(*net.TCPListener)
      cErr := tcpListener.Close()
      if cErr == nil {
         file, fErr := tcpListener.File()
         if fErr == nil {
            // ignore any errors cleaning up the file
            _ = file.Close()
         }
         return port, nil
      }
   }

   return "", errors.New("no free ports")
}
```

我们现在可以使用那个空闲端口，并将`config`文件中的地址替换为使用空闲端口的地址，就像这样：

```go
// get a free port (so tests can run concurrently)
port, err := getFreePort()
require.NoError(t, err)

// override config port with free one
cfg.Address = net.JoinHostPort("0.0.0.0", port)
```

现在我们陷入了困境。目前，要创建服务器的实例，代码看起来是这样的：

```go
// start REST server
server, err := initializeServer()
if err != nil {
   os.Exit(-1)
}

server.Listen(ctx.Done())
```

配置会自动注入，我们没有机会使用我们的自定义配置。幸运的是，Wire 也可以帮助解决这个问题。

为了能够在我们的测试中手动注入配置，但不修改`main()`，我们需要将我们的提供者集分成两部分。第一部分是除了配置之外的所有依赖项：

```go
var wireSetWithoutConfig = wire.NewSet(
   // *exchange.Converter
   exchange.NewConverter,

   // *get.Getter
   get.NewGetter,

   // *list.Lister
   list.NewLister,

   // *register.Registerer
   wire.Bind(new(register.Exchanger), &exchange.Converter{}),
   register.NewRegisterer,

   // *rest.Server
   wire.Bind(new(rest.GetModel), &get.Getter{}),
   wire.Bind(new(rest.ListModel), &list.Lister{}),
   wire.Bind(new(rest.RegisterModel), &register.Registerer{}),
   rest.New,
)
```

第二个包括第一个，然后添加配置和所有相关的绑定：

```go
var wireSet = wire.NewSet(
   wireSetWithoutConfig,

   // *config.Config
   config.Load,

   // *exchange.Converter
   wire.Bind(new(exchange.Config), &config.Config{}),

   // *get.Getter
   wire.Bind(new(get.Config), &config.Config{}),

   // *list.Lister
   wire.Bind(new(list.Config), &config.Config{}),

   // *register.Registerer
   wire.Bind(new(register.Config), &config.Config{}),

   // *rest.Server
   wire.Bind(new(rest.Config), &config.Config{}),
)
```

下一步是创建一个以 config 为参数的注入器。在我们的情况下，这有点奇怪，因为这是由我们的 config 注入引起的，但它看起来是这样的：

```go
func initializeServerCustomConfig(_ exchange.Config, _ get.Config, _ list.Config, _ register.Config, _ rest.Config) *rest.Server {
   wire.Build(wireSetWithoutConfig)
   return nil
}
```

运行 Wire 后，我们现在可以启动我们的测试服务器，就像下面的代码所示：

```go
// start the test server on a random port
go func() {
   // start REST server
   server := initializeServerCustomConfig(cfg, cfg, cfg, cfg, cfg)
   server.Listen(ctx.Done())
}()
```

将所有内容放在一起，我们现在有一个函数，它在一个随机端口上创建一个服务器，并返回服务器的地址，这样我们的测试就知道在哪里调用。以下是完成的函数：

```go
func startTestServer(t *testing.T, ctx context.Context) string {
   // load the standard config (from the ENV)
   cfg, err := config.Load()
   require.NoError(t, err)

   // get a free port (so tests can run concurrently)
   port, err := getFreePort()
   require.NoError(t, err)

   // override config port with free one
   cfg.Address = net.JoinHostPort("0.0.0.0", port)

   // start the test server on a random port
   go func() {
      // start REST server
      server := initializeServerCustomConfig(cfg, cfg, cfg, cfg, cfg)
      server.Listen(ctx.Done())
   }()

   // give the server a chance to start
   <-time.After(100 * time.Millisecond)

   // return the address of the test server
   return "http://" + cfg.Address
}
```

现在，让我们来看一个测试。同样，我们将使用注册端点作为示例。首先，我们的测试需要启动一个测试服务器。在下面的示例中，您还会注意到我们正在定义一个带有超时的上下文。当上下文完成时，通过超时或被取消，测试服务器将关闭；因此，这个超时成为了我们测试的*最大执行时间*。以下是启动服务器的代码：

```go
// start a context with a max execution time
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// start test server
serverAddress := startTestServer(t, ctx)
```

接下来，我们需要构建并发送请求。在这种情况下，我们选择了硬编码负载和 URL。这可能看起来有点奇怪，但实际上有点帮助。如果负载或 URL（这两者都构成我们服务的 API）意外更改，这些测试将会失败。另一方面，考虑一下，如果我们使用一个常量来配置服务器的 URL。如果那个常量被更改，API 将会更改，并且会破坏我们的用户。负载也是一样，我们可以使用内部使用的相同 Go 对象，但那里的更改也不会导致测试失败。

是的，这种重复工作更多，确实使测试更加脆弱，这两者都不好，但是我们的测试出问题总比我们的用户出问题要好。

构建和发送请求的代码如下：

```go
    // build and send request
   payload := bytes.NewBufferString(`
{
   "fullName": "Bob",
   "phone": "0123456789",
   "currency": "AUD"
}
`)

   req, err := http.NewRequest("POST", serverAddress+"/person/register", payload)
   require.NoError(t, err)

   resp, err := http.DefaultClient.Do(req)
   require.NoError(t, err)
```

现在剩下的就是验证结果。将所有内容放在一起后，我们有了这个：

```go
func TestRegister(t *testing.T) {
   // start a context with a max execution time
   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
   defer cancel()

   // start test server
   serverAddress := startTestServer(t, ctx)

   // build and send request
   payload := bytes.NewBufferString(`
{
   "fullName": "Bob",
   "phone": "0123456789",
   "currency": "AUD"
}
`)

   req, err := http.NewRequest("POST", serverAddress+"/person/register", payload)
   require.NoError(t, err)

   resp, err := http.DefaultClient.Do(req)
   require.NoError(t, err)

   // validate expectations
   assert.Equal(t, http.StatusCreated, resp.StatusCode)
   assert.NotEmpty(t, resp.Header.Get("Location"))
}
```

就是这样。我们现在有了一个自动化测试，确保我们的应用程序启动，可以被调用，并且响应如我们所期望的那样。如果您感兴趣，本章的代码中还有另外两个端点的测试。

# 现成的注入的缺点

尽管框架作者希望他们的工作成为一种万能解决方案，解决所有世界上的 DI 问题，但很遗憾，事实并非如此；采用框架是有一些成本的，也有一些原因可能会选择不使用它。这些包括以下内容。

**仅支持构造函数注入**-你可能已经注意到在本章中，所有的例子都使用构造函数注入。这并非偶然。与许多框架一样，Wire 只支持构造函数注入。我们不必删除其他 DI 方法的使用，但框架无法帮助我们处理它。

**采用可能成本高昂**-正如你在前一节中看到的，采用框架的最终结果可能相当不错，但我们的服务规模较小，而且我们已经在使用 DI。如果这两者中有任何一种情况不成立，我们将需要进行大量的重构工作。正如我们之前讨论过的，我们做的改变越多，我们承担的风险就越大。

这些成本和风险可以通过具有框架的先前经验以及在项目早期采用框架来减轻。

**意识形态问题**-这本身并不是一个缺点，而更多的是你可能不想采用框架的原因。在 Go 社区中，你会遇到一种观点，即框架与 Go 的哲学不符。虽然我没有找到官方声明或文件支持这一观点，但我相信这是基于 Go 的创作者是 Unix 哲学的粉丝和作者，该哲学规定*在隔离中做琐事，然后组合起来使事情有用*。

框架可能被视为违反这种意识形态，特别是如果它们成为整个系统的普遍部分。我们在本章中提到的框架范围相对较小；所以和其他一切一样，我会让你自己做决定。

# 总结

在本章中，我们讨论了使用 DI 框架来减轻管理和注入依赖关系的负担。我们讨论了 DI 框架中常见的优缺点，并将 Google 的 Wire 框架应用到我们的示例服务中。

这是我们将讨论的最后一个 DI 方法，在下一章中，我们将采取完全不同的策略，看看不使用 DI 的原因。我们还将看看应用 DI 实际上使代码变得更糟的情况。

# 问题

1.  在采用 DI 框架时，你可以期待获得什么？

1.  在评估 DI 框架时，你应该注意哪些问题？

1.  采用现成的注入的理想用例是什么？

1.  为什么重要保护服务免受意外 API 更改的影响？
