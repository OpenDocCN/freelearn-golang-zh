# 所有关于数据库和存储

在本章中，将涵盖以下食谱：

+   MySQL 的 database/sql 包

+   执行数据库事务接口

+   SQL 的连接池、速率限制和超时

+   与 Redis 一起工作

+   使用 MongoDB 和 mgo 的 NoSQL

+   为数据可移植性创建存储接口

# 简介

Go 应用程序通常需要使用长期存储。这通常以关系型和非关系型数据库、键值存储等形式存在。当与这些存储应用程序一起工作时，将你的操作封装在接口中很有帮助。本章中的食谱将检查各种存储接口，考虑连接池等并行访问，并查看集成新库的一般技巧，这在使用新存储技术时通常是情况。

# MySQL 的 database/sql 包

关系型数据库是一些最被理解和常见的数据库选项之一。MySQL 和 Postgres 是最受欢迎的开源关系型数据库。本食谱将演示 `database/sql` 包，这是一个提供多个关系型数据库钩子的包，并自动处理连接池、连接时长，并提供对多个基本数据库操作的访问。

本包的未来版本将包括对上下文和超时的支持。

# 准备工作

根据以下步骤配置你的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到你的操作系统，并配置你的 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序，导航到你的 `GOPATH/src` 并创建一个项目目录，例如

    `$GOPATH/src/github.com/你的用户名/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

1.  运行 `go get github.com/go-sql-driver/mysql` 命令。

1.  使用 [`dev.mysql.com/doc/mysql-getting-started/en/`](https://dev.mysql.com/doc/mysql-getting-started/en/) 安装和配置 MySQL。

1.  运行 `export MYSQLUSERNAME=<你的 MySQL 用户名>` 命令。

1.  运行 `export MYSQLPASSWORD=<你的 MySQL 密码>` 命令。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序创建并导航到目录 `chapter5/database`。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter5/database`](https://github.com/agtorre/go-cookbook/tree/master/chapter5/database) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `config.go` 的文件，内容如下：

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

1.  创建一个名为 `create.go` 的文件，内容如下：

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

1.  创建一个名为 `query.go` 的文件，内容如下：

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
        func Query(db *sql.DB) error {
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

1.  创建一个名为 `exec.go` 的文件，内容如下：

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

1.  创建并导航到 `example` 目录。

1.  创建一个名为 `main.go` 的文件，内容如下；请确保将 `database` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
            "github.com/agtorre/go-cookbook/chapter5/database"
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

1.  运行 `go run main.go`。

1.  您也可以运行以下命令：

```go
 go build ./example

```

您应该看到以下输出：

```go
 $ go run main.go
 Results:
 Name: Aaron
 Created: 2017-02-16 19:02:36 +0000 UTC

```

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

代码中的 `_ "github.com/go-sql-driver/mysql"` 行是如何将各种数据库连接器连接到 `database/sql` 包的。如果您要连接到 Postgres、SQLite 或其他实现 `database/sql` 接口的数据库，命令将类似。

连接后，该包设置了一个连接池，这在 *Connection pooling, rate limiting, and timeouts for SQL* 菜谱中有介绍，您可以直接在连接上执行 SQL，或者创建可以执行 `commit` 和 `rollback` 命令的所有连接可以执行的事务对象。

`mysql` 包在与数据库通信时为 Go 时间对象提供了一些便利支持。此菜谱还从 `MYSQLUSERNAME` 和 `MYSQLPASSWORD` 环境变量中检索用户名和密码。

# 执行数据库事务接口

当与数据库等服务进行连接时，编写测试可能很困难。这是因为 Go 在运行时模拟或鸭子类型化事物很困难。虽然我建议在处理数据库时使用存储接口，但在这个接口内部模拟数据库事务接口仍然很有用。*创建用于数据可移植性的存储接口* 菜谱将涵盖存储接口；这个菜谱将专注于包装数据库连接和事务对象的接口。

为了展示此类接口的使用，我们将重写之前菜谱中的创建和查询文件，以使用我们的接口。最终输出将相同，但创建和查询操作都将在一个事务中执行。

# 准备工作

根据以下步骤配置您的环境：

1.  参考菜谱 *The database/sql package with MySQL* 中的 *Getting ready* 部分的步骤。

1.  运行 `go get https://github.com/agtorre/go-cookbook/tree/master/chapter5/database` 命令或使用 *The database/sql package with MySQL* 菜谱编写自己的命令。

# 如何操作...

这些步骤涵盖了编写和运行您的应用程序：

1.  在您的终端/控制台应用程序中，创建并导航到 `chapter5/dbinterface` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter5/dbinterface`](https://github.com/agtorre/go-cookbook/tree/master/chapter5/dbinterface) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `transaction.go` 的文件，内容如下：

```go
        package database

        import _ "github.com/go-sql-driver/mysql" //we import supported
        libraries for database/sql
        // Exec grabs a new connection
        // creates tables, and later drops them
        // and issues some queries
        func Exec() error {
            db, err := Setup()
            if err != nil {
                return err
            }
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

1.  创建一个名为 `create.go` 的文件，内容如下：

```go
        package dbinterface

        import _ "github.com/go-sql-driver/mysql" //we import supported
        libraries for database/sql

        // Create makes a table called example
        // and populates it
        func Create(db DB) error {
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

1.  创建一个名为 `query.go` 的文件，内容如下：

```go
        package dbinterface

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter5/database"
        )

        // Query grabs a new connection
        // creates tables, and later drops them
        // and issues some queries
        func Query(db DB) error {
            name := "Aaron"
            rows, err := db.Query("SELECT name, created FROM example 
            where name=?", name)
            if err != nil {
                return err
            }
            defer rows.Close()
            for rows.Next() {
                var e database.Example
                if err := rows.Scan(&e.Name, &e.Created); err != nil {
                    return err
                }
                fmt.Printf("Results:\n\tName: %s\n\tCreated: %v\n", 
                e.Name, e.Created)
            }
            return rows.Err()
        }

```

1.  创建一个名为 `exec.go` 的文件，内容如下：

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

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `dbinterface` 导入路径修改为你在第 2 步中设置的路径：

```go
        package main

        import (
            "github.com/agtorre/go-cookbook/chapter5/database"
            "github.com/agtorre/go-cookbook/chapter5/dbinterface"
            _ "github.com/go-sql-driver/mysql" //we import supported 
            libraries for database/sql
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

         if err := dbinterface.Exec(db); err != nil {
             panic(err)
         }
         if err := tx.Commit(); err != nil {
             panic(err)
         }
        }

```

1.  运行 `go run main.go`.

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Results:
 Name: Aaron
 Created: 2017-02-16 20:00:00 +0000 UTC

```

1.  如果你复制或编写了自己的测试，请向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

这个菜谱与上一个菜谱非常相似，但展示了同时使用事务和创建通用的数据库函数，这些函数可以与 `sql.DB` 连接和 `sql.Transaction` 对象一起使用。正如你将在第八章 *Testing* 中看到的，这些接口也很容易模拟。

# SQL 的连接池、速率限制和超时

虽然`database/sql`包提供了对连接池、速率限制和超时的支持，但通常需要调整默认设置以更好地适应你的数据库配置。当你对微服务进行横向扩展且不希望保持过多的数据库活动连接时，这可能会变得很重要。

# 准备工作

根据以下步骤配置你的环境：

1.  参考菜谱 *The database/sql package with MySQL* 中 *准备工作* 部分的步骤。

1.  运行 `go get https://github.com/agtorre/go-cookbook/tree/master/chapter5/database` 命令，或者使用 *The database/sql package with MySQL* 菜单编写自己的命令。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建并导航到 `chapter5/pools` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter5/pools`](https://github.com/agtorre/go-cookbook/tree/master/chapter5/pools) 复制测试，或者将此作为练习编写一些自己的代码。

1.  创建一个名为 `pools.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `timeout.go` 的文件，并包含以下内容：

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
            ctx, can := context.WithDeadline(ctx, time.Now())

            // call cancel after we complete
            defer can()

            // our transaction is context aware
            _, err = db.BeginTx(ctx, nil)
            return err
        }

```

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `pools` 导入路径修改为你在第 2 步中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter5/pools"

        func main() {
            if err := pools.ExecWithTimeout(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 panic: context deadline exceeded

 goroutine 1 [running]:
 main.main()
 /go/src/github.com/agtorre/go-  
      cookbook/chapter5/pools/example/main.go:7 +0x4e
 exit status 2

```

1.  如果你复制或编写了自己的测试，请向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

能够控制连接池的深度非常有用。这将使我们不会过载数据库，但重要的是要考虑在超时上下文中的含义。如果你强制执行固定数量的连接和严格基于上下文的时间超时，就像我们在本菜谱中所做的那样，那么在尝试建立过多连接的过载应用程序中，你将会有请求频繁超时的情况。

这是因为连接将在等待连接变得可用时超时。`database/sql` 新增的上下文功能使得为整个请求（包括执行查询的步骤）设置共享超时变得更加简单。

与其他食谱一样，使用全局 `config` 对象传递给 `Setup()` 函数是有意义的，尽管这个食谱只是使用了环境变量。

# 使用 Redis

有时你可能需要持久化存储或第三方库和服务提供的附加功能。本食谱将探索 Redis 作为一种非关系型数据存储的形式，并展示一种如 Go 的语言如何与这些服务交互。

由于 Redis 支持使用简单接口进行键值存储，因此它非常适合用于会话存储或具有持续时间的数据。能够指定存储在 Redis 中的数据超时非常宝贵。本食谱将探索从配置到查询再到使用自定义排序的基本 Redis 使用方法。

# 准备工作

按照以下步骤配置你的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到你的操作系统，并配置你的 `GOPATH` 环境变量。

1.  打开一个终端/控制台应用程序。

1.  导航到你的 `GOPATH/src` 并创建一个项目目录，例如 `$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

1.  运行 `go get gopkg.in/redis.v5` 命令。

1.  使用 [`redis.io/topics/quickstart`](https://redis.io/topics/quickstart) 安装和配置 Redis。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter5/redis` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter5/redis`](https://github.com/agtorre/go-cookbook/tree/master/chapter5/redis) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `config.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `exec.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `sort.go` 的文件，并包含以下内容：

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

            if err := conn.LPush("list", 1).Err(); err != nil {
                return err
            }
            if err := conn.LPush("list", 3).Err(); err != nil {
                return err
            }
            if err := conn.LPush("list", 2).Err(); err != nil {
                return err
            }

            res, err := conn.Sort("list", redis.Sort{Order: 
            "ASC"}).Result()
            if err != nil {
                return err
            }
            fmt.Println(res)
            conn.Del("list")
            return nil
        }

```

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `redis` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter5/redis"

        func main() {
            if err := redis.Exec(); err != nil {
                panic(err)
            }

            if err := redis.Sort(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 result = value
 [1 2 3]

```

1.  如果你复制或编写了自己的测试，请向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

在 Go 中使用 Redis 与在 MySQL 中使用非常相似，尽管没有标准库，但很多相同的约定都遵循了，比如使用`Scan()`函数从 Redis 读取数据到 Go 类型。在这种情况下选择最佳库可能会很具挑战性，我建议定期调查可用的库，因为事情可能会迅速变化。

这个菜谱使用`redis`包来进行基本的设置和获取，执行更复杂的排序函数，以及基本配置。就像`database/sql`一样，你可以以写入超时、连接池大小等形式设置额外的配置。Redis 本身也提供了很多额外的功能，包括 Redis 集群支持、Zscore 和计数器对象、分布式锁等。

如前一个菜谱中建议的，我建议使用一个`config`对象，它存储你的 Redis 设置和配置细节，以便于设置和安全。

# 使用 MongoDB 和 mgo 的 NoSQL

你可能会首先认为 Go 更适合关系型数据库，因为 Go 有结构体，并且 Go 是一种静态类型语言。当使用类似`mgo`包的东西时，Go 可以几乎任意地存储和检索结构体对象。如果你对对象进行版本控制，你的模式可以适应，并且它可以提供一个非常灵活的开发环境。

一些库在隐藏或提升这些抽象方面做得更好。`mgo`包是一个很好的例子，它出色地完成了前者。这个菜谱将以类似 Redis 和 MySQL 的方式创建连接，但将存储和检索对象，甚至不需要定义具体的模式。

# 准备工作

根据以下步骤配置你的环境：

1.  从[`golang.org/doc/install`](https://golang.org/doc/install)下载并安装 Go 到你的操作系统上，并配置你的`GOPATH`环境变量。

1.  打开一个终端/控制台应用程序。

1.  导航到你的`GOPATH/src`并创建一个项目目录，例如`$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用`go get github.com/agtorre/go-cookbook/`命令安装代码的最新测试版本。

1.  运行`go get gopkg.in/mgo.v2`命令。

1.  要运行代码，你需要一个连接到 MongoDB 实例的工作数据库连接，本书将不会涉及这部分内容。

1.  基本设置是[`docs.mongodb.com/getting-started/shell/`](https://docs.mongodb.com/getting-started/shell/)。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到`chapter5/mongodb`目录。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter5/mongodb`](https://github.com/agtorre/go-cookbook/tree/master/chapter5/mongodb)复制测试，或者将其作为练习编写一些你自己的代码。

1.  创建一个名为`config.go`的文件，内容如下：

```go
        package mongodb

        import mgo "gopkg.in/mgo.v2"

        // Setup initializes a redis client
        func Setup() (*mgo.Session, error) {
            session, err := mgo.Dial("localhost")
            if err != nil {
                return nil, err
            }
            return session, nil
        }

```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
        package mongodb

        import (
            "fmt"

            "gopkg.in/mgo.v2/bson"
        )

        // State is our data model
        type State struct {
            Name string `bson:"name"`
            Population int `bson:"pop"`
        }

        // Exec creates then queries an Example
        func Exec() error {
            db, err := Setup()
            if err != nil {
                return err
            }

            conn := db.DB("gocookbook").C("example")

            // we can inserts many rows at once
            if err := conn.Insert(&State{"Washington", 7062000}, 
            &State{"Oregon", 3970000}); err != nil {
                return err
            }

            var s State
            if err := conn.Find(bson.M{"name": "Washington"}).One(&s); 
            err!= nil {
                return err
            }

            if err := conn.DropCollection(); err != nil {
                return err
            }

            fmt.Printf("State: %#vn", s)
            return nil
        }

```

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，内容如下；请确保将`mongodb`导入修改为使用你在步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter5/mongodb"

        func main() {
            if err := mongodb.Exec(); err != nil {
                panic(err)
            }
        }

```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 State: mongodb.State{Name:"Washington", Population:7062000}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`mgo`包还提供了连接池，以及许多调整和配置与`mongodb`数据库连接的方法。此配方的示例相当基础，但它们说明了推理和查询基于文档的数据库是多么容易。该包实现了 BSON 数据类型，并且与它的序列化和反序列化非常类似于处理 JSON。

一致性保证和`mongodb`的最佳实践超出了本书的范围--但在 Go 语言中使用它是一种乐趣。

# 创建用于数据可移植性的存储接口

当与外部存储接口一起工作时，将操作抽象化在接口后面可能会有所帮助。这样做是为了方便模拟、在更改存储后端时的可移植性，以及关注点的隔离。这种方法的缺点可能在于，如果你需要在事务内部执行多个操作。在这种情况下，创建组合操作或允许通过上下文对象或额外的函数参数传递它是有意义的。

此配方将实现一个用于与 MongoDB 中的项目交互的非常简单的接口。这些项目将有一个名称和价格，我们将使用接口来持久化和检索这些对象。

# 准备工作

请参考`使用 MongoDB 和 mgo 的 NoSQL`配方中“准备工作”部分给出的步骤。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到`chapter5/mongodb`目录。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter5/mongodb`](https://github.com/agtorre/go-cookbook/tree/master/chapter5/mongodb)复制测试或将其作为练习编写你自己的代码。

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

        import mgo "gopkg.in/mgo.v2"

        // MongoStorage implements our storage interface
        type MongoStorage struct {
            *mgo.Session
            DB string
            Collection string
        }

        // NewMongoStorage initializes a MongoStorage
        func NewMongoStorage(connection, db, collection string) 
        (*MongoStorage, error) {
            session, err := mgo.Dial("localhost")
            if err != nil {
                return nil, err
            }
            ms := MongoStorage{
                Session: session,
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

            "gopkg.in/mgo.v2/bson"
        )

        // GetByName queries mongodb for an item with
        // the correct name
        func (m *MongoStorage) GetByName(ctx context.Context, name 
        string) (*Item, error) {
            c := m.Session.DB(m.DB).C(m.Collection)
            var i Item
            if err := c.Find(bson.M{"name": name}).One(&i); err != nil 
            {
                return nil, err
            }

            return &i, nil
        }

        // Put adds an item to our mongo instance
        func (m *MongoStorage) Put(ctx context.Context, i *Item) error 
        {
            c := m.Session.DB(m.DB).C(m.Collection)
            return c.Insert(i)
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
            m, err := NewMongoStorage("localhost", "gocookbook", 
            "items")
            if err != nil {
                return err
            }
            if err := PerformOperations(m); err != nil {
                return err
            }

            if err := 
            m.Session.DB(m.DB).C(m.Collection).DropCollection(); 
            err != nil {
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
            fmt.Printf("Result: %#vn", candles)
                return nil
        }

```

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，内容如下；请确保将`storage`导入修改为使用你在步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter5/storage"

        func main() {
            if err := storage.Exec(); err != nil {
                panic(err)
            }
        }

```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Result: &storage.Item{Name:"candles", Price:100}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

展示此菜谱最重要的功能是 `PerformOperation`。此函数接受存储接口作为参数。这意味着我们可以动态地替换底层的存储，甚至无需修改此函数。例如，将存储连接到单独的 API 以消费和修改它将变得非常简单。

我们为这些接口添加了上下文，以增加额外的灵活性并允许接口处理超时。将您的应用程序逻辑与底层存储分离提供了各种好处，但选择合适的边界位置可能很困难，并且这会因应用程序而异。
