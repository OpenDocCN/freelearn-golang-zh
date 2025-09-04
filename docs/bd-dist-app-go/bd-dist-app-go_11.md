# 第八章：在 AWS 上部署应用程序

本章将指导你如何在 **Amazon Web Services**（**AWS**）上部署 **API**。它还将进一步解释如何使用自定义域名通过 **HTTPS** 提供应用程序，并在 Kubernetes 和 **Amazon Elastic Container Service**（**Amazon ECS**）上扩展基于 Gin 的 API。

因此，我们将关注以下主题：

+   在 **Amazon Elastic Compute Cloud**（**Amazon EC2**）实例上部署 Gin 网络应用程序

+   在 Amazon **ECS**（**弹性容器服务**）上部署

+   使用 **Amazon Elastic Kubernetes Service**（**Amazon EKS**）在 Kubernetes 上部署

# 技术要求

要遵循本章的说明，你需要以下内容：

+   对前一章的完整理解——本章是前一章的后续，它将使用相同的源代码。因此，一些片段将不会解释，以避免重复。

+   使用 AWS 的先前经验是强制性的。

+   需要具备对 Kubernetes 的基本理解。

本章的代码包托管在 GitHub 上，网址为 [`github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter08`](https://github.com/PacktPublishing/Building-Distributed-Applications-in-Gin/tree/main/chapter08)。

# 在 EC2 实例上部署

在本书的整个过程中，你已经学习了如何使用 Gin 框架构建分布式网络应用程序以及如何对 API 进行本地加载和测试的扩展。在本节中，我们将介绍如何在云上部署以下架构并向外部用户提供服务。

这里可以看到应用架构的概述：

![Figure 8.1 – Application architecture![img/B17115_08_01_v2.jpg](img/B17115_08_01_v2.jpg)

图 8.1 – 应用架构

AWS 在云服务提供商方面是领导者——它提供了一系列基础设施服务，如负载均衡器、服务器、数据库和网络服务。

要开始，请创建一个 AWS 账户 ([`aws.amazon.com`](https://aws.amazon.com))。大多数 AWS 服务都提供丰富的免费层资源，因此部署你的应用程序将花费你很少或没有费用。

## 启动 EC2 实例

创建 AWS 账户后，你现在可以启动 EC2 实例。为此，请按照以下步骤操作：

1.  登录到 **AWS 管理控制台**([`console.aws.amazon.com`](https://console.aws.amazon.com)) 并搜索 **EC2**。在 **EC2** 仪表板中，单击 **启动实例**按钮以配置新的 EC2 实例。

1.  选择 **Amazon Linux 2 AMI** 作为 **Amazon Machine Image**（**AMI**）。这是将运行 EC2 实例的 **操作系统**（**OS**）。以下截图提供了该概述：![Figure 8.2 – AMI    ![img/B17115_08_02_v2.jpg](img/B17115_08_02_v2.jpg)

    图 8.2 – AMI

1.  接下来，选择实例类型。您可以从`t2.micro`实例开始，如果需要，稍后升级。然后，点击**配置实例详情**并保留默认设置，如下面的截图所示：![图 8.3 – 实例配置    ![图片](img/B17115_08_03_v2.jpg)

    Figure 8.3 – 实例配置

1.  现在，点击`GP2`到`GP3`或预配 IOPS。

    备注

    MongoDB 需要快速存储。因此，如果您计划在 EC2 上托管 MongoDB 容器，EBS 优化的类型可以提高**输入/输出（I/O**）操作。

1.  然后，点击`Name=application-sg`，如图所示。保留安全组在默认设置（允许端口 22 的 SSH 入站流量）。然后，点击**审查并启动**：![图 8.4 – 安全组    ![图片](img/B17115_08_04_v2.jpg)

    Figure 8.4 – 安全组

    备注

    作为最佳实践，您应该始终仅将**安全外壳协议（SSH**）限制为已知的静态**互联网协议（IP**）地址或网络。

1.  点击**启动**并分配一个密钥对或创建一个新的 SSH 密钥对。然后，点击**创建实例**。

1.  通过点击**查看实例**按钮返回**实例**仪表板——实例启动并运行可能需要几秒钟，但您应该能在屏幕上看到它，如下面的截图所示：![图 8.5 – EC2 仪表板    ![图片](img/Figure_8.5_B17115.jpg)

    Figure 8.5 – EC2 仪表板

1.  一旦实例准备就绪，打开您的终端会话，使用您的 SSH 密钥对中的公钥`key.pem`通过 SSH 连接到实例，如图所示：

    ```go
    ssh ec2-user@IP –I key.pem
    ```

1.  确认消息将出现——输入`Yes`。然后，运行以下命令安装 Git，`sudo su`命令用于在根级别提供权限。

    这里，我们使用 Docker `19.03.13-ce` 和 Docker Compose `1.29.0` 版本：

![图 8.6 – Docker 版本![图片](img/Figure_8.6_B17115.jpg)

Figure 8.6 – Docker 版本

您已成功配置并启动了 EC2 实例。

当 EC2 实例运行时，您可以部署在第*第六章*中介绍的**Docker Compose**堆栈，*扩展 Gin 应用程序*。为此，执行以下步骤：

1.  克隆以下 GitHub 仓库，其中包含分布式 Gin Web 应用程序的组件和文件：

    ```go
    chapter06/docker-compose.yml file:

    ```

    version: "3.9"

    services:

    api:

    image: api

    environment:

    - MONGO_URI=mongodb://admin:password

    @mongodb:27017/test?authSource=admin

    &readPreference=primary&ssl=false

    - MONGO_DATABASE=demo

    - REDIS_URI=redis:6379

    external_links:

    - mongodb

    - redis

    scale: 5

    dashboard:

    image: dashboard

    redis:

    image: redis

    mongodb:

    image: mongo:4.4.3

    environment:

    - MONGO_INITDB_ROOT_USERNAME=admin

    - MONGO_INITDB_ROOT_PASSWORD=password

    nginx:

    image: nginx

    ports:

    - 80:80

    volumes:

    - $PWD/nginx.conf:/etc/nginx/nginx.conf

    depends_on:

    - api

    - dashboard

    ```go

    The stack consists of the following services:a. RESTful API written with Go and the Gin frameworkb. A dashboard written with JavaScript and the React frameworkc. MongoDB for data storaged. Redis for in-memory storage and API cachinge. Nginx as a reverse proxy
    ```

1.  在部署堆栈之前，构建 RESTful API 和 Web 仪表板的 Docker 镜像。转到每个服务的相应文件夹并运行`docker build`命令。例如，以下命令用于构建 RESTful API 的 Docker 镜像：

    ```go
    cd api
    docker build -t api .
    ```

    命令输出如下：

    ![图 8.7 – Docker 构建日志    ![图片](img/Figure_8.7_B17115.jpg)

    图 8.7 – Docker 构建日志

1.  构建镜像后，发出以下命令：

    ```go
    cd ..
    docker-compose up –d
    ```

    服务将被部署，并将创建五个 API 实例，如下截图所示：

![图 8.8 – Docker 应用程序![图片](img/Figure_8.8_B17115.jpg)

图 8.8 – Docker 应用程序

应用程序启动并运行后，转到网页浏览器并粘贴用于连接到您的 EC2 实例的 IP 地址。然后，您应该看到以下错误消息：

![图 8.9 – 请求超时![图片](img/Figure_8.9_B17115.jpg)

图 8.9 – 请求超时

要修复此问题，您需要允许 80 端口的入站流量，这是 nginx 代理暴露的端口。转到 EC2 仪表板中的**安全组**，并搜索分配给运行应用程序的 EC2 实例的安全组。找到后，添加一个入站规则，如下所示：

![图 8.10 – 端口 80 上的入站规则![图片](img/Figure_8.10_B17115.jpg)

图 8.10 – 端口 80 上的入站规则

返回到您的网页浏览器并向实例 IP 发出 HTTP 请求。这次，nginx 代理将被调用并返回响应。如果您向`/api/recipes`端点发出请求，应该返回一个空数组，如下截图所示：

![图 8.11 – RESTful API 响应![图片](img/Figure_8.11_B17115.jpg)

图 8.11 – RESTful API 响应

MongoDB 的`recipes`集合为空。因此，通过在`/api/recipes`端点上发出以下 JSON 有效负载的`POST`请求来创建一个新的食谱：

![图 8.12 – 创建新食谱的 POST 请求![图片](img/Figure_8.12_B17115.jpg)

图 8.12 – 创建新食谱的 POST 请求

确保在`POST`请求中包含`Authorization`头。刷新网页浏览器页面，然后应该在网页仪表板上返回一个食谱，如下截图所示：

![图 8.13 – 新食谱![图片](img/Figure_8.13_B17115.jpg)

图 8.13 – 新食谱

现在，单击**登录**按钮，您应该会有一个不安全的源错误，如下所示：

![图 8.14 – Auth0 要求客户端通过 HTTPS 运行![图片](img/Figure_8.14_B17115.jpg)

图 8.14 – Auth0 要求客户端通过 HTTPS 运行

错误是由于 Auth0 需要在通过 HTTPS 协议提供服务的 Web 应用程序上运行。您可以通过在 EC2 实例之上设置一个**负载均衡器**来通过 HTTPS 提供服务。

## 使用应用程序负载均衡器进行 SSL 卸载

要通过 HTTPS 运行 API，我们需要一个 **安全套接字层**（**SSL**）证书。您可以通过 **AWS 证书管理器**（**ACM**）轻松获取 SSL 证书。此服务使得在 AWS 管理资源上配置、管理和部署 SSL/**传输层安全性**（**TLS**）证书变得容易。要生成 SSL 证书，请按照以下步骤操作：

1.  前往 ACM 仪表板，通过点击 **请求证书** 按钮，并选择 **请求公共证书** 来为您的域名请求免费的 SSL 证书。

1.  在 `domain.com` 上。

    备注

    `domain.com` 域名可以有多个子域名，例如 `sandbox.domain.com`、`production.domain.com` 和 `api.domain.com`。

1.  在 **选择验证方法** 页面上，选择 **DNS 验证** 并将 ACM 提供的 **规范名称**（**CNAME**）记录添加到您的 **域名系统**（**DNS**）配置中。颁发公共证书可能需要几分钟，但一旦域名验证完成，证书将被颁发，并在 ACM 仪表板中显示为 **已颁发** 状态，如下面的截图所示：![图 8.15 – 使用 ACM 请求公共证书    ](img/Figure_8.15_B17115.jpg)

    图 8.15 – 使用 ACM 请求公共证书

1.  接下来，从 EC2 仪表板中的 **负载均衡器** 部分创建一个应用程序负载均衡器，如下面的截图所示：![图 8.16 – 应用程序负载均衡器    ](img/Figure_8.16_B17115.jpg)

    图 8.16 – 应用程序负载均衡器

1.  在随后的页面上，输入负载均衡器的名称，并从下拉列表中指定方案为 **面向互联网**。在 **可用区** 部分中，为每个 **可用区**（**AZ**）选择一个子网以提高弹性。然后，在 **监听器** 部分中，添加一个 HTTPS 监听器和 HTTP 监听器，分别位于端口 443 和 80，如下面的截图所示：![图 8.17 – HTTP 和 HTTPS 监听器    ](img/B17115_08_17_v2.jpg)

    图 8.17 – HTTP 和 HTTPS 监听器

1.  点击 **配置安全设置** 按钮继续操作，并从下拉列表中选择 ACM 中创建的证书，如下面的截图所示：![图 8.18 – 证书配置    ](img/B17115_08_18_v2.jpg)

    图 8.18 – 证书配置

1.  现在，点击 **配置路由** 并创建一个新的名为 **application** 的目标组。确保协议设置为 HTTP，端口设置为 80，因为 nginx 代理监听端口 80。使用此配置，负载均衡器和实例之间的流量将使用 HTTP 传输，即使客户端向负载均衡器发出的 HTTPS 请求也是如此。您可以在以下截图中查看配置：![图 8.19 – 配置目标组    ](img/B17115_08_19_v2.jpg)

    图 8.19 – 配置目标组

1.  在 `HTTP` 和路径 `/api/recipes` 上。

1.  点击**注册目标**，选择运行应用程序的 EC2 实例，然后点击**添加到已注册**，如下所示：![图 8.20 – 注册 EC2 实例    ![图片](img/Figure_8.20_B17115.jpg)

    图 8.20 – 注册 EC2 实例

1.  当你完成实例选择后，选择**下一步：审查**。审查你选择的设置，然后点击**创建**按钮。配置过程可能需要几分钟，但你应该会看到如下屏幕：![图 8.21 – 负载均衡器 DNS 名称    ![图片](img/Figure_8.21_B17115.jpg)

    图 8.21 – 负载均衡器 DNS 名称

1.  一旦状态变为指向 Route 53 中负载均衡器的公共 DNS 名称的 A 记录（[`aws.amazon.com/route53/`](https://aws.amazon.com/route53/））或在你的 DNS 注册商那里，如图下所示：

![图 8.22 – Route 53 新 A 记录![图片](img/Figure_8.22_B17115.jpg)

图 8.22 – Route 53 新 A 记录

一旦你做出必要的更改，更改可能需要长达 48 小时才能在其他 DNS 服务器上传播。

通过浏览到 HTTPS://recipes.domain.com 来验证你的域名记录更改是否已传播。这应该会导致负载均衡器显示应用程序的安全 Web 仪表板。点击浏览器地址栏中的*锁*图标，它应该显示域名和 SSL 证书的详细信息，如图下所示：

![图 8.23 – 通过 HTTPS 提供服务![图片](img/Figure_8.23_B17115.jpg)

图 8.23 – 通过 HTTPS 提供服务

你的应用程序负载均衡器现在已配置了运行在 AWS 上的 Gin 应用程序的 SSL 证书。你可以使用 Auth0 服务通过 Web 仪表板登录并添加新食谱。

# 在 Amazon ECS 上部署

在上一节中，我们学习了如何部署一个 EC2 实例并配置它来运行我们的 Gin 应用程序。在本节中，我们将学习如何在不需要管理 EC2 实例的情况下获得相同的结果。AWS 提出了两种容器编排服务：**ECS**和**EKS**。

在本节中，你将了解 ECS，这是一个完全管理的容器编排服务。在我们将应用程序部署到 ECS 之前，我们需要将应用程序 Docker 镜像存储在远程仓库中。这就是**弹性容器注册库**（**ECR**）仓库发挥作用的地方。

## 在私有仓库中存储镜像

ECR 是一个广泛使用的私有 Docker 注册库。要在私有仓库中存储镜像，你首先需要在 ECR 中创建一个仓库。为了实现这一点，请按照以下步骤操作：

1.  从`mlabouardy/recipes-api`跳转到 ECR 仪表板，将其作为你的 Gin RESTful API 仓库的名称，如图下所示：![图 8.24 – 新的 ECR 仓库    ![图片](img/Figure_8.24_B17115.jpg)

    图 8.24 – 新的 ECR 仓库

    注意

    你可以在 Docker Hub 上托管你的 Docker 镜像。如果你选择这种方法，你可以跳过这部分。

1.  点击**创建存储库**按钮，然后选择存储库并点击**查看推送命令**。复制命令以进行身份验证并将 API 镜像推送到存储库，如下面的截图所示：![图 8.25 – ECR 登录和推送命令    ](img/Figure_8.25_B17115.jpg)

    图 8.25 – ECR 登录和推送命令

    注意

    有关如何安装 AWS **命令行界面**（**CLI**）的逐步指南，请参阅官方文档[`docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html`](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)。

1.  按照图 8.25 中显示的命令进行 ECR 的身份验证。标记镜像并将其推送到远程存储库，如下所示（将`ID`、`REGION`和`USER`变量替换为您自己的值）：

```go
aws ecr get-login-password --region REGION | docker login --username AWS --password-stdin ID.dkr.ecr.REGION.amazonaws.com
docker tag api ID.dkr.ecr.REGION.amazonaws.com/USER/recipes-api:latest
docker push ID.dkr.ecr.REGION.amazonaws.com/USER/recipes-api:latest
```

命令日志如下所示：

![图 8.26 – 将镜像推送到 ECR](img/Figure_8.26_B17115.jpg)

图 8.26 – 将镜像推送到 ECR

镜像现在将在 ECR 上可用，如下面的截图所示：

![图 8.27 – 存储在 ECR 上的镜像](img/Figure_8.27_B17115.jpg)

图 8.27 – 存储在 ECR 上的镜像

在 ECR 中存储 Docker 镜像后，您可以在 ECS 中部署应用程序。

现在，更新`docker-compose.yml`文件，在`image`部分引用 ECR 存储库 URI，如下所示：

```go
version: "3.9"
services:
 api:
   image: ACCOUNT_ID.dkr.ecr.eu-central-1.amazonaws.com/
      mlabouardy/recipes-api:latest
   environment:
     - MONGO_URI=mongodb://admin:password@mongodb:27017/
           test?authSource=admin&readPreference=
           primary&ssl=false
     - MONGO_DATABASE=demo
     - REDIS_URI=redis:6379
   external_links:
     - mongodb
     - redis
   scale: 5
 dashboard:
   image: ACCOUNT_ID.dkr.ecr.eu-central-1.amazonaws.com/
       mlabouardy/dashboard:latest
```

## 创建 ECS 集群

我们的`docker-compose.yml`文件现在引用了存储在 ECR 中的镜像。我们已准备好启动 ECS 集群并在其上部署应用程序。

您可以从**AWS 管理控制台**手动部署 ECS 集群，或通过 AWS ECS CLI 进行部署。根据您的操作系统，遵循官方说明从[`docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.htm`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.htm)安装 Amazon ECS CLI。

安装完成后，通过提供 AWS 凭证和创建集群的 AWS 区域来配置 Amazon ECS CLI，如下所示：

```go
ecs-cli configure profile --profile-name default --access-key KEY --secret-key SECRET
```

在配置 ECS 集群之前，定义一个任务执行`IAM`角色，以便允许 Amazon ECS 容器代理代表我们调用 AWS API。创建一个名为`task-execution-assule-role.json`的文件，并包含以下内容：

```go
{
   "Version": "2012-10-17",
   "Statement": [
       {
           "Sid": "",
           "Effect": "Allow",
           "Principal": {
               "Service": "ecs-tasks.amazonaws.com"
           },
           "Action": "sts:AssumeRole"
       }
   ]
}
```

使用 JSON 文件创建任务执行角色，并将`AmazonECSTaskExecutionRolePolicy`任务执行角色策略附加到它，如下所示：

```go
aws iam --region REGION create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json
aws iam --region REGION attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

使用以下命令完成配置，默认集群名称和启动类型。然后，使用`ecs-cli`的`up`命令创建 Amazon ECS 集群：

```go
ecs-cli configure --cluster sandbox --default-launch-type FARGATE --config-name sandbox --region REGION
ecs-cli up --cluster-config sandbox --aws-profile default
```

此命令可能需要几分钟才能完成，因为您的资源（EC2 实例、负载均衡器、安全组等）正在创建。此命令的输出如下所示：

![图 8.28 – 创建 ECS 集群](img/Figure_8.28_B17115.jpg)

图 8.28 – 创建 ECS 集群

跳转到 ECS 仪表板——沙盒集群应该已经启动并运行，如下面的截图所示：

![图 8.29 – 沙盒集群](img/Figure_8.29_B17115.jpg)

图 8.29 – 沙盒集群

要部署应用程序，您可以使用上一节中提供的`docker-compose`文件。除了这些，您还需要在配置文件中提供某些特定于 Amazon ECS 的参数，如下所示：

+   **子网**：应替换为 EC2 实例应部署的公共子网列表

+   **安全组和资源使用**：**中央处理单元**（**CPU**）和内存

创建一个包含以下内容的`ecs-params.yml`文件：

```go
version: 1
task_definition:
 task_execution_role: ecsTaskExecutionRole
 ecs_network_mode: awsvpc
 task_size:
   mem_limit: 2GB
   cpu_limit: 256
run_params:
 network_configuration:
   awsvpc_configuration:
     subnets:
       - "subnet-e0493c88"
       - "subnet-7472493e"
     security_groups:
       - "sg-d84cb3b3"
     assign_public_ip: ENABLED
```

接下来，使用以下命令将`docker compose`文件部署到集群中。`--create-log-groups`选项为容器日志创建 CloudWatch 日志组：

```go
ecs-cli compose --project-name application -f 
docker-compose.ecs.yml up --cluster sandbox 
--create-log-groups
```

部署日志如下所示：

![图 8.30 – 任务部署](img/Figure_8.30_B17115.jpg)

图 8.30 – 任务部署

将创建一个`application`任务。**任务**是一组元数据（内存、CPU、端口映射、环境变量），它描述了容器应该如何部署。您可以在以下位置查看其概述：

![图 8.31 – 任务定义](img/Figure_8.31_B17115.jpg)

图 8.31 – 任务定义

使用 AWS CLI，添加一个安全组规则以允许 80 端口的入站流量，如下所示：

```go
aws ec2 authorize-security-group-ingress --group-id SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --region REGION
```

输入以下命令以查看在 ECS 中运行的容器：

```go
ecs-cli compose --project-name application service ps –cluster--config sandbox --ecs-profile default
```

该命令将列出正在运行的容器以及 nginx 服务的 IP 地址和端口号。如果您将网络浏览器指向该地址，您应该能看到 Web 仪表板。

太好了！您现在有一个运行中的 ECS 集群，其中包含 Docker 化的 Gin 应用程序。

# 使用 Amazon EKS 在 Kubernetes 上部署

ECS 可能是一个适合初学者和小型工作负载的好解决方案。然而，对于大型部署和一定规模，您可能需要考虑转向 Kubernetes（也称为**K8s**）。对于那些 AWS 高级用户，Amazon EKS 是一个自然的选择。

AWS 在 EKS 服务下提供了一种托管的 Kubernetes 解决方案。

要开始，我们需要部署一个 EKS 集群，如下所示：

1.  跳转到 EKS 仪表板，并使用以下参数创建一个新的集群：![图 8.32 – EKS 集群创建    ](img/Figure_8.32_B17115.jpg)

    图 8.32 – EKS 集群创建

    集群的`IAM`角色应包括以下`AmazonEKSWorkerNodePolicy`、`AmazonEKS_CNI_Policy`和`AmazonEC2ContainerRegistryReadOnly`。

1.  在**指定网络**页面，选择一个现有的**虚拟专用云**（**VPC**）用于集群和子网，如图所示。其余部分保持默认设置：![图 8.33 – EKS 网络配置    ](img/Figure_8.33_B17115.jpg)

    图 8.33 – EKS 网络配置

1.  对于集群端点访问，为了简单起见，启用公共访问。对于生产使用，限制对您的网络**无类别域间路由**（**CIDR**）的访问或仅启用对集群 API 的私有访问。

1.  然后，在 **配置日志记录** 页面上，启用所有日志类型，以便能够轻松地从 CloudWatch 控制台调试或排除网络问题。

1.  查看信息，点击 `eksctl`，前往官方指南 [`docs.aws.amazon.com/eks/latest/userguide/create-cluster.html`](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)。

1.  一旦集群处于 **活动** 状态，创建一个托管节点组，容器将在其中运行。

1.  点击集群名称，选择 `workers` 并创建一个节点 IAM 角色，如图所示：![图 8.35 – EKS 节点组    ](img/Figure_8.35_B17115.jpg)

    图 8.35 – EKS 节点组

    注意

    有关如何配置节点组的更多信息，请参阅官方文档 [`docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role`](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role)。

1.  在下一页上，选择 `Amazon Linux 2` 作为 AMI，并选择 `t3.medium` `按需` 实例，如图所示：![图 8.36 – 工作节点配置    ](img/Figure_8.36_B17115.jpg)

    图 8.36 – 工作节点配置

    注意

    对于生产使用，您可能会使用 `Spot-Instances` 而不是 `按需`。由于可能出现的突发中断，`Spot-Instances` 通常会有很好的折扣。这些中断可以被 Kubernetes 优雅地处理，让您节省额外的费用。

    以下图示显示了配置是如何进行扩展的：

    ![图 8.37 – 扩展配置    ](img/Figure_8.37_B17115.jpg)

    图 8.37 – 扩展配置

1.  最后，指定两个节点将要部署的子网。在 **审查和创建** 页面上，审查您的托管节点组配置，然后点击 **创建**。

现在您已经部署了您的 EKS 集群，您需要配置 `kubectl`。

## 配置 kubectl

`kubectl` 是一个用于与集群 API 服务器通信的命令行工具。要安装此工具，请执行以下命令：

```go
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl 
chmod +x ./kubectl
mv kubectl /usr/local/bin/
```

在这本书中，我们使用的是最新版本的 `kubectl`，即 1.21.0，如图所示：

```go
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"darwin/amd64"}
```

接下来，生成一个包含 `kubectl` 与 EKS 集群交互所需凭证的 `kubeconfig` 文件，如下所示：

```go
aws eks update-kubeconfig --name sandbox --region eu-central-1
```

您现在可以通过以下命令测试凭证，列出集群的节点：

```go
kubectl get nodes
```

命令将列出两个节点，正如我们所看到的：

![图 8.38 – EKS 节点](img/Figure_8.38_B17115.jpg)

图 8.38 – EKS 节点

太棒了！您已成功配置 `kubectl`。

现在您已经设置了您的 EKS 集群，要在 Kubernetes 上运行服务，您需要将您的 `compose service` 定义转换为 Kubernetes 对象。**Kompose** 是一个开源工具，可以加快转换过程。

注意

您不必编写多个 Kubernetes **YAML Ain't Markup Language** (**YAML**)文件，您可以将整个应用程序打包在 Helm 图表中([`docs.helm.sh/`](https://docs.helm.sh/))，并将其存储在远程注册库中进行分发。

## 将 Docker Compose 工作流程迁移到 Kubernetes

Kompose 是一个开源工具，可以将`docker-compose.yml`文件转换为 Kubernetes 部署文件。要开始使用 Kompose，请按照以下步骤操作：

1.  导航到项目的 GitHub 发布页面([`github.com/kubernetes/kompose/releases`](https://github.com/kubernetes/kompose/releases))并下载适用于您的操作系统的二进制文件。这里使用的是版本 1.22.0：

    ```go
    curl -L https://github.com/kubernetes/kompose/releases/download/v1.22.0/kompose-darwin-amd64 -o kompose
    chmod +x kompose
    sudo mv ./kompose /usr/local/bin/kompose
    ```

1.  安装 Kompose 后，使用以下命令将服务定义转换为：

    ```go
    kompose convert -o deploy
    ```

1.  运行此命令后，Kompose 将输出它创建的文件信息，如下所示：![图 8.39 – 使用 Kompose 将 Docker Compose 转换为 Kubernetes 资源    ](img/Figure_8.39_B17115.jpg)

    ```go
    apiVersion: apps/v1
    kind: Deployment
    metadata:
     annotations:
       kompose.cmd: kompose convert
       kompose.version: 1.22.0 (955b78124)
     creationTimestamp: null
     labels:
       io.kompose.service: api
     name: api
    spec:
     replicas: 1
     selector:
       matchLabels:
         io.kompose.service: api
     strategy: {}
     template:
       metadata:
         annotations:
           kompose.cmd: kompose convert
           kompose.version: 1.22.0 (955b78124)
         creationTimestamp: null
         labels:
           io.kompose.network/api_network: "true"
           io.kompose.service: api
       spec:
         containers:
           - env:
               - name: MONGO_DATABASE
                 value: demo
               - name: MONGO_URI
                 value: mongodb://admin:password
                     @mongodb:27017/test?authSource=admin
                     &readPreference=primary&ssl=false
               - name: REDIS_URI
                 value: redis:6379
             image: ID.dkr.ecr.REGION.amazonaws.com/USER
                 /recipes-api:latest
             name: api
             resources: {}
         restartPolicy: Always
    status: {}
    ```

1.  现在，通过以下命令创建 Kubernetes 对象，并测试您的应用程序是否按预期工作：

    ```go
    kubectl apply -f .
    ```

    您将看到以下输出，表示已创建对象：

    ![图 8.40 – 部署和服务    ](img/Figure_8.40_B17115.jpg)

    图 8.40 – 部署和服务

1.  要检查您的 Pod 是否正在运行，使用以下命令部署 Kubernetes 仪表板：

    ```go
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.5/aio/deploy/recommended.yaml
    ```

1.  接下来，创建一个`eks-admin`服务帐户和集群角色绑定，您可以使用它以管理员权限安全地连接到仪表板，如下所示：

    ```go
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: eks-admin
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: eks-admin
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: eks-admin
      namespace: kube-system
    ```

1.  将内容保存到`eks-admin-service-account.yml`文件中，并使用以下命令将服务帐户应用到您的集群中：

    ```go
    kubectl apply -f eks-admin-service-account.yml
    ```

1.  在连接到仪表板之前，使用以下命令获取认证令牌：

    ```go
    kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
    ```

    Kubernetes 仪表板令牌的样式如下：

    ![图 8.41 – Kubernetes 仪表板令牌    ](img/Figure_8.41_B17115.jpg)

    图 8.41 – Kubernetes 仪表板令牌

1.  使用以下命令在本地运行代理：

    ```go
    kubectl proxy
    ```

1.  访问`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login`并粘贴认证令牌。

您应该被重定向到仪表板，在那里您可以查看分布式应用程序容器，以及它们的指标和状态，如下面的截图所示：

![图 8.42 – Kubernetes 仪表板](img/Figure_8.42_B17115.jpg)

图 8.42 – Kubernetes 仪表板

您可以轻松地监控在 EKS 中运行的应用程序，并在需要时扩展 API Pod。

注意

当您完成对 EKS 的实验后，删除您创建的所有资源是个好主意，这样 AWS 就不会为此向您收费。

# 摘要

在本章中，您学习了如何在 AWS 上使用 Amazon EC2 服务运行 Gin Web 应用程序，以及如何通过应用程序负载均衡器和 ACM 通过 HTTPS 提供服务。

您还探索了如何在不管理底层 EC2 节点的情况下，使用 ECS 将应用程序部署到托管集群。在这个过程中，您还了解了如何使用 ECR 将 Docker 镜像存储在远程注册库中，以及如何使用 Amazon EKS 进行可扩展的应用程序部署。

在下一章中，您将了解如何使用**持续集成/持续部署**（**CI/CD**）管道自动部署您的 Gin 应用程序到 AWS。

# 问题

1.  您将如何配置 MongoDB 容器数据的持久卷？

1.  在 AWS EC2 上部署 RabbitMQ。

1.  使用 Kubernetes Secrets 创建 MongoDB 凭证。

1.  使用`kubectl`将 API pod 扩展到五个实例。

# 进一步阅读

+   *《开发者 Docker》，作者：Richard Bullington-McGuire, Andrew K. Dennis, 和 Michael Schwartz。Packt 出版社*

+   *《精通 Kubernetes – 第三版》，作者：Gigi Sayfan。Packt 出版社*

+   *《亚马逊网络服务上的 Docker》，作者：Justin Menga。Packt 出版社*
