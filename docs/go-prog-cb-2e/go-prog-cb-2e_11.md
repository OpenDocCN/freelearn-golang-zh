# 第十一章：分布式系统

有时，应用级并行性是不够的，开发中看起来简单的事情在部署过程中可能变得复杂。分布式系统在开发单台机器时找不到的一些挑战。这些应用程序增加了一些复杂性，比如监控、编写需要强一致性保证的应用程序和服务发现。此外，你必须时刻注意单点故障，比如数据库，否则你的分布式应用在这个单一组件失败时也会失败。

本章将探讨管理分布式数据、编排、容器化、指标和监控的方法。这些将成为你编写和维护微服务和大型分布式应用程序的工具箱的一部分。

在本章中，我们将涵盖以下配方：

+   使用 Consul 进行服务发现

+   使用 Raft 实现基本共识

+   使用 Docker 进行容器化

+   编排和部署策略

+   监控应用程序

+   收集指标

# 技术要求

要遵循本章中的所有配方，根据以下步骤配置你的环境：

1.  从[`golang.org/doc/install`](https://golang.org/doc/install)在你的操作系统上下载并安装 Go 1.12.6 或更高版本。

1.  从[`www.consul.io/intro/getting-started/install.html`](https://www.consul.io/intro/getting-started/install.html)安装 Consul。

1.  打开一个终端或控制台应用程序，并创建并进入一个项目目录，比如`~/projects/go-programming-cookbook`。所有的代码都将在这个目录中运行和修改。

1.  将最新的代码克隆到`~/projects/go-programming-cookbook-original`，（可选）从该目录中工作，而不是手动输入示例。

```go
$ git clone git@github.com:PacktPublishing/Go-Programming-Cookbook-Second-Edition.git go-programming-cookbook-original
```

# 使用 Consul 进行服务发现

当使用微服务方法来开发应用程序时，你最终会得到很多服务器监听各种 IP、域和端口。这些 IP 地址会因环境（测试与生产）而异，并且在服务之间保持静态以进行配置可能会很棘手。你还想知道何时一台机器或服务因网络分区而宕机或不可达。网络分区发生在网络的两个部分无法相互到达时。例如，如果两个数据中心之间的交换机失败，那么一个数据中心内的服务就无法到达另一个数据中心内的服务。Consul 是一个提供了很多功能的工具，但在这里，我们将探索如何使用 Consul 注册服务并从其他服务中查询它们。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter11/discovery`的新目录并进入。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/discovery 
```

你应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/discovery    
```

1.  从`~/projects/go-programming-cookbook-original/chapter11/discovery`复制测试，或者利用这个机会编写一些你自己的代码！

1.  创建一个名为`client.go`的文件，内容如下：

```go
        package discovery

        import "github.com/hashicorp/consul/api"

        // Client exposes api methods we care
        // about
        type Client interface {
            Register(tags []string) error
            Service(service, tag string) ([]*api.ServiceEntry,  
            *api.QueryMeta, error)
        }

        type client struct {
            client *api.Client
            address string
            name string
            port int
        }

        //NewClient iniitalizes a consul client
        func NewClient(config *api.Config, address, name string, port         
        int) (Client, error) {
            c, err := api.NewClient(config)
            if err != nil {
                return nil, err
            }
            cli := &client{
                client: c,
                name: name,
                address: address,
                port: port,
            }
            return cli, nil
        }
```

1.  创建一个名为`operations.go`的文件，内容如下：

```go
        package discovery

        import "github.com/hashicorp/consul/api"

        // Register adds our service to consul
        func (c *client) Register(tags []string) error {
            reg := &api.AgentServiceRegistration{
                ID: c.name,
                Name: c.name,
                Port: c.port,
                Address: c.address,
                Tags: tags,
            }
            return c.client.Agent().ServiceRegister(reg)
        }

        // Service return a service
        func (c *client) Service(service, tag string) 
        ([]*api.ServiceEntry, *api.QueryMeta, error) {
            return c.client.Health().Service(service, tag, false, 
            nil)
        }
```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
package discovery

import "fmt"

// Exec creates a consul entry then queries it
func Exec(cli Client) error {
  if err := cli.Register([]string{"Go", "Awesome"}); err != nil {
    return err
  }

  entries, _, err := cli.Service("discovery", "Go")
  if err != nil {
    return err
  }
  for _, entry := range entries {
    fmt.Printf("%#v\n", entry.Service)
  }

  return nil
}
```

1.  创建一个名为`example`的新目录并进入。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import "github.com/PacktPublishing/
                Go-Programming-Cookbook-Second-Edition/
                chapter11/discovery"

        func main() {
            if err := discovery.Exec(); err != nil {
                panic(err)
            }
        }
```

1.  使用`consul agent -dev -node=localhost`命令在一个单独的终端中启动 Consul。

1.  运行`go run main.go`命令。

1.  你也可以运行以下命令：

```go
$ go build $ ./example
```

你应该看到以下输出：

```go
$ go run main.go
&api.AgentService{ID:"discovery", Service:"discovery", Tags:    
[]string{"Go", "Awesome"}, Port:8080, Address:"localhost",     
EnableTagOverride:false, CreateIndex:0x23, ModifyIndex:0x23}
```

1.  `go.mod`文件可能会被更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果你复制或编写了自己的测试，返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

Consul 提供了一个强大的 Go API 库。当您第一次开始时，可能会感到令人生畏，但这个配方展示了您可能如何封装它。进一步配置 Consul 超出了此配方的范围；这显示了注册服务和在给定密钥和标签时查询其他服务的基础知识。

可以使用此功能在启动时注册新的微服务，查询所有依赖服务，并在关闭时注销。您可能还希望缓存此信息，以便不必为每个请求访问 Consul，但此配方提供了您可以扩展的基本工具。Consul 代理还使这些重复请求变得快速和高效（[`www.consul.io/intro/getting-started/agent.html`](https://www.consul.io/intro/getting-started/agent.html)）。一旦您

# 使用 Raft 实现基本共识

Raft 是一种共识算法。它允许分布式系统保持共享和受控状态（[`raft.github.io/`](https://raft.github.io/)）。建立 Raft 系统在许多方面都很复杂-首先，您需要共识才能进行选举并成功。当您使用多个节点时，这可能很难引导，并且可能很难开始。在单个节点/领导者上可以运行基本集群。但是，如果您需要冗余性，至少需要三个节点，以防止单个节点故障导致数据丢失。这个概念被称为法定人数，您必须维护(*n*/2)+1 个可用节点，以确保可以将新日志提交到 Raft 集群。基本上，如果您可以维持法定人数，集群将保持健康和可用。

此配方实现了一个基本的内存 Raft 集群，构建了一个可以在某些允许的状态之间转换的状态机，并将分布式状态机连接到可以触发转换的 Web 处理程序。在实现 Raft 所需的基本有限状态机接口或进行测试时，这可能非常有用。此配方使用[`github.com/hashicorp/raft`](https://github.com/hashicorp/raft)作为基本 Raft 实现。

# 如何做...

以下步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter11/consensus`的新目录并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/consensus 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/consensus    
```

1.  复制`~/projects/go-programming-cookbook-original/chapter11/consensus`中的测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`state.go`的文件，其中包含以下内容：

```go
        package consensus

        type state string

        const (
            first state = "first"
            second = "second"
            third = "third"
        )

        var allowedState map[state][]state

        func init() {
            // setup valid states
            allowedState = make(map[state][]state)
            allowedState[first] = []state{second, third}
            allowedState[second] = []state{third}
            allowedState[third] = []state{first}
        }

        // CanTransition checks if a new state is valid
        func (s *state) CanTransition(next state) bool {
            for _, n := range allowedState[*s] {
                if n == next {
                    return true
                }
            }
            return false
        }

        // Transition will move a state to the next
        // state if able
        func (s *state) Transition(next state) {
            if s.CanTransition(next) {
                *s = next
            }
        }
```

1.  创建一个名为`raftset.go`的文件，其中包含以下内容：

```go
package consensus

import (
  "fmt"

  "github.com/hashicorp/raft"
)

// keep a map of rafts for later
var rafts map[raft.ServerAddress]*raft.Raft

func init() {
  rafts = make(map[raft.ServerAddress]*raft.Raft)
}

// raftSet stores all the setup material we need
type raftSet struct {
  Config *raft.Config
  Store *raft.InmemStore
  SnapShotStore raft.SnapshotStore
  FSM *FSM
  Transport raft.LoopbackTransport
  Configuration raft.Configuration
}

// generate n raft sets to bootstrap the raft cluster
func getRaftSet(num int) []*raftSet {
  rs := make([]*raftSet, num)
  servers := make([]raft.Server, num)
  for i := 0; i < num; i++ {
    addr := raft.ServerAddress(fmt.Sprint(i))
    _, transport := raft.NewInmemTransport(addr)
    servers[i] = raft.Server{
      Suffrage: raft.Voter,
      ID: raft.ServerID(addr),
      Address: addr,
    }
    config := raft.DefaultConfig()
    config.LocalID = raft.ServerID(addr)

    rs[i] = &raftSet{
      Config: config,
      Store: raft.NewInmemStore(),
      SnapShotStore: raft.NewInmemSnapshotStore(),
      FSM: NewFSM(),
      Transport: transport,
    }
  }

  // configuration needs to be consistent between
  // services and so we need the full serverlist in this
  // case
  for _, r := range rs {
    r.Configuration = raft.Configuration{Servers: servers}
  }

  return rs
}
```

1.  创建一个名为`config.go`的文件，其中包含以下内容：

```go
package consensus

import (
  "github.com/hashicorp/raft"
)

// Config creates num in-memory raft
// nodes and connects them
func Config(num int) {

  // create n "raft-sets" consisting of
  // everything needed to represent a node
  rs := getRaftSet(num)

  //connect all of the transports
  for _, r1 := range rs {
    for _, r2 := range rs {
      r1.Transport.Connect(r2.Transport.LocalAddr(), r2.Transport)
    }
  }

  // for each node, bootstrap then connect
  for _, r := range rs {
    if err := raft.BootstrapCluster(r.Config, r.Store, r.Store, r.SnapShotStore, r.Transport, r.Configuration); err != nil {
      panic(err)
    }
    raft, err := raft.NewRaft(r.Config, r.FSM, r.Store, r.Store, r.SnapShotStore, r.Transport)
    if err != nil {
      panic(err)
    }
    rafts[r.Transport.LocalAddr()] = raft
  }
}
```

1.  创建一个名为`fsm.go`的文件，其中包含以下内容：

```go
        package consensus

        import (
            "io"

            "github.com/hashicorp/raft"
        )

        // FSM implements the raft FSM interface
        // and holds a state
        type FSM struct {
            state state
        }

        // NewFSM creates a new FSM with
        // start state of "first"
        func NewFSM() *FSM {
            return &FSM{state: first}
        }

        // Apply updates our FSM
        func (f *FSM) Apply(r *raft.Log) interface{} {
            f.state.Transition(state(r.Data))
            return string(f.state)
        }

        // Snapshot needed to satisfy the raft FSM interface
        func (f *FSM) Snapshot() (raft.FSMSnapshot, error) {
            return nil, nil
        }

        // Restore needed to satisfy the raft FSM interface
        func (f *FSM) Restore(io.ReadCloser) error {
            return nil
        }
```

1.  创建一个名为`handler.go`的文件，其中包含以下内容：

```go
package consensus

import (
  "net/http"
  "time"
)

// Handler grabs the get param ?next= and tries
// to transition to the state contained there
func Handler(w http.ResponseWriter, r *http.Request) {
  r.ParseForm()
  state := r.FormValue("next")
  for address, raft := range rafts {
    if address != raft.Leader() {
      continue
    }

    result := raft.Apply([]byte(state), 1*time.Second)
    if result.Error() != nil {
      w.WriteHeader(http.StatusBadRequest)
      return
    }
    newState, ok := result.Response().(string)
    if !ok {
      w.WriteHeader(http.StatusInternalServerError)
      return
    }

    if newState != state {
      w.WriteHeader(http.StatusBadRequest)
      w.Write([]byte("invalid transition"))
      return
    }
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(newState))
    return
  }
}
```

1.  创建一个名为`example`的新目录并转到该目录。

1.  创建一个名为`main.go`的文件，其中包含以下内容：

```go
        package main

        import (
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter11/consensus"
        )

        func main() {
            consensus.Config(3)

            http.HandleFunc("/", consensus.Handler)
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }
```

1.  运行`go run main.go`命令。或者，您也可以运行以下命令：

```go
$ go build
$ ./example
```

现在您应该看到以下输出：

```go
$ go run main.go
2019/05/04 21:06:46 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:0 Address:0} {Suffrage:Voter ID:1 Address:1} {Suffrage:Voter ID:2 Address:2}]
2019/05/04 21:06:46 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:0 Address:0} {Suffrage:Voter ID:1 Address:1} {Suffrage:Voter ID:2 Address:2}]
2019/05/04 21:06:46 [INFO] raft: Node at 0 [Follower] entering Follower state (Leader: "")
2019/05/04 21:06:46 [INFO] raft: Node at 1 [Follower] entering Follower state (Leader: "")
2019/05/04 21:06:46 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:0 Address:0} {Suffrage:Voter ID:1 Address:1} {Suffrage:Voter ID:2 Address:2}]
2019/05/04 21:06:46 [INFO] raft: Node at 2 [Follower] entering Follower state (Leader: "")
2019/05/04 21:06:47 [WARN] raft: Heartbeat timeout from "" reached, starting election
2019/05/04 21:06:47 [INFO] raft: Node at 0 [Candidate] entering Candidate state in term 2
2019/05/04 21:06:47 [DEBUG] raft: Votes needed: 2
2019/05/04 21:06:47 [DEBUG] raft: Vote granted from 0 in term 2\. Tally: 1
2019/05/04 21:06:47 [DEBUG] raft: Vote granted from 1 in term 2\. Tally: 2
2019/05/04 21:06:47 [INFO] raft: Election won. Tally: 2
2019/05/04 21:06:47 [INFO] raft: Node at 0 [Leader] entering Leader state
2019/05/04 21:06:47 [INFO] raft: Added peer 1, starting replication
2019/05/04 21:06:47 [INFO] raft: Added peer 2, starting replication
2019/05/04 21:06:47 [INFO] raft: pipelining replication to peer {Voter 1 1}
2019/05/04 21:06:47 [INFO] raft: pipelining replication to peer {Voter 2 2}
```

1.  在另一个终端中，运行以下命令：

```go
$ curl "http://localhost:3333/?next=second" 
second

