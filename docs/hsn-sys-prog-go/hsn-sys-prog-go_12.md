# 第九章：网络编程

本章将涵盖网络编程。这将使我们的应用程序能够与在任何远程计算机上运行的其他程序通信，或者在同一本地网络上，甚至在互联网上。

我们将从网络和体系结构的一些理论开始。然后，我们将讨论套接字级通信，并解释如何创建 Web 服务器。最后，我们将讨论 Go 内置模板引擎的工作原理。

本章将涵盖以下主题：

+   网络

+   套接字编程

+   Web 服务器

+   模板引擎

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，您可以参考第三章，*Go 概述*。

此外，它需要在您的计算机上安装 OpenSSL。许多 Linux 发行版已经附带了一些 OpenSSL 版本。它也可以在 Windows 上安装，使用官方安装程序或第三方软件包管理器，如 Chocolatey 或 Scoop。

# 通过网络通信

即使应用程序位于同一台机器上，应用程序之间也可以通过网络进行通信。为了传输信息，它们需要建立一个共同的协议，该协议规定了从应用程序到传输介质的所有过程。

# OSI 模型

**开放系统互联**（**OSI**）模型是一个理论模型，可以追溯到 20 世纪 70 年代初。它定义了一种通信标准，无论网络的物理或技术结构如何，都可以提供不同网络的互操作性。

该模型定义了七个不同的层，从一到七编号，每一层的抽象级别都比前一层更高。前三层通常被称为**媒体层**，而后四层则是**主机层**。让我们在以下各节中逐一检查每一层。

# 第 1 层-物理层

OSI 模型的第一层是物理层，负责从设备传输未经处理的数据，类似于以太网端口，以及传输介质，如以太网电缆。该层定义了与连接的物理/材料性质相关的所有特征-连接器的大小、形状、电压、频率和时序。

物理层定义的另一个方面是传输的方向，可以是以下之一：

+   **单工**：通信是单向的。

+   **半双工**：通信是双向的，但通信只能单向进行。

+   **全双工**：双向通信，两端可以同时通信。

许多知名技术，包括蓝牙和以太网，都包括它们正在使用的物理层的定义。

# 第 2 层-数据链路层

下一层是数据链路层，它定义了两个直接连接的节点之间的数据传输应该如何进行。它负责以下内容：

+   检测第一层的通信错误

+   纠正物理错误

+   控制节点之间的流/传输速率

+   连接终止

数据链路层定义的一些现实世界示例是以太网（802.3）和 Wi-Fi（802.11）。

# 第 3 层-网络层

网络层是下一层，它专注于称为数据包的数据序列，可以具有可变长度。数据包从一个节点传输到另一个节点，这两个节点可以位于同一网络上，也可以位于不同的网络上。

该层将网络定义为一系列连接到相同介质的节点，由前两层标识。网络能够传递消息，只知道其目的地地址。

# 第 4 层-传输层

第四层是传输层，确保数据包从发送方到接收方。这是通过目的地发送**确认**（**ACK**）和**否认确认**（**NACK**）消息来实现的，这些消息可以触发消息的重复，直到它们被正确接收。还有其他机制在起作用，例如将消息分割成块进行传输（分段），将部分重新组装成单个消息（去分段），并检查数据是否成功发送和接收（错误控制）。

OSI 模型规定了五种不同的传输协议 - TP0、TP1、TP2、TP3 和 TP4。TP0 是最简单的，只执行消息的分段和重组。其他类别在其基础上添加其他功能，例如重传或超时。

# 第五层 - 会话层

第五层引入了会话的概念，这是两台计算机之间临时交互信息的交换。它负责创建连接和终止连接（同时跟踪会话），并允许检查点和恢复。

# 第六层 - 表示层

倒数第二层是表示层，负责处理应用程序之间的语法和语义，通过处理复杂的数据表示。它允许最后一层独立于用于表示数据的编码。OSI 模型的表示使用 ASN.1 编码，但有许多不同的表示协议被广泛使用，例如 XML 和 JSON。

# 第七层 - 应用层

最后一层，应用层，是直接与应用程序通信的层。应用程序不被视为 OSI 模型的一部分，该层负责定义应用程序使用的接口。它包括 FTP、DNS 和 SMTP 等协议。

# TCP/IP - 互联网协议套件

**传输控制协议/互联网协议**（**TCP/IP**），或者互联网协议套件，是由比 OSI 模型更少层次组成的模型，被广泛采用。

# 第一层 - 链路层

第一层是链路层，是 OSI 的物理和数据链路的组合，它定义了本地网络通信的方式，指定了协议，例如 MAC（包括以太网和 Wi-Fi）。

# 第二层 - 互联网层

互联网层是第二层，可以与 OSI 的网络进行比较。它定义了一个通用接口，允许不同的网络在不了解彼此的底层拓扑的情况下有效地进行通信。该层负责局域网（LAN）中节点之间的通信，以及构成互联网的全球互联网络之间的通信。

# 第三层 - 传输层

第三层类似于第四层 OSI。它处理两个设备的端到端通信，还负责错误检查和恢复，使上层不了解数据的复杂性。它定义了两个主要协议 - TCP，通过使用确认系统允许接收方按正确顺序获取数据，以及用户数据协议（UDP），不对接收方应用错误控制或确认。

# 第四层 - 应用层

最后一层，应用层，总结了 OSI 的最后三个级别 - 会话、表示和应用。该层定义了应用程序使用的体系结构，例如点对点或客户端和服务器，以及应用程序使用的协议，例如 SSH、HTTP 或 SMTP。每个进程都是一个具有虚拟通信端点的地址，称为*端口*。

# 理解套接字编程

Go 标准库允许我们轻松地与传输层进行交互，使用 TCP 和 UDP 连接。在本节中，我们将看看如何使用套接字公开服务，以及如何在另一个应用程序中查找并使用它。

# 网络包

创建和处理 TCP 连接所需的工具位于`net`包内。该包的主要接口是`Conn`，表示一个连接。

它有四种实现：

+   `IPConn`：使用 IP 协议的原始连接，TCP 和 UDP 连接都是基于它构建的

+   `TCPConn`：使用 TCP 协议的 IP 连接

+   `UDPConn`：使用 UDP 协议的 IP 连接

+   `UnixConn`：Unix 域套接字，连接用于同一台机器上的进程

在接下来的章节中，我们将看看如何不同地使用 TCP 和 UDP，以及如何使用 IPConn 来实现通信协议的自定义实现。

# TCP 连接

TCP 是互联网上最常用的协议，它能够传递有序的数据（字节）。该协议的主要重点是可靠性，通过建立双向通信来实现，接收方在成功接收数据报时发送确认信号。

可以使用`net.Dial`函数创建新连接。这是一个通用函数，可以接受不同的网络，例如以下内容：

+   `tcp`，`tcp4`（仅限 IPv4），`tcp6`（仅限 IPv6）

+   `udp`，`udp4`（仅限 IPv4），`udp6`（仅限 IPv6）

+   `ip`，`ip4`（仅限 IPv4），`ip6`（仅限 IPv6）

+   `unix`（套接字流），`unixgram`（套接字数据报），和`unixpacket`（套接字数据包）

可以创建 TCP 连接，指定`tcp`协议，以及主机和端口：

```go
conn, err := net.Dial("tcp", "localhost:8080")
```

