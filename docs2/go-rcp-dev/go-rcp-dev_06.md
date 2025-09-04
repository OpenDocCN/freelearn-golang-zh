

# 与泛型一起工作

经常发生的情况是，您编写了一个使用特定类型（例如整数）的值的函数来进行某些计算，但随着开发的进行，您突然需要使用另一种数据类型（例如 `float64`）来做同样的事情。因此，您复制粘贴第一个函数并修改它以具有不同的名称和数据类型。这种情况最明显和最著名的例子可能是容器数据类型，如映射和集合。您为整数值构建容器类型，然后为字符串做同样的事情，然后为结构体，依此类推。

泛型是一种在编译时使用代码模板进行代码复制粘贴的方式。首先，您创建一个函数模板（泛型函数）或数据类型模板（泛型类型）。通过提供类型来实例化泛型函数或类型。编译器会负责使用您提供的类型实例化模板，并检查实例化的泛型类型或函数是否可以与您提供的类型一起编译。

在本章中，您将学习如何使用泛型函数和数据类型处理常见场景：

+   泛型函数

    +   编写一个添加数字的泛型函数

    +   将约束声明为接口

    +   将泛型函数用作适配器和访问器

+   泛型类型

    +   编写一个类型安全的集合

    +   有序映射 -- 使用多个类型参数

# 泛型函数

一个泛型函数是一个函数模板，它接受类型作为参数。泛型函数必须为其参数的所有可能的类型赋值编译。泛型函数可以接受的数据类型由“类型约束”描述。我们将在本节中学习这些概念。

## 编写一个添加数字的泛型函数

一个很好的泛型示例是添加数字的函数。这些数字可以是各种整数或浮点数类型。在这里，我们将研究具有不同功能的几个配方。

### 如何做到这一点...

接受 `int` 和 `float64` 数字的一个泛型求和函数如下：

```go
func SumT int | float64 T {
  var result T
  for _, x := range values {
    result += x
  }
  return result
}
```

构造 `[T int | float64]` 定义了 `Sum` 函数的类型参数：

+   `T` 是类型名称。例如，如果您为 `int` 实例化 `Sum` 函数，那么 `T` 就是 `int`。

+   `int | float64` 表达式是 `T` 的类型约束。在这种情况下，它意味着“`T` 要么是 `int`，要么是 `float64`。”这个约束告诉编译器，`Sum` 函数只能实例化为 `int` 或 `float64` 值。

如我之前所解释的，泛型函数只是一个模板。例如，您不能声明一个函数变量并将其分配给 `Sum`，因为 `Sum` 不是一个真正的函数。以下语句为 `int` 实例化了 `Sum` 泛型函数：

```go
fmt.Println(Sumint)
```

对于许多情况，编译器可以推断类型参数，因此以下也是有效的。由于所有参数都是 `int` 值，编译器推断出这里的意思是 `Sum[int]`：

```go
fmt.Println(Sum(1,2,3))
```

但在以下情况下，实例化的函数是`Sum[float64]`，并且参数被解释为`float64`值：

```go
fmt.Println(Sumfloat64)
```

泛型函数必须对所有可能的`T`成功编译。在这种情况下，`T`可以是`int`或`float64`，因此函数体必须对`T`是`int`和`T`是`float64`有效。类型约束允许编译器产生有意义的编译时错误。例如，`[T int | float64 | big.Int]`约束无法编译，因为`result+=x`对`big.Int`不起作用。

`Sum`函数对从`int`或`float64`派生的类型不起作用，例如：

```go
type ID int
```

即使`ID`是`int`类型，`Sum[ID]`也会导致编译错误，因为`ID`是一个新类型。要包含从`int`派生的所有类型，在约束中使用`~int` - 例如：

```go
func SumT ~int | ~float64 T{...}
```

此声明将处理从`int`和`float64`派生的所有类型。

## 将约束声明为接口

在声明新函数时重复约束并不实用。相反，您可以在接口中定义它们，作为类型列表或方法列表。

