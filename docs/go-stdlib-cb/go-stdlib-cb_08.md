# 使用数据库

本章包含以下配方：

+   连接数据库

+   验证连接

+   执行语句

+   使用预处理语句进行操作

+   取消挂起的查询

+   读取查询结果元数据

+   从查询结果中检索数据

+   将查询结果解析为映射

+   处理事务

+   执行存储过程和函数

# 介绍

每个数据库服务器都有自己的特点，而且协议也不同。自然地，语言库内部与数据库通信必须定制以适用于特定协议。

Go 标准库提供了用于与数据库服务器通信和操作的统一 API。此 API 位于`sql`包中。要使用特定的数据库服务器，必须导入驱动程序。此驱动程序需要符合`sql`包的规范。这样，您将能够受益于统一的方法。在本章中，我们将描述数据库操作的基础知识、事务处理以及如何使用存储过程。请注意，我们将在 PostgreSQL 数据库上说明该方法，但这些方法适用于大多数其他数据库。

# 连接数据库

与数据库工作的关键部分是与数据库本身的连接。Go 标准包仅涵盖了与数据库交互的抽象，必须使用第三方驱动程序。

在本配方中，我们将展示如何连接到 PostgreSQL 数据库。但是，这种方法适用于所有其他驱动程序实现了标准 API 的数据库。

# 准备就绪

通过在终端中调用`go version`命令验证 Go 是否已正确安装。如果命令失败，请执行以下操作：

+   通过`go get -u github.com/lib/pq`获取 PostgreSQL 驱动程序

+   安装 PostgreSQL 数据库服务器（可选择使用 Docker 镜像而不是安装到主机系统）

+   我们将使用默认用户`postgres`和密码`postgres`

+   创建名为`example`的数据库

# 如何做...

1.  打开控制台并创建文件夹`chapter08/recipe01`。

1.  导航到目录。

1.  使用以下内容创建`connect.go`文件：

```go
       package main

       import (
         "database/sql"
         "fmt"

         _ "github.com/lib/pq"
       )

       func main() {
         connStr := "postgres://postgres:postgres@
                     localhost:5432/example?sslmode=disable"
         db, err := sql.Open("postgres", connStr)
         if err != nil {
           panic(err)
         }
         defer db.Close()
         err = db.Ping()
         if err != nil {
           panic(err)
         }
         fmt.Println("Ping OK")
       }
```

1.  通过`go run connect.go`执行代码。

1.  查看输出：

![](img/dc43c34f-78b8-4cd5-82e5-6541753840dc.png)

# 工作原理...

标准库包`database/sql`提供了`Open`函数，用于使用驱动程序名称和连接详细信息（在本例中为连接 URL）初始化与数据库的连接。请注意，`Open`函数不会立即创建连接，可能只会验证传递给函数的参数。

可以通过返回的`DB`结构指针中可用的`Ping`方法验证与数据库的连接。

驱动程序本身在`driver`包的`init`函数中初始化。驱动程序通过`sql`包的`Register`函数向驱动程序名称注册自身。`github.com/lib/pq`驱动程序将自身注册为`postgres`。

# 验证连接

驱动程序实现中的数据库连接可能被池化，并且可能从池中拉出的连接已经断开。本配方将展示如何验证连接是否仍然有效。

# 准备就绪

通过在终端中调用`go version`命令验证 Go 是否已正确安装。如果命令失败，请按照本章第一个配方中的*准备就绪*部分进行操作。

# 如何做...

1.  打开控制台并创建文件夹`chapter08/recipe02`。

1.  导航到目录。

1.  使用以下内容创建`verify.go`文件：

```go
        package main

        import (
          "context"
          "database/sql"
          "fmt"
          "time"

          _ "github.com/lib/pq"
        )

        func main() {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          defer db.Close()
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          fmt.Println("Ping OK.")
          ctx, _ := context.WithTimeout(context.Background(),
                                        time.Nanosecond)
          err = db.PingContext(ctx)
          if err != nil {
            fmt.Println("Error: " + err.Error())
          }

          // Verify the connection is
          conn, err := db.Conn(context.Background())
          if err != nil {
            panic(err)
          }
          defer conn.Close()
          err = conn.PingContext(context.Background())
          if err != nil {
            panic(err)
          }
          fmt.Println("Connection Ping OK.")

        }
```

