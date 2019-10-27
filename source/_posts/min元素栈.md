---
title: min元素栈
tags:
  - Algorithm
date: 2015-07-16 10:53:54
categories: 技术
---

## 剑指offer面试题系列


> 题目：定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素 min 函数，在该栈中，调用 min，push，pop 的时间复杂度都是 O(1).

### 分析

刚拿到题的时候，想了半天，觉得如果用一个栈来实现，怎么也不可能达到调用min的时间复杂度为O(1),于是考虑建立一个辅助栈来存放min元素。就在这里没有想通，当时这么想：即使用一个辅助栈来存放最小的元素，那岂不意味着每次向主栈压入一个元素时，都需要把该元素压入辅助站的适当位置？然后出栈时也面临这样一个问题：如果从出栈中弹出一个元素，那么辅助栈中也需要将相应的元素弹出来。（当时想着主栈和辅助栈中的元素应该是一对一的关系，即辅助站中存放的是排序后的主栈元素，真笨！）

其实，完全没有必要这么麻烦的实现。**在主栈中设置一个成员变量来保存当前栈的最小元素，每次向主栈压入元素时，只需要比较这个元素和最小值的大小即可，如果小于最小值，则更新之，如果没有小于最小值，则在辅助栈中再次压入一个最小值元素即可。**

出栈时，只需要主栈中弹出一个元素，辅助栈中也弹出一个元素即可。

**想起一个不太恰当的比喻：决定一个团队的上限，永远是精英人物，哈哈。精英人物少一个，团队层次就下降一档**



### 代码实现

		package com.study;
		
		/*
		 * 最小元素栈
		 * 要求pop/push/min的算法复杂度均为O(1)
		 * */
		
		class AuxiliaryStack {  //一个辅助栈
			private int top = 0;
			private static final int MAXSIZE = 10;
			private int[] arr = new int[MAXSIZE];
		
			public int getTopElement() {
				if (top - 1 < 0) {
					System.out.println("Stack is Empty");
					return -1;
				}
				return arr[top - 1];
			}
		
			public void push(int data) {
				if (top < MAXSIZE) {
					arr[top++] = data;
				} else {
					System.out.println("The Stack is full.");
				}
			}
		
			public int pop() {
				if (top == 0) {
					System.out.println("The Stack is already empty");
					return -1;
				}
				return arr[--top];
			}
		}
		
		class MinStack {    //取得最小元素的栈
			private static final int MAXSIZE = 10;
		
			private int min = 0;
		
			private AuxiliaryStack astack = new AuxiliaryStack(); // 辅助栈
		
			private int point = 0; // 指向栈顶
		
			private int[] arr = new int[MAXSIZE];
		
			public int getElementNum() { // 获取当前的栈的容量
				return point;
			}
		
			public int getMinData() { // 获取最小元素
				return astack.getTopElement();
			}
		
			public void push(int data) {
				if (point == 0) { // Empty Stack
					min = data;
					arr[point++] = min;
					astack.push(data);
					return;
				}
		
				if (point < MAXSIZE) {
					if (data < min) {
						min = data;
						astack.push(data);
					} else {
						astack.push(min);
					}
					arr[point++] = data;
				} else {
					System.out.println("The stack is full");
				}
			}
		
			public int pop() {
				if (point > 0) {
					astack.pop();
					point--;
					return arr[point];
				} else {
					System.out.println("The stack is empty");
					return -1;
				}
			}
		
		}
		
		public class suanfa13 {
			public static void main(String[] args) {
				int[] arr = { 4, 7, 8, 3, 5, 2, 1 };
				int i = 0;
				MinStack mstack = new MinStack();
				while (i < arr.length) {
					mstack.push(arr[i]);
					i++;
				}
				System.out.println("入栈成功");
		
				while (i > 0) {
					System.out.println("现在最小的元素是：" + mstack.getMinData());
					System.out.println("出栈元素为：" + mstack.pop());
					i--;
				}
			}
		}


