---
title: "Git in Gogs with SourceTree"
date: 2019-09-11 15:49:10
tags: [git,gogs,sourceTree]
categories: [git]
---

[TOC]

## 背景

> 使用`git clone https://url.com/project/web.git`速度比较慢.
>
> 直接使用`git clone git@url.com:project/web.git`速度相对较快.而且比较简洁.

## 使用方法

### 服务端配置

- 生成自己的ssh秘钥,如果你已经有了自己的秘钥,可以跳过这一步

```shell
$ ssh-keygen
## 一路enter即可生成,
## 生成的秘钥在 ~/.ssh/id_rsa,保存好
## 公钥在 ~/.ssh/id_rsa.pub
```

- 将自己的公钥添加到git服务器端, 这里使用gogs作为轻量的git服务器端.其他的git服务器端类似.
- 点击个人`账户设置` ==> `SSH 秘钥` ==> `增加密钥`,将生成的`id_rsa.pub`信息填入保存即可.

![1568185919212](http://pic.fenghong.tech/1568185919212.png)

### SourceTree端的配置

- 检查是否能git直连服务器,出现以下信息说明能使用git进行clone\push\pull等常规操作了.

```shell
$ ssh -T git@url.com
Hi there, You've successfully authenticated, but Gogs does not provide shell access.
If this is unexpected, please log in with password and setup Gogs under another user.
```

- 配置SourceTree

打开sourceTree软件,点击主菜单的`工具` ==> `选项` ==> `SSH 密钥`地址,`SSH客户端`选择`OpenSSH`.

![1568186314623](http://pic.fenghong.tech/1568186314623.png)

- 配置仓库

公司的项目一般很多,比较分散,要一个一个替换的确比较麻烦,也比较费脑.这里写了一个命令来帮助大家一键更改.windows上面的`git-bash`是支持的,可以放心使用.

```
## 1. 查找匹配需要替换的url路径,一般源码都是放在一个总的文件夹,然后其他的子项目,或者子子项目都在子文件夹.
$ find ./ -maxdepth 4  -name 'config'  |xargs grep "repodomain"
./chizhan/baodun/.git/config:   url =https://repodomain/chizhan/baodun.git
./chizhan/cz-springbootx/.git/config:   url =https://repodomain/chizhan/springbootx.git

参数说明 : -maxdepth 4 只在前4层目录进行查找. 
比如 ./chizhan/baodun/.git/ 这个为三层,则config必定在第四层.即我们需要更改的文件配置.


## 2. 真正的替换操作,谨慎操作,谨慎操作,谨慎操作,重要的说三遍,先查找一下确定是你要修改的,再执行此命令!!
$  find ./ -maxdepth 4  -name 'config'  |xargs sed -i "s&https://gogs.qianxiangbank.com/& git@gogs.qianxiangbank.com:&g"
```

- 享受非一般的速度吧~

## 总结

https中间的转换多了nginx的反向代理的一层网络,多了一层开销,会有一定的延迟,使用git相当于直连了.所以速度提升了.