# Go 包、算法和数据结构

本章的主要主题将是 Go 包、算法和数据结构。如果您将所有这些结合起来，您将得到一个完整的程序，因为 Go 程序以包的形式提供，其中包含处理数据的算法。这些包包括 Go 自带的包和您自己创建的包，以便操作您的数据。

因此，在本章中，您将学习以下内容：

+   大 O 符号

+   两种排序算法

+   `sort.Slice()`函数

+   链表

+   树

+   在 Go 中创建自己的哈希表数据结构

+   Go 包

+   Go 中的垃圾回收（GC）

# 关于算法

了解算法及其工作方式肯定会在您需要处理大量数据时帮助您。此外，如果您选择对于特定工作使用错误的算法，可能会减慢整个过程并使您的软件无法使用。

传统的 Unix 命令行实用程序，如`awk(1)`、`sed(1)`、`vi(1)`、`tar(1)`和`cp(1)`，是好算法如何帮助的很好的例子，这些实用程序可以处理比机器内存大得多的文件。这在早期的 Unix 时代非常重要，因为当时 Unix 机器上的总 RAM 量大约为 64K 甚至更少！

# 大 O 符号

**大 O 符号**用于描述算法的复杂性，这与其性能直接相关。算法的效率是通过其计算复杂性来判断的，这主要与算法需要访问其输入数据的次数有关。通常，您会想了解最坏情况和平均情况。

因此，O(n)算法（其中 n 是输入的大小）被认为比 O(n²)算法更好，后者又比 O(n³)算法更好。然而，最糟糕的算法是具有 O(n!)运行时间的算法，因为这使得它们几乎无法用于超过 300 个元素的输入。请注意，大 O 符号更多地是关于估计而不是给出精确值。因此，它主要用作比较值而不是绝对值。

此外，大多数内置类型的 Go 查找操作，比如查找地图键的值或访问数组元素，都具有常数时间，用 O(1)表示。这意味着内置类型通常比自定义类型更快，通常应该优先选择它们，除非你想完全控制后台发生的事情。另外，并非所有数据结构都是平等的。一般来说，数组操作比地图操作更快，而地图比数组更灵活！

# 排序算法

最常见的算法类别涉及对数据进行排序，即将其放置在给定顺序中。最著名的两种排序算法如下：

+   **快速排序**：这被认为是最快的排序算法之一。快速排序对其数据进行排序所需的平均时间为 O(n log n)，但在最坏情况下可能增长到 O(n²)，这主要与数据呈现方式有关。

+   **冒泡排序**：这个算法非常容易实现，平均复杂度为 O(n²)。如果您想开始学习排序，可以先从冒泡排序开始，然后再研究更难开发的算法。

尽管每种算法都有其缺点，但如果您没有大量数据，那么只要它能完成工作，算法就不是真正重要的。

您应该记住的是，Go 内部实现排序的方式无法由开发人员控制，并且将来可能会发生变化；因此，如果您想完全控制排序，应该编写自己的实现。

# sort.Slice()函数

本节将说明首次出现在 Go 版本 1.8 中的`sort.Slice()`函数的用法。该函数的用法将在`sortSlice.go`中进行说明，该文件将分为三部分呈现。

第一部分是程序的预期序言和新结构类型的定义，如下所示：

```go
package main 

import ( 
   "fmt" 
   "sort" 
) 

type aStructure struct { 
   person string 
   height int 
   weight int 
} 
```

正如您所期望的，您必须导入`sort`包才能使用其`Slice()`函数。

第二部分包含了切片的定义，其中包含四个元素：

```go
func main() { 

   mySlice := make([]aStructure, 0) 
   a := aStructure{"Mihalis", 180, 90}

   mySlice = append(mySlice, a) 
   a = aStructure{"Dimitris", 180, 95} 
   mySlice = append(mySlice, a) 
   a = aStructure{"Marietta", 155, 45} 
   mySlice = append(mySlice, a) 
   a = aStructure{"Bill", 134, 40} 
   mySlice = append(mySlice, a)
```

因此，在第一部分中，您声明了一个结构的切片，该切片将在程序的其余部分中以两种方式进行排序，其中包含以下代码：

```go
   fmt.Println("0:", mySlice) 
   sort.Slice(mySlice, func(i, j int) bool { 
         return mySlice[i].weight <mySlice[j].weight 
   }) 
   fmt.Println("<:", mySlice) 
   sort.Slice(mySlice, func(i, j int) bool { 
         return mySlice[i].weight >mySlice[j].weight 
   }) 
   fmt.Println(">:", mySlice) 
} 
```

这段代码包含了所有的魔法：您只需定义您想要对`slice`进行`sort`的方式，Go 就会完成其余工作。`sort.Slice()`函数将匿名排序函数作为其参数之一；另一个参数是您想要`sort`的`slice`变量的名称。请注意，排序后的切片保存在`slice`变量中。

