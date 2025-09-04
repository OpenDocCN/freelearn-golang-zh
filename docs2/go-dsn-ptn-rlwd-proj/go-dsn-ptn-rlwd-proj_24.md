# 第十章。并发模式 - 工作池和发布/订阅设计模式

我们已经到达了本书的最后一章，我们将讨论一些具有并发结构的模式。我们将详细解释每个步骤，以便您可以仔细地跟随示例。

我们的目的是了解如何使用 Go 的惯用方法设计并发应用程序的模式。我们大量使用通道和 Goroutines，而不是锁或共享变量。

+   我们将探讨一种开发工作池的方法。这对于控制执行中的 Goroutines 数量非常有用。

+   第二个示例是对观察者模式的重写，我们在第七章中看到了它，*行为模式 - 访问者、状态、中介者和观察者设计模式*，它使用并发结构编写。通过这个例子，我们将更深入地了解并发结构，并看看它们如何与常见方法不同。

# 工作池

我们在之前的一些并发方法中可能会遇到的一个问题是它们的上下文无界。我们不能让一个应用程序无限制地创建 Goroutines。Goroutines 很轻量，但它们执行的工作可能非常繁重。工作池可以帮助我们解决这个问题。

## 描述

使用工作池，我们希望限制可用的 Goroutines 数量，以便我们能够更深入地控制资源池。通过为每个工作创建一个通道并让工作处于空闲或忙碌状态，这很容易实现。这项任务可能看起来很艰巨，但实际上并非如此。

## 目标

创建工作池完全是关于资源控制：CPU、RAM、时间、连接等等。工作池设计模式帮助我们做到以下几点：

+   使用配额控制对共享资源的访问

+   每个应用程序创建有限数量的 Goroutines

+   为其他并发结构提供更多的并行能力

## 管道池

在上一章中，我们看到了如何使用管道。现在我们将启动有限数量的管道，以便 Go 调度器可以尝试并行处理请求。这里的想法是通过并发结构控制 Goroutines 的数量，当应用程序完成时优雅地停止它们，并最大限度地提高并行性，同时避免竞争条件。

我们将使用的管道与上一章中使用的类似，当时我们在生成数字，将它们平方，并求和最终结果。在这种情况下，我们将传递字符串，我们将向其中添加和添加前缀数据。

## 接受标准

在商业术语中，我们希望得到一些信息，表明工作者已经处理了一个请求，一个预定义的结束，以及解析为大写的传入数据：

1.  当使用字符串值（任何值）进行请求时，它必须是大写的。

1.  一旦字符串是大写的，必须向其中添加一个预定义的文本。这个文本不应该是大写的。

1.  根据前面的结果，必须将工作器 ID 前缀添加到最终字符串中。

1.  生成的字符串必须传递给预定义的处理程序。

我们还没有讨论如何从技术上实现它，只是讨论了业务需求。有了整个描述，我们至少会有工作器、请求和处理程序。

## 实现

最开始是一个请求类型。根据描述，它必须包含将进入管道的字符串以及处理函数：

```go
   // workers_pipeline.go file 
    type Request struct { 
          Data    interface{} 
          Handler RequestHandler 
    } 

```

`return` 在哪里？我们有一个 `Data` 字段，其类型为 `interface{}`，因此我们可以使用它来传递一个字符串。通过使用接口，我们可以重用此类型来传递 `string`、`int` 或 `struct` 数据类型。接收者是必须知道如何处理传入接口的那个人。

`Handler` 字段具有 `Request` 处理程序的类型，我们还没有定义它：

```go
type RequestHandler func(interface{}) 

```

请求处理程序是任何接受接口作为其第一个参数且不返回任何内容的函数。再次，我们看到 `interface{}`，我们通常在这里看到一个字符串。这是我们之前提到的一个接收者，我们需要将其转换为传入的结果。

因此，在发送请求时，我们必须在 `Data` 字段中填充一些值并实现一个处理程序；例如：

```go
func NewStringRequest(s string, id int, wg *sync.WaitGroup) Request { 
    return := Request{ 
        Data: "Hello", Handler: func(i interface{})
        { 
            defer wg.Done() 
            s, ok := i.(string) 
                if !ok{ 
                    log.Fatal("Invalid casting to string") 
                 } 
             fmt.Println(s) 
         } 
    } 
} 

```

处理程序是通过使用闭包定义的。我们再次检查接口的类型（并在最后延迟调用 `Done()` 方法）。如果接口不正确，我们简单地打印其内容并返回。如果转换是正确的，我们也打印它们，但在这里我们通常会做些操作来处理操作的结果；我们必须使用类型转换来检索 `interface{}`（它是一个字符串）的内容。这必须在管道的每个步骤中完成，尽管这会引入一点开销。

现在，我们需要一个可以处理 `Request` 类型的类型。可能的实现几乎是无限的，因此最好首先定义一个接口：

```go
   // worker.go file 
    type WorkerLauncher interface { 
        LaunchWorker(in chan Request) 
    } 

```

`WorkerLauncher` 接口必须仅实现 `LaunchWorker(chan Request)` 方法。任何实现此接口的类型都必须接收一个 `Request` 类型的通道以满足它。这个 `Request` 类型的通道是管道的单个入口点。

### 分发器

现在，为了并行启动工作器并处理所有可能的传入通道，我们需要一个类似分发器的工具：

```go
   // dispatcher.go file 
    type Dispatcher interface { 
        LaunchWorker(w WorkerLauncher) 
        MakeRequest(Request) 
        Stop() 
    } 

```

`Dispatcher` 接口可以在其自己的 `LaunchWorker` 方法中启动注入的 `WorkerLaunchers` 类型。`Dispatcher` 接口必须使用 `WorkerLauncher` 类型中的任何 `LaunchWorker` 方法来初始化管道。这样我们就可以重用 `Dispatcher` 接口来启动许多类型的 `WorkerLaunchers`。

当使用 `MakeRequest(Request)` 时，`Dispatcher` 接口提供了一个将新的 `Request` 注入工作池的便捷方法。

最后，当所有 Goroutines 都必须完成时，用户必须调用 stop。我们必须在我们的应用程序中处理优雅的关闭，并且我们希望避免 Goroutine 泄露。

我们已经有了足够的接口，所以让我们从稍微简单一点的分发器开始：

```go
    type dispatcher struct { 
        inCh chan Request 
    } 

```

我们的`dispatcher`结构在其字段之一中存储了一个`Request`类型的通道。这将是要进入任何管道请求的单一点。我们说它必须实现三个方法，如下所示：

```go
    func (d *dispatcher) LaunchWorker(id int, w WorkerLauncher) { 
        w.LaunchWorker(d.inCh) 
    } 

    func (d *dispatcher) Stop(){ 
        close(d.inCh) 
    } 

    func (d *dispatcher) MakeRequest(r Request) { 
        d.inCh <- r 
    } 

```

