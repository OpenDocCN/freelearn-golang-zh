# 第三章。Go 控制流

Go 从 C 家族语言中借用了其控制流语法的大部分。它支持所有预期的控制结构，包括 `if...else`、`switch`、`for` 循环，甚至 `goto`。然而，`while` 或 `do...while` 语句却明显缺失。本章以下内容将探讨 Go 的控制流元素，其中一些你可能已经熟悉，而其他则带来了一组在其他语言中找不到的新功能：

+   `if` 语句

+   `switch` 语句

+   类型 `Switch`

+   `for` 语句

# `if` 语句

Go 中的 `if` 语句借鉴了其他 C 类似语言的基本结构形式。当跟在 `if` 关键字后面的布尔表达式评估为 `true` 时，该语句有条件地执行一个代码块，如下面的简化的程序所示，该程序显示有关世界货币的信息：

```go
import "fmt" 

type Currency struct { 
  Name    string 
  Country string 
  Number  int 
} 

var CAD = Currency{ 
    Name: "Canadian Dollar",  
    Country: "Canada",  
    Number: 124} 

var FJD = Currency{ 
    Name: "Fiji Dollar",  
    Country: "Fiji",  
    Number: 242} 

var JMD = Currency{ 
    Name: "Jamaican Dollar",  
    Country: "Jamaica",  
    Number: 388} 

var USD = Currency{ 
    Name: "US Dollar",  
    Country: "USA",  
    Number: 840} 

func main() { 
  num0 := 242 
  if num0 > 100 || num0 < 900 { 
    fmt.Println("Currency: ", num0) 
    printCurr(num0) 
  } else { 
    fmt.Println("Currency unknown") 
  } 

  if num1 := 388; num1 > 100 || num1 < 900 { 
    fmt.Println("Currency:", num1) 
    printCurr(num1) 
  } 
} 

func printCurr(number int) { 
  if CAD.Number == number { 
    fmt.Printf("Found: %+v\n", CAD) 
  } else if FJD.Number == number { 
    fmt.Printf("Found: %+v\n", FJD) 
  } else if JMD.Number == number { 
    fmt.Printf("Found: %+v\n", JMD) 
  } else if USD.Number == number { 
    fmt.Printf("Found: %+v\n", USD) 
  } else { 
    fmt.Println("No currency found with number", number) 
  } 
} 

```

golang.fyi/ch03/ifstmt.go

Go 中的 `if` 语句看起来与其他语言类似。然而，它省略了一些语法规则，同时强制执行新的规则：

+   测试表达式的括号不是必需的。虽然以下 `if` 语句可以编译，但它不符合习惯用法：

    ```go
          if (num0 > 100 || num0 < 900) { 
            fmt.Println("Currency: ", num0) 
            printCurr(num0) 
          } 

    ```

+   使用以下代替：

    ```go
          if num0 > 100 || num0 < 900 { 
            fmt.Println("Currency: ", num0) 
            printCurr(num0) 
          } 

    ```

+   代码块的括号总是必需的。以下片段将无法编译：

    ```go
          if num0 > 100 || num0 < 900 printCurr(num0) 

    ```

+   然而，这将编译：

    ```go
          if num0 > 100 || num0 < 900 {printCurr(num0)} 

    ```

+   然而，将 `if` 语句写在一行或多行上（无论语句块多么简单）是习惯用法。这鼓励了良好的风格和清晰度。以下片段将无问题编译：

    ```go
          if num0 > 100 || num0 < 900 {printCurr(num0)} 

    ```

+   然而，语句的推荐习惯布局是使用多行，如下所示：

    ```go
          if num0 > 100 || num0 < 900 { 
            printCurr(num0) 
          }
    ```

+   `if` 语句可以包含一个可选的 `else` 块，当 `if` 块中的表达式评估为 `false` 时执行。`else` 块中的代码必须使用多行括号包裹，如下面的片段所示：

    ```go
          if num0 > 100 || num0 < 900 { 
            fmt.Println("Currency: ", num0) 
            printCurr(num0) 
          } else { 
            fmt.Println("Currency unknown") 
          } 

    ```

