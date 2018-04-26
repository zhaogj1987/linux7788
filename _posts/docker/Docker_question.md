title: 关于的Docker问与答
date: 2018/04/26 10:32:25
categories:
- Docker
tags:
- Docker

---

## 问：Docker 和 传统的虚拟机有什么差别？
答：我们知道inux系统将自身划分为两部分，一部分为核心软件，也称作内核空间(`kernel`)，另一部分为普通应用程序，这部分称为用户空间(`userland`)。
容器内的进程是直接运行于宿主内核的，这点和宿主进程一致，只是容器的 `userland` 不同，容器的 `userland` 由容器镜像提供，也就是说镜像提供了 `rootfs`。
假设宿主是 `Ubuntu`，容器是 `CentOS`。`CentOS` 容器中的进程会直接向 `Ubuntu` 宿主内核发送 `syscall`，而不会直接或间接的使用任何 `Ubuntu` 的 `userland` 的库。
这点和虚拟机有本质的不同，虚拟机是虚拟环境，在现有系统上虚拟一套物理设备，然后在虚拟环境内运行一个虚拟环境的操作系统内核，在内核之上再跑完整系统，并在里面调用进程。
还以上面的例子去考虑，虚拟机中，`CentOS` 的进程发送 `syscall` 内核调用，该请求会被虚拟机内的 `CentOS` 的内核接到，然后 `CentOS` 内核访问虚拟硬件时，由虚拟机的服务软件截获，并使用宿主系统，也就是 `Ubuntu` 的内核及 `userland` 的库去执行。
而且，Linux 和 Windows 在这点上非常不同。Linux 的进程是直接发 `syscall` 的，而 Windows 则把 `syscall` 隐藏于一层层的 `DLL` 服务之后，因此 Windows 的任何一个进程如果要执行，不仅仅需要 Windows 内核，还需要一群服务来支撑，所以如果 Windows 要实现类似的机制，容器内将不会像 Linux 这样轻量级，而是非常臃肿。看一下微软移植的 Docker 就非常清楚了。
所以不要把 Docker 和虚拟机弄混，Docker 容器只是一个进程而已，只不过利用镜像提供的 `rootfs` 提供了调用所需的 `userland` 库支持，使得进程可以在受控环境下运行而已，它并没有虚拟出一个机器出来。
## 问：如何安装 Docker？
答：很多人问到 `docker`, `docker.io`, `docker-engine` 甚至 `lxc-docker` 都有什么区别？
其中，RHEL/CentOS 软件源中的 Docker 包名为 `docker`；Ubuntu 软件源中的 Docker 包名为 `docker.io`；而很古老的 Docker 源中 Docker 也曾叫做 `lxc-docker`。这些都是非常老旧的 Docker 版本，并且基本不会更新到最新的版本，而对于使用 Docker 而言，使用最新版本非常重要。另外，17.04 以后，包名从 `docker-engine` 改为 `docker-ce`，因此从现在开始安装，应该都使用 `docker-ce `这个包。
正确的安装方法有两种：
- 一种是参考官方安装文档去配置 apt 或者 yum 的源；
- 另一种则是使用官方提供的安装脚本快速安装。
官方文档对配置源的方法已经有很详细的讲解:[官方文档地址](https://docs.docker-cn.com/engine/installation/linux/docker-ce/centos/)

<!--more-->
**17.04 及以后的版本安装方法***：
从 `17.04` 以后，可以用下面的命令安装。

```bash
export CHANNEL=stable
curl -fsSL https://get.docker.com/ | sh -s -- --mirror Aliyun
```
这里使用的是官方脚本安装，通过环境变量指定安装通道为 `stable`，（如果喜欢尝鲜可以改为 `edge`, `test`），并且指定使用阿里云的源(`apt/yum`)来安装 `Docker CE` 版本。

**17.03 及以前的版本安装方法***：
早期的版本可以使用阿里云或者 DaoCloud 老的脚本安装：
使用`阿里云`的安装脚本：

```bash
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
```
使用`DaoCloud`的Docker安装脚本：

```bash
curl -sSL https://get.daocloud.io/docker | sh
```

**Centos7机器直接用RPM包安装方法***：
可去阿里云寻找到最新的rpm包并下载安装：[阿里云docker rpm地址]:(https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/)

```bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-18.03.0.ce-1.el7.centos.x86_64.rpm -O docker-ce-18.03.0.ce-1.el7.centos.x86_64.rpm
yum install -y container-selinux
rpm -ivh docker-ce-17.12.0.ce-1.el7.centos.x86_64.rpm
```
## 问：Docker pull 镜像如何加速？
我们可以使用 Docker 镜像加速器来解决这个问题，加速器就是镜像、代理的概念。国内有不少机构提供了免费的加速器以方便大家使用，这里列出一些常用的加速器服务：

- Docker 官方的中国镜像加速器：从2017年6月9日起，Docker 官方提供了在中国的加速器，以解决墙的问题。不用注册，直接使用加速器地址：`https://registry.docker-cn.com` 即可。
- 中国科技大学的镜像加速器：中科大的加速器不用注册，直接使用地址[https://docker.mirrors.ustc.edu.cn/](https://docker.mirrors.ustc.edu.cn/)配置加速器即可。进一步的信息可以访问：[http://mirrors.ustc.edu.cn/help/dockerhub.html?highlight=docker](http://mirrors.ustc.edu.cn/help/dockerhub.html?highlight=docker)
- 阿里云加速器：注册阿里云开发账户(免费的)后，访问这个链接就可以看到加速器地址： [https://cr.console.aliyun.com/#/accelerator](https://cr.console.aliyun.com/#/accelerator)
- DaoCloud 加速器：注册 DaoCloud 账户(支持微信登录)，然后访问： [https://www.daocloud.io/mirror#accelerator-doc](https://www.daocloud.io/mirror#accelerator-doc)
当然你也可以自己搞个梯子。

## 问：如果 Docker 升级或者重启的话，那容器是不是都会被停掉然后重启啊？
在 `1.12` 以前的版本确实如此，但是从 `1.12` 开始，Docker 引擎加入了 `--live-restore` 参数，使用该参数可以避免引擎升级、重启导致容器停止服务的情况。
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

有些人服务器上线后，发现突然多了一些莫名奇妙的容器在跑。比如下面这个例子：

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
一般情况是不需要指定容器 IP 地址的。这不是虚拟主机，而是容器。其地址是供容器间通讯的，容器间则不用 IP 直接通讯，而使用容器名、服务名、网络别名。

为了保持向后兼容，`docker run` 在不指定 `--network` 时，所在的网络是 `default bridge`，在这个网络下，需要使用 `--link`参数才可以让两个容器找到对方。

这是有局限性的，因为这个时候使用的是 `/etc/hosts` 静态文件来进行的解析，比如一个主机挂了后，重新启动IP可能会改变。虽然说这种改变Docker是可能更新`/etc/hosts`文件，但是这有诸多问题，可能会因为竞争冒险导致 `/etc/hosts` 文件损毁，也可能还在运行的容器在取得 `/etc/hosts` 的解析结果后，不再去监视该文件是否变动。种种原因都可能会导致旧的主机无法通过容器名访问到新的主机。

参考官网文档：[https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/)

如果可能不要使用这种过时的方式，而是用下面说的自定义网络的方式。

而对于新的环境（`Docker 1.10`以上），应该给容器建立自定义网络，同一个自定义网络中，可以使用对方容器的容器名、服务名、网络别名来找到对方。这个时候帮助进行服务发现的是Docker 内置的DNS。所以，无论容器是否重启、更换IP，内置的DNS都能正确指定到对方的位置。
参考官网文档：[https://docs.docker.com/engine/userguide/networking/work-with-networks/#linking-containers-in-user-defined-networks](https://docs.docker.com/engine/userguide/networking/work-with-networks/#linking-containers-in-user-defined-networks)

## 问：如何修改容器的 /etc/hosts 文件？
容器内的 `/etc/hosts` 文件不应该被随意修改，如果必须添加主机名和 IP 地址映射关系，应该在 `docker run` 时使用 `--add-host` 参数，或者在 `docker-compose.yml` 中添加 `extra_hosts` 项。

不过在用之前，应该再考虑一下真的需要修改 `/etc/hosts` 么？如果只是为了容器间互相访问，应该建立自定义网络，并使用 Docker 内置的 DNS 服务。

## 问：怎么映射宿主端口？Dockerfile 中的EXPOSE和 docker run -p 有啥区别？

Docker中有两个概念，一个叫做 `EXPOSE` ，一个叫做 `PUBLISH` 。
- `EXPOSE` 是镜像/容器声明要暴露该端口，可以供其他容器使用。这种声明，在没有设定 `--icc=false`的时候，实际上只是一种标注，并不强制。也就是说，没有声明 `EXPOSE` 的端口，其它容器也可以访问。但是当强制 `--icc=false` 的时候，那么只有 `EXPOSE` 的端口，其它容器才可以访问。
- `PUBLISH` 则是通过映射宿主端口，将容器的端口公开于外界，也就是说宿主之外的机器，可以通过访问宿主IP及对应的该映射端口，访问到容器对应端口，从而使用容器服务。

`EXPOSE` 的端口可以不 `PUBLISH`，这样只有容器间可以访问，宿主之外无法访问。而 `PUBLISH` 的端口，可以不事先 `EXPOSE`，换句话说 `PUBLISH` 等于同时隐式定义了该端口要 `EXPOSE`。

`docker run` 命令中的 `-p`, `-P` 参数，以及 `docker-compose.yml` 中的  `ports` 部分，实际上均是指 `PUBLISH`。

小写 `-p` 是端口映射，格式为 `[宿主IP:]<宿主端口>:<容器端口>`，其中宿主端口和容器端口，既可以是一个数字，也可以是一个范围，比如：`1000-2000:1000-2000`。对于多宿主的机器，可以指定宿主`IP`，不指定宿主`IP`时，守护所有接口。

大写 `-P `则是自动映射，将所有定义 `EXPOSE` 的端口，随机映射到宿主的某个端口。

## 问：我要映射好几百个端口，难道要一个个   `-p` 么？
`-p` 是可以用范围的：

```bash
-p 8001-8010:8001-8010
```

## 问：为什么 `-p` 后还是无法通过映射端口访问容器里面的服务？

首先，当然是检查这个 `docker` 的容器是否启动正常： `docker ps`、`docker top <容器ID>`、`docker logs <容器ID>`、`docker exec -it <容器ID> bash`等，这是比较常用的排障的命令；如果是 `docker-compose` 也有其对应的这一组命令，所以排障很容易。

如果确保服务一切正常，甚至在容器里，可以访问到这些服务，`docker ps` 也显示出了端口映射成功，那么就需要检查防火墙了。


------------------------------------------------------------------------------------------------------------------
**未完待续**