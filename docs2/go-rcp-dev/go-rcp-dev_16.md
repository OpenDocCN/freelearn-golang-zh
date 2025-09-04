

# 日志记录

从程序中打印日志消息可以是故障排除的重要工具。日志消息告诉您在任何给定时刻正在发生什么，并在出现问题提供所需上下文信息。Go 标准库提供了方便的包来生成和管理程序中的日志消息。在这里，我们将探讨使用`log`包，它可以用来生成文本消息，以及`slog`包，它可以用来从程序中生成结构化日志消息。

本章包含以下食谱：

+   使用标准日志记录器

    +   编写日志消息

    +   控制格式

    +   更改日志记录位置

+   使用结构化日志记录器

    +   使用全局日志记录器进行日志记录

    +   使用不同级别编写结构化日志

    +   在运行时更改日志级别

    +   使用具有附加属性的日志记录器

    +   更改日志记录位置

    +   从上下文中添加日志信息

# 使用标准日志记录器

标准库日志记录器定义在`log`包中。它是一个简单的日志库，可以用来打印格式化的日志消息，显示程序的进度。对于大多数实际用途，标准库日志记录器功能过于有限，但它可以是一个有用的工具，对于概念验证和较小的程序，它需要最少的设置。对于任何非平凡项目，请使用结构化日志记录器`log/slog`包。

## 编写日志消息

标准日志记录器是一个简单的日志实现，用于打印诊断消息。它不提供结构化输出或多个日志级别，但对于日志消息面向最终用户或开发者的程序来说可能很有用。

### 如何做到这一点...

您可以使用默认日志记录器来打印日志消息：

```go
log.Println("This is a log message similar to fmt.Println")
log.Printf("This is a log message similar to fmt.Printf")
```

这里是输出：

```go
2024/09/17 23:05:26 This is a log message similar to fmt.Println
2024/09/17 23:05:26 This is a log message similar to fmt.Printf
```

上述函数使用`log.Logger`的单例实例，可以通过`log.Default()`获取。换句话说，调用`log.Println`相当于调用`log.Default().Println`。

您还可以创建一个新的日志记录器，配置它，并将其传递出去：

```go
logger := log.New(os.Stderr, "", log.LstdFlags)
logger.Println("This is a log message written to stderr")
```

这里是输出：

```go
2024/09/17 23:10:34 This is a log message written to stderr
```

除了`log.Println`和`log.Printf`之外，您还可以使用`log.Fatal`或`log.Panic`来停止程序：

```go
log.Fatal("Fatal error")
```

这将使程序以退出代码`1`终止并输出以下内容：

```go
2024/09/17 23:05:26 Fatal error
```

我们可以通过以下内容观察到类似的情况：

```go
log.Panic("Fatal error")
```

这将引发恐慌并生成以下输出：

```go
2024/09/17 23:05:26 Fatal error
panic: Fatal error
goroutine 1 [running]:
log.Panic({0xc000104f30?, 0xc00007c060?, 0x556310?})
    /usr/local/go-faketime/src/log/log.go:432 +0x5a
main.main()
    /tmp/sandbox255937470/prog.go:8 +0x38
```

## 控制格式

您可以使用位标志来控制日志记录器的输出格式。您还可以为后续的日志消息定义一个前缀。

### 如何做到这一点...

您可以创建一个新的日志记录器，并使用以下方式为其设置前缀：

```go
logger := log.New(log.Writer(), "prefix: ", log.LstdFlags)
logger.Println("This is a log message with a prefix")
```

这将输出以下内容：

```go
prefix: 2024/09/17 23:10:34 This is a log message with a prefix
```

您还可以设置现有日志记录器的前缀：

```go
logger.SetPrefix("newPrefix")
logger.Println("This is a log message with the new prefix")
```

这里是输出：

```go
newPrefix: 2024/09/17 23:10:34 This is a log message with the new prefix
```

输出字段及其打印方式由标志控制。`log.LstdFlags`告诉日志记录器日志还应包含日期和时间。

`log.Lshortfile`打印出文件名和行号，显示日志语句的位置：

```go
logger.SetFlags(log.LstdFlags | log.Lshortfile)
logger.Println("This is a log message with a prefix and file name")
```

这将产生以下输出：

```go
prefix: 2024/09/17 23:10:34 main.go:17: This is a log message with a prefix and file name
```

`log.Llongfile`打印出完整路径：

