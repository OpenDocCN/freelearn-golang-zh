# 第八章：部署 Goophr

在第六章中，*Goophr Concierge*和第七章中，*Goophr Librarian*，我们构建了 Goophr 的两个组件：Concierge 和 Librarian。我们花时间了解了每个组件设计背后的原理，以及它们如何预期一起工作。

在本章中，我们将通过实现以下目标来完成 Goophr 的构建：

+   更新`concierge/api/query.go`，以便 Concierge 可以查询多个 Librarian 实例的搜索词

+   更新`docker-compose.yaml`，以便我们可以轻松运行完整的 Goophr 系统

+   通过向索引添加文档并通过 REST API 查询索引来测试设置

## 更新 Goophr Concierge

为了使 Concierge 按照 Goophr 的设计完全功能，我们需要执行以下操作：

+   从多个 Librarian 请求搜索结果

+   对组合搜索结果进行排名

让我们详细讨论这些要点。

### 处理多个 Librarian

Goophr Librarian 的核心功能是更新索引并根据搜索词返回相关的`DocID`。正如我们在实现 Librarian 的代码库时所看到的，我们需要更新索引，检索相关的`DocID`，然后根据相关性对其进行排序，然后返回查询结果。涉及许多操作，并且在查找和更新时使用了许多映射。这些操作可能看起来微不足道。然而，随着查找表（映射）的大小增加，查找表上的操作性能将开始下降。为了避免性能下降，可以采取许多方法。

我们的主要目标是在 Go 的上下文中理解分布式系统，因此，我们将拆分 Librarian 以仅处理一定范围的索引。分区是数据库中使用的标准技术之一，其中数据库被分成多个分区。在我们的情况下，我们将运行三个 Librarian 实例，每个实例负责处理分配给每个分区的字符范围内的所有令牌的索引：

+   `a_m_librarian`：负责以字符“A”到“M”开头的令牌的图书管理员

+   `n_z_librarian`：负责以字符“N”到“Z”开头的令牌的图书管理员

+   `others_librarian`：负责以数字开头的令牌的图书管理员

### 聚合搜索结果

下一步将是从多个 Librarian 实例聚合搜索词的结果，并将它们作为有效载荷返回给查询请求。这将要求我们执行以下操作：

+   获取所有可用图书管理员的 URL 列表

+   在接收到查询时从所有 Librarian 请求搜索结果

+   根据`DocID`聚合搜索结果

+   按相关性分数降序排序结果

+   根据 Swagger API 定义形成并返回 JSON 有效载荷

现在我们了解了拥有多个 Librarian 实例的原因，以及我们将如何根据这个新配置处理查询，我们可以将这些更改应用到`concierge/api/query.go`中。

## 使用 docker-compose 进行编排

我们一直在我们系统的 localhost 上以硬编码的网络端口值运行 Librarian 和 Concierge 的服务器。到目前为止，我们还没有遇到任何问题。然而，当我们考虑到我们将运行三个 Librarian 实例，需要连接所有这些实例到 Concierge 并且能够轻松地启动和监视服务器时，我们意识到有很多移动部分。这可能导致在操作系统时出现不必要的错误。为了让我们的生活变得更轻松，我们可以依赖于`docker-compose`，它将为我们处理所有这些复杂性。我们所要做的就是定义一个名为`docker-compose.yaml`的配置 YAML 文件，其中包含以下信息：

+   确定我们想要一起运行的服务

+   在 YAML 文件中为每个服务定义的相应的 Dockerfile 或 Docker 镜像的位置或名称，以便我们可以为所有这些服务构建 Docker 镜像并将它们作为容器运行

+   要为每个正在运行的容器公开的端口

+   我们可能想要注入到我们的服务器实例中的任何其他环境变量

+   确保 Concierge 容器可以访问所有其他正在运行的容器

### 环境变量和 API 端口

我们提到我们将在`docker-compose.yaml`中指定我们希望每个容器运行的端口。但是，我们还需要更新`{concierge,librarian}/main.go`，以便它们可以在环境变量定义的端口上启动服务器。我们还需要更新`concierge/query.go`，以便它可以访问由`docker-compose`定义的 URL 和端口上的 Librarian 实例。

### 文件服务器

为了通过将文档加载到索引中快速测试我们的设置，以便能够查询系统并验证查询结果，我们还将包括一个简单的 HTTP 服务器，用于提供包含几个单词的文档。

## Goophr 源代码

在前两章中，第六章 *Goophr Concierge* 和 第七章 *Goophr Librarian*，我们分别讨论了 Concierge 和 Librarian 的代码。为了使用`docker-compose`运行完整的 Goophr 应用程序，我们需要将 Librarian 和 Concierge 的代码库合并为一个单一的代码库。代码库还将包括`docker-compose.yaml`和文件服务器的代码。

