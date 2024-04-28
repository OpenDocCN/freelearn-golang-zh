# 安全外壳（SSH）

**安全外壳**（**SSH**）是一种用于在不安全网络上通信的加密网络协议。 SSH 最常见的用途是连接到远程服务器并与 shell 进行交互。文件传输也通过 SSH 协议上的 SCP 和 SFTP 进行。 SSH 是为了取代明文协议 Telnet 而创建的。 随着时间的推移，已经有了许多 RFC 来定义 SSH。 以下是部分列表，以便让您了解定义的内容。 由于它是如此常见和关键的协议，值得花时间了解细节。 以下是一些 RFC：

+   *RFC 4250* ([`tools.ietf.org/html/rfc4250`](https://tools.ietf.org/html/rfc4250)): *安全外壳（SSH）协议分配的数字*

+   *RFC 4251* ([`tools.ietf.org/html/rfc4251`](https://tools.ietf.org/html/rfc4251)): *安全外壳（SSH）协议架构*

+   *RFC 4252* ([`tools.ietf.org/html/rfc4252`](https://tools.ietf.org/html/rfc4252)): *安全外壳（SSH）认证协议*

+   *RFC 4253* ([`tools.ietf.org/html/rfc4253`](https://tools.ietf.org/html/rfc4253)): *安全外壳（SSH）传输层协议*

+   *RFC 4254* ([`tools.ietf.org/html/rfc4254`](https://tools.ietf.org/html/rfc4254)): *安全外壳（SSH）连接协议*

+   *RFC 4255* ([`tools.ietf.org/html/rfc4255`](https://tools.ietf.org/html/rfc4255)): *使用 DNS 安全发布安全外壳（SSH）密钥指纹*

+   *RFC 4256* ([`tools.ietf.org/html/rfc4256`](https://tools.ietf.org/html/rfc4256)): *安全外壳协议（SSH）的通用消息交换认证*

+   *RFC 4335* ([`tools.ietf.org/html/rfc4335`](https://tools.ietf.org/html/rfc4335)): **安全外壳（SSH）会话通道中断扩展**

+   *RFC 4344* ([`tools.ietf.org/html/rfc4344`](https://tools.ietf.org/html/rfc4344)): *安全外壳（SSH）传输层加密模式*

+   *RFC 4345* ([`tools.ietf.org/html/rfc4345`](https://tools.ietf.org/html/rfc4345)): *安全外壳（SSH）传输层协议的改进 Arcfour 模式*

稍后还对标准进行了额外的扩展，您可以在[`en.wikipedia.org/wiki/Secure_Shell#Standards_documentation`](https://en.wikipedia.org/wiki/Secure_Shell#Standards_documentation)上阅读相关内容。

SSH 是互联网上常见的暴力破解和默认凭据攻击目标。 因此，您可能考虑将 SSH 放在非标准端口上，但保持在系统端口（小于 1024）上，以便低特权用户在服务关闭时无法潜在地劫持端口。 如果将 SSH 保留在默认端口上，则诸如`fail2ban`之类的服务对于限制速率和阻止暴力破解攻击至关重要。 理想情况下，应完全禁用密码身份验证，并要求密钥身份验证。

SSH 包并不随标准库一起打包，尽管它是由 Go 团队编写的。 它正式是 Go 项目的一部分，但在主 Go 源树之外，因此默认情况下不会随 Go 一起安装。 它可以从[`golang.org/`](https://golang.org/)获取，并且可以使用以下命令进行安装：

```go
go get golang.org/x/crypto/ssh
```

在本章中，我们将介绍如何使用 SSH 客户端进行连接，执行命令和使用交互式 shell。 我们还将介绍使用密码或私钥等不同的身份验证方法。 SSH 包提供了用于创建服务器的函数，但本书中我们只涵盖客户端。

本章将专门涵盖 SSH 的以下内容：

+   使用密码进行身份验证

+   使用私钥进行身份验证

+   验证远程主机的密钥

+   通过 SSH 执行命令

+   启动交互式 shell

# 使用 Go SSH 客户端

`golang.org/x/crypto/ssh`包提供了一个与 SSH 版本 2 兼容的 SSH 客户端，这是最新版本。该客户端将与 OpenSSH 服务器以及遵循 SSH 规范的任何其他服务器一起工作。它支持传统的客户端功能，如子进程、端口转发和隧道。

# 身份验证方法

身份验证不仅是第一步，也是最关键的一步。不正确的身份验证可能导致机密性、完整性和可用性的潜在损失。如果未验证远程服务器，可能会发生中间人攻击，导致窃听、操纵或阻止数据。弱密码身份验证可能会被暴力攻击利用。

这里提供了三个例子。第一个例子涵盖了密码认证，这是常见的，但由于密码的熵和位数与加密密钥相比较低，因此不建议使用。第二个例子演示了如何使用私钥对远程服务器进行身份验证。这两个例子都忽略了远程主机提供的公钥。这是不安全的，因为您可能最终连接到一个您不信任的远程主机，但对于测试来说已经足够了。身份验证的第三个例子是理想的流程。它使用密钥进行身份验证并验证远程服务器。

请注意，本章不使用 PEM 格式的密钥文件，而是使用 SSH 格式的密钥，这是处理 SSH 最常见的格式。这些例子与 OpenSSH 工具和密钥兼容，如`ssh`、`sshd`、`ssh-keygen`、`ssh-copy-id`和`ssh-keyscan`。

我建议您使用`ssh-keygen`生成用于身份验证的公钥和私钥对。这将以 SSH 密钥格式生成`id_rsa`和`id_rsa.pub`文件。`ssh-keygen`工具是 OpenSSH 项目的一部分，并且默认情况下已经打包到 Ubuntu 中：

```go
ssh-keygen
```

使用`ssh-copy-id`将您的公钥（`id_rsa.pub`）复制到远程服务器的`~/.ssh/authorized_keys`文件中，以便您可以使用私钥进行身份验证：

```go
ssh-copy-id yourserver.com
```

# 使用密码进行身份验证

通过 SSH 进行密码身份验证是最简单的方法。此示例演示了如何使用`ssh.ClientConfig`结构配置 SSH 客户端，然后使用`ssh.Dial()`连接到 SSH 服务器。客户端被配置为使用密码，通过指定`ssh.Password()`作为身份验证函数：

```go
package main

import (
   "golang.org/x/crypto/ssh"
   "log"
)

var username = "username"
var password = "password"
var host = "example.com:22"

func main() {
   config := &ssh.ClientConfig{
      User: username,
      Auth: []ssh.AuthMethod{
         ssh.Password(password),
      },
      HostKeyCallback: ssh.InsecureIgnoreHostKey(),
   }
   client, err := ssh.Dial("tcp", host, config)
   if err != nil {
      log.Fatal("Error dialing server. ", err)
   }

   log.Println(string(client.ClientVersion()))
} 
```

# 使用私钥进行身份验证

与密码相比，私钥具有一些优势。它比密码长得多，使得暴力破解变得更加困难。它还消除了输入密码的需要，使连接到远程服务器变得更加方便。无密码身份验证对于需要在没有人为干预的情况下自动运行的 cron 作业和其他服务也是有帮助的。一些服务器完全禁用密码身份验证并要求使用密钥。

在您可以使用私钥进行身份验证之前，远程服务器将需要您的公钥作为授权密钥。

如果您的系统上有`ssh-copy-id`工具，您可以使用它。它将把您的公钥复制到远程服务器，放置在您的家目录 SSH 目录（`~/.ssh/authorized_keys`）中，并设置正确的权限：

```go
ssh-copy-id example.com 
```

下面的例子与前面的例子类似，我们使用密码进行身份验证，但`ssh.ClientConfig`被配置为使用`ssh.PublicKeys()`作为身份验证函数，而不是`ssh.Password()`。我们还将创建一个名为`getKeySigner()`的特殊函数，以便从文件中加载客户端的私钥：

```go
package main

import (
   "golang.org/x/crypto/ssh"
   "io/ioutil"
   "log"
)

var username = "username"
var host = "example.com:22"
var privateKeyFile = "/home/user/.ssh/id_rsa"

func getKeySigner(privateKeyFile string) ssh.Signer {
   privateKeyData, err := ioutil.ReadFile(privateKeyFile)
   if err != nil {
      log.Fatal("Error loading private key file. ", err)
   }

   privateKey, err := ssh.ParsePrivateKey(privateKeyData)
   if err != nil {
      log.Fatal("Error parsing private key. ", err)
   }
   return privateKey
}

func main() {
   privateKey := getKeySigner(privateKeyFile)
   config := &ssh.ClientConfig{
      User: username,
      Auth: []ssh.AuthMethod{
         ssh.PublicKeys(privateKey), // Pass 1 or more key
      },
      HostKeyCallback: ssh.InsecureIgnoreHostKey(),
   }

   client, err := ssh.Dial("tcp", host, config)
   if err != nil {
      log.Fatal("Error dialing server. ", err)
   }

   log.Println(string(client.ClientVersion()))
} 
```

请注意，您可以将多个私钥传递给`ssh.PublicKeys()`函数。它接受无限数量的密钥。如果您提供多个密钥，但只有一个适用于服务器，它将自动使用适用的密钥。

如果您想使用相同的配置连接到多台服务器，这将非常有用。您可能希望使用 1,000 个唯一的私钥连接到 1,000 个不同的主机。您可以重用包含所有私钥的单个配置，而不必创建多个 SSH 客户端配置。

# 验证远程主机

要验证远程主机，在`ssh.ClientConfig`中，将`HostKeyCallback`设置为`ssh.FixedHostKey()`，并传递远程主机的公钥。如果您尝试连接到服务器并提供了不同的公钥，连接将被中止。这对于确保您连接到预期的服务器而不是恶意服务器非常重要。如果 DNS 受到损害，或者攻击者执行了成功的 ARP 欺骗，您的连接可能会被重定向或成为中间人攻击的受害者，但攻击者将无法模仿真实服务器而没有相应的服务器私钥。出于测试目的，您可以选择忽略远程主机提供的密钥。

这个例子是连接最安全的方式。它使用密钥进行身份验证，而不是密码，并验证远程服务器的公钥。

此方法将使用`ssh.ParseKnownHosts()`。这使用标准的`known_hosts`文件。`known_hosts`格式是 OpenSSH 的标准。该格式在*sshd(8)*手册页中有文档记录。

请注意，Go 的`ssh.ParseKnownHosts()`只会解析单个条目，因此您应该创建一个包含服务器单个条目的唯一文件，或者确保所需的条目位于文件顶部。

要获取远程服务器的公钥以进行验证，请使用`ssh-keyscan`。这将以`known_hosts`格式返回服务器密钥，将在以下示例中使用。请记住，Go 的`ssh.ParseKnownHosts`命令只读取`known_hosts`文件的第一个条目：

```go
ssh-keyscan yourserver.com
```

`ssh-keyscan`程序将返回多个密钥类型，除非使用`-t`标志指定密钥类型。确保选择具有所需密钥算法的密钥类型，并且`ssh.ClientConfig()`中列出的`HostKeyAlgorithm`与之匹配。此示例包括每个可能的`ssh.KeyAlgo*`选项。我建议您选择尽可能高强度的算法，并且只允许该选项：

```go
package main

import (
   "golang.org/x/crypto/ssh"
   "io/ioutil"
   "log"
)

var username = "username"
var host = "example.com:22"
var privateKeyFile = "/home/user/.ssh/id_rsa"

// Known hosts only reads FIRST entry
var knownHostsFile = "/home/user/.ssh/known_hosts"

func getKeySigner(privateKeyFile string) ssh.Signer {
   privateKeyData, err := ioutil.ReadFile(privateKeyFile)
   if err != nil {
      log.Fatal("Error loading private key file. ", err)
   }

   privateKey, err := ssh.ParsePrivateKey(privateKeyData)
   if err != nil {
      log.Fatal("Error parsing private key. ", err)
   }
   return privateKey
}

func loadServerPublicKey(knownHostsFile string) ssh.PublicKey {
   publicKeyData, err := ioutil.ReadFile(knownHostsFile)
   if err != nil {
      log.Fatal("Error loading server public key file. ", err)
   }

   _, _, publicKey, _, _, err := ssh.ParseKnownHosts(publicKeyData)
   if err != nil {
      log.Fatal("Error parsing server public key. ", err)
   }
   return publicKey
}

func main() {
   userPrivateKey := getKeySigner(privateKeyFile)
   serverPublicKey := loadServerPublicKey(knownHostsFile)

   config := &ssh.ClientConfig{
      User: username,
      Auth: []ssh.AuthMethod{
         ssh.PublicKeys(userPrivateKey),
      },
      HostKeyCallback: ssh.FixedHostKey(serverPublicKey),
      // Acceptable host key algorithms (Allow all)
      HostKeyAlgorithms: []string{
         ssh.KeyAlgoRSA,
         ssh.KeyAlgoDSA,
         ssh.KeyAlgoECDSA256,
         ssh.KeyAlgoECDSA384,
         ssh.KeyAlgoECDSA521,
         ssh.KeyAlgoED25519,
      },
   }

   client, err := ssh.Dial("tcp", host, config)
   if err != nil {
      log.Fatal("Error dialing server. ", err)
   }

   log.Println(string(client.ClientVersion()))
} 
```

请注意，除了`ssh.KeyAlgo*`常量之外，如果使用证书，还有`ssh.CertAlgo*`常量。

# 通过 SSH 执行命令

现在我们已经建立了多种身份验证和连接到远程 SSH 服务器的方式，我们需要让`ssh.Client`开始工作。到目前为止，我们只是打印出客户端版本。第一个目标是执行单个命令并查看输出。

一旦创建了`ssh.Client`，就可以开始创建会话。一个客户端可以同时支持多个会话。会话有自己的标准输入、输出和错误。它们是标准的读取器和写入器接口。

要执行命令，有几个选项：`Run()`、`Start()`、`Output()`和`CombinedOutput()`。它们都非常相似，但行为略有不同：

+   `session.Output(cmd)`: `Output()`函数将执行命令，并将`session.Stdout`作为字节片返回。

+   `session.CombinedOutput(cmd)`: 这与`Output()`相同，但它返回标准输出和标准错误的组合。

+   `session.Run(cmd)`: `Run()`函数将执行命令并等待其完成。它将填充标准输出和错误缓冲区，但不会对其进行任何操作。您必须手动读取缓冲区，或在调用`Run()`之前将会话输出设置为转到终端输出（例如，`session.Stdout = os.Stdout`）。只有在程序以错误代码`0`退出并且没有复制标准输出缓冲区时，它才会返回而不出现错误。

+   `session.Start(cmd)`: `Start()`函数类似于`Run()`，但它不会等待命令完成。如果要阻塞执行直到命令完成，必须显式调用`session.Wait()`。这对于启动长时间运行的命令或者对应用程序流程有更多控制的情况非常有用。

一个会话只能执行一个操作。一旦调用`Run()`、`Output()`、`CombinedOutput()`、`Start()`或`Shell()`，就不能再使用该会话执行任何其他命令。如果需要运行多个命令，可以用分号将它们串联在一起。例如，可以像这样在单个命令字符串中传递多个命令：

```go
df -h; ps aux; pwd; whoami;
```

否则，您可以为需要运行的每个命令创建一个新会话。一个会话等同于一个命令。

以下示例使用密钥认证连接到远程 SSH 服务器，然后使用`client.NewSession()`创建一个会话。然后将会话的标准输出连接到我们本地终端的标准输出，然后调用`session.Run()`，这将在远程服务器上执行命令：

```go
package main

import (
   "golang.org/x/crypto/ssh"
   "io/ioutil"
   "log"
   "os"
)

var username = "username"
var host = "example.com:22"
var privateKeyFile = "/home/user/.ssh/id_rsa"
var commandToExecute = "hostname"

func getKeySigner(privateKeyFile string) ssh.Signer {
   privateKeyData, err := ioutil.ReadFile(privateKeyFile)
   if err != nil {
      log.Fatal("Error loading private key file. ", err)
   }

   privateKey, err := ssh.ParsePrivateKey(privateKeyData)
   if err != nil {
      log.Fatal("Error parsing private key. ", err)
   }
   return privateKey
}

func main() {
   privateKey := getKeySigner(privateKeyFile)
   config := &ssh.ClientConfig{
      User: username,
      Auth: []ssh.AuthMethod{
         ssh.PublicKeys(privateKey),
      },
      HostKeyCallback: ssh.InsecureIgnoreHostKey(),
   }

   client, err := ssh.Dial("tcp", host, config)
   if err != nil {
      log.Fatal("Error dialing server. ", err)
   }

   // Multiple sessions per client are allowed
   session, err := client.NewSession()
   if err != nil {
      log.Fatal("Failed to create session: ", err)
   }
   defer session.Close()

   // Pipe the session output directly to standard output
   // Thanks to the convenience of writer interface
   session.Stdout = os.Stdout

   err = session.Run(commandToExecute)
   if err != nil {
      log.Fatal("Error executing command. ", err)
   }
} 
```

# 启动交互式 shell

在前面的例子中，我们演示了如何运行命令字符串。还有一个选项可以打开一个 shell。通过调用`session.Shell()`，可以执行一个交互式登录 shell，加载用户的默认 shell 和默认配置文件（例如`.profile`）。调用`session.RequestPty()`是可选的，但是当请求一个伪终端时，shell 的工作效果要好得多。您可以将终端名称设置为`xterm`、`vt100`、`linux`或其他自定义名称。如果由于输出颜色值而导致输出混乱的问题，可以尝试使用`vt100`，如果仍然不起作用，可以使用非标准的终端名称或您知道不支持颜色的终端名称。许多程序会在不识别终端名称时禁用颜色输出。一些程序在未知的终端类型下根本无法工作，比如`tmux`。

有关 Go 终端模式常量的更多信息，请访问[`godoc.org/golang.org/x/crypto/ssh#TerminalModes`](https://godoc.org/golang.org/x/crypto/ssh#TerminalModes)。终端模式标志是 POSIX 标准，并在*RFC 4254*，*终端模式的编码*（第 8 节）中定义，您可以在[`tools.ietf.org/html/rfc4254#section-8`](https://tools.ietf.org/html/rfc4254#section-8)找到。

以下示例使用密钥认证连接到 SSH 服务器，然后使用`client.NewSession()`创建一个新会话。与前面的例子不同，我们将使用`session.RequestPty()`来获取一个交互式 shell，远程会话的标准输入、输出和错误流都连接到本地终端，因此您可以像与任何其他 SSH 客户端（例如 PuTTY）一样实时交互：

```go
package main

import (
   "fmt"
   "golang.org/x/crypto/ssh"
   "io/ioutil"
   "log"
   "os"
)

func checkArgs() (string, string, string) {
   if len(os.Args) != 4 {
      printUsage()
      os.Exit(1)
   }
   return os.Args[1], os.Args[2], os.Args[3]
}

func printUsage() {
   fmt.Println(os.Args[0] + ` - Open an SSH shell

Usage:
  ` + os.Args[0] + ` <username> <host> <privateKeyFile>

Example:
  ` + os.Args[0] + ` nanodano devdungeon.com:22 ~/.ssh/id_rsa
`)
}

func getKeySigner(privateKeyFile string) ssh.Signer {
   privateKeyData, err := ioutil.ReadFile(privateKeyFile)
   if err != nil {
      log.Fatal("Error loading private key file. ", err)
   }

   privateKey, err := ssh.ParsePrivateKey(privateKeyData)
   if err != nil {
      log.Fatal("Error parsing private key. ", err)
   }
   return privateKey
}

func main() {
   username, host, privateKeyFile := checkArgs()

   privateKey := getKeySigner(privateKeyFile)
   config := &ssh.ClientConfig{
      User: username,
      Auth: []ssh.AuthMethod{
         ssh.PublicKeys(privateKey),
      },
      HostKeyCallback: ssh.InsecureIgnoreHostKey(),
   }

   client, err := ssh.Dial("tcp", host, config)
   if err != nil {
      log.Fatal("Error dialing server. ", err)
   }

   session, err := client.NewSession()
   if err != nil {
      log.Fatal("Failed to create session: ", err)
   }
   defer session.Close()

   // Pipe the standard buffers together
   session.Stdout = os.Stdout
   session.Stdin = os.Stdin
   session.Stderr = os.Stderr

   // Get psuedo-terminal
   err = session.RequestPty(
      "vt100", // or "linux", "xterm"
      40,      // Height
      80,      // Width
      // https://godoc.org/golang.org/x/crypto/ssh#TerminalModes
      // POSIX Terminal mode flags defined in RFC 4254 Section 8.
      // https://tools.ietf.org/html/rfc4254#section-8
      ssh.TerminalModes{
         ssh.ECHO: 0,
      })
   if err != nil {
      log.Fatal("Error requesting psuedo-terminal. ", err)
   }

   // Run shell until it is exited
   err = session.Shell()
   if err != nil {
      log.Fatal("Error executing command. ", err)
   }
   session.Wait()
} 
```

# 总结

阅读完本章后，您现在应该了解如何使用 Go SSH 客户端连接和使用密码或私钥进行身份验证。此外，您现在应该了解如何在远程服务器上执行命令或开始交互式会话。

您如何以编程方式应用 SSH 客户端？您能想到任何用例吗？您管理多个远程服务器吗？您能自动化任何任务吗？

SSH 包还包含用于创建 SSH 服务器的类型和函数，但我们在本书中没有涵盖它们。阅读有关创建 SSH 服务器的更多信息，请访问[`godoc.org/golang.org/x/crypto/ssh#NewServerConn`](https://godoc.org/golang.org/x/crypto/ssh#NewServerConn)，以及有关 SSH 包的更多信息，请访问[`godoc.org/golang.org/x/crypto/ssh`](https://godoc.org/golang.org/x/crypto/ssh)。

在下一章中，我们将讨论暴力攻击，即猜测密码，直到最终找到正确的密码为止。暴力破解是我们可以使用 SSH 客户端以及其他协议和应用程序进行的操作。继续阅读下一章，了解如何执行暴力攻击。
