---
title: "docker 小技巧"
date: 2019-11-25T10:54:18+08:00
tags: [docker]
categories: [docker]
---

小记一下, 经常用到的命令..

## dockerfile构建容器镜像

```shell script
$ docker build [OPTIONS] PATH | URL | -
# example
$ docker build -t louisehong/jre-alpine:latest . 
$ docker build -t louisehong/jre-alpine:latest . 
# 私有仓库
$  docker build -t  harbor.youpenglai.cn/louisehong/jre-alpine:lates  .
```
对已经存在的镜像重新tag

```shell script
$ docker tag  louisehong/jre-alpine:latest harbor.youpenglai.cn/ louisehong/jre-alpine:latest
```

推送镜像, 需要对镜像仓库有读写权限

```shell script
# 先docker login
$ docker login harbor.youpenglai.cn
username:...
password:...
# 生成的授权信息在~/.docker/config.json
$ cat ~/.docker/config.json
{
	"auths": {
		"harbor.youpenglai.cn": {
			"auth": "*******************"
		}
}
# 如果是latest镜像, 则tag可以省略不写. 
$ docker push harbor.youpenglai.cn/louisehong/jre-alpine
```

## 容器启动

常用使用选项

```shell script
$ docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
[ OPTIONS ]
常用选项
-d 后台运行
--rm 退出容器后自动删除容器
-e  --env , key=value, kv命令行读取
    --env-file, 从文件读取
-i, --interactive ,交互式, 配合-t使用
-t, --tty , 启动一个tty, 配合-i使用
-p  --publish  , 暴露端口
-P  --publish-all, 暴露容器内所有端口映射宿主机上随机端口
-v  --volume  , 绑定挂载卷轴.

# example
$ docker run -p 8888:8888 --rm -e JAVA_OPTS=$JAVA_OPTS  harbor.youpenglai.cn/louisehong/jre-alpine
# 交互式, 一般是进入容器查询.
$ docker run -it --rm  harbor.youpenglai.cn/louisehong/jre-alpine sh
```

## 批量删除停止容器

 显示所有的容器，过滤出Exited状态的容器，取出这些容器的ID，然后删掉 

```bash
$ docker ps -a|grep Exited|awk '{print $1}'
$ docker rm $(docker ps -a|grep Exited|awk '{print $1}')
```

使用`prune`, 但是是在docker 1.13版本之后

```bash
 $ docker container prune
```

根据容器的状态删除

```bash
 $ docker rm $(docker ps -qf status=exited)
```

直接删除, 但是已经运行的是无法删除的

```bash
$ docker rm $(docker ps -a -q)
```

批量删除<none>镜像

```bash
$ docker rmi ` docker images |grep none |awk '{print $3}'`
```
