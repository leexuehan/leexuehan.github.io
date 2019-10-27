---
title: Tomcat原理简述
tags:
  - Struts2
date: 2015-09-08 12:03:36
categories: 技术
---

本篇先介绍一下 Tomcat 的架构，然后就是处理Http请求的原理以及过程实现。

## 架构

![](http://i.imgur.com/O6ODYK4.png)

从结构上来划分,就是四个重要模块：

1.Server

Server 是Tomcat构成的顶级元素。所有一切都包含在 Server 中，Server 的实现类 StandardServer 可以包含多个Service。

2.Service 

次顶级元素 Service 的实现类 StandardService 调用了Container接口，实际上是调用了 Servlet Engine。

3.Container

接下来的构成元素就是 Container。


> 主机(Host)、上下文(Context)和引擎(Engine)均继承自Container接口，所以它们都是容器。

![](http://i.imgur.com/15d6wij.png)

上图就是一张典型的容器模型。

但是，它们是有父子关系的，在主机(Host)、上下文(Context)和引擎(Engine)这三类容器中，引擎是顶级容器，直接包含是主机容器，而主机容器又包含上下文容器，所以引擎(Engine)、主机(Host)和上下文(Context)从大小上来说又构成父子关系，虽然它们都继承自Container接口。

4.Connector

连接器(Connector)将Service和Container连接起来，首先它需要注册到一个Service，它的作用就是把来自客户端的请求转发到Container(容器)，这就是它为什么称作连接器的原因。


那么从功能上来划分呢？

从功能上划分可以分为5个模块：

1.Jsper

这个子模块负责jsp页面的解析、jsp属性的验证，同时也负责将jsp页面动态转换为java代码并编译成class文件。

2.Servlet和Jsp规范的实现模块

这个子模块的源代码属于javax.servlet包及其子包，如我们非常熟悉的javax.servlet.Servlet接口、javax.servet.http.HttpServlet类及javax.servlet.jsp.HttpJspPage就位于这个子模块中。

3.**Catalina子模块**（重要）

这个子模块包含了所有以org.apache.catalina开头的java源代码。

该子模块的任务是规范了Tomcat的总体架构，定义了Server、Service、Host、Connector、Context、Session及Cluster等关键组件及这些组件的实现，

这个子模块大量运用了Composite设计模式。同时也规范了Catalina的启动及停止等事件的执行流程。从代码阅读的角度看，这个子模块应该是我们阅读和学习的重点。

4.Connectors子模块

如果说上面三个子模块实现了Tomcat应用服务器的话，那么这个子模块就是Web服务器的实现。

所谓连接器(Connector)就是一个连接客户和应用服务器的桥梁，它接收用户的请求，并把用户请求包装成标准的Http请求(包含协议名称，请求头Head，请求方法是Get还是Post等等)。同时，这个子模块还按照标准的Http协议，负责给客户端发送响应页面，比如在请求页面未发现时，connector就会给客户端浏览器发送标准的Http 404错误响应页面。

5.Resource子模块

这个子模块包含一些资源文件，如Server.xml及Web.xml配置文件。严格说来，这个子模块不包含java源代码，但是它还是Tomcat编译运行所必需的。

## 处理 Http 请求

### 再谈connector

前面说了，Connectors就是一个连接器，把用户的请求包装成一个标准的 Http 请求发送到服务器。

**种类：**

1.Http Connector

基于HTTP协议，负责建立HTTP连接。它又分为BIO Http Connector与NIO Http Connector两种，后者提供非阻塞IO与长连接Comet支持。

2.AJP Connector

基于AJP协议，AJP是专门设计用来为tomcat与http服务器之间通信专门定制的协议，能提供较高的通信速度和效率。如与Apache服务器集成时，采用这个协议。

3.APR HTTP Connector

用C实现，通过JNI调用的。主要提升对静态资源（如HTML、图片、CSS、JS等）的访问性能。现在这个库已独立出来可用在任何项目中。Tomcat在配置APR之后性能非常强劲。

**配置：**

	<Connector port=”8080” protocol=”HTTP/1.1” maxThreads=”150” 
		connectionTimeout=”20000” redirectPort=”8443”/>

其他一些重要属性：

	acceptCount : 接受连接request的最大连接数目，默认值是10
	address : 绑定IP地址，如果不绑定，默认将绑定任何IP地址
	allowTrace : 如果是true,将允许TRACE HTTP方法
	compressibleMimeTypes : 各个mimeType, 以逗号分隔，如text/html,text/xml
	compression : 如果带宽有限的话，可以用GZIP压缩
	connectionTimeout : 超时时间，默认为60000ms (60s)
	maxKeepAliveRequest : 默认值是100
	maxThreads : 处理请求的Connector的线程数目，默认值为200

可以看到，这些属性都是与连接有关系的。

### 收到 Http 请求的运行流程

1.请求被发送到本机端口8080，被在那里侦听的Coyote HTTP/1.1 Connector获得

2.Connector把该请求交给它所在的Service的Engine来处理，并等待Engine的回应

3.Engine获得请求localhost:8080/test/index.jsp，匹配它所有虚拟主机Host
Engine匹配到名为localhost的Host（即使匹配不到也把请求交给该Host处理，因为该Host被定义为该Engine的默认主机）

4.localhost Host获得请求/test/index.jsp，匹配它所拥有的所有Context
Host匹配到路径为/test的Context（如果匹配不到就把该请求交给路径名为""的Context去处理）

5.path="/test"的Context获得请求/index.jsp，在它的mapping table中寻找对应的servlet

6.Context匹配到URL PATTERN为*.jsp的servlet，对应于JspServlet类
构造HttpServletRequest对象和HttpServletResponse对象，作为参数调用JspServlet的doGet或doPost方法

7.Context把执行完了之后的HttpServletResponse对象返回给Host

8.Host把HttpServletResponse对象返回给Engine

9.Engine把HttpServletResponse对象返回给Connector

10.Connector把HttpServletResponse对象返回给客户browser

画一个简单的图：

![](http://i.imgur.com/SLFhWXr.png)