创建连接的更直接的方法是`net.DialTCP`，它允许您指定本地和远程地址。使用它需要创建一个`net.TCPAddr`：

```go
addr, err := net.ResolveTCPAddr("tcp", "localhost:8080")
if err != nil {
    // handle error
}
conn, err := net.DialTCP("tcp", nil, addr)
if err != nil {
    // handle error
}
```

为了接收和处理连接，还有另一个接口`net.Listener`，它有四种不同的实现方式，每种连接类型一个。对于连接，有一个通用的`net.Listen`函数和一个特定的`net.ListenTCP`函数。

我们可以尝试构建一个简单的应用程序，创建一个 TCP 监听器并连接到它，发送来自标准输入的任何内容。该应用程序应该创建一个监听器来启动后台连接，将标准输入发送到连接，然后接受并处理它。我们将使用换行符作为消息的分隔符。如下面的代码所示：

```go
func main() {
    if len(os.Args) != 2 {
        log.Fatalln("Please specify an address.")
    }
    addr, err := net.ResolveTCPAddr("tcp", os.Args[1])
    if err != nil {
        log.Fatalln("Invalid address:", os.Args[1], err)
    }
    listener, err := net.ListenTCP("tcp", addr)
    if err != nil {
        log.Fatalln("Listener:", os.Args[1], err)
    }
    log.Println("<- Listening on", addr)

    go createConn(addr)

    conn, err := listener.AcceptTCP()
    if err != nil {
        log.Fatalln("<- Accept:", os.Args[1], err)
    }
    handleConn(conn)
}
```

连接创建非常简单。它创建连接并从标准输入读取消息，并通过写入将其转发到连接：

```go
func createConn(addr *net.TCPAddr) {
    defer log.Println("-> Closing")
    conn, err := net.DialTCP("tcp", nil, addr)
    if err != nil {
        log.Fatalln("-> Connection:", err)
    }
    log.Println("-> Connection to", addr)
    r := bufio.NewReader(os.Stdin)
    for {
        fmt.Print("# ")
        msg, err := r.ReadBytes('\n')
        if err != nil {
            log.Println("-> Message error:", err)
        }
        if _, err := conn.Write(msg); err != nil {
            log.Println("-> Connection:", err)
            return
        }
    }
}
```

在我们的用例中，发送数据的连接将通过特殊消息`\q`关闭，这将被解释为一个命令。在监听器中接受连接会创建另一个连接，代表由拨号操作获得的连接。监听器创建的连接将接收来自拨号连接的消息并相应地执行。它将解释特殊消息，如`\q`，并执行特定操作；否则，它将只是在屏幕上打印消息，如下面的代码所示：

```go
func handleConn(conn net.Conn) {
    r := bufio.NewReader(conn)
    time.Sleep(time.Second / 2)
    for {
        msg, err := r.ReadString('\n')
        if err != nil {
            log.Println("<- Message error:", err)
            continue
        }
        switch msg = strings.TrimSpace(msg); msg {
        case `\q`:
            log.Println("Exiting...")
            if err := conn.Close(); err != nil {
                log.Println("<- Close:", err)
            }
            time.Sleep(time.Second / 2)
            return
        case `\x`:
            log.Println("<- Special message `\\x` received!")
        default:
            log.Println("<- Message Received:", msg)
        }
    }
}
```

下面的代码示例在一个应用程序中创建了客户端和服务器，但它可以很容易地分成两个应用程序——一个服务器（能够同时处理多个连接）和一个客户端，创建到服务器的单个连接。服务器将具有一个`Accept`循环，处理单独的 goroutine 上接收的连接。`handleConn`函数与我们之前定义的相同：

```go
func main() {
    if len(os.Args) != 2 {
        log.Fatalln("Please specify an address.")
    }
    addr, err := net.ResolveTCPAddr("tcp", os.Args[1])
    if err != nil {
        log.Fatalln("Invalid address:", os.Args[1], err)
    }
    listener, err := net.ListenTCP("tcp", addr)
    if err != nil {
        log.Fatalln("Listener:", os.Args[1], err)
    }
    for {
        time.Sleep(time.Millisecond * 100)
        conn, err := listener.AcceptTCP()
        if err != nil {
            log.Fatalln("<- Accept:", os.Args[1], err)
        }
        go handleConn(conn)
    }
}
```

客户端将创建连接并发送消息。`createConn`将与我们之前定义的相同：

```go
func main() {
    if len(os.Args) != 2 {
        log.Fatalln("Please specify an address.")
    }
    addr, err := net.ResolveTCPAddr("tcp", os.Args[1])
    if err != nil {
        log.Fatalln("Invalid address:", os.Args[1], err)
    }
    createConn(addr)
}
```

在分离的客户端和服务器中，可以测试当客户端或服务器关闭连接时会发生什么。

# UDP 连接

UDP 是另一种在互联网上广泛使用的协议。它专注于低延迟，这就是为什么它不像 TCP 那样可靠。它有许多应用，从在线游戏到媒体流媒体，再到互联网语音协议（VoIP）。在 UDP 中，如果一个数据包没有收到，它就会丢失，并且不会像在 TCP 中那样再次发送。想象一下 VoIP 通话，如果有连接问题，你将会丢失部分对话，但当你恢复时，你几乎可以实时地继续通信。对于这种类型的应用程序使用 TCP 可能会导致每个数据包丢失都会积累延迟，使得对话变得不可能。

在下面的示例中，我们将创建一个客户端和一个服务器应用程序。服务器将是一种回声，将从客户端接收到的消息发送回去，但它还将颠倒消息内容。

客户端将与 TCP 的客户端非常相似，但也有一些例外——它将使用`net.ResolveUDPAddr`函数来获取地址，并使用`net.DialUDP`来获取连接：

```go
func main() {
    if len(os.Args) != 2 {
        log.Fatalln("Please specify an address.")
    }
    addr, err := net.ResolveUDPAddr("udp", os.Args[1])
    if err != nil {
        log.Fatalln("Invalid address:", os.Args[1], err)
    }
    conn, err := net.DialUDP("udp", nil, addr)
    if err != nil {
        log.Fatalln("-> Connection:", err)
    }
    log.Println("-> Connection to", addr)
    r := bufio.NewReader(os.Stdin)
    b := make([]byte, 1024)
    for {
        fmt.Print("# ")
        msg, err := r.ReadBytes('\n')
        if err != nil {
            log.Println("-> Message error:", err)
        }
        if _, err := conn.Write(msg); err != nil {
            log.Println("-> Connection:", err)
            return
        }
        n, err := conn.Read(b)
        if err != nil {
            log.Println("<- Receive error:", err)
        }
        msg = bytes.TrimSpace(b[:n])
        log.Printf("<- %q", msg)
    }
}
```

服务器将与 TCP 的服务器非常不同。主要区别在于，使用 TCP 时，我们有一个监听器来接受不同的连接，这些连接是分开处理的；与此同时，UDP 监听器是一个连接。它可以盲目地接收数据，或者使用`ReceiveFrom`方法，该方法还将返回接收者的地址。这可以在`WriteTo`方法中使用来进行回答，如下面的代码所示：

