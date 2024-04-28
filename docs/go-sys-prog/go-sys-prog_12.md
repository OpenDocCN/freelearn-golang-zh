# 网络编程

在上一章中，我们讨论了在 Go 中开发 Web 应用程序、与数据库通信以及处理 JSON 数据。

本章的主题是开发在 TCP/IP 网络上运行的 Go 应用程序。此外，您还将学习如何创建 TCP 和 UDP 客户端和服务器。本章的核心 Go 包将是`net`包：它的大多数函数都是相当低级的，需要对 TCP/IP 及其协议家族有很好的了解。

然而，请记住，网络编程是一个庞大的主题，无法在单独的一章中涵盖。本章将为您提供如何在 Go 中创建 TCP/IP 应用程序的基本方向。

更具体地说，本章将讨论以下主题：

+   TCP/IP 的操作方式

+   Go 标准包`net`

+   开发 TCP 客户端和服务器

+   编程 UDP 客户端和服务器

+   开发 RPC 客户端

+   实现 RPC 服务器

+   Wireshark 和`tshark(1)`网络流量分析器

+   Unix 套接字

+   从 Go 程序执行 DNS 查找

# 关于网络编程

**网络编程**是开发可以使用 TCP/IP 在计算机网络上运行的应用程序。因此，如果不了解 TCP/IP 及其协议的工作方式，就无法创建网络应用程序和开发 TCP/IP 服务器。

我可以给网络应用程序开发人员的最好的两个建议是了解他们想要执行的任务背后的理论，并且知道网络由于多种原因而经常失败。网络故障中最恶劣的类型与故障或配置错误的 DNS 服务器有关，因为这类问题很难找到并且难以纠正。

# 关于 TCP/IP

**TCP/IP**是一组协议，帮助互联网运行。它的名称来自其两个最著名的协议：**TCP**和**IP**。

每个使用 TCP/IP 的设备必须具有 IP 地址，至少在其本地网络中是唯一的。它还需要一个与当前网络相关的**网络掩码**（用于将大型 IP 网络划分为较小的网络），一个或多个**DNS 服务器**（用于将 IP 地址转换为人类可记忆的格式，反之亦然），以及如果要与本地网络之外的设备通信，则需要一个将充当**默认网关**（当 TCP/IP 找不到其他发送位置时，将网络数据包发送到的网络设备）的设备的 IP 地址。

每个 TCP/IP 服务实际上是一个 Unix 进程，监听一个对每台机器都是唯一的端口号。请注意，端口号 0-1023 受限制，只能由 root 用户使用，因此最好避免使用它们，并选择其他内容，前提是它尚未被不同进程使用。

# 关于 TCP

**TCP**代表**传输** **控制** **协议**。TCP 软件使用称为 TCP **数据包**的段在机器之间传输数据。TCP 的主要特点是它是一种可靠的协议，这意味着它试图确保数据包已传送。如果没有数据包传送的证据，TCP 会重新发送该特定数据包。除其他事项外，TCP 数据包可用于建立连接、传输数据、发送确认和关闭连接。

当两台机器之间建立 TCP 连接时，类似于电话呼叫的全双工虚拟电路将在这两台机器之间创建。这两台机器不断通信以确保数据正确发送和接收。如果由于某种原因连接失败，这两台机器会尝试找到问题并向相关应用程序报告。

TCP 为每个传输的数据包分配一个序列号，并期望接收 TCP 堆栈的正面确认（ACK）。如果在超时间隔内未收到 ACK，则数据将被重新传输，因为原始数据包被视为未传递。当数据包以无序方式到达时，接收 TCP 堆栈使用序列号重新排列段，这也消除了重复的段。

每个数据包的 TCP 头包括**源端口和目标端口**字段。这两个字段加上源和目标 IP 地址被组合在一起，以唯一标识每个 TCP 连接。TCP 头还包括一个 6 位标志字段，用于在 TCP 对等方之间传递控制信息。可能的标志包括 SYN，FIN，RESET，PUSH，URG 和 ACK。SYN 和 ACK 标志用于初始 TCP 3 次握手。RESET 标志表示接收方希望中止连接。

# TCP 握手！

当建立连接时，客户端向服务器发送 TCP SYN 数据包。TCP 头还包括一个序列号字段，在 SYN 数据包中具有任意值。服务器发送回一个 TCP [SYN，ACK]数据包，其中包括相反方向的序列号和对先前序列号的确认。最后，为了真正建立 TCP 连接，客户端发送 TCP ACK 数据包以确认服务器的序列号。

尽管所有这些操作都是自动进行的，但了解幕后发生的事情是很好的！

# 关于 UDP 和 IP

**IP**代表**Internet Protocol**。IP 的主要特点是它本质上不是一种可靠的协议。IP 封装了在 TCP/IP 网络中传输的数据，因为它负责根据 IP 地址将数据包从源主机传递到目标主机。IP 必须找到一种寻址方法，以有效地将数据包发送到其目的地。尽管存在称为路由器的专用设备来执行 IP 路由，但每个 TCP/IP 设备都必须执行一些基本路由。

**UDP**（**用户数据报协议**的缩写）基于 IP，这意味着它也是不可靠的。一般来说，UDP 比 TCP 简单，主要是因为 UDP 本身设计上就不可靠。因此，UDP 消息可能会丢失、重复或无序到达。此外，数据包可能比接收方处理它们的速度更快。因此，当速度比可靠性更重要时，使用 UDP！一个例子是实时视频和音频应用程序，其中追赶速度比缓冲和不丢失任何数据更重要！

