# Go 中的错误处理

即使是最基本的 Go 程序，错误处理也很重要。Go 中的错误实现了`Error`接口，并且必须在代码的每一层中处理。Go 的错误不像异常那样工作，未处理的错误可能会导致巨大的问题。您应该努力处理和考虑每当出现错误时。

本章还涵盖了日志记录，因为在实际错误发生时通常会记录日志。我们还将研究包装错误，以便给定的错误在返回到函数堆栈时提供额外的上下文，这样更容易确定某些错误的实际原因。

在本章中，将介绍以下配方：

+   处理错误和 Error 接口

+   使用 pkg/errors 包和包装错误

+   使用日志包并了解何时记录错误

+   使用 apex 和 logrus 包进行结构化日志记录

+   使用上下文包进行日志记录

+   使用包级全局变量

+   捕获长时间运行进程的 panic

# 技术要求

为了继续本章中的所有配方，请根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.12.6 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  打开终端/控制台应用程序；创建并导航到项目目录，例如`~/projects/go-programming-cookbook`。所有代码将在此目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，或者可以选择从该目录工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 处理错误和 Error 接口

`Error`接口是一个非常小且简单的接口：

```go
type Error interface{
  Error() string
}
```

这个接口很简洁，因为很容易制作任何东西来满足它。不幸的是，这也给需要根据接收到的错误采取某些操作的包带来了困惑。

在 Go 中创建错误的方法有很多种；本篇将探讨创建基本错误、具有分配值或类型的错误，以及使用结构创建自定义错误。

# 操作步骤...

以下步骤涵盖了编写和运行应用程序：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter4/basicerrors`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/basicerrors 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/basicerrors    
```

1.  从`~/projects/go-programming-cookbook-original/chapter4/basicerrors`复制测试，或者利用这个机会编写一些您自己的代码！

1.  创建一个名为`basicerrors.go`的文件，其中包含以下内容：

```go
package basicerrors

import (
  "errors"
  "fmt"
)

// ErrorValue is a way to make a package level
// error to check against. I.e. if err == ErrorValue
var ErrorValue = errors.New("this is a typed error")

// TypedError is a way to make an error type
// you can do err.(type) == ErrorValue
type TypedError struct {
  error
}

//BasicErrors demonstrates some ways to create errors
func BasicErrors() {
  err := errors.New("this is a quick and easy way to create an error")
  fmt.Println("errors.New: ", err)

  err = fmt.Errorf("an error occurred: %s", "something")
  fmt.Println("fmt.Errorf: ", err)

  err = ErrorValue
  fmt.Println("value error: ", err)

  err = TypedError{errors.New("typed error")}
  fmt.Println("typed error: ", err)

}
```

1.  创建一个名为`custom.go`的文件，其中包含以下内容：

```go
package basicerrors

import (
  "fmt"
)

// CustomError is a struct that will implement
// the Error() interface
type CustomError struct {
  Result string
}

func (c CustomError) Error() string {
  return fmt.Sprintf("there was an error; %s was the result", c.Result)
}

// SomeFunc returns an error
func SomeFunc() error {
  c := CustomError{Result: "this"}
  return c
}
```

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter4/basicerrors"
        )

        func main() {
            basicerrors.BasicErrors()

            err := basicerrors.SomeFunc()
            fmt.Println("custom error: ", err)
        }
```

1.  运行`go run main.go`。

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

您现在应该看到以下输出：

```go
$ go run main.go
errors.New: this is a quick and easy way to create an error
fmt.Errorf: an error occurred: something
typed error: this is a typed error
custom error: there was an error; this was the result
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

无论您使用`errors.New`、`fmt.Errorf`还是自定义错误，最重要的是要记住，您不应该在代码中留下未处理的错误。定义错误的这些不同方法提供了很大的灵活性。例如，您可以在结构中添加额外的函数来进一步查询错误，并在调用函数中将接口转换为您的错误类型以获得一些附加功能。

接口本身非常简单，唯一的要求是返回一个有效的字符串。将其连接到结构可能对一些高级应用程序有用，这些应用程序在整个过程中具有一致的错误处理，但希望与其他应用程序良好地配合。

# 使用 pkg/errors 包和包装错误

