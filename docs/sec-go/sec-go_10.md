# 网络爬取

从网络中收集信息在许多情况下都是有用的。网站可以提供丰富的信息。这些信息可以用于在进行社会工程攻击或钓鱼攻击时提供帮助。您可以找到潜在目标的姓名和电子邮件，或者收集关键词和标题，这些可以帮助快速了解网站的主题或业务。您还可以通过网络爬取技术潜在地了解企业的位置，找到图像和文档，并分析网站的其他方面。

了解目标可以让您创建一个可信的借口。借口是攻击者用来欺骗毫无戒心的受害者，使其遵从某种方式上损害用户、其账户或其设备的请求的常见技术。例如，有人调查一家公司，发现它是一家在特定城市拥有集中式 IT 支持部门的大公司。他们可以打电话或给公司的人发电子邮件，假装是支持技术人员，并要求他们执行操作或提供他们的密码。公司公共网站上的信息可能包含许多用于设置借口情况的细节。

Web 爬行是爬取的另一个方面，它涉及跟随超链接到其他页面。广度优先爬行是指尽可能找到尽可能多的不同网站，并跟随它们以找到更多的站点。深度优先爬行是指在转移到下一个站点之前，爬取单个站点以找到所有可能的页面。

在本章中，我们将涵盖网络爬取和网络爬行。我们将通过示例向您介绍一些基本任务，例如查找链接、文档和图像，寻找隐藏文件和信息，并使用一个名为`goquery`的强大的第三方包。我们还将讨论减轻对您自己网站的爬取的技术。

在本章中，我们将具体涵盖以下主题：

+   网络爬取基础知识

+   字符串匹配

+   正则表达式

+   从响应中提取 HTTP 头

+   使用 cookies

+   从页面中提取 HTML 注释

+   在 Web 服务器上搜索未列出的文件

+   修改您的用户代理

+   指纹识别 Web 应用程序和服务器

+   使用 goquery 包

+   列出页面中的所有链接

+   列出页面中的所有文档链接

+   列出页面的标题和标题

+   计算页面上使用最频繁的单词

+   列出页面中所有外部 JavaScript 源

+   深度优先爬行

+   广度优先爬行

+   防止网络爬取

# 网络爬取基础知识

Web 爬取，如本书中所使用的，是从 HTML 结构化页面中提取信息的过程，这些页面是为人类查看而不是以编程方式消费的。一些服务提供了高效的用于编程使用的 API，但有些网站只提供他们的信息在 HTML 页面中。这些网络爬取示例演示了从 HTML 中提取信息的各种方法。我们将看一下基本的字符串匹配，然后是正则表达式，然后是一个名为`goquery`的强大包，用于网络爬取。

# 使用 strings 包在 HTTP 响应中查找字符串

要开始，让我们看一下如何进行基本的 HTTP 请求并使用标准库搜索字符串。首先，我们将创建`http.Client`并设置任何自定义变量；例如，客户端是否应该遵循重定向，应该使用哪组 cookies，或者应该使用哪种传输。

`http.Transport`类型实现了执行 HTTP 请求和获取响应的网络请求操作。默认情况下，使用`http.RoundTripper`，这执行单个 HTTP 请求。对于大多数用例，默认传输就足够了。默认情况下，使用环境中的 HTTP 代理，但也可以在传输中指定代理。如果要使用多个代理，这可能很有用。此示例不使用自定义的`http.Transport`类型，但我想强调`http.Transport`是`http.Client`中的嵌入类型。

我们正在创建一个自定义的 `http.Client` 类型，但只是为了覆盖 `Timeout` 字段。默认情况下，没有超时，应用程序可能会永远挂起。

可以在 `http.Client` 中覆盖的另一种嵌入类型是 `http.CookieJar` 类型。`http.CookieJar` 接口需要的两个函数是：`SetCookies()` 和 `Cookies()`。标准库附带了 `net/http/cookiejar` 包，并且其中包含了 `CookieJar` 的默认实现。多个 cookie jar 的一个用例是登录并存储与网站的多个会话。您可以登录多个用户，并将每个会话存储在一个 cookie jar 中，并根据需要使用每个会话。此示例不使用自定义 cookie jar。

HTTP 响应包含作为读取器接口的主体。我们可以使用接受读取器接口的任何函数从读取器中提取数据。这包括函数，如 `io.Copy()`、`io.ReadAtLeast()`、`io.ReadlAll()` 和 `bufio` 缓冲读取器。在此示例中，`ioutil.ReadAll()` 用于快速将 HTTP 响应的全部内容存储到字节切片变量中。

以下是此示例的代码实现：

```go
// Perform an HTTP request to load a page and search for a string
package main

import (
   "fmt"
   "io/ioutil"
   "log"
   "net/http"
   "os"
   "strings"
   "time"
)

func main() {
   // Load command line arguments
   if len(os.Args) != 3 {
      fmt.Println("Search for a keyword in the contents of a URL")
      fmt.Println("Usage: " + os.Args[0] + " <url> <keyword>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com NanoDano")
      os.Exit(1)
   }
   url := os.Args[1]
   needle := os.Args[2] // Like searching for a needle in a haystack

   // Create a custom http client to override default settings. Optional
   // Use http.Get() instead of client.Get() to use default client.
   client := &http.Client{
      Timeout: 30 * time.Second, // Default is forever!
      // CheckRedirect - Policy for following HTTP redirects
      // Jar - Cookie jar holding cookies
      // Transport - Change default method for making request
   }

   response, err := client.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   // Read response body
   body, err := ioutil.ReadAll(response.Body)
   if err != nil {
      log.Fatal("Error reading HTTP body. ", err)
   }

   // Search for string
   if strings.Contains(string(body), needle) {
      fmt.Println("Match found for " + needle + " in URL " + url)
   } else {
      fmt.Println("No match found for " + needle + " in URL " + url)
   }
} 
```

# 使用正则表达式在页面中查找电子邮件地址

正则表达式，或者 regex，实际上是一种独立的语言形式。本质上，它是一个表达文本搜索模式的特殊字符串。在使用 shell 时，您可能熟悉星号（`*`）。诸如 `ls *.txt` 的命令使用简单的正则表达式。在这种情况下，星号代表*任何东西*；因此只要以 `.txt` 结尾，任何字符串都会匹配。正则表达式除了星号之外还有其他符号，比如句号（`.`），它匹配任何单个字符，而不是星号，星号将匹配任意长度的字符串。甚至可以使用少量可用的符号来构建更强大的表达式。

