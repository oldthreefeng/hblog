---
title: "kubernetes集群升级至v1.18.3且调整证书年限"
date: 2020-06-16T20:34:18+08:00
tags: [kubernetes,upgrade,v1.18.3]
categories: [kubernetes]
---

[TOC]

> 安装`kubenetes1.18.0`集群后, 发现最版本是`v1.18.3`, 所以有了这个升级, `.0`版本的问题有点多. 建议上`statble`版本, 不论是生产还是测试环境.  

## 更新源
```
$ cat > /etc/yum.repos.d/kubernetes.repo < kubeadm-config-upgrade.yaml
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 升级kubelet和kubeadm版本

重要: **master和node节点都得升级**
```
### 查看可用源
$ yum list --showduplicates kubeadm --disableexcludes=kubernetes
### 安装v1.18.3版本
$ yum install  kubelet-1.18.3 kubeadm-1.18.3 kubectl-1.18.3 -y
### 重启服务
$ systemctl daemon-reload &&  systemctl restart kubelet
```

### 查看集群配置
```
$ kubeadm config view > kubeadm-config-upgrade.yaml
```

修改配置文件, 主要是两个字段`kubernetesVersion: v1.18.3`和`imageRepository: registry.aliyuncs.com/k8sxio`, 这里用的是[张馆长](https://zhangguanzhang.github.io/)的源.

### 升级至最新版本

```
$ vim kubeadm-config-upgrade.yaml
apiServer:
  certSANs:
  - 127.0.0.1
  - apiserver.cluster.local
  - 192.168.0.31
  - 10.103.97.2
  extraArgs:
    authorization-mode: Node,RBAC
    feature-gates: TTLAfterFinished=true
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    pathType: File
    readOnly: true
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: apiserver.cluster.local:6443
controllerManager:
  extraArgs:
    experimental-cluster-signing-duration: 876000h
    feature-gates: TTLAfterFinished=true
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    pathType: File
    readOnly: true
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/k8sxio
kind: ClusterConfiguration
kubernetesVersion: v1.18.3
networking:
  dnsDomain: cluster.local
  podSubnet: 100.64.0.0/10
  serviceSubnet: 10.96.0.0/12
scheduler:
  extraArgs:
    feature-gates: TTLAfterFinished=true
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    pathType: File
    readOnly: true

```

### 执行升级

```
## 先干跑, 测试一下
$ kubeadm  upgrade  apply  -f --config  kubeadm-config-upgrade.yaml  --dry-run
```

### 升级至v1.18.3

```
$ kubeadm  upgrade  apply  -f --config  kubeadm-config-upgrade.yaml 

W0616 11:03:31.275157   23794 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[upgrade/config] Making sure the configuration is correct:
W0616 11:03:31.287499   23794 common.go:94] WARNING: Usage of the --config flag for reconfiguring the cluster during upgrade is not recommended!
W0616 11:03:31.289003   23794 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.18.3"
[upgrade/versions] Cluster version: v1.18.0
[upgrade/versions] kubeadm version: v1.18.3
[upgrade/prepull] Will prepull images for components [kube-apiserver kube-controller-manager kube-scheduler etcd]
[upgrade/prepull] Prepulling image for component etcd.
[upgrade/prepull] Prepulling image for component kube-apiserver.
[upgrade/prepull] Prepulling image for component kube-controller-manager.
[upgrade/prepull] Prepulling image for component kube-scheduler.
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-controller-manager
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-etcd
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-apiserver
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-scheduler
[upgrade/prepull] Prepulled image for component kube-apiserver.
[upgrade/prepull] Prepulled image for component kube-controller-manager.
[upgrade/prepull] Prepulled image for component kube-scheduler.
[upgrade/prepull] Prepulled image for component etcd.
[upgrade/prepull] Successfully prepulled the images for all the control plane components
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.18.3"...
Static pod: kube-apiserver-k8s-master hash: e77d7ad5a9ffd5dcedfde09689852fa8
Static pod: kube-controller-manager-k8s-master hash: 55ac3d2123f8d2853aec1b91f6356fe8
Static pod: kube-scheduler-k8s-master hash: c4ab4fe810f1d424e4548c5a1a8d9408
[upgrade/etcd] Upgrading to TLS for etcd
[upgrade/etcd] Non fatal issue encountered during upgrade: the desired etcd version for this Kubernetes version "v1.18.3" is "3.4.3-0", but the current etcd version is "3.4.3". Won't downgrade etcd, instead just continue
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests523692244"
W0616 11:06:26.651289   23794 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-06-16-11-06-21/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-apiserver-k8s-master hash: e77d7ad5a9ffd5dcedfde09689852fa8
Static pod: kube-apiserver-k8s-master hash: e77d7ad5a9ffd5dcedfde09689852fa8
Static pod: kube-apiserver-k8s-master hash: 5063bb70bf194c1036ccebff991fbaab
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-06-16-11-06-21/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-controller-manager-k8s-master hash: 55ac3d2123f8d2853aec1b91f6356fe8
Static pod: kube-controller-manager-k8s-master hash: 1ad23772d655e5cb3223c503b03ed2b2
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-06-16-11-06-21/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-scheduler-k8s-master hash: c4ab4fe810f1d424e4548c5a1a8d9408
Static pod: kube-scheduler-k8s-master hash: a3aa0a013314dd1f87b99ce93b006ffb
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.18.3". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

