# 第四章。使用模板

在第二章中，*服务和路由*，我们探讨了如何将 URL 转换为网络应用程序中的不同页面。这样做的结果是，我们构建了动态的 URL，并从我们（非常简单的）`net/http`处理程序中获得了动态响应。

我们将我们的数据呈现为真实的 HTML，但我们将我们的 HTML 直接硬编码到我们的 Go 源代码中。这对于生产级环境来说并不理想，原因有很多。

幸运的是，Go 配备了一个强大但有时棘手的模板引擎，用于文本模板和 HTML 模板。

与许多其他模板语言不同，这些语言将逻辑排除在演示方面，Go 的模板包使您能够在模板中使用一些逻辑结构，例如循环、变量和函数声明。这使您能够将一些逻辑偏移至模板，这意味着您可以编写应用程序，但需要允许模板方面为产品提供一些可扩展性，而无需重写源代码。

我们说一些逻辑结构，因为 Go 模板被称为无逻辑。我们将在稍后讨论这个话题。

在本章中，我们将探讨不仅呈现数据的方式，还将探索本章中的一些更高级的可能性。最后，我们将能够将我们的模板转化为推进演示和源代码分离的方式。

我们将涵盖以下主题：

+   介绍模板、上下文和可见性

+   HTML 模板和文本模板

+   显示变量和安全性

+   使用逻辑和控制结构

# 介绍模板、上下文和可见性

很值得注意的是，虽然我们正在讨论将 HTML 部分从源代码中提取出来，但是在 Go 应用程序中使用模板是可能的。事实上，像这样声明模板是没有问题的：

```go
tpl, err := template.New("mine").Parse(`<h1>{{.Title}}</h1>`)
```

然而，如果我们这样做，每次模板需要更改时，我们都需要重新启动应用程序。如果我们使用基于文件的模板，就不必这样做；相反，我们可以在不重新启动的情况下对演示（和一些逻辑）进行更改。

从应用程序内的 HTML 字符串转移到基于文件的模板的第一件事是创建一个模板文件。让我们简要地看一下一个示例模板，它在某种程度上接近我们在本章后面将得到的结果：

```go
<!DOCTYPE html>
<html>
<head>
<title>{{.Title}}</title>
</head>
<body>
  <h1>{{.Title}}</h1>

  <div>{{.Date}}</div>

  {{.Content}}
</body>
</html>
```

非常简单，对吧？变量通过双大括号内的名称清楚地表示。那么所有的句号/点是怎么回事？与其他一些类似风格的模板系统（如 Mustache、Angular 等）一样，句号表示范围或上下文。

最容易演示这一点的地方是变量可能重叠的地方。想象一下，我们有一个标题为**博客条目**的页面，然后我们列出所有已发布的博客文章。我们有一个页面标题，但我们也有单独的条目标题。我们的模板可能看起来类似于这样：

```go
{{.Title}}
{{range .Blogs}}
  <li><a href="{{.Link}}">{{.Title}}</a></li>
{{end}}
```

这里的点指定了特定的范围，这种情况下是通过 range 模板操作符语法进行循环。这允许模板解析器正确地使用`{{.Title}}`作为博客的标题，而不是页面的标题。

这一切都值得注意，因为我们将创建的第一个模板将利用通用范围变量，这些变量以点表示。

# HTML 模板和文本模板

在我们第一个示例中，我们将从数据库中将博客的值显示到网络上，我们生成了一个硬编码的 HTML 字符串，并直接注入了我们的值。

以下是我们在第三章中使用的两行：

```go
  html := `<html><head><title>` + thisPage.Title + `</title></head><body><h1>` + thisPage.Title + `</h1><div>` + thisPage.Content + `</div></body></html>
  fmt.Fprintln(w, html)
```

这不难理解为什么这不是一个可持续的系统，用于将我们的内容输出到网络上。最好的方法是将其转换为模板，这样我们就可以将演示与应用程序分开。

为了尽可能简洁地做到这一点，让我们修改调用前面代码的方法`ServePage`，使用模板而不是硬编码的 HTML。

所以我们将删除之前放置的 HTML，而是引用一个文件，该文件将封装我们想要显示的内容。从你的根目录开始，创建一个`templates`子目录，并在其中创建一个`blog.html`。

以下是我们包含的非常基本的 HTML，随意添加一些花样：

```go
<html>
<head>
<title>{{.Title}}</title>
</head>
<body>
  <h1>{{.Title}}</h1>
  <p>
    {{.Content}}
  </p>
  <div>{{.Date}}</div>
