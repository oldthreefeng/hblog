---
title: jumpserver用户使用
date: 2018-08-28 09:59:32
urlname: jpserver2
tags: 
- Linux
- Jumpserver
- MFA
categories: server
---

# 用户页面

通过管理员发送的邮件里面的 Jumpserver 地址登录进行用户初始化


![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537346899_31.png)


## 1.0 添加密码及MFA

点击设置密码

```
要求10位数，必须有大写字母，其他无所谓
```

![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537346915_59.png)
```
登录名为：您的英文名（全小写）。
密码为：您刚设置的密码。
```
![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537346932_41.png)


```
点击下一步--->
	跳过google应用--->
		点击下一步---->
```
![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537346943_80.png)


- MFA

到达上图所示界面时，即开始验证MFA，建议使用阿里云的MFA，不要使用google的,所以跳过了这个过程，下面这个是阿里云网址截图的app。

![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537346966_26.png)


下载阿里云app完毕后，进入app界面（建议不要卸载）。

```
点击---->>
	控制台----->>	
		虚拟MFA----->>
			点击右上角+，扫描添加MFA，绑定成功返回登录,登录成功后填写个人信息。
```



## 1.1 查看个人信息

个人信息页面展示了用户的名称、角色、邮件、所属用户组、SSh 公钥、创建日期、最后登录日期和失效日期等信息：

![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537337395_70.png)

## 1.2 修改密码

在个人信息页面点击"更改密码"按钮，跳转到修改密码页面，正确输入新旧密码，即可完成密码修改:

![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537337414_83.png)

## 1.3 设置或禁用 MFA

在个人信息页面点击"设置MFA"按钮（设置完成后按钮会禁用MFA），根据提示处理即可，MFA全称是Multi-Factor Authentication，遵循（TOTP）标准（RFC 6238）

## 1.4 修改 SSH 公钥

点击"重置 SSH 密钥"按钮，跳转到修改 SSH 密钥信息页，复制 SSH 密钥信息到指定框中，即可完成 SSH 密钥修改：

查看 SSH 公钥信息：

```
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDadDXxxx......
```

![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537337436_46.png)

## 1.5 查看个人资产

自己被授权的资产，增减授权资产的需求请联系管理员

![图片描述](http://pic.fenghong.tech/tapd_23280401_base64_1537337460_35.png)
