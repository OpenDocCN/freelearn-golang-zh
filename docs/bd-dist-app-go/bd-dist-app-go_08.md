# 第六章：扩展 Gin 应用程序

在本章中，你将学习如何提高使用 Gin 框架编写的分布式 Web 应用程序的性能和可伸缩性。本章将涵盖如何使用缓存机制来缓解性能瓶颈。在这个过程中，你将学习如何使用如 RabbitMQ 的消息代理解决方案来扩展 Web 应用程序。最后，你将学习如何**容器化**应用程序并使用**Docker Compose**进行扩展。

在本章中，我们将涵盖以下主题：

+   使用消息代理扩展工作负载

+   使用 Docker 副本水平扩展

+   使用 Nginx 反向代理

+   使用 HTTP 缓存头缓存资源

到本章结束时，你将能够使用 Gin 框架、Docker 和 RabbitMQ 构建一个高度可用和分布式的 Web 应用程序。

# 技术要求

要跟随本章的内容，你需要以下条件：

+   完全理解前一章的内容。本章是前一章的后续，因为它将使用相同的源代码。因此，一些代码片段将不会解释，以避免重复。

+   理解 Docker 及其架构。理想情况下，一些使用消息队列服务（如 RabbitMQ、ActiveMQ、Kafka 等）的先前经验将有所帮助。

本章的代码包托管在 GitHub 上，网址为[`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter06`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter06)。

# 使用消息代理扩展工作负载

在开发 Web 应用程序时，用户体验的一个重要方面是响应时间，这往往被忽视。没有任何东西能比一个缓慢而迟钝的应用程序更快地让用户离开。在前几章中，你学习了如何使用 Redis 减少数据库查询以实现更快的数据访问。在本章中，你将进一步学习如何使用 Gin 框架编写的 Web 应用程序进行扩展。

在我们深入探讨为什么需要扩展应用程序的工作负载之前，让我们在架构中添加另一个模块。新的服务将解析 Reddit RSS 源并将源条目插入 MongoDB 的`recipes`集合中。以下图示说明了新服务如何与架构集成：

![图 6.1 – 解析 Reddit RSS 源](img/Figure_6.1_B17115.jpg)

图 6.1 – 解析 Reddit RSS 源

该服务将接受一个 subreddit RSS URL 作为参数。我们可以在现有的 subreddit URL 末尾添加`.rss`来创建一个 RSS 源：

