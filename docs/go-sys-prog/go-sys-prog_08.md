# 进程和信号

在上一章中，我们讨论了许多有趣的主题，包括处理 Unix 系统文件，处理 Go 中的日期和时间，查找有关文件权限和用户的信息，以及正则表达式和模式匹配。

本章的核心主题是开发能够处理 Unix 信号的 Go 应用程序。Go 提供了`os/signal`包来处理信号，它使用 Go 通道。尽管通道在下一章中得到了充分的探讨，但这并不妨碍你学习如何在 Go 程序中处理 Unix 信号。

此外，你将学习如何创建可以与 Unix 管道一起工作的 Go 命令行实用程序，如何在 Go 中绘制条形图，以及如何实现`cat(1)`实用程序的 Go 版本。因此，在本章中，你将学习以下主题：

+   列出 Unix 机器的进程

+   Go 中的信号处理

+   Unix 机器支持的信号以及如何使用`kill(1)`命令发送这些信号

+   让信号做你想要的工作

+   在 Go 中实现`cat(1)`实用程序的简单版本

+   在 Go 中绘制数据

+   使用管道将一个程序的输出发送到另一个程序

+   将一个大程序转换为两个较小的程序，它们将通过 Unix 管道协作

+   为 Unix 套接字创建一个客户端

# 关于 Unix 进程和信号

严格来说，**进程**是包含指令、用户数据和系统数据部分以及在运行时获得的其他类型资源的执行环境，而**程序**是一个包含指令和数据的文件，用于初始化进程的指令和用户数据部分。

# 进程管理

总的来说，Go 在处理进程和进程管理方面并不那么擅长。尽管如此，本节将介绍一个小的 Go 程序，通过执行 Unix 命令并获取其输出来列出 Unix 机器的所有进程。程序的名称将是`listProcess.go`。它适用于 Linux 和 macOS 系统，并将分为三个部分。

程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "os/exec" 
   "syscall" 
) 
```

`listProcess.go`的第二部分包含以下 Go 代码：

```go
func main() { 

   PS, err := exec.LookPath("ps") 
   if err != nil { 
         fmt.Println(err) 
   } 
fmt.Println(PS) 

   command := []string{"ps", "-a", "-x"} 
   env := os.Environ() 
   err = syscall.Exec(PS, command, env) 
```

正如你所看到的，你首先需要使用`exec.LookPath()`获取可执行文件的路径，以确保你不会意外地执行另一个二进制文件，然后使用切片定义你想要执行的命令，包括命令的参数。接下来，你将需要使用`os.Environ()`读取 Unix 环境。此外，你可以使用`syscall.Exec()`执行所需的命令，它将自动打印其输出，这并不是一个非常优雅的执行命令的方式，因为你无法控制任务，并且因为你是在最低级别调用进程，而不是使用更高级别的库，比如`os/exec`。

程序的最后一部分是用于打印前面代码的错误消息，如果有的话：

```go
   if err != nil { 
         fmt.Println(err) 
   } 
} 
```

执行`listProcess.go`将生成以下输出：使用`head(1)`实用程序来获取较小的输出：

```go
$ go run listProcess.go | head -3
/bin/ps
  PID TTY           TIME CMD
    1 ??         0:30.72 /sbin/launchd
signal: broken pipe
```

# 关于 Unix 信号

你是否曾经按下*Ctrl* + *C*来停止程序运行？如果是的话，那么你已经熟悉信号，因为*Ctrl* + *C*会向程序发送`SIGINT`信号。

严格来说，Unix**信号**是可以通过名称或数字访问的软件中断，提供了处理异步事件的方式，例如当子进程退出或在 Unix 系统上暂停进程时。

程序无法处理所有信号；其中一些信号是不可捕获和不可忽略的。`SIGKILL`和`SIGSTOP`信号无法被捕获、阻塞或忽略。原因是它们为内核和 root 用户提供了一种停止任何进程的方式。`SIGKILL`信号，也称为数字 9，通常在需要迅速采取行动的极端情况下调用；因此，它通常按数字调用，因为这样做更快。在这里要记住的最重要的事情是，并非所有的 Unix 信号都可以被处理！

# Go 中的 Unix 信号

Go 为程序员提供了`os/signal`包，以帮助他们处理传入的信号。但是，我们将从介绍`kill（1）`实用程序开始讨论处理。

# kill（1）命令

`kill（1）`命令用于终止进程或向其发送一个不那么残酷的信号。请记住，您可以向进程发送信号并不意味着该进程可以或者有代码来处理此信号。

默认情况下，`kill（1）`发送`SIGTERM`信号。如果要查找 Unix 机器支持的所有信号，应执行`kill -l`命令。在 macOS Sierra 机器上，`kill -l`的输出如下：

```go
$ kill -l
1) SIGHUP   2) SIGINT        3) SIGQUIT   4) SIGILL
5) SIGTRAP  6) SIGABRT       7) SIGEMT    8) SIGFPE
9) SIGKILL 10) SIGBUS        11) SIGSEGV 12) SIGSYS
13) SIGPIPE 14) SIGALRM       15) SIGTERM 16) SIGURG
17) SIGSTOP 18) SIGTSTP       19) SIGCONT 20) SIGCHLD
21) SIGTTIN 22) SIGTTOU       23) SIGIO   24) SIGXCPU
25) SIGXFSZ 26) SIGVTALRM     27) SIGPROF 28) SIGWINCH
29) SIGINFO 30) SIGUSR1       31) SIGUSR2
```

如果您在 Debian Linux 机器上执行相同的命令，您将获得更丰富的输出：

```go
$ kill -l
 1) SIGHUP   2) SIGINT   3) SIGQUIT  4) SIGILL   5) SIGTRAP
 6) SIGABRT  7) SIGBUS   8) SIGFPE   9) SIGKILL 10) SIGUSR1
