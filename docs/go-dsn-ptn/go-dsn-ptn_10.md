# 第十章：并发模式 - 工作者池和发布/订阅设计模式

我们已经到达了本书的最后一章，在这里我们将讨论一些具有并发结构的模式。我们将详细解释每一步，以便您可以仔细跟随示例。

这个想法是学习如何在 Go 中设计并发应用程序的模式。我们大量使用通道和 Goroutines，而不是锁或共享变量。

+   我们将看一种开发工作者池的方法。这对于控制执行中的 Goroutines 数量非常有用。

+   第二个例子是对观察者模式的重写，我们在第七章中看到了，*行为模式 - 访问者、状态、中介者和观察者设计模式*，使用并发结构编写。通过这个例子，我们将更深入地了解并发结构，并看看它们与常规方法有何不同。

# 工作者池

我们可能会遇到的一个问题是，以前的一些并发方法的上下文是无限的。我们不能让一个应用程序创建无限数量的 Goroutines。Goroutines 很轻，但它们执行的工作可能非常繁重。工作者池帮助我们解决了这个问题。

## 描述

通过一组工作者，我们希望限制可用的 Goroutines 数量，以便更深入地控制资源池。通过为每个工作者创建一个通道，并使工作者处于空闲或繁忙状态，这很容易实现。这项任务可能看起来令人生畏，但实际上并不是。

## 目标

创建一个工作者池主要是关于资源控制：CPU、RAM、时间、连接等等。工作者池设计模式帮助我们做到以下几点：

+   使用配额控制对共享资源的访问

+   为每个应用程序创建有限数量的 Goroutines

+   为其他并发结构提供更多的并行能力

## 一组管道

在上一章中，我们看到了如何使用管道。现在我们将启动有限数量的管道，以便 Go 调度器可以尝试并行处理请求。这里的想法是控制 Goroutines 的数量，在应用程序完成时优雅地停止它们，并使用并发结构最大化并行性，而不会出现竞争条件。

我们将使用的管道类似于我们在上一章中使用的管道，那里我们生成数字，将它们提升到 2 的幂，并求和最终结果。在这种情况下，我们将传递字符串，然后附加和前缀数据。

## 验收标准

从业务角度来看，我们希望有一些东西告诉我们，工作者已经处理了一个请求，有一个预定义的结尾，并且传入的数据被解析为大写：

1.  当使用一个字符串值（任意）发出请求时，它必须是大写的。

1.  一旦字符串变成大写，预定义的文本必须附加到它上面。这个文本不应该是大写的。

1.  使用前面的结果，工作者 ID 必须添加到最终字符串的前缀。

1.  结果字符串必须传递给预定义的处理程序。

我们还没有讨论如何在技术上实现，只是业务需求。通过整个描述，我们至少会有工作者、请求和处理程序。

## 实施

最开始是一个请求类型。根据描述，它必须包含将进入管道的字符串以及处理程序函数：

```go
   // workers_pipeline.go file 
    type Request struct { 
          Data    interface{} 
          Handler RequestHandler 
    } 

```

“字符串”在哪里？我们有一个`Data`字段，类型为`interface{}`，所以我们可以用它来传递一个字符串。通过使用接口，我们可以重用这种类型来传递一个`string`，一个`int`，或一个`struct`数据类型。接收者必须知道如何处理传入的接口。

“处理程序”字段的类型是“请求”处理程序，我们还没有定义：

```go
type RequestHandler func(interface{}) 

```

请求处理程序是接受接口作为其第一个参数并返回空的任何函数。再次看到`interface{}`，在这里我们通常会看到一个字符串。这是我们之前提到的接收器之一，我们需要用它来转换传入的结果。

因此，当发送请求时，我们必须在`Data`字段中填充一些值并实现处理程序；例如：

```go
func NewStringRequest(s string, id int, wg *sync.WaitGroup) Request { 
    myRequest := Request{ 
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

处理程序是通过闭包定义的。我们再次检查接口的类型（并在最后推迟调用`Done()`方法）。如果接口不正确，我们只需打印其内容并返回。如果转换正确，我们也会打印它们，但这是我们通常会对操作结果做一些事情的地方；我们必须使用类型转换来检索`interface{}`的内容（这是一个字符串）。尽管这会引入一些开销，但这必须在管道的每一步中完成。

现在我们需要一个可以处理`Request`类型的类型。可能的实现方式几乎是无限的，因此最好先定义一个接口：

```go
   // worker.go file 
    type WorkerLauncher interface { 
        LaunchWorker(in chan Request) 
    } 

```

`WorkerLauncher`接口只需实现`LaunchWorker(chan Request)`方法。任何实现此接口的类型都必须接收一个`Request`类型的通道来满足它。这种类型的`Request`通道是管道的唯一入口点。

### 调度程序

现在，为了并行启动工作程序并处理所有可能的传入通道，我们需要类似于调度程序的东西：

```go
   // dispatcher.go file 
    type Dispatcher interface { 
        LaunchWorker(w WorkerLauncher) 
        MakeRequest(Request) 
        Stop() 
    } 

```

`Dispatcher`接口可以在其自己的`LaunchWorker`方法中启动注入的`WorkerLaunchers`类型。`Dispatcher`接口必须使用任何`WorkerLauncher`类型的`LaunchWorker`方法来初始化管道。这样我们就可以重用`Dispatcher`接口来启动许多类型的`WorkerLaunchers`。

在使用`MakeRequest(Request)`时，`Dispatcher`接口公开了一个很好的方法，可以将新的`Request`注入到工作池中。

最后，用户必须在所有 Goroutines 都完成时调用 stop。我们必须在应用程序中处理优雅的关闭，并且我们希望避免 Goroutine 泄漏。

我们有足够的接口，所以让我们从稍微不那么复杂的调度程序开始：

```go
    type dispatcher struct { 
        inCh chan Request 
    } 

