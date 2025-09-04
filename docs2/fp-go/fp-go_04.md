# 4

# 使用纯函数编写可测试的代码

当你阅读有关函数式编程的内容时，相当常见的是指的是“纯”函数式编程。正如我们在第一章中提到的，这并不是函数式编程或函数式语言的严格要求。如果你决定学习函数式编程语言，你很可能选择像 Haskell 或 Elm 这样的语言。如果是这样，你就选择了两种纯函数式语言，并且可能会将你对*纯函数式*的理解与*函数式*相结合。另一方面，如果你选择了像 Lisp、Clojure 或 Erlang 这样的语言，你就选择了一种不纯但仍然是函数式的语言。

在本章中，我们将讨论以下主题：

+   纯度究竟是什么？

+   为什么纯度很重要？

+   我们如何创建纯函数？

+   学习如何通过编写纯函数来影响单元测试

# 技术要求

对于本章，可以使用 Go 1.12 之后的任何版本。你可以在[`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter4`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter4)找到完整的示例。

# 什么是纯度？

当谈论纯函数式编程语言时，我们指的是一种每个函数都遵循以下特性的语言：

+   不产生任何副作用

+   提供相同的输入时返回相同的输出（幂等性）

这意味着我们的函数是完全确定性的。

最好的前进方式可能是通过展示一些例子来演示我们正在讨论的内容。因此，在本节中，我们将查看两个函数，一个是纯函数，另一个是不纯函数。然后，我们将更多地讨论这类函数的特性及其对我们所编写的程序的重要性。

## 展示纯函数与不纯函数调用的区别

这的一个简单例子就是一个加法函数。这是一个接收两个整数作为输入并返回和作为输出的函数：

```go
func add(a, b int) int {
	return a + b
}
```

当我们用相同的输入调用这个函数时，我们将得到一致的结果。因此，无论我调用`add(10,5)`函数多少次，代码总是会返回相同的输出：15。在创建纯函数时，这一点非常简单。我们没有在函数外部使用任何状态来确定答案，也没有在函数外部更新任何内容。

接下来，让我们看看一个不纯函数的例子，其输出总是随机的：

```go
func rollDice() int {
	return rand.Intn(6)
}
```

当调用`rollDice`函数时，输出并不一致。如果它总是输出相同的数字，那将是一个非常糟糕的随机化函数。如果我们调用`rollDice`函数五次，我们会得到五个不同的输出：

```go
func main() {
	for i := 0; i < 5; i++ {
		fmt.Printf("dice roll: %v\n", rollDice())
	}
}
```

这将产生以下输出：

```go
dice roll: 5
dice roll: 3
dice roll: 5
dice roll: 5
dice roll: 1
```

## 指称透明性

有一个特性帮助我们思考纯函数，那就是引用透明性的特性。在数学和计算机科学中，如果一个函数可以被其输出所替换，而不改变程序的结果，那么这个函数就被认为是引用透明的。在数学中，这一点很容易理解。如果我们解出任何公式，我们实际上可以用其结果替换方程的一部分，而不会改变结果。例如，考虑以下方程：

```go
X = 1 + (2 * 2)
```

结果是 5。如果我们用其结果替换乘法，我们同样可以得到相同的结果，如下所示：

```go
X = (1 + 4)
```

这个特性就是我们所说的引用透明性。所有数学运算都具有这个特性，我们中的许多人都在代数、微积分或其他数学课程中利用了这个特性来解决问题。

让我们回到软件工程的领域，进一步探讨这个问题。在编程语言中，引用透明性意味着函数调用可以被其结果所替换。如果我们将这个相同的测试应用到我们之前编写的`add`函数上，我们可以看到这是如何成立的。让我们用一小段代码来演示这一点：

```go
func main() {
	fmt.Printf("%v\n", add(10, add(10, 5)))
	fmt.Printf("%v\n", add(10, 15))
}
func add(a, b int) int {
	return a + b
}
```

在这个例子中，我们用其结果替换了其中一个`add`函数。果然，我们程序的输出保持一致，并且功能正确。你可能认为这是显而易见的，但有许多我们依赖的函数并不具备这个特性。让我们引入另一个打破这个特性的函数。我们将保持简单，创建一个告诉我们当前时间的程序：

```go
func main() {
	fmt.Printf("%v\n", time.Now())
}
```

在这个片段中，我们使用了`time.Now`函数。没有哪个值可以替换这个函数调用，同时保证你的程序在功能上等效且正确。如果我们硬编码当前时间，那么在程序编译和运行时，这个时间就会是错误的。