在本章中，我们不会列出 Librarian 和 Concierge 中所有文件的代码，而只列出有更改的文件。让我们先看一下完整项目的结构：

```go
$ tree -a
.
ε2;── goophr
 ├── concierge
 │ ├── api
 │ │ ├── feeder.go
 │ │ ├── feeder_test.go
 │ │ └── query.go
 │ ├── common
 │ │ └── helpers.go
 │ ├── Dockerfile
 │ └── main.go
 ├── docker-compose.yaml
 ├── .env
 ├── librarian
 │ ├── api
 │ │ ├── index.go
 │ │ └── query.go
 │ ├── common
 │ │ └── helpers.go
 │ ├── Dockerfile
 │ └── main.go
 └── simple-server
 ├── Dockerfile
 └── main.go

8 directories, 15 files
```

### librarian/main.go

我们希望允许 Librarian 根据传递给它的环境变量`API_PORT`在自定义端口上启动：

```go
package main 

import ( 
    "fmt" 
    "net/http" 
    "os" 

    "github.com/last-ent/distributed-go/chapter8/goophr/librarian/api" 
    "github.com/last-ent/distributed-go/chapter8/goophr/librarian/common" 
) 

func main() { 
    common.Log("Adding API handlers...") 
    http.HandleFunc("/api/index", api.IndexHandler) 
    http.HandleFunc("/api/query", api.QueryHandler) 

    common.Log("Starting index...") 
    api.StartIndexSystem() 

    port := fmt.Sprintf(":%s", os.Getenv("API_PORT")) 
    common.Log(fmt.Sprintf("Starting Goophr Librarian server on port %s...", port)) 
    http.ListenAndServe(port, nil) 
} 
```

### concierge/main.go

允许 Concierge 根据传递给它的环境变量`API_PORT`在自定义端口上启动：

```go
package main 

import ( 
    "fmt" 
    "net/http" 
    "os" 

    "github.com/last-ent/distributed-go/chapter8/goophr/concierge/api" 
    "github.com/last-ent/distributed-go/chapter8/goophr/concierge/common" 
) 

func main() { 
    common.Log("Adding API handlers...") 
    http.HandleFunc("/api/feeder", api.FeedHandler) 
    http.HandleFunc("/api/query", api.QueryHandler) 

    common.Log("Starting feeder...") 
    api.StartFeederSystem() 

    port := fmt.Sprintf(":%s", os.Getenv("API_PORT")) 
    common.Log(fmt.Sprintf("Starting Goophr Concierge server on port %s...", port)) 
    http.ListenAndServe(port, nil) 
} 
```

### concierge/api/query.go

查询所有可用的 Librarian 实例以检索搜索查询结果，按顺序对其进行排名，然后将结果发送回去：

```go
package api 

import ( 
    "bytes" 
    "encoding/json" 
    "fmt" 
    "io" 
    "io/ioutil" 
    "log" 
    "net/http" 
    "os" 
    "sort" 

    "github.com/last-ent/distributed-go/chapter8/goophr/concierge/common" 
) 

var librarianEndpoints = map[string]string{} 

func init() { 
    librarianEndpoints["a-m"] = os.Getenv("LIB_A_M") 
    librarianEndpoints["n-z"] = os.Getenv("LIB_N_Z") 
    librarianEndpoints["*"] = os.Getenv("LIB_OTHERS") 
} 

type docs struct { 
    DocID string 'json:"doc_id"' 
    Score int    'json:"doc_score"' 
} 

type queryResult struct { 
    Count int    'json:"count"' 
    Data  []docs 'json:"data"' 
} 

func queryLibrarian(endpoint string, stBytes io.Reader, ch chan<- queryResult) { 
    resp, err := http.Post( 
        endpoint+"/query", 
        "application/json", 
        stBytes, 
    ) 
    if err != nil { 
        common.Warn(fmt.Sprintf("%s -> %+v", endpoint, err)) 
        ch <- queryResult{} 
        return 
    } 
    body, _ := ioutil.ReadAll(resp.Body) 
    defer resp.Body.Close() 

    var qr queryResult 
    json.Unmarshal(body, &qr) 
    log.Println(fmt.Sprintf("%s -> %#v", endpoint, qr)) 
    ch <- qr 
} 

func getResultsMap(ch <-chan queryResult) map[string]int { 
    results := []docs{} 
    for range librarianEndpoints { 
        if result := <-ch; result.Count > 0 { 
            results = append(results, result.Data...) 
        } 
    } 

    resultsMap := map[string]int{} 
    for _, doc := range results { 
            docID := doc.DocID 
            score := doc.Score 
            if _, exists := resultsMap[docID]; !exists { 
                resultsMap[docID] = 0 
            } 
            resultsMap[docID] = resultsMap[docID] + score 
        } 

    return resultsMap 
} 

func QueryHandler(w http.ResponseWriter, r *http.Request) { 
    if r.Method != "POST" { 
        w.WriteHeader(http.StatusMethodNotAllowed) 
        w.Write([]byte('{"code": 405, "msg": "Method Not Allowed."}')) 
        return 
    } 

    decoder := json.NewDecoder(r.Body) 
    defer r.Body.Close() 

    var searchTerms []string 
    if err := decoder.Decode(&searchTerms); err != nil { 
        common.Warn("Unable to parse request." + err.Error()) 

        w.WriteHeader(http.StatusBadRequest) 
        w.Write([]byte('{"code": 400, "msg": "Unable to parse payload."}')) 
        return 
    } 

    st, err := json.Marshal(searchTerms) 
    if err != nil { 
        panic(err) 
    } 
    stBytes := bytes.NewBuffer(st) 

    resultsCh := make(chan queryResult) 

    for _, le := range librarianEndpoints { 
        func(endpoint string) { 
            go queryLibrarian(endpoint, stBytes, resultsCh) 
        }(le) 
    } 

    resultsMap := getResultsMap(resultsCh) 
    close(resultsCh) 

    sortedResults := sortResults(resultsMap) 

    payload, _ := json.Marshal(sortedResults) 
    w.Header().Add("Content-Type", "application/json") 
    w.Write(payload) 

    fmt.Printf("%#v\n", sortedResults) 
} 

func sortResults(rm map[string]int) []document { 
    scoreMap := map[int][]document{} 
    ch := make(chan document) 

    for docID, score := range rm { 
        if _, exists := scoreMap[score]; !exists { 
            scoreMap[score] = []document{} 
        } 

        dGetCh <- dMsg{ 
            DocID: docID, 
            Ch:    ch, 
        } 
        doc := <-ch 

        scoreMap[score] = append(scoreMap[score], doc) 
    } 

    close(ch) 

    scores := []int{} 
    for score := range scoreMap { 
        scores = append(scores, score) 
    } 
    sort.Sort(sort.Reverse(sort.IntSlice(scores))) 

    sortedResults := []document{} 
    for _, score := range scores { 
        resDocs := scoreMap[score] 
        sortedResults = append(sortedResults, resDocs...) 
    } 
    return sortedResults 
} 
```