$ curl "http://localhost:3333/?next=third" 
third

$ curl "http://localhost:3333/?next=second" 
invalid transition

$ curl "http://localhost:3333/?next=first" 
first
```

1.  `go.mod`文件可能会更新，`go.sum`文件现在应该存在于顶级配方目录中。

1.  如果您复制或编写了自己的测试，请返回到上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

当应用程序启动时，我们初始化多个 Raft 对象。每个对象都有自己的地址和传输方式。`InmemTransport{}`函数还提供了一个连接其他传输方式的方法，称为`Connect()`。一旦建立了这些连接，Raft 集群就会进行选举。在 Raft 集群中通信时，客户端必须与领导者通信。在我们的情况下，一个处理程序可以与所有节点通信，因此处理程序负责拥有`Raft`领导者的`call Apply()`对象。这反过来又在所有其他节点上运行`apply()`。

`InmemTransport{}`函数通过允许所有内容驻留在内存中来简化选举和引导过程。在实践中，除了测试和概念验证之外，这并不是很有帮助，因为 Goroutines 可以自由访问共享内存。一个更适合生产的实现会使用类似 HTTP 传输的东西，这样服务实例可以跨机器通信。这可能需要一些额外的簿记或服务发现，因为服务实例必须监听和提供服务，同时还必须能够发现并建立彼此的连接。

# 使用 Docker 进行容器化

Docker 是一种用于打包和运输应用程序的容器技术。其他优势包括可移植性，因为容器无论在哪个主机操作系统上都会以相同的方式运行。它提供了虚拟机的许多优势，但是以更轻量的容器形式。可以限制单个容器的资源消耗并隔离您的环境。在本地为应用程序和在生产环境中部署代码时，拥有一个共同的环境非常有用。Docker 是用 Go 语言编写的，是开源的，因此很容易利用客户端和库。这个配方将为一个基本的 Go 应用程序设置一个 Docker 容器，存储一些关于容器的版本信息，并演示如何从 Docker 端点访问处理程序。

# 准备工作

根据以下步骤配置您的环境：

1.  参考本章的*技术要求*部分，配置环境的步骤。

1.  从[`docs.docker.com/install`](https://docs.docker.com/install)安装 Docker。这也将包括 Docker Compose。

# 如何做...

这些步骤涵盖了编写和运行应用程序的过程：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter11/docker`的新目录，并进入该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/docker 
```

你应该会看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/docker    
```

