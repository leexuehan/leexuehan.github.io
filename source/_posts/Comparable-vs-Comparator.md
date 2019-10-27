---
title: Comparable vs Comparator
tags:
  - Java
date: 2015-08-26 21:22:01
categories: 技术
---

关于这一点在 《高质量编程》一书中说得非常好，这里先只把我自己的理解写下来，之后再回头完善。

我的理解就是比如有一个类 A，如果在定义类的时候让它实现了 Comparable接口，则必须在其中重写
compareTo 方法。这表明了A的所有对象之间是可以比较的。

	public class A implements Comparable<T> {
		public int compareTo(T o) {
		...
		}
	}

可是如果在实际需求中不够用，则完全无需重写该类的 compareTo 方法，直接再定义一个工具类如AComparator，让其实现 Comparator 接口来使之可以对于 A 对象进行比较。同时需要注意要重写该接口的 compare 方法和 equals 方法。

如：

	public class AComparator implements Comparator<T> {
		public int compare(T o1, T o2) {
		...
		}
		public boolean equals(Object obj) {
			...
		}
	}

简言之， Comparable是类内置的排序方法，而Comparator 只是一个扩展的对该类的排序工具，与本类没有半毛钱关系。