---
title: "RabbitMq高可用集群搭建"
date: 2019-11-06T17:54:18+08:00
tags: [ansible,rabbitmq,cluster,haproxy]
categories: [server]
---

> RabbitMQ这款消息队列中间件产品本身是基于Erlang编写，Erlang语言天生具备分布式特性（通过同步Erlang集群各节点的magic cookie来实现）。因此，RabbitMQ天然支持Clustering。

`RabbitMq`依赖环境及安装一览, `$ 表示shell , # 表示注释, > 表示mysql` 

```bash
$ rabbitmqctl --version
3.8.1

$ erl --version
Erlang/OTP 22 

$ java -version
java version "1.8.0_111"
Java(TM) SE Runtime Environment (build 1.8.0_111-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)
```

### 初始化环境

分别修改主机名

```shell script
$ hostnamectl set-hostname c1 ## c1 作为当前主机名.
```

如果 DNS 不支持解析主机名称，则需要修改每台机器的 `/etc/hosts` 文件

```bash
$ cat >> /etc/hosts <<EOF
192.168.133.133 c4
192.168.133.128	c1
192.168.133.129 c2
192.168.133.130 c3
EOF
```

解除linux系统最大进程数和最大文件打开数

```bash 
$ cat >> /etc/security/limits.conf << EOF
*   soft noproc   65535  
*   hard noproc   65535  
*   soft nofile   65535  
*   hard nofile   65535
EOF

$ cat >> /etc/profile.d/limits.sh <<EOF
ulimit -u 65535        ## max user processes
ulimit -n 65535        ## open files
ulimit -d unlimited    ## data seg size
ulimit -m unlimited    ## max memory size
ulimit -s unlimited    ## stack size
ulimit -t unlimited    ## cpu time
ulimit -v unlimited    ## virtual memory
EOF

$ source /etc/profile.d/limits.sh
```