1.  通过`go run verify.go`执行代码。

1.  查看输出：

![](img/9d5ba3af-08c2-4c54-93bd-25541dc94e8e.png)

# 工作原理...

如前一篇中提到的*连接数据库*，`Open`函数可能只是验证连接细节，但不一定立即连接数据库。实际连接到数据库通常是延迟加载的，并且是通过对数据库的第一次语句执行创建的。

`DB`结构的指针提供了`Ping`方法，通常对数据库进行幂等调用。`Ping`方法的变体是`PingContext`，它只是添加了取消或超时数据库调用的能力。请注意，如果`Ping`函数失败，连接将从数据库池中移除。

`DB`结构的指针还提供了`Conn`方法，用于从数据库池中检索连接。通过使用连接，您实际上保证使用相同的数据库会话。同样，`DB`结构的指针包含`PingContext`方法，`Conn`指针提供了`PingContext`方法来检查连接是否仍然活动。

# 执行语句

在以前的示例中，我们已经学习了如何连接和验证与数据库的连接。本示例将描述如何执行针对数据库的语句。

# 准备工作

通过在终端中调用`go version`命令来验证 Go 是否已正确安装。如果命令失败，请按照本章第一篇中的*准备工作*部分进行操作。

按照本章第一篇中的说明设置 PostgreSQL 服务器。

# 如何做...

1.  对您的示例数据库运行以下 SQL 脚本：

```go
        DROP TABLE IF EXISTS post;
        CREATE TABLE post (
          ID serial,
          TITLE varchar(40),
          CONTENT varchar(255),
          CONSTRAINT pk_post PRIMARY KEY(ID)
        );
        SELECT * FROM post;
```

1.  打开控制台并创建文件夹`chapter08/recipe03`。

1.  导航到目录。

1.  创建`statement.go`文件，内容如下：

```go
        package main

        import (
          "database/sql"
          "fmt"
          _ "github.com/lib/pq"
        )

        const sel = "SELECT * FROM post;"
        const trunc = "TRUNCATE TABLE post;"
        const ins = "INSERT INTO post(ID,TITLE,CONTENT)
                     VALUES (1,'Title 1','Content 1'),
                     (2,'Title 2','Content 2') "

        func main() {
          db := createConnection()
          defer db.Close()

          _, err := db.Exec(trunc)
          if err != nil {
            panic(err)
          }
          fmt.Println("Table truncated.")
          r, err := db.Exec(ins)
          if err != nil {
            panic(err)
          }
          affected, err := r.RowsAffected()
          if err != nil {
            panic(err)
          }
          fmt.Printf("Inserted rows count: %d\n",
                     affected)

          rs, err := db.Query(sel)
          if err != nil {
            panic(err)
          }
          count := 0
          for rs.Next() {
            count++
          }
          fmt.Printf("Total of %d was selected.\n", count)
        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run statement.go`执行代码。

1.  查看输出：

![](img/17b952d6-d23c-442a-bfb1-cbb262c3e22b.png)

# 工作原理...

通常，我们可以执行两种类型的语句。对于第一种类型的语句，我们不期望任何行作为结果，最终我们得到的是没有输出或者只是受影响的行数。这种类型的语句通过`DB`结构指针上的`Exec`方法执行。在前面的示例代码中，我们有`TRUNCATE`和`INSERT`语句。但是这种方式也可以执行 DDL 和 DCL 语句。

有四种主要的语句类别：

+   **DDL**（**数据定义语言**）：此语言允许您创建和修改数据库模式

+   **DML**（**数据建模语言**）：此语言帮助您修改数据

+   **DCL**（**数据控制语言**）：此语言定义了对对象的访问控制

+   **TCL**（**事务控制语言**）：此语言控制事务。

第二种类型是我们期望以行的形式得到结果的语句；这些通常被称为查询。这种类型的语句通常通过`Query`或`QueryContext`方法执行。

# 使用准备好的语句

准备好的语句带来了安全性、效率和便利性。当然，可以使用它们与 Go 标准库一起使用；本示例将展示如何使用。

# 准备工作

通过在终端中调用`go version`命令来验证 Go 是否已正确安装。如果命令失败，请按照本章第一篇中的*准备工作*部分进行操作。

按照本章第一篇中的说明设置 PostgreSQL 服务器。

# 如何做...