执行`sortSlice.go`将生成以下输出：

```go
$ go run sortSlice.go
0: [{Mihalis 180 90} {Dimitris 180 95} {Marietta 155 45} {Bill 134 40}]
<: [{Bill 134 40} {Marietta 155 45} {Mihalis 180 90} {Dimitris 180 95}]
>: [{Dimitris 180 95} {Mihalis 180 90} {Marietta 155 45} {Bill 134 40}]
```

如您所见，您可以通过在 Go 代码中更改一个字符来轻松地按升序或降序进行`sort`！

此外，如果您的 Go 版本不支持`sort.Slice()`，您将收到类似以下的错误消息：

```go
$ go version
go version go1.3.3 linux/amd64
$ go run sortSlice.go
# command-line-arguments
./sortSlice.go:27: undefined: sort.Slice
./sortSlice.go:31: undefined: sort.Slice
```

# Go 中的链表

**链表**是具有有限元素集的结构，其中每个元素使用至少两个内存位置：一个用于存储数据，另一个用于将当前元素链接到构成链表的元素序列中的下一个元素的指针。链表的最大优势是易于理解和实现，并且足够通用，可用于许多不同情况并模拟许多不同类型的数据。

链表的第一个元素称为**头部**，而列表的最后一个元素通常称为**尾部**。定义链表时，首先要做的是将列表的头部保留在单独的变量中，因为头部是您需要访问整个链表的唯一内容。

请注意，如果丢失单链表的第一个节点的指针，将无法再次找到它。

以下图显示了链表和双向链表的图形表示。双向链表更灵活，但需要更多的维护：

![](img/834e20cc-7099-4649-b740-da9fa61fbd3d.png)

链表和双向链表的图形表示

因此，在本节中，我们将介绍在`linkedList.go`中保存的 Go 中链表的简单实现。

当创建自己的数据结构时，最重要的元素是节点的定义，通常使用结构来实现。

`linkedList.go`的代码将分为四部分呈现。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
) 
```

第二部分包含以下 Go 代码：

```go
type Node struct { 
   Value int 
   Next  *Node 
} 

func addNode(t *Node, v int) int { 
   if root == nil { 
         t = &Node{v, nil} 
         root = t 
         return 0 
   } 

   if v == t.Value { 
         fmt.Println("Node already exists:", v) 
         return -1 
   } 

   if t.Next == nil { 
         t.Next = &Node{v, nil} 
         return -2 
   } 

   return addNode(t.Next, v)

} 
```

在这里，您定义了将保存列表中每个元素的结构以及允许您向列表添加新节点的函数。为了避免重复条目，您应该检查值是否已经存在于列表中。请注意，`addNode()`是一个递归函数，因为它调用自身，这种方法可能比迭代稍慢，需要更多的内存。

代码的第三部分是`traverse()`函数：

```go
func traverse(t *Node) { 
   if t == nil { 
         fmt.Println("-> Empty list!") 
         return 
   } 

   for t != nil {

         fmt.Printf("%d -> ", t.Value) 
         t = t.Next 
   } 
   fmt.Println() 
} 
```

`for`循环实现了访问链表中所有节点的迭代方法。

最后一部分如下：

```go
var root = new(Node)
func main() { 
   fmt.Println(root) 
   root = nil 
   traverse(root) 
   addNode(root, 1) 
   addNode(root, 1) 
   traverse(root) 
   addNode(root, 10) 
   addNode(root, 5) 
   addNode(root, 0) 
   addNode(root, 0) 
   traverse(root) 
   addNode(root, 100) 
   traverse(root) 
}
```

在本书中首次看到不是常量的全局变量的使用。全局变量可以从程序的任何地方访问和更改，这使得它们的使用既实用又危险。使用名为`root`的全局变量来保存链表的`root`是为了显示链表是否为空。这是因为在 Go 中，整数值被初始化为`0`；因此`new(Node)`实际上是`{0 <nil>}`，这使得在不传递额外变量给每个操作链表的函数的情况下，无法判断列表的头部是否为空。

执行`linkedList.go`将生成以下输出：

```go
$ go run linkedList.go
&{0 <nil>}
-> Empty list!
Node already exists: 1
1 ->
Node already exists: 0
1 -> 10 -> 5 -> 0 ->
1 -> 10 -> 5 -> 0 -> 100 ->
```

# Go 中的树

**图**是一个有限且非空的顶点和边的集合。**有向图**是一个边带有方向的图。**有向无环图**是一个没有循环的有向图。**树**是一个满足三个原则的有向无环图：首先，它有一个根节点：树的入口点；其次，除了根之外，每个顶点只有一个入口点；第三，存在一条连接根和每个顶点的路径，并且属于树。

因此，根是树的第一个节点。每个节点可以连接到一个或多个节点，具体取决于树的类型。如果每个节点只指向一个其他节点，那么树就是一个链表！

最常用的树类型是二叉树，因为每个节点最多可以有两个子节点。下图显示了二叉树数据结构的图形表示：

![](img/4f4a000e-6fe7-4025-81f3-ff52adfbd59f.png)

二叉树

所呈现的代码只会向您展示如何创建二叉树以及如何遍历它以打印出所有元素，以证明 Go 可以用于创建树数据结构。因此，它不会实现二叉树的完整功能，其中还包括删除树节点和平衡树。

`tree.go`的代码将分为三部分呈现。

第一部分是预期的序言以及节点的定义，如下所示：

```go
package main 

