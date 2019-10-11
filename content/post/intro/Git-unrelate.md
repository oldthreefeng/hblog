---
title: "refusing to merge unrelated histories"
date: 2019-07-16 14:43:10
tags: [git]
categories: [git]
---

## Git的报错

在使用Git的过程中有时会出现一些问题，那么在解决了每个问题的时候，都需要去总结记录下来，下次不再犯。

### `fatal: refusing to merge unrelated histories`

好几天前开了新项目，开发人员在dev分支开发，对master没有写权限，从init提交到版本第一期快开发完了，才想到要合master。结果就发发生了这个错误，记录一下防止下次再犯.

```
$ git pull
$ git checkout master
$ git merge develop
fatal: refusing to merge unrelated histories

## 解决方案 ##
$ git merge develop --allow-unrelated-histories
```

如果是`git pull`.

```
git pull
fatal: refusing to merge unrelated histories 

$ git pull origin master --allow-unrelated-histories
```

