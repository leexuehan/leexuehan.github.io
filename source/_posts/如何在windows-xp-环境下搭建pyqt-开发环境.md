---
title: 如何在windows xp 环境下搭建pyqt 开发环境
date: 2017-08-11 21:49:20
tags: pyqt
categories: 技术
---
前面的文章介绍了如何在 mac 环境下搭建pyqt开发环境，但是实际过程中，客户需要在 windows xp环境下使用，所以需要将程序打包成 exe 文件发布。在 mac 环境下目前查不到如何打包成 exe 环境的资料，于是只好在 windows 环境下重新搭建一套环境，来开发了。这篇文章主要讲如何搭建好开发环境，下一篇文章讲如何打包成 exe 文件发布。

<!--more-->

## 搭建 python 环境

### 安装 python

下载 python 安装包。由于python从3.5版本就开始不支持 windows xp 版本了，所以最高版本只能用 python 3.4.x 系列的（不想用 python2.7了。）官网下载速度慢的可以移步这里：

http://pan.baidu.com/s/1slQAxad

加入环境变量 python 的路径 和 python 下面的 Script 的路径（其他文章均有介绍，这里略之）

打开“运行”，输入 cmd，打开 windows 的 DOS 窗口，运行 python 命令，如果没有出现“没有该命令”的字样，则说明python已经正确安装。

### 安装 pip

最简单的命令：

python -m pip install -U pip setuptools

执行成功之后，可以看到已经成功安装了 pip。

### 安装 PyQt5

个人喜欢用PyQt5。执行命令：

pip install --update PyQt5

## IDE开发环境

个人喜欢用 PyCharm，所以需要在官网下载该软件，下载 Community 社区版即可。

之后，可以需要将两个工具集成到软件中：QtDesigner 和 pyrcc5.exe

前一个用于以拖控件的方式设计界面，后者用于编译 ui 文件成为 py 文件。

设置方式：

Ctrl+Alt+S

“Tools”-“External Tools”

点击+号

第一步找到 QtDesigner.exe 的路径：pyhton 路径--Libs--site-packages--PyQt5

第二步找到 pyrcc5.exe 的路径（同样在 PyQt5文件夹下）

配置方式参见：http://blog.csdn.net/Jmilk/article/details/50730095

I
