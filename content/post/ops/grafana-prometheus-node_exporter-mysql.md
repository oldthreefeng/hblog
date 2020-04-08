---
title: "搭建prometheus监控系统"
date: "2020-04-08T13:54:18+08:00"
tags: [prometheus,grafana,mysql,node_exporter,nginx]
categories: [server,ops]
---

[TOC]

摘要：

- 系统监控
- prometheus

> 背景：由于公司内网有几台服务器, 想着这几台服务没有什么监控, 故而整个监控系统玩玩. 练练手 

## 效果图

服务器基本状态效果

![node_export](http://pic.fenghong.tech/prometheus/node_20200408170900.jpg)

mysql监控状态

![node_export](http://pic.fenghong.tech/prometheus/mysql-20200408165104.jpg)

## 前置知识

  在编写应用程序的时候，通常会记录 Log 以便事后分析，在很多情况下是产生了问题之后，再去查看 Log ，是一种事后的静态分析。在很多时候，我们可能需要了解整个系统在当前，或者某一时刻运行的情况，比如当前系统中对外提供了多少次服务，这些服务的响应时间是多少，随时间变化的情况是什么样的，系统出错的频率是多少。这些动态的准实时信息对于监控整个系统的运行健康状况来说很重要。于是就产生了 metrics 这种数据. 详情查看[https://monitor.lucien.ink/metrics](https://monitor.lucien.ink/metrics)


### prometheus简介

> Prometheus 是一套开源的系统监控、报警、时间序列数据库的组合，最初有 SoundCloud 开发的，后来随着越来越多公司使用，于是便独立成开源项目。
> 我们常用的 Kubernetes 容器集群管理中，通常会搭配 Prometheus 一起来进行监控。Prometheus 基本原理是通过 Http 协议周期性抓取被监控组件的状态，而输出这些被监控的组件的 Http 接口为 Exporter.
> 现在各个公司常用的 Exporter 都已经提供了，可以直接安装使用，如 haproxy_exporter、blockbox_exporter、mysqld_exporter、node_exporter 等等，更多支持的组件

### Grafana 介绍

> Grafana 是一个可视化仪表盘，它拥有美观的图标和布局展示，功能齐全的仪表盘和图形编辑器，默认支持 CloudWatch、Graphite、Elasticsearch、InfluxDB、Mysql、PostgreSQL、Prometheus、OpenTSDB 等作为数据源。
>
> 我们可以将 Prometheus 抓取的数据，通过 Grafana 优美的展示出来，非常直观。

### Grafana、Prometheus、Exporter之间的关系

>  Exporter 的主要任务是提供 metrics 信息。
>
> metrics 大多数人是看不懂的，所以 Prometheus 为这种格式的信息提供了 Prometheus Query Language (PromQL) ，可以进行一些类似数据库那样的联合查询、过滤等操作，这样一来就能提炼出我们想要的东西，类似于内存占用、负载等.
>
> 虽然 PromQL 非常的强大，但是对于大部分人来说是有很高的学习成本的，所以 Grafana 就将各种 PromQL 封装起来，并将 PromQL 的结果以图表的形式展示出来。
>
> 当然了，Prometheus 和 Grafana 的功能远不止如此，更强大的是报警功能，但这不是本文的主题。

## 部署

本文采用的安装方式皆为二进制 + systemd 托管的安装方式。 更多的有docker部署， k8s部署. 详情可以查阅官网.

### 下载

node_exporter： https://github.com/prometheus/node_exporter/releases
Prometheus：https://github.com/prometheus/prometheus/releases
Grafana（选择 Standalone Linux Binaries 版本）：https://grafana.com/grafana/download

### 安装

```
$ mkdir /usr/local/monitor

$ wget https://github.com/prometheus/prometheus/releases/download/v2.17.1/prometheus-2.17.1.linux-amd64.tar.gz
$ wget https://dl.grafana.com/oss/release/grafana-6.7.2.linux-amd64.tar.gz
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.0.0-rc.0/node_exporter-1.0.0-rc.0.linux-amd64.tar.gz

$ vim install.sh
for FILE in `ls *.gz`; do tar -xzvf ${FILE} && rm ${FILE}; done
FILE_LIST="grafana prometheus node_exporter "
for FILE in ${FILE_LIST}; do rm -rf /usr/local/${FILE} && mv ${FILE}* /usr/local/${FILE}; done
rm -f /lib/systemd/system/grafana-server.service
rm -f /lib/systemd/system/prometheus.service
rm -f /lib/systemd/system/node_exporter.service
cat>/usr/local/grafana/grafana-server.service<<EOF
[Unit]
Description=Grafana Server
After=network.target
 
[Service]
Type=simple
User=root
WorkingDirectory=/usr/local/grafana
ExecStart=/usr/local/grafana/bin/grafana-server
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
cat>/usr/local/prometheus/prometheus.service<<EOF
[Unit]
Description=Prometheus
After=network.target
 
[Service]
Type=simple
User=root
WorkingDirectory=/usr/local/prometheus
ExecStart=/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
cat>/usr/local/node_exporter/node_exporter.service<<EOF
[Unit]
Description=Node Exporter
After=network.target
Wants=network-online.target
 
[Service]
Type=simple
User=root
ExecStart=/usr/local/node_exporter/node_exporter
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
ln -s /usr/local/grafana/grafana-server.service /usr/local/prometheus/prometheus.service /usr/local/node_exporter/node_exporter.service /lib/systemd/system/
systemctl daemon-reload
systemctl start node_exporter
systemctl start grafana-server
systemctl start prometheus

$ sh install.sh
```

### 验证是否安装成功

```
# 查看运行状态
$ systemctl status node_exporter
$ systemctl status prometheus
$ systemctl status grafana-server

$ # 查看metrics
$ curl localhost:9100/metrics
$ curl localhost:9090/metrics
$ curl localhost:3000/metrics
```

### 配置服务

```
$ sed -i 's/localhost:9090/localhost:9100/'  /usr/local/prometheus/prometheus.yml
## 这个修改会让 Prometheus 从 localhost:9100/metrics 进行 metrics 信息的读取，
# 默认的 9090 是 Prometheus 本身的 metrics 信息。
# 9100 是 node_exporter的metrics 信息读取端口

$ systemctl restart prometheus
```

### 配置grafana插件

```
# 安装一个 饼图 的插件
$ cd /usr/local/grafana/bin
$ chmod +x grafana-cli
$ ./grafana-cli plugins install grafana-piechart-panel
$ systemctl restart grafana-server
$ cp -r /var/lib/grafana/plugins/grafana-piechart-panel /usr/local/grafana/data/plugins
```

### 登录进行数据接入

访问 `http://localhost:3000`, 账号密码默认为: `admin/admin`.

成功访问后, 点击 `Add data source`。选择 `Prometheus`。

Http URL 中填入 `http://localhost:9090` ，也就是 `prometheus` 提供的接口。然后保存即可.

### 引入Exporter写好的dashboard

然后把鼠标挪到`左上角的 +` 上，注意是挪上去，然后在弹出的菜单中点击 `Import`。

选择id为`8919`的一个dashboard. 国人自己写的. https://grafana.com/dashboards/8919

点击空白处之后会自动导入对应的 `Dashboard` ，此时会让你设置数据来源，在 `Options prometheus_111` 这里选择我们刚才添加的 `Prometheus` ，然后点击 Import 就可以了.

### 监控多个节点

在完成了本文的 、 部分之后，仅仅是完成了监控本机的过程，如果要监控其它的节点，需在被监控的节点上安装相应的 Exporter，下面以本文中提到的 `node_exporter` 为例，介绍如何添加节点。

```
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.0.0-rc.0/node_exporter-1.0.0-rc.0.linux-amd64.tar.gz
$ cat>/usr/local/node_exporter/node_exporter.service<<EOF
[Unit]
Description=Node Exporter
After=network.target
Wants=network-online.target
 
[Service]
Type=simple
User=root
ExecStart=/usr/local/node_exporter/node_exporter
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF

$ ln -s /usr/local/node_exporter/node_exporter.service /lib/systemd/system/
$ systemctl start node_exporter
## 如果9100端口被占用了 . 可以使用下面的命令更改监听端口. 
$ cd /usr/local/node_exporter/
$ nohup ./node_exporter --web.listen-address=":9101" &
```

### 配置prometheus

```
## 在监控节点上编辑 Prometheus 的配置文件 /usr/local/prometheus/prometheus.yml。
$ vim /usr/local/prometheus/prometheus.yml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9100', '$addr:9100'] ############### 我们需要修改这里
```

将`$addr:9100`换为自己的真实ip及端口. 保存后重启服务即可.

 `targets` 传入的是一个数组，Prometheus 会收集数组中的每个元素的 metrics ，然后 Grafana 再处理这些数据。

```
$ systemctl restart prometheus
```

## 监控mysqld

下载mysqld_exporter

```
$ cd /usr/local
$ wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.12.1/mysqld_exporter-0.12.1.linux-amd64.tar.gz
```

部署mysqld_exporter

```
$ cd /usr/local/
$ tar xf mysqld_exporter-0.12.1.linux-amd64.tar.gz 
$ cd mysqld_exporter-0.12.1.linux-amd64
## 这里创建mysql用户和授权
mysql> grant replication client, process on *.* to prometheus@"%" identified by "weakpass";
mysql> grant select on performance_schema.* to prometheus@"%";
mysql> flush previleges;

$ vim .my.cnf
[client]
host=192.168.0.21
port=3306
user=prometheus
password=weakpass

## 默认监听是9104端口
$ nohup ./mysqld_exporter --config.my-cnf=./.my.cnf &
```

### 验证

```
$ curl localhost:9104/metrics
```

配置Prometheus接入mysql的metrics数据。

```
## 严格注意缩进
$ vim /sur/local/prometheus/prometheus.yml  
  - job_name: 'mysql'
    static_configs:
    - targets: ['192.168.0.21:9104']
```

### 登录进行数据接入

访问 `http://localhost:3000`, 账号密码默认为: `admin/admin`.

成功访问后, 点击 `Add data source`。选择 `Prometheus`。

Http URL 中填入 `http://localhost:9090` ，也就是 `prometheus` 提供的接口。然后保存即可.

### 引入Exporter写好的dashboard

然后把鼠标挪到`左上角的 +` 上，注意是挪上去，然后在弹出的菜单中点击 `Import`。选择id为`7362`.  

![node_export](http://pic.fenghong.tech/prometheus/dashboard_20200408174127.jpg)

选择Folder为general。 Prometheus选择Prometheus， 再导入即可。

![node_export](http://pic.fenghong.tech/prometheus/import_20200408174759.jpg)

至此， 基本结束。

