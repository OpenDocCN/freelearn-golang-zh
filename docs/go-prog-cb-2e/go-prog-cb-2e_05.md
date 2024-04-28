# 第五章：网络编程

Go 标准库为网络操作提供了大量支持。它包括允许您使用 TCP/IP、UDP、DNS、邮件和使用 HTTP 的 RPC 的包。第三方包也可以填补标准库中包含的内容的空白，包括`gorilla/websockets` ([`github.com/gorilla/websocket/`](https://github.com/gorilla/websocket/))，用于 WebSocket 实现，可以在普通的 HTTP 处理程序中使用。本章探讨了这些库，并演示了一些简单的用法。这些用法将帮助那些无法使用更高级的抽象，如 REST 或 GRPC，但需要网络连接的开发人员。它对需要执行 DNS 查找或处理原始电子邮件的 DevOps 应用程序也很有用。阅读完本章后，您应该已经掌握了基本的网络编程，并准备深入学习。

在本章中，将涵盖以下用法：

+   编写 TCP/IP 回显服务器和客户端

+   编写 UDP 服务器和客户端

+   处理域名解析

+   使用 WebSockets

+   使用 net/rpc 调用远程方法

+   使用 net/mail 解析电子邮件

# 技术要求

为了继续本章中的所有用法，请按照以下步骤配置您的环境：

1.  从[`golang.org/doc/install`](https://golang.org/doc/install)下载并安装 Go 1.12.6 或更高版本到您的操作系统上。

1.  打开终端或控制台应用程序，然后创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。所有代码都将从这个目录运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，您可以选择从该目录工作，而不是手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 编写 TCP/IP 回显服务器和客户端

TCP/IP 是一种常见的网络协议，HTTP 协议是在其上构建的。TCP 要求客户端连接到服务器以发送和接收数据。这个用法将使用`net`包在客户端和服务器之间建立 TCP 连接。客户端将把用户输入发送到服务器，服务器将用`strings.ToUpper()`的结果将输入的相同字符串转换为大写形式进行响应。客户端将打印从服务器接收到的任何消息，因此它应该输出我们输入的大写版本。

# 如何做...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter5/tcp`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/tcp 
```

您应该会看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/tcp    
```

1.  从`~/projects/go-programming-cookbook-original/chapter5/tcp`复制测试，或者使用这个作为练习编写一些您自己的代码！

1.  创建一个名为`server`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "bufio"
  "fmt"
  "net"
  "strings"
)

const addr = "localhost:8888"

func echoBackCapitalized(conn net.Conn) {
  // set up a reader on conn (an io.Reader)
  reader := bufio.NewReader(conn)

  // grab the first line of data encountered
  data, err := reader.ReadString('\n')
  if err != nil {
    fmt.Printf("error reading data: %s\n", err.Error())
    return
  }
  // print then send back the data
  fmt.Printf("Received: %s", data)
  conn.Write([]byte(strings.ToUpper(data)))
  // close up the finished connection
  conn.Close()
}

func main() {
  ln, err := net.Listen("tcp", addr)
  if err != nil {
    panic(err)
  }
  defer ln.Close()
  fmt.Printf("listening on: %s\n", addr)
  for {
    conn, err := ln.Accept()
    if err != nil {
      fmt.Printf("encountered an error accepting connection: %s\n", 
                  err.Error())
      // if there's an error try again
      continue
    }
    // handle this asynchronously
    // potentially a good use-case
    // for a worker pool
    go echoBackCapitalized(conn)
  }
}
```

1.  导航到上一个目录。

1.  创建一个名为`client`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "bufio"
  "fmt"
  "net"
  "os"
)

const addr = "localhost:8888"

func main() {
  reader := bufio.NewReader(os.Stdin)
  for {
    // grab a string input from the clie
    fmt.Printf("Enter some text: ")
    data, err := reader.ReadString('\n')
    if err != nil {
      fmt.Printf("encountered an error reading input: %s\n", 
                  err.Error())
      continue
    }
    // connect to the addr
    conn, err := net.Dial("tcp", addr)
    if err != nil {
      fmt.Printf("encountered an error connecting: %s\n", 
                  err.Error())
    }

    // write the data to the connection
    fmt.Fprintf(conn, data)

    // read back the response
    status, err := bufio.NewReader(conn).ReadString('\n')
    if err != nil {
      fmt.Printf("encountered an error reading response: %s\n", 
                  err.Error())
    }
    fmt.Printf("Received back: %s", status)
    // close up the finished connection
    conn.Close()
  }
}
```

1.  导航到上一个目录。

1.  运行`go run ./server`，您将看到以下输出：

```go
$ go run ./server
listening on: localhost:8888
```

1.  在另一个终端中，从`tcp`目录运行`go run ./client`，您将看到以下输出：

```go
$ go run ./client 
Enter some text:
```

1.  输入`this is a test`并按*Enter*。您将看到以下内容：

```go
$ go run ./client 
Enter some text: this is a test
Received back: THIS IS A TEST
Enter some text: 
```

1.  按下*Ctrl* + *C*退出。

1.  如果您复制或编写了自己的测试，请返回上一个目录并运行`go test`。确保所有测试都通过。

# 工作原理...

服务器正在侦听端口`8888`。每当有请求时，服务器必须接收请求并管理客户端连接。在这个程序的情况下，它会派发一个 Goroutine 来从客户端读取请求，将接收到的数据大写，发送回客户端，最后关闭连接。服务器立即再次循环，等待接收新的客户端连接，同时处理先前的连接。

客户端从`STDIN`读取输入，通过 TCP 连接到地址，写入从输入中读取的消息，然后打印服务器的响应。之后，它关闭连接并再次从`STDIN`循环读取。您也可以重新设计此示例，使客户端保持连接，直到程序退出，而不是在每个请求上。

# 编写 UDP 服务器和客户端

UDP 协议通常用于游戏和速度比可靠性更重要的地方。UDP 服务器和客户端不需要相互连接。这个示例将创建一个 UDP 服务器，它将监听来自客户端的消息，将它们的 IP 添加到其列表中，并向先前看到的每个客户端广播消息。

每当客户端连接时，服务器将向`STDOUT`写入一条消息，并将相同的消息广播给所有客户端。这条消息的文本应该是`Sent <count>`，其中`<count>`将在服务器向所有客户端广播时递增。因此，`count`的值可能会有所不同，这取决于您连接到客户端所需的时间，因为服务器将无论发送消息给多少客户端都会广播。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter5/udp`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/udp 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/udp    
```

1.  从`~/projects/go-programming-cookbook-original/chapter5/udp`复制测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`server`的新目录，并导航到该目录。

1.  创建一个名为`broadcast.go`的文件，内容如下：

```go
package main

import (
  "fmt"
  "net"
  "sync"
  "time"
)

type connections struct {
  addrs map[string]*net.UDPAddr
  // lock for modifying the map
  mu sync.Mutex
}

func broadcast(conn *net.UDPConn, conns *connections) {
  count := 0
  for {
    count++
    conns.mu.Lock()
    // loop over known addresses
    for _, retAddr := range conns.addrs {

      // send a message to them all
      msg := fmt.Sprintf("Sent %d", count)
      if _, err := conn.WriteToUDP([]byte(msg), retAddr); err != nil {
        fmt.Printf("error encountered: %s", err.Error())
        continue
      }

    }
    conns.mu.Unlock()
    time.Sleep(1 * time.Second)
  }
}
```

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "fmt"
  "net"
)

const addr = "localhost:8888"

func main() {
  conns := &connections{
    addrs: make(map[string]*net.UDPAddr),
  }

  fmt.Printf("serving on %s\n", addr)

  // construct a udp addr
  addr, err := net.ResolveUDPAddr("udp", addr)
  if err != nil {
    panic(err)
  }

  // listen on our specified addr
  conn, err := net.ListenUDP("udp", addr)
  if err != nil {
    panic(err)
  }
  // cleanup
  defer conn.Close()

  // async send messages to all known clients
  go broadcast(conn, conns)

  msg := make([]byte, 1024)
  for {
    // receive a message to gather the ip address
    // and port to send back to
    _, retAddr, err := conn.ReadFromUDP(msg)
    if err != nil {
      continue
    }

    //store it in a map
    conns.mu.Lock()
    conns.addrs[retAddr.String()] = retAddr
    conns.mu.Unlock()
    fmt.Printf("%s connected\n", retAddr)
  }
}
```

1.  导航到上一个目录。

1.  创建一个名为`client`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "fmt"
  "net"
)

const addr = "localhost:8888"

func main() {
  fmt.Printf("client for server url: %s\n", addr)

  addr, err := net.ResolveUDPAddr("udp", addr)
  if err != nil {
    panic(err)
  }

  conn, err := net.DialUDP("udp", nil, addr)
  if err != nil {
    panic(err)
  }
  defer conn.Close()

  msg := make([]byte, 512)
  n, err := conn.Write([]byte("connected"))
  if err != nil {
    panic(err)
  }
  for {
    n, err = conn.Read(msg)
    if err != nil {
      continue
    }
    fmt.Printf("%s\n", string(msg[:n]))
  }
}
```

1.  导航到上一个目录。

1.  运行`go run ./server`，您将看到以下输出：

```go
$ go run ./server
serving on localhost:8888
```

1.  在另一个终端中，从`udp`目录运行`go run ./client`，您将看到以下输出，尽管计数可能有所不同：

```go
$ go run ./client 
client for server url: localhost:8888
Sent 3
Sent 4
Sent 5
```

1.  导航到运行服务器的终端，您应该看到类似以下的内容：

```go
$ go run ./server 
serving on localhost:8888
127.0.0.1:64242 connected
```

1.  按下*Ctrl* + *C*退出服务器和客户端。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

服务器正在监听端口`8888`，就像在上一个示例中一样。如果客户端启动，它会向服务器发送一条消息，服务器会将其地址添加到地址列表中。因为客户端可以异步连接，所以服务器在修改或读取列表之前必须使用互斥锁。

一个单独的广播 Goroutine 独立运行，并将相同的消息发送到以前发送消息的所有客户端地址。假设它们仍在监听，它们将在大致相同的时间从服务器接收相同的消息。您还可以连接更多的客户端来查看这种效果。

# 使用域名解析

`net`包提供了许多有用的 DNS 查找功能。这些信息与使用 Unix 的`dig`命令获得的信息相似。这些信息对于您实现任何需要动态确定 IP 地址的网络编程非常有用。

本教程将探讨如何收集这些数据。为了演示这一点，我们将实现一个简化的`dig`命令。我们将寻求将 URL 映射到其所有 IPv4 和 IPv6 地址。通过修改`GODEBUG=netdns=`为`go`或`cgo`，它将使用纯 Go DNS 解析器或`cgo`解析器。默认情况下，使用纯 Go DNS 解析器。

# 如何做...

以下步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter5/dns`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/dns 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/dns
```

1.  从`~/projects/go-programming-cookbook-original/chapter5/dns`复制测试，或者使用此作为练习来编写一些您自己的代码！

1.  创建一个名为`dns.go`的文件，内容如下：

```go
package dns

import (
  "fmt"
  "net"

  "github.com/pkg/errors"
)

// Lookup holds the DNS information we care about
type Lookup struct {
  cname string
  hosts []string
}

// We can use this to print the lookup object
func (d *Lookup) String() string {
  result := ""
  for _, host := range d.hosts {
    result += fmt.Sprintf("%s IN A %s\n", d.cname, host)
  }
  return result
}

// LookupAddress returns a DNSLookup consisting of a cname and host
// for a given address
func LookupAddress(address string) (*Lookup, error) {
  cname, err := net.LookupCNAME(address)
  if err != nil {
    return nil, errors.Wrap(err, "error looking up CNAME")
  }
  hosts, err := net.LookupHost(address)
  if err != nil {
    return nil, errors.Wrap(err, "error looking up HOST")
  }

  return &Lookup{cname: cname, hosts: hosts}, nil
}
```

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "fmt"
  "log"
  "os"

  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/dns"
)

