# 第七章：Goophr 图书管理员

在第六章中，*Goophr Concierge*，我们构建了负责接受新文档并将其分解为索引中使用的标记的端点。然而，Concierge 的`api.indexAdder`的当前实现在打印标记到控制台后返回。在本章中，我们将实现 Goophr 图书管理员，它可以与 Concierge 交互以接受标记，并响应标记搜索查询。

在本章中，我们将讨论以下主题：

+   标准索引模型

+   倒排索引模型

+   文档索引器

+   查询解析器 API

## 标准索引模型

考虑一本书中的索引。每本书都有自己的索引，按字母顺序列出所有单词，并显示它们在书中的位置。然而，如果我们想要跟踪单词在多本书中的出现，检查每本书的索引就相当低效。让我们看一个例子。

### 一个例子 - 具有单词索引的书籍

假设我们有三本书：`Book 1`，`Book 2`和`Book 3`，它们各自的索引如下。每个单词旁边的数字表示单词出现在哪一页：

```go
* Book 1 (Index)
 - apple - 4, 10, 20
 - cat - 10, 21, 22
 - zebra - 15, 25, 63

* Book 2 (Index)
 - banana - 14, 19, 66
 - cake - 10, 37, 45
 - zebra - 67, 100, 129

* Book 3 (Index)
 - apple - 36, 55, 74
 - cake - 1, 9, 77
 - Whale - 11, 59, 79  
```

让我们尝试从书的索引中找到三个词。一个天真的方法可能是选择每本书并扫描它，直到找到或错过这个词：

+   `苹果`

+   `香蕉`

+   `鹦鹉`

```go
* Searching for 'apple'
 - Scanning Book 1\. Result: Found.
 - Scanning Book 2\. Result: Not Found.
 - Scanning Book 3\. Result: Found.

* Searching for 'banana'
 - Scanning Book 1\. Result: Not Found.
 - Scanning Book 2\. Result: Found.
 - Scanning Book 3\. Result: Not Found.

* Searching for 'parrot'
 - Scanning Book 1\. Result: Not Found.
 - Scanning Book 2\. Result: Not Found.
 - Scanning Book 3\. Result: Not Found.  
```

简而言之，对于每个术语，我们都要遍历每本书的索引并搜索这个词。我们对每个单词都进行了整个过程，包括`鹦鹉`，而这个词并不存在于任何一本书中！起初，这可能在性能上看起来是可以接受的，但是考虑当我们需要查找超过一百万本书时，我们意识到这种方法是不切实际的。

## 倒排索引模型

根据前面的例子，我们可以陈述如下：

+   我们需要快速查找以确定一个词是否存在于我们的索引中

+   对于任何给定的单词，我们需要一种高效的方法来列出该单词可能出现在的所有书籍

通过使用倒排索引，我们可以实现这两个好处。标准索引的映射顺序是**书籍** → **单词 → **出现（页码、行号等），如前面的例子所示。如果我们使用倒排索引，映射顺序变为**单词 → **书籍 → **出现（页码、行号等）。

这个改变可能看起来并不重要，但它大大改善了查找。让我们用另一个例子来看一下。

### 一个例子 - 书中单词的倒排索引

让我们从之前的相同例子中获取数据，但现在根据倒排索引进行分类：

```go
* apple
 - Book 1 - 4, 10, 20
 - Book 3 - 36, 55, 74

* banana
 - Book 2 - 14, 19, 66

* cake
 - Book 2 - 10, 37, 45
 - Book 3 - 1, 9, 77

* cat
 - Book 1 - 10, 21, 22

* whale
 - Book 3 - 11, 59, 79

* zebra
 - Book 1 - 15, 25, 63
 - Book 2 - 67, 100, 129  
```

有了这个设置，我们可以高效地回答以下问题：

+   一个词是否存在于索引中？

+   一个词存在于哪些书中？

+   给定书中一个词出现在哪些页面上？

让我们再次尝试从倒排索引中找到三个单词：

+   `苹果`