11) SIGSEGV 12) SIGUSR2 13) SIGPIPE 14) SIGALRM 15) SIGTERM
16) SIGSTKFLT     17) SIGCHLD 
18) SIGCONT       19) SIGSTOP 20) SIGTSTP
21) SIGTTIN       22) SIGTTOU 
23) SIGURG        24) SIGXCPU 25) SIGXFSZ
26) SIGVTALRM     27) SIGPROF 28) SIGWINCH 
29) SIGIO         30) SIGPWR
31) SIGSYS        34) SIGRTMIN 
35) SIGRTMIN+1    36) SIGRTMIN+2    37) SIGRTMIN+3
38) SIGRTMIN+4    39) SIGRTMIN+5 
40) SIGRTMIN+6    41) SIGRTMIN+7    42) SIGRTMIN+8
43) SIGRTMIN+9    44) SIGRTMIN+10 
45) SIGRTMIN+11   46) SIGRTMIN+12   47) SIGRTMIN+13
48) SIGRTMIN+14   49) SIGRTMIN+15 
50) SIGRTMAX-14   51) SIGRTMAX-13   52) SIGRTMAX-12
53) SIGRTMAX-11   54) SIGRTMAX-10 
55) SIGRTMAX-9    56) SIGRTMAX-8    57) SIGRTMAX-7
58) SIGRTMAX-6    59) SIGRTMAX-5 
60) SIGRTMAX-4    61) SIGRTMAX-3    62) SIGRTMAX-2
63) SIGRTMAX-1    64) SIGRTMAX
```

如果您尝试杀死或向另一个用户的进程发送另一个信号而没有所需的权限，这很可能会发生，如果您不是*root*用户，`kill（1）`将无法完成任务，并且您将收到类似以下的错误消息：

```go
$ kill 2908
-bash: kill: (2908) - Operation not permitted
```

# Go 中的简单信号处理程序

本小节将介绍一个简单的 Go 程序，仅处理`SIGTERM`和`SIGINT`信号。`h1s.go`的 Go 代码将分为三部分呈现；第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "os/signal" 
   "syscall" 
   "time" 
) 

func handleSignal(signal os.Signal) { 
   fmt.Println("Got", signal) 
} 
```

除了程序的序言之外，还有一个名为`handleSignal（）`的函数，当程序接收到两个支持的信号中的任何一个时，将调用该函数。

`h1s.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   sigs := make(chan os.Signal, 1) 
   signal.Notify(sigs, os.Interrupt, syscall.SIGTERM) 
   go func() { 
         for { 
               sig := <-sigs 
               fmt.Println(sig) 
               handleSignal(sig) 
         } 
   }() 
```

先前的代码使用了**goroutine**和 Go**channel**，这是本书中尚未讨论的 Go 功能。不幸的是，您必须等到第九章*，* *Goroutines - Basic Features*，才能了解更多关于它们的信息。请注意，尽管`os.Interrupt`和`syscall.SIGTERM`属于不同的 Go 包，但它们都是信号。

目前，理解这种技术很重要；它包括三个步骤：

1.  通道的定义，作为传递数据的方式，对于技术（`sigs`）是必需的。

1.  调用`signal.Notify（）`以定义您希望能够捕获的信号列表。

1.  定义一个匿名函数，它在`signal.Notify（）`之后的 goroutine（`go func（）`）中运行，用于决定在收到所需信号时要执行的操作。

在这种情况下，将调用`handleSignal（）`函数。匿名函数内部的`for`循环用于使程序保持处理所有信号，并在接收到第一个信号后不停止。

`h1s.go`的最后部分如下：

```go
   for { 
         fmt.Printf(".") 
         time.Sleep(10 * time.Second) 
   } 
} 
```

这是一个无限的`for`循环，它永远延迟程序的结束：在其位置上，您很可能会放置程序的实际代码。执行`h1s.go`并从另一个终端向其发送信号将使`h1s.go`生成以下输出：

```go
$ ./h1s
......................^Cinterrupt
Got interrupt
^Cinterrupt
Got interrupt
.Hangup: 1
```

这里的坏处是，当接收到`SIGHUP`信号时，`h1s.go`将停止，因为当程序没有专门处理`SIGHUP`时，默认操作是杀死进程！下一小节将展示如何更好地处理三个信号，之后的小节将教您如何处理所有可处理的信号。

# 处理三种不同的信号！

这一小节将教您如何创建一个可以处理三种不同信号的 Go 应用程序：程序的名称将是`h2s.go`，它将处理`SIGTERM`、`SIGINT`和`SIGHUP`信号。

`h2s.go`的 Go 代码将分为四部分呈现。

程序的第一部分包含了预期的序言：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "os/signal" 
   "syscall" 
   "time" 
) 
```

第二部分包含以下 Go 代码：

```go
func handleSignal(signal os.Signal) { 
   fmt.Println("* Got:", signal) 
} 

func main() { 
   sigs := make(chan os.Signal, 1) 
   signal.Notify(sigs, os.Interrupt, syscall.SIGTERM, syscall.SIGHUP) 
```

在这里，最后一句告诉您，程序只会处理`os.Interrupt`、`syscall.SIGTERM`和`syscall.SIGHUP`信号。

`h2s.go`的第三部分如下：

```go
   go func() { 
         for { 
               sig := <-sigs 
               switch sig { 
               case os.Interrupt: 
                     handleSignal(sig) 
               case syscall.SIGTERM: 
                     handleSignal(sig) 
               case syscall.SIGHUP: 
                     fmt.Println("Got:", sig) 
                     os.Exit(-1) 
               } 
         } 
   }() 
```

在这里，您可以看到，当捕获到特定信号时，不一定要调用单独的函数；也可以在`for`循环内处理它，就像`syscall.SIGHUP`一样。但是，我认为使用命名函数更好，因为它使 Go 代码更易于阅读和修改。好处是 Go 有一个处理所有信号的中心位置，这使得很容易找出程序的运行情况。

此外，`h2s.go`专门处理`SIGHUP`信号，尽管`SIGHUP`信号仍将终止程序；但是，这次是我们的决定。

请记住，通常最好让一个信号处理程序来停止程序，否则您将不得不通过发出`kill -9`命令来终止它。

`h2s.go`的最后一部分如下：

```go
   for { 
         fmt.Printf(".") 
         time.Sleep(10 * time.Second) 
   } 
}
```

执行`h2s.go`并从另一个 shell 发送四个信号（`SIGINT`、`SIGTERM`、`SIGHUP`和`SIGKILL`）给它将生成以下输出：

```go
$ go build h2s.go
$ ./h2s
..* Got: interrupt
* Got: terminated
.Got: hangup
.Killed: 9
```

构建`h2s.go`的原因是更容易找到自主程序的进程 ID：`go run`命令在后台构建了一个临时可执行程序，这种情况下提供的灵活性较少。如果要改进`h2s.go`，可以让它调用`os.Getpid()`来打印其进程 ID，这样就不必自己查找了。

程序在收到无法处理的`SIGKILL`信号之前处理了三个信号，因此终止了！

# 捕获每个可以处理的信号

这一小节将介绍一种简单的技术，允许您捕捉每个可以处理的信号：再次强调，您不能处理所有信号！程序将在收到`SIGTERM`信号后停止运行。

程序的名称将是`catchAll.go`，将分为三部分呈现。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "os" 
   "os/signal" 
   "syscall" 
   "time" 
) 

func handleSignal(signal os.Signal) { 
   fmt.Println("* Got:", signal) 
} 
```

程序的第二部分如下：

