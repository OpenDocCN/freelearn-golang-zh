

# 第十九章：测试

概述

测试是软件开发的一个关键方面，它确保了代码的可靠性和正确性。在 Go 中，全面的测试方法涵盖了各种类型的测试，每种测试都服务于独特的目的。本章探讨了 Go 中可用的不同测试技术和工具，以赋予开发者构建健壮和可维护应用程序的能力。

在本章结束时，你将了解 Go 开发者所实施的各类测试。我们将讨论三大测试类型：单元测试、集成测试和**端到端测试**（**E2E**）。然后，我们将介绍其他几种测试类型，例如 HTTP 测试和模糊测试。我们将涵盖测试套件、基准测试和代码覆盖率，甚至为项目利益相关者创建最终的测试报告，或者只是为了分享你的代码真正测试得有多好。你还将看到定期自动测试你的代码的好处，同时持续迭代你的代码库。这些技能对于开发生产就绪和行业级应用程序至关重要。测试也是软件开发生命周期（**SDLC**）的重要组成部分，我们作为开发者，在项目过程中会经历这一部分。

# 技术要求

对于本章，你需要 Go 版本 1.21 或更高版本。本章的代码可以在[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter19`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter19)找到。

# 简介

测试是软件开发的一个基本方面，它确保了应用程序的可靠性、正确性和稳定性。在 Go 中，测试在维护软件系统的健壮性方面发挥着关键作用。

如我之前所述，测试是软件开发生命周期（**SDLC**）的一个关键部分，它涵盖了从需求收集到部署的各个阶段。质量保证从实施有效的测试策略开始。健壮的测试不仅能够识别和纠正代码中的错误和缺陷，还能促进可维护性和可扩展性。考虑一个场景，一个关键的金融应用程序缺乏全面的测试。一个无害的代码更改可能会无意中引入一个未被注意到的错误，直到应用程序在灾难性失败时才被发现。因此，测试充当了一个安全网，在开发生命周期的早期阶段捕捉潜在的问题。如果一个团队受到一个在他们的不同环境中推出的错误的冲击，开发者应该积极创建测试来覆盖该场景，以便未来使用——无论他们使用什么语言编码。

Go 提供了一个简单而强大的内置测试框架。利用测试包，开发者可以创建单元测试、基准测试，甚至是基于示例的文档测试。测试包旨在具有表现力，这使得编写、阅读和维护测试变得容易。通过测试文件后缀（`_test.go`）和以`Test`开头的前缀清晰的测试函数签名等约定，Go 鼓励一种标准化的测试方法，这有助于在所有项目中保持一致性。值得注意的是，在 Go 中进行测试时没有`main()`函数来控制程序流程。每个测试函数都是独立且连续执行的。

在本章中，我们将深入探讨 Go 测试的细节，包括编写有效的单元测试、基准测试、表驱动测试等内容。有了对 Go 测试原则的扎实理解，开发者将能够构建可靠且可维护的软件应用程序。

# 单元测试

测试你的应用程序的一个基本方面是从单元测试开始的。单元测试专注于单个组件，验证每个函数或方法是否按预期工作。在 Go 中，单元测试使用内置的测试包编写，这使得定义测试函数和断言变得容易。

你通常会定义正例和反例单元测试。例如，如果你有一个连接几个字符串的函数，那么你会有一些正例测试用例，例如`"hi " + "sam" = "hi sam"`和`"bye," + " sam" = "bye, sam"`。你还会添加一些反例测试用例，以验证输入是否发生了错误，例如`"hi" + "there" expecting the result of "hi sam"`。这并不等价，也不是我们期望的输出，因此我们的反例测试用例期望出现错误。

你还可以考虑边缘情况，例如连接包含标点符号的字符串，并确保它们包含在连接中，并且强制执行正确的语法语法和大小写。这为测试用例提供了覆盖，你期望它们能正常工作，测试用例你期望会产生错误，以及测试覆盖你函数或方法的边缘或角落情况。

由于其惯用性，对于 Go 程序员来说，编写符合习惯和易于阅读的代码应该是自然而然的。因此，Go 采用表驱动测试结构进行所有测试，包括单元测试，这并不令人惊讶。表驱动测试保持代码可读性、灵活性和适应未来变化的能力。它们包括定义一个匿名`using`结构体，定义为*Tests*或*TestCases*，其中你包括测试用例的名称、输入或参数和预期输出。让我们看看这个例子是如何实施的。

## 练习 19.01 – 表驱动测试

让我们看看创建惯用表驱动单元测试的例子：

1.  在你的文件系统中创建一个新的文件夹，并在其中创建一个`main_test.go`文件，并编写以下代码。我们包括`gotest.tools`模块中的`assert`包，因为它为 Go 中的测试提供了实用工具和增强。具体来说，`assert`包以表达性和可读性的方式提供断言，所以它是一个很好的包：

    ```go
    package main
    import (
      "testing"
      "gotest.tools/assert"
    )
    ```

1.  创建一个函数，用于计算两个数字的和：

    ```go
    func add(x, y int) int {
      return x + y
    }
    ```

1.  定义一个表格驱动的测试函数来检查数值加法是否正确，并添加一些测试用例进行检查。我们将添加一个预期值错误的测试用例来查看其表现：

    ```go
    func TestAdd(t *testing.T) {
      tests := []struct {
        name string
        inputs []int
        want int
      }{
        {
          name: "Test Case 1",
          inputs: []int{5, 6},
          want: 11,
        },
        {
          name: "Test Case 2",
          inputs: []int{11, 7},
          want: 18,
        },
        {
          name: "Test Case 3",
          inputs: []int{1, 8},
          want: 9,
        },
        {
          name: "Test Case 4 (intentional failure)",
          inputs: []int{2, 3},
          want: 0, // This should be 5, intentionally incorrect to demonstrate failure
        },
      }
    ```

1.  遍历每个测试用例，断言接收到的值是预期的，然后关闭函数。我们同样可以使用`if`条件语句而不是`assert`包；然而，它可以使代码更加紧凑和整洁，所以你通常会看到这个包被用来断言测试值是否正确：

    ```go
        for _, test := range tests {
            got := add(test.inputs[0], test.inputs[1])
            assert.Equal(t, test.want, got)
        }
      }
    ```

1.  运行程序：

    ```go
    go test main_test.go
    ```

    你将看到以下输出：

    ```go
        main_test.go:45: assertion failed: 8 (test.want int) != 5 (got int)
    FAIL
    FAIL    command-line-arguments  0.168s
    FAIL
    ```

    如你所见，`assert`包使得在测试函数中看到失败变得容易。然而，这个输出使得知道特定的哪个测试用例失败变得有些困难。

1.  现在，如果我们修复故意错误的测试用例，我们将看到以下输出：

    ```go
    ok      command-line-arguments  0.153s
    ```

    你现在已经看到了 Go 中单元测试的表格驱动测试的样子。然而，我们可以对这个代码进行一些改进，使其更加易读——例如，我们可以调整代码，使其利用子测试。Go 中的子测试提供了几个好处：

    +   如果适用，隔离设置和清理逻辑

    +   清晰的测试输出

    +   能够添加并行执行

    +   结构化测试组织

    +   条件测试执行

    +   改进测试可读性

    现在我们已经看到了子测试的一些好处，让我们看看当它被添加到之前的单元测试函数中时的样子。为此，我们可以简单地更新 for 循环逻辑：

    ```go
    for _, test := range tests {
      test := test
      t.Run(test.name, func(t *testing.T) {
        got := add(test.inputs[0], test.inputs[1])
        assert.Equal(t, test.want, got)
      })
    }
    ```

1.  更新后再次运行程序：

    ```go
    go test main_test.go
    ```

你将看到以下输出：

```go
--- FAIL: TestAdd (0.00s)
Running tool: /usr/local/go/bin/go test -timeout 30s -run ^TestAdd$ github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Exercise19.01
--- FAIL: TestAdd (0.00s)
--- FAIL: TestAdd/Test_Case_4_(intentional_failure) (0.00s)
/Users/samcoyle/go/src/github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Exercise19.01/main_test.go:45: assertion failed: 0 (test.want int) != 5 (got int)
FAIL
FAIL github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Exercise19.01 0.164s
FAIL
```

从这里，你可以看到失败的测试输出如何通过测试用例中的`name`字段帮助我们确定哪个测试用例失败。此外，测试函数的 for 循环现在包括`test := test`。这是由于在函数字面量中使用变量在范围作用域内。如果你不包含这一行，那么 lint 器会因为函数字面量（闭包）中`test`循环变量的使用问题而抱怨。

当你在函数字面量中使用循环变量时，它会通过引用捕获循环变量。这可能导致意外的行为，因为循环变量在 for 循环的所有迭代中是共享的。为了纠正这个问题，你可以在循环内创建循环变量的局部副本，以避免通过引用捕获它，使用我们添加的行。

注意

本练习的完整代码可在[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/main/Chapter19/Example01/main_test.go`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/blob/main/Chapter19/Example01/main_test.go)找到。