import ( 
   "fmt" 
   "math/rand" 
   "time" 
) 
type Tree struct { 
   Left  *Tree 
   Value int 
   Right *Tree 
} 
```

第二部分包含允许您遍历树以打印所有元素、使用随机生成的数字创建树以及将节点插入其中的函数：

```go
func traverse(t *Tree) { 
   if t == nil { 
         return 
   } 
   traverse(t.Left) 
   fmt.Print(t.Value, " ") 
   traverse(t.Right) 
} 

func create(n int) *Tree { 
   var t *Tree 
   rand.Seed(time.Now().Unix()) 
   for i := 0; i< 2*n; i++ { 
         temp := rand.Intn(n) 
         t = insert(t, temp) 
   } 
   return t 
} 

func insert(t *Tree, v int) *Tree { 
   if t == nil { 
         return&Tree{nil, v, nil} 
   } 
   if v == t.Value { 
         return t 
   } 
   if v <t.Value { 
         t.Left = insert(t.Left, v) 
         return t 
   } 
   t.Right = insert(t.Right, v) 
   return t 
} 
```

`insert()`的第二个`if`语句检查树中是否已经存在值，以免重复添加。第三个`if`语句标识新元素将位于当前节点的左侧还是右侧。

最后一部分是`main()`函数的实现：

```go
func main() { 
   tree := create(30) 
   traverse(tree) 
   fmt.Println() 
   fmt.Println("The value of the root of the tree is", tree.Value) 
} 
```

执行`tree.go`将生成以下输出：

```go
$ go run tree.go
0 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 21 22 23 24 25 26 27 28 29
The value of the root of the tree is 16
```

请注意，由于树的节点的值是随机生成的，程序的输出每次运行时都会不同。如果您希望始终获得相同的元素，则在`create()`函数中使用种子值的常量。

# 在 Go 中开发哈希表

严格来说，**哈希表**是一种数据结构，它存储一个或多个键值对，并使用键的`hashFunction`计算出数组中的桶或槽的索引，从中可以检索到正确的值。理想情况下，`hashFunction`应该将每个键分配到一个唯一的桶中，前提是您有所需数量的桶。

一个良好的`hashFunction`必须能够产生均匀分布的哈希值，因为拥有未使用的桶或桶的基数差异很大是低效的。此外，`hashFunction`应该能够一致地工作，并为相同的键输出相同的哈希值，否则将无法找到所需的信息！如果你认为哈希表并不那么有用、方便或聪明，你应该考虑以下：当哈希表有*n*个键和*k*个桶时，其搜索速度从线性搜索的 O(n)变为 O(n/k)!虽然改进看起来很小，但你应该意识到，对于只有 20 个插槽的哈希数组，搜索时间将减少 20 倍！这使得哈希表非常适用于诸如字典或任何其他类似的应用程序，其中需要搜索大量数据。尽管使用大量桶会增加程序的复杂性和内存使用量，但有时这是值得的。

下图显示了一个具有 10 个桶的简单哈希表的图形表示。很容易理解`hashFunction`是取模运算符：

![](img/30731fad-6b66-43d1-8e49-25ae564255a1.png)

一个简单的哈希表

尽管所呈现的哈希表版本使用数字，因为它们更容易实现和理解，但只要你能找到合适的`hashFunction`来处理输入，你可以使用任何数据类型。`hash.go`的源代码将分为三部分呈现。

第一个是以下内容：

```go
package main 

import ( 
   "fmt" 
) 

type Node struct { 
   Value int 
   Next  *Node 
} 

type HashTablestruct { 
   Table map[int]*Node

   Size  int 
} 
```

`Node struct`的定义取自您之前看到的链表的实现。使用`map`变量的`Table`而不是切片的原因是，切片的索引只能是自然数，而`map`的键可以是任何东西。

第二部分包含以下 Go 代码：

```go
func hashFunction(i, size int) int { 
   return (i % size) 
} 

func insert(hash *HashTable, value int) int { 
   index := hashFunction(value, hash.Size) 
   element := Node{Value: value, Next: hash.Table[index]} 
   hash.Table[index] = &element 
   return index 
} 

