# 第二章。服务和路由

作为商业实体的 Web 的基石——营销和品牌依赖的基础——是 URL。虽然我们还没有看到顶级域处理，但我们需要掌握我们的 URL 及其路径（或端点）。

在本章中，我们将通过引入多个路由和相应的处理程序来做到这一点。首先，我们将通过简单的平面文件服务来做到这一点，然后我们将引入复杂的混合物，通过实现一个利用正则表达式的路由的库来实现更灵活的路由。

在本章结束时，您应该能够在本地主机上创建一个可以通过任意数量的路径访问并返回相对于请求路径的内容的站点。

在本章中，我们将涵盖以下主题：

+   直接提供文件

+   基本路由

+   使用 Gorilla 进行更复杂的路由

+   重定向请求

+   提供基本错误

# 直接提供文件

在上一章中，我们利用了`fmt.Fprintln`函数在浏览器中输出了一些通用的 Hello, World 消息。

这显然有限的效用。在 Web 和 Web 服务器的早期，整个 Web 都是通过将请求定向到相应的静态文件来提供的。换句话说，如果用户请求`home.html`，Web 服务器将查找名为`home.html`的文件并将其返回给用户。

今天这可能看起来有点古怪，因为现在绝大多数的 Web 都以某种动态方式提供，内容通常是通过数据库 ID 确定的，这允许页面在没有人修改单个文件的情况下生成和重新生成。

让我们看看我们可以以类似于 Web 早期的方式提供文件的最简单方法：

```go
package main

import (
  "net/http"
)

const (
  PORT = ":8080"
)

func main() {

  http.ListenAndServe(PORT, http.FileServer(http.Dir("/var/www")))
}
```

相当简单，对吧？对站点发出的任何请求都将尝试在我们本地的`/var/www`目录中找到相应的文件。但是，虽然与第一章 *介绍和设置 Go*中的例子相比，这更具实际用途，但仍然相当有限。让我们看看如何扩展我们的选择。

# 基本路由

在第一章 *介绍和设置*中，我们生成了一个非常基本的 URL 端点，允许静态文件服务。

以下是我们为该示例生成的简单路由：

```go
func main() {
  http.HandleFunc("/static",serveStatic)
  http.HandleFunc("/",serveDynamic)
  http.ListenAndServe(Port,nil)
}
```

回顾一下，你可以看到两个端点，`/static`和`/`，它们要么提供单个静态文件，要么生成`http.ResponseWriter`的输出。

我们可以有任意数量的路由器并排坐着。但是，考虑这样一个情景，我们有一个基本的网站，包括关于、联系和员工页面，每个页面都驻留在`/var/www/about/index.html`、`/var/www/contact.html`和`/var/www/staff/home.html`。虽然这是一个故意晦涩的例子，但它展示了 Go 内置和未修改的路由系统的局限性。我们无法在本地将所有请求路由到同一个目录，我们需要一些提供更灵活 URL 的东西。

# 使用 Gorilla 进行更复杂的路由

在上一节中，我们看了基本路由，但这只能带我们走到这里，我们必须明确地定义我们的端点，然后将它们分配给处理程序。如果我们的 URL 中有通配符或变量会发生什么？这是 Web 和任何严肃的 Web 服务器的绝对必要部分。

举一个非常简单的例子，考虑托管一个博客，每篇博客文章都有唯一的标识符。这可以是代表数据库 ID 条目的数字 ID，也可以是基于文本的全局唯一标识符，比如`my-first-block-entry`。

### 注意

在上面的例子中，我们希望将类似`/pages/1`的 URL 路由到名为`1.html`的文件。或者，在基于数据库的情况下，我们希望使用`/pages/1`或`/pages/hello-world`来映射到具有 GUID`1`或`hello-world`的数据库条目。为了做到这一点，我们要么需要包含一个可能的端点的详尽列表，这是非常浪费的，要么通过正则表达式实现通配符，这是理想的。

无论哪种情况，我们都希望能够直接在应用程序中利用 URL 中的值。这在使用`GET`或`POST`的 URL 参数时非常简单。我们可以简单地提取这些参数，但它们在干净、分层或描述性 URL 方面并不特别优雅，而这些通常是搜索引擎优化所必需的。

内置的`net/http`路由系统可能出于设计考虑相对简单。要从任何给定请求的值中获得更复杂的内容，我们要么需要扩展路由功能，要么使用已经完成这一点的包。

在 Go 公开可用并且社区不断发展的几年中，出现了许多 Web 框架。我们将在本书的后续部分更深入地讨论这些内容，但其中一个特别受欢迎和非常有用的是 Gorilla Web Toolkit。

正如其名称所暗示的，Gorilla 更像是一组非常有用的工具，而不是一个框架。具体来说，Gorilla 包含：

+   `gorilla/context`：这是一个用于从请求中创建全局可访问变量的包。它对于在整个应用程序中共享 URL 的值而不重复访问代码非常有用。

