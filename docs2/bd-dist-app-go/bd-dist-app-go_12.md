# 第九章：实现 CI/CD 管道

本章将向您展示如何构建一个 CI/CD 工作流程来自动化 Gin 服务的部署。我们还将讨论在构建基于 Gin 的 API 时采用 GitFlow 方法的重要性。

在本章中，我们将涵盖以下主题：

+   探索 CI/CD 实践

+   构建持续集成工作流程

+   维护多个运行时环境

+   实现持续交付

到本章结束时，您将能够自动化 Gin 网络应用的测试、构建和部署过程。

# 技术要求

为了跟随本章的内容，您需要以下条件：

+   对上一章内容的完整理解。本章是上一章的后续，因为它将使用相同的源代码。因此，为了避免重复，一些代码片段将不会进行解释。

+   推荐您有 CI/CD 实践的前期经验，这样您就可以轻松地跟随本章内容。

本章的代码包托管在 GitHub 上，网址为[`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter09`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter09)。

# 探索 CI/CD 实践

在前面的章节中，您学习了如何在 AWS 上设计、构建和部署 Gin 网络应用。目前，部署新更改可能是一个耗时的过程。当部署到 EC2 实例、Kubernetes 或**平台即服务**（**PaaS**）时，涉及一些手动步骤，这些步骤有助于将新更改推出去。

幸运的是，许多这些部署步骤可以自动化，从而节省开发时间，消除人为错误的可能性，并减少发布周期时间。这就是为什么在本节中，您将学习如何采用**持续集成**（**CI**）、**持续部署**（**CD**）和**持续交付**来加速您应用程序的**上市时间**（**TTM**），以及确保每个迭代都发送高质量的功能。但首先，这些实践意味着什么？

## 持续集成

**持续集成**（**CI**）是一个拥有集中式代码仓库（例如，GitHub、Bitbucket、GitLab 等）的过程，并且所有更改和功能在集成到远程仓库之前都需要通过一个管道。一个经典的管道会在代码提交（或推送事件）发生时触发构建，运行预集成测试，构建工件（例如，Docker 镜像、JAR、npm 包等），并将结果推送到私有注册库进行版本控制：

