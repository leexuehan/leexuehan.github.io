---
title: HashMap 深度解析
tags:
  - Collection
  - Java
date: 2015-07-26 19:43:21
categories: 技术
---

今天开始从源码角度分析Java集合框架的实现原理。

本文主要讲HashMap的源码实现方式。

对此，有一篇文章讲的相当透彻，本人也是受益匪浅，不敢掠人之美，贴上链接

http://zhangshixi.iteye.com/blog/672697

之所以还要自己写一篇，一方面是为了能使自己印象更深刻一点，想从自己所理解的角度来谈一下自己的理解；另一方面此文中提到的源码实现都是基于JDK 1.6的，而在新的JDK版本中已经大不一样了，HashMap的实现有较大的变动。

本文中的源码都是JDK1.8的。

### HashMap的数据结构


> 在JDK1.6中，HashMap采用**位桶+链表**实现，即使用链表处理冲突，同一hash值的链表都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。
> 
> 而JDK1.8中，HashMap采用**位桶+链表+红黑树**实现，当链表长度超过阈值（8）时，将链表转换为红黑树，这样大大减少了查找时间。


1.链表

			//单向链表节点元素Node，实现了Map.Entry接口
			static class Node<K,V> implements Map.Entry<K,V> {
		        final int hash;
		        final K key;
		        V value;
		        Node<K,V> next;//指向下一个Node
		
		        Node(int hash, K key, V value, Node<K,V> next) {
		            this.hash = hash;
		            this.key = key;
		            this.value = value;
		            this.next = next;
		        }
		
		        public final K getKey()        { return key; }
		        public final V getValue()      { return value; }
		        public final String toString() { return key + "=" + value; }
		
		        public final int hashCode() {
		            return Objects.hashCode(key) ^ Objects.hashCode(value);
		        }
		
		        public final V setValue(V newValue) {
		            V oldValue = value;
		            value = newValue;
		            return oldValue;
		        }
		
				//判断两个node是否相等,若key和value都相等，返回true。可以与自身比较为true
		        public final boolean equals(Object o) {
		            if (o == this)
		                return true;
		            if (o instanceof Map.Entry) {
		                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
		                if (Objects.equals(key, e.getKey()) &&
		                    Objects.equals(value, e.getValue()))
		                    return true;
		            }
		            return false;
		        }
		    }

2.红黑树

鉴于篇幅，三百多行的代码此处略去了。如有机会，此后在研究红黑树的时候，再专门拿出来研究研究。


3.位桶

		/**
	     * The table, initialized on first use, and resized as
	     * necessary. When allocated, length is always a power of two.
	     * (We also tolerate length zero in some operations to allow
	     * bootstrapping mechanics that are currently not needed.)
	     */
	    transient Node<K,V>[] table;

在英文注释中说得很明白了，在第一次使用的时候进行初始化，必要的时候进行扩容，长度总是2的幂次方。



### HashMap的存取实现

#### 存放

在JDK 1.8 中的源码中存放元素的代码如下：

		public V put(K key, V value) {
	        return putVal(hash(key), key, value, false, true);
	    }
		/**
	     * Implements Map.put and related methods
	     *
	     * @param hash hash for key
	     * @param key the key
	     * @param value the value to put
	     * @param onlyIfAbsent if true, don't change existing value
	     * @param evict if false, the table is in creation mode.
	     * @return previous value, or null if none
	     */
	
		final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
	                   boolean evict) {
	        Node<K,V>[] tab; Node<K,V> p; int n, i;
	
			//如果位桶为空，要进行扩容
	        if ((tab = table) == null || (n = tab.length) == 0)
	            n = (tab = resize()).length;
	
			// i = hash & (length-1)得到对象的保存位，如果为空，则直接进行put操作
	        if ((p = tab[i = (n - 1) & hash]) == null)
	            tab[i] = newNode(hash, key, value, null);
	
			// 如果相应的位置上已经有了元素
	        else {
	            Node<K,V> e; K k;
	
				//第一节节点hash值同，且key值与插入key相同  
	            if (p.hash == hash &&
	                ((k = p.key) == key || (key != null && key.equals(k))))
	                e = p;
	            else if (p instanceof TreeNode) //属于红黑树处理冲突
	                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
	            else {	//属于链表处理冲突  
	                for (int binCount = 0; ; ++binCount) {
	
						//p 第一次指向表头,以后依次后移
	                    if ((e = p.next) == null) {
	
							//e为空，表示已到表尾也没有找到key值相同节点，则新建节点 
	                        p.next = newNode(hash, key, value, null);
	
							//新增节点后如果节点个数到达阈值，则将链表转换为红黑树 
	                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
	                            treeifyBin(tab, hash);
	                        break;
	                    }
	                    if (e.hash == hash &&
	                        ((k = e.key) == key || (key != null && key.equals(k))))
	                        break;
	                    p = e;//更新p指向下一个节点
	                }
	            }
	            if (e != null) { // existing mapping for key
	                V oldValue = e.value;
	                if (!onlyIfAbsent || oldValue == null)
	                    e.value = value;
	                afterNodeAccess(e);
	                return oldValue;
	            }
	        }
	        ++modCount;
	        if (++size > threshold)
	            resize();
	        afterNodeInsertion(evict);
	        return null;
	    }

