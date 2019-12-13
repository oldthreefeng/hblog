---
title: "A cli tool to deploy ansible-playbook"
date: "2019-12-13 09:59:32"
tags: [ops,ansible,golang,cobra]
categories: [ops]
---

## ango

基于golang开发的一个用于部署项目至生产环境的部署工具

目前仅使用playbook部署/回滚相关业务并使用钉钉的`webhook`通知, 文档查看: https://github.com/oldthreefeng/ango

## Required

- `go version go1.13.4 linux/amd64`
- `export GO111MODULE="on"`
- `ansible2.6+`
- `.yml is ready to go`

## Usage

### download and compile

use git to download source code

```bash
$ git clone https://github.com/oldthreefeng/ango.git
$ cd ango && go mod download

# Linux
$ make linux
# darwin
$ make darwin

$ ./ango
ango is cli tools to running Ansible playbooks from Golang.
run "ango -h" get more help, more see https://github.com/oldthreefeng/ango
ango version :       1.0.0
Git Commit Hash:     a9a3c28
UTC Build Time :     2019-12-13 04:16:36 UTC
Go Version:          go version go1.13.4 linux/amd64
Author :             louis.hong
```

use go get 

```bash
$ go get github.com/oldthreefeng/ango
```

### run with palybook

first, to config your ansible, more to see [ansible](https://github.com/ansible/ansible)

```bash
$ vim /etc/ansible/hosts
[test]
192.168.0.62
192.168.0.63
$ ansible test -m ping
192.168.0.62 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
192.168.0.63 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

second, to export some env to hook to Dingding

```bash
$ export DingDingMobiles="158****6468"
$ export DingDingUrl="https://oapi.dingtalk.com/robot/send?access_token=*****"
```

third, to deploy your project

```bash
$ ango deploy -f test.yml -v v1.23  
## It's equal to  `ansible-playbook test.yml -e version=v1.23 -f 1`
## and to Post a test Message to Dingding

$ ango deploy -h 
use ango to deploy project with webhook to dingding

Usage:
  ango deploy [flags]

Examples:
  ango deploy -f api.yml -t v1.2.0

Flags:
  -m, --comments string   add comments when send message to dingding
  -h, --help              help for deploy

Global Flags:
      --author string     author name for copyright attribution (default "louis.hong")
  -f, --filename string   ansible-playbook for yml config(requried)
  -t, --tags string       tags for the project version(requried)
  -v, --verbose           verbose mode to see more detail infomation
```
fourth, to rollback your project 

```
$ ango rollback -f test.yml -t v1.2 --real

## 
rollback 回退版本, 需要指定回退版本的yml文件及要回退的version

Usage:
  ango rollback [flags]

Examples:
  ango rollback -f roll_api.yml -t v1.2  -r 

Flags:
  -h, --help   help for rollback
  -r, --real   really to rollback this version

Global Flags:
      --author string     author name for copyright attribution (default "louis.hong")
  -f, --filename string   ansible-playbook for yml config(requried)
  -t, --tags string       tags for the project version(requried)
  -v, --verbose           verbose mode to see more detail infomation
```

### logs

```bash 
# 查看发布日志
# -r is requried when rollback
$ ango rollback  -f test.yml -t v1.2.0  -r
$ ango deploy -f test.yml -t v1.4.0
$ tail -f fabu.log
[INFO] 2019-12-12 18:36:29 test-v1.2 回滚成功
[INFO] 2019-12-12 18:37:00 test-v1.4 部署成功
```

## 实现细节
golang调用shell执行ansible-playbook, 记录操作日志,并将执行结果钉钉通知到钉钉群

```
/*
 * Copyright (c) 2019. The ango Authors. All rights reserved.
 * Use of this source code is governed by a MIT-style
 * license that can be found in the LICENSE file.
 */

package cmd

import (
	"fmt"
	"github.com/oldthreefeng/ango/play"
	"os"
	"os/exec"
	"strings"
	"time"
)

var (
	DingDingUrl string = os.Getenv("DingDingUrl")
	AllMo string = os.Getenv("DingDingMobiles")
)

const (
	WeiMo  = "177*****702"
	CardMo = "137*****987"
	Adcom  = "158*****925"
)

func Exec(cmdStr, Type string) error {
	fmt.Println(cmdStr)
	// yj-admall.yml ==> yj-admall
	args := strings.Split(Config, ".")[0]
	//fmt.Printf("%s,%s", args, Config)
	cmd := exec.Command("sh", "-c", cmdStr)
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		return err
	}
	if err = cmd.Start(); err != nil {
		return err
	}
	// 读取每行输出
	for {
		tmp := make([]byte, 1024)
		_, err := stdout.Read(tmp)
		fmt.Print(string(tmp))
		if err != nil {
			break
		}
	}

	if err = cmd.Wait(); err != nil {
		return err
	}

	var t play.Text
	//t.Title = fmt.Sprintf("%s-%s", args, Tag)
	t.Text = fmt.Sprintf("%s:%s %s成功, 请测试确认\n%s", args, Tag, Type, Comments)
	if Type == "rollback" {
		t.AtMobiles = AllMo
	} else {
		switch args {
		case "api", "yj-mall", "yj-h5", "plmall", "yj-admall":
			t.AtMobiles = WeiMo
		case "adcom", "www-ypl":
			t.AtMobiles = Adcom
		case "card":
			t.AtMobiles = CardMo
		default:
			t.AtMobiles = AllMo
		}
	}

	err = t.Dingding(DingDingUrl)
	if err != nil {
		return err
	}
	return WriteToLog(Type)
}