我们已经看到了如何编写单元测试以确保单个组件的正确性，并在 Go 中使用表格驱动测试。你也看到了 Go 中所有测试函数都以`Test`开头，并涉及传递`testing.T`参数。你现在知道了命名测试用例、定义正负测试用例以覆盖边缘情况，以及在单元测试中注意良好实践和测试覆盖率的好处。单元测试是添加起来最容易的测试之一，因此它们应该很多，并且随着代码的增长而容易适应。

# 集成测试

集成测试验证了应用程序中不同组件或服务之间的交互。这可能包括一个服务与另一个服务交互的测试，或者一个服务与多个服务交互的测试。总的来说，这些测试确保集成系统作为一个整体正确运行。它们通常比单元测试需要更多的设置，并且实现起来需要更多的时间。这些测试旨在验证应用程序各个部分之间的协作和通信。

集成测试在确保系统的不同组件无缝协作中起着至关重要的作用。它们有助于揭示与数据流、**应用程序编程接口**（**API**）集成、数据库交互和其他协作方面相关的问题，这些问题在单元测试期间可能不明显。这类测试对于检测组件在现实场景中交互时可能出现的问题非常重要。

在设置集成测试时，你需要考虑以下几点：

+   **测试环境**：集成测试通常需要一个与生产环境相似或可能需要专用测试数据库和/或模拟服务的测试环境。