### simple-server/Dockerfile

让我们使用`Dockerfile`来创建一个简单的文件服务器：

```go
FROM golang:1.10 

ADD . /go/src/littlefs 

WORKDIR /go/src/littlefs 

RUN go install littlefs 

ENTRYPOINT /go/bin/littlefs
```

### simple-server/main.go

让我们来看一个简单的程序，根据`bookID`返回一组单词作为 HTTP 响应：

```go
package main 

import ( 
    "log" 
    "net/http" 
) 

func reqHandler(w http.ResponseWriter, r *http.Request) { 
    books := map[string]string{ 
        "book1": 'apple apple cat zebra', 
        "book2": 'banana cake zebra', 
        "book3": 'apple cake cake whale', 
    } 

    bookID := r.URL.Path[1:] 
    book, _ := books[bookID] 
    w.Write([]byte(book)) 
} 

func main() { 

    log.Println("Starting File Server on Port :9876...") 
    http.HandleFunc("/", reqHandler) 
    http.ListenAndServe(":9876", nil) 
} 
```

### docker-compose.yaml

该文件将允许我们从单个界面构建、运行、连接和停止我们的容器。

```go
version: '3' 

services: 
  a_m_librarian: 
    build: librarian/. 
    environment: 
      - API_PORT=${A_M_PORT} 
    ports: 
      - ${A_M_PORT}:${A_M_PORT} 
  n_z_librarian: 
      build: librarian/. 
      environment: 
        - API_PORT=${N_Z_PORT} 
      ports: 
        - ${N_Z_PORT}:${N_Z_PORT} 
  others_librarian: 
      build: librarian/. 
      environment: 
        - API_PORT=${OTHERS_PORT} 
      ports: 
        - ${OTHERS_PORT}:${OTHERS_PORT} 
  concierge: 
    build: concierge/. 
    environment: 
      - API_PORT=${CONCIERGE_PORT} 
      - LIB_A_M=http://a_m_librarian:${A_M_PORT}/api 
      - LIB_N_Z=http://n_z_librarian:${N_Z_PORT}/api 
      - LIB_OTHERS=http://others_librarian:${OTHERS_PORT}/api 
    ports: 
      - ${CONCIERGE_PORT}:${CONCIERGE_PORT} 
    links: 
      - a_m_librarian 
      - n_z_librarian 
      - others_librarian 
      - file_server 
  file_server: 
    build: simple-server/. 
    ports: 
      - ${SERVER_PORT}:${SERVER_PORT} 
```

可以使用服务名称作为域名来引用链接的服务。

### .env

`.env`在`docker-compose.yaml`中用于加载模板变量。它遵循`<template-variable>=<value>`的格式：

