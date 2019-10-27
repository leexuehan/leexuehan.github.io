---
title: 如何打包pyqt程序为exe 文件
date: 2017-08-11 22:21:24
tags: python
categories: 技术
---
本篇主要讲如何将写好的pyqt程序打包成为 exe 文件。

<!--more-->

## pyinstaller 的安装

pip install --update pyinstaller

## 执行打包命令

pyinstaller --paths [PyQt5的路径] -F -w --onefiel [主入口py文件的路径]

参考文章：http://blog.csdn.net/pipisorry/article/details/50620122

## 可能会遇到的问题

### Fail to execute script

一般是由于程序本身的bug导致的，比如配置文件没有放到指定路径下：注意打包之后的路径和在IDE中运行的路径会有差别

### 无法找到程序入口


一般是由于系统没有更新导致，所以需要安装更新包：

可以搜索 SP3 下载补丁包进行更新，也可以从网盘下载：

http://pan.baidu.com/s/1i5JziPf

更新后重启，重新打包即可。