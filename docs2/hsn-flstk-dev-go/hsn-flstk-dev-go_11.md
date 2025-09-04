# 第八章：测试和基准测试你的 Web API

在生产软件环境中，测试至关重要。应用程序不仅需要测试功能，还需要进行基准测试和性能分析，以便我们可以检查应用程序的性能。本章将提供广泛的实用信息，介绍如何正确测试和基准测试你的应用程序。

在本章中，我们将涵盖以下主题：

+   Go 语言中的模拟类型

+   Go 语言中的单元测试

+   Go 语言中的基准测试

本章的代码可以在本书的 GitHub 仓库中找到：[`github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/tree/master/Chapter08`](https://github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/tree/master/Chapter08)。

# Go 语言中的测试

任何软件测试过程中的一个构建块被称为**单元测试**。单元测试在几乎任何编程语言中都是一个非常流行的概念，并且有众多的软件框架和语言扩展，允许你尽可能高效地执行单元测试。

单元测试的想法是单独测试你的软件中的每个单元或组件。单元可以简单地定义为你的软件中最小的可测试部分。

Go 语言配备了测试包，以及一些 Go 命令，使单元测试过程更加容易。该包可以在[`golang.org/pkg/testing/`](https://golang.org/pkg/testing/)找到。

在本节中，我们将更深入地探讨如何在 Go 语言中构建单元测试。然而，在我们开始编写 Go 语言的单元测试之前，我们首先需要了解模拟的概念。

# 模拟

模拟的概念在单元测试软件领域非常流行。它最好通过一个例子来描述。假设我们想要对 GoMusic 应用程序的 HTTP 处理函数之一进行单元测试。`GetProducts()`方法是一个展示我们例子的好方法，因为该方法的目的就是返回 GoMusic 商店中所有可销售产品的列表。以下是`GetProducts()`方法的代码：

```go
func (h *Handler) GetProducts(c *gin.Context) {
  if h.db == nil {
    c.JSON(http.StatusInternalServerError, gin.H{"error": "server database error"})
    return
  }
  products, err := h.db.GetAllProducts()
  if err != nil {
    c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
    return
  }
  fmt.Printf("Found %d products\n", len(products))
  c.JSON(http.StatusOK, products)
}
```

此方法只是从我们的数据库中检索所有产品，然后以 HTTP 响应的形式返回结果。此方法使用了`h.db.GetAllProducts()`方法从我们的数据库中检索数据。

因此，当需要对`GetProducts()`进行单元测试时，我们应该能够测试该方法的功能，而无需实际数据库。此外，我们还应该能够注入一些错误场景，例如让`h.db.GetAllProducts()`失败，并确保`GetProducts()`按预期反应。

你可能会想知道，为什么能够在不需要真实数据库的情况下测试像`GetProducts()`这样的方法很重要？答案是简单的——单元测试只关注你当前正在测试的单元，即`GetProducts()`方法，而不是你的数据库连接。

模拟对象类型可以被定义为对象类型，你可以使用它来模拟或伪造某种行为。换句话说，在`h.db.GetAllProducts()`方法的例子中，我们不是使用连接到真实数据库的对象类型，而是可以使用一个不连接到真实数据库但可以给我们提供所需结果的模拟类型，以执行`GetProducts()`方法的单元测试。

让我们回到记忆的深处，回忆一下`h.db.GetAllProducts()`是如何构建的。这段代码的数据库部分只是一个名为`DBLayer`的接口，我们用它来描述我们需要从数据库层获取的所有行为。

下面是`DBLayer`接口的样子：

```go
type DBLayer interface {
  GetAllProducts() ([]models.Product, error)
  GetPromos() ([]models.Product, error)
  GetCustomerByName(string, string) (models.Customer, error)
  GetCustomerByID(int) (models.Customer, error)
  GetProduct(int) (models.Product, error)
  AddUser(models.Customer) (models.Customer, error)
  SignInUser(username, password string) (models.Customer, error)
  SignOutUserById(int) error
  GetCustomerOrdersByID(int) ([]models.Order, error)
  AddOrder(models.Order) error
  GetCreditCardCID(int) (string, error)
  SaveCreditCardForCustomer(int, string) error
}
```

要为我们的数据库层创建一个模拟类型，我们只需要创建一个具体的类型，该类型将实现`DBLayer`接口，但不会连接到真实的数据库。

模拟对象需要返回一些模拟数据，这些数据我们用于我们的测试。我们可以简单地在这个模拟对象内部存储这些数据，以切片或映射的形式。

现在我们已经知道了什么是模拟，让我们创建我们的模拟数据库类型。

# 创建模拟数据库类型

在我们的`backend/src/dblayer`文件夹中，让我们添加一个名为`mockdblayer.go`的新文件。在这个新文件中，让我们创建一个名为`MockDBLayer`的类型：

```go
package dblayer

import (
  "encoding/json"
  "fmt"
  "strings"

  "github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/tree/master/Chapter08/backend/src/models"
)

type MockDBLayer struct {
 err error
 products []models.Product
 customers []models.Customer
 orders []models.Order
} 
```

`MockDBLayer`类型包含四种类型：

+   `err`：这是一个错误类型，我们可以在需要模拟错误场景时随意设置。我们将在编写单元测试时查看如何使用它。

+   `products`：这是我们存储模拟产品列表的地方。

+   `customers`：这是我们存储模拟客户列表的地方。

+   `orders`：这是我们存储模拟订单列表的地方。

接下来，让我们为我们的模拟类型编写一个构造函数：

```go
func NewMockDBLayer(products []models.Product, customers []models.Customer, orders []models.Order) *MockDBLayer {
  return &MockDBLayer{
    products: products,
    customers: customers,
    orders: orders,
  }
}
```

构造函数接受三个参数：一个`products`列表、一个`customers`列表和一个`orders`列表。这给其他开发者提供了定义他们自己的测试数据的机会，这对于灵活性来说是个好事情。然而，其他开发者应该能够使用一些预加载的模拟数据来初始化`MockDBLayer`类型：

```go
func NewMockDBLayerWithData() *MockDBLayer {
  PRODUCTS := `[
    {
        "ID": 1,
        "CreatedAt": "2018-08-14T07:54:19Z",
        "UpdatedAt": "2019-01-11T00:28:40Z",
        "DeletedAt": null,
        "img": "img/strings.png",
        "small_img": "img/img-small/strings.png",
        "imgalt": "string",
        "price": 100,
        "promotion": 0,
        "productname": "Strings",
        "Description": ""
    },
    {
        "ID": 2,
        "CreatedAt": "2018-08-14T07:54:20Z",
        "UpdatedAt": "2019-01-11T00:29:11Z",
        "DeletedAt": null,
        "img": "img/redguitar.jpeg",
        "small_img": "img/img-small/redguitar.jpeg",
        "imgalt": "redg",
        "price": 299,
        "promotion": 240,
        "productname": "Red Guitar",
        "Description": ""
    },
    {
        "ID": 3,
        "CreatedAt": "2018-08-14T07:54:20Z",
        "UpdatedAt": "2019-01-11T22:05:42Z",
        "DeletedAt": null,
        "img": "img/drums.jpg",
        "small_img": "img/img-small/drums.jpg",
        "imgalt": "drums",
        "price": 17000,
        "promotion": 0,
        "productname": "Drums",
        "Description": ""
    },
    {
        "ID": 4,
        "CreatedAt": "2018-08-14T07:54:20Z",
        "UpdatedAt": "2019-01-11T00:29:53Z",
        "DeletedAt": null,
        "img": "img/flute.jpeg",
        "small_img": "img/img-small/flute.jpeg",
        "imgalt": "flute",
        "price": 210,
        "promotion": 190,
        "productname": "Flute",
        "Description": ""
    },
    {
        "ID": 5,
        "CreatedAt": "2018-08-14T07:54:20Z",
        "UpdatedAt": "2019-01-11T00:30:12Z",
        "DeletedAt": null,
        "img": "img/blackguitar.jpeg",
        "small_img": "img/img-small/blackguitar.jpeg",
        "imgalt": "Black guitar",
        "price": 200,
        "promotion": 0,
        "productname": "Black Guitar",
        "Description": ""
    },
    {
        "ID": 6,
        "CreatedAt": "2018-08-14T07:54:20Z",
        "UpdatedAt": "2019-01-11T00:30:35Z",
        "DeletedAt": null,
        "img": "img/saxophone.jpeg",
        "small_img": "img/img-small/saxophone.jpeg",
        "imgalt": "Saxophone",
        "price": 1000,
        "promotion": 980,
        "productname": "Saxophone",
        "Description": ""
    }
]
`

  ORDERS := `[
  {
      "ID": 1,
      "CreatedAt": "2018-12-29T23:35:36Z",
      "UpdatedAt": "2018-12-29T23:35:36Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 1,
      "product_id": 1,
      "sell_price": 90,
      "purchase_date": "2018-12-29T23:34:32Z"
  },
  {
      "ID": 2,
      "CreatedAt": "2018-12-29T23:35:48Z",
      "UpdatedAt": "2018-12-29T23:35:48Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 1,
      "product_id": 2,
      "sell_price": 299,
      "purchase_date": "2018-12-29T23:34:53Z"
  },
  {
      "ID": 3,
      "CreatedAt": "2018-12-29T23:35:57Z",
      "UpdatedAt": "2018-12-29T23:35:57Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 1,
      "product_id": 3,
      "sell_price": 16000,
      "purchase_date": "2018-12-29T23:35:05Z"
  },
  {
      "ID": 4,
      "CreatedAt": "2018-12-29T23:36:18Z",
      "UpdatedAt": "2018-12-29T23:36:18Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 2,
      "product_id": 1,
      "sell_price": 95,
      "purchase_date": "2018-12-29T23:36:18Z"
  },
  {
      "ID": 5,
      "CreatedAt": "2018-12-29T23:36:39Z",
      "UpdatedAt": "2018-12-29T23:36:39Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 2,
      "product_id": 2,
      "sell_price": 299,
      "purchase_date": "2018-12-29T23:36:39Z"
  },
  {
      "ID": 6,
      "CreatedAt": "2018-12-29T23:38:13Z",
      "UpdatedAt": "2018-12-29T23:38:13Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 2,
      "product_id": 4,
      "sell_price": 205,
      "purchase_date": "2018-12-29T23:37:01Z"
  },
  {
      "ID": 7,
      "CreatedAt": "2018-12-29T23:38:19Z",
      "UpdatedAt": "2018-12-29T23:38:19Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 3,
      "product_id": 4,
      "sell_price": 210,
      "purchase_date": "2018-12-29T23:37:28Z"
  },
  {
      "ID": 8,
      "CreatedAt": "2018-12-29T23:38:28Z",
      "UpdatedAt": "2018-12-29T23:38:28Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 3,
      "product_id": 5,
      "sell_price": 200,
      "purchase_date": "2018-12-29T23:37:41Z"
  },
  {
      "ID": 9,
      "CreatedAt": "2018-12-29T23:38:32Z",
      "UpdatedAt": "2018-12-29T23:38:32Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 3,
      "product_id": 6,
      "sell_price": 1000,
      "purchase_date": "2018-12-29T23:37:54Z"
  },
  {
      "ID": 10,
      "CreatedAt": "2019-01-13T00:44:55Z",
      "UpdatedAt": "2019-01-13T00:44:55Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 19,
      "product_id": 6,
      "sell_price": 1000,
      "purchase_date": "2018-12-29T23:37:54Z"
  },
  {
      "ID": 11,
      "CreatedAt": "2019-01-14T06:03:08Z",
      "UpdatedAt": "2019-01-14T06:03:08Z",
      "DeletedAt": null,
      "img": "",
      "small_img": "",
      "imgalt": "",
      "price": 0,
      "promotion": 0,
      "productname": "",
      "Description": "",
      "name": "",
      "firstname": "",
      "lastname": "",
      "email": "",
      "password": "",
      "loggedin": false,
      "orders": null,
      "customer_id": 1,
      "product_id": 3,
      "sell_price": 17000,
      "purchase_date": "0001-01-01T00:00:00Z"
  }
]
`
  CUSTOMERS := `[
  {
      "ID": 1,
      "CreatedAt": "2018-08-14T07:52:54Z",
      "UpdatedAt": "2019-01-13T22:00:45Z",
      "DeletedAt": null,
      "name": "",
      "firstname": "Mal",
      "lastname": "Zein",
      "email": "mal.zein@email.com",
      "password": "$2a$10$ZeZI4pPPlQg89zfOOyQmiuKW9Z7pO9/KvG7OfdgjPAZF0Vz9D8fhC",
      "loggedin": true,
      "orders": null
  },
  {
      "ID": 2,
      "CreatedAt": "2018-08-14T07:52:55Z",
      "UpdatedAt": "2019-01-12T22:39:01Z",
      "DeletedAt": null,
      "name": "",
      "firstname": "River",
      "lastname": "Sam",
      "email": "river.sam@email.com",
      "password": "$2a$10$mNbCLmfCAc0.4crDg3V3fe0iO1yr03aRfE7Rr3vdfKMGVnnzovCZq",
      "loggedin": false,
      "orders": null
  },
  {
      "ID": 3,
      "CreatedAt": "2018-08-14T07:52:55Z",
      "UpdatedAt": "2019-01-13T21:56:05Z",
      "DeletedAt": null,
      "name": "",
      "firstname": "Jayne",
      "lastname": "Ra",
      "email": "jayne.ra@email.com",
      "password": "$2a$10$ZeZI4pPPlQg89zfOOyQmiuKW9Z7pO9/KvG7OfdgjPAZF0Vz9D8fhC",
      "loggedin": false,
      "orders": null
  },
  {
      "ID": 19,
      "CreatedAt": "2019-01-13T08:43:44Z",
      "UpdatedAt": "2019-01-13T15:12:25Z",
      "DeletedAt": null,
      "name": "",
      "firstname": "John",
      "lastname": "Doe",
      "email": "john.doe@bla.com",
      "password": "$2a$10$T4c8rmpbgKrUA0sIqtHCaO0g2XGWWxFY4IGWkkpVQOD/iuBrwKrZu",
      "loggedin": false,
      "orders": null
  }
]
`
  var products []models.Product
  var customers []models.Customer
  var orders []models.Order
  json.Unmarshal([]byte(PRODUCTS), &products)
  json.Unmarshal([]byte(CUSTOMERS), &customers)
  json.Unmarshal([]byte(ORDERS), &orders)
  return NewMockDBLayer(products, customers, orders)
}
```

前面的函数内部有一些硬编码的数据，这些数据随后被传递给`MockDBLayer`构造函数。这允许开发者使用`MockDBLayer`类型，这样他们就可以立即使用它，而无需首先想出数据。

接下来，我们需要提供方法来暴露`MockDBLayer`类型正在使用的数据：

```go
func (mock *MockDBLayer) GetMockProductData() []models.Product {
  return mock.products
}

func (mock *MockDBLayer) GetMockCustomersData() []models.Customer {
  return mock.customers
}

func (mock *MockDBLayer) GetMockOrdersData() []models.Order {
  return mock.orders
}
```

现在，我们需要一个方法，使我们能够完全控制`MockDBLayer`方法返回的错误。这很重要，因为在我们的单元测试中，我们很可能会需要测试如果发生错误，代码将如何表现。当我们开始编写单元测试时，我们将重新审视这个概念。现在，让我们编写一个方法，使我们能够设置由我们的模拟类型返回的错误：

```go
func (mock *MockDBLayer) SetError(err error) {
  mock.err = err
}
```

现在，是时候实现`DBLayer`接口的方法了。让我们从`GetAllProducts()`方法开始。它将看起来像这样：

```go
func (mock *MockDBLayer) GetAllProducts() ([]models.Product, error) {
  //Should we return an error?
  if mock.err != nil {
    return nil, mock.err
  }
  //return products list
  return mock.products, nil
}
```

我们首先需要检查的是`MockDBLayer`类型是否返回错误。如果需要返回错误，我们就直接返回错误。否则，我们返回我们保存在模拟类型中的产品列表。

接下来，让我们看看`GetPromos()`方法：

```go
func (mock *MockDBLayer) GetPromos() ([]models.Product, error) {
  if mock.err != nil {
    return nil, mock.err
  }
  promos := []models.Product{}
  for _, product := range mock.products {
    if product.Promotion > 0 {
      promos = append(promos, product)
    }
  }
  return promos, nil
}
```

在前面的代码中，我们首先检查是否应该返回错误，就像我们之前做的那样。然后我们遍历产品列表，选择具有促销的产品。

接下来，让我们探索`GetProduct(id)`方法。此方法应该能够根据提供的`id`检索产品。它看起来是这样的：

```go
func (mock *MockDBLayer) GetProduct(id int) (models.Product, error) {
  result := models.Product{}
  if mock.err != nil {
    return result, mock.err
  }
  for _, product := range mock.products {
    if product.ID == uint(id) {
      return product, nil
    }
  }
  return result, fmt.Errorf("Could not find product with id %d", id)
}
```

与其他方法一样，我们首先需要检查是否需要返回错误，如果是的话，我们就返回错误并退出方法。否则，我们检索此方法查询的数据。在`GetProduct(id)`的情况下，我们遍历产品列表，然后返回具有请求`id`的产品。如果我们把产品存储在列表中而不是映射中，这个循环可以被简单的映射检索所替代。在生产环境中，您需要决定您希望在模拟对象（映射和/或切片）中如何表示您的数据。在这种情况下，我决定为了简单起见使用切片。在更复杂的模拟对象中，您可能希望对于返回所有数据的函数存储数据在切片中，而对于返回特定项的函数存储数据在映射中。

模拟对象的其余代码将继续实现`DBLayer`接口方法。

这里是按名称获取客户的代码：

```go
func (mock *MockDBLayer) GetCustomerByName(first, last string) (models.Customer, error) {
  result := models.Customer{}
  if mock.err != nil {
    return result, mock.err
  }
  for _, customer := range mock.customers {
    if strings.EqualFold(customer.FirstName, first) && strings.EqualFold(customer.LastName, last) {
      return customer, nil
    }
  }
  return result, fmt.Errorf("Could not find user %s %s", first, last)
}
```

这里是按 ID 获取客户的代码：

```go

func (mock *MockDBLayer) GetCustomerByID(id int) (models.Customer, error) {
  result := models.Customer{}
  if mock.err != nil {
    return result, mock.err
  }

  for _, customer := range mock.customers {
    if customer.ID == uint(id) {
      return customer, nil
    }
  }
  return result, fmt.Errorf("Could not find user with id %d", id)
}
```

这里是添加用户的代码：

```go
func(mock *MockDBLayer) AddUser(customer models.Customer) (models.Customer, error){
  if mock.err != nil {
    return models.Customer{}, mock.err
  }
  mock.customers = append(mock.customers, customer)
  return customer, nil
}
```

这里是登录用户的代码：

```go
func (mock *MockDBLayer) SignInUser(email, password string) (models.Customer, error) {
  if mock.err != nil {
    return models.Customer{}, mock.err
  }
  for _, customer := range mock.customers {
    if strings.EqualFold(email, customer.Email) && customer.Pass == password {
      customer.LoggedIn = true
      return customer, nil
    }
  }
  return models.Customer{}, fmt.Errorf("Could not sign in user %s", email)
}
```

这里是按 ID 注销用户的代码：

```go
func (mock *MockDBLayer) SignOutUserById(id int) error {
  if mock.err != nil {
    return mock.err
  }
  for _, customer := range mock.customers {
    if customer.ID == uint(id) {
      customer.LoggedIn = false
      return nil
    }
  }
  return fmt.Errorf("Could not sign out user %d", id)
}
```

这里是按 ID 获取客户订单的代码：

```go
func (mock *MockDBLayer) GetCustomerOrdersByID(id int) ([]models.Order, error) {
  if mock.err != nil {
    return nil, mock.err
  }
  for _, customer := range mock.customers {
    if customer.ID == uint(id) {
      return customer.Orders, nil
    }
  }
  return nil, fmt.Errorf("Could not find customer id %d", id)
}
```

这里是添加订单的代码：

```go
func (mock *MockDBLayer) AddOrder(order models.Order) error {
  if mock.err != nil {
    return mock.err
  }
  mock.orders = append(mock.orders, order)
  for _, customer := range mock.customers {
    if customer.ID == uint(order.CustomerID) {
      customer.Orders = append(customer.Orders, order)
      return nil
    }
  }
  return fmt.Errorf("Could not find customer id %d for order", order.CustomerID)
}
```

最后，以下方法只是信用卡处理逻辑的占位符。在本章中我们将探讨的单元测试不会覆盖信用卡处理，为了简化问题；让我们现在就先将其作为占位符：

```go

//The credit card related mock methods will need more work. They are out of scope of this chapter.
func (mock *MockDBLayer) GetCreditCardCID(id int) (string, error) {
  if mock.err != nil {
    return "", mock.err
  }
  return "", nil
}

func (mock *MockDBLayer) SaveCreditCardForCustomer(int, string) error {
  if mock.err != nil {
    return mock.err
  }
  return nil
}
```

值得注意的是，在 Go 语言中存在一些第三方开源项目，可以帮助创建和使用模拟对象。然而，在本章中，我们已经构建了自己的。

现在我们已经创建了一个模拟数据库类型，让我们来谈谈 Go 语言中的单元测试。

# Go 语言中的单元测试

现在是时候探索 Go 语言中的单元测试并利用我们在上一节中构建的`MockDBLayer`类型了。

在 Go 语言中编写单元测试的第一步是在您想要测试的包所在的同一文件夹中创建一个新文件。文件名必须以`_test.go`结尾。在我们的例子中，由于我们想要测试`rest`包中的`GetProducts()`方法，我们将在同一文件夹中创建一个新文件，并将其命名为`handler_test.go`。

此文件仅在运行单元测试时构建和执行，而在常规构建期间不会执行。你可能想知道，我如何在 Go 中运行单元测试？答案是简单的——你使用`go test`命令！每次你运行`go test`时，只有以`_test.go`结尾的文件才会构建和运行。

如果你从不同于你想要测试的包的文件夹运行`go test`命令，那么你只需指向你想要测试的包：

```go
go test <Your_Package_Path>
```

例如，如果我们想运行`rest`包的单元测试，命令将看起来像这样：

```go
go test github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/tree/master/Chapter08/backend/src/rest
```

现在，让我们进一步深入到`handler_test.go`文件。我们需要做的第一件事是声明包：

```go
package rest
```

接下来，我们需要编写一个表示我们的单元测试的函数。在 Go 中，你需要遵循一些特定的规则来确保你的函数作为`go test`命令产生的单元测试的一部分执行：

+   你的函数必须以单词`Test`开头

+   在`Test`之后的首字母必须大写

+   函数需要以`*testing.T`类型作为参数

`*testing.T`类型提供了一些重要的方法，将帮助我们表明测试是否失败或通过。该类型还提供了一些我们可以使用的日志功能。当我们开始编写测试代码时，我们将很快看到`*testing.T`类型的实际应用。

因此，遵循前面的三个规则，我们将创建一个名为`TestHandler_GetProducts`的新函数，用于托管我们 HTTP 处理器中`GetProducts()`方法的单元测试代码。这个函数将看起来像这样：

```go
func TestHandler_GetProducts(t *testing.T) {
}
```

我们需要做的第一件事是启用 Gin 框架的测试模式。Gin 框架的测试模式可以防止过多的日志记录：

```go
func TestHandler_GetProducts(t *testing.T) {
  // Switch to test mode so you don't get such noisy output
  gin.SetMode(gin.TestMode)
}
```

接下来，让我们初始化我们的`mockdbLayer`类型。我们将使用包含一些硬编码数据的构造函数：

```go
func TestHandler_GetProducts(t *testing.T) {
  // Switch to test mode so you don't get such noisy output
  gin.SetMode(gin.TestMode)
  mockdbLayer := dblayer.NewMockDBLayerWithData()
}
```

我们在本节中寻求测试的`GetProducts()`方法是一个 HTTP 处理器函数，它预期返回 GoMusic 商店可用的产品列表。正如我们在前面的章节中所述，HTTP 处理器函数可以定义为当向特定的相对 URL 发送 HTTP 请求时执行的操作。HTTP 处理器将处理 HTTP 请求并通过 HTTP 返回响应。

在我们的单元测试中，我们需要测试`GetProducts()`作为方法的功能，以及它对 HTTP 请求的反应。

我们需要定义一个相对 URL，该 URL 将激活该 HTTP 处理器函数。让我们称它为`/products`并将其设为一个常量：

```go
func TestHandler_GetProducts(t *testing.T) {
  // Switch to test mode so you don't get such noisy output
  gin.SetMode(gin.TestMode)
  mockdbLayer := dblayer.NewMockDBLayerWithData()
  h := NewHandlerWithDB(mockdbLayer)
 const productsURL string = "/products"
}
```

我们将在下一章中看到如何利用这个常数。

在下一节中，我们将介绍一个称为表驱动开发的重要概念。

# 表驱动开发

在典型的实际单元测试中，我们试图测试某个函数或方法，看看它将如何响应某些输入和错误条件。这意味着单元测试的代码需要多次调用我们试图测试的函数或方法，并使用不同的输入和错误条件。而不是编写大量的大规模`if`语句来使用不同的输入进行调用，我们可以遵循一个非常流行的设计模式，称为**表驱动开发**。

测试驱动开发背后的思想很简单——我们将使用 Go 结构体的数组或映射来表示我们的测试。结构体数组或映射将包含我们想要传递给正在测试的函数/方法的输入和错误条件的信息。然后我们将遍历 Go 结构体数组并调用当前输入和错误条件下的待测试方法/函数。这种方法将在主单元测试下产生多个子测试。

在我们的单元测试中，我们将使用 Go 结构体的数组来表示我们的不同子测试。以下是数组的结构：

```go
tests := []struct {
    name string
    inErr error
    outStatusCode int
    expectedRespBody interface{}
  }{
    }
```

让我们把前面的代码称为我们的`test`表。以下是 Go 结构字段将表示的内容：

+   `name`: 这是子测试的名称。

+   `inErr`: 这是输入错误。我们将把这个错误注入到模拟数据库层类型中，并监控`GetProducts()`方法的行为。

+   `outStatusCode`: 这是调用`GetProducts()` HTTP 处理器产生的预期 HTTP 状态码。如果从调用`GetProducts()`作为 HTTP 处理器返回的 HTTP 状态码不匹配此值，则单元测试失败。

+   `expectedRespBody`: 这是从调用`GetProducts()` HTTP 处理器返回的预期 HTTP 响应体。如果返回的 HTTP 体不匹配此值，则单元测试失败。该字段是`interface{}`类型，因为它可以是产品切片或错误消息。Go 语言中的`interface{}`类型可以表示任何其他数据类型。

表驱动测试设计模式的美丽之处在于它的灵活性；你可以简单地添加更多字段来测试更多条件。

从`GetProducts()` HTTP 处理器可以产生两个预期的 HTTP 响应体——我们要么得到一些产品列表，要么得到一个错误消息。错误消息采用以下格式：`{error:"the error message"}`。

在我们从测试表中开始运行子测试之前，让我们定义一个`struct`类型来表示错误消息，这样我们就可以在测试中使用它：

```go
 type errMSG struct {
    Error string `json:"error"`
  }
```

接下来，我们需要定义我们的子测试列表。为了简单起见，我们将选择两种不同的场景进行测试：

```go
 tests := []struct {
    name string
    inErr error
    outStatusCode int
    expectedRespBody interface{}
  }{
 {
 "getproductsnoerrors",
 nil,
 http.StatusOK,
 mockdbLayer.GetMockProductData(),
 },
 {
 "getproductswitherror",
 errors.New("get products error"),
 http.StatusInternalServerError,
 errMSG{Error: "get products error"},
 },
  }
```

在前面的代码中，我们定义了两个不同的子测试：

+   第一个方法称为`getproductsnoerrors`。它代表一个直接执行场景，其中没有发生错误，一切正常。我们没有向模拟数据库层类型注入任何错误，因此我们预期`GetProducts()`方法不会返回任何错误。我们预期 HTTP 响应状态为`OK`，并且我们预期将获得存储在模拟数据库层中的产品数据列表作为 HTTP 响应体。我们之所以预期获得存储在模拟数据库层中的产品数据列表作为我们的输出，是因为模拟数据库类型将是我们的`GetProducts()`方法的数据库层。

+   第二个方法称为`getproductswitherror`。它代表一个发生错误的执行场景。我们将一个名为`"get products error"`的错误注入到模拟数据库层类型中。这个错误预期将作为`GetProducts()`处理函数调用的 HTTP 响应体返回。预期的 HTTP 状态码将是`StatusInternalServerError`。

我们单元测试代码的其余部分将遍历测试表并执行我们的测试。`*testing.T`类型，它作为参数传递给我们的单元测试函数，提供了我们可以用来在单元测试中定义子测试的方法，然后我们可以并行运行这些子测试。

首先，为了在我们的单元测试中定义一个子测试，我们必须使用以下方法：

```go
t.Run(name string,f func(t *T))bool
```

第一个参数是子测试的名称，而第二个参数是一个表示我们想要运行的子测试的函数。在我们的情况下，我们需要遍历我们的`test`表并对每个子测试调用`t.Run()`。下面是这个过程的示例：

```go
 for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        //run our sub-test
    }
}
```

现在，让我们专注于运行子测试的代码。我们首先需要做的是将一个错误注入到模拟类型中：

```go
 for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      //set the input error
      mockdbLayer.SetError(tt.inErr)
    }
}
```

接下来，我们需要创建一个测试 HTTP 请求来表示将被我们的`GetProducts()` HTTP 处理程序接收到的 HTTP 请求。同样，Go 通过一个名为`httptest`的标准包来提供帮助。这个包使你能够创建特殊的数据类型，允许你测试与 HTTP 相关的功能。`httptest`提供的一个函数是名为`NewRequest()`的函数，它返回一个我们可以用于测试的 HTTP 请求类型：

```go
//Create a test request
req := httptest.NewRequest(http.MethodGet, productsURL, nil)    
```

函数接受三个参数：HTTP 方法的类型、预期发送 HTTP 请求的相对 URL，以及请求体（如果有）。

在`GetProducts()` HTTP `handler`方法的案例中，它期望一个针对`/products`相对 URL 的 HTTP `GET`请求。我们已经在`productsURL`常量中存储了`/products`值。

`httptest` 包还提供了一个名为 `ResponseRecorder` 的数据类型，可以用来捕获 HTTP 处理函数调用的 HTTP 响应。`ResponseRecorder` 数据类型实现了 Go 的 `http.ResponseWriter` 接口，这使得 `ResponseRecorder` 可以注入到任何使用 `http.ResponseWriter` 的代码中。我们需要获取这个数据类型的一个值，以便在测试中使用它：

```go
//create an http response recorder
w := httptest.NewRecorder()
```

接下来，我们需要创建 Gin 框架引擎的一个实例，以便在我们的测试中使用。这是因为我们试图测试的 `GetProducts()` 方法是一个 Gin 引擎路由器的 HTTP 处理函数，所以它需要一个 `*gin.Context` 类型的输入。下面是这个函数签名的样子：

```go
GetProducts(c *gin.Context)
```

幸运的是，Gin 框架已经为我们准备了一个名为 `CreateTestContext()` 的函数，专门用于创建 Gin 上下文实例和 Gin 引擎实例，以便在测试中使用。`CreateTestContext()` 函数接受一个 `http.ResponseWriter` 接口作为输入，这意味着我们可以传递我们的 `httptest.ResponseRecorder` 作为输入，因为它实现了 `http.ResponseWriter` 接口。下面是代码的示例：

```go
//create an http response recorder  
w := httptest.NewRecorder()

//create a fresh gin engine object from the response recorder, we will ignore the context value
_, engine := gin.CreateTestContext(w)
```

如我们之前提到的，`CreateTestContext()` 函数返回两个值：一个 Gin 上下文实例和一个 Gin 引擎实例。在我们的情况下，我们不会使用 Gin 上下文实例，这就是为什么在之前的代码中没有接收到它的值。我们不使用 Gin 上下文实例的原因是我更喜欢使用 Gin 引擎实例进行测试，因为它允许我测试 HTTP 请求被完全处理的工作流程。

接下来，我们将使用 Gin 引擎实例将我们的 `GetProducts()` 方法映射到 `productsURL` 相对 URL 地址，通过一个 HTTP `GET` 请求。下面是这个过程的示例：

```go
//configure the get request
 engine.GET(productsURL, h.GetProducts)
```

现在，是时候让我们的 Gin 引擎处理 HTTP 请求，并将 HTTP 响应传递给我们的 `ResponseRecorder`：

```go
//serve the request
 engine.ServeHTTP(w, req)
```

这实际上会将我们的测试 HTTP 请求发送到 `GetProducts()` 处理方法，因为测试请求的目标是 `productsURL`。然后，`GetProducts()` 处理方法将处理请求并通过 `w`（我们的 `ResponseRecorder`）发送 HTTP 响应。

现在是时候测试 `GetProducts()` 如何处理 HTTP 请求了。我们首先需要做的是从 `w` 中提取 HTTP 响应。这是通过 `ResponseRecorder` 对象类型的 `Result()` 方法完成的：

```go
//test the output
response := w.Result()
```

然后，我们需要测试结果的 HTTP 状态码。如果它不等于预期的 HTTP 状态码，那么我们将失败测试用例，并记录失败的原因：

```go
if response.StatusCode != tt.outStatusCode {
        t.Errorf("Received Status code %d does not match expected status code %d", response.StatusCode, tt.outStatusCode)
 }
```

如前述代码所示，`*testing.T`类型自带一个名为`Errorf`的方法，可以用来记录消息然后失败测试。如果我们想要记录消息而不失败测试，我们可以使用名为`Logf`的方法。如果我们想要立即失败测试，我们可以调用名为`Fail`的方法。`t.Errorf`方法是`t.Logf`和`t.Fail`的组合。

接下来，我们需要捕获响应的 HTTP 正文，以便我们能够将其与预期此子测试的 HTTP 响应正文进行比较。有两种情况需要考虑：要么在我们的子测试中注入了错误，这意味着预期结果是错误消息，要么没有注入错误，这意味着预期 HTTP 响应正文是一个产品列表：

```go
/*
Since we don't know the data type to expect from the http response, we'll use interface{} as the type 
*/      
var respBody interface{}      
//If an error was injected, then the response should decode to an error message type     
 if tt.inErr != nil {
        var errmsg errMSG
        json.NewDecoder(response.Body).Decode(&errmsg)
        //Assign decoded error message to respBody
        respBody = errmsg      
} else {
        //If an error was not injected, the response should decode to a slice of products data types
        products := []models.Product{}
        json.NewDecoder(response.Body).Decode(&products)
        //Assign decoded products list to respBody
        respBody = products      
}
```

我们最后需要做的事情是将预期的 HTTP 响应正文与实际收到的 HTTP 响应正文进行比较。要进行比较，我们需要利用 Go 的`reflect`包中的一个非常有用的函数。这个函数叫做`reflect.DeepEqual()`，它帮助我们完全比较两个值并确定它们是否是彼此的副本。如果两个值不相等，那么我们将记录一个错误并失败测试。以下是代码的样子：

```go
if !reflect.DeepEqual(respBody, tt.expectedRespBody) {
       t.Errorf("Received HTTP response body %+v does not match expected HTTP response Body %+v", respBody, tt.expectedRespBody)
 }
```

这样，我们的单元测试就完成了！让我们看看整体测试代码将是什么样子：

```go
func TestHandler_GetProducts(t *testing.T) {
  // Switch to test mode so you don't get such noisy output
  gin.SetMode(gin.TestMode)
  mockdbLayer := dblayer.NewMockDBLayerWithData()
  h := NewHandlerWithDB(mockdbLayer)
  const productsURL string = "/products"
  type errMSG struct {
    Error string `json:"error"`
  }
 // Use table driven testing
  tests := []struct {
    name string
    inErr error
    outStatusCode int
    expectedRespBody interface{}
  }{
    {
      "getproductsnoerrors",
      nil,
      http.StatusOK,
      mockdbLayer.GetMockProductData(),
    },
    {
      "getproductswitherror",
      errors.New("get products error"),
      http.StatusInternalServerError,
      errMSG{Error: "get products error"},
    },
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      //set the input error
      mockdbLayer.SetError(tt.inErr)
      //Create a test request
      req := httptest.NewRequest(http.MethodGet, productsURL, nil)
      //create an http response recorder
      w := httptest.NewRecorder()
      //create a fresh gin context and gin engine object from the response recorder
      _, engine := gin.CreateTestContext(w)

      //configure the get request
      engine.GET(productsURL, h.GetProducts)
      //serve the request
      engine.ServeHTTP(w, req)

      //test the output
      response := w.Result()
      if response.StatusCode != tt.outStatusCode {
        t.Errorf("Received Status code %d does not match expected status code %d", response.StatusCode, tt.outStatusCode)
      }
      //Since we don't know the data type to expect from the http response, we'll use interface{} as the type 
      var respBody interface{}
      //If an error was injected, then the response should decode to an error message type
      if tt.inErr != nil {
        var errmsg errMSG
        json.NewDecoder(response.Body).Decode(&errmsg)
        respBody = errmsg
      } else {
        //If an error was not injected, the response should decode to a slice of products data types
        products := []models.Product{}
        json.NewDecoder(response.Body).Decode(&products)
        respBody = products
      }

      if !reflect.DeepEqual(respBody, tt.expectedRespBody) {
        t.Errorf("Received HTTP response body %+v does not match expected HTTP response Body %+v", respBody, tt.expectedRespBody)
      }
    })
  }
}
```

值得注意的是，Go 给你提供了运行你的子测试相互并行的能力。你可以在子测试中调用`t.Parallel()`来触发这种行为。以下是它的样子：

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            //your concurrent code
    }
}
```

然而，当你运行并发子测试时，你必须确保它们共享的任何数据类型在并行时不会改变状态或行为，否则你的测试结果将不可靠。例如，在我们的代码中，我们使用了一个单独的模拟数据库层类型对象，该对象是在子测试外部初始化的。这意味着每次我们在子测试中更改模拟数据库层的错误状态时，它可能会影响同时运行的其它子测试，并使用相同的模拟数据库层对象。

本节剩余的任务是运行我们的单元测试并见证结果。正如我们之前提到的，你可以从包含你想要测试的包的文件夹内部运行`go test`命令，或者你可以从你的包文件夹外部使用`go test <your_package_path>`命令。如果单元测试通过，你将看到类似以下的输出：

```go
PASS
ok github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/tree/master/Chapter08/backend/src/rest 0.891s
```

默认输出显示了被测试的包的完整名称以及运行测试所需的时间。

如果你想看到更多信息，你可以运行`go test -v`。这个命令将返回以下内容：

```go
c:\Programming_Projects\GoProjects\src\github.com\PacktPublishing\Hands-On-Full-Stack-Development-with-Go\8-Testing-and-benchmarking\backend\src\rest>go test -v
=== RUN TestHandler_GetProducts
=== RUN TestHandler_GetProducts/getproductsnoerrors
=== RUN TestHandler_GetProducts/getproductswitherror
--- PASS: TestHandler_GetProducts (0.00s)
 --- PASS: TestHandler_GetProducts/getproductsnoerrors (0.00s)
 --- PASS: TestHandler_GetProducts/getproductswitherror (0.00s)
