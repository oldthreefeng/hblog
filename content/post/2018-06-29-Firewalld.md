---
title: firewalld
date: 2018-06-29 15:59:32
urlname: firewalld
tags: 
- Linux
- server
- Safe
categories: internet
---
摘要：

- firewalld介绍
- firewalld配置命令
- rich规则
- 伪装及端口转发

# FireWalld

- firewalld是CentOS 7.0新推出的管理netfilter的工具
- firewalld是配置和监控防火墙规则的系统守护进程。可以实现iptables,ip6tables,ebtables的功能

- firewalld服务由firewalld包提供
- firewalld支持划分区域zone,每个zone可以设置独立的防火墙规则
- 归入zone顺序：
  - 先根据数据包中源地址，将其纳为某个zone
  - 纳为网络接口所属zone
  - 纳入默认zone，默认为public zone,管理员可以改为其它zone
- 网卡默认属于public zone,lo网络接口属于trusted zone

![1530369127584](http://pic.fenghong.tech/1530369127584.png)

## Firewalld配置

- `firewall-cmd --get-services` 查看预定义服务列表
- `/usr/lib/firewalld/services/*.xml`预定义服务的配置
- 三种配置方法

> firewall-config （firewall-config包）图形工具
> firewall-cmd （firewalld包）命令行工具
> /etc/firewalld 配置文件，一般不建议

## firewall-cmd 命令选项

```
]# yum install firewalld -y
]# systemctl restart firewalld
```

- 参数说明：

```
--get-zones  列出所有可用区域
--get-default-zone  查询默认区域
--set-default-zone=<ZONE> 设置默认区域
--get-active-zones  列出当前正使用的区域
--add-source=<CIDR>[--zone=<ZONE>] 添加源地址的流量到指定区域，如果无--zone= 选项，使用默认区域
--remove-source=<CIDR> [--zone=<ZONE>] 从指定区域中删除源地址的流量，如无--zone= 选项，使用默认区域
--add-interface=<INTERFACE>[--zone=<ZONE>]  添加来自于指定接口的流量到特定区域，如果无--zone= 选项，使用默认区域
--change-interface=<INTERFACE>[--zone=<ZONE>] 改变指定接口至新的区域，如果无--zone= 选项，使用默认区域
--add-service=<SERVICE> [--zone=<ZONE>] 允许服务的流量通过，如果无--zone= 选项，使用默认区域
--add-port=<PORT/PROTOCOL>[--zone=<ZONE>] 允许指定端口和协议的流量，如果无--zone= 选项，使用默认区域
--remove-service=<SERVICE> [--zone=<ZONE>]  从区域中删除指定服务，禁止该服务流量，如果无--zone= 选项，使用默认区域
--remove-port=<PORT/PROTOCOL>[--zone=<ZONE>]  从区域中删除指定端口和协议，禁止该端口的流量，如果无--zone= 选项，使用默认区域
--reload 删除当前运行时配置，应用加载永久配置
--list-services 查看开放的服务
--list-ports 查看开放的端口
--list-all [--zone=<ZONE>] 列出指定区域的所有配置信息，包括接口，源地址，端口，服务等，如果无--zone= 选项，使用默认区域
```
### 示例

- 查看默认zone
```
firewall-cmd --get-default-zone
```
- 默认zone设为dmz
```
firewall-cmd --set-default-zone=dmz
```
- 在internal zone中增加源地址192.168.0.0/24的永久规则
```
firewall-cmd --permanent --zone=internal --add-source=192.168.1.0/24
```
- 在internal zone中增加协议mysql的永久规则
```
firewall-cmd --permanent –zone=internal --add-service=mysql
```
- 加载新规则以生效
```
firewall-cmd --reload
```
## rich-rules规则

- 当基本firewalld语法规则不能满足要求时，可以使用以下更复杂的规则
- rich-rules 富规则，功能强,表达性语言
- Direct configuration rules 直接规则，灵活性差。帮助：`man 5 firewalld.direct`
- rich规则比基本的firewalld语法实现更强的功能，不仅实现允许/拒绝，还可以实现日志syslog和auditd，也可以实现端口转发，伪装和限制速率
- rich语法

```
rule
[source]
[destination]
service|port|protocol|icmp-block|masquerade|forward-port
[log]
[audit]
[accept|reject|drop]
```

- 基本用法
```
--add-rich-rule='<RULE>'  
Add <RULE> to the specified zone, or the default zone if no zone is specified.
--remove-rich-rule='<RULE>'  
Remove <RULE> to the specified zone, or the default zone if no zone is specified.
--query-rich-rule='<RULE>'  
Query if <RULE> has been added to the specified zone, or the default zone ifno zone is specified. Returns 0 if the rule is present, otherwise 1. 
--list-rich-rules  
Outputs all rich rules for the specified zone, or the default zone if no zone isspecified.
```
- rich规则示例

1. 拒绝从192.168.0.11的所有流量，当address 选项使用source 或 destination时，必须用family= ipv4 |ipv6.
```
]# firewall-cmd --permanent --zone=classroom --add-rich-rule='rule family=ipv4 source address=192.168.0.11/32 reject‘
```
2. 限制每分钟只有两个连接到ftp服务
```
firewall-cmd --add-rich-rule=‘rule service name=ftp limit value=2/m accept’
```
3. 抛弃esp（ IPsec 体系中的一种主要协议）协议的所有数据包
```
firewall-cmd --permanent --add-rich-rule='rule protocol value=esp drop'
```
4. 接受所有192.168.1.0/24子网端口5900-5905范围的TCP流量
```
]# firewall-cmd --permanent --zone=vnc --add-rich-rule='rule family=ipv4 source address=192.168.1.0/24 port port=5900-5905 protocol=tcp accept'
```
5. 接受ssh新连接，记录日志到syslog的notice级别，每分钟最多三条信息
```
]# firewall-cmd --permanent --zone=work --add-rich-rule='rule servicename="ssh" log prefix="ssh " level="notice" limit value="3/m" accept
```
6. 从2001:db8::/64子网的DNS连接在5分钟内被拒绝，并记录到日志到audit,每小时最大记录一条信息
```
]# firewall-cmd --add-rich-rule='rule family=ipv6 source address="2001:db8::/64" service name="dns" audit limit value="1/h" reject' --timeout=300
```
## 伪装和端口转发

NAT网络地址转换，firewalld支持伪装和端口转发两种NAT方式

- 伪装NAT

```
firewall-cmd --permanent --zone= <ZONE> --add-masquerade
firewall-cmd --query-masquerade 检查是否允许伪装
firewall-cmd --add-masquerade 允许防火墙伪装IP
firewall-cmd --remove-masquerade 禁止防火墙伪装IP
```

- 示例：

```
firewall-cmd --add-rich-rule='rule family=ipv4 source address=192.168.0.0/24 masquerade'
```

- 端口转发

将发往本机的特定端口的流量转发到本机或不同机器的另一个端口。通常要配合地址伪装才能实现

命令行如下：

```
firewall-cmd --permanent --zone= <ZONE> --add-forward-port=port= <PORTNUMBER> :proto= <PROTOCOL> [:toport= <PORTNUMBER> ][:toaddr=  ]
说明：toport= 和toaddr= 至少要指定一个
```
- 示例：
  转发传入的连接9527/TCP，到防火墙的80/TCP到public zone 的192.168.0.254
```
  firewall-cmd --add-masquerade 启用伪装
  firewall-cmd --zone=public --add-forward-port=port=9527:proto=tcp:toport=80:toaddr=192.168.0.254
```