```

我们的`dispatcher`结构在其字段中存储了一个`Request`类型的通道。这将是任何管道中请求的唯一入口点。我们说它必须实现三种方法，如下所示：

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

在这个例子中，`Dispatcher`接口在启动工作程序之前不需要对自身执行任何特殊操作，因此`Dispatcher`上的`LaunchWorker`方法只是执行传入`WorkerLauncher`的`LaunchWorker`方法，后者也有一个`LaunchWorker`方法来初始化自身。我们之前已经定义了`WorkerLauncher`类型至少需要一个 ID 和一个传入请求的通道，所以这就是我们传递的内容。

在`Dispatcher`接口中实现`LaunchWorker`方法似乎是不必要的。在不同的场景中，保存运行中的工作程序 ID 在调度程序中控制哪些工作程序正在运行或关闭可能是有趣的；这个想法是隐藏启动实现细节。在这种情况下，`Dispatcher`接口只是作为一个 Facade 设计模式，隐藏了一些实现细节。

第二种方法是`Stop`。它关闭了传入请求通道，引发了一连串的反应。我们在管道示例中看到，当关闭传入通道时，每个 Goroutines 中的 for-range 循环都会中断，并且 Goroutine 也会结束。在这种情况下，当关闭共享通道时，它会引发相同的反应，但是在每个监听 Goroutine 中，因此所有管道都将停止。酷，对吧？

请求的实现非常简单；我们只需将请求作为参数传递给传入请求的通道。Goroutine 将永远阻塞在那里，直到通道的另一端检索到请求。永远？如果发生了什么事情，这似乎很长。我们可以引入一个超时，如下所示：

```go
    func (d *dispatcher) MakeRequest(r Request) { 
        select { 
        case d.inCh <- r: 
        case <-time.After(time.Second * 5): 
            return 
        } 
    } 

```

如果你还记得之前的章节，我们可以使用 select 来控制对通道的操作。就像 `switch` 语句一样，只能执行一个操作。在这种情况下，我们有两种不同的操作：发送和接收。

第一种情况是发送操作——尝试发送这个，它将在那里阻塞，直到有人在通道的另一侧取走值。然后并没有太大的改进。第二种情况是接收操作；如果无法成功发送大写请求，它将在 5 秒后触发，并返回。在这里返回错误会非常方便，但为了简单起见，我们将留空。

最后，在调度程序中，为了方便起见，我们将定义一个 `Dispatcher` 创建者：

```go
    func NewDispatcher(b int) Dispatcher { 
        return &dispatcher{ 
            inCh:make(chan Request, b), 
        } 
    } 

```

通过使用这个函数而不是手动创建调度程序，我们可以简单地避免小错误，比如忘记初始化通道字段。正如你所看到的，`b` 参数指的是通道中的缓冲区大小。

### 流水线

所以，我们的调度程序已经完成，我们需要开发验收标准中描述的流水线。首先，我们需要一个类型来实现 `WorkerLauncher` 类型：

```go
   // worker.go file 
    type PreffixSuffixWorker struct { 
        id int 
        prefixS string 
        suffixS string 
    } 

    func (w *PreffixSuffixWorker) LaunchWorker(i int, in chan Request) {} 

```

`PreffixSuffixWorker` 变量存储一个 ID，一个要添加的字符串，以及 `Request` 类型的传入数据的另一个字符串。因此，要添加的值和后缀将在这些字段中是静态的，并且我们将从这里获取它们。

我们将稍后实现 `LaunchWorker` 方法，并从流水线的每一步开始。根据*第一个验收标准*，传入的字符串必须是大写。因此，大写方法将是我们流水线中的第一步：

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

很好。就像在前一章中一样，流水线中的一步接受传入数据的通道，并返回相同类型的通道。它的方法与我们在前一章中开发的示例非常相似。不过，这一次，我们没有使用包函数，大写是 `PreffixSuffixWorker` 类型的一部分，传入的数据是一个 `struct` 而不是一个 `int`。

`msg` 变量是一个 `Request` 类型，它将具有一个处理函数和一个接口形式的数据。`Data` 字段应该是一个字符串，所以我们在使用它之前对其进行类型转换。当对值进行类型转换时，我们将收到请求类型的相同值和一个 `true` 或 `false` 标志（由 `ok` 变量表示）。如果 `ok` 变量为 `false`，则转换无法完成，我们将不会将该值传递到流水线中。我们通过向处理程序发送 `nil` 来停止这个 `Request`（这也会引发类型转换错误）。

一旦我们在 `s` 变量中有一个好的字符串，我们就可以将其大写并再次存储在 `Data` 字段中，以便将其发送到流水线的下一步。请注意，该值将再次作为接口发送，因此下一步将需要再次对其进行转换。这是使用这种方法的缺点。

第一步完成后，让我们继续进行第二步。根据*第二个验收标准*，现在必须添加预定义的文本。这个文本是存储在 `suffixS` 字段中的文本：

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

`append` 函数的结构与 `uppercase` 函数相同。它接收并返回一个传入请求的通道，并启动一个新的 Goroutine，该 Goroutine 遍历传入的通道直到它关闭。我们需要对传入的值进行类型转换，如前所述。

在流水线中的这一步中，传入的字符串是大写的（在进行类型断言后）。要附加任何文本，我们只需要使用 `fmt.Sprintf()` 函数，就像我们之前做过很多次一样，它使用提供的数据格式化一个新的字符串。在这种情况下，我们将 `suffixS` 字段的值作为第二个值传递，以将其附加到字符串的末尾。

流水线中只缺少最后一步，即前缀操作：

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

这个函数中吸引你注意力的是什么？是的，它现在不返回任何通道。我们可以以两种方式完成整个管道。我想你已经意识到我们使用了`Future`处理函数来执行管道中的最终结果。第二种方法是传递一个通道将数据返回到其原始位置。在某些情况下，Future 就足够了，而在其他情况下，将通道传递可能更方便，以便它可以连接到不同的管道（例如）。

无论如何，管道中的一步的结构对您来说应该已经非常熟悉了。我们对值进行转换，检查转换的结果，并在出现问题时向处理程序发送 nil。但是，如果一切顺利，最后要做的就是再次格式化文本，将`prefixS`字段放在文本开头，通过调用请求的处理程序将生成的字符串发送回原始位置。

现在，几乎完成了我们的工作程序，我们可以实现`LaunchWorker`方法：

```go
    func (w *PreffixSuffixWorker) LaunchWorker(in chan Request) { 
        w.prefix(w.append(w.uppercase(in))) 
    } 

