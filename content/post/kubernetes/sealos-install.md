---
title: "sealos 一键安装 kubernetes1.18.0"
date: 2020-06-01T17:34:18+08:00
tags: [docker,nginxIngress,kubernetes,kuboard,prometheus]
categories: [kubernetes]
---

[TOC]

kubeadm是Kubernetes官方提供的用于快速安装Kubernetes集群的工具，伴随Kubernetes每个版本的发布都会同步更新，kubeadm会对集群配置方面的一些实践做调整，通过实验kubeadm可以学习到Kubernetes官方在集群配置上一些新的最佳实践。

## 效果图

dashboard

![](https://pic.fenghong.tech/k8s/k8s_20200601171304.jpg)

grafana

![](https://pic.fenghong.tech/k8s/k8s_20200601171608.jpg)
## 环境准备

### 主机名

设置永久主机名称，然后重新登录:

```
$ hostnamectl set-hostname k8s-master # 将 master 替换为当前主机名
$ cat /etc/redhat-release 
CentOS Linux release 7.7.1908 (Core)
```

- 设置的主机名保存在 `/etc/hostname` 文件中；

如果 DNS 不支持解析主机名称，则需要修改每台机器的 `/etc/hosts` 文件，添加主机名和 IP 的对应关系：

```
cat >> /etc/hosts <<EOF
192.168.59.128 k8s-master
192.168.59.133 k8s-node1
192.168.59.134 k8s-node2
EOF
```

### 一键安装k8s集群

```
$ wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin
$ wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/7b6af025d4884fdd5cd51a674994359c-1.18.0/kube1.18.0.tar.gz

$ sealos init --passwd 123456   \
	--master 192.168.59.128   \
	--node 192.168.59.133 \
	--node 192.168.59.134   \
	--pkg-url /root/kube1.18.0.tar.gz  \
    --version v1.18.0
```

### 安装ingress-controller

```
$ kubectl apply -f https://kuboard.cn/install-script/v1.18.x/nginx-ingress.yaml
```

### 安装k8s集群管理页面

```
$ sealos install --pkg-url https://github.com/sealstore/dashboard/releases/download/v1.0-1/kuboard.tar
```

### 安装k8s-prometheus监控

```
$ git clone https://github.com/coreos/kube-prometheus.git
$ cd kube-prometheus
$ kubectl create -f manifests/setup
$ until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
$ kubectl create -f manifests/
```

这里官方并没有给出ingress的配置文件. 我用的如下文件.这个是用kuboard生成的默认文件. 建议用kuboard添加ingress, 方便快捷.

```
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    k8s.eip.work/workload: grafana
  creationTimestamp: '2020-06-01T08:52:09Z'
  generation: 2
  labels:
    app: grafana
  managedFields:
    - apiVersion: networking.k8s.io/v1beta1
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata': {}
      manager: Mozilla
      operation: Update
      time: '2020-06-01T09:15:12Z'
  name: grafana
  namespace: monitoring
  resourceVersion: '833297'
  selfLink: /apis/networking.k8s.io/v1beta1/namespaces/monitoring/ingresses/grafana
  uid: 63c58ae7-8e4a-4d5e-a4b5-56635e3adb02
spec:
  rules:
    - host: moni.fenghong.tech
      http:
        paths:
          - backend:
              serviceName: grafana
              servicePort: http
            path: /
            pathType: ImplementationSpecific
  tls:
    - hosts:
        - moni.fenghong.tech
      secretName: fenghong.tech

```

### 参考

- [TLS Secret证书管理](https://blog.frognew.com/2018/09/using-helm-manage-tls-secret.html)
- [安装Kuboard](https://kuboard.cn/install/install-dashboard-offline.html)
- [sealos安装k8s集群](https://github.com/fanux/sealos)
- [kube-prometheus](https://github.com/coreos/kube-prometheus)
- [安装ingress-controller](https://kuboard.cn/install/install-k8s.html)

