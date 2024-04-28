# 第四章：取证

取证是收集证据以侦测犯罪。数字取证简单地指寻找数字证据，包括定位可能包含相关信息的异常文件，搜索隐藏数据，弄清楚文件最后修改时间，弄清楚谁发送了电子邮件，对文件进行散列，收集有关攻击 IP 的信息，或者捕获网络通信。

除了取证，本章还将涵盖隐写术的基本示例——将存档隐藏在图像中。隐写术是一种用来隐藏信息在其他信息中的技巧，使其不容易被发现。

散列，虽然与取证相关，但在《密码学》第六章中有所涵盖，数据包捕获则在第五章中有所涵盖，《数据包捕获和注入》。您将在本书的所有章节中找到对取证调查员有用的示例。

在本章中，您将学习以下主题：

+   文件取证

+   获取基本文件信息

+   查找大文件

+   查找最近更改的文件

+   读取磁盘的引导扇区

+   网络取证

+   查找主机名和 IP 地址

+   查找 MX 邮件记录

+   查找主机的名称服务器

+   隐写术

+   在图像中隐藏存档

+   检测图像中隐藏的存档

+   生成随机图像

+   创建 ZIP 存档

# 文件

文件取证很重要，因为攻击者可能留下痕迹，需要在进行更多更改或丢失任何信息之前收集证据。这包括确定谁拥有文件，它上次更改是什么时候，谁可以访问它，以及查看文件中是否有任何隐藏数据。

# 获取文件信息

让我们从简单的事情开始。这个程序将打印有关文件的信息，即最后修改时间，所有者是谁，它有多少字节，以及它的权限是什么。这也将作为一个很好的测试，以确保您的 Go 开发环境设置正确。

如果调查员发现了异常文件，首先要做的是检查所有基本元数据。这将提供有关文件所有者、哪些组可以访问它、最后修改时间、它是否是可执行文件以及它有多大的信息。所有这些信息都有潜在的用途。

我们将使用的主要函数是`os.Stat()`。这将返回一个`FileInfo`结构，我们将打印出来。我们必须在开始时导入`os`包以调用`os.Stat()`。从`os.Stat()`返回两个变量，这与许多只允许一个返回变量的语言不同。您可以使用下划线(`_`)符号代替变量名来忽略返回变量，例如您想要忽略的错误。

我们导入的`fmt`（格式）包包含典型的打印函数，如`fmt.Println()`和`fmt.Printf()`。`log`包包含`log.Printf()`和`log.Println()`。`fmt`和`log`之间的区别在于`log`在消息之前打印出`时间戳`，并且它是线程安全的。

`log`包有一个`fmt`中没有的函数，那就是`log.Fatal()`，它在打印后立即调用`os.Exit(1)`。`log.Fatal()`函数对处理某些错误条件很有用，通过打印错误并退出。如果您想要干净的输出并具有完全控制，请使用`fmt print`函数。如果每条消息上都有时间戳会很有用，请使用`log`包的打印函数。在收集取证线索时，记录您执行每个操作的时间很重要。

在这个示例中，变量在`main`函数之前的自己的部分中定义。这个范围内的变量对整个包都是可用的。这意味着每个函数都在同一个文件中，其他文件与相同的包声明在同一个目录中。定义变量的这种方法只是为了表明在 Go 中是可能的。这是 Pascal 对语言的影响之一，还有`:=`运算符。在后续示例中，为了节省空间，我们将利用*声明和赋值*运算符或`:=`符号。这在编写代码时很方便，因为你不必先声明变量类型。它会在编译时推断数据类型。然而，在阅读源代码时，显式声明变量类型可以帮助读者浏览代码。我们也可以将整个`var`声明放在`main`函数内部以进一步限制范围：