位于`github.com/pkg/errors`的`errors`包是标准 Go `errors`包的一个可替换项。此外，它还提供了一些非常有用的功能来包装和处理错误。前面的示例中的类型和声明的错误就是一个很好的例子——它们可以用来向错误添加额外的信息，但以标准方式包装它将改变其类型并破坏类型断言：

```go
// this wont work if you wrapped it 
// in a standard way. that is,
// fmt.Errorf("custom error: %s", err.Error())
if err == Package.ErrorNamed{
  //handle this error in a specific way
}
```

本示例将演示如何使用`pkg/errors`包在整个代码中添加注释。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter4/errwrap`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/errwrap 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/errwrap    
```

1.  从`~/projects/go-programming-cookbook-original/chapter4/errwrap`复制测试，或者将其用作练习编写自己的一些代码！

1.  创建一个名为`errwrap.go`的文件，内容如下：

```go
        package errwrap

        import (
            "fmt"

            "github.com/pkg/errors"
        )

        // WrappedError demonstrates error wrapping and
        // annotating an error
        func WrappedError(e error) error {
            return errors.Wrap(e, "An error occurred in WrappedError")
        }

        // ErrorTyped is a error we can check against
        type ErrorTyped struct{
            error
        }

        // Wrap shows what happens when we wrap an error
        func Wrap() {
            e := errors.New("standard error")

            fmt.Println("Regular Error - ", WrappedError(e))

            fmt.Println("Typed Error - ", 
            WrappedError(ErrorTyped{errors.New("typed error")}))

            fmt.Println("Nil -", WrappedError(nil))

        }
```

1.  创建一个名为`unwrap.go`的文件，内容如下：

```go
        package errwrap

        import (
            "fmt"

            "github.com/pkg/errors"
        )

        // Unwrap will unwrap an error and do
        // type assertion to it
        func Unwrap() {

            err := error(ErrorTyped{errors.New("an error occurred")})
            err = errors.Wrap(err, "wrapped")

            fmt.Println("wrapped error: ", err)

            // we can handle many error types
            switch errors.Cause(err).(type) {
            case ErrorTyped:
                fmt.Println("a typed error occurred: ", err)
            default:
                fmt.Println("an unknown error occurred")
            }
        }

        // StackTrace will print all the stack for
        // the error
        func StackTrace() {
            err := error(ErrorTyped{errors.New("an error occurred")})
            err = errors.Wrap(err, "wrapped")

            fmt.Printf("%+v\n", err)
        }
```

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个`main.go`文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter4/errwrap"
        )

        func main() {
            errwrap.Wrap()
            fmt.Println()
            errwrap.Unwrap()
            fmt.Println()
            errwrap.StackTrace()
        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

现在您应该看到以下输出：

```go
$ go run main.go
Regular Error - An error occurred in WrappedError: standard 
error
Typed Error - An error occurred in WrappedError: typed error
Nil - <nil>

wrapped error: wrapped: an error occurred
a typed error occurred: wrapped: an error occurred

an error occurred
github.com/PacktPublishing/Go-Programming-Cookbook-Second- 
Edition/chapter4/errwrap.StackTrace
/Users/lothamer/go/src/github.com/agtorre/go-
cookbook/chapter4/errwrap/unwrap.go:30
main.main
/tmp/go/src/github.com/agtorre/go-
cookbook/chapter4/errwrap/example/main.go:14
```

1.  `go.mod`文件应该已更新，顶级示例目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回到上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`pkg/errors`包是一个非常有用的工具。使用这个包来包装每个返回的错误以提供额外的上下文记录和错误调试是有意义的。当错误发生时，它足够灵活，可以打印整个堆栈跟踪，也可以在打印错误时只是添加前缀。它还可以清理代码，因为包装的 nil 返回一个`nil`值；例如，考虑以下代码：

```go
func RetError() error{
 err := ThisReturnsAnError()
 return errors.Wrap(err, "This only does something if err != nil")
}
```

在某些情况下，这可以使您免于在简单返回错误之前首先检查错误是否为`nil`。本示例演示了如何使用该包来包装和解包错误，以及基本的堆栈跟踪功能。该包的文档还提供了一些其他有用的示例，例如打印部分堆栈。该库的作者 Dave Cheney 还写了一些有用的博客并就此主题发表了一些演讲；您可以访问[`dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully`](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)了解更多信息。

# 使用日志包并了解何时记录错误

通常在错误是最终结果时应记录日志。换句话说，当发生异常或意外情况时记录日志是有用的。如果您使用提供日志级别的日志，可能还适合在代码的关键部分添加调试或信息语句，以便在开发过程中快速调试问题。过多的日志记录会使查找有用信息变得困难，但日志记录不足可能导致系统崩溃而无法了解根本原因。本示例将演示默认的 Go `log`包和一些有用的选项的使用，还展示了何时可能应该记录日志。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter4/log`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/log 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/log    
```

1.  从`~/projects/go-programming-cookbook-original/chapter4/log`复制测试，或者将其用作练习编写自己的一些代码！

1.  创建一个名为`log.go`的文件，内容如下：

```go
        package log

        import (
            "bytes"
            "fmt"
            "log"
        )

        // Log uses the setup logger
        func Log() {
            // we'll configure the logger to write
            // to a bytes.Buffer
            buf := bytes.Buffer{}

            // second argument is the prefix last argument is about 
            // options you combine them with a logical or.
            logger := log.New(&buf, "logger: ",
            log.Lshortfile|log.Ldate)

            logger.Println("test")

            logger.SetPrefix("new logger: ")

            logger.Printf("you can also add args(%v) and use Fatalln to 
            log and crash", true)

            fmt.Println(buf.String())
        }
```

1.  创建一个名为`error.go`的文件，内容如下：

```go
        package log

        import "github.com/pkg/errors"
        import "log"

        // OriginalError returns the error original error
        func OriginalError() error {
            return errors.New("error occurred")
        }

        // PassThroughError calls OriginalError and
        // forwards the error along after wrapping.
        func PassThroughError() error {
            err := OriginalError()
            // no need to check error
            // since this works with nil
            return errors.Wrap(err, "in passthrougherror")
        }

        // FinalDestination deals with the error
        // and doesn't forward it
        func FinalDestination() {
            err := PassThroughError()
            if err != nil {
                // we log because an unexpected error occurred!
               log.Printf("an error occurred: %s\n", err.Error())
               return
            }
        }
```

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为 `main.go` 的文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter4/log"
        )

        func main() {
            fmt.Println("basic logging and modification of logger:")
            log.Log()
            fmt.Println("logging 'handled' errors:")
            log.FinalDestination()
        }
