# 处理文件

Unix 和 Linux 系统的一个显著特点是将所有内容都视为文件。进程、文件、目录、套接字、设备和管道都被视为文件。鉴于操作系统的这一基本特性，学习如何操作文件是一项关键技能。本章提供了几个不同方式操作文件的示例。

首先，我们将看一下基础知识，即创建、截断、删除、打开、关闭、重命名和移动文件。我们还将看一下如何获取有关文件的详细属性，例如权限和所有权、大小和符号链接信息。

本章的一个专门部分是关于从文件中读取和写入的不同方式。有多个包含有用函数的包；此外，读取器和写入器接口可以实现许多不同的选项，例如缓冲读取器和写入器，直接读取和写入，扫描器，以及用于快速操作的辅助函数。

此外，还提供了用于归档和解档、压缩和解压缩、创建临时文件和目录以及通过 HTTP 下载文件的示例。

具体来说，本章将涵盖以下主题：

+   创建空文件和截断文件

+   获取详细的文件信息

+   重命名、移动和删除文件

+   操作权限、所有权和时间戳

+   符号链接

+   多种读写文件的方式

+   归档

+   压缩

+   临时文件和目录

+   通过 HTTP 下载文件

# 文件基础知识

因为文件是计算生态系统中不可或缺的一部分，了解 Go 中处理文件的选项至关重要。本节涵盖了一些基本操作，如打开、关闭、创建和删除文件。此外，它还涵盖了重命名和移动文件，查看文件是否存在，修改权限、所有权、时间戳以及处理符号链接。这些示例中大多数使用了一个硬编码的文件名`test.txt`。如果要操作不同的文件，请更改此文件名。

# 创建空文件

Linux 中常用的一个工具是**touch**程序。当您需要快速创建具有特定名称的空文件时，它经常被使用。以下示例复制了**touch**的一个常见用例，即创建一个空文件。

创建空文件的用途有限，但让我们考虑一个例子。假设有一个服务将日志写入一组旋转的文件中。每天都会创建一个带有当前日期的新文件，并将当天的日志写入该文件。开发人员可能会聪明地对日志文件设置非常严格的权限，以便只有管理员可以读取它们。但是，如果他们在目录上留下了宽松的权限会怎么样？如果您创建了一个带有下一天日期的空文件会发生什么？服务可能只会在不存在日志文件时创建新的日志文件，但如果存在一个文件，它将在不检查权限的情况下使用它。您可以利用这一点，创建一个您有读取权限的空文件。该文件应该以服务命名日志文件的方式命名。例如，如果服务使用以下格式记录日志：`logs-2018-01-30.txt`，您可以创建一个名为`logs-2018-01-31.txt`的空文件，第二天，服务将写入该文件，因为它已经存在，而您将具有读取权限，而不是服务如果没有文件存在则创建一个具有仅根权限的新文件。

以下是此示例的代码实现：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   newFile, err := os.Create("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Println(newFile) 
   newFile.Close() 
} 
```

# 截断文件

截断文件是指将文件修剪到最大长度。截断通常用于完全删除文件的所有内容，但也可以用于将文件限制为特定的最大大小。`os.Truncate()`的一个显着特点是，如果文件小于指定的截断限制，它将实际增加文件的长度。它将用空字节填充任何空白空间。

截断文件比创建空文件有更多的实际用途。当日志文件变得太大时，可以截断它们以节省磁盘空间。如果您正在攻击，可能希望截断`.bash_history`和其他日志文件以掩盖您的踪迹。恶意行为者可能仅仅为了破坏数据而截断文件。

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   // Truncate a file to 100 bytes. If file 
   // is less than 100 bytes the original contents will remain 
   // at the beginning, and the rest of the space is 
   // filled will null bytes. If it is over 100 bytes, 
   // Everything past 100 bytes will be lost. Either way 
   // we will end up with exactly 100 bytes. 
   // Pass in 0 to truncate to a completely empty file 

   err := os.Truncate("test.txt", 100) 
   if err != nil { 
      log.Fatal(err) 
   } 
} 
```

# 获取文件信息

以下示例将打印有关文件的所有可用元数据。它包括显而易见的属性，即名称、大小、权限、上次修改时间以及它是否是目录。它包含的最后一个数据片段是`FileInfo.Sys()`接口。这包含有关文件底层来源的信息，最常见的是硬盘上的文件系统：

```go
package main 

import ( 
   "fmt" 
   "log" 
   "os" 
) 

func main() { 
   // Stat returns file info. It will return 
   // an error if there is no file. 
   fileInfo, err := os.Stat("test.txt") 
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

# 重命名文件

标准库提供了一个方便的函数来移动文件。重命名和移动是同义词；如果要将文件从一个目录移动到另一个目录，请使用`os.Rename()`函数，如下面的代码块所示：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   originalPath := "test.txt" 
   newPath := "test2.txt" 
   err := os.Rename(originalPath, newPath) 
   if err != nil { 
      log.Fatal(err) 
   } 
} 
```