```go
package main

import (
   "fmt"
   "log"
   "os"
)

var (
   fileInfo os.FileInfo
   err error
)

func main() {
   // Stat returns file info. It will return
   // an error if there is no file.
   fileInfo, err = os.Stat("test.txt")
   if err != nil {
      log.Fatal(err)
   }
   fmt.Println("File name:", fileInfo.Name())
   fmt.Println("Size in bytes:", fileInfo.Size())
   fmt.Println("Permissions:", fileInfo.Mode())
   fmt.Println("Last modified:", fileInfo.ModTime())
   fmt.Println("Is Directory: ", fileInfo.IsDir())
   fmt.Printf("System interface type: %T\n", fileInfo.Sys())
   fmt.Printf("System info: %+v\n\n", fileInfo.Sys())
}
```

# 查找最大的文件

在调查时，大文件总是主要嫌疑对象。大型数据库转储、密码转储、彩虹表、信用卡缓存、窃取的知识产权和其他数据通常存储在一个大型存档中，如果你有合适的工具，很容易发现。此外，找到异常大的图像或视频文件也会很有帮助，因为它们可能包含了隐写信息。隐写术在本章中进一步介绍。

该程序将在一个目录和所有子目录中搜索所有文件并按文件大小进行排序。我们将使用`ioutil.ReadDir()`来探索初始目录，以获取`os.FileInfo`结构的内容切片。要检查文件是否为目录，我们将使用`os.IsDir()`。然后，我们将创建一个名为`FileNode`的自定义数据结构来存储我们需要的信息。我们使用链表来存储文件信息。在将元素插入列表之前，我们将遍历它以找到正确的位置，以便保持列表正确排序。请注意，在类似`/`的目录上运行程序可能需要很长时间。尝试更具体的目录，比如你的`home`文件夹：

```go
package main

import (
   "container/list"
   "fmt"
   "io/ioutil"
   "log"
   "os"
   "path/filepath"
)

type FileNode struct {
   FullPath string
   Info os.FileInfo
}

func insertSorted(fileList *list.List, fileNode FileNode) {
   if fileList.Len() == 0 { 
      // If list is empty, just insert and return
      fileList.PushFront(fileNode)
      return
   }

   for element := fileList.Front(); element != nil; element =    
      element.Next() {
      if fileNode.Info.Size() < element.Value.(FileNode).Info.Size()       
      {
         fileList.InsertBefore(fileNode, element)
         return
      }
   }
   fileList.PushBack(fileNode)
}

func getFilesInDirRecursivelyBySize(fileList *list.List, path string) {
   dirFiles, err := ioutil.ReadDir(path)
   if err != nil {
      log.Println("Error reading directory: " + err.Error())
   }

   for _, dirFile := range dirFiles {
      fullpath := filepath.Join(path, dirFile.Name())
      if dirFile.IsDir() {
         getFilesInDirRecursivelyBySize(
            fileList,
            filepath.Join(path, dirFile.Name()),
         )
      } else if dirFile.Mode().IsRegular() {
         insertSorted(
            fileList,
            FileNode{FullPath: fullpath, Info: dirFile},
         )
      }
   }
}

func main() {
   fileList := list.New()
   getFilesInDirRecursivelyBySize(fileList, "/home")

   for element := fileList.Front(); element != nil; element =   
      element.Next() {
      fmt.Printf("%d ", element.Value.(FileNode).Info.Size())
      fmt.Printf("%s\n", element.Value.(FileNode).FullPath)
   }
}
```

# 查找最近修改过的文件

在对受害者机器进行取证时，你可以做的第一件事之一是查找最近修改过的文件。这可能会给你一些线索，比如攻击者在哪里寻找，他们修改了什么设置，或者他们的动机是什么。

然而，如果调查人员正在查看攻击者的机器，那么目标略有不同。最近访问的文件可能会给出一些线索，比如他们用来攻击的工具，他们可能隐藏数据的地方，或者他们使用的软件。

以下示例将搜索一个目录和子目录，找到所有文件并按最后修改时间进行排序。这个示例非常类似于前一个示例，只是排序是通过使用`time.Time.Before()`函数比较时间戳来完成的：

