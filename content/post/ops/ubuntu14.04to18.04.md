---
title: "ubuntu 14.04 upgrade to 18.04(LTS)"
date: "2019-09-03 15:11:12"
tags: ops
  - ubuntu
  - upgrade
categories: [ops]
---

[TOC]


> ### 前言
>
> 升级ubuntu,为什么要升级呢?公司打算使用docker,老版本的ubuntu不支持docker-ce最新版,需要升级至18.04(LTS),这就很蛋疼了,对ubuntu的熟悉感差不多就只会dpcg,apt等简单的命令安装.查询了相关文档后,记录一下踩过的坑.

### requirment

To install Docker Engine - Community, you need the 64-bit version of one of these Ubuntu versions:

- Disco 19.04
- Cosmic 18.10
- Bionic 18.04 (LTS)
- Xenial 16.04 (LTS)

### 开启升级之旅14.04-16.04

三条命名直接升级.

```
$ sudo apt-get update 
$ sudo apt-get dist-upgrade
$ sudo do-release-upgrade

```

开启升级之旅16.04-18.04

```
$ sudo apt-get update 
$ sudo apt-get dist-upgrade
$ sudo do-release-upgrade
```

验证是否成功升级,发现lsb模块丢失

```
$ lsb_release -a  
No LSB modules are available. 
```

升级安装丢失的模块lsb即可

```
$ sudo apt-get install lsb-core
```

升级的过程如果只需要几条命令,那需要运维干啥呢~~.因为系统比较老,直接升级到最新版本,可能引发很多问题.

这边引发的问题是`python3`,` lsb-core` ,`lsb-release` ,`python3-apt`

```
louis@qx-prod-qukuailian:~$ sudo apt update && sudo apt upgrade
Hit:1 http://xxg.mirrors.ucloud.cn/ubuntu bionic InRelease
Get:2 http://xxg.mirrors.ucloud.cn/ubuntu bionic-updates InRelease [88.7 kB]         
Hit:5 https://mirrors.aliyun.com/docker-ce/linux/ubuntu bionic InRelease                                       
Get:3 http://xxg.mirrors.ucloud.cn/ubuntu bionic-backports InRelease [74.6 kB]                                 
Get:4 http://xxg.mirrors.ucloud.cn/ubuntu bionic-security InRelease [88.7 kB]
Fetched 252 kB in 1s (304 kB/s)                               
Reading package lists... Done
Building dependency tree       
Reading state information... Done
All packages are up to date.
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
4 not fully installed or removed.
After this operation, 0 B of additional disk space will be used.
Do you want to continue? [Y/n] y
Setting up python3 (3.6.7-1~18.04) ...
running python rtupdate hooks for python3.6...
dpkg-query: package 'dh-python' is not installed
Use dpkg --info (= dpkg-deb --info) to examine archive files,
and dpkg --contents (= dpkg-deb --contents) to list their contents.
Traceback (most recent call last):
  File "/usr/bin/py3clean", line 210, in <module>
    main()
  File "/usr/bin/py3clean", line 196, in main
    pfiles = set(dpf.from_package(options.package))
  File "/usr/share/python3/debpython/files.py", line 53, in from_package
    raise Exception("cannot get content of %s" % package_name)
Exception: cannot get content of dh-python
error running python rtupdate hook dh-python
dpkg: error processing package python3 (--configure):
 installed python3 package post-installation script subprocess returned error exit status 4
dpkg: dependency problems prevent configuration of lsb-core:
 lsb-core depends on python3; however:
  Package python3 is not configured yet.
 lsb-core depends on python3:any (>= 3.4~); however:
  Package python3 is not configured yet.

dpkg: error processing package lsb-core (--configure):
 dependency problems - leaving unconfigured
dpkg: dependency problems prevent configuration of lsb-release:
 lsb-release depends on python3:any (>= 3.4~); however:
  Package python3 is not configured yet.

dpkg: error processing package lsb-release (--configure):
 dependency problems - leaving unconfigured
dpkg: dependency problems prevent configuration of python3-apt:
 python3-apt depends on python3 (<< 3.7); however:
  Package python3 is not configured yet.
 python3-apt depends on python3 (>= 3.6~); however:
  Package python3 is not configured yet.
 python3-apt depends on python3:any (>= 3.3.2-2~); however:
  Package python3 is not configured yet.

dpkg: error processing package python3-apt (--configure):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 python3
 lsb-core
 lsb-release
 python3-apt
```

首先定位一下`python3`,查找`python3`相关依赖

```shell
root@qx-prod-qukuailian:~# dpkg --configure python3
Setting up python3 (3.6.7-1~18.04) ...
running python rtupdate hooks for python3.6...
dpkg-query: package 'dh-python' is not installed
Use dpkg --info (= dpkg-deb --info) to examine archive files,
and dpkg --contents (= dpkg-deb --contents) to list their contents.
Traceback (most recent call last):
  File "/usr/bin/py3clean", line 210, in <module>
    main()
  File "/usr/bin/py3clean", line 196, in main
    pfiles = set(dpf.from_package(options.package))
  File "/usr/share/python3/debpython/files.py", line 53, in from_package
    raise Exception("cannot get content of %s" % package_name)
Exception: cannot get content of dh-python
error running python rtupdate hook dh-python
dpkg: error processing package python3 (--configure):
 installed python3 package post-installation script subprocess returned error exit status 4
Errors were encountered while processing:
 python3
```

