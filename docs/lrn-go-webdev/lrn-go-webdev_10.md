# 第十章：缓存、代理和性能改进

我们已经涵盖了大量关于 Web 应用程序的内容，您需要连接数据源，渲染模板，利用 SSL/TLS，为单页应用构建 API 等等。

尽管基本原理很清楚，但您可能会发现，根据这些准则构建的应用程序投入生产后可能会迅速出现一些问题，特别是在负载较重的情况下。

在上一章中，我们通过解决 Web 应用程序中一些最常见的安全问题，实施了一些最佳安全实践。让我们在本章中也做同样的事情，通过应用最佳实践来解决一些性能和速度方面的最大问题。

为了做到这一点，我们将查看管道中一些最常见的瓶颈，并看看我们如何减少这些瓶颈，使我们的应用在生产中尽可能高效。

具体来说，我们将确定这些瓶颈，然后寻找反向代理和负载平衡，将缓存实施到我们的应用程序中，利用**SPDY**，以及了解如何使用托管云服务来通过减少到达我们应用程序的请求数来增强我们的速度计划。

通过本章的结束，我们希望能够提供工具，帮助任何 Go 应用程序充分利用我们的环境，发挥最佳性能。

在本章中，我们将涵盖以下主题：

+   识别瓶颈

+   实施反向代理

+   实施缓存策略

+   实施 HTTP/2

# 识别瓶颈

为了简化事情，对于您的应用程序，有两种类型的瓶颈，一种是由开发和编程缺陷引起的，另一种是由底层软件或基础设施限制引起的。

对于前者的答案很简单，找出糟糕的设计并修复它。在糟糕的代码周围打补丁可能会隐藏安全漏洞，或者延迟更大的性能问题被及时发现。

有时，这些问题源于缺乏压力测试；在本地性能良好的代码并不保证在不施加人为负载的情况下能够扩展。缺乏这种测试有时会导致生产中出现意外的停机时间。

然而，忽略糟糕的代码作为问题的根源，让我们来看看一些其他常见的问题：

+   磁盘 I/O

+   数据库访问

+   高内存/CPU 使用率

+   缺乏并发支持

当然还有数百种问题，例如网络问题、某些应用程序中的垃圾收集开销、不压缩有效载荷/标头、非数据库死锁等等。

高内存和 CPU 使用率往往是结果而不是原因，但许多其他原因是特定于某些语言或环境的。

对于我们的应用程序，数据库层可能是一个薄弱点。由于我们没有进行缓存，每个请求都会多次命中数据库。ACID 兼容的数据库（如 MySQL/PostgreSQL）因负载而崩溃而臭名昭著，而对于不那么严格的键/值存储和 NoSQL 解决方案来说，在相同硬件上不会有问题。数据库一致性的成本对此有很大的影响，这是选择传统关系数据库的权衡之一。

# 实施反向代理

正如我们现在所知道的，与许多语言不同，Go 配备了完整和成熟的 Web 服务器平台，其中包括`net/http`。

最近，一些其他语言已经配备了用于本地开发的小型玩具服务器，但它们并不适用于生产。事实上，许多明确警告不要这样做。一些常见的是 Ruby 的 WEBrick，Python 的 SimpleHTTPServer 和 PHP 的-S。其中大多数都存在并发问题，导致它们无法成为生产中的可行选择。

Go 的`net/http`是不同的；默认情况下，它可以轻松处理这些问题。显然，这在很大程度上取决于底层硬件，但在紧要关头，您可以成功地原生使用它。许多网站正在使用`net/http`来提供大量的流量。

但即使是强大的基础 web 服务器也有一些固有的局限性：

+   它们缺乏故障转移或分布式选项

+   它们在上游具有有限的缓存选项

+   它们不能轻易负载平衡传入的流量

+   它们不能轻易集中日志记录

这就是反向代理的作用。反向代理代表一个或多个服务器接受所有传入的流量，并通过应用前述（和其他）选项和优势来分发它。另一个例子是 URL 重写，这更适用于可能没有内置路由和 URL 重写的基础服务。

在你的 web 服务器（如 Go）前放置一个简单的反向代理有两个重要的优势：缓存选项和无需访问基础应用程序即可提供静态内容的能力。

反向代理站点的最受欢迎的选项之一是 Nginx（发音为 Engine-X）。虽然 Nginx 本身是一个 web 服务器，但它早期因轻量级和并发性而广受赞誉。它很快成为了前端应用程序在其他较慢或较重的 web 服务器（如 Apache）前的首要防御。近年来情况有所改变，因为 Apache 在并发选项和利用替代事件和线程的方面已经赶上了。以下是一个反向代理 Nginx 配置的示例：

```go
server {
  listen 80;
  root /var/;
  index index.html index.htm;

  large_client_header_buffers 4 16k;

  # Make site accessible from http://localhost/
  server_name localhost

  location / {
    proxy_pass http://localhost:8080;
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

}
```