+   `香蕉`

+   `鹦鹉`

```go
* Searching for 'apple'
 - Scanning Inverted Index. Result: Found a list of books.

* Searching for 'banana'
 - Scanning Inverted Index. Result: Found a list of books.

* Searching for 'parrot'
  - Scanning Inverted Index. Result: Not Found.  
```

总结一下，我们不是逐本书进行查找，而是对每个术语进行单次查找，确定术语是否存在，如果存在，则返回包含该术语的书籍列表，这是我们的最终目标。

## 排名

排名和搜索结果的相关性是一个有趣且复杂的话题。所有主要的搜索引擎都有一群专门的软件工程师和计算机科学家，他们花费大量时间和精力来确保他们的算法最准确。

对于 Goophr，我们将简化排名并将其限制为搜索词的频率。搜索词频率越高，排名越高。

## 重新审视 API 定义

让我们来审视图书管理员的 API 定义：

```go
openapi: 3.0.0 
servers: 
  - url: /api 
info: 
  title: Goophr Librarian API 
  version: '1.0' 
  description: | 
    API responsible for indexing & communicating with Goophr Concierge. 
paths: 
  /index: 
    post: 
      description: | 
        Add terms to index. 
      responses: 
        '200': 
          description: | 
            Terms were successfully added to the index. 
        '400': 
          description: > 
            Request was not processed because payload was incomplete or 
            incorrect. 
          content: 
            application/json: 
              schema: 
                $ref: '#/components/schemas/error' 
      requestBody: 
        content: 
          application/json: 
            schema: 
              $ref: '#/components/schemas/terms' 
        description: | 
          List of terms to be added to the index. 
        required: true 
  /query: 
    post: 
      description: | 
        Search for all terms in the payload. 
      responses: 
        '200': 
          description: | 
            Returns a list of all the terms along with their frequency, 
            documents the terms appear in and link to the said documents. 
          content: 
            application/json: 
              schema: 
                $ref: '#/components/schemas/results' 
        '400': 
          description: > 
            Request was not processed because payload was incomplete or 
            incorrect. 
          content: 
            application/json: 
              schema: 
                $ref: '#/components/schemas/error' 
    parameters: [] 
components: 
  schemas: 
    error: 
      type: object 
      properties: 
        msg: 
          type: string 
    term: 
      type: object 
      required: 
        - title 
        - token 
        - doc_id 
        - line_index 
        - token_index 
      properties: 
        title: 
          description: | 
            Title of the document to which the term belongs. 
          type: string 
        token: 
          description: | 
            The term to be added to the index. 
          type: string 
        doc_id: 
          description: | 
            The unique hash for each document. 
          type: string 
        line_index: 
          description: | 
            Line index at which the term occurs in the document. 
          type: integer 
        token_index: 
          description: | 
            Position of the term in the document. 
          type: integer 
    terms: 
      type: object 
      properties: 
        code: 
          type: integer 
        data: 
          type: array 
          items: 
            $ref: '#/components/schemas/term' 
    results: 
      type: object 
      properties: 
        count: 
          type: integer 
        data: 
          type: array 
          items: 
            $ref: '#/components/schemas/result' 
    result: 
      type: object 
      properties: 
        doc_id: 
          type: string 
        score: 
          type: integer  
```

根据 API 定义，我们可以陈述如下：

+   所有通信都是通过 JSON 格式进行

+   图书管理员的两个端点是：`/api/index`和`/api/query`

+   `/api/index`使用`POST`方法向反向索引添加新的标记

+   `/api/query`使用`POST`方法接收搜索查询词，并返回索引包含的所有文档的列表

## 文档索引器 - REST API 端点

`/api/index`的主要目的是接受 Concierge 的令牌并将其添加到索引中。让我们看看我们所说的“将其添加到索引”是什么意思。

文档索引可以定义为以下一系列连续的任务：

1.  我们依赖有效负载提供我们存储令牌所需的所有元信息。