#### 读取元素

		public V get(Object key) {
	        Node<K,V> e;
	        return (e = getNode(hash(key), key)) == null ? null : e.value;
	    }
	
	    /**
	     * Implements Map.get and related methods
	     *
	     * @param hash hash for key
	     * @param key the key
	     * @return the node, or null if none
	     */
	    final Node<K,V> getNode(int hash, Object key) {
	        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
			//得到对象的保存位
	        if ((tab = table) != null && (n = tab.length) > 0 &&
	            (first = tab[(n - 1) & hash]) != null) {
	            if (first.hash == hash && // always check first node
	                ((k = first.key) == key || (key != null && key.equals(k))))
	                return first;
	            if ((e = first.next) != null) {
					//如果第一个节点是TreeNode,说明采用的是数组+红黑树结构处理冲突  
	        		//遍历红黑树，得到节点值 
	                if (first instanceof TreeNode)
	                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
	                do {
	                    if (e.hash == hash &&
	                        ((k = e.key) == key || (key != null && key.equals(k))))
	                        return e;
	                } while ((e = e.next) != null);
	            }
	        }
	        return null;
	    }

#### hash

在上面的存取节点时用到的 hash 函数：

	 	static final int hash(Object key) {
	        int h;
	        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	    }

HashMap则通过 h&（length-1）替代取模，得到所在数组位置，这样效率会高很多。

#### 位桶扩容

构造hash表时，如果不指明初始大小，默认大小为16（即Node数组大小16），如果Node[]数组中的元素达到（填充比*Node.length）

请看代码：