在这个例子中，`Dispatcher`接口在启动一个 worker 之前不需要对自己做任何特殊的事情，所以`Dispatcher`上的`LaunchWorker`方法简单地执行传入的`WorkerLauncher`的`LaunchWorker`方法，这个`WorkerLauncher`也有一个`LaunchWorker`方法来自启动自己。我们之前定义了一个`WorkerLauncher`类型至少需要一个 ID 和一个用于传入请求的通道，所以这就是我们传递的内容。

在`Dispatcher`接口中实现`LaunchWorker`方法可能看起来是不必要的。在不同的场景中，保存正在运行的 worker ID 到分发器中，以控制哪些是开启或关闭的，可能是有趣的；这个想法是隐藏启动实现的细节。在这种情况下，`Dispatcher`接口仅仅充当一个外观设计模式，隐藏了一些实现细节给用户。

第二种方法是`Stop`。它关闭了传入请求的通道，从而引发连锁反应。我们在管道示例中看到，当关闭传入通道时，Goroutines 内部的每个 for-range 循环都会中断，Goroutine 也会结束。在这种情况下，当关闭共享通道时，它将在每个监听 Goroutine 中引发相同的反应，因此所有管道都将停止。酷吧？

请求实现非常简单；我们只需将请求作为参数传递给传入请求的通道。Goroutine 将在这里永久阻塞，直到通道的另一端检索到请求。永远？如果发生什么情况，这似乎有点多。我们可以引入一个超时，如下所示：

```go
    func (d *dispatcher) MakeRequest(r Request) { 
        select { 
        case d.inCh <- r: 
        case <-time.After(time.Second * 5): 
            return 
        } 
    } 

```

如果你记得前面的章节，我们可以使用 select 来控制对通道执行的操作。就像`switch`案例一样，只能执行一个操作。在这种情况下，我们有两个不同的操作：发送和接收。

第一种情况是发送操作--尝试发送这个，它将在这里阻塞，直到有人从通道的另一端取走值。这并没有太大的改进。第二种情况是接收操作；如果上面的请求无法成功发送，它将在 5 秒后触发，函数将返回。在这里返回一个错误会很方便，但为了简单起见，我们将它留空。

最后，在分发器中，为了方便，我们将定义一个`Dispatcher`创建器：

```go
    func NewDispatcher(b int) Dispatcher { 
        return &dispatcher{ 
            inCh:make(chan Request, b), 
        } 
    } 

```

通过使用这个函数而不是手动创建分发器，我们可以简单地避免一些小错误，比如忘记初始化通道字段。正如你所见，`b`参数指的是通道中的缓冲区大小。

### 管道

因此，我们的调度器已经完成，我们需要开发符合验收标准的管道。首先，我们需要一个类型来实现`WorkerLauncher`类型：

```go
   // worker.go file 
    type PreffixSuffixWorker struct { 
        id int 
        prefixS string 
        suffixS string 
    } 

    func (w *PreffixSuffixWorker) LaunchWorker(i int, in chan Request) {} 

```

`PreffixSuffixWorker`变量存储一个 ID，一个要前缀的字符串，以及一个要后缀`Request`类型输入数据的另一个字符串。因此，要前缀和追加的值将在这两个字段中是静态的，我们将从那里获取它们。

我们将在稍后实现`LaunchWorker`方法，并从管道中的每个步骤开始。根据*第一个验收标准*，输入的字符串必须是大写的。因此，大写方法将是我们的管道中的第一步：

```go
    func (w *PreffixSuffixWorker) uppercase(in <-chan Request) <-chan Request { 
        out := make(chan Request) 

        go func() { 
            for msg := range in { 
                s, ok := msg.Data.(string) 

                if !ok { 
                    msg.handler(nil) 
                    continue 
                } 

                msg.Data = strings.ToUpper(s) 

                out <- msg 
            } 

            close(out) 
        }() 

        return out 
    } 

```

好的。正如前一章中提到的，管道中的步骤接受一个输入数据通道，并返回一个相同类型的通道。它与我们前一章开发的例子有非常相似的方法。不过，这次我们并没有使用包函数，大写是`PreffixSuffixWorker`类型的一部分，而输入数据是一个`struct`而不是`int`。

`msg`变量是`Request`类型，它将有一个处理函数和数据，数据形式为一个接口。`Data`字段应该是一个字符串，因此在使用它之前我们应该进行类型转换。当进行类型转换时，我们将收到请求的类型值和一个`true`或`false`标志（由`ok`变量表示）。如果`ok`变量是`false`，则无法进行转换，我们不会将值向下传递到管道中。我们通过向处理程序发送`nil`来停止这个`Request`（这也会引发类型转换错误）。

一旦我们在`s`变量中有一个好的字符串，我们就可以将其转换为大写，并将其再次存储在`Data`字段中，以便发送到管道的下一个步骤。请注意，值将再次作为接口发送，因此下一个步骤将需要再次进行类型转换。这是使用这种方法的一个缺点。

第一步完成后，让我们继续第二步。根据现在的*第二个验收标准*，必须追加一个预定义的文本。这个文本是存储在`suffixS`字段中的：

```go
func (w *PreffixSuffixWorker) append(in <-chan Request) <-chan Request { 
    out := make(chan Request) 
    go func() { 
        for msg := range in { 
        uppercaseString, ok := msg.Data.(string) 

        if !ok { 
            msg.handler(nil) 
            continue 
            } 
        msg.Data = fmt.Sprintf("%s%s", uppercaseString, w.suffixS) 
        out <- msg 
        } 
        close(out) 
    }() 
    return out 
} 

```

`append`函数的结构与`uppercase`函数相同。它接收并返回一个输入请求通道，并启动一个新的 Goroutine，该 Goroutine 迭代输入通道直到其关闭。我们需要像之前提到的那样对输入值进行类型转换。

在这个管道步骤中，输入的字符串是大写的（在类型断言之后）。要向其追加任何文本，我们只需使用`fmt.Sprintf()`函数，就像我们之前多次做的那样，它使用提供的数据格式化一个新的字符串。在这种情况下，我们将`suffixS`字段的值作为第二个值传递，以将其追加到字符串的末尾。

管道中只缺少最后一步，即前缀操作：

```go
    func (w *PreffixSuffixWorker) prefix(in <-chan Request) { 
        go func() { 
            for msg := range in { 
                uppercasedStringWithSuffix, ok := msg.Data.(string) 

                if !ok { 
                    msg.handler(nil) 
                    continue 
                } 

                msg.handler(fmt.Sprintf("%s%s", w.prefixS, uppercasedStringWithSuffix)) 
            } 
        }() 
    } 

```

在这个函数中，什么引起了你的注意？是的，它现在不返回任何通道。我们可以用两种方式完成整个管道。我想你可能已经意识到我们在管道中使用了`Future`处理程序函数来执行最终结果。第二种方法是将通道传递回其原始位置。在某些情况下，`Future`可能足够，而在其他情况下，传递通道可能更方便，以便它可以连接到不同的管道（例如）。