+   `else` 关键字可以立即跟在另一个 `if` 语句后面，形成一个 `if...else...if` 链，正如前面列出的源代码中的 `printCurr()` 函数所使用的那样：

    ```go
          if CAD.Number == number { 
            fmt.Printf("Found: %+v\n", CAD) 
          } else if FJD.Number == number { 
            fmt.Printf("Found: %+v\n", FJD) 
          } 

    ```

`if...else...if` 语句链可以按需增长，并且可以通过一个可选的 `else` 语句来终止，以表达所有其他未测试的条件。再次强调，这是在 `printCurr()` 函数中完成的，该函数使用 `if...else...if` 块测试四个条件。最后，它包括一个 `else` 语句块来捕获任何其他未测试的条件：

```go
func printCurr(number int) { 
  if CAD.Number == number { 
    fmt.Printf("Found: %+v\n", CAD) 
  } else if FJD.Number == number { 
    fmt.Printf("Found: %+v\n", FJD) 
  } else if JMD.Number == number { 
    fmt.Printf("Found: %+v\n", JMD) 
  } else if USD.Number == number { 
    fmt.Printf("Found: %+v\n", USD) 
  } else { 
    fmt.Println("No currency found with number", number) 
  } 
}
```

然而，在 Go 中，编写这种深层次的 `if...else...if` 代码块的习惯用法，是使用无表达式的 `switch` 语句。这将在后面的 *Switch 语句* 部分中介绍。

## `if` 语句的初始化

`if` 语句支持复合语法，其中测试表达式前面有一个初始化语句。在运行时，初始化在评估测试表达式之前执行，如下面的代码片段（来自前面列出的程序）所示：

```go
if num1 := 388; num1 > 100 || num1 < 900 { 
  fmt.Println("Currency:", num1) 
  printCurr(num1) 
}  

```

初始化语句遵循正常的变量声明和初始化规则。初始化变量的作用域绑定到 `if` 语句块，超出此范围它们将变得不可达。这是 Go 中常用的一种惯用法，并且在本章中介绍的其他流程控制结构中也得到了支持。

# `switch` 语句

Go 还支持类似于 C 或 Java 等其他语言的 `switch` 语句。Go 中的 `switch` 语句通过评估 `case` 子句中的值或表达式来实现多路分支，如下所示，这是缩写的源代码：

```go
import "fmt" 

type Curr struct { 
  Currency string 
  Name     string 
  Country  string 
  Number   int 
} 

var currencies = []Curr{ 
  Curr{"DZD", "Algerian Dinar", "Algeria", 12}, 
  Curr{"AUD", "Australian Dollar", "Australia", 36}, 
  Curr{"EUR", "Euro", "Belgium", 978}, 
  Curr{"CLP", "Chilean Peso", "Chile", 152}, 
  Curr{"EUR", "Euro", "Greece", 978}, 
  Curr{"HTG", "Gourde", "Haiti", 332}, 
  ... 
} 

func isDollar(curr Curr) bool { 
  var bool result 
  switch curr { 
  default: 
    result = false 
  case Curr{"AUD", "Australian Dollar", "Australia", 36}: 
    result = true 
  case Curr{"HKD", "Hong Kong Dollar", "Hong Koong", 344}: 
    result = true 
  case Curr{"USD", "US Dollar", "United States", 840}: 
    result = true 
  } 
  return result 
} 
func isDollar2(curr Curr) bool { 
  dollars := []Curr{currencies[2], currencies[6], currencies[9]} 
  switch curr { 
  default: 
    return false 
  case dollars[0]: 
    fallthrough 
  case dollars[1]: 
    fallthrough 
  case dollars[2]: 
    return true 
  } 
  return false 
} 

func isEuro(curr Curr) bool { 
  switch curr { 
  case currencies[2], currencies[4], currencies[10]: 
    return true 
  default: 
    return false 
  } 
} 

func main() { 
  curr := Curr{"EUR", "Euro", "Italy", 978} 
  if isDollar(curr) { 
    fmt.Printf("%+v is Dollar currency\n", curr) 
  } else if isEuro(curr) { 
    fmt.Printf("%+v is Euro currency\n", curr) 
  } else { 
    fmt.Println("Currency is not Dollar or Euro") 
  } 
  dol := Curr{"HKD", "Hong Kong Dollar", "Hong Koong", 344} 
  if isDollar2(dol) { 
    fmt.Println("Dollar currency found:", dol) 
  } 
} 

```

