# Web 规模架构的基础

第五章，*介绍 Goophr*，第六章，*Goophr Concierge*，和第七章，*Goophr Librarian*，是关于从基本概念到运行各个组件并验证它们按预期工作的分布式搜索索引系统的设计和实现。在第八章，*部署 Goophr*，我们使用**docker-compose**将各个组件连接起来，以便我们可以以简单可靠的方式启动和连接所有组件。在过去的四章中，我们取得了相当大的进展，但你可能已经注意到我们在单台机器上运行了所有东西，很可能是我们的笔记本电脑或台式机。

理想情况下，我们应该尝试准备我们的分布式系统在大量用户负载下可靠工作，并将其暴露在 Web 上供一般使用。然而，现实情况是，我们将不得不对我们当前的系统进行大量升级，以使其足够可靠和有弹性，能够在真实世界的流量下工作。

在本章中，我们将讨论在尝试为 Web 设计时应该牢记的各种因素。我们将关注以下内容：

+   扩展 Web 应用程序

+   单体应用程序与微服务

+   部署选项

## 扩展 Web 应用程序

在本章中，我们将不讨论 Goophr，而是一个简单的用于博客的 Web 应用程序，以便我们可以专注于为 Web 扩展它。这样的应用程序可能包括运行数据库和博客服务器的单个服务器实例。

扩展 Web 应用程序是一个复杂的主题，我们将花费大量时间来讨论这个主题。正如我们将在本节中看到的，有多种方式可以扩展系统：

+   整体扩展系统

+   拆分系统并扩展各个组件

+   选择特定的解决方案以更好地扩展系统

让我们从最基本的设置开始，即单个服务器实例。

### 单个服务器实例

单服务器设置通常包括：

+   用于提供网页并处理服务器端逻辑的 Web 服务器

+   用于保存与博客相关的所有用户数据（博客文章、用户登录详细信息等）的数据库

以下图显示了这样一个服务器的外观：

![](img/74eb2b09-6ebf-4c93-9a5b-9db3bd3e2f4b.png)

该图显示了一个简单的设置，用户与博客服务器进行交互，博客服务器将在内部与数据库进行交互。这种在同一实例上设置数据库和博客服务器将仅在一定数量的用户上是高效和响应的。

当系统开始变慢或存储空间开始填满时，我们可以将我们的应用程序（数据库和博客服务器）重新部署到具有更多存储空间、RAM 和 CPU 功率的不同服务器实例上；这被称为**垂直扩展**。正如你可能怀疑的那样，这可能是耗时和不便的升级服务器的方式。如果我们能尽可能地推迟这次升级，那不是更好吗？

需要考虑的一个重要问题是，问题可能是由以下任何组合因素导致的：

+   由于数据库或博客服务器而导致内存不足

+   由于 Web 服务器或数据库需要更多 CPU 周期而导致性能下降

+   由于数据库的存储空间不足

为了解决上述任何因素，扩展完整应用程序并不是处理问题的最佳方式，因为我们在本可以用更少的资源解决问题的地方花费了很多钱！那么我们应该如何设计我们的系统，以便以正确的方式解决正确的问题呢？

### 为 Web 和数据库分层

如果我们考虑前面提到的三个问题，我们可以通过一两种方式解决每个问题。让我们首先看看它们：

**问题＃1**：内存不足

**解决方案**：

+   **由于数据库**：为数据库增加 RAM

+   **由于博客服务器**：为博客服务器增加 RAM

**问题＃2**：性能下降

**解决方案**：

+   **由于数据库**：增加数据库的 CPU 功率

+   **由于博客服务器**：增加博客服务器的 CPU 功率

**问题＃3**：存储空间不足

**解决方案**：

+   **由于数据库**：增加数据库的存储空间

使用此列表，我们可以根据我们面临的特定问题随时升级我们的系统。然而，我们首先需要正确识别导致问题的组件。因此，即使在我们开始垂直扩展我们的应用程序之前，我们也应该像图中所示将我们的数据库与 Web 服务器分开。