func traverse(hash *HashTable) { 
   for k := range hash.Table { 
         if hash.Table[k] != nil { 
               t := hash.Table[k] 
               for t != nil { 
                     fmt.Printf("%d -> ", t.Value) 
                     t = t.Next 
               } 
               fmt.Println() 
         } 
   } 
}
```

请注意，`traverse()`函数使用`linkedList.go`中的 Go 代码来遍历哈希表中每个桶的元素。另外，请注意，`insert`函数不会检查值是否已经存在于哈希表中，以节省空间，但通常情况下并非如此。另外，出于速度和简单性的考虑，新元素被插入到每个列表的开头。

最后一部分包含了`main()`函数的实现：

```go
func main() { 
   table := make(map[int]*Node, 10) 
   hash := &HashTable{Table: table, Size: 10} 
   fmt.Println("Number of spaces:", hash.Size) 
   for i := 0; i< 95; i++ { 
         insert(hash, i) 
   } 
   traverse(hash) 
} 
```

执行`hash.go`将生成以下输出，证明哈希表按预期工作：

```go
$ go run hash.go
Number of spaces: 10 89 -> 79 -> 69 -> 59 -> 49 -> 39 -> 29 -> 19 -> 9 ->
86 -> 76 -> 66 -> 56 -> 46 -> 36 -> 26 -> 16 -> 6 ->
92 -> 82 -> 72 -> 62 -> 52 -> 42 -> 32 -> 22 -> 12 -> 2 ->
94 -> 84 -> 74 -> 64 -> 54 -> 44 -> 34 -> 24 -> 14 -> 4 ->
85 -> 75 -> 65 -> 55 -> 45 -> 35 -> 25 -> 15 -> 5 ->
87 -> 77 -> 67 -> 57 -> 47 -> 37 -> 27 -> 17 -> 7 ->
88 -> 78 -> 68 -> 58 -> 48 -> 38 -> 28 -> 18 -> 8 ->
90 -> 80 -> 70 -> 60 -> 50 -> 40 -> 30 -> 20 -> 10 -> 0 ->
91 -> 81 -> 71 -> 61 -> 51 -> 41 -> 31 -> 21 -> 11 -> 1 ->
93 -> 83 -> 73 -> 63 -> 53 -> 43 -> 33 -> 23 -> 13 -> 3 ->
```

如果你多次执行`hash.go`，你会发现打印行的顺序会变化。这是因为`traverse()`函数中`range hash.Table`的输出是无法预测的，这是因为 Go 对哈希的返回顺序没有指定。

# 关于 Go 包

包用于将相关函数和常量分组，以便您可以轻松地传输它们并在自己的 Go 程序中使用。因此，除了主包之外，包不是独立的程序。

每个 Go 发行版都附带许多有用的 Go 包，包括以下内容：

+   `net`包：这支持可移植的 TCP 和 UDP 连接

+   `http`包：这是 net 包的一部分，提供了 HTTP 服务器和客户端的实现

+   `math`包：这提供了数学函数和常量

+   `io`包：这处理原始的输入和输出操作

+   `os`包：这为您提供了一个便携式的操作系统功能接口

+   `time`包：这允许您处理时间和日期

有关标准 Go 包的完整列表，请参阅[`golang.org/pkg/`](https://golang.org/pkg/)。我强烈建议您在开始开发自己的函数和包之前，先了解 Go 提供的所有包，因为你要寻找的功能很可能已经包含在标准 Go 包中。

# 使用标准 Go 包

您可能已经知道如何使用标准的 Go 包。但是，您可能不知道的是，一些包有一个结构。例如，`net`包有几个子目录，命名为`http`、`mail`、`rpc`、`smtp`、`textproto`和`url`，应该分别导入为`net/http`、`net/mail`、`net/rpc`、`net/smtp`、`net/textproto`和`net/url`。Go 在这些情况下对包进行分组，但是如果它们是为了分发而不是功能而分组，这些包也可以是独立的包。

您可以使用`godoc`实用程序查找有关 Go 标准包的信息。因此，如果您正在寻找有关`net`包的信息，您应该执行`godoc net`。

# 创建您自己的包

包使得大型软件系统的设计、实现和维护更加简单和容易。此外，它们允许多个程序员在同一个项目上工作而不会发生重叠。因此，如果您发现自己一直在使用相同的函数，您应该认真考虑将它们包含在您自己的 Go 包中。

Go 包的源代码，可以包含多个文件，可以在一个目录中找到，该目录以包的名称命名，除了主包，主包可以有任何名称。

在本节中将开发的`aSimplePackage.go`文件的 Go 代码将分为两部分呈现。

第一部分是以下内容：

```go
package aSimplePackage 

import ( 
   "fmt" 
) 
```

这里没有什么特别的；您只需定义包的名称并包含必要的导入语句，因为一个包可以依赖于其他包。

第二部分包含以下 Go 代码：

```go
const Pi = "3.14159" 

