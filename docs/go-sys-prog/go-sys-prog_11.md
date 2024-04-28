# 第十一章：在 Go 中编写 Web 应用程序

在上一章中，我们讨论了许多与 goroutines 和通道相关的高级主题，以及共享内存和互斥锁。

本章的主要内容是在 Go 中开发 Web 应用程序。然而，本章还将讨论如何在 Go 程序中与两个流行的数据库进行交互。Go 标准库提供了可以帮助你使用更高级函数开发 Web 应用程序的包，这意味着你可以通过调用几个带有正确参数的 Go 函数来做复杂的事情，比如读取网页。虽然这种编程方式隐藏了请求背后的复杂性，并且对细节的控制较少，但它允许你使用更少的代码开发复杂的应用程序，这也导致程序中的错误更少。

然而，由于本书是关于系统编程的，本章不会深入讨论：你可以将所呈现的信息视为任何想学习在 Go 中进行 Web 开发的人的良好起点。

更具体地说，本章将讨论以下主题：

+   为 MySQL 数据库管理员创建一个 Go 实用程序

+   管理 MongoDB 数据库

+   使用 Go MongoDB 驱动程序与 MongoDB 数据库通信

+   在 Go 中创建 Web 服务器

+   在 Go 中创建 Web 客户端

+   `http.ServeMux`类型

+   处理 Go 中的 JSON 数据

+   `net/http`包

+   `html/template` Go 标准包

+   开发一个在给定关键字中搜索网页的命令行实用程序

# 什么是 Web 应用程序？

Web 应用程序是一个客户端-服务器软件应用程序，其中客户端部分在 Web 浏览器上运行。Web 应用程序包括网络邮件、即时通讯服务和在线商店。

# 关于 net/http Go 包

本章的主角将是`net/http`包，它可以帮助你在 Go 中编写 Web 应用程序。然而，如果你对在较低级别处理 TCP/IP 连接感兴趣，那么你应该去第十二章，*网络编程*，它讨论使用较低级别函数调用开发 TCP/IP 应用程序。

`net/http`包提供了一个内置的 Web 服务器和一个内置的 Web 客户端，它们都非常强大。`http.Get()`方法可用于发出 HTTP 和 HTTPS 请求，而`http.ListenAndServe()`函数可用于通过指定服务器将监听的 IP 地址和 TCP 端口以及处理传入请求的函数来创建简单的 Web 服务器。

另一个非常方便的包是`html/template`，它是 Go 标准库的一部分，允许你使用 Go HTML 模板文件生成 HTML 输出。

# 在 Go 中开发 Web 客户端

在本节中，你将学习如何在 Go 中开发 Web 客户端，以及如何超时处理需要太长时间才能完成的 Web 连接。

# 获取单个 URL

在本小节中，你将学习如何使用`http.Get()`函数读取单个网页，这将在`getURL.go`程序中进行演示。该实用程序将分为四个部分；程序的第一部分是预期的序言：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "net/http" 
   "os" 
   "path/filepath" 
) 
```

虽然这里没有什么新东西，但你可能会发现令人印象深刻的是，即使你从互联网读取数据，你也会使用与文件输入和输出操作相关的 Go 包。这背后的解释非常简单：Go 具有统一的接口，用于读取和写入数据，无论数据所在的介质如何。

`getURL.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Printf("Usage: %s URL\n", filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   URL :=os.Args[1] 
   data, err := http.Get(URL) 
```

你想获取的 URL 作为程序的命令行参数给出。此外，你可以看到对`http.Get()`的调用，它完成了所有的脏活！`http.Get()`返回的是一个`Response`变量，实际上是一个具有各种属性和方法的 Go 结构。

第三部分如下：

```go
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } else { 
```

调用`http.Get()`后如果出现错误，这是检查错误的地方。

第四部分包含以下 Go 代码：

```go
         defer data.Body.Close() 
         _, err := io.Copy(os.Stdout, data.Body) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
   } 
}
```

正如您所看到的，`URL`的数据是使用`os.Stdout`写入标准输出的，这是在屏幕上打印数据的首选方式。此外，数据保存在`http.Get()`调用的返回值的`Body`属性中。然而，并非所有的 HTTP 请求都是简单的。如果响应流式传输视频或类似内容，逐段读取它而不是一次性获取所有内容是有意义的。您可以使用`io.Reader`和响应的`Body`部分来实现这一点。

执行`getURL.go`将生成以下原始结果，这就是 Web 浏览器将获得并呈现的内容：

```go
$ go run getURL.go http://www.mtsoukalos.eu/ | head
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML+RDFa 1.0//EN"
  "http://www.w3.org/MarkUp/DTD/xhtml-rdfa-1.dtd">
<html xml:lang="en" version="XHTML+RDFa 1.0" dir="ltr"
xmlns:content=http://purl.org/rss/1.0/modules/content/
. . .
</script>
</body>
</html>
```

一般来说，虽然`getURL.go`可以完成所需的工作，但它的工作方式并不那么复杂，因为它不提供灵活性或创造性的方式。

# 设置超时

在本小节中，您将学习如何为`http.Get()`请求设置超时。出于简单起见，它将基于`getURL.go`的 Go 代码。程序的名称将是`timeoutHTTP.go`，并将以五个部分的形式呈现。

程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "net" 
   "net/http" 
   "os" 
   "path/filepath" 
   "time" 
) 

var timeout = time.Duration(time.Second) 
```

在这里，您将所需的超时时间声明为全局参数，即 1 秒。

`timeoutHTTP.go`的第二部分包含以下 Go 代码：

```go
func Timeout(network, host string) (net.Conn, error) { 
   conn, err := net.DialTimeout(network, host, timeout) 
   if err != nil { 
         return nil, err 
   } 
   conn.SetDeadline(time.Now().Add(timeout)) 
   return conn, nil 
} 
```

在这里，您定义了两种类型的超时，第一种是使用`net.DialTimeout()`定义的，用于客户端连接到服务器所需的时间。第二种是读/写超时，与连接到 Web 服务器后等待获取响应的时间有关：这是使用`conn.SetDeadline()`函数定义的。

所呈现程序的第三部分如下：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Printf("Usage: %s URL\n", filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   URL :=os.Args[1] 
```

程序的第四部分如下：

```go
   t := http.Transport{ 
         Dial: Timeout, 
   } 

   client := http.Client{ 
         Transport: &t, 
   } 
   data, err := client.Get(URL) 
```

在这里，您可以使用`http.Transport`变量定义连接的所需参数。

程序的最后部分包含以下 Go 代码：

```go
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } else { 
         deferdata.Body.Close() 
         _, err := io.Copy(os.Stdout, data.Body) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
   } 
} 
```

该程序的这一部分都是关于错误处理的！

执行`timeoutHTTP.go`将在超时的情况下生成以下输出：

```go
$ go run timeoutHTTP.go http://localhost:8001
Get http://localhost:8001: read tcp [::1]:58018->[::1]:8001: i/o timeout
exit status 100
```

故意在 Web 连接期间创建超时的最简单方法是在 Web 服务器的处理程序函数中调用`time.Sleep()`函数。

# 开发更好的网络客户端

虽然`getURL.go`可以很快地完成所需的工作，并且不需要编写太多的 Go 代码，但它在某种程度上不够灵活或信息丰富。它只是打印一堆原始的 HTML 代码，没有其他信息，也没有将 HTML 代码分成逻辑部分的能力。因此，需要改进`getURL.go`！

新实用程序的名称将是`webClient.go`，并将以五个 Go 代码段的形式呈现给您。

该实用程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net/http" 
   "net/http/httputil" 
   "net/url" 
   "os" 
   "path/filepath" 
   "strings" 
) 
```