golang.fyi/ch03/switchstmt.go

Go 中的 `switch` 语句有一些有趣的特性和规则，使得它易于使用和推理：

+   从语义上讲，Go 的 `switch` 语句可以在两个上下文中使用：

    +   表达式 `switch` 语句

    +   类型 `switch` 语句

+   可以使用 `break` 语句提前退出 `switch` 代码块。

+   `switch` 语句可以包含一个默认情况，当没有其他情况表达式评估为匹配时。只能有一个默认情况，并且它可以放置在 `switch` 块内的任何位置。

## 使用表达式切换

表达式切换非常灵活，可以在许多需要程序控制流遵循多个路径的上下文中使用。表达式切换支持许多属性，如下列要点所述：

+   表达式切换可以测试任何类型的值。例如，以下代码片段（来自前面的程序列表）测试了类型为 `struct` 的变量 `Curr`：

    ```go
          func isDollar(curr Curr) bool { 
            var bool result 
            switch curr { 
              default: 
              result = false 
              case Curr{"AUD", "Australian Dollar", "Australia", 36}: 
              result = true 
              case Curr{"HKD", "Hong Kong Dollar", "Hong Koong", 344}: 
              result = true 
              case Curr{"USD", "US Dollar", "United States", 840}: 
              result = true 
            } 
            return result 
          } 
    ```

+   `case` 子句中的表达式从左到右、从上到下进行评估，直到找到一个与 `switch` 表达式相等的值（或表达式）。

+   当遇到第一个与 `switch` 表达式匹配的情况时，程序将执行 `case` 块中的语句，然后立即退出 `switch` 块。与其他语言不同，Go 的 `case` 语句不需要使用 `break` 来避免跌入下一个情况（请参阅 *Fallthrough cases* 部分）。例如，调用 `isDollar(Curr{"HKD", "Hong Kong Dollar", "Hong Kong", 344})` 将匹配前面函数中的第二个 `case` 语句。代码将结果设置为 `true` 并立即退出 `switch` 代码块。

+   `Case` 子句可以有多个值（或表达式），它们之间用逗号分隔，并隐含逻辑 `OR` 操作符。例如，在以下代码片段中，`switch` 表达式 `curr` 被测试与 `currencies[2]`、`currencies[4]` 或 `currencies[10]` 的值匹配，使用一个情况子句直到找到匹配项：

    ```go
          func isEuro(curr Curr) bool { 
            switch curr { 
              case currencies[2], currencies[4], currencies[10]: 
              return true 
              default: 
              return false 
            } 
          } 

    ```

+   `switch` 语句是 Go 中编写复杂条件语句的更简洁、更首选的惯用法。当将前面的代码片段与以下使用 `if` 语句执行相同比较的代码进行比较时，这一点很明显：

    ```go
          func isEuro(curr Curr) bool { 
            if curr == currencies[2] || curr == currencies[4],  
            curr == currencies[10]{ 
            return true 
          }else{ 
            return false 
          } 
        } 

    ```

## 跌入情况

```go
switch statement with a fallthrough in each case block:
```

```go
func isDollar2(curr Curr) bool { 
  switch curr { 
  case Curr{"AUD", "Australian Dollar", "Australia", 36}: 
    fallthrough 
  case Curr{"HKD", "Hong Kong Dollar", "Hong Kong", 344}: 
    fallthrough 
  case Curr{"USD", "US Dollar", "United States", 840}: 
    return true 
  default: 
    return false 
  } 
} 

```

golang.fyi/ch03/switchstmt.go