func Add(x, y int) int { 
   return x + y 
} 

func Println(x int) { 
   fmt.Println(x) 
} 
```

因此，`aSimplePackage`包提供了两个函数和一个常量。

完成`aSimplePackage.go`的代码编写后，您应该执行以下命令，以便能够在其他 Go 程序或包中使用该包：

```go
$ mkdir ~/go
$ mkdir ~/go/src
$ mkdir ~/go/src/aSimplePackage
$ export GOPATH=~/go
$ vi ~/go/src/aSimplePackage/aSimplePackage.go
$ go install aSimplePackage 
```

除了前两个`mkdir`命令，您应该为您创建的每个 Go 包执行所有这些操作，这两个命令只需要执行一次。

如您所见，每个包都需要在`~/go/src`目录下有自己的文件夹。在执行上述命令后，`go tool`将自动生成一个 Go 包的`ar(1)`存档文件，该文件刚刚在`pkg`目录中编译完成：

```go
$ ls -lR ~/go
total 0
drwxr-xr-x  3 mtsouk  staff  102 Apr  4 22:35 pkg
drwxr-xr-x  3 mtsouk  staff  102 Apr  4 22:35 src

/Users/mtsouk/go/pkg:
total 0
drwxr-xr-x  3 mtsouk  staff  102 Apr  4 22:35 darwin_amd64

/Users/mtsouk/go/pkg/darwin_amd64:
total 8
-rw-r--r--  1 mtsouk  staff  2918 Apr  4 22:35 aSimplePackage.a

/Users/mtsouk/go/src:
total 0
drwxr-xr-x  3 mtsouk  staff  102 Apr  4 22:35 aSimplePackage

/Users/mtsouk/go/src/aSimplePackage:
total 8
-rw-r--r--  1 mtsouk  staff  148 Apr  4 22:30 aSimplePackage.go
```

尽管您现在已经准备好使用`aSimplePackage`包，但是没有一个独立的程序，您无法看到包的功能。

# 私有变量和函数

私有变量和函数与公共变量和函数不同，它们只能在包内部使用和调用。控制哪些函数和变量是公共的或不公共的也被称为封装。

Go 遵循一个简单的规则，即以大写字母开头的函数、变量、类型等都是公共的，而以小写字母开头的函数、变量、类型等都是私有的。但是，这个规则不影响包名。

现在您应该明白为什么`fmt.Printf()`函数的命名是这样的，而不是`fmt.printf()`。

为了说明这一点，我们将对`aSimplePackage.go`模块进行一些更改，并添加一个私有变量和一个私有函数。新的独立包的名称将是`anotherPackage.go`。您可以使用`diff(1)`命令行实用程序查看对其所做的更改：

```go
$ diff aSimplePackage.go anotherPackage.go
1c1
<packageaSimplePackage
---
>packageanotherPackage
7a8
>const version = "1.1"
15a17,20
>
>func Version() {
>     fmt.Println("The version of the package is", version)
> }
```

# init()函数

每个 Go 包都可以有一个名为`init()`的函数，在执行开始时自动执行。因此，让我们在`anotherPackage.go`包的代码中添加以下`init()`函数：

```go
func init() { 
   fmt.Println("The init function of anotherPackage") 
} 
```

`init()`函数的当前实现是简单的，没有特殊操作。但是，有时您希望在开始使用包之前执行重要的初始化操作，例如打开数据库和网络连接：在这些相对罕见的情况下，`init()`函数是非常宝贵的。

# 使用您自己的 Go 包

本小节将向你展示如何在你自己的 Go 程序中使用`aSimplePackage`和`anotherPackage`包，通过展示两个名为`usePackage.go`和`privateFail.go`的小型 Go 程序。

为了使用`GOPATH`目录下的`aSimplePackage`包，你需要在另一个 Go 程序中编写以下 Go 代码：

```go
package main 

import ( 
   "aSimplePackage" 
   "fmt" 
) 

func main() { 
   temp := aSimplePackage.Add(5, 10) 
   fmt.Println(temp)

   fmt.Println(aSimplePackage.Pi) 
} 
```

首先，如果`aSimplePackage`尚未编译并位于预期位置，编译过程将失败，并显示类似以下的错误消息：

```go
$ go run usePackage.go
usePackage.go:4:2: cannot find package "aSimplePackage" in any of:
      /usr/local/Cellar/go/1.8/libexec/src/aSimplePackage (from $GOROOT)
      /Users/mtsouk/go/src/aSimplePackage (from $GOPATH)
```

然而，如果`aSimplePackage`可用，`usePackage.go`将会被成功执行：

```go
$ go run usePackage.go
15
3.14159
```

现在，让我们看看另一个使用`anotherPackage`的小程序的 Go 代码：

```go
package main 

