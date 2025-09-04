# 12

# 附录：构建测试环境

本书每一章都包含 Go 代码示例，以说明我们在文本中提出的一些观点。您可以在本书的 GitHub 仓库中找到所有这些 Go 程序（见本章的*进一步阅读*部分）。虽然您不必执行所有这些程序，但我们相信手动运行代码并观察结果可能有助于巩固所学内容并解释更细微的细节。

本书的前一部分，*第一章*到*第五章*，包括相对简短的代码示例，您可以在 Go Playground（*进一步阅读*）或任何已安装 Go 的计算机上运行。有关安装 Go 的说明，您可以参考*第一章*或遵循官方的下载和安装程序（*进一步阅读*）。

本书其余部分，从*第六章*开始，假设您能够与虚拟拓扑交互，我们通过`containerlab`（*进一步阅读*）在容器中运行它。本*附录*记录了构建测试环境的过程，包括`containerlab`的兼容版本和其他相关依赖项，以确保您在本书的任何章节中运行示例时都能获得无缝体验。

# 什么是测试环境？

主要目标是构建一个具有正确硬件和软件组合的环境，以满足执行代码示例的最小要求。我们基于您部署**虚拟机**（**VM**）的假设，因为我们意识到您可能不会在专用的裸机服务器上部署它。

当涉及到部署虚拟机进行测试（测试平台）时，您有两个选择，我们将在后面讨论：

+   您可以在自托管环境中部署此虚拟机，例如 VMware 或**基于内核的虚拟机**（**KVM**）。

+   您可以使用云托管环境——例如，**亚马逊网络服务**（**AWS**）。

从硬件角度来看，我们假设底层 CPU 架构是 64 位 x86，我们建议至少为虚拟机分配 2 个 vCPU 和 4GB 的 RAM，理想情况下加倍以提高速度。

我们在本书的 GitHub 仓库中描述了所有软件配置和配置（*进一步阅读*）。我们强烈建议您使用我们为您准备的自动化方法来安装所有依赖项，以便运行书中的代码示例。

您仍然可以在任何 Linux 发行版上安装这些软件包——例如，**Windows Subsystem for Linux 版本 2**（**WSL 2**）。如果您想手动进行安装，我们在此处包含了一个完整的依赖项列表：

| **包** | **版本** |
| --- | --- |
| Go | 1.18.1 |
| `containerlab` | 0.25.1 |
| Docker | 20.10.14 |
| `ansible-core`（仅适用于*第七章*） | 2.12.5 |
| Terraform（仅适用于*第七章*） | 1.1.9 |

表 12.1 – 软件依赖

## 第 1 步 – 构建测试环境

在下一节中，我们描述了构建测试环境的两种自动化方法。如果您不确定哪个选项适合您，我们建议您选择第一个，因为它具有最少的依赖项，并且完全由云服务提供商管理。这也是我们（本书的作者）唯一可以测试和验证的选项，因此它应该为您提供最一致的经验。

### 选项 1 – 云托管

我们选择 AWS 作为云服务提供商，因为它在我们的行业中非常受欢迎，并且普遍熟悉。在本书的 GitHub 仓库 (*进一步阅读*) 中，我们包含了一个 Ansible 剧本，该剧本完全自动化了在 AWS 中创建虚拟机所需的所有任务。您可以使用任何其他云服务提供商，但您将不得不手动进行配置。

测试环境是一个在 AWS 运行的单个 Linux 虚拟机，使用 `containerlab` 创建基于容器的网络拓扑。下一张图展示了 AWS 环境的样子：

![图 12.1 – 目标环境](img/B16971_12_01.jpg)

图 12.1 – 目标环境

为了符合之前声明的硬件要求，我们建议您至少运行一个 `t2.medium`-大小的虚拟机，理想情况下运行一个 `t2.large`-大小的虚拟机（**弹性计算云**（**EC2**）实例）。但是，AWS 免费层计划 (*进一步阅读*) 不包括这些实例类型，因此您应该预计会因虚拟机的运行而产生一些费用。我们假设您熟悉 AWS 的成本和计费结构，并在使用云托管环境时使用财务常识。

