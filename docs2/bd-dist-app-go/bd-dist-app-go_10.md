# 第七章：测试 Gin HTTP 路由

在本章中，您将学习如何测试基于 Gin 的 Web 应用程序，这涉及到运行 Go 单元和集成测试。在这个过程中，我们将探讨如何集成外部工具来识别 Gin Web 应用程序中的潜在安全漏洞。最后，我们将介绍如何使用 Postman 集合运行器功能测试 **API** **HTTP** 方法。

因此，我们将涵盖以下主题：

+   测试 Gin HTTP 处理器

+   生成代码覆盖率报告

+   发现安全漏洞

+   运行 Postman 集合

到本章结束时，您应该能够从头开始编写、执行和自动化 Gin Web 应用程序的测试。

# 技术要求

要遵循本章中的说明，您需要以下内容：

+   完全理解前一章的内容——本章是前一章的后续，它将使用相同的源代码。因此，一些代码片段将不会进行解释，以避免重复。

+   使用 Go 测试包的先前经验。

本章的代码包托管在 GitHub 上，网址为 [`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter07`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter07)。

# 测试 Gin HTTP 处理器

到目前为止，我们已经学习了如何使用 Gin 框架设计、构建和扩展分布式 Web 应用程序。在本章中，我们将介绍如何集成不同类型的测试以消除发布时可能出现的错误。我们将从 **单元测试** 开始。

注意

值得注意的是，您需要事先采用 **测试驱动开发**（**TDD**）的方法，以便在编写可测试的代码方面取得领先。

为了说明如何为 Gin Web 应用程序编写单元测试，您需要直接进入一个基本示例。让我们以 *第二章* 中涵盖的“设置 API 端点”为例，即 `hello world` 示例。路由声明和 HTTP 服务器设置已从 `main` 函数中提取出来，以便为测试做准备，如下面的代码片段所示：

```go
package main
import (
   "net/http"
   "github.com/gin-gonic/gin"
)
func IndexHandler(c *gin.Context) {
   c.JSON(http.StatusOK, gin.H{
       "message": "hello world",
   })
}
func SetupServer() *gin.Engine {
   r := gin.Default()
   r.GET("/", IndexHandler)
   return r
}
func main() {
   SetupServer().Run()
}
```

运行应用程序，然后在 `localhost:8080` 上发出一个 `GET` 请求。将返回一个 `hello world` 消息，如下所示：

```go
curl localhost:8080
{"message":"hello world"}
```

在重构完成后，用 Go 编程语言编写一个单元测试。为此，请执行以下步骤：

1.  在同一项目目录中定义一个 `main_test.go` 文件，并包含以下代码。我们之前重构的 `SetupServer()` 方法被注入到一个测试服务器中：

    ```go
    package main
    func TestIndexHandler(t *testing.T) {
       mockUserResp := `{"message":"hello world"}`
       ts := httptest.NewServer(SetupServer())
       defer ts.Close()
       resp, err := http.Get(fmt.Sprintf("%s/", ts.URL))
       if err != nil {
           t.Fatalf("Expected no error, got %v", err)
       }
       defer resp.Body.Close()
       if resp.StatusCode != http.StatusOK {
           t.Fatalf("Expected status code 200, got %v", 
                    resp.StatusCode)
       }
       responseData, _ := ioutil.ReadAll(resp.Body)
       if string(responseData) != mockUserResp {
           t.Fatalf("Expected hello world message, got %v", 
                     responseData)
       }
    }
    ```

    每个测试方法都必须以 `Test` 前缀开始——例如，`TestXYZ` 将是一个有效的测试。前面的代码使用 Gin 引擎设置了一个测试服务器并发出一个 `GET` 请求。然后，它检查状态码和响应负载。如果实际结果与预期结果不匹配，将抛出一个错误。因此，测试将失败。

1.  要在 Golang 中运行测试，请执行以下命令：

    ```go
    go test
    ```

    如下所示的屏幕截图所示，测试将成功执行：