```go
func main() {
    if len(os.Args) != 2 {
        log.Fatalln("Please specify an address.")
    }
    addr, err := net.ResolveUDPAddr("udp", os.Args[1])
    if err != nil {
        log.Fatalln("Invalid address:", os.Args[1], err)
    }
    conn, err := net.ListenUDP("udp", addr)
    if err != nil {
        log.Fatalln("Listener:", os.Args[1], err)
    }

    b := make([]byte, 1024)
    for {
        n, addr, err := conn.ReadFromUDP(b)
        if err != nil {
            log.Println("<-", addr, "Message error:", err)
            continue
        }
        msg := bytes.TrimSpace(b[:n])
        log.Printf("<- %q from %s", msg, addr)
        for i, l := 0, len(msg); i < l/2; i++ {
            msg[i], msg[l-1-i] = msg[l-1-i], msg[i]
        }
        msg = append(msg, '\n')
        if _, err := conn.WriteTo(b[:n], addr); err != nil {
            log.Println("->", addr, "Send error:", err)
        }
    }
}
```

# 编码和校验和

在客户端和服务器之间设置某种形式的编码是一个很好的做法，如果编码包括校验和以验证数据完整性，那就更好了。我们可以改进上一节的示例，使用既进行编码又进行校验和的自定义协议。让我们从定义编码函数开始，给定消息将返回以下字节序列：

| **函数** | **字节序列** |
| --- | --- |
| 前四个字节将遵循一个序列 | `2A 00 2A 00` |
| 两个字节将以小端序（最低有效字节在前）存储消息长度 | `08 00` |
| 四个字节用于数据校验和 | `00 00 00 00` |
| 紧随原始消息 | `0F 1D 3A FF ...` |
| 以相同的起始序列结尾 | `2A 00 2A 00` |

`Checksum`函数将通过对消息内容进行求和来计算，使用五个字节的小端序（最低有效字节在前），逐个添加任何剩余的字节，然后将求和的前四个字节作为小端序：

```go
func Checksum(b []byte) []byte {
    var sum uint64
    for len(b) >= 5 {
        for i := range b[:5] {
            v := uint64(b[i])
            for j := 0; j < i; j++ {
                v = v * 256
            }
            sum += v
        }
        b = b[5:]
    }
    for _, v := range b {
        sum += uint64(v)
    }
    s := make([]byte, 8)
    binary.LittleEndian.PutUint64(s, sum)
    return s[:4]
}
```

现在，让我们创建一个函数，用来使用我们定义的协议封装消息：

```go
var ErrLength = errors.New("message too long")

func CreateMessage(content []byte) ([]byte, error) {
    if len(content) > 65535 {
        return nil, ErrLength
    }
    data := make([]byte, 0, len(content)+14)
    data = append(data, Sequence...)
    data = append(data, byte(len(content)/256), byte(len(content)%256))
    data = append(data, Checksum(content)...)
    data = append(data, content...)
    data = append(data, Sequence...)
    return data, nil
}
```

我们还需要另一个函数，用来检查消息是否有效并提取其内容：

```go
func MessageContent(b []byte) ([]byte, error) {
    n := len(b)
    if n < 14 {
        return nil, fmt.Errorf("Too short")
    }
    if open := b[:4]; !bytes.Equal(open, Sequence) {
        return nil, fmt.Errorf("Wrong opening sequence %x", open)
    }
    if length := int(b[4])*256 + int(b[5]); n-14 != length {
        return nil, fmt.Errorf("Wrong length: %d (expected %d)", length, n-14)
    }
    if close := b[n-4 : n]; !bytes.Equal(close, Sequence) {
        return nil, fmt.Errorf("Wrong closing sequence %x", close)
    }
    content := b[10 : n-4]
    if !bytes.Equal(Checksum(content), b[6:10]) {
        return nil, fmt.Errorf("Wrong checksum")
    }
    return content, nil
}
```

现在我们可以用它们来对消息进行编码和解码。例如，我们可以改进上一节中的 UDP 客户端和服务器，并在发送时进行编码：

```go
// Send
data, err := common.CreateMessage(msg)
if err != nil {
    log.Println("->", addr, "Encode error:", err)
    continue
}
if _, err := conn.WriteTo(data, addr); err != nil {
    log.Println("->", addr, "Send error:", err)
}
```

我们还可以解码接收到的字节以提取内容：

```go
//Receive
n, addr, err := conn.ReadFromUDP(b)
if err != nil {
    log.Println("<-", addr, "Message error:", err)
    continue
}
msg, err := common.MessageContent(b[:n])
if err != nil {
    log.Println("<-", addr, "Decode error:", err)
    continue
}
log.Printf("<- %q from %s", msg, addr)
```

为了验证我们收到的内容是否有效，我们使用了之前定义的`MessageContent`实用程序函数。这将检查头部、长度和校验和。它只会提取组成消息的字节。

# Go 中的 Web 服务器

Go 语言最大和最成功的应用之一是创建 Web 服务器。在本节中，我们将看到 Web 服务器实际上是什么，HTTP 协议是如何工作的，以及如何使用标准库和第三方包来实现 Web 服务器应用程序。

# Web 服务器

Web 服务器应用程序是一种可以使用 HTTP 协议（以及一些其他相关协议）在 TCP/IP 网络上提供内容的软件。有许多知名的 Web 服务器应用程序，如 Apache、NGINX 和 Microsoft IIS。常见的服务器使用情况包括以下几种：

+   **提供静态文件，如网站和相关资源**：HTML 页面、图像、样式表和脚本。

+   **暴露 Web 应用程序**：在服务器上运行的具有基于 HTML 的界面的应用程序，需要浏览器才能访问。

+   **暴露 Web API**：不是由用户而是由其他应用程序使用的远程接口。有关更多详细信息，请参阅第一章，*系统编程简介*。

# HTTP 协议

HTTP 协议是 Web 服务器的基石。它的设计始于 1989 年。HTTP 的主要用途是请求和响应范式，其中客户端发送请求，服务器返回响应给客户端。

**统一资源定位符**（**URL**）是 HTTP 请求的唯一标识符，其结构如下：

| **部分** | **示例** |
| --- | --- |
| 协议 | `http` |
| `://` | `://` |
| 主机 | `www.website.com` |
| 路径 | `/path/to/some-resource` |
| `?` | `?` |
| 查询（可选） | `query=string&with=values` |

从上表中，我们可以得出以下结论：

+   除了 HTTP 及其加密版本（HTTPS）之外，还有几种不同的协议，如**文件传输协议**（**FTP**）及其安全对应协议，**SSH 文件传输协议**（**SFTP**）。

+   主机可以是实际 IP 或主机名。当选择主机名时，还有另一个参与者，即**域名服务器**（**DNS**），它充当主机名和物理地址之间的电话簿。DNS 将主机名转换为 IP。

+   路径是服务器中所需的资源，它总是绝对的。

+   查询字符串是在问号后面添加到路径中的内容。它是一系列以`key=value`形式的键值对，它们由`&`符号分隔。

HTTP 是一种文本协议，它包含 URL 的一些元素和其他信息，如方法、标题和正文。

请求正文是发送到服务器的信息，如表单值或上传的文件。

标题是相对于请求的元数据，每行一个，以`Key: Value; extra data`形式。有一系列定义的具有特定功能的标题，如`Authorization`，`User-Agent`和`Content-Type`。

一些方法表示对资源执行的操作。这些是最常用的方法：

+   `GET`：所选资源的表示

+   `HEAD`：类似于`GET`，但没有任何响应体

+   `POST`：向服务器提交资源，通常是新资源

+   `PUT`：提交资源的新版本