为了进一步说明这一点，让我们看一个比`time.Now`函数更大的例子。在下面的代码片段中，让我们假设我们正在编写一个函数来选择游戏的起始玩家。我们将使用从`Player`到`string`的简单类型别名，而不是创建一个完整的结构体。由于这是一个游戏，我们希望我们的起始玩家在每次程序运行时都是随机选择的：

```go
type Player string
const (
	PlayerOne Player = "Remi"
	PlayerTwo Player = "Yvonne"
)
func selectStartingPlayer() Player {
	randomized := rand.Intn(2)
	switch randomized {
	case 0:
		return PlayerOne
	case 1:
		return PlayerTwo
	}
	panic("No further player available")
}
```

在前面的代码中，我们违反了代码的引用透明性要求，因为没有办法用一个单一值替换这个函数调用，同时保持程序等价的结果。前面的代码也不可测试。思考一下——你将如何为这个函数编写单元测试？这将证明在代码当前状态下是无法做到的，并且需要一些重构。我们将在本章后面展示如何重构此代码并使其可测试，你可以在 GitHub 上找到它：[`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter4/TestableCode`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter4/TestableCode)。

## 幂等性

纯函数的另一个特性是它们是幂等的。这意味着无论函数执行多少次，只要输入参数保持不变，它总是会返回相同的输出。在前面的例子中，`add`函数总是返回相同的两个数的和，前提是输入相同。另一方面，`time.Now`函数不是（这也不是预期的行为）。

你可能对幂等性很熟悉，因为它在实现`REST`服务或处理 HTTP 调用时也会出现。当正确实现时，`GET, HEAD, PUT, 和 DELETE`方法应该是幂等的。一个值得注意的例外是`POST`方法。

## 无状态

纯函数不应依赖于系统的任何状态。这意味着输入和输出都不应改变状态。通常说 Web 请求是无状态的；每个请求可以独立于其他请求运行，并且仍然产生相同的结果。在 Go 的术语中，这也意味着我们的函数不应依赖于诸如全局变量、文件系统上的文件或一般的 I/O 操作等事物。

## 副作用

之前提到的属性结合在一起，创建出没有副作用的功能。副作用是指你的函数所做的任何改变系统状态的操作。在下一章中，我们将更深入地探讨在`struct`级别上状态不可变的意义。在这一章中，我们将考虑状态为程序运行的系统。

# 为什么纯度能提高我们的代码？

到目前为止，我们已经探讨了纯函数代码的一些特性。我们也看到了纯函数和不纯函数的例子。现在，让我们看看编写纯函数代码可以期待哪些好处。

## 提高了我们代码的可测试性

当编写纯函数时，你的函数将更容易测试。这是它们既是幂等的又是无状态的后果：

+   **幂等性**：运行函数任意次数都会得到相同的结果

+   **无状态**：每个函数将独立于系统的状态运行

对于幂等性，很容易看出这一点是正确的。在我们的测试套件中，如果函数对于相同的输入返回不同的输出，那么为该函数编写测试将会很困难。毕竟，如果你无法预测某个函数的输出，你只能猜测应该测试的值。它无状态的好处可能并不立即明显。这归结于我们的测试套件无法在我们的生产系统环境中运行。因此，如果我们以某种方式依赖于系统状态，我们必须保证我们的测试状态在函数被调用的那一刻复制了生产状态。让我们用一个例子来演示这一点。

回想一下本章前面提到的，当我们创建一个函数来为游戏选择一个随机玩家时？让我们将这段代码重构为更易于测试的形式。我们需要做两个改动——首先，我们需要使函数具有确定性。这听起来好像打破了随机性，确实如此，但我们将很快展示如何解决这个问题。我们将做的第二个改动是移除任何副作用。在我们的第一个例子中，如果随机化函数返回大于 1 的整数，我们有一个`panic`函数。我们将用从我们的函数返回一个包含`(Player, error)`元组的做法来替换这个`panic`，这遵循了 Go 语言中常见的错误处理惯例。有了这些改动，我们新的函数看起来是这样的：

```go
func PlayerSelectPure(i int) (Player, error) {
	switch i {
	case 0:
		return PlayerOne, nil
	case 1:
		return PlayerTwo, nil
	}
	return Player(""), fmt.Errorf("no player matching input:  	        %v", i)
}
```

在这些改动到位后，我们的函数现在是确定的。对于每个输入，我们总是生成相同的输出，如下所示：

```go
PlayerSelectPure(0) = PlayerOne, nil
PlayerSelectPure(1) = PlayerTwo, nil
PlayerSelectPure(n > 1) = Player{}, error
```

注意在最后一种情况下，当`n`大于一时，我们并没有简单地返回`nil`和错误。这需要一些解释。其核心思想是我们将尽可能避免在我们的代码中使用指针。在 Go 语言中，如果你不使用指针，你就无法表示`nil`。我们为什么要避免使用它以及它的含义将在下一章中详细解释，*第五章*。