```go
package main

import (
   "container/list"
   "fmt"
   "io/ioutil"
   "log"
   "os"
   "path/filepath"
)

type FileNode struct {
   FullPath string
   Info os.FileInfo
}

func insertSorted(fileList *list.List, fileNode FileNode) {
   if fileList.Len() == 0 { 
      // If list is empty, just insert and return
      fileList.PushFront(fileNode)
      return
   }

   for element := fileList.Front(); element != nil; element = 
      element.Next() {
      if fileNode.Info.ModTime().Before(element.Value.
        (FileNode).Info.ModTime()) {
            fileList.InsertBefore(fileNode, element)
            return
        }
    }

    fileList.PushBack(fileNode)
}

func GetFilesInDirRecursivelyBySize(fileList *list.List, path string) {
    dirFiles, err := ioutil.ReadDir(path)
    if err != nil {
        log.Println("Error reading directory: " + err.Error())
    }

    for _, dirFile := range dirFiles {
        fullpath := filepath.Join(path, dirFile.Name())
        if dirFile.IsDir() {
            GetFilesInDirRecursivelyBySize(
            fileList,
            filepath.Join(path, dirFile.Name()),
            )
        } else if dirFile.Mode().IsRegular() {
           insertSorted(
              fileList,
              FileNode{FullPath: fullpath, Info: dirFile},
           )
        }
    }
}

func main() {
    fileList := list.New()
    GetFilesInDirRecursivelyBySize(fileList, "/")

    for element := fileList.Front(); element != nil; element =    
       element.Next() {
        fmt.Print(element.Value.(FileNode).Info.ModTime())
        fmt.Printf("%s\n", element.Value.(FileNode).FullPath)
    }
}
```

# 读取引导扇区

该程序将读取磁盘的前 512 个字节，并将结果打印为十进制值、十六进制和字符串。`io.ReadFull()`函数类似于普通读取，但它确保你提供的数据字节片段完全填充。如果文件中的字节数不足以填充字节片段，则返回错误。

这个程序的一个实际用途是检查机器的引导扇区是否已被修改。Rootkits 和恶意软件可能通过修改引导扇区来劫持引导过程。您可以手动检查它是否有任何奇怪的东西，或者与已知的良好版本进行比较。也许可以比较机器的备份映像或新安装，看看是否有任何变化。

请注意，您可以在技术上传递任何文件名，而不是特定的磁盘，因为在 Linux 中，一切都被视为文件。如果直接传递设备的名称，例如`/dev/sda`，它将读取磁盘的前`512`个字节，即引导扇区。主要磁盘设备通常是`/dev/sda`，但也可能是`/dev/sdb`或`/dev/sdc`。使用`mount`或`df`工具获取有关磁盘名称的更多信息。您需要以`sudo`身份运行应用程序，以便具有直接读取磁盘设备的权限。

有关文件、输入和输出的更多信息，请查看`os`、`bufio`和`io`包，如下面的代码块所示：

```go
package main

// Device is typically /dev/sda but may also be /dev/sdb, /dev/sdc
// Use mount, or df -h to get info on which drives are being used
// You will need sudo to access some disks at this level

import (
   "io"
   "log"
   "os"
)

func main() {
   path := "/dev/sda"
   log.Println("[+] Reading boot sector of " + path)

   file, err := os.Open(path)
   if err != nil {
      log.Fatal("Error: " + err.Error())
   }

   // The file.Read() function will read a tiny file in to a large
   // byte slice, but io.ReadFull() will return an
   // error if the file is smaller than the byte slice.
   byteSlice := make([]byte, 512)
   // ReadFull Will error if 512 bytes not available to read
   numBytesRead, err := io.ReadFull(file, byteSlice)
   if err != nil {
      log.Fatal("Error reading 512 bytes from file. " + err.Error())
   }

   log.Printf("Bytes read: %d\n\n", numBytesRead)
   log.Printf("Data as decimal:\n%d\n\n", byteSlice)
   log.Printf("Data as hex:\n%x\n\n", byteSlice)
   log.Printf("Data as string:\n%s\n\n", byteSlice)
}
```

# 隐写术

