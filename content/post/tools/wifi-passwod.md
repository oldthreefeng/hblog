---
title: "查看WIFI历史密码"
date: 2017-11-16 15:56:12
tags: [learning,mac,wifi]
categories: [tools]
---

[TOC]

> 背景: WIFI密码老是忘记, 每次找起来特别麻烦, 有时候的重启路由器相关设置, 重新设置wifi密码, 这很困扰, 因此产生此篇文章.

## mac查看WIFI历史密码

### terminal

打开`terminal`, 替换成相应的wifi名称.

```
$ sudo security find-generic-password -ga "替换成wifi名称" | grep password
```

### keyChain

打开应用程序中『实用工具』文件夹中的『钥匙串访问』

![查看MAC系统中原来已经保存的wifi密码](https://oss.fenghong.tech/tools/2016050520582085637.png)



选择左侧的『密码』，就可以看到右侧有『airport网络密码』了

![查看MAC系统中原来已经保存的wifi密码](https://oss.fenghong.tech/tools/2016050521005138149.png)



双击需要查看密码的wifi名称，勾选下方的『显示密码』。根据提示输入系统密码后，wifi的密码就显示出来了。

![查看MAC系统中原来已经保存的wifi密码](https://oss.fenghong.tech/tools/2016050521035114828.png)

## Windows查看历史wifi密码

### terminal

`win+R`输入cmd. 进入terminal终端;

如果连WIFI名称记得不清楚,这个`netsh wlan show profiles`可以显示所有连接的WIFI历史信息

```bat
C:\Users\Administrator> netsh wlan show profiles

Profiles on interface WLAN:

Group policy profiles (read only)
---------------------------------
    <None>

User profiles
-------------
    All User Profile     : ucloud-guest
    All User Profile     : CMCC-vz2X
    All User Profile     : ROUTER-001-0043
    All User Profile     : ChinaNet-Starbucks
    All User Profile     : TP-LINK_5G_858F
    All User Profile     : CMCC-w9A4
    All User Profile     : @701_5G
    All User Profile     : @701
    All User Profile     : qianxiangG_A_5G
    All User Profile     : qianxiangG_B_5G
    All User Profile     : qianxiangG_C
    All User Profile     : qianxiangG_B
    All User Profile     : qianxiangG_A
    All User Profile     : LOVE_FENG
    All User Profile     : 混蛋！你问啥问
    All User Profile     : WELUV_TRV
    All User Profile     : explorers0128
    All User Profile     : NIMABO
    All User Profile     : “Coco”的 iPhone
    All User Profile     : ChinaNet-CHu5-5G
    All User Profile     : Mi8
```

找到自己想要记得的WIFI的名称,比如`LOVE_FENG`是我的WIFI名称

 输入`netsh wlan show profiles "LOVE_FENG" key=clear| find "Key"`

```bat
C:\Users\Administrator> netsh wlan show profiles "LOVE_FENG" key=clear |find "Key"

// 输出如下密码信息
Key Content            : BABYf520...  //这项就是密码了
```

### ~~一行命令~~

~~如果不想一行一行找,直接输入以下命令自己慢慢找~~

```bat
C:\Users\Administrator>  for /f "skip=9 tokens=1,2 delims=:" %i in ('netsh wlan show profiles') do @echo %j | findstr -i -v echo | netsh wlan show profiles %j key=clear 
```

### 参考

- [windows的文本工具findstr](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/findstr)

谢谢