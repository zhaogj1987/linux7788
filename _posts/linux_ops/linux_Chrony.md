title: Linux系统下校时服务Chrony使用
date: 2017/11/29 11:58:25
categories:
- Linux系统运维
tags:
- Chrony

---
Linux系统下校时服务Chrony使用
================

Chrony 应用本身已经有几年了，其是是网络时间协议的 (NTP) 的另一种实现。一直以来众多发行版里标配的都是ntpd对时服务，自rhel7/centos7 起，Chrony做为了发行版里的标配服务，不过老的ntpd服务依旧在rhel7/centos7里可以找到 。Chrony可以同时做为ntp服务的客户端和服务端。默认安装完后有两个程序chronyd和chronyc 。chronyd是一个在系统后台运行的守护进程，chronyc是用来监控chronyd性能和配置其参数程序
## 安装和启用
```
[root@linux7788.com ~]# yum install -y chrony
[root@linux7788.com ~]# cat << EOF > /etc/chrony.conf
# 使用上层的internet ntp服务器
server time1.aliyun.com iburst
server time2.aliyun.com iburst
server time3.aliyun.com iburst
server time4.aliyun.com iburst
server time5.aliyun.com iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
noclientlog
logchange 0.5
logdir /var/log/chrony
EOF

[root@linux7788.com ~]# systemctl stop ntpd
[root@linux7788.com ~]# systemctl disable ntpd
[root@linux7788.com ~]# systemctl enable chronyd.service
[root@linux7788.com ~]# systemctl start chronyd.service
```
#### 配置文件解析
如果本局域网内有对时服务开启的话，通过将上面的几条serer记录删除，增加指定局域网内的对时服务器并restart chrony服务即可。其中主要的配置参数有如下几个：
<!--more-->
- server - 该参数可以多次用于添加时钟服务器，必须以"server "格式使用。一般而言，你想添加多少服务器，就可以添加多少服务器；
- stratumweight - stratumweight指令设置当chronyd从可用源中选择同步源时，每个层应该添加多少距离到同步距离。默认情况下，CentOS中设置为0，让chronyd在选择源时忽略源的层级；
- driftfile - chronyd程序的主要行为之一，就是根据实际时间计算出计算机增减时间的比率，将它记录到一个文件中是最合理的，它会在重启后为系统时钟作出补偿，甚至可能的话，会从时钟服务器获得较好的估值；
- rtcsync - rtcsync指令将启用一个内核模式，在该模式中，系统时间每11分钟会拷贝到实时时钟（RTC）；
- allow / deny - 这里你可以指定一台主机、子网，或者网络以允许或拒绝NTP连接到扮演时钟服务器的机器；
- cmdallow / cmddeny - 跟上面相类似，只是你可以指定哪个IP地址或哪台主机可以通过chronyd使用控制命令；
- bindcmdaddress - 该指令允许你限制chronyd监听哪个网络接口的命令包（由chronyc执行）。该指令通过cmddeny机制提供了一个除上述限制以外可用的额外的访问控制等级。
- makestep - 通常，chronyd将根据需求通过减慢或加速时钟，使得系统逐步纠正所有时间偏差。在某些特定情况下，系统时钟可能会漂移过快，导致该调整过程消耗很长的时间来纠正系统时钟。该指令强制chronyd在调整期大于某个阀值时步进调整系统时钟，但只有在因为chronyd启动时间超过指定限制（可使用负值来禁用限制），没有更多时钟更新时才生效。

## 查看同步状态
检查ntp源服务器状态：
```
[root@linux7788.com ~]# chronyc sourcestats
210 Number of sources = 4
Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
==============================================================================
time6.aliyun.com           14  10   81m     -0.017      0.182  +1290us   261us
time4.aliyun.com           11   5   77m     -0.122      0.825   +513us  1029us
time5.aliyun.com           12   7   73m     +0.052      0.262  -1194us   275us
```
检查ntp详细同步状态：
```
[root@linux7788.com ~]#  chronyc sources -v
210 Number of sources = 4

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
| /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^+ time6.aliyun.com              2  10   377  1021  +1438us[+1345us] +/-   25ms
^+ time4.aliyun.com              2   9   377     2  +2047us[+2047us] +/-   35ms
^* time5.aliyun.com              2   9   377   506  -1712us[-1851us] +/-   18ms
```
## 使用chronyc
可以通过运行chronyc命令来修改设置，命令如下：

- accheck - 检查NTP访问是否对特定主机可用

- activity - 该命令会显示有多少NTP源在线/离线

- add server - 手动添加一台新的NTP服务器。

- clients - 在客户端报告已访问到服务器

- delete - 手动移除NTP服务器或对等服务器

- settime - 手动设置守护进程时间

- tracking - 显示系统时间信息

输入help命令可以查看更多chronyc的交互命令。
```
[root@linux7788.com ~]# chronyc
chrony version 2.1.1
Copyright (C) 1997-2003, 2007, 2009-2015 Richard P. Curnow and others
chrony comes with ABSOLUTELY NO WARRANTY.  This is free software, and
you are welcome to redistribute it under certain conditions.  See the
GNU General Public License version 2 for details.

chronyc> activity
200 OK
4 sources online
0 sources offline
0 sources doing burst (return to online)
0 sources doing burst (return to offline)
1 sources with unknown address
```
## chrony的优势
Chrony 的优势包括：

- 更快的同步只需要数分钟而非数小时时间，从而最大程度减少了时间和频率误差，这对于并非全天 24 小时运行的台式计算机或系统而言非常有用。
- 能够更好地响应时钟频率的快速变化，这对于具备不稳定时钟的虚拟机或导致时钟频率发生变化的节能技术而言非常有用。
- 在初始同步后，它不会停止时钟，以防对需要系统时间保持单调的应用程序造成影响。
- 在应对临时非对称延迟时（例如，在大规模下载造成链接饱和时）提供了更好的稳定性。
- 无需对服务器进行定期轮询，因此具备间歇性网络连接的系统仍然可以快速同步时钟。

## 官方文档
[https://chrony.tuxfamily.org/documentation.html](https://chrony.tuxfamily.org/documentation.html)