在任何情况下，管道中步骤的结构对你来说应该已经很熟悉了。我们转换值，检查转换的结果，如果出现任何错误，则向处理器发送 nil。但是，如果一切正常，最后要做的就是再次格式化文本，将`prefixS`字段放置在文本的开头，通过调用请求的处理程序将结果字符串发送回原始位置。

现在，随着我们的工作者几乎完成，我们可以实现`LaunchWorker`方法：

```go
    func (w *PreffixSuffixWorker) LaunchWorker(in chan Request) { 
        w.prefix(w.append(w.uppercase(in))) 
    } 

```

对于工作者来说，这就结束了！我们只需将返回的通道传递到管道中的下一步，就像我们在上一章中所做的那样。记住，管道是从调用内部到外部执行的。那么，任何传入管道的数据的执行顺序是什么？

1.  数据通过在`uppercase`方法中启动的 Goroutine 进入管道。

1.  然后，它进入在`append`中启动的 Goroutine。

1.  最后，它进入在`prefix`方法中启动的 Goroutine，这个 Goroutine 不返回任何内容，但在给传入的字符串添加更多数据后执行处理器。

现在，我们有一个完整的管道和管道分发器。分发器将启动尽可能多的管道实例，以便将传入的请求路由到任何可用的工作者。

如果在 5 秒内没有工作者接收请求，请求就会丢失。

让我们在一个小型应用程序中使用这个库。

## 使用工作者池的应用程序

我们将启动我们定义的管道的三个工作者。我们使用`NewDispatcher`函数创建分发器和接收所有请求的通道。这个通道有一个固定的缓冲区，能够在阻塞之前存储多达 100 条传入的消息：

```go
   // workers_pipeline.go 
    func main() { 
        bufferSize := 100 
        var dispatcher Dispatcher = NewDispatcher(bufferSize) 

```

然后，我们将通过在`Dispatcher`接口中三次调用`LaunchWorker`方法并使用已经填充的`WorkerLauncher`类型来启动工作者：

```go
    workers := 3 
    for i := 0; i < workers; i++ { 
        var w WorkerLauncher = &PreffixSuffixWorker{ 
            prefixS: fmt.Sprintf("WorkerID: %d -> ", i), 
            suffixS: " World", 
            id:i, 
        } 
        dispatcher.LaunchWorker(w) 
    } 

```

每个`WorkerLauncher`类型是`PreffixSuffixWorker`的一个实例。前缀将是一个显示工作者 ID 的小文本，后缀文本为`world`。

在这个阶段，我们有了三个工作者和三个 Goroutines，它们并发运行并等待消息的到来：

```go
    requests := 10 

    var wg sync.WaitGroup 
    wg.Add(requests) 

```

我们将发起 10 个请求。我们还需要一个 WaitGroup 来正确同步应用程序，以便它不会太早退出。当处理并发应用程序时，您可能会大量使用 WaitGroups。对于 10 个请求，我们需要等待 10 次对`Done()`方法的调用，因此我们使用带有*增量*为 10 的`Add()`方法。它被称为增量，因为您也可以稍后传递-5，以使其在五个请求中完成。在某些情况下，这可能很有用：

```go
    for i := 0; i < requests; i++ { 
        req := NewStringRequest("(Msg_id: %d) -> Hello", i, &wg) 
        dispatcher.MakeRequest(req) 
    } 

    dispatcher.Stop() 

    wg.Wait() 
}
```

为了发起请求，我们将迭代一个`for`循环。首先，我们使用我们在实现部分开头编写的函数`NewStringRequest`创建一个`Request`。在这个值中，`Data`字段将是我们将通过管道传递的文本，它将是追加和后缀操作中的“中间”文本。在这种情况下，我们将发送消息编号和单词`hello`。

一旦我们收到请求，我们就用它调用`MakeRequest`方法。完成所有请求后，我们停止调度器，正如之前解释的那样，这将引发连锁反应，停止管道中的所有 Goroutines。

最后，我们等待组，以便接收到所有对`Done()`方法的调用，这表示所有操作都已完成。现在是时候尝试一下了：

```go
 go run *
 WorkerID: 1 -> (MSG_ID: 0) -> HELLO World
 WorkerID: 0 -> (MSG_ID: 3) -> HELLO World
 WorkerID: 0 -> (MSG_ID: 4) -> HELLO World
 WorkerID: 0 -> (MSG_ID: 5) -> HELLO World
 WorkerID: 2 -> (MSG_ID: 2) -> HELLO World
 WorkerID: 1 -> (MSG_ID: 1) -> HELLO World
 WorkerID: 0 -> (MSG_ID: 6) -> HELLO World
 WorkerID: 2 -> (MSG_ID: 9) -> HELLO World
 WorkerID: 0 -> (MSG_ID: 7) -> HELLO World
 WorkerID: 0 -> (MSG_ID: 8) -> HELLO World

```

让我们分析第一条消息：

1.  这将是零，所以发送的消息是`(Msg_id: 0) -> Hello`。

1.  然后，文本被转换为大写，所以我们现在有`(MSG_ID: 0) -> HELLO`。

1.  在将带有文本`world`（注意文本开头的空格）的追加操作转换为大写后完成。这将给我们文本`(MSG_ID: 0) -> HELLO World`。

1.  最后，将文本`WorkerID: 1`（在这种情况下，第一个工作器接受了任务，但可能是任何一个）追加到步骤 3 中的文本，以给出完整的返回消息，`WorkerID: 1 -> (MSG_ID: 0) -> HELLO World`。

## 没有测试吗？

并发应用程序很难测试，尤其是如果您正在进行网络操作。这可能很困难，代码可能需要大量更改才能进行测试。在任何情况下，不进行测试都是不可取的。在这种情况下，测试我们的小应用程序并不特别困难。创建一个测试并将`main`函数的内容复制/粘贴到那里：

```go
//workers_pipeline.go file 
package main 

import "testing" 

func Test_Dispatcher(t *testing.T){ 
    //pasted code from main function 
 bufferSize := 100
 var dispatcher Dispatcher = NewDispatcher(bufferSize)
 workers := 3
 for i := 0; i < workers; i++ 
    {
 var w WorkerLauncher = &PreffixSuffixWorker{
 prefixS: fmt.Sprintf("WorkerID: %d -> ", i), 
suffixS: " World", 
id: i,
}
 dispatcher.LaunchWorker(w)
 }
 //Simulate Requests
 requests := 10
 var wg 
    sync.WaitGroup
 wg.Add(requests) 
} 

```

现在我们必须重写我们的处理程序来测试返回的内容是否是我们预期的。转到`for`循环来修改我们传递给每个`Request`的处理函数：

