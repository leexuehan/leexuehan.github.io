---
title: DeadLock
tags:
  - Concurrency
date: 2015-07-23 10:02:45
categories: 技术
---

### DeadLock

刚开始想着 methodA 方法用两个同步锁来实现，即成为如下形式：

		synchronized(A) {
		//Do something
		}
		
		synchronized(B) {
		//Do something
		}

methodB 方法中反过来，成为如下形式。

		synchronized(B) {
		//Do something
		}
		
		synchronized(A) {
		//Do something
		}

但是在试验失败之后，想明白了，**所谓死锁的前提就是你占用一个竞争资源，然后你还要去申请你想要的竞争资源，而对方则占用着你想要的竞争资源，去申请你现在正在占有着的资源。**也就是申请之前双方各占一个资源，这样才能形成死锁，但是如上面的形式，则在申请之前，早已经将自己的锁释放了，也就不会造成死锁了。

### Code

		package Concurrency;
		
		//写一个死锁
		public class DeadLock implements Runnable {
			private static String A = "A";
			private static String B = "B";
		
			public static void methodA() {
				synchronized (A) {
					try {
						System.out.println(A);
						Thread.sleep(2000);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("Begin to wait for B....");
					synchronized (B) {
						System.out.println(B);
					}
		
				}
		
			}
		
			public static void methodB() {
				synchronized (B) {
					System.out.println(B);
					try {
						Thread.sleep(2000);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("Begin to wait for A....");
					synchronized (A) {
						System.out.println(A);
					}
				}
		
			}
		
			@Override
			public void run() {
				String name = Thread.currentThread().getName();
				if (name.equals("TestA")) {
					methodA();
				} else {
					methodB();
				}
			}
		
			public static void main(String[] args) {
				Thread t1 = new Thread(new DeadLock());
				Thread t2 = new Thread(new DeadLock());
				t1.setName("TestA");
				t2.setName("TestB");
				t1.start();
				t2.start();
			}
		
		}

运行结果：

	B
	A
	Begin to wait for B....
	Begin to wait for A....


### Summary

正是纸上得来终觉浅，绝知此事要躬行啊。自己以前认为很简单的死锁，实现起来，也不是那么容易。

自己写一遍，理解就会更深了。

多实践才是王道。