# 前言

直到最近，信息一直是*Go 和函数式编程——不要这样做*。

函数式编程（FP）非常适合多核、并行处理。Go 是一个并发球员（具有 Goroutines、通道等），并且已经在每个可用的 CPU 核心上运行。FP 减少了复杂性；简单性是 Go 最大的优势之一。

那么，FP 能为 Go 带来什么，实际上会改进我们的软件应用程序？它提供了什么：

+   **构成**：FP 向我们展示了如何分解我们的应用程序，并通过重用小的构建模块来重建它们。

+   **单子**：使用单子，我们能够将我们的工作流安全地排序为数据转换的管道。

+   **错误处理**：我们可以利用单子错误处理，同时保持与成熟的 Go 代码的兼容性。

+   **性能**：引用透明性是我们可以评估我们的函数一次，然后随后引用其预先计算的值的地方。

+   **表达性代码**：FP 允许我们在代码中简洁地表达业务意图。我们声明我们的函数做什么，而不必在每个函数调用后进行错误检查的混乱，也不必遵循状态变化（纯 FP 意味着不可变变量）。

+   **更简单的代码**：没有共享数据意味着不必处理信号量、锁、竞争条件或死锁。

大多数人都很难掌握 FP。

我也是如此。当我懂了，我写了这本书。和我一起踏上这段旅程。我们将看到数百幅插图，阅读易于理解的解释，并在途中实现 Go 代码中的 FP。

我喜欢指导足球。我用来确定我是否成功作为教练的试金石是这个简单问题的答案：*他们是否都注册了下个赛季并请求我成为他们的教练？* 就像计划练习一样，我计划了每一章，从简单的概念开始，然后逐渐添加。阅读这本书，然后你也能说，*我懂了*。

如果你想提高你的 FP 技能，这本书适合你。

## 本书涵盖了什么

第一章，*Go 中的纯函数式编程*，介绍了声明式编程风格，并演示了使用斐波那契序列的递归、记忆化和 Go 的并发构造。我们将学习如何对递归代码进行基准/性能测试，我们将得到一些坏消息。

第二章，*操作集合*，向我们展示了如何使用中间（Map、Filter 和 Sort）和终端（Reduce、GroupBy 和 Join）函数执行数据转换。我们使用类似 Mocha 的 BDD Go 框架来测试谓词函数。Itertools 帮助我们掌握 FP 集合操作函数的广度，我们还看了一个分布式 MapReduce 解决方案：Gleam = Go + LuaJIT + Unix Pipes。

第三章，*使用高阶函数*，涵盖了 27 个 FP 特征的列表：匿名函数、闭包、柯里化、Either 数据类型、一级函数、函数、函数组合、Hindley-Milner 类型系统、幂等性、不可变状态、不可变变量、Lambda 表达式、列表单子、Maybe 数据类型、Maybe 单子、单子错误处理、无副作用、运算符重载、选项类型、参数多态性、部分函数应用、递归、引用透明性、和类型的总和或联合类型、尾调用优化、类型类和单元类型。它还涵盖了泛型的示例，并说明了它对 FP 程序员的价值。我们实现了 Map、Filter 和 Reduce 函数，以及使用 Goroutines 和 Go 通道进行惰性评估。

第四章，“Go 中的 SOLID 设计”，讨论了 Gophers 为什么憎恨 Java，良好软件设计原则的应用，如何应用单一职责原则、函数组合、开闭原则、FP 合同和鸭子类型。它还涵盖了如何使用接口建模行为，使用接口隔离原则和嵌入接口来组合软件。我们将学习使用紫色 Monoid 链的结合律，并揭示 Monads 链的延续。

第五章，“使用装饰添加功能”，演示了使用 Go 的互补 Reader 和 Writer 接口进行接口组合。接下来，我们将学习过程式设计与函数式控制反转的比较。我们将实现以下装饰器：授权、日志记录和负载平衡。此外，我们将向我们的应用程序添加 easy-metrics，以查看我们的装饰器模式的实际效果。

