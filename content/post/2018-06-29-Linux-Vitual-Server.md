---
title: Linux Vitual Server
date: 2018-06-29 16:59:32
urlname: lvs
tags: 
- Linux
- server
- internet
categories: LVS
---

摘要：

- 集群概念
- LVS介绍
- LVS实现
- ldirectord

# Cluster概念

- 系统扩展方式：
  Scale UP：向上扩展,增强
  Scale Out：向外扩展,增加设备，调度分配问题，Cluster
- Cluster：集群,为解决某个特定问题将多台计算机组合起来形成的单个系统
- Linux Cluster类型：
  - LB：Load Balancing，负载均衡
  - HA：High Availiablity，高可用，SPOF（single Point Of failure）
    > MTBF:Mean Time Between Failure 平均无故障时间
    > MTTR:Mean Time To Restoration（ repair）平均恢复前时间
    > A=MTBF/（MTBF+MTTR） (0,1)：99%, 99.5%, 99.9%, 99.99%, 99.999%
  - HPC：High-performance computing，高性能 www.top500.org
- 分布式系统：
  分布式存储：云盘,fastdfs
  分布式计算：hadoop，Spark

![1530261982946](http://pic.fenghong.tech/1530261982946.png)

## cluster 分类

- 基于工作的协议层次划分：

  >传输层（通用）：DPORT
  >LVS：
  >nginx：stream
  >haproxy：mode tcp
  >
  >应用层（专用）：针对特定协议，自定义的请求模型分类
  >
  >proxy server：
  >http：nginx, httpd, haproxy(mode http), ...
  >fastcgi：nginx, httpd, ...
  >mysql：mysql-proxy, ...

## cluster 相关

- 会话保持：负载均衡

```
(1) session sticky：始终将同一个请求连接定向至同一个RS，同一用户调度固定服务器
Source IP：LVS sh算法（对某一特定服务而言）
Cookie
(2) session replication：每台服务器拥有全部session,因此对于大规模集群环境不适用
session multicast cluster
(3) session server：专门的session服务器,利用单独部署的服务器来统一管理session。
Memcached，Redis
```
-  HA集群实现方案

```
keepalived:vrrp协议
ais:应用接口规范
heartbeat
cman+rgmanager(RHCS)
coresync_pacemaker
```
# LVS

- LVS：Linux Virtual Server，负载调度器，集成于内核，[章文嵩](https://baike.baidu.com/item/%E7%AB%A0%E6%96%87%E5%B5%A9)博士开发


官网：[http://www.linuxvirtualserver.org/](http://www.linuxvirtualserver.org/)
```
VS: Virtual Server，负责调度
RS: Real Server，负责真正提供服务
L4：四层路由器或交换机
```
-  工作原理：VS根据请求报文的目标IP和目标协议及端口将其调度转发至某RS，根据调度算法来挑选RS

-  iptables/netfilter：

```
iptables：用户空间的管理工具
netfilter：内核空间上的框架
流入：PREROUTING --> INPUT
流出：OUTPUT --> POSTROUTING
转发： PREROUTING --> FORWARD --> POSTROUTING
DNAT：目标地址转换； PREROUTING
```
![1530261982946](http://pic.fenghong.tech/LVS_Cluster.png)

- lvs集群类型中的术语：
```
VS：Virtual Server，Director Server(DS)
Dispatcher(调度器)，Load Balancer
RS：Real Server(lvs), upstream server(nginx)backend server(haproxy)
CIP：Client IP
VIP: Virtual serve IP  VS外网的IP
DIP: Director IP VS内网的IP
RIP: Real server IP

访问流程：CIP <--> VIP == DIP <--> RIP
```
- lvs: ipvsadm/ipvs

> ipvsadm：用户空间的命令行工具，规则管理器，用于管理集群服务及RealServer
>
> ipvs：工作于内核空间netfilter的INPUT钩子上的框架

- lvs集群的类型：

> lvs-nat：修改请求报文的目标IP,多目标IP的DNAT
>
> lvs-dr：操纵封装新的MAC地址
>
> lvs-tun：在原请求IP报文之外新加一个IP首部
>
> lvs-fullnat：修改请求报文的源和目标IP


## lvs-nat：
本质是多目标IP的DNAT，通过将请求报文中的目标地址和目标端口修改为某挑出的RS的RIP和PORT实现转发
1. RIP和DIP应在同一个IP网络，最好使用私网地址；RS的网关要指向DIP
2. 请求报文和响应报文都必须经由Director转发，Director易于成为系统瓶颈
3. 支持端口映射，可修改请求报文的目标PORT
4. VS必须是Linux系统，RS可以是任意OS系统

![1530265255549](http://pic.fenghong.tech/1530265255549.png)

## LVS-DR

 LVS-DR：Direct Routing，直接路由，LVS默认模式,应用最广泛,通过为请求报文重新
封装一个MAC首部进行转发，源MAC是DIP所在的接口的MAC，目标MAC是某挑选出
的RS的RIP所在接口的MAC地址；源IP/PORT，以及目标IP/PORT均保持不变
1. Director和各RS都配置有VIP
2. 确保前端路由器将目标IP为VIP的请求报文发往Director
```
1.在前端网关做静态绑定VIP和Director的MAC地址
2.在RS上使用arptables工具
arptables -A IN -d $VIP -j DROP
arptables -A OUT -s $VIP -j mangle --mangle-ip-s $RIP
3.在RS上修改内核参数以限制arp通告及应答级别 
/proc/sys/net/ipv4/conf/all/arp_ignore
/proc/sys/net/ipv4/conf/all/arp_announce
]# echo net.ipv4.conf.all.arp_ignore = 1 >> /etc/sysctl.conf
]# echo net.ipv4.conf.all.arp_announce = 2 >> /etc/sysctl.conf
]# sysctl -p
推荐修改内核配置，数字说明。
    arp_announce :
   0：默认值，把本机所有接口的所有信息向每个接口的网络进行通告
   1：尽量避免将接口信息向非直接连接网络进行通告
   2：必须避免将接口信息向非本网络进行通告
    arp_ignore:
    0：默认值，表示可使用本地任意接口上配置的任意地址进行响应
    1: 仅在请求的目标IP配置在本地主机的接收到请求报文的接口上时，才给予响应
```
3. RS的RIP可以使用私网地址，也可以是公网地址；RIP与DIP在同一IP网络；RIP的网关不能指向DIP，以确保响应报文不会经由Director
4. RS和Director要在同一个物理网络
5. 请求报文要经由Director，但响应报文不经由Director，而由RS直接发往Client
6. 不支持端口映射（端口不能修败）
7. RS可使用大多数OS系统

![LVS_dr](http://pic.fenghong.tech/LVS_dr.png)

## lvs-tun

转发方式：不修改请求报文的IP首部（源IP为CIP，目标IP为VIP），而在原IP报文之外再封装一个IP首部（源IP是DIP，目标IP是RIP），将报文发往挑选出的目标RS；RS直接响应给客户端（源IP是VIP，目标IP是CIP）
> 1. DIP, VIP, RIP都应该是公网地址
> 2. RS的网关一般不能指向DIP
> 3. 请求报文要经由Director，但响应不能经由Director
> 4. 不支持端口映射
> 5. RS的OS须支持隧道功能

## lvs调度算法

lvs的调度方法：10种

- 静态方法：仅根据算法本身进行调度

> RR：Round Robin
> 
> WRR: Weigted RR
> 
> SH: Source Hashing，实现session sticky，源IP地址hash
> 
> DH:Destination Hashing，

- 动态方法:主要根据每RS当前的负载状态及调度算法进行调度Overhead=value，较小的RS将被调度

> LC： Least Connection
> 
> ​	Overhead= Active*256+Inactive
> 
> WLC: Weigted LC，默认调度方法
> 
> ​	Overhead=(Active*256+Inactive)/Weight
> 
> SED: Shortest Expect Delay
> 
> ​	Overhead=(Active+1)*256/weight
> 
> NQ：Never Queue，第一轮均匀分配，后续SED
> 
> LBLC：Locality-Based LC，动态的DH算法，使用场景：根据负载状态实现正向代理
> 
> LBLCR：LBLC with Replication，带复制功能的LBLC解决LBLC负载不均衡问题，从负载重的复制到负载轻的RS

## 总结

- lvs-nat与lvs-fullnat：请求和响应报文都经由Director

lvs-nat：RIP的网关要指向DIP
lvs-fullnat：RIP和DIP未必在同一IP网络，但要能通信

- lvs-dr与lvs-tun：请求报文要经由Director，但响应报文由RS直接发往Client

lvs-dr：通过封装新的MAC首部实现，通过MAC网络转发
lvs-tun：通过在原IP报文外封装新IP头实现转发，支持远距离通信