```go
CONCIERGE_PORT=9090
A_M_PORT=6060
N_Z_PORT=7070
OTHERS_PORT=8080
SERVER_PORT=9876  
```

我们可以通过运行以下命令查看替换值后的`docker-compose.yaml`：

```go
$ pwd GO-WORKSPACE/src/github.com/last-ent/distributed-go/chapter8/goophr $ docker-compose config services: a_m_librarian: build: context: /home/entux/Documents/Code/GO-WORKSPACE/src/github.com/last-ent/distributed-go/chapter8/goophr/librarian environment: API_PORT: '6060' ports: - 6060:6060/tcp concierge: build: context: /home/entux/Documents/Code/GO-WORKSPACE/src/github.com/last-ent/distributed-go/chapter8/goophr/concierge environment: API_PORT: '9090' LIB_A_M: http://a_m_librarian:6060/api LIB_N_Z: http://n_z_librarian:7070/api LIB_OTHERS: http://others_librarian:8080/api links: - a_m_librarian - n_z_librarian - others_librarian - file_server ports: - 9090:9090/tcp file_server: build: context: /home/entux/Documents/Code/GO-WORKSPACE/src/github.com/last-ent/distributed-go/chapter8/goophr/simple-server ports: - 9876:9876/tcp n_z_librarian: build: context: /home/entux/Documents/Code/GO-WORKSPACE/src/github.com/last-ent/distributed-go/chapter8/goophr/librarian environment: API_PORT: '7070' ports: - 7070:7070/tcp others_librarian: build: context: /home/entux/Documents/Code/GO-WORKSPACE/src/github.com/last-ent/distributed-go/chapter8/goophr/librarian environment: API_PORT: '8080' ports: - 8080:8080/tcp version: '3.0' 
```

## 使用 docker-compose 运行 Goophr

现在我们已经准备就绪，让我们启动完整的应用程序：

```go
$ docker-compose up --build Building a_m_librarian ... Successfully built 31e0b1a7d3fc Building n_z_librarian ... Successfully built 31e0b1a7d3fc Building others_librarian ... Successfully built 31e0cdb1a7d3fc Building file_server ... Successfully built 244831d4b86a Building concierge ... Successfully built ba1167718d29 Starting goophr_a_m_librarian_1 ... Starting goophr_file_server_1 ... Starting goophr_a_m_librarian_1 Starting goophr_n_z_librarian_1 ... Starting goophr_others_librarian_1 ... Starting goophr_file_server_1 Starting goophr_n_z_librarian_1 Starting goophr_others_librarian_1 ... done Starting goophr_concierge_1 ... Starting goophr_concierge_1 ... done Attaching to goophr_a_m_librarian_1, goophr_n_z_librarian_1, goophr_file_server_1, goophr_others_librarian_1, goophr_concierge_1 a_m_librarian_1 | 2018/01/21 19:21:00 INFO - Adding API handlers... a_m_librarian_1 | 2018/01/21 19:21:00 INFO - Starting index... a_m_librarian_1 | 2018/01/21 19:21:00 INFO - Starting Goophr Librarian server on port :6060... n_z_librarian_1 | 2018/01/21 19:21:00 INFO - Adding API handlers... others_librarian_1 | 2018/01/21 19:21:01 INFO - Adding API handlers... others_librarian_1 | 2018/01/21 19:21:01 INFO - Starting index... others_librarian_1 | 2018/01/21 19:21:01 INFO - Starting Goophr Librarian server on port :8080... n_z_librarian_1 | 2018/01/21 19:21:00 INFO - Starting index... n_z_librarian_1 | 2018/01/21 19:21:00 INFO - Starting Goophr Librarian server on port :7070... file_server_1 | 2018/01/21 19:21:01 Starting File Server on Port :9876... concierge_1 | 2018/01/21 19:21:02 INFO - Adding API handlers... concierge_1 | 2018/01/21 19:21:02 INFO - Starting feeder... concierge_1 | 2018/01/21 19:21:02 INFO - Starting Goophr Concierge server on port :9090... 
```

### 向 Goophr 添加文档

由于我们的文件服务器中有三个文档，我们可以使用以下`curl`命令将它们添加到 Goophr 中：

```go
$ curl -LX POST -d '{"url":"http://file_server:9876/book1","title":"Book 1"}' localhost:9090/api/feeder | jq && > curl -LX POST -d '{"url":"http://file_server:9876/book2","title":"Book 2"}' localhost:9090/api/feeder | jq && > curl -LX POST -d '{"url":"http://file_server:9876/book3","title":"Book 3"}' localhost:9090/api/feeder | jq % Total % Received % Xferd Average Speed Time Time Time Current Dload Upload Total Spent Left Speed 100 107 100 51 100 56 51 56 0:00:01 --:--:-- 0:00:01 104k { "code": 200, "msg": "Request is being processed." } % Total % Received % Xferd Average Speed Time Time Time Current Dload Upload Total Spent Left Speed 100 107 100 51 100 56 51 56 0:00:01 --:--:-- 0:00:01 21400 { "code": 200, "msg": "Request is being processed." } % Total % Received % Xferd Average Speed Time Time Time Current Dload Upload Total Spent Left Speed 100 107 100 51 100 56 51 56 0:00:01 --:--:-- 0:00:01 21400 { "code": 200, "msg": "Request is being processed." } 
```