第六章，“在架构层面应用函数式编程”，使用分层架构构建应用程序框架，解决循环依赖错误。我们将学习如何应用好莱坞原则，以及观察者模式和依赖注入之间的区别。我们将使用控制反转（IoC）来控制逻辑流，并构建一个分层应用程序。此外，我们将构建一个有效的表驱动框架来测试我们应用程序的 API。

第七章，“函数参数”，让我们明白了为什么我们从 Java 和面向对象编程中学到的很多东西并不适用于 Go，教会我们使用函数选项更好地重构长参数列表，并帮助我们理解柯里化和部分应用之间的区别。我们将学习如何应用部分应用来创建另一个具有较小 arity 的函数。我们将使用上下文来优雅地关闭服务器，并了解如何使用上下文取消和回滚长时间运行的数据库事务。

第八章，“使用流水线提高性能”，涵盖了数据流类型（读取、拆分、转换、合并和写入），并教会我们何时以及如何构建数据转换流水线。我们使用缓冲区来增加吞吐量，使用 goroutines 和通道来更快地处理数据，使用接口来改善 API 的可读性，并实现一些有用的过滤器。我们还实现并比较了用于处理信用卡交易的命令式和函数式流水线设计。

第九章，“函子、幺半群和泛型”，让我们对 Go 中缺乏对泛型的支持有了更深入的了解。我们将看到如何使用代码生成工具来解决重复样板代码的问题。我们将深入研究函数组合，实现一些函子，并学习如何在不同世界之间进行映射。我们还将学习如何编写一个 Reduce 函数来实现发票处理幺半群。

第十章，“Monad、类型类和泛型”，向我们展示了 Monad 的工作原理，并教会我们如何使用 Bind 操作组合函数。它向我们展示了 Monad 如何处理错误并处理输入/输出（I/O）。本章通过 Go 中的 monadic 工作流程实现。我们将介绍 Lambda 演算是什么，以及它与 Monad 有什么关系，看看 Lambda 演算如何实现递归，并学习 Y-组合器在 Go 中的工作原理。接下来，我们将使用 Y-组合器来控制工作流程，并学习如何在管道的末尾处理所有错误。我们将学习类型类的工作原理，并在 Go 中实现一些类型类。最后，我们将回顾 Go 中泛型的优缺点。

第十一章，*适用的范畴论*，让我们对范畴论有了一个实际的理解。我们将学会欣赏范畴论、逻辑和类型理论之间的深刻联系。我们将通过 FP 历史之旅增进我们的理解。本章使用一个维恩图来帮助解释各种编程语言的范畴。我们将理解在 lambda 表达式的上下文中绑定、柯里化和应用的含义。本章向我们展示了 Lambda 演算就像巧克力牛奶。本章涵盖了 FP 的类型系统含义，向我们展示了不同类别的同态和何时使用它们，并使用数学和足球的飞行来增进我们对态射的理解。我们将用线性和二次函数来进行函数组合，并学习接口驱动开发。我们将探索知识驱动系统的价值，并学会如何应用我们对范畴论的理解来构建更好的应用。

附录，*杂项信息和操作指南*，向我们展示了作者建议我们如何构建和运行本书中的 Go 项目。它向我们展示了如何提出对 Go 的更改，介绍了词法工作流解决方案：一种处理错误的 Go 兼容方式，提供了一个提供反馈的地方和一个 FP 资源页面，讨论了 Minggatu-Catalan 数，并提供了世界和平的解决方案。

## 你需要为这本书做好什么准备

如果你想运行每章讨论的 Go 项目，你需要安装 Go。接下来，你需要启动你的 Go 开发环境并开始编写代码。

阅读*附录*中*如何构建和运行 Go 项目*部分的*TL;DR*子部分。转到第一章，*Go 中的纯函数式编程*，开始阅读*获取源代码*部分。继续阅读如何设置和运行你的第一个项目。

其他 Go 资源包括：