当匹配到某个 case 时，`fallthrough` 语句会级联到后续 `case` 块的第一个语句。因此，如果 `curr = Curr{"AUD", "Australian Dollar", "Australia", 36}"，第一个 case 将会被匹配。然后流程级联到第二个 case 块的第一个语句，它也是一个 `fallthrough` 语句。这导致第三个 case 块的第一个语句执行，以返回 `true`。这在功能上等同于以下代码片段：

```go
switch curr {  
case Curr{"AUD", "Australian Dollar", "Australia", 36},  
     Curr{"HKD", "Hong Kong Dollar", "Hong Kong", 344},  
     Curr{"USD", "US Dollar", "United States", 840}:  
  return true 
default: 
   return false 
}  

```

## 无表达式开关

Go 支持一种不指定表达式的 `switch` 语句形式。在这种格式中，每个 `case` 表达式必须评估为布尔值 `true`。以下简化的源代码说明了无表达式的 `switch` 语句的用法，如函数 `find()` 中所示。该函数遍历 `Curr` 值的切片，根据传入的 `struct` 函数中的字段值搜索匹配项：

```go
import ( 
  "fmt" 
  "strings" 
) 
type Curr struct { 
  Currency string 
  Name     string 
  Country  string 
  Number   int 
} 

var currencies = []Curr{ 
  Curr{"DZD", "Algerian Dinar", "Algeria", 12}, 
  Curr{"AUD", "Australian Dollar", "Australia", 36}, 
  Curr{"EUR", "Euro", "Belgium", 978}, 
  Curr{"CLP", "Chilean Peso", "Chile", 152}, 
  ... 
} 

func find(name string) { 
  for i := 0; i < 10; i++ { 
    c := currencies[i] 
    switch { 
    case strings.Contains(c.Currency, name), 
      strings.Contains(c.Name, name), 
      strings.Contains(c.Country, name): 
      fmt.Println("Found", c) 
    } 
  } 
} 

```

golang.fyi/ch03/switchstmt2.go

注意在先前的例子中，函数 `find()` 中的 `switch` 语句没有包含表达式。每个 `case` 表达式之间用逗号分隔，并且必须评估为布尔值，每个 `case` 表达式之间隐含地使用 `OR` 操作符。先前的 `switch` 语句等同于以下使用 `if` 语句实现相同逻辑的用法：

```go
func find(name string) { 
  for I := 0; i < 10; i++ { 
    c := currencies[i] 
    if strings.Contains(c.Currency, name) || 
      strings.Contains(c.Name, name) || 
      strings.Contains(c.Country, name){ 
      fmt.Println""Foun"", c) 
    } 
  } 
} 

```

## 开关初始化器

`switch` 关键字可以立即后跟一个简单的初始化语句，其中可以声明并初始化局部于 `switch` 代码块中的变量。这种方便的语法在初始化语句和 `switch` 表达式之间使用分号来声明变量，这些变量可以出现在 `switch` 代码块的任何位置。以下代码示例展示了如何通过初始化两个变量，`name` 和 `curr`，作为 `switch` 声明的一部分来实现这一点：

```go
func assertEuro(c Curr) bool {  
  switch name, curr := "Euro", "EUR"; {  
  case c.Name == name:  
    return true  
  case c.Currency == curr:  
    return true 
  }  
  return false  
} 

switch statement with an initializer. Notice the trailing semi-colon to indicate the separation between the initialization statement and the expression area for the switch. In the example, however, the switch expression is empty.
```

## 类型开关

给定 Go 强大的类型支持，该语言支持查询类型信息的能力应该不会令人惊讶。`type switch` 是一个使用 Go 接口类型来比较值的底层类型信息的语句（或表达式）。关于接口类型和类型断言的完整讨论超出了本节的范围。您可以在第八章*方法、接口和对象*中找到更多关于这个主题的详细信息。

尽管如此，为了完整性，这里提供了一个关于类型开关的简短讨论。目前，您需要知道的是，Go 提供了 `interface{}` 类型，或空接口，作为类型系统中所有其他类型的超类型。当一个值被赋予 `interface{}` 类型时，可以使用 `type switch` 来查询其底层类型信息，如下面的代码片段中的 `findAny()` 函数所示，以查询其底层类型信息：

```go
func find(name string) { 
  for i := 0; i < 10; i++ { 
    c := currencies[i] 
    switch { 
    case strings.Contains(c.Currency, name), 
      strings.Contains(c.Name, name), 
      strings.Contains(c.Country, name): 
      fmt.Println("Found", c) 
    } 
  } 
}  

