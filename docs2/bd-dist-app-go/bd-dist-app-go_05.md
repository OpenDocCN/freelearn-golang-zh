# 第三章：使用 MongoDB 管理数据持久性

在上一章中，我们学习了如何使用 Gin 网络框架构建 RESTful API。在本章中，我们将集成 MongoDB 到后端进行数据存储，同时也会介绍如何使用 Redis 作为缓存层来优化数据库查询。

在本章中，我们将涵盖以下主题：

+   使用 Docker 设置 MongoDB

+   配置 Go MongoDB 驱动

+   实现 MongoDB 查询和 CRUD 操作

+   标准的 Go 项目布局

+   使用 Docker 部署 Redis

+   使用缓存优化 API 响应时间

+   使用 Apache Benchmark 进行性能基准测试

到本章结束时，您将能够使用 Go 在 MongoDB 数据库上执行 CRUD 操作。

# 技术要求

要跟随本章的内容，您将需要以下内容：

+   由于本章是前一章的后续，因此您必须对前一章有完整理解；它将使用相同的源代码。因此，为了避免重复，一些代码片段将不会进行解释。

+   对 NoSQL 概念和 MongoDB 基本查询有一定的了解。

本章的代码包托管在 GitHub 上，网址为 [`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter03`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter03)。

# 运行 MongoDB 服务器

我们迄今为止构建的 API 还未连接到数据库。对于实际应用，我们需要使用某种形式的数据存储；否则，如果 API 崩溃或托管 API 的服务器宕机，数据将会丢失。MongoDB 是最受欢迎的 NoSQL 数据库之一。

以下架构图显示了 MongoDB 将如何集成到 API 架构中：