func main() {
  if len(os.Args) < 2 {
    fmt.Printf("Usage: %s <address>\n", os.Args[0])
    os.Exit(1)
  }
  address := os.Args[1]
  lookup, err := dns.LookupAddress(address)
  if err != nil {
    log.Panicf("failed to lookup: %s", err.Error())
  }
  fmt.Println(lookup)
}
```

1.  运行`go run main.go golang.org`命令。

1.  您还可以运行以下命令：

```go
$ go build $ ./example golang.org
```

您应该看到以下输出：

```go
$ go run main.go golang.org
golang.org. IN A 172.217.5.17
golang.org. IN A 2607:f8b0:4009:809::2011
```

1.  `go.mod`文件可能已更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

本教程执行了提供的地址的`CNAME`和主机查找。在我们的案例中，我们使用了`golang.org`。我们将结果存储在一个查找结构中，该结构使用`String()`方法打印输出结果。当我们将对象打印为字符串时，将自动调用此方法，或者我们可以直接调用该方法。我们在`main.go`中实现了一些基本的参数检查，以确保在运行程序时提供了地址。

# 使用 WebSockets

WebSockets 允许服务器应用程序连接到用 JavaScript 编写的基于 Web 的客户端。这使您可以创建具有双向通信的 Web 应用程序，并创建更新，例如聊天室等。

本教程将探讨如何在 Go 中编写 WebSocket 服务器，并演示客户端使用 WebSocket 服务器的过程。它使用`github.com/gorilla/websocket`将标准处理程序升级为 WebSocket 处理程序，并创建客户端应用程序。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter5/websocket`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/websocket 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/websocket    
```

1.  从`~/projects/go-programming-cookbook-original/chapter5/websocket`复制测试，或者使用此作为练习来编写一些您自己的代码！

1.  创建一个名为`server`的新目录，并导航到该目录。

1.  创建一个名为`handler.go`的文件，内容如下：

```go
package main