因此，当您不需要太多的网络数据包来传输所需的信息时，使用基于 IP 的协议可能比使用 TCP 更有效，即使您必须重新传输网络数据包，因为没有来自 TCP 握手的流量开销。

# 关于 Wireshark 和 tshark

**Wireshark**是一款用于分析几乎任何类型的网络流量的图形应用程序。然而，有时您需要一些更轻便的东西，可以在没有图形用户界面的情况下远程执行。在这种情况下，您可以使用`tshark`，这是 Wireshark 的命令行版本。

为了帮助您找到真正想要的网络数据，Wireshark 和`tshark`支持捕获过滤器和显示过滤器。

捕获过滤器是在网络数据捕获过程中应用的过滤器；因此，它们使 Wireshark 丢弃不符合过滤条件的网络流量。显示过滤器是在数据包捕获后应用的过滤器；因此，它们只是隐藏一些网络流量而不是删除它：您可以随时禁用显示过滤器并恢复隐藏的数据。一般来说，显示过滤器被认为比捕获过滤器更有用和更灵活，因为通常情况下，您事先不知道要捕获或要检查什么。然而，在捕获时应用过滤器可以节省时间和磁盘空间，这是使用它们的主要原因。

以下屏幕截图显示了 Wireshark 捕获的 TCP 握手流量的更详细信息。客户端 IP 地址为`10.0.2.15`，目标 IP 地址为`80.244.178.150`。此外，简单的显示过滤器(`tcp && !http`)使 Wireshark 显示更少的数据包，并使输出更清晰，因此更容易阅读：

![](img/4cd7d321-edd4-4d49-8713-bc9cea9535f6.png)

TCP 握手！

可以使用`tshark(1)`以文本格式查看相同的信息：

```go
$ tshark -r handshake.pcap -Y '(tcp.flags.syn==1 ) || (tcp.flags == 0x0010 && tcp.seq==1 && tcp.ack==1)'
       18   5.144264    10.0.2.15 → 80.244.178.150 TCP 74 59897 → 80 [SYN] Seq=0 Win=29200 Len=0 MSS=1460 SACK_PERM=1 TSval=1585402 TSecr=0 WS=128
       19   5.236792 80.244.178.150 → 10.0.2.15    TCP 60 80 → 59897 [SYN, ACK] Seq=0 Ack=1 Win=65535 Len=0 MSS=1460
       20   5.236833    10.0.2.15 → 80.244.178.150 TCP 54 59897 → 80 [ACK] Seq=1 Ack=1 Win=29200 Len=0
```

`-r`参数后跟一个现有的文件名，允许您在屏幕上重放先前捕获的数据文件，而更复杂的显示过滤器在`-Y`参数之后定义，完成其余工作！

您可以在[`www.wireshark.org/`](https://www.wireshark.org/)了解更多关于 Wireshark 的信息，并通过查看其文档[`www.wireshark.org/docs/`](https://www.wireshark.org/docs/)。

# 关于 netcat 实用程序

有时您需要测试 TCP/IP 客户端或 TCP/IP 服务器：`netcat(1)`实用程序可以通过在 TCP 或 UDP 应用程序中扮演客户端或服务器的角色来帮助您。

您可以使用`netcat(1)`作为 TCP 服务的客户端，该服务在具有`192.168.1.123` IP 地址的计算机上运行，并侦听端口号`1234`，如下所示：

```go
$ netcat 192.168.1.123 1234
```

同样，您可以使用`netcat(1)`作为运行在名为`amachine.com`的 Unix 机器上并侦听端口号`2345`的 UDP 服务的客户端，如下所示：

```go
$ netcat -vv -u amachine.com 2345
```

`-l`选项告诉`netcat(1)`监听传入连接，这使`netcat(1)`充当 TCP 或 UDP 服务器。如果尝试使用`netcat(1)`作为具有已在使用的端口的服务器，则将获得以下输出：

```go
$ netcat -vv -l localhost -p 80
Can't grab 0.0.0.0:80 with bind : Permission denied
```

# net Go 标准包

用于创建 TCP/IP 应用程序的最有用的 Go 包是`net` Go 标准包。`net.Dial()`函数用于作为客户端连接到网络，`net.Listen()`函数用于作为服务器接受连接。这两个函数的第一个参数都是网络类型，但相似之处就到此为止了。

对于`net.Dial()`函数，网络类型可以是 tcp、tcp4（仅限 IPv4）、tcp6（仅限 IPv6）、udp、udp4（仅限 IPv4）、udp6（仅限 IPv6）、ip、ip4（仅限 IPv4）、ip6（仅限 IPv6）、Unix、Unixgram 或 Unixpacket。对于`net.Listen()`函数，第一个参数可以是 tcp、tcp4、tcp6、Unix 或 Unixpacket。

`net.Dial()`函数的返回值是`net.Conn`接口类型，该接口实现了`io.Reader`和`io.Writer`接口！这意味着您已经知道如何访问`net.Conn`接口的变量！

因此，尽管创建网络连接的方式与创建文本文件的方式不同，但它们的访问方法是相同的，因为`net.Conn`接口实现了`io.Reader`和`io.Writer`接口。因此，由于网络连接被视为文件，您可能需要在此时查看第六章*，* *文件输入和输出*。

# Unix 套接字重温

回到第八章*，* *进程和信号*，我们简要讨论了 Unix 套接字，并介绍了一个作为 Unix 套接字客户端的小型 Go 程序。本节还将创建一个 Unix 套接字服务器，以便更清楚地说明问题。但是，Unix 套接字客户端的 Go 代码也将在此处更详细地解释，并丰富了错误处理代码。

# 一个 Unix 套接字服务器

Unix 套接字服务器将充当 Echo 服务器，这意味着它将将接收到的消息发送回客户端。程序的名称将是`socketServer.go`，将分为四部分介绍给您。

`socketServer.go`的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
) 
```

Unix 套接字服务器的第二部分如下：

```go
func echoServer(c net.Conn) { 
   for { 
         buf := make([]byte, 1024) 
         nr, err := c.Read(buf) 
         if err != nil { 
               return 
         } 

         data := buf[0:nr] 
         fmt.Printf("->: %v\n", string(data)) 
         _, err = c.Write(data) 
         if err != nil { 
               fmt.Println(err) 
         } 
   } 
} 
```

这是实现服务传入连接的函数所在之处。

程序的第三部分包含以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a socket file.") 
         os.Exit(100) 
   } 
   socketFile := arguments[1] 

   l, err := net.Listen("unix", socketFile) 
   if err != nil { 
         fmt.Println(err) 
os.Exit(100) 
   } 
```

