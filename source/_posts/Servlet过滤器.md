---
title: Servlet过滤器
tags:
  - Struts2
date: 2015-07-22 21:04:14
categories: 技术
---

过滤器允许拦截请求，而Servlet对此完全透明。

与Servlet非常类似，过滤器就是Java组件，请求发送到Servlet之前可以用过滤器截获和处理请求，Servlet结束工作之后，在响应发回客户之前，可以用过滤器处理响应。


过滤器是模块化的，可以在DD中进行配置.
Java类实现了Filter接口，则成为一个过滤器。

## 过滤器的实现

作为过滤器必须实现Filter接口。

1.必须实现init方法，通常指在其内部保存config对象。这个对象允许通过getServletContext方法访问ServletContext。

	private FileterConfig fc;
	public void init(Filter config) throws ServletException {
		this.fc = config;
	}

2.必须实现doFilter方法。在其中添加具体的工作。

	//doFilter参数是普通的请求和响应
	public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException,IOException {
		HttpServletRequest httpReq = (HttpServletRequest) req;
		//TO DO SOMETHING
		chain.doFilter(req,resp);//如果还有过滤器的话
	}

3.实现destroy方法。处理善后工作。

	public void destroy() {
		// 清理
	}

## 声明和确定过滤器的顺序

	<filter>
		<filter-name>XXXrequest</filter-name>
		<filter-class>package.xxx.xxx</filter-class>
		<init-param>
			....如果有参数的话
		</init-param>
	</filter>
	
	<!--声明URL模式的过滤器映射-->
	<filter-mapping>
		<filter-name>sss</filter-name>
		<url-pattern>xxx/xx/x</url-pattern>
	</filter-mapping>
	<!--声明对应Servlet名的过滤器映射-->
	<filter-mapping>
		<filter-name>sss</filter-name>
		<servlet-name>xxx</servlet-name>
	</filter-mapping>

那么如果这种请求不是用户直接输入浏览器而是通过转发和请求分派产生的呢？

这可以用以下方式解决之：

	<filter-mapping>
		<filter-name>sss</filter-name>
		<url-pattern>xxx/xx/x</url-pattern>
		<dispatcher>REQUEST(或者INCLUDE或者FORWARD或者ERROR)</dispatcher>
	</filter-mapping>

## 过滤器的应用场合

1.日志过滤器

可以建立一个文本文件记录请求的URI，文本文件的名称可以通过一个初始参数来配置，可以利用此文件，记录网站的访问量或者资源的受欢迎程度。

2.图片保护过滤器

防止通过在浏览器的地址栏中直接输入图片的URL来下载图片。只有在单击页面中的图片链接时，才会显示应用中的图片。这可以通过查看HTTP表头的referer的值来进行判断。

3.下载计数过滤器

简单实现，将资源的下载次数保存在一个属性文件中，而不是保存在数据库中。因为多个线程可以同时访问一个过滤器，因此就有一个线程的安全性的问题需要解决。

用户请求资源，过滤器读取相应的属性值，将它递增并存回新值。由于多线程的存在这样的计数必然不会准确。要用同步值的方法，又会涉及到一个可扩展性的问题。

在一个版本中这样实现：

>利用Queue和Executor解决此线程安全问题。简言之，就是所有进来的请求都>在一个线程Executor的队列中放置一项任务。因为放置是异步操作，所以比较快，不需要等待任务完成。Executor每次从队列中取出一个项目，并递增正确的属性。由于Executor值使用了一个线程，因此消除了多线程访问该属性文件的可能。-------《Servlet&JSP： A Tutorial》
