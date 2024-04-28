# 第三章：数据转换和组合

理解 Go 的类型系统是掌握 Go 开发各个层面的关键步骤。本章将展示一些转换数据类型、处理非常大的数字、处理货币、使用不同类型的编码和解码（包括 Base64 和`gob`），以及使用闭包创建自定义集合的示例。在本章中，将介绍以下配方：

+   转换数据类型和接口转换

+   使用 math 和 math/big 处理数值数据类型

+   货币转换和 float64 考虑

+   使用指针和 SQL NullTypes 进行编码和解码

+   编码和解码 Go 数据

+   在 Go 中使用结构标签和基本反射

+   使用闭包实现集合

# 技术要求

为了继续本章中的所有配方，请根据以下步骤配置您的环境：

1.  在您的操作系统上下载并安装 Go 1.12.6 或更高版本，网址为[`golang.org/doc/install`](https://golang.org/doc/install)。

1.  打开一个终端/控制台应用程序，并创建并导航到一个项目目录，例如`~/projects/go-programming-cookbook`。所有的代码都将从这个目录运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`。如果愿意，您可以从该目录中工作，而不必手动输入示例：

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 转换数据类型和接口转换

通常情况下，Go 在将数据从一种类型转换为另一种类型时非常灵活。一种类型可以继承另一种类型，如下所示：

```go
type A int
```

我们总是可以将类型强制转换回我们继承的类型，如下所示：

```go
var a A = 1
fmt.Println(int(a))
```

还有一些方便的函数，可以使用类型转换进行数字之间的转换，使用`fmt.Sprint`和`strconv`进行字符串和其他类型之间的转换，使用反射进行接口和类型之间的转换。本配方将探讨一些基本的转换，这些转换将贯穿本书使用。

# 如何做...

以下步骤涵盖了如何编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter3/dataconv`的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/dataconv 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/dataconv    
```

1.  从`~/projects/go-programming-cookbook-original/chapter3/dataconv`复制测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`dataconv.go`的文件，内容如下：

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

1.  创建一个名为`strconv.go`的文件，内容如下：

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

1.  创建一个名为`interfaces.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter3/dataconv"

        func main() {
            dataconv.ShowConv()
            if err := dataconv.Strconv(); err != nil {
                panic(err)
            }
            dataconv.Interfaces()
        }
```

1.  运行`go run main.go`。您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

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

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

本配方演示了如何通过使用`strconv`包和接口反射将类型包装在新类型中来进行类型转换。这些方法允许 Go 开发人员快速在各种抽象的 Go 类型之间进行转换。前两种方法在编译期间将返回错误，但接口反射中的错误可能直到运行时才会被发现。如果您错误地反射到一个不受支持的类型，您的程序将会崩溃。在不同类型之间切换是一种泛化的方式，本配方也进行了演示。

转换对于诸如`math`这样专门操作`float64`的包非常重要。

# 使用 math 和 math/big 处理数值数据类型

`math`和`math/big`包专注于向 Go 语言公开更复杂的数学运算，如`Pow`、`Sqrt`和`Cos`。`math`包本身主要操作`float64`，除非函数另有说明。`math/big`包用于无法用 64 位值表示的数字。这个配方将展示`math`包的一些基本用法，并演示如何使用`math/big`来进行斐波那契数列。

# 如何做...

以下步骤涵盖了如何编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter3/math`的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/math 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/math    
```

1.  从`~/projects/go-programming-cookbook-original/chapter3/math`复制测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`fib.go`的文件，内容如下：

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
                return big.NewInt(1)
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

1.  创建一个名为`math.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter3/math"
        )

        func main() {
            math.Examples()

            for i := 0; i < 10; i++ {
                fmt.Printf("%v ", math.Fib(i))
            }
            fmt.Println()
        }
```

1.  运行`go run main.go`。您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
5
10
9
Pi: 3.141592653589793 E: 2.718281828459045
1 1 2 3 5 8 13 21 34 55
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`math`包使得在 Go 语言中执行复杂的数学运算成为可能。这个配方应该与这个包一起使用，用于执行复杂的浮点运算并根据需要在各种类型之间进行转换。值得注意的是，即使使用`float64`，对于某些浮点数仍可能存在舍入误差；以下配方演示了一些处理这种情况的技巧。

`math/big`部分展示了一个递归的斐波那契数列。如果您修改`main.go`以循环超过 10 次，如果使用`big.Int`而不是`int64`，您将很快溢出。`big.Int`包还有一些辅助方法，可以将大类型转换为其他类型。

# 货币转换和 float64 注意事项

处理货币始终是一个棘手的过程。将货币表示为`float64`可能很诱人，但在进行计算时可能会导致一些非常棘手（和错误的）舍入错误。因此，最好将货币以美分的形式存储为`int64`实例。

当从表单、命令行或其他来源收集用户输入时，货币通常以美元形式表示。因此，最好将其视为字符串，并直接将该字符串转换为美分，而不进行浮点转换。这个配方将介绍将货币的字符串表示转换为`int64`（美分）实例的方法，并再次转换回去。

# 如何做...

以下步骤涵盖了如何编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter3/currency`的新目录。