1.  对您的示例数据库运行以下 SQL 脚本：

```go
        DROP TABLE IF EXISTS post;
        CREATE TABLE post (
          ID serial,
          TITLE varchar(40),
          CONTENT varchar(255),
          CONSTRAINT pk_post PRIMARY KEY(ID)
        );
        SELECT * FROM post;
```

1.  打开控制台并创建文件夹`chapter08/recipe04`。

1.  导航到目录。

1.  创建`prepared.go`文件，内容如下：

```go
        package main

        import (
          "database/sql"
          "fmt"
          _ "github.com/lib/pq"
        )

        const trunc = "TRUNCATE TABLE post;"
        const ins = "INSERT INTO post(ID,TITLE,CONTENT)
                     VALUES ($1,$2,$3)"

        var testTable = []struct {
          ID int
          Title string
          Content string
        }{
          {1, "Title One", "Content of title one"},
          {2, "Title Two", "Content of title two"},
          {3, "Title Three", "Content of title three"},
        }

        func main() {
          db := createConnection()
          defer db.Close()

          // Truncate table
          _, err := db.Exec(trunc)
          if err != nil {
            panic(err)
          }

          stm, err := db.Prepare(ins)
          if err != nil {
            panic(err)
          }

          inserted := int64(0)
          for _, val := range testTable {
            fmt.Printf("Inserting record ID: %d\n", val.ID)
            // Execute the prepared statement
            r, err := stm.Exec(val.ID, val.Title, val.Content)
            if err != nil {
              fmt.Printf("Cannot insert record ID : %d\n",
                         val.ID)
            }
            if affected, err := r.RowsAffected(); err == nil {
              inserted = inserted + affected
            }
          }

          fmt.Printf("Result: Inserted %d rows.\n", inserted)

        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run prepared.go`执行代码。

1.  查看输出：

![](img/f93601d9-2fb8-4db8-a3dd-e263f03b0cf0.png)

# 工作原理...

要创建准备好的语句，需要调用指向`DB`结构的`Prepare`方法。之后，使用给定的参数调用`Stmt`指针上的`Exec`或`Query`方法。

准备好的语句是在`DB`指针的范围内创建的，但是在连接池中的特定连接上。语句记住了使用过的连接，并且在调用时尝试使用相同的连接。如果连接忙或已关闭，则重新创建准备好的语句并在新连接上调用语句。

如果在打开的事务`*Tx`中使用准备好的语句，则情况会发生变化，在这种情况下，准备好的语句绑定到与事务相关的一个连接。

请注意，事务中准备的语句不能与 DB 指针一起使用，反之亦然。

通常，准备好的语句的工作方式是在数据库端创建语句。数据库返回准备好的语句的标识符。准备好的语句在以下调用期间执行，并且只提供语句的参数。

# 取消挂起的查询

在某些情况下，您需要取消长时间运行的语句以限制资源的消耗，或者仅当结果不相关或语句运行时间过长时。自 Go 1.8 以来，取消查询是可能的。本配方解释了如何使用此功能。

# 准备工作

通过在终端中调用`go version`命令验证 Go 是否已正确安装。如果命令失败，请按照本章第一个配方中的*准备工作*部分进行操作。

按照本章第一个配方中提到的方式设置 PostgreSQL 服务器。

# 操作步骤...

1.  对您的示例数据库运行以下 SQL 脚本：

```go
        DROP TABLE IF EXISTS post;
        CREATE TABLE post (
          ID serial,
          TITLE varchar(40),
          CONTENT varchar(255),
          CONSTRAINT pk_post PRIMARY KEY(ID)
        );
        SELECT * FROM post;
        INSERT INTO post(ID,TITLE,CONTENT) VALUES
                        (1,'Title One','Content One'),
                        (2,'Title Two','Content Two');
```

1.  打开控制台并创建文件夹`chapter08/recipe05`。

1.  导航到目录。

1.  使用以下内容创建`cancelable.go`文件：

