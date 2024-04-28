# 第十章：使用 Go 进行数据编码

本章将向您展示如何使用更常见的编码来交换应用程序中的数据。编码是将数据转换的过程，当应用程序必须与另一个应用程序通信时可以使用它——使用相同的编码将允许两个程序相互理解。本章将解释如何处理基于文本的协议，如首先是 JSON，然后是如何使用二进制协议，如`gob`。

本章将涵盖以下主题：

+   使用基于文本的编码，如 JSON 和 XML

+   学习二进制编码，如`gob`和`protobuf`

# 技术要求

本章需要安装 Go 并设置您喜欢的编辑器。有关更多信息，请参阅第三章，*Go 概述*。

为了使用协议缓冲区，您需要安装`protobuf`库。有关说明，请访问[`github.com/golang/protobuf`](https://github.com/golang/protobuf)。

# 理解基于文本的编码

最易读的数据序列化格式是基于文本的格式。在本节中，我们将分析一些最常用的基于文本的编码方式，如 CSV、JSON、XML 和 YAML。

# CSV

**逗号分隔值**（**CSV**）是一种以文本形式存储数据的编码类型。每一行都是表格条目，一行的值由一个特殊字符分隔，通常是逗号，因此称为 CSV。CSV 文件的每个记录必须具有相同的值计数，并且第一个记录可以用作标题来描述每个记录字段：

```go
name,age,country
```

字符串值可以用引号引起来，以允许使用逗号。

# 解码值

Go 允许用户从任何`io.Reader`创建 CSV 读取器。可以使用`Read`方法逐个读取记录：

```go
func main() {
    r := csv.NewReader(strings.NewReader("a,b,c\ne,f,g\n1,2,3"))
    for {
        r, err := r.Read()
        if err != nil {
            log.Fatal(err)
        }
        log.Println(r)
    }
}
```

前面代码的完整示例可在[`play.golang.org/p/wZgVzMqAN_K`](https://play.golang.org/p/wZgVzMqAN_K)找到。

请注意，每条记录都是一个字符串切片，读取器期望每行的长度保持一致。如果一行的条目比第一行多或少，这将导致错误。还可以使用`ReadAll`一次读取所有记录。使用此方法的相同示例将如下所示：

```go
func main() {
 r := csv.NewReader(strings.NewReader("a,b,c\ne,f,g\n1,2,3"))
 records, err := r.ReadAll()
 if err != nil {
 log.Fatal(err)
 }
 for _, r := range records {
 log.Println(r)
 }
}
```

前面代码的完整示例可在[`play.golang.org/p/RJ-wxBB5fs6`](https://play.golang.org/p/RJ-wxBB5fs6)找到。

# 编码值

可以使用任何`io.Writer`创建 CSV 写入器。生成的写入器将被缓冲，因此为了不丢失数据，需要调用其方法`Flush`：这将确保缓冲区被清空，并且所有内容都传输到写入器。

`Write`方法接收一个字符串切片并以 CSV 格式对其进行编码。让我们看看下面的示例中它是如何工作的：

```go
func main() {
    const million = 1000000
    type Country struct {
        Code, Name string
        Population int
    }
    records := []Country{
        {Code: "IT", Name: "Italy", Population: 60 * million},
        {Code: "ES", Name: "Spain", Population: 46 * million},
        {Code: "JP", Name: "Japan", Population: 126 * million},
        {Code: "US", Name: "United States of America", Population: 327 * million},
    }
    w := csv.NewWriter(os.Stdout)
    defer w.Flush()
    for _, r := range records {
        if err := w.Write([]string{r.Code, r.Name, strconv.Itoa(r.Population)}); err != nil {
            fmt.Println("error:", err)
            os.Exit(1)
        }
    }
}
```

前面代码的完整示例可在[`play.golang.org/p/qwaz3xCJhQT`](https://play.golang.org/p/qwaz3xCJhQT)找到。

正如读者所知，有一种方法可以一次写入多条记录。它被称为`WriteAll`，我们可以在下一个示例中看到它：

```go
func main() {
    const million = 1000000
    type Country struct {
        Code, Name string
        Population int
    }
    records := []Country{
        {Code: "IT", Name: "Italy", Population: 60 * million},
        {Code: "ES", Name: "Spain", Population: 46 * million},
        {Code: "JP", Name: "Japan", Population: 126 * million},
        {Code: "US", Name: "United States of America", Population: 327 * million},
    }
    w := csv.NewWriter(os.Stdout)
    defer w.Flush()
    var ss = make([][]string, 0, len(records))
    for _, r := range records {
        ss = append(ss, []string{r.Code, r.Name, strconv.Itoa(r.Population)})
    }
    if err := w.WriteAll(ss); err != nil {
        fmt.Println("error:", err)
        os.Exit(1)
    }
}
```

前面代码的完整示例可在[`play.golang.org/p/lt_GBOLvUfk`](https://play.golang.org/p/lt_GBOLvUfk)找到。

`Write`和`WriteAll`之间的主要区别是第二个操作使用更多资源，并且在调用之前需要将记录转换为字符串切片。

# 自定义选项

读取器和写入器都有一些选项，可以在创建后更改。两个结构共享`Comma`字段，该字段是用于分隔字段的字符。还属于仅写入器的另一个重要字段是`FieldsPerRecord`，它是一个整数，确定读取器应为每个记录期望多少个字段。

+   如果大于`0`，它将是所需字段的数量。

+   如果等于`0`，它将设置为第一条记录的字段数。

+   如果为负，则将跳过对字段计数的所有检查，从而允许读取不一致的记录集。

让我们看一个实际的例子，一个不检查一致性并使用空格作为分隔符的读取器：

```go
func main() {
    r := csv.NewReader(strings.NewReader("a b\ne f g\n1"))
    r.Comma = ' '
    r.FieldsPerRecord = -1
    records, err := r.ReadAll()
    if err != nil {
        log.Fatal(err)
    }
    for _, r := range records {
        log.Println(r)
    }
}
```

前面代码的完整示例可在[`play.golang.org/p/KPHXRW5OxXT`](https://play.golang.org/p/KPHXRW5OxXT)找到。

# JSON

**JavaScript 对象表示法**（**JSON**）是一种轻量级的基于文本的数据交换格式。它的性质使人类能够轻松阅读和编写它，其小的开销使其非常适合基于 Web 的应用程序。

JSON 由两种主要类型的实体组成：

+   **名称/值对的集合**：名称/值表示为对象、结构或字典在各种编程语言中。

+   **有序值列表**：这些是集合或值的列表，通常表示为数组或列表。

对象用大括号括起来，每个键用冒号分隔，每个值用逗号分隔。列表用方括号括起来，元素用逗号分隔。这两种类型可以结合使用，因此列表也可以是值，对象可以是列表中的元素。在名称和值之外的空格、换行和制表符将被忽略，并用于缩进数据，使其更易于阅读。

取这个样本 JSON 对象：

```go
{
    "name: "Randolph",
    "surname": "Carter",
    "job": "writer",
    "year_of_birth": 1873
}
```

它可以压缩成一行，去除缩进，因为当数据长度很重要时，这是一个很好的做法，比如在 Web 服务器或数据库中：

```go
{"name:"Randolph","surname":"Carter","job":"writer","year_of_birth":1873}
```

在 Go 中，与 JSON 字典和列表相关联的默认类型是`map[string]interface{}`和`[]interface{}`。这两种类型（非常通用）能够承载任何 JSON 数据结构。

# 字段标签

`struct`也可以承载特定的 JSON 数据；所有导出的键将具有与相应字段相同的名称。为了自定义键，Go 允许我们在结构中的字段声明后跟一个字符串，该字符串应包含有关字段的元数据。

这些标签采用冒号分隔的键/值形式。值是带引号的字符串，可以使用逗号（例如`job,omitempty`）添加附加信息。如果有多个标签，空格用于分隔它们。让我们看一个使用结构标签的实际例子：

```go
type Character struct {
    Name        string `json:"name" tag:"foo"`
    Surname     string `json:"surname"`
    Job         string `json:"job,omitempty"`
    YearOfBirth int    `json:"year_of_birth,omitempty"`
}
```

此示例显示了如何为相同字段使用两个不同的标签（我们同时使用`json`和`foo`），并显示了如何指定特定的 JSON 键并引入`omitempty`标签，用于输出目的，以避免在字段具有零值时进行编组。

# 解码器

在 JSON 中解码数据有两种方式——第一种是使用字节片作为输入的`json.Unmarshal`函数，第二种是使用通用的`io.Reader`获取编码内容的`json.Decoder`类型。我们将在示例中使用后者，因为它将使我们能够使用诸如`strings.Reader`之类的结构。解码器的另一个优点是可以使用以下方法进行定制：

+   `DisallowUnknownFields`：如果发现接收数据结构中未知的字段，则解码将返回错误。

+   `UseNumber`：数字将存储为`json.Number`而不是`float64`。

这是使用`json.Decoder`类型进行数据解码的实际示例：

```go
r := strings.NewReader(`{
    "name":"Lavinia",
    "surname":"Whateley",
    "year_of_birth":1878
}`)
d := json.NewDecoder(r)
var c Character
if err := d.Decode(&c); err != nil {
    log.Fatalln(err)
}
log.Printf("%+v", c)
```

完整的示例在此处可用：[`play.golang.org/p/a-qt5Mk9E_J`](https://play.golang.org/p/a-qt5Mk9E_J)。

# 编码器

数据编码以类似的方式工作，使用`json.Marshal`函数获取字节片和`json.Encoder`类型，该类型使用`io.Writer`。后者更适合于灵活性和定制的明显原因。它允许我们使用以下方法更改输出：

+   `SetEscapeHTML`：如果为 true，则指定是否应在 JSON 引用的字符串内部转义有问题的 HTML 字符。

+   `SetIndent`：这允许我们指定每行开头的前缀，以及用于缩进输出 JSON 的字符串。

以下示例使用 encore 将数据结构编组到标准输出，使用制表符进行缩进：

```go
e := json.NewEncoder(os.Stdout)
e.SetIndent("", "\t")
c := Character{
    Name: "Charles Dexter",
    Surname: "Ward",
    YearOfBirth: 1902,
}
if err := e.Encode(c); err != nil {
    log.Fatalln(err)
}
```

这就是我们可以看到`Job`字段中`omitempty`标签的实用性。由于值是空字符串，因此跳过了它的编码。如果标签不存在，那么在姓氏之后会有`"job":"",`行。

# 编组器和解组器

通常使用反射包进行编码和解码，这是非常慢的。在诉诸它之前，编码器和解码器将检查数据类型是否实现了`json.Marshaller`和`json.Unmarshaller`接口，并使用相应的方法：

```go
type Marshaler interface {
        MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
        UnmarshalJSON([]byte) error
}
```

实现此接口可以实现更快的编码和解码，并且可以执行其他类型的操作，否则不可能，例如读取或写入未导出字段；它还可以嵌入一些操作，比如对数据进行检查。

如果目标只是包装默认行为，则需要定义另一个具有相同数据结构的类型，以便它失去所有方法。否则，在方法内调用`Marshal`或`Unmarshal`将导致递归调用，最终导致堆栈溢出。

在这个实际的例子中，我们正在定义一个自定义的`Unmarshal`方法，以在`Job`字段为空时设置默认值：

```go
func (c *Character) UnmarshalJSON(b []byte) error {
    type C Character
    var v C
    if err := json.Unmarshal(b, &v); err != nil {
        return err
    }
    *c = Character(v)
    if c.Job == "" {
        c.Job = "unknown"
    } 
    return nil
}
```

完整示例在此处可用：[`play.golang.org/p/4BjFKiMiVHO`](https://play.golang.org/p/4BjFKiMiVHO)。

`UnmarshalJSON`方法需要一个指针接收器，因为它必须实际修改数据类型的值，但对于`MarshalJSON`方法，没有真正的需要，最好使用值接收器——除非数据类型在`nil`时应该执行不同的操作：

```go
func (c Character) MarshalJSON() ([]byte, error) {
    type C Character
    v := C(c)
    if v.Job == "" {
        v.Job = "unknown"
    }
    return json.Marshal(v)
}
```

完整示例在此处可用：[`play.golang.org/p/Q-q-9y6v6u-`](https://play.golang.org/p/Q-q-9y6v6u-)。

# 接口

当使用接口类型时，编码部分非常简单，因为应用程序知道接口中存储了哪种数据结构，并将继续进行编组。做相反的操作并不那么简单，因为应用程序接收到的是一个接口而不是数据结构，并且不知道该怎么做，因此最终什么也没做。

一种非常有效的策略（即使涉及一些样板文件）是使用具体类型的容器，这将允许我们在`UnmarshalJSON`方法中处理接口。让我们通过定义一个接口和一些不同的实现来创建一个快速示例：

```go
type Fooer interface {
    Foo()
}

type A struct{ Field string }

func (a *A) Foo() {}

type B struct{ Field float64 }

func (b *B) Foo() {}
```

然后，我们定义一个包装接口并具有`Type`字段的类型：

```go
type Wrapper struct {
    Type string
    Value Fooer
}
```

然后，在编码之前填充`Type`字段：

```go
func (w Wrapper) MarshalJSON() ([]byte, error) {
    switch w.Value.(type) {
    case *A:
        w.Type = "A"
    case *B:
        w.Type = "B"
    default:
        return nil, fmt.Errorf("invalid type: %T", w.Value)
    }
    type W Wrapper
    return json.Marshal(W(w))
}
```

解码方法是更重要的：它使用`json.RawMessage`，这是一种用于延迟解码的特殊字节片类型。我们将首先从字符串字段中获取类型，并将值保留在原始格式中，以便使用正确的数据结构进行解码：

```go
func (w *Wrapper) UnmarshalJSON(b []byte) error {
    var W struct {
        Type string
        Value json.RawMessage
    }
    if err := json.Unmarshal(b, &W); err != nil {
        return err
    }
    var value interface{}
    switch W.Type {
    case "A":
        value = new(A)
    case "B":
        value = new(B)
    default:
        return fmt.Errorf("invalid type: %s", W.Type)
    }
    if err := json.Unmarshal(W.Value, &value); err != nil {
        return err
    }
    w.Type, w.Value = W.Type, value.(Fooer)
    return nil
}
```

完整示例在此处可用：[`play.golang.org/p/GXMK_hC8Bpv`](https://play.golang.org/p/GXMK_hC8Bpv)。

# 生成结构体

有一个非常有用的应用程序，当给定一个 JSON 字符串时，会自动尝试推断字段类型生成 Go 类型。您可以在此地址找到一个部署的：[`mholt.github.io/json-to-go/`](https://mholt.github.io/json-to-go/)。

它可以节省一些时间，大多数情况下，在简单转换后，数据结构已经是正确的。有时，它需要一些更改，比如数字类型，例如，如果您想要一个字段是`float`，但您的示例 JSON 是一个整数。

# JSON 模式

JSON 模式是描述 JSON 数据并验证数据有效性的词汇。它可用于测试，也可用作文档。模式指定元素的类型，并可以对其值添加额外的检查。如果类型是数组，还可以指定每个元素的类型和详细信息。如果类型是对象，则描述其字段。让我们看一个我们在示例中使用的`Character`结构的 JSON 模式：

```go
{
    "type": "object",
    "properties": {
        "name": { "type": "string" },
        "surname": { "type": "string" },
        "year_of_birth": { "type": "number"},
        "job": { "type": "string" }
    },
    "required": ["name", "surname"]
}
```

我们可以看到它指定了一个带有所有字段的对象，并指示哪些字段是必需的。有一些第三方 Go 包可以让我们非常容易地根据模式验证 JSON，例如[github.com/xeipuuv/gojsonschema](https://github.com/xeipuuv/gojsonschema)。

# XML

**可扩展标记语言**（**XML**）是另一种广泛使用的数据编码格式。它像 JSON 一样既适合人类阅读又适合机器阅读，并且是由**万维网联盟**（**W3C**）于 1996 年定义的。它专注于简单性，易用性和通用性，并且实际上被用作许多格式的基础，包括 RSS 或 XHTML。

# 结构

每个 XML 文件都以一个声明语句开始，该语句指定文件中使用的版本和编码，以及文件是否是独立的（使用的模式是内部的）。这是一个示例 XML 声明：

```go
<?xml version="1.0" encoding="UTF-8"?>
```

声明后面跟着一个 XML 元素树，这些元素由以下形式的标签界定：

+   `<tag>`：开放标签，定义元素的开始

+   `</tag>`：关闭标签，定义元素的结束

+   `<tag/>`：自关闭标签，定义没有内容的元素

通常，元素是嵌套的，因此一个标签内部有其他标签：

```go
<outer>
    <middle>
        <inner1>content</inner1>
        <inner2/>
    </middle>
</outer>
```

每个元素都可以以属性的形式具有附加信息，这些信息是在开放或自关闭标签内找到的以空格分隔的键/值对。键和值由等号分隔，并且值由双引号括起来。以下是具有属性的元素示例：

```go
<tag attribute="value" another="something">content</tag>
<selfclosing a="1000" b="-1"/>
```

# 文档类型定义

**文档类型定义**（**DTD**）是定义其他 XML 文档的结构和约束的 XML 文档。它可用于验证 XML 的有效性是否符合预期。XML 可以和应该指定自己的模式，以便简化验证过程。DTD 的元素如下：

+   **模式**：这代表文档的根。

+   **复杂类型**：它允许元素具有内容。

+   **序列**：这指定了描述的序列中必须出现的子元素。

+   **元素**：这代表一个 XML 元素。

+   **属性**：这代表父标签的 XML 属性。

这是我们在本章中使用的`Character`结构的示例模式声明：

```go
<?xml version="1.0" encoding="UTF-8" ?>
<xs:schema >
  <xs:element name="character">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="name" type="xs:string" use="required"/>
        <xs:element name="surname" type="xs:string" use="required"/>
        <xs:element name="year_of_birth" type="xs:integer"/>
        <xs:element name="job" type="xs:string"/>
      </xs:sequence>
      <xs:attribute name="id" type="xs:string" use="required"/>
    </xs:complexType>
 </xs:element>
</xs:schema>
```

我们可以看到它是一个包含其他元素序列的复杂类型元素（字符）的模式。

# 解码和编码

就像我们已经看到的 JSON 一样，数据解码和编码可以通过两种不同的方式实现：通过使用`xml.Unmarshal`和`xml.Marshal`提供或返回一个字节片，或者通过使用`xml.Decoder`和`xml.Encoder`类型与`io.Reader`或`io.Writer`一起使用。

我们可以通过将`Character`结构中的`json`标签替换为`xml`或简单地添加它们来实现：

```go
type Character struct {
    Name        string `xml:"name"`
    Surname     string `xml:"surname"`
    Job         string `xml:"job,omitempty"`
    YearOfBirth int    `xml:"year_of_birth,omitempty"`
}
```

然后，我们使用`xml.Decoder`来解组数据：

```go
r := strings.NewReader(`<?xml version="1.0" encoding="UTF-8"?>
<character>
 <name>Herbert</name>
 <surname>West</surname>
 <job>Scientist</job>
</character>
}`)
d := xml.NewDecoder(r)
var c Character
if err := d.Decode(&c); err != nil {
 log.Fatalln(err)
}
log.Printf("%+v", c)
```

完整示例可在此处找到：[`play.golang.org/p/esopq0SMhG_T`](https://play.golang.org/p/esopq0SMhG_T)。

在编码时，`xml`包将从使用的数据类型中获取根节点的名称。如果数据结构有一个名为`XMLName`的字段，则相对的 XML `struct`标签将用于根节点。因此，数据结构变为以下形式：

```go
type Character struct {
    XMLName     struct{} `xml:"character"`
    Name        string   `xml:"name"`
    Surname     string   `xml:"surname"`
    Job         string   `xml:"job,omitempty"`
    YearOfBirth int      `xml:"year_of_birth,omitempty"`
}
```

编码操作也非常简单：

```go
e := xml.NewEncoder(os.Stdout)
e.Indent("", "\t")
c := Character{
    Name:        "Henry",
    Surname:     "Wentworth Akeley",
    Job:         "farmer",
    YearOfBirth: 1871,
}
if err := e.Encode(c); err != nil {
    log.Fatalln(err)
}
```

完整示例可在此处找到：[`play.golang.org/p/YgZzdPDoaLX`](https://play.golang.org/p/YgZzdPDoaLX)。

# 字段标签

根标签的名称可以使用数据结构中的`XMLName`字段进行更改。字段标签的一些其他特性可能非常有用：

+   带有`-`的标记被省略。

+   带有`attr`选项的标记成为父元素的属性。

+   带有`innerxml`选项的标记被原样写入，对于懒惰解码很有用。

+   `omitempty`选项与 JSON 的工作方式相同；它不会为零值生成标记。

+   标记可以包含 XML 中的路径，使用`>`作为分隔符，如`a > b > c`。

+   匿名结构字段被视为其值的字段在外部结构中的字段。

让我们看一个使用其中一些特性的实际示例：

```go
type Character struct {
    XMLName     struct{} `xml:"character"`
    Name        string   `xml:"name"`
    Surname     string   `xml:"surname"`
    Job         string   `xml:"details>job,omitempty"`
    YearOfBirth int      `xml:"year_of_birth,attr,omitempty"`
    IgnoreMe    string   `xml:"-"`
}
```

这个结构产生以下 XML：

```go
<character year_of_birth="1871">
  <name>Henry</name>
  <surname>Wentworth Akeley</surname>
  <details>
    <job>farmer</job>
  </details>
</character>
```

完整示例在这里：[`play.golang.org/p/6zdl9__M0zF`](https://play.golang.org/p/6zdl9__M0zF)。

# 编组器和解组器

就像我们在 JSON 中看到的那样，`xml`包提供了一些接口来自定义类型在编码和解码操作期间的行为——这可以避免使用反射，或者可以用于建立不同的行为。该包提供的接口来获得这种行为是以下内容：

```go
type Marshaler interface {
    MarshalXML(e *Encoder, start StartElement) error
}

type MarshalerAttr interface {
    MarshalXMLAttr(name Name) (Attr, error)
}

type Unmarshaler interface {
        UnmarshalXML(d *Decoder, start StartElement) error
}

type UnmarshalerAttr interface {
        UnmarshalXMLAttr(attr Attr) error
}
```

有两对函数——一对用于解码或编码类型作为元素时使用，而另一对用于其作为属性时使用。让我们看看它的作用。首先，我们为自定义类型定义一个`MarshalXMLAttr`方法：

```go
type Character struct {
    XMLName struct{} `xml:"character"`
    ID ID `xml:"id,attr"`
    Name string `xml:"name"`
    Surname string `xml:"surname"`
    Job string `xml:"job,omitempty"`
    YearOfBirth int `xml:"year_of_birth,omitempty"`
}

type ID string

func (i ID) MarshalXMLAttr(name xml.Name) (xml.Attr, error) {
    return xml.Attr{
        Name: xml.Name{Local: "codename"},
        Value: strings.ToUpper(string(i)),
    }, nil
}
```

然后，我们对一些数据进行编组，我们会看到属性名称被替换为`codename`，其值为大写，正如方法所指定的那样：

```go
e := xml.NewEncoder(os.Stdout)
e.Indent("", "\t")
c := Character{
    ID: "aa",
    Name: "Abdul",
    Surname: "Alhazred",
    Job: "poet",
    YearOfBirth: 700,
}
if err := e.Encode(c); err != nil {
    log.Fatalln(err)
}
```

完整示例在这里：[`play.golang.org/p/XwJrMozQ6RY`](https://play.golang.org/p/XwJrMozQ6RY)。

# 生成结构

就像 JSON 一样，有一个第三方包可以从编码文件生成 Go 结构。对于 XML，我们有[`github.com/miku/zek`](https://github.com/miku/zek)。

它处理任何类型的 XML 数据，包括带有属性的元素，元素之间的间距或注释。

# YAML

**YAML**是一个递归缩写，代表**YAML 不是标记语言**，它是另一种广泛使用的数据编码格式的名称。它的成功部分归功于它比 JSON 和 XML 更容易编写，它的轻量级特性和灵活性。

# 结构

YAML 使用缩进来表示范围，使用换行符来分隔实体。序列中的元素以破折号开头，后跟一个空格。键和值之间用冒号分隔，用井号表示注释。这是样本 YAML 文件的样子：

```go
# list of characters
characters: 
    - name: "Henry"
      surname: "Armitage"
      year_of_birth: 1855
      job: "librarian"
    - name: "Francis"
      surname: "Wayland Thurston"
      job: "anthropologist"
```

JSON 和 YAML 之间更重要的区别之一是，虽然前者只能使用字符串作为键，但后者可以使用任何类型的标量值（字符串、数字和布尔值）。

# 解码和编码

YAML 不包含在 Go 标准库中，但有许多第三方库可用。处理此格式最常用的包是`go-yaml`包([`gopkg.in/yaml.v2`](https://gopkg.in/yaml.v2))。

它是使用以下标准编码包结构构建的：

+   有编码器和解码器。

+   有`Marshal`/`Unmarshal`函数。

+   它允许`struct`标记。

+   类型的行为可以通过实现定义的接口的方法来自定义。

接口略有不同——`Unmarshaler`接收默认编组函数作为参数，然后可以与不同于类型的数据结构一起使用：

```go
type Marshaler interface {
    MarshalYAML() (interface{}, error)
}

type Unmarshaler interface {
    UnmarshalYAML(unmarshal func(interface{}) error) error
}
```

我们可以像使用 JSON 标记一样使用`struct`标记：

```go
type Character struct {
    Name        string `yaml:"name"`
    Surname     string `yaml:"surname"`
    Job         string `yaml:"job,omitempty"`
    YearOfBirth int    `yaml:"year_of_birth,omitempty"`
}
```

我们可以使用它们来编码数据结构，或者在这种情况下，一系列结构：

```go
var chars = []Character{{
    Name:        "William",
    Surname:     "Dyer",
    Job:         "professor",
    YearOfBirth: 1875,
}, {
    Surname: "Danforth",
    Job:     "student",
}}
e := yaml.NewEncoder(os.Stdout)
if err := e.Encode(chars); err != nil {
    log.Fatalln(err)
}
```

解码方式相同，如下所示：

```go
r := strings.NewReader(`- name: John Raymond
 surname: Legrasse
 job: policeman
- name: "Francis"
 surname: Wayland Thurston
 job: anthropologist`)
// define a new decoder
d := yaml.NewDecoder(r)
var c []Character
// decode the reader
if err := d.Decode(&c); err != nil {
 log.Fatalln(err)
}
log.Printf("%+v", c)
```

我们可以看到创建`Decoder`所需的全部内容是`io.Reader`和接收结构以执行解码。

# 了解二进制编码

二进制编码协议使用字节，因此它们的字符串表示不友好。它们通常不可读作为字符串，很难编写，但它们的大小更小，导致应用程序之间的通信更快。

# BSON

BSON 是 JSON 的二进制版本。它被 MongoDB 使用，并支持一些在 JSON 中不可用的数据类型，例如日期和二进制。

有一些包实现了 BSON 编码和解码，其中两个非常广泛。一个在官方的 MongoDB Golang 驱动程序内部，`github.com/mongodb/mongo-go-driver`。另一个不是官方的，但自 Go 开始就存在，并且是非官方 MongoDB 驱动程序的一部分，`gopkg.in/mgo.v2`。

第二个与 JSON 包非常相似，无论是接口还是函数。这些接口被称为 getter 和 setter：

+   `GetBSON`返回将被编码的实际数据结构。

+   `SetBSON`接收`bson.Raw`，它是`[]byte`的包装器，可以与`bson.Unmarshal`一起使用。

这些 getter 和 setter 的用例如下：

```go
type Setter interface {
    SetBSON(raw Raw) error
}

type Getter interface {
    GetBSON() (interface{}, error)
}
```

# 编码

BSON 是为文档/实体设计的格式；因此，用于编码和解码的数据结构应该是结构体或映射，而不是切片或数组。`mgo`版本的`bson`不提供通常的编码器，而只提供 marshal：

```go
var char = Character{
    Name: "Robert",
    Surname: "Olmstead",
}
b, err := bson.Marshal(char)
if err != nil {
    log.Fatalln(err)
}
log.Printf("%q", b)
```

# 解码

相同的事情也适用于`Unmarshal`函数：

```go
r := []byte(",\x00\x00\x00\x02name\x00\a\x00\x00" +
 "\x00Robert\x00\x02surname\x00\t\x00\x00\x00" +
 "Olmstead\x00\x00")
var c Character
if err := bson.Unmarshal(r, &c); err != nil {
 log.Fatalln(err)
}
log.Printf("%+v", c)
```

# gob

`gob`编码是另一种内置于标准库中的二进制编码类型，实际上是由 Go 本身引入的。它是一系列数据项，每个数据项前面都有一个类型声明，并且不允许使用指针。它使用它们的值，禁止使用`nil`指针（因为它们没有值）。该包还存在与具有创建递归结构的指针的类型相关的问题，这可能导致意外的行为。

数字具有任意精度，可以是浮点数、有符号数或无符号数。有符号整数可以存储在任何有符号整数类型中，无符号整数可以存储在任何无符号整数类型中，浮点值可以接收到任何浮点变量中。但是，如果变量无法表示该值（例如溢出），解码将失败。字符串和字节切片使用非常高效的表示存储，尝试重用相同的基础数组。结构体只会解码导出的字段，因此函数和通道将被忽略。

# 接口

`gob`用于替换默认编组和解组行为的接口可以在`encoding`包中找到：

```go
type BinaryMarshaler interface {
        MarshalBinary() (data []byte, err error)
}

type BinaryUnmarshaler interface {
        UnmarshalBinary(data []byte) error
}
```

在解码阶段，任何不存在的结构字段都会被忽略，因为字段名称也是序列化的一部分。

# 编码

让我们尝试使用`gob`对一个结构进行编码：

```go
var char = Character{
    Name:    "Albert",
    Surname: "Wilmarth",
    Job:     "assistant professor",
}
s := strings.Builder{}
e := gob.NewEncoder(&s)
if err := e.Encode(char); err != nil {
    log.Fatalln(err)
}
log.Printf("%q", s.String())
```

# 解码

解码数据也非常简单；它的工作方式与我们已经看到的其他编码包相同：

```go
r := strings.NewReader("D\xff\x81\x03\x01\x01\tCharacter" +
    "\x01\xff\x82\x00\x01\x04\x01\x04Name" +
    "\x01\f\x00\x01\aSurname\x01\f\x00\x01\x03" +
    "Job\x01\f\x00\x01\vYearOfBirth\x01\x04\x00" +
    "\x00\x00*\xff\x82\x01\x06Albert\x01\bWilmarth" +
    "\x01\x13assistant professor\x00")
d := gob.NewDecoder(r)
var c Character
if err := d.Decode(&c); err != nil {
    log.Fatalln(err)
}
log.Printf("%+v", c)
```

现在，让我们尝试在不同的结构中解码相同的数据——原始数据和一些带有额外或缺少字段的数据。我们将这样做来查看该包的行为。让我们定义一个通用的解码函数，并将不同类型的结构传递给解码器：

```go
func runDecode(data []byte, v interface{}) {
    if err := gob.NewDecoder(bytes.NewReader(data)).Decode(v); err != nil {
        log.Fatalln(err)
    }
    log.Printf("%+v", v)    
}
```

让我们尝试改变结构体中字段的顺序，看看`gob`解码器是否仍然有效：

```go
runDecode(data, new(struct {
    YearOfBirth int    `gob:"year_of_birth,omitempty"`
    Surname     string `gob:"surname"`
    Name        string `gob:"name"`
    Job         string `gob:"job,omitempty"`
}))
```

让我们删除一些字段：

```go

runDecode(data, new(struct {
    Name string `gob:"name"`
}))
```

让我们在中间加一个字段：

```go
runDecode(data, new(struct {
    Name        string `gob:"name"`
    Surname     string `gob:"surname"`
    Country     string `gob:"country"`
    Job         string `gob:"job,omitempty"`
    YearOfBirth int    `gob:"year_of_birth,omitempty"`
}))
```

我们可以看到，即使我们混淆、添加或删除字段，该包仍然可以正常工作。但是，如果我们尝试将现有字段的类型更改为另一个类型，它会失败：

```go
runDecode(data, new(struct {
    Name []byte `gob:"name"`
}))
```

# 接口

关于该包的另一个注意事项是，如果您使用接口，它们的实现应该首先进行注册，使用以下函数：

```go
func Register(value interface{})
func RegisterName(name string, value interface{})
```

这将使该包了解指定的类型，并使我们能够在接口类型上调用解码。让我们首先定义一个接口及其实现，用于我们的结构：

```go

type Greeter interface {
    Greet(w io.Writer)
}

type Character struct {
    Name        string `gob:"name"`
    Surname     string `gob:"surname"`
    Job         string `gob:"job,omitempty"`
    YearOfBirth int    `gob:"year_of_birth,omitempty"`
}

func (c Character) Greet(w io.Writer) {
    fmt.Fprintf(w, "Hello, my name is %s %s", c.Name, c.Surname)
    if c.Job != "" {
        fmt.Fprintf(w, " and I am a %s", c.Job)
    }
}
```

如果我们尝试在没有`gob.Register`函数的情况下运行以下代码，会返回一个错误：

```go
gob: name not registered for interface: "main.Character"
```

但是如果我们注册了该类型，它就会像魅力一样工作。请注意，该数据是通过对包含`Character`结构的`Greeter`的指针进行编码而获得的：

```go
func main() {
    gob.Register(Greeter(Character{}))
    r := strings.NewReader("U\x10\x00\x0emain.Character" +
        "\xff\x81\x03\x01\x01\tCharacter\x01\xff\x82\x00" +
        "\x01\x04\x01\x04Name\x01\f\x00\x01\aSurname" +
        "\x01\f\x00\x01\x03Job\x01\f\x00\x01\vYearOfBirth" +
        "\x01\x04\x00\x00\x00\x1f\xff\x82\x1c\x01\x05John" +
        " \x01\aKirowan\x01\tprofessor\x00")
    var char Greeter
    if err := gob.NewDecoder(r).Decode(&char); err != nil {
        log.Fatalln(err)
    }
    char.Greet(os.Stdout)
}
```

# Proto

协议缓冲区是由谷歌制作的序列化协议。它是语言和平台中立的，开销很小，非常高效。其背后的想法是定义数据的结构一次，然后使用一些工具为应用程序的目标语言生成源代码。

# 结构

生成代码所需的主文件是`.proto`文件，它使用特定的语法。我们将专注于协议语法的最新版本`proto3`。

我们在第一行指定要使用的文件语法版本：

```go
syntax = "proto3";
```

可以使用`import`语句使用其他文件中的定义：

```go
import "google/protobuf/any.proto";
```

文件的其余部分包含消息（数据类型）和服务的定义。服务是用于定义 RPC 服务的接口：

```go
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

消息由它们的字段组成，服务由它们的方法组成。字段类型分为标量（包括各种整数、有符号整数、浮点数、字符串和布尔值）和其他消息。每个字段都有一个与之关联的数字，这是它的标识符，一旦选择就不应更改，以便与消息的旧版本保持兼容性。

使用`reserved`关键字可以防止一些字段或 ID 被重用，这对于避免错误或问题非常有用：

```go
message Foo {
  // lock field IDs
  reserved 2, 15, 9 to 11;
  // lock field names
  reserved "foo", "bar";
}
```

# 代码生成

为了从`.proto`文件生成代码，您需要`protoc`应用程序和官方的 proto 生成包：

```go
go get -u github.com/golang/protobuf/protoc-gen-go
```

安装的包带有`protoc-gen-go`命令；这使得`protoc`命令可以使用`--go_out`标志在所需的文件夹中生成 Go 源文件。Go 的 1.4 版本可以指定特殊注释以使用其`go generate`命令自动生成代码，这些注释以`//go:generate`开头，后跟命令，如下例所示：

```go
//go:generate protoc -I=$SRC_PATH --go_out=$DST_DIR source.proto
```

它使我们能够指定导入查找的源路径、输出目录和源文件。路径是相对于找到注释的包目录的，可以使用`go generate $pkg`命令调用。

让我们从一个简单的`.proto`文件开始：

```go
syntax = "proto3";

message Character {
    string name = 1;
    string surname = 2;
    string job = 3;
    int32 year_of_birth = 4;
}
```

让我们在相同的文件夹中创建一个带有用于生成代码的注释的 Go 源文件：

```go
package gen

//go:generate protoc --go_out=. char.proto
```

现在，我们可以生成`go`命令，它将生成一个与`.proto`文件相同名称和`.pb.go`扩展名的文件。该文件将包含`.proto`文件中定义的类型和服务的 Go 源代码：

```go
// Code generated by protoc-gen-go. DO NOT EDIT.
// source: char.proto
...
type Character struct {
  Name        string `protobuf:"bytes,1,opt,name=name"`
  Surname     string `protobuf:"bytes,2,opt,name=surname"`
  Job         string `protobuf:"bytes,3,opt,name=job" json:"job,omitempty"`
  YearOfBirth int32  `protobuf:"varint,4,opt,name=year_of_birth,json=yearOfBirth"`
}
```

# 编码

这个包允许我们使用`proto.Buffer`类型来编码`pb.Message`值。由`protoc`创建的类型实现了定义的接口，因此`Character`类型可以直接使用：

```go
var char = gen.Character{
    Name:        "George",
    Surname:     "Gammell Angell",
    YearOfBirth: 1834,
    Job:         "professor emeritus",
}
b := proto.NewBuffer(nil)
if err := b.EncodeMessage(&char); err != nil {
    log.Fatalln(err)
}
log.Printf("%q", b.Bytes())
```

生成的编码数据与其他编码相比几乎没有额外开销。

# 解码

解码操作也需要使用`proto.Buffer`方法和生成的类型来执行：

```go
b := proto.NewBuffer([]byte(
    "/\n\x06George\x12\x0eGammell Angell" +
    "\x1a\x12professor emeritus \xaa\x0e",
))
var char gen.Character
if err := b.DecodeMessage(&char); err != nil {
    log.Fatalln(err)
}
log.Printf("%+v", char)
```

# gRPC 协议

谷歌使用协议缓冲编码来构建名为**gRPC**的 Web 协议。它是一种使用 HTTP/2 建立连接和使用协议缓冲区来编组和解组数据的远程过程调用类型。

第一步是在目标语言中生成与服务器相关的代码。这将产生一个服务器接口和一个客户端工作实现。接下来，需要手动创建服务器实现，最后，目标语言将使实现能够在 gRPC 服务器中使用，然后使用客户端连接和与之交互。

`go-grpc`包中有不同的示例，包括客户端/服务器对。客户端使用生成的代码，只需要一个工作的 gRPC 连接到服务器，然后可以使用服务中指定的方法：

```go
conn, err := grpc.Dial(address, grpc.WithInsecure())
if err != nil {
    log.Fatalf("did not connect: %v", err)
}
defer conn.Close()
c := pb.NewGreeterClient(conn)

// Contact the server and print out its response
r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
```

完整的代码可在[grpc/grpc-go/blob/master/examples/helloworld/greeter_client/main.go](https://github.com/grpc/grpc-go/blob/master/examples/helloworld/greeter_client/main.go)找到。

服务器是客户端接口的实现：

```go
// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    log.Printf("Received: %v", in.Name)
    return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}
```

这个接口实现可以传递给生成的注册函数`RegisterGreeterServer`，连同一个有效的 gRPC 服务器，它可以使用 TCP 监听器来服务传入的连接：

```go
func main() {
    lis, err := net.Listen("tcp", port)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    s := grpc.NewServer()
    pb.RegisterGreeterServer(s, &server{})
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

完整的代码可在[grpc/grpc-go/blob/master/examples/helloworld/greeter_server/main.go](https://github.com/grpc/grpc-go/blob/master/examples/helloworld/greeter_server/main.go)找到。

# 摘要

在本章中，我们探讨了 Go 标准包和第三方库提供的编码方法。它们可以分为两大类。第一种是基于文本的编码方法，对人类和机器来说都易于阅读和编写。然而，它们的开销更大，而且往往比它们的对应的基于二进制的编码要慢得多。基于二进制的编码方法开销很小，但不易阅读。

在基于文本的编码中，我们发现了 JSON、XML 和 YAML。前两者由标准库处理，最后一个需要外部依赖。我们探讨了 Go 如何允许我们指定结构标签来改变默认的编码和解码行为，以及如何在这些操作中使用这些标签。然后，我们检查并实现了定义在编组和解组操作期间自定义行为的接口。有一些第三方工具可以让我们从 JSON 文件或 JSON 模式生成数据结构，JSON 模式是用于定义其他 JSON 文档结构的 JSON 文件。

XML 是另一种广泛使用的文本格式，HTML 就是基于它的。我们检查了 XML 语法和组成元素，然后展示了一种特定类型的文档，称为 DTD，用于定义其他 XML 文件的内容。我们学习了 XML 中编码和解码的工作原理，以及与 JSON 有关的`struct`标签的区别，这些标签允许我们为类型定义嵌套的 XML 元素，或者从属性中存储或加载字段。最后，我们介绍了基于文本的编码与第三方 YAML 包。

我们展示的第一个基于二进制的编码是 BSON，这是 JSON 的二进制版本，被 MongoDB 使用（由第三方包处理）。`gob`是另一种二进制编码方法，但它是 Go 标准库的一部分。我们了解到编码和解码以及涉及的接口，都是以标准包的方式工作的——类似于 JSON 和 XML。

最后，我们看了一下协议缓冲编码，如何编写`.proto`文件以及其 Go 代码生成用法，以及如何使用它对数据进行编码和解码。我们还介绍了 gRPC 编码的一个实际示例，利用这种编码来创建客户端/服务器应用程序。

在下一章中，我们将开始深入研究 Go 的并发模型，从内置类型开始——通道和 goroutine。

# 问题

1.  文本和二进制编码之间的权衡是什么？

1.  Go 默认情况下如何处理数据结构？

1.  这种行为如何改变？

1.  结构字段如何在 XML 属性中编码？

1.  需要什么操作来解码`gob`接口值？

1.  什么是协议缓冲编码？