隐写术是将消息隐藏在非机密消息中的做法。它不应与速记术混淆，速记术是指像法庭记录员一样记录口述的话语的做法。隐写术在历史上已经存在很长时间，一个老式的例子是在服装的缝纫中缝入摩尔斯电码消息。

在数字世界中，人们可以将任何类型的二进制数据隐藏在图像、音频或视频文件中。原始文件的质量可能会受到这一过程的影响。一些图像可以完全保持其原始完整性，但它们在形式上隐藏了额外的数据，如`.zip`或`.rar`存档。一些隐写术算法很复杂，将原始二进制数据隐藏在每个字节的最低位中，只轻微降低原始质量。其他隐写术算法更简单，只是将图像文件和存档合并成一个文件。我们将看看如何将存档隐藏在图像中，以及如何检测隐藏的存档。

# 生成具有随机噪声的图像

该程序将创建一个 JPEG 图像，其中每个像素都设置为随机颜色。这是一个简单的程序，因此我们只有一个可用的 jpeg 图像可供使用。Go 标准库配备了`jpeg`、`gif`和`png`包。所有不同类型的图像的接口都是相同的，因此从`jpeg`切换到`gif`或`png`包非常容易：

```go
package main

import (
   "image"
   "image/jpeg"
   "log"
   "math/rand"
   "os"
)

func main() {
   // 100x200 pixels
   myImage := image.NewRGBA(image.Rect(0, 0, 100, 200))

   for p := 0; p < 100*200; p++ {
      pixelOffset := 4 * p
      myImage.Pix[0+pixelOffset] = uint8(rand.Intn(256)) // Red
      myImage.Pix[1+pixelOffset] = uint8(rand.Intn(256)) // Green
      myImage.Pix[2+pixelOffset] = uint8(rand.Intn(256)) // Blue
      myImage.Pix[3+pixelOffset] = 255 // Alpha
   }

   outputFile, err := os.Create("test.jpg")
   if err != nil {
      log.Fatal(err)
   }

   jpeg.Encode(outputFile, myImage, nil)

   err = outputFile.Close()
   if err != nil {
      log.Fatal(err)
   }
}
```

# 创建 ZIP 存档

该程序将创建一个 ZIP 存档，以便我们在隐写术实验中使用。Go 标准库有一个`zip`包，但它也支持`tar`包的 TAR 存档。此示例生成一个包含两个文件的 ZIP 文件：`test.txt`和`test2.txt`。为了保持简单，每个文件的内容都被硬编码为源代码中的字符串：

```go
package main

import (
   "crypto/md5"
   "crypto/sha1"
   "crypto/sha256"
   "crypto/sha512"
   "fmt"
   "io/ioutil"
   "log"
   "os"
)

func printUsage() {
   fmt.Println("Usage: " + os.Args[0] + " <filepath>")
   fmt.Println("Example: " + os.Args[0] + " document.txt")
}

func checkArgs() string {
   if len(os.Args) < 2 {
      printUsage()
      os.Exit(1)
   }
   return os.Args[1]
}

func main() {
   filename := checkArgs()

   // Get bytes from file
   data, err := ioutil.ReadFile(filename)
   if err != nil {
      log.Fatal(err)
   }

   // Hash the file and output results
   fmt.Printf("Md5: %x\n\n", md5.Sum(data))
   fmt.Printf("Sha1: %x\n\n", sha1.Sum(data))
   fmt.Printf("Sha256: %x\n\n", sha256.Sum256(data))
   fmt.Printf("Sha512: %x\n\n", sha512.Sum512(data))
}
```

# 创建隐写图像存档

现在我们有了一张图像和一个 ZIP 存档，我们可以将它们组合在一起，将存档“隐藏”在图像中。这可能是最原始的隐写术形式。更高级的方法是逐字节拆分文件，将信息存储在图像的低位中，使用特殊程序从图像中提取数据，然后重建原始数据。这个例子很好，因为我们可以很容易地测试和验证它是否仍然作为图像加载，并且仍然像 ZIP 存档一样运行。