import (
  "log"
  "net/http"

  "github.com/gorilla/websocket"
)

// upgrader takes an http connection and converts it
// to a websocket one, we're using some recommended
// basic buffer sizes
var upgrader = websocket.Upgrader{
  ReadBufferSize: 1024,
  WriteBufferSize: 1024,
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
  // upgrade the connection
  conn, err := upgrader.Upgrade(w, r, nil)
  if err != nil {
    log.Println("failed to upgrade connection: ", err)
    return
  }
  for {
    // read and echo back messages in a loop
    messageType, p, err := conn.ReadMessage()
    if err != nil {
      log.Println("failed to read message: ", err)
      return
    }
    log.Printf("received from client: %#v", string(p))
    if err := conn.WriteMessage(messageType, p); err != nil {
      log.Println("failed to write message: ", err)
      return
    }
  }
}
```

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "fmt"
  "log"
  "net/http"
)

func main() {
  fmt.Println("Listening on port :8000")
 // we mount our single handler on port localhost:8000 to handle all
  // requests
  log.Panic(http.ListenAndServe("localhost:8000", http.HandlerFunc(wsHandler)))
}
```

1.  导航到上一个目录。

1.  创建一个名为`client`的新目录，并导航到该目录。

1.  创建一个名为`process.go`的文件，内容如下：

