---
title: Let’s Encrypt使用
date: 2018-11-08 23:59:32
urlname: https-encrypt
tags: 
- Linux
- nginx
- certbot
- https
- acme_tiny
- Python
categories: server
---

###  **Let’s Encrypt 及 Certbot 简介**

Let’s Encrypt 是 一个叫 [ISRG](https://linuxstory.org/tag/isrg/) （ Internet Security Research Group ，互联网安全研究小组）的组织推出的免费安全证书计划。参与这个计划的组织和公司可以说是互联网顶顶重要的先驱，除了前文提到的三个牛气哄哄的发起单位外，后来又有思科（全球网络设备制造商执牛耳者）、 Akamai 加入，甚至连 Linux 基金会也加入了合作，这些大牌组织的加入保证了这个项目的可信度和可持续性。

![lets-encrypt](https://linuxstory.org/wp-content/uploads/2016/11/lets-encrypt.png)

尽管项目本身以及有该项目签发的证书很可信，但一开始 Let’s Encrypt 的安全证书配置起来比较麻烦，需要手动获取及部署。存在一定的门槛，没有一些技术底子可能比较难搞定。然后有一些网友就自己做了一些脚本来优化和简化部署过程。
**1. 获取 Certbot 客户端**


```
wget https://dl.eff.org/certbot-auto
chmod a+x ./certbot-auto
./certbot-auto --help 
```
**2. 配置 nginx 、验证域名所有权**

在虚拟主机配置文件`/usr/local/nginx/conf/vhost/fenghong.tech.conf`中添加如下内容，这一步是为了通过 Let’s Encrypt 的验证，验证 fenghong.tech 这个域名是属于我的管理之下。（具体解释可见下一章“一些补充说明”的“ certbot 的两种工作方式”）

```

location ^~ /.well-known/acme-challenge/ {  
default_type "text/plain";   
root     /www;
} 
location = /.well-known/acme-challenge/ {   
return 404;
} 
```

**3. 重载 nginx**

配置好 [Nginx](https://fenghong.tech/tag/nginx/) 配置文件，重载使修改生效（如果是其他系统 nginx 重载方法可能不同）
```
 sudo nginx -s reload 
```


**4. 生成证书**


```
./certbot-auto certonly --webroot -w /www -d  fenghong.tech 
```

中间会有一些自动运行及安装的软件，不用管，让其自动运行就好，有一步要求输入邮箱地址的提示，照着输入自己的邮箱即可，顺利完成的话，屏幕上会有提示信息。

**此处有坑！如果顺利执行请直接跳到第五步，我在自己的服务器上执行多次都提示**

```
connection :: The server could not connect to the client for DV :: DNS query timed out 
```

发现问题出在 DNS 服务器上，我用的是 DNSpod ，无法通过验证，最后是将域名的 DNS 服务器临时换成 Godaddy 的才解决问题，通过验证，然后再换回原来的 DNSpod 。
证书生成成功后，会有 Congratulations 的提示，并告诉我们证书放在 /etc/letsencrypt/live 这个位置
```
IMPORTANT NOTES:
 - The following errors were reported by the server:

   Domain: fenghong.tech
   Type:   unauthorized
   Detail: Invalid response from
   http://fenghong.tech/.well-known/acme-challenge/kx-juv4XwQFz1TkhL1xGNda5Nm8_fwa8rQoRUfvS01c:
   "<!DOCTYPE html>\n<html>\n  <head>\n    <meta
   http-equiv=\"Content-type\" content=\"text/html; charset=utf-8\">\n
   <meta http-equiv=\"Co"

   Domain: www.fenghong.tech
   Type:   unauthorized
   Detail: Invalid response from
   http://www.fenghong.tech/.well-known/acme-challenge/B0jELU0RmyeEt9xA9FKi6NTxj4m5PjJlvx4iCXNR4d8:
   "<!DOCTYPE html>\n<html>\n  <head>\n    <meta
   http-equiv=\"Content-type\" content=\"text/html; charset=utf-8\">\n
   <meta http-equiv=\"Co"

   To fix these errors, please make sure that your domain name was
   entered correctly and the DNS A/AAAA record(s) for that domain
   contain(s) the right IP address.

```
  这个问题是因为自己的A记录是指向了github.io，导致配置文件根本读取不到，取消gitub的A记录即可，保持自己的A记录指向自己的IP哦！

```
IMPORTANT NOTES:
- Congratulations! Your certificate and chain have been saved at   /etc/letsencrypt/live/fenghong.tech/fullchain.pem. Your cert   will expire on 2019-02-0. To obtain a new version of the   certificate in the future, simply run Let's Encrypt again.
```



**5. 配置 Nginx**（修改 `/usr/local/nginx/conf/vhost/fenghong.tech.conf`），使用 SSL 证书


```
listen 443 ssl;
server_name fenghong.tech www.fenghong.tech;
index index.html index.htm index.php;
root  /www; 
ssl_certificate      /etc/letsencrypt/live/fenghong.tech/fullchain.pem;
ssl_certificate_key  /etc/letsencrypt/live/fenghong.tech/privkey.pem;
```

上面那一段是配置了 https 的访问，我们再添加一段 http 的自动访问跳转，将所有通过 [http://www.fenghong.tech](http://www.fenghong.tech/) 的访问请求自动重定向到 [https://fenghong.tech](https://fenghong.tech/)

```
server {    
	listen 80;    
	server_name fenghong.tech www.fenghong.tech;    
	return 301 https://$server_name$request_uri;
} 
```

**6. 重载 nginx，大功告成，此时打开网站就可以显示绿色小锁了**




```
 sudo nginx -s reload 
```



### **♦后续工作**

出于安全策略， Let’s Encrypt 签发的证书有效期只有 90 天，所以需要每隔三个月就要更新一次安全证书，虽然有点麻烦，但是为了网络安全，这是值得的也是应该的。好在 Certbot 也提供了很方便的更新方法。

1. 测试一下更新，这一步没有在真的更新，只是在调用 Certbot 进行测试

```
./certbot-auto renew --dry-run 
```
如果出现类似的结果，就说明测试成功了（总之有 Congratulations 的字眼）

```
Congratulations, all renewals succeeded. The following certs have been renewed:
  /etc/letsencrypt/live/wiki.fenghong.tech/fullchain.pem (success)
  /etc/letsencrypt/live/fenghong.tech/fullchain.pem (success)
** DRY RUN: simulating 'certbot renew' close to cert expiry
**          (The test certificates above have not been saved.)
```

2. 手动更新的方法

```
./certbot-auto renew -v 
```
3. 自动更新的方法

```
./certbot-auto renew --quiet --no-self-upgrade 

```


### **♦一些补充说明解释**

1、certbot-auto 和 certbot

certbot-auto 和 certbot 本质上是完全一样的；不同之处在于运行 certbot-auto 会自动安装它自己所需要的一些依赖，并且自动更新客户端工具。因此在你使用 certbot-auto 情况下，只需运行在当前目录执行即可

```
./certbot-auto 
```
2、certbot的两种工作方式

certbot （实际上是 certbot-auto ） 有两种方式生成证书：

- **standalone** 方式： certbot 会自己运行一个 web server 来进行验证。如果我们自己的服务器上已经有 web server 正在运行 （比如 Nginx 或 Apache ），用 standalone 方式的话需要先关掉它，以免冲突。
- **webroot** 方式： certbot 会利用既有的 web server，在其 web root目录下创建隐藏文件， Let’s Encrypt 服务端会通过域名来访问这些隐藏文件，以确认你的确拥有对应域名的控制权。

本文用的是 webroot 方式，也只推荐 webroot 方式，这也是前文第二步验证域名所有权在 nginx 虚拟主机配置文件中添加 location 段落内容的原因。


试了下[这个脚本](https://github.com/xdtianyu/scripts/tree/master/lets-encrypt)，在它的基础上改了一些，签发/更新比较方便（其实就是重新签发）。核心是使用[diafygi/acme-tiny](https://github.com/diafygi/acme-tiny)，相对于certbot复杂以及各种环境检查，安装一堆东西，这个Python写的工具我感觉好用多了，在傻瓜式和使用上选择了一个折中合适的点。


### 一个快速获取/更新 Let's encrypt 证书的 shell script
------------

调用 acme_tiny.py 认证、获取、更新证书，不需要额外的依赖。

**下载到本地**

```
wget https://raw.githubusercontent.com/oldthreefeng/scripts/master/lets-encrypt/letsencrypt.conf
wget https://raw.githubusercontent.com/oldthreefeng/scripts/master/lets-encrypt/letsencrypt.sh
chmod +x letsencrypt.sh
```

**配置文件**

只需要修改 DOMAIN_KEY DOMAIN_DIR DOMAINS 为你自己的信息

```
ACCOUNT_KEY="letsencrypt-account.key"
DOMAIN_KEY="example.com.key"
DOMAIN_DIR="/var/www/example.com"
DOMAINS="DNS:example.com,DNS:whatever.example.com"
#ECC=TRUE
#LIGHTTPD=TRUE
```

执行过程中会自动生成需要的 key 文件。其中 `ACCOUNT_KEY` 为账户密钥， `DOMAIN_KEY` 为域名私钥， `DOMAIN_DIR` 为域名指向的目录，`DOMAINS` 为要签的域名列表， 需要 `ECC` 证书时取消 `#ECC=TRUE` 的注释，需要为 `lighttpd` 生成 `pem` 文件时，取消 `#LIGHTTPD=TRUE` 的注释。

**运行**

```
./letsencrypt.sh letsencrypt.conf
```

**注意**

需要已经绑定域名到 `/var/www/example.com` 目录，即通过 `http://example.com` `http://whatever.example.com` 可以访问到 `/var/www/example.com` 目录，用于域名的验证

**将会生成如下几个文件**

    lets-encrypt-x1-cross-signed.pem
    example.chained.crt          # 即网上搜索教程里常见的 fullchain.pem
    example.com.key              # 即网上搜索教程里常见的 privkey.pem 
    example.crt
    example.csr

**在 nginx 里添加 ssl 相关的配置**

    ssl_certificate     /path/to/cert/example.chained.crt;
    ssl_certificate_key /path/to/cert/example.com.key;

**cron 定时任务**

每个月自动更新一次证书，可以在脚本最后加入 service nginx reload等重新加载服务。

```
0 0 1 * * /etc/nginx/certs/letsencrypt.sh /etc/nginx/certs/letsencrypt.conf >> /var/log/lets-encrypt.log 2>&1  && nginx -s reload
```

**多个域名处理**

我有aaa.com和bbb.com，在同一个主机里进行https证书生成，这时，我们生成两个比如aaa.conf和bbb.conf.生成两个文件后，脚本运行两次即可。当然配置文件内容请看上面的信息，按需更改。
利用crontab 进行每月更新，具体如上，就不赘述了。

```
./letsencrypt.sh aaa.conf
./letsencrypt.sh bbb.conf
```

### lets-dns

**下载**

```
wget https://github.com/xdtianyu/scripts/raw/master/le-dns/le-dnspod.sh
wget https://github.com/xdtianyu/scripts/raw/master/le-dns/dnspod.conf
chmod +x le-dnspod.sh
```

**配置**

`dnspod.conf` 文件内容

```
TOKEN="YOUR_TOKEN_ID,YOUR_API_TOKEN"
RECORD_LINE="默认"
DOMAIN="example.com"
CERT_DOMAINS="example.com www.example.com im.example.com"
#ECC=TRUE
```

修改其中的 `TOKEN` 为您的 [dnspod api token](https://www.dnspod.cn/console/user/security) ，注意格式为`123456,556cxxxx`。
修改 `DOMAIN` 为你的根域名，修改 `CERT_DOMAINS` 为您要签的域名列表，需要 `ECC` 证书时请取消 `#ECC=TRUE` 的注释。

**运行**
```
[root@aliyun_file nginx]# ./le-dnspod.sh dnspod.conf 
# INFO: Using main config file dnspod.conf
+ Account already registered!
# INFO: Using main config file dnspod.conf
Processing www.fenghong.tech with alternative names: wiki.fenghong.tech cloud.fenghong.tech
 + Creating new directory ./certs/www.fenghong.tech ...
 + Signing domains...
 + Generating private key...
 + Generating signing request...
 + Requesting new certificate order from CA...
 + Received 3 authorizations URLs from the CA
 + Handling authorization for cloud.fenghong.tech
 + Handling authorization for www.fenghong.tech
 + Handling authorization for wiki.fenghong.tech
 + 3 pending challenge(s)
 + Deploying challenge tokens...
cloud.fenghong.tech MszO0HsThuxa2OD5JOggj3xodgCjwhRybBpKaDUjarM gpxslsY3Hl9croCAQBHYUOveXZpuwoPaZlvqtzOMtBg
_acme-challenge.cloud.fenghong.tech
fenghong.tech 69699373
_acme-challenge.cloud:391234194
UPDATE RECORD
DNS UPDATE SUCCESS
www.fenghong.tech rGUqFsbLKL8ygoZSjn-6iMrP6LriMaaMwqcD5iI0MLc jxbDA17eWGgxq1gydYyh1mZ0Y-1UxP3KmASkg5vV4oA
_acme-challenge.www.fenghong.tech
fenghong.tech 69699373
_acme-challenge.www:391234826
UPDATE RECORD
DNS UPDATE SUCCESS
wiki.fenghong.tech Tpx0U3AJxgC3k7jfr2kUPTPcRVYV4hyaH6uWhikfFDo Hs7rqkI5sDhLjAPe7Q8gpYt6PW2Ov6eV1U4-sKurwaU
_acme-challenge.wiki.fenghong.tech
fenghong.tech 69699373
_acme-challenge.wiki:391218779
UPDATE RECORD
DNS UPDATE SUCCESS
 + Responding to challenge for cloud.fenghong.tech authorization...
 + Challenge is valid!
 + Responding to challenge for www.fenghong.tech authorization...
 + Challenge is valid!
 + Responding to challenge for wiki.fenghong.tech authorization...
 + Challenge is valid!
 + Cleaning challenge tokens...
 + Requesting certificate...
 + Checking certificate...
 + Done!
 + Creating fullchain.pem...
 + Done!
```

最后生成的文件在当前目录的 certs 目录下

**cron 定时任务**

如果证书过期时间不少于30天， [letsencrypt.sh](https://github.com/lukas2511/letsencrypt.sh) 脚本会自动忽略更新，所以至少需要29天运行一次更新。

每隔20天(每个月的5号和25号)自动更新一次证书，可以在 `le-dnspod.sh` 脚本最后加入 service nginx reload等重新加载服务。

`0 0 5/20 * * /etc/nginx/le-dnspod.sh /etc/nginx/le-dnspod.conf >> /var/log/le-dnspod.log 2>&1`

**注意** `ubuntu 16.04` 不能定义 `day of month` 含有开始天数的 `step values`，可以替换命令中的 `5/20` 为 `5,25`。

更详细的 crontab 参数请参考 [crontab.guru](http://crontab.guru/) 进行自定义
下载文件


### 其它参考

- [Let's Encrypt，免费好用的 HTTPS 证书](https://imququ.com/post/letsencrypt-certificate.html)
- [Let’s Encrypt免费HTTPS SSL证书获取教程](https://blog.kuoruan.com/71.html)
- [用Let’s Encrypt获取免费证书](https://www.paulyang.cn/blog/archives/39)
- [免费SSL证书Let’s Encrypt安装使用教程:Apache和Nginx配置SSL](http://www.freehao123.com/lets-encrypt/)

- [How To Secure Nginx with Let's Encrypt on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04)
- [一个快速获取/更新 Let's encrypt 证书的 shell script](https://www.v2ex.com/t/241819) | [另外一个](https://github.com/xdtianyu/scripts/blob/master/lets-encrypt/)
- [Cipherli.st](https://cipherli.st/) 提供了各种webserver和一些软件的ssl推荐配置
- [SSL Server Test](https://www.ssllabs.com/ssltest/index.html) 站点https安全分析/检查
- [实践个人网站迁移HTTPS与HTTP/2](https://www.freemindworld.com/blog/2016/160301_migrate_to_https_and_http2.shtml)
- [Wiki · Tanky Woo](https://wiki.tankywoo.com/tool/letsencrypt.html)
