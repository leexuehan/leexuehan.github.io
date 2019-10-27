---
title: 找出最小的 K 个数
tags:
  - Algorithm
date: 2015-07-21 21:11:51
categories: 技术
---

## 剑指offer面试题系列

> 题目：输入 n 个整数，找出其中最小的 k 个数。


### 分析

思路一：

构建小顶堆，每次取最顶上的数字，然后进行调整。时间复杂度：O(klgN).

思路二：

就是前面提到的，基于快速排序的Partion方法，不断递归分割直到枢纽元素的位置位于第K个。

**因为以上两种方法都对原数组做了改变，如果要求不能对于数组进行改变的话，则就需要想另外一种方法了**。

思路三：

先分配一个大小为K的数组，存放最小的四个元素。初始化为整数序列的前 K 个元素。然后将这 K 个元素构建称为大顶堆。

从第 K+1 个整数开始遍历，如果遍历到的整数小于根节点的元素，则将根节点的元素换成此整数，进行大顶堆调整。依次类推，直到遍历完整个整数序列。

数组中剩下的 K 个元素就是最小的 K 个数。

算法复杂度：O(NlgK)

因为前面的文章里已经写了如何构造大顶堆了（参见《算法温习之排序》），所以此处大顶堆的代码略去，只写思路一用构建小顶堆的方法。

### 代码

		package com.study;
		
		/*找出最小的k个数*/
		public class Solution17 {
			private static final int NUM = 4;
			public static void FindMinK(int[] a, int[] minK, int K) {
				int len = a.length;
				BuildMinHeap(a, a.length);
				for (int i = 0; i < K; i++) {
					minK[i] = a[0];
					len--;
					a[0] = a[len];
					BuildMinHeap(a, len);
				}
			}
		
			public static void BuildMinHeap(int[] arr, int len) {
				int hole = 0;
				int child = 0;
				for (int i = len / 2 - 1; i >= 0; i--) {
					hole = i;
					int tmp = arr[hole];
					for (; hole * 2 < len; hole = child) {
						child = 2 * hole + 1;
						if (child + 1 < len && arr[child] > arr[child + 1]) {
							child++;
						}
						if (tmp > arr[child]) {
							arr[hole] = arr[child];
						} else {
							break;
						}
					}
					arr[hole] = tmp;
		
				}
			}
		
			public static void Swap(int[] arr, int i, int j) {
				int tmp = arr[i];
				arr[i] = arr[j];
				arr[j] = tmp;
			}
		
			public static void main(String[] args) {
				int[] arr = { 9, 7, 4, 2, 5, 3, 1, 8, 6 };
				int[] out = new int[NUM];
				FindMinK(arr, out, NUM);
				for (int i = 0; i < arr.length; i++) {
					System.out.print(arr[i] + " ");
				}
		
				System.out.println();
				for (int k = 0; k < out.length; k++) {
					System.out.print(out[k] + " ");
				}
			}
		}


### 总结

又复习了一下堆排序，发现还是前面的理解还是没有太透彻，所以又写了一个构建小顶堆来复习了一下。一位学长说，基础的这些排序得写十遍才能记住，所言不虚，每写一遍，理解都深刻一点。

堆排序在处理大数据方面有着重要应用。尤其是外排序，基本都用到了堆排序的思想，要对其滚瓜烂熟。