# 删除文件

以下示例很简单，演示了如何删除文件。标准包提供了`os.Remove()`，它需要一个文件路径：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   err := os.Remove("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
} 
```

# 打开和关闭文件

在打开文件时，有几个选项。当调用`os.Open()`时，它只需要一个文件名，并提供一个只读文件。另一个选项是使用`os.OpenFile()`，它需要更多的选项。您可以指定是否要只读或只写文件。您还可以选择在打开时读取和写入、追加、如果不存在则创建，或者截断。将所需的选项与逻辑或运算符结合。通过在文件对象上调用`Close()`来关闭文件。您可以显式关闭文件，也可以推迟调用。有关`defer`关键字的更多详细信息，请参阅第二章，*Go 编程语言*。以下示例不使用`defer`关键字选项，但后续示例将使用：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   // Simple read only open. We will cover actually reading 
   // and writing to files in examples further down the page 
   file, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   }  
   file.Close() 

   // OpenFile with more options. Last param is the permission mode 
   // Second param is the attributes when opening 
   file, err = os.OpenFile("test.txt", os.O_APPEND, 0666) 
   if err != nil { 
      log.Fatal(err) 
   } 
   file.Close() 

   // Use these attributes individually or combined 
   // with an OR for second arg of OpenFile() 
   // e.g. os.O_CREATE|os.O_APPEND 
   // or os.O_CREATE|os.O_TRUNC|os.O_WRONLY 

   // os.O_RDONLY // Read only 
   // os.O_WRONLY // Write only 
   // os.O_RDWR // Read and write 
   // os.O_APPEND // Append to end of file 
   // os.O_CREATE // Create is none exist 
   // os.O_TRUNC // Truncate file when opening 
} 
```

# 检查文件是否存在

检查文件是否存在是一个两步过程。首先，必须在文件上调用`os.Stat()`以获取`FileInfo`。如果文件不存在，则不会返回`FileInfo`结构，而是返回一个错误。`os.Stat()`可能返回多个错误，因此必须检查错误类型。标准库提供了一个名为`os.IsNotExist()`的函数，它将检查错误，以查看是否是因为文件不存在而引起的。

如果文件不存在，以下示例将调用`log.Fatal()`，但您可以优雅地处理错误，并在需要时继续而不退出：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   // Stat returns file info. It will return 
   // an error if there is no file. 
   fileInfo, err := os.Stat("test.txt") 
   if err != nil { 
      if os.IsNotExist(err) { 
         log.Fatal("File does not exist.") 
      } 
   } 
   log.Println("File does exist. File information:") 
   log.Println(fileInfo) 
} 
```

# 检查读取和写入权限

与前面的示例类似，通过检查错误使用名为`os.IsPermission()`的函数来检查读取和写入权限。如果错误是由于权限问题引起的，该函数将返回 true，如下例所示：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   // Test write permissions. It is possible the file 
   // does not exist and that will return a different 
   // error that can be checked with os.IsNotExist(err) 
   file, err := os.OpenFile("test.txt", os.O_WRONLY, 0666) 
   if err != nil { 
      if os.IsPermission(err) { 
         log.Println("Error: Write permission denied.") 
      } 
   } 
   file.Close() 

   // Test read permissions 
   file, err = os.OpenFile("test.txt", os.O_RDONLY, 0666) 
   if err != nil { 
      if os.IsPermission(err) { 
         log.Println("Error: Read permission denied.") 
      } 
   } 
   file.Close()
} 
```

# 更改权限、所有权和时间戳

如果您拥有文件或有相应的权限，可以更改所有权、时间戳和权限。标准库提供了一组函数。它们在这里给出：

+   `os.Chmod()`

+   `os.Chown()`

+   `os.Chtimes()`

以下示例演示了如何使用这些函数来更改文件的元数据。

```go
package main 

import ( 
   "log" 
   "os" 
   "time" 
) 

func main() { 
   // Change permissions using Linux style 
   err := os.Chmod("test.txt", 0777) 
   if err != nil { 
      log.Println(err) 
   } 

   // Change ownership 
   err = os.Chown("test.txt", os.Getuid(), os.Getgid()) 
   if err != nil { 
      log.Println(err) 
   } 

   // Change timestamps 
   twoDaysFromNow := time.Now().Add(48 * time.Hour) 
   lastAccessTime := twoDaysFromNow 
   lastModifyTime := twoDaysFromNow 
   err = os.Chtimes("test.txt", lastAccessTime, lastModifyTime) 
   if err != nil { 
      log.Println(err) 
   } 
} 
```

# 硬链接和符号链接

典型的文件只是硬盘上的一个指针，称为 inode。硬链接会创建一个指向相同位置的新指针。只有在删除所有指向文件的链接后，文件才会从磁盘中删除。硬链接只能在相同的文件系统上工作。硬链接是您可能认为是“正常”链接的东西。