现在我们已经看到了每种情况下的预期输出，并且我们同意这个函数是纯的，我们可以编写一个测试用例来确认输出是否符合我们的预期：

```go
func TestPlayerSelectionPure(t *testing.T) {
	selectPlayerOne, err := PlayerSelectPure(0)
	if selectPlayerOne != PlayerOne || err != nil {
		t.Errorf("expected %v but got %v\n", PlayerOne,  	            selectPlayerOne)
	}
	selectPlayerTwo, err := PlayerSelectPure(1)
	if selectPlayerTwo != PlayerTwo || err != nil {
		t.Errorf("expected %v but got %v\n", PlayerOne,  	            selectPlayerTwo)
	}
	_, err = PlayerSelectPure(2)
	if err == nil {
		t.Error("Expected error but received nil")
	}
}
```

上述代码中发生的一切都很简单明了。对于每个有效输入（0 和 1），我们确认分别返回第一个或第二个玩家。对于大于 1 的输入，我们确认抛出了一个错误。技术上，你可以扩展这个单元测试，以穷举测试所有可能的整数输入，并确认每个输入都会抛出错误。不过，对于这个简单的函数来说，这可能有点过于详尽。

在这种情况下，唯一需要解决的事情就是：我们的代码不再选择一个随机玩家，而是期望一个整数输入并返回一个确定性的值。你可能注意到，我们只是将问题转移了，因为随机选择函数仍然需要存在于某个地方。这是正确的。如果我们看看在实际游戏中如何使用这段代码，我们可能会发现这样的代码：

```go
func main() {
	random := rand.Intn(2)
	player.PlayerSelectPure(random)
	// start the game
}
```

在这里，我们可以看到一种重复出现的模式，因为我们旨在提高代码的纯净度。策略将是限制副作用和非确定性可能发生的地方。当你改变思考代码结构的方式，更倾向于函数纯净并隔离破坏它的位置时，你可能会得到 90%的纯代码和 10%的不纯代码。当然，你并不是 100%的纯函数式，但我们在 Go 中编程，我们可以原谅自己 10%的不纯代码。正如我们详细探讨的那样，纯函数式编程是函数式编程的一个子集。此外，没有纯函数式编程警察会因为你写了一个不纯的函数而追捕你。

这是否意味着完全纯净是不可能的？好吧，并不完全是这样。毕竟，有一些纯函数式编程语言，如 Haskell，可以在现实世界的生产环境中使用。它们处理这些不纯函数的方式是通过一种称为**单子**的封装形式。虽然可以在 Go 中创建单子，但这可能会引起不必要的摩擦，这就是我为什么主张拥抱函数式而非纯函数式代码的理念。为了娱乐和扩展我们对纯函数式代码的探索，我们将在下一章中探讨单子。

## 提高我们代码的信心

虽然这与提高可测试性密切相关，但你对代码的信心提升远不止于此。当处理不纯函数和状态时，你的程序更难以理解。如果你在一个足够复杂的系统中工作，该系统包含不纯函数和状态突变，如通过全局变量，推理起来会更困难。想象一下，你在一个这样的复杂系统中工作，一个用户报告了一个错误。如果系统是可变的，你需要完全理解错误出现时整个系统的样子，才能开始调试。这可能导致许多痛苦且浪费时间的调试。有一个流行的概念，称为**海森堡虫**，这是其后果之一。在这种情况下，如果导致错误的函数依赖于系统的状态，你可能需要重复用户所做的确切步骤才能重现错误。

另一个好处是，我们的代码更容易调试。在调试程序时，任何足够先进的调试器也会显示调试期间你的系统状态。它会告诉你程序各个部分在内存中持有的值。这是一个伟大的工具，可以帮助你找到错误并消除它们。但如果你程序根本不依赖于这种状态呢？这将消除对像高级调试器这样的关键的需求。

你可以查看一个单独的函数，并对其所做的事情进行推理，而无需同时记住执行时刻整个系统的样子。人类在保持“工作记忆”方面很糟糕；我们可以在任何给定时刻处理大约 7 ± 2 件事情。如果我们优化并尝试让我们的程序对大多数人类来说是可理解的，我们就必须将状态变量限制为只有 5 个。这是忽略我们函数可能有一些变量的事实。因此，我们很快就会超过人类记忆的上限。

## 提高对函数名称和签名的信心

提高你代码的可读性和可理解性的另一个巨大好处是，你突然会对你的函数有额外的信心。想象一下，你正在阅读一个代码库，你遇到了以下这段代码：

```go
func main() {
	i := 1
	for i < 10 {
		i = add1(i)
		fmt.Printf("%v,", i)
	}
}
```

