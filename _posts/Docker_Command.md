title: Docker常用命令，不定期更新补充。
date: 2017/12/2 21:58:25
categories:
- Docker
tags:
- Command
- Docker
---
#### 清理命令
```
docker system prune
docker image prune --force --all
```
#### 查询Docker Engine日志
```
journalctl -r -u docker
```
#### 获取镜像
```
docker pull ubuntu:12.04
```
<!--more-->
#### 列出镜像
```
docker images

docker images -a
```
#### 导出镜像
```
docker save -o ubuntu_12.04.tar ubuntu:12.04
```
#### 导入镜像
```
docker load --input ubuntu_12.04.tar
```
#### 删除镜像
```
docker rmi 57bca5139a13
```
#### 普通启动
```
docker run -t -i ubuntu:12.04 /bin/bash
```
#### 后台运行
```
docker run -d ubuntu:12.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
```
#### 启动、停止、重启容器
```
docker start fe1dce92b833 #启动容器，最后一位是容器ID
docker stop fe1dce92b833  #停止容器
docker ps -a #查看状态
docker restart fe1dce92b833 #重启容器
```
#### 进入容器后台
```
docker exec -it  fe1dce92b833 /bin/bash
```
#### 删除容器
```
docker rm 0f2eaa78f1fc     #删除ID为0f2eaa78f1fc的容器
docker rm -f fe1dce92b833  #强制删除运行的容器
```
#### 上传、下载镜像
```
docker tag 57bca5139a13  127.0.0.1:5000/test/ubuntu-reg-test  #使用tag参数先做个标记
docker push 127.0.0.1:5000/test/ubuntu-reg-test  #推送私有仓库
docker pull 127.0.0.1:5000/test/ubuntu-reg-test #下载私有镜像
```
#### 删除所有停止的容器
```
docker rm $( docker ps -q -f status=exited)
```
#### 删除所有none容器
```
docker rmi $( docker images -q -f dangling=true)
```
#### 导入导出
```
docker save -o 17usoft.tomcat7.tar docker.17usoft.com:5000/prod/tomcat7
docker load --input 17usoft.tomcat7.tar
```
#### 拷贝容器文件
```
docker cp f52560aeaf1b:/var/jenkins_home/ . #从容器中拷贝文件到物理机当前目录
docker cp . f52560aeaf1b:/var/jenkins_home/ #将物理机当前目录文件拷贝到容器中
```
#### 提交容器到镜像
```
docker commit -m "Alter config of tomcat" -a "Kaiden"  e7c9721d0f1b docker.17usoft.com/itlsa/tomcat7
docker push docker.17usoft.com/itlsa/tomcat7
```
#### 查看容器资源使用情况
```
docker stats hotel
```
