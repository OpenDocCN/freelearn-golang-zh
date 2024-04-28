# 解析 HTML

在之前的章节中，我们处理了整个网页，这对大多数网络爬虫来说并不是很实用。虽然从网页中获取所有内容很好，但大多数情况下，你只需要从每个页面中获取一小部分信息。为了提取这些信息，你必须学会解析网络的标准格式，其中最常见的是 HTML。

本章将涵盖以下主题：

+   HTML 格式是什么

+   使用字符串包进行搜索

+   使用正则表达式包进行搜索

+   使用 XPath 查询进行搜索

+   使用层叠样式表选择器进行搜索

# HTML 格式是什么？

HTML 是用于提供网页上下文的标准格式。HTML 页面定义了浏览器应该绘制哪些元素，元素的内容和样式，以及页面应该如何响应用户的交互。回顾我们的[`example.com/index.html`](http://example.com/index.html)响应，你可以看到以下内容，这就是 HTML 文档的样子：

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
    <p>This domain is established to be used for illustrative examples 
       in documents. You may use this domain in examples without prior
       coordination or asking for permission.</p>
    <p><a href="http://www.iana.org/domains/example">More 
        information...</a></p>
  </div>
</body>
</html>
```

遵循 HTML 规范的文件遵循一套严格的规则，定义了文档的语法和结构。通过学习这些规则，你可以快速轻松地从任何网页中检索任何信息。

# 语法

HTML 文档通过使用带有元素名称的标签来定义网页的元素。标签总是被尖括号包围，比如`<body>`标签。每个元素通过在标签名称之前使用斜杠来定义标签集的结束，比如`</body>`。元素的内容位于一对开放和关闭标签之间。例如，`<body>`和匹配的`</body>`标签之间的所有内容定义了 body 元素的内容。

一些标签还有额外的属性，以键值对的形式定义，称为属性。这些属性用于描述元素的额外信息。在所示的示例中，有一个带有名为`href`的属性的`<a>`标签，其值为[`www.iana.org/domains/example`](https://www.iana.org/domains/example)。在这种情况下，`href`是`<a>`标签的一个属性，并告诉浏览器这个元素链接到提供的 URL。我们将在后面的章节中更深入地了解如何导航这些链接。

# 结构

每个 HTML 文档都有一个特定的布局，从`<!doctype>`标签开始。这个标签用于定义用于验证特定文档的 HTML 规范的版本。在我们的情况下，`<!doctype html>`指的是 HTML 5 规范。有时你可能会看到这样的标签：

```go
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
```

这将描述一个遵循提供的 URL 提供的定义的`HTML 4.01`（严格）网页。我们不会在本书中使用提供的定义来验证页面，因为通常不需要这样做。

在`<!doctype>`标签之后是`<html>`标签，其中包含网页的实际内容。在`<html>`标签内，你会找到文档的`<head>`和`<body>`标签。`<head>`标签包含有关页面本身的元数据，比如标题，以及用于构建网页的外部文件。这些文件可能是用于样式，或者用于描述元素如何对用户交互做出反应。

在实际网页[`example.com/index.html`](http://example.com/index.html)上，你可以看到`<style>`标签用于描述网页上各种类型元素的大小、颜色、字体和间距。为了节省空间，本书中删除了 HTML 文档中的这些信息。

`<body>`标签包含了你感兴趣的大部分数据。在`<body>`元素内，你会找到所有文本、图片、视频和包含你网页抓取需求信息的链接。从网页中收集你需要的数据可以通过许多不同的方式完成；你将在接下来的章节中看到一些常见的方法。

# 使用字符串包进行搜索

搜索内容的最基本方法是使用 Go 标准库中的`strings`包。`strings`包允许您对 String 对象执行各种操作，包括搜索匹配项，计算出现次数以及将字符串拆分为数组。此包的实用性可以涵盖您可能遇到的一些用例。

# 示例-计算链接

我们可以使用`strings`包提取的一条快速且简单的信息是计算网页中包含的链接数量。`strings`包有一个名为`Count()`的函数，它返回字符串中子字符串出现的次数。正如我们之前所见，链接包含在`<a>`标记中。通过计算`"<a"`的出现次数，我们可以大致了解页面中链接的数量。示例如下所示：

```go
package main

import (
  "fmt"
  "io/ioutil"
  "net/http"
  "strings"
)

func main() {
  resp, err := http.Get("https://www.packtpub.com/")
  if err != nil {
    panic(err)
  }

  data, err := ioutil.ReadAll(resp.Body)
  if err != nil {
    panic(err)
  }

  stringBody := string(data)

  numLinks := strings.Count(stringBody, "<a")
  fmt.Printf("Packt Publishing homepage has %d links!\n", numLinks)
}
```

在此示例中，`Count()`函数用于查找 Packt Publishing 网站主页中`"<a"`的出现次数。

# 示例-Doctype 检查

`strings`包中的另一个有用方法是`Contains()`方法。这用于检查字符串中子字符串的存在。例如，您可以检查用于构建网页的 HTML 版本，类似于此处给出的示例：

```go
package main

import (
  "io/ioutil"
  "net/http"
  "strings"
)

func main() {
  resp, err := http.Get("https://www.packtpub.com/")
  if err != nil {
    panic(err)
  }

  data, err := ioutil.ReadAll(resp.Body)
  if err != nil {
    panic(err)
  }

  stringBody := strings.ToLower(string(data))

  if strings.Contains(stringBody, "<!doctype html>") {
    println("This webpage is HTML5")
  } else if strings.Contains(stringBody, "html/strict.dtd") {
    println("This webpage is HTML4 (Strict)")
  } else if strings.Contains(stringBody, "html/loose.dtd") {
    println("This webpage is HTML4 (Tranistional)")
  } else if strings.Contains(stringBody, "html/frameset.dtd") {
    println("This webpage is HTML4 (Frameset)")
  } else {
    println("Could not determine doctype!")
  }
}
```

此示例查找包含在`<!doctype>`标记中的信息，以检查它是否包含 HTML 版本的特定指示符。运行此代码将向您显示 Packt Publishing 的主页是按照 HTML 5 规范构建的。

依赖`strings`包可以揭示有关网页的一些非常轻的信息，但它也有其缺点。在前面的两个示例中，如果文档中包含字符串的句子出现在意想不到的位置，匹配可能会误导。过于概括的字符串搜索可能导致误导，可以通过使用更健壮的工具避免。

# 使用 regexp 包进行搜索

Go 标准库中的`regexp`包通过使用正则表达式提供了更深层次的搜索。这定义了一种语法，允许您以更复杂的术语搜索字符串，并从文档中检索字符串。通过在正则表达式中使用捕获组，您可以从网页中提取与查询匹配的数据。以下是`regexp`包可以帮助您实现的一些有用任务。

# 示例-查找链接

在上一节中，我们使用了`strings`包来计算页面上链接的数量。通过使用`regexp`包，我们可以进一步使用以下正则表达式检索实际链接：

```go
 <a.*href\s*=\s*"'["'].*>
```

此查询应匹配任何看起来像 URL 的字符串，位于`<a>`标记内的`href`属性内。

以下程序打印 Packt Publishing 主页上的所有链接。使用相同的技术可以用于收集所有图像，通过查询`<img>`标记的`src`属性：

```go
package main

import (
  "fmt"
  "io/ioutil"
  "net/http"
        "regexp"
)

func main() {
  resp, err := http.Get("https://www.packtpub.com/")
  if err != nil {
    panic(err)
  }

  data, err := ioutil.ReadAll(resp.Body)
  if err != nil {
    panic(err)
  }

  stringBody := string(data)

        re := regexp.MustCompile(`<a.*href\s*=\s*"'["'].*>`)
        linkMatches := re.FindAllStringSubmatch(stringBody, -1)

        fmt.Printf("Found %d links:\n", len(linkMatches))
        for _,linkGroup := range(linkMatches){
            println(linkGroup[1])
        }
}
```

# 示例-查找价格

正则表达式也可以用于查找网页本身显示的内容。例如，您可能正在尝试查找物品的价格。让我们看一下以下示例，显示了 Packt Publishing 网站上*Hands-On Go Programming*书的价格：

```go
package main

import (
  "fmt"
  "io/ioutil"
  "net/http"
        "regexp"
)

func main() {
  resp, err := http.Get("https://www.packtpub.com/application-development/hands-go-programming")
  if err != nil {
    panic(err)
  }

  data, err := ioutil.ReadAll(resp.Body)
  if err != nil {
    panic(err)
  }

  stringBody := string(data)

  re := regexp.MustCompile(`.*main-book-price.*\n.*(\$[0-9]*\.[0-9]{0,2})`)
  priceMatches := re.FindStringSubmatch(stringBody)

  fmt.Printf("Book Price: %s\n", priceMatches[1])
}
```

该程序查找与`main-book-price`匹配的文本字符串，然后查找以下行中的 USD 格式的小数。

您可以看到，正则表达式可用于提取文档中的字符串，而`strings`包主要用于发现字符串。这两种技术都存在同样的问题：您可能会在意想不到的地方匹配字符串。为了更细粒度地进行搜索，搜索需要更加结构化。

# 使用 XPath 查询进行搜索

在以前的解析 HTML 文档的示例中，我们将 HTML 简单地视为可搜索的文本，您可以通过查找特定字符串来发现信息。幸运的是，HTML 文档实际上是有结构的。您可以看到每组标签可以被视为某个对象，称为节点，它又可以包含更多的节点。这创建了一个根节点、父节点和子节点的层次结构，提供了一个结构化的文档。特别是，HTML 文档与 XML 文档非常相似，尽管它们并非完全符合 XML 标准。由于这种类似 XML 的结构，我们可以使用 XPath 查询在页面中搜索内容。

XPath 查询定义了在 XML 文档中遍历节点层次结构并返回匹配元素的方法。在我们之前的示例中，我们需要搜索字符串来查找`<a>`标签以计算和检索链接。如果在 HTML 文档的意外位置（如代码注释或转义文本）中找到类似的匹配字符串，这种方法可能会有问题。如果我们使用 XPath 查询，如`//a/@href`，我们可以遍历 HTML 文档结构以找到实际的`<a>`标签节点并检索`href`属性。

# 示例 - 每日优惠

使用结构化查询语言如 XPath，您也可以轻松收集未格式化的数据。在我们之前的示例中，我们主要关注产品的价格。价格更容易处理，因为它们通常遵循特定的格式。例如，您可以使用正则表达式来查找美元符号，后跟一个或多个数字，一个句点和两个更多数字。另一方面，如果您想要检索没有格式的文本块或多个文本块，那么使用基本的字符串搜索将变得更加困难。XPath 通过允许您检索节点内的所有文本内容来简化此过程。

Go 标准库对 XML 文档和元素的处理有基本支持；不幸的是，它不支持 XPath。然而，开源社区已经为 Go 构建了各种 XPath 库。我推荐的是 GitHub 用户`antchfx`的`htmlquery`。

您可以使用以下命令获取此库：

```go
go get github.com/antchfx/htmlquery
```

以下示例演示了如何使用 XPath 查询来获取一些基本产品信息来抓取每日优惠：

```go
package main

import (
  "regexp"
  "strings"

  "github.com/antchfx/htmlquery"
)

func main() {
  doc, err := htmlquery.LoadURL("https://www.packtpub.com/packt/offers/free-learning")
  if err != nil {
    panic(err)
  }

  dealTextNodes := htmlquery.Find(doc, `//div[@class="dotd-main-book-summary float-left"]//text()`)

  if err != nil {
    panic(err)
  }

  println("Here is the free book of the day!")
  println("----------------------------------")

  for _, node := range dealTextNodes {
    text := strings.TrimSpace(node.Data)
    matchTagNames, _ := regexp.Compile("^(div|span|h2|br|ul|li)$")
    text = matchTagNames.ReplaceAllString(text,"")
    if text != "" {
      println(text)
    }
  }
}
```

该程序选择包含`class`属性的`div`元素中找到的任何`text()`。此查询还返回目标`div`元素的子节点的名称，例如`div`和`h2`，以及空文本节点。因此，我们放弃任何已知的 HTML 标签（使用正则表达式），并且只打印不是空字符串的剩余文本节点。

# 示例 - 收集产品

在这个示例中，我们将使用 XPath 查询来从 Packt Publishing 网站检索最新的发布信息。在这个网页上，有一系列包含更多`<div>`标签的`<div>`标签，最终会导致我们的信息。这些`<div>`标签中的每一个都有一个称为`class`的属性，描述了节点的用途。特别是，我们关注`landing-page-row`类。`landing-page-row`类中与书籍相关的`<div>`标签有一个称为`itemtype`的属性，告诉我们这个`div`是用于书籍的，并且应该包含其他包含名称和价格的属性。使用`strings`包无法实现这一点，而使用正则表达式将非常费力。

让我们看看以下示例：

```go
package main