```Java
		//可用来初始化HashMap大小 或重新调整HashMap大小 变为原来2倍大小  
	   final Node<K,V>[] resize() {  
	       Node<K,V>[] oldTab = table;  
	       int oldCap = (oldTab == null) ? 0 : oldTab.length;  
	       int oldThr = threshold;  
	       int newCap, newThr = 0;  
	       if (oldCap > 0) {  
	           if (oldCap >= MAXIMUM_CAPACITY) {//超过1>>30大小,无法扩容只能改变 阈值  
	               threshold = Integer.MAX_VALUE;  
	               return oldTab;  
	           }  
	           else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  
	                    oldCap >= DEFAULT_INITIAL_CAPACITY)//新的容量为旧的2倍 最小也是16  
	               newThr = oldThr << 1; // 扩容阈值加倍  
	       }  
	       else if (oldThr > 0)   
	           newCap = oldThr;//oldCap=0 ,oldThr>0此时newThr=0   
	       else {               //oldCap=0,oldThr=0 相当于使用默认填充比和初始容量 初始化  
	           newCap = DEFAULT_INITIAL_CAPACITY;  
	           newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  
	       }  
	         
	       if (newThr == 0) {  
	           float ft = (float)newCap * loadFactor;  
	           newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?  
	                     (int)ft : Integer.MAX_VALUE);  
	       }  
	       threshold = newThr;  
	       @SuppressWarnings({"rawtypes","unchecked"})  
	           Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  
	       //数组辅助到新的数组中，分红黑树和链表讨论  
	       table = newTab;  
	       if (oldTab != null) {  
	           for (int j = 0; j < oldCap; ++j) {  
	               Node<K,V> e;  
	               if ((e = oldTab[j]) != null) {  
	                   oldTab[j] = null;  
	                   if (e.next == null)  
	                       newTab[e.hash & (newCap - 1)] = e;  
	                   else if (e instanceof TreeNode)  
	                       ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  
	                   else { // preserve order  
	                       Node<K,V> loHead = null, loTail = null;  
	                       Node<K,V> hiHead = null, hiTail = null;  
	                       Node<K,V> next;  
	                       do {  
	                           next = e.next;  
	                           if ((e.hash & oldCap) == 0) {  
	                               if (loTail == null)  
	                                   loHead = e;  
	                               else  
	                                   loTail.next = e;  
	                               loTail = e;  
	                           }  
	                           else {  
	                               if (hiTail == null)  
	                                   hiHead = e;  
	                               else  
	                                   hiTail.next = e;  
	                               hiTail = e;  
	                           }  
	                       } while ((e = next) != null);  
	                       if (loTail != null) {  
	                           loTail.next = null;  
	                           newTab[j] = loHead;  
	                       }  
	                       if (hiTail != null) {  
	                           hiTail.next = null;  
	                           newTab[j + oldCap] = hiHead;  
	                       }  
	                   }  
	               }  
	           }  
	       }  
	       return newTab;  
	   }  
```



在HashMap实现中还可以看到如下代码取代了以前版本JDK1.6的while循环来保证哈希表的容量一直是2的整数倍数，用移位操作取代了循环移位。

	//这段代码保证HashMap的容量总是2的n次方  
	static final int tableSizeFor(int cap) {  
	    int n = cap - 1;  
	    n |= n >>> 1;  
	    n |= n >>> 2;  
	    n |= n >>> 4;  
	    n |= n >>> 8;  
	    n |= n >>> 16;  
	    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;  
	}  

可以从源码看出，在HashMap的构造函数中，都直接或间接的调用了tableSizeFor函数。


原因：

> length为2的整数幂保证了length-1最后一位（当然是二进制表示）为1，从而保证了取索引操作 h&（length-1）的最后一位同时有为0和为1的可能性，保证了散列的均匀性。反过来讲，当length为奇数时，length-1最后一位为0，这样与h按位与的最后一位肯定为0，即索引位置肯定是偶数，这样数组的奇数位置全部没有放置元素，浪费了大量空间。

### HashMap的性能参数
HashMap 包含如下几个构造器：


- HashMap()：构建一个负载因子为 0.75 的 HashMap。


- HashMap(int initialCapacity)：构建一个初始容量为 initialCapacity，负载因子为 0.75 的 HashMap。


- HashMap(int initialCapacity, float loadFactor)：以指定初始容量、指定的负载因子创建一个 HashMap。


HashMap的基础构造器HashMap(int initialCapacity, float loadFactor)带有两个参数，它们是初始容量initialCapacity和加载因子loadFactor。


initialCapacity：HashMap的最大容量，即为底层数组的长度。

**loadFactor：负载因子loadFactor定义为：散列表的实际元素数目(n)/ 散列表的容量(m)。**

在 JDK 1.5 中

		threshold = (int)(capacity * loadFactor); 

但是在 JDK 1.8里

		threshold = tableSizeFor(initialCapacity);

> 负载因子衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，反之愈小。对于使用链表法的散列表来说，查找一个元素的平均时间是O(1+a)，因此如果负载因子越大，对空间的利用更充分，然而后果是查找效率的降低；如果负载因子太小，那么散列表的数据将过于稀疏，对空间造成严重浪费。


		public HashMap(int initialCapacity) {
	        this(initialCapacity, DEFAULT_LOAD_FACTOR);
	    }
	
	    public HashMap() {
	        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
	    }