```go
        package main

        import (
          "context"
          "database/sql"
          "fmt"
          "time"
          _ "github.com/lib/pq"
        )

        const sel = "SELECT * FROM post p CROSS JOIN
           (SELECT 1 FROM generate_series(1,1000000)) tbl"

        func main() {
          db := createConnection()
          defer db.Close()

          ctx, canc := context.WithTimeout(context.Background(),
                                           20*time.Microsecond)
          rows, err := db.QueryContext(ctx, sel)
          canc() //cancel the query
          if err != nil {
            fmt.Println(err)
            return
          }
          defer rows.Close()
          count := 0
          for rows.Next() {
            if rows.Err() != nil {
              fmt.Println(rows.Err())
              continue
            }
            count++
          }

          fmt.Printf("%d rows returned\n", count)

        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run cancelable.go`执行代码。

1.  查看输出：

![](img/dd8da9ab-b8ad-4b5b-bf43-3f0d766910ea.png)

# 工作原理...

`database/sql`包提供了取消挂起语句的可能性。`DB`结构指针的所有名为`XXXContext`的方法都会消耗上下文，并且可以取消挂起的语句。

只有在驱动程序支持`Context`变体时才能取消语句。如果不支持，将执行不带`Context`的变体。

使用`Context`变体和`context.WithTimeout`，您可以创建语句调用的超时。

请注意，示例代码执行以错误`pq: canceling statement due to user request`结束，这与调用查询后立即调用的`CancelFunc`相对应。

# 读取查询结果元数据

除了数据本身，查询结果还包含与结果集相关的元数据。这包含有关列名、类型和数据的其他信息。本配方将解释如何检索数据。

# 准备工作

通过在终端中调用`go version`命令验证 Go 是否已正确安装。如果命令失败，请按照本章第一个配方中的*准备工作*部分进行操作。

按照本章第一个配方中提到的方式设置 PostgreSQL 服务器。

# 操作步骤...

1.  对您的示例数据库运行以下 SQL 脚本：

```go
        DROP TABLE IF EXISTS post;
        CREATE TABLE post (
          ID serial,
          TITLE varchar(40),
          CONTENT varchar(255),
          CONSTRAINT pk_post PRIMARY KEY(ID)
        );
        SELECT * FROM post;
        INSERT INTO post(ID,TITLE,CONTENT) VALUES
                        (1,'Title One','Content One'),
                        (2,'Title Two','Content Two');

```

1.  打开控制台并创建文件夹`chapter08/recipe06`。

1.  导航到目录。

1.  使用以下内容创建`metadata.go`文件：

```go
        package main

        import (
          "database/sql"
          "fmt"
          _ "github.com/lib/pq"
        )

        const sel = "SELECT * FROM post p"

        func main() {

          db := createConnection()
          defer db.Close()

          rs, err := db.Query(sel)
          if err != nil {
            panic(err)
          }
          defer rs.Close()
          columns, err := rs.Columns()
          if err != nil {
            panic(err)
          }
          fmt.Printf("Selected columns: %v\n", columns)

          colTypes, err := rs.ColumnTypes()
          if err != nil {
            panic(err)
          }
          for _, col := range colTypes {
            fmt.Println()
            fmt.Printf("%+v\n", col)
          }

        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run metadata.go`执行代码。

1.  查看输出：

![](img/48545a27-4614-44a1-a20b-5abdf0ae7f82.png)

# 工作原理...

`DB`结构指针的`Query`和`QueryContext`方法会导致`Rows`结构指针。`Rows`指针提供`Columns`和`ColumnTypes`方法，其中包含有关返回结果集结构的信息。

`Columns`方法返回带有列名的字符串切片。

`ColumnTypes`方法返回`ColumnType`指针的切片，其中包含有关返回结果集的更丰富信息。上述代码打印出了`ColumnType`指针公开的详细信息。

# 从查询结果中检索数据

在与数据库交互时，基本部分是通过执行查询来提取数据。本配方将说明使用标准库`database/sql`包时如何执行此操作。

# 准备工作

验证 Go 是否已正确安装，通过在终端中调用`go version`命令。如果命令失败，请按照本章第一个配方中的*准备工作*部分进行操作。

按照本章第一个配方中的说明设置 PostgreSQL 服务器。

# 操作步骤...

1.  对样本数据库运行以下 SQL 脚本：

```go
       DROP TABLE IF EXISTS post;
       CREATE TABLE post (
         ID serial,
         TITLE varchar(40),
         CONTENT varchar(255),
         CONSTRAINT pk_post PRIMARY KEY(ID)
       );
       SELECT * FROM post;
       INSERT INTO post(ID,TITLE,CONTENT) VALUES
                       (1,'Title One','Content One'),
                       (2,NULL,'Content Two');
```