以下示例将采用 JPEG 图像和 ZIP 存档，并将它们组合在一起创建一个隐藏的存档。文件将保留`.jpg`扩展名，并且仍然可以像正常图像一样运行和查看。但是，该文件仍然可以作为 ZIP 存档工作。您可以解压缩`.jpg`文件，存档文件将被提取出来：

```go
package main

import (
   "io"
   "log"
   "os"
)

func main() {
   // Open original file
   firstFile, err := os.Open("test.jpg")
   if err != nil {
      log.Fatal(err)
   }
   defer firstFile.Close()

   // Second file
   secondFile, err := os.Open("test.zip")
   if err != nil {
      log.Fatal(err)
   }
   defer secondFile.Close()

   // New file for output
   newFile, err := os.Create("stego_image.jpg")
   if err != nil {
      log.Fatal(err)
   }
   defer newFile.Close()

   // Copy the bytes to destination from source
   _, err = io.Copy(newFile, firstFile)
   if err != nil {
      log.Fatal(err)
   }
   _, err = io.Copy(newFile, secondFile)
   if err != nil {
      log.Fatal(err)
   }
}

```

# 在 JPEG 图像中检测 ZIP 存档

如果使用前面示例中的技术隐藏数据，则可以通过在图像中搜索 ZIP 文件签名来检测数据。文件可能具有`.jpg`扩展名，并且在照片查看器中仍然可以正确加载，但它可能仍然在文件中存储有 ZIP 存档。以下程序将搜索文件并查找 ZIP 文件签名。我们可以对前面示例中创建的文件运行它：

```go
package main

import (
   "bufio"
   "bytes"
   "log"
   "os"
)

func main() {
   // Zip signature is "\x50\x4b\x03\x04"
   filename := "stego_image.jpg"
   file, err := os.Open(filename)
   if err != nil {
      log.Fatal(err)
   }
   bufferedReader := bufio.NewReader(file)

   fileStat, _ := file.Stat()
   // 0 is being cast to an int64 to force i to be initialized as
   // int64 because filestat.Size() returns an int64 and must be
   // compared against the same type
   for i := int64(0); i < fileStat.Size(); i++ {
      myByte, err := bufferedReader.ReadByte()
      if err != nil {
         log.Fatal(err)
      }

      if myByte == '\x50' { 
         // First byte match. Check the next 3 bytes
         byteSlice := make([]byte, 3)
         // Get bytes without advancing pointer with Peek
         byteSlice, err = bufferedReader.Peek(3)
         if err != nil {
            log.Fatal(err)
         }

         if bytes.Equal(byteSlice, []byte{'\x4b', '\x03', '\x04'}) {
            log.Printf("Found zip signature at byte %d.", i)
         }
      }
   }
}
```

# 网络

有时，日志中会出现奇怪的 IP 地址，您需要查找更多信息，或者可能有一个您需要根据 IP 地址定位的域名。这些示例演示了收集有关主机的信息。数据包捕获也是网络取证调查的一个重要部分，但是关于数据包捕获还有很多要说，因此第五章，*数据包捕获和注入*专门讨论了数据包捕获和注入。

# 从 IP 地址查找主机名

该程序将接受一个 IP 地址并找出主机名。`net.parseIP()`函数用于验证提供的 IP 地址，`net.LookupAddr()`完成了查找主机名的真正工作。

默认情况下，使用纯 Go 解析器。可以通过设置`GODEBUG`环境变量的`netdns`值来覆盖解析器。将`GODEBUG`的值设置为`go`或`cgo`。您可以在 Linux 中使用以下 shell 命令来执行此操作：

```go
export GODEBUG=netdns=go # force pure Go resolver (Default)
export GODEBUG=netdns=cgo # force cgo resolver
```

以下是程序的代码：

