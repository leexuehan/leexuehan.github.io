---
title: BlockingQueue 学习
tags:
  - Concurrency
date: 2015-08-18 11:26:26
categories: 技术
---

## 概述

BlockingQueue 是一个接口。
是支持两个附加操作的 Queue。

这两个操作分别是：获取元素时等待队列变为非空，以及存储元素时等待空间变得可用。

以下是其内部方法的轮廓图：

![](http://i.imgur.com/xOt0Wf2.png)



	 	 		抛出异常	特殊值		 阻塞			超时
	插入 		add(e)		offer(e)	put(e)		offer(e, time, unit)
	移除	 	remove()	poll()		take()		poll(time, unit)	
	检查	 	element()	peek()		不可用		  不可用

BlockingQueue 不接受 null 元素。试图 add、put 或 offer 一个 null 元素时，某些实现会抛出 NullPointerException。null 被用作指示 poll 操作失败的警戒值。

BlockingQueue 可以是限定容量的。它在任意给定时间都可以有一个 remainingCapacity，超出此容量，便无法无阻塞地 put 附加元素。没有任何内部容量约束的 BlockingQueue 总是报告 Integer.MAX_VALUE 的剩余容量。

BlockingQueue 实现**主要用于生产者-使用者队列**，但它另外还支持 Collection 接口。因此，举例来说，使用 remove(x) 从队列中移除任意一个元素是有可能的。然而，这种操作通常不会有效执行，只能有计划地偶尔使用，比如在取消排队信息时。

BlockingQueue 实现是线程安全的。所有排队方法都可以使用内部锁或其他形式的并发控制来自动达到它们的目的。然而，大量的 Collection 操作（addAll、containsAll、retainAll 和 removeAll）没有 必要自动执行，除非在实现中特别说明。因此，举例来说，在只添加了 c 中的一些元素后，addAll(c) 有可能失败（抛出一个异常）。

### 方法简述：

**add**

	boolean add(E e)

将指定的元素插入此队列（如果立即可行且不会违反容量限制），在成功时返回 true，如果当前没有可用的空间，则抛出 IllegalStateException。

**offer**

	boolean offer(E e)

将指定的元素插入此队列（如果立即可行且不会违反容量限制），当使用有容量限制的队列时，此方法通常要优于 add(E)，后者可能无法插入元素，而只是抛出一个异常。

**remove**

	E remove()

获取并移除此队列的头。此方法与 poll 唯一的不同在于：此队列为空时将抛出一个异常。

**poll**

	E poll()

获取并移除此队列的头，如果此队列为空，则返回 null。

**element**

	E element()

获取，但是不移除此队列的头。此方法与 peek 唯一的不同在于：此队列为空时将抛出一个异常。

**peek**

	E peek()

获取但不移除此队列的头；如果此队列为空，则返回 null。

## 实现

### ArrayBlockingQueue

ArrayBlockingQueue 实现了 BlockingQueue 接口，并继承 AbstractQueue。其轮廓如下：

	public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
		/** 用来储存队列元素的数据结构  */
	    private final E[] items;
	    /** 下一个要获取的元素下标 */
	    private int takeIndex;
	    /** 下一个要放入的元素下标 */
	    private int putIndex;
	    /** 队列中的元素总数 */
	    private int count;

	    /** Main lock guarding all access */
	    private final ReentrantLock lock;
	    /** Condition for waiting takes */
	    private final Condition notEmpty;
	    /** Condition for waiting puts */
	    private final Condition notFull;
			...
	}

add 方法：

	public boolean add(E e) {
		return super.add(e);
    }

可见其调用了基本的queue的add 方法。

offer 方法：

offer 方法分两种，一种是如下面的源码所示，上锁然后插入元素；另一种就是定时插入。

	public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                insert(e); //插入元素到队列中
                return true;
            }
        } finally {
            lock.unlock();
        }
    }

	public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
	long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                if (count != items.length) {
                    insert(e);
                    return true;
                }
                if (nanos <= 0)
                    return false;
                try {
                    nanos = notFull.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    notFull.signal(); // propagate to non-interrupted thread
                    throw ie;
                }
            }
        } finally {
            lock.unlock();
        }
    }

注意到其中调用了一个 insert 方法。其源代码如下：

	private void insert(E x) {
        items[putIndex] = x;
        putIndex = inc(putIndex); //循环增加
        ++count;
        notEmpty.signal();
    }

	final int inc(int i) {
        return (++i == items.length)? 0 : i;
    }

put 方法：

	public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        final E[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            try {
                while (count == items.length)
                    notFull.await();
            } catch (InterruptedException ie) {
                notFull.signal(); // propagate to non-interrupted thread
                throw ie;
            }
            insert(e);
        } finally {
            lock.unlock();
        }
    }

remove 方法：

	public boolean remove(Object o) {
        if (o == null) return false;
        final E[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int i = takeIndex;
            int k = 0;
            for (;;) {
                if (k++ >= count)
                    return false;
                if (o.equals(items[i])) {
                    removeAt(i);
                    return true;
                }
                i = inc(i);
            }

        } finally {
            lock.unlock();
        }
    }
	
	void removeAt(int i) {
        final E[] items = this.items;
        // if removing front item, just advance
        if (i == takeIndex) {
            items[takeIndex] = null;
            takeIndex = inc(takeIndex);
        } else {
            // slide over all others up through putIndex.
            for (;;) {
                int nexti = inc(i);
                if (nexti != putIndex) {
                    items[i] = items[nexti];
                    i = nexti;
                } else {
                    items[i] = null;
                    putIndex = i;
                    break;
                }
            }
        }
        --count;
        notFull.signal();
    }