1.  打开控制台并创建文件夹`chapter08/recipe07`。

1.  导航到目录。

1.  创建`data.go`文件，内容如下：

```go
        package main

        import (
          "database/sql"
          "fmt"
          _ "github.com/lib/pq"
        )

        const sel = `SELECT title,content FROM post;
        SELECT 1234 NUM; `

        const selOne = "SELECT title,content FROM post
                        WHERE ID = $1;"

        type Post struct {
          Name sql.NullString
          Text sql.NullString
        }

        func main() {
          db := createConnection()
          defer db.Close()

          rs, err := db.Query(sel)
          if err != nil {
            panic(err)
          }
          defer rs.Close()

          posts := []Post{}
          for rs.Next() {
            if rs.Err() != nil {
              panic(rs.Err())
            }
            p := Post{}
            if err := rs.Scan(&p.Name, &p.Text); err != nil {
              panic(err)
            }
            posts = append(posts, p)
          }

          var num int
          if rs.NextResultSet() {
            for rs.Next() {
              if rs.Err() != nil {
                panic(rs.Err())
              }
              rs.Scan(&num)
            }
          }

          fmt.Printf("Retrieved posts: %+v\n", posts)
          fmt.Printf("Retrieved number: %d\n", num)

          row := db.QueryRow(selOne, 100)
          or := Post{}
          if err := row.Scan(&or.Name, &or.Text); err != nil {
            fmt.Printf("Error: %s\n", err.Error())
            return
          }

          fmt.Printf("Retrieved one post: %+v\n", or)

        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run data.go`执行代码。

1.  查看输出：

![](img/2ac9cc08-6adb-48da-b0a6-a89657d80791.png)

# 工作原理...

来自指向`DB`结构的`Query`方法的`Rows`指针提供了从结果集中读取和提取数据的方法。

请注意，首先应调用`Next`方法将光标移动到下一个结果行。`Next`方法如果有其他行则返回`true`，否则返回`false`。

在通过`Next`获取新行后，可以调用`Scan`方法将数据提取到变量中。变量的数量必须与`SELECT`中的列数匹配，否则`Scan`方法无法提取数据。

代码的重要部分是，在每次调用`Next`方法后，应调用`Err`方法来查找在读取下一行时是否出现错误。

上述示例故意对第二条记录使用了`NULL`值。`NULL`数据库值无法提取到不可为空类型，例如`string`，在这种情况下，必须使用`NullString`类型。

为了完整起见，示例代码涵盖了`QueryRow`方法，它与`Query`方法略有不同。这个方法返回指向`Row`结构的指针，该结构仅提供`Scan`方法。请注意，只有在调用`Scan`方法之后才能检测到没有行的情况。

# 将查询结果解析为映射

有时查询结果或表的结构不清晰，需要将结果提取到某种灵活的结构中。这就引出了这个配方，其中将介绍将值提取到与列名映射的灵活结构中。

# 准备工作

验证 Go 是否已正确安装，通过在终端中调用`go version`命令。如果命令失败，请按照本章第一个配方中的*准备工作*部分进行操作。

按照本章第一个配方中的说明设置 PostgreSQL 服务器。

# 操作步骤...

1.  对样本数据库运行以下 SQL 脚本：

```go
        DROP TABLE IF EXISTS post;
        CREATE TABLE post (
          ID serial,
          TITLE varchar(40),
          CONTENT varchar(255),
          CONSTRAINT pk_post PRIMARY KEY(ID)
        );
        SELECT * FROM post;
        INSERT INTO post(ID,TITLE,CONTENT) VALUES 
                        (1,NULL,'Content One'),
                        (2,'Title Two','Content Two');
```

1.  打开控制台并创建文件夹`chapter08/recipe08`。

1.  导航到目录。

1.  创建`querymap.go`文件，内容如下：

