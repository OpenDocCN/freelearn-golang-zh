# 第六章：关于数据库和存储的一切

Go 应用程序经常需要使用长期存储。这通常以关系和非关系数据库的形式存在，以及键值存储等。在处理这些存储应用程序时，将操作封装在接口中是有帮助的。本章的配方将检查各种存储接口，考虑诸如连接池等并行访问的问题，并查看集成新库的一般提示，这在使用新的存储技术时经常发生。

在本章中，将涵盖以下配方：

+   使用 database/sql 包与 MySQL

+   执行数据库事务接口

+   连接池、速率限制和 SQL 的超时

+   使用 Redis

+   使用 MongoDB 的 NoSQL

+   创建数据可移植性的存储接口

# 使用 database/sql 包与 MySQL

关系数据库是一些最为人熟知和常见的数据库选项。MySQL 和 PostgreSQL 是两种最流行的开源关系数据库。这个配方将演示`database/sql`包，它提供了一些关系数据库的钩子，并自动处理连接池和连接持续时间，并提供了一些基本的数据库操作。

这个配方将使用 MySQL 数据库建立连接，插入一些简单的数据并查询它。它将在使用后通过删除表来清理数据库。

# 准备工作

根据以下步骤配置你的环境：

1.  在你的操作系统上下载并安装 Go 1.12.6 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  打开一个终端或控制台应用程序，创建一个项目目录，比如`~/projects/go-programming-cookbook`，并导航到该目录。所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，并选择从该目录工作，而不是手动输入示例。

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

1.  使用[`dev.mysql.com/doc/mysql-getting-started/en/`](https://dev.mysql.com/doc/mysql-getting-started/en/)安装和配置 MySQL。

1.  运行`export MYSQLUSERNAME=<your mysql username>`命令。

1.  运行`export MYSQLPASSWORD=<your mysql password>`命令。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter6/database`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/database 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/database    
```

1.  从`~/projects/go-programming-cookbook-original/chapter6/database`复制测试，或者利用这个练习编写一些自己的代码！

1.  创建一个名为`config.go`的文件，内容如下：

```go
        package database

        import (
            "database/sql"
            "fmt"
            "os"
            "time"

            _ "github.com/go-sql-driver/mysql" //we import supported 
            libraries for database/sql
        )

        // Example hold the results of our queries
        type Example struct {
            Name string
            Created *time.Time
        }

        // Setup configures and returns our database
        // connection poold
        func Setup() (*sql.DB, error) {
            db, err := sql.Open("mysql", 
            fmt.Sprintf("%s:%s@/gocookbook? 
            parseTime=true", os.Getenv("MYSQLUSERNAME"), 
            os.Getenv("MYSQLPASSWORD")))
            if err != nil {
                return nil, err
            }
            return db, nil
        }
```

1.  创建一个名为`create.go`的文件，内容如下：

```go
        package database

        import (
            "database/sql"

            _ "github.com/go-sql-driver/mysql" //we import supported 
            libraries for database/sql
        )

        // Create makes a table called example
        // and populates it
        func Create(db *sql.DB) error {
            // create the database
            if _, err := db.Exec("CREATE TABLE example (name 
            VARCHAR(20), created DATETIME)"); err != nil {
                return err
            }

            if _, err := db.Exec(`INSERT INTO example (name, created) 
            values ("Aaron", NOW())`); err != nil {
                return err
            }

            return nil
        }
```

1.  创建一个名为`query.go`的文件，内容如下：

```go
        package database

        import (
            "database/sql"
            "fmt"

            _ "github.com/go-sql-driver/mysql" //we import supported 
            libraries for database/sql
        )

        // Query grabs a new connection
        // creates tables, and later drops them
        // and issues some queries
        func Query(db *sql.DB, name string) error {
            name := "Aaron"
            rows, err := db.Query("SELECT name, created FROM example 
            where name=?", name)
            if err != nil {
                return err
            }
            defer rows.Close()
            for rows.Next() {
                var e Example
                if err := rows.Scan(&e.Name, &e.Created); err != nil {
                    return err
                }
                fmt.Printf("Results:\n\tName: %s\n\tCreated: %v\n", 
                e.Name, e.Created)
            }
            return rows.Err()
        }
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package database

        // Exec replaces the Exec from the previous
        // recipe
        func Exec(db DB) error {

            // uncaught error on cleanup, but we always
            // want to cleanup
            defer db.Exec("DROP TABLE example")

            if err := Create(db); err != nil {
                return err
            }

            if err := Query(db, "Aaron"); err != nil {
                return err
            }
            return nil
        }
```

1.  创建并导航到`example`目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "PacktPublishing/Go-Programming-Cookbook-Second-Edition/
             go-cookbook/chapter6/database"
            _ "github.com/go-sql-driver/mysql" //we import supported 
            libraries for database/sql
        )

        func main() {
            db, err := database.Setup()
            if err != nil {
                panic(err)
            }

            if err := database.Exec(db); err != nil {
                panic(err)
            }
        }
