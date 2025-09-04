# 第四章：Go 中的错误处理

在本章中，将涵盖以下配方：

+   处理错误和错误接口

+   使用 pkg/errors 包和包装错误

+   使用日志包并了解何时记录错误

+   使用 apex 和 logrus 包进行结构化日志记录

+   使用上下文包进行日志记录

+   使用包级全局变量

+   捕获长时间运行进程的恐慌

# 简介

错误处理对于最基本的 Go 程序也很重要。Go 中的错误实现了 `Error` 接口，必须在代码的每一层进行处理。Go 错误不像异常那样工作，未处理的错误可能导致巨大的问题。您应该努力在错误发生时处理和考虑错误。

本章还涵盖了日志记录，因为每当实际发生错误时，通常都会进行日志记录。我们还将研究包装错误，以便给定的错误具有适当的上下文，以便调用函数。

# 处理错误和错误接口

`Error` 接口是一个非常小且简单的接口：

```go
type Error interface{
  Error() string
}

```

这个接口很优雅，因为它简单，任何东西都可以满足它。不幸的是，这也为需要根据接收到的错误执行某些操作的包造成了混淆。

在 Go 中创建错误有多种方式，这个配方将探讨创建基本错误、具有分配的值或类型的错误以及使用结构体创建的自定义错误。

# 准备工作

根据以下步骤配置您的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 在您的操作系统上下载并安装 Go，并配置您的 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序。

