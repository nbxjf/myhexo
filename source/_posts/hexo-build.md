---
title: 使用hexo + github 搭建博客
date: 2016-05-28 13:00:49
categories: hexo
tags:
	- hexo
---
# 前言
很久之前就想搭个博客了，但是由于懒，也苦于找不到很有价值书写的东西，拖到了今天。回头再看，其实整个过程很简单，只要拥有很简单的基础，就可以很快的完成博客的搭建。
<!--more-->

# 安装
### git
* 下载并安装git,用于将blog部署到github上
* 下载地址：[git](https://git-scm.com/download/)

### node.js

* node用于生成静态页面，建议下载最新版本，我用的6.2版本的
* 下载地址： [node](https://nodejs.org/dist/v6.2.0/node-v6.2.0-x64.msi)
### 安装hexo

* 先介绍几个常用命令

		$ hexo g #完整命令为hexo generate,用于生成静态文件
		$ hexo s #完整命令为hexo server,用于启动服务器，主要用来本地预览
		$ hexo d #完整命令为hexo deploy,用于将本地文件发布到github上
		$ hexo n #完整命令为hexo new,用于新建一篇文章
* 在任意地方添加命令，

		$ npm install-g hexo

* 然后在博客目录下初始化hexo

		$ hexo init
* 在根目录下安装依赖包

		$ npm install
* 然后运行

		$ hexo g
		$ hexo s
	* 运行结果
	
			INFO  Start processing
			INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
	* 但是我打开http://localhost:4000/，并没有任何显示，我就猜是端口被占用了，换个端口
	* 
			$ hexo s -p 5000
	* 成功

### 部署到github上

* 注册github账号，很简单
* 创建仓库
![](http://i.imgur.com/cP2CRtu.png)
	* 这个名字的格式必须为youname.github.io
* github账号以及仓库准备好了，此时，编辑根目录下_config.yml文件，修改最下方代码

		deploy:
  			type: git
 			repository: https://github.com/nbxjf/nbxjf.github.io.git
  			branch: master
	* 其中repository为你的repository地址
* 配置好后，将博客部署到github上去，建议配置SSH，我第一次是没有配置的，后面才配置

		$ hexo g
		$ hexo d
* 此时，博客就已经搭建好了，访问yourname.github.io就可以访问你的博客了