---
title: logstash初探
date: 2018-12-03 22:50:12
tags: [elk,logstash,log,aliYun]
categories: [server]
---

>LogStash 可以用来对日志进行收集并进行过滤整理后输出到 ES 中，FileBeats 是一个更加轻量级的日志收集工具。
>现在最常用的方式是通过 FileBeats 收集目标日志，然后统一输出到 LogStash 做进一步的过滤，在由 LogStash 输出到 ES 中进行存储。

官方提供了压缩包下载， https://www.elastic.co/downloads/logstash 。 下载完成后解压即可。
下载后,解压缩

```bash
$ tar xf  logstash-6.4.3.tar.gz -C /usr/local
$ mv logstash-6.4.3 logstash

```
logstash必须运行在java环境中,.[下载jdk](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

```bash
$ vi /etc/profile
    export JAVA_HOME=/usr/local/jdk1.8.0_91
    export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    export PATH=$PATH:$JAVA_HOME/bin
```
输入 `java -version`若看到如下信息，则java环境配置成功

```bash
$ java -version
java version "1.8.0_91"
Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.91-b14, mixed mode)
```

将本地的log4j或者nginx日志传输至logstash。
- 本地测试

```bash
$ cat config/simple.conf
input {
  stdin {}
}
output {
  stdout {
    codec => rubydebug }
}
```
- 日志上传至阿里云服务

```bash
# cat config/www.conf
input {
  file {
  path=> [ "/udisk/log4j/borrowWap/*.log",
           "/udisk/log4j/cron/*.log",
           "/udisk/log4j/manage/*.log",
           "/udisk/log4j/rms/*.log",
           "/udisk/log4j/wap/*.log",
           "/udisk/log4j/www/*.log" ]
	}
}

filter {
        date {
                match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
        }
	multiline {
    		pattern => "^[^\[]"
    		what => "previous"
  	}
	if [message] =~ "\[ERR\]" and [message] !~ "404 Page" {
         mutate {
           add_tag => "ERROR"
           remove_tag => "mutiline"
    }
  }

        if [message] =~ "\[WARNING\]" and [message] !~ "AVG"{
         mutate {
           add_tag => "WARNING"
           remove_tag => "mutiline"
     }
       }
}
output {
	logservice {
	endpoint => "http://cn-shanghai.log.aliyuncs.com"
	project => "******"
	logstore => "*****"
	topic => ""
	source => ""
	access_key_id => "********************"
	access_key_secret => "*******************"
	max_send_retry => 10
		}

if "ERROR" in [tags] {
email {
	to => "hongfeng@qianxiangbank.com"
	from => "alilog@qianxiangbank.com"
	address => "smtp.mxhichina.com"
	username => "alilog@qianxiangbank.com"
	password => "**************"
	via => "smtp"
	subject => "ERROR on the qianxiang-pc"
	body => " server_ip:****** \n  message: %{message} ; \n path: %{path}; \n url: https://sls.console.aliyun.com/****"
   }
  }

if "WARNING" in [tags] {
email {
	to => "hongfeng@qianxiangbank.com"
	from => "alilog@qianxiangbank.com"
	address => "smtp.mxhichina.com"
	username => "alilog@qianxiangbank.com"
	password => "**************"
	via => "smtp"
	subject => "ERROR on the qianxiang-pc"
	body => " server_ip:****** \n  message: %{message} ; \n path: %{path}; \n url: https://sls.console.aliyun.com/****"
   }    
  }
}
```
因为java的日志分多行，这里必须得统计在一起，才有意义。刚好logstash提供这个功能，multiline ，首行匹配^[^\[]，即可认为是新行。

gork日志分析正则debug地址(需要翻墙)：[gork debug](http://grokdebug.herokuapp.com/)

例如下面这个日志

```
[INFO]2018-12-03 00:00:00 --------------我是:defaultSource - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 --------------我是:defaultSource - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 --------------我是:slave - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 --------------我是:defaultSource - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 --------------我是:slave - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 --------------我是:slave - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 --------------我是:slave - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 --------------我是:slave - com.qianxiang.aop.mysql.intercept.DataSourceAspect
[INFO]2018-12-03 00:00:00 now_start:2018-12-03 00:00:00;now_end:2018-12-03 17:00:00 - com.qianxiang.web.business.bill.service
.impl.TUserWithdrawalsServiceImpl
[INFO]2018-12-03 00:00:00 分页总数sql:select count(0) from (select
                        r.id,
                        r.invest_id as investId,
                        r.red_package_cnt-r.has_cnt as hasCnt,
                        r.red_package_cnt as redPackageCnt,
                        r.red_package_total as redPackageTotal,
                        CAST((r.has_cnt+1)/2 AS signed) as selfCnt,
                        r.has_cnt as useCnt,
                        r.time,
[INFO]2018-12-03 00:00:00 --------------我是:slave - com.qianxiang.aop.mysql.intercept.DataSourceAspect                        

```

这个时候，多行合并就显得很关键。

```
filter {
	multiline {
    		pattern => "^[^\[]"
    		what => "previous"
  	}
}
```

测试logstash的时候可以用`--configtest` 或者`-t`,来检测配置文件的语法是否有错误，类似`nginx -t`

```bash
$ ./bin/logstash -f  ./config/sample.conf --configtest

#未安装插件会报错
$ ./bin/logstash -f  ./config/sample.conf -t
[2019-04-26T14:28:00,738][ERROR][org.logstash.Logstash    ] java.lang.IllegalStateException: Logstash stopped processing because of an error: (SystemExit) exit

#安装阿里云logstash-output-logservice插件
$ ./bin/logstash-plugin install logstash-output-logservice
Validating logstash-output-logservice
Installing logstash-output-logservice
Installation successful

#安装mutiline插件
$ ./bin/logstash-plugin install logstash-filter-multiline
Validating logstash-filter-multiline
Installing logstash-filter-multiline
Installation successful
```
