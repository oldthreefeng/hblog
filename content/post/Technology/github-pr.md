---
title: "github pull request"
date: "2018-11-11 20:11:12"
tags: [github]
categories: [github,tools]
---



### 如何给开源项目贡献代码

- 代码仓库管理者给你添加该仓库的写入权限，这样的话可以直接push
- 如果不能直接push（大多数情况），采用经典的fork & pull request来提交代码，下面讲述这种情况

### PR方式贡献代码步骤

- 在 GitHub 上 `fork` 到自己的仓库，如 `my_user/WxJava`，然后 `clone` 到本地，并设置用户信息。

```
$ git clone git@github.com:my_user/WxJava.git
$ cd weixin-java-tools
$ git config user.name "yourname"
$ git config user.email "your email"
```

- 修改代码后提交，并推送到自己的仓库。

```
$ #do some change on the content
$ git commit -am "Fix issue #1: change something"
$ git push
```

- 在 GitHub 网站上提交 Pull Request。
- 定期使用项目仓库内容更新自己仓库内容。

```
$ git remote add upstream https://github.com/Wechat-Group/WxJava
$ git fetch upstream
$ git checkout develop
$ git rebase upstream/develop
$ git push -f origin develop
```