在您运行剧本之前，您需要确保您满足以下要求：

1.  创建一个 AWS 账户（AWS 免费层 (*进一步阅读*)）。

1.  创建一个 AWS 访问密钥（AWS 程序化访问 (*进一步阅读*)）。

1.  一个具有以下软件包的 Linux 操作系统：

    +   Git

    +   Docker

    +   GNU Make

在所有这些准备就绪后，您可以使用 `git clone` 命令继续克隆本书的 GitHub 仓库 (*进一步阅读*)：

```go
$ git clone https://github.com/PacktPublishing/Network-Automation-with-Go
```

在您克隆了仓库之后，切换到该目录。

#### 输入变量

在您开始部署之前，您需要提供您的 AWS 账户凭证（`AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY`）。您可以通过导出一对包含密钥 ID 和密钥值的环境变量来完成此操作，如下所示。有关如何创建访问密钥的说明，请参阅 AWS 程序化访问 (*进一步阅读*)：

```go
$ export AWS_ACCESS_KEY_ID='…'
$ export AWS_SECRET_ACCESS_KEY='…'
```

除了这些必需的变量之外，还有其他三个可选输入变量，您可以调整这些变量以微调您的部署环境：

| **名称** | **值** |
| --- | --- |
| `AWS_DISTRO` | `fedora` 或 `ubuntu`（默认：`fedora`） |
| `AWS_REGION` | AWS 区域之一（默认：`us-east-1`） |
| `VM_SIZE` | AWS 实例类型之一（默认：`t2.large`） |

表 12.2 – 测试虚拟机选项

如果您选择更改这些默认值，您可以像创建 AWS 访问密钥一样进行操作。以下是一个示例：

```go
$ export AWS_DISTRO=ubuntu
$ export AWS_REGION=eu-west-2
```

在那种情况下，我们选择了 Ubuntu 作为虚拟机的 Linux 发行版，并将伦敦（`eu-west-2`）作为部署的 AWS 区域。

#### 部署过程

一旦你设置了所有必需的输入变量，你可以部署测试环境。在书仓库目录内，运行 `make env-build` 命令，该命令将部署虚拟机并安装所有必需的软件包：

```go
Network-Automation-with-Go$ make env-build
AWS_ACCESS_KEY_ID is AKIAVFPUEFZCFVFGXXXX
AWS_SECRET_ACCESS_KEY is **************************
Using /etc/ansible/ansible.cfg as config file
PLAY [Create EC2 instance] *************************************************************************************************************************************************************
TASK [Gathering Facts] *****************************************************************************************************************************************************************
ok: [localhost]
### ... <omitted for brevity > ... ###
TASK [Print out instance information for the user] *************************************************************************************************************************************
ok: [testbed] => {}
MSG:
['SSH: ssh -i lab-state/id_rsa fedora@ec2-54-86-51-96.compute-1.amazonaws.com\n', 'To upload cEOS image: scp -i lab-state/id_rsa ~/Downloads/cEOS64-lab-4.28.0F.tar fedora@ec2-54-86-51-96.compute-1.amazonaws.com:./network-automation-with-go\n']
PLAY RECAP *****************************************************************************************************************************************************************************
localhost                  : ok=28   changed=9    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
testbed                    : ok=36   changed=24   unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
```

假设 Playbook 已成功完成，你可以在日志中看到虚拟机访问详情，如前面的输出所示。你还可以在部署环境后运行 `make env-show` 命令来查看连接详情：

```go
Network-Automation-with-Go$ make env-show
fedora@ec2-54-86-51-96.compute-1.amazonaws.com
```

现在，你可以使用这些信息来连接到已配置的虚拟机。Playbook 生成一个 `lab-state/id_rsa`），所以请务必始终使用它进行 SSH 认证：

```go
Network-Automation-with-Go$ ssh -i lab-state/id_rsa fedora@ec2-54-86-51-96.compute-1.amazonaws.com 
fedora@testbed:~$  go version
go version go1.18.1 linux/amd64
fedora@testbed:~$  ls network-automation-with-go/
LICENSE  Makefile  README.md  ch01  ch02  ch03  ch04  ch05  ch06  ch07  ch08  ch09  ch10  ch12  lab-state  topo-base  topo-full
```

