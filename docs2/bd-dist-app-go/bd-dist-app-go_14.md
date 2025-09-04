# 评估

本节包含所有章节的问题答案。

# *第一章* – 开始使用 Gin

1.  **Golang** 目前是软件开发行业中增长最快的编程语言之一。它是一种轻量级、开源的语言，适合今天的微服务架构。

1.  存在多个 Web 框架，其中最受欢迎的是 **Gin**，**Martini** 和 **Gorilla**。

1.  Go 模块是将一组包组合在一起并为其指定一个版本号以标记其在特定时间点存在的方式。

1.  由 Gin 框架支持的 HTTP 服务器的默认端口是 `8080`。

1.  您可以使用 `c.JSON()` 或 `c.XML()` 方法返回字面量 JSON 或 XML 结构体。

# *第二章* – 设置 API 端点

1.  **GitFlow** 是开发者在使用版本控制时可以遵循的分支策略。要应用 GitFlow 模型，您需要一个中央 Git 仓库和两个主要分支：

    **主分支**：它存储官方发布历史。

    **开发**：它作为功能集成的分支。

1.  模型是具有基本 Go 类型的普通结构体。要在 Go 中声明结构体，请使用以下格式：

    ```go
    Type ModelName struct{ 
        Field1 TYPE 
        Fiel2 TYPE 
    }
    ```

1.  要将请求体绑定到类型，我们使用 Gin 模型绑定。Gin 支持绑定 JSON、XML 和 YAML。Gin 提供了两套绑定方法：

    应该绑定：`ShouldBindJSON()`，`ShouldBindXML()`，`ShouldBindYAML()`

    必须绑定：使用指定的绑定引擎绑定结构体指针。如果发生任何错误，它将终止请求并返回 HTTP 400。

1.  首先，定义一个带有 ID 作为路径参数的路由：

    ```go
    router.GET("/recipes/:id", GetRecipeHandler)
    ```

    `GetRecipeHandler` 函数解析 ID 参数，并使用 `go` 循环遍历配方列表。如果 ID 与列表中的配方匹配，则将其返回，否则将抛出 `404 错误`，如下所示：

    ```go
    func GetRecipeHandler(c *gin.Context) {
         id := c.Query("id")
         for i := 0; i < len(recipes); i++ {
              if recipes[i].ID == id {
                   c.JSON(http.StatusOK, recipes[i])
              }
         }
         c.JSON(http.StatusNotFound, gin.H{"error": "Recipe  	                                       not found"})
    }
    ```

1.  要定义一个参数，我们使用 swagger:parameters 注解：

    ```go
    // swagger:parameters recipes newRecipe
    type Recipe struct {
         //swagger:ignore
         ID string `json:"id"`
         Name string `json:"name"`
         Tags []string `json:"tags"`
         Ingredients []string `json:"ingredients"`
         Instructions []string `json:"instructions"`
         PublishedAt time.Time `json:"publishedAt"`
    }
    ```

    使用 `swagger` generate 命令生成规范并在 **Swagger UI** 上加载结果。

    您现在可以直接从 Swagger UI 填写配方字段来发出 POST 请求。

# *第三章* – 使用 MongoDB 管理数据持久性

1.  您可以使用 `collection.DeleteOne()` 或 `collection.DeleteMany()` 删除配方。在这里，您将 `bson.D({})` 作为过滤器参数传递，这将匹配集合中的所有文档。

    按照以下方式更新 `DeleteRecipeHandler`：

    ```go
    func (handler *RecipesHandler) DeleteRecipeHandler(c *gin.Context) {
       id := c.Param("id")
       objectId, _ := primitive.ObjectIDFromHex(id)
       _, err := handler.collection.DeleteOne(handler.ctx, bson.M{
           "_id": objectId,
       })
       if err != nil {
           c.JSON(http.StatusInternalServerError,  	 	 	              gin.H{"error": err.Error()})
           return
       }
       c.JSON(http.StatusOK, gin.H{"message": "Recipe has  	                               been deleted"})
    }
    ```

    确保按照以下方式在 `DELETE /recipes/{id}` 资源上注册处理器：

    ```go
    router.DELETE("/recipes/:id", recipesHandler.DeleteRecipeHandler)
    ```