```

1.  运行 `go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
basic logging and modification of logger:
logger: 2017/02/05 log.go:19: test
new logger: 2017/02/05 log.go:23: you can also add args(true) 
and use Fataln to log and crash

logging 'handled' errors:
2017/02/05 18:36:11 an error occurred: in passthrougherror: 
error occurred
```

1.  `go.mod` 文件将被更新，`go.sum` 文件现在应该存在于顶级配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

您可以初始化一个记录器并传递它使用 `log.NewLogger()`，或者使用 `log` 包级别的记录器来记录消息。这个配方中的 `log.go` 文件执行前者，`error.go` 执行后者。它还显示了在错误到达最终目的地后记录可能是有意义的时间；否则，可能会为一个事件记录多次。

这种方法存在一些问题。首先，您可能在其中一个中间函数中有额外的上下文，比如您想要记录的变量。其次，记录一堆变量可能会变得混乱，使其令人困惑和难以阅读。下一个配方将探讨提供灵活性的结构化日志记录，以记录变量，并且在以后的配方中，我们将探讨实现全局包级别记录器。

# 使用 apex 和 logrus 包进行结构化日志记录

记录信息的主要原因是在事件发生或过去发生时检查系统的状态。当有大量微服务记录日志时，基本的日志消息很难查看。

如果您可以将日志记录到它们理解的数据格式中，那么有各种第三方包可以对日志进行检索。这些包提供索引功能、可搜索性等。`sirupsen/logrus` 和 `apex/log` 包提供了一种结构化日志记录的方式，您可以记录许多字段，这些字段可以重新格式化以适应这些第三方日志读取器。例如，可以简单地以 JSON 格式发出日志，以便被各种服务解析。

# 如何做...

这些步骤涵盖了您的应用程序的编写和运行：

1.  从您的终端/控制台应用程序中，创建一个名为 `~/projects/go-programming-cookbook/chapter4/structured` 的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/structured 
```

