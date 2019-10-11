---
title: "go flag"
date: 2019-08-28 18:10:10
tags: [go,flag]
categories: [golang] 
---

[TOC]

# flag - 命令行参数解析

在写命令行程序（工具、server）时，对命令参数进行解析是常见的需求。各种语言一般都会提供解析命令行参数的方法或库，以方便程序员使用。如果命令行参数纯粹自己写代码解析，对于比较复杂的，还是挺费劲的。在 go 标准库中提供了一个包：`flag`，方便进行命令行解析。

**注：区分几个概念**

1. 命令行参数（或参数）：是指运行程序提供的参数
2. 已定义命令行参数：是指程序中通过flag.Xxx等这种形式定义了的参数
3. 非flag（non-flag）命令行参数（或保留的命令行参数）：后文解释

## 1.1. 使用示例

我们以 `nginx` 为例，执行 `nginx -h`，输出如下：

```
$ nginx -h
nginx version: nginx/1.12.2
Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
  -?,-h         : this help
  -v            : show version and exit
  -V            : show version and configure options then exit
  -t            : test configuration and exit
  -T            : test configuration, dump it and exit
  -q            : suppress non-error messages during configuration testing
  -s signal     : send signal to a master process: stop, quit, reopen, reload
  -p prefix     : set prefix path (default: /usr/share/nginx/)
  -c filename   : set configuration file (default: /etc/nginx/nginx.conf)
  -g directives : set global directives out of configuration file

```

我们通过 `flag` 实现类似 nginx 的这个输出，创建文件 nginx.go，内容如下：

```go
package main

import (
    "flag"
    "fmt"
    "os"
)

// 实际中应该用更好的变量名
var (
    h bool

    v, V bool
    t, T bool
    q    *bool

    s string
    p string
    c string
    g string
)

func init() {
    flag.BoolVar(&h, "h", false, "this help")

    flag.BoolVar(&v, "v", false, "show version and exit")
    flag.BoolVar(&V, "V", false, "show version and configure options then exit")

    flag.BoolVar(&t, "t", false, "test configuration and exit")
    flag.BoolVar(&T, "T", false, "test configuration, dump it and exit")

    // 另一种绑定方式
    q = flag.Bool("q", false, "suppress non-error messages during configuration testing")

    // 注意 `signal`。默认是 -s string，有了 `signal` 之后，变为 -s signal
    flag.StringVar(&s, "s", "", "send `signal` to a master process: stop, quit, reopen, reload")
    flag.StringVar(&p, "p", "/usr/local/nginx/", "set `prefix` path")
    flag.StringVar(&c, "c", "conf/nginx.conf", "set configuration `filename`")
    flag.StringVar(&g, "g", "conf/nginx.conf", "set global `directives` out of configuration file")

    // 改变默认的 Usage
    flag.Usage = usage
}

func main() {
    flag.Parse()

    if h {
        flag.Usage()
    }
}

func usage() {
    fmt.Fprintf(os.Stderr, `nginx version: nginx/1.12.2
Usage: nginx [-hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
`)
    flag.PrintDefaults()
}
```

执行：`go run nginx.go -h`，（或 `go build -o nginx && ./nginx -h`）输出如下：

```
nginx version: nginx/1.12.2
Usage: nginx [-hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

Options:
  -T    test configuration, dump it and exit
  -V    show version and configure options then exit
  -c file
        set configuration file (default "conf/nginx.conf")
  -g directives
        set global directives out of configuration file (default "conf/nginx.conf")
  -h    this help
  -p prefix
        set prefix path (default "/usr/local/nginx/")
  -q    suppress non-error messages during configuration testing
  -s signal
        send signal to a master process: stop, quit, reopen, reload
  -t    test configuration and exit
  -v    show version and exit
```

仔细理解以上例子，如果有不理解的，看完下文的讲解再回过头来看。

## flag 包概述

`flag` 包实现了命令行参数的解析。

### 定义 flags 有两种方式

1）flag.Xxx()，其中 `Xxx` 可以是 Int、String 等；返回一个相应类型的指针，如：

```
var ip = flag.Int("flagname", 1234, "help message for flagname")
```

2）flag.XxxVar()，将 flag 绑定到一个变量上，如：

```go
var flagvar int
flag.IntVar(&flagvar, "flagname", 1234, "help message for flagname")
```

### 自定义 Value

另外，还可以创建自定义 flag，只要实现 flag.Value 接口即可（要求 `receiver` 是指针），这时候可以通过如下方式定义该 flag：

```
flag.Var(&flagVal, "name", "help message for flagname")
```

例如，解析接收到的电话号码，我们希望直接解析到 slice 中，我们可以定义如下 Value：

```go
type Receives []string

func newReceives(vals []string, r *[]string) *Receives {
    *r = vals
    return (*Receives)(r)
}

func (r *Receives) Set(value string) error {
    if len(*r) >0 {
		return errors.New("interval flag already set")
	}
	for _,dt :=range strings.Split(value, ",") {
		*r = append(*r ,dt)
	}
	return nil
}

func (r *Receives) Get() interface{} { return []string(*r) }

func (r *Receives) String() string { return fmt.Sprint(*r) }
```

之后可以这么使用：

```
var receievs []string
flag.Var(newSliceValue([]string{}, &receievs), "r", "receievs")
```

这样通过 `-r "123,456"` 这样的形式传递参数，`languages` 得到的就是 `[123,456]`。

flag 中对 Duration 这种非基本类型的支持，使用的就是类似这样的方式。

### 1.2.3. 解析 flag

在所有的 flag 定义完成之后，可以通过调用 `flag.Parse()` 进行解析。

命令行 flag 的语法有如下三种形式：

```
-flag // 只支持bool类型
-flag=x
-flag x // 只支持非bool类型
```

其中第三种形式只能用于非 bool 类型的 flag，原因是：如果支持，那么对于这样的命令 cmd -x *，如果有一个文件名字是：0或false等，则命令的原意会改变（之所以这样，是因为 bool 类型支持 `-flag` 这种形式，如果 bool 类型不支持 `-flag` 这种形式，则 bool 类型可以和其他类型一样处理。也正因为这样，Parse()中，对 bool 类型进行了特殊处理）。默认的，提供了 `-flag`，则对应的值为 true，否则为 `flag.Bool/BoolVar` 中指定的默认值；如果希望显示设置为 false 则使用 `-flag=false`。

int 类型可以是十进制、十六进制、八进制甚至是负数；`bool `类型可以是1, 0, t, f, true, false, TRUE, FALSE, True, False。Duration 可以接受任何` time.ParseDuration `能解析的类型。

如果想要更复杂的flag实现[pflag](https://github.com/spf13/pflag)或者[cobra](https://github.com/spf13/cobra)

