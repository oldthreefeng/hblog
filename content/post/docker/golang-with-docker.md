---
title: "golang项目docker化"
date: 2019-11-21T15:42:18+08:00
lastmod: 2019-11-25T10:54:18+08:00
tags: [docker,go]
categories: [docker]
---

### golang 项目 docker化

Golang 支持交叉编译，在一个平台上生成另一个平台的可执行程序 

最终的`dockerfile`  踩坑过程记录。

```
FROM golang:alpine as builder

WORKDIR /go/src/service-msite
ENV GOPROXY=https://goproxy.cn
ENV GO111MODULE=on
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
  && apk update \
  && apk add git \
  && apk add gcc \
  && apk add libc-dev
  
COPY go.mod go.sum ./
RUN go mod download
COPY ../go .

RUN  GOOS=linux GOARCH=amd64 go build -ldflags "-extldflags -static  -X 'main.Buildstamp=`date -u '+%Y-%m-%d %I:%M:%S%p'`' -X 'main.Githash=`git rev-parse HEAD`' -X 'main.Goversion=`go version`'" -o /service-msite 

FROM alpine

WORKDIR /opt/
RUN apk add tzdata ca-certificates && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && apk del tzdata && rm -rf /var/cache/apk/* 
COPY --from=builder /service-msite  .
COPY --from=builder /go/src/service-msite/conf.d/  conf.d

EXPOSE 22902 22903
ENTRYPOINT ["/opt/service-msite"]
```

### 编译golang项目存在C语言之踩坑

一开始， 我不知道项目有需要用到`gcc`编译器的, 直接` CGO_ENABLED=0`, 报错如下， 发现不了源码。很纳闷~和开发沟通后. 

```bash
$ docker build -t service-msite:0 .
Step 9/16 : RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -o /service-msite
 ---> Running in 65f75673ef1e
build service-msite: cannot load service-msite/routers/controllers/exhibitor: no Go source files
```

在`dockerfile`中添加了 `apk add gcc` ,  重新`build`,发现用到了七牛的[iconv](github.com/qiniu/iconv),  还是爆出了compilation错误`iconv.h: No such file or directory`, 发现不了源码文件.

```bash
$ docker build -t service-msite:0 .
Step 9/16 : RUN GOOS=linux GOARCH=amd64 go build  -o /service-msite
 ---> Running in 3b5479d74b99
# github.com/qiniu/iconv
/go/pkg/mod/github.com/qiniu/iconv@v0.0.0-20160413084200-e9ee0ddd1a3a/iconv.go:9:11: fatal error: iconv.h: No such file or directory
 // #include <iconv.h>
           ^~~~~~~~~
compilation terminated.
# service-msite/routers/controllers/exhibitor
_cgo_export.c:3:10: fatal error: stdlib.h: No such file or directory
 #include <stdlib.h>
          ^~~~~~~~~~
compilation terminated.
The command '/bin/sh -c GOOS=linux GOARCH=amd64 go build -o /service-msite' returned a non-zero code: 2
```

查询相关文档后, 发现在`golang:alpine`镜像中,libc是要单独安装的`apk add libc-dev`, 在dockerfile中加入后, 重新编写镜像

```bash
$ docker build -t service-msite:0 .
Step 13/16 : COPY --from=builder /service-msite .
 ---> 42edaa10cbe7
Removing intermediate container 11abbb3e796b
Step 14/16 : COPY --from=builder /go/src/service-msite/conf.d/ conf.d
 ---> eef43e376975
Removing intermediate container 29aac52c5592
Step 15/16 : EXPOSE 22902 22903
 ---> Running in 560b460ad2e4
 ---> 388ab5135e69
Removing intermediate container 560b460ad2e4
Step 16/16 : ENTRYPOINT /opt/service-msite
 ---> Running in 2e7364eb8443
 ---> 5704e7239be8
Removing intermediate container 2e7364eb8443
Successfully built 5704e7239be8
Successfully tagged service-msite:0

```