+   **数据设置**：为集成测试准备模拟真实场景的数据是常见的。这可能涉及用特定数据填充数据库或配置外部服务以使用测试用例数据。

+   `gomock`和`testify/mock`可以帮助创建用于测试的模拟。

## 练习 19.02 – 带数据库的集成测试

使用内存数据库对于集成测试来说是一个不错的选择，因为它们不会影响实时数据库。让我们看看一个模拟数据库、期望数据库发生某些事件并使用`assert`包检查我们值的练习：

1.  在你的文件系统中创建一个新的文件夹，并在其中创建一个`main_test.go`文件，并编写以下内容：

    ```go
    package main
    import (
      "context"
      "database/sql"
      "testing"
      "github.com/DATA-DOG/go-sqlmock"
      "github.com/stretchr/testify/assert"
      "github.com/stretchr/testify/require"
    )
    ```

1.  定义一个`Record`数据对象，你可以用它来检查数据库操作：

    ```go
    type Record struct {
      ID int
      Name string
      Value string
    }
    ```

1.  创建数据库结构和创建新数据库的函数：

    ```go
    type Database struct {
      conn *sql.DB
    }
    func NewDatabase(conn *sql.DB) *Database {
      return &Database{conn: conn}
    }
    ```

1.  创建数据库的插入函数：

    ```go
    func (d *Database) InsertRecord(ctx context.Context, record Record) error {
      _, err := d.conn.ExecContext(ctx, "INSERT INTO records (id, name, value) VALUES ($1, $2, $3)", record.ID, record.Name, record.Value)
      return err
    }
    ```

1.  创建一个从数据库检索插入对象的函数：

    ```go
    func (d *Database) GetRecordByID(ctx context.Context, id int) (Record, error) {
      var record Record
      row := d.conn.QueryRowContext(ctx, "SELECT id, name, value FROM records WHERE id = $1", id)
      err := row.Scan(&record.ID, &record.Name, &record.Value)
      return record, err
    }
    ```