+   Go 之旅 ([`tour.golang.org/welcome/1`](https://tour.golang.org/welcome/1))

+   Go by Example ([`gobyexample.com/`](https://gobyexample.com/))

+   学习 Go 书籍 ([`www.miek.nl/go/`](https://www.miek.nl/go/))

+   Go 语言规范 ([`golang.org/ref/spec`](https://golang.org/ref/spec))

当我想到其他要添加的东西时，我会把信息放在这里：[`lexsheehan.blogspot.com/2017/11/what-you-need-for-this-book.html`](https://lexsheehan.blogspot.com/2017/11/what-you-need-for-this-book.html)。

## 这本书适合谁

这本书中的很多信息只需要高中学历。

对于本书中的编程部分，你应该至少有一年的编程经验。精通 Go 或 Haskell 是理想的，但有其他语言（如 C/C++、Python、Javascript、Java、Scala 或 Ruby）的经验也足够了。你应该对使用命令行有一定的了解。

这本书应该吸引两个群体：

1.  非程序员（阅读第十一章，*适用的范畴论*）如果你是其中之一：

+   K-12 数学教师，想知道你所教的内容为什么重要

+   数学教师，想知道你所教的内容与数学的其他分支有何关联

+   法学院的学生，想了解在为客户辩护时你将要做什么

+   足球爱好者，喜欢数学

+   对范畴论感兴趣的人

+   Lambda 演算的爱好者，想看到用图表、图片和 Go 代码来说明它

+   软件项目经理，想看到需求收集、实施和测试之间有更好的对应关系

+   高管，想了解是什么激励和激发了你的 IT 员工

1.  程序员：如果你是其中之一：

+   软件爱好者，想学习函数式编程

+   软件测试人员，想看到需求收集、实施和测试之间有更好的对应关系

+   软件架构师，想要了解如何使用 FP

+   Go 开发人员，喜欢足球

+   Go 开发人员，并希望使用更具表现力的代码实现您的业务用例编程任务

+   Go 开发人员，并希望了解泛型

+   Java 开发人员，并希望了解为什么我们说*少即是多*

+   *您的语言*开发人员，了解 FP 并希望将您的技能转移到 Go

+   Go 开发人员寻找更好的方法来构建数据转换管道

+   Go 开发人员，并希望看到编写更少代码的可行方法，即更少的*err != nil*块

+   有经验的 Go 开发人员，并希望学习 FP 或为工具箱添加一些工具

+   参与软件开发并希望了解以下任何术语的人。

如果您是一名 Go 开发人员，正在寻找以下任何工作代码，并且需要逐行解释，那么这本书适合您：

+   基准测试

+   并发（Goroutines/Channels）

+   柯里化

+   数据转换管道

+   装饰者模式

+   依赖注入

+   鸭子类型

+   嵌入接口

+   错误处理程序

+   函数组合

+   函数参数

+   函子

+   通过代码生成实现泛型

+   好莱坞原则

+   接口驱动开发

+   I18N（语言翻译）

+   IoC

+   Go 中的 Lambda 表达式

+   分层应用框架

+   日志处理程序

+   单子

+   单子

+   观察者模式

+   部分应用

+   处理信用卡支付的管道

+   递归

+   减少函数以求和发票总额

+   解决循环依赖错误

+   基于表的 http API 测试框架

+   类型类

+   将文件上传/下载到/从 Google Cloud Buckets

+   Y-组合子

如果我决定更改格式或更新此信息，我会在这里放置它：[`lexsheehan .blogspot.com/2017/11/who-this-book-is-for.html`](http://lexsheehan%C2%A0.blogspot.com/2017/11/who-this-book-is-for.html)。

## 约定

在本书中，您将找到一些文本样式，用以区分不同类型的信息。以下是一些样式的示例及其含义的解释。文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下：“我们更新代码，运行`glide-update`和`go-run`命令，并重复直到完成。”代码块设置如下：

```go
func newSlice(s []string) *Collection {
  return &Collection{INVALID_INT_VAL, s}
}
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示：

```go
[default] 
exten => s,1,Dial(Zap/1|30) 
exten => s,2,Voicemail(u100) 
exten => s,102,Voicemail(b100) 
exten => i,1,Voicemail(s0) 
```

任何命令行输入或输出都按如下方式编写：

```go
go get --help
```

**新术语**和**重要单词**以粗体显示。屏幕上看到的单词，例如菜单或对话框中的单词，以这种方式出现在文本中：“为了下载新模块，我们将转到文件 | 设置 | 项目名称 | 项目解释器。”

警告或重要说明如下。

技巧以这种方式出现。
