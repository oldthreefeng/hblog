---
title: "sonarqube 安装部署"
date: 2019-03-08 08:43:12
tags: [java,sonar]
categories: [java]
---
[TOC]

## 说明

- \# 开头的行表示注释
- \> 开头的行表示需要在 mysql 中执行
- $ 开头的行表示需要执行的命令

## 先决条件和概述

运行SonarQube的唯一先决条件是在您的计算机上安装Java（Oracle JRE 8或OpenJDK 8）。

mysql 5.6 or 5.7

linux 平台可以使用以下命令查看值：

```
sysctl vm.max_map_count
sysctl fs.file-max
ulimit -n
ulimit -u
```

可以通过运行以下命令动态设置它们：

```
sysctl -w vm.max_map_count=262144
sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 2048
```

### seccomp

您可以使用以下命令检查内核上是否有seccomp：

```
$ grep SECCOMP /boot/config-$(uname -r)
CONFIG_HAVE_ARCH_SECCOMP_FILTER=y
CONFIG_SECCOMP_FILTER=y
CONFIG_SECCOMP=y
```

如果你的内核没有有seccomp，你会看到：

```
$ grep SECCOMP /boot/config-$(uname -r)
# CONFIG_SECCOMP is not set
```

如果没有，则设置`$SONARQUBE_HOME/conf/sonar.properties`:

```
sonar.search.javaAdditionalOpts=-Dbootstrap.system_call_filter=false
```

### 安装系统配置

```
CPU model            : Intel(R) Xeon(R) CPU E5-26xx v4  @ 2.50GHz
Number of cores      : 4
CPU frequency        : 2500.028 MHz
Total amount of ram  : 7821 MB
Total amount of swap : 0 MB
System uptime        : up 148 days, 16:19
Load average         : 0.34, 0.10, 0.07
OS                   : CentOS 7.5.1804
Arch                 : x86_64 (64 Bit)
Kernel               : 3.10.0-862.14.4.el7.x86_64
Hostname             : dev
IPv4 address         : **********
```

安装预览：

```
mysql  Ver 14.14 Distrib 5.6.42, for Linux (x86_64) using  EditLine wrapper

sonarqube-7.6
sonarqube Location: $SONAR_HOME=/usr/local/sonarqube

Apache Maven 3.5.3 (3383c37e1f9e9b3bc3df5050c29c8aff9f295297; 2018-02-25T03:49:05+08:00)
Maven home: /opt/apache-maven-3.5.3

java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
Default locale: en_US, platform encoding: UTF-8
```

## sonarqube安装

下载.zip压缩包

```
$ wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.6.zip 
$ unzip sonarqube-7.6.zip 
$ mv sonarqube-7.6 /usr/local/sonarqube
$ cd /usr/local/sonarqube
```
创建数据库sonar并授权

```
> CREATE DATABASE sonar CHARACTER SET utf8 COLLATE utf8_general_ci;
> GRANT ALL PRIVILEGES ON *.* TO 'sonar'@'%' IDENTIFIED BY 'sonar';
> GRANT ALL PRIVILEGES ON *.* TO 'sonar'@'localhost' IDENTIFIED BY 'sonar';
> FLUSH PRIVILEGES;
```

添加sonar用户，es的启动必须非root用户，所以要创建一个用户，配置文件主要是数据库和端口。

```
$ useradd sonar
$ echo zNQ0G8GtN9jfLSGz |passwd --stdin sonar
$ chown -R sonar.sonar ../sonarqube/
$ vim conf/sonar.properties 
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false
sonar.sorceEncoding=UTF-8
sonar.web.host=0.0.0.0  
sonar.web.port=9999 
sonar.web.context=/ 
sonar.scm.disabled=true
```
使用以下内容创建文件/etc/init.d/sonar：

```
#!/bin/sh
#
# rc file for SonarQube
#
# chkconfig: 345 96 10
# description: SonarQube system (www.sonarsource.org)
#
### BEGIN INIT INFO
# Provides: sonar
# Required-Start: $network
# Required-Stop: $network
# Default-Start: 3 4 5
# Default-Stop: 0 1 2 6
# Short-Description: SonarQube system (www.sonarsource.org)
# Description: SonarQube system (www.sonarsource.org)
### END INIT INFO
 
su sonar -lc "/usr/local/sonarqube/bin/linux-x86-64/sonar.sh $*"
```

