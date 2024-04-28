# 命令行工具

命令行应用程序是处理用户输入和输出的最简单方式之一。本章将重点介绍基于命令行的交互，如命令行参数、配置和环境变量。最后，我们将介绍一个用于在 Unix 和 Bash for Windows 中着色文本输出的库。

通过本章的配方，您应该能够处理预期和意外的用户输入。*捕获和处理信号*配方是一个例子，说明用户可能向您的应用程序发送意外信号的情况，而管道配方是相对于标志或命令行参数来说获取用户输入的一个很好的替代方法。

ANSI 颜色配方有望提供一些清理输出给用户的示例。例如，在日志记录中，能够根据其用途着色文本有时可以使大块文本变得更清晰。

在本章中，我们将介绍以下配方：

+   使用命令行标志

+   使用命令行参数

+   读取和设置环境变量

+   使用 TOML、YAML 和 JSON 进行配置

+   使用 Unix 管道

+   捕获和处理信号

+   一个 ANSI 着色应用程序

# 技术要求

为了继续本章中的所有配方，请根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.12.6 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  打开终端或控制台应用程序，并创建并导航到项目目录，例如`~/projects/go-programming-cookbook`。我们所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，并从该目录中工作，而不是手动输入示例： 

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 使用命令行标志

`flag`包使得向 Go 应用程序添加命令行标志参数变得简单。它有一些缺点——您往往需要重复大量的代码来添加标志的简写版本，并且它们按照帮助提示的字母顺序排列。有许多第三方库试图解决这些缺点，但本章将重点介绍标准库版本，而不是这些库。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter2/flags`的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/flags 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/flags    
```

1.  从`~/projects/go-programming-cookbook-original/chapter2/flags`复制测试，或者利用这个机会编写一些您自己的代码！

1.  创建一个名为`flags.go`的文件，内容如下：

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

1.  创建一个名为`custom.go`的文件，内容如下：

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

1.  运行以下命令：

```go
$ go mod tidy
```

1.  创建一个名为`main.go`的文件，内容如下：

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
$ go build $ ./flags -h
```

1.  尝试这些和其他一些参数；您应该看到以下输出：

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

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`，确保所有测试都通过。

# 它是如何工作的...

该配方试图演示`flag`包的大多数常见用法。它显示自定义变量类型、各种内置变量、简写标志，并将所有标志写入一个公共结构。这是第一个需要主函数的配方，因为应该从主函数中调用 flag 的主要用法（`flag.Parse()`）。因此，正常的示例目录被省略了。

该应用程序的示例用法显示，您会自动得到`-h`以获取包含的标志列表。还有一些需要注意的是，布尔标志是在没有参数的情况下调用的，而标志的顺序并不重要。

`flag`包是一种快速构建命令行应用程序输入的方式，并提供了一种灵活的方式来指定用户输入，比如设置日志级别或应用程序的冗长程度。在*使用命令行参数*示例中，我们将探讨标志集并使用参数在它们之间切换。

# 使用命令行参数

上一个示例中的标志是一种命令行参数。本章将扩展这些参数的其他用途，通过构建支持嵌套子命令的命令来演示标志集，并使用传递给应用程序的位置参数。

与上一个示例一样，这个示例需要一个主函数来运行。有许多第三方包处理复杂的嵌套参数和标志，但我们将探讨如何仅使用标准库来实现这一点。

# 如何操作...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter2/cmdargs`的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/cmdargs 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/cmdargs   
```

1.  从`~/projects/go-programming-cookbook-original/chapter2/cmdargs`中复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`cmdargs.go`的文件，内容如下：

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

1.  创建一个名为`main.go`的文件，内容如下：

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

  if err := menu.Parse(os.Args[1:]); err != nil {
    fmt.Printf("Error parsing params %s, error: %v", os.Args[1:], err)
    return
  }

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
        if err := f.Parse(os.Args[3:]); err != nil {
          fmt.Fprintf(os.Stderr, "Error parsing params %s, error: %v", os.Args[3:], err)
          return
        }

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

1.  运行`go build`。

1.  运行以下命令，并尝试一些其他参数的组合：

```go
$ ./cmdargs -h 
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

1.  如果你复制或编写了自己的测试，返回上一级目录并运行`go test`，确保所有测试都通过。

# 它是如何工作的...

标志集可用于设置独立的预期参数列表、使用字符串等。开发人员需要对许多参数进行验证，将正确的子集参数解析到命令中，并定义使用字符串。这可能容易出错，并需要大量迭代才能完全正确。