您应该看到一个名为 `go.mod` 的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/structured    
```

1.  从 `~/projects/go-programming-cookbook-original/chapter4/structured` 复制测试，或者将其作为练习编写一些自己的代码！

1.  创建一个名为 `logrus.go` 的文件，内容如下：

```go
        package structured

        import "github.com/sirupsen/logrus"

        // Hook will implement the logrus
        // hook interface
        type Hook struct {
            id string
        }

        // Fire will trigger whenever you log
        func (hook *Hook) Fire(entry *logrus.Entry) error {
            entry.Data["id"] = hook.id
            return nil
        }

        // Levels is what levels this hook will fire on
        func (hook *Hook) Levels() []logrus.Level {
            return logrus.AllLevels
        }

        // Logrus demonstrates some basic logrus functionality
        func Logrus() {
            // we're emitting in json format
            logrus.SetFormatter(&logrus.TextFormatter{})
            logrus.SetLevel(logrus.InfoLevel)
            logrus.AddHook(&Hook{"123"})

            fields := logrus.Fields{}
            fields["success"] = true
            fields["complex_struct"] = struct {
                Event string
                When string
            }{"Something happened", "Just now"}

            x := logrus.WithFields(fields)
            x.Warn("warning!")
            x.Error("error!")
        }
```

1.  创建一个名为 `apex.go` 的文件，内容如下：

```go
        package structured

        import (
            "errors"
            "os"

            "github.com/apex/log"
            "github.com/apex/log/handlers/text"
        )

        // ThrowError throws an error that we'll trace
        func ThrowError() error {
            err := errors.New("a crazy failure")
            log.WithField("id", "123").Trace("ThrowError").Stop(&err)
            return err
        }

        // CustomHandler splits to two streams
        type CustomHandler struct {
            id string
            handler log.Handler
        }

        // HandleLog adds a hook and does the emitting
        func (h *CustomHandler) HandleLog(e *log.Entry) error {
            e.WithField("id", h.id)
            return h.handler.HandleLog(e)
        }

        // Apex has a number of useful tricks
        func Apex() {
            log.SetHandler(&CustomHandler{"123", text.New(os.Stdout)})
            err := ThrowError()

            //With error convenience function
            log.WithError(err).Error("an error occurred")
        }
```

1.  创建一个名为 `example` 的新目录并导航到该目录。

1.  创建一个名为 `main.go` 的文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter4/structured"
        )

        func main() {
            fmt.Println("Logrus:")
            structured.Logrus()

            fmt.Println()
            fmt.Println("Apex:")
            structured.Apex()
        }
```

1.  运行 `go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您现在应该看到以下输出：

```go
$ go run main.go
Logrus:
WARN[0000] warning! complex_struct={Something happened Just now} 
id=123 success=true
ERRO[0000] error! complex_struct={Something happened Just now} 
id=123 success=true

