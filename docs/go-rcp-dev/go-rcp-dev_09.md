

# 第九章：上下文包

上下文是某事发生的情境。当我们谈论程序时，上下文是程序环境、设置等。对于服务器程序（响应客户端请求的 HTTP 服务器、响应函数调用的 RPC 服务器等）或响应用户请求的程序（交互式程序、命令行应用程序等），你可以谈论特定请求的上下文。特定请求的上下文是在服务器或程序开始处理特定请求时创建的，在处理结束时终止。请求上下文包含有助于你在处理请求时识别日志消息的请求标识符等信息，或者调用者的身份，以便你可以确定调用者的访问权限。`context`包的一个用途是提供此类请求上下文的抽象，即保持请求特定数据的对象。

你可能还对请求的运行时间有所顾虑。你通常希望限制请求处理的时间，或者你可能想检测客户端是否不再对请求的结果感兴趣（例如 WebSocket 对等端断开连接）。`context`包旨在处理这些用例。

`context`包定义了`context.Context`接口。它有两个用途：

+   为请求处理添加超时和/或取消

+   将请求特定的元数据传递到堆栈中

`context.Context`的使用不仅限于服务器程序。术语“请求处理”应从广义上理解：请求可以是 TCP 连接的网络请求、HTTP 请求、从命令行读取的命令、以特定标志运行程序等。因此，`context.Context`的用途更加多样化。

本章展示了`context`包的常见用法。在本章中，你将了解以下内容：

+   使用上下文传递请求作用域的数据

+   使用上下文进行取消和超时

# 使用上下文传递请求作用域的数据

请求作用域的对象是在请求处理开始时创建的，在请求处理结束时丢弃的对象。这些通常是轻量级对象，例如请求标识符、标识调用者的认证信息或记录器。在本节中，你将了解如何使用上下文传递这些对象。

## 如何实现...

向上下文中添加数据值的惯用方法是以下：

1.  定义上下文键类型。这可以避免意外的名称冲突。使用以下未导出类型名称是常见的做法。这种模式限制了将此特定类型的上下文值放入或从当前包中获取的能力：

    ```go
    type requestIDKeyType int
    ```

警告

你可能会想在这里使用 `struct{}` 而不是 `int`。毕竟，`struct{}` 不消耗任何额外的内存。当你与 0 大小结构工作时必须非常小心，因为 Go 语言规范没有提供关于两个 0 大小结构等价性的任何保证。也就是说，如果你创建了多个 0 大小类型的变量，它们有时可能相等，有时可能不相等。简而言之，不要使用 `struct{}`。

1.  使用键类型定义键值，或值。在以下代码行中，`requestIDKey` 被定义为 `requestIDKeyType` 类型，其值为 `0`（`requestIDKey` 在声明时初始化为其 `0` 值）：

    ```go
    var requestIDKey requestIDKeyType
    ```

1.  使用 `context.WithValue` 将新值添加到上下文中。你可以定义一些辅助函数来设置和获取上下文中的值：

    ```go
    func WithRequestID(ctx context.Context,requestID string) context.Context {
      return context.WithValue(ctx,requestIDKey,requestID)
    }
    func GetRequestID(ctx context.Context) string {
      id,_:=ctx.Value(requestIDKey).(string)
      return id
    }
    ```

1.  将新上下文传递给从当前函数调用的函数：

    ```go
    newCtx:=WithRequestID(ctx,requestID)
    handleRequest(newCtx)
    ```

## 它是如何工作的...

你可能已经注意到，`context.Context` 并不完全像一个键值映射（没有 `SetValue` 方法；实际上，`context.Context` 是不可变的），尽管你可以用它来存储键值对。实际上，你不能向上下文中添加键值，但你可以在保持旧上下文的同时获取包含该键值的新上下文。上下文就像洋葱一样有层级；向上下文中添加的每个元素都会创建一个新的上下文，它与旧上下文相连，但具有更多功能：

```go
// ctx: An empty context
ctx := context.Background()
// ctx1: ctx + {key1:value1}
ctx1 := context.WithValue(ctx, "key1", "value1")
// ctx2: ctx1 + {key2:value2}
ctx2 := context.WithValue(ctx, "key2", "value2")
```

