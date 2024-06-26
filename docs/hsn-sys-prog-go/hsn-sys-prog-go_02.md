# 第一章：系统编程简介

本章是系统编程的介绍，探讨了从最初定义到随着系统演变而发生变化的一系列主题。本章提供了一些基本概念和 Unix 及其资源的概述，包括内核和应用程序编程接口（API）。这些概念中的许多是在这里定义的，并在本书的其余部分中使用。

本章将涵盖以下主题：

+   什么是系统编程？

+   应用程序编程接口

+   了解保护环如何工作

+   系统调用概述

+   POSIX 标准

# 技术要求

如果您使用 Linux，则本章不需要您安装任何特殊软件。

如果您是 Windows 用户，可以安装 Windows 子系统用于 Linux（WSL）。按照以下步骤安装 WSL：

1.  以管理员身份打开 PowerShell 并运行以下命令：

```go
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

1.  在提示时重新启动计算机。

1.  从 Microsoft Store 安装您喜欢的 Linux 发行版。

# 开始系统编程

多年来，IT 领域发生了巨大变化。挑战冯·诺伊曼机的多核 CPU、互联网和分布式系统只是过去 30 年发生的一些变化。那么，系统编程在这个领域中处于什么位置呢？

# 为软件而设计的软件

让我们首先从标准教科书的定义开始。

系统编程（或系统编程）是编程计算机系统软件的活动。与应用程序编程相比，系统编程的主要区别特征在于，应用程序编程旨在生产直接为用户提供服务的软件（例如，文字处理器），而系统编程旨在生产为其他软件提供服务的软件和软件平台，并且设计为在性能受限的环境中工作，例如操作系统、计算科学应用程序、游戏引擎和 AAA 视频游戏、工业自动化和软件即服务应用程序。

该定义突出了系统应用程序的两个主要概念如下：

+   被其他软件使用的软件，而不是直接由最终用户使用。

+   软件具有硬件意识（它知道硬件如何工作），并且面向性能。

这使得很容易将操作系统内核、硬件驱动程序、编译器和调试器等系统软件识别为系统软件，而不是系统软件、聊天客户端或文字处理器。

从历史上看，系统程序是使用汇编语言和 C 创建的。然后出现了用于将系统程序提供的功能联系在一起的 shell 和脚本语言。系统语言的另一个特征是对内存分配的控制。

# 语言和系统演变

在过去的十年中，脚本语言变得越来越受欢迎，以至于一些脚本语言有了显著的性能改进，并且整个系统都是用它们构建的。例如，让我们想一想 JavaScript 的 V8 引擎和 Python 的 PyPy 实现，它们显著改变了这些语言的性能。

其他语言，如 Go，证明了垃圾收集和性能并不是互斥的。特别是，Go 在 1.5 版中成功用 Go 编写的本地版本替换了 C 编写的内存分配器，将性能提高到可比较的水平。

与此同时，系统开始变得分布式，应用程序开始以容器的形式进行部署，由其他系统软件（例如 Kubernetes）进行编排。这些系统旨在维持巨大的吞吐量，并通过两种主要方式实现：

+   通过扩展-增加托管系统的机器数量或资源

+   通过优化软件以提高资源利用效率

# 系统编程和软件工程

系统编程的一些实践——比如将应用程序与硬件绑定、以性能为导向、在资源受限的环境中工作——是一种在构建分布式系统时也可以有效的方法，其中限制资源使用可以减少所需的实例数量。看起来系统编程是解决通用软件工程问题的好方法。

这意味着学习系统编程的概念，关于如何有效地使用机器资源——从内存使用到文件系统访问——将有助于构建任何类型的应用程序。

# 应用程序编程接口

API 是一系列子例程定义、通信协议和构建软件的工具。API 最重要的方面是它提供的功能，以及它的文档，这些文档可以帮助用户使用和实现软件在另一个软件中的使用。API 可以是允许应用软件使用系统软件的接口。

API 通常有一个特定的发布政策，旨在供特定的接收者使用。这可以是以下内容：

+   私有的，仅供内部使用。

+   合作伙伴和可由确定的群体使用——这可能包括希望将服务与自己的公司整合的公司

+   公开并可供每个用户使用

# API 的类型

我们将看到有几种类型的 API，从用于使不同的应用软件一起工作的 API，到操作系统向其他软件公开的内部 API。

# 操作系统

API 可以指定如何与应用程序和操作系统进行接口。例如，Windows、Linux 和 macOS 都有一个接口，可以操作文件系统和文件。

# 库和框架

与软件库相关的 API 描述并规定（提供如何使用它的说明）其每个元素应该如何行为，包括最常见的错误场景。API 的行为和接口通常被称为**库规范**，而库是这种规范中描述的规则的实现。库和框架通常是语言绑定的，但也有一些工具可以使库在不同的语言中使用。你可以使用 CGO 在 Go 中使用 C 代码，在 Python 中你可以使用 CPython。

# 远程 API

这些使得可以使用特定的通信标准来操作远程资源，这些标准允许不同的技术一起工作，无论语言或平台如何。一个很好的例子是**Java 数据库连接**（**JDBC**）API，它允许使用相同的函数查询许多不同类型的数据库，或者 Java 远程方法调用 API（Java RMI），它允许像本地函数一样使用远程函数。

# Web API

Web API 是定义有关使用的协议、消息编码和可用端点及其预期输入和输出值的一系列规范的接口。这种 API 有两种主要的范式——REST 和 SOAP：

+   REST API 具有以下特点：

+   它们将数据视为资源。

+   每个资源都由 URL 标识。

+   操作类型由 HTTP 方法指定。

+   SOAP 协议具有以下特点：

+   它们由 W3C 标准定义。

+   XML 是消息的唯一编码方式。

+   它们使用一系列 XML 模式来验证数据。

# 理解保护环

**保护环**，也称为**分层保护域**，是用于保护系统免受故障的机制。它的名称源自其权限级别的分层结构，由同心圆环表示，当移动到外部环时，特权会减少。在每个环之间有特殊的门，允许外部环以受限的方式访问内部环的资源。

# 架构差异

环的数量和顺序取决于 CPU 架构。它们通常以权限降低的顺序编号，使 ring 0 成为最具特权的环。这对于使用四个环（从 ring 0 到 ring 3）的 i386 和 x64 架构是正确的，但对于使用相反顺序（从 EL3 到 EL0）的 ARM 架构是不正确的。大多数操作系统不使用所有四个级别；它们最终使用两级层次结构—用户/应用程序（ring 3）和内核（ring 0）。

# 内核空间和用户空间

在操作系统下运行的软件将在用户（ring 3）级别执行。为了访问机器资源，它将必须与运行在 ring 0 的操作系统内核进行交互。以下是 ring 3 应用程序无法执行的一些操作：

+   修改当前段描述符，确定当前环

+   修改页表，防止一个进程看到其他进程的内存

+   使用 LGDT 和 LIDT 指令，防止它们注册中断处理程序

+   使用 I/O 指令，比如 in 和 out，可以忽略文件权限并直接从磁盘读取。

例如，对磁盘内容的访问将由内核进行调解，内核将验证应用程序是否有权限访问数据。这种协商方式提高了安全性，避免了故障，但会带来重要的开销，影响应用程序的性能。

有些应用程序可以直接在硬件上运行，而不需要操作系统提供的框架。这对于实时系统来说是真实的，因为在实时系统中，响应时间和性能都不容许妥协。

# 深入系统调用

系统调用是操作系统为应用程序提供对资源访问的方式。这是内核实现的 API，用于安全地访问硬件。

# 提供的服务

有一些类别可以用来分割操作系统提供的众多功能。这些包括控制运行应用程序及其流程、文件系统访问和网络。

# 进程控制

这种类型的服务包括`load`，它将程序添加到内存并在将控制传递给程序本身之前准备执行，或者`execute`，它在现有进程的上下文中运行可执行文件。属于这一类别的其他操作如下：

+   `end`和`abort`—第一个要求应用程序退出，而第二个强制退出。

+   `CreateProcess`，也称为 Unix 系统上的`fork`或 Windows 中的`NtCreateProcess`。

+   终止进程。

+   获取/设置进程属性。

+   等待时间、等待事件或信号事件。

+   分配和释放内存。

# 文件管理

文件和文件系统的处理属于文件管理系统调用。有*create*和*delete*文件，可以向文件系统添加或删除条目，以及`open`和`close`操作，可以控制文件以执行读写操作。还可以读取和更改文件属性。

# 设备管理

设备管理处理除文件系统之外的所有其他设备，如帧缓冲区或显示器。它包括从设备的请求开始的所有操作，包括与设备的通信（读取、写入、寻址）以及其释放。它还包括更改设备属性和逻辑附加和分离设备的所有操作。

# 信息维护

读取和写入系统日期和时间属于信息维护类别。这个类别还负责其他系统数据，比如环境。还有一组重要的操作属于这里，包括请求和处理进程、文件和设备属性。

# 通信

所有网络操作，从处理套接字到接受连接，都属于通信类别。这包括连接的创建、删除和命名，以及发送和接收消息。

# 操作系统之间的差异

Windows 具有一系列不同的系统调用，涵盖了所有内核操作。其中许多与 Unix 等效的操作完全对应。以下是一些重叠的系统调用列表：

|  | **Windows** | **Unix** |
| --- | --- | --- |

| **进程控制** | `CreateProcess()` `ExitProcess()`

`WaitForSingleObject()` | `fork()` `exit()`

`wait()`

|

| **文件操作** | `CreateFile()` `ReadFile()`

`WriteFile()`

`CloseHandle()` | `open()` `read()`

`write()`

`close()` |

| **文件保护** | `SetFileSecurity()` `InitializeSecurityDescriptor()`

`SetSecurityDescriptorGroup()` | `chmod()` `umask()`

`chown()` |

| **设备管理** | `SetConsoleMode()` `ReadConsole()`

`WriteConsole()` | `ioctl()` `read()`

`write()` |

| **信息维护** | `GetCurrentProcessID()` `SetTimer()`

`Sleep()` | `getpid()` `alarm()`

`sleep()` |

| **通信** | `CreatePipe()` `CreateFileMapping()`

`MapViewOfFile()` | `pipe()` `shmget()`

`mmap()` |

# 理解 POSIX 标准

为了确保操作系统之间的一致性，IEEE 对操作系统进行了一些标准化。这些标准在以下部分中描述。

# POSIX 标准和特性

Unix 的**可移植操作系统接口**（**POSIX**）代表了操作系统接口的一系列标准。第一个版本可以追溯到 1988 年，涵盖了诸如文件名、shell 和正则表达式等一系列主题。

POSIX 定义了许多特性，它们分为四个不同的标准，每个标准都专注于 Unix 兼容性的不同方面。它们都以 POSIX 加一个数字命名。

# POSIX.1 - 核心服务

POSIX.1 是 1988 年的原始标准，最初被命名为 POSIX，但后来改名以便在不放弃名称的情况下添加更多的标准。它定义了以下特性：

+   进程创建和控制

+   信号：

+   浮点异常

+   分段/内存违规

+   非法指令

+   总线错误

+   定时器

+   文件和目录操作

+   管道

+   C 库（标准 C）

+   I/O 端口接口和控制

+   进程触发器

# POSIX.1b 和 POSIX.1c - 实时和线程扩展

POSIX.1b 专注于实时应用程序和需要高性能的应用程序。它专注于以下方面：

+   优先级调度

+   实时信号

+   时钟和定时器

+   信号量

+   消息传递

+   共享内存

+   异步和同步 I/O

+   内存锁定接口

POSIX.1c 引入了多线程范式，并定义了以下内容：

+   线程创建、控制和清理

+   线程调度

+   线程同步

+   信号处理

# POSIX.2 - shell 和实用程序

POSIX.2 为命令行解释器和实用程序（如`cd`、`echo`或`ls`）指定了标准。

# 操作系统遵从性

并非所有操作系统都符合 POSIX 标准。例如，Windows 诞生于该标准之后，因此不符合标准。从认证的角度来看，macOS 比 Linux 更符合标准，因为后者使用了另一个建立在 POSIX 之上的标准。

# Linux 和 macOS

大多数 Linux 发行版遵循**Linux 标准基础**（**LSB**），这是另一个包括 POSIX 和更多内容的标准，专注于维护不同 Linux 发行版之间的互操作性。它并未被认为是官方符合标准，因为开发人员没有进行认证过程。

然而，自 2007 年的 Snow Leopard 发行版起，macOS 已经完全兼容，并且自那时起就获得了 POSIX 认证。

# Windows

Windows 不符合 POSIX 标准，但有许多尝试使其符合。有一些开源倡议，如 Cygwin 和 MinGW，它们提供了一个不太符合 POSIX 标准的开发环境，并支持使用 Microsoft Visual C 运行时库的 C 应用程序。微软本身也尝试过 POSIX 兼容性，比如 Microsoft POSIX 子系统。微软最新的兼容层是 Windows Linux 子系统，这是 Windows 10 中可选的功能，受到了开发人员（包括我自己）的好评。

# 总结

在本章中，我们看到了系统编程的含义——编写具有严格要求的系统软件，例如与硬件绑定、使用低级语言以及在资源受限的环境中工作。在构建通常需要优化资源使用的分布式系统时，它的实践可以非常有用。我们讨论了 API，定义了允许软件被其他软件使用的不同类型，包括操作系统中的 API、库和框架中的 API，以及远程和 Web API。

我们分析了在操作系统中，对资源的访问是通过称为**保护环**的分层级别进行安排的，以防止不受控制的使用，以提高安全性并避免应用程序的故障。Linux 模型简化了这种层次结构，只将其分为称为*用户*和*内核*空间的两个级别。所有应用程序都在用户空间中运行，为了访问机器的资源，它们需要内核进行干预。

然后我们看到了一种特定类型的 API，称为**系统调用**，它允许应用程序向内核请求资源，并调解进程控制、文件访问和管理，以及设备和网络通信。

我们概述了 POSIX 标准，该标准定义了 Unix 系统的互操作性。在定义的特性中，还包括 C API、CLI 实用程序、shell 语言、环境变量、程序退出状态、正则表达式、目录结构、文件名和命令行实用程序 API 约定。

在下一章中，我们将探讨 Unix 操作系统资源，如文件系统和 Unix 权限模型。我们将研究进程是什么，它们如何相互通信，以及它们如何处理错误。

# 问题

1.  应用程序编程和系统编程之间有什么区别？

1.  什么是 API？API 为什么如此重要？

1.  你能解释一下保护环是如何工作的吗？

1.  你能举一些在用户空间无法完成的例子吗？

1.  什么是系统调用？

1.  Unix 中使用哪些调用来管理进程？

1.  POSIX 为什么有用？

1.  Windows 是否符合 POSIX 标准？
