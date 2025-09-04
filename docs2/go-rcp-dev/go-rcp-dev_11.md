# 11

# 与 JSON 一起工作

JSON 是 JavaScript 对象表示法的缩写。它是一种流行的数据交换格式，因为 JSON 对象与结构化类型（Go 中的`struct`）非常相似，并且它是基于文本的编码，使得编码后的数据可读。它支持数组、对象（键值对）以及相对较少的基本类型（字符串、数字、布尔值和`null`）。这些特性使得 JSON 成为一个相对容易处理的格式。

编码是指将数据元素转换成一系列字节的进程。当你使用 JSON 编码（或序列化）数据元素时，你会在遵循 JSON 语法规则的同时创建这些数据元素的文字表示。相反的过程，解码（或反序列化）将 JSON 值分配给 Go 对象。编码过程是有损的：你必须将数据值描述为文本，而对于复杂的数据类型来说，这并不总是显而易见的。当你解码这样的数据时，你必须知道如何解释文本表示，以便你能正确地解析 JSON 表示。

在本章中，我们将首先查看基本数据类型的编码和解码。然后我们将查看一些处理更复杂的数据类型和用例的食谱。在实现自己的解决方案时，你应该将这些食谱作为指南。这些食谱展示了特定用例的解决方案，你可能需要根据你的具体需求对其进行调整。

本章包括以下食谱：

+   结构体的编码

+   处理嵌套结构

+   不定义结构体的编码

+   结构体的解码

+   使用接口、映射和切片进行解码

+   解码数字的其他方式

+   自定义数据类型的序列化和反序列化

+   对象键的自定义序列化和反序列化

+   动态字段名

+   多态数据结构

+   流式传输 JSON 数据

## 序列化和反序列化基础

标准库中的`encoding/json`包提供了方便的函数和约定来编码/解码 JSON 数据。

# 结构体的编码

Go 结构体类型通常被编码为 JSON 对象。本节展示了处理数据类型编码的标准库工具。

## 如何做...

1.  使用`json`标签注释结构体字段及其 JSON 键：

```go
type Config struct {
  Version   string `json:"ver"`  // Encoded as "ver"
  Name      string               // Encoded as "Name"
  Type      string `json:"type,omitempty"` // Encoded as "type",
                                           // and will be omitted if 
                                           // empty
  Style     string `json:"-"`    // Not encoded
  value     string               // Unexported field, not encoded
  kind      string `json:"kind"` // Unexported field, not encoded
}
```

1.  使用`json.Marshal`函数将 Go 数据对象编码为 JSON。标准库为基本类型使用以下约定：

| **Go 声明** | **值** | **JSON 输出** |
| --- | --- | --- |
| `NumberValue` `int json:”num”` | `0` | `“``num”: 0` |
| `NumberValue *``int json:”num”` | `nil` | `“``num”: null` |
| `NumberValue *``int json:”num,omitempty”` | `nil` | omitted |
| `BoolValue` `bool json:”bvalue”` | `true` | `“``bvalue”: true` |
| `BoolValue *``bool json:”bvalue”` | `nil` | `“``bvalue”: null` |
| `BoolValue *``bool json:”bvalue,omitempty”` | `nil` | omitted |
| `StringValue` `string json:”svalue”` | `“``str”` | `“``svalue”:”str”` |
| `StringValue` `string json:”svalue”` | `“”` | `“``svalue”:””` |
| `StringValue` `string json:”svalue,omitempty”` | `“``str”` | `“``svalue”:”str”` |
| `StringValue string json:"svalue,omitempty"` | `""` | 被省略 |
| `StringValue *```gostring `json:”svalue”`` | `nil` | `“``svalue”: null` |
| `StringValue *```string `json:”svalue,omitempty”`` | `nil` | 被省略 |

+   `struct` 和 `map` 类型被编码为 JSON 对象

+   切片和数组类型被编码为 JSON 数组

+   如果一个类型实现了 `json.Marshaler` 接口，则调用变量的实例的 `json.Marshaler.MarshalJSON` 方法来编码数据