1.  导航到您的 `GOPATH/src` 并创建一个项目目录，例如，`$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

# 如何做...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中创建 `chapter4/basicerrors` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter4/basicerrors`](https://github.com/agtorre/go-cookbook/tree/master/chapter4/basicerrors) 复制测试或将其用作练习编写一些自己的代码。

1.  创建一个名为 `basicerrors.go` 的文件，内容如下：

```go
        package basicerrors

        import (
            "errors"
            "fmt"
        )

        // ErrorTyped is a way to make a package level
        // error to check against. I.e. if err == TypedError
        var ErrorTyped = errors.New("this is a typed error")

        //BasicErrors demonstrates some ways to create errors
        func BasicErrors() {
            err := errors.New("this is a quick and easy way to create 
            an error")
            fmt.Println("errors.New: ", err)

            err = fmt.Errorf("an error occurred: %s", "something")
            fmt.Println("fmt.Errorf: ", err)

            err = ErrorTyped
            fmt.Println("typed error: ", err)
        }

```

1.  创建一个名为 `custom.go` 的文件，内容如下：

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
        type TypedError struct{ 
            error
        }

        //BasicErrors demonstrates some ways to create errors
        func BasicErrors() {
            err := errors.New("this is a quick and easy way to create 
            an error")
            fmt.Println("errors.New: ", err)

            err = fmt.Errorf("an error occurred: %s", "something")
            fmt.Println("fmt.Errorf: ", err)

            err = ErrorValue
            fmt.Println("value error: ", err)

            err = TypedError{errors.New("typed error")}
            fmt.Println("typed error: ", err)
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个 `main.go` 文件，内容如下。确保您修改 `basicerrors` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter4/basicerrors"
        )

        func main() {
            basicerrors.BasicErrors()

            err := basicerrors.SomeFunc()
            fmt.Println("custom error: ", err)
        }

```

1.  运行 `go run main.go`。

1.  您还可以运行：

```go
 go build ./example

```

您现在应该看到以下输出：

```go
 $ go run main.go
 errors.New: this is a quick and easy way to create an error
 fmt.Errorf: an error occurred: something
 typed error: this is a typed error
 custom error: there was an error; this was the result

```

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

无论你使用`errors.New`、`fmt.Errorf`还是自定义错误，最重要的是你绝对不应该在你的代码中留下未处理的错误。这些定义错误的不同方法提供了很大的灵活性。例如，你可以在你的结构体中添加额外的函数来进一步调查错误，并在调用函数中将接口转换为你的错误类型以获得一些附加功能。

该接口本身非常简单，唯一的要求是你返回一个有效的字符串。对于一些具有一致错误处理但希望与其他应用程序良好协作的高级应用程序，将此与结构体连接起来可能很有用。

# 使用 pkg/errors 包和错误包装

位于`github.com/pkg/errors`的`errors`包是标准 Go `errors`包的直接替代品。此外，它提供了一些非常实用的功能，用于包装和处理错误。前面菜谱中提到的类型化和声明性错误是一个很好的例子——它们可以用来向错误添加额外信息，但按照标准方式包装它将改变其类型并破坏类型断言：

```go
// this wont work if you wrapped it 
// in a standard way. i.e.
// fmt.Errorf("custom error: %s", err.Error())
if err == Package.ErrorNamed{
  //handle this error in a specific way
}

```

本菜谱将演示如何使用`pkg/errors`包在你的代码中对错误添加注释。

# 准备中

根据以下步骤配置你的环境：

1.  参考本章中“处理错误和错误接口”菜谱的“准备就绪”部分。

1.  运行`go get github.com/pkg/errors/`命令。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建名为`chapter4/errwrap`的目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter4/errwrap`](https://github.com/agtorre/go-cookbook/tree/master/chapter4/errwrap)复制测试用例，或者将其作为练习编写你自己的代码。

1.  创建一个名为`errwrap.go`的文件，并包含以下内容：

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

1.  创建一个名为`unwrap.go`的文件，并包含以下内容：

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

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个`main.go`文件，并包含以下内容。确保你修改`errwrap`导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter4/errwrap"
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

1.  你还可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下输出：

```go
 $ go run main.go
 Regular Error - An error occurred in WrappedError: standard 
      error
 Typed Error - An error occurred in WrappedError: typed error
 Nil - <nil>

 wrapped error: wrapped: an error occurred
 a typed error occurred: wrapped: an error occurred

 an error occurred
 github.com/agtorre/go-cookbook/chapter4/errwrap.StackTrace
 /Users/lothamer/go/src/github.com/agtorre/go-
      cookbook/chapter4/errwrap/unwrap.go:30
 main.main
 /tmp/go/src/github.com/agtorre/go-
      cookbook/chapter4/errwrap/example/main.go:14

```

1.  如果你复制或编写了自己的测试用例，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`pkg/errors`包是一个非常实用的工具。对于基本上每个返回的错误使用此包来提供额外的日志记录和错误调试的上下文是有意义的。它足够灵活，可以在发生错误时打印整个堆栈跟踪，或者只是在打印错误时添加一个前缀。它还可以清理代码，因为包装后的 nil 返回一个`nil`值。例如：

```go
func RetError() error{
 err := ThisReturnsAnError()
 return errors.Wrap(err, "This only does something if err != nil")
}

```

在某些情况下，这可以让你在直接返回错误之前，无需首先检查错误是否为 `nil`。本食谱演示了如何使用该包来包装和展开错误，以及基本的堆栈跟踪功能。该包的文档还提供了一些其他有用的示例，例如打印部分堆栈。本库的作者 Dave Cheney 还撰写了许多有用的博客，并就这一主题发表了演讲，请访问 [`dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully`](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully) 了解更多信息。

# 使用日志包和理解何时记录错误

日志记录通常发生在错误是最终结果时。换句话说，当发生异常或意外情况时，记录日志是有用的。如果你使用提供日志级别的日志，那么在代码的关键部分添加调试或信息语句，可以在开发过程中快速调试问题也是合适的。过多的日志记录会使找到任何有用的信息变得困难，但日志记录不足可能导致系统崩溃，无法洞察根本原因。本食谱将演示默认的 Go `log` 包的使用和一些有用的选项，并展示何时可能需要记录日志。

# 准备工作

根据以下步骤配置你的环境：

1.  参考本章中“处理错误和 Error 接口”食谱的“准备工作”部分。

1.  运行 `go get github.com/pkg/errors/` 命令。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter4/log` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter4/log`](https://github.com/agtorre/go-cookbook/tree/master/chapter4/log) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `log.go` 的文件，内容如下：

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

            logger.Printf("you can also add args(%v) and use Fataln to 
            log and crash", true)

            fmt.Println(buf.String())
        }

```

1.  创建一个名为 `error.go` 的文件，内容如下：

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

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个 `main.go` 文件，内容如下。确保你修改 `log` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter4/log"
        )

        func main() {
            fmt.Println("basic logging and modification of logger:")
            log.Log()
            fmt.Println("logging 'handled' errors:")
            log.FinalDestination()
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

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

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

你可以初始化一个日志记录器并使用 `log.NewLogger()` 将其传递，或者使用 `log` 包级别的日志记录器来记录消息。本食谱中的日志文件执行前者，而错误执行后者。它还展示了在错误达到最终目的地后何时记录日志可能是有意义的，否则你可能会为同一事件记录多次。

这种方法有几个问题。首先，你可能在一个中间函数中有额外的上下文，例如你想要记录的变量。其次，记录大量变量可能会变得混乱，难以阅读和理解。下一个菜谱将探讨提供变量记录灵活性的结构化日志记录，稍后的菜谱将探讨实现全局包级日志记录器。

# 使用 apex 和 logrus 包进行结构化日志记录

记录信息的主要原因是检查系统在事件发生或过去发生时所处的状态。当你有大量记录日志的微服务时，基本的日志消息很难整理。

如果你可以将日志转换为它们理解的数据格式，那么有各种各样的第三方包可以用来处理日志。这些包提供索引功能、可搜索性以及更多功能。`sirupsen/logrus` 和 `apex/log` 包提供了一种进行结构化日志记录的方法，你可以记录多个字段，这些字段可以被重新格式化以适应这些第三方日志读取器。例如，将日志以 JSON 格式发射以便由各种服务解析是相当简单的。

# 准备工作

根据以下步骤配置你的环境：

1.  参考菜谱 *Handling errors and the Error interface* 中的 *Getting ready* 部分。

1.  运行 `go get github.com/sirupsen/logrus` 命令。

1.  运行 `go get github.com/apex/log` 命令。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter4/structured` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter4/structured`](https://github.com/agtorre/go-cookbook/tree/master/chapter4/structured) 复制测试或使用这个练习来编写你自己的代码。

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

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个 `main.go` 文件，内容如下。确保你修改 `structured` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter4/structured"
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

1.  你也可以运行：

```go
 go build ./example

```

你现在应该看到以下输出：

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

1.  如果你复制或编写了自己的测试，请向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`sirupsen/logrus` 和 `apex/log` 包都是优秀的结构化日志记录器。它们都提供了钩子，用于向多个事件发射或向日志条目添加额外字段。例如，使用 `logrus` 钩子或 `apex` 自定义处理程序添加行号以及服务名称相对简单。钩子的另一个用途可能包括 `traceID` 以追踪跨越不同服务的一个请求。

虽然`logrus`将钩子和格式化器分开，但`apex`将它们合并。此外，`apex`还添加了一些便利函数，如`WithError`，用于添加`error`字段以及跟踪，这些都在配方中进行了演示。将`logrus`的钩子适配到`apex`处理程序也很简单。对于这两种解决方案，将转换为 JSON 格式而不是 ANSI 彩色文本只是一个简单的更改。

# 使用`context`包进行日志记录

这个配方将演示在各个函数之间传递日志字段的方法。Go 的`pkg/context`包是传递额外变量和取消操作在函数之间的一种极好方式。这个配方将探讨使用这种功能在函数之间分配变量以用于日志记录。

这种风格可以适应前面的配方中的`logrus`或`apex`。我们将使用`apex`进行这个配方。

# 准备工作

根据以下步骤配置你的环境：

1.  参考配方*Handling errors and the Error interface*中的*Getting ready*部分。

1.  运行`go get github.com/apex/log`命令。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中创建并导航到`chapter4/context`目录。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter4/context`](https://github.com/agtorre/go-cookbook/tree/master/chapter4/context)复制测试或使用这个练习来编写你自己的代码。

1.  创建一个名为`log.go`的文件，内容如下：

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

1.  创建一个名为`collect.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个包含以下内容的`main.go`文件。确保你修改`context`导入以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter4/context"

        func main() {
            context.Initialize()
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
 INFO[0000] starting id=123
 INFO[0000] after gatherName id=123 name=Go Cookbook
 INFO[0000] after gatherLocation city=Seattle id=123 name=Go 
       Cookbook state=WA

```

1.  如果你复制或编写了自己的测试，请向上导航一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`context`包现在出现在各种包中，包括数据库和 HTTP 包。这个配方将允许你将日志字段附加到上下文中，并用于日志记录目的。想法是，不同的方法可以在上下文传递过程中附加更多字段，然后最终调用点可以执行日志记录和聚合变量。

这个配方模仿了在前面配方中找到的日志包中的`WithField`和`WithFields`方法。这些方法修改上下文中存储的单个值，并提供使用上下文的其它好处：取消、超时和线程安全。

# 使用包级别的全局变量

在前面的例子中，`apex`和`logrus`包都使用了包级别的全局变量。有时，将你的库结构化以支持具有各种方法的 struct 和顶级函数是有用的，这样你就可以直接使用它们而无需传递它们。

这个配方还展示了使用 `sync.Once` 来确保全局日志器只初始化一次。它也可以通过 `Set` 方法绕过。这个配方只导出 `WithField` 和 `Debug`，但可以想象导出附加到 `log` 对象上的所有方法。

# 准备工作

按照以下步骤配置你的环境：

1.  参考本章中“处理错误和 Error 接口”配方中的“准备工作”部分。

1.  运行 `go get github.com/sirupsen/logrus` 命令。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中创建

    进入 `chapter4/global` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter4/global`](https://github.com/agtorre/go-cookbook/tree/master/chapter4/global) 复制测试或将其用作练习来编写一些你自己的代码。

1.  创建一个名为 `global.go` 的文件，内容如下：

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

        // Init sets up the logger intially
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

1.  创建一个名为 `log.go` 的文件，内容如下：

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

1.  创建一个名为 `example` 的新目录并导航到 `example`。

1.  创建一个包含以下内容的 `main.go` 文件。确保你修改 `global` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter4/global"

        func main() {
            if err := global.UseLog(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`.

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 {"key":"value","level":"debug","msg":"hello","time":"2017-02-
      12T19:22:50-08:00"}
 {"level":"debug","msg":"test","time":"2017-02-12T19:22:50-
      08:00"}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

这些全局包级对象的常见模式是保持全局未导出，并通过方法仅公开所需的功能。通常，你还可以为希望获得日志对象的项目提供一个返回全局日志器副本的方法。

`sync.Once` 类型是一种新引入的结构。这个结构，结合 `Do` 方法，将在代码中只执行一次。我们在初始化代码中使用它，如果 `Init` 被多次调用，`Init` 函数将抛出错误。

虽然这个例子使用了日志，你也可以想象在数据库连接、数据流和其他许多用例中这可能是有用的。

# 捕获长时间运行进程的恐慌

在实现长时间运行的过程时，某些代码路径可能会导致恐慌。这通常与未初始化的映射和指针以及用户输入验证不良时的除以零问题有关。

在这些情况下，程序完全崩溃通常比恐慌本身要糟糕得多，因此捕获和处理恐慌可能是有帮助的。

# 准备工作

参考本章中“处理错误和 Error 接口”配方中的“准备工作”部分。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中创建 `chapter4/panic` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter4/panic`](https://github.com/agtorre/go-cookbook/tree/master/chapter4/panic) 复制测试或将其作为练习来编写你自己的代码。

1.  创建一个名为 `panic.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `example` 的新目录，并导航到 `example` 目录。

1.  创建一个 `main.go` 文件，并包含以下内容。确保你修改 `panic` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter4/panic"
        )

        func main() {
            fmt.Println("before panic")
            panic.Catcher()
            fmt.Println("after panic")
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 before panic
 panic occurred: runtime error: integer divide by zero
 after panic

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

这个示例是一个非常基础的如何捕获恐慌的示例。你可以想象，在更复杂的中间件中，你可以在运行许多嵌套函数之后延迟恢复并捕获它。在恢复过程中，你可以基本上做任何你想做的事情，尽管输出日志是很常见的。

在大多数 Web 应用程序中，当发生恐慌时，捕获恐慌并输出一个 `http.InternalServerError` 消息是很常见的。