Apex:
INFO[0000] ThrowError id=123
ERROR[0000] ThrowError duration=133ns error=a crazy failure 
id=123
ERROR[0000] an error occurred error=a crazy failure
```

1.  `go.mod` 文件应该被更新，`go.sum` 文件现在应该存在于顶级配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`sirupsen/logrus` 和 `apex/log` 包都是优秀的结构化记录器。两者都提供了钩子，可以用于发出多个事件或向日志条目添加额外字段。例如，可以相对简单地使用 `logrus` 钩子或 `apex` 自定义处理程序来向所有日志添加行号以及服务名称。钩子的另一个用途可能包括 `traceID`，以跟踪请求在不同服务之间的传递。

虽然 `logrus` 将钩子和格式化器分开，但 `apex` 将它们合并在一起。除此之外，`apex` 还添加了一些方便的功能，比如 `WithError` 添加一个 `error` 字段以及跟踪，这两者都在这个配方中进行了演示。从 `logrus` 转换到 `apex` 处理程序的适配也相对简单。对于这两种解决方案，将转换为 JSON 格式，而不是 ANSI 彩色文本，将是一个简单的改变。

# 使用上下文包进行日志记录

这个配方将演示一种在各种函数之间传递日志字段的方法。Go `pkg/context`包是在函数之间传递附加变量和取消的绝佳方式。这个配方将探讨使用这个功能将变量分发到函数之间以进行日志记录。

这种风格可以从前一个配方中适应`logrus`或`apex`。我们将在这个配方中使用`apex`。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter4/context`的新目录，并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/context 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/context    
```

1.  从`~/projects/go-programming-cookbook-original/chapter4/context`中复制测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`log.go`的文件，其中包含以下内容：

```go
        package context

        import (
            "context"

            "github.com/apex/log"
        )

        type key int

        // logFields is a key we use
        // for our context logging
        const logFields key = 0

        func getFields(ctx context.Context) *log.Fields {
            fields, ok := ctx.Value(logFields).(*log.Fields)
            if !ok {
                f := make(log.Fields)
                fields = &f
            }
            return fields
        }

        // FromContext takes an entry and a context
        // then returns an entry populated from the context object
        func FromContext(ctx context.Context, l log.Interface) 
        (context.Context, *log.Entry) {
            fields := getFields(ctx)
            e := l.WithFields(fields)
            ctx = context.WithValue(ctx, logFields, fields)
            return ctx, e
        }

        // WithField adds a log field to the context
        func WithField(ctx context.Context, key string, value 
           interface{}) context.Context {
               return WithFields(ctx, log.Fields{key: value})
        }

        // WithFields adds many log fields to the context
        func WithFields(ctx context.Context, fields log.Fielder) 
        context.Context {
            f := getFields(ctx)
            for key, val := range fields.Fields() {
                (*f)[key] = val
            }
            ctx = context.WithValue(ctx, logFields, f)
            return ctx
        }
```

1.  创建一个名为`collect.go`的文件，其中包含以下内容：

```go
        package context

        import (
            "context"
            "os"

            "github.com/apex/log"
            "github.com/apex/log/handlers/text"
        )

        // Initialize calls 3 functions to set up, then
        // logs before terminating
        func Initialize() {
            // set basic log up
            log.SetHandler(text.New(os.Stdout))
            // initialize our context
            ctx := context.Background()
            // create a logger and link it to
            // the context
            ctx, e := FromContext(ctx, log.Log)

            // set a field
            ctx = WithField(ctx, "id", "123")
            e.Info("starting")
            gatherName(ctx)
            e.Info("after gatherName")
            gatherLocation(ctx)
            e.Info("after gatherLocation")
           }

           func gatherName(ctx context.Context) {
               ctx = WithField(ctx, "name", "Go Cookbook")
           }

           func gatherLocation(ctx context.Context) {
               ctx = WithFields(ctx, log.Fields{"city": "Seattle", 
               "state": "WA"})
        }
```

1.  创建一个名为`example`的新目录，并转到该目录。

1.  创建一个`main.go`文件，其中包含以下内容：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter4/context"

        func main() {
            context.Initialize()
        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
INFO[0000] starting id=123
INFO[0000] after gatherName id=123 name=Go Cookbook
INFO[0000] after gatherLocation city=Seattle id=123 name=Go 
Cookbook state=WA
```

1.  `go.mod`文件已更新，`go.sum`文件现在应该存在于顶层配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`context`包现在出现在各种包中，包括数据库和 HTTP 包。这个配方将允许您将日志字段附加到上下文中，并将它们用于日志记录目的。其思想是不同的方法可以在上下文中附加更多字段，然后最终的调用站点可以执行日志记录和聚合变量。

这个配方模仿了前一个配方中日志包中找到的`WithField`和`WithFields`方法。这些方法修改了上下文中存储的单个值，并提供了使用上下文的其他好处：取消、超时和线程安全。

# 使用包级全局变量

在之前的示例中，`apex`和`logrus`包都使用了包级全局变量。有时，将您的库结构化以支持具有各种方法和顶级函数的结构是有用的，这样您可以直接使用它们而不必传递它们。

这个配方还演示了使用`sync.Once`来确保全局记录器只初始化一次。它也可以被`Set`方法绕过。该配方只导出`WithField`和`Debug`，但您可以想象导出附加到`log`对象的每个方法。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter4/global`的新目录，并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/global 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/global    
```

1.  复制`~/projects/go-programming-cookbook-original/chapter4/global`中的测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`global.go`的文件，其中包含以下内容：