1.  导航到这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/currency 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/currency    
```

1.  从`~/projects/go-programming-cookbook-original/chapter3/currency`复制测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`dollars.go`的文件，内容如下：

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

1.  创建一个名为`pennies.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter3/currency"
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

1.  运行`go run main.go`。您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

```go
$ go run main.go
User input converted to 1593 pennies
Added 15 cents, new values is 16.08 dollars
```

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个配方利用`strconv`和`strings`包将货币在字符串格式的美元和`int64`的便士之间进行转换。它可以在不转换为`float64`类型的情况下进行，这可能会导致舍入误差，并且仅在验证时才这样做。

`strconv.ParseInt`和`strconv.FormatInt`函数非常有用，用于将`int64`和字符串相互转换。我们还利用了 Go 字符串可以根据需要轻松添加和切片的特点。

# 使用指针和 SQL NullTypes 进行编码和解码

在 Go 中对对象进行编码或解码时，未明确设置的类型将被设置为它们的默认值。例如，字符串将默认为空字符串（`""`），整数将默认为`0`。通常情况下，这是可以的，除非`0`对于您的 API 或服务来说有特殊含义。

此外，如果您使用`struct`标签，比如`json omitempty`，即使它们是有效的，`0`值也会被忽略。另一个例子是从 SQL 返回的`Null`。什么值最能代表`Int`的`Null`？这个示例将探讨 Go 开发人员处理这个问题的一些方法。

# 如何做...

以下步骤涵盖了如何编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter3/nulls`的新目录。

1.  进入这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/nulls 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/nulls    
```

1.  从`~/projects/go-programming-cookbook-original/chapter3/nulls`复制测试，或者使用这个练习来编写一些您自己的代码！

1.  创建一个名为`base.go`的文件，内容如下：

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

1.  创建一个名为`pointer.go`的文件，内容如下：

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

1.  创建一个名为`nullencoding.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录，并进入该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter3/nulls"
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

1.  运行`go run main.go`。您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

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

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

从值切换到指针是在编组和解组时表达空值的一种快速方式。设置这些值可能有点棘手，因为您不能直接将它们分配给指针，`-- *a := 1`，但是，这是一种灵活的处理方法。

这个示例还演示了使用`sql.NullInt64`类型的替代方法。这通常用于 SQL，如果返回的不是`Null`，则`valid`会被设置；否则，它会设置`Null`。我们添加了一个`MarshalJSON`方法和一个`UnmarshallJSON`方法，以允许这种类型与`JSON`包进行交互，并且我们选择使用指针，以便`omitempty`可以继续按预期工作。

# 编码和解码 Go 数据

Go 提供了许多除了 JSON、TOML 和 YAML 之外的替代编码类型。这些主要用于在 Go 进程之间传输数据，比如使用线协议和 RPC，或者在某些字符格式受限的情况下。

这个示例将探讨如何编码和解码`gob`格式和`base64`。后面的章节将探讨诸如 GRPC 之类的协议。

# 如何做...

以下步骤涵盖了如何编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter3/encoding`的新目录。

