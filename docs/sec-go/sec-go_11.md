# 第十一章：主机发现和枚举

主机发现是查找网络上的主机的过程。如果你已经访问了私有网络上的一台机器，并且想要查看网络上的其他机器并开始收集网络的情况，这是很有用的。你也可以将整个互联网视为网络，并寻找特定类型的主机或者只是寻找任何主机。Ping 扫描和端口扫描是识别主机的常用技术。用于此目的的常用工具是 nmap。在本章中，我们将介绍使用 TCP 连接扫描和横幅抓取进行基本端口扫描，这是 nmap 的两种最常见用例。我们还将介绍可以用于手动交互和探索服务器端口的原始套接字连接。

枚举是一个类似的概念，但是指的是主动检查特定机器以获取尽可能多的信息。这包括扫描服务器的端口以查看哪个端口是开放的，获取横幅以检查服务，调用各种服务以获取版本号，并通常搜索攻击向量。

主机发现和枚举是有效渗透测试的关键步骤，因为如果你甚至不知道机器的存在，就无法利用它。例如，如果攻击者只知道如何使用`ping`命令查找主机，那么你可以通过简单地忽略 ping 请求来轻松地将所有主机隐藏起来，让攻击者无法找到。

主机发现和枚举需要与机器进行主动连接，这样你就会留下日志，可能触发警报，或者被注意到。有一些方法可以偷偷摸摸，比如只执行 TCP SYN 扫描，这样就不会建立完整的 TCP 连接，或者在连接时使用代理，这样不会隐藏你的存在，但会让它看起来好像你是从其他地方连接的。如果 IP 被阻止，使用代理隐藏你的 IP 可能是有用的，因为你可以简单地切换到新的代理。

本章还涵盖了模糊测试，尽管只是简要提及。模糊测试需要有自己的章节，事实上，已经有整本书专门讨论了这个主题。模糊测试在逆向工程或搜索漏洞时更有用，但也可以用于获取有关服务的信息。例如，一个服务可能不返回任何响应，让你对其用途一无所知，但如果你用错误的数据进行模糊测试，它返回一个错误，你可能会了解它期望接收的输入类型。

在本章中，我们将专门涵盖以下主题：

+   TCP 和 UDP 套接字

+   端口扫描

+   横幅抓取

+   TCP 代理

+   在网络上查找命名主机

+   模糊测试网络服务

# TCP 和 UDP 套接字

套接字是网络的构建模块。服务器使用套接字监听，客户端使用套接字拨号来绑定并共享信息。**Internet Protocol**（**IP**）层指定了机器的地址，但**Transmission Control Protocol**（**TCP**）或**User Datagram Protocol**（**UDP**）指定了机器上应该使用的端口。

两者之间的主要区别是连接状态。TCP 保持连接活动并验证消息是否已接收。UDP 只是发送消息而不从远程主机接收确认。

# 创建服务器

以下是一个示例服务器。如果要更改协议，可以将`net.Listen()`的`tcp`参数更改为`udp`：

```go
package main

import (
   "net"
   "fmt"
   "log"
)

var protocol = "tcp" // tcp or udp
var listenAddress = "localhost:3000"

func main() {
   listener, err := net.Listen(protocol, listenAddress)
   if err != nil {
      log.Fatal("Error creating listener. ", err)
   }
   log.Printf("Now listening for connections.")

   for {
      conn, err := listener.Accept()
      if err != nil {
         log.Println("Error accepting connection. ", err)
      }
      go handleConnection(conn)
   }
}

func handleConnection(conn net.Conn) {
   incomingMessageBuffer := make([]byte, 4096)

   numBytesRead, err := conn.Read(incomingMessageBuffer)
   if err != nil {
      log.Print("Error reading from client. ", err)
   }

   fmt.Fprintf(conn, "Thank you. I processed %d bytes.\n", 
      numBytesRead)
} 
```

# 创建客户端

这个示例创建了一个简单的网络客户端，可以与前面示例中的服务器一起工作。这个示例使用了 TCP，但是像`net.Listen()`一样，如果要切换协议，可以在`net.Dial()`中将`tcp`简单地替换为`udp`：

```go
package main

import (
   "net"
   "log"
)

var protocol = "tcp" // tcp or udp
var remoteHostAddress = "localhost:3000"

func main() {
   conn, err := net.Dial(protocol, remoteHostAddress)
   if err != nil {
      log.Fatal("Error creating listener. ", err)
   }
   conn.Write([]byte("Hello, server. Are you there?"))

   serverResponseBuffer := make([]byte, 4096)
   numBytesRead, err := conn.Read(serverResponseBuffer)
   if err != nil {
      log.Print("Error reading from server. ", err)
   }
   log.Println("Message recieved from server:")
   log.Printf("%s\n", serverResponseBuffer[0:numBytesRead])
} 
```

