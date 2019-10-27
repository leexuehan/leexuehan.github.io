---
title: Java对象的序列化
tags:
  - Java
date: 2015-07-13 22:55:20
categories: 技术
---

序列化机制是Java语言内建的一种对象持久化方式，可以很容易的在JVM中的活动对象和字节数组之间转换。它的一个重要用途就是远程方法调用的时候，用来对开发人员屏蔽底层实现细节（远端的开发人员不知道这个对象的具体实现细节，通过序列化技术可以直接还原为一个对象，直接拿来用即可）。
### 什么是序列化

序列化就是将 Java 对象转化成字节序列，这些字节序列可以保存在磁盘上也可以通过网络传输，以备以后重新恢复成原来的对象。其逆过程为反序列化。

### 如何实现序列化

待序列化的类只要实现Serializable接口即可，该接口是一个标记接口，没有任何方法。

实际的序列化由ObjectOutputStream和ObjectInputStream来完成。前者可以调用其writeObject方法来将一个Java对象写入流中，后者可以用readObject方法来从流中读取一个Java对象。

**在默认的序列化实现中，Java对象的非静态和非瞬时域都会被包括进来，而与域的可见性声明没有关系。**

如果你的对象中含有敏感信息，比如user这个对象里面含有password这个域，则如果不想被序列化，必须在前面添加transient关键词。或者添加一个serialPersistentFields域来声明序列化时要包含的域。

	private static final objectStreamField[] serialPersistentFileds = {
		new ObjectStreamFiled("firstName", String.class);
	};


### 自定义对象序列化

通过自定义可以对序列化的过程进行更加细粒度的控制，但是**要在类中添加writeObject和readObject方法。**

在通过ObjectOutputStream的writeObject方法写入对象的时候，如果这个对象定义了writeObject方法，就会调用该方法，并把当前的ObjectOutputStream对象作为参数传递进去。

writeObject方法中一般会包含自定义的序列化逻辑，比如在写入之前修改域的值，或是写入额外的数据等。

在添加自己的逻辑之前，推荐的做法是先调用Java的默认实现，在writeObject方法中通过ObjectOutputStream的defaultWriteObject来完成。在readObject方法实现方式也类似。

		private void writeObject(ObjectOutputStream output) throws IOException {
			output.defaultWriteObject();
			// 在这里添加自定义逻辑
			output.writeUTF("Hello World");
		}

### 对象引用的序列化

假设当程序序列化一个 Teacher 对象时，如果该 Teacher 对象持有一个 Person 对象的引用，则Person 对象也必须是可以序列化的，否则将不可以序列化 Teacher。

还需要注意一个问题：

>假设类 A/B/C， A中持有对 C 的引用， B中也持有对C的引用，则如果将三个对象 都序列化会发生什么呢？
会不会出现C 对象被序列化三次的现象？
当然不会。为了防止这种现象发生， Java序列化机制采用了一种特殊的序列化算法：所有保存到磁盘中的对象都会有一个序列化编号，当程序试图序列化一个对象时，程序将首先检查该对象是否已经被序列化过，如果已经被序列化过，则程序将只是输出一个序列化编号，而不是再次重新序列化该对象。

### 序列化的对象替换

假设你要在网上买东西，在购物表单上添加信息，然后将这个请求发送到远端，即把一个order对象发送到远端，准备在远端将其还原，如果order对象的属性里面有customer的对象引用，则在序列化order的时候，可能也会将customer也序列化。这时候，就涉及到用户的隐私泄露的问题。

所以，必须采取别的方法实现只在远端还原order对象即可。

这个方法就是：用一个替换对象类orderReplace只保存order的ID，在Oder类的writeReplace方法中返回一个OrderReplace对象。这个对象会被作为替代写入到流中，同样，需要OrderReplace类中定义一个readResolve方法，用在读取的时候再转换回Order类对象。

OrderReplace类：

	private static class OrderReplace implements Serializable {
	    private static final long serialVersionUID = 4654546423735192613L;
	    private String orderId;
	    public OrderReplace(Order order) {
	        this.orderId = order.getId();
	    }
	    private Object readResolve() throws ObjectStreamException {
	        //根据orderId查找Order对象并返回
	    }
	}

Order类：

	private Object writeReplace() throws ObjectStreamException {
	    return new OrderReplace(this);
	} 



