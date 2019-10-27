---
title: JDK源码之PriorityQueue
date: 2019-08-18 19:57:24
tags: JDK
categories: 技术
---

# JDK源码之PriorityQueue实现

探讨下 JDK 中关于优先级队列的实现。

## 堆排序

堆，或者二叉堆，是一个数组，可以被看成一个近似的完全二叉树，树上的每一个节点对应数组中的一个元素。

如下图所示是一棵树和数组之间的对应关系。

<div align=center>![堆与数组](./heap_tree.png)

观察图中关系，我们可以知道树中的一个结点下标 `i` 和它的父结点、左孩子和右孩子的下标对应关系。
$$
PARENT(i)=i/2(向下取整)
$$

$$
LEFT(i)=2*i
$$

$$
RIGHT(i)=2*i+1
$$

二叉堆分为最大堆和最小堆。

最大堆满足性质：
$$
A[PARENT(i)]≥A[i]
$$
最小堆满足性质：
$$
A[PARENT(i)]≤A[i]
$$
这里我们以最大堆为例，介绍堆排序算法的思路。

```
MAX-HEAPIFY（A,i）
	l=LEFT(i)
	r=RIGHT(i)
	if l <= A.heap-size and A[l]>A[i]
		largest=l;
	if r<= A.heap-size and A[r]>A[i]
		largest=r;
	if largest != i
		exchange A[i] with A[largest]
		MAX-HEAPIFY(A,largest)
```

上面的伪代码是一个子过程，该过程的作用就是**维护最大堆**。

采用的思路是“逐级下沉”的方式，从下标 i 开始调整，如果 `i` 以及其左右孩子节点都满足最大堆，则不进行下一步，否则调换使得 `i` 处的节点最大，然后递归调整其子节点。

在伪代码中，A 数组的下标是从 `1` 开始的，而且 A.heap-size 与 A.length 要注意区分。下同。

```
BUILD-MAX-HEAP(A)
	A.heap-size=A.length
	for i = [A.length/2](向下取整) downto 1
		MAX-HEAPIFY(A,i)
```

这个子过程的作用是利用数组创建一个最大堆。

这个过程采用的思路是“自顶向上”的方式。从**最后一个有孩子节点的节点**开始向上调整，维护最大堆的性质。

```
HEAPSORT(A)
	BUILD-MAX-HEAP(A)
	for i=A.length downto 2
		exchange A[1] with A[i]
		A.heap-size=A.heap-size-1
		MAX-HEAPIFY(A,1)
```

上述子过程结合了前面的两个子过程，开始真正排序。

该过程也是采取自底向上的方式。先把数组的第一个元素放到最后（第一个元素在堆顶，故一定是最大的），然后开始调整（注意：此时要减小堆的长度，因为最后一个元素已经不参与排序）。以此类推。

示意图如下：

<div align=center>![堆排序过程](./heap_sort.png)

现在开始估计复杂度。

MAX-HEAPIFY 最大时间为 O(lgN)。HEAP-SORT 子过程需要调用 n 次 MAX-HEAPIFY 子过程，所以时间复杂度为 O(NlogN).

## 优先级队列

有了以上基础，我们可以接下来看看JDK中优先级队列的实现。

官方对优先级队列给出的介绍大概如下：

1. 基于优先级堆的无界队列
2. 队列元素的是根据它们的自然顺序或者在构造队列时传入的 Comparator 比较器进行排序的。
3. 不允许 null 元素。
4. 最小堆

现在我们来看看，这个类的一些关键代码。

优先级队列的底层是使用一个对象数组来实现的。

```
transient Object[] queue; 
```

### 插入元素

```
public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        //扩容
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        //空队列
        if (i == 0)
            queue[0] = e;
        //非空队列，从队尾插入
        else
            siftUp(i, e);
        return true;
    }
```

很显然，核心代码在 siftUp 中，最终调用到了  siftUpComparable 或者 siftUpUsingComparator 方法，其核心思路是一样的，只不过使用的比较器不同。

```
private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
```

从上面代码中我们可以看到，首先维护最小堆的方式**不是递归方式**。而是采用了更改新元素 `k` 的位置，自底向上，逐级比较，如果满足最小堆性质，则直接插入到该位置，结束，如果不满足，则继续往上走，直到满足条件为止。

### 删除元素

```
public E poll() {
        if (size == 0)
            return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];
        E x = (E) queue[s];
        queue[s] = null;
        if (s != 0)
            siftDown(0, x);
        return result;
    }
```

直接取得 queue 的第一个元素，将队尾元素赋给 `x`，然后开始调用 siftDown 进行最小堆调整。

同样的，该方法最终调用了 siftDownComparable 方法。

```
private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        int half = size >>> 1;        // loop while a non-leaf
        while (k < half) {
            int child = (k << 1) + 1; // assume left child is least
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];
            if (key.compareTo((E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = key;
    }
```

这回是采用“逐级下沉”的方式进行调整。

在位置 k 处插入元素 x，比较 k 处节点与其左右孩子的大小，如果满足最小堆的性质，则直接插入即可，否则更新 k 的值，继续进行比较。

### 建堆

PriorityQueue 可以根据三种集合建堆：

1. 排序集合
2. 优先级队列
3. 普通集合

```
public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }

```

针对情况1、2：已经是有序集合，所以直接拷贝到queue中；

针对情况3：普通的无序集合。

```
private void initFromCollection(Collection<? extends E> c) {
        initElementsFromCollection(c);
        heapify();
}
private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
}    

```

这里是从最后一个有孩子节点的节点开始到堆顶，采用逐级下沉的方式维护最小堆的性质。



（全文完）

## 参考资料

1. 《算法导论（第三版）》