---
title: mac下利用hexo搭建博客(三)
date: 2016-06-09 15:40:21
tags: hexo
categories: 技术
---

继续前面的工作，对博客进行功能的拓展。

## 添加标签云

在网上搜了老半天，不同主题，实现方式还是不一样的，next主题实现的方式其实很简单。

<!--more-->

只需要在hexo根目录下执行下面的命令：

		hexo new page "tags"
		
这个时候，就可以看到跟发表文章的 _posts 同级目录已经生成了 tags 文件夹，里面已经有了一个 index.md 文件。

编辑此文件，在开头增加下面两项：

	type: "tags"
	comments: false
	
当然，你还需要检查在主题配置文件的menu选项下面是否有 tags 选项，如下所示：

	menu:
  		home: /
  		techs: categories/技术
  		readings: categories/读书
  		chating: categories/闲聊
  		archives: /archives
  		about: categories/关于
  		tags: /tags
  	
  		
配置好后，更新配置即可。

## 添加友情链接

更改主题配置文件，找到下面的地方，将social前面的#取消，然后加上链接即可。链接的形式如下：

			# Social Links
			# Key is the link label showing to end users.
			# Value is the target link (E.g. GitHub: https://github.com/iissnan)
			social:
  			#LinkLabel: Link
  				GitHub: https://github.com/leexuehan（示例，下同）
  				Weibo:  http://weibo.com/leexuehan
  				Zhihu:  https://www.zhihu.com/people/leexuehan
  				Douban: https://www.douban.com/people/leexuehan/



## 添加404页面

在 source 文件夹下面，创建一个名为 404.html 的文件，将下面这段代码贴进去，更新配置即可。

```

		<!DOCTYPE html>
			<html>
				<head>
				</head>
				<body>
					<script type="text/javascript" src="http://www.qq.com/404/					search_children.js" charset="utf-8" homePageUrl="http://					www.tianrunlin.com" homePageName="回到主页"></script>
				</body>
			</html>
			
```

注意：上面的代码中的homePageUrl,是返回自己博客的链接，需要换成自己的博客地址，否则点击回到主页，就返回到别人的博客了。

这段代码是腾讯做的公益，每次页面找不到的时候，就会出现一个被拐小孩的资料，也算做一件善事了。

## 设置博客的图床

其实，也正因为这个原因，才开始大费周章地搭建自己的博客。之前一直都用的是github的issues写博客，也挺方便的，最近突然图片很难上传到github的图库了，这让我很不爽，于是开始学着用hexo自己搭建博客。

我的博客图床用的是七牛，体验了下，还是不错的，经过认证有 10G 的免费存储空间。

登陆 [七牛](https://portal.qiniu.com)，需要进行一系列注册。话说七牛认证还是挺麻烦的，人工认证需要审核三天，支付宝认证又不敢授权，所以只好采用前者了。

之后需要自己建立一个工作空间，在内容管理选项中，点击上传图片（具体步骤，网上可以很轻易搜到，这里不赘述）。

如果博客中配图的话，需要先上传图片到七牛云存储中，之后选中图片复制外链地址，插入博客中即可。


### 更换网址小图标

这里的小图标就是打开网页后，选项卡上的小图标。

搭好图床，这就很好办了，只需要将小图标下载下来。

这里推荐一个logo网站：[iconfont](http://www.iconfont.cn)。这里面的资源非常多，可以方便地搜索到。

之后需要改变主题配置文件即可。

	# Put your favicon.ico into `hexo-site/source/` directory.
	favicon: http://o8ahjnaal.bkt.clouddn.com/favicon.ico（这里就是小图标的外链地址）
	
更新配置即可。

## 安装第三方插件

hexo安装第三方插件极为方便。[hexo官网](https://hexo.io/plugins/)，这里有很多插件，安装这些插件，只需要执行简单的命令： 

		npm install [插件名]
		
卸载插件

		npm uninstall [插件名]