1.  我们沿着倒排索引树向下，创建路径中尚未创建的任何节点，最后添加令牌详细信息。

## 查询解析器-REST API 端点

`/api/query`的主要目的是在倒排索引中找到一组搜索词，并按相关性递减的顺序返回文档 ID 列表。让我们看看我们所说的“查询搜索词”和“相关性”是什么意思。

查询解析可以定义为以下一系列连续的任务：

1.  对于每个搜索词，我们希望以倒排索引形式检索所有可用的书籍。

1.  接下来，我们希望在简单的查找表（`map`）中存储每本书中所有单词的出现计数。

1.  一旦我们有了一本书及其相应计数的映射，我们就可以将查找表转换为有序文档 ID 及其相应分数的数组。

## 代码约定

本章的代码非常简单直接，并且遵循与第六章相同的代码约定，*Goophr Concierge*。所以让我们直接进入代码。

## Librarian 源代码

现在我们已经详细讨论了 Librarian 的设计，让我们看看项目结构和源代码：

```go
$ tree . ├── api │ ├── index.go │ └── query.go ├── common │ ├── helpers.go ├── Dockerfile ├── main.go                               
```

两个目录和五个文件！

现在让我们看看每个文件的源代码。

### main.go

源文件负责初始化路由，启动索引系统和启动 Web 服务器：

```go
package main 

import ( 
    "net/http" 

    "github.com/last-ent/distributed-go/chapter7/goophr/librarian/api" 
    "github.com/last-ent/distributed-go/chapter7/goophr/librarian/common" 
) 

func main() { 
    common.Log("Adding API handlers...") 
    http.HandleFunc("/api/index", api.IndexHandler) 
    http.HandleFunc("/api/query", api.QueryHandler) 

    common.Log("Starting index...") 
    api.StartIndexSystem() 

    common.Log("Starting Goophr Librarian server on port :9090...") 
    http.ListenAndServe(":9090", nil) 
} 
```

### common/helpers.go

源文件包含专门针对一个处理程序的代码。

```go
package common 

import ( 
    "fmt" 
    "log" 
) 

func Log(msg string) { 
    log.Println("INFO - ", msg) 
} 

func Warn(msg string) { 
    log.Println("---------------------------") 
    log.Println(fmt.Sprintf("WARN: %s", msg)) 
    log.Println("---------------------------") 
} 
```

### api/index.go

包含代码以处理并向索引添加新项的源文件。