func findNumber(num int) { 
  for _, curr := range currencies { 
    if curr.Number == num { 
      fmt.Println("Found", curr) 
    } 
  } 
}  

func findAny(val interface{}) {  
  switch i := val.(type) {  
  case int:  
    findNumber(i)  
  case string:  
    find(i)  
  default:  
    fmt.Printf("Unable to search with type %T\n", val)  
  }  
} 

func main() { 
findAny("Peso") 
  findAny(404) 
  findAny(978) 
  findAny(false) 
} 

```

golang.fyi/ch03/switchstmt2.go

函数 `findAny()` 以 `interface{}` 作为其参数。使用 `switch` 语句通过类型断言表达式确定变量 `val` 的底层类型和值：

```go
switch i := val.(type) 

```

注意前面类型断言表达式中关键字 `type` 的使用。每个情况子句将针对从 `val.(type)` 查询的类型信息进行测试。变量 `i` 将分配底层类型的实际值，并用于调用具有相应值的函数。默认块被调用以防止将任何意外的类型分配给参数 `val`。然后可以像以下代码片段所示那样使用 `findAny` 函数调用具有不同类型的值：

```go
findAny("Peso")  
findAny(404)  
findAny(978)  
findAny(false)  

```

# `for` 语句

作为与 C 家族相关的语言，Go 也支持 `for` 循环样式控制结构。然而，正如你现在可能已经预料到的，Go 的 `for` 语句以有趣且简单的方式工作。Go 中的 `for` 语句支持四种不同的惯用法，如下表所示：

| **for 语句** | **用途** |
| --- | --- |

| 条件 `for` | 用于在语义上替换 `while` 和 `do...while` 循环：

```go
for x < 10 { 
... 
}

```

|

| 无限循环 | 可以省略条件表达式以创建无限循环：

```go
for {
...
}
```

|

| 传统 | 这是 C 家族 `for` 循环的传统形式，具有初始化、测试和更新子句：

```go
for x:=0; x < 10; x++ {
...
}
```

|

| 范围 `for` | 用于遍历表示存储在数组、字符串（rune 数组）、切片、映射和通道中的项目集合的表达式：

```go
for i, val := range values {
...
}
```

|

注意，与 Go 中的所有其他控制语句一样，`for` 语句在其表达式周围不使用括号。所有循环代码块中的语句都必须在花括号内，否则编译器将产生错误。

## 条件

`for` 条件使用与在其他语言中找到的 `while` 循环语义上等价的结构。它使用关键字 `for`，后跟一个布尔表达式，只要评估为真，循环就会继续。以下简化的源代码列表显示了这种形式的 `for` 循环的示例：

```go
type Curr struct {  
  Currency string  
  Name     string  
  Country  string  
  Number   int  
}  
var currencies = []Curr{  
  Curr{"KES", "Kenyan Shilling", "Kenya", 404},  
  Curr{"AUD", "Australian Dollar", "Australia", 36},  
... 
} 

func listCurrs(howlong int) {  
  i := 0  
  for i < len(currencies) {  
    fmt.Println(currencies[i])  
    i++  
  }  
} 

```

golang.fyi/ch03/forstmt.go

在函数 `listCurrs()` 中，`for` 语句在条件表达式 `i < len(currencencies)` 返回 `true` 时迭代。必须注意确保每次迭代时更新 `i` 的值，以避免创建意外的无限循环。

## 无限循环

当在 `for` 语句中省略布尔表达式时，循环将无限进行，如下面的示例所示：

```go
for { 
  // statements here 
} 

```

这等价于在其他语言（如 C 或 Java）中找到的 `for(;;)` 或 `while (true)`。

## 传统 `for` 语句

```go
sortByNumber:
```

```go
type Curr struct {  
  Currency string  
  Name     string  
  Country  string  
  Number   int  
}  