```go
        package main

        import (
          "database/sql"
          "fmt"
          _ "github.com/lib/pq"
        )

        const selOne = "SELECT id,title,content FROM post
                        WHERE ID = $1;"

        func main() {
          db := createConnection()
          defer db.Close()

          rows, err := db.Query(selOne, 1)
          if err != nil {
            panic(err)
          }
          cols, _ := rows.Columns()
          for rows.Next() {
            m := parseWithRawBytes(rows, cols)
            fmt.Println(m)
            m = parseToMap(rows, cols)
            fmt.Println(m)
          }
        }

        func parseWithRawBytes(rows *sql.Rows, cols []string)
                               map[string]interface{} {
          vals := make([]sql.RawBytes, len(cols))
          scanArgs := make([]interface{}, len(vals))
          for i := range vals {
            scanArgs[i] = &vals[i]
          }
          if err := rows.Scan(scanArgs...); err != nil {
            panic(err)
          }
          m := make(map[string]interface{})
          for i, col := range vals {
            if col == nil {
              m[cols[i]] = nil
            } else {
              m[cols[i]] = string(col)
            }
          }
          return m
        }

        func parseToMap(rows *sql.Rows, cols []string)
                        map[string]interface{} {
          values := make([]interface{}, len(cols))
          pointers := make([]interface{}, len(cols))
          for i := range values {
            pointers[i] = &values[i]
          }

          if err := rows.Scan(pointers...); err != nil {
            panic(err)
          }

          m := make(map[string]interface{})
          for i, colName := range cols {
            if values[i] == nil {
              m[colName] = nil
            } else {
              m[colName] = values[i]
            }
          }
          return m
        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run querymap.go`执行代码。

1.  查看输出：

![](img/d69cc84f-33e3-48b3-9d36-1a85b6f52253.png)

# 工作原理...

请注意，上述代码表示了两种方法。`parseWithRawBytes`函数使用了首选方法，但它高度依赖于驱动程序的实现。它的工作方式是创建与结果中列数相同长度的`RawBytes`切片。因为`Scan`函数需要值的指针，所以我们需要创建指向`RawBytes`切片（字节切片的切片）的指针切片，然后将其传递给`Scan`函数。

提取成功后，我们只需重新映射值。在示例代码中，我们将其转换为`string`，因为如果`RawBytes`是目标，驱动程序使用`string`类型来存储值。请注意，存储值的形式取决于驱动程序的实现。

第二种方法`parseToMap`在第一种方法不起作用的情况下是可用的。它几乎使用相同的方法，但值的切片被定义为空接口的切片。这种方法依赖于驱动程序。驱动程序应确定要分配给值指针的默认类型。

# 处理事务

事务控制是在处理数据库时需要牢记的常见事情。本配方将向您展示如何使用`sql`包处理事务。

# 准备工作

通过在终端中调用`go version`命令来验证 Go 是否已正确安装。如果命令失败，请按照本章第一个配方中的*准备工作*部分进行操作。

设置 PostgreSQL 服务器，如本章第一个配方中所述。

# 如何做...

1.  对您的示例数据库运行以下 SQL 脚本：

```go
        DROP TABLE IF EXISTS post;
        CREATE TABLE post (
          ID serial,
          TITLE varchar(40),
          CONTENT varchar(255),
          CONSTRAINT pk_post PRIMARY KEY(ID)
        );
        SELECT * FROM post;
        INSERT INTO post(ID,TITLE,CONTENT) VALUES
                        (1,'Title One','Content One'),
                        (2,NULL,'Content Two');
```

1.  打开控制台并创建文件夹`chapter08/recipe09`。

1.  导航到目录。

1.  创建`transaction.go`文件，内容如下：

```go
        package main

        import (
          "database/sql"
          "fmt"
          _ "github.com/lib/pq"
        )

        const selOne = "SELECT id,title,content FROM post
                        WHERE ID = $1;"
        const insert = "INSERT INTO post(ID,TITLE,CONTENT)
                VALUES (4,'Transaction Title','Transaction Content');"

        type Post struct {
          ID int
          Title string
          Content string
        }

        func main() {
          db := createConnection()
          defer db.Close()

          tx, err := db.Begin()
          if err != nil {
            panic(err)
          }
          _, err = tx.Exec(insert)
          if err != nil {
            panic(err)
          }
          p := Post{}
          // Query in other session/transaction
          if err := db.QueryRow(selOne, 4).Scan(&p.ID,
                &p.Title, &p.Content); err != nil {
            fmt.Println("Got error for db.Query:" + err.Error())
          }
          fmt.Println(p)
          // Query within transaction
          if err := tx.QueryRow(selOne, 4).Scan(&p.ID,
                 &p.Title, &p.Content); err != nil {
            fmt.Println("Got error for db.Query:" + err.Error())
          }
          fmt.Println(p)
          // After commit or rollback the
          // transaction need to recreated.
          tx.Rollback()

        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@
                      localhost:5432/example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run transaction.go`执行代码。

