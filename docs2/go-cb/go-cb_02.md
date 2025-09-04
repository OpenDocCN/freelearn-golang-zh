# 命令行工具

在本章中，将涵盖以下配方：

+   使用命令行标志

+   使用命令行参数

+   读取和设置环境变量

+   使用 TOML、YAML 和 JSON 进行配置

+   使用 Unix 管道进行工作

+   捕获和处理信号

+   ANSI 着色应用程序

# 简介

命令行应用程序是处理用户输入和输出的最简单方法之一。本章将专注于基于命令行的交互，如命令行参数、配置和环境变量。它将以一个用于在 Unix 和 Bash for Windows 中着色文本输出的库结束。

通过本章中的配方，你应该能够处理预期的和意外的用户输入。信号配方是用户可能向你的应用程序发送意外信号的例子，而管道配方是相对于标志或命令行参数获取用户输入的一个很好的替代方案。

ANSI 颜色配方可能会提供一些清理用户输出的示例。例如，在日志记录中，根据文本的目的对文本进行着色有时可以使大量文本变得更加清晰。

# 使用命令行标志

`flag` 包使得向 Go 应用程序添加命令行标志参数变得简单。它有一些缺点--你往往需要重复大量代码来添加标志的缩写版本，并且它们按字母顺序排列在帮助提示中。有一些第三方库试图解决这些缺点，但本章将专注于标准库版本，而不是那些库。

# 准备工作

根据以下步骤配置你的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到你的操作系统上，并配置你的 `GOPATH` 环境变量：

1.  打开终端/控制台应用程序，导航到你的 `GOPATH/src` 并创建一个项目目录，例如，`$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序创建并导航到 `chapter2/flags` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter2/flags`](https://github.com/agtorre/go-cookbook/tree/master/chapter2/flags) 复制测试，或者将其作为练习编写一些你自己的代码！

1.  创建一个名为 `flags.go` 的文件，内容如下：

```go
        package main

        import (
             "flag"
             "fmt"
        )

        // Config will be the holder for our flags
        type Config struct {
             subject string
             isAwesome bool
             howAwesome int
             countTheWays CountTheWays
        }

        // Setup initializes a config from flags that
        // are passed in
        func (c *Config) Setup() {
            // you can set a flag directly like so:
            // var someVar = flag.String("flag_name", "default_val",           
            // "description")
            // but in practice putting it in a struct is generally 
            // better longhand
            flag.StringVar(&c.subject, "subject", "", "subject is a           
            string, it defaults to empty")
            // shorthand
            flag.StringVar(&c.subject, "s", "", "subject is a string, 
            it defaults to empty (shorthand)")

           flag.BoolVar(&c.isAwesome, "isawesome", false, "is it 
           awesome or what?")
           flag.IntVar(&c.howAwesome, "howawesome", 10, "how awesome 
           out of 10?")

           // custom variable type
           flag.Var(&c.countTheWays, "c", "comma separated list of 
           integers")
        }

        // GetMessage uses all of the internal
        // config vars and returns a sentence
        func (c *Config) GetMessage() string {
            msg := c.subject
            if c.isAwesome {
                msg += " is awesome"
            } else {
                msg += " is NOT awesome"
            }

            msg = fmt.Sprintf("%s with a certainty of %d out of 10\. Let 
            me count the ways %s", msg, c.howAwesome, 
            c.countTheWays.String())
            return msg
        }

```

1.  创建一个名为 `custom.go` 的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "strconv"
            "strings"
        )

        // CountTheWays is a custom type that
        // we'll read a flag into
        type CountTheWays []int

        func (c *CountTheWays) String() string {
            result := ""
            for _, v := range *c {
                if len(result) > 0 {
                    result += " ... "
                }
                result += fmt.Sprint(v)
            }
            return result
        }

        // Set will be used by the flag package
        func (c *CountTheWays) Set(value string) error {
            values := strings.Split(value, ",")

            for _, v := range values {
                i, err := strconv.Atoi(v)
                if err != nil {
                    return err
                }
                *c = append(*c, i)
            }

            return nil
        }

