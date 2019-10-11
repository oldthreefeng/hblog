---
title: 磁盘清理
date: 2018-08-01 09:59:32
tags: [Linux,disk,find]
categories: [server]
---

# 磁盘利用率100%问题解决
```
df -h 查看磁盘占用率100%
分析相关问题
1. df -i查看inode号是否占用，一般情况不会占用。
2. cd / && du -h --max-depth=1 ,查看根目录文件大小 是否是磁盘大小。是的话选中3，否的话选择4.
3. lsof |grep deleted 查看释放空间，发现jenkins占用16G,kill -9 jenkis对应的pid,重启jenkins。
4. find / -size +100M -exec ls -lh {} \; 查询根目录下大于100M文件，并列出来。按不需删除。
这次真实的原因是因为磁盘中比较大并且以有在使用的数据，但是在删除的时候使用的是rm命令直接删除
```

