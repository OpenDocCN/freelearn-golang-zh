# 第四章：使用流行的 Go 框架简化 RESTful 服务

在本章中，我们将涵盖使用框架简化构建 REST 服务相关的主题。首先，我们将快速了解 go-restful，一个 REST API 创建框架，然后转向一个名为`Gin`的框架。我们将在本章尝试构建一个地铁 API。我们将讨论的框架是完整的 Web 框架，也可以用来在短时间内创建 REST API。在本章中，我们将大量讨论资源和 REST 动词。我们将尝试将一个名为`Sqlite3`的小型数据库与我们的 API 集成。最后，我们将检查`Revel.go`，看看如何用它原型化我们的 REST API。

总的来说，本章我们将涵盖的主题如下：

+   如何在 Go 中使用 SQLite3

+   使用 go-restful 包创建 REST API

+   介绍用于创建 REST API 的 Gin 框架

+   介绍 Revel.go 用于创建 REST API

+   构建 CRUD 操作的基础知识

# 获取代码

您可以从[`github.com/narenaryan/gorestful/tree/master/chapter4`](https://github.com/narenaryan/gorestful/tree/master/chapter4)获取本章的代码示例。本章的示例以项目的形式而不是单个程序的形式呈现。因此，将相应的目录复制到您的`GOPATH`中以正确运行代码示例。

# go-restful，一个用于创建 REST API 的框架

`go-restful`是一个用于在 Go 中构建 REST 风格 Web 服务的包。REST，正如我们在前面的部分中讨论的，要求开发人员遵循一组设计协议。我们已经讨论了 REST 动词应该如何定义以及它们对资源的影响。

使用`go-restful`，我们可以将 API 处理程序的逻辑分离并附加 REST 动词。这样做的好处是，通过查看代码，清楚地告诉我们正在创建什么 API。在进入示例之前，我们需要为`go-restful`的 REST API 安装一个名为 SQLite3 的数据库。安装步骤如下：

+   在 Ubuntu 上，运行以下命令：

```go
 apt-get install sqlite3 libsqlite3-dev
```

+   在 OS X 上，您可以使用`brew`命令安装 SQLite3：

```go
 brew install sqlite3
```

+   现在，使用以下`get`命令安装`go-restful`包：

```go
 go get github.com/emicklei/go-restful
```

我们已经准备好了。首先，让我们编写一个简单的程序，展示`go-restful`在几行代码中可以做什么。让我们创建一个简单的 ping 服务器，将服务器时间回显给客户端：

```go
package main
import (
    "fmt"
    "github.com/emicklei/go-restful"
    "io"
    "net/http"
    "time"
)
func main() {
    // Create a web service
    webservice := new(restful.WebService)
    // Create a route and attach it to handler in the service
    webservice.Route(webservice.GET("/ping").To(pingTime))
    // Add the service to application
    restful.Add(webservice)
    http.ListenAndServe(":8000", nil)
}
func pingTime(req *restful.Request, resp *restful.Response) {
    // Write to the response
   io.WriteString(resp, fmt.Sprintf("%s", time.Now()))
}
```

如果我们运行这个程序：

```go
go run basicExample.go
```

服务器将在本地主机的端口`8000`上运行。因此，我们可以使用 curl 请求或浏览器来查看`GET`请求的输出：

```go
curl -X GET "http://localhost:8000/ping"
2017-06-06 07:37:26.238146296 +0530 IST
```

在上述程序中，我们导入了`go-restful`库，并使用`restful.WebService`结构的新实例创建了一个新的服务。接下来，我们可以使用以下语句创建一个 REST 动词：

```go
webservice.GET("/ping")
```

我们可以附加一个函数处理程序来执行这个动词；`pingTime`就是这样一个函数。这些链接的函数被传递给`Route`函数以创建一个路由器。然后是以下重要的语句：

```go
restful.Add(webservice)
```

这将注册新创建的`webservice`到`go-restful`。如果您注意到，我们没有将任何`ServeMux`对象传递给`http.ListenServe`函数；`go-restful`会处理它。这里的主要概念是使用基于资源的 REST API 创建`go-restful`。从基本示例开始，让我们构建一些实际的东西。

假设你的城市正在建设新的地铁，并且你需要为其他开发人员开发一个 REST API 来消费并相应地创建一个应用程序。我们将在本章中创建这样一个 API，并使用各种框架来展示实现。在此之前，对于**创建、读取、更新、删除**（**CRUD**）操作，我们应该知道如何使用 Go 代码查询或将它们插入到 SQLite 数据库中。

# CRUD 操作和 SQLite3 基础知识

所有的 SQLite3 操作都将使用一个名为`go-sqlite3`的库来完成。我们可以使用以下命令安装该包：

```go
go get github.com/mattn/go-sqlite3
```

这个库的特殊之处在于它使用了 Go 的内部`sql`包。我们通常导入`database/sql`并使用`sql`在数据库（这里是 SQLite3）上执行数据库查询：

```go
import "database/sql"
```

现在，我们可以创建一个数据库驱动程序，然后使用`Query`方法在其上执行 SQL 命令：

`sqliteFundamentals.go`:

```go
package main
import (
    "database/sql"
    "log"
    _ "github.com/mattn/go-sqlite3"
)
// Book is a placeholder for book
type Book struct {
    id int
    name string
    author string
}
func main() {
    db, err := sql.Open("sqlite3", "./books.db")
    log.Println(db)
    if err != nil {
        log.Println(err)
    }
    // Create table
    statement, err := db.Prepare("CREATE TABLE IF NOT EXISTS books (id
INTEGER PRIMARY KEY, isbn INTEGER, author VARCHAR(64), name VARCHAR(64) NULL)")
    if err != nil {
        log.Println("Error in creating table")
    } else {
        log.Println("Successfully created table books!")
    }
    statement.Exec()
    // Create
    statement, _ = db.Prepare("INSERT INTO books (name, author, isbn) VALUES (?, ?, ?)")
    statement.Exec("A Tale of Two Cities", "Charles Dickens", 140430547)
    log.Println("Inserted the book into database!")
    // Read
    rows, _ := db.Query("SELECT id, name, author FROM books")
    var tempBook Book
    for rows.Next() {
        rows.Scan(&tempBook.id, &tempBook.name, &tempBook.author)
        log.Printf("ID:%d, Book:%s, Author:%s\n", tempBook.id,
tempBook.name, tempBook.author)
    }
    // Update
    statement, _ = db.Prepare("update books set name=? where id=?")
    statement.Exec("The Tale of Two Cities", 1)
    log.Println("Successfully updated the book in database!")
    //Delete
    statement, _ = db.Prepare("delete from books where id=?")
    statement.Exec(1)
    log.Println("Successfully deleted the book in database!")
}
```

这个程序解释了如何在 SQL 数据库上执行 CRUD 操作。目前，数据库是 SQLite3。让我们使用以下命令运行它：

```go
go run sqliteFundamentals.go
```

输出如下，打印所有的日志语句：

```go
2017/06/10 08:04:31 Successfully created table books!
2017/06/10 08:04:31 Inserted the book into database!
2017/06/10 08:04:31 ID:1, Book:A Tale of Two Cities, Author:Charles Dickens
2017/06/10 08:04:31 Successfully updated the book in database!
2017/06/10 08:04:31 Successfully deleted the book in database!
```

这个程序在 Windows 和 Linux 上都可以正常运行。在 Go 版本低于 1.8.1 的情况下，你可能会在 macOS X 上遇到问题，比如*Signal Killed*。这是因为 Xcode 版本的问题，请记住这一点。

关于程序，我们首先导入`database/sql`和`go-sqlite3`。然后，我们使用`sql.Open()`函数在文件系统上打开一个`db`文件。它接受两个参数，数据库类型和文件名。如果出现问题，它会返回一个错误，否则返回一个数据库驱动程序。在`sql`库中，为了避免 SQL 注入漏洞，该包提供了一个名为`Prepare`的函数：

```go
statement, err := db.Prepare("CREATE TABLE IF NOT EXISTS books (id INTEGER PRIMARY KEY, isbn INTEGER, author VARCHAR(64), name VARCHAR(64) NULL)")
```

前面的语句只是创建了一个语句，没有填充任何细节。实际传递给 SQL 查询的数据使用语句中的`Exec`函数。例如，在前面的代码片段中，我们使用了：

```go
statement, _ = db.Prepare("INSERT INTO books (name, author, isbn) VALUES (?, ?, ?)")
statement.Exec("A Tale of Two Cities", "Charles Dickens", 140430547)
```

如果你传递了不正确的值，比如导致 SQL 注入的字符串，驱动程序会立即拒绝 SQL 操作。要从数据库中获取数据，使用`Query`方法。它返回一个迭代器，使用`Next`方法返回匹配查询的所有行。我们应该在循环中使用该迭代器进行处理，如下面的代码所示：

```go
rows, _ := db.Query("SELECT id, name, author FROM books")
var tempBook Book
for rows.Next() {
     rows.Scan(&tempBook.id, &tempBook.name, &tempBook.author)
     log.Printf("ID:%d, Book:%s, Author:%s\n", tempBook.id, tempBook.name, tempBook.author)
}
```

如果我们需要向`SELECT`语句传递条件，那么你应该准备一个语句，然后将通配符(?)数据传递给它。

# 使用 go-restful 构建地铁 API

让我们利用前一节学到的知识，为我们在前一节谈到的城市地铁项目创建一个 API。路线图如下：

1.  设计 REST API 文档。

1.  为数据库创建模型。

1.  实现 API 逻辑。

# 设计规范

在创建任何 API 之前，我们应该知道 API 的规范是什么样的，以文档的形式。我们在前几章中展示了一些例子，包括 URL 缩短器 API 设计文档。让我们尝试为这个地铁项目创建一个。看一下下面的表格：

| **HTTP 动词** | **路径** | **操作** | **资源** |
| --- | --- | --- | --- |
| `POST` | `/v1/train` (details as JSON body) | 创建 | 火车 |
| `POST` | `/v1/station` (details as JSON body) | 创建 | 站点 |
| `GET` | `/v1/train/id`  | 读取 | 火车 |
| `GET` | `/v1/station/id` | 读取 | 站点 |
| `POST` | `/v1/schedule` (source and destination) | 创建 | 路线 |

我们还可以包括`UPDATE`和`DELETE`方法。通过实现前面的设计，用户可以很容易地自行实现它们。

# 创建数据库模型

让我们编写一些 SQL 字符串，为前面的火车、站点和路线资源创建表。我们将为这个 API 创建一个项目布局。项目布局将如下截图所示：

![](img/bd044302-d60b-436b-b223-7ea7454c6d0e.png)

我们在`$GOPATH/src/github.com/user/`中创建我们的项目。这里，用户是`narenaryan`，`railAPI`是我们的项目源，`dbutils`是我们自己的处理数据库初始化实用函数的包。让我们从`dbutils/models.go`文件开始。我将在`models.go`文件中为火车、站点和时间表各添加三个模型：

```go
package dbutils

const train = `
      CREATE TABLE IF NOT EXISTS train (
           ID INTEGER PRIMARY KEY AUTOINCREMENT,
           DRIVER_NAME VARCHAR(64) NULL,
           OPERATING_STATUS BOOLEAN
        )
`

const station = `
        CREATE TABLE IF NOT EXISTS station (
          ID INTEGER PRIMARY KEY AUTOINCREMENT,
          NAME VARCHAR(64) NULL,
          OPENING_TIME TIME NULL,
          CLOSING_TIME TIME NULL
        )
`
const schedule = `
        CREATE TABLE IF NOT EXISTS schedule (
          ID INTEGER PRIMARY KEY AUTOINCREMENT,
          TRAIN_ID INT,
          STATION_ID INT,
          ARRIVAL_TIME TIME,
          FOREIGN KEY (TRAIN_ID) REFERENCES train(ID),
          FOREIGN KEY (STATION_ID) REFERENCES station(ID)
        )
`
```

这些都是用反引号（`` ` ``）字符括起来的普通多行字符串。该时刻表保存了在给定时间到达特定车站的列车的信息。在这里，火车和车站是时间表的外键。对于train，与之相关的细节是列。包名是`dbutils`，当我们提到包名时，包中的所有Go程序都可以共享导出的变量和函数，而不需要实际导入。

现在，让我们在`init-tables.go`文件中添加代码来初始化（创建表）数据库：

```go

package dbutils
import "log"
import "database/sql"
func Initialize(dbDriver *sql.DB) {
    statement, driverError := dbDriver.Prepare(train)
    if driverError != nil {
        log.Println(driverError)
    }
    // 创建火车表
    _, statementError := statement.Exec()
    if statementError != nil {
        log.Println("Table already exists!")
    }
    statement, _ = dbDriver.Prepare(station)
    statement.Exec()
    statement, _ = dbDriver.Prepare(schedule)
    statement.Exec()
    log.Println("All tables created/initialized successfully!")
}

```

我们导入`database/sql`以将参数类型传递给函数。函数中的所有其他语句与我们在上述代码中给出的 SQLite3 示例类似。它只是在 SQLite3 数据库中创建了三个表。我们的主程序应该将数据库驱动程序传递给此函数。如果你观察这里，我们没有导入 train、station 和 schedule。但是，由于此文件位于`db utils`包中，`models.go`中的变量是可访问的。

现在我们的初始包已经完成。你可以使用以下命令为此包构建对象代码：

```go

go build github.com/narenaryan/dbutils

```

直到我们创建并运行我们的主程序才有用。所以，让我们编写一个简单的主程序，从`dbutils`包导入`Initialize`函数。让我们将文件命名为`main.go`：

```go

package main
import (
    "database/sql"
    "log"
    _ "github.com/mattn/go-sqlite3"
    "github.com/narenaryan/dbutils"
)
func main() {
    // 连接到数据库
    db, err := sql.Open("sqlite3", "./railapi.db")
    if err != nil {
        log.Println("Driver creation failed!")
    }
    // 创建表
    dbutils.Initialize(db)
}

```

并使用以下命令从`railAPI`目录运行程序：

```go

go run main.go

```

你看到的输出应该类似于以下内容：

```go

2017/06/10 14:05:36 所有表格成功创建/初始化！

```

在上述程序中，我们添加了创建数据库驱动程序的代码，并将表创建任务传递给了`dbutils`包中的`Initialize`函数。我们可以直接在主程序中完成这个任务，但是将逻辑分解成多个包和组件是很好的。现在，我们将扩展这个简单的布局，使用`go-restful`包创建一个 API。API 应该实现我们的 API 设计文档中的所有函数。

当我们运行我们的主程序时，上述目录树图片中的`railapi.db`文件将被创建。如果数据库文件不存在，SQLite3 将负责创建数据库文件。SQLite3 数据库是简单的文件。你可以使用`$ sqlite3 file_name`命令进入 SQLite shell。

让我们将主程序修改为一个新的程序。我们将逐步进行，并在此示例中了解如何使用`go-restful`构建 REST 服务。首先，向程序中添加必要的导入：

```go

package main
import (
    "database/sql"
    "encoding/json"
    "log"
    "net/http"
    "time"
    "github.com/emicklei/go-restful"
    _ "github.com/mattn/go-sqlite3"
    "github.com/narenaryan/dbutils"
)

```

我们需要两个外部包，`go-restful`和`go-sqlite3`，用于构建 API 逻辑。第一个用于处理程序，第二个用于添加持久性特性。`dbutils`是我们之前创建的。`time`和`net/http`包用于一般任务。

尽管 SQLite 数据库表中给出了具体的列名称，在 GO 编程中，我们需要一些结构体来处理数据进出数据库。我们需要为所有模型定义数据持有者，所以下面我们将定义它们。看一下以下代码片段：

```go

// DB Driver visible to whole program
var DB *sql.DB
// TrainResource is the model for holding rail information
type TrainResource struct {
    ID int
    DriverName string
    OperatingStatus bool
}
// StationResource holds information about locations
type StationResource struct {
    ID int
    Name string
    OpeningTime time.Time
    ClosingTime time.Time
}
// ScheduleResource links both trains and stations
type ScheduleResource struct {
    ID int
    TrainID int
    StationID int
    ArrivalTime time.Time
}

```

`DB`变量被分配为保存全局数据库驱动程序。上面的所有结构体都是 SQL 中数据库模型的确切表示。Go 的`time.Time`结构体类型实际上可以保存数据库中的`TIME`字段。

现在是真正的`go-restful`实现。我们需要为我们的 API 在`go-restful`中创建一个容器。然后，我们应该将 Web 服务注册到该容器中。让我们编写`Register`函数，如下面的代码片段所示：

```go

// Register adds paths and routes to container
func (t *TrainResource) Register(container *restful.Container) {
    ws := new(restful.WebService)
    ws.Path("/v1/trains").
    Consumes(restful.MIME_JSON).
    Produces(restful.MIME_JSON) // you can specify this per route as well
    ws.Route(ws.GET("/{train-id}").To(t.getTrain))
    ws.Route(ws.POST("").To(t.createTrain))
    ws.Route(ws.DELETE("/{train-id}").To(t.removeTrain))
    container.Add(ws)
}

```

在`go-restful`中，Web 服务主要基于资源工作。所以在这里，我们定义了一个名为`Register`的函数在`TrainResource`上，接受容器作为参数。我们创建了一个新的`WebService`并为其添加路径。路径是 URL 端点，路由是附加到函数处理程序的路径参数或查询参数。`ws`是用于提供`Train`资源的 Web 服务。我们将三个 REST 方法，即`GET`、`POST`和`DELETE`分别附加到三个函数处理程序上，分别是`getTrain`、`createTrain`和`removeTrain`：

```go

Path("/v1/trains").
Consumes(restful.MIME_JSON).
Produces(restful.MIME_JSON)

```

这些语句表明 API 将只接受请求中的`Content-Type`为 application/JSON。对于所有其他类型，它会自动返回 415--媒体不支持错误。返回的响应会自动转换为漂亮的 JSON 格式。我们还可以有一个格式列表，比如 XML、JSON 等等。`go-restful`提供了这个功能。

现在，让我们定义函数处理程序：

```go

// GET http://localhost:8000/v1/trains/1
func (t TrainResource) getTrain(request *restful.Request, response *restful.Response) {
    id := request.PathParameter("train-id")
    err := DB.QueryRow("select ID, DRIVER_NAME, OPERATING_STATUS FROM train where id=?", id).Scan(&t.ID, &t.DriverName, &t.OperatingStatus)
    if err != nil {
        log.Println(err)
        response.AddHeader("Content-Type", "text/plain")
        response.WriteErrorString(http.StatusNotFound, "Train could not be found.")
    } else {
        response.WriteEntity(t)
    }
}
// POST http://localhost:8000/v1/trains
func (t TrainResource) createTrain(request *restful.Request, response *restful.Response) {
    log.Println(request.Request.Body)
    decoder := json.NewDecoder(request.Request.Body)
    var b TrainResource
    err := decoder.Decode(&b)
    log.Println(b.DriverName, b.OperatingStatus)
    // Error handling is obvious here. So omitting...
    statement, _ := DB.Prepare("insert into train (DRIVER_NAME, OPERATING_STATUS) values (?, ?)")
    result, err := statement.Exec(b.DriverName, b.OperatingStatus)
    if err == nil {
        newID, _ := result.LastInsertId()
        b.ID = int(newID)
        response.WriteHeaderAndEntity(http.StatusCreated, b)
    } else {
        response.AddHeader("Content-Type", "text/plain")
        response.WriteErrorString(http.StatusInternalServerError, err.Error())
    }
}
// DELETE http://localhost:8000/v1/trains/1
func (t TrainResource) removeTrain(request *restful.Request, response *restful.Response) {
    id := request.PathParameter("train-id")
    statement, _ := DB.Prepare("delete from train where id=?")
    _, err := statement.Exec(id)
    if err == nil {
        response.WriteHeader(http.StatusOK)
    } else {
        response.AddHeader("Content-Type", "text/plain")
        response.WriteErrorString(http.StatusInternalServerError, err.Error())
    }
}

```

所有这些 REST 方法都在`TimeResource`结构的实例上定义。谈到`GET`处理程序，它将`Request`和`Response`作为其参数传递。可以使用`request.PathParameter`函数获取路径参数。传递给它的参数将与我们在前面的代码段中添加的路由保持一致。也就是说，`train-id`将被返回到处理程序中，以便我们可以剥离它并将其用作从我们的 SQLite 数据库中获取记录的条件。

在`POST`处理程序函数中，我们使用 JSON 包的`NewDecoder`函数解析请求体。`go-restful`没有一个函数可以解析客户端发布的原始数据。有函数可用于剥离查询参数和表单参数，但这个缺失了。所以，我们编写了自己的逻辑来剥离和解析 JSON 主体，并使用这些结果将数据插入我们的 SQLite 数据库中。该处理程序正在为请求中提供的细节创建一个`db`记录。

如果您理解前两个处理程序，`DELETE`函数就很明显了。我们使用`DB.Prepare`创建一个`DELETE` SQL 命令，并返回 201 状态 OK，告诉我们删除操作成功了。否则，我们将实际错误作为服务器错误发送回去。现在，让我们编写主函数处理程序，这是我们程序的入口点：

```go

func main() {
    var err error
    DB, err = sql.Open("sqlite3", "./railapi.db")
    if err != nil {
        log.Println("Driver creation failed!")
    }
    dbutils.Initialize(DB)
    wsContainer := restful.NewContainer()
    wsContainer.Router(restful.CurlyRouter{})
    t := TrainResource{}
    t.Register(wsContainer)
    log.Printf("start listening on localhost:8000")
    server := &http.Server{Addr: ":8000", Handler: wsContainer}
    log.Fatal(server.ListenAndServe())
}

```

这里的前四行执行与数据库相关的工作。然后，我们使用`restful.NewContainer`创建一个新的容器。然后，我们使用称为`CurlyRouter`的路由器（它允许我们在路径中使用`{train_id}`语法来设置路由）来为我们的容器设置路由。接下来，我们创建了`TimeResource`结构的实例，并将该容器传递给`Register`方法。该容器确实可以充当 HTTP 处理程序；因此，我们可以轻松地将其传递给`http.Server`。

使用 `request.QueryParameter` 从 HTTP 请求中获取查询参数在`go-restful`处理程序中。

此代码可在 GitHub 仓库中找到。现在，当我们在`$GOPATH/src/github.com/narenaryan`目录中运行`main.go`文件时，我们会看到这个：

```go

go run railAPI/main.go

```

并进行 curl `POST`请求创建一个火车：

```go

curl -X POST \
    http://localhost:8000/v1/trains \
    -H 'cache-control: no-cache' \
    -H 'content-type: application/json' \
    -d '{"driverName": "Menaka", "operatingStatus": true}'

```

这会创建一个带有驾驶员和操作状态详细信息的新火车。响应是新创建的分配了火车`ID`的资源：

```go

{
    "ID": 1,
    "DriverName": "Menaka",
    "OperatingStatus": true
}

```

现在，让我们进行一个 curl 请求来检查`GET`：

```go

CURL -X GET "http://localhost:8000/v1/trains/1"

```

您将看到以下 JSON 输出：

```go

{
    "ID": 1,
    "DriverName": "Menaka",
    "OperatingStatus": true
}

```

可以对发布的数据和返回的 JSON 使用相同的名称，但为了显示两个操作之间的区别，使用了不同的变量名称。现在，使用`DELETE`API 调用删除我们在前面代码片段中创建的资源：

```go

CURL -X DELETE "http://localhost:8000/v1/trains/1"

```

如果操作成功，它不会返回任何响应体，而是返回`Status 200 ok`。现在，如果我们尝试对`ID`为 1 的火车进行`GET`操作，它会返回以下响应：

```go

Train could not be found.

```

这些实现可以扩展到`PUT`和`PATCH`。我们需要在`Register`方法中添加两个额外的路由，并定义相应的处理程序。在这里，我们为`Train`资源创建了一个 web 服务。类似地，还可以为`Station`和`Schedule`表上的 CRUD 操作创建 web 服务。这项任务就留给读者去探索。

`go-restful`是一个轻量级的库，在创建 RESTful 服务时具有强大的功能。主题是将资源（模型）转换成可消费的 API。使用其他繁重的框架可能会加快开发速度，但因为代码包装的原因，API 可能会变得更慢。`go-restful`是一个用于 API 创建的精简且底层的包。

`go-restful` 还提供了对使用**swagger**文档化 REST API 的内置支持。它是一个运行并生成我们构建的 REST API 文档模板的工具。通过将其与基于`go-restful`的 web 服务集成，我们可以实时生成文档。欲了解更多信息，请访问[github.com/emicklei/go-restful-swagger12](https://github.com/emicklei/go-restful-swagger12)。

# 使用 Gin 框架构建 RESTful API

`Gin-gonic`是基于`httprouter`的框架。我们在第二章*处理我们的 REST 服务的路由*中学习了`httprouter`。它是一个 HTTP 多路复用器，类似于 Gorilla Mux，但更快。 `Gin`允许以清晰的方式创建 REST 服务的高级 API。 `Gin`将自己与另一个名为`martini`的 web 框架进行比较。所有 web 框架都允许我们做更多的事情，如模板化和 web 服务器设计，除了服务创建。使用以下命令安装`Gin`包：

```go

go get gopkg.in/gin-gonic/gin.v1

```

让我们写一个简单的 hello world 程序在`Gin`中熟悉`Gin`的构造。文件名是`ginBasic.go`：

```go

package main
import (
    "time"
    "github.com/gin-gonic/gin"
)
func main() {
r := gin.Default()
    /* GET takes a route and a handler function
    Handler takes the gin context object
    */
    r.GET("/pingTime", func(c *gin.Context) {
        // JSON serializer is available on gin context
        c.JSON(200, gin.H{
            "serverTime": time.Now().UTC(),
        })
    })
    r.Run(":8000") // 在 0.0.0.0:8080 上监听并提供服务
}

```

这个简单的服务器尝试实现一个向客户端提供 UTC 服务器时间的服务。我们在第三章*使用中间件和 RPC 工作*中实现了一个这样的服务。但在这里，如果你看，`Gin`允许你用几行代码做很多事情；所有的样板细节都被省去了。来到前面的程序，我们用`gin.Default`函数创建了一个路由器。然后，我们附加了与 REST 动词相对应的路由，就像在`go-restful`中做的那样；一个到函数处理程序的路由。然后，我们通过传递要运行的端口来调用`Run`函数。默认端口将是`8080`。

`c`是保存单个请求信息的`gin.Context`。我们可以使用`context.JSON`函数将数据序列化为 JSON，然后发送回客户端。现在，如果我们运行并查看前面的程序：

```go

go run ginExamples/ginBasic.go

```

发出一个 curl 请求：

```go

curl -X GET "http://localhost:8000/pingTime"

Output
=======
{"serverTime":"2017-06-11T03:59:44.135062688Z"}

```

与此同时，我们运行`Gin`服务器的控制台上漂亮地呈现了调试消息：

![](img/54d2bbb0-6c1c-466d-b187-1828e5283490.png)

这是显示端点、请求的延迟和 REST 方法的 Apache 风格的调试日志。

为了在生产模式下运行`Gin`，设置`GIN_MODE = release`环境变量。然后控制台输出将被静音，日志文件可用于监视日志。

现在，让我们在`Gin`中编写我们的 Rail API，以展示如何使用`Gin`框架实现完全相同的东西。我将使用相同的项目布局，将我的新项目命名为`railAPIGin`，并使用`dbutils`如它所在。首先，让我们准备好我们程序的导入：

```go

package main
import (
    "database/sql"
    "log"
    "net/http"
    "github.com/gin-gonic/gin"
    _ "github.com/mattn/go-sqlite3"
    "github.com/narenaryan/dbutils"
)

```

我们导入了`sqlite3`和`dbutils`用于与数据库相关的操作。我们导入了`gin`用于创建我们的 API 服务器。`net/http`在提供与响应一起发送的直观状态代码方面很有用。看一下下面的代码片段：

```go

// DB Driver visible to whole program
var DB *sql.DB
// StationResource holds information about locations
type StationResource struct {
    ID int `json:"id"`
    Name string `json:"name"`
    OpeningTime string `json:"opening_time"`
    ClosingTime string `json:"closing_time"`
}

```

我们创建了一个数据库驱动程序，该驱动程序对所有处理程序函数都可用。 `StationResource`是我们从请求体和来自数据库的数据解码而来的 JSON 的占位符。如果你注意到了，它与`go-restful`的示例略有不同。现在，让我们编写实现`GET`、`POST`和`DELETE`方法的`station`资源的处理程序：

```go

// GetStation returns the station detail
    func GetStation(c *gin.Context) {
    var station StationResource
    id := c.Param("station_id")
    err := DB.QueryRow("select ID, NAME, CAST(OPENING_TIME as CHAR), CAST(CLOSING_TIME as CHAR) from station where id=?", id).Scan(&station.ID, &station.Name, &station.OpeningTime, &station.ClosingTime)
    if err != nil {
        log.Println(err)
        c.JSON(500, gin.H{
            "error": err.Error(),
        })
    } else {
        c.JSON(200, gin.H{
        "result": station,
        })
    }
}
// CreateStation handles the POST
func CreateStation(c *gin.Context) {
    var station StationResource
    // Parse the body into our resrource
    if err := c.BindJSON(&station); err == nil {
        // Format Time to Go time format
        statement, _ := DB.Prepare("insert into station (NAME, OPENING_TIME, CLOSING_TIME) values (?, ?, ?)")
        result, _ := statement.</span>Exec(station.Name, station.OpeningTime, station.ClosingTime)
        if err == nil {
            newID, _ := result.LastInsertId()
            station.ID = int(newID)
            c.JSON(http.StatusOK, gin.H{
                "result": station,
            })
        } else {
            c.String(http.StatusInternalServerError, err.Error())
        }
    } else {
        c.String(http.StatusInternalServerError, err.Error())
    }
}
// RemoveStation handles the removing of resource
func RemoveStation(c *gin.Context) {
    id := c.Param("station-id")
    statement, _ := DB.Prepare("delete from station where id=?")
    _, err := statement.Exec(id)
    if err != nil {
        log.Println(err)
        c.JSON(500, gin.H{
            "error": err.Error(),
        })
    } else {
        c.String(http.StatusOK, "")
    }
}

```

在`GetStation`中，我们使用`c.Param`来剥离`station_id`路径参数。之后，我们使用该 ID 从 SQLite3 站点表中检索数据库记录。如果您仔细观察，SQL 查询有点不同。我们使用`CAST`方法将 SQL `TIME`字段检索为 Go 可以正确消耗的字符串。如果删除类型转换，将引发恐慌错误，因为我们尝试在运行时将`TIME`字段加载到 Go 字符串中。为了给您一个概念，`TIME`字段看起来像*8:00:00*，*17:31:12*，等等。接下来，如果没有错误，我们将使用`gin.H`方法返回结果。

在`CreateStation`中，我们试图执行插入查询。但在此之前，为了从`POST`请求的主体中获取数据，我们使用了一个名为`c.BindJSON`的函数。这个函数将数据加载到传递的结构体中。这意味着站点结构将加载来自主体提供的数据。这就是为什么`StationResource`具有 JSON 推断字符串来告诉期望的键值是什么。例如，这是`StationResource`结构的一个字段，带有推断字符串。

```go

ID int `json:"id"`

```

在收集数据后，我们正在准备一个数据库插入语句并执行它。结果是插入记录的 ID。我们使用该 ID 将站点详细信息发送回客户端。在`RemoveStation`中，我们执行`DELETE` SQL 查询。如果操作成功，则返回`200 OK`状态。否则，我们会发送适当的原因给`500 Internal Server Error`。

现在来看主程序，它首先运行数据库逻辑以确保表已创建。然后，它尝试创建`Gin`路由器并向其添加路由：

```go

func main() {
    var err error
    DB, err = sql.Open("sqlite3", "./railapi.db")
    if err != nil {
        log.Println("Driver creation failed!")
    }
    dbutils.Initialize(DB)
    r := gin.Default()
    // Add routes to REST verbs
    r.GET("/v1/stations/:station_id", GetStation)
    r.POST("/v1/stations", CreateStation)
    r.DELETE("/v1/stations/:station_id", RemoveStation)
    r.Run(":8000") // 默认监听并在 0.0.0.0:8080 上提供服务
}

```

我们正在使用`Gin`路由器注册`GET`、`POST`和`DELETE`路由。然后，我们将路由和处理程序传递给它们。最后，我们使用 Gin 的`Run`函数以`8000`作为端口启动服务器。运行前述程序，如下所示：

```go

go run railAPIGin/main.go

```

现在，我们可以通过执行`POST`请求来插入新记录：

```go

curl -X POST \
    http://localhost:8000/v1/stations \
    -H 'cache-control: no-cache' \
    -H 'content-type: application/json' \
    -d '{"name":"Brooklyn", "opening_time":"8:12:00", "closing_time":"18:23:00"}'

```

它返回：

```go

{"result":{"id":1,"name":"Brooklyn","opening_time":"8:12:00","closing_time":"18:23:00"}}

```

现在尝试使用`GET`获取详细信息：

```go

CURL -X GET "http://10.102.78.140:8000/v1/stations/1"

Output
======
{"result":{"id":1,"name":"Brooklyn","opening_time":"8:12:00","closing_time":"18:23:00"}}

```

我们也可以使用以下命令删除站点记录：

```go

CURL -X DELETE "http://10.102.78.140:8000/v1/stations/1"

```

它返回`200 OK`状态，确认资源已成功删除。正如我们已经讨论的那样，`Gin`提供了直观的调试功能，显示附加的处理程序，并使用颜色突出显示延迟和 REST 动词：

![](img/c1f2942f-5dfc-4fda-b9d3-7a9470ca687d.png)

例如，`200`是绿色的，`404`是黄色的，`DELETE`是红色的，等等。`Gin`提供了许多其他功能，如路由的分类、重定向和中间件函数。

如果您要快速创建 REST Web 服务，请使用`Gin`框架。您还可以将其用于许多其他用途，如静态文件服务等。请记住，它是一个完整的 Web 框架。在 Gin 中获取查询参数，请使用以下方法在`Gin`上下文对象上：`c.Query("param")`。

# 使用 Revel.go 构建一个 RESTful API

Revel.go 也是一个像 Python 的 Django 一样完整的 Web 框架。它比 Gin 还要早，并被称为高生产力的 Web 框架。它是一个异步的、模块化的、无状态的框架。与 `go-restful` 和 `Gin` 框架不同，Revel 直接生成了一个可用的脚手架。

使用以下命令安装`Revel.go`：

```go

go get github.com/revel/revel

```

为了运行脚手架工具，我们应该安装另一个附加包：

```go

go get github.com/revel/cmd/revel

```

确保 `$GOPATH/bin` 在您的 `PATH` 变量中。一些外部包将二进制文件安装在 `$GOPATH/bin` 目录中。如果在路径中，我们可以在系统范围内访问可执行文件。在这里，Revel 将安装一个名为`revel`的二进制文件。在 Ubuntu 或 macOS X 上，您可以使用以下命令执行：

```go

export PATH=$PATH:$GOPATH/bin

```

将上面的内容添加到 `~/.bashrc` 以保存设置。在 Windows 上，您需要直接调用可执行文件的位置。现在我们已经准备好开始使用 Revel 了。让我们在 `github.com/narenaryan` 中创建一个名为 `railAPIRevel` 的新项目：

```go

revel new railAPIRevel

```

这样就可以在不写一行代码的情况下创建一个项目脚手架。这就是 Web 框架在快速原型设计中的抽象方式。Revel 项目布局树看起来像这样：

```go

conf/         Configuration directory
app.conf      Main app configuration file
routes        路由定义文件
app/          应用程序源
init.go       拦截器注册
controllers/  这里放置应用程序控制器
views/        模板目录
messages/     消息文件
public/       公共静态资产
css/          CSS 文件
js/           Javascript 文件
images/       图像文件
tests/        测试套件

```

在所有那些样板目录中，有三个重要的东西用于创建一个 API。那是：

+   `app/controllers`

+   `conf/app.conf`

+   `conf/routes`

控制器是执行 API 逻辑的逻辑容器。`app.conf` 允许我们设置 `host`、`port`、`dev` 模式/生产模式等。`routes` 定义了端点、REST 动词和函数处理程序（这里是控制器的函数）。这意味着在控制器中定义一个函数，并在路由文件中将其附加到路由上。

让我们使用我们之前看到的 `go-restful` 的相同例子，为列车创建一个 API。但由于冗余，我们将删除数据库逻辑。稍后我们将看到如何使用 Revel 为 API 构建 `GET`、`POST` 和 `DELETE` 操作。现在，将路由文件修改为这样：

```go

# 路由配置
#
# 此文件定义了所有应用程序路由（优先级较高的路由优先）
#

module:testrunner
# module:jobs

GET /v1/trains/:train-id App.GetTrain
POST /v1/trains App.CreateTrain
DELETE /v1/trains/:train-id App.RemoveTrain

```

语法可能看起来有点新。这是一个配置文件，我们只需以这种格式定义一个路由：

```go

VERB END_POINT HANDLER

```

我们还没有定义处理程序。在端点中，路径参数使用`:param` 注释进行访问。这意味着对于文件中的 `GET` 请求，`train-id` 将作为 `path` 参数传递。现在，转到 `controllers` 文件夹，并将 `app.go` 文件中的现有控制器修改为这样：

```go

package controllers
import (
    "log"
    "net/http"
    "strconv"
    "github.com/revel/revel"
)
type App struct {
    *revel.Controller
}
// TrainResource 是用于保存铁路信息的模型
type TrainResource struct {
    ID int `json:"id"`
    DriverName string `json:"driver_name"`
    OperatingStatus bool `json:"operating_status"`
}
// GetTrain 处理对火车资源的 GET
func (c App) GetTrain() revel.Result {
    var train TrainResource
    // 从路径参数中获取值。
    id := c.Params.Route.Get("train-id")
    // 使用此 ID 从数据库查询并填充 train 表....
    train.ID，_ = strconv.Atoi（id）
    train.DriverName = "Logan" // 来自数据库
    train.OperatingStatus = true // 来自数据库
    c.Response.Status = http.StatusOK
    return c.RenderJSON(train)
}
// CreateTrain 处理对火车资源的 POST
func (c App) CreateTrain() revel.Result {
    var train TrainResource
    c.Params.BindJSON(&train)
    // 使用 train.DriverName 和 train.OperatingStatus 插入到 train 表中....
    train.ID = 2
    c.Response.Status = http.StatusCreated
    return c.RenderJSON(train)
}
// RemoveTrain 实现对火车资源的 DELETE
func (c App) RemoveTrain() revel.Result {
    id := c.Params.Route.Get("train-id")
    // 使用 ID 从 train 表中删除记录....
    log.Println("成功删除资源：", id)
    c.Response.Status = http.StatusOK
    return c.RenderText("")
}

```

我们在文件 `app.go` 中创建了 API 处理程序。这些处理程序的名称应与我们在路由文件中提到的名称匹配。我们可以使用带有 `*revel.Controller` 作为其成员的结构创建一个 Revel 控制器。然后，我们可以向其附加任意数量的处理程序。控制器保存了传入 HTTP 请求的信息，因此我们可以在处理程序中使用信息，如查询参数、路径参数、JSON 主体、表单数据等。

我们正在定义 `TrainResource` 作为一个数据持有者。在 `GetTrain` 中，我们使用 `c.Params.Route.Get` 函数获取路径参数。该函数的参数是我们在路由文件中指定的路径参数（这里是 `train-id`）。该值将是一个字符串。我们需要将其转换为 `Int` 类型以与 `train.ID` 进行映射。然后，我们使用 `c.Response.Status` 变量（而不是函数）将响应状态设置为 `200 OK`。`c.RenderJSON` 接受一个结构体并将其转换为 JSON 主体。

在 `CreateTrain` 中，我们添加了 `POST` 请求逻辑。我们创建了一个新的 `TrainResource` 结构体，并将其传递给一个名为 `c.Params.BindJSON` 的函数。`BindJSON` 的作用是从 JSON `POST` 主体中提取参数，并尝试在结构体中查找匹配的字段并填充它们。当我们将 Go 结构体编组为 JSON 时，字段名将按原样转换为键。但是，如果我们将 `jason:"id"` 字符串格式附加到任何结构字段上，它明确表示从该结构编组的 JSON 应具有键 `id`，而不是 **ID**。在使用 JSON 时，这是 Go 中的一个良好做法。然后，我们向 HTTP 响应添加了一个 201 创建的状态。我们返回火车结构体，它将在内部转换为 JSON。

`RemoveTrain` 处理程序逻辑与 `GET` 类似。一个微妙的区别是没有发送任何内容。正如我们之前提到的，数据库 CRUD 逻辑在上述示例中被省略。读者可以通过观察我们在 `go-restful` 和 `Gin` 部分所做的工作来尝试添加 SQLite3 逻辑。

最后，默认端口号是 `9000`，Revel 服务器运行的配置更改端口号在 `conf/app.conf` 文件中。让我们遵循在 `8000` 上运行我们的应用程序的传统。因此，将文件的 `http` 端口部分修改为以下内容。这告诉 Revel 服务器在不同的端口上运行：

```go

......

# 要监听的 IP 地址。
http.addr = "0.0.0.0"
# 要监听的端口。
http.port = 8000 # 从 9000 更改为 8000 或任何端口
# 是否使用 SSL。
http.ssl = false
......

```

现在，我们可以使用以下命令运行 Revel API 服务器：

```go

revel run github.com/narenaryan/railAPIRevel

```

我们的应用服务器在 `http://localhost:8000` 上启动。现在，让我们进行一些 API 请求：

```go

CURL -X GET "http://10.102.78.140:8000/v1/trains/1"

output
=======
{
    "id": 1,
    "driver_name": "Logan",
    "operating_status": true
}

```

`POST` 请求：


```go

curl -X POST \
    http://10.102.78.140:8000/v1/trains \
    -H 'cache-control: no-cache' \
    -H 'content-type: application/json' \
    -d '{"driver_name":"Magneto", "operating_status": true}'

output
======
{
    "id": 2,
    "driver_name": "Magneto",
    "operating_status": true
}

```

`DELETE`与`GET`相同，但不返回主体。这里，代码是为了展示如何处理请求和响应。请记住，Revel 不仅仅是一个简单的 API 框架。它是一个类似于 Django（Python）或 Ruby on Rails 的完整的 Web 框架。我们在 Revel 中内置了模板，测试和许多其他功能。

确保为`GOPATH/user`创建一个新的 Revel 项目。否则，当运行项目时，Revel 命令行工具可能找不到项目。

我们在本章中看到的所有 Web 框架都支持中间件。 `go-restful`将其中间件命名为`Filters`，而`Gin`将其命名为自定义中间件。 Revel 将其中间件拦截器。中间件在函数处理程序之前和之后分别读取或写入请求和响应。在第三章中，*使用中间件和 RPC*，我们将更多地讨论中间件。

# 摘要

在本章中，我们尝试使用 Go 中的一些 Web 框架构建了一个地铁轨道 API。最受欢迎的是`go-restful`，`Gin Gonic`和`Revel.go`。我们首先学习了如何在 Go 应用程序中进行第一个数据库集成。我们选择了 SQLite3，并尝试使用`go-sqlite3`库编写了一个示例应用程序。

接下来，我们探索了`go-restful`，并详细了解了如何创建路由和处理程序。`go-restful`具有在资源之上构建 API 的概念。它提供了一种直观的方式来创建可以消耗和产生各种格式（如 XML 和 JSON）的 API。我们使用火车作为资源，并构建了一个在数据库上执行 CRUD 操作的 API。我们解释了为什么`go-restful`轻量级，并且可以用来创建低延迟的 API。接下来，我们看到了`Gin`框架，并尝试重复相同的 API，但是创建了一个围绕车站资源的 API。我们看到了如何在 SQL 数据库时间字段中存储时间。我们建议使用`Gin`来快速原型化您的 API。

最后，我们尝试使用`Revel.go`网络框架在火车资源上创建另一个 API。我们开始创建一个项目，检查了目录结构，然后继续编写一些服务（没有`db`集成）。我们还看到了如何运行应用程序并使用配置文件更改端口。

本章的主题是为您提供一些创建 RESTful API 的精彩框架。每个框架可能有不同的做事方式，选择您感到舒适的那个。当您需要一个端到端的网络应用程序（模板和用户界面）时，请使用`Revel.go`，当您需要快速创建 REST 服务时，请使用`Gin`，当 API 的性能至关重要时，请使用`go-rest`。
