# 第二章。Go 中的 RESTful 服务

当人们通常设计 API 和 Web 服务时，他们通常将它们作为事后思考，或者至少作为大型应用程序的最后一步。

这背后有很好的逻辑——应用程序首先出现，当桌子上没有产品时满足开发人员并不太有意义。因此，通常当应用程序或网站创建时，那就是核心产品，任何额外的 API 资源都是其次的。

随着 Web 近年来的变化，这个系统也有了一些变化。现在，写 API 或 Web 服务然后再写应用程序并不是完全不常见。这通常发生在高度响应的单页应用程序或移动应用程序中，其中结构和数据比演示层更重要。

我们的总体项目——一个社交网络——将展示数据和架构优先的应用程序的性质。我们将拥有一个功能齐全的社交网络，可以在 API 端点上进行遍历和操作。然而，在本书的后面，我们将在演示层上玩一些有趣的东西。

尽管这背后的概念可能被视为完全示范性的，但现实是，这种方法是当今许多新兴服务和应用程序的基础。一个新站点或服务通常会使用 API 进行启动，有时甚至只有 API。

在本章中，我们将讨论以下主题：

+   设计我们的应用程序的 API 策略

+   REST 的基础知识

+   其他 Web 服务架构和方法

+   编码数据和选择数据格式

+   REST 动作及其作用

+   使用 Gorilla 的 mux 创建端点

+   应用程序版本控制的方法

# 设计我们的应用程序

当我们着手构建更大的社交网络应用程序时，我们对我们的数据集和关系有一个大致的想法。当我们将这些扩展到 Web 服务时，我们不仅要将数据类型转换为 API 端点，还要转换关系和操作。

例如，如果我们希望找到一个用户，我们会假设数据保存在一个名为`users`的数据库中，并且我们希望能够使用`/api/users`端点检索数据。这是合理的。但是，如果我们希望获取特定用户呢？如果我们希望查看两个用户是否连接？如果我们希望编辑一个用户在另一个用户的照片上的评论？等等。

这些是我们应该考虑的事情，不仅在我们的应用程序中，也在我们围绕它构建的 Web 服务中（或者在这种情况下，反过来，因为我们的 Web 服务首先出现）。

到目前为止，我们的应用程序有一个相对简单的数据集，所以让我们以这样的方式来完善它，以便我们可以创建、检索、更新和删除用户，以及创建、检索、更新和删除用户之间的关系。我们可以把这看作是在传统社交网络上“加为好友”或“关注”某人。

首先，让我们对我们的`users`表进行一些维护。目前，我们只在`user_nickname`变量上有一个唯一索引，但让我们为`user_email`创建一个索引。考虑到理论上一个人只能绑定一个特定的电子邮件地址，这是一个相当常见和合乎逻辑的安全点。将以下内容输入到您的 MySQL 控制台中：

```go
ALTER TABLE `users`
  ADD UNIQUE INDEX `user_email` (`user_email`);
```

现在我们每个电子邮件地址只能有一个用户。这是有道理的，对吧？

接下来，让我们继续创建用户关系的基础。这些将不仅包括加为好友/关注的概念，还包括屏蔽的能力。因此，让我们为这些关系创建一个表。再次，将以下代码输入到您的控制台中：

```go
CREATE TABLE `users_relationships` (
  `users_relationship_id` INT(13) NOT NULL,
  `from_user_id` INT(10) NOT NULL,
  `to_user_id` INT(10) unsigned NOT NULL,
  `users_relationship_type` VARCHAR(10) NOT NULL,
  `users_relationship_timestamp` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`users_relationship_id`),
  INDEX `from_user_id` (`from_user_id`),
  INDEX `to_user_id` (`to_user_id`),
  INDEX `from_user_id_to_user_id` (`from_user_id`, `to_user_id`),

  INDEX `from_user_id_to_user_id_users_relationship_type` (`from_user_id`, `to_user_id`, `users_relationship_type`)
)
```

我们在这里做的是为包括各种用户的关系创建了一个表，以及时间戳字段告诉我们关系是何时创建的。

那么，我们现在在哪里？嗯，现在，我们有能力创建、检索、更新和删除用户信息以及用户之间的关系。我们的下一步将是构想一些 API 端点，让我们的网络服务的消费者能够做到这一点。

在上一章中，我们创建了我们的第一个端点，`/api/user/create`和`/api/user/read`。然而，如果我们想要完全控制刚才讨论的数据，我们需要更多。

