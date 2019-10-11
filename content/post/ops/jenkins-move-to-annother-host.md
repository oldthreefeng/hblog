---
title: "jenkins自动化发布工具迁移"
date: 2019-09-09 15:56:12
tags: 
  - git
  - jenkins
  - docker
categories: [ops,jenkins]
---

[TOC]

> ## 背景
> 公司服务器缩减，运维平台迁移，以前的项目得一个一个迁移，比如gogs仓库迁移，jenkins迁移，sonarqube迁移，gogs迁移过程算是幸运，从阿里云迁移至ucloud，包括打包仓库，错误处理，总共花费2小时。在此记录一下jenkins迁移记录，后续的迁移也会一一记录的。
>
> `# 表示注释， $ 表示shell `

## 服务器迁移准备


###  迁移思路

- 做好数据备份，包括`/var/lib/jenkins`数据文件的重要备份。
- `scp`或者`stfp`将打包的文件转移至新服务器。
- 由于老版本是基于宿主机部署的，各个文件比较分散，这次转移采用`docker-compose`部署。快捷方便，一旦错误，可以快速定位，可以形成一个包。
- 启动程序快速定位问题，学会查看日志，学会`google`大法。

## 部署

- 备份，转移操作。

```shell
## jenkins
$ cd /var/lib/jenkins
$ tar zcf jenkins.tar.gz jenkins
$ scp -P6822 jenkins.tar.gz  myhost:/data/jenkins
```

- 目录结构如下：

```shell
$ mkdir /data/jenkins/data
$ touch docker-compose.yaml
$ tree -L 1
.
├── data
└── docker-compose.yaml
```

- 基于docker-compose部署：

```yaml
version: '2'
services:
  jenkins:
    image: jenkins/jenkins:latest
    container_name: jenkins
    restart: always
    ports:
      - 18000:8080
      - 50000:50000
    volumes:
      - /data/jenkins/data:/var/jenkins_home
```

- 导入数据文件

```shell
$ cd /data/jenkins
$ tar zcf jenkins.tar.gz
$ mv jenkins/users data/
$ mv jenkins/plugins data/
$ mv jenkins/config.xml data/
$ mv jenkins/jobs data/
$ mv jenkins/workspace data/
```

- 部署

```
## 调式的时候去掉-d,便于观察日志是否有报错信息,根据报错信息依次解决.
$ docker-compose up -d

## 如果去权限错误,记得chown 修改为docker的启动用户权限;实在不777大法
$ chmod -R 777 data/
```

- 创建ssh密钥，可以远程部署,这个主要是原来的ssh秘钥密码忘记了,所以重新弄了~~

```shell
## (一路enter，记得要使用密码)
$ ssh-keggen  
$ ls ~/.ssh/
id_rsa 		id_rsa.pub
## 将此密钥作为jenkins，连接远程主机执行shell脚本，实现pipeline自动化。
## 具体原理请查阅ssh原理，这里不再赘述.
```

### 总结

jenkins的部署相对简单,主要是一些插件的使用比较繁琐,需要慢慢的学习.这次迁移基本没有什么报错,一路成功.

### 参考

- [jenkins迁移](https://yq.aliyun.com/articles/346685)