# 第四章 结构型模式 - 代理、外观、装饰器和享元设计模式

通过本章，我们将完成结构型模式的学习。我们留了一些最复杂的模式到后面，这样你可以更习惯设计模式的机制，以及 Go 语言的特点。

在本章中，我们将编写一个用于访问数据库的缓存、一个收集天气数据的库、一个具有运行时中间件的服务器，并讨论一种通过在类型值之间保存可共享状态来节省内存的方法。

# 代理设计模式

我们将以代理模式开始结构型模式的最后一章。这是一个简单的模式，只需付出很少的努力就能提供有趣的功能和可能性。

## 描述

代理模式通常封装一个对象以隐藏其一些特性。这些特性可能包括它是一个远程对象（远程代理）、一个非常重的对象，如非常大的图像或兆字节的数据库转储（虚拟代理），或者一个受限访问的对象（保护代理）。

## 目标

代理模式的可能性很多，但总的来说，它们都试图提供以下相同的功能：

+   在代理后面隐藏一个对象，以便可以隐藏、限制等功能。

+   提供一个易于工作的新抽象层，并且可以轻松更改。

## 示例

对于我们的示例，我们将创建一个远程代理，它将成为访问数据库前的对象缓存。让我们想象我们有一个包含许多用户的数据库，但每次我们想要获取用户信息时，我们不会直接访问数据库，而是将用户信息存储在一个代理模式中的**先进先出**（**FIFO**）堆栈中（FIFO 是一种表示当缓存需要清空时，它将删除最先进入的第一个对象的方式）。

## 接受标准

我们将使用我们的代理模式封装一个虚拟的数据库，用一个切片表示，然后该模式必须遵守以下接受标准：

1.  所有对用户数据库的访问都将通过代理类型进行。

1.  将`n`个最近用户堆叠保存在代理中。

1.  如果用户已经在堆叠中存在，它将不会查询数据库，而是返回存储的记录。

1.  如果查询的用户不在堆栈中，它将查询数据库，如果堆栈已满，则删除最旧的用户，存储新的用户，并返回它。

## 单元测试

自 Go 1.7 版本以来，我们可以通过使用闭包在测试中嵌入测试，这样我们可以以更易于阅读的方式对它们进行分组，并减少`Test_`函数的数量。请参阅第一章，*准备... 稳定... 开始！*了解如果当前版本低于 1.7 版本，如何安装 Go 的新版本。

此模式的类型将是代理用户和用户列表结构以及数据库和 Proxy 将实现的 `UserFinder` 接口。这是关键，因为代理必须实现与它试图包装的类型相同的接口：

```go
type UserFinder interface { 
  FindUser(id int32) (User, error) 
} 

```

`UserFinder` 是数据库和 Proxy 实现的接口。`User` 是一个具有名为 `ID` 的成员的类型，其类型为 `int32`：

```go
type User struct { 
  ID int32 
} 

```

最后，`UserList` 是用户切片的类型。考虑以下语法：

```go
type UserList []User 

```

如果你问为什么我们不直接使用用户切片，答案是：通过这种方式声明用户序列，我们可以实现 `UserFinder` 接口，但如果我们使用切片，则不能。

最后，`Proxy` 类型，称为 `UserListProxy`，将由 `UserList` 切片组成，这将是我们的数据库表示。`StackCache` 成员也将是 `UserList` 类型，为了简单起见，`StackCapacity` 将给我们的栈赋予我们想要的大小。

为了本教程的目的，我们将稍微作弊一下，在名为 `DidDidLastSearchUsedCache` 的字段上声明一个布尔状态，该状态将保留最后一次执行搜索是否使用了缓存，或者是否访问了数据库：

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

`UserListProxy` 类型将缓存最多 `StackCapacity` 个用户，并在达到此限制时旋转缓存。`StackCache` 成员将从 `SomeDatabase` 类型的对象中填充。

第一次测试被称为 `TestUserListProxy`，并列在下面：

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

前面的测试创建了一个包含一百万个随机名称的用户列表。为此，我们通过调用带有某些常量种子的 `Seed()` 函数来喂养随机数生成器，这样我们的随机结果也是恒定的；用户 ID 就是根据这个生成的。可能会有一些重复，但这满足了我们的目的。

接下来，我们需要一个具有对 `someDatabase` 的引用的代理，这是我们刚刚创建的：

```go
proxy := UserListProxy{ 
  SomeDatabase:  &someDatabase, 
  StackCapacity:  2, 
  StackCache: UserList{}, 
} 

```

到目前为止，我们有一个由一百万个用户组成的模拟数据库和一个大小为 2 的 FIFO 栈组成的 `proxy` 对象。现在我们将从 `someDatabase` 中获取三个随机 ID，用于我们的栈：

```go
knownIDs := [3]int32 {someDatabase[3].ID, someDatabase[4].ID,someDatabase[5].ID} 

```

我们从切片中取了第四、第五和第六个 ID（记住，数组和切片从 0 开始，所以索引 3 实际上是切片中的第四个位置）。

这将是启动嵌入式测试之前的起点。为了创建一个嵌入式测试，我们必须调用 `testing.T` 指针的 `Run` 方法，并带有描述和具有 `func(t *testing.T)` 签名的闭包：

```go
t.Run("FindUser - Empty cache", func(t *testing.T) { 
  user, err := proxy.FindUser(knownIDs[0]) 
  if err != nil { 
    t.Fatal(err) 
  } 

FindUser - Empty cache. Then we define our closure. First it tries to find a user with a known ID, and checks for errors. As the description implies, the cache is empty at this point, and the user will have to be retrieved from the someDatabase array:
```

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

最后，我们检查返回的用户是否与 `knownIDs` 切片索引 0 处的预期用户 ID 相同，并且代理缓存现在的大小为 1。代理成员 `DidLastSearchUsedCache` 的状态必须不是 `true`，否则我们将不会通过测试。记住，这个成员告诉我们最后一次搜索是否是从表示数据库的切片中检索的，还是从缓存中检索的。

代理模式的第二个嵌入式测试是请求之前相同的用户，现在必须从缓存中返回。这与之前的测试非常相似，但现在我们必须检查用户是否是从缓存中返回的：

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

因此，我们再次请求第一个已知的 ID。在这次搜索之后，代理缓存必须保持大小为 1，而这次 `DidLastSearchUsedCache` 成员必须是 true，否则测试将失败。

最后一个测试将使 `proxy` 类型的 `StackCache` 数组溢出。我们将搜索两个新的用户，我们的 `proxy` 类型将不得不从数据库中检索这些用户。我们的栈大小为 2，因此它必须删除第一个用户以为第二个和第三个用户分配空间：

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

我们已经检索了前三个用户。我们不会检查错误，因为这是之前测试的目的。重要的是要记住，没有必要过度测试你的代码。如果这里有任何错误，它将在之前的测试中出现。此外，我们已经检查了 `user2` 和 `user3` 查询没有使用缓存；它们还不应该被存储在那里。

现在我们将在代理中查找 `user1` 查询。它不应该存在，因为栈的大小为 2，而 `user1` 是第一个进入的，因此也是第一个出去的：

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

无论我们请求多少用户，我们的缓存大小都不会超过我们配置的大小。