在那之前，让我们谈谈与网络服务相关的最重要的概念，特别是那些利用 REST 的概念。

# 看看 REST

那么，REST 到底是什么，它从哪里来？首先，REST 代表**表述性状态转移**。这很重要，因为数据（及其元数据）的表述是数据传输的关键部分。

缩写中的**状态**方面有点误导，因为无状态实际上是架构的核心组件。

简而言之，REST 提供了一种简单的、无状态的机制，用于通过 HTTP（以及其他一些协议）呈现数据，这种机制是统一的，并包括缓存指令等控制机制。

这种架构最初是作为罗伊·菲尔丁在加州大学尔湾分校的论文的一部分而产生的。从那时起，它已经被**万维网联盟**（**W3C**）进行了编码和标准化。

一个 RESTful 应用程序或 API 将需要几个重要的组件，我们现在将概述这些组件。

## 在 API 中进行表述

API 最重要的组成部分是我们将作为网络服务一部分传递的数据。通常，它是 JSON、RSS/XML 格式的格式化文本，甚至是二进制数据。

为了设计一个网络服务，确保您的格式与您的数据匹配是一个好习惯。例如，如果您创建了一个用于传递图像数据的网络服务，很容易将这种数据塞进文本格式中。将二进制数据转换为 Base64 编码并通过 JSON 发送并不罕见。

然而，API 的一个重要考虑因素是数据大小的节俭。如果我们以前的例子并将我们的图像数据编码为 Base64，我们最终得到的 API 有效负载将增加近 40%。通过这样做，我们将增加服务的延迟并引入潜在的烦恼。如果我们可以可靠地传输数据，那就没有理由这样做。

模型中的表述也应该起到重要的作用——满足客户端更新、删除或检索特定资源的所有要求。

## 自我描述

当我们说自我描述时，我们也可以将其描述为自包含，以包括 REST 的两个核心组件——响应应该包括客户端每个请求所需的一切，并且应该包括（明确或隐含地）有关如何处理信息的信息。

第二部分涉及缓存规则，我们在第一章中简要提到了*我们在 Go 中的第一个 API*。

提供有关 API 请求中包含的资源的有价值的缓存信息是重要的。这可以消除以后的冗余或不必要的请求。

这也引入了 REST 的无状态性概念。我们的意思是每个请求都是独立存在的。正如前面提到的，任何单个请求都应该包括满足该请求所需的一切。

最重要的是，这意味着放弃普通的 Web 架构的想法，其中您可以设置 cookie 或会话变量。这本质上不是 RESTful。首先，我们的客户端不太可能支持 cookie 或持续会话。但更重要的是，它减少了对任何给定 API 端点所期望的响应的全面和明确的性质。

### 提示

自动化流程和脚本当然可以处理会话，并且它们可以像 REST 的初始提案一样处理它们。这更多是一种演示而不是 REST 拒绝将持久状态作为其精神的一部分的原因。

## URI 的重要性

出于我们稍后将在本章讨论的原因，URI 或 URL 是良好 API 设计中最关键的因素之一。有几个原因：

+   URI 应该是有信息的。我们不仅应该了解数据端点的信息，还应该知道我们可能期望看到的返回数据。其中一些是程序员的习惯用法。例如，`/api/users`会暗示我们正在寻找一组用户，而`/api/users/12345`则表示我们期望获取有关特定用户的信息。

+   URI 不应该在将来中断。很快，我们将讨论版本控制，但这只是一个地方，稳定的资源端点的期望非常重要。如果您的服务的消费者在时间上发现其应用程序中缺少或损坏的链接而没有警告，这将导致非常糟糕的用户体验。

+   无论您在开发 API 或 Web 服务时有多少远见，事情都会发生变化。考虑到这一点，我们应该通过利用 HTTP 状态代码来对现有 URI 指示新位置或错误，而不是允许它们简单地中断。

### HATEOAS

**HATEOAS**代表**超媒体作为应用程序状态的引擎**，是 REST 架构中 URI 的主要约束。其背后的核心原则要求 API 不应引用固定的资源名称或实际的层次结构本身，而应该专注于描述所请求的媒体和/或定义应用程序状态。

### 注意