```go
for i := 0; i < requests; i++ { 
    req := Request{ 
        Data: fmt.Sprintf("(Msg_id: %d) -> Hello", i), 
        handler: func(i interface{}) 
        { 
            s, ok := i.(string) 
            defer wg.Done() 
 if !ok 
            {
 t.Fail()
 }
 ok, err := regexp.Match(
`WorkerID\: \d* -\> \(MSG_ID: \d*\) -> [A-Z]*\sWorld`,
 []byte(s)) 
 if !ok || err != nil {
 t.Fail()
 } 
        }, 
    } 
    dispatcher.MakeRequest(req) 
} 

```

我们将使用正则表达式来测试业务。如果您不熟悉正则表达式，它们是一种非常强大的功能，可以帮助您在字符串中匹配内容。如果您记得在我们的练习中，当我们在使用`strings`包时。`Contains`是用于在字符串中查找文本的函数。我们也可以使用正则表达式来做这件事。

问题在于正则表达式相当昂贵，消耗大量资源。

我们正在使用`regexp`包的`Match`函数提供一个模板进行匹配。我们的模板是`WorkerID: \d* -> (MSG_ID: \d) -> [A-Z]*\sWorld`（不带引号）。具体来说，它描述了以下内容：

+   一个包含内容`WorkerID: \d* -> (MSG_ID: \d*`的字符串，这里`"\d*"`表示零次或多次出现的任何数字，因此它将匹配`WorkerID: 10 -> (MSG_ID: 1"`和`"WorkerID: 1 -> (MSG_ID: 10"`。

+   `"\) -> [A-Z]*\sWorld"`（括号必须使用反斜杠转义）。`"*"`表示零次或多次出现的任何大写字母，所以`"\s"`是一个空白字符，它必须以文本`World`结束，所以`) -> HELLO World"`将匹配，但`) -> Hello World"`不会匹配，因为`"Hello"`必须全部大写。

运行这个测试给出了以下输出：

```go
go test -v .
=== RUN   Test_Dispatcher
--- PASS: Test_Dispatcher (0.00s)
PASS
ok

```

还不错，但我们并没有测试代码是否正在并发执行，所以这更像是业务测试而不是单元测试。并发测试将迫使我们以完全不同的方式编写代码，以检查它是否创建了正确数量的 Goroutine，并且管道是否遵循预期的流程。这并不坏，但相当复杂，超出了本书的上下文。

## 工作者池的封装

使用工作者池，我们有了第一个复杂的并发应用程序，它可以用于现实世界的生产系统。它也有改进的空间，但这是一个非常好的设计模式来构建并发的有界应用程序。

关键是我们始终要控制正在启动的 Goroutine 的数量。虽然启动数千个 Goroutine 以在应用程序中实现更多并行性很容易，但我们必须非常小心，确保它们没有可能导致无限循环的代码。

使用工作者池，我们现在可以将一个简单操作分解成许多并行任务。想想看；这可以通过一个简单的`fmt.Printf`调用实现相同的结果，但我们已经通过它建立了一个管道；然后，我们启动了这个管道的几个实例，最后，将工作负载分配给所有这些管道。

# 并发发布/订阅设计模式

在本节中，我们将实现之前在行为模式中展示的观察者设计模式，但使用并发结构和线程安全。

## 描述

如果你还记得前面的解释，观察者模式维护一个观察者或订阅者列表，这些观察者或订阅者希望被通知特定事件。在这种情况下，每个订阅者将运行在不同的 Goroutine 中，以及发布者。我们将遇到构建这种结构的新问题：

+   现在，对订阅者列表的访问必须进行序列化。如果我们用一个 Goroutine 读取列表，我们不能从其中移除订阅者，否则我们将遇到竞争条件。

+   当订阅者被移除时，订阅者的 Goroutine 也必须关闭，否则它将无限迭代，我们将遇到 Goroutine 泄漏。

+   当停止发布者时，所有订阅者也必须停止它们的 Goroutine。

## 目标

发布/订阅的目标与我们在观察者模式中写下的目标相同。这里的区别在于我们将如何开发它。理念是创建一个并发结构以实现相同的功能，具体如下：

+   提供一个事件驱动的架构，其中一个事件可以触发一个或多个动作

+   解耦执行的动作与触发它们的动作

+   提供多个源事件以触发相同的动作

理念是将发送者与接收者解耦，对发送者隐藏将处理其事件的接收者身份，并隐藏接收者从可以与之通信的发送者数量。

特别是，如果我在某个应用程序中的按钮上开发一个点击事件，它可能执行某些操作（例如登录到某个地方）。几周后，我们可能会决定让它显示一个弹出窗口。如果我们每次想要向这个按钮添加一些功能时，都必须更改处理点击动作的代码，那么这个函数将变得很大，并且不太适合其他项目。如果我们使用发布者和每个动作的一个观察者，点击函数只需要使用发布者发布一个单一的事件，每次我们想要改进功能时，我们只需为这个事件编写订阅者即可。这在具有用户界面的应用程序中尤为重要，因为单个 UI 动作中要执行的多项任务可能会降低界面的响应速度，完全破坏用户体验。

通过使用并发结构来开发观察者模式，如果定义了并发结构并且设备允许我们执行并行任务，UI 就无法感觉到正在后台执行的所有任务。

## 示例 - 并发通知器

我们将开发一个类似于我们在第七章中开发的那个通知器，*行为模式 - 访问者、状态、中介者和观察者设计模式*。这是为了关注结构的并发性，而不是详细说明已经解释过的太多内容。我们已经开发了一个观察者，因此我们对这个概念很熟悉。

这个特定的通知器将通过传递`interface{}`值来工作，就像在工作者池示例中一样。这样，我们可以通过在接收者上进行类型转换时引入一些开销，来使用它处理多种类型。

现在，我们将使用两个接口。首先是一个`Subscriber`接口：

```go
    type Subscriber interface { 
        Notify(interface{}) error 
        Close() 
    } 

```

就像在之前的示例中一样，它必须在`Subscriber`接口中有一个`Notify`方法来处理新事件。这是接受`interface{}`值并返回错误的`Notify`方法。然而，`Close()`方法却是新的，它必须触发停止订阅者监听新事件的 Goroutine 所需的任何动作。

第二个也是最后一个接口是`Publisher`接口：

```go
    type Publisher interface { 
        start() 
        AddSubscriberCh() chan<- Subscriber 
        RemoveSubscriberCh() chan<- Subscriber 
        PublishingCh() chan<- interface{} 
        Stop() 
    } 

```

`Publisher`接口具有我们已知的发布者相同的操作，但与通道一起工作。`AddSubscriberCh`和`RemoveSubscriberCh`方法接受一个`Subscriber`接口（任何满足`Subscriber`接口的类型）。它必须有一个发布消息的方法和一个`Stop`方法来停止所有（发布者和订阅者 Goroutine）。

## 验收标准

与第七章 *行为模式 - 访问者、状态、中介者和观察者设计模式* 中的示例之间的要求必须保持不变。这两个示例的目标是相同的，因此要求也必须相同。在这种情况下，我们的要求是技术的，因此我们实际上需要添加一些额外的验收标准：

