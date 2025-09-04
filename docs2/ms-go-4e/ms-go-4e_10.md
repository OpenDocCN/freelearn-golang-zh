# 10

# 与 TCP/IP 和 WebSocket 一起工作

TCP/IP 是互联网的基础，因此，在开发网络服务时能够创建 TCP/IP 服务器和客户端是至关重要的。本章教你如何使用 `net` 包来处理 TCP/IP 的底层协议，即 TCP 和 UDP，这样你就可以开发 TCP/IP 服务器和客户端，并对其功能有更多控制。本章包含的 TCP 和 UDP 工具的 Go 代码使我们能够创建自己的高级 TCP/IP 服务，因为 TCP/IP 的核心原则和逻辑保持不变。

此外，本章还介绍了 WebSocket 协议的服务器和客户端的开发，该协议基于 HTTP，并展示了如何与 RabbitMQ 交互，RabbitMQ 是一个开源的 *消息代理*。

WebSocket 协议在单个 TCP 连接上提供全双工通信通道。另一方面，像 RabbitMQ 和 Apache Kafka 这样的消息代理因其速度而闻名，这也是它们被包含在处理大量数据的工作流程中的主要原因。

更详细地说，本章涵盖了：

+   TCP/IP

+   `net` 包

+   开发 TCP 客户端

+   开发 TCP 服务器

+   开发 UDP 客户端

+   开发 UDP 服务器

+   开发并发 TCP 服务器

+   创建 WebSocket 服务器

+   创建 WebSocket 客户端

+   与 RabbitMQ 一起工作

# TCP/IP

TCP/IP 是一组帮助互联网运行的协议。它的名字来源于其最著名的两个协议：TCP 和 IP。

TCP 代表传输控制协议。TCP 软件使用段（也称为 TCP 数据包）在机器之间传输数据。TCP 的主要特点是它是一个 **可靠的协议**，这意味着它确保每个数据包都成功交付，而不需要程序员编写任何额外的代码。如果没有数据包交付的证明，TCP 会重新发送该数据包。除此之外，TCP 数据包可以用来建立连接、传输数据、发送确认和关闭连接。

当两台机器之间建立 TCP 连接时，会在这两台机器之间创建一个全双工虚拟电路，类似于电话通话。这两台机器持续通信以确保数据正确发送和接收。如果连接因某种原因失败，两台机器会尝试找出问题并向相关应用程序报告。每个数据包的 TCP 报头包括源端口和目的端口字段。这两个字段，加上源和目的 IP 地址，组合起来可以唯一地标识每个 TCP 连接。所有这些细节都由 TCP/IP 处理，只要你提供所需细节，无需额外努力。

在创建 TCP/IP 服务器进程时，请记住端口号 0-1024 有受限访问权限，只能由 root 用户使用，这意味着您需要管理员权限才能使用该范围内的任何端口。以 root 权限运行进程是一个安全风险，必须避免。

IP 代表互联网协议。IP 的主要特征是它本质上不是一个可靠的协议。IP 封装了在 TCP/IP 网络上传输的数据，因为它负责根据 IP 地址将数据包从源主机传输到目标主机。IP 必须找到一种寻址方法，以便有效地将数据包发送到其目的地。尽管有专门的设备，称为路由器，执行 IP 路由，但每个 TCP/IP 设备都必须执行一些基本路由。

IP 协议的第一个版本现在被称为 IPv4，以区分最新的 IP 协议版本，称为 IPv6。IPv4 的主要问题是它即将耗尽可用的 IP 地址，这是创建 IPv6 协议的主要原因。这是因为 IPv4 地址仅使用 32 位表示，允许有 2³²（4,294,967,296）个不同的 IP 地址。另一方面，IPv6 使用 128 位来定义其地址中的每一个。IPv4 地址的格式是`10.20.32.245`（由点分隔的四个部分，值从 0 到 255），而 IPv6 地址的格式是`3fce:1706:4523:3:150:f8ff:fe21:56cf`（由冒号分隔的八个部分）。

UDP（用户数据报协议）基于 IP，这意味着它也是不可靠的。UDP 比 TCP 简单，主要是因为 UDP 设计上不可靠。因此，UDP 消息可能会丢失、重复或顺序错误地到达。此外，数据包可能会比接收者处理它们的速度更快地到达。因此，当速度比可靠性更重要时，使用 UDP。

本章实现了 TCP 和 UDP 软件——TCP 和 UDP 服务是互联网的基础。但首先，让我们谈谈方便的`nc(1)`实用程序。

## `nc(1)` 命令行实用程序

当您想要测试 TCP/IP 服务器和客户端时，`nc(1)`实用程序，也称为`netcat(1)`，非常方便：`nc(1)`是一个涉及 TCP 和 UDP 以及 IPv4 和 IPv6 的实用程序，包括但不限于打开 TCP 连接、发送和接收 UDP 消息以及充当 TCP 服务器。

您可以使用`nc(1)`作为运行在具有`10.10.1.123` IP 地址并监听端口号`1234`的机器上的 TCP 服务的客户端，如下所示：

```go
$ nc 10.10.1.123 1234 
```

`-l` 选项告诉 `netcat(1)` 作为服务器运行，这意味着当提供 `-l` 选项时，`netcat(1)` 将在指定的端口号上监听传入的连接。默认情况下，`nc(1)` 使用 TCP 协议。但是，如果您使用 `-u` 标志执行 `nc(1)`，它将使用 UDP 协议，无论是作为客户端还是服务器。最后，`-v` 和 `-vv` 选项告诉 `netcat(1)` 生成详细输出，这在您想要调试网络连接时可能很有用。

# `net` 包

Go 标准库中的 `net` 包主要涉及 TCP/IP、UDP、域名解析和 UNIX 域套接字。`net.Dial()` 函数用于作为客户端连接到网络，而 `net.Listen()` 函数用于指示 Go 程序接受传入的网络连接，从而充当服务器。

`net.Dial()` 和 `net.Listen()` 的返回值都是 `net.Conn` 数据类型，它实现了 `io.Reader` 和 `io.Writer` 接口——这意味着您可以使用与文件 I/O 相关的代码来读取和写入 `net.Conn` 连接。`net.Dial()` 和 `net.Listen()` 的第一个参数是网络类型，但这是它们相似之处结束的地方。

`net.Dial()` 函数用于连接到远程服务器。`net.Dial()` 函数的第一个参数定义了将要使用的网络协议，而第二个参数定义了服务器地址，其中必须包括端口号。第一个参数的有效值包括 `tcp`、`tcp4`（仅 IPv4）、`tcp6`（仅 IPv6）、`udp`、`udp4`（仅 IPv4）、`udp6`（仅 IPv6）、`ip`、`ip4`（仅 IPv4）、`ip6`（仅 IPv6）、`unix`（UNIX 套接字）、`unixgram` 和 `unixpacket`。另一方面，`net.Listen()` 的有效值包括 `tcp`、`tcp4`、`tcp6`、`unix` 和 `unixpacket`。

