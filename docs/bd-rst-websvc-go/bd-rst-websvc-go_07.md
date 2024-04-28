# 第七章：使用 PostgreSQL、JSON 和 Go 进行工作

在本章中，我们将从宏观角度看 SQL。在之前的章节中，我们讨论了 SQLite3，这是一个用于快速原型设计的小型数据库。但是，当涉及到生产级应用程序时，人们更喜欢 MySQL 或 PostgreSQL。在 Web 应用程序领域，两者都经过了充分验证。首先，我们将讨论 PostgreSQL 的内部，然后转向在 Go 中编写数据库模型。然后，我们将尝试通过一个实例来实现 URL 缩短服务。

在本章中，我们将涵盖以下主题：

+   介绍 PostgreSQL 数据库

+   安装 PostgreSQL 并创建用户和数据库

+   了解`pq`，Go 中的数据库驱动程序

+   使用 PostgreSQL 和 Base62 算法实现 URL 缩短服务

+   探索 PostgreSQL 中的 JSON 存储

+   介绍`gorm`，Go 的强大 ORM

+   实施电子商务 REST API

# 获取代码

您可以在以下网址找到本章的代码示例：[`github.com/narenaryan/gorestful/tree/master/chapter7`](https://github.com/narenaryan/gorestful/tree/master/chapter7)。在上一章中，我们讨论了协议缓冲区和 GRPC。但是在这里，我们回到了使用 JSON 的 REST API，并看看 PostgreSQL 如何补充 JSON。

# 安装 PostgreSQL 数据库

PostgreSQL 是一个可以安装在多个平台上的开源数据库。在 Ubuntu 上，可以使用以下命令进行安装：

将存储库添加到软件包列表中：

```go

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list' 
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -

```

要更新软件包列表：

```go

sudo apt-get update
apt-get install postgresql postgresql-contrib
```

这将在 Ubuntu 机器上安装数据库并在端口`5432`上启动服务器。现在，为了进入数据库 shell，使用以下命令。PostgreSQL 创建一个名为`postgres`的默认用户以登录。看一下以下命令：

```go
sudo su - postgres
```

现在用户可以访问数据库。使用`psql`命令启动 PostgreSQL shell：

```go
psql
```

这表明 PostgreSQL 与其他类似数据库（如 MySQL 或 SQLite3）相比，采用了不同的进入 shell 的方法。在 Windows 上，通过单击二进制安装程序文件来进行安装。这是一个基于 GUI 的安装，应提供超级用户的端口和密码。安装数据库后，我们可以使用**pgAdmin3**工具进行检查。macOS X 的设置与 Ubuntu 类似，只是安装是通过 Homebrew 完成的。看一下以下命令：

```go
brew install postgresql
```

然后，通过使用以下命令使数据库服务器在系统重新启动时运行：

```go
pg_ctl -D /usr/local/var/postgres start && brew services start postgresql
```

现在，PostgreSQL 服务器开始运行，并且可以在 macOS X 上存储和检索数据。

# 在 PostgreSQL 中添加用户和数据库

现在，我们应该知道如何创建新用户和数据库。为此，我们将以 Ubuntu/Mac 为一般示例。我们在一个名为`psql`的 shell 中执行此操作。使用`\?`命令可以在`psql`中看到所有可用命令。为了进入`psql`，首先切换到`postgres`用户。在 Ubuntu 上，您可以使用以下命令来执行：

```go
sudo su postgres
```

现在，它将我们转换为一个名为`postgres`的用户。然后，使用`psql`命令启动`psql` shell。如果在其中输入`\?`，您将看到所有可用命令的输出：

![](img/a59458f4-5721-4fac-906b-ce4696b696f1.png)

要列出所有可用用户及其权限，您将在 shell 帮助的`Informational`部分中找到一个命令，即：

```go
\du - List roles
```

角色是授予用户的访问权限。列表中的默认角色是`postgres`：

```go
postgres=# \du

 List of roles 
 Role name |      Attributes               | Member of 
-----------+------------------------------------------------------------+-----------
 postgres | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

上述命令列出了角色（用户）及其属性（角色允许执行的操作）和其他选项。要添加新用户，我们只需输入此`psql`命令：

```go
CREATE ROLE naren with LOGIN PASSWORD 'passme123';
```

这将创建一个名为`naren`的新用户和密码`passme123.`现在，使用以下命令为用户授予创建数据库和进一步角色的权限：

```go
ALTER USER naren CREATEDB, CREATEROLE;
```

要删除用户，只需在相同上下文中使用`DROP`命令：

```go
DROP ROLE naren;
```

不要尝试更改默认`postgres`用户的密码。它旨在成为一个 sudo 帐户，不应该作为普通用户保留。相反，创建一个角色并为其分配所需的权限。

现在我们知道如何创建一个角色。让我们看看一些更多的 CRUD 命令，这些命令实际上是我们在其他关系数据库中看到的 SQL 命令。看一下下表：

| **操作** | **SQL 命令** |
| --- | --- |
| 创建数据库 |

```go
CREATE DATABASE mydb;
```

|

| 创建表 |
| --- |

```go
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);
```

|

| 插入到表中 |
| --- |

```go
INSERT INTO products VALUES (1, 'Rice', 5.99);
```

|

| 更新表 |
| --- |

```go
UPDATE products SET price = 10 WHERE price = 5.99;
```

|

| 从表中删除 |
| --- |

```go
DELETE FROM products WHERE price = 5.99;
```

|

现在，让我们从 Go 中看看如何与 PostgreSQL 交流，并尝试使用一个简单的例子来执行前面的操作。

# pq，一个纯 PostgreSQL 数据库驱动程序

在之前的章节中，当我们处理 SQLite3 时，我们使用了一个名为`go-sqlite3`的外部库。同样，有一个数据库驱动程序库可用于连接 Go 和 PostgreSQL。该库称为`pq`。我们可以使用以下命令安装该库：

```go
go get github.com/lib/pq
```

获得这个库之后，我们需要以与 SQLite3 相似的方式使用它。API 将与 Go 的`database/sql`包一致。为了创建一个新表，我们应该初始化 DB。要创建一个新数据库，只需在`psql` shell 中输入以下命令，如下所示；这是一次性的事情：

```go
CREATE DATABASE mydb;
```

现在，我们将编写一个小的代码示例，解释了`pq`驱动程序的用法。在你的`$GOPATH`中创建一个名为`models`的目录。在这里，我的`GOPATH`是`/home/naren/workspace/`。与前几章中的所有示例一样，我们将在`src/`目录中创建我们的包和应用程序源代码：

```go
mkdir github.com/narenaryan/src/models
```

现在，添加一个名为`web_urls.go`的文件。这个文件将包含表创建逻辑：

```go
package models

