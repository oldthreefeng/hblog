---
title: "Gogs仓库迁移"
date: 2019-09-09 15:56:12
tags: 
  - git
  - gogs
  - docker
categories: [ops]
---

[TOC]

> 公司服务器缩减，运维平台迁移，以前的项目得一个一个迁移，比如gogs仓库迁移，jenkins迁移，sonarqube迁移，gogs迁移过程算是幸运，从阿里云迁移至ucloud，包括打包仓库，错误处理，总共花费2小时。在此记录一下gogs迁移记录，后续的迁移也会一一记录的。
>
> `# 表示注释， $ 表示shell `

## 服务器迁移准备

检查依赖，主要为gogs版本和数据库版本。

```
gogs 版本为最新版
mariadb 10.2.6 大概是2017年装的.数据库不敢动，依旧保持这个数据库。
```

###  迁移思路

- 做好数据备份，包括`gogs-repositories`,`conf`,`backup.sql`等重要备份。
- `scp`或者`stfp`将打包的文件转移至新服务器。
- 由于老版本是基于宿主机部署的，各个文件比较分散，这次转移采用`docker-compose`部署。快捷方便，一旦错误，可以快速定位，可以形成一个包。
- 启动程序快速定位问题，学会查看日志，学会`google`大法。

## 部署

- 备份，转移操作。

```shell
## gogs的配置和repo文件
$ tar zcf gogs.tar.gz gogs
$ tar zcf gogs-repositories.tar.gz gogs-repositories
$ scp gogs.tar.gz newhost:/data/gogs
$ scp gogs-repositories.tar.gz  newhost:/data/gogs

## gogs的数据文件
$ mysqldump -uroot -h'127.0.0.1' -ppassword --databases gogs > gogs.sql 
$ scp gogs.sql newhost:/data/gogs
```

- 目录结构如下：

```
$ mkdir /data/gogs/{data,mysql} -p
$ cd /data/gogs
## 文件结构如下
$ tree  -L 1
.
├── data
├── docker-compose.yaml
└── mysql

```

- 基于docker-compose部署gogs

编写`docker-compose.yaml`文件，如下：

```yaml
$ cat docker-compose.yaml 
version: '2'
services:
  gogsdb:
    container_name: gogsdb
    image: mariadb:10.2.6
    volumes:
      - "/data/gogs/mysql:/var/lib/mysql"
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: $yourpassword 
      MYSQL_DATABASE: gogs
      MYSQL_USER: root
      MYSQL_PASSWORD: $yourpassword
  gogs:
    container_name: gogs
    depends_on:
      - gogsdb
    image: gogs/gogs
    volumes:
      - /data/gogs/data:/data
    links:
      - gogsdb
    ports:
      - "10080:3000"
      - "22:22"
    restart: always
```

### 导入数据库文件

- 启动服务导入数据,然后进入安装页面

```shell
## 1. 启动服务
$ cd /data/gogs
$ docker-compose up 
### 这里建议先不使用-d，选择查看启动日志，是否由报错，基本不会报错，等待mysql数据库生成好文件，大概几秒钟的时间。

## 2. 导入mysql数据库
$ mysql -p$yourpassword  < gogs.sql
```

### 安装页面初始化

如果直接把你以前的配置导入，会有500的internel报错，具体原因我也没找到。所以这里博主采用先安装初始化，然后直接更改配置文件的方法。具体初始话可以看我上篇[gogs安装](https://wiki.fenghong.tech/go/gogs-repo-install.html)

- **注意事项**
- 安装的数据库填写自己`docker-compose.yaml`文件的数据库主机`gogsdb:3306`和密码.
- 应用的基本设置安装`app.ini`文件填写，也可以随便填写，后面需要可以把以前的配置文件按需复制
- 管理员配置直接跳过，因为数据库里面的数据已经存在了管理员，如果你需要重置管理员，那自己随意。

![](https://pic.fenghong.tech/gogsinstall.png)

- 点击跳过管理员后，直接会跳转到登陆界面，登陆，用户名密码为你前系统的用户名密码，没有改变。可能这里有人会说，你的repo数据还没有导入呢，对，这点击进入的任何repo有关的都是报的500，找不到。

### 导入repo仓库数据

```bash
$ cd /data/gogs/
$ tar xf gogs-repositories.tar.gz
$ cp -rf  /data/gogs/gogs-repositories  /data/gogs/data/git/gogs-repositories

## 重启gogs服务
$ docker-compose up -d --force

## 做到这一步基本网页访问正常，git pull没有问题了，本地只需要改一下repo地址就可以完美迁移完毕。
```

## 突然来的问题

### push error, the hooks/update path is not config file path

在`git pull`没有问题的情况，博主就点了一杯奶茶，喝了几口压压惊。不一会，就有小伙伴跑过来说`git push`用不了，这让我心头一沉，怎么回事，docker部署哪里由问题么，赶紧放下刚润嗓子的奶茶，就查起问题。

```bash
$ git push origin hotfix_xxxx_xxxx
remote: hooks/update: line 2: /app/gogs/gogs: No such file or directory
remote: error: hook declined to update refs/heads/master
...
pre-receive hook declined
```

`google`大法好,这里参照了[push error, the hooks/update path is not config file path](https://github.com/gogs/gogs/issues/1916)这个方法解决的。

主要原因是:

```
I suppose you copied the repositories from another Gogs installation.

Go to admin dashboard and do:
```

![image](https://pic.fenghong.tech/pusherror.png)

### 22端口为git的访问失败

用户在`用户设置`-->`ssh 密钥`设置中添加自己的ssh公钥

却发现还是不能`git clone`或者`git pull`

```shell
1. 22端口被防火墙封禁，打开22端口即可
$ ssh -T git@fenghong.tech
ssh_exchange_identification: read: Connection reset by peer

2. permission deny
$ ssh -T git@fenghong.tech
Permission denied (publickey,keyboard-interactive).

## 解决，进入容器查看git用户的authorized_keys文件权限。
$ docker exec -it gogs bash
bash-5.0# 
bash-5.0# cd /home/git/.ssh/
bash-5.0# ls -la
total 16
drwx------    2 git      git           4096 Sep 10 07:47 .
drwxr-xr-x    4 git      git           4096 Sep 10 08:07 ..
-rwxr-xr-x    1 git      git            860 Sep 10 07:47 authorized_keys
-rwxr-xr-x    1 git      git             23 Sep 10 03:48 environment
bash-5.0# chmod 600 *

3. 验证，可以正常使用git了。
$ ssh -T git@fenghong.tech
Hi there, You've successfully authenticated, but Gogs does not provide shell access.
If this is unexpected, please log in with password and setup Gogs under another user.
```

### 参考

- [无闻大佬](https://github.com/gogs/gogs)