---
layout: post
title: 用github和hexo建立第一个blog
categories: hexo
description: some word here
keywords: keyword1, keyword2
---
# 在GitHub中创建自己的blog地址

1. 点击 + new repository按钮，然后在Repository name中输入以自己的Github用户名为开头，github.io为结尾的Repository name，以我的为例：liutaohua.github.io,然后点击Create repository.

1. 在新创建的仓库中创建一个文件，命名为index.html

1. 在浏览器中输入自己的GitHub仓库名称，以我的为例：http://liutaohua.github.io

# 安装hexo

> 浏览hexo官方网站，查看基本的安装方式，因为博客的更新速度有可能不会跟上软件的更新速度，安装方式可能会改变，推荐使用官方安装方式。

1. 安装nodejs<br/>
&emsp;&emsp;我安装时候hexo依赖的node版本为0.12.＊，我选择的版本是0.12.7，下载地址为 https://nodejs.org/dist/v0.12.7/

1. 安装hexo,并初始化博客
```
$ npm install hexo-cli -g
$ hexo init blog
$ cd blog
$ npm install
$ hexo server
```
1. 在浏览器中查看：localhost:4000

# 将生成的博客push到GitHub

1. clone 我们在第一步中生成的repository,以我的为例:
```
$ git clone https://github.com/liutaohua/liutaohua.github.io
```
1. 在我们刚才生成的blog文件夹中，执行:
```
$ hexo g
```
1. 将生成的public文件夹中的内容拷贝到我们刚clone下来的repository文件夹中，然后:
```
$ git add ./*
$ git commit -m "init blog"
$ git push origin master
```
稍等片刻，然后打开我们的blog地址，享受第一个blog带来的欣喜吧！