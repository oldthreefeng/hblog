---
title: "Linux上tcp处于time_wait"
date: 2019-02-22 23:48:12
tags: [tcp]
categories: [tools]
---

[TOC]

### linux上大量tcp端口处于TIME_WAIT的问题

最近发现几个监控用的脚本在连接监控数据库的时候偶尔会连不上，报错：

```
 Couldn't connect to host:3306/tcp: IO::Socket::INET: connect: Cannot assign requested address 
```

查看了一下发现系统中存在大量处于TIME_WAIT状态的tcp端口

```
$netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}' 
TIME_WAIT 50013
ESTABLISHED 27
SYN_RECV 1
```

由于要监控的主机太多，监控的agent可能在短时间内创建大量连接到监控数据库(MySQL)并释放造成的。在网上查阅了一些tcp参数的相关资料，最后通过修改了几个系统内核的tcp参数缓解了该问题：

```
#vi /etc/sysctl.conf

net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1

#sysctl -p
```

其中：
`net.ipv4.tcp_tw_reuse = 1 `表示开启重用。允许将`TIME-WAIT sockets`重新用于新的TCP连接，默认为0，表示关闭；
`net.ipv4.tcp_tw_recycle = 1 `表示开启TCP连接中`TIME-WAIT sockets`的快速回收，默认为0，表示关闭。

修改完成并生效后，系统中处于`TIME_WAIT`状态的tcp端口数量迅速下降到100左右：

```
$netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}' 
TIME_WAIT 82
ESTABLISHED 36
```

简单记录于此，备忘。

