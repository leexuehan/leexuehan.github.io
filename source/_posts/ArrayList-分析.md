---
title: ArrayList 分析
tags:
  - Collection
  - Java
date: 2015-08-21 20:34:39
categories: 技术
---

### ArrayList 概述

ArrayList 是 List 接口的**大小可变**的数组实现。实现了所有可选列表操作，并允许包括 null 在内的所有元素。除了实现 List 接口外，此类还提供一些方法来操作内部用来存储列表的数组的大小。（此类大致上等同于 Vector 类，除了此类是不同步的。）

size、isEmpty、get、set、iterator 和 listIterator 操作都以固定时间运行。

add 操作以分摊的固定时间 运行，也就是说，添加 n 个元素需要 O(n) 时间。其他所有操作都以线性时间运行（大体上讲）。

与用于 LinkedList 实现的常数因子相比，此实现的常数因子较低。

**注意，此实现不是同步的。**

此类的 iterator 和 listIterator 方法返回的迭代器是快速失败的：在创建迭代器之后，除非通过迭代器自身的 remove 或 add 方法从结构上对列表进行修改，否则在任何时间以任何方式对列表进行修改，迭代器都会抛出 ConcurrentModificationException。因此，面对并发的修改，迭代器很快就会完全失败，而不是冒着在将来某个不确定时间发生任意不确定行为的风险。

### 核心源码分析

#### 构造函数

	public ArrayList(int initialCapacity) {
		super();//调用父类构造函数进行初始化
	        if (initialCapacity < 0)
	            throw new IllegalArgumentException("Illegal Capacity: "+
	                                               initialCapacity);
		this.elementData = new Object[initialCapacity];//这里面的 elementData 是一个Object类型的数组
    }

    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
		this(10);
    }

#### 增

	public boolean add(E e) { //最常规的实现
		ensureCapacity(size + 1);  // Increments modCount!!
		elementData[size++] = e;
		return true;
    }

	public void add(int index, E element) { //在指定位置添加
	if (index > size || index < 0)
	    throw new IndexOutOfBoundsException(
		"Index: "+index+", Size: "+size);

	ensureCapacity(size+1);  // Increments modCount!!
	System.arraycopy(elementData, index, elementData, index + 1,
			 size - index);//这里用到了一个工具类
	//参数的意义依次为： 源数组，原数组起始下标，目的数组，目的数组起始下标，拷贝数组的长度
	elementData[index] = element;
	size++;
    }

	/**
     * Increases the capacity of this <tt>ArrayList</tt> instance, if
     * necessary, to ensure that it can hold at least the number of elements
     * specified by the minimum capacity argument.
     */
    public void ensureCapacity(int minCapacity) {
	modCount++;
	int oldCapacity = elementData.length;
	if (minCapacity > oldCapacity) { //参数设定的容量大于原数组的容量
	    Object oldData[] = elementData;
	    int newCapacity = (oldCapacity * 3)/2 + 1; //相当于增大为原容量的 1.5 倍 + 1
    	    if (newCapacity < minCapacity) //如果扩容后的容量还是没有办法满足所设定的参数，则将
							   			   //新的容量变为设定的参数大小
				newCapacity = minCapacity;
            // minCapacity is usually close to size, so this is a win:
            elementData = Arrays.copyOf(elementData, newCapacity);
	}
    }

上面的 ensureCapacity 函数中的注释已经说得很清楚了，主要作用是必要时增大 ArrayList 的容量，至少也应该和参数设定的容量相当。后面会多次用到这个函数。

#### 删

	public E remove(int index) {//删除指定下标的元素
	RangeCheck(index); //下标合法性检查

	modCount++;
	E oldValue = (E) elementData[index]

	int numMoved = size - index - 1; //算出需要移动的元素个数
	if (numMoved > 0)
	    System.arraycopy(elementData, index+1, elementData, index,
			     numMoved);//再一次用到了系统类 System 的方法
	elementData[--size] = null; // Let gc do its work

	return oldValue;
    }

 	public boolean remove(Object o) { //删除等于某个对象的元素
	if (o == null) {
            for (int index = 0; index < size; index++)
				if (elementData[index] == null) {
				    fastRemove(index);
				    return true;
				}
	} else {
	    for (int index = 0; index < size; index++)
		if (o.equals(elementData[index])) {
		    fastRemove(index);
		    return true;
		}
        }
	return false;
    }

这里都用到了 一个方法叫 fastRemove。

	private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // Let gc do its work
    }

**全部删除**

	public void clear() {
	modCount++;

	// Let gc do its work
	for (int i = 0; i < size; i++)
	    elementData[i] = null;

	size = 0;
    }

#### 改

	public E set(int index, E element) {
	RangeCheck(index);

	E oldValue = (E) elementData[index];
	elementData[index] = element;
	return oldValue;
    }

#### 查

查元素

 	public E get(int index) {
	RangeCheck(index);

	return (E) elementData[index];
    }

查大小

	public int size() {
	return size;
    }