+   `DELETE`：删除资源

+   `PATCH`：请求对资源进行特定更改

这是 HTTP 请求的样子：

```go
POST /resource/ HTTP/1.1
User-Agent: Mozilla/4.0 (compatible; MSIE5.01; Windows NT)
Host: www.website.com
Content-Length: 1024
Accept-Language: en-us

the actual request content
that is optional
```

从上述代码中，我们可以看到以下内容：

+   第一行是空格分隔的三元组：方法—路径—协议。

+   每个标题后面都跟着一行。

+   一个空行作为分隔符。

+   可选的请求体。

对于每个请求，都有一个响应，其结构与 HTTP 请求非常相似。唯一不同的部分是包含不同空格分隔的三元组的第一行：HTTP 版本—状态码—原因。

状态码是代表请求结果的整数。有四个主要的状态类别：

+   `100`：已接收信息/请求，并将进行进一步处理

+   `200`：成功的请求；例如，`OK 200`或`Created 201`

+   `300`：重定向到另一个 URL，临时或永久

+   `400`：客户端错误，如`Not Found 404`或`Conflict 409`

+   `500`：服务器端错误，如`Internal Server Error 503`

这是 HTTP 响应的样子：

```go
HTTP/1.1 200 OK
Content-Length: 88
Content-Type: text/html

<html>
  <body>
    <h1>Sample Page</h1>
  </body>
</html>
```

# HTTP/2 和 Go

最常用的 HTTP 版本是 HTTP/1.1，日期为 1997 年。2009 年，Google 启动了一个新项目，创建了一个更快的 HTTP/1.1 后继者，名为 SPDY。该协议最终成为现在的**超文本传输协议**的 2.0 版本，**HTTP/2**。

它是以现有 Web 应用程序的工作方式构建的，但对于使用新协议的应用程序，包括更快的通信速度，有新功能。一些不同之处包括以下内容：

+   它是二进制的（HTTP/1.1 是文本的）。

+   它是完全多路复用的，并且可以使用一个 TCP 连接并行请求数据。

+   它使用头部压缩来减少开销。

+   服务器可以向客户端推送响应，而不是被客户端周期性地询问。

+   它具有更快的协议协商——感谢**应用层协议协商**（**ALPN**）扩展。

所有主要的现代浏览器都支持 HTTP/2。Go 1.6 版本包含了对 HTTP/2 的透明支持，1.8 版本引入了服务器向客户端推送响应的能力。

# 使用标准包

现在我们将看到如何在 Go 中使用标准包创建一个 Web 服务器。一切都包含在`net/http`包中，该包公开了一系列用于发出 HTTP 请求和创建 HTTP 服务器的函数。

# 发出 HTTP 请求

该包公开了一个`http.Client`类型，可用于发出请求。如果请求是简单的`GET`或`POST`，则有专用方法。该包还提供了一个同名的函数，但它只是`DefaultClient`实例的相应方法的简写。检查以下代码：

```go
resp, err := http.Get("http://example.com/")
resp, err := client.Get("http://example.com/")
...
resp, err := http.Post("http://example.com/upload", "image/jpeg", &buf)
resp, err := client.Post("http://example.com/upload", "image/jpeg", &buf)
...
values := url.Values{"key": {"Value"}, "id": {"123"}}
resp, err := http.PostForm("http://example.com/form", values)
resp, err := client.PostForm("http://example.com/form", values)
```

对于任何其他类型的需求，`Do`方法允许我们执行特定的`http.Request`。`NewRequest`函数允许我们指定任何`io.Reader`：

```go
req, err := http.NewRequest("GET", "http://example.com", nil)
// ...
req.Header.Add("Content-Type", "text/html")
resp, err := client.Do(req)
// ...
```

`http.Client`有几个字段，其中许多是允许我们使用默认实现或自定义实现的接口。第一个是`CookieJar`，它允许客户端存储和重用 Web cookies。Cookie 是浏览器发送给客户端的数据，客户端可以发送回服务器以替换头部，例如身份验证。默认客户端不使用 cookie jar。另一个接口是`RoundTripper`，它只有一个方法`RoundTrip`，它获取一个请求并返回一个响应。如果未指定值，则使用`DeafultTransport`值，也可以用于组成`RoundTripper`的自定义实现。客户端返回的`http.Response`也有一个 body，它是`io.ReadCloser`，其关闭由应用程序负责。这就是为什么建议在获得响应后立即使用延迟的`Close`语句。在下面的示例中，我们将实现一个自定义传输，该传输记录请求的 URL 并在执行标准往返之前修改一个头部：

```go
type logTripper struct {
    http.RoundTripper
}

func (l logTripper) RoundTrip(r *http.Request) (*http.Response,  
    error) {
        log.Println(r.URL)
        r.Header.Set("X-Log-Time", time.Now().String())
        return l.RoundTripper.RoundTrip(r)
}
```

我们将在一个客户端中使用这个传输来发出一个简单的请求：

```go
func main() {
    client := http.Client{Transport: logTripper{http.DefaultTransport}}
    req, err := http.NewRequest("GET", "https://www.google.com/search?q=golang+net+http", nil)
    if err != nil {
        log.Fatal(err)
    }
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()
    log.Println("Status code:", resp.StatusCode)
}
```

# 创建一个简单的服务器

该包提供的另一个功能是服务器创建。该包的主要接口是`Handle`，它有一个方法`ServeHTTP`，使用请求来写入响应。它的最简单的实现是`HandlerFunc`，它是一个具有`ServeHTTP`相同签名的函数，并通过执行自身来实现`Handler`。

`ListenAndServe`函数使用给定的地址和处理程序启动 HTTP 服务器。如果未指定处理程序，则使用`DefaultServeMux`变量。`ServeMux`是一种特殊类型的`Handler`，它管理对不同处理程序的执行，具体取决于所请求的 URL 路径。它有两种方法，`Handle`和`HandleFunc`，允许用户指定路径和相应的处理程序。该包还提供了类似于我们为`Client`所见的通用处理程序函数，它们将调用默认`ServerMux`的同名方法。

在下面的示例中，我们将创建一个`customHandler`并创建一个带有一些端点的简单服务器，包括自定义端点：

```go
type customHandler int

func (c *customHandler) ServeHTTP(w http.ResponseWriter, r  
    *http.Request) {
        fmt.Fprintf(w, "%d", *c)
        *c++
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/hello", func(w http.ResponseWriter, r 
        *http.Request) {
            fmt.Fprintf(w, "Hello!")
    })
    mux.HandleFunc("/bye", func(w http.ResponseWriter, r 
        *http.Request) {
            fmt.Fprintf(w, "Goodbye!")
    })
    mux.HandleFunc("/error", func(w http.ResponseWriter, r 
        *http.Request) {
            w.WriteHeader(http.StatusInternalServerError)
            fmt.Fprintf(w, "An error occurred!")
    })
    mux.Handle("/custom", new(customHandler))
    if err := http.ListenAndServe(":3000", mux); err != nil {
        log.Fatal(err)
    }
}
```

# 提供文件系统

Go 标准包允许我们轻松地在文件系统中为特定目录提供服务，使用`net.FileServer`函数，当给定`net.FileSystem`接口时，返回一个用于提供该目录的`Handler`。默认实现是`net.Dir`，它是一个表示系统中目录的自定义字符串。`FileServer`函数已经有了一个保护机制，防止我们使用相对路径（如`../../../dir`）访问提供服务的目录之外的目录。