```go
func main() { 
   sigs := make(chan os.Signal, 1) 
   signal.Notify(sigs) 
   go func() { 
         for { 
               sig := <-sigs 
               switch sig { 
               case os.Interrupt: 
                     handleSignal(sig) 
               case syscall.SIGTERM: 
                     handleSignal(sig) 
                     os.Exit(-1) 
               case syscall.SIGUSR1: 
                     handleSignal(sig) 
               default: 
                     fmt.Println("Ignoring:", sig) 
               } 
         } 
   }() 
```

在这种情况下，调用`signal.Notify()`的方式对您的代码产生了影响。如果您没有定义任何特定的信号，程序将能够处理任何可以处理的信号。但是，匿名函数内的`for`循环只处理了三个信号，而忽略了其余的！请注意，我认为这是在 Go 中处理信号的最佳方式：捕获一切，同时只处理您感兴趣的信号。但是，有些人认为明确处理您处理的内容是更好的方法。这里没有对错之分。

`catchAll.go`程序在收到`SIGHUP`时不会终止，因为`switch`块的`default`情况处理了它。

最后一部分是对`time.Sleep()`函数的预期调用：

```go
   for { 
         fmt.Printf(".") 
         time.Sleep(10 * time.Second) 
   } 
} 
```

执行`catchAll.go`将产生以下输出：

```go
$ ./catchAll
.Ignoring: hangup
.......................................* Got: interrupt
* Got: user defined signal 1
.Ignoring: user defined signal 2
Ignoring: hangup
.* Got: terminated
$
```

# 重新审视旋转日志文件！

正如我在第七章中告诉过您，本章将向您介绍一种技术，可以让您以更常规的方式结束程序并旋转日志文件，这是通过信号和信号处理来实现的。

`rotateLog.go`的新版本名称将是`rotateSignals.go`，将分为四个部分呈现。此外，当实用程序接收`os.Interrupt`时，它将旋转当前日志文件，而当它接收`syscall.SIGTERM`时，它将终止执行。可以处理的任何其他信号都将创建一个日志条目，而不会执行其他操作。

`rotateSignals.go`的第一部分是预期的序言：

```go
package main 

import ( 
   "fmt" 
   "log" 
   "os" 
   "os/signal" 
   "strconv" 
   "syscall" 
   "time" 
) 

var TOTALWRITES int = 0 
var openLogFile os.File 
```

`rotateSignals.go`的第二部分包含以下 Go 代码：

```go
func rotateLogFile(filename string) error { 
   openLogFile.Close() 
   os.Rename(filename, filename+"."+strconv.Itoa(TOTALWRITES)) 
   err := setUpLogFile(filename) 
   return err 
} 

func setUpLogFile(filename string) error { 
   openLogFile, err := os.OpenFile(filename, os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644) 
   if err != nil { 
         return err 
   } 
   log.SetOutput(openLogFile) 
   return nil 
} 
```

您刚刚在这里定义了两个执行两项任务的函数。`rotateSignals.go`的第三部分包含以下 Go 代码：

```go
func main() { 
   filename := "/tmp/myLog.log" 
   err := setUpLogFile(filename) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(-1) 
   } 

   sigs := make(chan os.Signal, 1) 
   signal.Notify(sigs) 
```

再次，所有信号都将被捕获。`rotateSignals.go`的最后一部分如下：

```go
   go func() { 
         for { 
               sig := <-sigs 
               switch sig { 
               case os.Interrupt: 
                     rotateLogFile(filename) 
                     TOTALWRITES++ 
               case syscall.SIGTERM: 
                     log.Println("Got:", sig) 
                     openLogFile.Close() 
                     TOTALWRITES++ 
                     fmt.Println("Wrote", TOTALWRITES, "log entries in total!") 
                     os.Exit(-1) 
               default: 
                     log.Println("Got:", sig) 
                     TOTALWRITES++ 
               } 
         } 
   }() 

   for { 
         time.Sleep(10 * time.Second) 
   } 
} 
```

正如您所看到的，`rotateSignals.go`通过为每个信号编写一个日志条目记录了它接收到的信号的信息。虽然呈现`rotateSignals.go`的整个代码是不错的，但是看到`diff(1)`实用程序的输出以显示`rotateLog.go`和`rotateSignals.go`之间的代码差异将是非常有教育意义的：

```go
$ diff rotateLog.go rotateSignals.go
6a7
>     "os/signal"
7a9
>     "syscall"
12,13d13
< var ENTRIESPERLOGFILE int = 100
< var WHENTOSTOP int = 230
33d32
<     numberOfLogEntries := 0
41,51c40,59
<     for {
<           log.Println(numberOfLogEntries, "This is a test log entry")
<           numberOfLogEntries++
<           TOTALWRITES++
<           if numberOfLogEntries > ENTRIESPERLOGFILE {
<                 _ = rotateLogFile(filename)
<                 numberOfLogEntries = 0
<           }
<           if TOTALWRITES > WHENTOSTOP {
<                 _ = rotateLogFile(filename)
<                 break
---
>     sigs := make(chan os.Signal, 1)
>     signal.Notify(sigs)
>
>     go func() {
>           for {
>                 sig := <-sigs
>                 switch sig {
>                 case os.Interrupt:
>                       rotateLogFile(filename)
>                       TOTALWRITES++
>                 case syscall.SIGTERM:
>                       log.Println("Got:", sig)
>                       openLogFile.Close()
>                       TOTALWRITES++
>                       fmt.Println("Wrote", TOTALWRITES, "log entries in total!")
>                       os.Exit(-1)
>                 default:
>                       log.Println("Got:", sig)
>                       TOTALWRITES++
>                 }
53c61,64
<           time.Sleep(time.Second)
---
>     }()
>
>     for {
>           time.Sleep(10 * time.Second)
55d65
<     fmt.Println("Wrote", TOTALWRITES, "log entries!")
```

这里的好处是，在`rotateSignals.go`中使用信号使得`rotateLog.go`中使用的大多数全局变量变得不必要，因为现在您可以通过发送信号来控制实用程序。此外，`rotateSignals.go`的设计和结构比`rotateLog.go`更简单，因为您只需要理解匿名函数的功能。

执行`rotateSignals.go`并向其发送一些信号后，`/tmp/myLog.log`的内容将如下所示：

```go
$ cat /tmp/myLog.log
2017/06/03 14:53:33 Got: user defined signal 1
2017/06/03 14:54:08 Got: user defined signal 1
2017/06/03 14:54:12 Got: user defined signal 2
2017/06/03 14:54:19 Got: terminated
```

此外，您将在`/tmp`目录下有以下文件：

```go
$ ls -l /tmp/myLog.log*
-rw-r--r--  1 mtsouk  wheel  177 Jun  3 14:54 /tmp/myLog.log
-rw-r--r--  1 mtsouk  wheel  106 Jun  3 13:42 /tmp/myLog.log.0
```

# 改进文件复制