​	
		public HashMap(int initialCapacity, float loadFactor) {
	        if (initialCapacity < 0)
	            throw new IllegalArgumentException("Illegal initial capacity: " +
	                                               initialCapacity);
	        if (initialCapacity > MAXIMUM_CAPACITY)
	            initialCapacity = MAXIMUM_CAPACITY;
	        if (loadFactor <= 0 || Float.isNaN(loadFactor))
	            throw new IllegalArgumentException("Illegal load factor: " +
	                                               loadFactor);
	        this.loadFactor = loadFactor;
	        this.threshold = tableSizeFor(initialCapacity);
	    }


### fail-fast机制

我们知道java.util.HashMap不是线程安全的，因此如果在**使用迭代器进行遍历的过程中**有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓**fail-fast策略**。

下面是一个HashMap迭代器的抽象类源码：

	 abstract class HashIterator {
	    Node<K,V> next;        // next entry to return
	    Node<K,V> current;     // current entry
	    int expectedModCount;  // for fast-fail
	    int index;             // current slot
		
		//这一策略在源码中的实现是通过modCount域，modCount顾名思义就是修改次数，对HashMap内容的修
		//改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的expectedModCount。
	    HashIterator() {
	        expectedModCount = modCount;
	        Node<K,V>[] t = table;
	        current = next = null;
	        index = 0;
	        if (t != null && size > 0) { // advance to first entry
	            do {} while (index < t.length && (next = t[index++]) == null);
	        }
	    }
	
	    public final boolean hasNext() {
	        return next != null;
	    }
	
	    final Node<K,V> nextNode() {
	        Node<K,V>[] t;
	        Node<K,V> e = next;
	        if (modCount != expectedModCount) //fail-fast机制
	            throw new ConcurrentModificationException();
	        if (e == null)
	            throw new NoSuchElementException();
	        if ((next = (current = e).next) == null && (t = table) != null) {
	            do {} while (index < t.length && (next = t[index++]) == null);
	        }
	        return e;
	    }
	
	    public final void remove() {
	        Node<K,V> p = current;
	        if (p == null)
	            throw new IllegalStateException();
	        if (modCount != expectedModCount)
	            throw new ConcurrentModificationException();
	        current = null;
	        K key = p.key;
	        removeNode(hash(key), key, null, false, false);
	        expectedModCount = modCount;
	    }
	}

 这一策略在源码中的实现是通过modCount域，modCount顾名思义就是修改次数，对HashMap内容的修改都将增加这个值，那么在**迭代器初始化过程中**会将这个值赋给迭代器的expectedModCount。

下面的代码中的KeyIterator继承了上面的HashIterator，用来遍历HashMap中的key。

	final class KeySet extends AbstractSet<K> {
		public final Iterator<K> iterator()    { return new KeyIterator(); }
	}

### equals 和 == 的区别

1.如果是基本变量，没有hashcode和equals方法，基本变量的比较方式就只有==；

2.如果是变量，由于在java中所有变量定义都是一个指向实际存储的一个句柄（你可以理解为c++中的指针），在这里==是比较句柄的地址（你可以理解为指针的存储地址），而不是句柄指向的实际内存中的内容，如果要比较实际内存中的内容，那就要用equals方法，但是！！！

如果是你自己定义的一个类，比较自定义类用equals和==是一样的，都是比较句柄地址，因为自定义的类是继承于object，而object中的equals就是用==来实现的，源码：

	 	public boolean equals(Object obj) {
	        return (this == obj);
	    }


那为什么我们用的String等等类型equals是比较实际内容呢，是因为String等常用类已经重写了object中的equals方法，让equals来比较实际内容，源码：

		public boolean equals(Object anObject) {
	        if (this == anObject) {
	            return true;
	        }
	        if (anObject instanceof String) {
	            String anotherString = (String)anObject;
	            int n = value.length;
	            if (n == anotherString.value.length) {
	                char v1[] = value;
	                char v2[] = anotherString.value;
	                int i = 0;
	                while (n-- != 0) {
	                    if (v1[i] != v2[i])
	                        return false;
	                    i++;
	                }
	                return true;
	            }
	        }
	        return false;
	    }


好累啊，终于写完了，Java的一大好处就是所有的源码你想看都可以看到，同时代码的易读性也很好。学习源码是了解Java 的一大法宝。