import ( 
   "anotherPackage" 
   "fmt" 
) 

func main() { 
   anotherPackage.Version() 
   fmt.Println(anotherPackage.version) 
   fmt.Println(anotherPackage.Pi) 
} 
```

如果你尝试从`anotherPackage`调用私有函数或使用私有变量，你的 Go 程序`privateFail.go`将无法运行，并显示以下错误消息：

```go
$ go run privateFail.go
# command-line-arguments
./privateFail.go:10: cannot refer to unexported name anotherPackage.version
./privateFail.go:10: undefined: anotherPackage.version
```

我真的很喜欢显示错误消息，因为大多数书籍都试图隐藏它们，好像它们不存在一样。当我学习 Go 时，我花了大约 3 个小时的调试，直到我发现一个我无法解释的错误消息的原因是一个变量的名字！

然而，如果你从`privateFail.go`中删除对私有变量的调用，程序将在没有错误的情况下执行。此外，你会看到`init()`函数实际上会自动执行：

```go
$ go run privateFail.go
The init function of anotherPackage
The version of the package is 1.1
3.14159
```

# 使用外部 Go 包

有时候，包可以在互联网上找到，并且你希望通过指定它们的互联网地址来使用它们。一个这样的例子是 Go 的`MySQL`驱动程序，可以在`github.com/go-sql-driver/mysql`找到。

看看以下的 Go 代码，保存为`useMySQL.go`：

```go
package main 

import ( 
   "fmt" 
   _ "github.com/go-sql-driver/mysql"
) 

func main() { 
   fmt.Println("Using the MySQL Go driver!") 
} 
```

使用`_`作为包标识符将使编译器忽略包未被使用的事实：绕过编译器的唯一合理理由是当你的未使用包中有一个你想要执行的`init`函数时。另一个合理的理由是为了说明一个 Go 概念！

如果你尝试执行`useMySQL.go`，编译过程将失败：

```go
$ go run useMySQL.go
useMySQL.go:5:2: cannot find package "github.com/go-sql-driver/mysql" in any of:
      /usr/local/Cellar/go/1.8/libexec/src/github.com/go-sql-driver/mysql (from $GOROOT)
      /Users/mtsouk/go/src/github.com/go-sql-driver/mysql (from $GOPATH)
```

为了编译`useMySQL.go`，你应该首先执行以下步骤：

```go
$ go get github.com/go-sql-driver/mysql
$ go run useMySQL.go
Using the MySQL Go driver!
```

成功下载所需的包后，`~/go`目录的内容将验证所需的 Go 包已被下载：

```go
$ ls -lR ~/go
total 0
drwxr-xr-x  3 mtsouk  staff  102 Apr  4 22:35 pkg
drwxr-xr-x  5 mtsouk  staff  170 Apr  6 21:32 src

/Users/mtsouk/go/pkg:
total 0
drwxr-xr-x  5 mtsouk  staff  170 Apr  6 21:32 darwin_amd64

/Users/mtsouk/go/pkg/darwin_amd64:
total 24
-rw-r--r--  1 mtsouk  staff  2918 Apr  4 23:07 aSimplePackage.a
-rw-r--r--  1 mtsouk  staff  6102 Apr  4 22:50 anotherPackage.a
drwxr-xr-x  3 mtsouk  staff   102 Apr  6 21:32 github.com

/Users/mtsouk/go/pkg/darwin_amd64/github.com:
total 0
drwxr-xr-x  3 mtsouk  staff  102 Apr  6 21:32 go-sql-driver

/Users/mtsouk/go/pkg/darwin_amd64/github.com/go-sql-driver:
total 728
-rw-r--r--  1 mtsouk  staff  372694 Apr  6 21:32 mysql.a

/Users/mtsouk/go/src:
total 0
drwxr-xr-x  3 mtsouk  staff  102 Apr  4 22:35 aSimplePackage
drwxr-xr-x  3 mtsouk  staff  102 Apr  4 22:50 anotherPackage
drwxr-xr-x  3 mtsouk  staff  102 Apr  6 21:32 github.com

/Users/mtsouk/go/src/aSimplePackage:
total 8
-rw-r--r--  1 mtsouk  staff  148 Apr  4 22:30 aSimplePackage.go

/Users/mtsouk/go/src/anotherPackage:
total 8
-rw-r--r--@ 1 mtsouk  staff  313 Apr  4 22:50 anotherPackage.go

/Users/mtsouk/go/src/github.com:
total 0
drwxr-xr-x  3 mtsouk  staff  102 Apr  6 21:32 go-sql-driver

/Users/mtsouk/go/src/github.com/go-sql-driver:
total 0
drwxr-xr-x  35 mtsouk  staff  1190 Apr  6 21:32 mysql

