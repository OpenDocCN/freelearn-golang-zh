# 将 GoMail 连接到真实电子邮件服务器

在本书的许多章节中，我们探讨了构建基于 Go 的电子邮件应用**GoMail**的方法。所有这些示例都使用了虚拟电子邮件服务器——`client`包中的某些代码，这使得我们能够在不需要管理服务器通信的情况下构建邮件客户端的 GUI 部分。在本附录的最后部分，我们将逐步添加代码以连接到真实电子邮件服务器。

在第十二章的探索基础上，*并发、网络和云服务*（特别是认证—*OAuth 2.0*示例），我们将使用 Gmail 公共 API 和 Go 语言的内置功能来实现这一点。

# 下载 Gmail 凭证

在[第十二章](https://cdp.packtpub.com/hands_on_gui_application_development_in_go/wp-admin/post.php?post=709&action=edit#post_35)，*并发、网络和云服务*中，我们仅使用标准库编写了 OAuth2 处理程序和 Gmail 集成。对于这次最后的代码探索，我们将使用 Google 为与 Gmail 服务器交互而创建的有用库。要使用此库，我们需要以不同的格式（`credentials.json`）的客户端凭证。要访问此文件，请登录您的 Google 账户，并转到 Go 快速入门页面[`developers.google.com/gmail/api/quickstart/go`](https://developers.google.com/gmail/api/quickstart/go)。在此处，您需要点击“启用 Gmail API”，然后下载“客户端配置”。这将下载我们将在下一节*创建服务器提供者*中初始化库所需的`credentials.json`文件。

下载凭证文件后，您需要使用`go get -u google.golang.org/api/gmail/v1`和`go get -u golang.org/x/oauth2/google`安装两个必需的库。然后，您就可以添加代码来连接到 Gmail 并访问您的电子邮件了。

# 创建服务器提供者

以下代码概述了位于本书仓库`client`包中的`gmail.go`文件的包含内容。如果您想直接尝试此功能，只需将您的`credentials.json`文件复制到当前目录，然后跳到下一节*更新示例以使用 Gmail*。

我们首先通过复制来自 Google 的 Gmail 快速入门 Go 文件中的`getClient()`、`getTokenFromWeb()`、`tokenFromFile()`和`saveToken()`函数来添加必要的 OAuth2 设置和令牌存储。这些函数与之前创建的 OAuth2 代码非常相似，但与 Google 库配合得更好。

# 下载收件箱消息

接下来，我们需要从已保存的凭证文件（在当前目录中）设置客户端。我们添加了一个新函数来解析数据，设置身份验证，并使用以下代码配置 `*gmail.Service`：

```go
func setupService() *gmail.Service {
   b, err := ioutil.ReadFile("credentials.json")
   if err != nil {
      log.Fatalf("Unable to read client secret file: %v", err)
   }

   config, err := google.ConfigFromJSON(b, gmail.GmailReadonlyScope,
      gmail.GmailComposeScope)
   if err != nil {
      log.Fatalf("Unable to parse client secret file to config: %v", err)
   }
   client := getClient(config)

   srv, err := gmail.New(client)
   if err != nil {
      log.Fatalf("Unable to retrieve Gmail client: %v", err)
   }

   return srv
}
```

从此函数返回的服务将用于对 Gmail API 的每次后续调用，因为它包含身份验证配置和凭证。接下来，我们需要通过下载用户收件箱中的所有消息来准备电子邮件列表。使用 `INBOX` 标签 ID 来过滤未存档的消息。此函数请求消息列表并遍历元数据以启动每个消息的完整下载。对于完整实现，我们需要添加分页支持（响应包含 `nextPageToken`，它指示更多数据何时可用），但此示例将处理多达 100 条消息：

```go
func downloadMessages(srv *gmail.Service) {
   req := srv.Users.Messages.List(user)
   req.LabelIds("INBOX")
   resp, err := req.Do()
   if err != nil {
      log.Fatalf("Unable to retrieve Inbox items: %v", err)
   }

   var emails []*EmailMessage
   for _, message := range resp.Messages {
      email := downloadMessage(srv, message)
      emails = append(emails, email)
   }
}
```

要下载每条单独的消息，我们需要实现之前提到的 `downloadMessage()` 函数。对于指定的消息，我们使用 Gmail Go API 下载完整内容。从结果数据中，我们从消息头中提取所需信息。除了解析 `Date` 头之外，我们还需要解码消息正文，它是以序列化、Base64 编码的格式：

```go
func downloadMessage(srv *gmail.Service, message *gmail.Message) *EmailMessage {
   mail, err := srv.Users.Messages.Get(user, message.Id).Do()
   if err != nil {
      log.Fatalf("Unable to retrieve message payload: %v", err)
   }

   var subject string
   var to, from Email
   var date time.Time

   content := decodeBody(mail.Payload)
   for _, header := range mail.Payload.Headers {
      switch header.Name {
      case "Subject":
         subject = header.Value
      case "To":
         to = Email(header.Value)
      case "From":
         from = Email(header.Value)
      case "Date":
         value := strings.Replace(header.Value, "(UTC)", "", -1)
         date, err = time.Parse("Mon, _2 Jan 2006 15:04:05 -0700",
            strings.TrimSpace(value))
         if err != nil {
            log.Println("Error: Could not parse date", value)
            date = time.Now()
         } else {
            log.Println("date", header.Value)
         }
      }
   }

   return NewMessage(subject, content, to, from, date)
}
```

`decodeBody()` 函数如下所示。对于纯文本电子邮件，内容位于 `Body.Data` 字段。对于多部分消息（其中正文为空），我们访问多个部分中的第一个并对其进行解码。解码 Base64 内容由标准库解码器处理：

```go
func decodeBody(payload *gmail.MessagePart) string {
   data := payload.Body.Data
   if data == "" {
      data = payload.Parts[0].Body.Data
   }
   content, err := base64.StdEncoding.DecodeString(data)
   if err != nil {
      fmt.Println("Failed to decode body", err)
   }

   return string(content)
}
```

准备此代码的最终步骤是完成 `EmailServer` 接口方法。`ListMessages()` 函数将返回 `downloadMessages()` 的结果，我们可以设置 `CurrentMessage()` 以返回列表顶部的电子邮件。完整实现见本书的代码库。

# 发送消息

要发送消息，我们必须将数据打包成原始格式以通过 API 发送。我们将重新使用 第十二章 中的 `Post` 示例的 `ToGMailEncoding()` 函数，*并发、网络和云服务*。在编码电子邮件之前，我们设置适当的 "From" 电子邮件地址（务必使用您登录的电子邮件地址或已注册的别名）和发送时的当前日期。编码后，我们将数据设置为 `gmail.Message` 类型的 `Raw` 字段，并将其传递给 Gmail 的 `Send()` 函数：

```go
func (g *gMailServer) Send(email *EmailMessage) {
   email.From = "YOUR EMAIL ADDRESS"
   email.Date = time.Now()

   data := email.ToGMailEncoding()
   msg := &gmail.Message{Raw:data}

   srv.Users.Messages.Send(user, msg).Do()
}
```

这段最小代码足以实现发送消息。所有艰苦的工作都由之前的设置代码完成——它提供了 `srv` 对象。

# 监听新消息

尽管谷歌提供了使用推送消息的能力，但设置非常复杂——因此，我们将改为轮询新消息。每 10 秒，我们应该下载任何到达的新消息。为此，我们可以使用历史 API，该 API 返回自历史特定点（使用`StartHistoryId()`设置）之后出现的任何消息。`HistoryId`是一个按时间顺序标记消息到达顺序的数字。在我们可以使用历史 API 之前，我们需要一个有效的`HistoryId`——我们可以通过在`downloadMessage()`函数中添加以下行来实现：

```go
g.history = uint64(math.Max(float64(g.history), float64(mail.HistoryId)))
```

一旦我们有一个历史点来查询，我们需要一个新的函数，可以下载从这个时间点以来的任何消息。以下代码与前面代码中的`downloadMessages()`类似，但只会下载新消息：

```go
func (g *gMailServer) downloadNewMessages(srv *gmail.Service) []*EmailMessage{
   req := srv.Users.History.List(g.user)
   req.StartHistoryId(g.history)
   req.LabelId("INBOX")
   resp, err := req.Do()
   if err != nil {
      log.Fatalf("Unable to retrieve Inbox items: %v", err)
   }

   var emails []*EmailMessage
   for _, history := range resp.History {
      for _, message := range history.Messages {
         email := downloadMessage(srv, message)
         emails = append(emails, email)
      }
   }

   return emails
}
```

为了完成功能，我们更新我们的`Incoming()`方法，使其设置通道并启动一个线程来轮询新消息。每`10`秒，我们将下载任何出现的新消息并将每个消息传递到创建的`in`通道：

```go
func (g *gMailServer) Incoming() chan *EmailMessage {
   in := make(chan *EmailMessage)

   go func() {
      for {
         time.Sleep(10 * time.Second)

         for _, email := range downloadNewMessages(srv) {
            g.emails = append([]*EmailMessage{email}, g.emails...)
            in <- email
         }
      }
   }()

   return in
}
```

完整的代码可以在本书代码仓库的`client`包中找到。让我们看看如何在之前的示例中使用这个新的电子邮件服务器。

# 更新示例以使用 Gmail

在 GoMail 示例应用中的任何一个，你都需要编辑`main.go`中的主要服务器设置。将服务器初始化更改为将`client.NewTestServer()`更改为`client.NewGMailServer()`。在放置了`credentials.json`文件后，运行此新代码将获得连接到你的 Gmail 账户以读取和发送电子邮件。请注意，对于此示例，你需要从命令行运行并遵循 OAuth2 设置步骤。为了提供更好的用户体验，你可以提供一个更复杂的`getTokenFromWeb()`函数实现。