```go
        package global

        import (
            "errors"
            "os"
            "sync"

            "github.com/sirupsen/logrus"
        )

        // we make our global package level
        // variable lower case
        var (
            log *logrus.Logger
            initLog sync.Once
        )

        // Init sets up the logger initially
        // if run multiple times, it returns
        // an error
        func Init() error {
            err := errors.New("already initialized")
            initLog.Do(func() {
                err = nil
                log = logrus.New()
                log.Formatter = &logrus.JSONFormatter{}
                log.Out = os.Stdout
                log.Level = logrus.DebugLevel
            })
            return err
        }

        // SetLog sets the log
        func SetLog(l *logrus.Logger) {
            log = l
        }

        // WithField exports the logs withfield connected
        // to our global log
        func WithField(key string, value interface{}) *logrus.Entry {
            return log.WithField(key, value)
        }

        // Debug exports the logs Debug connected
        // to our global log
        func Debug(args ...interface{}) {
            log.Debug(args...)
        }
```

1.  创建一个名为`log.go`的文件，其中包含以下内容：

```go
        package global

        // UseLog demonstrates using our global
        // log
        func UseLog() error {
            if err := Init(); err != nil {
               return err
         }

         // if we were in another package these would be
         // global.WithField and
         // global.Debug
         WithField("key", "value").Debug("hello")
         Debug("test")

         return nil
        }
```

1.  创建一个名为`example`的新目录，并转到该目录。

1.  创建一个`main.go`文件，其中包含以下内容：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter4/global"

        func main() {
            if err := global.UseLog(); err != nil {
                panic(err)
            }
        }
```

1.  运行`go run main.go`。

1.  您还可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
{"key":"value","level":"debug","msg":"hello","time":"2017-02-
12T19:22:50-08:00"}
{"level":"debug","msg":"test","time":"2017-02-12T19:22:50-
08:00"}
```

1.  `go.mod`文件已更新，`go.sum`文件现在应该存在于顶层配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这些`global`包级别对象的常见模式是保持`global`变量未导出，并仅通过方法公开所需的功能。通常，你还可以包括一个方法来返回`global`日志记录器的副本，以供需要`logger`对象的包使用。

`sync.Once`类型是一个新引入的结构。这个结构与`Do`方法一起，只会在代码中执行一次。我们在初始化代码中使用这个结构，如果`Init`被调用多次，`Init`函数将抛出错误。如果我们想要向我们的`global`日志传递参数，我们使用自定义的`Init`函数而不是内置的`init()`函数。

尽管这个例子使用了日志，你也可以想象在数据库连接、数据流和许多其他用例中这可能是有用的情况。

# 捕获长时间运行进程的 panic

在实现长时间运行的进程时，可能会出现某些代码路径导致 panic 的情况。这通常是常见的情况，比如未初始化的映射和指针，以及在用户输入验证不良的情况下出现的除零问题。

在这些情况下，程序完全崩溃通常比 panic 本身更糟糕，因此捕获和处理 panic 是有帮助的。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter4/panic`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/panic 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter4/panic    
```

1.  从`~/projects/go-programming-cookbook-original/chapter4/panic`复制测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`panic.go`的文件，内容如下：

```go
        package panic

        import (
            "fmt"
            "strconv"
        )

        // Panic panics with a divide by zero
        func Panic() {
            zero, err := strconv.ParseInt("0", 10, 64)
            if err != nil {
                panic(err)
            }

            a := 1 / zero
            fmt.Println("we'll never get here", a)
        }

        // Catcher calls Panic
        func Catcher() {
            defer func() {
                if r := recover(); r != nil {
                    fmt.Println("panic occurred:", r)
                }
            }()
            Panic()
        }
```

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter4/panic"
        )

        func main() {
            fmt.Println("before panic")
            panic.Catcher()
            fmt.Println("after panic")
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
before panic
panic occurred: runtime error: integer divide by zero
after panic
```

1.  如果你复制或编写了自己的测试，那么返回上一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个示例是如何捕获 panic 的一个非常基本的例子。你可以想象使用更复杂的中间件，如何可以延迟恢复并在运行许多嵌套函数后捕获它。在恢复中，你可以做任何你想做的事情，尽管发出日志是常见的。

在大多数 Web 应用程序中，捕获 panic 并在发生 panic 时发出`http.InternalServerError`消息是很常见的。
