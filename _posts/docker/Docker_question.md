title: 关于Docker的50个问与答
date: 2018/04/26 10:32:25
categories:
- Docker
tags:
- Docker

---


## 问：Docker 和 传统的虚拟机有什么差别？
**答：** 我们知道inux系统将自身划分为两部分，一部分为核心软件，也称作内核空间(`kernel`)，另一部分为普通应用程序，这部分称为用户空间(`userland`)。
容器内的进程是直接运行于宿主内核的，这点和宿主进程一致，只是容器的 `userland` 不同，容器的 `userland` 由容器镜像提供，也就是说镜像提供了 `rootfs`。
假设宿主是 `Ubuntu`，容器是 `CentOS`。`CentOS` 容器中的进程会直接向 `Ubuntu` 宿主内核发送 `syscall`，而不会直接或间接的使用任何 `Ubuntu` 的 `userland` 的库。
这点和虚拟机有本质的不同，虚拟机是虚拟环境，在现有系统上虚拟一套物理设备，然后在虚拟环境内运行一个虚拟环境的操作系统内核，在内核之上再跑完整系统，并在里面调用进程。
还以上面的例子去考虑，虚拟机中，`CentOS` 的进程发送 `syscall` 内核调用，该请求会被虚拟机内的 `CentOS` 的内核接到，然后 `CentOS` 内核访问虚拟硬件时，由虚拟机的服务软件截获，并使用宿主系统，也就是 `Ubuntu` 的内核及 `userland` 的库去执行。
而且，Linux 和 Windows 在这点上非常不同。Linux 的进程是直接发 `syscall` 的，而 Windows 则把 `syscall` 隐藏于一层层的 `DLL` 服务之后，因此 Windows 的任何一个进程如果要执行，不仅仅需要 Windows 内核，还需要一群服务来支撑，所以如果 Windows 要实现类似的机制，容器内将不会像 Linux 这样轻量级，而是非常臃肿。看一下微软移植的 Docker 就非常清楚了。
所以不要把 Docker 和虚拟机弄混，Docker 容器只是一个进程而已，只不过利用镜像提供的 `rootfs` 提供了调用所需的 `userland` 库支持，使得进程可以在受控环境下运行而已，它并没有虚拟出一个机器出来。

**Linux 容器不是模拟一个完整的操作系统，而是对进程进行隔离。**

## 问：如何安装 Docker？
**答：** 很多人问到 `docker`, `docker.io`, `docker-engine` 甚至 `lxc-docker` 都有什么区别？
其中，RHEL/CentOS 软件源中的 Docker 包名为 `docker`；Ubuntu 软件源中的 Docker 包名为 `docker.io`；而很古老的 Docker 源中 Docker 也曾叫做 `lxc-docker`。这些都是非常老旧的 Docker 版本，并且基本不会更新到最新的版本，而对于使用 Docker 而言，使用最新版本非常重要。另外，17.04 以后，包名从 `docker-engine` 改为 `docker-ce`，因此从现在开始安装，应该都使用 `docker-ce `这个包。
正确的安装方法有两种：

- 一种是参考官方安装文档去配置 apt 或者 yum 的源；
- 另一种则是使用官方提供的安装脚本快速安装。

官方文档对配置源的方法已经有很详细的讲解:[官方文档地址](https://docs.docker-cn.com/engine/installation/linux/docker-ce/centos/)

<!--more-->
**17.04 及以后的版本安装方法**：
从 `17.04` 以后，可以用下面的命令安装。

```bash
export CHANNEL=stable
curl -fsSL https://get.docker.com/ | sh -s -- --mirror Aliyun
```
这里使用的是官方脚本安装，通过环境变量指定安装通道为 `stable`，（如果喜欢尝鲜可以改为 `edge`, `test`），并且指定使用阿里云的源(`apt/yum`)来安装 `Docker CE` 版本。

**17.03 及以前的版本安装方法**：
早期的版本可以使用阿里云或者 DaoCloud 老的脚本安装：
使用`阿里云`的安装脚本：

```bash
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
```
使用`DaoCloud`的Docker安装脚本：

```bash
curl -sSL https://get.daocloud.io/docker | sh
```

**Centos7机器直接用RPM包安装方法**：
可去阿里云寻找到最新的rpm包并下载安装：[阿里云docker rpm地址]:(https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/)

```bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-18.03.0.ce-1.el7.centos.x86_64.rpm -O docker-ce-18.03.0.ce-1.el7.centos.x86_64.rpm
yum install -y container-selinux
rpm -ivh docker-ce-17.12.0.ce-1.el7.centos.x86_64.rpm
```


## 问：Docker中使用了那些核心技术？
**答：** `Linux Namespace`命名空间、`Linux CGroup`全称`Linux Control Group控制组`和 `UnionFS` 全称`Union File System联合文件系统`，三大技术支撑了目前 Docker 的实现，也是 Docker 能够出现的最重要原因。

### `Linux Namespace`是Linux提供的一种内核级别环境隔离的方法。
通过隔离要做到的效果是：如果某个 `Namespace` 中有进程在里面运行，它们只能看到该 `Namespace` 的信息，无法看到 `Namespace` 以外的东西。Linux 的命名空间机制提供了以下七种不同的命名空间，包括` CLONE_NEWCGROUP`、`CLONE_NEWIPC`、`CLONE_NEWNET`、`CLONE_NEWNS`、`CLONE_NEWPID`、`CLONE_NEWUSER` 和 `CLONE_NEWUTS`，通过这七个选项我们能在创建新的进程时设置新进程应该在哪些资源上与宿主机器进行隔离。这些 `Namespace` 基本上覆盖了一个程序运行所需的环境，保证运行在的隔离的 `Namespace` 中的，会让程序不会受到其他收到 `Namespace `程序的干扰。但不是所有的系统资源都能隔离，时间就是个例外，没有对应的 `Namespace`，因此同一台 Linux 启动的容器时间都是相同的。

名称	| 宏定义	| 隔离的内容
---------| --------- | ---------
IPC |	CLONE_NEWIPC |	System V IPC, POSIX message queues (since Linux 2.6.19)
Network	| CLONE_NEWNET	| network device interfaces, IPv4 and IPv6 protocol stacks, IP routing tables, firewall rules, the /proc/net and /sys/class/net directory trees, sockets, etc (since Linux 2.6.24)
Mount |	CLONE_NEWNS	| Mount points (since Linux 2.4.19)
PID	| CLONE_NEWPID |	Process IDs (since Linux 2.6.24)
User |	CLONE_NEWUSER	| User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)
UTS	| CLONE_NEWUTS	| Hostname and NIS domain name (since Linux 2.6.19)
Cgroup |	CLONE_NEWCGROUP	| Cgroup root directory (since Linux 4.6)

### `Linux CGroup`是Linux内核用来限制，控制与分离一个进程组群的资源（如CPU、内存、磁盘输入输出等）。
`Linux CGroupCgroup` 可​​​让​​​您​​​为​​​系​​​统​​​中​​​所​​​运​​​行​​​任​​​务​​​（进​​​程​​​）的​​​用​​​户​​​定​​​义​​​组​​​群​​​分​​​配​​​资​​​源​​​ — 比​​​如​​​ `CPU 时​​​间​​​`、​​​`系​​​统​​​内​​​存​​​`、​​​`网​​​络​​​带​​​宽​​​`或​​​者​​​这​​​些​​​资​​​源​​​的​​​组​​​合​​​。​​​您​​​可​​​以​​​监​​​控​​​您​​​配​​​置​​​的​​​ `cgroup`，拒​​​绝​​​` cgroup` 访​​​问​​​某​​​些​​​资​​​源​​​，甚​​​至​​​在​​​运​​​行​​​的​​​系​​​统​​​中​​​动​​​态​​​配​​​置​​​您​​​的​​​ `cgroup`。
主要提供了如下功能：

- `Resource limitation`: 限制进程使用的资源上限，比如最大内存、文件系统缓存使用限制。
- `Prioritization`: 优先级控制，比如：CPU利用和磁盘IO吞吐。
- `Accounting`: 一些审计或一些统计，主要目的是为了计费。
- `Control`: 挂起一组进程，或者重启一组进程。

前面说过，`cgroups` 是用来对进程进行资源管理的，因此 `cgroup` 需要考虑如何抽象这两种概念：进程和资源，同时如何组织自己的结构。`cgroups `中有几个非常重要的概念：

- `task`：任务，对应于系统中运行的一个实体，一般是指进程
- `subsystem`：子系统，具体的资源控制器（`resource class` 或者 `resource controller`），控制某个特定的资源使用。比如 CPU 子系统可以控制 CPU 时间，memory 子系统可以控制内存使用量
- `cgroup`：控制组，一组任务和子系统的关联关系，表示对这些任务进行怎样的资源管理策略
- `hierarchy`：层级树，一系列 `cgroup` 组成的树形结构。每个节点都是一个 `cgroup`，`cgroup` 可以有多个子节点，子节点默认会继承父节点的属性。系统中可以有多个 `hierarchy`

`cgroups`为每种可以控制的资源定义了一个子系统。典型的子系统介绍如下：

- `Block IO（blkio)`：限制块设备（磁盘、SSD、USB 等）的 IO 速率
- `CPU Set(cpuset)`：限制任务能运行在哪些 CPU 核上
- `CPU Accounting(cpuacct)`：生成 `cgroup` 中任务使用 CPU 的报告
- `CPU (CPU)`：限制调度器分配的 CPU 时间
- `Devices (devices)`：允许或者拒绝 cgroup 中任务对设备的访问
- `Freezer (freezer)`：挂起或者重启 cgroup 中的任务
- `Memory (memory)`：限制 cgroup 中任务使用内存的量，并生成任务当前内存的使用情况报告
- `Network Classifier(net_cls)`：为 cgroup 中的报文设置上特定的 classid 标志，这样 tc 等工具就能根据标记对网络进行配置
- `Network Priority (net_prio)`：对每个网络接口设置报文的优先级
- `perf_event`：识别任务的 cgroup 成员，可以用来做性能分析

使用 `cgroups` 的方式有几种：

- 使用 `cgroups` 提供的虚拟文件系统，直接通过创建、读写和删除目录、文件来控制 `cgroups`
- 使用命令行工具，比如 `libcgroup `包提供的 `cgcreate`、`cgexec`、`cgclassify` 命令
- 使用 `rules engine daemon` 提供的配置文件
- 当然，`systemd`、`lxc`、`docker` 这些封装了 `cgroups` 的软件也能让你通过它们定义的接口控制 `cgroups` 的内容

在实践中，系统管理员一般会利用`CGroup`做下面这些事（有点像为某个虚拟机分配资源似的）：

- 隔离一个进程集合（比如：`nginx`的所有进程），并限制他们所消费的资源，比如绑定CPU的核。
- 为这组进程 分配其足够使用的内存
- 为这组进程分配相应的网络带宽和磁盘存储限制
- 限制访问某些设备（通过设置设备的白名单）

### `UnionFS`就是把不同物理位置的目录合并mount到同一个目录中

`UnionFS `其实是一种为 Linux 操作系统设计的用于把多个文件系统『联合』到同一个挂载点的文件系统服务。

操作系统中，联合挂载`（union mounting）`是一种将多个目录结合成一个目录的方式，这个目录看起来就像包含了他们结合的内容一样。

联合文件系统的实现通常辅有`“写时复制（CoW）”`的实现技术，这样任何对于底层文件系统分层的更改都会被`“向上拷贝”`到文件系统的一个临时、工作、或高层的分层里面。这个可写的层然后可以被看做是一个`“改动（diff）”`，能将之应用到下层只读的层，而这些层很可能作为底层被很多容器的进程中共享。这是一个很重要的点。

而 `AUFS` 即 `Advanced UnionFS` 其实就是 `UnionFS` 的升级版，它能够提供更优秀的性能和效率。
`AUFS`有所有`Union FS`的特性，把多个目录，合并成同一个目录，并可以为每个需要合并的目录指定相应的权限，实时的添加、删除、修改已经被`mount`好的目录。而且，他还能在多个可写的`branch/dir`间进行负载均衡。
`AUFS` 作为联合文件系统，它能够将不同文件夹中的层联合`（Union）`到了同一个文件夹中，这些文件夹在 `AUFS` 中称作分支，整个『联合』的过程被称为联合挂载`（Union Mount）`
`AUFS` 只是 Docker 使用的存储驱动的一种，除了 `AUFS` 之外，Docker 还支持了不同的存储驱动，包括 `aufs`、`devicemapper`、`overlay2`、`zfs` 和` vfs `等等，在最新的 Docker 中，`overlay2` 取代了 `aufs` 成为了推荐的存储驱动，但是在没有 `overlay2` 驱动的机器上仍然会使用 `aufs `作为 Docker 的默认驱动。