符号链接或软链接有点不同，它不直接指向磁盘上的位置。符号链接只通过名称引用其他文件。它们可以指向不同文件系统上的文件。但是，并非所有系统都支持符号链接。

在历史上，Windows 对符号链接的支持并不好，但这些示例在 Windows 10 专业版中进行了测试，如果您拥有管理员权限，硬链接和符号链接都可以正常工作。要以管理员身份从命令行执行 Go 程序，首先右键单击命令提示符并选择以管理员身份运行。然后您可以执行程序，符号链接和硬链接将按预期工作。

以下示例演示了如何创建硬链接和符号链接文件，以及如何确定文件是否是符号链接，以及如何修改符号链接文件的元数据而不更改原始文件：

```go
package main 

import ( 
   "fmt" 
   "log" 
   "os" 
) 

func main() { 
   // Create a hard link 
   // You will have two file names that point to the same contents 
   // Changing the contents of one will change the other 
   // Deleting/renaming one will not affect the other 
   err := os.Link("original.txt", "original_also.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 

   fmt.Println("Creating symlink") 
   // Create a symlink 
   err = os.Symlink("original.txt", "original_sym.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 

   // Lstat will return file info, but if it is actually 
   // a symlink, it will return info about the symlink. 
   // It will not follow the link and give information 
   // about the real file 
   // Symlinks do not work in Windows 
   fileInfo, err := os.Lstat("original_sym.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Printf("Link info: %+v", fileInfo) 

   // Change ownership of a symlink only 
   // and not the file it points to 
   err = os.Lchown("original_sym.txt", os.Getuid(), os.Getgid()) 
   if err != nil { 
      log.Fatal(err) 
   } 
} 
```

# 读写

读写文件可以通过多种方式完成。Go 提供了接口，使得编写自己的函数来处理文件或任何其他读取/写入接口变得容易。

通过`os`、`io`和`ioutil`包，您可以找到适合您需求的正确函数。这些示例涵盖了许多可用选项。

# 复制文件

以下示例使用`io.Copy()`函数将内容从一个读取器复制到另一个写入器：

```go
package main 

import ( 
   "io" 
   "log" 
   "os" 
) 

func main() { 
   // Open original file 
   originalFile, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer originalFile.Close() 

   // Create new file 
   newFile, err := os.Create("test_copy.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer newFile.Close() 

   // Copy the bytes to destination from source 
   bytesWritten, err := io.Copy(newFile, originalFile) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Copied %d bytes.", bytesWritten) 

   // Commit the file contents 
   // Flushes memory to disk 
   err = newFile.Sync() 
   if err != nil { 
      log.Fatal(err) 
   }  
} 
```

# 在文件中寻找位置

`Seek()`函数用于将文件光标设置在特定位置。默认情况下，它从偏移量 0 开始，并随着读取字节而向前移动。您可能希望将光标重置到文件的开头，或者直接跳转到特定位置。`Seek()`函数允许您执行此操作。

`Seek()`接受两个参数。第一个是距离，即你想要以字节为单位移动光标。它可以通过正整数向前移动，或者通过提供负数向文件后退。第一个参数，即距离，是一个相对值，而不是文件中的绝对位置。第二个参数指定了相对点的起始位置，称为`whence`。`whence`参数是相对偏移的参考点。它可以是`0`、`1`或`2`，分别表示文件的开头、当前位置和文件的结尾。

例如，如果指定了`Seek(-1, 2)`，它将把文件光标设置在文件末尾的前一个字节。`Seek(2, 0)`将在`file.Seek(5, 1)`的开始处寻找第二个字节，这将使光标从当前位置向前移动 5 个字节：

```go
package main 

import ( 
   "fmt" 
   "log" 
   "os" 
) 

func main() { 
   file, _ := os.Open("test.txt") 
   defer file.Close() 

   // Offset is how many bytes to move 
   // Offset can be positive or negative 
   var offset int64 = 5 

   // Whence is the point of reference for offset 
   // 0 = Beginning of file 
   // 1 = Current position 
   // 2 = End of file 
   var whence int = 0 
   newPosition, err := file.Seek(offset, whence) 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Println("Just moved to 5:", newPosition) 

   // Go back 2 bytes from current position 
   newPosition, err = file.Seek(-2, 1) 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Println("Just moved back two:", newPosition) 

   // Find the current position by getting the 
   // return value from Seek after moving 0 bytes 
   currentPosition, err := file.Seek(0, 1) 
   fmt.Println("Current position:", currentPosition) 

   // Go to beginning of file 
   newPosition, err = file.Seek(0, 0) 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Println("Position after seeking 0,0:", newPosition) 
} 
```

# 向文件写入字节

