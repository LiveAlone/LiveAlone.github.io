---
title: 虚拟机容器基础监听命令(1)
tags:
  - Monitor
  - Linux
categories:
  - 技术
  - 监控
  - Linux
date: 2020-09-07 17:38:23
---


#### uptime CPU

load average 系统负荷平均统计，单核 CPU 1.0, 系统刚好执行完成所有任务。 通过 ```cat /proc/cpuinfo, grep -c 'model name' /proc/cpuinfo``` 统计CPU 数量， 一般线上服务 0.7 负载相对有问题了， 需要排查问题原因

```shell
[deploy@sns-jarvis-staging01 ~]$ uptime
19:07:32 (当前时间) up 671 days, 19:46, (运行时间)  2 users,(用户数量)  load average: 49.57, 45.00, 41.24 (平均负载)
```

#### vmstat CPU

务器的CPU使用率，内存使用，虚拟内存交换情况,IO读写情况。

```shell
[deploy@sns-jarvis-staging02 ~/cron-mqworker_yqj1]$ vmstat 2
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
53  0      0 1829688 546056 5312848    0    0    22   103    0    0 27 12 60  0  0
24  0      0 1764004 546056 5312872    0    0     0   196 51070 317194 31 14 55  0  0
71  0      0 1748140 546056 5312880    0    0     0    74 48609 299565 41 12 47  0  0
```

- 进程相关
  - r 运行进程数量, 核对CPU 数量对应
  - b 阻塞进程数量
- 内存相关
  - swpd 系统内存不够时候， 会置换出内存模块，进入磁盘中， > 0 时候， 说明内存不足了。 通常线上服务是关闭内存置换的。
  - free 这里空闲内存大小， **注意 不是真实的可用内存，Linux 预先占用所有内存等待比例释放内存** 关注度不高
  - buffer 缓冲区， 进程读写文件时候，预先写入缓冲区， 什么时候刷入磁盘， 等待操作系统的控制。可以 sync 强制刷盘操作。保证数据录入正确。
  - cache 为了提高数据读取性能，防止重复读取磁盘造成，程序运行速度下降。
- 磁盘IO相关
  - si so 磁盘中读取，写入虚拟内存的数据大小，大于 0, 同样内存泄露
  - bi bo 所有的块设备， in, out 数量，接近0，不然IO 过于频繁。
  - in: cpu 中断次数， cs: cpu 上线文切换次数
  - us: 用户态CPU 时间比例， sy 系统态时间比率， id cpu 空闲比率。 us + sy + id = 100
  - wa 系统等待 IO 时间，> 0, IO 影响系统等待
  - st 容器化系统偷取时间 stolen time

#### mpstat CPU

Multiprocessor Statistics 缩写，监控多线程 CPU 状态。 命令格式 ```mpstat [-P {cpu|ALL}] [internal [count]]```

```shell
[deploy@sns-jarvis-staging02 ~]$ mpstat -P ALL 2 10
Linux 3.10.0-514.10.2.el7.x86_64 (sns-jarvis-staging02) 	09/08/2020 	_x86_64_	(8 CPU)

01:10:14 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
01:10:16 AM  all   46.17    0.00   25.54    0.13    0.00    0.19    0.00    0.00    0.00   27.97
01:10:16 AM    0   56.85    0.00   23.35    1.02    0.00    0.00    0.00    0.00    0.00   18.78
01:10:16 AM    1   50.00    0.00   23.98    0.00    0.00    0.00    0.00    0.00    0.00   26.02
01:10:16 AM    2   43.01    0.00   27.46    0.00    0.00    0.00    0.00    0.00    0.00   29.53
01:10:16 AM    3   39.38    0.00   27.98    0.00    0.00    0.00    0.00    0.00    0.00   32.64
01:10:16 AM    4   30.61    0.00   15.82    0.51    0.00    0.51    0.00    0.00    0.00   52.55
01:10:16 AM    5   29.02    0.00   34.20    0.00    0.00    1.04    0.00    0.00    0.00   35.75
01:10:16 AM    6   60.61    0.00   20.71    0.00    0.00    0.00    0.00    0.00    0.00   18.69
01:10:16 AM    7   58.00    0.00   31.50    0.00    0.00    0.50    0.00    0.00    0.00   10.00

```

返回结果Linux 内核版本， cpu 核数， 和每个CPU 的详细状态信息。 参数含义

- %user      在internal时间段里，用户态的CPU时间(%)，不包含nice值为负进程  (usr/total)*100
- %nice      在internal时间段里，nice值为负进程的CPU时间(%)   (nice/total)*100
- %sys       在internal时间段里，内核时间(%)       (system/total)*100
- %iowait    在internal时间段里，硬盘IO等待时间(%) (iowait/total)*100
- %irq       在internal时间段里，硬中断时间(%)     (irq/total)*100
- %soft      在internal时间段里，软中断时间(%)     (softirq/total)*100
- %idle      在internal时间段里，CPU除去等待磁盘IO操作外的因为任何原因而空闲的时间闲置时间(%) (idle/total)*100

#### free MEM

内存监控 free -m 监控内存使用情况， 通过M 单位显示内存结构， linux 内存占用思想 used = used - buffer - cached , free = free + buffer = cache 真正内存大小。

```shell
[deploy@sns-jarvis-staging02 ~]$ free -m
              total        used        free      shared  buff/cache   available
Mem:          32013       24718        1618          13        5676        6733
Swap:             0           0           0
```