import (
        "database/sql"
        "log"
        _ "github.com/lib/pq"
)

func InitDB() (*sql.DB, error) {
        var err error
        db, err := sql.Open("postgres", "postgres://naren:passme123@localhost/mydb?sslmode=disable")
        if err != nil {
                return nil, err
        } else {
                // Create model for our URL service
                stmt, err := db.Prepare("CREATE TABLE WEB_URL(ID SERIAL PRIMARY KEY, URL TEXT NOT NULL);")
                if err != nil {
                        log.Println(err)
                        return nil, err
                }
                res, err := stmt.Exec()
                log.Println(res)
                if err != nil {
                        log.Println(err)
                        return nil, err
                }
                return db, nil
        }
}
```

我们在这里导入了`pq`库。我们使用`sql.Open`函数来启动一个新的数据库连接池。如果你观察连接字符串，它由多个部分组成。看一下下图：

![](img/c5cf7c9a-a38f-4642-a72b-54de1f5c7c41.png)

连接字符串应该包括数据库类型、`username:password`对、数据库服务器 IP 和 sslmode 设置。然后我们创建一个名为`web_url`的表。所有的错误处理程序都在那里，以指定如果出现问题。`InitDB`函数将数据库连接对象返回给导入该函数的任何程序。让我们编写主程序来使用这个包：

```go
package main

import (
       "log"
      "github.com/narenaryan/models"
)

