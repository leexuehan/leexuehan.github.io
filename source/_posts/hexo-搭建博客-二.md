---
title: mac 环境下 hexo 搭建博客(二)
date: 2016-06-09 09:37:51
tags: hexo
categories: 技术
---
书接上回。

## 如何更换主题

更换主题时，首先需要下载主题。执行下面命令：

		cd themes
		git clone git@github.com:leexuehan/leexuehan.github.io.git(仓库地址)
这个时候可以看到themes文件夹的里面已经有该主题的名字了。

<!--more-->

接下来，需要更改 _config.yml**站点配置文件**来更新主题。

> 需要说明的是：在hexo你可发现两个这样的文件，一个叫**站点配置文件**，是用来更改整个博客的全局设置的，包括使用的主题、包括使用的语言等等，在hexo的根目录下；另一个叫**主题配置文件**，是用来配置主题里面具体的细节的，位置在 themes里面主题文件夹里。

		# Extensions
		## Plugins: https://hexo.io/plugins/
		## Themes: https://hexo.io/themes/
		theme: hexo-theme-next # 名称与已经下载的主题保持一致
更新后，记得还是执行那三步：

		hexo clean
		hexo g
		hexo d
		
后面有改动后，同样要执行这样三步，不赘述。



## 如何添加文章分类

下面主要讲，如何在首页添加类似 主页 技术 读书 关于 的文章分类标签。

		# ---------------------------------------------------------------
		# Menu Settings
		# ---------------------------------------------------------------

		# When running the site in a subdirectory (e.g. domain.tld/blog), remove 		the leading slash (/archives -> archives)
		menu:
  			home: /
  			techs: categories/技术
  			readings: categories/读书
  			chating: categories/闲聊
  			archives: /archives
  			about: categories/关于
  			tags: categories/tags
  			#commonweal: /404.html

之后，需要在发表文章的时候需要进行相应的设置，下面进行详述。

然后在hexo目录下找到 scaffolds 目录，编辑里面的 post.md 文件，添加一栏 categories 即可。

此时，在主页上显示的还是 home techs readings chating ....

怎么显示中文呢？

就以 next主题为例，此时必须到该主题的 language 文件夹编辑 zh-Hans.yml 配置文件，来设置相应条目的对应关系。

			menu:
  				home: 主页
  				archives: 归档
  				techs: 技术
  				readings: 读书
  				chating: 闲聊
  				categories: 分类
  				tags: 标签
  				about: 关于
  				search: 搜索
  				
此时更新配置，就可以看到相应的中文显示了。

## 如何发表文章

发表文章执行下面命令

		hexo new ""(引号里面的是文件名也是文章的题目)。
此时可以看到在hexo的站点目录的 source/_posts 中，已经生成了一篇 markdown 格式的文章。
使用编辑器打开该文章呢，可以看到文章的开头已经自动生成了如下的样式：

		---
		title: hexo 搭建博客(二)
		date: 2016-06-09 09:37:51
		tags: hexo
		categories: 技术 # 默认是没有的，如果想发表到相应分类下面，加上这一条目即可。
		---
之后就是正文了。编辑好正文之后，更新配置即可（老三步）

## 如何出现“阅读更多”样式

在文章你想要显示摘要的地方，加上下面的类似分割符的东西：

			<!--more-->
在此分隔符后面的内容将会隐藏。

## 如何添加评论

一般hexo博客使用多说评论的人较多，这里就作为范例细说下。

多说是一个专业的第三方评论插件，导入还是方便的。

在添加插件之前需要对主题配置文件进行简单配置。

		# Duoshuo ShortName
		duoshuo_shortname:  # 在这里填写你的shortname，可以随便写

首先登陆 http://duoshuo.com/ 页面，点击“我要安装”， 之后会出现下面的页面让你填写。填写注意事项如下图：

![image](http://o8ahjnaal.bkt.clouddn.com/%E5%A4%9A%E8%AF%B4.png)

之后点击“创建”。

会自然生成一段代码，让你粘贴到合适的位置，此处你需要找到自己主题的 layout/_partials/ 目录下面，找到对应的 comments.xxx(格式不同主题可能不一样)，打开之后替换掉原来的代码即可。

之后更新配置即可。

>在部署的过程中可能会遇到下面的问题：

>1.虽然起作用了，但是在文章的末尾会出现代码的情况，这个时候，建议在获取代码那一栏，不选择“通用”，换成“Discuz”，即可。

>2.在部署的过程中，执行 hexo 老三步 的时候回出现异常，处理 .DS_Store 文件异常，这个时候其实不影响整个博客的部署，可以不管，但是对于有强迫症的我和你来说，每次都报错这是我们不能忍的，于是跟着我一起执行：
 
 		ls -al
 		rm -f -r .DS_Store
 		
> 哪个目录下报错，就去相应目录下将该文件删除（就这么简单粗暴）。

>3.更新配置后，可能不会马上起作用，这是因为系统处理会有时间延迟问题，等几秒钟即可。







