---
title: Atomic 学习
tags:
  - Concurrency
date: 2015-09-04 10:09:18
categories: 技术
---

首先声明：http://blog.csdn.net/hsuxu/article/details/9467651 这篇文章写得特别详实，看了好多博客，只有这一篇是从硬件底层过来解释的。

眼前有景道不得，崔颢题诗在上头。

虽然，有这样一篇博客足矣，但是一来觉得，还需要自己消化，二来，个人觉得这篇文章的排版不是很工整。出于这个目的，写了这篇博客。


### 概述

这个包里面提供了一组**原子变量**类。

Java从JDK1.5开始提供了java.util.concurrent.atomic包，方便程序员在多线程环境下，无锁的进行原子操作。

其基本的特性就是在多线程环境下，当有多个线程同时执行这些类的实例包含的方法时，具有排他性，即当某个线程进入方法，执行其中的指令时，不会被其他线程打断，而别的线程就像自旋锁一样，一直等到该方法执行完成，才由JVM从等待队列中选择一个另一个线程进入，这只是一种逻辑上的理解。实际上是借助硬件的相关指令来实现的，不会阻塞线程(或者说只是在硬件级别上阻塞了)。

原子变量的底层使用了处理器提供的原子指令，但是不同的CPU架构可能提供的原子指令不一样，也有可能需要某种形式的内部锁,所以该方法不能绝对保证线程不被阻塞。

原子变量类相当于一种**泛化的volatile变量**，能够支持原子的和有条件的读-改-写操作。

在了解具体实现之前，有必要先知道 CAS 的知识，因为它是整个包的前提。
关于CAS，此处只介绍一句话：


> CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

更多关于 CAS 的知识请参见本人其他博客。


### 具体介绍

java.util.concurrent.atomic中的类可以分成4组：

- 标量类（Scalar）：AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference
- 数组类：AtomicIntegerArray，AtomicLongArray，AtomicReferenceArray
- 更新器类：AtomicLongFieldUpdater，AtomicIntegerFieldUpdater，AtomicReferenceFieldUpdater
- 复合变量类：AtomicMarkableReference，AtomicStampedReference

第一组 AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference这四种基本类型用来处理布尔，整数，长整数，对象四种数据，其内部实现不是简单的使用synchronized，而是一个更为高效的方式CAS (compare and swap) + volatile和native方法，从而避免了synchronized的高开销，执行效率大为提升。

此处仅以 AtomicInteger 作为第一组的代表介绍：

AtomicInteger 常用方法如下：

- int addAndGet(int delta) ：以原子方式将输入的数值与实例中的值（AtomicInteger里的value）相加，并返回结果
- boolean compareAndSet(int expect, int update) ：如果输入的数值等于预期值，则以原子方式将该值设置为输入的值。

		public final boolean compareAndSet(int expect, int update) {
			return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
	    }

- int getAndIncrement()：以原子方式将当前值加1，注意：这里返回的是自增前的值。
- void lazySet(int newValue)：最终会设置成newValue，使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。


	 	/**
	     * Atomically adds the given value to the current value.
	     * @param delta the value to add
	     * @return the updated value
	     */
	    public final int addAndGet(int delta) {
	        for (;;) {
	            int current = get();
	            int next = current + delta;
	            if (compareAndSet(current, next))
	                return next;
	        }
	    }


当没有使用 AtomicInteger时，仅用 valatile 关键字，并不能保证内存同步，这个在之前也做过介绍。

下面请看自己亲测的一个例子：

		package lee;
		
		import java.util.concurrent.atomic.AtomicInteger;
		
		public class Atomic {
			private static volatile int i = 0;
			private static final int THREAD_NUM = 50;
			public static void increase() {
				i++;
			}
			public static void main(String[] args) {
				// AtomicInteger ai = new AtomicInteger();
				Thread[] threads = new Thread[THREAD_NUM];
				for(int i = 0; i < THREAD_NUM; i++) {
					threads[i] = new Thread(new Runnable() {
						public void run() {
							for(int i = 0; i < 100000; i++) {
								increase();
							}
						}
					});
					threads[i].start();
				}
				
		
				while( Thread.activeCount() > 1)
					Thread.yield();
				System.out.println("累加的结果是：" + i);
			}
		
		}

开启 50 个线程，每个线程都对int型整数 i 加10000次，期望结果就是 500000。但是结果呢？

		累加的结果是：4394704

显然与预期结果相差还不少。原因就在于i++这个操作不具有原子性。

那么采用 AtomicInteger 包装之后呢？


		package lee;
		
		import java.util.concurrent.atomic.AtomicInteger;
		
		public class Atomic {
			private static AtomicInteger i = new AtomicInteger(0);
			private static final int THREAD_NUM = 50;
			public static void increase() {
				i.getAndIncrement();
			}
			public static void main(String[] args) {
				// AtomicInteger ai = new AtomicInteger();
				Thread[] threads = new Thread[THREAD_NUM];
				for(int i = 0; i < THREAD_NUM; i++) {
					threads[i] = new Thread(new Runnable() {
						public void run() {
							for(int i = 0; i < 100000; i++) {
								increase();
							}
						}
					});
					threads[i].start();
				}
				
		
				while( Thread.activeCount() > 1)
					Thread.yield();
				System.out.println("累加的结果是：" + i);
			}
		
		}

结果是：

	累加的结果是：5000000


可以看出来 getAndIncrement() 这个方法确实具有原子性。

看一下它的源码是怎么实现的？

	public final int getAndIncrement() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return current;
        }
    }


此方法在一个无限循环中，不断尝试将一个比当前值大1的新值赋给自己，如果失败了，那说明在执行“获取-设置”操作时，已经有了修改，于是再次循环进行下一步操作，直到设置成功为止。