var currencies = []Curr{  
  Curr{"KES", "Kenyan Shilling", "Kenya", 404},  
  Curr{"AUD", "Australian Dollar", "Australia", 36},  
... 
} 

func sortByNumber() {  
  N := len(currencies)  
  for i := 0; i < N-1; i++ {  
     currMin := i  
     for k := i + 1; k < N; k++ {  
    if currencies[k].Number < currencies[currMin].Number {  
         currMin = k  
    }  
     }  
     // swap  
     if currMin != i {  
        temp := currencies[i]  
    currencies[i] = currencies[currMin]  
    currencies[currMin] = temp  
     } 
  }  
} 

```

结果表明，传统的 `for` 语句是之前讨论的其他循环形式的超集，如下表所示：

| **for 语句** | **描述** |
| --- | --- |

|

```go
k:=initialize()
for ; k < 10; 
++{
...
}
```

| 初始化语句被省略。变量 `k` 在 `for` 语句外部初始化。然而，习惯上是用 `for` 语句初始化变量。 |
| --- |

|

```go
for k:=0; k < 10;{
...
}
```

| 在这里省略了 `update` 语句（在最后一个分号之后）。开发者必须在其他地方提供更新逻辑，否则可能会创建一个无限循环。 |
| --- |

|

```go
for ; k < 10;{
...
}
```

| 这与前面讨论的 `for` 条件形式（`for k < 10 { ... }`）等价。再次强调，变量 `k` 应在循环之前声明。必须小心更新 `k`，否则可能会创建一个无限循环。 |
| --- |

|

```go
for k:=0; ;k++{
...
}
```

| 这里省略了条件表达式。与之前一样，这会评估条件为 `true`，如果没有在循环中引入适当的终止逻辑，则会产生无限循环。 |
| --- |

|

```go
for ; ;{ ... }
```

| 这与形式 `for{ ... }` 等价，并会产生无限循环。 |
| --- |

在 `for` 循环中，初始化语句和 `update` 语句是常规的 Go 语句。因此，它们可以用来初始化和更新多个变量，正如 Go 所支持的。为了说明这一点，下一个示例在语句子句中同时初始化和更新了两个变量 `w1` 和 `w2`：

```go
import ( 
  "fmt" 
  "math/rand" 
) 

var list1 = []string{ 
"break", "lake", "go",  
"right", "strong",  
"kite", "hello"}  

var list2 = []string{ 
"fix", "river", "stop",  
"left", "weak", "flight",  
"bye"}  

func main() {  
  rand.Seed(31)  
  for w1, w2:= nextPair();  
  w1 != "go" && w2 != "stop";  
  w1, w2 = nextPair() {  

    fmt.Printf("Word Pair -> [%s, %s]\n", w1, w2)  
  }  
}  

func nextPair() (w1, w2 string) {  
  pos := rand.Intn(len(list1))  
  return list1[pos], list2[pos]  
} 

```

golang.fyi/ch03/forstmt2.go

初始化语句通过调用函数 `nextPair()` 来初始化变量 `w1` 和 `w2`。条件使用一个复合逻辑表达式，只要它评估为真，循环就会继续运行。最后，变量 `w1` 和 `w2` 都通过调用 `nextPair()` 在循环的每次迭代中更新。

## for range

最后，`for` 语句支持一种额外的形式，使用关键字 `range` 来遍历一个求值为数组、切片、映射、字符串或通道的表达式。for-range 循环具有以下通用形式：

*for [<标识符列表>] := range <表达式> { ... }*

根据 `range` 表达式产生的类型，每次迭代可以产生多达两个变量，如下表所示：

| **范围表达式** | **范围变量** |
| --- | --- |

| 遍历数组或切片：

```go
for i, v := range []V{1,2,3} {
...
}
```

| 范围生成两个值，其中 `i` 是循环索引，`v` 是集合中的值 `v[i]`。关于数组和切片的进一步讨论请参阅第七章，*复合类型*。 |
| --- |

| 遍历字符串值：

```go
for i, v := range "Hello" {
...
}
```

| 范围生成两个值，其中 `i` 是字符串中的字节索引，`v` 是 UTF-8 编码的字节值，在 `v[i]` 处返回为 rune。关于字符串类型的进一步讨论请参阅第四章，*数据类型*。 |
| --- |

| 遍历映射：

```go
for k, v := range map[K]V {
...
}
```

| `range` 产生两个值，其中 `k` 被分配为类型 `K` 的映射键的值，而 `v` 被存储在 `map[k]` 中，类型为 `V`。关于映射的进一步讨论请参阅第七章，*复合类型*。 |
| --- |

在通道值上循环：

```go
var ch chan T
for c := range ch {
...
}
```

| 有关通道的充分讨论请参阅第九章，*并发*。通道是一种双向导线，能够接收和发出值。`for...range` 语句将每次从通道接收到的值分配给变量 `c`。 |
| --- |

你应该知道，每次迭代发出的值是源中存储的原始项的副本。例如，在以下程序中，循环完成后切片中的值不会更新：

```go
import "fmt" 

