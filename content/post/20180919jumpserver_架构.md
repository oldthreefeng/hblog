---
title: jumpserver架构
date: 2018-09-08 09:59:32
urlname: jpserver3
tags: 
- Linux
- Jumpserver
- Coco
- Luna
- sftp
categories: server
---

# 架构说明

![组件架构图](http://pic.fenghong.tech/tapd_23280401_base64_1537347696_26.png)

# 组件说明

## Jumpserver

现指 Jumpserver 管理后台，是核心组件（Core）, 使用 Django Class Based View 风格开发，支持 Restful API。

[Github](https://github.com/jumpserver/jumpserver.git)

## Coco

实现了 SSH Server 和 Web Terminal Server 的组件，提供 SSH 和 WebSocket 接口, 使用 Paramiko 和 Flask 开发。

[Github](https://github.com/jumpserver/coco.git)

## Luna

现在是 Web Terminal 前端，计划前端页面都由该项目提供，Jumpserver 只提供 API，不再负责后台渲染html等。

[Github](https://github.com/jumpserver/luna.git)



# problem

luna 的页面不启动解决,2222未打开等问题

```
rm -f /data/opt/coco/keys/.access_key
```

重启服务即可，

```
登陆jumpserver，
打开会话管理----->
打开终端管理----->
更新即可
```



# 用户资产

## Web 连接资产

点击页面左边的 Web 终端：

[![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537337879_68.png)

打开资产所在的节点：

[![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537337733_20.png)

点击资产名字，就连上资产了，如果显示连接错误，请联系管理员解决

## SSH 连接资产

咨询管理员 跳板机服务器地址 及 端口 ，使用 ssh 方式输入自己的用户名和密码登录（与Web登录的用户密码一致）

[![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537337623_63.png)

## SSH 主机登出

推荐退出主机时使用 exit 命令或者 ctrl + d 退出会话

## SFTP 上传文件到 Linux 资产

咨询管理员 跳板机服务器地址 及 端口 ，使用 ssh 方式输入自己的用户名和密码登录（与 SSH 登录跳板机的用户密码一致）

连接成功后，可以看到当前拥有权限的资产，打开资产，然后选择系统用户，即可到资产的 /tmp 目录（/tmp 目录为管理员自定义）
