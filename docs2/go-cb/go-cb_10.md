# 分布式系统

在本章中，我们将涵盖以下食谱：

+   使用 Consul 进行服务发现

+   使用 Raft 实现基本共识

+   使用 Docker 进行容器化

+   编排和部署策略

+   监控应用程序

+   收集指标

# 简介

有时，应用级别的并行性不足以解决问题，开发中看似简单的事情在部署时可能会变得复杂。分布式系统在单机开发时不会遇到许多挑战。这些应用程序为监控、编写需要强一致性保证的应用程序和服务发现等问题增加了复杂性。此外，您必须始终注意单点故障，例如数据库。否则，当这个单一组件失败时，您的分布式应用程序可能会失败。

本章将探讨管理分布式数据、编排、容器化、指标和监控的方法。这些将成为您编写和维护微服务和大型分布式应用程序的工具箱的一部分。

# 使用 Consul 进行服务发现

当使用微服务方法构建应用程序时，你将拥有许多服务器，它们监听各种 IP、域名和端口。这些 IP 地址会因环境（预发布与生产）而异，并且在不同服务之间保持静态配置可能会很棘手。你还想了解当机器或服务因网络分区而无法访问或宕机时的情况。Consul 是一个提供许多功能的工具，但我们将探讨如何使用 Consul 注册服务以及如何从我们的其他服务中查询它们。

# 准备工作

根据以下步骤配置您的环境：