1.  我们必须有一个具有`PublishingCh`方法的发布者，该方法返回一个用于发送消息的通道，并在每个已订阅的观察者上触发`Notify`方法。

1.  我们必须有一个方法来向发布者添加新订阅者。

1.  我们必须有一个方法来从发布者中移除新订阅者。

1.  我们必须有一个方法来停止订阅者。

1.  我们必须有一个方法来停止一个`Publisher`接口，这将也会停止所有订阅者。

1.  所有跨 Goroutine 通信必须同步，以确保没有 Goroutine 被锁定等待响应。在这种情况下，在指定超时时间过后返回错误。

好吧，这些标准似乎相当令人畏惧。我们省略了一些会增加更多复杂性的要求，例如移除无响应的订阅者或检查以监控发布者 Goroutine 始终处于活动状态。

## 单元测试

我们之前提到，测试并发应用程序可能很困难。有了正确的机制，这仍然可以做到，所以让我们看看我们可以在不遇到大麻烦的情况下测试多少。

### 测试订阅者

从订阅者开始，它们似乎具有更封装的功能，第一个订阅者必须将发布者传入的消息打印到`io.Writer`接口。我们之前提到，订阅者有一个接口，包含两个方法，`Notify(interface{}) error`和`Close()`方法：

```go
    // writer_sub.go file 
    package main 

    import "errors" 

    type writerSubscriber struct { 
        id int 
        Writer io.Writer 
    } 

    func (s *writerSubscriber) Notify(msg interface{}) error { 
        return erorrs.NeW("Not implemented yet") 
    } 
    func (s *writerSubscriber) Close() {} 

```

好的。这将是我们`writer_sub.go`文件。创建相应的测试文件，称为`writer_sub_test.go`文件：

```go
    package main 
    func TestStdoutPrinter(t *testing.T) { 

```

现在，我们面临的首要问题是功能输出到`stdout`，因此没有返回值可以检查。我们可以用三种方法解决这个问题：

+   捕获`stdout`方法。

+   注入`io.Writer`接口以打印到它。这是首选解决方案，因为它使代码更易于管理。

+   将`stdout`方法重定向到不同的文件。

我们将采取第二种方法。重定向也是一个可能的选择。`os.Stdout`是一个指向`os.File`类型的指针，因此这涉及到用我们控制的文件替换这个文件，并从中读取：

```go
    func TestWriter(t *testing.T) { 
        sub := NewWriterSubscriber(0, nil) 

```

`NewWriterSubscriber`订阅者尚未定义。它必须帮助创建这个特定的订阅者，返回一个满足`Subscriber`接口的类型，因此让我们快速在`writer_sub.go`文件中声明它：

```go
    func NewWriterSubscriber(id int, out io.Writer) Subscriber { 
        return &writerSubscriber{} 
    } 

```

理想情况下，它必须接受一个 ID 和一个`io.Writer`接口作为其写入的目的地。在这种情况下，我们需要为我们的测试创建一个自定义的`io.Writer`接口，因此我们将在`writer_sub_test.go`文件中创建一个`mockWriter`：

```go
    type mockWriter struct { 
        testingFunc func(string) 
    } 

    func (m *mockWriter) Write(p []byte) (n int, err error) { 
        m.testingFunc(string(p)) 
        return len(p), nil 
    } 

```

`mockWriter`结构将接受一个`testingFunc`作为其字段之一。这个`testingFunc`字段接受一个表示写入到`mockWriter`结构的字节的字符串。为了实现`io.Writer`接口，我们需要定义一个`Write([]byte) (int, error)`方法。在我们的定义中，我们将`p`的内容作为字符串传递（记住，我们总是在每个`Write`方法上返回读取的字节数和错误，或者不返回错误）。这种方法将`testingFunc`的定义委托给测试的作用域。

我们将在`Subcriber`接口上调用`Notify`方法，这个接口必须像`mockWriter`结构一样实现`io.Writer`接口。因此，在调用`Notify`方法之前，我们将定义`mockWriter`结构的`testingFunc`：

```go
    // writer_sub_test.go file 
    func TestPublisher(t *testing.T) { 
        msg := "Hello" 

        var wg sync.WaitGroup 
        wg.Add(1) 

        stdoutPrinter := sub.(*writerSubscriber) 
        stdoutPrinter.Writer = &mockWriter{ 
            testingFunc: func(res string) { 
                if !strings.Contains(res, msg) { 
                    t.Fatal(fmt.Errorf("Incorrect string: %s", res)) 
                } 
                wg.Done() 
            }, 
        } 

```

我们将发送`Hello`消息。这也意味着无论`Subscriber`接口执行什么操作，它最终必须在提供的`io.Writer`接口上打印出`Hello`消息。

因此，如果最终在测试函数中接收到一个字符串，我们需要与`Subscriber`接口同步，以避免测试中的竞态条件。这就是为什么我们使用了大量的`WaitGroup`。它是一个非常方便且易于使用的类型，用于处理这种情况。一个`Notify`方法调用将需要等待一个`Done()`方法调用，因此我们调用`Add(1)`方法（一个单位）。

理想情况下，`NewWriterSubscriber`函数必须返回一个接口，因此我们需要在测试期间将其类型断言为我们正在使用的类型，在这种情况下是`stdoutPrinter`方法。我故意省略了类型转换时的错误检查，只是为了使事情变得简单。一旦我们有了`writerSubscriber`类型，我们就可以访问它的`Write`字段，并用`mockWriter`结构替换它。我们本可以直接在`NewWriterSubscriber`函数中传递一个`io.Writer`接口，但这样我们就无法覆盖传递 nil 对象并将其设置为默认值的场景。

因此，测试函数最终将接收到一个包含订阅者写入内容的字符串。我们只需检查接收到的字符串，即`Subscriber`接口将接收到的字符串，在某个时刻是否打印了单词`Hello`，而对于这一点，`strings.Contains`函数是最好的选择。所有这些都定义在测试函数的作用域内，因此我们可以使用`t`对象的值来指示测试失败。

一旦完成检查，我们必须调用`Done()`方法来表示我们已经测试了预期的结果：

```go
err := sub.Notify(msg) 
if err != nil { 
    t.Fatal(err) 
    } 

    wg.Wait() 
    sub.Close() 
} 

```

我们实际上必须调用`Notify`和`Wait`方法来检查调用`Done`方法是否正确。

### 注意

你意识到我们没有在测试中定义行为，而是基本上是反过来的吗？这在并发应用程序中非常常见。有时可能会令人困惑，因为如果我们不能线性地跟踪调用，就很难知道一个函数可能正在做什么，但你会很快习惯它。与其“这样做，然后这样做，然后那样做”的想法不同，它更像是“当执行那个时将会调用这个”。这也是因为在并发应用程序中，执行顺序在某个点之前是未知的，除非我们使用同步原语（如 WaitGroups 和通道）在特定时刻暂停执行。