执行 `go doc net.Listen` 和 `go doc net.Dial` 命令以获取这两个函数的详细信息。

# 开发 TCP 客户端

本节是关于开发 TCP 客户端。接下来的两个小节展示了两种开发 TCP 客户端等效的方法。

## 使用 `net.Dial()` 开发 TCP 客户端

首先，我们将介绍最广泛使用的方法，它在 `tcpC.go` 中实现：

```go
package main
import (
    "bufio"
"fmt"
"net"
"os"
"strings"
) 
```

`import` 块包含 `bufio` 和 `fmt` 等也用于文件 I/O 操作的包。

```go
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Please provide host:port.")
        return
    } 
```

首先，我们读取我们想要连接的 TCP 服务器的详细信息。

```go
 connect := arguments[1]
    c, err := net.Dial("tcp", connect)
    if err != nil {
        fmt.Println(err)
        os.Exit(5)
    } 
```

在连接详细信息的基础上，我们调用 `net.Dial()`——它的第一个参数是我们想要使用的协议，在这个例子中是 `tcp`，而第二个参数包含连接详细信息。成功的 `net.Dial()` 调用返回一个打开的连接（一个 `net.Conn` 接口），这是一个通用的面向流的网络连接。

```go
 reader := bufio.NewReader(os.Stdin)
    for {
        fmt.Print(">> ")
        text, _ := reader.ReadString('\n')
        fmt.Fprintf(c, "%s\n", text)
        message, _ := bufio.NewReader(c).ReadString('\n')
        fmt.Print("->: " + message)
        if strings.TrimSpace(string(text)) == "STOP" {
            fmt.Println("TCP client exiting...")
            return
        }
    }
} 
```

TCP 客户端的最后部分会持续读取用户输入，直到输入`STOP`为止——在这种情况下，客户端在`STOP`后等待服务器响应才终止，因为这是`for`循环构建的方式。这主要是因为服务器可能对我们有一个有用的答案，我们不希望错过。所有用户输入都通过`fmt.Fprintf()`发送（写入）到打开的 TCP 连接，而`bufio.NewReader()`用于从 TCP 连接中读取数据，就像处理常规文件一样。

请记住，不检查`reader.ReadString('\n')`返回的`error`值的原因是简单性。我们永远不应该忽略错误。

使用`tcpC.go`连接到 TCP 服务器，在这个例子中，服务器是用`nc(1)`实现的，命令为`nc -l 1234`，会产生以下类型的输出：

```go
$ go run tcpC.go localhost:1234
>> Hello!
->: Hi from nc -l 1234
>> STOP
->: Bye!
TCP client exiting... 
```

以`>>`开头的行表示用户输入，而以`->`开头的行表示服务器消息。在发送`STOP`后，我们等待服务器响应，然后客户端结束 TCP 连接。前面的代码演示了如何在 Go 中创建一个带有额外逻辑和功能的适当 TCP 客户端（`STOP`关键字）。

下一个小节将展示创建 TCP 客户端的另一种方法。

## 使用`net.DialTCP()`开发 TCP 客户端

本小节介绍了一种开发 TCP 客户端的替代方法。区别在于用于建立 TCP 连接的 Go 函数，即`net.DialTCP()`和`net.ResolveTCPAddr()`，而不是客户端的功能。

`otherTCPclient.go`的代码如下：

```go
package main
import (
    "bufio"
"fmt"
"net"
"os"
"strings"
) 
```

尽管我们正在使用 TCP/IP 连接，但我们仍需要像`bufio`这样的包，因为 UNIX 将网络连接视为文件，所以我们基本上是在网络上进行 I/O 操作。

```go
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Please provide a server:port string!")
        return
    } 
```

我们需要读取我们想要连接的 TCP 服务器的详细信息，包括所需的端口号。在处理 TCP/IP 时，除非我们正在开发一个非常专业的 TCP 客户端，否则实用程序不能使用默认参数操作。

```go
 connect := arguments[1]
    tcpAddr, err := net.ResolveTCPAddr("tcp4", connect)
    if err != nil {
        fmt.Println("ResolveTCPAddr:", err)
        return
    } 
```

`net.ResolveTCPAddr()`函数是针对 TCP 连接的，因此得名，它将给定的地址解析为`*net.TCPAddr`值，这是一个表示 TCP 端点地址的结构——在这种情况下，端点是我们要连接的 TCP 服务器。

```go
 conn, err := net.DialTCP("tcp4", nil, tcpAddr)
    if err != nil {
        fmt.Println("DialTCP:", err)
        return
    } 
```

在手头有 TCP 端点的情况下，我们调用`net.DialTCP()`来连接到服务器。除了使用`net.ResolveTCPAddr()`和`net.DialTCP()`之外，与 TCP 客户端和 TCP 服务器交互相关的其余代码完全相同。

```go
 reader := bufio.NewReader(os.Stdin)
    for {
        fmt.Print(">> ")
        text, _ := reader.ReadString('\n')
        fmt.Fprintf(conn, text+"\n")
        message, _ := bufio.NewReader(conn).ReadString('\n')
        fmt.Print("->: " + message)
        if strings.TrimSpace(string(text)) == "STOP" {
            fmt.Println("TCP client exiting...")
            conn.Close()
            return
        }
    }
} 
```

最后，使用无限`for`循环与 TCP 服务器交互。TCP 客户端读取用户数据，并将其发送到服务器。然后，它从 TCP 服务器读取数据。再次使用`STOP`关键字在客户端侧使用`Close()`方法终止 TCP 连接。

使用`otherTCPclient.go`与 TCP 服务器进程交互会产生以下类型的输出：

```go
$ go run otherTCPclient.go localhost:1234
>> Hello!
->: Hi from nc -l 1234
>> STOP
->: Thanks for connecting!
TCP client exiting... 
```

交互与`tcpC.go`相同——我们只是学会了开发 TCP 客户端的另一种方式。如果我的意见能被采纳，我更喜欢`tcpC.go`中的实现，因为它使用了更通用的函数。然而，这仅仅是个人喜好。

下一个部分将展示如何编程 TCP 服务器。

# 开发 TCP 服务器

本节介绍了两种开发 TCP 服务器的方法，这些服务器可以与 TCP 客户端交互，就像我们与 TCP 客户端交互一样。

## 使用`net.Listen()`开发 TCP 服务器

本节中介绍的 TCP 服务器使用`net.Listen()`，将当前日期和时间作为一个网络数据包发送给客户端。在实践中，这意味着在接收客户端连接后，服务器从操作系统获取时间和日期，并将这些数据发送回客户端。`net.Listen()`函数监听连接，而`net.Accept()`方法等待下一个连接，并返回一个包含客户端信息的通用`net.Conn`变量。`tcpS.go`的代码如下：

```go
package main
import (
    "bufio"
"fmt"
"net"
"os"
"strings"
"time"
)
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Please provide port number")
        return
    } 
```