1.  进入这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/encoding 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/encoding    
```

1.  从`~/projects/go-programming-cookbook-original/chapter3/encoding`复制测试，或者使用这个练习来编写一些您自己的代码！

1.  创建一个名为`gob.go`的文件，内容如下：

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

1.  创建一个名为`base64.go`的文件，内容如下：

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

            if _, err := encoder.Write([]byte("encoding some other 
            data")); err != nil {
                return err
            }

            // be sure to close
            if err := encoder.Close(); err != nil {
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

1.  创建一个名为`example`的新目录，并进入该目录。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter3/encoding"
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

1.  运行`go run main.go`。您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

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

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

Gob 编码是一种以 Go 数据类型为基础构建的流格式。当发送和编码许多连续的项目时，它是最有效的。对于单个项目，其他编码格式，如 JSON，可能更有效和更便携。尽管如此，`gob`编码使得将大型、复杂的结构编组并在另一个进程中重建它们变得简单。尽管这里没有展示，`gob`也可以在自定义类型或具有自定义`MarshalBinary`和`UnmarshalBinary`方法的非导出类型上操作。

Base64 编码对于通过 URL 在`GET`请求中进行通信或生成二进制数据的字符串表示编码非常有用。大多数语言都可以支持这种格式，并在另一端解组数据。因此，在不支持 JSON 格式的情况下，通常会对诸如 JSON 有效负载之类的东西进行编码。

# Go 中的结构标签和基本反射

反射是一个复杂的主题，无法在一篇文章中完全涵盖；然而，反射的一个实际应用是处理结构标签。在本质上，`struct`标签只是键-值字符串：你查找键，然后处理值。正如你所想象的那样，对于诸如 JSON 编组和解组这样的事情，处理这些值有很多复杂性。

`reflect`包旨在审查和理解接口对象。它有助手方法来查看不同种类的结构、值、`struct`标签等。如果你需要超出基本接口转换的东西，比如本章开头的内容，这就是你应该查看的包。

# 如何做...

以下步骤涵盖了如何编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter3/tags`的新目录。

1.  进入这个目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/tags 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/tags    
```

1.  从`~/projects/go-programming-cookbook-original/chapter3/tags`复制测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`serialize.go`的文件，内容如下：

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

1.  创建一个名为`deserialize.go`的文件，内容如下：

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

1.  创建一个名为`tags.go`的文件，内容如下：

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

1.  创建一个名为`example`的新目录并进入。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter3/tags"
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

1.  运行`go run main.go`。你也可以运行以下命令：

```go
$ go build $ ./example
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
Deserialize results: tags.Person{Name:"Aaron", City:"Seattle",         
State:"WA", Misc:"", Year:0}
```

1.  如果你复制或编写了自己的测试，返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个示例创建了一个字符串序列化格式，它接受一个`struct`值并将所有字符串字段序列化为可解析的格式。这个示例不处理某些边缘情况；特别是，字符串不能包含冒号（`:`）或分号（`;`）字符。以下是它的行为摘要：

+   如果一个字段是字符串，它将被序列化/反序列化。

+   如果一个字段不是字符串，它将被忽略。

+   如果字段的`struct`标签包含序列化的“键”，那么该键将成为返回的序列化/反序列化环境。

+   不处理重复。

+   如果未指定`struct`标签，则使用字段名。

+   如果`struct`标签的值是连字符（`-`），则该字段将被忽略，即使它是一个字符串。

还有一些需要注意的是，反射不能完全处理非导出值。

# 通过闭包实现集合

如果您一直在使用函数式或动态编程语言，您可能会觉得`for`循环和`if`语句会产生冗长的代码。对列表进行处理时使用`map`和`filter`等函数构造可能很有用，并且可以使代码看起来更可读；但是，在 Go 中，这些类型不在标准库中，并且在没有泛型或非常复杂的反射和使用空接口的情况下很难泛化。这个配方将为您提供使用 Go 闭包实现集合的一些基本示例。

# 如何做...

以下步骤涵盖了如何编写和运行您的应用程序：

1.  从您的终端/控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter3/collections`的新目录。

1.  转到此目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/collections 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter3/collections    
```

1.  从`~/projects/go-programming-cookbook-original/chapter3/collections`复制测试，或者将其用作编写自己代码的练习！

1.  创建一个名为`collections.go`的文件，其中包含以下内容：

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

1.  创建一个名为`functions.go`的文件，其中包含以下内容：

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

1.  创建一个名为`example`的新目录，并转到该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "fmt"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter3/collections"
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

1.  运行`go run main.go`。您也可以运行以下命令：

```go
$ go build $ ./example
```

您应该看到以下输出：

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
[]collections.WorkWith{collections.WorkWith{Data:"example 2",        
Version:3}}
```

1.  如果您复制或编写了自己的测试，请返回上一个目录并运行`go test`。确保所有测试都通过。

# 工作原理...

Go 中的闭包非常强大。虽然我们的`collections`函数不是通用的，但它们相对较小，可以很容易地应用于我们的`WorkWith`结构，而只需使用各种函数添加最少的代码。您可能会注意到，我们没有在任何地方返回错误。这些函数的理念是它们是纯粹的：原始列表没有副作用，除了我们选择在每次调用后覆盖它。

如果您需要对列表或列表结构应用修改层，则此模式可以帮助您避免许多混乱，并使测试变得非常简单。还可以将映射和过滤器链接在一起，以实现非常表达的编码风格。
