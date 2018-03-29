---
title: HEXO切换电脑重新部署
date: 2017-12-19 23:33:26
tags: [Setup]
categories: [Setup]
---

### 文件夹拷贝

> 将原来的文件拷贝到新电脑中，但是要注意哪些文件是必须的，哪些文件是可以删除的。
1. 讨论下哪些文件是必须拷贝的：首先是之前自己修改的文件，像站点配置**_config.yml**，**theme**文件夹里面的主题，以及**source**里面自己写的博客文件，这些肯定要拷贝的。除此之外，还有三个文件需要有，就是**scaffolds**文件夹（文章的模板）、**package.json**（说明使用哪些包）和**.gitignore**（限定在提交的时候哪些文件可以忽略）。其实，这三个文件不是我们修改的，所以即使丢失了，也没有关系，我们可以建立一个新的文件夹，然后在里面执行hexo init，就会生成这三个文件，我们只需要将它们拷贝过来使用即可。
**总结**:**_config.yml**，**theme/**，**source/**，**scaffolds/**，**package.json**，**.gitignore**，是需要拷贝的。
2. 再讨论下哪些文件是不必拷贝的，或者说可以删除的：首先是**.git**文件，无论是在站点根目录下，还是主题目录下的**.git**文件，都可以删掉。然后是文件夹**node_modules**（在用**npm install**会重新生成），**public**（这个在用**hexo g**时会重新生成），**.deploy_git**文件夹（在使用hexo d时也会重新生成），**db.json**文件。其实上面这些文件也就是是**.gitignore**文件里面记载的可以忽略的内容。
**总结**：**.git/**，**node_modules/**，**public/**，**.deploy_git/**，**db.json**文件需要删除。

<!-- more --> 

### 模块安装

> 在新拷贝的文件夹里，右键选择**git bash**，使用**npm install**命令，进行模块安装。
> *这里不要使用**hexo init**初始化*，因为有的文件我们已经拷贝生成过来了，所以不必用hexo init去整体初始化，如果不慎在此时用了hexo init，则站点的配置文件_config.yml里面内容会被清空使用默认值，所以这一步一定要慎重，不要用hexo init。

### 安装一些必要组件

> 1. 为了使用hexo d来部署到git上，需要安装:
>
> ```
npm install hexo-deployer-git --save```
> 
> 2. 为了建立RSS订阅，需要安装:
> 
> ```
npm install hexo-generator-feed --save```
> 
> 3. 为了建立站点地图，需要安装:
> 
> ```
npm install hexo-generator-sitemap --save```
> 
> * 至此环境在环境已经具备的情况下，使用指令部署到Github:
>
> ```
hexo generate
hexo deploy```