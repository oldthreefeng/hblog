---
title: "k8s安装prometheus并持久化数据"
date: 2020-06-11T15:34:18+08:00
tags: [pvc,pv,nfs,nginxIngress,kubernetes,kuboard,prometheus]
categories: [kubernetes]
---

[TOC]

> Prometheus 部署已经完成, 但是由于官方的coreos中没有持久化数据, 没有部署ingress, pod重启后数据就会消失. 所以持久化数据就显得比较重要. 

- 前置要求:  已经部署了NFS或者其他存储的K8s集群.

## PV示意图

![](https://oss.fenghong.tech/k8s/pvc_20200611162616.jpg)

## 部署kube-prometheus

```
$ git clone https://github.com/coreos/kube-prometheus.git
$ kubectl create -f manifests/setup
$ until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
$ kubectl create -f manifests/
```

持久化数据我这里用的是NFS创建动态的`pv`, 具体看我以前写一篇[kuberbetes 基于nfs的pvc](https://fenghong.tech/post/kubernetes/kubeadm-pvc/).  也可以用cephfs. 这里不展开了.  确保已经创建了一个sc.

```
$ kubectl get sc
NAME                                 PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/nfs-23   nfs-nfs-23    Delete          WaitForFirstConsumer   false                  5d2h
```

## kube-prometheus的组件简介及配置变更

从整体架构看，prometheus 一共四大组件。 exporter 通过接口暴露监控数据， prometheus-server 采集并存储数据， grafana 通过prometheus-server查询并友好展示数据， alertmanager 处理告警，对外发送。

### prometheus-operator

prometheus-operator 服务是deployment方式部署，他是整个基础组件的核心，他监控我们自定义的 prometheus 和alertmanager，并生成对应的 statefulset。 就是prometheus和alertmanager服务是通过他部署出来的。

### grafana-pvc

创建grafana的存储卷. 并修改`grafana-deployment.yaml`文件, 将官方的`emptyDir`更换为`persistentVolumeClaim`

```
$ cd kube-prometheus/manifests/
$ cat > grafana-pvc.yaml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    k8s.kuboard.cn/pvcType: Dynamic
    pv.kubernetes.io/bind-completed: 'yes'
    pv.kubernetes.io/bound-by-controller: 'yes'
  name: grafana
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs-23
status:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 100Gi
EOF

$ kubectl apply -f grafana-pvc.yaml
$ vim grafana-deployment.yaml

...
	  ##找到 grafana-storage, 添加上面创建的pvc: grafana. 然后保存.
      volumes:
      - name: grafana-storage
          persistentVolumeClaim:
            claimName: grafana
...

$ kubectl apply -f grafana-deployment.yaml
```

### prometheus-k8s持久化

prometheus-server 获取各端点数据并存储与本地，创建方式为自定义资源 crd中的prometheus。 创建自定义资源prometheus后，会启动一个statefulset，即prometheus-server.  默认是没有配置持久化存储的

这里更改配置文件.

```
$ cd kube-prometheus/manifests/
$ vim prometheus-prometheus.yaml  
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - name: alertmanager-main
      namespace: monitoring
      port: web
      
  storage: #这部分为持久化配置
    volumeClaimTemplate:
      spec:
        storageClassName: nfs-23 
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
            
  image: quay.io/prometheus/prometheus:v2.17.2
  nodeSelector:
    kubernetes.io/os: linux
  podMonitorNamespaceSelector: {}
  podMonitorSelector: {}
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: v2.17.2
```

执行变更, 这里会自动创建两个指定大小的pv（`prometheus-k8s-0`，`prometheus-k8s-1`）

```
$ kubectl apply -f manifests/prometheus-prometheus.yaml 
```

修改存储时长. 

```
$ vim manifests/setup/prometheus-operator-deployment.yaml
....
      - args:
        - --kubelet-service=kube-system/kubelet
        - --logtostderr=true
        - --config-reloader-image=jimmidyson/configmap-reload:v0.3.0
        - --prometheus-config-reloader=quay.io/coreos/prometheus-config-reloader:v0.39.0
        - storage.tsdb.retention.time=180d   ## 修改存储时长
....
$ kubectl apply -f manifests/setup/prometheus-operator-deployment.yaml
```

### 添加ingress访问grafana和promethues

```
$ cat > ingress.yml <<EOF
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    k8s.eip.work/workload: grafana
    k8s.kuboard.cn/workload: grafana
  generation: 2
  labels:
    app: grafana
  name: grafana
  namespace: monitoring
spec:
  rules:
    - host: k8s-moni.fenghong.tech
      http:
        paths:
          - backend:
              serviceName: grafana
              servicePort: http
            path: /
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    k8s.kuboard.cn/workload: prometheus-k8s
  generation: 2
  labels:
    app: prometheus
    prometheus: k8s
  managedFields:
    - apiVersion: networking.k8s.io/v1beta1
  name: prometheus-k8s
  namespace: monitoring
spec:
  rules:
    - host: k8s-prom.fenghong.tech
      http:
        paths:
          - backend:
              serviceName: prometheus-k8s
              servicePort: web
            path: /
```

执行apply

````
## 安装 ingress controller
$ kubectl apply -f https://kuboard.cn/install-script/v1.18.x/nginx-ingress.yaml

## 暴露grafana及prometheus服务
$ kubectl apply -f ingress.yml
````

**配置域名解析**

将域名 `k8s-prom.fenghong.tech` 解析到任意 work节点 的 IP 地址如: `192.168.0.30` 

**验证配置**

在浏览器访问 `k8s-prom.fenghong.tech`，将得到prometheus的页面 

在浏览器访问 `k8s-moni.fenghong.tech`，将得到grafana的访问页面 

### 参考

- [安装Kuboard](https://kuboard.cn/install/install-dashboard-offline.html)
- [kube-prometheus](https://github.com/coreos/kube-prometheus)
- [安装ingress-controller](https://kuboard.cn/install/install-k8s.html)