func main() {
  db, err := models.InitDB()
  if err != nil {
    log.Println(db)
  }
}
```

该程序导入了`models`包，并使用了其中的`InitDB`函数。我们只是打印了数据库连接，这将是一个地址。如果你运行程序，你会看到对象的地址被打印出来：

```go
go run main.go
```

这将在`mydb`数据库中创建一个`web_url`表。我们可以通过进入`psql` shell 并输入以下内容来交叉检查：

```go
\c mydb \dt
```

它将用户连接到`mydb`数据库并列出所有可用的表，如下面的代码片段所示：

```go
You are now connected to database "mydb" as user "postgres".
 List of relations
 Schema | Name | Type | Owner
--------+---------+-------+-------
 public | web_url | table | naren
(1 row)
```

在 PostgreSQL 中，AUTO INCREMENT 类型需要在为表创建提供模式时替换为 SERIAL。

# 使用 Postgres 和 pq 实现 URL 缩短服务

让我们编写 URL 缩短服务来解释我们在前一节讨论的所有概念。在那之前，让我们设计一个实现 Base62 算法的包，其中包括编码/解码函数。URL 缩短技术需要 Base62 算法来将长 URL 转换为短 URL，反之亦然。然后，我们编写一个实例来展示这种编码是如何工作的。在`GOPATH`中创建一个名为`base62`的目录：

```go
mkdir $GOPATH/src/github.com/narenaryan/base62
```

现在，添加一个名为`encodeutils.go`的文件，其中包含我们的编码和解码函数。

# 定义 Base62 算法

我们在前几章中看到了 Base62 算法的工作原理。这是该算法的坚实实现。这个逻辑是纯数学的，可以在网上找到。看一下下面的代码：

```go
package base62

import (
     "math"
     "strings"
)

const base = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
const b = 62

// Function encodes the given database ID to a base62 string
func ToBase62(num int) string{
    r := num % b
    res := string(base[r])
    div := num / b
    q := int(math.Floor(float64(div)))

    for q != 0 {
        r = q % b
        temp := q / b
        q = int(math.Floor(float64(temp)))
        res = string(base[int(r)]) + res
    }

    return string(res)
}

// Function decodes a given base62 string to datbase ID
func ToBase10(str string) int{
    res := 0
    for _, r := range str {
        res = (b * res) + strings.Index(base, string(r))
    }
    return res
}
```

在上述程序中，我们定义了两个名为`ToBase62`和`ToBase10`的函数。第一个函数接受一个整数并生成一个`base62`字符串，而后一个函数则反转了这个效果；也就是说，它接受一个`base62`字符串并给出原始数字。为了说明这一点，让我们创建一个简单的程序，使用这两个函数来展示编码/解码：

```go
vi $GOPATH/src/github.com/narenaryan/usebase62.go
```

将以下内容添加到其中：

```go
package main

import (
      "log"
      base62 "github.com/narenaryan/base62"
)

func main() {
  x := 100
  base62String := base62.ToBase62(x)
  log.Println(base62String)
  normalNumber := base62.ToBase10(base62String)
  log.Println(normalNumber)
}
```

在这里，我们使用了`base62`包中的函数，并尝试查看输出。如果我们使用以下命令运行这个程序（从`$GOPATH/src/github.com/narenaryan`）：

```go
go run usebase62.go
```

它打印出：

```go
2017/08/07 23:00:05 1C
2017/08/07 23:00:05 100
```

`100`的`base62`编码是`1C`。这是因为索引 100 在我们的`base62`逻辑中缩小为`1C`：

```go
const base = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
```

原始数字将用于映射此基本字符串中的字符。然后，将数字除以 62 以找出下一个字符。这种算法的美妙之处在于为每个给定的数字创建一个独特的、更短的字符串。我们使用这种技术将数据库 ID 传递到`ToBase62`算法中，并得到一个更短的字符串。每当 URL 缩短请求到达我们的服务器时，它应执行以下步骤：

1.  将 URL 存储在数据库中，并获取插入记录的 ID。

1.  将此 ID 作为 API 响应传递给客户端。

1.  每当客户端加载缩短的 URL 时，它会访问我们的 API 服务器。

1.  然后 API 服务器将短 URL 转换回数据库 ID，并从原始 URL 中获取记录。

1.  最后，客户端可以使用此 URL 重定向到原始站点。

我们将在这里编写一个 Go 项目，实现上述步骤。让我们组成程序。我正在为我们的项目创建一个目录结构。我们从前面的示例中获取处理编码/解码`base62`和数据库逻辑的文件。目录结构如下：

```go
urlshortener
├── main.go
├── models
│   └── models.go
└── utils
 └── encodeutils.go