使用`os`包就可以进行写入操作，因为打开文件时已经需要它。由于所有的 Go 可执行文件都是静态链接的二进制文件，每导入一个包都会增加可执行文件的大小。其他包如`io`、`ioutil`和`bufio`提供了一些帮助，但并非必需品：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   // Open a new file for writing only 
   file, err := os.OpenFile( 
      "test.txt", 
      os.O_WRONLY|os.O_TRUNC|os.O_CREATE, 
      0666, 
   ) 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer file.Close() 

   // Write bytes to file 
   byteSlice := []byte("Bytes!\n") 
   bytesWritten, err := file.Write(byteSlice) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Wrote %d bytes.\n", bytesWritten) 
} 
```

# 快速写入文件

`ioutil`包有一个有用的函数叫做`WriteFile()`，它将处理创建/打开、写入字节片段和关闭。如果您只需要一种快速的方法将字节片段转储到文件中，这将非常有用：

```go
package main 

import ( 
   "io/ioutil" 
   "log" 
) 

func main() { 
   err := ioutil.WriteFile("test.txt", []byte("Hi\n"), 0666) 
   if err != nil { 
      log.Fatal(err) 
   } 
} 
```

# 带缓冲的写入器

`bufio`包允许您创建一个带缓冲的写入器，以便您可以在将其写入磁盘之前在内存中处理缓冲区。如果您需要在将数据写入磁盘之前对数据进行大量操作以节省磁盘 IO 时间，则这是有用的。如果您一次只写入一个字节，并且希望在将其一次性存储到文件之前在内存缓冲区中存储大量数据，否则您将为每个字节执行磁盘 IO。这会对您的磁盘造成磨损，并减慢进程速度。

可以检查缓冲写入器，查看它当前存储了多少未缓冲的数据，以及剩余多少缓冲空间。缓冲区也可以重置以撤消自上次刷新以来的任何更改。缓冲区也可以调整大小。

以下示例打开名为`test.txt`的文件，并创建一个包装文件对象的缓冲写入器。一些字节被写入缓冲区，然后写入一个字符串。然后检查内存缓冲区，然后将缓冲区的内容刷新到磁盘上的文件。它还演示了如何重置缓冲区，撤消尚未刷新的任何更改，以及如何检查缓冲区中剩余的空间。最后，它演示了如何将缓冲区的大小调整为特定大小：

```go
package main 

import ( 
   "bufio" 
   "log" 
   "os" 
) 

func main() { 
   // Open file for writing 
   file, err := os.OpenFile("test.txt", os.O_WRONLY, 0666) 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer file.Close() 

   // Create a buffered writer from the file 
   bufferedWriter := bufio.NewWriter(file) 

   // Write bytes to buffer 
   bytesWritten, err := bufferedWriter.Write( 
      []byte{65, 66, 67}, 
   ) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Bytes written: %d\n", bytesWritten) 

   // Write string to buffer 
   // Also available are WriteRune() and WriteByte() 
   bytesWritten, err = bufferedWriter.WriteString( 
      "Buffered string\n", 
   ) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Bytes written: %d\n", bytesWritten) 

   // Check how much is stored in buffer waiting 
   unflushedBufferSize := bufferedWriter.Buffered() 
   log.Printf("Bytes buffered: %d\n", unflushedBufferSize) 

   // See how much buffer is available 
   bytesAvailable := bufferedWriter.Available() 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Available buffer: %d\n", bytesAvailable) 

   // Write memory buffer to disk 
   bufferedWriter.Flush() 

   // Revert any changes done to buffer that have 
   // not yet been written to file with Flush() 
   // We just flushed, so there are no changes to revert 
   // The writer that you pass as an argument 
   // is where the buffer will output to, if you want 
   // to change to a new writer 
   bufferedWriter.Reset(bufferedWriter) 

   // See how much buffer is available 
   bytesAvailable = bufferedWriter.Available() 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Available buffer: %d\n", bytesAvailable) 

   // Resize buffer. The first argument is a writer 
   // where the buffer should output to. In this case 
   // we are using the same buffer. If we chose a number 
   // that was smaller than the existing buffer, like 10 
   // we would not get back a buffer of size 10, we will 
   // get back a buffer the size of the original since 
   // it was already large enough (default 4096) 
   bufferedWriter = bufio.NewWriterSize( 
      bufferedWriter, 
      8000, 
   ) 

   // Check available buffer size after resizing 
   bytesAvailable = bufferedWriter.Available() 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Available buffer: %d\n", bytesAvailable) 
} 
```

# 从文件中读取最多 n 个字节

`os.File`类型带有一些基本函数。其中之一是`File.Read()`。`Read()`需要传递一个字节切片作为参数。字节从文件中读取并放入字节切片中。`Read()`将尽可能多地读取字节，直到缓冲区填满，然后停止读取。

在调用`Read()`之前，可能需要多次调用`Read()`，具体取决于提供的缓冲区大小和文件的大小。如果在调用`Read()`期间到达文件的末尾，则会返回一个`io.EOF`错误：

```go
package main 

