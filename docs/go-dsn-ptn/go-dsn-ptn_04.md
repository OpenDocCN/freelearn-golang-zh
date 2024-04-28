# 第四章：结构模式 - 代理，外观，装饰器和享元设计模式

通过本章，我们将完成结构模式。我们将最复杂的一些模式留到最后，以便您更加熟悉设计模式的机制和 Go 语言的特性。

在本章中，我们将致力于编写一个用于访问数据库的缓存，一个用于收集天气数据的库，一个带有运行时中间件的服务器，并讨论通过在类型值之间保存可共享状态来节省内存的方法。

# 代理设计模式

我们将以代理模式开始最终章节。这是一个简单的模式，可以提供有趣的功能和可能性，而且只需很少的努力。

## 描述

代理模式通常包装一个对象，以隐藏其某些特征。这些特征可能是它是一个远程对象（远程代理），一个非常重的对象，例如非常大的图像或千兆字节数据库的转储（虚拟代理），或者是一个受限制的访问对象（保护代理）。

## 目标

代理模式的可能性很多，但总的来说，它们都试图提供以下相同的功能：

+   隐藏对象在代理后面，以便可以隐藏，限制等功能

+   提供一个易于使用和易于更改的新抽象层

## 示例

对于我们的示例，我们将创建一个远程代理，它将是在访问数据库之前对象的缓存。假设我们有一个包含许多用户的数据库，但是我们不会每次想要获取有关用户的信息时都访问数据库，而是在代理模式下拥有一个用户的**先进先出**（**FIFO**）堆栈（FIFO 是一种说法，当缓存需要清空时，它将删除最先进入的对象）。

## 验收标准

我们将使用代理模式包装一个由切片表示的想象数据库。然后，该模式将必须遵循以下验收标准：

1.  所有对用户数据库的访问都将通过代理类型完成。

1.  代理中将保留`n`个最近用户的堆栈。

1.  如果用户已经存在于堆栈中，则不会查询数据库，并将返回存储的用户

1.  如果查询的用户不在堆栈中，则将查询数据库，如果堆栈已满，则删除堆栈中最旧的用户，存储新用户，并返回它。

## 单元测试

自 Go 的 1.7 版本以来，我们可以通过使用闭包在测试中嵌入测试，以便以更易读的方式对它们进行分组，并减少`Test_`函数的数量。请参阅第一章，*准备...开始...Go！*，了解如何安装新版本的 Go，如果您当前的版本早于 1.7 版本。

此模式的类型将是代理用户和用户列表结构以及`UserFinder`接口，数据库和代理将实现该接口。这很关键，因为代理必须实现与其尝试包装的类型的特性相同的接口：

```go
type UserFinder interface { 
  FindUser(id int32) (User, error) 
} 

```

`UserFinder`是数据库和代理实现的接口。`User`是一种具有名为`ID`的成员的类型，它是`int32`类型：

```go
type User struct { 
  ID int32 
} 

```

最后，`UserList`是用户切片的一种类型。考虑以下语法：

```go
type UserList []User 

```

如果您想知道为什么我们不直接使用用户切片，答案是，通过这种方式声明用户序列，我们可以实现`UserFinder`接口，但是使用切片，我们无法。

最后，代理类型称为`UserListProxy`，将由`UserList`切片组成，这将是我们的数据库表示。`StackCache`成员也将是`UserList`类型，以简化`StackCapacity`，以便给我们的堆栈指定大小。

为了本教程的目的，我们将稍微作弊，声明一个名为`DidDidLastSearchUsedCache`的字段上的布尔状态，该状态将保存上次执行的搜索是否使用了缓存，或者是否访问了数据库。

```go
type UserListProxy struct { 
  SomeDatabase UserList 
  StackCache UserList 
  StackCapacity int 
  DidDidLastSearchUsedCache bool 
} 

func (u *UserListProxy) FindUser(id int32) (User, error) { 
  return User{}, errors.New("Not implemented yet") 
} 

```

`UserListProxy`类型将缓存最多`StackCapacity`个用户，并在达到此限制时旋转缓存。`StackCache`成员将从`SomeDatabase`类型的对象中填充。

第一个测试称为`TestUserListProxy`，并列在下面：

```go
import ( 
   "math/rand" 
   "testing" 
) 

func Test_UserListProxy(t *testing.T) { 
  someDatabase := UserList{} 

  rand.Seed(2342342) 
  for i := 0; i < 1000000; i++ { 
    n := rand.Int31() 
    someDatabase = append(someDatabase, User{ID: n}) 
  } 

```

前面的测试创建了一个包含随机名称的 100 万用户的用户列表。为此，我们通过调用`Seed()`函数使用一些常量种子来为随机数生成器提供输入，以便我们的随机化结果也是常量；用户 ID 是从中生成的。它可能有一些重复，但它满足了我们的目的。

接下来，我们需要一个代理，它引用了刚刚创建的`someDatabase`：

```go
proxy := UserListProxy{ 
  SomeDatabase:  &someDatabase, 
  StackCapacity:  2, 
  StackCache: UserList{}, 
} 

```

此时，我们有一个由 1 百万用户组成的模拟数据库和一个大小为 2 的 FIFO 堆栈实现的缓存的`proxy`对象。现在我们将从`someDatabase`中获取三个随机 ID 来使用我们的堆栈：

```go
knownIDs := [3]int32 {someDatabase[3].ID, someDatabase[4].ID,someDatabase[5].ID} 

```

我们从切片中取出了第四、第五和第六个 ID（请记住，数组和切片从 0 开始，因此索引 3 实际上是切片中的第四个位置）。

这将是我们在启动嵌入式测试之前的起点。要创建嵌入式测试，我们必须调用`testing.T`指针的`Run`方法，其中包括描述和具有`func(t *testing.T)`签名的闭包：

```go
t.Run("FindUser - Empty cache", func(t *testing.T) { 
  user, err := proxy.FindUser(knownIDs[0]) 
  if err != nil { 
    t.Fatal(err) 
  } 

```

例如，在前面的代码片段中，我们给出了描述`FindUser - Empty cache`。然后我们定义我们的闭包。首先它尝试查找具有已知 ID 的用户，并检查错误。由于描述暗示，此时缓存为空，用户将不得不从`someDatabase`数组中检索：

```go
  if user.ID != knownIDs[0] { 
    t.Error("Returned user name doesn't match with expected") 
  } 

  if len(proxy.StackCache) != 1 { 
    t.Error("After one successful search in an empty cache, the size of it must be one") 
  } 

  if proxy.DidLastSearchUsedCache { 
    t.Error("No user can be returned from an empty cache") 
  } 
} 

```

最后，我们检查返回的用户是否具有与`knownIDs`切片的索引 0 处的预期用户相同的 ID，并且代理缓存现在的大小为 1。成员`DidLastSearchUsedCache`的状态代理不能是`true`，否则我们将无法通过测试。请记住，此成员告诉我们上次搜索是从表示数据库的切片中检索的，还是从缓存中检索的。

代理模式的第二个嵌入式测试是要求与之前相同的用户，现在必须从缓存中返回。这与以前的测试非常相似，但现在我们必须检查用户是否从缓存中返回：

```go
t.Run("FindUser - One user, ask for the same user", func(t *testing.T) { 
  user, err := proxy.FindUser(knownIDs[0]) 
  if err != nil { 
    t.Fatal(err) 
  } 

  if user.ID != knownIDs[0] { 
    t.Error("Returned user name doesn't match with expected") 
  } 

  if len(proxy.StackCache) != 1 { 
    t.Error("Cache must not grow if we asked for an object that is stored on it") 
  } 

  if !proxy.DidLastSearchUsedCache { 
    t.Error("The user should have been returned from the cache") 
  } 
}) 

```

因此，我们再次要求第一个已知的 ID。代理缓存在此搜索后必须保持大小为 1，并且这次`DidLastSearchUsedCache`成员必须为 true，否则测试将失败。

最后的测试将使`proxy`类型的`StackCache`数组溢出。我们将搜索两个新用户，我们的`proxy`类型将不得不从数据库中检索这些用户。我们的堆栈大小为 2，因此它将不得不删除第一个用户以为第二个和第三个用户分配空间：

```go
user1, err := proxy.FindUser(knownIDs[0]) 
if err != nil { 
  t.Fatal(err) 
} 

user2, _ := proxy.FindUser(knownIDs[1]) 
if proxy.DidLastSearchUsedCache { 
  t.Error("The user wasn't stored on the proxy cache yet") 
} 

user3, _ := proxy.FindUser(knownIDs[2]) 
if proxy.DidLastSearchUsedCache { 
  t.Error("The user wasn't stored on the proxy cache yet") 
} 

```