1.  创建一个测试函数来检查与内存数据库的集成，并通过创建内存 SQL 模拟以及与数据库交互的测试记录来执行设置：

    ```go
    func TestDatabaseIntegration(t *testing.T) {
      db, mock, err := sqlmock.New()
      require.NoError(t, err)
      defer db.Close()
      testRecord := Record{
        ID: 1,
        Name: "TestRecord",
        Value: "TestValue",
      }
    ```

1.  设置 SQL 模拟数据库的期望：

    ```go
      mock.ExpectExec("INSERT INTO records").WithArgs(testRecord.ID, testRecord.Name, testRecord.Value).WillReturnResult(sqlmock.NewResult(1, 1))
      rows := sqlmock.NewRows([]string{"id", "name", "value"}).AddRow(testRecord.ID, testRecord.Name, testRecord.Value)
      mock.ExpectQuery("SELECT id, name, value FROM records").WillReturnRows(rows)
    Create the database and insert a record into it:
      dbInstance := NewDatabase(db)
      err = dbInstance.InsertRecord(context.Background(), testRecord)
      assert.NoError(t, err, "Error inserting record into the database")
    ```

1.  验证您可以从数据库中检索插入的记录，确保数据库上所有模拟的期望都得到满足，并关闭测试函数：

    ```go
      retrievedRecord, err := dbInstance.GetRecordByID(context.Background(), 1)
      assert.NoError(t, err, "Error retrieving record from the database")
      assert.Equal(t, testRecord, retrievedRecord, "Retrieved record does not match the inserted record")
      assert.NoError(t, mock.ExpectationsWereMet())
    }
    ```

1.  运行程序：

    ```go
    go test main_test.go
    ```

您将看到以下输出：

```go
  ok      command-line-arguments  0.252s
```

我们现在已经看到了如何使用模拟资源进行集成测试以及执行数据库交互的样子。这个测试检查了在内存数据库中记录的插入和检索。您可以轻松扩展此代码以检查项目可能交互的不同数据库，或者用它作为灵感来检查额外的项目交互。

# 端到端测试

在 Go 语言中，端到端测试对于评估整个系统至关重要。与关注独立代码单元的单元测试不同，或者与可能检查某些组件是否按预期合作的集成测试不同，端到端测试对整个系统进行测试，模拟真实用户场景。这些测试有助于捕捉可能由各种组件集成引起的问题，确保应用程序的整体功能。

端到端测试的目的是验证整个应用程序，包括其用户界面、API 和底层服务，是否按预期运行。这些测试模拟用户与系统交互的动作，覆盖多个层次和组件。通过测试应用程序的完整流程，端到端测试有助于识别系统中的集成问题、配置问题或可能在现实世界环境中出现的意外行为。

端到端测试有几个显著特点：

+   **现实场景**：端到端测试模拟用户流程或业务流程，确保应用程序从最终用户的角度来看表现如预期。这种现实感有助于捕捉在更隔离的测试中可能不明显的问题。

+   **多组件交互**：在典型应用中，各种组件，如数据库、API 和用户界面，共同工作以提供功能。端到端测试同时对这些组件进行测试，验证它们的交互和兼容性。

+   **与生产环境相似的环境**：端到端测试通常在接近生产设置的环境中运行。这确保了测试能够准确地反映应用程序在实际条件下的行为。

在 Go 中实现 E2E 测试时，可以使用 `testing` 和 `testing/httptest` 等工具包，这些工具包有助于有效地构建和执行 E2E 测试。这些测试通常涉及设置应用程序，以编程方式与之交互，并断言预期的结果与实际结果相匹配。

下面是一些 E2E 测试的最佳实践：

+   **隔离**：E2E 测试应在隔离环境中运行，以防止与其他测试或生产级系统发生干扰。这确保了测试结果的可靠性和一致性。

+   **自动化**：由于这些测试的复杂性，自动化至关重要。自动化的 E2E 测试可以集成到 **持续集成**（**CI**）管道中，允许定期验证应用程序的 E2E 功能。

