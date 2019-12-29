---
title: "mysql执行sql导入数据库中文乱码"
date: 2019-12-29 11:05:12
tags: [mysql,coding]
categories: [mysql]
---
[TOC]

- \# 开头的行表示注释
- \> 开头的行表示需要在 mysql 中执行
- $ 开头的行表示需要执行的命令

> 背景: 生产环境mysql执行sql脚本, 因疏忽大意, 没检查中文编码,
> 导致数据里面数据中文部分乱码. 特此总结记录. 并给出解决方案
## mysql 导入中文数据乱码

先看一段sql, 里面有中文`微信公众号绑定`

```sql
-- cat 2.sql
INSERT INTO `base_enterprise_menu` VALUES
 ('1210077099628462080', null, '1210077099620073472', 'merchant_mall_manager', 'WechatBind', '微信公众号绑定', '', '2-10-3-1', null, '2019-12-26 13:57:17', '1');
```
直接在服务器执行, 后面获取字段的肯定就乱码了.

```bash
$ mysql -uroot -ppassword  yourdb  < 2.sql
```

## 解决方案

### 使用Navicat执行sql
 
把文件内容复制下来, 在`Navicat mysql`里面执行

### 使用命令行添加参数

```bash
$ mysql -uroot -ppassword  yourdb  < 2.sql --default-character-set = utf8
```

### 使用sql命令行

```bash
$ mysql -uroot -ppassword -hhost
> use yourdb;
> set names utf8;
> source 2.sql;
```
### 改变sql文件的编码

```bash
$ file 2.sql 
2.sql: ISO-8859 text, with CRLF line terminators
$ cat 2.sql
INSERT INTO `base_enterprise_menu` VALUES
 ('1210077099628462080', null, '1210077099620073472', 'merchant_mall_manager', 'WechatBind', '微信公众号绑定', '', '2-10-3-1', null, '2019-12-26 13:57:17', '1');
$ vi 2-utf8.sql ## 将内容复制过来后, :wq保存
$ file 2-utf8.sql
2-utf8.sql: UTF-8 Unicode C program text, with CRLF line terminators
```
