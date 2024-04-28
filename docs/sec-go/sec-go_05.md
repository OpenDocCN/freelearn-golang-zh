# 第五章：数据包捕获和注入

数据包捕获是监视通过网络传输的原始流量的过程。这适用于有线以太网和无线网络设备。在数据包捕获方面，`tcpdump`和`libpcap`包是标准。它们是在 20 世纪 80 年代编写的，至今仍在使用。`gopacket`包不仅包装了 C 库，还添加了 Go 抽象层，使其更符合 Go 的习惯用法并且更实用。

`pcap`库允许您收集有关网络设备的信息，从网络中读取数据包，将流量存储在`.pcap`文件中，根据多种条件过滤流量，或伪造自定义数据包并通过网络设备发送它们。对于`pcap`库，过滤是使用**伯克利数据包过滤器**（**BPF**）完成的。

数据包捕获有无数种用途。它可以用于设置蜜罐并监视接收到的流量类型。它可以帮助进行取证调查，以确定哪些主机表现恶意，哪些主机被利用。它可以帮助识别网络中的瓶颈。它也可以被恶意使用来从无线网络中窃取信息，执行数据包扫描，模糊测试，ARP 欺骗和其他类型的攻击。

这些示例需要一个非 Go 依赖项和一个`libpcap`包，因此在运行时可能会更具挑战性。如果您尚未将 Linux 作为主要桌面系统使用，我强烈建议您在虚拟机中使用 Ubuntu 或其他 Linux 发行版以获得最佳结果。

Tcpdump 是由`libpcap`的作者编写的应用程序。Tcpdump 提供了一个用于捕获数据包的命令行实用程序。这些示例将允许您复制`tcpdump`包的功能，并将其嵌入到其他应用程序中。其中一些示例与`tcpdump`的现有功能非常相似，如果适用，将提供`tcpdump`的示例用法。由于`gopacket`和`tcpdump`都依赖于相同的底层`libpcap`包，因此它们之间的文件格式是兼容的。您可以使用`tcpdump`捕获文件，并使用`gopacket`读取它们，也可以使用`gopacket`捕获数据包，并使用任何使用`libpcap`的应用程序读取它们，例如 Wireshark。

