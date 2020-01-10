---
title: "服务器巡检并汇总Excel发送邮件通知"
date: 2020-01-10T22:42:18+08:00
lastmod: 2020-01-10T22:42:18+08:00
tags: [Python,shell,ansible]
categories: [Python]
---

## 背景

> 接到一个需求, 要求每天都要服务器巡检, 并统计巡检信息, 归档备查. 公司服务器有13台, 一台台巡查, 手动操作又很繁琐, 身为运维, 决不能让重复性劳动占据太多时间.

## 解决思路

- 写一个监控脚本, 监控服务器的所有的状态, 如服务器磁盘, CPU, 内存, IP, 主机名, load等.
- 将监控的服务器状态转为文本.
- 将文本内容转化为Excel.
- 将Excel发送相关人员邮箱.

想到就要干, 开始写脚本

项目结构

```
.
├── a.sh            #服务器巡检脚本
├── manage.log      #服务器执行脚本日志
├── manage.sh	    #总执行脚本
├── py3			    #python3 虚拟环境
├── __pycache__	    #python3编译生成的cache文件夹
├── server-monitor  #xls文件归档目录
└── transformation.py #python脚本
```



### shell巡检脚本

```
$ cat > a.sh <<-eof
#!/usr/bin/env bash
#获取主机名
system_hostname=$(hostname | awk '{print $1}')

#获取服务器IP
system_ip=$(curl http://100.100.100.200/latest/meta-data/private-ipv4)

#获取总内存
mem_total=$(free -m | grep Mem| awk -F " " '{print $2}')

#获取剩余内存
mem_free=$(free -m | grep "Mem" | awk '{print $4+$6}')

#获取已用内存
mem_use=$(free -m | grep Mem| awk -F " " '{print $3}')

#获取当前平均一分钟负载
load_1=`top -n 1 -b | grep average | awk -F ':' '{print $5}' | sed -e 's/\,//g' | awk -F " " '{print $1}'`

#获取当前平均五分钟负载
load_5=`top -n 1 -b | grep average | awk -F ':' '{print $5}' | sed -e 's/\,//g' | awk -F " " '{print $2}'`

#获取当前平均十五分钟负载
load_15=`top -n 1 -b | grep average | awk -F ':' '{print $5}' | sed -e 's/\,//g' | awk -F " " '{print $3}'`

#取磁盘信息，并加入描述
disk_1=$(df -Ph |grep -v overlay|grep -v grep| awk '{if(+$5>2) print "分区:"$1,"总空间:"$2,"使用空间:"$3,"剩余空间:"$4,"磁盘使用率:"$5}')

#拆分
disk_fq=$(df -Ph|grep -v overlay|grep -v grep  | awk '{if(+$5>2) print "分区:"$1}')
disk_to=$(df -Ph|grep -v overlay|grep -v grep  | awk '{if(+$5>2) print "总空间:"$2}')
disk_us=$(df -Ph|grep -v overlay|grep -v grep  | awk '{if(+$5>2) print "使用空间:"$3}')
disk_fe=$(df -Ph|grep -v overlay|grep -v grep  | awk '{if(+$5>2) print "剩余空间:"$4}')
disk_ul=$(df -Ph|grep -v overlay|grep -v grep  | awk '{if(+$5>2) print "磁盘使用率:"$5}')
disk_ux=$(df -Ph|grep -v overlay|grep -v grep  | awk '{if(+$5>2) print $5}')

#信息存放的文件路径
[ -d "/tmp/sftp/log" ] || mkdir /tmp/sftp/log -p
path=/tmp/sftp/log/monitor_"$system_ip".txt

#内存阈值
mem_mo='60'

echo -e " " > $path
echo -e "主机名:"$system_hostname >> $path
echo -e "服务器IP:"$system_ip >> $path
if [[ $(echo $disk_ux | sed s/%//g) -gt 50 ]]
   then
    echo $disk_fq >>$path
    echo $disk_to >>$path
    echo $disk_us >>$path
    echo $disk_fe >>$path
    echo $disk_ul >>$path
    echo 磁盘巡检状态:不正常 >>$path
   else
    echo $disk_fq >>$path
    echo $disk_to >>$path
    echo $disk_us >>$path
    echo $disk_fe >>$path
    echo $disk_ul >>$path
    echo 磁盘巡检状态:正常 >>$path
 fi
PERCENT=$(printf "%d%%" $(($mem_use*100/$mem_total)))
PERCENT_1=$(echo $PERCENT|sed 's/%//g')
if [[ $PERCENT_1 -gt $mem_mo ]]
    then
     echo -e 总内存大小:$mem_total MB>> $path
     echo -e 已用内存:$mem_use MB >> $path
     echo -e 内存剩余大小:$mem_free MB >> $path
     echo -e 内存使用率:$PERCENT >> $path
     echo -e 内存巡检结果:不正常 >> $path
    else
     echo -e 总内存大小:$mem_total MB>> $path
     echo -e 已用内存:$mem_use MB >> $path
     echo -e 内存剩余大小:$mem_free MB >> $path
     echo -e 内存使用率:$PERCENT >> $path
     echo 内存巡检结果:正常 >> $path
fi
    echo -e 平均1分钟负载:$load_1"\n"平均5分钟负载:$load_5"\n"平均15分钟:$load_15 >> $path
```