`webClient.go`中的 Go 代码的第二部分如下：

```go
func main() { 
   if len(os.Args) != 2 { 
         fmt.Printf("Usage: %s URL\n", filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   URL, err :=url.Parse(os.Args[1]) 
   if err != nil { 
         fmt.Println("Parse:", err) 
         os.Exit(100) 
   } 
```

这里唯一的新内容是使用`url.Parse()`函数，它从给定的字符串创建一个`URL`结构。

`webClient.go`的第三部分包含以下 Go 代码：

```go
   c := &http.Client{} 

   request, err := http.NewRequest("GET", URL.String(), nil) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   httpData, err := c.Do(request) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

在这段 Go 代码中，您首先创建一个`http.Client`变量。然后，您使用`http.NewRequest()`构造一个`GET` HTTP 请求。最后，您使用`Do()`函数发送 HTTP 请求，该函数返回保存在`httpData`变量中的实际响应数据。

该实用程序的第四部分代码如下：

```go
   fmt.Println("Status code:", httpData.Status) 
   header, _ := httputil.DumpResponse(httpData, false) 
   fmt.Print(string(header)) 

   contentType := httpData.Header.Get("Content-Type") 
   characterSet := strings.SplitAfter(contentType, "charset=") 
   fmt.Println("Character Set:", characterSet[1]) 

   if httpData.ContentLength == -1 { 
         fmt.Println("ContentLength in unknown!") 
   } else { 
         fmt.Println("ContentLength:", httpData.ContentLength) 
   } 
```

在这里，您可以使用`Status`属性找到 HTTP 请求的状态代码。然后，您可以对响应的`Header`部分进行一些挖掘，以找到响应的字符集。最后，您可以检查`ContentLength`属性的值，对于动态页面，它等于`-1`：这意味着您事先不知道页面的大小。

程序的最后部分包含以下 Go 代码：

```go
   length := 0 
   var buffer [1024]byte

   r := httpData.Body 
   for { 
         n, err := r.Read(buffer[0:]) 
         if err != nil { 
               fmt.Println(err) 
               break 
         } 
         length = length + n 
   } 
   fmt.Println("Response data length:", length) 
} 
```

在这里，您通过从`Body`读取器中读取数据并计算其数据长度来找到响应的长度。如果要打印响应的内容，这是正确的位置。

执行`webClient.go`将创建以下输出：

```go
$ go run webClient.go invalid
Get invalid: unsupported protocol scheme ""
exit status 100
$ go run webClient.go https://www.mtsoukalos.eu/
Get https://www.mtsoukalos.eu/: dial tcp 109.74.193.253:443: getsockopt: connection refused
exit status 100
$ go run webClient.go http://www.mtsoukalos.eu/
Status code: 200 OK
HTTP/1.1 200 OK
Accept-Ranges: bytes
Age: 0
Cache-Control: no-cache, must-revalidate
Connection: keep-alive
Content-Language: en
Content-Type: text/html; charset=utf-8
Date: Mon, 10 Jul 2017 07:29:48 GMT
Expires: Sun, 19 Nov 1978 05:00:00 GMT
Server: Apache/2.4.10 (Debian) PHP/5.6.30-0+deb8u1 mod_wsgi/4.3.0 Python/2.7.9
Vary: Accept-Encoding
Via: 1.1 varnish-v4
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Generator: Drupal 7 (http://drupal.org)
X-Powered-By: PHP/5.6.30-0+deb8u1
X-Varnish: 6922264

Character Set: utf-8
ContentLength in unknown!
EOF
Response data length: 50176
```

# 一个小型的 web 服务器

够了，关于 Web 客户端的内容：在本节中，您将学习如何在 Go 中开发 Web 服务器！

可以在`webServer.go`中找到一个简单 Web 服务器实现的 Go 代码，并且将以四部分呈现；第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net/http" 
   "os" 
) 
```

第二部分是事情开始变得棘手和奇怪的地方：

```go
func myHandler(w http.ResponseWriter, r *http.Request) { 
   fmt.Fprintf(w, "Serving: %s\n", r.URL.Path) 
   fmt.Printf("Served: %s\n", r.Host) 
} 
```

这是一种处理 HTTP 请求的函数：该函数接受两个参数，一个`http.ResponseWriter`变量和一个指向`http.Request`变量的指针。第一个参数将用于构造 HTTP 响应，而`http.Request`变量保存了服务器接收到的 HTTP 请求的详细信息，包括请求的 URL 和客户端的 IP 地址。

`webServer.go`的第三部分包含以下 Go 代码：

```go
func main() { 
   PORT := ":8001" 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Using default port number: ", PORT) 
   } else { 
         PORT = ":" + arguments[1] 
   } 
```

在这里，您只需处理 web 服务器的端口号：默认端口号是`8001`，除非有命令行参数。

`webServer.go`的最后一部分 Go 代码如下：

```go
   http.HandleFunc("/", myHandler) 
   err := http.ListenAndServe(PORT, nil) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(10) 
   } 
} 
```

`http.HandleFunc()`调用定义了处理程序函数的名称（`myHandler`）以及它将支持的 URL：您可以多次调用`http.HandleFunc()`。当前处理程序支持`/URL`，在 Go 中匹配所有 URL！

在完成`http.HandleFunc()`调用后，您可以准备调用`http.ListenAndServe()`并开始等待传入的连接！如果在`http.ListenAndServe()`函数调用中未指定 IP 地址，则 Web 服务器将侦听计算机的所有配置的网络接口。

执行`webServer.go`将不会生成任何输出，除非您尝试从中获取一些数据：在这种情况下，它将在您的终端上打印日志信息，显示请求的服务器名称（`localhost`）和端口号（`8001`），如下所示：

```go
$ go run webServer.go
Using default port number:  :8001 
Served: localhost:8001 Served: localhost:8001
Served: localhost:8001
```

以下屏幕截图显示了在 Web 浏览器上`webServer.go`的三个输出：

![](img/92cc0e7a-c289-4559-ae6f-6fc6e9d42ad3.png)

使用 webServer.go

但是，如果您使用`wget(1)`或`getURL.go`等命令行实用程序而不是 Web 浏览器，当您尝试连接到 Go Web 服务器时，您将获得以下输出：

```go
$ go run getURL.go http://localhost:8001/
Serving: /
```

您从自定义的 web 服务器中获得的最大优势是安全性，因为当以安全性以及更容易的定制为目标开发时，它们真的很难被黑客攻击。

下一小节将展示如何使用`http.ServeMux`创建 Web 服务器。

# http.ServeMux 类型

在本小节中，您将学习如何使用`http.ServeMux`类型来改进 Go Web 服务器的操作方式。简单地说，`http.ServeMux`是一个 HTTP 请求路由器。

# 使用 http.ServeMux

本节的 Web 服务器实现将使用`http.ServeMux`来支持多个路径，这将在将显示为四部分的`serveMux.go`程序中进行说明。

程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net/http" 
   "time" 
) 
```

`serveMux.go`的第二部分包含以下 Go 代码：

```go
func about(w http.ResponseWriter, r *http.Request) { 
   fmt.Fprintf(w, "This is the /about page at %s\n", r.URL.Path) 
   fmt.Printf("Served: %s\n", r.Host) 
} 

