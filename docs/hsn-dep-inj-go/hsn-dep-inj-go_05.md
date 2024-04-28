# 第五章：使用猴子补丁进行依赖注入

您的代码是否依赖于全局变量？您的代码是否依赖于文件系统？您是否曾经尝试过测试数据库错误处理代码？

在本章中，我们将研究猴子补丁作为一种在测试期间*替换*依赖项的方法，并以一种其他情况下不可能的方式进行测试。无论这些依赖项是对象还是函数，我们都将应用猴子补丁到我们的示例服务中，以便我们可以将测试与数据库解耦；将不同层解耦，并且所有这些都不需要进行重大重构。

在继续我们务实、怀疑的方法时，我们还将讨论猴子补丁的优缺点。

本章将涵盖以下主题：

+   猴子魔术——猴子补丁简介

+   猴子补丁的优点

+   应用猴子补丁

+   猴子补丁的缺点

# 技术要求

熟悉我们在第四章中介绍的服务的代码将是有益的，*ACME 注册服务简介*。您可能还会发现阅读和运行本章的完整代码版本对您有所帮助，这些代码可在[`github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch05`](https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch05)中找到。

获取代码并配置示例服务的说明可在此处的 README 中找到[`github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/`](https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/)。

您可以在`ch05/acme`中找到我们服务的代码，并已应用本章的更改。

# 猴子魔术！

猴子补丁是在运行时改变程序，通常是通过替换函数或变量来实现的。

虽然这不是传统的**依赖注入**（**DI**）形式，但它可以在 Go 中用于进行测试。事实上，猴子补丁可以用于以其他方式不可能的方式进行测试。

首先让我们考虑一个现实世界的类比。假设您想测试车祸对人体的影响。您可能不会自愿成为测试期间车内的人。也不允许您对车辆进行更改以便进行测试。但是您可以在测试期间将人类换成碰撞测试假人（猴子补丁）。

在代码中进行猴子补丁的过程与现实情况相同；更改仅在测试期间存在，并且在许多情况下对生产代码的影响很小。

对于熟悉 Ruby、Python 和 JavaScript 等动态语言的人来说，有一个快速说明：可以对单个类方法进行猴子补丁，并在某些情况下对标准库进行补丁。Go 只允许我们对变量进行补丁，这可以是对象或函数，正如我们将在本章中看到的。

# 猴子补丁的优点

猴子补丁作为一种 DI 形式，在实施和效果上与本书中介绍的其他方法非常不同。因此，在某些情况下，猴子补丁是唯一的选择或唯一简洁的选择。猴子补丁的其他优点将在本节详细介绍。

**通过 monkey patching 进行 DI 的实现成本低廉**——在这本书中，我们谈论了很多关于解耦的内容，即我们的代码的各个部分应该保持独立，即使它们使用/依赖于彼此。我们引入抽象并将它们注入到彼此中。让我们退后一步，考虑一下为什么我们首先要求代码解耦。这不仅仅是为了更容易测试。它还允许代码单独演变，并为我们提供了小组，可以单独思考代码的不同部分。正是这种解耦或分离，使得 monkey patching 可以应用。

考虑这个函数：

```go
func SaveConfig(filename string, cfg *Config) error {
   // convert to JSON
   data, err := json.Marshal(cfg)
   if err != nil {
      return err
   }

   // save file
   err = ioutil.WriteFile(filename, data, 0666)
   if err != nil {
      log.Printf("failed to save file '%s' with err: %s", filename, err)
      return err
   }

   return nil
}
```

我们如何将这个函数与操作系统解耦？换个说法：当文件丢失时，我们如何测试这个函数的行为？

我们可以用`*os.File`或`io.Writer`替换文件名，但这只是把问题推到了别处。我们可以将这个函数重构为一个结构体，将对`ioutil.WriteFile`的调用改为一个抽象，然后进行模拟。但这听起来像是很多工作。

使用 monkey patching，有一个更便宜的选择：

```go
func SaveConfig(filename string, cfg *Config) error {
   // convert to JSON
   data, err := json.Marshal(cfg)
   if err != nil {
      return err
   }

   // save file
   err = writeFile(filename, data, 0666)
   if err != nil {
      log.Printf("failed to save file '%s' with err: %s", filename, err)
      return err
   }

   return nil
}

// Custom type that allows us to Monkey Patch
var writeFile = ioutil.WriteFile
```

一行代码，我们就给自己提供了用模拟替换`writeFile()`的能力，这样我们就可以轻松测试正常路径和错误场景。

**允许我们模拟其他包，而不完全了解其内部情况**——在前面的例子中，您可能已经注意到我们在模拟一个标准库函数。您知道如何使`ioutil.WriteFile()`失败吗？当然，我们可以在标准库中进行搜索；虽然这是提高 Go 技能的好方法，但这不是我们得到报酬的方式。在这种情况下，`ioutil.WriteFile()`可能会失败并不重要。真正重要的是我们的代码如何对错误做出反应。

