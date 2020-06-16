---
title: "kuberbetes serviceaccout访问集群"
date: 2020-06-06T15:34:18+08:00
tags: [pv,nfs,pvc,kubernetes]
categories: [kubernetes]
---

[TOC]

>  所有的 Kubernetes 集群都有两类用户：Kubernetes 管理的 Service Account 和普通用户。
>
>  普通用户由 Kubernetes 集群之外的独立服务管理，例如 keycloak、LDAP、OpenID Connect Identity Provider（Google Account、MicroSoft Account、GitLab Account）等。此类服务对用户的注册、分组、密码更改、密码策略、用户失效策略等有一系列管控过程，或者，也可以简单到只是一个存储了用户名密码的文件。Kubernetes 中，没有任何对象用于代表普通的用户账号，普通用户也不能通过 API 调用添加到 Kubernetes 集群。
>
>  与普通用户相对，Service Account 是通过 Kubernetes API 管理的用户。Service Account 是名称空间级别的对象，可能由 ApiServer 自动创建，或者通过调用 API 接口创建。Service Account 都绑定了一组 `Secret`，Secret 可以被挂载到 Pod 中，以便 Pod 中的进程可以获得调用 Kubernetes API 的权限。
>
>  对 API Server 的每次接口调用都被认为是：
>
>  - 由一个普通用户或者一个 Service Account 发起
>  - 或者是由一个匿名用户发起。
>
>  这意味着，集群内外的任何一个进程，在调用 API Server 的接口时，都必须认证其身份，或者被当做一个匿名用户。可能的场景有：
>
>  - 集群中某一个 Pod 调用 API Server 的接口查询集群的信息
>  - 用户通过 kubectl 执行指令，kubectl 调用 API Server 的接口完成用户的指令

## 创建Service Account 

example ：我想创建一个对namespace名称空间为huohua的有admin权限， 并且对其他名称空间有读权限的账号。

如果创建 `ClusterRoleBinding`，则，用户可以访问集群中的所有名称空间；

如果创建 `RoleBinding`，则，用户只能访问 RoleBinding 所在名称空间;

Kubernetes 集群默认预置了三个面向用户的 ClusterRole：

- `view` 可以查看 K8S 的主要对象，但是不能编辑
- `edit` 具备`view` 的所有权限，同时，可以编辑主要的 K8S 对象
- `admin` 具备 `edit` 的所有权限，同时，可以创建 Role 和 RoleBinding （在名称空间内给用户授权）

编辑weber.yaml文件, 可以为一个 Service Account 创建多个 Secret, 您可以定期更换 ServiceAccount 的 Secret Token，以增强系统的安全性.

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: '2020-06-06T05:39:00Z'
  name: weber
  namespace: huohua
  resourceVersion: '94695'
  selfLink: /api/v1/namespaces/huohua/serviceaccounts/weber
  uid: 9bcad8d1-9b32-4410-817d-bcb3a3ea9b9d
```

创建rolebinding

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: '2020-06-06T05:32:48Z'
  managedFields:
    - apiVersion: rbac.authorization.k8s.io/v1
      fieldsType: FieldsV1
      fieldsV1: {}
      manager: Mozilla
      operation: Update
      time: '2020-06-06T05:32:48Z'
  name: weber-rolebinding-2ra75
  namespace: huohua
  resourceVersion: '93652'
  selfLink: >-
    /apis/rbac.authorization.k8s.io/v1/namespaces/huohua/rolebindings/weber-rolebinding-2ra75
  uid: 91297d16-7fab-40eb-a9b4-16464548d5c8
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
  - kind: ServiceAccount
    name: weber
    namespace: default
```

创建clusterrolebinding

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: '2020-06-06T05:42:13Z'
  managedFields:
    - apiVersion: rbac.authorization.k8s.io/v1
      fieldsType: FieldsV1
      fieldsV1: {}
      manager: Mozilla
      operation: Update
      time: '2020-06-06T05:42:13Z'
  name: weber-clusterrolebinding-nbm3k
  resourceVersion: '95236'
  selfLink: >-
    /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/weber-clusterrolebinding-nbm3k
  uid: d8519a7b-34f9-4060-8ed9-78b35cb4d716
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: weber
    namespace: huohua
```

使用kubectl命令行创建

```
$ kubectl create serviceaccount weber -n huohua
$ kubectl create clusterrolebinding weber --clusterrole=view --serviceaccount=huohua:weber
$ kubectl create rolebinding weber --clusterrole=admin --serviceaccount=huohua:weber

## 查看token。
$ kubectl get secrets -n huohua $(kubectl -n huohua get secret | awk '/weber/{print $1}') -o go-template='{{.data.token}}' | base64 -d
$ kubectl describe secrets -n huohua $(kubectl -n huohua get secret | awk '/weber/{print $1}') | awk '/token:/{print $2}'
```

## 使用创建的sa账号登陆集群

使用kubectl命令创建`~/.kube/config`，master的ip解析`apiserver.cluster.local`

```
$ TOKEN=$(kubectl get secrets -n huohua $(kubectl -n huohua get secret | awk '/weber/{print $1}') -o go-template='{{.data.token}}' | base64 -d)
$ kubectl config set-credentials huohua-weber --token=$TOKEN
$ kubectl config set-cluster ctx-apiserver-cluster.local --insecure-skip-tls-verify=true --server=https://apiserver.cluster.local:6443
$ kubectl config set-context ctx-apiserver-cluster.local --cluster=ctx-apiserver-cluster.local --user=huohua-weber
$ kubectl config use-context ctx-apiserver-cluster.local
$ kubectl get pods -n huohua
```

直接编辑`~/.kube/config`， master的ip解析`apiserver.cluster.local`

```
apiVersion: v1
kind: Config
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://apiserver.cluster.local:6443
  name: ctx-apiserver-cluster.local
contexts:
- context:
    cluster: ctx-apiserver-cluster.local
    namespace: huohua
    user: huohua-weber
  name: ctx-apiserver-cluster.local
preferences: {}
current-context: ctx-apiserver-cluster.local
users:
- name: huohua-weber
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IndSVmdDbEtrX3VUWDVZTnliYmtOT0RXcWw5TTdnZkgwN3lNQUs2NjBjUWMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJodW9odWEiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoid2ViZXItdG9rZW4tOW0yZDciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoid2ViZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5YmNhZDhkMS05YjMyLTQ0MTAtODE3ZC1iY2IzYTNlYTliOWQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6aHVvaHVhOndlYmVyIn0.OWeAabBT3VVCVhWq99JX84SlgRguJ6MRzkxT4jI0ZZp8Lw8AT1EdtV8rjcLBwdqj_LYHYl8dpGpbUcl1PTmZQ8bDJ3YYwYP-3ND49H360T17DRPk6ds-shs_h0VshxMNCD_UJ4jh-qMzQ9yO7k-xCbPWKrOczwh19A-cDECl2YG1mEUVtsGOb9AMMYgSwvqkHL2g7ASjhA8qvrGdqtRw3YrfNbxkdx6cOE9dRkRTviFNvSvDxBCLAKy5WfIQYr_PCzVcq3641HU6txwcPjJ46_IuH1bYpuluPq5k5vBJ6y7aFGATN1zwwOazHMoF1r8GAZE3IT5vj5sZt6m-lW3zyw
```