import (
  "fmt"
  "strconv"

  "github.com/antchfx/htmlquery"
)

func main() {
  doc, err := htmlquery.LoadURL("https://www.packtpub.com/latest-
  releases")
  if err != nil {
    panic(err)
  }

  nodes := htmlquery.Find(doc, `//div[@class="landing-page-row 
  cf"]/div[@itemtype="http://schema.org/Product"]`)
  if err != nil {
    panic(err)
  }

  println("Here are the latest releases!")
  println("-----------------------------")

  for _, node := range nodes {
    var title string
    var price float64

    for _, attribute := range node.Attr {
      switch attribute.Key {
      case "data-product-title":
        title = attribute.Val
      case "data-product-price":
        price, err = strconv.ParseFloat(attribute.Val, 64)
        if err != nil {
          println("Failed to parse price")
        }
      }
    }
    fmt.Printf("%s ($%0.2f)\n", title, price)
  }
}
```

通过使用直接针对文档中的元素的 XPath 查询，我们能够导航到确切节点的确切属性，以检索每本书的名称和价格。

# 使用层叠样式表选择器进行搜索

您可以看到，使用结构化查询语言比基本字符串搜索更容易搜索和检索信息。但是，XPath 是为通用 XML 文档设计的，而不是 HTML。还有一种专门用于 HTML 的结构化查询语言。**层叠样式表**（**CSS**）是为 HTML 页面添加样式元素的一种方法。在 CSS 文件中，您可以定义到一个元素或多个元素的路径，以及描述外观。元素路径的定义称为 CSS 选择器，专门为 HTML 文档编写。

CSS 选择器了解我们可以在搜索 HTML 文档中使用的常见属性。在以前的 XPath 示例中，我们经常使用这样的查询`div[@class="some-class"]`来搜索具有类名`some-class`的元素。CSS 选择器通过简单地使用`.`为`class`属性提供了一种简写。相同的 XPath 查询在 CSS 查询中看起来像`div.some-class`。这里使用的另一个常见简写是搜索具有`id`属性的元素，这在 CSS 中表示为`#`符号。为了找到具有`main-body` id 的元素，您将使用`div#main-body`作为 CSS 选择器。CSS 选择器规范中还有许多其他便利之处，可以扩展 XPath 的功能，同时简化常见查询。

尽管 Go 标准库中不支持 CSS 选择器，但再次，开源社区有许多工具提供了这种功能，其中最好的是 GitHub 用户`PuerkitoBio`的`goquery`。

您可以使用以下命令获取该库：

```go
go get github.com/PuerkitoBio/goquery
```

# 示例-每日优惠

以下示例将使用`goquery`代替`htmlquery`来完善 XPath 示例：

```go
package main

import (
  "fmt"
  "strconv"

  "github.com/PuerkitoBio/goquery"
)

func main() {
  doc, err := goquery.NewDocument("https://www.packtpub.com/latest-
  releases")
  if err != nil {
    panic(err)
  }

  println("Here are the latest releases!")
  println("-----------------------------")
  doc.Find(`div.landing-page-row div[itemtype$="/Product"]`).
    Each(func(i int, e *goquery.Selection) {
      var title string
      var price float64

      title,_ = e.Attr("data-product-title")
      priceString, _ := e.Attr("data-product-price")
      price, err = strconv.ParseFloat(priceString, 64)
      if err != nil {
        println("Failed to parse price")
      }
      fmt.Printf("%s ($%0.2f)\n", title, price)
    })
}
```

使用`goquery`，搜索每日优惠变得更加简洁。在此查询中，我们使用 CSS 选择器提供的一个辅助功能，即使用`$=`运算符。我们不再寻找`itemtype`属性，匹配确切的字符串`http://schema.org/Product`，而是简单地匹配以`/Product`结尾的字符串。我们还使用`.`运算符来查找`landing-page-row`类。需要注意的一个关键区别是，与 XPath 示例之间的一个关键区别是，您不需要匹配类属性的整个值。当我们使用 XPath 搜索时，我们必须使用`@class="landing-page-row cf"`作为查询。在 CSS 中，不需要对类进行精确匹配。只要元素包含`landing-page-row`类，它就匹配。

# 示例-收集产品

在此提供的代码中，您可以看到收集产品示例的 CSS 选择器版本：

```go
package main

import (
  "bufio"
  "strings"

  "github.com/PuerkitoBio/goquery"
)

func main() {
  doc, err := goquery.NewDocument("https://www.packtpub.com/packt/offers/free-learning")
  if err != nil {
    panic(err)
  }

  println("Here is the free book of the day!")
  println("----------------------------------")
  rawText := doc.Find(`div.dotd-main-book-summary div:not(.eighteen-days-countdown-bar)`).Text()
  reader := bufio.NewReader(strings.NewReader(rawText))

  var line []byte
  for err == nil{
    line, _, err = reader.ReadLine()
    trimmedLine := strings.TrimSpace(string(line))
    if trimmedLine != "" {
      println(trimmedLine)
    }
  }
}
```

在这个例子中，您可以使用 CSS 查询来返回所有子元素的所有文本。我们使用`:not()`运算符来排除倒计时器，并最终处理文本行以忽略空格和空行。

# 总结

您可以看到有各种方法可以使用不同的工具从 HTML 页面中提取数据。基本字符串搜索和`regex`搜索可以使用非常简单的技术收集信息，但也有需要更多结构化查询语言的情况。XPath 通过假设文档是 XML 格式并可以进行通用搜索，提供了出色的搜索功能。CSS 选择器是搜索和提取 HTML 文档中数据的最简单方法，并提供了许多有用的 HTML 特定功能。

在第五章中，*网络爬虫导航*，我们将探讨高效和安全地爬取互联网的最佳方法。
