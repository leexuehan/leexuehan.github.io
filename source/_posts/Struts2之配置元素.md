---
title: Struts2之配置元素
tags:
  - Struts2
date: 2015-07-24 10:51:47
categories: 技术
---

这一篇从Struts2配置元素来讲Struts2.

## 


> 配置就是程序的影子，无论什么框架，框架总是作为一个重要的组成部分，在框架的运行过程中发挥作用。

### 表现形式

Struts2支持所有的配置形式包括XML和Properties。从形式上来讲，以XML为核心，Properties作为辅助。


### 作用范围

struts-default.xml和default.properties是框架级别的配置文件。这两个文件蕴含在Struts2的核心包中，应用程序启动时被初始化程序读取并加载。

struts.xml和struts.Properties作用于应用级别，其中定义的所有内容将覆盖框架级别的定义。

struts-plugin.xml则通过插件形式进行应用级别的定义。

### 配置文件的必要性

应用程序级别的配置文件都不是必须存在的。只有Web.xml中的配置是必须的，因为在其中定义了Struts2的入口程序。

如图：

![](http://i.imgur.com/tHuaUNU.png)

### 配置元素的定义

下图是struts-default.xml中截取的一部分：

![](http://i.imgur.com/AzSRzJG.png)

这只是冰山一角。

接下来将分析其中的含义。

1.include节点

最为常见的使用方法，我们可以在应用级别的配置文件里，将整个应用的配置根据一定的逻辑划分成若干个独立的配置文件，以方便的进行**模块化处理**。

2.bean节点

从上面的图中也可以看出，这个节点实际上一个用于描述**接口及其实现类映射关系的节点。**

bean节点的寻址是通过name属性和type属性共同构成一个逻辑主键来共同决定一个class属性。

Struts2在框架级别实现了一个对象容器，**并将配置文件中所有的bean节点所定义的对象纳入容器之中进行管理。Struts2通过容器在框架级别负责这些对象的创建、销毁以及依赖关系的处理。**

在应用级别配置文件中，通过**增加bean节点**，就可以把一个对象纳入Struts2的容器中进行管理。

3.constant节点

在配置文件中的存在形式
	
	<constant name="struts.excludedClasses"
	              value="
	                java.lang.Object,
	                java.lang.Runtime,
	                java.lang.System,
	                java.lang.Class,
	                java.lang.ClassLoader,
	                java.lang.Shutdown,
	                ognl.OgnlContext,
	                ognl.MemberAccess,
	                ognl.ClassResolver,
	                ognl.TypeConverter,
	                com.opensymphony.xwork2.ActionContext" />

是一个非常典型的键值对类型的配置，**主要用于定义Struts2运行时的参数**。
其实与Properties文件配置的作用是完全一样的。

4.package节点

是一个复杂的复合节点，在其中包含了众多子节点：ResultType、Interceptor、Action等。这些子节点全部继承自XWork框架。

**一个Package节点实际上可以被看做是一条简单的XWork生产流水线，其中包含XWork框架如何应对某些请求并选择相应的执行序列进行处理的具体方式。**

package节点的属性之中，name是唯一的标识符，namespace则从命名的空间的角度为整个事件请求机制划分不同的种类。

extends属性允许继承。

### Struts2配置元素的分类

XML配置文件中的bean节点和constant节点，可以归为**容器配置元素**。

XML配置文件中的package节点定义了一种事件请求响应的映射关系，可以归为**映射关系元素**。