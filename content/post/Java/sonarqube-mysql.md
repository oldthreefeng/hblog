---
title: "mysql-5.7.22 编译安装"
date: 2019-03-10 08:43:12
tags: 
  - mysql
  - sonar
categories: [java,mysql]
---

[TOC]

## mysql 5.7的优势

sonarqube建议我们使用mysql5.6或者mysql5.7；我这边选择了mysql5.7，网上查询到如下的优点~

> - 对于多核CPU、固态硬盘、锁有着更好的优化，每秒100W QPS已不再是MySQL的追求，下个版本能否上200W QPS才是用户更关心的。
> - 更好的InnoDB存储引擎
> - 更为健壮的复制功能
> 复制带来了数据完全不丢失的方案，传统金融客户也可以选择使用。MySQL数据库。此外，GTID在线平滑升级也变得可能。
> - 更好的优化器
>  优化器代码重构的意义将在这个版本及以后的版本中带来巨大的改进，Oracle官方正在解决MySQL之前最大的难题。
> - 原生JSON类型的支持
> - 更好的地理信息服务支持
>  InnoDB原生支持地理位置类型，支持GeoJSON，GeoHash特性
> - 新增sys库
>   以后这会是DBA访问最频繁的库MySQL 5.7已经作为数据库可选项添加到《OneinStack》

## 安装依赖

编译依赖

```
$ yum install ncurses-devel libaio-devel gcc -y
```

- cmake编译

由于从 MySQL5.5 版本开始弃用了常规的 configure 编译方法，所以需要 CMake 编译器，用于设置 mysql 的编译参数。如：安装目录、数据存放目录、字符编码、排序规则等。

```
$ cd /usr/local/src
$ wget https://cmake.org/files/v3.8/cmake-3.8.0.tar.gz
$ tar -zxvf cmake-3.8.0.tar.gz
$ cd cmake-3.8.0
$ ./configure
$ gmake -j `grep processor /proc/cpuinfo | wc -l`    && gmake install
```

- bison编译

Linux 下 C/C++语法分析器    

```
$ cd /usr/local/src
$ wget http://ftp.gnu.org/gnu/bison/bison-3.0.4.tar.gz
$ tar -zxvf bison-3.0.4.tar.gz
$ cd bison-3.0.4
$ ./configure
$ make -j `grep processor /proc/cpuinfo | wc -l`    && make install
```

- boost编译

从 MySQL 5.7.5 开始 Boost 库是必需的，mysql 源码中用到了 C++的 Boost 库，要求必须安装 boost1.59.0 或以上版本    

```
$ cd /usr/local/src
$ wget https://nchc.dl.sourceforge.net/project/boost/boost/1.59.0/boost_1_59_0.tar.bz2 --no-check-certificate
$ tar -jxvf boost_1_59_0.tar.bz2
$ mv boost_1_59_0 /usr/local/boost
```

## 编译安装

- mysql源码下载地址[sohu镜像mysql5.7.24](http://mirrors.sohu.com/mysql/MySQL-5.7/mysql-5.7.24.tar.gz)

```
$ wget https://dev.mysql.com/get/archives/mysql-5.7/mysql-5.7.22.tar.gz
$ tar xf  mysql-5.7.22.tar.gz
$ cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql-5.7.22 \
-DMYSQL_DATADIR=/data/mysql57 \
-DMYSQL_UNIX_ADDR=/data/mysql/data/mysql.sock \
-DSYSCONFDIR=/usr/local/mysql-5.7.22/etc \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=all \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DENABLE_DOWNLOADS=1 \
-DWITH_ZLIB=bundled \
-DWITH_READLINE=1 \
-DWITH_EMBEDDED_SERVER=1 \
-DWITH_DEBUG=0 \
-DWITH_BOOST=/usr/local/boost

$ make -j `grep processor /proc/cpuinfo | wc -l`    && make install
```

- 配置文件,仅参考

```
$ cat > /etc/my.cnf << EOF
[mysql]
# CLIENT #
port                           = 3305
socket                         = /data/mysql57/mysql.sock
[mysqld]
# GENERAL #
port                           = 3305
user                           = mysql
default-storage-engine         = InnoDB
socket                         = /data/mysql57/mysql.sock
pid-file                       = /data/mysql57/mysql.pid
explicit_defaults_for_timestamp = 1
# INNODB #
innodb-log-files-in-group      = 2
innodb-log-file-size           = 256M
innodb-flush-log-at-trx-commit = 2
innodb-file-per-table          = 1
innodb-buffer-pool-size        = 2G
# CACHES AND LIMITS #
tmp-table-size                 = 32M
max-heap-table-size            = 32M
max-connections                = 4000
thread-cache-size              = 50
open-files-limit               = 4096
table-open-cache               = 1600
# SAFETY #
max-allowed-packet             = 256M
max-connect-errors             = 1000000
# DATA STORAGE #
datadir                        = /data/mysql57
# LOGGING #
log-error                      = /data/mysql57/mysql-error.log
log-bin                        = /data/mysql57/mysql-bin
max_binlog_size                = 1073741824
binlog-format                  = row
server-id= 1
```

- 创建用户及目录

这边考虑到这台服务器已经运行了mariadb，只能多实例安装mysql-5.7.22，在`/usr/local/mysql/etc/my.cnf`创建了一个新的配置文件，可以参考上面的`/etc/my.cnf`进行配置。

```
$ ln -sv /usr/local/mysql-5.7.22 /usr/local/mysql
$ mkdir /data/mysql57 -p
$ groupadd -r mysql && useradd -r -g mysql -s /sbin/nologin -M mysql 
$ chown -R mysql.mysql /data/mysql57
```
-  启动服务及授权用户

```
# 创建数据库/data/mysql57相关文件
$ /usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql57 

# 启动mysql服务
$ /usr/local/mysql/bin/mysqld_safe --defaults-file=/usr/local/mysql/etc/my.cnf &> /dev/null &

# 设置root密码，可以跳过，进入数据库自己设置
$ /bin/mysql_secure_installation

# 通过sock文件直接进入mysql，授权用户
$ mysql  -S /data/mysql57/mysql.sock
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 192
Server version: 5.7.22-log Source distribution

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
MySQL [(none)]> grant all privileges on *.* to 'louis@%' identified by 'hi.louis@888';
Query OK, 0 rows affected, 1 warning (0.00 sec)
MySQL [(none)]> flush privileges;
Query OK, 0 rows affected (0.01 sec)
```

> 说明"--initialize-insecure"不会生成密码，MySQL之前版本mysql_install_db是在mysql_basedir/script下，MySQL 5.7直接放在了mysql_basedir/bin目录下。