以下是由`docker-compose`看到的前述 cURL 请求的日志：

```go
n_z_librarian_1 | 2018/01/21 19:29:23 Token received api.tPayload{Token:"zebra", Title:"Book 1", DocID:"6911b2295fd23c77fca7d739c00735b14cf80d3c", LIndex:0, Index:3} concierge_1 | adding to librarian: zebra concierge_1 | adding to librarian: apple concierge_1 | adding to librarian: apple concierge_1 | adding to librarian: cat concierge_1 | 2018/01/21 19:29:23 INFO - Request was posted to Librairan. Msg:{"code": 200, "msg": "Tokens are being added to index."} ... concierge_1 | 2018/01/21 19:29:23 INFO - Request was posted to Librairan. Msg:{"code": 200, "msg": "Tokens are being added to index."} a_m_librarian_1 | 2018/01/21 19:29:23 Token received api.tPayload{Token:"apple", Title:"Book 1", DocID:"6911b2295fd23c77fca7d739c00735b14cf80d3c", LIndex:0, Index:0} ... n_z_librarian_1 | 2018/01/21 19:29:23 Token received api.tPayload{Token:"zebra", Title:"Book 2", DocID:"fbf2b6c400680389459dff13283cb01dfe9be7d6", LIndex:0, Index:2} concierge_1 | adding to librarian: zebra concierge_1 | adding to librarian: banana concierge_1 | adding to librarian: cake ... concierge_1 | adding to librarian: whale concierge_1 | adding to librarian: apple concierge_1 | adding to librarian: cake concierge_1 | adding to librarian: cake ... concierge_1 | 2018/01/21 19:29:23 INFO - Request was posted to Librairan. Msg:{"code": 200, "msg": "Tokens are being added to index."} 
```

## 使用 Goophr 搜索关键词

现在我们已经运行了完整的应用程序并且索引中有一些文档，让我们通过搜索一些关键词来测试它。以下是我们将要搜索的术语列表以及预期的顺序：

+   **"apple"** - book1 (score: 2), book 3 (score: 1)

+   **"cake"** - book 3 (score: 2), book 2 (score: 1)

+   **"apple"**, "**cake"** - book 3 (score 3), book 1 (score: 2), book 2 (score: 1)

### 搜索 – "apple"

让我们使用 cURL 命令单独搜索`"apple"`：

```go
$ curl -LX POST -d '["apple"]' localhost:9090/api/query | jq 
 % Total % Received % Xferd Average Speed Time Time Time Current 
 Dload Upload Total Spent Left Speed 
100 124 100 115 100 9 115 9 0:00:01 --:--:-- 0:00:01 41333 
[ 
 { 
 "title": "Book 1", 
 "url": "http://file_server:9876/book1" 
 }, 
 { 
 "title": "Book 3", 
 "url": "http://file_server:9876/book3" 
 } 
] 

```

当我们搜索`"apple"`时，以下是`docker-compose`的日志：

```go
concierge_1 | 2018/01/21 20:27:11 http://n_z_librarian:7070/api -> api.queryResult{Count:0, Data:[]api.docs{}}
concierge_1 | 2018/01/21 20:27:11 http://a_m_librarian:6060/api -> api.queryResult{Count:2, Data:[]api.docs{api.docs{DocID:"7bded23abfac73630d247b6ad24370214fe1811c", Score:2}, api.docs{DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", Score:1}}}
concierge_1 | []api.document{api.document{Doc:"apple apple cat zebra", Title:"Book 1", DocID:"7bded23abfac73630d247b6ad24370214fe1811c", URL:"http://file_server:9876/book1"}, api.document{Doc:"apple cake cake whale", Title:"Book 3", DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", URL:"http://file_server:9876/book3"}}
concierge_1 | 2018/01/21 20:27:11 http://others_librarian:8080/api -> api.queryResult{Count:0, Data:[]api.docs{}}

```

### 搜索 – "cake"

让我们使用 cURL 命令单独搜索`"cake"`：

```go
$ curl -LX POST -d '["cake"]' localhost:9090/api/query | jq 
 % Total % Received % Xferd Average Speed Time Time Time Current 
    Dload Upload Total Spent Left Speed 
100 123 100 115 100 8 115 8 0:00:01 --:--:-- 0:00:01 61500 
[ 
 { 
 "title": "Book 3", 
 "url": "http://file_server:9876/book3" 
 }, 
 { 
 "title": "Book 2", 
 "url": "http://file_server:9876/book2" 
 } 
] 
```