func cv(w http.ResponseWriter, r *http.Request) { 
   fmt.Fprintf(w, "This is the /CV page at %s\n", r.URL.Path) 
   fmt.Printf("Served: %s\n", r.Host) 
} 

func timeHandler(w http.ResponseWriter, r *http.Request) { 
   currentTime := time.Now().Format(time.RFC1123) 
   title := currentTime 
   Body := "The current time is:" 
   fmt.Fprintf(w, "<h1 align=\"center\">%s</h1><h2 align=\"center\">%s</h2>", Body, title) 
   fmt.Printf("Served: %s for %s\n", r.URL.Path, r.Host) 
} 
```

在这里，您有三个 HTTP 处理程序函数的实现。前两个显示静态页面，而第三个显示当前时间，这是一个动态文本。

程序的第三部分如下：

```go
func home(w http.ResponseWriter, r *http.Request) { 
   ifr.URL.Path == "/" { 
         fmt.Fprintf(w, "Welcome to my home page!\n") 
   } else { 
         fmt.Fprintf(w, "Unknown page: %s from %s\n", r.URL.Path, r.Host) 
   } 
   fmt.Printf("Served: %s for %s\n", r.URL.Path, r.Host) 
} 
```

`home()`处理程序函数将必须确保它实际上正在服务于`/Path`，因为`/Path`会捕捉一切！

`serveMux.go`的最后部分包含以下 Go 代码：

```go
func main() { 
   m := http.NewServeMux() 
   m.HandleFunc("/about", about) 
   m.HandleFunc("/CV", cv) 
   m.HandleFunc("/time", timeHandler) 
   m.HandleFunc("/", home) 

   http.ListenAndServe(":8001", m) 
} 
```

在这里，您定义了您的 Web 服务器将支持的路径。请注意，路径区分大小写，并且在前面的代码中最后一个路径会捕捉一切。这意味着如果您首先放置`m.HandleFunc("/", home)`，您将无法匹配其他任何内容。简单地说，`m.HandleFunc()`语句的顺序很重要。还要注意，如果您想同时支持`/about`和`/about/`，您应该同时拥有`m.HandleFunc("/about", about)`和`m.HandleFunc("/about/", about)`。

运行`serveMux.go`将生成以下输出：

```go
$ go run serveMux.go Served: / for localhost:8001 Served: /123 for localhost:8001
Served: localhost:8001
Served: /cv for localhost:8001
```

以下截图显示了`serveMux.go`在 Web 浏览器上生成的各种输出类型：请注意，浏览器输出与`go run serveMux.go`命令之前的输出无关：

![](img/67d1c2e9-8fd4-4134-8ae7-f992fdb8d0d7.png)

使用 serveMux.go

如果你使用`wget(1)`而不是 Web 浏览器，你将得到以下输出：

```go
$ wget -qO- http://localhost:8001/CV
This is the /CV page at /CV
$ wget -qO- http://localhost:8001/cv
Unknown page: /cv from localhost:8001
$ wget -qO- http://localhost:8001/time
<h1 align="center">The current time is:</h1><h2 align="center">Mon, 10 Jul 2017 13:13:27 EEST</h2>
$ wget -qO- http://localhost:8001/time/
Unknown page: /time/ from localhost:8001
```

因此，`http.HandleFunc()`是库中默认调用的函数，将用于首次实现，而`http.NewServeMux()`的`HandleFunc()`函数则用于其他情况。简单来说，除了在最简单的情况下，最好使用`http.NewServeMux()`版本而不是默认版本。

# html/template 包

**模板**主要用于分离输出的格式和数据部分。请注意，Go 模板可以是文件或字符串：一般的想法是对较小的模板使用字符串，对较大的模板使用文件。

在本节中，我们将通过一个示例来讨论`html/template`包，该示例可以在`template.go`文件中找到，并将分为六部分呈现。`template.go`背后的一般思想是，你正在读取一个包含你想要以 HTML 格式呈现的记录的文本文件。鉴于包的名称是`html/template`，程序的更好名称应该是`genHTML.go`或`genTemplate.go`。

还有`text/template`包，更适用于创建纯文本输出。但是，你不能在同一个 Go 程序中导入`text/template`和`html/template`，除非采取一些额外的步骤来消除歧义，因为这两个包具有相同的包名（`template`）。这两个包之间的关键区别在于，`html/template`对 HTML 注入的数据进行了消毒处理，这意味着它更安全。

源文件的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "html/template" 
   "net/http" 
   "os" 
   "strings" 
) 

type Entry struct { 
   WebSite string 
   WebName string 
   Quality string 
} 

var filename string 
```

结构的定义非常重要，因为这是数据传递到`template`文件的方式。

`template.go`的第二部分包含以下 Go 代码：

```go
func dynamicContent(w http.ResponseWriter, r *http.Request) { 
   var Data []Entry 
   var f *os.File 
   if filename == "" { 
         f = os.Stdin 
   } else { 
         fileHandler, err := os.Open(filename) 
         if err != nil { 
               fmt.Printf("error opening %s: %s", filename, err) 
               os.Exit(1) 
         } 
         f = fileHandler 
   } 
   defer f.Close() 
   scanner := bufio.NewScanner(f) 
   myT := template.Must(template.ParseGlob("template.gohtml")) 
```

`template.ParseGlob()`函数用于读取外部模板文件，它可以有任何你想要的文件扩展名。在项目中查找 Go 模板文件时，使用`.gohtml`扩展名可能会让你的生活更简单。

尽管我个人更喜欢使用`.gohtml`扩展名来命名 Go 模板文件，但`.tpl`是一个非常常见的扩展名，被广泛使用。你可以选择你喜欢的任何一个。

`template.go`的第三部分代码如下：

```go
       for scanner.Scan() { 

         parts := strings.Fields(scanner.Text()) 
         if len(parts) == 3 { 
               temp := Entry{WebSite: parts[0], WebName: parts[1], Quality: parts[2]} 
               Data = append(Data, temp) 
         } 
   } 

   fmt.Println("Serving", r.Host, "for", r.URL.Path) 
   myT.ExecuteTemplate(w, "template.gohtml", Data) 
} 
```

`ExecuteTemplate()`函数的第三个参数是你要处理的数据。在这种情况下，你将一个记录的切片传递给它。

