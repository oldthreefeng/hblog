---
title: Http(Apache1)
date: 2018-06-19 16:03:32
urlname: httpd1
tags: Linux
categories: Http
---

摘要：Apache的httpd服务常见配置解析，配置文件的实验----

# httpd 2.4常见配置

`grep -v '^$\|^\s*#' /etc/httpd/conf/httpd.conf`,查看配置文件的非注释内容。

## 网站家目录/服务器版本/监听端口的修改

centos7版本的http服务，更换`documentroot`的时候，需要放开权限，才能实现访问，不然会报404错误，另外apache服务的版本号的隐藏，在`ServerTokens`语句块中，修改为Prod即可,很多大型的网站，如京东，淘宝，`apache或者nginx`的版本号都会隐藏。监听端口的配置修改，配置文件中加入`listen 80`，`http`监听端口默认80

```http
]# vim /etc/httpd/conf.d/test.conf
ServerRoot '/etc/httpd'
ServerTokens Prod   #curl -I http://192.168.1.8
listen 192.168.1.8:80
StartServers 20
StartThreads 50
DocumentRoot "/data/www/"
<Directory "/data/www">
<RequireAll>
    Require all granted
    Require not ip 192.168.1.17
</RequireAll>
</Directory>
]# systemctl restart httpd
]#curl -I 192.168.1.8
HTTP/1.1 200 OK
Date: Wed, 20 Jun 2018 11:38:30 GMT
Server: Apache
Last-Modified: Wed, 20 Jun 2018 11:37:49 GMT
ETag: "6-56f113a6fe504"
Accept-Ranges: bytes
Content-Length: 6
Content-Type: text/html; charset=UTF-8
```

在禁止访问的主机ip为`192.168.1.11`上访问,会出现`Forbidden 403`，在其他的授权主机上即可正常访问。

```
]#curl -I 192.168.1.8
HTTP/1.1 403 Forbidden
Date: Wed, 20 Jun 2018 11:39:39 GMT
Server: Apache
Last-Modified: Thu, 16 Oct 2014 13:20:58 GMT
ETag: "1321-5058a1e728280"
Accept-Ranges: bytes
Content-Length: 4897
Content-Type: text/html; charset=UTF-8
```

## 持久链接

系统默认的持久链接为5s，时间较短，可以稍微调大。

```
]# vim /etc/httpd/conf.d/test.conf
KeepAlivetimeout  50   #持久链接
MaxKeepAliveRequests 100
]# systemctl restart httpd
]# telnet /192.168.1.8 HTTP/1.1
GET /index.html
HOST:6.6.6.6
```

## MPM三种工作模式

