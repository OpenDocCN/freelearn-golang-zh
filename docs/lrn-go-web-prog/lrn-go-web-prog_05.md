# 第五章。RESTful API 与前端集成

在第二章*服务和路由*中，我们探讨了如何将 URL 路由到我们 Web 应用程序中的不同页面。在这样做时，我们构建了动态的 URL，并从我们（非常简单的）`net/http`处理程序中获得了动态响应。

我们刚刚触及了 Go 模板的一小部分功能，随着我们的继续，我们还将探索更多主题，但在本章中，我们试图介绍直接开始使用模板所必需的核心概念。

我们已经研究了简单的变量以及在应用程序中使用模板本身实现的方法。我们还探讨了如何绕过对受信任内容的注入保护。

网站开发的呈现方面很重要，但也是最不根深蒂固的方面。几乎任何框架都会呈现其内置的 Go 模板和路由语法的扩展。真正将我们的应用程序提升到下一个水平的是构建和集成 API，用于通用数据访问，以及允许我们的呈现层更具动态驱动性。

在本章中，我们将开发一个后端 API，以 RESTful 方式访问信息，并读取和操作我们的基础数据。这将允许我们在模板中使用 Ajax 做一些更有趣和动态的事情。

在本章中，我们将涵盖以下主题：

+   设置基本的 API 端点

+   RESTful 架构和最佳实践

+   创建我们的第一个 API 端点

+   实施安全性

+   使用 POST 创建数据

+   使用 PUT 修改数据

# 设置基本的 API 端点

首先，我们将为页面和单独的博客条目设置一个基本的 API 端点。

我们将为`GET`请求创建一个 Gorilla 端点路由，该请求将返回有关我们页面的信息，还有一个接受 GUID 的请求，GUID 匹配字母数字字符和连字符：

```go
routes := mux.NewRouter()
routes.HandleFunc("/api/pages", APIPage).
  Methods("GET").
  Schemes("https")
routes.HandleFunc("/api/pages/{guid:[0-9a-zA\\-]+}", APIPage).
  Methods("GET").
  Schemes("https")
routes.HandleFunc("/page/{guid:[0-9a-zA\\-]+}", ServePage)
http.Handle("/", routes)
http.ListenAndServe(PORT, nil)
```

请注意，我们再次捕获了 GUID，这次是为我们的`/api/pages/*`端点，它将反映网页端点的功能，返回与单个页面相关的所有元数据。

```go
func APIPage(w http.ResponseWriter, r *http.Request) {
vars := mux.Vars(r)
pageGUID := vars["guid"]
thisPage := Page{}
fmt.Println(pageGUID)
err := database.QueryRow("SELECT page_title,page_content,page_date FROM pages WHERE page_guid=?", pageGUID).Scan(&thisPage.Title, &thisPage.RawContent, &thisPage.Date)
thisPage.Content = template.HTML(thisPage.RawContent)
if err != nil {
  http.Error(w, http.StatusText(404), http.StatusNotFound)
  log.Println(err)
  return
}
APIOutput, err := json.Marshal(thisPage)
    fmt.Println(APIOutput)
if err != nil {
  http.Error(w, err.Error(), http.StatusInternalServerError)
  return
}
w.Header().Set("Content-Type", "application/json")
fmt.Fprintln(w, thisPage)
}
```

前面的代码代表了最简单的基于 GET 的请求，它从我们的`/pages`端点返回单个记录。现在让我们来看看 REST，看看我们将如何构建和实现其他动词和数据操作。

# RESTful 架构和最佳实践

在 Web API 设计领域，已经有一系列迭代的，有时是竞争的努力，以找到跨多个环境传递信息的标准系统和格式。

近年来，网站开发社区似乎已经—至少是暂时地—将 REST 作为事实上的方法。REST 在几年 SOAP 的主导之后出现，并引入了一种更简单的数据共享方法。