2 directories, 3 files
```

将此目录复制到`$GOPATH/src/github.com/narenaryan`。再次小心。用你的用户名替换`narenaryan`。从前面的示例中复制`encodeutils.go`和`models.go`。然后，开始编写主程序：

```go
package main

import (
    "database/sql"
    "encoding/json"
    "io/ioutil"
    "log"
    "net/http"
    "time"

    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
    "github.com/narenaryan/urlshortener/models"
    base62 "github.com/narenaryan/urlshortener/utils"
)

// DB stores the database session imformation. Needs to be initialized once
type DBClient struct {
  db *sql.DB
}

// Model the record struct
type Record struct {
  ID  int    `json:"id"`
  URL string `json:"url"`
}

// GetOriginalURL fetches the original URL for the given encoded(short) string
func (driver *DBClient) GetOriginalURL(w http.ResponseWriter, r *http.Request) {
  var url string
  vars := mux.Vars(r)
  // Get ID from base62 string
  id := base62.ToBase10(vars["encoded_string"])
  err := driver.db.QueryRow("SELECT url FROM web_url WHERE id = $1", id).Scan(&url)
  // Handle response details
  if err != nil {
    w.Write([]byte(err.Error()))
  } else {
    w.WriteHeader(http.StatusOK)
    w.Header().Set("Content-Type", "application/json")
    responseMap := map[string]interface{}{"url": url}
    response, _ := json.Marshal(responseMap)
    w.Write(response)
  }
}

// GenerateShortURL adds URL to DB and gives back shortened string
func (driver *DBClient) GenerateShortURL(w http.ResponseWriter, r *http.Request) {
  var id int
  var record Record
  postBody, _ := ioutil.ReadAll(r.Body)
  json.Unmarshal(postBody, &record)
  err := driver.db.QueryRow("INSERT INTO web_url(url) VALUES($1) RETURNING id", record.URL).Scan(&id)
  responseMap := map[string]interface{}{"encoded_string": base62.ToBase62(id)}
  if err != nil {
    w.Write([]byte(err.Error()))
  } else {
    w.Header().Set("Content-Type", "application/json")
    response, _ := json.Marshal(responseMap)
    w.Write(response)
  }
}

func main() {
  db, err := models.InitDB()
  if err != nil {
    panic(err)
  }
  dbclient := &DBClient{db: db}
  if err != nil {
    panic(err)
  }
  defer db.Close()
  // Create a new router
  r := mux.NewRouter()
  // Attach an elegant path with handler
  r.HandleFunc("/v1/short/{encoded_string:[a-zA-Z0-9]*}", dbclient.GetOriginalURL).Methods("GET")
  r.HandleFunc("/v1/short", dbclient.GenerateShortURL).Methods("POST")
  srv := &http.Server{
    Handler: r,
    Addr:    "127.0.0.1:8000",
    // Good practice: enforce timeouts for servers you create!
    WriteTimeout: 15 * time.Second,
    ReadTimeout:  15 * time.Second,
  }
  log.Fatal(srv.ListenAndServe())
}
```

首先，我们导入了`postgres`库和其他必要的库。我们从模型中导入了数据库会话。接下来，我们导入了我们的编码/解码 base62 算法来实现我们的逻辑：

```go
// DB stores the database session imformation. Needs to be initialized once
type DBClient struct {
  db *sql.DB
}

