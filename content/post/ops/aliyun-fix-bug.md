---
title: "阿里云5月bug修复"
date: "2020-05-29T09:58:18+08:00"
tags: [bug,fix]
categories: [server,ops]
---

[TOC]

>  阿里云五月份的漏洞检测， 爆出了大量的高危漏洞
`$表示shell, #表示注释, > 表示 数据库`

1：RHSA-2019:2197-低危: elfutils security,bug fix,和 enhancement update

```
yum update elfutils-libs  elfutils-libelf  elfutils-default-yama-scope
```

2：RHSA-2019:2079-中危: Xorg 安全和BUG修复更新

```
yum update libX11-common  libX11  libX11-devel -y
```

3：RHSA-2019:2030-中危: python 安全和BUG修复更新

```
yum update python  python-libs -y
```

4：RHSA-2019:3976-低危: tcpdump 安全更新

```
yum update tcpdump -y
```

5：RHSA-2019:4190-高危: nss,nss-softokn,nss-util 安全更新

```
yum update nss-softokn-freebl  nss-softokn  nss-util  nss-tools  nss-sysinit  nss -y
```

6：RHSA-2019:2143-低危: openssh security,bug fix,和 enhancement update

```
yum update openssh-server  openssh  openssh-clients -y
```

7：RHSA-2020:0203-重要: libarchive 安全更新

```
yum update libarchive -y
```

8：RHSA-2019:2169-高危: linux-firmware security,bug fix,和 enhancement update

```
yum update -y  iwl2030-firmware iwl6050-firmware iwl5000-firmware \
iwl4965-firmware iwl3945-firmware iwl135-firmware iwl7260-firmware \
linux-firmware iwl3160-firmware iwl1000-firmware iwl6000-firmware \
iwl105-firmware iwl2000-firmware iwl100-firmware  iwl6000g2b-firmware \
iwl6000g2a-firmware  iwl5150-firmware linux-firmware iwl6050-firmware \
iwl5000-firmware iwl4965-firmware iwl3945-firmware  iwl135-firmware  \
iwl7260-firmware iwl3160-firmware  iwl1000-firmware iwl6000-firmware  \
iwl105-firmware iwl2000-firmware  iwl100-firmware iwl7265-firmware  \
iwl6000g2b-firmware  iwl6000g2a-firmware iwl6000g2a-firmware 
```

9：RHSA-2019:2189-中危: procps-ng 安全和BUG修复更新

```
yum update procps-ng -y
```

10：RHSA-2019:3888-高危: ghostscript 安全更新

```
yum update libgs -y ghostscript-cups  ghostscript 
```

11：RHSA-2019:4326-重要: fribidi 安全更新

```
yum update fribidi -y
```

12：RHSA-2019:2159-低危: unzip 安全更新

```
yum update unzip -y
```

13：RHSA-2019:2118-中危: glibc 安全和BUG修复更新

```
yum update nscd  glibc-common  glibc glibc-headers  glibc-devel -y
```

14：RHSA-2019:2571-高危: pango 安全更新

```
yum update pango -y
```

15：RHSA-2019:2586-高危: ghostscript 安全更新

```
yum update ghostscript -y
```

16：RHSA-2019:2110-中危: rsyslog 安全和BUG修复更新

```
yum update rsyslog -y
```

17：RHSA-2019:2077-低危: ntp security,bug fix,和 enhancement update

```
yum update ntp ntpdate -y
```

18：RHSA-2020:0124-重要: git 安全更新

```
yum update perl-Git git -y
```

19：RHSA-2019:2091-中危: systemd security,bug fix,和 enhancement update

```
yum update systemd-libs systemd-sysv  libgudev1  systemd -y
```

20：RHSA-2019:2462-高危: ghostscript 安全更新

```
yum update sudo -y
```

21：RHSA-2020:0630-重要: ppp 安全更新

```
yum update ppp -y
```

22：RHSA-2019:2964-高危: patch 安全更新

```
yum update patch -y
```


### 参考

- [博客园大佬](https://www.cnblogs.com/willamwang/p/12935256.html)