1.  要查找配方，您需要一个过滤器文档以及一个指向可以解码结果的值的指针。要查找单个配方，请使用 `collection.FindOne()`。此方法返回单个结果，可以解码到 Recipe 结构体。您将使用与更新查询中相同的过滤器变量来匹配 ID 为 `600dcc85a65917cbd1f201b0` 的配方。

    在 `main.go` 文件上注册处理器：

    ```go
    router.GET("/recipes/:id", recipesHandler.GetOneRecipeHandler)
    ```

    然后，在 `handler.go` 中声明 `GetOneRecipeHandler`，内容如下：

    ```go
    func (handler *RecipesHandler) GetOneRecipeHandler(c *gin.Context) {
       id := c.Param("id")
       objectId, _ := primitive.ObjectIDFromHex(id)
       cur := handler.collection.FindOne(handler.ctx, bson.M{
           "_id": objectId,
       })
       var recipe models.Recipe
       err := cur.Decode(&recipe)
       if err != nil {
           c.JSON(http.StatusInternalServerError, 	 	 	              gin.H{"error": err.Error()})
           return
       }
       c.JSON(http.StatusOK, recipe)
    }
    ```

1.  MongoDB 中的 JSON 文档以称为 **BSON**（**二进制编码的 JSON**）的二进制表示形式存储。此格式包括如下的附加类型：

    a. 双精度

    b. 字符串

    c. 对象

    d. 数组

    e. 二进制数据

    f. 未定义

    g. 对象 ID

    h. 布尔

    i. 日期

    j. 空值

    这使得应用程序能够更可靠地处理、排序和比较数据。

1.  **最近最少使用**（**LRU**）算法使用最近过去来近似近未来。它简单地删除了最长时间未被使用的键。

# *第四章* – 构建 API 身份验证

1.  为了创建用户或注册他们，我们需要定义一个带有 `SignUpHandle`r 的 HTTP 处理器，如下所示：

    ```go
    func (handler *AuthHandler) SignUpHandler(c *gin.Context) {
       var user models.User
       if err := c.ShouldBindJSON(&user); err != nil {
           c.JSON(http.StatusBadRequest, gin.H{"error":                                            err.Error()})
           return
       }
       cur := handler.collection.FindOne(handler.ctx, bson.M{
           "username": user.Username,
       })
       if curTalent.Err() == mongo.ErrNoDocuments {
           err := handler.collection.InsertOne(handler.ctx,  	                                           user)
           if err != nil {
               c.JSON(http.StatusInternalServerError, 	 	                  gin.H{"error": err.Error()})
               return
           }
           c.JSON(http.StatusAccepted, gin.H{"message": 	 	              "Account has been created"})
       }
       c.JSON(http.StatusInternalServerError, gin.H{"error":  	          "Username already taken"})
    }
    Then, register the handler on POST /signup route:
    router.POST("/signup", authHandler.SignUpHandler)
    ```

    为了确保用户名字段在所有用户的条目中都是唯一的，你可以为用户名字段创建一个唯一索引。

1.  定义一个具有以下主体的 `ProfileHandler`：

    ```go
    func (handler *AuthHandler) ProfileHandler(c *gin.Context) {
       var user models.User
       username, _ := c.Get("username")
       cur := handler.collection.FindOne(handler.ctx, bson.M{
           "username": user.Username,
       })
       cur.Decode(&user)
       c.JSON(http.StatusAccepted, user)
    }
    Register the HTTP handler on the router group as below:
    authorized := router.Group("/")
    authorized.Use(authHandler.AuthMiddleware()){
           authorized.POST("/recipes",                        recipesHandler.NewRecipeHandler)
           authorized.PUT("/recipes/:id",                       recipesHandler.UpdateRecipeHandler)
           authorized.DELETE("/recipes/:id",                       recipesHandler.DeleteRecipeHandler)
           authorized.GET("/recipes/:id",                       recipesHandler.GetOneRecipeHandler)
           authorized.GET("/profile",                       authHandler.ProfileHandler)
    }
    ```

1.  在 `SignOutHandler` 签名上方添加以下 Swagger 注解：

    ```go
    // swagger:operation POST /signout auth signOut
    // Signing out
    // ---
    // responses:
    //     '200':
    //         description: Successful operation
    func (handler *AuthHandler) SignOutHandler(c *gin.Context) {}
    ```

# *第五章* – 在 Gin 中提供静态 HTML