以下是一个使用提供的目录作为文件服务的根目录的示例文件服务器：

```go
func main() {
    if len(os.Args) != 2 {
        log.Fatalln("Please specify a directory")
    }
    s, err := os.Stat(os.Args[1])
    if err == nil && !s.IsDir() {
        err = errors.New("not a directory")
    }
    if err != nil {
        log.Fatalln("Invalid path:", err)
    }
    http.Handle("/", http.FileServer(http.Dir(os.Args[1])))
    if err := http.ListenAndServe(":3000", nil); err != nil {
        log.Fatal(err)
    }
}
```

# 通过路由和方法导航

使用的 HTTP 方法存储在`Request.Method`字段中。这个字段可以在处理程序内部使用，以便为每种支持的方法设置不同的行为：

```go
switch r.Method {
case http.MethodGet:
    // GET implementation
case http.MethodPost:
    // POST implementation
default:
    http.NotFound(w, r)
}
```

`http.Handler`接口的优势在于我们可以定义自定义类型。这可以使代码更易读，并且可以概括这种特定于方法的行为：

```go
type methodHandler map[string]http.Handler

func (m methodHandler) ServeHTTP(w http.ResponseWriter, r 
        *http.Request) {
            h, ok := m[strings.ToUpper(r.Method)]
            if !ok {
                http.NotFound(w, r)
                return
            }
    h.ServeHTTP(w, r)
}
```

这将使代码更易读，并且可以重复用于不同的路径：

```go
func main() {
    http.HandleFunc("/path1", methodHandler{
        http.MethodGet: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            fmt.Fprint(w, "Showing record")
        }),
        http.MethodPost: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            fmt.Fprint(w, "Updated record")
        }),
    })
    if err := http.ListenAndServe(":3000", nil); err != nil {
        log.Fatal(err)
    }
}
```

# 多部分请求和文件

请求体是一个`io.ReadCloser`。这意味着关闭它是服务器的责任。对于文件上传，请求体不是文件的内容，而是通常是一个多部分请求，它在头部指定一个边界，并在体内使用它来将消息分成部分。

这是一个示例多部分消息：

```go
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary=xxxx

This part before boundary is ignored
--xxxx
Content-Type: text/plain

First part of the message. The next part is binary data encoded in base64
--xxxx
Content-Type: application/octet-stream
Content-Transfer-Encoding: base64

PGh0bWw+CiAgPGhlYWQ+CiAgPC9oZWFkPgogIDxib2R5PgogICAgPHA+VGhpcyBpcyB0aGUg
Ym9keSBvZiB0aGUgbWVzc2FnZS48L3A+CiAgPC9ib2R5Pgo8L2h0bWw+Cg==
--xxxx--
```

我们可以看到边界有两个破折号作为前缀，后面跟着一个换行符，最终边界也有两个破折号作为后缀。在下面的示例中，服务器将处理文件上传，使用一个小表单从浏览器发送请求。

让我们定义一些在处理程序中将使用的常量：

```go
const (
    param = "file"
    endpoint = "/upload"
    content = `<html><body>` +
        `<form enctype="multipart/form-data" action="%s" method="POST">` +
        `<input type="file" name="%s"/><input type="submit" 
    value="Upload"/>` +
        `</form></html></body>`
)
```

现在，我们可以定义处理程序函数。第一部分应该在方法为`GET`时显示模板，因为它在`POST`上执行上传，并在其他情况下返回未找到状态：

```go
mux.HandleFunc(endpoint, func(w http.ResponseWriter, r 
    *http.Request) {
        if r.Method == "GET" {
            fmt.Fprintf(w, content, endpoint, param)
            return
        } else if r.Method != "POST" {
            http.NotFound(w, r)
            return
        }

    path, err := upload(r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    fmt.Fprintf(w, "Uploaded to %s", path)
})
```

`upload`函数将使用`Request.FormFile`方法返回文件及其元数据：

```go
func upload(r *http.Request) (string, error) {
    f, h, err := r.FormFile(param)
    if err != nil {
        return "", err
    }
    defer f.Close()

    p := filepath.Join(os.TempDir(), h.Filename)
    fw, err := os.OpenFile(p, os.O_WRONLY|os.O_CREATE, 0666)
    if err != nil {
        return "", err
    }
    defer fw.Close()

    if _, err = io.Copy(fw, f); err != nil {
        return "", err
    }
    return p, nil
}
```

# HTTPS

如果您希望您的 Web 服务器使用 HTTPS 而不是依赖外部应用程序（如 NGINX），如果您已经有有效的证书，您可以很容易地这样做。如果没有，您可以使用 OpenSSL 创建一个：

```go
> openssl genrsa -out server.key 2048

> openssl req -new -x509 -sha256 -key server.key -out server.crt -days 3650
```

第一条命令生成私钥，而第二条命令创建了一个服务器所需的公共证书。第二条命令还需要大量的额外信息来创建证书，从国家名称到电子邮件地址。

一切准备就绪后，为了创建一个 HTTPS 服务器，需要用其安全对应物`http.ListenAndServeTLS`替换`http.ListenAndServe`函数：

```go
func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r 
        *http.Request) {
            fmt.Fprint(w, "Hello!")
    })
    err := http.ListenAndServeTLS(":443", "server.crt", "server.key", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

# 第三方包

Go 开源社区开发了许多与`net/http`集成的包，实现了`Handler`接口，但提供了一组独特的功能，可以更轻松地开发 Web 服务器。

# gorilla/mux

`github.com/gorilla/mux`包包含了`Handler`的另一个实现，增强了标准`ServeMux`的功能：

+   更好的 URL 匹配到处理程序，使用 URL 中的任何元素，包括模式、方法、主机或查询值。

+   URL 元素，如主机、路径和查询键可以有占位符（也可以使用正则表达式）。

+   通过子路由，可以分层定义路由，测试路径的一部分。

+   处理程序也可以用作中间件，在主处理程序之前，用于所有路径或一部分路径。

让我们从使用其他路径元素进行匹配的示例开始：

```go
r := mux.NewRouter()
// only local requests
r.Host("localhost:3000")
// only when some header is present
r.Headers("X-Requested-With", "XMLHttpRequest")
only when a query parameter is specified
r.Queries("access_key", "0x20")
```

变量是另一个非常有用的功能，它允许我们指定占位符，并使用辅助函数`mux.Vars`获取它们的值，如下例所示：

```go
r := mux.NewRouter()
r.HandleFunc("/products/", ProductsHandler)
r.HandleFunc("/products/{key}/", ProductHandler)
r.HandleFunc("/products/{key}/details", ProductDetailsHandler)
...
// inside an handler
vars := mux.Vars(request)
key:= vars["key"]
```

`Subrouter`是另一个有用的函数，用于对相同前缀路由进行分组。这使我们可以将上一个代码简化为以下代码片段：

```go
r := mux.NewRouter()
s := r.PathPrefix("/products").Subrouter()
s.HandleFunc("/", ProductsHandler)
s.HandleFunc("/{key}/", ProductHandler)
s.HandleFunc("/{key}/details", ProductDetailsHandler)
```

当与子路由结合使用时，中间件也非常有用，可以执行一些常见任务，如身份验证和验证：

```go
r := mux.NewRouter()
pub := r.PathPrefix("/public").Subrouter()
pub.HandleFunc("/login", LoginHandler)
priv := r.PathPrefix("/private").Subrouter()
priv.Use(AuthHandler)
priv.HandleFunc("/profile", ProfileHandler)
priv.HandleFunc("/logout", LogoutHandler)
```

# gin-gonic/gin

`github.com/gin-gonic/gin`包是另一个 Go Web 框架，它通过许多简写和辅助函数扩展了 Go HTTP 服务器的功能。其特点包括以下内容：

+   **速度**：它的路由速度快，内存占用很小。

+   **中间件**：它允许定义和使用中间处理程序，并完全控制它们的流程。

+   **无 Panic**：它带有从 panic 中恢复的中间件。

+   **分组**：它可以将具有相同前缀的路由分组在一起。

+   **错误**：它管理和收集请求期间发生的错误。

+   渲染：它默认带有大多数 Web 格式的渲染器（JSON、XML、HTML）。

该包的核心是`gin.Engine`，也是一个`http.Handler`。`gin.Default`函数返回一个使用两个中间件的引擎——`Logger`，它打印每个接收到的 HTTP 请求的结果，以及`Recovery`，它从 panic 中恢复。另一个选项是使用`gin.New`函数，返回一个没有中间件的引擎。

它允许我们将处理程序绑定到单个 HTTP 方法，使用一系列引擎方法，这些方法以它们的 HTTP 对应物命名：

+   `DELETE`

+   `GET`

+   `HEAD`

+   `OPTIONS`

+   `PATCH`

+   `POST`

+   `PUT`

+   `Any`（适用于任何 HTTP 方法）

还有一个`group`方法，返回一个选定路径的路由分组，公开了所有前面的方法：

```go
router := gin.Default()