![](img/cf3987e3-753d-44d1-844f-a0a4ac2ee2fc.png)

具有数据库和博客服务器在单独的服务器实例上的新设置将使我们能够监视哪个组件存在问题，并且仅垂直扩展该特定组件。我们应该能够使用这种新设置为更大的用户流量提供服务。

然而，随着服务器负载的增加，我们可能会遇到其他问题。例如，如果我们的博客服务器变得无响应会发生什么？我们将无法继续提供博客文章，也没有人能够在博客文章上发表评论。这是没有人愿意面对的情况。如果我们能够在博客服务器宕机时继续提供流量，那不是很好吗？

### 多个服务器实例

使用单个服务器实例为我们的博客服务器或任何应用程序（业务逻辑）服务器提供大量用户流量是危险的，因为我们实质上正在创建一个单点故障。避免这种情况的最合乎逻辑和最简单的方法是复制我们的博客服务器实例以处理传入的用户流量。将单个服务器扩展到多个实例的这种方法称为**横向扩展**。然而，这带来了一个问题：我们如何可靠地在博客服务器的各个实例之间分发流量？为此，我们使用**负载均衡器**。

#### 负载均衡器

负载均衡器是一种 HTTP 服务器，负责根据开发人员定义的规则将流量（路由）分发到各种 Web 服务器。总的来说，负载均衡器是一个非常快速和专业的应用程序。在 Web 服务器中尝试实现类似的逻辑可能不是最佳选择，因为您的 Web 服务器可用资源必须在处理业务逻辑的请求和需要路由的请求之间进行分配。此外，负载均衡器为我们提供了许多开箱即用的功能，例如：

+   **负载均衡算法**：以下是一些负载均衡的算法。

+   **随机**：在服务器之间随机分发。

+   **轮询**：在服务器之间均匀顺序地分发。

+   **不对称负载**：以一定比例在服务器之间分发。例如，对于 100 个请求，将 80 个发送到 A 服务器，20 个发送到 B 服务器。

+   **最少连接**：将新请求发送到具有最少活动连接数的服务器（不对称负载也可以与最少连接集成）。

+   **会话持久性**：想象一个电子商务网站，用户已将商品添加到购物车中，购物车中的商品信息存储在 A 服务器上。然而，当用户想要完成购买时，请求被发送到另一台服务器 B！这对用户来说是一个问题，因为与他的购物车相关的所有详细信息都在 A 服务器上。负载均衡器可以确保将这些请求重定向到相关的服务器。

+   **HTTP 压缩**：负载均衡器还可以使用`gzip`压缩传出响应，以便向用户发送更少的数据。这往往会极大地改善用户体验。

+   **HTTP 缓存**：对于提供 REST API 内容的站点，许多文件可以被缓存，因为它们不经常更改，并且缓存的内容可以更快地传递。

根据使用的负载均衡器，它们可以提供比上述列出的更多功能。这应该让人了解负载均衡器的能力。

以下图显示了负载均衡器和多个服务器如何一起工作：

![](img/0a855d3a-5d7c-4b76-9f86-d8b145ad2f71.png)

用户的请求到达负载均衡器，然后将请求路由到博客服务器的多个实例之一。然而，请注意，即使现在我们仍然在使用相同的数据库进行读写操作。

### 多可用区域

在前一节中，我们谈到了单点故障以及为什么有多个应用服务器实例是一件好事。我们可以进一步扩展这个概念；如果我们所有的服务器都在一个位置，由于某种重大故障或故障，所有的服务器都宕机了怎么办？我们将无法为任何用户流量提供服务。

我们可以看到，将我们的服务器放在一个位置也会造成单点故障。解决这个问题的方法是在多个位置提供应用服务器实例。然后下一个问题是：我们如何决定部署服务器的位置？我们应该将服务器部署到单个国家内的多个位置，还是应该将它们部署到多个国家？我们可以使用云计算术语重新表达问题如下。