import ( 
   "log" 
   "os" 
) 

func main() { 
   // Open file for reading 
   file, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer file.Close() 

   // Read up to len(b) bytes from the File 
   // Zero bytes written means end of file 
   // End of file returns error type io.EOF 
   byteSlice := make([]byte, 16) 
   bytesRead, err := file.Read(byteSlice) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Number of bytes read: %d\n", bytesRead) 
   log.Printf("Data read: %s\n", byteSlice) 
} 
```

# 读取确切的 n 个字节

在前面的例子中，如果`File.Read()`只包含 10 个字节的文件，但您提供了一个包含 500 个字节的字节切片缓冲区，它将不会返回错误。有些情况下，您希望确保整个缓冲区都被填满。如果整个缓冲区没有被填满，`io.ReadFull()`函数将返回错误。如果`io.ReadFull()`没有任何数据可读，将返回 EOF 错误。如果它读取了一些数据，然后遇到 EOF，它将返回`ErrUnexpectedEOF`错误：

```go
package main 

import ( 
   "io" 
   "log" 
   "os" 
) 

func main() { 
   // Open file for reading 
   file, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 

   // The file.Read() function will happily read a tiny file in to a    
   // large byte slice, but io.ReadFull() will return an 
   // error if the file is smaller than the byte slice. 
   byteSlice := make([]byte, 2) 
   numBytesRead, err := io.ReadFull(file, byteSlice) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Number of bytes read: %d\n", numBytesRead) 
   log.Printf("Data read: %s\n", byteSlice) 
} 
```

# 至少读取 n 个字节

`io`包提供的另一个有用函数是`io.ReadAtLeast()`。如果至少没有指定数量的字节，则会返回错误。与`io.ReadFull()`类似，如果没有找到数据，则返回`EOF`错误，如果在遇到文件结束之前读取了一些数据，则返回`ErrUnexpectedEOF`错误：

```go
package main 

import ( 
   "io" 
   "log" 
   "os" 
) 

func main() { 
   // Open file for reading 
   file, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 

   byteSlice := make([]byte, 512) 
   minBytes := 8 
   // io.ReadAtLeast() will return an error if it cannot 
   // find at least minBytes to read. It will read as 
   // many bytes as byteSlice can hold. 
   numBytesRead, err := io.ReadAtLeast(file, byteSlice, minBytes) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Number of bytes read: %d\n", numBytesRead) 
   log.Printf("Data read: %s\n", byteSlice) 
} 
```

# 读取文件的所有字节

`ioutil`包提供了一个函数，可以读取文件中的每个字节并将其作为字节切片返回。这个函数很方便，因为在进行读取之前不必定义字节切片。缺点是，一个非常大的文件将返回一个可能比预期更大的大切片。

`io.ReadAll()`函数期望一个已经用`os.Open()`或`Create()`打开的文件：

```go
package main 

import ( 
   "fmt" 
   "io/ioutil" 
   "log" 
   "os" 
) 

func main() { 
   // Open file for reading 
   file, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 

   // os.File.Read(), io.ReadFull(), and 
   // io.ReadAtLeast() all work with a fixed 
   // byte slice that you make before you read 

   // ioutil.ReadAll() will read every byte 
   // from the reader (in this case a file), 
   // and return a slice of unknown slice 
   data, err := ioutil.ReadAll(file) 
   if err != nil { 
      log.Fatal(err) 
   } 

   fmt.Printf("Data as hex: %x\n", data) 
   fmt.Printf("Data as string: %s\n", data) 
   fmt.Println("Number of bytes read:", len(data)) 
} 
```

# 快速将整个文件读取到内存中

与前面示例中的`io.ReadAll()`函数类似，`io.ReadFile()`将读取文件中的所有字节并返回一个字节切片。两者之间的主要区别在于`io.ReadFile()`期望一个文件路径，而不是已经打开的文件对象。`io.ReadFile()`函数将负责打开、读取和关闭文件。您只需提供一个文件名，它就会提供字节。这通常是加载文件数据的最快最简单的方法。

虽然这种方法非常方便，但它有一些限制；因为它直接将整个文件读取到内存中，非常大的文件可能会耗尽系统的内存限制：

```go
package main 

import ( 
   "io/ioutil" 
   "log" 
) 

