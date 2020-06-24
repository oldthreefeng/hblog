---
title: "jenkins+java+docker+kubernetes实现自动化部署"
date: 2020-06-24T10:54:18+08:00
tags: 
  - git
  - jenkins
  - docker
  - kubernetes
categories: [ops,jenkins]
---

[TOC]

## 背景
> 基于springboot项目, 部署在kubernetes集群, 并实现自动化构建部署.  减轻开发的部署压力,  仅仅提交代码, 后续便由运维部署. 开发无需关心部署内在逻辑. 

## 项目架构图

![](https://pic.fenghong.tech/k8s/java_devops.jpg)

公司内部的k8s集群，主要跑的一个内部的crm系统，该系统的源码及构建管理使用的是`gitlab+docker+maven`, 代码测试用的`sonarqube`, 包管理使用的`harbor`, 部署使用的`jenkins+kubernetes`, 监控使用`kube-prometheus`, 日志使用`loki`,  配置中心使用`nacos`, 消息队列使用`rabbitmq`, 数据库使用`mysql+redis`

整个访问链路内网使用nginx进行负载均衡代理至kubernetes的ingress. 因为需要外网介入访问, 我们这边使用了内网穿透[frp](https://github.com/fatedier/frp)进行穿透访问crm系统. 

### 项目构建

使用jenkins进行构建, 项目名称即为java的项目名称.

源码管理`SourceCode Management`选择git, 填入项目的源码地址即可. `build trigger`

build阶段使用mvn进行构建 `mvn clean install -Dmaven.test.skip=true`

手写Dockerfile文件. 

```
cd $WORKSPACE/$JOB_BASE_NAME

mvn clean package -Dmaven.test.skip=true

cat >> $WORKSPACE/$JOB_BASE_NAME/target/Dockerfile  <<EOF
FROM harbor.youpenglai.cn/library/jre-8-alpine-bash:latest
MAINTAINER louis <louis.hong@junhsue.com>
WORKDIR /opt/
ADD $JOB_BASE_NAME.jar .
CMD [ "bash", "-c", "java -Xmx512m -Xms512m -Xss256k -Xmn128m -Dspring.profiles.active=dev  -Djava.security.egd=file:/dev/./urandom -jar /opt/$JOB_BASE_NAME.jar"]

EOF
```

镜像打包及push阶段, 使用jenkins的[docker-step](https://plugins.jenkins.io/docker-build-step/)插件.  

![](https://pic.fenghong.tech/k8s/jenkins_20200624112935.jpg)

### 部署阶段

```
NAMESPACE=huohua
REGISTRY=harbor.youpenglai.cn
K8S_TOKEN='**********'
## 使用kubernetes的API进行部署
curl -X PATCH \
  -H "content-type: application/strategic-merge-patch+json" \
  -H "Authorization:Bearer $K8S_TOKEN" \
  -d '{"spec":{"template":{"spec":{"containers":[{"name":"'${JOB_BASE_NAME}'","image":"'${REGISTRY}'/'${NAMESPACE}'/'${JOB_BASE_NAME}':'${BUILD_NUMBER}'"}]}}}}' \
  "https://kuboard.youpenglai.com/k8s-api/apis/apps/v1/namespaces/${NAMESPACE}/deployments/svc-${JOB_BASE_NAME}"

sleep  20s 

unset LabelSelector

LabelSelector="k8s.kuboard.cn/name=svc-${JOB_BASE_NAME},release=${JOB_BASE_NAME}"

newPod=$(kubectl get po -l ${LabelSelector} -n ${NAMESPACE} | awk 'NR>1{print $1;exit}')

[ -z $newPod ] && exit 

kubectl logs --tail 100  $newPod  -n ${NAMESPACE}
```