当我们搜索`"cake"`时，以下是`docker-compose`的日志：

```go
concierge_1 | 2018/01/21 20:30:13 http://a_m_librarian:6060/api -> api.queryResult{Count:2, Data:[]api.docs{api.docs{DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", Score:2}, api.docs{DocID:"28582e23c02ed3f14f8b4bdae97f91106273c0fc", Score:1}}}
concierge_1 | 2018/01/21 20:30:13 ---------------------------
concierge_1 | 2018/01/21 20:30:13 WARN: http://others_librarian:8080/api -> Post http://others_librarian:8080/api/query: http: ContentLength=8 with Body length 0
concierge_1 | 2018/01/21 20:30:13 ---------------------------
concierge_1 | 2018/01/21 20:30:13 http://n_z_librarian:7070/api -> api.queryResult{Count:0, Data:[]api.docs{}}
concierge_1 | []api.document{api.document{Doc:"apple cake cake whale", Title:"Book 3", DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", URL:"http://file_server:9876/book3"}, api.document{Doc:"banana cake zebra", Title:"Book 2", DocID:"28582e23c02ed3f14f8b4bdae97f91106273c0fc", URL:"http://file_server:9876/book2"}}

```

### 搜索 – "apple", "cake"

让我们使用 cURL 命令一起搜索`"apple"`和`"cake"`：

```go
$ curl -LX POST -d '["cake", "apple"]' localhost:9090/api/query | jq 
 % Total % Received % Xferd Average Speed Time Time Time Current 
 Dload Upload Total Spent Left Speed 
100 189 100 172 100 17 172 17 0:00:01 --:--:-- 0:00:01 27000 
[ 
 { 
 "title": "Book 3", 
 "url": "http://file_server:9876/book3" 
 }, 
 { 
 "title": "Book 1", 
 "url": "http://file_server:9876/book1" 
 }, 
 { 
 "title": "Book 2", 
 "url": "http://file_server:9876/book2" 
 } 
] 
```

当我们搜索`"apple"`和`"cake"`时，以下是`docker-compose`日志：

```go
concierge_1 | 2018/01/21 20:31:06 http://a_m_librarian:6060/api -> api.queryResult{Count:3, Data:[]api.docs{api.docs{DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", Score:3}, api.docs{DocID:"7bded23abfac73630d247b6ad24370214fe1811c", Score:2}, api.docs{DocID:"28582e23c02ed3f14f8b4bdae97f91106273c0fc", Score:1}}}
concierge_1 | 2018/01/21 20:31:06 http://n_z_librarian:7070/api -> api.queryResult{Count:0, Data:[]api.docs{}}
concierge_1 | 2018/01/21 20:31:06 ---------------------------
concierge_1 | 2018/01/21 20:31:06 WARN: http://others_librarian:8080/api -> Post http://others_librarian:8080/api/query: http: ContentLength=16 with Body length 0
concierge_1 | 2018/01/21 20:31:06 ---------------------------
concierge_1 | []api.document{api.document{Doc:"apple cake cake whale", Title:"Book 3", DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", URL:"http://file_server:9876/book3"}, api.document{Doc:"apple apple cat zebra", Title:"Book 1", DocID:"7bded23abfac73630d247b6ad24370214fe1811c", URL:"http://file_server:9876/book1"}, api.document{Doc:"banana cake zebra", Title:"Book 2", DocID:"28582e23c02ed3f14f8b4bdae97f91106273c0fc", URL:"http://file_server:9876/book2"}}
```

### 使用 docker-compose 的个人日志

我们还可以单独查看每个服务的日志。以下是礼宾的日志：