我们已经检索到了前三个用户。我们不检查错误，因为这是以前测试的目的。重要的是要记住，没有必要过度测试您的代码。如果这里有任何错误，它将在以前的测试中出现。此外，我们已经检查了`user2`和`user3`查询是否未使用缓存；它们不应该被存储在那里。

现在我们将在代理中查找`user1`查询。它不应该存在，因为堆栈的大小为 2，而`user1`是第一个进入的，因此也是第一个出去的：

```go
for i := 0; i < len(proxy.StackCache); i++ { 
  if proxy.StackCache[i].ID == user1.ID { 
    t.Error("User that should be gone was found") 
  } 
} 

if len(proxy.StackCache) != 2 { 
  t.Error("After inserting 3 users the cache should not grow" + 
" more than to two") 
} 

```

无论我们要求一千个用户，我们的缓存都不能大于我们配置的大小。

最后，我们将再次遍历存储在缓存中的用户，并将它们与我们查询的最后两个用户进行比较。这样，我们将检查只有这些用户存储在缓存中。两者都必须在其中找到：

```go
  for _, v := range proxy.StackCache { 
    if v != user2 && v != user3 { 
      t.Error("A non expected user was found on the cache") 
    } 
  } 
} 

```

现在运行测试应该会出现一些错误，像往常一样。现在让我们运行它们：

```go
$ go test -v .
=== RUN   Test_UserListProxy
=== RUN   Test_UserListProxy/FindUser_-_Empty_cache
=== RUN   Test_UserListProxy/FindUser_-_One_user,_ask_for_the_same_user
=== RUN   Test_UserListProxy/FindUser_-_overflowing_the_stack
--- FAIL: Test_UserListProxy (0.06s)
 --- FAIL: Test_UserListProxy/FindUser_-_Empty_cache (0.00s)
 proxy_test.go:28: Not implemented yet
 --- FAIL: Test_UserListProxy/FindUser_-_One_user,_ask_for_the_same_user (0.00s)
 proxy_test.go:47: Not implemented yet
 --- FAIL: Test_UserListProxy/FindUser_-_overflowing_the_stack (0.00s)
 proxy_test.go:66: Not implemented yet
FAIL
exit status 1
FAIL

```

因此，让我们实现`FindUser`方法以充当我们的代理。

## 实施

在我们的代理中，`FindUser`方法将在缓存列表中搜索指定的 ID。如果找到它，它将返回 ID。如果没有找到，它将在数据库中搜索。最后，如果它不在数据库列表中，它将返回一个错误。

如果您记得，我们的代理模式由两种`UserList`类型组成（其中一种是指针），它们实际上是`User`类型的切片。我们还将在`User`类型中实现一个`FindUser`方法，该方法与`UserFinder`接口具有相同的签名：

```go
type UserList []User 

func (t *UserList) FindUser(id int32) (User, error) { 
  for i := 0; i < len(*t); i++ { 
    if (*t)[i].ID == id { 
      return (*t)[i], nil 
    } 
  } 
  return User{}, fmt.Errorf("User %s could not be found\n", id) 
} 

```

`UserList`切片中的`FindUser`方法将遍历列表，尝试找到与`id`参数相同 ID 的用户，或者如果找不到则返回错误。

您可能想知道为什么指针`t`在括号之间。这是为了在访问其索引之前取消引用底层数组。如果没有它，您将会遇到编译错误，因为编译器会在取消引用指针之前尝试搜索索引。

因此，代理`FindUser`方法的第一部分可以编写如下：

```go
func (u *UserListProxy) FindUser(id int32) (User, error) { 
  user, err := u.StackCache.FindUser(id) 
  if err == nil { 
    fmt.Println("Returning user from cache") 
    u.DidLastSearchUsedCache = true 
    return user, nil 
  } 

```

我们使用上述方法在`StackCache`成员中搜索用户。如果找到用户，错误将为 nil，因此我们检查这一点，以便在控制台打印一条消息，将`DidLastSearchUsedCache`的状态更改为`true`，以便测试可以检查用户是否从缓存中检索，并最终返回用户。

因此，如果错误不是 nil，则意味着它无法在堆栈中找到用户。因此，下一步是在数据库中搜索：

```go
  user, err = u.SomeDatabase.FindUser(id) 
  if err != nil { 
    return User{}, err 
  } 

```

在这种情况下，我们可以重用我们为`UserList`数据库编写的`FindUser`方法，因为在这个例子的目的上，两者具有相同的类型。同样，它在数据库中搜索由`UserList`切片表示的用户，但在这种情况下，如果找不到用户，则返回`UserList`中生成的错误。

当找到用户（`err`为 nil）时，我们必须将用户添加到堆栈中。为此，我们编写了一个专用的私有方法，该方法接收`UserListProxy`类型的指针：

```go
func (u *UserListProxy) addUserToStack(user User) { 
  if len(u.StackCache) >= u.StackCapacity { 
    u.StackCache = append(u.StackCache[1:], user) 
  } 
  else { 
    u.StackCache.addUser(user) 
  } 
} 

func (t *UserList) addUser(newUser User) { 
  *t = append(*t, newUser) 
} 

```

`addUserToStack`方法接受用户参数，并将其放置在堆栈中。如果堆栈已满，则在添加之前删除其中的第一个元素。我们还编写了一个`addUser`方法来帮助我们在`UserList`中。因此，现在在`FindUser`方法中，我们只需添加一行：

```go
u.addUserToStack(user) 

```

这将新用户添加到堆栈中，必要时删除最后一个。

最后，我们只需返回堆栈的新用户，并在`DidLastSearchUsedCache`变量上设置适当的值。我们还向控制台写入一条消息，以帮助测试过程：

```go
  fmt.Println("Returning user from database") 
  u.DidLastSearchUsedCache = false 
  return user, nil 
} 

```

有了这个，我们就有足够的内容来通过我们的测试：

```go
$ go test -v .
=== RUN   Test_UserListProxy
=== RUN   Test_UserListProxy/FindUser_-_Empty_cache
Returning user from database
=== RUN   Test_UserListProxy/FindUser_-_One_user,_ask_for_the_same_user
Returning user from cache
=== RUN   Test_UserListProxy/FindUser_-_overflowing_the_stack
Returning user from cache
Returning user from database
Returning user from database
--- PASS: Test_UserListProxy (0.09s) 
--- PASS: Test_UserListProxy/FindUser_-_Empty_cache (0.00s)
--- PASS: Test_UserListProxy/FindUser_-_One_user,_ask_for_the_same_user (0.00s)
--- PASS: Test_UserListProxy/FindUser_-_overflowing_the_stack (0.00s)
PASS
ok

```

您可以在前面的消息中看到，我们的代理已经完美地工作。它已经从数据库中返回了第一次搜索。然后，当我们再次搜索相同的用户时，它使用了缓存。最后，我们进行了一个新的测试，调用了三个不同的用户，通过查看控制台输出，我们可以观察到只有第一个用户是从缓存中返回的，其他两个是从数据库中获取的。

## 围绕操作进行代理

在需要进行一些中间操作的类型周围包装代理，比如为用户提供授权或提供对数据库的访问，就像我们的示例一样。

我们的示例是将应用程序需求与数据库需求分离的好方法。如果我们的应用程序对数据库的访问过多，解决方案并不在于数据库。请记住，代理使用与其包装的类型相同的接口，对于用户来说，两者之间不应该有任何区别。

# 装饰器设计模式

我们将继续本章，介绍代理模式的大哥，也许是最强大的设计模式之一。**装饰器**模式非常简单，但是在处理旧代码时提供了许多好处。

## 描述

装饰器设计模式允许您在不实际触及它的情况下为已经存在的类型添加更多的功能特性。这是如何可能的呢？嗯，它使用了一种类似于*玛特里奥什卡娃娃*的方法，您可以将一个小娃娃放在一个相同形状但更大的娃娃中，依此类推。

装饰器类型实现了它装饰的类型的相同接口，并在其成员中存储该类型的实例。这样，您可以通过简单地将旧的装饰器存储在新装饰器的字段中来堆叠尽可能多的装饰器（玩偶）。

## 目标

当您考虑扩展旧代码而不会破坏任何东西时，您应该首先考虑装饰器模式。这是一种处理这个特定问题的非常强大的方法。

装饰器非常强大的另一个领域可能并不那么明显，尽管当基于用户输入、偏好或类似输入创建具有许多功能的类型时，它会显现出来。就像瑞士军刀一样，您有一个基本类型（刀的框架），然后您展开其功能。

那么，我们什么时候会使用装饰器模式呢？对这个问题的回答：