TCP 服务器应该知道它将要使用的端口号——这作为命令行参数给出。

```go
 PORT := ":" + arguments[1]
    l, err := net.Listen("tcp", PORT)
    if err != nil {
        fmt.Println(err)
        return
    }
    defer l.Close() 
```

`net.Listen()`函数监听连接，这使得该特定程序成为一个服务器进程。如果`net.Listen()`的第二个参数包含一个没有 IP 地址或主机名的端口号，`net.Listen()`将监听本地系统上所有可用的 IP 地址，这里就是这种情况。

这是一个个人偏好；尽管被认为是不良实践，使用如`PORT`和`SERVER`这样的变量名，但这是我自己表示重要或全局变量的方式。

```go
 c, err := l.Accept()
    if err != nil {
        fmt.Println(err)
        return
    } 
```

我们调用`Accept()`并等待客户端连接——`Accept()`会阻塞，直到新的连接到来。这个特定的 TCP 服务器有一些不寻常的地方：它只能服务即将连接到它的第一个 TCP 客户端，因为`Accept()`调用在`for`循环之外，因此**只调用一次**。每个单独的客户端都应该由不同的`Accept()`调用服务，但这里并没有这样做。纠正这一点留给读者作为练习。

```go
 for {
        netData, err := bufio.NewReader(c).ReadString('\n')
        if err != nil {
            fmt.Println(err)
            return
        }
        if strings.TrimSpace(string(netData)) == "STOP" {
            fmt.Println("Exiting TCP server!")
            return
        }
        fmt.Print("-> ", string(netData))
        t := time.Now()
        myTime := t.Format(time.RFC3339) + "\n"
        c.Write([]byte(myTime))
    }
} 
```

这个无休止的`for`循环会一直与同一个 TCP 客户端交互，直到客户端发送`STOP`这个词。就像 TCP 客户端一样，`bufio.NewReader()`用于从网络连接中读取数据，而`Write()`用于向 TCP 客户端发送数据。

运行`tcpS.go`并与 TCP 客户端交互会产生以下类型的输出：

```go
$ go run tcpS.go 1234
-> Hello!
-> Have to leave now!
Exiting TCP server! 
```

服务器连接因客户端连接自动结束，因为当`bufio.NewReader(c).ReadString('\n')`没有更多内容可读时，`for`循环结束了。客户端是`nc(1)`，它产生了以下输出：

```go
$ nc localhost 1234
Hello!
2023-10-09T20:02:55+03:00
Have to leave now!
2023-10-09T20:03:01+03:00
STOP 
```

我们已经使用`STOP`关键字结束了连接。

因此，我们现在知道如何在 Go 中开发 TCP 服务器。与 TCP 客户端一样，还有另一种开发 TCP 服务器的方法，将在下一小节中介绍。

## 使用`net.ListenTCP()`开发 TCP 服务器

这次，这个 TCP 服务器的替代版本实现了回显服务。简单来说，TCP 服务器将接收到的数据发送回客户端。

`otherTCPserver.go`的代码如下：

```go
package main
import (
    "fmt"
"net"
"os"
"strings"
)
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Please provide a port number!")
        return
    }
    SERVER := "localhost" + ":" + arguments[1]
    s, err := net.ResolveTCPAddr("tcp", SERVER)
    if err != nil {
        fmt.Println(err)
        return
    } 
```

之前的代码通过命令行参数获取 TCP 端口号值，该值用于`net.ResolveTCPAddr()`——这是定义 TCP 服务器将要监听的端口号所必需的。

那个函数只与 TCP 一起工作，因此得名。

```go
 l, err := net.ListenTCP("tcp", s)
    if err != nil {
        fmt.Println(err)
        return
    } 
```

同样，`net.ListenTCP()`只与 TCP 一起工作，这使得该程序成为一个准备接受传入连接的 TCP 服务器。

```go
 buffer := make([]byte, 1024)
    conn, err := l.Accept()
    if err != nil {
        fmt.Println(err)
        return
    } 
```

如前所述，由于`Accept()`被调用的位置，这个特定的实现只能与单个客户端一起工作。这是出于简单性的考虑。在本章后面开发的并发 TCP 服务器将`Accept()`调用放在了一个无休止的`for`循环中。

```go
 for {
        n, err := conn.Read(buffer)
        if err != nil {
            fmt.Println(err)
            return
        }
        if strings.TrimSpace(string(buffer[0:n])) == "STOP" {
            fmt.Println("Exiting TCP server!")
            conn.Close()
            return
        } 
```

您需要使用`strings.TrimSpace()`来删除输入中的任何空格字符，并将结果与具有特殊意义的`STOP`关键字进行比较。一旦从客户端接收到`STOP`关键字，服务器将使用`Close()`方法关闭连接。

```go
 fmt.Print("> ", string(buffer[0:n-1]), "\n")
        _, err = conn.Write(buffer)
        if err != nil {
            fmt.Println(err)
            return
        }
    }
} 
```

所有之前的代码都是与 TCP 客户端交互，直到客户端决定关闭连接。运行`otherTCPserver.go`并与 TCP 客户端交互会产生以下类型的输出：

```go
$ go run otherTCPserver.go 1234
> Hello from the client!
Exiting TCP server! 
```

以`>`开头的第一行是客户端消息，而第二行是服务器在接收到客户端的`STOP`消息时的输出。因此，TCP 服务器按照程序处理客户端请求，并在接收到`STOP`消息时退出，这是期望的行为。

下一个部分是关于开发 UDP 客户端。

# 开发 UDP 客户端

这一部分演示了如何开发一个可以与 UDP 服务交互的 UDP 客户端。`udpC.go`的代码如下：

```go
package main
import (
    "bufio"
"fmt"
"net"
"os"
"strings"
)
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Please provide a host:port string")
        return
    }
    CONNECT := arguments[1] 
```

这是我们从用户获取 UDP 服务器详情的方式。

```go
 s, err := net.ResolveUDPAddr("udp4", CONNECT)
    c, err := net.DialUDP("udp4", nil, s) 
```

前两行声明我们正在使用 UDP，并且我们想要连接到由`net.ResolveUDPAddr()`返回值指定的 UDP 服务器。实际的连接是通过`net.DialUDP()`发起的。

```go
 if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("The UDP server is %s\n", c.RemoteAddr().String())
    defer c.Close() 
```

这部分程序通过调用`RemoteAddr()`方法找到 UDP 服务器的详细信息。

```go
 reader := bufio.NewReader(os.Stdin)
    for {

        fmt.Print(">> ")
        text, _ := reader.ReadString('\n')
        data := []byte(text + "\n")
        _, err = c.Write(data) 
```

使用`bufio.NewReader(os.Stdin)`从用户读取数据，并使用`Write()`将数据写入 UDP 服务器。

```go
 if strings.TrimSpace(string(data)) == "STOP" {
            fmt.Println("Exiting UDP client!")
            return
        } 
```