看起来像`python3`安装不行,google了半天,发现需要重装`Python3`

```shell
$ sudo apt-get install --reinstall python3
Reading package lists... Done
Building dependency tree       
Reading state information... Done
0 upgraded, 0 newly installed, 1 reinstalled, 0 to remove and 0 not upgraded.
4 not fully installed or removed.
After this operation, 0 B of additional disk space will be used.
E: Internal Error, No file name for python3:amd64
root@qx-prod-qukuailian:~# sudo apt update && sudo apt upgrade
Hit:1 http://xxg.mirrors.ucloud.cn/ubuntu bionic InRelease
Hit:2 http://xxg.mirrors.ucloud.cn/ubuntu bionic-updates InRelease                   
Hit:5 https://mirrors.aliyun.com/docker-ce/linux/ubuntu bionic InRelease             
Hit:3 http://xxg.mirrors.ucloud.cn/ubuntu bionic-backports InRelease
Hit:4 http://xxg.mirrors.ucloud.cn/ubuntu bionic-security InRelease
Reading package lists... Done                      
Building dependency tree       
Reading state information... Done
All packages are up to date.
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
4 not fully installed or removed.
After this operation, 0 B of additional disk space will be used.
Do you want to continue? [Y/n] y
Setting up python3 (3.6.7-1~18.04) ...
running python rtupdate hooks for python3.6...
dpkg-query: package 'dh-python' is not installed
Use dpkg --info (= dpkg-deb --info) to examine archive files,
and dpkg --contents (= dpkg-deb --contents) to list their contents.
Traceback (most recent call last):
  File "/usr/bin/py3clean", line 210, in <module>
    main()
  File "/usr/bin/py3clean", line 196, in main
    pfiles = set(dpf.from_package(options.package))
  File "/usr/share/python3/debpython/files.py", line 53, in from_package
    raise Exception("cannot get content of %s" % package_name)
Exception: cannot get content of dh-python
error running python rtupdate hook dh-python
dpkg: error processing package python3 (--configure):
 installed python3 package post-installation script subprocess returned error exit status 4
dpkg: dependency problems prevent configuration of lsb-core:
 lsb-core depends on python3; however:
  Package python3 is not configured yet.
 lsb-core depends on python3:any (>= 3.4~); however:
  Package python3 is not configured yet.

dpkg: error processing package lsb-core (--configure):
 dependency problems - leaving unconfigured
dpkg: dependency problems prevent configuration of lsb-release:
 lsb-release depends on python3:any (>= 3.4~); however:
  Package python3 is not configured yet.

dpkg: error processing package lsb-release (--configure):
 dependency problems - leaving unconfigured
dpkg: dependency problems prevent configuration of python3-apt:
 python3-apt depends on python3 (<< 3.7); however:
  Package python3 is not configured yet.
 python3-apt depends on python3 (>= 3.6~); however:
  Package python3 is not configured yet.
 python3-apt depends on python3:any (>= 3.3.2-2~); however:
  Package python3 is not configured yet.

dpkg: error processing package python3-apt (--configure):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 python3
 lsb-core
 lsb-release
 python3-apt
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

黑人问号,搞了半天,陷入循环了么,升级依赖需要这些`python3`,` lsb-core` ,`lsb-release` ,`python3-apt`依赖,我重新安装又得依赖这些,陷入了先有鸡还是先有蛋的问题;然后想了半天,重新再次查看日志

```
dpkg-query: package 'dh-python' is not installed
```

想了想,是不是应该安装这个依赖呢,直接就开启安装模式

```
root@qx-prod-qukuailian:~# apt-get install dh-python
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  python3-distutils python3-lib2to3
The following NEW packages will be installed:
  dh-python python3-distutils python3-lib2to3
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
4 not fully installed or removed.
Need to get 0 B/307 kB of archives.
After this operation, 2,556 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Selecting previously unselected package python3-lib2to3.
(Reading database ... 22162 files and directories currently installed.)
Preparing to unpack .../python3-lib2to3_3.6.8-1~18.04_all.deb ...
Unpacking python3-lib2to3 (3.6.8-1~18.04) ...
Selecting previously unselected package python3-distutils.
Preparing to unpack .../python3-distutils_3.6.8-1~18.04_all.deb ...
Unpacking python3-distutils (3.6.8-1~18.04) ...
Selecting previously unselected package dh-python.
Preparing to unpack .../dh-python_3.20180325ubuntu2_all.deb ...
Unpacking dh-python (3.20180325ubuntu2) ...
Setting up python3 (3.6.7-1~18.04) ...
running python rtupdate hooks for python3.6...
running python post-rtupdate hooks for python3.6...
Setting up lsb-release (9.20170808ubuntu1) ...
Setting up python3-lib2to3 (3.6.8-1~18.04) ...
Setting up python3-distutils (3.6.8-1~18.04) ...
Setting up python3-apt (1.6.4) ...
Setting up lsb-core (9.20170808ubuntu1) ...
Setting up dh-python (3.20180325ubuntu2) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
W: APT had planned for dpkg to do more than it reported back (24 vs 28).
   Affected packages: python3:amd64
