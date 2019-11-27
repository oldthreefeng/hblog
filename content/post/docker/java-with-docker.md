---
title: "java的spring-boot项目docker化"
date: 2019-11-23T15:42:18+08:00
lastmod: 2019-11-23T21:54:18+08:00
tags: [docker,java,jvm]
categories: [docker]
---
## 创建spring-boot项目

使用spring-boot项目的pom.xml

mvn打包使用jar包方式

java的基础运行环境, 做一个基础的jre-alpine镜像,

添加相关工具包,使得jer镜像环境体积最小 

```
FROM openjdk:8-jre-alpine
MAINTAINER "louis <louis.hong@junhsue.com>"

## 工作目录为opt. 需要用到bash和zip包
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
	&& apk add --update bash &&  apk add zip \
	&& rm -rf /var/cache/apk/*   \
	&& mkdir /opt/config
```

构建spring-boot项目镜像

```
FROM louisehong/jre-8-alpine-bash
MAINTAINER "louis <louis.hong@junhsue.com>"

WORKDIR /opt
ENV PHASE=qa
ADD target/weimall-api-single-*.jar app.jar
ADD entrypoint.sh entrypoint.sh
EXPOSE 8888
ENTRYPOINT ["./entrypoint.sh"]
```

`entrypoint.sh`如下:

```bash
#!/bin/sh
#
set -x 
# 默认为qa环境.
PHASE=${PHASE:-"qa"}
cd /opt && \
	unzip -o -j app.jar  BOOT-INF/classes/application-${PHASE}.yml -d config/ && \
	# sed -i "s/PORT/$PORT/g" application-${PHASE}.yml
	java -jar /opt/app.jar -server --spring.profiles.active=${PHASE} $JAVA_OPTS
```

MVN打包

```bash
$ mvn clean package  -Dmaven.test.skip=true
```

项目启动

```bash
$ docer run --name  $JOB_BASE_NAME   --restart=always -d -p 8888:8888  -e PHASE= 
```