官方参考文档：

- [https://docs.docker.com/storage/storagedriver/select-storage-driver/#supported-backing-filesystems](https://docs.docker.com/storage/storagedriver/select-storage-driver/#supported-backing-filesystems)


## 问：OCI、runC 、Containerd 、Docker-shim 是什么？他们之间有怎么样的关联？

**答：** 
Open Container Initiative（OCI）是：

> OCI在2015年6月宣布成立，旨在围绕容器格式和运行时制定一个开放的工业化标准。OCI的目标是为了避免容器的生态分裂为“小生态王国”，确保一个引擎上构建的容器可以运行在其他引擎之上。这是实现容器可移植性至关重要的部分。只要Docker是唯一的运行时，它就是事实上的行业标准。但是随着可用（和采纳）和其他引擎，有必要从技术的角度上定义“什么是容器”，以便不同实现上可以达成最终的一致。

runC是：

> runC是从Docker的libcontainer中迁移而来的，实现了容器启停、资源隔离等功能。runC是一个轻量级的工具，它是用来运行容器的，只用来做这一件事，并且这一件事要做好。如果你了解过Docker引擎早期的历史，你应该知道当时启动和管理一个容器需要使用LXC工具集，然后在使用libcontainer。libcontainer就是使用类似cgroup和namespace一样的Linux内核设备接口编写的一小段代码，它是容器的基本构建模块。为了是过程更加简单，runC基本上是一个小命令行工具且它可以不用通过Docker引擎，直接就可以使用容器。这是一个独立的二进制文件，使用OCI容器就可以运行它。

Containerd是：

> Containerd是一个简单的守护进程，它可以使用runC管理容器，使用gRPC暴露容器的其他功能。相比较Docker引擎，使用gRPC，containerd暴露出针对容器的增删改查的接口，然而Docker引擎只是使用full-blown HTTP API接口对Images、Volumes、network、builds等暴露出这些方法。containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化，确保原有接口向下兼容。向下通过containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。

Docker-shim 是：

> docker-shim是一个真实运行的容器的真实垫片载体，每启动一个容器都会起一个新的docker-shim的一个进程，
他直接通过指定的三个参数：容器id，boundle目录 。运行是二进制（默认为runc）来调用runc的api创建一个容器。

它们之间的关联关系：

我们`ls`看下主机上的Docker二进制文件，会发现有`docker`,`dockerd`,`docker-containerd`,`docker-containerd-shim`,和`docker-containerd-ctr`，`docker-runc`等。

```bash
$ ls /usr/bin |grep docker
docker
docker-containerd
docker-containerd-ctr
docker-containerd-shim
dockerd
docker-init
docker-proxy
docker-runc
```



- `dockerd`是`docker engine`守护进程，`dockerd`启动时会启动`docker-containerd`子进程。

- `dockerd`与`docker-containerd`通过`grpc`进行通信。

- `docker-containerd-ctr`是`docker-containerd`的`cli`。

- `docker-containerd`通过`docker-containerd-shim`操作`docker-runc`，`docker-runc`真正控制容器生命周期 。

- 启动一个容器就会启动一个`docker-containerd-shim`进程。

- `docker-containerd-shim`直接调用`docker-runc`的包函数。

- 真正用户想启动的进程由`docker-runc`的`init`进程启动，即`runc init [args ...]` 。

进程关系模型：

```bash

docker     ctr
  |         |
  V         V
dockerd -> containerd ---> shim -> runc -> runc init -> process
                      |-- > shim -> runc -> runc init -> process
                      +-- > shim -> runc -> runc init -> process
```

![images](https://linux7788.com/images/posts/docker-containerd-runc.png)

Docker引擎仍然管理者`images`，然后移交给`containerd`运行，`containerd`再使用`runC`运行容器。

`Containerd`只处理`containers`管理容器的开始，停止，暂停和销毁。由于容器运行时是孤立的引擎，引擎最终能够启动和升级而无需重新启动容器。

## 问：Docker pull 镜像如何加速？

**答：**我们可以使用 Docker 镜像加速器来解决这个问题，加速器就是镜像、代理的概念。国内有不少机构提供了免费的加速器以方便大家使用，这里列出一些常用的加速器服务：

- Docker 官方的中国镜像加速器：从2017年6月9日起，Docker 官方提供了在中国的加速器，以解决墙的问题。不用注册，直接使用加速器地址：`https://registry.docker-cn.com` 即可。
- 中国科技大学的镜像加速器：中科大的加速器不用注册，直接使用地址[https://docker.mirrors.ustc.edu.cn/](https://docker.mirrors.ustc.edu.cn/)配置加速器即可。进一步的信息可以访问：[http://mirrors.ustc.edu.cn/help/dockerhub.html?highlight=docker](http://mirrors.ustc.edu.cn/help/dockerhub.html?highlight=docker)
- 阿里云加速器：注册阿里云开发账户(免费的)后，访问这个链接就可以看到加速器地址： [https://cr.console.aliyun.com/#/accelerator](https://cr.console.aliyun.com/#/accelerator)
- DaoCloud 加速器：注册 DaoCloud 账户(支持微信登录)，然后访问： [https://www.daocloud.io/mirror#accelerator-doc](https://www.daocloud.io/mirror#accelerator-doc)
当然你也可以自己搞个梯子。

## 问：如果 Docker 升级或者重启的话，那容器是不是都会被停掉然后重启啊？
**答：**在 `1.12` 以前的版本确实如此，但是从 `1.12` 开始，Docker 引擎加入了 `--live-restore` 参数，使用该参数可以避免引擎升级、重启导致容器停止服务的情况。
默认情况该功能不会被启动，如需启动，需要配置 `docker` 服务配置文件。比如 `Centos 7.3` 这类 `systemd` 的系统，可以修改 `/usr/lib/systemd/system/docker.servic`e 文件，在 `ExecStart=` 后面配置上 `--live-restore`：

```bash
ExecStart=/usr/bin/dockerd \
    --registry-mirror=https://registry.docker-cn.com \
    --live-restore
```

上面的格式中使用了行尾 `\` 的换行形式，这点和 `bash` 脚本一样，`systemd` 支持这种换行形式，如对此不了解可以先去学习 `bash` 程序设计。
需要注意的是，`--live-restore` 和 `Swarm Mode` 不兼容，所以在集群环境中不要使用。实际上集群环境也不用担心某个服务器重启的问题，因为其上的服务都会被调度到别的节点上，因此服务并不会被中断。

官方参考文档：

- [https://docs.docker.com/config/containers/live-restore/#enable-live-restore](https://docs.docker.com/config/containers/live-restore/#enable-live-restore)


## 问：服务器上线后，怎么发现总有个 `xmrig` 的容器在跑，删了还出来，这是什么鬼？

**警告！！你的服务器已经被入侵了！！**

**答：**有些人服务器上线后，发现突然多了一些莫名奇妙的容器在跑。比如下面这个例子：

```bash
$ docker ps -a
IMAGE           COMMAND                  CREATED      STATUS                      PORTS    NAMES
linuxrun/cpu2   "./xmrig --algo=cr...."  4 hours ago  Exited (137) 7 minutes ago           linuxrun-cpu2
...
```

这就是有人在你的 Docker 宿主上跑了一个 `xmrig` 挖矿的蠕虫，因为你的系统被入侵了……。
在你大叫 Docker 不安全之前，先检讨一下自己是不是做错了。检查一下 `dockerd` 引擎是否配置错误：`ps -ef | grep dockerd`，如果你看到的是这样子的：

```bash
$ ps -ef | grep dockerd
123  root   12:34   /usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
```

如果在其中没有 `--tlsverify` 类的 TLS 配置参数，那就说明你将你的系统大门彻底敞开了。这是配置上**严重的安全事故**。

`-H tcp://0.0.0.0:2375` 是说你希望通过 `2375/tcp` 来操控你的 Docker 引擎，但是如果你没有加 `--tlsverify` 类的配置，就表明你的意图是允许任何人来操控你的 Docker 引擎，而 Docker 引擎是以 `root` 权限运行的，因此，你等于给了地球上所有人你服务器的 `root` 权限，而且还没密码。

如果细心一些，去查看 dockerd 的服务日志，journalctl -u docker，日志中有明确的警告，警告你这么配置是极端危险的：

```bash
$ journalctl -u docker
...
level=warning msg="[!] DON'T BIND ON ANY IP ADDRESS WITHOUT setting --tlsverify IF YOU DON'T KNOW WHAT YOU'RE DOING [!]"
...
```

如果这些你都忽略了，那么被别人入侵就太正常了，是你自己邀请别人来的。所以，Docker 服务绑定端口，必须通过 TLS 保护起来，以后见到 `-H tcp://.... `就要检查，是否同时配置了 `--tlsverify`，如果没看到，那就是严重错误了。
这也是为什么推荐使用 `docker-machine` 进行 Docker 宿主管理的原因，因为 `docker-machine` 会帮你创建证书、配置 TLS，确保服务器的安全。

进一步如何配置 TLS 的信息，可以查看官网文档：

- [https://docs.docker.com/engine/security/https/](https://docs.docker.com/engine/security/https/)

关于 `docker-machine` 的介绍，可以看官网文档：

- [https://docs.docker.com/machine/overview/](https://docs.docker.com/machine/overview/)


## 问：怎么固定容器 IP 地址？每次重启容器都要变化 IP 地址怎么办？

**答：**一般情况是不需要指定容器 IP 地址的。这不是虚拟主机，而是容器。其地址是供容器间通讯的，容器间则不用 IP 直接通讯，而使用容器名、服务名、网络别名。

为了保持向后兼容，`docker run` 在不指定 `--network` 时，所在的网络是 `default bridge`，在这个网络下，需要使用 `--link`参数才可以让两个容器找到对方。

这是有局限性的，因为这个时候使用的是 `/etc/hosts` 静态文件来进行的解析，比如一个主机挂了后，重新启动IP可能会改变。虽然说这种改变Docker是可能更新`/etc/hosts`文件，但是这有诸多问题，可能会因为竞争冒险导致 `/etc/hosts` 文件损毁，也可能还在运行的容器在取得 `/etc/hosts` 的解析结果后，不再去监视该文件是否变动。种种原因都可能会导致旧的主机无法通过容器名访问到新的主机。

参考官网文档：

- [https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/)

如果可能不要使用这种过时的方式，而是用下面说的自定义网络的方式。

而对于新的环境（`Docker 1.10`以上），应该给容器建立自定义网络，同一个自定义网络中，可以使用对方容器的容器名、服务名、网络别名来找到对方。这个时候帮助进行服务发现的是Docker 内置的DNS。所以，无论容器是否重启、更换IP，内置的DNS都能正确指定到对方的位置。
参考官网文档：

- [https://docs.docker.com/engine/userguide/networking/work-with-networks/#linking-containers-in-user-defined-networks](https://docs.docker.com/engine/userguide/networking/work-with-networks/#linking-containers-in-user-defined-networks)

## 问：如何修改容器的 /etc/hosts 文件？
**答：**容器内的 `/etc/hosts` 文件不应该被随意修改，如果必须添加主机名和 IP 地址映射关系，应该在 `docker run` 时使用 `--add-host` 参数，或者在 `docker-compose.yml` 中添加 `extra_hosts` 项。

不过在用之前，应该再考虑一下真的需要修改 `/etc/hosts` 么？如果只是为了容器间互相访问，应该建立自定义网络，并使用 Docker 内置的 DNS 服务。

## 问：怎么映射宿主端口？Dockerfile 中的EXPOSE和 docker run -p 有啥区别？

**答：**Docker中有两个概念，一个叫做 `EXPOSE` ，一个叫做 `PUBLISH` 。
- `EXPOSE` 是镜像/容器声明要暴露该端口，可以供其他容器使用。这种声明，在没有设定 `--icc=false`的时候，实际上只是一种标注，并不强制。也就是说，没有声明 `EXPOSE` 的端口，其它容器也可以访问。但是当强制 `--icc=false` 的时候，那么只有 `EXPOSE` 的端口，其它容器才可以访问。
- `PUBLISH` 则是通过映射宿主端口，将容器的端口公开于外界，也就是说宿主之外的机器，可以通过访问宿主IP及对应的该映射端口，访问到容器对应端口，从而使用容器服务。

`EXPOSE` 的端口可以不 `PUBLISH`，这样只有容器间可以访问，宿主之外无法访问。而 `PUBLISH` 的端口，可以不事先 `EXPOSE`，换句话说 `PUBLISH` 等于同时隐式定义了该端口要 `EXPOSE`。

`docker run` 命令中的 `-p`, `-P` 参数，以及 `docker-compose.yml` 中的  `ports` 部分，实际上均是指 `PUBLISH`。

小写 `-p` 是端口映射，格式为 `[宿主IP:]<宿主端口>:<容器端口>`，其中宿主端口和容器端口，既可以是一个数字，也可以是一个范围，比如：`1000-2000:1000-2000`。对于多宿主的机器，可以指定宿主`IP`，不指定宿主`IP`时，守护所有接口。

大写 `-P `则是自动映射，将所有定义 `EXPOSE` 的端口，随机映射到宿主的某个端口。

## 问：我要映射好几百个端口，难道要一个个   `-p` 么？

**答：** `-p` 是可以用范围的：

```bash
-p 8001-8010:8001-8010
```

## 问：为什么 `-p` 后还是无法通过映射端口访问容器里面的服务？

**答：** 首先，当然是检查这个 `docker` 的容器是否启动正常： `docker ps`、`docker top <容器ID>`、`docker logs <容器ID>`、`docker exec -it <容器ID> bash`等，这是比较常用的排障的命令；如果是 `docker-compose` 也有其对应的这一组命令，所以排障很容易。

如果确保服务一切正常，甚至在容器里，可以访问到这些服务，`docker ps` 也显示出了端口映射成功，那么就需要检查防火墙了。



## 问：如何让一个容器连接两个网络？

**答：**如果是使用 `docker run`，那很不幸，一次只可以连接一个网络，因为 `docker run` 的 `--network` 参数只可以出现一次（如果出现多次，最后的会覆盖之前的）。不过容器运行后，可以用命令 `docker network connect` 连接多个网络。
假设我们创建了两个网络：

```bash
$ docker network create mynet1
$ docker network create mynet2
```

然后，我们运行容器，并连接这两个网络。

```bash
$ docker run -d --name web --network mynet1 nginx
$ docker network connect mynet2 web
```

但是如果使用 `docker-compose` 那就没这个问题了。因为实际上，`Docker Remote API` 是支持一次性指定多个网络的，但是估计是命令行上不方便，所以 `docker run` 限定为只可以一次连一个。`docker-compose` 直接就可以将服务的容器连入多个网络，没有问题。

```bash
version: '2'
services:
    web:
        image: nginx
        networks:
            - mynet1
            - mynet2
networks:
    mynet1:
    mynet2:
```

## 问：Docker 多宿主网络怎么配置？

**答**：Docker 跨节点容器网络互联，最通用的是使用 `overlay` 网络。
一代 `Swarm` 已经不再使用，它要求使用 `overlay `网络前先准备好分布式键值库，比如 `etcd`, `consul` 或 `zookeeper`。然后在每个节点的 Docker 引擎中，配置 `--cluster-store` 和 `--cluster-advertise` 参数。这样才可以互连。
现在都在使用二代 `Swarm`，也就是 `Docker Swarm Mode`，非常简单，只要 `docker swarm init` 建立集群，其它节点 `docker swarm join`加入集群后，集群内的服务就自动建立了 `overlay` 网络互联能力。

需要注意的是，如果是多网卡环境，无论是 `docker swarm init` 还是 `docker swarm join`，都不要忘记使用参数 `--advertise-addr` 指定宣告地址，否则自动选择的地址很可能不是你期望的，从而导致集群互联失败。格式为 `--advertise-addr <地址>:<端口>`，地址可以是 `IP `地址，也可以是网卡接口，比如 `eth0`。端口默认为 `2377`，如果不改动可以忽略。

此外，这是供服务使用的 `overlay`，因此所有 `docker service create` 的服务容器可以使用该网络，而 `docker run` 不可以使用该网络，除非明确该网络为 `--attachable`。

关于` overlay` 网络的进一步信息，可以参考官网文档：

- [https://docs.docker.com/engine/userguide/networking/get-started-overlay/](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)

虽然默认使用的是 `overlay` 网络，但这并不是唯一的多宿主互联方案。Docker 内置了一些其它的互联方案，比如效率比较高的 `macvlan`。如果在局域网络环境下，对 `overlay` 的额外开销不满意，那么可以考虑 `macvlan` 以及 `ipvlan`，这是比较好的方案。

- [https://docs.docker.com/engine/userguide/networking/get-started-macvlan/](https://docs.docker.com/engine/userguide/networking/get-started-macvlan/)

此外，还有很多第三方的网络可以用来进行跨宿主互联，可以访问官网对应文档进一步查看：

- [https://docs.docker.com/engine/extend/legacy_plugins/#/network-plugins](https://docs.docker.com/engine/extend/legacy_plugins/#/network-plugins)

## 问：明明 docker network ls 中看到了建立的 overlay 网络，怎么 docker run 还说网络不存在啊？

**答：** 如果在 `docker network ls` 中看到了如下的 `overlay` 网络：

```bash
NETWORK ID          NAME                DRIVER              SCOPE
...
24pz359114y0        mynet               overlay             swarm
...
```

那么这个名为 `mynet` 的网络是不可以连接到 `docker run` 的容器。如果试图连接则会出现报错。
如果是 1.12 的系统，会看到这样报错信息：

```bash
$ docker run --rm --network mynet busybox
docker: Error response from daemon: network mynet not found.
See 'docker run --help'.
```

报错说 `mynet` 网络找不到。其实如果仔细观察，会看到这个名为 `mynet` 的网络，驱动是 `overlay` 没有错，但它的 `Scope` 是 `swarm`。这个意思是说这个网络是在二代 `Swarm` 环境中建立的 `overlay` 网络，因此只可以由 `Swarm` 环境下的服务容器才可以使用。而 `docker run `所运行的只是零散的容器，并非 `Service`，因此自然在零散容器所能使用的网络中，不存在叫 `mynet` 网络。

`docker run` 可以使用的 `overlay` 网络是 `Scope` 为 `global` 的 `overlay` 网络，也就是使用外置键值库所建立的 `overlay` 网络，比如一代 `Swarm` 的 `overlay` 网络。

这点在 1.13 后稍有变化。如果是 1.13 以后的系统，会看到这样的信息：

```bash
$ docker run --rm --network mynet busybox
docker: Error response from daemon: Could not attach to network mynet: rpc error: code = 7
 desc = network mynet not manually attachable.
```

报错信息不再说网络找不到，而是说这个 `mynet` 网络无法连接。这是由于从 1.13 开始，允许在建立网络的时候声明这个网络是否可以被零散的容器所连接。如果 `docker network create` 加了 `--attachable` 的参数，那么在后期，这个网络是可以被普通容器所连接的。

但是这是在安全模型上开了一个口子，因此，默认不允许普通容器链接，并且不建议使用。

## 问：使用 Swarm Mode 的时，看到有个叫 ingress 的 overlay 网络，它和自己创建的网络有什么区别？

**答：**  在启用了二代 Swarm 后，可能会在网络列表时看到一个名为 ingress 的 overlay 网络。

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
6beb824623a4        bridge              bridge              local
f3f636574c7a        docker_gwbridge     bridge              local
cfeb2513a4a3        host                host                local
88smbt683r5p        ingress             overlay             swarm
24pz359114y0        mynet               overlay             swarm
d35d69ece740        none                null                local
```

这里可以看到两个 `overlay` 网络，其中一个是我们创建的 `mynet`，另一个则是 Docker 引擎自己创建的 `ingress`，从驱动和 `Scope` 可以看出两个网络都是给 Swarm Mode 使用的 `overlay` 网络。

`ingress` 是 `overlay` 网络，但并不是普通的 `overlay network`，它是为边界进入流量特殊准备的网络。这个网络存在于集群中每一个Docker宿主上，不需要额外建立。

当我们使用 `docker service create -p 80:80` 这种形式创建一个服务的时候，我们要求映射集群端口 `80` 到服务容器的 `80` 端口上。其效果是访问任一节点的 `80` 端口，即使这个节点没有运行我们所需的容器，依旧可以连接到容器服务，并且取得结果。实现这样效果的一个原因就是因为 `ingress` 网络的存在。

Swarm 中的每个节点，都会有一个隐藏的沙箱容器监听宿主的服务端口，用于接收来自集群外界的访问。

我们可以通过 `docker network inspect ingress` 来看到这个沙箱容器：

```bash
$ docker network inspect ingress
[
   {
       "Name": "ingress",
       "Id": "88smbt683r5p7c0l7sd0dpniw",
       "Scope": "swarm",
       "Driver": "overlay",
       "EnableIPv6": false,
       "IPAM": {
           "Driver": "default",
           "Options": null,
           "Config": [
               {
                   "Subnet": "10.255.0.0/16",
                   "Gateway": "10.255.0.1"
               }
           ]
       },
       "Internal": false,
       "Containers": {
           "faff08692b5f916fcb15aa7ac6bc8633a0fa714a52a1fb75e57525c94581c45a": {
               "Name": "web.1.1jyunyva6picwsztzrj6t2cio",
               "EndpointID": "58240770eb25565b472384731b1b90e36141a633ce184a5163829cf96e9d1195",
               "MacAddress": "02:42:0a:ff:00:05",
               "IPv4Address": "10.255.0.5/16",
               "IPv6Address": ""
           },
           "ingress-sbox": {
               "Name": "ingress-endpoint",
               "EndpointID": "fe8f89d4f99d7bacb14c5cb723682c180278d62e9edd10b523cdd81a45695c5d",
               "MacAddress": "02:42:0a:ff:00:03",
               "IPv4Address": "10.255.0.3/16",
               "IPv6Address": ""
           }
       },
       "Options": {
           "com.docker.network.driver.overlay.vxlanid_list": "256"
       },
       "Labels": {}
   }
]
```


在上面的命令返回信息中，我们可以看到一个名为 `ingress-endpoint` 的容器，这就是边界沙箱容器。

当我们创建服务时，使用了 `-p` 参数后，服务容器就会被自动的加入到 `ingress `网络中，同时会在沙箱中注册映射信息，告知哪个服务要求守护哪个端口，具体对应容器是哪些。

因此当沙箱收到外部连接后，通过访问端口就可以知道具体服务在守护，然后会通过这个 `ingress` 网络去将连接请求转发给对应服务容器。而由于 `ingress` 的本质是 `overlay network`，因此，无论服务容器运行于哪个节点上，沙箱都可以成功的将连接转发给正确的服务容器。

所以，`ingress` 是特殊用途的网络，只要服务有` -p `选项，那么服务容器就会自动被加入该网络。因此把 `ingress` 网络当做普通的 `overlay `网络使用的话，除了会干扰 `Swarm` 正常的边界负载均衡的能力，也会破坏服务隔离的安全机制。所以不要把这个网络当做普通的 `overlay `网络来使用，需要控制服务互联和隔离时，请用自行创建的 `overlay` 网络。

## 问：听说 --link 过时不再用了？那容器互联、服务发现怎么办？

**答：** 在 1-2 年前，Docker 所有容器都连接于默认的桥接网络上，也就是很多老文章鼓捣的 `docker0 `桥接网卡。因此实际上默认情况下所有容器都是可以互联的，没有隔离，当然这样安全性不好。而服务发现，是在这种环境下发展出来的，通过修改容器内的 `/etc/hosts `文件来完成的。凡是 `--link` 的主机的别名就会出现于 `/etc/hosts` 中，其地址由 Docker 引擎维护。因此容器间才可以通过别名互访。

但是这种办法并不是好的解决方案，Docker 早在一年多以前就已经使用自定义网络了。在同一个网络中的容器，可以互联，并且，Docker 内置了 DNS，容器内的应用可以使用服务名、容器名、别名来进行服务发现，名称会经由内置的 DNS 进行解析，其结果是动态的；而不在同一网络中的容器，不可以互联。

因此，现在早就不用 `--link` 了，而且非常不建议使用。

首先是因为使用 `--link` 就很可能还在用默认桥接网络，这很不安全，所有容器都没有适度隔离，用自定义网络才比较方便互联隔离。

其次，修改 `/etc/hosts` 文件有很多弊病。比如，高频繁的容器启停环境时，容易产生竞争冒险，导致 `/etc/hosts` 文件损坏，出现访问故障；或者有些应用发现是来自于 `/etc/hosts` 文件后，就假定其为静态文件，而缓存结果不再查询，从而导致容器启停 `IP` 变更后，使用旧的条目而无法连接到正确的容器等等。

另外，在一代 `Swarm` 环境中，在 `docker-compose.yml` 中使用了 `links `就意味着服务间的强依赖关系，因此调度时不会将服务运行于不同节点，而是全部运行于一个节点，使得横向扩展失败。

所以不要再使用 `--link` 以及 `docker-compose.yml` 中的 `links` 了。应该使用 `docker network`，建立网络，而 `docker run --network` 来连接特定网络。或者使用 `version: '2'` 的 `docker-compose.yml` 直接定义自定义网络并使用。

## 问：CNM和CNI分别是啥东东？

**答：** 目前关于Linux容器网络接口的配置有两种的可能的标准：`容器网络模型（CNM）`和`容器网络接口（CNI）`。网络是相当复杂的，而且提供某种功能的方式也多种多样。

`CNM`是一个被 Docker 公司提出的规范。现在已经被`Cisco Contiv`,` Kuryr`, `Open Virtual Networking (OVN)`, `Project Calico`, `VMware` 和 `Weave `这些公司和项目所采纳。

`CNI`是由`CoreOS`提出的一个容器网络规范。已采纳改规范的包括`Apache Mesos`, `Cloud Foundry`, `Kubernetes`, `Kurma` 和 `rkt`。另外 `Contiv Networking`, `Project Calico` 和 `Weav`e这些项目也为CNI提供插件。

这两种方案都使用了驱动模型或者插件模型来为容器创建网络栈。这样的设计使得用户可以自由选择。两者都支持多个网络驱动被同时使用，也允许容器加入一个或多个网络。两者也都允许容器`runtime`在它自己的命名空间中启动网络。

`CNM` 模式下的网络驱动不能访问容器的网络命名空间。这样做的好处是`libnetwork`可以为冲突解决提供仲裁。一个例子是：两个独立的网络驱动提供同样的静态路由配置，但是却指向不同的下一跳IP地址。与此不同，`CNI`允许驱动访问容器的网络命名空间。`CNI`正在研究在类似情况下如何提供仲裁。

`CNI`支持与第三方`IPAM`的集成，可以用于任何容器`runtime`。`CNM`从设计上就仅仅支持Docker。由于`CNI`简单的设计，许多人认为编写`CNI`插件会比编写`CNM`插件来得简单。

**`CNI`官方网络插件**

地址：

- [https://github.com/containernetworking/cni](https://github.com/containernetworking/cni)

所有的标准和协议都要有具体的实现，才能够被大家使用。`CNI` 也不例外，目前官方在 `github` 上维护了同名的 `CNI`代码库，里面已经有很多可以直接拿来使用的 `CNI `插件。

官方提供的插件目前分成三类：`main`、`meta` 和 `ipam`。
`main` 是主要的实现了某种特定网络功能的插件；
`meta` 本身并不会提供具体的网络功能，它会调用其他插件，或者单纯是为了测试；
`ipam` 是分配 IP 地址的插件。`ipam` 并不提供某种网络功能，只是为了灵活性把它单独抽象出来，这样不同的网络插件可以根据需求选择 `ipam`，或者实现自己的 `ipam`。

这些插件的功能详细说明如下：

- main
-- loopback：这个插件很简单，负责生成 lo 网卡，并配置上 127.0.0.1/8 地址
-- bridge：和 docker 默认的网络模型很像，把所有的容器连接到虚拟交换机上
-- macvlan：使用 macvlan 技术，从某个物理网卡虚拟出多个虚拟网卡，它们有独立的 ip 和 mac 地址
-- ipvlan：和 macvlan 类似，区别是虚拟网卡有着相同的 mac 地址
-- ptp：通过 veth pair 在容器和主机之间建立通道
- meta
-- flannel：结合 bridge 插件使用，根据 flannel 分配的网段信息，调用 bridge 插件，保证多主机情况下容器
- ipam
-- host-local：基于本地文件的 ip 分配和管理，把分配的 IP 地址保存在文件中
-- dhcp：从已经运行的 DHCP 服务器中获取 ip 地址



## 问：容器怎么取宿主机 IP 啊？

**答：**

**单机环境**

如果是单机环境，很简单，不必琢磨怎么突破命名空间限制，直接用环境变量送进去即可。

`docker run -d -e HOST_IP=<宿主的IP地址> nginx `

然后容器内直接读取` HOST_IP`环境变量即可。


**集群环境**

集群环境相对比较复杂，`docker service create` 中的 `-e` 以及 `--env-file`是在服务创建时指定、读取环境变量内容，而不是运行时，因此对于每个节点都是一样的。而且目前不存在 `dockerd -e` 选项，所以直接使用这些选项达不到我们想要的效果。

不过有变通的办法，可以在宿主上建立一个 `/etc/variables` 文件（名字随意，这里用这个文件举例）。其内容为：

```bash
HOST_IP=1.2.3.4
```

其中 `1.2.3.4` 是这个节点的宿主 IP，因此每个节点的 `/etc/variables` 的内容不同。

而在启动服务时，指定挂载这个服务端本地文件：

```bash
docker service create --name app \
    --mount type=bind,source=/etc/variables,target=/etc/variables:ro \
    myapp
```

由于 `--mount` 是发生于容器运行时，因此所加载的是所运行的服务器的 `/etc/variables`，里面所包含的也是该服务器的 `IP` 地址。

在 `myapp` 这个镜像的入口脚本加入加载该环境变量文件的命令：


```bash
source /etc/variables
```


这样 `app` 这个服务容器就会拥有 `HOST_IP` 环境变量，其值为所运行的宿主 `IP`。

## 问：容器磁盘可以限制配额么？

**答：** 对于 `devicemapper`, `btrfs`, `zfs` 来说，可以通过 `--storage-opt size=100G `这种形式限制 `rootfs` 的大小。

`docker create -it --storage-opt size=120G fedora /bin/bash`

参考官网文档：

- [https://docs.docker.com/engine/reference/commandline/run/#/set-storage-driver-options-per-container](https://docs.docker.com/engine/reference/commandline/run/#/set-storage-driver-options-per-container)

## 问：我在容器里面看到的内存使用量是真实的该容器内存使用情况？

**答：** 不是的，众所周知，Docker相比于虚拟机，在隔离性上略显不足，这个Docker隔离性不足导致资源显示问题。进入容器看到是完整的物理机资源。
虽然 Docker原生的资源查询接口可以正确地识别分配的资源，但是用户常用的 `top`、`free`等命令却未能正确地识别我们施加于 Docker的资源限制，那么原因究竟是怎样。
事实上，类似 `top`、`free`等命令，其实多半都是从一些系统文件中获取资源信息,`/proc/cpuinfo`,`/proc/meminfo`而 Docker的隔离性不足的问题里，就包括跟宿主机共享 `sys`、`proc`等系统文件，因此如果在容器中使用依赖这些文件的命令，如 `uptime`等，实际上都显示的是宿主机信息。

容器的显示问题，在很早期的版本中就有人提出过。而 Docker官方可能是出于某些原因的考虑，并没有试图修复这些显示问题。目前来说，解决显示问题还没办法很好地在 Docker中进行集成，仍然需要在 Docker之外做一些修改。

目前社区中常见的做法是利用 `lxcfs`来提供容器中的资源可见性。`lxcfs` 是一个开源的`FUSE（用户态文件系统）`实现来支持`LXC`容器，它也可以支持Docker容器。

`LXCFS`通过用户态文件系统，在容器中提供下列` procfs` 的文件。

```bash
/proc/cpuinfo
/proc/diskstats
/proc/meminfo
/proc/stat
/proc/swaps
/proc/uptime
```

容器中进程读取相应文件内容时，`LXCFS`的`FUSE`实现会从容器对应的`Cgroup`中读取正确的内存限制。从而使得应用获得正确的资源约束设定。

使用方法：

安装 `lxcfs` 的`RPM`包

```bash
$ wget https://copr-be.cloud.fedoraproject.org/results/ganto/lxd/epel-7-x86_64/00486278-lxcfs/lxcfs-2.0.5-3.el7.centos.x86_64.rpm
$ yum install lxcfs-2.0.5-3.el7.centos.x86_64.rpm
```

启动 `lxcfs`

```bash
$ mkdir -p /var/lib/lxcfs
$ lxcfs /var/lib/lxcfs &
```


运行Docker容器测试

```bash
$ docker run -it -m 300m \
      -v /var/lib/lxcfs/proc/cpuinfo:/proc/cpuinfo:rw \
      -v /var/lib/lxcfs/proc/diskstats:/proc/diskstats:rw \
      -v /var/lib/lxcfs/proc/meminfo:/proc/meminfo:rw \
      -v /var/lib/lxcfs/proc/stat:/proc/stat:rw \
      -v /var/lib/lxcfs/proc/swaps:/proc/swaps:rw \
      -v /var/lib/lxcfs/proc/uptime:/proc/uptime:rw \
      centos:7 /bin/bash

[root@e851562db40d /]# free -m
              total        used        free      shared  buff/cache   available
Mem:            300           3         295          29           0         295
Swap:           300           0         300
[root@e851562db40d /]# cat /proc/meminfo
MemTotal:         307200 kB
MemFree:          302768 kB
MemAvailable:     302768 kB
Buffers:               0 kB
.......................
```

我们可以看到`MemTotal`的内存为`300MB`，配置已经生效。

官方项目地址：

- [https://github.com/lxc/lxcfs](https://github.com/lxc/lxcfs)


## 问：数据容器、数据卷、命名卷、匿名卷、挂载目录这些都有什么区别？

**答：** 首先，挂载分为`挂载本地宿主目录` 和 `挂载数据卷(Volume)`。而数据卷又分为`匿名数据卷`和`命名数据卷`。

绑定宿主目录的概念很容易理解，就是将宿主目录绑定到容器中的某个目录位置。这样容器可以直接访问宿主目录的文件。其形式是

```bash
docker run -d -v /var/www:/app nginx
```

这里注意到 `-v` 的参数中，前半部分是绝对路径。在 `docker run` 中必须是绝对路径，而在 `docker-compose` 中，可以是相对路径，因为 `docker-compose `会帮你补全路径。

另一种形式是使用 `Docker Volume`，也就是数据卷。这是很多看古董书的人不了解的概念，不要跟数据容器（Data Container）弄混。数据卷是 Docker 引擎维护的存储方式，使用 `docker volume create` 命令创建，可以利用卷驱动支持多种存储方案。其默认的驱动为 `local`，也就是本地卷驱动。本地驱动支持命名卷和匿名卷。

顾名思义，命名卷就是有名字的卷，使用 `docker volume create --name xxx` 形式创建并命名的卷；而匿名卷就是没名字的卷，一般是 `docker run -v /data `这种不指定卷名的时候所产生，或者 Dockerfile 里面的定义直接使用的。

有名字的卷，在用过一次后，以后挂载容器的时候还可以使用，因为有名字可以指定。所以一般需要保存的数据使用命名卷保存。

而匿名卷则是随着容器建立而建立，随着容器消亡而淹没于卷列表中（对于 docker run 匿名卷不会被自动删除）。对于二代 Swarm 服务而言，匿名卷会随着服务删除而自动删除。 因此匿名卷只存放无关紧要的临时数据，随着容器消亡，这些数据将失去存在的意义。

此外，还有一个叫数据容器 (Data Container) 的概念，也就是使用` --volumes-from `的东西。这早就不用了，如果看了书还在说这种方式，那说明书已经过时了。按照今天的理解，这类数据容器，无非就是挂了个匿名卷的容器罢了。

在 `Dockerfile` 中定义的挂载，是指 `匿名数据卷`。`Dockerfile` 中指定 `VOLUME` 的目的，只是为了将某个路径确定为卷。

我们知道，按照最佳实践的要求，不应该在容器存储层内进行数据写入操作，所有写入应该使用卷。如果定制镜像的时候，就可以确定某些目录会发生频繁大量的读写操作，那么为了避免在运行时由于用户疏忽而忘记指定卷，导致容器发生存储层写入的问题，就可以在 `Dockerfile` 中使用` VOLUME `来指定某些目录为匿名卷。这样即使用户忘记了指定卷，也不会产生不良的后果。

这个设置可以在运行时覆盖。通过 `docker run` 的 `-v`参数或者 `docker-compose.yml` 的 `volumes` 指定。使用命名卷的好处是可以复用，其它容器可以通过这个命名数据卷的名字来指定挂载，共享其内容（不过要注意并发访问的竞争问题）。

比如，`Dockerfile` 中说 `VOLUME /data`，那么如果直接 `docker run`，其 `/data` 就会被挂载为匿名卷，向 `/data `写入的操作不会写入到容器存储层，而是写入到了匿名卷中。但是如果运行时 `docker run -v mydata:/data`，这就覆盖了 `/data` 的挂载设置，要求将 `/data` 挂载到名为 `mydata` 的命名卷中。所以说 `Dockerfile` 中的 `VOLUME` 实际上是一层保险，确保镜像运行可以更好的遵循最佳实践，不向容器存储层内进行写入操作。

数据卷默认可能会保存于` /var/lib/docker/volumes`，不过一般不需要、也不应该访问这个位置。


## 问：卷和挂载目录有什么区别？

**答：** `卷 (Docker Volume)` 是受控存储，是由 `Docker` 引擎进行管理维护的。因此使用卷，你可以不必处理 `uid`、`SELinux` 等各种权限问题，Docker 引擎在建立卷时会自动添加安全规则，以及根据挂载点调整权限。并且可以统一列表、添加、删除。另外，除了本地卷外，还支持网络卷、分布式卷。

而挂载目录那就没人管了，属于用户自行维护。你就必须手动处理所有权限问题。特别是在 `CentOS` 上，很多人碰到 `Permission Denied`，就是因为没有使用卷，而是挂载目录，而且还对 `SELinux` 安全权限一无所知导致。

## 问：为什么绑定了宿主的文件到容器，宿主修改了文件，容器内看到的还是旧的内容啊？

**答：** 在绑定宿主内容的形式中，有一种特殊的形式，就是绑定宿主文件，既：

```bash
docker run -d -v $PWD/myapp.ini:/app/app.ini myapp

```

在 `myapp.ini` 文件不发生改变的情况下，这样的绑定是和绑定宿主目录性质一样，同样是将宿主文件绑定到容器内部，容器内可以看到这个文件。但是，一旦文件发生改变，情况则有不同。

简单的文件修改，比如 `echo "name = jessie" >> myapp.ini`，这类修改依旧还是原来的文件，宿主（或容器）对文件进行的改动，另一方是可以看到的。

而复杂的文件操作，比如使用 `vim`，或者其它编辑器编辑文件，则很有可能会导致一方的修改，另一方看不到。

其原因是这类编辑器在保存文件的时候，经常会采用一种避免写入过程中发生故障而导致文件丢失的策略，既先把内容写到一个新的文件中去，写好了后，再删除旧的文件，然后把新文件改名为旧的文件名，从而完成保存的操作。从这个操作流程可以看出，虽然修改后的文件的名字和过去一样，但对于文件系统而言是一个新的文件了。换句话说，虽然是同名文件，但是旧的文件的 `inode` 和修改后的文件的 `inode` 不同。

```bash
$ ls -i
268541 hello.txt
$ vi hello.txt
$ ls -i
268716 hello.txt
```

如上面的例子可以看到，经过 `vim` 编辑文件后，`inode` 从 `268541` 变为了 `268716`，这就是刚才说的，名字还是那个名字，文件已不是原来的文件了。

而 Docker 的 绑定宿主文件，实际上在文件系统眼里，针对的是 `inode`，而不是文件名。因此容器内所看到的，依旧是之前旧的 `inode` 对应的那个文件，也就是旧的内容。

这就出现了之前的那个问题，在宿主内修改绑定文件的内容，结果发现容器内看不到改变，其原因就在于宿主的那个文件已不是原来的文件了。

这类问题解决办法很简单，如果文件可能改变，那么就不要绑定宿主文件，而是绑定一个宿主目录，这样只要目录不跑，里面文件爱咋改就咋改。

## 问：多个 Docker 容器之间共享数据怎么办？

**答：** 如果是不同宿主，则可以使用分布式数据卷驱动，让分布在不同宿主的容器都可以访问到的分布式存储的位置。如S3之类：


- [https://docs.docker.com/engine/extend/legacy_plugins/#authorization-plugins](https://docs.docker.com/engine/extend/legacy_plugins/#authorization-plugins)

## 问：既然一个容器一个应用，那么我想在该容器中用计划任务 cron 怎么办？

**答：** `cron` 其实是另一个服务了，所以应该另起一个容器来进行，如需访问该应用的数据文件，那么可以共享该应用的数据卷即可。而 `cron` 的容器中，`cron` 以前台运行即可。

比如，我们希望有个 `python` 脚本可以定时执行。那么可以这样构建这个容器。

首先基于 `python` 的镜像定制：

```bash
FROM python:3.5.2
ENV TZ=Asia/Shanghai
RUN apt-get update \
    && apt-get install -y cron \
    && apt-get autoremove -y
COPY ./cronpy /etc/cron.d/cronpy
CMD ["cron", "-f"]
```

中所提及的 `cronpy` 就是我们需要计划执行的 `cron` 脚本。

```bash
* * * * * root /app/task.py >> /var/log/task.log 2>&1
```

在这个计划中，我们希望定时执行 `/app/task.py` 文件，日志记录在` /var/log/task.log` 中。这个 `task.py` 是一个非常简单的文件，其内容只是输出个时间而已。

```python
#!/usr/local/bin/python
from datetime import datetime
print ("Cron job has run at {0} with environment variable ".format(str(datetime.now())))
```

这 `task.py` 可以在构建镜像时放进去，也可以挂载宿主目录。在这里，我以挂载宿主目录举例。

```bash
# 构建镜像
docker build -t cronjob:latest .
# 运行镜像
docker run \
    --name cronjob \
    -d \
    -v $(pwd)/task.py:/app/task.py \
    -v $(pwd)/log/:/var/log/ \
    cronjob:latest

```

需要注意的是，应该在构建主机上赋予 `task.py` 文件可执行权限。


## 问：为什么说数据库不适合放在 Docker 容器里运行？

**答：** 不为什么，因为这个说法不对，大部分认为数据库必须放到容器外运行的人根本不知道 `Docker Volume` 为何物。

在早年 Docker 没有 `Docker Volume` 的时候，其数据持久化是一个问题，但是这已经很多年过去了。现在有 `Docker Volume `解决持久化问题，从本地目录绑定、受控存储空间、块设备、网络存储到分布式存储，`Docker Volume` 都支持，不存在数据读写类的服务不适于运行于容器内的说法。

Docker 不是虚拟机，使用数据卷是直接向宿主写入文件，不存在性能损耗。而且卷的生存周期独立于容器，容器消亡卷不消亡，重新运行容器可以挂载指定命名卷，数据依然存在，也不存在无法持久化的问题。

建议去阅读一下官方文档：


- [https://docs.docker.com/engine/tutorials/dockervolumes/](https://docs.docker.com/engine/tutorials/dockervolumes/)
- [https://docs.docker.com/engine/reference/commandline/volume_create/](https://docs.docker.com/engine/reference/commandline/volume_create/)
- [https://docs.docker.com/engine/extend/legacy_plugins/#/volume-plugins](https://docs.docker.com/engine/extend/legacy_plugins/#/volume-plugins)

## 问：如何列出容器和所使用的卷的关系？
**答：** 要感谢强大的 `Go Template`，可以使用下面的命令来显示：

```bash
docker inspect --format '{{.Name}} => {{with .Mounts}}{{range .}}
    {{.Name}},{{end}}{{end}}' $(docker ps -aq)
```

**注意这里的换行和空格是有意如此的**，这样就可以再返回结果控制缩进格式。其结果将是如下形式：

```bash
$ docker inspect --format '{{.Name}} => {{with .Mounts}}{{range .}}
    {{.Name}}{{end}}{{end}}' $(docker ps -aq)
/device_api_1 =>
/device_dashboard-debug_1 =>
/device_redis_1 =>
    device_redis-data
/device_mongo_1 =>
    device_mongo-data
    61453e46c3409f42e938324d7feffc6aeb6b7ce16d2080566e3b128c910c9570
/prometheus_prometheus_1 =>
    fc0185ed3fc637295de810efaff7333e8ff2f6050d7f9368a22e19fb2c1e3c3f

```

## 问：docker pull 下来的镜像文件都在哪？


**答：**  Docker不是虚拟机，Docker 镜像也不是虚拟机的 ISO 文件。Docker 的镜像是分层存储，每一个镜像都是由很多层，很多个文件组成。而不同的镜像是共享相同的层的，所以这是一个树形结构，不存在具体哪个文件是 `pull` 下来的镜像的问题。

## 问：docker images 命令显示的镜像占了好大的空间，怎么办？

**答：** 这个显示的大小是计算后的大小，要知道 `docker image` 是分层存储的，在`1.10`之前，不同镜像无法共享同一层，所以基本上确实是下载大小。但是从`1.10`之后，已有的层（通过SHA256来判断），不需要再下载。只需要下载变化的层。所以实际下载大小比这个数值要小。而且本地硬盘空间占用，也比d`ocker images`列出来的东西加起来小很多，很多重复的部分共享了。
用以下命令可以清理旧的和未使用的Docker镜像：

```bash
$ docker image prune  [OPTIONS] #命令用于删除未使用的映像。 如果指定了-a，还将删除任何容器未引用的所有映像。

名称，简写	默认	说明
--all, -a	false	显示所有映像(默认隐藏中间映像)
--force, -f	false	不要提示确认
```

## 问：docker images -a 后显示了好多 <none> 的镜像？都是什么呀？能删么？

**答：**  简单来说，`<none>` 就是说该镜像没有打标签。而没有打标签镜像一般分为两类，一类是依赖镜像，一类是丢了标签的镜像。

**依赖镜像**

Docker的镜像、容器的存储层是`Union FS`，分层存储结构。所以任何镜像除了最上面一层打上标签(tag)外，其它下面依赖的一层层存储也是存在的。这些镜像没有打上任何标签，所以在 `docker images -a` 的时候会以 `<none>` 的形式显示。注意观察一下 `docker pull `的每一层的`sha256`的校验值，然后对比一下` <none>` 中的相同校验值的镜像，它们就是依赖镜像。这些镜像不应当被删除，因为有标签镜像在依赖它们。

**丢了标签的镜像**

这类镜像可能本来有标签，后来丢了。原因可能很多，比如：

- `docker pull` 了一个同样标签但是新版本的镜像，于是该标签从旧版本的镜像转移到了新版本镜像上，那么旧版本的镜像上的标签就丢了；
- `docker build` 时指定的标签都是一样的，那么新构建的镜像拥有该标签，而之前构建的镜像就丢失了标签。

这类镜像被称为 `dangling` 虚悬镜像。这些镜像可以删除，手动删除 `dangling` 镜像：

```bash
docker rmi $(docker images -aq -f "dangling=true")
```




## 问：为什么说不要使用 import, export, save, load, commit 来构建镜像？


**答：** Docker 提供了很好的 `Dockerfile` 的机制来帮助定制镜像，可以直接使用` Shell `命令，非常方便。而且，这样制作的镜像更加透明，也容易维护，在基础镜像升级后，可以简单地重新构建一下，就可以继承基础镜像的安全维护操作。

使用 `docker commit` 制作的镜像被称为`黑箱镜像`，换句话说，就是里面进行的是黑箱操作，除本人外无人知晓。即使这个制作镜像的人，过一段时间后也不会完整的记起里面的操作。那么当有些东西需要改变时，或者因基础镜像更新而需要重新制作镜像时，会让一切变得异常困难，就如同重新安装调试配置服务器一样，失去了 Docker 的优势了。

另外，Docker 不是虚拟机，其文件系统是 `Union FS`，分层式存储，每一次 `commit` 都会建立一层，上一层的文件并不会因为 `rm `而删除，只是在当前层标记为删除而看不到了而已，每次 `docker pull` 的时候，那些不必要的文件都会如影随形，所得到的镜像也必然臃肿不堪。而且，随着文件层数的增加，不仅仅镜像更臃肿，其运行时性能也必然会受到影响。这一切都违背了 Docker 的最佳实践。

使用 `commit` 的场合是一些特殊环境，比如入侵后保存现场等等，这个命令不应该成为定制镜像的标准做法。所以，请用 `Dockerfile` 定制镜像。

`import` 和 `export` 的做法，实际上是将一个容器来保存为 `tar` 文件，然后在导入为镜像。这样制作的镜像同样是黑箱镜像，不应该使用。而且这类导入导出会导致原有分层丢失，合并为一层，而且会丢失很多相关镜像元数据或者配置，比如 `CMD` 命令就可能丢失，导致镜像无法直接启动。

`save` 和 `load` 确实是镜像保存和加载，但是这是在没有 `registry` 的情况下，手动把镜像考来考去，这是回到了十多年的 U 盘时代。这同样是不推荐的，镜像的发布、更新维护应该使用 `registry`。无论是自己架设私有 `registry` 服务，还是使用公有 `registry` 服务，如 `Docker Hub`。


## 问：Dockerfile 怎么写？

**答：** 最直接也是最简单的办法是看官方文档。

这篇文章讲述具体 Dockerfile 的命令语法：


- [https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)

然后，学习一下官方的 Dockerfile 最佳实践：

- [https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)

最后，去 `Docker Hub` 学习那些`官方(Official)镜像` `Dockerfile` 咋写的。

`Dockerfile` 不等于 `.sh` 脚本,`Dockerfile` 确实是描述如何构建镜像的，其中也提供了` RUN `这样的命令，可以运行 `shell `命令。但是和普通 `shell` 脚本还有很大的不同。

`Dockerfile` 描述的实际上是镜像的每一层要如何构建，所以每一个`RUN`是一个独立的一层。所以一定要理解“分层存储”的概念。上一层的东西不会被物理删除，而是会保留给下一层，下一层中可以指定删除这部分内容，但实际上只是这一层做的某个标记，说这个路径的东西删了。但实际上并不会去修改上一层的东西。每一层都是静态的，这也是容器本身的` immutable` 特性，要保持自身的静态特性。

所以很多新手会常犯下面这样的错误，把 `Dockerfile` 当做 `shell` 脚本来写了：

```bash
RUN yum update
RUN yum -y install gcc
RUN yum -y install python
ADD jdk-xxxx.tar.gz /tmp
RUN cd xxxx && install
RUN xxx && configure && make && make install

```

这是相当错误的。除了无畏的增加了很多层，而且很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。

正确的写法应该是把同一个任务的命令放到一个 `RUN` 下，多条命令应该用` &&` 连接，并且在最后要打扫干净所使用的环境。比如下面这段摘自官方 `redis` 镜像 `Dockerfile `的部分：

```bash
RUN buildDeps='gcc libc6-dev make' \
    && set -x \
    && apt-get update && apt-get install -y $buildDeps --no-install-recommends \
    && rm -rf /var/lib/apt/lists/* \
    && wget -O redis.tar.gz "$REDIS_DOWNLOAD_URL" \
    && echo "$REDIS_DOWNLOAD_SHA1 *redis.tar.gz" | sha1sum -c - \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && rm redis.tar.gz \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

但也不能绝对的说把所有命令都合并到一个 RUN 就对，不是把所有命令都合为一个 RUN，要合理分层，以加快构建和部署。

**合理分层就是将具有不同变更频繁程度的层，进行拆分，让稳定的部分在基础，更容易变更的部分在表层，使得资源可以重复利用，以增加构建和部署的速度。**

以 node.js 的应用示例镜像为例，其中的复制应用和安装依赖的部分，如果都合并一起，会写成这样：

```bash
COPY . /usr/src/app
RUN npm install
```

但是，在 `node.js` 应用镜像示例中，则是这么写的：

```bash
COPY package.json /usr/src/app/
RUN npm install
COPY . /usr/src/app
```

从层数上看，确实多了一层。但实际上，这三行分开是故意这样做的，其目的就是合理分层，充分利用 `Docker` 分层存储的概念，以增加构建、部署的效率。

在 `docker build` 的构建过程中，如果某层之前构建过，而且该层未发生改变的情况下，那么 docker 就会直接使用缓存，不会重复构建。因此，合理分层，充分利用缓存，会显著加速构建速度。

第一行的目的是将 `package.json` 复制到应用目录，而不是整个应用代码目录。这样只有 `pakcage.json` 发生改变后，才会触发第二行 `RUN npm install`。而只要 `package.json `没有变化，那么应用的代码改变就不会引发 `npm install`，只会引发第三行的 `COPY . /usr/src/app`，从而加快构建速度。

而如果按照前面所提到的，合并为两层，那么任何代码改变，都会触发 `RUN npm install`，从而浪费大量的带宽和时间。

合理分层除了可以加快构建外，还可以加快部署，要知道，`docker pull` 的时候，是分层下载的，并且已存在的层就不会重复下载。

比如，这里的 `RUN npm install` 这一层，往往会几百 `MB` 甚至上 `GB`。而在 `package.json` 未发生变更的情况下，那么只有 `COPY . /usr/src/app` 这一层会被重新构建，并且也只有这一层会在各个节点 `docker pull` 的过程中重新下载，往往这一层的代码量只有几十 `MB`，甚至更小。这对于大规模的并行部署中，所节约的东西向流量是非常显著的。特别是敏捷开发环境中，代码变更的频繁度要比依赖变更的频繁度高很多，每次重复下载依赖，会导致不必要的流量和时间上的浪费。


## 问：context 到底是一个什么概念？

**答：** `context`，上下文，是 `docker build` 中很重要的一个概念。构建镜像必须指定 `context`：

```bash
docker build -t xxx <context路径>
```

或者 `docker-compose.yml` 中的

```bash

app:
    build:
        context: <context路径>
        dockerfile: dockerfile
```

这里都需要指定 `context`。
`context` 是工作目录，但不要和构建镜像的`Dockerfile` 中的` WORKDIR `弄混，`context` 是 `docker build `命令的工作目录。

`docker build` 命令实际上是客户端，真正构建镜像并非由该命令直接完成。`docker build` 命令将 `context` 的目录上传给 `Docker` 引擎，由它负责制作镜像。

在 `Dockerfile` 中如果写 `COPY ./package.json /app/ `这种命令，实际的意思并不是指执行 `docker build` 所在的目录下的 `package.json`，也不是指 Dockerfile 所在目录下的` package.json`，而是指 `context` 目录下的 `package.json`。

这就是为什么有人发现 `COPY ../package.json /app` 或者 `COPY /opt/xxxx /app `无法工作的原因，因为它们都在 `context` 之外，如果真正需要，应该将它们复制到 `context `目录下再操作。

话说，有一些网文甚至搞笑的说要把 `Dockerfile `放到磁盘根目录，才能构建如何如何。这都是对 `context` 完全不了解的表现。想象一下把整个磁盘几十个 `GB`当做上下文发送给 `dockerd` 引擎的情况。

`docker build -t xxx .` 中的这个`.`，实际上就是在指定 `Context` 的目录，而并非是指定 `Dockerfile` 所在目录。

默认情况下，如果不额外指定 `Dockerfile` 的话，会将 `Context` 下的名为 `Dockerfile` 的文件作为 `Dockerfile`。所以很多人会混淆，认为这个` .` 是在说 `Dockerfile` 的位置，其实不然。

一般项目中，`Dockerfile` 可能被放置于两个位置。

- 一个可能是放置于项目顶级目录，这样的好处是在顶级目录构建时，项目所有内容都在上下文内，方便构建；
- 另一个做法是，将所有 Docker 相关的内容集中于某个目录，比如 `docker` 目录，里面包含所有不同分支的` Dockerfile`，以及 `docker-compose.yml `类的文件、`entrypoint `的脚本等等。这种情况的上下文所在目录不再是 `Dockerfile` 所在目录了，因此需要注意指定上下文的位置。


此外，项目中可能会包含一些构建不需要的文件，这些文件不应该被发送给 `dockerd` 引擎，但是它们处于上下文目录下，这种情况，我们需要使用 `.dockerignore `文件来过滤不必要的内容。`.dockerignore` 文件应该放置于上下文顶级目录下，内容格式和 `.gitignore` 一样。

```bash
tmp
db
```

这样就过滤了 `tmp` 和` db `目录，它们不会被作为上下文的一部分发给 `dockerd `引擎。



## 问：ENTRYPOINT 和 CMD 到底有什么不同？

**答：** `Dockerfile` 的目的是制作镜像，换句话说，实际上是准备的是主进程运行环境。那么准备好后，需要执行一个程序才可以启动主进程，而启动的办法就是调用 `ENTRYPOINT`，并且把 `CMD` 作为参数传进去运行。也就是下面的概念：

```bash
ENTRYPOINT "CMD"
```

假设有个 `myubuntu` 镜像 `ENTRYPOINT` 是 `sh -c`，而我们 `docker run -it myubuntu uname -a`。那么 `uname -a `就是运行时指定的 `CMD`，那么 Docker 实际运行的就是结合起来的结果：

```bash
sh -c "uname -a"
```

- 如果没有指定 `ENTRYPOINT`，那么就只执行 `CMD`；

- 如果指定了 `ENTRYPOINT` 而没有指定 `CMD`，自然执行 `ENTRYPOINT`;

- 如果 `ENTRYPOINT` 和 `CMD` 都指定了，那么就如同上面所述，执行 `ENTRYPOINT "CMD"`；

- 如果没有指定 `ENTRYPOINT`，而 `CMD` 用的是上述那种 `shell` 命令的形式，则自动使用 `sh -c` 作为 `ENTRYPOINT`。

注意最后一点的区别，这个区别导致了同样的命令放到 `CMD` 和 `ENTRYPOINT` 下效果不同，因此有可能放在 `ENTRYPOINT` 下的同样的命令，由于需要 `tty` 而运行时忘记了给（比如忘记了`docker-compose.yml` 的 `tty:true`）导致运行失败。

这种用法可以很灵活，比如我们做个 `git` 镜像，可以把 `git` 命令指定为 `ENTRYPOINT`，这样我们在 `docker run` 的时候，直接跟子命令即可。比如 `docker run git log `就是显示日志。

## 问：拿到一个镜像，如何获得镜像的 Dockerfile ？

**答：**

- 直接去 `Docker Hub` 上看：大多数 `Docker Hub `上的镜像都会有 `Dockerfile`，直接在 `Docker Hub` 的镜像页面就可以看到 `Dockerfile` 的链接；

- 如果没有 `Dockerfile`，一般这类镜像就不应该考虑使用了，这类黑箱似的镜像很容有有问题。如果是什么特殊原因，那继续往下看；

- `docker history` 可以看到镜像每一层的信息，包括命令，当然黑箱镜像的 `commit` 看不见操作；

- `docker inspect` 可以分析镜像很多细节。

- 直接运行镜像，进入`shell`，然后根据上面的分析结果去进一步分析日志、文件内容及变化。

## 问：Docker 日志都在哪里？

日志分两类，一类是 `Docker 引擎日志`；另一类是 `容器日志`。

**Docker 引擎日志**

Docker 引擎日志 一般是交给了 `Upstart(Ubuntu 14.04) `或者 `systemd (CentOS 7, Ubuntu 16.04)`。前者一般位于 `/var/log/upstart/docker.log` 下，后者一般通过 `jounarlctl -u docker`来读取或系统日志里面`/var/log/messages` 。

**容器日志**

容器的日志 则可以通过 `docker logs` 命令来访问，而且可以像 `tail -f` 一样，使用 `docker logs -f` 来实时查看。如果使用 `Docker Compose`，则可以通过 `docker-compose logs <服务名>`来查看。

如果深究其日志位置，每个容器的日志默认都会以 `json-file` 的格式存储于` /var/lib/docker/containers/<容器id>/<容器id>-json.log `下，不过并不建议去这里直接读取内容，因为 Docker 提供了更完善地日志收集方式 - Docker 日志收集驱动。

关于日志收集，Docker 内置了很多日志驱动，可以通过类似于` fluentd`, `syslog` 这类服务收集日志。无论是 Docker 引擎，还是容器，都可以使用日志驱动。比如，如果打算用 `fluentd` 收集某个容器日志，可以这样启动容器：

```bash
$ docker run -d \
    --log-driver=fluentd \
    --log-opt fluentd-address=10.2.3.4:24224 \
    --log-opt tag="docker.{{.Name}}" \
    nginx
```

其中 `10.2.3.4:24224` 是 `fluentd` 服务地址，实际环境中应该换成真实的地址。

## 问：不同容器的日志汇聚到 fluentd 后如何区分？

**答：** 有两种概念的区分，一种是区分开不同容器的日志，另一种是区分开来不同服务的日志。

区分不同容器的日志是很直观的想法。运行了几个不同的容器，日志都送向日志收集，那么显然不希望` nginx `容器的日志和 `MySQL` 容器的日志混杂在一起看。

但是在` Swarm` 集群环境中，区分容器就已经不再是合理的做法了。因为同一个服务可能有许多副本，而又有很多个服务，如果一个个的容器区分去分析，很难看到一个整体上某个服务的服务状态是什么样子的。而且，容器是短生存周期的，在维护期间容器生存死亡是很常见的事情。如果是像传统虚拟机那样子以容器为单元去分析日志，其结果很难具有价值。因此更多的时候是对某一个服务的日志整体分析，无需区别日志具体来自于哪个容器，不需要关心容器是什么时间产生以及是否消亡，只需要以服务为单元去区分日志即可。

这两类的区分日志的办法，Docker 都可以做到，这里我们以 `fluentd` 为例说明。

```bash
version: '2'
services:
    web:
        image: nginx:1.11-alpine
        ports:
            - "3000:80"
        labels:
            section: frontend
            group: alpha
            service: web
            image: nginx
            base_os: alpine
        logging:
            driver: fluentd
            options:
                fluentd-address: "localhost:24224"
                tag: "frontend.web.nginx.{{.Name}}"
                labels: "section,group,service,image,base_os"
```

这里我们运行了一个 `nginx:alpine` 的容器，服务名为 `web`。容器的日志使用 `fluentd` 进行收集，并且附上标签 `frontend.web.nginx.<容器名>`。除此以外，我们还定义了一组 `labels`，并且在 `logging` 的 `options` 中的 `labels `中指明希望哪些标签随日志记录。这些信息中很多一部分都会出现在所收集的日志里。

让我们来看一下 `fluentd `收到的信息什么样子的。


```bash
{
  "frontend.web.nginx.service_web_1": {
    "image": "nginx",
    "base_os": "alpine",
    "container_id": "f7212f7108de033045ddc22858569d0ac50921b043b97a2c8bf83b1b1ee50e34",
    "section": "frontend",
    "service": "web",
    "log": "172.20.0.1 - - [09/Dec/2016:15:02:45 +0000] \"GET / HTTP/1.1\" 200 612 \"-\" \"curl/7.49.1\" \"-\"",
    "group": "alpha",
    "container_name": "/service_web_1",
    "source": "stdout",
    "remote": "172.20.0.1",
    "host": "-",
    "user": "-",
    "method": "GET",
    "path": "/",
    "code": "200",
    "size": "612",
    "referer": "-",
    "agent": "curl/7.49.1",
    "forward": "-"
  }
}
```


如果去除` nginx` 正常的访问日志项目外，我们就可以更清晰的看到有哪些元数据信息可以利用了。


```bash

  "frontend.web.nginx.service_web_1": {
    "image": "nginx",
    "base_os": "alpine",
    "container_id": "f7212f7108de033045ddc22858569d0ac50921b043b97a2c8bf83b1b1ee50e34",
    "section": "frontend",
    "service": "web",
    "group": "alpha",
    "container_name": "/service_web_1",
    "source": "stdout",
  }
}
```


可以看到，我们在 `logging` 下所有指定的 `labels` 都在。我们完全可以对每个服务设定不同的标签，通过标签来区分服务。
比如这里，我们对 `web` 服务指定了 `service=web` 的标签，我们同样可以对数据库的服务设定标签为 `service=mysql`，这样在汇总后，只需要对 `service` 标签分组过滤即可，分离聚合不同服务的日志。

此外，我们可以设置不止一个标签，比如上面的例子，我们设置了多组不同颗粒度的标签，在后期分组的时候，可以很灵活的进行组合，以满足不同需求。

此外，注意 `frontend.web.nginx.service_web_1`，这是我们之前利用 `--log-opt tag=frontend.web.nginx.<容器名>` 进行设定的，其中<容器名> 我们使用的是 Go 模板表达式。Go 模板很强大，我们可以用它实现非常复杂的标签。在 `fluentd` 中，`<match> `项可以根据标签来进行筛选。

这里可以唯一表示容器的，有容器 `ID container_id`，而容器名` container_name` 也从某种程度上可以用来区分不同容器。因此进行容器区分日志的时候，可以使用这两项。

还有一个 `source`，这表示了日志是从标准输出还是标准错误输出得到的，由此可以区分正常日志和错误日志。

现在我们可以知道，除了容器自身输出的信息外，Docker 还可以为每一个容器的日志添加很多元数据，以帮助后期的日志处理中应对不同需求的搜索和过滤。

在后期处理中，`fluentd `中可以利用` <match>` 或者 `<filter>` 插件根据 `tag` 或者其它元数据进行分别处理。而日志到了 `ElasticSearch `这类系统后，则可以用更丰富的查询语言进行过滤、聚合。

## 问：为什么容器一运行就退出啊？

**答：** 这是初学 Docker 常常碰到的问题，此时还以虚拟机来理解 Docker，认为启动 Docker 就是启动虚拟机，也没有搞明白前台和后台的区别。

首先，碰到这类问题应该查日志和容器主进程退出码。

检查容器日志：

```bash
docker logs <容器ID>
```

查看容器退出码：


```bash
CONTAINER ID        IMAGE                           COMMAND             CREATED             STATUS                      PORTS                                                                  NAMES
cc2aa3f4745f        ubuntu                          "/bin/bash"         23 hours ago        Exited (0) 22 hours ago                                                                            clever_lewin
25510a2cb171        twang2218/gitlab-ce-zh:8.15.3   "/assets/wrapper"   2 days ago          Exited (127) 2 days ago
```


在 `STATUS` 一栏中，可以看到退出码是多少。

- 如果看到了 `Exited (127)` 那很可能是由于内存超标导致触发 `Out Of Memory` 然后被强制终止了。

- 如果看到了 `Exited (0)`，这说明容器主进程正常退出了。

- 如果是其他情况，应该检查容器日志。

初学 Docker 的人常常会不理解既然正常怎么会退出的意思。不得不在强调一遍，Docker 不是虚拟机，容器只是进程。因此当执行 `docker run`的时候，实际所做的只是启动一个进程，如果进程退出了，那么容器自然就终止了。

那么进程为什么会退出？

- 如果是执行 `service nginx start` 这类启动后台服务程序的命令，那说明还是把 Docker 当做虚拟机了。Docker 启动的是进程，因此所谓的后台服务应该放到前台，比如应该 `nginx -g 'daemon off;' `这样直接前台启动应用才对。

- 如果发现 `COMMAND` 一栏是 `/bin/bash`，那还是说明把 Docker 当虚拟机了。`COMMAND `应该是应用程序，而不交互式操作界面，容器不需要交互式操作界面。此外，如果使用 `/bin/bash` 希望起一个交互式的界面，那么也必须提供给其输入和终端，因此必须加 `-it` 选项，比如 `docker run -it ubuntu /bin/bash`


## 问：我想在docker容器里面运行docker命令，该如何操作？

**答：** 首先，不要在 Docker 容器中安装、运行 Docker 引擎，也就是所谓的 `Docker In Docker (DIND)`

因为Docker-in-Docker有很多问题和缺陷，参考文章：

- [https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/)


为了让容器内可以构建镜像，应该使用 `Docker Remote API` 的客户端来直接调用宿主的 `Docker Engine`。可以是原生的 `Docker CLI （docker 命令）`，也可以是其它语言的库。

为 `Jenkins` 添加 Docker 命令行


```bash
FROM jenkins:alpine
# 下载安装Docker CLI
USER root
RUN curl -O https://get.docker.com/builds/Linux/x86_64/docker-latest.tgz \
    && tar zxvf docker-latest.tgz \
    && cp docker/docker /usr/local/bin/ \
    && rm -rf docker docker-latest.tgz
# 将 `jenkins` 用户的组 ID 改为宿主 `docker` 组的组ID，从而具有执行 `docker` 命令的权限。
ARG DOCKER_GID=999
USER jenkins:${DOCKER_GID}
```


在这个例子里，我们下载了静态编译的 `docker` 可执行文件，并提取命令行安装到系统目录下。然后调整了 `jenkins` 用户的组 `ID`，调整为宿主 `docker `组`ID`，从而使其具有执行 `docker` 命令的权限。

组 `ID` 使用了 `DOCKER_GID` 参数来定义，以方便进一步定制。构建时可以通过 `--build-arg `来改变 `DOCKER_GID` 的默认值，运行时也可以通过 `--user jenkins:1234` 来改变运行用户的身份。

用下面的命令来构建镜像（假设镜像名为 `jenkins-docker`）：


```bash
$ docker build -t jenkins-docker .
```


如果需要构建时调整 `docker` 组 `ID`，可以使用 `--build-arg` 来覆盖参数默认值：


```bash
$ docker build -t jenkins-docker --build-arg DOCKER_GID=1234 .
```


在启动容器的时候，将宿主的 `/var/run/docker.sock` 文件挂载到容器内的同样位置，从而让容器内可以通过 `unix socket` 调用宿主的 Docker 引擎。

比如，可以用下面的命令启动 `jenkins`：


```bash
$ docker run --name jenkins \
    -d \
    -p 8080:8080 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    jenkins-docker
```


在 `jenkins` 容器中，就已经可以执行 `docker` 命令了，可以通过 `docker exec `来验证这个结果：


```bash
$ docker exec -it jenkins sh
/ $ id
uid=1000(jenkins) gid=999(ping) groups=999(ping)
/ $ docker version
Client:
 Version:      1.12.3
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   6b644ec
 Built:        Wed Oct 26 23:26:11 2016
 OS/Arch:      linux/amd64
Server:
 Version:      1.13.0-rc2
 API version:  1.25
 Go version:   go1.7.3
 Git commit:   1f9b3ef
 Built:        Wed Nov 23 06:32:39 2016
 OS/Arch:      linux/amd64
/ $
```


## 问：都说不要用 root 去运行服务，但我看到的 Dockerfile 都是用 root 去运行，这不安全吧？

**答：** 并非所有官方镜像的 `Dockerfile` 都是用 `root` 用户去执行的。比如`mysql` 镜像的执行身份就是 `mysql` 用户；`redis` 镜像的服务运行用户就是 `redis`；`mongo `镜像内的服务执行身份是 `mongo` 用户；`jenkins` 镜像内是 `jenkins` 用户启动服务等等。所以说 `“都是用 root 去运行”` 是不客观的。

当然，这并不是说在容器内使用 `root` 就非常危险。容器内的 `root` 和宿主上的 `root` 不同，容器内的 `root` 虽然 `uid` 也默认为 `0`，但是却处于一个隔离的命名空间，而且被去掉了大量的特权。容器内的 `root` 是一个没有什么特权的用户，危险的操作基本都无法执行。

不过，如果用户可以打破这个安全保护，那就是另外一回事了。比如，如果用户挂载了宿主目录给容器，这就是打通了一个容器内的 `root `操控宿主的一个通道，使得容器内的 `root` 可以修改所挂载的目录下的任何文件。

因为当前版本的 Docker 中，默认情况下容器的 `user namespace` 并未开启，所以容器内的用户和宿主用户共享 `uid` 空间。容器内的 `uid` 为 `0 `的 `root`，就被系统视为 `uid=0` 的宿主 `root`，因此磁盘读写时，具有宿主` root` 同等读写权限。这也是为什么一般不推荐挂载宿主目录、特别是挂载宿主系统目录的原因之一。这一切只要定制镜像的时候，容器内不使用` root` 启动服务就没这个问题了。

当然，上面说的问题只是默认情况下 `user namespace `不会启用的问题。`dockerd` 有一个 `--userns-remap` 参数，只要配置了这个参数，就可以确保容器内的 `uid` 是独立命名空间，容器内的 `uid `变到宿主的时候，会被` remap` 到另一个范围。因此，容器内的 `uid=0` 的 `root` 将完全跟 `root `没有任何关系，仅仅是个普通用户而已。

相关信息请参考官方文档：

- `--userns-remap` 的介绍：[https://docs.docker.com/engine/reference/commandline/dockerd/#/daemon-user-namespace-options](https://docs.docker.com/engine/reference/commandline/dockerd/#/daemon-user-namespace-options)


- Docker 安全：[https://docs.docker.com/engine/security/security/](https://docs.docker.com/engine/security/security/)

## 问：我在容器里运行 systemctl start xxx 怎么报错啊？

**答：** 如果在容器内使用 `systemctl` 命令，经常会发现碰到这样的错误：


```bash
Failed to get D-Bus connection: Operation not permitted
```


这很正常，因为 `systemd` 是完整系统的服务启动、维护的系统服务程序，而且需要特权去执行。但是容器不是完整系统，既没有配合的服务，也没有特权，所以自然用不了。

如果你碰到这样的问题，只能再次提醒你，Docker 不是虚拟机。试图在容器里执行 `systemctl` 命令的，大多都是还没有搞明白容器和虚拟机的区别，因为看到了可以有 `Shell`，就以为这是个虚拟机，试图重复自己在完整系统上的体验。这是用法错误，不要把 Docker 当做虚拟机去用，容器有自己的用法。

**Docker 不是虚拟机，容器只是受限进程。**

容器内根本不需要后台服务，也不需要服务调度和维护，自然也不需要 `systemd`。容器只有一个主进程，也就是应用进程。容器的生存周期就是围绕着这个主进程而存在的，所以所试图启动的后台服务，应该改为直接在前台运行，根本不需要也不应该使用 `systemctl` 命令去在后台加载。日志之类的也是直接从 `stdout/stderr` 输出，而不是走 `journald`。

## 问：容器内的时间和宿主不一致，如何处理？

**答：** 一般情况直接设置环境变量 `TZ` 就够了，比如：


```bash
$ docker run -it -e TZ=Asia/Shanghai debian bash
root@8e6d6c588328:/# date
Tue Dec 13 09:41:21 CST 2016
```


看到了么？时区调整到了 `CST`，也就是 `China Standard Time - 中国标准时间` ，因此显示就正常了。

还有一种方法在构建镜像的时侯调整下时区：


```bash
FROM centos:7
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
```


## 问：我想让我的程序平滑退出，为什么截获 SIGTERM 信号不管用啊？

**答：** `docker stop`, `docker service rm` 在停止容器时，都会先发 `SIGTERM` 信号，等待一段时间`（默认为 10 秒）`后，如果程序没响应，则强行 `SIGKILL `杀掉进程。

这样应用进程就有机会平滑退出，在接收到` SIGTERM` 后，可以去 Flush 缓存、完成文件读写、关闭数据库连接、释放文件资源、释放锁等等，然后再退出。所以试图截获 SIGTERM 信号的做法是对的。

但是，可能在截获 `SIGTERM` 时却发现，却发现应用并没有收到 `SIGTERM`，于是盲目的认为 Docker 不支持平滑退出，其实并非如此。

还记得我们提到过，Docker 不是虚拟机，容器只是受限进程，而一个容器只应该跑一个主进程的说法么？如果你发现你的程序没有截获到 `SIGTERM`，那就很可能你没有遵循这个最佳实践的做法。因为` SIGTERM `只会发给主进程，也就是容器内 `PID `为` 1` 的进程。

至于说主进程启动的那些子进程，完全看主进程是否愿意转发 `SIGTERM` 给子进程了。所以那些把 Docker 当做虚拟机用的，主进程跑了个` bash`，然后 `exec `进去启动程序的，或者来个 `&` 让程序跑后台的情况，应用进程必然无法收到 `SIGTERM`。

还有一种可能是在 Dockerfile 中的 `CMD` 那行用的是 `shell` 格式写的命令，而不是 `exec `格式。还记得前面提到过的 `shell` 格式的命令，会加一个 `sh -c` 来去执行么？因此使用 `shell` 格式写 `CMD` 的时候，`PID` 为 `1` 的进程是 `sh`，而它不转发信号，所以主程序收不到。

明白了道理，解决方法就很简单，换成 `exec` 格式，并且将主进程执行文件放在第一位即可。这也是为什么之前推荐 `exec` 格式的原因之一。

进程管理在Docker容器中和在完整的操作系统有一些不同之处。在每个容器的`PID1`进程，需要能够正确的处理`SIGTERM`信号来支持容器应用的优雅退出，同时要能正确的处理孤儿进程和僵尸进程。必要的时候使用Docker新提供的 `docker run --init` 参数可以解决相应问题。

## 问：Docker Swarm（一代swarm） 和Swarm mode（二代swarm）两者的区别是什么？

**答：** 因为 `docker run` 和 `docker service create `是两个不同理念的东西。

一代 Swarm 中，将 Swarm 集群视为一个巨大的 Docker 主机，本质上和单机没有区别，都是直接调度运行容器。因此依旧使用单机的 `docker run `的方式来启动特定容器。

二代 Swarm 则改变了这个理念，增加了`服务栈(Stack)`、`服务(Service)`、`任务(Task)` 的概念。在二代 Swarm 中，一组服务可以组成一个整体进行部署，也就是部署服务栈，这相当于是之前的` Docker Compose` 所完成的目的。但是这次，是真正的针对服务的。

一个服务并非一个容器，一个服务可以有多个副本任务，每个任务对应一个容器。这个概念在一代 Swarm 和单机环境中是没有的，因此 `Docker Compose `为了实现服务的概念，用了各种办法去模拟，包括使用 `labels`，使用网络别名等等，但是本质上，依旧是以容器为单位进行运行，也就是本质上还是一组 `docker run`。

正是由于二代 Swarm 中用户操作的单元是服务，所以传统的以容器为中心的 `docker run` 就不再适用，因此有新的一组针对服务的命令，`docker service`。

## 问：我自建了私有镜像仓库Registry，我如何搜索查询仓库中的镜像？

**答：** 如果只使用开源的 `docker registry` 自建仓库的话，目前只能用 `API` 访问其内容。除此以外，官方还有商业版的 [Docker Trusted Registry](https://docs.docker.com/datacenter/dtr/2.0/) 项目，里面有一些增值的内容在里面，提供了类似于 Docker Hub 似得 UI 等，可以搜索过滤。目前 Docker Trusted Registry 属于 Docker Datacenter 的一部分。

另外，第三方也有一些提供了UI的。比如 [VMWare Harbor](https://github.com/vmware/harbor)。VMWare Harbor是 VMWare 中国基于开源 `docker registry `进一步开发的项目，有更复杂的上层逻辑。包括用户管理、镜像管理、Registry集群之类的功能。Harbor 是开源的，免费的。

第三方的 `registry` 还有 `Java` 世界里常见的 `Nexus`，其第三代支持 `Docker Registry API`。

或者自己可以编写个脚本去查询：


```python
#!/usr/bin/env python
import requests
#import simplejson as json
import json
import sys

def main():
  registry = "http://127.0.0.1:5000" ## 自己镜像仓库地址
  res = requests.get(registry + "/v2/")
  #assert res.status_code == 200

  res = requests.get(registry + "/v2/_catalog?n=100000")
  assert res.status_code == 200
  repositories = res.json().get("repositories", [])
  #print("registry reports {} repositories".format(len(repositories)))

  for repository in repositories:
    res = requests.get(registry + "/v2/{}/tags/list".format(repository))
    tags = res.json().get("tags", None)
    if tags:
      for tag in tags:
        image = format(repository)
        tag = format(tag)
        print image+":"+tag

if __name__ == "__main__":
  main()

```



## 问：如何删除私有 registry 中的镜像？

**答：** 首先，在默认情况下，`docker registry` 是不允许删除镜像的，需要在配置`config.yml`中启用：


```bash
delete:
    enabled: true
```


然后，使用 `API GET /v2/<镜像名>/manifests/<tag>` 来取得要删除的 `镜像:Tag` 所对应的 `digest` 。

`Registry 2.3` 以后，必须加入头 `Accept: application/vnd.docker.distribution.manifest.v2+json` ，否则取到的 `digest` 是错误的，这是为了防止误删除。

比如，要删除 `myimage:latest` 镜像，那么取得 `digest` 的命令是：


```bash
$ curl --header "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  -I -X HEAD http://192.168.99.100:5000/v2/myimage/manifests/latest \
  | grep Digest
Docker-Content-Digest: sha256:3a07b4e06c73b2e3924008270c7f3c3c6e3f70d4dbb814ad8bff2697123ca33c
```


然后调用  `API DELETE /v2/<镜像名>/manifests/<digest> ` 来删除镜像。比如：


```bash
curl  -X DELETE http://192.168.99.100:5000/v2/myimage/manifests/sha256:3a07b4e06c73b2e3924008270c7f3c3c6e3f70d4dbb814ad8bff2697123ca33c
```


至此，镜像已从 `registry` 中标记删除，外界访问 `pull` 不到了。但是 `registry ` 的本地空间并未释放，需要等待垃圾收集才会释放。而垃圾收集不可以在线进行，必须停止 `registry`，然后执行。比如，假设 `registry `是用` Compose ` 运行的，那么下面命令用来垃圾收集：


```bash
docker-compose stop
docker run -it --name gc --rm --volumes-from registry_registry_1 registry:2 garbage-collect /etc/registry/config.yml
docker-compose start
```


其中 `registry_registry_1` 可以替换为实际的 `registry` 的容器名，而 `/etc/registry/config.yml` 则替换为实际的 `registry` 配置文件路径。

参考官网文档：


- [https://docs.docker.com/registry/configuration/#/delete](https://docs.docker.com/registry/configuration/#/delete)

- [https://docs.docker.com/registry/spec/api/#/deleting-an-image](https://docs.docker.com/registry/spec/api/#/deleting-an-image)


## 问：自己架的 registry 怎么任何用户都可以取到镜像？这不安全啊？

**答：** 那是因为没有加认证，不加认证的意思就是允许任何人访问的。

添加认证有两种方式：

- Registry 配置中加入认证： [https://docs.docker.com/registry/configuration/#/auth](https://docs.docker.com/registry/configuration/#/auth)

```bash
auth:
  token:
    realm: token-realm
    service: token-service
    issuer: registry-token-issuer
    rootcertbundle: /root/certs/bundle
  htpasswd:
    realm: basic-realm
    path: /path/to/htpasswd
```

- 前端架设 nginx 进行认证：[https://docs.docker.com/registry/recipes/nginx/](https://docs.docker.com/registry/recipes/nginx/)

```bash
location /v2/ {
    ...
    auth_basic "Registry realm";
    auth_basic_user_file /etc/nginx/conf.d/nginx.htpasswd;
    ...
}
```

使用VMWare Harbor部署镜像仓库，Harbor 提供了高级的安全特性，诸如用户管理，访问控制和活动审计等。

## 问：CentOS 7 默认的内核太老了 3.10，是不是很多 Docker 功能不支持？
**答：** 是的，有一些功能无法支持，比如 `overlay2` 的存储驱动就无法在` CentOS` 上使用，但并非所有需要高版本内核的功能都不支持。

比如 `Overlay FS` 需要 `Linux 3.18`，而` Overlay network` 需要 `Linux 3.16`。而 `CentOS 7` 内核为 `3.10`，确实低于这些版本需求。但实际上，红帽团队会把一些新内核的功能 `backport` 回老的内核。比如` overlay fs`等。所以一些功能依旧会支持。因此 `CentOS 7 `的 `Docker Engine` 同样可以支持 `overlay network`，以及` overlay` 存储驱动（不是`overlay2`）。因此在新的 `Docker 1.12` 中，`CentOS/RHEL 7` 才有可能支持 `Swarm Mode`。

即使红帽会把一些高版本内核的功能 `backport` 回 `3.10 `内核中，这种修修补补出来的功能，并不一定稳定。如果观察 `Docker Issue` 列表，会发现大量的由于 `CentOS` 老内核导致的问题，特别是在使用了 `1.12` 内置的 `Swarm Mode` 集群功能后，存储、网络出现的问题很多。

所以想要在Centos 系统上更好的使用Docker，建议检查和升级下系统内核：

```bash
wget http://mirror.rc.usf.edu/compute_lock/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.11.1-1.el7.elrepo.x86_64.rpm
yum -y install linux-firmware
rpm -ivh kernel-ml-4.11.1-1.el7.elrepo.x86_64.rpm
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
package-cleanup --oldkernels --count=1 -y
```

然后需要重启下机器以启用新的内核。

## 听说 Windows 10、Windows Server 2016 内置 Docker 了？和 Docker 官网下载的 Docker for Windows 有什么区别啊？

二者完全不同。

`Windows 10` 或者 `Windows Server 2016` 自带的 Docker，被称为 `Docker on Windows`，其运行于 `Windows NT `内核至上，以 Docker 类似的方式提供 Windows 容器服务，因此只可以运行 Windows 程序。

而 Docker 官网下载的，被称为 `Docker for Windows`。这是我们常说的 Docker，它是运行于 `Linux` 内核上的 Docker。在 Windows 上运行时实际上是在` Hyper-V` 上的一个 `Alpine Linux` 虚拟机上运行的 Docker。它只可以运行 `Linux` 程序。

`Docker on Windows` 极为臃肿，最小镜像也近 `GB`，启动时间并不快；而 `Docker for Windows` 则是正常的 Docker，最小镜像也就几十 `KB`，一般的镜像都在几百兆以内，而且启动时间基本是毫秒级。

--------------------------------------------------------------------------------------------------------------------------------------

**未完待续**