我们需要决定是否要将我们的服务器部署到**多个区域**或**多个区域**，或者两者兼而有之。

重要的一点要注意的是，部署到多个区域可能会导致网络延迟，我们可能希望先部署到多个地区。然而，在我们部署到多个地区和区域之前，我们需要确保两个事实：

+   我们的网站有大量流量，我们的单服务器设置已经无法处理

+   我们有相当多的用户来自另一个国家，将服务器部署在他们附近的区域可能是一个好主意

一旦我们考虑了这些因素并决定部署到额外的区域和区域，我们的博客系统整体可能看起来像这样：

![](img/c8356325-b9ee-4215-9fbc-9bca596afbfa.png)

### 数据库

我们一直在扩展应用程序/博客服务器，并看到了如何垂直和水平扩展服务器，以及如何为整个系统的高可用性和性能因素化多个区域和区域。

您可能已经注意到在所有先前的设计中，我们仍然依赖单个数据库实例。到现在为止，您可能已经意识到，任何服务/服务器的单个实例都可能成为单点故障，并可能使系统完全停滞。

棘手的部分是，我们不能像为应用服务器那样简单地运行多个数据库实例的策略。我们之所以能够为应用服务器使用这种策略，是因为应用服务器负责业务逻辑，它自身维护的状态很少是临时的，而所有重要的信息都被推送到数据库中，这构成了真相的唯一来源，也是讽刺的是，单点故障的唯一来源。在我们深入探讨数据库扩展的复杂性和随之而来的挑战之前，让我们首先看一下需要解决的一个重要主题。

#### SQL 与 NoSQL

对于初学者来说，数据库有两种类型：

+   **关系型数据库**：这些使用 SQL（略有变化）来查询数据库

+   **NoSQL 数据库**：这些可以存储非结构化数据并使用特定的数据库查询语言

关系数据库已经存在很长时间了，人们已经付出了大量的努力来优化它们的性能，并使它们尽可能健壮。然而，可靠性和性能要求我们计划和组织我们的数据到定义良好的表和关系中。我们的数据受限于数据库表的模式。每当我们需要向我们的表中添加更多字段/列时，我们将不得不将表迁移到新的模式，并且这将要求我们创建迁移脚本来处理添加新字段，并且还要提供条件和数据来填充已存在的表中的新创建字段。

NoSQL 数据库往往具有更自由的结构。我们不需要为我们的表定义模式，因为数据存储为单行/文档。我们可以将任何模式的数据插入单个表中，然后对其进行查询。鉴于数据不受模式规则的限制，我们可能会将错误或格式不正确的数据插入到我们的数据库中。这意味着我们将不得不确保我们检索到正确的数据，并且还必须采取预防措施，以确保不同模式的数据不会使程序崩溃。

##### 我们应该使用哪种类型的数据库？

起初，人们可能会倾向于选择 NoSQL，因为这样我们就不需要担心构造我们的数据和连接查询。然而，重要的是要意识到，我们将不再以 SQL 形式编写这些查询，而是将所有数据检索到用户空间，即程序中，然后在程序中编写手动连接查询。

相反，如果我们依赖关系数据库，我们可以确保更小的存储空间，更高效的连接查询，以及具有定义良好模式的数据。所有关系数据库和一些 NoSQL 数据库都提供索引，这也有助于优化更快的搜索查询。然而，使用表和连接的关系数据库的一个主要缺点是，随着数据的增长，连接可能会变得更慢。到这个时候，您将清楚地知道您的数据的哪些部分可以利用 NoSQL 解决方案，并且您将开始在 SQL 和 NoSQL 系统的组合中维护您的数据。

简而言之，从关系数据库开始，一旦表中有大量数据且无法进行进一步的数据库调优，那么考虑将确实需要 NoSQL 数据存储的表移动过去。

#### 数据库复制

既然我们已经确定了为什么选择使用关系数据库，让我们转向下一个问题：我们如何确保我们的数据库不会成为单点故障？