输出会是什么样子？你可能自然会假设`add1`是一个纯函数，输出如下：

```go
1,2,3,4,5,6,7,8,9,10,
```

但是，你会错的。实际输出如下：

```go
1, 2, 3, 4, 5, 6, panic: can not increment any more
goroutine 1 [running]:
main.add1(...)
	/tmp/sandbox1318301126/prog.go:17
main.main()
	/tmp/sandbox1318301126/prog.go:10 +0xa5
Program exited.
```

要理解为什么，让我们看看`add1`函数：

```go
func add1(input int) int {
	if input != 0 && input > rand.Intn(input) {
		panic("can not increment any more")
	}
	return input + 1
}
```

在前面的函数中，我们可以看到`add1`函数是不纯的。它不是确定的，因为每次运行的输出取决于生成的随机数。此外，它还产生了副作用。每次一个函数中有一个`panic`语句时，该语句会产生一个超出你函数正常输出的副作用。这是一个有点人为的例子，但它表明，当在一个函数可以包含副作用且不是幂等的环境中工作时，你对函数签名本身的信任度会降低。

## 更安全的并发

Go 的一个卖点，以及它区别于许多主流语言的特点之一，是它处理并发的容易程度。使用 Go，启动多个线程并使它们并行工作非常简单。这是通过**通道**和**goroutines**概念实现的。关于 Go 中并发的工作方式有很多可以说的，足以写一本书。在这里，我们将简要关注并发的正确性方面。在 Go 中启动 goroutines 和并行处理是否比 Java 等语言更容易？不正确的是，编写正确的并发代码更容易。

让我们看看一些并发代码。在这个例子中，我们将创建一个整数切片，并在`addToSlice`函数中向其中追加。在我们的`main`函数中，我们将一个整数推送到切片中：

```go
var (
	integers = []int{}
)
func addToSlice(i int, wg *sync.WaitGroup) {
	integers = append(integers, i)
	wg.Done()
}
func main() {
	wg := sync.WaitGroup{}
	numbersToAdd := 10
	wg.Add(numbersToAdd)
	for i := 0; i < numbersToAdd; i++ {
		go addToSlice(i, &wg)
	}
	wg.Wait()
	fmt.Println(integers)
}
```

想想这个程序，并尝试猜测输出结果会是什么。正确答案是这个程序的输出是非确定性的。我们在多个线程中运行，向我们的切片中追加内容，最后调用`wg.Done()`。当与这些等待组一起工作时，我们传递几个线程去等待。这是在`wg.Add(numbersToAdd)`中完成的。每次调用`wg.Done()`时，等待的线程数减一。在这个例子中，我们正在处理一个共享的整数切片，因此无法准确预测在最后一个线程执行`add`操作时切片看起来是什么样子。这意味着我们的输出可能是所有数字随机排序的，例如`[9 0 1 2 3 4 5 6 7 8]`，但同样可能的结果是`[4 9 0 1 2]`。在并发函数中有可变的数据源是灾难性的，会导致一些难以追踪的 bug。

所以，正如你可以从这个小片段中看到的，启动多个线程非常简单，但避免代码中的 bug 并不那么简单。纯函数可以帮助解决这个问题。记住，当一个函数是纯的，相同的输入总是生成相同的输出，而不会引起任何副作用。

在这个例子中，我们的副作用是修改了切片，这在 Go 中不是线程安全的。程序不会崩溃，但结果将是随机的。如果我们将纯函数编程推向极端，我们将消除所有这样的不纯函数，并且在这个过程中，我们可以无限并行地运行所有函数而不会引起任何麻烦。

注意

在实践中，有使用互斥锁来避免这种情况发生的方法。一些库会处理并行性，从而抽象掉一些复杂性。

# 不应该编写纯函数的情况

到目前为止，我们已经看到了纯函数是什么以及纯函数可以提供什么样的优势。但我们应该至少花一点时间思考一下我们可能想要牺牲函数纯度的场合。现在，如果你向“纯粹主义者”提出这个问题，这个问题的答案可能大致是这样的：“永远不，决不，绝不。”这是可以的，有些语言使得编写不牺牲函数纯度的函数代码变得相当简单。但是，让我们看看一些牺牲一些函数纯度是有意义的例子。现在，在我们深入这些例子之前，让我首先承认，所有这些所谓的都是可以规避的。是的，像 Haskell 这样的语言处理这些问题大多数情况下都很优雅。

但我们不是在用 Haskell 编程；我们是在用 Go 编程。虽然 Go 允许我们编写纯函数代码，如果我们愿意的话，但有些事情通过暂时原谅自己编写不纯代码的“罪行”更容易实现。

