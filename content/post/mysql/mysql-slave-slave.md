---
title: "mysql slave slave"
date: 2019-07-16 15:56:12
tags: mysql
categories: [mysql]
---

### mysql-slave-slave级联复制

> Normally, a slave does not write to its own binary log any updates that are received from a master server. This option causes the slave to write the updates performed by its SQL thread to its own binary log. For this option to have any effect, the slave must also be started with the [`--log-bin`](https://dev.mysql.com/doc/mysql-replication-excerpt/5.6/en/replication-options-binary-log.html#option_mysqld_log-bin) option to enable binary logging. A warning is issued if you use the [`--log-slave-updates`](https://dev.mysql.com/doc/mysql-replication-excerpt/5.6/en/replication-options-slave.html#option_mysqld_log-slave-updates) option without also starting the server with the [`--log-bin`](https://dev.mysql.com/doc/mysql-replication-excerpt/5.6/en/replication-options-binary-log.html#option_mysqld_log-bin) option.

> [`--log-slave-updates`](https://dev.mysql.com/doc/mysql-replication-excerpt/5.6/en/replication-options-slave.html#option_mysqld_log-slave-updates) is used when you want to chain replication servers. For example, you might want to set up replication servers using this arrangement:

```none
A -> B -> C
```

> Here, `A` serves as the master for the slave `B`, and `B` serves as the master for the slave `C`. For this to work, `B` must be both a master *and* a slave. You must start both `A` and `B` with [`--log-bin`](https://dev.mysql.com/doc/mysql-replication-excerpt/5.6/en/replication-options-binary-log.html#option_mysqld_log-bin) to enable binary logging, and `B` with the [`--log-slave-updates`](https://dev.mysql.com/doc/mysql-replication-excerpt/5.6/en/replication-options-slave.html#option_mysqld_log-slave-updates) option so that updates received from `A` are logged by `B` to its binary log.

### master上

在A服务器上配置my.cnf

```
server_id = 1    #独一无二的，不可以重复
log-bin=mysql.bin
binlog-ignore-db=mysql 
skip-name-resolve   /*  dns 反向解析时间 * grant 时，必须使用ip不能使用主机名  */
```

### slave1上

在B服务器上，此服务器既是C的master，又是A的slave，如果没有配置好，导致主从级联复制失效，配置完重启mysql服务。

```
server-id=2 
read_only=TURE 
binlog-ignore-db=mysql 
log_slave_updates=1                            /*  关键一步 */         
log-bin=mysql.bin
```

### slave2上

在C服务器上，配置`my.cnf`，并重启mysql服务

```
server-id=2 
read_only=TURE 
binlog-ignore-db=mysql
```

### 数据备份及同步

全备master数据库，在`slave1`上操作：

```
$ wget 'http://udbbackup.cn-bj.ufileos.com/udcjw/udb-fup_20190717093059.sql.gz?UCloudPublicKey=ucloududb@ucloud.cn1426152414000604875621&Expires=1563342918&Signature=luF9VxjAhemQQ9I6n6E=' -O udb-fbackup_20190717093059.sql.gz
$ gunzip udb_backup_20190717093059.sql.gz
$ head -n25 udb_backup_20190717093059.sql
--
-- Position to start replication or point-in-time recovery from (the master of this slave)
--

-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000224', MASTER_LOG_POS=187983347;
```

配置主从同步，由于master数据量太大，如果一次性从零开始，会导致数据库`io`瓶颈，因此，

先导入全备的数据,

```
$ mysql -S /tmp/mysql.sock -uroot -ppassword < udb_backup_20190717093059.sql
$ mysql -S /tmp/mysql.sock
> CHANGE MASTER TO
  	MASTER_HOST='master.host',
  	MASTER_USER='replication',
  	MASTER_PASSWORD='bigs3cret',
  	MASTER_PORT=3306,
  	MASTER_LOG_FILE='mysql-bin.000224',
  	MASTER_LOG_POS=187983347,
  	MASTER_CONNECT_RETRY=10;
  
 > start slave;
 > show slave status\G
```

做完主从同步之后，开始全备`slave1`数据库,可以把`slave1`当成`slave2`的`master`了.在`slave2`上操作:

```
$ mysqldump -uuser -hhost -ppassword  -A -E -R --single-transaction --master-data=1 --flush-privileges > full-`date +%Y%m%d_%H%M%S`.sql
$ head -n25 full-20190717_113628.sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000024', MASTER_LOG_POS=129201560;
```

备份完数据后，开始导入数据

```
$ mysql -S /data/mysqlslave/mysql/mysql.sock < full-20190717_113628.sql
$ mysql -S /data/mysqlslave/mysql/mysql.sock
mysql> CHANGE MASTER TO
  		MASTER_HOST='master.host',
  		MASTER_USER='replication',
  		MASTER_PASSWORD='bigs3cret',
  		MASTER_PORT=3306,
  		MASTER_LOG_FILE='mysql-bin.000024',
  		MASTER_LOG_POS=129201560,
  		MASTER_CONNECT_RETRY=10;
mysql> start slave;
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.9.94.184
                  Master_User: bak
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000024
          Read_Master_Log_Pos: 148379396
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 19109915
        Relay_Master_Log_File: mysql-bin.000024
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 148311192
              Relay_Log_Space: 19178286
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 158    --->> 不为0，说明未同步成功。
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 168386232
                  Master_UUID: 57dd3b35-5478-525400f054ca
             Master_Info_File: /data/mysqlslave/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: updating
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
1 row in set (0.00 sec)

mysql> show slave status\G
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.9.94.184
                  Master_User: bak
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000024
          Read_Master_Log_Pos: 148379396
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 19178119
        Relay_Master_Log_File: mysql-bin.000024
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 148379396
              Relay_Log_Space: 19178286
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 168386232
                  Master_UUID: 57dd3b35-5478-525400f054ca
             Master_Info_File: /data/mysqlslave/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0

主要查看是  Seconds_Behind_Master: 0这个数据，当为0的时候说明同步完成。
```

### 进行验证

在master上修改数据，查看`slave1`合`slave2`查询是否同步

```
mysql> use qianxiang_fengkong;
mysql> UPDATE t_fk_control_person SET number = CONCAT('QXHNR-00','871') where id= 871;
```

在`slave1`和`slave2`

```
mysql> SELECT number from qianxiang_fengkong.t_fk_control_person WHERE id =871;
+-------------+
| number      |
+-------------+
| QXHNR-00871 |
+-------------+
1 row in set (0.00 sec)

```