当`cp(1)`实用程序接收`SIGINFO`信号时，它会打印有用的信息，如下所示：

```go
$ cp FileToCopy /tmp/copy
FileToCopy -> /tmp/copy  26%
FileToCopy -> /tmp/copy  29%
FileToCopy -> /tmp/copy  31%
```

因此，本节的其余部分将为`cp(1)`命令的 Go 实现实现相同的功能。本节中的 Go 代码将基于`cp.go`程序，因为当使用较小的缓冲区大小时，它可能非常慢，从而为我们提供测试时间。新的复制实用程序的名称将是`cpSignal.go`，将分为四个部分呈现。

`cpSignal.go`和`cp.go`之间的基本区别在于`cpSignal.go`应该找到输入文件的大小，并在给定点保持已写入的字节数。除了这些修改之外，您不必担心其他任何事情，因为两个版本的核心功能，即复制文件，完全相同。

程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "os" 
   "os/signal" 
   "path/filepath" 
   "strconv" 
   "syscall" 
) 

var BUFFERSIZE int64 
var FILESIZE int64 
var BYTESWRITTEN int64 
```

为了使开发人员更容易，程序引入了两个名为`FILESIZE`和`BYTESWRITTEN`的全局变量，它们分别保持输入文件的大小和已写入的字节数。这两个变量都被处理`SIGINFO`信号的函数使用。

第二部分如下：

```go
func Copy(src, dst string, BUFFERSIZE int64) error { 
   sourceFileStat, err := os.Stat(src) 
   if err != nil { 
         return err 
   } 

   FILESIZE = sourceFileStat.Size() 

   if !sourceFileStat.Mode().IsRegular() { 
         return fmt.Errorf("%s is not a regular file.", src) 
   } 

   source, err := os.Open(src) 
   if err != nil { 
         return err 
   } 
   defer source.Close() 

   _, err = os.Stat(dst) 
   if err == nil { 
         return fmt.Errorf("File %s already exists.", dst) 
   } 

   destination, err := os.Create(dst) 
   if err != nil { 
         return err 
   } 
   defer destination.Close() 

   if err != nil { 
         panic(err) 
   } 

   buf := make([]byte, BUFFERSIZE) 
   for { 
         n, err := source.Read(buf) 
         if err != nil && err != io.EOF { 
               return err 
         } 
         if n == 0 { 
               break 
         } 
         if _, err := destination.Write(buf[:n]); err != nil { 
               return err 
         } 
         BYTESWRITTEN = BYTESWRITTEN + int64(n) 
   } 
   return err 
} 
```

在这里，您使用`sourceFileStat.Size()`函数获取输入文件的大小，并设置`FILESIZE`全局变量的值。

第三部分是您定义信号处理的地方：

```go
func progressInfo() { 
   progress := float64(BYTESWRITTEN) / float64(FILESIZE) * 100 
   fmt.Printf("Progress: %.2f%%\n", progress) 
} 

func main() { 
   if len(os.Args) != 4 { 
         fmt.Printf("usage: %s source destination BUFFERSIZE\n", filepath.Base(os.Args[0])) 
         os.Exit(1) 
   } 

   source := os.Args[1] 
   destination := os.Args[2] 
   BUFFERSIZE, _ = strconv.ParseInt(os.Args[3], 10, 64) 
   BYTESWRITTEN = 0 

   sigs := make(chan os.Signal, 1) 
   signal.Notify(sigs) 
```

在这里，您选择捕获所有信号。但是，匿名函数的 Go 代码只会在接收到`syscall.SIGINFO`信号后调用`progressInfo()`。

如果您想要一种优雅地终止程序的方法，您可能希望使用`SIGINT`信号，因为当捕获所有信号时，优雅地终止程序将不再可能：您将需要发送`SIGKILL`来终止程序，这有点残酷。

`cpSignal.go`的最后一部分如下：

```go
   go func() { 
         for {
               sig := <-sigs 
               switch sig { 
               case syscall.SIGINFO:
                     progressInfo() 
               default: 
                     fmt.Println("Ignored:", sig) 
               } 
         } 
   }() 

   fmt.Printf("Copying %s to %s\n", source, destination) 
   err := Copy(source, destination, BUFFERSIZE) 
   if err != nil { 
         fmt.Printf("File copying failed: %q\n", err) 
   } 
} 
```

执行`cpSignal.go`并向其发送两个`SIGINFO`信号将生成以下输出：

```go
$ ./cpSignal FileToCopy /tmp/copy 2
Copying FileToCopy to /tmp/copy
Ignored: user defined signal 1
Progress: 21.83%
^CIgnored: interrupt
Progress: 29.78%
```

# 绘制数据

本节将开发一个实用程序，它将读取多个日志文件，并将创建一个图像，其中每个条将表示在日志文件中找到给定 IP 地址的次数。

然而，Unix 哲学告诉我们，我们应该制作两个不同的实用程序，而不是开发一个单一的实用程序：一个用于处理日志文件并创建报告，另一个用于绘制第一个实用程序生成的数据：这两个实用程序将使用 Unix 管道进行通信。尽管本节将实现第一种方法，但您将在本章的*The * `plotIP.go` *utility revisited*部分中看到第二种方法的实现。

所提供实用程序的想法来自我为一本杂志撰写的教程，我在其中开发了一个小型的 Go 程序进行绘图：即使是小型和天真的程序也可以激发您开发更大的东西，因此不要低估它们的力量。

实用程序的名称将是`plotIP.go`，并且将分为七个部分：好处是`plotIP.go`将重用`countIP.go`和`findIP.go`的一些代码。`plotIP.go`唯一不做的事情就是将文本写入图像，因此您只能绘制条形图，而不知道实际值或特定条形图的相应日志文件：您可以尝试将文本功能添加到程序中作为练习。

此外，`plotIP.go`将需要至少三个参数，即图像的宽度和高度以及将要使用的日志文件的名称：为了使`plotIP.go`更小，`plotIP.go`将不使用`flag`包，并假定您将按正确的顺序提供其参数。如果您提供更多的参数，它将把它们视为日志文件。

`plotIP.go`的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "image" 
   "image/color" 
   "image/png" 
   "io" 
   "os" 
   "path/filepath" 
   "regexp" 
   "strconv" 
) 

var m *image.NRGBA
var x int 
var y int 
var barWidth int 
```

这些全局变量与图像的尺寸（`x`和`y`）、图像作为 Go 变量（`m`）以及其中一个条形图的宽度（`barWidth`）有关，该宽度取决于图像的大小和将要绘制的条形图的数量。请注意，在这里使用`x`和`y`作为变量名而不是像`IMAGEWIDTH`和`IMAGEHEIGHT`之类的名称可能有点错误和危险。

第二部分是以下内容：