如果从用户读取的输入是`STOP`关键字，那么连接将被终止。

```go
 if err != nil {
            fmt.Println(err)
            return
        }
        buffer := make([]byte, 1024)
        n, _, err := c.ReadFromUDP(buffer) 
```

使用`ReadFromUDP()`方法从 UDP 连接中读取数据。

```go
 if err != nil {
            fmt.Println(err)
            return
        }
        fmt.Printf("Reply: %s\n", string(buffer[0:n]))
    }
} 
```

`for`循环将一直进行，直到接收到作为输入的`STOP`关键字或以其他方式终止程序。

使用`udpC.go`就像以下这样——客户端使用`nc(1)`实现：

```go
$ go run udpC.go localhost:1234
The UDP server is 127.0.0.1:1234 
```

`127.0.0.1:1234`是`c.RemoteAddr().String()`的值，它显示了我们所连接的 UDP 服务器的详细信息。

```go
>> Hello!
Reply: Hi from the server. 
```

我们的客户端向 UDP 服务器发送了`Hello!`，并收到了`Hi from the server.`的回复。

```go
>> Have to leave now :)
Reply: OK - bye from nc -l -u 1234 
```

我们的客户端向 UDP 服务器发送 `Have to leave now :)` 并收到 `OK - bye from nc -l -u 1234` 的回复。UDP 服务器开始使用 `nc -l -u 1234`。

```go
>> STOP
Exiting UDP client! 
```

最后，在向服务器发送 `STOP` 关键字后，客户端打印 `Exiting UDP client!` 并终止——该消息在 Go 代码中定义，可以是任何你想要的内容。

下一节是关于编程 UDP 服务器的内容。

# 开发 UDP 服务器

本节展示了如何开发一个 UDP 服务器，该服务器为客户端生成并返回随机数。UDP 服务器（`udpS.go`）的代码如下：

```go
package main
import (
    "fmt"
"math/rand"
"net"
"os"
"strconv"
"strings"
"time"
)
func random(min, max int) int {
    return rand.Intn(max-min) + min
}
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Please provide a port number!")
        return
    }
    PORT := ":" + arguments[1] 
```

服务器将要监听的 UDP 端口号作为命令行参数提供。

```go
 s, err := net.ResolveUDPAddr("udp4", PORT)
    if err != nil {
        fmt.Println(err)
        return
    } 
```

`net.ResolveUDPAddr()` 函数创建一个 UDP 服务器将要监听的 UDP 端点。

```go
 connection, err := net.ListenUDP("udp4", s)
    if err != nil {
        fmt.Println(err)
        return
    } 
```

`net.ListenUDP("udp4", s)` 函数调用使此过程成为使用其第二个参数指定的详细信息的 `udp4` 协议的服务器。

```go
 defer connection.Close()
    buffer := make([]byte, 1024) 
```

`buffer` 变量存储一个 1024 字节的字节切片，用于从 UDP 客户端读取数据。

```go
 rand.Seed(time.Now().Unix())
    for {
        n, addr, err := connection.ReadFromUDP(buffer)
        fmt.Print("-> ", string(buffer[0:n-1])) 
```

`ReadFromUDP()` 和 `WriteToUDP()` 方法分别用于从 UDP 连接读取数据并将数据写入 UDP 连接。此外，由于 UDP 的操作方式，UDP 服务器可以服务多个客户端。

```go
 if strings.TrimSpace(string(buffer[0:n])) == "STOP" {
            fmt.Println("Exiting UDP server!")
            return
        } 
```

当任何一个客户端发送 `STOP` 消息时，UDP 服务器将终止。除此之外，`for` 循环将永远运行。

```go
 data := []byte(strconv.Itoa(random(1, 1001)))
        fmt.Printf("data: %s\n", string(data)) 
```

字节切片存储在 `data` 变量中，并用于将所需数据写入客户端。

```go
 _, err = connection.WriteToUDP(data, addr)
        if err != nil {
            fmt.Println(err)
            return
        }
    }
} 
```

与 `udpS.go` 一起工作就像以下这样：

```go
$ go run udpS.go 1234
-> Hello from client!
data: 403 
```

以 `->` 开头的行显示来自客户端的数据。以 `data:` 开头的行显示 UDP 服务器生成的随机数——在本例中为 `403`。

```go
-> Going to terminate the connection now.
data: 154 
```

前两行显示了与 UDP 客户端的另一个交互。

```go
-> STOP
Exiting UDP server! 
```

一旦 UDP 服务器从客户端接收到 `STOP` 关键字，它将关闭连接并退出。

在使用 `udpC.go` 的客户端方面，我们有以下交互：

```go
$ go run udpC.go localhost:1234
The UDP server is 127.0.0.1:1234
>> Hello from client!
Reply: 403 
```

客户端向服务器发送 `Hello from client!` 消息并收到 `403`。

```go
>> Going to terminate the connection now.
Reply: 154 
```

客户端向服务器发送 `Going to terminate the connection now.` 并接收随机数 `154`。

```go
>> STOP
Exiting UDP client! 
```

当客户端接收到 `STOP` 作为用户输入时，它将终止 UDP 连接并退出。

下一节展示了如何开发一个使用 goroutines 为其客户端提供服务的并发 TCP 服务器。

# 开发并发 TCP 服务器

本节教你一个开发并发 TCP 服务器的模式，这些服务器在成功调用 `Accept()` 后使用单独的 goroutines 为其客户端提供服务。因此，这样的服务器可以同时为多个 TCP 客户端提供服务。这是现实世界生产服务器和服务实现的方式。

`concTCP.go` 的代码如下：

```go
package main
import (
    "bufio"
"fmt"
"net"
"os"
"strconv"
"strings"
)
var count = 0
func handleConnection(c net.Conn, myCount int) {
    fmt.Print(".") 
```

前面的语句不是必需的——它只是通知我们一个新的客户端已连接。

```go
 netData, err := bufio.NewReader(c).ReadString('\n')
    if err != nil {
        fmt.Println(err)
        return
    }
    for {

        temp := strings.TrimSpace(string(netData))
        if temp == "STOP" {
            break
        }
        fmt.Println(temp)
        counter := "Client number: " + strconv.Itoa(myCount) + "\n"
        c.Write([]byte(string(counter)))
    } 
```

`for` 循环确保 `handleConnection()` 不会自动退出。再次强调，`STOP` 关键字停止了当前客户端连接的 goroutine——然而，服务器进程以及所有其他活跃的客户端连接将继续运行。

```go
 defer c.Close()
} 
```

这是执行为服务客户端的 goroutine 的函数的结束。为了服务一个客户端，你需要一个带有 TCP 客户端详细信息的 `net.Conn` 参数。在读取客户端数据后，服务器会向当前的 TCP 客户端发送一条消息，指示到目前为止已服务的 TCP 客户端总数。