Monkey patching，就像其他形式的模拟一样，为我们提供了不必关心依赖的内部情况，但却能让它按我们需要的方式运行的能力。

我建议从*外部*进行测试，无论如何都是正确的。解耦我们对依赖的思考方式可以确保任何测试对内部情况的了解更少，因此不容易受到实现或环境变化的影响。如果`io.WriteFile()`的内部实现细节发生任何变化，它们都不会破坏我们的测试。我们的测试只依赖于我们的代码，因此它们的可靠性完全取决于我们自己。

**通过 monkey patching 进行 DI 对现有代码的影响很小**——在前面的例子中，我们将外部依赖定义如下：

```go
var writeFile = ioutil.WriteFile
```

让我们稍微改变一下：

```go
type fileWriter func(filename string, data []byte, perm os.FileMode) error

var writeFile fileWriter = ioutil.WriteFile
```

这让你想起了什么吗？在这个版本中，我们明确地定义了我们的需求，就像我们在第二章中的*Go 的 SOLID 设计原则*部分所做的那样。虽然这种改变完全是多余的，但它确实引发了一些有趣的问题。

让我们回过头来看看，如果不使用 monkey patching 来测试我们的方法，我们需要做哪些改变。第一个选择是将`io.WriteFile`注入到函数中，如下面的代码所示：

```go
func SaveConfig(writer fileWriter, filename string, cfg *Config) error {
   // convert to JSON
   data, err := json.Marshal(cfg)
   if err != nil {
      return err
   }

   // save file
   err = writer(filename, data, 0666)
   if err != nil {
      log.Printf("failed to save file '%s' with err: %s", filename, err)
      return err
   }

   return nil
}

// This custom type is not strictly needed but it does make the function 
// signature a little cleaner
type fileWriter func(filename string, data []byte, perm os.FileMode) error
```

这有什么问题吗？就我个人而言，我对此有三个问题。首先，这是一个小而简单的函数，只有一个依赖项；如果我们有更多的依赖项，这个函数将变得非常丑陋。换句话说，代码的用户体验很糟糕。

其次，它会破坏函数实现的封装（信息隐藏）。这可能会让人觉得我在进行狂热的争论，但我并不是这样认为的。想象一下，如果我们重构`SaveConfig()`的实现，以至于我们需要将`io.WriteFile`更改为其他内容。在这种情况下，我们将不得不更改我们函数的每次使用，可能会有很多更改，因此也会有很大的风险。

最后，这种改变可以说是测试引起的伤害，正如我们在第三章的*测试引起的伤害*部分所讨论的，*用户体验编码*，因为这是一种只用于改进测试而不增强非测试代码的改变。

另一个可能会想到的选择是将我们的函数重构为一个对象，然后使用更传统的 DI 形式，如下面的代码所示：

```go
type ConfigSaver struct {
   FileWriter func(filename string, data []byte, perm os.FileMode) error
}

func (c ConfigSaver) Save(filename string, cfg *Config) error {
   // convert to JSON
   data, err := json.Marshal(cfg)
   if err != nil {
      return err
   }

   // save file
   err = c.FileWriter(filename, data, 0666)
   if err != nil {
      log.Printf("failed to save file '%s' with err: %s", filename, err)
      return err
   }

   return nil
}
```

遗憾的是，这种重构遭受了与之前相似的问题，其中最重要的是它有可能需要大量的改变。正如你所看到的，monkey patching 需要的改变明显比传统方法少得多。