// Model the record struct
type Record struct {
  ID  int    `json:"id"`
  URL string `json:"url"`
}
```

需要`DBClient`以便在各种函数之间传递数据库驱动程序。记录是类似于插入数据库的记录的结构。我们在我们的代码中定义了两个函数`GenerateShortURL`和`GetOriginalURL`，用于将 URL 添加到数据库，然后从数据库中获取它。正如我们已经解释了 URL 缩短的内部技术，使用此服务的客户端将得到必要的响应。让我们在跳入更多细节之前运行程序并查看输出：

```go
go run $GOPATH/src/github.com/narenaryan/urlshortener/main.go
```

如果您的`$GOPATH/bin`已经在系统的`PATH`变量中，我们可以首先安装二进制文件，然后像这样运行它：

```go
go install github.com/narenaryan/urlshortener/main.go
```

然后只是程序名称：

```go
urlshortener
```

最好的做法是安装二进制文件，因为它可以在整个系统中使用。但对于较小的程序，我们可以通过访问程序的目录来运行`main.go`。

现在它运行 HTTP 服务器并开始收集 URL 缩短服务的请求。打开控制台并输入以下 CURL 命令：

```go
curl -X POST \
 http://localhost:8000/v1/short \
 -H 'cache-control: no-cache' \
 -H 'content-type: application/json' \
 -d '{
 "url": "https://www.forbes.com/forbes/welcome/?toURL=https://www.forbes.com/sites/karstenstrauss/2017/04/20/the-highest-paying-jobs-in-tech-in-2017/&refURL=https://www.google.co.in/&referrer=https://www.google.co.in/"
}'
```

它返回缩短的字符串：

```go
{
  "encoded_string": "1"
}
```

编码的字符串只是`"1"`。Base62 算法从`1`开始分配更短的字符串，直到组合字母数字。现在，如果我们需要检索原始 URL，我们可以执行`GET`请求：

```go
curl -X GET \
 http://localhost:8000/v1/short/1 \
 -H 'cache-control: no-cache' \
```

它返回以下 JSON：

```go
{   
"url":"https://www.forbes.com/forbes/welcome/?toURL=https://www.forbes.com/sites/karstenstrauss/2017/04/20/the-highest-paying-jobs-in-tech-in-2017/\u0026refURL=https://www.google.co.in/\u0026referrer=https://www.google.co.in/"}
```

因此，服务可以使用此结果将用户重定向到原始 URL（站点）。在这里，生成的字符串不取决于 URL 的长度，因为只有数据库 ID 是编码的标准。

在 PostgreSQL 中需要向`INSERT` SQL 命令添加`RETURNING`关键字以获取最后插入的数据库 ID。这在 MySQL 或 SQLite3 的`INSERT INTO web_url( ) VALUES($1) RETURNING id, record.URL`中并非如此。这个 DB 查询返回最后插入记录的 ID。如果我们去掉`RETURNING`关键字，查询将返回空。

# 在 PostgreSQL 中探索 JSON 存储

**PostgreSQL >9.2**有一个突出的功能 9.2" dbid="254735"叫做 JSON 存储。PostgreSQL 引入了一种新的数据类型来存储 JSON 数据。PostgreSQL 允许用户插入一个`jsonb`字段类型，它保存 JSON 字符串。它在对结构更加灵活的真实世界数据进行建模时非常有用。PostgreSQL 通过允许我们存储 JSON 字符串以及关系类型来发挥了最佳的作用。

在本节中，我们将尝试实现我们在前几章中为电子商务网站定义的一些 JSON 模型。但在这里，我们将使用 JSON 字段在 PostgreSQL 中存储和检索项目。对于访问 PostgreSQL 的 JSON 存储，普通的`pq`库非常繁琐。因此，为了更好地处理它，我们可以使用一个称为**GORM**的**对象关系映射器**（**ORM**）。

# GORM，Go 的强大 ORM

这个 ORM 具有`database/sql`包中可以执行的所有操作的 API。我们可以使用这个命令安装 GORM：

```go
go get -u github.com/jinzhu/gorm
```

有关此 ORM 的完整文档，请访问[`jinzhu.me/gorm/`](http://jinzhu.me/gorm/)。让我们编写一个实现用户和订单类型 JSON 模型的程序。用户可以下订单。我们将使用我们在上一章中定义的模型。我们可以在`$GOPATH/src/github.com/narenaryan`中创建一个名为`jsonstore`的新目录，并在其中为我们的模型创建一个新目录：

```go
mkdir jsonstore
mkdir jsonstore/models
touch jsonstore/models/models.go
```

现在，将`models.go`文件编辑为：

```go
package models

import (
  "github.com/jinzhu/gorm"
  _ "github.com/lib/pq"
)

type User struct {
  gorm.Model
  Orders []Order
  Data string `sql:"type:JSONB NOT NULL DEFAULT '{}'::JSONB" json:"-"`
}

type Order struct {
  gorm.Model
  User User
  Data string `sql:"type:JSONB NOT NULL DEFAULT '{}'::JSONB"`
}

