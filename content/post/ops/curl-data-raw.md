---
title: "记一次curl版本yum升级至7.71.0"
date: 2020-06-28T10:54:18+08:00
tags: [curl,yum,libcurl,libssh2]
categories: [ops,jenkins]
---

[TOC]

## 背景
> 偶然有一次机会使用到了curl 命令行中 `--data-raw` 选项， 但是提示是`curl: option --data-raw: is unknown`, 网上查询了蛮多， 也没有写什么解决方案，估摸着是版本太低的缘故。

目前的centos 7 的yum仓库版本, 最新版应该到了` 7.71.0` 了 ,`Release-Date: 2020-06-24`

```
$ curl -V
curl 7.29.0 (x86_64-redhat-linux-gnu) libcurl/7.29.0 NSS/3.44 zlib/1.2.7 libidn/1.28 libssh2/1.8.0
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp scp sftp smtp smtps telnet tftp 
Features: AsynchDNS GSS-Negotiate IDN IPv6 Largefile NTLM NTLM_WB SSL libz unix-sockets 
```

在 RedHat/CentOS 系统中，curl 默认使用的密码学库是 NSS，升级 curl 有两种方法，分别是编译安装和包安装。选择yum升级方便，编译升级太耗时。

查看 curl 官方页面 http://curl.haxx.se/download.html#LinuxRedhat，找到对应页面 https://mirror.city-fan.org/ftp/contrib/sysutils/Mirroring，这个页面的介绍非常详细。

### 升级

一开始没考虑依赖``libcurl`和`libssh2`问题。 直接安装curl,  直接报错~ 

`libcurl(x86-64) >= 7.71.0-1.0.cf.rhel7`

`libssh2(x86-64) >= 1.9.0`

```
$ yum install http://www.city-fan.org/ftp/contrib/yum-repo/rhel7/x86_64/curl-7.71.0-1.0.cf.rhel7.x86_64.rpm
...
Error: Package: curl-7.71.0-1.0.cf.rhel7.x86_64 (/curl-7.71.0-1.0.cf.rhel7.x86_64)
           Requires: libcurl(x86-64) >= 7.71.0-1.0.cf.rhel7
           Installed: libcurl-7.29.0-57.el7.x86_64 (@base)
               libcurl(x86-64) = 7.29.0-57.el7


$ yum install http://www.city-fan.org/ftp/contrib/yum-repo/rhel7/x86_64/curl-7.71.0-1.0.cf.rhel7.x86_64.rpm  http://www.city-fan.org/ftp/contrib/yum-repo/rhel7/x86_64/libcurl-7.71.0-1.0.cf.rhel7.x86_64.rpm
...
Error: Package: libcurl-7.71.0-1.0.cf.rhel7.x86_64 (/libcurl-7.71.0-1.0.cf.rhel7.x86_64)
           Requires: libssh2(x86-64) >= 1.9.0
           Installed: libssh2-1.8.0-3.el7.x86_64 (@anaconda)
               libssh2(x86-64) = 1.8.0-3.el7

```

安装依赖`libcurl`和`libssh2`即可解决

```
$ yum install http://www.city-fan.org/ftp/contrib/yum-repo/rhel7/x86_64/curl-7.71.0-1.0.cf.rhel7.x86_64.rpm  http://www.city-fan.org/ftp/contrib/yum-repo/rhel7/x86_64/libcurl-7.71.0-1.0.cf.rhel7.x86_64.rpm http://www.city-fan.org/ftp/contrib/yum-repo/rhel7/x86_64/libssh2-1.9.0-5.0.cf.rhel7.x86_64.rpm -y
```

安装完成后, 查看安装的版本号, `centos 6`安装应该大同小异, 这边就不赘述了.  

```
$ curl -V
curl 7.71.0 (x86_64-redhat-linux-gnu) libcurl/7.71.0 NSS/3.44 zlib/1.2.7 libpsl/0.7.0 (+libicu/50.1.2) libssh2/1.9.0 nghttp2/1.33.0
Release-Date: 2020-06-24
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp scp sftp smb smbs smtp smtps telnet tftp 
Features: AsynchDNS GSS-API HTTP2 HTTPS-proxy IPv6 Kerberos Largefile libz Metalink NTLM NTLM_WB PSL SPNEGO SSL UnixSockets
```