程序的第四部分如下：

```go
func staticPage(w http.ResponseWriter, r *http.Request) { 
   fmt.Println("Serving", r.Host, "for", r.URL.Path) 
   myT := template.Must(template.ParseGlob("static.gohtml")) 
   myT.ExecuteTemplate(w, "static.gohtml", nil) 
} 
```

这个函数显示一个静态的 HTML 页面，我们将通过模板引擎传递`nil`数据，这由`ExecuteTemplate()`函数的第三个参数表示。如果你有相同的函数处理不同的数据片段，可能会出现没有内容可渲染的情况，但保留它是为了保持通用的代码结构。

`template.go`的第五部分包含以下 Go 代码：

```go
func main() { 
   arguments := os.Args 

   if len(arguments) == 1 { 
         filename = "" 
   } else { 
         filename = arguments[1] 
   } 
```

`template.go`中的最后一部分 Go 代码是你定义支持的路径并使用端口号`8001`启动 Web 服务器的地方：

```go
   http.HandleFunc("/static", staticPage) 
   http.HandleFunc("/dynamic", dynamicContent) 
   http.ListenAndServe(":8001", nil) 
} 
```

`template.gohtml`文件的内容如下：

```go
<!doctype html> 
<htmllang="en"> 
<head> 
   <meta charset="UTF-8"> 
   <title>Using Go HTML Templates</title> 
   <style> 
         html { 
               font-size: 16px; 
         } 
         table, th, td { 
         border: 3px solid gray; 
         } 
   </style> 
</head> 
<body> 

<h2 alight="center">Presenting Dynamic content!</h2> 

<table> 
   <thead> 
         <tr> 
               <th>Web Site</th> 
               <th>Quality</th> 
         </tr> 
   </thead> 
   <tbody> 
{{ range . }} 
<tr> 
   <td><a href="{{ .WebSite }}">{{ .WebName }}</a></td> 
   <td> {{ .Quality }} </td> 
</tr> 
{{ end }} 
   </tbody> 
</table> 

</body> 
</html> 
```

句点（`.`）字符代表当前正在处理的数据：简单来说，句点（`.`）字符是一个变量。`{{ range . }}`语句相当于一个`for`循环，遍历输入切片的所有元素，在这种情况下是结构。你可以访问每个结构的字段，如`.WebSite`、`.WebName`和`.Quality`。

`static.gohtml`文件的内容如下：

```go
<!doctype html> 
<htmllang="en"> 
<head> 
   <meta charset="UTF-8"> 
   <title>A Static HTML Template</title> 
</head> 
<body> 

<H1>Hello there!</H1> 

</body> 
</html> 
```

如果你执行`template.go`，你将在屏幕上看到以下输出：

```go
$ go run template.go /tmp/sites.html
Serving localhost:8001 for /dynamic
Serving localhost:8001 for /static
```

以下屏幕截图显示了`template.go`的两个输出，显示在 Web 浏览器上。`sites.html`文件有三列，分别是 URL、名称和质量，可以有多行。好处在于，如果更改`/tmp/sites.html`文件的内容并重新加载网页，您将看到更新后的内容！

![](img/31188a00-51f7-4535-9f6b-2ce30fca63c9.png)

使用 template.go

# 关于 JSON

**JSON**代表 JavaScript 对象表示法。这是一种基于文本的格式，旨在作为在 JavaScript 系统之间传递信息的一种简单轻便的方式。

一个简单的 JSON 文档具有以下格式：

```go
{ "name":"Mihalis", 
"surname":"Tsoukalos",
"country":"Greece" }
```

前面的 JSON 文档有三个字段，分别命名为`name`、`surname`和`country`。每个字段都有一个单一值。

然而，JSON 文档可以具有更复杂的结构，具有多个深度级别。

在看一些代码之前，我认为首先讨论`encoding/json` Go 包将非常有用。`encoding/json`包提供了`Encode()`和`Decode()`函数，允许将 Go 对象转换为 JSON 文档，反之亦然。此外，`encoding/json`包还提供了`Marshal()`和`Unmarshal()`函数，其工作方式类似于`Encode()`和`Decode()`，并且基于`Encode()`和`Decode()`方法。

`Marshal()-Unmarshal()`和`Encode()-Decode()`之间的主要区别在于前者函数适用于单个对象，而后者函数可以处理多个对象以及字节流。

最后，`encoding/json` Go 包包括两个名为`Marshaler`和`Unmarshaler`的接口：它们每个都需要实现一个单一方法，分别命名为`MarshalJSON()`和`UnmarshalJSON()`。这两个接口允许您在 Go 中执行自定义 JSON **编组**和**解组**。不幸的是，这两个接口将不在本书中介绍。

# 保存 JSON 数据

本小节将教您如何将常规数据转换为 JSON 格式，以便通过网络连接发送。本小节的 Go 代码将保存为`writeJSON.go`，并将分为四个部分呈现。

Go 代码的第一部分是程序的预期序文，以及分别命名为`Record`和`Telephone`的两个新`struct`类型的定义：

```go
package main 

import ( 
   "encoding/json" 
   "fmt" 
   "os" 
) 

type Record struct { 
   Name    string 
   Surname string 
   Tel     []Telephone 
} 

type Telephone struct { 
   Mobile bool 
   Number string 
} 
```

请注意，结构的成员只有以大写字母开头的成员才会出现在 JSON 输出中，因为以小写字母开头的成员被视为私有：在这种情况下，`Record`和`Telephone`结构的所有成员都是公共的，并将被导出。

第二部分是定义名为`saveToJSON()`的函数：

```go
funcsaveToJSON(filename string, key interface{}) { 
   out, err := os.Create(filename) 
   if err != nil { 
         fmt.Println(err) 
         return 
   } 

   encodeJSON := json.NewEncoder(out) 
   err = encodeJSON.Encode(key) 
   if err != nil { 
         fmt.Println(err) 
         return 
   } 

   out.Close() 
} 
```

`saveToJSON()`函数为我们完成所有工作，因为它创建了一个名为`encodeJSON`的 JSON 编码器变量，它与文件名相关联，数据将保存在那里。然后，调用`Encode()`将记录的数据保存到相关的文件名，我们就完成了！正如您将在下一节中看到的那样，类似的过程将帮助您读取 JSON 文件并将其转换为 Go 变量。

程序的第三部分具有以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a filename!") 
         os.Exit(100) 
   } 

   filename := arguments[1] 
