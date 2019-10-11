---
title: "Gogs仓库搭建"
date: 2019-05-09 15:56:12
tags: 
  - go
  - git
  - gogs
categories: [ops]
---

[TOC]

### 介绍

> Have you used Gogs? It’s great. Gogs is a Git service, much like GitHub and GitLab, but written in Go. It’s a immensely lighter than GitLab and it’s not lacking at all in features. 

### `Gogs`仓库构建

`gogs `安装方式很多，二进制安装/源码安装/docker安装,具体细节可以查看[官网](https://gogs.io/)或者[github](https://github.com/gogs/gogs)

这里我选择的是docker-compose安装,使用docker-compose方便快捷.

### 安装docker

这里不便赘述查看先前写的`安装kubernetes`里面的[安装docker步骤](https://wiki.fenghong.tech/go/kubeadm安装kubernetes1.15.html#docker)

当然也可以选择[官网安装](https://docs.docker.com/get-started/)

- 安装docker-compose

```
curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

- 编写`docker-compose.yml`文件

```
$ cd /data/gogs
$ vi docker-compose.yml
version: '2'
services:
  gogsdb:
    container_name: gogsdb
    image: mariadb:10.2.18 
    volumes:
      - "/usr/local/mariadb/data:/var/lib/mysql"
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: $PASSWORD
      MYSQL_DATABASE: gogs
      MYSQL_USER: $USER
      MYSQL_PASSWORD: $PASSWORD
  gogs:
    container_name: gogs
    depends_on:
      - gogsdb
    image: gogs/gogs
    volumes:
      - /var/gogs:/data
    links:
      - gogsdb
    ports:
      - "10080:3000"	#暴露的http端口
      - "10022:22"		#暴露的ssh端口,按照自己的喜好更换		
    restart: always
  jenkins:
    image: jenkins/jenkins:2.183
    container_name: jenkins
    restart: always
    ports:
      - 18080:8080
      - 50000:50000
    links:
      - gogs:gogs
    volumes:
      - /data/jenkins:/var/jenkins_home
```

- 启动服务

```
docker-compose up -d
```

- 停止服务

```
docker-compose down -v
```

### `gogs`配置文件

部署成功后,直接访问[gogs项目](http://gogs.wangke.co/),然后开始进行配置

首先是数据库的配置按照配置文件写入即可

其次是应用的基本设置,这个设置比较重要,关系到`git clone`的操作参数

其中应用名称,可以写公司的名称,比如`gogs`;仓库的根目录`/data/git/gogs-repositories`,可以选择默认

域名选择以后想要`git clone`的域名,比如`gogs.wangke.co`,ssh端口号填写自己映射的端口比如`10022`

`http`端口号这个必须写`3000`,这个监听在容器内,不然容器起不来.

应用`Url`,填写`git clone`的http服务的url,比如`http://gogs.wangke.co:10080/`,这里端口填写自己对外映射的端口.这里我做了一次反向代理了,使用`nginx`反向代理了本地的`http://127.0.0.1:10080`.所以把端口省略了.

`admin`选项自己填写即可.以下是生成的`app.ini`配置文件.

```
APP_NAME = Gogs
RUN_USER = git
RUN_MODE = prod

[database]
DB_TYPE  = mysql
HOST     = gogsdb:3306
NAME     = gogs
USER     = $USER
PASSWD   = $PASSWORD
SSL_MODE = disable
PATH     = data/gogs.db

[repository]
ROOT = /data/git/gogs-repositories

[server]
DOMAIN           = gogs.wangke.co
HTTP_PORT        = 3000
ROOT_URL         = http://gogs.wangke.co/
DISABLE_SSH      = false
SSH_PORT         = 10022
START_SSH_SERVER = false
OFFLINE_MODE     = false

[mailer]
ENABLED = false

[service]
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL     = false
DISABLE_REGISTRATION   = false
ENABLE_CAPTCHA         = true
REQUIRE_SIGNIN_VIEW    = false

[picture]
DISABLE_GRAVATAR        = false
ENABLE_FEDERATED_AVATAR = false

[session]
PROVIDER = file

[log]
MODE      = file
LEVEL     = Info
ROOT_PATH = /app/gogs/log

[security]
INSTALL_LOCK = true
SECRET_KEY   = RNcQAMf374kihMd
```

### 改变ssh端口为22

- 使用ssh默认的22端口,看起来会比较优雅,这里可以看一下`gogs`官网推荐的一篇文章.

[在Docker和本地系统内的Gogs之间共享端口22](http://www.ateijelo.com/blog/2016/07/09/share-port-22-between-docker-gogs-ssh-and-local-system)

- 解决方案

	这里提供我的思路,不用利用`realServer`的ssh来映射默认的22端口,宿主机因为安全问题,我一般都会改变ssh的默认端口,比如改22端口为9527,这里,系统默认的ssh端口22就空出来了,刚好可以作为`gogs`的补缺.直接将`gogs`容器端口22映射到宿主机的端口22.更改配置文件里面的`SSH_PORT `为22,即可.

```
cd /data/gogs
sed -i "/10022/22/g"  docker-compose.yml
sed -i "/SSH_PORT         = 10022/SSH_PORT         = 22/g" /var/gogs/gogs/conf/app.ini
docker restart mygogs
```

- 如果主机的ssh端口为改变,可以参照`gogs`官方网站推荐的进行操作~