你可以连接到虚拟机并检查已安装的 Go 版本，并查看书仓库的文件。

### 选项 2 – 自托管

另一个选项是在私有环境中创建虚拟机。这个环境可以是运行虚拟机管理程序（如 VirtualBox、ESXi 服务器、OpenStack 集群或其他任何可以分配虚拟机运行实验室拓扑所需的 CPU 和内存的设备）。虚拟机上的操作系统必须是 Ubuntu 22.04 或 Fedora 35。

一旦你构建了带有 SSH 服务的虚拟机，确保你可以通过虚拟机的 IP 地址 SSH 连接到它，并使用其凭证访问。然后，将个人电脑上此书 GitHub 仓库的 `ch12/testbed` 文件夹（*进一步阅读*）中的 Ansible inventory 文件（`inventory`）更改为指向你的虚拟机。它应该看起来像这样：

```go
# inventory
[local-vm]
192.168.122.18
[local-vm:vars]
ansible_user=fedora
ansible_password=fedora
ansible_sudo_pass=fedora
```

至少包括到达虚拟机的 IP 地址（`ansible_host`），以及 `ansible_user`、`ansible_password` 或 `ansible_ssh_private_key_file` 用户凭证。

在相同的 `ch12/testbed` 文件夹（*进一步阅读*）中，有一个 Ansible Playbook 调用 `configure_instance` 角色。使用此 Playbook 自动配置你的虚拟机以运行书中的示例，如下所示：

```go
# configure-local-vm.yml
- name: Configure Instance(s)
  hosts: local-vm
  gather_facts: true
  vars_files:
    - ./vars/go_inputs.yml
    - ./vars/clab_inputs.yml
    - ./vars/aws_common.yml
  roles:
    - {role: configure_instance, become: true}
```

Playbook 文件名为 `configure-local-vm.yml`，inventory 文件名为 `inventory`，因此从 `ch12/testbed` 文件夹（*进一步阅读*）中运行 `ansible-playbook configure-local-vm.yml -i inventory -v` 以准备虚拟机。

## 第 2 步 – 上传容器镜像

并非所有网络供应商都使访问基于容器的 **网络操作系统（NOSes**）变得简单。如果你不能直接从容器注册库（如 Docker Hub）拉取镜像，你可能需要从他们的网站下载镜像并将其上传到测试虚拟机。在撰写本书时，书中唯一无法从公共注册库拉取的容器镜像是我们无法拉取的 Arista 的 **cEOS** 镜像。在这里，我们描述了将此镜像上传到测试环境的过程。

