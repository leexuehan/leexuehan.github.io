---
title: mac+python selenium 环境搭建
date: 2018-03-21 21:33:36
tags: Crawler
categories: 技术
---

本文主要讲搭建python selenium 环境时自己走过的一些坑。
<!--more-->

## 安装python selenium

安装python，本文从略。

安装selenium，可以使用 pip 命令安装，也可以直接在 IDE 环境下安装。

## 下载以及安装 ChromeDriver

chromedriver 和 Chrome 浏览器的版本映射表，参见链接：

http://blog.csdn.net/huilan_same/article/details/51896672

chromedriver 对应驱动下载，专门有一个镜像地址，里面有各种版本的Chrome 驱动：

http://npm.taobao.org/mirrors/chromedriver

下载下来之后解压，之后将 chromedrvier 文件拷贝到 /usr/local/bin 目录下即可。

## 运行测试程序

		driver = webdriver.Chrome()
	    driver.get("http://www.baidu.com")
	    driver.find_element_by_id('kw').send_keys('gggg')
	    driver.find_element_by_id('su').click()

## 错误排查

安装好之后如果报以下奇奇怪怪错误：

		unknown error: call function result missing 'value'

或者其他类似的错误的话，一定是版本不对应，参见前面步骤中的版本映射关系。


