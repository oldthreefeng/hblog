---
title: "java overload"
date: 2019-09-11 08:43:12
tags: 
  - java
categories: [java]
---

[TOC]

> 近期java应用，CPU使用率一直很高，经常达到100%，通过以下步骤完美解决，分享一下。

#### 方法一：

[转载](http://www.linuxhot.com/java-cpu-used-high.html)

```
## 1.获取Java进程的PID。  
$ ps axu |grep java   
$ top

## 2. 导出CPU占用高进程的线程栈。
jstack pid >> java.txt 

## 3. 查看对应进程的哪个线程占用CPU过高。
$ top -H -p PID

## 4. 将线程的PID转换为16进制,大写转换为小写。
$ pidhex=`echo "obase=16; PID" | bc | tr "[:upper:]" "[:lower:]"`

## 5. 在第二步导出的java.txt中查找转换成为16进制的线程PID。找到对应的线程栈
$ grep $pidhex -A 30 java.txt

## 分析负载高的线程栈都是什么业务操作。优化程序并处理问题。
```

#### 方法二：

```
## 1.使用top 定位到占用CPU高的进程PID
$ top 
$ ps aux | grep PID命令

## 2.获取线程信息，并找到占用CPU高的线程
$ ps -mp pid -o THREAD,tid,time | sort -rn

##3.将需要的线程ID转换为16进制格式
$ printf "%x\n" tid

## 4.打印线程的堆栈信息
$ jstack pid |grep tid -A 30
```