1.  从 [`golang.org/doc/install`](https://golang.org/doc/install) 下载并安装 Go 到您的操作系统上，并配置您的 `GOPATH` 环境变量。

1.  打开终端/控制台应用程序。

1.  导航到 `GOPATH/src` 并创建一个项目目录，例如，`$GOPATH/src/github.com/yourusername/customrepo`。

    所有代码都将从这个目录运行和修改。

1.  可选地，通过运行 `go get github.com/agtorre/go-cookbook/` 命令安装最新测试版本的代码。

1.  从 [`www.consul.io/intro/getting-started/install.html`](https://www.consul.io/intro/getting-started/install.html) 安装 Consul。

1.  运行 `go get github.com/hashicorp/consul/api` 命令。

# 如何做到...

这些步骤涵盖了编写和运行应用程序：

1.  从您的终端/控制台应用程序中创建 `chapter10/discovery` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter10/discovery`](https://github.com/agtorre/go-cookbook/tree/master/chapter10/discovery) 复制测试，或者将其作为练习编写一些自己的代码。

1.  创建一个名为 `client.go` 的文件，内容如下：

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

1.  创建一个名为 `operations.go` 的文件，内容如下：

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

1.  创建一个名为 `exec.go` 的文件，并包含以下内容：

```go
        package discovery

        import (
            "fmt"

            consul "github.com/hashicorp/consul/api"
        )

        // Exec creates a consul entry then queries it
        func Exec() error {
            config := consul.DefaultConfig()
            config.Address = "localhost:8500"
            name := "discovery"

            // faked name and port for example
            cli, err := NewClient(config, "localhost", name, 8080)
            if err != nil {
                return err
            }

            if err := cli.Register([]string{"Go", "Awesome"}); err !=   
            nil {
                return err
            }

            entries, _, err := cli.Service(name, "Go")
            if err != nil {
                return err
            }
            for _, entry := range entries {
                fmt.Printf("%#v\n", entry.Service)
            }

            return nil
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容。确保你修改 `channels` 导入以使用你在步骤 2 中设置的路径：

```go
        package main

        import "github.com/agtorre/go-cookbook/chapter10/discovery"

        func main() {
            if err := discovery.Exec(); err != nil {
                panic(err)
            }
        }

```

1.  在一个单独的终端中使用 `consul agent -dev -node=localhost` 命令启动 Consul。

1.  运行 `go run main.go` 命令。

1.  你还可以运行：

```go
      go build
      ./example

```

你应该看到以下输出：

```go
 $ go run main.go
 &api.AgentService{ID:"discovery", Service:"discovery", Tags:    
      []string{"Go", "Awesome"}, Port:8080, Address:"localhost",     
      EnableTagOverride:false, CreateIndex:0x23, ModifyIndex:0x23}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

Consul 提供了一个健壮的 Go API 库。一开始可能会觉得有些令人畏惧，但这个配方展示了你如何接近封装它。配置 Consul 超出了本配方的范围，但本配方展示了注册服务和根据键和标签查询其他服务的基本方法。

使用这个方法可以在启动时注册新的微服务，查询所有依赖的服务，并在关闭时注销。你可能还希望缓存这些信息，这样你就不必为每个请求都调用 Consul，但本配方提供了你可以扩展的基本工具。Consul 代理还使这些重复请求变得快速高效（[`www.consul.io/intro/getting-started/agent.html`](https://www.consul.io/intro/getting-started/agent.html)）。

# 使用 Raft 实现基本共识

Raft 是一种共识算法，允许分布式系统保持共享和管理状态（[`raft.github.io/`](https://raft.github.io/)）。在许多方面设置 Raft 系统都是复杂的，例如，你需要达成共识以进行选举并成功。当与多个节点一起工作时，这可能很难启动，也可能很难开始。一个基本的集群可以在单个节点/领导者上运行，但如果你想要冗余，至少需要三个节点以允许单个节点故障。

本配方实现了一个基本的内存 Raft 集群，构建了一个可以在某些允许的状态之间转换的状态机，并将分布式状态机连接到一个可以触发转换的 Web 处理器。当你实现 Raft 所需的基本有限状态机接口或进行测试时，这可能很有用。本配方使用 [`github.com/hashicorp/raft`](https://github.com/hashicorp/raft) 作为基本的 Raft 实现。

# 准备工作

根据以下步骤配置你的环境：

1.  参考本章中 *使用 Consul 进行服务发现* 配方的 *准备工作* 部分。

1.  运行 `go get github.com/hashicorp/raft` 命令。

# 如何做到这一点...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter10/consensus` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter10/consensus`](https://github.com/agtorre/go-cookbook/tree/master/chapter10/consensus) 复制测试，或者将其作为练习编写一些自己的代码。

1.  创建一个名为 `state.go` 的文件，内容如下：

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

1.  创建一个名为 `config.go` 的文件，内容如下：

```go
        package consensus

        import "github.com/hashicorp/raft"

        var rafts map[string]*raft.Raft

        func init() {
            rafts = make(map[string]*raft.Raft)
        }

        // Config creates num in-memory raft
        // nodes and connects them
        func Config(num int) {
            conf := raft.DefaultConfig()
            snapshotStore := raft.NewDiscardSnapshotStore()

            addrs := []string{}
            transports := []*raft.InmemTransport{}
            for i := 0; i < num; i++ {
                addr, transport := raft.NewInmemTransport("")
                addrs = append(addrs, addr)
                transports = append(transports, transport)
            }
            peerStore := &raft.StaticPeers{StaticPeers: addrs}
            memstore := raft.NewInmemStore()

            for i := 0; i < num; i++ {
                for j := 0; j < num; j++ {
                    if i != j {
                        transports[i].Connect(addrs[j], transports[j])
                    }
                }

                r, err := raft.NewRaft(conf, NewFSM(), memstore, 
                memstore, snapshotStore, peerStore, transports[i])
                if err != nil {
                    panic(err)
                }
                r.SetPeers(addrs)
                rafts[addrs[i]] = r
            }
        }

```

1.  创建一个名为 `fsm.go` 的文件，内容如下：

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

1.  创建一个名为 `handler.go` 的文件，内容如下：

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
            for k, rf := range rafts {
                if k == rf.Leader() {
                    state := r.FormValue("next")
                    result := rf.Apply([]byte(state), 1*time.Second)
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
                    w.Write([]byte(result.Response().(string)))
                    return
                }
            }
        }

```

1.  创建一个名为 `example` 的新目录，并导航到它。

1.  创建一个名为 `main.go` 的文件，内容如下。确保你修改 `channels` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
            "net/http"

            "github.com/agtorre/go-cookbook/chapter10/consensus"
        )

        func main() {
            consensus.Config(3)

            http.HandleFunc("/", consensus.Handler)
            err := http.ListenAndServe(":3333", nil)
            panic(err)
        }

```

1.  运行 `go run main.go` 命令。或者，你也可以运行以下命令：

```go
 go build
 ./example

```

通过运行前面的命令，你现在应该会看到以下输出：

```go
 $ go run main.go
 2017/04/23 16:49:24 [INFO] raft: Node at 95c86c4c-9192-a8a6-  
      5e38-66c033bb3955 [Follower] entering Follower state (Leader:   
      "")
 2017/04/23 16:49:24 [INFO] raft: Node at 2406e36b-7e3e-0965- 
      8863-70a5dc1a2e69 [Follower] entering Follower state (Leader: 
      "")
 2017/04/23 16:49:24 [INFO] raft: Node at 2b5367e6-eea6-e195-  
      df40-1aeebfe8cdc7 [Follower] entering Follower state (Leader:   
      "")
 2017/04/23 16:49:25 [WARN] raft: Heartbeat timeout from ""   
      reached, starting election
 2017/04/23 16:49:25 [INFO] raft: Node at 2406e36b-7e3e-0965-  
      8863-70a5dc1a2e69 [Candidate] entering Candidate state
 2017/04/23 16:49:25 [DEBUG] raft: Votes needed: 2
 2017/04/23 16:49:25 [DEBUG] raft: Vote granted from 2406e36b-
      7e3e-0965-8863-70a5dc1a2e69\. Tally: 1
 2017/04/23 16:49:25 [DEBUG] raft: Vote granted from 95c86c4c-  
      9192-a8a6-5e38-66c033bb3955\. Tally: 2
 2017/04/23 16:49:25 [INFO] raft: Election won. Tally: 2
 2017/04/23 16:49:25 [INFO] raft: Node at 2406e36b-7e3e-0965-  
      8863-70a5dc1a2e69 [Leader] entering Leader state
 2017/04/23 16:49:25 [INFO] raft: pipelining replication to peer   
      95c86c4c-9192-a8a6-5e38-66c033bb3955
 2017/04/23 16:49:25 [INFO] raft: pipelining replication to peer   
      2b5367e6-eea6-e195-df40-1aeebfe8cdc7
 2017/04/23 16:49:25 [DEBUG] raft: Node 2406e36b-7e3e-0965-8863- 
      70a5dc1a2e69 updated peer set (2): [2406e36b-7e3e-0965-8863- 
      70a5dc1a2e69 95c86c4c-9192-a8a6-5e38-66c033bb3955 2b5367e6-
      eea6-e195-df40-1aeebfe8cdc7]
 2017/04/23 16:49:25 [DEBUG] raft: Node 95c86c4c-9192-a8a6-5e38-  
      66c033bb3955 updated peer set (2): [2406e36b-7e3e-0965-8863-  
      70a5dc1a2e69 95c86c4c-9192-a8a6-5e38-66c033bb3955 2b5367e6-
      eea6-e195-df40-1aeebfe8cdc7]
 2017/04/23 16:49:25 [DEBUG] raft: Node 2b5367e6-eea6-e195-df40- 
      1aeebfe8cdc7 updated peer set (2): [2406e36b-7e3e-0965-8863-  
      70a5dc1a2e69 95c86c4c-9192-a8a6-5e38-66c033bb3955 2b5367e6-  
      eea6-e195-df40-1aeebfe8cdc7]

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

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

当应用程序启动时，我们初始化多个 Raft 对象。每个对象都有自己的地址和传输。`InmemTransport{}` 函数还提供了一个连接其他传输的方法，称为 `Connect()`。一旦建立这些连接，Raft 集群就会进行选举。当与 Raft 集群通信时，客户端必须与领导者通信。在我们的情况下，一个处理器可以与所有节点通信，因此处理器负责让领导者 `Raft` 对象调用 `Apply()`。这反过来在所有其他节点上运行 `apply()`。

此配方不处理快照，仅关注有限状态机（FSM）状态的变化。

`InmemTransport{}` 函数通过允许所有内容都驻留在内存中来简化选举和引导过程。在实践中，这除了测试和概念验证之外并不很有用，因为 go 线程可以自由访问共享内存。

# 使用 Docker 进行容器化

Docker 是一种用于打包和运输应用程序的容器技术。其他优点包括可移植性，容器将在宿主操作系统上以相同的方式运行。它提供了虚拟机的大部分优点，但更轻量级的容器。可以限制单个容器的资源消耗并沙盒化环境。对于在本地为应用程序创建一个通用环境以及将代码部署到生产环境时非常有用。Docker 用 Go 编写且是开源的，因此可以利用客户端和库。本配方将为基本的 Go 应用程序设置 Docker 容器，存储有关容器的某些版本信息，并演示从 Docker 端点调用处理器。

# 准备工作

根据以下步骤配置你的环境：

1.  参考本章中 *使用服务发现进行 Consul* 配方的 *准备工作* 部分。

1.  从[`store.docker.com/search?type=edition&offering=community`](https://store.docker.com/search?type=edition&offering=community)安装 Docker。这将包括 Docker Compose。

# 如何做...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建`chapter10/docker`目录并导航到它。

1.  从[`github.com/agtorre/go-cookbook/tree/master/chapter10/docker`](https://github.com/agtorre/go-cookbook/tree/master/chapter10/docker)复制测试或使用这个练习来编写一些你自己的代码。

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

1.  创建一个名为`example`的新目录并导航到它。

1.  创建一个名为`main.go`的文件，内容如下。确保你修改了`channels`导入以使用你在步骤 2 中设置的路径：

```go
        package main

        import (
            "fmt"
            "net/http"
            "strconv"
            "time"

            "github.com/agtorre/go-cookbook/chapter10/docker"
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

1.  返回到起始目录。

1.  运行以下命令：

```go
 $ bash setup.sh

```

你现在应该看到以下输出：

```go
 $ bash setup.sh
 ~/go/src/github.com/agtorre/go- 
      cookbook/chapter10/docker/example   
      ~/go/src/github.com/agtorre/go-cookbook/chapter10/docker
 ~/go/src/github.com/agtorre/go-cookbook/chapter10/docker
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

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行`go test`。确保所有测试都通过。

# 它是如何工作的...

这个菜谱创建了一个脚本，用于编译 Linux 架构的 Go 二进制文件，并在`main.go`中设置各种私有变量。这些变量用于在版本端点上返回版本信息。一旦编译了二进制文件，就会创建一个包含二进制文件的 Docker 容器。这允许我们使用非常小的容器镜像，因为 Go 运行时在二进制文件中是自包含的。然后我们运行容器，同时暴露容器监听 HTTP 流量的端口。最后，我们在 localhost 上 curl 该端口，并看到返回的版本信息。

# 编排和部署策略

Docker 使得编排和部署变得更加简单。在这个菜谱中，我们将设置一个连接到 MongoDB 的连接，从 Docker 容器中插入文档并查询它。这个菜谱将设置与第五章中“所有关于数据库和存储”的*使用 NoSQL 与 MongoDB 和 mgo*菜谱相同的相同环境，但将在容器内运行应用程序和环境，并使用 Docker Compose 来编排和连接它们。这可以后来与 Docker Swarm 结合使用，这是一个集成的 Docker 工具，允许你管理一个集群，创建和部署可以轻松扩展或缩减的节点，并管理负载均衡([`docs.docker.com/engine/swarm/`](https://docs.docker.com/engine/swarm/))。容器编排的另一个好例子是 Kubernetes([`kubernetes.io/`](https://kubernetes.io/))，这是一个由 Google 使用 Go 编程语言编写的容器编排框架。

# 准备工作

根据以下步骤配置你的环境：

1.  参考使用 Docker 进行容器化的*准备工作*部分。

1.  执行 `go get gopkg.in/mgo.v2` 命令。

1.  执行 `go get github.com/tools/godep` 命令。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  在你的终端/控制台应用程序中，创建 `chapter10/orchestrate` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter10/orchestrate`](https://github.com/agtorre/go-cookbook/tree/master/chapter10/orchestrate) 复制测试或使用此作为练习编写一些你自己的代码。

1.  创建一个名为 `dockerfile` 的文件，内容如下：

```go
        FROM golang:alpine

        ENV GOPATH /code/
        ADD . /code/src/github.com/agtorre/go-   
        cookbook/chapter10/docker
        WORKDIR /code/src/github.com/agtorre/go-    
        cookbook/chapter10/docker/example
        RUN go build

        ENTRYPOINT /code/src/github.com/agtorre/go-  
        cookbook/chapter10/docker/example/example

```

1.  创建一个名为 `docker-compose.yml` 的文件，内容如下：

```go
        version: '2'
        services:
         app:
         build: .
         mongodb:
         image: "mongo:latest"

```

1.  创建一个名为 `mongo.go` 的文件，内容如下：

```go
        package orchestrate

        import (
            "fmt"

            mgo "gopkg.in/mgo.v2"
            "gopkg.in/mgo.v2/bson"
        )

        // State is our data model
        type State struct {
            Name string `bson:"name"`
            Population int `bson:"pop"`
        }

        // ConnectAndQuery connects, inserts a document, then
        // queries it
        func ConnectAndQuery(session *mgo.Session) error {
            conn := session.DB("gocookbook").C("example")

            // we can inserts many rows at once
            if err := conn.Insert(&State{"Washington", 7062000}, 
            &State{"Oregon", 3970000}); err != nil {
                return err
            }

            var s State
            if err := conn.Find(bson.M{"name": "Washington"}).One(&s); 
            err!= nil {
                return err
            }
            fmt.Printf("State: %#v\n", s)
            return nil
        }

```

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个 `main.go` 文件，内容如下。确保你修改 `orchestrate` 导入以使用步骤 2 中设置的路径：

```go
        package main

        import (
             "github.com/agtorre/go-cookbook/chapter10/orchestrate"
             mgo "gopkg.in/mgo.v2"
        )

        func main() {
            session, err := mgo.Dial("mongodb")
            if err != nil {
                panic(err)
            }
            if err := orchestrate.ConnectAndQuery(session); err != nil 
            {
                panic(err)
            }
        }

```

1.  返回到起始目录。

1.  执行 `godep save ./...` 命令。

1.  执行 `docker-compose up -d` 命令。

1.  执行 `docker logs docker_app_1` 命令。

你现在应该看到以下输出：

```go
 $ docker logs docker_app_1
 State: docker.State{Name:"Washington", Population:7062000}

```

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 工作原理...

这种配置适合本地开发。一旦运行了 `docker-compose up` 命令，本地目录将被重建，它将使用最新版本建立与 MongoDB 实例的连接，并开始对其操作。这个菜谱使用 godeps 进行依赖管理，因此不需要通过 `Dockerfile` 文件挂载整个 `GOPATH` 环境变量。

这在开始需要连接到外部服务的应用程序时可以提供一个良好的基线，所有 第五章，*关于数据库和存储的一切*，都可以使用这种方法，而不是创建数据库的本地实例。对于生产环境，你可能不希望在 Docker 容器后面运行数据存储，但你通常也会有静态的主机名用于配置。

# 监控应用程序

监控 Go 应用程序有多种方法。其中一种最简单的方法是设置 Prometheus，这是一个用 Go 编写的监控应用程序 ([`prometheus.io`](https://prometheus.io))。这是一个根据你的配置文件轮询端点的应用程序，并收集大量关于你的应用程序的信息，包括 goroutine 的数量、内存使用情况等等。此应用程序将使用前一个菜谱中的技术来设置一个 Docker 环境，以托管 Prometheus 并连接到它。

# 准备工作

根据以下步骤配置你的环境：

1.  参考使用 Docker 容器化的 *Using containerization with Docker* 菜谱中的 *Getting ready* 部分。

1.  执行 `go get github.com/prometheus/client_golang/prometheus/promhttp` 命令。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter10/monitoring` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter10/monitoring`](https://github.com/agtorre/go-cookbook/tree/master/chapter10/monitoring) 复制测试或使用此作为练习编写一些自己的代码。

1.  创建一个名为 `Dockerfile` 的文件，并包含以下内容：

```go
        FROM golang:alpine

        ENV GOPATH /code/
        ADD . /code/src/github.com/agtorre/go-
        cookbook/chapter10/monitoring
        WORKDIR /code/src/github.com/agtorre/go-
        cookbook/chapter10/monitoring
        RUN go build

        ENTRYPOINT /code/src/github.com/agtorre/go-
        cookbook/chapter10/monitoring/monitoring

```

1.  创建一个名为 `docker-compose.yml` 的文件，并包含以下内容：

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

1.  创建一个名为 `main.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `prometheus.yml` 的文件，并包含以下内容：

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

1.  运行 `godep save ./...` 命令。

1.  运行 `docker-compose up -d` 命令。

你现在应该看到以下内容：

```go
 $ docker-compose up
 Creating monitoring_app_1
 Creating monitoring_prometheus_1
 Attaching to monitoring_app_1, monitoring_prometheus_1
 prometheus_1 | time="2017-04-30T02:35:17Z" level=info 
      msg="Starting prometheus (version=1.6.1, branch=master,       
      revision=4666df502c0e239ed4aa1d80abbbfb54f61b23c3)" 
      source="main.go:88" 
 prometheus_1 | time="2017-04-30T02:35:17Z" level=info msg="Build       
      context (go=go1.8.1, user=root@7e45fa0366a7, date=20170419-
      14:32:22)" source="main.go:89" 
 prometheus_1 | time="2017-04-30T02:35:17Z" level=info 
      msg="Loading configuration file /etc/prometheus/prometheus.yml"       
      source="main.go:251" 
 prometheus_1 | time="2017-04-30T02:35:17Z" level=info 
      msg="Loading series map and head chunks..."      
      source="storage.go:421" 
 prometheus_1 | time="2017-04-30T02:35:17Z" level=info msg="0       
      series loaded." source="storage.go:432" 
 prometheus_1 | time="2017-04-30T02:35:17Z" level=info 
      msg="Starting target manager..." source="targetmanager.go:61" 
 prometheus_1 | time="2017-04-30T02:35:17Z" level=info 
      msg="Listening on :9090" source="web.go:259" 

```

1.  将你的浏览器导航到 `http://localhost:9090/`。你应该能看到与你的应用程序相关的各种指标！

# 它是如何工作的...

Prometheus 客户端处理程序将返回有关你的应用程序的各种统计信息到 Prometheus 服务器。这允许你将多个 Prometheus 服务器指向应用程序，而无需重新配置或部署应用程序。大多数这些统计信息是通用的，对检测内存泄漏等事物有益。许多其他解决方案需要你定期向服务器发送信息。下一个配方 *收集指标* 将演示如何将自定义指标发送到 Prometheus 服务器。

# 收集指标

除了关于你的应用程序的一般信息之外，发出特定于应用程序的指标可能很有帮助。例如，我们可能想要收集时间数据或跟踪事件发生的次数。

此配方将使用 `github.com/rcrowley/go-metrics` 包来收集指标并通过端点公开它们。有各种导出工具可以将指标导出到 Prometheus 和 InfluxDB 等地方，这些工具也用 Go 编写。

# 准备就绪

根据以下步骤配置你的环境：

1.  参考本章中 *使用 Consul 进行服务发现* 配方的 *准备就绪* 部分。

1.  运行 `go get github.com/rcrowley/go-metrics` 命令。

# 如何操作...

这些步骤涵盖了编写和运行你的应用程序：

1.  从你的终端/控制台应用程序中，创建 `chapter10/metrics` 目录并导航到它。

1.  从 [`github.com/agtorre/go-cookbook/tree/master/chapter10/metrics`](https://github.com/agtorre/go-cookbook/tree/master/chapter10/metrics) 复制测试或使用此作为练习编写一些自己的代码。

1.  创建一个名为 `handler.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `report.go` 的文件，并包含以下内容：

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

1.  创建一个名为 `example` 的新目录并导航到它。

1.  创建一个名为 `main.go` 的文件，并包含以下内容。确保将 `channels` 导入修改为你在第二步中设置的路径：

```go
        package main

        import (
            "net/http"

            "github.com/agtorre/go-cookbook/chapter10/metrics"
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

1.  运行 `go run main.go`。或者，你也可以运行以下命令：

```go
 go build ./example

```

你现在应该看到以下内容：

```go
 $ go run main.go
 listening on :8080

```

1.  在单独的 shell 中运行以下命令：

```go
 $ curl localhost:8080/counter 
 success

 $ curl localhost:8080/timer 
 success

 $ curl localhost:8080/report 
 {"counterhandler.counter":{"count":1},
      "reporthandler.writemetrics":      {"15m.rate":0,"1m.rate":0,"5m.
      rate":0,"75%":0,"95%":0,"99%":0,"99.9%":0,"count":0,"max":0,"mean
      ":0,"mean.rate":0,"median":0,"min":0,"stddev":0},"timerhandler.ti
      mer":{"15m.rate":0.0011080303990206543,"1m.rate"
      :0.015991117074135343,"5m.rate":0.0033057092356765017,"75%":60485
      ,"95%":60485,"99%":60485,"99.9%":60485,"count":1,"max":60485,"mea
      n":60485,"mean.rate":1.1334543719787356,"median":60485,"min":6048
      5,"stddev":0}}

```

1.  尝试多次访问所有端点，看看它们是如何变化的。

1.  如果你复制或编写了自己的测试，请向上移动一个目录并运行 `go test`。确保所有测试都通过。

# 它是如何工作的...

Gometrics 将所有度量存储在一个注册表中。一旦设置好，你可以使用任何度量选项，例如计数器或计时器，它将这个更新存储在注册表中。有多个导出器可以将度量导出到第三方工具。在我们的案例中，我们设置了一个处理器，将所有度量以 JSON 格式导出。

我们设置了三个处理器——一个用于增加计数器，一个用于记录退出处理器的时间，还有一个用于打印报告（同时也会增加一个额外的计数器）。`GetOrRegister` 函数在原子操作中获取或创建一个度量发射器非常有用，如果它当前不存在于线程安全的方式中。或者，你也可以预先一次性注册所有内容。