func main() { 
   // Read file to byte slice 
   data, err := ioutil.ReadFile("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 

   log.Printf("Data read: %s\n", data) 
} 
```

# 缓冲读取器

创建一个缓冲读取器将存储一些内容的内存缓冲区。缓冲读取器还提供了一些`os.File`或`io.Reader`类型上不可用的其他函数。默认缓冲区大小为 4096，最小大小为 16。缓冲读取器提供了一组有用的函数。一些可用的函数包括但不限于以下内容：

+   `Read()`: 这是将数据读入字节切片

+   `Peek()`: 这是在不移动文件光标的情况下检查下一个字节

+   `ReadByte()`: 这是读取单个字节

+   `UnreadByte()`: 这会取消上次读取的最后一个字节

+   `ReadBytes()`: 这会读取字节，直到达到指定的分隔符

+   `ReadString()`: 这会读取字符串，直到达到指定的分隔符

以下示例演示了如何使用缓冲读取器从文件获取数据。首先，它打开一个文件，然后创建一个包装文件对象的缓冲读取器。一旦缓冲读取器准备好了，它就展示了如何使用前面的函数：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "log" 
   "os" 
) 

func main() { 
   // Open file and create a buffered reader on top 
   file, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   bufferedReader := bufio.NewReader(file) 

   // Get bytes without advancing pointer 
   byteSlice := make([]byte, 5) 
   byteSlice, err = bufferedReader.Peek(5) 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Printf("Peeked at 5 bytes: %s\n", byteSlice) 

   // Read and advance pointer 
   numBytesRead, err := bufferedReader.Read(byteSlice) 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Printf("Read %d bytes: %s\n", numBytesRead, byteSlice) 

   // Ready 1 byte. Error if no byte to read 
   myByte, err := bufferedReader.ReadByte() 
   if err != nil { 
      log.Fatal(err) 
   }  
   fmt.Printf("Read 1 byte: %c\n", myByte) 

   // Read up to and including delimiter 
   // Returns byte slice 
   dataBytes, err := bufferedReader.ReadBytes('\n') 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Printf("Read bytes: %s\n", dataBytes) 

   // Read up to and including delimiter 
   // Returns string 
   dataString, err := bufferedReader.ReadString('\n') 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Printf("Read string: %s\n", dataString) 

   // This example reads a few lines so test.txt 
   // should have a few lines of text to work correct 
} 
```

# 使用缓冲读取器读取

Scanner 是`bufio`包的一部分。它对于在特定分隔符处逐步浏览文件很有用。通常，换行符被用作分隔符来按行分割文件。在 CSV 文件中，逗号将是分隔符。`os.File`对象可以像缓冲读取器一样包装在`bufio.Scanner`对象中。我们将调用`Scan()`来读取到下一个分隔符，然后使用`Text()`或`Bytes()`来获取已读取的数据。

分隔符不仅仅是一个简单的字节或字符。实际上有一个特殊的函数，您必须实现它，它将确定下一个分隔符在哪里，向前推进指针的距离以及要返回的数据。如果没有提供自定义的`SplitFunc`类型，则默认为`ScanLines`，它将在每个换行符处分割。`bufio`中包含的其他分割函数有`ScanRunes`和`ScanWords`。

要定义自己的分割函数，请定义一个与此指纹匹配的函数：

```go
type SplitFuncfunc(data []byte, atEOF bool) (advance int, token []byte, 
   err error)
```

返回（`0`，`nil`，`nil`）将告诉扫描器再次扫描，但使用更大的缓冲区，因为没有足够的数据达到分隔符。

在下面的示例中，从文件创建了`bufio.Scanner`，然后逐字扫描文件：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "log" 
   "os" 
) 