[`www.reddit.com/r/THREAD_NAME/.rss`](https://www.reddit.com/r/THREAD_NAME/.rss)

例如，让我们看看以下截图中的 recipes subreddit：

![图 6.2 – Recipes subreddit](img/Figure_6.2_B17115.jpg)

图 6.2 – Recipes subreddit

该 subreddit 的 RSS 源 URL 如下：

[`www.reddit.com/r/recipes/.rss`](https://www.reddit.com/r/recipes/.rss)

如果您访问上述 URL，您应该收到一个 XML 响应。以下是由食谱 subreddits 的 RSS URL 返回的 XML 结构示例：

```go
<?xml version="1.0" encoding="UTF-8"?>
<feed
   xmlns="http://www.w3.org/2005/Atom"
   xmlns:media="http://search.yahoo.com/mrss/">
   <title>recipes</title>
   <entry>
       <author>
           <name>/u/nolynskitchen</name>
           <uri>https://www.reddit.com/user/nolynskitchen
           </uri>
       </author>
       <category term="recipes" label="r/recipes"/>
       <id>t3_m4uvlm</id>
       <media:thumbnail url="https://b.thumbs.
          redditmedia.com
          /vDz3xCmo10TFkokqy9y1chopeIXdOqtGA33joNBtTDA.jpg" 
        />
       <link href="https://www.reddit.com/r/recipes
                  /comments/m4uvlm/best_butter_cookies/" />
       <updated>2021-03-14T12:57:05+00:00</updated>
       <title>Best Butter Cookies!</title>
   </entry>
</feed>
```

要开始，请按照以下步骤操作：

1.  创建一个 `rss-parser` 项目，将其加载到 VSCode 编辑器中，并编写一个 `main.go` 文件。在文件中，声明一个 `Feed` 结构体以反映 XML 结构：

    ```go
    type Feed struct {
       Entries []Entry `xml:"entry"`
    }
    type Entry struct {
       Link struct {
           Href string `xml:"href,attr"`
       } `xml:"link"`
       Thumbnail struct {
           URL string `xml:"url,attr"`
       } `xml:"thumbnail"`
       Title string `xml:"title"`
    } 
    ```

1.  接下来，编写一个 `GetFeedEntries` 方法，该方法接受 RSS URL 作为参数，并返回一个条目列表：

    ```go
    func GetFeedEntries(url string) ([]Entry, error) {
       client := &http.Client{}
       req, err := http.NewRequest("GET", url, nil)
       if err != nil {
           return nil, err
       }
       req.Header.Add("User-Agent", "Mozilla/5.0 (
          Windows NT 10.0; Win64; x64) AppleWebKit/537.36 
          (KHTML, like Gecko) Chrome/70.0.3538.110 
          Safari/537.36")
       resp, err := client.Do(req)
       if err != nil {
           return nil, err
       }
       defer resp.Body.Close()
       byteValue, _ := ioutil.ReadAll(resp.Body)
       var feed Feed
       xml.Unmarshal(byteValue, &feed)
       return feed.Entries, nil
    }
    ```

    此方法使用 HTTP 客户端向 `GetFeedEntries` 方法参数中给出的 URL 发起 GET 请求。然后，将响应体编码到 `Feed` 结构体中。最后，它返回 `Entries` 属性。

    注意使用 `User-Agent` 请求头模拟从浏览器发送的请求，以避免被 Reddit 服务器阻止：

    ```go
    req.Header.Add("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.110 Safari/537.36") 
    ```

    注意

    可以在以下 URL 找到有效 User-Agent 列表（它定期更新）：[`developers.whatismybrowser.com/useragents/explore/`](https://developers.whatismybrowser.com/useragents/explore/)。

1.  接下来，使用 Gin 路由器创建一个 Web 服务器，并在 `/parse` 端点公开 POST 请求。然后，定义一个名为 `ParserHandler` 的路由处理程序：

    ```go
    func main() {
       router := gin.Default()
       router.POST("/parse", ParserHandler)
       router.Run(":5000")
    }
    ```

    `ParserHandler` 的名称是自解释的：它将请求有效载荷序列化为 `Request` 结构体。然后，它使用 `Request` 结构体的 `URL` 属性调用 `GetFeedEntries` 方法。最后，根据方法响应，它返回一个 500 错误代码或 200 状态代码，以及一个条目列表：

    ```go
    func ParserHandler(c *gin.Context) {
       var request Request
       if err := c.ShouldBindJSON(&request); err != nil {
           c.JSON(http.StatusBadRequest, gin.H{
              "error": err.Error()})
           return
       }
       entries, err := GetFeedEntries(request.URL)
       if err != nil {
           c.JSON(http.StatusInternalServerError, 
              gin.H{"error": "Error while parsing 
                     the rss feed"})
           return
       }
       c.JSON(http.StatusOK, entries)
    } 
    ```

    `Request` 结构体有一个 `URL` 属性：

    ```go
    type Request struct {
      URL string `json:"url"`
    }
    ```

1.  为了测试它，在不同的端口（例如，`5000`）上运行服务器，以避免与已运行在端口 `8080` 上的食谱 API 发生端口冲突：![图 6.3 – RSS 解析日志    ](img/Figure_6.3_B17115.jpg)

    图 6.3 – RSS 解析日志

1.  在 Postman 客户端，对 `/parse` 端点发起一个 POST 请求，请求体中包含 subreddits 的 URL。服务器将解析 RSS 源并返回一个条目列表，如下截图所示：![图 6.4 – RSS 源条目    ](img/Figure_6.4_B17115.jpg)

    图 6.4 – RSS 源条目

1.  现在，通过连接到之前章节中部署的 MongoDB 服务器将结果插入 MongoDB。在 `init()` 方法中定义连接指令，如下所示：

    ```go
    var client *mongo.Client
    var ctx context.Context
    func init() {
       ctx = context.Background()
       client, _ = mongo.Connect(ctx, 
          options.Client().ApplyURI(os.Getenv("MONGO_URI")))
    }
    ```

1.  然后，更新 HTTP 处理器以使用 `InsertOne` 操作将条目插入到 `recipes` 集合中：

    ```go
    func ParserHandler(c *gin.Context) {
       ...
       collection := client.Database(os.Getenv(
          "MONGO_DATABASE")).Collection("recipes")
       for _, entry := range entries[2:] {
           collection.InsertOne(ctx, bson.M{
               "title":     entry.Title,
               "thumbnail": entry.Thumbnail.URL,
               "url":       entry.Link.Href,
           })
       }
       ...
    }
    ```

1.  重新运行应用程序，但这次，提供 `MONGO_URI` 和 `MONGO_DATABASE` 环境变量，如下所示：

    ```go
    MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin&readPreference=primary&appname=MongoDB%20Compass&ssl=false" MONGO_DATABASE=demo go run main.go 
    ```

1.  使用 Postman 或 `curl` 命令重新发起 POST 请求。回到 MongoDB Compass 并刷新 `recipes` 集合。RSS 条目应该已成功插入，如下所示：

![图 6.5 – 食谱集合](img/Figure_6.5_B17115.jpg)

图 6.5 – 食谱集合

注意

如果您正在使用与之前章节中显示的相同数据库和集合，在插入新食谱之前，您可能需要删除现有文档。

`recipes`集合现在已初始化为一系列食谱。

您可以重复相同的步骤来解析其他 subreddit RSS 源。然而，如果您想解析成千上万的 subreddits，处理如此大量的工作负载将需要大量的资源（CPU/RAM），并且会耗费大量时间。这就是为什么我们将服务逻辑分割成多个松散耦合的服务，然后根据传入的工作负载进行扩展。

这些服务需要相互通信，最有效的通信方式是使用消息代理。这就是 RabbitMQ 发挥作用的地方。

## 部署 RabbitMQ 与 Docker

**RabbitMQ** ([`www.rabbitmq.com/#getstarted`](https://www.rabbitmq.com/#getstarted))是一个可靠且高度可用的消息代理。以下架构描述了 RabbitMQ 将在应用程序架构中的应用方式：

![图 6.6 – 使用 RabbitMQ 进行扩展](img/Figure_6.6_B17115.jpg)

图 6.6 – 使用 RabbitMQ 进行扩展

要使用 Docker 部署 RabbitMQ，使用官方的 RabbitMQ Docker 镜像([`hub.docker.com/_/rabbitmq`](https://hub.docker.com/_/rabbitmq))并执行以下步骤：

1.  输入以下命令从 RabbitMQ 镜像运行容器：

    ```go
    8080 and the server on port 5672.
    ```

1.  容器部署完成后，运行以下命令以显示服务器日志：

    ```go
    docker logs -f CONTAINER_ID 
    ```

    这就是启动日志的显示方式：

    ![图 6.7 – RabbitMQ 启动日志    ](img/Figure_6.7_B17115.jpg)

    图 6.7 – RabbitMQ 启动日志

1.  一旦服务器初始化完成，通过浏览器导航到`localhost:8080`。将显示一个 RabbitMQ 登录页面。使用您的用户/密码凭据登录。您将进入仪表板：![图 6.8 – RabbitMQ 仪表板    ](img/B17115_06_08_v2.jpg)

    图 6.8 – RabbitMQ 仪表板

1.  现在，创建一个消息队列，服务将把 RSS URL 推送到该队列。点击导航栏中的**队列**，然后点击**添加新队列**来创建一个新的队列：![图 6.9 – 创建新的 RabbitMQ 队列    ](img/B17115_06_09_v2.jpg)

    图 6.9 – 创建新的 RabbitMQ 队列

1.  确保将**持久性**字段设置为**持久**，以便在 RabbitMQ 断电时数据能够持久化到磁盘。

在 RabbitMQ 运行起来后，我们可以继续实现一个生产者服务，将传入的 RSS URL 推送到 RabbitMQ，以及一个消费者服务从队列中消费 URL。

## 探索生产者/消费者模式

在我们深入实现之前，我们需要探索生产者/消费者模式。以下架构说明了这两个概念：

![图 6.10 – 生产者/消费者模式](img/B17115_06_10_v2.jpg)

图 6.10 – 生产者/消费者模式

现在主要概念已经清晰，让我们开始吧：

1.  创建一个名为 `producer` 的新 Go 项目，并使用以下命令安装 RabbitMQ SDK for Golang：

    ```go
    go get github.com/streadway/amqp
    ```

1.  编写一个 `main.go` 文件，并使用以下代码片段设置与 RabbitMQ 服务器的 TCP 连接：

    ```go
    var channelAmqp *amqp.Channel
    func init() {
       amqpConnection, err := amqp.Dial(os.Getenv(
          "RABBITMQ_URI"))
       if err != nil {
           log.Fatal(err)
       }
       channelAmqp, _ = amqpConnection.Channel()
    }
    ```

    AMQP 连接字符串将通过 `RABBITMQ_URI` 环境变量提供，以及密码。

1.  接下来，在 `/parse` 端点定义一个 HTTP 处理器。该处理器将请求体中给出的 URL 使用 `Publish` 方法推送到 RabbitMQ 队列：

    ```go
    func ParserHandler(c *gin.Context) {
       var request Request
       if err := c.ShouldBindJSON(&request); err != nil {
           c.JSON(http.StatusBadRequest, gin.H{
              "error": err.Error()})
           return
       }
       data, _ := json.Marshal(request)
       err := channelAmqp.Publish(
           "",
           os.Getenv("RABBITMQ_QUEUE"),
           false,
           false,
           amqp.Publishing{
               ContentType: "application/json",
               Body:        []byte(data),
           })
       if err != nil {
           fmt.Println(err)
           c.JSON(http.StatusInternalServerError, 
              gin.H{"error": "Error while publishing 
              to RabbitMQ"})
           return
       }
       c.JSON(http.StatusOK, map[string]string{
          "message": "success"})
    }
    func main() {
      router := gin.Default()
      router.POST("/parse", ParserHandler)
      router.Run(":5000")
    }
    ```

1.  最后，使用 `RABBITMQ_URI` 和 `RABBITMQ_QUEUE` 变量运行应用程序，如下所示：

    ```go
    RABBITMQ_URI="amqp://user:password@localhost:5672/" RABBITMQ_QUEUE=rss_urls go run main.go
    ```

1.  然后，在 `/parse` 端点执行 POST 请求。你应该会收到一个 200 **成功** 消息，如图所示：![Figure 6.11 – 在 RabbitMQ 中发布数据

    ![img/Figure_6.11_B17115.jpg]

    图 6.11 – 在 RabbitMQ 中发布数据

1.  返回 RabbitMQ 仪表板，转到 **Queues** 部分，并点击 **rss_urls** 队列。你应该会被重定向到 **Queue metrics** 页面。在这里，你会注意到队列中的一个消息：

![Figure 6.12 – 队列指标页面

![img/B17115_06_12_v2.jpg]

图 6.12 – 队列指标页面

在生产者服务运行起来后，你需要构建工作/消费者以消费 RabbitMQ 队列中可用的消息/URL：

1.  创建一个名为 `consumer` 的新 Go 项目，并创建一个名为 `main.go` 的新文件。在文件内编写以下代码：

    ```go
    func main() {
       amqpConnection, err := amqp.Dial(os.Getenv(
           "RABBITMQ_URI"))
       if err != nil {
           log.Fatal(err)
       }
       defer amqpConnection.Close()
       channelAmqp, _ := amqpConnection.Channel()
       defer channelAmqp.Close()
       forever := make(chan bool)
       msgs, err := channelAmqp.Consume(
           os.Getenv("RABBITMQ_QUEUE"),
           "",
           true,
           false,
           false,
           false,
           nil,
       )
       go func() {
           for d := range msgs {
               log.Printf("Received a message: %s", d.Body)
           }
       }()
       log.Printf(" [*] Waiting for messages. 
                  To exit press CTRL+C")
       <-forever
    }
    ```

    代码很简单：它设置了一个到 RabbitMQ 服务器的连接，并订阅了 `rss_urls` 队列。然后，它创建了一个无限循环，并从队列中获取一个消息，之后在控制台上显示消息体，并等待新消息。

1.  通过传递 RabbitMQ URI 和队列名称作为环境变量来运行消费者项目：

    ```go
    RABBITMQ_URI="amqp://user:password@localhost:5672/" RABBITMQ_QUEUE=rss_urls go run main.go
    ```

    一旦启动，消费者将获取之前由生产者推送的消息，并在控制台上显示其内容。然后，它将从队列中删除该消息：

    ![Figure 6.13 – 从 RabbitMQ 订阅和获取消息

    ![img/B17115_06_13_v2.jpg]

    图 6.13 – 从 RabbitMQ 订阅和获取消息

1.  通过刷新 **Queue metrics** 页面来验证消息已被删除。**Queued messages** 图表应确认此删除：

![Figure 6.14 – 从队列中删除消息

![img/Figure_6.14_B17115.jpg]

图 6.14 – 从队列中删除消息

这样，你的工作/消费者已经构建完成！

到目前为止，你已经看到消费者显示了消息的内容。现在，让我们更进一步，将消息体编码到 `Request` 结构体中，并通过调用我们之前提到的 `GetFeedEntries` 方法获取内容条目。然后，这些条目将被保存在 MongoDB 的 `recipes` 集合中：

```go
go func() {
       for d := range msgs {
           log.Printf("Received a message: %s", d.Body)
           var request Request
           json.Unmarshal(d.Body, &request)
           log.Println("RSS URL:", request.URL)
           entries, _ := GetFeedEntries(request.URL)
           collection := mongoClient.Database(os.Getenv(
              "MONGO_DATABASE")).Collection("recipes")
           for _, entry := range entries[2:] {
               collection.InsertOne(ctx, bson.M{
                   "title":     entry.Title,
                   "thumbnail": entry.Thumbnail.URL,
                   "url":       entry.Link.Href,
               })
           }
       }
}()
```

重新运行应用程序，但这次，除了 RabbitMQ 参数外，还需要提供 MongoDB 连接参数：

```go
RABBITMQ_URI="amqp://user:password@localhost:5672/" RABBITMQ_QUEUE=rss_urls MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin&readPreference=primary&appname=MongoDB%20Compass&ssl=false" MONGO_DATABASE=demo go run main.go 
```

为了测试这一点，向生产服务器发送一个包含 RSS feed URL 的请求体中的 POST 请求。生产者将在 RabbitMQ 队列中发布该 URL。从那里，消费者将获取消息并获取 RSS URL 的 XML 响应，将响应编码为条目数组，并将结果保存到 MongoDB 中：

![图 6.15 – 消费者服务器日志![img/B17115_06_15.jpg](img/B17115_06_15.jpg)

图 6.15 – 消费者服务器日志

你可以向生产服务器发送多个 subreddit URL。这次，消费者将逐个获取 URL，如下所示：

![图 6.16 – 解析多个 RSS URLs![img/B17115_06_16.jpg](img/B17115_06_16.jpg)

图 6.16 – 解析多个 RSS URLs

要查看已保存在 MongoDB 中的条目，构建一个简单的仪表板以列出 `recipes` 集合中的所有食谱。你可以从头创建一个新项目，或者在前一章中构建的 Web 应用程序上暴露一个额外的路由以提供食谱的 HTML 表示形式：

```go
router.GET("/dashboard", IndexHandler)
```

然后，确保你更新 `Recipe` 结构体字段以反映 MongoDB 文档字段的架构：

```go
type Recipe struct {
   Title     string `json:"title" bson:"title"`
   Thumbnail string `json:"thumbnail" bson:"thumbnail"`
   URL       string `json:"url" bson:"url"`
} 
```

路由处理程序将简单地调用 `recipes` 集合上的 `Find` 操作以返回所有食谱。然后，它将结果编码到 `recipes` 切片中。最后，它将 `recipes` 变量传递给 HTML 模板以显示结果：

```go
func IndexHandler(c *gin.Context) {
   cur, err := collection.Find(ctx, bson.M{})
   if err != nil {
       c.JSON(http.StatusInternalServerError, 
           gin.H{"error": err.Error()})
       return
   }
   defer cur.Close(ctx)
   recipes := make([]Recipe, 0)
   for cur.Next(ctx) {
       var recipe Recipe
       cur.Decode(&recipe)
       recipes = append(recipes, recipe)
   }
   c.HTML(http.StatusOK, "index.tmpl", gin.H{
       "recipes": recipes,
   })
} 
```

以下为 HTML 模板的内容。它使用 Bootstrap 框架构建一个吸引人的用户界面。它还使用 `range` 关键字遍历 `recipes` 切片中的每个食谱并显示其详细信息（标题、缩略图图像和 Reddit URL）：

```go
<section class="container">
       <div class="row">
           <ul class="list-group">
               {{range .recipes}}
               <li class="list-group-item">
                   <div style="width: 100%;">
                       <img src="img/{{ .Thumbnail }}" 
                          class="card-img-top thumbnail">
                       <span class="title">{{ .Title 
                       }}</span>
                       <a href="{{ .URL }}" target="_blank"
                          class="btn btn-warning 
                          btn-sm see_recipe">See recipe</a>
                   </div>
               </li>
               {{end}}
           </ul>
       </div>
</section> 
```

将 Gin 服务器配置为在端口 `3000` 上运行，并使用 `go run main.go` 命令以及 `MONGO_URI` 和 `MONGO_DATABASE` 变量来执行服务器。在你的浏览器中，转到 Localhost:3000/dashboard，除了返回食谱列表的地方，如下面的截图所示：

![图 6.17 – Trending Reddit recipes![img/B17115_06_17.jpg](img/B17115_06_17.jpg)

图 6.17 – Trending Reddit recipes

注意

应用布局和样式表可以在本书的 GitHub 仓库中找到：[`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter06/dashboard/templates/index.tmpl`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter06/dashboard/templates/index.tmpl)。

太棒了！你现在已经熟悉了如何使用像 RabbitMQ 这样的消息代理来扩展你的 Gin 分布式应用程序。在下一节中，我们将演示另一种通过 Docker 扩展 Gin 分布式应用程序的技术。

# 使用 Docker 副本来水平扩展

到目前为止，你已经学会了如何使用 Gin 框架和 RabbitMQ 构建生产者/消费者架构。在本节中，我们将介绍如何扩展消费者组件，以便我们可以将传入的工作量分配给多个消费者。

您可以通过构建消费者项目的 Docker 镜像并基于该镜像构建多个容器来实现这一点。Docker 镜像是不可变的，这保证了每次基于运行该镜像的镜像创建容器时，环境都是相同的。

以下架构图说明了如何使用多个消费者/工作者：

![图 6.18 – 使用 Docker 规模化多个工作者](img/B17115_06_18.jpg)

图 6.18 – 使用 Docker 规模化多个工作者

要创建 Docker 镜像，我们需要定义一个 `Dockerfile` – 一个包含运行消费者项目所有指令的蓝图。在您的工作者/消费者目录中创建一个 `Dockerfile`，内容如下：

```go
FROM golang:1.16
WORKDIR /go/src/github.com/worker
COPY main.go go.mod go.sum ./
RUN go mod download
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
FROM alpine:latest 
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/worker/app .
CMD ["./app"]
```

此 `Dockerfile` 使用多阶段构建功能构建一个轻量级 Docker 镜像。我们将在下一节中看到它是如何工作的。

## 使用 Docker 多阶段构建

在您的 `Dockerfile` 中的 `FROM` 语句和从一阶段复制工件到另一阶段，留下您在最终镜像中不需要的所有内容。

在前面的示例中，您使用了 `golang:1.16` 作为基础镜像来构建单个二进制文件。然后，第二个 `FROM` 指令以 Alpine 镜像作为其基础启动了新的构建阶段。从这里，您可以使用 `COPY –from=0` 指令从上一个阶段复制二进制文件。结果，您将得到一个小的 Docker 镜像。

要构建镜像，请运行以下命令。末尾的点很重要，因为它指向当前目录：

```go
docker build -t worker . 
```

构建过程应在几秒钟内完成。然后，您将找到以下 Docker 构建日志：

![图 6.19 – Docker 构建日志](img/B17115_06_19.jpg)

图 6.19 – Docker 构建日志

如果您回顾前面的输出，您将看到 Docker 根据我们 `Dockerfile` 中的步骤记录了构建工作者镜像的每条指令。一旦镜像构建完成，运行以下命令以列出您机器上的可用镜像：

```go
docker image ls
```

工作者/消费者镜像应列在列表的顶部：

![图 6.20 – 工作者 Docker 镜像](img/B17115_06_20.jpg)

图 6.20 – 工作者 Docker 镜像

使用 `docker run` 命令基于镜像运行容器。您需要使用 `–e` 标志提供 MongoDB 和 RabbitMQ 的 URI 作为环境变量。可以使用 `–link` 标志在容器内与 MongoDB 和 RabbitMQ 交互：

```go
docker run -d -e MONGO_URI="mongodb://admin:password@mongodb:27017/test?authSource=admin&readPreference=primary&appname=MongoDB%20Compass&ssl=false" -e MONGO_DATABASE=demo2 -e RABBITMQ_URI="amqp://user:password@rabbitmq:5672/" -e RABBITMQ_QUEUE=rss_urls --link rabbitmq --link mongodb --name worker worker 
```

容器日志如下：

![图 6.21 – 工作者的容器日志](img/B17115_06_21.jpg)

图 6.21 – 工作者的容器日志

通过这样，您已经将工作者服务容器化了。接下来，我们将使用 Docker Compose 进行扩展。

## 使用 Docker Compose 规模化服务

**Docker Compose** 是一个基于 Docker 引擎构建的容器编排工具。它可以帮助您通过单个命令行管理应用程序堆栈或多个容器。

使用 Docker Compose 与创建 Docker 镜像一样简单：

1.  在您项目的根目录下定义一个 `docker-compose.yml` 文件，并输入以下 YAML 代码：

    ```go
    version: "3.9"
    services:
     worker:
       image: worker
       environment:
         - MONGO_URI="mongodb://admin:password
               @mongodb:27017/test?authSource=admin
               &readPreference=primary&ssl=false"
         - MONGO_DATABASE=demo2
         - RABBITMQ_URI=amqp://user:password@rabbitmq:5672
         - RABBITMQ_QUEUE=rss_urls
       networks:
         - app_network
       external_links:
         - mongodb
         - rabbitmq
    networks:
     app_network:
       external: true
    ```

    配置指定了工作节点所需的环境变量和网络拓扑。

1.  定义一个外部网络，其中将运行工作节点、MongoDB 和 RabbitMQ 服务。执行以下命令：

    ```go
    docker network create app_network
    ```

1.  重新部署 RabbitMQ 和 MongoDB 容器，但这次，通过传递`–network`标志在自定义网络`app_network`中部署它们：

    ```go
    docker run -d --name rabbitmq -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=password -p 8080:15672 -p 5672:5672 --network app_network rabbitmq:3-management
    ```

1.  容器配置正确后，执行以下命令以部署工作节点：

    ```go
    docker-compose up -d
    ```

    `-d`标志指示 Docker Compose 在后台（分离模式）运行容器。

1.  执行以下命令以列出正在运行的服务：![图 6.22 – Docker Compose 服务    ![图片 B17115_06_22.jpg](img/B17115_06_22.jpg)

    图 6.22 – Docker Compose 服务

1.  要扩展工作节点，重新运行之前的命令并带有`–scale`标志：

    ```go
    docker-compose up -d --scale worker=5
    ```

    最终输出将如下所示：

    ![图 6.23 – 扩展五个工作节点    ![图片 B17115_06_23.jpg](img/B17115_06_23.jpg)

    图 6.23 – 扩展五个工作节点

1.  要测试一切，创建一个名为`threads`的文件，其中包含 Reddit 上最佳烹饪和食谱子版块的列表。以下列表因简洁而被裁剪：

    ```go
    https://www.reddit.com/r/recipes/.rss
    https://www.reddit.com/r/food/.rss
    https://www.reddit.com/r/Cooking/.rss
    https://www.reddit.com/r/IndianFood/.rss
    https://www.reddit.com/r/Baking/.rss
    https://www.reddit.com/r/vegan/.rss
    https://www.reddit.com/r/fastfood/.rss
    https://www.reddit.com/r/vegetarian/.rss
    https://www.reddit.com/r/cookingforbeginners/.rss
    https://www.reddit.com/r/MealPrepSunday/.rss
    https://www.reddit.com/r/EatCheapAndHealthy/.rss
    https://www.reddit.com/r/Cheap_Meals/.rss
    https://www.reddit.com/r/slowcooking/.rss
    https://www.reddit.com/r/AskCulinary/.rss
    https://www.reddit.com/r/fromscratch/.rss
    ```

1.  然后，编写一个`bulk.sh`shell 脚本，逐行读取`threads`文件并发出对生产者服务的 POST 请求：

    ```go
    #!/bin/bash
    while IFS= read -r thread
    do
       printf "\n$thread\n"
       curl -X POST http://localhost:5000/parse -d 
         '{"url":"$thread"}' http://localhost:5000/parse
    done < "threads" 
    ```

1.  要运行脚本，添加执行权限，并使用以下命令执行文件：

    ```go
    chmod +x bulk.sh
    ./bulk.sh 
    ```

    注意

    确保生产者服务正在运行，否则使用`curl`命令发出的 HTTP 请求将超时。

    脚本将逐行读取`threads`文件并发出 POST 请求，如下所示：

    ![图 6.24 – Shell 脚本输出    ![图片 B17115_06_24.jpg](img/B17115_06_24.jpg)

    图 6.24 – Shell 脚本输出

1.  运行`docker-compose logs -f`命令。这次，你应该注意到正在使用多个工作节点。此外，Docker Compose 为实例分配了颜色，并且每个消息都由不同的工作节点获取：

![图 6.25 – 在多个工作节点间分配工作负载![图片 B17115_06_25.jpg](img/B17115_06_25.jpg)

图 6.25 – 在多个工作节点间分配工作负载

这样，你已经成功地将工作负载分配到多个工作节点上。这种方法被称为**水平扩展**。

注意

在*第八章*中，*在 AWS 上部署应用程序*，我们将介绍如何在 AWS 上部署 Web 应用程序以及如何使用**简单队列服务**（**SQS**）而不是 RabbitMQ 来扩展工作节点。

# 使用 NGINX 反向代理

在上一节中，你学习了如何扩展负责解析子版块 URL 的工作节点。在本节中，你将探索如何通过在反向代理后面提供服务来扩展我们在上一章中构建的 Recipes API。

最常用的反向代理之一是 **Nginx**。它面向客户端并接收传入的 HTTP(S)请求。然后，它以轮询的方式将它们重定向到 API 实例之一。要部署多个 Recipes API 实例，你将使用 Docker Compose 来编排容器。以下架构图说明了 **单实例架构** 与使用 Nginx 的 **负载均衡多实例架构** 之间的区别：

![图 6.26 – 使用 Nginx 的负载均衡多实例架构](img/Figure_6.26_B17115.jpg)

图 6.26 – 使用 Nginx 的负载均衡多实例架构

另一种扩展 Recipes API 的解决方案是**垂直扩展**，这包括增加服务运行系统的 CPU/RAM。然而，这种方法在长期来看往往有一些限制（不经济）。这就是为什么在这里，你将采用水平扩展方法，并将负载分配到多个 API 实例。

注意

Nginx 的替代方案是 Traefik ([`doc.traefik.io/traefik/`](https://doc.traefik.io/traefik/))。这是一个以可扩展性为设计理念的开放源代码项目。

以下是对 Recipes API 输出的快速回顾：

![图 6.27 – GET /recipes 端点响应](img/Figure_6.27_B17115.jpg)

图 6.27 – GET /recipes 端点响应

要部署多个 Recipes API 实例，请按照以下步骤操作：

1.  在包含你的 Recipes API 实现的文件夹中构建 Docker 镜像并编写 Dockerfile，内容如下：

    ```go
    FROM golang:1.16
    WORKDIR /go/src/github.com/api
    COPY . .
    RUN go mod download
    RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
    FROM alpine:latest 
    RUN apk --no-cache add ca-certificates
    WORKDIR /root/
    COPY --from=0 /go/src/github.com/api/app .
    CMD ["./app"]
    ```

1.  这个 `Dockerfile` 使用多阶段特性来传输一个轻量级镜像。`docker build -t recipes-api.` 命令的输出如下：![图 6.28 – Docker 构建日志    ](img/Figure_6.28_B17115.jpg)

    图 6.28 – Docker 构建日志

    构建镜像后，创建一个 `docker-compose.yml` 文件并定义三个服务：

    **API**：Recipes API 容器

    **Redis**：内存缓存数据库

    **MongoDB**：存储食谱的非关系型数据库

1.  然后，将以下行添加到文件中：

    ```go
    version: "3.9"
    services:
     api:
       image: recipes-api
       environment:
         - MONGO_URI=mongodb://admin:password
              @mongodb:27017/test?authSource=admin
              &readPreference=primary&ssl=false
         - MONGO_DATABASE=demo
         - REDIS_URI=redis:6379
       networks:
         - api_network
       external_links:
         - mongodb
         - redis
     redis:
       image: redis
       networks:
         - api_network
     mongodb:
       image: mongo:4.4.3
       networks:
         - api_network
       environment:
         - MONGO_INITDB_ROOT_USERNAME=admin
         - MONGO_INITDB_ROOT_PASSWORD=password
    networks:
       api_network:
    ```

1.  接下来，定义一个 Nginx 服务，如下所示：

    ```go
    nginx:
       image: nginx
       ports:
         - 80:80
       volumes:
         - $PWD/nginx.conf:/etc/nginx/nginx.conf
       depends_on:
         - api
       networks:
         - api_network 
    ```

    这段代码将本地的 `nginx.conf` 文件映射到容器内的 `/etc/nginx/nginx.conf`。该文件提供了 Nginx 如何处理传入的 HTTP 请求的指令。以下是一个简化版本：

    ```go
    events {
       worker_connections 1024;
    }
    http {
     server_tokens off;
     server {
       listen 80;
       root  /var/www;
       location /api/ {
         proxy_set_header X-Forwarded-For $remote_addr;
         proxy_set_header Host            $http_host;
         proxy_pass http://api:8080/;
       }
     }
    } 
    ```

1.  在 `location /api` 中设置反向代理，将请求转发到运行在端口 `8080`（内部）的 API。

1.  使用 `docker-compose up –d` 命令部署整个堆栈。然后，执行以下命令以显示正在运行的服务：

    ```go
    docker-compose ps
    ```

    命令的输出如下：

    ![图 6.29 – 应用堆栈    ](img/Figure_6.29_B17115.jpg)

    图 6.29 – 应用堆栈

1.  Nginx 服务在端口 `80` 上公开。访问 `localhost/api/recipes`；服务器将调用 Recipes API 并转发食谱列表响应，如下所示：![图 6.30 – 使用 Nginx 转发 HTTP 响应    ](img/Figure_6.30_B17115.jpg)

    图 6.30 – 使用 Nginx 转发 HTTP 响应

    注意

    对于生产使用，确保您使用 HTTPS 保护您的 API 端点非常重要。幸运的是，您可以使用 Nginx 的 "Let's Encrypt" 扩展来自动生成 TLS 证书。

1.  要确保响应是从 Recipes API 转发的，请使用以下命令检查 Nginx 服务日志：

    ```go
    docker-compose logs –f nginx 
    ```

1.  您应该看到类似以下内容：

    ```go
    /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
    /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
    /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
    10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
    10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
    /docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
    /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
    /docker-entrypoint.sh: Configuration complete; ready for start up
    172.21.0.1 - - [21/Mar/2021:18:11:02 +0000] "GET /api/recipes HTTP/1.1" 200 2 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.192 Safari/537.36"
    ```

1.  到目前为止，Recipes API 容器的一个实例正在运行。要扩展它，使用 `docker-compose up` 命令中的 `–scale` 标志或在 `docker-compose.yml` 文件中定义副本数量，如下所示：

    ```go
    api:
       image: recipes-api
       environment:
         - MONGO_URI=mongodb://admin:password
              @mongodb:27017/test?authSource=admin
              &readPreference=primary&ssl=false
         - MONGO_DATABASE=demo
         - REDIS_URI=redis:6379
       networks:
         - api_network
       external_links:
         - mongodb
         - redis
       scale: 5
    ```

1.  重新执行 `docker-compose up` 命令。将根据 Recipes API Docker 镜像创建四个额外的服务。以下是对服务日志的描述：![图 6.31 – 扩展 Recipes API    ](img/Figure_6.31_B17115.jpg)

    图 6.31 – 扩展 Recipes API

    现在，当客户端发送请求时，它将击中 Nginx，然后以轮询方式转发到 API 服务之一。这有助于我们均匀地分配负载。

    注意

    在 *第十章**，捕获 Gin 应用程序指标* 中，我们将学习如何设置监控平台以在需求增加时触发扩展事件来增加服务数量。

    使用反向代理的优势在于，您可以为您整个分布式 Web 应用程序设置一个单一的入口点。后端和 Web 应用程序都将位于相同的 URL 上。这样，您就不需要在 API 服务器上处理 CORS。

1.  与 Recipes API 类似，为 `react-ui` 服务创建一个 Docker 镜像。以下是我们 `Dockerfile` 的内容：

    ```go
    FROM node:14.15.1
    COPY package-lock.json .
    COPY package.json .
    RUN npm install
    CMD npm start
    ```

    如您所见，这非常简单。在这里，您正在使用预构建的 Node.js 基础镜像，因为 `react-ui` 服务是用 JavaScript 编写的。

1.  使用以下命令构建 Docker 镜像 `docker build -t dashboard`。然后，更新 `docker-compose.yml` 以运行以下代码块中的 Docker 服务：

    ```go
    dashboard:
       image: dashboard
       networks:
         - api_network
    ```

1.  接下来，更新 `nginx.conf` 以确保它将传入请求转发到 URL 的根级别，到仪表板服务：

    ```go
    events {
       worker_connections 1024;
    }
    http {
     server_tokens off;
     server {
       listen 80;
       root  /var/www;
       location / {
         proxy_set_header X-Forwarded-For $remote_addr;
         proxy_set_header Host            $http_host;
         proxy_pass http://dashboard:3000/;
       }
       location /api/ {
         proxy_set_header X-Forwarded-For $remote_addr;
         proxy_set_header Host            $http_host;
         proxy_pass http://api:8080/;
       }
     }
    }
    ```

1.  重新运行 `docker-compose up` `-d` 命令以使更改生效：![图 6.32 – 从 Nginx 提供网络仪表板    ](img/Figure_6.32_B17115.jpg)

    图 6.32 – 从 Nginx 提供网络仪表板

1.  访问 `localhost/dashboard`；您将被重定向到我们在上一章中编写的 Web 仪表板：

![图 6.33 – 从同一 URL 提供两个后端](img/Figure_6.33_B17115.jpg)

图 6.33 – 从同一 URL 提供两个后端

现在，RESTful API 和仪表板都从同一域名提供服务。

要关闭您的容器，您可以使用以下命令：

```go
docker-compose down
```

注意

如果您使用基于会话的认证，您需要在 Nginx 上配置 cookie stickiness 以保持用户会话在启动它的服务器上。

您还可以通过将仪表板位置替换为另一个位置部分，从 Nginx 服务器提供 subreddits 应用（在本章开头构建），到`nginx.conf`文件：

```go
location /reddit/ {
     proxy_set_header X-Forwarded-For $remote_addr;
     proxy_set_header Host            $http_host;
     proxy_pass http://reddit-trending:3000/;
}
```

与`react-ui`类似，为 subreddits 应用创建一个 Docker 镜像。以下是我们`Dockerfile`的内容：

```go
FROM node:14.15.1
COPY . .
COPY package-lock.json .
COPY package.json .
RUN npm install
CMD npm start
```

这里，您正在使用预构建的 Node.js 基础镜像，因为仪表板服务是用 JavaScript 编写的。使用`"docker build -t dashboard"`构建 Docker 镜像。

此外，别忘了将应用添加到`docker-compose.yml`文件中：

```go
  reddit-trending:
    image: web
    networks:
      - api_network 
```

一旦您使用`docker-compose`重新部署了堆栈，请转到[localhost/reddit](http://localhost/reddit)；您将被重定向到以下 UI：

![图 6.34 – 热门食谱应用](img/Figure_6.34_B17115.jpg)

图 6.34 – 热门食谱应用

应用布局损坏是因为`app.css`文件正在从错误的后端提供服务。您可以通过在 Chrome 中打开调试控制台来确认这一点：

![图 6.35 – 样式表位置](img/Figure_6.35_B17115.jpg)

图 6.35 – 样式表位置

您可以通过在`nginx.conf`中添加以下代码块来强制 Nginx 从 subreddit 应用容器中提供`app.css`文件：

```go
location /assets/css/app.css {
     proxy_set_header X-Forwarded-For $remote_addr;
     proxy_set_header Host            $http_host;
     proxy_pass http://reddit-trending:3000/assets
       /css/app.css;
}
```

现在，刷新网页；应用布局将被修复：

![图 6.36 – 应用布局](img/Figure_6.36_B17115.jpg)

图 6.36 – 应用布局

如您所见，仪表板显示了食谱缩略图。每次刷新页面时，这些图片都会从后端提供。为了减轻后端的压力，您可以配置 Nginx 来缓存静态文件。在 Nginx 配置文件中的`server`部分之前插入以下代码片段：

```go
map $sent_http_content_type $expires {
   default                    off;
   text/html                  epoch;
   text/css                   max;
   application/javascript     max;
   ~image/                    max;
}
```

`~image`关键字将处理所有类型的图片（PNG、JPEG、GIF、SVG 等）。现在，在`server`部分使用`expires`指令配置过期时间：

```go
http {
 server {
   listen 80;
   expires $expires;
   ...
 }
}
```

然后，使用以下命令重新部署堆栈：

```go
docker-compose up –d
```

现在图片应该被缓存，这减少了击中后端请求的数量。在下一节中，我们将介绍如何使用 Gin 在后台获得相同的结果。

# 使用 HTTP 缓存头缓存资源

您还可以使用 Gin 框架管理缓存。为了说明这一点，编写一个简单的 Web 应用来提供图片。代码如下：

```go
func IllustrationHandler(c *gin.Context) {
   c.File("illustration.png")
}
func main() {
   router := gin.Default()
   router.GET("/illustration", IllustrationHandler)
   router.Run(":3000")
}
```

当用户点击`/illustration`资源 URL 时，应用应提供一张图片：

![图 6.37 – 使用 Gin 提供图片](img/Figure_6.37_B17115.jpg)

图 6.37 – 使用 Gin 提供图片

由于相同的图片总是被发送，我们需要确保我们正在缓存图片。这样，我们可以避免不必要的流量并提高网页性能。让我们看看这是如何实现的。

## 设置 HTTP 缓存头

要缓存此 HTTP 请求，您可以将一个带有`Etag`键值的`If-None-Match`字段附加到请求中。如果`If-None-Match`字段与生成的键匹配，则返回 304 状态码：

```go
func IllustrationHandler(c *gin.Context) {
   c.Header("Etag", "illustration")
   c.Header("Cache-Control", "max-age=2592000")
   if match := c.GetHeader("If-None-Match"); match != "" {
       if strings.Contains(match, "illustration") {
           c.Writer.WriteHeader(http.StatusNotModified)
           return
       }
   }
   c.File("illustration.png")
}
```

一旦你使用前面的代码更新了 HTTP 处理程序，就测试一下。第一次你请求 `/illustration` 资源时，你应该得到状态 `200 OK`。然而，对于第二次请求，你应该得到 `304 StatusNotModified` 响应：

![图 6.38 – 使用 Gin 的响应缓存![img/Figure_6.38_B17115.jpg](img/Figure_6.38_B17115.jpg)

图 6.38 – 使用 Gin 的响应缓存

你可能也注意到，第二次请求的延迟比第一次短。通过将查询数量保持在最低，你可以减轻 API 对应用程序性能的影响。

# 摘要

在本章中，你学习了如何使用基于微服务架构的 Gin 框架构建分布式网络应用程序。

你还探索了如何设置 RabbitMQ 作为微服务之间的消息代理，以及如何使用 Docker 扩展这些服务。在这个过程中，你学习了如何使用 Docker 的多阶段构建功能来维护服务镜像的大小，以及如何使用 Nginx 和 HTTP 缓存头来提高 API 的性能。

在下一章中，你将学习如何为 Gin 网络应用程序编写单元和集成测试。

# 进一步阅读

+   *《RabbitMQ 精要 – 第二版》，作者：Lovisa Johansson，Packt 出版*

+   *《开发者 Docker》，作者：Richard Bullington-McGuire，Andrew K. Dennis，Michael Schwartz，Packt 出版*。
