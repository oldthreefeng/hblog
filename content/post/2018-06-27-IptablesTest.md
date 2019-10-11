---
title: IptablesTest
date: 2018-06-27 15:59:32
urlname: iptables1
tags: 
- Linux
- server
- Safe
categories: internet
---

摘要：iptabels练习题



# 习题

- 说明：以下练习INPUT和OUTPUT默认策略均为DROP

1. 限制本地主机的web服务器在周一不允许访问；新请求的速率不能超过100个每秒；web服务器包含了admin字符串的页面不允许访问；web服务器仅允许响应报文离开本机
2. 在工作时间，即周一到周五的8:30-18:00，开放本机的ftp服务给172.20.0.0网络中的主机访问；数据下载请求的次数每分钟不得超过5个
3. 开放本机的ssh服务给172.20.x.1-172.20.x.100中的主机，x为你的学号，新请求建立的速率一分钟不得超过2个；仅允许响应报文通过其服务端口离开本机
4. 拒绝TCP标志位全部为1及全部为0的报文访问本机；
5. 允许本机ping别的主机；但不开放别的主机ping本机

# 答案
## one

- 限制本地主机的web服务器在周一不允许访问；新请求的速率不能超过100个每秒；web服务器包含了admin字符串的页面不允许访问；web服务器仅允许响应报文离开本机


```
#周一不允许访问：time扩展
#新请求的速率不能超过100个每秒：state扩展+limit扩展
#admin字符串的页面不允许访问：string扩展
#仅允许响应报文离开本机：tcp：80为源端口
#把正在连接的ssh设为接受
iptables -A INPUT -p tcp --dport 22 -s 172.20.0.16 -j ACCEPT
iptables -A OUTPUT  -p tcp --sport 22 -d 172.20.0.16 -j ACCEPT 
#新建一个链，负责有关web服务
iptables -N web1
#1.周一不允许访问：centos7默认UTC时间
iptables -A web1 -p tcp --dport 80  -m time  --weekdays 1 -j REJECT
#(北京时间） 
iptables -A web1 -p tcp --dport 80  -m time --weekdays 7 --timestart 16:00  \
--timestop 23:59:59 -m time --weekdays 1 --timestart 00:00 --timestop 16:00  -j REJECT
#2.新请求的速率不能超过100个每秒
iptables -A web1 -p tcp --dport 80  -m state --state NEW \
-m limit --limit 100/second  -j REJECT
#3.admin字符串的页面不允许访问
iptables -A OUTPUT -p tcp --sport 80  -m string --algo bm --string "admin" -j REJECT
#4.仅允许响应报文离开本机
iptables -A OUTPUT -p tcp --sport 80  -j ACCEPT  
#5.将新建的链加入到INPUT链 
iptables -A INPUT -j web1
#6.其余关于INPUT和OUTPUT的全部DROP
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
```

## two

- 在工作时间，即周一到周五的8:30-18:00，开放本机的ftp服务给172.20.0.0网络中的主机访问；数据下载请求的次数每分钟不得超过5个

```
#允许正在链接的用户
iptables -I INPUT  -p tcp --dport 22 -s 172.20.0.16 -j ACCEPT 
iptables -I OUTPUT  -p tcp --sport 22 -d 172.20.0.16 -j ACCEPT
#新建链
ipytables -N ftpi
#ftp为多端口服务，首先把state状态的相关包和端口打开
iptables -A ftpi -m state --state ESTABLISHED -m state --state RELATED -j ACCEPT
#周一到周五的8:30-18:00，开放ftp服务给172.20.0.0：tcp:21端口
iptables -A ftpi -s 172.20.0.0/16 -p tcp --dport 21 -m time \
--timestart 00:30 --timestop 10:00  --weekdays 1,2,3,4,5  -j ACCEPT 
#数据下载请求的次数每分钟不得超过5个
#应该放到state状态的相关包和端口打开的前面
iptables -I ftpi -p tcp --dport 21 -m state --state RELATED \ 
-m string --algo bm --string "get"  -m limit  --limit 5/minute -j ACCEPT
#把链接放到INPUT中
iptables -A INPUT -j ftpi
#剩余全丢弃 
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
```

## three

- 开放本机的ssh服务给172.20.x.1-172.20.x.100中的主机，x为你的学号，新请求建立的速率一分钟不得超过2个；仅允许响应报文通过其服务端口离开本机

```
#允许正在链接的用户
iptables -I INPUT  -p tcp --dport 22 -s 172.20.0.16 -j ACCEPT 
iptables -I OUTPUT  -p tcp --sport 22 -d 172.20.0.16 -j ACCEPT
#新建链sshi
iptables -N sshi
#ssh服务给172.20.0.1-172.20.0.100中的主机，新请求建立的速率一分钟不得超过2个
iptables -A sshi  -m iprange --src-rang 172.20.0.1-172.20.0.100 -p tcp \
--dport 22 -m state --state NEW -m limit --limit 2/minute -j ACCEPT
#除了新建立链接，别的指定IP ssh 链接都可以通过
iptables -A sshi  -m iprange --src-rang 172.20.0.1-172.20.0.100 \
-p tcp --dport 22  -j ACCEPT 
#新建链
 iptables -N ssho   
#仅允许响应报文通过其服务端口离开本机
iptables -A ssho -m iprange --dst-rang 172.20.0.1-172.20.0.100 \
-p tcp --sport 22 -j ACCEPT
#将链接加入INPUT和OUTPUT中
iptables -I INPUT -j sshi
iptables -I OUTPUT -j ssho  
#剩余全丢弃
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
```

## four

- 拒绝TCP标志位全部为1及全部为0的报文访问本机；

```
#允许正在链接的用户
iptables -I INPUT  -p tcp --dport 22 -s 172.20.0.16 -j ACCEPT 
iptables -I OUTPUT  -p tcp --sport 22 -d 172.20.0.16 -j ACCEPT
#新建链tcpi
iptables -N tcpi
iptables -A tcpi -p tcp  --tcp-flags ALL ALL -j REJECT
iptables -A tcpi -p tcp  --tcp-flags ALL NONE -j REJECT    
#允许正在链接的用户
iptables -I INPUT -j tcpi
#剩余全丢弃
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
```

## five

- 允许本机ping别的主机；但不开放别的主机ping本机

```
#允许正在链接的用户
iptables -I INPUT  -p tcp --dport 22 -s 172.20.0.16 -j ACCEPT 
iptables -I OUTPUT  -p tcp --sport 22 -d 172.20.0.16 -j ACCEPT
#允许OUTPUT发出请求包
iptables -I OUTPUT -p icmp --icmp-type 8/0 -j ACCEPT
#允许INPUT接受回应包
iptables -I INPUT -p icmp --icmp-type 0/0 -j ACCEPT
#拒绝INPUT的请求包 
iptables -I INPUT -p icmp --icmp-type 8/0 -j REJECT  
#剩余全丢弃
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
```

