---
title: "基于uk8s部署的CI/CD"
date: 2019-11-01T21:54:18+08:00
tags: [pv,ci,pvc,kubernetes,uk8s,jenkins,gogs]
categories: [kubernetes]
---

## 基于uk8s部署CI/CD ##

### uk8s的集群创建 ###

> 创建uk8s集群.参考[创建集群](https://docs.ucloud.cn/compute/uk8s/userguide/createcluster)

> 获取内网凭证. 创建集群大概5分钟左右, 然后去集群页面获取凭证

![1572608654035](https://code.aliyun.com/louisehong/images/raw/master/k8s/20191101194247.png)

> 管理主机为同VPC下的一台Uhost.

首先创建凭证,安装kubectl进行管理集群.

```
$ mkdir .kube
$ vim .kube/config ## 将上面获取的内网凭证复制即可
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.16.0/bin/linux/amd64/kubectl

$ chmod +x kubectl && mv kubectl /usr/bin
$ kubectl get nodes
NAME            STATUS   ROLES      AGE   VERSION
10.23.140.24    Ready    k8s-node   8h    v1.15.5
10.23.162.135   Ready    k8s-node   8h    v1.15.5
10.23.89.169    Ready    k8s-node   8h    v1.15.5
```

## 基本的服务构建 ##

### 项目docker化 ##

项目docker化, 以golang项目作为例子.写一个`http server`, 如`hello-ucloud.go`,

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		log.Printf("%s %s %s %s", r.Method, r.URL, r.Host, r.RemoteAddr)
		version := os.Getenv("VERSION")
		if version == "" {
			version = "v1"
			// version = "v2" 模拟发布. v3 v4 v5 v6
		}
		fmt.Fprintf(w, "hello UCloud! version: %s\n", version)
	})
	log.Fatal(http.ListenAndServe(":8000", nil))
}
```

编写`dockerfile`, 主要是编译go项目, 然后把生成的二进制拷贝到Alpine运行, 依赖最小, 生成的镜像也很小, 只有30M左右

```
FROM  uhub.service.ucloud.cn/hahahahaha/golang:1.12.7  AS builder

ENV GO111MODULE=on
ENV GOPROXY=https://goproxy.io

WORKDIR /root

COPY hello-ucloud.go .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build  -o /deploy hello-ucloud.go

