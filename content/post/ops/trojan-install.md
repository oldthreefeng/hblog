---
title: "搭建trojan服务"
date: "2020-03-04T13:54:18+08:00"
tags: [trojan,ss,nginx,ssl,acme,chrome]
categories: [server,ops]
---

[TOC]

摘要：

- 科学上网
- trojan

> 背景：由于ss，ssr流量特征不明显导致其明显， 故而转战trojan。 

## trojan简介

An unidentifiable mechanism that helps you bypass GFW。

## 原理 


![jwkZ9mtbovqGISi](http://pic.fenghong.tech/trojan/jwkZ9mtbovqGISi.jpg)

```
如图所示，Trojan工作在443端口，所以它会占用443端口，
处理来自外界的HTTPS请求，如果是Trojan请求，那么为该请求提供服务，
如果不是它就会将该流量转交给Nginx，由Nginx为其提供服务。

通过这个工作过程可以知道，Trojan的一切表现均与Nginx一致，
不会引入额外特征，从而达到无法识别的效果。

当然，为了防止恶意探测，我们需要将80端口的流量全部重定向到443端口，
并且服务器只暴露80和443端口，80端口还是由nginx管理，
但443则由trojan管理，所以要赋予它监听443的权力，
这样可以使得服务器与常见的Web服务器表现一致。
```

## 前提要求

- **系统要求：[Ubuntu](https://ubuntu.com/) >= 16.04** or **[Debian](https://www.debian.org/) >= 9** or **[CentOS](https://centos.org/) >=** **7**

- **服务器：1H256M （最低配），带宽>100M 大小4G+**

- **域名**： 任意
- **torjan**: v1.14.1

## 服务器安装依赖程序及证书配置

安装`wget`和`nginx`

```
$ yum install wget nginx vim  -y
```

安装`trojan`

```
$ cd /opt
$ wget https://github.com/trojan-gfw/trojan/releases/download/v1.14.1/trojan-1.14.1-linux-amd64.tar.xz
$ tar xf trojan-1.14.1-linux-amd64.tar.xz
```

安装`ssl`证书, 使用acme项目进行安装`ssl`证书, 具体细节可以查看我之前写的免费证书[证书ssl安装](http://www.fenghong.tech/post/ops/acme-ssl-cert/). 采用阿里云的api进行申请免费证书. (默认域名在阿里云购买)

```
$ curl https://get.acme.sh | sh
$ export Ali_Key="sdfsdfsdfljlbjkljlkjsdfoiwje"
$ export Ali_Secret="jlsdflanljkljlfdsaklkjflsa"
$ acme.sh --issue --dns dns_ali  -d '*.example.com'  --force
$ acme.sh --install-cert -d *.example.com --key-file  /etc/nginx/certs/cert.example.com.key --fullchain-file /etc/nginx/certs/example.fullchain.cer --force
```

## 服务器相关配置修改

`trojan`配置修改

```
$ cd /opt/trojan
$ cat config.json
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 443,
    "remote_addr": "127.0.0.1",
    "remote_port": 80,
    "password": [
        "password"    //修改成自己的密码
    ],
    "log_level": 1,
    "ssl": {
        "cert": "/etc/nginx/certs/fenghong.tech/fullchain.cer",    //修改成自己生成的证书
        "key": "/etc/nginx/certs/fenghong.tech/cert.fenghong.key", //修改成自己的生成的秘钥
        "key_password": "",
        "cipher": "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384",
        "cipher_tls13": "TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_256_GCM_SHA384",
        "prefer_server_cipher": true,
        "alpn": [
            "http/1.1"
        ],
        "reuse_session": true,
        "session_ticket": false,
        "session_timeout": 600,
        "plain_http_response": "",
        "curves": "",
        "dhparam": ""
    },
    "tcp": {
        "prefer_ipv4": false,
        "no_delay": true,
        "keep_alive": true,
        "reuse_port": false,
        "fast_open": false,
        "fast_open_qlen": 20
    },
    "mysql": {
        "enabled": false,
        "server_addr": "127.0.0.1",
        "server_port": 3306,
        "database": "trojan",
        "username": "trojan",
        "password": ""
    }
}

$ ./trojan -c config.json &
```

`nginx`配置

主要修改的已经在注释里面写清楚了.

```
server {
	listen 127.0.0.1:80 default_server;
	# `server_name`的值`www.fenghong.tech`改为你自己的域名；
	server_name www.fenghong.tech;
	## 后面这个是我的blog, 如果没有, 可以反向代理到任意没有敏感信息的网站即可.
    error_page 404 /404.html;
	root         /www;

    access_log  /var/log/nginx/access_www.log  main;
    error_log  /var/log/nginx/error_www.log  ;
    location ~ /api/v1/ {
            proxy_set_header Host  $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8080$request_uri;

            proxy_cookie_path ~(.*) /;
    }
    location /lottery {
            root /app/js;
    }
    location  /wallpaper {
            auth_basic "Please input password"; #这里是验证时的提示信息 
            auth_basic_user_file /etc/nginx/passwd;
            root         /www;
            autoindex on; # 开启目录文件列表
            autoindex_exact_size on; # 显示出文件的确切大小，单位是bytes
            autoindex_localtime on; # 显示的文件时间为文件的服务器时间
    }
}


server {
	listen 127.0.0.1:80;
	## server_name的值8.12.22.32改为你自己的IP；
	server_name 8.12.22.32;
	## `www.fenghong.tech`改为你自己的域名；
	return 301 https://www.fenghong.tech$request_uri;
}

server {
	listen 80;
	server_name _;
  	return 301 https://$host$request_uri;
}
```

这里解释一下三个nginx虚拟主机的作用:

```
第一个server接收来自Trojan的流量，与上面Trojan配置文件对应；

第二个server也是接收来自Trojan的流量，但是这个流量尝试使用IP而不是域名访问服务器，
所以将其认为是异常流量，并重定向到域名；

第三个server接收除127.0.0.1:80外的所有80端口的流量并重定向到443端口，
这样便开启了全站https，可有效的防止恶意探测。

注意到，第一个和第二个server对应综述部分原理图中的蓝色数据流，
第三个server对应综述部分原理图中的红色数据流，
综述部分原理图中的绿色数据流不会流到Nginx。
```

![jwkZ9mtbovqGISi](http://pic.fenghong.tech/trojan/jwkZ9mtbovqGISi.jpg)

如果你本机已经有Nginx服务，那么Nginx配置文件需要做适当修改以和现有服务兼容。

```
在原服务与Trojan使用同一个域名且原来是监听443端口的情况下，
那么需要将你的ssl配置删除并将监听地址改为第一个server监听的地址127.0.0.1:80，
然后直接用修改好的server代替上述配置文件中第一个server即可。
这样https加密部分将会由Trojan处理之后转发给Nginx而不是由Nginx处理
原来的服务对于客户端来说就没有变化。

如果原来的服务与Trojan使用不同的域名，建议是修改Trojan与原来的服务使用同一个域名，
如果非要使用不同的域名，那么请自己琢磨Nginx的sni，
参考连接：[ngx_stream_ssl_preread_module]
(https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html)。

如果原来的服务是监听80端口，想要继续监听80端口那么直接去除第三个server即可，
如果要改为监听443端口参考第1点。
```

## Windows或Mac客户端部署

几点说明，目前客户端Trojan不能使用全局代理，所以需要配合其他软件使用，比如proxifier等。推荐使用Trojan+Chrome插件SwitchyOmega实现只能Chrome的目的。

这样Trojan只用监听一个端口，由Chrome插件决定当前流量是否走代理。如果你有别的用途可以单独在某个软件内部使用SOCKS5协议指定代理，地址为Trojan的监听地址：127.0.0.1:1080。

### 配置Windows客户端

Windows客户端下载地址[Trojan for Windows](https://github.com/trojan-gfw/trojan/releases)，打开之后下载最新版本的win.zip压缩包。

下载成功之后解压，修改目录中的`config.json`配置文件中的`local_port`、`remote_addr`和`password`即可。其中，`remote_addr`填写自己的域名，`local_port`开启本地端口，

用来接收本地数据，建议修改为不常用端口，否则容易冲突，本文仅使用默认端口1080演示。Trojan不需要安装就可以直接运行，拷贝Trojan文件夹到电脑里面，双击即可运行。

如果启动报错，那么说明你的系统里面没有C++运行环境，需要安装[VC++运行环境](https://support.microsoft.com/en-us/help/2977003/the-latest-supported-visual-c-downloads)（1.12.3及以前版本安装x86环境，1.13.0及以后版本安装x64环境，或者两个版本都安装也行），

然后重新启动Trojan，确认Trojan没有报错即可。如果启动Trojan会一闪而过，那么应该是你配置文件有错误，请仔细检查。可以使用控制台运行Trojan，能看到具体是哪一行有错.

### 配置Mac客户端

Mac客户端下载地址[Trojan for Mac](https://github.com/trojan-gfw/trojan/releases)，打开之后下载最新版的macos.zip，编辑配置文件同Windows客户端，编辑好配置文件后双击运行`start.command`即可。如果出

现`bind: Permission denied`错误，需要在终端使用`killall trojan`命令杀掉现有的Trojan相关的进程。如果出现`fatal: config.json(n): invalid code sequence`错误，

那么是你的配置文件第`n`行有错误，请检查。

### 安装SwitchyOmega

不会翻墙，下面有下载链接：[走你](https://github.com/FelisCatus/SwitchyOmega/releases)

还有个百度经验：[度娘知道](https://jingyan.baidu.com/article/219f4bf7a0b737de442d38e8.html)

安装好`SwitchyOmega`后, 可以一键导入配置

```
http://www.fenghong.tech/OmegaOptions.bak
```

至此客户端Trojan已经配置完成，尽情享受吧！！！

### 参考

- [TROJAN搭建与BBR魔改开启](https://bk.shunleite.com/post-45.html)