+   `gorilla/rpc`：这实现了 RPC-JSON，这是一种用于远程代码服务和通信的系统，而不实现特定协议。这依赖于 JSON 格式来定义任何请求的意图。

+   `gorilla/schema`：这是一个允许将表单变量简单打包到`struct`中的包，否则这是一个繁琐的过程。

+   `gorilla/securecookie`：毫不奇怪，这个包实现了应用程序的经过身份验证和加密的 cookie。

+   `gorilla/sessions`：类似于 cookie，这个包通过使用基于文件和/或基于 cookie 的会话系统提供了独特的、长期的和可重复的数据存储。

+   `gorilla/mux`：旨在创建灵活的路由，允许正则表达式来指示路由器可用的变量。

+   最后一个包是我们在这里最感兴趣的包，它还带有一个相关的包叫做`gorilla/reverse`，它基本上允许您反转基于正则表达式的 mux 创建过程。我们将在后面的章节中详细介绍这个主题。

### 注意

您可以通过它们的 GitHub 位置使用`go get`获取单独的 Gorilla 包。例如，要获取 mux 包，只需访问[github.com/gorilla/mux](http://github.com/gorilla/mux)即可将该包带入您的`GOPATH`。有关其他包的位置（它们都相当自明），请访问[`www.gorillatoolkit.org/`](http://www.gorillatoolkit.org/)。

让我们深入了解如何创建一个灵活的路由，并使用正则表达式将参数传递给我们的处理程序：

```go
package main

import (
  "github.com/gorilla/mux"
  "net/http"
)

const (
  PORT = ":8080"
)
```

这应该看起来很熟悉，除了 Gorilla 包的导入之外：

```go
func pageHandler(w http.ResponseWriter, r *http.Request) {
  vars := mux.Vars(r)
  pageID := vars["id"]
  fileName := "files/" + pageID + ".html"
  http.ServeFile(w,r,fileName)
}
```

在这里，我们创建了一个路由处理程序来接受响应。这里需要注意的是使用了`mux.Vars`，这是一个方法，它将从`http.Request`中查找查询字符串变量并将它们解析成一个映射。然后可以通过键引用结果来访问这些值，本例中是`id`，我们将在下一节中介绍。

```go
func main() {
  rtr := mux.NewRouter()
  rtr.HandleFunc("/pages/{id:[0-9]+}",pageHandler)
  http.Handle("/",rtr)
  http.ListenAndServe(PORT,nil)
}
```

在这里，我们可以看到处理程序中的（非常基本的）正则表达式。我们将`/pages/`后面的任意数量的数字分配给名为`id`的参数，即`{id:[0-9]+}`；这是我们在`pageHandler`中提取出来的值。

一个更简单的版本显示了如何用它来划分不同的页面，可以通过添加一对虚拟端点来看到：

```go
func main() {
  rtr := mux.NewRouter()
  rtr.HandleFunc("/pages/{id:[0-9]+}", pageHandler)
  rtr.HandleFunc("/homepage", pageHandler)
  rtr.HandleFunc("/contact", pageHandler)
  http.Handle("/", rtr)
  http.ListenAndServe(PORT, nil)
}
```

当我们访问与此模式匹配的 URL 时，我们的`pageHandler`会尝试在`files/`子目录中找到页面并直接返回该文件。

对`/pages/1`的响应会像这样：

![使用 Gorilla 进行更复杂的路由](img/B04294_02_01.jpg)

在这一点上，你可能已经在问，但是如果我们没有请求的页面怎么办？或者，如果我们移动了那个位置会发生什么？这引出了网络服务中的两个重要机制——返回错误响应，以及作为其中一部分，可能重定向已移动或具有其他需要向最终用户报告的有趣属性的请求。

# 重定向请求

在我们看简单和非常常见的错误，比如 404 之前，让我们先讨论重定向请求的想法，这是非常常见的。尽管并非总是对于普通用户来说是明显或可触及的原因。

那么我们为什么要将请求重定向到另一个请求呢？好吧，根据 HTTP 规范的定义，有很多原因可能导致我们在任何给定的请求上实现自动重定向。以下是其中一些及其相应的 HTTP 状态码：

+   非规范地址可能需要重定向到规范地址以用于 SEO 目的或站点架构的更改。这由*301 永久移动*或*302 找到*处理。

+   在成功或不成功的`POST`之后重定向。这有助于防止意外重新提交相同的表单数据。通常，这由*307 临时重定向*定义。

+   页面不一定丢失，但现在位于另一个位置。这由状态码*301 永久移动*处理。

在基本的 Go 中使用`net/http`执行任何一个都非常简单，但是正如你所期望的那样，使用更健壮的框架，比如 Gorilla，可以更加方便和改进。

# 提供基本错误

在这一点上，谈论一下错误是有些合理的。很可能，当你玩我们的基本平面文件服务服务器时，特别是当你超出两三页时，你可能已经遇到了错误。

我们的示例代码包括四个用于平面服务的示例 HTML 文件，编号为`1.html`，`2.html`等等。然而，当你访问`/pages/5`端点时会发生什么？幸运的是，`http`包会自动处理文件未找到错误，就像大多数常见的网络服务器一样。

此外，与大多数常见的网络服务器类似，错误页面本身很小，单调，毫无特色。在接下来的部分中，你可以看到我们从 Go 得到的**404 页面未找到**状态响应：

![提供基本错误](img/B04294_02_02.jpg)

正如前面提到的，这是一个非常基本和毫无特色的页面。通常情况下，这是一件好事——错误页面包含的信息或风格超过必要的可能会产生负面影响。

考虑这个错误——`404`——作为一个例子。如果我们包含对同一服务器上存在的图像和样式表的引用，如果这些资产也丢失了会发生什么？

简而言之，你很快就会遇到递归错误——每个`404`页面都会调用一个触发`404`响应的图像和样式表，循环重复。即使网络服务器足够聪明以停止这一点，而且很多都是，它也会在日志中产生噩梦般的场景，使它们充满了噪音，变得毫无用处。

让我们看一些代码，我们可以用来为我们的`/files`目录中任何丢失的文件实现一个全局的`404`页面：

```go
package main

import (
  "github.com/gorilla/mux"
  "net/http"
  "os"
)

const (
  PORT = ":8080"
)

func pageHandler(w http.ResponseWriter, r *http.Request) {
  vars := mux.Vars(r)
  pageID := vars["id"]
  fileName := "files/" + pageID + ".html"_, 
  err := os.Stat(fileName)
    if err != nil {
      fileName = "files/404.html"
    }

  http.ServeFile(w,r,fileName)
}
```

在这里，你可以看到我们首先尝试使用`os.Stat`检查文件（及其潜在错误），并输出我们自己的`404`响应：

```go
func main() {
  rtr := mux.NewRouter()
  rtr.HandleFunc("/pages/{id:[0-9]+}",pageHandler)
  http.Handle("/",rtr)
  http.ListenAndServe(PORT,nil)
}
```

现在，如果我们看一下`404.html`页面，我们会发现我们创建了一个自定义的 HTML 文件，它产生的东西比我们之前调用的默认**Go 页面未找到**消息更加用户友好。

让我们看看这是什么样子，但请记住，它可以看起来任何你想要的样子：

```go
<!DOCTYPE html>
<html>
<head>
<title>Page not found!</title>
<style type="text/css">
body {
  font-family: Helvetica, Arial;
  background-color: #cceeff;
  color: #333;
  text-align: center;
}
</style>
<link rel="stylesheet" type="text/css" media="screen" href="http://code.ionicframework.com/ionicons/2.0.1/css/ionicons.min.css"></link>
</head>

<body>
<h1><i class="ion-android-warning"></i> 404, Page not found!</h1>
<div>Look, we feel terrible about this, but at least we're offering a non-basic 404 page</div>
</body>

</html>
```

另外，请注意，虽然我们将`404.html`文件保存在与其他文件相同的目录中，但这仅仅是为了简单起见。

实际上，在大多数生产环境中，具有自定义错误页面，我们更希望它存在于自己的目录中，最好是在我们网站的公开可用部分之外。毕竟，现在您可以通过访问`http://localhost:8080/pages/404`的方式访问错误页面，这实际上并不是一个错误。这会返回错误消息，但实际情况是，在这种情况下找到了文件，我们只是返回它。

让我们通过访问`http://localhost/pages/5`来看一下我们新的、更漂亮的`404`页面，这指定了一个在我们的文件系统中不存在的静态文件：

![提供基本错误](img/B04294_02_03.jpg)

通过显示更加用户友好的错误消息，我们可以为遇到错误的用户提供更有用的操作。考虑一些其他可能受益于更具表现力的错误页面的常见错误。

# 总结

现在我们不仅可以从`net/http`包中产生基本路由，还可以使用 Gorilla 工具包产生更复杂的路由。通过利用 Gorilla，我们现在可以创建正则表达式，并实现基于模式的路由，并允许我们的路由模式更加灵活。

有了这种增加的灵活性，我们现在也必须注意错误，因此我们已经考虑了处理基于错误的重定向和消息，包括自定义的**404，页面未找到**消息，以产生更定制的错误消息。

现在我们已经掌握了创建端点、路由和处理程序的基础知识，我们需要开始进行一些非平凡的数据服务。

在第三章 *连接到数据*中，我们将开始从数据库中获取动态信息，这样我们就可以更智能、更可靠地管理数据。通过连接到一些不同的常用数据库，我们将能够构建强大、动态和可扩展的 Web 应用程序。