完美编译成功.

### docker部署踩坑

启动刚build的docker镜像. 没有发现文件`service-msite`二进制文件.

```bash
$ docker run -p 22902:22902 -p 22903:22903  service-msite:0
standard_init_linux.go:190: exec user process caused "no such file or directory"
```

>There are a number of reasons that folks are in love with golang. One the most mentioned is the static linking.

> As long as the source being compiled is native go, the go compiler will statically link the executable. Though when you need to use [cgo](http://golang.org/cmd/cgo/), then the compiler has to use its external linker.

在宿主机上进行` GOOS=linux GOARCH=amd64 go build `完全可以运行, 在docker里面竟然执行不了. 百度之后, 发现`golang`的魅力在于静态链接. 一个是静态链接，一个是动态链接，动态链接的在微型镜像alpine上不支持。 

```bash
$ GOOS=linux GOARCH=amd64 go build
$ file service-msite 
service-msite: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, not stripped
$ ldd service-msite 
	linux-vdso.so.1 =>  (0x00007ffd00159000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fc75d00f000)
	libc.so.6 => /lib64/libc.so.6 (0x00007fc75cc41000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fc75d22b000)
```
加上`-ldflags "-extldflags -static"` 标签, 里面变成静态链接

```bash
$ GOOS=linux GOARCH=amd64 go build -ldflags "-extldflags -static"
$ file service-msite 
service-msite: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.32, not stripped
$ ldd service-msite 
      not a dynamic executable
```

完美收官, docker和golang的项目, 真的是匹配~

### 附言

附带编译打包是的源码

```bash
$ go build -ldfalgs "-X 'main.Buildstamp=`date -u '+%Y-%m-%d %I:%M:%S%p'`' -X 'main.Githash=`git rev-parse HEAD`' -X 'main.Goversion=`go version`'"
```
很巧妙的编译. 记录学习一下.

```go
package main

import (
	"flag"
	"fmt"
)

const Version = "1.4.1"

var (
	Buildstamp = ""
	Githash    = ""
	Goversion  = ""
    infoFlag   = false
)
func init() {
    flag.BoolVar(&infoFlag, "V", false, "version info")
}
func main () {
	flag.Parse()
	if infoFlag {
		fmt.Printf("Version:             %s\n", Version)
		fmt.Printf("Git Commit Hash:     %s\n", Githash)
		fmt.Printf("UTC Build Time :     %s\n", Buildstamp)
		fmt.Printf("Go Version:          %s\n", Goversion)
		return
	}
    ...
}

```

结合jenkins做发布

```bash
Started by user louishong
Running as SYSTEM
Building in workspace /root/.jenkins/workspace/test-service-msite
No credentials specified
 > /usr/bin/git submodule update --init --recursive proto-user
[test-service-msite] $ /bin/sh -xe /tmp/jenkins4336241043767081707.sh
+ cd /root/.jenkins/workspace/test-service-msite
+ cat
+ docker build -t harbor.youpenglai.cn/hudong/service-msite:35 .
Sending build context to Docker daemon  79.74MB

Step 1/16 : FROM golang:alpine as builder
 ---> 3024b4e742b0
Step 2/16 : WORKDIR /go/src/service-msite
 ---> Using cache
 ---> ce0f39f06f4d
Step 3/16 : ENV GOPROXY https://goproxy.cn
 ---> Using cache
 ---> 5a648682b413
Step 4/16 : ENV GO111MODULE on
 ---> Using cache
 ---> c07567032634
Step 5/16 : RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories   && apk update   && apk add git   && apk add gcc   && apk add libc-dev
 ---> Using cache
 ---> 148ce213a987
Step 6/16 : COPY go.mod go.sum ./
 ---> Using cache
 ---> 07bb01709b53
Step 7/16 : RUN go mod download
 ---> Using cache
 ---> d30737f26923
Step 8/16 : COPY . .
 ---> Using cache
 ---> 0a374d967956
Step 9/16 : RUN GOOS=linux GOARCH=amd64 go build -ldflags "-extldflags -static" -x -o /service-msite
 ---> Using cache
 ---> 386768cecb3b
Step 10/16 : FROM alpine
 ---> 965ea09ff2eb
Step 11/16 : WORKDIR /opt/
 ---> Using cache
 ---> 1f76f5442286
Step 12/16 : RUN apk add tzdata ca-certificates && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime     && echo "Asia/Shanghai" > /etc/timezone     && apk del tzdata && rm -rf /var/cache/apk/*
 ---> Using cache
 ---> 2a9386bcf1a8
Step 13/16 : COPY --from=builder /service-msite .
 ---> Using cache
 ---> ec46dc96dcad
Step 14/16 : COPY --from=builder /go/src/service-msite/conf.d/ conf.d
 ---> Using cache
 ---> 97b266fab23d
Step 15/16 : EXPOSE 22902 22903
 ---> Using cache
 ---> efa77439cb11
Step 16/16 : ENTRYPOINT /opt/service-msite
 ---> Using cache
 ---> 2019ce457320
Successfully built 2019ce457320
Successfully tagged harbor.youpenglai.cn/hudong/service-msite:35
+ docker push harbor.youpenglai.cn/hudong/service-msite:35
The push refers to a repository [harbor.youpenglai.cn/hudong/service-msite]
fe2f34e1bae5: Preparing
bea1df178045: Preparing
b702ba588b63: Preparing
77cae8ab23bf: Preparing
b702ba588b63: Layer already exists
77cae8ab23bf: Layer already exists
fe2f34e1bae5: Layer already exists
bea1df178045: Layer already exists
35: digest: sha256:4ce369a7729cce6c71831379e303385b36e2f177fa89675a3f1f83e107a33834 size: 1158
[SSH] script:
JOB_BASE_NAME="test-service-msite"
BUILD_NUMBER="35"

docker pull harbor.youpenglai.cn/hudong/service-msite:${BUILD_NUMBER}
docker stop  $JOB_BASE_NAME 
docker rm   $JOB_BASE_NAME 
docker run --name  $JOB_BASE_NAME   --restart=always  -d   -p  22902:22902 -p 22903:22903  -e MFW_CONSUL_IP=192.168.0.89  harbor.youpenglai.cn/hudong/service-msite:${BUILD_NUMBER}


[SSH] executing...
Error response from daemon: no such id: test-service-msite
Error: failed to stop containers: [test-service-msite]
Error response from daemon: no such id: test-service-msite
Error: failed to remove containers: [test-service-msite]
35: Pulling from harbor.youpenglai.cn/hudong/service-msite
9449b80f3b71: Already exists
c806779f19ea: Already exists
4bec7ab1dd18: Already exists
ef08506cfaf7: Already exists
e4b17e4ffd21: Already exists
a8cfba2ab5a7: Already exists
aa70e406cd52: Already exists
188ab4fb4e1e: Already exists
Digest: sha256:eb175e1ac8fbbeffe47cf02d00fd23941664683b245ba45ee4d3268388885267
Status: Downloaded newer image for harbor.youpenglai.cn/hudong/service-msite:35
3f509ec0d74fc067af9dd37d3e24fa2f34f3ccdf0aa0128fccd6eb075c59f490
[SSH] completed
[SSH] exit-status: 0

Finished: SUCCESS
```
查看容器状态. 成功部署.

```
$ docker ps
CONTAINER ID        IMAGE                                          COMMAND                CREATED             STATUS              PORTS                                  NAMES
3f509ec0d74f        harbor.youpenglai.cn/hudong/service-msite:35   "/opt/service-msite"   10 seconds ago      Up 9 seconds        0.0.0.0:22902-22903->22902-22903/tcp   test-service-msite
$ ss -ntl |grep 22902
LISTEN     0      128                      :::22902                   :::*     
LISTEN     0      128                      :::22902                   :::*
```