+   **清晰的测试场景**：定义清晰且具有代表性的测试场景，涵盖关键的用户旅程。这些场景应包括用户在应用程序中采取的最常见路径。

+   **数据管理**：有效地设置和管理测试数据。这包括创建数据固定或使用数据库迁移，以确保每次测试运行的一致状态，确保一个测试不会干扰另一个测试。

E2E 测试是设置项目时最复杂的测试之一，并且完成它们需要花费最多的时间。然而，它们有助于增强对系统整体功能的信心，并且可以在开发早期阶段捕捉到集成问题。

# HTTP 测试

在 Go 中，HTTP 测试有助于验证网络服务和应用程序的行为和功能。这些测试确保 HTTP 端点对各种请求做出正确的响应，适当地处理错误，并与底层逻辑无缝交互。测试 HTTP 层对于构建健壮和可靠的应用程序至关重要，因为它允许开发者验证其 API 实现的正确性，并在开发早期阶段捕捉到问题。

HTTP 测试的重要性可以通过以下方面总结：

+   **功能验证**：HTTP 测试验证了 API 端点的功能方面，确保它们在不同场景下产生预期的响应。这包括检查状态码、响应体和头信息。

+   **集成验证**：HTTP 测试有助于验证应用程序不同组件之间的集成。它们有助于确认各种服务是否通过 HTTP 正确通信，同时遵守定义的契约。

+   **错误处理**：测试 HTTP 错误场景对于确保您的 API 对错误请求做出适当的响应至关重要。这包括测试适当的错误代码、错误消息和错误处理行为。

+   **安全保证**：HTTP 测试是安全测试的一个组成部分。它允许开发者验证认证和授权机制是否按预期工作，以及敏感信息是否得到安全处理。

Go 提供了一个强大的测试框架，使得编写 HTTP 测试变得简单直接。`net/http/httptest`包提供了一个测试服务器，它允许创建用于测试的隔离 HTTP 环境。这对于测试你的应用程序如何与外部服务或 API 交互特别有用。

## 练习 19.03 – 与测试服务器的认证集成

考虑这样一个场景，我们有一个通过 HTTP 与认证服务通信的应用程序。我们希望有一个模拟一些认证逻辑并测试主应用程序用户认证功能的认证服务测试服务器。让我们开始吧：

1.  在你的文件系统中创建一个新的文件夹，并在其中创建一个`main_test.go`文件，并编写以下内容：

    ```go
    package main
    import (
      "net/http"
      "net/http/httptest
      "testing"
      "github.com/stretchr/testify/assert"
    )
    ```

1.  定义一个`User`结构体和一个`Application`结构体，以及一个新的应用程序函数：

    ```go
    type User struct {
      UserID string
      Username string
    }
    type Application struct {
      AuthServiceURL string
    }
    func NewApplication(authServiceURL string) *Application {
      return &Application{
        AuthServiceURL: authServiceURL,
      }
    }
    ```

1.  创建一个模拟用户认证过程的函数：

    ```go
    func (app *Application) AuthenticateUser(token string) (*User, error) {
      return &User{
        UserID: "123",
        Username: "testuser",
      }, nil
    }
    ```

1.  定义测试函数并设置用于认证服务的测试服务器：

    ```go
    func TestAuthenticationIntegration(t *testing.T) {
      authService := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "Bearer valid_token" {
          w.WriteHeader(http.StatusOK)
          w.Write([]byte(`{"user_id": "123", "username": "testuser"}`))
        } else {
          w.WriteHeader(http.StatusUnauthorized)
        }
      }))
      defer authService.Close()
    ```

1.  创建应用程序，测试认证过程，然后关闭测试功能：

    ```go
      app := NewApplication(authService.URL)
      token := "valid_token"
      gotUser, err := app.AuthenticateUser(token)
      assert.NoError(t, err)
      assert.Equal(t, "123", gotUser.UserID)
      assert.Equal(t, "testuser", gotUser.Username)
    }
    ```

1.  运行程序：

    ```go
    go test main_test.go
    ```

你将看到以下输出：

```go
  ok      command-line-arguments  0.298s
```

