---
title: "阿里云6月bug修复"
date: "2020-06-16T16:58:18+08:00"
tags: [bug,fix]
categories: [server,ops]
---

[TOC]

>  阿里云六月份的漏洞检测， 爆出了大量的高危漏洞
>
>  `$表示shell, #表示注释, > 表示 数据库`

1：RHSA-2020:1176-低危: avahi 安全更新

```
yum update avahi-libs -y
```

2：RHSA-2020:1022-低危: file 安全更新

```
yum update file-libs -y
```

3：RHSA-2020:1138-低危: gettext 安全和BUG修复更新

```
yum update gettext gettext-common-devel gettext-devel -y
```

4：RHSA-2020:1020-低危: curl 安全和BUG修复更新

```
yum update libcurl-devel -y
```

5：RHSA-2020:1135-低危: polkit 安全和BUG修复更新

```
yum update polkit  -y
```

6：RHSA-2020:1011-中危: expat 安全更新

```
yum update  expat -y
```

7：RHSA-2020:1113-中危: bash 安全更新

```
yum update bash -y
```

8：RHSA-2020:1190-中危: libxml2 安全更新

```
yum update -y  libxml2
```

9：RHSA-2020:1050-中危: cups 安全和BUG修复更新

```
yum update cups-libs cups-client -y
```

10：RHSA-2020:1061-中危: bind 安全和BUG修复更新

```
yum update bind-export-libs  bind-libs-lite bind-license -y 
```

11：RHSA-2020:0897-重要: icu 安全更新

```
yum update libicu-devel libicu -y
```

12：RHSA-2020:1334-重要: telnet 安全更新

```
yum update telnet -y
```

13：RHSA-2020:0850-中危: python-pip 安全更新

```
yum update python3-pip python3  -y
```