让我们首先考虑如果数据库失败会有什么后果：

+   我们无法向数据库中写入新数据

+   我们无法从数据库中读取

在这两种后果中，后者更为关键。考虑我们的博客应用，虽然能够写新的博客文章很重要，但我们网站上绝大多数的用户将是读者。这是大多数日常用户界面应用的常态。因此，我们应该尽量确保我们总是能够从数据库中读取数据，即使我们不再能够向其中写入新数据。

数据库复制和冗余性试图解决这些问题，通常解决方案作为数据库或插件的一部分包含在其中。在本节中，我们将讨论用于数据库复制的三种策略：

+   主-副本复制

+   主-主复制

+   故障转移集群复制

##### 主-副本复制

这是最直接的复制方法。可以解释如下：

1.  我们采用数据库集群：

![](img/05818c20-497a-4808-b4cb-a9f2945a25cc.png)

数据库集群

1.  将其中一个指定为主数据库，其余数据库为副本：

![](img/b380eeef-f090-425c-a3d2-7ed6a47bc54c.png)

DB-3 被指定为主数据库

1.  所有写入都是在主数据库上执行的：

![](img/bd4944ec-aba0-413f-a338-a05e46b08a9b.png)

主数据库上执行三次写入

1.  所有读取都是从副本执行的：

>![](img/542a9156-003e-4c02-a8be-59d1ebd0b1de.png)

从副本执行的读取

1.  主数据库确保所有副本都具有最新状态，即主数据库的状态：

![](img/4b07728c-7236-4a5f-a59d-89839bd3fbb6.png)

主数据库将所有副本更新为最新更新

1.  主数据库故障仍允许从副本数据库读取，但不允许写入：

![](img/fd0bb311-32cb-47da-9bc6-ada8f6e5b8d9.png)

主数据库故障；只读取，不写入

##### 主-主复制

您可能已经注意到主-副本设置存在两个问题：

+   主数据库被广泛用于数据库写入，因此处于持续压力之下

+   副本解决了读取的问题，但写入的单点故障仍然存在

主-主复制尝试通过使每个数据库成为主数据库来解决这些问题。可以解释如下：

1.  我们采用数据库集群：

![](img/adc01142-21d5-48a6-8e21-7b4b31c39aab.png)

数据库集群

1.  我们将每个数据库指定为主数据库：

![](img/f97237e8-3f47-4085-8663-16ca50668ab0.png)

所有数据库都被指定为主数据库

1.  可以从任何主数据库执行读取：

![](img/04b29bf9-962f-48f5-86ed-d89509748c73.png)

在主数据库上执行读取

1.  可以在任何主数据库上执行写入：

![](img/5509f284-1103-4d8b-a789-3aea941a01f1.png)

写入 DB-1 和 DB-3

1.  每个主数据库都使用写入更新其他主数据库：

![](img/6dfbc383-b6d5-4444-9e53-e19d58552f2c.png)

数据库状态在主数据库之间同步

1.  因此，状态在所有数据库中保持一致：

![](img/c78429cb-281f-4406-8e5d-2197b54b4e98.png)

DB-1 故障，成功读取和写入

这种策略似乎运行良好，但它有自己的局限性和挑战；主要的问题是解决写入之间的冲突。这里有一个简单的例子。

我们有两个主-主数据库**DB-1**和**DB-2**，并且两者都具有数据库系统的最新状态：

![](img/ec564223-6449-4fd9-b4e3-ce86ad1af8bb.png)

DB-1 和 DB-2 的最新状态

我们有两个同时进行的写操作，因此我们将“Bob”发送到**DB-1**，将“Alice”发送到**DB-2***.*

![](img/39c2dd82-bfa7-4a4b-a150-22b179f055fb.png)

将“Bob”写入 DB-1，将“Alice”写入 DB-2

现在，两个数据库都已将数据写入其表，它们需要使用自己的最新状态更新另一个主数据库：

![](img/cda8ddb8-8758-407a-bf96-0e99ed0d29c0.png)

