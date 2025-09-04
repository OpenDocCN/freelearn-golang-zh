# 第十三章：*附录 C*: 交叉编译

当构建需要访问本地 API 和图形硬件的应用程序时，我们可以使用**CGo**。尽管对于常规开发来说并不困难，但这确实使得交叉编译变得更加复杂。对于你想要为每个目标平台构建的，都必须有一个**C 编译器**知道如何创建本地二进制文件。本附录概述了设置本附录中先前提到的每个组合所需的交叉编译目标步骤。

重要提示

请注意，交叉编译对于日常开发不是必需的。对于大多数开发，你不需要交叉编译器的设置。Go 编译器和*附录 A**:* *开发者工具安装*中讨论的标准工具，就足够你在标准计算机上进行开发了。本附录是关于安装创建针对不同操作系统或架构的编译应用程序的附加工具。

在本附录中，我们将介绍两种可能的编译方法：

+   手动安装交叉编译工具链

+   使用`fyne-cross`自动处理编译

我们将从手动过程开始，因为在看到自动化过程之前了解编译的复杂性是有用的。

# 手动安装编译器

安装编译器和工具链是复杂的，但本附录将尝试引导你通过主要步骤。这种方法有时被希望管理他们计算机每个细节的开发者所青睐。如果你的计算机之前曾用于**C**开发并创建多个平台的本地应用程序，这也可能更容易。

准备为不同的目标编译有所不同，这取决于你想要编译的系统。我们将首先查看**macOS**，然后探索**Windows**和**Linux**。由于我们在*附录 B**:* *安装移动构建工具*中安装了这些工具，因此不需要遵循这些步骤来构建移动应用程序。

## 为 macOS 进行交叉编译

当为 macOS 进行交叉编译时，有必要安装来自**苹果**的**软件开发工具包**（**SDK**）以及一个合适的编译器。Windows 的说明（使用**MSYS2**，如*附录 A*，*开发者工具安装*中所述）和 Linux 几乎相同；我们主要需要做的是安装 macOS SDK。

你需要注意的一件事是，苹果公司已经移除了对构建 32 位二进制文件的支持。如果你希望支持不是 64 位的旧设备，你需要安装较旧的 Xcode（9.4.1 或更低版本）和 Go（1.13 或更低版本）。

安装 macOS SDK 最简单的方法是使用 `osxcross` 项目。以下步骤显示了如何下载和安装 SDK 以及构建 macOS Fyne 应用所需的所有必要工具，而无需使用 Macintosh 计算机。这里我们使用 Linux，但对于使用 MSYS2 命令行工具的 Windows 开发者来说，过程是相同的：

1.  我们将使用 `clang` 编译器代替 `gcc`，因为它的设计更便携。为了使此过程正常工作，您需要使用您的包管理器安装 `clang`、`cmake` 和 `libxml2-dev`：

    +   在 Linux 上，使用 `apt-get install clang cmake libxml2-dev`（或根据您的发行版使用适当的 `pacman` 或 `dnf` 命令）

    +   在 Windows 上，使用 `pacman -S mingw-w64-x86_64-clang mingw-w64- x86_64-cmake mingw-w64-x86_64-libxml2`