// GORM creates tables with plural names. Use this to suppress it
func (User) TableName() string {
  return "user"
}

func (Order) TableName() string {
  return "order"
}

func InitDB() (*gorm.DB, error) {
  var err error
  db, err := gorm.Open("postgres", "postgres://naren:passme123@localhost/mydb?sslmode=disable")
  if err != nil {
    return nil, err
  } else {
    /*
    // The below AutoMigrate is equivalent to this
    if !db.HasTable("user") {
      db.CreateTable(&User{})
    }

    if !db.HasTable("order") {
      db.CreateTable(&Order{})        
    }
    */
    db.AutoMigrate(&User{}, &Order{})
    return db, nil
  }
}
```

这看起来与我们在本章前面定义的模型类似。在这里，对我们来说有很多新的东西。我们在 GORM 中创建的每个模型（表）都应该表示为一个结构。这就是我们创建了两个结构，`User`和`Order`的原因。第一行应该是`gorm.Model`。其他字段是表的字段。默认情况下，将创建一个递增的 ID。在之前的 URL 缩短器模型中，我们在操作之前手动检查表的存在。但在这里，有一个函数：

```go
db.AutoMigrate(&User{}, &Order{})
```

这个函数为作为参数传递的结构创建表。它确保如果表已经存在，它会跳过创建。如果你仔细观察，我们为这些结构添加了一个函数，`TableName`。默认情况下，GORM 创建的所有表名都是复数名（`User`的`users`被创建）。为了强制它创建给定的名称，我们需要覆盖该函数。另一个有趣的事情是，在结构中，我们使用了一个叫做`Data`的字段。它的类型是：

```go
`sql:"type:JSONB NOT NULL DEFAULT '{}'::JSONB" json:"-"`
```

是的，它是一个`jsonb`类型的字符串。我们现在将其类型添加为`string.PostgreSQL`，GORM 会处理它。然后我们将数据库连接返回给导入`models`包的人。 

# 实现电子商务 REST API

在开始之前，让我们设计 API 规范表，其中显示了各种 URL 终端的 REST API 签名。请参考以下表：

| **终端** | **方法** | **描述** |
| --- | --- | --- |
| `/v1/user/id` | `GET` | 使用 ID 获取用户 |
| `/v1/user` | `POST` | 创建新用户 |
| `/v1/user?first_name=NAME` | `GET` | 通过给定的名字获取所有用户 |
| `/v1/order/id` | `GET` | 获取具有给定 ID 的订单 |
| `/v1/order` | `POST` | 创建新订单 |

现在我们来到主程序；让我们向我们的`jsonstore`项目添加一个文件。在这个程序中，我们将尝试实现前三个终端。我们建议读者将剩下的两个终端的实现作为一个作业。看一下以下命令：

```go
touch jsonstore/main.go
```

程序结构遵循我们到目前为止看到的所有程序的相同风格。我们使用 Gorilla Mux 作为我们的 HTTP 路由器，并将数据库驱动程序导入到我们的程序中：

```go
package main

import (
  "encoding/json"
  "io/ioutil"
  "log"
  "net/http"
  "time"

  "github.com/gorilla/mux"
  "github.com/jinzhu/gorm"
    _ "github.com/lib/pq"
  "github.com/narenaryan/jsonstore/models"
)

// DB stores the database session imformation. Needs to be initialized once
type DBClient struct {
  db *gorm.DB
}

// UserResponse is the response to be send back for User
type UserResponse struct {
  User models.User `json:"user"`
  Data interface{} `json:"data"`
}

// GetUsersByFirstName fetches the original URL for the given encoded(short) string
func (driver *DBClient) GetUsersByFirstName(w http.ResponseWriter, r *http.Request) {
  var users []models.User
  name := r.FormValue("first_name")
  // Handle response details
  var query = "select * from \"user\" where data->>'first_name'=?"
  driver.db.Raw(query, name).Scan(&users)
  w.WriteHeader(http.StatusOK)
  w.Header().Set("Content-Type", "application/json")
  //responseMap := map[string]interface{}{"url": ""}
  respJSON, _ := json.Marshal(users)
  w.Write(respJSON)
}

