---
title: 如何使用HEXO创建博客
date: 2017-12-12 23:37:03
tags: [Hexo]
categories: [Hexo]
---

### 一、快速开始

> Create a new post

``` bash
$ hexo new "My New Post"
```

> Run server

``` bash
$ hexo server
```

> Generate static files

``` bash
$ hexo generate
```

> Deploy to remote sites

``` bash
$ hexo deploy
```
<!-- more --> 

More Info:

- [Writing](https://hexo.io/docs/writing.html)
- [Server](https://hexo.io/docs/server.html)
- [Generating](https://hexo.io/docs/generating.html)
- [Deployment](https://hexo.io/docs/deployment.html)


### 二、设置自动滚动

``` bash
# Automatically saving scroll position on each post/page in cookies.
save_scroll: false
```

### 三、设置多个tag或category

> 伪JavaScript数组写法

> tag: `[Hexo,HTML,JavaScript]`
> categories: `[Hexo,HTML]`

### 三、关于Hexo next主题如何在首页隐藏指定的文章

<!-- http://forwardkth.github.io/2016/05/08/next-theme-post-visibility/ -->

> 修改next主题文件夹下的layout中的index.swig文件，\themes\next\layout\index.swig

![图示](http://ww1.sinaimg.cn/large/74505a4cgw1f3onp6eculj20ds0d70vb.jpg)

> 在新的post中添加visible字段来控制是否首页显示


> title: 关于Hexo next主题如何在首页隐藏指定的文章
> visible: hide 这里如果加上hide则该文章就不会在文章首页显示，如果留空则表示默认显示
