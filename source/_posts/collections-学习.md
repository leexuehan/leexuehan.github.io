---
title: collections 学习
tags:
  - Concurrency
date: 2015-09-02 11:35:57
categories: 技术
---

### Queue

#### ConcurrentLinkedQueue

#### BlockingQueue

关于这一部分移步：https://github.com/leexuehan/leexuehan.github.com/issues/65

#### Deque


### CopyOnWriteArrayList

	public class CopyOnWriteArrayList<E> extends Object
	      implements 
	      List<E>, 
	      RandomAccess, 
	      Cloneable, 
	      Serializable {
	}

**它是ArrayList 的一个线程安全的变体，其中所有可变操作（add、set 等等）都是通过对底层数组进行一次新的复制来实现的。**

在CopyOnWriteArrayList里处理写操作（包括add、remove、set等）是先将原始的数据通过Arrays.copyof()来生成一份新的数组，然后在新的数据对象上进行写，写完后再将原来的引用指向到当前这个数据对象，并且加锁。

这一般需要很大的开销，但是当**遍历操作的数量大大超过可变操作的数量**时，这种方法可能比其他替代方法更 有效。在不能或不想进行同步遍历，但又需要从并发线程中排除冲突时，它也很有用。

致谢：http://caoyaojun1988-163-com.iteye.com/blog/1754686

### CopyOnWriteArraySet

	public class CopyOnWriteArraySet<E>
	      extends 
	      AbstractSet<E>
	      implements 
	      Serializable

对其所有操作使用内部 CopyOnWriteArrayList 的 Set。因此，它共享以下相同的基本属性：

- 它最适合于具有以下特征的应用程序：set 大小通常保持很小，只读操作远多于可变操作，需要在遍历期间防止线程间的冲突。
- 它是线程安全的。
- 因为通常需要复制整个基础数组，所以可变操作（add、set 和 remove 等等）的开销很大。
- 迭代器不支持可变 remove 操作。
- 使用迭代器进行遍历的速度很快，并且不会与其他线程发生冲突。在构造迭代器时，迭代器依赖于不变的数组快照。



### ConcurrentSkipList


### ConcurrentHashMap

这一部分移步：https://github.com/leexuehan/leexuehan.github.com/issues/63