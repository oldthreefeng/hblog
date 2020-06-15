---
title: "白嫖网易云音乐vip~"
date: 2020-06-15T17:34:18+08:00
tags: [learning,UnblockNeteaseMusic]
categories: [tools]
---

[TOC]

> 背景: 提高生活质量, 白嫖网易云vip歌曲.  
>
> 底层原理:  借用其他源替代网易云音乐源. 

## 要求

- linux服务器一台并且安装`docker`及`docker-compose`
- 懂一点网络的知识即可(配置代理)

## 配置UnblockNeteaseMusic服务端

安装`docker`环境及`docker-compose`, 启动服务即可. 假设服务器的ip为 `192.168.0.23`

```
## 安装docker
$ cd /etc/yum.repo.d/ && wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 
$ yum install docker-ce -y 
$ systemctl start docker && systemctl enable docker

## 安装docker-compose
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
$ chmod +x /usr/bin/docker-compose
$ cat > docker-compose.yml <<EOF 
version: '3'
services:
  unblockneteasemusic:
    image: nondanee/unblockneteasemusic
    environment:
      NODE_ENV: production
    ports:
      - 18188:8080
EOF
$ docker-compose up -d 
## 出现下面即部署成功. 
$ ss -tnl |grep 18188
LISTEN     0      128       [::]:18188                 [::]:*   
```

## 配置客户端

windows的网易云音乐客户端配置, 点击确定重启客户端即可白嫖...

![](https://pic.fenghong.tech/tools/wangyiyun_20200615172435.jpg)

安卓手机设置

```
WIFI -> 网络详情 -> 代理  -> 自动代理配置  -> http://192.168.0.23:18188/proxy.pac
```

mac和ios手机设置大同小异. 