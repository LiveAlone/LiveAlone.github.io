---
title: 虚拟机容器基础监听命令(2)
tags:
  - Monitor
  - Linux
categories:
  - 技术
  - 监控
  - Linux
date: 2020-09-09 00:52:52
---


#### ss

- ```ss -antp > $DUMP_DIR/ss.dump 2>&1``` Dump 所有的tpc 链接方式
- ```ss -s``` summary  显示 socket 统计信息， 链接数量， 防止端口占用完。
  - ss -l 显示本地打开的所有端口
  - ss -pl 显示每个进程具体打开的socket
  - ss -t -a 显示所有tcp socket
  - ss -u -a 显示所有的UDP Socekt
  - ss -o state established '( dport = :smtp or sport = :smtp )' 显示所有已建立的SMTP连接
  - ss -o state established '( dport = :http or sport = :http )' 显示所有已建立的HTTP连接
  - ss -x src /tmp/.X11-unix/* 找出所有连接X服务器的进程
  - ss -s 列出当前socket详细信息

监控端口状态通过网络图片说明，![io2.png](/images/20200908/io2.png)

主动连接端可能的状态有： CLOSED SYN_SEND ESTABLISHED
主动关闭端可能的状态有： FIN_WAIT_1 FIN_WAIT_2 TIME_WAIT
被动连接端可能的状态有： LISTEN SYN_RECV ESTABLISHED
被动关闭端可能的状态有： CLOSE_WAIT LAST_ACK CLOSED

一般情况统计 io wait 等待状态进入关注队列中， 防止端口占用完毕了。

#### netstat

参数含义

- a (all)显示所有选项，默认不显示LISTEN相关
- t (tcp)仅显示tcp相关选项
- u (udp)仅显示udp相关选项
- n 拒绝显示别名，能显示数字的全部转化成数字。
- l 仅列出有在 Listen (监听) 的服務状态
- p 显示建立相关链接的程序名
- r 显示路由信息，路由表
- e 显示扩展信息，例如uid等
- s 按各个协议进行统计
- c 每隔一个固定时间，执行该netstat命令。

一般使用方式 通过grep 过滤不同端口的监控状态信息。-r 获取路由表配置。netstat -lu -lt -lx 不同监听状态

#### sar

系统监控瓶颈分析, todo

#### lsof

linux 进程资源管理，通过 lsof 列出所有资源列表，```lsof -i:8080， lsof -p 1``` 监听端口，监听进程资源列表，过滤资源列表等。

命令条件, 常用 -i -p

- -a 列出打开文件存在的进程
- -c<进程名> 列出指定进程所打开的文件
- -g 列出GID号进程详情
- -d<文件号> 列出占用该文件号的进程
- +d<目录> 列出目录下被打开的文件
- +D<目录> 递归列出目录下被打开的文件
- -n<目录> 列出使用NFS的文件
- -i<条件> 列出符合条件的进程。（4、6、协议、:端口、 @ip ）
- -p<进程号> 列出指定进程号所打开的文件
- -u 列出UID号进程详情
- -h 显示帮助信息
- -v 显示版本信息

TYPE 指定文件描述类型， 目录设备， 套接字类型。
（1）DIR：表示目录
（2）CHR：表示字符类型
（3）BLK：块设备类型
（4）UNIX： UNIX 域套接字
（5）FIFO：先进先出 (FIFO) 队列
（6）IPv4：网际协议 (IP) 套接字

FD:  文件描述符类型
（1）cwd：表示current work dirctory，即：应用程序的当前工作目录，这是该应用程序启动的目录，除非它本身对这个目录进行更改
（2）txt ：该类型的文件是程序代码，如应用程序二进制文件本身或共享库，如上列表中显示的 /sbin/init 程序
（3）lnn：library references (AIX);
（4）er：FD information error (see NAME column);
（5）jld：jail directory (FreeBSD);
（6）ltx：shared library text (code and data);
（7）mxx ：hex memory-mapped type number xx.
（8）m86：DOS Merge mapped file;
（9）mem：memory-mapped file;
（10）mmap：memory-mapped device;
（11）pd：parent directory;
（12）rtd：root directory;
（13）tr：kernel trace file (OpenBSD);
（14）v86  VP/ix mapped file;
（15）0：表示标准输入
（16）1：表示标准输出
（17）2：表示标准错误
一般在标准输出、标准错误、标准输入后还跟着文件状态模式：r、w、u等
（1）u：表示该文件被打开并处于读取/写入模式
（2）r：表示该文件被打开并处于只读模式
（3）w：表示该文件被打开并处于
（4）空格：表示该文件的状态模式为unknow，且没有锁定
（5）-：表示该文件的状态模式为unknow，且被锁定
同时在文件状态模式后面，还跟着相关的锁
（1）N：for a Solaris NFS lock of unknown type;
（2）r：for read lock on part of the file;
（3）R：for a read lock on the entire file;
（4）w：for a write lock on part of the file;（文件的部分写锁）
（5）W：for a write lock on the entire file;（整个文件的写锁）
（6）u：for a read and write lock of any length;
（7）U：for a lock of unknown type;
（8）x：for an SCO OpenServer Xenix lock on part      of the file;
（9）X：for an SCO OpenServer Xenix lock on the      entire file;
（10）space：if there is no lock.