/Users/mtsouk/go/src/github.com/go-sql-driver/mysql:
total 584
-rw-r--r--  1 mtsouk  staff   2066 Apr  6 21:32 AUTHORS
-rw-r--r--  1 mtsouk  staff   5581 Apr  6 21:32 CHANGELOG.md
-rw-r--r--  1 mtsouk  staff   1091 Apr  6 21:32 CONTRIBUTING.md
-rw-r--r--  1 mtsouk  staff  16726 Apr  6 21:32 LICENSE
-rw-r--r--  1 mtsouk  staff  18610 Apr  6 21:32 README.md
-rw-r--r--  1 mtsouk  staff    470 Apr  6 21:32 appengine.go
-rw-r--r--  1 mtsouk  staff   4965 Apr  6 21:32 benchmark_test.go
-rw-r--r--  1 mtsouk  staff   3339 Apr  6 21:32 buffer.go
-rw-r--r--  1 mtsouk  staff   8405 Apr  6 21:32 collations.go
-rw-r--r--  1 mtsouk  staff   8525 Apr  6 21:32 connection.go
-rw-r--r--  1 mtsouk  staff   1831 Apr  6 21:32 connection_test.go
-rw-r--r--  1 mtsouk  staff   3111 Apr  6 21:32 const.go
-rw-r--r--  1 mtsouk  staff   5036 Apr  6 21:32 driver.go
-rw-r--r--  1 mtsouk  staff   4246 Apr  6 21:32 driver_go18_test.go
-rw-r--r--  1 mtsouk  staff  47090 Apr  6 21:32 driver_test.go
-rw-r--r--  1 mtsouk  staff  13046 Apr  6 21:32 dsn.go
-rw-r--r--  1 mtsouk  staff   7872 Apr  6 21:32 dsn_test.go
-rw-r--r--  1 mtsouk  staff   3798 Apr  6 21:32 errors.go
-rw-r--r--  1 mtsouk  staff    989 Apr  6 21:32 errors_test.go
-rw-r--r--  1 mtsouk  staff   4571 Apr  6 21:32 infile.go
-rw-r--r--  1 mtsouk  staff  31362 Apr  6 21:32 packets.go
-rw-r--r--  1 mtsouk  staff   6453 Apr  6 21:32 packets_test.go
-rw-r--r--  1 mtsouk  staff    600 Apr  6 21:32 result.go
-rw-r--r--  1 mtsouk  staff   3698 Apr  6 21:32 rows.go
-rw-r--r--  1 mtsouk  staff   3609 Apr  6 21:32 statement.go
-rw-r--r--  1 mtsouk  staff    729 Apr  6 21:32 transaction.go
-rw-r--r--  1 mtsouk  staff  17924 Apr  6 21:32 utils.go
-rw-r--r--  1 mtsouk  staff   5784 Apr  6 21:32 utils_test.go
```

# go clean 命令

有时候，你正在开发一个使用大量非标准 Go 包的大型 Go 程序，并且希望从头开始编译过程。Go 允许你清理一个包的文件，以便稍后重新创建它。以下命令清理一个包，而不影响包的代码：

```go
$ go clean -x -i aSimplePackage
cd /Users/mtsouk/go/src/aSimplePackage
rm -f aSimplePackage.test aSimplePackage.test.exe
rm -f /Users/mtsouk/go/pkg/darwin_amd64/aSimplePackage.a
```

同样，你也可以清理从互联网下载的包，这也需要使用其完整路径：

```go
$ go clean -x -i github.com/go-sql-driver/mysql
cd /Users/mtsouk/go/src/github.com/go-sql-driver/mysql
rm -f mysql.test mysql.test.exe appengine appengine.exe
rm -f /Users/mtsouk/go/pkg/darwin_amd64/github.com/go-sql-driver/mysql.a
```

请注意，当你想将项目转移到另一台机器上而不包括不必要的文件时，`go clean`命令也特别有用。

# 垃圾收集

在本节中，我们将简要讨论 Go 如何处理 GC，它试图高效地释放未使用的内存。`garbageCol.go`的 Go 代码可以分为两部分。

第一部分如下：

```go
package main 

import ( 
   "fmt" 
   "runtime" 
   "time" 
) 

