# 第三章：网络爬取礼仪

在深入了解太多代码之前，你需要记住一些要点，当你开始运行网络爬虫时。重要的是要记住，为了让每个人都和睦相处，我们都必须成为互联网的良好公民。牢记这一点，有许多工具和最佳实践可供遵循，以确保在向外部网络服务器添加负载时，你是公平和尊重的。违反这些准则可能会使你的网络爬虫面临被网络服务器屏蔽的风险，或者在极端情况下，你可能会陷入法律纠纷。

在本章中，我们将涵盖以下主题：

+   什么是 robots.txt 文件？

+   什么是用户代理字符串？

+   你如何限制你的网络爬虫？

+   你如何使用缓存？

# 什么是 robots.txt 文件？

大多数网站页面都可以被网络爬虫和机器人访问。允许这样做的原因之一是为了被搜索引擎索引，或者允许内容策展人发现页面。Googlebot 是大多数网站都很乐意让其访问其内容的工具之一。然而，有些网站可能不希望所有内容都出现在谷歌搜索结果中。想象一下，如果你可以谷歌一个人，立即获得他们所有的社交媒体资料，包括联系信息和地址。这对这个人来说是个坏消息，对于托管网站的公司来说也不是一个好的隐私政策。为了控制网站不同部分的访问权限，你需要配置一个`robots.txt`文件。

`robots.txt`文件通常托管在网站的根目录下的`/robots.txt`资源中。这个文件包含了谁可以访问网站中的哪些页面的定义。这是通过描述与`User-Agent`字符串匹配的机器人，并指定允许和不允许的路径来完成的。通配符也支持在`Allow`和`Disallow`语句中。以下是 Twitter 的一个例子`robots.txt`文件：

```go
User-agent: *
Disallow: /
```

