---
title: "中间证书缺少导致request fail"
date: "2019-12-09 09:59:32"
tags: [linux,ssl]
categories: [server]
---

[TOC]

>  启用SSL已经有段时间了 ,  微信小程序开发过程中发现接口访问报 request fail 错误，各个厂商的ssl检查也没有问题。 

## 中间证书

先认识一下中间证书

> 中间证书，其实也叫中间CA（中间证书颁发机构，Intermediate certificate authority, Intermedia CA），对应的是根证书颁发机构（Root certificate authority ，Root CA）。为了验证证书是否可信，必须确保证书的颁发机构在设备的可信 CA 中。SSL的验证机制是由一级一级追溯验证，当前CA不可信则向上层CA验证，直到发现可信或没有可信CA为止，注意有时候Intermedia CA也有可能在设备的可信CA中，这样不用一直追溯到Root CA，即可认证完成。
>
> 根证书，必然是一个自签名的证书，“使用者”和“颁发者”都是相同的，所以不会进一步向下检查，如果根 CA 不是可信 CA ，则将不允许建立可信连接，并提示错误。
>
> 一般Root CA是要求离线保存的，如果要签发证书也是通过人工方式签发，这样能最大程序保证Root CA的安全，而Intermedia CA则相对宽松，允许在线签发证书的，这样方便高效，安全性灵活，即便通过Root CA签发的Intermedia CA发生意外泄露，也可以通过Root CA进行撤销



###  **客户端自动完成中间证书下载** 

 一般情况下系统可以通过URL自行完成中间证书的下载，如Windows、iOS、OSX（macOS Sierra）都支持这种方式，但Android和Java客户端不支持自行下载，还有一种情况就是无法访问根证书地址的情况如内网 .

![](https://pic.fenghong.tech/ssl/sslextradowanload.jpg)

### 服务器推送中间证书

 除了通过证书中URL下载中间证书外，还可以通过服务器向客户端主动推送中间证书，这样即可兼容系统或客户端不支持自动下载中间证书的情况 。

![](https://pic.fenghong.tech/ssl/sslsentbyserver.jpg)

 至此微信小程序接口报错的问题原因即暴露出来了，是系统或客户端不支持自动下载中间证书，其实在大家完成证书部署之后应该检测一下这样更保险[证书检测]( https://www.ssllabs.com/ssltest/analyze.html )

## fullchain证书生成

```bash
$ echo -n \
	| openssl s_client -host fenghong.tech -port 443 -showcerts 2>/dev/null \
	| sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' \
	> fenghong.tech.fullchain.pem
```

说明：

```
echo -n 
	gives a response to the server, so that the connection is released
-showcerts  
	显示所有的证书链
-host       
	域名
-port       
	端口
2>/dev/null 
	不显示错误信息
sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p'
	只显示证书信息
```

利用`fullchain.pem`证书生成`.pfx`

```bash
$ openssl pkcs12 -export -out fenghong.tech.pfx \
	-inkey fenghong.tech.key  -in fenghong.tech.fullchain.pem  \
	-certfile fenghong.tech.fullchain.pem
## 要输入密码确认, 证书和key合成在一起了. 所以需要密码保护.
```

至此, 合成的pfx支持server发送中间证书, 而不用客户端自行下载了.如果使用nginx作为ssl的web服务器, 可以直接使用pem和key证书. 不需要合成`pfx`证书. 

## 参考

- [how to download ssl certificate from a website]( https://serverfault.com/questions/139728/how-to-download-the-ssl-certificate-from-a-website )