root@qx-prod-qukuailian:~# sudo apt update && sudo apt upgrade
Hit:1 http://xxg.mirrors.ucloud.cn/ubuntu bionic InRelease
Hit:5 https://mirrors.aliyun.com/docker-ce/linux/ubuntu bionic InRelease
Hit:2 http://xxg.mirrors.ucloud.cn/ubuntu bionic-updates InRelease      
Hit:3 http://xxg.mirrors.ucloud.cn/ubuntu bionic-backports InRelease
Hit:4 http://xxg.mirrors.ucloud.cn/ubuntu bionic-security InRelease
Reading package lists... Done                      
Building dependency tree       
Reading state information... Done
All packages are up to date.
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

哟,没有报错,说明安装成功,日志的查看也非常重要,仔细分析报错内容,针对性的执行命令会更容易解决问题,验证一下升级是否成功.

```
$ lsb_release -a
LSB Version:	core-9.20170808ubuntu1-noarch:security-9.20170808ubuntu1-noarch
Distributor ID:	Ubuntu
Description:	Ubuntu 18.04.3 LTS
Release:	18.04
Codename:	bionic
```

升级完,查看`systemctl`,也就是systemd是否可以执行,执行命令发现python又报了一个错误.

```
root@qx-prod-qukuailian:~# systemctl
Traceback (most recent call last):
  File "/usr/lib/python3.6/dbm/gnu.py", line 4, in <module>
    from _gdbm import *
ModuleNotFoundError: No module named '_gdbm'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/CommandNotFound/CommandNotFound.py", line 7, in <module>
    import dbm.gnu as gdbm
  File "/usr/lib/python3.6/dbm/gnu.py", line 6, in <module>
    raise ImportError(str(msg) + ', please install the python3-gdbm package')
ImportError: No module named '_gdbm', please install the python3-gdbm package

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/lib/command-not-found", line 27, in <module>
    from CommandNotFound.util import crash_guard
  File "/usr/lib/python3/dist-packages/CommandNotFound/__init__.py", line 3, in <module>
    from CommandNotFound.CommandNotFound import CommandNotFound
  File "/usr/lib/python3/dist-packages/CommandNotFound/CommandNotFound.py", line 9, in <module>
    import gdbm
ModuleNotFoundError: No module named 'gdbm'
```

缺乏`gdbm`包,这里安装一下这个包就可以解决问题.

```
$ sudo apt-get install gdbm
```

重头戏来了,做了这么多,就是想要系统可以支持docker-ce最新版

```shell
$ curl -fsSL get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh --mirror Aliyun
$ sudo systemctl start docker
System has not been booted with systemd as init system (PID 1). Can't operate.
```

`docker`是成功安装了,但是`systemd`却没有启动起来,所以这里想了下,既然`docker`已经安装了,直接用`service`启动是否可以呢?结果发现启动成功

```shell
$ sudo service docker start
docker start/running, process 1451
$ sudo docker info
Client:
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 19.03.1
 Storage Driver: aufs
  Root Dir: /var/lib/docker/aufs
  Backing Filesystem: extfs
  Dirs: 0
  Dirperm1 Supported: false
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 894b81a4b802e4eb2a91d1ce216b8817763c29fb
 runc version: 425e105d5a03fabd737a126ad93d62a9eeede87f
 init version: fec3683
 Security Options:
  apparmor
  seccomp
   Profile: default
 Kernel Version: 3.13.0-63-generic
 Operating System: Ubuntu 18.04.3 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 7.798GiB
 Name: qx-prod-qukuailian
 ID: 3H4E:BQXC:DVYP:DOPO:FDKT:4HJH:MPMB:LSRP:3LDS:CW56:VZZL:Q6T2
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false

WARNING: No swap limit support
WARNING: the aufs storage-driver is deprecated, and will be removed in a future release.
```

### 总结

大功告成,在整个升级的过程中,曲曲折折,中途都想要重装系统了,但是通过一步一步分析报错信息,一步一步查找解决方案,去试错,最终问题就被解决.但是记得要有`PlanB`,升级之前记得做好快照,即使最后没有解决,我们可以通过快照恢复,不会破坏整个系统.

### 参考

- [No LSB modules are available.  ](https://askubuntu.com/questions/230766/how-lsb-module-affects-system-and-can-be-made-available-to-the-system)
- [No module named 'gdbm'](https://askubuntu.com/questions/720416/no-module-named-gdbm)
- [A Packages Problem,Unable to install anything due to unconfigured and depandesies](https://ubuntuforums.org/showthread.php?t=2148838)