

# 对`BookSwap`网络应用程序进行端到端测试

我们在如何使用 TDD（测试驱动开发）来实现和测试 Go 代码的目标上取得了很大的进展。到目前为止，我们已经涵盖了实现单元测试和集成测试的广泛技术。单元测试验证每个组件按预期工作，而集成测试则扩展其范围，涵盖不同单元之间的接口和交互。此外，我们还学习了如何将这些技术应用到各种示例中，包括在前几章中介绍的`BookSwap`网络应用程序。

这些测试为我们提供了代码更改的快速反馈循环，因为它们不需要在运行测试套件之前启动整个应用程序并使其可用。如*第五章*中所述，*执行集成测试*，我们学习了如何利用`httptest`和`ginkgo`库轻松编写和运行网络应用程序的集成测试。我们还学习了如何利用**行为驱动开发**（**BDD**）来编写测试，这是一种编写集成和**端到端**（**E2E**）测试的流行技术。

然而，虽然我们可以依赖单元和集成测试来确保应用程序在各种场景下正确运行，但我们不应忽视端到端测试为我们的测试策略带来的好处。只有端到端测试才能让我们验证整个应用程序的行为并复制用户流程和体验。简单来说，这些测试让我们了解了生产环境中的用户体验，这是我们发布应用程序之前应该进行的最终验证。

本章致力于讨论之前章节中介绍的`BookSwap`网络应用程序的端到端测试套件的实现。我们将利用 Docker 来简化创建和销毁相同应用程序的过程，并讨论我们需要对应用程序进行的更改以使用数据库。然后，我们将学习如何使用 Godog 编写和运行端到端测试。最后，我们将讨论我们应该在测试中包含哪些数据库断言。

在本章中，我们将涵盖以下主题：

+   `BookSwap`应用程序的要求

+   网络应用程序中数据库存储的实现

+   开始使用 Cucumber 和 Godog

+   端到端测试的实现

+   数据库起始位置和断言

# 技术要求