1.  从`~/projects/go-programming-cookbook-original/chapter11/docker`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`dockerfile`的文件，内容如下：

```go
        FROM alpine

        ADD ./example/example /example
        EXPOSE 8000
        ENTRYPOINT /example 
```

1.  创建一个名为`setup.sh`的文件，内容如下：

```go
        #!/usr/bin/env bash

        pushd example
        env GOOS=linux go build -ldflags "-X main.version=1.0 -X     
        main.builddate=$(date +%s)"
        popd
        docker build . -t example
        docker run -d -p 8000:8000 example 
```

1.  创建一个名为`version.go`的文件，内容如下：

```go
        package docker

        import (
            "encoding/json"
            "net/http"
            "time"
        )

        // VersionInfo holds artifacts passed in
        // at build time
        type VersionInfo struct {
            Version string
            BuildDate time.Time
            Uptime time.Duration
        }

        // VersionHandler writes the latest version info
        func VersionHandler(v *VersionInfo) http.HandlerFunc {
            t := time.Now()
            return func(w http.ResponseWriter, r *http.Request) {
                v.Uptime = time.Since(t)
                vers, err := json.Marshal(v)
                    if err != nil {
                        w.WriteHeader
                        (http.StatusInternalServerError)
                        return
                    }
                    w.WriteHeader(http.StatusOK)
                    w.Write(vers)
            }
        }
```