REST API 不受格式限制，通常可以缓存并通过 HTTP 或 HTTPS 传递。

开始时最重要的是遵守 HTTP 动词；最初为 Web 指定的那些动词在其原始意图上受到尊重。例如，HTTP 动词，如`DELETE`和`PATCH`，尽管非常明确地说明了它们的目的，但在多年的不使用后，REST 已成为使用正确方法的主要推动力。在 REST 之前，很常见看到`GET`和`POST`请求被互换使用来做各种事情，而这些事情本来是内置在 HTTP 设计中的。

在 REST 中，我们遵循**创建-读取-更新-删除**（CRUD）的方法来检索或修改数据。`POST`主要用于创建，`PUT`用于更新（尽管它也可以用于创建），熟悉的`GET`用于读取，`DELETE`用于删除，就是这样。

也许更重要的是，一个符合 RESTful 的 API 应该是无状态的。我们的意思是每个请求应该独立存在，而服务器不一定需要了解先前或潜在的未来请求。这意味着会话的概念在技术上违反了这一原则，因为我们会在服务器上存储某种状态。有些人持不同意见；我们将在以后详细讨论这个问题。

最后一点是关于 API URL 结构，因为方法已经作为请求的一部分嵌入到头部中，所以我们不需要在请求中明确表达它。

换句话说，我们不需要像`/api/blogs/delete/1`这样的东西。相反，我们可以简单地使用`DELETE`方法向`api/blogs/1`发出请求。

URL 结构没有严格的格式，您可能很快就会发现一些操作缺乏合理的 HTTP 动词，但简而言之，我们应该追求一些目标：

+   资源在 URL 中清晰表达

+   我们正确地利用 HTTP 动词

+   我们根据请求的类型返回适当的响应

我们在本章的目标是用我们的 API 实现前面三点。

如果有第四点，它会说我们与我们的 API 保持向后兼容。当您检查这里的 URL 结构时，您可能会想知道版本是如何处理的。这往往因组织而异，但一个很好的政策是保持最近的 URL 规范，并废弃显式版本的 URL。

例如，即使我们的评论可以在`/api/comments`中访问，但旧版本将在`/api/v2.0/comments`中找到，其中`2`显然代表我们的 API，就像它在版本`2.0`中存在一样。

### 注意

尽管在本质上相对简单且定义明确，REST 是一个常常争论的主题，有足够的模糊性，往往会引发很多辩论。请记住，REST 不是一个标准；例如，W3C 从未并且可能永远不会对 REST 是什么以及不是什么发表意见。如果您还没有，您将开始对什么是真正符合 REST 的内容产生一些非常强烈的看法。

# 创建我们的第一个 API 端点

鉴于我们希望从客户端和服务器之间访问数据，我们需要开始通过 API 公开其中的一些数据。

对我们来说最合理的事情是简单地读取，因为我们还没有方法在直接的 SQL 查询之外创建数据。我们在本章的开头就用我们的`APIPage`方法做到了这一点，通过`/api/pages/{UUID}`端点路由。

这对于`GET`请求非常有用，因为我们不会操纵数据，但是如果我们需要创建或修改数据，我们需要利用其他 HTTP 动词和 REST 方法。为了有效地做到这一点，现在是时候在我们的 API 中调查一些身份验证和安全性了。

# 实施安全性

当您考虑使用我们刚刚设计的 API 创建数据时，您首先会考虑什么问题？如果是安全性，那就太好了。访问数据并不总是没有安全风险，但当我们允许修改数据时，我们需要真正开始考虑安全性。

在我们的情况下，读取数据是完全无害的。如果有人可以通过`GET`请求访问我们所有的博客条目，那又有什么关系呢？好吧，我们可能有一篇关于禁运的博客，或者意外地在某些资源上暴露了敏感数据。

无论如何，安全性始终应该是一个关注点，即使是像我们正在构建的博客平台这样的小型个人项目。

有两种分离这些问题的方法：