**通过 monkey patching 进行 DI 允许测试全局变量和单例** - 你可能会认为我疯了，Go 语言没有单例。严格来说可能不是，但你有没有读过`math/rand`标准库包（[`godoc.org/math/rand`](https://godoc.org/math/rand)）的代码？在其中，你会发现以下内容：

```go
// A Rand is a source of random numbers.
type Rand struct {
   src Source

   // code removed
}

// Int returns a non-negative pseudo-random int.
func (r *Rand) Int() int {
   // code changed for brevity
   value := r.src.Int63()
   return int(value)
}

/*
 * Top-level convenience functions
 */

var globalRand = New(&lockedSource{})

// Int returns a non-negative pseudo-random int from the default Source.
func Int() int { return globalRand.Int() }

// A Source represents a source of uniformly-distributed
// pseudo-random int64 values in the range 0, 1<<63).
type Source interface {
   Int63() int64

   // code removed
}
```

你如何测试`Rand`结构？你可以用一个返回可预测的非随机结果的模拟来交换`Source`，很容易。

现在，你如何测试方便函数`Int()`？这并不容易。这个方法，根据定义，返回一个随机值。然而，通过 monkey patching，我们可以，如下面的代码所示：

```go
func TestInt(t *testing.T) {
   // monkey patch
   defer func(original *Rand) {
      // restore patch after use
      globalRand = original
   }(globalRand)

   // swap out for a predictable outcome
   globalRand = New(&stubSource{})
   // end monkey patch

   // call the function
   result := Int()
   assert.Equal(t, 234, result)
}

// this is a stubbed implementation of Source that returns a 
// predictable value
type stubSource struct {
}

func (s *stubSource) Int63() int64 {
   return 234
}
```

通过 monkey patching，我们能够测试单例的使用，而不需要对客户端代码进行任何更改。通过其他方法实现这一点，我们将不得不引入一层间接，这反过来又需要对客户端代码进行更改。

# 应用 monkey patching

让我们将 monkey patching 应用到我们在[第四章中介绍的 ACME 注册服务上，*ACME 注册服务简介*。我们希望通过服务改进许多事情之一是测试的可靠性和覆盖范围。在这种情况下，我们将在`data`包上进行工作。目前，我们只有一个测试，看起来是这样的：

```go
func TestData_happyPath(t *testing.T) {
   in := &Person{
      FullName: "Jake Blues",
      Phone:    "01234567890",
      Currency: "AUD",
      Price:    123.45,
   }

   // save
   resultID, err := Save(in)
   require.Nil(t, err)
   assert.True(t, resultID > 0)

   // load
   returned, err := Load(resultID)
   require.NoError(t, err)

   in.ID = resultID
   assert.Equal(t, in, returned)

   // load all
   all, err := LoadAll()
   require.NoError(t, err)
   assert.True(t, len(all) > 0)
}
```

在这个测试中，我们进行了保存，然后使用`Load()`和`LoadAll()`方法加载新保存的注册。

这段代码至少有三个主要问题。

首先，我们只测试*快乐路径*；我们根本没有测试错误处理。

其次，测试依赖于数据库。有些人会认为这没问题，我不想加入这场辩论。在这种情况下，使用实时数据库会导致我们对`LoadAll()`的测试不够具体，这使得我们的测试不如可能的彻底。

最后，我们一起测试所有的函数，而不是孤立地测试。考虑当测试的以下部分失败时会发生什么：

```go
returned, err := Load(resultID)
require.NoError(t, err)
```

问题在哪里？是`Load()`出了问题还是`Save()`出了问题？这是关于孤立测试的论点的基础。

`data`包中的所有函数都依赖于`*sql.DB`的全局实例，它代表了一个数据库连接池。因此，我们将对该全局变量进行 monkey patching，并引入一个模拟版本。

# 介绍 SQLMock

SQLMock 包（[`github.com/DATA-DOG/go-sqlmock`](https://github.com/DATA-DOG/go-sqlmock)）自述如下：

一个模拟实现 sql/driver 的模拟库。它只有一个目的 - 在测试中模拟任何 sql driver 的行为，而不需要真正的数据库连接

我发现 SQLMock 很有用，但通常比直接使用数据库更费力。作为一个务实的程序员，我很乐意使用任何一种。通常，选择使用哪种取决于我希望测试如何工作。如果我想要非常精确，没有与表的现有内容相关的问题，并且没有由表的并发使用引起的数据竞争的可能性，那么我会花额外的精力使用 SQLMock。

当两个或更多 goroutines 同时访问变量，并且至少有一个 goroutine 正在写入变量时，就会发生数据竞争。

让我们看看如何使用 SQLMock 进行测试。考虑以下函数：

```go
func SavePerson(db *sql.DB, in *Person) (int, error) {
   // perform DB insert
   query := "INSERT INTO person (fullname, phone, currency, price) VALUES (?, ?, ?, ?)"
   result, err := db.Exec(query, in.FullName, in.Phone, in.Currency, in.Price)
   if err != nil {
      return 0, err
   }

   // retrieve and return the ID of the person created
   id, err := result.LastInsertId()
   if err != nil {
      return 0, err
   }
   return int(id), nil
}
```

这个函数以`*Person`和`*sql.DB`作为输入，将人保存到提供的数据库中，然后返回新创建记录的 ID。这个函数使用传统的 DI 形式将数据库连接池传递给函数。这使我们可以轻松地用假的数据库连接替换真实的数据库连接。现在，让我们构建测试。首先，我们使用 SQLMock 创建一个模拟数据库：

```go
testDb, dbMock, err := sqlmock.New()
require.NoError(t, err)
```

然后，我们将期望的查询定义为正则表达式，并使用它来配置模拟数据库。在这种情况下，我们期望一个单独的`db.Exec`调用返回`2`，即新创建记录的 ID，以及`1`，即受影响的行：

```go
queryRegex := `\QINSERT INTO person (fullname, phone, currency, price) VALUES (?, ?, ?, ?)\E`

dbMock.ExpectExec(queryRegex).WillReturnResult(sqlmock.NewResult(2, 1))
```

现在我们调用这个函数：

```go
resultID, err := SavePerson(testDb, person)
```

然后，我们验证结果和模拟的期望：

```go
require.NoError(t, err)
assert.Equal(t, 2, resultID)
assert.NoError(t, dbMock.ExpectationsWereMet())
```

现在我们已经有了如何利用 SQLMock 来测试我们的数据库交互的想法，让我们将其应用到我们的 ACME 注册代码中。

# 使用 SQLMock 进行 monkey patching

首先，快速回顾一下：当前的`data`包不使用 DI，因此我们无法像前面的例子中那样传入`*sql.DB`。该函数当前的样子如下所示：

```go
// Save will save the supplied person and return the ID of the newly 
// created person or an error.
// Errors returned are caused by the underlying database or our connection
// to it.
func Save(in *Person) (int, error) {
   db, err := getDB()
   if err != nil {
      logging.L.Error("failed to get DB connection. err: %s", err)
      return defaultPersonID, err
   }

   // perform DB insert
   query := "INSERT INTO person (fullname, phone, currency, price) VALUES (?, ?, ?, ?)"
   result, err := db.Exec(query, in.FullName, in.Phone, in.Currency, in.Price)
   if err != nil {
      logging.L.Error("failed to save person into DB. err: %s", err)
      return defaultPersonID, err
   }

   // retrieve and return the ID of the person created
   id, err := result.LastInsertId()
   if err != nil {
      logging.L.Error("failed to retrieve id of last saved person. err: %s", err)
      return defaultPersonID, err
   }
   return int(id), nil
}
```

我们可以重构成这样，也许将来我们可能会这样做，但目前我们几乎没有对这段代码进行任何测试，而没有测试进行重构是一个可怕的想法。你可能会想到类似于*但如果我们使用 monkey patching 编写测试，然后将来进行不同风格的 DI 重构，那么我们将不得不重构这些测试*，你是对的；这个例子有点牵强。也就是说，写测试来为你提供安全保障或高水平的信心，然后以后删除它们是没有错的。这可能会感觉像是在做重复的工作，但这肯定比在一个正在运行且人们依赖的系统中引入回归，以及调试这种回归的工作要少得多。

首先引人注目的是 SQL。我们几乎需要在我们的测试中使用完全相同的字符串。因此，为了更容易地长期维护代码，我们将其转换为常量，并将其移到文件顶部。由于测试将与我们之前的例子非常相似，让我们首先仅检查 monkey patching。从之前的例子中，我们有以下内容：

```go
// define a mock db
testDb, dbMock, err := sqlmock.New()
defer testDb.Close()

require.NoError(t, err)
```

在这些行中，我们正在创建`*sql.DB`的测试实例和一个控制它的模拟。在我们可以对`*sql.DB`的测试实例进行 monkey patching 之前，我们首先需要创建原始实例的备份，以便在测试完成后进行恢复。为此，我们将使用`defer`关键字。

对于不熟悉的人来说，`defer`是一个在当前函数退出之前运行的函数，也就是说，在执行`return`语句和将控制权返回给当前函数的调用者之间。`defer`的另一个重要特性是参数会立即求值。这两个特性的结合允许我们在`defer`求值时复制原始的`sql.DB`，而不用担心当前函数何时或如何退出，从而避免了潜在的大量*清理*代码的复制和粘贴。这段代码如下所示：

```go
defer func(original sql.DB) {
   // restore original DB (after test)
   db = &original
}(*db)

// replace db for this test
db = testDb

```

完成后，测试如下所示：

```go
func TestSave_happyPath(t *testing.T) {
   // define a mock db
   testDb, dbMock, err := sqlmock.New()
   defer testDb.Close()
   require.NoError(t, err)

   // configure the mock db
   queryRegex := convertSQLToRegex(sqlInsert)
   dbMock.ExpectExec(queryRegex).WillReturnResult(sqlmock.NewResult(2, 1))

   // monkey patching starts here
   defer func(original sql.DB) {
      // restore original DB (after test)
      db = &original
   }(*db)

   // replace db for this test
   db = testDb
   // end of monkey patch

   // inputs
   in := &Person{
      FullName: "Jake Blues",
      Phone:    "01234567890",
      Currency: "AUD",
      Price:    123.45,
   }

   // call function
   resultID, err := Save(in)

   // validate result
   require.NoError(t, err)
   assert.Equal(t, 2, resultID)
   assert.NoError(t, dbMock.ExpectationsWereMet())
}
```

太棒了，我们已经完成了快乐路径测试。不幸的是，我们只测试了函数中的 13 行中的 7 行；也许更重要的是，我们不知道我们的错误处理代码是否正确工作。

# 测试错误处理

有三种可能的错误需要处理：

+   SQL 插入可能会失败

+   未能获取数据库

+   我们可能无法检索到插入记录的 ID

那么，我们如何测试 SQL 插入失败呢？使用 SQLMock 很容易：我们复制上一个测试，而不是返回`sql.Result`，我们返回一个错误，如下面的代码所示：

```go
// configure the mock db
queryRegex := convertSQLToRegex(sqlInsert)
dbMock.ExpectExec(queryRegex).WillReturnError(errors.New("failed to insert"))
```

然后我们可以将我们的期望从结果更改为错误，如下面的代码所示：

```go
require.Error(t, err)
assert.Equal(t, defaultPersonID, resultID)
assert.NoError(t, dbMock.ExpectationsWereMet())
```

接下来是测试*无法获取数据库*，这时 SQLMock 无法帮助我们，但是可以使用 monkey patching。目前，我们的`getDB()`函数如下所示：

```go
func getDB() (*sql.DB, error) {
   if db == nil {
      if config.App == nil {
         return nil, errors.New("config is not initialized")
      }

      var err error
      db, err = sql.Open("mysql", config.App.DSN)
      if err != nil {
         // if the DB cannot be accessed we are dead
         panic(err.Error())
      }
   }

   return db, nil
}
```

让我们将函数更改为变量，如下面的代码所示：

```go
var getDB = func() (*sql.DB, error) {
    // code removed for brevity
}
```

我们并没有改变函数的实现。现在我们可以对该变量进行 monkey patch，得到如下的测试结果：

```go
func TestSave_getDBError(t *testing.T) {
   // monkey patching starts here
   defer func(original func() (*sql.DB, error)) {
      // restore original DB (after test)
      getDB = original
   }(getDB)

   // replace getDB() function for this test
   getDB = func() (*sql.DB, error) {
      return nil, errors.New("getDB() failed")
   }
   // end of monkey patch

   // inputs
   in := &Person{
      FullName: "Jake Blues",
      Phone:    "01234567890",
      Currency: "AUD",
      Price:    123.45,
   }

   // call function
   resultID, err := Save(in)
   require.Error(t, err)
   assert.Equal(t, defaultPersonID, resultID)
}
```

您可能已经注意到正常路径和错误路径测试之间存在大量重复。这在 Go 语言测试中有些常见，可能是因为我们有意地重复调用一个函数，使用不同的输入或环境，从根本上来说是在为我们测试的对象记录和强制执行行为契约。

鉴于这些基本职责，我们应该努力确保我们的测试既易于阅读又易于维护。为了实现这些目标，我们可以应用 Go 语言中我最喜欢的一个特性，即表驱动测试（[`github.com/golang/go/wiki/TableDrivenTests`](https://github.com/golang/go/wiki/TableDrivenTests)）。

# 使用表驱动测试减少测试膨胀

使用表驱动测试，我们在测试开始时定义一系列场景（通常是函数输入、模拟配置和我们的期望），然后是一个场景运行器，通常是测试的一部分，否则会重复。让我们看一个例子。`Load()`函数的正常路径测试如下所示：

```go
func TestLoad_happyPath(t *testing.T) {
   expectedResult := &Person{
      ID:       2,
      FullName: "Paul",
      Phone:    "0123456789",
      Currency: "CAD",
      Price:    23.45,
   }

   // define a mock db
   testDb, dbMock, err := sqlmock.New()
   require.NoError(t, err)

   // configure the mock db
   queryRegex := convertSQLToRegex(sqlLoadByID)
   dbMock.ExpectQuery(queryRegex).WillReturnRows(
      sqlmock.NewRows(strings.Split(sqlAllColumns, ", ")).
         AddRow(2, "Paul", "0123456789", "CAD", 23.45))

   // monkey patching the database
   defer func(original sql.DB) {
      // restore original DB (after test)
      db = &original
   }(*db)

   db = testDb
   // end of monkey patch

   // call function
   result, err := Load(2)

   // validate results
   assert.Equal(t, expectedResult, result)
   assert.NoError(t, err)
   assert.NoError(t, dbMock.ExpectationsWereMet())
}
```

这个函数大约有 11 行功能代码（去除格式化后），其中大约有 9 行在我们对 SQL 加载失败的测试中几乎是相同的。将其转换为表驱动测试得到如下结果：

```go
func TestLoad_tableDrivenTest(t *testing.T) {
   scenarios := []struct {
      desc            string
      configureMockDB func(sqlmock.Sqlmock)
      expectedResult  *Person
      expectError     bool
   }{
      {
         desc: "happy path",
         configureMockDB: func(dbMock sqlmock.Sqlmock) {
            queryRegex := convertSQLToRegex(sqlLoadAll)
            dbMock.ExpectQuery(queryRegex).WillReturnRows(
               sqlmock.NewRows(strings.Split(sqlAllColumns, ", ")).
                  AddRow(2, "Paul", "0123456789", "CAD", 23.45))
         },
         expectedResult: &Person{
            ID:       2,
            FullName: "Paul",
            Phone:    "0123456789",
            Currency: "CAD",
            Price:    23.45,
         },
         expectError: false,
      },
      {
         desc: "load error",
         configureMockDB: func(dbMock sqlmock.Sqlmock) {
            queryRegex := convertSQLToRegex(sqlLoadAll)
            dbMock.ExpectQuery(queryRegex).WillReturnError(
                errors.New("something failed"))
         },
         expectedResult: nil,
         expectError:    true,
      },
   }

   for _, scenario := range scenarios {
      // define a mock db
      testDb, dbMock, err := sqlmock.New()
      require.NoError(t, err)

      // configure the mock db
      scenario.configureMockDB(dbMock)

      // monkey db for this test
      original := *db
      db = testDb

      // call function
      result, err := Load(2)

      // validate results
      assert.Equal(t, scenario.expectedResult, result, scenario.desc)
      assert.Equal(t, scenario.expectError, err != nil, scenario.desc)
      assert.NoError(t, dbMock.ExpectationsWereMet())

      // restore original DB (after test)
      db = &original
      testDb.Close()
   }
}
```

抱歉，这里有很多内容，让我们把它分成几个部分：

```go
scenarios := []struct {
   desc            string
   configureMockDB func(sqlmock.Sqlmock)
   expectedResult  *Person
   expectError     bool
}{
```

这些行定义了一个切片和一个匿名结构，它将是我们的场景列表。在这种情况下，我们的场景包含以下内容：

+   **描述**：这对于添加到测试错误消息中很有用。

+   **模拟配置**：由于我们正在测试代码如何对来自数据库的不同响应做出反应，这就是大部分魔法发生的地方。

+   **预期结果**：相当标准，考虑到输入和环境（即模拟配置）。这是我们想要得到的。

+   **一个布尔值，表示我们是否期望出现错误**：我们可以在这里使用错误值；这样会更精确。但是，我更喜欢使用自定义错误，这意味着输出不是常量。我还发现错误消息可能随时间而改变，因此检查的狭窄性使测试变得脆弱。基本上，我在测试的特定性和耐久性之间进行了权衡。

然后我们有我们的场景，每个测试用例一个：

```go
{
   desc: "happy path",
   configureMockDB: func(dbMock sqlmock.Sqlmock) {
      queryRegex := convertSQLToRegex(sqlLoadAll)
      dbMock.ExpectQuery(queryRegex).WillReturnRows(
         sqlmock.NewRows(strings.Split(sqlAllColumns, ", ")).
            AddRow(2, "Paul", "0123456789", "CAD", 23.45))
   },
   expectedResult: &Person{
      ID:       2,
      FullName: "Paul",
      Phone:    "0123456789",
      Currency: "CAD",
      Price:    23.45,
   },
   expectError: false,
},
{
  desc: "load error",
  configureMockDB: func(dbMock sqlmock.Sqlmock) {
    queryRegex := convertSQLToRegex(sqlLoadAll)
    dbMock.ExpectQuery(queryRegex).WillReturnError(
        errors.New("something failed"))
  },
  expectedResult: nil,
  expectError: true,
},

```

现在有测试运行器，基本上是对所有场景的循环：

```go
for _, scenario := range scenarios {
   // define a mock db
   testDb, dbMock, err := sqlmock.New()
   require.NoError(t, err)

   // configure the mock db
   scenario.configureMockDB(dbMock)

   // monkey db for this test
   original := *db
   db = testDb

   // call function
   result, err := Load(2)

   // validate results
   assert.Equal(t, scenario.expectedResult, result, scenario.desc)
   assert.Equal(t, scenario.expectError, err != nil, scenario.desc)
   assert.NoError(t, dbMock.ExpectationsWereMet())

   // restore original DB (after test)
   db = &original
   testDb.Close()
}
```

这个循环的内容与我们原始测试的内容非常相似。通常先编写正常路径测试，然后通过添加其他场景将其转换为表驱动测试更容易。

也许我们的测试运行器和原始函数之间唯一的区别是我们在进行 monkey patch。我们不能在`for`循环中使用`defer`，因为`defer`只有在函数退出时才会运行；因此，我们必须在循环结束时恢复数据库。

在这里使用表驱动测试不仅减少了测试代码中的重复，而且还有其他两个重要的优点。首先，它将测试简化为输入等于输出，使它们非常容易理解，也很容易添加更多的场景。

其次，可能会发生变化的代码，即函数调用本身，只存在一个地方。如果该函数被修改以接受其他输入或返回其他值，我们只需要在一个地方进行修复，而不是每个测试场景一次。

# 包之间的猴子补丁

到目前为止，我们已经看到了在我们的`data`包内部进行测试的目的而进行猴子补丁私有全局变量或函数。但是如果我们想测试其他包会发生什么呢？将业务逻辑层与数据库解耦也许是个好主意？这样可以确保我们的业务逻辑层测试不会因为无关的事件（例如优化我们的 SQL 查询）而出错。

再次，我们面临一个困境；我们可以开始大规模的重构，但正如我们之前提到的，这是一项艰巨的工作，而且风险很大，特别是没有测试来避免麻烦。让我们看看我们拥有的最简单的业务逻辑包，即`get`包：

```go
// Getter will attempt to load a person.
// It can return an error caused by the data layer or 
// when the requested person is not found
type Getter struct {
}

// Do will perform the get
func (g *Getter) Do(ID int) (*data.Person, error) {
   // load person from the data layer
   person, err := data.Load(ID)
   if err != nil {
      if err == data.ErrNotFound {
         // By converting the error we are encapsulating the 
         // implementation details from our users.
         return nil, errPersonNotFound
      }
      return nil, err
   }

   return person, err
}
```

如你所见，这个函数除了从数据库加载人员之外几乎没有做什么。因此，你可以认为它不需要存在；别担心，我们稍后会赋予它更多的责任。

那么，我们如何在没有数据库的情况下进行测试呢？首先想到的可能是像之前一样对数据库池或`getDatabase()`函数进行猴子补丁。

这样做是可行的，但会很粗糙，并且会污染`data`包的公共 API，这是测试引起的破坏的明显例子。这也不会使该包与`data`包的内部实现解耦。事实上，这会使情况变得更糟。`data`包的实现的任何更改都可能破坏我们对该包的测试。

另一个需要考虑的方面是，我们可以进行任何想要的修改，因为这项服务很小，而且我们拥有所有的代码。这通常并非如此；该包可能由另一个团队拥有，它可能是外部依赖的一部分，甚至是标准库的一部分。因此，最好养成的习惯是保持我们的更改局限于我们正在处理的包。

考虑到这一点，我们可以采用我们在上一节中简要介绍的一个技巧，即*猴子补丁的优势*。让我们拦截`get`包对`data`包的调用，如下面的代码所示：

```go
// Getter will attempt to load a person.
// It can return an error caused by the data layer or 
// when the requested person is not found
type Getter struct {
}

// Do will perform the get
func (g *Getter) Do(ID int) (*data.Person, error) {
   // load person from the data layer
   person, err := loader(ID)
   if err != nil {
      if err == data.ErrNotFound {
         // By converting the error we are hiding the 
         // implementation details from our users.
         return nil, errPersonNotFound
      }
      return nil, err
   }

   return person, err
}

// this function as a variable allows us to Monkey Patch during testing
var loader = data.Load

```

现在，我们可以通过猴子补丁拦截调用，如下面的代码所示：

```go
func TestGetter_Do_happyPath(t *testing.T) {
   // inputs
   ID := 1234

   // monkey patch calls to the data package
   defer func(original func(ID int) (*data.Person, error)) {
      // restore original
      loader = original
   }(loader)

   // replace method
   loader = func(ID int) (*data.Person, error) {
      result := &data.Person{
         ID:       1234,
         FullName: "Doug",
      }
      var resultErr error

      return result, resultErr
   }
   // end of monkey patch

   // call method
   getter := &Getter{}
   person, err := getter.Do(ID)

   // validate expectations
   require.NoError(t, err)
   assert.Equal(t, ID, person.ID)
   assert.Equal(t, "Doug", person.FullName)
}
```

现在，我们的测试不依赖于数据库或`data`包的任何内部实现细节。虽然我们并没有完全解耦这些包，但我们已经大大减少了`get`包中的测试必须正确执行的事项。这可以说是通过猴子补丁实现 DI 的一个要点，通过减少对外部因素的依赖并增加测试的焦点，减少测试可能出错的方式。

# 当魔法消失时

在本书的早些时候，我挑战你以批判的眼光审视本书中提出的每种 DI 方法。考虑到这一点，我们应该考虑猴子补丁的潜在成本。

**数据竞争**——我们在示例中看到，猴子补丁是用执行特定测试所需的方式替换全局变量的过程。这也许是最大的问题。用特定的东西替换全局的，因此是共享的，会在该变量上引发数据竞争。

为了更好地理解这种数据竞争，我们需要了解 Go 如何运行测试。默认情况下，包内的测试是按顺序执行的。我们可以通过在测试中标记`t.Parallel()`来减少测试执行时间。对于我们当前的`data`包测试，将测试标记为并行会导致数据竞争出现，从而导致测试结果不可预测。

Go 测试的另一个重要特性是，Go 可以并行执行多个包。像`t.Parallel()`一样，这对我们的测试执行时间来说可能是很棒的。通过我们当前的代码，我们可以确保安全，因为我们只在与测试相同的包内进行了猴子补丁。如果我们在包边界之间进行了猴子补丁，那么数据竞争就会出现。

如果您的测试不稳定，并且怀疑存在数据竞争，您可以尝试使用 Go 的内置竞争检测器（[`golang.org/doc/articles/race_detector.html`](https://golang.org/doc/articles/race_detector.html)）：

```go
$ go test -race ./...
```

如果这样找不到问题，您可以尝试按顺序运行所有测试：

```go
$ go test -p 1 ./...
```

如果测试开始一致通过，那么您将需要开始查找数据竞争。

**详细测试**——正如您在我们的测试中所看到的，猴子补丁和恢复的代码可能会变得相当冗长。通过一点重构，可以减少样板代码。例如，看看这个：

```go
func TestSaveConfig(t *testing.T) {
   // inputs
   filename := "my-config.json"
   cfg := &Config{
      Host: "localhost",
      Port: 1234,
   }

   // monkey patch the file writer
   defer func(original func(filename string, data []byte, perm os.FileMode) error) {
      // restore the original
      writeFile = original
   }(writeFile)

   writeFile = func(filename string, data []byte, perm os.FileMode) error {
      // output error
      return nil
   }

   // call the function
   err := SaveConfig(filename, cfg)

   // validate the result
   assert.NoError(t, err)
}
```

我们可以将其更改为：

```go
func TestSaveConfig_refactored(t *testing.T) {
   // inputs
   filename := "my-config.json"
   cfg := &Config{
      Host: "localhost",
      Port: 1234,
   }

   // monkey patch the file writer
   defer restoreWriteFile(writeFile)

   writeFile = mockWriteFile(nil)

   // call the function
   err := SaveConfig(filename, cfg)

   // validate the result
   assert.NoError(t, err)
}

func mockWriteFile(result error) func(filename string, data []byte, perm os.FileMode) error {
   return func(filename string, data []byte, perm os.FileMode) error {
      return result
   }
}

// remove the restore function to reduce from 3 lines to 1
func restoreWriteFile(original func(filename string, data []byte, perm os.FileMode) error) {
   // restore the original
   writeFile = original
}
```

在这次重构之后，我们的测试中重复的部分大大减少，从而减少了维护的工作量，但更重要的是，测试不再被所有与猴子补丁相关的代码所掩盖。

**混淆的依赖关系**——这不是猴子补丁本身的问题，而是一般依赖管理风格的问题。在传统的 DI 中，依赖关系作为参数传递，使关系显式可见。

从用户的角度来看，这种缺乏参数可以被认为是代码 UX 的改进；毕竟，更少的输入通常会使函数更容易使用。但是，当涉及测试时，事情很快变得混乱。

在我们之前的示例中，“SaveConfig（）”函数依赖于“ioutil.WriteFile（）”，因此对该依赖进行模拟以测试“SaveConfig（）”似乎是合理的。但是，当我们需要测试调用“SaveConfig（）”的函数时会发生什么？

`SaveConfig（）`的用户如何知道他们需要模拟`ioutil.WriteFile（）`？

由于关系混乱，所需的知识增加了，测试长度也相应增加；不久之后，我们在每个测试的开头就会有半屏幕的函数猴子补丁。

# 总结

在本章中，我们学习了如何利用猴子补丁来在测试中*替换*依赖关系。通过猴子补丁，我们已经测试了全局变量，解耦了包，并且消除了对数据库和文件系统等外部资源的依赖。我们通过一些实际示例来改进了我们示例服务的代码，并坦率地讨论了使用猴子补丁的优缺点。

在下一章中，我们将研究第二种，也许是最传统的 DI 技术，即构造函数注入的依赖注入。通过它，我们将进一步改进我们服务的代码。

# 问题

1.  猴子补丁是如何工作的？

1.  猴子补丁的理想用例是什么？

1.  如何使用猴子补丁来解耦两个包而不更改依赖包？

# 进一步阅读

Packt 还有许多其他关于猴子补丁的学习资源：

+   **掌握 JQuery**：[`www.packtpub.com/mapt/book/web_development/9781785882166/12/ch12lvl1sec100/monkey-patching`](https://www.packtpub.com/mapt/book/web_development/9781785882166/12/ch12lvl1sec100/monkey-patching)

+   **学习使用 Ruby 编码**：[`www.packtpub.com/mapt/video/application_development/9781788834063/40761/41000/monkey-patching-ii`](https://www.packtpub.com/mapt/video/application_development/9781788834063/40761/41000/monkey-patching-ii)
