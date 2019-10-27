---
title: Tomcat 与 Servlet
tags:
  - Struts2
date: 2015-07-22 21:21:14
categories: 技术
---

这篇博客将以Tomcat为例讲一讲Servlet的原理

## Servlet容器

Servlet与Servlet容器的关系举个不恰当的例子就像枪和子弹的关系。而Servlet就是子弹，容器就是枪。子弹都有统一的标准，枪却各有各的不同。

所以Servlet只是个标准，但是容器却各有不同。比较流行的有Jetty，用于移动领域。但是最为熟悉的还是Tomcat。

本博客的所有实现的描述，都是基于Tomcat的。

先看一张Tomcat容器的模型图

![6-29-1](https://cloud.githubusercontent.com/assets/12984324/8826213/836bacd6-30b7-11e5-987b-f64ad9c96c39.png)

从图中可以看出，Tomcat容器分为四个等级：Container,Engine,Host,Context。
真正管理Servlet的容器是Context.一个Context对应一个Web工程。

## Tomcat容器的启动

### 大概过程
1.获取Tomcat实例

	Tomcat tomcat = getTomcatInstance();

	/**
	从Tomcat 7开始支持嵌入式功能，增加了一个启动类：
		org.apache.catalina.startup.Tomcat
	除此之外，还可以调用该对象的方法来增加和修改
	Tomcat的配置参数，如动态增加Context、Servlet*/

2.添加应用到容器中

	File appDir = new File(xxxxx);
	tomcat.addWebapp(xxxxxxxx);
	/**
		添加一个Web应用将会创建一个StandardContext容器，并且会
		给这个Context容器设置必要的参数，如url和path。最重要的
		一个配置是ContextConfig，这个类将会负责整个Web应用配置
		的解析工作。然后将这个容器加入父容器Host中。
		*/

3.启动Tomcat(核心)

	tomcat.start();
	/*
	Tomcat的启动逻辑是基于观察者模式设计的，所有的容器都会继承
	Lifecycle接口。它管理着容器的整个生命周期，所有的容器的修改
	和状态的改变都会由它去通知已经注册的观察者（Listener）。*/

Tomcat的启动过程较为复杂，关键点是**StandardContext的初始化**，
当Context初始化状态设为init时，添加到Context的容器的Listener，即ContextConfig将会被调用（ContextConfig继承了LifecycleListener接口，在调用Tomcat.addWebapp时被加入到StandardContext容器中，负责整个Web应用的配置文件的解析工作）。

ContextConfig将会完成对于每个Web应用项目的配置解析和Web应用的初始化工作：

		ContextConfig的init方法将会完成以下工作：
		
		- 创建用于解析XML配置文件的contextDigester对象。
		
		- 读取默认的context.xml文件，如果存在就解析它。
		
		- 读取默认的host配置文件，如果存在就解析它。
		
		- 读取默认的Context自身配置文件，如果存在就解析它。
		
		- 设置Context的Docbase

		ContextConfig的configureStart方法实现Web应用的初始化工作：主要是完成web.xml文件（这里不单单指应用的web.xml）
		的解析。之后会将Web.xml中对象的属性设置到Context容器中，
		包括创建Servlet对象、filter、Listener等。
		这里会将Servlet包装成Context容器中的StandardWrapper(因为StandardWrapper是容器的一部分，
		具有容器的特征，而Servlet是一个独立的开发标准，不应该耦合在Tomcat中)。
		从这里可以看到Context的重要性了：
		web.xml属性都解析到Context中，所以Context容器才是真正运行Servlet的Servlet容器。

## Servlet实例的创建

在上面的Tomcat启动过程中，已经完成了Servlet的解析工作，并且被包装成StandardWrapper添加在Context中，但是因为它没有被实例化，所以仍然不能为我们工作。下面主要讲Servlet是如何创建以及初始化的：

### 创建Servlet对象

如果Servlet的load-on-startup配置项大于0，则在Context容器的启动过程中就会被实实例化。在web.xml文件中定义了两个Servlet：

org.apache.catalina.DefaultServlet和 org.apache.jasper.servlet.JspServlet

创建Servlet实例的方法从Wrapper.loadServlet开始，此方法要获取servletClass然后将它交给InstanceManager去创建一个基于servletClass.class的对象。

### 初始化Servlet

初始化的方法在StandardWrapper的initServlet方法中。此方法调用Servlet的init()方法，同时把包装了StandardWrapper对象的StandardWrapperFacade作为ServletConfig传给Servlet。

如果该实例关联的是一个JSP文件，则初始化JspServlet，接下来会模拟一次简单的请求，调用这个JSP文件，来编译之，初始化之。

## Servlet的体系结构

Servlet的运行模式是一个典型的“握手型交互式模式”。

>所谓的“握手型交互式模式”，即两个模块为了交换数据通常都会准备一个交易场景，这个场景一直跟随这个交易指导这个交易完成为止。这个交易的场景的初始化就是根据这次交易对象指定的参数来定制的，这些指定参数通常是一些配置类。

交易场景由ServletContext来描述，定制参数集合由ServletConfig来描述。ServletRequest和ServletResponse就是要交互的具体对象。

##Servlet如何工作
用户通过url向服务器发起请求，服务器如何根据此url映射到正确的Servlet容器呢？

在Tomcat7中，这种映射工作由一个专门的类来完成，即org.apache.tomcat.util.http.mapper
此类保存了Tomcat的Container容器的所有子容器的信息。

那么Mapper类怎么会有容器的完整关系？

因为MapperListener类作为监听者把加到整个Container容器的每个子容器中，这样只要任何一个容器发生变化，MapperListener都会被通知到。相应的保存容器关系的MapperListener的mapper属性页会被修改。

>现在的Web应用很少直接将交互的页面全部用Servlet来实现，而是采用更加高效的MVC框架来实现。这些MVC框架的基本原理就是将所有的请求都映射到一个Servlet，然后去实现Service方法，这个方法也是MVC框架的入口。

## Filter如何工作？

在Tomcat容器中，FilterConfig和FilterChain的实现类分别是ApplicationFilterConfig和ApplicationFilterChain，而Filter的实现类由用户自己定义，只要实现Filter接口定义的三个方法就行。

- init():初始化接口

- doFilter(ServletRequest,ServletResponse,FilterChain):在每个用户请求进来时，这个方法都会被调用，并且在Service方法之前。Filter采用的责任链模式。

- destroy:当filter对象被销毁时，此方法被调用。


----------


至此，Servlet系列的博客算是告一段落了，后面有时间，会继续往深处挖，探究其奥秘。

		
