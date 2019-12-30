---
title: "记一次linux的bios引导失败"
date: 2019-12-30T15:22:18+08:00
lastmod: 2019-12-30T19:07:18+08:00
tags: [grub,secure,linux]
categories: [linux]
---
> 周末公司整座大楼停电,  所有物理机全部关机然后等待来电启动, 其中一台linux系统启动报错如下:
>
> Reboot and Select proper Boot device
>
> or Insert Boot Media in selected Boot device and press a key.

## 解决思路

百度了一阵, 大概意思是, 重新启动, 选择一个正确的引导设备或者插入媒体引导设备.   第一想法就是引导失效, grub阶段都进入不了.

```
GRUB共分为三个阶段：
 stage1主要负责BIOS和GRUB之间的交接，载入存放于各个分区中的开机文件；
 stage1.5是连接stage1和stage2之间的通道，起着过渡的作用，负责识别stage2所在/boot分区的文件系统，以便进入stage2；
 stage2是grub的核心部分，在这个阶段完成加载内核、加载根文件系统驱动、挂载根等工作。
```

我们这边主要是stage1方式缺失. 需要采用新的光盘引导进入`resuce` 救援模式

```
Resuce system
```

选择完Resuce后, 静待1分钟, 会有一个交互式选择

```
在询问是否同意将系统挂载到/mnt/sysimage，选择continue即可, 默认为1
选择enter即可进入shell.
```

进入恢复, 查看系统安装位置, 目前我们的系统安装在`/dev/sda1`

```bash
sh# chroot /mnt/sysimage
bash-4.1# lsblk    
bash-4.1#  grub-install  /dev/sda1
bash-4.1#  sync

```

查看grub文件是否存在, 发现grub.conf文件也不存在. 将备份的conf复制过来

```bash
bash-4.1#  cat /boot/grub/grub.conf
cat: /boot/grub/grub.conf: No such file or directory
bash-4.1#  cp /etc/grub.conf /boot/grub/grub.conf
```

查看`/boot/efi`文件, 文件也丢失了

```bash
bash-4.1# ls /boot/efi/EFI/centos/
bash-4.1# cp /etc/grub.conf   /boot/efi/EFI/centos/
bash-4.1# reboot
```