PASS
ok github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/8-Testing-and-benchmarking/backend/src/rest 1.083s
```

`-v`标志显示详细输出——它会显示正在运行的单元测试的名称，以及单元测试中正在运行的子测试的名称。

让我们看看下一节的基准测试。

# 基准测试

在测试软件的世界中，另一个关键主题是基准测试。**基准测试**是衡量你代码性能的实践。Go 的 `testing` 包为你提供了执行强大基准测试的能力。

让我们先针对一段代码，展示如何使用 Go 进行基准测试。一个很好的基准测试函数是 `hashpassword()` 函数，它被我们的数据库层使用。这个函数可以在 `backend/src/dblayer/orm.go` 文件中找到。它接受一个字符串的引用作为参数，然后使用 `bcrypt` 散列来散列字符串。以下是代码：

```go
func hashPassword(s *string) error {
  if s == nil {
    return errors.New("Reference provided for hashing password is nil")
  }
  //converd password string to byte slice
  sBytes := []byte(*s)
  //Obtain hashed password
  hashedBytes, err := bcrypt.GenerateFromPassword(sBytes, bcrypt.DefaultCost)
  if err != nil {
    return err
  }
  //update password string with the hashed version
  *s = string(hashedBytes[:])
  return nil
}
```

假设我们想要测试这个函数的性能。我们应该如何开始？

第一步是创建一个新文件。文件名应以 `_test.go` 结尾。该文件需要存在于与 `dblayer` 包相同的文件夹中，该包是我们想要测试的函数所在的位置。让我们称这个文件为 `orm_test.go`。

如我们之前提到的，以 `_test.go` 结尾的文件将不会成为常规构建过程的一部分。相反，它们会在我们通过 `go test` 命令运行测试时激活。

接下来，在文件内部，我们首先声明该文件所属的 Go 包，即 `dblayer`。然后，我们需要导入我们将在代码中使用的测试包：

```go
package dblayer

