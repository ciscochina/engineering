---
layout: post
title:  "git cherry-pick for backport"
date:   2018-03-30
author: Peng Xiao
categories: linux
---

本文主要介绍Git cherry-pick的用法。首先我们看看什么是 `backport` 。

假如我们当前在master分支进行开发，开发到一定程度，我们release了1.0， 这时候我们一般会为其创建一个 `stable` 的分支，比如叫 `stable/1.0`

这时候用户在测试或者使用1.0版本代码的时候发现一个bug，这个bug可能在 stable/1.0分支和master分支都存在，这时候我们想在master和stable/1.0上都修复这个bug怎么办？

我们肯定不能在master分支上修复完，以后把master分支merge到stable/1.0, 这不被允许，因为master分支有太多新的feature，并不稳定, 而且会导致版本紊乱。

有一种操作就是我们把代码分别在master和stable/1.0上进行修改，但是这样非常不方便。

这时候我们可以使用cherry-pick, 把在master分支提交的修复bug的commit 提交到其他旧的分支用于修复bug，这就叫 `backport` ,  怎么用呢，给大家举例



#### 准备代码库和分支

首先我们先准备一个代码库.

```
➜  tmp mkdir cherry-pick-demo
➜  tmp
➜  tmp cd cherry-pick-demo
➜  cherry-pick-demo git init
Initialized empty Git repository in /Users/penxiao/tmp/cherry-pick-demo/.git/
➜  cherry-pick-demo git:(master) echo "this is a test file" > test.txt
➜  cherry-pick-demo git:(master) ✗ ls
test.txt
➜  cherry-pick-demo git:(master) ✗ git add test.txt
➜  cherry-pick-demo git:(master) ✗ git commit -m "init commit"
[master (root-commit) 39ee3a7] init commit
 1 file changed, 1 insertion(+)
 create mode 100644 test.txt
➜  cherry-pick-demo git:(master)
```

然后我们基于当前master创建一个分支 stable/1.0 , 然后回到master

```
➜  cherry-pick-demo git:(master) git checkout  -b stable/1.0
Switched to a new branch 'stable/1.0'
➜  cherry-pick-demo git:(stable/1.0) git checkout master
Switched to branch 'master'
➜  cherry-pick-demo git:(master)
```

#### 发现bug并在master分支fix

这时候，用户使用stable/1.0发现一个bug，这个bug也在master分支上存在。我们在master分支做一个bugfix的提交


```
➜  cherry-pick-demo git:(master) ✗ echo "this is a test file" > test.txt
➜  cherry-pick-demo git:(master)
➜  cherry-pick-demo git:(master) echo "this is a bug fix" >> test.txt
➜  cherry-pick-demo git:(master) ✗ more test.txt
this is a test file
this is a bug fix
➜  cherry-pick-demo git:(master) ✗ git add test.txt
➜  cherry-pick-demo git:(master) ✗ git commit -m "this is a bug fix commit"
[master 095e9ec] this is a bug fix commit
 1 file changed, 1 insertion(+)
➜  cherry-pick-demo git:(master) git log
➜  cherry-pick-demo git:(master)
```

git log我们能看到这次bug fix的commit hash

```
commit 095e9ec7633e841cabc9c9c9f8b22687ba324da4 (HEAD -> master)
Author: Peng Xiao <xiaoquwl@gmail.com>
Date:   Fri Mar 30 13:28:23 2018 +0800

    this is a bug fix commit

commit 39ee3a7fef1726b3a7bbc9a16a642d290922327a (stable/1.0)
Author: Peng Xiao <xiaoquwl@gmail.com>
Date:   Fri Mar 30 13:20:05 2018 +0800

    init commit
```

#### 把bug fix的commit backport到其他分支

这时候如果我们想把在master分支提交的bugfix commit也弄到 stable/1.0分支上，可以这样操作。

切换到stable/1.0分支，然后使用cherry-pick把bugfix的提交重新提交到 stable/1.0分支

```
➜  cherry-pick-demo git:(master) git checkout stable/1.0
Switched to branch 'stable/1.0'
➜  cherry-pick-demo git:(stable/1.0) more test.txt
this is a test file
➜  cherry-pick-demo git:(stable/1.0) git cherry-pick 095e9ec7633e841cabc9c9c9f8b22687ba324da4  # maste分支bugfix的commit ID
[stable/1.0 eb173ea] this is a bug fix commit
 Date: Fri Mar 30 13:28:23 2018 +0800
 1 file changed, 1 insertion(+)
➜  cherry-pick-demo git:(stable/1.0) more test.txt
this is a test file
this is a bug fix
➜  cherry-pick-demo git:(stable/1.0)
```

当然，这是理想情况，`git cherry-pick` 这句运行的时候也可能会出现冲突，这时候我们和其它情况一样，手工处理下冲突，然后再commit就行了。

好的，这就是所谓的backport了。