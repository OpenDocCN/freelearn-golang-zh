# 第三章：数据转换和组合

在本章中，将介绍以下食谱：

+   转换数据类型和接口转换

+   使用 math 和 math/big 操作数值数据类型

+   货币转换和 float64 考虑

+   使用指针和 SQL NullTypes 进行编码和解码

+   编码和解码 Go 数据

+   Go 中的结构体标签和基本反射

+   通过闭包实现集合

# 简介

理解 Go 的类型系统是 Go 开发所有级别的关键步骤。本章将展示在数据类型之间转换、处理非常大的数字、处理货币、编码和解码的类型（包括 base64 和 gob）以及使用闭包创建自定义集合的示例。

# 转换数据类型和接口转换

Go 在数据之间的转换通常非常灵活。一个类型可以继承另一个类型，如下所示：

```go
type A int

```

然后，我们可以始终将类型转换回我们继承的类型，如下所示：

```go
var a A = 1
fmt.Println(int(a))

```

还有一些方便的函数用于在数字之间进行转换（使用类型转换），在字符串和其他类型之间使用 `fmt.Sprint` 和 `strconv` 进行转换，以及使用反射在接口和类型之间进行转换。这个食谱将探索一些将在整本书中使用的这些基本转换。

# 准备工作

根据以下步骤配置你的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到你的操作系统上，并配置你的 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序，导航到你的 `GOPATH/src` 并创建一个项目目录，例如 `$GOPATH/src/github.com/yourusername/customrepo`。

所有代码都将从这个目录运行和修改。

1.  可选地，使用 `go get github.com/agtorre/go-cookbook/` 命令安装代码的最新测试版本。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建并导航到 `chapter3/dataconv` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter3/dataconv`](https://github.com/agtorre/go-cookbook/tree/master/chapter3/dataconv) 复制测试或使用此作为练习编写一些你自己的代码。

1.  创建一个名为 `dataconv.go` 的文件，内容如下：

```go
        package dataconv

        import "fmt"

        // ShowConv demonstrates some type conversion
        func ShowConv() {
            // int
            var a = 24

            // float 64
            var b = 2.0

            // convert the int to a float64 for this calculation
            c := float64(a) * b
            fmt.Println(c)

            // fmt.Sprintf is a good way to convert to strings
            precision := fmt.Sprintf("%.2f", b)

            // print the value and the type
            fmt.Printf("%s - %T\n", precision, precision)
        }

```

1.  创建一个名为 `strconv.go` 的文件，内容如下：

```go
        package dataconv

        import (
            "fmt"
            "strconv"
        )

        // Strconv demonstrates some strconv
        // functions
        func Strconv() error {
            //strconv is a good way to convert to and from strings
            s := "1234"
            // we can specify the base (10) and precision
            // 64 bit
            res, err := strconv.ParseInt(s, 10, 64)
            if err != nil {
                return err
          }

          fmt.Println(res)

          // lets try hex
          res, err = strconv.ParseInt("FF", 16, 64)
          if err != nil {
              return err
          }

          fmt.Println(res)

          // we can do other useful things like:
          val, err := strconv.ParseBool("true")
          if err != nil {
              return err
          }

          fmt.Println(val)

          return nil
        }

