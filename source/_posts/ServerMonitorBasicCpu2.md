---
title: ServerMonitorBasicCpu2
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

相关命令 

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

端口监听 lsof -i:8080
进程资源列表 todo
