# 第十五章：使用反射

本章是关于**反射**，这是一种工具，允许应用程序检查自己的代码，克服 Go 静态类型和泛型缺乏所施加的一些限制。例如，这对于生成能够处理其接收到的任何类型输入的包可能非常有帮助。

本章将涵盖以下主题：

+   理解接口和类型断言

+   了解与基本类型的交互

+   使用复杂类型进行反射

+   评估反射的成本

+   学习反射使用的最佳实践

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅第三章，*Go 概述*。

# 什么是反射？

反射是一种非常强大的功能，允许**元编程**，即应用程序检查自身结构的能力。它非常有用，可以在运行时分析应用程序中的类型，并且在许多编码包中使用，例如 JSON 和 XML。

# 类型断言

我们在[第三章](https://cdp.packtpub.com/hands_on_systems_programming_with_go/wp-admin/post.php?post=38&action=edit#post_26)，*Go 概述*中简要提到了类型断言的工作原理。类型断言是一种操作，允许我们从接口到具体类型以及反之进行转换。它采用以下形式：

```go
# unsafe assertion
v := SomeVar.(SomeType)

# safe assertion
v, ok := SomeVar.(SomeType)
```

第一个版本是不安全的，它将一个值分配给一个变量。

将断言用作函数的参数也被视为不安全。如果断言错误，此类操作将引发**panic**：

```go
func main() {
    var a interface{} = "hello"
    fmt.Println(a.(string)) // ok
    fmt.Println(a.(int))    // panics!
}
```

完整示例可在此处找到：[`play.golang.org/p/hNN87SuprGR`](https://play.golang.org/p/hNN87SuprGR)。

第二个版本使用布尔值作为第二个值，并且它将显示操作的成功。如果断言不可能，第一个值将始终是断言类型的零值：

```go
func main() {
    var a interface{} = "hello"
    s, ok := a.(string) // true
    fmt.Println(s, ok)
    i, ok := a.(int) // false
    fmt.Println(i, ok)
}
```

完整示例可在此处找到：[`play.golang.org/p/BIba2ywkNF_j`](https://play.golang.org/p/BIba2ywkNF_j)。

# 接口断言

断言也可以从一个接口到另一个接口进行。想象一下有两个不同的接口：

```go
type Fooer interface {
    Foo()
}

type Barer interface {
    Bar()
}
```

让我们定义一个实现其中一个的类型，另一个实现两者的类型：

```go
type A int

func (A) Foo() {}

type B int

func (B) Bar() {}
func (B) Foo() {}
```

如果我们为第一个接口定义一个新变量，只有在底层值具有实现两者的类型时，对第二个的断言才会成功；否则，它将失败：

```go
func main() {
    var a Fooer 

    a = A(0)
    v, ok := a.(Barer)
    fmt.Println(v, ok)

    a = B(0) 
    v, ok = a.(Barer)
    fmt.Println(v, ok)
}
```

完整示例可在此处找到：[`play.golang.org/p/bX2rnw5pRXJ`](https://play.golang.org/p/bX2rnw5pRXJ)。

一个使用场景可能是拥有`io.Reader`接口，检查它是否也是`io.Closer`接口，并在需要时使用`ioutil.NopCloser`函数（返回`io.ReadCloser`接口）进行包装：

```go
func Closer(r io.Reader) io.ReadCloser {
    if rc, ok := r.(io.ReadCloser); ok {
        return rc
    }
    return ioutil.NopCloser(r)
}

func main() {
    log.Printf("%T", Closer(&bytes.Buffer{}))
    log.Printf("%T", Closer(&os.File{}))
}
```

完整示例可在此处找到：[`play.golang.org/p/hUEsDYHFE7i`](https://play.golang.org/p/hUEsDYHFE7i)。

在跳转到反射之前，接口有一个重要的方面需要强调——它的表示始终是一个元组接口值，其中值是一个具体类型，不能是另一个接口。

# 理解基本机制

`reflection`包允许您从任何`interface{}`变量中提取类型和值。可以使用以下方法完成：

+   使用`reflection.TypeOf`返回接口的类型到`reflection.Type`变量。

+   `reflection.ValueOf`函数使用`reflection.Value`变量返回接口的值。

# 值和类型方法

`reflect.Value`类型还携带可以使用`Type`方法检索的类型信息：

```go
func main() {
    var a interface{} = int64(23)
    fmt.Println(reflect.TypeOf(a).String())
    // int64
    fmt.Println(reflect.ValueOf(a).String())
    // <int64 Value>
    fmt.Println(reflect.ValueOf(a).Type().String())
    // int64
}
```

完整示例可在此处找到：[`play.golang.org/p/tmYuMc4AF1T`](https://play.golang.org/p/tmYuMc4AF1T)。

# 种类

`reflect.Type`的另一个重要属性是`Kind`，它是基本类型和通用复杂类型的枚举。`reflect.Kind`和`reflect.Type`之间的主要关系是，前者表示后者的内存表示。

对于内置类型，`Kind`和`Type`是相同的，但对于自定义类型，它们将不同 - `Type`值将是预期的值，但`Kind`值将是自定义类型定义的内置类型之一：

```go
func main() {
    var a interface{}

    a = "" // built in string
    t := reflect.TypeOf(a)
    fmt.Println(t.String(), t.Kind())

    type A string // custom type
    a = A("")
    t = reflect.TypeOf(a)
    fmt.Println(t.String(), t.Kind())
}
```

完整示例在此处可用：[`play.golang.org/p/qjiouk88INn`](https://play.golang.org/p/qjiouk88INn)。

对于复合类型，它将反映出主要类型而不是底层类型。这意味着指向结构或整数的指针是相同类型，`reflect.Pointer`：

```go
func main() {
    var a interface{}

    a = new(int) // int pointer
    t := reflect.TypeOf(a)
    fmt.Println(t.String(), t.Kind())

    a = new(struct{}) // struct pointer
    t = reflect.TypeOf(a)
    fmt.Println(t.String(), t.Kind())
}
```

完整示例在此处可用：[`play.golang.org/p/-uJjZvTuzVf`](https://play.golang.org/p/-uJjZvTuzVf)。

相同的推理适用于所有其他复合类型，例如数组，切片，映射和通道。

# 值到接口

就像我们可以从任何`interface{}`值获取`reflect.Value`一样，我们也可以执行相反的操作，并从`reflect.Value`获取`interface{}`。这是使用反射值的`Interface`方法完成的，并且如果需要，可以转换为具体类型。如果感兴趣的方法或函数接受空接口，例如`json.Marshal`或`fmt.Println`，则返回的值可以直接传递，而无需任何转换：

```go
func main() {
    var a interface{} = int(12)
    v := reflect.ValueOf(a)
    fmt.Println(v.String())
    fmt.Printf("%v", v.Interface())
}
```

完整示例在此处可用：[`play.golang.org/p/1942Dhm5sap`](https://play.golang.org/p/1942Dhm5sap)。

# 操纵值

将值转换为其反射形式，然后再转回值，如果值本身无法更改，这是没有什么用的。这就是为什么我们的下一步是看看如何使用`reflection`包来更改它们。

# 更改值

`reflect.Value`类型有一系列方法，允许您更改底层值：

+   `Set`: 使用另一个`reflect.Value`

+   `SetBool`: 布尔值

+   `SetBytes`: 字节切片

+   `SetComplex`: 任何复杂类型

+   `SetFloat`: 任何浮点类型

+   `SetInt`: 任何有符号整数类型

+   `SetPointer`: 指针

+   `SetString`: 字符串

+   `SetUint`: 任何无符号整数

为了设置一个值，它需要是可编辑的，这发生在特定条件下。为了验证这一点，有一个方法`CanSet`，如果一个值可以被更改，则返回`true`。如果值无法更改，但仍然调用了`Set`方法，应用程序将会引发恐慌：

```go
func main() {
    var a = int64(12)
    v := reflect.ValueOf(a)
    fmt.Println(v.String(), v.CanSet())
    v.SetInt(24)
}
```

完整示例在此处可用：[`play.golang.org/p/hKn8qNtn0gN`](https://play.golang.org/p/hKn8qNtn0gN)。

为了进行更改，值需要是可寻址的。如果可以修改对象保存的实际存储位置，则值是可寻址的。当使用基本内置类型（例如`string`）创建新值时，传递给函数的是`interface{}`，它包含字符串的副本。

更改此副本将导致副本的变化，而不会影响原始变量。这将非常令人困惑，并且会使反射等实用工具的使用变得更加困难。这就是为什么，`reflect`包会引发恐慌 - 这是一个设计选择。这就解释了为什么最后一个示例会引发恐慌。

我们可以使用要更改的值的指针创建`reflect.Value`，并使用`Elem`方法访问该值。这将给我们一个可寻址的值，因为我们复制了指针而不是值，所以反射的值仍然是变量的指针：

```go
func main() {
    var a = int64(12)
    v := reflect.ValueOf(&a)
    fmt.Println(v.String(), v.CanSet())
    e := v.Elem()
    fmt.Println(e.String(), e.CanSet())
    e.SetInt(24)
    fmt.Println(a)
}
```

完整示例在此处可用：[`play.golang.org/p/-X5JsBrlr4Q`](https://play.golang.org/p/-X5JsBrlr4Q)。

# 创建新值

`reflect`包还允许我们使用类型创建新值。有几个函数允许我们创建一个值：

+   `MakeChan`创建一个新的通道值

+   `MakeFunc`创建一个新的函数值

+   `MakeMap`和`MakeMapWithSize`创建一个新的映射值

+   `MakeSlice`创建一个新的切片值

+   `New`创建一个指向该类型的新指针

+   `NewAt` 使用所选地址创建类型的新指针

+   `Zero` 创建所选类型的零值

以下代码显示了如何以几种不同的方式创建新值：

```go
func main() {
    t := reflect.TypeOf(int64(100))
    // zero value
    fmt.Printf("%#v\n", reflect.Zero(t))
    // pointer to int
    fmt.Printf("%#v\n", reflect.New(t))
}
```

完整的示例在这里：[`play.golang.org/p/wCTILSK1F1C`](https://play.golang.org/p/wCTILSK1F1C)。

# 处理复杂类型

在了解如何处理反射基础知识之后，我们现在将看到如何使用反射处理结构和地图等复杂数据类型。

# 数据结构

为了可更改性，结构与基本类型的工作方式完全相同； 我们需要获取指针的反射，然后访问其元素以能够更改值，因为直接使用结构会产生其副本，并且在更改值时会出现恐慌。

我们可以使用 `Set` 方法替换整个结构的值，然后获取新值的反射：

```go
func main() {
    type X struct {
        A, B int
        c string
    }
    var a = X{10, 100, "apple"}
    fmt.Println(a)
    e := reflect.ValueOf(&a).Elem()
    fmt.Println(e.String(), e.CanSet())
    e.Set(reflect.ValueOf(X{1, 2, "banana"}))
    fmt.Println(a)
}
```

完整的示例在这里：[`play.golang.org/p/mjb3gJw5CeA`](https://play.golang.org/p/mjb3gJw5CeA)。

# 更改字段

也可以使用 `Field` 方法修改单个字段：

+   `Field` 使用其索引返回一个字段

+   `FieldByIndex` 使用一系列索引返回嵌套字段

+   `FieldByName` 使用其名称返回一个字段

+   `FieldByNameFunc` 使用 `func(string) bool` 返回一个字段

让我们定义一个结构来更改字段的值，使用简单和复杂类型，至少有一个未导出的字段：

```go
type A struct {
    B
    x int
    Y int
    Z int
}

type B struct {
    F string
    G string
}
```

现在我们有了结构，我们可以尝试以不同的方式访问字段：

```go
func main() {
    var a A
    v := reflect.ValueOf(&a)
    func() {
        // trying to get fields from ptr panics
        defer func() {
            log.Println("panic:", recover())
        }()
        log.Printf("%s", v.Field(1).String())
    }()
    v = v.Elem()
    // changing fields by index
    for i := 0; i < 4; i++ {
        f := v.Field(i)
        if f.CanSet() && f.Type().Kind() == reflect.Int {
            f.SetInt(42)
        }
    }
    // changing nested fields by index
    v.FieldByIndex([]int{0, 1}).SetString("banana")

    // getting fields by name
    v.FieldByName("B").FieldByName("F").SetString("apple")

    log.Printf("%+v", a)
}
```

完整的示例在这里：[`play.golang.org/p/z5slFkIU5UE`](https://play.golang.org/p/z5slFkIU5UE)。

在处理 `reflect.Value` 和结构字段时，您得到的是其他值，无法与结构区分。 相反，当处理 `reflect.Type` 时，您获得一个 `reflect.StructField` 结构，它是另一种携带字段所有信息的类型。

# 使用标签

结构字段携带大量信息，从字段名称和索引到其标记：

```go
type StructField struct {
    Name string
    PkgPath string

    Type Type      // field type
    Tag StructTag  // field tag string
    Offset uintptr // offset within struct, in bytes
    Index []int    // index sequence for Type.FieldByIndex
    Anonymous bool // is an embedded field
}
```

可以使用 `reflect.Type` 方法获取 `reflect.StructField` 值：

+   `Field`

+   `FieldByName`

+   `FieldByIndex`

它们是由 `reflect.Value` 使用的相同方法，但它们返回不同的类型。 `NumField` 方法返回结构的字段总数，允许我们执行迭代：

```go
type Person struct {
    Name string `json:"name,omitempty" xml:"-"`
    Surname string `json:"surname,omitempty" xml:"-"`
}

func main() {
    v := reflect.ValueOf(Person{"Micheal", "Scott"})
    t := v.Type()
    fmt.Println("Type:", t)
    for i := 0; i < t.NumField(); i++ {
       fmt.Printf("%v: %v\n", t.Field(i).Name, v.Field(i))
    }
}
```

完整的示例在这里：[`play.golang.org/p/nkEADg77zFC`](https://play.golang.org/p/nkEADg77zFC)。

标签对于反射非常重要，因为它们可以存储有关字段的额外信息以及其他包如何与其交互的信息。 要向字段添加标签，需要在字段名称和类型之后插入一个字符串，该字符串应具有 `key:"value"` 结构。 一个字段可以在其标记中有多个元组，并且每对由空格分隔。 让我们看一个实际的例子：

```go
type A struct {
    Name    string `json:"name,omitempty" xml:"-"`
    Surname string `json:"surname,omitempty" xml:"-"`
}
```

该结构有两个字段，都带有标签，每个标签都有两对。 `Get` 方法返回特定键的值：

```go
func main() {
    t := reflect.TypeOf(A{})
    fmt.Println(t)
    for i := 0; i < t.NumField(); i++ {
        f := t.Field(i)
        fmt.Printf("%s JSON=%s XML=%s\n", f.Name, f.Tag.Get("json"), f.Tag.Get("xml"))
    }
}
```

完整的示例在这里：[`play.golang.org/p/P-Te8O1Hyyn`](https://play.golang.org/p/P-Te8O1Hyyn)。

# 地图和切片

您可以轻松使用反射来读取和操作地图和切片。 由于它们是编写应用程序的重要工具，让我们看看如何使用反射执行操作。

# 地图

`map` 类型允许您使用 `Key` 和 `Elem` 方法获取值和键的类型：

```go
func main() {
    maps := []interface{}{
        make(map[string]struct{}),
        make(map[int]rune),
        make(map[float64][]byte),
        make(map[int32]chan bool),
        make(map[[2]string]interface{}),
    }
    for _, m := range maps {
        t := reflect.TypeOf(m)
        fmt.Printf("%s k:%-10s v:%-10s\n", m, t.Key(), t.Elem())
    }
}
```

完整的示例在这里：[`play.golang.org/p/j__1jtgy-56`](https://play.golang.org/p/j__1jtgy-56)。

可以以正常访问映射的所有方式访问值：

+   通过键获取值

+   通过键的范围

+   通过值的范围

让我们看一个实际的例子：

```go
func main() {
    m := map[string]int64{
        "a": 10,
        "b": 20,
        "c": 100,
        "d": 42,
    }

    v := reflect.ValueOf(m)

    // access one field
    fmt.Println("a", v.MapIndex(reflect.ValueOf("a")))
    fmt.Println()

    // range keys
    for _, k := range v.MapKeys() {
        fmt.Println(k, v.MapIndex(k))
    }
    fmt.Println()

    // range keys and values
    i := v.MapRange()
    for i.Next() {
        fmt.Println(i.Key(), i.Value())
    }
}
```

请注意，我们无需传递指向地图的指针以使其可寻址，因为地图已经是指针。

每种方法都非常直接，并取决于您对映射的访问类型。设置值也是可能的，并且应该始终是可能的，因为映射是通过引用传递的。以下代码片段显示了一个实际示例：

```go
func main() {
    m := map[string]int64{}
    v := reflect.ValueOf(m)

    // setting one field
    v.SetMapIndex(reflect.ValueOf("key"), reflect.ValueOf(int64(1000)))

    fmt.Println(m)
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/JxK_8VPoWU0`](https://play.golang.org/p/JxK_8VPoWU0)。

还可以使用此方法取消设置变量，就像我们在调用`delete`函数时使用`reflect.Value`的零值作为第二个参数一样：

```go
func main() {
    m := map[string]int64{"a": 10}
    fmt.Println(m, len(m))

    v := reflect.ValueOf(m)

    // deleting field
    v.SetMapIndex(reflect.ValueOf("a"), reflect.Value{})

    fmt.Println(m, len(m))
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/4bPqfmaKzTC`](https://play.golang.org/p/4bPqfmaKzTC)。

输出将少一个字段，因为在`SetMapIndex`之后，映射的长度减少了。

# 切片

切片允许您使用`Len`方法获取其大小，并使用`Index`方法访问其元素。让我们在以下代码中看看它的运行情况：

```go
func main() {
    m := []int{10, 20, 100}
    v := reflect.ValueOf(m)

    for i := 0; i < v.Len(); i++ {
        fmt.Println(i, v.Index(i))
    }
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/ifq0O6bFIZc.`](https://play.golang.org/p/ifq0O6bFIZc)

由于始终可以获取切片元素的地址，因此也可以使用`reflect.Value`来更改切片中相应元素的内容：

```go
func main() {
    m := []int64{10, 20, 100}
    v := reflect.ValueOf(m)

    for i := 0; i < v.Len(); i++ {
        v.Index(i).SetInt(v.Index(i).Interface().(int64) * 2)
    }
    fmt.Println(m)
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/onuIvWyQ7GY`](https://play.golang.org/p/onuIvWyQ7GY)。

还可以使用`reflect`包将内容附加到切片。如果值是从切片的指针获得的，则此操作的结果也可以用于替换原始切片：

```go
func main() {
    var s = []int{1, 2}
    fmt.Println(s)

    v := reflect.ValueOf(s)
    // same as append(s, 3)
    v2 := reflect.Append(v, reflect.ValueOf(3))
    // s can't and does not change
    fmt.Println(v.CanSet(), v, v2)

    // using the pointer allows change
    v = reflect.ValueOf(&s).Elem()
    v.Set(v2)
    fmt.Println(v.CanSet(), v, v2)
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/2hXRg7Ih9wk`](https://play.golang.org/p/2hXRg7Ih9wk)。

# 函数

使用反射处理方法和函数可以收集有关特定条目签名的信息，并调用它。

# 分析函数

包中有一些`reflect.Type`的方法将返回有关函数的信息。这些方法如下：

+   `NumIn`：返回函数的输入参数数量

+   `In`：返回所选输入参数

+   `IsVariadic`：告诉您函数的最后一个参数是否是可变参数

+   `NumOut`：返回函数返回的输出值的数量

+   `Out`：返回选择输出的`Type`值

请注意，如果`reflect.Type`的类型不是`Func`，所有这些方法都会引发恐慌。我们可以通过定义一系列函数来测试这些方法：

```go
func Foo() {}

func Bar(a int, b string) {}

func Baz(a int, b string) (int, error) { return 0, nil }

func Qux(a int, b ...string) (int, error) { return 0, nil }
```

现在我们可以使用`reflect.Type`的方法来获取有关它们的信息：

```go
func main() {
    for _, f := range []interface{}{Foo, Bar, Baz, Qux} {
        t := reflect.TypeOf(f)
        name := runtime.FuncForPC(reflect.ValueOf(f).Pointer()).Name()
        in := make([]reflect.Type, t.NumIn())
        for i := range in {
            in[i] = t.In(i)
        }
        out := make([]reflect.Type, t.NumOut())
        for i := range out {
            out[i] = t.Out(i)
        }
        fmt.Printf("%q %v %v %v\n", name, in, out, t.IsVariadic())
    }
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/LAjjhw8Et60`](https://play.golang.org/p/LAjjhw8Et60)。

为了获取函数的名称，我们使用`runtime.FuncForPC`函数，它返回包含有关函数的运行时信息的`runtime.Func`，包括`name`、`file`和`line`。该函数以`uintptr`作为参数，可以通过函数的`reflect.Value`和其`Pointer`方法获得。

# 调用函数

虽然函数的类型显示了有关它的信息，但为了调用函数，我们需要使用它的值。

我们将向函数传递参数值列表，并获取函数调用返回的值：

```go
func main() {
    for _, f := range []interface{}{Foo, Bar, Baz, Qux} {
        v, t := reflect.ValueOf(f), reflect.TypeOf(f)
        name := runtime.FuncForPC(v.Pointer()).Name()
        in := make([]reflect.Value, t.NumIn())
        for i := range in {
            switch a := t.In(i); a.Kind() {
            case reflect.Int:
                in[i] = reflect.ValueOf(42)
            case reflect.String:
                in[i] = reflect.ValueOf("42")
            case reflect.Slice:
                switch a.Elem().Kind() {
                case reflect.Int:
                    in[i] = reflect.ValueOf(21)
                case reflect.String:
                    in[i] = reflect.ValueOf("21")
                }
            }
        }
        out := v.Call(in)
        fmt.Printf("%q %v%v\n", name, in, out)
    }
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/jPxO_G7YP2I`](https://play.golang.org/p/jPxO_G7YP2I)。

# 通道

反射允许我们创建通道，发送和接收数据，并且还可以使用`select`语句。

# 创建通道

可以通过`reflect.MakeChan`函数创建一个新的通道，该函数需要一个`reflect.Type`接口值和一个大小：

```go
func main() {
    t := reflect.ChanOf(reflect.BothDir, reflect.TypeOf(""))
    v := reflect.MakeChan(t, 0)
    fmt.Printf("%T\n", v.Interface())
}
```

一个完整的示例可以在这里找到：[`play.golang.org/p/7_RLtzjuTcz`](https://play.golang.org/p/7_RLtzjuTcz)。

# 发送、接收和关闭

`reflect.Value`类型提供了一些方法，必须与通道一起使用，`Send`和`Recv`用于发送和接收，`Close`用于关闭通道。让我们看一下这些函数和方法的一个示例用法：

```go
func main() {
    t := reflect.ChanOf(reflect.BothDir, reflect.TypeOf(""))
    v := reflect.MakeChan(t, 0)
    go func() {
        for i := 0; i < 10; i++ {
            v.Send(reflect.ValueOf(fmt.Sprintf("msg-%d", i)))
        }
        v.Close()
    }()
    for msg, ok := v.Recv(); ok; msg, ok = v.Recv() {
        fmt.Println(msg)
    }
}
```

这里有一个完整的示例：[`play.golang.org/p/Gp8JJmDbLIL`](https://play.golang.org/p/Gp8JJmDbLIL)。

# 选择语句

`select`语句可以使用`reflect.Select`函数执行。每个 case 由一个数据结构表示：

```go
type SelectCase struct {
    Dir  SelectDir // direction of case
    Chan Value     // channel to use (for send or receive)
    Send Value     // value to send (for send)
}
```

它包含操作的方向以及通道和值（用于发送操作）。方向可以是发送、接收或无（用于默认语句）：

```go
func main() {
    v := reflect.ValueOf(make(chan string, 1))
    fmt.Println("sending", v.TrySend(reflect.ValueOf("message"))) // true 1 1
    branches := []reflect.SelectCase{
        {Dir: reflect.SelectRecv, Chan: v, Send: reflect.Value{}},
        {Dir: reflect.SelectSend, Chan: v, Send: reflect.ValueOf("send")},
        {Dir: reflect.SelectDefault},
    }

    // send, receive and default
    i, recv, closed := reflect.Select(branches)
    fmt.Println("select", i, recv, closed)

    v.Close()
    // just default and receive
    i, _, closed = reflect.Select(branches[:2])
    fmt.Println("select", i, closed) // 1 false
}
```

这里有一个完整的示例：[`play.golang.org/p/_DgSYRIBkJA`](https://play.golang.org/p/_DgSYRIBkJA)。

# 反射反射

在讨论了反射在所有方面的工作原理之后，我们现在将专注于其缺点，即在标准库中使用它的情况，以及何时在包中使用它。

# 性能成本

反射允许代码灵活处理未知数据类型，通过分析它们的内存表示。这并不是没有成本的，除了复杂性之外，反射影响的另一个方面是性能。

我们可以创建一些示例来演示使用反射执行一些琐碎操作时的速度要慢得多。我们可以创建一个超时，并在 goroutines 中不断重复这些操作。当超时到期时，两个例程都将终止，我们将比较结果：

```go
func baseTest(fn1, fn2 func(int)) {
    ctx, canc := context.WithTimeout(context.Background(), time.Second)
    defer canc()
    go func() {
        for i := 0; ; i++ {
            select {
            case <-ctx.Done():
                return
            default:
                fn1(i)
            }
        }
    }()
    go func() {
        for i := 0; ; i++ {
            select {
            case <-ctx.Done():
                return
            default:
                fn2(i)
            }
        }
    }()
    <-ctx.Done()
}
```

我们可以比较普通的 map 写入与使用反射进行相同操作的速度：

```go
func testMap() {
    m1, m2 := make(map[int]int), make(map[int]int)
    m := reflect.ValueOf(m2)
    baseTest(func(i int) { m1[i] = i }, func(i int) {
        v := reflect.ValueOf(i)
        m.SetMapIndex(v, v)
    })
    fmt.Printf("normal %d\n", len(m1))
    fmt.Printf("reflect %d\n", len(m2))
}
```

我们还可以测试一下使用反射和不使用反射时读取的速度以及结构字段的设置：

```go
func testStruct() {
    type T struct {
        Field int
    }
    var m1, m2 T
    m := reflect.ValueOf(&m2).Elem()
    baseTest(func(i int) { m1.Field++ }, func(i int) {
        f := m.Field(0)
        f.SetInt(int64(f.Interface().(int) + 1))
    })
    fmt.Printf("normal %d\n", m1.Field)
    fmt.Printf("reflect %d\n", m2.Field)
}
```

通过反射执行操作时，性能至少下降了 50％，与标准的静态操作方式相比。当性能在应用程序中非常重要时，这种下降可能非常关键，但如果不是这种情况，那么使用反射可能是一个合理的选择。

# 标准库中的使用

标准库中有许多不同的包使用了`reflect`包：

+   `archive/tar`

+   `context`

+   `database/sql`

+   `encoding/asn1`

+   `encoding/binary`

+   `encoding/gob`

+   `encoding/json`

+   `encoding/xml`

+   `fmt`

+   `html/template`

+   `net/http`

+   `net/rpc`

+   `sort/slice`

+   `text/template`

我们可以思考他们对反射的处理方式，以编码包为例。这些包中的每一个都提供了编码和解码的接口，例如`encoding/json`包。我们定义了以下接口：

```go
type Marshaler interface {
    MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
    UnmarshalJSON([]byte) error
}
```

该包首先查看未知类型是否在解码或编码时实现了接口，如果没有，则使用反射。我们可以将反射视为包使用的最后资源。即使`sort`包也有一个通用的`slice`方法，使用反射来设置值和一个排序接口，避免使用反射。

还有其他包，比如`text/template`和`html/template`，它们读取运行时文本文件，其中包含关于要访问或使用的方法或字段的指令。在这种情况下，除了反射之外，没有其他方法可以完成它，也没有可以避免它的接口。

# 在包中使用反射

在了解了反射的工作原理以及它给代码增加的复杂性之后，我们可以考虑在我们正在编写的包中使用它。来自其创作者 Rob Pike 的 Go 格言之一拯救了我们：

清晰比聪明更好。反射从来不清晰。

反射的威力是巨大的，但这也是以使代码更加复杂和隐式为代价的。只有在极端必要的情况下才应该使用它，就像在模板场景中一样，并且应该在任何其他情况下避免使用它，或者至少提供一个接口来避免使用它，就像在编码包中一样。

# 属性文件

我们可以尝试使用反射来创建一个读取属性文件的包。

我们可以使用反射来创建一个读取属性文件的包：

1.  我们应该做的第一件事是定义一个避免使用反射的接口：

```go
type Unmarshaller interface {
    UnmarshalProp([]byte) error
}
```

1.  然后，我们可以定义一个解码器结构，它将利用一个`io.Reader`实例，使用行扫描器来读取各个属性：

```go
type Decoder struct {
    scanner *bufio.Scanner
}

func NewDecoder(r io.Reader) *Decoder {
    return &Decoder{scanner: bufio.NewScanner(r)}
}
```

1.  解码器也将被`Unmarshal`方法使用：

```go
func Unmarshal(data []byte, v interface{}) error {
    return NewDecoder(bytes.NewReader(data)).Decode(v)
}
```

1.  我们可以通过构建字段名称和索引的缓存来减少我们将使用反射的次数。这将很有帮助，因为在反射中，字段的值只能通过索引访问，而不能通过名称访问。

```go
var cache = make(map[reflect.Type]map[string]int)

func findIndex(t reflect.Type, k string) (int, bool) {
    if v, ok := cache[t]; ok {
        n, ok := v[k]
        return n, ok
    }
    m := make(map[string]int)
    for i := 0; i < t.NumField(); i++ {
        f := t.Field(i)
        if s := f.Name[:1]; strings.ToLower(s) == s {
            continue
        }
        name := strings.ToLower(f.Name)
        if tag := f.Tag.Get("prop"); tag != "" {
            name = tag
        }
        m[name] = i
    }
    cache[t] = m
    return findIndex(t, k)
}
```

1.  下一步是定义`Decode`方法。这将接收一个指向结构的指针，然后继续从扫描器中处理行并填充结构字段：

```go
func (d *Decoder) Decode(v interface{}) error {
    val := reflect.ValueOf(v)
    t := val.Type()
    if t.Kind() != reflect.Ptr && t.Elem().Kind() != reflect.Struct {
        return fmt.Errorf("%v not a struct pointer", t)
    }
    val = val.Elem()
    t = t.Elem()
    line := 0
    for d.scanner.Scan() {
        line++
        b := d.scanner.Bytes()
        if len(b) == 0 || b[0] == '#' {
            continue
        }
        parts := bytes.SplitN(b, []byte{':'}, 2)
        if len(parts) != 2 {
            return decodeError{line: line, err: errNoSep}
        }
        index, ok := findIndex(t, string(parts[0]))
        if !ok {
            continue
        }
        value := bytes.TrimSpace(parts[1])
        if err := d.decodeValue(val.Field(index), value); err != nil {
            return decodeError{line: line, err: err}
        }
    }
    return d.scanner.Err()
}
```

最重要的工作将由私有的`decodeValue`方法完成。首先要验证`Unmarshaller`接口是否满足，如果满足，则使用它。否则，该方法将使用反射来正确解码接收到的值。对于每种类型，它将使用`reflection.Value`的不同`Set`方法，并且如果遇到未知类型，则会返回错误：

```go
func (d *Decoder) decodeValue(v reflect.Value, value []byte) error {
    if v, ok := v.Addr().Interface().(Unmarshaller); ok {
        return v.UnmarshalProp(value)
    }
    switch valStr := string(value); v.Type().Kind() {
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        i, err := strconv.ParseInt(valStr, 10, 64)
        if err != nil {
            return err
        }
        v.SetInt(i)
    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        i, err := strconv.ParseUint(valStr, 10, 64)
        if err != nil {
            return err
        }
        v.SetUint(i)
    case reflect.Float32, reflect.Float64:
        i, err := strconv.ParseFloat(valStr, 64)
        if err != nil {
            return err
        }
        v.SetFloat(i)
    case reflect.String:
        v.SetString(valStr)
    case reflect.Bool:
        switch value := valStr; value {
        case "true":
            v.SetBool(true)
        case "false":
            v.SetBool(false)
        default:
            return fmt.Errorf("invalid bool: %s", value)
        }
    default:
        return fmt.Errorf("invalid type: %s", v.Type())
    }
    return nil
}
```

# 使用包

为了测试包是否按预期运行，我们可以创建一个满足`Unmarshaller`接口的自定义类型。实现的类型在解码时将字符串转换为大写：

```go
type UpperString string

func (u *UpperString) UnmarshalProp(b []byte) error {
        *u = UpperString(strings.ToUpper(string(b)))
        return nil
}
```

现在我们可以将类型用作结构字段，并验证它在`decode`操作中是否被正确转换：

```go
func main() {
        r := strings.NewReader(
                "\n# comment, ignore\nkey1: 10.5\nkey2: some string" +
                        "\nkey3: 42\nkey4: false\nspecial: another string\n")
        var v struct {
                Key1 float32
                Key2 string
                Key3 uint64
                Key4 bool
                Key5 UpperString `prop:"special"`
                key6 int
        }
        if err := prop.NewDecoder(r).Decode(&v); err != nil {
                log.Fatal(r)
        }
        log.Printf("%+v", v)
}
```

# 总结

在本章中，我们详细回顾了 Go 语言接口的内存模型，强调了接口始终包含一个具体类型。我们利用这些信息更好地了解了类型转换，并理解了当一个接口被转换为另一个接口时会发生什么。

然后，我们介绍了反射的基本机制，从类型和值开始，这是该包的两种主要类型。它们分别表示变量的类型和值。值允许您读取变量的内容，如果变量是可寻址的，还可以写入它。为了使变量可寻址，需要从其地址访问变量，例如使用指针。

我们还看到了如何使用反射处理复杂的数据类型，了解如何访问结构字段值。结构的数据类型可以用于获取关于字段的元数据，包括名称和标签，这些在编码包和其他第三方库中被广泛使用。

我们看到了如何创建和操作映射，包括添加、设置和删除值。对于切片，我们看到了如何编辑它们的值以及如何执行追加操作。我们还展示了如何使用通道发送和接收数据，甚至如何像静态类型编程一样使用`select`语句。

最后，我们列出了标准库中使用反射的地方，并对其计算成本进行了快速分析。我们用一些关于何时何地使用反射的提示来结束本章，无论是在库中还是在您编写的任何应用程序中。

下一章是本书的最后一章，它解释了如何使用 CGO 在 Go 语言中利用现有的 C 库。

# 问题

1.  Go 语言中接口的内存表示是什么？

1.  当一个接口类型被转换为另一个接口类型时会发生什么？

1.  反射中的`Value`、`Type`和`Kind`是什么？

1.  如果一个值是可寻址的，这意味着什么？

1.  为什么 Go 语言中结构字段标签很重要？

1.  反射的一般权衡是什么？

1.  你能描述一种使用反射的良好方法吗？