1.  接下来，我们需要下载 macOS SDK，它包含在 Xcode 中。如果您还没有 Apple 开发者账户，您需要注册并同意他们的条款和条件。使用此账户，登录到 [developer.apple.com/download/more/?name=Xcode%2010.2](http://developer.apple.com/download/more/?name=Xcode%2010.2) 下载 `XCode.dmg`（**10.2** 推荐用于针对 64 位分布的 **osxcross**，尽管如果您想支持 32 位计算机，也可以下载 9.4）。

1.  然后，我们必须安装 `git` 命令：

    ```go
    $ git clone https://github.com/tpoechtrager/osxcross.git
    ```

1.  下载完成后，进入新目录。使用此存储库中的包工具，我们必须从下载的 `Xcode.dmg` 文件中提取 macOS SDK：

    ```go
    MacOSX10.11.sdk.tar.xz file should be copied into the tarballs/ directory.
    ```

1.  最后，我们必须通过执行提供的构建脚本来构建 **osxcross** 编译器扩展：

    ```go
    $ ./build.sh
    ```

1.  完成此操作后，将出现一个名为 `target/bin/` 的新目录，您应该将其添加到您的 `PATH` 环境变量中。现在可以在 `CC=o64-clang` 环境变量中使用编译器；例如：

    ```go
    $ CC=o64-clang GOOS=darwin CGO_ENABLED=1 go build .
    ```

关于此过程以及如何将其适应其他平台的更多详细信息，可在 **osxcross** 项目网站上找到，网址为 [github.com/tpoechtrager/osxcross](http://github.com/tpoechtrager/osxcross)。

## 为 Windows 进行交叉编译

从其他平台为 Windows 构建需要我们安装 `mingw` 工具链（这与我们在 Windows 上安装的类似，以支持 CGo）。这应该在您的包管理器中可用，名称类似于 `mingw-w64-clang` 或 `w64- mingw`，如果没有，您可以直接使用 [github.com/tpoechtrager/wclang](http://github.com/tpoechtrager/wclang) 上的说明进行安装。

### 在 macOS 上安装 Windows 工具

在 macOS 上安装软件包时，建议您使用 `brew.sh`。一旦 Homebrew 设置完成，可以使用以下命令安装编译器软件包：

```go
$ brew install mingw-w64
```

安装完成后，可以通过设置 `CC=x86_64-w64-mingw64-gcc` 来使用编译器，如下所示：

```go
$ CC= x86_64-w64-mingw64-gcc GOOS=windows CGO_ENABLED=1 go build .
```

在下一节中，我们将学习如何在 Linux 上安装 Windows 工具。

### 在 Linux 上安装 Windows 工具

在 Linux 上安装只需在发行版的列表中找到正确的包即可。例如，对于 **Debian** 或 **Ubuntu**，你会执行以下命令：

```go
$ sudo apt-get install gcc-mingw-w64
```

安装完成后，可以通过设置 `CC=x86_64-w64-mingw64-gcc` 来使用 CGo，如下面的命令所示：

```go
$ CC= x86_64-w64-mingw64-gcc GOOS=windows CGO_ENABLED=1 go build .
```

最后，我们将探讨如何使用手动工具链安装来为 Linux 计算机编译。

## 为 Linux 进行交叉编译

要为 Linux 进行交叉编译，我们需要 `musl-cross`（`musl` 有许多其他优点，你可以在 [www.etalabs.net/compare_libcs.html](http://www.etalabs.net/compare_libcs.html) 上了解更多）。在 Windows 上，`linux-gcc` 包是绰绰有余的。让我们逐一完成这些步骤。

### 在 macOS 上安装 Linux 编译器

为了安装依赖项以便我们可以为 Linux 进行交叉编译，我们将再次使用 Homebrew 软件包管理器 – 请参阅 *在 macOS 上安装 Windows 工具* 部分，或访问 `brew.sh` 网站以获取安装说明。

使用 Homebrew，我们可以通过打开 `HOMEBREW_BUILD_FROM_SOURCE` 变量来解决 `musl-cross` 的问题，这取决于可能过时的库版本）：

```go
$ export HOMEBREW_BUILD_FROM_SOURCE=1
$ brew install FiloSottile/musl-cross/musl-cross
```

安装完成后（这可能需要一些时间，因为它需要从源代码构建完整的编译器工具链），你应该能够为 Linux 构建程序。为此，你需要设置 `CC=x86_64-linux-musl-gcc` 环境变量，如下所示：

```go
$ CC=x86_64-linux-musl-gcc GOOS=linux CGO_ENABLED=1 go build
```

这个过程对于 Windows 也是类似的，我们将在下一节中看到。

### 在 Windows 上安装 Linux 编译器

使用 MSYS2，就像我们在 *为 macOS 交叉编译* 部分中所做的那样，我们可以安装 `gcc` 包以提供 Linux 的交叉编译：

```go
$ pacman -S gcc
```

安装完成后，我们可以通过设置 `CC=gcc` 环境变量来告诉我们的 Go 编译器使用 `gcc`。如果你按照当前示例中的说明操作，编译现在应该会成功，如下所示：

```go
$ CC=gcc GOOS=linux CGO_ENABLED=1 go build
```

在这一点上，你可能会看到由于缺少头文件而引起的额外错误。为了解决这个问题，你需要搜索并安装所需的库。这通常是因为编译器没有内置的关于 Linux 桌面图形工作方式的了解。

例如，如果你的错误信息表明找不到 `X11` 头文件，那么你可以使用 `pacman -Ss x11` 来搜索安装正确包。在这种情况下，所需的包是 `mingw-w64-libxcb`（`X11` 库的 Windows 版本），可以按照以下方式安装：

```go
$ pacman –S mingw-w64-libxcb
```

如果你找不到合适的包，你可以尝试使用 Windows 子系统 for Linux。更多信息可在 [docs.microsoft.com/en-us/windows/wsl](http://docs.microsoft.com/en-us/windows/wsl) 找到（这将在你的 Windows 桌面上带来完整的 Linux 发行版）。

# 使用 fyne-cross

为了更方便地管理用于交叉编译项目的各种开发环境和工具链，我们可以使用 `fyne-cross` 命令。这个交叉编译工具使用 **Docker** 容器下载和运行不同操作系统的开发工具，直接从您最喜欢的桌面计算机上操作。

## 安装 fyne-cross

在您安装 `fyne-cross` 之前，您需要准备一个最新的 `docker` 软件包。有关使用 Docker 的更多信息，请参阅*第九章*，*资源打包和发布准备*。

一旦安装了 Docker，您就可以使用标准的 Go 工具安装 `fyne-cross`。以下命令就足够了：

```go
$ go get github.com/fyne-io/fyne-cross
```

`fyne-cross` 二进制文件将被安装在 `$GOPATH/bin`。如果以下章节中提供的命令不起作用，请检查二进制文件是否在您的全局 `$PATH` 环境中。

## 使用 fyne-cross

安装完成后，`fyne-cross` 命令可以用来为任何目标操作系统和架构构建应用程序。它有一个必需的参数：目标系统。要从 Linux 计算机上构建 Windows 可执行文件，只需调用以下命令：

```go
$ fyne-cross windows
```

有许多可用的选项（更多信息请参阅 `fyne-cross help`）。最常用的两个参数是 `-arch`，允许您指定不同的架构，以及 `-release`，它调用 Fyne 发布管道来构建准备上传到市场的应用程序（您可以在*第十章*，*分发 – 应用商店及其他*中了解更多。为了创建适用于较小 Linux 计算机上的应用程序，例如 **Raspberry Pi**，我们可以使用以下命令：

```go
$ fyne-cross linux –arch arm64
```

要构建用于发布的 Android 应用程序，您可以使用以下命令：

```go
$ fyne-cross android -release
```

上述步骤应能帮助您以最小的硬件投资构建和分发适用于任何平台的应用程序。

请注意，如果您正在运行 `fyne-cross` 来构建 iOS 应用程序，您必须在 macOS 主机计算机上运行它。这是苹果许可协议的要求。