## 证书相关升级

kubeadm自带的工具是1年证书过期. 时间太短. 

```
$ [root@k8s-master ~]# kubeadm alpha  certs check-expiration  
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jun 16, 2021 03:06 UTC   364d                                    no      
apiserver                  Jun 16, 2021 03:06 UTC   364d            ca                      no      
apiserver-etcd-client      Jun 16, 2021 03:06 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Jun 16, 2021 03:06 UTC   364d            ca                      no      
controller-manager.conf    Jun 16, 2021 03:06 UTC   364d                                    no      
etcd-healthcheck-client    Jun 16, 2021 03:06 UTC   364d             etcd-ca                 no      
etcd-peer                  Jun 16, 2021 03:06 UTC   364d            etcd-ca                 no      
etcd-server                Jun 16, 2021 03:06 UTC   364d             etcd-ca                 no      
front-proxy-client         Jun 16, 2021 03:06 UTC   364d            front-proxy-ca          no      
scheduler.conf             Jun 16, 2021 03:06 UTC   364d                                    no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jun 16, 2021 03:06 UTC   364d             no      
etcd-ca                 Jun 16, 2021 03:06 UTC   364d            no      
front-proxy-ca          Jun 16, 2021 03:06 UTC   364d             no 
```

自己编译当前版本的kubeadm-v1.18.3. 查看源码得知

前提要求: 

- go 1.14.3 环境 (我的go 版本是1.14.3, 建议不低于1.13.9)
- vim编辑器
- 最好是linux环境.

下载源码

```
$ mkdir $GOPATH/src/k8s.io
$ cd $GOPATH/src/k8s.io/
$ git clone https://gitee.com/louisehong/kubernetes.git
$ cd kubernetes
$ git checkout v1.18.3

```

重新编译kubeadm, 更改生成证书时长为100年

```

$ vim cmd/kubeadm/app/constants/constants.go
// CertificateValidity defines the validity for all the signed certificates generated by kubeadm
// 这里的时间再 乘 100 , 生成的证书时间即为100年. 
CertificateValidity = time.Hour * 24 * 365 * 100 

只编译kubeadm
$ make WHAT=cmd/kubeadm GOFLAGS=-v
$ cp _output/bin/kubeadm  /usr/bin/kubeadm
```

查看版本信息

```
## 改了一个文件, 版本就是dirty了~
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"18+", GitVersion:"v1.18.3-dirty", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"dirty", BuildDate:"2020-06-16T12:07:09Z", GoVersion:"go1.14.3", Compiler:"gc", Platform:"linux/amd64"}

```

### 备份以前的证书文件并更改集群证书时间.

```
$ cp -r  /etc/kubernetes/pki  /etc/kubernetes/pki.bak

## 更新所有证书

$ kubeadm  alpha certs renew all

## 查看证书过期时间. 发现有限期为100年了~~ 
$ # kubeadm alpha  certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 May 23, 2120 12:14 UTC   99y                                     no      
apiserver                  May 23, 2120 12:14 UTC   99y             ca                      no      
apiserver-etcd-client      May 23, 2120 12:14 UTC   99y             etcd-ca                 no      
apiserver-kubelet-client   May 23, 2120 12:14 UTC   99y             ca                      no      
controller-manager.conf    May 23, 2120 12:14 UTC   99y                                     no      
etcd-healthcheck-client    May 23, 2120 12:14 UTC   99y             etcd-ca                 no      
etcd-peer                  May 23, 2120 12:14 UTC   99y             etcd-ca                 no      
etcd-server                May 23, 2120 12:14 UTC   99y             etcd-ca                 no      
front-proxy-client         May 23, 2120 12:14 UTC   99y             front-proxy-ca          no      
scheduler.conf             May 23, 2120 12:14 UTC   99y                                     no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      May 12, 2120 10:13 UTC   99y             no      
etcd-ca                 May 12, 2120 10:13 UTC   99y             no      
front-proxy-ca          May 12, 2120 10:13 UTC   99y             no 

```