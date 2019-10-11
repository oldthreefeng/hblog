---
title: KeepAlive
date: 2018-07-11 19:59:32
urlname: keepalive
tags: 
- Linux
- server
- internet
categories: internet
---

摘要：

- HA cluster的概念
- Keepalive的应用及配置
- keepalived+haporxy实验
- ansible+keepalived+nginx实验

# HA cluster

​	HA是High Available缩写，是双机集群系统简称，指高可用性集群，是保证业务连续性的有效解决方案，一般有两个或两个以上的节点，且分为活动节点及备用节点。 

- LB：负载均衡集群

> lvs负载均衡
> nginx反向代理
> HAProxy

- HA：高可用集群

> eartbeat
> eepalived
> edhat5 : cman + rgmanager , conga(WebGUI) –> RHCS（Cluster Suite）集群套件
> edhat6 : cman + rgmanager , corosync + pacemaker
> edhat7 : corosync + pacemaker

- HP：高性能集群

```
 > total/2 with quorum
<= total/2 without quorum
```

- heartbeat:

```
heartbeat:
	heratbeat
	cluster-glye
	pacemaker
corosync + pacemaker  (100个节点集群)
	STONITH: shooting the other node in the head
cman + rgmanager
```

# keepalive

- keepalived的相关概念

```
vrrp协议：Virtual Redundant Routing Protocol 虚拟冗余路由协议
Virtual Router：虚拟路由器
VRID(0-255)：虚拟路由器标识
master：主设备，当前工作的设备
backup：备用设备
priority：优先级，优先级越大优先工作，具体情况示工作方式决定
VIP：虚拟IP地址，正真向客户服务的IP地址
VMAC：虚拟MAC地址(00-00-5e-00-01-VRID)
抢占式：如果有优先级高的节点上线，则将此节点转为master
非抢占式：即使有优先级高的节点上线，在当前master工作无故障的情况运行抢占；等到此master故障后重新按优先级选举master
心跳：master将自己的心跳信息通知集群内的所有主机，证明自己正常工作
安全认证机制：
无认证：任何主机都可成为集群内主机，强烈不推荐
简单的字符认证：使用简单的密码进行认证
AH认证
sync group：同步组，VIP和DIP配置到同一物理服务器上
MULTICAST：组播，多播
Failover：master故障，故障切换，故障转移
Failback：故障节点重新上线，故障切回
```

- keepalived的模型结构如下：

