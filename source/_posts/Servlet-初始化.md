---
title: Servlet 初始化
tags:
  - Struts2
date: 2015-07-22 21:13:23
categories: 技术
---


### 如何获取初始化参数
容器在初始化的时候，会为了这个Servlet创建一个唯一的ServletConfig，容器会从DD读出Servlet的初始化参数，并把这个参数交给ServletConfig，然后ServletConfig传递给Servlet的init()方法。

<font color="red">这样做的好处就是当你需要更改某个参数的值时，只要在配置文件中更改即可，如果是在Servlet中硬编码，则需要重新编译。</font>

>在DD文件中这样配置

>       <servlet>
>             <servlet-name>name</servlet-name>
>             <servlet-class>class-name</servlet-class> 
>             
>             <init-param>
>                  <param-name>name</param-name>      
>                  <param-value>value</param-value>
>             </init-param>
>       </servlet>

>在Servlet代码中只要这样调用就行：

>       out.println(getServletConfig().getInitParameter("name"));

**注意：获得此参数的时机：不要在Servlet的构造函数中调用此方法，因为这时候还没有执行init方法，所以没有ServletConfig。**

### JSP能不能获取到Servlet初始化参数

ServletConfig仅用于Servlet的初始化配置，所以如果想让应用的其他部分也能使用这部分初始化配置信息，则需要别的方法。

解决办法：

因为在前面提到过，ServletContext是整个web应用的参数，所以无论Servlet还是JSP都能访问到，所以在这里配置，应该是极好的。

>      <Servlet>
>           ............
>      </servlet>
       //注意不能嵌套在上面的Servlet配置里面。
>      <context-param>
>              <param-name>name</param-name>
>              <param-value>value</para-value>
>      </context-param>

调用方法：

>      out.println(getServletContext().getInitParameter("canshu"));

**如果应用是分布式应用，则在每一个JVM中都会有一个ServletContext.**

那么问题来了，上面的访问方法时Servlet访问ServletContext的方法，在JSP中有怎么样呢？

在JSP中有一些所谓的“隐式对象”，其中就包括ServletContext.关于JSP的详细介绍请参见本博客。

### 应用初始化参数如果不只是String类型呢，比如DataSource？

此时需要一个监听器（listener），即监听一个上下文初始化事件，这样就能得到一个上下文初始化参数，并在应用为客户提供服务之前运行一些代码。

比如获取数据库连接，则需要一个ServletContextListener。它可以建立一个单独的类，不是Servlet或者JSP，用来监听ServletContext的创建和撤销。此类实现了javax.servlet.ServletContextListener

关于此例，需要做到：

- 上下文初始化（应用部署中）得到通知
     
    从ServletContext得到上下文初始化参数；

	从初始化参数查找名建立一个数据库连接

	把数据库连接存储为一个属性，使得Web应用的各个部分都能访问

- 上下文撤销时，得到通知

	关闭数据库连接

- 代码段

>        import javax.servlet.*;

>        public class MyServletContextListener implements ServletContextListener {

>        public void contextInitialized(ServletContextEvent event)    {    
>             //code to initialize the database connection
>             //and store it as a context attribute
>        }

>        public  void contextDestroyed(ServletContextEvent event) {
>        //code to close the database connection
>       }
>       }

### 如何建立一个上下文监听者

1.创建一个监听类实现监听接口，并且像上面的代码所示，实现两个方法；

2.要在web.xml部署文件放一个<listener>元素

		<listener>
           <listener-class>class</listener-class>
		</listener>
		//这个文件要部署到Servlet外面去

注意：通过getAttribute方法获取一个属性时，一定要注意强制转换，因为此方法返回的是一个Object类型。

### 除了监听上下文还能监听什么？

事实上，除了监听上下文事件，还可以监听与上下文属性、Servlet请求和属性，以及HTTP会话和会话属性相关的事情。

以下列出了八种监听器以及适用场合

![](http://i.imgur.com/GUnfQfZ.png)

### 什么是属性

上面提到了属性，到底什么是属性呢？

属性实质就是一个对象，可能设置到或者绑定到ServletContext、HttpServletRequest或者HttpSession对象中的某一个。
简言之，就是一个映射实例对象中的键值对(String-Object)。

### 属性和参数有什么区别？
类型不同：属性类型包括应用，请求，会话；参数类型包括应用初始化参数，请求参数，Servlet初始化参数。

设置方法不同：属性通过setAttribute(String name, Object value)设置；而参数通过DD设置

返回类型不同：属性返回的是一个对象，参数返回的一个字符串

获取方法：属性通过getAttribute(String name)获取，参数通过getInitParameter(String name)方法获取。

### 属性的API

上下文、请求和会话都由相应的接口处理，即ServletConfig、ServletRequest和HttpSession。每个接口的API方法完全相同。

		Object getAttribute(String name)
		
		void setAttribute(String name,Object value)
		
		void removeAttribute(String name)
		
		Enumeration getAttributeNames()


需要注意的一点是：上下文作用域不是线程安全的。<font color="blue">因为应用中可能有多个Servlet，而每一个Servlet都能访问上下文属性，而请求又是并发的，所以会出现线程安全问题。</font>

**如何做到上下文属性线程安全？**

可以有一个思路就是：同步。

在每个Servlet的doGet()方法中同步。但是这样做的一个后果就是Servlet再也不能进行并发处理了，一次只能处理一个客户。

而且只能意味着Servlet中一次只能运行一个线程，同一个Servlet中的其他线程访问上下文属性，但是**不能阻止另一个不同的servlet的访问**。

所以一个可行的办法是：对于上下文加锁。

代码实例：

	public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException,ServletException {
	    synchronized(getServletContext()) {
	         //get the attribute and do something
		}
	}
    

**如何做到会话属性线程安全？**

首先应该弄清楚一个概念，会话属性是线程安全的吗？
答：否。

因为同一个用户很有可能会在同时做出多个请求（可以想象：同一个客户可能打开一个新的浏览器窗口，容器吧来自第二个窗口的请求看作是来自同一个会话）。

代码实例：

	public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException,ServletException {
			HttpSession session = request.getSession();
		    synchronized(session) {
		         //make use of the session and set Attributes
			}
	}

关于同步的忠告：因为同步要检查、获得和释放锁，所以确实会增加一些额外的开销，但是不论是哪一种形式的锁都有一条标准：**一定要在最短的时间完成目标，使持有锁的时间最短。要让同步块尽可能的小。**

### 怎么让Servlet一次只得到一个请求？

首先要了解一个接口：SingleThreadModel(STM).

这个接口没有任何方法，如果一个Servlet实现了这个接口，就可以保证不会在该Servlet的服务方法中并发执行两个线程。通过同步实现对Servlet单个实例的访问，或者通过维护一个Servlet实例池并把每个新请求分配到一个空闲的Servlet，而Servlet容器可以做到这样一点。

记住：只有**请求属性和局部变量**是线程安全的。

### 关于RequestDispatcher

只有两个方法：forward()和include()

1.从ServletRequest得到RequestDispatcher

		RequestDispatcher view = request.getRequestDispatcher("result.jsp");
		//注意：上面的参数如果不是以 /开头，则是相对路径，反之，则是绝对路径。

2.从ServletContext得到RequestDispatcher

		RequestDispatcher view = getServletContext().getRequestDispatcher("/result.jsp");
		//注意：此处路径必须以/开头

然后，在RequestDispatcher上调用forward()方法

		view.forward(request,response);