```go
func findIP(input string) string { 
   partIP := "(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])" 
   grammar := partIP + "\\." + partIP + "\\." + partIP + "\\." + partIP 
   matchMe := regexp.MustCompile(grammar) 
   return matchMe.FindString(input) 
} 

func plotBar(width int, height int, color color.RGBA) { 
   xx := 0
   for xx < barWidth { 
         yy := 0 
         for yy < height { 
               m.Set(xx+width, y-yy, color) 
               yy = yy + 1 
         } 
         xx = xx + 1 
   } 
} 
```

在这里，您实现了一个名为`plotBar()`的 Go 函数，该函数根据条形图的高度、宽度和颜色进行绘制。这个函数是`plotIP.go`中最具挑战性的部分。

第三部分包含以下 Go 代码：

```go
func getColor(x int) color.RGBA { 
   switch {

   case x == 0: 
         return color.RGBA{0, 0, 255, 255} 
   case x == 1: 
         return color.RGBA{255, 0, 0, 255} 
   case x == 2: 
         return color.RGBA{0, 255, 0, 255} 
   case x == 3: 
         return color.RGBA{255, 255, 0, 255} 
   case x == 4: 
         return color.RGBA{255, 0, 255, 255} 
   case x == 5: 
         return color.RGBA{0, 255, 255, 255} 
   case x == 6: 
         return color.RGBA{255, 100, 100, 255} 
   case x == 7: 
         return color.RGBA{100, 100, 255, 255} 
   case x == 8: 
         return color.RGBA{100, 255, 255, 255} 
   case x == 9: 
         return color.RGBA{255, 255, 255, 255} 
   } 
   return color.RGBA{0, 0, 0, 255} 
} 
```

此函数允许您定义输出中将出现的颜色：如果需要，可以更改它们。

第四部分包含以下 Go 代码：

```go
func main() { 
   var data []int 
   arguments := os.Args 
   if len(arguments) < 4 { 
         fmt.Printf("%s X Y IP input\n", filepath.Base(arguments[0])) 
         os.Exit(0) 
   } 

   x, _ = strconv.Atoi(arguments[1]) 
   y, _ = strconv.Atoi(arguments[2]) 
   WANTED := arguments[3] 
   fmt.Println("Image size:", x, y) 
```

在这里，您可以读取所需的 IP 地址，该地址保存在`WANTED`变量中，并读取生成的 PNG 图像的尺寸。

第五部分包含以下 Go 代码：

```go
   for _, filename := range arguments[4:] { 
         count := 0 
         fmt.Println(filename) 
         f, err := os.Open(filename) 
         if err != nil { 
               fmt.Fprintf(os.Stderr, "Error: %s\n", err) 
               continue 
         } 
         defer f.Close() 

         r := bufio.NewReader(f) 
         for { 
               line, err := r.ReadString('\n') 
               if err == io.EOF { 
                     break 
               } 

if err != nil { 
                fmt.Fprintf(os.Stderr, "Error in file: %s\n", err) 
                     continue 
               } 
               ip := findIP(line) 
               if ip == WANTED { 
                     count++

               } 
         } 
         data = append(data, count) 
   } 
```

在这里，您逐个处理输入的日志文件，并将计算的值存储在`data`切片中。错误消息将打印到`os.Stderr`：从将错误消息打印到`os.Stderr`中获得的主要优势是，您可以轻松地将错误消息重定向到文件，同时以不同的方式使用写入到`os.Stdout`的数据。

`plotIP.go`的第六部分包含以下 Go 代码：

```go
   fmt.Println("Slice length:", len(data)) 
   if len(data)*2 > x { 
         fmt.Println("Image size (x) too small!") 
         os.Exit(-1) 
   } 

   maxValue := data[0] 
   for _, temp := range data { 
         if maxValue < temp { 
               maxValue = temp 
         } 
   } 

   if maxValue > y { 
         fmt.Println("Image size (y) too small!") 
         os.Exit(-1) 
   } 
   fmt.Println("maxValue:", maxValue) 
   barHeighPerUnit := int(y / maxValue) 
   fmt.Println("barHeighPerUnit:", barHeighPerUnit) 
   PNGfile := WANTED + ".png" 
   OUTPUT, err := os.OpenFile(PNGfile, os.O_CREATE|os.O_WRONLY, 0644) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(-1) 
   } 
   m = image.NewNRGBA(image.Rectangle{Min: image.Point{0, 0}, Max: image.Point{x, y}}) 
```

在这里，您可以计算有关绘图的事项，并使用`os.OpenFile()`创建输出图像文件。由`plotIP.go`实用程序生成的 PNG 文件以给定的 IP 地址命名，以使事情变得更简单。

`plotIP.go`的 Go 代码的最后一部分如下：

```go
   i := 0 
   barWidth = int(x / len(data)) 
   fmt.Println("barWidth:", barWidth) 
   for _, v := range data { 
         c := getColor(v % 10) 
         yy := v * barHeighPerUnit 
         plotBar(barWidth*i, yy, c) 
         fmt.Println("plotBar", barWidth*i, yy) 
         i = i + 1 
   } 
   png.Encode(OUTPUT, m) 
} 
```

在这里，您可以读取`data`切片的值，并通过调用`plotBar()`函数为每个值创建一个条形图。

执行`plotIP.go`将生成以下输出：

```go
$ go run plotIP.go 1300 1500 127.0.0.1 /tmp/log.*
Image size: 1300 1500
/tmp/log.1
/tmp/log.2
/tmp/log.3
Slice length: 3
maxValue: 1500
barHeighPerUnit: 1
barWidth: 433
plotBar 0 1500
plotBar 433 1228
plotBar 866 532
$  ls -l 127.0.0.1.png
-rw-r--r-- 1 mtsouk mtsouk 11023 Jun  5 18:36 127.0.0.1.png
```

然而，除了生成的文本输出之外，重要的是生成的 PNG 文件，可以在以下图中看到：

![](img/0705a55e-044d-4918-bfea-70d6b7d9377e.png)

由 plotIP.go 实用程序生成的输出

如果要将错误消息保存到不同的文件中，可以使用以下命令的变体：

```go
$ go run plotIP.go 130 150 127.0.0.1 doNOTExist 2> err
Image size: 130 150
doNOTExist
Slice length: 0
$ cat err
Error: open doNOTExist: no such file or directory
panic: runtime error: index out of range

goroutine 1 [running]:
main.main()
     /Users/mtsouk/Desktop/goBook/ch/ch8/code/plotIP.go:112 +0x12de
exit status 2
```

以下命令通过将其发送到`/dev/null`来丢弃所有错误消息：

```go
$ go run plotIP.go 1300 1500 127.0.0.1 doNOTExist 2>/dev/null
Image size: 1300 1500
doNOTExist
Slice length: 0  
```

# 在 Go 中的 Unix 管道