+   如果一个类型实现了 `encoding.TextMarshaler` 接口，则值将被编码为 JSON 字符串，字符串值是从值的 `encoding.TextMarshaler.MarshalText` 方法中获得的

+   其他任何内容都将因 `UnsupportedValueError` 而失败

提示

只有结构体类型的导出字段可以被编码。

提示

如果结构体字段没有 JSON 标签，其 JSON 对象键将与字段名相同。

考虑以下代码段：

```go
type Config struct {
  Version   string `json:"ver"`  // Encoded as "ver"
  Name      string               // Encoded as "Name"
  Type      string `json:"type,omitempty"` // Encoded as "type",
                                           // and will be omitted if 
                                           // empty
  Style     string `json:"-"`    // Not encoded
  value     string               // Unexported field, not encoded
  kind      string `json:"kind"` // Unexported field, not encoded
}
...
cfg := Config{
     Version: "1.1",
     Name:    "name",
     Type:    "example",
     Style:   "json",
     value:   "example config value",
     kind:    "test",
}
data, err := json.Marshal(cfg)
fmt.Println(string(err))
```

这将打印以下内容：

```go
{"ver":"1.1","Name":"name","type":"example"}
```

提示

编码 JSON 对象中字段的顺序与字段声明的顺序相同。

# 处理嵌套结构体

结构体类型的字段将被编码为 JSON 对象。如果存在嵌套的结构体，则编码器有两个选择：将嵌套的结构体编码在与封装结构体相同的级别，或者作为一个新的 JSON 对象。

## 如何实现...

1.  使用 JSON 标签命名封装结构体字段和嵌套结构体字段：

    ```go
    type Enclosing struct {
         Field string `json:"field"`
         Embedded
    }
    type Embedded struct {
         Field string `json:"embeddedField"`
    }
    ```

1.  使用 `json.Marshal` 将结构体编码为 JSON 对象：

    ```go
    enc := Enclosing{
         Field: "enclosing",
         Embedded: Embedded{
              Field: "embedded",
         },
    }
    data, err = json.Marshal(enc)
    // {"field":"enclosing","embeddedField":"embedded"}
    ```

1.  在嵌套结构体中添加 `json` 标签将创建一个嵌套的 JSON 对象：

    ```go
    type Enclosing struct {
         Field string `json:"field"`
         Embedded `json:"embedded"`
    }
    type Embedded struct {
         Field string `json:"embeddedField"`
    }
    ...
    enc := Enclosing{
         Field: "enclosing",
         Embedded: Embedded{
              Field: "embedded",
         },
    }
    data, err = json.Marshal(enc)
    // {"field":"enclosing","embedded":{"embeddedField":"embedded"}}
    ```

# 不定义结构体的编码

基本数据类型、切片和映射可以用来编码 JSON 数据。

## 如何实现...

+   使用映射来表示 JSON 对象：

    ```go
    config:=map[string]any{
      "ver": "1.0",
      "Name": "config",
      "type": "example",
      }
    data, err:=json.Marshal(config)
    // `{"ver":"1.0","Name":"config","type":"example"}`
    ```

+   使用切片来表示 JSON 数组：

    ```go
    numbersWithNil:=[]any{ 1, 2, nil, 3 }
    data, err:=json.Marshal(numbersWithNil)
    // `[1,2,null,3]`
    ```