+   当您需要向一些无法访问的代码添加功能，或者您不希望修改以避免对代码产生负面影响，并遵循开放/封闭原则（如旧代码）时。

+   当您希望动态创建或更改对象的功能，并且功能数量未知且可能快速增长时

## 示例

在我们的示例中，我们将准备一个`Pizza`类型，其中核心是披萨，配料是装饰类型。我们的披萨上会有一些配料，比如洋葱和肉。

## 验收标准

装饰器模式的验收标准是具有一个公共接口和一个核心类型，所有层都将在其上构建：

+   我们必须有所有装饰器都将实现的主要接口。这个接口将被称为`IngredientAdd`，它将具有`AddIngredient() string`方法。

+   我们必须有一个核心`PizzaDecorator`类型（装饰器），我们将向其添加配料。

+   我们必须有一个实现相同`IngredientAdd`接口的配料`onion`，它将向返回的披萨添加字符串`onion`。

+   我们必须有一个实现`IngredientAdd`接口的配料`meat`，它将向返回的披萨添加字符串`meat`。

+   在顶层对象上调用`AddIngredient`方法时，它必须返回一个带有文本`Pizza with the following ingredients: meat, onion`的完全装饰的`pizza`。

## 单元测试

要启动我们的单元测试，我们必须首先根据验收标准创建基本结构。首先，所有装饰类型必须实现的接口如下：

```go
type IngredientAdd interface { 
  AddIngredient() (string, error) 
} 

```

以下代码定义了`PizzaDecorator`类型，其中必须包含`IngredientAdd`，并且它也实现了`IngredientAdd`：

```go
type PizzaDecorator struct{ 
  Ingredient IngredientAdd 
} 

func (p *PizzaDecorator) AddIngredient() (string, error) { 
  return "", errors.New("Not implemented yet") 
} 

```

`Meat`类型的定义将与`PizzaDecorator`结构的定义非常相似：

```go
type Meat struct { 
  Ingredient IngredientAdd 
} 

func (m *Meat) AddIngredient() (string, error) { 
  return "", errors.New("Not implemented yet") 
} 

```

现在我们以类似的方式定义`Onion`结构体：

```go
type Onion struct { 
  Ingredient IngredientAdd 
} 

func (o *Onion) AddIngredient() (string, error) { 
  return "", errors.New("Not implemented yet") 
}  

```

这已足以实现第一个单元测试，并允许编译器在没有任何编译错误的情况下运行它们：

```go
func TestPizzaDecorator_AddIngredient(t *testing.T) { 
  pizza := &PizzaDecorator{} 
  pizzaResult, _ := pizza.AddIngredient() 
  expectedText := "Pizza with the following ingredients:" 
  if !strings.Contains(pizzaResult, expectedText) { 
    t.Errorf("When calling the add ingredient of the pizza decorator it must return the text %sthe expected text, not '%s'", pizzaResult, expectedText) 
  } 
} 

```

现在它必须能够无问题地编译，这样我们就可以检查测试是否失败：

```go
$ go test -v -run=TestPizzaDecorator .
=== RUN   TestPizzaDecorator_AddIngredient
--- FAIL: TestPizzaDecorator_AddIngredient (0.00s)
decorator_test.go:29: Not implemented yet
decorator_test.go:34: When the the AddIngredient method of the pizza decorator object is called, it must return the text
Pizza with the following ingredients:
FAIL
exit status 1
FAIL 

```

我们的第一个测试已经完成，我们可以看到`PizzaDecorator`结构体还没有返回任何东西，这就是为什么它失败了。现在我们可以继续进行`Onion`类型的测试。`Onion`类型的测试与`Pizza`装饰器的测试非常相似，但我们还必须确保我们实际上将配料添加到`IngredientAdd`方法而不是空指针：

```go
func TestOnion_AddIngredient(t *testing.T) { 
  onion := &Onion{} 
  onionResult, err := onion.AddIngredient() 
  if err == nil { 
    t.Errorf("When calling AddIngredient on the onion decorator without" + "an IngredientAdd on its Ingredient field must return an error, not a string with '%s'", onionResult) 
  } 

```

前面测试的前半部分检查了当没有将`IngredientAdd`方法传递给`Onion`结构体初始化程序时返回错误。由于没有可用的披萨来添加配料，必须返回错误：

```go
  onion = &Onion{&PizzaDecorator{}} 
  onionResult, err = onion.AddIngredient() 

  if err != nil { 
    t.Error(err) 
  } 
  if !strings.Contains(onionResult, "onion") { 
    t.Errorf("When calling the add ingredient of the onion decorator it" + "must return a text with the word 'onion', not '%s'", onionResult) 
  } 
} 

```

`Onion`类型测试的第二部分实际上将`PizzaDecorator`结构传递给初始化程序。然后，我们检查是否没有返回错误，以及返回的字符串是否包含单词`onion`。这样，我们可以确保洋葱已添加到比萨中。

最后对于`Onion`类型，我们当前实现的测试的控制台输出将如下所示：

```go
$ go test -v -run=TestOnion_AddIngredient .
=== RUN   TestOnion_AddIngredient
--- FAIL: TestOnion_AddIngredient (0.00s)
decorator_test.go:48: Not implemented yet
decorator_test.go:52: When calling the add ingredient of the onion decorator it must return a text with the word 'onion', not ''
FAIL
exit status 1
FAIL

```

`meat`成分完全相同，但我们将类型更改为肉而不是洋葱：

```go
func TestMeat_AddIngredient(t *testing.T) { 
  meat := &Meat{} 
  meatResult, err := meat.AddIngredient() 
  if err == nil { 
    t.Errorf("When calling AddIngredient on the meat decorator without" + "an IngredientAdd in its Ingredient field must return an error," + "not a string with '%s'", meatResult) 
  } 

  meat = &Meat{&PizzaDecorator{}} 
  meatResult, err = meat.AddIngredient() 
  if err != nil { 
    t.Error(err) 
  } 

  if !strings.Contains(meatResult, "meat") { 
    t.Errorf("When calling the add ingredient of the meat decorator it" + "must return a text with the word 'meat', not '%s'", meatResult) 
  } 
} 

```

因此，测试的结果将是类似的：

```go
go test -v -run=TestMeat_AddIngredient .
=== RUN   TestMeat_AddIngredient
--- FAIL: TestMeat_AddIngredient (0.00s)
decorator_test.go:68: Not implemented yet
decorator_test.go:72: When calling the add ingredient of the meat decorator it must return a text with the word 'meat', not ''
FAIL
exit status 1
FAIL

```

最后，我们必须检查完整的堆栈测试。创建一个带有洋葱和肉的比萨必须返回文本`带有以下配料的比萨：肉，洋葱`：

```go
func TestPizzaDecorator_FullStack(t *testing.T) { 
  pizza := &Onion{&Meat{&PizzaDecorator{}}} 
  pizzaResult, err := pizza.AddIngredient() 
  if err != nil { 
    t.Error(err) 
  } 

  expectedText := "Pizza with the following ingredients: meat, onion" 
  if !strings.Contains(pizzaResult, expectedText){ 
    t.Errorf("When asking for a pizza with onion and meat the returned " + "string must contain the text '%s' but '%s' didn't have it", expectedText,pizzaResult) 
  } 

  t.Log(pizzaResult) 
} 

```

我们的测试创建了一个名为`pizza`的变量，就像`套娃`玩偶一样，嵌入了多个级别的`IngredientAdd`方法的类型。调用`AddIngredient`方法执行"洋葱"级别的方法，该方法执行"肉"级别的方法，最后执行`PizzaDecorator`结构的方法。在检查是否没有返回错误后，我们检查返回的文本是否符合*验收标准 5*的需求。测试使用以下命令运行：

```go
go test -v -run=TestPizzaDecorator_FullStack .
=== RUN   TestPizzaDecorator_FullStack
--- FAIL: TestPizzaDecorator_FullStack (0.
decorator_test.go:80: Not implemented yet
decorator_test.go:87: When asking for a pizza with onion and meat the returned string must contain the text 'Pizza with the following ingredients: meat, onion' but '' didn't have it
FAIL
exit status 1
FAIL

```

从前面的输出中，我们可以看到测试现在为我们装饰的类型返回一个空字符串。当然，这是因为尚未进行任何实现。这是最后一个测试，用于检查完全装饰的实现。然后让我们仔细看看实现。

## 实施

我们将开始实现`PizzaDecorator`类型。它的作用是提供完整比萨的初始文本：

```go
type PizzaDecorator struct { 
  Ingredient IngredientAdd 
} 

func (p *PizzaDecorator) AddIngredient() (string, error) { 
  return "Pizza with the following ingredients:", nil 
} 

```

在`AddIngredient`方法的返回上进行了一行更改就足以通过测试：

