title: Docker容器资源限制测试
date: 2017/12/3 18:10:25
categories:
- Docker
tags:
- Docker_Container_Limit
- Docker
---
## Docker容器资源限制测试

Docker的运行时容器的本质是进程。在Linux中，通过Namespace进行资源隔离，Cgroups进行资源限制。

### 一、Docker容器Cpu资源限制测试

####  容器资源CPU限制设置测试

默认所有的容器对于 CPU 的利用占比都是一样的，-c 或者 --cpu-shares 可以设置 CPU 利用率权重，默认为 1024，可以设置权重为 2 或者更高(单个 CPU 为 1024，两个为 2048，以此类推)。如果设置选项为 0，则系统会忽略该选项并且使用默认值 1024。通过以上设置，只会在 CPU 密集(繁忙)型运行进程时体现出来。当一个 container 空闲时，其它容器都是可以占用 CPU 的。cpu-shares 值为一个相对值，实际 CPU 利用率则取决于系统上运行容器的数量。

假如一个 1core 的主机运行 3 个 container，其中一个 cpu-shares 设置为 1024，而其它 cpu-shares 被设置成 512。当 3 个容器中的进程尝试使用 100% CPU 的时候「尝试使用 100% CPU 很重要，此时才可以体现设置值」，则设置 1024 的容器会占用 50% 的 CPU 时间。如果又添加一个 cpu-shares 为 1024 的 container，那么两个设置为 1024 的容器 CPU 利用占比为 33%，而另外两个则为 16.5%。简单的算法就是，所有设置的值相加，每个容器的占比就是 CPU 的利用率，如果只有一个容器，那么此时它无论设置 512 或者 1024，CPU 利用率都将是 100%。当然，如果主机是 3core，运行 3 个容器，两个 cpu-shares 设置为 512，一个设置为 1024，则此时每个 container 都能占用其中一个 CPU 为 100%。
<!--more-->
####  1.1、通过参数--cpu-shares分配cpu使用权重

现在运行两个测试 container，一个权重设置为 2，一个权重设置 4，启动命令如下：
```
[root@ok188 ~]# docker run -it -d --cpu-shares 2 --name 2_cpu centos:7 /bin/bash
f3f125f7455974be77e58c0864d045b3b56ae2d007bd9095c47faca50893547c
[root@ok188 ~]# docker run -it -d --cpu-shares 4 --name 4_cpu centos:7 /bin/bash
5e623b55a22ef6d1e41e5978dd1c5d05d743b3a91498697db3b4b9c493f03f8b
```
通过压测工具如Stress进行压测，Stress使用实例:
- 产生13个cpu进程4个io进程1分钟后停止运行
```
[root@ok188 ~]# stress -c 13 -i 4 --verbose --timeout 1m
```
- 测试硬盘，通过mkstemp()生成800K大小的文件写入硬盘，对CPU、内存的使用要求很低
```
[root@ok188 ~]# stress -d 1 --hdd-noclean --hdd-bytes 800k
```
- 产生13个进程，每个进程都反复不停的计算由rand ()产生随机数的平方根
```
[root@ok188 ~]# stress -c 13
```
- 向磁盘中写入固定大小的文件，这个文件通过调用mkstemp()产生并保存在当前目录下，默认是文件产生后就被执行unlink(清除)操作，但是可以使用--hdd-bytes选项将产生的文件全部保存在当前目录下，这会将你的磁盘空间逐步耗尽
```
# 生成小文件
[root@ok188 ~]# stress -d 1 --hdd-noclean --hdd-bytes 13
# 生成大文件
[root@ok188 ~]# stress -d 1 --hdd-noclean --hdd-bytes 3G
```

对两个容器同时进行 CPU压测，在宿主机中查看两个容器CPU占用情况
```
[root@ok188 ~]# docker stats 2_cpu
CONTAINER      CPU %         MEM USAGE / LIMIT     MEM %      NET I/O         BLOCK I/O                 
2_cpu          33.31%        24.34MiB / 1.932GiB   1.23%     18.2MB / 322kB   49.1MB / 27.7MB  
```
```   
[root@ok188 ~]# docker stats 4_cpu
CONTAINER     CPU %         MEM USAGE / LIMIT     MEM %      NET I/O          BLOCK I/O                
4_cpu         65.96%        38.82MiB / 1.932GiB   1.96%     18.3MB / 451kB    8.19kB / 27.7MB
```
观察以上结果发现容器名为4_cpu权重比2_cpu大2倍，所以4_cpu可使用的cpu更多。
停止压测名为4_cpu的容器, 在宿主机中查看两个容器CPU占用情况
```
[root@ok188 ~]# docker stats 2_cpu
CONTAINER      CPU %         MEM USAGE / LIMIT     MEM %      NET I/O         BLOCK I/O                 
2_cpu          99.50%        24.34MiB / 1.932GiB   1.23%     18.2MB / 322kB   49.1MB / 27.7MB
```
```
[root@ok188 ~]# docker stats 4_cpu
CONTAINER     CPU %         MEM USAGE / LIMIT     MEM %      NET I/O          BLOCK I/O                
4_cpu         0.00%         37.5MiB / 1.932GiB   1.90%     18.3MB / 452kB    8.19kB / 27.7MB
```
结论参数cpu-shares 只是设置cpu使用权重，只会在 CPU 密集(繁忙)型运行进程时体现出来。当一个 container 空闲时，其它容器都是可以占用 CPU 的。
备注：物理主机为多核的情况下，显示的cpu使用比例有差异，本人测试主机为单核。
#### 1.2、通过 --cpu-period & --cpu-quota 限制容器的 CPU 使用上限