![1531276298776](http://pic.fenghong.tech/1531276298776.png)



## 安装

```
]# yum install -y keepalived
]# rpm -ql keepalived
/etc/keepalived
/etc/keepalived/keepalived.conf  			#主配置文件       
/etc/sysconfig/keepalived				#uint files配置文件		
/usr/bin/genhash
/usr/lib/systemd/system/keepalived.service	#uint files
/usr/libexec/keepalived
/usr/sbin/keepalived			#主程序文件
```

## 配置

需开启`multicast`，基于多播模式。

```
]# ip link set dev ens33 multicast on #基于多播模式
```

- 全局配置段

```
global_defs {
   notification_email {  #发送通知email，收件人
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1  #邮件服务器地址
   smtp_connect_timeout 30  #超时时长
   router_id LVS_DEVEL  #路由器标识ID
   vrrp_skip_check_adv_addr  #跳过的检查地址
   vrrp_strict  #严格模式
   vrrp_garp_interval 0  #免费arp
   vrrp_gna_interval 0
}
```

- 虚拟路由示例段

```
vrrp_instance <STRING> {
    state MASTER|BACKUP：#当前节点在此虚拟路由器上的初始状态；只能有一个是MASTER，余下的都应该为BACKUP；
    interface IFACE_NAME：#绑定为当前虚拟路由器使用的物理接口；
    virtual_router_id VRID：#当前虚拟路由器的惟一标识，范围是0-255；
    priority 100：#当前主机在此虚拟路径器中的优先级；范围1-254；
    advert_int 1：#vrrp通告的时间间隔；
    authentication {
        auth_type AH|PASS #pass为简单认证
        auth_pass <PASSWORD> #认证密码，8为密码
    }
    virtual_ipaddress {  #VIP配置
        <IPADDR>/<MASK> brd <IPADDR> dev <STRING> scope <SCOPE> label <LABEL>
        192.168.200.17/24 dev eth1
        192.168.200.18/24 dev eth2 label eth2:1
    }
    track_interface {  #配置要监控的网络接口，一旦接口出现故障，则转为FAULT状态；
        eth0
        eth1
        ...
    }
    nopreempt：定义工作模式为非抢占模式；
    preempt_delay 300：抢占式模式下，节点上线后触发新选举操作的延迟时长；
    notify_master <STRING>|<QUOTED-STRING>：当前节点成为主节点时触发的脚本；
    notify_backup <STRING>|<QUOTED-STRING>：当前节点转为备节点时触发的脚本；
    notify_fault <STRING>|<QUOTED-STRING>：当前节点转为“失败”状态时触发的脚本；
    notify <STRING>|<QUOTED-STRING>：通用格式的通知触发机制，一个脚本可完成以上三种状态的转换时的通知；
}
```

- 虚拟服务器配置

```
 delay_loop <INT>：服务轮询的时间间隔；
 lb_algo rr|wrr|lc|wlc|lblc|sh|dh：定义调度方法；
 lb_kind NAT|DR|TUN：集群的类型；
 persistence_timeout <INT>：持久连接时长；
 protocol TCP：服务协议，仅支持TCP；
sorry_server <IPADDR> <PORT>：备用服务器地址；
real_server <IPADDR> <PORT>
{
	 weight <INT>
	 notify_up <STRING>|<QUOTED-STRING>
	 notify_down <STRING>|<QUOTED-STRING>
	 HTTP_GET|SSL_GET|TCP_CHECK|SMTP_CHECK|MISC_CHECK { ... }：定义当前主机的健康状态检测方法；
}

HTTP_GET|SSL_GET：应用层检测

HTTP_GET|SSL_GET {
	url {
		path <URL_PATH>：定义要监控的URL；
		status_code <INT>：判断上述检测机制为健康状态的响应码；
		digest <STRING>：判断上述检测机制为健康状态的响应的内容的校验码；
	}
	nb_get_retry <INT>：重试次数；
	delay_before_retry <INT>：重试之前的延迟时长；
	connect_ip <IP ADDRESS>：向当前RS的哪个IP地址发起健康状态检测请求
	connect_port <PORT>：向当前RS的哪个PORT发起健康状态检测请求
	bindto <IP ADDRESS>：发出健康状态检测请求时使用的源地址；
	bind_port <PORT>：发出健康状态检测请求时使用的源端口；
	connect_timeout <INTEGER>：连接请求的超时时长；
}

 TCP_CHECK {
	connect_ip <IP ADDRESS>：向当前RS的哪个IP地址发起健康状态检测请求
	connect_port <PORT>：向当前RS的哪个PORT发起健康状态检测请求
	bindto <IP ADDRESS>：发出健康状态检测请求时使用的源地址；
	bind_port <PORT>：发出健康状态检测请求时使用的源端口；
	connect_timeout <INTEGER>：连接请求的超时时长；
}
```

- 脚本

```
vrrp_script <SCRIPT_NAME> {
    script ""  #定义执行脚本
    interval INT  #多长时间检测一次
    weight -INT  #如果脚本的返回值为假，则执行权重减N的操作
    rise 2  #检测2次为真，则上线
    fall 3  #检测3次为假，则下线
}
vrrp_instance VI_1 {
    track_script {  #在虚拟路由实例中调用此脚本
        SCRIPT_NAME_1
        SCRIPT_NAME_2
        ...
    }
}
```

- `ipvs+keepalive`的配置

```
virtual_server IP port|
virtual_server fwmark int
{
    ···
    real_server{
        ···
    }
}
```

# keepalived+HAproxy

环境：

```
192.168.0.10:80
192.168.0.11:80
192.168.0.12:80
三台web服务器已经搭好
]# yum install -y httpd
]# echo `hostname` > /var/www/html/index.html
]# systemctl start httpd
```

- 配置HAProxy，两台主机一样的配置

```
]# vim /etc/haproxy/haproxy.cfg
frontend web *:80
    default_backend websrvs
backend websrvs
    balance roundrobin
    server srv1 192.168.0.10:80 check
    server srv2 192.168.0.11:80 check
    server srv3 192.168.0.12:80 check
```

- 配置keepalived实现高可用，一台为MASTER一台为BACKUP.

```
]# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from keepalived@localhoat
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id node1
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
   vrrp_mcast_group4 224.0.111.111
   vrrp_iptables
}
vrrp_script chk_haproxy {
        script "killall -0 haproxy"  #监控haproxy进程
        interval 1
        weight -5
        fall 2
        rise 1
}

vrrp_script chk_down {
    script "/bin/bash -c '[[ -f /etc/keepalived/down ]]' && exit 1 || exit 0"  #在keepalived中要特别地指明作为bash的参数的运行
    interval 1
    weight -10
}

vrrp_instance VI_1 {
    state MASTER			#一台为BACKUP
    interface eth0
    virtual_router_id 51
    priority 100			#当为BACKUP时，priority适当减少，建议95。
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass fd57721a
    }
    virtual_ipaddress {
        192.168.0.2/24 dev eth0
    }
    track_script {  #调用监控脚本
            chk_haproxy
            chk_down
    }
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}

测试：创建down文件后使得降优先级，从而使得VIP漂移到node2，进入维护模式
]# touch /etc/keepalived/down
```

# ansible实现双主keepalived+nginx反代

拓扑图：

![1531399406134](http://pic.fenghong.tech/1531399406134.png)

环境：

- 各节点时间必须同步；
- 确保iptables及selinux的正确配置；
- 各节点之间可通过主机名互相通信（对KA并非必须），建议使用/etc/hosts文件实现；
- 确保各节点的用于集群服务的接口支持MULTICAST通信；D类：224-239；`ip link set dev eth0 multicast off | on`

## 配置过程

- 主机配置及同步时间

```
]# ssh-keygen -t rsa
]# ssh-copy-id -i .ssh/id_rsa.pub root@192.168.1.28
]# ssh-copy-id -i .ssh/id_rsa.pub root@192.168.1.30
]# vim /etc/ansible/hosts
[nginx]
192.168.1.30 state1=MASTER priority1=100 state2=BACKUP priority2=90	#两套变量
192.168.1.28 state1=BACKUP priority1=90 state2=MASTER priority2=100
ansbile all -s  'ntpdate 210.72.145.44 ' //是中国国家授时中心的官方服务器。
```

- 配置palybook的tasks任务

```
vim /etc/ansible/roles/nginx/tasks/main.yml
- name: install package
  yum: name={{ item }}
  with_items:
  - nginx
  - keepalived
- name: config keepalived
  template: src=keepalived.conf.j2 dest=/etc/keepalived/keepalived.conf
  notify: restart keepalived
- name: file notify.sh
  copy: src=notify.sh dest=/etc/keepalived/
- name: config nginx
  template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
  notify: restart nginx
- name: start service
  service: name={{ item }} state=started enabled=true
  with_items:
  - keepalived
  - nginx
```

- 添加handlers

```
vim /etc/ansible/roles/nginx/handlers/main.yml
- name: restart keepalived
  service: name=keepalived state=restarted
- name: restart nginx
  service: name=nginx state=restarted
```

- 准备keepalived配置文件

```
cat >> /app/keepalived.conf.j2  <<EOF
! Configuration File for keepalived

global_defs {
    notification_email {
        root@localhost
    }
    notification_email_from keepalived@localhost
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    vrrp_iptables
    router_id {{ ansible_hostname }}
    vrrp_mcast_group4 224.0.100.19
}

vrrp_script chk_down {
    script "/bin/bash -c '[[ -f /etc/keepalived/down ]]' && exit 1 || exit 0"
    interval 1
    weight -15
}

vrrp_script chk_nginx {
    script "killall -0 nginx && exit 0 || exit 1"
    interval 1
    weight -15
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state {{ state1 }}  
    interface ens33 
    virtual_router_id 14
    priority {{ priority1 }} 
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 571f87b2
    }
    virtual_ipaddress {
        192.168.1.100
    }
    track_script {
        chk_down
        chk_nginx
    }
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}

vrrp_instance VI_r2 {
    state {{ state2 }}  
    interface ens33 
    virtual_router_id 24
    priority {{ priority2 }} 
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 571f97b2
    }
    virtual_ipaddress {
        192.168.1.200
    }
    track_script {
        chk_down
        chk_nginx
    }
    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}  
EOF
```
- notify.sh模板文件

```
cat >> files/notify.sh.j2 << EOF
#!/bin/bash
#
contact='root@localhost'

notify() {
	local mailsubject="$(hostname) to be $1, vip floating"
	local mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"
	echo "$mailbody" | mail -s "$mailsubject" $contact
}

case $1 in
master)
	notify master
	;;
backup)
	notify backup
	;;
fault)
	notify fault
	;;
*)
	echo "Usage: $(basename $0) {master|backup|fault}"
	exit 1
	;;
esac
EOF
```
- nginx反代模板配置文件

```
cat >> nginx.conf.j2 << EOF
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events {
    worker_connections 1024;
}

http {
	#add upstream
	upstream web {                       
        server 192.168.1.38;
        server 192.168.1.48;
    }
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;
    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    include /etc/nginx/conf.d/*.conf;
    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;
        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        #add proxy_pass
        location / {
        	proxy_pass http://web;
        }
        error_page 404 /404.html;
            location = /40x.html {
        }
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}    
```

- 编制nginx剧本

```
vim /etc/ansible/nginx.yml
- hosts: nginx
  remote_user: root
  roles:
  - nginx
```

- 开始表演

```
]#ansible-playbook  /etc/ansible/nginx.yml 
PLAY [nginx] *******************************************************************
TASK [Gathering Facts] *********************************************************
ok: [192.168.1.30]
ok: [192.168.1.28]

TASK [nginx : install package] *************************************************
changed: [192.168.1.30] => (item=[u'nginx', u'keepalived'])
changed: [192.168.1.28] => (item=[u'nginx', u'keepalived'])

TASK [nginx : config keepalived] ***********************************************
changed: [192.168.1.30]
changed: [192.168.1.28]

TASK [nginx : file notify.sh] **************************************************
changed: [192.168.1.30]
changed: [192.168.1.28]

TASK [nginx : config nginx] ****************************************************
changed: [192.168.1.30]
changed: [192.168.1.28]

TASK [nginx : start service] ***************************************************
changed: [192.168.1.28] => (item=keepalived)
changed: [192.168.1.30] => (item=keepalived)
changed: [192.168.1.28] => (item=nginx)
changed: [192.168.1.30] => (item=nginx)

RUNNING HANDLER [nginx : restart keepalived] ***********************************
changed: [192.168.1.30]
changed: [192.168.1.28]

RUNNING HANDLER [nginx : restart nginx] ****************************************
changed: [192.168.1.30]
changed: [192.168.1.28]

PLAY RECAP *********************************************************************
192.168.1.28               : ok=8    changed=7    unreachable=0    failed=0   
192.168.1.30               : ok=8    changed=7    unreachable=0    failed=0
```

- 在keepalive上调式

```

]# tcpdump  -i  ens33 -nn host 224.0.100.19
20:11:55.223233 IP 192.168.1.28 > 224.0.100.19: VRRPv2, Advertisement, vrid 24, prio 100, authtype simple, intvl 1s, length 20
20:11:55.531460 IP 192.168.1.30 > 224.0.100.19: VRRPv2, Advertisement, vrid 14, prio 100, authtype simple, intvl 1s, length 20
20:11:56.226411 IP 192.168.1.28 > 224.0.100.19: VRRPv2, Advertisement, vrid 24, prio 100, authtype simple, intvl 1s, length 20
```
- 在192.168.1.28上创建down文件,权限立马被192.168.1.30抢去，同理反之亦然。

```
]# touch /etc/keepalived/down
]# tcpdump  -i  ens33 -nn host 224.0.100.19
20:12:46.765300 IP 192.168.1.30 > 224.0.100.19: VRRPv2, Advertisement, vrid 14, prio 100, authtype simple, intvl 1s, length 20
20:12:47.346001 IP 192.168.1.30 > 224.0.100.19: VRRPv2, Advertisement, vrid 24, prio 90, authtype simple, intvl 1s, length 20
20:12:47.769385 IP 192.168.1.30 > 224.0.100.19: VRRPv2, Advertisement, vrid 14, prio 100, authtype simple, intvl 1s, length 20
```