```go
package api 

import ( 
    "bytes" 
    "encoding/json" 
    "fmt" 
    "net/http" 
) 

// tPayload is used to parse the JSON payload consisting of Token data. 
type tPayload struct { 
    Token  string 'json:"token"' 
    Title  string 'json:"title"' 
    DocID  string 'json:"doc_id"' 
    LIndex int    'json:"line_index"' 
    Index  int    'json:"token_index"' 
} 

type tIndex struct { 
    Index  int 
    LIndex int 
} 

func (ti *tIndex) String() string { 
    return fmt.Sprintf("i: %d, li: %d", ti.Index, ti.LIndex) 
} 

type tIndices []tIndex 

// document - key in Indices represent Line Index. 
type document struct { 
    Count   int 
    DocID   string 
    Title   string 
    Indices map[int]tIndices 
} 

func (d *document) String() string { 
    str := fmt.Sprintf("%s (%s): %d\n", d.Title, d.DocID, d.Count) 
    var buffer bytes.Buffer 

    for lin, tis := range d.Indices { 
        var lBuffer bytes.Buffer 
        for _, ti := range tis { 
            lBuffer.WriteString(fmt.Sprintf("%s ", ti.String())) 
        } 
        buffer.WriteString(fmt.Sprintf("@%d -> %s\n", lin, lBuffer.String())) 
    } 
    return str + buffer.String() 
} 

// documentCatalog - key represents DocID. 
type documentCatalog map[string]*document 

func (dc *documentCatalog) String() string { 
    return fmt.Sprintf("%#v", dc) 
} 

// tCatalog - key in map represents Token. 
type tCatalog map[string]documentCatalog 

func (tc *tCatalog) String() string { 
    return fmt.Sprintf("%#v", tc) 
} 

type tcCallback struct { 
    Token string 
    Ch    chan tcMsg 
} 

type tcMsg struct { 
    Token string 
    DC    documentCatalog 
} 

// pProcessCh is used to process /index's payload and start process to add the token to catalog (tCatalog). 
var pProcessCh chan tPayload 

// tcGet is used to retrieve a token's catalog (documentCatalog). 
var tcGet chan tcCallback 

func StartIndexSystem() { 
    pProcessCh = make(chan tPayload, 100) 
    tcGet = make(chan tcCallback, 20) 
    go tIndexer(pProcessCh, tcGet) 
} 

// tIndexer maintains a catalog of all tokens along with where they occur within documents. 
func tIndexer(ch chan tPayload, callback chan tcCallback) { 
    store := tCatalog{} 
    for { 
        select { 
        case msg := <-callback: 
            dc := store[msg.Token] 
            msg.Ch <- tcMsg{ 
                DC:    dc, 
                Token: msg.Token, 
            } 

        case pd := <-ch: 
            dc, exists := store[pd.Token] 
            if !exists { 
                dc = documentCatalog{} 
                store[pd.Token] = dc 
            } 

            doc, exists := dc[pd.DocID] 
            if !exists { 
                doc = &document{ 
                    DocID:   pd.DocID, 
                    Title:   pd.Title, 
                    Indices: map[int]tIndices{}, 
                } 
                dc[pd.DocID] = doc 
            } 

            tin := tIndex{ 
                Index:  pd.Index, 
                LIndex: pd.LIndex, 
            } 
            doc.Indices[tin.LIndex] = append(doc.Indices[tin.LIndex], tin) 
            doc.Count++ 
        } 
    } 
} 

func IndexHandler(w http.ResponseWriter, r *http.Request) { 
    if r.Method != "POST" { 
        w.WriteHeader(http.StatusMethodNotAllowed) 
        w.Write([]byte('{"code": 405, "msg": "Method Not Allowed."}')) 
        return 
    } 

    decoder := json.NewDecoder(r.Body) 
    defer r.Body.Close() 

    var tp tPayload 
    decoder.Decode(&tp)

    log.Printf("Token received%#v\n", tp) 

    pProcessCh <- tp 

    w.Write([]byte('{"code": 200, "msg": "Tokens are being added to index."}')) 
} 
```

### api/query.go

源文件包含负责根据搜索词返回排序结果的代码。