1.  创建一个名为`example`的新目录并进入。

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "fmt"
            "net/http"
            "strconv"
            "time"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter11/docker"
        )

        // these are set at build time
        var (
            version string
            builddate string
            )

            var versioninfo docker.VersionInfo

            func init() {
                // parse buildtime variables
                versioninfo.Version = version
                i, err := strconv.ParseInt(builddate, 10, 64)
                    if err != nil {
                        panic(err)
                    }
                    tm := time.Unix(i, 0)
                    versioninfo.BuildDate = tm
            }

            func main() {
            http.HandleFunc("/version",     
            docker.VersionHandler(&versioninfo))
            fmt.Printf("version %s listening on :8000\n",   
            versioninfo.Version)
            panic(http.ListenAndServe(":8000", nil))
        }
```

1.  导航回起始目录。

1.  运行以下命令：

```go
$ bash setup.sh
```

现在你应该会看到以下输出：

```go
$ bash setup.sh
~/go/src/github.com/PacktPublishing/Go-Programming-Cookbook-
Second-Edition/chapter11/docker/example   
~/go/src/github.com/PacktPublishing/Go-Programming-Cookbook-
Second-Edition/chapter11/docker
~/go/src/github.com/PacktPublishing/Go-Programming-Cookbook-   
Second-Edition/chapter11/docker
Sending build context to Docker daemon 6.031 MB
Step 1/4 : FROM alpine
 ---> 4a415e366388