FROM alpine:3.7
RUN apk add tzdata ca-certificates && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && apk del tzdata && rm -rf /var/cache/apk/* \
COPY --from=builder /deploy /bin/deploy
ENTRYPOINT ["/bin/deploy"]
```

编写yaml文件进行部署至k8s, 一个deployment和一个service, 暴露方式为LoadBalancer+ClusterIp

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
  labels:
    app: hello-world
    env: testing
    release: hello-go  ## 这个与jenkins的job名字相同.用来筛选
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers: 
      - name: helloworldc
        image: uhub.service.ucloud.cn/gitci/hello-go:v1 
        ports: 
        - containerPort: 8000
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hello-world
  name: hello-world-deployment
spec:
  ports: 
  - port: 8080
    protocol: TCP
    targetPort: 8000
  selector:
    app: hello-world
  type: LoadBalancer  ##会生成10M宽带的EIP
```
### gogs搭建仓库 ### 

```bash
#!/bin/bash
data=`pwd`
docker run -d \
 --hostname gogs.wangke.co \ 
 --name=gogs -p 22:22 -p 80:3000 \
 -v ${data}/data:/data \ 
 gogs/gogs
```
具体细节看查询[gogs安装](https://blog.fenghong.tech/post/ops/gogs-repo-install/)

仓库的`webhook`设置

比如我jenkins的job名称为`hello-go`,jenkins的url为jenkins的部署地址. 则`webhook`为 `http://10.23.168.57:8080/gogs-webhook/?job=hello-go`, 记得使用`secrets`, 避免瞎调用.

### jenkins搭建 ###

jenkins搭建, 使用docker进行部署.

```bash
#!/bin/bash
docker stop jenkins && docker rm jenkins
docker run  --privileged \
 --name jenkins  -d  \
 -p 8080:8080   -p 50000:50000  \
 -v /srv/jenkins/data:/var/jenkins_home  \
 -v /var/run/docker.sock:/var/run/docker.sock \
 -v $(which docker)r:/bin/docker \
 -v /usr/bin/kubectl:/bin/kubectl \
 -v /root/.kube:/var/jenkins_home/.kube   
 uhub.service.ucloud.cn/gitci/blueocaen:latest
```
这里说明一下, 使用jenkins做发布, 希望jenkins能直接调用kubectl命令来调用apiserver,实现资源管理.同时在jenkins容器内部要完成打包工作. 万一出现docker命令有permission的问题
`chmod 777 /var/run/docker.sock`.

配置`webhook`, 采用gogs的插件, `branch` 选择为master, 则只构建master的推送, 其他都不进行构建.

![1572620352737](https://code.aliyun.com/louisehong/images/raw/master/k8s/1572620352737.png)

jenkins里面的job部署简易脚本. 带回滚的版本, 如果镜像更新失败, 则打印生成的容器100行日志.并执行undo回滚操作.

![1572619800580](https://code.aliyun.com/louisehong/images/raw/master/k8s/1572619800580.png)

```bash
#!/bin/bash
UPDATE_TIMEOUT=180s
LabelSelector="env=testing,release=${JOB_BASE_NAME}"
deploymentName=$(kubectl get deployment  -l ${LabelSelector} -o go-template --template='{{range .items}}{{ .metadata.name}}{{end}}')
[[ -d hello ]] && rm -rf hello/ 
git clone git@10.23.168.57:louis/hello.git
cd hello/
docker build -t uhub.service.ucloud.cn/gitci/hello-go:${BUILD_NUMBER} .
docker push uhub.service.ucloud.cn/gitci/hello-go:${BUILD_NUMBER}
kubectl set image deploy ${deploymentName} *=uhub.service.ucloud.cn/gitci/hello-go:${BUILD_NUMBER}
timeout --signal SIGKILL ${UPDATE_TIMEOUT} kubectl rollout status deployment ${deploymentName}
if [[ $? -ne 0 ]];then
    	echo "################# Update timeout， rollback!!!!!"
        samplePod=$(kubectl get po -l ${LabelSelector} | awk 'NR>1{print $1;exit}')
        kubectl logs --tail=100 ${samplePod}
        kubectl rollout undo deployment  ${deploymentName}
        exit 500
fi
```


gogs的push输出的日志.

```
Gogs-ID: ceaba96b-9a78-424f-a128-2f6d49d1acfc
Running as SYSTEM
Building in workspace /var/jenkins_home/workspace/hello-go
[hello-go] $ /bin/sh -xe /tmp/jenkins1322734338419902619.sh
+ UPDATE_TIMEOUT=180s
+ LabelSelector='env=testing,release=hello-go'
+ kubectl get deployment -l 'env=testing,release=hello-go' -o go-template '--template={{range .items}}{{ .metadata.name}}{{end}}'
+ deploymentName=hello-world-deployment
+ '[[' -d hello ]]
+ git clone git@10.23.168.57:louis/hello.git
Cloning into 'hello'...
+ cd hello/
+ docker build -t uhub.service.ucloud.cn/gitci/hello-go:2 .
Sending build context to Docker daemon  54.78kB

Step 1/10 : FROM  uhub.service.ucloud.cn/hahahahaha/golang:1.12.7  AS builder
 ---> be63d15101cb
Step 2/10 : ENV GO111MODULE=on
 ---> Using cache
 ---> c85ce3a02464
Step 3/10 : ENV GOPROXY=https://goproxy.io
 ---> Using cache
 ---> eb6fe76cb916
Step 4/10 : WORKDIR /root
 ---> Using cache
 ---> 87c78943ae7a
Step 5/10 : COPY hello-ucloud.go .
 ---> Using cache
 ---> ac1cc24cf35c
Step 6/10 : RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build   -o /deploy hello-ucloud.go
 ---> Using cache
 ---> c041bea3c691
Step 7/10 : FROM alpine:3.7
 ---> 6d1ef012b567
Step 8/10 : RUN apk add tzdata ca-certificates && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime     && echo "Asia/Shanghai" > /etc/timezone     && apk del tzdata && rm -rf /var/cache/apk/*     && sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories     && apk add --no-cache git     && apk add --no-cache openssh
 ---> Using cache
 ---> e82cf08ba888
Step 9/10 : COPY --from=builder /deploy /bin/deploy
 ---> Using cache
 ---> da478659e18e
Step 10/10 : ENTRYPOINT ["/bin/deploy"]
 ---> Using cache
 ---> 47b75b287f9a
Successfully built 47b75b287f9a
Successfully tagged uhub.service.ucloud.cn/gitci/hello-go:2
+ docker push uhub.service.ucloud.cn/gitci/hello-go:2
The push refers to repository [uhub.service.ucloud.cn/gitci/hello-go]
472428a9d8ca: Preparing
d1d372f97ce2: Preparing
3fc64803ca2d: Preparing
472428a9d8ca: Layer already exists
d1d372f97ce2: Layer already exists
3fc64803ca2d: Layer already exists
2: digest: sha256:0a5e3faf33f3514ab188a6bdb28ca5f8915e9cad791d667827237ecffc3dbd25 size: 950
+ kubectl set image deploy hello-world-deployment '*=uhub.service.ucloud.cn/gitci/hello-go:2'
deployment.extensions/hello-world-deployment image updated
+ timeout --signal SIGKILL 180s kubectl rollout status deployment hello-world-deployment
Waiting for deployment "hello-world-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "hello-world-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "hello-world-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "hello-world-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hello-world-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "hello-world-deployment" successfully rolled out
+ '[[' 0 -ne 0 ]]
Finished: SUCCESS
```

![1572619225378](https://code.aliyun.com/louisehong/images/raw/master/k8s/1572619225378.png)

推送一个push-trigger,立马就更新了. 

![1572619287421](https://code.aliyun.com/louisehong/images/raw/master/k8s/1572619287421.png)

源码仓库在[gogs.wangke.co](https://gogs.wangke.co/go/uk8s-test)

### 不足之处 ### 

项目的镜像打包, 到`docker push`, 在生产环境的时候, 应该分别构建, 万一构建镜像失败则直接返回, 不用调用api了, 调用kubeapi操作应该在构建镜像之后分开脚本. 这里因为是test环境, 所以放在一起了. 



### 参考 ###

- [u8ks](https://docs.ucloud.cn/compute/uk8s/manageviakubectl/connectviakubectl)