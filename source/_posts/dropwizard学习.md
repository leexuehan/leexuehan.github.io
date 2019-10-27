---
title: dropwizard学习（一)
date: 2016-06-07 22:49:42
tags: dropwizard
categories: 技术
---
## 前面的废话
不知道从什么时候开始，自己觉得之前的方法有些效率低，于是尝试改变一下之前不好的学习习惯。自己总结科学的学习无非就是遵从人的理解规律和记忆规律。

看过金字塔原理的人都知道，人对于梳理和归类好的东西有天然的好感，即更容易记忆和理解。所以自己不允许接触到一个新东西之后没有自己的理解和梳理。所谓的理解和梳理最好的呈现方式无非就是：归类和画图。

个人觉得理解任何东西，都要从架构上去把握，然后再去抠细节。

<!--more-->

这几天正好用到了 dropwizard，所以想把自己对于这个轻量级框架的理解梳理一下。

## dropwizard是什么

dropwizard按照官方给出的定义如下：

> Dropwizard is a Java framework for developing ops-friendly, high-performance, RESTful web services.

翻译成中文就是：Dropwizard是一个用来开发友好、高性能 RESTful服务的Java框架。

那么什么又是 RESTful 服务呢？

如果你去谷歌的话，会有好多特别专业的术语冒出来，让人晕头转向。我看到一个最通俗的说法就是：

>看url就知道想要什么

>看http method就知道要干什么

>看http status code就知道结果如何

可能这个说法不是很准确，但是至少给我们形成了一个大致的概念，我想对于初学者应该够了。

## dropwizard架构

dropwizard的主要模块如下：

![](http://o8ahjnaal.bkt.clouddn.com/dropwizard.PNG)

可以看到，dropwizard里面继承了一个jetty服务器，用来处理http请求。利用jersy处理web应用，jackson映射对象。此外，还有其他模块，但是核心的就这三块。

## 怎么使用dropwizard编写restful程序

我画了一张程序组织图：

![](http://o8ahjnaal.bkt.clouddn.com/%E7%A8%8B%E5%BA%8F%E7%BB%84%E7%BB%87%E7%BB%93%E6%9E%84.png)

比较清楚地描述出来dropwizard的程序应该怎么写。

具体的例子，可以在官方网站上搜到，这里就不作文抄公了。

我想在你了解了整个架构之后，就可以直接移步这里了。[官网链接](http://www.dropwizard.io/0.9.2/docs/getting-started.html).