默认的 CPU CFS「Completely Fair Scheduler」period 是 100ms。我们可以通过 --cpu-period 值限制容器的 CPU 使用。一般 --cpu-period 配合 --cpu-quota 一起使用。

为啥把这两个参数放一起呢？因为这两个参数是相互配合的，--cpu-period和--cpu-quota的这种配置叫Ceiling Enforcement Tunable Parameters，--cpu-shares的这种配置叫Relative Shares Tunable Parameters。--cpu-period是用来指定容器对CPU的使用要在多长时间内做一次重新分配，而--cpu-quota是用来指定在这个周期内，最多可以有多少时间用来跑这个容器。跟--cpu-shares不同的是这种配置是指定一个绝对值，而且没有弹性在里面，容器对CPU资源的使用绝对不会超过配置的值。

设置 cpu-period 为 100000，cpu-quota 为 50000，表示最多可以使用 cpu到50%。
```
[root@nsj-13-58 ~]# docker run -it --cpu-period=100000 --cpu-quota=50000 --name 0.5_p_cpu centos:7 /bin/bash
```   
压测容器，并查看CPU占用情况：

```
[root@ok188 ~]# docker stats 0.5_p_cpu
CONTAINER       CPU %     MEM USAGE / LIMIT     MEM %         NET I/O             BLOCK I/O
0.5_p_cpu       50.33%    22.12MiB / 1.932GiB   1.12%         18.3MB / 348kB      524kB / 27.7MB      
```

通过以上测试可以得知，--cpu-period 结合 --cpu-quota 配置是固定的，无论宿主机系统 CPU 是闲还是繁忙，如上配置，容器最多只能使用 CPU 到 50%。

###  二、Docker容器Memory资源限制测试
默认启动一个container，对于容器的内存是没有任何限制的。
#### 2.1、默认启动一个 container，对于容器的内存是没有任何限制的。 ####
```
[root@ok188 ~]# docker run -it -d --name no_limit_memory centos:7 /bin/bash
[root@ok188 ~]# docker stats no_limit_memory
CONTAINER           CPU %      MEM USAGE / LIMIT   MEM %      NET I/O             BLOCK I/O
no_limit_memory     0.00%      932KiB / 1.932GiB   0.05%      586B / 0B           8.19kB / 0B         
[root@ok188 ~]# free -g
              total        used        free      shared  buff/cache   available
Mem:              1           0           0           0           1           1
Swap:             1           0           1
```
MEM  LIMIT显示的是容器宿主机的内存大小，Mem+Swap的总大小
#### 2.2、通过 -m 参数限制内存大小 ####

设置-m值为500Mb，表示容器程序使用内存受限。按照官方文档的理解，如果指定 -m 内存限制时不添加 --memory-swap 选项，则表示容器中程序可以使用 500m内存和500m swap 内存。那么容器里程序可以跑到500m*2=1g后才会被oom给杀死。

```
[root@ok188 ~]# docker run -it -d -m 500m --name limit_memory_1 centos:7 /bin/bash     
21eb95cffa45972603cb0e67b7ee0724d019cd7182fa5668bf07665ddf4f83cc
[root@ok188 ~]# docker stats limit_memory_1
CONTAINER           CPU %       MEM USAGE / LIMIT   MEM %      NET I/O             BLOCK I/O
limit_memory_1      0.00%       916KiB / 500MiB       0.09%      586B / 0B           0B / 0B             
```

在libcontainer源码里
```
memory.memsw.limit_in_bytes
```
值是被设置成我们指定的内存参数的两倍。

