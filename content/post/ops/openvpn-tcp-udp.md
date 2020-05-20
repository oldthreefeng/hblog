---
title: "openvpn实现tcp和udp共存"
date: 2020-05-20T10:54:18+08:00
tags: [ops,vpn,udp,tcp,server]
categories: [ops,iptables]
---


[toc]

> 背景: 疫情期间， 在家办公让vpn突然火了起来。openvpn的udp和tcp两种模式都可以正常工作，  因为公司是通过动态拨号上网，没有固定的外网地址，所以VPN是通过映射到内网来实现. 由于端口映射导致tls错误，将udp协议改成了tcp协议。 解决了映射相关错误后 ，又重新开启了udp端口。

`$表示bash shell, #表示注释, > 表示数据库`

## 安装配置
修改配置文件

```
$ cd /etc/openvpnserver/
$ cp server.conf udp.conf
$ vim udp.conf

# 修改proto
proto udp
# 修改ip, 将老的ip替换为其他的即可, 其他配置暂时不用修改
server 10.8.0.0 255.255.255.0

```

配置systemd文件

```
$ cat /lib/systemd/system/openvpn@.service 
[Unit]
Description=OpenVPN Robust And Highly Flexible Tunneling Application On %I
After=network.target

[Service]
Type=notify
PrivateTmp=true
ExecStart=/usr/sbin/openvpn --cd /etc/openvpnserver/ --config %i.conf

[Install]
WantedBy=multi-user.target

```
解释一下

- systemd的配置文件 %i 为占位符,  Unit 文件名中在 @ 符号之后的部分，不包括 @ 符号和 .service 后缀名, 详细的占位符可以查看官方文档[官方](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Specifiers)

## 启动

```
$ systemctl start openvpn@server
$ systemctl start openvpn@udp
$ systemctl status openvpn@server
● openvpn@server.service - OpenVPN Robust And Highly Flexible Tunneling Application On server
   Loaded: loaded (/usr/lib/systemd/system/openvpn@.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2020-05-19 09:29:31 CST; 1 day 5h ago
 Main PID: 3545 (openvpn)
   Status: "Initialization Sequence Completed"
   CGroup: /system.slice/system-openvpn.slice/openvpn@server.service
           └─3545 /usr/sbin/openvpn --cd /etc/openvpnserver/ --config server.conf

May 19 09:29:31 server11-new systemd[1]: Starting OpenVPN Robust And Highly Flexible Tunneling Applicati...er...
May 19 09:29:31 server11-new systemd[1]: Started OpenVPN Robust And Highly Flexible Tunneling Applicatio...rver.
Hint: Some lines were ellipsized, use -l to show in full.
$  systemctl status openvpn@udp
● openvpn@udp.service - OpenVPN Robust And Highly Flexible Tunneling Application On udp
   Loaded: loaded (/usr/lib/systemd/system/openvpn@.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-05-20 11:34:08 CST; 3h 22min ago
 Main PID: 6212 (openvpn)
   Status: "Initialization Sequence Completed"
   CGroup: /system.slice/system-openvpn.slice/openvpn@udp.service
           └─6212 /usr/sbin/openvpn --cd /etc/openvpnserver/ --config udp.conf

May 20 11:34:08 server11-new systemd[1]: Starting OpenVPN Robust And Highly Flexible Tunneling Applicati...dp...
May 20 11:34:08 server11-new systemd[1]: Started OpenVPN Robust And Highly Flexible Tunneling Applicatio... udp.
Hint: Some lines were ellipsized, use -l to show in full.

```

配置iptables转发, 因为服务器是内网, 内网ip基本不会变化, 这边用的SNAT替代了MASQUERADE. 

```
iptables -t nat -A POSTROUTING -s 172.8.8.0/24 -j SNAT --to-source 192.168.0.11
iptables -t nat -A POSTROUTING -s 10.8.0.0/24  -j SNAT --to-source 192.168.0.11
```

## 开机自启动

```
$ systemctl enable openvpn@server
$ systemctl enable openvpn@udp
```

配置完成. 

## 客户端配置文件秘钥合并

```
## 可以合并到脚本文件里面.
$ echo "
client
dev tun
proto tcp
remote 123.45.67.89 6666

resolv-retry infinite
nobind
persist-key
persist-tun
<key>
$(cat /etc/openvpnclient/${vpn_user}.key)
</key>
<cert>
$(openssl x509 -in /etc/openvpnclient/${vpn_user}.crt)
</cert>
<ca>
$(cat /etc/openvpnclient/ca.crt)
</ca>
remote-cert-tls server

comp-lzo
verb 3
" > /etc/openvpnclient/new/${vpn_user}.ovpn
```

### 参考

- [SNAT和MASQUERADE的区别](https://www.cnblogs.com/Dicky-Zhang/p/5934657.html)
- [tun和tap模式的区别](https://www.jianshu.com/p/09f9375b7fa7)
- [Tun/Tap设备基本原理](https://github.com/ICKelin/article/issues/9)

- [centos7安装openvpn](https://www.xbzdr.com/263.html)

谢谢您的观看

