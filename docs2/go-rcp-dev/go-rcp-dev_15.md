# 15

# 数据库

大多数应用程序至少需要与一种类型的数据库进行交互。SQL 数据库足够常见，以至于 Go 标准库提供了一个统一的方式来连接和使用它们。本章展示了你可以使用的某些模式，以与 SQL 包的标准库实现一起工作。

许多数据库在功能和查询语言方面都提供了非标准扩展。即使你使用标准库与数据库接口，你也应该始终检查特定供应商的数据库驱动程序，以了解潜在的限制、实现差异和支持的 SQL 语法。

在这里，提到 NoSQL 数据库可能会有所帮助。Go 标准库不提供 NoSQL 数据库包。这是因为，与 SQL 不同，大多数 NoSQL 数据库都有非标准的查询语言，这些语言是为特定数据库专门定制的。为特定工作负载构建的 NoSQL 数据库比通用 SQL 数据库表现要好得多。如果你正在使用此类数据库，请参阅其文档。然而，本章中提出的许多概念在一定程度上也适用于 NoSQL 数据库。

本章包含以下食谱：

+   连接到数据库

+   执行 SQL 语句

+   不使用显式事务执行 SQL 语句

+   使用事务执行 SQL 语句

+   在事务中执行预定义语句

+   从查询中获取值

+   动态构建 SQL 语句

+   构建 `UPDATE` 语句

+   构建 `WHERE` 子句

# 连接到数据库

你可以将数据库集成到你的应用程序中的两种方式：你可以使用数据库服务器或嵌入式数据库。让我们首先定义一下它们是什么。

数据库服务器作为一个独立进程在同一主机或不同主机上运行，但与你的应用程序无关。通常，你的应用程序通过网络连接连接到这个数据库服务器，因此你必须知道它的网络地址和端口号。通常有一个库需要导入到你的程序中，这是一个针对你使用的数据库服务器的“数据库驱动程序”。这个驱动程序通过管理连接、查询、事务等，为你的应用程序和数据库提供接口。

嵌入式数据库不是一个独立的进程。它作为库包含在你的应用程序中，并在相同的地址空间中运行。数据库驱动程序充当适配器，向应用程序提供一个标准接口（即使用 `database/sql` 包）。当使用嵌入式数据库时，你必须注意与其他进程共享的资源。许多嵌入式数据库不允许多个程序访问相同的基本数据。

在执行任何操作之前，你必须连接到数据库服务器（如 MySQL 或 PostgreSQL 服务器）或嵌入式数据库引擎（如 SQLite）。

小贴士

