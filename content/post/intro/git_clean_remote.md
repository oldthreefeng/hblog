---
title: "git 远程分支清理"
date: 2018-11-16 16:17:10
tags: git
categories: [git]
---

[TOC]

## git 远程分支清理。
远程分支的查看只需要在 `git branch` 命令加一个 `-r`(`--remotes`) 参数即可

```
git branch -r
```

远程分支的清理，一方面是清理远程分支中，已经合入 `master` 的分支，另一方面是清理远程仓库已经删除了的分支，而本地还在跟踪的。


事实上，我们可以在每次 `git fetch` 时，添加一个参数 `-p` (`--prune`)，这样每次 fetch 远程仓库时都可以顺手删掉本地多余的分支（建议将 `git fetch -p` 直接 alias 到 `git fetch` 命令~）。

再来看第一种情况，虽然同样可以通过 `git branch -r --merged` 来查看已经合入 `master` 的分支，但由于远程分支不只是自己开发的，所以还需要别人的确认才能进行删除。
好在我们可以在命令行的帮助下快速筛选出每个人的分支，然后就可以把这份统计摘要发给 TA 来确认。
```
for branch in `git branch -r --merged | grep -v HEAD`; 
	do echo -e `git show --format="%ci %cr %an" $branch | head -n 1`; 
done | sort -r | grep AUTHOR_NAME
```
如果想查看更多的信息，可以在 `git show` 的 `format` 加上 `%s`（提交信息）和 `%h`（commit SHA1 前缀）

## git remote prune origin
第二种情况的清理非常简单，只需要执行

```
$ git remote prune origin --dry-run
Pruning origin
URL: http://*************************/qianxiang/web.git
 * [would prune] origin/0627_allen
 * [would prune] origin/develop_09061904_evan_julebu
 * [would prune] origin/feature_02211355_1000298_video
 * [would prune] origin/feature_06261651
 * [would prune] origin/feature_06271046_dengebenxi
 * [would prune] origin/feature_06271105
 * [would prune] origin/feature_06271105_evan_julebu
 * [would prune] origin/feature_06271228
 * [would prune] origin/feature_06271336
 * [would prune] origin/feature_06271341
 * [would prune] origin/feature_06281159_evan_fengkong4
 * [would prune] origin/feature_06281159_evan_fengkong4_mike
 * [would prune] origin/feature_07031939_summer
 * [would prune] origin/feature_07051049
 * [would prune] origin/feature_07060945
 * [would prune] origin/feature_07061030_jiekuan
 * [would prune] origin/feature_07091425
 * [would prune] origin/feature_07101552
 * [would prune] origin/feature_07111011_hongbao
 * [would prune] origin/feature_07161028_duxie
 * [would prune] origin/feature_07181505
 * [would prune] origin/feature_07191242
 * [would prune] origin/feature_07191545
 * [would prune] origin/feature_07201343_yemian
 * [would prune] origin/feature_07201413_fabu
```
`--dry-run`,仅是干跑一次，查询一下需要清理的分支。
如果需要整体清理
```
$ git remote prune origin
```