```

1.  创建一个名为 `interfaces.go` 的文件，内容如下：

```go
        package dataconv

        import "fmt"

        // CheckType will print based on the
        // interface type
        func CheckType(s interface{}) {
            switch s.(type) {
            case string:
                fmt.Println("It's a string!")
            case int:
                fmt.Println("It's an int!")
            default:
                fmt.Println("not sure what it is...")
            }
        }

        // Interfaces demonstrates casting
        // from anonymous interfaces to types
        func Interfaces() {
            CheckType("test")
            CheckType(1)
            CheckType(false)

            var i interface{}
            i = "test"

            // manually check an interface
            if val, ok := i.(string); ok {
                fmt.Println("val is", val)
            }

            // this one should fail
            if _, ok := i.(int); !ok {
                fmt.Println("uh oh! glad we handled this")
            }
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，内容如下。确保将 `dataconv` 导入修改为使用步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter3/dataconv"

        func main() {
            dataconv.ShowConv()
            if err := dataconv.Strconv(); err != nil {
                panic(err)
            }
            dataconv.Interfaces()
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
      48
 2.00 - string
 1234
 255
 true
 It's a string!
 It's an int!
 not sure what it is...
 val is test
 uh oh! glad we handled this

```

1.  如果你复制或编写了自己的测试，向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

本食谱通过使用新的类型包装它们、使用 `strconv` 包以及使用接口反射来演示类型之间的转换。这些方法允许 Go 开发者快速在多种抽象的 Go 类型之间进行转换。这些方法在编译过程中都会揭示错误，但反射可能更加复杂。如果你错误地反射到一个不受支持的类型，你将导致程序崩溃。根据类型进行切换是一种通用方法，本食谱中也进行了演示。

对于像 `math` 这样的包，转换变得很重要，因为它们仅使用 float64 进行操作。

# 使用 math 和 math/big 进行数值数据类型的操作

`math` 和 `math/big` 包专注于向 Go 语言暴露更复杂的数学运算，例如 `Pow`、`Sqrt` 和 `Cos`。`math` 包本身主要在 float64 上操作，除非函数有其他说明。`math/big` 包用于表示超过 64 位值的大数。本食谱将展示 `math` 包的一些基本用法，并演示 `math/big` 在斐波那契数列中的应用。

# 准备就绪

参考在 *转换数据类型和接口转换* 食谱的 *准备就绪* 部分中给出的步骤。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter3/math` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter3/math`](https://github.com/agtorre/go-cookbook/tree/master/chapter3/math) 复制测试或将其作为练习编写一些自己的代码。

1.  创建一个名为 `math.go` 的文件，内容如下：

```go
        package math

        import (
         "fmt"
         "math"
        )

        // Examples demonstrates some of the functions
        // in the math package
        func Examples() {
            //sqrt Examples
            i := 25

            // i is an int, so convert
            result := math.Sqrt(float64(i))

            // sqrt of 25 == 5
            fmt.Println(result)

            // ceil rounds up
            result = math.Ceil(9.5)
            fmt.Println(result)

            // floor rounds down
            result = math.Floor(9.5)
            fmt.Println(result)

            // math also stores some consts:
            fmt.Println("Pi:", math.Pi, "E:", math.E)
        }

```

1.  创建一个名为 `fib.go` 的文件，内容如下：

```go
        package math

        import "math/big"

        // global to memoize fib
        var memoize map[int]*big.Int

        func init() {
            // initialize the map
            memoize = make(map[int]*big.Int)
        }

        // Fib prints the nth digit of the fibonacci sequence
        // it will return 1 for anything < 0 as well...
        // it's calculated recursively and use big.Int since
        // int64 will quickly overflow
        func Fib(n int) *big.Int {
            if n < 0 {
                return nil
            }

            // base case
            if n < 2 {
                memoize[n] = big.NewInt(1)
            }

            // check if we stored it before
            // if so return with no calculation
            if val, ok := memoize[n]; ok {
                return val
            }

            // initialize map then add previous 2 fib values
            memoize[n] = big.NewInt(0)
            memoize[n].Add(memoize[n], Fib(n-1))
            memoize[n].Add(memoize[n], Fib(n-2))

            // return result
            return memoize[n]
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，内容如下；确保将 `math` 导入修改为步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter3/math"
        )

        func main() {
            math.Examples()

            for i := 0; i < 10; i++ {
                fmt.Printf("%v ", math.Fib(i))
            }
            fmt.Println()
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 5
 10
 9
 Pi: 3.141592653589793 E: 2.718281828459045
 1 1 2 3 5 8 13 21 34 55

```

1.  如果你复制或编写了自己的测试，向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

`math` 包使得在 Go 中执行复杂的数学运算成为可能。本食谱应与该包结合使用，以执行复杂的浮点运算和根据需要转换类型。值得注意的是，即使使用 float64，某些浮点数仍然可能存在舍入误差，以下食谱展示了处理这些误差的一些技术。

`math/big` 部分展示了递归的斐波那契数列。如果你修改 `main.go` 以超过 10 的循环，如果你使用 `int64` 而不是 `big.Int`，你将很快溢出。此包还具有将大类型转换为其他类型的一些辅助方法。

# 货币转换和 float64 考虑事项

处理货币总是一个棘手的过程。可能会诱使你将金钱表示为 float64，但在进行计算时，这可能会导致一些相当棘手（且错误）的舍入误差。因此，最好将金钱视为分，并以 Int64 存储它。

当从表单、命令行或其他来源收集用户输入时，金钱通常以美元形式表示。因此，最好将其视为字符串，并直接将该字符串转换为便士，而不进行浮点数转换。这个食谱将展示如何将货币的字符串表示转换为 int64（便士）以及再次转换回来。

# 准备就绪

参考转换数据类型和接口转换食谱*中*的 *准备就绪* 部分的步骤。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建并导航到 `chapter3/currency` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter3/currency`](https://github.com/agtorre/go-cookbook/tree/master/chapter3/currency) 复制测试或将其作为练习编写一些自己的测试。

1.  创建一个名为 `dollars.go` 的文件，并包含以下内容：

```go
        package currency

        import (
            "errors"
            "strconv"
            "strings"
        )

        // ConvertStringDollarsToPennies takes a dollar amount
        // as a string, i.e. 1.00, 55.12 etc and converts it
        // into an int64
        func ConvertStringDollarsToPennies(amount string) (int64, 
        error) {
            // check if amount can convert to a valid float
            _, err := strconv.ParseFloat(amount, 64)
            if err != nil {
                return 0, err
            }

            // split the value on "."
            groups := strings.Split(amount, ".")

            // if there is no . result will still be
            // captured here
            result := groups[0]

            // base string
            r := ""

            // handle the data after the "."
            if len(groups) == 2 {
                if len(groups[1]) != 2 {
                    return 0, errors.New("invalid cents")
                }
                r = groups[1]
                if len(r) > 2 {
                    r = r[:2]
                }
            }

            // pad with 0, this will be
            // 2 0's if there was no .
            for len(r) < 2 {
                r += "0"
            }

            result += r

            // convert it to an int
            return strconv.ParseInt(result, 10, 64)
        }

```

1.  创建一个名为 `pennies.go` 的文件，并包含以下内容：

```go
        package currency

        import (
            "strconv"
        )

        // ConvertPenniesToDollarString takes a penny amount as 
        // an int64 and returns a dollar string representation
        func ConvertPenniesToDollarString(amount int64) string {
            // parse the pennies as a base 10 int
            result := strconv.FormatInt(amount, 10)

            // check if negative, will set it back later
            negative := false
            if result[0] == '-' {
                result = result[1:]
                negative = true
            }

            // left pad with 0 if we're passed in value < 100
            for len(result) < 3 {
                result = "0" + result
            }
            length := len(result)

            // add in the decimal
            result = result[0:length-2] + "." + result[length-2:]

            // from the negative we stored earlier!
            if negative {
                result = "-" + result
            }

            return result
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；请确保将 `currency` 导入修改为你在第 2 步中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter3/currency"
        )

        func main() {
            // start with our user input
            // of fifteen dollars and 93 cents
            userInput := "15.93"

            pennies, err := 
            currency.ConvertStringDollarsToPennies(userInput)
            if err != nil {
                panic(err)
            }

            fmt.Printf("User input converted to %d pennies\n", pennies)

            // adding 15 cents
            pennies += 15

            dollars := currency.ConvertPenniesToDollarString(pennies)

            fmt.Printf("Added 15 cents, new values is %s dollars\n", 
            dollars)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下操作：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 User input converted to 1593 pennies
 Added 15 cents, new values is 16.08 dollars

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 如何工作...

这个食谱使用了 `strconv` 和 `strings` 包来在美元字符串格式和 int64 的便士之间转换货币。它甚至不需要将值转换为 float64，除了作为验证。

`strconv.ParseInt` 和 `strconv.FormatInt` 函数在将 int64 和字符串相互转换时非常有用。我们还利用了 Go 字符串可以轻松地按需追加和切片的事实。

# 使用指针和 SQL NullTypes 进行编码和解码

当你在 Go 中编码或解码到对象时，未显式设置的类型将使用其默认值。例如，字符串将默认为空字符串 ""，整数将默认为 `0`。通常情况下，这是可以的，除非 `0` 对于你的 API 或服务来说意味着消耗用户输入或返回输入。

此外，如果你使用如 `json omitempty` 这样的结构标签，即使它们是有效的，0 值也会被忽略。另一个例子是 SQL 返回的 `Null`。对于 `Int`，哪个值最能代表 `Null`？这个食谱将探讨一些 Go 开发者处理这个问题的方法。

# 准备就绪

参考转换数据类型和接口转换食谱*中*的 *准备就绪* 部分的步骤。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter3/nulls` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter3/nulls`](https://github.com/agtorre/go-cookbook/tree/master/chapter3/nulls) 复制测试代码或使用此作为练习编写一些你自己的代码。

1.  创建一个名为 `base.go` 的文件，并包含以下内容：

```go
        package nulls

        import (
            "encoding/json"
            "fmt"
        )

        // json that has name but not age
        const (
            jsonBlob = `{"name": "Aaron"}`
            fulljsonBlob = `{"name":"Aaron", "age":0}`
        )

        // Example is a basic struct with age
        // and name fields
        type Example struct {
            Age int `json:"age,omitempty"`
            Name string `json:"name"`
        }

        // BaseEncoding shows encoding and
        // decoding with normal types
        func BaseEncoding() error {
            e := Example{}

            // note that no age = 0 age
            if err := json.Unmarshal([]byte(jsonBlob), &e); err != nil 
            {
                return err
            }
            fmt.Printf("Regular Unmarshal, no age: %+v\n", e)

            value, err := json.Marshal(&e)
            if err != nil {
                return err
            }
            fmt.Println("Regular Marshal, with no age:", string(value))

            if err := json.Unmarshal([]byte(fulljsonBlob), &e);
            err != nil {
                return err
            }
            fmt.Printf("Regular Unmarshal, with age = 0: %+v\n", e)

            value, err = json.Marshal(&e)
            if err != nil {
                return err
            }
            fmt.Println("Regular Marshal, with age = 0:", 
            string(value))

            return nil
        }

```

1.  创建一个名为 `pointer.go` 的文件，并包含以下内容：

```go
        package nulls

        import (
            "encoding/json"
            "fmt"
        )

        // ExamplePointer is the same, but
        // uses a *Int
        type ExamplePointer struct {
            Age *int `json:"age,omitempty"`
            Name string `json:"name"`
        }

        // PointerEncoding shows methods for
        // dealing with nil/omitted values
        func PointerEncoding() error {

            // note that no age = nil age
            e := ExamplePointer{}
            if err := json.Unmarshal([]byte(jsonBlob), &e); err != nil 
            {
                return err
            }
            fmt.Printf("Pointer Unmarshal, no age: %+v\n", e)

            value, err := json.Marshal(&e)
            if err != nil {
                return err
            }
            fmt.Println("Pointer Marshal, with no age:", string(value))

            if err := json.Unmarshal([]byte(fulljsonBlob), &e);
            err != nil {
                return err
            }
            fmt.Printf("Pointer Unmarshal, with age = 0: %+v\n", e)

            value, err = json.Marshal(&e)
            if err != nil {
                return err
            }
            fmt.Println("Pointer Marshal, with age = 0:",
            string(value))

            return nil
        }

```

1.  创建一个名为 `nullencoding.go` 的文件，并包含以下内容：

```go
        package nulls

        import (
            "database/sql"
            "encoding/json"
            "fmt"
        )

        type nullInt64 sql.NullInt64

        // ExampleNullInt is the same, but
        // uses a sql.NullInt64
        type ExampleNullInt struct {
            Age *nullInt64 `json:"age,omitempty"`
            Name string `json:"name"`
        }

        func (v *nullInt64) MarshalJSON() ([]byte, error) {
            if v.Valid {
                return json.Marshal(v.Int64)
            }
            return json.Marshal(nil)
        }

        func (v *nullInt64) UnmarshalJSON(b []byte) error {
            v.Valid = false
            if b != nil {
                v.Valid = true
                return json.Unmarshal(b, &v.Int64)
            }
            return nil
        }

        // NullEncoding shows an alternative method
        // for dealing with nil/omitted values
        func NullEncoding() error {
            e := ExampleNullInt{}

            // note that no means an invalid value
            if err := json.Unmarshal([]byte(jsonBlob), &e); err != nil 
            {
                return err
            }
            fmt.Printf("nullInt64 Unmarshal, no age: %+v\n", e)

            value, err := json.Marshal(&e)
            if err != nil {
                return err
            }
            fmt.Println("nullInt64 Marshal, with no age:",
            string(value))

            if err := json.Unmarshal([]byte(fulljsonBlob), &e);
            err != nil {
                return err
            }
            fmt.Printf("nullInt64 Unmarshal, with age = 0: %+v\n", e)

            value, err = json.Marshal(&e)
            if err != nil {
                return err
            }
            fmt.Println("nullInt64 Marshal, with age = 0:",
            string(value))

            return nil
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `nulls` 导入路径修改为你在第 2 步中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter3/nulls"
        )

        func main() {
            if err := nulls.BaseEncoding(); err != nil {
                panic(err)
            }
            fmt.Println()

            if err := nulls.PointerEncoding(); err != nil {
                panic(err)
            }
            fmt.Println()

            if err := nulls.NullEncoding(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Regular Unmarshal, no age: {Age:0 Name:Aaron}
 Regular Marshal, with no age: {"name":"Aaron"}
 Regular Unmarshal, with age = 0: {Age:0 Name:Aaron}
 Regular Marshal, with age = 0: {"name":"Aaron"}

 Pointer Unmarshal, no age: {Age:<nil> Name:Aaron}
 Pointer Marshal, with no age: {"name":"Aaron"}
 Pointer Unmarshal, with age = 0: {Age:0xc42000a610 Name:Aaron}
 Pointer Marshal, with age = 0: {"age":0,"name":"Aaron"}

 nullInt64 Unmarshal, no age: {Age:<nil> Name:Aaron}
 nullInt64 Marshal, with no age: {"name":"Aaron"}
 nullInt64 Unmarshal, with age = 0: {Age:0xc42000a750 
      Name:Aaron}
 nullInt64 Marshal, with age = 0: {"age":0,"name":"Aaron"}

```

1.  如果你复制或编写了自己的测试代码，请向上导航一个目录并运行 `go test`。确保所有测试都通过。

# 工作原理...

从值转换为指针是表达序列化和反序列化时的空值的一种快速方法。在设置这些值时可能有点不清楚，因为你不能直接将它们赋值给指针 `-- *a := 1`，但除此之外，这是一种灵活处理它的方法。

本食谱还演示了使用 `sql.NullInt64` 类型的替代方法。这通常与 SQL 一起使用，如果返回的值不是 `Null`，则有效，否则设置为 `Null`。我们添加了 `MarshalJSON` 和 `UnmarshallJSON` 方法，以便此类型可以与 `JSON` 包交互，我们选择使用指针，以便 `omitempty` 能够按预期继续工作。

# Go 数据的编码和解码

Go 除了 JSON、TOML 和 YAML 之外，还提供了一些替代编码类型。这些类型主要用于在 Go 进程之间传输数据，例如使用网络协议和 RPC，或者在某些字符格式受限制的情况下。

本食谱将探讨 gob 格式和 base64 的编码和解码。后续章节将探讨如 GRPC 这样的协议。

# 准备工作

参考在 *转换数据类型和接口转换* 食谱的 *准备工作* 部分中给出的步骤*.*。

# 如何实现...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter3/encoding` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter3/encoding`](https://github.com/agtorre/go-cookbook/tree/master/chapter3/encoding) 复制测试代码或使用此作为练习编写一些你自己的代码。

1.  创建一个名为 `gob.go` 的文件，并包含以下内容：

```go
        package encoding

        import (
            "bytes"
            "encoding/gob"
            "fmt"
        )

        // pos stores the x, y position
        // for Object
        type pos struct {
            X      int
            Y      int
            Object string
        }

        // GobExample demonstrates using
        // the gob package
        func GobExample() error {
            buffer := bytes.Buffer{}

            p := pos{
                X:      10,
                Y:      15,
                Object: "wrench",
            }

            // note that if p was an interface
            // we'd have to call gob.Register first

            e := gob.NewEncoder(&buffer)
            if err := e.Encode(&p); err != nil {
                return err
            }

            // note this is a binary format so it wont print well
            fmt.Println("Gob Encoded valued length: ", 
            len(buffer.Bytes()))

            p2 := pos{}
            d := gob.NewDecoder(&buffer)
            if err := d.Decode(&p2); err != nil {
                return err
            }

            fmt.Println("Gob Decode value: ", p2)

            return nil
        }

```

1.  创建一个名为 `base64.go` 的文件，并包含以下内容：

```go
        package encoding

        import (
            "bytes"
            "encoding/base64"
            "fmt"
            "io/ioutil"
        )

        // Base64Example demonstrates using
        // the base64 package
        func Base64Example() error {
            // base64 is useful for cases where
            // you can't support binary formats
            // it operates on bytes/strings

            // using helper functions and URL encoding
            value := base64.URLEncoding.EncodeToString([]byte("encoding 
            some data!"))
            fmt.Println("With EncodeToString and URLEncoding: ", value)

            // decode the first value
            decoded, err := base64.URLEncoding.DecodeString(value)
            if err != nil {
                return err
            }
            fmt.Println("With DecodeToString and URLEncoding: ", 
            string(decoded))

            return nil
        }

        // Base64ExampleEncoder shows similar examples
        // with encoders/decoders
        func Base64ExampleEncoder() error {
            // using encoder/ decoder
            buffer := bytes.Buffer{}

            // encode into the buffer
            encoder := base64.NewEncoder(base64.StdEncoding, &buffer)

            // be sure to close
            if err := encoder.Close(); err != nil {
                return err
            }
            if _, err := encoder.Write([]byte("encoding some other 
            data")); err != nil {
                return err
            }

            fmt.Println("Using encoder and StdEncoding: ", 
            buffer.String())

            decoder := base64.NewDecoder(base64.StdEncoding, &buffer)
            results, err := ioutil.ReadAll(decoder)
            if err != nil {
                return err
            }

            fmt.Println("Using decoder and StdEncoding: ", 
            string(results))

            return nil
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `encoding` 导入路径修改为你在第 2 步中设置的路径：

```go
        package main

        import (
            "github.com/agtorre/go-cookbook/chapter3/encoding"
        )

        func main() {
            if err := encoding.Base64Example(); err != nil {
                panic(err)
            }

            if err := encoding.Base64ExampleEncoder(); err != nil {
                panic(err)
            }

            if err := encoding.GobExample(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下命令：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 With EncodeToString and URLEncoding: 
      ZW5jb2Rpbmcgc29tZSBkYXRhIQ==
 With DecodeToString and URLEncoding: encoding some data!
 Using encoder and StdEncoding: ZW5jb2Rpbmcgc29tZSBvdGhlciBkYXRh
 Using decoder and StdEncoding: encoding some other data
 Gob Encoded valued length: 57
 Gob Decode value: {10 15 wrench}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试通过。

# 它是如何工作的...

Gob 编码是一种考虑 Go 数据类型构建的流格式。在发送和编码许多连续项时效率最高。对于单个项，其他编码格式，如 JSON，可能更高效且更便携。尽管如此，gob 编码使得对大型复杂结构体的序列化和在单独进程中重建变得简单。尽管这里没有展示，gob 还可以对具有自定义 `MarshalBinary` 和 `UnmarshalBinary` 方法的自定义类型或未导出类型进行操作。

Base64 编码在通过 `GET` 请求通过 URL 进行通信或生成二进制数据的字符串表示编码时很有用。大多数语言都可以支持这种格式并在另一端解包数据。因此，在 JSON 格式不受支持的情况下，通常会将 JSON 负载编码。

# Go 中的结构体标签和基本反射

反射是一个复杂的话题，无法在一个食谱中完全涵盖。然而，反射的一个实际应用是处理结构体标签。在本质上，结构体标签只是键值字符串。你查找键，然后处理值。正如你可以想象的那样，对于像 JSON 序列化和反序列化这样的操作，处理这些值有很多复杂性。

`reflect` 包旨在查询和理解接口对象。它有一些辅助方法来查看结构体的类型、值、结构体标签等。如果你需要像本章开头那样进行基本的接口转换之外的操作，你应该查看这个包。

# 准备工作

参考转换数据类型和接口转换食谱*中*的 *准备工作* 部分的步骤。

# 如何做到...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter3/tags` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter3/tags`](https://github.com/agtorre/go-cookbook/tree/master/chapter3/tags) 复制测试或将其用作练习来编写你自己的代码。

1.  创建一个名为 `serialize.go` 的文件，并包含以下内容：

```go
        package tags

        import "reflect"

        // SerializeStructStrings converts a struct
        // to our custom serialization format
        // it honors serialize struct tags for string types
        func SerializeStructStrings(s interface{}) (string, error) {
            result := ""

            // reflect the interface into
            // a type
            r := reflect.TypeOf(s)
            value := reflect.ValueOf(s)

            // if a pointer to a struct is passed
            // in, handle it appropriately
            if r.Kind() == reflect.Ptr {
                r = r.Elem()
                value = value.Elem()
            }

            // loop over all of the fields
            for i := 0; i < r.NumField(); i++ {
                field := r.Field(i)
                // struct tag found
                key := field.Name
                if serialize, ok := field.Tag.Lookup("serialize"); ok {
                    // ignore "-" otherwise that whole value
                    // becomes the serialize 'key'
                    if serialize == "-" {
                        continue
                    }
                    key = serialize
                }

                switch value.Field(i).Kind() {
                // this recipe only supports strings!
                case reflect.String:
                    result += key + ":" + value.Field(i).String() + ";"
                    // by default skip it
                default:
                    continue
               }
            }
            return result, nil
        }

```

1.  创建一个名为 `deserialize.go` 的文件，并包含以下内容：

```go
        package tags

        import (
            "errors"
            "reflect"
            "strings"
        )

        // DeSerializeStructStrings converts a serialized
        // string using our custom serialization format
        // to a struct
        func DeSerializeStructStrings(s string, res interface{}) error          
        {
            r := reflect.TypeOf(res)

            // we're setting using a pointer so
            // it must always be a pointer passed
            // in
            if r.Kind() != reflect.Ptr {
                return errors.New("res must be a pointer")
            }

            // dereference the pointer
            r = r.Elem()
            value := reflect.ValueOf(res).Elem()

            // split our serialization string into
            // a map
            vals := strings.Split(s, ";")
            valMap := make(map[string]string)
            for _, v := range vals {
                keyval := strings.Split(v, ":")
                if len(keyval) != 2 {
                    continue
                }
                valMap[keyval[0]] = keyval[1]
            }

            // iterate over fields
            for i := 0; i < r.NumField(); i++ {
                field := r.Field(i)

               // check if in the serialize set
               if serialize, ok := field.Tag.Lookup("serialize"); ok {
                   // ignore "-" otherwise that whole value
                   // becomes the serialize 'key'
                   if serialize == "-" {
                       continue
                   }
                   // is it in the map
                   if val, ok := valMap[serialize]; ok {
                       value.Field(i).SetString(val)
                   }
               } else if val, ok := valMap[field.Name]; ok {
                   // is our field name in the map instead?
                   value.Field(i).SetString(val)
               }
            }
            return nil
        }

```

1.  创建一个名为 `tags.go` 的文件，并包含以下内容：

```go
        package tags

        import "fmt"

        // Person is a struct that stores a persons
        // name, city, state, and a misc attribute
        type Person struct {
            Name string `serialize:"name"`
            City string `serialize:"city"`
            State string
             Misc string `serialize:"-"`
             Year int `serialize:"year"`
        }

        // EmptyStruct demonstrates serialize
        // and deserialize for an Empty struct
        // with tags
        func EmptyStruct() error {
            p := Person{}

            res, err := SerializeStructStrings(&p)
            if err != nil {
                return err
            }
            fmt.Printf("Empty struct: %#v\n", p)
            fmt.Println("Serialize Results:", res)

            newP := Person{}
            if err := DeSerializeStructStrings(res, &newP); err != nil 
            {
                return err
            }
            fmt.Printf("Deserialize results: %#v\n", newP)
                return nil
            }

           // FullStruct demonstrates serialize
           // and deserialize for an Full struct
           // with tags
           func FullStruct() error {
               p := Person{
                   Name: "Aaron",
                   City: "Seattle",
                   State: "WA",
                   Misc: "some fact",
                   Year: 2017,
               }
               res, err := SerializeStructStrings(&p)
               if err != nil {
                   return err
               }
               fmt.Printf("Full struct: %#v\n", p)
               fmt.Println("Serialize Results:", res)

               newP := Person{}
               if err := DeSerializeStructStrings(res, &newP);
               err != nil {
                   return err
               }
               fmt.Printf("Deserialize results: %#v\n", newP)
               return nil
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，并包含以下内容；确保将 `tags` 导入修改为你在步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter3/tags"
        )

        func main() {

            if err := tags.EmptyStruct(); err != nil {
                panic(err)
            }

            fmt.Println()

            if err := tags.FullStruct(); err != nil {
                panic(err)
            }
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行这个：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Empty struct: tags.Person{Name:"", City:"", State:"", Misc:"", 
      Year:0}
 Serialize Results: name:;city:;State:;
 Deserialize results: tags.Person{Name:"", City:"", State:"", 
      Misc:"", Year:0}

 Full struct: tags.Person{Name:"Aaron", City:"Seattle", 
      State:"WA", Misc:"some fact", Year:2017}
 Serialize Results: name:Aaron;city:Seattle;State:WA;
 Deserialize results: tags.Person{Name:"Aaron", City:"Seattle",        State:"WA", Misc:"", Year:0}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试通过。

# 它是如何工作的...

这个配方创建了一种字符串序列化格式，它接受一个结构体，并将所有字符串字段序列化为可解析的格式。这个配方不处理某些边缘情况；特别是，字符串不得包含 `:` 或 `;` 字符。以下是其行为摘要：

1.  如果字段是字符串，它将被序列化/反序列化。

1.  如果字段不是字符串，它将被忽略。

1.  如果字段的 struct 标签包含 `serialize"key"`，则键将是返回的序列化/反序列化环境。

1.  重复项不会被处理。

1.  如果没有指定结构体标签，则使用字段名。

1.  如果指定了 `serialize `-`，即使字段是字符串，该字段也会被忽略。

需要注意的其他事项是，反射在非导出值上并不完全起作用。

# 通过闭包实现集合

如果你一直在使用函数式或动态编程语言，你可能会觉得 `for` 循环和 `if` 语句会产生冗长的代码。用于处理列表的函数式结构，如 `map` 和 `filter`，可能很有用，并使代码看起来更易读。然而，在 Go 中，这些类型不在标准库中，并且没有泛型或非常复杂的反射和空接口的使用，很难泛化。这个配方将为你提供一些使用 Go 闭包实现集合的基本示例。

# 准备工作

参考配方 *Converting Data Types and Interface Casting* 中的 *Getting ready* 部分给出的步骤*.*。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建并导航到 `chapter3/collections` 目录。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter3/collections`](https://github.com/agtorre/go-cookbook/tree/master/chapter3/collections) 复制测试或将其用作练习编写一些你自己的代码。

1.  创建一个名为 `collections.go` 的文件，内容如下：

```go
        package collections

        // WorkWith is the struct we'll
        // be implementing collections for
        type WorkWith struct {
            Data    string
            Version int
        }

        // Filter is a functional filter. It takes a list of
        // WorkWith and a WorkWith Function that returns a bool
        // for each "true" element we return it to the resultant
        // list
        func Filter(ws []WorkWith, f func(w WorkWith) bool) []WorkWith 
        {
            // depending on results, smalles size for result
            // is len == 0
            result := make([]WorkWith, 0)
            for _, w := range ws {
                if f(w) {
                    result = append(result, w)
                }
            }
            return result
        }

        // Map is a functional map. It takes a list of
        // WorkWith and a WorkWith Function that takes a WorkWith
        // and returns a modified WorkWith. The end result is
        // a list of modified WorkWiths
        func Map(ws []WorkWith, f func(w WorkWith) WorkWith) []WorkWith 
        {
            // the result should always be the same
            // length
            result := make([]WorkWith, len(ws))

            for pos, w := range ws {
                newW := f(w)
                result[pos] = newW
            }
            return result
        }

```

1.  创建一个名为 `functions.go` 的文件，内容如下：

```go
        package collections

        import "strings"

        // LowerCaseData does a ToLower to the
        // Data string of a WorkWith
        func LowerCaseData(w WorkWith) WorkWith {
            w.Data = strings.ToLower(w.Data)
            return w
        }

        // IncrementVersion increments a WorkWiths
        // Version
        func IncrementVersion(w WorkWith) WorkWith {
            w.Version++
            return w
        }

        // OldVersion returns a closures
        // that validates the version is greater than
        // the specified amount
        func OldVersion(v int) func(w WorkWith) bool {
            return func(w WorkWith) bool {
                return w.Version >= v
            }
        }

```

1.  创建一个名为 `example` 的新目录。

1.  导航到 `example`。

1.  创建一个名为 `main.go` 的文件，内容如下；请确保将 `collections` 导入修改为你在步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"

            "github.com/agtorre/go-cookbook/chapter3/collections"
        )

        func main() {
            ws := []collections.WorkWith{
                collections.WorkWith{"Example", 1},
                collections.WorkWith{"Example 2", 2},
            }

            fmt.Printf("Initial list: %#v\n", ws)

            // first lower case the list
            ws = collections.Map(ws, collections.LowerCaseData)
            fmt.Printf("After LowerCaseData Map: %#v\n", ws)

            // next increment all versions
            ws = collections.Map(ws, collections.IncrementVersion)
            fmt.Printf("After IncrementVersion Map: %#v\n", ws)

            // lastly remove all versions older than 3
            ws = collections.Filter(ws, collections.OldVersion(3))
            fmt.Printf("After OldVersion Filter: %#v\n", ws)
        }

```

1.  运行 `go run main.go`。

1.  你也可以运行以下操作：

```go
 go build ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 Initial list:         
      []collections.WorkWith{collections.WorkWith{Data:"Example", 
      Version:1}, collections.WorkWith{Data:"Example 2", Version:2}}
 After LowerCaseData Map:         
      []collections.WorkWith{collections.WorkWith{Data:"example", 
      Version:1}, collections.WorkWith{Data:"example 2", Version:2}}
 After IncrementVersion Map: 
      []collections.WorkWith{collections.WorkWith{Data:"example", 
      Version:2}, collections.WorkWith{Data:"example 2", Version:3}}
 After OldVersion Filter: 
      []collections.WorkWith{collections.WorkWith{Data:"example 2",        Version:3}}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

Go 中的闭包非常强大。尽管我们的集合函数不是泛型的，但它们相对较小，并且可以很容易地应用于我们的 `WorkWith` 结构体的各种函数。你可能注意到，我们在任何地方都没有返回错误。这些函数的想法是它们是纯的。除了我们在每次调用后选择覆盖它之外，原始列表没有副作用。

如果你需要将修改层应用于列表或列表的列表结构，这种模式可以帮你节省很多困惑，并使测试变得非常直接。同时，也可以将映射和过滤器链式连接起来，以实现非常表达性的编码风格。