```go
$ docker-compose logs concierge
Attaching to goophr_concierge_1
concierge_1 | 2018/01/21 19:18:30 INFO - Adding API handlers...
concierge_1 | 2018/01/21 19:18:30 INFO - Starting feeder...
concierge_1 | 2018/01/21 19:18:30 INFO - Starting Goophr Concierge server on port :9090...
concierge_1 | 2018/01/21 19:21:02 INFO - Adding API handlers...
concierge_1 | 2018/01/21 19:21:02 INFO - Starting feeder...
concierge_1 | 2018/01/21 19:21:02 INFO - Starting Goophr Concierge server on port :9090...
concierge_1 | adding to librarian: zebra
concierge_1 | adding to librarian: apple
concierge_1 | adding to librarian: apple
concierge_1 | adding to librarian: cat
concierge_1 | 2018/01/21 19:25:40 INFO - Request was posted to Librairan. Msg:{"code": 200, "msg": "Tokens are being added to index."}
concierge_1 | 2018/01/21 20:31:06 http://a_m_librarian:6060/api -> api.queryResult{Count:3, Data:[]api.docs{api.docs{DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", Score:3}, api.docs{DocID:"7bded23abfac73630d247b6ad24370214fe1811c", Score:2}, api.docs{DocID:"28582e23c02ed3f14f8b4bdae97f91106273c0fc", Score:1}}}
concierge_1 | 2018/01/21 20:31:06 http://n_z_librarian:7070/api -> api.queryResult{Count:0, Data:[]api.docs{}}
concierge_1 | 2018/01/21 20:31:06 ---------------------------
concierge_1 | 2018/01/21 20:31:06 WARN: http://others_librarian:8080/api -> Post http://others_librarian:8080/api/query: http: ContentLength=16 with Body length 0
concierge_1 | 2018/01/21 20:31:06 ---------------------------
concierge_1 | []api.document{api.document{Doc:"apple cake cake whale", Title:"Book 3", DocID:"3c9c56d31ccd51bc7ac0011020819ef38ccd74a4", URL:"http://file_server:9876/book3"}, api.document{Doc:"apple apple cat zebra", Title:"Book 1", DocID:"7bded23abfac73630d247b6ad24370214fe1811c", URL:"http://file_server:9876/book1"}, api.document{Doc:"banana cake zebra", Title:"Book 2", DocID:"28582e23c02ed3f14f8b4bdae97f91106273c0fc", URL:"[`file_server:9876/book2`](http://file_server:9876/book2)"}}
```

## Web 服务器上的授权

我们的搜索应用程序信任每个传入的请求。然而，有时限制访问可能是正确的方式。如果对每个传入请求都能够接受和识别来自某些用户的请求，那将是可取的。这可以通过**授权令牌**（**auth tokens**）来实现。授权令牌是在标头中发送的秘密代码/短语，用于密钥**Authorization**。

授权和认证令牌是深奥而重要的话题。在本节中不可能涵盖主题的复杂性。相反，我们将构建一个简单的服务器，该服务器将利用认证令牌来接受或拒绝请求。让我们看看源代码。

### secure/secure.go

`secure.go`显示了简单服务器的逻辑。它已分为四个函数：

+   `requestHandler`函数用于响应传入的 HTTP 请求。

+   `isAuthorized`函数用于检查传入请求是否经过授权。

+   `getAuthorizedUser`函数用于检查令牌是否有关联用户。如果令牌没有关联用户，则认为令牌无效。

+   `main`函数用于启动服务器。

现在让我们看看代码：

```go
// secure/secure.go 
package main 

import ( 
    "fmt" 
    "log" 
    "net/http" 
    "strings" 
) 

var authTokens = map[string]string{ 
    "AUTH-TOKEN-1": "User 1", 
    "AUTH-TOKEN-2": "User 2", 
} 

// getAuthorizedUser tries to retrieve user for the given token. 
func getAuthorizedUser(token string) (string, error) { 
    var err error 

    user, valid := authTokens[token] 
    if !valid { 
        err = fmt.Errorf("Auth token '%s' does not exist.", token) 
    } 

    return user, err 
} 

// isAuthorized checks request to ensure that it has Authorization header 
// with defined value: "Bearer AUTH-TOKEN" 
func isAuthorized(r *http.Request) bool { 
    rawToken := r.Header["Authorization"] 
    if len(rawToken) != 1 { 
        return false 
    } 

    authToken := strings.Split(rawToken[0], " ") 
    if !(len(authToken) == 2 && authToken[0] == "Bearer") { 
        return false 
    } 

    user, err := getAuthorizedUser(authToken[1]) 
    if err != nil { 
        log.Printf("Error: %s", err) 
        return false 
    } 

    log.Printf("Successful request made by '%s'", user) 
    return true 
} 

var success = []byte("Received authorized request.") 
var failure = []byte("Received unauthorized request.") 

func requestHandler(w http.ResponseWriter, r *http.Request) { 
    if isAuthorized(r) { 
        w.Write(success) 
    } else { 
        w.WriteHeader(http.StatusUnauthorized) 
        w.Write(failure) 
    } 
} 

func main() { 
    http.HandleFunc("/", requestHandler) 
    fmt.Println("Starting server @ http://localhost:8080") 
    http.ListenAndServe(":8080", nil) 
} 
```

### secure/secure_test.go

接下来，我们将尝试使用单元测试测试我们在`secure.go`中编写的逻辑。一个好的做法是测试每个函数的所有可能的成功和失败情况。测试名称解释了测试的意图，所以让我们看看代码：