+   将所需的 JSON 结构与 Go 等效项匹配：

    ```go
    configurations:=map[string]map[string]any {
      "cfg1": {
         "ver": "1.0",
         "Name": "config1",
      },
      "cfg2": {
         "ver": "1.1",
         "Name" : "config2",
     },
    }
    data, err:=json.Marshal(configurations)
    // {"cfg1":{"Name":"config1","ver":"1.0"},
    "cfg2":{"Name":"config2","ver":"1.1"}}`
    ```

# 解码结构体

在 JSON 中编码 Go 数据对象是一个相对简单的任务：定义良好的数据类型和语义被转换为表达性较低的表现形式，通常会导致一些信息丢失。例如，整数变量和 `float64` 变量可能被编码为相同的输出。因此，解码 JSON 数据通常更困难。

## 如何实现...

1.  使用 JSON 标签将 JSON 键映射到结构体字段。

1.  使用 `json.Unmarshal` 函数将 JSON 数据解码为 Go 数据对象。标准库使用以下约定来处理基本类型：

| **JSON 输入** | **Go 类型** | **结果** |
| --- | --- | --- |
| `"strValue"` | `string` | `"strValue"` |
| `1 (number)` | `int` | `1` |
| `1.2 (number)` | `int` | `error` |
| `1.2 (number)` | `float64, float32` | `1.2` |
| `true` | `bool` | `true` |
| `null` | `string` | 变量保持未修改 |
| `null` | `int` | 变量保持未修改 |
| `"strValue"` | `*string` | `"strValue"` |
| `null` | `*string` | `nil` |
| `1` | `*int` | `1` |
| `null` | `*int` | `nil` |
| `true` | `*bool` | `true` |
| `null` | `*bool` | `nil` |

如果 Go 类型是 `interface{}`，标准库将使用以下约定创建对象：

| **JSON 输入** | **结果** |
| --- | --- |
| `"strValue"` | `"strValue"` |
| `1` | `float64(1)` |
| `1.2` | `float64(1.2)` |
| `true` | `true` |
| `null` | `nil` |
| JSON 对象 | `map[string]any` |
| JSON 数组 | `[]any` |

+   如果目标 Go 类型实现了 `json.Unmarshaler` 接口，则调用 `json.Unmarshal.UnmarshalJSON` 来解码数据。此操作可能需要根据需要创建目标类型的新实例。

+   如果目标 Go 类型实现了 `encoding.TextUnmarshaler` 接口，并且输入是引号中的 JSON 字符串，则调用 `encoding.TextUnmarshaler.UnmarshalText` 来解码值。

+   其他任何内容都将因 `UnsupportedValueError` 而失败。

提示

如果 JSON 输入包含各种数值类型的值，数值可能会导致混淆。例如，如果 JSON 数值被反序列化为 `int` 值，如果 JSON 数据可以表示为整数，则可以正常工作，但如果 JSON 数据包含浮点值，则将失败。

提示

JSON 解码器永远不会更改结构体的非导出字段。解码器使用反射，并且只有导出字段可以通过反射访问。

提示

没有匹配 Go 字段的 JSON 字段将被忽略。

# 使用接口、映射和切片进行解码

当将 Go 值解码为 JSON 时，Go 值类型决定了 JSON 编码的方式。JSON 没有像 Go 那样丰富的类型系统。有效的 JSON 类型是字符串、数字、布尔值、对象、数组和 null。当你将 JSON 数据解码到 Go 结构体时，仍然是 Go 类型系统决定了如何解释 JSON 数据。但是，当你将 JSON 解码到 `interface{}` 时，情况就改变了。现在 JSON 数据决定了如何构建 Go 值，这有时会导致意外的结果。

## 如何操作...

要将 JSON 数据反序列化到接口中，请使用以下方法：

```go
var output interface{}
err:=json.Unmarshal(jsonData,&output)
```

这将根据以下翻译规则创建基于以下翻译规则的对象树：

| **JSON** | **Go** |
| --- | --- |
| 对象 | `map[string]interface{}` |
| 数组 | `[]interface{}` |
| 数字 | `float64` |
| 布尔值 | `bool` |
| 字符串 | `string` |
| null | `nil` |

# 解码数字的其他方式

当解码到 `interface{}` 时，JSON 数值被转换为 `float64`。这并不总是期望的结果。你可以使用 `json.Number` 代替。

## 如何操作...

使用 `json.Decoder` 并设置 `UseNumber`：

```go
var output interface{}
decoder:=json.NewDecoder(strings.NewReader(`[1.1,2,3,4.4]`))
// Tell the decoder to use json.Number instead of float64
decoder.UseNumber()
err:=decoder.Decode(&output)
// [1.1 2 3 4.4]
```

在前面的示例中，`output` 的每个元素都是 `json.Number` 的实例。你可以根据需要将其转换为 `int`、`float64` 或 `big.Int`。

# 处理缺失和可选值

通常你必须处理缺少字段的 JSON 输入，并生成省略空字段的 JSON。本节提供了处理这些场景的配方。

# 编码时省略空字段

在 JSON 编码中省略空字段通常可以节省空间并使 JSON 更易于阅读。然而，“空”的含义应该是明确的。

## 如何操作...

使用 `,omitempty` JSON 标签省略空字符串值、零整数/浮点值、零 `time.Duration` 值和 `nil` 指针值。

`,omitempty` 标签对 `time.Time` 值不起作用。使用 `*time.Time` 并将其设置为 `nil` 以省略空时间值：

```go
type Config struct {
    ...
     Type       string `json:"type,omitempty"`
     IntValue   int     `json:"intValue,omitempty"`
     FloatValue float64 `json:"floatValue,omitempty"`
     When       *time.Time    `json:"when,omitempty"`
     HowLong    time.Duration `json:"howLong,omitempty"`
}
```

有时区分空字符串和 null 字符串很重要。在 JavaScript 和 JSON 中，`null`是字符串的有效值。如果是这种情况，请使用 `*string`：

```go
type Config struct {
  Value  *string `json:"value,omitempty"`
  ...
}
...
emptyString := ""
emptyValue := Config {
   Value: &emptyString,
}
// JSON output: { "value": "" }
nullValue := Config {
   Value: nil,
}
// JSON output: {}
```

# 解码时处理缺失字段

有几种用例，开发者必须处理不包含所有数据字段的稀疏 JSON 数据。例如，部分更新 API 调用可能接受一个仅包含应更新的字段而不修改任何未指定数据字段的 JSON 对象。在这种情况下，识别哪些字段被提供变得很重要。然后还有适合为缺失字段假设默认值的用例。

## 如何做到这一点...

如果你想确定在 JSON 输入中指定了哪些字段，请使用指针字段。输入中缺失的任何字段都将保持为 `nil`。

要为缺失字段提供默认值，在反序列化之前将这些字段初始化为其默认值：

```go
type APIRequest struct {
   // If type is not specified, it will be nil
   Type    *string `json:"type"`
   // There will be a default value for seq
   Seq     int     `json:"seq"`
   ...
}
func handler(w http.ResponseWriter,r *http.Request) {
  data, err:=io.ReadAll(r.Body)
  if err!=nil {
     http.Error(w, "Bad request",http.StatusBadRequest)
     return
  }
  req:=APIRequest{
     Seq: 1,  // Set the default value
  }
  if err:=json.Unmarshal(data, &req); err!=nil {
     http.Error(w, "Bad request", http.StatusBadRequest)
     return
  }
  // Check which fields are provided
  if req.Type!=nil {
     ...
  }
  // If seq is provided in the input, req.Seq will be set to that 
  // value. Otherwise, it will be 1.
  if req.Seq==1 {
    ...
  }
}
```

## 自定义 JSON 编码/解码

有时某些数据结构的 JSON 编码与其在程序中的表示不匹配。当这种情况发生时，你必须自定义如何将某个特定数据元素编码为 JSON 或从 JSON 中解码。

# 序列化/反序列化自定义数据类型

当你有数据元素，其 JSON 表示需要程序化生成时，请使用这些食谱。

## 如何做到这一点...

要控制数据对象在 JSON 中的编码方式，实现 `json.Marshaler` 接口：

```go
// TypeAndID is encoded to JSON as type:id
type TypeAndID struct {
  Type string
  ID int
}
// Implementation of json.Marshaler
func (t TypeAndID) MarshalJSON() (out []byte, err error) {
  s := fmt.Sprintf(`"%s:%d"`,t.Type,t.ID)
  out=[]byte(s)
  return
}
```

要控制数据对象从 JSON 中解码的方式，实现 `json.Unmarshaler` 接口：

提示

解码器必须有一个指针接收器。

```go
// Implementation of json.Unmarshaler. Note the pointer receiver
func (t *TypeAndID) UnmarshalJSON(in []byte) (err error) {
    if len(in)<2 || in[0] != '"' || in[len(in)-1] != '"' {
        err = ErrInvalidTypeAndID
        return
    }
    in = in[1 : len(in)-1]
    parts := strings.Split(string(in), ":")
    if len(parts) != 2 {
        err = ErrInvalidTypeAndID
        return
     }
    // The second part must be a valid integer
    t.ID, err = strconv.Atoi(parts[1])
    if err != nil {
        return
    }
    t.Type = parts[0]
    return
}
```

# 自定义对象键的序列化/反序列化

映射作为 JSON 对象进行序列化/反序列化。但如果你有一个键类型不是字符串类型的映射，你如何将其序列化/反序列化为 JSON？

## 如何做到这一点...

解决方案取决于键的确切类型：

1.  从字符串或整数类型派生的键类型的映射可以采用标准库方法进行序列化/反序列化：

    ```go
    type Key int64
    func main() {
         var m map[Key]int
         err := json.Unmarshal([]byte(`{"123":123}`), &m)
        if err!=nil {
           panic(err)
        }
         fmt.Println(m[123]) // Prints 123
    }
    ```

1.  如果映射键在序列化/反序列化时需要额外的处理，实现 `encoding.TextMarshaler` 和 `encoding.TextUnmarshaler` 接口：

    ```go
    // Key is an uint that is encoded as an hex strings for JSON key
    type Key uint
    func (k *Key) UnmarshalText(data []byte) error {
         v, err := strconv.ParseInt(string(data), 16, 64)
         if err != nil {
              return err
         }
         *k = Key(v)
         return nil
    }
    func (k Key) MarshalText() ([]byte, error) {
         s := strconv.FormatUint(uint64(k), 16)
         return []byte(s), nil
    }
    func main() {
         input := `{
        "13AD": "5037",
        "3E22": "15906",
        "90A3": "37027"
      }`
         var data map[Key]string
         if err := json.Unmarshal([]byte(input), &data); err != nil {
              panic(err)
         }
         fmt.Println(data)
         d, err := json.Marshal(map[Key]any{
              Key(123): "123",
              Key(255): "255",
         })
         if err != nil {
              panic(err)
         }
         fmt.Println(string(d))
    }
    ```

## 动态字段名

有时字段名（对象键）不是固定的。例如，一个 API 可能更喜欢以 JSON 对象的形式返回对象的列表，其中每个对象的唯一标识符是键。在这种情况下，无法在结构体中使用 `json` 标签。

## 如何做到这一点...

使用 `map[string]ValueType` 来表示具有动态字段名的对象：

```go
type User struct {
     Name string `json:"name"`
     Type string `json:"type"`
}
type Users struct {
     Users map[string]User `json:"users"`
}
func main() {
     input := `{
  "users": {
      "abb64dfe-d4a8-47a5-b7b0-7613fe3fd11f": {
         "name": "John",
         "type": "admin"
      },
      "b158161c-0588-4c67-8e4b-c07a8978f711": {
         "name": "Amy",
         "type": "editor"
      }
   }
  }`
     var users Users
     if err := json.Unmarshal([]byte(input), &users); err != nil {
          panic(err)
     }
}
```

## 多态数据结构

多态数据结构可以是几种不同类型中的一种，这些类型共享一个公共接口。实际类型是在运行时确定的。对于运行时对象，Go 的类型系统通过使用这些字段确保类型安全的操作。使用接口，多态对象可以轻松地序列化为 JSON。当你需要反序列化一个多态 JSON 对象时，就会出现问题。在这个菜谱中，我们将探讨实现这一目标的一种方法。

# 使用两次遍历进行自定义反序列化

第一次遍历会反序列化判别器字段，而将输入的其余部分留待未处理。根据判别器，构建并反序列化对象的具体实例。

## 如何做...

1.  在本节中，我们将使用一个示例 `Key` 结构。`Key` 结构包含不同类型的加密公钥，其类型在 `Type` 字段中给出：

    ```go
    type KeyType string
    const (
         KeyTypeRSA     = "rsa"
         KeyTypeED25519 = "ed25519"
    )
    type Key struct {
         Type KeyType          `json:"type"`
         Key  crypto.PublicKey `json:"key"`
    }
    ```

1.  按照惯例定义数据结构的 JSON 标签。大多数多态结构可以不使用自定义序列化器进行序列化，因为在序列化期间已知对象的运行时类型。

    定义另一个与原始结构相同的结构，将动态类型部分替换为 `json.RawMessage` 类型字段：

    ```go
    type keyUnmarshal struct {
         Type KeyType         `json:"type"`
         Key  json.RawMessage `json:"key"`
    }
    ```

1.  为原始结构创建一个反序列化器。在这个反序列化器中，首先将输入反序列化到步骤 2 中创建的结构实例：

    ```go
    func (k *Key) UnmarshalJSON(in []byte) error {
         var key keyUnmarshal
         err := json.Unmarshal(in, &key)
         if err != nil {
              return err
         }
    ```

1.  使用类型判别器字段，决定如何解码动态部分。以下示例使用工厂来获取特定类型的反序列化器：

    ```go
         k.Type = key.Type
         unmarshaler := KeyUnmarshalers[key.Type]
         if unmarshaler == nil {
              return ErrInvalidKeyType
         }
    ```

1.  将动态类型部分（它是一个 `json.RawMessage`）反序列化到正确类型的变量实例中：

    ```go
         k.Key, err = unmarshaler(key.Key)
       if err != nil {
            return err
       }
       return nil
    }
    ```

    工厂是一个简单的映射，它知道不同类型键的反序列化器：

    ```go
    var (
         KeyUnmarshalers = map[KeyType]func(json.RawMessage) 
         (crypto.PublicKey, error){}
    )
    func RegisterKeyUnmarshaler(keyType KeyType, unmarshaler func(json.RawMessage) (crypto.PublicKey, error)) {
         KeyUnmarshalers[keyType] = unmarshaler
    }
    ...
    RegisterKeyUnmarshaler(KeyTypeRSA, func(in json.RawMessage) (crypto.PublicKey, error) {
         var key rsa.PublicKey
         if err := json.Unmarshal(in, &key); err != nil {
              return nil, err
         }
         return &key, nil
    })
    RegisterKeyUnmarshaler(KeyTypeED25519, func(in json.RawMessage) (crypto.PublicKey, error) {
         var key ed25519.PublicKey
         if err := json.Unmarshal(in, &key); err != nil {
              return nil, err
         }
         return &key, nil
    })
    ```

这是一个可扩展的工厂框架，可以在构建时初始化额外的反序列化器。只需为对象类型创建一个反序列化函数，并使用前面的 `RegisterKeyUnmarshaler` 函数注册它以支持新的键类型。

小贴士

注册此类功能的一种常见方式是使用包的 `init()` 函数。当你导入该包时，该包支持的反序列化类型将被注册。

## 流式传输 JSON 数据

当你需要高效地处理大量数据时，你应该考虑流式传输数据而不是一次性处理整个数据集。本节描述了一些流式传输 JSON 数据的方法。

# 流式传输对象数组

如果你有一个生成器（goroutine、数据库游标等）产生数据元素，并且你想将这些元素作为 JSON 数组流式传输而不是在序列化之前存储所有内容，这个菜谱就很有用。

## 如何做...

1.  创建一个生成器。这可以是

    +   一个通过通道发送数据元素的 goroutine，

    +   一个类似游标的对象，包含一个 `Next()` 方法，

    +   或其他数据生成器。

1.  创建一个 `json.Encoder` 实例，它使用 `io.Writer` 表示目标。目标可以是文件、标准输出、缓冲区、网络连接等。

1.  为数组写入开始分隔符，即`[`。

1.  如果需要，在数据元素前加上逗号进行编码。

1.  写入数组的结束分隔符，即`]`。

以下示例假设存在一个生成器 goroutine 将`Data`实例写入`input`通道。当没有更多的`Data`实例时，生成器会关闭通道。在这里，我们假设`Data`是 JSON 可序列化的：

```go
func stream(out io.Writer, input <-chan Data) error {
     enc := json.NewEncoder(out)
     if _, err := out.Write([]byte{'['}); err != nil {
          return err
     }
     first := true
     for obj := range input {
          if first {
               first = false
          } else {
               if _, err := out.Write([]byte{','}); err != nil {
                    return err
               }
          }
          if err := enc.Encode(obj); err != nil {
               return err
          }
     }
     if _, err := out.Write([]byte{']'}); err != nil {
          return err
     }
     return nil
}
```

# 解析对象数组

如果你有一个提供对象数组的 JSON 数据源，你可以解析这些元素并使用`json.Decoder`处理它们。

## 如何实现...

1.  创建从输入流读取的`json.Decoder`。

1.  使用`json.Decoder.Token()`解析数组开始分隔符（`[`）。

1.  解码数组中的每个元素，直到解码失败。

1.  当解码失败时，你必须确定是流结束了，还是确实存在错误。为了检查这一点，使用`json.Decoder.Token()`读取下一个标记。如果成功读取下一个标记，并且它是数组结束分隔符`]`，那么流解析成功结束。否则，输入数据中存在错误。

以下示例假设`json.Decoder`已经构建好，用于从输入流中读取。输出存储在切片中。或者，可以在解析元素时处理输出，或者将每个元素通过通道发送到处理 goroutine：

```go
func parse(input *json.Decoder) (output []Data, err error) {
     // Parse the array beginning delimiter
     var tok json.Token
     tok, err = input.Token()
     if err != nil {
          return
     }
     if tok != json.Delim('[') {
          err = fmt.Errorf("Array begin delimiter expected")
          return
     }
     // Parse array elements using Decode
     for {
          var data Data
          err = input.Decode(&data)
          if err != nil {
               // Decode failed. Either there is an input error, or
               // we are at the end of the stream
               tok, err = input.Token()
               if err != nil {
                    // Data error
                    return
               }
               // Are we at the end?
               if tok == json.Delim(']') {
                    // Yes, there is no error
                    err = nil
                    break
               }
          }
          output = append(output, data)
     }
     return
}
```

# 其他流式传输 JSON 的方式

还有其他方式来流式传输 JSON：

+   连接 JSON 简单地连续写入 JSON 对象

+   换行符分隔的 JSON 将每个 JSON 对象作为单独的一行写入

+   记录分隔符分隔的 JSON 使用特殊的记录分隔符字符 0x1E，并在每个 JSON 对象之间可选地添加换行符

+   长度前缀 JSON 将每个 JSON 对象的字符串长度作为十进制数字进行前缀

所有这些都可以使用`json.Decoder`和`json.Encoder`进行读取和写入。一个简单的 JSON 流式传输包可以在以下位置找到：[`github.com/bserdar/jsonstream`](https://github.com/bserdar/jsonstream)。

## 安全考虑

无论何时从你的应用程序外部接受数据（用户输入的数据、API 调用、读取文件等），你都必须关注恶意输入。JSON 输入相对安全，因为 JSON 解析器不执行数据扩展，如 YAML 或 XML 解析器所做的那样。然而，在处理 JSON 数据时，你仍然需要考虑一些事情。

## 如何实现...

+   在接受第三方 JSON 输入时限制数据量。不要盲目使用`io.ReadAll`或`json.Decode`：

    ```go
    const MessageSizeLimit = 10240
    func handler(w http.ResponseWriter, r *http.Request) {
      reader:=http.MaxBytesReader(w,r.Body,MessageSizeLimit)
      data, err := io.ReadAll(reader)
      if errors.Is(err,&http.MaxBytesError{}) {
        // If this happens, error is already sent.
        return
      }
      ...
    }
    ```

+   基于从第三方输入读取的数据，始终为资源分配提供上限。例如，如果你正在读取一个长度前缀的 JSON 流，其中每个 JSON 对象都由其长度前缀，不要分配一个`[]byte`来存储下一个对象。如果长度太大，则拒绝输入。