```go
package main

import (
  "bufio"
  "fmt"
  "log"
  "os"
  "strings"

  "github.com/gorilla/websocket"
)

func process(c *websocket.Conn) {
  reader := bufio.NewReader(os.Stdin)
  for {
    fmt.Printf("Enter some text: ")
    // this will block ctrl-c, to exit press it then hit enter
    // or kill from another location
    data, err := reader.ReadString('\n')
    if err != nil {
      log.Println("failed to read stdin", err)
    }

    // trim off the space from reading the string
    data = strings.TrimSpace(data)

    // write the message as a byte across the websocket
    err = c.WriteMessage(websocket.TextMessage, []byte(data))
    if err != nil {
      log.Println("failed to write message:", err)
      return
    }

    // this is an echo server, so we can always read after the write
    _, message, err := c.ReadMessage()
    if err != nil {
      log.Println("failed to read:", err)
      return
    }
    log.Printf("received back from server: %#v\n", string(message))
  }
}
```

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "log"
  "os"
  "os/signal"

  "github.com/gorilla/websocket"
)

// catchSig cleans up our websocket conenction if we kill the program
// with a ctrl-c
func catchSig(ch chan os.Signal, c *websocket.Conn) {
  // block on waiting for a signal
  <-ch
  err := c.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))
  if err != nil {
    log.Println("write close:", err)
  }
  return
}

