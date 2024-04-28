# 第九章 安全

在上一章中，我们看了如何存储应用程序生成的信息，以及向我们的套件添加单元测试，以确保应用程序的行为符合我们的期望，并在不符合期望时诊断错误。

在那一章中，我们没有为我们的博客应用程序添加太多功能；所以现在让我们回到那里。我们还将把本章的一些日志记录和测试功能扩展到我们的新功能中。

到目前为止，我们一直在开发一个 Web 应用程序的框架，该应用程序实现了博客数据和用户提交的评论的一些基本输入和输出。就像任何公共网络服务器一样，我们的服务器也容易受到各种攻击。

这些问题并不是 Go 独有的，但我们有一系列工具可以实施最佳实践，并扩展我们的服务器和应用程序以减轻常见问题。

在构建一个公开访问的网络应用程序时，一个快速简便的常见攻击向量参考指南是**开放网络应用程序安全项目**（**OWASP**），它提供了一个定期更新的最关键的安全问题清单。OWASP 可以在[`www.owasp.org/`](https://www.owasp.org/)找到。其十大项目编制了最常见和/或最关键的网络安全问题。虽然它不是一个全面的清单，并且在更新之间容易过时，但在编制潜在攻击向量时仍然是一个很好的起点。

多年来，一些最普遍的攻击向量不幸地一直存在；尽管安全专家一直在大声疾呼其严重性。有些攻击向量在 Web 上的曝光迅速减少（比如注入），但它们仍然会长期存在，甚至在遗留应用程序逐渐淘汰的情况下。

以下是 2013 年末最近的十大漏洞中的四个概述，其中一些我们将在本章中讨论：

+   **注入**：任何未经信任的数据有机会在不转义的情况下被处理，从而允许数据操纵或访问数据或系统，通常不会公开暴露。最常见的是 SQL 注入。

+   **破坏的身份验证**：这是由于加密算法不佳，密码要求不严格，会话劫持是可行的。

+   **XSS**：跨站点脚本允许攻击者通过在另一个站点上注入和执行脚本来访问敏感信息。

+   **跨站点请求伪造**：与 XSS 不同，这允许攻击向量来自另一个站点，但它会欺骗用户在另一个站点上完成某些操作。

虽然其他攻击向量从相关到不相关都有，但值得评估我们没有涵盖的攻击向量，看看其他可能存在利用的地方。

首先，我们将看一下使用 Go 在应用程序中实现和强制使用 HTTPS 的最佳方法。

# 到处使用 HTTPS - 实施 TLS

在第五章中，*前端与 RESTful API 的集成*，我们讨论了创建自签名证书并在我们的应用程序中使用 HTTPS/TLS。但让我们快速回顾一下为什么这对于我们的应用程序和 Web 的整体安全性如此重要。

首先，简单的 HTTP 通常不会为流量提供加密，特别是对于重要的请求头值，如 cookie 和查询参数。我们在这里说通常是因为 RFC 2817 确实指定了在 HTTP 协议上使用 TLS 的系统，但几乎没有使用。最重要的是，它不会给用户提供必要的明显反馈，以注册网站的安全性。

其次，HTTP 流量容易受到中间人攻击。

另一个副作用是：Google（也许其他搜索引擎）开始偏爱 HTTPS 流量而不是不太安全的对应物。

直到相对最近，HTTPS 主要被限制在电子商务应用程序中，但利用 HTTP 的不足的攻击的可用性和普遍性的增加——如侧面攻击和中间人攻击——开始将 Web 的大部分推向 HTTPS。

您可能已经听说过由此产生的运动和座右铭**HTTPS 无处不在**，这也渗透到了强制网站使用实施最安全可用协议的浏览器插件中。

我们可以做的最简单的事情之一是扩展第六章中的工作，*会话和 Cookie*是要求所有流量通过 HTTPS 重新路由 HTTP 流量。还有其他方法可以做到这一点，正如我们将在本章末看到的那样，但它可以相当简单地实现。

首先，我们将实现一个`goroutine`来同时为我们的 HTTPS 和 HTTP 流量提供服务，分别使用`tls.ListenAndServe`和`http.ListenAndServe`：

```go
  var wg sync.WaitGroup
  wg.Add(1)
  go func() {
    http.ListenAndServe(PORT, http.HandlerFunc(redirectNonSecure))
    wg.Done()
  }()
  wg.Add(1)
  go func() {
    http.ListenAndServeTLS(SECUREPORT, "cert.pem", "key.pem", routes)
    wg.Done()
  }()

  wg.Wait()
```

这假设我们将一个`SECUREPORT`常量设置为`":443"`，就像我们将`PORT`设置为`":8080"`一样，或者您选择的任何其他端口。没有什么可以阻止您在 HTTPS 上使用另一个端口；这里的好处是浏览器默认将`https://`请求重定向到端口`443`，就像它将 HTTP 请求重定向到端口`80`，有时会回退到端口`8080`一样。请记住，在许多情况下，您需要以 sudo 或管理员身份运行以使用低于`1000`的端口启动。

您会注意到在前面的示例中，我们使用了一个专门用于 HTTP 流量的处理程序`redirectNonSecure`。这实现了一个非常基本的目的，正如您在这里所看到的：

```go
func redirectNonSecure(w http.ResponseWriter, r *http.Request) {
  log.Println("Non-secure request initiated, redirecting.")
  redirectURL := "https://" + serverName + r.RequestURI
  http.Redirect(w, r, redirectURL, http.StatusMovedPermanently)
}
```

在这里，`serverName`被明确设置。

从请求中获取域名或服务器名称可能存在一些潜在问题，因此最好在可能的情况下明确设置这一点。

在这里添加的另一个非常有用的部分是**HTTP 严格传输安全**（**HSTS**），这种方法与兼容的消费者结合使用，旨在减轻协议降级攻击（如强制/重定向到 HTTP）。

这只是一个 HTTPS 标头，当被使用时，将自动处理并强制执行`https://`请求，以替代使用较不安全的协议。

OWASP 建议为此标头使用以下设置：

```go
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

请注意，此标头在 HTTP 上被忽略。

# 防止 SQL 注入

尽管注入仍然是当今 Web 上最大的攻击向量之一，但大多数语言都有简单而优雅的方法来通过准备好的语句和经过消毒的输入来防止或大大减轻留下易受攻击的 SQL 注入的可能性。

但即使使用提供这些服务的语言，仍然有机会留下漏洞的空间。

无论是在 Web 上还是在服务器上或独立的可执行文件中，任何软件开发的核心原则都是不要相信从外部（有时是内部）获取的输入数据。

这个原则对于任何语言都是正确的，尽管有些语言通过准备好的查询或抽象（如对象关系映射（ORM））使与数据库的交互更安全和/或更容易。

从本质上讲，Go 没有任何 ORM，因为从技术上讲，甚至没有一个 O（对象）（Go 不是纯粹的面向对象的），很难在这个领域复制许多面向对象语言所拥有的东西。

然而，有许多第三方库试图通过接口和结构来强制 ORM，但是很多这些都可以很容易地手工编写，因为您可能比任何库更了解您的模式和数据结构，即使是在抽象的意义上。

然而，对于 SQL，Go 具有几乎支持 SQL 的任何数据库的强大和一致的接口。

为了展示 SQL 注入漏洞如何在 Go 应用程序中简单地出现，我们将比较原始的 SQL 查询和准备好的语句。

当我们从数据库中选择页面时，我们使用以下查询：

```go
err := database.QueryRow("SELECT page_title,page_content,page_date FROM pages WHERE page_guid="+requestGUID, pageGUID).Scan(&thisPage.Title, &thisPage.Content, &thisPage.Date)
```

这向我们展示了如何通过接受未经处理的用户输入来打开您的应用程序以进行注入漏洞。在这种情况下，任何请求类似于

`/page/foo;delete from pages`理论上可以迅速清空你的`pages`表。

我们在路由器级别有一些初步的消毒工作，这在这方面有所帮助。由于我们的 mux 路由只包括字母数字字符，我们可以避免一些需要被转义的字符被路由到我们的`ServePage`或`APIPage`处理程序中：

```go
  routes.HandleFunc("/page/{guid:[0-9a-zA\\-]+}", ServePage)
  routes.HandleFunc("/api/page/{id:[\\w\\d\\-]+}", APIPage).
    Methods("GET").
    Schemes("https")
```

然而，这并不是一个绝对可靠的方法。前面的查询接受了原始输入并将其附加到 SQL 查询中，但是我们可以在 Go 中使用参数化、准备好的查询来更好地处理这个问题。以下是我们最终使用的内容：

```go
  err := database.QueryRow("SELECT page_title,page_content,page_date FROM pages WHERE page_guid=?", pageGUID).Scan(&thisPage.Title, &thisPage.Content, &thisPage.Date)
  if err != nil {
    http.Error(w, http.StatusText(404), http.StatusNotFound)
    log.Println("Couldn't get page!")
    return
  }
```

这种方法在 Go 的任何查询接口中都可以使用，它使用`?`来代替值作为可变参数的查询：

```go
res, err := db.Exec("INSERT INTO table SET field=?, field2=?", value1, value2)
rows, err := db.Query("SELECT * FROM table WHERE field2=?",value2)
statement, err := db.Prepare("SELECT * FROM table WHERE field2=?",value2)
row, err := db.QueryRow("SELECT * FROM table WHERE field=?",value1)
```

虽然所有这些在 SQL 世界中有着略微不同的目的，但它们都以相同的方式实现了准备好的查询。

# 防止跨站脚本攻击

我们简要提到了跨站脚本攻击和限制它作为一种向量，这使得您的应用程序对所有用户更安全，而不受少数不良分子的影响。问题的关键在于一个用户能够添加危险内容，并且这些内容将被显示给用户，而不会清除使其危险的方面。

最终你在这里有一个选择——在输入时对数据进行消毒，或者在呈现给其他用户时对数据进行消毒。

换句话说，如果有人产生了一个包含`script`标签的评论文本块，你必须小心阻止其他用户的浏览器渲染它。你可以选择保存原始 HTML，然后在输出渲染时剥离所有或只剥离敏感标签。或者，你可以在输入时对其进行编码。

没有标准答案；然而，您可能会发现遵循前一种方法有价值，即接受任何内容并对输出进行消毒。

这两种方法都存在风险，但这种方法允许您保留消息的原始意图，如果您选择在以后改变您的方法。缺点是当然你可能会意外地允许一些原始数据通过未经处理的：

```go
template.HTMLEscapeString(string)
template.JSEscapeString(inputData)
```

第一个函数将获取数据并删除 HTML 的格式，以产生用户输入的消息的纯文本版本。

第二个函数将做类似的事情，但是针对 JavaScript 特定的值。您可以使用类似以下示例的快速脚本很容易地测试这些功能：

```go
package main

import (
  "fmt"
  "github.com/gorilla/mux"
  "html/template"
  "net/http"
)

func HTMLHandler(w http.ResponseWriter, r *http.Request) {
  input := r.URL.Query().Get("input")
  fmt.Fprintln(w, input)
}

func HTMLHandlerSafe(w http.ResponseWriter, r *http.Request) {
  input := r.URL.Query().Get("input")
  input = template.HTMLEscapeString(input)
  fmt.Fprintln(w, input)
}

func JSHandler(w http.ResponseWriter, r *http.Request) {
  input := r.URL.Query().Get("input")
  fmt.Fprintln(w, input)
}

func JSHandlerSafe(w http.ResponseWriter, r *http.Request) {
  input := r.URL.Query().Get("input")
  input = template.JSEscapeString(input)
  fmt.Fprintln(w, input)
}

func main() {
  router := mux.NewRouter()
  router.HandleFunc("/html", HTMLHandler)
  router.HandleFunc("/js", JSHandler)
  router.HandleFunc("/html_safe", HTMLHandlerSafe)
  router.HandleFunc("/js_safe", JSHandlerSafe)
  http.ListenAndServe(":8080", router)
}
```

如果我们从不安全的端点请求，我们将得到我们的数据返回：

![防止跨站脚本攻击](img/B04294_09_01.jpg)

将此与`/html_safe`进行比较，后者会自动转义输入，您可以在其中看到内容以其经过处理的形式：

![防止跨站脚本攻击](img/B04294_09_02.jpg)

这一切都不是绝对可靠的，但如果您选择按用户提交的方式接受输入数据，您将希望寻找一些方法来在结果显示时传递这些信息，而不会让其他用户受到跨站脚本攻击的威胁。

# 防止跨站请求伪造（CSRF）

虽然我们在这本书中不会深入讨论 CSRF，但总的来说，它是一系列恶意行为者可以使用的方法，以欺骗用户在另一个站点上执行不需要的操作。

由于它至少在方法上与跨站脚本攻击有关，现在谈论它是值得的。

这最明显的地方是在表单中；把它想象成一个允许你发送推文的 Twitter 表单。如果第三方强制代表用户在没有他们同意的情况下请求，想象一下类似这样的情况：

```go
<h1>Post to our guestbook (and not twitter, we swear!)</h1>
  <form action="https://www.twitter.com/tweet" method="POST">
  <input type="text" placeholder="Your Name" />
  <textarea placeholder="Your Message"></textarea>
  <input type="hidden" name="tweet_message" value="Make sure to check out this awesome, malicious site and post on their guestbook" />
  <input type="submit" value="Post ONLY to our guestbook" />
</form>
```

没有任何保护，任何发布到这个留言簿的人都会无意中帮助传播垃圾邮件到这次攻击中。

显然，Twitter 是一个成熟的应用程序，早就处理了这个问题，但你可以得到一个大致的想法。你可能会认为限制引用者会解决这个问题，但这也可以被欺骗。

最简单的解决方案是为表单提交生成安全令牌，这可以防止其他网站能够构造有效的请求。

当然，我们的老朋友 Gorilla 在这方面也提供了一些有用的工具。最相关的是`csrf`包，其中包括用于生成请求令牌的工具，以及预先制作的表单字段，如果违反或忽略将产生`403`。

生成令牌的最简单方法是将其作为您的处理程序将用于生成模板的接口的一部分，就像我们的`ApplicationAuthenticate()`处理程序一样：

```go
    Authorize.TemplateTag = csrf.TemplateField(r)
    t.ExecuteTemplate(w, "signup_form.tmpl", Authorize)
```

此时，我们需要在我们的模板中公开`{{.csrfField}}`。要进行验证，我们需要将其链接到我们的`ListenAndServe`调用：

```go
    http.ListenAndServe(PORT, csrf.Protect([]byte("32-byte-long-auth-key"))(r))
```

# 保护 cookie

我们之前研究过的攻击向量之一是会话劫持，我们在 HTTP 与 HTTPS 的背景下讨论了这个问题，以及其他人如何看到网站身份关键信息的方式。

对于许多非 HTTPS 应用程序来说，在公共网络上找到这些数据非常简单，这些应用程序利用会话作为确定性 ID。事实上，一些大型应用程序允许会话 ID 在 URL 中传递。

在我们的应用程序中，我们使用了 Gorilla 的`securecookie`包，它不依赖于 HTTPS，因为 cookie 值本身是使用 HMAC 哈希编码和验证的。

生成密钥本身可以非常简单，就像我们的应用程序和`securecookie`文档中所演示的那样：

```go
var hashKey = []byte("secret hash key")
var blockKey = []byte("secret-er block key")
var secureKey = securecookie.New(hashKey, blockKey)
```

### 注意

有关 Gorilla 的`securecookie`包的更多信息，请参见：[`www.gorillatoolkit.org/pkg/securecookie`](http://www.gorillatoolkit.org/pkg/securecookie)

目前，我们应用程序的服务器首先使用 HTTPS 和安全 cookie，这意味着我们可能对在 cookie 本身中存储和识别数据感到更有信心。我们大部分的创建/更新/删除操作都是在 API 级别进行的，这仍然实现了会话检查，以确保我们的用户已经通过身份验证。

# 使用 secure 中间件

在本章中，快速实施一些安全修复（和其他内容）的更有帮助的软件包之一是 Cory Jacobsen 的一个软件包，贴心地称为`secure`。

Secure 提供了许多有用的实用程序，例如 SSL 重定向（正如我们在本章中实现的那样），允许的主机，HSTS 选项和 X-Frame-Options 的简写，用于防止您的网站被加载到框架中。

这其中涵盖了我们在本章中研究的一些主题，并且基本上是最佳实践。作为一个中间件，secure 可以是一种快速覆盖这些最佳实践的简单方法。

### 注意

要获取`secure`，只需在[github.com/unrolled/secure](http://github.com/unrolled/secure)上获取它。

# 摘要

虽然本章并不是对 Web 安全问题和解决方案的全面审查，但我们希望解决一些由 OWASP 和其他人提出的最大和最常见的向量之一。

在本章中，我们涵盖或审查了防止这些问题渗入您的应用程序的最佳实践。

在第十章中，*缓存、代理和性能改进*，我们将讨论如何使您的应用程序在流量增加的同时保持可扩展性和速度。