您需要安装**Go 版本 1.19**或更高版本才能运行本章中的代码示例。安装过程在官方 Go 文档中描述，网址为[`go.dev/doc/install`](https://go.dev/doc/install)。

本书中包含的代码示例可在[`github.com/PacktPublishing/Test-Driven-Development-in-Go/chapter06`](https://github.com/PacktPublishing/Test-Driven-Development-in-Go/chapter06)公开获取。

# 用例 - 扩展`BookSwap`应用程序

`BookSwap`网络应用程序在第*第四章*中介绍，*构建高效的测试套件*。其主要功能允许用户列出他们的书籍并与其他用户交换。我们学习了其主要组件和端点，以及如何将表测试应用于其`BookService`。然后，在第*第五章*中，*执行集成测试*，我们学习了如何为其`Index`请求处理器编写集成测试。在本章中，我们将继续构建此应用程序的功能，更详细地查看每个端点的用户流程和预期功能。

*图 6.1*展示了`BookSwap`应用程序三个主要服务（`BookService`、`UserService`和`PostingService`）的责任概要：

![图 6.1 – BookSwap 应用程序主要组件的责任](img/Figure_6.1_B18371.jpg)

图 6.1 – BookSwap 应用程序主要组件的责任

每个服务都有自己的专业化和独立责任：

+   `BookService`负责所有书籍管理方面。此服务实现了创建、更新、列出和过滤书籍的功能。由于此应用程序在功能上相当有限，书籍仅按所有者的用户 ID 进行过滤，我们不会实现任何对`books`库存的搜索。

+   `UserService`负责所有用户管理方面。此服务实现了创建和更新用户配置文件的功能。它还可以通过 ID 获取特定用户，并依赖于`BookService`以接收所有书籍的列表，其所有者的 ID 与提供的用户 ID 相对应。

+   `PostingService`是`BookSwap`应用程序的外部服务，负责发布和交换书籍的细节。为了实现目的，我们将使用`BookSwap`应用程序。`PostingService`不是一个实际存在的服务，但我们将使用内部存根来模拟调用外部服务。

什么是存根？

**存根**是另一个组件的具体实现。存根不使用模拟框架，因为它们由实现代码使用。它们使测试更容易，并允许我们构建代码，就像外部组件已经构建并实现一样。由于 Go 接口的灵活性，存根实现可以轻松替换为真实实现。

如*第四章*中所述，*构建高效的测试套件*，`BookSwap`将其数据保存在映射中，目前没有数据库或持久化存储。在本章中，我们将更改其实施方式，以使用**PostgreSQL**数据库。

## 用户旅程

在本章中，我们将专注于端到端测试的实现。这些测试的焦点是验证应用程序在典型**用户旅程**下的行为。因此，在编写任何端到端测试用例之前，建立典型用户旅程或请求流程是很重要的。

什么是用户旅程？

用户旅程是指应用程序用户为了实现目标而采取的路径或请求序列。通常，这些旅程在生产环境中被跟踪，以了解用户如何使用服务。

*图 6.2*描述了`BookSwap`应用程序新用户的预期请求流程：

![图 6.2 – 新 BookSwap 应用程序用户的请求流程](img/Figure_6.2_B18371.jpg)

图 6.2 – 新 BookSwap 应用程序用户的请求流程

请求流程图让我们了解到哪些应用程序部分对于用户旅程的成功完成是必需的。对于新用户，应用程序的预期使用情况如下概述：

1.  `POST /users`端点，将用户配置文件详情作为 JSON 请求体传递。`UserService`服务将创建用户并将其返回给客户端。

1.  `BookSwap`应用程序和查看哪些书籍可供交换。客户端发出`GET /books`请求。`BookService`将获取书籍列表，并通过可用状态进行过滤。

1.  `POST /books/{id}`请求，传递他们想要预订的书的 ID。然后，他们将他们自己的用户 ID 作为 URL 参数传递，完成请求的 URL 为`POST /books/id?user={userId}`。这也可以通过请求体来实现。

观察图 6.2 中的主要组件，我们注意到有一个名为`HTTP Server`的额外组件。`BookSwap`应用程序中`HTTP Server`的实现包括几个不同的部分：

+   具有使用`Index`示例的处理器自定义类型在*第五章*中介绍，*执行集成测试*。通常，处理器自定义类型可以访问其执行公开操作所需的所有依赖项。

+   每个处理函数都使用`gorilla/mux`库来处理配置和将请求路由到相应的处理函数。您可以在[`github.com/gorilla/mux`](https://github.com/gorilla/mux)上了解更多关于`gorilla/mux`库的信息。

+   最后，一旦设置了路由和处理程序，我们就启动服务器并配置它监听指定的端口。这是使用 Go 标准库中的`net/http`库完成的。

什么是多路复用器？

`gorilla/mux`是 Go 开发者中流行的解决方案。

*图 6.2*展示了新用户的成功旅程。端到端测试应该覆盖各种场景，因此我们规划了多个请求流程。然而，由于它们的成本和运行时间较高，它们通常只覆盖**基本案例**或**成功路径**。

## 使用 Docker

到目前为止，我们使用 `go run` 命令运行应用程序，并使用 `go test` 命令进行测试。这种方法的缺点是我们必须在本地设置 Go 环境和任何依赖项之后才能本地构建和运行代码。在*第五章*中，*执行集成测试*，我们介绍了 Docker 作为解决这些问题的解决方案。

什么是 Dockerfile？

Dockerfile 是一个包含构建镜像所需所有命令的文件。然后，Docker 引擎使用它自动创建和启动容器。我们可以将其视为 Docker 容器设置的规范。

我们需要为我们的 `BookSwap` 应用程序创建一个自定义的 Dockerfile，因为它在 Docker Hub 镜像库中没有预定义的镜像。`Dockerfile.book-swap.chapter06` 文件包含 `BookSwap` 的规范：

```go
FROM golang:1.19-alpine
WORKDIR /app
COPY go.mod ./
COPY go.sum ./
COPY . .
RUN go mod download
RUN go build ./chapter06/cmd
EXPOSE ${BOOKSWAP_PORT}
CMD [ "./cmd" ]
```

这个相对简单的文件展示了我们需要有效使用 Docker 与我们的 Go 应用程序相关的所有基本知识：

1.  `FROM` 命令指示此构建阶段的基镜像。我们从 Docker Hub 选择了一个具有 `alpine` 镜像的图像，这些镜像轻量级且运行在 **Linux** **BusyBox** 发行版上。

1.  `WORKDIR` 命令创建并设置 Docker 容器的当前工作目录。我们文件中的所有后续命令都在这个目录中执行。

1.  接下来，`COPY` 命令将所有源文件从我们的本地目录复制到容器的当前工作目录。请记住，容器与底层本地目录是隔离的，因此这些文件必须复制到容器中。

1.  `RUN` 命令通过首先下载其依赖项，然后指定包含我们的应用程序入口点的目录，来执行构建我们的 Go 可执行文件所需的命令。Dockerfile 文件放置在根目录中的 `go.mod` 文件旁边，因此我们需要明确指出我们的章节入口点是从哪个构建的。

1.  `EXPOSE` 命令指示 Docker 容器在指定的端口上监听网络请求，如 `BOOKSWAP_PORT` 环境变量所示。这个变量对于 `BookSwap` 应用程序是必需的，所以确保在运行应用程序之前在您的终端会话中设置它。设置环境变量的说明将根据您的操作系统而有所不同。如果您想使用默认设置运行，请将 `BOOKSWAP_PORT` 环境变量设置为 `3000`。

1.  最后，`CMD` 命令指定容器启动后应运行的命令。我们从 `go` 的 `build` 步骤运行可执行文件。

这就是我们需要在任何运行 Docker 并具有互联网连接的环境中运行我们的应用程序所需指定的全部内容！大多数 Docker 规范都将使用这个简单的配方来编写和运行他们的自定义镜像。我们将在本章后面看到这个规范的实际应用。

## 持久化存储

我们将要进行的下一个更改是添加 `BookSwap` 应用程序，允许我们在应用程序关闭后保存状态。由于 SQL 数据库仍然是最受欢迎的持久化存储解决方案，我们将在演示应用程序中使用 SQL 数据库。

SQL 数据库管理解决方案

有几种流行的 SQL 数据库解决方案，您可能已经熟悉。其中一些是 **Oracle MySQL**、**Microsoft SQL Server** 和 **PostgreSQL**。它们都允许我们管理和与底层的 SQL 数据库交互，但在我们与之交互的方式上可能存在差异。因此，在测试环境中使用与生产环境中相同的 SQL 解决方案是很重要的。

我们将使用 **PostgreSQL 数据库**，这是一个开源的关系型数据库，在生产环境中被广泛使用。它还拥有跨公共云提供商的优秀云支持，因此它是我们技术栈的自然选择。

您可以按照官方文档中的步骤轻松下载并本地安装，官方文档可在 [`www.postgresql.org/download/`](https://www.postgresql.org/download/) 找到，遵循您操作系统的说明。一旦您本地安装了它，请记下您的宿主、端口、用户名和密码。您将需要这些信息来连接到您的数据库并在其上运行指令。

*图 6.3* 展示了我们应用程序所需的两个主要表：

![图 6.3 – BookSwap 应用程序的 SQL 表](img/Figure_6.3_B18371.jpg)

图 6.3 – BookSwap 应用程序的 SQL 表

有两个表：`books` 和 `users`。每个表都有一个用于每种类型数据字段的列。由于 `books` 的 `owner_id` 值应该属于一个现有用户，因此有一个 `id` 字段。所有列的类型都是 `varchar`，对应于 `string` 类型。

在我们启动应用程序之前，我们需要创建应用程序运行所需的所有表。然而，我们不想在源代码之外执行 SQL 指令，而是允许我们对其进行版本控制和审查。

`golang-migrate` 项目 ([`github.com/golang-migrate/migrate`](https://github.com/golang-migrate/migrate)) 允许我们编写迁移并在应用程序启动时运行它们。您可以通过遵循项目文档中 *入门* 教程中的安装步骤，轻松地在您的环境中安装 `migrate` CLI 工具。

安装完成后，我们可以为我们所需的两个表生成迁移：

```go
$ migrate create -ext sql -dir chapter06/db/migrations -seq create_users_table
$ migrate create -ext sql -dir chapter06/db/migrations -seq create_books_table
```

`migrate` CLI 工具为每个表创建两个文件，分别命名为 `*.up.sql` 和 `*.down.sql`。这些文件预期的用法是，up 迁移在启动时创建表，down 迁移在应用程序关闭时删除表。这确保了每次应用程序运行后都会删除表，并且它们总是在启动时创建。然后，我们添加每个表所需的列。这里展示了 `users` 表的配置：

```go
BEGIN;
CREATE TABLE IF NOT EXISTS users
(
   id VARCHAR (50) PRIMARY KEY,
   name VARCHAR (50) NOT NULL,
   // other column definitions
);
COMMIT;
```

迁移在不存在表的情况下创建一个表，并将 `id` 列标记为主键。指定与 *图 6**.3* 中定义的列相匹配。向下迁移是一行，它删除了表：

```go
DROP TABLE IF EXISTS users;
```

`books` 表的配置也是以同样的方式进行。最后，我们将迁移添加到 `BookSwap` 应用程序的入口点：

```go
func main() {
// other initialisation code
  postgresURL, ok := os.LookupEnv("BOOKSWAP_DB_URL")
  if !ok {
    log.Fatal("env variable BOOKSWAP_DB_URL not found")
  }
  m, err := migrate.New("file://chapter06/db/migrations", postgresURL)
  if err != nil {
    log.Fatal(err)
  }
  if err := m.Up(); err != nil {
    log.Fatal(err)
  }
   defer func() {
      m.Down()
  }()
  // other initialisation code
}
```

与 `migrate` 库交互需要在应用程序启动时添加三个额外的步骤：

1.  创建一个 `postgres://user:password@host:port/database`。`BookSwap` 应用程序需要一个名为 `BOOKSWAP_DB_URL` 的环境变量，其中包含此值。确保在启动应用程序之前设置此环境变量。

1.  一旦创建迁移实例，我们就调用 `Up()` 方法。此方法查看当前的迁移版本，并应用我们在 `*.up.sql` 文件中定义的所有迁移。

1.  如果我们想在应用程序关闭后或发生错误的情况下进行清理，我们将延迟调用迁移 `Down()` 方法。此方法使用相同的版本管理并运行 `*.down.sql` 文件的内容。

`golang-migrate` 库和 CLI 使我们能够将数据库配置与源代码一起保存，从而实现易于版本控制和版本管理。

一旦我们的数据库和表创建完成，我们需要重构我们的 `UserService` 和 `BookService` 实现以使用 SQL 表，而不是我们迄今为止使用的内置映射。通常，工程师会使用一个 **对象关系映射** (**ORM**) 库，它允许我们在 Go 自定义类型和 PostgreSQL 数据库之间创建桥梁。

Go 生态系统中有几个 ORM 解决方案。其中最受欢迎的一个是 **GORM** ([`github.com/go-gorm/gorm`](https://github.com/go-gorm/gorm))，这是一个开源的易于使用的 Go 库。这个库将使我们能够轻松地与我们的数据库解决方案交互，从而无需在源代码中以原始字符串的形式管理 SQL。

使用 GORM 的设置与使用 `golang-migrate` 所做的非常相似：

```go
func main() {
  // other initialisation code
  postgresURL, ok := os.LookupEnv("BOOKSWAP_DB_URL")
  if !ok {
    log.Fatal("$BOOKSWAP_DB_URL not found")
  }
  dbConn, err := gorm.Open(postgres.Open(postgresURL), &gorm.Config{})
  if err != nil {
    log.Fatal(err)
  } 
  ps := db.NewPostingService()
  b := db.NewBookService(dbConn, ps)
  u := db.NewUserService(dbConn, b)
  // initialisation code continues
 }
```

首先，我们使用之前与 `golang-migrate` 连接时使用的数据库 URL 连接到数据库。一旦成功连接，它将返回 GORM 数据库包装器类型 `*gorm.DB`。

如果我们无法连接到数据库，我们将终止应用程序。我们还更改了 `NewBookService` 和 `NewUserService` 初始化函数的签名，以接受初始化的数据库会话。

`BookService` 和 `UserService` 的所有之前将模型保存到内置映射类型的操作现在都必须利用 GORM 数据库包装器的操作。一个例子是 `BookService` 的 `ListByUser` 方法：

```go
// ListByUser returns the list of books for a given user.
func (bs *BookService) ListByUser(userID string) ([]Book, error) {
  var items []Book
  if result := bs.DB.Where("owner_id = ?", userID).Find(&items);
    result.Error != nil {
      return nil, result.Error
  }
return items, nil
}
```

此方法列出所有所有者 ID 与给定用户 ID 匹配的书籍。使用 ORM 解决方案的使用允许我们通过一个能够使用方法调用构造正确 SQL 查询的服务来在数据库上执行操作。这使我们能够减少错误并消除管理原始 SQL 字符串的需要。

## 运行 BookSwap 应用程序

对于我们需要对`BookSwap`应用程序进行的扩展，最后一部分是如何与数据库一起运行它。如*第五章*中所述，*执行集成测试*，我们使用的`docker-compose`工具是用来管理多个 Docker 容器的。现在我们有两个`BookSwap`应用程序的服务，或部分——服务器端应用程序及其数据库。数据库设置也需要在 Docker 容器中运行，以避免在每个环境中设置数据库。

`docker-compose`命令从`.yml`文件中获取输入，这使得指定不同的服务和它们的需求变得容易。我们`docker-compose.book-swap.chapter06.yml`文件中的简单指定定义了我们的`BookSwap`应用程序和它可以使用的一个 PostgreSQL 数据库：

```go
version: '3'
services:
 books:
  build:
   context: .
   dockerfile: Dockerfile.book-swap.chapter06
  ports:
   - "${BOOKSWAP_PORT}:${BOOKSWAP_PORT}"
  depends_on:
   db:
condition: service_healthy
  restart: on-failure
  env_file:
   - docker.env db:
  image: postgres:15.0-alpine
  ports:
   - "5432:5432"
  expose:
   - "5432"
  env_file:
   - docker.env
  restart: on-failure
```

这个相对简单的文件中的配置指定了我们所需的一切：

1.  我们为启动的所有服务定义一个服务块。在我们的情况下，我们将定义`books`服务和`db`服务，每个服务都在其自己的子块中配置。

1.  `books`服务的配置指定以下内容：

    1.  该服务是从当前目录中的自己的 Dockerfile 构建的。这是我们之前在本章的*使用 Docker*部分讨论过的 Dockerfile。

    1.  该服务在其网络上公开由`BOOKSWAP_PORT`环境变量指定的端口。这将允许我们运行需要本地访问端口的测试。

    1.  该服务依赖于`db`服务成功启动。Docker 引擎在启动我们的服务时将考虑这一点，并指示首先启动`db`服务。

    1.  该服务使用`docker.env`文件进行环境变量配置。这将指定我们需要的其他环境变量，例如之前提到的`BOOKSWAP_DB_URL`。

1.  `db`服务的配置指定以下内容：

    1.  容器应使用 Docker 仓库中的现有镜像。在撰写本文时，这是`5432`的最新镜像，这是 PostgreSQL 的常规做法。

    1.  如该镜像的文档所示，它需要指定数据库用户名、密码和名称的多个环境变量。所有这些变量都将定义在我们提供给该服务的`docker.env`文件中。

1.  两个服务都有定义的重启策略。这意味着如果容器失败，Docker 将自动重启容器。

我们可以为`docker.env`提供以下示例配置，但你可以通过编辑文件轻松根据你自己的偏好进行更改：

```go
POSTGRES_USER=root
POSTGRES_PASSWORD=root
POSTGRES_DB=books
BOOKSWAP_DB_URL=postgres://root:root@db:5432/books?sslmode=disable
BOOKSWAP_PORT=3000
```

运行两个服务所需的全部配置就是从项目根目录使用`docker compose -f docker-compose.book-swap.chapter06.yml up --build`命令。这个文件包含了一个典型的配置，你可以在自己的项目中重用。此外，它允许我们在不同的环境中一致地启动和运行`BookSwap`应用程序。这为我们提供了能够轻松启动整个应用程序的测试环境的关键优势。有了这些构建块，让我们看看如何利用这个关键优势来提高我们应用程序的测试覆盖率。

# 探索 Godog

在本章中，我们对`BookSwap`应用程序进行了相当多的更改，扩展了其范围和复杂性。现在我们可以轻松地使用 Docker 容器启动和关闭应用程序，是时候将我们的注意力转向编写应用程序的端到端测试了。

在*第五章*中，*执行集成测试*，我们探讨了如何编写 BDD 风格的测试。这种测试风格允许我们编写人类可读的测试场景，并使用**给定-当-然后**结构。这些可读的测试可以作为我们项目的文档，使我们能够涉及多个利益相关者，并编写真正覆盖我们应用程序功能的测试。

我们还探讨了`ginkgo`测试库，它允许我们使用这种风格编写测试。Godog（[`github.com/cucumber/godog`](https://github.com/cucumber/godog)）是另一个我们将要探索的测试库，用于编写 BDD 风格的测试。`ginkgo`允许我们在单元测试中添加 BDD 风格的断言，但 Godog 提供了额外的代码生成功能，使其非常适合编写功能测试。我们将学习如何使用这个优秀的库进行集成和端到端测试。

这里是 Godog 的一些亮点：

+   与我们迄今为止使用的库不同，Godog 不是使用`go test`命令运行测试，而是使用`godog run`命令。这个命令具有双重功能，既可以生成测试文件，也可以运行已经实现的测试。

+   测试被组织在**功能文件**中，这些文件描述了特定场景下特定功能部分的预期行为。Godog 使用一种称为**Gherkin**（[`cucumber.io/docs/gherkin/reference/`](https://cucumber.io/docs/gherkin/reference/)）的领域特定语言。在本章的剩余部分，我们将探讨如何以这种格式编写测试。

+   Godog 是一个开源库，由社区和 Cucumber 组织维护。你可以自由地探索源代码，甚至可以贡献。

就像我们迄今为止使用的其他依赖项一样，Godog 通过在终端中运行`go install`命令进行安装：

```go
$ go install github.com/cucumber/godog/cmd/godog@latest
```

现在我们已经了解了 Godog 的基本用法并成功安装了它，我们开始编写我们的第一个`BookSwap`应用程序：

```go
Feature: New user signs up
 In order to use the BookSwap application
 As a new user
 I need to be able to sign up.
Background: Verify configuration
 Given the BookSwap app is up
Scenario: Sign up
 Given user details
 When sent to the users endpoint
 Then a new user profile is created
```

功能文件描述了`BookSwap`应用程序新用户所需的功能的一部分：

+   该功能描述了作为应用程序新用户的注册场景。

+   作为背景步骤，`BookSwap`应用程序应该处于运行状态。这允许我们在运行整个应用程序的同时编写端到端测试，并在旁边运行测试。

+   当功能完成时，以下功能将可用：

    +   新客户将能够创建用户个人资料。

    +   当用户创建其个人资料时，他们将看到他们的用户摘要并收到他们的用户 ID，这将使他们能够进一步与应用程序交互。

    +   注册后，客户将能够通过使用他们的用户 ID 查看他们的个人资料。

    +   与应用程序的任何进一步交互都不在本功能的范围内。

正如所讨论的，功能文件基于应用程序预期的用户旅程和请求流程。功能文件应该易于阅读和理解，因此我们应该为覆盖其他功能和场景创建单独的文件，并使用非技术性语言。

在下一节中，我们将学习如何使用 Godog 实现和运行此功能文件。

# 使用 Godog 实现测试

在安装 Godog 并概述了我们的第一个功能之后，让我们将注意力转向此测试的实现。

我们将采取的主要步骤来实现概述的功能如下：

1.  创建功能和测试文件。

1.  实现我们`BookSwap`应用程序的功能测试步骤。

1.  运行应用程序以及测试。

如前所述，我们将使用 Godog 来实现 BDD 风格的端到端测试，因此在我们运行测试之前，我们需要确保应用程序正在运行。然而，这并不是 Godog 的要求，因此我们可以使用这个易于使用的库在任何级别编写测试。

## 创建测试文件

如前所述，Godog 依赖于代码生成来简化开发者的工作。该过程包括从终端复制代码并自行创建文件。让我们看看涉及到的步骤。

### 第 1 步 – 创建功能文件

功能文件存储在 Go 项目根目录下的`/features`目录中。由于我们在仓库中使用项目文件夹，我们需要在`/chapter06/features`下创建一个文件。我们将在该目录下创建一个文件，并将功能文本添加到其中：

```go
$ mkdir chapter06/features
$ vim chapter06/features/newUserSignsUp.feature
```

注意，文件命名是根据功能名称进行的，这使得理解文件关联的功能变得容易。

### 第 2 步 – 生成步骤定义

一旦功能文件包含我们的文本，我们就可以使用 Godog 生成实现该功能所需的步骤。`godog run`命令将以下生成的代码打印到终端：

```go
func aNewUserProfileIsCreated() error {
  return godog.ErrPending
}
func sentToTheUsersEndpoint() error {
  return godog.ErrPending
}
func theBookSwapAppIsUp() error {
  return godog.ErrPending
}
func userDetails() error {
  return godog.ErrPending
}

func InitializeScenario(ctx *godog.ScenarioContext) {
  ctx.Step(`^a new user profile is create, 
    aNewUserProfileIsCreated)
  ctx.Step(`^sent to the users endpoint$`, 
    sentToTheUsersEndpoint)
  ctx.Step(`^the BookSwap app is up$`, theBookSwapAppIsUp)
  ctx.Step(`^user details$`, userDetails)
}
```

*图 6**.4* 展示了我们场景中的步骤序列，以及它们所发出的任何 HTTP 请求：

![图 6.4 – 我们场景中执行的步骤和 HTTP 请求](img/Figure_6.4_B18371.jpg)

图 6.4 – 我们场景中执行的步骤和 HTTP 请求

生成的代码包含我们场景每个步骤的函数：

1.  `aNewUserProfileIsCreated` 函数向 `GET /users/{id}` 端点发送请求，并验证用户资料是否成功创建。它还将验证可以通过指定的用户 ID 成功检索用户资料。

1.  `sentToTheUsersEndpoint` 函数向 `POST /users` 端点发送 JSON 负载，并验证端点以正确的用户详情响应。它还将获取应用程序为新用户资料生成的用户 ID。

1.  `theBookSwapAppIsUp` 函数向 `GET /` 端点发送请求，并验证应用程序以 `200 OK` 状态码响应。在生产环境中，我们通常公开一个单独的 `/health` 端点，但我们将为了 `BookSwap` 演示应用程序的目的使用根端点。

1.  `userDetails` 函数将创建一个 `db.User` 实例，我们将将其序列化为 JSON 负载并发送到 `sentToTheUsersEndpoint` 步骤。它还将作为我们测试断言中的预期值，或 `want` 变量。

我们需要实现这些函数以调用我们应用程序的功能。

最后，`InitializeScenario` 函数将这些函数组合成步骤，并按字母顺序排序。当我们实现测试文件时，我们必须根据我们的功能定义正确排序它们。

虽然生成的代码很简单，但它为我们提供了测试代码的脚手架，并处理了与 Godog 测试运行器的交互。

### 步骤 3 – 创建测试文件

正如常规单元测试一样，Godog 测试也位于 `*_test.go` 文件中，并与它们所测试的包位于同一位置。由于我们将测试整个应用程序，我们在 `root` 目录中创建一个测试文件，与 `/features` 目录处于同一级别。我们创建一个与功能名称匹配的测试文件，并将生成的文件粘贴到其中：

```go
$ vi /cmd/newUserAddsBook_test.go
```

虽然测试名称不需要匹配，但使用匹配的测试将允许 Godog 将测试与功能匹配。

创建测试文件和代码后，我们再次执行 `godog run`。测试运行器将场景标记为 `pending`：

```go
Background: Verify configuration
  Given the BookSwap app is up # newUserSignsUp_test.go:148 -> theBookSwapAppIsUp
      TODO: write pending definition

Scenario: Sign up       # features/newUserSignsUp.feature:9
  Given user details    # newUserSignsUp_test.go:152 -> userDetails
  When sent to the users endpoint # newUserSignsUp_test.go:144 -> sentToTheUsersEndpoint
  Then a new user profile is created # newUserSignsUp_test.go:140 -> aNewUserProfileIsCreate
```

方便的是，输出还会打印出每个步骤的行号，显示我们遗漏了哪些实现测试所需的步骤。

## 实现测试步骤

现在，Godog 已经方便地为我们生成了测试步骤的脚手架，我们开始根据 `BookSwap` 应用程序的功能编写测试代码。然而，如前所述，我们需要在测试步骤之间传递信息。

在 Godog 中实现这一点的办法是通过链式上下文传递信息。Godog 将在测试步骤之间传递上下文，允许我们以安全的方式在步骤之间传递信息。为了做到这一点，我们需要更改测试步骤的签名，使其接受一个上下文并返回一个上下文和一个错误：

```go
func theBookSwapAppIsUp(ctx context.Context)(context.Context, error) { 
  // test step implementation 
}
```

测试步骤接受一个上下文并返回一个上下文和错误。在底层，Godog 将正确处理这些返回值：将返回的上下文链接到后续的测试步骤，并在出现非空错误时失败测试。

上下文复习

`context`类型是 Go 的标准库的一部分，其目的是携带截止日期、取消和请求范围内的变量。上下文应在函数之间传播，使我们能够将函数调用与我们的应用程序各层中的请求联系起来。创建一个新的上下文需要一个父上下文。取消随后将在子上下文链中传播。

为了我们的目的，我们将使用上下文来携带请求范围内的变量。我们将创建一个新的`contextKey`自定义类型，它将携带我们在测试步骤之间需要传递的所有变量：

```go
// contextKey is used to pass information between test 
// steps.
type contextKey struct {
  UsersURL string
  User     db.User
}
```

在我们的情况下，我们将传播`BookSwap`应用的`UsersURL`和创建用户的所需值。在我们的后台步骤`theBookSwapAppIsUp`中，我们可以看到如何使用上下文将信息传递给后续步骤：

```go
func theBookSwapAppIsUp(ctx context.Context) (context.Context, error) {
  url, err := getTestURL()
  if err != nil {
    return ctx, fmt.Errorf("incorrect config:%v", err)
  }
resp, err := http.Get(url)
  if err != nil || resp.StatusCode != http.StatusOK {
    return ctx, fmt.Errorf("bookswap not up:%v", err)
  }
  return context.WithValue(ctx, contextKey{}, contextKey{
    UsersURL: url + "/users",
  }), nil
}
```

此代码片段演示了与 REST 端点交互的步骤实现：

1.  我们通过调用`getTestURL`辅助函数来设置我们将要测试的环境的 URL 值。此函数根据为应用程序指定的环境变量构建 URL。这使得我们能够轻松地配置我们的测试以在不同的测试环境中运行，无论是本地还是远程。如果您想使用默认值运行，请将`BOOKSWAP_BASE_URL`环境变量设置为`http://localhost`，并将`BOOKSWAP_PORT`环境变量设置为`3000`到您的终端会话中。

1.  我们使用`http.Get`方法与定义的 URL 进行交互，保存错误和响应。我们从之前的章节中熟悉了`net/http`库。在这个测试中，其用法没有不同。

1.  如果出现错误或状态码不是`200 OK`，我们将返回一个错误。这将使此步骤失败并结束测试。

1.  最后，在成功的情况下，我们使用`context.WithValue`函数从`ctx`参数值创建一个子上下文，传递一个带有填充的`UsersURL`的`contextKey`值。在后续步骤中，我们将能够使用此 URL 进行我们的请求。

我们需要对生成的测试步骤进行的一个其他更改是按照它们应该运行的顺序重新排序它们。如果您从未使用过 Godog，这个步骤可能不太直观，但如果忘记了，它将很容易追踪，因为您的测试将失败。

## 运行测试套件

一切都实现之后，是时候对我们的测试进行测试了。首先，我们记得运行`BookSwap`应用程序，可以使用`docker compose -f docker-compose.book-swap.chapter06.yml up --build`命令。除非你已更改配置，否则这将通过`http://localhost:3000` URL 公开应用程序。你可以通过对这个端点执行`curl`命令来轻松验证应用程序是否正在运行：

```go
$ curl --location --request GET 'http://localhost:3000'
{"message":"Welcome to the BookSwap service!"}
```

如果你看到欢迎响应，那么应用程序正在运行并且正确连接到其数据库。

当应用程序运行时，我们使用`godog`的`run`命令执行我们的测试：

```go
$ cd chapter06 && godog run 
Feature: New user signs up
  In order to use the BookSwap application
  As a new user
  I need to be able to sign up.
Background: Verify configuration
  Given the BookSwap app is up  # newUserSignsUp_test.go:23 -> theBookSwapAppIsUp
Scenario: Sign up             # features/newUserSignsUp.feature:9
  Given user details     # newUserSignsUp_test.go:35 -> userDetails
  When sent to the users endpoint # newUserSignsUp_test.go:50 -> sentToTheUsersEndpoint
  Then a new user profile is created # newUserSignsUp_test.go:84 -> aNewUserProfileIsCreated
1 scenarios (1 passed)
4 steps (4 passed)
11.876996ms
```

如我们从终端输出中可以看到，Godog 运行了一个场景，所有四个步骤都通过了。或者，如果你不想安装 Godog CLI，可以使用`go test`命令运行测试，但这样不会格式化测试结果，就像你看到的前面的输出一样。

我们已经成功编写并运行了`BookSwap`应用程序的第一个端到端测试，该应用程序已扩展为具有持久化存储。该测试使用 Godog 开源测试库编写，这使得我们可以编写易于阅读的 BDD 风格测试。我们正在朝着成为 Go 测试专家的方向前进。

# 使用数据库断言

我们已经学会了如何在测试环境中启动我们的应用程序，以及如何为我们的应用程序编写和运行端到端测试。这使我们大大接近验证应用程序的行为，但如何确保存储的数据和数据库组件是正确的呢？帮助我们回答这个问题的端到端测试的最后一个方面是数据库测试。查看我们迄今为止编写的测试，我们注意到两件事：

+   数据库通常初始化为空，然后在应用程序关闭后拆毁表。这有一个优点，即我们知道测试中不会有持久数据干扰，但缺点是必须将所需数据作为测试的一部分设置。例如，在我们的案例中，注册可用的书需要用户 ID，因此在我们进行任何与书籍相关的任务之前，我们必须首先创建一个用户。这可能会使我们的测试套件运行时间更长。

+   数据库中的条目通过`BookSwap`端点进行断言。例如，我们通过向`GET /users/{id}`发送带有相应用户 ID 的请求来检查用户是否被正确存储在数据库中。然而，随着请求在整个应用程序堆栈中传递，确定错误来源可能变得困难。

让我们进一步探讨这两个痛点，以更好地了解如何解决它们。

## 初始化数据

如前文在*持久化存储*部分所述，我们使用`golang-migrate`编写数据库迁移，这允许我们在应用程序启动之前创建和设置我们的数据库以供使用。然后，我们使用**GORM**库作为我们的 ORM，这使得我们可以使用自定义类型轻松地与数据库交互。

下一步是将一些数据插入到我们新创建的表中。这种初始数据被称为`INSERT`命令：

```go
INSERT INTO users VALUES ('ABC-123','Initial user', '1 London Road', 'N1', 'UK');
```

然而，我们探索的所有工具都没有创建和添加随机种子为我们应用程序的能力。我们可以添加另一个库依赖来生成随机数据，但相反，我们可以在测试运行之前使用 GORM 的`DB`类型将随机数据插入到我们的数据库中：

```go
func addUser() error {
  dbConn, err := gorm.Open(postgres.Open(postgresURL), &gorm.Config{})
  if err != nil {
    return err
  }
  dbConn.Save(&db.User{
     ID:       uuid.New().String(),
     Name:     "Generated User",
  })
  return nil
}
```

之前的代码片段展示了如何在运行 Godog 测试步骤的同时将数据插入数据库，具体细节如下：

1.  就像在应用程序启动时一样，我们打开一个新的连接到我们的 PostgreSQL 数据库。数据库连接应尽可能在测试之间共享，并且不应为许多测试迭代打开。然而，由于数据库起始位置通常需要用于端到端测试，以这种方式设置我们的数据库是可行的。

1.  一旦数据库连接成功打开，我们使用 GORM 数据库的`Save`方法保存一个生成的用户。之后，数据库将包含生成数据，并可以在我们的测试中使用。

## 测试用例和断言

当涉及到对数据库内容的断言时，我们可以采取与添加生成数据时相同的方法：

```go
func verifyUser(want db.User) error {
  dbConn, err := gorm.Open(postgres.Open(postgresURL), &gorm.Config{})
  if err != nil {
    log.Fatal(err)
  }
  var got db.User
  if err := dbConn.Where("id = ?", want.ID).First(&got); 
  err != nil {
    return err.Error
  }
  if want != got {
    return fmt.Errorf("user does not match:got %v, want %v",got, want)
  }
  return nil
}
```

观察验证代码，我们看到同样的方法：

1.  我们使用连接字符串连接到数据库。当被多个 goroutine 使用时，GORM 会优化数据库连接的使用。

1.  然后，我们使用数据库方法查询数据库以获取传递给方法的用户 ID。请注意，我们直接依赖于 GORM 数据库，而不是我们自己的`UserService`方法，从而消除了我们可能引入的任何可能的错误。

虽然 GORM 使用起来很简单，但编写完整的数据库查询以断言返回值可能会感觉相当冗长。`dbassert`开源库([`github.com/hashicorp/dbassert`](https://github.com/hashicorp/dbassert))提供了包装和辅助函数，可以使这个过程更容易。您可以自行探索这个库，看看它如何帮助简化您的测试代码。

# 摘要

在本章中，我们致力于扩展`BookSwap`应用程序。我们首先讨论了用户典型的用户旅程，向其中添加了 PostgreSQL 数据库，并将其配置为使用 Docker 运行。然后，我们探讨了 Godog 测试库，它使得编写 BDD 风格的测试以及端到端测试变得容易。我们利用 Godog 验证用户能否在`BookSwap`应用程序上注册，利用 Godog 的代码生成能力。最后，我们简要讨论了在数据库级别直接创建数据库起始位置和断言的挑战。

在*第七章*中，我们将讨论 Go 语言的代码重构工具和技术，以及如何将单体应用拆分成多个服务。这将让我们对如何依靠测试来验证重构不会导致错误或破坏现有功能有一个现实的理解。

# 问题

1.  用户旅程的目的是什么？

1.  ORM 是什么？

1.  使用 Docker Compose 的优势是什么？

1.  数据库初始化是什么？

# 进一步阅读

+   *《用户故事映射：发现整个故事，构建正确的产品》*，作者 *杰夫·帕顿*，*彼得·艾康尼*。由 *O’Reilly* 出版。

+   *《Kubernetes 宝典：容器技术的实战深入探究》*，作者 *艾伦·霍恩*。由 *No Starch Press* 出版。

+   *《SQL 与 NoSQL 数据库：大数据管理模型、语言、一致性选项和架构》*，作者 *安德烈亚斯·迈尔*，*迈克尔·考夫曼*。由 *Springer Vieweg* 出版。