</body>
</html>
```

回到我们的应用程序，在`ServePage`处理程序中，我们将稍微改变我们的输出代码，不再留下显式的字符串，而是解析和执行我们刚刚创建的 HTML 模板：

```go
func ServePage(w http.ResponseWriter, r *http.Request) {
  vars := mux.Vars(r)
  pageGUID := vars["guid"]
  thisPage := Page{}
  fmt.Println(pageGUID)
  err := database.QueryRow("SELECT page_title,page_content,page_date FROM pages WHERE page_guid=?", pageGUID).Scan(&thisPage.Title, &thisPage.Content, &thisPage.Date)
  if err != nil {
    http.Error(w, http.StatusText(404), http.StatusNotFound)
    log.Println("Couldn't get page!")
    return
  }
  // html := <html>...</html>

  t, _ := template.ParseFiles("templates/blog.html")
  t.Execute(w, thisPage)
}
```

如果你以某种方式未能创建文件或者文件无法访问，应用程序在尝试执行时将会发生 panic。如果你引用了不存在的`struct`值，也会发生 panic——我们需要更好地处理错误。

### 注意

注意：不要忘记在你的导入中包含`html/template`。

远离静态字符串的好处是显而易见的，但现在我们已经为一个更具扩展性的呈现层奠定了基础。

如果我们访问`http://localhost:9500/page/hello-world`，我们将看到类似于这样的东西：

![HTML 模板和文本模板](img/B04294_04_01.jpg)

# 显示变量和安全性

为了演示这一点，让我们通过在 MySQL 命令行中添加这个 SQL 命令来创建一个新的博客条目：

```go
INSERT INTO `pages` (`id`, `page_guid`, `page_title`, page_content`, `page_date`)
```

值：

```go
  (2, 'a-new-blog', 'A New Blog', 'I hope you enjoyed the last blog!  Well brace yourself, because my latest blog is even <i>better</i> than the last!', '2015-04-29 02:16:19');
```

另一个令人兴奋的内容，当然。但是请注意，当我们尝试给单词 better 加上斜体时，我们在其中嵌入了一些 HTML。

不管如何存储格式的争论，这使我们能够查看 Go 的模板如何默认处理这个问题。如果我们访问`http://localhost:9500/page/a-new-blog`，我们将看到类似于这样的东西：

![显示变量和安全性](img/B04294_04_02.jpg)

正如你所看到的，Go 会自动为我们的输出数据进行消毒。有很多非常非常明智的原因来做这个，这就是为什么这是默认行为的最大原因。当然，最大的原因是为了避免来自不受信任的输入源（例如网站的一般用户等）的 XSS 和代码注入攻击向量。

但表面上，我们正在创建这个内容，应该被视为受信任的。因此，为了将其验证为受信任的 HTML，我们需要改变`template.HTML`的类型：

```go
type Page struct {
  Title   string
  Content template.HTML
  Date   string
}
```

如果你尝试将生成的 SQL 字符串值简单地扫描到`template.HTML`中，你会发现以下错误：

```go
sql: Scan error on column index 1: unsupported driver -> Scan pair: []uint8 -> *template.HTML
```

解决这个问题的最简单方法是保留`RawContent`中的字符串值，并将其重新分配给`Content`：

```go
type Page struct {
  Title    string
  RawContent string
  Content    template.HTML
  Date    string
}
  err := database.QueryRow("SELECT page_title,page_content,page_date FROM pages WHERE page_guid=?", pageGUID).Scan(&thisPage.Title, &thisPage.RawContent, &thisPage.Date)
  thisPage.Content = template.HTML(thisPage.RawContent)
```

如果我们再次`go run`，我们将看到我们的 HTML 是受信任的：

![显示变量和安全性](img/B04294_04_03.jpg)

# 使用逻辑和控制结构

在本章的前面，我们看到了如何在我们的模板中使用范围，就像我们直接在我们的代码中使用一样。看一下下面的代码：

```go
{{range .Blogs}}
  <li><a href="{{.Link}}">{{.Title}}</a></li>
{{end}}
```

你可能还记得我们说过，Go 的模板没有任何逻辑，但这取决于你如何定义逻辑，以及共享逻辑是否完全存在于应用程序、模板中，还是两者都有一点。这是一个小问题，但因为 Go 的模板提供了很大的灵活性，所以这是值得思考的一个问题。

在前面的模板中具有一个范围功能，本身就为我们的博客的新呈现打开了很多可能性。现在我们可以显示博客列表，或者将我们的博客分成段落，并允许每个段落作为一个单独的实体存在。这可以用来允许评论和段落之间的关系，这在最近的一些出版系统中已经开始成为一个功能。

但现在，让我们利用这个机会在一个新的索引页面中创建一个博客列表。为此，我们需要添加一个路由。由于我们有`/page`，我们可以选择`/pages`，但由于这将是一个索引，让我们选择`/`和`/home`：

```go
  routes := mux.NewRouter()
  routes.HandleFunc("/page/{guid:[0-9a-zA\\-]+}", ServePage)
  routes.HandleFunc("/", RedirIndex)
  routes.HandleFunc("/home", ServeIndex)
  http.Handle("/", routes)
```

我们将使用`RedirIndex`自动重定向到我们的`/home`端点作为规范的主页。

在我们的方法中提供简单的`301`或`永久移动`重定向需要非常少的代码，如下所示：

```go
func RedirIndex(w http.ResponseWriter, r *http.Request) {
  http.Redirect(w, r, "/home", 301)
}
```

这足以接受来自`/`的任何请求，并自动将用户带到`/home`。现在，让我们看看如何在`ServeIndex`HTTP 处理程序中循环遍历我们的博客在我们的索引页面上：