+   我们的 API 请求是否安全且私密？

+   我们是否在控制对数据的访问？

让我们先解决第 2 步。如果我们想允许用户创建或删除信息，我们需要为他们提供对此的特定访问权限。

有几种方法可以做到这一点：

我们可以提供 API 令牌，允许短暂的请求窗口，这可以通过共享密钥进行验证。这是 Oauth 的本质；它依赖于共享密钥来验证加密编码的请求。没有共享密钥，请求及其令牌将永远不匹配，然后 API 请求可以被拒绝。

`cond`方法是一个简单的 API 密钥，这将我们带回到上述列表中的第 1 点。

如果我们允许明文 API 密钥，那么我们可能根本不需要安全性。如果我们的请求可以轻松地从线路上被嗅探到，那么甚至要求 API 密钥也没有多大意义。

这意味着无论我们选择哪种方法，我们的服务器都应该通过 HTTPS 提供 API。幸运的是，Go 提供了一种非常简单的方式来利用 HTTP 或 HTTPS 通过**传输层安全性**（**TLS**）；TLS 是 SSL 的后继者。作为 Web 开发人员，您必须已经熟悉 SSL，并且也意识到其安全问题的历史，最近是其易受 POODLE 漏洞攻击的问题，该漏洞于 2014 年曝光。

为了允许任一方法，我们需要有一个用户注册模型，这样我们就可以有新用户，他们可以有某种凭据来修改数据。为了调用 TLS 服务器，我们需要一个安全证书。由于这是一个用于实验的小项目，我们不会太担心具有高度信任级别的真实证书。相反，我们将自己生成。

创建自签名证书因操作系统而异，超出了本书的范围，因此让我们只看看 OS X 的方法。

自签名证书显然没有太多的安全价值，但它允许我们在不需要花费金钱或时间验证服务器所有权的情况下测试事物。对于任何希望被认真对待的证书，您显然需要做这些事情。

要在 OS X 中快速创建一组证书，请转到终端并输入以下三个命令：

```go
openssl genrsa -out key.pem
openssl req -new -key key.pem -out cert.pem
openssl req -x509 -days 365 -key key.pem -in cert.pem -out certificate.pem

```

在这个例子中，我使用 Ubuntu 上的 OpenSSL 生成了证书。

### 注意

注意：OpenSSL 预装在 OS X 和大多数 Linux 发行版上。如果您使用后者，请在寻找特定于 Linux 的说明之前尝试上述命令。如果您使用 Windows，特别是较新版本，如 8，您可以以多种方式执行此操作，但最可访问的方式可能是通过 MSDN 提供的 MakeCert 工具。