最后，我们将再次遍历存储在缓存中的用户，并将它们与最后查询的两个用户进行比较。这样，我们将检查只有那些用户被存储在缓存中。两者都必须在上面找到：

```go
  for _, v := range proxy.StackCache { 
    if v != user2 && v != user3 { 
      t.Error("A non expected user was found on the cache") 
    } 
  } 
} 

```

现在运行测试应该会给出一些错误，就像往常一样。我们现在就运行它们：

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

因此，让我们实现 `FindUser` 方法以作为我们的代理。

## 实现

在我们的代理中，`FindUser` 方法将在缓存列表中搜索指定的 ID。如果找到了，它将返回该 ID。如果没有找到，它将在数据库中搜索。最后，如果它不在数据库列表中，它将返回一个错误。

如果你还记得，我们的代理模式由两种 `UserList` 类型（其中一个是指针）组成，实际上它们是 `User` 类型的切片。我们将在 `User` 类型中实现一个 `FindUser` 方法，顺便说一下，它的签名与 `UserFinder` 接口相同：

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

`UserList` 切片中的 `FindUser` 方法将会遍历列表以尝试找到与 `id` 参数相同的用户，如果找不到，则返回错误。

你可能想知道为什么指针 `t` 在括号之间。这是在访问其索引之前取消引用底层数组。没有它，你将遇到编译错误，因为编译器试图在取消引用指针之前搜索索引。

因此，代理 `FindUser` 方法的第一部分可以写成如下：

```go
func (u *UserListProxy) FindUser(id int32) (User, error) { 
  user, err := u.StackCache.FindUser(id) 
  if err == nil { 
    fmt.Println("Returning user from cache") 
    u.DidLastSearchUsedCache = true 
    return user, nil 
  } 

```

我们使用前面的方法在`StackCache`成员中搜索用户。如果找到，错误将为 nil，因此我们检查这一点以向控制台打印消息，将`DidLastSearchUsedCache`的状态更改为`true`，以便测试可以检查用户是否从缓存中检索，最后返回用户。

因此，如果错误不为 nil，这意味着它无法在栈中找到用户。所以下一步是搜索数据库：

```go
  user, err = u.SomeDatabase.FindUser(id) 
  if err != nil { 
    return User{}, err 
  } 

```

在这种情况下，我们可以重用我们为`UserList`数据库编写的`FindUser`方法，因为在这个示例中，两者具有相同的类型。再次强调，它会在`UserList`切片表示的数据库中搜索用户，但在这个例子中，如果找不到用户，它将返回`UserList`中生成的错误。

当找到用户（`err`为 nil）时，我们必须将用户添加到栈中。为此，我们编写了一个专门的私有方法，该方法接收一个类型为`UserListProxy`的指针：

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

`addUserToStack`方法接受用户参数，并将其就地添加到栈中。如果栈已满，则在添加之前会移除其中的第一个元素。我们为此还编写了一个`addUser`方法来帮助。因此，现在在`FindUser`方法中，我们只需添加一行：

```go
u.addUserToStack(user) 

```

这会将新用户添加到栈中，如果需要，则移除最后一个。

最后，我们只需返回栈中的新用户，并在`DidLastSearchUsedCache`变量上设置适当的值。我们还向控制台写入一条消息，以帮助测试过程：

```go
  fmt.Println("Returning user from database") 
  u.DidLastSearchUsedCache = false 
  return user, nil 
} 

```

这样，我们就有了足够的测试通过：

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

你可以从前面的消息中看到，我们的代理工作得非常完美。它从数据库中返回了第一次搜索结果。然后，当我们再次搜索同一用户时，它使用缓存。最后，我们创建了一个新的测试，调用三个不同的用户，并且我们可以通过查看控制台输出观察到，只有第一个是从缓存中返回的，而其他两个是从数据库中检索的。

## 代理操作

将代理包装在需要一些中间操作的类型周围，例如向用户提供授权或提供数据库访问，就像在我们的例子中一样。

我们的例子是一个很好的方法，可以将应用程序需求与数据库需求分开。如果我们的应用程序过多地访问数据库，解决方案不在于你的数据库。记住，代理使用与它包装的类型相同的接口，对于用户来说，两者之间不应该有任何区别。

# 装饰器设计模式

我们将继续本章，介绍代理模式的“大哥”，也许是最强大的设计模式之一。**装饰器**模式相当简单，但例如，当与遗留代码一起工作时，它提供了很多好处。

## 描述

装饰者设计模式允许你在不实际接触现有类型的情况下，为其添加更多功能特性。这是如何实现的呢？嗯，它使用了一种类似于 *套娃* 的方法，你有一个小娃娃可以放在形状相同但更大的娃娃里面，以此类推。

装饰者类型实现了被装饰类型的相同接口，并在其成员中存储该类型的实例。这样，你可以通过简单地存储旧装饰者在新装饰者的字段中，来堆叠尽可能多的装饰者（娃娃）。

## 目标

当你考虑在不破坏任何东西的风险下扩展遗留代码时，你应该首先想到装饰者模式。这是一种处理这个特定问题的非常强大的方法。

装饰者模式非常强大的另一个不同领域可能并不明显，尽管在创建基于用户输入、偏好或类似输入的具有许多功能类型时，它会显现出来。就像瑞士军刀一样，你有一个基础类型（刀的框架），然后从这里展开其功能。

那么，我们究竟在什么时候会使用装饰者模式呢？回答这个问题：

+   当你需要向某些代码添加功能，而你无法访问这些代码，或者你不想修改以避免对代码产生负面影响，并遵循开闭原则（如遗留代码）

+   当你想动态创建或修改对象的功能，而功能数量未知且可能快速增长时

## 示例

在我们的示例中，我们将准备一个 `Pizza` 类型，其中核心是披萨，成分是装饰类型。我们将为我们的披萨添加一些成分——洋葱和肉。

## 接受标准

装饰者模式的接受标准是拥有一个公共接口和一个核心类型，即所有层都将构建在其上的类型：

+   我们必须有一个所有装饰者都将实现的主体接口。这个接口将被称为 `IngredientAdd`，它将有一个 `AddIngredient() string` 方法。

+   我们必须有一个核心 `PizzaDecorator` 类型（装饰者），我们将向其中添加成分。

+   我们必须有一个名为 "onion" 的成分实现相同的 `IngredientAdd` 接口，该接口将字符串 `onion` 添加到返回的披萨中。

+   我们必须有一个成分 "meat" 实现了 `IngredientAdd` 接口，该接口将字符串 `meat` 添加到返回的披萨中。

+   当在顶层对象上调用 `AddIngredient` 方法时，它必须返回一个完全装饰的 `pizza`，文本为 `包含以下成分的披萨：meat, onion`。

## 单元测试

为了启动我们的单元测试，我们必须首先创建符合接受标准的基结构。首先，所有装饰类型必须实现的接口如下：

```go
type IngredientAdd interface { 
  AddIngredient() (string, error) 
} 

```

以下代码定义了 `PizzaDecorator` 类型，它必须包含 `IngredientAdd`，并且也实现了 `IngredientAdd`：

```go
type PizzaDecorator struct{ 
  Ingredient IngredientAdd 
} 

func (p *PizzaDecorator) AddIngredient() (string, error) { 
  return "", errors.New("Not implemented yet") 
} 

```