```

这里没有什么特别的：您只需获取程序的第一个命令行参数。

该实用程序的最后一部分如下：

```go
   myRecord := Record{ 
         Name:    "Mihalis", 
         Surname: "Tsoukalos", 
         Tel: []Telephone{Telephone{Mobile: true, Number: "1234-567"}, 
               Telephone{Mobile: true, Number: "1234-abcd"}, 
               Telephone{Mobile: false, Number: "abcc-567"}, 
         }} 

   saveToJSON(filename, myRecord) 
} 
```

在这里，我们做了两件事。第一件事是定义一个新的`Record`变量并填充它的数据。第二件事是调用`saveToJSON()`将`myRecord`变量以 JSON 格式保存到所选文件中。

执行`writeJSON.go`将生成以下输出：

```go
$ go run writeJSON.go /tmp/SavedFile
```

之后，`/tmp/SavedFile`的内容将如下所示：

```go
$ cat /tmp/SavedFile
{"Name":"Mihalis","Surname":"Tsoukalos","Tel":[{"Mobile":true,"Number":"1234-567"},{"Mobile":true,"Number":"1234-abcd"},{"Mobile":false,"Number":"abcc-567"}]}
```

通过网络发送 JSON 数据需要使用 net Go 标准包，这将在下一章中讨论。

# 解析 JSON 数据

本小节将说明如何读取 JSON 记录并将其转换为一个可以在您自己的程序中使用的 Go 变量。所呈现的程序的名称将是`readJSON.go`，并将分为四个部分呈现给您。

该实用程序的第一部分与`writeJSON.go`实用程序的第一部分相同：

```go
package main 

import ( 
   "encoding/json" 
   "fmt" 
   "os" 
) 

type Record struct { 
   Name    string 
   Surname string 
   Tel     []Telephone 
} 

type Telephone struct { 
   Mobile bool 
   Number string 
} 
```

Go 代码的第二部分如下：

```go
funcloadFromJSON(filename string, key interface{}) error { 
   in, err := os.Open(filename) 
   if err != nil { 
         return err 
   } 

   decodeJSON := json.NewDecoder(in) 
   err = decodeJSON.Decode(key) 
   if err != nil { 
         return err 
   } 
   in.Close() 
   return nil 
} 
```

在这里，您定义了一个名为`loadFromJSON()`的新函数，用于根据作为第二个参数给出的数据结构解码 JSON 文件。您首先调用`json.NewDecoder()`函数创建一个与文件关联的新 JSON 解码变量，然后调用`Decode()`函数来实际解码文件的内容。

`readJSON.go`的第三部分包含以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   iflen(arguments) == 1 { 
         fmt.Println("Please provide a filename!") 
         os.Exit(100) 
   } 

   filename := arguments[1] 
```

程序的最后部分如下：

```go
   var myRecord Record 
   err := loadFromJSON(filename, &myRecord) 
   if err == nil { 
         fmt.Println(myRecord) 
   } else { 
         fmt.Println(err) 
   } 
} 
```

如果运行`readJSON.go`，将得到以下输出：

```go
$ go run readJSON.go /tmp/SavedFile
{Mihalis Tsoukalos [{true 1234-567} {true 1234-abcd} {false abcc-567}]}
```

从网络读取 JSON 数据将在下一章讨论，因为 JSON 记录在网络上传输时与任何其他类型的数据没有区别。

# 使用 Marshal()和 Unmarshal()

在本小节中，您将看到如何使用`Marshal()`和`Unmarshal()`来实现`readJSON.go`和`writeJSON.go`的功能。展示`Marshal()`和`Unmarshal()`函数的 Go 代码可以在`marUnmar.go`中找到，并将分为四部分呈现。

`marUnmar.go`的第一部分是预期的序言：

```go
package main 

import ( 
   "encoding/json" 
   "fmt" 
   "os" 
) 

type Record struct { 
   Name    string 
   Surname string 
   Tel     []Telephone 
} 

type Telephone struct { 
   Mobile bool 
   Number string 
} 
```

程序的第二部分包含以下 Go 代码：

```go
func main() { 
   myRecord := Record{ 
         Name:    "Mihalis", 
         Surname: "Tsoukalos", 
         Tel: []Telephone{Telephone{Mobile: true, Number: "1234-567"}, 
               Telephone{Mobile: true, Number: "1234-abcd"}, 
               Telephone{Mobile: false, Number: "abcc-567"}, 
         }} 
```

这是在`writeJSON.go`程序中使用的相同记录。因此，到目前为止没有什么特别的。

`marUnmar.go`的第三部分是编组发生的地方：

```go
   rec, err := json.Marshal(&myRecord) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
   fmt.Println(string(rec)) 
```

请注意，`json.Marshal()`需要一个指针来传递数据，即使值是一个 map、数组或切片。

程序的最后部分包含以下执行解组操作的 Go 代码：

```go
   var unRec Record 
   err1 := json.Unmarshal(rec, &unRec) 
   if err1 != nil { 
         fmt.Println(err1) 
         os.Exit(100) 
   } 
   fmt.Println(unRec) 
} 
```

从代码中可以看出，`json.Unmarshal()`需要使用指针来保存数据，即使值是一个 map、数组或切片。

执行`marUnmar.go`将创建以下输出：

```go
$ go run marUnmar.go
{"Name":"Mihalis","Surname":"Tsoukalos","Tel":[{"Mobile":true,"Number":"1234-567"},{"Mobile":true,"Number":"1234-abcd"},{"Mobile":false,"Number":"abcc-567"}]}
{Mihalis Tsoukalos [{true 1234-567} {true 1234-abcd} {false abcc-567}]}
```

如您所见，`Marshal()`和`Unmarshal()`函数无法帮助您将数据存储到文件中：您需要自己实现。

# 使用 MongoDB

关系数据库是严格组织成表的结构化数据的集合。查询数据库的主要语言是 SQL。NoSQL 数据库，如**MongoDB**，不使用 SQL，而是使用各种其他查询语言，并且在其表中没有严格的结构，这在 NoSQL 术语中称为**集合**。

您可以根据其数据模型将 NoSQL 数据库分类为文档、键值、图形和列族。MongoDB 是最流行的面向文档的 NoSQL 数据库，适用于 Web 应用程序。

文档数据库并不是用来处理 Microsoft Word 文档的，而是用来存储半结构化数据的。

# 基本的 MongoDB 管理

如果您想在 Go 应用程序中使用 MongoDB，了解如何在 MongoDB 数据库上执行一些基本的管理任务将非常实用。

本节中介绍的大多数任务将从 Mongo shell 执行，该 shell 通过执行`mongo`命令启动。如果您的 Unix 机器上没有运行 MongoDB 实例，将得到以下输出：

```go
$ mongo
MongoDB shell version v3.4.5
connecting to: mongodb://127.0.0.1:27017
2017-07-06T19:37:38.291+0300 W NETWORK  [thread1] Failed to connect to 127.0.0.1:27017, in(checking socket for error after poll), reason: Connection refused
2017-07-06T19:37:38.291+0300 E QUERY    [thread1] Error: couldn't connect to server 127.0.0.1:27017, connection attempt failed :
connect@src/mongo/shell/mongo.js:237:13
@(connect):1:6
exception: connect failed
```

前面的输出告诉我们两件事：

+   MongoDB 服务器进程的默认 TCP 端口号为`27017`

+   mongo 可执行文件尝试连接到`127.0.0.1` IP 地址，这是本地机器的 IP 地址

为了执行以下命令，您应该在本地机器上启动一个 MongoDB 服务器实例。一旦 MongoDB 服务器进程启动并运行，执行`mongo`将创建以下输出：

```go
$ mongo
MongoDB shell version: 2.4.10
connecting to: test
>
```

