---
title: "sonarqube 权限配置使用"
date: 2019-03-11 08:43:12
tags: [java,sonar]
categories: [java,sonar]
---

[TOC]

## 权限控制

- 新建群组 

管理员登陆，新建用户群组`CAJX-group`，手动将对应项目组用户添加至群组中：

![img](https://oss.fenghong.tech/sonarqube/sonarqube2.jpg)

- 新建权限模板 

在新建权限模板时需要指定过滤条件，比如项目以CAJX开头，就在过滤条件中添加CAJX.*，下次新建的项目就会根据此过滤条件自动加入CAJX-group。 

![img](https://oss.fenghong.tech/sonarqube/sonarqube1.jpg)

![img](https://oss.fenghong.tech/sonarqube/sonarqube5.jpg)

- 授权用户权限

创建CAJX-01用户，管理CAJX项目组

![img](https://oss.fenghong.tech/sonarqube/sonarqube3.jpg)

- 管理项目

创建项目标识，以CAJX开头的都会计入CAJX项目组，如果应用失败，可以手动应用至改权限模板。

![img](https://oss.fenghong.tech/sonarqube/sonarqube6.jpg)

- 访问

用对应权限账号登陆，即可看到属于自己群组的项目。

![img](https://oss.fenghong.tech/sonarqube/sonarqube7.jpg)