`Meat` 类型的定义将非常类似于 `PizzaDecorator` 结构体的定义：

```go
type Meat struct { 
  Ingredient IngredientAdd 
} 

func (m *Meat) AddIngredient() (string, error) { 
  return "", errors.New("Not implemented yet") 
} 

```

现在我们以类似的方式定义 `Onion` 结构体：

```go
type Onion struct { 
  Ingredient IngredientAdd 
} 

func (o *Onion) AddIngredient() (string, error) { 
  return "", errors.New("Not implemented yet") 
}  

```

这就足够实现第一个单元测试，并允许编译器在没有编译错误的情况下运行它们：

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

现在它必须无问题地编译，因此我们可以检查测试是否失败：

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

我们的第一项测试已经完成，我们可以看到 `PizzaDecorator` 结构体还没有返回任何内容，这就是为什么它失败了。现在我们可以继续到 `Onion` 类型。`Onion` 类型的测试与 `Pizza` 装饰器的测试非常相似，但我们还必须确保我们实际上是将配料添加到 `IngredientAdd` 方法中，而不是一个空指针：

```go
func TestOnion_AddIngredient(t *testing.T) { 
  onion := &Onion{} 
  onionResult, err := onion.AddIngredient() 
  if err == nil { 
    t.Errorf("When calling AddIngredient on the onion decorator without" + "an IngredientAdd on its Ingredient field must return an error, not a string with '%s'", onionResult) 
  } 

```

前一个测试的前半部分检查了在将没有任何 `IngredientAdd` 方法传递给 `Onion` 结构体初始化器时返回的错误。由于没有披萨可以添加配料，必须返回一个错误：

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

`Onion` 类型测试的第二部分实际上将 `PizzaDecorator` 结构体传递给初始化器。然后，我们检查没有返回错误，并且返回的字符串中是否包含单词 `onion`。这样，我们可以确保洋葱已经添加到披萨中。

最后，对于 `Onion` 类型，这个测试的当前实现下的控制台输出将是以下内容：

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

`meat` 配料完全相同，但我们将其类型改为肉而不是洋葱：

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

最后，我们必须检查完整的堆栈测试。用洋葱和肉制作披萨必须返回文本 `Pizza with the following ingredients: meat, onion`：

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

我们的测试创建了一个名为 `pizza` 的变量，它就像套娃一样，嵌套了 `IngredientAdd` 方法的类型在几个层级中。调用 `AddIngredient` 方法将执行 "onion" 层的方法，然后执行 "meat" 层的方法，最后执行 `PizzaDecorator` 结构体的方法。在确认没有返回错误后，我们检查返回的文本是否符合 *验收标准 5* 的需求。测试使用以下命令运行：

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

从前面的输出中，我们可以看到，现在测试为我们的装饰类型返回了一个空字符串。这当然是因为还没有进行任何实现。这是最后一个测试，用于检查完全装饰的实现。那么，让我们仔细看看实现。

## 实现

我们将开始实现 `PizzaDecorator` 类型。它的作用是提供完整披萨的初始文本：

```go
type PizzaDecorator struct { 
  Ingredient IngredientAdd 
} 

func (p *PizzaDecorator) AddIngredient() (string, error) { 
  return "Pizza with the following ingredients:", nil 
} 

```

在 `AddIngredient` 方法的返回值上做单行更改就足以通过测试：

```go
go test -v -run=TestPizzaDecorator_Add .
=== RUN   TestPizzaDecorator_AddIngredient
--- PASS: TestPizzaDecorator_AddIngredient (0.00s)
PASS
ok

```

接下来是 `Onion` 结构体的实现，我们必须取 `IngredientAdd` 返回字符串的开头，并在其末尾添加单词 `onion`，以便返回一个组合披萨：

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

首先检查我们确实有一个指向`IngredientAdd`的指针，我们使用内部`IngredientAdd`的内容，并检查是否有错误。如果没有错误发生，我们将收到一个由这个内容、一个空格和单词`onion`（以及没有错误）组成的新字符串。看起来足够好，可以运行测试：

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

下面是它们的测试执行：

```go
go test -v -run=TestMeat_AddIngredient .
=== RUN   TestMeat_AddIngredient
--- PASS: TestMeat_AddIngredient (0.00s)
PASS
ok

```

好的。所以，现在所有组件都需要单独测试。如果一切正常，*完整堆叠*解决方案的测试必须顺利通过：

```go
go test -v -run=TestPizzaDecorator_FullStack .
=== RUN   TestPizzaDecorator_FullStack
--- PASS: TestPizzaDecorator_FullStack (0.00s)
decorator_test.go:92: Pizza with the following ingredients: meat, onion,
PASS
ok

```

太棒了！使用装饰者模式，我们可以连续堆叠`IngredientAdds`，这些`IngredientAdds`会调用它们的内部指针来向`PizzaDecorator`添加功能。我们也没有触及核心类型，也没有修改或实现新事物。所有的新功能都是由外部类型实现的。

## 真实生活中的例子 - 服务器中间件

到现在为止，你应该已经理解了装饰者模式的工作原理。现在我们可以尝试一个更高级的例子，使用我们在适配器模式部分设计的 HTTP 小服务器。你了解到可以通过使用`http`包并实现`http.Handler`接口来创建 HTTP 服务器。这个接口只有一个方法，叫做`ServeHTTP(http.ResponseWriter, http.Request)`。我们能否使用装饰者模式向服务器添加更多功能？当然可以！

我们将向这个服务器添加几个组件。首先，我们将记录所有连接到它的连接到`io.Writer`接口（为了简单起见，我们将使用`os.Stdout`接口的`io.Writer`实现，以便输出到控制台）。第二个组件将为服务器发出的每个请求添加基本的 HTTP 身份验证。如果身份验证通过，将显示`Hello Decorator!`消息。最后，用户将能够选择在服务器中想要的装饰项目数量，服务器将在运行时进行结构和创建。

### 从通用接口开始，http.Handler

我们已经有了将要使用嵌套类型进行装饰的通用接口。我们首先需要创建我们的核心类型，它将是一个返回句子`Hello Decorator!`的`Handler`：

```go
type MyServer struct{} 

func (m *MyServer) ServeHTTP(w http.ResponseWriter, r *http.Request) { 
  fmt.Fprintln(w, "Hello Decorator!") 
} 

```

这个处理器可以被分配给`http.Handle`方法来定义我们的第一个端点。现在让我们通过创建包的`main`函数并向它发送`GET`请求来检查这一点：

```go
func main() { 
  http.Handle("/", &MyServer{}) 

  log.Fatal(http.ListenAndServe(":8080", nil)) 
} 

```

使用终端执行服务器，运行`**go run main.go**`命令。然后，打开一个新的终端来发起`GET`请求。我们将使用`curl`命令来发起请求：

```go
$ curl http://localhost:8080
Hello Decorator!

```

我们已经完成了装饰服务器的第一里程碑。下一步是为它添加日志功能。为此，我们必须在一个新类型中实现`http.Handler`接口，如下所示：

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

我们称这种类型为 `LoggerServer`。正如你所见，它不仅存储了一个 `Handler`，还存储了一个 `io.Writer` 来写入日志的输出。我们实现的 `ServeHTTP` 方法打印请求 URI、主机、内容长度和使用的 `io.Writer` 方法。打印完成后，它调用其内部 `Handler` 字段的 `ServeHTTP` 函数。