### 如何实现...

Go 接口指定了一个方法集。Go 泛型实现扩展了此定义，以便当用作约束时，接口定义类型集。这需要对基本类型进行一些修改，因为基本类型（如`int`）没有方法。因此，在接口作为约束时，有两种语法：

1.  类型列表指定了可以替代类型参数的类型列表。例如，以下`UnsignedInteger`约束接受所有无符号整数类型以及从无符号整数派生的所有类型：

    ```go
    type UnsignedInteger interface {
      ~uint8 | ~uint16 | ~uint32 | ~uint64
    }
    ```

1.  方法集指定了可接受类型必须实现的方法。以下`Stringer`约束接受所有具有`String()` `string`方法的类型：

    ```go
    type Stringer interface {
      String() string
    }
    ```

这些约束可以组合。例如，以下`UnsignedIntegerStringer`约束接受从无符号整数类型派生的类型，并且具有`String()` `string`方法：

```go
type UnsignedIntegerString interface {
  UnsignedInteger
  Stringer
}
```

`Stringer`接口既可以用作约束，也可以用作接口。`UnsignedInteger`和`UnsignedIntegerString`接口只能用作约束。

## 将泛型函数用作访问器和适配器

泛型函数为类型安全的访问器和类型适配器提供了实用解决方案。例如，使用常量值初始化`*int`变量需要声明一个临时值，这可以通过泛型函数简化。这个配方包括几个这样的访问器和适配器。

### 如何实现...

此泛型函数将任意值转换为指针：

```go
func ToPtrT any *T {
  return &value
}
```

这可以用来初始化指针而不使用临时变量：

```go
type UpdateRequest struct {
  Name *string
  ...
}
...
request:=UpdateRequest {
  Name:ToPtr("test"),
}
```

同样，这个泛型函数可以从任意值创建切片：

```go
func ToSliceT any []T {
        return []T{value}
}
func main() {
  fmt.Println(ToSlice(1))
  // Prints an int slice: [1]
}
```

以下泛型函数返回切片的最后一个元素：

```go
func LastT any (T, bool) {
  if len(slice) == 0 {
    var zero T
    return zero, false
  }
  return slice[len(slice)-1], true
}
```

如果切片为空，则返回`false`。

以下泛型函数可以用来适配返回值和错误的函数，以便在只接受值的上下文中使用。如果存在错误，函数会引发恐慌：

```go
func MustT any T {
  if err != nil {
    panic(err)
  }
  return value
}
```

这将 `f() (T, error)` 函数适配为 `Must(f()) T`。

## 从泛型函数返回零值

如我之前所说，一个泛型函数必须对所有允许的类型约束进行编译。这可能在创建零值时引起麻烦。

### 如何做...

要创建参数化类型的零值，只需声明一个变量：

```go
func Search[T []E, E comparable](slice T,value E) (E, bool) {
  for _,v:=range slice {
    if v==value {
      return v,true
    }
  }
  // Declare a zero value like this
  var zero E
  return zero, false
}
```

## 在泛型参数上使用类型断言

有时，根据泛型函数中值的类型，你需要做不同的事情。这需要类型断言或类型选择器 – 两者都适用于接口。然而，没有保证函数会为接口实例化。这个配方展示了你如何实现这一点。

### 如何做...

假设你有一个处理整数不同的泛型函数：

```go
func PrintT any {
  // The following does not work because value is not necessarily an 
  // interface{}.
  if intValue, ok:=value.(int); ok {
    ...
  } else {
    ...
  }
}
```

要使这生效，你必须确保 `value` 是一个接口：

```go
func PrintT any {
  // Convert value to an interface
  valueIntf := any{value)
  if intValue, ok:=valueIntf.(int); ok {
    // Value is an integer
  } else {
    // Value is not an integer
  }
}
```

同样的想法也适用于类型选择器：