在这个练习中，我们使用了`httptest.NewServer`函数来创建认证服务的测试服务器，使我们能够在测试期间控制其行为。然后测试设置了主应用程序，触发了认证过程，并断言了预期的结果。这有助于突出如何将 HTTP 测试集成到构建可靠的 Web 应用程序中。

# 模糊测试

模糊测试，或称模糊化，涉及向函数提供随机或格式不正确的输入以发现漏洞。换句话说，它是一种测试技术，涉及向程序提供无效、意外或随机数据作为输入，目的是发现漏洞和错误。Go 标准库中的测试包包括对模糊测试的支持，使开发者能够发现代码中的意外行为。

与传统的测试不同，传统的测试依赖于预定义的测试用例，模糊测试则广泛地探索输入空间。你可以将其视为模糊测试经常使程序承受大量随机或格式不正确的输入数据，以观察会发生什么。由于模糊测试与传统测试用例有很大不同，它们可以帮助识别边缘情况和可能触发错误或漏洞的意外输入场景。如果以自动化的方式建立，它们还可以提供持续的安全性和稳定性验证。

对于我们之前提到的`add`函数的一个简单模糊测试示例如下：

```go
func add(x, y int) int {
  return x + y
}
func FuzzAdd(f *testing.F) {
  f.Fuzz(func(t *testing.T, i int, j int) {
    got := add(i, j)
    assert.Equal(t, i + j, got)
  })
}
```

与我们之前看到的测试函数相比，这个函数有一些不同之处。例如，模糊测试函数以 *Fuzz* 开头，并传递 `f *testing.F` 而不是 `t *testing.T`。您还可以看到，由于我们不必担心生成自己的输入，与本章早期所有测试用例相比，该函数非常简单且干净。此外，我们不再使用 `t.Run` 执行，而是使用 `f.Fuzz`，它接受一个模糊目标函数以及我们常用的 `t *testing.T` 和模糊输入类型。

最后，需要注意的是，要查看模糊测试结果，必须运行以下命令：

```go
go test –fuzz .
```

这将运行测试文件中存在的模糊测试。需要注意的是，您的模糊测试可以与测试文件中的其他测试共存。您还将看到类似于以下的结果：

```go
fuzz: elapsed: 0s, gathering baseline coverage: 0/1 completed
fuzz: elapsed: 0s, gathering baseline coverage: 1/1 completed, now fuzzing with 10 workers
fuzz: elapsed: 3s, execs: 331780 (110538/sec), new interesting: 0 (total: 1)
fuzz: elapsed: 6s, execs: 709743 (126040/sec), new interesting: 0 (total: 1)
fuzz: elapsed: 9s, execs: 1123414 (137875/sec), new interesting: 0 (total: 1)
fuzz: elapsed: 12s, execs: 1417293 (97927/sec), new interesting: 0 (total: 1)
fuzz: elapsed: 15s, execs: 1713062 (98627/sec), new interesting: 0 (total: 1)
fuzz: elapsed: 18s, execs: 2076324 (121080/sec), new interesting: 0 (total: 1)
^Cfuzz: elapsed: 20s, execs: 2237555 (107504/sec), new interesting: 0 (total: 1)
PASS
ok      github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Example    19.697s
```

前面的输出中新的有趣之处在于添加到语料库中的输入数量，这些输入提供了独特的结果。*Execs* 是运行的单个测试的数量。如您所见，这比我们之前在单元测试用例中手动测试的数量大得多。模糊测试可以帮助提供大量的输入。如果您有一个失败的模糊测试，那么失败的种子语料库条目将被写入文件并放置在包目录中。然后您可以使用该信息来纠正失败。

# 基准测试

基准测试通过测量特定函数的执行时间来评估代码的性能。`testing` 包提供了对基准测试的支持，允许开发者识别性能瓶颈，确定开发者是否实现了项目的**服务级别指标**（**SLIs**）或**服务级别目标**（**SLOs**），并深入了解他们的应用程序。

与模糊测试有略微不同的语法，但遵循我们通常的 Go 测试设置期望一样，基准测试看起来也不同，但与我们习惯的非常相似。例如，基准测试以单词 *Benchmark* 开头，并接受 `b *testing.B`。测试运行器会执行每个基准函数多次，每次运行都会增加 `b.N` 的值。

