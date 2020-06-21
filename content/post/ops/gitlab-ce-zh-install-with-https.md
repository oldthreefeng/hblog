---
title: "gitlab-ce-zh部署并开启https"
date: "2020-05-04T11:54:18+08:00"
tags: [gitlab,git,nginx,ssl,acme]
categories: [server,ops]
---

[TOC]

> 背景:  源码仓库管理有很多, gitlab算是很经典的一款.  
>
> 前提: 
>
> - 安装 `docker`, `docker-compose`, `nginx`, `acme.sh`

## 效果成品图

![node_export](https://oss.fenghong.tech/gitlab/gitlab_20200507153757.jpg)

## 部署

采用docker部署. 方便快捷.  数据持久化备份也简单. 

```
$ mkdir /home/data/gitlab/{config,data,logs} -p
$ cd /home/data/gitlab/
$ vim docker-compose.yml
version: "3"
services:
  gitlab:
    image: twang2218/gitlab-ce-zh:latest
    container_name: gitlab
    privileged: true
    restart: always
    hostname: 'mxqh168.co'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url "https://mxqh168.co"
        nginx['enable'] = true
        nginx['client_max_body_size'] = '1024m'
        nginx['redirect_http_to_https'] = true
        nginx['redirect_http_to_https_port'] = 80
        nginx['ssl_certificate'] = "/etc/gitlab/ssl/mxqh168.co.cer"
        nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/mxqh168.co.key"
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 22
        gitlab_rails['gitlab_email_enabled'] = true
        gitlab_rails['gitlab_email_from'] = '18795605909@163.com'
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtp.163.com"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "18795605909@163.com"
        gitlab_rails['smtp_password'] = "*************"   #这个是网易的授权登录码
        gitlab_rails['smtp_domain'] = "163.com"
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
    ports:
      - '10080:80'
      - '10443:443'
      - '22:22'
    volumes:
      - '/home/data/gitlab/config:/etc/gitlab'
      - '/home/data/gitlab/logs:/var/log/gitlab'
      - '/home/data/gitlab/data:/var/opt/gitlab'
     
$ docker-compose up -d 
```

> 关于ssh端口问题.

gitlab的服务器使用的是 SSH 默认端口 `22` 去映射容器 SSH 端口。其目的是希望比较自然的使用类似 `git@gitlab.example.com:myuser/awesome-project.git` 的形式来访问服务器版本库。但是，宿主服务器上默认的 SSH 服务也是使用的 22 端口。因此默认会产生端口冲突。

修改宿主的 SSH 端口，使用非 `22` 端口。比如修改 SSHD 配置文件，`/etc/ssh/sshd_config`，将其中的 `Port 22` 改为其它端口号，然后 `service sshd restart`。这种方式比较推荐，因为管理用的宿主 SSH 端口改成别的其实更安全。

## nginx 配置

nginx配置和ssl证书是需要同步进行的.  先配置80端口, 然后配置ssl证书,然后再配置https. 这些操作就懒得再重复写了.

```
server {
	listen       80;
	server_name  mxqh168.co;

	return      301  https://mxqh168.co$request_uri;
	# 用户ssl证书生成
	location /.well-known/acme-challenge/ {
		alias /var/www/ssl/.well-known/acme-challenge/;
	}
}
server {
	listen       443 ssl;
	server_name  mxqh168.co;
	ssl_certificate      /etc/nginx/certs/mxqh168.co.cer;
	ssl_certificate_key  /etc/nginx/certs/mxqh168.co.key;

	location / {
		proxy_pass https://gitlab;
	}
	location /.well-known/acme-challenge/ {
		alias /var/www/ssl/.well-known/acme-challenge/;
	}
}
upstream gitlab {
	server   127.0.0.1:10443 weight=1 max_fails=2 fail_timeout=30s;
}

# 启动nginx
$ nginx
```

## ssl证书生成

采用acme.sh进行免费生成ssl证书. 这里没有用到api生成, 采用的是目录验证方式. 

```
$ curl https://get.acme.sh | sh
$ mkdir /var/www/ssl/.well-known/acme-challenge -p
## 生成ssl证书.
$ acme.sh --issue -d mxqh168.co -w /var/www/ssl/
## 生成宿主机nginx的ssl证书
$ acme.sh  --install-cert -d mxqh168.co \
--key-file /etc/nginx/certs/mxqh168.co.key \
--fullchain-file /etc/nginx/certs/mxqh168.co.cer \
--reloadcmd "nginx -s reload"
## 生成docker容器里面的证书
$ acme.sh  --install-cert -d mxqh168.co --key-file \
/home/data/gitlab/config/ssl/mxqh168.co.key --fullchain-file \
/home/data/gitlab/config/ssl/mxqh168.co.cer
```

访问 https://mxqh168.co 即可

## gitlab管理员密码更改

```
## 1. 查看容器
$ docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS                 PORTS                                                               NAMES
3da8f1560c55        twang2218/gitlab-ce-zh   "/assets/wrapper"   2 weeks ago         Up 2 weeks (healthy)   0.0.0.0:22->22/tcp, 0.0.0.0:10080->80/tcp, 0.0.0.0:10443->443/tcp   gitlab
## 2. 进入gitlab容器
$ docker exec -it 3da8f1560c55 /bin/sh

## 3. 加载gitlab-rails控制台
# gitlab-rails console -e production
-------------------------------------------------------------------------------------
 GitLab:       11.1.4 (63daf37)
 GitLab Shell: 7.1.4
 postgresql:   9.6.8
-------------------------------------------------------------------------------------
Loading production environment (Rails 4.2.10)
irb(main):001:0> 

## 4. 查找admin用户
### 方法一 
irb(main):001:0> user = User.where(id: 1).first
### 方法二
irb(main):001:0> user = User.find_by(email: 'admin@example.com')

## 5. 更改密码并保存
irb(main):002:0> user.password = 'secret_pass'
irb(main):003:0> user.password_confirmation = 'secret_pass'
irb(main):004:0> user.save!

```

