---
title: LinkedList 分析
tags:
  - Collection
  - Java
date: 2015-08-21 21:34:04
categories: 技术
---

### LinkedList 概述

List 接口的**链接列表**实现。

实现所有可选的列表操作，并且允许所有元素（包括 null）。除了实现 List 接口外，LinkedList 类还为在列表的开头及结尾 get、remove 和 insert 元素提供了统一的命名方法。这些操作允许将链接列表用作堆栈、队列或双端队列。

此类实现 Deque 接口，为 add、poll 提供先进先出队列操作，以及其他堆栈和双端队列操作。

所有操作都是按照双重链接列表的需要执行的。在列表中编索引的操作将从开头或结尾遍历列表（从靠近指定索引的一端）。

**注意，此实现不是同步的。**

### 核心源码分析

#### 储存结构

这个 List 里面的储存单元为 Entry，可以看出来其节点是一个双向链表。

	private transient Entry<E> header = new Entry<E>(null, null, null);
	...
	private static class Entry<E> {
	E element;
	Entry<E> next;
	Entry<E> previous;

	Entry(E element, Entry<E> next, Entry<E> previous) {
	    this.element = element;
	    this.next = next;
	    this.previous = previous;
	}
    }

#### 两种构造函数

	public LinkedList() {
        header.next = header.previous = header;
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
    public LinkedList(Collection<? extends E> c) {
	this();
	addAll(c);
    }

#### 增

最为常见的一种实现

	public boolean add(E e) {
	addBefore(e, header);
        return true;
    }

addBefore 函数的源码如下：

	private Entry<E> addBefore(E e, Entry<E> entry) { //双向链表的插入，查到双链表header的前面
	Entry<E> newEntry = new Entry<E>(e, entry, entry.previous);//这三行代码很重要，要熟练掌握
	newEntry.previous.next = newEntry;
	newEntry.next.previous = newEntry;
	size++;
	modCount++;
	return newEntry;
    }


插到链表的第一位

 	public void addFirst(E e) {
	addBefore(e, header.next);
    }

因为是循环链表，且是头插法，所以 header 的next 指针正好指向第一个元素。
类似的，可以得到插到最后一位的代码：


 	public void addLast(E e) {
	addBefore(e, header);
    }

还有一种插入方法，可以看到实际上内部还是用add方法来实现之。
    
 	public boolean offer(E e) {
        return add(e);
    }

类似 offerLast 和 offerFirst 也一样。

还有一种用作栈来使用的插入方法：

	public void push(E e) {
        addFirst(e);
    }

#### 删

	private E remove(Entry<E> e) { //双向循环链表的删除方法
	if (e == header)
	    throw new NoSuchElementException();
    E result = e.element;
	//个人认为以下两句代码的顺序可以颠倒，但是有一个原则要遵守，就是一次只能先删除“进来的指针”，后删除“出去的指针”。
	e.previous.next = e.next;
	e.next.previous = e.previous;
    e.next = e.previous = null;
    e.element = null;
	size--;
	modCount++; //快速失败机制，所以需要这一句
    return result;
    }


	public boolean remove(Object o) {
        if (o==null) {
            for (Entry<E> e = header.next; e != header; e = e.next) { //注意 e 的开始指向位置
                if (e.element==null) {
                    remove(e);
                    return true;
                }
            }
        } else {
            for (Entry<E> e = header.next; e != header; e = e.next) {
                if (o.equals(e.element)) {
                    remove(e);
                    return true;
                }
            }
        }
        return false;
    }

	public E poll() {
        if (size==0)
            return null;
        return removeFirst();
    }

	public E pop() {
        return removeFirst();
    }

#### 改

	public E set(int index, E element) {
        Entry<E> e = entry(index);
        E oldVal = e.element;
        e.element = element;
        return oldVal;
    }

上面用到了 entry 方法：

返回下标为 index 的元素。

	private Entry<E> entry(int index) {
        if (index < 0 || index >= size)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+size);
        Entry<E> e = header;
        if (index < (size >> 1)) {//先二分
            for (int i = 0; i <= index; i++)
                e = e.next;
        } else {
            for (int i = size; i > index; i--)
                e = e.previous;
        }
        return e;
    }


#### 查

 	public E get(int index) {
        return entry(index).element;
    }

	public E peek() {
        if (size==0)
            return null;
        return getFirst();
    }