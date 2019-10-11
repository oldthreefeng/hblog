---
title: rsync和inotify实现文件同步
date: 2018-11-07 09:59:32
urlname: rsync
tags: 
- Linux
- rsync
- inotify
categories: server
---

# rsync

## 什么是rsync

rsync是一个远程数据同步工具，可通过LAN/WAN快速同步多台主机间的文件。它使用所谓的“Rsync演算法”来使本地和远程两个主机之间的文件达到同步，这个算法只传送两个文件的不同部分，而不是每次都整份传送，因此速度相当快。所以通常可以作为备份工具来使用。

运行Rsync server的机器也叫backup server，一个Rsync server可同时备份多个client的数据；也可以多个Rsync server备份一个client的数据。Rsync可以搭配ssh甚至使用daemon模式。Rsync server会打开一个873的服务通道(port)，等待对方rsync连接。连接时，Rsync server会检查口令是否相符，若通过口令查核，则可以开始进行文件传输。第一次连通完成时，会把整份文件传输一次，下一次就只传送二个文件之间不同的部份。

**基本特点：**

1. 可以镜像保存整个目录树和文件系统；
2. 可以很容易做到保持原来文件的权限、时间、软硬链接等；
3. 无须特殊权限即可安装；
4. 优化的流程，文件传输效率高；
5. 可以使用rcp、ssh等方式来传输文件，当然也可以通过直接的socket连接；
6. 支持匿名传输。

## rsync同步过程：

