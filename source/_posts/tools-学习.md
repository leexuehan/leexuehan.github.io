---
title: tools 学习
tags:
  - Concurrency
date: 2015-09-02 21:36:21
categories: 技术
---

### CountDownLatch

	public class CountDownLatch
	      extends 
	      Object

一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。

用给定的计数 初始化 CountDownLatch。由于调用了 countDown() 方法，所以在当前计数到达零之前，await 方法会一直受阻塞。之后，会释放所有等待的线程，await 的所有后续调用都将立即返回。

CountDownLatch 的一个有用特性是，**它不要求调用 countDown 方法的线程等到计数到达零时才继续，而在所有线程都能通过之前，它只是阻止任何线程继续通过一个 await。**

方法摘要：


	 void	await() 
	        //使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断。
	 boolean	await(long timeout, TimeUnit unit) 
	        //使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断或超出了指定的等待时间。
	 void	countDown() 
	        //递减锁存器的计数，如果计数到达零，则释放所有等待的线程。
	 long	getCount() 
	        //返回当前计数。
	 String	toString() 
	         //返回标识此锁存器及其状态的字符串。
示例用法：

	package com.java;

	import java.util.concurrent.CountDownLatch;
	
	public class TestCountDownLatch {
		private final static int N = 10;
		public static void main(String[] args) throws InterruptedException {
			CountDownLatch startSignal = new CountDownLatch(1);
			CountDownLatch doneSignal = new CountDownLatch(N);
			
			for(int i = 0; i < N; i++) {
				new Thread(new Worker(startSignal, doneSignal)).start();
			}
			
			doSomethingElse ();
			startSignal.countDown();
			doSomethingElse ();
			doneSignal.await();
			doSomethingElse ();
		}
		
		public static void doSomethingElse () {
			System.out.println("Doing something else");
		}
	}
	
	class Worker implements Runnable {
		private final CountDownLatch startSignal;
		private final CountDownLatch doneSignal;
		
		public Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
			this.startSignal = startSignal;
			this.doneSignal = doneSignal;
		}
		
		@Override
		public void run() {
			try {
				startSignal.await();
				doWork();
				doneSignal.countDown();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		
		public void doWork() {
			System.out.println("Worker is working...");
		}
	}

运行结果：

		Doing something else
		Doing something else
		Worker is working...
		Worker is working...
		Worker is working...
		Worker is working...
		Worker is working...
		Worker is working...
		Worker is working...
		Worker is working...
		Worker is working...
		Worker is working...
		Doing something else

分析：

因为在 Worker类的 run 方法调用了 await 方法，所以会在 startSignal 计数器的值为 0 之前，一直阻塞。
同理，在 main 方法里面也调用了 doneSignal 的await 方法，所以会在 doneSignal 的计数器归零之前，一直阻塞。所以当 10 个线程都运行完毕之后，计数器也归零，主线程进入就绪状态，打印出了最后一条语句。

另外，在JDK文档中，还举了另外一种用法，就是：

> 将一个问题分成 N 个部分，用执行每个部分并让锁存器倒计数的 Runnable 来描述每个部分，然后将所有 Runnable 加入到 Executor 队列。当所有的子部分完成后，协调线程就能够通过 await。

代码如下：

		package com.java;
		
		import java.util.concurrent.ArrayBlockingQueue;
		import java.util.concurrent.BlockingQueue;
		import java.util.concurrent.CountDownLatch;
		import java.util.concurrent.Executor;
		import java.util.concurrent.ThreadPoolExecutor;
		import java.util.concurrent.TimeUnit;
		
		public class TestCountDownLatch {
			private final static int N = 10;
		
			public static void main(String[] args) throws InterruptedException {
				CountDownLatch startSignal = new CountDownLatch(1);
				CountDownLatch doneSignal = new CountDownLatch(N);
				BlockingQueue<Runnable> bq = new ArrayBlockingQueue<Runnable>(10);
				Executor e = new ThreadPoolExecutor(3, 5, 50, TimeUnit.MILLISECONDS, bq);
		
				for (int i = 0; i < N; i++) {
					e.execute(new Worker(i, doneSignal));
				}
		
				doSomethingElse();
				startSignal.countDown();
				System.out.println("main thread is waiting...");
				doneSignal.await();
				doSomethingElse();
			}
		
			public static void doSomethingElse() {
				System.out.println("Doing something else");
			}
		}
		
		class Worker implements Runnable {
			private final int i;
			private final CountDownLatch doneSignal;
		
			public Worker(int i, CountDownLatch doneSignal) {
				this.i = i;
				this.doneSignal = doneSignal;
			}
		
			@Override
			public void run() {
				try {
					doWork(i);
					System.out.println("The Current Count is : " + doneSignal.getCount());
					doneSignal.countDown();
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		
			public void doWork(int i) {
				System.out.println(Thread.currentThread().getName()
						+ " Worker is working at " + i);
			}
		}

运行结果：

		Doing something else
		main thread is waiting...
		pool-1-thread-2 Worker is working at 1
		The Current Count is : 10
		pool-1-thread-2 Worker is working at 3
		The Current Count is : 9
		pool-1-thread-2 Worker is working at 4
		The Current Count is : 8
		pool-1-thread-2 Worker is working at 5
		The Current Count is : 7
		pool-1-thread-2 Worker is working at 6
		The Current Count is : 6
		pool-1-thread-2 Worker is working at 7
		The Current Count is : 5
		pool-1-thread-2 Worker is working at 8
		The Current Count is : 4
		pool-1-thread-2 Worker is working at 9
		The Current Count is : 3
		pool-1-thread-1 Worker is working at 0
		The Current Count is : 2
		pool-1-thread-3 Worker is working at 2
		The Current Count is : 1
		Doing something else

可以看到在计数值为 0 的时候，主动权又一次交给了主线程。

### Executors

注意：这里的Executors 和 Executor 不要搞混了，Executor 是一个线程执行接口，而这里的 Executors 只是一个普通的类，与 Executor 没有任何关系。

		public class Executors
		      extends 
		      Object

从声明中也可以看出来，Executors 就是一个继承 Object 的普通的类。

重要方法摘要：

		static Callable<Object>	callable(Runnable task) 
		          返回 Callable 对象，调用它时可运行给定的任务并返回 null。
		static <T> Callable<T>	callable(Runnable task, T result) 
		          返回 Callable 对象，调用它时可运行给定的任务并返回给定的结果。

		static ThreadFactory	defaultThreadFactory() 
		          返回用于创建新线程的默认线程工厂。

		static ExecutorService	newCachedThreadPool() 
		          创建一个可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们。
		static ExecutorService	newCachedThreadPool(ThreadFactory threadFactory) 
		          创建一个可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们，并在需要时使用提供的 ThreadFactory 创建新线程。
		static ExecutorService	newFixedThreadPool(int nThreads) 
		          创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程。
		static ExecutorService	newFixedThreadPool(int nThreads, ThreadFactory threadFactory) 
		          创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程，在需要时使用提供的 ThreadFactory 创建新线程。

		static ScheduledExecutorService	newScheduledThreadPool(int corePoolSize) 
		          创建一个线程池，它可安排在给定延迟后运行命令或者定期地执行。
		static ScheduledExecutorService	newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory) 
		          创建一个线程池，它可安排在给定延迟后运行命令或者定期地执行。

		static ExecutorService	newSingleThreadExecutor() 
		          创建一个使用单个 worker 线程的 Executor，以无界队列方式来运行该线程。
		static ExecutorService	newSingleThreadExecutor(ThreadFactory threadFactory) 
		          创建一个使用单个 worker 线程的 Executor，以无界队列方式来运行该线程，并在需要时使用提供的 ThreadFactory 创建新线程。
		static ScheduledExecutorService	newSingleThreadScheduledExecutor() 
		          创建一个单线程执行程序，它可安排在给定延迟后运行命令或者定期地执行。
		static ScheduledExecutorService	newSingleThreadScheduledExecutor(ThreadFactory threadFactory) 
		          创建一个单线程执行程序，它可安排在给定延迟后运行命令或者定期地执行。
		

### semaphore

		public class Semaphore
		      extends 
		      Object
		      implements 
		      Serializable

一个计数信号量.

Semaphore 通常用于限制可以访问某些资源（物理或逻辑的）的线程数目。

Semaphore当前在多线程环境下被扩放使用，操作系统的信号量是个很重要的概念，在进程控制方面都有应用。

Java 并发库 的Semaphore 可以很轻松完成信号量控制，Semaphore可以控制某个资源可被同时访问的个数，通过 acquire() 获取一个许可，如果没有就等待，而 release() 释放一个许可。

构造方法：

		Semaphore(int permits) 
		          //创建具有给定的许可数和非公平的公平设置的 Semaphore。
		Semaphore(int permits, boolean fair) 
		          //创建具有给定的许可数和给定的公平设置的 Semaphore。

重要方法：


	 void	acquire() 
	          从此信号量获取一个许可，在提供一个许可前一直将线程阻塞，否则线程被中断。
	 void	acquire(int permits) 
	          从此信号量获取给定数目的许可，在提供这些许可前一直将线程阻塞，或者线程已被中断。

	 protected  void reducePermits(int reduction) 
          根据指定的缩减量减小可用许可的数目。
	
	 boolean	tryAcquire() 
	          仅在调用时此信号量存在一个可用许可，才从信号量获取许可。
	 boolean	tryAcquire(int permits) 
	          仅在调用时此信号量中有给定数目的许可时，才从此信号量中获取这些许可。
	 boolean	tryAcquire(int permits, long timeout, TimeUnit unit) 
	          如果在给定的等待时间内此信号量有可用的所有许可，并且当前线程未被中断，则从此信号量获取给定数目的许可。
	 boolean	tryAcquire(long timeout, TimeUnit unit) 
	          如果在给定的等待时间内，此信号量有可用的许可并且当前线程未被中断，则从此信号量获取一个许可。
	 

	 void	release() 
	          释放一个许可，将其返回给信号量。
	 void	release(int permits) 
	          释放给定数目的许可，将其返回到信号量。


使用例子：

下面的类使用信号量控制对内容池的访问：

		package com.java;

		import java.util.concurrent.ExecutorService;
		import java.util.concurrent.Executors;
		import java.util.concurrent.Semaphore;
		
		public class TestSemaphore {
			public static final int MAX_AVAILABLE = 5;
			private static final Semaphore available = new Semaphore(MAX_AVAILABLE, true);
		
			public static void main(String[] args) {
				ExecutorService exec = Executors.newCachedThreadPool();
				for(int i = 0; i < MAX_AVAILABLE + 5; i++) {
					exec.execute(new Runnable() {
						public void run() {
							try {
								available.acquire();
								System.out.println("线程" + Thread.currentThread().getName()
										+ " 正在访问.... ");
								  Thread.sleep((long) (Math.random() * 10000));
								available.release();
								System.out.println("------可用资源-----------"+available.availablePermits());
							} catch (InterruptedException e) {
								e.printStackTrace();
							}
						}
					});
				}
				exec.shutdown();
			}
		
		}

运行结果：

		线程pool-1-thread-1 正在访问.... 
		线程pool-1-thread-3 正在访问.... 
		线程pool-1-thread-2 正在访问.... 
		线程pool-1-thread-4 正在访问.... 
		线程pool-1-thread-6 正在访问.... 
		------可用资源-----------1
		线程pool-1-thread-8 正在访问.... 
		------可用资源-----------1
		线程pool-1-thread-10 正在访问.... 
		------可用资源-----------1
		线程pool-1-thread-5 正在访问.... 
		------可用资源-----------1
		线程pool-1-thread-7 正在访问.... 
		------可用资源-----------1
		线程pool-1-thread-9 正在访问.... 
		------可用资源-----------1
		------可用资源-----------2
		------可用资源-----------3
		------可用资源-----------4
		------可用资源-----------5

### exchanger

		public class Exchanger<V>
		      extends 
		      Object


Exchanger可以在两个线程之间交换数据，只能是2个线程，他不支持更多的线程之间互换数据。

当线程A调用Exchange对象的exchange()方法后，他会陷入阻塞状态，直到线程B也调用了exchange()方法，然后以线程安全的方式交换数据，之后线程A和B继续运行。

	 V	exchange(V x); 
	          //等待另一个线程到达此交换点（除非当前线程被中断），然后将给定的对象传送给该线程，并接收该线程的对象。
	 V	exchange(V x, long timeout, TimeUnit unit); 
	          //等待另一个线程到达此交换点（除非当前线程被中断，或者超出了指定的等待时间），然后将给定的对象传送给该线程，同时接收该线程的对象。


使用实例：

		package com.java;
		
		import java.util.ArrayList;
		import java.util.List;
		import java.util.concurrent.Exchanger;
		
		public class TestExchanger {
			public static void main(String[] args) {
				Exchanger<List<Integer>> exchanger = new Exchanger<List<Integer>>();
				new Producer(exchanger).start();
				new Consumer(exchanger).start();
			}
		}
		
		class Producer extends Thread {
			List<Integer> list = new ArrayList<Integer>();
			Exchanger<List<Integer>> exchanger = null;
			public Producer(Exchanger<List<Integer>> exchanger) {
				super();
				this.exchanger = exchanger;
			}
			@Override
			public void run(){
			
				for(int i = 0; i < 10; i++) {
					list.add(i+1);
				}
				
				try {
					list = exchanger.exchange(list);
				} catch(InterruptedException e) {
					e.printStackTrace();
				}
				
				
			}
			
		}
		
		class Consumer extends Thread {
			List<Integer> list = new ArrayList<Integer>();
			Exchanger<List<Integer>> exchanger = null;
			public Consumer(Exchanger<List<Integer>> exchanger) {
				super();
				this.exchanger = exchanger;
			}
			@Override
			public void run(){
				try {
					list = exchanger.exchange(list);
				} catch(InterruptedException e) {
					e.printStackTrace();
				}
				
				for(int i = 0; i < 10; i++) {
					System.out.print(list.get(i) + " ");
				}
				System.out.println();
				
			}
		}

运行结果：

	1 2 3 4 5 6 7 8 9 10 

可以看出来，Consumer线程收到的数据确实是 Producer 发送的。


### CyclicBarrier

		public class CyclicBarrier
		      extends 
		      Object
可以看出来，这是个普通的类。

一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)。

在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待，此时 CyclicBarrier 很有用。因为该 barrier 在释放等待线程后可以重用，所以称它为循环 的 barrier。


构造函数：

		CyclicBarrier(int parties) 
		          创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动，但它不会在启动 barrier 时执行预定义的操作。
		CyclicBarrier(int parties, Runnable barrierAction) 
		          创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动，并在启动 barrier 时执行给定的屏障操作，该操作由最后一个进入 barrier 的线程执行。


方法摘要：

	 int	await() 
	          在所有参与者都已经在此 barrier 上调用 await 方法之前，将一直等待。
	 int	await(long timeout, TimeUnit unit) 
	          在所有参与者都已经在此屏障上调用 await 方法之前将一直等待,或者超出了指定的等待时间。
	 int	getNumberWaiting() 
	          返回当前在屏障处等待的参与者数目。
	 int	getParties() 
	          返回要求启动此 barrier 的参与者数目。
	 boolean	isBroken() 
	          查询此屏障是否处于损坏状态。
	 void	reset() 
	          将屏障重置为其初始状态。

代码实例：

		package com.java;
		
		import java.io.IOException;
		import java.util.Random;
		import java.util.concurrent.BrokenBarrierException;
		import java.util.concurrent.CyclicBarrier;
		import java.util.concurrent.ExecutorService;
		import java.util.concurrent.Executors;
		
		public class TestCyclicBarrier {
			public static void main(String[] args) throws IOException, InterruptedException {  
		        //如果将参数改为4，但是下面只加入了3个选手，这永远等待下去  
		        //Waits until all parties have invoked await on this barrier.   
		        CyclicBarrier barrier = new CyclicBarrier(3, new ReadyAction());  
		  
		        ExecutorService executor = Executors.newFixedThreadPool(3);
		        System.out.println("各就各位。。。。");
		        executor.submit(new Thread(new Runner(barrier, "1号选手")));  
		        executor.submit(new Thread(new Runner(barrier, "2号选手")));  
		        executor.submit(new Thread(new Runner(barrier, "3号选手")));  
		  
		        executor.shutdown();  
		    }  
		}
		
		class ReadyAction implements Runnable {
			@Override
			public void run() {
				System.out.println("枪响。。。。");
			}
		}
		  
		class Runner implements Runnable {  
		    // 一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)  
		    private CyclicBarrier barrier;  
		  
		    private String name;  
		  
		    public Runner(CyclicBarrier barrier, String name) {  
		        super();  
		        this.barrier = barrier;  
		        this.name = name;  
		    }  
		  
		    @Override  
		    public void run() {  
		        try {  
		            Thread.sleep(1000 * (new Random()).nextInt(8));  
		            System.out.println(name + " 准备好了...");  
		            // barrier的await方法，在所有参与者都已经在此 barrier 上调用 await 方法之前，将一直等待。  
		            barrier.await();  
		        } catch (InterruptedException e) {  
		            e.printStackTrace();  
		        } catch (BrokenBarrierException e) {  
		            e.printStackTrace();  
		        }  
		        System.out.println(name + " 起跑！");  
		    }  
		}  


运行结果：

		各就各位。。。。
		1号选手 准备好了...
		2号选手 准备好了...
		3号选手 准备好了...
		枪响。。。。
		3号选手 起跑！
		1号选手 起跑！
		2号选手 起跑！

### 后记


至此，java.util.concurrent.tools 包就完全介绍完了，同时还配有简单的示例，至于源码分析，在后面会陆续完善。