```go
go test -v -run=TestPizzaDecorator_Add .
=== RUN   TestPizzaDecorator_AddIngredient
--- PASS: TestPizzaDecorator_AddIngredient (0.00s)
PASS
ok

```

转到`Onion`结构的实现，我们必须取得我们返回的`IngredientAdd`字符串的开头，并在其末尾添加单词`onion`，以便得到一份组合的比萨：

```go
type Onion struct { 
  Ingredient IngredientAdd 
} 

func (o *Onion) AddIngredient() (string, error) { 
  if o.Ingredient == nil { 
    return "", errors.New("An IngredientAdd is needed in the Ingredient field of the Onion") 
  } 
  s, err := o.Ingredient.AddIngredient() 
  if err != nil { 
    return "", err 
  } 
  return fmt.Sprintf("%s %s,", s, "onion"), nil 
} 

```

首先检查我们是否实际上有一个指向`IngredientAdd`的指针，我们使用内部`IngredientAdd`的内容，并检查是否有错误。如果没有错误发生，我们将收到一个由此内容、一个空格和单词`onion`（没有错误）组成的新字符串。看起来足够好来运行测试：

```go
go test -v -run=TestOnion_AddIngredient .
=== RUN   TestOnion_AddIngredient
--- PASS: TestOnion_AddIngredient (0.00s)
PASS
ok

```

`Meat`结构的实现非常相似：

```go
type Meat struct { 
  Ingredient IngredientAdd 
} 

func (m *Meat) AddIngredient() (string, error) { 
  if m.Ingredient == nil { 
    return "", errors.New("An IngredientAdd is needed in the Ingredient field of the Meat") 
  } 
  s, err := m.Ingredient.AddIngredient() 
  if err != nil { 
    return "", err 
  } 
  return fmt.Sprintf("%s %s,", s, "meat"), nil 
} 

```

他们的测试执行如下：

```go
go test -v -run=TestMeat_AddIngredient .
=== RUN   TestMeat_AddIngredient
--- PASS: TestMeat_AddIngredient (0.00s)
PASS
ok

```

好的。现在所有的部分都要分别测试。如果一切正常，*完全堆叠*解决方案的测试必须顺利通过：

```go
go test -v -run=TestPizzaDecorator_FullStack .
=== RUN   TestPizzaDecorator_FullStack
--- PASS: TestPizzaDecorator_FullStack (0.00s)
decorator_test.go:92: Pizza with the following ingredients: meat, onion,
PASS
ok

```

太棒了！使用装饰器模式，我们可以不断堆叠调用它们内部指针以向`PizzaDecorator`添加功能的`IngredientAdds`。我们也不会触及核心类型，也不会修改或实现新的东西。所有新功能都是由外部类型实现的。

## 一个现实生活的例子-服务器中间件

到目前为止，您应该已经了解了装饰器模式的工作原理。现在我们可以尝试使用我们在适配器模式部分设计的小型 HTTP 服务器的更高级示例。您已经学会了可以使用`http`包创建 HTTP 服务器，并实现`http.Handler`接口。该接口只有一个名为`ServeHTTP(http.ResponseWriter, http.Request)`的方法。我们可以使用装饰器模式为服务器添加更多功能吗？当然可以！

我们将向此服务器添加一些部分。首先，我们将记录对其进行的每个连接到`io.Writer`接口（为简单起见，我们将使用`os.Stdout`接口的`io.Writer`实现，以便将其输出到控制台）。第二部分将在发送到服务器的每个请求上添加基本的 HTTP 身份验证。如果身份验证通过，将出现`Hello Decorator!`消息。最后，用户将能够选择他/她在服务器中想要的装饰项的数量，并且服务器将在运行时进行结构化和创建。

### 从常见接口 http.Handler 开始

我们已经有了我们将使用嵌套类型进行装饰的通用接口。我们首先需要创建我们的核心类型，这将是返回句子`Hello Decorator!`的`Handler`。

```go
type MyServer struct{} 

func (m *MyServer) ServeHTTP(w http.ResponseWriter, r *http.Request) { 
  fmt.Fprintln(w, "Hello Decorator!") 
} 

```

这个处理程序可以归因于`http.Handle`方法，以定义我们的第一个端点。现在让我们通过创建包的`main`函数来检查这一点，并向其发送一个`GET`请求：

```go
func main() { 
  http.Handle("/", &MyServer{}) 

  log.Fatal(http.ListenAndServe(":8080", nil)) 
} 

```

使用终端执行服务器以执行`**go run main.go**`命令。然后，打开一个新的终端进行`GET`请求。我们将使用`curl`命令进行请求：

```go
$ curl http://localhost:8080
Hello Decorator!

```

我们已经跨越了我们装饰服务器的第一个里程碑。下一步是用日志功能装饰它。为此，我们必须实现`http.Handler`接口，以新类型的形式进行如下实现：

```go
type LoggerServer struct { 
  Handler   http.Handler 
  LogWriter io.Writer 
} 

func (s *LoggerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) { 
  fmt.Fprintf(s.LogWriter, "Request URI: %s\n", r.RequestURI) 
  fmt.Fprintf(s.LogWriter, "Host: %s\n", r.Host) 
  fmt.Fprintf(s.LogWriter, "Content Length: %d\n",  
r.ContentLength) 
  fmt.Fprintf(s.LogWriter, "Method: %s\n", r.Method)fmt.Fprintf(s.LogWriter, "--------------------------------\n") 

  s.Handler.ServeHTTP(w, r) 
} 

```

我们称这种类型为`LoggerServer`。正如你所看到的，它不仅存储`Handler`，还存储`io.Writer`以写入日志的输出。我们的`ServeHTTP`方法的实现打印请求 URI、主机、内容长度和使用的方法`io.Writer`。打印完成后，它调用其内部`Handler`字段的`ServeHTTP`函数。

我们可以用`LoggerMiddleware`装饰`MyServer`：

```go
func main() { 
  http.Handle("/", &LoggerServer{ 
    LogWriter:os.Stdout, 
    Handler:&MyServer{}, 
  }) 

  log.Fatal(http.ListenAndServe(":8080", nil)) 
} 

```

现在运行`**curl **`命令：

```go
$ curl http://localhost:8080
Hello Decorator!

```

我们的**curl**命令返回相同的消息，但是如果你查看运行 Go 应用程序的终端，你可以看到日志：

```go
$ go run server_decorator.go
Request URI: /
Host: localhost:8080
Content Length: 0
Method: GET

```

我们已经用日志功能装饰了`MyServer`，而实际上并没有修改它。我们能否用相同的方法进行身份验证？当然可以！在记录请求后，我们将使用**HTTP 基本身份验证**进行身份验证：

```go
type BasicAuthMiddleware struct { 
  Handler  http.Handler 
  User     string 
  Password string 
} 

```

**BasicAuthMiddleware**中间件存储三个字段--一个要装饰的处理程序，就像前面的中间件一样，一个用户和一个密码，这将是访问服务器内容的唯一授权。`decorating`方法的实现将如下进行：

```go
func (s *BasicAuthMiddleware) ServeHTTP(w http.ResponseWriter, r *http.Request) { 
  user, pass, ok := r.BasicAuth() 

  if ok { 
    if user == s.User && pass == s.Password { 
      s.Handler.ServeHTTP(w, r) 
    } 
    else { 
      fmt.Fprintf(w, "User or password incorrect\n") 
    } 
  } 
  else { 
    fmt.Fprintln(w, "Error trying to retrieve data from Basic auth") 
  } 
} 

```

在前面的实现中，我们使用`http.Request`的`BasicAuth`方法自动从请求中检索用户和密码，以及解析操作的`ok/ko`。然后我们检查解析是否正确（如果不正确则向请求者返回消息，并结束请求）。如果在解析过程中没有检测到问题，我们将检查用户名和密码是否与`BasicAuthMiddleware`中存储的用户名和密码匹配。如果凭据有效，我们将调用装饰类型（我们的服务器），但如果凭据无效，我们将收到`用户或密码不正确`的消息，并结束请求。

现在，我们需要为用户提供一种选择不同类型服务器的方式。我们将在主函数中检索用户输入数据。我们将有三个选项可供选择：

+   简单服务器

+   带有日志的服务器

+   带有日志和身份验证的服务器

我们必须使用`Fscanf`函数从用户那里检索输入：

```go
func main() { 
  fmt.Println("Enter the type number of server you want to launch from the  following:") 
  fmt.Println("1.- Plain server") 
  fmt.Println("2.- Server with logging") 
  fmt.Println("3.- Server with logging and authentication") 

  var selection int 
  fmt.Fscanf(os.Stdin, "%d", &selection) 
} 

```

