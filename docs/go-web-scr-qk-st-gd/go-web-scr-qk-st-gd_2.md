# 请求/响应循环

在构建网络爬虫之前，您必须花一点时间思考互联网是如何工作的。在其核心，互联网是一组通过**域名查找系统**（**DNS**）服务器连接在一起的计算机网络。当您想访问一个网站时，您的浏览器将网站 URL 发送到 DNS 服务器，URL 被翻译成 IP 地址，然后您的浏览器发送请求到该 IP 地址的机器。这台机器称为 Web 服务器，接收并检查请求，并决定发送什么内容回到您的浏览器。然后您的浏览器解析服务器发送的信息，并根据数据的格式在屏幕上显示内容。Web 服务器和浏览器之间能够通信是因为它们遵守了一套全球规则，称为 HTTP。在本章中，您将学习 HTTP 请求和响应循环的一些关键点。

本章涵盖以下主题：

+   HTTP 请求是什么样子的？

+   HTTP 响应是什么样子的？

+   HTTP 状态码是什么？

+   Go 中的 HTTP 请求/响应是什么样子的？

# HTTP 请求是什么样子的？

当客户端（如浏览器）从服务器请求网页时，它发送一个 HTTP 请求。这种请求的格式定义了一个操作、一个资源和 HTTP 协议的版本。一些 HTTP 请求包括额外的信息供服务器处理，如查询或特定的元数据。根据操作，您还可能向服务器发送新信息供服务器处理。

# HTTP 请求方法

目前有九种 HTTP 请求方法，它们定义了客户端期望的一般操作。每种方法都带有特定的含义，告诉服务器应该如何处理请求。这九种请求方法如下：

+   `GET`

+   `POST`

+   `PUT`

+   `DELETE`

+   `HEAD`

+   `CONNECT`

+   `TRACE`

+   `OPTIONS`

+   `PATCH`

您将需要的最常见的请求方法是`GET`、`POST`和`PUT`。`GET`请求用于从网站检索信息。`POST`和`PUT`请求用于向网站发送信息，例如用户登录数据。这些类型的请求通常只在提交某种形式的表单数据时发送，我们将在本书的后面章节中介绍这些内容。