执行脚本

```
$ sh a.sh
$ cat /tmp/sftp/log/monitor_172.16.111.118.txt

主机名:iZbp167av7wvow0nqs68rlZ
服务器IP:172.16.111.118
分区:/dev/vda1
总空间:296G
使用空间:78G
剩余空间:204G
磁盘使用率:28%
磁盘巡检状态:正常
总内存大小:16081 MB
已用内存:15806 MB
内存剩余大小:882 MB
内存使用率:98%
内存巡检结果:不正常
平均1分钟负载:1.08
平均5分钟负载:1.08
平均15分钟:1.01

```

### python将文本txt转为Excel

```python
$ cat transformation.py
#!/usr/bin/python
# -*- coding: UTF-8 -*-
import xlwt
import datetime
#import sendmail
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.header import Header
#每天的几点发送邮件及附件加时间戳
now = datetime.datetime.now().strftime('%Y-%m-%d-%H') 
mail_host="smtp.mxhichina.com"  #设置服务器
mail_user="louis@wangke.co"    #用户名
mail_pass="123456"   #口令
style = "font:colour_index red; align: wrap on, vert centre, horiz center;"
styleb = xlwt.XFStyle()  # 创建一个样式对象，初始化样式
al = xlwt.Alignment()
al.horz = 0x02      # 设置水平居中
al.vert = 0x01      # 设置垂直居中
styleb.alignment = al
red_style = xlwt.easyxf(style)
title_style = xlwt.easyxf('font: height 200, name Arial Black, colour_index blue, bold on; align: wrap on, vert centre, horiz center;' )
def getlist():  # 读取txt
    with open('hebing.txt', 'r+',encoding='utf-8') as f:
        s1 = f.readlines()
    f.close()
    s2 = []
    for i in s1:
            if '\n' in i:
                    s2.append(i[:-1])
            else:
                    s2.append(i)
    return s2
def fenge():  # 分割
    list0 = []  # 存贮空格行
    for num, val0 in enumerate(getlist()):
        if val0.split(':')[0] == '主机名':
            list0.append(num)
    list0.append(len(getlist()))
    list1 = []   # 存贮内容
    for num1,val1 in enumerate(list0[1:]):
        temp = getlist()[list0[num1]:list0[num1+1]]
        list1.append(temp)
    return list1
def wxls():   # 写入表格
    title = ['主机名','服务器IP','分区','总空间','使用空间','剩余空间','磁盘使用率','磁盘巡检状态','总内存大小',
    '已用内存','内存剩余大小','内存使用率','内存巡检结果','平均1分钟负载','平均5分钟负载','平均15分钟', 'ceshi']
    workbook = xlwt.Workbook(encoding='utf-8')
    worksheet = workbook.add_sheet('sheet1')
    for i1, val in enumerate(title):
        worksheet.write(0, i1, label=val,style = title_style)
        first_col = worksheet.col(i1)
        first_col.width = 180 * 20
    for i2, val2 in enumerate(title):
        for i3, val3 in enumerate(fenge()):
            for j in val3:
                if j.split(':')[0] == val2:
                    #print i2,i3,j.split(':')[1].decode('utf8')
                    if j.split(':')[1] == '不正常':
                        worksheet.write(i3 + 1, i2, label=j.split(':')[1],style=red_style)
                    else:
                        worksheet.write(i3+1, i2, label=j.split(':')[1] ,style = styleb)
    name = now + 'miontior.xls'
    workbook.save(name)


def send_mail(): 
    sender = 'louis@wangke.co'
    receivers = ['louis@wangke.co']  # 接收邮件，可设置为你的QQ邮箱或者其他邮箱
     
    #创建一个带附件的实例
    message = MIMEMultipart()
    message['From'] = Header("louis@wangke.co", 'utf-8')
    message['To'] =  Header("louis@wangke.co", 'utf-8')
    subject = now+'服务器巡检表格'
    message['Subject'] = Header(subject, 'utf-8')
     
    #邮件正文内容
    message.attach(MIMEText('各位好: \n' + now + ': 生产环境各服务器状态见附件.', 'plain', 'utf-8'))
     
    # 构造附件1，传送当前目录下的 test.txt 文件
    att1 = MIMEText(open(now + 'miontior.xls', 'rb').read(), 'base64', 'utf-8')
    att1["Content-Type"] = 'application/octet-stream'
    # 这里的filename可以任意写，写什么名字，邮件中显示什么名字
    att1["Content-Disposition"] = 'attachment; filename="{}moniter.xls"'.format(now+'-')
    message.attach(att1)
     
    try:
        smtpObj = smtplib.SMTP_SSL(mail_host, 465)
        smtpObj.login(mail_user,mail_pass)
        smtpObj.sendmail(sender, receivers, message.as_string())
        print ("邮件发送成功")
    except smtplib.SMTPException:
        print ("Error: 无法发送邮件")

wxls()
send_mail()
```