在这里，您可以看到使用`net.Listen()`函数和`unix`参数创建所需的套接字文件。

最后，最后一部分包含以下 Go 代码：

```go
   for { 
         fd, err := l.Accept() 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
         go echoServer(fd) 
   } 
} 
```

如您所见，每个连接首先由`Accept()`函数处理，并由其自己的 goroutine 提供服务。

当`socketServer.go`为客户端提供服务时，它会生成以下输出：

```go
$ go run socketServer.go /tmp/aSocket
->: Hello Server!
```

如果无法创建所需的套接字文件，例如，如果它已经存在，您将收到类似以下的错误消息：

```go
$ go run socketServer.go /tmp/aSocket
listen unix /tmp/aSocket: bind: address already in use
exit status 100
```

# 一个 Unix 套接字客户端

Unix 套接字客户端程序的名称是`socketClient.go`，将分为四部分介绍。

实用程序的第一部分包含了预期的序言：

```go
package main 

import ( 
   "fmt" 
   "io" 
   "log" 
   "net" 
   "os" 
   "time" 
) 
```

这里没有什么特别的，只是所需的 Go 包。第二部分包含了一个 Go 函数的定义：

```go
func readSocket(r io.Reader) {

   buf := make([]byte, 1024) 
   for { 
         n, err := r.Read(buf[:]) 
         if err != nil { 
               fmt.Println(err) 
               return 
         } 
         fmt.Println("-> ", string(buf[0:n])) 
   } 
} 
```

`readSocket()`函数使用`Read()`从套接字文件中读取数据。请注意，尽管`socketClient.go`只是从套接字文件中读取数据，但套接字是双向的，这意味着您也可以向其写入数据。

第三部分包含以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a socket file.") 
         os.Exit(100) 
   } 
   socketFile := arguments[1] 

   c, err := net.Dial("unix", socketFile) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
   defer c.Close() 
```

使用正确的第一个参数的`net.Dial()`函数允许您在尝试从中读取之前连接到套接字文件。

`socketClient.go`的最后一部分如下：

```go
   go readSocket(c) 
   for { 
         _, err := c.Write([]byte("Hello Server!")) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
         time.Sleep(1 * time.Second) 
   } 
} 
```

要使用`socketClient.go`，您必须有另一个处理 Unix 套接字文件的程序，在本例中将是`socketServer.go`。因此，如果`socketServer.go`已经在运行，您将从`socketClient.go`获得以下输出：

```go
$ go run socketClient.go /tmp/aSocket
->: Hello Server!
```

如果您没有足够的 Unix 文件权限来读取所需的套接字文件，那么`socketClient.go`将失败，并显示以下错误消息：

```go
$ go run socketClient.go /tmp/aSocket
dial unix /tmp/aSocket: connect: permission denied
exit status 100
```

同样，如果您要读取的套接字文件不存在，`socketClient.go`将失败，并显示以下错误消息：

```go
$ go run socketClient.go /tmp/aSocket
dial unix /tmp/aSocket: connect: no such file or directory
exit status 100
```

# 执行 DNS 查找

存在许多类型的 DNS 查找，但其中两种最受欢迎。在第一种类型中，您希望从 IP 地址转到域名，而在第二种类型中，您希望从域名转到 IP 地址。

以下输出显示了第一种类型的 DNS 查找的示例：

```go
$ host 109.74.193.253
253.193.74.109.in-addr.arpa domain name pointer li140-253.members.linode.com.
```

以下输出显示了第二种类型的 DNS 查找的三个示例：

```go
$ host www.mtsoukalos.eu
www.mtsoukalos.eu has address 109.74.193.253
$ host www.highiso.net
www.highiso.net has address 109.74.193.253
$ host -t a cnn.com
cnn.com has address 151.101.1.67
cnn.com has address 151.101.129.67
cnn.com has address 151.101.65.67
cnn.com has address 151.101.193.67
```

正如您在上述示例中所看到的，一个 IP 地址可以为多个主机提供服务，一个主机名可以有多个 IP 地址。

Go 标准库提供了`net.LookupHost()`和`net.LookupAddr()`函数，可以为您回答 DNS 查询。但是，它们都不允许您定义要查询的 DNS 服务器。虽然使用标准的 Go 库是理想的，但存在外部的 Go 库，允许您选择所需的 DNS 服务器，这在排除 DNS 配置问题时是非常重要的。

# 使用 IP 地址作为输入

将返回 IP 地址的主机名的 Go 实用程序的名称将是`lookIP.go`，将分为三部分介绍。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
) 
```

