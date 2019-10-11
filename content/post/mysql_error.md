---
title: "mysql常见错误汇总及解决方案"
date: 2018-11-15 17:59:32
urlname: mysql11
tags: 
- mysql
- error
categories: MYSQL
---

近期mysql数据库报错，及解决方案；记录一下

## mysql报错汇总

**626**

```
mysqldump: Error 2020: Got packet bigger than 'max_allowed_packet' bytes when dumping table `blt_bulletinannex` at row: 626
```

报错条件：一般是存在blob，text等字段，单条记录超过默认的24M
解决措施：mysqldump调大max_allow_packet参数，在服务器端修改这个参数无效

**1143**

```
mysqldump: Couldn't execute 'show table status like 'members\_ban\_user\_view'': SELECT command denied to user ''@'%' for column 'user_id' in table 'members_ban_log' (1143)
```

报错条件：相应的视图的账户给的权限不足；或者是用户不存在
解决措施：需要视图定义账户的Create_view_priv和Show_view_priv权限；或者添加对应的用户和权限；删除该视图

**1146**

```
mysqldump: Couldn't execute 'show create table `innodb_index_stats`': Table 'MySQL.innodb_index_stats' doesn't exist (1146)
```

报错条件：mysql5.6，系统表损坏，该表是innodb引擎
解决措施：物理删除该表的frm文件和ibd文件，找到系统表的定义sql，重建系统表

解决措施：删除或修改出问题的视图定义语句

**1045**

```
mysqldump: Got error: 1045: Access denied for user 'ucloudbackup'@'10.10.1.242' (using password: YES) when trying to connect
```

报错条件：无法连接，密码，账户，host，port有问题
解决措施：先保证mysql能正常连接

**145**

```
mysqldump: Couldn't execute 'show create table `userarenalog`': Table './tank_11/userarenalog' is marked as crashed and should be repaired (145)
```

报错条件：myisam表损坏
解决措施：repair table XXX修复损坏的表，最好mysqlcheck一下所有表

**126**

```
mysqldump: Couldn't execute 'show fields from `TB_CROWDFUNDING_PROJECT`': Incorrect key file for table 'ql-5.5/14310da6-644a-472a-b170-0e7e75cfda87/tmp/#sql_32606_0.MYI'; try to repair it (126)
```

报错条件：临时表使用过程中/tmp空间不足，导致myisam临时表损坏
解决措施：增大磁盘空间就好

**1548**

```
mysqldump: Couldn't execute 'SHOW FUNCTION STATUS WHERE Db = 'analysis'': Cannot load from mysql.proc. The table is probably corrupted (1548)
```

报错条件：升级导致
解决措施：运行mysql_upgrade更新db，或者更新对应版本的mysql.proc表结构
5.1执行

```
mysql> alter table mysql.proc MODIFY COLUMN `comment` char(64) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL AFTER `sql_mode`;
```

5.5执行

```
mysql> alter table mysql.proc MODIFY COLUMN `comment` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL AFTER `sql_mode`;
```

**1577**

```
mysqldump: Couldn't execute 'show events': Cannot proceed because system tables used by Event Scheduler were found damaged at server start (1577)
```

报错原因：不合理的升级mysql版本导致
解决措施：先mysql_upgrade,不行再重启db看看(不大确定)

**1064**

```
mysqldump: Couldn't execute 'SHOW FUNCTION STATUS WHERE Db = 'mysql'': You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1 (1064)
```

报错原因：select * from information_schema.ROUTINES limit 1报一样的错误
再查看mysql.proc表发现有函数或者存储过程定义有问题，比如根本不存在的db或者user出现在定义中，猜测是备份时没有加-R参数，直接导入到db后没有正常建立对应的函数或存储过程导致的
解决措施：先尝试drop语法删除mysql.proc中定义有问题的函数或存储过程记录，如果不行就直接delete from的方式删除

**2013**

```
mysqldump: Error 2013: Lost connection to MySQL server during query when dumping table `vitality_flow` at row: 31961089
```

报错原因：1 该表是分区表 2 该表是innodb，存在大量的blob text等字段 3 上传NFS或者边备份边压缩
解决措施：针对1和3的原因，需要调大net_write_timeout参数；针对2的原因，需要调大max_allow_packet；

**2013**

```
mysqldump: Couldn't execute 'SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('15616156','mysql','test','wx00','wx01','wx02','wx03','wxid')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME': Lost connection to MySQL server during query (2013)
```

报错原因：备份过程中因内存不足而oom
解决措施：加大内存

**1213**

```
mysqldump: Couldnt execute show create table `shop_his_9`: Deadlock found when trying to get lock; try restarting transaction (1213) fails
```

报错原因：mysqldump过程中发生死锁
解决措施：重试即可

**1049**

```
mysqldump: Got error: 1049: Unknown database cfcara when selecting the database fails
```

报错原因：随意修改大小写敏感问题导致
解决措施：先解决大小写问题

**1045**

```
mysqldump: Couldn't execute 'STOP SLAVE SQL_THREAD': Access denied for user 'root'@'172.19.%.%' (using password: NO) (1045)
```

报错原因：从库备份，备份账户权限不足，无法登陆

**1168**

```
mysqldump: Couldnt execute show create table `sk_order_38`: Unable to open underlying table which is differently defined or of non-MyISAM type or doesnt exist (1168) fails
```

报错原因：mrg表定义出错导致的吧
解决措施：把这个表删除

**启动服务失败**

```
ERROR! The server quit without updating PID file (/udisk/mysql/mysql/qx_sit.pid).
```

报错原因： 权限错误
解决措施： 

```
# chown -R mysql.mysql /data/mysql/
```
**1820**
安装数据库时，必须修改密码才可以执行sql语句。

```
mysql> SELECT 1;
ERROR 1820 (HY000): You must SET PASSWORD before executing this statement

mysql> SET PASSWORD = PASSWORD('new_password');
Query OK, 0 rows affected (0.01 sec)
```
**1418**
在导入数据的时候，报如下错误。

```
ERROR 1418 (HY000) at line 22997: This function has none of DETERMINISTIC, NO SQL, or READS SQL DATA in its declaration and binary logging is enabled (you *might* want to use the less safe log_bin_trust_function_creators variable)

##解决措施
$ mysql -uroot -p
mysql> SET GLOBAL log_bin_trust_function_creators = 1;
```
