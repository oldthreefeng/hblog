---
title: "内网穿透使用wss"
date: 2020-05-12T17:58:18+08:00
tags: [frp,websocket,http,ssl]
categories: [server]
---



[toc]

## 前言

> 公司测试环境需要用到websocket项目. 因小程序需要https连接. 所以要弄wss. WebSocket可以使用 ws 或 wss 来作为统一资源标志符，类似于 HTTP 或 HTTPS。其中 ，wss 表示在 TLS 之上的 WebSocket，相当于 HTTPS。默认情况下，WebSocket的 ws 协议基于Http的 80 端口；当运行在TLS之上时，wss 协议默认是基于Http的 443 端口。说白了，wss 就是 ws 基于 SSL 的安全传输，与 HTTPS 一样样的道理。所以，如果你的网站是 HTTPS 协议的，那你就不能使用 ws:// 了，浏览器会 block 掉连接，和 HTTPS 下不允许 HTTP 请求一样。

## `FRP`作用

对于没有公网 `IP` 的内网用户来说，远程管理或在外网访问内网机器上的服务是一个问题。通常解决方案就是用内网穿透工具将内网的服务穿透到公网中，便于远程管理和在外部访问。内网穿透的工具很多, 比如 [Ngrok](https://github.com/inconshreveable/ngrok.git).

推荐一款好用到炸裂的内网穿透的工具`frp`, 全名: `Fast Reverse Proxy`.  `frp` 是一个可用于内网穿透的高性能的反向代理应用，支持 `tcp, udp` 协议，为 `http` 和` https `应用协议提供了额外的能力，且尝试性支持了点对点穿透。 

- 利用处于内网或防火墙后的机器，对外网环境提供 `HTTP` 或 `HTTPS` 服务。
- 对于 `HTTP`, `HTTPS` 服务支持基于域名的虚拟主机，支持自定义域名绑定，使多个域名可以共用一个 80 端口。
- 利用处于内网或防火墙后的机器，对外网环境提供 `TCP` 和 `UDP` 服务，例如在家里通过 `SSH` 访问处于公司内网环境内的主机。
- ...

## FRP架构

 ![architecture](http://pic.fenghong.tech/frp/architecture.png) 

## 内网使用websocket逻辑图

 ![architecture](http://pic.fenghong.tech/frp/wss.jpg)

内部机器部署外网访问, 必要的就是一个公网ip + frp , 后面就是正常的nginx做web应用服务即可.

## 配置相关

### frps配置

```
[common]
bind_addr = 0.0.0.0
bind_port = 7200
bind_udp_port = 7201
kcp_bind_port = 7200

vhost_http_port = 80
vhost_https_port = 443

dashboard_addr = 0.0.0.0
dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = admin

log_file = ./frps.log
log_level = debug
log_max_days = 3

token = 123456

max_pool_count = 5
max_ports_per_client = 0
authentication_timeout = 0
tcp_mux = true
```

### fprc配置

```
[common]
server_addr = frps公网服务器ip
server_port = 7200
token = 123456
admin_addr = 127.0.0.1
admin_port = 7400

[http]
type = http
local_ip = 192.168.0.21
local_port = 80
remote_port = 80
use_encryption = false
use_compression = true
custom_domains =  www.fenghong.tech

[https]
type = https
local_ip = 192.168.0.21
local_port = 443
remote_port = 443
use_encryption = false
use_compression = true
use_gzip = true
custom_domains =  www.fenghong.tech
```

### nginx配置

```
server {
	server_name www.fenghong.tech
...
	location /webSocketServer {
		proxy_pass http://websocket; # 本地服务器地址及端口
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header Host $host;
		proxy_set_header X-Forward-Proto https;
		proxy_http_version 1.1;
        # for websocket
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
	}
...
}  
upstream websocket {
	ip_hash;
	server   192.168.0.63:7081  weight=1 max_fails=2 fail_timeout=30s;
	server   192.168.0.64:7081  weight=1 max_fails=2 fail_timeout=30s;
}

```

配置结束.



