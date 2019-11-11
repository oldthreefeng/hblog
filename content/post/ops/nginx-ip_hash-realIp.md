---
title: "nginx基于ip_hash访问策略的不生效的分析."
date: 2019-11-11T15:52:18+08:00
tags: [nginx,hash,upstream]
categories: [server,nginx]
---

nginx基于ip_hash访问策略的不生效的分析.

> 一大早上, 开发过来和我说, nginx的负载均衡是不是不生效了, 后端的日志大小不一样, 相差甚大. 想了想, 访问量在非常大的情况下, 这些日志里的大小应该相对平均. 因此上服务器具体查看相关信息.
>

## 查日志,定位

查看相关的日志量, 来进行定位.

```bash
$ ansible yj-new -m shell -a "du -sh /usr/local/e-mall/logs"
172.16.111.140 | CHANGED | rc=0 >>
486M	/usr/local/e-mall/logs

172.16.111.142 | CHANGED | rc=0 >>
189M	/usr/local/e-mall/logs

172.16.111.143 | CHANGED | rc=0 >>
1.6M	/usr/local/e-mall/logs

172.16.111.141 | CHANGED | rc=0 >>
562M	/usr/local/e-mall/logs

```

查看配置文件, 发现nginx的负载均衡策略为`ip_hash`

nginx中配置的是ip-hash算法来负载,`remote_addr`为具体的某个ip时, 负载至后端是固定的。初步断定是由于*`remote_addr`为某些固定ip原因造成* .

分析了两台web的nginx访问ip次数. 亮瞎了我的眼, 日志总共260W＋,访问的ip却只有600+. 那的确可能是`remote_addr`存在相关的问题. 日活大概在500W的PV. UV绝对不止600+; 因为服务器前端有CDN/SLB, 获取的`remote_addr`部分是`client_real_ip`, 不具有代表性.

```
$ wc -l /usr/local/nginx/log/emall.access.log
2661917 /usr/local/nginx/log/emall.access.log

$ awk '{print $1}' emall.access.log |sort -n |uniq -c| wc -l
659

$ awk '{print $1}' emall.access.log |sort -n |uniq -c| wc -l
664
```
查询具体的ip及次数并重定向只文件中, 分析ip来源.

```
$ awk '{print $1}' emall.access.log |sort -n|uniq -c|sort -nr|head -100 
  47754 120.27.74.189
  43620 120.27.74.159
  38094 120.27.74.157
  37226 120.27.74.180
  31201 120.27.74.174
  29928 222.186.49.162
  29491 14.17.67.49
  29257 222.186.49.191
  28835 14.17.67.28
  28188 222.186.49.150
  27456 120.27.74.154
  26047 222.186.49.194
  25379 120.27.74.181
  25081 116.211.216.219
  23177 116.211.216.193
  21665 222.186.49.160
  21449 120.27.74.169
  20319 116.211.216.202
  ....
```

对结果的ip进行分析, 使用curl工具, 对ip进行分析, 使用的是[lionsoul/ip2region]( https://gitee.com/lionsoul/ip2region.git)来查询的ip信息.基本都是阿里云的ISP,,电信机房, 移动机房.

```bash
$ awk "{print $ 2}" ip.txt  | while read ip ; do curl -H "DEVOPS-API-TOKEN: ${louis_token}" https://api.wangke.co/api/v1/queryip?ip=$ip >> ip.md ;echo "\n" >> ip.md ; done

$ cat ip.md
{"data":{"ip":"120.27.74.189","ipInfo":{"CityId":0,"Country":"中国","Region":"0","Province":"山东","City":"青岛","ISP":"阿里云"}},"entryType":"Query IP","errmsg":"","requestId":"b0ff1d40-a487-4613-ac82-e9922d8dd8f8","statuscode":0}\n
{"data":{"ip":"120.27.74.159","ipInfo":{"CityId":0,"Country":"中国","Region":"0","Province":"山东","City":"青岛","ISP":"阿里云"}},"entryType":"Query IP","errmsg":"","requestId":"f3ba9b71-ee85-4a6e-aa50-9fd689e49d0b","statuscode":0}\n
{"data":{"ip":"120.27.74.180","ipInfo":{"CityId":0,"Country":"中国","Region":"0","Province":"山东","City":"青岛","ISP":"阿里云"}},"entryType":"Query IP","errmsg":"","requestId":"d9901ab2-863c-4d23-bb94-89221859e15c","statuscode":0}\n
{"data":{"ip":"120.27.74.157","ipInfo":{"CityId":0,"Country":"中国","Region":"0","Province":"山东","City":"青岛","ISP":"阿里云"}},"entryType":"Query IP","errmsg":"","requestId":"471f45ba-0792-4462-9a61-4e818c64ab6d","statuscode":0}\n
{"data":{"ip":"14.17.67.49","ipInfo":{"CityId":0,"Country":"中国","Region":"0","Province":"广东","City":"东莞","ISP":"电信"}},"entryType":"Query IP","errmsg":"","requestId":"2c62a280-4ff0-428e-b058-e1085940d1a4","statuscode":0}\n
{"data":{"ip":"222.186.49.191","ipInfo":{"CityId":1111,"Country":"中国","Region":"0","Province":"江苏省","City":"镇江市","ISP":"电信"}},"entryType":"Query IP","errmsg":"","requestId":"7c802b1c-5526-49e7-b4b5-2800fd422857","statuscode":0}\n
{"data":{"ip":"14.17.67.28","ipInfo":{"CityId":0,"Country":"中国","Region":"0","Province":"广东","City":"东莞","ISP":"电信"}},"entryType":"Query IP","errmsg":"","requestId":"019ce261-621d-4609-a744-e624aeb1293a","statuscode":0}\n
....
```
## 解决

和开发沟通, 基于session不丢失问题, 采用ip_hash是相对稳定的方法, 但是session目前是存在redis里面, 因此可以将nginx的分发策略改为rr或者wrr.

对应nginx不是最前端时，如果前端做了CDN/SLB等二次代理的，造成`remote_addr`为固定ip时可以采用下列方式

```nginx
## 写在http模块中, 获取代理的

map $http_x_forwarded_for  $clientRealIp {
       ""      $remote_addr;
       ~^(?P<firstAddr>[0-9\.]+),?.*$  $firstAddr;

}

## upstream模块中的ip_hash改成 hash $clientRealIp

upstream mall {
		hash $clientRealIp;
		server   172.16.111.121:8888 max_fails=2 fail_timeout=30s;
		server   172.16.111.132:8888 max_fails=2 fail_timeout=30s;
		server   172.16.111.140:8888 max_fails=2 fail_timeout=30s;
		server   172.16.111.141:8888 max_fails=2 fail_timeout=30s;
		server   172.16.111.142:8888 max_fails=2 fail_timeout=30s;
		server   172.16.111.143:8888 max_fails=2 fail_timeout=30s;
}
```

即可. 