---
title: Condition 学习
tags:
  - Concurrency
date: 2015-09-01 21:26:37
categories: 技术
---

二话不说，先上代码。

### 代码

		package com.java;
		
		import java.util.concurrent.locks.Condition;
		import java.util.concurrent.locks.Lock;
		import java.util.concurrent.locks.ReentrantLock;
		
		public class ConditionStudy {
			static class NumberWrapper {
				int value = 1;
			}
		
			public static void main(String[] args) {
				final Lock lock = new ReentrantLock();
				final Condition reachThreeCondition = lock.newCondition();
				final Condition reachSixCondition = lock.newCondition();
				final NumberWrapper num = new NumberWrapper();
				Thread threadA = new Thread(new Runnable() {
					@Override
					public void run() {
						try {
							lock.lock();
							System.out.println("ThreadA starts to write...");
							while (num.value <= 3) {
								System.out.println(num.value);
								num.value++;
							}
							reachThreeCondition.signal();//在 A 线程中的 value 大于 3 之后，通过信号量 reachThreeCondition 通知 B 线程
						} finally {
							lock.unlock();
						}
		
						
						try {
							lock.lock();
							reachSixCondition.await();
							System.out.println("ThreadA starts to write...");
							while (num.value <= 9) {
								System.out.println(num.value);
								num.value++;
							}
		
						} catch (InterruptedException e) {
							e.printStackTrace();
						} finally {
							lock.unlock();
						}
		
					}
				});
		
				Thread threadB = new Thread(new Runnable() {
					@Override
					public void run() {
						try {
							lock.lock();
							System.out.println("ThreadB starts to write...");
							while (num.value <= 3) {
								reachThreeCondition.await(); //B 线程在value值大于3之前一直在等 reachThreeCondition 信号量
							}
						} catch (InterruptedException e) {
							e.printStackTrace();
						} finally {
							lock.unlock();
						}
		
						try {
							lock.lock();
							System.out.println("ThreadB starts to write...");
							while (num.value <= 6) {
								System.out.println(num.value);
								num.value++;
							}
							reachSixCondition.signal();
						} finally {
							lock.unlock();
						}
					}
				});
		
				threadA.start();
				threadB.start();
			}
		}

运行结果：

		ThreadA starts to write...
		1
		2
		3
		ThreadB starts to write...
		ThreadB starts to write...
		4
		5
		6
		ThreadA starts to write...
		7
		8
		9

### 分析

可以看出来这个程序的目的是让线程 A 和线程 B 接力把 1 到 9 打印出来。
而 Condition 就像一个信件一样，起到使两个线程互相沟通的作用。
在线程 A 在执行任务时， B 通过 await 方法翘首以盼， A 执行完任务后，通过signal 方法传达到 B线程。
以此类推，直到 A 和 B将任务完成。

上面提到了线程之间如何沟通的问题，但是同时我们知道，要想使两个线程互斥，则必须有锁，所以说lock与Condition之间是分不开的。

如何通过lock获取锁呢？

JDK文档里面如是说：

> Condition 实例实质上被绑定到一个锁上。要为特定 Lock 实例获得 Condition 实例，请使用其 newCondition() 方法。

 
上面是网上一位大神写的例子，下面看看在官方给出的例子：

	class BoundedBuffer {
		   final Lock lock = new ReentrantLock();
		   final Condition notFull  = lock.newCondition();//获得两个信号量 
		   final Condition notEmpty = lock.newCondition(); 
		
		   final Object[] items = new Object[100];//开辟一个大小为100的数组
		   int putptr, takeptr, count;//三个分别代表下一个要存放的位置，下一个要取出的位置，目前已经存放的元素总数
		
		   public void put(Object x) throws InterruptedException {
		     lock.lock();
		     try {
		       while (count == items.length) 
		         	notFull.await();//很简单，当满了的时候，就要等“不满”的消息
		       items[putptr] = x; 
		       if (++putptr == items.length) 
					putptr = 0;
		       ++count;
		       notEmpty.signal();//当有元素存放的时候向take方法发送“已经有了元素”的消息
		     } finally {
		       lock.unlock();
		     }
		   }
		
		   public Object take() throws InterruptedException {
		     lock.lock();
		     try {
		       while (count == 0) 
		         	notEmpty.await();//当元素个数为0的时候，阻塞等待“不空”的消息
		       Object x = items[takeptr]; 
		       if (++takeptr == items.length) 
					takeptr = 0;
		       --count;
		       notFull.signal(); //当取走一个元素的时候，向put方法发送“不满”消息
		       return x;
		     } finally {
		       lock.unlock();
		     }
		   } 
	 }

很简单的一个例子，但却使用到了几乎所有的并发控制的方法。


### lock vs synchronized

这里主要介绍用 lock 和 synchronized 进行同步的区别：