在前面的代码中，`ctx`、`ctx1` 和 `ctx2` 是三个不同的上下文。`ctx` 上下文为空。`ctx1` 包含 `ctx` 和 `key1: value1` 键值对。`ctx2` 包含 `ctx1` 和 `key2: value2` 键值对。所以，如果你这样做：

```go
val1,_ := ctx2.Value("key1")
val2,_ := ctx2.Value("key2")
fmt.Println(val1, val2)
```

这将打印以下内容：

```go
value1 value2
```

假设你对 `ctx1` 做同样的操作：

```go
val1,_ = ctx1.Value("key1")
val2,_ = ctx1.Value("key2")
fmt.Println(val1, val2)
```

这将打印以下内容：

```go
value1 <nil>
```

以下用于 `ctx`：

```go
val1,_ = ctx.Value("key1")
val2,_ = ctx.Value("key2")
fmt.Println(val1, val2)
```

这将打印以下内容：

```go
<nil> <nil>
```

小贴士

即使你无法在上下文中设置值（也就是说，上下文是不可变的），你仍然可以设置一个指向结构的指针，并在该结构中设置值。

也就是说：

```go
type ctxData struct {
  value int
}
...
ctx:=context.WithValue(context.Background(),dataKey, &ctxData{})
...
if data,exists:=ctx.Value(dataKey); exists {
  data.(*ctxData).value=1
}
```

标准库提供了一些预定义的上下文值：

+   `context.Background()` 返回一个没有值且无法取消的上下文。这通常是大多数操作的基础上下文。

+   `context.TODO()` 与 `context.Background()` 类似，其名称表明在任何使用它的地方最终都应该重构以接受真实上下文。

## 还有更多...

上下文通常在多个 goroutine 之间共享。你必须小心并发问题，特别是如果你在上下文中放置对象的指针。看看以下示例，它展示了 HTTP 服务的身份验证中间件：

```go
type AuthInfo struct {
  // Set when AuthInfo is created
  UserID string
  // Lazy-initialized
  privileges map[string]Privilege
}
type authInfoKeyType int
var authInfoKey authInfoKeyType
// Set the privileges if is it not initialized.
// Do not do this!!
func (auth *AuthInfo) GetPrivileges() map[string]Privilege {
   if auth.privileges==nil {
      auth.privileges=GetPrivileges(auth.UserID)
   }
   return auth.privileges
}
// Authentication middleware
func AuthMiddleware(next http.Handler) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.
        Request) {
            // Authenticate the caller
            var authInfo *AuthInfo
            var err error
            authInfo, err = authenticate(r)
            if err != nil {
                http.Error(w, err.Error(), http.StatusUnauthorized)
                return
            }
            // Create a new context with the authentication info
            newCtx := context.WithValue(r.Context(), authInfoKey, 
            authInfo)
            // Pass the new context to the next handler
            next.ServeHTTP(w, r.WithContext(newCtx))
        })
    }
}
```

认证中间件创建了一个 `*AuthInfo` 实例，并使用带有认证信息的上下文调用链中的下一个处理器。在这段代码中存在的问题是 `*AuthInfo` 包含一个 `privileges` 字段，它在调用 `AuthInfo.GetPrivileges` 时被初始化。由于上下文可以通过处理器传递给多个 goroutines，这种延迟初始化方案容易发生数据竞争；多个 goroutines 调用 `AuthInfo.GetPrivileges` 可能会尝试多次初始化映射，一次覆盖另一次。

可以使用互斥锁来纠正这个问题：

```go
type AuthInfo struct {
  sync.Mutex
  UserID string
  privileges map[string]Privilege
}
func (auth *AuthInfo) GetPrivileges() map[string]Privilege {
   // Use mutex to initialize the privileges in a thread-safe way
   auth.Lock()
   defer auth.Unlock()
   if auth.privileges==nil {
      auth.privileges=GetPrivileges(auth.UserID)
   }
   return auth.privileges
}
```

这也可以通过在中间件中一次性初始化权限来纠正：

```go
     authInfo, err=authenticate(r)
     if err!=nil {
       http.Error(w,err.Error(),http.StatusUnauthorized)
       return
     }
     // Initialize the privileges here when the structure is created
     authInfo.GetPrivileges()
```

# 使用上下文进行取消操作