```go
func ServeIndex(w http.ResponseWriter, r *http.Request) {
  var Pages = []Page{}
  pages, err := database.Query("SELECT page_title,page_content,page_date FROM pages ORDER BY ? DESC", "page_date")
  if err != nil {
    fmt.Fprintln(w, err.Error)
  }
  defer pages.Close()
  for pages.Next() {
    thisPage := Page{}
    pages.Scan(&thisPage.Title, &thisPage.RawContent, &thisPage.Date)
    thisPage.Content = template.HTML(thisPage.RawContent)
    Pages = append(Pages, thisPage)
  }
  t, _ := template.ParseFiles("templates/index.html")
  t.Execute(w, Pages)
}
```

这是`templates/index.html`：

```go
<h1>Homepage</h1>

{{range .}}
  <div><a href="!">{{.Title}}</a></div>
  <div>{{.Content}}</div>
  <div>{{.Date}}</div>
{{end}}
```

使用逻辑和控制结构

在这里我们突出了`Page struct`的一个问题——我们无法获取页面的`GUID`引用。因此，我们需要修改我们的`struct`以包括可导出的`Page.GUID`变量：

```go
type Page struct {
  Title  string
  Content  template.HTML
  RawContent  string
  Date  string
  GUID   string
}
```

现在，我们可以将我们索引页面上的列表链接到它们各自的博客条目，如下所示：

```go
  var Pages = []Page{}
  pages, err := database.Query("SELECT page_title,page_content,page_date,page_guid FROM pages ORDER BY ? DESC", "page_date")
  if err != nil {
    fmt.Fprintln(w, err.Error)
  }
  defer pages.Close()
  for pages.Next() {
    thisPage := Page{}
    pages.Scan(&thisPage.Title, &thisPage.Content, &thisPage.Date, &thisPage.GUID)
    Pages = append(Pages, thisPage)
  }
```

我们可以使用以下代码更新我们的 HTML 部分：

```go
<h1>Homepage</h1>

{{range .}}
  <div><a href="/page/{{.GUID}}">{{.Title}}</a></div>
  <div>{{.Content}}</div>
  <div>{{.Date}}</div>
{{end}}
```

但这只是模板强大功能的开始。如果我们有一个更长的内容，并且想要截断它的描述呢？

我们可以在`Page struct`中创建一个新字段并对其进行截断。但这有点笨拙；它要求该字段始终存在于`struct`中，无论是否填充了数据。将方法暴露给模板本身要高效得多。

所以让我们这样做。

首先，创建另一个博客条目，这次内容值更大。选择任何你喜欢的内容，或者按照所示选择`INSERT`命令：

```go
INSERT INTO `pages` (`id`, `page_guid`, `page_title`, `page_content`, `page_date`)
```

值：

```go
  (3, 'lorem-ipsum', 'Lorem Ipsum', 'Lorem ipsum dolor sit amet, consectetur adipiscing elit. Maecenas sem tortor, lobortis in posuere sit amet, ornare non eros. Pellentesque vel lorem sed nisl dapibus fringilla. In pretium...', '2015-05-06 04:09:45');
```

### 注意

注意：为了简洁起见，我们已经截断了我们之前的 Lorem Ipsum 文本的完整长度。

现在，我们需要将我们的截断表示为`Page`类型的方法。让我们创建该方法，以返回表示缩短文本的字符串。

这里的酷之处在于，我们可以在应用程序和模板之间共享方法：

```go
func (p Page) TruncatedText() string {
  chars := 0
  for i, _ := range p.Content {
    chars++
    if chars > 150 {
      return p.Content[:i] + ` ...`
    }
  }
  return p.Content
}
```

这段代码将循环遍历内容的长度，如果字符数超过`150`，它将返回索引中的切片直到该数字。如果它从未超过该数字，`TruncatedText`将返回整个内容。

在模板中调用这个方法很简单，只是你可能期望需要传统的函数语法调用，比如`TruncatedText()`。相反，它被引用为作用域内的任何变量一样：

```go
<h1>Homepage</h1>

{{range .}}
  <div><a href="/page/{{.GUID}}">{{.Title}}</a></div>
  <div>{{.TruncatedText}}</div>
  <div>{{.Date}}</div>
{{end}}
```

通过调用.`TruncatedText`，我们本质上通过该方法内联处理值。结果页面反映了我们现有的博客，而不是截断的博客，以及我们新的博客条目，其中包含截断的文本和省略号：

使用逻辑和控制结构

我相信你可以想象在模板中直接引用嵌入方法将打开一系列的演示可能性。

# 总结

我们只是初步了解了 Go 模板的功能，随着我们的继续探索，我们将进一步探讨更多的主题，但是这一章节已经介绍了开始直接利用模板所需的核心概念。

我们已经研究了简单的变量，以及在应用程序中实现方法，在模板本身中实现方法。我们还探讨了如何绕过受信任内容的注入保护。

在下一章中，我们将集成后端 API，以 RESTful 方式访问信息以读取和操作底层数据。这将允许我们在模板上使用 Ajax 做一些更有趣和动态的事情。
