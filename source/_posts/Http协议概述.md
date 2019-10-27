---
title: Http协议概述
tags:
  - Http
date: 2015-09-07 17:44:38
categories: 技术
---

### 概述

HTTP，即超文本传输协议，是Http客户端（一般情况下就是浏览器）和Http服务器（Web服务器）之间通信的协议。Http协议目前的版本是 HTTP/1.1。

它是一种无状态协议。

所谓的无状态就是：客户端发送请求到服务器，服务器回送内容到客户端，然后这次连接就关闭了，服务端不保留任何连接有关的信息。

Http遵循请求（Request）-响应（Response）模型。所有的Http连接都被构造成一套请求和响应。

### 通信机制

一次Http连接总共分为下面所示的大致七个步骤：

1.建立TCP连接


> HTTP是比TCP更高层次的**应用层协议**，根据规则，只有低层协议建立之后才能，才能进行更高层协议的连接，因此，首先要建立TCP连接，一般TCP连接的端口号是80。在Web上，HTTP协议使用TCP协议而不是UDP协议的原因在于一个网页必须传送很多数据，而且保证其完整性。

2.Web浏览器向Web服务器发送请求命令

建好连接之后，发送请求命令类似：例如：GET/sample/hello.jsp HTTP/1.1

3.Web浏览器向Web服务器发送头信息

在发送请求命令之后，Web浏览器还需要发送头信息到Web服务器。头信息结束方式很简单：只要留一个空白的行就行了。服务器就知道了：空白行前面的就是头信息。

4.Web服务器应答

类似：HTTP/1.1 200 OK

含义解释请看后面。

5.Web服务器发送应答头信息

类似第三步。

6.Web服务器向Web浏览器发送数据

Web服务器向浏览器发送头信息后，它会发送一个空白行来表示头信息的发送到此为结束，接着，它就以Content-Type应答头信息所描述的格式发送用户所请求的实际数据

7.Web服务器关闭连接

> 一般情况下，一旦Web服务器向浏览器发送了请求数据，它就要关闭TCP连接，然后如果浏览器或者服务器在其头信息加入了这行代码
> 
> 	Connection:keep-alive
> 	
> TCP连接在发送后将仍然保持打开状态，于是，浏览器可以继续通过相同的连接发送请求。保持连接节省了为每个请求建立新连接所需的时间，还节约了网络带宽。

### 报文格式详解

#### Http请求

Http请求信息由3部分组成：请求方法，请求头，请求正文。

	GET/sample.jspHTTP/1.1
	Accept:image/gif.image/jpeg,*/*
	Accept-Language:zh-cn
	Connection:Keep-Alive
	Host:localhost
	User-Agent:Mozila/4.0(compatible;MSIE5.01;Window NT5.0)
	Accept-Encoding:gzip,deflate
	 
	username= leexuehan &password=1234

上面就是一个典型的例子：



1.请求方法 URI协议/版本

请求的第一行是“方法URL议/版本”：GET/sample.jsp HTTP/1.1

以上代码中“GET”代表请求方法，“/sample.jsp”表示URI，“HTTP/1.1代表协议和协议的版本。

根据HTTP标准，HTTP请求可以使用多种请求方法。

例如：HTTP1.1支持7种请求方法：GET、POST、HEAD、OPTIONS、PUT、DELETE和TARCE。

在Internet应用中，最常用的方法是GET和POST。

URL完整地指定了要访问的网络资源，通常只要给出相对于服务器的根目录的相对目录即可，因此总是以“/”开头，最后，协议版本声明了通信过程中使用HTTP的版本。


2.请求头(Request Header)

从第二行开始就是请求头部分。

请求头包含许多有关的客户端环境和请求正文的有用信息。例如，请求头可以声明浏览器所用的语言，请求正文的长度等。例如：

		Accept:image/gif.image/jpeg.*/*
		Accept-Language:zh-cn
		Connection:Keep-Alive
		Host:localhost
		User-Agent:Mozila/4.0(compatible:MSIE5.01:Windows NT5.0)
		Accept-Encoding:gzip,deflate.

3.请求正文

请求头和请求正文之间是一个空行，这个行非常重要，它表示请求头已经结束，接下来的是请求正文。请求正文中可以包含客户提交的查询字符串信息：

		username=leexuehan&password=1234

在以上的例子的HTTP请求中，请求的正文只有一行内容。当然，在实际应用中，HTTP请求正文可以包含更多的内容。

**请求方法**

HTTP请求方法我这里只讨论GET方法与POST方法.

GET方法

