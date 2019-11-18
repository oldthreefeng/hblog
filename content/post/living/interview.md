---
title: "2019年面试"
date: 2019-11-12 20:56:12
tags: [interview]
categories: [living]
hiddenFromHomePage: true
---

[TOC]

### 银基安全 20191015 

一、请问对于批量修改配置文件的ip或者某一个字段,如何用shell或者python来写?或者你觉得用什么方式来处理最好.

```
shell的话用find + sed 
```

二、1000台虚拟机如何管理系统请问你有思路么?

```cgo
这个主要考你的devops, 首先,考虑告警处理, 1000台的问题如果人工运维的话,基本不可以,需要考虑自动化报警处理. 比如搭建监控系统. 
系统的配置更新, 比如使用ansible等自动化工具来进行管理. 
```

三、请说lvs,haProxy,nginx,keepalive的区别以及应用场景?

四、请问,你在公司运维一年多的时间内,碰到的最棘手的问题?(或者是最具有代表性的问题)

```
磁盘删除文件的问题, 删除大日志文件, 但是磁盘空间未释放问题
mysql数据库批量更新, 开发同学的一个`update`语句
公司单点系统转向负载均衡高可用的过程
服务器权限隔离,审计等功能的实现
```

五、请问你了解ittl管理么 (不知道是什么,没听清,it的方法管理论)

六、kubernetes的备份如何做? (基于命令行备份 ectdctl, ectd存储了k8s集群的所有状态.)

七、给你一个应用程序,需要部署, 已知并发峰值为100000.请问如何设计.

```cgo
数据库层面:不要让其每秒请求支撑超过2000，一般控制在2000左右。就是在上万并发请求的场景下，部署个5台服务器，每台服务器上都部署一个数据库实例。
大量分表的策略保证可能未来10年，每个表的数据量都不会太大，这可以保证单表内的SQL执行效率和性能
nginx网络应用层面, 实际生产环境能到2-3万并发连接数
采用lvs或者Haproxy.
```

### 20191016 比格基地

一、docker容器对CPU,内存,磁盘io等资源的控制?

