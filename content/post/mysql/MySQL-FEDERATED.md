---
title: "MySQL FEDERATED"
date: 2019-05-23 10:44:32
tags: [mysql,federated]
categories: [mysql]
---

##  FEDERATED 存储引擎描述

```
FEDERATED存储引擎能让你访问远程的MySQL数据库而不使用replication或cluster技术(类似于Oracle的dblink),使用FEDERATED存储引擎的表,本地只存储表的结构信息,数据都存放在远程数据库上,查询时通过建表时指定的连接符去获取远程库的数据返回到本地。
FEDERATED存储引擎默认不启用
    如果是使用的源码，需要使用CMake 加上DWITH_FEDERATED_STORAGE_ENGINE选项。
    如果是二进制包,则在启动MySQL时指定 [--federated] 选项开启或在my.cnf文件中的[mysqld]部分加上federated参数
```

 

##  FEDERATED 存储引擎架构

```
1 本地服务器 FEDERATED 存储引擎的表只存放表的.frm结构文件
2 远程服务器 存放了.frm和数据文件
3 增删改查操作都是通过建立的连接来访问远程数据库进行操作,把结果返回给本地。
4 远程数据表的存储引擎为MySQL支持的存储引擎,如MyISAM,InnoDB等
```

 

![img](https://images2015.cnblogs.com/blog/610544/201603/610544-20160330120217348-765684434.png)

 

##  FEDERATED 存储引擎操作步骤

```mysql
    远程库:
        开启 FEDERATED 存储引擎
        建立远程访问用户
        授予访问对象的权限
    本地库：
        测试登陆远程库是否能成
        创建 FEDERATED 表
        查询是否成功
        
select engine,support from information_schema.engines where engine='FEDERATED';
+-----------+---------+
| engine | support |
+-----------+---------+
| FEDERATED | YES |
+-----------+---------+
1 row in set (0.00 sec)

--也可使用show engines查看支持的存储引擎

(root@localhost) [(none)]>show engines;  --排版问题不贴出执行结果

--如果support 为NO,则需要在my.cnf中[mysqld]增加federated参数,并重启MySQL服务器生效配置
```

### 建立远程访问用户并授权

 

```
(root@localhost) [(none)]>select user,host from mysql.user;   --查看数据库用户
+-----------+-----------+
| user      | host      |
+-----------+-----------+
| root      | %         |
| mysql.sys | localhost |
| root      | localhost |
+-----------+-----------+
3 rows in set (0.00 sec)

(root@localhost) [(none)]>create user 'fed'@'%' identified by 'fed_test';  --创建一个federated连接的用户
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]>grant all on employees.* to 'fed'@'%';   --授予创建的fed用户访问employees数据库所有表的权限
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]>select user,host from mysql.user;   --查看用户信息
+-----------+-----------+
| user      | host      |
+-----------+-----------+
| fed       | %         |
| root      | %         |
| mysql.sys | localhost |
| root      | localhost |
+-----------+-----------+
4 rows in set (0.00 sec)

(root@localhost) [(none)]>show grants for 'fed'@'%';   --查看用户权限
+----------------------------------------------------+
| Grants for fed@%                                   |
+----------------------------------------------------+
| GRANT USAGE ON *.* TO 'fed'@'%'                    |
| GRANT ALL PRIVILEGES ON `employees`.* TO 'fed'@'%' |
+----------------------------------------------------+
2 rows in set (0.00 sec)
```

 

 

## 本地库

### 测试登陆远程库是否能成

```
[root@RHEL6 ~]# mysql -ufed -h172.25.21.10 -P3306 -p   --在本地服务器上去连接远程库
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 23
Server version: 5.7.10-log MySQL Community Server (GPL)
Copyright (c) 2000, 2014, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

fed@172.25.21.10 [(none)]>   --成功通过远程机新建的用户登录上远程数据库

--如果没有连接成功
　 判断是否防火墙的问题
   查看端口是否正确
```

### 创建本地 FEDERATED 表

```
root@localhost [(none)]>create database test;    --在本地创建一个数据库,也可使用已存在的数据库
Query OK, 1 row affected (0.01 sec)

root@localhost [(none)]>use test;    --使用新建的数据库
Database changed
root@localhost [test]>CREATE TABLE `employees_fed` (
    ->   `emp_no` int(11) NOT NULL,
    ->   `birth_date` date NOT NULL,
    ->   `first_name` varchar(14) NOT NULL,
    ->   `last_name` varchar(16) NOT NULL,
    ->   `gender` enum('M','F') NOT NULL,
    ->   `hire_date` date NOT NULL,
    ->   PRIMARY KEY (`emp_no`)
    -> ) ENGINE=federated DEFAULT CHARSET=utf8mb4
    -> connection='mysql://fed:fed_test@172.25.21.10:3306/employees/employees';   
Query OK, 0 rows affected (0.00 sec)

connection语法：
scheme://user_name[:password]@host_name[:port_num]/db_name/tbl_name

具体语法及含义参考官方文档链接:
http://dev.mysql.com/doc/refman/5.7/en/federated-create-connection.html

--创建完成federated存储引擎的表,注意:本地表employees_fed的结构要和远程表employees一样,可以提前在远程表中通过show create table table_name来获取表结构并修改或增加标红的语句。
```

### 验证是否配置成功

```
root@localhost [test]>select * from employees_fed limit 10;    --成功获取到远程数据库中的数据
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |
+--------+------------+------------+-----------+--------+------------+
10 rows in set (0.36 sec)
```

```
root@localhost [test]>show variables like 'datadir';   --查看数据文件存放目录
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| datadir       | /var/lib/mysql/ |
+---------------+-----------------+
1 row in set (0.00 sec)

root@localhost [test]>system ls -l /var/lib/mysql/test/* 
-rw-rw----. 1 mysql mysql   61 Mar 30 13:41 /var/lib/mysql/test/db.opt
-rw-rw----. 1 mysql mysql 8768 Mar 30 13:41 /var/lib/mysql/test/employees_fed.frm   --确定本地只保存了表的结构信息

--配置成功
```

## **使用 CREATE SERVER 方式创建 FEDERATED表**



```
--创建一个server

root@localhost [test]>CREATE SERVER emp_link
    -> FOREIGN DATA WRAPPER mysql
    -> OPTIONS (USER 'fed', PASSWORD 'fed_test',HOST '172.25.21.10',PORT 3306,DATABASE 'employees');

CREATER SERVER语法：
CREATE SERVER server_name
FOREIGN DATA WRAPPER wrapper_name
OPTIONS (option [, option] ...)

具体语法及含义参考官方文档链接:
http://dev.mysql.com/doc/refman/5.7/en/federated-create-server.html

--查看已创建的server


root@localhost [test]>select * from mysql.servers\G;
*************************** 1. row ***************************
Server_name: emp_link
Host: 172.25.21.10
Db: employees
Username: fed
Password: fed_test
Port: 3306
Socket: 
Wrapper: mysql
Owner: 
1 row in set (0.00 sec)

```

### 创建基于SERVER 的FEDERATED表

```
root@localhost [test]>CREATE TABLE `employees_link` (
    ->   `emp_no` int(11) NOT NULL,
    ->   `birth_date` date NOT NULL,
    ->   `first_name` varchar(14) NOT NULL,
    ->   `last_name` varchar(16) NOT NULL,
    ->   `gender` enum('M','F') NOT NULL,
    ->   `hire_date` date NOT NULL,
    ->   PRIMARY KEY (`emp_no`)
    -> ) ENGINE=FEDERATED DEFAULT CHARSET=utf8mb4
    -> CONNECTION='emp_link/employees';
Query OK, 0 rows affected (0.01 sec)
```

### 验证是否配置成功

```
root@localhost [test]>select * from employees_link limit 10;
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
|  10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |
|  10006 | 1953-04-20 | Anneke     | Preusig   | F      | 1989-06-02 |
|  10007 | 1957-05-23 | Tzvetan    | Zielinski | F      | 1989-02-10 |
|  10008 | 1958-02-19 | Saniya     | Kalloufi  | M      | 1994-09-15 |
|  10009 | 1952-04-19 | Sumant     | Peac      | F      | 1985-02-18 |
|  10010 | 1963-06-01 | Duangkaew  | Piveteau  | F      | 1989-08-24 |
+--------+------------+------------+-----------+--------+------------+
10 rows in set (0.35 sec)
--配置成功
--这种方式的好处在于创建本地FEDERATED表时,在connection中直接指定已经创建好的server link,不需要每次都配置一个新的连接。
--并且便于统一管理,只需要修改server link即可
```

## FEDERATED 引擎使用注意事项

```
1、FEDERATED 表可能会被复制到其他的slave数据库,你需要确保slave服务器也能够使用定义在connection中或mysql.servers表中的link的用户名/密码 连接上远程服务器。

2、远程服务器必须是MySQL数据库

3、在访问FEDERATED表中定义的远程数据库的表前,远程数据库中必须存在这张表。

4、FEDERATED 表不支持通常意义的索引,服务器从远程库获取所有的行然后在本地进行过滤,不管是否加了where条件或limit限制。

　　--查询可能造成性能下降和网络负载,因为查询返回的数据必须存放在内存中,所以容易造成使用系统的swap分区或挂起。

5、FEDERATED表不支持字段的前缀索引

6、FEDERATED表不支持ALTER TABLE语句或者任何DDL语句

7、FEDERATED表不支持事务

8、本地FEDERATED表无法知道远程库中表结构的改变

9、任何drop语句都只是对本地库的操作,不对远程库有影响
```













软件和硬件费用如下：

|    名称     | CPU总数 | 内存 | 硬盘大小 | 数量(台) | 价格(万元/年) | 总价(万元/年) |
| :---------: | ------- | ---- | -------- | -------- | ------------- | ------------- |
| 应用服务器  | 8       | 24 G | 320.0 G  | 10       | 1.1           | 11            |
| 数据库MySQL | 32      | 24 G | 200G     | 4        | 1.3           | 5.2           |
| redis服务器 | 32      | 1G   | 1G       | 1        | 0.1           | 0.1           |
|  台式电脑   | I5-8400 | 16G  | 1220G    | 54       | 0.5           | 27            |
|    合计     |         |      |          |          |               | 43.3          |

宽带费用:  服务器所用公网带宽为40M，每年的费用为4.5 万元。

云安全费用： 10 万元

短信费用： 5万元

测试手机费用：2.15万元

| 序号 | 型号               | 运行内存 | 存储空间 | 系统   | 价格(万元) |
| ---- | ------------------ | :------: | -------- | ------ | ---------- |
| 1    | iPhone6SP          |   2GB    | 32GB     | 10.3.3 | 0.5        |
| 2    | iPhone6            |   1GB    | 16GB     | 12.1.4 | 0.5        |
| 3    | iPhone5S           |   1GB    | 16GB     | 11.1.2 | 0.5        |
| 4    | 三星 SM-J5108      |   4GB    | 16GB     | 5.1.1  | 0.1        |
| 5    | 荣耀 SCL-AL00      |   2GB    | 8GB      | 5.1.1  | 0.05       |
| 5    | 荣耀 SCL-AL00      |   2GB    | 8GB      | 5.1.1  | 0.05       |
| 6    | 荣耀 CAM-TL00H     |   2GB    | 16GB     | 6.0.0  | 0.1        |
| 7    | 华为 P7-L07        |   2GB    | 16GB     | 5.1.1  | 0.05       |
| 8    | 红米手机           |   2GB    | 16GB     | 4.4.4  | 0.05       |
| 9    | 红米手机           |   2GB    | 16GB     | 5.1.1  | 0.05       |
| 10   | OPPOK1             |   4GB    | 64GB     | 8.1.0  | 0.1        |
| 11   | 荣耀10青春版 ALOOa |   4GB    | 64GB     | 9.0.1  | 0.1        |

智能办公费用： 9.88

| 名称             | 单价 | 数量 | 总价 |
| ---------------- | :--: | ---- | ---- |
| 远程视频会议系统 | 2.5  | 2    | 5    |
| 智能会议         | 1.9  | 2    | 3.8  |
| 环信即时通讯     | 0.54 | 2    | 1.08 |

合计： 74.83 万元