你需要做的第一件事是从[arista.com](http://arista.com)下载镜像（*进一步阅读*）。你应该从 4.28(F)系列中选择 64 位 cEOS 镜像——例如，`cEOS64-lab-4.28.0F.tar`。你可以使用生成的 SSH 私钥通过`scp`命令将镜像复制到测试虚拟机：

```go
Network-Automation-with-Go$ scp -i lab-state/id_rsa ~/Downloads/cEOS64-lab-4.28.0F.tar fedora@ec2-54-86-51-96.compute-1.amazonaws.com:./network-automation-with-go
cEOS64-lab-4.28.0F.tar                        100%  434MB  26.6MB/s   00:16
```

然后，通过 SSH 连接到实例，并使用`docker`命令导入镜像：

```go
Network-Automation-with-Go$ ssh -i lab-state/id_rsa fedora@ec2-54-86-51-96.compute-1.amazonaws.com
fedora@testbed:~$  cd network-automation-with-go 
fedora@testbed:~$  docker import cEOS64-lab-4.28.0F.tar ceos:4.28
sha256:dcdc721054804ed4ea92f970b5923d8501c28526ef175242cfab0d1 58ac0085c
```

你现在可以在拓扑文件中一个或多个路由器的`image`部分使用这个镜像（`ceos:4.28`）。

## 第 3 步 – 与测试环境交互

我们建议你在*第六章*至*第八章*的开始处使用虚拟网络拓扑的新构建。为了编排拓扑，我们使用`containerlab`，它在测试虚拟机中可用。`containerlab`提供了一种快速运行基于它们在可读 YAML 文件中提供的定义的任意网络拓扑的方法。

重要提示

`containerlab`是用 Go 语言编写的，是一个交互式 CLI 程序的优秀示例，它可以编排本地容器资源。

你可以在本书 GitHub 仓库的`topo-base`目录中找到以下`base`拓扑定义文件（*进一步阅读*）：

```go
name: netgo
topology:
  nodes:
    srl:
      kind: srl
      image: ghcr.io/nokia/srlinux:21.6.4
    ceos:
      kind: ceos
      image: ceos:4.28.0F
      startup-config: ceos-startup
    cvx:
      kind: cvx
      image: networkop/cx:5.0.0
      runtime: docker
  links:
    - endpoints: ["srl:e1-1", "ceos:eth1"]
    - endpoints: ["cvx:swp1", "ceos:eth2"]
```

此 YAML 文件定义了一个三节点拓扑，如下一个图表所示。一个节点运行诺基亚 SR Linux，另一个运行 NVIDIA Cumulus Linux，最后一个运行 Arista cEOS。在这种情况下，所有网络设备都使用它们的默认启动配置，在每个章节中，我们都描述了如何在这三个设备之间建立完整的端到端可达性：

![图 12.2 – “Base”网络拓扑](img/B16971_12_02.jpg)

图 12.2 – “Base”网络拓扑

接下来的两个章节（*第九章*和*第十章*）依赖于前面拓扑的略微不同版本。与`base`拓扑不同，`full`拓扑完全配置并包含一组额外的节点来模拟连接到网络设备的物理服务器：

![图 12.3 – “Full”网络拓扑](img/B16971_12_03.jpg)

图 12.3 – “Full”网络拓扑

这些终端主机运行不同的应用程序，它们与现有的网络拓扑进行交互。

# 启动虚拟网络拓扑

你可以使用`containerlab`二进制文件来部署测试拓扑。为了方便，我们包含了一些`make`目标，你可以使用：

+   `make lab-base`来创建在*第六章*至*第八章*中使用的`base`拓扑。

+   使用`make lab-full`创建在*第九章*和*第十章*中使用的`full`拓扑。

以下是如何在测试虚拟机内部创建`base`拓扑的示例：

```go
fedora@testbed network-automation-with-go$ make lab-base
...
+---+-----------------+--------------+--------------
| # | Name            | Container ID | Image
+---+-----------------+--------------+--------------
| 1 | clab-netgo-ceos | fe422727f351 | ceos:4.28.0F
| 2 | clab-netgo-cvx  | 85e5b9135e1b | cx:5.0.0
| 3 | clab-netgo-srl  | 00106bef1d4e |srlinux:21.6.4
+---+-----------------+--------------+--------------
```

现在，你已经准备好了`clab-netgo-ceos`、`clab-netgo-cvx`和`clab-netgo-srl`路由器。

## 连接到设备

`containerlab`使用 Docker 运行容器。这意味着我们可以使用标准的 Docker 功能来连接到设备——例如，你可以使用`docker exec`命令在容器内启动任何进程：

```go
fedora@testbed:~$  docker exec -it clab-netgo-srl sr_cli
Welcome to the srlinux CLI.                      
A:srl# show version | grep Software
Software Version  : v21.6.4
```

在前面的示例中，`sr_cli` 是 SR Linux 设备的 CLI 进程。下表显示了每个虚拟网络设备的“默认 shell”进程：

| **NOS** | **命令** |
| --- | --- |
| Cumulus Linux | `bash` 或 `vtysh` |
| SR Linux | `sr_cli` |
| EOS | `Cli` |

表 12.3 – 设备默认 shell

您也可以使用 SSH 连接到默认 shell。下表提供了连接到每个设备的主机名和相应的凭证：

| **设备** | **用户名** | **密码** |
| --- | --- | --- |
| `clab-netgo-srl` | `admin` | `admin` |
| `clab-netgo-ceos` | `admin` | `admin` |
| `clab-netgo-cvx` | `cumulus` | `cumulus` |

表 12.4 – 设备凭证

例如，这是您连接到 Arista cEOS 和 Cumulus Linux 的方法：

```go
fedora@testbed:~$  ssh admin@clab-netgo-ceos
(admin@clab-netgo-ceos) Password: admin
ceos>en
ceos#exit
fedora@testbed:~$
fedora@testbed:~$  ssh cumulus@clab-netgo-cvx
cumulus@clab-netgo-cvx's password: cumulus
Welcome to NVIDIA Cumulus (R) Linux (R)
cumulus@cvx:mgmt:~$
```

一旦您完成本章，您可以销毁拓扑。

## 销毁网络拓扑

您可以使用 `make cleanup` 命令清理虚拟网络拓扑：

```go
fedora@testbed:~/network-automation-with-go$ make cleanup
```

`make cleanup` 命令仅清理虚拟网络拓扑，而所有云资源仍在运行。

## 第 4 步 – 清理云托管环境

一旦您完成与云托管测试环境的工作，您可以清理它，这样您就不会为可能不再需要的东西付费。您可以使用另一个 Ansible playbook 来确保您之前创建的所有 AWS 资源现在都被清除：

```go
etwork-Automation-with-Go$ make env-delete
AWS_ACCESS_KEY_ID is AKIAVFPUEFZCFVFGXXXX
AWS_SECRET_ACCESS_KEY is **************************
PLAY [Delete EC2 instance] *************************************************************************************************************************************************************
TASK [Gathering Facts] *****************************************************************************************************************************************************************
ok: [localhost]
### ... <omitted for brevity > ... ###
TASK [Cleanup state files] *************************************************************************************************************************************************************
changed: [localhost] => (item=.region)
changed: [localhost] => (item=.vm)
PLAY RECAP *****************************************************************************************************************************************************************************
localhost                  : ok=21   changed=8    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

# 进一步阅读

+   书籍的 GitHub 仓库：[`github.com/PacktPublishing/Network-Automation-with-Go`](https://github.com/PacktPublishing/Network-Automation-with-Go)

+   Go Playground：[`play.golang.org/`](https://play.golang.org/)

+   官方下载和安装流程：[`golang.org/doc/install#install`](https://golang.org/doc/install#install)

+   `containerlab`：[`containerlab.dev/`](https://containerlab.dev/)

+   AWS 免费层：[`aws.amazon.com/free/`](https://aws.amazon.com/free/)

+   AWS 程序化访问：[`docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys`](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys)

+   `ch12/testbed`：[`github.com/PacktPublishing/Network-Automation-with-Go/blob/main/ch12/testbed`](https://github.com/PacktPublishing/Network-Automation-with-Go/blob/main/ch12/testbed)

+   `ch12/testbed/inventory`：[`github.com/PacktPublishing/Network-Automation-with-Go/blob/main/ch12/testbed/inventory`](https://github.com/PacktPublishing/Network-Automation-with-Go/blob/main/ch12/testbed/inventory)

+   Arista：[`www.arista.com/en/support/software-download`](https://www.arista.com/en/support/software-download)

+   初学者指南—下载 Python：[`wiki.python.org/moin/BeginnersGuide/Download`](https://wiki.python.org/moin/BeginnersGuide/Download)

+   使用 `pip` 安装 Ansible：[`docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-with-pip`](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-with-pip)

+   *入门 - 安装* *Git*：[`git-scm.com/book/en/v2/Getting-Started-Installing-Git`](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

+   安装 `pip`—*支持* *方法*：[`pip.pypa.io/en/stable/installation/#supported-methods`](https://pip.pypa.io/en/stable/installation/#supported-methods)

+   获取 Arista cEOS：[`github.com/PacktPublishing/Network-Automation-with-Go/blob/main/ch12/testbed/get_arista_ceos.md`](https://github.com/PacktPublishing/Network-Automation-with-Go/blob/main/ch12/testbed/get_arista_ceos.md)

+   AWS 访问密钥：[`docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys`](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys)

+   AWS 区域：[`docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html`](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)

+   AWS 实例类型：[`aws.amazon.com/ec2/instance-types/`](https://aws.amazon.com/ec2/instance-types/)
