

# 第四章：构建 CLIs 的流行框架

本章将探讨可用于快速开发现代 CLI 应用程序的最受欢迎的框架。在看到手动创建命令和结构化 CLI 应用程序所需的所有代码后，您将看到 Cobra 如何使开发者能够快速生成 CLI 应用程序所需的全部框架，并轻松添加新命令。

Viper 可以轻松与 Cobra 集成，以多种格式在本地或远程配置您的应用程序。选项非常广泛，开发者可以选择他们认为最适合他们项目且他们感到舒适的方式。本章将通过以下主题深入探讨 Cobra 和 Viper：

+   Cobra – 用于构建现代 CLI 应用程序的库

+   Viper – 为 CLI 提供简单配置

+   使用 Cobra 和 Viper 的基本计算器 CLI

# 技术要求

为了轻松跟随本章中的代码，您需要执行以下操作：

+   安装 Cobra CLI：[`github.com/spf13/cobra-cli`](https://github.com/spf13/cobra-cli)

+   获取 Cobra 包：[`github.com/spf13/cobra`](https://github.com/spf13/cobra)

+   获取 Viper 包：[`github.com/spf13/viper`](https://github.com/spf13/viper)

+   下载以下代码：[`github.com/PacktPublishing/Building-Modern-CLI-Applications-in-Go/tree/main/Chapter04`](https://github.com/PacktPublishing/Building-Modern-CLI-Applications-in-Go/tree/main/Chapter04)

# Cobra – 用于构建现代 CLI 应用程序的库

Cobra 是一个 Go 库，用于构建强大且现代的 CLI 应用程序。它使得定义简单和复杂的嵌套命令变得容易。Cobra `Command` 对象的广泛字段列表允许您访问完整的自文档帮助和 man 页面。Cobra 还提供了一些额外的有趣功能，包括智能 shell 自动完成、CLI 框架、代码生成以及与 Viper 配置解决方案的集成。

Cobra 库提供的命令结构比从头开始编写的命令结构要强大得多。正如之前提到的，使用 Cobra CLI 有许多优点，因此我们将通过一些示例来展示其功能。从头开始使用 Cobra 创建 CLI 只需要三个步骤。首先，确保 `cobra-cli` 已正确安装。为您的项目创建一个新的文件夹，并按顺序执行以下步骤以设置新的 CLI：

1.  切换到您的项目文件夹，`audiofile-cli`：

`cd audiofile-cli`

1.  创建一个模块并初始化您的当前目录：

`go mod init <模块路径>`

1.  初始化您的 Cobra CLI：

`cobra-cli init`

只需运行三个命令，`ls` 就会显示文件夹结构已经创建，并且可以添加命令。运行 `main.go` 文件会返回默认的长描述，但一旦添加了命令，audiofile CLI 的用法将显示帮助和示例。

如果你单独运行`cobra-cli`以查看可用的选项，你会看到只有四个命令，`add`、`completion`、`help`和`init`。由于我们已经使用`init`初始化了我们的项目，接下来，我们将使用`add`来创建新命令的模板代码。

## 创建子命令

从 Cobra CLI 添加新命令最快的方法是运行`cobra-cli`命令，`add`。要获取有关此命令的更多详细信息，我们运行`cobra-cli` `add` `–help`，这显示了运行`add`命令的语法。

要尝试从上一章创建示例`upload`命令，我们会运行以下命令：

```go
cobra-cli add upload
```

让我们快速尝试调用为`upload`命令生成的代码：

```go
  audiofile-cli go run main.go upload
upload called
```

默认情况下，返回`upload called`输出。现在，让我们看看生成的代码。在同一个文件中，有一个`init`函数，它将此命令添加到`root`或`entry`命令。

让我们清理这个文件并为我们`upload`命令填写一些细节：

```go
package cmd
import (
    "github.com/spf13/cobra"
)
// uploadCmd represents the upload command
var uploadCmd = &cobra.Command{
    Use:   "upload [audio|video] [-f|--filename]
      <filename>",
    Short: "upload an audio or video file",
    Long: `This command allows you to upload either an
      audio or video file for metadata extraction.
    To pass in a filename, use the -f or --filename flag
     followed by the path of the file.
    Examples:
    ./audiofile-cli upload audio -f audio/beatdoctor.mp3
    ./audiofile-cli upload video --filename video/
      musicvideo.mp4`,
}
func init() {
    rootCmd.AddCommand(uploadCmd)
}
```

现在，让我们为`upload`命令创建这两个新的子命令，以指定音频或视频：

```go
  cobra-cli add audio
audio created at /Users/marian/go/src/github.com/
  marianina8/audiofile-cli
  cobra-cli add video
video created at /Users/marian/go/src/github.com/
  marianina8/audiofile-cli
```

我们将`audioCmd`和`videoCmd`添加为`uploadCmd`的子命令。仅包含生成代码的`audio`命令需要修改，以便被识别为子命令。此外，我们还需要为`audio`子命令定义文件名标志。`audio`命令的`init`函数将如下所示：

```go
func init() {
    audioCmd.Flags().StringP("filename", "f", "", "audio
      file")
    uploadCmd.AddCommand(audioCmd)
}
```

文件名标志的解析发生在`Run`函数中。然而，我们希望在文件名标志缺失时返回错误，因此我们将`audioCmd`上的函数更改为返回错误并使用`RunE`方法：

```go
    RunE: func(cmd *cobra.Command, args []string) error {
        filename, err := cmd.Flags().GetString("filename")
        if err != nil {
            fmt.Printf("error retrieving filename: %s\n",
            err.Error())
            return err
        }
        if filename == "" {
            return errors.New("missing filename")
        }
        fmt.Println("uploading audio file, ", filename)
        return nil
    },
```

让我们先尝试这段代码，看看我们是否在未传递子命令时得到错误，以及当我们运行正确的示例命令时：

```go
cobra-cli add upload
```

我们现在得到一个与`upload`命令使用相关的错误消息：

```go
  go run main.go upload
This command allows you to upload either an audio or video
  file for metadata extraction.
    To pass in a filename, use the -f or --filename flag
  followed by the path of the file.
     Examples:
     ./audiofile-cli upload audio -f audio/beatdoctor.mp3
     ./audiofile-cli upload video --filename video/musicvideo.mp4
Usage:
  audiofile-cli upload [command]
Available Commands:
  audio       sets audio as the upload type
  video       sets video as the upload type
```

让我们正确地使用简写或长写标志名称运行命令：

```go
cobra-cli add upload audio [-f|--filename]
  audio/beatdoctor.mp3
```

命令随后返回预期的输出：

```go
  go run main.go upload audio -f audio/beatdoctor.mp3
uploading audio file,audio/beatdoctor.mp3
```

我们已经为`upload`命令创建了一个子命令`audio`。现在，视频和音频的实现通过单独的子命令调用。

## 全局、本地和必需标志

Cobra 允许用户定义不同类型的标志：全局和本地标志。让我们快速定义每种类型：

+   全局：全局标志对分配给它的命令及其所有子命令可用

+   本地：本地标志仅对分配给它的命令可用

注意到视频和`audio`子命令都需要一个标志来解析`filename`字符串。可能更容易将此标志设置为`uploadCmd`的全局标志。让我们从`audioCmd`的`init`函数中删除标志定义：

```go
func init() {
    uploadCmd.AddCommand(audioCmd)
}
```

相反，让我们将其添加为`uploadCmd`的全局命令，以便它也可以被`videoCmd`使用。现在`uploadCmd`的`init`函数将如下所示：

```go
var (
    Filename = ""
)
func init() {
    uploadCmd.PersistentFlags().StringVarP(&Filename,
      "filename", "f", "", "file to upload")
    rootCmd.AddCommand(uploadCmd)
}
```

这个 `PersistentFlags()` 方法将标志设置为全局和持久，适用于所有子命令。运行命令以 `upload` 音频文件仍然按预期工作：

```go
  go run main.go upload audio -f audio/beatdoctor.mp3
uploading audio file,  audio/beatdoctor.mp3
```

在 `audio` 子命令实现中，我们检查文件名是否已设置。如果我们使文件成为必需的，这是一个不必要的步骤。让我们将 `init` 改成这样：

```go
func init() {
    uploadCmd.PersistentFlags().StringVarP(&Filename,
      "filename", "f", "", "file to upload")
    uploadCmd.MarkPersistentFlagRequired("filename")
    rootCmd.AddCommand(uploadCmd)
}
```

对于本地标志，命令将是 `MarkFlagRequired("filename")`。现在让我们尝试不传递文件名标志来运行该命令：

```go
  go run main.go upload audio
Error: required flag(s) "filename" not set
Usage:
  audiofile-cli upload audio [flags]
Flags:
  -h, --help   help for audio
Global Flags:
  -f, --filename string   file to upload
exit status 1
```

Cobra 会抛出错误，而无需手动检查是否解析了文件名标志。因为音频和视频命令是 `upload` 命令的子命令，它们需要新定义的持久文件名标志。正如预期的那样，会抛出一个错误来提醒用户文件名标志尚未设置。CLI 应用程序还可以在用户输入命令错误时帮助指导用户。

## 智能建议

默认情况下，如果用户输入了错误的命令，Cobra 将提供命令建议。一个例子是当命令输入为：

```go
  go run main.go uload audio
Cobra will automatically respond with some intelligent
  suggestions:
Error: unknown command "uload" for "audiofile-cli"
Did you mean this?
     upload
Run 'audiofile-cli --help' for usage.
exit status 1
```

要禁用智能建议，只需在根命令的 `init` 函数中添加 `rootCmd.DisableSuggestions = true` 行。要更改建议的 Levenshtein 距离，修改命令上的 `SuggestionsMinimumDistance` 的值。您还可以使用命令上的 `SuggestFor` 属性来明确指定建议，这对于逻辑上是替代品但 Levenshtein 距离不接近的命令是有意义的。另一种指导 CLI 的首次用户的方法是提供您应用程序的帮助和 man 页面。Cobra 框架提供了一个简单的方法来自动生成帮助和 man 页面。

## 自动生成的帮助和 man 页面

正如我们已经看到的，输入错误的命令，或者将 `-h` 或 `–help` 标志添加到命令中，将导致 CLI 返回帮助文档，这些文档是自动从 `cobra.Command` 结构内设置的详细信息生成的。此外，通过添加以下导入可以生成 man 页面：`"github.com/spf13/cobra/doc"`。

注意

如何生成 man 页面文档的详细信息将在 *第九章* 中详细说明，*开发的同理心一面*，其中包括如何编写正确的帮助和文档。

## 为 CLI 提供动力

如您所见，使用 Cobra 库为 CLI 提供动力有许多好处，默认提供了许多功能。该库还附带了自己的 CLI，用于为新的应用程序生成脚手架以及添加命令，这些命令在 `cobra.Command` 结构中提供了所有可用的选项，使您能够构建一个强大且高度可定制的 CLI。

与从头开始编写 CLI 而不使用框架相比，您可以使用许多内置优势节省数小时的时间：命令脚手架、优秀的命令、参数和标志解析、智能建议以及自动生成的帮助文本和 man 页面。您还可以将 Cobra CLI 与 Viper 配对，以获得额外的优势来配置您的应用程序。

# Viper – CLI 的简单配置

Steve Francia，Cobra 的作者，还创建了一个配置工具 Viper，以便轻松与 Cobra 集成。对于您在本地机器上运行的单个简单应用程序，您可能最初不需要配置工具。然而，如果您的应用程序可能运行在不同的环境中，这些环境需要不同的集成、API 密钥或更适用于配置文件而不是硬编码的一般自定义，Viper 将帮助简化配置应用程序的过程。

## 配置类型

Viper 允许您以多种方式设置应用程序的配置：

+   从配置文件中读取

+   使用环境变量

+   使用远程配置系统

+   使用命令行标志

+   使用缓冲区

这些配置类型接受的配置格式包括 JSON、TOML、YAML、HCL、INI、envfile 和 Java 属性格式。为了更好地理解，让我们逐一介绍每种配置类型。

### 配置文件

假设我们根据不同的环境需要连接到不同的 URL 和端口值。我们可以设置一个 YAML 配置文件 `config.yml`，其外观如下，并存储在我们的应用程序的主文件夹中：

```go
environments:
  test:
    url: 89.45.23.123
    port: 1234
  prod:
    url: 123.23.45.89
    port: 5678
loglevel: 1
keys:
  assemblyai: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

使用代码读取配置并测试，打印出生产环境的 URL：

```go
viper.SetConfigName("config) // config filename, omit
  extension
viper.AddConfigPath(".")      // optional locations for
  searching for config files
err = viper.ReadInConfig()    // using the previous
  settings above, attempt to find and read in the
    configuration
if err != nil { // Handle errors
    panic(fmt.Errorf("err: %w \n", err))
}
fmt.Println("prod environment url:",
  viper.Get("environments.prod.url"))
```

运行代码确认 `Println` 将返回 `environments.prod.url` 为 `123.23.45.89`。

### 环境变量

配置也可以通过环境变量设置；请注意，Viper 对环境变量的识别是区分大小写的。在处理环境变量时，可以使用几种方法。

`SetEnvPrefix` 告诉 Viper，使用 `BindEnv` 和 `AutomaticEnv` 方法时使用的环境变量将被添加一个特定的唯一前缀。例如，假设测试 URL 是通过环境变量设置的：

```go
viper.SetEnvPrefix("AUDIOFILE")
viper.BindEnv("TEST_URL")
os.Setenv("AUDIOFILE_TEST_URL", "89.45.23.123") //sets the
  environment variable
fmt.Println("test environment url from environment
  variable:", viper.Get("TEST_URL"))
```

如前所述，前缀 `AUDIOFILE` 附加到传递给 `BindEnv` 或 `Get` 方法的每个环境变量的开头。当运行前面的代码时，从 `AUDIOFILE_TEST_URL` 环境变量打印出的测试环境 URL 值为 `89.45.23.123`，正如预期的那样。

### 命令行标志

Viper 支持通过几种不同类型的标志进行配置。

+   标志：使用标准 Go 库 flag 包定义的标志

+   Pflags：使用 Cobra/Viper 的 `pflag` 定义定义的标志

+   标志接口：满足 Viper 所需的标志接口的自定义结构

让我们详细检查这些内容。

#### 标志

在标准 Go 标志包的基础上构建。Viper 的 `flags` 包扩展了标准标志包的功能，提供了额外的特性，例如环境变量支持和为标志设置默认值的能力。使用 Viper 标志，您可以定义字符串、布尔值、整数和浮点数类型的标志，以及这些类型的数组。

一些示例代码可能如下所示：

```go
viper.SetDefault("host", "localhost")
viper.SetDefault("port", 1234)
viper.BindEnv("host", "AUDIOFILE_HOST")
viper.BindEnv("port", "AUDIOFILE_PORT")
```

在前面的示例中，我们为“`host`”和“`port`”标志设置了默认值，然后使用 `viper.BindEnv` 将它们绑定到环境变量。设置环境变量后，我们可以使用 `viper.GetString("host")` 和 `viper.GetInt("port")` 访问标志的值。

#### Pflags

`pflag` 是 Cobra 和 Viper 特定的标志包。值可以被解析和绑定。`viper.BindPFFlag` 用于单个标志，而 `viper.BindPFFlags` 用于标志集，用于在访问时绑定标志的值，而不是在定义时。

一旦解析并绑定标志，就可以使用 Viper 的 `Get` 方法在任何代码位置访问这些值。为了检索端口，我们会使用以下代码：

```go
port := viper.GetInt("port")
```

在 `init` 函数中，您可以定义一个命令行标志集，并在访问值后绑定它们。以下是一个示例：

```go
pflag.CommandLine.AddGoFlagSet(flag.CommandLine)
pflag.Int("port", 1234, "port")
pflag.String("url", "12.34.567.123", "url")
plag.Parse()
viper.BindPFlags(pflag.CommandLine)
```

#### 标志接口

Viper 还允许自定义标志，这些标志满足以下 Go 接口：`FlagValue` 和 `FlagValueSet`。

`FlagValue` 接口如下：

```go
type FlagValue interface {
    HasChanged() bool
    Name() string
    ValueString() string
    ValueType() string
}
```

Viper 接受的第二个接口是 `FlagValueSet`：

```go
type FlagValueSet interface {
    VisitAll(fn func(FlagValue))
}
```

满足此接口的代码示例如下：

```go
type customFlagSet struct {
    flags []customFlag
}
func (set customFlagSet) VisitAll(fn func(FlagValue)) {
    for i, flag := range set.flags {
fmt.Printf("%d: %v\n", i, flag)
        fn(flag)
    }
}
```

### 缓冲区

最后，Viper 允许用户使用缓冲区来配置他们的应用程序。使用第一个示例中配置文件中存在的相同值，我们将 YAML 数据作为原始字符串传递到一个字节切片中：

```go
var config = []byte(`
    environments:
    test:
      url: 89.45.23.123
      port: 1234
    prod:
      url: 123.23.45.89
      port: 5678
  loglevel: 1
  keys:
    assemblyai: ad915a59802309238234892390482304
`)
viper.SetConfigType("yaml")
viper.ReadConfig(bytes.NewBuffer(config))
viper.Get("environments.test.url") // 89.45.23.123
```

现在您已经了解了配置您的命令行应用程序的不同类型或方式——从文件、环境变量、标志或缓冲区——让我们看看如何监视这些配置类型的实时更改。

## 监视实时配置更改

可以监视远程和本地配置。确保所有配置路径都已添加后，调用 `WatchConfig` 方法来监视任何实时更改，并通过实现一个函数传递给 `OnConfigChange` 方法来采取行动：

```go
viper.OnConfigChange(func(event fsnotify.Event) {
    fmt.Println("Config modified:", event)
})
viper.WatchConfig()
```

要监视远程配置的更改，首先使用 `ReadRemoteConfig()` 读取远程配置，然后在 Viper 配置实例上调用 `WatchRemoteConfig()` 方法。以下是一些示例代码：

```go
var remoteConfig = viper.New()
remoteConfig.AddRemoteProvider("consul",
  "http://127.0.0.1:2380", "/config/audiofile-cli.json")
remoteConfig.SetConfigType("json")
err := remoteConfig.ReadRemoteConfig()
if err != nil {
    return err
}
remoteConfig.Unmarshal(&remote_conf)
```

以下是一个示例 goroutine，它将连续监视远程配置的更改：

```go
go func(){
    for {
        time.Sleep(time.Second * 1)
        _:= remoteConfig.WatchRemoteConfig()
        remoteConfig.Unmarshal(&remote_conf)
    }
}()
```

我认为利用配置库而不是从头开始有很多好处，这再次可以节省您数小时并加快您的开发过程。除了您可以为应用程序配置的不同方式外，您还可以提供远程配置并实时监视任何更改。这进一步创建了一个更健壮的应用程序。

# 使用 Cobra 和 Viper 的基本计算器 CLI

让我们将一些部分组合起来，使用 Cobra CLI 框架和 Viper 创建一个独立且简单的 CLI。一个我们可以轻松实现的基本想法是能够进行加、减、乘、除的基本计算器。这个演示的代码位于`Chapter-4-Demo`仓库中，供您参考。

## Cobra CLI 命令

命令是通过以下`cobra-cli`命令调用来创建的：

```go
cobra-cli add add
cobra-cli add subtract
cobra-cli add multiply
cobra-cli add divide
```

成功调用这些命令将生成每个命令的代码，以便我们填充细节。让我们展示每个命令以及它们各自相似和不同的地方。

### 添加命令

`add`命令`addCmd`被定义为`cobra.Command`类型的指针。在这里，我们设置了命令的字段：

```go
// addCmd represents the add command
var addCmd = &cobra.Command{
    Use: "add number",
    Short: "Add value",
    Run: func(cmd *cobra.Command, args []string) {
        if len(args) > 1 {
            fmt.Println("only accepts a single argument")
            return
        }
        if len(args) == 0 {
            fmt.Println("command requires input value")
            return
        }
        floatVal, err := strconv.ParseFloat(args[0], 64)
        if err != nil {
            fmt.Printf("unable to parse input[%s]: %v",
              args[0], err)
            return
        }
        value = storage.GetValue()
        value += floatVal
        storage.SetValue(value)
        fmt.Printf("%f\n", value)
    },
}
```

让我们快速浏览一下`Run`字段，它是一个一等函数。在进行任何计算之前，我们检查`args`。命令只接受一个数值字段；如果更多或更少，将打印用法说明并返回以下内容：

```go
if len(args) > 1 {
    fmt.Println("only accepts a single argument")
    return
}
if len(args) == 0 {
    fmt.Println("command requires input value")
    return
}
```

我们取第一个且唯一的参数，返回它，将其设置在`args[0]`中，并使用以下代码将其解析为一个扁平变量。如果转换到`float64`值失败，则命令会打印出无法解析输入的消息，然后返回：

```go
floatVal, err := strconv.ParseFloat(args[0], 64)
if err != nil {
    fmt.Printf("unable to parse input[%s]: %v", args[0],
      err)
    return
}
```

如果转换成功，并且字符串转换没有返回错误，那么我们为`floatVal`设置了一个值。在我们的基本计算器 CLI 中，我们将其存储在文件中，这是本例中存储它的最简单方式。`storage`包和 Viper 在配置中的使用将在命令之后讨论。在更高层次上，我们从存储中获取当前值，将其应用于`floatVal`，然后将其保存回存储：

```go
value = storage.GetValue()
value += floatVal
storage.SetValue(value)
```

最后但同样重要的是，将值打印回用户：

```go
fmt.Printf("%f\n", value)
```

这就结束了我们对`add`命令的`Run`函数的探讨。`Use`字段描述了用法，`Short`字段给出了命令的简要描述。这就结束了添加命令的浏览。在各自的命令上，减法、乘法和除法的`Run`函数非常相似，所以我只会指出一些需要注意的差异。

### 减法命令

`subtractCmd`的`Run`函数使用了相同的代码，只有一个小的例外。我们不是将值添加到`floatVal`，而是用以下行进行减法操作：

```go
value -= floatVal
```

### 乘法命令

同样的代码用于`multiplyCmd`的`Run`函数，除了我们用以下行进行乘法操作：

```go
value *= floatVal
```

### 除法命令

最后，同样的代码用于`divideCmd`的`Run`函数，除了用`floatVal`进行除法操作：

```go
value /= floatVal
```

### 清除命令

`clear`命令将存储的值重置为`0`。`clearCmd`的代码简短且简单：

```go
// clearCmd represents the clear command
var clearCmd = &cobra.Command{
    Use: "clear",
    Short: "Clear result",
    Run: func(cmd *cobra.Command, args []string) {
        if len(args) > 0 {
            fmt.Println("command does not accept args")
            return
        }
        storage.SetValue(0)
        fmt.Println(0.0)
    },
}
```

我们检查是否传递了任何 `args`，如果是，则打印该命令不接受任何参数并返回。如果命令是 `./calculator clear`，则存储 `0` 值并将其打印回用户。

## Viper 配置

现在，让我们讨论使用 Viper 配置的简单方法。为了跟踪对其应用了操作的值，我们需要存储此值。存储数据的最简单方法是将其保存在文件中。

### 存储包

在存储库中，有一个名为 `storage/storage.go` 的文件，其中包含以下代码来设置值：

```go
func SetValue(floatVal float64) error {
    f, err := os.OpenFile(viper.GetString("filename"),
       os.O_RDWR|os.O_CREATE|os.O_TRUNC, 0755)
    if err != nil {
        return err
    }
    defer f.Close()
    _, err = f.WriteString(fmt.Sprintf("%f", floatVal))
    if err != nil {
        return err
    }
    return nil
}
```

此代码将数据写入 `viper.GetString("filename")` 返回的文件名。从文件获取值的代码如下：

```go
func GetValue() float64 {
    dat, err := os.ReadFile(viper.GetString("filename"))
    if err != nil {
        fmt.Println("unable to read from storage")
        return 0
    }
    floatVal, err := strconv.ParseFloat(string(dat), 64)
    if err != nil {
        return 0
    }
    return floatVal
}
```

再次强调，获取文件名、读取、解析然后返回包含的数据的方法是相同的。

### 初始化配置

在 `main` 函数中，我们在执行命令之前调用 Viper 方法来初始化我们的配置：

```go
func main() {
    viper.AddConfigPath(".")
    viper.SetConfigName("config")
    viper.SetConfigType("json")
    err := viper.ReadInConfig()
    if err != nil {
        fmt.Println("error reading in config: ", err)
    }
    cmd.Execute()
}
```

注意

`AddConfigPath` 方法用于设置 Viper 搜索配置文件的路径。`SetConfigName` 方法允许你设置配置文件名，不带扩展名。实际的配置文件是 `config.json`，但我们传递 `config`。最后，`ReadInConfig` 方法读取配置，使其在整个应用程序中可用。

### 配置文件

最后，配置文件 `config.json` 存储文件名值：

```go
{
   "filename": "storage/result"
}
```

此文件位置适用于基于 UNIX 或 Linux 的系统。根据您的平台进行更改，并亲自尝试演示！

## 运行基本计算器

要在 UNIX 或 Linux 上快速构建基本计算器，请运行 `go build -o calculator main.go`。在 Windows 上，请运行 `go build -o calculator.exe main.go`。

我在我的基于 UNIX 的终端上运行了这个应用程序，并得到了以下输出：

```go
% ./calculator clear
0
% ./calculator add 123456789
123456789.000000
% ./calculator add 987654321
1111111110.000000
% ./calculator add 1
1111111111.000000
% ./calculator multiply 8
8888888888.000000
% ./calculator divide 222222222
40.000000
% ./calculator subtract 40
0.000000
```

希望这个简单的演示能让你对如何使用 Cobra CLI 加速开发以及 Viper 以简单方式配置应用程序有一个良好的理解。

# 摘要

本章向您介绍了构建现代 CLI 最受欢迎的库——Cobra——及其配置伙伴库——Viper。详细解释了 Cobra 包，并使用示例描述了 CLI 的用法。我们通过示例引导你使用 Cobra CLI 生成初始应用程序代码，添加新命令和修改脚手架，以自动生成有用的帮助和 man 页面。Viper 作为与 Cobra 完美搭配的配置工具，其许多选项也进行了详细描述。

在下一章中，我们将讨论如何处理 CLI 的输入——无论是命令、参数或标志形式的文本，还是允许你退出终端仪表板的控制字符。我们还将讨论处理这种输入的不同方式以及如何将结果输出给用户。

# 问题

1.  如果你想定义一个可以被命令及其所有子命令访问的标志，应该定义什么样的标志以及如何定义？

1.  Viper 接受哪些配置格式选项？

# 答案

1.  在定义命令上的标志时使用`PersistentFlag()`方法创建的全局标志。

1.  JSON、TOML、YAML、HCL、INI、envfile 和 Java 属性格式。

# 进一步阅读

+   *Cobra – Go 中现代 CLI 应用程序的框架* ([`cobra.dev/`](https://cobra.dev/)) 为 Cobra 提供了广泛的文档，包括使用 Cobra 的示例以及指向 Viper 文档的链接

# 第二部分：CLI 的内外

本部分重点介绍命令行应用程序的解剖结构以及它可以接收的不同类型的输入，例如子命令、参数和标志，以及其他输入，如标准输入、信号和控制字符。它还涵盖了处理数据的不同方法以及如何返回结果，包括与外部命令或 API 服务交互时处理错误和超时。本章还突出了 Go 的跨平台能力，使用诸如 os、time、path 和 runtime 等包。

本部分包含以下章节：

+   *第五章*，*定义命令行进程*

+   *第六章*，*调用外部进程，处理错误和超时*

+   *第七章*，*为不同平台开发*
