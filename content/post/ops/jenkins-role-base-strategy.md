---
title: "jenkins实现不同角色不同权限"
date: "2019-08-10 10:08:32"
tags: 
  - ops
  - jenkins
  - Authorization
categories: [ops,jenkins]
---

[TOC]

> 背景
>
> 公司的`jenkins`管理了很多的项目,不同项目的开发人员不同,需要对不同的人员进行权限分类,google了一下,在官网找到了[ Role-based Authorization Strategy](<https://wiki.jenkins.io/display/JENKINS/Role+Strategy+Plugin>)这个插件,基本能满足要求.

## 基于 Role based  Strategy

- 安装插件

在系统管理页面点击` Manage Jenkins` -->`Manage Plugins`-->`Available`,在`Filter`中输入`Role based`,找到我们想要的插件,安装即可.

- 配置使用插件

在系统管理页面点击`Configure Global Security` -->`Access Contral` ,在`Authorization`字段勾选`Role-based  Strategy`

![](https://oss.fenghong.tech/jenkins00.jpg)

> *官网上安全域设置为**Servlet**容器代理，实际操作发现**Jenkins**专有用户数据库也是可以的。*

### 配置权限

*在系统管理页面点击**Manage and Assign Roles**进入角色管理页面：* 

![](https://oss.fenghong.tech/jenkins02.jpg)

这里有两个参数,一个是`Manage Roles`,一个是`Assign Roles`

- 管理角色（Manage Roles）

选择该项可以创建全局角色、项目角色，并可以为角色分配权限。

在`global roles`添加用户组member.添加`all/read`权限

![](https://oss.fenghong.tech/jenkins03.jpg)

Project角色 就是可以根据不任务前缀 进行隔离，以下创建了 sonar 分组  ,该创建了2个角色，管理员 （具有配置构建等权限）普通角色（只有构建权限）

注意： Pattern 是任务前缀的匹配,必须要写`sonar.*`而不是`sonar`,当然中文也支持.例如任务名 sonar 开头的任务只会被sonar分组的用户看到.

![](https://oss.fenghong.tech/jenkins04.jpg)

然后sava退出.

- 分配角色权限

在系统管理页面点击**Manage and Assign Roles**进入分配角色页面：

在`global roles`中,将`sonar`和`sonarM`两个角色加入`member`这个管理组,

在`Item roles`中,将`sonar`用户加入sonar项目组;将`sonarM`用户加入sonarM项目管理组

![](https://oss.fenghong.tech/jenkins05.jpg)

然后save保存退出.

### 验证

使用sonarM用户登录,只有sonar开头的项目才能被展示,且拥有所有的管理权限

![](https://oss.fenghong.tech/jenkins06.jpg)

使用sonar用户登录,只有build权限

![](https://oss.fenghong.tech/jenkins07.jpg)