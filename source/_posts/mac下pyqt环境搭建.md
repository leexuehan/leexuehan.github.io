---
title: mac下pyqt环境搭建
date: 2016-09-11 17:56:35
tags: pyqt
categories: 技术
---
最近给朋友解决一个关于 excel 的问题时，出于方便易用考虑使用了 python。

为了更加易用，想做成界面的形式，于是在使用tkinter做了第一版之后，觉得界面使用极其不便，且api文档或者资料很少，布局尤为不便，于是选择了 PyQt 作为界面的组件。

<!--more-->

但是搭建PyQt环境时还是遇到了不少问题，在此记录如下，避免其他人走弯路。

搭建环境总共分为三步。

### 安装python和 xcode

这方面网上有大量的文章，在此不赘述了。

### 安装Qt

网址： http://qt-project.org/downloads
也可以在文后附加的网盘链接里下载。

我的mac是 10.11.2 系统的，之前装了一个 Qt5.0版本的，在make的时候回报错不支持该版本。于是在官网上下载了最新的一版，目前是 5.7 的。

下载好根据提示一步一步安装即可。

### 下载 sip

官网链接：http://www.riverbankcomputing.co.uk/software/sip/download

安装步骤：

		tar xvf sip-4.17.tar.gz
		cd sip-4.17
		python3.5 configure.py -d /Library/Frameworks/		Python.framework/Versions/3.5/lib/python3.5/site-		packages
		make
		sudo make install
		
我的电脑上 python装的是 3.5 版本的。

这里需要注意，会遇到这样的错误：


		cp -f sip /System/Library/Frameworks/Python.framework/Versions/2.7/bin/sip
		cp: /System/Library/Frameworks/Python.framework/Versions/2.7/bin/sip: 		Operation not permitted
		make[1]: *** [install] Error 1
		make: *** [install] Error 2
		
解决问题参见链接  [解决方法](http://www.2cto.com/kf/201604/498456.html)

		重启系统
		按住 Command + R 进入 Recoverary 模式
		点击 实用工具 > 终端
		输入 csrutil disable
		重启系统
		
### 安装 PyQt5

官网下载地址： https://riverbankcomputing.com/software/pyqt/download5

编译安装步骤：

		tar xvf PyQt-gpl-5.5.1.tar.gz（你下载的版本号）
		cd PyQt-gpl-5.5.1 
		python3.5 configure.py --qmake /Users/lixuehan/Qt5.7.0/5.7/clang_64/bin/		qmake -disable=QtPositioning -d /Library/Frameworks/Python.framework/		Versions/3.5/lib/python3.5/site-packages
		make
		sudo make install
		

注意：中间的qmake在你的第一步 Qt 的安装路径里。所以在实际运行时，需要替换成你的安装路径。


如果在执行 python3.5 XXXX 时，没有问题的话，你就成功一半了。这个时候，执行make命令，之后你需要喝一杯茶或者抽一根烟，因为接下来会度过一段漫长的等待。

### 检验安装成功

在终端中输入

		python3.5 -c "import PyQt5"
		
如果没有报错的话，则说明安装正常，如果你还不放心的话，可以去你的PyQt安装包里面找一个qtdemo.py的文件执行，如果弹出一个软件界面，则说明安装完成。


### 软件包网盘地址

https://pan.baidu.com/s/1gfqeTu7

### 参考资料

1.http://blog.chinaunix.net/uid-26783235-id-4418059.html

2.http://blog.csdn.net/djstavaV/article/details/50218157



	






