---
title: "pipeline实现并发构建"
date: "2019-09-16 15:08:32"
tags: 
  - ops
  - jenkins
  - pipeline
categories: [ops,jenkins]
---

[TOC]

> 背景
>
> 公司的`jenkins`管理了很多的项目,基于主机Ecs管理,比较传统,采用的是脚本打包构建,前端采用nginx进行反向代理,后端采用的是jetty容器构建java项目.后端主机有两台,这样构建打包需要分别在两台主机上打包.构建速度慢.因此,想采用并发构建的方法.pipeline中的parallel刚好满足要求.

## 用到的相关插件

> pipeline
>
> ssh-steps-plugin

## 配置pipeline

构建好job,直接配置`pipeline`.这里提供两种语法

### Scripted Pipeline

在脚本式的pipeline中,只在一个或者多个node块执行

脚本语言的`pipeline`,语法比较简单,这里采用的方法是ssh远程连接`test01`和`test02`主机,然后并发执行脚本,远程连接具体可以参考[ssh-steps-plugin](https://github.com/jenkinsci/ssh-steps-plugin)

- `credentials`

`jenkins`的用户验证,提前设置好,建议使用ssh基于key的认证. 具体设置参考官方,[credentials](https://jenkins.io/zh/doc/book/using/using-credentials/)

```
node {
    stage('Parallel Stage') {
        parallel 'test02': {
                def remote = [:]
                remote.name = "test01"
                remote.host = "host"
                remote.port = port
                remote.allowAnyHosts = true
                withCredentials([sshUserPrivateKey(credentialsId: 'louis', keyFileVariable: 'identity', passphraseVariable: 'passphrase', usernameVariable: 'username')]) {
                remote.user = username
                remote.identityFile = identity
                remote.passphrase = passphrase
                stage("test01-build") {
                sshCommand remote: remote, command: 'uname -r'
                }
            }
        }, 'test02': {
                def remote = [:]
                remote.name = "test02"
                remote.host = "host"
                remote.port = port
                remote.allowAnyHosts = true
                withCredentials([sshUserPrivateKey(credentialsId: 'louis', keyFileVariable: 'identity', passphraseVariable: 'passphrase', usernameVariable: 'username')]) {
                remote.user = username
                remote.identityFile = identity
                remote.passphrase = passphrase
                stage("test02-build") {
                sshCommand remote: remote, command: 'uname -r'
                }
            }
        }
    }
    stage('DingDing') {
        script {
            def msg = "构建失败，请及时查看原因"
            def imageUrl = "https://www.iconsdb.com/icons/preview/red/x-mark-3-xxl.png"
            def dingdingtoken = "https://oapi.dingtalk.com/robot/send?access_token=****************************"
            if (currentBuild.currentResult=="SUCCESS"){
                imageUrl= "http://icons.iconarchive.com/icons/paomedia/small-n-flat/1024/sign-check-icon.png"
                msg ="发布成功"
            }
            sh "sh ${JENKINS_HOME}/dingding.sh ${BUILD_TAG} ${BUILD_URL} ${msg} ${imageUrl} ${dingdingtoken}"
        }
    }
}
```

- 采用dingding脚本通知,钉钉的插件只能使用声明式pipeline中.所以只能自己造轮子了.

### curl发送钉钉消息

使用钉钉机器人通知非常简单，通过 curl 命令行工具即可发送通知。
```
curl 'https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxx' \
   -H 'Content-Type: application/json' \
   -d '
  {"msgtype": "text", 
    "text": {
        "content": "我就是我, 是不一样的烟火"
     }
  }'
```

主要的方式,采用shell脚本,利用curl工具实现.

```
$cat dingding.sh
#!/bin/sh
title=$1
messageUrl=$2
picUrl=$4
text=$3
PHONE="158215*****"
TOKEN=$5
DING="curl -H \"Content-Type: application/json\" -X POST --data '{\"msgtype\": \"link\", \"link\": {\"messageUrl\": \"${messageUrl}\", \"title\": \"${title}\", \"picUrl\": \"${picUrl}\", \"text\": \"${text}\",}, \"at\": {\"atMobiles\": [${PHONE}], \"isAtAll\": false}}' ${TOKEN}"
eval $DING
```

- 验证

`jenkins`的环境变量比较多,建议参考这篇文章[jenkins可用环境变量列表及使用](https://www.cnblogs.com/EasonJim/p/6758382.html)

```
获取$1,$2,$3,$4,$5.
$1 为title,这里我设置为build_tag
$2 为messageUrl, 这里我这种为Build_Url
$3 为text(消息内容), 这里根据构建成功还是失败,进行判断
$4 为picUrl(对勾还是xx), 
$5 DingDingToken.
查看pipeline中的日志,有执行sh的记录.模板记录如下:
sh /var/jenkins_home/dingding.sh jenkins-test-qx-44 http://106.75.107.122:8088/job/test-qx/44/ 发布成功 http://icons.iconarchive.com/icons/paomedia/small-n-flat/1024/sign-check-icon.png https://oapi.dingtalk.com/robot/send?access_token=****
```

### Declarative Pipeline

声明式的pipeline语法,pipeline贯穿了整个工作流.

```
pipeline {
    agent any
    stages {
        stage('Parallel Stage') {
            failFast true
            parallel {
                stage('并行一') {
                    steps {
                        script {
                            try {
                                echo "并行一"
                                sh "ssh -f -n -p port root@host uname -r"
                            }catch(err){
                                echo "${err}"
                                sh 'exit 1'
                            }
                        }


                    }
                }
                stage('并行二') {
                    steps {
                        script {
                            try {
                                echo "并行二"
                                sh "ssh -f -n -p port root@host uname -r"
                            }catch(err){
                                echo "${err}"
                                sh 'exit 1'
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                def msg = "构建失败，请及时查看原因"
                def imageUrl = "https://www.iconsdb.com/icons/preview/red/x-mark-3-xxl.png"
                if (currentBuild.currentResult=="SUCCESS"){
                    imageUrl= "http://icons.iconarchive.com/icons/paomedia/small-n-flat/1024/sign-check-icon.png"
                    msg ="发布成功，干得不错！"
                }
                dingTalk accessToken:"***********",message:"${msg}",imageUrl:"${imageUrl}",jenkinsUrl:"${BUILD_URL}",notifyPeople: ''
            }
        }
    }
}
```

更多语法参考[官网](https://jenkins.io/doc/book/pipeline/syntax/)



## 参考

- [dingding](https://ding-doc.dingtalk.com/doc#/serverapi2/ye8tup)
- [CURL 钉钉机器人 JSON 传参](https://blog.csdn.net/u011836730/article/details/80430042)