在这种情况下，请确保您的 Go 应用程序正在端口`8080`上运行，并重新启动 Nginx。对`http//:port 80`的请求将通过 Nginx 作为反向代理传递到您的应用程序。您可以通过查看标头或在浏览器的**开发人员工具**中进行检查。

![实施反向代理](img/B04294_10_01.jpg)

请记住，我们希望尽可能支持 TLS/SSL，但在这里提供反向代理只是改变端口的问题。我们的应用程序应该在另一个端口上运行，可能是一个附近的端口，以便清晰，然后我们的反向代理将在端口`443`上运行。

提醒一下，任何端口都可以用于 HTTP 或 HTTPS。但是，当未指定端口时，浏览器会自动将其定向到`443`以进行安全连接。只需修改`nginx.conf`和我们应用程序的常量即可。

```go
server {
  listen 443;
  location / {
     proxy_pass http://localhost:444;
```

让我们看看如何修改我们的应用程序，如下面的代码所示：

```go
const (
  DBHost  = "127.0.0.1"
  DBPort  = ":3306"
  DBUser  = "root"
  DBPass  = ""
  DBDbase = "cms"
  PORT    = ":444"
)
```

这使我们能够通过前端代理传递 SSL 请求。

### 提示

在许多 Linux 发行版中，您需要 SUDO 或 root 权限才能使用 1000 以下的端口。

# 实施缓存策略

有许多方法可以决定何时创建和何时过期缓存项，因此我们将看一种更简单更快的方法。但是，如果您有兴趣进一步开发，您可能会考虑其他缓存策略；其中一些可以提供资源使用效率和性能。

## 使用最近最少使用

在分配的资源（磁盘空间、内存）内保持缓存稳定性的一种常见策略是**最近最少使用**（**LRU**）系统用于缓存过期。在这种模型中，利用有关最后缓存访问时间（创建或更新）的信息，缓存管理系统可以移除列表中最老的条目。

这对性能有很多好处。首先，如果我们假设最近创建/更新的缓存条目是当前最受欢迎的条目，我们可以更快地移除那些没有被访问的条目；以便为现有和可能更频繁访问的新资源释放资源。

这是一个公平的假设，假设用于缓存的分配资源并不是微不足道的。如果你有大量的文件缓存或大量的内存用于内存缓存，那么最老的条目，就最后一次访问而言，很可能并没有被频繁地使用。

还有一个相关且更精细的策略叫做最不常用，它严格维护缓存条目本身的使用统计。这不仅消除了对缓存数据的假设，还增加了统计维护的开销。

在这里的演示中，我们将使用 LRU。

## 通过文件缓存

我们的第一个方法可能最好描述为一个经典的缓存方法，但并非没有问题。我们将利用磁盘为各个端点（API 和 Web）创建基于文件的缓存。

那么与在文件系统中缓存相关的问题是什么呢？嗯，在本章中我们提到过，磁盘可能会引入自己的瓶颈。在这里，我们做了一个权衡，以保护对数据库的访问，而不是可能遇到磁盘 I/O 的其他问题。

如果我们的缓存目录变得非常大，这将变得特别复杂。在这一点上，我们将引入更多的文件访问问题。

另一个缺点是我们必须管理我们的缓存；因为文件系统不是短暂的，我们的可用空间是有限的。我们需要手动过期缓存文件。这引入了另一轮维护和另一个故障点。

尽管如此，这仍然是一个有用的练习，如果你愿意承担一些潜在的问题，它仍然可以被利用：

```go
package cache

const (
  Location "/var/cache/"
)

type CacheItem struct {
  TTL int
  Key string
}

func newCache(endpoint string, params ...[]string) {

}

func (c CacheItem) Get() (bool, string) {
  return true, ""
}

func (c CacheItem) Set() bool {

}

func (c CacheItem) Clear() bool {

}
```

这为我们做了一些准备，比如基于端点和查询参数创建唯一的键，检查缓存文件的存在，如果不存在，按照正常情况获取请求的数据。

在我们的应用程序中，我们可以简单地实现这一点。让我们在`/page`端点前面放一个文件缓存层，如下所示：

```go
func ServePage(w http.ResponseWriter, r *http.Request) {
  vars := mux.Vars(r)
  pageGUID := vars["guid"]
  thisPage := Page{}
  cached := cache.newCache("page",pageGUID)
```

前面的代码创建了一个新的`CacheItem`。我们利用可变参数`params`来生成一个引用文件名：

```go
func newCache(endpoint string, params ...[]string) CacheItem {
cacheName := endponit + "_" + strings.Join(params, "_")
c := CacheItem{}
return c
}
```

