```
title: "git-rebase合并多commit记录"
date: 2020-06-30T10:18:18+08:00
tags: [git,rebase]
categories: [git]
```

[toc]

### 新建测试仓库

为了不破坏现有的仓库，我们首先创建一个新建一个实验repo，所有操作都在该仓库下操作。创建命令如下：

```
$ mkdir rebase-repo

$ cd rebase-repo

$ git init rebase-repo

$ git commit --allow-empty -m "init"

$ git remote add origin git@code.aliyun.com:louisehong/rebase-repo.git
```

### 模拟开发分支的多次提交

新建一个squash分支, 即日常自己的开发分支, 模拟多次提交, 查看提交的日志

```
$ git checkout -b squash

$ for i in H e l l o , ' ' C h o n g c h o n g; do
	echo "$i" >>hello.txt
	git add hello.txt
	git commit -m "增加Hello，$i"
done
$  git log
commit e782719bcaa05a177ddd0566e31f808d032602da (HEAD -> squash)
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:06 2020 +0800

    增加Hello，g

commit ffa3f28eb1a771d81167c1bfbfe52b2cc54564e0
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:06 2020 +0800

    增加Hello，n

commit 7fc2daf990c837c33506ab56b251c01156d8aa2c
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:06 2020 +0800

    增加Hello，o

commit d862c714350af92548417bceeda0c7c7c9857162
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:05 2020 +0800

    增加Hello，h

commit c68317bdb7a734e4cb3dbdb73a83c05099ba5cfe
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:05 2020 +0800

    增加Hello，c

commit e240ef875d7a89f0043e0cf349953b589940fdd5
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:05 2020 +0800

    增加Hello，g

commit 2e50a7d72f409672af874b82c5f4b00e1b9f56b4
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:05 2020 +0800

    增加Hello，n

commit 8a4a3b86f01619d6b68cae1cb8abae5a14a58515
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:05 2020 +0800

    增加Hello，o

commit 0e3f4c8407581027265286769b57edfbbb9e5ee4
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:05 2020 +0800

    增加Hello，h

commit 3ca487fdc148c4b40163e00de1880548cffeb709
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:05 2020 +0800

    增加Hello，C

commit 912bd67bee15d5608958f4ccd19b76c1afffc210
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，

commit 5e8415bab1156ebdc82172aee6863c1b50fa4ed9
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，,

commit 4f60c257112de5509270ccc3116cb7a91602e987
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，o

commit 66a94f9130b186b65ac70ecc4ede93bc8aac1445
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，l

commit bf632988e5581742d53f81007891f9c793778920
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，l

commit 38b3aa83624df2f073b7489dcefda022fc324eec
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，e

commit cb7b1f8ec5da79a106fb1efd2d98b539c6d716f4
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，H

commit ddfd3c128a7fcbc59b6f9f7075b8d19d25273a45 (origin/master, master)
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:43:13 2020 +0800

    init
```

可以看到很多的提交日志. 合并多个提交, 使用rebase功能. 这里会进入vim模式. 将前面的提交都显示出来, 使用替换命令`2,$s/pick/squash`,然后`:wq`保存,合并多个提交. 

```
 $ git rebase -i master
pick 424c375 增加Hello，H
pick f1416a9 增加Hello，e
pick 4c37233 增加Hello，l
pick 7446b97 增加Hello，l
pick fb53788 增加Hello，o
pick e87bbee 增加Hello，,
pick d085a0e 增加Hello，
pick 830fdfc 增加Hello，C
pick 34e9c95 增加Hello，h
pick e654386 增加Hello，o
pick a17be3c 增加Hello，n
pick c932c0d 增加Hello，g
pick e1d4d22 增加Hello，c
pick 34d2fe8 增加Hello，h
pick 9ee2e6c 增加Hello，o
pick 327c529 增加Hello，n
pick 6a6f4c0 增加Hello，g

# Rebase 252ce30..6a6f4c0 onto 327c529 (17 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
```

保存后, 会有一个提交信息的整合, 然后也会进入vim模式,这里 编辑你想要的commit信息即可, 然后`:wq`保存. 

```
# This is a combination of 17 commits.
# This is the 1st commit message:

增加Hello，World , Test Rebase

[detached HEAD 5810da3] 增加Hello，World , Test Rebase
 Date: Tue Jun 30 10:07:04 2020 +0800
 1 file changed, 17 insertions(+)
Successfully rebased and updated refs/heads/squash.
```

### rebase至master分支

到了这一步, 基本的`rebase`合并已经做完. 将自己的squash分支rebase到master分支即可, `git log`果然没有多次提交记录了. 

```
$ git checkout master
$ git rebase squash
$ git branch -D squash 
$ git log
commit 5810da36cce6d0558c11daa80f9f7f5c220c6d70 (HEAD -> master)
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 10:07:04 2020 +0800

    增加Hello，World , Test Rebase
    
commit 252ce30404ba0ff2a071fa7cd27d0781365eafae (origin/master)
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:47:04 2020 +0800

    增加Hello，Hello,World

    
commit ddfd3c128a7fcbc59b6f9f7075b8d19d25273a45
Author: louisehong <louisehong4168@gmail.com>
Date:   Tue Jun 30 09:43:13 2020 +0800

    init
$ git push origin master
```