其中代码如下：

    // By default, MemorySwap is set to twice the size of RAM.
    // If you want to omit MemorySwap, set it to `-1'.
    if d.c.MemorySwap != -1 {
    if err := writeFile(dir, "memory.memsw.limit_in_bytes", strconv.FormatInt(d.c.Memory*2, 10)); err != nil {
    return err
    }

使用压测工具进行压测，当压测值是 memory + swap之和上限时，则容器中的进程会被直接 OOM kill。


#### 2.3参数--memory-swappiness=0 表示禁用容器 swap 功能。
```
[root@ok188 ~]# docker run -it -d -m 500m --memory-swappiness=0 --name limit_memory_noswap_1 centos:7 /bin/bash
```

使用压测工具进行压测，当压测值是 1G ，则容器中的进程会被直接 OOM kill。查看容器内系统日志：


#### 2.4指定限制内存大小并且设置 memory-swap 值为 -1 ####

表示容器程序使用内存受限，而 swap 空间使用不受限制（宿主 swap 支持使用多少则容器即可使用多少。如果 --memory-swap 设置小于 --memory 则设置不生效，使用默认设置）。--memory-swap -1
```
[root@ok188 ~]# docker run -it -d -m 500m --memory-swap -1  --name limit_memory_2 centos:7 /bin/bash
```
#### 2.5指定限制内存大小并且设置 memory-swap 值 ####

指定限制内存大小500Mb并且设置 memory-swap 值400Mb当压测值是900Mb时，则容器中的进程会被直接 OOM kill。
```
[root@ok188 ~]# docker run -it -d -m 500m --memory-swap 400m  --name limit_memory_3 centos:7 /bin/bash
```
备注：实际生产环境不推荐使用swap功能，建议直接禁用swap功能。


#### 2.6 参数--oom-kill-disable ，加上之后则达到限制内存之后也不会被 kill ####

正常情况不添加 --oom-kill-disable 容器程序内存使用超过限制后则会直接 OOM kill，加上之后则达到限制内存之后也不会被 kill。
```
[root@ok188 ~]# docker run -it -d -m 500m --oom-kill-disable --name limit_memory_4 centos:7 /bin/bash
```
==注意如果是以下的这种没有对容器作任何资源限制的情况，添加 --oom-kill-disable 选项就比较危险了：==

```
[root@ok188 ~]# docker run -it -d --oom-kill-disable --name limit_memory_5 centos:7 /bin/bash
```

因为此时容器内存没有限制，使用的上限值是物理内存的上限值。 并且不会被 oom kill，此时系统则会 kill 系统进程用于释放内存。

### 后记：目前 Docker 支持的资源限制选项
Option | Description
---|---
-m, --memory="" | Memory limit (format: <number>[<unit>]). Number is a positive integer. Unit can be one of b, k, m, or g. Minimum is 4M.  
--memory-swap="" | Total memory limit (memory + swap, format: <number>[<unit>]). Number is a positive integer. Unit can be one of b, k, m, or g.
--memory-reservation="" | Memory soft limit (format: <number>[<unit>]). Number is a positive integer. Unit can be one of b, k, m, or g.
--kernel-memory="" | Kernel memory limit (format: <number>[<unit>]). Number is a positive integer. Unit can be one of b, k, m, or g. Minimum is 4M.
-c, --cpu-shares=0 | CPU shares (relative weight)
--cpu-period=0 | Limit the CPU CFS (Completely Fair Scheduler) period
--cpuset-cpus="" | CPUs in which to allow execution (0-3, 0,1)
--cpuset-mems="" | Memory nodes (MEMs) in which to allow execution (0-3, 0,1). Only effective on NUMA systems.
--cpu-quota=0 | Limit the CPU CFS (Completely Fair Scheduler) quota
--blkio-weight=0 | Block IO weight (relative weight) accepts a weight value between 10 and 1000.
--blkio-weight-device="" | Block IO weight (relative device weight, format: DEVICE_NAME:WEIGHT)
--device-read-bps="" | Limit read rate from a device (format: <device-path>:<number>[<unit>]). Number is a positive integer. Unit can be one of kb, mb, or gb.
--device-write-bps="" | Limit write rate to a device (format: <device-path>:<number>[<unit>]). Number is a positive integer. Unit can be one of kb, mb, or gb.
--device-read-iops="" | Limit read rate (IO per second) from a device (format: <device-path>:<number>). Number is a positive integer.
--device-write-iops="" | Limit write rate (IO per second) to a device (format: <device-path>:<number>). Number is a positive integer.
--oom-kill-disable=false | Whether to disable OOM Killer for the container or not.
--memory-swappiness="" | Tune a container’s memory swappiness behavior. Accepts an integer between 0 and 100.
--shm-size="" | Size of /dev/shm. The format is <number><unit>. number must be greater than 0. Unit is optional and can be b (bytes), k (kilobytes), m (megabytes), or g (gigabytes). If you omit the unit, the system uses bytes. If you omit the size entirely, the system uses 64m.

##### 附Docker容器限制源码路径：

[github.com/opencontainers/runc/libcontainer/cgroups/fs](https://github.com/opencontainers/runc/tree/master/libcontainer/cgroups/fs)
