# 第三章：介绍 Docker

在我们继续本书的内容之前，我们需要先了解一下被称为 Docker 的这个小东西，在我们开始之前，别忘了克隆示例代码仓库 [`github.com/building-microservices-with-go/chapter3.git`](https://github.com/building-microservices-with-go/chapter3.git)。

# 使用 Docker 介绍容器

Docker 是一个在过去三年中崛起的平台；它诞生于简化构建、运输和运行应用程序过程的愿望。Docker 不是容器的发明者，Jacques Gélinas 在 2001 年创建了 VServer 项目，从那时起，其他主要项目还包括 IBM 的 LXC 和 CoreOS 的 rkt。

如果你想了解更多关于历史的信息，我推荐阅读 Redhat 的这篇优秀的博客文章：[`rhelblog.redhat.com/2015/08/28/the-history-of-containers`](http://rhelblog.redhat.com/2015/08/28/the-history-of-containers)，本节将重点介绍 Docker，这是目前最受欢迎的技术。

容器的概念是进程隔离和应用程序打包。引用 Docker 的话：

容器镜像是一个轻量级、独立、可执行的软件包，它包含了运行该软件所需的一切：代码、运行时、系统工具、系统库、设置。

...

容器将软件与其环境隔离开来，例如，开发环境和预发布环境之间的差异，并有助于减少在相同基础设施上运行不同软件的团队之间的冲突。

它们在应用开发中的好处是，在部署这些应用程序时，我们可以利用这一点，因为它允许我们将它们打包得更紧密，节省硬件资源。

从开发和测试生命周期来看，容器使我们能够在开发机器上运行生产代码，而无需复杂的设置；它还允许我们创建一个“洁净室”环境，而无需安装不同实例的同数据库来测试新软件。

容器已成为打包微服务的首选方案，随着我们在本书中的示例进展，你将了解到这对你的工作流程是多么宝贵。

容器通过隔离进程和文件系统来工作。除非明确指定，否则容器无法访问彼此的文件系统。除非再次指定，否则它们也不能通过 TCP 或 UDP 套接字相互交互。

Docker 由许多部分组成；然而，其核心是 Docker 引擎，这是一个轻量级的应用程序运行时，具有编排、调度、网络和安全功能。Docker 引擎可以安装在物理或虚拟主机上的任何位置，并且支持 Windows 和 Linux。容器允许开发者将大量或少量代码及其依赖项打包到一个隔离的包中。

我们还可以从大量的预先创建的镜像中获取资源，几乎所有的软件供应商，从 MySQL 到 IBM 的 WebSphere，都有官方镜像可供我们使用。

Docker 也使用 Go 语言，实际上，几乎所有的 Docker Engine 和其他应用程序的代码都是用 Go 编写的。

我们不如通过实例来检查每个功能，而不是写一篇关于 Docker 如何工作的论文。到本章结束时，我们将使用在第一章，“微服务简介”中创建的一个简单示例，并为它创建一个 Docker 镜像。

# 安装 Docker

前往[`docs.docker.com/engine/installation/`](https://docs.docker.com/engine/installation/)，并在你的机器上安装正确的 Docker 版本。你将找到适用于 Mac、Windows 和 Linux 的版本。

# 运行我们的第一个容器

为了验证 Docker 是否正确安装，让我们运行我们的第一个容器，**hello-world**实际上是一个镜像，镜像是一个不可变的容器快照。一旦我们用以下命令启动它们，它们就变成了容器，把它想象成类型和实例，类型定义了构成行为的字段和方法。实例是这个类型的活生生的实例化，你可以将其他类型分配给字段并调用方法来执行操作。

```go
$ docker run --rm hello-world  

```

你首先应该看到的是：

```go
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c04b14da8d14: Pull complete 
Digest: sha256:0256e8a36e2070f7bf2d0b0763dbabdd67798512411de4cdcf9431a1feb60fd9
Status: Downloaded newer image for hello-world:latest

```

当你执行`docker run`时，引擎首先会检查你是否已经安装了镜像。如果没有，它会连接到默认的注册表，在这种情况下，[`hub.docker.com/`](https://hub.docker.com/)，以检索它。

一旦镜像被下载，守护进程可以从下载的镜像创建一个容器，所有输出都会流到你的终端上：

```go
Hello from Docker!
This message shows that your installation appears to be working correctly.  

```

`--rm`标志告诉 Docker 引擎在退出时删除容器并删除它所使用的任何资源，如卷。除非我们想在某个时候重新启动容器，否则使用`--rm`标志来保持我们的文件系统干净是一个好习惯，否则，所有创建的临时卷都会存在并占用空间。让我们尝试一个稍微复杂一些的例子，这次，我们将启动一个容器并在其中创建一个 shell 来展示如何导航到内部文件系统。在你的终端中执行以下命令： 

```go
$ docker run -it --rm alpine:latest sh  

```

Alpine 是 Linux 的一个轻量级版本，非常适合运行 Go 应用程序。`-it`标志代表**交互式终端**，它将你的终端的标准输入映射到正在运行的容器的输入。在我们要运行的镜像名称之后的`sh`语句是我们希望在容器启动时执行的命令的名称。

如果一切顺利，你现在应该已经在一个容器的 shell 中了。如果你通过执行`ls`命令检查当前目录，你会看到以下内容，希望这并不是你在运行命令之前的目录：

```go
bin      etc      lib      media    proc     run      srv      tmp      var
dev      home     linuxrc  mnt      root     sbin     sys      usr

```

这是新启动的容器的根文件夹，容器是不可变的，所以你对运行中的容器文件系统所做的任何更改，在容器停止时都会被丢弃。虽然这看起来可能是一个问题，但有一些持久化数据的方法，我们稍后会看看，但现在，重要的是要记住：

“容器是不可变的镜像实例，并且默认情况下数据卷是非持久的”

在设计你的服务时，你需要记住这一点，为了说明这是如何工作的，请看这个简单的例子。

打开另一个终端并执行以下命令：

```go
$ docker ps  

```

你应该看到以下输出：

```go
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
43a1bea0009e        alpine:latest       "sh"                6 minutes ago       Up 6 minutes                            tiny_galileo

```

`docker ps` 命令查询引擎并返回一个容器列表，默认情况下这仅显示正在运行的容器，然而，如果我们添加 `-a` 标志，我们也可以看到停止的容器。

我们之前启动的 Alpine Linux 容器目前正在运行，所以回到你之前的终端窗口，在根文件系统中创建一个文件：

```go
$ touch mytestfile.txt  

```

如果我们再次列出目录结构，我们可以看到在文件系统的根目录下创建了一个文件：

```go
bin             lib             mytestfile.txt  sbin            usr
dev             linuxrc         proc            srv             var
etc             media           root            sys
home            mnt             run             tmp  

```

现在使用 `exit` 命令退出容器，并再次运行 `docker ps`，你应该看到以下输出：

```go
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES  

```

如果我们添加 `-a` 标志命令来查看停止的容器，我们也应该看到我们之前启动的容器：

```go
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
518c8ae7fc94        alpine:latest       "sh"                5 seconds ago       Exited (0) 2 seconds ago                       pensive_perlman  

```

现在，再次使用 `docker run` 命令启动另一个容器，并列出根文件夹中的目录内容。

没有 `mytestfile.txt` 吧？这个文件不存在的原因是因为我们之前讨论的原则，我认为再次提到这一点很重要，因为如果你是第一次使用 Docker，这可能会让你感到意外：

“容器是不可变的镜像实例，并且默认情况下数据卷是非持久的。”

然而，有一点值得注意，除非你明确删除容器，否则它将保持在 Docker 主机上的停止状态。

删除容器有两个重要原因；第一个是如果你不记得这一点，你很快就会填满主机的磁盘，因为每次你创建一个容器，Docker 都会在主机上为容器卷分配空间。第二个原因是容器可以被重启。

重启听起来很酷，实际上，这是一个方便的功能，不是你应该在生产环境中使用的东西，为此你需要记住黄金法则并相应地设计你的应用程序：

“容器是不可变的镜像实例，并且默认情况下数据卷是非持久的。”

然而，Docker 的使用远远超出了仅仅为你的微服务运行应用程序。这是一个管理你的开发依赖项而不会让你的开发机器变得杂乱无章的绝佳方式。我们稍后会看看这一点，但现在，我们感兴趣的是我们如何重启一个停止的容器。

如果我们执行`docker ps -a`命令，我们会看到现在有两个停止的容器。最老的一个是我们第一次启动并添加了`mytestfile.txt`的容器。这是我们想要重新启动的容器，所以获取容器的 ID 并执行以下命令：

```go
$ docker start -it [container_id] sh  

```

再次强调，如果你检查容器的目录内容，你应该在容器的根目录下的 shell 中，你认为你会找到什么？

没错，`mytestfile.txt`；这是因为当你重新启动容器时，引擎重新挂载了你在第一次运行命令时附加的卷。这些就是之前提到的你修改以添加文件的相同卷。

因此我们可以重新启动我们的容器；然而，我只想最后一次重复黄金法则：

"容器是镜像的不变实例，数据卷默认是非持久的。"

在生产环境中运行时，你不能确保可以重新启动一个容器。这里有成千上万的原因，其中一个主要的原因是我们将在查看编排时更深入地探讨，那就是容器通常运行在一组主机上。由于无法保证容器将在哪个主机上重新启动，甚至无法保证容器之前运行的主机实际上存在。有许多项目试图解决这个问题，但最好的方法是完全避免这种复杂性。如果你需要持久化文件，那么将它们存储在为这项工作设计的某些东西中，比如 Amazon S3 或 Google Cloud Storage。围绕这个原则设计你的应用程序，这样当不可避免的事情发生时，你将花更少的时间惊慌，而且你的超级敏感数据容器不会消失。

好的，在我们更深入地了解 Docker 卷之前，让我们清理一下。

退出你的容器并回到 Docker 主机上的 shell。如果我们运行`docker ps -a`，我们会看到有两个停止的容器。为了删除这些容器，我们可以使用`docker rm containerid`命令。

现在使用你列表中的第一个`containerid`运行此命令，如果成功，你请求删除的容器 ID 将被回显给你，并且容器将被删除。

如果你想要删除所有停止的容器，可以使用以下命令：

```go
$ docker rm -v $(docker ps -a -q)  

```

`docker ps -a -q`，`-a`标志将列出所有容器，包括停止的容器，`-q`将返回容器 ID 的列表而不是完整详情。我们将此作为参数列表传递给`docker rm`，它将删除列表中的所有容器。

为了避免需要删除容器，我们可以在启动新容器时使用`--rm`标志。此标志告诉 Docker 在容器停止时删除它。

# Docker 卷

我们已经看到了 Docker 容器是不可变的；然而，在某些情况下，您可能希望将一些文件写入磁盘，或者当您想要从磁盘读取数据，例如在开发环境中。Docker 有卷的概念，可以从运行 Docker 机器的主机或另一个 Docker 容器挂载。

# 联合文件系统

为了保持我们的镜像高效且紧凑，Docker 使用了联合文件系统的概念。联合文件系统允许我们通过将不同的目录和或文件组合在一起来表示一个逻辑文件系统。它使用**写时复制**技术，在我们修改文件系统时复制层，这样我们在创建新镜像时只使用大约 1MB 的空间。当数据写入文件系统时，Docker 会复制层并将其放在栈的顶部。在构建镜像和扩展现有镜像时，我们利用了这项技术，同样，在启动镜像和创建容器时，唯一的区别就是这个可写层，这意味着我们不需要每次都复制所有层并填满我们的磁盘。

# 挂载卷

`-v` 或 `--volume` 参数允许您指定一对值，这对值对应于您希望在主机上挂载的文件系统以及您希望在容器内部挂载卷的路径。

让我们尝试之前的示例，但这次是在本地文件系统上挂载一个卷：

```go
$ docker run -it -v $(pwd):/host alpine:latest /bin/sh  

```

如果您切换到主机文件夹，您会看到从您运行`docker run`命令的位置可以访问相同的文件夹。`-v`参数值的语法是`hostfolder:destinationfolder`，我认为需要指出的一点是，这些路径必须是绝对路径，您不能使用相对路径，如`./`或`../foldername`。您刚刚挂载的卷具有读写访问权限，您所做的任何更改都将同步到主机上的文件夹，所以请小心不要执行`rm -rf *`。在生产环境中创建卷应该非常谨慎使用，我建议在可能的情况下完全避免这样做，因为在生产环境中，没有保证如果容器死亡并被重新创建，它将替换为之前所在的主机。这意味着您对卷所做的任何更改都将丢失。

# Docker 端口

当在容器内运行 Web 应用时，我们通常会需要将一些端口暴露给外部世界。默认情况下，Docker 容器是完全隔离的，如果您在容器内启动一个运行在端口`8080`的服务器，除非您明确指定该端口可以从外部访问，否则它将不可访问。

从安全角度来看，映射端口是一件好事，因为我们遵循的是不信任的原则。同时，暴露这些端口也毫不费力。使用我们在第一章“微服务简介”中创建的示例，让我们看看这有多简单。

移动到您检出示例代码的文件夹，并运行以下 Docker 命令：

```go
$ docker run -it --rm -v $(pwd):/src -p 8080:8080 -w /src golang:alpine /bin/sh  

```

我们传递的`-w`标志是用来设置工作目录的，这意味着我们在容器中运行的任何命令都将在这个文件夹内运行。当我们启动 shell 时，你会看到，我们不需要切换到我们在卷挂载的第二部分指定的文件夹，我们已经在那个文件夹里，可以运行我们的应用程序。我们这次也使用了一个稍微不同的镜像。我们不是使用`alpine:latest`，这是一个轻量级的 Linux 版本，我们使用的是`golang:alpine`，这是一个安装了最新 Go 工具的 Alpine 版本。

如果我们使用`go run main.go`命令启动我们的应用程序；我们应该会看到以下输出：

```go
2016/09/02 05:53:13 Server starting on port 8080  

```

现在切换到另一个 shell，并尝试 curl API 端点：

```go
$ curl -XPOST localhost:8080/helloworld -d '{"name":"Nic"}'  

```

你应该会看到类似以下的消息返回：

```go
{"message":"Hello Nic"}  

```

如果我们运行`docker ps`命令来检查正在运行的容器，我们会看到没有端口被暴露。回到你之前的终端窗口，终止命令并退出容器。

这次，当我们启动它时，我们将添加`-p`参数来指定端口。就像卷一样，这需要一对由冒号（`:`）分隔的值。第一个是我们希望绑定到主机上的目标端口，第二个是我们应用程序绑定的 Docker 容器上的源端口。

因为这绑定到了主机上的端口，就像你因为端口绑定而无法在本地两次启动程序一样，你也不能在 Docker 的主机端口映射中这样做。当然，你可以在不同的容器中启动你的代码的多个实例，并将它们绑定到不同的端口，我们将在稍后看到如何做到这一点。

但首先让我们看看那个端口命令，而不是启动一个容器并创建一个 shell 来运行我们的应用程序，我们可以通过将`/bin/sh`命令替换为我们的`go run`命令，用一条命令来完成这个操作。试一试，看看你能否运行你的应用程序。

明白了？

你应该输入了类似以下的内容：

```go
$ docker run -it --rm -v $(pwd):/src -w /src -p 8080:8080 golang:alpine go run reading_writing_json_8.go  

```

现在再次尝试使用`curl`向 API 发送一些数据，你应该会看到以下输出：

```go
{"message":"Hello Nic"}  

```

就像卷一样，你可以指定多个`-p`参数实例，这使你能够为多个端口设置绑定。

# 删除以显式名称开始的容器

以名称参数开始的容器即使指定了`--rm`参数也不会自动删除。要删除以这种方式启动的容器，我们必须手动使用`docker rm`命令。如果我们向命令中添加`-v`选项，我们还可以删除与其关联的卷。我们真的应该现在就做这件事，或者当我们试图在本章的后面重新创建容器时，你可能会感到有些困惑：

```go
$ docker rm -v server  

```

# Docker 网络

我从未打算让这一章成为官方 Docker 文档的完整复制；我只是试图解释一些关键概念，这些概念将帮助您在阅读本书的其余部分时进步。

Docker 网络是一个有趣的话题，默认情况下，Docker 支持以下网络模式：

+   bridge

+   host

+   none

+   overlay

# 桥接网络

桥接网络是当你启动容器时它们将连接到的默认网络；这是我们能够在上一个示例中将容器连接在一起的原因。为了实现这一点，Docker 使用了一些核心 Linux 功能，如网络命名空间和虚拟以太网接口（或`veth`接口）。

当 Docker 引擎启动时，它会在主机机器上创建`docker0`虚拟接口。`docker0`接口是一个虚拟以太网桥，它自动将数据包转发到连接到它的任何其他网络接口。当容器启动时，它会创建一个`veth`对，它将一个分配给容器，这成为它的`eth0`，另一个连接到`docker0`桥。

# 主机网络

主网络实际上是与 Docker 引擎运行在同一个网络。当你将容器连接到主网络时，容器暴露的所有端口都会自动映射到主机上，它还共享主机的 IP 地址。虽然这看起来像是一种方便，但 Docker 始终被设计为能够在引擎上运行同一容器的多个实例，并且由于在 Linux 中使用`host network`只能将套接字绑定到一个端口，这限制了这一功能。

主网络也可能对您的容器构成安全风险，因为它不再受“不信任”原则的保护，您也不再能够明确控制端口是否暴露。话虽如此，由于主机网络的效率，在某些情况下，如果您预计容器将大量使用网络，将容器连接到主机网络可能是合适的。API 网关可能就是这样一个例子，这个容器仍然可以将请求路由到位于桥接网络上的其他 API 容器。

# 无网络

在某些情况下，您可能希望从任何网络中移除您的容器。考虑这种情况：您有一个只处理存储在文件中的数据的应用程序。利用“不信任”原则，我们可能确定最安全的事情是不将其连接到任何容器，并且只允许它写入挂载在主机上的卷。将您的容器连接到`none`网络正好提供了这种能力，尽管用例可能有些有限，但它确实存在，了解这一点是很好的。

# 覆盖网络

Docker 的覆盖网络是一种独特的 Docker 网络，用于连接运行在不同主机上的容器。正如我们之前所学的，使用桥接网络时，网络通信被限制在 Docker 主机上，这在开发软件时通常是可行的。然而，当你将代码部署到生产环境中时，所有这些都会改变，因为你通常会在多个主机上运行多个容器，作为你的高可用性配置的一部分。容器仍然需要相互通信，虽然我们可以通过**ESB**（企业服务总线）路由所有流量，但在微服务世界中这有点反模式。正如我们将在后面的章节中看到的，推荐的方法是服务负责自己的发现和负载均衡客户端调用。Docker 覆盖网络解决了这个问题，实际上它是在机器之间创建一个网络隧道，将流量无修改地通过物理网络传递。覆盖网络的问题是你不能再依赖 Docker 为你更新`etc/hosts`文件，你必须依赖于动态服务注册。

# 自定义网络驱动程序

Docker 还支持基于其开源`libnetwork`项目的网络插件，你可以编写自定义网络插件来替换 Docker 引擎的网络子系统。它们还提供了将非 Docker 应用程序连接到容器网络的能力，例如物理数据库服务器。

# Weaveworks

Weaveworks 是最受欢迎的插件之一，它使你能够安全地链接你的 Docker 主机，并提供一系列额外的工具，如 weavedns 的服务发现和 weavescope 的可视化，这样你可以看到你的网络是如何连接在一起的。

[`www.weave.works`](https://www.weave.works)

# Project Calico

Project Calico 试图解决使用虚拟局域网、桥接和隧道可能引起的速度和效率问题。它通过将你的容器连接到 vRouter 来实现这一点，然后直接在 L3 网络上路由流量。当你需要在多个数据中心之间发送数据时，这可以带来巨大的优势，因为没有依赖 NAT，较小的数据包大小减少了 CPU 利用率。

[`www.projectcalico.org`](https://www.projectcalico.org)

# 创建自定义桥接网络

实现自定义覆盖网络超出了本书的范围，然而，了解如何创建自定义桥接网络是我们应该关注的，因为 Docker-Compose，我们将在本章后面介绍，利用了这些概念。

与许多 Docker 工具一样，创建桥接网络相当简单。要查看 Docker 引擎上当前正在运行的网络，我们可以执行以下命令：

```go
$ docker network ls  

```

输出应该类似于以下内容：

```go
NETWORK ID          NAME                DRIVER              SCOPE
8e8c0cc84f66        bridge              bridge              local 
0c2ecf158a3e        host                host                local 
951b3fde8001        none                null                local       

```

你会发现默认创建了三个网络，这是我们之前讨论的三个之一。因为这些是默认网络，我们无法删除它们，Docker 需要这些网络才能正确运行，允许你删除它们确实是不好的。

# 创建一个桥接网络

要创建一个桥接网络，我们可以使用以下命令：

```go
$ docker network create testnetwork  

```

现在在你的终端中运行这个命令，再次列出网络以查看结果。

你会看到现在在你的列表中有一个使用桥接驱动程序并且名称是你指定的一个参数的网络。默认情况下，当你创建一个网络时，它使用`bridge`作为默认驱动程序，当然，你可以创建一个使用自定义驱动程序的网络，这可以通过指定额外的参数`-d drivername`来轻松实现。

# 将容器连接到自定义网络

要将容器连接到自定义网络，让我们再次使用我们在第一章中创建的示例应用程序：*微服务简介*：

```go
$ docker run -it --rm -v $(pwd):/src -w /src --name server --network=testnetwork golang:alpine go run main.go  

```

你是否收到了错误信息，说名称已被使用，因为你忘记在早期部分移除容器？如果是这样，可能需要回翻几页。

假设一切顺利，你应该看到服务器启动的消息，现在让我们尝试使用我们之前执行的相同命令来 curl 容器：

```go
$ docker run --rm appropriate/curl:latest curl -i -XPOST server:8080/helloworld -d '{"name":"Nic"}'  

```

您应该已经收到了以下错误信息：

```go
curl: (6) Couldn't resolve host 'server'

```

这是预期的，尝试更新`docker run`命令，使其与我们的 API 容器一起工作。

明白了？

如果没有，这里有一个添加了网络参数的修改后的命令：

```go
$ docker run --rm --network=testnetwork appropriate/curl:latest curl -i -XPOST server:8080/helloworld -d '{"name":"Nic"}'  

```

这个命令应该第二次运行得很好，你应该看到预期的输出。现在移除服务器容器，我们将看看如何编写你自己的 Docker 文件。

# 编写 Dockerfile

Dockerfile 是我们的镜像配方；它们定义了基本镜像、要安装的软件，并赋予我们设置应用程序所需的各种结构的权限。

在本节中，我们将探讨如何为我们的示例 API 创建 Dockerfile。再次强调，这不会是 Dockerfile 如何工作的全面概述，因为有许多书籍和在线资源专门为此目的存在。我们将查看关键点，这将给我们基础知识。

我们将要做的第一件事是构建我们的应用程序代码，因为当我们将其打包到 Dockerfile 中时，我们将执行一个二进制文件，而不是使用`go run`命令。我们将创建的镜像将只包含运行我们的应用程序所需的软件。在创建镜像时限制安装的软件是 Docker 的最佳实践，因为它通过仅包含必要的软件来减少攻击面。

# 为 Docker 构建应用程序代码

我们将执行一个与通常的`go build`略有不同的命令来创建我们的文件：

```go
$ CGO_ENABLED=0 GOOS=linux GOARCH=386 go build -a -installsuffix cgo -ldflags '-s' -o server  

```

在前面的命令中，我们传递了参数 `-ldflags '-s'`，这个参数在构建应用程序时将 `-s` 参数传递给链接器，并告诉它静态链接所有依赖项。当我们使用流行的 Scratch 容器作为基础时，这非常有用；Scratch 是最轻的基础之一，它没有任何应用程序框架或应用程序，这与占用约 150MB 的 Ubuntu 相反。Scratch 和 Ubuntu 之间的区别在于 Scratch 没有访问标准 C 库 `GLibC` 的权限。

如果我们不构建静态二进制文件，那么如果我们尝试在 Scratch 容器中运行它，它将不会执行。这是因为虽然您可能认为您的 Go 应用程序是一个静态二进制文件，但它仍然依赖于 `GLibC`，`net` 和 `os/user` 包都链接到 `GLibC`，因此如果我们要在 Scratch 基础镜像上运行我们的应用程序，我们需要静态链接这个库。然而，好处是图像非常小，我们最终得到的图像大小大约为 4MB，正好是我们编译的 Go 应用程序的大小。

因为 Docker 引擎是在 Linux 上运行的，所以我们还需要为 Linux 架构构建我们的 Go 可执行文件。即使您使用 Docker for Mac 或 Docker for Windows，底层发生的事情是 Docker 引擎在 `HyperV` 或 Mac 的 `xhyve` 虚拟机上运行一个轻量级的虚拟机。

如果您不是使用 Linux 运行 go build 命令，并且由于 Go 具有出色的跨平台编译能力，您不需要做太多。您只需要像我们在前面的示例中做的那样，在 go build 命令前加上架构变量 `GOOS=linux GOARCH=386`。

现在我们已经为我们的应用程序创建了一个二进制文件，让我们来看看 Dockerfile：

```go
1 FROM scratch
2 MAINTAINER jackson.nic@gmail.com
3
4 EXPOSE 8080
5
6 COPY ./server ./
7 
8 ENTRYPOINT ./server  

```

# FROM

`FROM` 指令为后续指令设置基础镜像。您可以使用存储在远程注册表或本地 Docker 引擎上的任何镜像。当您执行 `docker build` 时，如果您还没有这个镜像，Docker 将在构建过程的第一个步骤中从注册表中拉取它。`FROM` 命令的格式与您在发出 `docker run` 命令时使用的格式相同，它可以是：

+   `FROM image` // 假设为最新版本

+   `FROM image:tag` // 其中您可以指定要使用的标签

在 **第 1 行**，我们使用的是 image 名称 scratch，这是一种特殊的镜像，基本上是一个空白画布。我们可以使用 Ubuntu、Debian、Alpine 或几乎所有其他东西，但由于我们只需要运行我们的 Go 应用程序本身，因此我们可以使用 scratch 来生成可能最小的镜像。

# 维护者

`MAINTAINER` 指令允许您设置生成的镜像的作者。这是一个可选指令；然而，即使您不打算将您的镜像发布到公共注册表，包含这个指令也是一个好的实践。

# EXPOSE

`EXPOSE` 指令通知 Docker 容器在运行时监听指定的网络端口。`Expose` 不会使端口对主机可访问；此功能仍需要使用 `-p` 映射来执行。

# COPY

`COPY` 指令将文件从指令第一部分的源复制到第二部分指定的目标：

+   `COPY <src> <dest>`

+   `COPY ["<src>", "<dest>"]` // 当路径包含空格时很有用

`COPY` 指令中的 `<src>` 可以包含通配符，匹配规则使用 Go 的 `filepath.Match` 规则。

注意：

+   `<src>` 必须是构建上下文的一部分，您不能指定相对文件夹，例如 `../;`

+   在 `<src>` 中指定的根 `/` 将是上下文的根

+   在 `<dest>` 中指定的根 `/` 将映射到容器的根文件系统

+   指定没有目标的 `COPY` 指令将文件或文件夹复制到与原始文件同名的 `WORKDIR`

# ENTRYPOINT

`ENTRYPOINT` 允许您配置在容器启动时您希望运行的可执行文件。使用 `ENTRYPOINT` 可以在 `docker run` 命令中指定参数，这些参数将附加到 `ENTRYPOINT`。

`ENTRYPOINT` 有两种形式：

+   `ENTRYPOINT ["executable", "param1", "param2"]` // 优先形式

+   `ENTRYPOINT command param1 param2` // shell 形式

例如，在我们的 Docker 文件中，我们指定了 `ENTRYPOINT ./server`。这是我们想要运行的 Go 可执行文件。当我们使用以下 `docker run helloworld` 命令启动容器时，我们不需要明确告诉容器执行二进制文件并启动服务器。然而，我们可以通过 `docker run` 命令的参数传递额外的参数给应用程序；这些参数将在应用程序运行之前附加到 `ENTRYPOINT`。例如：

```go
$ docker run --rm helloworld --config=/configfile.json  

```

前一个命令将参数附加到入口点中定义的执行语句，这相当于执行以下 shell 命令：

```go
$ ./server --config=configfile.json  

```

# CMD

`CMD` 指令有三种形式：

+   `CMD ["executable", "param1", "param2"]` // exec 形式

+   `CMD ["param1", "param2"]` // 将默认参数附加到 `ENTRYPOINT`

+   `CMD command param1 param2` // shell 形式

当使用 `CMD` 为 `ENTRYPOINT` 指令提供默认参数时，应使用 `JSON` 数组格式指定 `CMD` 和 `ENTRYPOINT` 指令。

如果我们为 `CMD` 指定默认值，我们仍然可以通过传递命令参数到 `docker run` 命令来覆盖它。

Dockerfile 中只允许有一个 `CMD` 指令。

# 创建 Dockerfile 的良好实践

考虑到所有这些，我们需要记住 Docker 中的联合文件系统是如何工作的，以及我们如何利用它来创建小型和紧凑的镜像。每次我们在 Dockerfile 中发出命令时，Docker 都会创建一个新的层。当我们修改这个命令时，这个层必须完全重新创建，甚至可能所有后续的层也需要重新创建，这可能会显著减慢你的构建速度。因此，建议你尽可能紧密地分组你的命令，以减少这种情况发生的可能性。

很常见，你会看到 Dockerfile，而不是为每个我们想要执行的命令单独有一个`RUN`命令，我们使用标准的 bash 格式来链式执行这些命令。

例如，考虑以下内容，它将从包管理器安装软件。

**不良实践：**

```go
RUN apt-get update
RUN apt-get install -y wget
RUN apt-get install -y curl
RUN apt-get install -y nginx  

```

**良好实践：**

```go
RUN apt-get update && \
 apt-get install -y wget curl nginx

```

第二个示例只会创建一层，这又会进一步创建一个更小、更紧凑的图像，将更改最少的语句放在 Dockerfile 的上方也是一种良好的实践，这样即使这些层没有变化，也可以避免后续层的无效化。

# 从 Dockerfile 构建镜像

要从我们的 Dockerfile 构建一个镜像，我们可以执行一个简单的命令：

```go
$ docker build -t testserver .  

```

分解这个`-t`参数是我们想要赋予容器的标签，它采用 name:tag 的形式，如果我们像示例命令中那样省略了`tag`部分，那么将自动分配标签`latest`。

如果你运行`docker images`，你会看到我们的`testserver`镜像已经被赋予了这个标签。

最后一个参数是我们想要发送给 Docker Engine 的上下文。当你运行 Docker 构建时，上下文会自动转发到服务器。这听起来可能有些奇怪，但你要记住，Docker Engine 可能不会在你的本地机器上运行，因此它将无法访问你的本地文件系统。因此，我们应该小心设置上下文的位置，因为这可能意味着大量数据被发送到引擎，这会减慢速度。上下文然后成为你的`COPY`命令的根。

现在我们有了正在运行的容器，让我们来测试它。为什么不从一个新构建的镜像中启动一个容器，并通过 curl 端点来检查 API 呢：

```go
$ docker run --rm -p 8080:8080 testserver
$ curl -XPOST localhost:8080/helloworld -d '{"name":"Nic"}'

```

# Docker 构建上下文

当我们运行 Docker 构建命令时，我们将上下文路径设置为最后一个参数。当命令执行时实际上发生的情况是上下文被传输到服务器。如果你有一个大的源文件夹，这可能会导致问题，因此只发送需要打包到容器内或构建容器时需要的文件是一种良好的实践。我们可以通过两种方式来减轻这个问题。第一种是确保我们的上下文只包含我们需要的文件。由于这并不总是可能的，所以我们还有一个次要选项，即使用 `.dockerignore` 文件。

# Docker 忽略文件

在 CLI 将上下文发送到引擎之前，`.dockerignore` 文件类似于 git 忽略文件，它排除了与 `.dockerignore` 文件中匹配模式的文件和目录。它使用 Go 的 `filepath.Match` 规则定义的模式，你可以在以下 Go 文档中找到更多关于它们的信息：[`godoc.org/path/filepath#Match`](https://godoc.org/path/filepath#Match)

| **规则** | **行为** |
| --- | --- |
| `# comment` | 被忽略。 |
| `*/temp*` | 排除根目录任何直接子目录中以 temp 开头的文件和目录。例如，普通文件 `/somedir/temporary.txt` 被排除，同样 `/somedir/temp` 目录也被排除。 |
| `*/*/temp*` | 排除根目录下两级以下子目录中以 temp 开头的文件和目录。例如，`/somedir/subdir/temporary.txt` 被排除。 |
| `temp?` | 排除根目录中名称为 temp 的一字符扩展名的文件和目录。例如，`/tempa` 和 `/tempb` 被排除。 |

[`docs.docker.com/engine/reference/builder/#/dockerignore-file`](https://docs.docker.com/engine/reference/builder/#/dockerignore-file)

[﻿](https://docs.docker.com/engine/reference/builder/#/dockerignore-file)

# 在容器中运行守护进程

当你在虚拟机或物理服务器上部署应用程序时，你可能习惯于使用 `initd` 或 `systemd` 这样的守护进程运行器来确保应用程序在后台启动并继续运行，即使它崩溃了。当你使用 Docker 容器时，这是一个反模式，因为 Docker 要成功停止应用程序，它将尝试杀死 PID 为 1 的进程。守护进程通常以 PID 1 启动，并使用另一个进程 ID 启动你的应用程序，这意味着当停止 Docker 容器时，它们不会被杀死。这可能导致在执行 `docker stop` 命令时容器挂起。

如果你需要确保应用程序即使在崩溃后也能继续运行，那么你可以将这个责任委托给启动你的 Docker 容器的编排器。我们将在稍后的章节中学习更多关于编排的内容。

# Docker Compose

这一切都很简单，现在让我们看看 Docker 的一个引人注目的特性，它允许您通过存储在方便的 YAML 文件中的堆栈定义一次性启动多个容器。

# 在 Linux 上安装 Docker Compose

如果您已经安装了 Docker for Mac 或 Docker for Windows，那么`docker-compose`已经捆绑在其中了；然而，如果您正在使用 Linux，那么您可能需要自己安装它，因为它不是默认 Docker 包的一部分。

要在 Linux 上安装 Docker Compose，请在您的终端中执行以下命令：

```go
$ curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose  && chmod +x /usr/local/bin/docker-compose  

```

在我们看看如何使用`docker-compose`运行我们的应用程序之前，让我们看看我们将要运行的文件以及它的一些重要方面：

```go
1 version: '2' 
2 services: 
3   testserver: 
4     image: testserver 
5   curl: 
6     image: appropriate/curl 
7     entrypoint: sh -c  "sleep 3 && curl -XPOST testserver:8080/helloworld -d '{\"name\":\"Nic\"}'" 

```

Docker Compose 文件是用 YAML 编写的，在这个文件中，您可以定义将构成您应用程序的服务。在我们的简单示例中，我们只描述了两个服务。第一个是我们刚刚构建的示例代码，第二个是一个简单的服务，它会向这个 API 发送 curl 请求。我承认，作为一个生产示例，这并不特别有用，但它只是用来展示如何设置这些文件。随着我们进入后面的章节，我们将大量依赖这些文件来创建我们的数据库和其他数据存储，这些数据存储构成了我们的应用程序。

**第 1 行**定义了我们使用的 Docker Compose 文件的版本，版本 2 是最新版本，与版本 1 相比是一个重大变化，`--link`指令现在已弃用，将在未来的版本中删除。

在**第 2 行**中我们定义了服务。服务是您希望与堆栈一起启动的容器。每个服务都必须在 compose 文件中有一个唯一的名称，但不必与在您的 Docker Engine 上运行的容器名称相同。为了避免在启动堆栈时发生冲突，我们可以将`-p projectname`传递给`docker-compose up`命令；这将给我们的任何容器的名称前加上指定的项目名称。

您需要为服务指定的最小信息是镜像，这是您希望从中启动容器的镜像。与`docker run`的工作方式相同，这可以是 Docker Engine 上的本地镜像，也可以是远程注册表中镜像的引用。当您启动堆栈时，compose 将检查镜像是否本地可用，如果不可用，它将自动从注册表中拉取。

**第 6 行**定义了我们的第二个服务；这只是一个简单地执行命令来向第一个服务暴露的 API 发送 curl 请求。

在这个服务定义块中，我们既指定了镜像又指定了入口点。

# 服务启动

之前的命令看起来有点奇怪，但 Docker Compose 有一个陷阱，很多人都会犯这个错误，那就是 compose 没有真正的方法知道应用程序何时正在运行。即使我们使用了 `depends-on` 配置，我们只是在通知 compose 存在依赖关系，并且它应该控制服务的启动顺序。

```go
sh -c  "sleep 3 && curl -XPOST testserver:8080/helloworld -d '{\"name\":\"Nic\"}'"

```

Compose 所做的只是检查容器是否已启动。一般问题发生在对容器启动等于它准备好接收请求的误解。通常情况下并非如此，应用程序启动并准备好接受请求可能需要一些时间。如果你有一个像我们在 curl 入口点指定的那样依赖另一个服务的端点，那么我们不能假设在执行我们的命令之前，依赖的服务已经准备好接收请求。我们将在 第六章 “微服务框架”中介绍处理这种模式的方法，但现在我们可以意识到：

"容器已启动，服务已准备好并不等同于它已经准备好接收请求。"

在我们的简单示例中，我们知道服务启动大概需要一秒钟左右的时间，所以我们只需等待三秒钟，给服务足够的时间准备，然后再执行我们的命令。这种方法并不是一个好的实践，它只是为了说明我们如何使用 compose 来连接服务。实际上，你很可能不会在你的 compose 文件中启动单个命令，就像我们在这里做的那样。

当你使用 Docker 网络，Docker 会自动将映射添加到容器的 `resolve.conf` 文件中，指向内置的 Docker DNS 服务器，然后我们可以通过名称引用连接到同一网络的其他容器。查看我们的 curl 命令，这个 DNS 功能正是允许我们使用主机名 testserver 的原因。

好的，是时候测试一下了，从您的终端运行以下命令：

```go
$ docker-compose up  

```

如果一切顺利，你应该在输出中看到以下消息：

```go
{"message":"Hello Nic"}  

```

*Ctrl* + *C* 将退出 compose，然而，由于我们使用 `docker run` 命令并传递了 `--rm` 参数来删除容器，我们需要确保自己清理。要删除使用 `docker-compose` 启动的任何已停止容器，我们可以使用特定的 compose 命令 `rm` 并传递 `-v` 参数来删除任何相关卷：

```go
$ docker-compose rm -v  

```

# 指定 compose 文件的存储位置

每次运行 `docker-compose` 时，它都会在当前文件夹中查找名为 `docker-compose.yml` 的文件作为默认文件。要指定一个不同的文件，我们可以向 compose 传递 `-f` 参数，并指定我们想要加载的 compose 文件路径：

```go
$ docker-compose -f ./docker-compose.yml up  

```

# 指定项目名称

正如我们之前在启动`docker-compose`时讨论的那样，它将创建在您的 Compose 文件中给定名称的服务，并在其后面附加项目名称`default`。如果我们需要运行多个此 Compose 文件实例，那么`docker-compose`不会启动另一个实例，因为它会先检查是否有任何服务以给定名称运行。为了覆盖这一点，我们可以指定项目名称，以替换默认的`default`名称。为此，我们只需在命令中指定`-p projectname`参数，如下所示：

```go
$ docker-compose -p testproject up  

```

这将创建两个容器：

+   `testproject_testserver`

+   `testproject_curl`

# 总结

总结来说，在本章中我们学习了如何使用 Docker，虽然这只是一个简要概述，但我建议您查阅文档，更深入地了解 Dockerfile、Composefile、Docker Engine 和 Docker Compose 的概念。Docker 是开发、测试和生产中不可或缺的工具，随着我们进入接下来的章节，我们将广泛使用这些概念。在下一章，我们将探讨测试，这将建立在您迄今为止所学的一切之上。