poll 方法：

	public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == 0)
                return null;
            E x = extract();
            return x;
        } finally {
            lock.unlock();
        }
    }

	private E extract() {
        final E[] items = this.items;
        E x = items[takeIndex];
        items[takeIndex] = null;
        takeIndex = inc(takeIndex);
        --count;
        notFull.signal();
        return x;
    }

take 方法：

 	public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            try {
                while (count == 0)
                    notEmpty.await();
            } catch (InterruptedException ie) {
                notEmpty.signal(); // propagate to non-interrupted thread
                throw ie;
            }
            E x = extract();
            return x;
        } finally {
            lock.unlock();
        }
    }

peek 方法：

	public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : items[takeIndex];
        } finally {
            lock.unlock();
        }
    }

size 方法：

	public int size() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }

### LinkedBlockingQueue

一个基于基于链表的、任选范围的阻塞双端队列。

可选的容量范围构造方法参数是一种防止过度膨胀的方式。如果未指定容量，那么容量将等于 Integer.MAX_VALUE。

只要插入元素不会使双端队列超出容量，每次插入后都将动态地创建链接节点。

LinkedBlockingQueue之所以能够高效的处理并发数据，还因为其对于生产者端和消费者端分别采用了独立的锁来控制数据同步，这也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。

	public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
		/**
     	* Linked list node class
     	*/
    	static class Node<E> {
        	/** The item, volatile to ensure barrier separating write and read */
        	volatile E item;
        	Node<E> next;
       	 	Node(E x) { item = x; }
    	}
		/** The capacity bound, or Integer.MAX_VALUE if none */
	    private final int capacity;
	
	    /** Current number of elements */
	    private final AtomicInteger count = new AtomicInteger(0);
	
	    /** Head of linked list */
	    private transient Node<E> head;
	
	    /** Tail of linked list */
	    private transient Node<E> last;
	
	    /** Lock held by take, poll, etc */
	    private final ReentrantLock takeLock = new ReentrantLock();
	
	    /** Wait queue for waiting takes */
	    private final Condition notEmpty = takeLock.newCondition();
	
	    /** Lock held by put, offer, etc */
	    private final ReentrantLock putLock = new ReentrantLock();
	
	    /** Wait queue for waiting puts */
	    private final Condition notFull = putLock.newCondition();
		...

	}

构造函数的三种重载方式：

	public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE); //默认大小为 Integer.MAX_VALUE
    }

    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }

    public LinkedBlockingQueue(Collection<? extends E> c) { //用集合来构造队列
        this(Integer.MAX_VALUE);
        for (E e : c)
            add(e);
    }

put 方法：

	public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset
        // local var holding count  negative to indicate failure unless set.
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from
             * capacity. Similarly for all other uses of count in
             * other wait guards.
             */
            try {
                while (count.get() == capacity)
                    notFull.await();
            } catch (InterruptedException ie) {
                notFull.signal(); // propagate to a non-interrupted thread
                throw ie;
            }
            insert(e);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }

offer 方法：

 	public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            for (;;) {
                if (count.get() < capacity) {
                    insert(e);
                    c = count.getAndIncrement();
                    if (c + 1 < capacity)
                        notFull.signal();
                    break;
                }
                if (nanos <= 0)
                    return false;
                try {
                    nanos = notFull.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    notFull.signal(); // propagate to a non-interrupted thread
                    throw ie;
                }
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }

上面两个方法都调用了一个相同的方法：insert。

	private void insert(E x) {
        last = last.next = new Node<E>(x);//插入到队尾，这个连等用得非常漂亮
    }

take 方法：

	public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            try {
                while (count.get() == 0)
                    notEmpty.await();
            } catch (InterruptedException ie) {
                notEmpty.signal(); // propagate to a non-interrupted thread
                throw ie;
            }

            x = extract();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

poll 方法：

 	public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = extract();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }


上面两种方法共同调用了一个extract 方法：

	 private E extract() {
        Node<E> first = head.next;
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }

peek 方法：

	public E peek() {
        if (count.get() == 0)
            return null;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            Node<E> first = head.next;
            if (first == null)
                return null;
            else
                return first.item;
        } finally {
            takeLock.unlock();
        }
    }

remove 方法

	public boolean remove(Object o) {
        if (o == null) return false;
        boolean removed = false;
        fullyLock();
        try {
            Node<E> trail = head;
            Node<E> p = head.next;
            while (p != null) {
                if (o.equals(p.item)) {
                    removed = true;
                    break;
                }
                trail = p;
                p = p.next;
            }
            if (removed) {
                p.item = null;
                trail.next = p.next;
                if (last == p)
                    last = trail;
                if (count.getAndDecrement() == capacity)
                    notFull.signalAll();
            }
        } finally {
            fullyUnlock();
        }
        return removed;
    }