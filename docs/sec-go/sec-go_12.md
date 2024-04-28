# 社会工程

社会工程是指攻击者操纵或欺骗受害者执行某项行动或提供私人信息。这通常是通过冒充信任的人、制造紧急感或制造虚假前提来推动受害者采取行动。行动可能只是泄露信息，也可能更复杂，比如下载和执行恶意软件。

本章涵盖了蜜罐，尽管有时它们旨在欺骗机器人而不是人类。目标是故意欺骗，这是社会工程的核心。我们提供了基本的蜜罐示例，包括 TCP 和 HTTP 蜜罐。

本书未涵盖许多其他类型的社会工程。这包括物理或面对面的情况，例如尾随和假装是维护工作人员，以及其他数字和远程方法，例如电话呼叫、短信和社交媒体消息。

社会工程在法律上可能是一个灰色地带。例如，即使公司允许您对其员工进行社会工程，也不代表您有权钓取员工的个人电子邮件凭据。要意识到法律和道德的边界。

在本章中，我们将具体涵盖以下主题：

+   使用 Reddit 的 JSON REST API 收集个人情报

+   使用 SMTP 发送钓鱼邮件

+   生成 QR 码和对图像进行 base64 编码

+   蜜罐

# 通过 JSON REST API 收集情报

REST 与 JSON 正成为 Web API 的事实标准接口。每个 API 都不同，因此此示例的主要目标是展示如何从 REST 端点处理 JSON 数据。

此示例将以 Reddit 用户名作为参数，并打印该用户的最新帖子和评论，以了解他们讨论的话题。选择 Reddit 作为示例的原因是因为对于某些端点不需要进行身份验证，这样可以方便进行测试。其他提供 REST API 的服务，例如 Twitter 和 LinkedIn，也可以用于情报收集。

请记住，此示例的重点是提供从 REST 端点解析 JSON 的示例。由于每个 API 都不同，此示例应该作为参考，以便在编写自己的程序与 JSON API 交互时使用。必须定义一个数据结构以匹配 JSON 端点的响应。在此示例中，创建的数据结构与 Reddit 的响应匹配。

在 Go 中使用 JSON 时，首先需要定义数据结构，然后使用`Marshal`和`Unmarshal`函数在原始字符串和结构化数据格式之间进行编码和解码。以下示例创建了一个与 Reddit 返回的 JSON 结构匹配的数据结构。然后使用`Unmarshal`函数将字符串转换为 Go 数据对象。您不必为 JSON 中的每个数据创建一个变量。您可以省略不需要的字段。

JSON 响应中的数据是嵌套的，因此我们将利用匿名结构。这样可以避免为每个嵌套级别创建单独的命名类型。此示例创建了一个命名结构，其中所有嵌套级别都存储为嵌入的匿名结构。

Go 数据结构中的变量名与 JSON 响应中提供的变量名不匹配，因此在定义结构时，JSON 变量名直接提供在数据类型之后。这样可以使变量正确地从 JSON 数据映射到 Go 结构。这通常是必要的，因为 Go 数据结构中的变量名是区分大小写的。

请注意，每个网络服务都有自己的服务条款，这可能会限制或限制您访问其网站的方式。一些网站有规定禁止抓取数据，其他网站有访问限制。虽然这可能不构成刑事犯罪，但服务可能会因违反服务条款而封锁您的账户或 IP 地址。请务必阅读您与之互动的每个网站或 API 的服务条款。

此示例的代码如下：

```go
package main

import (
   "encoding/json"
   "fmt"
   "io/ioutil"
   "log"
   "net/http"
   "os"
   "time"
)

// Define the structure of the JSON response
// The json variable names are specified on
// the right since they do not match the
// struct variable names exactly
type redditUserJsonResponse struct {
   Data struct {
      Posts []struct { // Posts & comments
         Data struct {
            Subreddit  string  `json:"subreddit"`
            Title      string  `json:"link_title"`
            PostedTime float32 `json:"created_utc"`
            Body       string  `json:"body"`
         } `json:"data"`
      } `json:"children"`
   } `json:"data"`
}

func printUsage() {
   fmt.Println(os.Args[0] + ` - Print recent Reddit posts by a user