`flag`包使解析参数变得更加容易，并包括方便的方法来获取标志的数量、参数等。这个示例演示了使用参数构建复杂命令行应用程序的基本方法，包括包级配置、必需的位置参数、多级命令使用，以及如何将这些内容拆分成多个文件或包（如果需要）。

# 读取和设置环境变量

环境变量是另一种可以将状态传递到应用程序中的方式，除了从文件中读取数据或通过命令行显式传递数据。这个示例将探讨一些非常基本的获取和设置环境变量的方法，然后使用非常有用的第三方库`envconfig`（[`github.com/kelseyhightower/envconfig`](https://github.com/kelseyhightower/envconfig)）。

我们将构建一个应用程序，可以通过 JSON 或环境变量读取`config`文件。下一个示例将探讨替代格式，包括 TOML 和 YAML。

# 如何操作...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter2/envvar`的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/envvar 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/envvar   
```

1.  复制`~/projects/go-programming-cookbook-original/chapter2/envvar`中的测试，或者利用这个机会编写一些自己的代码！

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

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "bytes"
            "fmt"
            "io/ioutil"
            "os"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter2/envvar"
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

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶级示例目录中。

1.  如果你复制或编写了自己的测试，返回上一级目录并运行`go test`，确保所有测试都通过。

# 它是如何工作的...

使用`os`包读取和写入环境变量非常简单。这个配方使用的`envconfig`第三方库是一种聪明的方式，可以捕获环境变量并使用`struct`标签指定某些要求。

`LoadConfig`函数是一种灵活的方式，可以从各种来源获取配置信息，而不需要太多的开销或太多额外的依赖。将主要的`config`转换为除 JSON 以外的其他格式，或者始终使用环境变量也很简单。

还要注意错误的使用。我们在这个配方的代码中包装了错误，这样我们就可以注释错误而不会丢失原始错误的信息。在第四章中会有更多关于这个的细节，*Go 中的错误处理*。

# 使用 TOML、YAML 和 JSON 进行配置

Go 有许多配置格式，通过使用第三方库，支持。其中三种最流行的数据格式是 TOML、YAML 和 JSON。Go 可以直接支持 JSON，其他格式有关于如何为这些格式编组/解组或编码/解码数据的线索。这些格式除了配置之外还有许多好处，但本章主要关注将 Go 结构转换为配置结构的过程。这个配方将探讨使用这些格式进行基本输入和输出。

这些格式还提供了一个接口，通过这个接口，Go 和其他语言编写的应用程序可以共享相同的配置。还有许多处理这些格式并简化与它们一起工作的工具。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter2/confformat`的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/confformat 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/confformat   
```

1.  从`~/projects/go-programming-cookbook-original/chapter2/confformat`复制测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`toml.go`的文件，内容如下：

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

1.  创建一个名为`yaml.go`的文件，内容如下：

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

1.  创建一个名为`json.go`的文件，内容如下：

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

1.  创建一个名为`marshal.go`的文件，内容如下：

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

1.  创建一个名为`unmarshal.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录并导航到该目录。

1.  创建一个`main.go`文件，内容如下：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter2/confformat"

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

1.  运行`go run main.go`。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

1.  你应该看到以下输出：

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

1.  `go.mod`文件可能会被更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果你复制或编写了自己的测试，返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方为我们提供了如何使用 TOML、YAML 和 JSON 解析器的示例，用于将原始数据写入 go 结构并从中读取数据并转换为相应的格式。就像第一章中的配方，*I/O 和文件系统*，我们看到了在`[]byte`、`string`、`bytes.Buffer`和其他 I/O 接口之间快速切换是多么常见。

`encoding/json`包在提供编码、编组和其他方法以处理 JSON 格式方面是最全面的。我们通过`ToFormat`函数将这些抽象出来，非常简单地可以附加多个类似的方法，这样我们就可以使用一个结构快速地转换成任何这些类型，或者从这些类型转换出来。

这个配方还涉及结构标签及其用法。上一章也使用了这些，它们是一种常见的方式，用于向包和库提供关于如何处理结构中包含的数据的提示。

# 使用 Unix 管道进行工作

当我们将一个程序的输出传递给另一个程序的输入时，Unix 管道非常有用。例如，看一下以下代码：

```go
$ echo "test case" | wc -l
 1
```

在 Go 应用程序中，管道的左侧可以使用`os.Stdin`进行读取，它的作用类似于文件描述符。为了演示这一点，本教程将接受管道左侧的输入，并返回一个单词列表及其出现次数。这些单词将在空格上进行标记化。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter2/pipes`的新目录。

1.  导航到此目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/pipes
```

您应该会看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/pipes   
```

1.  从`~/projects/go-programming-cookbook-original/chapter2/pipes`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`pipes.go`的文件，其中包含以下内容：

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

1.  运行`echo "some string" | go run pipes.go`。

1.  您还可以运行以下命令：

```go
$ go build echo "some string" | ./pipes
```

您应该会看到以下输出：

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

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

在 Go 中使用管道非常简单，特别是如果您熟悉使用文件。例如，您可以使用第一章中的管道教程，*I/O 和文件系统*，创建一个**tee**应用程序（[`en.wikipedia.org/wiki/Tee_(command)`](https://en.wikipedia.org/wiki/Tee_(command)）其中所有输入的内容都立即写入到`stdout`和文件中。

本教程使用扫描程序来标记`os.Stdin`文件对象的`io.Reader`接口。您可以看到在完成所有读取后必须检查错误。

# 捕获和处理信号

信号是用户或操作系统终止正在运行的应用程序的有用方式。有时，以比默认行为更优雅的方式处理这些信号是有意义的。Go 提供了一种机制来捕获和处理信号。在本教程中，我们将通过使用处理 Go 例程的信号来探讨信号的处理。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter2/signals`的新目录。

1.  导航到此目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/signals 
```

您应该会看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/signals   
```

1.  从`~/projects/go-programming-cookbook-original/chapter2/signals`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`signals.go`的文件，其中包含以下内容：

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
            // the program blocks until someone writes to done
            <-done
            fmt.Println("Done!")

        }