```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

你应该看到以下输出：

```go
$ go run main.go
Results:
 Name: Aaron
 Created: 2017-02-16 19:02:36 +0000 UTC
```

1.  `go.mod`文件可能会被更新，顶层配方目录中现在应该存在`go.sum`文件。

1.  如果你复制或编写了自己的测试，返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

代码中的`_ "github.com/go-sql-driver/mysql"`行是将各种数据库连接器连接到`database/sql`包的方法。还有其他可以以类似方式导入的 MySQL 包，以获得类似的结果。如果你要连接到 PostgreSQL、SQLite 或其他实现了`database/sql`接口的数据库，命令也会类似。

一旦连接，该包将设置一个连接池，该连接池在*SQL 的连接池、速率限制和超时*配方中有所涵盖，您可以直接在连接上执行 SQL，也可以创建可以使用`commit`和`rollback`命令执行所有连接操作的事务对象。

当与数据库通信时，`mysql`包为 Go 时间对象提供了一些便利支持。这个配方还从`MYSQLUSERNAME`和`MYSQLPASSWORD`环境变量中检索用户名和密码。

# 执行数据库事务接口

在与数据库等服务的连接工作时，编写测试可能会很困难。这是因为在 Go 中很难在运行时模拟或鸭子类型化。虽然我建议在处理数据库时使用存储接口，但在这个接口内部模拟数据库事务接口仍然很有用。*为数据可移植性创建存储接口*配方将涵盖存储接口；这个配方将专注于包装数据库连接和事务对象的接口。

为了展示这样一个接口的使用，我们将重写前一个配方中的创建和查询文件以使用我们的接口。最终输出将是相同的，但创建和查询操作将都在一个事务中执行。

# 准备工作

参考*使用 database/sql 包与 MySQL*配方中的*准备工作*部分。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter6/dbinterface`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/dbinterface 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/dbinterface    
```

1.  从`~/projects/go-programming-cookbook-original/chapter6/dbinterface`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`transaction.go`的文件，内容如下：

```go
package dbinterface

import "database/sql"

// DB is an interface that is satisfied
// by an sql.DB or an sql.Transaction
type DB interface {
  Exec(query string, args ...interface{}) (sql.Result, error)
  Prepare(query string) (*sql.Stmt, error)
  Query(query string, args ...interface{}) (*sql.Rows, error)
  QueryRow(query string, args ...interface{}) *sql.Row
}

// Transaction can do anything a Query can do
// plus Commit, Rollback, or Stmt
type Transaction interface {
  DB
  Commit() error
  Rollback() error
}
```

1.  创建一个名为`create.go`的文件，内容如下：

```go
package dbinterface

import _ "github.com/go-sql-driver/mysql" //we import supported libraries for database/sql

// Create makes a table called example
// and populates it
func Create(db DB) error {
  // create the database
  if _, err := db.Exec("CREATE TABLE example (name VARCHAR(20), created DATETIME)"); err != nil {
    return err
  }

  if _, err := db.Exec(`INSERT INTO example (name, created) values ("Aaron", NOW())`); err != nil {
    return err
  }

  return nil
}
```

1.  创建一个名为`query.go`的文件，内容如下：

```go
package dbinterface

import (
  "fmt"

  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/database"
)

// Query grabs a new connection
// creates tables, and later drops them
// and issues some queries
func Query(db DB) error {
  name := "Aaron"
  rows, err := db.Query("SELECT name, created FROM example where name=?", name)
  if err != nil {
    return err
  }
  defer rows.Close()
  for rows.Next() {
    var e database.Example
    if err := rows.Scan(&e.Name, &e.Created); err != nil {
      return err
    }
    fmt.Printf("Results:\n\tName: %s\n\tCreated: %v\n", e.Name, 
                e.Created)
  }
  return rows.Err()
}
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
package dbinterface