```go
logger.SetFlags(log.LstdFlags | log.Llongfile) 
logger.Println("This is a log message with a prefix and long file name")
```

这里是输出：

```go
prefix: 2024/09/17 23:10:34 /home/github.com/PacktPublishing/Go-Recipes-for-Developers/blob/main/src/chp16/stdlogger/main.go:19: This is a log message with a prefix and long file name
```

您可以使用位运算符 `|` 或操作符组合多个标志。`log.Lmsgprefix` 将前缀字符串（如果存在）移动到消息的开始，而不是日志行的开始：

```go
logger.SetFlags(log.LstdFlags | log.Lshortfile | log.Lmsgprefix)
logger.Println("This is a log message with a prefix moved to the beginning of the message
```

这里是输出结果：

```go
2024/09/17 23:10:34 main.go:21: prefix: This is a log message with a prefix moved to the beginning of the message
```

以下标志会以 UTC 时间打印时间和日期，以及短文件名：

```go
logger.SetFlags(log.LstdFlags | log.Lshortfile | log.LUTC)
logger.Println("This is a log message with with UTC time") ```

```go

This outputs the following:

```

前缀：2024/09/18 05:10:34 main.go:23: 这是一条带有 UTC 时间的日志消息

```go

## Changing where to log

By default, the logging output goes to standard error (`os.Stderr`), but it can be changed without affecting the logging directives.

### How to do it...

You can create a logger with a given output using `log.NewLogger`. The following example creates `logger` to print its output to standard error:

```

logger := log.New(os.Stderr, "", log.LstdFlags)

```go

You can then change the logging target using `Logger.SetOutput`:

```

output, err := os.Create("log.txt")

if err != nil {

log.Fatal(err)

}

defer output.Close()

logger.SetOutput(output)

logger.Println("这是要记录到 log.txt 的日志消息")

logger.SetOutput(os.Stderr)

logger.Println("要记录到 log.txt 的消息已写入")

```go

Use `io.Discard` as the log output to stop logging:

```

logger.SetOutput(io.Discard)

logger.Println("这条消息不会被记录")

```go

# Using the structured logger

Since the standard logger has limited practical use, many third-party logging libraries were developed by the community. Some of the patterns that emerged from these libraries emphasized structured logging and performance. The structured logging package was added to the standard library with these usage patterns in mind. The `log` package is still a useful tool for development as it provides a simple interface for developers and the users of the program, but the `log/slog` package is a production quality library that enables automated log analysis tools while providing a simple-to-use and flexible interface.

## Logging using the global logger

Similar to the `log` package, there is a global structured logger accessible via the `slog.Default()` function. You can simply configure a global logger and use that in your program.

Tip

It is advisable to pass an instance of a logger around for any nontrivial project. The logging requirements may change from environment to environment, so having a dedicated logger helps.

### How to do it...

Use `slog` logging functions to write logs:

```

slog.Debug("这是一条调试信息")

slog.Info("这是一条包含整数字段的 info 信息", "arg", 42)

slog.Info("这是一条包含整数字段的 info 信息", slog.Int("arg",42))

```go

You cannot modify the settings of the default logger, but you can create a new one and set it as the default. The following example shows how you can set a JSON logger as the default logger:

```

logger := slog.New(slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{

级别：slog.LevelDebug,

},

))

slog.SetDefault(logger)

