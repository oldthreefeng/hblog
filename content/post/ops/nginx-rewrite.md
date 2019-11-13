---
title: "nginx rewrite的break和last"
date: "2019-11-12T15:54:18+08:00"
tags: [nginx]
categories: [server,nginx]
---

[TOC]

last 和 break 当出现在location 之外时，两者的作用是一致的没有任何差异 

> 出现在location内部时.
>
> break和last都能阻止继续执行后面的rewrite指令，last如果在location下的话，对于重写后的URI会重新匹配location，而break不会重新匹配location。 

具体

>  last：停止当前这个请求，并根据rewrite匹配的规则重新发起一个请求。新请求又从第一阶段开始执行…
>
> break：相对last，break并不会重新发起一个请求，只是跳过当前的rewrite阶段，并执行本请求location后续的执行阶段 

为了测试方便,添加了nginx的echo模块, 测试环境重装了一下nginx. 如果是已经安装的nginx, 建议查看这篇[在已经安装Nginx的基础上增加新Nginx-echo模块]( https://blog.csdn.net/hb1707/article/details/52510611 )

```
$ wget https://github.com/openresty/echo-nginx-module/archive/v0.61.tar.gz
$ tar xf v0.61.tar.gz
$ cd echo-nginx-module-0.61/
$ pwd
/data/echo-nginx-module-0.61/
$ wget https://nginx.org/download/nginx-1.16.1.tar.gz
$ tar xf nginx-1.16.1.tar.gz && cd nginx-1.16.1
$ ./configure  --prefix=/etc/nginx \ 
--conf-path=/etc/nginx/nginx.conf \
--user=nginx --group=nginx  \
--add-module=/data/echo-nginx-module-0.61 \
--with-http_ssl_module \
--with-http_realip_module
$ make -j4 && make install
```

添加一个`test.conf`文件

```
server {
	listen 80;
	server_name ii.com;

	location /break/ {
		rewrite ^/break/(.*) /test/$1 break;
		echo "break page";
		echo "break1 page";
	} 

	location /last/ {
		rewrite ^/last/(.*) /test/$1 last;
		echo "last page";
	}  

	location /test/ {
		echo "test page";
	}
}

```

测试

```
$ curl ii.com/break/a
break page
break1 page
$ curl ii.com/test/a
test page
$ curl ii.com/last/a
test page
```

分析

> break是跳过当前请求的rewrite阶段, 继续执行本请求的其他阶段,即`echo`阶段.
>
>  last与break最大的不同是，last会重新发起一个新请求，并重新匹配location，所以对于/last,重新匹配请求以后会匹配到/test/,所以最终对应的content阶段的输出是test page; 

有个优先级的判断, `^~ /` 与`/api`, 直接访问`/api/`会执行第二个location. 添加另外一个`test2.conf`

```
server
{
	listen       80;
	server_name  i.com;
	index  index.html index.php index.htm index.jsp index.do default.do;

	location ^~ / {
		rewrite ^\/(.*)$  /$1 break;
		echo 'test ^~ /';
	}

	location /api/ {
		rewrite ^\/api/(.*)$  /$1 break;
		echo 'test /api/';
	}
}
```

测试

```
$ curl i.com/
test ^~ /
$ curl i.com/api/a
test /api/
```

分析

> rewrite规则, 应该是全局扫描, 匹配则执行rewrite语句. 