### 批量操作

上面的都是单机操作, 单机操作其实没必要写脚本了, 手动也花不了多少时间. 万一服务器有100台呢? 每台执行相同的命令, 依旧没有解决问题. 这里, 可以将监控的文本txt全部fecth到某台服务器进行集中管理, 即可解决痛点. 使用的是anisible进行管理的主机

```bash
$ cat manage.sh
#!/bin/bash
DATE=$(date +%F-%H:%M:%S)
source py3/bin/activate
echo "$DATE exec $0" >> manage.log 

## 所有主机执行a.sh, 即完成服务器巡检操作
ansible all -m script -a "/data/louis/a.sh"  >> manage.log

## 将所有主机的生成的txt文件同步到ansible管理的主机上
ansible all -m synchronize -a "src=/tmp/sftp/log/ dest=/data/louis mode=pull"  >> manage.log

## 生成主机合并的信息文件
> hebing.txt
## 遍历同步过来的主机巡检信息文件, 追加至文件
for file in `ls ./mo*.txt` ; do
	cat $file >> hebing.txt
done

## 将合并的信息转为xls文件

python3.4 transformation.py >> manage.log

## 将不需要的文件删掉并归档.
rm -f *.txt
mv *.xls server-monitor
```

关于ansible的细节操作就不在重复了, 具体细节点击标签, 查看ansible相关文档.

over~