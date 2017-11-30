title: 基于Docker容器部署Hexo：一个在github运行的个人网站
date: 2016/3/21 21:58:25
categories:
- 网站建设
tags:
- Hexo
- NexT
- Github
- Docker
---
## 关于hexo
hexo是一个基于Node.js的静态博客程序，可以方便的生成静态网页托管在github上。
优势：

- 生成静态页面快
- 支持 Markdown
- 兼容于 Windows, Mac & Linux
- 部署方便。日常使用仅需五个命令。
- 高扩展性、自订性，文件少、小，易理解

### 搭建博客准备工作 ###

**配置ssh**

- 使用hexo博客必须配置SSH，打开git bash，输入

```
cd ~/.ssh
```
如果提示：No such file or directory 说明未配置SSH。

- 本地生成密钥对

```
ssh-keygen -t rsa -C "你的邮件地址"
```
注意命令中的大小写不要搞混。按提示指定保存文件夹，不设置密码。

- 添加公钥到Github

找到公钥文件（默认为id_rsa.pub），用记事本打开，全选并复制。
登录Github，右上角 头像 -> Settings —> SSH keys —> Add SSH key。把公钥粘贴到key中，填好title并点击 Add key。

- 测试连接情况git bash中输入命令

```
ssh -T git@github.com ，选yes，等待片刻可看到成功提示。
```
- 修改本地的ssh remote url，不用https协议，改用git协议
Github仓库中获取ssh协议相应的url
本地仓库执行命令

```
git remote set-url origin SSH对应的url，
```
配置完后可用

```
git remote -v查看结果
```
这样

```
git push
```
或

```
hexo d
```
时不再需要输入账号密码。

## 关于Docker

Docker是一个基于轻量级虚拟化技术的容器，整个项目基于Go语言开发，并采用了Apache 2.0协议。Docker可以将我们的应用程序打包封装到一个容器中，该容器包含了应用程序的代码、运行环境、依赖库、配置文件等必需的资源，通过容器就可以实现方便快速并且与平台解耦的自动化部署方式，无论你部署时的环境如何，容器中的应用程序都会运行在同一种环境下。

### 利用Dockerfile构建一个包含Hexo环境的镜像 ###

<!--more-->

编写启动脚本

```
cat << EOF > start.sh

if [ "\$1" = 's' ] || [ "\$1" = 'server' ]; then
    set -- /usr/bin/hexo s -p 80
fi

if [ "\$1" = 'd' ] || [ "\$1" = 'deploy' ]; then
    set -- /usr/bin/hexo cl && /usr/bin/hexo d -g
fi
exec "\$@"

EOF

chmod a+x start.sh
```
生成Dockerfile文件
```
cat << EOF > Dockerfile

FROM centos:7 

ENV TZ=Asia/Shanghai LC_ALL=en_US.UTF-8 

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

RUN curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo \\
&& for packages in wget net-tools unzip vim-enhanced git;do yum -y install \$packages; done \\
&& yum clean all

RUN curl --silent --location https://rpm.nodesource.com/setup_8.x | bash - \\
&& yum -y install nodejs 

WORKDIR /data/ok188.net

RUN npm install hexo-cli -g \\
&& hexo init . \\
&& npm install \\
&& npm install hexo-generator-sitemap --save \\
&& npm install hexo-generator-feed --save \\
&& npm install hexo-deployer-git --save \\
&& npm install hexo-generator-searchdb --save

RUN git clone https://github.com/iissnan/hexo-theme-next themes/next \\
&& git config --global user.name "your github name" \\
&& git config --global user.email "your email"


COPY start.sh /data/ok188.net/start.sh

EOF
```
备注:这里面用了dumb-init来初始化容器系统，避免运行中的容器里产生僵死进程。

创建Dockerfile后可通过docker build命令制作成镜像。
```
docker build -t hexo-docker-centos7:v1.0 .
```
### 创建和运行容器
```
docker run -d \
-v /data/site/source/_posts:/data/ok188.net/source/_posts \
--init=true \
--name hexo-s \
--net=host \
hexo-docker-centos7:v1.0 \
sh  /data/ok188.net/start.sh s
```
```
docker run -d \
-v /root/.ssh:/root/.ssh \
-v /data/site/source/_posts:/data/ok188.net/source/_posts \
--init=true \
--name hexo-s \
--net=host \
hexo-docker-centos7:v1.0 \
sh  /data/ok188.net/start.sh d
```
### 域名解析 ####

在网站source目录下创建一个CNAME文件，填写你要解析的域名到CNAME文件中。
前往你的DNS服务商新建一个CNAME解析。


