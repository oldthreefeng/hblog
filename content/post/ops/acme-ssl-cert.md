---
title: "use acme.sh to renew ssl cert"
date: "2019-09-02 09:59:32"
tags: [ssl,acmeTiny]
categories: [server,ops]
---

[TOC]

>  acme.sh：https://github.com/Neilpang/acme.sh
>
> Let's Encrypt 已经支持通配符SSL证书，这是很让人开心的事情。由于通配符域名的特殊性，比如说你不应该在只持有子域名如ailion.github.com时，就能申请到*.github.com的通配符证书，不然github就完蛋了。因此，通配符证书的申请要比普通证书申请更加复杂和严格。

## 泛域名证书生成

其实github上的操作已经够细致了，这边主要解释利用阿里云的DNSAPI来生产泛域名证书。

首先,登录阿里云账户,获取api授权. https://ak-console.aliyun.com/#/accesskey, 

建议使用子账户Ram授权方式,[https://ram.console.aliyun.com/policies/AliyunDNSFullAccess](https://ram.console.aliyun.com/policies/AliyunDNSFullAccess)

只读权限是不可以的，因为它本质上还是通过添加TXT记录的方式来验证，它会用脚本来帮你添加,所以这里给的是`AliyunDNSFullAccess`。

```
export Ali_Key="sdfsdfsdfljlbjkljlkjsdfoiwje"
export Ali_Secret="jlsdflanljkljlfdsaklkjflsa"
```

Ok, let's issue a cert now:

```
acme.sh --issue --dns dns_ali -d *.example.com
```

如果是root账户进行授权issue,加上`--force`:

```
acme.sh --issue --dns dns_ali -d *.example.com --force
```

具体看[https://github.com/Neilpang/acme.sh/wiki/sudo](https://github.com/Neilpang/acme.sh/wiki/sudo)

The `Ali_Key` and `Ali_Secret` will be saved in `~/.acme.sh/account.conf` and will be reused when needed。

一般的错误:环境变量设置错误.

```
 ~]# acme.sh --issue -d '*.example.com' --dns dns_ali --force
[Mon Sep  2 10:06:58 CST 2019] Single domain='*.example.com'
[Mon Sep  2 10:06:58 CST 2019] Getting domain auth token for each domain
[Mon Sep  2 10:07:00 CST 2019] Getting webroot for domain='*.example.com'
[Mon Sep  2 10:07:00 CST 2019] Adding txt value: eAp0uvSMnH529uRdumebXJmn6PQabQBfchpgwNqQHoM for domain:  _acme-challenge.example.com
[Mon Sep  2 10:07:00 CST 2019] You don't specify aliyun api key and secret yet.
[Mon Sep  2 10:07:00 CST 2019] Error add txt for domain:_acme-challenge.example.com
[Mon Sep  2 10:07:00 CST 2019] Please add '--debug' or '--log' to check more details.
[Mon Sep  2 10:07:00 CST 2019] See: https://github.com/Neilpang/acme.sh/wiki/How-to-debug-acme.sh
```

成功一般是这样的:

```
# ./acme.sh --issue --dns dns_ali  -d '*.example.com'  --force
[Mon Sep  2 10:13:46 CST 2019] Single domain='*.example.com'
[Mon Sep  2 10:13:46 CST 2019] Getting domain auth token for each domain
[Mon Sep  2 10:13:48 CST 2019] Getting webroot for domain='*.example.com'
[Mon Sep  2 10:13:48 CST 2019] Adding txt value: VYhSy8k-BV6UhSLPJmDuRxxgzE-3YLWmlMq6i9rpt-U for domain:  _acme-challenge.example.com
[Mon Sep  2 10:13:50 CST 2019] The txt record is added: Success.
[Mon Sep  2 10:13:50 CST 2019] Let's check each dns records now. Sleep 20 seconds first.
[Mon Sep  2 10:14:11 CST 2019] Checking example.com for _acme-challenge.example.com
[Mon Sep  2 10:14:13 CST 2019] Domain example.com '_acme-challenge.example.com' success.
[Mon Sep  2 10:14:13 CST 2019] All success, let's return
[Mon Sep  2 10:14:13 CST 2019] Verifying: *.example.com
[Mon Sep  2 10:14:16 CST 2019] Success
[Mon Sep  2 10:14:16 CST 2019] Removing DNS records.
[Mon Sep  2 10:14:16 CST 2019] Removing txt: VYhSy8k-BV6UhSLPJmDuRxxgzE-3YLWmlMq6i9rpt-U for domain: _acme-challenge.example.com
[Mon Sep  2 10:14:19 CST 2019] Removed: Success
[Mon Sep  2 10:14:19 CST 2019] Verify finished, start to sign.
[Mon Sep  2 10:14:19 CST 2019] Lets finalize the order, Le_OrderFinalize: https://acme-v02.api.letsencrypt.org/acme/finalize/64821731/1006905583
[Mon Sep  2 10:14:21 CST 2019] Download cert, Le_LinkCert: https://acme-v02.api.letsencrypt.org/acme/cert/03a8c20bd0553f46fc90702473498ef6e4ae
[Mon Sep  2 10:14:22 CST 2019] Cert success.
-----BEGIN CERTIFICATE-----
*************************
-----END CERTIFICATE-----
[Mon Sep  2 10:14:22 CST 2019] Your cert is in  /root/.acme.sh/*.example.com/*.example.com.cer 
[Mon Sep  2 10:14:22 CST 2019] Your cert key is in  /root/.acme.sh/*.example.com/*.example.com.key 
[Mon Sep  2 10:14:22 CST 2019] The intermediate CA cert is in  /root/.acme.sh/*.example.com/ca.cer 
[Mon Sep  2 10:14:22 CST 2019] And the full chain certs is there:  /root/.acme.sh/*.example.com/fullchain.cer
```

### 自动renew

```
$ acme.sh --install-cert -d *.example.com --key-file  /etc/nginx/sslkey/cert.example.com.key --fullchain-file /etc/nginx/sslkey/example.fullchain.cer  --reloadcmd "nginx -s reload" --force
[Mon Sep  2 10:20:34 CST 2019] Installing key to:/etc/nginx/sslkey/cert.example.com.key
[Mon Sep  2 10:20:34 CST 2019] Installing full chain to:/etc/nginx/sslkey/example.fullchain.cer
[Mon Sep  2 10:20:34 CST 2019] Run reload cmd: nginx -s reload
[Mon Sep  2 10:20:34 CST 2019] Reload success
[root@jx_web_slave .acme.sh]# acme.sh --cron -f
[Mon Sep  2 10:20:49 CST 2019] ===Starting cron===
[Mon Sep  2 10:20:49 CST 2019] Renew: '*.example.com'
[Mon Sep  2 10:20:49 CST 2019] Single domain='*.example.com'
[Mon Sep  2 10:20:49 CST 2019] Getting domain auth token for each domain
[Mon Sep  2 10:20:51 CST 2019] Getting webroot for domain='*.example.com'
[Mon Sep  2 10:20:51 CST 2019] *.example.com is already verified, skip dns-01.
[Mon Sep  2 10:20:51 CST 2019] Verify finished, start to sign.
[Mon Sep  2 10:20:51 CST 2019] Lets finalize the order, Le_OrderFinalize: https://acme-v02.api.letsencrypt.org/acme/finalize/64821731/1006938413
[Mon Sep  2 10:20:54 CST 2019] Download cert, Le_LinkCert: https://acme-v02.api.letsencrypt.org/acme/cert/03d49c06af74d208ed7d6a6d8bdc3ee20b30
[Mon Sep  2 10:20:54 CST 2019] Cert success.
-----BEGIN CERTIFICATE-----
*************************
-----END CERTIFICATE-----
[Mon Sep  2 10:20:54 CST 2019] Your cert is in  /root/.acme.sh/*.example.com/*.example.com.cer 
[Mon Sep  2 10:20:54 CST 2019] Your cert key is in  /root/.acme.sh/*.example.com/*.example.com.key 
[Mon Sep  2 10:20:54 CST 2019] The intermediate CA cert is in  /root/.acme.sh/*.example.com/ca.cer 
[Mon Sep  2 10:20:54 CST 2019] And the full chain certs is there:  /root/.acme.sh/*.example.com/fullchain.cer 
[Mon Sep  2 10:20:54 CST 2019] Installing key to:/etc/nginx/sslkey/cert.example.com.key
[Mon Sep  2 10:20:54 CST 2019] Installing full chain to:/etc/nginx/sslkey/example.fullchain.cer
[Mon Sep  2 10:20:54 CST 2019] Run reload cmd: nginx -s reload
[Mon Sep  2 10:20:54 CST 2019] Reload success
[Mon Sep  2 10:20:54 CST 2019] ===End cron===
```

这条命令会生成一个`crontab`;

`--install-cert`是将泛域名`*.example.com`进行安装到指定的目录.

`--key-file ` 是将泛域名`*.example.com`这个证书的key安装的位置.

比如`/etc/nginx/sslkey/cert.example.com.key`;

`--fullchain-file `是将泛域名`*.example.com`这个证书的`cer`安装的位置

如`/etc/nginx/sslkey/example.fullchain.cer`;

`--reloadcmd` 证书生成之后重启nginx,实现自动`renew`

```
$ acme.sh --install-cert -d *.example.com --key-file  /etc/nginx/sslkey/cert.example.com.key --fullchain-file /etc/nginx/sslkey/example.fullchain.cer  --reloadcmd "nginx -s reload" --force

$ crontab -l
47 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

在`nginx.conf`中的ssl配置片段如下:

```
server {
	listen 443 ssl;
	## server_name 可以是*.example.com任意一个,证书都是下面这个.
	server_name admin.example.com;
	ssl_certificate   /etc/nginx/sslkey/example.fullchain.cer;
	ssl_certificate_key  /etc/nginx/sslkey/cert.example.com.key;
	ssl_session_timeout 5m;
	ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_prefer_server_ciphers on;
}
```

至此,大功告成.

### 参考

- [Neilpang/acne.sh](https://github.com/Neilpang/acme.sh)

