---
title: hexo简易部署到云主机
date: 2018-06-06 22:33:14
tags: [hexo]
categories: [hexo]
---
# hexo简易部署到云主机

## 写在前面

一开始将自己`hexo`部署到`github`，结果发现打开页面速度有点慢，然后又将其同时部署到`coding`,实现双线路访问，国内解析记录到`coding`，国外解析到`github`，这样确实网站的速度能提高不少，但是国内访问因为是经过`coding`，所以打开网站会有广告，这点不能容忍，于是想到自己的服务器也还空闲着，于是想到可以部署到自己的服务器上，折腾开始 。 

## 简易部署思想

- 云主机上直接利用hexo，generate生成静态页面文件夹public，然后利用apche或者nginx服务，直接指向pubilc文件，从而简易实现部署，不需要利用GIT仓库中转。
  - 在云主机上搭建一个git裸仓库，然后使用nginx作为网页服务器，就可以轻松将Hexo博客通过git部署到云主机上。 这个方法有种周转的意思，既然是自己的主机，直接可以利用上诉思想。

## Hexo 简介

[Hexo](https://hexo.io/) 是一个 Node.js 编写的静态网站生成器。Hexo 主要用来做博客框架，同时 Hexo 也整合了将静态网站部署到 Github 的功能，所以也很适合用来做 Github 项目的文档。

我们可以使用 Hexo，根据写好的 HTML 布局（既 Hexo 的主题），将 MarkDown 文件生成成主题对应的静态 html/css/js 文件。Hexo 提供了将静态文件部署到 Github 分支上的配置。也就是说，我们可以使用 MarkDown 来维护文档，当写好部署配置之后，使用一个命令就可以将文档生成并发布到 Github 的 gh-pages 分支上。

# 服务器部署(推荐)

## 服务器端安装hexo

### centos6.9
直接yum安装npm会各种报错，下面这种用nvm安装npm不仅不报错，速度还可以
```
]# wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.31.1/install.sh | bash
]# . .bashrc
]# nvm install stable
]# npm install -g hexo-cli
```
### cnetos7.4
简单省事的方法可以按照cnetos6.9的方法，但是用惯了yum的同学还是习惯这个
```
]# yum install -y npm
]# npm install -g hexo-cli
```

## 安装ngnix

```
]# yum install -y nginx
```
## 配置文件,修改网站根目录

```
]# vim /etc/nginx/nginx.conf
server
   server {
        listen       80;
        # server_name 填写自己的域名
        server_name  www.fenghong.tech;
        # 这里root填写自己的网站根目录
        root         /hexo;
        index index.html index.php index.htm;
        #/usr/local/tomcat/webapps/Forum

}
]# systemctl start nginx  	#启动nginx服务
]# systemctl enable ngnix	#开机自动启动
```

## 在hexo的家目录中生成指定的静态页面

```
]# cd /data
]# hexo init hexo    	#会生成hexo文件夹
]# cd /data/hexo 	 	#进入到hexo的家目录
]# hexo g
]# ln -s /data/hexo/public /hexo	#将public文件夹指向nginx网站的根目录
```
## DNS解析

- 在`dns`设置解析记录，设置解析`A`记录`www`解析到服务器`IP地址`, 解析线路默认,增加如下记录
  - www 	A	34.231.75.23 
- 注意：hexo的家目录的权限，否则会出现403报错.

# 本地云同步

在服务器端操作总会感觉到延迟，操作起来很不流畅；想了想，还是在本地主机编辑好文章，直接利用rsync推送至vps，达到同步效果。

## 本地hexo主机配置

```
]# yum istall -y npm
]# npm install -g hexo-cli
]# npm install hexo-generator-search --save
]# npm install hexo-generator-searchdb --save
]# npm install hexo-deployer-git --save
]# npm install hexo-renderer-scss --save
]# cd /data
]# hexo init hexo    	#会生成hexo文件夹
]# cd /data/hexo 	 	#进入到hexo的家目录
]# hexo g				#生成public文件夹
```

## 推送至vps

利用`rsync`推送至远程主机，假设我主机`IP:46.123.43.127`，当然这不是我的主机。默认vps端已经安装好`nginx`，且主机的`nginx服务器网站根目录为/www`，如果未安装，请安装`nginx`服务器。

rsync的具体用法了解，[参见](https://www.cnblogs.com/f-ck-need-u/p/7220009.html)

```shell
]#vim push.sh
#!/bin/bash
[ "$1" == "clean" ] && hexo clean
hexo g && rsync -v -r /data/hexo/public/  46.123.43.127:/www
]# chmod +x push.sh
]# ./push.sh
```

我是个爱折腾的，至此，云主机搭建完毕，hexo静态页面很清爽！  

博客主题配置，及文章上传方法，请看上篇博文[hexo进阶](https://fenghong.tech/hexo-Developed.html)

# git

使用rsync也算是比较不fashion的，还是用回高大上的git吧。

### 使用`git`自动化部署博客

自动化部署主要用到了`git`-`hooks`同步

- 服务器建立裸库，这里要用`git`用户登录，确保`git`用户拥有仓库所有权

  ```
  su git
  cd /var/repo/
  git init --bare blog.git
  ```

- 使用 git-hooks 同步网站根目录
  在这里我们使用的是 `post-update`这个钩子（也有可能是`post-receive`，具体进入文件就知道了），当git有收发的时候就会调用这个钩子。 在 `/var/repo/blog.git` 裸库的 `hooks`文件夹中

  ```
  vim /var/repo/blog.git/hooks/post-receive
  # 编辑文件，写入以下内容
  ```

  ```
  #!/bin/sh
  git --work-tree=/var/www/hexo --git-dir=/var/repo/blog.git checkout -f
  ```

  保存后，要赋予这个文件可执行权限

  ```
  chmod +x post-receive
  ```

- 配置`_config.yml`,完成自动化部署
  打开`_config.yml`, 找到`deploy`，#云主机改变端口后可以按注释的。比如ssh端口为12345

  ```
  deploy:
    type: git
    repo:
      github: git@github.com:oldthreefeng/oldthreefeng.github.io.git
      #www: ssh://git@fenghong.tech:12345/blog/blog.git
      www: git@fenghong.tech:/var/repo/blog.git  
   
    branch: master
  ```

保存后，即可测试部署

```
hexo clean && hexo g -d
```

- 至此，我们已经成功部完成，并且访问自己的服务器端比访问`github`快多了，国外速度也是很好

### 常见问题

我在部署过程中，执行 `hexo d`发现部署老是出错，什么权限不允许之类的，这里我们需要检查我们在上述的git操作部署是否使用了`git`用户操作，若是没有，需要给相应的目录更改用户组；
使用`chown -R git:git /var/repo/`这条命令递归的将`repo`目录及其子目录用户组设置为`git`，同时`chown -R git:git /var/www/hexo`，这样即可解决此类问题.
