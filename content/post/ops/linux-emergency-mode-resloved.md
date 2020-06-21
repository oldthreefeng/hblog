---
title: "记一次开机挂载导致linux重启出现紧急模式的问题及解决"
date: "2020-06-09T09:58:18+08:00"
tags: [lnux,fix,mount]
categories: [server,ops]
---

[TOC]

>  一次意外停电, 公司一台linux主机重启失败,  远程获取ip失败. 只能进到机房查看问题. 
`$表示shell, #表示注释, > 表示 数据库`

![](https://oss.fenghong.tech/tools/1591692298225.jpg)

## 问题发现

linux出现 `Welcome to emergency mode `字样, 首先查看`journalctl -xb`. 检查问题点

发现是某个目录挂载失败.  开机自动挂载问题`/etc/fstab` 是管理系统开机挂载问题的

```
$ vi /etc/fstab
UUID=7d1e179f-25a0-4998-9979-dc5c306c6b3f /                       ext4    defaults        1 1
b3f7c6e1-7583-4204-b3a3-0e7d1a727ee6 /data                   ext4    defaults        1 1
1532f91e-f13a-48e7-acd4-2c6fca4d8dd3 swap                    swap    defaults        0 0
tmpfs                   /dev/shm                tmpfs   defaults        0 0
devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
sysfs                   /sys                    sysfs   defaults        0 0
proc                    /proc                   proc    defaults        0 0

```

发现fastb文件写错, 以UUID挂载得写上`UUID=`.  填写正确的即可.

```
vim /etc/fstab

UUID=7d1e179f-25a0-4998-9979-dc5c306c6b3f /                       ext4    defaults        1 1
UUID=b3f7c6e1-7583-4204-b3a3-0e7d1a727ee6 /boot                   ext4    defaults        1 2
UUID=1532f91e-f13a-48e7-acd4-2c6fca4d8dd3 swap                    swap    defaults        0 0
tmpfs                   /dev/shm                tmpfs   defaults        0 0
devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
sysfs                   /sys                    sysfs   defaults        0 0
proc                    /proc                   proc    defaults        0 0

```

### 结语

一般linux开机出现挂载问题, 都是由于`/etc/fatab`文件写错导致.  更新该文件即可. 