## 输入/输出操作

考虑从代码中完全消除副作用的影响。如果我们说我们在编写纯函数代码并消除了所有副作用，那么我们也消除了为我们的用户提供价值的一部分。每次我们从用户那里获取输入或以某种方式向用户显示输入时，在技术上都是副作用。每次我们在本地存储中存储数据或上传到服务器时，我们都在产生副作用。许多应用程序会接收某种类型的输入，许多也会生成某种类型的输出。

## 非确定性可能是期望的

我们可能不希望创建纯函数的另一个原因是，非确定性特性适合于我们构建的领域。如果我们正在构建《大富翁》游戏，那么期望`rollDice`函数返回一个非确定性结果是合乎愿望的。《大富翁》的例子并非偶然。随机性是我们周围许多游戏固有的特性，因此，纯确定性不是每个函数期望的结果。

## 当我们真的不得不`panic`时！

当你的程序处于无法正常继续运行的状态时，处理这种情况的典型方式是使用`panic`。虽然`panic`应该谨慎使用，但它们是你产生副作用的一个实例。在本章的早期，我们看到了一个函数在执行过程中可能会不可预测地引发`panic`的例子。那个例子是人为的，对于`panic`函数来说是一个相当糟糕的使用案例。但这并不意味着永远没有使用`panic`的有效理由。例如，如果你试图预留超出系统可用内存的内存，那可能就是引发`panic`的原因。一般来说，`panic`应该用来表示正常操作无法继续进行，且没有优雅地继续运行应用程序的方法。

有两点值得指出。第一点是使用`panic`关键字应该是例外而不是常规操作。第二点是 Go 语言中有一个常见的错误处理模式，即返回一个包含潜在错误值的元组。从函数中返回错误与使用`panic`是不同的操作，它们服务于不同的用例。

# 我们如何创建纯函数？

到目前为止，在本章中，我们查看了一些纯函数的性质。我们还简要提到了通过将所有函数编写为纯函数所能获得的一些优势。现在，让我们看看我们可以做些什么来使编写纯函数更容易。

## 避免全局状态

我们可以促进编写纯净函数式代码的一种方法是在我们的程序中避免全局状态。在 Go 语言中，这相当于尽可能避免在包级别使用`const`和`var`块。当你看到这些块时，有很大可能性是某些函数依赖于程序状态，从而产生副作用或具有非确定性的程序执行。虽然我们不可能完全避免这样的状态变量，但我们应尽可能限制它们的使用。防止函数依赖这种状态的方法是通过正常函数参数将状态传递给函数。这相当直接。以下是一个小例子，一次使用`var`块中的状态，一次不使用：

```go
var (
	name = "Remi"
)
func sayHello() string {
	return fmt.Sprintf("hello %s", name)
}
func main() {
	sayHello()
}
```

我们可以通过将`name`参数作为输入传递给我们的函数，而不使用`var`块来获得与前面块相同的功能：

```go
func sayHello(name string) string {
	return fmt.Sprintf("hello %s", name)
}
func main() {
	sayHello("Remi")
}
```

这就是重点。接下来，让我们看看处理包含杂质代码的一般方法。

## 区分纯净和杂质功能

如前所述，要完全纯净是很难的。我们不应试图消除 I/O 操作、API 调用等，因为消除这些操作，我们可能会丢弃使我们的程序有价值的大部分内容。主要的练习将是尝试创建尽可能多的小型、纯净函数，并将这些组合成更大的程序。仍然会有副作用，但我们将限制它们的发生。

### 错误冒泡

一种相对常见的副作用是由错误产生的。我们的程序最终处于无法优雅继续的状态，而且没有真正的方法可以绕过这个问题。在这里隔离纯净和杂质方面的一种方法是通过使用 Go 的错误处理习惯用法，并将错误“冒泡”到可以处理的公共层。我们在选择随机玩家的例子中已经看到了这一点。自 Go 1.13 以来，还有额外的内置工具可用于错误冒泡。

### 每个函数只做一件事

这是一般性的好建议。一般来说，一个函数应该只做一件事，这显著降低了我们的函数产生副作用的可能性。你同样可以在传统的面向对象语言中找到这个原则。业界或多或少都同意这是正确的方法，但打破这个良好意图却出奇地容易。看看以下简单加法函数的代码：

```go
 func add(a, b int) int {
	sum := a + b
	fmt.Println(sum)
	return sum
}
```

这不是一个纯净函数。这个片段的副作用是我们将`sum`值打印到标准输出。确实，这并不太有害，但如果我们的用户依赖于这个功能，我们如何确保这个函数正常工作？换句话说，你将如何测试这个函数是否将正确的输出打印到屏幕上？