第二部分包含以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide an IP address!") 
         os.Exit(100) 
   } 

   IP := arguments[1] 
   addr := net.ParseIP(IP) 
   if addr == nil { 
         fmt.Println("Not a valid IP address!") 
         os.Exit(100) 
   } 
```

`net.ParseIP()`函数允许您验证给定 IP 地址的有效性，并且对于捕获诸如`288.8.8.8`和`8.288.8.8`之类的非法 IP 地址非常方便。

实用程序的最后部分如下：

```go
   hosts, err := net.LookupAddr(IP) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   for _, hostname := range hosts { 
         fmt.Println(hostname) 
   } 
} 
```

正如您所看到的，`net.LookupAddr()`函数返回一个字符串切片，其中包含与给定 IP 地址匹配的名称列表。

执行`lookIP.go`将生成以下输出：

```go
$ go run lookIP.go 288.8.8.8
Not a valid IP address!
exit status 100
$ go run lookIP.go 8.8.8.8
google-public-dns-a.google.com.
```

您可以使用`host(1)`或`dig(1)`验证`dnsLookup.go`的输出：

```go
$ host 8.8.8.8
8.8.8.8.in-addr.arpa domain name pointer google-public-dns-a.google.com.
```

# 使用主机名作为输入

此 DNS 实用程序的名称将是`lookHost.go`，并将分为三部分呈现。`lookHost.go`实用程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
) 
```

程序的第二部分有以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide an argument!") 
         os.Exit(100) 
   } 

   hostname := arguments[1] 
   IPs, err := net.LookupHost(hostname) 
```

同样，`net.LookupHost()`函数也返回一个包含所需信息的字符串切片。

程序的第三部分包含以下代码，用于错误检查和打印`net.LookupHost()`的输出：

```go
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   for _, IP := range IPs { 
         fmt.Println(IP) 
   } 
} 
```

执行`lookHost.go`将生成以下输出：

```go
$ go run lookHost.go www.google
lookup www.google: no such host
exit status 100
$ go run lookHost.go www.google.com
2a00:1450:4001:81f::2004
172.217.16.164
```

输出的第一行是 IPv6 地址，而第二行输出是`www.google.com`的 IPv4 地址。

您可以通过将其输出与`host(1)`实用程序的输出进行比较来验证`lookHost.go`的操作：

```go
$ host www.google.com
www.google.com has address 172.217.16.164
www.google.com has IPv6 address 2a00:1450:4001:81a::2004
```

# 获取域的 NS 记录

本小节将介绍另一种返回给定域的域名服务器的 DNS 查找。这对于解决与 DNS 相关的问题并了解域的状态非常方便。所呈现的程序将被命名为`lookNS.go`，并将分为三部分呈现。

实用程序的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
) 
```

第二部分有以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a domain!") 
         os.Exit(100) 
   } 

   domain := arguments[1] 

   NSs, err := net.LookupNS(domain) 
```

`net.LookupNS()`函数通过返回`NS`元素的切片为我们完成所有工作。

代码的最后部分主要用于打印结果：

```go
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   for _, NS := range NSs { 
         fmt.Println(NS.Host) 
   } 
} 
```

执行`lookNS.go`将生成以下输出：

```go
$ go run lookNS.go mtsoukalos.eu
ns5.linode.com.
ns2.linode.com.
ns3.linode.com.
ns1.linode.com.
ns4.linode.com.
```

以下查询失败的原因是`www.mtsoukalos.eu`不是一个域，而是一个单个主机，这意味着它没有与之关联的`NS`记录：

```go
$ go run lookNS.go www.mtsoukalos.eu
lookup www.mtsoukalos.eu on 8.8.8.8:53: no such host
exit status 100
```

您可以使用`host(1)`实用程序验证先前的输出：

```go
$ host -t ns mtsoukalos.eu
mtsoukalos.eu name server ns5.linode.com.
mtsoukalos.eu name server ns4.linode.com.
mtsoukalos.eu name server ns3.linode.com.
mtsoukalos.eu name server ns1.linode.com.
mtsoukalos.eu name server ns2.linode.com.
$ host -t ns www.mtsoukalos.eu
www.mtsoukalos.eu has no NS record
```

# 开发一个简单的 TCP 服务器

本节将开发一个实现**Echo**服务的 TCP 服务器。Echo 服务通常使用 UDP 协议实现，因为它简单，但也可以使用 TCP 实现。Echo 服务通常使用端口号`7`，但我们的实现将使用其他端口号：

```go
$ grep echo /etc/services
echo        7/tcp
echo        7/udp
```

`TCPserver.go`文件将保存本节的 Go 代码，并将分为六部分呈现。出于简单起见，每个连接都在`main()`函数中处理，而不调用单独的函数。但是，这不是推荐的做法。

第一部分包含预期的序言：

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

TCP 服务器的第二部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide port number") 
         os.Exit(100) 
   } 
```

`TCPserver.go`的第三部分包含以下 Go 代码：

```go
   PORT := ":" + arguments[1] 
   l, err := net.Listen("tcp", PORT) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
   defer l.Close() 
```

在这里需要记住的重要一点是，`net.Listen()`返回一个`Listener`变量，这是一个用于面向流的协议的通用网络监听器。此外，`Listen()`函数可以支持更多格式：查看`net`包的文档以获取更多信息。

TCP 服务器的第四部分有以下 Go 代码：