// GetUser fetches the original URL for the given encoded(short) string
func (driver *DBClient) GetUser(w http.ResponseWriter, r *http.Request) {
  var user = models.User{}
  vars := mux.Vars(r)
  // Handle response details
  driver.db.First(&user, vars["id"])
  var userData interface{}
  // Unmarshal JSON string to interface
  json.Unmarshal([]byte(user.Data), &userData)
  var response = UserResponse{User: user, Data: userData}
  w.WriteHeader(http.StatusOK)
  w.Header().Set("Content-Type", "application/json")
  //responseMap := map[string]interface{}{"url": ""}
  respJSON, _ := json.Marshal(response)
  w.Write(respJSON)
}

// PostUser adds URL to DB and gives back shortened string
func (driver *DBClient) PostUser(w http.ResponseWriter, r *http.Request) {
  var user = models.User{}
  postBody, _ := ioutil.ReadAll(r.Body)
  user.Data = string(postBody)
  driver.db.Save(&user)
  responseMap := map[string]interface{}{"id": user.ID}
  var err string = ""
  if err != "" {
    w.Write([]byte("yes"))
  } else {
    w.Header().Set("Content-Type", "application/json")
    response, _ := json.Marshal(responseMap)
    w.Write(response)
  }
}

func main() {
  db, err := models.InitDB()
  if err != nil {
    panic(err)
  }
  dbclient := &DBClient{db: db}
  if err != nil {
    panic(err)
  }
  defer db.Close()
  // Create a new router
  r := mux.NewRouter()
  // Attach an elegant path with handler
  r.HandleFunc("/v1/user/{id:[a-zA-Z0-9]*}", dbclient.GetUser).Methods("GET")
  r.HandleFunc("/v1/user", dbclient.PostUser).Methods("POST")
  r.HandleFunc("/v1/user", dbclient.GetUsersByFirstName).Methods("GET")
  srv := &http.Server{
    Handler: r,
    Addr:    "127.0.0.1:8000",
    // Good practice: enforce timeouts for servers you create!
    WriteTimeout: 15 * time.Second,
    ReadTimeout:  15 * time.Second,
  }
  log.Fatal(srv.ListenAndServe())
}
```

这里有三个重要的方面：

+   我们用 GORM 驱动程序替换了传统的驱动程序

+   使用 GORM 函数进行 CRUD 操作

+   我们将 JSON 插入到 PostgreSQL 中，并在 JSON 字段中检索结果

让我们详细解释所有的元素。首先，我们导入了所有必要的包。有趣的是：

```go
  "github.com/jinzhu/gorm"
   _ "github.com/lib/pq"
  "github.com/narenaryan/jsonstore/models"
```

GORM 在内部在某种程度上使用了`database/sql`包。我们从我们在前面的代码中创建的包中导入了模型。接下来，我们创建了三个函数，实现了前三个 API 规范。它们是`GetUsersByFirstName`，`GetUser`和`PostUser`。每个函数都继承了数据库驱动程序，并作为`main`函数中 URL 端点的处理程序函数传递：

```go
 r.HandleFunc("/v1/user/{id:[a-zA-Z0-9]*}", dbclient.GetUser).Methods("GET")
 r.HandleFunc("/v1/user", dbclient.PostUser).Methods("POST")
 r.HandleFunc("/v1/user", dbclient.GetUsersByFirstName).Methods("GET")
