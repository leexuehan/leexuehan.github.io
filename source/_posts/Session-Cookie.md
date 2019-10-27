---
title: 'Session && Cookie '
tags:
  - Http
  - Struts2
date: 2015-07-22 21:15:03
categories: 技术
---

### 什么是Cookie ？

通俗地说，就是当一个用户通过 Http 访问一个服务器时，这个服务器会将一些 Key/Value 对返回给客户端浏览器，并将这些数据加上一些限制条件，在条件符合时，下次客户访问这个服务器时，数据又被完整地带回给服务器。所以 Cookie是**服务器产生的，保存在客户端**。

Cookie 分为两种：一种是存在于客户端的内存中，这样当会话关闭时，cookie 也自然消失，另一种就是存在于客户端的硬盘里面，在没有过期时，会一直存在。

### 用途

因为HTTP协议是无状态的，即服务器不知道用户上一次做了什么，这严重阻碍了交互式Web应用程序的实现。在典型的网上购物场景中，用户浏览了几个页面，买了一盒饼干和两瓶饮料。最后结帐时，由于HTTP的无状态性，不通过额外的手段，服务器并不知道用户到底买了什么。 所以Cookie就是用来绕开HTTP的无状态性的“额外手段”之一。服务器可以设置或读取Cookies中包含信息，借此维护用户跟服务器会话中的状态。
在刚才的购物场景中，当用户选购了第一项商品，服务器在向用户发送网页的同时，还发送了一段Cookie，记录着那项商品的信息。当用户访问另一个页面，浏览器会把Cookie发送给服务器，于是服务器知道他之前选购了什么。用户继续选购饮料，服务器就在原来那段Cookie里追加新的商品信息。结帐时，服务器读取发送来的Cookie就行了。

Cookie另一个典型的应用是当登录一个网站时，网站往往会请求用户输入用户名和密码，并且用户可以勾选“下次自动登录”。如果勾选了，那么下次访问同一网站时，用户会发现没输入用户名和密码就已经登录了。这正是因为前一次登录时，服务器发送了包含登录凭据（用户名加密码的某种加密形式）的Cookie到用户的硬盘上。第二次登录时，（如果该Cookie尚未到期）浏览器会发送该Cookie，服务器验证凭据，于是不必输入用户名和密码就让用户登录了。

### Cookie 的缺陷

1.由于 Cookie 是附加在 Http 请求中，所以无形中增加了流量。

2.Cookie是明文传输的，所以涉及到一些安全问题。

3.Cookie的大小有限制，一般是4KB，所以如果要传输复杂的数据不太适合。

##  什么是Session

上面说了 Cookie 会让服务端程序跟踪每一个客户端的访问，但是每次客户端的访问都必须传回这些Cookie，如果Cookie很多，则无形增加了客户端与服务端的数据传输量，而 Session 的出现正是为了解决这个问题。

同一个客户端访问服务器时，并不是每次都要传回所有的Cookie 值，只需要传回一个 ID ，这个 ID值就是客户端第一次访问服务器生成的，而且每个客户端是唯一的。

#### 什么是无状态呢？

客户与服务器建立连接、发出请求、得到响应、关闭连接。整个流程走完就算完了。下次服务器并不知道是否跟客户打过交道。

简言之，对于容器而言，每一个请求都来自于一个新的客户。

### 怎么让服务器记住你？

当客户端发起请求时，session会在客户端创建一个sessionid，这个sessionid存在于客户端的内存中，当会话关闭，这个sessionid也就消失了。

对于客户的一个请求，服务器会先检查客户端的cookie 里面有没有 sessionid，是否已经过期，如果有的话，根据sessionid 用一种哈希的方式将之检索出来，如果没有的话，容器会生成一个唯一的ID（是已经加密过的），并创建一个新的session，并通过响应把它返回给客户。客户再在以后的每一个请求中发回这个会话ID。容器看到此ID后，会找到响应的匹配的画画，并把这个会话与请求关联。而交换会话ID的最简单也最常用的方式是通过cookie。

