---
title: "端口扫描及用户安全"
date: "2019-01-12 09:59:32"
tags: 
  - safe
  - Linux
categories: [server]
---

[TOC]

### 端口扫描

采用`nmap`进行扫描`1-65535`端口

```
[root@qx_production_PC qianxiang]# nmap -sS -p 1-65535 localhost
Starting Nmap 5.51 ( http://nmap.org ) at 2019-01-16 10:30 CST
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0000020s latency).
Other addresses for localhost (not scanned): 127.0.0.1
Not shown: 65511 closed ports
PORT      STATE SERVICE
80/tcp    open  http
81/tcp    open  hosts2-ns
443/tcp   open  https
3306/tcp  open  mysql
6822/tcp  open  unknown
8005/tcp  open  mxi
8009/tcp  open  ajp13
...
```

### 用户 ###
空密码用户，root用户及root组用户名单查询
```
awk -F: 'length($2)==0 {print $1}' /etc/shadow
awk -F: '$3==0 {print $1}' /etc/passwd
awk -F: '$4==0 {print $1}' /etc/passwd
```

### 权限后门 ###
```
cat /etc/crontab
ls /var/spool/cron/
cat /etc/rc.d/rc.local
ls /etc/rc.d
ls /etc/rc3.d
find / -type f -perm 4000
```