GET方法是默认的HTTP请求方法，我们日常用GET方法来提交表单数据，然而用GET方法提交的表单数据只经过了简单的编码，同时它将作为URL的一部分向Web服务器发送，因此，如果使用GET方法来提交表单数据就存在着安全隐患上。

例如 Http://127.0.0.1/login.jsp?Name=zhangshi&Age=30&Submit=%cc%E+%BD%BB

从上面的URL请求中，很容易就可以辩认出表单提交的内容，这就涉及到一点安全问题。另外由于GET方法提交的数据是作为URL请求的一部分所以提交的数据量不能太大。

POST方法

POST方法是GET方法的一个替代方法，它主要是向Web服务器提交表单数据，尤其是**大批量的数据**。

POST方法克服了GET方法的一些缺点：

通过POST方法提交表单数据时，数据不是作为URL请求的一部分而是作为标准数据传送给Web服务器，这就克服了GET方法中的信息无法保密和数据量太小的缺点。因此，出于安全的考虑以及对用户隐私的尊重，通常表单提交时采用POST方法。

请求方法概括：

	GET：通过请求URI得到资源
	POST：用于添加新的内容
	PUT：用于修改某个内容
	DELETE：删除某个内容
	TRACE：用于远程诊断服务器
	HEAD：类似于GET, 但是不返回body信息，用于检查对象是否存在，以及得到对象的元数据

#### Http应答

HTTP应答与HTTP请求相似，HTTP响应也由3个部分构成，分别是：协议状态版本代码描述、响应头、响应正文。

下面是一个HTTP响应的例子：

	HTTP/1.1 200 OK
	Server:Apache Tomcat/5.0.12
	Date:Mon,6Oct2003 13:23:42 GMT
	Content-Length:112
	
	<html>
		<head>
			<title>HTTP响应示例<title>
		</head>
		<body>
			Hello HTTP!
		</body>
	</html>

1.协议状态版本代码描述

	HTTP/1.1 200 OK

可以看到这部分分为：协议版本，应答码。

协议版本无需多说，重点说一些应答码：

HTTP应答码也称为状态码，它反映了Web服务器处理HTTP请求状态。HTTP应答码由3位数字构成，其中首位数字定义了应答码的类型：

	1XX－信息类(Information),表示收到Web浏览器请求，正在进一步的处理中
	2XX－成功类（Successful）,表示用户请求被正确接收，理解和处理例如：200 OK
	3XX-重定向类(Redirection),表示请求没有成功，客户必须采取进一步的动作。
	4XX-客户端错误(Client Error)，表示客户端提交的请求有错误 
		例如：404 NOT Found，意味着请求中所引用的文档不存在。
	5XX-服务器错误(Server Error)表示服务器不能完成对请求的处理：如 500

2.响应头

响应头(Response Header)响应头也和请求头一样包含许多有用的信息，例如服务器类型、日期时间、内容类型和长度等。

	Server:Apache Tomcat/5.0.12
	Date:Mon,6Oct2003 13:23:42 GMT
	Content-Length:112

3.正文

无需多说，自然是HTML文件或者其他格式的文件。

### 安全问题

前面在谈GET方法谈到安全的问题。

这里就大概说一些Http的安全机制。

Web应用最常见的用途之一是电子商务，可以利用Web服务器端程序使人们能够网络购物，需要指出一点是，缺省情况下，通过Internet发送信息是不安全的，如果某人碰巧截获了你发给朋友的一则消息，他就能打开它，假想在里面有你的信用卡号码，这会有多么糟糕，幸运的是，很多Web服务器以及Web浏览器都有创立安全连接的能力，这样它们就可以安全的通信了。

通过Internet提供安全连接最常见的标准是安全套接层(Secure Sockets layer,SSL)协议。

SSL协议是一个应用层协议(和HTTP一样)，用于安全方式在Web上交换数据，SSL使用公开密钥编码系统。

从本质讲，这意味着业务中**每一方都拥有一个公开的和一个私有的密钥**。当一方使用另一方公开密钥进行编码时，只有拥有匹配密钥的人才能对其解码。

简单来讲，公开密钥编码提供了一种用于在两方之间交换数据的安全方法，SSL连接建立之后，**客户和服务器都交换公开密钥**，并在进行业务联系之前进行验证，一旦双方的密钥都通过验证，就可以安全地交换数据。

相当于别人送来一个包裹，这个包裹比较特殊，你要想打开它，必须有你的口令和寄方的一个口令。而在寄方寄包裹的时候，你就与它联系过了，你知道口令是什么。你的可能是：天王盖地虎。然后他的可能是：宝塔镇河妖。

当你收到他的包裹时，就可以用宝塔镇河妖和你自己的私密口令结合打开这个包裹。
