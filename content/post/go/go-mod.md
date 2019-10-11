---
title: "go module"
date: 2019-08-28 18:10:10
tags: [module]
categories: [golang] 
---

[TOC]

# use go module easy

演示一个简单的函数,要安装一个日志依赖`github.com/Sirupsen/logrus`. 借以演示go mod的用法, 请轻喷~

```
$ cat main.go
/*
@Time : 2019/9/8 1:35
@Author : louis
@File : main
@Software: GoLand
*/

package main

import  log "github.com/Sirupsen/logrus"

func main()  {
        log.Info("this is a log log")
        log.Warn("this is an another log log")
        log.Fatal("this a bad log")
}
```
如果没有modules,安装依赖需要一个一个`go get ...`

```
go run main.go
$ go run main.go
te.go:3:9: cannot find package "github.com/Sirupsen/logrus" in any of:
        c:\go\src\github.com\Sirupsen\logrus (from $GOROOT)
        D:\Souce_Code\fenghong\go\src\github.com\Sirupsen\logrus (from $GOPATH)
```

## 使用细节
当我们使用`go mod`的时候,只需要init一下就好,但是必须在`$GOAPTH`下面,而且`GO111MODULES=on`.


### 1.不在$GOPATH下,报这个错.

```
$ go mod init

go: cannot determine module path for source directory D:\Souce_Code\fenghong\wiki\content\go (outside GOPATH, module path must be specified)

Example usage:
        'go mod init example.com/m' to initialize a v0 or v1 module
        'go mod init example.com/m/v2' to initialize a v2 module

Run 'go help mod init' for more information.
```

### 2.生成go.mod文件

```
$ cd $GOPATH/yourproject && go mod init
```
### 3.查看mod文件

```
$ cat go.mod
module gogs.wangke.co/go/algo/gomod

go 1.13
```

### 4.运行程序

这里报错了,是因为导入的包是大写的,但是项目里面go.mod里面是小写的.更改之后就好了

```
$ go run main.go
go: github.com/Sirupsen/logrus: github.com/Sirupsen/logrus@v1.4.2: parsing go.mod:
        module declares its path as: github.com/sirupsen/logrus
                but was required as: github.com/Sirupsen/logrus
$ go run main.go
time="2019-09-08T02:16:46+08:00" level=info msg="this is a log log"
time="2019-09-08T02:16:46+08:00" level=warning msg="this is an another log log"
time="2019-09-08T02:16:46+08:00" level=fatal msg="this a bad log"
exit status 1

## 4.1 下载完之后,当前目录会生成一个go.sum文件

$ cat go.sum
github.com/davecgh/go-spew v1.1.1 h1:vj9j/u1bqnvCEfJOwUhtlOARqs3+rkHYY13jYWTU97c=
github.com/davecgh/go-spew v1.1.1/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/konsorten/go-windows-terminal-sequences v1.0.1 h1:mweAR1A6xJ3oS2pRaGiHgQ4OO8tzTaLawm8vnODuwDk=
github.com/konsorten/go-windows-terminal-sequences v1.0.1/go.mod h1:T0+1ngSBFLxvqU3pZ+m/2kptfBszLMUkC4ZK/EgS/cQ=
github.com/pmezard/go-difflib v1.0.0 h1:4DBwDE0NGyQoBHbLQYPwSUPoCMWR5BEzIk/f1lZbAQM=
github.com/pmezard/go-difflib v1.0.0/go.mod h1:iKH77koFhYxTK1pcRnkKkqfTogsbg7gZNVY4sRDYZ/4=
github.com/sirupsen/logrus v1.4.2 h1:SPIRibHv4MatM3XXNO2BJeFLZwZ2LvZgfQ5+UNI2im4=
github.com/sirupsen/logrus v1.4.2/go.mod h1:tLMulIdttU9McNUspp0xgXVQah82FyeX6MwdIuYE2rE=
github.com/stretchr/objx v0.1.1/go.mod h1:HFkY916IF+rwdDfMAkV7OtwuqBVzrE8GR6GFx+wExME=
github.com/stretchr/testify v1.2.2 h1:bSDNvY7ZPG5RlJ8otE/7V6gMiyenm9RtJ7IUVIAoJ1w=
github.com/stretchr/testify v1.2.2/go.mod h1:a8OnRcib4nhh0OaRAV+Yts87kKdq0PP7pXfy6kDkUVs=
golang.org/x/sys v0.0.0-20190422165155-953cdadca894 h1:Cz4ceDQGXuKRnVBDTS23GTn/pU5OE2C0WrNTOYK1Uuc=
golang.org/x/sys v0.0.0-20190422165155-953cdadca894/go.mod h1:h1NjWce9XRLGQEsW7wpKNCjG9DtNlClVuFLEZdDNbEs=
```
### 5.mod tidy

运行go mod tidy可以自行安装依赖,这里推荐使用GOPROXY=https://gorpoxy.cn
设置好代理,直接下载被抢的依赖.在也不用担心golang.org背墙了.

```
$ go mod tidy
go: downloading golang.org/x/sys v0.0.0-20190422165155-953cdadca894
go: downloading github.com/stretchr/testify v1.2.2
go: extracting github.com/stretchr/testify v1.2.2
go: extracting golang.org/x/sys v0.0.0-20190422165155-953cdadca894
```
### 6. mod vendor

使用go mod vendor会使自己的依赖全部复制至vendor包里面
vendor依赖这边就不说了,go寻找依赖包先找vendor,再找$GOPATH再找GOROOT.

```
$ go mod vendor
$ ls
go.mod  go.sum  main.go  readme.md  vendor/

```
当我们想知道`go.sum`里面为什么会有这个依赖的时候,就可以利用`why`这个flag来进行查询了

```
$ go mod why github.com/davecgh/go-spew
# github.com/davecgh/go-spew
(main module does not need package github.com/davecgh/go-spew)

$ go mod why -m github.com/davecgh/go-spew
# github.com/davecgh/go-spew
gogs.wangke.co/go/algo/gomod
github.com/sirupsen/logrus
github.com/sirupsen/logrus.test
github.com/stretchr/testify/assert
github.com/davecgh/go-spew/spew

```

简单的用法就说到这里~