Usage: ` + os.Args[0] + ` <username>
Example: ` + os.Args[0] + ` nanodano
`)
}

func main() {
   if len(os.Args) != 2 {
      printUsage()
      os.Exit(1)
   }
   url := "https://www.reddit.com/user/" + os.Args[1] + ".json"

   // Make HTTP request and read response
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error making HTTP request. ", err)
   }
   defer response.Body.Close()
   body, err := ioutil.ReadAll(response.Body)
   if err != nil {
      log.Fatal("Error reading HTTP response body. ", err)
   }

   // Decode response into data struct
   var redditUserInfo redditUserJsonResponse
   err = json.Unmarshal(body, &redditUserInfo)
   if err != nil {
      log.Fatal("Error parson JSON. ", err)
   }

   if len(redditUserInfo.Data.Posts) == 0 {
      fmt.Println("No posts found.")
      fmt.Printf("Response Body: %s\n", body)
   }

   // Iterate through all posts found
   for _, post := range redditUserInfo.Data.Posts {
      fmt.Println("Subreddit:", post.Data.Subreddit)
      fmt.Println("Title:", post.Data.Title)
      fmt.Println("Posted:", time.Unix(int64(post.Data.PostedTime), 
         0))
      fmt.Println("Body:", post.Data.Body)
      fmt.Println("========================================")
   }
} 
```

# 使用 SMTP 发送网络钓鱼电子邮件

网络钓鱼是攻击者试图通过伪装成可信任来源的合法电子邮件或其他形式的通信来获取敏感信息的过程。

网络钓鱼通常通过电子邮件进行，但也可以通过电话、社交媒体或短信进行。我们专注于电子邮件方法。网络钓鱼可以大规模进行，向大量收件人发送通用电子邮件，希望有人会上当。*尼日利亚王子*电子邮件诈骗是一种流行的网络钓鱼活动。其他提供激励的电子邮件也很受欢迎，并且相对有效，例如提供 iPhone 赠品或礼品卡，如果他们参与并按照您提供的链接登录其凭据。网络钓鱼电子邮件还经常模仿使用真实签名和公司标志的合法发件人。通常会制造紧急感，以说服受害者迅速采取行动，而不遵循标准程序。

您可以使用第十章中提取网页中的电子邮件的程序*网络抓取*来收集电子邮件。将电子邮件提取功能与提供的网络爬虫示例结合起来，您就可以强大地从域中抓取电子邮件。

**鱼叉式网络钓鱼**是针对少数目标的有针对性的网络钓鱼的术语，甚至可能只针对一个特定目标。鱼叉式网络钓鱼需要更多的研究和定位，定制特定于个人的电子邮件，创建一个可信的前提，也许是冒充他们认识的人。鱼叉式网络钓鱼需要更多的工作，但它增加了愚弄用户的可能性，并减少了被垃圾邮件过滤器抓住的机会。

在尝试鱼叉式网络钓鱼活动时，您应该在撰写电子邮件之前首先收集有关目标的尽可能多的信息。在本章的早些时候，我们谈到了使用 JSON REST API 来收集有关目标的数据。如果您的目标个人或组织有网站，您还可以使用第十章中的字数统计程序和标题抓取程序，*网络抓取*。收集网站的最常见单词和标题可能是快速了解目标所属行业或可能提供的产品和服务的方法。