此页面包含 SQL 驱动程序的列表：[`go.dev/wiki/SQLDrivers`](https://go.dev/wiki/SQLDrivers)。

## 如何做到这一点...

找到你需要的数据库特定驱动程序。这个驱动程序可能由数据库供应商提供，或者作为一个开源项目发布。在`main`包中导入这个数据库驱动程序。

你需要一个特定于驱动程序的驱动程序名称和连接字符串来连接到数据库服务器或嵌入式数据库引擎。如果你正在连接到数据库服务器，这个连接字符串通常包括主机/端口信息、认证信息和连接选项。如果是嵌入式数据库引擎，它可能包括文件名/目录信息。然后，你可以调用`sql.Open`或使用特定于驱动程序的连接函数，该函数返回一个`*sql.DB`。

数据库驱动程序可能会延迟实际连接到第一次数据库操作。也就是说，使用`sql.Open`连接到数据库可能不会立即连接。为确保你已连接到数据库，请使用`DB.Ping`。嵌入式数据库驱动程序通常不需要 ping。

以下是一个连接到 MySQL 数据库的示例：

```go
package main
import (
    "fmt"
    "database/sql"
    "context"
    // Import the mysql driver
    _ "github.com/go-sql-driver/mysql"
)
func main() {
    // Use mysql driver name and driver specific connection string
    db, err := sql.Open("mysql", "username:password@tcp(host:port)/
    databaseName")
    if err != nil {
        panic(err.Error())
    }
    defer db.Close()
    // Check if database connection succeeded, with 5 second timeout
    ctx, cancel := context.WithTimeout(context.
    Background(),5*time,Second)
    defer cancel()
    if err:=db.PingContext(ctx); err!=nil {
        panic(err)
    }
    fmt.Println("Success!")
}
```

以下是一个使用本地文件连接到内存中 SQLite 数据库的示例：

```go
package main
import (
    "database/sql"
    "fmt"
    "os"
    // Import the database driver
    _ "github.com/mattn/go-sqlite3"
)
func main() {
    // Open the sqlite database using the given local file ./database.
    // db
    db, err := sql.Open("sqlite3", "./database.db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    // You don't need to ping an embedded database
}
```

小贴士

注意使用空白标识符`_`来导入数据库驱动程序。这意味着包仅为了其副作用而导入，在这种情况下，是注册数据库驱动的`init()`函数。例如，在`main`包中导入`go-sqlite3`包会导致在`go-sqlite3`中声明的`init()`函数使用名称`sqlite3`注册自己。

# 运行 SQL 语句

在获取`*sql.DB`实例后，你可以运行 SQL 语句来修改或查询数据。这些查询只是 SQL 字符串，但 SQL 的样式在不同的数据库供应商之间有所不同。

## 不使用显式事务运行 SQL 语句

当与数据库交互时，一个重要的考虑因素是确定事务边界。如果你需要执行单个操作，例如插入一行或运行一个查询，你通常不需要显式创建事务。你可以执行一个将开始和结束事务的单个 SQL 语句。然而，如果你有多个 SQL 语句，这些语句应该作为一个原子单元运行或者根本不运行，你必须使用事务。

### 如何操作...

1.  要运行 SQL 语句更新数据，请使用`DB.Exec`或`DB.ExecContext`：

    ```go
    result, err:=db.ExecContext(ctx,`UPDATE users SET user.last_login=? WHERE user_id=?",time.Now(), userId)
    if err!=nil {
      // Handle error
    }
    n, err:=result.RowsAffected()
    if err!=nil {
      // Handle error
    }
    if n!=1 {
      return errors.New("Cannot update last login time")
    }
    ```

    要多次运行相同的语句但使用不同的值，请使用预编译语句。预编译语句通常将语句发送到数据库服务器，在那里它被解析和准备。然后，你可以简单地使用不同的参数运行这个解析后的语句，绕过数据库引擎的解析和优化阶段。

    当你完成使用预编译语句时，你应该关闭它：

    ```go
    func AddUsers(db *sql.DB, users []User) error {
      stmt, err := db.Prepare(`INSERT INTO users (user_name,email) 
      VALUES (?,?)`)
      if err!=nil {
        return err
      }
      // Close the prepared statement when done
      defer stmt.Close()
      for _,user:=range users {
        // Run the prepared statement with different arguments
        _, err := stmt.Exec(user.Name,user.Email)
        if err!=nil {
          return err
        }
      }
      return nil
    }
    ```

小贴士

在连接到数据库后，你可以创建预编译语句并在程序结束时使用它们。预编译语句可以从多个 goroutines 并发执行。

要运行返回结果的查询，请使用`DB.Query`或`DB.QueryContext`。要运行预期最多返回一行的查询，你可以使用`DB.QueryRow`或`DB.QueryRowContext`便利函数。

`DB.Query`和`DB.QueryContext`方法返回一个`*sql.Rows`对象，它本质上是对查询结果的单向游标。这提供了一个接口，允许你在不将所有结果加载到内存的情况下处理大型结果集。数据库引擎通常分批返回结果，而`*sql.Rows`对象允许你逐行遍历结果行，按需批量获取结果。

另一点需要记住的是，许多数据库引擎会延迟查询的实际执行，直到你开始获取结果。换句话说，仅仅因为你运行了一个查询，并不意味着该查询实际上被服务器评估。查询评估可能发生在你获取第一行结果时：

```go
func GetUserNamesLoggedInAfter(db *sql.DB, after time.Time) ([]string,error) {
  rows, err:=db.Query(`SELECT users.user_name FROM users WHERE 
  last_login > ?`, after)
  if err!=nil {
    return nil,err
  }
  defer rows.Close()
  names:=make([]string,0)
  for rows.Next() {
    var name string
    if err:=rows.Scan(&name); err!=nil {
      return nil,err
    }
    names=append(names,name)
  }
  // Check if iteration produced any errors
  if err:=rows.Err(); err!=nil {
    return nil,err
  }
  return names,nil
}
```

如果预期的结果集最多只有一行（换句话说，你正在寻找一个可能存在也可能不存在的特定对象），你可以通过使用`DB.QueryRow`或`DB.QueryRowContext`来缩短上述模式。你可以通过检查返回的错误是否为`sql.ErrNoRows`来确定操作是否找到了该行：

```go
func GetUserByID(db *sql.DB, id string) (*User, error) {
  var user User
  err:=db.QueryRow(`SELECT user_id, user_name, last_login FROM 
  users WHERE user_id=?`,id).
    Scan(&user.Id, &user.Name, &user.LastLogin)
  if errors.Is(err,sql.ErrNoRows) {
    return nil,nil
  }
  if err!=nil {
    return nil,err
  }
  return &user,nil
}
```

永远不要在未经验证的情况下使用用户提供的值、从配置文件中读取的值或从 API 请求中接收的值来构建 SQL 语句。使用查询参数来避免 SQL 注入攻击。

## 使用事务运行 SQL 语句

如果你需要原子性地执行多个更新，你必须在一个事务中执行这些更新。在这种情况下，原子性意味着要么所有更新都成功完成，要么没有任何一个更新完成。

事务隔离级别决定了其他并发事务如何看到事务内执行的更新。你可以找到许多描述事务隔离级别的资源。在这里，我将提供一个总结，帮助你决定哪种隔离级别最适合你的用例：

+   `sql.LevelReadUncommitted`：这是最低的事务隔离级别。一个事务可能看到另一个事务执行的未提交更改。另一个事务可能读取一些未提交的数据，并基于读取的内容执行业务逻辑。而这些未提交的数据可能会回滚，从而使业务逻辑无效。

+   `sql.ReadCommitted`：一个事务只读取另一个事务执行的已提交更改。这意味着如果一个事务试图读取/写入另一个事务正在修改的数据，第一个事务必须等待第二个事务完成。然而，一旦一个事务在 ReadCommitted 隔离级别读取了数据，另一个事务可能改变它。

+   `sql.RepeatableRead`：事务只读取另一个事务执行的已提交更改。此外，在可重复读隔离级别下，事务读取的值保证在事务提交或回滚之前保持不变。任何尝试修改可重复读事务读取的数据的其他事务都将等待直到可重复读事务结束。然而，此隔离级别不能防止其他事务向表中插入满足可重复读事务查询准则的行，因此使用范围查询查询同一表可能会得到不同的结果。

+   `sql.Serializable`：这是最高的事务隔离级别。可序列化事务只读取已提交的更改，防止其他事务修改它所读取的数据，并防止其他事务插入/更新/删除与事务内执行的任何查询的准则相匹配的行。

随着事务隔离级别的提高，并发级别会降低。这也影响性能：较低的隔离级别更快。你必须仔细选择隔离级别：选择对操作安全的最低隔离级别。通常，如果你没有明确指定级别，将使用特定于驱动程序的默认隔离级别。

### 如何操作...

使用期望的隔离级别启动事务：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
// 1\. Start transaction
tx, err := db.BeginTx(ctx, &sql.TxOptions{
  Isolation: sql.LevelReadCommitted,
  })
if err!=nil {
  // Handle error
}
// 2\. Call rollback with defer, so in case of error, transaction
// rolls back
defer tx.Rollback()
```

确保事务要么提交要么回滚。你可以通过延迟调用`tx.Rollback`来实现这一点。如果在函数返回之前没有提交事务，这将导致事务回滚。如果事务成功，你将提交事务。一旦事务被提交，延迟回滚将不会有任何效果。

使用事务执行数据库操作。所有使用`*sql.Tx`方法执行的数据库操作都将在该事务内完成：

```go
_, err:= tx.Exec(`UPDATE users SET user.last_login=? WHERE user_id=?",time.Now()`, userId)
if err!=nil {
  // Do not commit, handle error
}
```

如果没有错误，提交事务：

```go
tx.Commit()
```

小贴士

一些数据库驱动程序可能在由于约束违反（如唯一索引上的重复值）而无法完成查询时回滚并取消事务。请检查你的驱动程序文档，以查看它是否执行自动回滚。

# 在事务内运行准备好的语句

可以通过调用事务结构体的`*sql.Tx.Prepare`或`*sql.Tx.PrepareContext`方法来准备一个语句。这两个方法返回的准备语句仅与该事务相关联。也就是说，你不能使用一个事务准备一个语句，然后用于另一个事务。

## 如何操作...

你有两种方法可以在事务中使用准备好的语句。

第一种方法是使用由`*DB`准备的语句：

1.  使用`DB.Prepare`或`DB.PrepareContext`准备语句。

1.  获取特定于事务的事务副本：

    ```go
    txStmt := tx.Stmt(stmt)
    ```

1.  使用新语句运行操作。

1.  第二种方法是使用由`*Tx`准备的语句：

1.  使用 `Tx.Prepare` 或 `Tx.PrepareContext` 准备语句。

1.  使用此语句运行操作。

# 从查询中获取值

SQL 查询返回 `*sql.Rows`，或者如果你使用 `QueryRow` 方法，它返回 `*sql.Row`。接下来你必须做的事情是遍历行并将值扫描到 Go 变量中。

## 如何做到这一点...

运行 `Query` 或 `QueryContext` 意味着你期望从查询中获取零行或多行。因此，它返回 `*sql.Rows`。

对于本节中的代码片段，我们使用以下 `User` 结构体：

```go
type User struct {
  ID        uint64
  Name      string
  LastLogin time.Time
  AvatarURL string
}
```

这与以下表定义一起使用：

```go
CREATE TABLE users (
  user_id int not null,
  user_name varchar(32) not null,
  last_login timestamp null,
  avatar_url varchar(128) null
)
```

遍历行并处理每个单独的结果行。在以下示例中，查询返回零行或多行。对 `rows.Next` 的第一次调用将移动到结果集的第一行，对 `rows.Next` 的后续调用将移动到下一行。这允许使用 `for` 语句，如下面的示例所示：

```go
rows, err := db.Query(`SELECT user_id, user_name, last_login, avatar_url FROM users WHERE last_login > ?`, after)
if err!=nil {
  return err
}
// Close the rows object when done
defer rows.Close()
for rows.Next() {
  // Retrieve data from this row
}
```

对于每一行，使用 `Scan` 将数据复制到 Go 变量中：

```go
users:=make([]User,0)
for rows.Next() {
  // Retrieve data from this row
  var user User
  // avatar column is nullable, so we pass a *string instead of string
  var avatarURL *string
  if err:=rows.Scan(
    &user.ID,
    &user.Name,
    &user.LastLogin,
    &avatarURL);err!=nil {
      return err
    }
    // avatar URL can be nil in the db
    if avatarURL!=nil {
      user.AvatarURL=*avatarURL
    }
    users=append(users,user)
}
```

`Scan` 的参数顺序必须与从 `SELECT` 语句检索到的列的顺序相匹配。也就是说，第一个参数 `&user.ID` 对应于 `user_id` 列；下一个参数 `&user.Name` 对应于 `user_name` 列；依此类推。因此，`Scan` 的参数数量必须等于检索到的列数。

SQL 驱动程序将数据库原生类型转换为 Go 数据类型。如果转换导致数据或精度丢失，驱动程序通常会返回一个错误。例如，如果你尝试将大整数值扫描到 `int16` 变量中，并且转换无法表示该值，`Scan` 将返回一个错误。

如果数据库列定义为可空（在这个例子中，`avatar_url varchar(128) NULL`），并且如果从数据库检索到的数据值为空，那么 Go 值必须能够容纳空值。例如，如果我们使用 `&user.AvatarURL` 在 `Scan` 中，并且数据库中的值是空，那么 `Scan` 将返回一个错误，抱怨空值不能扫描到字符串。为了防止此类错误，我们使用了 `*string` 而不是 `string`。一般来说，如果底层数据库列是可空的，你应该在 `Scan` 中为该列使用指针。

在获取所有行后检查错误：

```go
// Check if there was an error during iteration
if err:=rows.Err(); err!=nil {
  return err
}
```

关闭 `*sql.Rows`。这通常通过之前的 `defer rows.Close()` 语句来完成。

运行 `QueryRow` 或 `QueryRowContext` 意味着你期望从查询中获取零行或一行。然后，返回一个 `*sql.Row` 对象，你可以使用它来扫描值并检查错误。

运行 `QueryRow` 或 `QueryRowContext`，并按前面描述的方式扫描值：

```go
var user User
row:=db.QueryRow(`SELECT user_id, user_name, last_login, avatar_url FROM users WHERE user_id = ?`, id)
if err:=row.Scan(
   &user.ID,
   &user.Name,
   &user.LastLogin,
   &avatarURL);err!=nil {
  return err
}
return user
```

如果查询执行期间发生错误，它将通过行返回。

# 动态构建 SQL 语句

在任何使用 SQL 数据库的非平凡应用程序中，你将不得不动态构建 SQL 语句。这在以下情况下变得必要：

+   使用灵活的搜索条件，这些条件可能根据用户输入或请求而变化

+   根据请求的字段可选地连接多个表

+   选择性地更新列子集

+   插入可变数量的列

本节展示了构建 SQL 语句的几种常见方法，适用于不同的用例。

小贴士

有许多开源查询构建器包。在编写自己的包之前，您可能想要探索这些包。

# 构建 UPDATE 语句

如果您需要更新表中的一定数量的列而不修改其他列，您可以遵循本节中给出的模式。

## 如何操作...

1.  执行 UPDATE 语句需要两份数据：

    +   **要更新的数据**：描述此类信息的一种常见方式是使用指针来表示更新的值。考虑以下示例：

        ```go
        type UpdateUserRequest struct {
          Name *string
          LastLogin *time.Time
          AvatarURL *string
        }
        ```

    在这里，只有当相应的字段不为空时，列才会被更新。例如，在以下`UpdateUserRequest`实例中，只有`LastLogin`和`AvatarURL`字段将被更新：

    ```go
    now:=time.Now()
    urlString:="https://example.org/avatar.jpg"
    update:=UpdateUserRequest {
      LastLogin: &now,
      AvatarURL: &urlString,
    }
    ```

    +   **记录定位器**：这通常是需要更新的行的唯一标识符。然而，也常见使用一个查询来定位多个记录。

    根据这些信息，编写更新函数的常见方式如下：

    ```go
    func UpdateUser(ctx context.Context, db *sql.DB, userId uint64, req *UpdateUserRequest) error {
      ...
    }
    ```

    在前面的代码中，记录定位器是`userId`。

    +   使用`strings.Builder`构建语句，同时在一个切片中跟踪查询参数：

        ```go
        query:=strings.Builder{}
        args:=make([]interface{},0)
        // Start building the query. Be mindful of spaces to separate 
        // query clauses
        query.WriteString("UPDATE users SET ")
        ```

1.  为每个需要更新的列创建一个`SET`子句：

    ```go
    if req.Name != nil {
      args=append(args,*req.Name)
      query.WriteString("user_name=?")
    }
    if req.LastLogin!=nil {
      if len(args)>0 {
        query.WriteString(",")
      }
      args=append(args,*req.LastLogin)
      query.WriteString("last_login=?")
    }
    if req.AvatarURL!=nil {
      if len(args)>0 {
        query.WriteString(",")
      }
      args=append(args,*req.AvatarURL)
      query.WriteString("avatar_url=?")
    }
    ```

1.  添加`WHERE`子句：

    ```go
    query.WriteString(" WHERE user_id=?")
    args=append(args,userId)
    ```

1.  执行该语句：

    ```go
    _,err:=db.ExecContext(ctx,query.String(),args...)
    ```

并非所有数据库驱动程序都使用`?`作为查询参数。例如，Postgres 的一个驱动程序使用`$n`，其中`n`是从 1 开始的数字，表示参数的顺序。对于此类驱动程序，算法略有不同：

```go
if req.Name != nil {
  args=append(args,*req.Name)
  fmt.Fprintf(&query,"user_name=$%d",len(args))
}
if req.LastLogin!=nil {
  if len(args)>0 {
    query.WriteString(",")
  }
  args=append(args,*req.LastLogin)
  fmt.Fprintf(&query,"last_login=$%d",len(args))
}
if req.AvatarURL!=nil {
  if len(args)>0 {
    query.WriteString(",")
  }
  args=append(args,*req.AvatarURL)
  fmt.Fprintf(&query,"avatar_url=$%d",len(args))
}
```

# 构建 WHERE 子句

`WHERE`子句可以是`SELECT`、`UPDATE`或`DELETE`语句的一部分。在这里，我将展示一个`SELECT`示例，您可以将此扩展到`UPDATE`和`DELETE`。请注意，`UPDATE`语句将包括更新列值的参数。

## 如何操作...

此示例显示了在搜索条件中使用 AND 的情况：

1.  您需要一个数据结构，以确定在`WHERE`子句中包含哪些列。以下是一个示例：

    ```go
    type UserSearchRequest struct {
      Ids            []uint64
      Name           *string
      LoggedInBefore *time.Time
      LoggedInAfter  *time.Time
      AvatarURL      *string
    }
    ```

    使用此结构，搜索函数如下所示：

    ```go
    func SearchUsers(ctx context.Context, db *sql.DB, req *UserSearchRequest) ([]User,error) {
      ...
    }
    ```

1.  使用`strings.Builder`构建语句部分，同时在一个切片中跟踪查询参数：

    ```go
    query:=strings.Builder{}
    where:= strings.Builder{}
    args:=make([]interface{},0)
    // Start building the query. Be mindful of spaces to separate 
    // query clauses
    query.WriteString("SELECT user_id, user_name, last_login, avatar_url FROM users ")
    ```

1.  为每个搜索项构建谓词：

    ```go
    if len(req.Ids)>0 {
       // Add this to the WHERE clause with an AND
       if where.Len()>0 {
          where.WriteString(" AND ")
       }
      // Build an IN clause.
      // We have to add one argument for each id
      where.WriteString("user_id IN (")
      for i,id:=range req.Ids {
        if i>0 {
          where.WriteString(",")
        }
        args=append(args,id)
        where.WriteString("?")
      }
      where.WriteString(")")
    }
    if req.Name!=nil {
      if where.Len()>0 {
        where.WriteString(" AND ")
      }
      args=append(args,*req.Name)
      where.WriteString("user_name=?")
    }
    if req.LoggedInBefore!=nil {
      if where.Len()>0 {
        where.WriteString(" AND ")
      }
      args=append(args,*req.LoggedInBefore)
      where.WriteString("last_login<?")
    }
    if req.LoggedInAfter!=nil {
      if where.Len()>0 {
        where.WriteString(" AND ")
      }
      args=append(args,*req.LoggedInAfter)
      where.WriteString("last_login>?")
    }
    if req.AvatarURL!=nil {
      if where.Len()>0 {
        where.WriteString(" AND ")
      }
      args=append(args,*req.AvatarURL)
      where.WriteString("avatar_url=?")
    }
    ```

1.  构建并运行查询：

    ```go
    if where.Len()>0 {
      query.WriteString(" WHERE ")
      query.WriteString(where.String())
    }
    rows, err:= db.QueryContext(ctx,query.String(), args...)
    ```

再次强调，并非所有数据库驱动程序都使用`?`占位符。如果您的数据库驱动程序是这些之一，请参阅上一节以获取替代方案。