```go
package api 

import ( 
    "encoding/json" 
    "net/http" 
    "sort" 

    "github.com/last-ent/distributed-go/chapter7/goophr/librarian/common" 
) 

type docResult struct { 
    DocID   string   'json:"doc_id"' 
    Score   int      'json:"doc_score"' 
    Indices tIndices 'json:"token_indices"' 
} 

type result struct { 
    Count int         'json:"count"' 
    Data  []docResult 'json:"data"' 
} 

// getResults returns unsorted search results & a map of documents containing tokens. 
func getResults(out chan tcMsg, count int) tCatalog { 
    tc := tCatalog{} 
    for i := 0; i < count; i++ { 
        dc := <-out 
        tc[dc.Token] = dc.DC 
    } 
    close(out) 

    return tc 
} 

func getFScores(docIDScore map[string]int) (map[int][]string, []int) { 
    // fScore maps frequency score to set of documents. 
    fScore := map[int][]string{} 

    fSorted := []int{} 

    for dID, score := range docIDScore { 
        fs := fScore[score] 
            fScore[score] = []string{} 
        } 
        fScore[score] = append(fs, dID) 
        fSorted = append(fSorted, score) 
    } 

    sort.Sort(sort.Reverse(sort.IntSlice(fSorted))) 

    return fScore, fSorted 
} 

func getDocMaps(tc tCatalog) (map[string]int, map[string]tIndices) { 
    // docIDScore maps DocIDs to occurences of all tokens. 
    // key: DocID. 
    // val: Sum of all occurences of tokens so far. 
    docIDScore := map[string]int{} 
    docIndices := map[string]tIndices{} 

    // for each token's catalog 
    for _, dc := range tc { 
        // for each document registered under the token 
        for dID, doc := range dc { 
            // add to docID score 
            var tokIndices tIndices 
            for _, tList := range doc.Indices { 
                tokIndices = append(tokIndices, tList...) 
            } 
            docIDScore[dID] += doc.Count 

            dti := docIndices[dID] 

            docIndices[dID] = append(dti, tokIndices...) 
        } 
    } 

    return docIDScore, docIndices 
} 

func sortResults(tc tCatalog) []docResult { 
    docIDScore, docIndices := getDocMaps(tc) 
    fScore, fSorted := getFScores(docIDScore) 

    results := []docResult{} 
    addedDocs := map[string]bool{} 

    for _, score := range fSorted { 
        for _, docID := range fScore[score] { 
            if _, exists := addedDocs[docID]; exists { 
                continue 
            } 
            results = append(results, docResult{ 
                DocID:   docID, 
                Score:   score, 
                Indices: docIndices[docID], 
            }) 
            addedDocs[docID] = false 
        } 
    } 
    return results 
} 

// getSearchResults returns a list of documents. 
// They are listed in descending order of occurences. 
func getSearchResults(sts []string) []docResult { 

    callback := make(chan tcMsg) 

    for _, st := range sts { 
        go func(term string) { 
            tcGet <- tcCallback{ 
                Token: term, 
                Ch:    callback, 
            } 
        }(st) 
    } 

    cts := getResults(callback, len(sts)) 
    results := sortResults(cts) 
    return results 
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
    decoder.Decode(&searchTerms) 

    results := getSearchResults(searchTerms) 

    payload := result{ 
        Count: len(results), 
        Data:  results, 
    } 

    if serializedPayload, err := json.Marshal(payload); err == nil { 
        w.Header().Add("Content-Type", "application/json") 
        w.Write(serializedPayload) 
    } else { 
        common.Warn("Unable to serialize all docs: " + err.Error()) 
        w.WriteHeader(http.StatusInternalServerError) 
        w.Write([]byte('{"code": 500, "msg": "Error occurred while trying to retrieve documents."}')) 
    } 
} 
```

## 测试 Librarian

为了测试 Librarian 是否按预期工作，我们需要测试两件事：

1.  检查`/api/index`是否接受索引项。

1.  检查`/api/query`是否返回正确的结果并且顺序符合预期。

我们可以使用一个单独的程序/脚本`feeder.go`来测试第 1 点，使用简单的 cURL 命令来测试第 2 点。

### 使用/api/index 测试`feeder.go`

这是`feeder.go`脚本，用于检查`/api/index`是否接受索引项：

```go
package main 

import ( 
    "bytes" 
    "encoding/json" 
    "io/ioutil" 
    "log" 
    "net/http" 
) 

type tPayload struct { 
    Token  string 'json:"token"' 
    Title  string 'json:"title"' 
    DocID  string 'json:"doc_id"' 
    LIndex int    'json:"line_index"' 
    Index  int    'json:"token_index"' 
} 

type msgS struct { 
    Code int    'json:"code"' 
    Msg  string 'json:"msg"' 
} 

func main() { 
    // Searching for "apple" should return Book 1 at the top of search results. 
    // Searching for "cake" should return Book 3 at the top. 
    for bookX, terms := range map[string][]string{ 
        "Book 1": []string{"apple", "apple", "cat", "zebra"}, 
        "Book 2": []string{"banana", "cake", "zebra"}, 
        "Book 3": []string{"apple", "cake", "cake", "whale"}, 
    } { 
        for lin, term := range terms { 
            payload, _ := json.Marshal(tPayload{ 
                Token:  term, 
                Title:  bookX + term, 
                DocID:  bookX, 
                LIndex: lin, 
            }) 
            resp, err := http.Post( 
                "http://localhost:9090/api/index", 
                "application/json", 
                bytes.NewBuffer(payload), 
            ) 
            if err != nil { 
                panic(err) 
            } 
            body, _ := ioutil.ReadAll(resp.Body) 
            defer resp.Body.Close() 

            var msg msgS 
            json.Unmarshal(body, &msg) 
            log.Println(msg) 
        } 
    } 
} 
```