我们可以用这个 `LoggerMiddleware` 装饰 `MyServer`：

```go
func main() { 
  http.Handle("/", &LoggerServer{ 
    LogWriter:os.Stdout, 
    Handler:&MyServer{}, 
  }) 

  log.Fatal(http.ListenAndServe(":8080", nil)) 
} 

```

现在运行 `**curl**` 命令：

```go
$ curl http://localhost:8080
Hello Decorator!

```

我们的 **curl** 命令返回相同的信息，但如果你查看运行 Go 应用的终端，你可以看到日志：

```go
$ go run server_decorator.go
Request URI: /
Host: localhost:8080
Content Length: 0
Method: GET

```

我们在实际上没有修改它的情况下，用日志功能装饰了 `MyServer`。我们能否用身份验证做到同样的事情？当然可以！在记录请求后，我们将通过以下方式使用 **HTTP Basic Authentication** 进行身份验证：

```go
type BasicAuthMiddleware struct { 
  Handler  http.Handler 
  User     string 
  Password string 
} 

```

**BasicAuthMiddleware** 中间件存储了三个字段——一个用于装饰的处理程序，就像之前的中间件一样，一个用户和一个密码，这些将作为访问服务器上内容的唯一授权。`decorating` 方法的实现将按以下步骤进行：

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

在前面的实现中，我们使用 `http.Request` 中的 `BasicAuth` 方法自动从请求中检索用户和密码，以及解析动作的 `ok/ko`。然后我们检查解析是否正确（如果错误，向请求者返回消息，并完成请求）。如果没有检测到任何问题，我们检查用户名和密码是否与存储在 `BasicAuthMiddleware` 中的匹配。如果凭据有效，我们将调用装饰的类型（我们的服务器），但如果凭据无效，我们将收到“用户名或密码错误”的消息，并完成请求。

现在，我们需要为用户提供一种方式来选择不同类型的服务器。我们将在主函数中检索用户输入数据。我们将有三个选项可供选择：

+   简单服务器

+   带日志的服务器

+   带日志和身份验证的服务器

我们必须使用 `Fscanf` 函数从用户那里获取输入：

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

`Fscanf` 函数需要一个 `io.Reader` 实现者作为第一个参数（这将是控制台中的输入），它从用户选择的服务器中获取服务器。我们将传递 `os.Stdin` 作为 `io.Reader` 接口以检索用户输入。然后，我们将写入要解析的数据类型。`%d` 说明符指的是一个整数。最后，我们将写入存储解析输入的内存方向，在这种情况下，是 `selection` 变量的内存位置。

一旦用户选择了选项，我们可以在运行时装饰基本服务器，切换到所选选项：

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

第一个选项将由默认的`switch`选项处理--一个普通的`MyServer`。在第二个选项的情况下，我们使用日志装饰一个普通服务器。第三个选项更复杂一些--我们再次使用`Fscanf`请求用户输入用户名和密码。请注意，你可以扫描多个输入，正如我们这样做来获取用户名和密码。然后，我们取基本服务器，用身份验证装饰它，最后，用日志装饰。

如果你遵循选项三嵌套类型的缩进，请求将通过记录器，然后是身份验证中间件，最后，如果一切正常，将通过`MyServer`参数。请求将遵循相同的路径。

主函数的末尾获取装饰后的处理程序，并在`8080`端口启动服务器：

```go
http.Handle("/", mySuperServer) 
log.Fatal(http.ListenAndServe(":8080", nil)) 

```

那么，让我们使用第三个选项来启动服务器：

```go
$go run server_decorator.go 
Enter the server type number you want to launch from the following: 
1.- Plain server 
2.- Server with logging 
3.- Server with logging and authentication 

Enter user and password separated by a space 
mario castro

```

我们将首先通过选择第一个选项来测试普通服务器。使用命令**go run server_decorator.go**运行服务器，并选择第一个选项。然后，在另一个终端中，使用 curl 运行基本请求，如下所示：

```go
$ curl http://localhost:8080
Error trying to retrieve data from Basic auth

```

哎呀！它没有给我们权限。我们没有传递任何用户名和密码，所以它告诉我们无法继续。让我们尝试使用一些随机的用户名和密码：

```go
$ curl -u no:correct http://localhost:8080
User or password incorrect

```

没有权限！我们也可以检查在终端中启动服务器的地方以及每个请求被记录的地方：

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

我们到了！我们的请求也被记录了，服务器也授予了我们权限。现在我们可以通过编写更多的中间件来装饰服务器的功能，尽可能多地改进我们的服务器。

## 关于 Go 的结构化类型的一些话

Go 有一个大多数人一开始都不喜欢的特性--结构化类型。这是当你的结构定义了你的类型而不需要明确写出它的时候。例如，当你实现一个接口时，你不需要明确写出你实际上正在实现它，与 Java 等语言不同，在这些语言中你必须写出关键字`implements`。如果你的方法遵循接口的签名，你实际上就是在实现接口。这也可能导致意外实现接口，这可能会引起难以追踪的错误，但这种情况非常不可能。

然而，结构化类型也允许你在定义实现者之后定义接口。想象一下以下`MyPrinter`结构体：

```go
type MyPrinter struct{} 
func(m *MyPrinter)Print(){ 
  println("Hello") 
} 

```

想象一下，我们已经与`MyPrinter`类型工作了数月，但它没有实现任何接口，所以它不可能是一个装饰器模式的可能候选者，或者也许它可以？如果我们几个月后编写了一个与它的`Print`方法匹配的接口呢？考虑以下代码片段：

```go
type Printer interface { 
  Print() 
} 

```

实际上，它实现了`Printer`接口，我们可以用它来创建一个装饰器解决方案。

结构化类型在编写程序时提供了很大的灵活性。如果你不确定一个类型是否应该是接口的一部分，你可以先将其留出，并在你完全确定后再添加接口。这样，你可以非常容易地装饰类型，并且对源代码的修改很小。

## 总结装饰器设计模式 - 代理与装饰器

你可能想知道，装饰器模式和代理模式有什么区别？在装饰器模式中，我们动态地装饰一个类型。这意味着装饰可能存在也可能不存在，或者它可能由一个或多个类型组成。如果你记得，代理模式以类似的方式封装类型，但它是在编译时进行的，更像是一种访问某些类型的方式。

同时，装饰器可能实现被装饰类型所实现的整个接口，也可能不实现。因此，你可以有一个包含 10 个方法的接口和一个只实现其中之一的方法的装饰器，它仍然有效。对装饰器未实现的方法的调用将被传递给被装饰的类型。这是一个非常强大的功能，但如果你忘记实现任何接口方法，在运行时可能会出现不期望的行为。

在这个方面，你可能认为代理模式不太灵活，确实如此。但装饰器模式较弱，因为你可能会在运行时遇到错误，而通过使用代理模式，你可以在编译时避免这些错误。只需记住，装饰器通常用于在运行时向对象添加功能，比如在我们的 Web 服务器中。这是在需求和为了实现它而愿意牺牲的东西之间的折衷。

# 外观设计模式