Step 2/4 : ADD ./example/example /example
 ---> de34c3c5451e
Removing intermediate container bdcd9c4f4381
Step 3/4 : EXPOSE 8000
 ---> Running in 188f450d4e7b
 ---> 35d1a2652b43
Removing intermediate container 188f450d4e7b
Step 4/4 : ENTRYPOINT /example
 ---> Running in cf0af4f48c3a
 ---> 3d737fc4e6e2
Removing intermediate container cf0af4f48c3a
Successfully built 3d737fc4e6e2
b390ef429fbd6e7ff87058dc82e15c3e7a8b2e
69a601892700d1d434e9e8e43b
```

1.  运行以下命令：

```go
$ docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
b390ef429fbd example "/bin/sh -c /example" 22 seconds ago Up 23    
seconds 0.0.0.0:8000->8000/tcp optimistic_wescoff

$ curl localhost:8000/version
{"Version":"1.0","BuildDate":"2017-04-   
30T21:55:56Z","Uptime":48132111264}

$docker kill optimistic_wescoff # grab from first output
optimistic_wescoff
```

1.  `go.mod`文件可能会更新，`go.sum`文件现在应该存在于顶层配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

这个示例创建了一个脚本，用于为 Linux 架构编译 Go 二进制文件，并在`main.go`中设置各种私有变量。这些变量用于在版本端点上返回版本信息。一旦编译了二进制文件，就会创建一个包含二进制文件的 Docker 容器。这允许我们使用非常小的容器映像，因为 Go 运行时在二进制文件中是自包含的。然后我们运行容器，同时暴露容器监听 HTTP 流量的端口。最后，我们在本地主机上`curl`端口，并看到我们的版本信息返回。

# 编排和部署策略

Docker 使编排和部署变得更加简单。在这个示例中，我们将建立与 MongoDB 的连接，然后从 Docker 容器中插入文档并查询它。这个示例将设置与第六章中*使用 NoSQL 与 MongoDB 和 mgo*配方相同的环境，*关于数据库和存储的一切*，但将应用程序和环境运行在容器内，并使用 Docker Compose 进行编排和连接。

这可以与 Docker Swarm 一起使用，Docker Swarm 是一个集成的 Docker 工具，允许您管理集群，创建和部署可以轻松扩展或缩减的节点，并管理负载平衡（[`docs.docker.com/engine/swarm/`](https://docs.docker.com/engine/swarm/)）。另一个很好的容器编排示例是 Kubernetes（[`kubernetes.io/`](https://kubernetes.io/)），这是一个由 Google 使用 Go 编程语言编写的容器编排框架。

# 操作步骤...

以下步骤涵盖了编写和运行应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter11/orchestrate`的新目录，并导航到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/orchestrate 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/orchestrate    
```

1.  从`~/projects/go-programming-cookbook-original/chapter11/orchestrate`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`Dockerfile`的文件，内容如下：

```go
FROM golang:1.12.4-alpine3.9

ENV GOPATH /code/
ADD . /code/src/github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/docker
WORKDIR /code/src/github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/docker/example
RUN GO111MODULE=on GOPROXY=off go build -mod=vendor

ENTRYPOINT /code/src/github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/docker/example/example
```

1.  创建一个名为`docker-compose.yml`的文件，内容如下：

```go
        version: '2'
        services:
         app:
         build: .
         mongodb:
         image: "mongo:latest"
```

1.  创建一个名为`config.go`的文件，内容如下：

```go
package mongodb

import (
  "context"
  "fmt"
  "time"

  "github.com/mongodb/mongo-go-driver/mongo"
  "go.mongodb.org/mongo-driver/mongo/options"
)

// Setup initializes a mongo client
func Setup(ctx context.Context, address string) (*mongo.Client, error) {
  ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
  defer cancel()

  fmt.Println(address)
  client, err := mongo.NewClient(options.Client().ApplyURI(address))
  if err != nil {
    return nil, err
  }

  if err := client.Connect(ctx); err != nil {
    return nil, err
  }
  return client, nil
}

```

1.  创建一个名为`exec.go`的文件，内容如下：

```go
package mongodb

import (
  "context"
  "fmt"

  "github.com/mongodb/mongo-go-driver/bson"
)

// State is our data model
type State struct {
  Name string `bson:"name"`
  Population int `bson:"pop"`
}

// Exec creates then queries an Example
func Exec(address string) error {
  ctx := context.Background()
  db, err := Setup(ctx, address)
  if err != nil {
    return err
  }

  conn := db.Database("gocookbook").Collection("example")

  vals := []interface{}{&State{"Washington", 7062000}, &State{"Oregon", 3970000}}

  // we can inserts many rows at once
  if _, err := conn.InsertMany(ctx, vals); err != nil {
    return err
  }

  var s State
  if err := conn.FindOne(ctx, bson.M{"name": "Washington"}).Decode(&s); err != nil {
    return err
  }

  if err := conn.Drop(ctx); err != nil {
    return err
  }

  fmt.Printf("State: %#v\n", s)
  return nil
}
```

1.  创建一个名为`example`的新目录，并导航到该目录。

1.  创建一个`main.go`文件，内容如下：

```go
package main

import mongodb "github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/orchestrate"

func main() {
  if err := mongodb.Exec("mongodb://mongodb:27017"); err != nil {
    panic(err)
  }
}
```

1.  返回到起始目录。

1.  运行`go mod vendor`命令。

1.  运行`docker-compose up -d`命令。

1.  运行`docker logs orchestrate_app_1`命令。现在应该看到以下输出：

```go
$ docker logs orchestrate_app_1
State: docker.State{Name:"Washington", Population:7062000}
```

1.  `go.mod`文件可能会被更新，顶级配方目录中现在应该存在`go.sum`文件。

1.  如果您复制或编写了自己的测试，请返回到上一级目录并运行`go test`。确保所有测试都通过。

# 工作原理...

这个配置适用于本地开发。一旦运行`docker-compose up`命令，本地目录将被重建，Docker 将使用最新版本与 MongoDB 实例建立连接，并开始对其进行操作。此示例使用 go mod vendor 进行依赖管理。因此，我们禁用`go mod cache`并告诉`go build`命令使用我们创建的 vendor 目录。

这可以为需要连接到外部服务的应用程序提供一个良好的基线；第六章中的所有配方，*关于数据库和存储的一切*，都可以使用这种方法，而不是创建数据库的本地实例。对于生产环境，您可能不希望在 Docker 容器后面运行数据存储，但通常也会有静态主机名用于配置。

# 监控应用程序

有多种方法可以监视 Go 应用程序。其中最简单的方法之一是设置 Prometheus，这是一个用 Go 编写的监视应用程序（[`prometheus.io`](https://prometheus.io)）。这是一个根据您的配置文件轮询端点并收集有关您的应用程序的大量信息的应用程序，包括 Goroutines 的数量、内存使用情况等等。这个应用程序将使用上一个教程中的技术来设置一个 Docker 环境来托管 Prometheus 并连接到它。

# 操作步骤...

以下步骤涵盖了编写和运行您的应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter11/monitoring`的新目录，并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/monitoring 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/monitoring    
```

1.  从`~/projects/go-programming-cookbook-original/chapter11/monitoring`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`Dockerfile`的文件，内容如下：

```go
        FROM golang:1.12.4-alpine3.9

        ENV GOPATH /code/
        ADD . /code/src/github.com/agtorre/go-
        cookbook/chapter11/monitoring
        WORKDIR /code/src/github.com/agtorre/go-
        cookbook/chapter11/monitoring
        RUN GO111MODULE=on GOPROXY=off go build -mod=vendor

        ENTRYPOINT /code/src/github.com/agtorre/go-
        cookbook/chapter11/monitoring/monitoring
```

1.  创建一个名为`docker-compose.yml`的文件，内容如下：

```go
        version: '2'
        services:
         app:
         build: .
         prometheus:
         ports: 
         - 9090:9090
         volumes: 
         - ./prometheus.yml:/etc/prometheus/prometheus.yml
         image: "prom/prometheus"
```

1.  创建一个名为`main.go`的文件，内容如下：

```go
        package main

        import (
            "net/http"

            "github.com/prometheus/client_golang/prometheus/promhttp"
        )

        func main() {
            http.Handle("/metrics", promhttp.Handler())
            panic(http.ListenAndServe(":80", nil))
        }
```

1.  创建一个名为`prometheus.yml`的文件，内容如下：

```go
        global:
         scrape_interval: 15s # By default, scrape targets every 15 
         seconds.

        # A scrape configuration containing exactly one endpoint to 
        scrape:
        # Here it's Prometheus itself.
        scrape_configs:
         # The job name is added as a label `job=<job_name>` to any 
         timeseries scraped from this config.
         - job_name: 'app'

         # Override the global default and scrape targets from this job          
         every 5 seconds.
         scrape_interval: 5s

         static_configs:
         - targets: ['app:80']
```

1.  运行`go mod vendor`命令。

1.  运行`docker-compose up`命令。现在您应该看到以下输出：

```go
$ docker-compose up
Starting monitoring_prometheus_1 ... done
Starting monitoring_app_1 ... done
Attaching to monitoring_app_1, monitoring_prometheus_1
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Starting prometheus (version=1.6.1, branch=master, revision=4666df502c0e239ed4aa1d80abbbfb54f61b23c3)" source="main.go:88" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Build context (go=go1.8.1, user=root@7e45fa0366a7, date=20170419-14:32:22)" source="main.go:89" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Loading configuration file /etc/prometheus/prometheus.yml" source="main.go:251" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Loading series map and head chunks..." source="storage.go:421" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=warning msg="Persistence layer appears dirty." source="persistence.go:846" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=warning msg="Starting crash recovery. Prometheus is inoperational until complete." source="crashrecovery.go:40" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=warning msg="To avoid crash recovery in the future, shut down Prometheus with SIGTERM or a HTTP POST to /-/quit." source="crashrecovery.go:41" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Scanning files." source="crashrecovery.go:55" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="File scan complete. 43 series found." source="crashrecovery.go:83" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Checking for series without series file." source="crashrecovery.go:85" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Check for series without series file complete." source="crashrecovery.go:131" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Cleaning up archive indexes." source="crashrecovery.go:411" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Clean-up of archive indexes complete." source="crashrecovery.go:504" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Rebuilding label indexes." source="crashrecovery.go:512" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Indexing metrics in memory." source="crashrecovery.go:513" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Indexing archived metrics." source="crashrecovery.go:521" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="All requests for rebuilding the label indexes queued. (Actual processing may lag behind.)" source="crashrecovery.go:540" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=warning msg="Crash recovery complete." source="crashrecovery.go:153" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="43 series loaded." source="storage.go:432" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Starting target manager..." source="targetmanager.go:61" 
prometheus_1 | time="2019-05-05T03:10:25Z" level=info msg="Listening on :9090" source="web.go:259" 
```

1.  `go.mod`文件可能已更新，并且顶级配方目录中现在应该存在`go.sum`文件。

1.  在浏览器中导航到`http://localhost:9090/`。您应该看到与您的应用程序相关的各种指标！

# 工作原理...

这个教程在 Go 中创建了一个简单的处理程序，使用 prometheus go 客户端将有关正在运行的应用程序的统计信息导出到 prometheus。我们将我们的应用程序连接到在 docker 中运行的 prometheus 服务器，并使用 docker-compose 处理网络连接和启动。收集数据的频率、应用程序提供服务的端口以及应用程序的名称都在`prometheus.yml`文件中指定。一旦两个容器都启动，prometheus 服务器就开始在指定的端口上收集和监控应用程序。它还公开了一个 Web 界面，我们可以在浏览器中访问以查看有关我们的应用程序的更多信息。

Prometheus 客户端处理程序将向 Prometheus 服务器返回有关您的应用程序的各种统计信息。这使您可以将多个 Prometheus 服务器指向一个应用程序，而无需重新配置或部署该应用程序。其中大多数统计信息是通用的，并且对于诸如检测内存泄漏之类的事情非常有益。许多其他解决方案要求您定期向服务器发送信息。下一个教程，*收集指标*，将演示如何将自定义指标发送到 Prometheus 服务器。

# 收集指标

除了关于您的应用程序的一般信息之外，发出特定于应用程序的指标也可能有所帮助。例如，我们可能希望收集定时数据或跟踪事件发生的次数。

这个教程将使用`github.com/rcrowley/go-metrics`包来收集指标并通过一个端点公开它们。有各种导出工具可以用来将指标导出到诸如 Prometheus 和 InfluxDB 之类的地方，这些工具也是用 Go 编写的。

# 准备工作

根据以下步骤配置您的环境：

1.  请参阅本章的*技术要求*部分，了解配置环境的步骤。

1.  运行`go get github.com/rcrowley/go-metrics`命令。

# 操作步骤...

这些步骤涵盖了编写和运行您的应用程序：

1.  从您的终端或控制台应用程序中，创建一个名为`~/projects/go-programming-cookbook/chapter11/metrics`的新目录，并转到该目录。

1.  运行以下命令：

```go
$ go mod init github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/metrics 
```

您应该看到一个名为`go.mod`的文件，其中包含以下内容：

```go
module github.com/PacktPublishing/Go-Programming-Cookbook-Second-Edition/chapter11/metrics    
```

1.  从`~/projects/go-programming-cookbook-original/chapter11/metrics`复制测试，或者利用这个机会编写一些自己的代码！

1.  创建一个名为`handler.go`的文件，内容如下：

```go
        package metrics

        import (
            "net/http"
            "time"

            metrics "github.com/rcrowley/go-metrics"
        )

        // CounterHandler will update a counter each time it's called
        func CounterHandler(w http.ResponseWriter, r *http.Request) {
            c := metrics.GetOrRegisterCounter("counterhandler.counter", 
            nil)
            c.Inc(1)

            w.WriteHeader(http.StatusOK)
            w.Write([]byte("success"))
        }

        // TimerHandler records the duration required to compelete
        func TimerHandler(w http.ResponseWriter, r *http.Request) {
            currt := time.Now()
            t := metrics.GetOrRegisterTimer("timerhandler.timer", nil)

            w.WriteHeader(http.StatusOK)
            w.Write([]byte("success"))
            t.UpdateSince(currt)
        }
```

1.  创建一个名为`report.go`的文件，内容如下：

```go
        package metrics

        import (
            "net/http"

            gometrics "github.com/rcrowley/go-metrics"
        )

        // ReportHandler will emit the current metrics in json format
        func ReportHandler(w http.ResponseWriter, r *http.Request) {

            w.WriteHeader(http.StatusOK)

            t := gometrics.GetOrRegisterTimer(
            "reporthandler.writemetrics", nil)
            t.Time(func() {
                gometrics.WriteJSONOnce(gometrics.DefaultRegistry, w)
            })
        }
```

1.  创建一个名为`example`的新目录并进入。

1.  创建一个名为`main.go`的文件：

```go
        package main

        import (
            "net/http"

            "github.com/PacktPublishing/
             Go-Programming-Cookbook-Second-Edition/
             chapter11/metrics"
        )

        func main() {
            // handler to populate metrics
            http.HandleFunc("/counter", metrics.CounterHandler)
            http.HandleFunc("/timer", metrics.TimerHandler)
            http.HandleFunc("/report", metrics.ReportHandler)
            fmt.Println("listening on :8080")
            panic(http.ListenAndServe(":8080", nil))
        }
```

1.  运行`go run main.go`。或者，您也可以运行以下命令：

```go
$ go build $ ./example
```

现在您应该看到以下输出：

```go
$ go run main.go
listening on :8080
```

1.  从单独的 shell 中运行以下命令：

```go
$ curl localhost:8080/counter 
success

$ curl localhost:8080/timer 
success

$ curl localhost:8080/report 
{"counterhandler.counter":{"count":1},
"reporthandler.writemetrics":      {"15m.rate":0,"1m.rate":0,"5m.rate":0,"75%":0,"95%":0,"99%":0,"99.9%":0,"count":0,"max":0,"mean":0,"mean.rate":0,"median":0,"min":0,"stddev":0},"timerhandler.timer":{"15m.rate":0.0011080303990206543,"1m.rate":0.015991117074135343,"5m.rate":0.0033057092356765017,"75%":60485,"95%":60485,"99%":60485,"99.9%":60485,"count":1,"max":60485,"mean":60485,"mean.rate":1.1334543719787356,"median":60485,"min":60485,"stddev":0}}
```

1.  尝试多次访问所有端点，看看它们如何变化。

1.  `go.mod`文件可能会被更新，而`go.sum`文件现在应该存在于顶层的配方目录中。

1.  如果您复制或编写了自己的测试，请返回上一级目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

`gometrics`将所有度量标准保存在注册表中。一旦设置好，您可以使用任何度量发射选项，比如`counter`或`timer`，它将把这个更新存储在注册表中。有多个导出器将度量标准导出到第三方工具。在我们的情况下，我们设置了一个以 JSON 格式发射所有度量标准的处理程序。

我们设置了三个处理程序——一个用于增加计数器，一个用于记录退出处理程序的时间，以及一个用于打印报告（同时还增加了一个额外的计数器）。`GetOrRegister`函数对于以线程安全的方式原子地获取或创建度量发射器非常有用。或者，您可以提前注册所有内容。