router.GET("/resource", getResource)
router.POST("/resource", createResource)
router.PUT("/resource", updateResoure)
router.DELETE("/resource", deleteResource)
// with use grouping
g := router.Group("/resource")
g.GET("", getResource)
g.POST("", createResource)
g.PUT("", updateResoure)
g.DELETE("", deleteResource)
```

该框架中的处理程序具有不同的签名。它不是使用响应写入器和请求作为参数，而是使用`gin.Context`，这是一个包装了两者的结构，并提供了许多简写和实用工具。例如，该包提供了在 URL 中使用占位符的可能性，而上下文使这些参数可以被读取：

```go
router := gin.Default()
router.GET("/hello/:name", func(c *gin.Context) {
    c.String(http.StatusOK, "Hello %s!", c.Param("name"))
})
```

我们还可以在示例中看到，上下文提供了一个`String`方法，使我们能够用一行代码编写 HTTP 状态和响应内容。

# 其他功能

Web 服务器还有其他功能。其中一些已经受到标准库的支持（如 HTTP/2 推送器），其他功能则可通过实验性包或第三方库获得（如 WebSockets）。

# HTTP/2 推送器

我们已经讨论过，自 Go 1.8 版本以来，Golang 支持 HTTP/2 服务器端推送功能。让我们看看如何在应用程序中使用它。它的使用非常简单；如果请求可以转换为`http.Pusher`接口，它可以用于在主接口中推送额外的请求。在这个例子中，我们用它来并行加载 SVG 图像，以及页面：

```go
func main() {
    const imgPath = "/image.svg"
    http.HandleFunc("/", func(w http.ResponseWriter, r 
        *http.Request) {
            pusher, ok := w.(http.Pusher)
            if ok {
                fmt.Println("Push /image")
                pusher.Push(imgPath, nil)
            }
        w.Header().Add("Content-Type", "text/html")
        fmt.Fprintf(w, `<html><body><img src="img/%s"/>`+
            `</body></html>`, imgPath)
    })
    http.HandleFunc(imgPath, func(w http.ResponseWriter, r 
        *http.Request) {
            w.Header().Add("Content-Type", "image/svg+xml")
            fmt.Fprint(w, `<?xml version="1.0" standalone="no"?>
<svg >
  <rect width="150" height="150" style="fill:blue"/>
</svg>`)
    })
    if err := http.ListenAndServe(":3000", nil); err != nil {
        fmt.Println(err)
    }
}
```

这将导致 HTTP/1 的两个单独请求，以及 HTTP/2 的一个单一请求，其中第二个请求是使用浏览器的推送功能获得的。

# WebSockets 协议

HTTP 协议只实现单向通信，而 WebSocket 协议是客户端和服务器之间的全双工通信。Go 实验性库通过`golang.org/x/net/websocket`包提供了对 WebSocket 的支持，Gorilla 还有另一个实现，使用了自己的`github.com/gorilla/websocket`。

第二个更加完整，它在`github.com/olahol/melody`包中使用，该包实现了一个简单的 WebSocket 通信框架。每个包都提供了 WebSocket 服务器和客户端对的不同工作示例。

# 从模板引擎开始

另一个非常强大的工具是 Go 模板引擎，可在`text/template`中使用。其功能在`html/template`包中得到复制和扩展，这构成了 Go Web 开发的另一个强大工具。

# 语法和基本用法

模板包使我们能够使用文本文件和数据结构将表示与数据分离。模板引擎定义了两个分隔符——左和右——用于表示数据评估的开启和关闭操作。默认的分隔符是`{{`和`}}`，模板只评估这些分隔符内包含的内容，其余部分保持不变。

通常绑定到模板的数据是一个结构或映射，并且可以在模板中的任何位置使用`$`变量访问。无论是映射还是结构，字段的访问方式始终相同，使用`.Field`语法。如果省略了美元符号，则该值将被引用为当前上下文，如果不在特殊语句中，例如循环，则为`$`。在这些例外之外，`{{$.Field}}`和`{{.Field}}`语句是等效的。

模板中的流程由条件语句`{{if}}`和循环语句`{{range}}`控制，并且两者都以`{{end}}`语句结束。条件语句还提供了链式`{{else if}}`语句的可能性来指定另一个条件，类似于开关，并且`{{else}}`语句可以被视为开关的默认情况。`{{else}}`可以与`range`语句一起使用，当 range 的参数为`nil`或长度为零时执行。

# 创建、解析和执行模板

`template.Template`类型是一个或多个模板的收集器，并且可以以多种方式初始化。`template.New`函数创建一个具有给定名称的新空模板，可以用于调用使用字符串创建模板的`Parse`方法。考虑以下代码：

```go
var data = struct {
    Question string
    Answer int
}{
    Question: "Answer to the Ultimate Question of Life, " +
        "the Universe, and Everything",
    Answer: 42,
}
tpl, err := template.New("question-answer").Parse(`
    <p>Question: {{.Question}}</p>
    <p>Answer: {{.Answer}}</p>
`)
if err != nil {
    log.Fatalln("Error:", err)
}
if err = tpl.Execute(os.Stdout, data); err != nil {
    log.Fatalln("Error:", err)
}
```

完整示例在此处可用：[`play.golang.org/p/k-t0Ns1b2Mv`](https://play.golang.org/p/k-t0Ns1b2Mv)

模板也可以从文件系统中加载和解析，使用`template.ParseFiles`，它接受一个文件列表，以及`template.ParseGlob`，它使用`glob` Unix 命令语法来选择文件列表。让我们创建一个包含以下内容的模板文件：

```go
<html>
    <body>
        <h1>{{.name}}</h1>
        <ul>
            <li>First appearance: {{.appearance}}</li>
            <li>Style: {{.style}}</li>
        </ul>
    </body>