本章我们将要看到的下一个模式是外观模式。当我们讨论代理模式时，你已经了解到它是一种封装类型以隐藏其部分复杂特性的方法。想象一下，我们将许多代理组合在一个单独的点，比如一个文件或一个库。这可以被视为一个外观模式。

## 描述

在建筑术语中，外观是隐藏建筑房间和走廊的前墙。它保护其居民免受寒冷和雨水的侵袭，并为他们提供隐私。它组织和划分住宅。

外观设计模式在代码中做的是同样的事情。它保护代码免受不想要的访问，组织一些调用，并从用户那里隐藏复杂性范围。

## 目标

当你想隐藏某些任务的复杂性时，你会使用外观模式，尤其是当这些任务大多数共享一些实用工具（如 API 中的认证）时。库是一种外观形式，其中有人必须为开发者提供一些方法，以便以友好的方式完成某些事情。这样，如果开发者需要使用你的库，他/她不需要知道所有内部任务来获取他/她想要的结果。

因此，你会在以下场景中使用外观设计模式：

+   当你想要降低我们代码中某些部分的复杂性时。你通过提供一个更易于使用的方法来隐藏这种复杂性。

+   当你想要在一个地方组合相关的操作时。

+   当你想要构建一个库，以便其他人可以使用你的产品而无需担心它是如何工作的。

## 示例

以为例，我们将迈出编写我们自己的库的第一步，该库可以访问 `OpenWeatherMaps` 服务。如果你不熟悉 `OpenWeatherMap` 服务，它是一个提供实时天气信息以及历史数据的 HTTP 服务。**HTTP REST** API 非常易于使用，并将是一个很好的示例，说明如何创建一个外观模式来隐藏 REST 服务背后的网络连接复杂性。

## 验收标准

`OpenWeatherMap` API 提供了大量信息，因此我们将专注于通过使用其纬度和经度值来获取某个地理位置上的一个城市的实时天气数据。以下是这个设计模式的必要条件和验收标准：

1.  提供一个单一的类型来访问数据。从 `OpenWeatherMap` 服务检索的所有信息都将通过它传递。

1.  创建一种获取某个国家某个城市天气数据的方法。

1.  创建一种获取某个纬度和经度位置天气数据的方法。

1.  只有第二点和第三点必须在外部包中可见；其他所有内容都必须隐藏（包括所有连接相关数据）。

## 单元测试

要开始我们的 API 外观，我们需要一个具有 *验收标准 2* 和 *验收标准 3* 中要求的方法的接口：

```go
type CurrentWeatherDataRetriever interface { 
  GetByCityAndCountryCode(city, countryCode string) (Weather, error) 
  GetByGeoCoordinates(lat, lon float32) (Weather, error) 
} 

```

我们将把 *验收标准 2* 命名为 `GetByCityAndCountryCode`；我们还需要一个城市名称和一个字符串格式的国家代码。国家代码是两个字符的代码，代表世界国家的 **国际标准化组织** (**ISO**) 名称。它返回一个 `Weather` 值，我们将在稍后定义，如果出现问题，它将返回一个错误。