![图 9.1 – CI/CD 实践![图 9.1 – CI/CD 实践](img/Figure_9.1_B17115.jpg)

图 9.1 – CI/CD 实践

如前图所示，CI 工作流程包括以下阶段：**检出**、**测试**、**构建**和**推送**。

## 持续部署

**持续部署**（**CD**）另一方面，是 CI 工作流的扩展。每个通过 CI 管道所有阶段的更改都会自动发布到预生产或预生产环境，QA 团队可以在那里运行验证和验收测试。

## 持续交付

**持续交付**与 CD 类似，但在将新版本部署到生产环境之前需要人工干预或业务验证。这种人工参与可能包括手动部署，通常由 QA 工程师执行，或者简单地点击一个按钮。这与 CD 不同，CD 中每个成功的构建都会发布到预生产环境。

采用这三个实践可以帮助提高代码的质量和可测试性，并有助于降低将损坏的发布版本发送到生产的风险。

现在你已经了解了这三个组件，到本章结束时，你将能够为我们的 Gin Web 应用程序构建一个端到端的部署流程，类似于以下图中所示：

![Figure 9.2 – CI/CD pipeline![Figure 9.2 – CI/CD pipeline](img/Figure_9.2_B17115.jpg)

Figure 9.2 – CI/CD pipeline

前面的管道分为以下阶段：

+   **检出**：从项目的 GitHub 仓库中拉取最新更改。

+   **测试**：在 Docker 容器内运行单元和质量测试。

+   **构建**：从 Dockerfile 编译和构建 Docker 镜像。

+   **推送**：标记镜像并将其存储在私有注册表中。

+   **部署**：将更改部署并提升到 AWS 环境（EC2、EKS 或 ECS）。

    注意

    我们使用 CircleCI 作为 CI 服务器，但可以使用其他 CI 解决方案（如 Jenkins、Travis CI、GitHub Actions 等）实现相同的流程。

# 构建 CI 工作流

我们为这本书构建的应用程序在 GitHub 仓库中进行版本控制。此仓库使用 GitFlow 模型作为分支策略，其中使用了三个主要分支。每个分支代表应用程序的一个运行环境：

+   **主分支**：此分支对应于在生产环境中运行的代码。

+   **预生产分支**：预生产环境，是生产环境的镜像。

+   **开发分支**：沙盒或开发环境。

要将应用程序从一个环境提升到另一个环境，你可以创建**功能分支**。你也可以为重大错误或问题创建**热修复分支**。

注意

要了解更多关于 GitFlow 工作流和最佳实践的信息，请查看官方文档：[`www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow`](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)。

下一个图显示了你的项目 GitHub 仓库的外观：

![Figure 9.3 – 项目 GitHub 仓库](img/Figure_9.3_B17115.jpg)

![Figure 9.3 – Project's GitHub repository](img/Figure_9.3_B17115.jpg)

Figure 9.3 – 项目 GitHub 仓库

您将使用 **CircleCI**来自动化 CI/CD 工作流程。如果您还没有 CircleCI 账户，请使用您的 GitHub 账户免费注册，网址为 https://circleci.com。无论 CI 服务器是什么，CI/CD 的原则都是相同的：

![图 9.4 – CircleCI 登录页面![图片](img/Figure_9.4_B17115.jpg)

图 9.4 – CircleCI 登录页面

一旦注册，您需要配置 CircleCI 以运行应用程序测试和构建 Docker 镜像。为此，您需要在模板文件中描述所有步骤，并将其保存在代码的 GitHub 仓库中。这种方法被称为 **代码即管道**。

## 代码即管道

当 CircleCI 构建被触发时，它会寻找一个 `.circleci/config.yml` 文件。此文件包含要在 CI 服务器上执行的指令。

首先创建一个 `.circleci` 文件夹和一个包含以下内容的 `.config.yml` 文件：

```go
version: 2.1
executors:
 environment:
   docker:
     - image: golang:1.15.6
   working_directory: 
jobs:
 test:
   executor: environment
   steps:
     - checkout
workflows:
 ci_cd:
   jobs:
     - test
```

以下代码片段将在由 Golang v1.15.6 Docker 镜像提供的环境中运行工作流程。大多数 CI/CD 步骤都将使用 Docker 执行，这使得在本地运行构建变得轻而易举，并且如果将来想要迁移到不同的 CI 服务器，我们仍然有选择余地（与供应商锁定相反）。首先运行的工作是测试阶段，它包括以下步骤：

```go
jobs:
 test:
   executor: environment
   steps:
     - checkout
     - restore_cache:
         keys:
           - go-mod-v4-{{ checksum "go.sum" }}
     - run:
         name: Install Dependencies
         command: go mod download
     - save_cache:
         key: go-mod-v4-{{ checksum "go.sum" }}
         paths:
           - "/go/pkg/mod"
     - run:
         name: Code linting
         command: >
           go get -u golang.org/x/lint/golint
           golint ./...
     - run:
         name: Unit tests
         command: go test -v ./...
```

`test` 作业将使用 `checkout` 指令从该项目的 GitHub 仓库获取最新更改。然后，它将下载项目依赖项并将它们缓存以供将来使用（以减少工作流程持续时间），之后，它将运行一系列测试：

+   **代码检查**：这会检查代码是否遵守标准编码约定。

+   **单元测试**：这会执行我们在前几章中编写的单元测试。

在 CircleCI 配置就绪后，让我们在 CircleCI 上为 Gin 应用程序创建一个项目。为此，请按照以下步骤操作：

1.  跳转到 CircleCI 控制台，点击项目仓库旁边的 **设置项目**：![图 9.5 – 设置 CircleCI 项目    ![图片](img/Figure_9.5_B17115.jpg)

    图 9.5 – 设置 CircleCI 项目

1.  点击 **使用现有配置** 按钮，因为我们已经有了 CircleCI 配置，然后点击 **开始构建**：![图 9.6 – CircleCI 配置    ![图片](img/Figure_9.6_B17115.jpg)

    图 9.6 – CircleCI 配置

1.  将启动一个新的管道；然而，由于代码仓库中不存在 `config.yml` 文件，它将失败。以下截图显示了此错误：![图 9.7 – 管道失败    ![图片](img/Figure_9.7_B17115.jpg)

    图 9.7 – 管道失败

1.  通过运行以下命令将 CircleCI 配置推送到 develop 分支上的 GitHub 仓库：

    ```go
    git add .
    git commit –m "added test stage"
    git push origin develop
    ```

    将自动触发一个新的管道。输出将类似于以下内容：

    ![图 9.8 – 管道已自动触发    ![图片](img/Figure_9.8_B17115.jpg)

    图 9.8 – 管道已自动触发

1.  点击 `config.yml` 文件。所有测试都将通过，您将能够构建您的 Docker 镜像：![图 9.9 – 运行自动化测试    ](img/Figure_9.9_B17115.jpg)

    图 9.9 – 运行自动化测试

    值得注意的是，该管道是自动触发的，因为在设置 CircleCI 项目时，项目 GitHub 仓库中自动创建了一个 webhook。这样，对于每次推送事件，都会向 CircleCI 服务器发送通知以触发相应的 CircleCI 管道：

    ![图 9.10 – GitHub Webhook    ](img/Figure_9.10_B17115.jpg)

    图 9.10 – GitHub Webhook

    让我们继续到集成应用程序的下一步，即构建 Docker 镜像。

1.  将 `build` 作业添加到 CI/CD 工作流程中：

    ```go
    workflows:
     ci_cd:
       jobs:
         - test
         - build
    ```

1.  `build` 作业负责根据我们存储在代码仓库中的 `Dockerfile` 构建一个 Docker 镜像。然后，它对构建的镜像进行标记并将其存储在远程 Docker 仓库中进行版本控制：

    ```go
    build:
        executor: environment
       steps:
         - checkout
         - setup_remote_docker:
            version: 19.03.13
         - run:
             name: Build image
             command: >
               TAG=0.1.$CIRCLE_BUILD_NUM
               docker build -t USER/recipes-api:$TAG .
         - run:
             name: Push image
             command: >
               docker tag USER/recipes-api:$TAG 
                   ID.dkr.ecr.REGION.amazonaws.com/USER/
                   recipes-api:$TAG
               docker tag USER/recipes-api:$TAG 
                   ID.dkr.ecr.REGION.amazonaws.com/USER/
                   recipes-api:develop
               docker push ID.dkr.ecr.REGION.amazonaws.com/
                   USER/recipes-api:$TAG
               docker push ID.dkr.ecr.REGION.amazonaws.com/
                   USER/recipes-api:develop
    ```

    注意

    如果您对 Dockerfile 感兴趣，可以在本书的 GitHub 仓库中找到它：[`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter10/Dockerfile`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/blob/main/chapter10/Dockerfile)。

    为了标记镜像，我们将使用语义版本控制（[`semver.org`](https://semver.org)）。版本的格式是三个数字，由点分隔：

    ![图 9.11 – 语义版本控制    ](img/Figure_9.11_B17115.jpg)

    图 9.11 – 语义版本控制

    当新更改破坏 API（向后不兼容的更改）时，主版本号会增加。当发布新功能时，次要版本号会增加，而补丁版本号会增加以修复错误。

    在 CircleCI 配置中，您使用 `$CIRCLE_BUILD_NUM` 环境变量为通过我们 Gin 应用程序的开发周期构建的每个 Docker 镜像创建一个唯一的版本。另一种选择是使用 `CIRCLE_SHA1` 变量，它是触发 CI 构建的 Git 提交的 SHA1 哈希。

1.  一旦镜像被标记，就将其存储在私有仓库中。在先前的示例中，您使用 **弹性容器注册库**（**ECR**）作为私有仓库，但也可以使用其他解决方案，如 DockerHub。

1.  使用以下命令将更改推送到 develop 分支：

    ```go
    git add .
    git commit –m "added build stage"
    git push origin develop
    ```

将会触发一个新的管道。一旦 `test` 作业完成，`build` 作业将被执行，如下面的截图所示：

![图 9.12 – 运行 "build" 作业](img/Figure_9.12_B17115.jpg)

图 9.12 – 运行 "build" 作业

以下为 `build` 作业的步骤。测试应该在 **推送镜像** 步骤失败，因为 CircleCI 由于缺少 AWS 权限而无法将镜像推送到 ECR：

![图 9.13 – 推送镜像步骤](img/Figure_9.13_B17115.jpg)

图 9.13 – 推送镜像步骤

让我们看看如何配置您的 CI 和 CD 工作流程：

1.  为了允许 CircleCI 与您的 ECR 仓库交互，创建一个具有适当 IAM 策略的专用 IAM 用户。

1.  跳转到 AWS 管理控制台([`console.aws.amazon.com/`](https://console.aws.amazon.com/))，导航到**身份与访问管理**（**IAM**）控制台。然后，为 CircleCI 创建一个新的 IAM 用户。勾选**程序访问**复选框，如图 9.14 所示：![图 9.14 – CircleCI IAM 用户    ](img/Figure_9.14_B17115.jpg)

    图 9.14 – CircleCI IAM 用户

1.  将以下 IAM 策略附加到 IAM 用户。此声明允许 CircleCI 将 Docker 镜像推送到 ECR 仓库。确保根据需要替换`ID`：

    ```go
    {
        "Version": "2008-10-17",
        "Statement": [
            {
                "Sid": "AllowPush",
                "Effect": "Allow",
                "Principal": {
                    "AWS": [
                        "arn:aws:iam::ID:circleci/
                                      push-pull-user-1"
                    ]
                },
                "Action": [
                    "ecr:PutImage",
                    "ecr:InitiateLayerUpload",
                    "ecr:UploadLayerPart",
                    "ecr:CompleteLayerUpload"
                ]
            }
        ]
    }
    ```

1.  IAM 用户创建完成后，从**安全凭证**选项卡创建一个访问密钥。然后，返回 CircleCI 仪表板并跳转到**项目设置**：![图 9.15 – CircleCI 环境变量    ](img/Figure_9.15_B17115.jpg)

    图 9.15 – CircleCI 环境变量

1.  在**环境变量**部分，点击**添加环境变量**按钮并添加以下变量：

    **AWS_ACCESS_KEY_ID**：指定与 CircleCI IAM 用户关联的 AWS 访问密钥。

    **AWS_SECRET_ACCESS_KEY**：指定与 CircleCI IAM 用户关联的 AWS 秘密访问密钥。

    **AWS_DEFAULT_REGION**：指定 ECR 仓库所在的 AWS 区域：

    ![图 9.16 – 将 AWS 凭证作为环境变量    ](img/Figure_9.16_B17115.jpg)

    图 9.16 – 将 AWS 凭证作为环境变量

1.  环境变量设置完成后，通过添加一个用于在执行`docker push`命令之前进行 ECR 身份验证的指令来更新 CircleCI 配置。新的更改将如下所示：

    ```go
        -run:
            name: Push image
            command: |
                TAG=0.1.$CIRCLE_BUILD_NUM
               docker tag USER/recipes-api:$TAG 
                   ID.dkr.ecr.REGION.amazonaws.com/USER/
                   recipes-api:$TAG
               docker tag USER/recipes-api:$TAG ID.dkr.ecr. 
                   REGION.amazonaws.com/USER/
                   recipes-api:develop
               aws ecr get-login-password --region REGION | 
                   docker login --username AWS --password-
                   stdin ID.dkr.ecr.REGION.amazonaws.com
               docker push ID.dkr.ecr.REGION.amazonaws.com
                   /USER/recipes-api:$TAG
        docker push ID.dkr.ecr.REGION.amazonaws.com
                   /USER/recipes-api:develop
    ```

1.  将`USER`、`ID`和`REGION`变量适当地替换为您自己的值，并将更改再次推送到远程仓库的 develop 分支下：

    ```go
    USER, ID, and REGION values in the configuration file, you can pass their values as environment variables from the CircleCI project settings.A new pipeline will be triggered. This time, the `build` job should be successful, and you should see something similar to the following:![Figure 9.17 – Building and storing a Docker image with CircleCI    ](img/Figure_9.17_B17115.jpg)Figure 9.17 – Building and storing a Docker image with CircleCINow, click on the `develop` tag that points to the latest built image in the develop branch.With the test and build stages being automated, you can go even further and automate the deployment process as well. 
    ```

1.  与之前的职位类似，将一个`deploy`任务添加到当前的工作流程中：

    ```go
    workflows:
     ci_cd:
       jobs:
         - test
         - build
         - deploy
    ```

    `deploy`任务将简单地使用我们在上一章中介绍的`docker-compose.yml`文件，在 EC2 实例上部署应用程序堆栈。文件将如下所示（为了简洁，已裁剪完整的 YAML 文件）：

    ```go
    version: "3" 
    services:
      api:
        image: ID.dkr.ecr.REGION.amazonaws.com/USER/
               recipes-api:develop
        environment:
          - MONGO_URI=mongodb://admin:password@mongodb
                :27017/test?authSource=admin
                 &readPreference=primary&ssl=false
          - MONGO_DATABASE=demo
          - REDIS_URI=redis:6379
        external_links:
          - mongodb
          - redis
      redis:
        image: redis
      mongodb:
        image: mongo:4.4.3
        environment:
          - MONGO_INITDB_ROOT_USERNAME=admin
          - MONGO_INITDB_ROOT_PASSWORD=password
    ```

1.  要将新更改部署到运行容器的 EC2 实例，您将需要 SSH 到远程服务器并执行两个`docker-compose`命令 – `pull`和`up`：

    ```go
    deploy:
       executor: environment
       steps:
         - checkout
         - run:
             name: Deploy with Docker Compose
             command: |
               ssh -oStrictHostKeyChecking=no ec2-user@IP 
                   docker-compose pull && docker-compose up –d
    ```

1.  确保将`IP`变量替换为运行沙盒环境的 EC2 实例的 IP 地址或 DNS 名称。

1.  要 SSH 到 EC2 实例，将用于在 AWS 中部署 EC2 实例的 SSH 密钥对添加到 CircleCI 项目设置中。在**SSH 密钥**下，点击**添加 SSH 密钥**并粘贴 SSH 密钥对的内容：![图 9.20 – 添加 SSH 密钥对    ](img/Figure_9.20_B17115.jpg)

    图 9.20 – 添加 SSH 密钥对

1.  使用以下命令提交并推送新的 CircleCI 配置到 GitHub：

    ```go
    git add .
    git commit –m "added deploy stage"
    git push origin develop
    ```

    将会触发一个新的管道，测试、构建和部署作业将依次执行。在**部署**作业结束时，新构建的镜像将被部署到沙盒环境中：

![图 9.21 – 持续部署![图片](img/Figure_9.21_B17115.jpg)

图 9.21 – 持续部署

如果您正在运行**弹性容器服务**（**ECS**），您可以使用以下 CircleCI 配置来强制 ECS 拉取新镜像：

```go
version: 2.1
orbs:
 aws-ecs: circleci/aws-ecs@0.0.11
workflows:
ci_cd:
   jobs:
    - test
    - build
    - aws-ecs/deploy-service-update:
       aws-region: AWS_DEFAULT_REGION
       family: 'demo'
       cluster-name: 'sandbox'
       container-image-name-updates: 
           'container=api,tag=0.1.${CIRCLE_BUILD_NUM}'
```

注意

确保您为 CircleCI IAM 用户分配 ECS 权限，以成功执行任务更新。

主要变化是我们正在使用 CircleCI orbs 而不是 Docker 镜像作为运行环境。通过使用 orbs，我们可以使用预构建的命令，这减少了我们配置文件中的代码行数。部署作业将部署更新的镜像到沙盒 ECS 集群。

如果您正在运行 Kubernetes，您可以使用以下 CircleCI 规范文件来更新镜像：

```go
version: 2.1
orbs:
   aws-eks: circleci/aws-eks@0.2.0
   kubernetes: circleci/kubernetes@0.3.0
jobs:
 deploy:
   executor: aws-eks/python3
   steps:
     - checkout
     - aws-eks/update-kubeconfig-with-authenticator:
         cluster-name: sandbox
         install-kubectl: true
         aws-region: AWS_REGION
     - kubernetes/create-or-update-resource:
         resource-file-path: "deployment/
         api.deployment.yaml"
         get-rollout-status: true
         resource-name: deployment/api
     - kubernetes/create-or-update-resource:
         resource-file-path: "deployment/api.service.yaml"

workflows:
 ci_cd:
  jobs:
    - test
    - build
    - deploy
```

在配置了 CI 和 CD 工作流程后，您可以通过为 Gin RESTful API 构建新功能来测试它们。您可以按照以下步骤操作：

1.  更新`main.go`文件，并使用 Gin 路由器在`/version`资源上公开一个新的端点。该端点将显示运行的 API 版本：

    ```go
    router.GET("/version", VersionHandler)
    ```

    HTTP 处理器是自解释的；它返回`API_VERSION`环境变量的值：

    ```go
    func VersionHandler(c *gin.Context) {
       c.JSON(http.StatusOK, gin.H{"version": os.Getenv(
           "API_VERSION")})
    }
    ```

1.  要动态注入环境变量，您可以使用 Docker 参数功能，这允许您在构建时传递值。更新我们的`Dockerfile`并声明`API_VERSION`为构建参数和环境变量：

    ```go
    FROM golang:1.16
    WORKDIR /go/src/github.com/api
    COPY . .
    RUN go mod download
    RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
    FROM alpine:latest 
    ARG API_VERSION
    ENV API_VERSION=$API_VERSION
    RUN apk --no-cache add ca-certificates
    WORKDIR /root/
    COPY --from=0 /go/src/github.com/api/app .
    CMD ["./app"]
    ```

1.  接下来，通过注入`$TAG`变量作为`API_VERSION`构建参数的值来更新`构建镜像`步骤：

    ```go
    run:
       name: Build image
        command: |
          TAG=0.1.$CIRCLE_BUILD_NUM
          docker build -t mlabouardy/
              recipes-api:$TAG --build-arg API_VERSION=${TAG} .
    ```

1.  使用以下代码将新更改推送到 develop 分支：

    ```go
    deploy job will run and deploy the application on EC2:![Figure 9.22 – Deploying Docker Stack    ](img/Figure_9.22_B17115.jpg)Figure 9.22 – Deploying Docker Stack
    ```

1.  要测试新更改，导航到实例 IP 地址，并将您的浏览器指向`/api/version`资源路径：

![图 9.23 – API 运行版本![图片](img/Figure_9.23_B17115.jpg)

图 9.23 – API 运行版本

这将返回 Gin RESTful API 运行的 Docker 镜像版本。

值得注意的是，当前的 CircleCI 配置不能保证作业总是按相同的顺序运行（测试 -> 构建 -> 部署）。为了保持 CI/CD 顺序，使用`requires`关键字，如下所示：

```go
workflows:
 ci_cd:
   jobs:
     - test
     - build:
       requires:
           - test
     - deploy:
       requires:
           - test
           - build
```

这样，您可以确保只有当测试和构建作业都成功时，部署作业才会执行：

![图 9.24 – CI/CD 工作流程![图片](img/Figure_9.24_B17115.jpg)

![图 9.24 – CI/CD 工作流程太棒了！现在，我们为我们的 Gin 应用程序拥有了一个完整的 CI/CD 管道。# 维护多个运行环境在实际场景中，您需要多个环境来避免在验证之前将损坏的功能或重大错误推送到沙盒或预发布环境（或者更糟，生产环境）。您可以通过运行基于我们在前几章中创建的沙盒环境的新 EC2 实例来创建一个 EC2 实例以托管预发布环境：1.  选择 **沙盒** 实例，从操作栏中点击 **操作**。然后，从 **镜像和模板** 下拉列表中点击 **启动更多类似项**：![图 9.25 – 复制沙盒环境    ](img/Figure_9.25_B17115.jpg)

    图 9.25 – 复制沙盒环境

    此选项将自动将所选实例的配置详细信息填充到 Amazon EC2 启动向导中。

1.  将 `Name` 标签的值更新为 `staging` 并点击 **启动** 以配置实例：![图 9.26 – 在 EC2 实例中运行的预发布环境    ](img/Figure_9.26_B17115.jpg)

    图 9.26 – 在 EC2 实例中运行的预发布环境

1.  一旦实例启动并运行，更新 CircleCI 以根据管道运行的分支名称标记 Docker 镜像。除了通过 `CIRCLE_BUILD_NUM` 环境变量创建的 `dynamic` 标签外，如果当前分支是 develop、preprod 或 master，还应推送一个 `fixed` 标签（develop、preprod 或 master）：

    ```go
    run:
       name: Push image
       command: |
         TAG=0.1.$CIRCLE_BUILD_NUM
         aws ecr get-login-password --region REGION | docker 
             login --username AWS --password-stdin 
             ID.dkr.ecr.REGION.amazonaws.com
         docker tag USER/recipes-api:$TAG 
         ID.dkr.ecr.REGION.amazonaws.com/USER/recipes-api:$TAG
         docker push ID.dkr.ecr.REGION.amazonaws.com/
             USER/recipes-api:$TAG
         if [ "${CIRCLE_BRANCH}" == "master" ] || [ 
             "${CIRCLE_BRANCH}" == "preprod" ] || [ 
             "${CIRCLE_BRANCH}" == "develop" ];
        then
            docker tag USER/recipes-api:$TAG 
                ID.dkr.ecr.REGION.amazonaws.com/USER/
                recipes-api:${CIRCLE_BRANCH}
                 docker push ID.dkr.ecr.REGION.amazonaws.com/
                     USER/recipes-api:${CIRCLE_BRANCH}
          fi
    ```

1.  接下来，更新 `deploy` 作业，以便可以根据当前的 Git 分支名称 SSH 到正确的 EC2 实例 IP 地址：

    ```go
    run:
       name: Deploy with Docker Compose
       command: |
         if [ "${CIRCLE_BRANCH}" == "master" ]
         then
            ssh -oStrictHostKeyChecking=no ec2-user@IP_PROD 
                "docker-compose pull && docker-compose up -d"
        elif [ "${CIRCLE_BRANCH}" == "preprod" ]
         then
            ssh -oStrictHostKeyChecking=no             ec2-user@IP_STAGING 
                "docker-compose pull && docker-compose up -d"
         else
            ssh -oStrictHostKeyChecking=no             ec2-user@IP_SANDBOX 
                "docker-compose pull && docker-compose up -d"
         fi
    ```

    注意

    IP 地址（`IP_PROD`、`IP_STAGING` 和 `IP_SANDBOX`）应在 CircleCI 项目的设置中定义为环境变量。

1.  最后，更新工作流程，以便仅在当前 Git 分支是 `develop`、`preprod` 或 `master` 分支时部署更改：

    ```go
    workflows:
     ci_cd:
       jobs:
         - test
         - build:
             requires:
               - test
         - deploy:
             requires:
               - test
               - build
             filters:
               branches:
                 only:
                   - develop
                   - preprod
                   - master
    ```

1.  使用以下命令在 GitHub 上提交并存储更改：

    ```go
    git add .
    git commit –m "continuous deployment"
    git push origin develop
    ```

    开发分支上会自动触发一个新的管道，更改将部署到沙盒环境：

![图 9.27 – 部署到沙盒环境](img/Figure_9.27_B17115.jpg)

图 9.27 – 部署到沙盒环境

一旦新更改已在沙盒环境中得到验证，您应该准备好将代码提升到预发布环境。

要部署到预发布环境，请按照以下步骤操作：

1.  创建一个 **拉取请求**（**PR**）以将开发分支合并到预生产分支，如下所示：![图 9.28 – 创建拉取请求    ](img/Figure_9.28_B17115.jpg)

    图 9.28 – 创建拉取请求

    注意，PR 已准备好合并，因为所有检查都已通过：

    ![图 9.29 – 成功的 Git 检查    ](img/Figure_9.29_B17115.jpg)

    图 9.29 – 成功的 Git 检查

1.  点击 **合并拉取请求** 按钮。预生产分支将触发一个新的管道，并依次执行三个作业 – **测试**、**构建** 和 **部署**：![图 9.30 – 部署到预发布环境    ](img/Figure_9.30_B17115.jpg)

    图 9.30 – 部署到预发布环境

1.  在 **构建** 阶段的末尾，预生产分支的新镜像以及其 CircleCI 构建号将被存储在 ECR 存储库中，如下所示：![图 9.31 – 预生产 Docker 镜像    ](img/Figure_9.31_B17115.jpg)

    图 9.31 – 预生产 Docker 镜像

1.  现在，使用`docker-compose up`命令将镜像部署到预发布环境。点击预发布环境的 IP 地址；你应该看到正在运行的 Docker 镜像的版本：

![图 9.32 – Docker 镜像版本。](img/Figure_9.32_B17115.jpg)

图 9.32 – Docker 镜像版本。

太好了！你现在有一个预发布环境，可以在将 API 功能提升到生产之前对其进行验证。

到目前为止，你已经学会了如何通过推送事件实现持续部署。然而，在生产环境中，你可能希望在将新版本发布到生产之前添加额外的验证。这就是**持续交付实践**发挥作用的地方。

# 实施持续交付

要将 Gin 应用程序部署到生产环境，你需要启动一个专门的 EC2 实例或 EKS 集群。在部署到生产环境之前，你必须进行手动验证。

使用 CircleCI，你可以使用`pause_workflow`作业与用户交互，在恢复管道之前请求批准。要这样做，请按照以下步骤操作：

1.  在`deploy`作业之前添加`pause_workflow`作业，并定义一个`release`作业，如下所示：

    ```go
    workflows:
     ci_cd:
       jobs:
         - test
         - build:
             requires:
               - test
         - deploy:
             requires:
               - test
               - build
             filters:
               branches:
                 only:
                   - develop
                   - preprod
         - pause_workflow:
             requires:
               - test
               - build
             type: approval
             filters:
               branches:
                 only:
                   - master
         - release:
             requires:
               - pause_workflow
             filters:
               branches:
                 only:
                   - master
    ```

1.  将更改推送到 develop 分支。将触发一个新的管道，并将更改部署到你的沙盒环境。接下来，创建一个拉取请求并合并 develop 和 preprod 分支。现在，提出一个拉取请求以合并 preprod 分支和主分支：![图 9.33 – 向主分支合并拉取请求    ](img/Figure_9.33_B17115.jpg)

    图 9.33 – 向主分支合并拉取请求

1.  一旦 PR 被合并，主分支上将被触发一个新的管道，并且将执行测试和构建作业：![图 9.34 – 在主分支上运行 CI/CD 工作流程    ](img/Figure_9.34_B17115.jpg)

    图 9.34 – 在主分支上运行 CI/CD 工作流程

1.  当达到`pause_workflow`作业时，管道将被暂停，如下所示：![图 9.35 – 请求用户批准    ](img/Figure_9.35_B17115.jpg)

    图 9.35 – 请求用户批准

1.  如果你点击**pause_workflow**框，将弹出一个确认对话框，你可以允许工作流程继续运行：![图 9.36 – 批准部署    ](img/Figure_9.36_B17115.jpg)

    图 9.36 – 批准部署

1.  一旦批准，管道将恢复，部署阶段将被执行。在 CI/CD 管道的末尾，应用程序将被部署到生产环境：

![图 9.37 – 将应用程序部署到生产环境](img/Figure_9.37_B17115.jpg)

图 9.37 – 将应用程序部署到生产环境

太棒了！有了这个，你已经实现了持续交付！

在结束之前，你可以通过在 CircleCI 上触发新构建时向开发团队发送**Slack**通知来改进工作流程，提高意识。

## 通过 Slack 改进反馈循环

你可以使用 **Slack RESTful API** 在 Slack 频道中发布通知，或者使用预构建 Slack 命令的 CircleCI Slack orb 来改进反馈循环。

要这样做，请按照以下步骤操作：

1.  将以下代码块添加到 `test` 作业中：

    ```go
    - slack/notify:
             channel: "#ci"
             event: always
             custom: |
               {
                 "blocks": [
                   {
                     "type": "section",
                     "text": {
                       "type": "mrkdwn",
                       "text": "*Build has started*! 
                                 :crossed_fingers:"
                     }
                   },
                   {
                     "type": "divider"
                   },
                   {
                     "type": "section",
                     "fields": [
                       {
                         "type": "mrkdwn",
                         "text": "*Project*:\
                             n$CIRCLE_PROJECT_REPONAME"
                       },
                       {
                         "type": "mrkdwn",
                         "text": "*When*:\n$(date +'%m/%d/%Y 
                                   %T')"
                       },
                       {
                         "type": "mrkdwn",
                         "text": "*Branch*:\n$CIRCLE_BRANCH"
                       },
                       {
                         "type": "mrkdwn",
                         "text": "*Author*:                              \n$CIRCLE_USERNAME"
                       }
                     ],
                     "accessory": {
                       "type": "image",
                       "image_url": "https://media.giphy.com/
                        media/3orieTfp1MeFLiBQR2/giphy.gif",
                       "alt_text": "CircleCI logo"
                     }
                   },
                   {
                     "type": "actions",
                     "elements": [
                       {
                         "type": "button",
                         "text": {
                           "type": "plain_text",
                           "text": "View Workflow"
                         },
                         "url": "https://circleci.com/
                            workflow-run/${                                     CIRCLE_WORKFLOW_ID}"
                       }
                     ]
                   }
                 ]
               }
    ```

1.  然后，通过导航到 [`api.slack.com/apps`](https://api.slack.com/apps) 并从页面标题点击 **Build app** 来在你的 Slack 工作空间中创建一个新的 Slack 应用程序：![图 9.38 – 新 Slack 应用程序    ](img/Figure_9.38_B17115.jpg)

    图 9.38 – 新 Slack 应用程序

1.  给应用程序一个有意义的名称，然后点击 **创建 App** 按钮。在 **OAuth & Permissions** 页面上，在 **Bot Token Scopes** 下添加以下权限：![图 9.39 – Slack 机器人权限    ](img/Figure_9.39_B17115.jpg)

    图 9.39 – Slack 机器人权限

1.  从左侧导航菜单中，点击 **OAuth & Permissions** 并复制 OAuth token：![图 9.40 – 机器人用户 OAuth token    ](img/Figure_9.40_B17115.jpg)

    图 9.40 – 机器人用户 OAuth token

1.  返回 CircleCI 项目设置，并添加以下环境变量：

    **SLACK_ACCESS_TOKEN**：我们之前生成的 OAuth Token。

    **SLACK_DEFAULT_CHANNEL**：你想要发布 CircleCI 构建通知的 Slack 频道。

    你应该看到以下内容：

    ![图 9.41 – Slack 环境变量    ](img/Figure_9.41_B17115.jpg)

    图 9.41 – Slack 环境变量

1.  将新的 CircleCI 配置更新推送到 GitHub。此时，将执行一个新的管道：

![图 9.42 – 管道开始时的 Slack 通知](img/Figure_9.42_B17115.jpg)

图 9.42 – 管道开始时的 Slack 通知

将发送 Slack 通知到配置的 Slack 频道，包含项目名称和 Git 分支名称：

![图 9.43 – 发送 Slack 通知](img/Figure_9.43_B17115.jpg)

图 9.43 – 发送 Slack 通知

你甚至可以更进一步，根据管道的状态发送通知。例如，如果你想当管道失败时收到警报（为了简洁，已裁剪完整的 JSON），请添加以下代码：

```go
- slack/notify:
     channel: "#ci"
     event: fail
     custom: |
       {
         "blocks": [
           {
             "type": "section",
             "text": {
                "type": "mrkdwn",
                "text": "*Tests failed, run for your 
                         life*! :fearful:"
              }
          ]
       }
```

你可以通过抛出一个不同于 `0` 的代码错误的错误来模拟管道失败：

```go
- run:
     name: Unit tests
     command: |
       go test -v ./...
       exit 1
```

现在，将更改推送到 develop 分支。当管道到达 `单元测试` 步骤时，将抛出错误，并且管道将失败：

![图 9.44 – 以编程方式抛出错误](img/Figure_9.44_B17115.jpg)

图 9.44 – 以编程方式抛出错误

在 Slack 频道中，你应该会收到类似于以下截图所示的通知：

![图 9.45 – 当管道失败时发送通知](img/Figure_9.45_B17115.jpg)

图 9.45 – 当管道失败时发送通知

大致就是这样。本章仅仅触及了 CI/CD 流水线可以做到的事情的表面。然而，它应该为你提供了足够的基石，以便开始实验并构建你自己的 Gin 应用程序的端到端工作流程。

# 摘要

在本章中，你学习了如何从头开始设置 CI/CD 流水线，以使用 CircleCI 自动化 Gin 应用的部署过程。此外，使用 CircleCI orbs 通过简化我们编写 Pipeline as Code 配置的方式提高了生产力。

你还探索了如何使用 Docker 运行自动化测试，以及如何通过 GitFlow 和多个 AWS 环境实现持续部署。在这个过程中，你设置了 Slack 通知，以便在构建失败或成功时得到提醒。

本书最后一章将涵盖如何排查和调试在生产中运行的 Gin 应用程序。

# 问题

1.  为我们在*第五章*，“使用 Gin 提供静态 HTML”中构建的 React 网络应用程序构建 CI/CD 流水线以自动化部署过程。

1.  在成功完成生产部署时添加 Slack 通知。

# 进一步阅读

+   Mohamed Labouardy 著，Packt 出版的《*使用 Go 进行实战型无服务器应用程序开发*》

+   Salle Ingle 著，Packt 出版的《*在 AWS 上实现 DevOps*》