</html>
```

我们可以使用这两个函数中的一个来加载并使用一些示例数据执行它：

```go
func main() {
    tpl, err := template.ParseGlob("ch9/template/parse/*.html")
    if err != nil {
        log.Fatal("Error:", err)
    }
    data := map[string]string{
        "name": "Jin Kazama",
        "style": "Karate",
        "appearance": "Tekken 3",
    }
    if err := tpl.Execute(os.Stdout, data); err != nil {
        log.Fatal("Error:", err)
    }
}
```

当加载多个模板时，`Execute`方法将使用最后一个。如果需要选择特定模板，则还有另一种方法`ExecuteTemplate`，它还接收模板名称作为参数，以指定要使用的模板。

# 条件和循环

`range`语句可以以不同的方式使用——最简单的方式就是调用`range`，后面跟着要迭代的切片或映射。

或者，您可以指定值，或索引和值：

```go
var a = []int{1, 2, 3, 4}
`{{ range . }} {{.}} {{ end }}` // simple
`{{ range $v := . }} {{$v}} {{ end }}` // value
`{{ range $i, $v := . }} {{$v}} {{ end }}` // index and value
```

在循环中，`{{.}}`变量假定为迭代中的当前元素的值。以下示例循环一个项目切片：

```go
var data = []struct {
    Question, Answer string
}{{
    Question: "Answer to the Ultimate Question of Life, " +
        "the Universe, and Everything",
    Answer: "42",
}, {
    Question: "Who you gonna call?",
    Answer: "Ghostbusters",
}}
tpl, err := template.New("question-answer").Parse(`{{range .}}
Question: {{.Question}}
Answer: {{.Answer}}
{{end}}`)
if err != nil {
    log.Fatalln("Error:", err)
}
if err = tpl.Execute(os.Stdout, data); err != nil {
    log.Fatalln("Error:", err)
}
```

完整示例在此处可用：[`play.golang.org/p/MtU_d9CsFb-`](https://play.golang.org/p/MtU_d9CsFb-)

下一个示例是条件语句的用例，也使用了`lt`函数：

```go
var data = []struct {
    Name string
    Score int
}{
    {"Michelangelo", 30},
    {"Donatello", 50},
    {"Leonardo", 80},
    {"Raffaello", 100},
}
tpl, err := template.New("question-answer").Parse(`{{range .}}
{{.Name}} scored {{.Score}}. He did {{if lt .Score 50}}bad{{else if lt .Score 75}}okay{{else if lt .Score 90}}good{{else}}great{{end}}
{{end}}`)
if err != nil {
    log.Fatalln("Error:", err)
}
if err = tpl.Execute(os.Stdout, data); err != nil {
    log.Fatalln("Error:", err)
}
```

完整示例在此处可用：[`play.golang.org/p/eBKDcJ47rPU`](https://play.golang.org/p/eBKDcJ47rPU)

我们将在下一节中更详细地探讨函数。

# 模板函数

函数是模板引擎的重要部分，有许多内置函数，例如比较（`eq`，`lt`，`gt`，`le`，`ge`）或逻辑（`AND`，`OR`，`NOT`）。函数通过它们的名称调用，后面跟着使用空格作为分隔符的参数。在前面的示例中使用的函数`lt a b`表示`lt(a,b)`。当函数嵌套更多时，需要用括号包裹函数和参数。例如，`not lt a b`语句表示`X`函数有三个参数，`not(lt, a, b)`。正确的版本是`not (lt a b)`，它告诉模板需要先解决括号中的元素。

在创建模板时，可以使用`Funcs`方法为其分配自定义函数，并在模板中使用。这非常有用，正如我们在这个例子中看到的：

```go
var data = struct {
    Name, Surname, Occupation, City string
}{
    "Bojack", "Horseman", "Actor", "Los Angeles",
}
tpl, err := template.New("question-answer").Funcs(template.FuncMap{
    "upper": func(s string) string { return strings.ToUpper(s) },
    "lower": func(s string) string { return strings.ToLower(s) },
}).Parse(`{{.Name}} {{.Surname}} - {{lower .Occupation}} from {{upper .City}}`)
if err != nil {
    log.Fatalln("Error:", err)
}
if err = tpl.Execute(os.Stdout, data); err != nil {
    log.Fatalln("Error:", err)
}
```

完整的示例在这里可用：[`play.golang.org/p/DdoKEOixDDB.`](https://play.golang.org/p/DdoKEOixDDB)

`|`运算符可用于将语句的输出链接到另一个语句的输入，类似于 Unix shell 中的情况。例如，`{{"put" | printf "%s%s" "out" | printf "%q"}}`语句将产生`"output"`。

# RPC 服务器

**远程过程调用**（**RPC**）是一种使用 TCP 协议从另一个系统调用应用功能执行的方法。Go 语言原生支持 RPC 服务器。

# 定义一个服务

Go RPC 服务器允许我们注册任何 Go 类型及其方法。这将使用 RPC 协议公开方法，并使我们能够通过名称从远程客户端调用它们。让我们创建一个辅助函数来跟踪我们在阅读书籍时的进度：

```go
// Book represents a book entry
type Book struct {
    ISBN string
    Title, Author string
    Year, Pages int
}

// ReadingList keeps tracks of books and pages read
type ReadingList struct {
    Books []Book
    Progress []int
}
```

首先，让我们定义一个名为`bookIndex`的小辅助方法，它使用书籍的标识符（ISBN）返回书籍的索引：

```go
func (r *ReadingList) bookIndex(isbn string) int {
    for i := range r.Books {
        if isbn == r.Books[i].ISBN {
            return i
        }
    }
    return -1
}
```

现在，我们可以定义`ReadingList`将能够执行的操作。它应该能够添加和删除书籍：

```go
// AddBook checks if the book is not present and adds it
func (r *ReadingList) AddBook(b Book) error {
    if b.ISBN == "" {
        return ErrISBN
    }
    if r.bookIndex(b.ISBN) != -1 {
        return ErrDuplicate
    }
    r.Books = append(r.Books, b)
    r.Progress = append(r.Progress, 0)
    return nil
}

// RemoveBook removes the book from list and forgets its progress
func (r *ReadingList) RemoveBook(isbn string) error {
    if isbn == "" {
        return ErrISBN
    }
    i := r.bookIndex(isbn)
    if i == -1 {
        return ErrMissing
    }
    // replace the deleted book with the last of the list
    r.Books[i] = r.Books[len(r.Books)-1]
    r.Progress[i] = r.Progress[len(r.Progress)-1]
    // shrink the list of 1 element to remove the duplicate
    r.Books = r.Books[:len(r.Books)-1]
    r.Progress = r.Progress[:len(r.Progress)-1]
    return nil
}
```

它还应该能够读取和修改书籍的进度：

```go
// GetProgress returns the progress of a book
func (r *ReadingList) GetProgress(isbn string) (int, error) {
 if isbn == "" {
 return -1, ErrISBN
 }
 i := r.bookIndex(isbn)
 if i == -1 {
 return -1, ErrMissing
 }
 return r.Progress[i], nil
}
```

然后，`SetProgress`改变书的进度，如下所示：

```go
func (r *ReadingList) SetProgress(isbn string, pages int) error {
 if isbn == "" {
 return ErrISBN
 }
 i := r.bookIndex(isbn)
 if i == -1 {
 return ErrMissing
 }
 if p := r.Books[i].Pages; pages > p {
 pages = p
 }
 r.Progress[i] = pages
 return nil
}
```

`AdvanceProgress`增加书的进度页数：

```go