```

现在，如果我们进入第一个函数，这很简单，这些语句会吸引我们的注意：

```go
driver.db.First(&user, vars["id"])
```

上述语句告诉数据库从具有给定第二参数`ID`的数据库中获取第一条记录。它将返回的数据填充到`user`结构中。我们在`GetUser`中使用`UserResponse`而不是`User`结构，因为`User`包含数据字段，它是一个字符串。但是，为了向客户端返回完整和正确的 JSON，我们需要将数据转换为一个适当的结构，然后进行编组：

```go
// UserResponse is the response to be send back for User
type UserResponse struct {
  User models.User `json:"user"`
  Data interface{} `json:"data"`
}
```

在这里，我们创建了一个可以容纳任何 JSON 数据的空接口。当我们使用驱动程序调用第一个函数时，用户结构具有一个数据字段，它是一个字符串。我们需要将该字符串转换为一个结构，然后将其与`UserResponse`中的其他详细信息一起发送。现在让我们看看这个过程。使用以下命令运行程序：

```go
go run jsonstore/main.go
```

并制作一些 CURL 命令来查看 API 响应：

创建用户：

```go
curl -X POST \
  http://localhost:8000/v1/user \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
     "username": "naren",
     "email_address": "narenarya@live.com",
     "first_name": "Naren",
     "last_name": "Arya"
}'
```

它返回了在数据库中插入的记录：

```go
{
  "id": 1
}
```

现在，如果我们`GET`插入记录的详细信息：

```go
curl -X GET http://localhost:8000/v1/user/1 
```

它返回有关用户的所有详细信息：

```go
{"user":{"ID":1,"CreatedAt":"2017-08-27T11:55:02.974371+05:30","UpdatedAt":"2017-08-27T11:55:02.974371+05:30","DeletedAt":null,"Orders":null},"data":{"email_address":"narenarya@live.com","first_name":"Naren","last_name":"Arya","username":"naren"}}
```

插入一条记录以检查名字 API：

```go
curl -X POST \
  http://localhost:8000/v1/user \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
     "username": "nareny",
     "email_address": "naren.yellavula@gmail.com",
     "first_name": "Naren",
     "last_name": "Yellavula"
}'
```

这插入了我们的第二条记录。让我们测试我们的第三个 API，`GetUsersByFirstName`：

```go
curl -X GET 'http://localhost:8000/v1/user?first_name=Naren' 
```

这将返回所有具有给定名字的用户：

```go
[{"ID":1,"CreatedAt":"2017-08-27T11:55:02.974371+05:30","UpdatedAt":"2017-08-27T11:55:02.974371+05:30","DeletedAt":null,"Orders":null},{"ID":2,"CreatedAt":"2017-08-27T11:59:41.84332+05:30","UpdatedAt":"2017-08-27T11:59:41.84332+05:30","DeletedAt":null,"Orders":null}]
```

这个项目的核心宗旨是展示如何从 PostgreSQL 中存储和检索 JSON。这里的特殊之处在于，我们查询了 JSON 字段，而不是`User`表中的普通字段。

记住，PostgreSQL 将其用户存储在一个名为 user 的表中。如果要创建一个新的用户表，请使用`"user"`（双引号）。即使在检索时也要使用双引号。否则，数据库将获取内部用户详细信息。

`SELECT * FROM "user"; // 正确的方式`

`SELECT * FROM user; // 错误的方式。它获取数据库用户`

这结束了我们对 PostgreSQL 的旅程。在 Postgres 中还有很多可以探索的地方。它通过允许我们在同一张表中存储关系型数据和 JSON 数据，将两者的优点发挥到了极致。

# 摘要

在本章中，我们通过安装 PostgreSQL 开始了我们的旅程。我们正式介绍了 PostgreSQL，并尝试看到所有可能的 CRUD 操作的 SQL 查询。然后我们看到了如何在 PostgreSQL 中添加用户和数据库。然后我们安装并解释了`pq`，这是 Go 语言的 Postgres 驱动程序。我们解释了驱动程序 API 如何执行原始的 SQL 查询。

然后是 URL 缩短服务的实现部分；该 REST 服务接受原始 URL 并返回缩短的字符串。它还接受缩短的 URL 并返回原始 URL。我们编写了一个示例程序来说明支持我们服务的 Base62 算法。我们随后在我们的服务中利用了这个算法，并创建了一个 REST API。

GORM 是 Go 语言中众所周知的对象关系映射器。使用 ORM，可以轻松管理数据库操作。GORM 提供了一些有用的函数，比如`AutoMigrate`（如果不存在则创建表），用于在传统的`database/sql`驱动程序上编写直观的 Go 代码。

PostgreSQL 还允许在 9.2 版本之后存储 JSON（称为 JSON 存储）。它允许开发人员以 JSON 格式获得关系数据库的好处。我们可以在 JSON 字段上创建索引，对 JSON 字段进行查询等。我们使用 GORM 为我们在前几章中定义的电子商务模型实现了 REST API。PostgreSQL 是一个成熟的、开源的关系数据库，可以满足我们的企业需求。Go 语言的驱动程序支持非常出色，包括`pq`和`gorm`。