*验收标准 3* 将被命名为 `GetByGeoCoordinates`，它需要 `float32` 格式的纬度和经度值。它还将返回一个 `Weather` 值和一个错误。`Weather` 值将根据 `OpenWeatherMap` API 返回的 JSON 定义。你可以在网页 [`openweathermap.org/current#current_JSON`](http://openweathermap.org/current#current_JSON) 上找到这个 JSON 的描述。

如果你查看 JSON 定义，它具有以下类型：

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

这是一个相当长的结构体，但我们包含了响应可能包含的一切。这个结构体被称为`Weather`，因为它由一个 ID、一个名称和一个代码（`Cod`）以及几个匿名结构体组成，这些结构体是：`Coord`、`Weather`、`Base`、`Main`、`Wind`、`Clouds`、`Rain`、`Dt`和`Sys`。我们可以通过给它们一个名字将这两个匿名结构体写在外部的`Weather`结构体之外，但只有在我们需要单独处理它们时才有用。

在我们的`Weather`结构体中的每个成员和结构体之后，你都可以找到一个` `` `json:"something"` `` `行。这在区分 JSON 键名和你的成员名时很有用。如果 JSON 键是`something`，我们不必强迫我们的成员被称为`something`。例如，我们的 ID 成员在 JSON 响应中将被称为`id`。

为什么我们不给我们类型中的 JSON 键命名呢？好吧，如果你的类型字段是小写的，`encoding/json`包将无法正确解析它们。此外，最后一个注解为我们提供了一定的灵活性，不仅在于更改成员的名称，还在于如果我们不需要某些键，我们可以省略它们，以下是其签名：

```go
`json:"something,omitempty" 

```

在末尾加上`omitempty`，如果这个键不在 JSON 键的字节表示中，解析不会失败。

好的，我们的验收标准 1 要求 API 的单一点访问。这将被称为`CurrentWeatherData`：

```go
type CurrentWeatherData struct { 
  APIkey string 
} 

```

`CurrentWeatherData`类型有一个作为公共成员的 API 密钥来工作。这是因为你必须成为`OpenWeatherMap`的注册用户才能享受他们的服务。请参阅`OpenWeatherMap` API 的网页以获取有关如何获取 API 密钥的文档。在我们的示例中我们不需要它，因为我们不会进行集成测试。

我们需要模拟数据，以便我们可以编写一个`mock`函数来检索数据。在发送 HTTP 请求时，响应以`io.Reader`形式包含在一个名为`body`的成员中。我们已经与实现了`io.Reader`接口的类型一起工作过，所以这应该对你来说很熟悉。我们的`mock`函数看起来是这样的：

```go
 func getMockData() io.Reader { 
  response := `{
    "coord":{"lon":-3.7,"lat":40.42},"weather : [{"id":803,"main":"Clouds","description":"broken clouds","icon":"04n"}],"base":"stations","main":{"temp":303.56,"pressure":1016.46,"humidity":26.8,"temp_min":300.95,"temp_max":305.93},"wind":{"speed":3.17,"deg":151.001},"rain":{"3h":0.0075},"clouds":{"all":68},"dt":1471295823,"sys":{"type":3,"id":1442829648,"message":0.0278,"country":"ES","sunrise":1471238808,"sunset":1471288232},"id":3117735,"name":"Madrid","cod":200}` 

  r := bytes.NewReader([]byte(response)) 
  return r 
} 

```

这个模拟数据是通过使用 API 密钥向`OpenWeatherMap`发出请求生成的。`response`变量是一个包含 JSON 响应的字符串。仔细观察用来打开和关闭字符串的重音符（`` ` ``）。这样，你可以使用任意多的引号而不会出现任何问题。

进一步来说，我们在 bytes 包中使用了一个特殊函数`NewReader`，它接受一个字节数组（我们通过将类型从字符串转换创建），并返回一个包含数组内容的`io.Reader`实现者。这完美地模仿了 HTTP 响应的`Body`成员。

我们将编写一个测试来尝试`response parser`。两种方法返回相同类型，所以我们可以为两者使用相同的`JSON parser`：

```go
func TestOpenWeatherMap_responseParser(t *testing.T) { 
  r := getMockData() 
  openWeatherMap := CurrentWeatherData{APIkey: ""} 

  weather, err := openWeatherMap.responseParser(r) 
  if err != nil { 
    t.Fatal(err) 
  } 

  if weather.ID != 3117735 { 
    t.Errorf("Madrid id is 3117735, not %d\n", weather.ID) 
  } 
} 

```

在前面的测试中，我们首先请求了一些模拟数据，我们将其存储在变量 `r` 中。后来，我们创建了一个名为 `openWeatherMap` 的 `CurrentWeatherData` 类型。最后，我们为提供的 `io.Reader` 接口请求了天气值，我们将其存储在变量 `weather` 中。在检查错误后，我们确保 ID 与我们从 `getMockData` 方法获得的模拟数据中的 ID 相同。

在运行测试之前，我们必须声明 `responseParser` 方法，否则代码将无法编译：

```go
func (p *CurrentWeatherData) responseParser(body io.Reader) (*Weather, error) { 
  return nil, fmt.Errorf("Not implemented yet") 
} 

```

在所有上述内容的基础上，我们可以运行这个测试：

```go
go test -v -run=responseParser .
=== RUN   TestOpenWeatherMap_responseParser
--- FAIL: TestOpenWeatherMap_responseParser (0.00s)
 facade_test.go:72: Not implemented yet
FAIL
exit status 1
FAIL

```

好的。我们不会编写更多的测试，因为剩下的将仅仅是集成测试，这超出了结构模式解释的范围，并且将迫使我们拥有一个 API 密钥以及互联网连接。如果你想看到这个示例的集成测试是什么样的，请参考书中附带的相关代码。

## 实现

首先，我们将实现我们的方法将使用的解析器，以解析来自 `OpenWeatherMap` REST API 的 JSON 响应：

```go
func (p *CurrentWeatherData) responseParser(body io.Reader) (*Weather, error) { 
  w := new(Weather) 
  err := json.NewDecoder(body).Decode(w) 
  if err != nil { 
    return nil, err 
  } 

  return w, nil 
} 

```

到现在为止，这应该足以通过测试：

```go
go test -v -run=responseParser . 
=== RUN   TestOpenWeatherMap_responseParser 
--- PASS: TestOpenWeatherMap_responseParser (0.00s) 
PASS 
ok

```

至少我们的解析器已经得到了很好的测试。让我们将代码结构化，使其看起来像是一个库。首先，我们将创建通过城市名称和国家代码获取城市天气的方法，以及使用其纬度和经度的方法：

```go
func (c *CurrentWeatherData) GetByGeoCoordinates(lat, lon float32) (weather *Weather, err error) { 
  return c.doRequest( 
  fmt.Sprintf("http://api.openweathermap.org/data/2.5/weather q=%s,%s&APPID=%s", lat, lon, c.APIkey)) 
} 

func (c *CurrentWeatherData) GetByCityAndCountryCode(city, countryCode string) (weather *Weather, err error) { 
  return c.doRequest(   
  fmt.Sprintf("http://api.openweathermap.org/data/2.5/weather?lat=%f&lon=%f&APPID=%s", city, countryCode, c.APIkey) ) 
} 

```

一件小菜一碟？当然！一切必须尽可能简单，这是好工作的标志。在这个外观背后的复杂性是创建到 `OpenWeatherMap` API 的连接和控制可能的错误。这个问题在我们示例的所有 Facade 方法之间是共享的，所以我们现在不需要编写超过一个 API 调用。

我们所做的是传递 REST API 需要的 URL 以返回我们所需的信息。这是通过 `fmt.Sprintf` 函数实现的，它为每种情况格式化字符串。例如，要使用城市名称和国家代码收集数据，我们使用以下字符串：

```go
fmt.Sprintf("http://api.openweathermap.org/data/2.5/weather?lat=%f&lon=%f&APPID=%s", city, countryCode, c.APIkey) 

```

这将使用预先格式化的字符串 [`openweathermap.org/api`](https://openweathermap.org/api) 并通过将每个 `%s` 指示符替换为城市、我们在参数中引入的 `countryCode` 以及 `CurrentWeatherData` 类型的 API key 成员来格式化它。

但是，我们还没有设置任何 API 密钥！是的，因为这是一个库，库的用户将必须使用他们自己的 API 密钥。我们正在隐藏创建 URI 和处理错误的复杂性。

最后，`doRequest` 函数是一个大问题，所以我们将逐步详细地查看它：

```go
func (o *CurrentWeatherData) doRequest(uri string) (weather *Weather, err error) { 
  client := &http.Client{} 
  req, err := http.NewRequest("GET", uri, nil) 
  if err != nil { 
    return 
  } 
  req.Header.Set("Content-Type", "application/json") 

```

首先，签名告诉我们`doRequest`方法接受一个 URI 字符串，并返回指向`Weather`变量的指针和一个错误。我们首先创建一个`http.Client`类，它将发起请求。然后，我们创建一个请求对象，它将使用`GET`方法，正如`OpenWeatherMap`网页上所描述的，以及我们传递的 URI。如果我们使用不同的方法，或者使用多个方法，它们必须通过签名中的参数来实现。然而，我们将只使用`GET`方法，所以我们可以在那里硬编码它。

然后，我们检查请求对象是否已成功创建，并设置一个表示内容类型为 JSON 的头部：

```go
resp, err := client.Do(req) 
if err != nil { 
  return 
} 

if resp.StatusCode != 200 { 
  byt, errMsg := ioutil.ReadAll(resp.Body) 
  if errMsg == nil { 
    errMsg = fmt.Errorf("%s", string(byt)) 
  } 
  err = fmt.Errorf("Status code was %d, aborting. Error message was:\n%s\n",resp.StatusCode, errMsg) 

  return 
} 

```

然后，我们发起请求，并检查是否有错误。因为我们已经为返回类型命名了，如果发生任何错误，我们只需返回函数，Go 将返回变量`err`和变量`weather`在那一刻的状态。

我们检查响应的状态码，因为我们只接受 200 作为良好的响应。如果返回的不是 200，我们将创建一个包含正文内容和返回状态码的错误消息：

```go
  weather, err = o.responseParser(resp.Body) 
  resp.Body.Close() 

  return 
} 

```

最后，如果一切顺利，我们使用我们之前编写的`responseParser`函数来解析 Body 的内容，它是一个`io.Reader`接口。你可能想知道为什么我们不是从`response parser`方法中控制`err`。这很有趣，因为我们实际上是在控制它。`responseParser`和`doRequest`有相同的返回签名。两者都返回一个`Weather`指针和一个错误（如果有），所以我们可以直接返回任何结果。

## 使用外观模式创建的库

我们使用外观模式为`OpenWeatherMap` API 创建了一个库的第一个里程碑。我们已经将访问`OpenWeatherMap` REST API 的复杂性隐藏在`doRequest`和`responseParser`函数中，我们的库用户有一个易于使用的语法来查询 API。例如，要检索西班牙马德里的天气，用户只需在开始时输入参数和一个 API 密钥：

```go
  weatherMap := CurrentWeatherData{*apiKey} 

  weather, err := weatherMap.GetByCityAndCountryCode("Madrid", "ES") 
  if err != nil { 
    t.Fatal(err) 
  } 

  fmt.Printf("Temperature in Madrid is %f celsius\n", weather.Main.Temp-273.15) 

```

编写此章节时马德里天气的控制台输出如下：

```go
$ Temperature in Madrid is 30.600006 celsius

```

一个典型的夏日！

# 享元设计模式

我们下一个模式是**享元**设计模式。它在计算机图形和视频游戏行业中非常常用，但在企业应用中并不那么常见。

## 描述

享元是一种模式，它允许在许多实例之间共享重型对象的某些类型的状态。想象一下，你必须创建和存储太多本质上相同的一些重型对象。你很快就会耗尽内存。这个问题可以通过享元模式以及工厂模式的额外帮助轻松解决。工厂通常负责封装对象创建，正如我们之前所看到的。

## 目标

多亏了享元模式，我们可以在单个公共对象中共享所有可能的对象状态，从而通过使用指向已创建对象的指针来最小化对象创建。

## 示例

为了举例说明，我们将模拟你在投注网页上找到的东西。想象一下欧洲锦标赛的决赛，整个大陆有成千上万的人观看。现在想象一下我们拥有一个投注网页，我们提供关于欧洲每个团队的历史信息。这是大量的信息，通常存储在一些分布式数据库中，每个团队都有关于他们的球员、比赛、锦标赛等等的数兆字节信息。

如果一百万用户访问关于一个团队的信息，并且为每个查询历史数据的用户创建一个新的信息实例，我们将在一瞬间耗尽内存。有了我们的代理解决方案，我们可以缓存最近的 *n* 次搜索以加快查询速度，但如果我们为每个团队返回一个克隆，我们仍然会内存不足（但得益于我们的缓存会更快）。有趣，对吧？

相反，我们将只存储每个团队的信息一次，并将向用户交付它们的引用。因此，如果我们面临一百万用户试图访问关于比赛的信息，实际上我们只有两个团队在内存中，并且有一百万个指向相同内存方向的指针。

## 接受标准

Flyweight 模式的接受标准必须始终减少使用的内存量，并且必须主要关注这个目标：

1.  我们将创建一个包含一些基本信息（如团队名称、球员、历史结果和描述他们盾牌的图像）的 `Team` 结构体。

1.  我们必须确保正确创建团队（注意这里的词 *creation*，是创建模式的候选词），并且没有重复。

1.  当两次创建相同的团队时，我们必须有两个指针指向相同的内存地址。

## 基本结构体和测试

我们的 `Team` 结构体将包含其他结构体，因此总共将创建四个结构体。`Team` 结构体具有以下签名：

```go
type Team struct { 
  ID             uint64 
  Name           string 
  Shield         []byte 
  Players        []Player 
  HistoricalData []HistoricalData 
} 

```

每个团队都有一个 ID、一个名称、一些表示团队盾牌的字节切片中的图像、一个玩家切片和一个历史数据切片。这样，我们将有两个团队的 ID：

```go
const ( 
  TEAM_A = iota 
  TEAM_B 
) 

```

我们通过使用 `const` 和 `iota` 关键字声明了两个常量。`const` 关键字简单地声明以下声明是常量。`iota` 是一个无类型的整数，它会在括号中的每个新常量之间自动增加其值。当我们声明 `TEAM_A` 时，`iota` 的值会重置为 0，因此 `TEAM_A` 等于 0。在 `TEAM_B` 变量上，`iota` 增加了一个，所以 `TEAM_B` 等于 1。`iota` 赋值是声明不需要特定值（如 `math` 包中的 *Pi* 常量）的常量值时的简洁方式。

我们的 `Player` 和 `HistoricalData` 如下：

```go
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

```

如您所见，我们还需要一个`Match`结构体，它存储在`HistoricalData`结构体中。在这个上下文中，`Match`结构体代表比赛的记录结果：

```go
type Match struct { 
  Date          time.Time 
  VisitorID     uint64 
  LocalID       uint64 
  LocalScore    byte 
  VisitorScore  byte 
  LocalShoots   uint16 
  VisitorShoots uint16 
} 

```

这足以表示一个团队，并满足*验收标准 1*。你可能已经猜到，每个团队都有很多信息，因为一些欧洲团队已经存在了 100 多年。

对于*验收标准 2*，单词*创建*应该给我们一些关于如何解决这个问题的一些线索。我们将构建一个工厂来创建和存储我们的团队。我们的工厂将包括一个年份的映射，其中包含指向`Teams`的指针作为值，以及一个`GetTeam`函数。如果我们事先知道他们的名字，使用映射将提高团队搜索的效率。我们还将提供一个方法来返回创建的对象数量，这个方法将被称为`GetNumberOfObjects`方法：

```go
type teamFlyweightFactory struct { 
  createdTeams map[string]*Team 
} 

func (t *teamFlyweightFactory) GetTeam(name string) *Team { 
  return nil 
} 

func (t *teamFlyweightFactory) GetNumberOfObjects() int { 
  return 0 
} 

```

这足以编写我们的第一个单元测试：

```go
func TestTeamFlyweightFactory_GetTeam(t *testing.T) { 
  factory := teamFlyweightFactory{} 

teamA1 := factory.GetTeam(TEAM_A) 
  if teamA1 == nil { 
    t.Error("The pointer to the TEAM_A was nil") 
  } 

  teamA2 := factory.GetTeam(TEAM_A) 
  if teamA2 == nil { 
    t.Error("The pointer to the TEAM_A was nil") 
  } 

  if teamA1 != teamA2 { 
    t.Error("TEAM_A pointers weren't the same") 
  } 

  if factory.GetNumberOfObjects() != 1 { 
    t.Errorf("The number of objects created was not 1: %d\n", factory.GetNumberOfObjects()) 
  } 
} 

```

在我们的测试中，我们验证所有验收标准。首先我们创建一个工厂，然后请求`TEAM_A`的指针。这个指针不能为`nil`，否则测试将失败。

然后我们调用指向同一团队的第二个指针。这个指针也不能为空，它应该指向与上一个指针相同的内存地址，这样我们才知道它没有分配新的内存。

最后，我们应该检查创建的团队数量是否只有一个，因为我们请求了同一个团队两次。我们有两个指针，但只有一个团队的实例。让我们运行测试：

```go
$ go test -v -run=GetTeam .
=== RUN   TestTeamFlyweightFactory_GetTeam
--- FAIL: TestTeamFlyweightFactory_GetTeam (0.00s)
flyweight_test.go:11: The pointer to the TEAM_A was nil
flyweight_test.go:21: The pointer to the TEAM_A was nil
flyweight_test.go:31: The number of objects created was not 1: 0
FAIL
exit status 1
FAIL

```

嗯，它失败了。两个指针都是`nil`，它没有创建任何对象。有趣的是，比较两个指针的函数并没有失败；总的来说，`nil`等于`nil`。

## 实现方式

我们的`GetTeam`方法需要扫描名为`createdTeams`的`map`字段，以确保查询的团队已经创建，如果已经创建，则返回它。如果团队尚未创建，它必须在返回之前创建它并将其存储在映射中：

```go
func (t *teamFlyweightFactory) GetTeam(teamID int) *Team { 
  if t.createdTeams[teamID] != nil { 
    return t.createdTeams[teamID] 
  } 

  team := getTeamFactory(teamID) 
  t.createdTeams[teamID] = &team 

  return t.createdTeams[teamID] 
} 

```

上述代码非常简单。如果参数名称存在于`createdTeams`映射中，则返回指针。否则，调用工厂进行团队创建。这足以让我们停下来分析一下。当你使用 Flyweight 模式时，非常常见的是有一个 Flyweight 工厂，它使用其他类型的创建模式来检索它需要的对象。

因此，`getTeamFactory`方法将给我们我们正在寻找的团队，我们将它在映射中存储，并返回它。团队工厂将能够创建两个团队：`TEAM_A`和`TEAM_B`：

```go
func getTeamFactory(team int) Team { 
  switch team { 
    case TEAM_B: 
    return Team{ 
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

```

我们简化了对象的内容，以便我们可以专注于 Flyweight 模式的实现。好吧，所以我们只需要定义一个函数来检索创建的对象数量，如下所示：

```go
func (t *teamFlyweightFactory) GetNumberOfObjects() int { 
  return len(t.createdTeams) 
} 

```

这相当简单。`len`函数返回数组或切片中的元素数量，字符串中的字符数量等等。看起来一切都已经完成，我们可以再次启动测试：

```go
$ go test -v -run=GetTeam . 
=== RUN   TestTeamFlyweightFactory_GetTeam 
--- FAIL: TestTeamFlyweightFactory_GetTeam (0.00s) 
panic: assignment to entry in nil map [recovered] 
        panic: assignment to entry in nil map 

goroutine 5 [running]: 
panic(0x530900, 0xc0820025c0) 
        /home/mcastro/Go/src/runtime/panic.go:481 +0x3f4 
testing.tRunner.func1(0xc082068120) 
        /home/mcastro/Go/src/testing/testing.go:467 +0x199 
panic(0x530900, 0xc0820025c0) 
        /home/mcastro/Go/src/runtime/panic.go:443 +0x4f7 
/home/mcastro/go-design-patterns/structural/flyweight.(*teamFlyweightFactory).GetTeam(0xc08202fec0, 0x0, 0x0) 
        /home/mcastro/Desktop/go-design-patterns/structural/flyweight/flyweight.go:71 +0x159 
/home/mcastro/go-design-patterns/structural/flyweight.TestTeamFlyweightFactory_GetTeam(0xc082068120) 
        /home/mcastro/Desktop/go-design-patterns/structural/flyweight/flyweight_test.go:9 +0x61 
testing.tRunner(0xc082068120, 0x666580) 
        /home/mcastro/Go/src/testing/testing.go:473 +0x9f 
created by testing.RunTests 
        /home/mcastro/Go/src/testing/testing.go:582 +0x899 
exit status 2 
FAIL

```

惊慌！我们忘记什么了吗？通过阅读恐慌信息的堆栈跟踪，我们可以看到一些地址、一些文件，并且看起来`GetTeam`方法正在尝试将一个条目分配给`flyweight.go`文件的第*71*行上的 nil 映射。让我们仔细看看*第 71*行（记住，如果你在遵循本教程编写代码时遇到错误，错误可能出现在不同的行上，所以请仔细查看你的堆栈跟踪）：

```go
t.createdTeams[teamName] = &team 

```

好吧，这一行是在`GetTeam`方法中，当方法通过这里时，这意味着它没有在映射中找到团队（它已经创建了变量 team），并试图将其分配给映射。但是映射是 nil，因为我们没有在创建工厂时初始化它。这有一个快速的解决方案。在我们的测试中，在创建工厂的地方初始化映射：

```go
factory := teamFlyweightFactory{ 
  createdTeams: make(map[int]*Team,0), 
} 

```

我相信你已经在这里看到了问题。如果我们无法访问包，我们可以初始化变量。好吧，我们可以使变量公开，这就足够了。但这将要求每个实现者都知道他们必须初始化映射，并且它的签名既不方便也不优雅。相反，我们将创建一个简单的工厂构建器来为我们完成这项工作。这是 Go 中一个非常常见的方法：

```go
func NewTeamFactory() teamFlyweightFactory { 
  return teamFlyweightFactory{ 
    createdTeams: make(map[int]*Team), 
  } 
} 

```

现在，在测试中，我们用对这个函数的调用替换工厂创建：

```go
func TestTeamFlyweightFactory_GetTeam(t *testing.T) { 
  factory := NewTeamFactory() 
  ... 
} 

```

再次运行测试：

```go
$ go test -v -run=GetTeam .
=== RUN   TestTeamFlyweightFactory_GetTeam
--- PASS: TestTeamFlyweightFactory_GetTeam (0.00s)
PASS
ok 

```

完美！让我们通过添加第二个测试来改进测试，以确保在更大规模的情况下一切都会按预期运行。我们将创建一百万个团队创建的调用，代表一百万个用户的调用。然后，我们将简单地检查创建的团队数量仅为两个：

```go
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

```

在这个测试中，我们分别检索`TEAM_A`和`TEAM_B` 500,000 次，以达到一百万用户。然后，我们确保只创建了两个对象：

```go
$ go test -v -run=Volume . 
=== RUN   Test_HighVolume 
--- PASS: Test_HighVolume (0.04s) 
PASS 
ok

```

完美！我们甚至可以检查指针指向的位置以及它们所在的位置。我们将以前三个为例进行检查。将这些行添加到上次测试的末尾，然后再次运行：

```go
for i:=0; i<3; i++ { 
  fmt.Printf("Pointer %d points to %p and is located in %p\n", i, teams[i], &teams[i]) 
} 

```

在前面的测试中，我们使用`Printf`方法打印有关指针的信息。`%p`标志给出了指针指向的对象的内存位置。如果你通过传递`&`符号引用指针，它将给出指针本身的指向。

再次使用相同的命令运行测试；你将在输出中看到三行新信息，类似于以下内容：

```go
Pointer 0 points to 0xc082846000 and is located in 0xc082076000
Pointer 1 points to 0xc082846000 and is located in 0xc082076008
Pointer 2 points to 0xc082846000 and is located in 0xc082076010

```

它告诉我们的是，映射中的前三个位置指向相同的位置，但实际上我们有三个不同的指针，它们实际上比我们的团队对象轻得多。

## 那么，单例和享元之间的区别是什么？

好吧，区别是微妙的，但确实存在。使用单例模式，我们确保同一类型只创建一次。此外，单例模式是一种创建模式。使用享元模式，它是一种结构模式，我们并不担心对象是如何创建的，而是关注如何以轻量级的方式对类型进行结构化以包含大量信息。我们谈论的结构是我们例子中的`map[int]*Team`结构。在这里，我们真的不关心对象是如何创建的；我们只是简单地为它编写了一个简单的`getTeamFactory`方法。我们非常重视拥有一个轻量级结构来持有可共享的对象（或对象），在这种情况下，是映射。

# 摘要

我们已经看到了几种组织代码结构的模式。结构模式关注的是如何创建对象，或者它们是如何进行业务处理的（我们将在行为模式中看到这一点）。

不要因为混合多种模式而感到困惑。如果你严格遵循每个模式的目标，你可能会轻松地混合六种或七种。只需记住，过度设计和不设计一样糟糕。我记得有一天晚上我在原型设计一个负载均衡器，经过两个小时疯狂的超前设计代码后，我头脑中一团糟，宁愿从头开始。

在下一章，我们将看到行为模式。它们稍微复杂一些，并且它们通常使用结构和创建模式来实现目标，但我相信读者会发现它们相当具有挑战性和趣味性。