```

1.  创建一个名为 `main.go` 的文件，内容如下：

```go
        package main

        import (
            "flag"
            "fmt"
        )

        func main() {
            // initialize our setup
            c := Config{}
            c.Setup()

            // generally call this from main
            flag.Parse()

            fmt.Println(c.GetMessage())
        }

```

1.  在命令行上运行以下命令：

```go
 go build ./flags -h

```

1.  尝试这些以及其他一些参数，你应该会看到以下输出：

```go
 $ go build 
 $ ./flags -h 
 Usage of ./flags:
 -c value
 comma separated list of integers
 -howawesome int
 how awesome out of 10? (default 10)
 -isawesome
 is it awesome or what? (default false)
 -s string
 subject is a string, it defaults to empty (shorthand)
 -subject string
 subject is a string, it defaults to empty
 $ ./flags -s Go -isawesome -howawesome 10 -c 1,2,3 
 Go is awesome with a certainty of 10 out of 10\. Let me count 
      the ways 1 ... 2 ... 3

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`，并确保所有测试通过。

# 它是如何工作的...

这个配方试图展示 `flag` 包的大部分常见用法。它展示了自定义变量类型、各种内置变量、缩写标志以及将所有标志写入一个公共结构体。这是第一个需要主函数的配方，因为 `flag` 的主要用法（`flag.Parse()`）应该从主函数中调用。因此，正常的示例目录被省略了。

此应用程序的示例用法表明，你可以自动获得 `-h` 来获取包含的标志列表。还有一些其他需要注意的事情，比如没有参数调用的布尔标志，以及标志的顺序并不重要。

`flag` 包是一种快速为命令行应用程序结构化输入并提供灵活指定预先用户输入的方法，例如设置日志级别或应用程序的详细程度。在命令行参数配方中，我们将探索标志集并在它们之间使用参数进行切换。

# 使用命令行参数

上一配方中的标志是一种命令行参数。本章将通过构建一个支持嵌套子命令的命令来扩展这些参数的其他用途。这将演示标志集并使用传递给应用程序的位置参数。

与上一个配方一样，这个配方也需要一个主函数来运行。有许多第三方包可以处理复杂的嵌套参数和标志，但我们将研究如何仅使用标准库来完成这项工作。

# 准备工作

参考在 *使用命令行标志* 配方中的 *准备工作* 部分的步骤。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建一个名为 `chapter2/cmdargs` 的新目录并导航到该目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter2/cmdargs`](https://github.com/agtorre/go-cookbook/tree/master/chapter2/cmdargs) 复制测试，或者将其作为练习来编写你自己的代码！

1.  创建一个名为 `cmdargs.go` 的文件，内容如下：

```go
        package main
        import (
            "flag"
            "fmt"
            "os"
        )
        const version = "1.0.0"
        const usage = `Usage:
        %s [command]
        Commands:
            Greet
            Version
        `
        const greetUsage = `Usage:
        %s greet name [flag]
        Positional Arguments:
            name
                the name to greet
        Flags:
        `
        // MenuConf holds all the levels
        // for a nested cmd line argument
        type MenuConf struct {
            Goodbye bool
        }
        // SetupMenu initializes the base flags
        func (m *MenuConf) SetupMenu() *flag.FlagSet {
            menu := flag.NewFlagSet("menu", flag.ExitOnError)
            menu.Usage = func() {
                fmt.Printf(usage, os.Args[0])
                menu.PrintDefaults()
            }
            return menu
        }
        // GetSubMenu return a flag set for a submenu
        func (m *MenuConf) GetSubMenu() *flag.FlagSet {
            submenu := flag.NewFlagSet("submenu", flag.ExitOnError)
            submenu.BoolVar(&m.Goodbye, "goodbye", false, "Say goodbye 
            instead of hello")
            submenu.Usage = func() {
                fmt.Printf(greetUsage, os.Args[0])
                submenu.PrintDefaults()
            }
            return submenu
        }
        // Greet will be invoked by the greet command
        func (m *MenuConf) Greet(name string) {
            if m.Goodbye {
                fmt.Println("Goodbye " + name + "!")
            } else {
                fmt.Println("Hello " + name + "!")
            }
        }
        // Version prints the current version that is
        // stored as a const
        func (m *MenuConf) Version() {
            fmt.Println("Version: " + version)
        }

```

1.  创建一个名为 `main.go` 的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "os"
            "strings"
        )

        func main() {
            c := MenuConf{}
            menu := c.SetupMenu()
            menu.Parse(os.Args[1:])

         // we use arguments to switch between commands
         // flags are also an argument
         if len(os.Args) > 1 {
             // we don't care about case
             switch strings.ToLower(os.Args[1]) {
             case "version":
                 c.Version()
             case "greet":
                 f := c.GetSubMenu()
                 if len(os.Args) < 3 {
                     f.Usage()
                     return
                 }
                 if len(os.Args) > 3 {
                 if.Parse(os.Args[3:])
                 }
                 c.Greet(os.Args[2])
             default:
                 fmt.Println("Invalid command")
                 menu.Usage()
                 return
             }
          } else {
             menu.Usage()
             return
          }
        }