注册服务cents6.5：

```
$ ln -s $SONAR_HOME/bin/linux-x86-64/sonar.sh /usr/bin/sonar
$ chmod +x /etc/init.d/sonar
$ chkconfig --add sonar
$ /etc/init.d/sonar start
```

查看端口是否开启:

```
$ netstat -nlp|grep java
tcp        0      0 0.0.0.0:9999       0.0.0.0:*   LISTEN      27185/java          
tcp        0      0 127.0.0.1:45972    0.0.0.0:*   LISTEN      27264/java          
tcp        0      0 127.0.0.1:32000    0.0.0.0:*   LISTEN      27031/java          
tcp        0      0 127.0.0.1:9001     0.0.0.0:*   LISTEN      27057/java
```

配置iptables:

```
/sbin/iptables -I INPUT -p tcp --dport 9999 -j ACCEPT
```

至此，sonarqube部署完毕。

#### 汉化

登录之后全是英文，可以选择汉化
安装汉化包试试：页面上找到`Administration > Marketplace`，在搜索框中输入`chinese`，出现一个`Chinese Pack`，点击右侧的`install`按钮。
安装成功后，会提示重启 SonarQube 服务器。
稍等一会，再看页面上已经显示中文了。

##  SonarQube Scanner for Maven

### 兼容性

```
maven-sonar-plugin 3.4.0.905之前版本，不再支持SonarQube <5.6。
如果使用5.6之前的SonarQube实例，则应使用maven-sonar-plugin 3.3.0.603。
maven-sonar-plugin 3.1之前版本，不再支持Maven <3.0。
如果在3.0之前使用Maven，则应使用maven-sonar-plugin 3.0.2。
```

### 先决条件

- Maven 3.x
- SonarQube已经 安装好了
- SonarQube服务器有jdk8

### 全局设置

编辑位于`$MAVEN_HOME/conf`中的 [settings.xml文件](http://maven.apache.org/settings.html)，以设置SonarQube服务器。

例：

```
<settings>
    <pluginGroups>
        <pluginGroup>org.sonarsource.scanner.maven</pluginGroup>
    </pluginGroups>
    <profiles>
        <profile>
            <id>sonar</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <!-- Optional URL to server. Default value is http://localhost:9000 -->
                <sonar.host.url>
                  http://myserver:9999
                </sonar.host.url>
            </properties>
        </profile>
     </profiles>
</settings>
```

### 分析Maven项目

分析Maven项目,在pom.xml文件所在的目录中,运行Maven：`mvn sonar:sonar`。

```
mvn clean verify sonar:sonar
  
# In some situation you may want to run sonar:sonar goal as a dedicated step. Be sure to use install as first step for multi-module projects
mvn clean install
mvn sonar:sonar
 
# Specify the version of sonar-maven-plugin instead of using the latest. See also 'How to Fix Version of Maven Plugin' below.
mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.6.0.1398:sonar
```

## sonar-scanner 3.3.0

```
$ wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-3.3.0.1492-linux.zip
$ unzip sonar-scanner-cli-3.3.0.1492-linux.zip
$ mv sonar-scanner-3.3.0.1492-linux/ /usr/local/sonar-scanner
$ vim /etc/profile
export SONAR_SACNNER=/usr/local/sonar-scanner
export PATH=$SONAR_SACNNER/bin:$PATH
$ . /etc/profile
```

配置`sonar-project.properties`

```
$ vim sonar-project.properties
sonar.projectKey=test
sonar.projectName=test
sonar.sources=src/
sonar.language=java
sonar.sourceEncoding=UTF-8
sonar.host.url=http://myserver:9999
sonar.login=***********************
sonar.projectVersion=1.0
sonar.java.binaries=target/classes
```

运行命令即可

```
$ sonar-scanner
usage: sonar-scanner [options]
  
Options:
 -D,--define <arg>     Define property
 -h,--help             Display help information
 -v,--version          Display version information
 -X,--debug            Produce execution debug output
 $ sonar-scanner -v
INFO: Scanner configuration file: /usr/local/sonar-scanner/conf/sonar-scanner.properties
INFO: Project root configuration file: NONE
INFO: SonarQube Scanner 3.3.0.1492
INFO: Java 1.8.0_121 Oracle Corporation (64-bit)
INFO: Linux 3.10.0-862.2.3.el7.x86_64 amd64
```