`Fscanf`函数需要一个`io.Reader`实现者作为第一个参数（这将是控制台中的输入），并从中获取用户选择的服务器。我们将传递`os.Stdin`作为`io.Reader`接口来检索用户输入。然后，我们将写入它将要解析的数据类型。`%d`指定符指的是整数。最后，我们将写入存储解析输入的内存地址，即`selection`变量的内存位置。

一旦用户选择了一个选项，我们就可以在运行时获取基本服务器并进行装饰，切换到所选的选项：

```go
   switch selection { 
   case 1: 
     mySuperServer = new(MyServer) 
   case 2: 
     mySuperServer = &LoggerMiddleware{ 
       Handler:   new(MyServer), 
       LogWriter: os.Stdout, 
     } 
   case 3: 
     var user, password string 

     fmt.Println("Enter user and password separated by a space") 
     fmt.Fscanf(os.Stdin, "%s %s", &user, &password) 

     mySuperServer = &LoggerMiddleware{ 
     Handler: &SimpleAuthMiddleware{ 
       Handler:  new(MyServer), 
       User:     user, 
       Password: password, 
     }, 
     LogWriter: os.Stdout, 
   } 
   default: 
   mySuperServer = new(MyServer) 
 } 

```

第一个选项将由默认的`switch`选项处理--一个普通的`MyServer`。在第二个选项的情况下，我们用日志装饰了一个普通服务器。第三个选项更加复杂--我们再次使用`Fscanf`要求用户输入用户名和密码。请注意，您可以扫描多个输入，就像我们正在检索用户和密码一样。然后，我们获取基本服务器，用身份验证进行装饰，最后再加上日志。

如果您遵循第三个选项的嵌套类型的缩进，请求将通过记录器，然后通过身份验证中间件，最后，如果一切正常，将通过`MyServer`参数。请求将遵循相同的路线。

主函数的结尾采用了装饰处理程序，并在`8080`端口上启动服务器：

```go
http.Handle("/", mySuperServer) 
log.Fatal(http.ListenAndServe(":8080", nil)) 

```

因此，让我们使用第三个选项启动服务器：

```go
$go run server_decorator.go 
Enter the server type number you want to launch from the following: 
1.- Plain server 
2.- Server with logging 
3.- Server with logging and authentication 

Enter user and password separated by a space 
mario castro

```

首先，我们将通过选择第一个选项来测试普通服务器。使用命令**go run server_decorator.go**运行服务器，并选择第一个选项。然后，在另一个终端中，使用 curl 运行基本请求，如下所示：

```go
$ curl http://localhost:8080
Error trying to retrieve data from Basic auth

```

哦，不！它没有给我们访问权限。我们没有传递任何用户名和密码，因此它告诉我们我们无法继续。让我们尝试一些随机的用户名和密码：

```go
$ curl -u no:correct http://localhost:8080
User or password incorrect

```

没有访问权限！我们还可以在启动服务器的终端中检查每个请求的记录位置：

```go
Request URI: /
Host: localhost:8080
Content Length: 0
Method: GET

```

最后，输入正确的用户名和密码：

```go
$ curl -u packt:publishing http://localhost:8080
Hello Decorator!

```

我们到这里了！我们的请求也已被记录，服务器已经授予我们访问权限。现在我们可以通过编写更多的中间件来改进服务器的功能。

## 关于 Go 的结构化类型的几句话

Go 有一个大多数人一开始不喜欢的特性 - 结构化类型。这是指您的结构定义了您的类型，而无需明确编写它。例如，当您实现一个接口时，您不必明确地写出您实际上正在实现它，与 Java 等语言相反，在那里您必须写出关键字`implements`。如果您的方法遵循接口的签名，那么您实际上正在实现接口。这也可能导致意外实现接口，这可能引起无法跟踪的错误，但这种情况非常罕见。

然而，结构化类型也允许您在定义实现者之后定义接口。想象一个`MyPrinter`结构如下：

```go
type MyPrinter struct{} 
func(m *MyPrinter)Print(){ 
  println("Hello") 
} 

```

假设我们现在已经使用`MyPrinter`类型工作了几个月，但它没有实现任何接口，因此不能成为装饰器模式的可能候选，或者可能可以？如果几个月后编写了一个与其`Print`方法匹配的接口，会发生什么？考虑以下代码片段：

```go
type Printer interface { 
  Print() 
} 

```

它实际上实现了`Printer`接口，我们可以使用它来创建一个装饰器解决方案。

结构化类型在编写程序时提供了很大的灵活性。如果您不确定类型是否应该是接口的一部分，可以将其留下，并在完全确定后再添加接口。这样，您可以非常轻松地装饰类型，并且在源代码中进行很少的修改。

## 总结装饰器设计模式 - 代理与装饰器

您可能会想知道装饰器模式和代理模式之间有什么区别？在装饰器模式中，我们动态地装饰一个类型。这意味着装饰可能存在也可能不存在，或者可能由一个或多个类型组成。如果您记得，代理模式以类似的方式包装类型，但它是在编译时这样做的，更像是一种访问某种类型的方式。

同时，装饰器可能实现其装饰的类型也实现的整个接口**或者不实现**。因此，您可以拥有一个具有 10 个方法的接口和一个只实现其中一个方法的装饰器，它仍然有效。对装饰器未实现的方法的调用将传递给装饰的类型。这是一个非常强大的功能，但如果您忘记实现任何接口方法，它也很容易出现运行时的不良行为。

在这方面，你可能会认为代理模式不够灵活，确实如此。但装饰器模式更弱，因为你可能会在运行时出现错误，而使用代理模式可以在编译时避免这些错误。只需记住，装饰器通常用于在运行时向对象添加功能，就像我们的 Web 服务器一样。这是你需要的东西和你愿意牺牲以实现它之间的妥协。

# 外观设计模式

在本章中我们将看到的下一个模式是外观模式。当我们讨论代理模式时，你了解到它是一种包装类型以隐藏某些特性或复杂性的方式。想象一下，我们将许多代理组合在一个单一点，比如一个文件或一个库。这就是外观模式。

## 描述

在建筑学中，外观是隐藏建筑物房间和走廊的前墙。它保护居民免受寒冷和雨水的侵袭，并为他们提供隐私。它对住宅进行排序和划分。

外观设计模式在我们的代码中做了相同的事情。它保护代码免受未经授权的访问，对一些调用进行排序，并将复杂性范围隐藏在用户视野之外。

## 目标

当你想要隐藏某些任务的复杂性时，特别是当大多数任务共享实用程序时（例如在 API 中进行身份验证）。库是外观的一种形式，其中某人必须为开发人员提供一些方法，以便以友好的方式执行某些操作。这样，如果开发人员需要使用你的库，他不需要知道检索所需结果的所有内部任务。

因此，在以下情况下使用外观设计模式：

+   当你想要减少我们代码的某些部分的复杂性时。你通过提供更易于使用的方法将复杂性隐藏在外观后面。

+   当你想要将相关的操作分组到一个地方时。

+   当你想要构建一个库，以便其他人可以使用你的产品而不必担心它是如何工作的。

## 例子

举例来说，我们将迈出编写访问`OpenWeatherMaps`服务的自己库的第一步。如果你不熟悉`OpenWeatherMap`服务，它是一个提供实时天气信息以及历史数据的 HTTP 服务。**HTTP REST** API 非常易于使用，并且将是一个很好的例子，说明如何为隐藏 REST 服务背后的网络连接的复杂性创建外观模式。

## 接受标准

`OpenWeatherMap` API 提供了大量信息，因此我们将专注于通过使用其纬度和经度值在某个地理位置获取实时天气数据。以下是此设计模式的要求和接受标准：

1.  提供一个单一类型来访问数据。从`OpenWeatherMap`服务检索到的所有信息都将通过它传递。

1.  创建一种获取某个国家的某个城市的天气数据的方法。

1.  创建一种获取某个纬度和经度位置的天气数据的方法。

1.  只有第二和第三点必须在包外可见；其他所有内容都必须隐藏（包括所有连接相关的数据）。

## 单元测试

为了开始我们的 API 外观，我们需要一个接口，其中包含*接受标准 2*和*接受标准 3*中要求的方法：

```go
type CurrentWeatherDataRetriever interface { 
  GetByCityAndCountryCode(city, countryCode string) (Weather, error) 
  GetByGeoCoordinates(lat, lon float32) (Weather, error) 
} 

```

我们将称*接受标准 2*为`GetByCityAndCountryCode`；我们还需要一个城市名称和一个国家代码，格式为字符串。国家代码是一个两个字符的代码，代表着世界各国的**国际标准化组织**（**ISO**）名称。它返回一个`Weather`值，我们稍后会定义，并且如果出现问题会返回一个错误。