# 端口扫描

在找到网络上的主机之后，也许在进行 ping 扫描或监视网络流量之后，通常希望扫描端口并查看哪些端口是打开的并接受连接。通过查看哪些端口是打开的，您可以了解有关机器的很多信息。您可能能够确定它是 Windows 还是 Linux，或者它是否托管电子邮件服务器、Web 服务器、数据库服务器等。

有许多类型的端口扫描，但这个例子演示了最基本和直接的端口扫描示例，即 TCP 连接扫描。它像任何典型的客户端一样连接，并查看服务器是否接受请求。它不发送或接收任何数据，并立即断开连接，记录是否成功。

以下示例仅扫描本地主机，并将检查的端口限制为保留端口 0-1024。数据库服务器，如 MySQL，通常在较高的端口上侦听，例如`3306`，因此您将需要调整端口范围或使用常见端口的预定义列表。

每个 TCP 连接请求都在单独的 goroutine 中完成，因此它们都将并发运行，并且完成非常快。使用`net.DialTimeout()`函数，以便我们可以设置我们愿意等待的最长时间：

```go
package main

import (
   "strconv"
   "log"
   "net"
   "time"
)

var ipToScan = "127.0.0.1"
var minPort = 0
var maxPort = 1024

func main() {
   activeThreads := 0
   doneChannel := make(chan bool)

   for port := minPort; port <= maxPort ; port++ {
      go testTcpConnection(ipToScan, port, doneChannel)
      activeThreads++
   }

   // Wait for all threads to finish
   for activeThreads > 0 {
      <- doneChannel
      activeThreads--
   }
}

func testTcpConnection(ip string, port int, doneChannel chan bool) {
   _, err := net.DialTimeout("tcp", ip + ":" + strconv.Itoa(port), 
      time.Second*10)
   if err == nil {
      log.Printf("Port %d: Open\n", port)
   }
   doneChannel <- true
} 
```

# 从服务获取横幅

确定打开的端口后，您可以尝试从连接中读取并查看服务是否提供横幅或初始消息。

以下示例与前一个示例类似，但不仅连接和断开连接，而是连接并尝试从服务器读取初始消息。如果服务器提供任何数据，则打印出来，但如果服务器没有发送任何数据，则不会打印任何内容：

```go
package main

import (
   "strconv"
   "log"
   "net"
   "time"
)

var ipToScan = "127.0.0.1"

func main() {
   activeThreads := 0
   doneChannel := make(chan bool)

   for port := 0; port <= 1024 ; port++ {
      go grabBanner(ipToScan, port, doneChannel)
      activeThreads++
   }

   // Wait for all threads to finish
   for activeThreads > 0 {
      <- doneChannel
      activeThreads--
   }
}

func grabBanner(ip string, port int, doneChannel chan bool) {
   connection, err := net.DialTimeout(
      "tcp", 
      ip + ":"+strconv.Itoa(port),  
      time.Second*10)
   if err != nil {
      doneChannel<-true
      return
   }

   // See if server offers anything to read
   buffer := make([]byte, 4096)
   connection.SetReadDeadline(time.Now().Add(time.Second*5)) 
   // Set timeout
   numBytesRead, err := connection.Read(buffer)
   if err != nil {
      doneChannel<-true
      return
   }
   log.Printf("Banner from port %d\n%s\n", port,
      buffer[0:numBytesRead])

   doneChannel <- true
} 
```

# 创建 TCP 代理

与第九章中的 HTTP 代理类似，TCP 级别代理对于调试、记录、分析流量和保护隐私都很有用。在进行端口扫描、主机发现和枚举时，代理可以隐藏您的位置和源 IP 地址。您可能希望隐藏您的来源地，伪装您的身份，或者只是使用一次性 IP，以防因执行请求而被列入黑名单。

以下示例将监听本地端口，将请求转发到远程主机，然后将远程服务器的响应发送回客户端。它还会记录任何请求。

您可以通过在上一节中运行服务器，然后设置代理以转发到该服务器来测试此代理。当回显服务器和代理服务器正在运行时，使用 TCP 客户端连接到代理服务器：

