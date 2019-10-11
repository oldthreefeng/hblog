---
title: "go ssh"
date: 2019-09-01 18:10:10
tags: 
  - go
  - ssh
categories: [algorithm] 
---

[TOC]

## ssh

>  SSH全称[Secure Shell](https://zh.wikipedia.org/wiki/Secure_Shell)是一种工作在应用层和传输层上的安全协议，能在非安全通道上建立安全通道。提供身份认证、密钥更新、数据校验、通道复用等功能，同时具有良好的可扩展性,由芬兰赫尔辛基大学研究员Tatu Ylönen,于1995年提出，其目的是用于替代非安全的Telnet、rsh、rexec等远程Shell协议。之后SSH发展了两个大版本,SSH-1和SSH-2, 开源实现OpenSSH对2者都支持。

SSH的主要特性:

- 加密: 避免数据内容泄漏
- 通信的完整性: 避免数据被篡改，以及发送或接受地址伪装(检查数据是否被篡改，数据是否来自发送者而非攻击者） SSH-2通过MD5和SHA-1实现该功能，SSH-1使用CRC-32
- 认证: 识别数据发送者和接收者身份 客户端验证SSH服务端的身份：防止攻击者仿冒SSH服务端身份，避免中介人攻击和重定向请求的攻击；OpenSSH通过在know-hosts中存储主机名和host key对服务端身份进行认证 服务端验证请求者身份：提供安全性较弱的用户密码方式，和安全性更强的per-user public-key signatures；此外SSH还支持与第三方安全服务系统的集成，如Kerberos等
- 授权: 用户访问控制
- 安全隧道: 转发或者为基于TCP/IP的回话提供加密隧道, 比如通过SSH为Telnet、FTP等提供通信安全保障，支持三种类型的Forwarding操作：Port Forwarding；X Forwarding；Agent Forwarding

通过使用SSH，你可以把所有传输的数据进行加密，这样”中间人”这种攻击方式就不可能实现了，而且也能够防止DNS欺骗和IP欺骗。使用SSH，还有一个额外的好处就是传输的数据是经过压缩的，所以可以加快传输的速度。SSH有很多功能，它既可以代替Telnet，又可以为FTP、PoP、甚至为PPP提供一个安全的”通道”。

## Go中ssh客户端实现

实现远程执行命令

```go
/*
@Time : 2019/8/30 11:22
@Author : louis
@File : ssh
@Software: GoLand
*/

package main

import (
	"fmt"
	flag "github.com/spf13/pflag"
	"golang.org/x/crypto/ssh"
	"io/ioutil"
	"log"
	"net"
	"os"
	"time"
)

type Conn struct {
	user string
	path string
	auth []ssh.AuthMethod
	addr string
}

var (
	client   *ssh.Client
	session  *ssh.Session
	password string
	host     string
	path     string
	port     int
	user     string
	help     bool
	cmd      string
)

func usage() {
	_, _ = fmt.Fprintf(os.Stderr, `sshgo version:1.0.0
Usage sshgo [-s host] [-p port] [-u user] [-a path] [-w password] [-c cmd]

example: 1) sshgo -u root -p 9527 -h 1.1.1.1 -a /path/to/id_rsa -w 123456
         2) sshgo --user root --port 9527 --host 1.1.1.1 --path /path/to/id_rsa --password 123456 

if use key to login, the password is the key password; if use user & pasword; the password is for user.