这种变体可能是将写入文件系统或数据库调用作为函数的一部分，而这种情况本不应该发生。让我们看看一个用于注册新用户到服务的函数。我们期望输入是一个用户名和一个密码，`User`结构体上定义了一些逻辑来确保密码符合密码规则：

```go
func createUser(username, password string) {
	u := User{username, password}
	if u.validPassword() {
		userDb.save(u)
	} else {
		panic("invalid password")
	}
}
```

这个函数的问题在于它试图做两件事。首先，它创建一个新的用户结构体并确认密码是否符合要求。接下来，它将`User`结构体存储在数据库中，假设密码有效；否则，它将引发恐慌。我们可以将这个操作拆分成多个函数，一个用于验证密码，一个用于存储用户，第三个函数用于协调这些操作：

```go
func signup(username, password string) {
	user, err := createUser(username, password)
	if err != nil {
		saveUser(user)
	} else {
		Panic("Could not create account")
	}
}
func createUser(username, password string) (User, error) {
	u := User{username, password}
	if u.validPassword() {
		return u, nil
	}
	return User{}, Errors.new("invalid password")
}
func saveUser(u User) {
	userDb.save(u)
}
```

在前面的例子中，我们已经分离了关注点，但我们仍然留下了两个不纯的函数。然而，问题现在被更严格地限制在单个函数内部。这段代码还不够完美，还有改进的空间，我们将在下一章中看到。不过，在去那里之前，让我们看看一个更广泛的例子。

# 示例 1 – 热狗店

在我们的第一个例子中，我们将查看一些以不纯方式编写的代码，这些代码几乎违反了编写纯函数的所有良好原则。我们将随着代码的进行进行重构，以创建更可测试的代码，同时提高代码的可读性和可理解性。

## 不好的热狗店

首先，让我们看看如何不创建这个热狗店系统。我们将从一个常量开始定义，这是一个全局变量，它决定了我们热狗的价格：

```go
const (
	HOTDOG_PRICE = 4
)
```

接下来，我们将创建一些结构体。我们需要一个结构体来表示热狗，还需要一个结构体来存储我们的信用卡信息。为了简化，目前热狗不包含任何状态变量，而信用卡只存储卡上可用的信用额度。在这个例子中，信用额度是一个整数值。它并不准确地代表现实生活中的货币价值，但对于这个例子来说已经足够了：

```go
type CreditCard struct {
	credit int
}
type Hotdog struct{}
```

定义好这些之后，我们可以着手实现我们关心的第一个功能。我们需要一种方式来为我们的信用卡充值一定金额：

```go
func (c *CreditCard) charge(amount int) {
	if amount <= c.credit {
		c.credit -= amount
	} else {
		panic("no more credit")
	}
}
```

在之前的`charge`方法中，我们通过减少卡上的可用信用额度来为信用卡收取一定金额。如果没有足够的信用额度来执行扣款，我们使用`panic`来停止程序。目前，这个函数的主要问题是使用了副作用。有两个副作用。首先，如果遇到某个分支，我们会使用`panic`。下一个副作用是我们改变了`CreditCard`的状态。结构体的不可变性是我们将在下一章详细讨论的主题，所以现在让我们忽略这个问题，继续编写我们的热狗店代码。对于用户来说最重要的功能是订购热狗。所以，让我们看看如何实现这个功能的实现：

```go
func orderHotdog(c *CreditCard) Hotdog {
	c.charge(HOTDOG_PRICE)
	return Hotdog{}
}
```

之前的代码，再次，是不纯的代码。用户的信用卡正被一个在函数外部定义的价格所扣除，使用了全局状态。这个函数做了不止一件事——它既为用户创建了一个要返回的热狗，同时也扣除了他们的信用卡。

想想你是如何测试这个功能的。测试这个功能是可能的——但并不方便。你需要测试或模拟信用卡，以确保这个函数确实返回了一个热狗。此外，你还得捕捉一个潜在的恐慌，这并不是在`orderHotdog`函数中发生的，而是在更深层次的调用中。另外，因为`charge`也是不纯的，`orderHotdog`的读者如果没有查看那个特定的函数，就没有意识到`charge`*可能*会引发恐慌。正如我们之前学到的，纯函数式代码在阅读代码时给我们带来了更多的信心。我们相信一个函数会做它所说的——不多，也不少。带着这个想法，让我们看看我们如何可以重构这段代码。

## 更好的热狗店