*验收标准 3*将被称为`GetByGeoCoordinates`，并且需要`float32`格式的纬度和经度值。它还将返回`Weather`值和错误。`Weather`值将根据`OpenWeatherMap` API 使用的返回 JSON 进行定义。您可以在网页[`openweathermap.org/current#current_JSON`](http://openweathermap.org/current#current_JSON)上找到此 JSON 的描述。

如果查看 JSON 定义，它具有以下类型：

```go
type Weather struct { 
  ID   int    `json:"id"` 
  Name string `json:"name"` 
  Cod  int    `json:"cod"` 
  Coord struct { 
    Lon float32 `json:"lon"` 
    Lat float32 `json:"lat"` 
  } `json:"coord"`  

  Weather []struct { 
    Id          int    `json:"id"` 
    Main        string `json:"main"` 
    Description string `json:"description"` 
    Icon        string `json:"icon"` 
  } `json:"weather"` 

  Base string `json:"base"` 
  Main struct { 
    Temp     float32 `json:"temp"` 
    Pressure float32 `json:"pressure"` 
    Humidity float32 `json:"humidity"` 
    TempMin  float32 `json:"temp_min"` 
    TempMax  float32 `json:"temp_max"` 
  } `json:"main"` 

  Wind struct { 
    Speed float32 `json:"speed"` 
    Deg   float32 `json:"deg"` 
  } `json:"wind"` 

  Clouds struct { 
    All int `json:"all"` 
  } `json:"clouds"` 

  Rain struct { 
    ThreeHours float32 `json:"3h"` 
  } `json:"rain"` 

  Dt  uint32 `json:"dt"` 
  Sys struct { 
    Type    int     `json:"type"` 
    ID      int     `json:"id"` 
    Message float32 `json:"message"` 
    Country string  `json:"country"` 
    Sunrise int     `json:"sunrise"` 
    Sunset  int     `json:"sunset"` 
  }`json:"sys"` 
} 

```

这是一个相当长的结构，但我们拥有响应可能包含的所有内容。该结构称为`Weather`，因为它由 ID，名称和代码（`Cod`）以及一些匿名结构组成，即`Coord`，`Weather`，`Base`，`Main`，`Wind`，`Clouds`，`Rain`，`Dt`和`Sys`。我们可以通过给它们命名来在`Weather`结构之外编写这些匿名结构，但是只有在我们必须单独使用它们时才有用。

在我们的`Weather`结构中的每个成员和结构之后，您可以找到一个``json：`something```行。当区分 JSON 键名和成员名时，这非常方便。如果 JSON 键是`something`，我们就不必将我们的成员称为`something`。例如，我们的 ID 成员在 JSON 响应中将被称为`id`。

为什么我们不将 JSON 键的名称给我们的类型？好吧，如果您的类型中的字段是小写的，则`encoding/json`包将无法正确解析它们。此外，最后的注释为我们提供了一定的灵活性，不仅可以更改成员名称，还可以省略一些键（如果我们不需要），具有以下签名：

```go
`json:"something,omitempty" 

```

在末尾使用`omitempty`，如果此键在 JSON 键的字节表示中不存在，则解析不会失败。

好的，我们的验收标准 1 要求对 API 进行单点访问。这将被称为`CurrentWeatherData`：

```go
type CurrentWeatherData struct { 
  APIkey string 
} 

```

`CurrentWeatherData`类型具有 API 密钥作为公共成员以工作。这是因为您必须是`OpenWeatherMap`中的注册用户才能享受其服务。请参阅`OpenWeatherMap` API 的网页，了解如何获取 API 密钥的文档。在我们的示例中，我们不需要它，因为我们不打算进行集成测试。

我们需要模拟数据，以便我们可以编写`mock`函数来检索数据。发送 HTTP 请求时，响应以`io.Reader`的形式包含在名为 body 的成员中。我们已经使用了实现`io.Reader`接口的类型，因此这对您来说应该很熟悉。我们的`mock`函数如下所示：

```go
 func getMockData() io.Reader { 
  response := `{
    "coord":{"lon":-3.7,"lat":40.42},"weather : [{"id":803,"main":"Clouds","description":"broken clouds","icon":"04n"}],"base":"stations","main":{"temp":303.56,"pressure":1016.46,"humidity":26.8,"temp_min":300.95,"temp_max":305.93},"wind":{"speed":3.17,"deg":151.001},"rain":{"3h":0.0075},"clouds":{"all":68},"dt":1471295823,"sys":{"type":3,"id":1442829648,"message":0.0278,"country":"ES","sunrise":1471238808,"sunset":1471288232},"id":3117735,"name":"Madrid","cod":200}` 

  r := bytes.NewReader([]byte(response)) 
  return r 
} 

```

通过对`OpenWeatherMap`使用 API 密钥进行请求生成了前面的模拟数据。`response`变量是包含 JSON 响应的字符串。仔细看一下重音符（```go) used to open and close the string. This way, you can use as many quotes as you want without any problem.

Further on, we use a special function in the bytes package called `NewReader`, which accepts an slice of bytes (which we create by converting the type from string), and returns an `io.Reader` implementor with the contents of the slice. This is perfect to mimic the `Body` member of an HTTP response.

We will write a test to try `response parser`. Both methods return the same type, so we can use the same `JSON parser` for both:

```

func TestOpenWeatherMap_responseParser（t * testing.T）{

r：= getMockData（）

openWeatherMap：= CurrentWeatherData {APIkey：""}

weather，err：= openWeatherMap.responseParser（r）

if err！= nil {

t.Fatal（err）

}

如果 weather.ID！= 3117735 {

t.Errorf（“马德里 id 为 3117735，而不是％d\n”，weather.ID）

}

}

```go

In the preceding test, we first asked for some mock data, which we store in the variable `r`. Later, we created a type of `CurrentWeatherData`, which we called `openWeatherMap`. Finally, we asked for a weather value for the provided `io.Reader` interface that we store in the variable `weather`. After checking for errors, we make sure that the ID is the same as the one stored in the mock data that we got from the `getMockData` method.

We have to declare the `responseParser` method before running tests, or the code won't compile:

```

func（p * CurrentWeatherData）responseParser（body io.Reader）（* Weather，error）{

返回 nil，fmt.Errorf（“尚未实施”）

}

```go

With all the aforementioned, we can run this test:

```

go test -v -run=responseParser。

=== RUN TestOpenWeatherMap_responseParser

---失败：测试 OpenWeatherMap_responseParser（0.00 秒）

facade_test.go：72：尚未实施

失败

退出状态 1

失败

```go

Okay. We won't write more tests, because the rest would be merely integration tests, which are outside of the scope of explanation of a structural pattern, and will force us to have an API key as well as an Internet connection. If you want to see what the integration tests look like for this example, refer to the code that comes bundled with the book.

## Implementation

First of all, we are going to implement the parser that our methods will use to parse the JSON response from the `OpenWeatherMap` REST API:

```

func（p * CurrentWeatherData）responseParser（body io.Reader）（* Weather，error）{

w：= new（Weather）

err：= json.NewDecoder（body）.Decode（w）

if err！= nil {

返回 nil，err

}

返回 w，nil

}

```go

And this should be enough to pass the test by now:

```

go test -v -run=responseParser。

=== RUN TestOpenWeatherMap_responseParser

---通过：测试 OpenWeatherMap_responseParser（0.00 秒）

通过

好的

```go

At least we have our parser well tested. Let's structure our code to look like a library. First, we will create the methods to retrieve the weather of a city by its name and its country code, and the method that uses its latitude and longitude:

```

func（c * CurrentWeatherData）GetByGeoCoordinates（lat，lon float32）（weather * Weather，err error）{

返回 c.doRequest（

fmt.Sprintf（“http://api.openweathermap.org/data/2.5/weather q =％s，％s＆APPID =％s”，lat，lon，c.APIkey）

}

func (c *CurrentWeatherData) GetByCityAndCountryCode(city, countryCode string) (weather *Weather, err error) {

return c.doRequest(

fmt.Sprintf("http://api.openweathermap.org/data/2.5/weather?lat=%f&lon=%f&APPID=%s", city, countryCode, c.APIkey) )

}

```go

A piece of cake? Of course! Everything must be as easy as possible, and it is a sign of a good job. The complexity in this facade is to create connections to the `OpenWeatherMap` API, and control the possible errors. This problem is shared between all the Facade methods in our example, so we don't need to write more than one API call right now.

What we do is pass the URL that the REST API needs in order to return the information we desire. This is achieved by the `fmt.Sprintf` function, which formats the strings in each case. For example, to gather the data using a city name and a country code, we use the following string:

```

fmt.Sprintf("http://api.openweathermap.org/data/2.5/weather?lat=%f&lon=%f&APPID=%s", city, countryCode, c.APIkey)

```go

This takes the pre-formatted string [`openweathermap.org/api`](https://openweathermap.org/api) and formats it by replacing each `%s` specifier with the city, the `countryCode` that we introduced in the arguments, and the API key member of the `CurrentWeatherData` type.

But, we haven't set any API key! Yes, because this is a library, and the users of the library will have to use their own API keys. We are hiding the complexity of creating the URIs, and handling the errors.

Finally, the `doRequest` function is a big fish, so we will see it in detail, step by step:

```

func (o *CurrentWeatherData) doRequest(uri string) (weather *Weather, err error) {

client := &http.Client{}

=== RUN   TestTeamFlyweightFactory_GetTeam

if err != nil {

return

}

req.Header.Set("Content-Type", "application/json")

```go

First, the signature tells us that the `doRequest` method accepts a URI string, and returns a pointer to the `Weather` variable and an error. We start by creating an `http.Client` class, which will make the requests. Then, we create a request object, which will use the `GET` method, as described in the `OpenWeatherMap` webpage, and the URI we passed. If we were to use a different method, or more than one, they would have to be brought about by arguments in the signature. Nevertheless, we will use just the `GET` method, so we could hardcode it there.

Then, we check whether the request object has been created successfully, and set a header that says that the content type is a JSON:

```

resp, err := client.Do(req)

--- FAIL: TestTeamFlyweightFactory_GetTeam (0.00s)

return

}

if resp.StatusCode != 200 {

byt, errMsg := ioutil.ReadAll(resp.Body)

if errMsg == nil {

errMsg = fmt.Errorf("%s", string(byt))

}

err = fmt.Errorf("状态码为%d，中止。错误消息为:\n%s\n",resp.StatusCode, errMsg)

return

}

```go

Then we make the request, and check for errors. Because we have given names to our return types, if any error occurs, we just have to return the function, and Go will return the variable `err` and the variable `weather` in the state they were in at that precise moment.

We check the status code of the response, as we only accept 200 as a good response. If 200 isn't returned, we will create an error message with the contents of the body and the status code returned:

```

weather, err = o.responseParser(resp.Body)

resp.Body.Close()

return

}

```go

Finally, if everything goes well, we use the `responseParser` function we wrote earlier to parse the contents of Body, which is an `io.Reader` interface. Maybe you are wondering why we aren't controlling `err` from the `response parser` method. It's funny, because we are actually controlling it. `responseParser` and `doRequest` have the same return signature. Both return a `Weather` pointer and an error (if any), so we can return directly whatever the result was.

## Library created with the Facade pattern

We have the first milestone for a library for the `OpenWeatherMap` API using the facade pattern. We have hidden the complexity of accessing the `OpenWeatherMap` REST API in the `doRequest` and `responseParser` functions, and the users of our library have an easy-to-use syntax to query the API. For example, to retrieve the weather for Madrid, Spain, a user will only have to introduce arguments and an API key at the beginning:

```

weatherMap := CurrentWeatherData{*apiKey}

weather, err := weatherMap.GetByCityAndCountryCode("Madrid", "ES")

/home/mcastro/Desktop/go-design-patterns/structural/flyweight/flyweight.go:71 +0x159

t.Fatal(err)

}

fmt.Printf("马德里的温度为%f 摄氏度\n", weather.Main.Temp-273.15)

```go

The console output for the weather in Madrid at the moment of writing this chapter is the following:

```

$ 马德里的温度为 30.600006 摄氏度

```go

A typical summer day!

# Flyweight design pattern

Our next pattern is the **Flyweight** design pattern. It's very commonly used in computer graphics and the video game industry, but not so much in enterprise applications.

## Description

Flyweight is a pattern which allows sharing the state of a heavy object between many instances of some type. Imagine that you have to create and store too many objects of some heavy type that are fundamentally equal. You'll run out of memory pretty quickly. This problem can be easily solved with the Flyweight pattern, with additional help of the Factory pattern. The factory is usually in charge of encapsulating object creation, as we saw previously.

## Objectives

Thanks to the Flyweight pattern, we can share all possible states of objects in a single common object, and thus minimize object creation by using pointers to already created objects.

## Example

To give an example, we are going to simulate something that you find on betting webpages. Imagine the final match of the European championship, which is viewed by millions of people across the continent. Now imagine that we own a betting webpage, where we provide historical information about every team in Europe. This is plenty of information, which is usually stored in some distributed database, and each team has, literally, megabytes of information about their players, matches, championships, and so on.

If a million users access information about a team and a new instance of the information is created for each user querying for historical data, we will run out of memory in the blink of an eye. With our Proxy solution, we could make a cache of the *n* most recent searches to speed up queries, but if we return a clone for every team, we will still get short on memory (but faster thanks to our cache). Funny, right?

Instead, we will store each team's information just once, and we will deliver references to them to the users. So, if we face a million users trying to access information about a match, we will actually just have two teams in memory with a million pointers to the same memory direction.

## Acceptance criteria

The acceptance criteria for a Flyweight pattern must always reduce the amount of memory that is used, and must be focused primarily on this objective:

1.  We will create a `Team` struct with some basic information such as the team's name, players, historical results, and an image depicting their shield.
2.  We must ensure correct team creation (note the word *creation* here, candidate for a creational pattern), and not having duplicates.
3.  When creating the same team twice, we must have two pointers pointing to the same memory address.

## Basic structs and tests

Our `Team` struct will contain other structs inside, so a total of four structs will be created. The `Team` struct has the following signature:

```

type Team struct {

ID             uint64

Name           string

Shield         []byte

Players        []Player

}

}

```go

Each team has an ID, a name, some image in an slice of bytes representing the team's shield, a slice of players, and a slice of historical data. This way, we will have two teams' ID:

```

const (

TEAM_A = iota

TEAM_B

)

```go

We declare two constants by using the `const` and `iota` keywords. The `const` keyword simply declares that the following declarations are constants. `iota` is a untyped integer that automatically increments its value for each new constant between the parentheses. The `iota` value starts to reset to 0 when we declare `TEAM_A`, so `TEAM_A` is equal to 0\. On the `TEAM_B` variable, `iota` is incremented by one so `TEAM_B` is equal to 1\. The `iota` assignment is an elegant way to save typing when declaring constant values that doesn't need specific value (like the *Pi* constant on the `math` package).

Our `Player` and `HistoricalData` are the following:

```

type Player struct {

Name    string

Surname string

PreviousTeam uint64

Photo   []byte

}

type HistoricalData struct {

Year          uint8

LeagueResults []Match

}

```go

As you can see, we also need a `Match` struct, which is stored within `HistoricalData` struct. A `Match` struct, in this context, represents the historical result of a match:

```

type Match struct {

Date          time.Time

VisitorID     uint64

LocalID       uint64

LocalScore    byte

VisitorScore  byte

LocalShoots   uint16

VisitorShoots uint16

}

```go

This is enough to represent a team, and to fulfill *Acceptance Criteria 1*. You have probably guessed that there is a lot of information on each team, as some of the European teams have existed for more than 100 years.

For *Acceptance Criteria 2,* the word *creation* should give us some clue about how to approach this problem. We will build a factory to create and store our teams. Our Factory will consist of a map of years, including pointers to `Teams` as values, and a `GetTeam` function. Using a map will boost the team search if we know their names in advance. We will also dispose of a method to return the number of created objects, which will be called the `GetNumberOfObjects` method:

```

type teamFlyweightFactory struct {

createdTeams map[string]*Team

req, err := http.NewRequest("GET", uri, nil)

func (t *teamFlyweightFactory) GetTeam(name string) *Team {

return nil

}

}

return 0

}

```go

This is enough to write our first unit test:

```

if err != nil {

factory := teamFlyweightFactory{}

teamA1 := factory.GetTeam(TEAM_A)

if teamA1 == nil {

t.Error("TEAM_A 的指针为空")

}

teamA2 := factory.GetTeam(TEAM_A)

if teamA2 == nil {

t.Error("TEAM_A 的指针为空")

}

if teamA1 != teamA2 {

t.Error("TEAM_A 的指针不同")

}

if factory.GetNumberOfObjects() != 1 {

t.Errorf("创建的对象数量不是 1: %d\n", factory.GetNumberOfObjects())

if err != nil {

}

```go

In our test, we verify all the acceptance criteria. First we create a factory, and then ask for a pointer of `TEAM_A`. This pointer cannot be `nil`, or the test will fail.

Then we call for a second pointer to the same team. This pointer can't be nil either, and it should point to the same memory address as the previous one so we know that it has not allocated a new memory.

Finally, we should check whether the number of created teams is only one, because we have asked for the same team twice. We have two pointers but just one instance of the team. Let's run the tests:

```

$ go test -v -run=GetTeam .

=== RUN   TestTeamFlyweightFactory_GetTeam

HistoricalData []HistoricalData

flyweight_test.go:11: TEAM_A 的指针为空

flyweight_test.go:21: TEAM_A 的指针为空

flyweight_test.go:31: 创建的对象数量不是 1: 0

FAIL

panic(0x530900, 0xc0820025c0)

FAIL

```go

Well, it failed. Both pointers were nil and it has not created any object. Interestingly, the function that compares the two pointers doesn't fail; all in all, nil equals nil.

## Implementation

Our `GetTeam` method will need to scan the `map` field called `createdTeams` to make sure the queried team is already created, and return it if so. If the team wasn't created, it will have to create it and store it in the map before returning:

```

func (t *teamFlyweightFactory) GetTeam(teamID int) *Team {

if t.createdTeams[teamID] != nil {

return t.createdTeams[teamID]

}

return Team{

t.createdTeams[teamID] = &team

return t.createdTeams[teamID]

}

```go

The preceding code is very simple. If the parameter name exists in the `createdTeams` map, return the pointer. Otherwise, call a factory for team creation. This is interesting enough to stop for a second and analyze. When you use the Flyweight pattern, it is very common to have a Flyweight factory, which uses other types of creational patterns to retrieve the objects it needs.

So, the `getTeamFactory` method will give us the team we are looking for, we will store it in the map, and return it. The team factory will be able to create the two teams: `TEAM_A` and `TEAM_B`:

```

func getTeamFactory(team int) Team {

switch team {

case TEAM_B:

func (t *teamFlyweightFactory) GetNumberOfObjects() int {

ID:   2,

Name: TEAM_B,

}

default:

return Team{

ID:   1,

Name: TEAM_A,

}

}

}

```go

We are simplifying the objects' content so that we can focus on the Flyweight pattern's implementation. Okay, so we just have to define the function to retrieve the number of objects created, which is done as follows:

```

func (t *teamFlyweightFactory) GetNumberOfObjects() int {

return len(t.createdTeams)

}

```go

This was pretty easy. The `len` function returns the number of elements in an array or slice, the number of characters in a `string`, and so on. It seems that everything is done, and we can launch our tests again:

```

$ go test -v -run=GetTeam .

goroutine 5 [running]:

--- FAIL: TestTeamFlyweightFactory_GetTeam (0.00s)

panic: 分配给 nil 映射中的条目 [已恢复]

panic: 分配给 nil 映射中的条目

func TestTeamFlyweightFactory_GetTeam(t *testing.T) {

team := getTeamFactory(teamID)

/home/mcastro/Go/src/runtime/panic.go:481 +0x3f4

测试.tRunner.func1(0xc082068120)

/home/mcastro/Go/src/testing/testing.go:467 +0x199

panic(0x530900, 0xc0820025c0)

/home/mcastro/Go/src/runtime/panic.go:443 +0x4f7

/home/mcastro/go-design-patterns/structural/flyweight.(*teamFlyweightFactory).GetTeam(0xc08202fec0, 0x0, 0x0)

exit status 1

/home/mcastro/go-design-patterns/structural/flyweight.TestTeamFlyweightFactory_GetTeam(0xc082068120)

/home/mcastro/Desktop/go-design-patterns/structural/flyweight/flyweight_test.go:9 +0x61

testing.tRunner(0xc082068120, 0x666580)

/home/mcastro/Go/src/testing/testing.go:473 +0x9f

created by testing.RunTests

/home/mcastro/Go/src/testing/testing.go:582 +0x899

exit status 2

FAIL

```go

Panic! Have we forgotten something? By reading the stack trace on the panic message, we can see some addresses, some files, and it seems that the `GetTeam` method is trying to assign an entry to a nil map on *line 71* of the `flyweight.go` file. Let's look at *line 71* closely (remember, if you are writing code while following this tutorial, that the error will probably be in a different line so look closely at your own stark trace):

```

t.createdTeams[teamName] = &team

```go

Okay, this line is on the `GetTeam` method, and, when the method passes through here, it means that it had not found the team on the map-it has created it (the variable team), and is trying to assign it to the map. But the map is nil, because we haven't initialized it when creating the factory. This has a quick solution. In our test, initialize the map where we have created the factory:

```

factory := teamFlyweightFactory{

createdTeams: make(map[int]*Team,0),

}

```go

I'm sure you have seen the problem here already. If we don't have access to the package, we can initialize the variable. Well, we can make the variable public, and that's all. But this would involve every implementer necessarily knowing that they have to initialize the map, and its signature is neither convenient, or elegant. Instead, we are going to create a simple factory builder to do it for us. This is a very common approach in Go:

```

func NewTeamFactory() teamFlyweightFactory {

return teamFlyweightFactory{

createdTeams: make(map[int]*Team),

}

}

```go

So now, in the test, we replace the factory creation with a call to this function:

```

func TestTeamFlyweightFactory_GetTeam(t *testing.T) {

factory := NewTeamFactory()

...

}

```go

And we run the test again:

```

$ go test -v -run=GetTeam .

=== RUN   TestTeamFlyweightFactory_GetTeam

--- PASS: TestTeamFlyweightFactory_GetTeam (0.00s)

PASS

ok

```go

Perfect! Let's improve the test by adding a second test, just to ensure that everything will be running as expected with more volume. We are going to create a million calls to the team creation, representing a million calls from users. Then, we will simply check that the number of teams created is only two:

```

func Test_HighVolume(t *testing.T) {

factory := NewTeamFactory()

teams := make([]*Team, 500000*2)

for i := 0; i < 500000; i++ {

teams[i] = factory.GetTeam(TEAM_A)

}

for i := 500000; i < 2*500000; i++ {

teams[i] = factory.GetTeam(TEAM_B)

}

if factory.GetNumberOfObjects() != 2 {

t.Errorf("The number of objects created was not 2: %d\n",factory.GetNumberOfObjects())

}

}

```go

In this test, we retrieve `TEAM_A` and `TEAM_B` 500,000 times each to reach a million users. Then, we make sure that just two objects were created:

```

$ go test -v -run=Volume .

=== RUN   Test_HighVolume

--- PASS: Test_HighVolume (0.04s)

PASS

ok

```go

Perfect! We can even check where the pointers are pointing to, and where they are located. We will check with the first three as an example. Add these lines at the end of the last test, and run it again:

```

for i:=0; i<3; i++ {

fmt.Printf("Pointer %d points to %p and is located in %p\n", i, teams[i], &teams[i])

}

```go

In the preceding test, we use the `Printf` method to print information about pointers. The `%p` flag gives you the memory location of the object that the pointer is pointing to. If you reference the pointer by passing the `&` symbol, it will give you the direction of the pointer itself.

Run the test again with the same command; you will see three new lines in the output with information similar to the following:

```

Pointer 0 points to 0xc082846000 and is located in 0xc082076000

Pointer 1 points to 0xc082846000 and is located in 0xc082076008

Pointer 2 points to 0xc082846000 and is located in 0xc082076010

```

它告诉我们的是，地图中的前三个位置指向相同的位置，但实际上我们有三个不同的指针，它们实际上比我们的团队对象轻得多。

## 那么单例模式和享元模式有什么区别呢？

嗯，差异微妙，但确实存在。使用单例模式，我们确保只创建一次相同的类型。此外，单例模式是一种创建模式。对于享元模式，它是一种结构模式，我们不关心对象是如何创建的，而是关心如何以轻量的方式构造一个类型来包含重的信息。我们谈论的结构是我们的例子中的`map[int]*Team`结构。在这里，我们真的不关心如何创建对象；我们只是为它编写了一个简单的`getTeamFactory`方法。我们非常重视拥有一个轻量级的结构来容纳可共享的对象（或对象），在这种情况下是地图。

# Summary

我们已经看到了几种组织代码结构的模式。结构模式关心如何创建对象，或者它们如何进行业务（我们将在行为模式中看到这一点）。

不要因为混合了几种模式而感到困惑。如果您严格遵循每种模式的目标，您很容易混合六七种模式。只要记住，过度设计和根本不设计一样糟糕。我记得有一天晚上我做了一个负载均衡器的原型，经过两个小时的疯狂过度设计的代码后，我的脑子里一团糟，我宁愿重新开始。

在下一章中，我们将看到行为模式。它们更加复杂，通常使用结构模式和创建模式来实现它们的目标，但我相信读者会觉得它们非常具有挑战性和有趣。
