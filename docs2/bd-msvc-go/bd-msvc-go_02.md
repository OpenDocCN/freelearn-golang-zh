# 设计一个优秀的 API

无论你是经验丰富的 API 和微服务构建者，正在寻找如何使用 Go 应用这些技术的技巧，还是你对微服务世界一无所知，花时间阅读这一章都是值得的。

编写 API 合约感觉像是艺术与科学的结合，当你与其他工程师讨论你的设计时，你很可能会同意不同意，不仅仅是关于制表符与空格的问题，但 API 合约确实有一些个人特色。

在本章中，我们将探讨两种最流行的选项，即 RESTful 和 RPC。我们将检查每种方法的语义，这将为你提供在不可避免讨论（即争论）发生时论证你观点的知识。选择 REST 或 RPC 可能完全取决于你当前的环境。如果你目前有运行实现 RESTful 方法的服务的，那么我建议你继续使用它，同样，如果你现在使用 RPC。我建议的一件事是，你应该阅读整个章节，以了解每种方法的语义、优点和缺点。

# RESTful API

术语**REST**是由 Roy Fielding 在 2000 年的博士论文中提出的。它代表**表征状态转移**，其描述如下：

"REST 强调组件交互的可扩展性、接口的通用性、组件的独立部署、以及中间组件以减少交互延迟、加强安全和封装遗留系统。"

具备符合 REST 原则的 API 才是 RESTful 的。

# URI

HTTP 协议中的一个主要组件是 URI。**URI**代表**统一资源标识符**，是你访问 API 的方法。你可能想知道 URI 和 URL（统一资源定位符）之间的区别是什么？当我开始写这一章时，我自己也对此感到困惑，并做了任何自重的开发者都会做的事情，即前往 Stack Overflow。不幸的是，我的困惑反而加深了，因为那里有很多详细的答案，但没有一个我认为特别有启发性。是时候前往地狱的内部圈层，也就是 W3C 标准，查找 RFC 以获取官方答案了。

简而言之，两者没有区别，URL 是 URI 的一种，通过其网络位置标识资源，在描述资源时可以互换使用术语。

