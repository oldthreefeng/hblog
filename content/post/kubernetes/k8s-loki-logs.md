---
title: "k8s基于helm安装loki并持久化数据"
date: 2020-06-19T21:34:18+08:00
tags: [pvc,pv,nfs,nginxIngress,kubernetes,kuboard,loki]
categories: [kubernetes]
---

[TOC]

> 基本的集群监控部署已经完成,  但是整个集群的日志管理目前已经没有着落，基于loki实现了对整个集群的日志管理， 相对于efk， loki更加轻量化。 使用更便捷。

前置要求:  

- 已经部署了NFS或者其他存储的K8s集群.
- 安装好helm

## 什么是Loki

[Loki](https://github.com/grafana/loki)是一个水平可扩展，高可用性，多租户的日志聚合系统，受到Prometheus的启发。它的设计非常经济高效且易于操作，因为它不会为日志内容编制索引，而是为每个日志流编制一组标签。官方介绍说到：Like Prometheus, but for logs.

与其他日志聚合系统相比，`Loki`：

- 不对日志进行全文索引。通过存储压缩的非结构化日志和仅索引元数据，`Loki`操作更简单，运行更便宜。
- 索引和组使用与`Prometheus`已使用的相同标签记录流，使您可以使用与`Prometheus`已使用的相同标签在指标和日志之间无缝切换。
- 特别适合存放`Kubernetes Pod`日志; 诸如`Pod`标签之类的元数据会被自动删除和编入索引。
- 在`Grafana`有本机支持（已经包含在Grafana 6.0或更新版本中）。

- 

Loki由3个组成部分组成：

- loki 是主服务器，负责存储日志和处理查询。
- promtail 是代理，负责收集日志并将其发送给loki。
- 用户界面的Grafana。

## 示意图

![](https://pic.fenghong.tech/k8s/k8s_20200619204216.jpg)

## 部署Loki

基于helm部署loki， 我们先安装一下[helm](https://helm.sh/docs/intro/quickstart/)。

```
$ wget https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz
$ tar xf helm-v3.2.4-linux-amd64.tar.gz && mv linux-amd64/helm /usr/bin
$ helm version
version.BuildInfo{Version:"v3.2.4", GitCommit:"0ad800ef43d3b826f31a5ad8dfbb4fe05d143688", GitTreeState:"clean", GoVersion:"go1.13.12"}
```

安装完成Helm后， 我们基于helm安装loki,  因为我们已经安装了grafana， 所以只需要安装一个loki加上promtail 。loki使用statefulset， promtail 使用daemonset。 

```
$ helm upgrade --install loki --namespace=monitoring loki/loki-stack
Release "loki" does not exist. Installing it now.
NAME: loki
LAST DEPLOYED: Fri Jun 19 21:12:12 2020
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
The Loki stack has been deployed to your cluster. Loki can now be added as a datasource in Grafana.

See http://docs.grafana.org/features/datasources/loki/ for more detail.

$  kubectl get pods -n monitoring | grep loki
loki-0                                 0/1     Running   0          23s
loki-promtail-ksrsn                    1/1     Running   0          23s
loki-promtail-ncs69                    1/1     Running   0          23s
loki-promtail-pz7hn                    1/1     Running   0          23s
loki-promtail-sxpkf                    1/1     Running   0          23s
```

这样就已经部署完毕了。 

## loki数据持久化

基于helm安装的没有数据持久化。 可以采用helm自定义的chart来进行数据持久化， 我这边提供一个简单的思路， 直接更改`statefulset`里面的数据卷挂载。 避免重新学习helmchart，给大家增加负担~~哈哈哈

```
$ kubectl get  statefulset loki -n monitoring  -o yaml  |grep -C 10 volumes
--
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 10001
        runAsGroup: 10001
        runAsNonRoot: true
        runAsUser: 10001
      serviceAccount: loki
      serviceAccountName: loki
      terminationGracePeriodSeconds: 4800
      volumes:
      - name: config
        secret:
          defaultMode: 420
          secretName: loki
      - name: storage
        emptyDir： {}
  updateStrategy:
    type: RollingUpdate
status:

这边看到是emptDir， 没有做storage。
```

首先， 我们创建一个pvc.我这边是基于nfs存储建立的， 如果是ceph或者其他分布式存储， 原理是一样的。

```
$  cat  loki-strorage.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: loki
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
  storageClassName: nfs-23

$ kubectl apply -f loki-strorage.yaml
```

其次 ，将我们上面的`statefulset/loki`保存为yaml文件。

```
$ kubectl get  statefulset loki -n monitoring  -o yaml >> loki-sf.yaml
$ vim loki-sf.yaml
...
      volumes:
      - name: config
        secret:
          defaultMode: 420
          secretName: loki
      - name: storage
          persistentVolumeClaim:  ## 将emtypDir改成pvc， 名称是上面创建的。
          claimName: loki

...

$ kubectl apply -f loki-sf.yaml
```

至此， 大功告成。

至于后面再grafana里面添加数据源就很简单了~~ 

```
http://loki:3100
```

然后查看explor， 选择数据源为Loki，即可访问我们所有的pod日志了。 这里选择`logs`模式， `metrics`模式是不支持的。

```
直接搜索会报错
Metrics mode does not support logs. Use an aggregation or switch to Logs mode.
```

## loki qury example

### 日志选择器

```

对于查询表达式的标签部分，将其用大括号括起来{}，然后使用键值语法选择标签。多个标签表达式用逗号分隔：

= 完全相等。
!= 不相等。
=~ 正则表达式匹配。
!~ 不进行正则表达式匹配。

# 根据任务名称来查找日志
{job="xiaoke/svc-job-admin"}
{job="kube-system/kube-controller-manager"}
{job="nginx-ingress/nginx-ingress"}
{namespace="kube-system",container="kuboard"}

```

### 使用日志过滤器来查找

```
编写日志流选择器后，您可以通过编写搜索表达式来进一步过滤结果

|= 行包含字符串
!= 行不包含字符串。
|~ 行匹配正则表达式。
!~ 行与正则表达式不匹配。

regex表达式接受RE2语法。默认情况下，匹配项区分大小写，并且可以将regex切换为不区分大小写的前缀(?i)。

1. 精确查找名称空间为kube-system下container为kuboard且包含有info关键字的日志
{namespace="kube-system",container="kuboard"} |= "info"

2. 正则查找
{job="huohua/svc-huohua-batch"} |~ "(duration|latency)s*(=|is|of)s*[d.]+"

3. 不包含。
{job="mysql"} |= "error" != "timeout"
```

LQ language可以参考[官方](https://github.com/grafana/loki/blob/master/docs/logql.md)

## 参考

- [安装Kuboard](https://kuboard.cn/install/install-dashboard-offline.html)
- [安装ingress-controller](https://kuboard.cn/install/install-k8s.html)
- [安装nfs存储](https://kuboard.cn/learning/k8s-intermediate/persistent/nfs.html)
- [kuberbetes 基于nfs的pvc](https://fenghong.tech/post/kubernetes/kubeadm-pvc/).