这里是一个针对我们的加法函数的简单基准函数示例：

```go
func BenchmarkAdd(b *testing.B) {
  for i := 0; i < b.N; i++ {
    add(1, 2)
  }
}
```

要运行基准测试，您必须为 Go 测试框架添加基准标志：

```go
go test –bench .
```

这将给出类似于以下的结果：

```go
goos: darwin
goarch: arm64
pkg: github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Example
BenchmarkAdd-10      1000000000      0.3444 ns/op
PASS
ok      github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Example  0.538s
```

`1000000000` 的数值表示基准测试执行的迭代次数或操作数；在这种情况下，是 10 亿次操作。`0.3444 ns/op` 是每次操作的平均时间，以纳秒为单位。这表明，平均而言，每次调用 `add` 函数大约花费了 `0.3444` 纳秒。

您可以编写更复杂的基准函数，以便从代码的性能中获得更多意义，从而包括每个操作的堆分配和每个操作的字节分配。

在基准测试中，使用真实世界的输入来获取代码的最准确性能指标非常重要。你还应该将基准测试分解成多个函数，以获得更好的粒度，这将有助于你评估代码的性能。

# 测试套件

测试套件将多个相关的测试组织成一个统一的单元。Go 中的`testing`包支持使用`TestMain`函数和`testing.M`类型的测试套件。`TestMain`是一个特殊函数，可以用于执行整个测试套件的设置和清理逻辑。当你需要设置跨多个测试套件的资源或配置时，这特别有用。

## 练习 19.04 – 使用 TestMain 执行多个测试函数

考虑一个场景，你有多个测试函数想要分组到一个测试套件中。让我们看看如何使用本机 Go 测试框架来实现这一点：

1.  在你的文件系统中创建一个新的文件夹，并在其中创建一个`main_test.go`文件，并编写以下内容：

    ```go
    package main
    import (
      "log"
      "testing"
    )
    ```

1.  定义设置和清理函数：

    ```go
    func setup() {
      log.Println("setup() running")
    }
    func teardown() {
      log.Println("teardown() running")
    }
    ```

1.  创建`TestMain`函数：

    ```go
    func TestMain(m *testing.M) {
      setup()
      defer teardown()
      m.Run()
    }
    ```

1.  定义一些测试用例以运行`TestMain`：

    ```go
    func TestA(t *testing.T) {
      log.Println("TestA running")
    }
    func TestB(t *testing.T) {
      log.Println("TestB running")
    }
    func TestC(t *testing.T) {
      log.Println("TestC running")
    }
    ```

1.  以详细模式运行程序以查看`TestMain`执行我们的测试函数：

    ```go
    go test –v main_test.go
    ```

你将看到以下输出：

```go
2024/02/05 23:34:29 setup() running
=== RUN   TestA
2024/02/05 23:34:29 TestA running
--- PASS: TestA (0.00s)
=== RUN   TestB
2024/02/05 23:34:29 TestB running
--- PASS: TestB (0.00s)
=== RUN   TestC
2024/02/05 23:34:29 TestC running
--- PASS: TestC (0.00s)
PASS
2024/02/05 23:34:29 teardown() running
ok      github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Exercise20.04    0.395s
```

在这里，你可以看到`TestMain`如何执行设置，然后执行我们在包中定义的测试函数，最后通过`defer`函数执行清理。这允许你编写可能相关的多个测试函数，并将它们分组为套件一起运行。通过使用`TestMain`，你可以更好地控制整个测试套件的设置和清理过程，从而有效地处理测试套件的共享资源和配置。

# 测试报告

编写测试对于维护应用程序的正确性至关重要；然而，了解测试结果同样重要。生成测试报告可以清晰地概述测试结果。这有助于识别问题并随着时间的推移跟踪代码库的改进。Go 的`test`命令支持各种标志来定制测试输出。

Go 提供了`json`标志，可用于生成可机器读取的 JSON 输出。然后，可以使用各种工具处理或分析此输出。要生成 JSON 格式的测试报告，请运行以下命令：