session 默认存储在服务器的硬盘里面。

>总结 Session 和 Cookie的区别就是：Sessionid 和 Cookie 都由服务端产生，但是保存的位置不一样，Cookie 保存在客户端， SessionId 保存在服务端。

### Session 的创建

有了 SessionID 服务器就可以创建 HttpSession 对象了，第一次触发通过 request.getSession() 方法。

如果当前的 Session ID 还没有对应的 HttpSession 对象，那么就创建一个新的，并将这个对象加到 org.apache.catalina.Manager 中的一个 Session 容器中保存。这个类将管理所有的 Session 的生命周期，Session过期将被回收，服务器关闭，Session 将会被序列化到磁盘中。

只要这个 HttpSession对象存在，用户就可以通过 Session Id 来获取这个对象。

这个 Manager的实现类名字叫：StandardManager。

当Servlet容器关闭时，它会调用unload方法将 sessions 集合中的 StandardSession 对象写到“SESSION.ser”文件中，然后在启动时在按照上面的状态重新恢复。

### 分布式 Session 框架

虽然 Session 比 Cookie 有安全性，但是不能解决分布式部署问题。

在大型互联网系统中，使用 Cookie 可以解决分布式部署问题，当有上百台服务器分工协作时，由于 Cookie 是将值存储在客户端的浏览器中，用户每次访问都会将最新的值带回给处理该请求的服务器，解决了同一个用户的请求可能不在同一台服务器处理而导致的Cookie不一致的问题。

但是Cookie的缺陷也是很明显的，具体见上文。

于是必须结合 Cookie 与 Session 的优点解决问题。

这里简单说一下其实现原理：

首先需要一个订阅服务器，在应用启动时，可以从这个订阅服务器订阅这个应用需要的可写 Session 项 和可写 Cookie项。

统一订阅服务器集中管理资源，如果应用要使用一个新增的 Cookie，则可以通过一个统一的平台来申请，申请通过则将这个配置项增加到订阅服务器中。

同时还需要一个分布式缓存服务器，要知道在大访问量的情况下，应用服务器都是一个集群，因为每一个应用服务器的内存不可能保存几十万访问量的session，即使够用，也需要涉及一个同步问题，要想将所有应用同步到所有的服务器中太困难了。

所以用分布式缓存来存放Session，可以供其他服务器随时读取和写入。


### 防止表单重复提交

防止表单重复提交，就要标识用户的每一次访问请求，使得每一次访问对服务端来说都是确定的唯一的。

为了标识用户的访问请求，可以在用户的请求一个表单时增加一个隐藏表单项，这个表单项的值每次都是唯一的 token。

         <form id = "form" method="post">
              <input tupe = hidden name = "crsf_token" value="xxx">
         </form>

### cookie是谁来管理的？

容器会做cookie的所有工作。包括：生成会话ID，创建新的cookie对象，把会话ID放在cookie中，把cookie设置为响应的一部分。

在响应中发送一个会话

	HttpSession session = request.getSession();

### 如何判断会话已经存在还是刚刚创建？

	session.isNew();//如果是刚创建的会返回true，反之则为false

### 只想要已经有的会话怎么办？

对此，专门有一个重载的方法getSession(boolean),如果不想创建一个新的会话，可以调用getSession(false),此会话要么返回null，要么返回一个已经有的HttpSession。

### 如果客户不接受cookie怎么办？

如果客户禁用cookie，则意味着getSession()总会返回一个新的会话，客户不会发回一个带有会话ID cookie首部的请求。

此时可以使用URL重写。

我们的目的是为了让容器和客户交换会话ID信息，最简单的方法当然是用cookie。但是此时如果不能把ID放在cookie中，怎么办？只有利用URL重写将cookie会话中的会话ID取出来放在访问应用的各个URL的最后。


### 会话怎么删除？

会话对象会占用资源，所以必须要让没用的对话删除。

那么问题来了？怎么才能让容器知道你什么时候删除一个对话呢？或者什么时候客户已经离开了？容器又在什么时候删除这个对话呢？