![Figure 7.1 – Test execution![img/Figure_7.1_B17115.jpg](img/Figure_7.1_B17115.jpg)

图 7.1 – 测试执行

虽然你可以使用测试包编写完整的测试，但你也可以安装一个第三方包，如 **testify**，以使用高级断言。为此，请按照以下步骤操作：

1.  使用以下命令下载 testify：

    ```go
    Go get github.com/stretchr/testify
    ```

1.  接下来，更新 `TestIndexHandler` 以使用 testify 包的 `assert` 属性对响应的正确性进行一些断言，如下所示：

    ```go
    func TestIndexHandler(t *testing.T) {
       mockUserResp := `{"message":"hello world"}`
       ts := httptest.NewServer(SetupServer())
       defer ts.Close()
       resp, err := http.Get(fmt.Sprintf("%s/", ts.URL))
       defer resp.Body.Close()
       assert.Nil(t, err)
       assert.Equal(t, http.StatusOK, resp.StatusCode)
       responseData, _ := ioutil.ReadAll(resp.Body)
       assert.Equal(t, mockUserResp, string(responseData))
    }
    ```

1.  执行 `go test` 命令，你将得到相同的结果。

这就是为 Gin 网络应用程序编写测试的方法。

让我们继续前进，为之前章节中涵盖的 RESTful API 的 HTTP 处理器编写单元测试。作为提醒，以下架构图说明了 REST API 提供的操作：

![Figure 7.2 – API HTTP 方法![img/Figure_7.2_B17115.jpg](img/Figure_7.2_B17115.jpg)

图 7.2 – API HTTP 方法

注意

API 源代码位于 GitHub 仓库中的 `chapter07` 文件夹下。建议基于仓库中可用的源代码开始本章的学习。

图像中的操作已在 Gin 默认路由器中注册，并分配给不同的 HTTP 处理器，如下所示：

```go
func main() {
   router := gin.Default()
   router.POST("/recipes", NewRecipeHandler)
   router.GET("/recipes", ListRecipesHandler)
   router.PUT("/recipes/:id", UpdateRecipeHandler)
   router.DELETE("/recipes/:id", DeleteRecipeHandler)
   router.GET("/recipes/:id", GetRecipeHandler)
   router.Run()
}
```

从一个 `main_test.go` 文件开始，定义一个方法来返回 Gin 路由的实例。然后，为每个 HTTP 处理器编写一个测试方法。例如，`TestListRecipesHandler` 处理器在下面的代码片段中显示：

```go
func SetupRouter() *gin.Engine {
   router := gin.Default()
   return router
}
func TestListRecipesHandler(t *testing.T) {
   r := SetupRouter()
   r.GET("/recipes", ListRecipesHandler)
   req, _ := http.NewRequest("GET", "/recipes", nil)
   w := httptest.NewRecorder()
   r.ServeHTTP(w, req)
   var recipes []Recipe
   json.Unmarshal([]byte(w.Body.String()), &recipes)
   assert.Equal(t, http.StatusOK, w.Code)
   assert.Equal(t, 492, len(recipes))
}
```

它在 `GET /recipes` 资源上注册了 `ListRecipesHandler` 处理器，然后发出一个 `GET` 请求。请求有效载荷随后被编码到 `recipes` 切片中。如果食谱的数量等于 `492` 且状态码为 `200-OK` 响应，则测试被认为是成功的。否则，将抛出错误，测试将失败。

然后，发出一个 `go test` 命令，但这次，禁用 Gin 调试日志并使用 `-v` 标志启用详细模式，如下所示：

```go
GIN_MODE=release go test -v
```

命令输出如下所示：

![Figure 7.3 – 运行测试与详细输出![img/Figure_7.3_B17115.jpg](img/Figure_7.3_B17115.jpg)

图 7.3 – 以详细输出运行测试

注意

在 *第十章**，捕获 Gin 应用程序指标* 中，我们将介绍如何自定义 Gin 调试日志以及如何将它们发送到集中式日志平台。

类似地，为 `NewRecipeHandler` 处理器编写一个测试。它将简单地发布一个新的食谱并检查返回的响应代码是否为 `200-OK` 状态。`TestNewRecipeHandler` 方法在下面的代码片段中显示：

```go
func TestNewRecipeHandler(t *testing.T) {
   r := SetupRouter()
   r.POST("/recipes", NewRecipeHandler)
   recipe := Recipe{
       Name: "New York Pizza",
   }
   jsonValue, _ := json.Marshal(recipe)
   req, _ := http.NewRequest("POST", "/recipes", 
                              bytes.NewBuffer(jsonValue))
   w := httptest.NewRecorder()
   r.ServeHTTP(w, req)
   assert.Equal(t, http.StatusOK, w.Code)
}
```

在前面的测试方法中，你使用 `Recipe` 结构体声明了一个食谱。然后，该结构体被序列化为 `NewRequest` 函数。

执行测试，`TestListRecipesHandler` 和 `TestNewRecipeHandler` 都应该成功，如下所示：

![Figure 7.4 – 运行多个测试![img/Figure_7.4_B17115.jpg](img/Figure_7.4_B17115.jpg)

图 7.4 – 运行多个测试

您现在已经熟悉了为 Gin HTTP 处理器编写单元测试。接下来，继续编写其余 API 端点的测试。

# 生成代码覆盖率报告

在本节中，我们将介绍如何使用 Go 生成**覆盖率报告**。测试覆盖率描述了通过运行包的测试来执行包中代码的程度。

运行以下命令以生成一个文件，该文件包含有关您在上一节中编写的测试覆盖了多少代码的统计信息：

```go
GIN_MODE=release go test -v -coverprofile=coverage.out ./...
```

命令将运行测试并显示测试覆盖的语句百分比。在以下示例中，我们覆盖了 16.9%的语句：

![图 7.5 – 测试覆盖率![图片](img/Figure_7.5_B17115.jpg)

图 7.5 – 测试覆盖率

生成的`coverage.out`文件包含单元测试覆盖的行数。为了简洁，已裁剪完整代码，但您可以在以下示例中看到说明：

```go
mode: set
/Users/mlabouardy/github/Building-Distributed-Applications-in-Gin/chapter7/api-without-db/main.go:51.41,53.2 1 1
/Users/mlabouardy/github/Building-Distributed-Applications-in-Gin/chapter7/api-without-db/main.go:65.39,67.50 2 1
/Users/mlabouardy/github/Building-Distributed-Applications-in-Gin/chapter7/api-without-db/main.go:72.2,77.31 4 1
/Users/mlabouardy/github/Building-Distributed-Applications-in-Gin/chapter7/api-without-db/main.go:67.50,70.3 2 0
/Users/mlabouardy/github/Building-Distributed-Applications-in-Gin/chapter7/api-without-db/main.go:98.42,101.50 3 0
/Users/mlabouardy/github/Building-Distributed-Applications-in-Gin/chapter7/api-without-db/main.go:106.2,107.36 2 0
```

您可以使用`go tool`命令可视化代码覆盖率，如下所示：

```go
go tool cover -html=coverage.out
```

命令将在您的默认浏览器中打开 HTML 演示文稿，显示绿色覆盖的源代码和红色未覆盖的代码，如下截图所示：

![图 7.6 – 查看结果![图片](img/Figure_7.6_B17115.jpg)

图 7.6 – 查看结果

现在更容易发现哪些方法被测试覆盖了，让我们为负责更新现有配方的 HTTP 处理器编写一个额外的测试。为此，请按照以下步骤操作：

1.  将以下代码块添加到`main_test.go`文件中：

    ```go
    func TestUpdateRecipeHandler(t *testing.T) {
       r := SetupRouter()
       r.PUT("/recipes/:id", UpdateRecipeHandler)
       recipe := Recipe{
           ID:   "c0283p3d0cvuglq85lpg",
           Name: "Gnocchi",
           Ingredients: []string{
               "5 large Idaho potatoes",
               "2 egges",
               "3/4 cup grated Parmesan",
               "3 1/2 cup all-purpose flour",
           },
       }
       jsonValue, _ := json.Marshal(recipe)
       reqFound, _ := http.NewRequest("PUT", 
          "/recipes/"+recipe.ID, bytes.NewBuffer(jsonValue))
       w := httptest.NewRecorder()
       r.ServeHTTP(w, reqFound)
       assert.Equal(t, http.StatusOK, w.Code)
       reqNotFound, _ := http.NewRequest("PUT", "/recipes/1", 
          bytes.NewBuffer(jsonValue))
       w = httptest.NewRecorder()
       r.ServeHTTP(w, reqNotFound)
       assert.Equal(t, http.StatusNotFound, w.Code)
    }
    ```

    代码发出两个 HTTP `PUT`请求。

    其中一个具有有效的配方 ID，并检查 HTTP 响应代码（`200-OK`）。

    另一个具有无效的配方 ID，并检查 HTTP 响应代码（`404-Not found`）。

1.  重新执行测试，覆盖率百分比应从 16.9%增加到 39.0%。以下输出证实了这一点：

![图 7.7 – 更多代码覆盖率![图片](img/Figure_7.7_B17115.jpg)

图 7.7 – 更多代码覆盖率

太棒了！您现在能够运行单元测试并获取代码覆盖率报告。所以，继续前进，测试并覆盖。

虽然单元测试是软件开发的重要部分，但同样重要的是，您编写的代码不仅要在隔离状态下进行测试。集成和端到端测试通过测试应用程序的各个部分来提供额外的信心。这些部分可能单独工作得很好，但在大型系统中，代码单元很少单独工作。这就是为什么在下一节中，我们将介绍如何编写和运行集成测试。

## 使用 Docker 执行集成测试

**集成测试**的目的是验证分离开发的组件是否能够正确地协同工作。与单元测试不同，集成测试可以依赖于数据库和外部服务。

到目前为止编写的分布式 Web 应用程序与外部服务 MongoDB 和 Reddit 进行交互，如下截图所示：

![图 7.8 – 分布式 Web 应用程序![图片](img/Figure_7.8_B17115.jpg)

图 7.8 – 分布式 Web 应用程序

要开始进行集成测试，请按照以下步骤操作：

1.  使用 Docker Compose 运行我们集成测试所需的服务。以下 `docker-compose.yml` 文件将启动 MongoDB 和 Redis 容器：

    ```go
    version: "3.9"
    services:
     redis:
       image: redis
       ports:
         - 6379:6379
     mongodb:
       image: mongo:4.4.3
       ports:
         - 27017:27017
       environment:
         - MONGO_INITDB_ROOT_USERNAME=admin
         - MONGO_INITDB_ROOT_PASSWORD=password
    ```

1.  现在，测试 RESTful API 暴露的每个端点。例如，要测试列出所有菜谱的端点，我们可以使用以下代码块：

    ```go
    func TestListRecipesHandler(t *testing.T) {
       ts := httptest.NewServer(SetupRouter())
       defer ts.Close()
       resp, err := http.Get(fmt.Sprintf("%s/recipes", 	 	                                     ts.URL))
       defer resp.Body.Close()
       assert.Nil(t, err)
       assert.Equal(t, http.StatusOK, resp.StatusCode)
       data, _ := ioutil.ReadAll(resp.Body)
       var recipes []models.Recipe
       json.Unmarshal(data, &recipes)
       assert.Equal(t, len(recipes), 10)
    }
    ```

1.  要运行测试，请提供 MongoDB `go test` 命令，如下所示：

    ```go
    MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin&readPreference=primary&ssl=false" MONGO_DATABASE=demo REDIS_URI=localhost:6379 go test
    ```

太好了！测试将成功通过，如下所示：

![图 7.9 – 运行集成测试]

![图片 7.9](img/Figure_7.9_B17115.jpg)

![图 7.9 – 运行集成测试]

测试向 `/recipes` 端点发出 `GET` 请求，并验证端点返回的菜谱数量是否等于 10。

另一个重要但常被忽视的测试是 **安全测试**。确保您的应用程序没有主要安全漏洞是强制性的，否则数据泄露和数据泄露的风险很高。

# 发现安全漏洞

有许多工具可以帮助您识别 Gin Web 应用程序中的主要安全漏洞。在本节中，我们将介绍两个工具，这些工具是您在构建 Gin 应用程序时可以采用的几个工具之一：**Snyk** 和 **Golang 安全检查器**（**Gosec**）。

在接下来的章节中，我们将展示如何使用这些工具来检查 Gin 应用程序中的安全漏洞。

## Gosec

**Gosec** 是一个用 Golang 编写的工具，通过扫描 Go **抽象语法树**（**AST**）来检查源代码中的安全问题。在我们检查 Gin 应用程序代码之前，我们需要安装 Gosec 二进制文件。

可以使用以下 cURL 命令下载二进制文件。这里使用的是版本 2.7.0：

```go
curl -sfL https://raw.githubusercontent.com/securego/gosec/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v2.7.0
```

一旦安装了命令，请在项目文件夹中运行以下命令。`./...` 参数设置为递归扫描所有 Go 包：

```go
gosec ./...
```

命令将识别与未处理错误相关的三个主要问题（**常见弱点枚举**（**CWE**）*703*（https://cwe.mitre.org/data/definitions/703.html），如下截图所示：

![图 7.10 – 未处理的错误]

![图片 7.10](img/Figure_7.10_B17115.jpg)

![图 7.10 – 未处理的错误]

默认情况下，Gosec 将扫描您的项目并对其规则进行验证。但是，您可以排除一些规则。例如，要排除负责未处理错误问题的规则，请发出以下命令：

```go
gosec -exclude=G104 ./...
```

注意

可用规则的完整列表可以在以下位置找到：

[`github.com/securego/gosec#available-rules`](https://github.com/securego/gosec#available-rules)

命令输出如下所示：

![图 7.11 – 排除 Gosec 规则]

![图片 7.11](img/Figure_7.11_B17115.jpg)

![图 7.11 – 排除 Gosec 规则]

现在，你应该能够扫描应用程序源代码以查找潜在的安全漏洞或错误来源。

## 使用 Snyk 保护 Go 模块

另一种检测潜在安全漏洞的方法是扫描 Go 模块。`go.mod` 文件包含了 Gin 网络应用程序所使用的所有依赖项。**Snyk** ([`snyk.io`](https://snyk.io)) 是一种用于识别和修复 Go 应用程序中安全漏洞的 **软件即服务** (**SaaS**) 解决方案。

注意

Snyk 支持包括 Java、Python、Node.js、Ruby、Scala 等在内的所有主要编程语言。

解决方案相当简单。要开始，请按照以下步骤操作：

1.  使用您的 GitHub 账户登录创建一个免费账户。

1.  然后，使用 **Node 包管理器** (**npm**) 安装 Snyk 官方 **命令行界面** (**CLI**)，如下所示：

    ```go
    npm install -g snyk
    ```

1.  接下来，通过运行以下命令将您的 Snyk 账户与 CLI 关联：

    ```go
    snyk auth
    ```

    上述命令将打开一个浏览器标签页，并重定向您使用 Snyk 账户验证 CLI。

1.  现在，您应该准备好使用以下命令扫描项目漏洞：

    ```go
    snyk test
    ```

    上述命令将列出所有识别出的漏洞（主要或次要），包括它们的路径和修复指南，如下截图所示：

    ![图 7.12 – Snyk 漏洞发现    ](img/Figure_7.12_B17115.jpg)

    图 7.12 – Snyk 漏洞发现

1.  根据输出，Snyk 识别出两个主要问题。其中一个是与当前版本的 Gin 框架有关。点击 `Info` URL—you will be redirected to a dedicated page where you can learn more about the vulnerability, as illustrated in the following screenshot:![图 7.13 – HTTP 响应拆分页面    ](img/Figure_7.13_B17115.jpg)

    图 7.13 – HTTP 响应拆分页面

1.  大多数安全漏洞可以通过升级到最新稳定版本来解决。运行以下命令升级您的项目依赖项：

    ```go
    go.mod file will be upgraded to the latest available version, as illustrated in the following screenshot:
    ```

![图 7.14 – 升级 Go 包](img/Figure_7.14_B17115.jpg)

图 7.14 – 升级 Go 包

对于发现的漏洞，GitHub 上有一个已合并并可在 Gin 1.7 版本中找到的公开拉取请求，如下截图所示：

![图 7.15 – 漏洞修复](img/Figure_7.15_B17115.jpg)

图 7.15 – 漏洞修复

那就结束了——现在您已经知道如何使用 Snyk 扫描您的 Go 模块了！

注意

我们将在 *第九章**，实现 CI/CD 流水线* 中介绍如何将 Snyk 集成到 **持续集成/持续部署** (**CI/CD**) 流水线中，以持续检查应用程序的源代码中的安全漏洞。

# 运行 Postman 集合

在本书中，您已经学习了如何使用 **Postman** REST 客户端测试 API 端点。除了发送 API 请求外，Postman 还可以通过在集合中定义一组 API 请求来构建测试套件。

要设置此功能，请按照以下步骤操作：

1.  打开 Postman 客户端，然后从标题栏点击**新建**按钮，然后选择**集合**，如图下所示截图：![图 7.16 – 新建 Postman 集合    ](img/Figure_7.16_B17115.jpg)

    图 7.16 – 新建 Postman 集合

1.  将弹出一个新窗口——将集合命名为`Recipes API`，然后点击`List Recipes`，如图下所示截图：![图 7.17 – 新建请求    ](img/Figure_7.17_B17115.jpg)

    图 7.17 – 新建请求

1.  在地址栏中点击`http://localhost:8080/recipes`并选择`GET`方法。

好的——现在，一旦完成这些，你将在**测试**部分编写一些 JavaScript 代码。

在 Postman 中，你可以编写将在发送请求之前（预请求脚本）或接收响应之后执行的 JavaScript 代码。让我们在下一节中探讨如何实现这一点。

## Postman 中的脚本编写

测试脚本可以用来测试你的 API 是否按预期工作，或者检查新功能是否影响了现有请求的功能。

要编写脚本，请按以下步骤操作：

1.  点击**测试**部分，粘贴以下代码：

    ```go
    pm.test("More than 10 recipes", function () {
       var jsonData = pm.response.json();
       pm.expect(jsonData.length).to.least(10)
    });
    ```

    脚本将检查 API 请求返回的菜谱数量是否等于 10 个菜谱，如图下所示截图：

    ![图 7.18 – Postman 中的脚本编写    ](img/Figure_7.18_B17115.jpg)

    图 7.18 – Postman 中的脚本编写

1.  点击**发送**按钮，并检查 Postman 控制台，如图下所示截图：

![图 7.19 – 运行测试脚本](img/Figure_7.19_B17115.jpg)

图 7.19 – 运行测试脚本

你可以在*图 7.19*中看到测试脚本已通过。

你可能已经注意到 API URL 硬编码在地址栏中。虽然这样工作得很好，但如果你在维护多个环境（沙盒、预发布和生产），你需要一种方式来测试你的 API 端点而不必复制你的集合请求。幸运的是，你可以在 Postman 中创建环境变量。

要使用 URL 参数，请按以下步骤操作：

1.  点击右上角的*眼睛*图标，然后点击`http://localhost:8080`，如图下所示截图。点击**保存**：![图 7.20 – 环境变量    ](img/Figure_7.20_B17115.jpg)

    图 7.20 – 环境变量

1.  返回到你的`GET`请求，并使用以下 URL 变量。确保从右上角的下拉菜单中选择**测试**环境，如图下所示截图：![图 7.21 – 参数化请求    ](img/Figure_7.21_B17115.jpg)

    图 7.21 – 参数化请求

1.  现在，继续添加另一个 API 请求的测试脚本。以下脚本将在响应负载中查找特定的菜谱：

    ```go
    pm.test("Gnocchi recipe", function () {
       var jsonData = pm.response.json();
       var found = false;
       jsonData.forEach(recipe => {
           if (recipe.name == 'Gnocchi') {
               found = true;
           }
       })
       pm.expect(found).to.true
    });
    ```

1.  点击**发送**按钮，两个测试脚本都应成功，如图所示：

![图 7.22 – 运行多个测试脚本](img/Figure_7.22_B17115.jpg)

图 7.22 – 运行多个测试脚本

您现在可以为您的 API 端点定义多个测试用例场景。

让我们更进一步，创建另一个 API 请求，这次是为添加新食谱的端点，如下面的截图所示：

![Figure 7.23 – 新食谱请求![Figure 7.23_B17115.jpg](img/Figure_7.23_B17115.jpg)

图 7.23 – 新食谱请求

要这样做，请按照以下步骤操作：

1.  定义一个测试脚本，以检查在成功插入操作返回的 HTTP 状态码是否为`200-OK`代码，如下所示：

    ```go
    pm.test("Status code is 200", function () {
       pm.response.to.have.status(200);
    });
    ```

1.  定义另一个来检查插入的 ID 是否为 24 个字符的字符串，如下所示：

    ```go
    pm.test("Recipe ID is not null", function(){
       var id = pm.response.json().id;
       pm.expect(id).to.be.a("string");
       pm.expect(id.length).to.eq(24);
    })
    ```

1.  点击`401 – Unauthorized`，这是正常的，因为端点期望在 HTTP 请求中有一个授权头。您可以在以下截图中看到输出：![Figure 7.24 – 401 Unauthorized response    ![Figure 7.24_B17115.jpg](img/Figure_7.24_B17115.jpg)

    图 7.24 – 401 未授权响应

    注意

    要了解更多关于 API 认证的信息，请回到*第四章*，*构建 API 认证*，以获取逐步指南。

1.  添加一个有效的**JSON Web Token**（**JWT**）的`Authorization`头。这次，测试脚本成功通过！

1.  您现在在集合中有两个不同的 API 请求。通过点击**运行**按钮来运行集合。将弹出一个新窗口，如下面的截图所示：![Figure 7.25 – 集合运行器    ![Figure 7.25_B17115.jpg](img/Figure_7.25_B17115.jpg)

    图 7.25 – 集合运行器

1.  点击**运行食谱 API**按钮，两个 API 请求将按顺序执行，如下面的截图所示！![Figure 7.26_B17115.jpg](img/Figure_7.26_B17115.jpg)

    图 7.26 – 运行结果屏幕

1.  您可以通过点击**导出**按钮导出集合和所有 API 请求。应该会创建一个具有以下结构的 JSON 文件：

    ```go
    {
       "info": {},
       "item": [
           {
               "name": "New Recipe",
               "event": [
                   {
                       "listen": "test",
                       "script": {
                           "exec": [
                               "pm.test(\"Recipe ID is not 
                                   null\", function(){",
                               "var id = pm.response
                                     .json().id;",
                               "pm.expect(id).
                                       to.be.a(\"string\");",
                               "pm.expect(id.length)
                                       .to.eq(24);",
                               "})"
                           ],
                           "type": "text/javascript"
                       }
                   }
               ],
               "request": {
                   "method": "POST",
                   "header": [],
                   "body": {
                       "mode": "raw",
                       "raw": "{\n    \"name\": \"New York 
                                Pizza\"\n}",
                       "options": {
                           "raw": {
                               "language": "json"
                           }
                       }
                   },
                   "url": {
                       "raw": "{{url}}/recipes",
                       "host": [
                           "{{url}}"
                       ],
                       "path": [
                           "recipes"
                       ]
                   }
               },
               "response": []
           }
       ],
       "auth": {}
    }
    ```

导出 Postman 集合后，您可以使用**Newman**从终端运行它（[`github.com/postmanlabs/newman`](https://github.com/postmanlabs/newman)）。

在下一节中，我们将使用 Newman CLI 运行之前的 Postman 集合。

## 使用 Newman 运行集合

在定义了所有测试后，让我们使用 Newman 命令行执行它们。值得一提的是，您可以将这些测试进一步运行在您的 CI/CD 工作流程中作为后集成测试，以确保新的 API 更改并且功能没有生成任何回归。

要开始，请按照以下步骤操作：

1.  使用 npm 安装**Newman**。这里我们使用版本 5.2.2：

    ```go
    npm install -g newman
    ```

1.  安装完成后，使用导出的集合文件作为参数运行 Newman，如下所示：

    ```go
    newman run postman.json
    ```

    由于 URL 参数没有被定义，API 请求应该失败，如下面的截图所示：

    ![Figure 7.27 – 包含失败测试的集合    ![Figure 7.27_B17115.jpg](img/Figure_7.27_B17115.jpg)

    图 7.27 – 包含失败测试的集合

1.  您可以使用`--env-var`标志来设置其值，如下所示：

    ```go
    newman run postman.json --env-var "url=http://localhost:8080"
    ```

    如果所有调用都通过，则应该是以下输出：

![Figure 7.28 – 包含成功测试的集合![Figure 7.28_B17115.jpg](img/Figure_7.28_B17115.jpg)

图 7.28 – 成功测试的集合

你现在应该能够使用 Postman 自动化 API 端点测试。

注意

在*第十章*“捕获 Gin 应用程序指标”中，我们将介绍如何在 CI/CD 管道中成功发布应用程序后触发`newman` `run`命令。

# 摘要

在本章中，你学习了如何为 Gin Web 应用程序运行不同的自动化测试。你还探讨了如何集成外部工具，如 Gosec 和 Snyk，以检查代码质量、检测错误和发现潜在的安全漏洞。

在下一章中，我们将介绍我们在云上的分布式 Web 应用程序，主要使用 Docker 和 Kubernetes 在**Amazon Web Services**（**AWS**）上。你现在应该能够发布几乎无错误的程序，并在发布新功能到生产环境之前发现潜在的安全漏洞。

# 问题

1.  为`UpdateRecipeHandler` HTTP 处理器编写单元测试。

1.  为`DeleteRecipeHandler` HTTP 处理器编写单元测试。

1.  为`FindRecipeHandler` HTTP 处理器编写单元测试。

# 进一步阅读

《*Go 设计模式*》由 Mario Castro Contreras 著，Packt Publishing 出版