func main() {
  // connect the os signal to our channel
  interrupt := make(chan os.Signal, 1)
  signal.Notify(interrupt, os.Interrupt)

  // use the ws:// Scheme to connect to the websocket
  u := "ws://localhost:8000/"
  log.Printf("connecting to %s", u)

  c, _, err := websocket.DefaultDialer.Dial(u, nil)
  if err != nil {
    log.Fatal("dial:", err)
  }
  defer c.Close()

  // dispatch our signal catcher
  go catchSig(interrupt, c)

  process(c)
}
```

1.  导航到上一个目录。

1.  运行`go run ./server`，您将看到以下输出：

```go
$ go run ./server
Listening on port :8000
```

1.  在另一个终端中，从`websocket`目录运行`go run ./client`，您将看到以下输出：

```go
$ go run ./client
2019/05/26 11:53:20 connecting to ws://localhost:8000/
Enter some text: 
```

1.  输入`test`字符串，您应该看到以下内容：

```go
$ go run ./client
2019/05/26 11:53:20 connecting to ws://localhost:8000/
Enter some text: test
2019/05/26 11:53:22 received back from server: "test"
Enter some text: 
```

1.  导航到运行服务器的终端，您应该看到类似以下内容的内容：

```go
$ go run ./server
Listening on port :8000
2019/05/26 11:53:22 received from client: "test"
```

1.  按下*Ctrl* + *C*退出服务器和客户端。在按下*Ctrl* + *C*后，您可能还需要按*Enter*。

1.  `go.mod`文件可能已更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

服务器正在端口`8000`上监听 WebSocket 连接。当请求到来时，`github.com/gorilla/websocket`包用于将请求升级为 WebSocket 连接。与先前的回声服务器示例类似，服务器等待在 WebSocket 连接上接收消息，并将相同的消息作为响应发送回客户端。因为它是一个处理程序，所以它可以异步处理许多 WebSocket 连接，并且它们将保持连接，直到客户端终止。

在客户端中，我们添加了一个`catchsig`函数来处理*Ctrl* + *C*事件。这使我们能够在客户端退出时清楚地终止与服务器的连接。否则，客户端只是在`STDIN`上接受用户输入并将其发送到服务器，记录响应，然后重复。

# 使用 net/rpc 调用远程方法

Go 通过`net/rpc`包为您的系统提供基本的 RPC 功能。这是在不依赖于 GRPC 或其他更复杂的 RPC 包的情况下进行 RPC 调用的潜在替代方案。但是，它的功能相当有限，您可能希望导出的任何函数都必须符合非常特定的函数签名。

代码中的注释指出了一些可以远程调用的方法的限制。本配方演示了如何创建一个共享函数，该函数通过结构传递了许多参数，并且可以远程调用。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter5/rpc`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/rpc 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/rpc    
```

1.  从`~/projects/go-programming-cookbook-original/chapter5/rpc`复制测试，或者利用这个机会编写一些您自己的代码！

1.  创建一个名为`tweak`的新目录，并导航到该目录。

1.  创建一个名为`tweak.go`的文件，内容如下：

```go
package tweak

import (
  "strings"
)

// StringTweaker is a type of string
// that can reverse itself
type StringTweaker struct{}

// Args are a list of options for how to tweak
// the string
type Args struct {
  String string
  ToUpper bool
  Reverse bool
}

// Tweak conforms to the RPC library which require:
// - the method's type is exported.
// - the method is exported.
// - the method has two arguments, both exported (or builtin) types.
// - the method's second argument is a pointer.
// - the method has return type error.
func (s StringTweaker) Tweak(args *Args, resp *string) error {

  result := string(args.String)
  if args.ToUpper {
    result = strings.ToUpper(result)
  }
  if args.Reverse {
    runes := []rune(result)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
      runes[i], runes[j] = runes[j], runes[i]
    }
    result = string(runes)

  }
  *resp = result
  return nil
}
```

1.  导航到上一个目录。

1.  创建一个名为`server`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "fmt"
  "log"
  "net"
  "net/http"
  "net/rpc"

  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/rpc/tweak"
)

func main() {
  s := new(tweak.StringTweaker)
  if err := rpc.Register(s); err != nil {
    log.Fatal("failed to register:", err)
  }

  rpc.HandleHTTP()

  l, err := net.Listen("tcp", ":1234")
  if err != nil {
    log.Fatal("listen error:", err)
  }

  fmt.Println("listening on :1234")
  log.Panic(http.Serve(l, nil))
}
```

1.  导航到上一个目录。

1.  创建一个名为`client`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
  "fmt"
  "log"
  "net/rpc"

  "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/rpc/tweak"
)

