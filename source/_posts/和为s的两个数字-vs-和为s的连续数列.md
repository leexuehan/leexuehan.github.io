---
title: 和为s的两个数字 vs 和为s的连续数列
tags:
  - Algorithm
date: 2015-09-08 22:33:55
categories: 技术
---

## 剑指offer面试题系列

### 题目

> 题目一：输入一个**递增排序数组**和一个数字s，在数组中查找两个数，使得他们的和正好是s。如果有多对数字的和等于s，输出任意**一对即可**。
> 
> 题目二：输入一个正数，打印出**所有**和为s的连续正数序列（至少含有两个数），例如输入15，结果打印1~5,4~6,7~8. 

### 思路

做算法题也有了几十道题了，觉得非常重要的就是：一定要审好题，一定要审好题，一定要审好题。

重要的事情说三遍。

第一道题：

说了要审题，但是还是没有审清题意，我看成了要求出来所有的数对了。所以代码就写成了下面这个样子。其实完全没有必要这么麻烦。

既然写都写了，说一下要打印所有的数字对的思路：先从第一个数开始遍历，因为和 Sum 已经完全确定了，所以另外一个数也就自然确定了，现在的问题是怎么找这个数。对于此题的排序数列，最快的办法我觉得是：二分搜索。

话说这道题也让我感觉自己的基础还是不扎实，二分搜索写了好半天才写得没有一点问题。平时看着如此简单的思路，可是真写，就暴露出来好多问题。汗颜!

所以好多问题，尽可能上手还是要上手。

要知行合一。

第二道题：

因为是输入一个数字，要你找到所有的连续数列。所以对于每一个序列都要从 1 开始检验。

思路是这样的：

声明两个变量：small和big。small指向这个序列最小的数字，big指向这个序列最大的数字。

small连续累加到big，和为sum。

如果 sum 大于 Num，则可以砍掉 small 指向的数字，反映在代码中也就是 small++。

如果 sum 小于 Num，则说明需要再添加一个数字，添加则要添加 big 后面的数字，也就是 big++。

如果 sum 等于 Num，说明遍历到一个数列，打印结果，并使 small++。

再次循环。

循环终止条件：small*2 = Num（没有必要循环到使small = Num）。

因为至少需要两个数字，且 small 后面的数字肯定比small大，和必然大于 Num。


### 代码

题目一：

		package com.ans;
		
		import java.util.Scanner;
		
		public class Solution11 {
			public static int binarySearch(int[] arr, int start, int end, int num) {
				if (arr == null || start > end)
					return -1;
				int mid;
				while (start <= end) {
					mid = (start + end) >> 1;
					if (num < arr[mid])
						end = mid - 1;
					else if (num > arr[mid])
						start = mid + 1;
					else if (num == arr[mid]) {
						return mid;
					}
				}
				return -1;
			}
		
			public static void main(String[] args) {
				int[] arr = { 1, 3, 5, 6, 8, 9, 11 };// Sorted Array
				int index = -1;
				Scanner scan = new Scanner(System.in);
				int Sum = 0;
				if (scan.hasNext())
					Sum = scan.nextInt();
		
				int dif;
				int pair = 0;
				for (int i = 0; i < arr.length - 1; i++) {
					dif = Sum - arr[i];
					if (dif <= 0)
						continue;
					index = binarySearch(arr, i+1, arr.length - 1, dif);
					if (index != -1 && index < arr.length) {
						pair++;
						System.out.println("第 " + pair + " 对: " + arr[i] + ","
								+ arr[index]);
						continue;
					}
					
				}
		
				if (pair == 0) {
					System.out.println("没有符合条件的数对");
				}
		
			}
		}

题目二：


		package com.ans;
		
		import java.util.Scanner;
		
		public class Solution11 {
			public static void main(String[] args) {
		
				System.out.println("请输入一个正数：");
				Scanner scan = new Scanner(System.in);
				int Num = 15, seqNum = 0;
				if (scan.hasNext())
					Num = scan.nextInt();
		
				int small = 1;// 指向连续数列的最小数字
				int big = 2;// 指向连续数列的最大数字
		
		
				int sum = 0;
				while (small * 2 < Num) {
					sum = 0;
					for (int i = small; i <= big; i++) {
						sum += i;
					}
					if (sum < Num)
						big++;
					else if (sum > Num)
						small++;
					else if (sum == Num) {
						System.out.println("找到一个连续数列:" + small + "~" + big);
						seqNum++;
						small++;
						continue;
					}
				}
				
				if(seqNum == 0) 
					System.out.println("没有找到符合条件的数列");
		
			}
		}


### 总结

在做第二道题的时候，做到后面感觉思路很乱，改改动动，越改越乱。最后，从头缕了一下，很快就把代码写出来了。所以关键还是要保证思路清晰。

在越改越乱的时候，不妨从头到尾把思路理一下，效果会很好的。


