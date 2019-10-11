---
title: "nginx if 多重判断语句"
date: "2018-12-12 10:44:32"
tag: [nginx]
categories: [server,nginx]
---

[TOC]

## nginx 多重判断

在Nginx 配置文件里面有简单的条件控制，但并不支持if条件的逻辑与／逻辑或运算 ，并且不支持if的嵌套语法，这就需要换一种方法来做了。

场景需求：
手机端访问web端的时候，需要跳转至手机端。但是某些页面又不需要跳转。刚好是公司的业务需要吧，所以有了nginx的多重判断
具体代码如下：

```
location = /web/address {
    if ($http_user_agent ~* "((MIDP)|(WindowsWechat)|(WAP)|(UP.Browser)|(Smartphone)|(Obigo)|(Mobile)|(AU.Browser)|(wxd.Mms)|(WxdB.Browser)|(CLDC)|(UP.Link)|(KM.Browser)|(UCWEB)|(SEMC\-Browser)|(Mini)|(Symbian)|(Palm)|(Nokia)|(Panasonic)|(MOT)|(SonyEricsson)|(NEC)|(Alcatel)|(Ericsson)|(BENQ)|(BenQ)|(Amoisonic)|(Amoi)|(Capitel)|(PHILIPS)|(SAMSUNG)|(Lenovo)|(Mitsu)|(Motorola)|(SHARP)|(WAPPER)|(LG)|(EG900)|(CECT)|(Compal)|(kejian)|(Bird)|(BIRD)|(G900/V1.0)|(Arima)|(CTL)|(TDG)|(Daxian)|(DAXIAN)|(DBTEL)|(Eastcom)|(EASTCOM)|(PANTECH)|(Dopod)|(Haier)|(HAIER)|(KONKA)|(KEJIAN)|(LENOVO)|(Soutec)|(SOUTEC)|(SAGEM)|(SEC)|(SED)|(EMOL)|(INNO55)|(ZTE)|(iPhone)|(Android)|(Windows CE)|(Wget)|(Java)|(Opera))"  ) {
        set $ismob 1;
    }

    if ( $ismob = 1 ) {
        rewrite ^/(.*)$  https://m.fenghong.tech/channel/64688 redirect;
    }
    if ( $ismob != 1 ) {
        rewrite ^/(.*)$  https://www.fenghong.tech/home/activity?un=wdxx redirect;
    }
}
```

附if语法： 
语法：`if(condition){………}`
配置作用域：`server,location`

```
"="和""!="" 变量等于不等于条件
"~" 和"~" 匹配到指定内容是否区分大小写
"!~"和"!~" 匹配到指定内容是否区分大小写
"-f"和"!-f" 检查一个文件是否存在
"-d" 和"!-d" 检查一个目录是否存在
"-e"和"!-e" 检查一个文件，目录，软连接是否存在
"-x"和"!-x" 检查一个是否有执行权限
匹配的内容可以是字符串也可以是一个正则表达式。
如果一个正则表达式包含"}"或者";"就必须包含在单引号或双引号里面。
```

## 参考：
- [AN SHEN](https://www.lshell.com/2018/01/nginx.html)
