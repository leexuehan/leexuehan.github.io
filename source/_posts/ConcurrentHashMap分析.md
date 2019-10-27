---
title: ConcurrentHashMap分析
tags:
  - Concurrency
date: 2015-08-17 20:18:23
categories: 技术
---

### 前面的话

这几天在看 Java.util.concurrent 包里的各种类，关于这个包的关系图，网上有一张很清晰的图如下：

![](http://i.imgur.com/iWxDC8g.png)

打算在接下来的一周时间里，将 collections 里面的 BlockingQueue 和 ConcurrentHashMap 以及 executor 重点看完，将其核心源码弄明白，如果之后还有时间的话，会写一些例子来使用它们。

这一篇主要讲 ConcurrentHashMap。

## ConcurrentHashMap 

### 锁分段

因为多线程环境下，HashMap 会有线程安全问题， Hashtable 虽然是线程安全的，但是它使用 synchronized 来保证线程安全，但这样一来，在大规模的线程访问的情况下就会出现效率低下的问题，因为如果有一个线程拥有这把锁，则其他的线程只能等待。

在这种情况下，软件大牛们想出了一个解决办法：**锁分段**。

什么是锁分段？

Hashtable 容器在竞争激烈的并发环境下表现出效率低下的原因是所有访问 Hashtable 的线程必须竞争同一把锁。设想一下，如果容器有多把锁，则每一把锁用于锁住容器其中的一部分数据，那么当多线程访问容器里不同的数据段的数据时，线程之间不会存在锁竞争，从而有效地提高并发访问率。这就是**锁分段技术**。

### 结构

![](http://i.imgur.com/QHn33G7.png)

上图就是 ConcurrentHashMap 的大致结构。

可以看出来，它是由 一个私有的Segment数组和 HashEntry 数组结构组成。而Segment是一种可重入锁 ReentrantLock。

 	static final class Segment<K,V> extends ReentrantLock implements Serializable {
		...
	}

Segment的结构与HashMap类似，是一种数据和链表结构，一个Segment 里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素，每个Segment守护着一个HashEntry数组里的元素，当对HashEntry数组的数据修改时，必须首先获取它对应的Segment锁。

结构如下：

![](http://i.imgur.com/rS67Ipe.png)

HashEntry 用于存储键值对数据。

### ConcurrentHashMap 的初始化

ConcurrentHashMap初始化方法是通过initialCapacity，loadFactor, concurrencyLevel几个参数来初始化segments数组，段偏移量segmentShift，段掩码segmentMask和每个segment里的HashEntry数组 。

以下是其初始化的源代码

		public ConcurrentHashMap(int initialCapacity,
	                             float loadFactor, int concurrencyLevel) {
	        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
	            throw new IllegalArgumentException();
	
	        if (concurrencyLevel > MAX_SEGMENTS)
	            concurrencyLevel = MAX_SEGMENTS;
	
	        // Find power-of-two sizes best matching arguments
	        int sshift = 0;
	        int ssize = 1;
	        while (ssize < concurrencyLevel) {
	            ++sshift;
	            ssize <<= 1;
	        }
	        segmentShift = 32 - sshift;
	        segmentMask = ssize - 1;
	        this.segments = Segment.newArray(ssize);
	
	        if (initialCapacity > MAXIMUM_CAPACITY)
	            initialCapacity = MAXIMUM_CAPACITY;
	        int c = initialCapacity / ssize;
	        if (c * ssize < initialCapacity)
	            ++c;
	        int cap = 1;
	        while (cap < c)
	            cap <<= 1;
	
	        for (int i = 0; i < this.segments.length; ++i)
	            this.segments[i] = new Segment<K,V>(cap, loadFactor);
	    }

代码解析：

1.初始化Segment数组的长度

由上面的代码可知segments数组的长度ssize通过concurrencyLevel计算得出。为了能通过按位与的哈希算法来定位segments数组的索引，必须保证segments数组的长度是2的N次方（power-of-two size），所以必须计算出一个是大于或等于concurrencyLevel的最小的2的N次方值来作为segments数组的长度。假如concurrencyLevel等于14，15或16，ssize都会等于16，即容器里锁的个数也是16。注意concurrencyLevel的最大大小是65535，意味着segments数组的长度最大为65536，对应的二进制是16位。

2.初始化segmentShift和segmentMask。

这两个全局变量在定位segment时的哈希算法里需要使用，sshift等于ssize从1向左移位的次数，在默认情况下concurrencyLevel等于16，1需要向左移位移动4次，所以sshift等于4。segmentShift用于定位参与hash运算的位数，segmentShift等于32减sshift，所以等于28，这里之所以用32是因为ConcurrentHashMap里的hash()方法输出的最大数是32位的，后面的测试中我们可以看到这点。segmentMask是哈希运算的掩码，等于ssize减1，即15，掩码的二进制各个位的值都是1。因为ssize的最大长度是65536，所以segmentShift最大值是16，segmentMask最大值是65535，对应的二进制是16位，每个位都是1。

3.初始化每个Segment。输入参数initialCapacity是ConcurrentHashMap的初始化容量，loadfactor是每个segment的负载因子，在构造方法里需要通过这两个参数来初始化数组中的每个segment。

			if (initialCapacity > MAXIMUM_CAPACITY)
	            initialCapacity = MAXIMUM_CAPACITY;
	        int c = initialCapacity / ssize;
	        if (c * ssize < initialCapacity)
	            ++c;
	        int cap = 1;
	        while (cap < c)
	            cap <<= 1;
	
	        for (int i = 0; i < this.segments.length; ++i)
	            this.segments[i] = new Segment<K,V>(cap, loadFactor);

上面代码中的变量cap就是segment里HashEntry数组的长度，它等于initialCapacity除以ssize的倍数c，如果c大于1，就会取大于等于c的2的N次方值，所以cap不是1，就是2的N次方。segment的容量threshold＝(int)cap*loadFactor，默认情况下initialCapacity等于16，loadfactor等于0.75，通过运算cap等于1，threshold等于零。

		Segment(int initialCapacity, float lf) {
            loadFactor = lf;
            setTable(HashEntry.<K,V>newArray(initialCapacity));
        }

这是 Segment 的构造函数。其中 newArray 的实现代码：

		static final <K,V> HashEntry<K,V>[] newArray(int i) {
	    	return new HashEntry[i];
		}

setTable的实现代码：

 		void setTable(HashEntry<K,V>[] newTable) {
            threshold = (int)(newTable.length * loadFactor);
            table = newTable;
        }


至此，初始化已经完成。

### ConcurrentHashMap 的重要操作

#### get 操作

	public V get(Object key) {
        int hash = hash(key.hashCode());
        return segmentFor(hash).get(key, hash);
    }

#### put 操作

	public V put(K key, V value) {
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key.hashCode());
        return segmentFor(hash).put(key, hash, value, false);
    }

#### remove 操作

 	public V remove(Object key) {
	int hash = hash(key.hashCode());
        return segmentFor(hash).remove(key, hash, null);
    }


以上三种操作有一个共同点就是：无论是增、删、查，最终都归结为 segment的相应操作，且都用到了segmentFor 这个函数 。关于Segment 后面会有介绍，此处先不表。

先简要说一下 segmentFor 这个函数。

	final Segment<K,V> segmentFor(int hash) {  
    	return segments[(hash >>> segmentShift) & segmentMask];  
	} 

这个函数用了位操作来确定Segment，根据传入的hash值向右无符号右移segmentShift位，然后和segmentMask进行与操作，结合我们之前说的segmentShift和segmentMask的值，就可以得出以下结论：假设Segment的数量是2的n次方，根据元素的hash值的高n位就可以确定元素到底在哪一个Segment中。

#### size 操作

size 操作有点不同，如果我们要统计整个ConcurrentHashMap里元素的大小，就必须统计所有Segment里元素的大小后求和。

Segment里的全局变量count是一个volatile变量，那么在多线程场景下，我们是不是直接把所有Segment的count相加就可以得到整个ConcurrentHashMap大小了呢？

不是的，虽然相加时可以获取每个Segment的count的最新值，但是拿到之后可能累加前使用的count发生了变化，那么统计结果就不准了。

所以最安全的做法，是在统计size的时候把所有Segment的put，remove和clean方法全部锁住，但是这种做法显然非常低效。 因为在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以ConcurrentHashMap的做法是**先尝试2次通过不锁住Segment的方式来统计各个Segment大小**，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。

那么ConcurrentHashMap是如何判断在统计的时候容器是否发生了变化呢？

使用modCount变量，在put , remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化。

	public int size() {
        final Segment<K,V>[] segments = this.segments;
        long sum = 0;
        long check = 0;
        int[] mc = new int[segments.length];
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
		// RETRIES_BEFORE_LOCK 的值为2.
        for (int k = 0; k < RETRIES_BEFORE_LOCK; ++k) {
            check = 0;//用来记录是否在检验累加过程中修改次数发生了改变
            sum = 0;//用来统计总长度
            int mcsum = 0; //用来统计各个 Segment 被修改的次数之和
            for (int i = 0; i < segments.length; ++i) {
                sum += segments[i].count;
                mcsum += mc[i] = segments[i].modCount;
            }
            if (mcsum != 0) {
                for (int i = 0; i < segments.length; ++i) {
                    check += segments[i].count;
                    if (mc[i] != segments[i].modCount) {
                        check = -1; // force retry
                        break;
                    }
                }
            }
            if (check == sum)
                break;
        }
        if (check != sum) { // 如果发生了改变，则将每一个Segment 都锁住，然后进行累加。
            sum = 0;
            for (int i = 0; i < segments.length; ++i)
                segments[i].lock();
            for (int i = 0; i < segments.length; ++i)
                sum += segments[i].count;
            for (int i = 0; i < segments.length; ++i)
                segments[i].unlock();
        }
        if (sum > Integer.MAX_VALUE)
            return Integer.MAX_VALUE;
        else
            return (int)sum;
    }

## Segment 的实现

Segment 只是 ConcurrentHashMap 内部的一个静态类。

	public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>  
        implements ConcurrentMap<K, V>, Serializable {  
  
  
    	/** 
     	*  key, hash, next都是不可改的 
  	 	*  value值可被重写 
   		*/  
    	static final class HashEntry<K,V> {  
        	final K key;  
        	final int hash;  
        	volatile V value;  
        	final HashEntry<K,V> next;  
  
        	...  
    	}  
  
    	static final class Segment<K,V> extends ReentrantLock implements Serializable {  
  
        transient volatile int count;  
        transient volatile HashEntry[] table;  
        // 当segment中元素个数达到threshold时，需要rehash  
        transient int threshold;  
    	}  
  
  		...  
	}   

### Segment 的 get 操作

	 	V get(Object key, int hash) {
            if (count != 0) { // read-volatile
                HashEntry<K,V> e = getFirst(hash);
                while (e != null) {
                    if (e.hash == hash && key.equals(e.key)) {
                        V v = e.value;
                        if (v != null)
                            return v;
                        return readValueUnderLock(e); // recheck
                    }
                    e = e.next;
                }
            }
            return null;
        }

下面是getFirst的代码：
	
 		HashEntry<K,V> getFirst(int hash) {
            HashEntry<K,V>[] tab = table;
            return tab[hash & (tab.length - 1)];
        }

由于HashEntry当中的key和next都是final的，所以segment之上的操作不可能影响HashEntry列表之间相对的顺序，而value是可变的，当第一次读值失败时，尝试加锁读。

以上源代码都是 JDK 1.6 版本的。


### Segment 的 put 操作


		V put(K key, int hash, V value, boolean onlyIfAbsent) {
            lock();
            try {
                int c = count;
                if (c++ > threshold) // ensure capacity
                    rehash(); //扩容
                HashEntry<K,V>[] tab = table;
                int index = hash & (tab.length - 1);
                HashEntry<K,V> first = tab[index]; // 用 first 指向下标为 index 的链表的第一个元素
                HashEntry<K,V> e = first;//e作为一个指针指向链表头，开始遍历
                while (e != null && (e.hash != hash || !key.equals(e.key))) //必须 hash 和 key 都相等才满足条件
                    e = e.next;

                V oldValue;
                if (e != null) { //找到了一样的元素
                    oldValue = e.value; //替换旧值，注意这里没有修改 modCount 的值
                    if (!onlyIfAbsent)
                        e.value = value;
                }
                else { //一直找到链表结尾
                    oldValue = null;
                    ++modCount; //修改次数增加
                    tab[index] = new HashEntry<K,V>(key, hash, first, value);//因为每一个 HashEntry 的 next 指针都不能修改，所以只能插到链表头
                    count = c; // write-volatile
                }
                return oldValue;
            } finally {
                unlock();
            }
        }

当数量达到了 threadhold 时，用到了 rehash 函数。其源码如下：

		void rehash() { //扩容
            HashEntry<K,V>[] oldTable = table;
            int oldCapacity = oldTable.length;
            if (oldCapacity >= MAXIMUM_CAPACITY)
                return;
            HashEntry<K,V>[] newTable = HashEntry.newArray(oldCapacity<<1);//扩大2倍
            threshold = (int)(newTable.length * loadFactor); //新的 threshhold
            int sizeMask = newTable.length - 1;
            for (int i = 0; i < oldCapacity ; i++) {
                HashEntry<K,V> e = oldTable[i];
                if (e != null) {
                    HashEntry<K,V> next = e.next;
                    int idx = e.hash & sizeMask; //计算节点所在数组的新的下标

                    // 链表上只有一个节点
                    if (next == null)
                        newTable[idx] = e;
					//链表上不止一个节点
                    else {
                        // Reuse trailing consecutive sequence at same slot
                        HashEntry<K,V> lastRun = e;
                        int lastIdx = idx;
                        for (HashEntry<K,V> last = next;
                             last != null;
                             last = last.next) {
                            int k = last.hash & sizeMask;
                            if (k != lastIdx) {
                                lastIdx = k;
                                lastRun = last;
                            }
                        }
                        newTable[lastIdx] = lastRun;//挂到新的下标上

                        // 复制链表上剩余的节点
                        for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                            int k = p.hash & sizeMask;
                            HashEntry<K,V> n = newTable[k];
                            newTable[k] = new HashEntry<K,V>(p.key, p.hash,
                                                             n, p.value);
                        }
                    }
                }
            }
            table = newTable;
        }

因为 HashEntry 的 next 属性是 final ，所以不能改变其指针的指向，只能对原来的链表进行复制。有统计规律指出，其实需要移动到新的下标上的元素只有 1/6.所以上面那一段代码对于性能的改善还是很重要的。