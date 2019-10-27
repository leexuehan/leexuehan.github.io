---
title: java.util.concurrent.locks 包
tags:
  - Concurrency
date: 2015-08-17 21:47:53
categories: 技术
---

### ReadWriteLock

ReadWriteLock 维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 writer，读取锁可以由多个 reader 线程同时保持。写入锁是独占的。

与互斥锁相比，读-写锁允许对共享数据进行**更高级别的并发访问。**

虽然一次只有一个线程（writer 线程）可以修改共享数据，但在许多情况下，任何数量的线程可以同时读取共享数据（reader 线程），读-写锁利用了这一点。从理论上讲，与互斥锁相比，使用读-写锁所允许的并发性增强将带来更大的性能提高。

与互斥锁相比，使用读-写锁能否提升性能则取决于读写操作期间读取数据相对于修改数据的频率，以及数据的争用——即在同一时间试图对该数据执行读取或写入操作的线程数。

例如，某个最初用数据填充并且之后不经常对其进行修改的 collection，因为经常对其进行搜索（比如搜索某种目录），所以这样的 collection 是使用读-写锁的理想候选者。但是，如果数据更新变得频繁，数据在大部分时间都被独占锁，这时，就算存在并发性增强，也是微不足道的。更进一步地说，如果读取操作所用时间太短，则读-写锁实现（它本身就比互斥锁复杂）的开销将成为主要的执行成本，在许多读-写锁实现仍然通过一小段代码将所有线程序列化时更是如此。最终，只有通过分析和测量，才能确定应用程序是否适合使用读-写锁。


> 一句话，读取频率远大于写入频率，则用 ReadWriteLock 是最合适的，如果写入频率远大于读取频率，则互斥锁是最合适的。要分清楚情况。

API

> Lock readLock() 
>           返回用于读取操作的锁。
> 
> Lock writeLock() 
>           返回用于写入操作的锁。


其实现类为 ReentrantReadWriteLock。

### condition

	public interface Condition

Condition 将 Object 监视器方法（wait、notify 和 notifyAll）分解成截然不同的对象，以便通过将这些对象与任意 Lock 实现组合使用，为每个对象提供多个等待 set（wait-set）。其中，Lock 替代了 synchronized 方法和语句的使用，Condition 替代了 Object 监视器方法的使用。

条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。因为访问此共享状态信息发生在不同的线程中，所以它必须受保护，因此要将某种形式的锁与该条件相关联。等待提供一个条件的主要属性是：以原子方式 释放相关的锁，并挂起当前线程，就像 Object.wait 做的那样。

Condition 实例实质上被绑定到一个锁上。要为特定 Lock 实例获得 Condition 实例，请使用其 newCondition() 方法。

下面看一个例子

	class BoundedBuffer {
		   final Lock lock = new ReentrantLock();
		   final Condition notFull  = lock.newCondition(); 
		   final Condition notEmpty = lock.newCondition(); 
	
		   final Object[] items = new Object[100];
		   int putptr, takeptr, count;
	
		   public void put(Object x) throws InterruptedException {
		     lock.lock();
		     try {
		       while (count == items.length) 
		         notFull.await();
		       items[putptr] = x; 
		       if (++putptr == items.length) putptr = 0;
		       ++count;
		       notEmpty.signal();
		     } finally {
		       lock.unlock();
		     }
		   }
	
		   public Object take() throws InterruptedException {
		     lock.lock();
		     try {
		       while (count == 0) 
		         notEmpty.await();
		       Object x = items[takeptr]; 
		       if (++takeptr == items.length) takeptr = 0;
		       --count;
		       notFull.signal();
		       return x;
		     } finally {
		       lock.unlock();
		     }
		   } 
	 }

下面是Condition 的 API

		void await() throws InterruptedException;  
		  
		void awaitUninterruptibly();  
		  
		long awaitNanos(long nanosTimeout) throws InterruptedException;  
		  
		boolean await(long time, TimeUnit unit) throws InterruptedException;  
		  
		boolean awaitUntil(Date deadline) throws InterruptedException;  
		  
		void signal();  
		  
		void signalAll();  



### synchronized 与 ReentrantLock 的区别

首先，ReentrantLock 拥有Synchronized相同的并发性和内存语义，此外还多了锁投票，定时锁等候和中断锁等候。

什么意思呢？

现在线程A和B都要获取对象O的锁定，假设A首先获取了对象O锁，B将等待A释放对O的锁定，
如果使用 synchronized ，如果A不释放，B将一直等下去，不能被中断。
如果使用 ReentrantLock，如果A不释放，可以使B在等待了足够长的时间以后，中断等待，而干别的事情，或者可以通过某种设定让 B 等候固定长的时间，若是没有等到锁，不再等候。

ReentrantLock获取锁定的三种方式：

1.lock()方法

	public void lock()

如果该锁没有被另一个线程保持，则获取该锁并立即返回，将锁的保持计数设置为 1。

如果当前线程已经保持该锁，则将保持计数加 1，并且该方法立即返回。

如果该锁被另一个线程保持，则出于线程调度的目的，禁用当前线程，并且在获得锁之前，该线程将一直处于休眠状态，此时锁保持计数被设置为 1。

2.tryLock() 方法

	public boolean tryLock()

仅在调用时锁未被另一个线程保持的情况下，才获取该锁。

如果该锁没有被另一个线程保持，并且立即返回 true 值，则将锁的保持计数设置为 1。即使已将此锁设置为使用公平排序策略，但是调用 tryLock() 仍将 立即获取锁（如果有可用的），而不管其他线程当前是否正在等待该锁。在某些情况下，此“闯入”行为可能很有用，即使它会打破公平性也如此。如果希望遵守此锁的公平设置，则使用 tryLock(0, TimeUnit.SECONDS) ，它几乎是等效的（也检测中断）。

如果当前线程已经保持此锁，则将保持计数加 1，该方法将返回 true。

如果锁被另一个线程保持，则此方法将立即返回 false 值。

3.tryLock(long timeout,TimeUnit unit) 方法

		public boolean tryLock(long timeout,TimeUnit unit)
                throws InterruptedException

 如果获取了锁定立即返回true，如果别的线程正持有锁，会等待参数给定的时间，在等待的过程中，如果获取了锁定，就返回true，如果等待超时，返回false；

4.lockInterruptibly() 方法

		public void lockInterruptibly()throws InterruptedException

如果获取了锁定立即返回，如果没有获取锁定，当前线程处于休眠状态，直到或者锁定，或者当前线程被别的线程中断。


其次，synchronized是在JVM层面上实现的，不但可以通过一些监控工具监控synchronized的锁定，而且在代码执行时出现异常，JVM会自动释放锁定，但是使用Lock则不行，lock是通过代码实现的，要保证锁定一定会被释放，就必须将unLock()放到finally{}中。

再次，锁绑定条件是指一个 ReentrantLock 对象可以同时绑定多个 Condition 对象，而在 synchronized 中，锁对象的 wait(),notify(),notifyAll方法可以实现一个隐含条件，如果要和多于一个条件关联时，就不得不额外添加一个锁，而 ReentrantLock只需要调用 newCondition 即可。

最后，在资源竞争不是很激烈的情况下，Synchronized的性能要优于ReetrantLock，但是在资源竞争很激烈的情况下，Synchronized的性能会下降几十倍，但是ReetrantLock的性能能维持常态；

