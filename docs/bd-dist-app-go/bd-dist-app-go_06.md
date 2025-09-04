# 第四章：构建 API 身份验证

本章致力于介绍在构建公共**表示状态转移**（**REST**）**应用程序编程接口**（**API**）时需要遵循的最佳实践和建议。它探讨了如何编写一个身份验证中间件来保护 API 端点的访问，以及如何通过**超文本传输协议安全**（**HTTPS**）来提供服务。

本章，我们将重点关注以下主要主题：

+   探索身份验证

+   介绍**JavaScript 对象表示法**（**JSON**）**Web 令牌**（**JWT**）

+   持久化客户端会话和 cookie

+   使用 Auth0 进行认证

+   构建 HTTPS 服务器

到本章结束时，你将能够构建一个具有私有和公开端点的 RESTful API。

# 技术要求

要遵循本章的说明，你需要以下内容：

+   完全理解上一章的内容——本章是上一章的后续，它将使用相同的源代码。因此，一些代码片段将不会进行解释，以避免重复。

+   对 API 认证概念和 HTTPS 协议有基本了解。

本章的代码包托管在 GitHub 上，网址为[`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter04`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter04)。

# 探索身份验证

在上一章中，我们构建的 API 公开了多个端点。目前，这些端点是公开的，不需要任何认证。在现实世界的场景中，你需要保护这些端点。

以下图表展示了本章结束时需要保护的所有端点：

![图 4.1 – 保护 RESTful API 端点](img/Figure_4.1_B17115.jpg)

图 4.1 – 保护 RESTful API 端点

**列出**菜谱不需要认证，而负责**添加**、**更新**或**删除**菜谱的端点则需要认证。

可以使用多种方法来保护前面的端点——以下是我们可能使用的一些方法：API 密钥、基本认证、客户端会话、OpenID Connect、**开放授权**（**OAuth**）2.0 等。最基本的认证机制是使用 API 密钥。

## 使用 API 密钥

在这种方法中，客户端提供一个秘密，称为 HTTP 请求中的`X-API-KEY`头；如果密钥错误或请求头中没有找到密钥，则会抛出一个未经授权的错误（`401`），如下面的代码片段所示（为了简洁，完整的代码已被裁剪）：

```go
func (handler *RecipesHandler) NewRecipeHandler(
             c *gin.Context) {
   if c.GetHeader("X-API-KEY") != os.Getenv("X_API_KEY") {
       c.JSON(http.StatusUnauthorized, gin.H{
          "error": "API key not provided or invalid"})
       return
   }
}
```

在运行 MongoDB 和 Redis 容器之后运行应用程序，但这次需要设置以下`X-API-KEY`环境变量：

```go
X_API_KEY=eUbP9shywUygMx7u  MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin" MONGO_DATABASE=demo go run *.go
```

注意

你可以使用以下命令使用 OpenSSL 生成一个随机的秘密字符串：`openssl rand` `-base64 16`。

如果你尝试添加一个新的菜谱，将会返回一个`401`错误消息，如下面的屏幕截图所示：

