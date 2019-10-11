---
title: Http(Apache)
date: 2018-06-19 10:03:32
urlname: httpd
tags: Linux
categories: Http
---

摘要：internet的发展，套接字socket的简介，HTTP协议，URL-------

# Internet And China

![1529724085331](http://pic.fenghong.tech/1529724085331.png)

- 1969年

> Internet最早来源于美国国防部高级研究计划局ARPA建立的ARPANet，1969年投入运行。1983年，ARPAnet分裂为两部分：ARPAnet和纯军事用的MILNET。当年1月，ARPA把TCP/IP协议作为ARPAnet的标准协议，这个以ARPAnet为主干网的网际互联网便被称为Internet。1986年，美国国家科学基金会建立计算机通信网络NSFnet。此后，NSFNet逐渐取代ARPANet在Internet的地位。1990年，ARPANet正式关闭.
> 北京时间1987年9月20日，钱天白建立起一个网络节点，通过电话拨号连接到国际互联网，向他的德国朋友发出来自中国的第一封电子邮件：Across theGreat Wall we can reach every corner in the world，自此，中国与国际计算机网络开始连接在一起.

- 1990年10月

> 钱天白教授代表中国正式在国际互联网络信息中心的前身DDN-NIC注册登记了我国的顶级域名CN，并且从此开通了使用中国顶级域名CN的国际电子邮件服务。由于当时中国尚未正式连入Internet，所以委托德国卡尔斯鲁厄大学运行CN域名服务器

- 1993年3月2日

> 中国科学院高能物理研究所租用AT&T公司的国际卫星信道接入美国斯坦福线性加速器中心（SLAC）的64K专线正式开通,专线开通后，美国政府以Internet上有许多科技信息和其它各种资源，不能让社会主义国家接入为由，只允许这条专线进入美国能源网而不能连接到其它地方。尽管如此，这条专线仍是我国部分连入Internet的第一根专线

- 1994年4月20日

> 中国实现与互联网的全功能连接，被国际上正式承认为有互联网的国家.

- 1994年5月21日

> 在钱天白教授和德国卡尔斯鲁厄大学的协助下，中国科学院计算机网络信息中心完成了中国国家顶级域名(CN)服务器的设置，改变了中国的CN顶级域名服务器一直放在国外的历史.

- 1996年1月

> 中国互联网全国骨干网建成并正式开通，开始提供服务.

# Socket概念

TCP/IP协议

![1529724743263](http://pic.fenghong.tech/1529724743263.png)

## 跨网络主机通讯

- 在建立通信连接的每一端，进程间的传输要有两个标志：
- IP地址和端口号，合称为套接字地址 socket address
- 客户机套接字地址定义了一个唯一的客户进程
- 服务器套接字地址定义了一个唯一的服务器进程

![1529724935070](http://pic.fenghong.tech/1529724935070.png)

## socket套接字

- Socket:套接字，进程间通信IPC的一种实现，允许位于不同主机（或同一主机）上

不同进程之间进行通信和数据交换，SocketAPI出现于1983年，4.2 BSD实现

- Socket API：封装了内核中所提供的socket通信相关的系统调用
- Socket Domain：根据其所使用的地址
```
AF_INET：Address Family，IPv4
AF_INET6：IPv6
AF_UNIX：同一主机上不同进程之间通信时使用
```
- Socket Type：根据使用的传输层协议
```
SOCK_STREAM：流，tcp套接字，可靠地传递、面向连接
SOCK_DGRAM：数据报，udp套接字，不可靠地传递、无连接
SOCK_RAW: 裸套接字,无须tcp或tdp,APP直接通过IP包通信
```
![1529725064099](http://pic.fenghong.tech/1529725064099.png)

## C/S程序套接字函数

![1529725613846](http://pic.fenghong.tech/1529725613846.png)

**套接字相关的系统调用：**
```
socket(): 创建一个套接字
bind()：绑定IP和端口
listen()：监听
accept()：接收请求
connect()：请求连接建立
write()：发送
read()：接收
close():关闭连接
```
## Socket通信示例：

服务器端:`tcpserver.py`

```python
#!/usr/bin/env python
import socket
HOST='127.0.0.1'
PORT=9527
BUFFER=4096
sock=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
sock.bind((HOST,PORT))
sock.listen(3)
print('tcpServer listen at: %s:%s\n\r' %(HOST,PORT))
while True:
client_sock,client_addr=sock.accept()
print('%s:%s connect' %client_addr)
	while True:
	recv=client_sock.recv(BUFFER)
		if not recv:
			client_sock.close()
			break
	print('[Client %s:%s said]:%s' %(client_addr[0],client_addr[1],recv))
	client_sock.send('tcpServer has received your message')
	sock.close()
```

**客户端：**`tcpclient.py`

```python
#!/usr/bin/env python
import socket
HOST='127.0.0.1'
PORT=9527
BUFFER=4096
sock=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
sock.connect((HOST,PORT))
sock.send('hello, tcpServer!')
recv=sock.recv(BUFFER)
print('[tcpServer said]: %s' % recv)
sock.close()
```

以普通用户运行`python tcpserver.py`时,对`port`的要求比较严格.
```
0-1023：系统端口或特权端口(仅管理员可用) ，众所周知，永久的分配给固定的系统应用使用，22/tcp(ssh), 80/tcp(http), 443/tcp(https)
```
# HTTP通讯及术语

![1529726658190](http://pic.fenghong.tech/1529726658190.png)

- http: `Hyper Text Transfer Protocol, 80/tcp`

- html: `Hyper Text Markup Language` 超文本标记语言，编程语言

- 示例：

```
<html>
<head>
<title>html语言
</title>
</head>
<body>
<img src="https://y.gtimg.cn/music/photo_new/T002R300x300M000001iSiol1uL9K3.jpg?max_age=2592000" >
<h1>标题1</h1>
<p><a href=http://www.baidu.com>百度一下</a>欢迎</p>
</body>
</html>
```
- CSS: `Cascading Style Sheet` 层叠样式表。类似`tempaltes`的模板文件
- js: `javascript`

MIME： Multipurpose Internet Mail Extensions

- 多用途互联网邮件扩展 `/etc/mime.types`

- 格式：`major/minor`
```
text/plain
text/html
text/css
image/jpeg
image/png
video/mp4
application/javascript
```
[参考](http://www.w3school.com.cn/)

# HTTP协议介绍

![1529728034881](http://pic.fenghong.tech/1529728034881.png)

- http/0.9：1991
>1. 原型版本，功能简陋，只有一个命令GET。`GET /index.html` ,服务器只能回应HTML格式字符串，不能回应别的格式.
- http/1.0: 1996年5月,支持`cache, MIME, method`
> 1. 每个TCP连接只能发送一个请求，发送数据完毕，连接就关闭，如果还要请求其他资源，就必须再新建一个连接;
> 2. 引入了POST命令和HEAD命令;
> 3. 头信息是 ASCII 码，后面数据可为任何格式。服务器回应时会告诉客户端，数据是什么格式，Content-Type字段的作用。这些数据类型总称为MIME 多用途互联网邮件扩展，每个值包括一级类型和二级类型，预定义的类型，也可自定义类型, 常见Content-Type值：text/xml image/jpeg audio/mp3.

-  http/1.1：1997年1月
> 1. 引入了持久连接（persistent connection），即TCP连接默认不关闭，可以被多个请求复用，不用声明Connection: keep-alive。对于同一个域名，大多数浏览器允许同时建立6个持久连接
> 2. 引入了管道机制（pipelining），即在同一个TCP连接里，客户端可以同时发送多个请求，进一步改进了HTTP协议的效率
> 3. 新增方法：PUT、PATCH、OPTIONS、DELETE
> 4. 同一个TCP连接里，所有的数据通信是按次序进行的。服务器只能顺序处理回应，前面的回应慢，会有许多请求排队，造成"队头堵塞"（Head-of-line blocking）
> 5. 为避免上述问题，两种方法：一是减少请求数，二是同时多开持久连接。网页优化技巧，如合并脚本和样式表、将图片嵌入CSS代码、域名分片（domain sharding）等
> 6. HTTP 协议不带有状态，每次请求都必须附上所有信息。请求的很多字段都是重复的，浪费带宽，影响速度

- Spdy：2009年,谷歌研发,解决 HTTP/1.1 效率不高问题
- http/2.0：2015年

> 1. 头信息和数据体都是二进制，称为头信息帧和数据帧
> 复用TCP连接，在一个连接里，客户端和浏览器都可以同时发送多个请求或回应，且不用按顺序一一对应，避免了“队头堵塞“,此双向的实时通信称为多工（Multiplexing）
> 2. 引入头信息压缩机制（header compression）,头信息使用gzip或compress压缩后再发送；客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，不发送同样字段，只发送索引号，提高速度
> 3. HTTP/2 允许服务器未经请求，主动向客户端发送资源，即服务器推送（server push）

# URI

- URI: Uniform Resource Identifier 统一资源标识，分为URL和URN

> 1. URN: Uniform Resource Naming，统一资源命名
>
> 		示例： P2P下载使用的磁力链接是URN的一种实现
> 	magnet:?xt=urn:btih:660557A6890EF888666
>
> 2. URL: Uniform Resorce Locator，统一资源定位符，用于描述某服务器某特定资源位置
> 3. 两者区别：URN如同一个人的名称，而URL代表一个人的住址。换言之，URN定义某事物的身份，而URL提供查找该事物的方法。URN仅用于命名，而不指定地址

URL的组成
```
<scheme>://<user>:<password>@<host>:<port>/<path>;<params>?<query>#<frag>
schame:方案，访问服务器以获取资源时要使用哪种协议
user:用户，某些方案访问资源时需要的用户名
password:密码，用户对应的密码，中间用：分隔
Host:主机，资源宿主服务器的主机名或IP地址
port:端口,资源宿主服务器正在监听的端口号，很多方案有默认端口号
path:路径,服务器资源的本地名，由一个/将其与前面的URL组件分隔
params:参数，指定输入的参数，参数为名/值对，多个参数，用;分隔
query:查询，传递参数给程序，如数据库，用？分隔,多个查询用&分隔
frag:片段,一小片或一部分资源的名字，此组件在客户端使用，用#分隔
###示例如下：
https://list.jd.com/list.html?cat=670,671,672&ev=149_2992&sort=sort_totalsales15_desc&trans=1
```