您可以通过访问 Roy Fielding 的博客[`roy.gbiv.com/untangled/`](http://roy.gbiv.com/untangled/)，阅读有关 REST 及其原始作者定义的要求的更多信息。

# 其他 API 架构

除了 REST，我们还将在本书中查看并实施一些其他常见的 API 和 Web 服务架构。

在大多数情况下，我们将专注于 REST API，但我们还将涉及 SOAP 协议和用于 XML 摄入的 API，以及允许持久性的较新的异步和基于 Web 套接字的服务。

## 远程过程调用

**远程过程调用**，或**RPC**，是一种长期存在的通信方法，构成了后来成为 REST 的基础。虽然仍然有一些使用 RPC 的价值，特别是 JSON-RPC，但我们不会在本书中花太多精力来适应它。

如果您对 RPC 不熟悉，与 REST 相比，其核心区别在于只有一个端点，请求本身定义了 Web 服务的行为。

### 注意

要了解有关 JSON-RPC 的更多信息，请访问[`json-rpc.org/`](http://json-rpc.org/)。

# 选择格式

使用的格式问题曾经是一个比今天更棘手的问题。我们曾经有许多特定于个人语言和开发人员的格式，但 API 世界已经导致这些格式的广度收缩了一些。

Node 和 JavaScript 作为数据传输格式的通用语言的崛起使大多数 API 首先考虑 JSON。 JSON 是一个相对紧凑的格式，现在几乎每种主要语言都有支持，Go 也不例外。

## JSON

以下是一个简单快速的示例，说明 Go 如何使用核心包发送和接收 JSON 数据：

```go
package main

import
(
  "encoding/json"
  "net/http"
  "fmt"
)

type User struct {
  Name string `json:"name"`
  Email string `json:"email"`
  ID int `json:"int"`
}

func userRouter(w http.ResponseWriter, r *http.Request) {
  ourUser := User{}
  ourUser.Name = "Bill Smith"
  ourUser.Email = "bill.smith@example.com"
  ourUser.ID = 100

  output,_ := json.Marshal(&ourUser)
  fmt.Fprintln(w, string(output))
}

func main() {

  fmt.Println("Starting JSON server")
  http.HandleFunc("/user", userRouter)
  http.ListenAndServe(":8080",nil)

}
```

这里需要注意的是`User`结构中变量的 JSON 表示。每当您在重音符号（```go) characters, this represents a rune. Although a string is represented in double quotes and a char in single, the accent represents Unicode data that should remain constant. Technically, this content is held in an `int32` value.

In a struct, a third parameter in a variable/type declaration is called a tag. These are noteworthy for encoding because they have direct translations to JSON variables or XML tags.

Without a tag, we'll get our variable names returned directly.

## XML

As mentioned earlier, XML was once the format of choice for developers. And although it's taken a step back, almost all APIs today still present XML as an option. And of course, RSS is still the number one syndication format.

As we saw earlier in our SOAP example, marshaling data into XML is simple. Let's take the data structure that we used in the earlier JSON response and similarly marshal it into the XML data in the following example.

Our `User` struct is as follows:

```

类型用户结构{

Name string `xml：“name”`

电子邮件字符串`xml：“email”`

ID int `xml：“id”`

}

```go

And our obtained output is as follows:

```

ourUser：= User{}

ourUser.Name =“Bill Smith”

ourUser.Email =“bill.smith@example.com”

ourUser.ID = 100

output，_：= xml.Marshal（&ourUser）

fmt.Fprintln（w，string（output））

```go

## YAML

**YAML** was an earlier attempt to make a human-readable serialized format similar to JSON. There does exist a Go-friendly implementation of YAML in a third-party plugin called `goyaml`.

You can read more about `goyaml` at [`godoc.org/launchpad.net/goyaml`](https://godoc.org/launchpad.net/goyaml). To install `goyaml`, we'll call a `go get launchpad.net/goyaml` command.

As with the default XML and JSON methods built into Go, we can also call `Marshal` and `Unmarshal` on YAML data. Using our preceding example, we can generate a YAML document fairly easily, as follows:

```

主要

进口

(

“ fmt”

“net/http”

“launchpad.net/goyaml”

）

类型用户结构{

Name string`}`

电子邮件字符串

ID int

}

func userRouter(w http.ResponseWriter, r *http.Request) {

ourUser := User{}

ourUser.Name = "Bill Smith"

ourUser.Email = "bill.smith@example.com"

ourUser.ID = 100

output,_ := goyaml.Marshal(&ourUser)

fmt.Fprintln(w, string(output))

}

func main() {

fmt.Println("Starting YAML server")

http.HandleFunc("/user", userRouter)

http.ListenAndServe(":8080",nil)

}

```go

The obtained output is as shown in the following screenshot:

![YAML](img/1304OS_02_02.jpg)

## CSV

The **Comma Separated Values** (**CSV**) format is another stalwart that's fallen somewhat out of favor, but it still persists as a possibility in some APIs, particularly legacy APIs.

Normally, we wouldn't recommend using the CSV format in this day and age, but it may be particularly useful for business applications. More importantly, it's another encoding format that's built right into Go.

Coercing your data into CSV is fundamentally no different than marshaling it into JSON or XML in Go because the `encoding/csv` package operates with the same methods as these subpackages.

# Comparing the HTTP actions and methods

An important aspect to the ethos of REST is that data access and manipulation should be restricted by verb/method.

For example, the `GET` requests should not allow the user to modify, update, or create the data within. This makes sense. `DELETE` is fairly straightforward as well. So, what about creating and updating? However, no such directly translated verbs exist in the HTTP nomenclature.

There is some debate on this matter, but the generally accepted method for handling this is to use `PUT` to update a resource and `POST` to create it.

### Note

Here is the relevant information on this as per the W3C protocol for HTTP 1.1:

The fundamental difference between the `POST` and `PUT` requests is reflected in the different meaning of the Request-URI. The URI in a `POST` request identifies the resource that will handle the enclosed entity. This resource might be a data-accepting process, a gateway to some other protocol, or a separate entity that accepts annotations. In contrast, the URI in a `PUT` request identifies the entity enclosed with the request—the user agent knows which URI is intended and the server *MUST NOT* attempt to apply the request to some other resource. If the server desires that the request to be applied to a different URI, it MUST send a 301 (Moved Permanently) response; the user agent MAY then make its own decision regarding whether or not to redirect the request.

So, if we follow this, we can assume that the following actions will translate to the following HTTP verbs:

| Actions | HTTP verbs |
| --- | --- |
| Retrieving data | `GET` |
| Creating data | `POST` |
| Updating data | `PUT` |
| Deleting data | `DELETE` |

Thus, a `PUT` request to, say, `/api/users/1234` will tell our web service that we're accepting data that will update or overwrite the user resource data for our user with the ID `1234`.

A `POST` request to `/api/users/1234` will tell us that we'll be creating a new user resource based on the data within.

### Note

It is very common to see the update and create methods switched, such that `POST` is used to update and `PUT` is used for creation. On the one hand, it's easy enough to do it either way without too much complication. On the other hand, the W3C protocol is fairly clear.

## The PATCH method versus the PUT method

So, you might think after going through the last section that everything is wrapped up, right? Cut and dry? Well, as always, there are hitches and unexpected behaviors and conflicting rules.

In 2010, there was a proposed change to HTTP that would include a `PATCH` method. The difference between `PATCH` and `PUT` is somewhat subtle, but, the shortest possible explanation is that `PATCH` is intended to supply only partial changes to a resource, whereas `PUT` is expected to supply a complete representation of a resource.

The `PATCH` method also provides the potential to essentially *copy* a resource into another resource given with the modified data.

For now, we'll focus just on `PUT`, but we'll look at `PATCH` later on, particularly when we go into depth about the `OPTIONS` method on the server side of our API.

# Bringing in CRUD

The acronym **CRUD** simply stands for **Create, Read (or Retrieve), Update, and Delete**. These verbs might seem noteworthy because they closely resemble the HTTP verbs that we wish to employ to manage data within our application.

As we discussed in the last section, most of these verbs have seemingly direct translations to HTTP methods. We say "seemingly" because there are some points in REST that keep it from being entirely analogous. We will cover this a bit more in later chapters.

`CREATE` obviously takes the role of the `POST` method, `RETRIEVE` takes the place of `GET`, `UPDATE` takes the place of `PUT`/`PATCH`, and `DELETE` takes the place of, well, `DELETE`.

If we want to be fastidious about these translations, we must clarify that `PUT` and `POST` are not direct analogs to `UPDATE` and `CREATE`. In some ways this relates to the confusion behind which actions `PUT` and `POST` should provide. This all relies on the critical concept of idempotence, which means that any given operation should respond in the same way if it is called an indefinite number of times.

### Tip

**Idempotence** is the property of certain operations in mathematics and computer science that can be applied multiple times without changing the result beyond the initial application.

For now, we'll stick with our preceding translations and come back to the nitty-gritty of `PUT` versus `POST` later in the book.

# Adding more endpoints

Given that we now have a way of elegantly handling our API versions, let's take a step back and revisit user creation. Earlier in this chapter, we created some new datasets and were ready to create the corresponding endpoints.

Knowing what you know now about HTTP verbs, we should restrict access to user creation through the POST method. The example we built in the first chapter did not work exclusively with the POST request (or with POST requests at all). Good API design would dictate that we have a single URI for creating, retrieving, updating, and deleting any given resource.

With all of this in mind, let's lay out our endpoints and what they should allow a user to accomplish:

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/api` | `OPTIONS` | To outline the available actions within the API |
| `/api/users` | `GET` | To return users with optional filtering parameters |
| `/api/users` | `POST` | To create a user |
| `/api/user/123` | `PUT` | To update a user with the ID `123` |
| `/api/user/123` | `DELETE` | To delete a user with the ID `123` |

For now, let's just do a quick modification of our initial API from Chapter 1, *Our First API in Go*, so that we allow user creation solely through the `POST` method.

Remember that we've used **Gorilla web toolkit** to do routing. This is helpful for handling patterns and regular expressions in requests, but it is also helpful now because it allows you to delineate based on the HTTP verb/method.

In our example, we created the `/api/user/create` and `/api/user/read` endpoints, but we now know that this is not the best practice in REST. So, our goal now is to change any resource requests for a user to `/api/users`, and to restrict creation to `POST` requests and retrievals to `GET` requests.

In our main function, we'll change our handlers to include a method as well as update our endpoint:

```

routes := mux.NewRouter()

路由.HandleFunc("/api/users", UserCreate).Methods("POST")

routes.HandleFunc("/api/users", UsersRetrieve).Methods("GET")

```go

You'll note that we also changed our function names to `UserCreate` and `UsersRetrieve`. As we expand our API, we'll need methods that are easy to understand and can relate directly to our resources.

Let's take a look at how our application changes:

```

package main

import (

"database/sql"

"encoding/json"

"fmt"

_ "github.com/go-sql-driver/mysql"

"github.com/gorilla/mux"

"net/http"

"log"

)

var database *sql.DB

```go

Up to this point everything is the same—we require the same imports and connections to the database. However, the following code is the change:

```

type Users struct {

Users []User `json:"users"`

}

```go

We're creating a struct for a group of users to represent our generic `GET` request to `/api/users`. This supplies a slice of the `User{}` struct:

```

type User struct {

ID int "json:id"

Name  string "json:username"

Email string "json:email"

First string "json:first"

Last  string "json:last"

}

func UserCreate(w http.ResponseWriter, r *http.Request) {

NewUser := User{}

NewUser.Name = r.FormValue("user")

NewUser.Email = r.FormValue("email")

NewUser.First = r.FormValue("first")

NewUser.Last = r.FormValue("last")

output, err := json.Marshal(NewUser)

fmt.Println(string(output))

if err != nil {

fmt.Println("Something went wrong!")

}

sql := "INSERT INTO users set user_nickname='" + NewUser.Name + "', user_first='" + NewUser.First + "', user_last='" + NewUser.Last + "', user_email='" + NewUser.Email + "'"

q, err := database.Exec(sql)

if err != nil {

fmt.Println(err)

}

fmt.Println(q)

}

```go

Not much has changed with our actual user creation function, at least for now. Next, we'll look at the user data retrieval method.

```

func UsersRetrieve(w http.ResponseWriter, r *http.Request) {

w.Header().Set("Pragma","no-cache")

rows,_ := database.Query("select * from users LIMIT 10")

Response 	:= Users{}

for rows.Next() {

user := User{}

rows.Scan(&user.ID, &user.Name, &user.First, &user.Last, &user.Email )

Response.Users = append(Response.Users, user)

}

output,_ := json.Marshal(Response)

fmt.Fprintln(w,string(output))

}

```go

On the `UsersRetrieve()` function, we're now grabbing a set of users and scanning them into our `Users{}` struct. At this point, there isn't yet a header giving us further details nor is there any way to accept a starting point or a result count. We'll do that in the next chapter.

And finally we have our basic routes and MySQL connection in the main function:

```

func main() {

db, err := sql.Open("mysql", "root@/social_network")

if err != nil {

}

database = db

routes := mux.NewRouter()

routes.HandleFunc("/api/users", UserCreate).Methods("POST")

routes.HandleFunc("/api/users", UsersRetrieve).Methods("GET")

http.Handle("/", routes)

http.ListenAndServe(":8080", nil)

}

```go

As mentioned earlier, the biggest differences in `main` are that we've renamed our functions and are now relegating certain actions using the `HTTP` method. So, even though the endpoints are the same, we're able to direct the service depending on whether we use the `POST` or `GET` verb in our requests.

When we visit `http://localhost:8080/api/users` (by default, a `GET` request) now in our browser, we'll get a list of our users (although we still just have one technically), as shown in the following screenshot:

![Adding more endpoints](img/1304OS_02_03.jpg)

# Handling API versions

Before we go nay further with our API, it's worth making a point about versioning our API.

One of the all-too-common problems that companies face when updating an API is changing the version without breaking the previous version. This isn't simply a matter of valid URLs, but it is also about the best practices in REST and graceful upgrades.

Take our current API for example. We have an introductory `GET` verb to access data, such as `/api/users` endpoint. However, what this really should be is a clone of a versioned API. In other words, `/api/users` should be the same as `/api/{current-version}/users`. This way, if we move to another version, our older version will still be supported but not at the `{current-version}` address.

So, how do we tell users that we've upgraded? One possibility is to dictate these changes via HTTP status codes. This will allow consumers to continue accessing our API using older versions such as `/api/2.0/users`. Requests here will also let the consumer know that there is a new version.

We'll create a new version of our API in Chapter 3, *Routing and Bootstrapping*.

# Allowing pagination with the link header

Here's another REST point that can sometimes be difficult to handle when it comes to statelessness: how do you pass the request for the next set of results?

You might think it would make sense to do this as a data element. For example:

```

{ "payload": [ "item","item 2"], "next": "http://yourdomain.com/api/users?page=2" }

```go

Although this may work, it violates some principles of REST. First, unless we're explicitly returning hypertext, it is likely that we will not be supplying a direct URL. For this reason, we may not want to include this value in the body of our response.

Secondly, we should be able to do even more generic requests and get information about the other actions and available endpoints.

In other words, if we hit our API simply at `http://localhost:8080/api`, our application should return some basic information to consumers about potential next steps and all the available endpoints.

One way to do this is with the link header. A **link** header is simply another header key/value that you set along with your response.

### Tip

JSON responses are often not considered RESTful because they are not in a hypermedia format. You'll see APIs that embed `self`, `rel`, and `next` link headers directly in the response in unreliable formats.

JSON's primary shortcoming is its inability to support hyperlinks inherently. This is a problem that is solved by JSON-LD, which provides, among other things, linked documents and a stateless context.

**Hypertext** **Application Language** (**HAL**) attempts to do the same thing. The former is endorsed by W3C but both have their supporters. Both formats extend JSON, and while we won't go too deep into either, you can modify responses to produce either format.

Here's how we could do it in our `/api/users` `GET` request:

```

func UsersRetrieve(w http.ResponseWriter, r *http.Request) {

log.Println("starting retrieval")

start := 0

limit := 10

next := start + limit

w.Header().Set("Pragma","no-cache")

w.Header().Set("Link","<http://localhost:8080/api/users?start="+string(next)+"; rel=\"next\"")

rows,_ := database.Query("select * from users LIMIT 10")

Response := Users{}

for rows.Next() {

user := User{}

rows.Scan(&user.ID, &user.Name, &user.First, &user.Last, &user.Email )

Response.Users = append(Response.Users, user)

}

output,_ := json.Marshal(Response)

fmt.Fprintln(w,string(output))

}

```

这告诉客户端去哪里进行进一步的分页。当我们进一步修改这段代码时，我们将包括向前和向后的分页，并响应用户参数。

# Summary

此时，您不仅应该熟悉在 REST 和其他一些协议中创建 API Web 服务的基本思想，还应该熟悉格式和协议的指导原则。

我们在本章中尝试了一些东西，我们将在接下来的几章中更深入地探讨，特别是在 Go 语言本身的各种模板实现中的 MVC。

在下一章中，我们将构建我们初始端点的其余部分，并探索更高级的路由和 URL muxing。