我们在第六章*，*文件输入和输出中首次讨论了管道。管道有两个严重的限制：首先，它们通常是单向通信的，其次，它们只能在具有共同祖先的进程之间使用。

管道背后的一般思想是，如果您没有要处理的文件，应该等待从标准输入获取输入。同样，如果没有要求将输出保存到文件，应该将输出写入标准输出，供用户查看或供其他程序处理。因此，管道可用于在两个进程之间流式传输数据，而不创建任何临时文件。

本节将呈现一些使用 Unix 管道编写的简单实用程序，以增加清晰度。

# 从标准输入读取

为了开发支持 Unix 管道的 Go 应用程序，您需要知道如何从标准输入读取。

开发的程序名为`readSTDIN.go`，将分为三部分呈现。

程序的第一部分是预期的序言：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "os" 
) 
```

`readSTDIN.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   filename := "" 
   var f *os.File 
   arguments := os.Args 
   if len(arguments) == 1 { 
         f = os.Stdin 
   } else { 
         filename = arguments[1] 
         fileHandler, err := os.Open(filename) 
         if err != nil { 
               fmt.Printf("error opening %s: %s", filename, err) 
               os.Exit(1) 
         } 
         f = fileHandler 
   } 
   defer f.Close() 
```

在这里，您可以确定是否有实际文件要处理，这可以通过程序的命令行参数数量来确定。如果没有要处理的文件，您将尝试从`os.Stdin`读取数据。确保您理解所呈现的技术，因为在本章中将多次使用它。

`readSTDIN.go`的最后一部分如下：

```go
   scanner := bufio.NewScanner(f) 
   for scanner.Scan() { 
         fmt.Println(">", scanner.Text()) 
   } 
} 
```

这段代码无论是处理实际文件还是`os.Stdin`都是一样的，这是因为在 Unix 中一切都是文件。请注意，程序输出以`>`字符开头。

执行`readSTDIN.go`将生成以下输出：

```go
$ cat /tmp/testfile
1
2
$ go run readSTDIN.go /tmp/testFile
> 1
> 2
$ cat /tmp/testFile | go run readSTDIN.go
> 1
> 2
$ go run readSTDIN.go
3
> 3
2
> 2
1
> 1
```

在最后一种情况下，`readSTDIN.go`会回显它读取的每一行，因为输入是逐行读取的：`cat(1)`实用程序的工作方式相同。

# 将数据发送到标准输出

本小节将向您展示如何以比仅使用`fmt.Println()`或`fmt`标准 Go 包中的任何其他函数更好的方式将数据发送到标准输出。Go 程序将被命名为`writeSTDOUT.go`，并将分为三部分呈现给您。

第一部分如下：

```go
package main 

import ( 
   "io" 
   "os" 
) 
```

`writeSTDOUT.go`的第二部分包含以下 Go 代码：

```go
func main() { 
   myString := "" 
   arguments := os.Args 
   if len(arguments) == 1 { 
         myString = "You did not give an argument!" 
   } else { 
         myString = arguments[1] 
   } 
```

`writeSTDOUT.go`的最后一部分如下：

```go
   io.WriteString(os.Stdout, myString) 
   io.WriteString(os.Stdout, "\n") 
} 
```

唯一微妙的是，在使用`io.WriteString()`将数据写入`os.Stdout`之前，您需要将文本放入一个切片中。

执行`writeSTDOUT.go`将生成以下输出：

```go
$ go run writeSTDOUT.go 123456
123456
$ go run writeSTDOUT.go
You do not give an argument!
```

# 在 Go 中实现 cat(1)

本小节将呈现`cat(1)`命令行实用程序的 Go 版本。如果您向`cat(1)`提供一个或多个命令行参数，那么`cat(1)`将在屏幕上打印它们的内容。但是，如果您只在 Unix shell 中键入`cat(1)`，那么`cat(1)`将等待您的输入，当您键入*Ctrl* + *D*时输入将终止。

Go 实现的名称将是`cat.go`，将分为三部分呈现。

`cat.go`的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "io" 
   "os" 
) 
```

第二部分如下：

```go
func catFile(filename string) error { 
   f, err := os.Open(filename) 
   if err != nil { 
         return err 
   } 
   defer f.Close() 
   scanner := bufio.NewScanner(f) 
   for scanner.Scan() { 
         fmt.Println(scanner.Text()) 
   } 
   return nil 
} 
```

当`cat.go`实用程序需要处理真实文件时，将调用`catFile()`函数。有一个函数来完成您的工作可以使程序设计更好。

最后一部分包含以下 Go 代码：

```go
func main() { 
   filename := "" 
   arguments := os.Args 
   if len(arguments) == 1 { 
         io.Copy(os.Stdout, os.Stdin) 
         os.Exit(0) 
   } 

   filename = arguments[1] 
   err := catFile(filename) 
   if err != nil { 
         fmt.Println(err) 
   } 
} 
```

因此，如果程序没有参数，则假定它必须从`os.Stdin`读取。在这种情况下，它只会回显您给它的每一行。如果程序有参数，则它将使用`catFile()`函数处理第一个参数作为文件。

执行`cat.go`将生成以下输出：

```go
$ go run cat.go /tmp/testFile  |  go run cat.go
1
2
$ go run cat.go
Mihalis
Mihalis
Tsoukalos
Tsoukalos $ echo "Mihalis Tsoukalos" | go run cat.go
Mihalis Tsoukalos
```

# 重新审视 plotIP.go 实用程序

正如本章的前一节所承诺的，本节将创建两个单独的实用程序，结合起来将实现`plotIP.go`的功能。个人而言，我更喜欢有两个单独的实用程序，并在需要时将它们结合起来，而不是只有一个实用程序可以执行两个或更多任务。

这两个实用程序的名称将是`extractData.go`和`plotData.go`。正如您可以轻松理解的那样，只有第二个实用程序才能够从标准输入获取输入，只要第一个实用程序将其输出打印在标准输出上，要么使用`os.Stdout`，这是正确的方式，要么使用`fmt.Println()`，通常可以完成任务。

我认为我现在应该告诉您我的小秘密：我首先创建了`extractData.go`和`plotData.go`，然后开发了`plotIP.go`，因为开发两个单独的实用程序比开发一个做所有事情的大型实用程序更容易！此外，使用两个不同的实用程序允许您使用标准 Unix 实用程序（如`tail(1)`、`sort(1)`和`head(1)`）过滤`extractData.go`的输出，这意味着您可以以不同的方式修改数据，而无需编写任何额外的 Go 代码。

将两个命令行实用程序并创建一个实用程序来实现这两个实用程序的功能要比将一个大型实用程序分割成两个或更多不同实用程序的功能更容易，因为后者通常需要更多的变量和更多的错误检查。