你可能想要取消计算的原因有几个：客户端可能已经断开连接，或者你可能有多条 goroutines 正在处理一个计算，其中一条失败了，因此你不再希望其他继续。你可以使用其他方法，例如关闭一个 `done` 通道来发出取消信号，但根据使用情况，上下文可能更方便。上下文可以被取消多次（只有第一次调用实际上会取消；其余的将被忽略），而你不能关闭已经关闭的通道，因为这会导致 panic。此外，你可以创建一个上下文树，其中取消一个上下文只会取消它控制的 goroutines，而不会影响其他 goroutines。

## 如何做到...

这些是创建可取消上下文和检测取消的步骤：

1.  使用 `context.WithCancel` 基于现有上下文和一个取消函数创建一个新的可取消上下文：

    ```go
    ctx:=context.Background()
    cancelable, cancel:=context.WithCancel(ctx)
    defer cancel()
    ```

    确保最终调用了 `cancel` 函数。取消释放了与上下文关联的资源。

1.  将可取消的上下文传递给可以取消的计算或 goroutines：

    ```go
    go cancelableGoroutine1(cancelable)
    go cancelableGoroutine2(cancelable)
    cancelableFunc(cancelable)
    ```

1.  在取消函数中，使用 `ctx.Done()` 通道或 `ctx.Err()` 检查上下文是否被取消：

    ```go
    func cancelableFunc(ctx context.Context) {
      // Process some data
      // Check context cancelation
      select {
         case <-ctx.Done():
            // Context canceled
            return
         default:
      }
      // Continue computation
    }
    ```

    或者，使用以下方法：

    ```go
     func cancelableFunc(ctx context.Context) {
       // Process some data
       // Check context cancelation
       if ctx.Err()!=nil {
           // Context canceled
           return
       }
       // Continue computation
    }
    ```

1.  要手动取消一个函数，调用取消函数：

    ```go
    ctx:=context.Background()
    cancelable, cancel:=context.WithCancel(ctx)
    defer cancel()
    wg:=sync.WaitGroup{}
    wg.Add(1)
    go cancelableGoroutine1(cancelable,&wg)
    if err:=process(ctx); err!=nil {
       // Cancel the context
       cancel()
       // Do other things
    }
    wg.Wait()
    ```

1.  确保最终调用了 `cancel` 函数（使用 `defer cancel()`）：

```go
cancelable, cancel := context.WithCancel(ctx)
defer cancel()
...
```

警告

确保调用 `cancel` 是很重要的。如果你没有取消一个可取消的上下文，与该上下文关联的 goroutines 将会泄漏（即，将无法终止这些 goroutines，它们将消耗内存）。

提示

`cancel` 函数可以被多次调用。后续的调用将被忽略。

## 它是如何工作的...

`context.WithCancel` 返回一个新的上下文和 `cancel` 闭包。返回的上下文是基于原始上下文的一个可取消上下文：

```go
// Empty context, no cancelation
originalContext := context.Background()
// Cancelable context based on originalContext
cancelableContext1, cancel1 := context.WithCancel(originalContext)
```

你可以使用这个上下文来控制多个 goroutines：

```go
go f1(cancelableContext1)
go f2(cancelableContext1)
```

你也可以基于一个可取消的上下文创建其他可取消的上下文：

```go
cancelableContext2, cancel2 := context.WithCancel(cancelableContext)
go g1(cancelableContext2)
go g2(cancelableContext2)
```

现在，我们有两个可取消的上下文。调用 `cancel2` 将只会取消 `cancelableContext2`：

```go
cancal2() // canceling g1 and g2 only
```

调用 `cancel1` 将会取消 `cancelableContext1` 和 `cancelableContext2`：

```go
cancel1() // canceling f1, f2, g1, g2
```

上下文取消不是自动取消 goroutines 的方式。您必须检查上下文取消并进行相应的清理：

```go
func f1(cancelableContext context.Context) {
   for {
      if cancelableContext.Err()!=nil {
         // Context is canceled
         // Cleanup and return
         return
      }
      // Process
   }
}
```

# 使用上下文进行超时

超时简单来说就是自动取消。上下文将在计时器到期后取消。这在限制不太可能完成的计算的资源消耗时很有用。

## 如何做到这一点...

创建具有超时并检测超时事件发生的步骤如下：