DB 同步之前的状态

这将导致冲突，因为在两个表中，**ID# 3**分别填充了**DB-1**的**Bob**和**DB-2**的**Alice**：

![](img/cd911985-d51b-40a7-be08-2cb8302b61b8.png)

在更新 DB-1 和 DB-2 状态时发生冲突，因为 ID# 3 已经被填充。

实际上，主-主策略将具有内置机制来处理这类问题，但它们可能会导致性能损失或其他挑战。这是一个复杂的主题，我们必须决定在使用主-主复制时值得做出哪些权衡。

##### 故障转移集群复制

主-副本复制允许我们在潜在风险的情况下对读取和写入进行简单设置，无法写入主数据库。主-主复制允许我们在其中一个主数据库故障时能够读取和写入数据库。然而，要在所有主数据库之间保持一致状态的复杂性和可能的性能损失可能意味着它并不是在所有情况下的理想选择。

故障转移集群复制试图采取中间立场，提供两种复制策略的功能。可以解释如下：

1.  我们采用数据库集群。

1.  根据使用的主选择策略，将数据库分配为主数据库，这可能因数据库而异。

1.  其余数据库被分配为副本。

1.  主服务器负责将副本更新为数据库的最新状态。

1.  如果主服务器因某种原因失败，将选择将剩余的数据库之一指定为新的主数据库。

那么我们应该使用哪种复制策略？最好从最简单的开始，也就是主-副本策略，因为这将非常轻松地满足大部分最初的需求。现在让我们看看如果我们使用主-副本策略进行数据库复制，我们的应用程序会是什么样子：

![](img/ffbc1129-0b4a-4b21-b1ec-d3ccb95e4879.png)

具有主-副本数据库设置的应用程序

## 单体架构与微服务

大多数新项目最初都是单一的代码库，所有组件通过直接函数调用相互交互。然而，随着用户流量和代码库的增加，我们将开始面临代码库的问题。以下是可能的原因：

+   您的代码库正在不断增长，这意味着任何新开发人员理解完整系统将需要更长的时间。

+   添加新功能将需要更长时间，因为我们必须确保更改不会破坏任何其他组件。

+   由于以下原因，为每个新功能重新部署代码可能会变得繁琐：

+   部署失败和/或

+   重新部署的组件出现了意外的错误，导致程序崩溃和/或

+   由于测试数量较多，构建过程可能需要更长时间

+   将完整应用程序扩展以支持 CPU 密集型组件

微服务通过将应用程序的主要组件拆分为单独的较小的应用程序/服务来解决这个问题。这是否意味着我们应该从一开始就将我们的应用程序拆分成微服务，以便我们不会面临这个问题？这是一种可能的处理方式。然而，这种方法也有一定的缺点：

+   **移动部件过多**：将每个组件分成自己的服务意味着我们必须监视和维护每个组件的服务器。

+   **增加的复杂性**：微服务增加了失败的可能原因。单体架构中的故障可能仅限于服务器宕机或代码执行问题。然而，对于微服务，我们必须：

+   识别哪个组件的服务器宕机或

+   如果一个组件失败，识别失败的组件，然后进一步调查失败是否是由于：

+   故障代码或

+   由于一个依赖组件的失败

+   整个系统更难调试：前面描述的增加的复杂性使得调试完整系统变得更加困难。

既然我们已经看到了微服务和单体架构的一些优缺点，哪一个更好呢？答案现在应该是相当明显的：

+   小到中等规模的代码库受益于单体架构提供的简单性

+   大型代码库受益于微服务架构提供的细粒度控制

这意味着我们应该设计我们的单体代码库，预期它最终可能会增长到非常庞大的规模，然后我们将不得不将其重构为微服务。为了尽可能轻松地将代码库重构为微服务，我们应该尽早确定可能的组件，并使用**中介者设计模式**实现它们与代码的其他部分之间的交互。

### 中介者设计模式

