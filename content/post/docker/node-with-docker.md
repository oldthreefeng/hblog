---
title: "node项目docker化"
date: 2019-12-05T17:42:18+08:00
lastmod: 2019-12-05T17:42:18+08:00
tags: [docker,go]
categories: [docker]
---

## node项目docker化

>  docker是一个开源的应用容器引擎，可以为我们提供安全、可移植、可重复的自动化部署的方式。docker采用虚拟化的技术来虚拟化出应用程序的运行环境。如上图一样。docker就像一艘轮船。而轮船上面的每个小箱子可以看成我们需要部署的一个个应用。使用docker可以充分利用服务器的系统资源，简化了自动化部署和运维的繁琐流程,减少很多因为开发环境中和生产环境中的不同引发的异常问题。从而提高生产力。 

首先, 创建一个`node`项目, 公司的项目由`pm2`管理, 故此篇文章主要记录如何以`pm2`为基础镜像管理公司项目.

```bash
$ tree -L 1
.
├── assets
├── ava.config.js
├── components
├── Dockerfile    ##Dockerfile
├── docs
├── gulpfile.js
├── hudong-web-msite.zip
├── layouts
├── middleware
├── nuxt.config.js
├── package.json
├── pages
├── plugins
├── README.md
├── server
├── start.config.js   ## 入口配置文件
├── static
├── store
└── test
```

### 创建`pm2`基础镜像

node我们选择的版本是` ENV NODE_VERSION=10.17.0 `, 选用的alpine镜像

```
FROM node:10-alpine
MAINTAINER louis <louis@wangke.co>
RUN npm install pm2 -g --registry=https://registry.npm.taobao.org
```

创建基础镜像

```bash
$ docker build -t louisehong/pm2-alpine .
```

### docker化`pm2`项目

平时我们部署项目的时候,  工程启动时加入一些参数，可以修改某些配置，增加部署的灵活性。 比较常见的有`env  test/dev/prev/prod`等环境配置.

如

```bash
$ pm2 start start.config.js --env prod
$ pm2 start start.config.js --env prev
$ pm2 start  start.config.js --env dev # 默认为dev环境
```

 表示node启动的配置文件时`prod/prev/dev`环境配置文件 

```
# 定义环境变量，会被后续的RUN命令使用，并且在容器运行期间保持
# 配置文件参数，默认为prev环境
FROM louisehong/pm2-alpine
MAINTAINER louis <louis@wangke.co>
WORKDIR /app
COPY . .
ENV NPM_CONFIG_LOGLEVEL warn
ENV PHASE="prev"
RUN npm install --registry=https://registry.npm.taobao.org && \
    npm run build
EXPOSE 22802
ENTRYPOINT  ["sh","-c", "pm2-docker start start.config.js --env $PHASE"]
```

创建项目后, 在docker启动的时候, 输入环境变量即可

```bash
$ docker build -t louiswz:laster .
$ docker run -d -p 22802:22802 \
  --name louiswz -e PHASE="prod" louisehong/louiswz
```

### 踩坑区

使用CMD命令的时候, 传入环境变量不生效.

```
CMD ["pm2-docker", "start", "start.config.json","--env","$PHASE"]
```