```go
func main() {
    arguments := os.Args
    if len(arguments) == 1 {
        fmt.Println("Please provide a port number!")

        os.Exit(5)
    }
    PORT := ":" + arguments[1]
    l, err := net.Listen("tcp4", PORT)
    if err != nil {
        fmt.Println(err)
        return
    }
    defer l.Close()
    for {
        c, err := l.Accept()
        if err != nil {
            fmt.Println(err)
            return
        }
        go handleConnection(c, count)
        count++
    }
} 
```

每次有新的客户端连接到服务器时，`count` 变量会增加。每个 TCP 客户端都由一个执行 `handleConnection()` 函数的单独的 goroutine 服务。这释放了服务器进程，并允许它接受新的连接。简单来说，当多个 TCP 客户端正在被服务时，TCP 服务器可以自由地与更多的 TCP 客户端交互。和之前一样，新的 TCP 客户端是通过使用 `Accept()` 函数连接的。

使用 `concTCP.go` 进行工作会产生以下类型的输出：

```go
$ go run concTCP.go 1234
.Hello
.Hi from  nc localhost 1234 
```

输出的第一行来自第一个 TCP 客户端，而第二行来自第二个 TCP 客户端。这意味着并发 TCP 服务器按预期工作。因此，当你想在 TCP 服务中服务多个 TCP 客户端时，你可以使用提供的技巧和代码作为开发自己的并发 TCP 服务器的模板。

下面的部分将涉及 WebSocket 协议。

# 创建 WebSocket 服务器