```

1.  运行以下命令：

```go
$ go build $ ./signals
```

1.  尝试运行代码，然后按*Ctrl* + *C*。您应该会看到以下内容：

```go
$./signals
Press ctrl-c to terminate...
^C
sig received: interrupt
handling a SIGINT now!
Done!
```

1.  尝试再次运行它。然后，从另一个终端确定 PID 并终止应用程序：

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

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

本教程使用了通道，这在第九章“并行和并发”中有更详细的介绍。`signal.Notify`函数需要一个通道来发送信号通知，还需要我们关心的信号类型。然后，我们在 Go 例程中设置一个函数来处理我们传递给该函数的通道上的任何活动。一旦我们收到信号，我们可以以任何我们想要的方式处理它。我们可以终止应用程序，回复消息，并对不同的信号有不同的行为。`kill`命令是测试向应用程序传递信号的好方法。

我们还使用一个 `done` 通道来阻止应用程序在接收到信号之前终止。否则，程序会立即终止。对于长时间运行的应用程序（如 Web 应用程序），这是不必要的。创建适当的信号处理例程来执行清理工作可能非常有用，特别是在具有大量 Go 协程并持有大量状态的应用程序中。一个优雅关闭的实际例子可能是允许当前处理程序完成其 HTTP 请求而不会在中途终止它们。

# 一个 ANSI 着色应用程序

对 ANSI 终端应用程序进行着色是通过一系列代码来处理的，在你想要着色的文本之前和之后。本教程将探讨一种基本的着色机制，可以将文本着色为红色或普通色。要了解完整的应用程序，请查看 [`github.com/agtorre/gocolorize`](https://github.com/agtorre/gocolorize)，它支持更多的颜色和文本类型，并且还实现了 `fmt.Formatter` 接口以便于打印。

# 如何做...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为 `~/projects/go-programming-cookbook/chapter2/ansicolor` 的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/ansicolor 
```

您应该会看到一个名为 `go.mod` 的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter2/ansicolor   
```

1.  从 `~/projects/go-programming-cookbook-original/chapter2/ansicolor` 复制测试，或者利用这个机会编写一些您自己的代码！

1.  创建一个名为 `color.go` 的文件，其中包含以下内容：

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

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter2/ansicolor"
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

1.  您也可以运行以下命令：

```go
$ go build $ ./example
```

1.  如果您的终端支持 ANSI 着色格式，您应该会看到以下输出的文本被着色：

```go
$ go run main.go
I'm red!
Now I'm green!
Back to normal...
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行 `go test`。确保所有测试都通过。

# 工作原理...

该应用程序利用一个结构来维护着色文本的状态。在这种情况下，它存储文本的颜色和值。当您调用 `String()` 方法时，最终的字符串将被渲染，根据结构中存储的值，它将返回着色文本或普通文本。默认情况下，文本将是普通的。