1.  创建一个包含以下内容的 `header.tmpl` 文件：

    ```go
    <head>
        <title>Recipes</title>
        <link rel="stylesheet" href="/assets/css/app.css">
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.0-beta2/dist/css/bootstrap.min.css" rel="stylesheet">
    </head>
    ```

    然后，使用以下代码块在 `recipe.tmpl` 中引用该文件：

    ```go
    {{template "/templates/header.tmpl.tmpl"}}
    ```

    按照相同的方法创建一个可重复使用的模板用于页脚部分。

1.  `NewRecipe.js` 组件的完整源代码可在 GitHub 仓库中找到，位于 *第五章* 的 *在 Gin 中提供静态 HTML* 文件夹下。

1.  通过设置指定目标操作系统和架构的必要环境变量来实现交叉编译。我们使用变量 `GOOS` 来指定目标操作系统，使用 `GOARCH` 来指定目标架构。要构建可执行文件，命令将采取以下形式：

    ```go
    GOOS=target-OS GOARCH=target-architecture go build –o main *.go
    ```

    例如，为了构建 Windows 的二进制文件，你可以使用以下命令：

    ```go
    GOOS=windows GOARCH=amd64 go build –o main main.go
    ```

# *第七章* – 测试 Gin HTTP 路由

1.  在 `main_test.go` 中定义 `TestUpdateRecipeHandler` 如下：

    ```go
    func TestUpdateRecipeHandler(t *testing.T) { 
       ts := httptest.NewServer(SetupServer()) 
       defer ts.Close() 

       recipe := Recipe{ 
           ID:   "c0283p3d0cvuglq85log", 
           Name: "Oregano Marinated Chicken", 
       } 

       raw, _ := json.Marshal(recipe) 
       resp, err := http.PUT(fmt.Sprintf("%s/recipes/%s", ts.URL, recipe.ID), bytes.NewBuffer(raw)) 
       defer resp.Body.Close() 
       assert.Nil(t, err) 
       assert.Equal(t, http.StatusOK, resp.StatusCode) 
       data, _ := ioutil.ReadAll(resp.Body) 

       var payload map[string]string 
       json.Unmarshal(data, &payload) 

       assert.Equal(t, payload["message"], "Recipe has been updated") 
    } 
    Define TestDeleteRecipeHandler in main_test.go as follows: 
    func TestDeleteRecipeHandler(t *testing.T) { 
       ts := httptest.NewServer(SetupServer()) 
       defer ts.Close() 

       resp, err := http.DELETE(fmt.Sprintf("%s/recipes/c0283p3d0cvuglq85log", ts.URL)) 
       defer resp.Body.Close() 
       assert.Nil(t, err) 
       assert.Equal(t, http.StatusOK, resp.StatusCode) 
       data, _ := ioutil.ReadAll(resp.Body) 

       var payload map[string]string 
       json.Unmarshal(data, &payload) 

       assert.Equal(t, payload["message"],                 "Recipe has been deleted") 
    } 
    ```

1.  在 `main_test.go` 中定义 `TestFindRecipeHandler` 如下：

    ```go
    func TestDeleteRecipeHandler(t *testing.T) { 
       ts := httptest.NewServer(SetupServer()) 
       defer ts.Close() 

       resp, err := http.DELETE(fmt.Sprintf("%s/recipes          /c0283p3d0cvuglq85log", ts.URL)) 
       defer resp.Body.Close() 
       assert.Nil(t, err) 
       assert.Equal(t, http.StatusOK, resp.StatusCode) 
       data, _ := ioutil.ReadAll(resp.Body) 

       var payload map[string]string 
       json.Unmarshal(data, &payload) 

       assert.Equal(t, payload["message"],                 "Recipe has been deleted") 
    }
    ```

1.  在 `main_test.go` 中定义 `TestFindRecipeHandler` 如下：

    ```go
    func TestFindRecipeHandler(t *testing.T) { 
       ts := httptest.NewServer(SetupServer()) 
       defer ts.Close() 

       expectedRecipe := Recipe{ 
           ID:   "c0283p3d0cvuglq85log", 
           Name: "Oregano Marinated Chicken", 
           Tags: []string{"main", "chicken"}, 
       } 

       resp, err := http.GET(fmt.Sprintf("%s/recipes/c0283p3d0cvuglq85log", ts.URL)) 
       defer resp.Body.Close() 
       assert.Nil(t, err) 
       assert.Equal(t, http.StatusOK, resp.StatusCode) 
       data, _ := ioutil.ReadAll(resp.Body) 

       var actualRecipe Recipe 
       json.Unmarshal(data, &actualRecipe) 

       assert.Equal(t, expectedRecipe.Name,                 actualRecipe.Name) 
       assert.Equal(t, len(expectedRecipe.Tags), len(actualRecipe.Tags)) 
    }
    ```

