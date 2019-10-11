---
title: "ubuntu安装postgresql"
date: 2019-09-07 02:56:11
tags: 
  - sql
  - ubuntu
categories: [server]
---

[TOC]

### 配置OS资源限制

```bash
# vi /etc/security/limits.conf

* soft    nofile  1024000
* hard    nofile  1024000
* soft    nproc   unlimited
* hard    nproc   unlimited
* soft    core    unlimited
* hard    core    unlimited
* soft    memlock unlimited
* hard    memlock unlimited
```
### 关闭 SELinux
关闭 SELinux，否则后续引起不必要的麻烦：
```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 下载源码二进制文件
官方的地址:[https://www.postgresql.org/download/linux/ubuntu/](https://www.postgresql.org/download/linux/ubuntu/)

```yaml
wget http://get.enterprisedb.com/postgresql/postgresql-9.4.24-1-linux-x64-binaries.tar.gz

# 解压到/data目录
tar xf postgresql-9.4.24-1-linux-x64-binaries.tar.gz -C /data/
```

- 配置环境变量

```yaml
export PG_INSTALL_DIR="/data/pgsql"
export PG_DATA_DIR="/data/pgdata"
export PATH=$PG_INSTALL_DIR/bin:$PATH
```

### 配置启动脚本

```bash
#!/bin/bash

set -e
set -u

# chkconfig: 2345 98 02
# description: PostgreSQL RDBMS

# where to find commands like su, echo, etc...
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

DB_ENCODING=SQL_ASCII

DB_LOCALE=C

PG_INSTALL_DIR="/data/pgsql"

PG_DATA_DIR="/data/pgdata"

PG_SERVER_LOG="$PG_DATA_DIR/serverlog"

PG_UNIX_USER=postgres

POSTGRES="$PG_INSTALL_DIR/bin/postgres"

PG_CTL="$PG_INSTALL_DIR/bin/pg_ctl"

INITDB="$PG_INSTALL_DIR/bin/initdb"

# die on first failure; do not keep trucking
set -e

if [ $# -ne 1 ]; then
    echo "please enter start/stop/restart etc..." 1>&2
    exit 1
fi

# Only start if we can find postgres and pg_ctl.
if [ ! -x $PG_CTL ]; then
    echo "$PG_CTL not found" 1>&2
    exit 1
fi

if [ ! -x $POSTGRES ]; then
    echo "$POSTGRES not found" 1>&2
    exit 1
fi

case $1 in
  init)
	su - $PG_UNIX_USER -c "$INITDB --pgdata='$PG_DATA_DIR' --encoding=$DB_ENCODING --locale=$DB_LOCALE"
	;;
  start)
	echo -n "Starting PostgreSQL: "
	su - $PG_UNIX_USER -c "$PG_CTL start -D '$PG_DATA_DIR' -l $PG_SERVER_LOG"
	echo "ok"
	;;
  stop)
	echo -n "Stopping PostgreSQL: "
	su - $PG_UNIX_USER -c "$PG_CTL stop -D '$PG_DATA_DIR' -s -m fast"
	echo "ok"
	;;
  restart)
	echo -n "Restarting PostgreSQL: "
	su - $PG_UNIX_USER -c "$PG_CTL stop -D '$PG_DATA_DIR' -s -m fast -w"
	su - $PG_UNIX_USER -c "$PG_CTL start -D '$PG_DATA_DIR' -l $PG_SERVER_LOG"
	echo "ok"
	;;
  reload)
        echo -n "Reload PostgreSQL: "
        su - $PG_UNIX_USER -c "$PG_CTL reload -D '$PG_DATA_DIR' -s"
        echo "ok"
        ;;
  status)
	su - $PG_UNIX_USER -c "$PG_CTL status -D '$PG_DATA_DIR'"
	;;
  *)
	# Print help
	echo "Usage: $0 {start|stop|restart|reload|status}" 1>&2
	exit 1
	;;
esac

exit 0
```

### 总结

公司的区块链项目用到`postgresql`这个数据来进行存储,用到了,就绪学习他的相关维护和安装,启动等;这次的部署也算是记录一下吧,下次有机会写点用法.

### 参考
- [布比区块链](https://docs.bumo.io/cn/)