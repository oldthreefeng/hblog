---
title: "mysql lost password"
date: 2018-12-11 17:05:12
tags: 
  - mysql
  - password
categories: [mysql]
---
[TOC]

## mysql 数据库密码忘记

一直以来，对于MySQL root密码的忘记，以为只有一种解法-skip-grant-tables。

问了下群里的大咖，第一反应也是skip-grant-tables。通过搜索引擎简单搜索了下，无论是百度，抑或Google，只要是用中文搜索，首页都是这种解法。可见这种解法在某种程度上已经占据了使用者的心智。下面具体来看看。

### **skip-grant-tables的解法**

**首先，关闭实例**

这里，只能通过kill mysqld进程的方式。


```
# ps -ef |grep mysqld
root      6220  6171  0 08:14 pts/0    00:00:00 /bin/sh bin/mysqld_safe --defaults-file=my.cnf
mysql      6347  6220  0 08:14 pts/0    00:00:01 /usr/local/mysql57/bin/mysqld --defaults-file=my.cnf --basedir=/usr/local/mysql57 --datadir=/usr/local/mysql57/data --plugin-dir=/usr/local/mysql57/lib/plugin --user=mysql --log-error=slowtech.err --pid-file=slowtech.pid --socket=/usr/local/mysql57/data/mysql.sock --port=3307
root      6418  6171  0 08:17 pts/0    00:00:00 grep --color=auto mysqld

# kill 6347  或者 kill -9  6220  6347
```



**使用--skip-grant-tables参数，重启实例**

```
# bin/mysqld_safe --defaults-file=my.cnf --skip-grant-tables  --skip-networking &
```

设置了该参数，则实例在启动过程中会跳过权限表的加载，这就意味着任何用户都能登录进来，并进行任何操作，相当不安全。

建议同时添加--skip-networking参数。其会让实例关闭监听端口，自然也就无法建立TCP连接，而只能通过本地socket进行连接。

MySQL8.0就是这么做的，在设置了--skip-grant-tables参数的同时会自动开启--skip-networking。

 

**修改密码**

```
# mysql -S /usr/local/mysql57/data/mysql.sock

mysql> update mysql.user set authentication_string=password('123456') where host='localhost' and user='root';
Query OK, 0 rows affected, 1 warning (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 1

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```



注意：

这里的update语句针对的是MySQL 5.7的操作，如果是在5.6版本，修改的应该是password字段，而不是authentication_string。

```
update mysql.user set password=password('123456') where host='localhost' and user='root';
```



而在MySQL 8.0.11版本中，这种方式基本不可行，因为其已移除了PASSWORD()函数及不再支持SET PASSWORD ... = PASSWORD ('auth_string')语法。

不难发现，这种方式的可移植性实在太差，三个不同的版本，就先后经历了列名的改变，及命令的不可用。



**下面，介绍另外一种更通用的做法，还是在skip-grant-tables的基础上。**

与上面不同的是，其会先通过flush privileges操作触发权限表的加载，再使用alter user语句修改root用户的密码，如：



```
# bin/mysql -S /usr/local/mysql57/data/mysql.sock

mysql> alter user 'root'@'localhost' identified by '123';
ERROR 1290 (HY000): The MySQL server is running with the --skip-grant-tables option so it cannot execute this statement

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> alter user 'root'@'localhost' identified by '123';
Query OK, 0 rows affected (0.00 sec)
```

免密码登录进来后，直接执行alter user操作是不行的，因为此时的权限表还没加载。可先通过flush privileges操作触发权限表的加载，再执行alter user操作。

需要注意的是，通过alter user修改密码只适用于MySQL5.7和8.0，如果是MySQL 5.6，此处可写成

```
update mysql.user set password=password('123456') where host='localhost' and user='root';
```

### **更优雅的解法**

相对于skip-grant-tables方案，我们来看看另外一种更优雅的解法，其只会重启一次，且基本上不存在安全隐患。

首先，依旧是关闭实例

其次，创建一个sql文件

写上密码修改语句

```
# vim init.sql 
alter user 'root'@'localhost' identified by '123456';
```

 

最后，使用--init-file参数，启动实例

```
# bin/mysqld_safe --defaults-file=my.cnf --init-file=/usr/local/mysql57/init.sql &
```

实例启动成功后，密码即修改完毕~

 

如果mysql实例是通过服务脚本来管理的，除了创建sql文件，整个操作可简化为一步。

```
# service mysqld restart --init-file=/usr/local/mysql57/init.sql 
```

注意：该操作只适用于/etc/init.d/mysqld这种服务管理方式，不适用于RHEL 7新推出的systemd。