阅读有关 MakeCert 的更多信息[`msdn.microsoft.com/en-us/library/bfsktky3%28v=vs.110%29.aspx`](https://msdn.microsoft.com/en-us/library/bfsktky3%28v=vs.110%29.aspx)。

一旦您拥有证书文件，请将它们放在文件系统中的某个位置，而不要放在您可以访问的应用程序目录/目录中。

要从 HTTP 切换到 TLS，我们可以使用对这些证书文件的引用；除此之外，在我们的代码中基本上是相同的。让我们首先将证书添加到我们的代码中。

### 注意

注意：再次，您可以选择在同一服务器应用程序中维护 HTTP 和 TLS/HTTPS 请求，但我们将全面切换。

早些时候，我们通过监听以下行来启动我们的服务器：

```go
http.ListenAndServe(PORT, nil)
```

现在，我们需要稍微扩展一下。首先，让我们加载我们的证书：

```go
  certificates, err := tls.LoadX509KeyPair("cert.pem", "key.pem")
  tlsConf := tls.Config{Certificates: []tls.Certificate{certificates}}
  tls.Listen("tcp", PORT, &tlsConf)
```

### 注意

注意：如果您发现您的服务器似乎没有错误地运行，但无法保持运行；您的证书可能存在问题。尝试再次运行上述生成代码，并使用新证书进行操作。

# 使用 POST 创建数据

现在我们已经有了一个安全证书，我们可以为我们的 API 调用切换到 TLS，包括`GET`和其他请求。让我们现在这样做。请注意，您可以保留 HTTP 用于我们其余的端点，或者在这一点上也将它们切换。

### 注意

注意：现在大多数人普遍采用仅使用 HTTPS 的方式，这可能是未来保护您的应用程序的最佳方式。这不仅适用于 API 或者明文发送显式和敏感信息的地方，隐私是首要考虑的；主要提供商和服务都在强调随处使用 HTTPS 的价值。

让我们在我们的博客上添加一个匿名评论的简单部分：

```go
<div id="comments">
  <form action="/api/comments" method="POST">
    <input type="hidden" name="guid" value="{{Guid}}" />
    <div>
      <input type="text" name="name" placeholder="Your Name" />
    </div>
    <div>
      <input type="email" name="email" placeholder="Your Email" />
    </div>
    <div>
      <textarea name="comments" placeholder="Your Com-ments"></textarea>
    </div>
    <div>
      <input type="submit" value="Add Comments" />
    </div>
  </form>
</div>
```

这将允许任何用户在我们的网站上对我们的任何博客项目添加匿名评论，如下截图所示：

![使用 POST 创建数据](img/B04294_05_01.jpg)

但是安全性呢？目前，我们只想创建一个开放的评论区，任何人都可以在其中发布他们的有效、明晰的想法，以及他们的垃圾药方交易。我们稍后会担心锁定这一点；目前我们只想演示 API 和前端集成的并行。

显然，我们的数据库中需要一个`comments`表，所以在实现任何 API 之前，请确保创建该表。

```go
CREATE TABLE `comments` (
`id` int(11) unsigned NOT NULL AUTO_INCREMENT,
`page_id` int(11) NOT NULL,
`comment_guid` varchar(256) DEFAULT NULL,
`comment_name` varchar(64) DEFAULT NULL,
`comment_email` varchar(128) DEFAULT NULL,
`comment_text` mediumtext,
`comment_date` timestamp NULL DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `page_id` (`page_id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

有了表格，让我们把表单`POST`到 API 端点。为了创建一个通用和灵活的 JSON 响应，你可以添加一个`JSONResponse struct`，它基本上是一个哈希映射，如下所示：

```go
type JSONResponse struct {
  Fields map[string]string
}
```

然后我们需要一个 API 端点来创建评论，所以让我们在`main()`的路由下添加它：

```go
func APICommentPost(w http.ResponseWriter, r *http.Request) {
  var commentAdded bool
  err := r.ParseForm()
  if err != nil {
    log.Println(err.Error)
  }
  name := r.FormValue("name")
  email := r.FormValue("email")
  comments := r.FormValue("comments")

  res, err := database.Exec("INSERT INTO comments SET comment_name=?, comment_email=?, comment_text=?", name, email, comments)

  if err != nil {
    log.Println(err.Error)
  }

  id, err := res.LastInsertId()
  if err != nil {
    commentAdded = false
  } else {
    commentAdded = true
  }
  commentAddedBool := strconv.FormatBool(commentAdded)
  var resp JSONResponse
  resp.Fields["id"] = string(id)
  resp.Fields["added"] =  commentAddedBool
  jsonResp, _ := json.Marshal(resp)
  w.Header().Set("Content-Type", "application/json")
  fmt.Fprintln(w, jsonResp)
}
```

关于前面的代码有一些有趣的事情：

首先，注意我们使用`commentAdded`作为`string`而不是`bool`。我们这样做主要是因为 json marshaller 不能优雅地处理布尔值，而且直接从布尔值转换为字符串也是不可能的。我们还利用`strconv`及其`FormatBool`来处理这个转换。

您可能还注意到，对于这个例子，我们直接将表单`POST`到 API 端点。虽然这是演示数据进入数据库的有效方式，但在实践中使用它可能会强制一些 RESTful 反模式，比如启用重定向 URL 以返回到调用页面。

通过客户端利用一个常见的库或者通过`XMLHttpRequest`本地化来实现 Ajax 调用是更好的方法。

### 注意

注意：虽然内部函数/方法的名称在很大程度上是个人偏好的问题，但我们建议通过资源类型和请求方法来保持所有方法的区分。这里使用的实际约定并不重要，但在遍历代码时，诸如`APICommentPost`、`APICommentGet`、`APICommentPut`和`APICommentDelete`这样的命名方式可以更好地组织方法，使其更易读。

考虑到前端和后端的代码，我们可以看到这将如何呈现给访问我们第二篇博客文章的用户：

![使用 POST 创建数据](img/B04294_05_02.jpg)

正如前面提到的，实际在这里添加评论将直接发送表单到 API 端点，希望它会悄悄成功。

# 使用 PUT 修改数据

根据您询问的人，`PUT`和`POST`可以互换地用于创建记录。有些人认为两者都可以用于更新记录，大多数人认为两者都可以用于创建记录，只要给定一组变量。为了避免陷入一场有些混乱且常常带有政治色彩的辩论，我们将两者分开如下：

+   创建新记录：`POST`

+   更新现有记录，幂等性：`PUT`

根据这些准则，当我们希望更新资源时，我们将利用`PUT`动词。我们将允许任何人编辑评论，仅仅作为使用 REST `PUT`动词的概念验证。

在第六章*会话和 Cookie*中，我们将更加严格地限制这一点，但我们也希望能够通过 RESTful API 演示内容的编辑；因此，这将代表一个将来更安全和完整的不完整存根。

与创建新评论一样，在这里没有安全限制。任何人都可以创建评论，任何人都可以编辑它。至少在这一点上，这是博客软件的狂野西部。

首先，我们希望能够看到我们提交的评论。为此，我们需要对我们的`Page struct`进行微小的修改，并创建一个`Comment struct`以匹配我们的数据库结构：

```go
type Comment struct {
  Id    int
  Name   string
  Email  string
  CommentText string
}

type Page struct {
  Id         int
  Title      string
  RawContent string
  Content    template.HTML
  Date       string
  Comments   []Comment
  Session    Session
  GUID       string
}
```

由于之前发布的所有评论都没有任何真正的喧闹，博客文章页面上没有实际评论的记录。为了解决这个问题，我们将添加一个简单的`Comments`查询，并使用`.Scan`方法将它们扫描到一个`Comment struct`数组中。

首先，我们将在`ServePage`中添加查询：

```go
func ServePage(w http.ResponseWriter, r *http.Request) {
  vars := mux.Vars(r)
  pageGUID := vars["guid"]
  thisPage := Page{}
  fmt.Println(pageGUID)
  err := database.QueryRow("SELECT id,page_title,page_content,page_date FROM pages WHERE page_guid=?", pageGUID).Scan(&thisPage.Id, &thisPage.Title, &thisPage.RawContent, &thisPage.Date)
  thisPage.Content = template.HTML(thisPage.RawContent)
  if err != nil {
    http.Error(w, http.StatusText(404), http.StatusNotFound)
    log.Println(err)
    return
  }

  comments, err := database.Query("SELECT id, comment_name as Name, comment_email, comment_text FROM comments WHERE page_id=?", thisPage.Id)
  if err != nil {
    log.Println(err)
  }
  for comments.Next() {
    var comment Comment
    comments.Scan(&comment.Id, &comment.Name, &comment.Email, &comment.CommentText)
    thisPage.Comments = append(thisPage.Comments, comment)
  }

  t, _ := template.ParseFiles("templates/blog.html")
  t.Execute(w, thisPage)
}
```

现在我们已经将`Comments`打包进我们的`Page struct`中，我们可以在页面上显示**Comments**：

![使用 PUT 修改数据](img/B04294_05_03.jpg)

由于我们允许任何人进行编辑，我们将不得不为每个项目创建一个表单，这将允许修改。一般来说，HTML 表单只允许`GET`或`POST`请求，所以我们被迫使用`XMLHttpRequest`来发送这个请求。为了简洁起见，我们将利用 jQuery 及其`ajax()`方法。

首先，对于我们模板中的评论范围：

```go
{{range .Comments}}
  <div class="comment">
    <div>Comment by {{.Name}} ({{.Email}})</div>
    {{.CommentText}}

    <div class="comment_edit">
    <h2>Edit</h2>
    <form onsubmit="return putComment(this);">
      <input type="hidden" class="edit_id" value="{{.Id}}" />
      <input type="text" name="name" class="edit_name" placeholder="Your Name" value="{{.Name}}" />
     <input type="text" name="email" class="edit_email" placeholder="Your Email" value="{{.Email}}" />
      <textarea class="edit_comments" name="comments">{{.CommentText}}</textarea>
      <input type="submit" value="Edit" />
    </form>
    </div>
  </div>
{{end}}
```

然后，我们的 JavaScript 将使用`PUT`来处理表单：

```go
<script>
    function putComment(el) {
        var id = $(el).find('.edit_id');
        var name = $(el).find('.edit_name').val();
        var email = $(el).find('.edit_email').val();
        var text = $(el).find('.edit_comments').val();
        $.ajax({
            url: '/api/comments/' + id,
            type: 'PUT',
            succes: function(res) {
                alert('Comment Updated!');
            }
        });
        return false;
    }
</script>
```

为了处理这个使用`PUT`动词的调用，我们需要一个更新路由和函数。现在让我们添加它们：

```go
  routes.HandleFunc("/api/comments", APICommentPost).
    Methods("POST")
  routes.HandleFunc("/api/comments/{id:[\\w\\d\\-]+}", APICommentPut).
 Methods("PUT")

```

这样就可以启用一个路由，现在我们只需要添加相应的函数，它看起来会和我们的`POST`/`Create`方法非常相似：

```go
func APICommentPut(w http.ResponseWriter, r *http.Request) {
  err := r.ParseForm()
  if err != nil {
  log.Println(err.Error)
  }
  vars := mux.Vars(r)
  id := vars["id"]
  fmt.Println(id)
  name := r.FormValue("name")
  email := r.FormValue("email")
  comments := r.FormValue("comments")
  res, err := database.Exec("UPDATE comments SET comment_name=?, comment_email=?, comment_text=? WHERE comment_id=?", name, email, comments, id)
  fmt.Println(res)
  if err != nil {
    log.Println(err.Error)
  }

  var resp JSONResponse

  jsonResp, _ := json.Marshal(resp)
  w.Header().Set("Content-Type", "application/json")
  fmt.Fprintln(w, jsonResp)
}
```

简而言之，这将把我们的表单转变为基于评论内部 ID 的数据更新。正如前面提到的，这与我们的`POST`路由方法并没有完全不同，就像那个方法一样，它也不返回任何数据。

# 总结

在本章中，我们从独占服务器生成的 HTML 演示转变为利用 API 的动态演示。我们研究了 REST 的基础知识，并为我们的博客应用程序实现了一个 RESTful 接口。

虽然这可以使用更多客户端的修饰，但我们有`GET`/`POST`/`PUT`请求是功能性的，并允许我们为我们的博客文章创建、检索和更新评论。

在第六章，“会话和 Cookie”中，我们将研究用户认证、会话和 Cookie，以及如何将本章中我们所建立的基本组件应用到一些非常重要的安全参数上。在本章中，我们对评论进行了开放式的创建和更新；我们将在下一章中将其限制为唯一用户。

通过这一切，我们将把我们的概念验证评论管理转变为可以在生产中实际使用的东西。
