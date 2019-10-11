---
title: "dmesg 添加时间戳"
date: 2019-05-16 09:48:12
tags: [dmesg]
categories: [tools]
---

[TOC]

公司系统是centos6.5的，生产环境偶发一次oom，想查询更多细节的问题，通过dmesg查询oom情况时，发现没有时间戳，于是有了这篇文章。

### 日志无时间戳的原因

- CentOS 7之前的版本的dmesg日志是没有时间戳的，原因是util-linux-ng版本太低,不具备日期显示功能
- 原因是`/sys/module/printk/parameters/time`为N即0，不开启状态
- `/sys/module/*`包含所有编译的模块信息

> 这里有系统中所有模块的信息，不论这些模块是以内联(inlined)方式编译到内核映像文件(vmlinuz)中还是编译为外部模块(ko文件)，都可能会出现在 /sys/module 中：
>
> - 编译为外部模块(ko文件)在加载后会出现对应的 /sys/module/<module_name>/, 并且在这个目录下会出现一些属性文件和属性目录来表示此外部模块的一些信息，如版本号、加载状态、所提供的驱动程序等；
> - 编译为内联方式的模块则只在当它有非0属性的模块参数时会出现对应的
>   **/sys/module/<module_name>**, 这些模块的可用参数会出现在
>   **/sys/modules/<modname>/parameters/<param_name>** 中

-  `/sys/module/printk/parameters/time` 这个可读写参

#### 修改`/sys/module/printk/parameters/time`参数

- 使其开始为今后日志添加时间戳，但是重启后会失效
- 可以使用dmesg查询

```
]$ echo 1 >/sys/module/printk/parameters/time
]$ cat  /sys/module/printk/parameters/time
Y
```

#### 在监控日志配置`/etc/rsyslog.conf`中，添加监控kern的信息，并重启rsyslog服务

- 从服务重启后开始生效，kern日志都记录在`/var/log/kern.log`中
- 但重启后用dmesg查看的日志依然没有时间戳；因为/sys/下的目录存放的是系统内存的信息，重启会失效；
- 同时，`/var/log/kern.log`中的日志的时间格式是人类易读的

```bash
]$ sed -i '/local7/a\kern.*       /var/log/kern.log' /etc/rsyslog.conf
]$ grep kern.log  /etc/rsyslog.conf  
kern.*      /var/log/kern.log
]$ service rsyslog restart
Shutting down system logger:                               [  OK  ]
Starting system logger:                                    [  OK  ]
]#
```

### 时间戳转换

- 由于dmesg时间戳可读性很差

```bash
$ dmesg
[18609174.454942] Adding 524280k swap on /swapfile.  Priority:-1 extents:4 across:663544k 
[18609179.675345] Adding 524280k swap on /swapfile.  Priority:-1 extents:4 across:663544k
```

- 利用awk来进行转换时间戳

```bash
$ vim  ts_dmesg.sh
#!/bin/sh
 
uptime_ts=`cat /proc/uptime | awk '{ print $1}'`
#echo $uptime_ts
dmesg | awk -v uptime_ts=$uptime_ts 'BEGIN {
    now_ts = systime();
    start_ts = now_ts - uptime_ts;
    #print "system start time seconds:", start_ts;
    #print "system start time:", strftime("[%Y/%m/%d %H:%M:%S]", start_ts);
 }
{
    print strftime("[%Y/%m/%d %H:%M:%S]", start_ts + substr($1, 2, length($1) - 2)), $0
}'

$ bash ts_dmesg.sh
[2019/05/16 09:10:59] [18609174.454942] Adding 524280k swap on /swapfile.  Priority:-1 extents:4 across:663544k 
[2019/05/16 09:11:04] [18609179.675345] Adding 524280k swap on /swapfile.  Priority:-1 extents:4 across:663544k 

```

ps： centos7.0以上，dmesg自带时间戳，`dmesg -T` 即可转换

### 参考

  [dmesg添加时间戳](https://www.jianshu.com/p/1780360cfd2b)