Go 标准库附带了用于发送电子邮件的 SMTP 包。Go 还有一个`net/mail`包，用于解析电子邮件（[`golang.org/pkg/net/mail/`](https://golang.org/pkg/net/mail/)）。`mail`包相对较小，本书未涵盖，但它允许您将电子邮件的完整文本解析为消息类型，从而让您单独提取正文和标题。此示例侧重于如何使用 SMTP 包发送电子邮件。

配置变量都在源代码的顶部定义。请确保设置正确的 SMTP 主机、端口、发件人和密码。常见的 SMTP 端口是`25`用于未加密访问，端口`465`和`587`通常用于加密访问。所有设置都将取决于您的 SMTP 服务器的配置。如果没有首先设置正确的服务器和凭据，此示例将无法正确运行。如果您有 Gmail 帐户，您可以重用大部分预填充的值，只需替换发件人和密码。

如果您正在使用 Gmail 发送邮件并使用双因素身份验证，则需要在[`security.google.com/settings/security/apppasswords`](https://security.google.com/settings/security/apppasswords)上创建一个应用程序专用密码。如果您不使用双因素身份验证，则可以在[`myaccount.google.com/lesssecureapps`](https://myaccount.google.com/lesssecureapps)上启用不安全的应用程序。

该程序创建并发送了两封示例电子邮件，一封是文本，一封是 HTML。还可以发送组合的文本和 HTML 电子邮件，其中电子邮件客户端选择渲染哪个版本。这可以通过将`Content-Type`标头设置为`multipart/alternative`并设置一个边界来区分文本电子邮件的结束和 HTML 电子邮件的开始来实现。这里没有涵盖发送组合的文本和 HTML 电子邮件，但值得一提。您可以在[`www.w3.org/Protocols/rfc1341/7_2_Multipart.html`](https://www.w3.org/Protocols/rfc1341/7_2_Multipart.html)上了解有关`multipart`内容类型的更多信息，*RFC 1341*。

Go 还提供了一个`template`软件包，允许您创建一个带有变量占位符的模板文件，然后使用来自结构体的数据填充占位符。如果您想要将模板文件与源代码分开，以便在不重新编译应用程序的情况下修改模板，模板将非常有用。以下示例不使用模板，但您可以在[`golang.org/pkg/text/template/`](https://golang.org/pkg/text/template/)上阅读更多关于模板的信息：

```go
package main

import (
   "log"
   "net/smtp"
   "strings"
)

var (
   smtpHost   = "smtp.gmail.com"
   smtpPort   = "587"
   sender     = "sender@gmail.com"
   password   = "SecretPassword"
   recipients = []string{
      "recipient1@example.com",
      "recipient2@example.com",
   }
   subject = "Subject Line"
)

func main() {
   auth := smtp.PlainAuth("", sender, password, smtpHost)

   textEmail := []byte(
      `To: ` + strings.Join(recipients, ", ") + `
Mime-Version: 1.0
Content-Type: text/plain; charset="UTF-8";
Subject: ` + subject + `

Hello,

This is a plain text email.
`)

   htmlEmail := []byte(
      `To: ` + strings.Join(recipients, ", ") + `
Mime-Version: 1.0
Content-Type: text/html; charset="UTF-8";
Subject: ` + subject + `

<html>
<h1>Hello</h1>
<hr />
<p>This is an <strong>HTML</strong> email.</p>
</html>
`)

   // Send text version of email
   err := smtp.SendMail(
      smtpHost+":"+smtpPort,
      auth,
      sender,
      recipients,
      textEmail,
   )
   if err != nil {
      log.Fatal(err)
   }

   // Send HTML version
   err = smtp.SendMail(
      smtpHost+":"+smtpPort,
      auth,
      sender,
      recipients,
      htmlEmail,
   )
   if err != nil {
      log.Fatal(err)
   }
}
```

# 生成 QR 码

**快速响应**（**QR**）码是一种二维条形码。它存储的信息比传统的一维线条形码更多。它们最初是在日本汽车工业中开发的，但已被其他行业采用。QR 码于 2000 年被 ISO 批准为国际标准。最新规范可在[`www.iso.org/standard/62021.html`](https://www.iso.org/standard/62021.html)上找到。

QR 码可以在一些广告牌、海报、传单和其他广告材料上找到。QR 码也经常用于交易中。您可能会在火车票上看到 QR 码，或者在发送和接收比特币等加密货币时看到 QR 码。一些身份验证服务，如双因素身份验证，利用 QR 码的便利性。

QR 码对社会工程学很有用，因为人类无法仅通过查看 QR 码来判断它是否恶意。往往 QR 码包含一个立即加载的 URL，使用户面临风险。如果您创建一个可信的借口，您可能会说服用户相信 QR 码。

此示例中使用的软件包称为`go-qrcode`，可在[`github.com/skip2/go-qrcode`](https://github.com/skip2/go-qrcode)上找到。这是一个在 GitHub 上可用的第三方库，不受 Google 或 Go 团队支持。`go-qrcode`软件包利用了标准库图像软件包：`image`，`image/color`和`image/png`。

使用以下命令安装`go-qrcode`软件包：

```go
go get github.com/skip2/go-qrcode/...
```

`go get`中的省略号（`...`）是通配符。它还将安装所有子软件包。

根据软件包作者的说法，QR 码的最大容量取决于编码的内容和错误恢复级别。最大容量为 2953 字节，4296 个字母数字字符，7089 个数字，或者是它们的组合。

该程序演示了两个主要点。首先是如何生成原始 PNG 字节形式的 QR 码，然后将要嵌入 HTML 页面的数据进行 base64 编码。完整的 HTML`img`标签被生成，并作为标准输出输出，可以直接复制粘贴到 HTML 页面中。第二部分演示了如何简单地生成 QR 码并直接写入文件。

这个例子生成了一个 PNG 格式的二维码图片。让我们提供你想要编码的文本和输出文件名作为命令行参数，程序将输出将你的数据编码为 QR 图像的图片：

```go
package main 

import (
   "encoding/base64"
   "fmt"
   "github.com/skip2/go-qrcode"
   "log"
   "os"
)

var (
   pngData        []byte
   imageSize      = 256 // Length and width in pixels
   err            error
   outputFilename string
   dataToEncode   string
)

// Check command line arguments. Print usage
// if expected arguments are not present
func checkArgs() {
   if len(os.Args) != 3 {
      fmt.Println(os.Args[0] + `

Generate a QR code. Outputs a PNG file in <outputFilename>.
Also outputs an HTML img tag with the image base64 encoded to STDOUT.

 Usage: ` + os.Args[0] + ` <outputFilename> <data>
 Example: ` + os.Args[0] + ` qrcode.png https://www.devdungeon.com`)
      os.Exit(1)
   }
   // Because these variables were above, at the package level
   // we don't have to return them. The same variables are
   // already accessible in the main() function
   outputFilename = os.Args[1]
   dataToEncode = os.Args[2]
}

func main() {
   checkArgs()

   // Generate raw binary data for PNG
   pngData, err = qrcode.Encode(dataToEncode, qrcode.Medium, 
      imageSize)
   if err != nil {
      log.Fatal("Error generating QR code. ", err)
   }

   // Encode the PNG data with base64 encoding
   encodedPngData := base64.StdEncoding.EncodeToString(pngData)

   // Output base64 encoded image as HTML image tag to STDOUT
   // This img tag can be embedded in an HTML page
   imgTag := "<img src=\"data:image/png;base64," + 
      encodedPngData + "\"/>"
   fmt.Println(imgTag) // For use in HTML

   // Generate and write to file with one function
   // This is a standalone function. It can be used by itself
   // without any of the above code
   err = qrcode.WriteFile(
      dataToEncode,
      qrcode.Medium,
      imageSize,
      outputFilename,
   )
   if err != nil {
      log.Fatal("Error generating QR code to file. ", err)
   }
} 
```

# Base64 编码数据

在前面的例子中，QR 码是 base64 编码的。由于这是一个常见的任务，值得介绍如何进行编码和解码。任何时候需要将二进制数据存储或传输为字符串时，base64 编码都是有用的。

这个例子演示了编码和解码字节切片的一个非常简单的用例。进行 base64 编码和解码的两个重要函数是`EncodeToString()`和`DecodeString()`：

```go
package main

import (
   "encoding/base64"
   "fmt"
   "log"
)

func main() {
   data := []byte("Test data")

   // Encode bytes to base64 encoded string.
   encodedString := base64.StdEncoding.EncodeToString(data)
   fmt.Printf("%s\n", encodedString)

   // Decode base64 encoded string to bytes.
   decodedData, err := base64.StdEncoding.DecodeString(encodedString)
   if err != nil {
      log.Fatal("Error decoding data. ", err)
   }
   fmt.Printf("%s\n", decodedData)
} 
```

# 蜜罐

蜜罐是你设置的用来捕捉攻击者的假服务。你故意设置一个服务，目的是引诱攻击者，让他们误以为这个服务是真实的，并包含某种敏感信息。通常，蜜罐被伪装成一个旧的、过时的、容易受攻击的服务器。日志记录或警报可以附加到蜜罐上，以快速识别潜在的攻击者。在你的内部网络上设置一个蜜罐可能会在任何系统被入侵之前警告你有攻击者的存在。

当攻击者攻击一台机器时，他们经常使用被攻击的机器来继续枚举、攻击和转移。如果你网络上的一个蜜罐检测到来自你网络上另一台机器的奇怪行为，比如端口扫描或登录尝试，那么表现奇怪的机器可能已经被攻击。

蜜罐有许多不同种类。它可以是从简单的 TCP 监听器，记录任何连接，到一个带有登录表单字段的假 HTML 页面，或者看起来像一个真实员工门户的完整的网络应用程序。如果攻击者认为他们已经找到了一个关键的应用程序，他们更有可能花时间试图获取访问权限。如果你设置有吸引力的蜜罐，你可能会让攻击者花费大部分时间在一个无用的蜜罐上。如果保留了详细的日志记录，你可以了解攻击者正在使用什么方法，他们有什么工具，甚至可能是他们的位置。

还有一些其他类型的蜜罐值得一提，但在这本书中没有进行演示：

+   **SMTP 蜜罐**：这模拟了一个开放的电子邮件中继，垃圾邮件发送者滥用它来捕捉试图使用你的邮件发送程序的垃圾邮件发送者。

+   **网络爬虫蜜罐**：这些是不打算被人访问的隐藏网页，但是链接到它的链接被隐藏在你网站的公共位置，比如 HTML 注释中，用来捕捉蜘蛛、爬虫和网页抓取器。

+   **数据库蜜罐**：这是一个带有详细日志记录以检测攻击者的假或真实数据库，可能还包含假数据以查看攻击者感兴趣的信息。

+   **蜜罐网络**：这是一个充满蜜罐的整个网络，旨在看起来像一个真实的网络，甚至可以自动化或伪造客户端流量到蜜罐服务，以模拟真实用户。

攻击者可能能够发现明显的蜜罐服务并避开它们。我建议你选择两个极端之一：尽可能使蜜罐模仿一个真实的服务，或者使服务成为一个不向攻击者透露任何信息的完全黑匣子。

我们在这一部分涵盖了非常基本的例子，以帮助你理解蜜罐的概念，并为你提供一个创建自己更加定制化蜜罐的模板。首先，演示了一个基本的 TCP 套接字蜜罐。这将监听一个端口，并记录任何连接和接收到的数据。为了配合这个例子，提供了一个 TCP 测试工具。它的行为类似于 Netcat 的原始版本，允许你通过标准输入向服务器发送单个消息。这可以用来测试 TCP 蜜罐，或者扩展和用于其他应用程序。最后一个例子是一个 HTTP 蜜罐。它提供了一个登录表单，记录了尝试进行身份验证，但总是返回错误。

确保你了解在你的网络上使用蜜罐的风险。如果你让一个蜜罐持续运行而不保持底层操作系统的更新，你可能会给你的网络增加真正的风险。

# TCP 蜜罐

我们将从一个 TCP 蜜罐开始。它将记录任何接收到的 TCP 连接和来自客户端的任何数据。

它将以身份验证失败的消息进行响应。由于它记录了来自客户端的任何数据，它将记录他们尝试进行身份验证的任何用户名和密码。你可以通过检查他们尝试的身份验证方法来了解他们的攻击方法，因为它就像一个黑匣子，不会给出任何关于它可能使用的身份验证机制的线索。你可以使用日志来查看他们是否将其视为 SMTP 服务器，这可能表明他们是垃圾邮件发送者，或者他们可能正在尝试与数据库进行身份验证，表明他们正在寻找信息。研究攻击者的行为可能非常有见地，甚至可以揭示你之前不知道的漏洞。攻击者可能会在蜜罐上使用服务指纹识别工具，你可能能够识别他们攻击方法中的模式，并找到阻止他们的方法。如果攻击者尝试使用真实的用户凭据登录，那么该用户很可能已经受到了威胁。

这个示例将记录高级请求，比如 HTTP 请求，以及低级连接，比如 TCP 端口扫描。TCP 连接扫描将被记录，但 TCP `SYN`（隐形）扫描将不会被检测到：

```go
package main

import (
   "bytes"
   "log"
   "net"
)

func handleConnection(conn net.Conn) {
   log.Printf("Received connection from %s.\n", conn.RemoteAddr())
   buff := make([]byte, 1024)
   nbytes, err := conn.Read(buff)
   if err != nil {
      log.Println("Error reading from connection. ", err)
   }
   // Always reply with a fake auth failed message
   conn.Write([]byte("Authentication failed."))
   trimmedOutput := bytes.TrimRight(buff, "\x00")
   log.Printf("Read %d bytes from %s.\n%s\n",
      nbytes, conn.RemoteAddr(), trimmedOutput)
   conn.Close()
}

func main() {
   portNumber := "9001" // or os.Args[1]
   ln, err := net.Listen("tcp", "localhost:"+portNumber)
   if err != nil {
       log.Fatalf("Error listening on port %s.\n%s\n",
          portNumber, err.Error())
   }
   log.Printf("Listening on port %s.\n", portNumber)
   for {
      conn, err := ln.Accept()
      if err != nil {
         log.Println("Error accepting connection.", err)
      }
      go handleConnection(conn)
   }
}
```

# TCP 测试工具

为了测试我们的 TCP 蜜罐，我们需要向它发送一些 TCP 流量。我们可以使用任何现有的网络工具，包括 Web 浏览器或 FTP 客户端来攻击蜜罐。一个很好的工具也是 Netcat，TCP/IP 瑞士军刀。不过，我们可以创建自己的简单克隆。它将简单地通过 TCP 读取和写入数据。输入和输出将分别通过标准输入和标准输出进行，允许你使用键盘和终端，或者通过文件和其他应用程序进行数据的输入或输出。

这个工具可以作为一个通用的网络测试工具使用，如果你有任何入侵检测系统或其他监控需要测试，它可能会有用。这个程序将从标准输入中获取数据并通过 TCP 连接发送它，然后读取服务器发送回来的任何数据并将其打印到标准输出。在运行这个示例时，你必须将主机和端口作为一个带有冒号分隔符的字符串传递，就像这样：`localhost:9001`。这是一个简单的 TCP 测试工具的代码：

```go
package main

import (
   "bytes"
   "fmt"
   "log"
   "net"
   "os"
)

func checkArgs() string {
   if len(os.Args) != 2 {
      fmt.Println("Usage: " + os.Args[0] + " <targetAddress>")
      fmt.Println("Example: " + os.Args[0] + " localhost:9001")
      os.Exit(0)
   }
   return os.Args[1]
}

func main() {
   var err error
   targetAddress := checkArgs()
   conn, err := net.Dial("tcp", targetAddress)
   if err != nil {
      log.Fatal(err)
   }
   buf := make([]byte, 1024)

   _, err = os.Stdin.Read(buf)
   trimmedInput := bytes.TrimRight(buf, "\x00")
   log.Printf("%s\n", trimmedInput)

   _, writeErr := conn.Write(trimmedInput)
   if writeErr != nil {
      log.Fatal("Error sending data to remote host. ", writeErr)
   }

   _, readErr := conn.Read(buf)
   if readErr != nil {
      log.Fatal("Error when reading from remote host. ", readErr)
   }
   trimmedOutput := bytes.TrimRight(buf, "\x00")
   log.Printf("%s\n", trimmedOutput)
} 
```

# HTTP POST 表单登录蜜罐

当你在网络上部署这个工具时，除非你是在进行有意的测试，任何表单提交都是一个红旗。这意味着有人试图登录到你的假服务器。由于没有合法的目的，只有攻击者才会有任何理由试图获取访问权限。这里不会有真正的身份验证或授权，只是一个幌子，让攻击者认为他们正在尝试登录。Go HTTP 包在 Go 1.6+中默认支持 HTTP 2。在[`golang.org/pkg/net/http/`](https://golang.org/pkg/net/http/)上阅读更多关于`net/http`包的信息。

以下程序将充当一个带有登录页面的 Web 服务器，只是将表单提交记录到标准输出。你可以运行这个服务器，然后尝试通过浏览器登录，登录尝试将被打印到终端上：

```go
package main 

import (
   "fmt"
   "log"
   "net/http"
)

// Correctly formatted function declaration to satisfy the
// Go http.Handler interface. Any function that has the proper
// request/response parameters can be used to process an HTTP request.
// Inside the request struct we have access to the info about
// the HTTP request and the remote client.
func logRequest(response http.ResponseWriter, request *http.Request) {
   // Write output to file or just redirect output of this 
   // program to file
   log.Println(request.Method + " request from " +  
      request.RemoteAddr + ". " + request.RequestURI)
   // If POST not empty, log attempt.
   username := request.PostFormValue("username")
   password := request.PostFormValue("pass")
   if username != "" || password != "" {
      log.Println("Username: " + username)
      log.Println("Password: " + password)
   }

   fmt.Fprint(response, "<html><body>")
   fmt.Fprint(response, "<h1>Login</h1>")
   if request.Method == http.MethodPost {
      fmt.Fprint(response, "<p>Invalid credentials.</p>")
   }
   fmt.Fprint(response, "<form method=\"POST\">")
   fmt.Fprint(response, 
      "User:<input type=\"text\" name=\"username\"><br>")
   fmt.Fprint(response, 
      "Pass:<input type=\"password\" name=\"pass\"><br>")
   fmt.Fprint(response, "<input type=\"submit\"></form><br>")
   fmt.Fprint(response, "</body></html>")
}

func main() {
   // Tell the default server multiplexer to map the landing URL to
   // a function called logRequest
   http.HandleFunc("/", logRequest)

   // Kick off the listener using that will run forever
   err := http.ListenAndServe(":8080", nil)
   if err != nil {
      log.Fatal("Error starting listener. ", err)
   }
} 
```

# HTTP 表单字段蜜罐

在上一个例子中，我们谈到了创建一个虚假的登录表单来检测有人尝试登录。如果我们想要确定是否是机器人呢？检测机器人尝试登录的能力也可以在生产网站上阻止机器人时派上用场。识别自动化机器人的一种方法是使用蜜罐表单字段。蜜罐表单字段是 HTML 表单上的输入字段，对用户隐藏，并且在表单被人类提交时预期为空白。机器人仍然会在表单中找到蜜罐字段并尝试填写它们。

目标是欺骗机器人，让它们认为表单字段是真实的，同时对用户隐藏。一些机器人会使用正则表达式来寻找关键词，比如`user`或`email`，并只填写这些字段；因此蜜罐字段通常使用名称，比如`email_address`或`user_name`，看起来像一个正常的字段。如果服务器在这些字段接收到数据，它可以假设表单是由机器人提交的。

如果我们在上一个例子中的登录表单中添加一个名为`email`的隐藏表单字段，机器人可能会尝试填写它，而人类则看不到它。表单字段可以使用 CSS 或`input`元素上的`hidden`属性来隐藏。我建议您使用位于单独样式表中的 CSS 来隐藏蜜罐表单字段，因为机器人可以轻松确定表单字段是否具有`hidden`属性，但要更难检测到输入是否使用样式表隐藏。

# 沙盒

一个相关的技术，本章没有演示，但值得一提的是沙盒。沙盒的目的与蜜罐不同，但它们都努力创建一个看起来合法的环境，实际上是严格受控和监视的。沙盒的一个例子是创建一个没有网络连接的虚拟机，记录所有文件更改和尝试的网络连接，以查看是否发生了可疑事件。

有时，沙盒环境可以通过查看 CPU 数量和内存来检测。如果恶意应用程序检测到资源较少的系统，比如 1 个 CPU 和 1GB 内存，那么它可能不是现代桌面机器，可能是一个沙盒。恶意软件作者已经学会了对沙盒环境进行指纹识别，并编程应用程序，以绕过任何恶意操作，如果它怀疑自己在沙盒中运行。

# 总结

阅读完本章后，您现在应该了解社会工程的一般概念，并能够提供一些例子。您应该了解如何使用 JSON 与 REST API 交互，生成 QR 码和 base64 编码数据，以及使用 SMTP 发送电子邮件。您还应该能够解释蜜罐的概念，并了解如何实现自己的蜜罐或扩展这些例子以满足自己的需求。

你还能想到哪些其他类型的蜜罐？哪些常见服务经常受到暴力破解或攻击？你如何定制或扩展社会工程的例子？你能想到其他可以用于信息收集的服务吗？

在下一章中，我们将涵盖后期利用的主题，比如部署绑定 shell、反向绑定 shell 或 Web shell；交叉编译；查找可写文件；以及修改文件时间戳、权限和所有权。