现在我们来执行这个类型的测试：

```go
go test -cover -v -run=TestWriter .
=== RUN   TestWriter
--- FAIL: TestWriter (0.00s)
 writer_sub_test.go:40: Not implemented yet
FAIL
coverage: 6.7% of statements
exit status 1
FAIL

```

它快速退出但失败了。实际上，调用`Done()`方法尚未执行，所以最好将我们的测试的最后部分改为这样：

```go
err := sub.Notify(msg)
if err != nil {
 wg.Done()
t.Error(err)
 }
 wg.Wait()
sub.Close()
 } 

```

现在，它不会停止执行，因为我们调用的是`Error`函数而不是`Fatal`函数，但我们调用了`Done()`方法，测试就在我们希望结束的地方结束，在调用`Wait()`方法之后。你可以再次尝试运行测试，但输出将相同。

### 测试发布者

我们已经看到了`Publisher`接口以及将满足该接口的类型，即`publisher`类型。我们唯一确定的是它将需要某种方式来存储订阅者，因此它至少将有一个`Subscribers`切片：

```go
    // publisher.go type 
    type publisher struct { 
        subscribers []Subscriber 
    } 

```

要测试`publisher`类型，我们还需要对`Subscriber`接口进行模拟：

```go
    // publisher_test.go 
    type mockSubscriber struct { 
        notifyTestingFunc func(msg interface{}) 
        closeTestingFunc func() 
    } 

    func (m *mockSubscriber) Close() { 
        m.closeTestingFunc() 
    } 

    func (m *mockSubscriber) Notify(msg interface{}) error { 
        m.notifyTestingFunc(msg) 
        return nil 
    } 

```

`mockSubscriber`类型必须实现`Subscriber`接口，因此它必须有一个`Close()`和一个`Notify(interface{}) error`方法。我们可以嵌入一个实现它的现有类型，例如`writerSubscriber`，并仅覆盖对我们有意义的那个方法，但我们需要定义两个，所以我们不会嵌入任何东西。

因此，在这种情况下，我们需要重写`Notify`和`Close`方法来调用存储在`mockSubscriber`类型字段上的测试函数：

```go
    func TestPublisher(t *testing.T) { 
        msg := "Hello" 

        p := NewPublisher() 

```

首先，我们将通过通道直接发送消息，这可能导致潜在的死锁，所以首先需要定义一个用于处理诸如向关闭的通道发送或没有 Goroutines 监听通道的情况的恐慌处理程序。我们将发送给订阅者的消息是`Hello`。因此，使用`AddSubscriberCh`方法返回的通道接收到的每个订阅者都必须接收到这条消息。我们还将使用一个名为`New`的函数来创建发布者，称为`NewPublisher`。现在将`publisher.go`文件更改如下：

```go
   // publisher.go file 
    func NewPublisher() Publisher { 
        return &publisher{} 
    } 

```

现在，我们将定义`mockSubscriber`并将其添加到已知的订阅者列表中。回到`publisher_test.go`文件：

```go
        var wg sync.WaitGroup 

        sub := &mockSubscriber{ 
            notifyTestingFunc: func(msg interface{}) { 
                defer wg.Done() 

                s, ok := msg.(string) 
                if !ok { 
                    t.Fatal(errors.New("Could not assert result")) 
                } 

                if s != msg { 
                    t.Fail() 
                } 
            }, 
            closeTestingFunc: func() { 
                wg.Done() 
            }, 
        } 

```

如同往常，我们从一个`WaitGroup`开始。首先，在订阅者函数执行结束时调用`Done()`方法。然后它需要将`msg`变量类型转换为字符串，因为它是一个接口。记住，这样我们就可以通过引入类型断言的开销，使用`Publisher`接口与许多类型一起使用。这是在行`s, ok := msg.(string)`上完成的。

一旦我们将`msg`类型转换为字符串`s`，我们只需检查订阅者接收到的值是否与我们发送的值相同，如果不相同，则测试失败：

```go
        p.AddSubscriberCh() <- sub 
        wg.Add(1) 

        p.PublishingCh() <- msg 
        wg.Wait() 

```

我们使用`AddSubscriberCh`方法添加`mockSubscriber`类型。我们准备好后立即发布消息，通过将`WaitGroup`加一，然后在设置`WaitGroup`等待之前，这样测试就不会继续，直到`mockSubscriber`类型调用`Done()`方法。

此外，我们还需要检查在调用`AddSubscriberCh`方法后，`Subscriber`接口的数量是否增加，因此我们需要在测试中获取发布者的具体实例：

```go
        pubCon := p.(*publisher) 
        if len(pubCon.subscribers) != 1 { 
            t.Error("Unexpected number of subscribers") 
        } 

```

类型断言是我们今天的良友！一旦我们有了具体类型，我们就可以访问`Publisher`接口的底层订阅者切片。在调用`AddSubscriberCh`方法一次之后，订阅者的数量必须是 1，否则测试将失败。下一步是检查相反的情况——当我们移除一个`Subscriber`接口时，它必须来自这个列表：

```go
   wg.Add(1) 
   p.RemoveSubscriberCh() <- sub 
   wg.Wait() 

   //Number of subscribers is restored to zero 
   if len(pubCon.subscribers) != 0 { 
         t.Error("Expected no subscribers") 
   } 

   p.Stop() 
}  

```

我们测试的最终步骤是停止发布者，这样就不会再发送更多消息，并且所有协程都会停止。

测试已经完成，但我们不能运行测试，直到`publisher`类型实现了所有方法；这必须是最终结果：

```go
    type publisher struct { 
        subscribers []Subscriber 
        addSubCh    chan Subscriber 
        removeSubCh chan Subscriber 
        in          chan interface{} 
        stop        chan struct{} 
    } 

    func (p *publisher) AddSubscriberCh() chan<- Subscriber { 
        return nil 
    } 

    func (p *publisher) RemoveSubscriberCh() chan<- Subscriber { 
        return nil 
    } 

    func (p *publisher) PublishingCh() chan<- interface{} { 
        return nil 
    } 

    func (p *publisher) Stop(){} 

```

使用这个空实现，在运行测试时什么好事都不会发生：