`gopacket`包的官方文档可在[`godoc.org/github.com/google/gopacket`](https://godoc.org/github.com/google/gopacket)找到。

# 先决条件

在运行这些示例之前，您需要安装`libpcap`。此外，我们还必须使用第三方 Go 包。幸运的是，这个包是由 Google 提供的，是一个可信赖的来源。Go 的`get`功能将下载并安装远程包。Git 也将需要用于使`go get`正常工作。

# 安装 libpcap 和 Git

`libpcap`包依赖项在大多数系统上都没有预安装，并且安装过程对每个操作系统都是不同的。在这里，我们将介绍 Ubuntu、Windows 和 macOS 上安装`libpcap`和`git`的步骤。我强烈建议您使用 Ubuntu 或其他 Linux 发行版以获得最佳结果。没有`libpcap`，`gopacket`将无法正常工作，而`git`是获取`gopacket`依赖项所必需的。

# 在 Ubuntu 上安装 libpcap

在 Ubuntu 中，默认情况下已经安装了`libpcap-0.8`。但是，要安装`gopacket`库，你还需要开发包中的头文件。你可以通过`libpcap-dev`包安装头文件。我们还将安装`git`，因为在安装`gopacket`时稍后需要运行`go get`命令。

```go
sudo apt-get install git libpcap-dev
```

# 在 Windows 上安装 libpcap

Windows 是最棘手的，也是出现最多问题的地方。Windows 实现的支持并不是很好，你的使用体验可能会有所不同。WinPcap 与 libpcap 兼容，这些示例中使用的源代码也可以直接在 Windows 上运行而无需修改。在 Windows 上运行时唯一显著的区别是网络设备的命名。

WinPcap 安装程序可从[`www.winpcap.org/`](https://www.winpcap.org/)获取，并且是一个必需的组件。如果需要开发人员包，可以在[`www.winpcap.org/devel.htm`](https://www.winpcap.org/devel.htm)获取，其中包含用 C 编写的包含文件和示例程序。对于大多数情况，您不需要开发人员包。Git 可以从[`git-scm.com/download/win`](https://git-scm.com/download/win)获取。您还需要 MinGW 作为编译器，可以从[`www.mingw.org`](http://www.mingw.org)获取。您需要确保 32 位和 64 位设置匹配。您可以设置`GOARCH=386`或`GOARCH=amd64`环境变量来在 32 位和 64 位之间切换。

# 在 macOS 上安装 libpcap

在 macOS 中，`libpcap`已经安装。您还需要 Git，可以通过 Homebrew 在[`brew.sh`](https://brew.sh)获取，或者通过 Git 软件包安装程序在[`git-scm.com/downloads`](https://git-scm.com/downloads)获取。

# 安装 gopacket

在满足`libpcap`和`git`软件包的要求后，您可以从 GitHub 获取`gopacket`软件包：

```go
go get github.com/google/gopacket  
```

# 权限问题

在 Linux 和 Mac 环境中执行程序时，可能会遇到访问网络设备时的权限问题。使用`sudo`来提升权限或切换用户到`root`来运行示例，但这并不推荐。

# 获取网络设备列表

`pcap`库的一部分包括一个用于获取网络设备列表的函数。

此程序将简单地获取网络设备列表并列出它们的信息。在 Linux 中，常见的默认设备名称是`eth0`或`wlan0`。在 Mac 上，是`en0`。在 Windows 上，名称不可读，因为它们更长并代表唯一的 ID。您可以使用设备名称作为字符串来标识以后示例中要捕获的设备。如果您没有看到确切设备的列表，可能需要以管理员权限运行示例（例如`sudo`）。

列出设备的等效`tcpdump`命令如下：

```go
tcpdump -D
```

或者，您可以使用以下命令：

```go
tcpdump --list-interfaces
```

您还可以使用`ifconfig`和`ip`等实用程序来获取您的网络设备的名称：

```go
package main

import (
   "fmt"
   "log"
   "github.com/google/gopacket/pcap"
)

func main() {
   // Find all devices
   devices, err := pcap.FindAllDevs()
   if err != nil {
      log.Fatal(err)
   }

   // Print device information
   fmt.Println("Devices found:")
   for _, device := range devices {
      fmt.Println("\nName: ", device.Name)
      fmt.Println("Description: ", device.Description)
      fmt.Println("Devices addresses: ", device.Description)
      for _, address := range device.Addresses {
         fmt.Println("- IP address: ", address.IP)
         fmt.Println("- Subnet mask: ", address.Netmask)
      }
   }
}
```

# 捕获数据包

以下程序演示了捕获数据包的基础知识。设备名称以字符串形式传递。如果您不知道设备名称，可以使用前面的示例来获取机器上可用设备的列表。如果您没有看到确切的设备名称列表，可能需要提升权限并使用`sudo`来运行程序。

混杂模式是一种选项，您可以启用它来监听并捕获并非发送给您设备的数据包。混杂模式在无线设备中尤其相关，因为无线网络设备实际上有能力捕获空中本来是发给其他接收者的数据包。

无线流量特别容易受到*嗅探*的影响，因为所有数据包都是通过空气广播而不是通过以太网传输，以太网需要物理访问才能拦截流量。提供没有加密的免费无线互联网在咖啡店和其他场所非常常见。这对客人来说很方便，但会使你的信息处于风险之中。如果场所提供加密的无线互联网，并不代表它自动更安全。如果密码被张贴在墙上，或者自由分发，那么任何有密码的人都可以解密无线流量。增加对客人无线网络的安全性的一种流行技术是使用捕获门户。捕获门户要求用户以某种方式进行身份验证，即使是作为访客，然后他们的会话被分割并使用单独的加密，这样其他人就无法解密它。

提供完全未加密流量的无线接入点必须小心使用。如果你连接到一个传递敏感信息的网站，请确保它使用 HTTPS，这样你和你访问的网站之间的数据就会被加密。VPN 连接也可以在未加密的通道上提供加密隧道。

一些网站是由不知情或疏忽的程序员构建的，他们在服务器上没有实现 SSL。一些网站只加密登录页面，以确保你的密码安全，但随后以明文传递会话 cookie。这意味着任何可以拦截无线流量的人都可以看到会话 cookie 并使用它来冒充受害者访问网站。网站将把攻击者视为受害者已登录。攻击者永远不会得知密码，但只要会话保持活动状态，他们就不需要密码。

一些网站的会话没有过期日期，它们会一直保持活动状态，直到明确退出登录。移动应用程序特别容易受到这种影响，因为用户很少退出并重新登录移动应用程序。关闭一个应用程序并重新打开它并不一定会创建一个新的会话。

这个例子将打开网络设备进行实时捕获，然后打印每个接收到的数据包的详细信息。程序将继续运行，直到使用*Ctrl* + *C* 杀死程序。

```go
package main

import (
   "fmt"
   "github.com/google/gopacket"
   "github.com/google/gopacket/pcap"
   "log"
   "time"
)

var (
   device            = "eth0"
   snapshotLen int32 = 1024
   promiscuous       = false
   err         error
   timeout     = 30 * time.Second
   handle      *pcap.Handle
)

func main() {
   // Open device
   handle, err = pcap.OpenLive(device, snapshotLen, promiscuous,  
      timeout)
   if err != nil {
      log.Fatal(err)
   }
   defer handle.Close()

   // Use the handle as a packet source to process all packets
   packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
   for packet := range packetSource.Packets() {
      // Process packet here
      fmt.Println(packet)
   }
}
```

# 使用过滤器捕获

以下程序演示了如何设置过滤器。过滤器使用 BPF 格式。如果你曾经使用过 Wireshark，你可能已经熟悉过滤器了。有许多可以逻辑组合的过滤选项。过滤器可以非常复杂，在网上有许多常见过滤器和巧妙技巧的速查表。以下是一些基本过滤器的示例，以便让你了解一些基本过滤器的想法：

+   `host 192.168.0.123`

+   `dst net 192.168.0.0/24`

+   `port 22`

+   `not broadcast and not multicast`

前面的一些过滤器应该是不言自明的。`host`过滤器将只显示发送到或从该主机的数据包。`dst net`过滤器将捕获发送到`192.168.0.*`地址的流量。`port`过滤器只监视端口`22`的流量。`not broadcast and not multicast`过滤器演示了如何否定和组合多个过滤器。过滤掉`广播`和`多播`是有用的，因为它们往往会使捕获变得混乱。

一个基本捕获的等效`tcpdump`命令只需运行它并传递一个接口：

```go
tcpdump -i eth0
```

如果你想传递过滤器，只需将它们作为命令行参数传递，就像这样：

```go
tcpdump -i eth0 tcp port 80
```

这个例子使用了一个只捕获 TCP 端口`80`流量的过滤器，这应该是 HTTP 流量。它没有指定本地端口或远程端口是否为`80`，因此它将捕获任何进出的端口`80`流量。如果你在个人电脑上运行它，你可能没有运行 web 服务器，所以它将捕获你通过 web 浏览器进行的 HTTP 流量。如果你在 web 服务器上运行捕获，它将捕获传入的 HTTP 请求流量。

在这个例子中，使用`pcap.OpenLive()`创建了一个网络设备的句柄。在从设备读取数据包之前，使用`handle.SetBPFFilter()`设置了过滤器，然后从句柄中读取数据包。在[`en.wikipedia.org/wiki/Berkeley_Packet_Filter`](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter)上了解更多关于过滤器的信息。

这个例子打开一个网络设备进行实时捕获，然后使用`SetBPFFilter()`设置一个过滤器。在这种情况下，我们将使用`tcp and port 80`过滤器来查找 HTTP 流量。捕获到的任何数据包都会被打印到标准输出：

```go
package main

import (
   "fmt"
   "github.com/google/gopacket"
   "github.com/google/gopacket/pcap"
   "log"
   "time"
)

var (
   device            = "eth0"
   snapshotLen int32 = 1024
   promiscuous       = false
   err         error
   timeout     = 30 * time.Second
   handle      *pcap.Handle
)

func main() {
   // Open device
   handle, err = pcap.OpenLive(device, snapshotLen, promiscuous,  
      timeout)
   if err != nil {
      log.Fatal(err)
   }
   defer handle.Close()

   // Set filter
   var filter string = "tcp and port 80" // or os.Args[1]
   err = handle.SetBPFFilter(filter)
   if err != nil {
      log.Fatal(err)
   }
   fmt.Println("Only capturing TCP port 80 packets.")

   packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
   for packet := range packetSource.Packets() {
      // Do something with a packet here.
      fmt.Println(packet)
   }
}
```

# 将数据保存到 pcap 文件

该程序将执行数据包捕获并将结果存储在文件中。在这个例子中的重要步骤是调用`pcapgo`包——`Writer`的`WriteFileHeader()`函数。之后，`WritePacket()`函数可以用来将所需的数据包写入文件。您可以捕获所有流量，并根据自己的过滤条件选择只写入特定的数据包，如果需要的话。也许您只想将奇数或格式错误的数据包写入日志以记录异常。

要使用`tcpdump`进行等效操作，只需使用`-w`标志和文件名，如下命令所示：

```go
tcpdump -i eth0 -w my_capture.pcap
```

使用这个例子创建的 pcap 文件可以使用 Wireshark 打开，并且可以像使用`tcpdump`创建的文件一样查看。

这个例子创建了一个名为`test.pcap`的输出文件，并打开一个网络设备进行实时捕获。它将 100 个数据包捕获到文件中，然后退出：

```go
package main

import (
   "fmt"
   "os"
   "time"

   "github.com/google/gopacket"
   "github.com/google/gopacket/layers"
   "github.com/google/gopacket/pcap"
   "github.com/google/gopacket/pcapgo"
)

var (
   deviceName        = "eth0"
   snapshotLen int32 = 1024
   promiscuous       = false
   err         error
   timeout     = -1 * time.Second
   handle      *pcap.Handle
   packetCount = 0
)

func main() {
   // Open output pcap file and write header
   f, _ := os.Create("test.pcap")
   w := pcapgo.NewWriter(f)
   w.WriteFileHeader(uint32(snapshotLen), layers.LinkTypeEthernet)
   defer f.Close()

   // Open the device for capturing
   handle, err = pcap.OpenLive(deviceName, snapshotLen, promiscuous, 
      timeout)
   if err != nil {
      fmt.Printf("Error opening device %s: %v", deviceName, err)
      os.Exit(1)
   }
   defer handle.Close()

   // Start processing packets
   packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
   for packet := range packetSource.Packets() {
      // Process packet here
      fmt.Println(packet)
      w.WritePacket(packet.Metadata().CaptureInfo, packet.Data())
      packetCount++

      // Only capture 100 and then stop
      if packetCount > 100 {
         break
      }
   }
}
```

# 从 pcap 文件中读取

您可以打开一个 pcap 文件进行离线检查，而不是打开一个设备进行实时捕获。无论是从`pcap.OpenLive()`还是`pcap.OpenOffline()`获取了一个句柄之后，该句柄都会被同等对待。一旦创建了句柄，实时设备和捕获文件之间就没有区别，只是实时设备将继续传递数据包，而文件最终会结束。

您可以使用任何`libpcap`客户端捕获的 pcap 文件，包括 Wireshark、`tcpdump`或其他`gopacket`应用程序。这个例子使用`pcap.OpenOffline()`打开一个名为`test.pcap`的文件，然后使用`range`迭代数据包并打印基本数据包信息。将文件名从`test.pcap`更改为您想要读取的任何文件：

```go
package main

// Use tcpdump to create a test file
// tcpdump -w test.pcap
// or use the example above for writing pcap files

import (
   "fmt"
   "github.com/google/gopacket"
   "github.com/google/gopacket/pcap"
   "log"
)

var (
   pcapFile = "test.pcap"
   handle   *pcap.Handle
   err      error
)

func main() {
   // Open file instead of device
   handle, err = pcap.OpenOffline(pcapFile)
   if err != nil {
      log.Fatal(err)
   }
   defer handle.Close()

   // Loop through packets in file
   packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
   for packet := range packetSource.Packets() {
      fmt.Println(packet)
   }
}
```

# 解码数据包层

数据包可以使用`packet.Layer()`函数逐层解码。该程序将检查数据包，查找 TCP 流量，然后输出以太网层、IP 层、TCP 层和应用层信息。当需要检查流量并根据信息做出决定时，这是非常有用的。当它到达应用层时，它会查找`HTTP`关键字，如果检测到，则打印一条消息：

```go
package main

import (
   "fmt"
   "github.com/google/gopacket"
   "github.com/google/gopacket/layers"
   "github.com/google/gopacket/pcap"
   "log"
   "strings"
   "time"
)

var (
   device            = "eth0"
   snapshotLen int32 = 1024
   promiscuous       = false
   err         error
   timeout     = 30 * time.Second
   handle      *pcap.Handle
)

func main() {
   // Open device
   handle, err = pcap.OpenLive(device, snapshotLen, promiscuous, 
      timeout)
   if err != nil {
      log.Fatal(err)
   }
   defer handle.Close()

   packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
   for packet := range packetSource.Packets() {
      printPacketInfo(packet)
   }
}

func printPacketInfo(packet gopacket.Packet) {
   // Let's see if the packet is an ethernet packet
   ethernetLayer := packet.Layer(layers.LayerTypeEthernet)
   if ethernetLayer != nil {
      fmt.Println("Ethernet layer detected.")
      ethernetPacket, _ := ethernetLayer.(*layers.Ethernet)
      fmt.Println("Source MAC: ", ethernetPacket.SrcMAC)
      fmt.Println("Destination MAC: ", ethernetPacket.DstMAC)
      // Ethernet type is typically IPv4 but could be ARP or other
      fmt.Println("Ethernet type: ", ethernetPacket.EthernetType)
      fmt.Println()
   }

   // Let's see if the packet is IP (even though the ether type told 
   //us)
   ipLayer := packet.Layer(layers.LayerTypeIPv4)
   if ipLayer != nil {
      fmt.Println("IPv4 layer detected.")
      ip, _ := ipLayer.(*layers.IPv4)

      // IP layer variables:
      // Version (Either 4 or 6)
      // IHL (IP Header Length in 32-bit words)
      // TOS, Length, Id, Flags, FragOffset, TTL, Protocol (TCP?),
      // Checksum, SrcIP, DstIP
      fmt.Printf("From %s to %s\n", ip.SrcIP, ip.DstIP)
      fmt.Println("Protocol: ", ip.Protocol)
      fmt.Println()
   }

   // Let's see if the packet is TCP
   tcpLayer := packet.Layer(layers.LayerTypeTCP)
   if tcpLayer != nil {
      fmt.Println("TCP layer detected.")
      tcp, _ := tcpLayer.(*layers.TCP)

      // TCP layer variables:
      // SrcPort, DstPort, Seq, Ack, DataOffset, Window, Checksum, 
      //Urgent
      // Bool flags: FIN, SYN, RST, PSH, ACK, URG, ECE, CWR, NS
      fmt.Printf("From port %d to %d\n", tcp.SrcPort, tcp.DstPort)
      fmt.Println("Sequence number: ", tcp.Seq)
      fmt.Println()
   }

   // Iterate over all layers, printing out each layer type
   fmt.Println("All packet layers:")
   for _, layer := range packet.Layers() {
      fmt.Println("- ", layer.LayerType())
   }

   // When iterating through packet.Layers() above,
   // if it lists Payload layer then that is the same as
   // this applicationLayer. applicationLayer contains the payload
   applicationLayer := packet.ApplicationLayer()
   if applicationLayer != nil {
      fmt.Println("Application layer/Payload found.")
      fmt.Printf("%s\n", applicationLayer.Payload())

      // Search for a string inside the payload
      if strings.Contains(string(applicationLayer.Payload()), "HTTP")    
      {
         fmt.Println("HTTP found!")
      }
   }

   // Check for errors
   if err := packet.ErrorLayer(); err != nil {
      fmt.Println("Error decoding some part of the packet:", err)
   }
}
```

# 创建自定义层

您不仅限于最常见的层，比如以太网、IP 和 TCP。您可以创建自己的层。对于大多数人来说，这种用途有限，但在一些极端罕见的情况下，用自定义层替换 TCP 层以满足特定要求可能是有意义的。

这个例子演示了如何创建一个自定义层。这对于实现`gopacket/layers`包中尚未包含的协议非常有用。`gopacket`中已经包含了 100 多种层类型。您可以在任何级别创建自定义层。

这段代码的第一件事是定义一个自定义数据结构来表示我们的层。数据结构不仅保存我们的自定义数据（`SomeByte`和`AnotherByte`），还需要一个字节片来存储实际负载的其余部分，以及其他层（`restOfData`）：

```go
package main

import (
   "fmt"
   "github.com/google/gopacket"
)

// Create custom layer structure
type CustomLayer struct {
   // This layer just has two bytes at the front
   SomeByte    byte
   AnotherByte byte
   restOfData  []byte
}

// Register the layer type so we can use it
// The first argument is an ID. Use negative
// or 2000+ for custom layers. It must be unique
var CustomLayerType = gopacket.RegisterLayerType(
   2001,
   gopacket.LayerTypeMetadata{
      "CustomLayerType",
      gopacket.DecodeFunc(decodeCustomLayer),
   },
)

// When we inquire about the type, what type of layer should
// we say it is? We want it to return our custom layer type
func (l CustomLayer) LayerType() gopacket.LayerType {
   return CustomLayerType
}

// LayerContents returns the information that our layer
// provides. In this case it is a header layer so
// we return the header information
func (l CustomLayer) LayerContents() []byte {
   return []byte{l.SomeByte, l.AnotherByte}
}

// LayerPayload returns the subsequent layer built
// on top of our layer or raw payload
func (l CustomLayer) LayerPayload() []byte {
   return l.restOfData
}

// Custom decode function. We can name it whatever we want
// but it should have the same arguments and return value
// When the layer is registered we tell it to use this decode function
func decodeCustomLayer(data []byte, p gopacket.PacketBuilder) error {
   // AddLayer appends to the list of layers that the packet has
   p.AddLayer(&CustomLayer{data[0], data[1], data[2:]})

   // The return value tells the packet what layer to expect
   // with the rest of the data. It could be another header layer,
   // nothing, or a payload layer.

   // nil means this is the last layer. No more decoding
   // return nil
   // Returning another layer type tells it to decode
   // the next layer with that layer's decoder function
   // return p.NextDecoder(layers.LayerTypeEthernet)

   // Returning payload type means the rest of the data
   // is raw payload. It will set the application layer
   // contents with the payload
   return p.NextDecoder(gopacket.LayerTypePayload)
}

func main() {
   // If you create your own encoding and decoding you can essentially
   // create your own protocol or implement a protocol that is not
   // already defined in the layers package. In our example we are    
   // just wrapping a normal ethernet packet with our own layer.
   // Creating your own protocol is good if you want to create
   // some obfuscated binary data type that was difficult for others
   // to decode. Finally, decode your packets:
   rawBytes := []byte{0xF0, 0x0F, 65, 65, 66, 67, 68}
   packet := gopacket.NewPacket(
      rawBytes,
      CustomLayerType,
      gopacket.Default,
   )
   fmt.Println("Created packet out of raw bytes.")
   fmt.Println(packet)

   // Decode the packet as our custom layer
   customLayer := packet.Layer(CustomLayerType)
   if customLayer != nil {
      fmt.Println("Packet was successfully decoded.")
      customLayerContent, _ := customLayer.(*CustomLayer)
      // Now we can access the elements of the custom struct
      fmt.Println("Payload: ", customLayerContent.LayerPayload())
      fmt.Println("SomeByte element:", customLayerContent.SomeByte)
      fmt.Println("AnotherByte element:",  
         customLayerContent.AnotherByte)
   }
}
```

# 将字节转换为数据包和从数据包转换

在某些情况下，可能有原始字节，您希望将其转换为数据包，或者反之亦然。这个例子创建了一个简单的数据包，然后获取组成数据包的原始字节。然后取这些原始字节并将其转换回数据包以演示这个过程。

在这个例子中，我们将使用`gopacket.SerializeLayers()`创建和序列化一个数据包。数据包由几个层组成：以太网、IP、TCP 和有效负载。在序列化过程中，如果任何数据包返回为 nil，这意味着它无法解码成正确的层（格式错误或不正确的数据包类型）。将数据包序列化到缓冲区后，我们将得到组成数据包的原始字节的副本，使用`buffer.Bytes()`。有了原始字节，我们可以使用`gopacket.NewPacket()`逐层解码数据。通过利用`SerializeLayers()`，您可以将数据包结构转换为原始字节，并使用`gopacket.NewPacket()`，您可以将原始字节转换回结构化数据。

`NewPacket()`将原始字节作为第一个参数。第二个参数是您想要解码的最底层层。它将解码该层以及其上的所有层。`NewPacket()`的第三个参数是解码类型，必须是以下之一：

+   `gopacket.Default`：这是一次性解码所有内容，也是最安全的。

+   `gopacket.Lazy`：这是按需解码，但不是并发安全的。

+   `gopacket.NoCopy`：这不会创建缓冲区的副本。只有在您可以保证内存中的数据包数据不会更改时才使用它

以下是将数据包结构转换为字节，然后再转换回数据包的完整代码：

```go
package main

import (
   "fmt"
   "github.com/google/gopacket"
   "github.com/google/gopacket/layers"
)

func main() {
   payload := []byte{2, 4, 6}
   options := gopacket.SerializeOptions{}
   buffer := gopacket.NewSerializeBuffer()
   gopacket.SerializeLayers(buffer, options,
      &layers.Ethernet{},
      &layers.IPv4{},
      &layers.TCP{},
      gopacket.Payload(payload),
   )
   rawBytes := buffer.Bytes()

   // Decode an ethernet packet
   ethPacket :=
      gopacket.NewPacket(
         rawBytes,
         layers.LayerTypeEthernet,
         gopacket.Default,
      )

   // with Lazy decoding it will only decode what it needs when it 
   //needs it
   // This is not concurrency safe. If using concurrency, use default
   ipPacket :=
      gopacket.NewPacket(
         rawBytes,
         layers.LayerTypeIPv4,
         gopacket.Lazy,
      )

   // With the NoCopy option, the underlying slices are referenced
   // directly and not copied. If the underlying bytes change so will
   // the packet
   tcpPacket :=
      gopacket.NewPacket(
         rawBytes,
         layers.LayerTypeTCP,
         gopacket.NoCopy,
      )

   fmt.Println(ethPacket)
   fmt.Println(ipPacket)
   fmt.Println(tcpPacket)
}
```

# 创建和发送数据包

这个例子做了几件事。首先，它将向您展示如何使用网络设备发送原始字节，这样您就可以几乎像串行连接一样使用它来发送数据。这对于真正的低级数据传输很有用，但如果您想与应用程序交互，您可能希望构建一个其他硬件和软件可以识别的数据包。

它接下来要做的事情是向您展示如何创建一个包含以太网、IP 和 TCP 层的数据包。不过，所有内容都是默认和空的，所以实际上并没有做任何事情。

最后，我们将创建另一个数据包，但实际上会为以太网层填写一些 MAC 地址，为 IPv4 填写一些 IP 地址，为 TCP 层填写一些端口号。您应该看到如何伪造数据包并冒充设备。

TCP 层结构具有`SYN`、`FIN`和`ACK`标志的布尔字段，可以读取或设置。这对于操纵和模糊化 TCP 握手、会话和端口扫描非常有用。

`pcap`库提供了一种发送字节的简单方法，但`gopacket`中的`layers`包协助我们创建了几个层的字节结构。

以下是此示例的代码实现：

```go
package main

import (
   "github.com/google/gopacket"
   "github.com/google/gopacket/layers"
   "github.com/google/gopacket/pcap"
   "log"
   "net"
   "time"
)

var (
   device            = "eth0"
   snapshotLen int32 = 1024
   promiscuous       = false
   err         error
   timeout     = 30 * time.Second
   handle      *pcap.Handle
   buffer      gopacket.SerializeBuffer
   options     gopacket.SerializeOptions
)

func main() {
   // Open device
   handle, err = pcap.OpenLive(device, snapshotLen, promiscuous, 
      timeout)
   if err != nil {
      log.Fatal("Error opening device. ", err)
   }
   defer handle.Close()

   // Send raw bytes over wire
   rawBytes := []byte{10, 20, 30}
   err = handle.WritePacketData(rawBytes)
   if err != nil {
      log.Fatal("Error writing bytes to network device. ", err)
   }

   // Create a properly formed packet, just with
   // empty details. Should fill out MAC addresses,
   // IP addresses, etc.
   buffer = gopacket.NewSerializeBuffer()
   gopacket.SerializeLayers(buffer, options,
      &layers.Ethernet{},
      &layers.IPv4{},
      &layers.TCP{},
      gopacket.Payload(rawBytes),
   )
   outgoingPacket := buffer.Bytes()
   // Send our packet
   err = handle.WritePacketData(outgoingPacket)
   if err != nil {
      log.Fatal("Error sending packet to network device. ", err)
   }

   // This time lets fill out some information
   ipLayer := &layers.IPv4{
      SrcIP: net.IP{127, 0, 0, 1},
      DstIP: net.IP{8, 8, 8, 8},
   }
   ethernetLayer := &layers.Ethernet{
      SrcMAC: net.HardwareAddr{0xFF, 0xAA, 0xFA, 0xAA, 0xFF, 0xAA},
      DstMAC: net.HardwareAddr{0xBD, 0xBD, 0xBD, 0xBD, 0xBD, 0xBD},
   }
   tcpLayer := &layers.TCP{
      SrcPort: layers.TCPPort(4321),
      DstPort: layers.TCPPort(80),
   }
   // And create the packet with the layers
   buffer = gopacket.NewSerializeBuffer()
   gopacket.SerializeLayers(buffer, options,
      ethernetLayer,
      ipLayer,
      tcpLayer,
      gopacket.Payload(rawBytes),
   )
   outgoingPacket = buffer.Bytes()
}
```

# 更快地解码数据包

如果我们知道要期望的层，我们可以使用现有的结构来存储数据包信息，而不是为每个数据包创建新的结构，这需要时间和内存。使用`DecodingLayerParser`更快。这就像编组和解组数据。

这个例子演示了如何在程序开始时创建层变量，并重复使用相同的变量，而不是为每个数据包创建新的变量。使用`gopacket.NewDecodingLayerParser()`创建一个解析器，我们提供了要使用的层变量。这里的一个注意事项是，它只会解码最初创建的层类型。

以下是此示例的代码实现：

```go
package main

import (
   "fmt"
   "github.com/google/gopacket"
   "github.com/google/gopacket/layers"
   "github.com/google/gopacket/pcap"
   "log"
   "time"
)

var (
   device            = "eth0"
   snapshotLen int32 = 1024
   promiscuous       = false
   err         error
   timeout     = 30 * time.Second
   handle      *pcap.Handle
   // Reuse these for each packet
   ethLayer layers.Ethernet
   ipLayer  layers.IPv4
   tcpLayer layers.TCP
)

func main() {
   // Open device
   handle, err = pcap.OpenLive(device, snapshotLen, promiscuous, 
   timeout)
   if err != nil {
      log.Fatal(err)
   }
   defer handle.Close()

   packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
   for packet := range packetSource.Packets() {
      parser := gopacket.NewDecodingLayerParser(
         layers.LayerTypeEthernet,
         &ethLayer,
         &ipLayer,
         &tcpLayer,
      )
      foundLayerTypes := []gopacket.LayerType{}

      err := parser.DecodeLayers(packet.Data(), &foundLayerTypes)
      if err != nil {
         fmt.Println("Trouble decoding layers: ", err)
      }

      for _, layerType := range foundLayerTypes {
         if layerType == layers.LayerTypeIPv4 {
            fmt.Println("IPv4: ", ipLayer.SrcIP, "->", ipLayer.DstIP)
         }
         if layerType == layers.LayerTypeTCP {
            fmt.Println("TCP Port: ", tcpLayer.SrcPort,               
               "->", tcpLayer.DstPort)
            fmt.Println("TCP SYN:", tcpLayer.SYN, " | ACK:", 
               tcpLayer.ACK)
         }
      }
   }
}
```

# 总结

阅读完本章后，您现在应该对`gopacket`包有很好的理解。您应该能够使用本章的示例编写一个简单的数据包捕获应用程序。再次强调，重要的不是记住所有的函数或层的细节。重要的是以高层次理解整体情况，并能够回忆起在范围和实施应用程序时可用的工具。

尝试根据这些示例编写自己的程序，以捕获来自您的计算机的有趣的网络流量。尝试捕获和检查特定端口或应用程序，以查看它在网络上传输时的工作方式。查看使用加密和以明文传输数据的应用程序之间的区别。您可能只是想捕获后台正在进行的所有流量，并查看在您空闲时哪些应用程序在网络上忙碌。

使用`gopacket`库可以构建各种有用的工具。除了基本的数据包捕获以供以后审查之外，您还可以实现一个监控系统，当识别到大量流量激增或发现异常流量时发出警报。

由于`gopacket`库也可以用于发送数据包，因此可以创建高度定制的端口扫描器。您可以制作原始数据包来执行仅进行 TCP SYN 扫描的操作，其中连接从未完全建立；XMAS 扫描，其中所有标志都被打开；NULL 扫描，其中每个字段都设置为 null；以及一系列其他需要对发送的数据包进行完全控制的扫描，包括故意发送格式错误的数据包。您还可以构建模糊测试器，向网络服务发送错误的数据包，以查看其行为。因此，看看您能想出什么样的想法。

在下一章中，我们将学习使用 Go 进行加密。我们将首先看一下哈希、校验和以及安全存储密码。然后我们将研究对称和非对称加密，它们是什么，它们的区别，为什么它们有用，以及如何在 Go 中使用它们。我们将学习如何创建带有证书的加密服务器，以及如何使用加密客户端进行连接。理解加密的应用对于现代安全至关重要，因此我们将研究最常见和实际的用例。