rsync在同步文件的时候， 接收端会从发送端的数据中读取由文件索引号确认的文件. 然后打开[本地文件](http://blog.uouo123.com/tags-2217.html)(被称为基础文件), 建立一个临时文件. 接收端会读取非匹配数据和匹配数据, 并按顺序重组他们成为最终文件. 当非匹配数据被读取, 它会被写入到临时文件. 当收到一个块匹配记录, 接收端会寻找这个块在基础文件中的偏移量, 将这个块拷贝到临时文件. 通过这种方式, 临时文件被从头到尾建立起来. 建立临时文件的时候生成了文件的校验. 重建文件结束后, 这个校验和来自发送端的校验比较. 如果校验不符, 临时文件会被删除. 如果失败一次, 文件会再被处理一次. 如果失败第二次, 一个错误会被报告. 临时文件建立后, 所有者, 权限和修改时间会被设置. 然后它会被重命名已替代基础文件.

## Rsync工作原理 

1）软件简介

Rsync 是一个远程数据同步工具，可通过 LAN/WAN 快速同步多台主机间的文件。Rsync 本来是用以取代rcp 的一个工具，它当前由 Rsync.samba.org 维护。Rsync 使用所谓的“Rsync 演算法”来使本地和远程两个主机之间的文件达到同步，这个算法只传送两个文件的不同部分，而不是每次都整份传送，因此速度相当快。运行 Rsync server 的机器也叫 backup server，一个 Rsync server 可同时备份[多个](http://blog.uouo123.com/tags-1126.html) client 的数据；也可以多个Rsync server 备份一个 client 的数据。

Rsync 可以搭配 rsh 或 ssh 甚至使用 daemon 模式。Rsync server 会打开一个873的服务通道（port），等待对方 Rsync 连接。连接时，Rsync server 会检查口令是否相符，若通过口令查核，则可以开始进行文件传输。第一次连通完成时，会把整份文件传输一次，下一次就只传送二个文件之间不同的部份。

Rsync 支持大多数的类 Unix 系统，无论是 Linux、Solaris 还是 BSD 上都经过了良好的测试。此外，它在windows 平台下也有相应的版本，比较知名的有 cwRsync 和 Sync2NAS。

Rsync 的基本特点如下：

可以镜像保存整个目录树和[文件系统](http://blog.uouo123.com/tags-1181.html)；
可以很容易做到保持原来文件的权限、时间、软硬链接等；
无须[特殊权限](http://blog.uouo123.com/tags-1207.html)即可安装；
优化的[流程](http://blog.uouo123.com/tags-656.html)，文件传输效率高；
可以使用 rcp、ssh 等方式来传输文件，当然也可以通过直接的 socket 连接；
支持匿名传输。

2）核心算法

假定在名为 α 和 β 的两台计算机之间同步相似的文件 A 与 B，其中 α 对文件A拥有访问权，β 对文件 B 拥有访问权。并且假定主机 α 与 β 之间的网络带宽很小。那么 Rsync 算法将通过下面的五个步骤来完成：

β 将文件 B 分割成一组不重叠的固定大小为 S 字节的数据块。[最后一块](http://blog.uouo123.com/tags-1066.html)可能会比 S 小。
β 对每一个分割好的数据块执行两种校验：一种是32位的滚动弱校验，另一种是128位的 MD4 强校验。
β 将这些校验结果发给 α。
α 通过搜索文件 A 的所有大小为 S 的数据块（偏移量可以任选，不一定非要是 S 的倍数），来寻找与文件B 的某一块有着相同的弱校验码和强校验码的数据块。这项工作可以借助滚动校验的特性很快完成。
α 发给 β 一串指令来生成文件 A 在 β 上的备份。这里的每一条指令要么是对文件 B 经拥有某一个数据块而不须重传的证明，要么是一个数据块，这个数据块肯定是没有与文件 B 的任何一个数据块匹配上的。

 

3） 文件级别的RSync（只传输变化的文件）工作过程：（我的理解）

\* 机器A构造FileList，FileList包含了需要与机器B sync的所有文件信息对name->id,（id用来唯一表示文件例如MD5）；
\* 机器A将FileList发送到机器B；
\* 机器B上运行的后台程序处理FileList，构建NewFileList，其中根据MD5的比较来删除机器B上已经存在的文件的信息对，只保留机器B上不存在或变化的文件;
\* 机器A得到NewFileList，对NewFileList中的文件从新传输到机器B；

## 安装：

rsync在CentOS6上默认已经安装，如果没有则可以使用`yum install rsync -y`，服务端和客户端是同一个安装包。

## 同步到远程服务器

在服务器间rsync传输文件，需要有一个是开着rsync的服务，而这一服务需要两个配置文件，说明当前运行的用户名和用户组，这个用户名和用户组在改变文件权限和相关内容的时候有用，否则有时候会出现提示权限问题。配置文件也说明了模块、模块化管理服务的安全性，每个模块的名称都是自己定义的，可以添加用户名密码验证，也可以验证IP，设置目录是否可写等，不同模块用于同步不同需求的目录。

### 服务端配置文件

**/etc/rsyncd.conf：** 

```
#2014-12-11 by Sean
uid = root
gid = root
use chroot = no
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log
port = 873
read only = no
[qianxiang]
path = /data/qianxiang/web/
ignore errors
auth users = root
secrets file = /etc/rsyncd.secrets
hosts allow = 192.168.1.1
hosts deny = *

[file]
path = /data/cun_web/
ignore errors
auth users = root
secrets file = /etc/rsyncd.secrets
hosts allow = 192.168.1.1
hosts deny = *

[nginx]
path = /etc/nginx/
ignore errors
auth users = root
secrets file = /etc/rsyncd.secrets
hosts allow = 192.168.1.1
hosts deny = *

```

这里配置socket方式传输文件，端口873，[module_test]开始定义一个模块，指定要同步的目录（接收）path，授权用户，密码文件，允许哪台服务器IP同步（发送）等。

经测试，上述配置文件每行后面不能使用`#`来来注释

**/etc/rsyncd.secrets：** 

```
root:passw0rd
```



一行一个用户，用户名:密码。请注意这里的用户名和密码与操作系统的用户名密码无关，可以随意指定，与`/etc/rsyncd.conf`中的`auth users`对应。

修改权限：`chmod 600 /etc/rsyncd.d/rsync_server.pwd`。

### 服务器启动rsync后台服务

修改`/etc/xinetd.d/rsync`文件，disable 改为 no

```
# default: off
# description: The rsync server is a good addition to an ftp server, as it \
#	allows crc checksumming etc.
service rsync
{
	disable	= no
	flags		= IPv6
	socket_type     = stream
	wait            = no
	user            = root
	server          = /usr/bin/rsync
	server_args     = --daemon
	log_on_failure  += USERID
}
```



执行`service xinetd restart`会一起重启rsync后台进程，默认使用配置文件`/etc/rsyncd.conf`。也可以使用`/usr/bin/rsync --daemon --config=/etc/rsyncd.conf`，重启建议`pkill rsync && sleep 1 && /usr/bin/rsync --daemon --config=/etc/rsyncd.conf  ` 

为了以防rsync写入过多的无用日志到`/var/log/message`（容易塞满从而错过重要的信息），建议注释掉`/etc/xinetd.conf`的success：

```
# log_on_success  = PID HOST DURATION EXIT
```



如果使用了防火墙，要添加允许IP到873端口的规则。

```
# iptables -A INPUT -p tcp -m state --state NEW  -m tcp --dport 873 -j ACCEPT
# iptables -L  查看一下防火墙是不是打开了 873端口
# netstat -anp|grep 873
```



建议关闭`selinux`，可能会由于强访问控制导致同步报错

### 客户端测试同步

单向同步时，客户端只需要一个包含密码的文件。
**/etc/rsync_client.pwd：**

```
passw0rd
```



chmod 600 /etc/rsync_client.pwd

**命令：**
将本地`/root/`目录同步到远程192.168.1.1的/tmp/rsync_bak2目录（module_test指定）：

```
/usr/bin/rsync -auvrtzopgP --progress --password-file=/etc/rsync_client.pwd /root/ sean@192.168.1.1::file
```



当然你也可以将远程的/tmp/rsync_bak2目录同步到本地目录/root/tmp：

```
/usr/bin/rsync -auvrtzopgP --progress --password-file=/etc/rsync_client.pwd sean@192.168.1.1::filet /root/
```

从上面两个命令可以看到，其实这里的服务器与客户端的概念是很模糊的，rsync daemon都运行在远程172.29.88.223上，第一条命令是本地主动推送目录到远程，远程服务器是用来备份的；第二条命令是本地主动向远程索取文件，本地服务器用来备份，也可以认为是本地服务器恢复的一个过程。



# inotify-tools

## 什么是inotify

inotify是一种强大的、细粒度的、异步的文件系统事件监控机制，Linux内核从2.6.13开始引入，允许监控程序打开一个独立文件描述符，并针对事件集监控一个或者多个文件，例如打开、关闭、移动/重命名、删除、创建或者改变属性。

CentOS6自然已经支持：
使用`ll /proc/sys/fs/inotify`命令，是否有以下三条信息输出，如果没有表示不支持。

```
total 0
-rw-r--r-- 1 root root 0 Dec 11 15:23 max_queued_events
-rw-r--r-- 1 root root 0 Dec 11 15:23 max_user_instances
-rw-r--r-- 1 root root 0 Dec 11 15:23 max_user_watches
```



- `/proc/sys/fs/inotify/max_queued_evnets`表示调用inotify_init时分配给inotify instance中可排队的event的数目的最大值，超出这个值的事件被丢弃，但会触发IN_Q_OVERFLOW事件。
- `/proc/sys/fs/inotify/max_user_instances`表示每一个real user ID可创建的inotify instatnces的数量上限。
- `/proc/sys/fs/inotify/max_user_watches`表示每个inotify instatnces可监控的最大目录数量。如果监控的文件数目巨大，需要根据情况，适当增加此值的大小。

## 安装

**inotify-tools：**

inotify-tools是为linux下inotify文件监控工具提供的一套C的开发接口库函数，同时还提供了一系列的命令行工具，这些工具可以用来监控文件系统的事件。 inotify-tools是用c编写的，除了要求内核支持inotify外，不依赖于其他。inotify-tools提供两种工具，一是`inotifywait`，它是用来监控文件或目录的变化，二是`inotifywatch`，它是用来统计文件系统访问的次数。

下载inotify-tools-3.14-1.el6.x86_64.rpm，通过rpm包安装：

```
$ rpm -ivh inotify-tools-3.14-1.el6.x86_64.rpm 
```

## inotifywait使用示例

监控/root/tmp目录文件的变化：

```
/usr/bin/inotifywait -mrq --timefmt '%Y/%m/%d-%H:%M:%S' --format '%T %w %f' \
 -e modify,delete,create,move,attrib /root/tmp/
```

上面的命令表示，持续监听`/root/tmp`目录及其子目录的文件变化，监听事件包括文件被修改、删除、创建、移动、属性更改，显示到屏幕。执行完上面的命令后，在`/root/tmp`下创建或修改文件都会有信息输出：



## 创建排除在外不同步的文件列表

排除不需要同步的文件或目录有两种做法，第一种是inotify监控整个目录，在rsync中加入排除选项，简单；第二种是inotify排除部分不监控的目录，同时rsync中也要加入排除选项，可以减少不必要的网络带宽和CPU消耗。我们选择第二种。

### inotifywait排除

这个操作在客户端进行，假设`/tmp/src/mail/2014/`以及`/tmp/src/mail/2015/cache/`目录下的所有文件不用同步，所以不需要监控，`/tmp/src/`下的其他文件和目录都同步。（其实对于打开的临时文件，可以不监听`modify`时间而改成监听`close_write`）

inotifywait排除监控目录有`--exclude <pattern>`和`--fromfile <file>`两种格式，并且可以同时使用，但主要前者可以用正则，而后者只能是具体的目录或文件。

```
# vi /etc/inotify_exclude.lst：
/tmp/src/pdf
@/tmp/src/2014
```



使用`fromfile`格式只能用绝对路径，不能使用诸如`*`正则表达式去匹配，`@`表示排除。

如果要排除的格式比较复杂，必须使用正则，那只能在`inotifywait`中加入选项，如`--exclude '(.*/*\.log|.*/*\.swp)$|^/tmp/src/mail/(2014|201.*/cache.*)'`，表示排除/tmp/src/mail/以下的2014目录，和所有201*目录下的带cache的文件或目录，以及/tmp/src目录下所有的以.log或.swp结尾的文件。

### rsync排除

使用inotifywait排除监控目录的情况下，必须同时使用rsync排除对应的目录，否则只要有触发同步操作，必然会导致不该同步的目录也会同步。与inotifywait类似，rsync的同步也有`--exclude`和`--exclude-from`两种写法。

个人还是习惯将要排除同步的目录卸载单独的文件列表里，便于管理。使用`--include-from=FILE`时，排除文件列表用绝对路径，但FILE里面的内容请用相对路径，如：
`/etc/rsyncd.d/rsync_exclude.lst`：

```
mail??*
src/*.html*
src/js/
src/ext3/
src/2014/20140[1-9]/
src/201*/201*/201*/.??*
membermail/
membermail??*
membermail/201*/201*/201*/.??*
```

## 客户端同步到远程的脚本`rsync.sh`

下面是一个完整的同步脚本，请根据需要进行裁剪，`rsync.sh`：

```
#variables
current_date=$(date +%Y%m%d_%H%M%S)
source_path=/tmp/src/
log_file=/var/log/rsync_client.log

#rsync
rsync_server=192.168.1.1
rsync_user=sean
rsync_pwd=/etc/rsync_client.pwd
rsync_module=file
INOTIFY_EXCLUDE='(.*/*\.log|.*/*\.swp)$|^/tmp/src/mail/(2014|20.*/.*che.*)'
RSYNC_EXCLUDE='/etc/rsyncd.d/rsync_exclude.lst'

#rsync client pwd check
if [ ! -e ${rsync_pwd} ];then
    echo -e "rsync client passwod file ${rsync_pwd} does not exist!"
    exit 0
fi

#inotify_function
inotify_fun(){
    /usr/bin/inotifywait -mrq --timefmt '%Y/%m/%d-%H:%M:%S' --format '%T %w %f' \
          --exclude ${INOTIFY_EXCLUDE}  -e modify,delete,create,move,attrib ${source_path} \
          | while read file
      do
          /usr/bin/rsync -auvrtzopgP --exclude-from=${RSYNC_EXCLUDE} --progress --bwlimit=1000 --password-file=${rsync_pwd} ${source_path} ${rsync_user}@${rsync_server}::${rsync_module} 
      done
}

#inotify log
inotify_fun >> ${log_file} 2>&1 &
```



`--bwlimit=1000`用于限制传输速率最大1000kb，因为在实际应用中发现如果不做速率限制，会导致巨大的CPU消耗。

在客户端运行脚本`# ./rsync.sh`即可实时同步目录.