正则表达式以慢而著称。所使用的实现保证以线性时间运行，而不是基于输入长度的指数时间。这意味着它将比许多其他不提供该保证的正则表达式实现运行得更快，比如 Perl。Go 的作者之一 Russ Cox 在 2007 年发表了两种不同方法的深度比较，可在[`swtch.com/~rsc/regexp/regexp1.html`](https://swtch.com/~rsc/regexp/regexp1.html)上找到。这对于我们搜索 HTML 页面内容的用例非常重要。如果正则表达式基于输入长度运行时间呈指数增长，可能需要很长时间才能执行某些表达式的搜索。

从[`en.wikipedia.org/wiki/Regular_expression`](https://en.wikipedia.org/wiki/Regular_expression)和相关的 Go 文档[`golang.org/pkg/regexp/`](https://golang.org/pkg/regexp/)中了解更多关于正则表达式的一般知识。

此示例使用正则表达式搜索嵌入在 HTML 中的电子邮件地址链接。它将搜索任何 `mailto` 链接并提取电子邮件地址。我们将使用默认的 HTTP 客户端，并调用 `http.Get()`，而不是创建自定义客户端来修改超时。

典型的电子邮件链接看起来像这样：

```go
<a href="mailto:nanodano@devdungeon.com">
<a href="mailto:nanodano@devdungeon.com?subject=Hello">
```

此示例中使用的正则表达式是：

`"mailto:.*?["?]`

让我们分解并检查每个部分：

+   `"mailto:`：整个片段只是一个字符串文字。第一个字符是引号（`"`），在正则表达式中没有特殊含义。它被视为普通字符。这意味着正则表达式将首先搜索引号字符。引号后面是文本 `mailto` 和一个冒号（`:`）。冒号也没有特殊含义。

+   `.*?`：句点（`.`）表示匹配除换行符以外的任何字符。星号表示基于前一个符号（句点）继续匹配零个或多个字符。在星号之后，是一个问号（`?`）。这个问号告诉星号不要贪婪。它将匹配可能的最短字符串。没有它，星号将继续匹配尽可能长的字符串，同时仍满足完整的正则表达式。我们只想要电子邮件地址本身，而不是任何查询参数，比如`?subject`，所以我们告诉它进行非贪婪或短匹配。

+   `["?]`：正则表达式的最后一部分是`["?]`集合。括号告诉正则表达式匹配括号内封装的任何字符。我们只有两个字符：引号和问号。这里的问号没有特殊含义，被视为普通字符。括号内的两个字符是电子邮件地址结束的两个可能字符。默认情况下，正则表达式将选择最后一个字符并返回最长的字符串，因为前面的星号会变得贪婪。然而，因为我们在前一节直接在星号后面添加了另一个问号，它将执行非贪婪搜索，并在第一个匹配括号内的字符的地方停止。

使用这种技术意味着我们只会找到在 HTML 中使用`<a>`标签明确链接的电子邮件。它不会找到在页面中以纯文本形式编写的电子邮件。创建一个正则表达式来搜索基于模式的电子邮件字符串，比如`<word>@<word>.<word>`，可能看起来很简单，但不同正则表达式实现之间的细微差别以及电子邮件可能具有的复杂变化使得很难制定一个能捕捉到所有有效电子邮件组合的正则表达式。如果您快速在网上搜索一个示例，您会看到有多少变化以及它们变得多么复杂。

如果您正在创建某种网络服务，重要的是通过发送电子邮件并要求他们以某种方式回复或验证链接来验证一个人的电子邮件帐户。我不建议您仅仅依赖正则表达式来确定电子邮件是否有效，我还建议您在使用正则表达式执行客户端电子邮件验证时要非常小心。用户可能有一个在技术上有效的奇怪电子邮件地址，您可能会阻止他们注册到您的服务。

以下是根据 1982 年*RFC 822*实际有效的电子邮件地址的一些示例：

+   `*.*@example.com`

+   `$what^the.#!$%@example.com`

+   `!#$%^&*=()@example.com`

+   `"!@#$%{}^&~*()|/="@example.com`

+   `"hello@example.com"@example.com`

2001 年，*RFC 2822*取代了*RFC 822*。在所有先前的示例中，只有最后两个包含 at（`@`）符号的示例被新的*RFC 2822*认为是无效的。所有其他示例仍然有效。在[`www.ietf.org/rfc/rfc822.txt`](https://www.ietf.org/rfc/rfc822.txt)和[`www.ietf.org/rfc/rfc2822.txt`](https://www.ietf.org/rfc/rfc2822.txt)上阅读原始 RFC。

这是该示例的代码实现：

```go
// Search through a URL and find mailto links with email addresses
package main

import (
   "fmt"
   "io/ioutil"
   "log"
   "net/http"
   "os"
   "regexp"
)

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("Search for emails in a URL")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Fetch the URL
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   // Read the response
   body, err := ioutil.ReadAll(response.Body)
   if err != nil {
      log.Fatal("Error reading HTTP body. ", err)
   }

   // Look for mailto: links using a regular expression
   re := regexp.MustCompile("\"mailto:.*?[?\"]")
   matches := re.FindAllString(string(body), -1)
   if matches == nil {
      // Clean exit if no matches found
      fmt.Println("No emails found.")
      os.Exit(0)
   }

   // Print all emails found
   for _, match := range matches {
      // Remove "mailto prefix and the trailing quote or question mark
      // by performing a slice operation to extract the substring
      cleanedMatch := match[8 : len(match)-1]
      fmt.Println(cleanedMatch)
   }
} 
```

# 从 HTTP 响应中提取 HTTP 标头

HTTP 标头包含有关请求和响应的元数据和描述信息。通过检查服务器提供的 HTTP 标头，您可以潜在地了解有关服务器的很多信息。您可以了解服务器的以下信息：

+   缓存系统

+   身份验证

+   操作系统

+   Web 服务器

+   响应类型

+   框架或内容管理系统

+   编程语言

+   口头语言

+   安全标头

+   Cookies

并非每个网络服务器都会返回所有这些标头，但从标头中尽可能多地学习是有帮助的。流行的框架，如 WordPress 和 Drupal，将返回一个`X-Powered-By`标头，告诉您它是 WordPress 还是 Drupal 以及版本。

会话 cookie 也可以透露很多信息。名为`PHPSESSID`的 cookie 告诉您它很可能是一个 PHP 应用程序。Django 的默认会话 cookie 的名称是`sessionid`，Java 的是`JSESSIONID`，Ruby on Rail 的会话 cookie 遵循`_APPNAME_session`的模式。您可以使用这些线索来识别 Web 服务器。如果您只想要头部而不需要页面的整个主体，您可以始终使用 HTTP `HEAD`方法而不是 HTTP `GET`。`HEAD`方法将只返回头部。

这个例子对 URL 进行了一个`HEAD`请求，并打印出了它的所有头部。`http.Response`类型包含一个名为`Header`的字符串到字符串的映射，其中包含每个 HTTP 头的键值对：

```go
// Perform an HTTP HEAD request on a URL and print out headers
package main

import (
   "fmt"
   "log"
   "net/http"
   "os"
)

func main() {
   // Load URL from command line arguments
   if len(os.Args) != 2 {
      fmt.Println(os.Args[0] + " - Perform an HTTP HEAD request to a URL")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Perform HTTP HEAD
   response, err := http.Head(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   // Print out each header key and value pair
   for key, value := range response.Header {
      fmt.Printf("%s: %s\n", key, value[0])
   }
} 
```

# 使用 HTTP 客户端设置 cookie

Cookie 是现代 Web 应用程序的一个重要组成部分。Cookie 作为 HTTP 头在客户端和服务器之间来回发送。Cookie 只是由浏览器客户端存储的文本键值对。它们用于在客户端上存储持久数据。它们可以用于存储任何文本值，但通常用于存储首选项、令牌和会话信息。

会话 cookie 通常存储与服务器相匹配的令牌。当用户登录时，服务器会创建一个带有与该用户相关联的标识令牌的会话。然后，服务器以 cookie 的形式将令牌发送回给用户。当客户端以 cookie 的形式发送会话令牌时，服务器会查找并在会话存储中找到匹配的令牌，这可能是数据库、文件或内存中。会话令牌需要足够的熵来确保它是唯一的，攻击者无法猜测。

如果用户在公共 Wi-Fi 网络上，并访问一个不使用 SSL 的网站，附近的任何人都可以看到明文的 HTTP 请求。攻击者可以窃取会话 cookie 并在自己的请求中使用它。当以这种方式 sidejacked cookie 时，攻击者可以冒充受害者。服务器将把他们视为已登录的用户。攻击者可能永远不会知道密码，也不需要知道。

因此，定期注销网站并销毁任何活动会话可能是有用的。一些网站允许您手动销毁所有活动会话。如果您运行一个 Web 服务，我建议您为会话设置合理的过期时间。银行网站通常做得很好，通常强制执行短暂的 10-15 分钟过期时间。

服务器在创建新 cookie 时向客户端发送一个`Set-Cookie`头。然后客户端使用`Cookie`头将 cookie 发送回服务器。

这是服务器发送的 cookie 头的一个简单示例：

```go
Set-Cookie: preferred_background=blue
Set-Cookie: session_id=PZRNVYAMDFECHBGDSSRLH
```

以下是来自客户端的一个示例头部：

```go
Cookie: preferred_background=blue; session_id=PZRNVYAMDFECHBGDSSRLH
```

Cookie 还可以包含其他属性，例如在第九章中讨论的`Secure`和`HttpOnly`标志，*Web 应用程序*。其他属性包括到期日期、域和路径。这个例子只是展示了最简单的应用程序。

在这个例子中，使用自定义会话 cookie 进行了一个简单的请求。会话 cookie 是在向网站发出请求时允许您*登录*的东西。这个例子应该作为如何使用 cookie 发出请求的参考，而不是一个独立的工具。首先，在`main`函数之前定义 URL。然后，首先创建 HTTP 请求，指定 HTTP `GET`方法。由于`GET`请求通常不需要主体，因此提供了一个空主体。然后，使用一个新的头部，cookie，更新新的请求。在这个例子中，`session_id`是会话 cookie 的名称，但这将取决于正在交互的 Web 应用程序。

一旦请求准备好，就会创建一个 HTTP 客户端来实际发出请求并处理响应。请注意，HTTP 请求和 HTTP 客户端是独立的实体。例如，您可以多次重用一个请求，使用不同的客户端使用一个请求，并使用单个客户端进行多个请求。这允许您创建多个具有不同会话 cookie 的请求对象，如果需要管理多个客户端会话。

以下是此示例的代码实现：

```go
package main

import (
   "fmt"
   "io/ioutil"
   "log"
   "net/http"
)

var url = "https://www.example.com"

func main() {
   // Create the HTTP request
   request, err := http.NewRequest("GET", url, nil)
   if err != nil {
      log.Fatal("Error creating HTTP request. ", err)
   }

   // Set cookie
   request.Header.Set("Cookie", "session_id=<SESSION_TOKEN>")

   // Create the HTTP client, make request and print response
   httpClient := &http.Client{}
   response, err := httpClient.Do(request)
   data, err := ioutil.ReadAll(response.Body)
   fmt.Printf("%s\n", data)
} 
```

# 在网页中查找 HTML 注释

HTML 注释有时可能包含惊人的信息。我个人见过在 HTML 注释中包含管理员用户名和密码的网站。我还见过整个菜单被注释掉，但链接仍然有效，可以直接访问。您永远不知道一个粗心的开发人员可能留下什么样的信息。

如果您要在代码中留下评论，最好将它们留在服务器端代码中，而不是在面向客户端的 HTML 和 JavaScript 中。在 PHP、Ruby、Python 或其他后端代码中进行注释。您永远不希望在代码中向客户端提供比他们需要的更多信息。

此程序中使用的正则表达式由几个特殊序列组成。以下是完整的正则表达式。它基本上是说，“匹配`<!--`和`-->`之间的任何内容。”让我们逐个检查它：

+   `<!--(.|\n)*?-->`：开头和结尾分别是`<!--`和`-->`，这是 HTML 注释的开始和结束标记。这些是普通字符，而不是正则表达式的特殊字符。

+   `(.|\n)*?`：这可以分解为两部分：

+   +   `(.|\n)`：第一部分有一些特殊字符。括号`()`括起一组选项。管道`|`分隔选项。选项本身是点`.`和换行字符`\n`。点表示匹配任何字符，除了换行符。因为 HTML 注释可以跨多行，我们希望匹配任何字符，包括换行符。整个部分`(.|\n)`表示匹配点或换行符。

+   +   `*?`：星号表示继续匹配前一个字符或表达式零次或多次。紧接在星号之前的是括号集，因此它将继续尝试匹配`(.|\n)`。问号告诉星号是非贪婪的，或者返回可能的最小匹配。没有问号，以指定它为非贪婪；它将匹配可能的最大内容，这意味着它将从页面中第一个注释的开头开始，并在页面中最后一个注释的结尾结束，包括中间的所有内容。

尝试运行此程序针对一些网站，并查看您能找到什么样的 HTML 注释。您可能会对您能发现的信息感到惊讶。例如，MailChimp 注册表单附带了一个 HTML 注释，实际上为您提供了绕过机器人注册预防的提示。MailChimp 注册表单使用了一个蜜罐字段，不应该填写，否则它会假定该表单是由机器人提交的。看看您能找到什么。

此示例首先获取提供的 URL，然后使用我们之前讨论过的正则表达式搜索 HTML 注释。然后将找到的每个匹配打印到标准输出：

```go
// Search through a URL and find HTML comments
package main

import (
   "fmt"
   "io/ioutil"
   "log"
   "net/http"
   "os"
   "regexp"
)

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("Search for HTML comments in a URL")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Fetch the URL and get response
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }
   body, err := ioutil.ReadAll(response.Body)
   if err != nil {
      log.Fatal("Error reading HTTP body. ", err)
   }

   // Look for HTML comments using a regular expression
   re := regexp.MustCompile("<!--(.|\n)*?-->")
   matches := re.FindAllString(string(body), -1)
   if matches == nil {
      // Clean exit if no matches found
      fmt.Println("No HTML comments found.")
      os.Exit(0)
   }

   // Print all HTML comments found
   for _, match := range matches {
      fmt.Println(match)
   }
} 
```

# 在网络服务器上查找未列出的文件

有一个名为 DirBuster 的流行程序，渗透测试人员用于查找未列出的文件。DirBuster 是一个 OWASP 项目，预装在流行的渗透测试 Linux 发行版 Kali 上。只需使用标准库，我们就可以创建一个快速、并发和简单的 DirBuster 克隆，只需几行代码。有关 DirBuster 的更多信息，请访问[`www.owasp.org/index.php/Category:OWASP_DirBuster_Project`](https://www.owasp.org/index.php/Category:OWASP_DirBuster_Project)。

这个程序是 DirBuster 的一个简单克隆，它基于一个单词列表搜索未列出的文件。你将不得不创建自己的单词列表。这里提供了一小部分示例文件名，以便给你一些想法，并用作起始列表。根据你自己的经验和源代码构建你的文件列表。一些 Web 应用程序有特定名称的文件，这将允许你指纹识别使用的框架。还要寻找备份文件、配置文件、版本控制文件、更改日志文件、私钥、应用程序日志以及任何不打算公开的东西。你也可以在互联网上找到预先构建的单词列表，包括 DirBuster 的列表。

以下是一个你可以搜索的文件的示例列表：

+   `.gitignore`

+   `.git/HEAD`

+   `id_rsa`

+   `debug.log`

+   `database.sql`

+   `index-old.html`

+   `backup.zip`

+   `config.ini`

+   `settings.ini`

+   `settings.php.bak`

+   `CHANGELOG.txt`

这个程序将使用提供的单词列表搜索一个域，并报告任何没有返回 404 NOT FOUND 响应的文件。单词列表应该用换行符分隔文件名，并且每行一个文件名。在提供域名作为参数时，尾随斜杠是可选的，程序将在有或没有域名尾随斜杠的情况下正常运行。但是协议必须被指定，这样请求才知道是使用 HTTP 还是 HTTPS。

`url.Parse()`函数用于创建一个正确的 URL 对象。使用 URL 类型，你可以独立修改`Path`而不修改`Host`或`Scheme`。这提供了一种简单的方法来更新 URL，而不必求助于手动字符串操作。

为了逐行读取文件，使用了一个 scanner。默认情况下，scanner 按照换行符分割，但可以通过调用`scanner.Split()`并提供自定义分割函数来覆盖。我们使用默认行为，因为单词应该是在单独的行上提供的。

```go
// Look for unlisted files on a domain
package main

import (
   "bufio"
   "fmt"
   "log"
   "net/http"
   "net/url"
   "os"
   "strconv"
)

// Given a base URL (protocol+hostname) and a filepath (relative URL)
// perform an HTTP HEAD and see if the path exists.
// If the path returns a 200 OK print out the path
func checkIfUrlExists(baseUrl, filePath string, doneChannel chan bool) {
   // Create URL object from raw string
   targetUrl, err := url.Parse(baseUrl)
   if err != nil {
      log.Println("Error parsing base URL. ", err)
   }
   // Set the part of the URL after the host name
   targetUrl.Path = filePath

   // Perform a HEAD only, checking status without
   // downloading the entire file
   response, err := http.Head(targetUrl.String())
   if err != nil {
      log.Println("Error fetching ", targetUrl.String())
   }

   // If server returns 200 OK file can be downloaded
   if response.StatusCode == 200 {
      log.Println(targetUrl.String())
   }

   // Signal completion so next thread can start
   doneChannel <- true
}

func main() {
   // Load command line arguments
   if len(os.Args) != 4 {
      fmt.Println(os.Args[0] + " - Perform an HTTP HEAD request to a URL")
      fmt.Println("Usage: " + os.Args[0] + 
         " <wordlist_file> <url> <maxThreads>")
      fmt.Println("Example: " + os.Args[0] + 
         " wordlist.txt https://www.devdungeon.com 10")
      os.Exit(1)
   }
   wordlistFilename := os.Args[1]
   baseUrl := os.Args[2]
   maxThreads, err := strconv.Atoi(os.Args[3])
   if err != nil {
      log.Fatal("Error converting maxThread value to integer. ", err)
   }

   // Track how many threads are active to avoid
   // flooding a web server
   activeThreads := 0
   doneChannel := make(chan bool)

   // Open word list file for reading
   wordlistFile, err := os.Open(wordlistFilename)
   if err != nil {
      log.Fatal("Error opening wordlist file. ", err)
   }

   // Read each line and do an HTTP HEAD
   scanner := bufio.NewScanner(wordlistFile)
   for scanner.Scan() {
      go checkIfUrlExists(baseUrl, scanner.Text(), doneChannel)
      activeThreads++

      // Wait until a done signal before next if max threads reached
      if activeThreads >= maxThreads {
         <-doneChannel
         activeThreads -= 1
      }
   }

   // Wait for all threads before repeating and fetching a new batch
   for activeThreads > 0 {
      <-doneChannel
      activeThreads -= 1
   }

   // Scanner errors must be checked manually
   if err := scanner.Err(); err != nil {
      log.Fatal("Error reading wordlist file. ", err)
   }
} 
```

# 更改请求的用户代理

一个常见的阻止爬虫和网络爬虫的技术是阻止特定的用户代理。一些服务会把包含关键词如`curl`和`python`的特定用户代理列入黑名单。你可以通过简单地将你的用户代理更改为`firefox`来绕过大部分这些限制。

要设置用户代理，你必须首先创建 HTTP 请求对象。在实际请求之前必须设置头部。这意味着你不能使用`http.Get()`等快捷便利函数。我们必须创建客户端，然后创建一个请求，然后使用客户端来`client.Do()`请求。

这个例子使用`http.NewRequest()`创建了一个 HTTP 请求，然后修改请求头来覆盖`User-Agent`头部。你可以用这个来隐藏、伪装或者诚实。为了成为一个良好的网络公民，我建议你为你的爬虫创建一个独特的用户代理，这样网站管理员可以限制或者阻止你的机器人。我还建议你在用户代理中包含一个网站或者电子邮件地址，这样网站管理员可以请求跳过你的爬虫。

以下是这个例子的代码实现：

```go
// Change HTTP user agent
package main

import (
   "log"
   "net/http"
)

func main() {
   // Create the request for use later
   client := &http.Client{}
   request, err := http.NewRequest("GET", 
      "https://www.devdungeon.com", nil)
   if err != nil {
      log.Fatal("Error creating request. ", err)
   }

   // Override the user agent
   request.Header.Set("User-Agent", "_Custom User Agent_")

   // Perform the request, ignore response.
   _, err = client.Do(request)
   if err != nil {
      log.Fatal("Error making request. ", err)
   }
} 
```

# 指纹识别 Web 应用程序技术栈

指纹识别 Web 应用程序是指尝试识别用于提供 Web 应用程序的技术。指纹识别可以在几个级别进行。在较低级别，HTTP 头可以提供关于正在运行的操作系统（如 Windows 或 Linux）和 Web 服务器（如 Apache 或 nginx）的线索。头部还可以提供有关应用程序级别使用的编程语言或框架的信息。在较高级别，Web 应用程序可以被指纹识别以确定正在使用哪些 JavaScript 库，是否包括任何分析平台，是否显示任何广告网络，正在使用的缓存层等信息。我们将首先查看 HTTP 头部，然后涵盖更复杂的指纹识别方法。

指纹识别是攻击或渗透测试中的关键步骤，因为它有助于缩小选项并确定要采取的路径。识别正在使用的技术还让您可以搜索已知的漏洞。如果一个 Web 应用程序没有及时更新，简单的指纹识别和漏洞搜索可能就足以找到并利用已知的漏洞。如果没有其他办法，它也可以帮助您了解目标。

# 基于 HTTP 响应头的指纹识别

我建议您首先检查 HTTP 头，因为它们是简单的键值对，通常每个请求只返回几个。手动浏览头部不会花费太长时间，所以您可以在继续应用程序之前首先检查它们。应用程序级别的指纹识别更加复杂，我们稍后会谈论这个。在本章的前面，有一个关于提取 HTTP 头并打印它们以供检查的部分（*从 HTTP 响应中提取 HTTP 头*部分）。您可以使用该程序来转储不同网页的头部并查看您能找到什么。

基本思想很简单。寻找关键字。特别是一些头部包含最明显的线索，例如`X-Powered-By`、`Server`和`X-Generator`头部。`X-Powered-By`头部可以包含正在使用的框架或**内容管理系统**（**CMS**）的名称，例如 WordPress 或 Drupal。

检查头部有两个基本步骤。首先，您需要获取头部。使用本章前面提供的示例来提取 HTTP 头。第二步是进行字符串搜索以查找关键字。您可以使用`strings.ToUpper()`和`strings.Contains()`直接搜索关键字，或者使用正则表达式。请参考本章前面的示例，了解如何使用正则表达式。一旦您能够搜索头部，您只需要能够生成要搜索的关键字列表。

有许多关键字可以搜索。您搜索的内容将取决于您要寻找的内容。我将尝试涵盖几个广泛的类别，以便给您一些寻找内容的想法。您可以尝试识别的第一件事是主机正在运行的操作系统。以下是一个示例关键字列表，您可以在 HTTP 头部中找到，以指示操作系统：

+   `Linux`

+   `Debian`

+   `Fedora`

+   `Red Hat`

+   `CentOS`

+   `Ubuntu`

+   `FreeBSD`

+   `Win32`

+   `Win64`

+   `Darwin`

以下是一些关键字，可以帮助您确定正在使用哪种 Web 服务器。这绝不是一个详尽的列表，但涵盖了几个关键字，如果您在互联网上搜索，将会产生结果：

+   `Apache`

+   `Nginx`

+   `Microsoft-IIS`

+   `Tomcat`

+   `WEBrick`

+   `Lighttpd`

+   `IBM HTTP Server`

确定正在使用的编程语言可以在攻击选择上产生很大的影响。像 PHP 这样的脚本语言对不同的东西都是脆弱的，与 Java 服务器或 ASP.NET 应用程序不同。以下是一些示例关键字，您可以使用它们在 HTTP 头中搜索，以确定哪种语言支持应用程序：

+   `Python`

+   `Ruby`

+   `Perl`

+   `PHP`

+   `ASP.NET`

会话 cookie 也是确定使用的框架或语言的重要线索。例如，`PHPSESSID`表示 PHP，`JSESSIONID`表示 Java。以下是一些会话 cookie，您可以搜索：

+   `PHPSESSID`

+   `JSESSIONID`

+   `session`

+   `sessionid`

+   `CFID/CFTOKEN`

+   `ASP.NET_SessionId`

# 指纹识别 Web 应用程序

一般来说，指纹识别 Web 应用程序涵盖的范围要比仅查看 HTTP 头部要广泛得多。您可以在 HTTP 头部中进行基本的关键字搜索，就像刚才讨论的那样，并且可以学到很多，但是在 HTML 源代码和服务器上的其他文件的内容或简单存在中也有大量信息。

在 HTML 源代码中，你可以寻找一些线索，比如页面本身的结构以及 HTML 元素的类和 ID 的名称。AngularJS 应用程序具有独特的 HTML 属性，比如`ng-app`，可以用作指纹识别的关键词。Angular 通常也包含在`script`标签中，就像其他框架如 jQuery 一样。`script`标签也可以被检查以寻找其他线索。寻找诸如 Google Analytics、AdSense、Yahoo 广告、Facebook、Disqus、Twitter 和其他第三方嵌入的 JavaScript 等内容。

仅仅通过 URL 中的文件扩展名就可以告诉你正在使用的是什么语言。例如，`.php`、`.jsp`和`.asp`分别表示正在使用 PHP、Java 和 ASP。

我们还研究了一个在网页中查找 HTML 注释的程序。一些框架和 CMS 会留下可识别的页脚或隐藏的 HTML 注释。有时标记以小图片的形式出现。

目录结构也可以是另一个线索。首先需要熟悉不同的框架。例如，Drupal 将站点信息存储在一个名为`/sites/default`的目录中。如果你尝试访问该 URL，并且收到 403 FORBIDDEN 的响应而不是 404 NOT FOUND 错误，那么你很可能找到了一个基于 Drupal 的网站。

寻找诸如`wp-cron.php`之类的文件。在*在 Web 服务器上查找未列出的文件*部分，我们研究了使用 DirBuster 克隆来查找未列出的文件。找到一组可以用来指纹识别 Web 应用程序的唯一文件，并将它们添加到你的单词列表中。你可以通过检查不同 Web 框架的代码库来确定要查找哪些文件。例如，WordPress 和 Drupal 的源代码是公开可用的。使用本章早些时候讨论的用于查找未列出文件的程序来搜索文件。你还可以搜索与文档相关的其他未列出的文件，比如`CHANGELOG.txt`、`readme.txt`、`readme.md`、`readme.html`、`LICENSE.txt`、`install.txt`或`install.php`。

通过指纹识别正在运行的应用程序的版本，可以更详细地了解 Web 应用程序。如果你可以访问源代码，这将更容易。我将以 WordPress 为例，因为它是如此普遍，并且源代码可以在 GitHub 上找到[`github.com/WordPress/WordPress`](https://github.com/WordPress/WordPress)。

目标是找出版本之间的差异。WordPress 是一个很好的例子，因为它们都带有包含所有管理界面的`/wp-admin/`目录。在`/wp-admin/`目录中，有`css`和`js`文件夹，分别包含样式表和脚本。当网站托管在服务器上时，这些文件是公开可访问的。对这些文件夹使用`diff`命令，以确定哪些版本引入了新文件，哪些版本删除了文件，哪些版本修改了现有文件。将所有这些信息结合起来，通常可以将应用程序缩小到特定版本，或者至少缩小到一小范围的版本。

举个假设的例子，假设版本 1.0 只包含一个文件：`main.js`。版本 1.1 引入了第二个文件：`utility.js`。版本 1.3 删除了这两个文件，并用一个文件`master.js`替换了它们。你可以向 Web 服务器发出 HTTP 请求获取这三个文件：`main.js`、`utility.js`和`master.js`。根据哪些文件返回 200 OK 错误，哪些文件返回 404 NOT FOUND 错误，你可以确定正在运行的是哪个版本。

如果相同的文件存在于多个版本中，你可以深入检查文件的内容。可以进行逐字节比较或对文件进行哈希处理并比较校验和。哈希处理和哈希处理的示例在第六章 *密码学*中有介绍。

有时，识别版本可能比刚才描述的整个过程简单得多。有时会有一个`CHANGELOG.txt`或`readme.html`文件，它会告诉您确切地运行的是哪个版本，而无需做任何工作。

# 如何防止您的应用程序被指纹识别

正如前面所示，有多种方法可以在技术堆栈的许多不同级别上对应用程序进行指纹识别。您真正应该问自己的第一个问题是，“我需要防止指纹识别吗？”一般来说，试图防止指纹识别是一种混淆形式。混淆有点具有争议性，但我认为每个人都同意混淆不是加密的安全性。它可能会暂时减慢、限制信息或使攻击者困惑，但它并不能真正防止利用任何漏洞。现在，我并不是说混淆根本没有好处，但它永远不能单独依赖。混淆只是一层薄薄的掩饰。

显然，您不希望透露太多关于您的应用程序的信息，比如调试输出或配置设置，但无论如何，当服务在网络上可用时，一些信息都将可用。您将不得不在隐藏信息方面做出选择，需要投入多少时间和精力。

有些人甚至会输出错误信息来误导攻击者。就我个人而言，在加固服务器时，输出虚假标头并不在我的清单上。我建议您做的一件事是在部署之前删除任何额外的文件，就像之前提到的那样。在部署之前，应删除诸如更改日志文件、默认设置文件、安装文件和文档文件等文件。不要公开提供不需要应用程序工作的文件。

混淆是一个值得单独章节甚至单独一本书的话题。有一些专门颁发最有创意和奇异的混淆形式的混淆竞赛。有一些工具可以帮助您混淆 JavaScript 代码，但另一方面也有反混淆工具。

# 使用 goquery 包进行网络抓取

`goquery`包不是标准库的一部分，但可以在 GitHub 上找到。它旨在与 jQuery 类似——这是一个用于与 HTML DOM 交互的流行 JavaScript 框架。正如前面所示，尝试使用字符串匹配和正则表达式进行搜索既繁琐又复杂。`goquery`包使得处理 HTML 内容和搜索特定元素变得更加容易。我建议这个包的原因是它是基于非常流行的 jQuery 框架建模的，许多人已经熟悉它。

您可以使用`go get`命令获取`goquery`包：

```go
go get https://github.com/PuerkitoBio/goquery  
```

文档可在[`godoc.org/github.com/PuerkitoBio/goquery`](https://godoc.org/github.com/PuerkitoBio/goquery)找到。

# 列出页面中的所有超链接

在介绍`goquery`包时，我们将看一个常见且简单的任务。我们将找到页面中的所有超链接并将它们打印出来。典型的链接看起来像这样：

```go
<a href="https://www.devdungeon.com">DevDungeon</a>  
```

在 HTML 中，`a`标签代表**锚点**，`href`属性代表**超链接引用**。可能会有一个没有`href`属性但只有`name`属性的锚标签。这些被称为书签，或命名锚点，用于跳转到同一页面上的位置。我们将忽略这些，因为它们只在同一页面内链接。`target`属性只是一个可选项，用于指定在哪个窗口或选项卡中打开链接。在这个例子中，我们只对`href`值感兴趣：

```go
// Load a URL and list all links found
package main

import (
   "fmt"
   "github.com/PuerkitoBio/goquery"
   "log"
   "net/http"
   "os"
)

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("Find all links in a web page")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Fetch the URL
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   // Extract all links
   doc, err := goquery.NewDocumentFromReader(response.Body)
   if err != nil {
      log.Fatal("Error loading HTTP response body. ", err)
   }

   // Find and print all links
   doc.Find("a").Each(func(i int, s *goquery.Selection) {
      href, exists := s.Attr("href")
      if exists {
         fmt.Println(href)
      }
   })
} 
```

# 在网页中查找文档

文档也是感兴趣的点。您可能希望抓取一个网页并查找文档。文字处理器文档、电子表格、幻灯片演示文稿、CSV、文本和其他文件可能包含各种目的的有用信息。

以下示例将通过 URL 搜索并根据链接中的文件扩展名搜索文档。在顶部定义了一个全局变量，方便列出应搜索的所有扩展名。自定义要搜索的扩展名列表以搜索目标文件类型。考虑扩展应用程序以从文件中获取文件扩展名列表，而不是硬编码。在尝试查找敏感信息时，您会寻找哪些其他文件扩展名？

以下是此示例的代码实现：

```go
// Load a URL and list all documents 
package main

import (
   "fmt"
   "github.com/PuerkitoBio/goquery"
   "log"
   "net/http"
   "os"
   "strings"
)

var documentExtensions = []string{"doc", "docx", "pdf", "csv", 
   "xls", "xlsx", "zip", "gz", "tar"}

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("Find all links in a web page")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Fetch the URL
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   // Extract all links
   doc, err := goquery.NewDocumentFromReader(response.Body)
   if err != nil {
      log.Fatal("Error loading HTTP response body. ", err)
   }

   // Find and print all links that contain a document
   doc.Find("a").Each(func(i int, s *goquery.Selection) {
      href, exists := s.Attr("href")
      if exists && linkContainsDocument(href) {
         fmt.Println(href)
      }
   })
} 

func linkContainsDocument(url string) bool {
   // Split URL into pieces
   urlPieces := strings.Split(url, ".")
   if len(urlPieces) < 2 {
      return false
   }

   // Check last item in the split string slice (the extension)
   for _, extension := range documentExtensions {
      if urlPieces[len(urlPieces)-1] == extension {
         return true
      }
   }
   return false
} 
```

# 列出页面标题和标题

标题是定义网页层次结构的主要结构元素，`<h1>`是最高级别，`<h6>`是最低级别。在 HTML 页面的`<title>`标签中定义的标题是显示在浏览器标题栏中的内容，它不是渲染页面的一部分。

通过列出标题和标题，您可以快速了解页面的主题是什么，假设它们正确格式化了他们的 HTML。应该只有一个`<title>`和一个`<h1>`标签，但并非每个人都符合标准。

此程序加载网页，然后将标题和所有标题打印到标准输出。尝试运行此程序针对几个 URL，并查看是否能够通过查看标题快速了解内容：

```go
package main

import (
   "fmt"
   "github.com/PuerkitoBio/goquery"
   "log"
   "net/http"
   "os"
)

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("List all headings (h1-h6) in a web page")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Fetch the URL
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   doc, err := goquery.NewDocumentFromReader(response.Body)
   if err != nil {
      log.Fatal("Error loading HTTP response body. ", err)
   }

   // Print title before headings
   title := doc.Find("title").Text()
   fmt.Printf("== Title ==\n%s\n", title)

   // Find and list all headings h1-h6
   headingTags := [6]string{"h1", "h2", "h3", "h4", "h5", "h6"}
   for _, headingTag := range headingTags {
      fmt.Printf("== %s ==\n", headingTag)
      doc.Find(headingTag).Each(func(i int, heading *goquery.Selection) {
         fmt.Println(" * " + heading.Text())
      })
   }

} 
```

# 爬取存储最常见单词的站点上的页面

此程序打印出网页上使用的所有单词列表，以及每个单词在页面上出现的次数。这将搜索所有段落标签。如果搜索整个正文，它将将所有 HTML 代码视为单词，这会使数据混乱，并且实际上并不帮助您了解站点的内容。它会修剪字符串中的空格、逗号、句号、制表符和换行符。它还会尝试将所有单词转换为小写以规范化数据。

对于它找到的每个段落，它将拆分文本内容。每个单词存储在将字符串映射到整数计数的映射中。最后，映射被打印出来，列出了每个单词以及在页面上看到了多少次：

```go
package main

import (
   "fmt"
   "github.com/PuerkitoBio/goquery"
   "log"
   "net/http"
   "os"
   "strings"
)

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("List all words by frequency from a web page")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Fetch the URL
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   doc, err := goquery.NewDocumentFromReader(response.Body)
   if err != nil {
      log.Fatal("Error loading HTTP response body. ", err)
   }

   // Find and list all headings h1-h6
   wordCountMap := make(map[string]int)
   doc.Find("p").Each(func(i int, body *goquery.Selection) {
      fmt.Println(body.Text())
      words := strings.Split(body.Text(), " ")
      for _, word := range words {
         trimmedWord := strings.Trim(word, " \t\n\r,.?!")
         if trimmedWord == "" {
            continue
         }
         wordCountMap[strings.ToLower(trimmedWord)]++

      }
   })

   // Print all words along with the number of times the word was seen
   for word, count := range wordCountMap {
      fmt.Printf("%d | %s\n", count, word)
   }

} 
```

# 在页面中打印外部 JavaScript 文件列表

检查包含在页面上的 JavaScript 文件的 URL 可以帮助您确定应用程序的指纹或确定加载了哪些第三方库。此程序将列出网页中引用的外部 JavaScript 文件。外部 JavaScript 文件可能托管在同一域上，也可能从远程站点加载。它检查所有`script`标签的`src`属性。

例如，如果 HTML 页面具有以下标签：

```go
<script src="img/jquery.min.js"></script>  
```

`src`属性的 URL 将被打印：

```go
/ajax/libs/jquery/3.2.1/jquery.min.js
```

请注意，`src`属性中的 URL 可能是完全限定的或相对 URL。

以下程序加载 URL，然后查找所有`script`标签。它将打印它找到的每个脚本的`src`属性。这将仅查找外部链接的脚本。要打印内联脚本，请参考文件底部关于`script.Text()`的注释。尝试运行此程序针对您经常访问的一些网站，并查看它们嵌入了多少外部和第三方脚本：

```go
package main

import (
   "fmt"
   "github.com/PuerkitoBio/goquery"
   "log"
   "net/http"
   "os"
)

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("List all JavaScript files in a webpage")
      fmt.Println("Usage: " + os.Args[0] + " <url>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   url := os.Args[1]

   // Fetch the URL
   response, err := http.Get(url)
   if err != nil {
      log.Fatal("Error fetching URL. ", err)
   }

   doc, err := goquery.NewDocumentFromReader(response.Body)
   if err != nil {
      log.Fatal("Error loading HTTP response body. ", err)
   }

   // Find and list all external scripts in page
   fmt.Println("Scripts found in", url)
   fmt.Println("==========================")
   doc.Find("script").Each(func(i int, script *goquery.Selection) {

      // By looking only at the script src we are limiting
      // the search to only externally loaded JavaScript files.
      // External files might be hosted on the same domain
      // or hosted remotely
      src, exists := script.Attr("src")
      if exists {
         fmt.Println(src)
      }

      // script.Text() will contain the raw script text
      // if the JavaScript code is written directly in the
      // HTML source instead of loaded from a separate file
   })
} 
```

此示例查找由`src`属性引用的外部脚本，但有些脚本直接在 HTML 中的开放和关闭`script`标签之间编写。这些类型的内联脚本不会有引用`src`属性。使用`goquery`对象上的`.Text()`函数获取内联脚本文本。请参考此示例底部，其中提到了`script.Text()`。

这个程序不打印内联脚本，而是只关注外部加载的脚本，因为那是引入许多漏洞的地方。加载远程 JavaScript 是有风险的，应该只能使用受信任的来源。即使如此，我们也不能百分之百保证远程内容提供者永远不会被入侵并提供恶意代码。考虑雅虎这样的大公司，他们公开承认他们的系统过去曾受到入侵。雅虎还有一个托管**内容传送网络**（**CDN**）的广告网络，为大量网站提供 JavaScript 文件。这将是攻击者的主要目标。在包含远程 JavaScript 文件时，请考虑这些风险。

# 深度优先爬行

深度优先爬行是指优先考虑相同域上的链接，而不是指向其他域的链接。在这个程序中，外部链接完全被忽略，只有相同域上的路径或相对链接被跟踪。

在这个例子中，唯一的路径被存储在一个切片中，并在最后一起打印出来。在爬行过程中遇到的任何错误都会被忽略。由于链接格式不正确，经常会遇到错误，我们不希望整个程序在这样的错误上退出。

不要试图使用字符串函数手动解析 URL，而是利用`url.Parse()`函数。它会将主机与路径分开。

在爬行时，忽略任何查询字符串和片段以减少重复。查询字符串在 URL 中用问号标记，片段，也称为书签，用井号标记。这个程序是单线程的，不使用 goroutines：

```go
// Crawl a website, depth-first, listing all unique paths found
package main

import (
   "fmt"
   "github.com/PuerkitoBio/goquery"
   "log"
   "net/http"
   "net/url"
   "os"
   "time"
)

var (
   foundPaths  []string
   startingUrl *url.URL
   timeout     = time.Duration(8 * time.Second)
)

func crawlUrl(path string) {
   // Create a temporary URL object for this request
   var targetUrl url.URL
   targetUrl.Scheme = startingUrl.Scheme
   targetUrl.Host = startingUrl.Host
   targetUrl.Path = path

   // Fetch the URL with a timeout and parse to goquery doc
   httpClient := http.Client{Timeout: timeout}
   response, err := httpClient.Get(targetUrl.String())
   if err != nil {
      return
   }
   doc, err := goquery.NewDocumentFromReader(response.Body)
   if err != nil {
      return
   }

   // Find all links and crawl if new path on same host
   doc.Find("a").Each(func(i int, s *goquery.Selection) {
      href, exists := s.Attr("href")
      if !exists {
         return
      }

      parsedUrl, err := url.Parse(href)
      if err != nil { // Err parsing URL. Ignore
         return
      }

      if urlIsInScope(parsedUrl) {
         foundPaths = append(foundPaths, parsedUrl.Path)
         log.Println("Found new path to crawl: " +
            parsedUrl.String())
         crawlUrl(parsedUrl.Path)
      }
   })
}

// Determine if path has already been found
// and if it points to the same host
func urlIsInScope(tempUrl *url.URL) bool {
   // Relative url, same host
   if tempUrl.Host != "" && tempUrl.Host != startingUrl.Host {
      return false // Link points to different host
   }

   if tempUrl.Path == "" {
      return false
   }

   // Already found?
   for _, existingPath := range foundPaths {
      if existingPath == tempUrl.Path {
         return false // Match
      }
   }
   return true // No match found
}

func main() {
   // Load command line arguments
   if len(os.Args) != 2 {
      fmt.Println("Crawl a website, depth-first")
      fmt.Println("Usage: " + os.Args[0] + " <startingUrl>")
      fmt.Println("Example: " + os.Args[0] + 
         " https://www.devdungeon.com")
      os.Exit(1)
   }
   foundPaths = make([]string, 0)

   // Parse starting URL
   startingUrl, err := url.Parse(os.Args[1])
   if err != nil {
      log.Fatal("Error parsing starting URL. ", err)
   }
   log.Println("Crawling: " + startingUrl.String())

   crawlUrl(startingUrl.Path)

   for _, path := range foundPaths {
      fmt.Println(path)
   }
   log.Printf("Total unique paths crawled: %d\n", len(foundPaths))
} 
```

# 广度优先爬行

广度优先爬行是指优先考虑查找新域并尽可能扩展，而不是以深度优先的方式继续通过单个域。

编写一个广度优先爬行器将根据本章提供的信息留给读者作为练习。它与上一节的深度优先爬行器并没有太大的不同，只是应该优先考虑指向以前未见过的域的 URL。

有几点需要记住。如果不小心，不设置最大限制，你可能最终会爬行宠字节的数据！你可能选择忽略子域，或者你可以进入一个具有无限子域的站点，你永远不会离开。

# 如何防止网页抓取

要完全防止网页抓取是困难的，甚至是不可能的。如果你从 Web 服务器提供信息，总会有一种方式可以以编程方式提取数据。你只能设置障碍。这相当于混淆，你可以说这不值得努力。

JavaScript 使得这更加困难，但并非不可能，因为 Selenium 可以驱动真实的 Web 浏览器，而像 PhantomJS 这样的框架可以用来执行 JavaScript。

需要身份验证可以帮助限制抓取的数量。速率限制也可以提供一些缓解。可以使用诸如 iptables 之类的工具进行速率限制，也可以在应用程序级别进行，基于 IP 地址或用户会话。

检查客户端提供的用户代理是一个浅显的措施，但可以有所帮助。丢弃带有关键字的用户代理的请求，如`curl`，`wget`，`go`，`python`，`ruby`和`perl`。阻止或忽略这些请求可以防止简单的机器人抓取您的网站，但客户端可以伪造或省略他们的用户代理，以便轻松绕过。

如果你想更进一步，你可以使 HTML 的 ID 和类名动态化，这样它们就不能用来查找特定信息。经常改变你的 HTML 结构和命名，玩起*猫鼠游戏*，让爬虫的工作变得不值得。这并不是一个真正的解决方案，我不建议这样做，但是值得一提，因为这会让爬虫感到恼火。

你可以使用 JavaScript 来检查关于客户端的信息，比如屏幕尺寸，在呈现数据之前。如果屏幕尺寸是 1 x 1 或 0 × 0，或者是一些奇怪的尺寸，你可以假设这是一个机器人，并拒绝呈现内容。

蜜罐表单是检测机器人行为的另一种方法。使用 CSS 或`hidden`属性隐藏表单字段，并检查这些字段是否提供了值。如果这些字段中有数据，就假设是机器人在填写所有字段并忽略该请求。

另一个选择是使用图像来存储信息而不是文本。例如，如果你只输出一个饼图的图像，对于某人来说要爬取数据就会更加困难，而当你将数据输出为 JSON 对象并让 JavaScript 渲染饼图时，情况就不同了。爬虫可以直接获取 JSON 数据。文本也可以放在图像中，以防止文本被爬取和防止关键字文本搜索，但是**光学字符识别**（**OCR**）可以通过一些额外的努力来解决这个问题。

根据应用程序，前面提到的一些技术可能会有用。

# 总结

阅读完本章后，你现在应该了解了网络爬虫的基础知识，比如执行 HTTP `GET`请求和使用字符串匹配或正则表达式查找 HTML 注释、电子邮件和其他关键字。你还应该了解如何提取 HTTP 头并设置自定义头以设置 cookie 和自定义用户代理字符串。此外，你应该了解指纹识别的基本概念，并对如何根据提供的源代码收集有关 Web 应用程序的信息有一些想法。

经过这一章的学习，你应该也了解了使用`goquery`包在 DOM 中以 jQuery 风格查找 HTML 元素的基础知识。你应该能够轻松地在网页中找到链接，找到文档，列出标题和标题，找到 JavaScript 文件，并找到广度优先和深度优先爬取之间的区别。

关于爬取公共网站的一点说明——要尊重。不要通过发送大批量请求或让爬虫不受限制地运行来给网站带来不合理的流量。在你编写的程序中设置合理的速率限制和最大页面计数限制，以免过度拖累远程服务器。如果你是为了获取数据而进行爬取，请始终检查是否有 API 可用。API 更高效，旨在以编程方式使用。

你能想到其他应用本章中所讨论的工具的方式吗？你能想到可以添加到提供的示例中的其他功能吗？

在下一章中，我们将探讨主机发现和枚举的方法。我们将涵盖诸如 TCP 套接字、代理、端口扫描、横幅抓取和模糊测试等内容。