`extractData.go`实用程序将分为四个部分；第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "io" 
   "os" 
   "path/filepath" 
   "regexp" 
) 
```

`extractData.go`的第二部分包含以下 Go 代码：

```go
func findIP(input string) string { 
   partIP := "(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])" 
   grammar := partIP + "\\." + partIP + "\\." + partIP + "\\." + partIP 
   matchMe := regexp.MustCompile(grammar) 
   return matchMe.FindString(input) 
} 
```

您应该熟悉`findIP()`函数，您在第七章中看到了`findIP.go`。

`extractData.go`的第三部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) < 3 { 
         fmt.Printf("%s IP <files>\n", filepath.Base(os.Args[0])) 
         os.Exit(-1) 
   } 

   WANTED := arguments[1] 
   for _, filename := range arguments[2:] { 
         count := 0 
         buf := []byte(filename)
         io.WriteString(os.Stdout, string(buf)) 
         f, err := os.Open(filename) 
         if err != nil { 
               fmt.Fprintf(os.Stderr, "Error: %s\n", err) 
               continue 
         } 
         defer f.Close() 
```

这里使用`buf`变量是多余的，因为`filename`是一个字符串，`io.WriteString()`期望一个字符串：这只是我的习惯，将`filename`的值放入字节片中。如果您愿意，可以将其删除。

再次，大部分 Go 代码来自`plotIP.go`实用程序。`extractData.go`的最后一部分如下：

```go
         r := bufio.NewReader(f) 
         for { 
               line, err := r.ReadString('\n') 
               if err == io.EOF { 
                     break 
               } else if err != nil { 
                     fmt.Fprintf(os.Stderr, "Error in file: %s\n", err) 
                     continue 
               } 

               ip := findIP(line) 
               if ip == WANTED { 
                     count = count + 1 
               } 
         } 
         buf = []byte(strconv.Itoa(count))
         io.WriteString(os.Stdout, " ") 
         io.WriteString(os.Stdout, string(buf)) 
         io.WriteString(os.Stdout, "\n") 
   } 
} 
```

在这里，`extractData.go`将其输出写入标准输出（`os.Stdout`），而不是使用`fmt`包的函数，以便更兼容管道。`extractData.go`实用程序至少需要两个参数：IP 地址和日志文件，但它可以处理任意数量的日志文件。

您可能希望将第三部分中的`filename`值的打印移至此处，以便将所有打印命令放在同一位置。

执行`extractData.go`将生成以下输出：

```go
$ ./extractData 127.0.0.1 access.log{,.1}
access.log 3099
access.log.1 6333
```

虽然`extractData.go`在每行打印两个值，但`plotData.go`只会使用第二个字段。最好的方法是使用`awk(1)`过滤`extractData.go`的输出：

```go
$ ./extractData 127.0.0.1 access.log{,.1} | awk '{print $2}'
3099
6333
```

正如您所理解的，`awk(1)`允许您对生成的值进行更多操作。

`plotData.go`实用程序也将分为六个部分；它的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "image" 
   "image/color" 
   "image/png" 
   "os" 
   "path/filepath" 
   "strconv" 
) 

var m *image.NRGBA 
var x int 
var y int 
var barWidth int 
```

再次，使用全局变量是为了避免向实用程序的某些函数传递太多参数。

`plotData.go`的第二部分包含以下 Go 代码：

```go
func plotBar(width int, height int, color color.RGBA) { 
   xx := 0
   for xx < barWidth { 
         yy := 0 
         for yy < height { 
               m.Set(xx+width, y-yy, color) 
               yy = yy + 1 
         } 
         xx = xx + 1 
   } 
} 
```

`plotData.go`的第三部分包含以下 Go 代码：

```go
func getColor(x int) color.RGBA { 
   switch {
   case x == 0: 
         return color.RGBA{0, 0, 255, 255} 
   case x == 1: 
         return color.RGBA{255, 0, 0, 255} 
   case x == 2: 
         return color.RGBA{0, 255, 0, 255} 
   case x == 3: 
         return color.RGBA{255, 255, 0, 255} 
   case x == 4: 
         return color.RGBA{255, 0, 255, 255} 
   case x == 5: 
         return color.RGBA{0, 255, 255, 255} 
   case x == 6: 
         return color.RGBA{255, 100, 100, 255} 
   case x == 7: 
         return color.RGBA{100, 100, 255, 255} 
   case x == 8: 
         return color.RGBA{100, 255, 255, 255} 
   case x == 9: 
         return color.RGBA{255, 255, 255, 255} 
   } 
   return color.RGBA{0, 0, 0, 255} 
} 
```

`plotData.go`的第四部分包含以下 Go 代码：

```go
func main() { 
   var data []int 
   var f *os.File 
   arguments := os.Args 
   if len(arguments) < 3 { 
         fmt.Printf("%s X Y input\n", filepath.Base(arguments[0])) 
         os.Exit(0) 
   } 

   if len(arguments) == 3 { 
         f = os.Stdin 
   } else { 
         filename := arguments[3] 
         fTemp, err := os.Open(filename) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(0) 
         } 
         f = fTemp 
   } 
   defer f.Close() 

   x, _ = strconv.Atoi(arguments[1]) 
   y, _ = strconv.Atoi(arguments[2]) 
   fmt.Println("Image size:", x, y) 
```

`plotData.go`的第五部分如下：

```go
   scanner := bufio.NewScanner(f) 
   for scanner.Scan() { 
         value, err := strconv.Atoi(scanner.Text()) 
         if err == nil { 
               data = append(data, value) 
         } else { 
               fmt.Println("Error:", value) 
         } 
   } 

   fmt.Println("Slice length:", len(data)) 
   if len(data)*2 > x { 
         fmt.Println("Image size (x) too small!") 
         os.Exit(-1) 
   } 

   maxValue := data[0] 
   for _, temp := range data { 
         if maxValue < temp { 
               maxValue = temp 
         } 
   } 

   if maxValue > y { 
         fmt.Println("Image size (y) too small!") 
         os.Exit(-1) 
   } 
   fmt.Println("maxValue:", maxValue) 
   barHeighPerUnit := int(y / maxValue) 
   fmt.Println("barHeighPerUnit:", barHeighPerUnit) 
