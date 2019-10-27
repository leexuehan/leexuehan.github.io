---
title: mac使用hexo搭建博客
date: 2016-06-08 21:45:19
tags:
---


## 环境准备

### git

在github上注册账号，新建repository（过程不赘述，相关资料很多）；

注意：repository的名字必须是 [username].github.io

### node.js

在官网下载安装即可（过程很简单）。

<!--more-->

## 安装HEXO

### 博客根目录

建立一个目录，可以叫 MyBlogs.

		mkdir MyBlogs
		cd MyBlogs
		
这个目录就是你要发表博客的地方。

然后执行下面命令，安装hexo。

		sudo npm install -g hexo

然后执行命令进行初始化。

		hexo init
		
如果没有什么问题的话，hexo已经安装成功。

		hexo generate
	
生成静态页面。

这个时候启动本地服务，测试hexo是否安装成功。

		hexo server
	
这个时候可以看到，服务已经启动。

在浏览器里面输入http://localhost:4000，测试是否出现博客页面，如果出现，说明hexo已经安装成功。

## 关联github

在上一步执行hexo g生成静态页面的时候，可以看到在MyBlogs目录下，已经有了_config.yml文件。我们可以通过编辑此文件来使此目录与github进行关联。

		vim  _config.yml

一直到文件的结尾处有

		# Deployment
		## Docs: https://hexo.io/docs/deployment.html
		deploy:
  			type: git（需要自己写）
  			repository: https://github.com/leexuehan/leexuehan.github.io.git（必须）
  			branch: master（必须）
  			
需要改变的就是上面注明的地方。

然后执行命令

		npm install hexo-deployer-git --save

对配置进行保存。

然后执行命令

		hexo deploy

可以将设置部署到博客中。

## 访问页面

		http://[username].github.io/
		
如果部署成功的话，就可以看到自己的博客了。

## 可能出现的错误

部署之后，打开页面为404，可能是以下原因（按优先级排查）：

#### 部署过程中有问题，检查yml文件后面的冒号后面是否加了空格（必须有空格）。

这个地方有一个坑，让我排查了好久。这里如果上面的yml文件设置得有问题的话，终端不会报错的。（也是被unix的没有消息就是好消息误导了半天,让自己误以为部署正确了）。

#### 文件生效问题

每次修改yml文件或者其他文件之后，一定要进行下面三步骤：

		hexo clean
		hexo generate
		hexo deploy

#### yml文件配置问题

	hexo version 

查看版本，如果是3.0以上的话，type后面为github可能会有问题，建议改为git。

#### 延时问题

有的人说，部署之后，半小时才能访问，反正我没遇到过，我的部署之后都是秒速访问。


其他错误暂时没有碰到过，欢迎补充！