```go
go test -cover -v -run=TestPublisher .
atal error: all goroutines are asleep - deadlock!
goroutine 1 [chan receive]:
testing.(*T).Run(0xc0420780c0, 0x5244c6, 0xd, 0x5335a0, 0xc042037d20)
 /usr/local/go/src/testing/testing.go:647 +0x31d
testing.RunTests.func1(0xc0420780c0)
 /usr/local/go/src/testing/testing.go:793 +0x74
testing.tRunner(0xc0420780c0, 0xc042037e10)
 /usr/local/go/src/testing/testing.go:610 +0x88
testing.RunTests(0x5335b8, 0x5ada40, 0x2, 0x2, 0x40d7e9)
 /usr/local/go/src/testing/testing.go:799 +0x2fc
testing.(*M).Run(0xc042037ed8, 0xc04200a4f0)
 /usr/local/go/src/testing/testing.go:743 +0x8c
main.main()
 go-design-patterns/concurrency_3/pubsub/_test/_testmain.go:56 +0xcd
goroutine 5 [chan send (nil chan)]:
go-design-patterns/concurrency_3/pubsub.TestPublisher(0xc042078180)
 go-design-patterns/concurrency_3/pubsub/publisher_test.go:55 +0x372
testing.tRunner(0xc042078180, 0x5335a0)
 /usr/local/go/src/testing/testing.go:610 +0x88
created by testing.(*T).Run
 /usr/local/go/src/testing/testing.go:646 +0x2f3
exit status 2
FAIL  go-design-patterns/concurrency_3/pubsub   1.587s

```

是的，它失败了，但这根本不是受控失败。这是故意为之，为了展示在 Go 中需要注意的一些事情。首先，这个测试中产生的错误是一个**致命**错误，通常指向代码中的错误。这很重要，因为虽然**panic**错误可以被恢复，但你不能对致命错误做同样的事情。

在这种情况下，错误告诉我们问题：`goroutine 5 [chan send (nil chan)]`，一个`nil`通道，这实际上是我们代码中的一个错误。我们如何解决这个问题？嗯，这也是很有趣的。

我们拥有一个`nil`通道的事实是由于我们编写的用于编译单元测试的代码造成的，但一旦编写了适当的代码，这个特定的错误就不会发生（因为我们永远不会在这种情况下返回一个`nil`通道）。我们可以返回一个永远不会使用的通道，这会导致死锁，从而根本没有任何进展。

解决这个问题的一个习惯用法是返回一个通道和一个错误，这样你就可以有一个包含实现`Error`接口的特定错误类型（如`NoGoroutinesListening`或`ChannelNotCreated`）的错误包。我们已经看到了许多这样的实现，所以我们将把这些留作读者的练习，并将继续前进，以保持对章节并发特性的关注。

没有什么令人惊讶的，所以我们可以进入实现阶段。

## 实现

为了回顾，`writerSubscriber`必须接收它将写入满足`io.Writer`接口的类型上的消息。

那么，我们从哪里开始呢？嗯，每个订阅者都会运行自己的 Goroutine，我们已经看到，与 Goroutine 通信的最佳方法是使用通道。因此，我们需要在`Subscriber`类型中有一个包含通道的字段。我们可以使用与管道中相同的方法，以`NewWriterSubscriber`函数和`writerSubscriber`类型结束：

```go
    type writerSubscriber struct { 
        in     chan interface{} 
        id     int 
        Writer io.Writer 
    } 

    func NewWriterSubscriber(id int, out io.Writer) Subscriber { 
        if out == nil { 
            out = os.Stdout 
        } 

        s := &writerSubscriber{ 
            id:     id, 
            in:     make(chan interface{}), 
            Writer: out, 
        } 

        go func(){ 
            for msg := range s.in { 
                fmt.Fprintf(s.Writer, "(W%d): %v\n", s.id, msg) 
            } 
        }() 

        return s 
    } 

```

在第一步中，如果没有指定写入器（`out`参数为 nil），则默认的`io.Writer`接口是`stdout`。然后，我们创建一个新的指向`writerSubscriber`类型的指针，该指针包含通过第一个参数传入的 ID，`out`的值（`os.Stdout`，或者如果它不为 nil，则传入的参数），以及一个名为`in`的通道，以保持与之前示例中的相同命名。

然后我们启动一个新的 Goroutine；这是我们之前提到的启动机制。就像在管道中一样，订阅者会在每次接收到新消息时遍历`in`通道，并将它的内容格式化为一个字符串，该字符串也包含当前订阅者的 ID。

正如我们之前学到的，如果`in`通道被关闭，`for range`循环将停止，并且那个特定的 Goroutine 将结束，所以我们在`Close`方法中实际上需要做的只是关闭`in`通道：

```go
    func (s *writerSubscriber) Close() { 
        close(s.in) 
    } 

```

好吧，只剩下`Notify`方法了；`Notify`方法是一个方便的方法，用于在通信时管理特定的行为，我们将使用在许多调用中常见的模式：

```go
    func (s *writerSubscriber) Notify(msg interface{}) (err error) { 
        defer func(){ 
            if rec := recover(); rec != nil { 
                err = fmt.Errorf("%#v", rec) 
            } 
        }() 

        select { 
        case s.in <- msg: 
        case <-time.After(time.Second): 
            err = fmt.Errorf("Timeout\n") 
        } 

        return 
    } 

```

当通过通道进行通信时，我们必须通常控制两种行为：一种是等待时间，另一种是通道关闭时。延迟函数实际上适用于函数内部可能发生的任何恐慌错误。如果 Goroutine 发生恐慌，它仍然会使用`recover()`方法执行延迟函数。`recover()`方法返回一个表示错误的接口，所以在这种情况下，我们将返回变量`error`设置为`recover`返回的格式化值（它是一个接口）。`"%#v"`参数在格式化为字符串时提供了关于任何类型的大部分信息。返回的错误可能看起来很糟糕，但它将包含我们可以提取的大部分错误信息。例如，对于已关闭的通道，它将返回“在已关闭的通道上发送”。嗯，这似乎已经很清楚了。

第二条规则是关于等待时间。当我们通过通道发送一个值时，我们将被阻塞，直到另一个 Goroutine 从其中取出值（在填充的缓冲通道中也会发生相同的情况）。我们不希望永远被阻塞，所以通过使用 select 处理程序设置了一个一秒钟的超时期。简而言之，通过 select 我们是在说：要么你在不到 1 秒内取走值，要么我将丢弃它并返回一个错误。

我们有`Close`、`Notify`和`NewWriterSubscriber`方法，因此我们可以再次尝试我们的测试：

```go
go test -run=TestWriter -v .
=== RUN   TestWriter
--- PASS: TestWriter (0.00s)
PASS
ok

```

现在好多了。`Writer`已经将我们在测试中编写的模拟写入器写入，并将我们传递给`Notify`方法的值写入其中。同时，关闭操作可能已经有效地关闭了通道，因为`Notify`方法在调用`Close`方法后返回了一个错误。有一点需要提及的是，我们无法在不与通道交互的情况下检查通道是否已关闭；这就是为什么我们必须延迟执行一个将检查`Notify`方法中`recover()`函数内容的闭包的执行。

### 实现发布者

好的，发布者还需要一个启动机制，但主要需要处理的问题是访问订阅者列表的竞态条件。我们可以使用`sync`包中的 Mutex 对象来解决这个问题，但我们已经看到了如何使用它，所以我们将使用通道。