1.  使用 `context.WithTimeout` 创建一个新的可取消上下文，该上下文将在给定持续时间后自动取消，基于现有上下文和取消函数：

    ```go
    ctx:=context.Background()
    timeoutable, cancel:=context.WithTimeout(ctx,5*time.Second)
    defer cancel()
    ```

    或者，您可以使用 `WithDeadline` 在指定时刻取消上下文。

    确保最终调用 `cancel` 函数。

1.  将超时上下文传递给可能超时的计算或 goroutine：

    ```go
    go longRunningGoroutine1(timeoutable)
    go longRunningGoroutine2(timeoutable)
    ```

1.  在 goroutine 中，使用 `ctx.Done()` 通道或 `ctx.Err()` 检查上下文是否被取消：

    ```go
    func longRunningGoroutine(ctx context.Context) {
      // Process some data
      // Check context cancelation
      select {
         case <-ctx.Done():
            // Context canceled
            return
         default:
      }
      // Continue computation
    }
    ```

    或者，使用以下方法：

    ```go
     func cancelableFunc(ctx context.Context) {
       // Process some data
       // Check context cancelation
       if ctx.Err()!=nil {
           // Context canceled
           return
       }
       // Continue computation
    }
    ```

1.  要手动取消函数，请调用取消函数：

    ```go
    ctx:=context.Background()
    timeoutable, cancel:=context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    wg:=sync.WaitGroup{}
    wg.Add(1)
    go longRunningGoroutine(timeoutable,&wg)
    if err:=process(ctx); err!=nil {
       // Cancel the context
       cancel()
       // Do other things
    }
    wg.Wait()
    ```

1.  确保最终调用 `cancel` 函数（使用 `defer cancel()`）：

    ```go
    timeoutable, cancel := context.WithTimeout(ctx,5*time.Second)
    defer cancel()
    ...
    ```

## 它是如何工作的...

超时功能简单来说就是附加了计时器的取消。当计时器到期时，上下文将被取消。

## 还有更多...

可能存在 goroutine 阻塞而没有明显取消方法的情况。例如，您可能阻塞等待从网络连接读取：

```go
func readData(conn net.Conn) {
  // Read a block of data from the connection
  msg:=make([]byte,1024)
  n, err:=conn.Read(msg)
  ...
}
```

此操作无法取消，因为 `Read` 不接受 `Context`。如果您想取消此类操作，可以异步关闭底层连接（或文件）。以下代码片段演示了一个用例，其中必须在 1 秒内读取连接的所有数据，否则 goroutine 将异步关闭连接：

```go
timeout, cancel := context.WithTimeout(context.Background(),1*time.Second)
defer cancel()
// Close the connection when context times out
go func() {
   // Wait for cancelation signal
   <-cancelable.Done()
   // Close the connection
   conn.Close()
}()
wg:=sync.WaitGroup()
wg.Add(1)
// This goroutine must complete within a second, or the connection 
// will be closed
go func() {
   defer wg.Done()
    // Read a block of data from the connetion
   msg:=make([]byte,1024)
   // This call may block
   n, err:=conn.Read(msg)
   if err!=nil {
      return
   }
   // Process data
}()
wg.Wait() // Wait for the processing of connection to complete
...
```

# 在服务器中使用取消和超时

网络服务器通常在接收到新请求时启动一个新的上下文。通常，服务器在请求者关闭连接时取消上下文。大多数 HTTP 框架，包括标准库，都遵循这个基本模式。如果您正在编写自己的 TCP 服务器，您必须自己实现它。

## 如何做到这一点...

处理具有超时或取消的网络连接的步骤如下：

1.  当您接受网络连接时，创建一个新的带有取消或超时的上下文：

1.  确保上下文最终被取消。

1.  将上下文传递给处理器：

    ```go
    ln, err:=net.Listen("tcp",":8080")
    if err!=nil {
      return err
    }
    for {
      conn, err:=ln.Accept()
      if err!=nil {
        return err
      }
      go func(c net.Conn) {
         // Step 1:
         // Request times out after duration: RequestTimeout
         ctx, cancel:=context.WithTimeout(context.
         Background(),RequestTimeout)
         // Step 2:
         // Make sure cancel is called
         defer cancel()
         // Step 3:
         // Pass the context to handler
         handleRequest(ctx,c)
      }(conn)
    }
    ```