// Exec replaces the Exec from the previous
// recipe
func Exec(db DB) error {

  // uncaught error on cleanup, but we always
  // want to cleanup
  defer db.Exec("DROP TABLE example")

  if err := Create(db); err != nil {
    return err
  }

  if err := Query(db); err != nil {
    return err
  }
  return nil
}
```

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/database"
  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/dbinterface"
  _ "github.com/go-sql-driver/mysql" //we import supported libraries for database/sql
)

func main() {
  db, err := database.Setup()
  if err != nil {
    panic(err)
  }

  tx, err := db.Begin()
  if err != nil {
    panic(err)
  }
  // this wont do anything if commit is successful
  defer tx.Rollback()

  if err := dbinterface.Exec(tx); err != nil {
    panic(err)
  }
  if err := tx.Commit(); err != nil {
    panic(err)
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
Results:
 Name: Aaron
 Created: 2017-02-16 20:00:00 +0000 UTC
```

1.  `go.mod`文件可能会被更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方的工作方式与前一个数据库配方*使用 database/sql 包与 MySQL*非常相似。这个配方执行了创建数据和查询数据的相同操作，但也演示了使用事务和创建通用数据库函数，这些函数可以与`sql.DB`连接和`sql.Transaction`对象一起使用。

以这种方式编写的代码允许我们重用执行数据库操作的函数，这些函数可以单独运行或在事务中运行。这样可以实现更多的代码重用，同时仍然将功能隔离到在数据库上操作的函数或方法中。例如，您可以为多个表格编写`Update(db DB)`函数，并将它们全部传递给一个共享的事务，以原子方式执行多个更新。这样也更容易模拟这些接口，正如您将在第九章中看到的，*测试 Go 代码*。

# SQL 的连接池、速率限制和超时

虽然`database/sql`包提供了连接池、速率限制和超时的支持，但通常需要调整默认值以更好地适应数据库配置。当您在微服务上进行水平扩展并且不希望保持太多活动连接到数据库时，这一点就变得很重要。

# 准备工作

参考*使用 database/sql 包与 MySQL*配方中的*准备工作*部分。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter6/pools`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/pools 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/pools    
```

1.  从`~/projects/go-programming-cookbook-original/chapter6/pools`复制测试，或者利用这个练习编写一些自己的代码！

1.  创建一个名为`pools.go`的文件，并包含以下内容：

```go
        package pools

        import (
            "database/sql"
            "fmt"
            "os"

            _ "github.com/go-sql-driver/mysql" //we import supported 
            libraries for database/sql
        )

        // Setup configures the db along with pools
        // number of connections and more
        func Setup() (*sql.DB, error) {
            db, err := sql.Open("mysql", 
            fmt.Sprintf("%s:%s@/gocookbook? 
            parseTime=true", os.Getenv("MYSQLUSERNAME"),         
            os.Getenv("MYSQLPASSWORD")))
            if err != nil {
                return nil, err
            }

            // there will only ever be 24 open connections
            db.SetMaxOpenConns(24)

            // MaxIdleConns can never be less than max open 
            // SetMaxOpenConns otherwise it'll default to that value
            db.SetMaxIdleConns(24)

            return db, nil
        }
```

1.  创建一个名为`timeout.go`的文件，并包含以下内容：

```go
package pools

import (
  "context"
  "time"
)

// ExecWithTimeout will timeout trying
// to get the current time
func ExecWithTimeout() error {
  db, err := Setup()
  if err != nil {
    return err
  }

  ctx := context.Background()

  // we want to timeout immediately
  ctx, cancel := context.WithDeadline(ctx, time.Now())

  // call cancel after we complete
  defer cancel()

  // our transaction is context aware
  _, err = db.BeginTx(ctx, nil)
  return err
}
```

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，并包含以下内容：

```go
        package main

        import "PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                go-cookbook/chapter6/pools"

        func main() {
            if err := pools.ExecWithTimeout(); err != nil {
                panic(err)
            }
        }
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
panic: context deadline exceeded

goroutine 1 [running]:
main.main()
/go/src/PacktPublishing/Go-Programming-Cookbook-Second-
Edition/go-cookbook/chapter6/pools/example/main.go:7 +0x4e
exit status 2
```