中介者充当代码中各个组件之间的中间人，这导致各个组件之间的耦合非常松散。这使我们可以对代码进行最小的更改，因为我们只需要更改中介者与被提取为自己的微服务的组件之间的交互。

让我们举个例子。我们有一个由 **Codebase A** 定义的单体应用。它由五个组件组成——**Component 1** 到 **Component 5**。我们意识到 **Component 1** 和 **Component 2** 依赖于与 **Component 5** 交互，而 **Component 2** 和 **Component 3** 依赖于 **Component 4**。如果 **Component 1** 和 **Component 2** 直接调用 **Component 5**，同样 **Component 2** 和 **Component 4** 直接调用 **Component 4**，那么我们将创建紧密耦合的组件。

如果我们引入一个函数，该函数从调用组件接收输入并调用必要的组件作为代理，并且所有数据都使用明确定义的结构传递，那么我们就引入了中介者设计模式。这可以在下图中看到：

![](img/805a38af-f4cb-437e-95c5-0aaad75e2466.png)

通过中介者连接的代码库中的组件

现在，如果出现需要将其中一个组件分离成自己独立的微服务的情况，我们只需要改变代理函数的实现。在我们的例子中，`Component 5` 被分离成了自己独立的微服务，并且我们已经改变了代理函数 **mediator 1** 的实现，以使用 HTTP 和 JSON 与 **Component 5** 进行通信，而不是通过函数调用和结构体进行通信。如下图所示：

![](img/3aeaa9a3-9b0c-4c7a-8831-29fc0e08ad24.png)

组件分离成微服务和中介者实现的更改

## 部署选项

我们已经研究了各种扩展应用程序的策略、不同类型的数据库、如何构建我们的代码，最后是如何使用中介者模式来实现从单体应用到微服务的过渡。然而，我们还没有讨论我们将在哪里部署所述的 Web 应用程序和数据库。让我们简要地看一下部署的情况。

直到 2000 年代初，大多数服务器都部署在由编写软件的公司拥有的硬件上。会有专门的基础设施和团队来处理这个软件工程的关键部分。这在很大程度上是数据中心的主题。

然而，在 2000 年代，公司开始意识到数据中心可以被抽象化，因为大多数开发人员对处理这些问题并不感兴趣。这使得软件的开发和部署变得更加便宜和快速，特别是对于 Web 应用程序。现在，开发人员不再购买数据中心的硬件和空间，而是可以通过 SSH 访问服务器实例。在这方面最著名的公司之一是亚马逊公司。这使他们的业务扩展到了电子商务之外。

这些服务也引发了一个问题：开发人员是否需要安装和维护诸如数据库、负载均衡器或其他类似服务的通用应用程序？事实是，并非所有开发人员或公司都希望参与维护这些服务。这导致了对现成应用实例的需求，这些实例将由销售这些应用作为服务的公司进行维护。

有许多最初作为软件公司开始并维护自己数据中心的公司——例如亚马逊、谷歌和微软等等——他们现在为一般消费者提供了一系列这样的服务。

### 多个实例的可维护性

提到的服务的可用性显著改善了我们的生活，但在维护跨多个服务器实例运行的大量应用程序时涉及了许多复杂性。例如：

+   如何更新服务器实例而不使整个服务停机？这可以用更少的工作量完成吗？

+   有没有一种可靠的方法可以轻松地扩展我们的应用程序（纵向和横向）？

考虑到所有现代部署都使用容器，我们可以利用容器编排软件来帮助解决可维护性问题。Kubernetes（[`kubernetes.io/`](https://kubernetes.io/)）和 Mesos（[`mesos.apache.org/`](http://mesos.apache.org/)）是两种解决方案的例子。

## 总结

在本章中，我们以一个简单的博客应用为例，展示了如何扩展以满足不断增长的用户流量的需求。我们还研究了扩展数据库涉及的复杂性和策略。

然后，我们简要介绍了如何设计我们的代码库以及我们可能需要考虑的权衡。最后，我们看了一种将代码库从单体架构轻松重构为微服务的方法。