```go
   c, err := l.Accept() 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

只有在成功调用`Accept()`之后，TCP 服务器才能开始与 TCP 客户端交互。尽管如此，当前版本的`TCPserver.go`有一个非常严重的缺点：它只能为单个 TCP 客户端提供服务，即连接到它的第一个客户端。

`TCPserver.go`代码的第五部分如下：

```go
   for { 
         netData, err := bufio.NewReader(c).ReadString('\n') 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
```

在这里，您可以使用`bufio.NewReader().ReadString()`从客户端读取数据。上述调用允许您逐行读取输入。此外，`for`循环允许您从 TCP 客户端持续读取数据，直到您希望停止为止。

Echo TCP 服务器的最后部分如下：

```go
         fmt.Print("-> ", string(netData)) 
         c.Write([]byte(netData)) 
         if strings.TrimSpace(string(netData)) == "STOP" { 
               fmt.Println("Exiting TCP server!") 
               return 
         } 
   } 
} 
```

当前版本的`TCPserver.go`在接收到`STOP`字符串作为输入时停止。虽然 TCP 服务器通常不会以这种方式终止，但这是终止仅为单个客户端提供服务的 TCP 服务器进程的一种非常方便的方式！

接下来，我们将使用`netcat(1)`测试`TCPserver.go`：

```go
$ go run TCPserver.go 1234
-> Hi!
-> STOP
Exiting TCP server!
```

`netcat(1)`部分如下：

```go
$ nc localhost 1234 
Hi!
Hi!
STOP
STOP
```

这里，第一行和第三行是我们的输入，而第二行和第四行是 Echo 服务器的响应。

如果您尝试使用不正确的端口号，`TCPserver.go`将生成以下错误消息并退出：

```go
$ go run TCPserver.go 123456
listen tcp: address 123456: invalid port
exit status 100
```

# 开发一个简单的 TCP 客户端

在本节中，我们将开发一个名为`TCPclient.go`的 TCP 客户端。客户端将尝试连接的端口号以及服务器地址将作为程序的命令行参数给出。TCP 客户端的 Go 代码将分为五个部分进行介绍；第一部分如下：

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

`TCPclient.go`的第二部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide host:port.") 
         os.Exit(100) 
   } 
```

`TCPclient.go`的第三部分包含以下 Go 代码：

```go
   CONNECT := arguments[1] 
   c, err := net.Dial("tcp", CONNECT) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

再次，您使用`net.Dial()`函数尝试连接到所需 TCP 服务器的所需端口。

TCP 客户端的第四部分如下：

```go
   for { 
         reader := bufio.NewReader(os.Stdin) 
         fmt.Print(">> ") 
         text, _ := reader.ReadString('\n') 
         fmt.Fprintf(c, text+"\n") 
```

在这里，您从用户那里读取数据，然后使用`fmt.Fprintf()`将其发送到 TCP 服务器。

`TCPclient.go`的最后部分如下：

```go
         message, _ := bufio.NewReader(c).ReadString('\n') 
         fmt.Print("->: " + message) 
         if strings.TrimSpace(string(text)) == "STOP" { 
               fmt.Println("TCP client exiting...") 
               return 
         } 
   } 
} 
```

在这部分中，您将使用`bufio.NewReader().ReadString()`从 TCP 服务器获取数据。使用`strings.TrimSpace()`函数的原因是从要与静态字符串（`STOP`）进行比较的变量中删除任何空格和换行符。

所以，现在是时候验证`TCPclient.go`是否按预期工作，使用它连接到`TCPserver.go`：

```go
$ go run TCPclient.go localhost:1024
>> 123
->: 123
>> Hello server!
->: Hello server!
>> STOP
->: STOP
TCP client exiting...
```

如果在指定的主机上指定的 TCP 端口没有进程在监听，那么您将收到类似以下的错误消息：

```go
$ go run TCPclient.go localhost:1024
dial tcp [::1]:1024: getsockopt: connection refused
exit status 100
```

# 使用其他函数来实现 TCP 服务器

在这个小节中，我们将使用一些略有不同的函数来开发`TCPserver.go`的功能。新的 TCP 服务器的名称将是`TCPs.go`，将分为四个部分进行介绍。

`TCPs.go`的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
) 
```

TCP 服务器的第二部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a port number!") 
         os.Exit(100) 
   } 

   SERVER := "localhost" + ":" + arguments[1] 
```

到目前为止，与`TCPserver.go`的代码没有区别。

区别在`TCPs.go`的第三部分开始，如下：

```go
   s, err := net.ResolveTCPAddr("tcp", SERVER) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   l, err := net.ListenTCP("tcp", s) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

在这里，您使用`net.ResolveTCPAddr()`和`net.ListenTCP()`函数。这个版本比`TCPserver.go`更好吗？实际上并不是。但是 Go 代码可能看起来更清晰一些，这对一些人来说是一个很大的优势。另外，`net.ListenTCP()`返回一个`TCPListener`值，当与`net.Accept()`而不是`net.Accept()`一起使用时，将返回`TCPConn`，它提供了更多的方法，允许您更改更多的套接字选项。

`TCPs.go`的最后部分包含以下 Go 代码：

```go
   buffer := make([]byte, 1024) 

   for { 
         conn, err := l.Accept() 
         n, err := conn.Read(buffer) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 

         fmt.Print("> ", string(buffer[0:n]))

         _, err = conn.Write(buffer) 

         conn.Close() 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
   } 
} 
```

这里没有什么特别的。您仍然使用`Accept()`来获取和处理客户端请求。但是，这个版本使用`Read()`一次性获取客户端数据，这在您不必处理大量输入时非常方便。

`TCPs.go`的操作与`TCPserver.go`的操作相同，因此这里不会展示。

如果您尝试使用无效的端口号创建 TCP 服务器，`TCPs.go`将生成如下信息的错误消息：