# *第八章* – 在 AWS 上部署应用程序

1.  使用以下命令创建一个 Docker 卷：

    ```go
    docker volume create mongodata
    ```

    然后，在运行 Docker 容器时挂载该卷：

    ```go
    docker run -d -p 27017:27017 -v mongodata:/data/db --name mongodb mongodb:4.4.3
    ```

1.  要部署 RabbitMQ，可以使用 docker-compose.yml 部署基于 RabbitMQ 官方镜像的附加服务，如下所示：

    ```go
    rabbitmq:
         image: rabbitmq:3-management
         ports:
           - 8080:15672
         environment:
           - RABBITMQ_DEFAULT_USER=admin
           - RABBITMQ_DEFAULT_PASS=password
    ```

1.  以 Kubernetes 机密的形式创建用户的凭证：

    ```go
    mongodb-deployment.yaml to use the Kubernetes secret:

    ```

    apiVersion: apps/v1

    kind: Deployment

    metadata:

    annotations:

    kompose.cmd: kompose convert

    kompose.version: 1.22.0 (955b78124)

    creationTimestamp: null

    labels:

    io.kompose.service: mongodb

    name: mongodb

    spec:

    replicas: 1

    selector:

    matchLabels:

        io.kompose.service: mongodb

    strategy: {}

    template:

    metadata:

        annotations:

        kompose.cmd: kompose convert

        kompose.version: 1.22.0 (955b78124)

        creationTimestamp: null

        标签：

        io.kompose.service: mongodb

    规范：

        容器：

        - 环境变量：

            - 名称：MONGO_INITDB_ROOT_PASSWORD

                值来源：

                密钥引用：

                    名称：mongodb-password

                    键：password

            - 名称：MONGO_INITDB_ROOT_USERNAME

                值：admin

            镜像：mongo:4.4.3

            名称：mongodb

            端口配置：

            - 容器端口：27017

            资源：{}

        重启策略：Always

    状态：{}

    ```go

    ```

1.  使用 `kubectl` 缩放 API pods，请执行以下命令：

    ```go
    kubectl scale deploy
    ```

# *第九章* – 实施 CI/CD 管道

1.  管道将包含以下阶段：

    a. 从 GitHub 仓库检出源代码。

    b. 使用 `npm install` 命令安装 NPM 包。

    c. 使用 `npm run build` 命令生成资源。

    d. 安装 AWS CLI 并将新资源推送到 S3 存储桶。

    e. `config.yml` 配置如下：

    ```go
    version: 2.1
    executors:
     environment:
       docker:
         - image: node:lts
       working_directory: /dashboard
    jobs:
     build:
       executor: environment
       steps:
         - checkout
         - restore_cache:
             key: node-modules-{{checksum "package.json"}}
         - run:
             name: Install dependencies
             command: npm install
         - save_cache:
             key: node-modules-{{checksum "package.json"}}
             paths:
               - node_modules
         - run:
             name: Build artifact
             command: CI=false npm run build
         - persist_to_workspace:
             root: .
             paths:
               - build
     deploy:
       executor: environment
       steps:
         - attach_workspace:
             at: dist
         - run:
             name: Install AWS CLI
             command: |
               apt-get update
               apt-get install -y python3-pip
               pip3 install awscli
         - run:
             name: Push to S3 bucket
             command: |
               cd dist/build/dashboard/
               aws configure set preview.cloudfront true
               aws s3 cp --recursive . s3://YOUR_S3_BUCKET/ --region YOUR_AWS_REGION
    workflows:
     ci_cd:
       jobs:
         - build
         - deploy:
             requires:
               - build
             filters:
               branches:
                 only:
                   - master
    ```

    在运行管道之前，您需要给 **CircleCI IAM** 用户授予 `S3:PutObject` 访问权限。

1.  您可以配置 Slack ORB，以便在管道成功时发送通知，如下所示：

    ```go
    - slack/notify:
         event: pass
         custom: |
          {
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "Current Job: $CIRCLE_JOB"
                }
              },
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "New release has been successfully  	                       deployed!"
                }
              }
             ]
          }
    ```