以下命令将向您展示如何创建一个新的 MongoDB 数据库和一个新的 MongoDB 集合，以及如何向该集合插入一些文档：

```go
>use go;
switched to db go
>db.someData.insert({x:0, y:1})
>db.someData.insert({x:1, y:2})
>db.someData.insert({x:2, y:3})
>db.someData.count()
3
```

一旦您尝试使用`db.someData.insert()`将文档插入到集合中，如果该集合（`someData`）不存在，它将被自动创建。最后一个命令计算了当前数据库的`someData`集合中存储的记录数。

MongoDB 不会通知您可能存在的任何拼写错误。简单地说，如果您错误地输入了数据库或集合的名称，MongoDB 将在您试图找出问题所在时创建一个全新的数据库或新集合！此外，如果您在文档中放入更多、更少或不同的字段并尝试保存它，MongoDB 也不会抱怨！

您可以使用`find()`函数找到集合的记录：

```go
>db.someData.find()
{ "_id" : ObjectId("595e84cd63883cb3fe7f42f3"), "x" : 0, "y" : 1 }
{ "_id" : ObjectId("595e84d263883cb3fe7f42f4"), "x" : 1, "y" : 2 }
{ "_id" : ObjectId("595e84d663883cb3fe7f42f5"), "x" : 2, "y" : 3 }
```

您可以按如下方式找到运行中的 MongoDB 实例上的数据库列表：

```go
>show databases;
LXF   0.203125GB
go    0.0625GB
local 0.078125GB
```

类似地，您可以按如下方式找到当前 MongoDB 数据库中存储的集合的名称：

```go
>db.getCollectionNames()
[ "someData", "system.indexes" ]
```

您可以按如下方式删除 MongoDB 集合的所有记录：

```go
>db.someData.remove()
>show collections
someData
system.indexes
```

最后，您可以按如下方式删除整个集合，包括其中的记录：

```go
>db.someData.drop()
true
>show collections
system.indexes
```

