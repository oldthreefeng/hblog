---
title: "ULB负载均衡踩的坑"
date: 2018-12-18 12:22:12
tags: [http]
categories: [ops]
---

[TOC]

## ULB的http强转https

负载均衡是高可用网络基础架构的关键组件，通常用于将工作负载分布到多个服务器来提高网站、应用、数据库或其他服务的性能和可靠性。

负载均衡（ULB）能够为多个主机或其它服务实例提供基于网络报文或代理方式的流量分发的功能。用于在高并发服务环境下，构建由多个服务节点组成的“负载均衡服务集群”。“服务集群”能够扩展服务的处理及容错能力，并可自动消除由于单一服务节点故障对服务整体的影响，提高服务的可用性。

目前ULB针对七层支持HTTP、HTTPs协议（类Nginx或HAproxy）；针对四层支持TCP协议及UDP协议（类LVS）。并且TCP协议支持两种方式：报文转发与请求代理。报文转发与请求代理模式详情可参考[TCP的请求代理与报文转发](https://docs.ucloud.cn/network/ulb/intro#tcp%E7%9A%84%E8%AF%B7%E6%B1%82%E4%BB%A3%E7%90%86%E4%B8%8E%E6%8A%A5%E6%96%87%E8%BD%AC%E5%8F%91)。四层ULB支持外网与内网两种模式，而七层ULB目前仅支持外网。您可以参考[如何选择ULB](https://docs.ucloud.cn/network/ulb/common#%E5%A6%82%E4%BD%95%E9%80%89%E6%8B%A9ulb)选择哪一层ULB来部署业务。

忙活了几天，对ulb的负载均衡总算是有点成果了。如下：

### 配置相关


第一步：前段负载均衡器的配置

1. 创建负载均衡器。
   进入ucloud的官网，加入开启一台ulb负载均衡器。这里对外开放，我们选择的是公网绑定

2. 创建`VServer`，和阿里云一样，创建两个`VServer`一个是80端口，一个是443端口，协议分别为`http`和`https`。

3. 用到443端口意味着我们需要上传ssl证书，上传也比较简单，上传申请的证书的crt和key文件即可。
4. `http`强制跳转至`https`，ulb暂未开通此项服务，只能在后端的服务器上添加转发规则。

第二步：后端服务器配置

1.   安装nginx ，这里就不多累述，前面已经写过。
[nginx编译安装](https://www.fenghong.tech/Nginx.html)

2.   配置nginx

```
vim /etc/nginx/conf.d/test.conf
server {
       listen       81;
       server_name  www.fenghong.tech;
       location / {
          	proxy_pass http://localhost:8080;
          	proxy_set_header       Host $host;
          	proxy_set_header  X-Real-IP  $remote_addr;
          	proxy_set_header  X-Forwarded-For$proxy_add_x_forwarded_for;
       }
        location = /50x.html {
           	root   html;
       }
}
server {
       listen       80;
       server_name  www.fenghong.tech;
       #实现http到https强转的问题
       rewrite ^(.*)$  https://www.fenghong.tech$1 permanent;
       location / {
        	proxy_pass http://localhost:8080;
       }
       location = /50x.html {
          	root   html;
       }
}
```

## 踩过的坑

- 配置后端的ssl证书且监听在443端口，导致网站被无限重定向，报错如下：

```
www2.fenghong.tech 将您重定向的次数过多。
尝试清除 Cookie.
ERR_TOO_MANY_REDIRECTS
```
-  504网关错误

由于nginx的每个`server`段的`location`里均有对ip的访问控制，取消限制或者加入白名单即可:

![504](/images/1545102659373.png?raw=true)


- 后端配置443端口证书且配置80端口，并且不配置重定向http-->https。出现以下400报错:


```
400 Bad Request
The plain HTTP request was sent to HTTPS port
```
-  最多的一次错误即403权限拒绝，公司内部的权限控制很多，缺少对这方面的敏锐性

首先在`nginx.conf`的主配置文件里面对ip有访问控制。

```
	allow  10.9.0.0/16;
	deny    all; 
```
其次在`test.conf`的主机配置文件里面，对ip有访问控制。

再其次在[知道创宇](https://www.yunaq.com)的域名访问里面，对ip有限制。

多重的权限设置是这次负载均衡踩坑的关键点。

```
403 forbidden
```

## ULB基于TCP转发的实验

四层转发或者说四层负载均衡就相对容易点。

创建ULB的负载均衡，这边实验我选择了内网转发，服务监听在内网更加安全点，这里适用场景为公司的后台管理，重要的资产端管理。

- 创建2个`vserver`，并且选择`tcp`协议，选择端口为80和443。
- 分别创建后端服务器，根据真实情况添加，一般后端服务器有2个，当然更多的后端自行创建。
- 后端服务器配置选择80和443端口。因为是4层转发，所以直接一一对应即可，方法可以选择轮询、源地址、一致性哈希、源地址，可以根据需求进行更改。

```
vserver的80端口  -----> 后端的realserver的80端口
vserver的443端口 -----> 后端的realserver的443端口
```

### 后端lo网卡的配置

- “报文转发模式”下，由于用户访问会经`ULB`直接透传，必须保证访问地址落在后端真实服务节点上，所以要将负载均衡的内/外网IP地址配置在后端服务节点中。

- 内网ULB时，这里的$VIP即为负载均衡器的内网IP地址。外网ULB时，即为负载均衡器的EIP地址。

- 将命令中得到的内容添加进"`/etc/sysconfig/network-scripts/ifcfg-lo:1`"中，即如下内容：

```
# vim /etc/sysconfig/network-scripts/ifcfg-lo:1
DEVICE=lo:1
IPADDR=$VIP
NETMASK=255.255.255.255
```

- 启动虚拟网卡

```
# ifup lo:1
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet 10.9.133.26/32 brd 10.9.133.26 scope global lo:1
```

### 后端nginx的配置

后端的nginx配置其实不用更改。下面这个例子供参考

```
server {
    listen       80;
    server_name  wiki.fenghong.tech;
	root /data/cun_web/rms;
	allow 10.9.0.0/16;
	deny all;
    rewrite ^/(.*)$ https://wiki.fenghong.tech/$1 permanent;
    location / {
        proxy_pass   http://127.0.0.1:8114;          
    }

	location =/ {  
        add_header Access-Control-Allow-Origin *;
		rewrite ^/$ /home/homeIndex last;
   
    }

	if ($http_user_agent ~* "JianKongBao") {
		return 403;
	}

 	access_log /data/logs/nginx/wiki.log  main;  
	error_log /data/logs/nginx/wiki_error.log ;
}

server {
    listen       443;
    server_name  wiki.fenghong.tech;
	root /data/cun_web/rms;
    index  index.html;

	error_page  500 502 503 504 404 403 /50_default;

	ssl_certificate      /etc/nginx/sslkey/_.q.com_bundle.crt; 
	ssl_certificate_key  /etc/nginx/sslkey/_.q.com.key; 
	ssl_session_timeout  10m;  
	ssl_session_cache shared:SSL:10m;
	ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;  
    ssl_ciphers  ALL:!DH:!EXPORT:!RC4:+HIGH:+MEDIUM:-LOW:!aNULL:!eNULL; 
    ssl_prefer_server_ciphers   on;  

	allow 10.9.0.0/16;
	deny all;

    location / {
        proxy_pass   http://127.0.0.1:8114;        
    }

	location =/ {  
        add_header Access-Control-Allow-Origin *;
	 	rewrite ^/$ /home/homeIndex last;
    }

	if ($http_user_agent ~* "JianKongBao") {
		return 403;
	}
 	access_log /data/logs/nginx/wiki_ssl.log  main;  
	error_log /data/logs/nginx/wiki_ssl_error.log ;
}
```

配置到这里也基本结束了。

## 域名指向

这里配置域名指向也是一个坑吧。正式上线的情况下，域名只有一个，要刷新`dns`缓存。

将自己的域名指向ULB的负载均衡的ip即可。但是也请谨慎.[一次DNS缓存引发的惨案](http://blog.jobbole.com/110165/).