在构建网络爬虫时，您将大部分时间向服务器发送 HTTP `GET`请求以获取网页。对于[`example.com/index.html`](http://example.com/index.html)的最简单的`GET`请求示例如下：

```go
GET /index.html HTTP/1.1
Host: example.com
```

客户端使用`GET`操作将此消息发送到服务器，以使用 HTTP 协议的`1.1`版本获取`index.html`资源。HTTP 请求的第一行称为请求行，是 HTTP 请求的核心。

# HTTP 头

在请求行下面是一系列键值对，提供了描述请求应该如何处理的元数据。这些元数据字段称为 HTTP 头。在我们之前的简单请求中，我们有一个单独的 HTTP 头，定义了我们要到达的目标主机。这些信息并不是 HTTP 协议要求的，但几乎总是发送以提供关于谁应该接收请求的澄清。

如果您检查您的 Web 浏览器发送的 HTTP 请求，您将看到更多的 HTTP 头。以下是 Google Chrome 浏览器发送给相同的[example.com](http://example.com)网站的示例：

```go
GET /index.html HTTP/1.1
Host: example.com
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
If-None-Match: "1541025663+gzip"
If-Modified-Since: Fri, 09 Aug 2013 23:54:35 GMT
```

HTTP 请求的基础是相同的，但是您的浏览器提供了更多的请求头，主要与如何处理缓存的 HTML 页面有关。我们将在接下来的章节中更详细地讨论其中一些头部。

服务器读取请求并处理所有头部，以决定如何响应您的请求。在最基本的情况下，服务器将回复说您的请求是 OK，并传送`index.html`的内容。

# 查询参数

对于一些 HTTP 请求，客户端需要提供额外的信息以便细化请求。这通常有两种不同的方式。对于 HTTP `GET`请求，有一种定义的方式来在请求中包含额外的信息。在 URL 的末尾放置一个`?`定义了 URL 资源的末尾，接下来的部分定义了查询参数。这些参数是键值对，定义了发送到服务器的额外信息。键值对的书写格式如下：

```go
key1=value1&key2=value2&key3 ...
```

在执行搜索时，您会经常看到这种情况。举个假设的例子，如果您在一个网站上搜索鞋子，您可能会遇到一个分页的结果页面，URL 可能看起来像这样：

```go
https://buystuff.com/product_search?keyword=shoes&page=1
```

注意资源是`product_search`，后面是`keyword`和`page`的查询参数。这样，您可以通过调整查询来收集所有页面的产品。

查询参数由网站定义。并没有所有网站都必须具有的标准参数，因此根据您正在抓取的网站的不同，您需要进行一些调查。

# 请求主体

查询参数通常只用于 HTTP `GET`请求。对于您向服务器发送数据的请求，比如`POST`和`PUT`请求，您将发送一个包含所有额外信息的请求主体。请求主体放置在 HTTP 请求的头部之后，它们之间有一行空格。以下是一个假设的用于登录到一个虚构网站的`POST`请求：

```go
POST /login HTTP/1.1
Host: myprotectedsite.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 38

username=myuser&password=supersecretpw
```

在这个请求中，我们将我们的`username`和`password`发送到`myprotectedsite.com/login`。这个请求的头部必须描述请求主体，以便服务器能够处理它。在这种情况下，我们声明请求主体以`x-www-form-urlencoded`格式，这是*查询参数*部分中使用的相同格式。我们可以使用其他格式，比如`JSON`或`XML`甚至纯文本，但前提是服务器支持。`x-www-form-urlencoded`格式是最广泛支持的，通常是一个安全的选择。我们在头部中定义的第二个参数是请求主体的字节长度。这允许服务器有效地准备处理数据，或者如果请求太大，则完全拒绝请求。

如果您熟悉结构，Go 标准库对构建 HTTP 请求有很好的支持。我们将在本章后面重新讨论如何做到这一点。

# HTTP 响应是什么样子的？

当服务器响应您的请求时，它通常会提供一个状态码、一些响应头和资源的内容。继续使用我们之前对[`www.example.com/index.html`](http://www.example.com/index.html)的请求，您将能够逐节看到典型响应的样子。

# 状态行

HTTP 响应的第一行称为状态行，通常看起来像这样：

```go
HTTP/1.1 200 OK
```

首先，它告诉您服务器正在使用的 HTTP 协议的版本。这应该始终与客户端 HTTP 请求发送的版本匹配。在这种情况下，我们的服务器正在使用版本`1.1`。接下来是 HTTP 状态码。这是用来指示响应状态的代码。大多数情况下，您应该看到状态码为 200，表示请求成功，并且会有一个响应主体跟随。这并不总是这样，我们将在下一节更深入地了解 HTTP 状态码。OK 是状态码的可读描述，仅供您参考使用。

# 响应头

HTTP 响应头跟随状态行，看起来与 HTTP 请求头非常相似。它们也提供了特定于响应的元数据，就像请求头一样。以下是我们[example.com](http://example.com)响应的头部：

```go
Accept-Ranges: bytes
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Mon, 29 Oct 2018 13:31:23 GMT
Etag: "1541025663"
Expires: Mon, 05 Nov 2018 13:31:23 GMT
Last-Modified: Fri, 09 Aug 2013 23:54:35 GMT
Server: ECS (dca/53DB)
Vary: Accept-Encoding
X-Cache: HIT
Content-Length: 1270
```

在此响应中，您可以看到一些描述页面内容、如何缓存以及剩余数据大小的标头。在接收到数据后，这些信息对于处理数据是有用的。

# 响应主体

响应的其余部分是实际呈现`index.html`的网页。您的浏览器将使用此内容绘制网页本身的文本、图像和样式，但是对于爬取的目的，这并非必要。响应主体的缩写版本类似于这样：

```go
<!doctype html>
<html>
<head>
 <title>Example Domain</title>
 <meta charset="utf-8" />
 <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
 <meta name="viewport" content="width=device-width, initial-scale=1" />
 <!-- The <style> section was removed for brevity -->
</head>
<body>
 <div>
  <h1>Example Domain</h1>
<p>This domain is established to be used for illustrative examples in
   documents. You may use this domain in examples without prior 
   coordination or asking for permission.</p>
<p><a href="http://www.iana.org/domains/example">More information...</a></p>
 </div>
</body>
</html>
```

大多数情况下，您将处理具有状态码 200 的 Web 服务器的响应，表示请求正常。但是，偶尔您会遇到其他状态代码，您的网络爬虫应该知道。

# HTTP 状态码是什么？

HTTP 状态码用于通知 HTTP 客户端 HTTP 请求的状态。在某些情况下，HTTP 服务器需要通知客户端请求未被理解，或者需要采取额外的操作才能获得完整的响应。HTTP 状态码分为四个独立的范围，每个范围覆盖特定类型的响应。

# 100–199 范围

这些代码用于向 HTTP 客户端提供关于如何传递请求的信息。这些代码通常由 HTTP 客户端自己处理，在您的网络爬虫需要担心它们之前就会被处理。

例如，客户端可能希望使用 HTTP 2.0 协议发送请求，并请求服务器进行更改。如果服务器支持 HTTP 2.0，它将以 101 状态代码响应，表示切换协议。这样的情况将由客户端在后台处理，因此您无需担心。

# 200–299 范围

`200-299`范围的状态代码表示请求已成功处理，没有问题。在这里需要注意的最重要的代码是 200 状态代码。这意味着您将收到一个响应主体，并且一切都很完美！

在某些情况下，您可能正在下载大文件的块（考虑到几十亿字节的规模），在这种情况下，成功的响应应该是 206，表示服务器正在返回原始文件的部分内容。

此范围内的其他代码表示请求成功，但服务器正在后台处理信息，或者根本没有内容。这些通常不会在网络爬取中看到。

# 300–399 范围

如果您遇到此范围内的状态代码，这意味着请求已被理解，但需要采取额外步骤才能获得实际内容。您在这里遇到的最常见情况是重定向。

301、302、307 和 308 状态代码都表示您正在寻找的资源可以在另一个位置找到。在此响应的标头中，服务器应指示响应标头中的最终位置在哪里。例如，301 响应可能如下所示：

```go
HTTP/1.1 301 Moved Permanently
Location: /blogs/index.html
Content-Length: 190

<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<h1>301 Moved Permanently</h1>
Please go to <a href="/blogs/index.html">/blogs/index.html</a>
</body>
</html>
```

服务器包括一个`Location`标头，告诉客户端资源的位置已移动，并且客户端应该将下一个请求发送到该位置。在大多数情况下，这里的内容可以被忽略。

此范围内的其他状态代码与代理和缓存信息的使用有关，这两者我们将在未来的章节中讨论。

# 400–499 范围

当您遇到此范围内的状态代码时，您应该关注。`400`范围表示您的请求出了问题。许多不同的问题可能触发这些响应，例如格式不佳、身份验证问题或异常请求。服务器将这些代码发送回给它们的客户端，告诉它们不会满足请求，因为某些东西看起来可疑。

您可能已经熟悉的一个状态码是 404 Not Found。当您请求服务器似乎找不到的资源时就会出现这种情况。这可能是由于资源拼写错误或页面根本不存在。有时，网站会在服务器上更新文件，可能忘记更新网页中与新位置的链接。这可能导致**损坏的链接**，尤其是当页面链接到外部网站时，这种情况特别常见。

在这个范围内，您可能会遇到的其他常见状态码是 401 Unauthorized 和 403 Forbidden。在这两种情况下，这意味着您正在尝试访问需要适当身份验证凭据的页面。网络上有许多不同形式的身份验证，本书将在未来的章节中仅涵盖基础知识。

我想要在这个范围内强调的最后一个状态码是 429 Too Many Requests。一些 Web 服务器配置了速率限制，这意味着您在一定时间内只能维持一定数量的请求。如果您超过了这个速率，那么不仅会对 Web 服务器造成不合理的压力，而且还会暴露您的网络爬虫，使其有被列入黑名单的风险。遵守适当的网络爬取礼仪对您和目标网站都有益处。

# 500-599 范围

这个范围内的状态码通常表示与服务器本身相关的错误。尽管这些错误通常不是您的错，但您仍需要意识到它们并适应情况。

状态码 502 Bad Gateway 和 503 Service Temporarily Unavailable 表示服务器由于服务器内部问题而无法生成资源。这并不一定意味着资源不存在，或者您无权访问它。当遇到这些代码时，最好将请求搁置，稍后再试。如果您经常看到这些代码，您可能希望停止所有请求，让服务器解决其问题。

有时候网页服务器会因为没有特定的原因而出现故障。在这种情况下，您将收到 500 Internal Server Error 状态码。这些错误是通用的，通常是服务器代码崩溃的原因。在这种情况下，重试您的请求或让您的抓取器暂时停止也是相关的建议。

# 在 Go 中，HTTP 请求/响应是什么样子的？

现在您已经熟悉了 HTTP 请求和响应的基础知识，是时候看看在 Go 中是什么样子了。Go 中的标准库提供了一个名为`net/http`的包，其中包含了构建客户端所需的所有工具，可以从 Web 服务器请求页面并以极少的努力处理响应。

让我们看一下本章开头的示例，我们在访问[`www.example.com/index.html`](http://www.example.com/index.html)的网页。底层的 HTTP 请求指示[example.com](http://example.com)的 Web 服务器`GET` `index.html`资源：

```go
GET /index.html HTTP/1.1
Host: example.com
```

使用 Go 的`net/http`包，您可以使用以下代码行：

```go
r, err := http.Get("http://www.example.com/index.html")
```

Go 编程语言允许从单个函数返回多个变量。这也是通常抛出和处理错误的方式。

这是使用`net/http`包的默认 HTTP 客户端请求`index.html`资源，返回两个对象：HTTP 响应（`r`）和错误（`err`）。在 Go 中，错误是作为值返回的，而不是被其他代码抛出和捕获。如果`err`等于`nil`，那么我们知道与 Web 服务器通信没有问题。

让我们看一下本章开头的响应。如果请求成功，服务器将返回类似以下内容：

```go
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Mon, 29 Oct 2018 13:31:23 GMT
Etag: "1541025663"
Expires: Mon, 05 Nov 2018 13:31:23 GMT
Last-Modified: Fri, 09 Aug 2013 23:54:35 GMT
Server: ECS (dca/53DB)
Vary: Accept-Encoding
X-Cache: HIT
Content-Length: 1270

<!doctype html>
<html>
<head>
 <title>Example Domain</title>
 <meta charset="utf-8" />
 <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
 <meta name="viewport" content="width=device-width, initial-scale=1" />
 <!-- The <style> section was removed for brevity -->
</head>
<body>
 <div>
 <h1>Example Domain</h1>
 <p>This domain is established to be used for illustrative examples in
    documents. You may use this
    domain in examples without prior coordination or asking for
    permission.</p>
 <p><a href="http://www.iana.org/domains/example">More information...</a></p>
 </div>
</body>
</html>
```

所有这些信息都包含在`r`变量中，这是从`http.Get()`函数返回的`*http.Response`。让我们来看看 Go 中`http.Response`对象的定义。以下`struct`在 Go 标准库中定义：

```go
type Response struct {
    Status string
    StatusCode int
    Proto string
    ProtoMajor int
    ProtoMinor int
    Header Header
    Body io.ReadCloser
    ContentLength int64
    TransferEncoding []string
    Close bool
    Uncompressed bool
    Trailer Header
    Request *Request
    TLS *tls.ConnectionState
}
```

`http.Response`对象包含处理 HTTP 响应所需的所有字段。特别是，`StatusCode`，`Header`和`Body`在爬取中会很有用。让我们在一个简单的例子中将请求和响应放在一起，将`index.html`文件保存到您的计算机上。

# 一个简单的请求示例

在您设置的`$GOPATH/src`文件夹中，创建一个名为`simplerequest`的文件夹。在`simplerequest`中，创建一个名为`main.go`的文件。将`main.go`的内容设置为以下代码：

```go
package main

import (
 "log"
 "net/http"
 "os"
)

func main() {
 // Create the variables for the response and error
 var r *http.Response
 var err error

 // Request index.html from example.com
 r, err = http.Get("http://www.example.com/index.html")

 // If there is a problem accessing the server, kill the program and print the error the console
 if err != nil {
  panic(err)
 }

 // Check the status code returned by the server
 if r.StatusCode == 200 {
  // The request was successful!
  var webPageContent []byte

  // We know the size of the response is 1270 from the previous example
  var bodyLength int = 1270

  // Initialize the byte array to the size of the data
  webPageContent = make([]byte, bodyLength)

  // Read the data from the server
  r.Body.Read(webPageContent)

  // Open a writable file on your computer (create if it does not 
     exist)
  var out *os.File
  out, err = os.OpenFile("index.html", os.O_CREATE|os.O_WRONLY, 0664)

  if err != nil {
   panic(err)
  }

  // Write the contents to a file
  out.Write(webPageContent)
  out.Close()
 } else {
  log.Fatal("Failed to retrieve the webpage. Received status code", 
  r.Status)
 }
}
```

这里给出的示例有点冗长，以便向您展示 Go 编程的基础知识。随着您在本书中的进展，您将了解到一些技巧，使您的代码更加简洁。

您可以在`terminal`窗口中输入以下命令来从`simplerequest`文件夹内运行此代码：

```go
go run main.go 
```

如果一切顺利，您不应该看到打印的消息，应该有一个名为`index.html`的新文件，其中包含响应主体的内容。您甚至可以用 Web 浏览器打开该文件！

有了这些基础知识，您应该可以创建一个在 Go 中可以使用几行代码创建 HTTP 请求和读取 HTTP 响应的网络爬虫。

# 摘要

在本章中，我们介绍了 HTTP 请求和响应的基本格式。我们还看到了如何在 Go 中进行 HTTP 请求，以及`http.Response`结构如何与真实的 HTTP 响应相关联。最后，我们创建了一个小程序，向[`www.example.com/index.html`](http://www.example.com/index.html)发送了一个 HTTP 响应，并处理了 HTTP 响应。有关完整的 HTTP 规范，我鼓励您访问[`www.w3.org/Protocols/`](https://www.w3.org/Protocols/)。

在第三章中，*网络爬虫礼仪*，我们将看到成为网络良好公民的最佳实践。