func WriteToLog(Type string) error {
	filename := "fabu.log"
	args := strings.Split(Config, ".")[0]
	date := time.Now().Format("2006-01-02 15:04:05")
	data := fmt.Sprintf("[INFO] %s %s-%s %s成功", date, args, Tag, Type)
	var (
		f *os.File
		err error
	)
	if !IsFile(filename) {
		// 文件不存在, 则创建
		f, err = os.Create(filename)
	} else {
		// 文件存在, 则append.
		f, err = os.OpenFile(filename, os.O_APPEND|os.O_WRONLY, 0644)
	}

	if err != nil {
		return err
	}
	_, err = fmt.Fprintln(f, data)
	defer f.Close()
	return err
}

func IsFile(filePath string) bool {
	f, e := os.Stat(filePath)
	if e != nil {
		return false
	}
	return !f.IsDir()
}
```

钉钉调用

```
/*
 * Copyright (c) 2019. The ango Authors. All rights reserved.
 * Use of this source code is governed by a MIT-style
 * license that can be found in the LICENSE file.
 */
package play

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
)

const (
	TextTemplate = `{
    "msgtype": "text", 
    "text": {
        "content": "%s"
    }, 
    "at": {
        "atMobiles": [
            "%s"
        ], 
        "isAtAll": false
    }
}`
	LinkTemplate = `{
    "msgtype": "link", 
    "link": {
        "text": "%s", 
        "title": "%s", 
        "picUrl": "http://icons.iconarchive.com/icons/paomedia/small-n-flat/1024/sign-check-icon.png", //jenkins 发布的对勾
        "messageUrl": "%s"
    }
}`
	MarkTemplate = `{
     "msgtype": "markdown",
     "markdown": {
         "title":"%s",
         "text": "%s"
     },
    "at": {
        "atMobiles": [
            "%s"
        ], 
        "isAtAll": false
    }
}`
)

// 可以自己去实现方法
type Alarm interface {
	Post(Dingdingurl string) error
}

type MarkDowning struct {
	Title     string `json:"title"`
	Text      string `json:"text"`
	AtMobiles string `json:"atMobiles"` //应该是[]string,图方便,改成这个

}

type Linking struct {
	Text       string `json:"text"`
	Title      string `json:"title"`
	PicUrl     string `json:"picUrl"`
	MessageUrl string `json:"messageUrl"`
}

type Text struct {
	MarkDowning  // 公用一下, 只是没有title.
}

func (m Text) Dingding(DingDingUrl string) error {
	baseBody := fmt.Sprintf(TextTemplate, m.Text, m.AtMobiles)
	return Post(DingDingUrl,baseBody)
}

func (m MarkDowning) Dingding(DingDingUrl string) error {
	baseBody := fmt.Sprintf(MarkTemplate, m.Title, m.Text, m.AtMobiles)
	return Post(DingDingUrl,baseBody)
}

func (m Linking) Dingding(DingDingUrl string) error {
	baseBody := fmt.Sprintf(LinkTemplate, m.Title, m.Text, m.MessageUrl)
	return Post(DingDingUrl,baseBody)
}

func Post(DingDingUrl,baseBody string) error {
	req, err := http.NewRequest("POST", DingDingUrl, strings.NewReader(baseBody))
	if err != nil {
		return err
	}
	client := &http.Client{}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("User-agent", "firefox")
	resp, err := client.Do(req)
	defer resp.Body.Close()
	fmt.Println(resp.StatusCode)
	body, _ := ioutil.ReadAll(resp.Body)
	// 调试的时候打开, 因为钉钉返回的比如: 缺少json参数等等..
	fmt.Println(string(body))
	return err
}
```



[thanks to jetbrains](https://www.jetbrains.com/?from=ginuse)

![](https://www.jetbrains.com/company/brand/img/jetbrains_logo.png)