![图 4.2 – 新菜谱![图片 4.2](img/Figure_4.2_B17115.jpg)

图 4.2 – 新食谱

然而，如果在 `POST` 请求中包含有效的 `X-API-KEY` 头部，食谱将被插入，如下面的截图所示：

![图 4.3 – POST 请求中的 X-API-KEY 头部![图片 4.3](img/Figure_4.3_B17115.jpg)

图 4.3 – POST 请求中的 X-API-KEY 头部

如果你不喜欢 Postman 作为 HTTP/s 客户端，你可以在你的终端上执行以下 cURL 命令：

```go
curl --location --request POST 'http://localhost:8080/recipes' \
--header 'X-API-KEY: eUbP9shywUygMx7u' \
--header 'Content-Type: application/json' \
--data-raw '{
   "name": "Homemade Pizza",
   "ingredients": ["..."],
   "instructions": ["..."],
   "tags": ["dinner", "fastfood"]
}'
```

目前，只有 `POST /recipes` 请求是受保护的。为了避免在其他 HTTP 端点中重复相同的代码片段，通过编写以下代码创建一个身份验证中间件：

```go
func AuthMiddleware() gin.HandlerFunc {
   return func(c *gin.Context) {
       if c.GetHeader("X-API-KEY") != 
               os.Getenv("X_API_KEY") {
           c.AbortWithStatus(401)
       }
       c.Next()
   }
}
```

在路由定义中，使用身份验证中间件。最后，将端点重新组合成一个组，如下所示：

```go
authorized := router.Group("/")
authorized.Use(AuthMiddleware()){
       authorized.POST("/recipes", 
                       recipesHandler.NewRecipeHandler)
       authorized.GET("/recipes", 
                      recipesHandler.ListRecipesHandler)
       authorized.PUT("/recipes/:id", 
                      recipesHandler.UpdateRecipeHandler)
       authorized.DELETE("/recipes/:id", 
                        recipesHandler.DeleteRecipeHandler)
}
```

在这个阶段，重新运行应用程序。如果你发出 `GET /recipes` 请求，将返回 `401` 错误，如下面的截图所示。这是正常的，因为列表食谱的路由处理程序位于身份验证中间件之后：

![图 4.4 – GET /recipes 需要 API 密钥![图片 4.4](img/Figure_4.4_B17115.jpg)

图 4.4 – GET /recipes 需要 API 密钥

你希望 `GET /recipes` 请求是公开的，因此将端点注册在组路由之外，如下所示：

```go
func main() {
   router := gin.Default()
   router.GET("/recipes", 
              recipesHandler.ListRecipesHandler)
   authorized := router.Group("/")
   authorized.Use(AuthMiddleware())
   {
       authorized.POST("/recipes", 
                       recipesHandler.NewRecipeHandler)
       authorized.PUT("/recipes/:id", 
                      recipesHandler.UpdateRecipeHandler)
       authorized.DELETE("/recipes/:id",   
                        recipesHandler.DeleteRecipeHandler)
       authorized.GET("/recipes/:id", 
                      recipesHandler.GetOneRecipeHandler)
   }

  router.Run()
}
```

如果你测试它，这次端点将返回食谱列表，如下面的截图所示：

![图 4.5 – 菜单列表![图片 4.5](img/Figure_4.5_B17115.jpg)

图 4.5 – 菜单列表

API 密钥很简单；然而，任何向 API 发送请求的人都会传输他们的密钥，在理论上，当不使用加密时，密钥很容易通过中间人攻击（**MITM**）被捕获。这就是为什么在下一节中，我们将介绍一种称为 JWT 的更安全的身份验证机制。

备注

MITM 指的是攻击者在两个当事人之间的对话中定位自己，以窃取他们的凭证的情况。更多详情，请查看以下链接：[`snyk.io/learn/man-in-the-middle-attack/`](https://snyk.io/learn/man-in-the-middle-attack/).

# 介绍 JWT

根据 **请求评论** (**RFC**) *7519* ([`tools.ietf.org/html/rfc7519`](https://tools.ietf.org/html/rfc7519)):

*"JSON Web Token (JWT) 是一个开放标准，它定义了一种紧凑且自包含的方式，用于以 JSON 对象的形式在各方之间安全地传输信息。由于它是数字签名的，因此该信息可以验证并受到信任。JWT 可以使用密钥或公钥/私钥对进行签名。"*

JWT 令牌由三个部分组成，由点分隔，如下面的截图所示：

![图 4.6 – JWT 的组成部分![图片 4.6](img/Figure_4.6_B17115.jpg)

图 4.6 – JWT 的组成部分

**头部**指示用于生成签名的算法。**载荷**包含有关用户的信息，以及令牌过期日期。最后，**签名**是使用密钥对头部和载荷部分进行散列的结果。

现在我们已经了解了 JWT 的工作原理，让我们将其集成到我们的 API 中。要开始，使用以下命令安装 JWT Go 实现：

```go
go get github.com/dgrijalva/jwt-go
```

该包将自动添加到`go.mod`文件中，如下所示：

```go
module github.com/mlabouardy/recipes-api
go 1.15
require (
   github.com/dgrijalva/jwt-go v3.2.0+incompatible 
   // indirect
   github.com/gin-gonic/gin v1.6.3
   github.com/go-redis/redis v6.15.9+incompatible
   github.com/go-redis/redis/v8 v8.4.10
   go.mongodb.org/mongo-driver v1.4.5
   golang.org/x/net v0.0.0-20201202161906-c7110b5ffcbb
)
```

在动手之前，让我解释一下 JWT 认证将如何实现。基本上，客户端需要使用用户名和密码进行登录。如果这些凭证有效，将生成 JWT 令牌并返回。客户端将在未来的请求中通过包含`Authorization`头来使用该令牌。如果向 API 发出请求，将通过将令牌的签名与使用密钥生成的签名进行比较来验证令牌。如果签名不匹配，则 JWT 被视为无效，并返回`401`错误。

下面的序列图展示了客户端和 API 之间的通信：

![图 4.7 – 序列图](img/Figure_4.7_B17115.jpg)

图 4.7 – 序列图

也就是说，在`handlers`文件夹下创建一个`auth.go`文件。这个文件将暴露处理认证工作流程的函数。以下是实现此功能的代码（为了简洁，已裁剪完整代码）：

```go
package handlers
import (
   "net/http"
   "os"
   "time"
   "github.com/dgrijalva/jwt-go"
)
type AuthHandler struct{}
type Claims struct {
   Username string `json:"username"`
   jwt.StandardClaims
}
type JWTOutput struct {
   Token   string    `json:"token"`
   Expires time.Time `json:"expires"`
}
func (handler *AuthHandler) SignInHandler(c *gin.Context) {}
```

接下来，你需要定义一个用户凭证的实体模型。在`models`文件夹中，创建一个具有用户名和密码属性的`user.go`结构体，如下所示：

```go
package models
type User struct {
   Password string `json:"password"`
   Username string `json:"username"`
}
```

模型定义完成后，我们可以继续实现认证处理器。

## 登录 HTTP 处理器

`SignInHandler`将请求体编码为`User`结构体，并验证凭证是否正确。然后，它将颁发一个有效期为 10 分钟的 JWT 令牌。JWT 的签名是头部和有效载荷的 Base64 表示形式以及一个密钥的组合输出（注意`JWT_SECRET`环境变量的使用）。然后将组合传递给`HS256`哈希算法。值得一提的是，你必须为了安全起见，将凭证从源代码中移除。实现方式如下：

```go
func (handler *AuthHandler) SignInHandler(c *gin.Context) {
   var user models.User
   if err := c.ShouldBindJSON(&user); err != nil {
       c.JSON(http.StatusBadRequest, gin.H{"error": 
           err.Error()})
       return
   }
   if user.Username != "admin" || user.Password != 
          "password" {
       c.JSON(http.StatusUnauthorized, gin.H{"error": 
          "Invalid username or password"})
       return
   }
   expirationTime := time.Now().Add(10 * time.Minute)
   claims := &Claims{
       Username: user.Username,
       StandardClaims: jwt.StandardClaims{
           ExpiresAt: expirationTime.Unix(),
       },
   }
   token := jwt.NewWithClaims(jwt.SigningMethodHS256, 
                              claims)
   tokenString, err := token.SignedString([]byte(
                       os.Getenv("JWT_SECRET")))
   if err != nil {
       c.JSON(http.StatusInternalServerError, 
              gin.H{"error": err.Error()})
       return
   }
   jwtOutput := JWTOutput{
       Token:   tokenString,
       Expires: expirationTime,
   }
   c.JSON(http.StatusOK, jwtOutput)
}
```

注意

关于哈希算法的工作原理的更多信息，请查看官方 RFC：[`tools.ietf.org/html/rfc7518`](https://tools.ietf.org/html/rfc7518)。

创建了`SignInHandler`处理器后，让我们通过更新`main.go`文件在`POST /signin`端点上注册此处理器，如下所示（为了简洁，已裁剪代码）：

```go
package main
import (
   ...
)
var authHandler *handlers.AuthHandler
var recipesHandler *handlers.RecipesHandler
func init() {
   ...
   recipesHandler = handlers.NewRecipesHandler(ctx, 
      collection, redisClient)
   authHandler = &handlers.AuthHandler{}
}
func main() {
   router := gin.Default()
   router.GET("/recipes", 
              recipesHandler.ListRecipesHandler)
   router.POST("/signin", authHandler.SignInHandler)
   ...
}
```

接下来，我们更新`handler/auth.go`中的认证中间件，以检查`Authorization`头而不是`X-API-KEY`属性。然后将头传递给`ParseWithClaims`方法。它使用`Authorization`头部的头部和有效载荷以及密钥生成签名。然后，它验证签名是否与 JWT 上的签名匹配。如果不匹配，则 JWT 被视为无效，并返回`401`状态码。Go 实现如下：

```go
func (handler *AuthHandler) AuthMiddleware() gin.HandlerFunc {
   return func(c *gin.Context) {
       tokenValue := c.GetHeader("Authorization")
       claims := &Claims{}
       tkn, err := jwt.ParseWithClaims(tokenValue, claims, 
              func(token *jwt.Token) (interface{}, error) {
           return []byte(os.Getenv("JWT_SECRET")), nil
       })
       if err != nil {
           c.AbortWithStatus(http.StatusUnauthorized)
       }
       if tkn == nil ||!tkn.Valid {
           c.AbortWithStatus(http.StatusUnauthorized)
       }
       c.Next()
   }
}
```

使用以下命令重新运行应用程序：

```go
JWT_SECRET=eUbP9shywUygMx7u MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin" MONGO_DATABASE=demo go run *.go
```

服务器日志如下所示：

![图 4.8 – 应用程序日志](img/Figure_4.8_B17115.jpg)

图 4.8 – 应用程序日志

现在，如果你尝试插入一个新的食谱，将会返回一个 `401` 错误，如下所示：

![图 4.9 – 未授权端点](img/Figure_4.9_B17115.jpg)

图 4.9 – 未授权端点

你需要首先使用 `admin/password` 凭证通过在 `/signin` 端点上执行 `POST` 请求来登录。一旦成功，端点将返回一个看起来像这样的令牌：

![图 4.10 – 登录端点](img/Figure_4.10_B17115.jpg)

图 4.10 – 登录端点

令牌由三个部分组成，由点分隔。你可以通过访问 [`jwt.io/`](https://jwt.io/) 来解码令牌，返回以下输出（你的结果可能不同）：

![图 4.11 – 解码 JWT 令牌](img/Figure_4.11_B17115.jpg)

图 4.11 – 解码 JWT 令牌

注意

头部和有效负载部分是 Base64 编码的，但你可以使用 `base64` 命令来解码它们的值。

现在，对于后续请求，你需要将令牌包含在 `Authorization` 头部中，以便能够访问受保护的端点，例如发布新食谱，如下面的截图所示：

![图 4.12 – 发布新食谱](img/Figure_4.12_B17115.jpg)

图 4.12 – 发布新食谱

到目前为止，一切都很顺利——然而，10 分钟后，令牌将过期。例如，如果你尝试发布一个新的食谱，即使你包含了 `Authorization` 头部，也会抛出一个 `401` 未授权消息，正如我们可以在下面的截图中所看到的：

![图 4.13 – 过期 JWT](img/Figure_4.13_B17115.jpg)

图 4.13 – 过期 JWT

因此，让我们看看如何在令牌过期后如何刷新此令牌。

## 刷新 JWT

你可以将过期时间增加，使 JWT 令牌持续更长时间；然而，这并不是一个永久的解决方案。你可以做的另一件事是暴露一个端点，允许用户刷新令牌，这样客户端应用程序就可以刷新令牌，而无需再次要求用户输入用户名和密码。函数处理程序如下代码片段所示——它接受之前的令牌并返回一个新的令牌，具有更新的过期时间：

```go
func (handler *AuthHandler) RefreshHandler(c *gin.Context) {
   tokenValue := c.GetHeader("Authorization")
   claims := &Claims{}
   tkn, err := jwt.ParseWithClaims(tokenValue, claims, 
          func(token *jwt.Token) (interface{}, error) {
       return []byte(os.Getenv("JWT_SECRET")), nil
   })
   if err != nil {
       c.JSON(http.StatusUnauthorized, gin.H{"error": 
          err.Error()})
       return
   }
   if tkn == nil ||!tkn.Valid {
       c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid	                                              token"})
       return
   }
   if time.Unix(claims.ExpiresAt, 0).Sub(time.Now()) > 
            30*time.Second {
       c.JSON(http.StatusBadRequest, gin.H{"error": 
            "Token is not expired yet"})
       return
   }
   expirationTime := time.Now().Add(5 * time.Minute)
   claims.ExpiresAt = expirationTime.Unix()
   token := jwt.NewWithClaims(jwt.SigningMethodHS256, 
                              claims)
   tokenString, err := token.SignedString(os.Getenv(
                       "JWT_SECRET"))
   if err != nil {
       c.JSON(http.StatusInternalServerError, 
           gin.H{"error": err.Error()})
       return
   }
   jwtOutput := JWTOutput{
       Token:   tokenString,
       Expires: expirationTime,
   }
   c.JSON(http.StatusOK, jwtOutput)
}
```

注意

在一个网络应用程序中，`/refresh` 端点可以用来在后台刷新 JWT 令牌，而无需每几分钟要求用户登录。

使用以下代码在 `POST /refresh` 端点上注册 `RefreshHandler` 处理程序：

```go
router.POST("/refresh", authHandler.RefreshHandler)
```

如果你重新运行应用程序，`/refresh` 端点将会暴露，如下面的截图所示：

![图 4.14 – /refresh 端点](img/Figure_4.14_B17115.jpg)

图 4.14 – /refresh 端点

你现在可以在 `/refresh` 端点上发出 `POST` 请求，并将生成并返回一个新的令牌。

太棒了——你现在有一个工作的认证工作流程！然而，用户凭证仍然硬编码在应用程序代码源中。你可以通过将它们存储在 **MongoDB** 服务器上来改进这一点。这里提供了一个更新的序列图：

![图 4.15 – 在 MongoDB 中存储凭证](img/Figure_4.15_B17115.jpg)

图 4.15 – 在 MongoDB 中存储凭证

为了能够与 MongoDB 服务器交互，您需要在`auth.go`文件中将 MongoDB 集合添加到`AuthHandler`结构体中，如下面的代码片段所示。然后，您可以对`users`集合执行`FindOne`操作，以验证给定的凭证是否存在：

```go
type AuthHandler struct {
   collection *mongo.Collection
   ctx        context.Context
}
func NewAuthHandler(ctx context.Context, collection 
          *mongo.Collection) *AuthHandler {
   return &AuthHandler{
       collection: collection,
       ctx:        ctx,
   }
}
```

接下来，更新`SignInHandler`方法，通过将它们与数据库中的条目进行比较来验证用户凭证是否有效。以下是实现方式：

```go
func (handler *AuthHandler) SignInHandler(c *gin.Context) {

   h := sha256.New()
   cur := handler.collection.FindOne(handler.ctx, bson.M{
       "username": user.Username,
       "password": string(h.Sum([]byte(user.Password))),
   })
   if cur.Err() != nil {
       c.JSON(http.StatusUnauthorized, gin.H{"error": 
           "Invalid username or password"})
       return
   }
   ...
}
```

在`init()`方法中，您需要设置到`users`集合的连接，然后将集合实例传递给`AuthHandler`实例，如下所示：

```go
collectionUsers := client.Database(os.Getenv(
                   "MONGO_DATABASE")).Collection("users")
authHandler = handlers.NewAuthHandler(ctx, collectionUsers)
```

确保保存`main.go`中的更改，然后 API 就准备好了！

## 对密码进行哈希处理和加盐

在运行应用程序之前，我们需要使用一些用户初始化`users`集合。在`main.go`文件中创建一个新的项目，内容如下：

```go
func main() {
   users := map[string]string{
       "admin":      "fCRmh4Q2J7Rseqkz",
       "packt":      "RE4zfHB35VPtTkbT",
       "mlabouardy": "L3nSFRcZzNQ67bcc",
   }
   ctx := context.Background()
   client, err := mongo.Connect(ctx, 
       options.Client().ApplyURI(os.Getenv("MONGO_URI")))
   if err = client.Ping(context.TODO(), 
          readpref.Primary()); err != nil {
       log.Fatal(err)
   }
   collection := client.Database(os.Getenv(
       "MONGO_DATABASE")).Collection("users")
   h := sha256.New()
   for username, password := range users {
       collection.InsertOne(ctx, bson.M{
           "username": username,
           "password": string(h.Sum([]byte(password))),
       })
   }
}
```

上述代码将三个用户（`admin`、`packt`、`mlabouardy`）插入到`users`集合中。出于安全目的，在将密码保存到 MongoDB 之前，使用*SHA256*算法对密码进行哈希处理和加盐。该算法为密码生成一个唯一的 256 位签名，该签名可以解密回原始密码。这样，敏感信息可以保持安全。

注意

不建议在数据库中存储明文密码。通过对用户密码进行哈希处理和加盐，我们确保黑客无法登录，因为被盗的数据不会包含凭证。

您可以使用`MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin" MONGO_DATABASE=demo go run main.go`命令运行代码。使用 MongoDB Compass 检查用户是否已插入（有关如何使用 MongoDB Compass 的逐步指南，请返回到*第三章**，使用 MongoDB 管理数据持久性*），如下面的截图所示：

![图 4.16 – 用户集合](img/Figure_4.16_B17115.jpg)

图 4.16 – 用户集合

如您可能注意到的，密码已经进行了哈希处理和加盐。当用户被插入到 MongoDB 中时，我们可以通过发出以下负载的`POST`请求来测试登录端点：

![图 4.17 – 登录端点](img/Figure_4.17_B17115.jpg)

图 4.17 – 登录端点

注意

我们可以通过在客户端设置 JWT 值作为 cookie 来改进 JWT 实现。这样，cookie 就会随请求一起发送。

您可以进一步创建一个注册端点来创建新用户并将其保存到 MongoDB 数据库中。现在您应该熟悉如何使用 JWT 实现身份验证机制。

# 客户端会话和 cookie 的持久化

到目前为止，你必须在每个请求中都包含 `Authorization` 头。更好的解决方案是生成一个 **会话 cookie**。会话 cookie 允许用户在应用程序内被识别，而无需每次都进行身份验证。没有 cookie，每次你发出 API 请求时，服务器都会把你当作一个全新的访客。

要生成会话 cookie，请按照以下步骤操作：

1.  使用以下命令安装 *Gin 中间件* 以进行会话管理：

    ```go
    go get github.com/gin-contrib/sessions
    ```

1.  使用以下代码将 *Redis* 配置为用户会话的存储：

    ```go
    store, _ := redisStore.NewStore(10, "tcp", 
          "localhost:6379", "", []byte("secret"))
    router.Use(sessions.Sessions("recipes_api", store))
    ```

    注意

    你可以不硬编码 Redis **统一资源标识符**（**URI**），而是使用环境变量。这样，你可以将配置从 API 源代码中分离出来。

1.  然后，更新 `SignInHandler` 以生成一个具有唯一 ID 的会话，如下面的代码片段所示。会话在用户登录后开始，并在之后某个时间点过期。登录用户的会话信息将存储在 Redis 缓存中：

    ```go
    func (handler *AuthHandler) SignInHandler(c *gin.Context) {
       var user models.User
       if err := c.ShouldBindJSON(&user); err != nil {
           c.JSON(http.StatusBadRequest, gin.H{"error": 
               err.Error()})
           return
       }
       h := sha256.New()
       cur := handler.collection.FindOne(handler.ctx, bson.M{
           "username": user.Username,
           "password": string(h.Sum([]byte(user.Password))),
       })
       if cur.Err() != nil {
           c.JSON(http.StatusUnauthorized, gin.H{"error": 
               "Invalid username or password"})
           return
       }
       sessionToken := xid.New().String()
       session := sessions.Default(c)
       session.Set("username", user.Username)
       session.Set("token", sessionToken)
       session.Save()
       c.JSON(http.StatusOK, gin.H{"message":                                "User signed in"})
    }
    ```

1.  接下来，更新 `AuthMiddleware` 以从请求 cookie 中获取令牌。如果 cookie 未设置，我们通过返回 `http.StatusForbidden` 响应来返回 `403` 代码（`Forbidden`），如下面的代码片段所示：

    ```go
    func (handler *AuthHandler) AuthMiddleware() gin.	  	      HandlerFunc {
       return func(c *gin.Context) {
           session := sessions.Default(c)
           sessionToken := session.Get("token")
           if sessionToken == nil {
               c.JSON(http.StatusForbidden, gin.H{
                   "message": "Not logged",
               })
               c.Abort()
           }
           c.Next()
       }
    }
    ```

1.  在端口 8080 上启动服务器，并在 `/signin` 端点上使用有效的用户名和密码发出 `POST` 请求。应该会生成一个 cookie，如下面的截图所示：

![图 4.18 – 会话 cookie](img/Figure_4.18_B17115.jpg)

图 4.18 – 会话 cookie

现在，会话将跨所有其他 API 路由持久化。因此，你可以与 API 端点交互，而无需包含任何授权头，如下面的截图所示：

![图 4.19 – 基于会话的认证](img/Figure_4.19_B17115.jpg)

图 4.19 – 基于会话的认证

之前的例子使用了 Postman 客户端，但如果你是 `cURL` 粉丝，请按照以下步骤操作：

1.  使用以下命令将生成的 cookie 存储到文本文件中：

    ```go
    curl -c cookies.txt -X POST http://localhost:8080/signin -d '{"username":"admin", "password":"fCRmh4Q2J7Rseqkz"}'
    ```

1.  然后，在未来的请求中注入 `cookies.txt` 文件，如下所示：

    ```go
    curl -b cookies.txt -X POST http://localhost:8080/recipes -d '{"name":"Homemade Pizza", "steps":[], "instructions":[]}'
    ```

    注意

    你可以实现一个刷新路由来生成一个新的会话 cookie，并更新其过期时间。

    API 生成的所有会话都将持久保存在 Redis 中。你可以使用 Redis Insight **用户界面**（**UI**）（托管在 Docker 容器中）来浏览保存的会话，如下面的截图所示：

    ![图 4.20 – Redis 中存储的会话列表    ](img/Figure_4.20_B17115.jpg)

    图 4.20 – Redis 中存储的会话列表

1.  要注销，你可以实现 `SignOutHandler` 处理器来使用以下命令清除会话 cookie：

    ```go
    func (handler *AuthHandler) SignOutHandler(c       *gin.Context) {
       session := sessions.Default(c)
       session.Clear()
       session.Save()
       c.JSON(http.StatusOK, gin.H{"message":                                "Signed out..."})
    }
    ```

1.  记得在 `main.go` 文件上注册处理器，如下所示：

    ```go
    router.POST("/signout", authHandler.SignOutHandler)
    ```

1.  运行应用程序，应该会暴露一个注销端点，如下面的截图所示：![图 4.21 – 注销处理器    ](img/Figure_4.21_B17115.jpg)

    图 4.21 – 注销处理器

1.  现在，使用 Postman 客户端执行一个`POST`请求来测试它，如下所示：![图 4.22 – 注销    ](img/Figure_4.22_B17115.jpg)

    图 4.22 – 注销

    会话 cookie 将被删除，如果你现在尝试添加一个新的食谱，将会返回一个`403`错误，如下截图所示：

    ![图 4.23 – 添加新食谱    ](img/Figure_4.23_B17115.jpg)

    图 4.23 – 添加新食谱

1.  确保通过创建一个新的功能分支将更改提交到 GitHub。然后，将分支合并到开发模式，如下所示：

```go
git add .
git commit -m "session based authentication"
git checkout -b feature/session
git push origin feature/session
```

以下截图显示了包含 JWT 和 cookie 认证功能的拉取请求：

![图 4.24 – GitHub 上的拉取请求](img/Figure_4.24_B17115.jpg)

图 4.24 – GitHub 上的拉取请求

太棒了！API 端点现在已安全，可以公开提供服务。

# 使用 Auth0 进行认证

到目前为止，认证机制已内置在应用程序中。维护这样一个系统可能会在长期成为瓶颈，这就是为什么你可能需要考虑使用外部服务，如**Auth0**。这是一个一站式认证解决方案，它为你提供了强大的报告和分析功能，以及一个**基于角色的访问控制**（**RBAC**）系统。

要开始，请按照以下步骤操作：

1.  创建一个免费账户([`auth0.com/signup`](https://auth0.com/signup))。创建后，在您所在的区域设置一个租户域，如下截图所示：![图 4.25 – Auth0 租户域    ](img/Figure_4.25_B17115.jpg)

    图 4.25 – Auth0 租户域

1.  然后，创建一个新的 API，命名为`Recipes API`。将标识符设置为[`api.recipes`](https://api.recipes)`.io`，并将签名算法设置为`RS256`，如下截图所示：![图 4.26 – Auth0 新 API    ](img/Figure_4.26_B17115.jpg)

    图 4.26 – Auth0 新 API

1.  一旦创建了 API，你需要将 Auth0 服务集成到 API 中。下载以下 Go 包：

    ```go
    go get -v gopkg.in/square/go-jose.v2
    go get -v github.com/auth0-community/go-auth0
    ```

1.  接下来，更新`AuthMiddleware`，如下代码片段所示。中间件将检查是否存在访问令牌以及它是否有效。如果通过检查，请求将继续。如果不通过，将返回一个`401 授权`错误：

    ```go
    func (handler *AuthHandler) AuthMiddleware() gin.HandlerFunc {
       return func(c *gin.Context) {
           var auth0Domain = "https://" + os.Getenv(
               "AUTH0_DOMAIN") + "/"
           client := auth0.NewJWKClient(auth0.JWKClientOptions{
               URI: auth0Domain + ".well-known/jwks.json"}, 
               nil)
           configuration := auth0.NewConfiguration(client, 
               []string{os.Getenv("AUTH0_API_IDENTIFIER")}, 
               auth0Domain, jose.RS256)
           validator := auth0.NewValidator(configuration, 	                                       nil)
           _, err := validator.ValidateRequest(c.Request)
           if err != nil {
               c.JSON(http.StatusUnauthorized,  	 	          	                  gin.H{"message": "Invalid token"})
               c.Abort()
               return
           }
           c.Next()
       }
    }
    ```

    Auth0 使用 RS256 算法签名访问令牌。验证过程使用位于`AUTH0_DOMAIN/.well-known/jwks.json`的公钥（以**JSON Web Key Set**（**JWKS**）格式），来验证给定的令牌。

1.  使用`AUTH0_DOMAIN`和`AUTH0_API_IDENTIFIER`运行应用程序，如下代码片段所示。确保用你在 Auth0 仪表板中复制的值替换这些变量：

    ```go
    AUTH0_DOMAIN=DOMAIN.eu.auth0.com  AUTH0_API_IDENTIFIER="https://api.recipes.io" MONGO_URI="mongodb://admin:password@localhost:27017/test?authSource=admin" MONGO_DATABASE=demo go run *.go
    ```

    现在，如果你尝试向你的 API 发送请求而不发送访问令牌，你会看到以下消息：

    ![图 4.27 – 未授权访问    ](img/Figure_4.27_B17115.jpg)

    图 4.27 – 未授权访问

1.  现在，更新您的 API 请求以包含访问令牌，如图所示：![图 4.29 – 授权访问    ](img/Figure_4.28_B17115.jpg)

    图 4.28 – 使用 cURL 生成访问令牌

1.  在您的终端会话中执行以下命令以生成一个可以用来与您的后端 API 通信的访问令牌：

    ```go
    curl --request POST \
      --url https://recipesapi-packt.eu.auth0.com/oauth/token \
      --data '{"client_id":"MyFRmUZS","client_secret":"7fArWGkSva","audience":"https://api.recipes.io","grant_type":"client_credentials"}'
    ```

    将生成一个访问令牌，如下所示（您应该有一个不同的值）：

    ```go
    {
     "access_token":"eyJhbGciOiJSUzI1NiIsInR5cCI 6IkpXVCIsImtpZCI6IkZ5T19SN2dScDdPakp3RmJQRVB3dCDz",
       "expires_in":86400,
       "token_type":"Bearer"
    }
    ```

1.  注意

    ](img/Figure_4.29_B17115.jpg)

    图 4.29 – 授权访问

1.  这也可以通过命令行使用 cURL 直接测试。只需将以下代码片段中显示的`ACCESS_TOKEN`值替换为您的测试令牌，然后将其粘贴到您的终端中：

    ```go
    curl --request POST \
      --url http://localhost:8080/recipes \
      --header 'Authorization: Bearer ACCESS_TOKEN'\
      --data '{"name":"Pizza "}'
    ```

太棒了！你刚刚使用 Go 语言和 Gin 框架开发了一个安全的 API。

# 要生成访问令牌，请返回到**Auth0**仪表板，点击**APIs**，然后选择**Recipes API**。从那里，点击**测试**选项卡并复制以下截图中的 cURL 命令：![图 4.28 – 使用 cURL 生成访问令牌使用 HTTPS 协议导航到转发 URL。URL 旁边应该显示**连接安全**消息，如图所示：图 4.32 – 通过 HTTPS 提供服务 1.  使用`ngrok`解决方案以支持 HTTP 和 HTTPS 的公共**统一资源定位符**（**URL**）来提供我们的本地 Web API。    ](img/Figure_4.31_B17115.jpg)

    在高级章节中，我们将探讨如何在云服务提供商如**亚马逊网络服务**（**AWS**）上免费购买域名并设置 HTTPS。

1.  从[`ngrok.com/download`](https://ngrok.com/download)的官方 Ngrok 页面下载基于您的**操作系统**（**OS**）的 ZIP 文件。在这本书中，我们将使用版本 2.3.35。下载后，使用以下命令在终端中解压 Ngrok：

    ```go
    unzip ngrok-stable-darwin-amd64.zip
    cp ngrok /usr/local/bin/
    chmod +x /usr/local/bin/ngrok
    ```

1.  通过执行以下命令来验证是否正确安装：

    ```go
    ngrok version
    ```

    应该输出以下消息：

    ![图 4.30 – Ngrok 版本    ](img/Figure_4.30_B17115.jpg)

    图 4.30 – Ngrok 版本

1.  通过运行以下命令配置 Ngrok 以监听并转发到 8080 端口，这是 RESTful API 暴露的端口：

    ```go
    ngrok http 8080
    ```

    将生成一个公共 URL，可以用作代理，从互联网上与 API 交互，如图所示：

    ](img/Figure_4.30_B17115.jpg)

    要设置此配置，请按照以下步骤操作：

    图 4.31 – Ngrok 转发

1.  构建 HTTPS 服务器

![图 4.32 – 通过 HTTPS 提供服务](img/Figure_4.32_B17115.jpg)

![图 4.31 – Ngrok 转发您现在可以从另一台机器或设备访问 API，或者与他人共享。在下一节中，我们将介绍如何创建自己的**安全套接字层**（**SSL**）证书来保护本地运行的域名。## 自签名证书 SSL 证书是网站从 HTTP 和 HTTPS 转移所使用的。该证书使用 SSL/**传输层安全性**（**TLS**）加密来保护用户数据安全，验证网站所有权，防止攻击者创建网站的假版本，并赢得用户信任。要创建自签名证书，请按以下步骤操作：1.  创建一个存储证书的目录，并使用 OpenSSL 命令行生成公钥和私钥，如下所示：    ```go    mkdir certs    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout certs/localhost.key -out certs/localhost.crt    ```1.  您需要填写一个简单的问卷调查——确保将完全限定主机名设置为 `localhost`，如图所示：![Figure 4.33 – 生成自签名证书    ![图片](img/Figure_4.33_B17115.jpg)

    Figure 4.33 – 生成自签名证书

    在这样做的时候，将生成两个文件，如下所示：

    `localhost.crt`：自签名证书

    `localhost.key`：私钥

    注意

    在高级章节中，我们将介绍如何使用 Let's Encrypt ([`letsencrypt.org`](https://letsencrypt.org)) 免费获取生产环境的有效证书。

1.  将 `main.go` 更新为使用自签名证书在 HTTPS 上运行服务器，如下所示（注意我们现在使用的是端口 443，这是 HTTPS 的默认端口）：

    ```go
    router.RunTLS(":443", "certs/localhost.crt", "certs/localhost.key")
    ```

1.  运行应用程序，日志将确认 API 通过 HTTPS 提供服务，如图所示：![Figure 4.34 – 监听和提供 HTTPS    ![图片](img/Figure_4.34_B17115.jpg)

    Figure 4.34 – 监听和提供 HTTPS

1.  打开浏览器，然后导航到 [`localhost/recipes`](https://localhost/recipes)。URL 框的左侧将显示一个安全网站图标，如图下截图所示：![Figure 4.35 – 连接到本地的加密连接    ![图片](img/Figure_4.35_B17115.jpg)

    ```go
    curl --cacert certs/localhost.crt https://localhost/recipes
    ```

    或者，您可以使用 `–k` 标志来跳过 SSL 验证，如下所示（如果与外部网站交互则不建议使用）：

    ```go
    curl -k https://localhost/recipes
    ```

1.  对于开发，只需继续使用 localhost，或从自定义域名访问 API。您可以通过在您的 `/etc/hosts` 文件中添加以下条目在本地创建一个域名别名：

    ```go
     127.0.0.1 api.recipes.io
    ```

1.  保存更改后，您可以通过在 `api.recipes.io` 上执行 `ping` 命令来测试它，如下所示：

![Figure 4.37 – ping 输出![图片](img/Figure_4.37_B17115.jpg)

Figure 4.37 – ping 输出

域名可达，指向 `127.0.0.1`。您现在可以通过导航到 [`api.recipes.io:8080`](https://api.recipes.io:8080) 来访问 RESTful API，如图下截图所示：

![Figure 4.38 – 别名域名![图片](img/Figure_4.38_B17115.jpg)

Figure 4.38 – 别名域名

太好了！现在您将能够通过身份验证来保护您的 API 端点，并通过自定义域名本地提供服务。

在我们结束本章之前，您需要更新 API 文档以包括我们在本章中实现的身份验证端点，如下所示：

1.  首先，更新一般元数据以包括 API 请求中的 `Authorization` 标头，如下所示：

    ```go
    // Recipes API
    //
    // This is a sample recipes API. You can find out more 
       about the API at 
       https://github.com/PacktPublishing/Building-
       Distributed-Applications-in-Gin.
    //
    //  Schemes: http
    //  Host: api.recipes.io:8080
    //  BasePath: /
    //  Version: 1.0.0
    //  Contact: Mohamed Labouardy 
    //  <mohamed@labouardy.com> https://labouardy.com
    //  SecurityDefinitions:
    //  api_key:
    //    type: apiKey
    //    name: Authorization
    //    in: header
    //
    //  Consumes:
    //  - application/json
    //
    //  Produces:
    //  - application/json
    // swagger:meta
    package main
    ```

1.  然后，在`SignInHandler`处理器上方编写一个`swagger:operation`注解，如下所示：

    ```go
    // swagger:operation POST /signin auth signIn
    // Login with username and password
    // ---
    // produces:
    // - application/json
    // responses:
    //     '200':
    //         description: Successful operation
    //     '401':
    //         description: Invalid credentials
    func (handler *AuthHandler) SignInHandler(c *gin.Context) {}
    ```

1.  在`RefreshHandler`处理器上方编写一个`swagger:operation`注解，如下所示：

    ```go
    // swagger:operation POST /refresh auth refresh
    // Get new token in exchange for an old one
    // ---
    // produces:
    // - application/json
    // responses:
    //     '200':
    //         description: Successful operation
    //     '400':
    //         description: Token is new and doesn't need 
    //                      a refresh
    //     '401':
    //         description: Invalid credentials
    func (handler *AuthHandler) RefreshHandler(c *gin.Context) 
    {}
    ```

1.  该操作期望包含以下属性的请求体：

    ```go
    // API user credentials
    // It is used to sign in
    //
    // swagger:model user
    type User struct {
      // User's password
      //
      // required: true
      Password string `json:"password"`
      // User's login
      //
      // required: true
      Username string `json:"username"`
    }
    ```

1.  生成 OpenAPI 规范，然后通过执行以下命令使用 Swagger UI 提供 JSON 文件：

    ```go
    /refresh and /signin) should be added to the list of operations, as shown in the following screenshot:![Figure 4.39 – Authentication operations    ](img/Figure_4.39_B17115.jpg)Figure 4.39 – Authentication operations
    ```

1.  现在，点击`signin`端点，你将能够直接从 Swagger UI 中填写用户名和密码属性，如下面的截图所示：![Figure 4.40 – Sign-in credentials    ![Figure 4.40 – Sign-in credentials](img/Figure_4.40_B17115.jpg)

    图 4.40 – 登录凭证

1.  接下来，点击`Authorization`头以与需要授权的端点交互，如下面的截图所示：

![Figure 4.41 – Authorization header

![img/Figure_4.41_B17115.jpg]

图 4.41 – 授权头

现在，你已经能够构建一个安全的 Gin RESTful API 并通过 HTTPS 协议提供服务。

# 摘要

在本章中，你学习了基于 Gin Web 框架构建安全 RESTful API 的一些最佳实践和建议。你还了解了如何在 Golang 中实现 JWT 以及如何在 API 请求之间持久化会话 cookie。

你还探讨了如何使用第三方解决方案，如 Auth0，作为身份验证提供者，以及如何将其与 Golang 集成以保护 API 端点。最后，你学习了如何通过 HTTPS 协议提供 API。

在下一章中，我们将使用 React Web 框架在 RESTful API 之上构建一个用户友好的 UI（也称为前端）。

# 问题

1.  你会如何实现一个注册端点来创建一个新的用户账户？

1.  你会如何实现一个个人资料端点以返回用户个人资料？

1.  你会如何生成一个注销端点的 Swagger 规范？

# 进一步阅读

+   *《OAuth 2.0 食谱》* 由阿道夫·埃洛伊·纳西メント著，Packt 出版社

+   *《SSL 完全指南 - HTTP 到 HTTPS》* 由博甘·斯塔什丘克著，Packt 出版社