func main() { 
   // Open file and create scanner on top of it 
   file, err := os.Open("test.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   scanner := bufio.NewScanner(file) 

   // Default scanner is bufio.ScanLines. Lets use ScanWords. 
   // Could also use a custom function of SplitFunc type 
   scanner.Split(bufio.ScanWords) 

   // Scan for next token. 
   success := scanner.Scan() 
   if success == false { 
      // False on error or EOF. Check error 
      err = scanner.Err() 
      if err == nil { 
         log.Println("Scan completed and reached EOF") 
      } else { 
         log.Fatal(err) 
      } 
   } 

   // Get data from scan with Bytes() or Text() 
   fmt.Println("First word found:", scanner.Text()) 

   // Call scanner.Scan() manually, or loop with for 
   for scanner.Scan() { 
      fmt.Println(scanner.Text()) 
   } 
} 
```

# 存档

存档是一种存储多个文件的文件格式。最常见的两种存档格式是 tar 文件和 ZIP 存档。Go 标准库有`tar`和`zip`包。这些示例使用 ZIP 格式，但 tar 格式可以很容易地互换。

# 存档（ZIP）文件

以下示例演示了如何创建一个包含多个文件的存档。示例中的文件是硬编码的，只有几个字节，但应该很容易适应其他需求：

```go
// This example uses zip but standard library 
// also supports tar archives 
package main 

import ( 
   "archive/zip" 
   "log" 
   "os" 
) 

func main() { 
   // Create a file to write the archive buffer to 
   // Could also use an in memory buffer. 
   outFile, err := os.Create("test.zip") 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer outFile.Close() 

   // Create a zip writer on top of the file writer 
   zipWriter := zip.NewWriter(outFile) 

   // Add files to archive 
   // We use some hard coded data to demonstrate, 
   // but you could iterate through all the files 
   // in a directory and pass the name and contents 
   // of each file, or you can take data from your 
   // program and write it write in to the archive without 
   var filesToArchive = []struct { 
      Name, Body string 
   }{ 
      {"test.txt", "String contents of file"}, 
      {"test2.txt", "\x61\x62\x63\n"}, 
   } 

   // Create and write files to the archive, which in turn 
   // are getting written to the underlying writer to the 
   // .zip file we created at the beginning 
   for _, file := range filesToArchive { 
      fileWriter, err := zipWriter.Create(file.Name) 
      if err != nil { 
         log.Fatal(err) 
      } 
      _, err = fileWriter.Write([]byte(file.Body)) 
      if err != nil { 
         log.Fatal(err) 
      } 
   } 

   // Clean up 
   err = zipWriter.Close() 
   if err != nil { 
      log.Fatal(err) 
   } 
} 
```

# 提取（解压）存档文件

以下示例演示了如何解压 ZIP 格式文件。它将通过创建必要的目录来复制存档中找到的目录结构：

```go
// This example uses zip but standard library 
// also supports tar archives 
package main 

import ( 
   "archive/zip" 
   "io" 
   "log" 
   "os" 
   "path/filepath" 
) 

func main() { 
   // Create a reader out of the zip archive 
   zipReader, err := zip.OpenReader("test.zip") 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer zipReader.Close() 

   // Iterate through each file/dir found in 
   for _, file := range zipReader.Reader.File { 
      // Open the file inside the zip archive 
      // like a normal file 
      zippedFile, err := file.Open() 
      if err != nil { 
         log.Fatal(err) 
      } 
      defer zippedFile.Close() 

      // Specify what the extracted file name should be. 
      // You can specify a full path or a prefix 
      // to move it to a different directory. 
      // In this case, we will extract the file from 
      // the zip to a file of the same name. 
      targetDir := "./" 
      extractedFilePath := filepath.Join( 
         targetDir, 
         file.Name, 
      ) 

      // Extract the item (or create directory) 
      if file.FileInfo().IsDir() { 
         // Create directories to recreate directory 
         // structure inside the zip archive. Also 
         // preserves permissions 
         log.Println("Creating directory:", extractedFilePath) 
         os.MkdirAll(extractedFilePath, file.Mode()) 
      } else { 
         // Extract regular file since not a directory 
         log.Println("Extracting file:", file.Name) 

         // Open an output file for writing 
         outputFile, err := os.OpenFile( 
            extractedFilePath, 
            os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 
            file.Mode(), 
         ) 
         if err != nil { 
            log.Fatal(err) 
         } 
         defer outputFile.Close() 

         // "Extract" the file by copying zipped file 
         // contents to the output file 
         _, err = io.Copy(outputFile, zippedFile) 
         if err != nil { 
            log.Fatal(err) 
         } 
      }  
   } 
} 
```

# 压缩

Go 标准库还支持压缩，这与存档不同。通常，存档和压缩结合在一起，将大量文件打包成一个紧凑的文件。最常见的格式可能是`.tar.gz`文件，这是一个 gzipped tar 文件。不要混淆 zip 和 gzip，它们是两种不同的东西。

Go 标准库支持多种压缩算法：

+   **bzip2**：bzip2 格式

+   **flate**：DEFLATE（RFC 1951）

+   **gzip**：gzip 格式（RFC 1952）

+   **lzw**：来自《高性能数据压缩技术，计算机，17（6）（1984 年 6 月），第 8-19 页》的 Lempel-Ziv-Welch 格式

+   **zlib**：zlib 格式（RFC 1950）

在[`golang.org/pkg/compress/`](https://golang.org/pkg/compress/)中阅读有关每个包的更多信息。这些示例使用 gzip 压缩，但应该很容易地互换上述任何包。

# 压缩文件

以下示例演示了如何使用`gzip`包压缩文件：

```go
// This example uses gzip but standard library also 
// supports zlib, bz2, flate, and lzw 
package main 

import ( 
   "compress/gzip" 
   "log" 
   "os" 
) 

func main() { 
   // Create .gz file to write to 
   outputFile, err := os.Create("test.txt.gz") 
   if err != nil { 
      log.Fatal(err) 
   } 

   // Create a gzip writer on top of file writer 
   gzipWriter := gzip.NewWriter(outputFile) 
   defer gzipWriter.Close() 

   // When we write to the gzip writer 
   // it will in turn compress the contents 
   // and then write it to the underlying 
   // file writer as well 
   // We don't have to worry about how all 
   // the compression works since we just 
   // use it as a simple writer interface 
   // that we send bytes to 
   _, err = gzipWriter.Write([]byte("Gophers rule!\n")) 
   if err != nil { 
      log.Fatal(err) 
   } 

   log.Println("Compressed data written to file.") 
} 
```

# 解压文件

以下示例演示了如何使用`gzip`算法解压文件：

```go
// This example uses gzip but standard library also 
// supports zlib, bz2, flate, and lzw 
package main 

import ( 
   "compress/gzip" 
   "io" 
   "log" 
   "os" 
) 

func main() { 
   // Open gzip file that we want to uncompress 
   // The file is a reader, but we could use any 
   // data source. It is common for web servers 
   // to return gzipped contents to save bandwidth 
   // and in that case the data is not in a file 
   // on the file system but is in a memory buffer 
   gzipFile, err := os.Open("test.txt.gz") 
   if err != nil { 
      log.Fatal(err) 
   } 

   // Create a gzip reader on top of the file reader 
   // Again, it could be any type reader though 
   gzipReader, err := gzip.NewReader(gzipFile) 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer gzipReader.Close() 

   // Uncompress to a writer. We'll use a file writer 
   outfileWriter, err := os.Create("unzipped.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer outfileWriter.Close() 

   // Copy contents of gzipped file to output file 
   _, err = io.Copy(outfileWriter, gzipReader) 
   if err != nil { 
      log.Fatal(err) 
   } 
} 
```

在结束关于文件处理的这一章之前，让我们看两个可能有用的实际示例。当您不想创建永久文件但需要一个文件进行操作时，临时文件和目录是有用的。此外，通过互联网下载文件是获取文件的常见方式。下面的示例演示了这些操作。

# 创建临时文件和目录

`ioutil`包提供了两个函数：`TempDir()`和`TempFile()`。调用者有责任在完成后删除临时项目。这些函数提供的唯一好处是，您可以将空字符串传递给目录，它将自动在系统的默认临时文件夹（在 Linux 上为`/tmp`）中创建该项目，因为`os.TempDir()`函数将返回默认的系统临时目录：

```go
package main 

import ( 
   "fmt" 
   "io/ioutil" 
   "log" 
   "os" 
) 

func main() { 
   // Create a temp dir in the system default temp folder 
   tempDirPath, err := ioutil.TempDir("", "myTempDir") 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Println("Temp dir created:", tempDirPath) 

   // Create a file in new temp directory 
   tempFile, err := ioutil.TempFile(tempDirPath, "myTempFile.txt") 
   if err != nil { 
      log.Fatal(err) 
   } 
   fmt.Println("Temp file created:", tempFile.Name()) 

   // ... do something with temp file/dir ... 

   // Close file 
   err = tempFile.Close() 
   if err != nil { 
      log.Fatal(err) 
   } 

   // Delete the resources we created 
   err = os.Remove(tempFile.Name()) 
   if err != nil { 
      log.Fatal(err) 
   } 
   err = os.Remove(tempDirPath) 
   if err != nil { 
      log.Fatal(err) 
   } 
}
```

# 通过 HTTP 下载文件

现代计算中的常见任务是通过 HTTP 协议下载文件。以下示例显示了如何快速将特定 URL 下载到文件中。

其他常见的工具包括`curl`和`wget`：

```go
package main 

import ( 
   "io" 
   "log" 
   "net/http" 
   "os" 
) 

func main() { 
   // Create output file 
   newFile, err := os.Create("devdungeon.html") 
   if err != nil { 
      log.Fatal(err) 
   } 
   defer newFile.Close() 

   // HTTP GET request devdungeon.com 
   url := "http://www.devdungeon.com/archive" 
   response, err := http.Get(url) 
   defer response.Body.Close() 

   // Write bytes from HTTP response to file. 
   // response.Body satisfies the reader interface. 
   // newFile satisfies the writer interface. 
   // That allows us to use io.Copy which accepts 
   // any type that implements reader and writer interface 
   numBytesWritten, err := io.Copy(newFile, response.Body) 
   if err != nil { 
      log.Fatal(err) 
   } 
   log.Printf("Downloaded %d byte file.\n", numBytesWritten) 
} 
```

# 总结

阅读完本章后，您现在应该熟悉了一些与文件交互的不同方式，并且可以轻松执行基本操作。目标不是要记住所有这些函数名，而是要意识到有哪些工具可用。如果您需要示例代码，本章可以用作参考，但我鼓励您创建一个类似这样的代码库。

有用的文件函数分布在多个包中。`os`包仅包含与文件的基本操作，如打开、关闭和简单读取。`io`包提供了可以在读取器和写入器接口上使用的函数，比`os`包更高级。`ioutil`包提供了更高级别的便利函数，用于处理文件。

在下一章中，我们将涵盖取证的主题。它将涵盖诸如寻找异常大或最近修改的文件之类的内容。除了文件取证，我们还将涵盖一些网络取证调查的主题，即查找主机名、IP 和主机的 MX 记录。取证章节还涵盖了隐写术的基本示例，展示了如何在图像中隐藏数据以及如何在图像中查找隐藏的数据。