import "testing"
```

现在，是时候编写基准测试 `hashpassword()` 函数的代码了。要在 Go 中编写基准函数，我们需要遵循以下规则：

+   函数名必须以单词 `Benchmark` 开头。

+   在单词 `Benchmark` 之后的第一封信需要大写。

+   该函数接受 `*testing.B` 作为参数。`*testing.B` 类型提供了便于基准测试我们代码的方法。

遵循这三个规则，我们将使用以下签名构建基准函数：

```go
func BenchmarkHashPassword(b *testing.B) {
}
```

接下来，我们将初始化一个要散列的字符串：

```go
func BenchmarkHashPassword(b *testing.B) {
  text := "A String to be Hashed"
}
```

要使用类型为 `*testing.B` 的对象来基准测试一段代码，我们需要运行目标代码 `b.N` 次。`N` 是 `*testing.B` 类型中的一个字段，它会调整其值，直到目标代码可以被可靠地测量。代码将如下所示：

```go
func BenchmarkHashPassword(b *testing.B) {
 text := "A String to be Hashed"
 for i := 0; i < b.N; i++ {
 hashPassword(&text)
 }
}
```

前面的代码将运行 `hashPassword()`，直到它被基准测试。要运行基准测试，我们可以使用带有 `-bench` 标志的 `go test` 命令：

```go
go test -bench .
```

`-bench` 标志需要提供一个正则表达式，以指示我们想要运行的基准函数。如果我们想运行所有可用的内容，我们可以使用 `.` 来表示所有。否则，如果我们只想运行包含 `HashPassword` 术语的基准测试，我们可以修改命令，如下所示：

```go
go test -bench HashPassword
```

输出将如下所示：

```go
goos: windows
goarch: amd64
pkg: github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/8-Testing-and-benchmarking/backend/src/dblayer
BenchmarkHashPassword-8 20 69609530 ns/op
PASS
ok github.com/PacktPublishing/Hands-On-Full-Stack-Development-with-Go/8-Testing-and-benchmarking/backend/src/dblayer 1.797s
```

输出简单地说明 `hashPassword` 函数运行了 20 次，并且它的速度大约是每循环 69,609,530 纳秒。

在我们的例子中，我们在运行基准测试之前只初始化了一个字符串，这是一个非常直接且简单的操作：

```go
func BenchmarkHashPassword(b *testing.B) {
 text := "A String to be Hashed"
  for i := 0; i < b.N; i++ {

    hashPassword(&text)
  }
}
```

然而，如果你的初始化非常复杂并且需要一些时间来完成，建议你在完成初始化后、进行基准测试之前运行 `b.ResetTimer()`。以下是一个示例：

```go
func BenchMarkSomeFunction(b *testing.B){
    someHeavyInitialization()
    b.ResetTimer()
    for i:=0;i<b.N;b++{
        SomeFunction()
    }
}
b.Run(name string, f func(b *testing.B))
```

`*testing.B` 类型还附带一个名为 `RunParallel()` 的额外方法，可以在并行设置中测试性能。这与一个名为 `go test -cpu` 的标志协同工作：

```go
b.RunParallel(func(pb *testing.PB){
    for pb.Next(){
        //your code
    }
})
```

# 概述

本章介绍了任何软件开发者都应该具备的关键技能，即在生产中进行适当的软件测试。我们关注了 Go 语言提供的功能，以支持 Go 代码的测试。

我们首先介绍了如何构建模拟类型以及为什么它们在软件测试中很重要。然后我们介绍了如何在 Go 中进行单元测试以及如何对你的软件进行基准测试。

在下一章中，我们将通过介绍 GopherJS 框架来探索同构 Go 编程的概念。

# 问题

1.  模拟类型的定义是什么？为什么它对软件测试有用？

1.  Go 中的测试包是什么？

1.  `*testing.T` 类型用于什么？

1.  `*testing.B` 类型用于什么？

1.  `*testing.T.Run()` 方法用于什么？

1.  `*testing.T.Parallel()` 方法用于什么？

1.  基准测试是什么意思？

1.  `*testing.B.ResetTimer()` 方法用于什么？

# 进一步阅读

如需更多信息，您可以查阅以下链接：

+   **Go 测试包**：[`golang.org/pkg/testing/`](https://golang.org/pkg/testing/)