```

1.  运行 `go build`。

1.  运行以下命令并尝试一些其他参数组合：

```go
 $./cmdargs -h 
 Usage:

 ./cmdargs [command]

 Commands:
 Greet
 Version

 $./cmdargs version
 Version: 1.0.0

 $./cmdargs greet
 Usage:

 ./cmdargs greet name [flag]

 Positional Arguments:
 name
 the name to greet

 Flags:
 -goodbye
 Say goodbye instead of hello

 $./cmdargs greet reader
 Hello reader!

 $./cmdargs greet reader -goodbye
 Goodbye reader!

```

1.  如果你复制或编写了自己的测试，向上移动一个目录并运行 `go test`，确保所有测试通过。

# 它是如何工作的...

标志集可以用来设置独立的预期参数列表、用法字符串等。开发者需要对多个参数进行验证，正确解析命令的正确参数子集，并定义用法字符串。这可能会出错，并且需要大量迭代才能完全正确。

`flag`包使解析参数变得容易，并包括获取标志数量、参数等便利方法。本菜谱演示了使用包括包级配置、必需的位置参数、多级命令使用以及如何将这些内容拆分为多个文件或包（如果需要）的基本方法来构建复杂的命令行应用程序。

# 读取和设置环境变量

环境变量是将状态传递给应用程序的另一种方式，除了从文件中读取数据或通过命令行显式传递之外。本菜谱将探讨一些非常基本的环境变量的获取和设置，然后使用高度有用的第三方库[`github.com/kelseyhightower/envconfig`](https://github.com/kelseyhightower/envconfig)。

我们将构建一个可以通过 JSON 或通过环境变量读取配置的应用程序。下一个菜谱将进一步探讨其他格式，包括 TOML 和 YAML。

# 准备就绪

根据以下步骤配置你的环境：

1.  请参考*准备就绪*部分中*使用命令行标志*菜谱的步骤。

1.  运行`go get github.com/kelseyhightower/envconfig/`命令。

1.  运行`go get github.com/pkg/errors/`命令。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建一个名为`chapter2/envvar`的新目录，并导航到该目录。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter2/envvar`](https://github.com/agtorre/go-cookbook/tree/master/chapter2/envvar)复制测试，或者将其作为练习编写一些自己的代码！

1.  创建一个名为`config.go`的文件，内容如下：

```go
        package envvar

        import (
            "encoding/json"
            "os"

            "github.com/kelseyhightower/envconfig"
            "github.com/pkg/errors"
        )

        // LoadConfig will load files optionally from the json file 
        // stored at path, then will override those values based on the 
        // envconfig struct tags. The envPrefix is how we prefix our 
        // environment variables.
        func LoadConfig(path, envPrefix string, config interface{}) 
        error {
            if path != "" {
               err := LoadFile(path, config)
               if err != nil {
                   return errors.Wrap(err, "error loading config from 
                   file")
               }
            }
            err := envconfig.Process(envPrefix, config)
            return errors.Wrap(err, "error loading config from env")
        }

        // LoadFile unmarshalls a json file into a config struct
        func LoadFile(path string, config interface{}) error {
            configFile, err := os.Open(path)
            if err != nil {
                return errors.Wrap(err, "failed to read config file")
         }
         defer configFile.Close()

         decoder := json.NewDecoder(configFile)
         if err = decoder.Decode(config); err != nil {
             return errors.Wrap(err, "failed to decode config file")
         }
         return nil
        }

```

1.  创建一个名为`example`的新目录。

1.  导航到`example`。

1.  创建一个名为`main.go`的文件，内容如下，并确保你修改了`envvar`导入以使用步骤 1 中设置的路径：

```go
        package main

        import (
            "bytes"
            "fmt"
            "io/ioutil"
            "os"

            "github.com/agtorre/go-cookbook/chapter2/envvar"
        )

        // Config will hold the config we
        // capture from a json file and env vars
        type Config struct {
            Version string `json:"version" required:"true"`
            IsSafe bool `json:"is_safe" default:"true"`
            Secret string `json:"secret"`
        }

        func main() {
            var err error

            // create a temporary file to hold
            // an example json file
            tf, err := ioutil.TempFile("", "tmp")
            if err != nil {
                panic(err)
            }
            defer tf.Close()
            defer os.Remove(tf.Name())

            // create a json file to hold
            // our secrets
            secrets := `{
                "secret": "so so secret"
            }`

            if _, err =   
            tf.Write(bytes.NewBufferString(secrets).Bytes()); 
            err != nil {
                panic(err)
            }

            // We can easily set environment variables
            // as needed
            if err = os.Setenv("EXAMPLE_VERSION", "1.0.0"); err != nil 
            {
                panic(err)
            }
            if err = os.Setenv("EXAMPLE_ISSAFE", "false"); err != nil {
                panic(err)
            }

            c := Config{}
            if err = envvar.LoadConfig(tf.Name(), "EXAMPLE", &c);
            err != nil {
                panic(err)
            }

            fmt.Println("secrets file contains =", secrets)

            // We can also read them
            fmt.Println("EXAMPLE_VERSION =", 
            os.Getenv("EXAMPLE_VERSION"))
            fmt.Println("EXAMPLE_ISSAFE =", 
            os.Getenv("EXAMPLE_ISSAFE"))

            // The final config is a mix of json and environment
            // variables
            fmt.Printf("Final Config: %#v\n", c)
        }

```

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

1.  你应该看到以下输出：

```go
 $ go run main.go
 secrets file contains = {
 "secret": "so so secret"
 }
 EXAMPLE_VERSION = 1.0.0
 EXAMPLE_ISSAFE = false
 Final Config: main.Config{Version:"1.0.0", IsSafe:false, 
      Secret:"so so secret"}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录，并运行`go test`，确保所有测试都通过。

# 它是如何工作的...

使用`os`包读取和写入环境变量相当简单。本菜谱使用的第三方库`envconfig`是一种巧妙的方式来捕获环境变量，并使用结构体标签指定某些要求。

`LoadConfig`函数是一种灵活的方式，可以从各种来源拉取配置信息，而无需大量开销或过多的额外依赖。将主要配置转换为 JSON 以外的格式或始终使用环境变量都很简单。

此外，请注意错误的使用。在本菜谱的代码中，我们包装了错误，这样我们就可以在不丢失原始错误信息的情况下注释错误。关于这一点，在第四章，*Go 中的错误处理*中将有更多细节。

# 使用 TOML、YAML 和 JSON 进行配置

Go 使用第三方库支持许多配置格式。其中三种最受欢迎的数据格式是 TOML、YAML 和 JSON。Go 可以直接支持 JSON，而其他格式则提供了如何对这些格式进行序列化/反序列化或编码/解码数据的线索。这些格式在配置之外还有许多好处，但本章将主要关注将 Go 结构体转换为配置结构体的形式。这个配方将探索使用这些格式的基本输入和输出。

这些格式还提供了一个接口，通过该接口 Go 和用其他语言编写的应用程序可以共享相同的配置。还有许多处理这些格式并简化它们使用的工具。

# 准备工作

根据以下步骤配置你的环境：

1.  请参考 *Using command-line flags* 配方中的 *Getting ready* 部分的步骤*.*。

1.  运行 `go get github.com/BurntSushi/toml` 命令。

1.  运行 `go get github.com/go-yaml/yaml` 命令。

# 如何做到...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建一个名为 `chapter2/confformat` 的新目录，并导航到该目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter2/confformat`](https://github.com/agtorre/go-cookbook/tree/master/chapter2/confformat) 复制测试，或者将其作为练习编写一些你自己的代码！

1.  创建一个名为 `toml.go` 的文件，内容如下：

```go
        package confformat

        import (
            "bytes"

            "github.com/BurntSushi/toml"
        )

        // TOMLData is our common data struct
        // with TOML struct tags
        type TOMLData struct {
            Name string `toml:"name"`
            Age int `toml:"age"`
        }

        // ToTOML dumps the TOMLData struct to
        // a TOML format bytes.Buffer
        func (t *TOMLData) ToTOML() (*bytes.Buffer, error) {
            b := &bytes.Buffer{}
            encoder := toml.NewEncoder(b)

            if err := encoder.Encode(t); err != nil {
                return nil, err
            }
            return b, nil
        }

        // Decode will decode into TOMLData
        func (t *TOMLData) Decode(data []byte) (toml.MetaData, error) {
            return toml.Decode(string(data), t)
        }

```

1.  创建一个名为 `yaml.go` 的文件，内容如下：

```go
        package confformat

        import (
            "bytes"

            "github.com/go-yaml/yaml"
        )

        // YAMLData is our common data struct
        // with YAML struct tags
        type YAMLData struct {
            Name string `yaml:"name"`
            Age int `yaml:"age"`
        }

        // ToYAML dumps the YAMLData struct to
        // a YAML format bytes.Buffer
        func (t *YAMLData) ToYAML() (*bytes.Buffer, error) {
            d, err := yaml.Marshal(t)
            if err != nil {
                return nil, err
            }

            b := bytes.NewBuffer(d)

            return b, nil
        }

        // Decode will decode into TOMLData
        func (t *YAMLData) Decode(data []byte) error {
            return yaml.Unmarshal(data, t)
        }

```

1.  创建一个名为 `json.go` 的文件，内容如下：

```go
        package confformat

        import (
            "bytes"
            "encoding/json"
            "fmt"
        )

        // JSONData is our common data struct
        // with JSON struct tags
        type JSONData struct {
            Name string `json:"name"`
            Age int `json:"age"`
        }

        // ToJSON dumps the JSONData struct to
        // a JSON format bytes.Buffer
        func (t *JSONData) ToJSON() (*bytes.Buffer, error) {
            d, err := json.Marshal(t)
            if err != nil {
                return nil, err
            }

            b := bytes.NewBuffer(d)

            return b, nil
        }

        // Decode will decode into JSONData
        func (t *JSONData) Decode(data []byte) error {
            return json.Unmarshal(data, t)
        }

        // OtherJSONExamples shows ways to use types
        // beyond structs and other useful functions
        func OtherJSONExamples() error {
            res := make(map[string]string)
            err := json.Unmarshal([]byte(`{"key": "value"}`), &res)
            if err != nil {
                return err
            }

            fmt.Println("We can unmarshal into a map instead of a 
            struct:", res)

            b := bytes.NewReader([]byte(`{"key2": "value2"}`))
            decoder := json.NewDecoder(b)

            if err := decoder.Decode(&res); err != nil {
                return err
            }

            fmt.Println("we can also use decoders/encoders to work with 
            streams:", res)

            return nil
        }

```

1.  创建一个名为 `marshal.go` 的文件，内容如下：

```go
        package confformat

        import "fmt"

        // MarshalAll takes some data stored in structs
        // and converts them to the various data formats
        func MarshalAll() error {
            t := TOMLData{
                Name: "Name1",
                Age: 20,
            }

            j := JSONData{
                Name: "Name2",
                Age: 30,
            }

            y := YAMLData{
                Name: "Name3",
                Age: 40,
            }

            tomlRes, err := t.ToTOML()
            if err != nil {
                return err
            }

            fmt.Println("TOML Marshal =", tomlRes.String())

            jsonRes, err := j.ToJSON()
            if err != nil {
                return err
            }

            fmt.Println("JSON Marshal=", jsonRes.String())

            yamlRes, err := y.ToYAML()
            if err != nil {
                return err
            }

            fmt.Println("YAML Marshal =", yamlRes.String())
                return nil
        }

```

1.  创建一个名为 `unmarshal.go` 的文件，内容如下：

```go
        package confformat

        import "fmt"

        const (
            exampleTOML = `name="Example1"
        age=99
            `

            exampleJSON = `{"name":"Example2","age":98}`

            exampleYAML = `name: Example3
        age: 97 
            `
        )

        // UnmarshalAll takes data in various formats
        // and converts them into structs
        func UnmarshalAll() error {
            t := TOMLData{}
            j := JSONData{}
            y := YAMLData{}

            if _, err := t.Decode([]byte(exampleTOML)); err != nil {
                return err
            }
            fmt.Println("TOML Unmarshal =", t)

            if err := j.Decode([]byte(exampleJSON)); err != nil {
                return err
            }
            fmt.Println("JSON Unmarshal =", j)

            if err := y.Decode([]byte(exampleYAML)); err != nil {
                return err
            }
            fmt.Println("Yaml Unmarshal =", y)
                return nil
            }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个 `main.go` 文件，内容如下，并确保你修改 `confformat` 导入以使用步骤 1 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter2/confformat"

        func main() {
            if err := confformat.MarshalAll(); err != nil {
                panic(err)
            }

            if err := confformat.UnmarshalAll(); err != nil {
                panic(err)
            }

            if err := confformat.OtherJSONExamples(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`。

1.  你还可以运行以下命令：

```go
 go build ./example

```

1.  你应该看到以下内容：

```go
 $ go run main.go
 TOML Marshal = name = "Name1"
 age = 20

 JSON Marshal= {"name":"Name2","age":30}
 YAML Marshal = name: Name3
 age: 40

 TOML Unmarshal = {Example1 99}
 JSON Unmarshal = {Example2 98}
 Yaml Unmarshal = {Example3 97}
 We can unmarshal into a map instead of a struct: map[key:value]
 we can also use decoders/encoders to work with streams: 
      map[key:value key2:value2]

```

1.  如果你复制或自己编写了测试，请向上移动一个目录并运行 `go test`。确保所有测试通过。

# 它是如何工作的...

这个配方给出了使用 TOML、YAML 和 JSON 解析器的示例，既可以向 go 结构体写入原始数据，也可以从其中读取数据并将其转换为相应的格式。就像在 第一章 的配方中一样，我们在 *I/O 和文件系统* 中看到，快速在 `[]byte`、`string`、`bytes.Buffer` 和其他 I/O 接口之间切换是多么常见。

`encoding/json` 包在提供编码、序列化和其他用于处理 JSON 格式的方法方面是最全面的。我们通过 `ToFormat` 函数将这些抽象出来，因此将多个方法附加到单个结构体上以使用这些类型，这将变得非常简单，该结构体可以快速转换为或从这些类型之一。

本节还使用了结构体标签及其用法。上一章也使用了这些，它们是 Go 中向包和库提供有关如何处理结构体中包含的数据提示的常见方式。

# 使用 Unix 管道

Unix 管道在将一个程序输出传递给另一个程序的输入时很有用。例如，看看这个：

```go
$ echo "test case" | wc -l
 1

```

在 Go 应用程序中，管道的左侧可以使用 `os.Stdin` 读取，并像文件描述符一样工作。为了演示这一点，本配方将从一个管道的左侧获取输入，并返回单词及其出现次数的列表。这些单词将在空白处进行标记化。

# 准备就绪

请参考 *准备就绪* 部分的步骤，在 *使用命令行标志* 配方中*.*。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建一个名为 `chapter2/pipes` 的新目录，并导航到该目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter2/pipes`](https://github.com/agtorre/go-cookbook/tree/master/chapter2/pipes) 复制测试，或者将此作为练习编写一些你自己的代码！

1.  创建一个名为 `pipes.go` 的文件，内容如下：

```go
        package main

        import (
            "bufio"
            "fmt"
            "io"
            "os"
        )

        // WordCount takes a file and returns a map
        // with each word as a key and it's number of
        // appearances as a value
        func WordCount(f io.Reader) map[string]int {
            result := make(map[string]int)

            // make a scanner to work on the file
            // io.Reader interface
            scanner := bufio.NewScanner(f)
            scanner.Split(bufio.ScanWords)

            for scanner.Scan() {
                result[scanner.Text()]++
            }

            if err := scanner.Err(); err != nil {
                fmt.Fprintln(os.Stderr, "reading input:", err)
            }

            return result
        }

        func main() {
            fmt.Printf("string: number_of_occurrences\n\n")
            for key, value := range WordCount(os.Stdin) {
                fmt.Printf("%s: %d\n", key, value)
            }
        }

```

1.  运行 `echo "some string" | go run pipes.go`。

1.  你也可以运行这些：

```go
 go build echo "some string" | ./pipes

```

你应该看到以下输出：

```go
 $ echo "test case" | go run pipes.go
 string: number_of_occurrences

 test: 1
 case: 1

 $ echo "test case test" | go run pipes.go
 string: number_of_occurrences

 test: 2
 case: 1

```

1.  如果你复制或编写了自己的测试，请进入上一级目录并运行 `go test`。确保所有测试都通过。

# 作用原理...

在 Go 中处理管道相当简单，特别是如果你熟悉处理文件。例如，你可以使用 第一章 中的管道配方，*I/O 和文件系统*，来创建一个 **tee** 应用程序 ([`en.wikipedia.org/wiki/Tee_(command)`](https://en.wikipedia.org/wiki/Tee_(command)))，其中所有通过管道输入的内容都会立即写入 stdout 和文件。

此配方使用扫描器来对 `os.Stdin` 文件对象的 `io.Reader` 接口进行标记化。你可以看到在完成所有读取后，你必须检查错误。

# 捕获和处理信号

信号是用户或操作系统终止运行中的应用程序的有用方式。有时，以比默认行为更优雅的方式处理这些信号是有意义的。Go 提供了一种捕获和处理信号的方法。在本配方中，我们将通过使用 Go 协程的信号处理来探索信号的处理。

# 准备就绪

请参考 *准备就绪* 部分的步骤，在 *使用命令行标志* 配方中。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建一个名为 `chapter2/signals` 的新目录，并导航到该目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter2/signals`](https://github.com/agtorre/go-cookbook/tree/master/chapter2/signals) 复制测试，或者将此作为练习编写一些你自己的代码！

1.  创建一个名为 `signals.go` 的文件，并包含以下内容：

```go
        package main

        import (
            "fmt"
            "os"
            "os/signal"
            "syscall"
        )

        // CatchSig sets up a listener for
        // SIGINT interrupts
        func CatchSig(ch chan os.Signal, done chan bool) {
            // block on waiting for a signal
            sig := <-ch
            // print it when it's received
            fmt.Println("nsig received:", sig)

            // we can set up handlers for all types of
            // sigs here
            switch sig {
            case syscall.SIGINT:
                fmt.Println("handling a SIGINT now!")
            case syscall.SIGTERM:
                fmt.Println("handling a SIGTERM in an entirely 
                different way!")
            default:
                fmt.Println("unexpected signal received")
            }

            // terminate
            done <- true
        }

        func main() {
            // initialize our channels
            signals := make(chan os.Signal)
            done := make(chan bool)

            // hook them up to the signals lib
            signal.Notify(signals, syscall.SIGINT, syscall.SIGTERM)

            // if a signal is caught by this go routine
            // it will write to done
            go CatchSig(signals, done)

            fmt.Println("Press ctrl-c to terminate...")
            // the program blogs until someone writes to done
            <-done
            fmt.Println("Done!")

        }

```

1.  运行以下命令：

```go
 go build ./signals

```

1.  尝试运行并按下 *Ctrl* + *C*，你应该会看到以下内容：

```go
 $./signals
 Press ctrl-c to terminate...
 ^C
 sig received: interrupt
 handling a SIGINT now!
 Done!

```

1.  再次尝试运行它，并在另一个终端中确定 PID，然后终止应用程序：

```go
 $./signals
 Press ctrl-c to terminate...

 # in a separate terminal
 $ ps -ef | grep signals
 501 30777 26360 0 5:00PM ttys000 0:00.00 ./signals

 $ kill -SIGTERM 30777

 # in the original terminal

 sig received: terminated
 handling a SIGTERM in an entirely different way!
 Done!

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方使用了通道，这在第九章中进行了更广泛的介绍，*并行和并发*。这是因为信号。`Notify` 函数需要一个通道来发送信号通知。`kill` 命令是测试向应用程序传递信号的好方法。我们通过信号注册我们关心的信号类型。`Notify` 函数。然后，我们在 Go 协程中设置一个函数来处理传递给该函数的通道上的任何活动。一旦我们收到信号，我们可以按自己的意愿处理它。我们可以终止应用程序，响应消息，并为不同的信号有不同的行为。

我们还使用一个 `done` 通道来阻塞应用程序，直到接收到信号，否则程序将立即终止。这对于长时间运行的应用程序，如 Web 应用程序来说是不必要的。创建适当的信号处理例程来执行清理工作非常有用，尤其是在有大量 Go 协程持有大量状态的应用程序中。一个优雅关闭的实用示例可能是允许当前处理程序完成它们的 HTTP 请求，而不会在途中终止它们。

# 一个 ANSI 着色应用程序

在你想要着色的文本部分之前和之后，由各种代码处理 ANSI 终端应用程序的着色。本章将探讨一个基本的着色机制，用于着色文本为红色或普通。对于完整的应用程序，请查看[`github.com/agtorre/gocolorize`](https://github.com/agtorre/gocolorize)，它支持更多颜色和文本类型，并且还实现了`fmt.Formatter`接口，以便于打印。

# 准备工作

请参考 *准备工作* 部分的步骤，在 *使用命令行标志* 配方中*.*。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter2/ansicolor` 目录。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter2/ansicolor`](https://github.com/agtorre/go-cookbook/tree/master/chapter2/ansicolor)复制测试，或者将其作为练习编写一些自己的代码！

1.  创建一个名为 `color.go` 的文件，并包含以下内容：

```go
        package ansicolor

        import "fmt"

        //Color of text
        type Color int

        const (
            // ColorNone is default
            ColorNone = iota
            // Red colored text
            Red
            // Green colored text
            Green
            // Yellow colored text
            Yellow
            // Blue colored text
            Blue
            // Magenta colored text
            Magenta
            // Cyan colored text
            Cyan
            // White colored text
            White
            // Black colored text
            Black Color = -1
        )

        // ColorText holds a string and its color
        type ColorText struct {
            TextColor Color
            Text      string
        }

        func (r *ColorText) String() string {
            if r.TextColor == ColorNone {
                return r.Text
            }

            value := 30
            if r.TextColor != Black {
                value += int(r.TextColor)
            }
            return fmt.Sprintf("33[0;%dm%s33[0m", value, r.Text)
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个包含以下内容的 `main.go` 文件，并确保你修改 `ansicolor` 导入以使用步骤 1 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter2/ansicolor"
        )

        func main() {
            r := ansicolor.ColorText{
                TextColor: ansicolor.Red,
                Text:      "I'm red!",
            }

            fmt.Println(r.String())

            r.TextColor = ansicolor.Green
            r.Text = "Now I'm green!"

            fmt.Println(r.String())

            r.TextColor = ansicolor.ColorNone
            r.Text = "Back to normal..."

            fmt.Println(r.String())
        }

```

1.  运行 `go run main.go`。

1.  你还可以运行以下命令：

```go
 go build ./example

```

1.  如果您的终端支持 ANSI 着色格式，您应该看到以下带有颜色的输出：

```go
 $ go run main.go
 I'm red!
 Now I'm green!
 Back to normal...

```

1.  如果您复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

此应用程序使用结构体来维护彩色文本的状态。在这种情况下，它存储文本的颜色和文本的值。当您调用`String()`方法时，将渲染最终的字符串，该方法将返回彩色文本或纯文本，具体取决于结构体中存储的值。默认情况下，文本将是纯文本。