```go
$ go run TCPs.go 123456
address 123456: invalid port
exit status 100
```

# 使用替代函数来实现 TCP 客户端

再次，我们将使用一些略有不同的函数来实现`TCPclient.go`，这些函数由`net` Go 标准包提供。新版本的名称将是`TCPc.go`，将分为四个代码段进行展示。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
) 
```

程序的第二个代码段如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a server:port string!") 
         os.Exit(100) 
   } 

   CONNECT := arguments[1] 
   myMessage := "Hello from TCP client!\n" 
```

这一次，我们将向 TCP 服务器发送一个静态消息。

`TCPc.go`的第三部分如下：

```go
   tcpAddr, err := net.ResolveTCPAddr("tcp", CONNECT) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   conn, err := net.DialTCP("tcp", nil, tcpAddr) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

在这部分中，您将看到`net.ResolveTCPAddr()`和`net.DialTCP()`的使用，这是`TCPc.go`和`TCPclient.go`之间的区别所在。

TCP 客户端的最后部分如下：

```go
   _, err = conn.Write([]byte(myMessage)) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   fmt.Print("-> ", myMessage) 
   buffer := make([]byte, 1024)

   n, err := conn.Read(buffer) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   fmt.Print(">> ", string(buffer[0:n])) 
   conn.Close() 
} 
```

您可能会问是否可以将`TCPc.go`与`TCPserver.go`或`TCPs.go`与`TCPclient.go`一起使用。答案是肯定的，因为实现和函数名称与实际进行的 TCP/IP 操作无关。

# 开发一个简单的 UDP 服务器

本节还将开发一个 Echo 服务器。但是，这次 Echo 服务器将使用 UDP 协议。程序的名称将是`UDPserver.go`，并将分为五个部分呈现给您。

第一部分包含了预期的序言：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
   "strings" 
) 
```

第二部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a port number!") 
         os.Exit(100) 
   } 
   PORT := ":" + arguments[1] 
```

`UDPserver.go`的第三部分如下：

```go
   s, err := net.ResolveUDPAddr("udp", PORT) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   connection, err := net.ListenUDP("udp", s) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

UDP 方法与 TCP 方法类似：只需调用不同名称的函数。

程序的第四部分包含以下 Go 代码：

```go
   defer connection.Close() 
   buffer := make([]byte, 1024) 

   for { 
         n, addr, err := connection.ReadFromUDP(buffer) 
         fmt.Print("-> ", string(buffer[0:n])) 
         data := []byte(buffer[0:n]) 
         _, err = connection.WriteToUDP(data, addr) 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
```

在 UDP 情况下，您使用`ReadFromUDP()`从 UDP 连接读取数据，并使用`WriteToUDP()`向 UDP 连接写入数据。此外，UDP 连接不需要调用类似于`net.Accept()`的函数。

UDP 服务器的最后一部分如下：

```go
         if strings.TrimSpace(string(data)) == "STOP" { 
               fmt.Println("Exiting UDP server!") 
               return 
         } 
   } 
} 
```

我们将再次使用`netcat(1)`测试`UDPserver.go`：

```go
$ go run UDPserver.go 1234
-> Hi!
-> Hello!
-> STOP
Exiting UDP server!
```

# 开发一个简单的 UDP 客户端

在本节中，我们将开发一个 UDP 客户端，我们将命名为`UDPclient.go`并分为五个部分。

正如您将看到的，`UDPclient.go`和`TCPc.go`的 Go 代码之间的代码差异基本上是所使用函数名称的差异：总体思路是完全相同的。

UDP 客户端的第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "os" 
) 
```

实用程序的第二部分包含以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a host:port string") 
         os.Exit(100) 
   } 
   CONNECT := arguments[1] 
```

`UDPclient.go`的第三部分如下：

```go
   s, err := net.ResolveUDPAddr("udp", CONNECT) 
   c, err := net.DialUDP("udp", nil, s) 

   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   fmt.Printf("The UDP server is %s\n", c.RemoteAddr().String()) 
   defer c.Close() 
```

这里没有什么特别的：只是使用`net.ResolveUDPAddr()`和`net.DialUDP()`来连接到 UDP 服务器。

UDP 客户端的第四部分如下：

```go
   data := []byte("Hello UDP Echo server!\n") 
   _, err = c.Write(data) 

   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

这次，您将使用`Write()`将数据发送到 UDP 服务器，尽管您将使用`ReadFromUDP()`从 UDP 服务器读取数据。

`UDPclient.go`的最后一部分如下：

```go
   buffer := make([]byte, 1024) 
   n, _, err := c.ReadFromUDP(buffer) 
   fmt.Print("Reply: ", string(buffer[:n])) 
} 
```

由于我们有`UDPserver.go`并且知道它可以工作，我们可以使用`UDPserver.go`来测试`UDPclient.go`的操作：

```go
$ go run UDPclient.go localhost:1234
The UDP server is 127.0.0.1:1234
Reply: Hello UDP Echo server!
```

如果您在没有 UDP 服务器监听所需端口的情况下执行`UDPclient.go`，您将获得以下输出，其中并未明确说明它无法连接到 UDP 服务器：它只显示了一个空回复：

```go
$ go run UDPclient.go localhost:1024
The UDP server is 127.0.0.1:1024
Reply:
```

# 一个并发的 TCP 服务器

在本节中，您将学习如何开发一个并发的 TCP 服务器：每个客户端连接将被分配给一个新的 goroutine 来为客户端请求提供服务。请注意，尽管 TCP 客户端最初连接到相同的端口，但它们使用的端口号与服务器的主端口号不同：这是由 TCP 自动处理的，也是 TCP 的工作方式。

虽然创建一个并发的 UDP 服务器也是可能的，但由于 UDP 的工作方式，这可能并不是绝对必要的。但是，如果您有一个非常繁忙的 UDP 服务，那么您可能需要考虑开发一个并发的 UDP 服务器。

程序的名称将是`concTCP.go`，并将分为五个部分呈现。好处是，一旦您定义了一个处理传入连接的函数，您所需要做的就是将该函数作为 goroutine 执行，其余的工作将由 Go 处理！

`concTCP.go`的第一部分如下：

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
```