```

工作程序就是这些了！我们只需将返回的通道传递给管道中的下一步，就像我们在上一章中所做的那样。请记住，管道是从调用内部到外部执行的。那么，任何传入管道的数据的执行顺序是什么？

1.  数据通过`uppercase`方法中启动的 Goroutine 进入管道。

1.  然后，它进入了在`append`中启动的 Goroutine。

1.  最后，它进入了在`prefix`方法中启动的 Goroutine，它不返回任何东西，但在给传入的字符串加上更多数据后执行处理程序。

现在我们有了一个完整的管道和一个管道的分发器。分发器将启动我们想要的管道实例，将传入的请求路由到任何可用的工作程序。

如果没有任何工作程序在 5 秒内接受请求，请求将丢失。

让我们在一个小应用程序中使用这个库。

## 使用工作程序池的应用程序

我们将启动我们定义的管道的三个工作程序。我们使用`NewDispatcher`函数创建分发器和接收所有请求的通道。该通道具有固定的缓冲区，可以在阻塞之前存储多达 100 条传入消息：

```go
   // workers_pipeline.go 
    func main() { 
        bufferSize := 100 
        var dispatcher Dispatcher = NewDispatcher(bufferSize) 

```

然后，我们将通过在`Dispatcher`接口中三次调用`LaunchWorker`方法，并使用已填充的`WorkerLauncher`类型来启动工作程序：

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

每个`WorkerLauncher`类型都是`PreffixSuffixWorker`的一个实例。前缀将是一个显示工作程序 ID 和后缀文本`world`的小文本。

此时，我们有三个工作程序，每个都有三个 Goroutines，同时运行并等待消息到达：

```go
    requests := 10 

    var wg sync.WaitGroup 
    wg.Add(requests) 

```

我们将发出 10 个请求。我们还需要一个 WaitGroup 来正确同步应用程序，以免过早退出。在处理并发应用程序时，您可能会经常使用 WaitGroups。对于 10 个请求，我们需要等待 10 次对`Done()`方法的调用，因此我们使用*delta*为 10 调用`Add()`方法。它被称为 delta，因为您稍后也可以传递-5 以使其在五个请求中保持。在某些情况下，这可能很有用：

```go
    for i := 0; i < requests; i++ { 
        req := NewStringRequest("(Msg_id: %d) -> Hello", i, &wg) 
        dispatcher.MakeRequest(req) 
    } 

    dispatcher.Stop() 

    wg.Wait() 
}
```

为了发出请求，我们将迭代一个`for`循环。首先，我们使用在实现部分开头编写的`NewStringRequest`函数创建一个`Request`。在这个值中，`Data`字段将是我们将通过管道传递的文本，它将是附加和后缀操作的“中间”文本。在这种情况下，我们将发送消息编号和单词`hello`。

一旦我们有一个请求，我们就用它调用`MakeRequest`方法。完成所有请求后，我们停止分发器，如前所述，这将引发一个连锁反应，将停止管道中的所有 Goroutines。

最后，我们等待组，以便接收对`Done()`方法的所有调用，这表明所有操作都已完成。是时候试一试了：

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

1.  然后，文本被转换为大写，所以现在我们有`(MSG_ID: 0) -> HELLO`。

1.  在将文本`world`（注意文本开头的空格）转换为大写后进行附加操作。这将给我们文本`(MSG_ID: 0) -> HELLO World`。

1.  最后，文本`WorkerID: 1`（在这种情况下，第一个工作者接受了任务，但也可能是任何一个工作者）被附加到步骤 3 的文本中，给我们完整的返回消息，`WorkerID: 1 -> (MSG_ID: 0) -> HELLO World`。

## 没有测试？

并发应用程序很难测试，特别是如果你正在进行网络操作。测试可能会很困难，代码可能会发生很大变化。无论如何，不进行测试是不可接受的。在这种情况下，测试我们的小应用并不特别困难。创建一个测试，将`main`函数的内容复制/粘贴到那里：

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

现在我们必须重写我们的处理程序，以测试返回的内容是否符合我们的期望。转到`for`循环，修改我们作为每个`Request`处理程序传递的函数：

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

我们将使用正则表达式来测试业务。如果你不熟悉正则表达式，它们是一个非常强大的功能，可以帮助你在字符串中匹配内容。如果你还记得我们在练习中使用`strings`包时。`Contains`是用来在字符串中查找文本的函数。我们也可以用正则表达式来做。

问题在于正则表达式非常昂贵，消耗大量资源。

我们正在使用`regexp`包的`Match`函数来提供一个匹配模板。我们的模板是`WorkerID\: \d* -> \(MSG_ID: \d\) -> [A-Z]*\sWorld`（不带引号）。具体来说，它描述了以下内容：

+   一个包含内容`WorkerID: \d* -> (MSG_ID: \d*`的字符串，这里的`\d*`表示任意数字写零次或多次，因此它将匹配`WorkerID: 10 -> (MSG_ID: 1"`和`"WorkerID: 1 -> (MSG_ID: 10`。

+   `"\) -> [A-Z]*\sWorld"`（括号必须使用反斜杠进行转义）。"`*`"表示任意大写字母写零次或多次，所以`"\s"`是一个空格，它必须以文本`World`结束，所以`) -> HELLO World"`会匹配，但`) -> Hello World"`不会，因为`"Hello`必须全部大写。

运行这个测试给我们以下输出：

```go
go test -v .
=== RUN   Test_Dispatcher
--- PASS: Test_Dispatcher (0.00s)
PASS
ok

```

不错，但我们没有测试代码是否在并发执行，所以这更像是业务测试而不是单元测试。并发测试会迫使我们以完全不同的方式编写代码，以检查它是否创建了适当数量的 Goroutines，并且流水线是否遵循了预期的工作流程。这并不是坏事，但它非常复杂，超出了本书的范围。

## 总结工作池

有了工作池，我们有了第一个可以在真实生产系统中使用的复杂并发应用程序。它还有改进的空间，但它是一个非常好的设计模式，可以构建并发有界应用程序。

非常重要的是，我们始终要控制正在启动的 Goroutines 的数量。虽然很容易启动成千上万个 Goroutines 来实现更多的并行性，但我们必须非常小心，确保它们没有代码会使它们陷入无限循环。

有了工作池，我们现在可以将一个简单操作分解为许多并行任务。想想看；这可以通过一次简单的`fmt.Printf`调用来实现相同的结果，但我们已经用它做了一个流水线；然后，我们启动了几个这样的流水线实例，最后，在所有这些管道之间分配了工作负载。

# 并发发布/订阅设计模式

在这一部分，我们将实现我们之前在行为模式中展示的观察者设计模式，但采用并发结构和线程安全。

## 描述

如果你还记得之前的解释，观察者模式维护了一个想要被通知特定事件的观察者或订阅者列表。在这种情况下，每个订阅者将在不同的 Goroutine 中运行，就像发布者一样。我们将在构建这个结构时遇到新的问题：

+   现在，订阅者列表的访问必须是串行化的。如果我们用一个 Goroutine 读取列表，我们就不能从中删除一个订阅者，否则就会出现竞争。

+   当一个订阅者被移除时，订阅者的 Goroutine 也必须被关闭，否则它将永远迭代下去，我们将遇到 Goroutine 泄漏的问题。

+   当停止发布者时，所有订阅者也必须停止它们的 Goroutines。

## 目标

这个发布/订阅的目标与我们在观察者模式中写的目标相同。这里的区别在于我们将如何开发它。这个想法是创建一个并发结构来实现相同的功能，即：

+   提供一个事件驱动的架构，其中一个事件可以触发一个或多个动作

+   将执行的动作与触发它们的事件解耦

+   提供多个触发相同动作的源事件

这个想法是解耦发送者和接收者，隐藏发送者处理其事件的接收者的身份，并隐藏接收者可以与之通信的发送者的数量。

特别是，如果我在某个应用程序的按钮上开发了一个点击，它可能会执行一些操作（比如在某处登录）。几周后，我们可能决定也让它显示一个弹出窗口。如果每次我们想要为这个按钮添加一些功能，我们都必须更改处理点击操作的代码，那么这个函数将变得非常庞大，而且对其他项目的可移植性也不是很好。如果我们为每个动作使用一个发布者和一个观察者，点击函数只需要使用一个发布者发布一个事件，每次我们想要改进功能时，我们只需要为这个事件编写订阅者。这在用户界面应用程序中尤为重要，因为在单个 UI 操作中要做的事情很多，可能会减慢界面的响应速度，完全破坏用户体验。

通过使用并发结构来开发观察者模式，如果定义了并发结构并且设备允许我们执行并行任务，UI 就无法感知到后台执行的所有任务。

## 示例 - 并发通知器

我们将开发一个类似于我们在第七章中开发的*notifier*，*行为模式 - 访问者、状态、中介者和观察者设计模式*。这是为了专注于结构的并发性质，而不是详细说明已经解释过的太多东西。我们已经开发了一个观察者，所以我们对这个概念很熟悉。

这个特定的通知器将通过传递`interface{}`值来工作，就像在工作池示例中一样。这样，我们可以通过在接收器上进行转换来将其用于多个类型，引入一些开销。

现在我们将使用两个接口。首先是`Subscriber`接口：

```go
    type Subscriber interface { 
        Notify(interface{}) error 
        Close() 
    } 

```

就像在之前的示例中一样，`Subscriber`接口必顶有一个`Notify`方法来通知新事件。这是接受`interface{}`值并返回错误的`Notify`方法。然而，`Close()`方法是新的，它必须触发停止订阅者正在监听新事件的 Goroutine 的任何操作。

第二个和最终的接口是`Publisher`接口：

```go
    type Publisher interface { 
        start() 
        AddSubscriberCh() chan<- Subscriber 
        RemoveSubscriberCh() chan<- Subscriber 
        PublishingCh() chan<- interface{} 
        Stop() 
    } 

```

`Publisher`接口具有与发布者相同的操作，但是用于与通道一起工作。`AddSubscriberCh`和`RemoveSubscriberCh`方法接受`Subscriber`接口（满足`Subscriber`接口的任何类型）。它必须有一种发布消息的方法和一个`Stop`方法来停止它们所有（发布者和订阅者 Goroutines）

## 验收标准

这个例子和第七章中的例子之间的要求*，行为模式 - 访问者，状态，中介者和观察者设计模式*不得更改。在这两个例子中，目标是相同的，因此要求也必须相同。在这种情况下，我们的要求是技术性的，因此我们实际上需要添加一些更多的验收标准：

1.  我们必须有一个带有`PublishingCh`方法的发布者，该方法返回一个通道以发送消息，并在每个订阅的观察者上触发`Notify`方法。

1.  我们必须有一种方法来向发布者添加新的订阅者。

1.  我们必须有一种方法来从发布者中移除新的订阅者。

1.  我们必须有一种方法来停止订阅者。

1.  我们必须有一种方法来停止`Publisher`接口，这也将停止所有订阅者。

1.  所有 Goroutine 之间的通信必顶是同步的，以便没有 Goroutine 被锁定等待响应。在这种情况下，超时后将返回错误。

嗯，这些标准似乎相当令人生畏。我们忽略了一些要求，这些要求将增加更多的复杂性，例如删除不响应的订阅者或检查以确保发布者 Goroutine 始终处于活动状态。

## 单元测试

我们之前提到测试并发应用程序可能很困难。通过正确的机制，仍然可以完成，因此让我们看看在没有大麻烦的情况下我们可以测试多少。

### 测试订阅者

从订阅者开始，它似乎具有更加封装的功能，第一个订阅者必须将来自发布者的传入消息打印到`io.Writer`接口。我们已经提到订阅者具有一个接口，其中包含两种方法，`Notify(interface{}) error`和`Close()`方法：

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

好的。这将是我们的`writer_sub.go`文件。创建相应的测试文件，称为`writer_sub_test.go`文件：

```go
    package main 
    func TestStdoutPrinter(t *testing.T) { 

```

现在，我们面临的第一个问题是功能打印到`stdout`，因此没有返回值可供检查。我们可以用三种方式解决它：

+   捕获`stdout`方法。

+   注入一个`io.Writer`接口进行打印。这是首选解决方案，因为它使代码更易管理。

+   将`stdout`方法重定向到不同的文件。

我们将采用第二种方法。重定向也是一种可能性。`os.Stdout`是指向`os.File`类型的指针，因此它涉及用我们控制的文件替换此文件，并从中读取：

```go
    func TestWriter(t *testing.T) { 
        sub := NewWriterSubscriber(0, nil) 

```

`NewWriterSubscriber`订阅者尚未定义。它必须帮助创建特定的订阅者，返回一个满足`Subscriber`接口的类型，因此让我们快速在`writer_sub.go`文件中声明它：

```go
    func NewWriterSubscriber(id int, out io.Writer) Subscriber { 
        return &writerSubscriber{} 
    } 

```

理想情况下，它必须接受一个 ID 和一个`io.Writer`接口作为其写入的目的地。在这种情况下，我们需要一个自定义的`io.Writer`接口进行测试，因此我们将在`writer_sub_test.go`文件中为其创建一个`mockWriter`：

```go
    type mockWriter struct { 
        testingFunc func(string) 
    } 

    func (m *mockWriter) Write(p []byte) (n int, err error) { 
        m.testingFunc(string(p)) 
        return len(p), nil 
    } 

```

`mockWriter`结构将接受`testingFunc`作为其字段之一。这个`testingFunc`字段接受一个代表写入到`mockWriter`结构的字节的字符串。为了实现`io.Writer`接口，我们需要定义一个`Write([]byte) (int, error)`方法。在我们的定义中，我们将`p`的内容作为字符串传递（请记住，我们总是需要在每个`Write`方法中返回读取的字节和错误，或者不需要返回）。这种方法将`testingFunc`的定义委托给测试的范围。

我们将在 `Subcriber` 接口上调用 `Notify` 方法，它必须像 `mockWriter` 结构一样写入 `io.Writer` 接口。因此，在调用 `Notify` 方法之前，我们将定义 `mockWriter` 结构的 `testingFunc`：

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

我们将发送 `Hello` 消息。这也意味着无论 `Subscriber` 接口做什么，它最终都必须在提供的 `io.Writer` 接口上打印 `Hello` 消息。

因此，如果最终我们在测试函数中收到一个字符串，我们将需要与 `Subscriber` 接口同步，以避免测试中的竞争条件。这就是为什么我们使用了这么多的 `WaitGroup`。这是一种非常方便和易于使用的类型，用于处理这种情况。一个 `Notify` 方法调用将需要等待一个 `Done()` 方法的调用，因此我们调用 `Add(1)` 方法（一个单位）。

理想情况下，`NewWriterSubscriber` 函数必须返回一个接口，因此我们需要对我们在测试中使用的类型进行类型断言，即 `stdoutPrinter` 方法。我故意在进行转换时省略了错误检查，只是为了简化事情。一旦我们有了 `writerSubscriber` 类型，我们就可以访问其 `Write` 字段，将其替换为 `mockWriter` 结构。我们本可以直接在 `NewWriterSubscriber` 函数上传递一个 `io.Writer` 接口，但这样我们就无法覆盖传递空对象并将 `os.Stdout` 实例设置为默认值的情况。

因此，测试函数最终将接收一个包含订阅者写入内容的字符串。我们只需要检查接收到的字符串，即 `Subscriber` 接口将接收到的字符串，是否在某个时刻打印了单词 `Hello`，而 `strings.Contains` 函数最适合这种情况。一切都在测试函数的范围内定义，因此我们可以使用 `t` 对象的值来表示测试失败。

一旦我们完成了检查，我们必须调用 `Done()` 方法来表示我们已经测试了预期的结果：

```go
err := sub.Notify(msg) 
if err != nil { 
    t.Fatal(err) 
    } 

    wg.Wait() 
    sub.Close() 
} 

```

我们实际上必须调用 `Notify` 和 `Wait` 方法，以便调用 `Done` 方法来检查一切是否正确。

### 注意

您是否意识到我们在测试中定义了行为，更多或更少是相反的？这在并发应用程序中非常常见。有时可能会令人困惑，因为如果我们无法线性地跟踪调用，就很难知道函数可能在做什么，但您会很快习惯。与其思考“它这样做，然后这样做，然后那样做”，不如思考“在执行那个时会调用这个”。这也是因为在并发应用程序中，执行顺序在某一点之前是未知的，除非我们使用同步原语（如 WaitGroups 和通道）在某些时刻暂停执行。

现在让我们执行此类型的测试：

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

它退出得很快，但失败了。实际上，`Done()` 方法的调用尚未执行，因此最好将我们测试的最后部分更改为这样：

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

现在，它不会停止执行，因为我们调用 `Error` 函数而不是 `Fatal` 函数，但我们调用 `Done()` 方法，测试在我们希望的位置结束，即在调用 `Wait()` 方法之后。您可以尝试再次运行测试，但输出将是相同的。

### 测试发布者

我们已经看到了 `Publisher` 接口和将满足其条件的类型，即 `publisher` 类型。我们唯一确定的是它将需要一些存储订阅者的方式，因此它至少会有一个 `Subscribers` 切片：

```go
    // publisher.go type 
    type publisher struct { 
        subscribers []Subscriber 
    } 

```

为了测试 `publisher` 类型，我们还需要一个 `Subscriber` 接口的模拟：

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

`mockSubscriber` 类型必须实现 `Subscriber` 接口，因此它必须有 `Close()` 和 `Notify(interface{}) error` 方法。我们可以嵌入一个已实现它的现有类型，比如 `writerSubscriber`，并且只覆盖我们感兴趣的方法，但我们需要定义两者，所以我们不会嵌入任何东西。

因此，在这种情况下，我们需要重写`Notify`和`Close`方法，以调用`mockSubscriber`类型字段上存储的测试函数。

```go
    func TestPublisher(t *testing.T) { 
        msg := "Hello" 

        p := NewPublisher() 

```

首先，我们将直接通过通道发送消息，这可能会导致潜在的意外死锁，因此首先要定义一个用于这种情况的 panic 处理程序，比如，向关闭的通道发送消息或没有 Goroutines 在通道上监听。我们将发送给订阅者的消息是`Hello`。因此，通过使用`AddSubscriberCh`方法返回的通道接收到的每个订阅者都必须接收到这条消息。我们还将使用一个*New*函数来创建发布者，称为`NewPublisher`。现在更改`publisher.go`文件来编写它。

```go
   // publisher.go file 
    func NewPublisher() Publisher { 
        return &publisher{} 
    } 

```

现在我们将定义`mockSubscriber`，并将其添加到已知订阅者列表中。回到`publisher_test.go`文件。

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

像往常一样，我们从一个 WaitGroup 开始。首先，在我们的订阅者中测试函数会在执行结束时延迟调用`Done()`方法。然后它需要对`msg`变量进行类型转换，因为它是作为接口传递的。记住，这样一来，我们可以通过引入类型断言的开销，使用`Publisher`接口与许多类型。这是在第`s, ok := msg.(string)`行完成的。

一旦我们将`msg`强制转换为字符串`s`，我们只需要检查订阅者接收到的值是否与我们发送的值相同，如果不同则测试失败。

```go
        p.AddSubscriberCh() <- sub 
        wg.Add(1) 

        p.PublishingCh() <- msg 
        wg.Wait() 

```

我们使用`AddSubscriberCh`方法添加`mockSubscriber`类型。在准备就绪后，我们通过将`WaitGroup`加一来发布我们的消息，并在将`WaitGroup`设置为等待之前发布消息，这样测试就不会继续进行，直到`mockSubscriber`类型调用`Done()`方法。

此外，我们需要检查在调用`AddSubscriberCh`方法后`Subscriber`接口的数量是否增加，因此我们需要在测试中获取发布者的具体实例。

```go
        pubCon := p.(*publisher) 
        if len(pubCon.subscribers) != 1 { 
            t.Error("Unexpected number of subscribers") 
        } 

```

类型断言是我们今天的朋友！一旦我们有了具体类型，我们就可以访问`Publisher`接口的基础订阅者切片。调用`AddSubscriberCh`方法后，订阅者的数量必须为 1，否则测试将失败。下一步是检查相反的情况--当我们移除一个`Subscriber`接口时，它必须从这个列表中被移除。

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

我们测试的最后一步是停止发布者，这样就不能再发送消息，所有的 Goroutines 都会停止。

测试已经完成，但在`publisher`类型实现了所有方法之前我们无法运行测试；这必须是最终结果。

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

有了这个空实现，当运行测试时就不会发生什么好事。

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

是的，它失败了，但这根本不是一个受控的失败。这是故意做的，以展示在 Go 中需要注意的一些事情。首先，这个测试产生的错误是一个**fatal**错误，通常指向代码中的一个 bug。这很重要，因为虽然**panic**错误可以被恢复，但对于致命错误却不能做同样的事情。

在这种情况下，错误告诉我们问题所在：`goroutine 5 [chan send (nil chan)]`，一个`nil`通道，所以这实际上是我们代码中的一个错误。我们该如何解决这个问题呢？这也很有趣。

我们有一个`nil`通道的事实是由我们编写的代码导致的，用于编译单元测试，但一旦编写了适当的代码，就不会引发这种特定的错误（因为在这种情况下我们永远不会返回一个`nil`通道）。我们可以返回一个从未使用的通道，这会导致死锁的致命错误，这也不会有任何进展。

一种解决方法是返回一个通道和一个错误，这样你就可以有一个错误包，其中包含一个实现了`Error`接口的类型，返回一个特定的错误，比如`NoGoroutinesListening`或`ChannelNotCreated`。我们已经看到了许多这样的实现，所以我们将把这些留给读者作为练习，然后我们将继续保持对本章并发性质的关注。

没有什么令人惊讶的，所以我们可以进入实施阶段。

## 实施

回顾一下，`writerSubscriber`必须接收它将写入的消息，并将其写入满足`io.Writer`接口的类型。

那么，我们从哪里开始呢？嗯，每个订阅者将运行自己的 Goroutine，而我们已经知道与 Goroutine 通信的最佳方法是使用通道。因此，我们将需要在`Subscriber`类型中使用一个带有通道的字段。我们可以使用与管道中相同的方法来结束`NewWriterSubscriber`函数和`writerSubscriber`类型：

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

在第一步中，如果没有指定写入器（`out`参数为 nil），默认的`io.Writer`接口是`stdout`。然后，我们使用传递给第一个参数的 ID、out 的值（`os.Stdout`，或者如果不为 nil，则是传入参数中的值），以及一个名为 in 的通道创建了一个新的指向`writerSubscriber`类型的指针，以保持与之前示例中相同的命名。

然后我们启动一个新的 Goroutine；这是我们提到的启动机制。就像在管道中一样，订阅者将在每次接收到新消息时迭代`in`通道，并将其内容格式化为一个字符串，其中还包含当前订阅者的 ID。

正如我们之前学到的那样，如果`in`通道关闭，`for range`循环将停止，那个特定的 Goroutine 将结束，所以`Close`方法中唯一需要做的事情就是实际关闭`in`通道：

```go
    func (s *writerSubscriber) Close() { 
        close(s.in) 
    } 

```

好的，只剩下`Notify`方法了；`Notify`方法是管理特定行为的便捷方法，我们将使用一个在许多调用中常见的模式：

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

在与通道通信时，通常有两种行为需要控制：一种是等待时间，另一种是通道关闭时的行为。延迟函数实际上适用于函数内部可能发生的任何恐慌错误。如果 Goroutine 发生恐慌，它仍将使用`recover()`方法执行延迟函数。`recover()`方法返回一个接口，无论错误是什么，所以在我们的情况下，我们将返回变量错误设置为`recover`返回的格式化值（这是一个接口）。`"%#v"`参数在将任何类型格式化为字符串时为我们提供了大部分信息。返回的错误会很丑陋，但它将包含我们可以提取的大部分关于错误的信息。例如，对于关闭的通道，它将返回"send on a closed channel"。嗯，这似乎足够清楚了。

第二条规则是关于等待时间。当我们通过通道发送一个值时，我们将被阻塞，直到另一个 Goroutine 从中取出该值（填充的缓冲通道也是如此）。我们不希望永远被阻塞，所以我们通过使用 select 处理程序设置了一秒的超时期。简而言之，使用 select 我们在说：要么你在 1 秒内取出值，要么我将丢弃它并返回一个错误。

我们有`Close`、`Notify`和`NewWriterSubscriber`方法，所以我们可以再次尝试我们的测试：

```go
go test -run=TestWriter -v .
=== RUN   TestWriter
--- PASS: TestWriter (0.00s)
PASS
ok

```

现在好多了。`Writer`已经接收了我们在测试中编写的模拟写入器，并向其传递了我们传递给`Notify`方法的值。同时，由于`Notify`方法在调用`Close`方法后返回了一个错误，所以`close`可能已经有效地关闭了通道。需要提到的一件事是，我们无法在不与其交互的情况下检查通道是否关闭；这就是为什么我们必须推迟执行一个将检查`Notify`方法中`recover()`函数的内容的闭包的执行。

### 实施发布者

好的，发布者还需要一个启动机制，但要处理的主要问题是访问订阅者列表时的竞争条件。我们可以使用`sync`包中的 Mutex 对象来解决这个问题，但我们已经知道如何使用它，所以我们将改用通道。

在使用通道时，我们将需要为每个可能被视为危险的操作创建一个通道——添加订阅者、移除订阅者、检索订阅者列表以通知`Notify`方法发送消息，以及一个用于停止所有订阅者的通道。我们还需要一个用于传入消息的通道：

```go
    type publisher struct { 
        subscribers []Subscriber 
        addSubCh    chan Subscriber 
        removeSubCh chan Subscriber 
        in          chan interface{} 
        stop        chan struct{} 
    } 

```

名称是自描述的，但简而言之，订阅者维护订阅者列表；这是需要多路访问的切片。`addSubCh`实例是与之通信的通道，当您想要添加新的订阅者时，就会使用它；这就是为什么它是一个订阅者的通道。相同的解释也适用于`removeSubCh`通道，但这个通道是用来移除订阅者的。`in`通道将处理必须广播给所有订阅者的传入消息。最后，当我们想要终止所有 Goroutines 时，必须调用 stop 通道。

好的，让我们从`AddSubscriberCh`、`RemoveSubscriber`和`PublishingCh`方法开始，这些方法必须返回通道以添加和移除订阅者以及向所有订阅者发送消息的通道：

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

`Stop()`函数通过关闭`stop`通道来停止它。这将有效地向每个正在监听的 Goroutine 传播信号：

```go
func (p *publisher) Stop(){ 
  close(p.stop) 
} 

```

`Stop`方法，用于停止发布者和订阅者，也会推送到其相应的通道，称为 stop。

也许你会想知道为什么我们不直接保留通道，让用户直接向该通道推送消息，而不使用代理函数。嗯，这样做的想法是，集成库到他们的应用程序中的用户不必处理与库相关的并发结构的复杂性，因此他们可以专注于他们的业务，同时尽可能地提高性能。

### 处理通道而不产生竞争条件。

到目前为止，我们已经将数据转发到发布者的通道上，但实际上我们还没有处理任何数据。将启动不同的 Goroutine 的机制将处理它们所有。

我们将创建一个`launch`方法，通过使用`go`关键字来执行它，而不是将整个函数嵌入`NewPublisher`函数中：

```go
func (p *publisher) start() { 
  for { 
    select { 
    case msg := <-p.in: 
      for _, ch := range p.subscribers { 
        sub.Notify(msg) 
      } 

```

`Launch`是一个私有方法，我们还没有测试过它。请记住，私有方法通常是从公共方法（我们已经测试过的方法）中调用的。通常情况下，如果一个私有方法没有从公共方法中调用，它就不能被调用！

使用这种方法时，我们首先注意到的是一个无限循环，它将在许多通道之间重复执行选择操作，但每次只能执行其中一个。这些操作中的第一个是接收要发布给订阅者的新消息。`case msg := <- p.in:` 代码处理这个传入操作。

在这种情况下，我们正在迭代所有订阅者并执行它们的`Notify`方法。也许你会想知道为什么我们不在前面加上`go`关键字，以便`Notify`方法作为一个不同的 Goroutine 执行，因此迭代速度更快。嗯，这是因为我们没有对接收消息和关闭消息的操作进行解复用。因此，如果我们在新的 Goroutine 中启动订阅者，并且在`Notify`方法中处理消息时关闭了它，我们将会出现竞争条件，即消息将尝试在`Notify`方法中发送到一个关闭的通道。事实上，当我们开发`Notify`方法时，我们考虑到了这种情况，但是，如果我们每次在新的 Goroutine 中调用`Notify`方法，我们就无法控制启动的 Goroutines 数量。为简单起见，我们只是调用`Notify`方法，但是控制在`Notify`方法执行中等待返回的 Goroutines 数量是一个很好的练习。通过在每个订阅者中缓冲`in`通道，我们也可以实现一个很好的解决方案：

```go
    case sub := <-p.addSubCh: 
    p.subscribers = append(p.subscribers, sub) 

```

下一个操作是当一个值到达通道以添加订阅者时该怎么办。在这种情况下很简单：我们更新它，将新值附加到它上。在执行此案例时，不能执行其他调用：

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

当一个值到达移除通道时，操作会变得更加复杂，因为我们必须在切片中搜索订阅者。我们使用了*O(N)*的方法，从开头开始迭代直到找到它，但搜索算法可以得到很大的改进。一旦我们找到相应的`Subscriber`接口，我们就将其从订阅者切片中移除并停止它。需要提到的一件事是，在测试中，我们直接访问订阅者切片的长度，而不进行多路复用操作。这显然是一种竞争条件，但通常在运行竞争检测器时不会反映出来。

解决方案将是开发一种方法，只是为了多路复用调用以获取切片的长度，但它不会属于公共接口。再次，为了简单起见，我们将保持现状，否则这个例子可能会变得太复杂而难以处理：

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

最后一个需要多路复用的操作是`stop`操作，它必须停止发布者和订阅者中的所有 Goroutines。然后，我们必须遍历存储在订阅者字段中的每个订阅者，执行它们的`Close()`方法，以便关闭它们的 Goroutines。最后，如果我们返回这个 Goroutine，它也会结束。

好的，是时候执行所有测试，看看情况如何：

```go
go test -race .
ok

```

还不错。所有测试都成功通过了，我们的观察者模式已经准备就绪。虽然这个例子仍然可以改进，但它是一个很好的例子，展示了我们如何使用 Go 中的通道来处理观察者模式。作为练习，我们鼓励您尝试使用互斥锁而不是通道来控制访问，这样做会更容易，也会让您了解如何使用互斥锁。

## 关于并发观察者模式的几句话

这个例子演示了如何利用多核 CPU 来构建一个并发消息发布者，通过实现观察者模式。虽然例子很长，但我们试图展示在使用 Go 开发并发应用程序时的常见模式。

# 总结

我们已经看到了一些开发并发结构的方法，可以并行运行。我们试图展示解决同一个问题的几种方式，一种是没有并发原语，另一种是有并发原语。我们已经看到了使用并发结构编写的发布/订阅示例与经典示例相比有多么不同。

我们还看到了如何使用管道构建并发操作，并通过使用工作池来并行化，这是一种最大化并行性的常见 Go 模式。

这两个例子都足够简单，可以理解，同时尽可能深入了解 Go 语言的本质，而不是问题本身。