WebSocket 协议是一种计算机通信协议，它通过单个 TCP 连接提供全双工（同时双向传输数据）通信通道。WebSocket 协议在 RFC 6455 ([`tools.ietf.org/html/rfc6455`](https://tools.ietf.org/html/rfc6455)) 中定义，并分别使用 `ws://` 和 `wss://` 替代 `http://` 和 `https://`。因此，客户端应通过使用以 `ws://` 开头的 URL 来开始 WebSocket 连接。

在本节中，我们将使用 `gorilla/websocket` ([`github.com/gorilla/websocket`](https://github.com/gorilla/websocket)) 模块开发一个小型但功能齐全的 WebSocket 服务器。该服务器实现了回声服务，这意味着它会自动将客户端输入返回给客户端。

`https://pkg.go.dev/golang.org/x/net/websocket` 包提供了另一种开发 WebSocket 客户端和服务器的方法。然而，根据其文档，`pkg.go.dev/golang.org/x/net/websocket` 缺少一些功能，建议使用 [`pkg.go.dev/github.com/gorilla/websocket`](https://pkg.go.dev/github.com/gorilla/websocket)，即这里使用的，或者 [`pkg.go.dev/nhooyr.io/websocket`](https://pkg.go.dev/nhooyr.io/websocket)。

你可能会问，为什么使用 WebSocket 协议而不是 HTTP。WebSocket 协议的优点包括以下内容：

+   WebSocket 连接是一个全双工、双向的通信通道。这意味着服务器不需要等待从客户端读取数据才能向客户端发送数据，反之亦然。

+   WebSocket 连接是原始的 TCP 套接字，这意味着它们不需要建立 HTTP 连接所需的开销。

+   WebSocket 连接也可以用来发送 HTTP 数据。然而，普通的 HTTP 连接不能作为 WebSocket 连接工作。

+   WebSocket 连接在它们被终止之前一直存在，因此没有必要总是重新打开它们。

+   WebSocket 连接可以用于实时 Web 应用程序。

+   数据可以在任何时候从服务器发送到客户端，即使客户端没有请求。

+   WebSocket 是 HTML5 规范的一部分，这意味着它被所有现代网络浏览器支持。

在展示服务器实现之前，了解`gorilla/websocket`包中的`websocket.Upgrader`方法将 HTTP 服务器连接升级到 WebSocket 协议，并允许你定义升级参数会很好。之后，你的 HTTP 连接就变成了 WebSocket 连接，这意味着你将不允许执行与 HTTP 协议相关的语句。

下一个小节将展示服务器的实现。

## 服务器实现

本小节展示了实现 echo 服务的 WebSocket 服务器，这在测试网络连接时非常有用。

代码被放置在`~/go/src/github.com/mactsouk/mGo4th/ch10/ws`目录中。`server`目录包含服务器的实现，而`client`目录包含 WebSocket 客户端的实现。

WebSocket 服务器的实现可以在`server.go`中找到：

```go
package main
import (
    "fmt"
"log"
"net/http"
"os"
"time"
"github.com/gorilla/websocket"
) 
```

这是用于处理 WebSocket 协议的外部包。

```go
var PORT = ":1234"
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        return true
    },
} 
```

这里定义了`websocket.Upgrader`的参数。它们将很快被使用。

```go
func rootHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome!\n")
    fmt.Fprintf(w, "Please use /ws for WebSocket!")
} 
```

这是一个常规的 HTTP 处理函数。

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    log.Println("Connection from:", r.Host)
    ws, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("upgrader.Upgrade:", err)
        return
    }
    defer ws.Close() 
```

WebSocket 服务器应用程序调用`Upgrader.Upgrade`方法从 HTTP 请求处理器获取 WebSocket 连接。在成功调用`Upgrader.Upgrade`之后，服务器开始与 WebSocket 连接和 WebSocket 客户端一起工作。

```go
 for {
        mt, message, err := ws.ReadMessage()
        if err != nil {
            log.Println("From", r.Host, "read", err)
            break
        }
        log.Print("Received: ", string(message))
        err = ws.WriteMessage(mt, message)
        if err != nil {
            log.Println("WriteMessage:", err)
            break
        }
    }
} 
```

`wsHandler()`中的`for`循环处理所有针对`/ws`的传入消息——你可以使用任何你想要的技巧来处理传入请求。此外，在所展示的实现中，除非有网络问题或服务器进程被终止，否则只有客户端被允许关闭现有的 WebSocket 连接。

最后，请记住，在 WebSocket 连接中，您不能使用`fmt.Fprintf()`语句向 WebSocket 客户端发送数据——如果您使用这些中的任何一个，或者任何其他可以执行相同功能的调用，WebSocket 连接将失败，您将无法发送或接收任何数据。因此，在用`gorilla/websocket`实现的 WebSocket 连接中发送和接收数据的唯一方法是分别通过`WriteMessage()`和`ReadMessage()`调用。当然，您总是可以通过处理原始网络数据来实现所需的功能，但这超出了本书的范围。

```go
func main() {
    arguments := os.Args
    if len(arguments) != 1 {
        PORT = ":" + arguments[1]
    } 
```

如果没有命令行参数，将使用存储在`PORT`全局变量中的默认端口号。否则，将使用给定的值。

```go
 mux := http.NewServeMux()
    s := &http.Server{
        Addr:         PORT,
        Handler:      mux,
        IdleTimeout:  10 * time.Second,
        ReadTimeout:  time.Second,
        WriteTimeout: time.Second,
    } 
```

这些是处理 WebSocket 连接的 HTTP 服务器的详细信息。

```go
 mux.Handle("/", http.HandlerFunc(rootHandler))
    mux.Handle("/ws", http.HandlerFunc(wsHandler)) 
```

用于 WebSocket 的端点可以是您想要的任何内容——在这种情况下，它是`/ws`。此外，您可以有多个端点，它们使用 WebSocket 协议。

```go
 log.Println("Listening to TCP Port", PORT)
    err := s.ListenAndServe()
    if err != nil {
        log.Println(err)
        return
    }
} 
```

所展示的代码使用`log.Println()`而不是`fmt.Println()`来打印消息——因为这是一个服务器进程，使用`log.Println()`比`fmt.Println()`更好，因为日志信息被发送到可以在以后检查的文件。然而，在开发过程中，您可能更喜欢`fmt.Println()`调用，并避免写入日志文件，因为您可以在屏幕上立即看到数据，而无需在其他地方查找。此外，如果您打算以 Docker 镜像运行服务器，使用`fmt.Println()`更有意义。但是，您应该记住，`log`包默认也会打印到屏幕上。

服务器实现简短，但功能齐全。代码中最重要的单个调用是`Upgrader.Upgrade`，因为这会将 HTTP 连接升级为 WebSocket 连接。

从 GitHub 获取并运行代码需要以下步骤——大多数步骤都与模块初始化和下载所需的包有关：

```go
$ go mod init
$ go mod tidy
$ go run server.go 
```

要测试该服务器，我们需要有一个客户端。由于我们迄今为止还没有开发自己的客户端，我们将使用`websocat`实用程序来测试 WebSocket 服务器。

## 使用 websocat

`websocat`是一个命令行实用程序，可以帮助您测试 WebSocket 连接。然而，由于`websocat`默认未安装，您需要使用您选择的包管理器在您的机器上安装它。如果目标地址有 WebSocket 服务器，您可以使用以下方式使用它：

```go
$ websocat ws://localhost:1234/ws
Hello from websocat! 
```

这是我们在服务器上键入并发送的内容。

```go
Hello from websocat! 
```

这是我们从 WebSocket 服务器返回的内容，该服务器实现了回显服务——不同的 WebSocket 服务器实现不同的功能。

```go
Bye! 
```

再次强调，上一行是用户输入给`websocat`的。

```go
Bye! 
```

最后一条是服务器发送回的数据。连接是通过在`websocat`客户端上按*Ctrl* + *D*来关闭的。

如果你希望从`websocat`获取详细输出，你可以使用`-v`标志执行它：

```go
$ websocat -v ws://localhost:1234/ws
[INFO  websocat::lints] Auto-inserting the line mode
[INFO  websocat::stdio_threaded_peer] get_stdio_peer (threaded)
[INFO  websocat::ws_client_peer] get_ws_client_peer
[INFO  websocat::ws_client_peer] Connected to ws
Hello from websocat!
Hello from websocat!
Bye!
Bye!
[INFO  websocat::sessionserve] Forward finished
[INFO  websocat::ws_peer] Received WebSocket close message
[INFO  websocat::sessionserve] Reverse finished
[INFO  websocat::sessionserve] Both directions finished 
```

在这两种情况下，我们的 WebSocket 服务器的输出应该类似于以下内容：

```go
$ go run server.go
2023/10/09 20:29:16 Listening to TCP Port :1234
2023/10/09 20:29:24 Connection from: localhost:1234
2023/10/09 20:29:31 Received: Hello from websocat!
2023/10/09 20:29:53 Received: Bye!
2023/10/09 20:30:01 From localhost:1234 read websocket: close 1005 (no status) 
```

下一个子节展示了如何在 Go 中开发 WebSocket 客户端。

# 创建 WebSocket 客户端

本节展示了如何在 Go 中编程 WebSocket 客户端。客户端读取用户数据，将其发送到服务器，并读取服务器响应。`client`目录包含 WebSocket 客户端的实现。`gorilla/websocket`包将帮助我们开发 WebSocket 客户端。

`./client/client.go`的代码如下：

```go
package main
import (
    "bufio"
"fmt"
"log"
"net/url"
"os"
"os/signal"
"syscall"
"time"
"github.com/gorilla/websocket"
)
var (
    SERVER       = ""
    PATH         = ""
    TIMESWAIT    = 0
    TIMESWAITMAX = 5
    in           = bufio.NewReader(os.Stdin)
) 
```

`in`变量只是`bufio.NewReader(os.Stdin)`的一个快捷方式。

```go
func getInput(input chan string) {
    result, err := in.ReadString('\n')
    if err != nil {
        log.Println(err)
        return
    }
    input <- result
} 
```

`getInput()`函数作为 goroutine 执行，获取用户输入并将其通过`input`通道传输到`main()`函数。每次程序读取一些用户输入时，旧的 goroutine 结束，并开始一个新的`getInput()` goroutine 以获取新的用户输入。

```go
func main() {
    arguments := os.Args
    if len(arguments) != 3 {
        fmt.Println("Need SERVER + PATH!")
        return
    }
    SERVER = arguments[1]
    PATH = arguments[2]
    fmt.Println("Connecting to:", SERVER, "at", PATH)
    interrupt := make(chan os.Signal, 1)
    signal.Notify(interrupt, os.Interrupt) 
```

WebSocket 客户端借助`interrupt`通道处理 UNIX 中断。当捕获到适当的信号（`syscall.SIGINT`）时，使用`websocket.CloseMessage`消息关闭与服务器的 WebSocket 连接。这正是专业工具的工作方式！

```go
 input := make(chan string, 1)
    go getInput(input)
    URL := url.URL{Scheme: "ws", Host: SERVER, Path: PATH}
    c, _, err := websocket.DefaultDialer.Dial(URL.String(), nil)
    if err != nil {
        log.Println("Error:", err)
        return
    }
    defer c.Close() 
```

WebSocket 连接从调用`websocket.DefaultDialer.Dial()`开始。所有发送到`input`通道的内容都使用`WriteMessage()`方法传输到 WebSocket 服务器。

```go
 done := make(chan struct{})
    go func() {
        defer close(done)
        for {
            _, message, err := c.ReadMessage()
            if err != nil {
                log.Println("ReadMessage() error:", err)
                return
            }
            log.Printf("Received: %s", message)
        }
    }() 
```

另一个 goroutine，这次使用匿名 Go 函数实现，负责使用`ReadMessage()`方法从 WebSocket 连接中读取数据。

```go
 for {
        select {
        case <-time.After(4 * time.Second):
            log.Println("Please give me input!", TIMESWAIT)
            TIMESWAIT++
            if TIMESWAIT > TIMESWAITMAX {
                syscall.Kill(syscall.Getpid(), syscall.SIGINT)
            } 
```

`syscall.Kill(syscall.Getpid(), syscall.SIGINT)`语句使用 Go 代码向程序发送中断信号。根据`client.go`的逻辑，中断信号使得程序关闭与服务器的 WebSocket 连接并终止其执行。这仅在当前超时周期数大于预定义的全局值时发生，在这个例子中等于`5`。

```go
 case <-done:
            return
case t := <-input:
            err := c.WriteMessage(websocket.TextMessage, []byte(t))
            if err != nil {
                log.Println("Write error:", err)
                return
            }
            TIMESWAIT = 0 
```

如果你收到用户输入，当前的超时周期数（`TIMESWAIT`）将被重置，并读取新的输入。

```go
 go getInput(input)
        case <-interrupt:
            log.Println("Caught interrupt signal - quitting!")
            err := c.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, "")) 
```

在我们关闭客户端连接之前，我们向服务器发送`websocket.CloseMessage`以正确结束连接。

```go
 if err != nil {
                log.Println("Write close error:", err)
                return
            }
            select {
            case <-done:
            case <-time.After(2 * time.Second):
            }
            return
        }
    }
} 
```

由于`./client/client.go`位于单独的目录中，我们需要运行以下命令来收集所需的依赖项并运行它：

```go
$ cd client
$ go mod init
$ go mod tidy 
```

与 WebSocket 服务器的交互产生以下类型的输出：

```go
$ go run client.go localhost:1234 ws
Connecting to: localhost:1234 at ws
Hello there!
2023/10/09 20:36:25 Received: Hello there! 
```

前两行显示了用户输入以及服务器响应。

```go
2023/10/09 20:36:29 Please give me input! 0
2023/10/09 20:36:33 Please give me input! 1
2023/10/09 20:36:37 Please give me input! 2
2023/10/09 20:36:41 Please give me input! 3
2023/10/09 20:36:45 Please give me input! 4
2023/10/09 20:36:49 Please give me input! 5
2023/10/09 20:36:49 Caught interrupt signal - quitting!
2023/10/09 20:36:49 ReadMessage() error: websocket: close 1000 (normal) 
```

输出的最后几行显示了自动超时过程的工作方式。

WebSocket 服务器为之前的交互生成了以下输出：

```go
2023/10/09 20:36:22 Connection from: localhost:1234
2023/10/09 20:36:25 Received: Hello there!
2023/10/09 20:36:49 From localhost:1234 read websocket: close 1000 (normal) 
```

然而，如果无法在提供的地址找到 WebSocket 服务器，WebSocket 客户端将产生以下输出：

```go
$ go run client.go localhost:1234 ws
Connecting to: localhost:1234 at ws
2023/10/09 08:11:20 Error: dial tcp [::1]:1234: connect: connection refused 
```

`connection refused`消息表明没有进程在`localhost`上监听端口号`1234`。

本章的下一节是关于与 RabbitMQ 消息代理一起工作。

# 与 RabbitMQ 一起工作

在本章的最后部分，我们将学习如何与 RabbitMQ 一起工作。RabbitMQ 是一个开源的消息代理，当您想要异步交换信息并需要一个安全存储消息的地方时特别有用。RabbitMQ 使您能够交换信息，这也是您不需要直接使用它除非您想执行管理任务的主要原因。

RabbitMQ 使用 AMQP 协议。AMQP 代表高级消息队列协议，是一种面向消息的中间件的开源协议。AMQP 的特点是消息导向、排队、路由、可靠性和安全性。AMQP 与二进制数据一起工作，并以帧的形式传输数据。根据您要执行的任务，存在九种类型的帧（打开连接、关闭连接、传输数据等）。

如果您必须在 RabbitMQ 和 Kafka 之间做出选择，它们都执行类似的工作，您应该首先考虑它们之间的差异。首先，Kafka 比 RabbitMQ 更快。其次，RabbitMQ 使用推送模型，而 Kafka 使用基于拉取的方法。第三，Kafka 支持批处理，而 RabbitMQ 不支持。最后，RabbitMQ 没有对有效负载大小的限制，而 Kafka 对有效负载大小有一些限制。要记住的关键点是，如果速度是您的主要关注点，那么 Kafka 可能是一个更好的选择，而如果您主要关注有效负载大小和简单性，那么 RabbitMQ 是一个更合理的选择。

数据存储在队列中。RabbitMQ 中的队列是一个 FIFO（先进先出）结构，支持两种操作：添加元素和获取元素。队列有名称，这意味着您应该知道您想要交互的队列的名称，无论是作为消息生产者还是消息消费者。

`github.com/rabbitmq/amqp091-go` Go 模块通过 AMQP 协议连接到 RabbitMQ，前提是您提供了正确的连接细节。

本节的所有文件，包括源文件和用于运行 RabbitMQ 的`docker-compose.yml`文件，都位于`~/go/src/github.com/mactsouk/mGo4th/ch10/MQ`。

## 运行 RabbitMQ

在本节的第一小节中，我们将学习如何使用 Docker 来执行 RabbitMQ——这是在您的机器上运行 RabbitMQ 的最干净解决方案，因为它只需要使用单个 Docker 镜像。位于`~/go/src/github.com/mactsouk/mGo4th/ch10/MQ/`目录下的`docker-compose.yml`文件的内容如下：

```go
version: "3.6"
services:
rabbitmq:
image: 'rabbitmq:3.12-management'
container_name: rabbit
ports:
- '5672:5672'
- '15672:15672'
environment:
AMQP_URL: 'amqp://rabbitmq?connection_attempts=5&retry_delay=5'
RABBITMQ_DEFAULT_USER: "guest"
RABBITMQ_DEFAULT_PASS: "guest"
networks:
- rabbit
networks:
rabbit:
driver: bridge 
```

运行`docker-compose.yml`就像在`~/go/src/github.com/mactsouk/mGo4th/ch10/MQ/`目录下使用`docker-compose up`一样简单。如果相关的 Docker 镜像不在活动机器上，它将首先被下载。

现在我们已经启动了 RabbitMQ，让我们学习如何向 RabbitMQ（生产者）发送数据，并从 RabbitMQ 队列（消费者）读取数据。

## 向 RabbitMQ 写入

RabbitMQ 生产者的代码命名为`sendMQ.go`，位于`producer`目录下。`sendMQ.go`源文件也分为两部分。第一部分包含以下代码：

```go
package main
import (
    "fmt"
    amqp "github.com/rabbitmq/amqp091-go"
)
func main() {
    fmt.Println("RabbitMQ producer")
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        fmt.Println("amqp.Dial():", err)
        return
    }
    ch, err := conn.Channel()
    if err != nil {
        fmt.Println(err)
        return
    }
    defer ch.Close() 
```

`amqp.Dial()`调用初始化与 RabbitMQ 的连接，而`Channel()`调用打开该连接的通道，使其准备好使用。

连接字符串（`amqp://guest:guest@localhost:5672/`）可以分为三个逻辑部分。第一部分（`amqp://`）是使用的协议，第二部分包含用户凭据，第三部分包含服务器名称和端口号（`localhost:5672/`）。

第二部分包含以下代码：

```go
 q, err := ch.QueueDeclare("Go", false, false, false, false, nil)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("Queue:", q)
    message := "Writing to RabbitMQ!"
    err = ch.PublishWithContext(nil, "", "Go", false, false,
        amqp.Publishing{ContentType: "text/plain", Body: []byte(message)},
    )
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println("Message published to Queue!")
} 
```

`QueueDeclare()`定义了我们将要使用的队列。如果`QueueDeclare()`指定的队列名称不存在，它将被创建。这意味着队列名称中的错误不会被捕获。在这种情况下，队列被命名为`Go`。

我们想要发送的纯文本数据保存在`message`变量中。之后，我们指定将要放入`amqp.Publishing{}`结构体中的数据格式，在这种情况下是纯文本（也支持 JSON 格式）。`message`变量的内容被转换为字节切片，并放入匿名`amqp.Publishing{}`结构体的`Body`字段中。

那个`amqp.Publishing{}`结构体是`ch.PublishWithContext()`调用的一个参数，它将结构体发送到 RabbitMQ。

当前版本的`sendMQ.go`只是使用`Publish()`将消息写入预定义的队列并退出。这意味着为了向 RabbitMQ 发送多个消息，我们必须多次执行`sendMQ.go`。

运行`sendMQ.go`生成以下类型的输出：

```go
$ go run sendMQ.go
RabbitMQ producer
Queue: {Go 0 0}
Message published to Queue!
$ go run sendMQ.go
RabbitMQ producer
Queue: {Go 1 0}
Message published to Queue! 
```

`{Go 1 0}`输出意味着我们目前在 RabbitMQ 队列中有两条消息。

下一个子节将展示如何从 RabbitMQ 队列中消费消息。

## 从 RabbitMQ 读取

RabbitMQ 消费者的代码命名为`readMQ.go`，位于`consumer`目录下。`readMQ.go`源文件也分为两部分。第一部分包含以下代码：

```go
package main
import (
    "fmt"
    amqp "github.com/rabbitmq/amqp091-go"
)
func main() {
    fmt.Println("RabbitMQ consumer")
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        fmt.Println("Failed Initializing Broker Connection")
        panic(err)
    }
    ch, err := conn.Channel()
    if err != nil {
        fmt.Println(err)
    }
    defer ch.Close() 
```

与`sendMQ.go`类似，我们定义连接细节，并打开连接。

第二部分如下：

```go
 msgs, err := ch.Consume("Go", "", true, false, false, false, nil)
    if err != nil {
        fmt.Println(err)
    }
    forever := make(chan bool)
    go func() {
        for d := range msgs {
            fmt.Printf("Received: %s\n", d.Body)
        }
    }()
    fmt.Println("Connected to the RabbitMQ server!")
    <-forever
} 
```

`Consume()`方法使用 goroutine 从`Go`队列读取消息。`forever`通道阻塞程序，防止其退出。接收到的消息的`Body`字段，它是一个结构体，包含我们想要的数据。

运行`readMQ.go`，在`sendMQ.go`向 RabbitMQ 中放入一些消息后，生成以下类型的输出：

```go
$ go run readMQ.go
RabbitMQ consumer
Connected to the RabbitMQ server!
Received: Writing to RabbitMQ!
Received: Writing to RabbitMQ! 
```

`readMQ.go`实用程序持续运行并等待新消息，这意味着我们需要自己通过按下*Ctrl* + *C*来结束它。

## 如何删除模块

这与 RabbitMQ 没有直接关系，但它是一个有用的提示。`consumer`目录中当前版本的`go.mod`如下所示：

```go
module github.com/mactsouk/mGo4th/ch10/MQ/consumer
go 1.21.1
require github.com/rabbitmq/amqp091-go v1.8.1 
```

您可以使用以下方式使用`none`从`go.mod`中删除模块：

```go
$ go get github.com/rabbitmq/amqp091-go@none
go: removed github.com/rabbitmq/amqp091-go v1.8.1 
```

之后，`go.mod`将如下所示：

```go
module github.com/mactsouk/mGo4th/ch10/MQ/consumer
go 1.21.1 
```

如果您想将`go.mod`恢复到之前的状态，可以运行`go mod tidy`。

# 摘要

本章主要介绍了`net`包、TCP/IP、TCP 和 UDP，它们实现了低级连接，以及 WebSocket 和 RabbitMQ。

WebSocket 为我们提供了创建服务的另一种方式。一般来说，当我们想要交换大量数据，并且希望连接始终保持开启状态，进行全双工数据交换时，WebSocket 是更好的选择。然而，如果我们不确定该选择什么，建议从 TCP/IP 服务开始，看看效果如何，然后再升级到 WebSocket 协议。

最后，当我们要将大量数据存储和检索到外部数据存储时，RabbitMQ 是一个合理的选择。

Go 可以帮助您创建各种并发服务器和客户端。

现在我们已经准备好开始开发自己的服务了！下一章将介绍 REST API、通过 HTTP 交换 JSON 数据以及开发 RESTful 客户端和服务器——Go 在开发 RESTful 客户端和服务器方面得到了广泛应用。

# 练习

+   开发一个并发 TCP 服务器，在预定义的范围内生成随机数。

+   开发一个并发 TCP 服务器，在 TCP 客户端给出的范围内生成随机数。这可以用作从集合中随机选择值的方法。

+   将 UNIX 信号处理添加到本章开发的并发 TCP 服务器中，以便在接收到指定信号时优雅地停止服务器进程。

+   编写一个客户端程序，从 RabbitMQ 服务器读取消息并将其发布到 TCP 服务器。

+   开发一个 WebSocket 服务器，该服务器创建一个可变数量的随机整数并发送给客户端。随机整数的数量由客户端在初始客户端消息中指定。

# 其他资源

+   WebSocket 协议：[`tools.ietf.org/rfc/rfc6455.txt`](https://tools.ietf.org/rfc/rfc6455.txt)

+   维基百科 WebSocket：[`en.wikipedia.org/wiki/WebSocket`](https://en.wikipedia.org/wiki/WebSocket)

+   Gorilla WebSocket 包：[`github.com/gorilla/websocket`](https://github.com/gorilla/websocket)

+   Gorilla WebSocket 文档：[`www.gorillatoolkit.org/pkg/websocket`](https://www.gorillatoolkit.org/pkg/websocket)

+   RabbitMQ Go 模块：[`github.com/rabbitmq/amqp091-go`](https://github.com/rabbitmq/amqp091-go)

+   在[`www.amqp.org/`](https://www.amqp.org/)和[`en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol`](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)了解更多关于 AMQP 的信息。

# 留下您的评价！

喜欢这本书？通过留下亚马逊评价帮助像您这样的读者。扫描下面的二维码以获得您选择的免费电子书。

![评价二维码](img/Review_QR_Code.png)
