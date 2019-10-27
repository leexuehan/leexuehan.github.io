---
title: 删除链表中倒数第k个结点
tags:
  - Algorithm
date: 2015-07-14 16:00:59
categories: 技术
---

## 剑指offer面试题系列


> 题目：输入一个链表，输出该链表的倒数第k个结点。为了符合大多数人的习惯，本题从1开始计数，即链表的尾节点是倒数第一个结点。

### 分析

此题如果按照常规思路来解，则是先将整个链表遍历完，求出链表的长度，然后再重头遍历一次，同时设定一个计数器，计数器 = 链表长度 - k，就停止，然后将其删除即可。

但是，还有更简洁的方法：设定两个指针，先让一个指针遍历 k 步，然后另一个指针开始从头遍历，让前面的指针指向尾节点的时候，后面的指针指的就是那个要删除的节点。这种方法只需要遍历一次即可完成。

### 代码

	public static void deleteLastKNode(ListNode head, int k) {
			if (head == null || k <= 0) {
				System.out.println("Input Invalid!");
				return;
			}
	
			int count = 0;
			ListNode p = head.next;
			while (p != null && count < k) {
				p = p.next;
				count++;
			}
	
			if (count < k) {
				System.out.println("the number k is illegal!");
				return;
			}
	
			ListNode q = head.next; //q指向待删除的结点
			while (p != null) {
				p = p.next;
				q = q.next;
			}
	
			q.data = q.next.data;
			q.next = q.next.next;
		}

### 备注

需要注意代码的健壮性，故需要检查输入的指针是否为空，数字是否合法，链表的长度和k是否构成冲突。

相关题目：

1.求链表的中间结点；

2.判断单链表是否有环形结构。