![图 3.1 – API 架构

![图 3.1 – API 架构

图 3.1 – API 架构

在我们开始之前，我们需要部署一个 MongoDB 服务器。有大量的部署选项：

+   您可以从以下网址下载 MongoDB 社区版二进制文件：[`www.mongodb.com/try/download/community`](https://www.mongodb.com/try/download/community)。根据您的操作系统选择一个软件包：

![图 3.2 – MongoDB 社区服务器

![图 3.2 – MongoDB 社区服务器

图 3.2 – MongoDB 社区服务器

+   您可以使用名为 MongoDB Atlas 的 MongoDB 作为服务解决方案（[`www.mongodb.com/cloud/atlas`](https://www.mongodb.com/cloud/atlas)），在云上运行一个免费的 500 MB 数据库。您可以在 AWS、Google Cloud Platform 或 Microsoft Azure 上部署一个完全管理的 MongoDB 服务器。

+   您可以使用 Docker 等容器化解决方案在本地运行 MongoDB。DockerHub 上有多个 MongoDB 服务器配置好的 Docker 镜像，可以直接使用。

我选择使用 Docker，因为它在运行临时环境方面既受欢迎又简单。

## 安装 Docker CE

Docker ([`www.docker.com/get-started`](https://www.docker.com/get-started)) 是一个开源项目，它允许您运行、构建和管理容器。容器就像一个独立的操作系统，但不是虚拟化；它只包含一个应用程序所需的依赖项，这使得容器可移植，可以在本地或云上部署。

以下图表显示了容器和虚拟机在架构方法上的主要区别：

![图 3.3 – 虚拟机与容器对比](img/B17115_03_03.jpg)

图 3.3 – 虚拟机与容器对比

虚拟机在硬件级别进行虚拟化，而容器在应用层进行虚拟化。因此，容器可以共享操作系统内核和库，这使得它们非常轻量级且资源高效（CPU、RAM、磁盘等）。

要开始，您需要在您的机器上安装 Docker 引擎。导航到 [`docs.docker.com/get-docker/`](https://docs.docker.com/get-docker/) 并为您的平台安装 Docker：

![图 3.4 – Docker 安装](img/B17115_03_04.jpg)

图 3.4 – Docker 安装

注意

Mac 用户也可以使用 Homebrew 工具通过 `brew install docker` 命令安装 Docker。

按照安装向导进行操作，完成后，通过执行以下命令验证一切是否正常：

```go
docker version
```

在撰写本书时，我正在使用 Docker **社区版** (**CE**) 20.10.2 版本，如下截图所示：

![图 3.5 – Docker 社区版 (CE) 版本](img/B17115_03_05.jpg)

图 3.5 – Docker 社区版 (CE) 版本

安装 Docker 后，您可以在终端会话中运行以下命令来部署您的第一个容器：

```go
docker run hello-world
```

以下命令将基于 `hello-world` 镜像部署容器。当容器运行时，它将打印一条 *Hello from Docker*! 消息并退出：

![图 3.6 – Docker hello-world 容器](img/B17115_03_06.jpg)

图 3.6 – Docker hello-world 容器

恭喜！您现在已成功运行 Docker。

## 运行 MongoDB 容器

MongoDB 的官方镜像可以在 DockerHub 上找到 ([`hub.docker.com/_/mongo`](https://hub.docker.com/_/mongo))。有大量的镜像可供选择，每个镜像代表 MongoDB 的不同版本。您可以使用 `latest` 标签来查找它们；然而，建议指定目标版本。在撰写本书时，MongoDB 4.4.3 是最新的稳定版本。执行以下命令以基于该版本部署容器：

```go
docker run -d --name mongodb -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=password -p 27017:27017 mongo:4.4.3
```

此命令将以分离模式（`-d` 标志）运行 MongoDB 容器。我们还映射了容器端口到主机端口，以便我们可以从主机级别访问数据库。最后，我们必须创建一个新用户，并通过 `MONGO_INITDB_ROOT_USERNAME` 和 `MONGO_INITDB_ROOT_PASSWORD` 环境变量设置该用户的密码。

目前，MongoDB 凭据以纯文本形式存在。另一种通过环境变量传递敏感信息的方法是使用 Docker Secrets。如果您在 Swarm 模式下运行，可以执行以下命令：

```go
openssl rand -base64 12 | docker secret create mongodb_password -
```

注意

Docker Swarm 模式原生集成在 Docker 引擎中。它是一个用于在节点集群中构建、部署和扩展容器的容器编排平台。

此命令将为 MongoDB 用户生成一个随机密码并将其设置为 Docker 机密。

接下来，更新 `docker run` 命令，使其使用 Docker 机密而不是纯文本中的密码：

```go
-e MONGO_INITDB_ROOT_PASSWORD_FILE=/run/secrets/mongodb_password
```

`docker run` 命令的输出如下。它从 DockerHub 下载镜像并从中创建一个实例（容器）：

![图 3.7 – 从 DockerHub 拉取 MongoDB 镜像](img/B17115_03_07.jpg)

图 3.7 – 从 DockerHub 拉取 MongoDB 镜像

值得注意的是，如果您已经运行了 MongoDB 容器，在执行前面的命令之前请确保将其删除；否则，您将收到“容器已存在”错误。要删除现有容器，请发出以下命令：

```go
docker rm -f container_name || true 
```

容器创建后，通过键入以下内容来检查日志：

```go
docker logs –f CONTAINER_ID
```

日志应显示 MongoDB 服务器的健康检查：

![图 3.8 – MongoDB 容器运行日志](img/B17115_03_08.jpg)

图 3.8 – MongoDB 容器运行日志

注意

建议您使用 Docker 卷将容器内的 `/data/db` 目录映射到底层主机系统。这样，如果 MongoDB 服务器失败或您的笔记本电脑重启，数据不会丢失（数据持久性）。在主机系统上创建一个数据目录，并使用以下命令将其挂载到 `/data/db` 目录：

`mkdir /home/data`

`docker run -d --name mongodb –v /home/data:/data/db -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=password -p 27017:27017 mongo:4.4.3`

要与 MongoDB 服务器交互，您可以使用 MongoDB 壳来在命令行中发出查询并查看数据。然而，有一个更好的选择：MongoDB Compass。

安装 MongoDB Compass

MongoDB Compass 是一个图形用户界面工具，它允许你轻松构建查询、理解你的数据库模式，并分析你的索引，而无需了解 MongoDB 的查询语法。

根据您的操作系统从 [`www.mongodb.com/try/download/compass?tck=docs_compass`](https://www.mongodb.com/try/download/compass?tck=docs_compass) 下载 Compass：

![图 3.9 – MongoDB Compass 下载页面](img/B17115_03_09.jpg)

图 3.9 – MongoDB Compass 下载页面

下载与您的操作系统相关的包后，运行安装程序并按照其后的步骤操作。安装完成后，打开 Compass，点击 `mongodb://admin:password@localhost:27017/test`。

MongoDB 在本地运行，因此主机名将是 localhost，端口将是 27017：

![图 3.10 – MongoDB Compass – 新连接](img/B17115_03_10.jpg)

图 3.10 – MongoDB Compass – 新连接

点击**连接**按钮。现在，您已连接到您的 MongoDB 服务器。您将看到可用的数据库列表：

![图 3.11 – MongoDB 默认数据库](img/B17115_03_11.jpg)

图 3.11 – MongoDB 默认数据库

到目前为止，我们已经有一个功能性的 MongoDB 部署。在下一节中，我们将使用上一章中构建的 Recipes API 与数据库进行交互。

注意

要停止 MongoDB 服务器，运行`docker ps`命令查看正在运行的容器列表，然后使用`docker stop CONTAINER_ID`停止容器。

# 配置 Go 的 MongoDB 驱动程序

我们在上一章中实现的 Recipes API 是用 Golang 编写的。因此，我们需要安装官方的 MongoDB Go 驱动程序（[`github.com/mongodb/mongo-go-driver`](https://github.com/mongodb/mongo-go-driver)）以与 MongoDB 服务器交互。该驱动程序完全集成到 MongoDB API 中，并支持 API 的所有主要查询和聚合功能。

输入以下命令从 GitHub 安装包：

```go
go get go.mongodb.org/mongo-driver/mongo
```

这将在`go.mod`文件下的`require`部分添加包作为依赖项：

```go
module github.com/mlabouardy/recipes-api
go 1.15
require (
   github.com/gin-gonic/gin v1.6.3
   github.com/rs/xid v1.2.1 
   go.mongodb.org/mongo-driver v1.4.5 
)
```

要开始使用，请在`main.go`文件中导入以下包：

```go
package main
import (
   "go.mongodb.org/mongo-driver/mongo"
   "go.mongodb.org/mongo-driver/mongo/options"
   "go.mongodb.org/mongo-driver/mongo/readpref"
)
```

在`init()`方法中，使用`Connect`函数创建一个`mongo.Client`。此函数接受一个上下文参数和连接字符串，该连接字符串由名为`MONGO_URI`的环境变量提供。同时创建以下全局变量；它们将在所有 CRUD 操作函数中使用：

```go
var ctx context.Context
var err error
var client *mongo.Client
func init() {
   ...
   ctx = context.Background()
   client, err = mongo.Connect(ctx, 
       options.Client().ApplyURI(os.Getenv("MONGO_URI")))
   if err = client.Ping(context.TODO(), 
           readpref.Primary()); err != nil {
       log.Fatal(err)
   }
   log.Println("Connected to MongoDB")
}
```

注意

为了使示例易于阅读和跟随，我已省略了一些代码。完整的源代码可在本书的 GitHub 仓库中找到，位于`chapter03`文件夹下。

一旦`Connect`方法返回客户端对象，我们可以使用`Ping`方法来检查连接是否成功。

将`MONGO_URI`环境变量传递给`go run`命令，并检查应用程序是否可以成功连接到您的 MongoDB 服务器：

```go
MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin" go run main.go
```

如果成功，将显示**已连接到 MongoDB**的消息：

![图 3.12 – 使用 Go 驱动程序的 MongoDB 连接](img/B17115_03_12.jpg)

图 3.12 – 使用 Go 驱动程序的 MongoDB 连接

现在，让我们用一些数据填充一个全新的数据库。

# 探索 MongoDB 查询

在本节中，我们将使用 CRUD 操作与 MongoDB 服务器进行交互，但首先，让我们创建一个数据库，用于存储 API 数据。

注意

您可以在 GoDoc 网站上查看 MongoDB Go 驱动程序的完整文档（[`godoc.org/go.mongodb.org/mongo-driver`](https://godoc.org/go.mongodb.org/mongo-driver)）。

## 插入多个操作

让我们使用上一章中创建的`recipes.json`文件初始化数据库。首先，从`Client`获取`Database`和`Collection`实例。`Collection`实例将用于插入文档：

```go
func init() {
   recipes = make([]Recipe, 0)
   file, _ := ioutil.ReadFile("recipes.json")
   _ = json.Unmarshal([]byte(file), &recipes)
   ctx = context.Background()
   client, err = mongo.Connect(ctx, 
       options.Client().ApplyURI(os.Getenv("MONGO_URI")))
   if err = client.Ping(context.TODO(), 
           readpref.Primary()); err != nil {
       log.Fatal(err)
   }
   log.Println("Connected to MongoDB")
   var listOfRecipes []interface{}
   for _, recipe := range recipes {
       listOfRecipes = append(listOfRecipes, recipe)
   }
   collection := client.Database(os.Getenv(
       "MONGO_DATABASE")).Collection("recipes")
   insertManyResult, err := collection.InsertMany(
       ctx, listOfRecipes)
   if err != nil {
       log.Fatal(err)
   }
   log.Println("Inserted recipes: ", 
               len(insertManyResult.InsertedIDs))
}
```

上述代码读取一个 JSON 文件([`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter03/recipes.json`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter03/recipes.json))，其中包含食谱列表，并将其编码为`Recipe`结构的数组。然后，它与 MongoDB 服务器建立连接并将食谱插入到`recipes`集合中。

要一次性插入多个文档，我们可以使用`InsertMany()`方法。此方法接受一个接口切片作为参数。因此，我们必须将`Recipes`结构切片映射到接口切片。

重新运行应用程序，但这次，将`MONGO_URI`和`MONGO_DATABASE`变量设置如下：

```go
MONGO_URI="mongodb://USER:PASSWORD@localhost:27017/test?authSource=admin" MONGO_DATABASE=demo go run main.go
```

确保将`USER`替换为您的数据库用户，将`PASSWORD`替换为在部署 MongoDB 容器时创建的用户密码。

应用程序将被启动；首先执行`init()`方法，并将食谱项插入到 MongoDB 集合中：

![图 3.13 – 启动时插入食谱![图片 B17115_03_13.jpg](img/B17115_03_13.jpg)

图 3.13 – 启动时插入食谱

要验证数据是否已加载到食谱集合中，请刷新 MongoDB Compass。你应该看到你创建的条目：

![图 3.14 – 食谱集合![图片 B17115_03_14.jpg](img/B17115_03_14.jpg)

图 3.14 – 食谱集合

现在，`recipes`集合已经准备好了，我们需要更新每个 API 端点的代码，使它们使用集合而不是硬编码的食谱列表。但首先，我们需要更新`init()`方法以删除`recipes.json`文件的加载和编码：

```go
func init() {
   ctx = context.Background()
   client, err = mongo.Connect(ctx, 
       options.Client().ApplyURI(os.Getenv("MONGO_URI")))
   if err = client.Ping(context.TODO(), 
                        readpref.Primary()); err != nil {
       log.Fatal(err)
   }
   log.Println("Connected to MongoDB")
}
```

值得注意的是，您可以使用`mongoimport`实用程序将`recipe.json`文件直接加载到`recipes`集合中，而无需在 Golang 中编写任何代码。此命令如下：

```go
mongoimport --username admin --password password --authenticationDatabase admin --db demo --collection recipes --file recipes.json --jsonArray
```

此命令将 JSON 文件的内容导入到`recipes`集合中：

![图 3.15 – 使用 mongoimport 导入数据![图片 B17115_03_15.jpg](img/B17115_03_15.jpg)

图 3.15 – 使用 mongoimport 导入数据

在下一节中，我们将更新现有的函数处理程序，以便从`recipes`集合中读取和写入。

## 查找操作

要开始，我们需要实现一个返回食谱列表的函数。更新`ListRecipesHandler`，使其使用`Find()`方法从`recipes`集合中获取所有项目：

```go
func ListRecipesHandler(c *gin.Context) {
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
   c.JSON(http.StatusOK, recipes)
}
```

`Find()`方法返回一个游标，它是一系列文档的流。我们必须遍历文档流，并将每个文档一次解码到`Recipe`结构中。然后，我们必须将文档追加到食谱列表中。

运行应用程序，然后在`/recipes`端点发出 GET 请求；将在`recipes`集合上执行`find()`操作。结果，将返回一个食谱列表：

![图 3.16 – 获取所有食谱![图片 B17115_03_16.jpg](img/B17115_03_16.jpg)

图 3.16 – 获取所有食谱

端点正在工作，并从集合中检索食谱项。

## InsertOne 操作

需要实现的第二个函数将负责保存新食谱。更新 `NewRecipeHandler` 函数，使其在 `recipes` 集合上调用 `InsertOne()` 方法：

```go
func NewRecipeHandler(c *gin.Context) {
   var recipe Recipe
   if err := c.ShouldBindJSON(&recipe); err != nil {
       c.JSON(http.StatusBadRequest, gin.H{"error": 
           err.Error()})
       return
   }
   recipe.ID = primitive.NewObjectID()
   recipe.PublishedAt = time.Now()
   _, err = collection.InsertOne(ctx, recipe)
   if err != nil {
       fmt.Println(err)
       c.JSON(http.StatusInternalServerError, 
           gin.H{"error": "Error while inserting
                  a new recipe"})
       return
   }
   c.JSON(http.StatusOK, recipe)
}
```

在这里，我们在将项目保存到集合之前使用 `primitive.NewObjectID()` 方法设置一个唯一标识符。因此，我们需要更改 `Recipe` 结构体的 ID 类型。注意 `bson` 标签的使用，它将 `struct` 字段映射到 MongoDB 集合中的 `document` 属性：

```go
// swagger:parameters recipes newRecipe
type Recipe struct {
   //swagger:ignore
   ID primitive.ObjectID `json:"id" bson:"_id"`
   Name string `json:"name" bson:"name"`
   Tags []string `json:"tags" bson:"tags"`
   Ingredients []string `json:"ingredients" bson:"ingredients"`
   Instructions []string `json:"instructions"                           bson:"instructions"`
   PublishedAt time.Time `json:"publishedAt"                           bson:"publishedAt"`
}
```

注意

默认情况下，Go 在编码结构体值时将结构体字段名转换为小写。如果需要不同的名称，可以使用 `bson` 标签覆盖默认机制。

通过使用 Postman 客户端调用以下 POST 请求来插入新食谱：

![图 3.17 – 创建新食谱](img/B17115_03_17.jpg)

图 3.17 – 创建新食谱

验证食谱是否已插入到 MongoDB 集合中，如下截图所示：

![图 3.18 – 获取最后插入的食谱](img/B17115_03_18.jpg)

图 3.18 – 获取最后插入的食谱

要获取最后插入的食谱，我们使用 `sort()` 操作。

## UpdateOne 操作

最后，为了更新集合中的项目，更新 `UpdateRecipeHandler` 函数，使其调用 `UpdateOne()` 方法。此方法需要一个过滤器文档来匹配数据库中的文档，以及一个更新器文档来描述更新操作。您可以使用 `bson.D{}` – 一个 **二进制编码的 JSON** （**BSON**）文档来构建过滤器：

```go
func UpdateRecipeHandler(c *gin.Context) {
   id := c.Param("id")
   var recipe Recipe
   if err := c.ShouldBindJSON(&recipe); err != nil {
       c.JSON(http.StatusBadRequest, gin.H{"error":                                            err.Error()})
       return
   }
   objectId, _ := primitive.ObjectIDFromHex(id)
   _, err = collection.UpdateOne(ctx, bson.M{
       "_id": objectId,
   }, bson.D{{"$set", bson.D{
       {"name", recipe.Name},
       {"instructions", recipe.Instructions},
       {"ingredients", recipe.Ingredients},
       {"tags", recipe.Tags},
   }}})
   if err != nil {
       fmt.Println(err)
       c.JSON(http.StatusInternalServerError, 
           gin.H{"error": err.Error()})
       return
   }
   c.JSON(http.StatusOK, gin.H{"message": "Recipe 
                               has been updated"})
}
```

此方法通过其 Object ID 过滤文档。我们通过将 `ObjectIDFromHex` 应用到路由参数 ID 来获取 Object ID。这使用请求体中的新值更新匹配的食谱字段。

通过对现有食谱发出 PUT 请求来验证端点是否正常工作：

![图 3.19 – 更新食谱](img/B17115_03_19.jpg)

图 3.19 – 更新食谱

请求将匹配 `ID` 为 `600dcc85a65917cbd1f201b0` 的食谱，并将其 `name` 从 "`Homemade Pizza`" 更改为 "`Homemade Pepperoni Pizza`"，并将 `instructions` 字段更新为制作 "`Pepperoni Pizza`" 的额外步骤。

因此，食谱已成功更新。您可以使用 MongoDB Compass 确认这些更改：

![图 3.20 – UpdateOne 操作结果](img/B17115_03_20.jpg)

图 3.20 – UpdateOne 操作结果

您现在应该熟悉基本的 MongoDB 查询。继续实施剩余的 CRUD 操作。

最后，确保使用以下命令将更改推送到远程仓库：

```go
git checkout –b feature/mongo_integration
git add .
git commit –m "added mongodb integration"
git push origin feature/mongo_integration
```

然后，创建一个拉取请求以将 `feature` 分支合并到 `develop`：

![图 3.21 – 新的拉取请求](img/B17115_03_21.jpg)

图 3.21 – 新的拉取请求

注意

端点的完整实现可以在本书的 GitHub 仓库中找到（[`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter03/main.go`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter03/main.go)）。

你刚刚看到了如何将 MongoDB 集成到应用程序架构中。在下一节中，我们将介绍如何重构应用程序的源代码，使其在长期内可维护、可扩展和可扩展。

# 设计项目的布局

到目前为止，我们编写的所有代码都在`main.go`文件中。虽然这样没问题，但确保代码结构良好很重要；否则，当项目增长时，你将会有很多隐藏的依赖和混乱的代码（意大利面代码）。

我们将从数据模型开始。让我们创建一个`models`文件夹，以便我们可以存储所有的模型结构体。目前，我们有一个模型，即`Recipe`结构体。在`models`文件夹下创建一个`recipe.go`文件，并粘贴以下内容：

```go
package models
import (
   "time"
   "go.mongodb.org/mongo-driver/bson/primitive"
)
// swagger:parameters recipes newRecipe
type Recipe struct {
   //swagger:ignore
   ID           primitive.ObjectID `json:"id" bson:"_id"`
   Name         string             `json:"name" 
                                               bson:"name"`
   Tags         []string           `json:"tags" 
                                               bson:"tags"`
   Ingredients  []string           `json:"ingredients" 
                                      bson:"ingredients"`
   Instructions []string           `json:"instructions" 
                                      bson:"instructions"`
   PublishedAt  time.Time          `json:"publishedAt" 
                                      bson:"publishedAt"`
}
```

然后，创建一个包含`handler.go`文件的`handlers`文件夹。这个文件夹，正如其名所示，通过暴露每个 HTTP 请求要调用的正确函数来处理任何传入的 HTTP 请求：

```go
package handlers
import (
   "fmt"
   "net/http"
   "time"
   "github.com/gin-gonic/gin"
   "github.com/mlabouardy/recipes-api/models"
   "go.mongodb.org/mongo-driver/bson"
   "go.mongodb.org/mongo-driver/bson/primitive"
   "go.mongodb.org/mongo-driver/mongo"
   "golang.org/x/net/context"
)
type RecipesHandler struct {
   collection *mongo.Collection
   ctx        context.Context
}
func NewRecipesHandler(ctx context.Context, collection *mongo.Collection) *RecipesHandler {
   return &RecipesHandler{
       collection: collection,
       ctx:        ctx,
   }
}
```

这段代码创建了一个包含 MongoDB 集合和上下文实例的`RecipesHandler`结构体。在我们早期的简单实现中，我们倾向于在主包中全局保持这些变量。在这里，我们将这些变量保存在结构体中。接下来，我们必须定义一个`NewRecipesHandler`，这样我们就可以从`RecipesHandler`结构体创建一个实例。

现在，我们可以定义`RecipesHandler`类型的端点处理器。处理器可以访问结构体中的所有变量，例如数据库连接，因为它是`RecipesHandler`类型的一个方法：

```go
func (handler *RecipesHandler) ListRecipesHandler(c *gin.Context) {
   cur, err := handler.collection.Find(handler.ctx, bson.M{})
   if err != nil {
       c.JSON(http.StatusInternalServerError, 
           gin.H{"error": err.Error()})
       return
   }
   defer cur.Close(handler.ctx)

   recipes := make([]models.Recipe, 0)
   for cur.Next(handler.ctx) {
       var recipe models.Recipe
       cur.Decode(&recipe)
       recipes = append(recipes, recipe)
   }

   c.JSON(http.StatusOK, recipes)
}
```

从我们的`main.go`文件开始，我们将提供所有数据库凭据并连接到 MongoDB 服务器：

```go
package main
import (
   "context"
   "log"
   "os"
   "github.com/gin-gonic/gin"
   handlers "github.com/mlabouardy/recipes-api/handlers"
   "go.mongodb.org/mongo-driver/mongo"
   "go.mongodb.org/mongo-driver/mongo/options"
   "go.mongodb.org/mongo-driver/mongo/readpref"
)
```

然后，我们必须创建一个全局变量来访问端点处理器。更新`init()`方法，如下所示：

```go
var recipesHandler *handlers.RecipesHandler
func init() {
   ctx := context.Background()
   client, err := mongo.Connect(ctx, 
       options.Client().ApplyURI(os.Getenv("MONGO_URI")))
   if err = client.Ping(context.TODO(), 
           readpref.Primary()); err != nil {
       log.Fatal(err)
   }
   log.Println("Connected to MongoDB")
   collection := client.Database(os.Getenv(
       "MONGO_DATABASE")).Collection("recipes")
   recipesHandler = handlers.NewRecipesHandler(ctx, 
       collection)
}
```

最后，使用`recipesHandler`变量来访问每个 HTTP 端点的处理器：

```go
func main() {
   router := gin.Default()
   router.POST("/recipes", recipesHandler.NewRecipeHandler)
   router.GET("/recipes", 
       recipesHandler.ListRecipesHandler)
   router.PUT("/recipes/:id", 
       recipesHandler.UpdateRecipeHandler)
   router.Run()
}
```

运行应用程序。这次，运行当前目录下的所有`.go`文件：

```go
MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin" MONGO_DATABASE=demo go run *.go
```

应用程序将按预期工作。服务器日志如下：

![Figure 3.22 – Gin debug logs

![img/B17115_03_22.jpg]

图 3.22 – Gin 调试日志

现在，你的项目结构应该看起来像这样：

```go
.
├── go.mod
├── go.sum
├── handlers
│   └── handler.go
├── main.go
├── models
│   └── recipe.go
├── recipes.json
└── swagger.json
```

这是一个 Go 应用程序项目的基本布局。我们将在接下来的章节中介绍 Go 目录。

将更改推送到 GitHub 上的功能分支，并将其合并到`develop`分支：

```go
git checkout –b fix/code_refactoring
git add .
git commit –m "code refactoring"
git push origin fix/code_refactoring
```

当运行与数据库交互的服务时，其操作可能会成为瓶颈，从而降低用户体验并影响你的业务。这就是为什么响应时间是开发 RESTful API 时评估的最重要指标之一。

幸运的是，我们可以添加一个缓存层来存储频繁访问的数据在内存中，从而加快速度，减少对数据库的操作/查询次数。

# 使用 Redis 缓存 API

在本节中，我们将介绍如何为我们的 API 添加缓存机制。让我们设想一下，我们 MongoDB 数据库中有大量的食谱。每次我们尝试查询食谱列表时，都会遇到性能问题。我们可以做的是使用内存数据库，例如 Redis，来重用之前检索到的食谱，避免在每次请求中都击中 MongoDB 数据库。

由于 Redis 始终在 RAM 中，因此它在检索数据方面始终更快——这就是为什么它是缓存的一个优秀选择。另一方面，MongoDB 可能需要从磁盘检索数据以进行查询。

根据官方文档([`redis.io/`](https://redis.io/))，Redis 是一个开源的、分布式的、内存中的键值数据库、缓存和消息代理。以下图表说明了 Redis 如何融入我们的 API 架构：

![图 3.23 – API 新架构](img/B17115_03_23.jpg)

图 3.23 – API 新架构

假设我们想要获取一个食谱列表。首先，API 将在 Redis 中查找。如果存在食谱列表，它将被返回（这被称为`find({})`查询，并将结果返回并保存在缓存中，以供未来的请求使用。

## 在 Docker 中运行 Redis

设置 Redis 最简单的方法是通过 Docker。我们将使用 DockerHub 上可用的 Redis 官方镜像来完成此操作([`hub.docker.com/_/redis`](https://hub.docker.com/_/redis))。在撰写本书时，最新稳定版本是 6.0。基于该镜像运行容器：

```go
docker run -d --name redis -p 6379:6379 redis:6.0
```

此命令执行以下两个主要操作：

+   `-d`标志以守护进程方式运行 Redis 容器。

+   `-p`标志将容器的 6379 端口映射到主机的 6379 端口。6379 是 Redis 服务器暴露的端口。

命令的输出如下：

![图 3.24 – 从 DockerHub 拉取 Redis 镜像](img/B17115_03_24.jpg)

图 3.24 – 从 DockerHub 拉取 Redis 镜像

总是检查 Docker 日志以查看事件链：

```go
docker logs –f CONTAINER_ID
```

日志提供了大量有用的信息，例如默认配置和暴露的服务器端口：

![图 3.25 – Redis 服务器日志](img/B17115_03_25.jpg)

图 3.25 – Redis 服务器日志

Redis 容器使用基本的缓存策略。对于生产使用，建议配置驱逐策略。您可以使用`redis.conf`文件配置策略：

```go
maxmemory-policy allkeys-lru
maxmemory 512mb
```

此配置为 Redis 分配了 512 MB 的内存，并将驱逐策略设置为**最近最少使用**（**LRU**）算法，该算法删除最不常使用的缓存项。因此，我们只保留有最高再次读取概率的项。

您可以使用以下命令在容器的运行时传递配置：

```go
docker run -d -v $PWD/conf:/usr/local/etc/redis --name redis -p 6379:6379 redis:6.0
```

这里，`$PWD/conf` 是包含 `redis.conf` 文件的文件夹。

现在，Redis 已启动，我们可以使用它来缓存 API 数据。但首先，让我们通过执行以下命令来安装官方的 Redis Go 驱动程序 ([`github.com/go-redis/redis`](https://github.com/go-redis/redis))：

```go
go get github.com/go-redis/redis/v8
```

在 `main.go` 文件中导入以下包：

```go
import "github.com/go-redis/redis"
```

现在，在 `init()` 方法中，使用 `redis.NewClient()` 初始化 Redis 客户端。此方法接受服务器地址、密码和数据库作为参数。接下来，我们将在 Redis 客户端上调用 `Ping()` 方法以检查与 Redis 服务器的连接状态：

```go
redisClient := redis.NewClient(&redis.Options{
       Addr:     "localhost:6379",
       Password: "",
       DB:       0,
})
status := redisClient.Ping()
fmt.Println(status)
```

此代码将在部署后设置与 Redis 服务器的连接：

![图 3.26 – 检查与 Redis 服务器的连接](img/B17115_03_26.jpg)

图 3.26 – 检查与 Redis 服务器的连接

如果连接成功，将显示 `ping: PONG` 消息，如前面的截图所示。

## 优化 MongoDB 查询

与 Redis 服务器建立连接后，我们可以更新 `RecipesHandler` 结构体以存储 Redis 客户端的实例，以便处理程序可以与 Redis 交互：

```go
type RecipesHandler struct {
   collection  *mongo.Collection
   ctx         context.Context
   redisClient *redis.Client
}
func NewRecipesHandler(ctx context.Context, collection 
    *mongo.Collection, redisClient *redis.Client) 
     *RecipesHandler {
   return &RecipesHandler{
       collection:  collection,
       ctx:         ctx,
       redisClient: redisClient,
   }
}
```

确保在 `init()` 方法中将 Redis 客户端实例传递给 `RecipesHandler` 实例：

```go
recipesHandler = handlers.NewRecipesHandler(ctx, collection,        	                                            redisClient)
```

接下来，我们必须更新 `ListRecipesHandler` 以检查食谱是否已缓存在 Redis 中。如果是，我们返回一个列表。如果不是，我们将从 MongoDB 获取数据并将其缓存在 Redis 中。我们必须对代码进行的新的更改如下：

```go
func (handler *RecipesHandler) ListRecipesHandler(c       *gin.Context) {
   val, err := handler.redisClient.Get("recipes").Result()
   if err == redis.Nil {
       log.Printf("Request to MongoDB")
       cur, err := handler.collection.Find(handler.ctx, 
                                           bson.M{})
       if err != nil {
           c.JSON(http.StatusInternalServerError, 
                  gin.H{"error": err.Error()})
           return
       }
       defer cur.Close(handler.ctx)
       recipes := make([]models.Recipe, 0)
       for cur.Next(handler.ctx) {
           var recipe models.Recipe
           cur.Decode(&recipe)
           recipes = append(recipes, recipe)
       }
       data, _ := json.Marshal(recipes)
       handler.redisClient.Set("recipes", string(data), 0)
       c.JSON(http.StatusOK, recipes)
   } else if err != nil {
       c.JSON(http.StatusInternalServerError, 
              gin.H{"error": err.Error()})
       return
   } else {
       log.Printf("Request to Redis")
       recipes := make([]models.Recipe, 0)
       json.Unmarshal([]byte(val), &recipes)
       c.JSON(http.StatusOK, recipes)
   }
}
```

值得注意的是，Redis 的值必须是一个字符串，因此我们必须使用 `json.Marshal()` 方法将 `recipes` 切片编码成字符串。

要测试新的更改，运行应用程序。然后，使用 Postman 客户端或 `cURL` 命令在 `/recipes` 端点发出 GET 请求。切换回你的终端并查看 Gin 日志。你应该在控制台中看到一条消息，对应于从 MongoDB 获取数据的第一次请求：

![图 3.27 – 从 MongoDB 获取数据](img/B17115_03_27.jpg)

图 3.27 – 从 MongoDB 获取数据

注意

有关如何使用 Postman 客户端或 `cURL` 命令的逐步指南，请参阅*第一章*，*使用 Gin 入门*。

如果你发出第二次 HTTP 请求，这次，数据将从 Redis 返回，因为它在第一次请求中被缓存：

![图 3.28 – 从 Redis 获取数据](img/B17115_03_28.jpg)

图 3.28 – 从 Redis 获取数据

如我们所见，从内存（Redis）中检索数据比从磁盘（MongoDB）中检索数据快得多。

我们可以通过在容器中运行 Redis CLI 来验证数据是否被缓存在 Redis 中。运行以下命令：

```go
docker ps
docker exec –it CONTAINER_ID bash
```

这些命令将通过交互式终端连接到 Redis 容器并启动 bash shell。你会注意到你现在正在使用你的终端，就像你身处容器内部一样，如下面的截图所示：

![图 3.29 – 在 Redis 容器内运行交互式会话

![图片 B17115_03_29.jpg]

图 3.29 – 在 Redis 容器内运行交互式会话

现在我们已连接到 Redis 容器，我们可以使用 Redis 命令行：

```go
redis-cli
```

从那里，我们可以使用 `EXISTS` 命令来检查 `recipes` 键是否存在：

```go
EXISTS recipes
```

此命令将返回 `1`（如果键存在）或 `0`（如果键不存在）。在我们的例子中，配方列表已被缓存在 Redis 中：

![图 3.30 – 检查 Redis 中是否存在键

![图片 B17115_03_30.jpg]

图 3.30 – 检查 Redis 中是否存在键

您可以使用 shell 客户端完成很多事情，但您已经了解了基本概念。输入 `exit` 退出 MongoDB shell，然后再次输入 `exit` 退出交互式 shell。

对于 GUI 粉丝，您可以使用 Redis Insights ([`redislabs.com/fr/redis-enterprise/redis-insight/`](https://redislabs.com/fr/redis-enterprise/redis-insight/))。它提供了一个直观的界面来探索 Redis 并与其数据交互。类似于 Redis 服务器，您可以使用 Docker 部署 Redis Insights：

```go
docker run -d --name redisinsight --link redis -p 8001:8001 redislabs/redisinsight
```

此命令将基于 Redis Insight 官方镜像运行容器，并在端口 8001 上公开接口。

使用浏览器导航到 `http://localhost:8081`。Redis Insights 主页应该会出现。点击 **我已经有一个数据库** 然后点击 **连接到 Redis 数据库** 按钮：

![图 3.31 – 配置 Redis 数据库

![图片 B17115_03_31.jpg]

图 3.31 – 配置 Redis 数据库

设置 `redis`、`6379` 并命名数据库。设置如下：

![图 3.32 – 新的 Redis 设置

![图片 B17115_03_32.jpg]

图 3.32 – 新的 Redis 设置

然后，点击 **添加 Redis 数据库**。**本地**数据库将被保存；点击它：

![图 3.33 – Redis Insights 数据库

![图片 B17115_03_33.jpg]

图 3.33 – Redis Insights 数据库

您将被重定向到 **摘要** 页面，其中包含关于 Redis 服务器的真实指标和统计数据：

![图 3.34 – Redis 服务器指标

![图片 B17115_03_34.jpg]

图 3.34 – Redis 服务器指标

如果您点击 **浏览**，您将看到存储在 Redis 中的所有键的列表。如图所示，recipes 键已被缓存：

![图 3.35 – Redis 键列表

![图片 B17115_03_35.jpg]

图 3.35 – Redis 键列表

现在，您可以使用该界面在 Redis 中探索、操作和可视化数据。

到目前为止，我们构建的 API 运行得很好，对吧？其实不然；想象一下，您向数据库中添加了一个新的配方：

![图 3.36 – 创建新配方

![图片 B17115_03_36.jpg]

图 3.36 – 创建新配方

现在，如果您发出 `GET /recipes` 请求，将找不到新的配方。这是因为数据是从缓存中返回的：

![图 3.37 – 没有找到配方

![图片 B17115_03_37.jpg]

图 3.37 – 没有找到配方

缓存引入的一个问题是，当数据发生变化时，如何保持缓存更新：

![图 3.38 – 所有后续请求都命中 Redis

![img/B17115_03_38.jpg]

图 3.38 – 所有后续请求都命中 Redis

在这种情况下，有两个分组规则来修复不一致性。首先，我们可以在 Redis 中为菜谱键添加一个**生存时间**（**TTL**）字段。其次，每次插入或更新新的菜谱时，我们可以在 Redis 中清除**菜谱**键。

注意

缓存的 TTL 保留时间取决于你的应用程序逻辑。你可能需要保存一个小时或几天，具体取决于数据更新的频率。

我们可以通过更新`NewRecipeHandler`函数来实现第二个解决方案，使其在插入新菜谱时删除`recipes`键。在这种情况下，实现方式如下：

```go
func (handler *RecipesHandler) NewRecipeHandler(c *gin.Context) {
   var recipe models.Recipe
   if err := c.ShouldBindJSON(&recipe); err != nil {
       c.JSON(http.StatusBadRequest, 
              gin.H{"error":err.Error()})
       return
   }
   recipe.ID = primitive.NewObjectID()
   recipe.PublishedAt = time.Now()
   _, err := handler.collection.InsertOne(handler.ctx, 
                                          recipe)
   if err != nil {
       c.JSON(http.StatusInternalServerError, 
       gin.H{"error": "Error while inserting 
             a new recipe"})
       return
   }
   log.Println("Remove data from Redis")
   handler.redisClient.Del("recipes")
   c.JSON(http.StatusOK, recipe)
}
```

重新部署应用程序。现在，如果你发起一个`GET /recipes`请求，数据将按预期从 MongoDB 返回；然后，它将被缓存在 Redis 中。第二个 GET 请求将返回 Redis 中的数据。然而，现在，如果我们发起一个`POST /recipes`请求来插入一个新的菜谱，Redis 中的`recipes`键将被清除，如`从 Redis 中删除数据`消息所确认的。这意味着下一个`GET /recipes`请求将从 MongoDB 获取数据：

![图 3.39 – 插入请求时清除缓存

![img/B17115_03_39.jpg]

图 3.39 – 插入请求时清除缓存

现在，新的菜谱将出现在菜谱列表中：

![图 3.40 – 新插入的菜谱

![img/B17115_03_40.jpg]

图 3.40 – 新插入的菜谱

注意

更新`UpdateRecipeHandler`，以便在`/recipes/{id}`端点发生 PUT 请求时清除缓存。

虽然缓存为具有大量读取的应用程序提供了巨大的好处，但它可能对执行大量数据库更新的应用程序没有同样的好处，并且可能会减慢写入速度。

## 性能基准测试

我们可以更进一步，看看 API 在大量请求下的表现。我们可以使用 Apache Benchmark（[`httpd.apache.org/docs/2.4/programs/ab.html`](https://httpd.apache.org/docs/2.4/programs/ab.html)）模拟多个请求。

首先，让我们测试没有缓存层的 API。你可以使用以下命令在`/recipes`端点上运行总共 2,000 个 GET 请求，有 100 个并发请求：

```go
ab -n 2000 -c 100 -g without-cache.data http://localhost:8080/recipes
```

完成所有请求可能需要几分钟。一旦完成，你应该看到以下结果：

![图 3.41 – 没有缓存层的 API

![img/B17115_03_41.jpg]

图 3.41 – 没有缓存层的 API

从这个输出中可以得出的重要信息如下：

+   **测试所用时间**：这意味着完成 2,000 个请求的总时间。

+   **每个请求的时间**：这意味着完成一个请求需要多少毫秒。

接下来，我们将发起相同的请求，但这次是在带有缓存的 API（Redis）上：

```go
ab -n 2000 -c 100 -g with-cache.data http://localhost:8080/recipes
```

完成对 2,000 个请求的处理应该需要几秒钟：

![图 3.42 – 带有缓存层的 API

![img/B17115_03_42.jpg]

图 3.42 – 带有缓存层的 API

要比较这两个结果，我们可以使用`gnuplot`实用程序根据`without-cache.data`和`with-cache.data`文件绘制图表。但首先，创建一个`apache-benchmark.p`文件以将数据渲染成图表：

```go
set terminal png
set output "benchmark.png"
set title "Cache benchmark"
set size 1,0.7
set grid y
set xlabel "request"
set ylabel "response time (ms)"
plot "with-cache.data" using 9 smooth sbezier with lines title "with cache", "without-cache.data" using 9 smooth sbezier with lines title "without cache"
```

这些命令将在同一图表上基于`.data`文件绘制两个图表，并将输出保存为 PNG 图像。接下来，运行`gnuplot`命令创建图像：

```go
gnuplot apache-benchmark.p
```

将创建一个`benchmark.png`图像，如下所示：

![图 3.43 – 带缓存和不带缓存的 API 基准测试](img/B17115_03_43.jpg)

图 3.43 – 带缓存和不带缓存的 API 基准测试

启用缓存机制的 API 响应时间与不带缓存的 API 响应时间相比非常快。

确保使用功能分支将更改推送到 GitHub。然后，创建一个拉取请求以合并到`develop`分支：

```go
git checkout –b feature/redis_integration
git add .
git commit –m "added redis integration"
git push origin feature/redis_integration
```

到本章结束时，你的 GitHub 仓库应该看起来像这样：

![图 3.44 – 项目的 GitHub 仓库](img/B17115_03_44.jpg)

图 3.44 – 项目的 GitHub 仓库

太好了！现在，你应该能够将 MongoDB 数据库集成到你的 API 架构中，以管理数据持久性。

# 摘要

在本章中，我们学习了如何构建一个利用 Gin 框架和 Go 驱动程序在 NoSQL 数据库（如 MongoDB）中创建查询和查询的 RESTful API。

我们还探讨了如何通过使用 Redis 缓存它访问的数据来加快 API 的速度。如果你的数据主要是静态的且不经常变化，这绝对是你应用程序的一个很好的补充。最后，我们介绍了如何使用 Apache Benchmark 进行性能基准测试。

我们迄今为止构建的 RESTful API 运行得非常顺畅，并且对公众开放（如果部署在远程服务器上）。如果你不验证 API，那么任何人都可以访问任何端点，这可能会非常不理想，因为用户可能会损坏你的数据。更糟糕的是，你可能会将数据库中的敏感信息暴露给整个互联网。这就是为什么在下一章中，我们将介绍如何通过身份验证来保护 API，例如使用 JWT。

# 问题

1.  当发生`DELETE`请求时，实现一个删除食谱操作。

1.  使用`FindOne`操作实现一个 GET /`recipes/{id}`端点。

1.  JSON 文档在 MongoDB 中是如何存储的？

1.  Redis 中的 LRU 驱逐策略是如何工作的？

# 进一步阅读

+   *《MongoDB 基础》*，作者：Amit Phaltankar，Juned Ahsan，Michael Harrison 和 Liviu Nedov，Packt 出版社

+   *《学习 MongoDB 4.x》*，作者：Doug Bierer，Packt 出版社

+   *《使用 Go 的实战 RESTful Web 服务 – 第二版》*，作者：Naren Yellavula，Packt 出版社