```

`plotData.go`的最后一部分如下：

```go
   PNGfile := arguments[1] + "x" + arguments[2] + ".png" 
   OUTPUT, err := os.OpenFile(PNGfile, os.O_CREATE|os.O_WRONLY, 0644) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(-1) 
   } 
   m = image.NewNRGBA(image.Rectangle{Min: image.Point{0, 0}, Max: image.Point{x, y}}) 

   i := 0 
   barWidth = int(x / len(data)) 
   fmt.Println("barWidth:", barWidth) 
   for _, v := range data { 
         c := getColor(v % 10) 
         yy := v * barHeighPerUnit 
         plotBar(barWidth*i, yy, c) 
         fmt.Println("plotBar", barWidth*i, yy) 
         i = i + 1 
   } 

   png.Encode(OUTPUT, m) 
} 
```

虽然您可以单独使用`plotData.go`，但使用`extractData.go`的输出作为`plotData.go`的输入就像执行以下命令一样简单：

```go
$ ./extractData.go 127.0.0.1 access.log{,.1} | awk '{print $2}' | ./plotData 6000 6500
Image size: 6000 6500
Slice length: 2
maxValue: 6333
barHeighPerUnit: 1
barWidth: 3000
plotBar 0 3129
plotBar 3000 6333
$ ls -l 6000x6500.png
-rw-r--r-- 1 mtsouk mtsouk 164915 Jun  5 18:25 6000x6500.png
```

前一个命令的图形输出可以是一个图像，就像您在以下图中看到的那样：

![](img/ee09e9bd-e219-47d1-98f4-47de7bc75848.png)

plotData.go 实用程序生成的输出

# 在 Go 中使用 Unix 套接字

存在两种类型的套接字：Unix 套接字和网络套接字。网络套接字将在第十二章，*网络编程*中解释，而 Unix 套接字将在本节中简要解释。然而，由于所呈现的 Go 函数也适用于 TCP/IP 套接字，因此您仍需等待第十二章，*网络编程*，以充分理解它们，因为它们在这里不会被解释。因此，本节将仅呈现 Unix 套接字客户端的 Go 代码，这是一个使用 Unix 套接字（一种特殊的 Unix 文件）来读取和写入数据的程序。该程序的名称将是`readUNIX.go`，将分为三部分呈现。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "net" 
   "strconv" 
   "time" 
) 
```

`readUNIX.go`的第二部分如下：

```go
func readSocket(r io.Reader) { 
   buf := make([]byte, 1024) 
   for { 
         n, _ := r.Read(buf[:]) 
         fmt.Print("Read: ", string(buf[0:n])) 
   } 
} 
```

最后一部分包含以下 Go 代码：

```go
func main() { 
   c, _ := net.Dial("unix", "/tmp/aSocket.sock") 
   defer c.Close() 

   go readSocket(c) 
   n := 0 
   for { 
         message := []byte("Hi there: " + strconv.Itoa(n) + "\n") 
         _, _ = c.Write(message) 
         time.Sleep(5 * time.Second) 
         n = n + 1 
   } 
} 
```

使用`readUNIX.go`需要另一个进程的存在，该进程也读取和写入同一个套接字文件(`/tmp/aSocket.sock`)。

生成的输出取决于另一部分的实现：在这种情况下，输出如下：

```go
$ go run readUNIX.go
Read: Hi there: 0
Read: Hi there: 1
```

如果找不到套接字文件或没有程序在监听它，您将收到以下错误消息：

```go
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10cfe77]

goroutine 1 [running]:
main.main()
      /Users/mtsouk/Desktop/goBook/ch/ch8/code/readUNIX.go:21 +0x67
exit status 2
```

# Go 中的 RPC

RPC 代表**远程过程调用**，是一种执行对远程服务器的函数调用并在客户端获取答案的方式。再次，您将不得不等到第十二章，*网络编程*，以了解如何在 Go 中开发 RPC 服务器和 RPC 客户端。

# 在 Go 中编程 Unix shell

本节将简要而天真地呈现可以用作 Unix shell 开发基础的 Go 代码。除了`exit`命令外，程序能识别的唯一其他命令是`version`命令，它只是打印程序的版本。所有其他用户输入都将在屏幕上回显。

`UNIXshell.go`的 Go 代码将分为三部分呈现。然而，在此之前，我将向您展示 shell 的第一个版本，其中主要包含注释，以更好地理解我通常如何开始实现一个相对具有挑战性的程序：

```go
package main 

import ( 
   "fmt" 
) 

func main() { 

   // Present prompt 

   // Read a line 

   // Get the first word of the line 

   // If it is a built-in shell command, execute the command 

   // otherwise, echo the command 

} 
```

这更多或多少是我作为起点使用的算法：好处是注释简要地展示了程序的操作方式。请记住，算法不依赖于编程语言。之后，开始实现事物会更容易，因为你知道你想要做什么。

因此，shell 最终版本的第一部分如下：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "os" 
   "strings" 
) 

var VERSION string = "0.2" 
```

第二部分如下：

```go
func main() { 
   scanner := bufio.NewScanner(os.Stdin) 
   fmt.Print("> ") 
   for scanner.Scan() { 

         line := scanner.Text() 
         words := strings.Split(line, " ") 
         command := words[0] 
```

在这里，您只需逐行从用户那里读取输入并找出输入的第一个单词。

`UNIXshell.go`的最后一部分如下：

```go
         switch command { 
         case "exit": 
               fmt.Println("Exiting...") 
               os.Exit(0) 
         case "version": 
               fmt.Println(VERSION) 
         default: 
               fmt.Println(line) 
         } 

         fmt.Print("> ") 
   } 
} 
```

上述的 Go 代码检查用户给出的命令并相应地采取行动。

执行`UNIXshell.go`并与其交互将生成以下输出：

```go
$ go run UNIXshell.go
> version
0.2
> ls -l
ls -l
> exit
Exiting...
```

如果你想了解如何在 Go 中创建自己的 Unix shell，可以访问[`github.com/elves/elvish`](https://github.com/elves/elvish)。

# 另一个小的 Go 更新

在我写这一章时，Go 已经更新：这是一个小更新，主要是修复了一些错误：

```go
$ date
Thu May 25 06:30:53 EEST 2017
$ go version
go version go1.8.3 darwin/amd64
```

# 练习

1.  将`plotIP.go`的绘图功能放入一个 Go 包中，并使用该包重写`plotIP.go`和`plotData.go`。

1.  查看第六章的`ddGo.go` Go 代码，*文件输入和输出*，以便在接收`SIGINFO`信号时打印有关其进度的信息。

1.  更改`cat.go`的 Go 代码以支持多个输入文件。

1.  更改`plotData.go`的代码，以便在生成的图像上打印网格线。

1.  更改`plotData.go`的代码，以便在图表的条之间留出一点空间。

1.  尝试通过为其添加新功能使`UNIXshell.go`程序变得更好一点。

# 摘要

在本章中，我们讨论了许多有趣和方便的主题，包括信号处理和在 Go 中创建图形图像。此外，我们还教会了您如何在 Go 程序中添加对 Unix 管道的支持。

在下一章中，我们将讨论 Go 最独特的特性，即 goroutines。您将学习什么是 goroutine，如何创建和同步它们，以及如何创建通道和管道。请记住，许多人来学习现代和安全的编程语言，但留下来是因为它的 goroutines！
