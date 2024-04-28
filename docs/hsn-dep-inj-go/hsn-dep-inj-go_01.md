# 永远不要停止追求更好

你想要更容易维护的代码吗？更容易测试吗？更容易扩展吗？**依赖注入**（**DI**）可能正是你需要的工具。

在本章中，我们将以一种有点非典型的方式定义 DI，并探讨可能表明你需要 DI 的代码异味。我们还将简要讨论 Go 以及我希望你如何对待本书中提出的想法。

你准备好和我一起踏上更好的 Go 代码之旅了吗？

我们将涵盖以下主题：

+   DI 为什么重要？

+   什么是 DI？

+   何时应用 DI？

+   我如何作为 Go 程序员改进？

# 技术要求

希望你已经安装了 Go。它可以从[`golang.org/`](https://golang.org/) 或你喜欢的软件包管理器下载。

本章中的所有代码都可以在[`github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch01`](https://github.com/PacktPublishing/Hands-On-Dependency-Injection-in-Go/tree/master/ch01)找到。

# DI 为什么重要？

作为专业人士，我们永远不应该停止学习。学习是确保我们保持需求并继续为客户提供价值的唯一真正途径。医生、律师和科学家都是备受尊敬的专业人士，他们都专注于不断学习。为什么程序员应该有所不同呢？

在本书中，我们将开始一段旅程，从一些*完成工作*的代码开始，然后通过有选择地应用 Go 中可用的各种 DI 方法，我们将把它转变成更容易维护、测试和扩展的东西。

本书中并非所有内容都是*传统*的，甚至可能不是*惯用*的，但我希望你在否定之前*尝试一下*。如果你喜欢，太棒了。如果不喜欢，至少你学到了你不想做什么。

# 那么，我如何定义 DI？

DI 是*以这样的方式编码，使得我们依赖的资源（即函数或结构）是抽象的*。因为这些依赖是抽象的，对它们的更改不需要更改我们的代码。这个花哨的词是**解耦**。

这里使用的抽象一词可能有点误导。我不是指像 Java 中那样的抽象类；Go 没有那个。不过，Go 确实有接口和函数文字（也称为**闭包**）。

考虑以下接口的例子和使用它的`SavePerson()`函数：

```go
// Saver persists the supplied bytes
type Saver interface {
  Save(data []byte) error
}

// SavePerson will validate and persist the supplied person
func SavePerson(person *Person, saver Saver) error {
  // validate the inputs
  err := person.validate()
  if err != nil {
    return err
  }

  // encode person to bytes
  bytes, err := person.encode()
  if err != nil {
    return err
  }

  // save the person and return the result
  return saver.Save(bytes)
}

// Person data object
type Person struct {
   Name  string
   Phone string
}

// validate the person object
func (p *Person) validate() error {
   if p.Name == "" {
      return errors.New("name missing")
   }

   if p.Phone == "" {
      return errors.New("phone missing")
   }

   return nil
}

// convert the person into bytes
func (p *Person) encode() ([]byte, error) {
   return json.Marshal(p)
}
```

在前面的例子中，`Saver`是做什么的？它在某个地方保存一些`bytes`。它是如何做到的？我们不知道，在编写`SavePerson`函数时，我们也不关心。

让我们看另一个使用函数文字的例子**：**

```go
// LoadPerson will load the requested person by ID.
// Errors include: invalid ID, missing person and failure to load 
// or decode.
func LoadPerson(ID int, decodePerson func(data []byte) *Person) (*Person, error) {
  // validate the input
  if ID <= 0 {
    return nil, fmt.Errorf("invalid ID '%d' supplied", ID)
  }

  // load from storage
  bytes, err := loadPerson(ID)
  if err != nil {
    return nil, err
  }

  // decode bytes and return
  return decodePerson(bytes), nil
}
```

`decodePerson`是做什么的？它将`bytes`转换为一个人。怎么做？我们现在不需要知道。

这是我要向你强调的 DI 的第一个优点：

**DI 通过以抽象或通用的方式表达依赖关系，减少了在处理一段代码时所需的知识**

现在，假设前面的代码来自一个将数据存储在**网络文件共享**（**NFS**）中的系统。我们如何为此编写单元测试？始终访问 NFS 将是一种痛苦。由于完全不相关的问题，例如网络连接问题，任何此类测试也会比应该更频繁地失败。

另一方面，通过依赖于抽象，我们可以用虚假代码替换保存到 NFS 的代码。这样，我们只测试我们的代码与 NFS 隔离的情况，如下面的代码所示：

```go
func TestSavePerson_happyPath(t *testing.T) {
   // input
   in := &Person{
      Name:  "Sophia",
      Phone: "0123456789",
   }

   // mock the NFS
   mockNFS := &mockSaver{}
   mockNFS.On("Save", mock.Anything).Return(nil).Once()

   // Call Save
   resultErr := SavePerson(in, mockNFS)

   // validate result
   assert.NoError(t, resultErr)
   assert.True(t, mockNFS.AssertExpectations(t))
}
```

不要担心前面的代码看起来陌生；我们将在本书的后面深入研究所有部分。

这带我们来到 DI 的第二个优点：

**DI 使我们能够在不依赖于我们的依赖关系的情况下测试我们的代码**

考虑前面的例子，我们如何测试我们的错误处理代码？我们可以通过一些外部脚本关闭 NFS，每次运行测试时，但这可能会很慢，肯定会惹恼依赖它的其他人。

另一方面，我们可以快速制作一个总是失败的假`Saver`，如下所示：

```go
func TestSavePerson_nfsAlwaysFails(t *testing.T) {
   // input
   in := &Person{
      Name:  "Sophia",
      Phone: "0123456789",
   }

   // mock the NFS
   mockNFS := &mockSaver{}
   mockNFS.On("Save", mock.Anything).Return(errors.New("save failed")).Once()

   // Call Save
   resultErr := SavePerson(in, mockNFS)

   // validate result
   assert.Error(t, resultErr)
   assert.True(t, mockNFS.AssertExpectations(t))
}
```

上面的测试快速、可预测、可靠。这是我们测试中想要的一切！

这给了我们 DI 的第三个优势：

**DI 使我们能够快速、可靠地测试其他情况**

不要忘记 DI 的传统销售点。如果明天我们决定将保存到 NoSQL 数据库而不是我们的 NFS，我们的`SavePerson`代码将如何改变？一点也不。我们只需要编写一个新的`Saver`实现，这给了我们 DI 的第四个优势：

**DI 减少了扩展或更改的影响**

归根结底，DI 是一个工具——一个方便的工具，但不是魔法子弹。它是一个可以使代码更容易理解、测试、扩展和重用的工具，也可以帮助减少常常困扰新 Go 开发人员的循环依赖问题。

# 表明您可能需要 DI 的代码气味

俗话说“对于只有一把锤子的人来说，每个问题都像一颗钉子”，这句话虽然古老，但在编程中却从未比现在更真实。作为专业人士，我们应该不断努力获取更多的工具，以便更好地应对工作中遇到的任何问题。DI 虽然是一个非常有用的工具，但只对特定的问题有效。在我们的情况下，这些问题是**代码气味**。代码气味是代码中潜在更深层问题的指示。

有许多不同类型的代码气味；在本节中，我们将仅讨论那些可以通过 DI 缓解的气味。在后面的章节中，我们将在试图从我们的代码中消除它们时引用这些气味。

代码气味通常可以分为四个不同的类别：

+   代码膨胀

+   对变化的抵抗

+   徒劳的努力

+   紧耦合

# 代码膨胀

代码膨胀的气味是指已经添加到结构体或函数中的笨重代码块，使得它们变得难以理解、维护和测试。在旧代码中经常发现，它们往往是逐渐恶化和缺乏维护的结果，而不是有意的选择。

它们可以通过对源代码进行视觉扫描或使用循环复杂度检查器（指示代码复杂性的软件度量标准）来发现，例如 gocyclo（[`github.com/fzipp/gocyclo`](https://github.com/fzipp/gocyclo)）。

这些气味包括以下内容：

+   **长方法**：虽然代码是在计算机上运行的，但是它是为人类编写的。任何超过 30 行的方法都应该分成更小的块。虽然对计算机没有影响，但对我们人类来说更容易理解。

+   **长结构体**：与长方法类似，结构体越长，就越难理解，因此也更难维护。长结构体通常也表明结构体做得太多。将一个结构体分成几个较小的结构体也是增加代码可重用性潜力的好方法。

+   **长参数列表**：长参数列表也表明该方法可能做了太多的事情。在添加新功能时，很容易向现有函数添加新参数，以适应新的用例。这是一个很危险的斜坡。这个新参数要么对现有用例是可选的/不必要的，要么表明方法的复杂性显著增加。

+   **长条件块**：Switch 语句很棒。问题在于它们很容易被滥用，而且往往像谚语中的兔子一样繁殖。然而，最重要的问题可能是它们对代码的可读性的影响。长条件块占用大量空间，打断了函数的可读性。考虑以下代码：

```go
func AppendValue(buffer []byte, in interface{}) []byte{
   var value []byte

   // convert input to []byte
   switch concrete := in.(type) {
   case []byte:
      value = concrete

   case string:
      value = []byte(concrete)

   case int64:
      value = []byte(strconv.FormatInt(concrete, 10))

   case bool:
      value = []byte(strconv.FormatBool(concrete))

   case float64:
      value = []byte(strconv.FormatFloat(concrete, 'e', 3, 64))
   }

   buffer = append(buffer, value...)
   return buffer
}
```

通过将`interface{}`作为输入，我们几乎被迫使用类似这样的开关。我们最好改为从`interface{}`改为接口，然后向接口添加必要的操作。这种方法在标准库中的`json.Marshaller`和`driver.Valuer`接口中得到了很好的说明。

将 DI 应用于这些问题通常会通过将其分解为更小的、独立的部分来减少代码的复杂性，从而使其更易于理解、维护和测试。

# 对变化的抵抗

这些情况下很难和/或缓慢地添加新功能。同样，测试通常更难编写，特别是对于失败条件的测试。与代码膨胀类似，这些问题可能是逐渐恶化和缺乏维护的结果，但也可能是由于缺乏前期规划或糟糕的 API 设计引起的。

它们可以通过检查拉取请求日志或提交历史来找到，特别是确定新功能是否需要在代码的不同部分进行许多小的更改。

如果您的团队跟踪功能速度，并且您注意到它在下降，这也可能是一个原因。

这些问题包括以下内容：

+   **散弹手术**：这是指对一个结构体进行的小改动需要改变其他结构体。这些变化意味着使用的组织或抽象是不正确的。通常，所有这些更改应该在一个类中。

在下面的例子中，您可以看到向人员数据添加电子邮件字段将导致更改所有三个结构体（`Presenter`、`Validator`和`Saver`）：

```go
// Renderer will render a person to the supplied writer
type Renderer struct{}

func (r Renderer) render(name, phone string, output io.Writer) {
  // output the person
}

// Validator will validate the supplied person has all the 
// required fields
type Validator struct{}

func (v Validator) validate(name, phone string) error {
  // validate the person
  return nil
}

// Saver will save the supplied person to the DB
type Saver struct{}

func (s *Saver) Save(db *sql.DB, name, phone string) {
  // save the person to db
}
```

+   **泄漏实现细节**：Go 社区中更受欢迎的习语之一是*接受接口，返回结构体*。这是一个引人注目的短语，但它的简单性掩盖了它的巧妙之处。当一个函数接受一个结构体时，它将用户与特定的实现联系在一起，这种严格的关系使得未来的更改或附加使用变得困难。此外，如果实现细节发生变化，API 也会发生变化，并迫使用户进行更改。

将 DI 应用于这些问题通常是对未来的良好投资。虽然不修复它们不会致命，但代码将逐渐恶化，直到你处理谚语中的*大泥球*。你知道这种类型——一个没有人理解、没有人信任的包，只有勇敢或愚蠢的人愿意进行更改。DI 使您能够脱离实现选择，从而更容易地重构、测试和维护代码的小块。

# 浪费的努力

这些问题是代码维护成本高于必要成本的情况。它们通常是由懒惰或缺乏经验引起的。复制/粘贴代码总是比仔细重构代码更容易。问题是，像这样编码就像吃不健康的零食。在当时感觉很棒，但长期后果很糟糕。

它们可以通过对源代码进行批判性审视并问自己*我真的需要这段代码吗？*或者*我能让这更容易理解吗？*来找到。

使用诸如 dupl ([`github.com/mibk/dupl`](https://github.com/mibk/dupl))或 PMD ([`pmd.github.io/`](https://pmd.github.io/))之类的工具也将帮助您识别需要调查的代码区域。

这些问题包括以下内容：

+   **过多的重复代码**：首先，请不要对此变得过分狂热。虽然在大多数情况下，重复的代码是一件坏事，但有时复制代码可以导致一个更容易维护和发展的系统。我们将在第八章中处理这种问题的常见来源，*通过配置进行依赖注入*。

+   **过多的注释**：为后来的人留下一条便签，即使只有 6 个月后的自己，也是一件友好和专业的事情。但当这个注释变成一篇文章时，就是重构的时候了。

```go
// Excessive comments
func outputOrderedPeopleA(in []*Person) {
  // This code orders people by name.
  // In cases where the name is the same, it will order by 
  // phone number.
  // The sort algorithm used is a bubble sort
  // WARNING: this sort will change the items of the input array
  for _, p := range in {
    // ... sort code removed ...
  }

  outputPeople(in)
}

// Comments replaced with descriptive names
func outputOrderedPeopleB(in []*Person) {
  sortPeople(in)
  outputPeople(in)
}
```

+   **过于复杂的代码**：代码越难让其他人理解，它就越糟糕。通常，这是某人试图过于花哨或者没有花足够的精力在结构或命名上的结果。从更自私的角度来看，如果只有你一个人能理解一段代码，那么只有你能够处理它。也就是说，你注定要永远维护它。以下代码是做什么的：

```go
for a := float64(0); a < 360; a++ {
   ra := math.Pi * 2 * a / 360
   x := r*math.Sin(ra) + v
   y := r*math.Cos(ra) + v
   i.Set(int(x), int(y), c)
}
```

+   **DRY/WET 代码**：**不要重复自己**（DRY）原则旨在通过将责任分组并提供清晰的抽象来减少重复的工作。相比之下，在 WET 代码中，有时也被称为**浪费每个人的时间**代码，你会发现同样的责任出现在许多地方。这种气味通常出现在格式化或转换代码中。这种代码应该存在于系统边界，也就是说，转换用户输入或格式化输出。

虽然许多这些气味可以在没有依赖注入的情况下修复，但依赖注入提供了一种更容易的方式来将重复的工作转移到一个抽象中，然后可以用来减少重复和提高代码的可读性和可维护性。

# 紧耦合

对于人来说，紧耦合可能是一件好事。但对于 Go 代码来说，真的不是。耦合是衡量对象之间关系或依赖程度的指标。当存在紧耦合时，这种相互依赖会迫使对象或包一起发展，增加了复杂性和维护成本。

耦合相关的气味可能是最隐匿和顽固的，但处理起来也是最有回报的。它们通常是由于缺乏面向对象设计或接口使用不足造成的。

遗憾的是，我没有一个方便的工具来帮助你找到这些气味，但我相信，在本书结束时，你将毫无困难地发现并处理它们。

经常情况下，我发现先以紧密耦合的形式实现一个功能，然后逐步解耦并彻底单元测试我的代码，然后再提交，这对我来说是特别有帮助的，尤其是在正确的抽象不明显的情况下。

这些气味包括以下内容：

+   **依赖于上帝对象**：这些是*知道太多*或*做太多*的大对象。虽然这是一种普遍的代码气味，应该像瘟疫一样避免，但从依赖注入的角度来看，问题在于太多的代码依赖于这个对象。当它们存在并且我们不小心时，很快 Go 就会因为循环依赖而拒绝编译。有趣的是，Go 认为依赖和导入不是在对象级别，而是在包级别。因此，我们也必须避免上帝包。我们将在第八章中解决一个非常常见的上帝对象问题，*通过配置进行依赖注入*。

+   **循环依赖**：这是指包 A 依赖于包 B，包 B 又依赖于包 A。这是一个容易犯的错误，有时很难摆脱。

在下面的例子中，虽然配置可以说是一个`上帝`对象，因此是一种代码气味，但我很难找到更好的方法来从一个单独的 JSON 文件中导入配置。相反，我会认为需要解决的问题是`orders`包对`config`包的使用。一个典型的上帝配置对象如下：

```go
package config

import ...

// Config defines the JSON format of the config file
type Config struct {
   // Address is the host and port to bind to.  
   // Default 0.0.0.0:8080
   Address string

   // DefaultCurrency is the default currency of the system
   DefaultCurrency payment.Currency
}

// Load will load the JSON config from the file supplied
func Load(filename string) (*Config, error) {
   // TODO: load currency from file
   return nil, errors.New("not implemented yet")
}
```

在对`config`包的尝试使用中，你可以看到`Currency`类型属于`Package`包，因此在`config`中包含它，如前面的例子所示，会导致循环依赖：

```go
package payment

import ...

// Currency is custom type for currency
type Currency string

// Processor processes payments
type Processor struct {
   Config *config.Config
}

// Pay makes a payment in the default currency
func (p *Processor) Pay(amount float64) error {
   // TODO: implement me
   return errors.New("not implemented yet")
}
```

+   **对象混乱**：当一个对象对另一个对象的内部知识和/或访问过多时，或者换句话说，*对象之间的封装不足*。因为这些对象*紧密耦合*，它们经常需要一起发展，增加了理解代码和维护代码的成本。考虑以下代码：

```go
type PageLoader struct {
}

func (o *PageLoader) LoadPage(url string) ([]byte, error) {
   b := newFetcher()

   // check cache
   payload, err := b.cache.Get(url)
   if err == nil {
      // found in cache
      return payload, nil
   }

   // call upstream
   resp, err := b.httpClient.Get(url)
   if err != nil {
      return nil, err
   }
   defer resp.Body.Close()

   // extract data from HTTP response
   payload, err = ioutil.ReadAll(resp.Body)
   if err != nil {
      return nil, err
   }

   // save to cache asynchronously
   go func(key string, value []byte) {
      b.cache.Set(key, value)
   }(url, payload)

   // return
   return payload, nil
}

type Fetcher struct {
   httpClient http.Client
   cache      *Cache
}

```

在这个例子中，`PageLoader`重复调用`Fetcher`的成员变量。以至于，如果`Fetcher`的实现发生了变化，`PageLoader`很可能会受到影响。在这种情况下，这两个对象应该合并在一起，因为`PageLoader`没有额外的功能。

+   **Yo-yo problem**：这种情况的标准定义是*当继承图如此漫长和复杂以至于程序员不得不不断地翻阅代码才能理解它*。鉴于 Go 没有继承，你可能会认为我们不会遇到这个问题。然而，如果你努力尝试，通过过度的组合是可能的。为了解决这个问题，最好保持关系尽可能浅和抽象。这样，我们在进行更改时可以集中在一个更小的范围内，并将许多小对象组合成一个更大的系统。

+   **Feature envy**：当一个函数广泛使用另一个对象时，它就是嫉妒它。通常，这表明该函数应该从它所嫉妒的对象中移开。DI 可能不是解决这个问题的方法，但这种情况表明高耦合，因此是考虑应用 DI 技术的指标：

```go
func doSearchWithEnvy(request searchRequest) ([]searchResults, error) {
   // validate request
   if request.query == "" {
      return nil, errors.New("search term is missing")
   }
   if request.start.IsZero() || request.start.After(time.Now()) {
      return nil, errors.New("start time is missing or invalid")
   }
   if request.end.IsZero() || request.end.Before(request.start) {
      return nil, errors.New("end time is missing or invalid")
   }

   return performSearch(request)
}

func doSearchWithoutEnvy(request searchRequest) ([]searchResults, error) {
   err := request.validate()
   if err != nil {
      return nil, err
   }

   return performSearch(request)
}
```

当你的代码变得不那么耦合时，你会发现各个部分（包、接口和结构）会变得更加专注。这被称为**高内聚**。低耦合和高内聚都是可取的，因为它们使代码更容易理解和处理。

# 健康的怀疑。

当我们阅读本书时，你将看到一些很棒的编码技巧，也会看到一些不太好的。我希望你花一些时间思考哪些是好的，哪些是不好的。持续学习应该与健康的怀疑相结合。对于每种技术，我会列出其利弊，但我希望你能深入思考。问问自己以下问题：

+   这种技术试图实现什么？

+   我应用这种技术后，我的代码会是什么样子？

+   我真的需要它吗？

+   使用这种方法有什么不利之处吗？

即使你内心的怀疑者否定了这种技术，你至少学会了识别自己不喜欢并且不想使用的东西，而学习总是一种胜利。

# 关于符合 Go 的惯例的简短说明

我个人尽量避免使用术语**符合 Go 的惯例**，但是一本 Go 书在某种程度上没有涉及它是不完整的。我避免使用它，因为我经常看到它被用来打击人。基本上，*这不是符合惯例的，因此是错误的*，并且由此推论，*我是符合惯例的，因此比你更好*。我相信编程是一门手艺，虽然手艺在应用中应该有一定的一致性，但是，就像所有手艺一样，它应该是灵活的。毕竟，创新通常是通过弯曲或打破规则来实现的。

那么对我来说，符合 Go 的惯例意味着什么？

我会尽量宽泛地定义它：

+   **使用`gofmt`格式化你的代码**：对我们程序员来说，真的少了一件要争论的事情。这是官方的风格，由官方工具支持。让我们找一些更实质性的事情来争论。

+   阅读，应用，并定期回顾《Effective Go》（[`golang.org/doc/effective_go.html`](https://golang.org/doc/effective_go.html)）和《Code Review Comments》（[`github.com/golang/go/wiki/CodeReviewComments`](https://github.com/golang/go/wiki/CodeReviewComments)）中的想法：这些页面中包含了大量的智慧，以至于可能不可能仅通过一次阅读就能全部领会。

+   **积极应用*Unix 哲学***：它规定我们应该*设计代码只做一件事，但要做得很好，并且与其他代码很好地协同工作**。*

虽然对我来说，这三件事是最低限度的，但还有一些其他的想法也很有共鸣：

+   **接受接口并返回结构体**：虽然接受接口会导致代码解耦，但返回结构体可能会让你感到矛盾。我知道一开始我也是这样认为的。虽然输出接口可能会让你感觉它更松散耦合，但实际上并不是。输出只能是一种东西——无论你编码成什么样。如果需要，返回接口是可以的，但强迫自己这样做最终只会让你写更多的代码。

+   **合理的默认值**：自从转向 Go 以来，我发现许多情况下我想要为用户提供配置模块的能力，但这样的配置通常不被使用。在其他语言中，这可能会导致多个构造函数或很少使用的参数，但通过应用这种模式，我们最终得到了一个更清晰的 API 和更少的代码来维护。

# 把你的包袱留在门口

如果你问我*新手 Go 程序员最常犯的错误是什么*？我会毫不犹豫地告诉你，那就是将其他语言的模式带入 Go 中。我知道这是我最初的最大错误。我的第一个 Go 服务看起来像是用 Go 编写的 Java 应用程序。结果不仅是次等的，而且相当痛苦，特别是当我试图实现诸如继承之类的东西时。我在使用`Node.js`中以函数式风格编程 Go 时也有类似的经历。

简而言之，请不要这样做。重新阅读*Effective Go*和 Go 博客，直到您发现自己使用小接口、毫不犹豫地启动 Go 例程、喜欢通道，并想知道为什么您需要的不仅仅是组合来实现良好的多态性。

# 总结

在本章中，我们开始了一段旅程——这段旅程将导致更容易维护、扩展和测试的代码。

我们首先定义了 DI，并检查了它可以给我们带来的一些好处。通过一些例子的帮助，我们看到了这在 Go 中可能是什么样子。

之后，我们开始识别需要注意的代码异味，并通过应用 DI 来解决或减轻这些问题。

最后，我们研究了我认为 Go 代码是什么样子的，并向您提出质疑，对本书中提出的技术持怀疑态度。

# 问题

1.  什么是 DI？

1.  DI 的四个突出优势是什么？

1.  它解决了哪些问题？

1.  为什么持怀疑态度很重要？

1.  对你来说，惯用的 Go 是什么意思？

# 进一步阅读

Packt 还有许多其他关于 DI 和 Go 的学习资源。

+   [`www.packtpub.com/application-development/java-9-dependency-injection`](https://www.packtpub.com/application-development/java-9-dependency-injection)

+   [`www.packtpub.com/application-development/dependency-injection-net-core-20`](https://www.packtpub.com/application-development/dependency-injection-net-core-20)

+   [`www.packtpub.com/networking-and-servers/mastering-go`](https://www.packtpub.com/networking-and-servers/mastering-go)