func main() {
  client, err := rpc.DialHTTP("tcp", "localhost:1234")
  if err != nil {
    log.Fatal("error dialing:", err)
  }

  args := tweak.Args{
    String: "this string should be uppercase and reversed",
    ToUpper: true,
    Reverse: true,
  }
  var result string
  err = client.Call("StringTweaker.Tweak", args, &result)
  if err != nil {
    log.Fatal("client call with error:", err)
  }
  fmt.Printf("the result is: %s", result)
}
```

1.  导航到上一个目录。

1.  运行`go run ./server`，您将看到以下输出：

```go
$ go run ./server
Listening on :1234
```

1.  在单独的终端中，从`rpc`目录运行`go run ./client`，您将看到以下输出：

```go
$ go run ./client
the result is: DESREVER DNA ESACREPPU EB DLUOHS GNIRTS SIHT
```

1.  按*Ctrl* + *C*退出服务器。

1.  如果您复制或编写了自己的测试，请返回上一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`StringTweaker`结构被放入一个单独的库中，以便客户端（用于设置参数）和服务器（用于注册 RPC 和启动服务器）可以访问其导出类型。它还符合本配方开头提到的规则，以便与`net/rpc`一起使用。

`StringTweaker`可用于接受输入字符串，并根据传递的选项，可选地反转和大写其中包含的所有字符。这种模式可以扩展为创建更复杂的函数，并且您还可以使用额外的函数使代码在增长时更易读。

# 使用 net/mail 解析电子邮件

`net/mail`包提供了许多有用的函数，可在处理电子邮件时帮助您。如果您有电子邮件的原始文本，可以将其解析为提取标题、发送日期信息等。本配方将通过解析硬编码为字符串的原始电子邮件来演示其中的一些功能。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter5/mail`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/mail 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter5/mail
```

1.  从 `~/projects/go-programming-cookbook-original/chapter5/mail` 复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为 `header.go` 的文件，内容如下：

```go
package main

import (
  "fmt"
  "net/mail"
  "strings"
)

// extract header info and print it nicely
func printHeaderInfo(header mail.Header) {

  // this works because we know it's a single address
  // otherwise use ParseAddressList
  toAddress, err := mail.ParseAddress(header.Get("To"))
  if err == nil {
    fmt.Printf("To: %s <%s>\n", toAddress.Name, toAddress.Address)
  }
  fromAddress, err := mail.ParseAddress(header.Get("From"))
  if err == nil {
    fmt.Printf("From: %s <%s>\n", fromAddress.Name, 
                fromAddress.Address)
  }

  fmt.Println("Subject:", header.Get("Subject"))

  // this works for a valid RFC5322 date
  // it does a header.Get("Date"), then a
  // mail.ParseDate(that_result)
  if date, err := header.Date(); err == nil {
    fmt.Println("Date:", date)
  }

  fmt.Println(strings.Repeat("=", 40))
  fmt.Println()
}
```

1.  创建一个名为 `main.go` 的文件，内容如下：

```go
package main

import (
  "io"
  "log"
  "net/mail"
  "os"
  "strings"
)

// an example email message
const msg string = `Date: Thu, 24 Jul 2019 08:00:00 -0700
From: Aaron <fake_sender@example.com>
To: Reader <fake_receiver@example.com>
Subject: Gophercon 2019 is going to be awesome!

Feel free to share my book with others if you're attending.
This recipe can be used to process and parse email information.
`

func main() {
  r := strings.NewReader(msg)
  m, err := mail.ReadMessage(r)
  if err != nil {
    log.Fatal(err)
  }

  printHeaderInfo(m.Header)

  // after printing the header, dump the body to stdout
  if _, err := io.Copy(os.Stdout, m.Body); err != nil {
    log.Fatal(err)
  }
}
```

1.  运行 `go run .` 命令。

1.  您也可以运行以下内容：

```go
$ go build $ ./mail 
```

您应该看到以下输出：

```go
$ go run .
To: Reader <fake_receiver@example.com>
From: Aaron <fake_sender@example.com>
Subject: Gophercon 2019 is going to be awesome!
Date: 2019-07-24 08:00:00 -0700 -0700
========================================

Feel free to share my book with others if you're attending.
This recipe can be used to process and parse email information. 
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`printHeaderInfo` 函数为此示例大部分工作。它将地址从标题中解析为 `*mail.Address` 结构，并将日期标题解析为日期对象。然后，它将消息中的所有信息格式化为可读格式。主函数解析初始电子邮件并传递此标题。