运行`feeder.go`（在另一个窗口中运行 Librarian）的输出如下：

```go
$ go run feeder.go 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
2018/01/04 12:53:31 {200 Tokens are being added to index.} 
```

前述程序的 Librarian 输出如下：

```go
$ go run goophr/librarian/main.go 
2018/01/04 12:53:25 INFO - Adding API handlers... 
2018/01/04 12:53:25 INFO - Starting index... 
2018/01/04 12:53:25 INFO - Starting Goophr Librarian server on port :9090... 
2018/01/04 12:53:31 Token received api.tPayload{Token:"banana", Title:"Book 2banana", DocID:"Book 2", LIndex:0, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"cake", Title:"Book 2cake", DocID:"Book 2", LIndex:1, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"zebra", Title:"Book 2zebra", DocID:"Book 2", LIndex:2, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"apple", Title:"Book 3apple", DocID:"Book 3", LIndex:0, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"cake", Title:"Book 3cake", DocID:"Book 3", LIndex:1, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"cake", Title:"Book 3cake", DocID:"Book 3", LIndex:2, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"whale", Title:"Book 3whale", DocID:"Book 3", LIndex:3, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"apple", Title:"Book 1apple", DocID:"Book 1", LIndex:0, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"apple", Title:"Book 1apple", DocID:"Book 1", LIndex:1, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"cat", Title:"Book 1cat", DocID:"Book 1", LIndex:2, Index:0} 
2018/01/04 12:53:31 Token received api.tPayload{Token:"zebra", Title:"Book 1zebra", DocID:"Book 1", LIndex:3, Index:0}   
```

#### 测试/api/query

为了测试`/api/query`，我们需要维护服务器的前置状态以进行有用的查询：

```go
$ # Querying for "apple" $ curl -LX POST -d '["apple"]' localhost:9090/api/query | jq % Total % Received % Xferd Average Speed Time Time Time Current Dload Upload Total Spent Left Speed 100 202 100 193 100 9 193 9 0:00:01 --:--:-- 0:00:01 40400 { "count": 2, "data": [ { "doc_id": "Book 1", "doc_score": 2, "token_indices": [ { "Index": 0, "LIndex": 0 }, { "Index": 0, "LIndex": 1 } ] }, { "doc_id": "Book 3", "doc_score": 1, "token_indices": [ { "Index": 0, "LIndex": 0 } ] } ] } $ # Querying for "cake" 
$ curl -LX POST -d '["cake"]' localhost:9090/api/query | jq % Total % Received % Xferd Average Speed Time Time Time Current Dload Upload Total Spent Left Speed 100 201 100 193 100 8 193 8 0:00:01 --:--:-- 0:00:01 33500 { "count": 2, "data": [ { "doc_id": "Book 3", "doc_score": 2, "token_indices": [ { "Index": 0, "LIndex": 1 }, { "Index": 0, "LIndex": 2 } ] }, { "doc_id": "Book 2", "doc_score": 1, "token_indices": [ { "Index": 0, "LIndex": 1 } ] } ] }  
```

## 总结

在本章中，我们了解了倒排索引并为 Librarian 实现了高效的存储和查找搜索词。我们还使用脚本`feeder.go`和 cURL 命令检查了我们的实现。

在下一章，第八章，*部署 Goophr*，我们将重写 Concierge 的`api.indexAdder`，以便它可以开始将要索引的令牌发送给 Librarian。我们还将重新访问`docker-compose.yaml`，以便我们可以运行完整的应用程序并将其用作分布式系统进行使用/测试。
