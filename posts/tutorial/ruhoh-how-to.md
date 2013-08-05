---
title: '使用Ruhoh和Github搭建博客'
date: '2013-08-05'
description: '使用Ruhoh生成静态网页，并部署到Github Pages'
tags:
- ruhoh
- github
- blog
categories:
- tutorial
---

#### 缘由
前一段时间我还在用golang＋Google App Engine[实现博客系统][goblog]，
一方面确实有需求，另一方面想学习golang，但是发觉太耗时且做得不够好，
于是Google并多次比较选择了[Ruhoh][ruhoh]。

#### 准备
关于[Github Pages][github-pages]；
关于[Markdown][markdown-tutorial]以及[Markdown Wiki][makrdown-wiki]；
以及[Ruhuh文档][ruhoh-doc]。

#### 步骤
* 安装：

		$ git clone 'git://github.com/ruhoh/blog.git' 'blog' #获取代码
		$ cd blog 
		$ bundle install #安装以来，会安装ruhoh，建议改成淘宝的源
		# Gemfile改成 source 'http://ruby.taobao.org/'
		$ bundle exec rackup -p 9292 #本地预览
这样就有一个基本的博客系统了。
可以在 http://localhost:9292/看到效果，
中文字体偏小，bootstrap可以定制，字体改成16px就行舒服多了。
这样的话，下面tags和categories的图标就没了。
* 发表：

		$ cd posts #很重要，不然compile了没用，文档就没说这个。
		$ ruhoh tutorial new "ruhoh how to"
		# 在posts目录中建一个“分类”为 tutorial的“名”为“ruhoh how to”的博文
		# 这个“分类”是给自己管理方便用的不是categories
适当修改一下.md就可以了，例子:

	`---`

	`title: '使用Ruhoh和Github搭建博客'`
	`....博客的标题.....`

	`date: '2013-08-05'`
	`....日期，会用来排序.....`
	
	`description: '使用Ruhoh生成静态网页，并部署到Github Pages'`
	`....描述，貌似不会render出来，给自己看的？.....`

	`tags:`
	`....标签，会自动合并统计到标签页面中.....`

	`- ruhoh`
	
	`- github`

	`- blog`

	`categories:`
	`....分类，会自动合并统计到分类页面中.....`

	`- tutorial`

	`---`

	`[该处就是是现在你看到的正文]`
* 其他

	+ 推荐使用[Markdown Mode][markdown-mode]。
	
	+ 我的博客代码（都是markdown）地址[https://github.com/henglinli/blog][henglinli-blog]。

***
[goblog]: https://github.com/henglinli/goblog/ "基于golang和GAE的博客系统"
[ruhoh]: http://ruhoh.com/ "ruhoh主页"
[github-pages]: http://pages.github.com/ "关于Github Pages"
[markdown-tutorial]: http://wowubuntu.com/markdown/ "mardown入门"
[makrdown-wiki]: http://zh.wikipedia.org/zh-cn/Markdown/ "markdown wiki"
[ruhoh-doc]: http://ruhoh.com/docs/2/ "ruhoh文档"

[markdown-mode]: http://jblevins.org/projects/markdown-mode/‎ "markdown mode"
[henglinli-blog]: https://github.com/henglinli/blog  "henglinli/blog" 