```go
package main

import (
   "net"
   "log"
)

var localListenAddress = "localhost:9999"
var remoteHostAddress = "localhost:3000" // Not required to be remote

func main() {
   listener, err := net.Listen("tcp", localListenAddress)
   if err != nil {
      log.Fatal("Error creating listener. ", err)
   }

   for {
      conn, err := listener.Accept()
      if err != nil {
         log.Println("Error accepting connection. ", err)
      }
      go handleConnection(conn)
   }
}

// Forward the request to the remote host and pass response 
// back to client
func handleConnection(localConn net.Conn) {
   // Create remote connection that will receive forwarded data
   remoteConn, err := net.Dial("tcp", remoteHostAddress)
   if err != nil {
      log.Fatal("Error creating listener. ", err)
   }
   defer remoteConn.Close()

   // Read from the client and forward to remote host
   buf := make([]byte, 4096) // 4k buffer
   numBytesRead, err := localConn.Read(buf)
   if err != nil {
      log.Println("Error reading from client.", err)
   }
   log.Printf(
      "Forwarding from %s to %s:\n%s\n\n",
      localConn.LocalAddr(),
      remoteConn.RemoteAddr(),
      buf[0:numBytesRead],
   )
   _, err = remoteConn.Write(buf[0:numBytesRead])
   if err != nil {
      log.Println("Error writing to remote host. ", err)
   }

   // Read response from remote host and pass it back to our client
   buf = make([]byte, 4096)
   numBytesRead, err = remoteConn.Read(buf)
   if err != nil {
      log.Println("Error reading from remote host. ", err)
   }
   log.Printf(
      "Passing response back from %s to %s:\n%s\n\n",
      remoteConn.RemoteAddr(),
      localConn.LocalAddr(),
      buf[0:numBytesRead],
   )
   _, err = localConn.Write(buf[0:numBytesRead])
   if err != nil {
      log.Println("Error writing back to client.", err)
   }
}
```

# 在网络上查找命名主机

如果您刚刚获得对网络的访问权限，您可以做的第一件事之一是了解网络上有哪些主机。您可以扫描子网上的所有 IP 地址，然后进行 DNS 查找，看看是否可以找到任何命名主机。主机名可以具有描述性或信息性的名称，可以提供有关服务器可能正在运行的内容的线索。

纯 Go 解析器是默认的，只能阻塞一个 goroutine 而不是系统线程，这样更有效率一些。您可以使用环境变量显式设置 DNS 解析器：

```go
export GODEBUG=netdns=go    # Use pure Go resolver (default)
export GODEBUG=netdns=cgo   # Use cgo resolver
```

这个例子寻找子网上的每个可能的主机，并尝试为每个 IP 解析主机名：

```go
package main

import (
   "strconv"
   "log"
   "net"
   "strings"
)

var subnetToScan = "192.168.0" // First three octets

func main() {
   activeThreads := 0
   doneChannel := make(chan bool)

   for ip := 0; ip <= 255; ip++ {
      fullIp := subnetToScan + "." + strconv.Itoa(ip)
      go resolve(fullIp, doneChannel)
      activeThreads++
   }

   // Wait for all threads to finish
   for activeThreads > 0 {
      <- doneChannel
      activeThreads--
   }
}

func resolve(ip string, doneChannel chan bool) {
   addresses, err := net.LookupAddr(ip)
   if err == nil {
      log.Printf("%s - %s\n", ip, strings.Join(addresses, ", "))
   }
   doneChannel <- true
} 
```

# 对网络服务进行模糊测试

模糊测试是指向应用程序发送故意格式不正确、过多或随机的数据，以使其行为异常、崩溃或泄露敏感信息。您可以识别缓冲区溢出漏洞，这可能导致远程代码执行。如果在向应用程序发送特定大小的数据后导致其崩溃或停止响应，可能是由于缓冲区溢出引起的。

有时，你可能会因为使服务使用过多内存或占用所有处理能力而导致拒绝服务。正则表达式因其速度慢而臭名昭著，并且可以在 Web 应用程序的 URL 路由机制中被滥用，用少量请求就可以消耗所有 CPU。

非随机但格式错误的数据可能同样危险，甚至更危险。一个正确格式错误的视频文件可能会导致 VLC 崩溃并暴露代码执行。一个正确格式错误的数据包，只需改变 1 个字节，就可能导致敏感数据暴露，就像 Heartbleed OpenSSL 漏洞一样。

以下示例将演示一个非常基本的 TCP 模糊器。它向服务器发送逐渐增加长度的随机字节。它从 1 字节开始，按 2 的幂指数级增长。首先发送 1 字节，然后是 2、4、8、16，一直持续到返回错误或达到最大配置限制。