#### iostat IO

iostat方便查看CPU、网卡、tty设备、磁盘、CD-ROM 等等设备的活动情况, 负载信息。

```shell
[deploy@sns-jarvis-staging02 ~]$ iostat
Linux 3.10.0-514.10.2.el7.x86_64 (sns-jarvis-staging02) 09/08/2020 _x86_64_(8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          27.39    0.00   12.20    0.12    0.00   60.29

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
vda              30.91       105.07       483.29 8105871565 37285428448
vdb               6.84        65.00       318.41 5014628833 24564686976

[deploy@sns-jarvis-staging02 ~]$ iostat -x
Linux 3.10.0-514.10.2.el7.x86_64 (sns-jarvis-staging02) 09/08/2020 _x86_64_(8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          27.39    0.00   12.20    0.12    0.00   60.29

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.02    45.59    3.34   27.57   105.07   483.29    38.07     0.01    0.32    5.34    1.73   0.32   0.98
vdb               0.00     4.33    2.33    4.50    65.00   318.40   112.18     0.02    2.73    3.07    2.55   0.46   0.31
```

命令参数定义
-C 显示CPU使用情况
-d 显示磁盘使用情况
-k 以 KB 为单位显示
-m 以 M 为单位显示
-N 显示磁盘阵列(LVM) 信息
-n 显示NFS 使用情况
-p[磁盘] 显示磁盘和分区的情况
-t 显示终端和CPU的信息
-x 显示详细信息
-V 显示版本信息

avg-cpu 这个上面定义相同, 不再说明，网上load 下来的图片 ![io](/images/20200908/io.png)

- rrqm/s， wrqm/s 每秒进行 merge 的读，写操作数量， 就是 rmerge/s，wmerge/s
  - 说明一下，app 多进程通过 内核调用，写入虚拟文件系统数据。 操作系统通过块设备，bio阻塞写入 或者 IO调度器写入队列，调用驱动写入磁盘中。 这个时候会对相邻块写入进行合并操作。
- r/s w/s 每秒完成 IO 的读写次数
- rsec/s wsec/s 每秒读写扇区数
- rkB/s wkB/s 每秒读写写入 KB 数量
- avgrq-sz：平均每次设备IO操作的数据大小（扇区）；平均单次IO大小。
- avgqu-sz：平均IO队列长度。
- await：从请求磁盘操作到系统完成处理，每次请求的平均消耗时间，包括请求队列等待时间；平均IO响应时间（毫秒）。
  - r_await， w_await 读写等待时间
- svctm：平均每次设备IO操作的服务时间（毫秒）
  - 通过图片，操作系统等地时间， 一个是 块设备处理时间
- %util：一秒中有百分之多少的时间用于 I/O 操作，即被io消耗的cpu百分比。

正常情况下svctm应该是小于await值的，而svctm的大小和磁盘性能有关，CPU、内存的负荷也会对svctm值造成影响，过多的请求也会间接的导致svctm值的增加。await值的大小一般取决于svctm的值和IO队列的长度以及IO请求模式，如果scvtm比较接近await，说明IO几乎没有等待时间；如果await远大于svctm，说明IO请求队列太长，IO响应太慢，则需要进行必要优化。
如果%util接近100%，说明产生的IO请求太多，IO系统已经满负荷，该磁盘可能存在瓶颈。
队列长度(avgqu-sz)也可作为衡量系统 I/O 负荷的指标，但由于 avgqu-sz 是按照单位时间的平均值，所以不能反映瞬间的 I/O 泛洪，如果avgqu-sz比较大，则说明有大量IO在等待。

#### top htop

统计CPU 内存的使用率，监控状态是进程， 通过 ```top -Hp ${processId}``` 监控运行进程状态。 htop 对应升级版本，进程角度监控运行状态。

```shell
[deploy@sns-jarvis-staging02 ~]$ top
top - 00:26:03 up 893 days, 11:25,  1 user,  load average: 22.49, 22.70, 25.52  （当前时间, 系统运行时间，用户数量，CPU 负载）
Tasks: 343 total,   2 running, 341 sleeping,   0 stopped,   0 zombie  （process 不同状态进程数量）
%Cpu(s): 43.5 us, 14.6 sy,  0.0 ni, 41.5 id,  0.1 wa,  0.0 hi,  0.3 si,  0.0 st （用户时间， 系统时间， 改变优先级占比， 空闲占比， IO等待，软硬中断）
KiB Mem : 32781488 total,  3090220 free, 26196908 used,  3494360 buff/cache （缓存Cache 相关）
KiB Swap:        0 total,        0 free,        0 used.  6007408 avail Mem  （swap 系统硬盘缓存相关）

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 5068 deploy    20   0 3535500 396468  12860 S 222.9  1.2   0:08.84 java
15860 deploy    20   0 6428388 727608      0 S  31.9  2.2  23159:40 java
20794 deploy    20   0 7056876 1.096g  14684 S  13.3  3.5 120:07.32 java
 3290 deploy    20   0 4609100 559288   3892 S  12.3  1.7   1440:04 java
```

- M 按照内存排序，  m 显示内存信息，
- P 按照 CPU 使用排序，
- T cpu 占用累计时间排序
- s 刷新时间间隔，  second  默认单位
- K  删除 对应进程。
- H thread 线程模式， 监控线程执行状态

#### dmesg

系统日志监控方式，对于挂了进程， 失败进程监控运行日志。
