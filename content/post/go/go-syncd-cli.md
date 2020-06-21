---
title: "syncd-cli 介绍及用法"
date: 2019-10-20T15:54:18+08:00
lastmod: 2019-10-21T11:54:18+08:00
keywords: []
description: ""
tags: [go,gin,flag,syncd,gorequest,logrus]
categories: [syncd]
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

<!--more-->

## 介绍

> syncd-cli 是自动化部署工具 [syncd](https://github.com/dreamans/syncd) 的一个命令行客户端，用于批量添加server，实现一键自动添加，提高开发效率。

> [Syncd](https://github.com/dreamans/syncd)是一款开源的代码部署工具，它具有简单、高效、易用等特点，可以提高团队的工作效率。
  

## 安装

required要求

- [X] go1.8+
- [X] syncd2.0+

`go get` 方式

```shell
$ go get gogs.wangke.co/go/syncd-cli
$ syncd-cli -h
```

`git clone` 方式

```shell
$ git clone https://gogs.wangke.co/go/syncd-cli.git
$ cd syncd-cli && go build -o syncd-cli syncd-cli.go
$ ./syncd-cli -h
```

## Usage
### @v1.0.0

```shell
# ./syncd-cli -h                                                                                                  [12:18:53]
syncd-cli version:1.0.0
Usage syncd-cli <command> [-aupginsh]

command [--add] [--list]  [user|server]

add server example:
        1) syncd-cli -d server -g 2 -i 192.168.1.1,test.example.com -n test01,test02 -s 9527,22
        2) syncd-cli --add server  --roleGroupId 2 --ipEmail 192.168.1.1 --names test01 --sshPort 9527
add user example:
        1) syncd-cli --add user  --ipEmail text@wangke.co --names test01
        2) syncd-cli  -d user  -i text@wangke.co -n test01
list server and user example:
        1) syncd-cli -l user
        2) syncd-cli -l server 
        3) syncd-cli --list 
        4) syncd-cli --list server 

Options:
  -d, --add string        add user or server
  -h, --help              this help
  -a, --hostApi string    sycnd server addr api (default "http//127.0.0.1:8878/")
  -i, --ipEmail strings   set ip/hostname to the cluster with names // or email for add user, use ',' to split
  -l, --list string       list server or user
  -n, --names strings     set names to the cluster with ips, use ',' to split
  -p, --password string   password for syncd tools (default "111111")
  -g, --roleGroupId int   group_id for cluster // or role_id for user, must be needed (default 1)
  -s, --sshPort ints      set sshPort to the cluster server, use ',' to split
  -u, --user string       user for syncd tools (default "syncd")

```

### @v1.1.0

从命令行读取批量文件的信息,感觉太冗余了, 不方便创建,还是以文件的方式创建批量容易一点,
所用想了一个从文件里面读取信息,然后根据信息创建相应资源的方法.

```shell
$ ./syncd-cli -h
syncd-cli version:1.1.0
Usage syncd-cli <command> [-afhpu]

command <apply>|<get>  [user|server] <?-f files>

add server example:
        1) syncd-cli apply user -f files
        2) syncd-cli apply server -f files
list server and user example:
        1) syncd-cli get user
        2) syncd-cli get server

Options:
  -f, --file string       add server/user from files
  -h, --help              this help
  -a, --hostApi string    sycnd server addr api (default "http://127.0.0.1:8878/")
  -p, --password string   password for syncd tools (default "111111")
  -u, --user string       user for syncd tools (default "syncd")

```

**file文件的格式**

`testserver`,`testuser`等文件名称无要求, 但是对文件的格式要求.以一个空格进行分割.

```shell
# 第一列是groupid,对应的集群id;
# 第二列是name, 对应的是server的名字
# 第三列是ip/hostname, 对应server的ip或者域名
# 第四列是sshport, 对应的是server的ssh端口
$ cat testserver
1 test01 test.wangke.co 22
1 test02 test01.wangke.co 9527
1 test03 test02.wangke.co 6822
```
`testuser`文件内容以空格区分,共四列.(后续可以添加至6列,源码的user还有电话号码,真实姓名等,非必须 批量创建的默认密码为`111111`)

```shell
# 第一列是role_id, 对应的是角色, 比如1是管理员
# 第二列是name, 对应的是用户名
# 第三列是email, 对应的是用户的邮箱
# 第四列是status, 对应的是用户能否登陆.
$ cat testuser
1 test01 test01@wangke.co 1
1 test02 test02@wangke.co 1

```

因为`testuser`和`testserver`的的文件格式和数据类型是一样的, 所用到的方法是一样的, 唯一的区分就是利用`apply user`还是`apply server`

```go
type server struct {
    id int
    name string
    ip string
    port int
}

type user struct {
	id int
	name string
	email string
	status int
}

```
### 重要提醒
```

方法是一样的. 所以标志位很重要, 不然创建错了就是连环错误了.

$ syncd-cli apply user -f testuser
$ syncd-cli apply server -f testserver
```

## Example
``` 
root@master-louis: ~/go/src/github.com/oldthreefeng/syncd-cli master ⚡
# ./syncd-cli -i 192.168.1.2,text.example.com -n test1,texte -s 9527,22              [12:18:58]
INFO[0000] your token is under .syncd-token             
INFO[0000] group_id=1&name=test1&ip=192.168.1.2&ssh_port=9527  
INFO[0000] {"code":0,"message":"success"}               
INFO[0000] group_id=1&name=texte&ip=text.example.com&ssh_port=22  
INFO[0000] {"code":0,"message":"success"}

# 将test01邮箱为text@wangke.co加入管理员,默认密码为111111
$./syncd-cli -d user -i text@wangke.co -n test01   
time="2019-10-20T17:59:08+08:00" level=info msg="your token is under .syncd-token\n"
time="2019-10-20T17:59:08+08:00" level=info msg="role_id=1&username=test01&password=1111111&email=text@wangke.co&status=1"
time="2019-10-20T17:59:08+08:00" level=info msg="{\"code\":0,\"message\":\"success\"}"

```
添加如下:
![](https://oss.fenghong.tech/syncd-cli.png)
![](https://oss.fenghong.tech/syncd-cli-add-user.png)

## 算法思路 ##

本来想开发和`kubectl`,`go`,`kubeadm`等类似的管理cli. 奈何时间水平有限. 

脑子里想的是这样的

```cgo
$ syncd get user 
$ syncd get server 
$ syncd apply -f adduser.yaml
```
实际上...

```cgo
$ syncd-cli --list user
$ syncd-cli --list server

$ syncd-cli --add user -i test@wangke.co -n test01 
```

> 整体上, 利用`http`的`GET`还有`POST`完成显示和添加动作的. [gorequest](https://github.com/parnurzeal/gorequest)的`GET/POST`的确好用,可以试试. 
>
> 记录日志当然是用的[logrus](https://github.com/sirupsen/logrus), 当时用的`go mod`学习教程就是用的这个模板, 日志的格式也可以.
>
> 命令行的开发主要就是用的[pflag](https://github.com/spf13/pflag), 看了`kubernetes`和`docker`源码相关, `kubectl`等命令行管理工具也是基于这个开发的.

首先, 登录验证, 获取token, 将token存入当前目录下的`.syncd-token`, 其次, 获取user/server列表或者添加user/server, 逻辑都是一样的,发送`POST`请求, 同时携带cookie, 将cookie的`name`和`value`封装成`http.cookie`, 每次需要用到,直接调用即可. 

## 代码 ##

```go
/*
Copyright 2019 louis.
@Time : 2019/10/20 10:00
@Author : louis
@File : syncd-cli
@Software: GoLand

*/

package main

import (
	"bufio"
	"crypto/md5"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"github.com/parnurzeal/gorequest"
	log "github.com/sirupsen/logrus"
	flag "github.com/spf13/pflag"
	"io/ioutil"
	"net/http"
	"os"
)

const (
	tokenFile = ".syncd-token"
	agent     = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:68.0) Gecko/20100101 Firefox/68.0"
)

var (
	//host     = "https://syncd.fenghong.tech/"
	host     string
	list     string
	user     string
	password string
	GroupId  int
	Names    []string
	Ips      []string
	SSHPort  []int
	add      string
	files    string
	h        bool
)

var _token string

func TokenFail() {
	RemoveToken()
	panic(fmt.Sprintf("login faild, please set the right password"))
}

func RemoveToken() {
	if err := os.Remove(tokenFile); err != nil {
		log.Infoln("remove .token failed")
	}
}

func SetToken(token string) {
	err := ioutil.WriteFile(tokenFile, []byte(token), 0644)
	if err != nil {
		log.Fatalln(err)
	}
	_token = token
}

func GetToken() string {
	if _token == "" {
		tokenByte, err := ioutil.ReadFile(tokenFile)
		if err != nil {
			log.Fatalln("need login")
		}

		_token = string(tokenByte)
	}
	return _token
}

func md5s(s string) string {
	h := md5.New()
	h.Write([]byte(s))
	return hex.EncodeToString(h.Sum(nil))
}

type RespData map[string]interface{}

type Response struct {
	Code    int      `json:"code"`
	Message string   `json:"message"`
	Data    RespData `json:"data"`
}

func listServerDetail(res RespData) {
	for _, v := range res {
		fmt.Println(v)
	}
}

func ParseResponse(respBody string) (RespData, error) {
	response := Response{}
	err := json.Unmarshal([]byte(respBody), &response)
	if err != nil {
		panic(err)
	}

	if response.Code == 1005 {
		TokenFail()
	}

	if response.Code != 0 {
		return nil, errors.New(response.Message)
	}

	return response.Data, nil
}

func login(user, password string) {
	url := host + "api/login"
	_, _, errs := gorequest.New().
		Post(url).
		Type("form").
		AppendHeader("Accept", "application/json").
		Send(fmt.Sprintf("username=%s&password=%s", user, md5s(password))).
		End(func(response gorequest.Response, body string, errs []error) {
			if response.StatusCode != 200 {
				panic(fmt.Sprintf("%s", errs))
			}

			respData, err := ParseResponse(body)
			if err != nil {
				panic(err)
			}

			//respData
			SetToken(respData["token"].(string))
		})

	if errs != nil {
		log.Fatalf("%s", errs)
	}
	log.Infof("your token is under %s\n", tokenFile)
}

func userAdd(roleId int, userName, email string, status int) {
	url := host + "api/user/add"
	pass := "111111"
	_, body, errs := gorequest.New().Post(url).
		AppendHeader("Accept", "application/json").
		AppendHeader("User-Agent", agent).
		AddCookie(authCookie()).
		Send(fmt.Sprintf("role_id=%d&username=%s&password=%s&email=%s&status=%d",
			roleId, userName, md5s(pass), email, status)).
		End(func(response gorequest.Response, body string, errs []error) {
			if response.StatusCode != 200 {
				panic(errs)
			}
		})
	if errs != nil {
		log.Fatalln(errs)
	}
	log.Infof("role_id=%d&username=%s&password=%s&email=%s&status=%d",
		roleId, userName, pass, email, status)
	log.Infoln(body)
}

func serverAdd(groupId int, name, ip string, sshPort int) {
	url := host + "api/server/add"
	_, body, errs := gorequest.New().Post(url).
		AppendHeader("Accept", "application/json").
		AppendHeader("User-Agent", agent).
		AddCookie(authCookie()).
		Send(fmt.Sprintf("group_id=%d&name=%s&ip=%s&ssh_port=%d",
			groupId, name, ip, sshPort)).
		End(func(response gorequest.Response, body string, errs []error) {
			if response.StatusCode != 200 {
				panic(errs)
			}
		})
	if errs != nil {
		log.Fatalln(errs)
	}
	log.Infof("group_id=%d&name=%s&ip=%s&ssh_port=%d\n",
		groupId, name, ip, sshPort)
	log.Infoln(body)
}

type QueryBind struct {
	Keyword string `form:"keyword"`
	Offset  int    `form:"offset"`
	Limit   int    `form:"limit" binding:"required,gte=1,lte=999"`
}

func List(api string) {
	url := host + api
	_, body, errs := gorequest.New().Get(url).Query(QueryBind{Keyword: "", Offset: 0, Limit: 7}).
		AppendHeader("Accept", "application/json").
		AppendHeader("User-Agent", agent).
		AddCookie(authCookie()).
		End()
	if errs != nil {
		log.Fatalln(errs)
	}
	var serverBody Response
	err := json.Unmarshal([]byte(body), &serverBody)
	if err != nil {
		log.Fatalln(err)
	}
	//log.Infoln(serverBody)
	listServerDetail(serverBody.Data)
}

func authCookie() *http.Cookie {
	cookie := http.Cookie{}
	cookie.Name = "_syd_identity"
	cookie.Value = GetToken()
	return &cookie
}

func usages() {
	_, _ = fmt.Fprintf(os.Stderr, `syncd-cli version:1.1.0
Usage syncd-cli <command> [-afhpu] 

command <apply|get>  <user|server> [?-f files]

add server example: 
	1) syncd-cli apply user -f files
	2) syncd-cli apply server -f files
list server and user example:
	1) syncd-cli get user
	2) syncd-cli get server

Options:
`)
	flag.PrintDefaults()
}

func init() {
	flag.StringVarP(&host, "hostApi", "a", "http://127.0.0.1:8878/", "sycnd server addr api")
	//flag.StringVarP(&host, "hostApi", "a", "https://syncd.fenghong.tech/", "sycnd server addr api")
	flag.StringVarP(&user, "user", "u", "syncd", "user for syncd tools")
	flag.StringVarP(&password, "password", "p", "111111", "password for syncd tools")

	flag.StringVarP(&add, "add", "d", "", "add user or server(deprecated)")
	flag.StringVarP(&files, "file", "f", "", "add server/user from files")
	flag.StringVarP(&list, "list", "l", "", "list server and user(deprecated)")
	flag.IntVarP(&GroupId, "roleGroupId", "g", 1, "group_id for cluster // or role_id for user, must be needed(deprecated)")
	flag.StringSliceVarP(&Ips, "ipEmail", "i", []string{""}, "set ip/hostname to the cluster with names // or email for add user, use ',' to split(deprecated)")
	flag.StringSliceVarP(&Names, "names", "n", []string{""}, "set names to the cluster with ips, use ',' to split(deprecated)")
	flag.IntSliceVarP(&SSHPort, "sshPort", "s", []int{}, "set sshPort to the cluster, use ',' to split(deprecated)")
	flag.BoolVarP(&h, "help", "h", false, "this help")
	flag.Usage = usages
}

func useV100() {
	//是否列出server,user

	switch list {
	case "user":
		List("api/user/list")
	case "server":
		List("api/server/list")
	default:
		fmt.Println("use `syncd-cli get [user | server]` instead")
	}

	if Ips[0] == "" || Names[0] == "" || SSHPort[0] == 0 {
		return
	}
	switch add {
	case "user":
		// userAdd() //easy to add
		for k, v := range Ips {
			userAdd(GroupId, Names[k], v, 1)
		}
	case "server":
		// Ips未指定,则返回
		for k, v := range Ips {
			serverAdd(GroupId, Names[k], v, SSHPort[k])
		}
	}
}

type server struct {
	gid  int
	name string
	ip   string
	port int
}

func readFromServerFile(file string) []server {
	openFile, err := os.Open(file)
	if err != nil {
		log.Fatalf("%v",err)
	}
	defer openFile.Close()
	var newserver []server
	scanner := bufio.NewScanner(openFile)
	scanner.Split(bufio.ScanLines)
	for scanner.Scan() {
		line := scanner.Text()

		if len(line) == 0 {
			break
		}
		var gid, port int
		var name, ip string
		_, err := fmt.Sscanf(line, "%d %s %s %d", &gid, &name, &ip, &port)
		if err != nil {
			return nil
		}
		newserver = append(newserver, server{
			gid:  gid,
			name: name,
			ip:   ip,
			port: port,
		})
	}
	return newserver
}

func main() {
	flag.Parse()
	if h {
		flag.Usage()
		return
	}

	// 登录认证
	login(user, password)

	// 使用v1.0.0
	if list != "" {
		useV100()
	}

	switch os.Args[1] {
	case "apply":
		switch os.Args[2] {
		case "server":
			ser := readFromServerFile(files)
			for _, v := range ser {
				serverAdd(v.gid, v.name, v.ip, v.port)
			}
		case "user":
			usr := readFromServerFile(files)
			for _, v := range usr {
				// 偷个懒, 数据类型一样,结构一样, 所以从文件读取的是一样的
				// v.gid ==> roleId
				// v.name==> username
				// v.ip  ==> email
				// v.port==> status
				userAdd(v.gid, v.name, v.ip, v.port)
			}
		default:
			fmt.Println("syncd-cli apply [user|server] -f files")
		}

	case "get":
		switch os.Args[2] {
		case "user":
			List("api/user/list")
		case "server":
			List("api/server/list")
		default:
			fmt.Println("syncd-cli get [user | server]")
		}
	default:
		fmt.Println()
		fmt.Println("	Use syncd-cli@v1.1.0 instead")
		fmt.Println("syncd-cli <get|apply> <user|server> [?-f filename>]")
		fmt.Println("syncd-cli <get|apply> <user|server> [?--file filename]")
	}

}

```