java.util.concurrent.lock 中的 Lock 框架是锁定的一个抽象，ReentrantLock 类实现了 Lock ，它拥有与 synchronized 相同的并发性和内存语义，**但是添加了类似轮询锁、定时锁等候和可中断锁等候的一些特性。此外，它还提供了在激烈争用情况下更佳的性能。**

1.synchronized 进行同步无法中断一个正在等候获得锁的线程，也无法通过轮询得到锁，如果不想等下去，也就没法得到锁并且同步还要求锁的释放只能在与获得锁所在的堆栈帧相同的堆栈帧中进行。

2.lock 必须在 finally 块中释放。否则，如果受保护的代码将抛出异常，锁就有可能永远得不到释放！这一点区别看起来可能没什么，但是实际上，它极为重要。忘记在 finally 块中释放锁，可能会在程序中留下一个定时炸弹，当有一天炸弹爆炸时，您要花费很大力气才有找到源头在哪；而使用同步，JVM 将确保锁会获得自动释放。

3.与目前的 synchronized 实现相比，争用下的 ReentrantLock 实现更具可伸缩性。（在未来的 JVM 版本中，synchronized 的争用性能很有可能会获得提高。）这意味着当许多线程都在争用同一个锁时，使用 ReentrantLock 的总体开支通常要比 synchronized 少得多。

4.为了在对象上 wait 或 notify ，您必须持有该对象的锁。就像 Lock 是同步的概括一样， Lock 框架包含了对 wait 和 notify 的概括，这个概括叫作 条件（Condition） 。 Lock 对象则充当绑定到这个锁的条件变量的工厂对象，与标准的 wait 和 notify 方法不同，对于指定的 Lock ，可以有不止一个条件变量与它关联。这样就简化了许多并发算法的开发。


> 例如， 条件（Condition） 的 Javadoc 显示了一个有界缓冲区实现的示例，该示例使用了两个条件变量，“not full”和“not empty”，它比每个 lock 只用一个 wait 设置的实现方式可读性要好一些（而且更有效）。 Condition 的方法与 wait 、 notify 和 notifyAll 方法类似，分别命名为 await 、 signal 和 signalAll ，因为它们不能覆盖 Object 上的对应方法。

5.ReentrantLock 构造器的一个参数是 boolean 值，它允许您选择想要一个 公平（fair）锁，还是一个 不公平（unfair）锁。

公平锁使线程按照请求锁的顺序依次获得锁；而不公平锁则允许直接获取锁，在这种情况下，线程有时可以比先请求锁的其他线程先得到锁。

那么同步又如何呢？内置的监控器锁是公平的吗？答案令许多人感到大吃一惊，它们是不公平的，而且永远都是不公平的。

但是没有人抱怨过线程饥渴，因为 JVM 保证了所有线程最终都会得到它们所等候的锁。确保统计上的公平性，对多数情况来说，这就已经足够了，而这花费的成本则要比绝对的公平保证的低得多。

所以，默认情况下 ReentrantLock 是“不公平”的，这一事实只是把同步中一直是事件的东西表面化而已。如果您在同步的时候并不介意这一点，那么在 ReentrantLock 时也不必为它担心。

### 什么时候使用同步？什么时候使用lock？

虽然 ReentrantLock 是个非常动人的实现，相对 synchronized 来说，它有一些重要的优势，但是我认为急于把 synchronized 视若敝屣，绝对是个严重的错误。 java.util.concurrent.lock 中的锁定类是用于高级用户和高级情况的工具 。

**一般来说，除非您对 Lock 的某个高级特性有明确的需要，或者有明确的证据（而不是仅仅是怀疑）表明在特定情况下，同步已经成为可伸缩性的瓶颈，否则还是应当继续使用 synchronized。**

因为对于 java.util.concurrent.lock 中的锁定类来说，synchronized 仍然有一些优势。

比如，在使用 synchronized 的时候，不能忘记释放锁；在退出 synchronized 块时，JVM 会为您做这件事。您很容易忘记用 finally 块释放锁，这对程序非常有害。您的程序能够通过测试，但会在实际工作中出现死锁，那时会很难指出原因（这也是为什么根本不让初级开发人员使用 Lock 的一个好理由。）

既然如此，我们什么时候才应该使用 ReentrantLock 呢？

答案非常简单 —— 在确实需要一些 synchronized 所没有的特性的时候，比如时间锁等候、可中断锁等候、无块结构锁、多个条件变量或者轮询锁。 ReentrantLock 还具有可伸缩性的好处，应当在高度争用的情况下使用它，但是请记住，大多数 synchronized 块几乎从来没有出现过争用，所以可以把高度争用放在一边。

我建议用 synchronized 开发，直到确实证明 synchronized 不合适，而不要仅仅是假设如果使用 ReentrantLock “性能会更好”。

请记住，这些是供高级用户使用的高级工具。（而且，真正的高级用户喜欢选择能够找到的最简单工具，直到他们认为简单的工具不适用为止。）。一如既往，首先要把事情做好，然后再考虑是不是有必要做得更快。