```go
func PrintT any {
  switch v:=any(value).(type) {
  case int:
    // Value is an integer
  default;
    // Value is not an integer
  }
}
```

# 泛型类型

泛型函数的语法自然扩展到泛型类型。泛型类型也有相同的类型参数和约束，并且该类型的每个方法也隐式具有与类型本身相同的参数。

## 编写类型安全的集合

可以使用 `map[T]struct{}` 实现一个类型安全的集合。需要注意的一件事是 `T` 不能是任何类型。只有可比较的类型可以作为映射键，并且有一个预定义的约束来满足这一需求。

### 如何做...

1.  使用 `map` 声明一个参数化集合类型：

    ```go
    type Set[T comparable] map[T]struct{}
    ```

1.  使用相同的类型参数（s）声明类型的函数。在声明方法时，你必须通过名称引用类型参数：

```go
// Has returns if the set has the given value
func (s Set[T]) Has(value T) bool {
     _, exists := s[value]
     return exists
}
// Add adds values to s
func (s Set[T]) Add(values ...T) {
     for _, v := range values {
          s[v] = struct{}{}
     }
}
// Remove removes values from s
func (s Set[T]) Remove(values ...T) {
     for _, v := range values {
          delete(s, v)
     }
}
```

1.  如果需要，为新的类型创建一个泛型构造函数：

```go
// NewSet creates a new set
func NewSet[T comparable]() Set[T] {
     return make(Set[T])
}
```

1.  实例化类型以使用它：

```go
stringSet := NewSet[string]()
```

注意 `NewSet` 函数使用 `string` 类型参数的显式实例化。编译器无法推断你指的是什么类型，所以你必须明确写出 `NewSet[string]()`。然后编译器实例化 `Set[string]` 类型，这也实例化了该类型的所有方法。

## 有序映射 – 使用多个类型参数

这种有序映射的实现允许你使用切片与映射的组合来保持添加到映射中的元素的顺序。

### 如何做...

1.  定义一个包含两个类型参数的结构体：

```go
type OrderedMap[Key comparable, Value any] struct {
     m     map[Key]Value
     slice []Key
}
```

由于 `Key` 将用作映射键，它必须是 `comparable` 的。对值类型没有约束。

为类型定义方法。现在方法使用 `Key` 和 `Value` 同时声明：

```go
// Add key:value to the map
func (m *OrderedMap[Key, Value]) Add(key Key, value Value) {
     _, exists := m.m[key]
     if exists {
          m.m[key] = value
     } else {
          m.slice = append(m.slice, key)
          m.m[key] = value
     }
}
// ValueAt returns the value at the given index
func (m *OrderedMap[Key, Value]) ValueAt(index int) Value {
     return m.m[m.slice[index]]
}
// KeyAt returns the key at the given index
func (m *OrderedMap[Key, Value]) KeyAt(index int) Key {
     return m.slice[index]
}
// Get returns the value corresponding to the key, and whether or not
// key exists
func (m *OrderedMap[Key, Value]) Get(key Key) (Value, bool) {
     v, bool := m.m[key]
     return v, bool
}
```

小贴士

接收器的类型参数是通过位置匹配的，而不是通过名称。换句话说，你可以这样定义一个方法：

`func (m *OrderedMap[K, V]) ValueAt(index int) V {`

`return m.m[m.slice[index]]`

`}`

在这里，`K` 代表 `Key`，而 `V` 代表 `Value`。

1.  如果需要，定义一个构造函数泛型函数：

```go
func NewOrderedMap[Key comparable, Value any]() *OrderedMap[Key, Value] {
     return &OrderedMap[Key, Value]{
          m:     make(map[Key]Value),
          slice: make([]Key, 0),
     }
}
```

小贴士

在这种情况下需要一个构造函数，因为我们想在泛型结构体中初始化映射。每次想要向容器中添加内容时检查空映射很有诱惑力。你必须在拥有零值即可使用的容器类型带来的便利和每次添加内容时检查空映射所付出的性能代价之间做出选择。