上述信息暂时可以帮助您入门，但如果您想了解更多关于 MongoDB 的信息，您应该访问 MongoDB 的文档网站[`docs.mongodb.com/`](https://docs.mongodb.com/)。

# 使用 MongoDB Go 驱动程序

为了在您的 Go 程序中使用 MongoDB，您应该首先在您的 Unix 机器上安装 MongoDB Go 驱动程序。MongoDB Go 驱动程序的名称是`mgo`，您可以通过访问[`github.com/go-mgo/mgo`](https://github.com/go-mgo/mgo)、[`labix.org/mgo`](https://labix.org/mgo)和[`docs.mongodb.com/ecosystem/drivers/go/`](https://docs.mongodb.com/ecosystem/drivers/go/)了解更多关于 MongoDB Go 驱动程序的信息。

由于驱动程序不是 Go 标准库的一部分，您应该首先使用以下两个命令下载所需的软件包：

```go
$ go get labix.org/v2/mgo
$ go get labix.org/v2/mgo/bson
```

之后，您将可以在自己的 Go 实用程序中使用它。如果您尝试在 Unix 系统上执行该程序而没有这两个软件包，您将收到类似以下的错误消息：

```go
$ go run testMongo.go
testMongo.go:5:2: cannot find package "labix.org/v2/mgo" in any of:
      /usr/local/Cellar/go/1.8.3/libexec/src/labix.org/v2/mgo (from $GOROOT)
      /Users/mtsouk/go/src/labix.org/v2/mgo (from $GOPATH)
testMongo.go:6:2: cannot find package "labix.org/v2/mgo/bson" in any of:
      /usr/local/Cellar/go/1.8.3/libexec/src/labix.org/v2/mgo/bson (from $GOROOT)
      /Users/mtsouk/go/src/labix.org/v2/mgo/bson (from $GOPATH)
```

请注意，您可能需要在您的 Unix 系统上安装 Bazaar 才能执行这两个`go get`命令。您可以在[`bazaar.canonical.com/`](https://bazaar.canonical.com/)获取有关 Bazaar 版本控制系统的更多信息。

因此，您应该首先尝试运行一个简单的 Go 程序，该程序连接到 MongoDB 数据库，创建一个新的数据库和一个新的集合，并向其中添加新的文档，以确保一切都按预期工作：程序的名称将是`testMongo.go`，并将分为四个部分呈现。

程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "labix.org/v2/mgo" 
   "labix.org/v2/mgo/bson" 
   "os" 
   "time" 
) 

type Record struct { 
   Xvalueint 
   Yvalueint 
} 
```

在这里，您可以看到在导入块中使用了 Go MongoDB 驱动程序。此外，您还可以看到定义了一个名为`Record`的新 Go 结构，它将保存每个 MongoDB 文档的数据。

`testMongo.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   mongoDBDialInfo := &mgo.DialInfo{ 
         Addrs:   []string{"127.0.0.1:27017"}, 
         Timeout: 20 * time.Second, 
   } 

   session, err := mgo.DialWithInfo(mongoDBDialInfo) 
   if err != nil { 
         fmt.Printf("DialWithInfo: %s\n", err) 
         os.Exit(100) 
   } 
   session.SetMode(mgo.Monotonic, true) 

   collection := session.DB("goDriver").C("someData") 
```

现在，`collection`变量将用于处理`goDriver`数据库的`someData`集合：数据库的更好名称应该是`myDB`。请注意，在运行 Go 程序之前，MongoDB 实例中没有`goDriver`数据库；这也意味着`someData`集合也不存在。

程序的第三部分如下：

```go
   err = collection.Insert(&Record{1, 0}) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   err = collection.Insert(&Record{-1, 0}) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

在这里，您可以使用`Insert()`函数将两个文档插入到 MongoDB 数据库中。

`testMongo.go`的最后一部分包含以下 Go 代码：

```go
   var recs []Record 
   err = collection.Find(bson.M{"yvalue": 0}).All(&recs) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   for x, y := range recs { 
         fmt.Println(x, y) 
   } 
   fmt.Println("Found:", len(recs), "results!") 
} 
```

由于您不知道从`Find()`查询中会得到多少文档，因此您应该使用记录的切片来存储它们。

另外，请注意，当您存储时，您应该在`Find()`函数中将`yvalue`字段小写，因为 MongoDB 在存储时会自动将`Record`结构的字段转换为小写！

现在，按照这里所示执行`testMongo.go`：

```go
$ go run testMongo.go
0 {1 0}
1 {-1 0}
Found: 2 results!
```

请注意，如果多次执行`testMongo.go`，您会发现相同的文档多次插入到`someData`集合中。但是，MongoDB 不会有任何问题区分所有这些文档，因为每个文档的键是`_id`字段，这是由 MongoDB 自动插入的，每次您向集合插入新文档时都会插入。

之后，使用`MongoDB` shell 命令连接到 MongoDB 实例，以确保一切按预期工作：

```go
$ mongo
MongoDB shell version v3.4.5
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.4.5
>use goDriver
switched to db goDriver
>show collections
someData
>db.someData.find()
{ "_id" : ObjectId("595f88593fb7048f4846e555"), "xvalue" : 1, "yvalue" : 0 }
{ "_id" : ObjectId("595f88593fb7048f4846e557"), "xvalue" : -1, "yvalue" : 0 }
>
```

在这里，重要的是要理解 MongoDB 文档以 JSON 格式呈现，这是您已经知道如何在 Go 中处理的。

另外，请注意，Go MongoDB 驱动程序具有比此处介绍的更多功能。不幸的是，更多讨论超出了本书的范围，但您可以通过访问[`github.com/go-mgo/mgo`](https://github.com/go-mgo/mgo)，[`labix.org/mgo`](https://labix.org/mgo)和[`docs.mongodb.com/ecosystem/drivers/go/`](https://docs.mongodb.com/ecosystem/drivers/go/)来了解更多信息。

# 创建一个显示 MongoDB 数据的 Go 应用程序

实用程序的名称将是`showMongo.go`，它将分为三部分呈现。该实用程序将连接到 MongoDB 实例，读取一个集合，并将集合的文档显示为网页。请注意，`showMongo.go`基于`template.go`的 Go 代码。

Web 应用程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "html/template" 
   "labix.org/v2/mgo" 
   "net/http" 
   "os" 
   "time" 
) 

var DatabaseName string 
var collectionName string 

type Document struct { 
   P1 int 
   P2 int 
   P3 int 
   P4 int 
   P5 int 
}

```

您应该提前了解要检索的 MongoDB 文档的结构，因为字段名称在`struct`类型中是硬编码的，并且需要匹配。

程序的第二部分如下：

```go
func content(w http.ResponseWriter, r *http.Request) { 
   var Data []Document 
   myT := template.Must(template.ParseGlob("mongoDB.gohtml")) 

   mongoDBDialInfo := &mgo.DialInfo{ 
         Addrs:   []string{"127.0.0.1:27017"}, 
         Timeout: 20 * time.Second, 
   } 

   session, err := mgo.DialWithInfo(mongoDBDialInfo) 
   if err != nil { 
         fmt.Printf("DialWithInfo: %s\n", err) 
         return 
   } 
   session.SetMode(mgo.Monotonic, true) 
   c := session.DB(DatabaseName).C(collectionName) 

   err = c.Find(nil).All(&Data) 
   if err != nil { 
         fmt.Println(err) 
         return 
   } 

   fmt.Println("Found:", len(Data), "results!") 
   myT.ExecuteTemplate(w, "mongoDB.gohtml", Data) 
} 
```

与以前一样，使用在`mgo.DialInfo`结构中定义的参数，使用`mgo.DialWithInfo()`连接到 MongoDB。

Web 应用程序的最后部分如下：

```go
func main() { 
   arguments := os.Args 

   iflen(arguments) <= 2 { 
         fmt.Println("Please provide a Database and a Collection!") 
         os.Exit(100) 
   } else { 
         DatabaseName = arguments[1] 
         collectionName = arguments[2] 
   } 

   http.HandleFunc("/", content) 
   http.ListenAndServe(":8001", nil) 
} 
```

`MongoDB.gohtml`的内容与`template.gohtml`的内容类似，这里不会呈现。您可以参考*html/template 包*部分了解`template.gohtml`的内容。

执行`showMongo.go`不会在屏幕上显示实际数据：您需要使用 Web 浏览器进行查看：

```go
$ go run showMongo.go goDriver Numbers
Found: 0 results!
Found: 10 results!
Found: 14 results!
```

好处是，如果集合的数据发生了变化，您无需重新编译 Go 代码即可查看更改：您只需要重新加载网页。

以下屏幕截图显示了在 Web 浏览器上显示的`showMongo.go`的输出：

![](img/a11f9a8c-3fc3-414b-83a7-ecffd7aca8de.png)

使用 showMongo.go

请注意，`Numbers`集合包含以下文档：

```go
>db.Numbers.findOne() 
{ 
      "_id" : ObjectId("596530aeaab5252f5c1ab100"),
      "p1" : -10,
      "p2" : -20,
      "p3" : 100,
      "p4" : -1000,
      "p5" : 10000
}
```

请记住，MongoDB 结构中的额外数据，如果在 Go 结构中没有相应的字段，则会被忽略。

# 创建一个显示 MySQL 数据的应用程序

在本小节中，我们将介绍一个在 MySQL 表上执行查询的 Go 实用程序。新的命令行实用程序的名称将是`showMySQL.go`，将分为五部分呈现。

请注意，`showMySQL.go`将使用`database/sql`包，该包为查询 MySQL 数据库提供了通用的 SQL 接口。

所提供的实用程序需要两个参数：具有管理权限的用户名及其密码。

`showMySQL.go`的第一部分如下：

```go
package main 

import ( 
   "database/sql"  
   "fmt" 
   _ "github.com/go-sql-driver/mysql" 
   "os" 
   "text/template" 
)

```

这里有一个小变化，因为`showMySQL.go`使用`text/template`而不是`html/template`。请注意，符合`database/sql`接口的驱动程序在代码中实际上从未直接引用，但它们仍然需要被初始化和导入。通过在`"github.com/go-sql-driver/mysql"`前面加上`_`字符，Go 会忽略`"github.com/go-sql-driver/mysql"`包实际上未在代码中使用的事实。

您还需要下载 MySQL Go 驱动程序：

```go
$ go get github.com/go-sql-driver/mysql
```

实用程序的第二部分包含以下 Go 代码：

```go
func main() { 
   var username string 
   var password string 

   arguments := os.Args 
   if len(arguments) == 3 { 
         username = arguments[1] 
         password = arguments[2] 
   } else { 
         fmt.Println("programName Username Password!") 
         os.Exit(100) 
   } 
```

来自`showMySQL.go`的第三个 Go 代码块如下：

```go
   connectString := username + ":" + password + "@unix(/tmp/mysql.sock)/information_schema" 
   db, err := sql.Open("mysql", connectString) 

   rows, err := db.Query("SELECT DISTINCT(TABLE_SCHEMA) FROM TABLES;") 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

在这里，您手动构建了到 MySQL 的连接字符串。出于安全原因，默认的 MySQL 安装使用套接字(`/tmp/mysql.sock`)而不是网络连接。将使用的数据库名称是连接字符串的最后一部分(`information_schema`)。

您很可能需要调整这些参数以适应自己的数据库。

`showMySQL.go`的第四部分如下：

```go
   var DATABASES []string 
   for rows.Next() { 
         var databaseName string 
         err := rows.Scan(&databaseName) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
         DATABASES = append(DATABASES, databaseName) 
   } 
   db.Close()

```

`Next()`函数遍历从`select`查询返回的所有记录，并借助`for`循环逐个返回它们。

程序的最后一部分如下：

```go
   t := template.Must(template.New("t1").Parse(` 
   {{range $k := .}} {{ printf "\tDatabase Name: %s" $k}} 
   {{end}} 
   `)) 
   t.Execute(os.Stdout, DATABASES) 
   fmt.Println() 
} 
```

这一次，您将以纯文本形式接收数据，而不是以网页形式呈现数据。此外，由于文本模板很小，因此可以使用`t`变量的帮助在一行中定义它。

这里是否需要使用模板？当然不需要！但是学习如何定义 Go 模板而不使用外部模板文件是很好的。

因此，`showMySQL.go`的输出将类似于以下内容：

```go
$ go run showMySQL.go root 12345

    Database Name: information_schema
    Database Name: mysql
    Database Name: performance_schema
    Database Name: sys
```

前面的输出显示了当前 MySQL 实例的可用数据库信息，这是一种在不使用 MySQL 客户端连接的情况下获取 MySQL 数据库信息的好方法。

# 一个方便的命令行实用程序

在本节中，我们将开发一个方便的命令行实用程序，该实用程序读取一些网页，这些网页可以在文本文件中找到或从标准输入中读取，并返回在这些网页中找到给定关键字的次数。为了更快，该实用程序将使用 goroutines 来获取所需的数据，并使用监控进程来收集数据并在屏幕上呈现。该实用程序的名称将是`findKeyword.go`，并将分为五个部分进行介绍。

实用程序的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "net/http" 
   "net/url" 
   "os" 
   "regexp" 
) 

type Data struct { 
   URL     string 
   Keyword string 
   Times   int 
   Error   error 
} 
```

`Data struct`类型将用于在通道之间传递信息。

`findKeyword.go`的第二部分包含以下 Go 代码：

```go
func monitor(values <-chan Data, count int) { 
   fori := 0; i< count; i++ { 
         x := <-values 
         if x.Error == nil { 
               fmt.Printf("\t%s\t", x.Keyword) 
               fmt.Printf("\t%d\t in\t%s\n", x.Times, x.URL) 
         } else { 
               fmt.Printf("\t%s\n", x.Error) 
         } 
   } 
} 
```

`monitor()`函数是收集和在屏幕上打印所有信息的地方。

第三部分如下：

```go
func processPage(myUrl, keyword string, out chan<- Data) { 
   var err error 
   times := 0 

   URL, err :=url.Parse(myUrl) 
   if err != nil { 
         out<- Data{URL: myUrl, Keyword: keyword, Times: 0, Error: err} 
         return 
   } 

   c := &http.Client{} 
   request, err := http.NewRequest("GET", URL.String(), nil) 
   if err != nil { 
         out<- Data{URL: myUrl, Keyword: keyword, Times: 0, Error: err} 
         return 
   } 

   httpData, err := c.Do(request) 
   if err != nil { 
         out<- Data{URL: myUrl, Keyword: keyword, Times: 0, Error: err} 
         return 
   } 

   bodyHTML := ""

   var buffer [1024]byte 
   reader := httpData.Body 
   for { 
         n, err := reader.Read(buffer[0:]) 
         if err != nil { 
               break 
         } 
         bodyHTML = bodyHTML + string(buffer[0:n]) 
   } 

   regExpr := keyword

   r := regexp.MustCompile(regExpr) 
   matches := r.FindAllString(bodyHTML, -1) 
   times = times + len(matches) 

   newValue := Data{URL: myUrl, Keyword: keyword, Times: times, Error: nil} 
   out<- newValue 
} 
```

在这里，您可以看到`processPage()`函数的实现，该函数在 goroutine 中执行。如果`Data`结构的`Error`字段不是`nil`，则表示出现了错误。

使用`bodyHTML`变量保存 URL 的整个内容是为了避免关键字在两次连续调用`reader.Read()`之间被分割。之后，使用正则表达式(`r`)在`bodyHTML`变量中搜索所需的关键字。

第四部分包含以下 Go 代码：

```go
func main() { 
   filename := "" 
   var f *os.File 
   var keyword string 

   arguments := os.Args 
   iflen(arguments) == 1 { 
         fmt.Println("Not enough arguments!") 
         os.Exit(-1) 
   } 

   iflen(arguments) == 2 { 
         f = os.Stdin 
         keyword = arguments[1] 
   } else { 
         keyword = arguments[1] 
         filename = arguments[2] 
         fileHandler, err := os.Open(filename) 
         if err != nil { 
               fmt.Printf("error opening %s: %s", filename, err) 
               os.Exit(1) 
         } 
         f = fileHandler 
   } 

   deferf.Close() 
```

正如您所看到的，`findKeyword.go` 期望从文本文件或标准输入中获取输入，这是常见的 Unix 做法：这种技术最早在第八章中的*进程和信号*部分进行了说明。

`findKeyword.go`的最后一部分 Go 代码如下：

```go
   values := make(chan Data, len(os.Args[1:])) 

   scanner := bufio.NewScanner(f) 
   count := 0 
   forscanner.Scan() { 
         count = count + 1 
         gofunc(URL string) { 
               processPage(URL, keyword, values) 
         }(scanner.Text()) 
   } 

   monitor(values, count) 
} 
```

这里没有什么特别的：您只需启动所需的 goroutines 和`monitor()`函数来管理它们。

执行`findKeyword.go`将创建以下输出：

```go
$ go run findKeyword.go Tsoukalos /tmp/sites.html
  Get http://really.doesnotexist.com: dial tcp: lookup really.doesnotexist.com: no such host
  Tsoukalos         8      in   http://www.highiso.net/
  Tsoukalos         4      in   http://www.mtsoukalos.eu/
  Tsoukalos         3      in   https://www.packtpub.com/networking-and-servers/go-systems-programming
  Tsoukalos         0      in   http://cnn.com/
  Tsoukalos         0      in   http://doesnotexist.com
```

有趣的是，`doesnotexist.com`域实际上是存在的！

# 练习

1.  在您的 Unix 机器上下载并安装 MongoDB。

1.  访问`net/http` Go 标准包的文档页面，网址为[`golang.org/pkg/net/http/`](https://golang.org/pkg/net/http/)。

1.  访问`html/template` Go 标准包的文档页面，网址为[`golang.org/pkg/html/template/`](https://golang.org/pkg/html/template/)。

1.  更改`getURL.go`的 Go 代码，以使其能够获取多个网页。

1.  阅读`encoding/json`包的文档，网址为[`golang.org/pkg/encoding/json/`](https://golang.org/pkg/encoding/json/)。

1.  访问 MongoDB 网站，网址为[`www.mongodb.org/`](https://www.mongodb.org/)。

1.  通过开发自己的示例来学习如何使用`text/template`。

1.  修改`findKeyword.go`的 Go 代码，以便能够搜索多个关键字。

# 总结

在本章中，我们讨论了 Go 中的 Web 开发，包括解析、编组和解组 JSON 数据，与 MongoDB 数据库交互；从 MySQL 数据库读取数据；在 Go 中创建 Web 服务器；在 Go 中创建 Web 客户端；以及使用`http.ServeMux`类型。

在下一章中，我们将讨论 Go 中的网络编程，其中包括使用低级命令创建 TCP 和 UDP 客户端和服务器。我们还将教你如何在 Go 中开发 RCP 客户端和 RCP 服务器。如果你喜欢开发 TCP/IP 应用程序，那么本书的最后一章就是为你准备的！
