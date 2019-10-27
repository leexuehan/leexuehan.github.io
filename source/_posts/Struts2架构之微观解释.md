---
title: Struts2架构之微观解释
tags:
  - Struts2
date: 2015-07-24 10:16:15
categories: 技术
---

本文从微观角度来介绍下Struts2.

还是按照上一篇讲的主线来看。

## 第一条主线——Struts2初始化

Struts2引入容器的概念，将**所有被需要管理的对象全部置于容器之中**。

整个Struts2的初始化过程围绕着容器来展开。构成整个Struts2初始化过程分为**数据结构定义**和**初始化行为的操作接口**两个部分。

### 数据结构定义

容器首先成为初始化主线中的核心元素，PackageConfig作为事件请求的配置元素也成为需要重点关注的构成元素。

容器及容器的实现类

	元素名称                     Java Package                       描述
	
	Container                   com.opensymphony.xwork2.inject    容器定义接口 
	ContainerImpl               com.opensymphony.xwork2.inject    容器实现接口，内部实现了	
															  	  Struts2进行对象声明周期管理
															      和依赖注入的基本功能

    PackageConfig               com.opensymphony.xwork2.config.   PackageConfig的实现类，其中
								entities						  定义了事件响应模型的完整数据
																  结构
	

### 行为定义

容器加载接口和容器的构造器


	元素名称                     Java Package                       描述
	
	ConfigurationProvider       com.opensymphony.xwork2.config    配置加载接口的统一接口
	ContainerProvider           com.opensymphony.xwork2.config    Container配置加载接口，其实
																  现类需要负责初始化容器内部的
																  所有对象

    PackageConfig               com.opensymphony.xwork2.config    PackageConfig的配置加载接口
  															      其实现类需要负责初始化用于处
																  理事件请求的配置对象
	ContainerBuilder            com.opensymphony.xwork2.inject    Container构造器，初始化时构
										   						  造容器
  	PackageConfigBuilder        PackageConfig的内部类              PackageConfig的构造器。



## 第二条主线


### 第一阶段：Http预处理请求

控制权在Struts2手中，这个阶段涉及的微观元素都是Struts2类。

	元素名称                  Java Package						描述
	
	Dispatcher               org.apache.struts2.dispatcher      Struts2的核心分发器，是Struts2进行Http请求处理的实际场所
	
	PrepareOperations        org.apache.struts2.dispatcher.ng   Http请求预处理的操作集合
	ExecuteOperations        org.apache.struts2.dispatcher.ng   Struts2 进行Http请求处理逻辑的操作集合


**Dispatcher是整个Struts2框架的核心，被称为核心分发器。**


### 第二条主线——第二阶段：XWork的执行逻辑

控制权到了XWork框架中。

	元素名称             Java Package                           描述
	
	ActionProxy         com.opensymphony.xwork2                XWork生产线中的执行环境，整个生产线的入口。
	ActionInvocation    com.opensymphony.xwork2                XWork生产线的调度者，负责调度整个生产线中各个元素的执行次序
	Interceptor         com.opensymphony.xwork2.interceptor    XWork生产线中的工序序列，可以丰富整个生产线的功能
	Action				com.opensymphony.xwork2                XWork生产线中的核心工序，负责核心业务逻辑的调用和实现
	ActionContext       com.opensymphony.xwork2                XWork生产线中的辅助设备，提供整个生产线工作运行的必须的数据环境。
	ValueStack 		    com.opensymphony.xwork2.util           XWork数据环境中提供表达式运算的工具类，也是XWork进行数据访问的基础。
	Result              com.opensymphony.xwork2                XWork生产线中的末端设备，负责输出生产线的生产结果


上面表单中的七大元素是贯穿整个XWork事件执行器的整个生命周期，各司其职，相互配合，提供了事件执行框架的足够的扩展接口，要熟记。

下图是它们之间的调用关系

![调用关系图](http://i.imgur.com/u4amYGh.png)




