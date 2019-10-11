---
title: "kuberbetes 基于nfs的pvc"
date: 2019-09-09 15:56:12
tags: 
  - pv
  - nfs
  - kubernetes1.15.3
  - pvc
categories: [kubernetes]
---

[TOC]

>  k8s提供了`emptyDir`,`hostPath`,`rbd`,`cephfs`等存储方式供容器使用,不过这些存储方式都有一个缺点:开发人员必须得知指定存储的相关配置信息,才能使用存储.例如要使用`cephfs`,`Pod`的配置信息就必须指明`cephfs`的`monitor`,`user`,`selectFile`等等,而这些应该是系统管理员的工作.对此,k8s提供了两个新的API资源:`PersistentVolume`,`PersistentVolumeClaim`

- PV(`PersistentVolume`)是管理员已经提供好的一块存储.在k8s集群中,`PV`像`Node`一样,是一个资源

- PVC(`PersistentVolumeClaim`)是用户对`PV`的一次申请.`PVC`对于`PV`就像`Pod`对于`Node`一样,`Pod`可以申请`CPU`和`Memory`资源,而`PVC`也可以申请`PV`的大小与权限

- 有了`PersistentVolumeClaim`,用户只需要告诉`Kubernetes`需要什么样的存储资源,而不必关心真正的空间从哪里分配,如何访问等底层细节信息;这些`Storage Provider`的底层信息交给管理员来处理,只有管理员才应该关心创建`PersistentVolume`的细节信息.

# 基于NFS创建动态的`pv`

## NFS

### 创建NFS服务端

安装`nfs-tuils`, ubuntu系统安装`nfs-common`

```bash
yum install nfs-utils
systemctl enable nfs
systemctl enable rpcbind
systemctl start rpcbind nfs
```

配置nfs共享目录,`/data/k8s/nfs`,具体含义看我当时写的一篇[nfs文档](https://www.fenghong.tech/nfs.html)

```
vim /etc/exports
/data/k8s/nfs 10.9.0.0/16(no_root_squash,rw,sync,no_subtree_check)
```

重启服务

```bash
systemctl restart rpcbind nfs
```

### 客户端配置

客户端需要安装`nfs-utils`,否则nfs挂载会一直pending,导致pvc分配失败.

```bash
## centos
yum install nfs-utils
## ubuntu
sudo apt-get install nfs-common 
```

## 配置动态nfs-client

此次配置均参考[Kubernetes NFS-Client Provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client/deploy)

nfs-client-provisioner 是一个Kubernetes的简易NFS的外部provisioner，本身不提供NFS，需要现有的NFS服务器提供存储

- PV以 `${namespace}-${pvcName}-${pvName}`的命名格式提供（在NFS服务器上）
- PV回收的时候以 `archieved-${namespace}-${pvcName}-${pvName}` 的命名格式（在NFS服务器上）

### 部署nfs-client-provisioner

修改deployment文件并部署 ,需要修改的地方只有NFS服务器所在的IP地址和挂载目录.

```
cat deployment.yaml 
```
```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs  ## 可以按需更改,换成自定义的,必须配合class.yaml
            - name: NFS_SERVER
              value: 10.9.106.18    ## NFS服务器所在的IP地址
            - name: NFS_PATH
              value: /data/k8s/nfs  ## NFS服务器所在的共享目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.9.106.18     ## NFS服务器所在的IP地址
            path: /data/k8s/nfs     ## NFS服务器所在的共享目录
```

执行apply即可

```
 kubectl apply -f deployment.yam
```

- class.yaml

修改StorageClass文件并部署

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage   
provisioner: fuseim.pri/ifs    ## 可以按需更改,换成自定义的,必须配合deployment.yaml
```

### 授权

- `rbac.yaml`

```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io

```

如果启用了RBAC,则执行

```
kubectl apply -f rbac.yaml
```

## 测试

- test.yaml

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: gcr.io/google_containers/busybox:1.24  ## 需要预先下好,或者使用dockerhub
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

- 执行

```
kubectl apply -f test.yaml
pod/test-pod created
persistentvolumeclaim/test-claim created


```

查看pvc和pod状态

```
$ kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Bound    pvc-ff0ee478-adf9-47a7-b376-d1a9edfd3046   1Mi        RWX            managed-nfs-storage   5m8s
```

验证:

```
$ cd /data/k8s/nfs/default-test-claim-pvc-ff0ee478-adf9-47a7-b376-d1a9edfd3046/
$ ls
SUCCESS
```

删除pvc,会生成archived

```
$ kubectl delete -f test.yaml
persistentvolumeclaim "test-claim" deleted
pod "test-pod" deleted
$  ls /data/k8s/nfs/archived-default-test-claim-pvc-ff0ee478-adf9-47a7-b376-d1a9edfd3046/SUCCESS 
/data/k8s/nfs/archived-default-test-claim-pvc-ff0ee478-adf9-47a7-b376-d1a9edfd3046/SUCCESS
```

### 报错相关

- 错误1.一直处于pending状态,由于k8s集群的其他客户端未安装`nfs-utils`,安装即可

```
$ kubectl get pvc |grep test
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Pending 							         managed-nfs-storage   5m8s
```

- 错误2.镜像` ImagePullBackOff`错误

```
$ kubectl get pods |grep test
test-pod                                         0/1     ImagePullBackOff   0          3m49s
```

```
# gcr.io处于不可描述的状态

# 使用镜像然后tag即可,或者使用dockerhub上的镜像也可以.这就需要修改test.yaml
$ docker pull gcr.azk8s.cn/google_containers/busybox:1.24
$ docker tag gcr.azk8s.cn/google_containers/busybox:1.24 gcr.io/google_containers/busybox:1.24
```

### 参考

- [jimmysong.io大佬](https://jimmysong.io/kubernetes-handbook/practice/using-nfs-for-persistent-storage.html)