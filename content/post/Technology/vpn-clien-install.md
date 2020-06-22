---
title: "vpn install"
date: "2019-01-18 20:11:12"
tags: [ops,vpn]
categories: [vpn]
---

[TOC]

### 安装vpn

双击`openvpn-install-2.3.6-I603-x86_64`.进入安装界面直至安装成功。
记住自己的安装目录。
![图片描述](https://pic.fenghong.tech/others/tapd_23280401_base64_1547807430_68.png)

### 安装秘钥

解压`qianxiangc`
生成两个文件夹`__MACOSX`和`qianxiangc`.
![图片描述](https://pic.fenghong.tech/others/tapd_23280401_base64_1547807282_69.png)
打开安装的目录。`C:\Program Files\OpenVPN`
进入安装目录，打开`config`文件夹,将解压的两个文件夹复制过来。
![图片描述](https://pic.fenghong.tech/others/tapd_23280401_base64_1547807545_100.png)

### 启动vpn

![图片描述](https://pic.fenghong.tech/others/tapd_23280401_base64_1547807627_27.png)

#### win10系统

右键属性，在目标的位置 `"C:\Program Files\OpenVPN\bin\openvpn-gui.exe" --connect qianxiang.ovpn`,如图
![图片描述](https://pic.fenghong.tech/others/tapd_23280401_base64_1547807698_66.png)

#### win7系统

直接以管理员身份启动vpn即可。