1.  `go.mod`文件可能会被更新，顶级示例目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

能够控制连接池的深度非常有用。这将防止我们过载数据库，但重要的是要考虑在超时的情况下会发生什么。如果您同时强制执行一组连接和严格基于上下文的超时，就像我们在这个示例中所做的那样，将会有一些情况下，您会发现请求经常在尝试建立太多连接的过载应用程序上超时。

这是因为连接将超时等待连接可用。对于`database/sql`的新添加的上下文功能使得为整个请求设置共享超时变得更加简单，包括执行查询所涉及的步骤。

通过这个和其他的示例，使用一个全局的`config`对象传递给`Setup()`函数是有意义的，尽管这个示例只是使用环境变量。

# 使用 Redis

有时您需要持久存储或第三方库和服务提供的附加功能。这个示例将探讨 Redis 作为非关系型数据存储的形式，并展示 Go 语言如何与这些第三方服务进行交互。

由于 Redis 支持具有简单接口的键值存储，因此它是会话存储或具有持续时间的临时数据的绝佳候选者。在 Redis 中指定数据的超时是非常有价值的。这个示例将探讨从配置到查询再到使用自定义排序的基本 Redis 用法。

# 准备工作

根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.11.1 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  从[`www.consul.io/intro/getting-started/install.html`](https://www.consul.io/intro/getting-started/install.html)安装 Consul。

1.  打开一个终端或控制台应用程序，并创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，然后（可选）从该目录中工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

1.  使用[`redis.io/topics/quickstart`](https://redis.io/topics/quickstart)安装和配置 Redis。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter6/redis`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/redis 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/redis    
```

1.  从`~/projects/go-programming-cookbook-original/chapter6/redis`复制测试，或者利用这个练习编写一些自己的代码！

1.  创建一个名为`config.go`的文件，并包含以下内容：

```go
        package redis

        import (
            "os"

            redis "gopkg.in/redis.v5"
        )

        // Setup initializes a redis client
        func Setup() (*redis.Client, error) {
            client := redis.NewClient(&redis.Options{
                Addr: "localhost:6379",
                Password: os.Getenv("REDISPASSWORD"),
                DB: 0, // use default DB
         })

         _, err := client.Ping().Result()
         return client, err
        }
```

1.  创建一个名为`exec.go`的文件，并包含以下内容：

```go
        package redis

        import (
            "fmt"
            "time"

            redis "gopkg.in/redis.v5"
        )

        // Exec performs some redis operations
        func Exec() error {
            conn, err := Setup()
            if err != nil {
                return err
            }

            c1 := "value"
            // value is an interface, we can store whatever
            // the last argument is the redis expiration
            conn.Set("key", c1, 5*time.Second)

            var result string
            if err := conn.Get("key").Scan(&result); err != nil {
                switch err {
                // this means the key
                // was not found
                case redis.Nil:
                    return nil
                default:
                    return err
                }
            }

            fmt.Println("result =", result)

            return nil
        }
```

1.  创建一个名为`sort.go`的文件，并包含以下内容：

```go
package redis

import (
  "fmt"

  redis "gopkg.in/redis.v5"
)

// Sort performs a sort redis operations
func Sort() error {
  conn, err := Setup()
  if err != nil {
    return err
  }

  listkey := "list"
  if err := conn.LPush(listkey, 1).Err(); err != nil {
    return err
  }
  // this will clean up the list key if any of the subsequent commands error
  defer conn.Del(listkey)

  if err := conn.LPush(listkey, 3).Err(); err != nil {
    return err
  }
  if err := conn.LPush(listkey, 2).Err(); err != nil {
    return err
  }

  res, err := conn.Sort(listkey, redis.Sort{Order: "ASC"}).Result()
  if err != nil {
    return err
  }
  fmt.Println(res)

  return nil
}
```

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import "PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                go-cookbook/chapter6/redis"

        func main() {
            if err := redis.Exec(); err != nil {
                panic(err)
            }

            if err := redis.Sort(); err != nil {
                panic(err)
            }
        }
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
result = value
[1 2 3]
```

1.  `go.mod`文件可能已更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一个目录并运行`go test`。确保所有测试都通过。

# 工作原理...

在 Go 中使用 Redis 与使用 MySQL 非常相似。虽然没有标准库，但是许多相同的约定都遵循了，例如使用`Scan()`函数将数据从 Redis 读取到 Go 类型中。在这种情况下，选择最佳库可能会有挑战，我建议定期调查可用的内容，因为事情可能会迅速改变。

这个示例使用`redis`包来进行基本的设置和获取，更复杂的排序功能以及基本的配置。与`database/sql`一样，您可以以写超时、池大小等形式设置额外的配置。Redis 本身还提供了许多额外的功能，包括 Redis 集群支持、Zscore 和计数器对象以及分布式锁。

与前面的示例一样，我建议使用一个`config`对象，它存储您的 Redis 设置和配置详细信息，以便轻松设置和安全性。

# 使用 NoSQL 与 MongoDB

您可能最初认为 Go 更适合关系数据库，因为 Go 结构和 Go 是一种类型化的语言。当使用`github.com/mongodb/mongo-go-driver`包时，Go 可以几乎任意存储和检索结构对象。如果对对象进行版本控制，您的模式可以适应，并且可以提供一个非常灵活的开发环境。

有些库更擅长隐藏或提升这些抽象。`mongo-go-driver`包就是一个很好的例子。下面的示例将以类似的方式创建一个连接，类似于 Redis 和 MySQL，但将存储和检索对象而无需定义具体的模式。

# 准备工作

根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.11.1 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  从[`www.consul.io/intro/getting-started/install.html`](https://www.consul.io/intro/getting-started/install.html)安装 Consul。

1.  打开一个终端或控制台应用程序，并创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，（可选）从该目录中工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

1.  安装和配置 MongoDB（[`docs.mongodb.com/getting-started/shell/`](https://docs.mongodb.com/getting-started/shell/)）。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter6/mongodb`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/mongodb 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/mongodb    
```

1.  从`~/projects/go-programming-cookbook-original/chapter6/mongodb`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`config.go`的文件，内容如下：

```go
package mongodb

import (
  "context"
  "time"

  "github.com/mongodb/mongo-go-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
)

// Setup initializes a mongo client
func Setup(ctx context.Context, address string) (*mongo.Client, error) {
  ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
  // cancel will be called when setup exits
  defer cancel()

  client, err := mongo.NewClient(options.Client().ApplyURI(address))
  if err != nil {
    return nil, err
  }

  if err := client.Connect(ctx); err != nil {
    return nil, err
  }
  return client, nil
}
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
package mongodb

import (
  "context"
  "fmt"

  "github.com/mongodb/mongo-go-driver/bson"
)

// State is our data model
type State struct {
  Name string `bson:"name"`
  Population int `bson:"pop"`
}

// Exec creates then queries an Example
func Exec(address string) error {
  ctx := context.Background()
  db, err := Setup(ctx, address)
  if err != nil {
    return err
  }

  coll := db.Database("gocookbook").Collection("example")

  vals := []interface{}{&State{"Washington", 7062000}, &State{"Oregon", 3970000}}

  // we can inserts many rows at once
  if _, err := coll.InsertMany(ctx, vals); err != nil {
    return err
  }

  var s State
  if err := coll.FindOne(ctx, bson.M{"name": "Washington"}).Decode(&s); err != nil {
    return err
  }

  if err := coll.Drop(ctx); err != nil {
    return err
  }

  fmt.Printf("State: %#v\n", s)
  return nil
}
```

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/mongodb"

func main() {
  if err := mongodb.Exec("mongodb://localhost"); err != nil {
    panic(err)
  }
}
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
State: mongodb.State{Name:"Washington", Population:7062000}
```

1.  `go.mod`文件可能已更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一个目录并运行`go test`。确保所有测试都通过。

# 工作原理...

`mongo-go-driver`包还提供连接池，并且有许多方法可以调整和配置与`mongodb`数据库的连接。本示例的示例相当基本，但它们说明了理解和查询基于文档的数据库是多么容易。该包实现了 BSON 数据类型，与处理 JSON 非常相似。

`mongodb`的一致性保证和最佳实践超出了本书的范围。然而，在 Go 语言中使用这些功能是一种乐趣。

# 为数据可移植性创建存储接口

在使用外部存储接口时，将操作抽象化到接口后面可能会有所帮助。这是为了方便模拟，如果更改存储后端，则可移植性，以及关注点的隔离。这种方法的缺点可能在于，如果您需要在事务内执行多个操作，那么最好是进行组合操作，或者允许通过上下文对象或附加函数参数传递它们。

此示例将实现一个非常简单的接口，用于在 MongoDB 中处理项目。这些项目将具有名称和价格，我们将使用接口来持久化和检索这些对象。

# 准备工作

请参考*使用 NoSQL 与 MongoDB*中*准备工作*部分中给出的步骤。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter6/storage`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/storage 
```

您应该会看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter6/storage    
```

1.  从`~/projects/go-programming-cookbook-original/chapter6/storage`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`storage.go`的文件，内容如下：

```go
        package storage

        import "context"

        // Item represents an item at
        // a shop
        type Item struct {
            Name  string
            Price int64
        }

        // Storage is our storage interface
        // We'll implement it with Mongo
        // storage
        type Storage interface {
            GetByName(context.Context, string) (*Item, error)
            Put(context.Context, *Item) error
        }
```

1.  创建一个名为`mongoconfig.go`的文件，内容如下：

```go
package storage

import (
  "context"
  "time"

  "github.com/mongodb/mongo-go-driver/mongo"
)

// MongoStorage implements our storage interface
type MongoStorage struct {
  *mongo.Client
  DB string
  Collection string
}

// NewMongoStorage initializes a MongoStorage
func NewMongoStorage(ctx context.Context, connection, db, collection string) (*MongoStorage, error) {
  ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
  defer cancel()

  client, err := mongo.Connect(ctx, "mongodb://localhost")
  if err != nil {
    return nil, err
  }

  ms := MongoStorage{
    Client: client,
    DB: db,
    Collection: collection,
  }
  return &ms, nil
}
```

1.  创建一个名为`mongointerface.go`的文件，内容如下：

```go
package storage

import (
  "context"

  "github.com/mongodb/mongo-go-driver/bson"
)

// GetByName queries mongodb for an item with
// the correct name
func (m *MongoStorage) GetByName(ctx context.Context, name string) (*Item, error) {
  c := m.Client.Database(m.DB).Collection(m.Collection)
  var i Item
  if err := c.FindOne(ctx, bson.M{"name": name}).Decode(&i); err != nil {
    return nil, err
  }

  return &i, nil
}

// Put adds an item to our mongo instance
func (m *MongoStorage) Put(ctx context.Context, i *Item) error {
  c := m.Client.Database(m.DB).Collection(m.Collection)
  _, err := c.InsertOne(ctx, i)
  return err
}
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
package storage

import (
  "context"
  "fmt"
)

// Exec initializes storage, then performs operations
// using the storage interface
func Exec() error {
  ctx := context.Background()
  m, err := NewMongoStorage(ctx, "localhost", "gocookbook", "items")
  if err != nil {
    return err
  }
  if err := PerformOperations(m); err != nil {
    return err
  }

  if err := m.Client.Database(m.DB).Collection(m.Collection).Drop(ctx); err != nil {
    return err
  }

  return nil
}

// PerformOperations creates a candle item
// then gets it
func PerformOperations(s Storage) error {
  ctx := context.Background()
  i := Item{Name: "candles", Price: 100}
  if err := s.Put(ctx, &i); err != nil {
    return err
  }

  candles, err := s.GetByName(ctx, "candles")
  if err != nil {
    return err
  }
  fmt.Printf("Result: %#v\n", candles)
  return nil
}
```

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import "PacktPublishing/Go-Programming-Cookbook-Second-Edition/
                go-cookbook/chapter6/storage"

        func main() {
            if err := storage.Exec(); err != nil {
                panic(err)
            }
        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该会看到以下输出：

```go
$ go run main.go
Result: &storage.Item{Name:"candles", Price:100}
```

1.  `go.mod`文件可能会被更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

演示此示例最重要的函数是`PerformOperations`。此函数将一个`Storage`接口作为参数。这意味着我们可以动态替换底层存储，甚至无需修改此函数。例如，连接存储到单独的 API 以消费和修改它将是很简单的。

我们使用上下文来为这些接口添加额外的灵活性，并允许接口处理超时。将应用程序逻辑与底层存储分离提供了各种好处，但很难选择正确的划界线的位置，这将因应用程序而异。