并发 TCP 服务器的第二部分如下：

```go
func handleConnection(c net.Conn) { 
   for { 
         netData, err := bufio.NewReader(c).ReadString('\n') 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 

         fmt.Print("-> ", string(netData)) 
         c.Write([]byte(netData)) 
         if strings.TrimSpace(string(netData)) == "STOP" { 
               break 
         } 
   } 
   time.Sleep(3 * time.Second) 
   c.Close() 
} 
```

这是处理每个 TCP 请求的函数的实现。最后的时间延迟用于给您足够的时间与另一个 TCP 客户端连接并证明`concTCP.go`可以为多个 TCP 客户端提供服务。

程序的第三部分包含以下 Go 代码：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a port number!") 
         os.Exit(100) 
   } 

   PORT := ":" + arguments[1] 
```

`concTCP.go`的第四部分包含以下 Go 代码：

```go
   l, err := net.Listen("tcp", PORT) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
   defer l.Close() 
```

到目前为止，`main()`函数中没有什么特别的，因为尽管`concTCP.go`将处理多个请求，但它只需要一次调用`net.Listen()`。

最后一部分 Go 代码如下：

```go
   for { 
         c, err := l.Accept() 
         if err != nil { 
               fmt.Println(err) 
               os.Exit(100) 
         } 
         go handleConnection(c) 
   } 
} 
```

`concTCP.go`处理其请求的所有差异都可以在 Go 代码的最后几行找到。每次程序使用`Accept()`接受新的网络请求时，都会启动一个新的 goroutine，并且`concTCP.go`立即准备好接受更多的请求。请注意，为了终止`concTCP.go`，您将需要按下*Ctrl* + *C*，因为`STOP`关键字用于终止程序的每个 goroutine。

执行`concTCP.go`并使用各种 TCP 客户端连接到它，将生成以下输出：

```go
$ go run concTCP.go 1234
-> Hi!
-> Hello!
-> STOP
...
```

# 远程过程调用（RPC）

**远程过程调用**（**RPC**）是一种用于进程间通信的客户端-服务器机制。请注意，RPC 客户端和 RPC 服务器使用 TCP/IP 进行通信，这意味着它们可以存在于不同的机器上。

为了开发 RPC 客户端或 RPC 服务器的实现，您需要按照一定的步骤调用一些函数。这两种实现都不难；您只需要遵循一定的步骤。

此外，请访问`https://golang.org/pkg/net/rpc/`上可以找到的`net/rpc` Go 标准包的文档页面。

请注意，所呈现的 RPC 示例将使用 TCP 进行客户端-服务器交互。但是，您也可以使用 HTTP 进行客户端-服务器通信。

# 一个 RPC 服务器

本小节将介绍一个名为`RPCserver.go`的 RPC 服务器。正如您将在`RPCserver.go`程序的前言中看到的那样，RPC 服务器导入了一个名为`sharedRPC`的包，该包在`sharedRPC.go`文件中实现：包的名称是任意的。其内容如下：

```go
package sharedRPC 

type MyInts struct { 
   A1, A2 uint 
   S1, S2 bool 
} 

type MyInterface interface {

   Add(arguments *MyInts, reply *int) error 
   Subtract(arguments *MyInts, reply *int) error 
} 
```

因此，在这里，您定义了一个新的结构，其中包含两个无符号整数的符号和值，并定义了一个名为`MyInterface`的新接口。

然后，您应该安装`sharedRPC.go`，这意味着您应该在尝试在程序中使用`sharedRPC`包之前执行以下命令：

```go
$ mkdir ~/go
$ mkdir ~/go/src
$ mkdir ~/go/src/sharedRPC
$ export GOPATH=~/go
$ vi ~/go/src/sharedRPC/sharedRPC.go
$ go install sharedRPC
```

如果您使用的是 macOS 机器（`darwin_amd64`）并且希望确保一切正常，您可以执行以下两个命令：

```go
$ cd ~/go/pkg/darwin_amd64/
$ ls -l sharedRPC.a
-rw-r--r--  1 mtsouk  staff  4698 Jul 27 11:49 sharedRPC.a
```

您真正需要记住的是，归根结底，RPC 服务器和 RPC 客户端之间交换的是函数名称及其参数。只有在`sharedRPC.go`接口中定义的函数才能在 RPC 交互中使用：RPC 服务器将需要实现`MyInterface`接口的函数。`RPCserver.go`的 Go 代码将分为五部分呈现；RPC 服务器的第一部分具有预期的前言，其中还包括我们制作的`sharedRPC`包：

```go
package main 

import ( 
   "fmt" 
   "net" 
   "net/rpc" 
   "os" 
   "sharedRPC" 
) 
```

`RPCserver.go`的第二部分如下：

```go
type MyInterface int 

func (t *MyInterface) Add(arguments *sharedRPC.MyInts, reply *int) error { 
   s1 := 1 
   s2 := 1 

   if arguments.S1 == true { 
         s1 = -1 
   } 

   if arguments.S2 == true { 
         s2 = -1 
   } 

   *reply = s1*int(arguments.A1) + s2*int(arguments.A2) 
   return nil 
} 
```

