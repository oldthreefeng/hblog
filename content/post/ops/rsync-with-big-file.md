---
title: "rsync大文件传输导致磁盘异常爆增"
date: 2020-02-10T11:12:18+08:00
lastmod: 2020-02-10T11:12:18+08:00
tags: [rsync,logs]
categories: [server,ops]
---

[TOC]

> 背景： 每天四次服务器磁盘巡检，一切正常， 突然到晚上十点的时候， 阿里云磁盘报警， 超过85%用量， 很纳闷， 查找并发现问题的过程， 记录下来。 

## 说明

- \# 开头的行表示注释
- \> 开头的行表示需要在 `mysql `中执行
- $ 开头的行表示需要执行的命令

```
本文档适用于有一定web运维经验的管理员或者工程师，文中不会对安装的软件做过多的解释
```

## 解决方法及过程

报警如下: 

![](http://pic.fenghong.tech/rsync/20200204202921.jpg)

发现磁盘是在下午16:46开始增长的。 到晚上22:42才发出报警, 这里在负载上面也达到了恐怖的60+。 

![1581306128857](http://pic.fenghong.tech/rsync/1581306128857.png)

磁盘读写情况: 

![1581306211304](http://pic.fenghong.tech/rsync/1581306211304.png)

登录服务器后, 先查看磁盘用量, 初步怀疑是日志异常增加导致的.

```
$ df -h
$ cd /data/prod-logs
$ du -sh *
111M	adcom
34M	c
773M	card
1.3G	card20191125bak
396M	card2-20191125bak
23M	cdxb
327M	consul
195G	e-mall
17G	e-mall2
300M	e-mall3
140M	fotoup
838M	hudong
73M	juncafe
18G	junzaoan
44G	nginx
1.2G	node-log
3.3G	task
30M	weihou
6.9G	ypl
[root@yunwei prod-logs]# du -sh 
288G	.

# 查看同步进程, 发现rsync同步出现大量进程
$ ps aux |grep rsync | wc -l 
10
$ ps aux |grep rsync
root       955 14.7  0.0 108888  2436 ?        Ds   11:21   0:01 rsync --server -vlogDtprze.iLs . /data/prod-logs/task/
root       962 14.0  0.0 108868  2524 ?        Rs   11:21   0:00 rsync --server -vlogDtprze.iLs . /data/prod-logs/node-log/118/
root      1051  0.0  0.0 108608   260 ?        S    11:21   0:00 rsync --server -vlogDtprze.iLs . /data/prod-logs/task/
root      1059  0.0  0.0 108608   208 ?        S    11:21   0:00 rsync --server -vlogDtprze.iLs . /data/prod-logs/node-log/118/
...
$ find ./ -name "\.*" -size +10M -type f  | xargs ls -lh
```

![](http://pic.fenghong.tech/rsync/rsync-01.png)

找到了异常文件的所在位置.  一开始很纳闷, 这些临时文件是怎么产生的.  google之后, 找到答案[rsync原理和工作流程分析](https://www.cnblogs.com/f-ck-need-u/p/7226781.html)

> **当α主机发现是匹配数据块时，将只发送这个匹配块的附加信息给β主机。同时，如果两个匹配数据块之间有非匹配数据，则还会发送这些非匹配数据。当β主机陆陆续续收到这些数据后，会创建一个临时文件，并通过这些数据重组这个临时文件，使其内容和A文件相同。临时文件重组完成后，修改该临时文件的属性信息(如权限、所有者、mtime等)，然后重命名该临时文件替换掉B文件，这样B文件就和A文件保持了同步。**

服务器上每三分钟同步一次日志.  多台服务器同步日志到192.168.0.21上. 

```
#*/3 * * * * sshpass -p '123456' rsync -avz -e 'ssh -p 33021' /usr/local/card/logs/ root@192.168.0.21:/data/prod-logs/card/
*/3 * * * * sshpass -p '123456' rsync -avz -e 'ssh -p 33021' /usr/local/photoup-server/logs/ root@192.168.0.21:/data/prod-logs/fotoup/
#*/3 * * * * sshpass -p '123456' rsync -avz -e 'ssh -p 33021' /usr/local/card/logs/ root@192.168.0.21:/data/prod-logs/card/
#*/3 * * * * sshpass -p '123456' rsync -avz -e 'ssh -p 33021' /usr/local/juncafe/logs/ root@192.168.0.21:/data/prod-logs/juncafe/

*/10 * * * * sshpass -p '123456' rsync -avz -e 'ssh -p 33021' /root/.pm2/logs/ root@192.168.0.21:/data/prod-logs/node-log/123/

*/3 * * * * sshpass -p '123456' rsync -avz -e 'ssh -p 33021' /usr/local/nginx/log/*.gz root@192.168.0.21:/data/prod-logs/nginx/123/
## 每天三次巡检
0 1,9,12,18 * * * /bin/sh  /data/louis/manage.sh
```

如果出现了大日志文件同步(比如5G以上的文件), 建议做分隔, 不然每3分钟同步, 会导致这样的问题:  3分钟时间, rsync还没有同步完新增文件, 服务器又开始新增同步文件, 导致无限循环. 服务器压力会持续增大, 最终导致服务器负载异常, 从而宕机. 

从源头上切割大文件日志, 然后日志同步时间增加, 比如3分钟同步改成5分钟或者十分钟, 可以暂时缓解这个问题.

## 结论

1.对大日志文件的管理, 必须每日进行切分. 防止此类事故再次发生(如使用`logrotate`管理).

```
$ cat /etc/logrotate.d/tomcat-daka
/usr/local/apache-tomcat-daka/logs/catalina.out {
daily
rotate 15
missingok
dateext
compress
notifempty
copytruncate
}

```

2.增加服务器负载到达10或者cpu到100%的报警通知, 此次告警为磁盘告警才被通知到. 加快处理响应时间.

![1581307240624](http://pic.fenghong.tech/rsync/1581307240624.png)