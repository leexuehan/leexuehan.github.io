---
title: Java并发之生产者消费者
tags:
  - Java
date: 2015-07-15 22:30:23
categories: 技术
---

### 一个生产者消费者的多线程例子
这里基于生产者消费者的思路，模拟了银行账户的提取和存放功能，运用了同步的方法使得银行账户的金额不会受到多个线程的同时操作而陷入混乱。

### 代码
	
	package com.study;
	
	class BankCount {
		private int account;
	
		public BankCount(int account) {
			this.account = account;
		}
	
		/*
		 * 金额减少，返回剩下的金额
		 */
		public synchronized int SubAccount(int money) {
			if (account <= 500) {
				System.out.println("账户余额少，请充钱");
				try {
					wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				return -1;
			}
			this.account -= money;
			System.out.println("账户提取：" + money + ",账户剩余：" + this.account);
			return this.account;
		}
	
		/*
		 * 金额增加，并返回余下的金额
		 */
		public synchronized int AddAccount(int money) {
			while (this.account > 2000) {
				try {
					wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
			this.account += money;
			System.out.println("账户存放：" + money + ",账户剩余：" + this.account);
			notifyAll();
			return this.account;
		}
	}
	
	class Drawer implements Runnable {
		private static final int DELAY = 1000;
		private BankCount bc = null;
		private int money;
	
		public Drawer(BankCount bc, int money) {
			this.bc = bc;
			this.money = money;
		}
	
		public int WithDraw(BankCount Accounts, int money) {
			return Accounts.SubAccount(money);
		}
	
		public void run() {
			try {
				while (true) {
					WithDraw(bc, money);
					Thread.sleep(DELAY);
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	
	}
	
	class Saver implements Runnable {
		private static final int DELAY = 2000;
		private int money;
		private BankCount bc;
	
		public Saver(BankCount bc, int money) {
			this.money = money;
			this.bc = bc;
		}
	
		public int SaveAccounts(BankCount Accounts, int money) {
			return Accounts.AddAccount(money);
		}
	
		public void run() {
			try {
				while (true) {
					SaveAccounts(bc, money);
					Thread.sleep(DELAY);
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
	
	public class ThreadStudy {
		public static void main(String[] args) {
			BankCount bc = new BankCount(500);
			Saver saver = new Saver(bc, 500);
			Drawer drawer = new Drawer(bc, 500);
			
			//设置两个存储线程，三个提取线程
			Thread s1 = new Thread(saver);
			Thread s2 = new Thread(saver);
			
			Thread d1 = new Thread(drawer);
			Thread d2 = new Thread(drawer);
			Thread d3 = new Thread(drawer);
			
			//先启动存储线程
			s1.start();
			s2.start();
			
			try {
				Thread.sleep(1000);
			}catch (InterruptedException e) {
				e.printStackTrace();
			}
			
			//之后启动提取线程
			d1.start();
			d2.start();
			d3.start();
		}
	}

### 总结

用一个典型的生产者-消费者例子，用来作为学习多线程的开始，就像hello world之于所有编程语言一样。之后会尽量分析写一些经典的例子来学习Java并发，少一些知识点的堆积。这样可能会使自己体会更深一点吧！