1.  看输出：

![](img/237bdb28-3ba6-4a4a-8c08-f1212e8f43fc.png)

# 工作原理...

正如前面的代码所示，事务处理非常简单。`DB`结构指针的`Begin`方法创建具有默认隔离级别的事务（取决于驱动程序）。事务本质上保留在单个连接上，并由返回的`Tx`结构指针表示。

指针`Tx`实现了`DB`结构指针可用的所有方法；唯一的例外是所有操作都在事务中完成（如果数据库能够在事务中处理语句）。通过在`Tx`结构指针上调用`Rollback`或`Commit`方法结束事务。在此调用之后，事务结束，其他操作将以错误`ErrTxDone`结束。

`DB`结构指针上还有一个有用的方法叫做`BeginTx`，它创建了事务`Tx`结构指针，同时也增强了给定的上下文。如果上下文被取消，事务将被回滚（进一步的`Commit`调用将导致错误）。`BeginTx`还消耗了`TxOptions`指针，这是可选的，可以定义隔离级别。

# 执行存储过程和函数

处理存储过程和函数总是比通常的语句更复杂，特别是如果过程包含自定义类型。标准库提供了处理这些的 API，但存储过程调用的支持程度取决于驱动程序的实现。本配方将展示一个非常简单的函数/过程调用。

# 准备工作

通过在终端中调用`go version`命令来验证 Go 是否已正确安装。如果命令失败，请按照本章第一个配方中的*准备工作*部分进行操作。

设置 PostgreSQL 服务器，如本章第一个配方中所述。

# 如何做...

1.  对您的示例数据库运行以下 SQL 脚本：

```go
        CREATE OR REPLACE FUNCTION format_name
        (firstname Text,lastname Text,age INT) RETURNS 
        VARCHAR AS $$
        BEGIN
          RETURN trim(firstname) ||' '||trim(lastname) ||' ('||age||')';
        END;
        $$ LANGUAGE plpgsql;
```

1.  打开控制台并创建文件夹`chapter08/recipe10`。

1.  导航到目录。

1.  创建`procedure.go`文件，内容如下：

```go
        package main

        import (
          "database/sql"
          "fmt"

          _ "github.com/go-sql-driver/mysql"
          _ "github.com/lib/pq"
        )

        const call = "select * from format_name($1,$2,$3)"

        const callMySQL = "CALL simpleproc(?)"

        type Result struct {
          Name string
          Category int
        }

        func main() {
          db := createConnection()
          defer db.Close()
          r := Result{}

          if err := db.QueryRow(call, "John", "Doe",
                    32).Scan(&r.Name); err != nil {
            panic(err)
          }
          fmt.Printf("Result is: %+v\n", r)
        }

        func createConnection() *sql.DB {
          connStr := "postgres://postgres:postgres@localhost:5432
                      /example?sslmode=disable"
          db, err := sql.Open("postgres", connStr)
          if err != nil {
            panic(err)
          }
          err = db.Ping()
          if err != nil {
            panic(err)
          }
          return db
        }
```

1.  通过`go run procedure.go`执行代码。

1.  看输出：

![](img/25194205-e33a-4b71-b9d9-e9653189ff44.png)

# 工作原理...

存储过程的调用高度依赖于驱动程序和数据库。请注意，在 PostgreSQL 数据库上检索结果与查询表非常相似。调用`DB`结构指针的`Query`或`QueryRow`方法，可以解析出结果行或行指针以获取值。

如果需要调用存储过程，MySQL 驱动程序将使用`CALL`语句。

几乎所有驱动程序的一般问题都是存储过程的`OUTPUT`参数。Go 1.9 增加了对这些参数的支持，但常用数据库的大多数驱动程序尚未实现这一功能。因此，解决方案可能是使用具有非标准 API 的驱动程序。

`OUTPUT`参数应该工作的方式是，存储过程调用将使用`database/sql`包中`Named`函数的`NamedArg`参数类型。`NamedArg`结构体的`Value`字段应该是`Out`类型，其中包含`Dest`字段，用于存放`OUTPUT`参数的实际值。