[博客园大佬](https://www.cnblogs.com/sammyliu/p/5886833.html) 写一个程序

```c
int main(void)
{
    int i = 0;
    for(;;) i++;
    return 0;
}
$ gcc -o hello a.c
$ ./hello &
$ top
top - 23:04:03 up 92 days, 11:49,  4 users,  load average: 0.20, 0.94, 0.62

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                        
13619 root      20   0    4208    352    276 R 99.9  0.0   2:13.70 hello                                                                          
```

对该资源的控制

```bash
# cgroup中对cpu的限制
$ mkdir /sys/fs/cgroup/cpu/hello
$ ls
cgroup.clone_children  cgroup.procs  cpuacct.usage         cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  notify_on_release
cgroup.event_control   cpuacct.stat  cpuacct.usage_percpu  cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat 
$ cat cpu.cfs_quota_us
-1
$ echo 20000 > cpu.cfs_quota_us
$ echo 13619 >> tasks

$ top 
top - 23:07:48 up 92 days, 11:53,  4 users,  load average: 0.00, 0.44, 0.48

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                        
13619 root      20   0    4208    352    276 R 20.3  0.0   2:49.65 hello
```

docker控制cpu,内存,io,是基于cgroup的控制.

二、磁盘io占满,如何排查是由什么进程占用的

```cgo
 iotop -oP
 命令的含义：只显示有I/O行为的进程
 pidstat -d 1
 命令的含义：展示I/O统计，每秒更新一次
```

三、文件系统上生成一个文件到落盘的过程

![img](https://img-blog.csdn.net/20170819213317556?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvb1podVpoaVl1YW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

四、linux的文件系统nfs和ext的区别

参考[知乎大佬](https://www.zhihu.com/question/24413471/answer/38883787)

```cgo
EXT文件系统：
是固定的inode节点
格式化慢
修复慢
文件系统存储量有限
XFS文件系统：
高容量，大存储
inode与block都是在需要时产生的
```

nginx的并发优化

```cgo
$ cat nginx.conf 
net.core.somaxconn = 20480
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 4096 16777216
net.ipv4.tcp_wmem = 4096 4096 16777216
net.ipv4.tcp_mem = 786432 2097152 3145728
net.ipv4.tcp_max_syn_backlog = 16384
net.core.netdev_max_backlog = 20000
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_orphans = 131072
net.ipv4.tcp_syncookies = 0
$ sysctl -p

nginx层面
worker_connections 20000;
worker_process 1;
```

五、写出你们公司的现有的架构, 并解释为什么这么构建? (现场演算+讲述)

六、取出nginx日志的访问前十的ip? (现场演算+讲述)

七、常用的linux系统工具, 请列举尽可能多的 (现场演算+讲述)

### 君学中国 20191026

地址: 裕安大厦17楼 已收offer.

```cgo
面试问题:

1. 介绍一下自己
2. 作为一个运维工程师,需要具备哪些特质
3. mysql了解么? 讲一下吧
4. 你还有什么要问我的么
```

### 平安普惠 20191105

和初面差不多， 问题一: 最近一份公司干的内容. 我这边从三个方面来讲述：

1. 搭建jumpserver堡垒跳板机， 实现权限分离， 安全， 审计功能， 多云环境资产管控。
2. 搭建owncloud客户管理资料共享云盘， 解决数据公布及共享问题。 类似百度云盘
3. 搭建k8s和docker集群测试环境。

问题二： java的容器使用的是什么？ java的一些jvm参数了解么， 可以详细说一下你对jvm的了解， 以及对gc的理解么？这里也深入的问了一下java相关的， 我没有答出来

```
回答是使用jetty，   java的启动参数如下: 
JAVA_OPTS="-Xms2048m -Xmx2048m -XX:PermSize=256M -XX:MaxPermSize=512m"

参数说明：
1.Xms：
TOMCAT中JVM内存最小可用内存，此值可以设置与-Xmx相同，以避免每次垃圾回收完成后JVM重新分配内存。
2.Xmx：
TOMCAT中JVM内存最大可用内存；
3.-XX:PermSize=256M
设置永久域（非堆内存）的初始值，默认是物理内存的1/64, 建议不要超过256M；
4.-XX:MaxPermSize=512M
设置永久域的最大值，默认是物理内存的1/4，建议修改为512M；
5.-XX:+UseParallelGC： 
选择垃圾收集器为并行收集器
```

问题三: 对DB了解么？(mysql), 那说说mysql的高可用以及mysql的主从复制原理吧. 并追问了mysql的存储引擎

master 起一个线程.
```
当从节点连接主节点时，主节点会创建一个log dump 线程，用于发送bin-log的内容。在读取bin-log中的操作时，此线程会对主节点上的bin-log加锁，当读取完成，甚至在发动给从节点之前，锁会被释放。
```

从节点 起两个线程

```
1. I/O 当从节点上执行`start slave`命令之后，从节点会创建一个I/O线程用来连接主节点，请求主库中更新的bin-log。I/O线程接收到主节点binlog dump 进程发来的更新之后，保存在本地relay-log中。
2. SQL线程 SQL线程负责读取`relay log`中的内容，解析成具体的操作并执行，最终保证主从数据的一致性。
```

问题四. 对Zabbix监控或者openflacon或者promethoues等开源监控了解么? 讲一下吧

```
其实面试官可能想知道更深入的, 比如监控分组, 监控哪些内容, 具体如何监控.
我这边主要用的是阿里云的云监控系统, 以前用过zabbix监控, 我就说一下zabbix的监控吧. 
首先, 使用agent来进行采集, 采集方式有四种, agent/snmp/IPMI/jmx
监控项主要是对应的应用分组
触发器用来触发一系列的event事件: 界定某特定的item采集到的数据的非合理区间或非合理状态
动作主要来操作恢复事项或者通知, 比如执行一段shell, 发生e-mail等.
```

问题五. 基于k8s和docker的测试环境, 看你用过k8s, 你讲一下k8s的一些资源概念吧. 追问了一下服务暴露的相关问题.

```
k8s的资源概念非常之多, 从最基础的组件 pod, service, deployment, daemonSet, configMap, endpoint, ingress等...

现在大多数的k8s环境, 暴露方式都是 ingress+clusterIP, clusterIP主要是内部的服务直接相互通信使用, 
当然也有公司采用LoadBalancer方式(比如阿里云集群或者uk8s集群), 当然比较好用的是ingress的方式.
ingress是整个流量的注入口, 后端有个 ingress controller来进行分发至 service服务, 再有service调度至pod, 完成整个访问. 
```

### 时代天使 20191112

1.ansible的使用的协议是什么? 你经常用到哪些模块?

```
ansible 默认使用ssh协议!

常用的模块有:
copy template shell playbook
```

2.`ln -s和 ln和cp`的区别

```
ln a b
硬链接: 硬链接实际上是为文件建一个别名，
链接文件和原文件实际上是同一个文件.可以通过ls -i来查看一下，
这两个文件的inode号是同一个，说明它们是同一个文件.

ln -sv a b
通过软链接建立的链接文件与原文件并不是同一个文件，
相当于原文件的快捷方式。具体理解的话，链接文件内存储的是原文件的inode，
也就是说是用来指向原文件文件，这两个文件的inode是不一样的.

cp a b
相当于将原文件进行一个拷贝，为另一个全新的文件，与原文件没有关系了
```

各自的特点

```
硬链接的特点是这样的：
它会在链接文件处创建一个和被链接文件一样大小的文件，类似于国外网站和国内镜像的关系，
硬链接占用的空间和被链接文件一样大（其实就是同一片空间）
修改链接文件和被链接文件中的其中一个，另外一个随之同样发生变化
硬链接的对象不能是目录，也就是说被链接文件不能为目录
硬链接的两个文件是独立的两个引用计数文件，他们共用同一份数据，所以他们- 的inode节点相同
删除硬链接中的任意一个文件，另外一个文件不会被删除。没有任何影响，链接文件一样可以访问，内容和被链接文件一模一样。

软链接的特点：
软连接的链接文件就是一个基本单元大小的文件，一般为3B，和被链接文件的大小没有关系
软链接的链接文件中存储的是被链接文件的元信息，路径或者inode节点
软连接的连接文件是一个独立的文件，有自己的元信息和inode节点
删除软链接的链接文件，被链接文件不会受到任何影响
删除软链接的被链接文件，链接文件会变成红色，这时打开链接文件会报错，报找不到被链接的文件这种错误
软链接可以链接任何类型的文件，包括目录和设备文件都可以作为被链接的对象

复制的特点：
复制产生的文件是一个独立的文件，有自己的元信息和inode节点
删除或修改复制文件，对原文件不会产生任何影响，反过来也是一样的
复制可以复制文件，也可以复制目录
```

3.取日志里面的ip

```shell
$ grep -E -o "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" /etc/hosts
```

4.dokcerfile的add和copy区别，cmd和endpoint区别. (韵达已经回答了前者)

```dockerfile
# CMD 指令指定容器启动时需要运行的程序,给出的是一个容器的默认的可执行体。
# 也就是容器启动以后，默认的执行的命令. 
CMD ["/bin/bash"] 

# ENTRYPOINT entrypoint才是正统地用于定义容器启动以后的执行体的，
# 其实我们从名字也可以理解，这个是容器的“入口”.
ENTRYPOINT ["echo"] 
CMD ["test cmd"]  
# 将cmd的参数传递给entrypoint. 
```

总结

```
一般还是会用entrypoint的中括号形式作为docker 容器启动以后的默认执行命令,
里面放的是不变的部分，可变部分比如命令参数可以使用cmd的形式提供默认版本，
也就是run里面没有任何参数时使用的默认参数。如果我们想用默认参数，就直接run，
否则想用其他参数，就run 里面加参数.
```

5.k8s里面的`replicaset和daemonset`的区别。

replicaSet

```
Kubernetes 中的 ReplicaSet 主要的作用是维持一组 Pod 副本的运行，
它的主要作用就是保证一定数量的 Pod 能够在集群中正常运行，
它会持续监听这些 Pod 的运行状态，在 Pod 发生故障重启数量减少时重新运行新的 Pod 副本。

原理: 
所有 ReplicaSet 对象的增删改查都是由 ReplicaSetController 控制器完成的，
该控制器会通过 Informer 监听 ReplicaSet 和 Pod 的变更事件并将其加入持有的待处理队列.
ReplicaSetController 中的 queue 其实就是一个存储待处理 ReplicaSet 的『对象池』，
它运行的几个 Goroutine 会从队列中取出最新的数据进行处理
```

DaemonSet

```
DaemonSet 可以保证集群中所有的或者部分的节点都能够运行同一份 Pod 副本，
每当有新的节点被加入到集群时，Pod 就会在目标的节点上启动，如果节点被从集群中剔除，
节点上的 Pod 也会被垃圾收集器清除

原理: 
所有的 DaemonSet 都是由控制器负责管理的，与其他的资源一样，
用于管理 DaemonSet 的控制器是 DaemonSetsController，
该控制器会监听 DaemonSet、ControllerRevision、Pod 和 Node 资源的变动
大多数的触发事件最终都会将一个待处理的 DaemonSet 资源入栈，
下游 DaemonSetsController 持有的多个工作协程就会从队列里面取出资源进行消费和同步。
```

6.`k8s`里面服务是怎么暴露的

```
现在大多数的k8s环境, 暴露方式都是 ingress+clusterIP, 
clusterIP主要是内部的服务直接相互通信使用, 当然也有公司采用LoadBalancer方式
(比如阿里云集群或者uk8s集群), 当然比较好用的是ingress的方式.  ingress是整个流量的注入口, 
后端有个 ingress controller来进行分发至 service服务, 再有service调度至pod, 完成整个访问的链路. 
```

7.`mysql`如何备份。

```shell
#!/bin/sh
# mysql data backup script
#
# use mysqldump --help,get more detail.
#
BakDir=/data/mysql
LogFile=/data/mysql/mysqlbak.log
DATE=`date +%Y%m%d_%H%M%S`
[ -z $1 ] && exit
database=$1
echo " " >> $LogFile
echo " " >> $LogFile
echo "--------------------------" >> $LogFile 
echo $(date +"%y-%m-%d %H:%M:%S") >> $LogFile 
echo "--------------------------" >> $LogFile 
cd $BakDir/${database}
GZDumpFile=${database}-$DATE.sql.gz
/usr/bin/mysqldump -ubak -h'localhost' -p'password' --databases $database |gzip > $GZDumpFile
echo "Dump Done" >> $LogFile
echo "[$GZDumpFile]Backup Success." >> $LogFile
find $BakDir/${database} -ctime +300 -exec rm {} \;
```

8.线上日志如何备份.

```shell
#!/bin/bash
#显示上个月的时间如201808
last_one_month=$(date +%Y%m --date="-1 month")
#last_two_month=$(date +%Y%m --date="-2 month")
logdir=/udisk/logs/
gzdir=/udisk/gz/logs
#nginx_backup
cd ${logdir}
## delete logs file before 1 month ago ##
find ${logdir}/ -mtime +35 -exec rm -f {} \;

## tar project logs ##
tar zcf  activity.${last_one_month}.tar.gz activity
tar zcf  nginx.${last_one_month}.tar.gz nginx
tar zcf  cms.${last_one_month}.tar.gz cms
tar zcf  cron.${last_one_month}.tar.gz cron
tar zcf  borrow.${last_one_month}.tar.gz borrow
tar zcf  manage.${last_one_month}.tar.gz manage
tar zcf  borrowWap.${last_one_month}.tar.gz borrowWap
tar zcf  rms.${last_one_month}.tar.gz rms
tar zcf  wap.${last_one_month}.tar.gz wap
tar zcf  www.${last_one_month}.tar.gz www
tar zcf  www2.${last_one_month}.tar.gz www2

## move the tar.gz file ##
mv *.gz ${gzdir}

## backup everyday ##
cd ${logdir}/mobile
for file in ./* ; do
        tar zcf  ${file}.tar.gz ${file}
        mv ${file}.tar.gz ${gzdir}
done
```

9.sql某个字段有索引，为什么执行了explain却发现没用到索引？

```sql
-- 1.like查询中，使用%

-- 2.隐式转换导致不走索引。
    SELECT * FROM T WHERE Y = 5 
    -- 但是Y列是VARCHAR, 编译器会存在一个隐式的转换
    SELECT * FROM T WHERE Y = '5' 

-- 3.索引列上有函数运算，导致不走索引
    SELECT * FROM T WHERE FUN(Y) = XXX    
      
-- 4. ！=或者<>(不等于），可能导致不走索引，也可能走 INDEX FAST FULL SCAN
    select id  from test where id<>100

-- 5. 查询谓词没有使用索引的主要边界,换句话说就是select *，可能会导致不走索引。
    SELECT * FROM T WHERE Y=XXX
    -- 假如你的T表上有一个包含Y值的组合索引，但是优化器会认为需要一行行的扫描会更有效
-- 6. 条件为not in ,not exist
   
...
```

10.如果让你来管理2000台虚拟机, 你应该如何管理?(银基安全已经出了这个题目)

```
这个主要考你的devops, 首先,考虑告警处理;
2000台的问题如果人工运维的话,基本不可能实现,
需要考虑自动化报警处理. 比如搭建监控系统zabbix, 
系统的配置批量更新, 比如使用ansible等自动化工具来进行管理. 
```

11.`arp`协议原理,请讲述一下

```
# 计算机中会维护一个ARP缓存表，这个表记录着IP地址与MAC地址的映射关系;
# 查看arp记录表可以用如下命令:
$ arp -a 

# 在以太网中，一个主机要和另一个主机进行直接通信，必须要知道目标主机的MAC地址。
# 但这个目标MAC地址是如何获得的呢？ 它就是通过地址解析协议获得的。
# 所谓“地址解析”就是主机在发送帧前将目标IP地址转换成目标MAC地址的过程。
# ARP协议的基本功能就是通过目标设备的IP地址，查询目标设备的MAC地址，以保证通信的顺利进行。

# ARP协议的主要工作就是建立、查询、更新、删除ARP表项。
```