func main() { 
  vals := []int{4, 2, 6} 
  for _, v := range vals { 
    v-- 
  } 
  fmt.Println(vals) 
} 

```

要使用 `for...range` 循环更新原始值，请使用索引表达式访问原始值，如下所示。

```go
func main() { 
  vals := []int{4, 2, 6} 
  for i, v := range vals { 
    vals[i] = v - 1 
  } 
  fmt.Println(vals) 
} 

```

在前面的示例中，值 `i` 被用于切片索引表达式 `vals[i]` 以更新存储在切片中的原始值。如果你只需要访问数组、切片或字符串（或映射的键）的索引值，则可以省略迭代值（赋值中的第二个变量）。例如，在以下示例中，`for...range` 语句仅在每个迭代中发出当前索引值：

```go
func printCurrencies() { 
  for i := range currencies { 
    fmt.Printf("%d: %v\n", i, currencies[i]) 
  } 
} 

```

golang.fyi/ch03/for-range-stmt.go

最后，有些情况下你可能对迭代生成的任何值都不感兴趣，而是对迭代机制本身感兴趣。Go 1.4 版本中引入了 for 语句的下一形式，以表达没有变量声明的 for range，如下代码片段所示：

```go
func main() { 
  for range []int{1,1,1,1} { 
    fmt.Println("Looping") 
  } 
}  

```

上述代码将在标准输出上打印 `"Looping"` 四次。当范围表达式在通道上时，会使用这种 `for...range` 循环的形式。它用于简单地通知通道中存在值。

# break、continue 和 goto 语句

Go 支持一组专门设计的语句，用于突然退出正在运行的代码块，例如 switch 和 for 语句，并将控制权转移到代码的不同部分。所有三个语句都可以接受一个标签标识符，该标识符指定了控制要转移到的代码中的目标位置。

## 标签标识符

在深入本节的核心之前，看看这些语句使用的标签是值得的。在 Go 中声明标签需要标识符后跟冒号，如下面的代码片段所示：

```go
DoSearch: 

```

标签的命名是风格问题。然而，应该遵循上一章中提到的标识符命名指南。标签必须位于函数内部。Go 编译器不允许在代码中悬挂未使用的标签。与变量类似，如果声明了标签，必须在代码中引用它。

## `break`语句

如同其他 C-like 语言一样，Go 的`break`语句终止并退出最内层封装的`switch`或`for`语句代码块，并将控制权转移到运行程序的另一部分。`break`语句可以接受一个可选的标签标识符，指定程序流程将从中恢复的标签位置。以下是关于`break`语句标签的一些属性需要记住：

+   标签必须在包含`break`语句的同一运行函数内声明

+   声明的标签必须立即跟随封装的控制语句（一个`for`循环或`switch`语句），其中嵌套了`break`语句

如果`break`语句后面跟着一个标签，控制权将转移到标签所在的位置，而不是标签后的语句。如果没有提供标签，`break`语句将突然退出，并将控制权转移到其封装的`for`语句（或`switch`语句）块之后的下一个语句。

以下代码是一个过度夸张的线性搜索，用于说明`break`语句的工作原理。它执行单词搜索，并在找到切片中单词的第一个实例时退出：

```go
import ( 
  "fmt" 
) 