```go
go test . -v -json > test-report.json
```

此命令运行测试，并将 JSON 格式的输出重定向到名为`test-report.json`的文件，使用点号表示可用的测试文件，尽管你可以指定某些测试文件而不是使用点号。生成的文件包含有关每个测试的信息，包括其名称、状态、持续时间以及任何失败消息。

然后，您可以使用各种工具使用此报告来分析和更好地可视化测试结果。然而，您必须注意，某些包可能会以未预料到的方式（或有用的方式）更改您的测试结果输出。例如，`stretchr/testify`包可以与我们在一些练习中看到的断言函数一起使用，以在测试报告中提供更有用的输出。我们关于从前一章节的练习中添加值的简单断言可以修改为在失败事件中提供清晰的输出。

假设我们有以下代码：

```go
assert.Equal(t, i+j, got)
```

这可以更新为以下内容：

```go
assert.Equal(t, i+j, got, "the values i and j should be summed together properly")
```

此外，您还可以进一步更新它，以提供更多关于`i`和`j`的哪些值未能通过测试等信息的洞察。

# 代码覆盖率

理解代码覆盖范围确保您的测试充分测试了代码库。Go 提供了一个名为`cover`的内置工具，它根据您已实施的测试用例生成测试覆盖率报告，并且可以识别。这对于单元测试来说很容易添加，这是 Go 1.20 为集成测试代码覆盖率添加的新功能。

应用程序开发的行业标准是项目力争达到 80%的代码覆盖率平均。您通常会在 CI 工具中看到覆盖率工具的使用，这些工具针对 main/master 分支运行拉取请求，以检查是否存在已测试的代码覆盖率的大幅下降。

您可以将`cover`标志添加到测试函数中，以计算测试覆盖率。还有一个标志，您可以在其中指定一个输出文件，以便从运行并收集其结果的代码覆盖率工具生成。当您的项目需要利益相关者和领导层验证项目开发中包含适当的测试代码覆盖率时，这是一个有用的报告。

让我们看看一个例子，看看这对于我们在本章中使用的`add.go`文件中的包可能是什么样子。如果您有一个包含相应`TestAdd()`函数的`add_test.go`文件，您可以使用以下命令检查代码覆盖率：

```go
go test . -cover
```

您还可以更新周期，使其指向您想要代码覆盖的具体包或目录。您应该看到典型的测试输出，然后是覆盖率数量：

```go
ok      github.com/packt-book/Go-Programming---From-Beginner-to-Professional-Second-Edition-/Chapter19/Example02        0.357s  coverage: 100.0% of statements
```

如您所见，由于我们的`add`函数有一个测试函数，所以我们有 100%的覆盖率。如果我们意外地添加了一个针对不同类型输入的加法函数，例如 float64 值，那么我们的覆盖率将下降到 50%，直到我们添加与我们的新函数相对应的测试函数。

注意

本练习的完整代码可在[`github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter19/Example02`](https://github.com/PacktPublishing/Go-Programming-From-Beginner-to-Professional-Second-Edition-/tree/main/Chapter19/Example02)找到。

一个开发团队可以使用代码覆盖率工具的结果来确定他们是否需要增加他们项目的测试用例数量，或者在项目的某些区域，因为输出文件可以展示代码覆盖率不足的区域。这是 Go 语言为开发者提供编写经过适当测试的代码的许多强大方式之一。

# 摘要

在本章中，我们探讨了 Go 语言测试的多样化领域，从单元测试到集成和端到端测试，以及一些其他类型的测试，包括 HTTP 测试和模糊测试。我们还了解了在项目代码覆盖率方面的一些测试套件和行业最佳实践。我们以我们可以对测试做的事情结束本章，包括创建测试报告以共享测试覆盖率，以及突出显示代码库性能的基准测试。

有了这些工具和技术，开发者可以确保他们编写的 Go 代码的可靠性和稳定性。在 Go 语言测试方面，我们覆盖了很多内容，但仅仅触及了 Go 工具链为我们开发者提供的功能的一角。在下一章中，我们将更深入地探讨 Go 工具及其为我们提供的功能。