调整`maxFuzzBytes`以设置要发送到服务的数据的最大大小。请注意，它会同时启动所有线程，所以要小心服务器的负载。寻找响应中的异常或服务器的崩溃。

```go
package main

import (
   "crypto/rand"
   "log"
   "net"
   "strconv"
   "time"
)

var ipToScan = "www.devdungeon.com"
var port = 80
var maxFuzzBytes = 1024

func main() {
   activeThreads := 0
   doneChannel := make(chan bool)

   for fuzzSize := 1; fuzzSize <= maxFuzzBytes; 
      fuzzSize = fuzzSize * 2 {
      go fuzz(ipToScan, port, fuzzSize, doneChannel)
      activeThreads++
   }

   // Wait for all threads to finish
   for activeThreads > 0 {
      <- doneChannel
      activeThreads--
   }
}

func fuzz(ip string, port int, fuzzSize int, doneChannel chan bool) {
   log.Printf("Fuzzing %d.\n", fuzzSize)

   conn, err := net.DialTimeout("tcp", ip + ":" + strconv.Itoa(port), 
      time.Second*10)
   if err != nil {
      log.Printf(
         "Fuzz of %d attempted. Could not connect to server. %s\n", 
         fuzzSize, 
         err,
      )
      doneChannel <- true
      return
   }

   // Write random bytes to server
   randomBytes := make([]byte, fuzzSize)
   rand.Read(randomBytes)
   conn.SetWriteDeadline(time.Now().Add(time.Second * 5))
   numBytesWritten, err := conn.Write(randomBytes)
   if err != nil { // Error writing
      log.Printf(
         "Fuzz of %d attempted. Could not write to server. %s\n", 
         fuzzSize,
         err,
      )
      doneChannel <- true
      return
   }
   if numBytesWritten != fuzzSize {
      log.Printf("Unable to write the full %d bytes.\n", fuzzSize)
   }
   log.Printf("Sent %d bytes:\n%s\n\n", numBytesWritten, randomBytes)

   // Read up to 4k back
   readBuffer := make([]byte, 4096)
   conn.SetReadDeadline(time.Now().Add(time.Second *5))
   numBytesRead, err := conn.Read(readBuffer)
   if err != nil { // Error reading
      log.Printf(
         "Fuzz of %d attempted. Could not read from server. %s\n", 
         fuzzSize,
         err,
      )
      doneChannel <- true
      return
   }

   log.Printf(
      "Sent %d bytes to server. Read %d bytes back:\n,
      fuzzSize,
      numBytesRead, 
   )
   log.Printf(
      "Data:\n%s\n\n",
      readBuffer[0:numBytesRead],
   )
   doneChannel <- true
} 
```

# 总结

阅读完本章后，你现在应该了解主机发现和枚举的基本概念。你应该能够在高层次上解释它们，并提供每个概念的基本示例。

首先，我们讨论了原始的 TCP 套接字，以一个简单的服务器和客户端为例。这些例子本身并不是非常有用，但它们是构建执行与服务的自定义交互的工具的模板。在尝试对未识别的服务进行指纹识别时，这将是有帮助的。

现在你应该知道如何运行一个简单的端口扫描，以及为什么你可能想要运行一个端口扫描。你应该了解如何使用 TCP 代理以及它提供了什么好处。你应该了解横幅抓取的工作原理以及为什么它是一种收集信息的有用方法。

还有许多其他形式的枚举。在 Web 应用程序中，你可以枚举用户名、用户 ID、电子邮件等。例如，如果一个网站使用 URL 格式[www.example.com/user_profile/1234](http://www.example.com/user_profile/1234)，你可以从数字 1 开始，逐渐增加 1，遍历网站上的每个用户资料。其他形式包括 SNMP、DNS、LDAP 和 SMB。

你还能想到哪些其他形式的枚举？如果你已经是一个权限较低的用户，你能想到什么样的枚举？一旦你拥有一个 shell，你会想收集关于服务器的什么样的信息？

一旦你在服务器上，你可以收集大量信息：用户名和组、主机名、网络设备信息、挂载的文件系统、正在运行的服务、iptables 设置、定时作业、启动服务等等。有关在已经访问到机器后该做什么的更多信息，请参阅第十三章，*后期利用*。

在下一章中，我们将讨论社会工程学以及如何通过 JSON REST API 从 Web 上收集情报，发送钓鱼邮件和生成 QR 码。我们还将看到多个蜜罐的例子，包括 TCP 蜜罐和两种 HTTP 蜜罐的方法。