```go
package main

import (
   "fmt"
   "log"
   "net"
   "os"
)

func main() {
   if len(os.Args) != 2 {
      log.Fatal("No IP address argument provided.")
   }
   arg := os.Args[1]

   // Parse the IP for validation
   ip := net.ParseIP(arg)
   if ip == nil {
      log.Fatal("Valid IP not detected. Value provided: " + arg)
   }

   fmt.Println("Looking up hostnames for IP address: " + arg)
   hostnames, err := net.LookupAddr(ip.String())
   if err != nil {
      log.Fatal(err)
   }
   for _, hostnames := range hostnames {
      fmt.Println(hostnames)
   }
}
```

# 从主机名查找 IP 地址

以下示例接受主机名并返回 IP 地址。它与先前的示例非常相似，但是顺序相反。`net.LookupHost()`函数完成了大部分工作：

```go
package main

import (
   "fmt"
   "log"
   "net"
   "os"
)

func main() {
   if len(os.Args) != 2 {
      log.Fatal("No hostname argument provided.")
   }
   arg := os.Args[1]

   fmt.Println("Looking up IP addresses for hostname: " + arg)

   ips, err := net.LookupHost(arg)
   if err != nil {
      log.Fatal(err)
   }
   for _, ip := range ips {
      fmt.Println(ip)
   }
}
```

# 查找 MX 记录

该程序将接受一个域名并返回 MX 记录。MX 记录，或邮件交换记录，是指向邮件服务器的 DNS 记录。例如，[`www.devdungeon.com/`](https://www.devdungeon.com/)的 MX 服务器是`mail.devdungeon.com`。`net.LookupMX()`函数执行此查找并返回`net.MX`结构的切片：

```go
package main

import (
   "fmt"
   "log"
   "net"
   "os"
)

func main() {
   if len(os.Args) != 2 {
      log.Fatal("No domain name argument provided")
   }
   arg := os.Args[1]

   fmt.Println("Looking up MX records for " + arg)

   mxRecords, err := net.LookupMX(arg)
   if err != nil {
      log.Fatal(err)
   }
   for _, mxRecord := range mxRecords {
      fmt.Printf("Host: %s\tPreference: %d\n", mxRecord.Host,   
         mxRecord.Pref)
   }
}
```

# 查找主机名的域名服务器

该程序将查找与给定主机名关联的域名服务器。这里的主要功能是`net.LookupNS()`：

```go
package main

import (
   "fmt"
   "log"
   "net"
   "os"
)

func main() {
   if len(os.Args) != 2 {
      log.Fatal("No domain name argument provided")
   }
   arg := os.Args[1]

   fmt.Println("Looking up nameservers for " + arg)

   nameservers, err := net.LookupNS(arg)
   if err != nil {
      log.Fatal(err)
   }
   for _, nameserver := range nameservers {
      fmt.Println(nameserver.Host)
   }
}
```

# 总结

阅读完本章后，您现在应该对数字取证调查的目标有基本的了解。关于这些主题中的每一个都可以说更多，取证是一门需要自己的书籍，更不用说一章了。

将您已阅读的示例作为起点，思考一下如果您收到了一台被入侵的机器，您将寻找什么样的信息，以及您的目标是弄清楚攻击者是如何进入的，发生的时间，他们访问了什么，他们修改了什么，他们的动机是什么，有多少数据被外泄，以及您可以找到的任何其他信息，以确定行动者是谁或在系统上采取了什么行动。

熟练的对手将尽一切努力掩盖自己的踪迹，避免取证检测。因此，重要的是要及时了解正在使用的最新工具和趋势，以便在调查时知道要寻找的技巧和线索。

这些示例可以进行扩展，自动化，并集成到执行更大规模的取证搜索的其他应用程序中。借助 Go 的可扩展性，可以轻松地创建工具，以有效的方式搜索整个文件系统或网络。

在下一章中，我们将学习使用 Go 进行数据包捕获。我们将从基础知识开始，例如获取网络设备列表和将网络流量转储到文件中。然后，我们将讨论使用过滤器查找特定的网络流量。此外，我们将探讨使用 Go 接口解码和检查数据包的更高级技术。我们还将涵盖创建自定义数据包层以及从网络卡发送数据包的技术，从而允许您发送任意数据包。