2001 年发布的澄清文档([`www.w3.org/TR/uri-clarification`](http://www.w3.org/TR/uri-clarification))继续解释说，在 90 年代初，有一个假设认为标识符被归入一个或两个类别。标识符可能指定资源的位置（URL）或其名称（统一资源名称 URN），而不考虑位置。URI 可以是 URL 或 URN。使用这个例子，`http://`将是一个 URL 方案，`isbn:`是一个 URN 方案。然而，随着时间的推移，这一级层次结构的重要性降低了。观点发生了变化，即单个方案不需要被归入离散类型集合中的一个。

传统的方法是`http:`是一个 URI 方案，`urn:`也是一个 URI 方案。URN 的形式为`urn:isbn:n-nn-nnnnnn-n`，`isbn:`是一个 URN 命名空间标识符，而不是 URN 方案或 URI 方案。

依据这一观点，术语 URL 并不指代 URI 空间的正式划分，相反，URL 是一个非正式的概念；URL 是一种 URI 类型，通过其网络位置来标识资源。

在本书的其余部分，我们将使用术语 URI，当我们这样做时，我们将讨论一种访问远程服务器上运行的资源的方法。

# URI 格式

2005 年发布的 RFC 3986[`www.ietf.org/rfc/rfc3986.txt`](https://www.ietf.org/rfc/rfc3986.txt)定义了使 URI 有效的格式：

```go
URI = scheme "://" authority "/" path [ "?" query] ["#" fragment"] 
URI = http://myserver.com/mypath?query=1#document 

```

我们将使用路径元素来定位我们服务器上运行的端点。在 REST 端点中，这可以包含参数以及文档位置。查询字符串同样重要，因为你将使用它来传递参数，如页码或排序，以控制返回的数据。

URI 格式化的一些通用规则：

+   正斜杠`/`用于表示资源之间的层次关系

+   URI 中不应包含尾随正斜杠`/`

+   使用连字符`-`可以提高可读性

+   在 URI 中不应使用下划线`_`

+   优先使用小写字母，因为大小写敏感性是 URI 路径部分的一个区分因素

许多规则背后的概念是 URI 应该易于阅读和构建。它也应该在构建方式上保持一致，因此你应该为 API 中的所有端点遵循相同的分类法。

# REST 服务的 URI 路径设计

路径被分解为文档、集合、存储和控制器。

# 集合

集合是资源的目录，通常通过参数来访问单个文档。例如：

```go
GET /cats   -> All cats in the collection 
GET /cats/1 -> Single document for a cat 1 

```

当定义一个集合时，我们应该始终使用复数名词，例如`cats`或`people`作为集合名称。

# 文档

文档是指向单个对象的资源，类似于数据库中的一行。它具有拥有子资源的能力，这些子资源可以是子文档或集合。例如：

```go
GET /cats/1           -> Single document for cat 1 
GET /cats/1/kittens   -> All kittens belonging to cat 1 
GET /cats/1/kittens/1 -> Kitten 1 for cat 1 

```

# 控制器

控制器资源就像一个过程，这通常用于资源无法映射到标准的 **CRUD**（**创建**、**检索**、**更新** 和 **删除**）函数时。

控制器的名称出现在 URI 路径的最后一段，没有子资源。如果控制器需要参数，这些参数通常包含在查询字符串中：

```go
POST /cats/1/feed           -> Feed cat 1 
POST /cats/1/feed?food=fish ->Feed cat 1 a fish 

```

定义控制器名称时，我们应该始终使用动词。动词是一个表示动作或状态的词，例如 `feed` 或 `send`。

# 存储

存储是一个客户端管理的资源仓库，它允许客户端添加、检索和删除资源。与集合不同，存储永远不会生成新的 URI，它将使用客户端指定的 URI。以下是一个示例，它将向我们的存储中添加一只新猫：

```go
PUT /cats/2 

```

这将在存储中添加一只 ID 为 `2` 的新猫，如果我们向集合中发布了没有 ID 的新猫，则响应需要包含对新定义的文档的引用，这样我们就可以稍后与之交互。像控制器一样，我们应该使用复数名词作为存储名称。

# CRUD 函数名称

在设计优秀的 REST URI 时，我们从不使用 CRUD 函数名称作为 URI 的一部分，而是使用 HTTP 动词。例如：

```go
DELETE /cats/1234

```

我们不在方法名称中包含动词，因为这由 HTTP 动词指定，以下 URI 被认为是反模式：

```go
GET /deleteCat/1234 
DELETE /deleteCat/1234 
POST /cats/1234/delete

```

在下一节中查看 HTTP 动词时，这将会更清楚。

# HTTP 动词

常用的 HTTP 动词有：

+   `GET`

+   `POST`

+   `PUT`

+   `PATCH`

+   `DELETE`

+   `HEAD`

+   `OPTIONS`

这些方法中的每一个都在我们的 REST API 的上下文中有一个明确的语义，正确的实现将帮助用户理解你的意图。

# GET

`GET` 方法用于检索资源，不应用于更改操作，例如更新记录。通常，`GET` 请求不会传递正文；然而，这样做并不构成无效的 `HTTP` 请求。

**请求**:

```go
GET /v1/cats HTTP/1.1 

```

**响应**:

```go
HTTP/1.1 200 OK 
Content-Type: application/json 
Content-Length: xxxx 

{"name": "Fat Freddie's Cat", "weight": 15} 

```

# POST

`POST` 方法用于在集合中创建一个新的资源或执行一个控制器。它通常是一个非幂等操作，这意味着多次向集合中创建一个元素将创建多个元素，而这些元素在第一次调用后不会被更新。

`POST` 方法在调用控制器时始终使用，因为其操作被认为是非幂等的。

**请求**:

```go
POST /v1/cats HTTP/1.1 
Content-Type: application/json 
Content-Length: xxxx 

{"name": "Felix", "weight": 5} 

```

**响应**:

```go
HTTP/1.1 201 Created 
Content-Type: application/json 
Content-Length: 0 
Location: /v1/cats/12343 

```

# PUT

`PUT` 方法用于更新可变资源，并且必须始终包含资源定位符。`PUT` 方法的调用也是幂等的，这意味着多次请求不会将资源状态更改为与第一次调用不同的状态。

**请求**:

```go
PUT /v1/cats HTTP/1.1 
Content-Type: application/json 
Content-Length: xxxx 

{"name": "Thomas", "weight": 7 } 

```

**响应**:

```go
HTTP/1.1 201 Created 
Content-Type: application/json 
Content-Length: 0 

```

# PATCH

`PATCH` 动词用于执行部分更新，例如，如果我们只想更新我们猫的名字，我们可以发出只包含我们想要更改的详细信息的 `PATCH` 请求。

**请求**:

```go
PATCH /v1/cats/12343 HTTP/1.1 
Content-Type: application/json 
Content-Length: xxxx 

{"weight": 9} 

```

**响应**:

```go
HTTP/1.1 204 No Body 
Content-Type: application/json 
Content-Length: 0 

```

在我的经验中，PATCH 更新很少被使用，通常的做法是使用 PUT 来更新整个对象，这不仅使得代码更容易编写，而且使得 API 更易于理解。

# 删除

当我们想要删除一个资源时，使用 `DELETE` 动词，通常我们会将资源的 ID 作为路径的一部分传递，而不是在请求体中。这样，我们就有了一个一致的方法来更新、删除和检索文档。

**请求**：

```go
DELETE /v1/cats/12343 HTTP/1.1 
Content-Type: application/json 
Content-Length: 0 

```

**响应**：

```go
HTTP/1.1 204 No Body 
Content-Type: application/json 
Content-Length: 0 

```

# 头部

当客户端想要检索资源的头部信息而不需要正文时，会使用 `HEAD` 动词。`HEAD` 动词通常用于替代 `GET` 动词，当客户端只想检查资源是否存在或读取元数据时。

**请求**：

```go
HEAD /v1/cats/12343 HTTP/1.1 
Content-Type: application/json 
Content-Length: 0 

```

**响应**：

```go
HTTP/1.1 200 OK 
Content-Type: application/json 
Last-Modified: Wed, 25 Feb 2004 22:37:23 GMT 
Content-Length: 45 

```

# 选项

当客户端想要检索资源的可能交互时，会使用 `OPTIONS` 动词。通常，服务器会返回一个 `Allow` 头部，其中包含可以与该资源一起使用的 `HTTP` 动词。

**请求**：

```go
OPTIONS /v1/cats/12343 HTTP/1.1 
Content-Length: 0 

```

**响应**：

```go
HTTP/1.1 200 OK 
Content-Length: 0 
Allow: GET, PUT, DELETE

```

# URI 查询设计

在 API 调用中使用查询字符串作为一部分是完全可接受的；然而，我建议不要用它来传递数据给服务。相反，查询应该用于执行如下操作：

+   分页

+   过滤

+   排序

如果我们需要调用一个控制器，我们之前讨论过，应该使用 `POST` 请求，因为这很可能是非幂等请求。为了传递数据给服务，我们应该在体中包含数据。然而，我们可以使用查询字符串来过滤控制器的动作：

```go
POST /sendStatusUpdateEmail?$group=admin 
{ 
  ""message": "": "All services are now operational\nPlease accept our 
              apologies for any inconvenience caused.\n 
              The Kitten API team"" 
} 

```

在前面的例子中，我们会发送包含在请求体中的状态更新电子邮件，因为我们使用了查询字符串中传递的组过滤器，我们可以将这个控制器的动作限制为只发送给管理员组。

如果我们将消息添加到查询字符串而没有传递消息体，那么我们可能会给自己造成两个问题。第一个问题是 URI 的最大长度为 2083 个字符。第二个问题是，通常一个 `POST` 请求总是会包括请求体。虽然这不是 `HTTP` 规范的要求，但这是大多数用户期望的行为。

# 响应代码

当编写一个优秀的 API 时，我们应该使用 `HTTP` 状态码来向客户端指示请求的成功或失败。在本章中，我们不会全面查看所有可用的状态码；互联网上有许多资源提供了这些信息。我们将提供一些进一步阅读的资源，我们将要做的是查看作为软件工程师，你希望你的微服务返回的状态码。

目前，普遍认为这是一种好的做法，因为它允许客户端立即确定请求的状态，而无需深入请求体以获得更多信息。在失败的情况下，如果 API 总是向用户返回包含进一步信息的`200 OK`响应，这不是一个好的做法，因为它要求客户端检查体以确定结果。这也意味着消息体将包含除了它应该表示的对象之外的其他信息。考虑以下不良做法：

错误请求体：

```go
POST /kittens 
RESPONSE HTTP 200 OK 
{ 
  ""status":": 401, 
  ""statusMessage": "": "Bad Request"" 
} 

```

成功的请求：

```go
POST /kittens 
RESPONSE HTTP 201 CREATED 
{ 
  ""status":": 201, 
  ""statusMessage": "": "Created",", 
  ""kitten":": { 
    ""id": "": "1234334dffdf23",", 
    ""name": "": "Fat Freddy'sFreddy's Cat"" 
  } 
} 

```

假设你正在编写一个针对先前请求的客户端，你需要在你能够读取和处理返回的小猫之前，在你的应用程序中添加逻辑来检查响应中的状态节点。

现在考虑一些更糟糕的情况：

更糟糕的失败例子：

```go
POST /kittens 
RESPONSE HTTP 200 OK 
{ 
  ""status":": 400, 
  ""statusMessage": "": "Bad Request"" 
} 

```

更糟糕的成功例子：

```go
POST /kittens 
RESPONSE HTTP 200 OK 
{ 
  ""id": "": "123434jhjh3433",", 
  ""name": "": "Fat Freddy'sFreddy's Cat"" 
} 

```

如果你的 API 作者做了像前面例子中的事情，你需要检查返回的响应是错误还是你期望的小猫。在编写这个 API 客户端的过程中，你每分钟会说出多少个 WTF，这不会让你对它的作者产生好感。这些可能看起来像是极端的例子，但野外确实有类似的情况，在我的职业生涯中，我相当确信我犯过这样的错误，但那时我没有读过这本书。

作者在最好的意图下所做的尝试是过于字面地理解 HTTP 状态码。W3C RFC2616 指出，HTTP 状态码与尝试理解和满足请求有关([`www.w3.org/Protocols/rfc2616/rfc2616-sec6.html#sec6.1.1`](https://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html#sec6.1.1))；然而，当你查看一些单独的状态码时，这有点模糊。现代共识是，使用 HTTP 状态码来指示 API 请求的处理状态，而不仅仅是服务器的处理能力是可以接受的。考虑一下，如果我们通过实施这种方法，我们如何使这些请求变得更好。

一个失败的例子：

```go
POST /kittens 
RESPONSE HTTP 400 BAD REQUEST 
{ 
  ""errorMessage": "": "Name should be between 1 and 256 characters in 
  length and only contain [A-Z] - ['-.]"'-.]" 
} 

```

一个成功的例子：

```go
POST /kittens 
RESPONSE HTTP 201 CREATED 
{ 
  ""id": "": "123hjhjh2322",", 
  ""name": "": "Fat Freddy'sFreddy's cat"" 
} 

```

这更加语义化；在失败的情况下，如果用户需要更多信息，他们只需要阅读响应。除此之外，我们还可以提供一个标准错误对象，该对象用于我们 API 的所有端点，它提供了进一步但非必需的信息，以确定请求失败的原因。我们稍后会看看错误对象，但现在让我们更深入地看看 HTTP 状态码。

# 2xx 成功

2xx 状态码表示客户端的请求已被成功接收并理解。

# 200 OK

这是一个通用的响应码，表示请求已成功。伴随此代码的响应通常是：

+   `GET`：一个对应于请求资源的实体

+   `HEAD`：请求的资源对应的头字段，没有消息体

+   `POST`：一个描述或包含操作结果的实体

# 201 已创建

当请求成功且结果是一个新实体被创建时，会发送创建的响应。除了响应之外，API 通常还会返回一个`Location`头，其中包含新创建实体的位置：

```go
201 Created 
Location: https://api.kittens.com/v1/kittens/123dfdf111 

```

返回对象体是否可选取决于此响应类型。

# 204 无内容

此状态通知客户端请求已成功处理；然而，响应中不会有消息体。例如，如果用户对集合发出`DELETE`请求，则响应可能返回 204 状态。

# 3xx 重定向

3xx 状态码类表示客户端必须采取额外操作以完成请求。许多这些状态码由 CDN 和其他内容重定向技术使用，然而，代码 304 在为我们的 API 设计提供语义反馈给客户端时非常有用。

# 301 永久移动

这告诉客户端他们请求的资源已被永久移动到不同的位置。虽然这传统上用于将页面或资源从 Web 服务器重定向，但它在我们构建 API 时也可能很有用。如果我们重命名一个集合，我们可以使用 301 重定向将客户端发送到正确的位置。然而，这应该被视为例外而不是常规做法。一些客户端不会隐式遵循 301 重定向，实现此功能会增加消费者额外的复杂性。

# 304 未修改

此响应通常由 CDN 或缓存服务器使用，并设置为指示自上次调用 API 以来响应未修改。这是为了节省带宽，请求将不会返回一个体，但将返回`Content-Location`和`Expires`头。

# 4xx 客户端错误

如果错误是由客户端而非服务器引起的，服务器将返回 4xx 响应，并且总是返回一个实体，其中包含有关错误的更多详细信息。

# 400 错误请求

此响应表示客户端由于请求格式错误或域验证失败（数据缺失或会导致无效状态的操作）而无法理解请求。

# 401 未授权

这表示请求需要用户身份验证，并将包含一个包含适用于请求资源的挑战的`WWW-Authenticate`头。如果用户已在`WWW-Authenticate`头中包含了所需的凭据，则响应应包含一个可能包含相关诊断信息的错误对象。

# 403 禁止

服务器已理解请求，但拒绝执行。这可能是因为对资源的访问级别不正确，而不是用户未认证。

如果服务器不希望公开请求无法访问资源的事实，那么可以返回`404 Not found`状态码而不是此响应。

# 404 未找到

此响应表示服务器未找到与请求 URI 匹配的内容。没有给出关于条件是暂时性还是永久性的指示。

客户端可以多次向此端点发送请求，因为状态可能不是永久的。

# 405 方法不允许

请求中指定的方法不允许对 URI 指示的资源进行操作。这可能是当客户端尝试通过向仅提供文档检索功能的集合发送`POST`、`PUT`或`PATCH`来修改集合时。

# 408 请求超时

客户端没有在服务器准备等待的时间内产生请求。客户端可以在稍后时间重复请求，无需修改。

# 5xx 服务器错误

范围在 500 之间的响应状态码表示服务器发生了“Bang”，服务器知道这一点，并为这种情况感到抱歉。

RFC 建议在响应中返回一个错误实体，说明这是永久性的还是暂时性的，并包含错误解释。当我们查看关于安全性的章节时，我们会看到关于在错误信息中不要透露太多信息的建议，因为这种状态可能是用户试图破坏您的系统而人为制造的。通过返回诸如堆栈跟踪或其他内部信息之类的 5xx 错误，实际上可能会帮助破坏您的系统。因此，目前通常的做法是，500 错误只会返回一个非常通用的信息。

# 500 内部服务器错误

一个通用的错误信息，表明事情并没有按计划进行。

# 503 服务不可用

服务器当前因临时过载或维护而不可用。在微服务发生故障或过载的情况下，有一个非常有用的模式可以实现，该模式可以避免级联故障。在这种情况下，微服务将监控其内部状态，在发生故障或过载时，将拒绝接受请求，并立即向客户端发出信号。我们将在第 xx 章中更详细地探讨这个模式；然而，这个实例可能是您想要返回 503 状态码的地方。这也可以作为您的健康检查的一部分使用。

# HTTP 头

请求头是 HTTP 请求和响应过程中的一个非常重要的部分，实施标准方法有助于您的用户从一个 API 过渡到另一个 API。在本节中，我们不会涵盖您可以在 API 中使用的所有可能的头，但我们将查看最常见的头，以获取有关 HTTP 协议的完整信息，请参阅 RFC 7231 [`tools.ietf.org/html/rfc7231`](https://tools.ietf.org/html/rfc7231)。该文档包含了对当前标准的全面概述。

# 标准请求头

请求头为 API 的请求和响应提供额外的信息。把它们想象成操作的元数据。它们可以用来增强响应中的其他数据，这些数据本身不属于主体，例如内容编码。它们也可以被客户端利用，提供有助于服务器处理响应的信息。在可能的情况下，我们应该始终使用标准头，因为这为你的用户提供了一致性，并为他们提供了多个端点从多个不同供应商的通用标准。

# Authorization - 字符串

授权是使用最广泛的请求头之一，即使你有公共只读 API，我也建议你要求用户授权他们的请求。通过要求用户授权请求，你就有能力执行用户级日志记录和速率限制等操作。通常，你可能会看到使用自定义请求头（如“X-API-Authorization”）进行授权。我建议你不要使用这种方法，因为 W3C RFC 2616 中指定的标准授权头（[`www.w3.org/Protocols/rfc2616/rfc2616-sec14.html`](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)）具有我们所需的所有功能。许多公司，如 Twitter 和 PayPal，使用此头进行请求认证。让我们看看 Twitter 开发者文档中的一个简单示例，看看这是如何实现的：

```go
Authorization:  
        OAuth oauth_consumer_key="="xvz1evFS4wEEPTGEFPHBog",",  
              oauth_nonce="="kYjzVBB8Y0ZFabxSWbWovY3uYSQ2pTgmZeNu2VS4cg",",  
              oauth_signature="="tnnArxj06cWHq44gCs1OSKk%2FjLY%3D",",  
              oauth_signature_method="="HMAC-SHA1",",  
              oauth_timestamp="="1318622958",",  
              oauth_token="="370773112-GmHxMAgYyLbNEtIKZeRNFsMKPR9EyMZeS9weJAEb",",  
              oauth_version="="1.0"" 

```

头部格式为`[Authorization method] [逗号分隔的 URL 编码值]`。这清楚地告知服务器授权类型是 OAuth，并且此授权的各个组件以逗号分隔的格式跟随。通过遵循这种标准方法，你可以让你的消费者使用实现此标准的第三方库，从而节省他们构建定制实现的工作。

# 日期

请求的 RFC 3339 格式的时间戳。

# Accept - 内容类型

响应请求的内容类型，例如：

+   `application/xml`

+   `text/xml`

+   `application/json`

+   `text/javascript`（用于 JSONP）

# Accept-Encoding - gzip, deflate

当适用时，REST 端点应始终支持 gzip 和 deflate 编码。

在 Go 中实现 gzip 支持相对简单；我们在第一章，“微服务简介”中展示了如何将中间件实现到你的微服务中。在下面的示例中，我们将使用这项技术来创建 gzip 响应 writer。

在 gzip 格式中编写响应的核心是`compress/gzip`包，它是标准库的一部分。它允许你创建一个实现`ioWriteCloser`接口的`Writer`，该接口包装现有的`io.Writer`，使用 gzip 压缩将数据写入给定的 writer：

```go
func NewWriter(w io.Writer) *Writer 

```

要创建我们的处理程序，我们将编写`NewGzipHandler`函数，这个函数返回一个新的`http.Handler`，它将包装我们的标准输出处理程序。

我们首先需要做的是创建自己的`ResponseWriter`，它嵌入`http.ResponseWriter`。

示例 2.1 `chapter2/gzip/gzip_deflate.go`：

```go
68 type GzipResponseWriter struct { 
69   gw *gzip.Writer 
70   http.ResponseWriter 
71} 

```

核心方法是实现`Write`方法：

```go
73 func (w GzipResponseWriter) Write(b []byte) (int, error) { 
74   if _, ok := w.Header()["()["Content-Type"];"]; !ok { 
75     // If content type is not set, infer it from the uncompressed body. 
76   w.Header().Set("("Content-Type",", http.DetectContentType(b)) 
77   } 
78   return w.gw.Write(b) 
79 } 

```

如果你查看标准`http.Response`结构体中`Write`方法的实现，那里有很多事情在进行，我们既不想丢失也不想重新实现，因为当我们对`gzip.Writer`对象调用`Write`时，它会反过来调用`http.Response`的`write`方法，我们不会丢失任何复杂性。

在我们的`NewGzipHandler`内部，我们的处理程序会检查客户端是否发送了`Accept-Encoding`头，如果是的话，我们将使用`GzipResponseWriter`方法来写入响应；如果客户端请求未压缩的内容，我们则只调用带有标准`ResponseWriter`的`ServeHttp`：

```go
40 type GZipHandler struct { 
41  next http.Handler 
42 } 
43 
44 func (h *GZipHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) { 
45  encodings := r.Header.Get("("Accept-Encoding")") 
46 
47  if strings.Contains(encodings, ""gzip")") { 
48      h.serveGzipped(w, r) 
49  } else if strings.Contains(encodings, ""deflate")") { 
50      panic("("Deflate not implemented")") 
51  } else { 
52      h.servePlain(w, r) 
53  } 
54 } 
55 
56 func (h *GZipHandler) serveGzipped(w http.ResponseWriter, r *http.Request) { 
57  gzw := gzip.NewWriter(w) 
58  defer gzw.Close() 
59 
60  w.Header().Set("("Content-Encoding", "", "gzip")") 
61  h.next.ServeHTTP(GzipResponseWriter{gzw, w}, r) 
62 } 

63 func (h *GZipHandler) servePlain(w http.ResponseWriter, r *http.Request) 64 { 
65  h.next.ServeHTTP(w, r) 
66 } 

```

这绝对不是一个全面的示例，而且有许多开源包，例如来自《纽约时报》团队的那个（[`github.com/NYTimes/gziphandler`](https://github.com/NYTimes/gziphandler)），它为你管理这些。

作为一个小型的编程测试，为什么不尝试修改这个示例来实现`DEFLATE`。

# 标准响应头

所有服务都应该返回以下头信息。

+   `Date`: 请求被处理的日期，格式为 RFC 3339。

+   `Content-Type`: 响应的内容类型。

+   `Content-Encoding`: gzip 或 deflate。

+   `X-Request-ID`/`X-Correlation-ID`：虽然你可能不会直接要求你的客户端实现这个头，但它可能是你在调用下游服务时添加到请求中的内容。当你试图调试在生产环境中运行的服务时，能够根据单个事务 ID 分组所有请求可以非常有用。当我们查看日志和监控时，我们会看到的一种常见做法是将所有日志存储在公共数据库中，例如 Elastic Search。通过在构建许多相互连接的微服务时设置标准的工作方式，它们在每个下游调用中传递关联 ID，你将能够使用 Kibana 或其他日志查询工具查询你的日志并将它们分组到单个事务中：

```go
        X-Request-ID: f058ebd6-02f7-4d3f-942e-904344e8cde 

```

# 返回错误

在失败的情况下，你的 API 用户应该能够编写一段代码来处理不同端点的错误。一个标准的错误实体将帮助你的消费者，使他们能够在客户端或服务器发生错误时编写 DRY（Don't Repeat Yourself）代码。

微软 API 指南推荐以下格式来处理这些实体：

```go
{ 
  ""error":": { 
    ""code": "": "BadArgument",", 
    ""message": "": "Previous passwords may not be reused",", 
    ""target": "": "password",", 
    ""innererror":": a { 
      ""code": "": "PasswordError",", 
  ""innererror":": { 
    ""code": "": "PasswordDoesNotMeetPolicy",", 
    ""minLength": "": "6",", 
    ""maxLength": "": "64",", 
    ""characterTypes": ["": ["lowerCase","","upperCase","","number","","symbol"],"], 
    ""minDistinctCharacterTypes": "": "2",", 
    ""innererror":": { 
      ""code": "": "PasswordReuseNotAllowed"" 
    } 
      } 
    } 
  } 
} 

```

**错误响应**：**对象**

`ErrorResponse`是返回响应的最高级对象，它包含以下字段：

| **属性** | **类型** | **必需** | **描述** |
| --- | --- | --- | --- |
| 错误 | 错误 | ![](img/e2f574a5-74e7-43c1-a797-49375f0cbcc0.png) | 错误对象。 |

**错误：对象**

`Error`对象是我们错误响应的详情；它提供了错误发生原因的完整详情：

| **属性** | **类型** | **必需** | **描述** |
| --- | --- | --- | --- |
| Code | `String` (枚举) | ![图片](img/9b35e8ba-feaa-4967-8bec-ebf5269f3b15.png) | 服务器定义的错误代码集合中的一个。 |
| message | String | ![图片](img/9b35e8ba-feaa-4967-8bec-ebf5269f3b15.png) | 错误的可读表示形式。 |
| Target | String | - | 错误的目标。 |
| Details | Error[] | - | 导致此报告错误的具体错误详情数组。 |
| innererror | InnerError | - | 包含比当前对象更具体错误信息的对象。 |

**InnerError**：**对象**

| **属性** | **类型** | **必需** | **描述** |
| --- | --- | --- | --- |
| Code | String | - | 比包含错误提供的错误代码更具体的错误代码。 |
| innererror | InnerError | - | 包含比当前对象更具体错误信息的对象。 |

微软提供了一个优秀的 API 指南资源，你可以通过以下链接了解更多关于返回错误的信息：

[`github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#51-errors`](https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#51-errors)

# 从 JavaScript 访问 API

网络浏览器实现了一个沙盒机制，该机制限制了一个域中的资源访问另一个域中的资源。例如，你可能有一个允许修改和检索用户数据的 API，以及一个提供此 API 接口的网站。如果浏览器没有实现“同源策略”，并且假设用户没有注销他们的会话，那么恶意页面就可以向 API 发送请求并修改它，而你却不知道。

为了解决这个问题，你的微服务可以实现两种方法来允许这种访问，**JSONP**（代表**带有填充的 JSON**）和**CORS**（**跨源资源共享**）。

# JSONP

JSONP 基本上是一个漏洞，并且大多数没有实现后续 CORS 标准的浏览器都实现了它。它仅限于`GET`请求，并且通过绕过问题来工作，即虽然`XMLHTTPRequest`被阻止向第三方服务器发出请求，但 HTML 脚本元素没有限制。

JSONP 请求将一个`<script src="img/...">`元素插入到浏览器的 DOM 中，API 的 URI 作为`src`目标。此组件返回一个带有 JSON 数据的函数调用，当它加载时，该函数执行并将数据传递给回调。

JavaScript 回调在代码中定义：

```go
function success(data) { 
  alert(data.message); 
} 

```

这是 API 调用的响应：

```go
success({"({"message":"":"Hello World"})"}) 

```

要表示请求返回 JSONP 格式的数据，通常会在 URI 中添加`callback=functionName`参数，在我们的例子中，这将是在`/helloworld?callback=success`。实现这一点特别简单，让我们看看我们的简单 Go `helloworld`示例，看看我们如何修改它以实现 JSONP。

有一个需要注意的事项是我们返回的`Content-Type`标题。我们不再返回`application/json`，因为我们不是返回 JSON，实际上我们在返回 JavaScript，因此我们必须相应地设置`Content-Type`标题：

```go
Content-Type: application/javascript 

```

示例 `chapter2/jsonp/jsonp.go`：

让我们快速看一下如何使用 Go 发送 JSONP 的示例，我们的响应对象将完全与第一章，*微服务简介*中的那些相同：

```go
18 type helloWorldResponse struct { 
19  Message string `json:":"message"`"` 
20 } 

```

差异全在于处理器，如果我们查看第**30**行，我们正在检查查询字符串中是否有回调参数。这将由客户端提供，并指示当响应返回时他们期望被调用的函数：

```go
23 func helloWorldHandler(w http.ResponseWriter, r *http.Request) { 
24  response := helloWorldResponse{Message: ""HelloWorld"}"} 
25  data, err := json.Marshal(response) 
26  if err != nil { 
27    panic("("Ooops")") 
28 } 
29 
30  callback := r.URL.Query().Get("("callback")") 
31  if callback != """" { 
32    r.Headers().Add("("Content-Type", "", "application/javascript")") 
33    fmt.Fprintf(w, "%"%s(%s)",)", callback, string(data)) 
34  } else { 
35    fmt.Fprint(w, string(data)) 
36  } 
37 } 

```

要以 JSONP 格式返回我们的响应，我们只需要将标准响应包装成 JavaScript 函数调用。在第**33**行，我们正在获取客户端传递的回调函数名称，并将我们通常要发送的响应封装起来。结果输出将类似于以下这样：

**请求**：

```go
GET /helloworld?callback=hello 

```

**响应**：

```go
hello({"message":"Hello World"})  

```

# CORS

假设您的用户正在使用过去五年内发布的桌面浏览器，或者 iOS 9 或 Android 4.2+等移动浏览器，那么实现 CORS 将绰绰有余。[`caniuse.com/#feat=cors`](http://caniuse.com/#feat=cors) 表示这超过了所有互联网用户的 92%。我期待着批评 IE 因未完全采用而受到的缺乏；然而，由于这自 IE8 以来就已经得到支持，我不得不抱怨移动用户。

CORS 是 W3C 的一个提案，旨在标准化浏览器中的跨源请求。它是通过浏览器内置的`HTTP`客户端在真实请求之前向 URI 发送一个`OPTIONS`请求来工作的。

如果另一端的服务器返回一个包含从该脚本加载的域的源头的标题，那么浏览器将信任该服务器，并允许进行跨站请求：

```go
Access-Control-Allow-Origin: origin.com 

```

在 Go 中实现这一点相当简单，我们可以创建一个中间件来全局管理这一点。为了简单起见，在我们的例子中，我们将其硬编码到处理器中：

示例 2.2 `chapter2/cors/cors.go`

```go
25 if r.Method == ""OPTIONS"" { 
26  w.Header().Add("("Access-Control-Allow-Origin", "*")", "*") 
27  w.Header().Add("("Access-Control-Allow-Methods", "", "GET")") 
28  w.WriteHeader(http.StatusNoContent) 
29  return 
30 }  

```

在**25**行，我们检测到请求方法是`OPTIONS`，而不是返回响应，我们返回客户端期望的`Access-Control-Allow-Origin`报头。在我们的示例中，我们简单地返回`\*`，这意味着所有域名都可以与此 API 交互。这不是最安全的实现，而且通常你会要求你的 API 用户注册将与 API 交互的域名，并将`Allow-Origin`限制只包括那些域名。除了`Allow-Origin`报头外，我们还返回以下内容：

```go
Access-Control-Allow-Methods: GET 

```

这告诉浏览器它只能对此 URI 发起`GET`请求，并且禁止发起`POST`、`PUT`等请求。这是一个可选的报头，但可以在与 API 交互时增强用户的安全性。需要注意的是，我们不是发送回`200 OK`响应，而是使用`204 No Content`，因为在`OPTIONS`请求中返回正文是不合法的。

# RPC API

RPC 代表远程过程调用；它是在远程机器上执行函数或方法的一种方法。RPC 自诞生以来一直存在，并且有众多不同类型的 RPC 技术，其中一些依赖于存在接口定义（如 SOAP、Thrift 协议缓冲区）。这种接口定义可以使得为不同的技术栈生成客户端和服务器存根变得更加容易。通常，接口是用**领域特定语言（DSL**）定义的，生成器程序将使用它来创建应用程序客户端和服务器。

与 REST 需要使用 HTTP 作为传输层不同，RPC 不受此限制，虽然可以将 RPC 调用发送到 HTTP，但如果选择，你也可以使用 TCP 或甚至 UDP 套接字的轻量级。

最近，RPC 的使用有所回升，许多由 Uber、Google、Netflix 等公司构建的大规模系统都在使用 RPC。这得益于不使用 HTTP 所带来的低延迟速度和性能，以及通过实现二进制消息格式而不是 JSON 或 XML 所获得的更小的消息大小。

RPC 的批评者提到，客户端和服务器之间可能会出现紧密耦合，即如果你更新了服务器上的合约，那么所有客户端也需要更新。在许多现代 RPC 实现中，这个问题已经不那么严重了，实际上，与 RESTful API 相比，这个问题并不更严重。虽然像 JMI 这样的旧技术紧密耦合，要求客户端和服务器共享相同的接口，但现代实现如 Protocol Buffers 合理地封装了对象，即使存在细微的差异也不会抛出错误。因此，通过遵循*版本化 API*部分的标准指南，你遇到的问题并不比实现 RESTful API 时更严重。

RPC 的一个好处是您可以快速为您的用户生成客户端，这允许从传输和消息类型中抽象出来，并使他们依赖于接口。作为创建者，您可以更改应用程序的底层实现，例如从 Thrift 迁移到 Proto buffers，而无需要求客户端做任何事情，只需使用您提供的最新版本的客户端。版本化还允许您保留与 REST 相同的前向兼容性。

# RPC API 设计

我们刚才讨论的创建良好 RESTful API 的一些原则也可以应用于 RPC。然而，主要区别之一是您可能不会使用 HTTP 作为传输协议；因此，您并不总是能够使用 HTTP 状态码作为成功或失败的指示。**RPC**代表**远程过程调用**，其历史可以追溯到互联网出现之前。最初，它被构想为执行可以在同一台机器上运行的独立应用程序中的过程，甚至可能在网络上。虽然我们现在认为这是理所当然的，但在 20 世纪 90 年代，这可是前沿技术。不幸的是，像 CORBA 和 Java RMI 这样的框架给 RPC 带来了坏名声，即使现在，如果您与 RPC 的反对者交谈，他们很可能会提到这两个框架。然而，好处是性能，使用二进制序列化在网络上非常高效，我们不再有 RMI 和 CORBA 强制执行的紧密耦合。我们也不再试图做任何太聪明的事情；我们不再尝试在两个进程之间共享对象，我们采取了一种更功能性的方法，即返回不可变对象的方法。这让我们拥有了两者之优；交互的简单性和二进制消息的速度与负载小。

# RPC 消息框架

这些天我们不再需要在客户端和服务器上使用相同的接口实现，这不符合我们独立可版本化和可部署的口号。幸运的是，框架更加灵活，我们可以采取与 REST 相同的方法，添加元素是可以的，但是删除元素或更改方法签名必须触发版本更新。

# Gob

我们已经在上一章中讨论了 gob，但作为一个快速回顾，gob 格式是专门为促进 Go 到 Go 通信而设计的，并且围绕着一个更容易使用且可能比类似协议缓冲区更高效的想法构建，但这牺牲了跨语言通信。

**gob 对象定义：**

```go
type HelloWorldRequest struct {
  Name string
}

```

关于 gob 的更多信息可以在 Go 文档中找到，网址为[`golang.org/pkg/encoding/gob/`](https://golang.org/pkg/encoding/gob/)

# Thrift

Facebook 创建了 Thrift 框架，并于 2007 年开源。目前由 Apache 软件基金会维护。Thrift 的主要目标是：

+   **简单性**：Thrift 代码简单易懂，没有不必要的依赖

+   **透明性**：Thrift 符合所有语言中最常见的习惯用法

+   **一致性**：特定于语言的特性属于扩展，而不是核心库

+   **性能**：首先追求性能，其次追求优雅

这是一个 Thrift 服务定义：

```go
struct User { 
  1: string name, 
  2: i32 id, 
  3: string email 
} 

struct Error { 
  1: i32 code, 
  2: string detail 
} 

service Users { 
  Error createUser(1: User user) 
} 

```

在[`thrift.apache.org`](https://thrift.apache.org)上找到更多关于 Apache Thrift 的信息。

# 协议缓冲

协议缓冲是谷歌的产品，它们刚刚进入第三个版本。协议缓冲采用提供一种 DSL 的方法，该生成器（用 C 编写）读取并可以生成超过十种语言的客户端和服务器存根，其中主要的前十种由谷歌维护，包括：Go、Java、C、NodeJS 的 JavaScript。

协议缓冲是一个可插拔的架构，因此可以编写自己的插件来生成各种端点，而不仅仅是 RPC；然而，RPC 是主要用例，因为它们与 gRPC 框架耦合。

gRPC 是由谷歌设计的一个快速且语言无关的 RPC 框架，它起源于一个内部项目，在该项目中延迟和速度在谷歌的架构中至关重要。默认情况下，gRPC 使用协议缓冲作为序列化和反序列化结构化数据的方法。以下示例展示了这种 DSL 的一个例子。

协议缓冲服务定义：

```go
service Users { 
  rpc CreateUser (User) returns (Error) {} 
} 

message User { 
  required string name = 1; 
  required int32 id = 2; 
  optional string email = 3; 
} 

message Error { 
  optional code int32 = 1 
  optional detail string = 2 
} 

```

在[`developers.google.com/protocol-buffers/`](https://developers.google.com/protocol-buffers/)上找到更多关于协议缓冲的信息。

# JSON-RPC

JSON-RPC 是尝试使用 JSON 表示对象以用于 RPC 的标准方式。这消除了解码任何专有二进制协议的需要，但以传输速度为代价。没有要求任何特定的客户端或服务器提供此数据格式，TCP 套接字，以及能够编写大多数所有编程语言都能管理的字符串的能力，这些都是您所需的所有。

与 Thrift 和协议缓冲不同，JSON-RPC 为消息序列化设定了标准。

JSON-RPC 实现了一些很好的功能，允许批量处理请求；每个请求都包含一个`id`参数，由客户端建立。当服务器响应时，它将返回相同的标识符，使客户端能够理解响应与哪个请求相关。

这是一个 JSON-RPC 序列化请求：

```go
{
  "jsonrpc": "2.0", 
  "method": "": "Users.v1.CreateUser",
  "params": {
    "name": "Nic Jackson", 
    "id": 12335432434
  }, 
  "id": 1
} 

```

这是一个 JSON-RPC 序列化响应：

```go
{
  "jsonrpc": "2.0", 
  "result": {...}, 
  "id":": 1
} 

```

在[`www.jsonrpc.org/specification`](http://www.jsonrpc.org/specification)上找到更多关于 JSON-RPC 2.0 的信息。

# 过滤

当我们查看 RESTful API 时，我们讨论了使用查询字符串执行过滤操作的概念，例如：

+   分页

+   过滤

+   排序

显然，如果我们正在编写 RPC API，我们没有查询字符串的便利；然而，实现这些概念非常有用。只要我们保持一致性，就没有任何理由我们不能在我们的请求对象上定义用于过滤条件的参数：

```go
{
 "jsonrpc": "2.0", 
 "method": "": "Users.v1.GetUserLog",
 "params": {
   "name": "Nic Jackson", 
   "id": 12335432434,
   "filter": { 
     "page_start":": 1,  //optional 
     "page_size"" : 10,  //optional 
     "sort": "name DESC" //optional 
   },
 "id": 1
}

```

这只是一个例子，你可能会选择根据自己特定的需求实现，然而，关键在于一致性。如果我们为每个方法使用相同的对象，我们可以合理地确信我们的用户会对此感到满意。

# API 版本控制

API 版本控制是你从一开始就应该考虑的事情，并且尽量避免。一般来说，你将需要修改你的 API，然而，维护*n*个不同版本可能会非常痛苦，所以一开始就进行前期设计思考可以节省你很多麻烦。

在我们查看您如何可以版本控制 API 之前，这相当直接，让我们看看您应该在何时进行版本控制。

当您引入重大变更时，您将增加 API 版本号。

重大变更包括：

+   删除或重命名 API 或 API 参数

+   改变 API 参数的类型，例如，从整数更改为字符串

+   修改响应代码、错误代码或故障合同

+   改变现有 API 的行为

不涉及重大变更的事情包括：

+   向返回实体添加参数

+   添加额外的端点或功能

+   修复错误或其他不包含在重大变更列表中的维护工作

# 语义版本控制

微服务应该实施主版本号方案。通常，设计者会选择只实现主版本号，并暗示次要版本为`.0`，根据语义版本控制原则[`semver.org`](http://semver.org)，次要版本通常表示以向后兼容的方式实现的功能添加。这可能是向您的 API 添加额外的端点。可以争辩说，由于这不会影响客户端与您的 API 交互的能力，因此您不必担心次要版本，只需关注主版本即可，因为客户端不需要请求特定的版本才能正常工作。

当对 API 进行版本控制时，我认为删除次要版本并只关注主版本会更简洁。我们会采取这种方法的两个原因：

+   URI 变得更加易读，点号仅用作网络位置分隔符。当使用 RPC API 时，点号仅用于分隔`API.VERSION.METHOD`，使一切更容易阅读。

+   我们应该通过我们的 API 版本控制推断出变化是一件大事，并且对客户端的功能有影响。内部我们仍然可以使用`Major.Minor`；然而，这不需要对客户端来说是一个需要考虑的事情，因为他们将没有能力选择使用 API 的次要版本。

# REST API 的版本控制格式

为了允许客户端请求特定的 API 版本，有三种常见的方法可以实现。

这也可以作为 URI 的一部分来完成：

```go
https://myserver.com/v1/helloworld 

```

也可以作为查询字符串参数来完成：

```go
https://myserver.com/helloworld?api-version=1 

```

最后，可以通过使用自定义 HTTP 头来完成：

```go
GET https://myserver.com/helloworld
api-version: 2

```

无论你选择哪种方式来实现版本控制，这都取决于你和你团队，但它在你的前期设计思考中应该扮演重要角色。一旦你决定了一个选项，坚持使用它，确保为你的消费者提供一致且优秀的体验应该是你的主要目标之一。

# RPC API 的版本控制格式

RPC 的版本控制可能稍微困难一些，因为你很可能没有使用 HTTP 作为传输。然而，这仍然是可能的。处理这种情况的最佳方式是处理程序的命名空间。

在 go 基础包中，你可以给你的处理程序命名，例如`Greet.v1.HelloWorld`。

# RPC 的命名

使用 RPC 时，你没有使用 HTTP 动词来传达 API 意图的奢侈，例如，你有用户集合。在使用 HTTP API 的情况下，你可以通过`GET`、`POST`、`DELETE`等来分割各种操作。这在 RPC API 中是不可能的，你需要像编写 Go 代码中的方法一样思考，所以例如：

```go
GET /v1/users 

```

前面的代码也可以写成如下 RPC 方法：

```go
Users.v1.Users 
GET /v1/users/123434 

```

或者，它也可以写成如下 RPC 方法：

```go
Users.v1.User 

```

子集合在语义上变得稍微少一些，而在 RESTful API 中，你可以做以下操作：

```go
GET /v1/users/12343/permissions/1232 

```

你不能使用 RPC API 来做这件事，你必须明确指定方法作为一个单独的实体：

```go
Permissions.v1.Permission 

```

方法名也需要推断 API 将要执行的操作；你不能依赖于 HTTP 动词的使用，所以如果你有一个可以删除用户的方法，你必须将删除动词添加到方法调用中，例如：

```go
DELETE /v1/users/123123 

```

前面的代码将变为：

```go
Users.v1.DeleteUser 

```

# 对象类型标准化

无论你使用的是自定义二进制序列化、JSON 还是 JSON-RPC，你都需要考虑你的用户将如何处理交易另一端的对象。许多使用 stub 生成客户端代码的序列化包，如 Protocol Buffers 和 Thrift，会愉快地处理简单类型如日期的序列化到本地类型，这使得消费者可以轻松使用和操作这些对象。然而，如果你使用 JSON 或 JSON-RPC，没有日期作为本地类型的概念，因此回退到 ISO 标准可能是有用的，客户端用户可以轻松反序列化。微软 API 设计指南提供了一些关于如何处理日期和持续时间的良好建议。

# 日期

当返回日期时，你应该始终使用`DateLiteral`格式，最好是`Iso8601Literal`。如果你需要以除`Iso8601Literal`之外的其他格式发送日期，则可以使用`StructuredDateLiteral`格式，这允许你在返回的实体中指定类型。

非正式的 `Iso8601Literal` 格式是使用最简单的方法，并且几乎任何消费您 API 的客户端都应该能够理解：

```go
{"date": "2016-07-14T16:00Z"} 

```

更正式的 `StucturedDateLiteral` 不返回字符串，而是一个包含两个属性 `kind` 和 `value` 的实体：

```go
{"date": {"kind": "U", "value": 1471186826}} 

```

允许的种类有：

+   `C`: **CLR**；自 2000 年 1 月 1 日午夜以来的毫秒数

+   `E`: **ECMAScript**；自 1970 年 1 月 1 日午夜以来的毫秒数

+   `I`: **ISO 8601**；一个限于 ECMAScript 子集的字符串

+   `O`: **OLE 日期**；整数部分是自 1899 年 12 月 31 日午夜以来的天数，小数部分是当天的时间（0.5 = 中午）

+   `T`: **刻度**；自 1601 年 1 月 1 日午夜以来的刻度（100 纳秒间隔）数

+   `U`: **UNIX**；自 1970 年 1 月 1 日午夜以来的秒数

+   `W`: **Windows**；自 1601 年 1 月 1 日午夜以来的毫秒数

+   `X`: **Excel**；与 `O` 相同，但 1900 年被错误地视为闰年，且天数为 "January 0 (零)"

# 持续时间

持续时间序列化为符合 ISO 8601，并以下列格式表示：

```go
P[n]Y[n]M[n]DT[n]H[n]M[n]S 

```

+   `P`: 这是持续时间标识符（历史上称为"周期"），放置在持续时间表示的开始处

+   `Y`: 这是跟随年数值的年标识符

+   `M`: 这是跟随月数值的月标识符

+   `W`: 这是跟随周数值的周标识符

+   `D`: 这是跟随天数值的日标识符

+   `T`: 这是时间表示中的时间组件之前的时间标识符

+   `H`: 这是跟随小时数值的时标识符

+   `M`: 这是跟随分钟数值的分钟标识符

+   `S`: 这是跟随秒数值的秒标识符

例如，`P3Y6M4DT12H30M5S` 表示 "三年，六个月，四天，十二小时，三十分钟和五秒" 的持续时间。

# 间隔

再次，ISO 8601 规范的一部分是，如果您需要接收或发送一个间隔，您可以使用以下格式：

+   开始和结束，例如 `2007-03-01T13:00:00Z/2008-05-11T15:30:00Z`

+   开始和持续时间，例如 `2007-03-01T13:00:00Z/P1Y2M10DT2H30M`

+   持续时间和结束，例如 `P1Y2M10DT2H30M/2008-05-11T15:30:00Z`

+   仅持续时间，例如 `P1Y2M10DT2H30M`，带有额外的上下文信息

在 [`github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#113-json-serialization-of-dates-and-times`](https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#113-json-serialization-of-dates-and-times) 查找有关日期和时间的 JSON 序列化的更多信息。

# 记录 API

记录 API 非常有用，无论你打算让 API 被公司内部的其他团队、外部用户，甚至只是你自己使用。你会感谢自己花时间记录 API 操作并保持文档更新。保持文档更新不应是一项艰巨的任务。有许多应用程序可以从你的源代码自动生成文档，所以你只需要在构建工作流程中运行此应用程序即可。

# 基于 REST 的 API

目前有三个主要标准正在争夺成为 REST API 文档的皇后：

+   Swagger

+   API Blueprint

+   RAML

# Swagger

Swagger 是由 SmartBear 设计的，并被选为 Open API 创新计划的一部分；这可能会给它带来最大的机会，成为文档化 RESTful API 的标准。然而，Open API 创新计划 ([`openapis.org`](https://openapis.org)) 是一个行业机构，它是否能够获得 W3C 在网络标准方面的认可，可能取决于更多知名企业的加入。

文档是用 YAML 编写的，各种代码生成工具既可以从源代码生成 Swagger 文档，也可以生成客户端 SDK。该标准在功能列表上非常全面，并且相对简单易写，同时被开发社区广泛理解。

Swagger 的代码示例如下所示：

```go
/pets: 
  get: 
    description: Returns all pets from the system that the user has access to 
    produces: 
      - application/json 
    responses: 
      '200''200': 
        description: A list of pets. 
  schema: 
    type: array 
    items: 
      $ref: ''#/definitions/Pet'Pet' 

definitions: 
  Pet: 
    type: object 
    properties: 
      name: 
  type: string 
    description: name of the pet 

```

在 [`swagger.io`](http://swagger.io) 查找有关 Swagger 的更多信息。

# API Blueprint

API Blueprint 是由 Apiary 设计的开放标准，并按照 MIT 许可证发布。它与 Apiary 的产品紧密相连。然而，它也可以独立使用，并且有各种开源工具可以读取和写入该格式。

文档是用 Markdown 编写的，这使得编写文档感觉更加自然，而不是处理嵌套的对象层。

API Blueprint 的代码示例如下所示：

```go
FORMAT: 1A 

# Data Structures 

## Pet (object) 
+ name: Jason (string) - Name of the pet. 

# Pets [/pets] 

Returns all pets from the system that the user has access to'to' 

## Retrieve all pets [GET] 
+ Response 200 (application/json) 
+ Attributes (array[Pet]) 

```

在 [`apiblueprint.org`](https://apiblueprint.org) 查找有关 API Blueprint 的更多信息。

# RAML

**RAML** 代表 **RESTful API Modelling Language**，并以 `YAML` 格式编写。它的目标是允许定义一种人类可读的格式，用于描述资源、方法、参数、响应、媒体类型以及其他构成你 API 基础的 HTTP 构造。

RAML 的代码示例如下所示：

```go
#%RAML 1.0 
title: Pets API 
mediaType: [ application/json] 
types: 
  Pet: 
    type: object 
    properties: 
      name: 
        type: string 
        description: name of the pet 
/pets: 
  description: Returns all pets from the system that the user has access to 
  get: 
    responses: 
      200: 
        body: Pet[] 

```

在 [`raml.org`](http://raml.org) 查找有关 RAML 的更多信息。

# 基于 RPC 的 API

在 RPC API 中，有一种观点认为你的合同就是你的文档，在以下示例中，我们使用协议缓冲区 DSL 定义接口，并根据需要添加任何必要的注释来帮助消费者。主要遵循的理论是自文档化代码，你的方法和参数名称应该推断意图并提供足够的描述，以消除注释的使用。

协议缓冲区示例：

```go
// The greeting service definition. 
service Users { 
  // Create user creates a user in the system with the given User details, 
  // it returns an Error message which will be nil on a successful operation 
  rpc CreateUser (User) returns (Error) {} 
} 

// Person describes a user entity 
message User { 
  // name is a required field and represents the name of 
  required string name = 1; 
  // id is the unique identifier for the user in the sytem 
  required int32 id = 2; 
  // email is the users email address and is an optional field  
  optional string email = 3; 
} 

message Error { 
  optional code int32 = 1 
  optional detail string = 2 
} 

```

你选择哪种标准完全取决于你，你的工作流程，你的团队标准，以及你的用户。然而，一旦你选择了与命名约定相同的方法，你应该坚持在你的所有 API 中保持一致的风格。

# 摘要

在本章中，我们没有花太多时间查看代码；然而，我们已经研究了编写优秀 API 的某些基本概念，这和能够编写代码一样重要。

本章的大部分内容都关注于 RESTful API，因为与 RPC 不同，我们需要在它们的用法上更加描述性。我们还有能力利用 HATEOAS 的原则，这是在使用 RPC 时所不具备的。

在下一章中，我们将开始探讨 Go 社区中存在的一些令人惊叹的框架，这样我们就可以开始应用这些原则，并进一步深化我们对微服务精通的进步。
