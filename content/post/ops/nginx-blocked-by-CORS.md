---
title: "记一次nginx跨域访问Content Security Policy设置"
date: 2020-06-22T14:58:18+08:00
tags: [nginx,CORS,http,ssl]
categories: [server]
---



[toc]

## 前言

> 在开发静态页面时，类似Vue的应用，我们常会调用一些接口，这些接口极可能是跨域，然后浏览器就会报cross-origin问题不给调。
>
> 最简单的解决方法，就是把浏览器设为忽略安全问题，设置--disable-web-security。不过这种方式开发PC页面到还好，如果是移动端页面就不行了。
>
> 当出现403跨域错误的时候 `No 'Access-Control-Allow-Origin' header is present on the requested resource`，需要给Nginx服务器配置响应的header参数. 

![](https://oss.fenghong.tech/linux/EA65BF85-83B9-4536-98EA-7FCA624E063A.png)

## 解决

一般的配置nginx添加一下参数即可.

```
location / {  
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

    if ($request_method = 'OPTIONS') {
        return 204;
    }
} 
```

- Access-Control-Allow-Origin

```
服务器默认是不被允许跨域的。给Nginx服务器配置`Access-Control-Allow-Origin *`后，表示服务器可以接受所有的请求源（Origin）,即接受所有跨域的请求。
```

### 另一种解决思路

知道通常情况下，HTTPS引用HTTP的资源就会出现[跨域](https://www.uedbox.com/post/50992/)错误，但今天我们的要求是允许它跨域，并且尽量保证它是基本安全的。

```
location / {  
    add_header Content-Security-Policy upgrade-insecure-requests;
} 
```

- Content-Security-Policy

内容安全策略（CSP）需要仔细调整和精确定义策略。如果启用，CSP会对浏览器呈现页面的方式产生重大影响（例如，默认情况下禁用内联JavaScript，并且必须在策略中明确允许）。CSP可防止各种攻击，包括跨站点脚本和其他跨站点注入。