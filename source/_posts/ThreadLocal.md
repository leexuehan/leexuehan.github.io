---
title: ThreadLocal
tags:
  - Concurrency
date: 2015-07-23 16:56:48
categories: 技术
---

访问共享的、可变的数据要求使用同步，一个可以避免同步的方式就是不共享数据。如果数据仅在单线程中被访问，就不需要任何同步。这种技术就是**线程封闭（Thread confinement）**。

## 栈封闭

线程封闭的一种特例，在此种限制中，只能通过本地变量才能触及对象。本地变量因为被限制在执行线程中，而其本身也存在于线程栈中，其他线程无法访问这个栈，由此实现了封闭。

## ThreadLocal

这是本文的重点。

是一种维护线程封闭更加规范的方式，它允许你**将每个线程与持有数值的对象关联在一起**。这种模式在Struts2和XWork框架中应用尤其广泛，是其核心技术。

让我们来回忆一下在传统Web开发中处理请求响应的过程。

最常用的方式就是用Servlet对象来进行Http请求的响应。关于Servlet的介绍请参见系列文章[https://github.com/leexuehan/leexuehan.github.com/issues/40](https://github.com/leexuehan/leexuehan.github.com/issues/40 "系列文章")。通过HttpServletRequest和HttpServletResponse对象，我们能轻松地与Web容器进行交互。

web收到http请求，Web容器的一个主调度线程会从事先定义好的线程池中分配一个当前的工作线程，将请求分配给当前的工作线程，由该线程来执行对应的Servlet对象中的service方法。

当这个工作线程正在工作时，Web容器可能又会收到另一个Http请求，主调度线程会同样从线程池中选择另一个工作线程来服务新的请求。

**也就是说，对于同一个Servlet的多个请求，Servlet的service方法将在一个多线程的环境下并发执行。**

**Web容器默认采用单例（单一Servlet实例）多线程的方式来处理http请求。**

如果在Servlet中定义了状态变量，则会产生线程安全的问题。

比如如下代码：

		public class ThreadUnsafe extends HttpServlet {
			private int counter = 0;
			public void doGet(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
				doPost(req,res);
			}
			
			public void doPost(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
				System.out.println(this + ":" + Thread.currentThread());
				for(int i = 0; i < 5; i++) {
					System.out.println("Counter = " + counter);
					try {
						Thread.sleep(10000);
						counter++;
					} catch (InterruptedException e) {
					}
				}
			}
		}


对于counter变量的改变就是线程不安全的。

比较容易想到的一种解决方法是在这些方法前面加一个synchronized修饰符，但是这样一来，同步问题解决了，但是效率问题暴露了，也违背了Servlet设计的初衷。

所以需要采用ThreadLocal模式来实现之。

### ThreadLocal 模式实现原理

ThreadLocal 类在维护变量时，实际使用了当前线程中的一个叫 **ThreadLocalMap** 的副本，每个线程都可以修改属于自己的副本而不会互相影响，从而实现了线程之间的隔离，避免了线程访问实例变量发生的冲突问题。

下面是JDK中源码

Thread类中的源码的一部分

	public class Thread implements Runnable {
	 	....
		ThreadLocal.ThreadLocalMap threadLocals = null;
	}

Thread类里面保存着一个ThreadLocal类中的ThreadLocalMap内部类的一个引用。

看一下，ThreadLocal类的源码

		public void set(T value) {
	        Thread t = Thread.currentThread();
	        ThreadLocalMap map = getMap(t);
	        if (map != null)
	            map.set(this, value);
	        else
	            createMap(t, value);
	    }

		public T get() {
	        Thread t = Thread.currentThread();
	        ThreadLocalMap map = getMap(t);
	        if (map != null) {
	            ThreadLocalMap.Entry e = map.getEntry(this);
	            if (e != null) {
	                @SuppressWarnings("unchecked")
	                T result = (T)e.value;
	                return result;
	            }
	        }
	        return setInitialValue();
	    }

	 	private T setInitialValue() {
	        T value = initialValue();
	        Thread t = Thread.currentThread();
	        ThreadLocalMap map = getMap(t);
	        if (map != null)
	            map.set(this, value);
	        else
	            createMap(t, value);
	        return value;
	    }

		ThreadLocalMap getMap(Thread t) {
	        return t.threadLocals;
	    }


		void createMap(Thread t, T firstValue) {
	        t.threadLocals = new ThreadLocalMap(this, firstValue);
	    }

以上只截取了部分ThreadLocal类的一部分主要代码。我们从中可以知道，它是怎么设置值和取值的。

可以看到在设置值的时候，首先是通过currentThread()方法得到当前线程的一个引用，然后通过这个引用，得到一个ThreadLocalMap 对象的引用（关于ThreadLocalMap的实现，为精简起见此处先略去了。但从字面意思我们可以看出，这是一个Map对象，键值对是当前使用它的线程和它使用的对象的引用）。如果这个Map还没有创建则调用 createMap()方法来构造一个这样的map对象。

在取值的时候，也是一样，如果有此对象则以当前线程为键值，取出对象即可，如果没有，则返回一个初始化的值。

现在总结如下：

1.ThreadLocalMap变量是属于线程的内部属性，不同的线程拥有完全不同的ThreadLocalMap变量。

2.ThreadLocalMap变量的值实在ThreadLocal对象进行set或者get操作时创建的。

3.在创建ThreadLocalMap变量之前，会首先检查当前线程中的ThreadLocalMap变量是否已经存在。

4.使用当前线程的ThreadLocalMap变量的关键在于使用当前的ThreadLocal实例作为key进行存储。

### 比较ThreadLocal 与 synchronized 关键字

1.ThreadLocal是一个Java类，通过对于当前线程中的局部变量操作来解决不同线程变量访问冲突的问题；

2.Java中的synchronized是一个保留字，它依靠JVM的锁机制来实现临界区的函数或者变量在访问中的原子性。

同步机制采用了以时间换空间的方式，提供一份变量让不同的线程排队访问；而THreadLocal采用了以空间换时间的方式为每一个线程都提供一份变量副本，从而实现同时访问互不影响。

### 一个ThreadLocal使用的例子

	package Concurrency;
	
	public class Counter {
		private static ThreadLocal<Integer> counterContext = new
				ThreadLocal<Integer>() {
			protected synchronized Integer initialValue() {
				return 10;
			}
		};
		
		public static Integer get() {
			return counterContext.get();
		}
		
		public static void set(Integer value) {
			counterContext.set(value);
		}
		
		public static Integer getNextCounter() {
			counterContext.set(counterContext.get() + 1);
			return counterContext.get();
		}
	}

**使用ThreadLocal模式最关键的地方在于创建一个任何地方都可以访问的ThreadLocal实例。**在上面的代码中，用一个静态的引用实现了这一目的。

	package Concurrency;
	
	public class ThreadLocalTest extends Thread {
		public void run() {
			for (int i = 0; i < 3; i++) {
				System.out.println("Thread[" + Thread.currentThread().getName()
						+ "],counter = " + Counter.getNextCounter());
			}
		}
	}
	
由于getNextCounter中的逻辑所操作的是 ThreadLocal 中的变量，所以无论同时有多少个线程在运行，返回的值将仅仅与当前线程的变量值相关。也就是在同一个线程中变量的值会累加。

	package Concurrency;
	
	public class Test {
		public static void main(String[] args) {
			ThreadLocalTest test1 = new ThreadLocalTest();
			ThreadLocalTest test2 = new ThreadLocalTest();
			ThreadLocalTest test3 = new ThreadLocalTest();
			test1.start();
			test2.start();
			test3.start();
		}
	}

运行上面的程序，产生了如下结果：

	Thread[Thread-2],counter = 11
	Thread[Thread-2],counter = 12
	Thread[Thread-2],counter = 13
	Thread[Thread-0],counter = 11
	Thread[Thread-0],counter = 12
	Thread[Thread-0],counter = 13
	Thread[Thread-1],counter = 11
	Thread[Thread-1],counter = 12
	Thread[Thread-1],counter = 13

验证了上面的判断。