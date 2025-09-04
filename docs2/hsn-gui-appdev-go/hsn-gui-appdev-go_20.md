# 第十六章：交叉编译器设置

当构建需要访问原生 API 的应用程序时，我们可以使用 CGo。尽管对于常规开发来说并不困难，但这确实使得交叉编译变得更加复杂。对于您想要为每个目标平台构建的，都必须有一个 C 编译器知道如何创建原生二进制文件。本附录概述了设置本书记载中之前提到的每个组合所需的交叉编译目标步骤。

大多数 Go 应用程序不需要为交叉编译设置此环境，因为 Go 编译器被设计为支持所有平台。如果生成的应用程序（或它们使用的工具包）通过 CGo 链接到操作系统库，则需要额外的步骤。

# 使用 CGo 为 macOS 进行交叉编译

当为 macOS 进行交叉编译时，需要安装苹果的 SDK（软件开发工具包）以及一个合适的编译器。Windows（使用 MSYS2——在之前的 附录，*安装详情*）和 Linux 的说明几乎相同；主要工作是安装 macOS SDK。

# 从 Linux 或 Windows 到 macOS

为了准备交叉编译到 Darwin，我们必须安装 macOS SDK 和一个可以使用的构建工具链。最简单的方法是使用 *osxcross* 项目。以下示例展示了如何下载和安装 SDK 和工具，以便在没有使用 Macintosh 计算机的情况下为 macOS 构建应用程序。此说明使用 Linux，但对于使用 MSYS2 或 Cygwin 命令提示符的 Windows 开发者来说，过程是相同的。

我们将使用 `clang` 而不是 `gcc`，因为它的设计更便携。为了使此过程正常工作，您需要使用您的包管理器安装 `clang`、`cmake` 和 `libxml2-dev`：

+   在 Linux 上使用：`pacman -S clang cmake libxml2-dev`（或 `apt-get` 或 `yum`，具体取决于您的发行版）

+   在 Windows 上使用：`pacman -S mingw-w64-x86_64-clang mingw-w64-x86_64-cmake mingw-w64-x86_64-libxml2`

接下来，我们需要下载 macOS SDK，它包含在 Xcode 中。如果您还没有 Apple 开发者账户，您需要注册并同意他们的条款和条件。使用此账户，登录到 [`developer.apple.com/download/more/?name=Xcode%207.3`](https://developer.apple.com/download/more/?name=Xcode%207.3) 下载 `XCode.dmg`（推荐使用 osxcross 的 7.3.1 版本）。

然后，我们可以安装 osxcross 工具——首先使用 `git clone https://github.com/tpoechtrager/osxcross.git` 下载它，然后切换到下载的目录。使用这些工具，我们可以使用提供的包工具 `./tools/gen_sdk_package_darling_dmg.sh <path to Xcode.dmg>` 从下载的 `Xcode.dmg` 文件中提取 macOS SDK。生成的 `MacOSX10.11.sdk.tar.xz` 文件应该复制到 `tarballs/` 目录。

最后，通过执行 `./build.sh` 来构建 osxcross 编译器扩展。之后，应该会创建一个名为 `target/bin/` 的新目录，你需要将其添加到你的 `PATH` 环境变量中。现在可以通过设置环境变量 `CC=o32-clang` 来在 CGo 构建中使用编译器。关于此过程以及如何适应其他平台的更多详细信息，可以在 osxcross 项目网站上找到，网址为 [`github.com/tpoechtrager/osxcross`](https://github.com/tpoechtrager/osxcross)。

# 使用 CGo 为 Windows 进行交叉编译

从其他平台为 Windows 构建需要安装 `mingw` 工具链（类似于我们在 Windows 上安装以支持 CGo 的工具链）。这应该在你的包管理器中可用，名称类似于 `mingw-w64-clang` 或 `w64-mingw`，但如果不可用，你可以直接使用 [`github.com/tpoechtrager/wclang`](https://github.com/tpoechtrager/wclang) 上的说明进行安装。

# 从 macOS 到 Windows

要在 macOS 上安装软件包，建议使用 Homebrew 包管理器。你可能已经从本书的早期章节中安装了它（例如，在设置 GTK+ 库时），如果没有，你可以从 [`brew.sh`](https://brew.sh) 下载它。一旦设置了 Homebrew，就可以使用 `brew install mingw-w64` 来安装编译器软件包。

安装完成后，可以通过设置 `CC=x86_64-w64-mingw32-gcc`（用于 C 工具链）和 `CXX=x86_64-w64-mingw32-g++`（用于 C++ 需求）来使用编译器与 CGo 一起。

# 从 Linux 到 Windows

在 Linux 上安装只需在发行版的列表中找到正确的软件包即可。例如，对于 Debian 或 Ubuntu，你会执行 `sudo apt-get install gcc-mingw-w64`。

安装完成后，可以通过设置 `CC=x86_64-w64-mingw32-gcc`（用于 C 工具链）和 `CXX=x86_64-w64-mingw32-g++`（用于 C++ 需求）来使用编译器与 CGo 一起。

# 使用 CGo 为 Linux 进行交叉编译

要为 Linux 进行交叉编译，我们需要一个可以构建 Linux 二进制文件的 GCC 或兼容编译器。在 macOS 上，最简单的平台是 musl-cross（musl 有许多其他优点，你可以在 [www.etalabs.net/compare_libcs.html](http://www.etalabs.net/compare_libcs.html) 上了解更多）。在 Windows 上，`linux-gcc` 软件包将是合适的。让我们逐一处理这些步骤。

# 从 macOS 到 Linux

要为 Linux 交叉编译安装依赖项，我们将再次使用 Homebrew 包管理器——请参阅前面的章节或 [`brew.sh/`](https://brew.sh/) 以获取安装说明。使用 Homebrew，我们将通过打开终端并执行以下命令来安装适当的软件包（`HOMEBREW_BUILD_FROM_SOURCE` 变量解决了 musl-cross 依赖于可能过时的库版本的问题）：

+   `export HOMEBREW_BUILD_FROM_SOURCE=1`

+   `brew install FiloSottile/musl-cross/musl-cross`

安装完成后（这可能需要一些时间，因为它需要从源代码构建完整的编译器工具链），你应该能够为 Linux 进行构建。为此，你需要设置环境变量，`CC=x86_64-linux-musl-gcc` 和 `CXX=x86_64-linux-musl-g++`。

# 从 Windows 到 Linux

使用 MSYS2，我们可以安装`gcc`包以提供 Linux 的交叉编译：

```go
pacman -S gcc
```

安装完成后，我们可以通过设置环境变量 `CC=gcc` 来告诉我们的 Go 编译器使用 `gcc`。现在，按照你当前示例中的说明进行编译应该会成功，例如以下内容：

```go
GOOS=linux CGO_ENABLED=1 CC=gcc go build
```

在这个阶段，你可能可能会看到由于缺少头文件而导致的额外错误。为了解决这个问题，你需要搜索并安装所需的库。例如，如果你的错误表明 SDL 找不到，那么你会使用 `pacman -Ss sdl` 来搜索要安装的正确包。如果你找不到合适的包，你可能需要安装 Cygwin [www.cygwin.com/](https://www.cygwin.com/)（因为它有一个更大的包库）或者 Windows 子系统（WSL）[docs.microsoft.com/en-us/windows/wsl/](https://docs.microsoft.com/en-us/windows/wsl/)（因为它将完整的 Linux 发行版带到你的 Windows 桌面）。