这是将要提供给 RPC 客户端的第一个函数的实现：您可以拥有尽可能多的函数，只要它们包含在接口中。

`RPCserver.go`的第三部分包含以下 Go 代码：

```go
func (t *MyInterface) Subtract(arguments *sharedRPC.MyInts, reply *int) error { 
   s1 := 1 
   s2 := 1 

   if arguments.S1 == true { 
         s1 = -1 
   } 

   if arguments.S2 == true { 
         s2 = -1 
   } 

   *reply = s1*int(arguments.A1) - s2*int(arguments.A2) 
   return nil 
} 
```

这是 RPC 服务器向 RPC 客户端提供的第二个函数。

`RPCserver.go`的第四部分包含以下 Go 代码：

```go
func main() { 
   PORT := ":1234" 

   myInterface := new(MyInterface) 
   rpc.Register(myInterface) 

   t, err := net.ResolveTCPAddr("tcp", PORT) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
   l, err := net.ListenTCP("tcp", t) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
```

由于我们的 RPC 服务器使用 TCP，您需要调用`net.ResolveTCPAddr()`和`net.ListenTCP()`来进行调用。但是，您首先需要调用`rpc.Register()`以便能够提供所需的接口。

程序的最后部分如下：

```go
   for { 
         c, err := l.Accept() 
         if err != nil { 
               continue 
         } 
         rpc.ServeConn(c) 
   } 
} 
```

在这里，您可以像往常一样使用`Accept()`接受新的 TCP 连接，但是使用`rpc.ServeConn()`来提供服务。

您将需要等待下一节和 RPC 客户端的开发，以便测试`RPCserver.go`的操作。

# 一个 RPC 客户端

在本节中，我们将开发一个名为`RPCclient.go`的 RPC 客户端。`RPCclient.go`的 Go 代码将分为五部分呈现；第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "net/rpc" 
   "os" 
   "sharedRPC" 
) 
```

请注意 RPC 客户端中`sharedRPC`包的使用。

`RPCclient.go`的第二部分如下：

```go
func main() { 
   arguments := os.Args 
   if len(arguments) == 1 { 
         fmt.Println("Please provide a host:port string!") 
         os.Exit(100) 
   } 

   CONNECT := arguments[1] 
```

程序的第三部分包含以下 Go 代码：

```go
   c, err := rpc.Dial("tcp", CONNECT) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 

   args := sharedRPC.MyInts{17, 18, true, false} 
   var reply int 
```

由于`MyInts`结构在`sharedRPC.go`中定义，因此您需要在 RPC 客户端中将其用作`sharedRPC.MyInts`。此外，您调用`rpc.Dial()`来连接到 RPC 服务器，而不是`net.Dial()`。

RPC 客户端的第四部分包含以下 Go 代码：

```go
   err = c.Call("MyInterface.Add", args, &reply) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
   fmt.Printf("Reply (Add): %d\n", reply) 
```

在这里，您使用`Call()`函数来执行 RPC 服务器中的所需函数。`MyInterface.Add()`函数的结果存储在先前声明的`reply`变量中。

`RPCclient.go`的最后部分如下：

```go
   err = c.Call("MyInterface.Subtract", args, &reply) 
   if err != nil { 
         fmt.Println(err) 
         os.Exit(100) 
   } 
   fmt.Printf("Reply (Subtract): %d\n", reply) 
} 
```

在这里，您执行`MyInterface.Subtract()`函数的方式与之前相同。

正如您可以猜到的，您无法在没有 RCP 服务器的情况下测试 RPC 客户端，反之亦然：`netcat(1)`不能用于 RPC。

首先，您需要启动`RPCserver.go`进程：

```go
$ go run RPCserver.go
```

然后，您将执行`RPCclient.go`程序：

```go
$ go run RPCclient.go localhost:1234
Reply (Add): 1
Reply (Subtrack): -35
```

如果`RPCserver.go`进程没有运行，而您尝试执行`RPCclient.go`，您将收到以下错误消息：

```go
$ go run RPCclient.go localhost:1234
dial tcp [::1]:1234: getsockopt: connection refused
exit status 100
```

当然，RPC 不是用于添加整数或自然数，而是用于执行更复杂的操作，您希望从一个中心点进行控制。

# 练习

1.  阅读 net 包的文档，以了解其可用函数列表：[`golang.org/pkg/net/`](https://golang.org/pkg/net/)。

1.  Wireshark 是分析任何类型网络流量的好工具：尝试更多地使用它。

1.  修改`socketClient.go`的代码，以便从用户那里读取输入。

1.  修改`socketServer.go`的代码，以便向客户端返回一个随机数。

1.  修改`TCPserver.go`的代码，以便在接收到用户给定的 Unix 信号时停止。

1.  修改`concTCP.go`的 Go 代码，以便跟踪它服务过的客户端数量，并在退出之前打印该数字。

1.  向`RPCserver.go`添加一个`quit()`函数，执行其名称所暗示的操作。

1.  开发您自己的 RPC 示例。

# 总结

在本章中，我们向您介绍了 TCP/IP，并讨论了如何在 Go 中开发 TCP 和 UDP 服务器和客户端，以及创建 RPC 客户端和服务器。

在这一点上，没有下一章，因为这是本书的最后一章！恭喜您阅读了整本书！您现在已经准备好开始在 Go 中开发有用的 Unix 命令行实用程序了；所以，继续并立即开始编程您自己的工具！