func (r *ReadingList) AdvanceProgress(isbn string, pages int) error {
    if isbn == "" {
        return ErrISBN
    }
    i := r.bookIndex(isbn)
    if i == -1 {
        return ErrMissing
    }
    if p := r.Books[i].Pages - r.Progress[i]; p < pages {
        pages = p
    }
    r.Progress[i] += pages
    return nil
}
```

我们在这些函数中使用的错误变量定义如下：

```go
// List of errors
var (
    ErrISBN = fmt.Errorf("missing ISBN")
    ErrDuplicate = fmt.Errorf("duplicate book")
    ErrMissing = fmt.Errorf("missing book")
)
```

# 创建服务器

现在我们有了可以轻松创建 RPC 服务器的服务。但是，所使用的类型必须遵守一些规则，以使其方法可用：

+   方法的类型和方法本身都是导出的。

+   该方法有两个参数，都是导出的。

+   第二个参数是一个指针。

+   该方法返回一个错误。

该方法应该看起来像这样：`func (t *T) Method(in T1, out *T2) error.`

下一步是创建一个满足这些规则的`ReadingList`的包装器：

```go
// ReadingService adapts ReadingList for RPC
type ReadingService struct {
    ReadingList
}

// sets the success pointer value from error
func setSuccess(err error, b *bool) error {
    *b = err == nil
    return err
}
```

我们可以重新定义书籍，使用`Book`添加和删除函数，这是一个导出类型和内置类型：

```go
func (r *ReadingService) AddBook(b Book, success *bool) error {
    return setSuccess(r.ReadingList.AddBook(b), success)
}

func (r *ReadingService) RemoveBook(isbn string, success *bool) error {
    return setSuccess(r.ReadingList.RemoveBook(isbn), success)
}
```

对于进度，我们有两个输入（ISBN 和页数），因此我们必须定义一个包含两者的结构，因为输入必须是单个参数：

```go
func (r *ReadingService) GetProgress(isbn string, pages *int) (err error) {
    *pages, err = r.ReadingList.GetProgress(isbn)
    return err
}

type Progress struct {
    ISBN string
    Pages int
}

func (r *ReadingService) SetProgress(p Progress, success *bool) error {
    return setSuccess(r.ReadingList.SetProgress(p.ISBN, p.Pages), success)
}

func (r *ReadingService) AdvanceProgress(p Progress, success *bool) error {
    return setSuccess(r.ReadingList.AdvanceProgress(p.ISBN, p.Pages), success)
}
```

定义的类型可以在 RPC 服务器中注册并使用，它将使用`rpc.HandleHTTP`来注册传入 RPC 消息的 HTTP 处理程序：

```go
if len(os.Args) != 2 {
    log.Fatalln("Please specify an address.")
}
if err := rpc.Register(&common.ReadingService{}); err != nil {
    log.Fatalln(err)
}
rpc.HandleHTTP()

l, err := net.Listen("tcp", os.Args[1])
if err != nil {
    log.Fatalln(err)
}
log.Println("Server Started")
if err := http.Serve(l, nil); err != nil {
    log.Fatal(err)
}
```

# 创建客户端

可以使用 RPC 包的`rpc.DialHTTP`函数创建客户端，使用相同的主机端口来获取客户端：

```go
if len(os.Args) != 2 {
    log.Fatalln("Please specify an address.")
}
client, err := rpc.DialHTTP("tcp", os.Args[1])
if err != nil {
    log.Fatalln(err)
}
defer client.Close()
```

然后，我们定义了一个我们将在示例中使用的书籍列表：

```go
const hp = "H.P. Lovecraft"
var books = []common.Book{
    {ISBN: "1540335534", Author: hp, Title: "The Call of Cthulhu", Pages: 36},
    {ISBN: "1980722803", Author: hp, Title: "The Dunwich Horror ", Pages: 53},
    {ISBN: "197620299X", Author: hp, Title: "The Shadow Over Innsmouth", Pages: 40},
    {ISBN: "1540335534", Author: hp, Title: "The Case of Charles Dexter Ward", Pages: 176},
}
```

考虑到格式包会打印内置类型指针的地址，我们将定义一个辅助函数来显示指针的内容：

```go
func callClient(client *rpc.Client, method string, in, out interface{}) {
    var r interface{}
    if err := client.Call(method, in, out); err != nil {
        out = err
    }
    switch v := out.(type) {
    case error:
        r = v
    case *int:
        r = *v
    case *bool:
        r = *v
    }
    log.Printf("%s: [%+v] -> %+v", method, in, r)
}
```

客户端以`type.method`的形式获取要执行的操作，因此我们将使用这样的函数：

```go
callClient(client, "ReadingService.GetProgress", books[0].ISBN, new(int))
callClient(client, "ReadingService.AddBook", books[0], new(bool))
callClient(client, "ReadingService.AddBook", books[0], new(bool))
callClient(client, "ReadingService.GetProgress", books[0].ISBN, new(int))
callClient(client, "ReadingService.AddBook", books[1], new(bool))
callClient(client, "ReadingService.AddBook", books[2], new(bool))
callClient(client, "ReadingService.AddBook", books[3], new(bool))
callClient(client, "ReadingService.SetProgress", common.Progress{
    ISBN: books[3].ISBN,
    Pages: 10,
}, new(bool))
callClient(client, "ReadingService.GetProgress", books[3].ISBN, new(int))
callClient(client, "ReadingService.AdvanceProgress", common.Progress{
    ISBN: books[3].ISBN,
    Pages: 40,
}, new(bool))
callClient(client, "ReadingService.GetProgress", books[3].ISBN, new(int))
```

这将输出每个操作及其结果。

# 总结

在这一章中，我们研究了 Go 语言中如何处理网络连接。我们从一些网络标准开始。首先，我们讨论了 OSI 模型，然后是 TCP/IP。

然后，我们检查了网络包，并学习了如何使用它来创建和管理 TCP 连接。这包括处理特殊命令以及如何从服务器端终止连接。接下来，我们看到如何使用 UDP 做同样的事情，并且我们已经看到如何实现具有校验和控制的自定义编码。

然后，我们讨论了 HTTP 协议，解释了第一个版本的工作原理，然后谈到了 HTTP/2 的差异和改进。然后，我们学习了如何使用 Go 发出 HTTP 请求，然后是如何设置 Web 服务器。我们探讨了如何提供现有文件，如何将不同的操作关联到不同的 HTTP 方法，以及如何处理多部分请求和文件上传。我们轻松地设置了一个 HTTPS 服务器，然后学习了一些第三方库为 Web 服务器提供的优势。最后，我们演示了模板引擎在 Go 中的工作原理，以及如何轻松构建 RPC 客户端/服务器。

在下一章中，我们将介绍如何使用 JSON 和 XML 等主要数据交换格式，这些格式也可以用于创建 Web 服务器。

# 问题

1.  使用通信模型有什么优势？

1.  TCP 连接和 UDP 连接之间有什么区别？

1.  在发送请求时，谁关闭了请求体？

1.  在服务器接收时，谁关闭了请求体？