当我们有一个`CacheItem`对象时，我们可以使用`Get()`方法进行检查，如果缓存仍然有效，它将返回`true`，否则该方法将返回`false`。我们利用文件系统信息来确定缓存项是否在其有效的存活时间内：

```go
  valid, cachedData := cached.Get()
  if valid {
    thisPage.Content = cachedData
    fmt.Fprintln(w, thisPage)
    return
  }
```

如果我们通过`Get()`方法找到一个现有的项目，我们将检查确保它在设置的`TTL`内已经更新：

```go
func (c CacheItem) Get() (bool, string) {

  stats, err := os.Stat(c.Key)
  if err != nil {
    return false, ""
  }

  age := time.Nanoseconds() - stats.ModTime()
  if age <= c.TTL {
    cache, _ := ioutil.ReadFile(c.Key)
    return true, cache
  } else {
    return false, ""
  }
}
```

如果代码有效并且在 TTL 内，我们将返回`true`，并且文件的主体将被更新。否则，我们将允许页面检索和生成的通过。在这之后我们可以设置缓存数据：

```go
  t, _ := template.ParseFiles("templates/blog.html")
  cached.Set(t, thisPage)
  t.Execute(w, thisPage)
```

然后我们将保存这个为：

```go
func (c CacheItem) Set(data []byte) bool {
  err := ioutil.WriteFile(c.Key, data, 0644)
}
```

这个函数有效地写入了我们的缓存文件的值。

我们现在有一个工作系统，它将接受各个端点和无数的查询参数，并创建一个基于文件的缓存库，最终防止了对数据库的不必要查询，如果数据没有改变的话。

在实践中，我们希望将这个限制在大部分基于读的页面上，并避免在任何写或更新端点上盲目地进行缓存，特别是在我们的 API 上。

## 内存中的缓存

正如文件系统缓存因存储价格暴跌而变得更加可接受，我们在 RAM 中也看到了类似的变化，紧随硬存储之后。这里的巨大优势是速度，内存中的缓存因为显而易见的原因可以非常快。

Memcache 及其分布式的兄弟 Memcached，是为了为 LiveJournal 和*Brad Fitzpatrick*的原型社交网络创建一个轻量级和超快的缓存而演变而来。如果这个名字听起来很熟悉，那是因为 Brad 现在在谷歌工作，并且是 Go 语言本身的重要贡献者。

作为我们文件缓存系统的一个替代方案，Memcached 将起到类似的作用。唯一的主要变化是我们的键查找，它将针对工作内存而不是进行文件检查。

### 注意

使用 Go 语言与 memcache 一起使用，访问*Brad Fitz*的网站 [godoc.org/github.com/bradfitz/gomemcache/memcache](http://godoc.org/github.com/bradfitz/gomemcache/memcache)，并使用`go get`命令进行安装。

# 实现 HTTP/2

谷歌在过去五年中投资的更有趣、也许更高尚的举措之一是专注于使网络更快。通过诸如 PageSpeed 之类的工具，谷歌试图推动整个网络变得更快、更精简、更用户友好。

毫无疑问，这项举措并非完全无私。谷歌建立了他们的业务在广泛的网络搜索上，爬虫始终受制于它们爬取的页面速度。网页加载得越快，爬取就越快、更全面；因此，需要的时间和基础设施就越少，所需的资金也就越少。这里的底线是，更快的网络对谷歌有利，就像对创建和查看网站的人一样。

但这是互惠的。如果网站更快地遵守谷歌的偏好，每个人都会从更快的网络中受益。

这将我们带到了 HTTP/2，这是 HTTP 的一个版本，取代了 1999 年引入的 1.1 版本，也是大部分网络的事实标准方法。HTTP/2 还包含并实现了许多 SPDY，这是谷歌开发并通过 Chrome 支持的临时协议。

HTTP/2 和 SPDY 引入了一系列优化，包括头部压缩和非阻塞和多路复用的请求处理。

如果您使用的是 1.6 版本，`net/http`默认支持 HTTP/2。如果您使用的是 1.5 或更早版本，则可以使用实验性包。

### 注意

要在 Go 版本 1.6 之前使用 HTTP/2，请从 [godoc.org/golang.org/x/net/http2](http://godoc.org/golang.org/x/net/http2) 获取。

# 总结

在本章中，我们专注于通过减少对底层应用程序瓶颈的影响来增加应用程序整体性能的快速获胜策略，即我们的数据库。

我们已经在文件级别实施了缓存，并描述了如何将其转化为基于内存的缓存系统。我们研究了 SPDY 和 HTTP/2，它现在已成为默认的 Go `net/http`包的一部分。

这绝不代表我们可能需要生成高性能代码的所有优化，但涉及了一些最常见的瓶颈，这些瓶颈可能会导致在开发中表现良好的应用在生产环境中在重负载下表现不佳。

这就是我们结束这本书的地方；希望大家都享受这段旅程！
