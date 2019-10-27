---
title: 打印最大的n位数
tags:
  - Algorithm
date: 2015-07-13 20:36:56
categories: 技术
---

## 剑指offer面试题系列


> 题目：输入数字n，按顺序打印出从1到最大的n位十进制数。比如，输入3，则打印出1，2，3，.....，一直到最大的3位数即999。

### 分析

首先，因为没有规定数字 n 的范围，所以需要考虑大数的情况，而不能简单地认为是一个小范围的数字。而大数一般都超出了用基本类型的数据表示的范围，所以**应该考虑选择数组或者字符串**。

本人比较习惯用数组来表示大数。所以代码示例中也用了数组这种表示方式。但是，这种表示方法需要注意一个问题，就是**高低位顺序**的问题。
数组的低位正好对应数字的高位。

然后需要考虑进位和结束的问题。

本例中用一个变量nTakeOver来表示进位，用一个布尔型变量isOverflow判断溢出。

思路：

刚开始打算用下面的方式来做

		public static void print(int bits) {
				int i, j, k;
				for (i = 0; i < 10; i++) {
					for (j = 0; j < 10; j++) {
						for (k = 0; k < 10; k++)
							System.out.println(i + "" + j + "" + k);
					}
				}
			}

但是这种方法有个缺点就是循环嵌套太深，且数字前面的0不能消掉，最后是 001 002 这种形式。

之后，尝试采用另外一种思路，有进位表示，然后用一个pointer来指示需要改变的位置，但是没有想明白一个问题：如何让低位发生进位后，指针向前移，同时还能照顾到低位的情况。

百思不得其解，于是看了剑指offer上面的答案，原来需要的是最原始的考虑方式，就是你怎么做加法，然后让程序按照你的思路去做就行了。

所以，你需要做的只是，每次从最低位开始计算，让一个位指针指向最低位，最低位加1，如果最低位满10，则产生进位然后让指针bits前移，去把进位加上去，如果没有满，则循环结束。

这样一直进行，直到溢出。

### 代码

		package com.study;
		
		import java.io.BufferedWriter;
		import java.io.FileWriter;
		import java.io.IOException;
		
		public class suanfa10 {
			/* 最终版，模拟了加法思想 */
			private static final int MAXBITS = 6;
		
			public static boolean increment(int[] arr) {
				boolean isOverflow = false;
				int nTakeOver = 0; // 进位
				int sum = 0;
				for (int bit = arr.length - 1; bit >= 0; bit--) {
					sum = arr[bit] + nTakeOver;
					if (bit == arr.length - 1) {
						sum++;
					}
					if (bit == 0) {
						isOverflow = true;
						break;
					}
					if (sum < 10) {
						arr[bit] = sum;
						break;
					} else {
						arr[bit] = sum - 10;
						nTakeOver = 1;
					}
				}
				return isOverflow;
			}
		
			public static void print(int[] a, BufferedWriter bw) throws IOException {
				int i;
				for (i = 0; a[i] == 0 && i < a.length; i++)
					// 去掉前面的0
					;
				for (int j = i; j < a.length; j++) {
					bw.write(a[j] + "");
					System.out.print(a[j] + "");// 拼接字符串
				}
				bw.write("\n");
				System.out.println();
			}
		
			public static void main(String[] args) throws IOException {
		
				int[] array = new int[MAXBITS + 1];
				BufferedWriter bw = new BufferedWriter(new FileWriter(
						"E:/OutputData.txt"));
				while (!increment(array)) {
					print(array, bw);
				}
				System.out.println("End Printing...");
				bw.flush();
				bw.close();
				System.out.println("End Print!");
		
			}
		}


### 备注

1.在代码中用了输出文件的方法，让结果直接输出到一个 OutputData.txt 文件中，可以防止在控制台的刷屏。
 
2.在成功实现了大数的打印之后，又想到以前遇到的一个问题，执行两个大数之间的加法，尝试用数组实现了一下，主要代码如下：

		public static void Add(int[] num1, int[] num2, int[] res) {
				int i = num1.length - 1;
				int j = num2.length - 1;
				int len = 0, less = j;
				int[] mNum, vNum;
				if (i > j) {
					len = i;
					less = j;
					mNum = num1;
					vNum = num2;
				} else {
					len = j;
					less = i;
					mNum = num2;
					vNum = num1;
				}
		
				int nTakeOver = 0;
				int Sum = 0;
				for (; len >= 0; len--, less--) {
					if (less >= 0) {
						Sum = mNum[len] + vNum[less] + nTakeOver;
						nTakeOver = 0;
						if (Sum >= 10) {
							res[len] = Sum - 10;
							nTakeOver = 1;
						} else {
							res[len] = Sum;
						}
					} else {
						Sum = mNum[len] + nTakeOver;
						nTakeOver = 0;
						res[len] = Sum;
					}
		
				}
			}

要考虑两个数的位数不一致的情况，如果调用这个函数，可以将输入的字符串转换成数组，代入即求得最终答案。

3.本题还可以用**全排列**的思想去做。

全排列的思路其实跟上面的第一版的代码有点相像，但是比第一版代码更为简洁，是用**递归**的思想来做的。从第一个元素开始向后递归，先设置高位，再设置低位。当到了最后一位时，打印出来。

主体代码如下：

		public static void OutputNum(int bits) throws IOException {
				BufferedWriter bw = new BufferedWriter(new FileWriter(
						"E:/OutputData.txt"));
				int[] a = new int[bits];
				printNumber(a, bits, 0, bw);
				System.out.println("End Printing...");
				bw.flush();
				bw.close();
				System.out.println("End Print!");
			}
		
			/* 按照全排列递归地打印此数字 */
			public static void printNumber(int[] a, int length, int index,
					BufferedWriter bw) throws IOException {
				System.out.println("index :" + index);
				if (index == length) {
					formatNumber(a, bw);
					return;
				} else {
					for (int i = 0; i < 10; i++) {
						a[index] = i;
						printNumber(a, length,index + 1, bw);
		
					}
				}
			}
		
			public static void formatNumber(int[] a, BufferedWriter bw)
					throws IOException {
				int i;
				for (i = 0; i < a.length && a[i] == 0; i++)
					// 去掉前面的0
					;
				for (int j = i; j < a.length; j++) {
					bw.write(a[j] + " ");
					System.out.print(a[j] + " ");// 拼接字符串
				}
				bw.write("\n");
				System.out.println();
			}

递归的思路很难想到，但在我看来，如果一件事情能拆分成各种相同的小事件，则可用递归去做。但是**难点在于如何拆分这样的事件**。这就需要多加练习了。

### 总结

还是要多做，在自己做的过程中，虽然知道是按照最直观的思路去做，可是怎么也找不到感觉。要不断加强代码功底。