Options:
`)
	flag.PrintDefaults()
}

func init() {
	flag.StringVarP(&password, "password", "w", " ", "" +
		"if use key to login, the password is the key password; " +
		"if use user & password to login; the password is for user.")
	flag.StringVarP(&path, "path", "a", " ", "private key path")
	flag.StringVarP(&host, "host", "s", " ", "remote host addr ip")
	flag.IntVarP(&port, "port", "p", 22, "remote host port")
	flag.StringVarP(&user, "user", "u", "root", "remote host user")
	flag.StringVarP(&cmd, "cmd", "c", "pwd", "execute cmd in server")
	flag.BoolVarP(&help, "help", "h", false, "this help")
	flag.Usage = usage
}

func (c *Conn) SetConf() (err error) {
	c.addr = fmt.Sprintf("%s:%d", host, port)
	c.user = user
	c.path = path
	c.auth = make([]ssh.AuthMethod, 0)
	var method ssh.AuthMethod
	// use privatakey to login,defualt is " "
	if path != " " {
		c.auth = make([]ssh.AuthMethod, 0)
		method, err = PublicFile(path, password)
		if err != nil {
			return err
		}
        // use password & user to login
	} else {
		method = ssh.Password(password)
	}
	c.auth = append(c.auth, method)
	return nil
}

func (c *Conn) SetSession() (session *ssh.Session, err error) {
	client, err = ssh.Dial("tcp", c.addr, &ssh.ClientConfig{
		User: c.user,
		Auth: c.auth,
		//需要验证服务端，不做验证返回nil就可以，点击HostKeyCallback看源码就知道了
		HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
			return nil
		},
		Timeout: time.Second * 2,
	})
	if err != nil {
		fmt.Println(err)
		return nil, err
	}

	// create session
	if session, err = client.NewSession(); err != nil {
		return nil, err
	}

	return session, nil
}

func isFile(path string) (err error) {
	_, err = os.Stat(path)
	if err != nil {
		return err
	}
	return nil
}

//采用公钥验证,这里封装了一下,使用秘钥+密码验证
func PublicFile(privateKeyPath, password string) (method ssh.AuthMethod, err error) {
	if err = isFile(privateKeyPath); err != nil {
		return nil, err
	}
	bufKey, err := ioutil.ReadFile(privateKeyPath)
	if err != nil {
		return nil, err
	}
	bufPwd := []byte(password)
	key, err := ssh.ParsePrivateKeyWithPassphrase(bufKey, bufPwd)
	if err != nil {
		return nil, err
	}
	return ssh.PublicKeys(key), nil
}



func main() {
	flag.Parse()
	if help {
		flag.Usage()
		return
	}
	c := Conn{}
	err := c.SetConf()
	if err != nil {
		log.Fatalln(err)
	}
	session, err = c.SetSession()
	if err != nil {
		log.Fatalln(err)
	}
	defer session.Close()

	session.Stdout = os.Stdout
	session.Stderr = os.Stderr
	_ = session.Run(cmd)
}
```

- 实现交互命令

远程执行命令和交互命令认证过程都一样,唯一区别的是会话

```go

func main() {
	flag.Parse()
	if help {
		flag.Usage()
		return
	}
	c := Conn{}
	err := c.SetConf()
	if err != nil {
		log.Fatalln(err)
	}
	session, err = c.SetSession()
	if err != nil {
		log.Fatalln(err)
	}
	defer session.Close()
	//当ssh连接建立过后, 我们就可以通过这个连接建立一个回话, 在回话上和远程主机通信。
	session.Stdout = os.Stdout
	session.Stderr = os.Stderr
	session.Stdin = os.Stdin

	modes := ssh.TerminalModes{
		ssh.ECHO:1,
		ssh.ECHOCTL:0,
		ssh.TTY_OP_ISPEED:14400,
		ssh.TTY_OP_OSPEED:14400,
	}
	
	termFd := int(os.Stdin.Fd())
	w,h,_ := terminal.GetSize(termFd)
	termState, _ := terminal.MakeRaw(termFd)
	// TODO 主动从服务器exit,会导致空指针.
	defer terminal.Restore(termFd,termState)

	err = session.RequestPty("xterm-256color",h,w,modes)
	if err != nil {
		log.Fatalln(err)
	}
	err = session.Shell()
	if err != nil {
		log.Fatalln(err)
	}
	err = session.Wait()
	if err != nil {
		log.Fatalln(err)
	}
}
```

### 验证

```
### 交互命令式验证
$ go build sshTunel.go
$ ./sshTunel.exe
Last login: Mon Sep  2 00:43:39 2019 from 183.192.10.3

                        Welcome to Alibaba Cloud Elastic Compute Service !

                Welcome to the testing environment of Louis.
                Feel free to use this system for testing your Linux
                skills. In case of any issues reach out to admin at
                louisehong4168@gmail.com. Thank you.


[oh-my-zsh] Random theme '/root/.oh-my-zsh/themes/gnzh.zsh-theme' loaded...
╭─root@master-louis ~ system: ruby 2.0.0p648
╰─➤


## 执行命令式
> go run ssh.go -a "d:/text/id_rsa" -c pwd
/root

> go run ssh.go -u feng -w password -c pwd
/home/feng
```

初步实现ssh客户端功能