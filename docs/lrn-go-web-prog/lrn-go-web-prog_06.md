# 第六章 会话和 Cookie

我们的应用现在开始变得更加真实；在上一章中，我们为它们添加了一些 API 和客户端接口。

在我们应用的当前状态下，我们已经添加了`/api/comments`、`/api/comments/[id]`、`/api/pages`和`/api/pages/[id]`，这样我们就可以以 JSON 格式获取和更新我们的数据，并使应用更适合 Ajax 和客户端访问。

虽然我们现在可以通过我们的 API 直接添加评论和编辑评论，但是对谁可以执行这些操作没有任何限制。在本章中，我们将探讨限制对某些资产的访问、建立身份和在拥有它们时进行安全认证的方法。

最终，我们应该能够让用户注册和登录，并利用会话、cookie 和闪存消息以安全的方式在我们的应用中保持用户状态。

# 设置 cookie

创建持久内存跨用户会话的最常见、基本和简单的方式是利用 cookie。

Cookie 提供了一种在请求、URL 端点甚至域之间共享状态信息的方式，并且它们已经被以各种可能的方式使用（和滥用）。

它们通常用于跟踪身份。当用户登录到一个服务时，后续的请求可以通过利用存储在 cookie 中的会话信息来访问前一个请求的某些方面（而不需要重复查找或登录模块）。

如果你熟悉其他语言中 cookie 的实现，基本的`struct`会很熟悉。即便如此，以下相关属性与向客户端呈现 cookie 的方式基本一致：

```go
type Cookie struct {
  Name       string
  Value      string
  Path       string
  Domain     string
  Expires    time.Time
  RawExpires string
  MaxAge   int
  Secure   bool
  HttpOnly bool
  Raw      string
  Unparsed []string
}
```

对于一个非常基本的`struct`来说，这是很多属性，所以让我们专注于重要的属性。

`Name`属性只是 cookie 的键。`Value`属性代表其内容，`Expires`是一个`Time`值，表示 cookie 应该被浏览器或其他无头接收者刷新的时间。这就是你在 Go 中设置一个有效 cookie 所需要的一切。

除了基础知识，如果你想要限制 cookie 的可访问性，你可能会发现设置`Path`、`Domain`和`HttpOnly`是有用的。

# 捕获用户信息

当一个具有有效会话和/或 cookie 的用户尝试访问受限数据时，我们需要从用户的浏览器中获取它。

一个会话本身就是一个在网站上的单个会话。它并不会自然地无限期持续，所以我们需要留下一个线索，但我们也希望留下一个相对安全的线索。

例如，我们绝不希望在 cookie 中留下关键的用户信息，比如姓名、地址、电子邮件等等。

然而，每当我们有一些标识信息时，我们都会留下一些不良行为的可能性——在这种情况下，我们可能会留下代表我们会话 ID 的会话标识符。在这种情况下，这个向量允许获得这个 cookie 的人以我们其中一个用户的身份登录并更改信息，查找账单详情等等。

这些类型的物理攻击向量远远超出了这个（以及大多数）应用的范围，而且在很大程度上，这是一个让步，即如果有人失去了对他们的物理机器的访问权限，他们也可能会遭受账户被破坏的风险。

在这里我们想要做的是确保我们不会在明文或没有安全连接的情况下传输个人或敏感信息。我们将在第九章 *安全*中介绍如何设置 TLS，所以在这里我们想要专注于限制我们在 cookie 中存储的信息量。

## 创建用户

在上一章中，我们允许非授权的请求通过`POST`命中我们的 REST API 来创建新的评论。在互联网上待了一段时间的人都知道一些真理，比如：

1.  评论部分通常是任何博客或新闻帖子中最有毒的部分

1.  即使用户必须以非匿名的方式进行身份验证，步骤 1 也是正确的

现在，让我们限制评论部分，以确保用户已注册并已登录。

我们现在不会深入探讨身份验证的安全方面，因为我们将在第九章 *安全*中更深入地讨论这个问题。

首先，在我们的数据库中添加一个`users`表：