在这个热狗店的版本中，我们将尝试解决我们在上一个例子中发现的一些问题。完整的代码可以在[`github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter4/Examples/HotdogShop/PureHotdogShop`](https://github.com/PacktPublishing/Functional-Programming-in-Go./tree/main/Chapter4/Examples/HotdogShop/PureHotdogShop)找到。

让我们从定义我们的类型开始：

```go
type CreditCard struct {
	credit int
}
type Hotdog struct {
	price int
}
type CreditError error
type PaymentFunc func(CreditCard, int) (CreditCard, 
  CreditError)
```

在这里，我们定义了所有我们需要表示这个小型应用程序数据的类型。我们的`CreditCard`结构体包含一个整数金额的信用额度，而热狗的价格也是一个整数。我们定义了一个名为`CreditError`的错误，以及一个支付函数的类型别名。让我们也为`CreditCard`和`Hotdog`设置一些类似构造函数的函数：

```go
func NewCreditCard(initialCredit int) CreditCard {
	return CreditCard{credit: initialCredit}
}
func NewHotdog() Hotdog {
	return Hotdog{price: 4}
}
```

这些相当直接。我们将添加一个全局变量来表示一个错误，即用户没有足够的信用额度来对信用卡执行操作：

```go
var (
	NOT_ENOUGH_CREDIT CreditError = CreditError(errors.
      New("not enough credit"))
)
```

如您可能记得的，之前我曾反对使用这类包级别的声明。这一点依然成立，我建议尽可能避免使用它们。然而，对于错误声明来说，这几乎是编写 Go 代码的公认、惯用的方式。

我们可以在这里避免它，并在适用的地方直接实例化错误，但这会稍微损害我们稍后要编写的测试代码。一般来说，请记住，我提倡在 Go 中使用*函数式编程*，而不是*纯*函数式编程。

无论哪种方式，让我们编写我们的第一个非平凡函数。我们将以纯净的方式重写最初的`charge`函数。这里的目的是通过返回一个包含潜在错误的元组来消除我们之前没有通过`panic`而是通过返回副作用的方式。

```go
func Charge(c CreditCard, amount int) (CreditCard, CreditError) {
	if amount <= c.credit {
		c.credit -= amount
		return c, nil
	}
	return c, NOT_ENOUGH_CREDIT
}
```

如您在前面的代码片段中可以看到，我们不仅返回了一个错误值，还返回了一个`CreditCard`类型的值。这并不是由调用者传递给函数的同一个`CreditCard`。因为我们没有使用`CreditCard`的指针，当函数调用`Charge`时，`CreditCard`将在`Charge`函数内部使用。由于我们正在操作一个副本，`c.credit -= amount`这一语句只会影响副本，而不会影响原始的`CreditCard`。这是新晋 Go 程序员常见的陷阱。在下一章中，我们将更深入地探讨不可变性，并讨论这种方法和基于指针的函数调用之间的权衡。但可以肯定的是，当前这个函数是*足够纯净的*。

这个`Charge`函数也易于测试。让我们编写一个单元测试来确保行为符合我们的预期。首先，我们将定义我们的测试用例。以下结构是*表格驱动测试*的设置：

```go
var (
	testChargeStruct = []struct {
		inputCard  CreditCard
		amount     int
		outputCard CreditCard
		err        CreditError
	}{
		{
			CreditCard{1000},
			500,
			CreditCard{500},
			nil,
		},
		{
			CreditCard{20},
			20,
			CreditCard{0},
			nil,
		},
		{
			CreditCard{150},
			1000,
			CreditCard{150},   // no money is withdrawn
			NOT_ENOUGH_CREDIT, 
               // payment fails with this error
		},
	}
)
```

在前面的代码片段中，我们正在测试我们的代码可以采取的几个路径。我们可以尝试在信用额度超过成本时扣款，当有恰好足够的金额时扣款，或者当信用额度不足时扣款。有了这种表格结构，添加更多测试用例变得非常简单。现在，让我们编写实际的单元测试，它将为之前定义的每个测试用例运行测试：

```go
func TestCharge(t *testing.T) {
	for _, test := range testChargeStruct {
		t.Run("", func(t *testing.T) {
			output, err := Charge(test.inputCard, test. 	                     amount)
			if output != test.outputCard || !errors. 	 	                  Is(err, test.err) {
				t.Errorf("expected %v but got %v\n,  	                         error expected %v but got %v",
				test.outputCard, output, test.err, err)
			}
		})
	}
}
```

哇！这是对`charge`函数的完整单元测试。这在非纯净示例中几乎是不可能的。现在，让我们也重构一下我们之前提到的`OrderHotdog`函数。就像任何事物一样，解决这个问题有多个方法。我们在这里实施的方法是使用高阶函数将计算延迟到更晚的阶段。这将把实际扣款的副作用向上移动到调用链中：

```go
func OrderHotdog(c CreditCard, pay PaymentFunc) (Hotdog, func() (CreditCard, error)) {
	hotdog := NewHotdog()
	chargeFunc := func() (CreditCard, error) {
		return pay(c, hotdog.price)
	}
	return hotdog, chargeFunc
}
```

让我们分析一下这里发生的事情。首先，是我们的函数签名。`OrderHotdog`函数仍然接受`CreditCard`作为输入，但还接受一个`PaymentFunc`。回想一下，我们定义`PaymentFunc`为一个接受`CreditCard`和一个`int`作为参数，并返回`CreditCard`和`CreditError`的函数。`OrderHotdog`函数返回热狗本身，以及一个将返回`CreditCard`和`error`的函数。这可能会一开始有些令人困惑，但在函数体中会变得清晰。

第一步是创建一个新的热狗。之后，我们必须在行内创建一个新的函数。回想一下，这是可能的，因为 Go 支持将函数作为一等公民。在这个函数内部，我们使用提供的信用卡，为热狗的价格调用`pay`。这是一个`OrderHotdog`函数，它返回热狗和刚刚创建的函数。需要注意的是，当调用`OrderHotdog`函数时，`chargeFunc`并不会被执行。这个函数中没有发生副作用；副作用被延迟到了后续的阶段。再次强调，我们将尽可能地将副作用隔离。在调用链的更高层次进行副作用处理是一个更好的选择，因为我们的代码通常是从抽象层次较高的地方开始阅读的。这可以避免在实现细节中隐藏的意外。

通过这种方式，我们已经重新创建了原始热狗店的功能。在我们查看测试`OrderHotdog`之前，我们将首先看看如何使用这个函数的例子。在下面的`main`函数中，我们将订购一个热狗，随后调用`pay`函数来对信用卡进行扣费：

```go
func main() {
	myCard := NewCreditCard(1000)
	hotdog, creditFunc := OrderHotdog(myCard, Charge)
	fmt.Printf("%+v\n", hotdog)
	newCard, err := creditFunc()
	if err != nil {
		panic("User has no credit")
	}
	myCard = newCard
	fmt.Printf("%+v\n", myCard)
}
```

好了，这是一个可用的订购热狗的例子。让我们看看我们是如何调用`OrderHotdog`的。我们传递了信用卡以及我们之前编写的`Charge`函数。你可以在 GitHub 的示例仓库中运行这个例子，并对其进行实验。让我们也通过编写单元测试函数来确认这段代码是可测试的。

对于这个例子，我们不需要一个表格驱动的测试。`OrderHotdog`函数需要被测试以确保它执行以下操作：

+   创建一个新的热狗

+   创建一个调用支付函数的函数

+   返回热狗和函数

我们的测试函数将确认是否创建了一个新的热狗，并且支付函数被调用。由于这是一个单元测试，我们并不关心支付函数本身。我们将模拟一个支付函数以确保它被从返回的函数中调用。实际的`charge`函数如之前所见，将被单独测试：

```go
func TestOrderHotdog(t *testing.T) {
	testCC := CreditCard{1000}
	calledInnerFunction := false
	mockPayment := func(c CreditCard, input int) (CreditCard, 
      CreditError) {
		calledInnerFunction = true
		testCC.credit -= input
		return testCC, nil
	}
	hotdog, resultF := OrderHotdog(testCC, mockPayment)
	if hotdog != NewHotdog() {
		t.Errorf("expected %v but got %v\n", NewHotdog(), 
            hotdog)
	}
	_, err := resultF()
	if err != nil {
		t.Errorf("encountered %v but expected no error\n", 
            err)
	}
	if calledInnerFunction == false {
		t.Errorf("Inner function did not get called\n")
	}
}
```

在前面的代码中，我们严格测试我们的函数是否为热狗和闭包创建了正确的值。在这种情况下，正确的闭包函数意味着返回的函数调用了传递给它的支付函数。注意我们如何可以模拟原始行为并创建一个`bool`来确保函数被调用。再次证明，这是在 Go 中拥有一等函数的力量。

# 摘要

在本章中，我们探讨了纯函数式编程。首先，我们探讨了编程语言为何被称为纯函数式，而不是不纯函数式，以及函数式编程的含义。接下来，我们更详细地探讨了纯代码如何通过消除副作用来提高可测试性。我们还了解到，纯代码让读者对所阅读的代码更有信心，因为函数更可预测，不会改变系统的状态。我们还讨论了何时不应使用纯函数，例如处理应生成随机行为的游戏函数或处理 I/O 的函数。

虽然我们只是简要地提到了它，但我们已经看到了通过不改变结构体的值，不可变性如何在编写纯函数中扮演核心角色。在下一章中，我们将深入探讨不可变性，它如何（或不如何）影响性能，以及我们如何利用它与纯函数结合来编写更易于维护的代码。
