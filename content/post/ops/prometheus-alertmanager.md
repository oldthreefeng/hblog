---
title: "配置prometheus告警规则"
date: "2020-04-24T11:54:18+08:00"
tags: [prometheus,grafana,alertmanager,node_exporter,nginx]
categories: [server,ops]
---

[TOC]

摘要：

- 系统监控报警
- prometheus
- alertmanager

> 背景：由于公司内网有几台服务器, 想着这几台服务没有什么监控, 故而整个监控系统玩玩. 练练手
>
> 有了监控, 必须得有告警啊, 不然时时刻刻盯着也不是事情. 

## 效果图

prometheus告警状态效果

![node_export](http://pic.fenghong.tech/prometheus/alert_20200410092213.jpg)

异常发送邮件告警

![node_export](http://pic.fenghong.tech/prometheus/mail_20200410092240.jpg)

## 部署

从github下载源码, 或者使用wget下载

```
$ cd /usr/local/
$ wget https://github.com/prometheus/alertmanager/releases/download/v0.20.0/alertmanager-0.20.0.linux-amd64.tar.gz
$ tar xf alertmanager-0.20.0.linux-amd64.tar.gz
$ mv alertmanager-0.20.0.linux-amd64 alertmanager
$ cat > /usr/local/alertmanager/alertmanager.service <<EOF
[Unit]
Description=alertmanager
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/alertmanager/alertmanager --config.file=/usr/local/alertmanager/alertmanager.yml
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
$ ln -s /usr/local/alertmanager/alertmanager.service /lib/systemd/system/alertmanager.service
```

### 配置alertmanager

```
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.mxhichina.com:465' # 邮箱smtp服务器代理
  smtp_from: 'louis.hong@junhsue.com' # 发送邮箱名称
  smtp_auth_username: 'louis.hong@junhsue.com' # 邮箱名称
  smtp_auth_password: '****' # 邮箱密码或授权码
  smtp_require_tls: false
route:
  group_by: ['alertname']
  group_wait: 10s      # 当收到告警的时候，等待三十秒看是否还有告警，如果有就一起发出去
  group_interval: 10s  # 发送警告间隔时间
  repeat_interval: 1h  # 重复报警的间隔时间
  receiver: 'mail'     # 全局报警组，这个参数是必选的，和下面报警组名要相同
receivers:
- name: 'mail'
  email_configs:
  - to: 'louis.hong@junhsue.com'
```

启动服务即可

```
systemctl start alertmanager
systemctl status alertmanager
```

### 配置prometheus规则

```
$ vim /usr/local/prometheus/prometheus.yml

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['localhost:9093']

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
    - "rule.yml"

$ vim /usr/local/prometheus/rule.yml
groups:
    - name: 主机状态-监控告警
      rules:
      - alert: 主机状态
        expr: up == 0
        for: 1m
        labels:
          status: 非常严重
        annotations:
          summary: "{{$labels.instance}}:服务器宕机"
          description: "{{$labels.instance}}:服务器延时超过5分钟"
      
      - alert: CPU使用情况
        expr: 100-(avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by(instance)* 100) > 80
        for: 1m
        labels:
          status: 一般告警
        annotations:
          summary: "{{$labels.mountpoint}} CPU使用率过高！"
          description: "{{$labels.mountpoint }} CPU使用大于80%(目前使用:{{$value}}%)"
  
      - alert: 内存使用
        expr: round(100- node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes*100) > 90
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "内存使用率过高"
          description: "当前使用率{{ $value }}%"

      - alert: IO性能
        expr: 100-(avg(irate(node_disk_io_time_seconds_total[1m])) by(instance)* 100) < 60
        for: 1m
        labels:
          status: 严重告警
        annotations:
          summary: "{{$labels.mountpoint}} 流入磁盘IO使用率过高！"
          description: "{{$labels.mountpoint }} 流入磁盘IO大于60%(目前使用:{{$value}})"
 
      - alert: 网络
        expr: ((sum(rate (node_network_receive_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance)) / 100) > 102400
        for: 1m
        labels:
          status: 严重告警
        annotations:
          summary: "{{$labels.mountpoint}} 流入网络带宽过高！"
          description: "{{$labels.mountpoint }}流入网络带宽持续2分钟高于100M. RX带宽使用率{{$value}}"
      
      - alert: TCP会话
        expr: node_netstat_Tcp_CurrEstab > 1000
        for: 1m
        labels:
          status: 严重告警
        annotations:
          summary: "{{$labels.mountpoint}} TCP_ESTABLISHED过高！"
          description: "{{$labels.mountpoint }} TCP_ESTABLISHED大于1000%(目前使用:{{$value}}%)"
 
      - alert: 磁盘容量
        expr: 100-(node_filesystem_free_bytes{fstype=~"ext4|xfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs"}*100) > 80
        for: 1m
        labels:
          status: 严重告警
        annotations:
          summary: "{{$labels.mountpoint}} 磁盘分区使用率过高！"
          description: "{{$labels.mountpoint }} 磁盘分区使用大于80%(目前使用:{{$value}}%)"
```

重新启动prometheus服务即可

```
$ systemctl restart prometheus
```

访问prometheus, http://serverip:9090/.

![node_export](http://pic.fenghong.tech/prometheus/alert_20200410092213.jpg)

## 验证

将报警的阈值调低即可收到邮件.