---
title: Servlet 解析
tags:
  - Struts2
date: 2015-07-22 21:07:19
categories: 技术
---

**Servlet的作用是什么**

因为我们的资源请求都是从浏览器中发出的，那么必然会有一个服务器来接受此请求，然后返回我们需要的资源。

因为我们发出的请求一般都是通过 Http 协议来承载的，所以服务器一般也就是 Http 服务器。如果你用的是FTP协议，可能还会有 FTP服务器。

Web请求的一般过程：建立连接-发送请求-应答过程-关闭连接。

这里有一张画得很不错的图：

![tomcat server](https://cloud.githubusercontent.com/assets/12984324/9716850/69a374ce-55a0-11e5-955a-8d9bcd7e0ce6.png)


可见：Servlet就是Http服务器用来处理此请求的一个载体。

从浏览器（Client）获取连接”/helloWorld.html”

Connector模块要做的事情：

Tomcat Connector模块将请求(Request)传递给 Container模块。

Container 模块会做以下事情：
分析HTPP请求信息，
组装成HttpServletRequest对象，
创建新的HttpServletResponse对象，
根据配置文件，搜索相应的Servlet，并创建一个线程用于处理本次请求。
此时线程会将上面Request和Response对象的索引，传递给Servlet。

新线程中的Servlet处理逻辑：
线程结束后，通过HttpServletResponse对象的PrintWriter，返回浏览器一个信息。

在建立连接以后的处理请求、发送响应等过程都离不开 Servlet。所以容器一般也叫 Servlet 引擎。而自然，因为 Http 请求方法众多，所以 HttpServlet 也必须有对应的 doGet(),doPost(),doPut()等等方法，来与之对应。

**Servlet 的生命周期**

类加载、实例化，init方法，service方法，destroy方法。关于这点，后面会有详细介绍。

**什么是容器？**

因为Servlet没有main方法，所以它受控于另一个Java应用，这个应用就叫容器。而Tomcat就是这样一个容器之一。

**容器的作用**

容器来管理和运行Servlet。具体负责以下几项：

1.通信支持：让Servlet与Web服务器对话，而不需要自己建立ServerSocket、监听端口、创建流等。容器知道自己和Web服务器的协议。

2.生命周期管理：负责加载类、实例化和初始化Servlet、调用Servlet方法，并使Servlet实例能够被垃圾回收。

3.多线程支持：容器自动为每个Servlet请求创建一个新的Java线程，针对客户的请求，如果Servlet已经运行完相应的HTTP服务方法，这个线程就会挂掉。

4.声明方式实现安全：利用容器可以使用XML部署描述文件来配置和修改安全性，而不必将其硬编码到Servlet类代码中。

>不用重新编译，就能修改和管理安全配置，不亦快哉！

5.最关键的是支持JSP：容器负责吧JSP翻译成java.

**Servlet为什么是Servlet？**

一般来说，绝大部分的Servlet中都是httpServlet，绝大部分的httpServlet都会覆盖doGet()或者doPost()方法。

doPost()或者doGet()方法里面的参数，就是创建的请求和响应对象的引用。

**容器怎么找到Servlet？**

一种通用的方法就是使用部署描述符文件将URL映射到Servlet。

这样的好处有很多，其中最大的好处就是：非程序员也可以修改和部署你的web应用。


----------
简言之，Servlet的存在就是要为客户服务，Servlet的任务就是得到一个请求，再发回一个响应。

----------
**servlet家族**

如图所示
![6-27-1](https://cloud.githubusercontent.com/assets/12984324/8826049/315fdbde-30b6-11e5-9306-3fc37d42d7bd.png)

**Servlet生命周期的三大重要时刻**

1.init():
在Servlet实例创建之后，并在为客户请求提供服务之前，容器要对Servlet调用此方法。

如果有初始化代码（比如得到一个数据库连接，或者向其他对象注册）就要覆盖此方法。

2.service():
第一个客户请求到来时，容器开始一个新线程，或者从线程池分配一个县城，并调用Servlet的service()方法。

3.doPost()/doGet()

主要在这里写业务代码。

### 注意：容器为每一个请求都提供一个单独的线程运行。

**ServletConfig对象**



- 每一个Servlet都有一个ServletConfig对象。



- 用于访问ServletContext。

**ServletContext对象（AppContext）**



- 每一个Web应用有一个ServletContext。



- 用于访问Web应用的参数



- 用于得到服务器信息，包括容器名和容器版本。

**关于请求和响应**

### 请求：
>HttpServletRequest和HttpServletResponse接口由谁来实现？

>答:由容器来实现。

>http请求方法除了POST和GET还有那些？

>HEAD -----只要求得到GET返回的首部部分，所以这有点想GET，但是响应中没有体。

>TRACE-----要求请求消息回送，这样客户能看到另一端接受了什么。以便排错。

>PUT----指出要把所包含的信息放在请求的URL上。

>DELETE----删除请求Url的一个资源。

>OPTIONS----要求得到一个HTTP方法列表。

>CONNECT----要求连接以便建立隧道。

**POST和GET的区别**

>区别一：POST有消息体。GET方法对请求参数有限制，参数数据只能放在请求行的内容中。而且POST方法可以发送较大的数据。

>区别二：安全性的问题。GET方法，参数的数据会显示到输入地址栏里面，是明文显示。

>区别三：使用场合不同。GET 方法仅仅用来得到某些东西。对服务器不能做出任何改变。POST则用户发送数据来进行处理。

### 响应：
响应是要返回给客户。这是浏览器得到、解析并呈现给用户的东西。一般来说，要用响应得到一个输出流，通常是Writer，并使用这个流写出HTML，返回去给客户。但是应该还有其他办法。

大多数情况下，使用响应是为了向客户发回数据，会对响应调用两个方法：setContentType()和getWriter().



- setContentType():

函数功能：告诉浏览器返回了什么，这样浏览器才能根据响应的内容，调用适当的辅助应用，来展现之。这里的内容类型就是MIME类型。

>常用的MIME类型：

>text/html
>
>application/pdf
>
>video/quicktime
>
>application/java
>
>image/jpeg
>
>application/jar
>
>application/octet-stream
>
>application/x-zip

>关于输出响应：

>ServletResponse接口只提供了两个流可供选择：ServletOutputStream用于输出字节，PrintWriter用于输出字符数据。

    PrintWriter writer = response.getWriter();

	writer.println("some text and HTML");

	ServletOutputStream out = response.getOutputStream();

	out.write(aByteArray);

PrintWriter 实际上包装了ServletOutputStream.关于装饰器模式，请参见博文[http://www.cnblogs.com/xuehanlee/p/4603998.html](http://www.cnblogs.com/xuehanlee/p/4603998.html)

**设置响应首部和增加响应首部**

response.setHeader();

response.addHeader();

这两种方法顾名思义，只有在响应中还没有首部时，才一样，如果响应中已经有了首部，则后者是增加一个值，而前者会覆盖现有的值。

### Servlet重定向

response.sendRedirect("url");

这个方法能使浏览器重新发一个请求，来请求url对应的资源。此时浏览器上的地址会发生变化。

### Servlet分派

重定向让浏览器重新发一条请求，而分派则是让服务器去指派资源来完成任务，此时用户根本不知道里面发生了什么，总之，最后服务器顺利的将资源呈现在浏览器。过程对于用户来说是完全透明的。