```go
// secure/secure_test.go 

package main 

import ( 
    "net/http" 
    "net/http/httptest" 
    "testing" 
) 

func TestIsAuthorizedSuccess(t *testing.T) { 
    req, err := http.NewRequest("GET", "http://example.com", nil) 
    if err != nil { 
        t.Error("Unable to create request") 
    } 

    req.Header["Authorization"] = []string{"Bearer AUTH-TOKEN-1"} 

    if isAuthorized(req) { 
        t.Log("Request with correct Auth token was correctly processed.") 
    } else { 
        t.Error("Request with correct Auth token failed.") 
    } 
} 

func TestIsAuthorizedFailTokenType(t *testing.T) { 
    req, err := http.NewRequest("GET", "http://example.com", nil) 
    if err != nil { 
        t.Error("Unable to create request") 
    } 

    req.Header["Authorization"] = []string{"Token AUTH-TOKEN-1"} 

    if isAuthorized(req) { 
        t.Error("Request with incorrect Auth token type was successfully processed.") 
    } else { 
        t.Log("Request with incorrect Auth token type failed as expected.") 
    } 
} 

func TestIsAuthorizedFailToken(t *testing.T) { 
    req, err := http.NewRequest("GET", "http://example.com", nil) 
    if err != nil { 
        t.Error("Unable to create request") 
    } 

    req.Header["Authorization"] = []string{"Token WRONG-AUTH-TOKEN"} 

    if isAuthorized(req) { 
        t.Error("Request with incorrect Auth token was successfully processed.") 
    } else { 
        t.Log("Request with incorrect Auth token failed as expected.") 
    } 
} 

func TestRequestHandlerFailToken(t *testing.T) { 
    req, err := http.NewRequest("GET", "http://example.com", nil) 
    if err != nil { 
        t.Error("Unable to create request") 
    } 

    req.Header["Authorization"] = []string{"Token WRONG-AUTH-TOKEN"} 

    // http.ResponseWriter it is an interface hence we use 
    // httptest.NewRecorder which implements the interface http.ResponseWriter 
    rr := httptest.NewRecorder() 
    requestHandler(rr, req) 

    if rr.Code == 401 { 
        t.Log("Request with incorrect Auth token failed as expected.") 
    } else { 
        t.Error("Request with incorrect Auth token was successfully processed.") 
    } 
} 

func TestGetAuthorizedUser(t *testing.T) { 
    if user, err := getAuthorizedUser("AUTH-TOKEN-2"); err != nil { 
        t.Errorf("Couldn't find User 2\. Error: %s", err) 
    } else if user != "User 2" { 
        t.Errorf("Found incorrect user: %s", user) 
    } else { 
        t.Log("Found User 2.") 
    } 
} 

func TestGetAuthorizedUserFail(t *testing.T) { 
    if user, err := getAuthorizedUser("WRONG-AUTH-TOKEN"); err == nil { 
        t.Errorf("Found user for invalid token!. User: %s", user) 
    } else if err.Error() != "Auth token 'WRONG-AUTH-TOKEN' does not exist." { 
        t.Errorf("Error message does not match.") 
    } else { 
        t.Log("Got expected error message for invalid auth token") 
    } 
} 
```

### 测试结果

最后，让我们运行测试，看看它们是否产生了预期的结果：

```go
$ go test -v ./... === RUN TestIsAuthorizedSuccess 2018/02/19 00:08:06 Successful request made by 'User 1' --- PASS: TestIsAuthorizedSuccess (0.00s) secure_test.go:18: Request with correct Auth token was correctly processed. === RUN TestIsAuthorizedFailTokenType --- PASS: TestIsAuthorizedFailTokenType (0.00s) secure_test.go:35: Request with incorrect Auth token type failed as expected. === RUN TestIsAuthorizedFailToken --- PASS: TestIsAuthorizedFailToken (0.00s) secure_test.go:50: Request with incorrect Auth token failed as expected. === RUN TestRequestHandlerFailToken --- PASS: TestRequestHandlerFailToken (0.00s) secure_test.go:68: Request with incorrect Auth token failed as expected. === RUN TestGetAuthorizedUser --- PASS: TestGetAuthorizedUser (0.00s) secure_test.go:80: Found User 2\. === RUN TestGetAuthorizedUserFail --- PASS: TestGetAuthorizedUserFail (0.00s) secure_test.go:90: Got expected error message for invalid auth token PASS ok chapter8/secure 0.003s 
```

## 总结

在本章中，我们首先尝试理解为什么需要运行多个 Goophr 图书管理员实例。接下来，我们看了如何实现更新的`concierge/api/query.go`，以便它可以与多个图书管理员实例一起工作。然后，我们研究了使用`docker-compose`编排应用程序可能是一个好主意的原因，以及使其工作的各种因素。我们还更新了图书管理员和礼宾代码库，以便它们可以与`docker-compose`无缝工作。最后，我们使用一些小文档测试了完整的应用程序，并推理了预期结果的顺序。

我们能够使用`docker-compose`在本地机器上编排运行完整的 Goophr 应用程序所需的所有服务器。然而，在互联网上设计一个能够承受大量用户流量的弹性 Web 应用程序的架构可能会非常具有挑战性。第九章，*Web 规模架构的基础*试图通过提供一些关于在 Web 设计时需要考虑的基本知识来解决这个问题。
