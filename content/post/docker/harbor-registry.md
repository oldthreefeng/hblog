---
title: "搭建https镜像仓库harbor记录"
date: 2019-11-27T15:22:18+08:00
lastmod: 2019-12-10T19:07:18+08:00
tags: [docker,harbor]
categories: [docker]
---

[TOC]
## 说明

- \# 开头的行表示注释
- \> 开头的行表示需要在 mysql 中执行
- $ 开头的行表示需要执行的命令

本文档适用于有一定web运维经验的管理员或者工程师，文中不会对安装的软件做过多的解释，仅对需要执行的内容注部分注释，更详细的内容请参考其他安装。

## 环境

- 系统 : CentOS Linux release 7.7.1908 (Core) , 3.10.0-1062.el7.x86_64

- ip: 192.168.0.65

- 目录: /home/louis

- 依赖: docker, docker-compose

## 项目结构

```bash
$ tree   -L 2
.
├── certs
│   ├── fenghong.tech.cer
│   └── fenghong.tech.key
├── data
│   ├── ca_download
│   ├── database
│   ├── job_logs
│   ├── psc
│   ├── redis
│   ├── registry
│   └── secret
├── harbor
│   ├── common
│   ├── docker-compose.yml
│   ├── harbor.v1.9.3.tar.gz
│   ├── harbor.yml
│   ├── install.sh
│   ├── LICENSE
│   └── prepare
└── logs
    ├── core.log
    ├── jobservice.log
    ├── portal.log
    ├── postgresql.log
    ├── proxy.log
    ├── redis.log
    ├── registryctl.log
    └── registry.log
```

## 部署步骤

```
# 下载harbor-offline
$ su louis
$ cd /home/louis/
$ wget https://github.com/goharbor/harbor/releases/download/v1.9.3/harbor-offline-installer-v1.9.3.tgz
# 解压
$ tar xf harbor-offline-installer-v1.9.3.tgz
# 修改配置文件
$ cd /home/louis/harbor
$ cp harbor.yml harbor.yml.bak
$ cat > harbor.yml <<eof
hostname: harbor.fenghong.tech 
http:
  port: 81
https:
  port: 443
  certificate: /home/louis/certs/fenghong.tech.cer
  private_key: /home/louis/certs/fenghong.tech.key
harbor_admin_password: Harbor12345
database:
  password: root123
  max_idle_conns: 50
  max_open_conns: 100
data_volume: /home/louis/data
clair:
  updaters_interval: 12
jobservice:
  max_job_workers: 10
notification:
  webhook_job_max_retry: 10
chart:
  absolute_url: disabled
log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /home/louis/logs
_version: 1.9.0
proxy:
  http_proxy:
  https_proxy:
  no_proxy: 127.0.0.1,localhost,.local,.internal,log,db,redis,nginx,core,portal,postgresql,jobservice,registry,registryctl,clair
  components:
    - core
    - jobservice
    - clair
eof
```

## 配置https证书并启动项目

参考[https://blog.fenghong.tech/post/ops/acme-ssl-cert/](https://blog.fenghong.tech/post/ops/acme-ssl-cert/)

```
$ su louis
$ curl https://get.acme.sh | sh
$ export Ali_Key="alikey"
$ export Ali_Secret="alikeySecret"
$ . .bashrc
$ acme.sh --issue --dns dns_ali -d *.fenghong.tech -d fenghong.tech
$ cd .acme.sh/\*.fenghong.tech/
$ acme.sh --install-cert -d *.fenghong.tech  --key-file /home/louis/certs/fenghong.tech.key --fullchain-file /home/louis/certs/fenghong.tech.cer
```

启动项目, 因为普通用户是没有权限的, 需要用到sudo去操作docker-compose, 且将louis用户加入到docker组.

```
$ sudo usermod -aG docker louis
$ sudo ./install
```

查看项目

```
$ cd  /home/louis/harbor && sudo  docker-compose ps 
Name            Command           	         State       Ports                   
-----------------------------------------------------------------------------------------------
harbor-core  /harbor/harbor_core            Up (healthy)                                         
harbor-db    /docker-entrypoint.sh          Up (healthy)   5432/tcp                             
harbor-jobser/harbor/harbor_jobservice  ... Up (healthy)                                         
harbor-log   /bin/sh -c /usr/local/bin/ ... Up (healthy)   127.0.0.1:1514->10514/tcp             
harbor-portalnginx -g daemon off;           Up (healthy)   8080/tcp                             
nginx        nginx -g daemon off;           Up (healthy)   0.0.0.0:81->8080/tcp, 0.0.0.0:443->8443/tcp
redis        redis-server /etc/redis.conf   Up (healthy)   6379/tcp                             
registry     /entrypoint.sh /etc/regist ... Up (healthy)   5000/tcp                             
registryctl  /harbor/start.sh               Up (healthy)     
```
重启项目

```
$ cd  /home/louis/harbor && sudo docker-compose down
$ sudo docker-compose up -d
```

## 配置DNS解析并开始访问

```
$ dig harbor.fenghong.tech 

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.68.rc1.el6_10.1 <<>> harbor.fenghong.tech
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52482
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;harbor.fenghong.tech.		IN	A

;; ANSWER SECTION:
harbor.fenghong.tech.	600	IN	A	192.168.0.65
```

访问网站[harbor](https://harbor.fenghong.tech)

### 部署支持helm的chart仓库

```bash
$ cd  /home/louis/harbor && sudo  docker-compose down -v 
$ sudo ./install.sh --with-chartmuseum
```

