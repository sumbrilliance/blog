---
title: 使用 hugo 搭建github page 博客
description: 2019
date: 2019-01-09T10:58:08-04:00
coverImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/home-bg-o.jpg"
thumbnailImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/home-bg-o.jpg"
thumbnailImagePosition: "left"
categories:
- 记录
tags: 
- 其他
---

之前使用的 jekyll 搭建的 github pages 博客，最近重新整理博客，决意迁移到 hugo。

本文介绍了这个博客迁移的过程。

<!--more-->

<!-- toc -->


# 1.生成静态文件

执行命令 

> hugo -t hugo-tranquilpeak-theme

![image-20190110001535959](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/image-20190110001535959.png)

生成静态文件到 public，如果所示：![site-public](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/site-public.png)

# 2.将生成的静态文件，push到博客仓库中

先进入 submodules，即 git 的子模块

> cd public

因为之前我们给 blog 仓库添加子模块，blog的public目录关联的是 sumbrilliance.github.io.git 这个仓库地址，所以我们进入到public后，进行的操作其实是针对 sumbrilliance.github.io 这个仓库的。现在我们进到 public 目录，push到 关联仓库中。

![site-push1](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/site-push1.png)

> git commit -m 'ref to blog repository'
>
> git push

成功之后，https://github.com/sumbrilliance/sumbrilliance.github.io 这个仓库，将会看到提交记录，

访问 https://sumbrilliance.github.io 即看到了博客页面



**一些注意事项**

• markdown 文件头部 "date" 字段，即文件发表时间，不能早于当前时间，否则不会显示

• title第一个字符不能为.否则会生成点开头的文件夹。我们都知道，点开头的文件文件夹是属于隐藏文件。所以上传github后，点击文章会报404。