> **refork**：多进程I/O模型，每个进程响应一个请求，默认模型
> 一个主进程：生成和回收n个子进程，创建套接字，不响应请求
> 多个子进程：工作work进程，每个子进程处理一个请求；系统初始时，预先生成多个空闲进程，等待请求，最大不超过1024个
>
> ![1529493571245](http://pic.fenghong.tech/1529493571245.png)

> **worker**：复用的多进程I/O模型,多进程多线程，IIS使用此模型
> 一个主进程：生成m个子进程，每个子进程负责生个n个线程，每个线程响应一个请求，并发响应请求：m*n 
>
> ![1529493599950](http://pic.fenghong.tech/1529493599950.png)

> **event**：事件驱动模型（worker模型的变种）
> 一个主进程：生成m个子进程，每个进程直接响应n个请求，并发响应请求：m*n，有专门的线程来管理这些keep-alive类型的线程，当有真实请求时，将请求传递给服务线程，执行完毕后，又允许释放。这样增强了高并发场景下的请求处理能力，示意图入下
>
> ![1529493624214](http://pic.fenghong.tech/1529493624214.png)

修改配置文件`/etc/httpd/conf.modules.d/00-mpm.conf`，改变MPM工作模式
```
LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
#LoadModule mpm_worker_module modules/mod_mpm_worker.so
#LoadModule mpm_event_module modules/mod_mpm_event.so
```

经ab测试比对，发现`prefork`和`event`模块工作并无大的差别。

## 访问控制

**基于用户的访问控制**

- 认证质询：`WWW-Authenticate`：响应码为401，拒绝客户端请求，并说明要求客户端提供账号和密码

- 认证：Authorization：客户端用户填入账号和密码后再次发送请求报文；认证通过时，则服务器发送响应的资源

- 认证方式两种：

  basic：明文
  digest：消息摘要认证,兼容性差

- 安全域：需要用户认证后方能访问的路径；应该通过名称对其进行标识，以便于告知用户认证的原因


```http
<Directory "/data/www">
<RequireAll>
    Require all granted
</ReqireAll>
</Directory>
<Files "*.conf">
	Require all denied
</Files>
<Filesmatch "\.(conf|ini)$">
	Require all denied
</Filesmatch>
<Location "/conf">
<RequireAny>
    Require all denied
    Require ip 192.168.1.11
</RequireAny>
</Location>
```

完成配置文件后，重启服务，进行实验效果,在其他主机上进行访问，`.conf`文件的确不能访问。

```
]#curl  192.168.1.8/php.conf
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /php.conf
on this server.</p>
</body></html>
]#curl  192.168.1.8/index.html
test1
```

<Directory>中**“基于源地址”实现访问控制**

`options`：禁止软连接`-FollowSymlinks`

`Indexes`:文件索引列表，搭建`yum`仓库可以用到,此项结合注释`conf.d/welcome.conf`,不然会报错

```
]# vim /etc/httpd/conf.d/test.conf
<Directory "/data/www">
	Require all granted
	options +Indexes -FollowSymlinks
</Directory>
]# ll /data/www
total 24
lrwxrwxrwx 1 root root   11 Jun 20 20:06 hexo -> /data/hexo/
```

重启服务后，进行访问实验,hexo文件夹为软连接,禁用软连接后，文件夹不能访问了。indexes的确是类似yum仓库的路径。

```
]# links 192.168.1.8/hexo
Forbidden
You don't have permission to access /hexo/ on this server.
```

 `AllowOverride`:
与访问控制相关的哪些指令可以放在指定目录下的`.htaccess`（由AccessFileName指定）文件中，覆盖之前的配置指令,只对<directory>语句有效

```
AllowOverride All: 所有指令都有效
AllowOverride None：.htaccess 文件无效
AllowOverride AuthConfig Indexes 除了AuthConfig 和Indexes的其它指令都无法覆盖
```
## 身份验证

- `htpasswd` :默认是MD5加密

- `htpasswd [options] /PATH/HTTPD_PASSWD_FILE username`
  -c：自动创建文件，仅应该在文件不存在时使用
  -p：明文密码
  -d：CRYPT格式加密，默认
  -m：md5格式加密
  -s: sha格式加密
  -D：删除指定用户

- group需要手动创建

我们需要先按照`htpassword`生成用户名和密码。第一次需要带`-c`选项
```
]# htpasswd -c /etc/httpd/conf.d/.httpuser tom
]# htpasswd -s /etc/httpd/conf.d/.httpuser jerry
]# cat /etc/httpd/conf.d/.httpuser 
user1:$apr1$O8G3R9KG$AjOvrOCyxNGXZC6Sm.Kx3.
user2:{SHA}QL0AFWMIX8NRZTKeof9cXsvbvu8=
]# vim /etc/httpd/conf.d/.httpgroup
g1: tom jerry
```
- 修改配置文件如下，然后重启服务，用户访问需要用到上面设置的用户和密码进行登录验证，由于是基于`http`服务，并没有加密，因此，容易被劫持然后被查询明文密码，是巨大的安全隐患。此项一般是配合`https`进行实现；实现方法1

```
]# vim /etc/httpd/conf.d/test.conf
<Directory "/data/www/admin">
	AuthType Basic
	AuthName "String"
	AuthUserFile "/etc/httpd/conf.d/.httpuser"
	Require user tom jerry
</Directory>
]# systemctl restart httpd
```
- 实现方法2

```
]# vim /etc/httpd/conf.d/test.conf
<Directory "/data/www/admin">
allowoverride authconfig
</Directory>
]# vim /data/www/admin/.htaccess
AuthType Basic
AuthName "String"
AuthUserFile "/etc/httpd/conf.d/.httpuser"
#AuthGroupFile "/etc/httpd/conf.d/.httpgroup"
#Require group g1
Require user tom jerry
```
## 日志

```
ErrorLog "logs/error_log"
LogLevel warn
<IfModule log_config_module>
    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
    LogFormat "%h %l %u %t \"%r\" %>s %b" common
%h			  --->  主机IP
%l %u		  --->  登录相关用户信息
%t 			  --->  时间格式GMT
%r			  --->  第一行的请求
%{Referer}i    --->  前一网站跳转的地址
%{User-Agent}i --->  客户端使用的浏览器
```

## 目录别名

格式： `Alias /URL/ "/PATH/"` 

```
]# vim /etc/httpd/conf.d/test.conf
<Directory "/data/hexo/categories">
    Require all granted
</Directory>
alias /hc /data/hexo/categories 
```

访问`http://192.168.1.8/hc/`，即相当于访问系统的path：`/data/hexo/categories` 

## **实现用户家目录的http共享**

- 基于模块`mod_userdir.so`实现
- SELinux: `http_enable_homedirs`
-  相关设置：启用家目录`http`共享,配置文件如下，并只允许特定`user`访问，但是基于`http`仍然存在安全隐患

```
]# vim /etc/httpd/conf.d/userdir.conf
···
<IfModule mod_userdir.c>
#UserDir disabled
UserDir public_html #指定共享目录的名称
···
</IfModule>
<Directory "/home/hong/public_html">
    AuthType Basic
    AuthName "hong home dir"
    AuthUserFile "/etc/httpd/conf.d/.httpuser"
    Require user hong   
</Directory>
```

- 准备家目录的相关文件及权限。

```
su - hong;mkdir ~/public_html；echo welcome to hong~ > ~/public_html/index.html
setfacl –m u:apache:x ~hong
```

- 访问:`http://localhost/~hong/index.html`

## server-status

LoadModule: `status_module` ,在配置文件中`modules/mod_status.so`，可以用`httpd -M`查看。

实现状态页

```
]# httpd -M |grep 'status'
status_module (shared)
]# vim /etc/httpd/conf.d/test.conf
<Location /status>    #url路径
SetHandler server-status
Order allow,deny
Allow from 192.168.
</Location>
]# systemctl restart httpd
```

压力测试，并在浏览器中查询服务器状态`http://localhost/status`

```
]# ab -c 500 -n 20000 http://192.168.1.8/m.txt
```

## 虚拟主机

- 站点标识： socket

  IP相同，但端口不同
  IP不同，但端口均为默认端口
  FQDN不同：
  	请求报文中首部
  	Host: www.magedu.com

- 有三种实现方案：
  基于ip：为每个虚拟主机准备至少一个ip地址
  基于port：为每个虚拟主机使用至少一个独立的port
  基于FQDN：为每个虚拟主机使用至少一个FQDN
- 注意：一般虚拟机不要与main主机混用；因此，要使用虚拟主机，一般先禁用main主机
  禁用方法：注释中心主机的DocumentRoot指令即可

准备工作

```
]#mkdir /data/web{1,2,3}
]#echo www.test1.com > /data/web1/index.html
]#echo www.test2.com > /data/web2/index.html
]#echo www.test3.com > /data/web3/index.html
```

### 实验一：基于Port

更改端口，完成虚拟主机的服务。默认已经完成准备工作

```
]# vim /etc/httpd/conf.d/test.conf
listen 81
listen 82
listen 83
<directory /data/>
require all granted
</directory>
<VirtualHost *:81>
  DocumentRoot "/data/web1"
  ServerName www.test1.com
  ErrorLog "logs/test1.com.error_log"
  TransferLog "logs/test1.com-access_log"
</VirtualHost>
<VirtualHost *:82>
  DocumentRoot "/data/web2"
  ServerName www.test2.com
  ErrorLog "logs/test2.com.error_log"
  TransferLog "logs/test2.com-access_log"
</VirtualHost>
<VirtualHost *:83>
  DocumentRoot "/data/web3"
  ServerName www.test3.com
  ErrorLog "logs/test3.com.error_log"
  TransferLog "logs/test3.com-access_log"
</VirtualHost>
```
​	在另外一台主机上，验证实验结果

```
]# curl http://192.168.1.8:81
www.test1.com
]#curl http://192.168.1.8:82
www.test2.com
]#curl http://192.168.1.8:83
www.test3.com
```

### 实验二：基于IP的虚拟主机,

​	先添加3个ip，这里采用的是`ifconfig。`然后修改配置文件，主要修改`<VirtualHost 192.168.1.11:80>`

```
]# ifconfig ens33:0 192.168.1.11 netmask 255.255.255.0 up
]# ifconfig ens33:1 192.168.1.22 netmask 255.255.255.0 up
]# ifconfig ens33:2 192.168.1.33 netmask 255.255.255.0 up
]# vim /etc/httpd/conf.d/test.conf
<directory /data/>
require all granted
</directory>
<VirtualHost 192.168.1.11:80>
  DocumentRoot "/data/web1"
  ServerName www.test1.com
  ErrorLog "logs/test1.com.error_log"
  TransferLog "logs/test1.com-access_log"
</VirtualHost>
<VirtualHost 192.168.1.22:80>
  DocumentRoot "/data/web2"
  ServerName www.test2.com
  ErrorLog "logs/test2.com.error_log"
  TransferLog "logs/test2.com-access_log"
</VirtualHost>
<VirtualHost 192.168.1.33:80>
  DocumentRoot "/data/web3"
  ServerName www.test3.com
  ErrorLog "logs/test3.com.error_log"
  TransferLog "logs/test3.com-access_log"
</VirtualHost>
```

​	在另外一台主机上，验证实验结果

```
]#curl 192.168.1.11
www.test1.com
]#curl 192.168.1.22
www.test2.com
]#curl 192.168.1.33
www.test3.com
```

### 实验三：FQDN

​	配置文件的更改，这里主要是的更改`ServerName`

```
]# vim /etc/httpd/conf.d/test.conf 
<directory /data/>
require all granted
</directory>
<VirtualHost *:80>
  DocumentRoot "/data/web1"
  ServerName www.test1.com
  ErrorLog "logs/test1.com.error_log"
  TransferLog "logs/test1.com-access_log"
</VirtualHost>
<VirtualHost *:80>
  DocumentRoot "/data/web2"
  ServerName www.test2.com
  ErrorLog "logs/test2.com.error_log"
  TransferLog "logs/test2.com-access_log"
</VirtualHost>
<VirtualHost *:80>
  DocumentRoot "/data/web3"
  ServerName www.test3.com
  ErrorLog "logs/test3.com.error_log"
  TransferLog "logs/test3.com-access_log"
</VirtualHost> 
```

​	在另外一台主机进行验证结果,不过基于主机的配置，需要DNS解析；这里为了方便，直接在hosts文件做名词解析；直接`curl 192.168.1.8`，这里`/var/www/html`已经失效，会默认访问基于虚拟主机的第一个网站

```
]#vim /etc/hosts
192.168.1.8 www.test1.com www.test2.com www.test3.com
]#curl www.test1.com
www.test1.com
]#curl www.test2.com
www.test2.com
]#curl www.test3.com
www.test3.com
]#curl 192.168.1.8
www.test1.com
```