var words = [][]string{  
  {"break", "lake", "go", "right", "strong", "kite", "hello"},  
  {"fix", "river", "stop", "left", "weak", "flight", "bye"},  
  {"fix", "lake", "slow", "middle", "sturdy", "high", "hello"},  
}  

func search(w string) {  
DoSearch:  
  for i := 0; i < len(words); i++ {  
    for k := 0; k < len(words[i]); k++ {  
      if words[i][k] == w {  
        fmt.Println("Found", w)  
        break DoSearch  
      }  
    }  
  }  
}  

break DoSearch statement will essentially exit out of the innermost for loop and cause the execution flow to continue after the outermost labeled for statement, which in this example, will simply end the program.
```

## `continue`语句

`continue`语句会导致控制流立即终止封装的`for`循环的当前迭代，并跳转到下一个迭代。`continue`语句也可以接受一个可选的标签。标签具有与`break`语句类似的属性：

+   标签必须在包含`continue`语句的同一运行函数内声明

+   声明的标签必须立即跟随一个封装的`for`循环语句，其中嵌套了`continue`语句

当`continue`语句存在于`for`语句块中时，如果存在，`for`循环将突然终止，并将控制权转移到最外层的带有标签的`for`循环块以继续。如果没有指定标签，`continue`语句将简单地转移到其封装的`for`循环块的开始，以继续下一个迭代。

为了说明，让我们回顾一下之前的单词搜索示例。这个版本使用了一个`continue`语句，它会导致在切片中找到搜索单词的多个实例：

```go
func search(w string) {  
DoSearch:  
  for i := 0; i < len(words); i++ {  
    for k := 0; k < len(words[i]); k++ {  
      if words[i][k] == w {  
        fmt.Println("Found", w)  
        continue DoSearch  
      }  
    }  
  }  
} 

```

golang.fyi/ch03/breakstmt2.go

`continue DoSearch`语句会导致最内层循环的当前迭代停止，并将控制权转移到带有标签的外层循环，使其继续下一个迭代。

## `goto`语句

`goto`语句更加灵活，因为它允许将流程控制转移到函数内定义的目标标签的任意位置。`goto`语句导致控制突然转移到由`goto`语句引用的标签。以下是一个简单但功能性的示例，展示了 Go 的`goto`语句的实际应用：

```go
import "fmt" 

func main() {  
  var a string 
Start:  
  for {  
    switch {  
    case a < "aaa":  
      goto A  
    case a >= "aaa" && a < "aaabbb":  
      goto B  
    case a == "aaabbb":  
      break Start  
    }  
  A:  
    a += "a"  
    continue Start  
  B:  
    a += "b"  
    continue Start  
  }  
fmt.Println(a) 
} 

```

golang.fyi/ch03/gotostmt.go

代码使用`goto`语句跳转到`main()`函数的不同部分。注意，`goto`语句可以针对代码中任何地方定义的标签。代码中多余的`Start:`标签保留是为了完整性，在此上下文中并非必需（因为不带标签的`continue`会有相同的效果）。以下是在使用`goto`语句时的一些指导：

+   除非实现的逻辑只能通过`goto`分支来实现，否则请避免使用`goto`语句。这是因为过度使用`goto`语句会使代码更难推理和调试。

+   当可能时，将`goto`语句及其目标标签放置在相同的封装代码块内。

+   避免将标签放置在`goto`语句会导致跳过新变量声明或重新声明变量的地方。

+   Go 允许你从内部跳转到外部的封装代码块。

+   如果你尝试跳转到同级或封装代码块，将会出现编译错误。

# 摘要

本章提供了 Go 中控制流机制的概述，包括`if`、`switch`和`for`语句。虽然 Go 的流程控制结构看起来简单且易于使用，但它们功能强大，实现了现代语言所期望的所有分支原语。读者通过充分的细节和示例了解每个概念，以确保对主题的清晰理解。下一章通过介绍 Go 类型系统，继续深入探讨 Go 的基本概念。