当使用通道时，我们需要为每个可能被视为危险的操作创建一个通道——添加订阅者、移除订阅者、检索订阅者列表以通过`Notify`方法通知他们消息，以及一个用于停止所有订阅者的通道。我们还需要一个用于接收消息的通道：

```go
    type publisher struct { 
        subscribers []Subscriber 
        addSubCh    chan Subscriber 
        removeSubCh chan Subscriber 
        in          chan interface{} 
        stop        chan struct{} 
    } 

```

名称具有自描述性，但简而言之，订阅者维护订阅者列表；这是需要复用访问的切片。`addSubCh`实例是在你想添加新订阅者时与之通信的通道；这就是为什么它是一个订阅者通道。同样的解释也适用于`removeSubCh`通道，但这个通道用于移除订阅者。`in`通道将处理必须广播给所有订阅者的传入消息。最后，当我们想要终止所有 Goroutine 时，必须调用停止通道。

好的，让我们从`AddSubscriberCh`、`RemoveSubscriber`和`PublishingCh`方法开始，这些方法必须返回用于添加和移除订阅者的通道以及用于向所有订阅者发送消息的通道：

```go
    func (p *publisher) AddSubscriber() { 
        return p.addSubCh 
    } 

    func (p *publisher) RemoveSubscriberCh() { 
        return p.removeSubCh 
    } 

    func (p *publisher) PublishMessage(){ 
        return p.in 
    } 

```

通过关闭`stop`通道来调用`Stop()`函数。这将有效地将信号传播到每个监听的 Goroutine：

```go
func (p *publisher) Stop(){ 
  close(p.stop) 
} 

```

`Stop`方法，用于停止发布者和订阅者的函数，也将推送到其各自的通道，称为停止通道。

你可能想知道为什么我们不简单地将通道保持可用，让用户直接向这个通道推送，而不是使用代理功能。好吧，想法是，将库集成到他们的应用程序中的用户不需要处理与库相关的并发结构的复杂性，这样他们就可以专注于他们的业务，尽可能最大化性能。

### 无竞态条件的通道处理

到目前为止，我们已经将数据转发到发布者的通道上，但我们实际上并没有处理任何这些数据。将要启动不同 Goroutine 的启动机制将处理所有这些数据。

我们将创建一个启动方法，我们将通过使用`go`关键字来执行它，而不是将整个函数嵌入到`NewPublisher`函数中：

```go
func (p *publisher) start() { 
  for { 
    select { 
    case msg := <-p.in: 
      for _, ch := range p.subscribers { 
        sub.Notify(msg) 
      } 

```

`Launch`是一个私有方法，我们还没有对其进行测试。记住，私有方法通常是从公共方法（我们已经测试过的方法）中调用的。通常，如果一个私有方法没有被公共方法调用，那么它根本不能被调用！

我们首先注意到这个方法是一个无限循环，它将在许多通道之间重复执行选择操作，但每次只能执行其中一个。这些操作中的第一个是接收新消息以发布给订阅者的操作。`case msg := <- p.in:`代码处理这个传入的操作。

在这种情况下，我们正在遍历所有订阅者并执行它们的`Notify`方法。你可能想知道为什么我们不添加`go`关键字，以便`Notify`方法作为一个不同的 Goroutine 执行，因此迭代得更快。好吧，这是因为我们不是解耦接收消息和关闭消息的动作。所以，如果我们在一个新的 Goroutine 中启动订阅者，而它在`Notify`方法处理消息时被关闭，我们就会有一个竞态条件，其中消息将尝试在`Notify`方法中向一个已关闭的通道发送。实际上，我们在开发`Notify`方法时考虑了这种场景，但如果我们每次都调用一个新的 Goroutine 中的`Notify`方法，我们仍然无法控制启动的 Goroutine 的数量。为了简单起见，我们只是调用`Notify`方法，但控制`Notify`方法执行中等待返回的 Goroutine 数量是一个很好的练习。通过在每个订阅者中缓冲`in`通道，我们也可以实现一个好的解决方案：

```go
    case sub := <-p.addSubCh: 
    p.subscribers = append(p.subscribers, sub) 

```

下一个操作是在值到达通道并添加订阅者时应该做什么。在这种情况下很简单：我们更新它，将其新值附加到它上面。当这个案例被执行时，在这个选择中不能执行其他调用：

```go
     case sub := <-p.removeSubCh: 
     for i, candidate := range p.subscribers { 
         if candidate == sub { 
             p.subscribers = append(p.subscribers[:i], p.subscribers[i+1:]...) 
             candidate.Close() 
             break 
        } 
    } 

```

当一个值到达远程通道时，操作稍微复杂一些，因为我们必须在切片中搜索订阅者。我们使用*O(N)*方法来处理，从开始迭代直到找到它，但搜索算法可以大大改进。一旦找到相应的`Subscriber`接口，我们就从订阅者切片中移除它并停止它。有一点需要提及的是，在测试中，我们直接访问订阅者切片的长度，而没有解复用操作。这显然是一个竞争条件，但通常在运行竞争检测器时并不会反映出来。

解决方案将是开发一个专门用于解复用获取切片长度调用的方法，但它不会属于公共接口。再次强调，为了简单起见，我们将保持现状，否则这个示例可能会变得过于复杂而难以处理：

```go
    case <-p.stop: 
    for _, sub := range p.subscribers { 
        sub.Close() 
            } 

        close(p.addSubCh) 
        close(p.in) 
        close(p.removeSubCh) 

        return 
        } 
    } 
} 

```

最后一个解复用操作是`stop`操作，它必须停止发布者和订阅者中的所有 Goroutines。然后我们必须遍历存储在订阅者字段中的每个`Subscriber`，以执行它们的`Close()`方法，这样它们的 Goroutines 也会关闭。最后，如果我们返回这个 Goroutine，它也会结束。

好了，现在是时候执行所有测试，看看进展如何：

```go
go test -race .
ok

```

还不错。所有测试都成功通过，我们已经准备好了观察者模式。虽然示例还可以改进，但它是一个很好的例子，展示了我们必须如何使用 Go 中的通道来处理观察者模式。作为练习，我们鼓励你尝试使用互斥锁而不是通道来控制访问的相同示例。这会稍微简单一些，也会让你了解如何与互斥锁一起工作。

## 关于并发观察者模式的一些话

这个示例展示了如何利用多核 CPU 通过实现观察者模式来构建一个并发消息发布者。虽然示例很长，但我们试图展示在 Go 中开发并发应用程序时的一个常见模式。

# 概述

我们看到了一些开发可以并行运行并发结构的方法。我们尝试展示了解决相同问题的几种方法，一种没有并发原语，另一种有。我们看到了使用并发结构编写的发布/订阅示例与经典示例相比有多么不同。

我们还看到了如何使用管道构建并发操作，并通过使用工作池来并行化它，这是 Go 中非常常见的一种模式，旨在最大化并行性。

两个示例都足够简单，易于理解，但在尽可能深入挖掘 Go 语言本质的同时，并没有深入到问题本身。