```go
CREATE TABLE `users` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `user_name` varchar(32) NOT NULL DEFAULT '',
  `user_guid` varchar(256) NOT NULL DEFAULT '',
  `user_email` varchar(128) NOT NULL DEFAULT '',
  `user_password` varchar(128) NOT NULL DEFAULT '',
  `user_salt` varchar(128) NOT NULL DEFAULT '',
  `user_joined_timestamp` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

我们当然可以深入研究用户信息，但这已经足够让我们开始了。正如前面提到的，我们不会深入研究安全性，所以现在我们只是为密码生成一个哈希值，不用担心盐。

最后，为了在应用程序中启用会话和用户，我们将对我们的 structs 进行一些更改：

```go
type Page struct {
  Id         int
  Title      string
  RawContent string
  Content    template.HTML
  Date       string
  Comments   []Comment
  Session    Session
}

type User struct {
  Id   int
  Name string
}

type Session struct {
  Id              string
  Authenticated   bool
  Unauthenticated bool
  User            User
}
```

以下是用于注册和登录的两个存根处理程序。同样，我们并没有将全部精力投入到将它们完善成健壮的东西，我们只是想打开一点门。

## 启用会话

除了存储用户本身之外，我们还需要一种持久性内存的方式来访问我们的 cookie 数据。换句话说，当用户的浏览器会话结束并且他们回来时，我们将验证和调和他们的 cookie 值与我们数据库中的值。

使用此 SQL 创建`sessions`表：

```go
CREATE TABLE `sessions` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `session_id` varchar(256) NOT NULL DEFAULT '',
  `user_id` int(11) DEFAULT NULL,
  `session_start` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `session_update` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `session_active` tinyint(1) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `session_id` (`session_id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

最重要的值是`user_id`、`session_id`和更新和开始的时间戳。我们可以使用后两者来决定在一定时间后会话是否实际上是有效的。这是一个很好的安全实践，仅仅因为用户有一个有效的 cookie 并不一定意味着他们应该保持身份验证，特别是如果您没有使用安全连接。

## 让用户注册

为了让用户能够自行创建账户，我们需要一个注册和登录的表单。现在，大多数类似的系统都会进行一些多因素身份验证，以允许用户备份系统进行检索，并验证用户的真实性和唯一性。我们会做到这一点，但现在让我们尽可能简单。

我们将设置以下端点，允许用户`POST`注册和登录表单：

```go
  routes.HandleFunc("/register", RegisterPOST).
    Methods("POST").
    Schemes("https")
  routes.HandleFunc("/login", LoginPOST).
    Methods("POST").
    Schemes("https")
```

请记住，这些目前设置为 HTTPS 方案。如果您不使用 HTTPS，请删除`HandleFunc`注册的部分。

由于我们只向未经身份验证的用户显示以下视图，我们可以将它们放在我们的`blog.html`模板中，并将它们包裹在`{{if .Session.Unauthenticated}} … {{end}}`模板片段中。我们在应用程序中的`Session` `struct`下定义了`.Unauthenticated`和`.Authenticated`，如下例所示：

```go
{{if .Session.Unauthenticated}}<form action="/register" method="POST">
  <div><input type="text" name="user_name" placeholder="User name" /></div>
  <div><input type="email" name="user_email" placeholder="Your email" /></div>
  <div><input type="password" name="user_password" placeholder="Password" /></div>
  <div><input type="password" name="user_password2" placeholder="Password (repeat)" /></div>
  <div><input type="submit" value="Register" /></div>
</form>{{end}}
```

和我们的`/register`端点：

```go
func RegisterPOST(w http.ResponseWriter, r *http.Request) {
  err := r.ParseForm()
  if err != nil {
    log.Fatal(err.Error)
  }
  name := r.FormValue("user_name")
  email := r.FormValue("user_email")
  pass := r.FormValue("user_password")
  pageGUID := r.FormValue("referrer")
  // pass2 := r.FormValue("user_password2")
  gure := regexp.MustCompile("[^A-Za-z0-9]+")
  guid := gure.ReplaceAllString(name, "")
  password := weakPasswordHash(pass)

  res, err := database.Exec("INSERT INTO users SET user_name=?, user_guid=?, user_email=?, user_password=?", name, guid, email, password)
  fmt.Println(res)
  if err != nil {
    fmt.Fprintln(w, err.Error)
  } else {
    http.Redirect(w, r, "/page/"+pageGUID, 301)
  }
}
```

请注意，由于多种原因，这种方式并不优雅。如果密码不匹配，我们不会检查并向用户报告。如果用户已经存在，我们也不会告诉他们注册失败的原因。我们会解决这个问题，但现在我们的主要目的是生成一个会话。

供参考，这是我们的`weakPasswordHash`函数，它只用于生成测试哈希：

```go
func weakPasswordHash(password string) []byte {
  hash := sha1.New()
  io.WriteString(hash, password)
  return hash.Sum(nil)
}
```

## 让用户登录

用户可能已经注册过了；在这种情况下，我们也希望在同一个页面上提供登录机制。这显然可以根据更好的设计考虑来实现，但我们只是想让它们都可用：

```go
<form action="/login" method="POST">
  <div><input type="text" name="user_name" placeholder="User name" /></div>
  <div><input type="password" name="user_password" placeholder="Password" /></div>
  <div><input type="submit" value="Log in" /></div>
</form>
```

然后我们将需要为每个 POST 表单设置接收端点。我们在这里也不会进行太多的验证，但我们也没有验证会话的位置。

# 启动服务器端会话

在 Web 上验证用户并保存其状态的最常见方式之一是通过会话。您可能还记得我们在上一章中提到过 REST 是无状态的，这主要是因为 HTTP 本身是无状态的。

如果您考虑一下，要建立与 HTTP 一致的状态，您需要包括一个 cookie 或 URL 参数或其他不是协议本身内置的东西。

会话是使用通常不是完全随机但足够唯一以避免大多数逻辑和合理情况下的冲突的唯一标识符创建的。当然，这并不是绝对的，当然，有很多（历史上的）会话令牌劫持的例子与嗅探无关。

作为一个独立的过程，会话支持在 Go 核心中并不存在。鉴于我们在服务器端有一个存储系统，这有点无关紧要。如果我们为生成服务器密钥创建一个安全的过程，我们可以将它们存储在安全的 cookie 中。

但生成会话令牌并不完全是微不足道的。我们可以使用一组可用的加密方法来实现这一点，但是由于会话劫持是一种非常普遍的未经授权进入系统的方式，这可能是我们应用程序中的一个不安全的点。

由于我们已经在使用 Gorilla 工具包，好消息是我们不必重新发明轮子，已经有一个强大的会话系统。

我们不仅可以访问服务器端会话，而且还可以获得一个非常方便的工具，用于会话中的一次性消息。这些工作方式与消息队列有些类似，一旦数据进入其中，当数据被检索时，闪存消息就不再有效。

## 创建存储

要使用 Gorilla 会话，我们首先需要调用一个 cookie 存储，它将保存我们想要与用户关联的所有变量。您可以通过以下代码很容易地测试这一点：

```go
package main

import (
  "fmt"
  "github.com/gorilla/sessions"
  "log"
  "net/http"
)

func cookieHandler(w http.ResponseWriter, r *http.Request) {
  var cookieStore = sessions.NewCookieStore([]byte("ideally, some random piece of entropy"))
  session, _ := cookieStore.Get(r, "mystore")
  if value, exists := session.Values["hello"]; exists {
    fmt.Fprintln(w, value)
  } else {
    session.Values["hello"] = "(world)"
    session.Save(r, w)
    fmt.Fprintln(w, "We just set the value!")
  }
}

func main() {
  http.HandleFunc("/test", cookieHandler)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

第一次访问您的 URL 和端点时，您将看到**我们刚刚设置了值！**，如下面的截图所示：

![创建存储](img/B04294_06_04.jpg)

在第二个请求中，您应该看到**(world)**，如下面的截图所示：

![创建存储](img/B04294_06_05.jpg)

这里有几点需要注意。首先，在通过`io.Writer`（在这种情况下是`ResponseWriter w`）发送任何其他内容之前，您必须设置 cookies。如果您交换这些行：

```go
    session.Save(r, w)
    fmt.Fprintln(w, "We just set the value!")
```

您可以看到这个过程。您永远不会得到设置为 cookie 存储的值。

现在，让我们将其应用到我们的应用程序中。我们将在对`/login`或`/register`的任何请求之前初始化一个会话存储。

我们将初始化一个全局的`sessionStore`：

```go
var database *sql.DB
var sessionStore = sessions.NewCookieStore([]byte("our-social-network-application"))
```

也可以自由地将这些分组在`var()`中。接下来，我们将创建四个简单的函数，用于获取活动会话，更新当前会话，生成会话 ID，并评估现有的 cookie。这将允许我们通过 cookie 的会话 ID 检查用户是否已登录，并启用持久登录。

首先是`getSessionUID`函数，如果会话已经存在，它将返回用户的 ID：

```go
func getSessionUID(sid string) int {
  user := User{}
  err := database.QueryRow("SELECT user_id FROM sessions WHERE session_id=?", sid).Scan(user.Id)
  if err != nil {
    fmt.Println(err.Error)
    return 0
  }
  return user.Id
}
```

接下来是更新函数，它将在每个面向前端的请求中调用，从而使时间戳更新或者在尝试新的登录时包含用户 ID：

```go
func updateSession(sid string, uid int) {
  const timeFmt = "2006-01-02T15:04:05.999999999"
  tstamp := time.Now().Format(timeFmt)
  _, err := database.Exec("INSERT INTO sessions SET session_id=?, user_id=?, session_update=? ON DUPLICATE KEY UPDATE user_id=?, session_update=?", sid, uid, tstamp, uid, tstamp)
  if err != nil {
    fmt.Println(err.Error)
  }
}
```

一个重要的部分是能够生成一个强大的随机字节数组（转换为字符串），以允许唯一的标识符。我们可以通过以下`generateSessionId()`函数来实现：

```go
func generateSessionId() string {
  sid := make([]byte, 24)
  _, err := io.ReadFull(rand.Reader, sid)
  if err != nil {
    log.Fatal("Could not generate session id")
  }
  return base64.URLEncoding.EncodeToString(sid)
}
```

最后，我们有一个函数，它将在每个请求中被调用，检查 cookie 的会话是否存在，如果不存在则创建一个。

```go
func validateSession(w http.ResponseWriter, r *http.Request) {
  session, _ := sessionStore.Get(r, "app-session")
  if sid, valid := session.Values["sid"]; valid {
    currentUID := getSessionUID(sid.(string))
    updateSession(sid.(string), currentUID)
    UserSession.Id = string(currentUID)
  } else {
    newSID := generateSessionId()
    session.Values["sid"] = newSID
    session.Save(r, w)
    UserSession.Id = newSID
    updateSession(newSID, 0)
  }
  fmt.Println(session.ID)
}
```

这是建立在有一个全局的`Session struct`的基础上的，在这种情况下定义如下：

```go
var UserSession Session
```

这让我们只剩下一个部分——在我们的`ServePage()`方法和`LoginPost()`方法上调用`validateSession()`，然后在后者上验证密码并在成功登录尝试时更新我们的会话：

```go
func LoginPOST(w http.ResponseWriter, r *http.Request) {
  validateSession(w, r)
```

在我们之前定义的对表单值的检查中，如果找到一个有效的用户，我们将直接更新会话：

```go
  u := User{}
  name := r.FormValue("user_name")
  pass := r.FormValue("user_password")
  password := weakPasswordHash(pass)
  err := database.QueryRow("SELECT user_id, user_name FROM users WHERE user_name=? and user_password=?", name, password).Scan(&u.Id, &u.Name)
  if err != nil {
    fmt.Fprintln(w, err.Error)
    u.Id = 0
    u.Name = ""
  } else {
    updateSession(UserSession.Id, u.Id)
    fmt.Fprintln(w, u.Name)
  }
```

## 利用闪存消息

正如本章前面提到的，Gorilla 会话提供了一种简单的系统，用于在请求之间利用基于单次使用和基于 cookie 的数据传输。

闪存消息背后的想法与浏览器/服务器消息队列并没有太大的不同。它最常用于这样的过程：

+   一个表单被提交

+   数据被处理

+   发起一个头部重定向

+   生成的页面需要一些关于`POST`过程（成功、错误）的信息访问

在这个过程结束时，应该删除消息，以便消息不会在其他地方错误地重复。Gorilla 使这变得非常容易，我们很快就会看到，但是展示一下如何在原生 Go 中实现这一点是有意义的。

首先，我们将创建一个包含起始点处理程序`startHandler`的简单 HTTP 服务器：

```go
package main

import (
  "fmt"
  "html/template"
  "log"
  "net/http"
  "time"
)

var (
  templates = template.Must(template.ParseGlob("templates/*"))
  port      = ":8080"
)

func startHandler(w http.ResponseWriter, r *http.Request) {
  err := templates.ExecuteTemplate(w, "ch6-flash.html", nil)
  if err != nil {
    log.Fatal("Template ch6-flash missing")
  }
}
```

我们在这里没有做任何特别的事情，只是渲染我们的表单：

```go
func middleHandler(w http.ResponseWriter, r *http.Request) {
  cookieValue := r.PostFormValue("message")
  cookie := http.Cookie{Name: "message", Value: "message:" + cookieValue, Expires: time.Now().Add(60 * time.Second), HttpOnly: true}
  http.SetCookie(w, &cookie)
  http.Redirect(w, r, "/finish", 301)
}
```

我们的`middleHandler`演示了通过`Cookie struct`创建 cookie，正如本章前面所述。这里没有什么重要的要注意，除了您可能希望将到期时间延长一点，以确保在请求之间没有办法使 cookie 过期（自然地）：

```go
func finishHandler(w http.ResponseWriter, r *http.Request) {
  cookieVal, _ := r.Cookie("message")

  if cookieVal != nil {
    fmt.Fprintln(w, "We found: "+string(cookieVal.Value)+", but try to refresh!")
    cookie := http.Cookie{Name: "message", Value: "", Expires: time.Now(), HttpOnly: true}
    http.SetCookie(w, &cookie)
  } else {
    fmt.Fprintln(w, "That cookie was gone in a flash")
  }

}
```

`finishHandler`函数执行闪存消息的魔术——仅在找到值时删除 cookie。这确保了 cookie 是一次性可检索的值：

```go
func main() {

  http.HandleFunc("/start", startHandler)
  http.HandleFunc("/middle", middleHandler)
  http.HandleFunc("/finish", finishHandler)
  log.Fatal(http.ListenAndServe(port, nil))

}
```

以下示例是我们用于将我们的 cookie 值 POST 到`/middle`处理程序的 HTML：

```go
<html>
<head><title>Flash Message</title></head>
<body>
<form action="/middle" method="POST">
  <input type="text" name="message" />
  <input type="submit" value="Send Message" />
</form>
</body>
</html>
```

如果您按照页面的建议再次刷新，cookie 值将被删除，页面将不会呈现，就像您之前看到的那样。

要开始闪存消息，我们点击我们的`/start`端点，并输入一个预期的值，然后点击**发送消息**按钮：

![利用闪存消息](img/B04294_06_01.jpg)

在这一点上，我们将被发送到`/middle`端点，该端点将设置 cookie 值并将 HTTP 重定向到`/finish`：

![利用闪存消息](img/B04294_06_02.jpg)

现在我们可以看到我们的价值。由于`/finish`端点处理程序还取消了 cookie，我们将无法再次检索该值。如果我们在第一次出现时按照`/finish`的指示做什么，会发生什么：

![利用闪存消息](img/B04294_06_03.jpg)

就这些了。

# 总结

希望到目前为止，您已经掌握了如何在 Go 中利用基本的 cookie 和会话，无论是通过原生 Go 还是通过使用 Gorilla 等框架。我们已经尝试演示了后者的内部工作原理，以便您能够在不使用额外库混淆功能的情况下进行构建。

我们已经将会话实现到我们的应用程序中，以实现请求之间的持久状态。这是 Web 身份验证的基础。通过在数据库中启用`users`和`sessions`表，我们能够登录用户，注册会话，并在后续请求中将该会话与正确的用户关联起来。

通过利用闪存消息，我们利用了一个非常特定的功能，允许在两个端点之间传输信息，而不需要启用可能看起来像错误或生成错误输出的额外请求。我们的闪存消息只能使用一次，然后过期。

在第七章中，*微服务和通信*，我们将研究如何连接现有和新 API 之间的不同系统和应用程序，以允许基于事件的操作在这些系统之间协调。这将有助于连接到同一环境中的其他服务，以及应用程序之外的服务。
