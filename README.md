## Welcome to Cisco China GDC Engineering Blog

This blog is maintained by Cisco China Dalian GDC team.

### 如何提交一篇新的博客

#### fork repository

把本代码库fork到自己的GitHub账号

#### create a new post

把源码clone到本地后，在 `_posts`文件夹里创建一个新的markdown格式文件，命名规则按照 `yy-mm-dd-name.md`.比如 `2012-12-02-linux-bridge.md`.

#### format header

每一个markdown的post文件，在正文前面必须包括下面的内容（也可参考`_posts`文件夹里其它文件）：

```
---
layout: post
title:  "xxx xxx xxxx xxxx xxxx"
date:   2017-07-08 08:43:59
author: Peng Xiao
categories: database
---
```

`title` 标题， `date` 创建日期， `author` 作者名称，`categories` 分类。

目前可用的分类有`network`, `linux`, `openstack`, `python`. 如果有新的分类需求，请开issue。

接下来是blog的正文，请用标准的markdown格式编写。


#### preview locally

本地预览，请参考https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/，主要参考requirement和step2，step4.


#### commit, push and merge request

本地预览没有问题，请commit，push代码到GitHub，然后创建merge request。

#### review and merge

管理员做review，最后merge。