func printStats(mem runtime.MemStats) { 
   runtime.ReadMemStats(&mem) 
   fmt.Println("mem.Alloc:", mem.Alloc) 
   fmt.Println("mem.TotalAlloc:", mem.TotalAlloc) 
   fmt.Println("mem.HeapAlloc:", mem.HeapAlloc) 
   fmt.Println("mem.NumGC:", mem.NumGC) 
   fmt.Println("-----") 
} 
```

每当你想要读取最新的内存统计信息时，你应该调用`runtime.ReadMemStats()`函数。

第二部分包含了`main()`函数的实现，其中包含以下 Go 代码：

```go
func main() { 
   var memruntime.MemStats 
   printStats(mem) 

   for i := 0; i< 10; i++ { 
         s := make([]byte, 100000000) 
         if s == nil { 
               fmt.Println("Operation failed!") 
         } 
   } 
   printStats(mem) 

   for i := 0; i< 10; i++ { 
         s := make([]byte, 100000000) 
         if s == nil { 
               fmt.Println("Operation failed!") 
         } 
         time.Sleep(5 * time.Second) 
   } 
   printStats(mem)

} 
```

在这里，你尝试获取大量内存，以触发垃圾收集器的使用。

执行`garbageCol.go`会生成以下输出：

```go
$ go run garbageCol.go
mem.Alloc: 53944
mem.TotalAlloc: 53944
mem.HeapAlloc: 53944
mem.NumGC: 0
-----
mem.Alloc: 100071680
mem.TotalAlloc: 1000146400
mem.HeapAlloc: 100071680
mem.NumGC: 10
-----
mem.Alloc: 66152
mem.TotalAlloc: 2000230496
mem.HeapAlloc: 66152
mem.NumGC: 20
-----
```

因此，输出呈现了与`garbageCol.go`程序使用的内存相关的属性信息。如果你想获得更详细的输出，可以执行`garbageCol.go`，如下所示：

```go
$ GODEBUG=gctrace=1 go run garbageCol.go
```

这个命令的版本将以以下格式给出信息：

```go
gc 11 @0.101s 0%: 0.003+0.083+0.020 ms clock, 0.030+0.059/0.033/0.006+0.16 mscpu, 95->95->0 MB, 96 MB goal, 8 P
```

`95->95->0 MB` 部分包含有关各种堆大小的信息，还显示了垃圾收集器的表现如何。第一个值是 GC 开始时的堆大小，而中间值显示了 GC 结束时的堆大小。第三个值是活动堆的大小。

# 您的环境

在本节中，我们将展示如何使用 `runtime` 包查找有关您的环境的信息：当您必须根据操作系统和您使用的 Go 版本采取某些操作时，这可能很有用。

使用 `runtime` 包查找有关您的环境的信息是直接的，并在 `runTime.go` 中有所说明：

```go
package main 

import ( 
   "fmt" 
   "runtime" 
) 

func main() { 
   fmt.Print("You are using ", runtime.Compiler, " ") 
   fmt.Println("on a", runtime.GOARCH, "machine") 
   fmt.Println("with Go version", runtime.Version()) 
   fmt.Println("Number of Goroutines:", runtime.NumGoroutine())
} 
```

只要您知道要从 runtime 包中调用什么，就可以获取所需的信息。这里的最后一个 `fmt.Println()` 命令显示有关 **goroutines** 的信息：您将在第九章*,* *Goroutines - Basic Features* 中了解更多关于 goroutines 的信息。

在 macOS 机器上执行 `runTime.go` 会生成以下输出：

```go
$ go run runTime.go
You are using gc on a amd64 machine
with Go version go1.8
Number of Goroutines: 1  
```

在使用旧版 Go 的 Linux 机器上执行 `runTime.go` 会得到以下结果：

```go
$ go run runTime.go
You are using gc on a amd64 machine
with Go version go1.3.3
Number of Goroutines: 4
```

# Go 经常更新！

在写完本章的最后，Go 进行了一点更新。因此，我决定在本书中包含这些信息，以便更好地了解 Go 的更新频率：

```go
$ date
Sat Apr  8 09:16:46 EEST 2017
$ go version
go version go1.8.1 darwin/amd64
```

# 练习

1.  访问 runtime 包的文档。

1.  创建您自己的结构，创建一个切片，并使用 `sort.Slice()` 对您创建的切片的元素进行排序。

1.  在 Go 中实现快速排序算法，并对一些随机生成的数字数据进行排序。

1.  实现一个双向链表。

1.  `tree.go` 的实现远未完成！尝试实现一个检查树中是否可以找到值的函数，以及一个允许您删除树节点的函数。

1.  同样，`linkedList.go` 文件的实现也是不完整的。尝试实现一个用于删除节点的函数，以及另一个用于在链表中某个位置插入节点的函数。

1.  再次，`hash.go` 的哈希表实现是不完整的，因为它允许重复条目。因此，在插入之前，实现一个在哈希表中搜索键的函数。

# 总结

在本章中，您学到了许多与算法和数据结构相关的知识。您还学会了如何使用现有的 Go 包以及如何开发自己的 Go 包。本章还讨论了 Go 中的垃圾收集以及如何查找有关您的环境的信息。

在下一章中，我们将开始讨论系统编程，并呈现更多的 Go 代码。更确切地说，第五章，*文件和目录*，将讨论如何在 Go 中处理文件和目录，如何轻松地遍历目录结构，以及如何使用 `flag` 包处理命令行参数。但更重要的是，我们将开始开发各种 Unix 命令行实用程序的 Go 版本。