这个时候要用到HttpSession接口了。他有一些很有用的方法可供调用。

	getCreationTime():返回第一次创建对话的时间。**可以用它来得出对话存在的时间。**
	
	getLastAccessedTime():返回容器最后一次得到包含这个会话ID请求后过去了多长时间。**可用来得出客户最后一次访问这个对话是什么时候**。
	
	setMaxInactiveInterval():指定对于这个会话客户请求的最大间隔时间。**可以用来撤销容器中废弃的会话**。
	
	getMaxInactiveIntervale():返回这个会话客户请求的最大间隔时间。
	
	invalidate():结束会话。

所以会话通常有三种死法：

- 超时

- 你在会话对象上调用validate

- 应用结束（崩溃或者取消部署）

1.在DD中配置会话超时

	<web-app ....>
		<servlet>
			...
		</servlet>
		<session-config>
			<session-timeout>15</session-timeout>
			<!--15分钟超时-->
		</session-config>
	</web-app>

2.设置一个特定会话超时

	session.setMaxInactiveInterval(20*60);//20 minutes

## 关于Cookie

### cookie还有什么别的作用？

前面提到了cookie可以帮助容器记住会话状态，那么除此之外还能干什么呢？

默认，cookie与会话寿命一样长，一旦客户离开浏览器，cookie就会消失。但是你可以让cookie活的更长一些，甚至在关闭浏览器之后还能存活。

创建一个cookie：

	Cookie cookie = new Cookie("username", name);

设置Cookie在客户端上存活多久：

	cookie.setMaxAge(30*60);//30 minutes

把Cookie发送到客户：

	response.addCookie(cookie);

从客户请求得到cookie:

	Cookie[] cookie = request.getCookies();
	for(int i= 0;i < cookies.length; i++) {
		Cookie cookie = cookies[i];
		if(cookie.getName().equals("username")) {
			String userName = cookie.getValue();
			break;
		}
	}

## HttpSessionBindingListener

这个监听器的作用是：当一个属性类实现了此接口是，这个实例增加到一个会话或者从一个会话中删除时，容器会调用该事件处理的回调方法。

代码示例：

	public class ClassName implements HttpSessionBindingListener {
	//省略其他属性方法定义，或者Setter、getter定义
	public void valueBound(HttpSessionBindingEvent event) {
       //添加处理事件
	}
	public void valueUnBound(HttpSessionBindingEvent event) {
	 	//添加处理事件
	}

## 会话迁移

在分布式应用上，应用的各个部分可以复制到网络的多个节点上，在一个集群环境中，容器可能会完成负载均衡，取得客户的请求并把它放在多个JVM上。

这个时候ServletContext, ServletConfig, HttpSession对象会发生什么变化呢？

答案是：**每个VM上都有一个ServletContext。每个VM上的每个Servlet有一个ServletConfig。但是每个Web应用的一个给定的会话ID，只有一个HttpSession对象，而不论应用分布在多少个VM上**。

所以尽管应用的其他部分会在每个节点VM上复制，但是会话对象<font color=#FF0000>不会复制，只会迁移</font>。

## HttpSessionActivationListener

如果你的属性类型是Serializable，那就用不着该监听器了。如果不是，可以实现该监听器，使得属性实现串行化。

## 总结与会话相关的监听器

	HttpSessionListener 
	
	主要方法：sessionCreated sessionDestroyed
	
	可以用来统计有多少个并发用户
	
	HttpSessionBindingListener
	
	HttpSessionBindingListener
	//上面两种监听器，前面已经介绍，此处从略

	HttpSessionAttributeListener
	
	主要方法：attributeAdded() attributeRemoved() attributeReplaced()
	
	可以知道会话中什么时候增加删除或者替换会话属性。

这是最后一篇关于Servlet基础知识的文章，通过三篇博客，大致介绍了Servlet的常用功能，我们知道怎么用它，但是具体Servlet的原理，请看下一篇。