习惯性做完互信工作,  习惯性禁用防火墙(生产环境自行使用`iptables`). c1主机为`ansible`管理主机,详细查询ansible[的这篇文章](https://blog.fenghong.tech/post/2018-06-10-ansible/)

```bash
$ ssh-keygen
$ ssh-copy-id localhost
$ for i in c1 c2 c3 c4 ;do scp -r /root/.ssh $i:/root/.ssh ;done
```

关闭 SELinux

```
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

防火墙配置 (选择做, 楼主实验用的策略是`iptables -F`, 生产环境建议如下)

```bash
$ firewall-cmd --permanent --add-port=25672/tcp
$ firewall-cmd --permanent --add-port=15672/tcp
$ firewall-cmd --permanent --add-port=5672/tcp
$ firewall-cmd --permanent --add-port=4369/tcp
$ systemctl restart firewalld.service
```

### 基于`ansible`安装

在c1主机上, 下载相关的tar文件

```bash
 $ cd /data/
 $ wget http://erlang.org/download/otp_src_22.1.tar.gz 
 $ wget https://dl.bintray.com/rabbitmq/all/rabbitmq-server/3.8.1/rabbitmq-server-generic-unix-3.8.1.tar.xz
```

`palybook`安装`erlang`及`RabbitMq`

```yaml
---
- hosts: all
  remote_user: root
  tasks:
  - name: mkdir
    file:
      path: /data
      state: directory
      mode: '0755'
  - name: cp
    copy:
      src: /data/rabbitmq-server-generic-unix-3.8.1.tar.xz
      dest: /data/rabbitmq-server-generic-unix-3.8.1.tar.xz
  - name: tar xf
    shell: "cd /data && tar xf rabbitmq-server-generic-unix-3.8.1.tar.xz  -C /usr/local/"
  - name: mkdir
    file:
      path: /data
      state: directory
      mode: '0755'
  - name: cp
    copy:
      src: /data/otp_src_22.1.tar.gz
      dest: /data/otp_src_22.1.tar.gz
  - name: tar xf make install 
    shell: "cd /data && tar xf otp_src_22.1.tar.gz && cd otp_src_22.1 && ./configure --prefix=/usr/local/bin/erlang --without-javac && make && make install"
```

不重启系统,生效环境变量

执行以下命令, 若返回`Eshell`,说明安装成功

```
$ ansible all -m shell -a  "echo export PATH=$PATH:/usr/local/bin/erlang/bin:/usr/local/rabbitmq_server-3.8.1/sbin >> /etc/profile"

$ ansible all -m shell -a "source /etc/profile && erl --version "
192.168.133.133 | SUCCESS | rc=0 >>
Eshell V10.5  (abort with ^G)
1> *** Terminating erlang (nonode@nohost)

192.168.133.130 | SUCCESS | rc=0 >>
Eshell V10.5  (abort with ^G)
1> *** Terminating erlang (nonode@nohost)

192.168.133.129 | SUCCESS | rc=0 >>
Eshell V10.5  (abort with ^G)
1> *** Terminating erlang (nonode@nohost)

192.168.133.128 | SUCCESS | rc=0 >>
Eshell V10.5  (abort with ^G)
1> *** Terminating erlang (nonode@nohost)
```

### 启动`RabbitMq`

各节点启动`rabbitmq`, 出现以下状态, 说明启动成功.

编辑每台`RabbitMQ`的cookie文件，以确保各个节点的cookie文件使用的是同一个值,可以scp其中一台机器上的cookie至其他各个节点，cookie的默认路径为`/var/lib/rabbitmq/.erlang.cookie`或者`$HOME/.erlang.cookie`，节点之间通过cookie确定相互是否可通信.

```bash
$ rabbitmq-server -deched

  ##  ##      RabbitMQ 3.8.1
  ##  ##
  ##########  Copyright (c) 2007-2019 Pivotal Software, Inc.
  ######  ##
  ##########  Licensed under the MPL 1.1. Website: https://rabbitmq.com

  Starting broker... completed with 0 plugins.  (可以Ctrl+C的, 已经后台运行了.)

$ for i in c1 c2 c3 c4 ;do scp  /root/.erlang.cookie $i:/root/ ;done
```

查看节点情况

```bash
$ rabbitmqctl status -n rabbit@c1
$ rabbitmqctl status -n rabbit@c2
$ rabbitmqctl status -n rabbit@c3
$ rabbitmqctl status -n rabbit@c4
```

在`RabbitMQ`集群中的节点只有两种类型：内存节点/磁盘节点，单节点系统只运行磁盘类型的节点。而在集群中，可以选择配置部分节点为内存节点。
 内存节点将所有的队列，交换器，绑定关系，用户，权限，和`vhost`的元数据信息保存在内存中。而磁盘节点将这些信息保存在磁盘中，但是内存节点的性能更高，为了保证集群的高可用性，必须保证集群中有两个以上的磁盘节点，来保证当有一个磁盘节点崩溃了，集群还能对外提供访问服务.

以c1为主节点, 在c1上执行,若以内存节点加入, 则在join_cluster的时候加上`--ram`

```bash
$ rabbitmqctl stop_app 
$ rabbitmqctl reset 
$ rabbitmqctl join_cluster rabbit@c2  ## 此时是以磁盘节点加入的
$ rabbitmqctl start_app 

#加入时候设置节点为内存节点（默认加入的为磁盘节点）
$ rabbitmqctl join_cluster rabbit@c1 --ram
```

#### 查看`rabbitMq`集群状态

```bash
$ rabbitmqctl cluster_status
Cluster name: rabbit@c1
Disk Nodes
	rabbit@c3
	rabbit@c4
RAM Nodes
	rabbit@c1
	rabbit@c2
Running Nodes
	rabbit@c1
	rabbit@c2
	rabbit@c3
	rabbit@c4
...
```

#### `rabbitMq`插件安装

```bash
## list 
$ rabbitmq-plugins list -v  
## list plugins whose name contains "management"
$ rabbitmq-plugins list -v management  
## install 
$ rabbitmq-plugins enable rabbitmq_management  
```

到这里,` rabbitMq`基本已经部署完毕. 

### 使用`haproxy`负载均衡`RabbitMq`

 对于消息的生产和消费者可以通过`HAProxy`的软负载将请求分发至`RabbitMQ`集群中的`Node1～Node2`节点，其中`Node3～Node4`的两个节点作为磁盘节点保存集群元数据和配置信息 .

安装`HaProxy`

```bash
$ yum install haproxy
$ vi ha.cfg
global
        #日志输出配置，所有日志都记录在本机，通过local0输出
        log 127.0.0.1 local0 info
        #最大连接数
        maxconn 4096
        #改变当前的工作目录
        chroot /apps/svr/haproxy
        #以指定的UID运行haproxy进程
        uid 99
        #以指定的GID运行haproxy进程
        gid 99
        #以守护进程方式运行haproxy #debug #quiet
        daemon
        #debug
        #当前进程pid文件
        pidfile /apps/svr/haproxy/haproxy.pid

#默认配置
defaults
        #应用全局的日志配置
        log global
        #默认的模式mode{tcp|http|health}
        #tcp是4层，http是7层，health只返回OK
        mode tcp
        #日志类别tcplog
        option tcplog
        #不记录健康检查日志信息
        option dontlognull
        #3次失败则认为服务不可用
        retries 3
        #每个进程可用的最大连接数
        maxconn 2000
        #连接超时
        timeout connect 5s
        #客户端超时
        timeout client 120s
        #服务端超时
        timeout server 120s

        maxconn 2000
        #连接超时
        timeout connect 5s
        #客户端超时
        timeout client 120s
        #服务端超时
        timeout server 120s

#绑定配置
listen rabbitmq_cluster
        bind 0.0.0.0:5671
        #配置TCP模式
        mode tcp
        #加权轮询
        balance roundrobin
        #RabbitMQ集群节点配置,其中ip1~ip2为RabbitMQ集群节点ip地址
        server rmq_node1 c1:5672 check inter 5000 rise 2 fall 3 weight 1
        server rmq_node2 c2:5672 check inter 5000 rise 2 fall 3 weight 1

#haproxy监控页面地址
listen monitor
        bind 0.0.0.0:8100
        mode http
        option httplog
        stats enable
        stats uri /stats
        stats refresh 5s
```

执行启动命令

```bash
$ haproxy -f ha.cfg
```

访问`http://ip:8100/stats`, 即可查看`haproxy`状态

![test](http://pic.fenghong.tech/rabbitmq/20191106180241.png)