这是你可能会遇到的最严格的`robots.txt`文件。它声明没有网络爬虫可以访问[twitter.com](http://twitter.com)的任何部分。违反这一规定将使你的网络爬虫面临被 Twitter 服务器列入黑名单的风险。另一方面，像 Medium 这样的网站则更加宽容。以下是他们的`robots.txt`文件：

```go
User-Agent: *
Disallow: /m/
Disallow: /me/
Disallow: /@me$
Disallow: /@me/
Disallow: /*/edit$
Disallow: /*/*/edit$
Allow: /_/
Allow: /_/api/users/*/meta
Allow: /_/api/users/*/profile/stream
Allow: /_/api/posts/*/responses
Allow: /_/api/posts/*/responsesStream
Allow: /_/api/posts/*/related
Sitemap: https://medium.com/sitemap/sitemap.xml
```

通过查看这些，你可以看到编辑配置文件是被以下指令禁止的：

+   `Disallow: /*/edit$`

+   `Disallow: /*/*/edit$`

与登录和注册相关的页面，这可能被用于自动帐户创建，也被`Disallow: /m/`禁止访问。

如果你重视你的网络爬虫，不要访问这些页面。`Allow`语句明确允许`/_/`路径中的路径，以及一些`api`相关资源。除了这里定义的内容，如果没有明确的`Disallow`语句，那么你的网络爬虫有权限访问这些信息。在 Medium 的情况下，这包括所有公开可用的文章，以及关于作者和出版物的公开信息。这个`robots.txt`文件还包括一个`sitemap`，这是一个列出网站上所有页面的 XML 编码文件。你可以把它想象成一个巨大的索引，非常有用。

`robots.txt`文件的另一个例子显示了一个网站为不同的`User-Agent`实例定义规则。以下`robots.txt`文件来自 Adidas：

```go
User-agent: *
Disallow: /*null*
Disallow: /*Cart-MiniAddProduct
Disallow: /jp/apps/shoplocator*
Disallow: /com/apps/claimfreedom*
Disallow: /us/help-topics-affiliates.html
Disallow: /on/Demandware.store/Sites-adidas-US-Site/en_US/
User-Agent: bingbot
Crawl-delay: 1
Sitemap: https://www.adidas.com/on/demandware.static/-/Sites-CustomerFileStore/default/adidas-US/en_US/sitemaps/adidas-US-sitemap.xml
Sitemap: https://www.adidas.com/on/demandware.static/-/Sites-CustomerFileStore/default/adidas-MLT/en_PT/sitemaps/adidas-MLT-sitemap.xml
```

这个例子明确禁止所有网络爬虫访问一些路径，以及对`bingbot`的特殊说明。`bingbot`必须遵守`Crawl-delay`为`1`秒的规定，这意味着它不能每秒访问超过一次的页面。要注意`Crawl-delays`非常重要，因为它们将定义你可以多快地进行网络请求。违反这一规定可能会为你的网络爬虫产生更多的错误，或者它可能会被永久屏蔽。

# 什么是用户代理字符串？

当 HTTP 客户端向 Web 服务器发出请求时，它们会标识自己的身份。这对于网络爬虫和普通浏览器都是成立的。你是否曾经想过为什么一个网站知道你是 Windows 用户还是 Mac 用户？这些信息包含在你的`User-Agent`字符串中。以下是 Linux 计算机上 Firefox 浏览器的`User-Agent`字符串示例：

```go
Mozilla/5.0 (X11; Linux x86_64; rv:57.0) Gecko/20100101 Firefox/57.0
```

你可以看到这个字符串标识了 Web 浏览器的系列、名称和版本，以及操作系统。这个字符串将随着每个来自该浏览器的请求一起发送到请求头，例如以下内容：

```go
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:57.0) Gecko/20100101 Firefox/57.0
```

并非所有的`User-Agent`字符串都包含这么多信息。非网页浏览器的 HTTP 客户端通常要小得多。以下是一些示例：

+   cURL: `curl/7.47.0`

+   Go: `Go-http-client/1.1`

+   Java: `Apache-HttpClient/4.5.2`

+   Googlebot（用于图像）：`Googlebot-Image/1.0`

`User-Agent`字符串是介绍你的机器人并对遵循`robots.txt`文件中设置的规则负责的一种好方法。通过使用这种机制，你将对任何违规行为负责。

# 示例

有一些开源工具可用于解析`robots.txt`文件并根据其验证网站 URL，以查看你是否有访问权限。我推荐的一个项目在 GitHub 上叫做`temoto`的`robotstxt`。为了下载这个库，在你的终端中运行以下命令：

```go
go get github.com/temoto/robotstxt
```

这里提到的`$GOPATH`是你在 Go 编程语言安装中设置的路径，见第一章，*介绍 Web 爬虫和 Go*。这是带有`src/ bin/`和`pkg/`目录的目录。

这将在你的机器上安装库到`$GOPATH/src/github/temoto/robotstxt`。如果愿意，你可以阅读代码以了解其工作原理。为了本书的目的，我们将只在我们自己的项目中使用该库。在你的`$GOPATH/src`文件夹中，创建一个名为`robotsexample`的新文件夹。在`robotsexample`文件夹中创建一个`main.go`文件。以下是`main.go`的代码，展示了如何使用`temoto/robotstxt`包的简单示例：

```go
package main

import (
  "net/http"

  "github.com/temoto/robotstxt"
)

func main() {
  // Get the contents of robots.txt from packtpub.com
  resp, err := http.Get("https://www.packtpub.com/robots.txt")
  if err != nil {
    panic(err)
  }
  // Process the response using temoto/robotstxt
  data, err := robotstxt.FromResponse(resp)
  if err != nil {
    panic(err)
  }
  // Look for the definition in the robots.txt file that matches the default Go User-Agent string
  grp := data.FindGroup("Go-http-client/1.1")
  if grp != nil {
    testUrls := []string{
      // These paths are all permissable
      "/all",
      "/all?search=Go",
      "/bundles",

      // These paths are not
      "/contact/",
      "/search/",
      "/user/password/",
    }

    for _, url := range testUrls {
      print("checking " + url + "...")

      // Test the path against the User-Agent group
      if grp.Test(url) == true {
        println("OK")
      } else {
        println("X")
      }
    }
  }
}
```

本示例使用 Go 语言的`range`操作符进行每个循环。`range`操作符返回两个变量，第一个是`迭代`的`索引`（我们通过将其分配给`_`来忽略），第二个是该索引处的值。

这段代码对[`www.packtpub.com/`](https://www.packtpub.com/)的`robots.txt`文件检查了六条不同的路径，使用了 Go HTTP 客户端的默认`User-Agent`字符串。如果`User-Agent`被允许访问页面，则`Test()`方法返回`true`。如果返回`false`，则你的爬虫不应该访问网站的这一部分。

# 如何限制你的爬虫速度

良好的网络爬取礼仪的一部分是确保你不会对目标 Web 服务器施加太大的负载。这意味着限制你在一定时间内发出的请求数量。对于较小的服务器，这一点尤为重要，因为它们的资源池更有限。一个很好的经验法则是，你应该只在你认为页面会发生变化的时候访问相同的网页。例如，如果你正在查看每日优惠，你可能只需要每天爬取一次。至于从同一个网站爬取多个页面，你应该首先遵循`robots.txt`文件中的`Crawl-Delay`。如果没有指定`Crawl-Delay`，那么你应该在每个页面后手动延迟一秒钟。

有许多不同的方法可以将延迟纳入你的爬虫中，从手动让程序休眠到使用外部队列和工作线程。本节将解释一些基本技术。当我们讨论 Go 编程语言并发模型时，我们将重新讨论更复杂的示例。

向您的网络爬虫添加节流的最简单方法是跟踪发出的请求的时间戳，并确保经过的时间大于您期望的速率。例如，如果您要以每 5 秒一个页面的速率抓取，它可能是这样的：

```go
package main

import (
  "fmt"
  "net/http"
  "time"
)

func main() {
  // Tracks the timestamp of the last request to the webserver
  var lastRequestTime time.Time

  // The maximum number of requests we will make to the webserver
  maximumNumberOfRequests := 5

  // Our scrape rate at 1 page per 5 seconds
  pageDelay := 5 * time.Second

  for i := 0; i < maximumNumberOfRequests; i++ {
    // Calculate the time difference since our last request
    elapsedTime := time.Now().Sub(lastRequestTime)
    fmt.Printf("Elapsed Time: %.2f (s)\n", elapsedTime.Seconds())
    //Check if there has been enough time
    if elapsedTime < pageDelay {
      // Sleep the difference between the pageDelay and elapsedTime
      var timeDiff time.Duration = pageDelay - elapsedTime
      fmt.Printf("Sleeping for %.2f (s)\n", timeDiff.Seconds())
      time.Sleep(pageDelay - elapsedTime)
    }

    // Just for this example, we are not processing the response
    println("GET example.com/index.html")
    _, err := http.Get("http://www.example.com/index.html")
    if err != nil {
      panic(err)
    }

    // Update the last request time
    lastRequestTime = time.Now()
  }
}
```

此示例在定义变量时有许多`:=`的实例。这是 Go 中同时声明和实例化变量的简写方式。它取代了需要说以下内容：

`var a string`

a = "value"

相反，它变成了：

`a := "value"`

在此示例中，我们每隔五秒向[`www.example.com/index.html`](http://www.example.com/index.html)发出一次请求。我们知道自上次请求以来的时间有多长，因为我们更新`lastRequestTime`变量并在进行每个请求之前检查它。这就是您抓取单个网站所需的全部内容，即使您要抓取多个页面。

如果您要从多个网站抓取数据，您需要将`lastRequestTime`分成每个网站一个变量。最简单的方法是使用`map`，Go 的键值结构，其中键将是主机名，值将是上次请求的时间戳。这将替换定义为以下内容：

```go
var lastRequestMap map[string]time.Time = map[string]time.Time{
  "example.com": time.Time{},
  "packtpub.com": time.Time{},
}
```

我们的`for`循环也会稍微改变，并将地图的值设置为当前抓取时间，但仅适用于我们正在抓取的网站。例如，如果我们要交替抓取页面，可能会是这样的：

```go
// Check if "i" is an even number
if i%2 == 0 {
  // Use the Packt Publishing site and elapsed time
  webpage = packtPage
  elapsedTime = time.Now().Sub(lastRequestMap["packtpub.com"])
} else {
  // Use the example.com elapsed time
  elapsedTime = time.Now().Sub(lastRequestMap["example.com"])
}
```

最后，要更新地图的最后已知请求时间，我们将使用类似的块：

```go
// Update the last request time
if i%2 == 0 {
  // Use the Packt Publishing elapsed time
  lastRequestMap["packtpub.com"] = time.Now()
} else {
  // Use the example.com elapsed time
  lastRequestMap["example.com"] = time.Now()
}
```

您可以在 GitHub 上找到此示例的完整源代码。

如果您查看终端中的输出，您将看到对任一站点的第一个请求没有延迟，每个休眠期略少于五秒。这表明爬虫正在独立地尊重每个站点的速率。

# 如何使用缓存

最后一个可以使您的爬虫受益并减少网站负载的技术是仅在内容更改时请求新内容。如果您的爬虫从 Web 服务器下载相同的旧内容，那么您将不会获得任何新信息，而 Web 服务器则会做不必要的工作。因此，大多数 Web 服务器实施技术以向客户端提供有关缓存的指令。

支持缓存的网站将向客户端提供有关可以存储什么以及存储多长时间的信息。这是通过响应头，如`Cache-Control`，`Etag`，`Date`，`Expires`和`Vary`来完成的。您的网络爬虫应该了解这些指令，以避免向网络服务器发出不必要的请求，从而节省您和服务器的时间和计算资源。让我们再次看看我们的[`www.example.com/index.html`](http://www.example.com/index.html)响应，如下所示：

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
...
```

响应的正文在此示例中未包含。

有一些响应头用于传达缓存指令，您应该遵循这些指令以增加网络爬虫的效率。这些头部将告诉您应缓存什么信息，以及多长时间，以及其他一些有用的信息，以使生活更轻松。

# Cache-Control

`Cache-Control`头用于指示此内容是否可缓存以及缓存多长时间。此头的常见值如下：

+   `no-cache`

+   `no-store`

+   `must-revalidated`

+   `max-age=<seconds>`

+   `public`

缓存指令，如`no-cache`，`no-store`和`must-revalidate`存在是为了防止客户端缓存响应。有时，服务器知道此页面上的内容经常更改，或者依赖于其控制范围之外的来源。如果没有发送这些指令中的任何一个，您应该能够使用提供的`max-age`指令缓存响应。这定义了您应将此内容视为新鲜的秒数。此时间后，响应被认为是陈旧的，并且应向服务器发出新请求。

在上一个示例的响应中，服务器发送了一个`Cache-Control`标头：

```go
Cache-Control: max-age=604800
```

这表示您应该将此页面缓存长达`604880`秒（七天）。

# Expires

`Expires`标头是另一种定义保留缓存信息时间的方法。此标头定义了确切的日期和时间，从该日期和时间开始，内容将被视为过时并应该被刷新。如果提供了`Cache-Control`标头的`max-age`指令，这个时间应该与之相符。

在我们的示例中，`Expires`标头与`Date`标头匹配，根据`Date`标头定义了请求何时被服务器接收，从而定义了 7 天的到期时间：

```go
Date: Mon, 29 Oct 2018 13:31:23 GMT
Expires: Mon, 05 Nov 2018 13:31:23 GMT
```

# Etag

`Etag`也是保持缓存信息的重要内容。这是此页面的唯一密钥，只有在页面内容更改时才会更改。在缓存过期后，您可以使用此标记与服务器检查是否实际上有新内容，而无需下载新副本。这通过发送包含`Etag`值的`If-None-Match`标头来实现。当发生这种情况时，服务器将检查当前资源上的`Etag`是否与`If-None-Match`标头中的`Etag`匹配。如果匹配，则表示没有更新，服务器将以 304 Not Modified 的状态代码响应，并附加一些标头以扩展您的缓存。以下是`304`响应的示例：

```go
HTTP/1.1 304 Not Modified
Accept-Ranges: bytes
Cache-Control: max-age=604800
Date: Fri, 02 Nov 2018 14:37:16 GMT
Etag: "1541025663"
Expires: Fri, 09 Nov 2018 14:37:16 GMT
Last-Modified: Fri, 09 Aug 2013 23:54:35 GMT
Server: ECS (dca/53DB)
Vary: Accept-Encoding
X-Cache: HIT
```

在这种情况下，服务器验证`Etag`并提供一个新的`Expires`时间，仍然与`max-age`匹配，从第二次请求被满足的时间开始。这样，您仍然可以节省时间，而无需通过网络读取更多数据。您仍然可以使用缓存的页面来满足您的需求。

# 在 Go 中缓存内容

缓存页面的存储和检索可以通过手动使用本地文件系统来实现，也可以通过数据库来保存数据和缓存信息。还有一些开源工具可用于简化这种技术。其中一个项目是 GitHub 用户`gregjones`的`httpcache`。

`httpcache`遵循**互联网工程任务组**（**IETF**）制定的缓存要求，这是互联网标准的管理机构。该库提供了一个模块，可以从本地机器存储和检索网页，以及一个用于 Go HTTP 客户端的插件，自动处理所有与缓存相关的 HTTP 请求和响应标头。它还提供多个存储后端，您可以在其中存储缓存信息，如 Redis、Memcached 和 LevelDB。这将允许您在不同的机器上运行网络爬虫，但连接到相同的缓存信息。

随着您的爬虫规模的增长，您需要设计分布式架构，像这样的功能将是至关重要的，以确保时间和资源不会浪费在重复的工作上。所有爬虫之间的稳定通信至关重要！

让我们来看一个使用`httpcache`的示例。首先，通过在终端中输入以下命令来安装`httpcache`，如下所示：

+   `go get github.com/gregjones/httpcache`

+   `go get github.com/peterbourgon/diskv`

`diskv`项目被`httpcache`用于在本地机器上存储网页。

在您的`$GOPATH/src`中，创建一个名为`cache`的文件夹，并在其中创建一个`main.go`。使用以下代码作为您的`main.go`文件：

```go
package main

import (
  "io/ioutil"

  "github.com/gregjones/httpcache"
  "github.com/gregjones/httpcache/diskcache"
)

func main() {
  // Set up the local disk cache
  storage := diskcache.New("./cache")
  cache := httpcache.NewTransport(storage)

  // Set this to true to inform us if the responses are being read from a cache
  cache.MarkCachedResponses = true
  cachedClient := cache.Client()

  // Make the initial request
  println("Caching: http://www.example.com/index.html")
  resp, err := cachedClient.Get("http://www.example.com/index.html")
  if err != nil {
    panic(err)
  }

  // httpcache requires you to read the body in order to cache the response
  ioutil.ReadAll(resp.Body)
  resp.Body.Close()

  // Request index.html again
  println("Requesting: http://www.example.com/index.html")
  resp, err = cachedClient.Get("http://www.example.com/index.html")
  if err != nil {
    panic(err)
  }

  // Look for the flag added by httpcache to show the result is read from the cache
  _, ok = resp.Header["X-From-Cache"]
  if ok {
    println("Result was pulled from the cache!")
  }
}
```

该程序使用本地磁盘缓存来存储来自[`www.example.com/index.html`](http://www.example.com/index.html)的响应。在底层，它读取所有与缓存相关的标头，以确定是否可以存储页面，并将到期日期与数据一起包括在内。在第二次请求时，`httpcache`检查内容是否已过期，并返回缓存的数据，而不是进行另一个 HTTP 请求。它还添加了一个额外的标头`X-From-Cache`，以指示这是从缓存中读取的。如果页面已过期，它将使用`If-None-Match`标头进行 HTTP 请求，并处理响应，包括在 304 Not Modified 响应的情况下更新缓存。

使用自动设置好处理缓存内容的客户端将使您的爬虫运行更快，同时减少您的网络爬虫被标记为不良公民的可能性。当这与尊重网站的`robots.txt`文件和适当节流请求结合使用时，您可以自信地进行爬取，知道自己是网络社区中值得尊敬的成员。

# 总结

在本章中，您学会了尊重地爬取网络的基本礼仪。您了解了什么是`robots.txt`文件，以及遵守它的重要性。您还学会了如何使用`User-Agent`字符串来正确表示自己。还介绍了通过节流和缓存来控制您的爬虫。有了这些技能，您离构建一个完全功能的网络爬虫又近了一步。

在第四章中，*解析 HTML*，我们将学习如何使用各种技术从 HTML 页面中提取信息。