```go

Tip

`slog.SetDefault()` also sets the `log` package default logger, so the `log` package functions call the `slog` functions. Use `slog.SetLogLoggerLevel` to set the level of the log package messages.

## Writing structured logs using different levels

The structured logger allows you to log messages at different levels. For instance, you can log detailed messages at the `slog.LevelDebug` level, warning messages at the `slog.LevelWarn` level, and error messages at the `slog.LevelError` level, and set the logging level of your program from a configuration or command line argument.

### How to do it...

1.  Create a `slog.Handler` with `slog.HandlerOptions.Level` set to the desired level. The following example creates a text log handler that prints every log message as a separate line of text. It uses `os.Stderr` as the output, and the logging level is set to `slog.LevelDebug`:

    ```

    handler:= slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{

    级别：slog.LevelDebug,

    })

    ```go

2.  Create a logger using the handler:

    ```

    logger := slog.New(handler)

    ```go

3.  Use the logger to create messages at different levels. Only those messages that are equal to or above the level determined by the handler options will be printed to the output:

    ```

    logger.Debug("这是一条调试信息")

    logger.Info("这是一条包含整数字段的 info 信息", "arg", 42)

    logger.Warn("这是一条包含字符串参数的警告信息", "arg", "foo")

    ```go

4.  If logging performance is a concern, you can check whether a specific logging level is enabled:

    ```

    // 检查是否为特定级别启用了日志记录

    if logger.Enabled(context.Background(), slog.LevelError) {

    logger.Error("这是一条错误信息", slog.String("arg", "foo"))

    }

    ```go

## Changing log level at runtime

Most applications set up a logger at the beginning of the application using a command line option or a configuration file and do not change logging at runtime. However, the ability to set log levels at runtime can be an invaluable tool to identify production problems. You can set the debug level of a running server to `slog.LevelDebug`, record logs to find out about a troubling behavior, and set it back to its original level. This recipe shows how you can do this.

### How to do it...

1.  Use a `slog.LevelVar` to wrap a log level value (this is called **boxing** a variable):

    ```

    level = new(slog.LevelVar)

    ```go

2.  Set the initial log level:

    ```

    level.Set(slog.LevelError)

    ```go

3.  Create a handler using the `boxed` level:

    ```

    handler:=slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{

    级别：level,

    })

    ```go

4.  Create a logger using the handler:

    ```

    logger:=slog.New(handler)

    ```go

5.  Change `level` to control the log level:

    ```

    level.Set(slog.LevelDebug)

    // 现在所有日志记录器都将开始打印调试级别的消息

    ```go

## Using loggers with additional attributes

Let’s say you have a server where you handle requests using functions that are shared among multiple request handlers. When the request is received, you can log which handler is running, but when you pass that logger to the common functions, they lose that information. They don’t know which request handler called. Instead of passing this information to those common functions (after all, they don’t really need that information), you can decorate a logger with such information and pass the logger.

### How to do it...

1.  Create a new logger using `Logger.With`, and attach additional attributes:

    ```

    func HandlerA(w http.ResponseWriter, req *http.Request) {

    reqId:=getRequestIdFromRequest(req)

    // 创建一个新的具有附加属性的日志记录器

    logger:=slog.With(slog.String("handler", "a"),slog.

    String("reqId",reqId))

    logger.Debug("开始处理请求")

    defer logger.Debug("请求完成")

    ```go

2.  Use this logger to log messages:

    ```

    HandleRequest(logger, w,req)

    ```go

    This will output a log message that looks like this:

    ```

    {"time":"2024-09-19T14:49:42.064787730-06:00","level":"DEBUG","msg":"开始处理请求","handler":"a","reqId":"123"}

    {"time":"2024-09-19T14:49:42.308187758-06:00","level":"DEBUG","msg":"这是一条调试信息","handler":"a","reqId":"123"}

    {"time":"2024-09-19T14:49:42.945674637-06:00","level":"DEBUG","msg":"请求完成","handler":"a","reqId":"123"}

    ```go

## Changing where to log

The default logger writes to `os.Stderr`, and similar to the `log` package, this can be changed when you create the logger.

### How to do it...

The logger output is determined by the `slog.Handler`. The following example creates `logger` to print its output to standard error:

```

logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{

级别：slog.LevelDebug,

}))

```go

Unlike the `log` package, you cannot change where to log after creating a logger, unless you write your own handler.

## Adding logging information from the context

Often, the information you need to log is available in the context. Every `slog` logging function has two variants, one with context and one without. If you use the variants with context, you can write a handler that can extract information from that context containing information from the call site.

### How to do it...

Create a new handler, potentially wrapping an existing one. The following code snippet shows a handler that will extract an `id` from the context by wrapping a `slog.Handler`:

```

type ContextIDHandler struct {

slog.Handler

}

```go

Define the `Handle` method. Extract information from the context, modify the log record, and pass it to the wrapped handler:

```

func (h ContextIDHandler) Handle(ctx context.Context, r slog.Record) error {

// 如果上下文有一个字符串 id，检索它并将其添加到

// 记录

if id, ok := ctx.Value("id").(string); ok {

r.Add(slog.String("id", id))

}

return h.Handler.Handle(ctx, r)

}

```go

Use the logging functions that take `context.Context`:

```

func Handler(w http.ResponseWriter, req *http.Request) {

logger.Debug(req.Context(),"处理器启动")

...

```go

This will add the `id` from the request context to the log message if there is one:

```

{"时间":"2024-09-19T15:02:12.163787730-06:00","级别":"DEBUG","信息":"处理器启动","ID":"123"}

```
