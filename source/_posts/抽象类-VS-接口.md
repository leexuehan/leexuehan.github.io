---
title: 抽象类 VS 接口
tags:
  - Java
date: 2015-09-06 17:17:07
categories: 技术
---

什么是抽象类？

在面向对象的概念中，我们知道所有的对象都是通过类来描绘的，但是反过来却不是这样。并不是所有的类都是用来描绘对象的，如果一个类中**没有包含足够的信息**来描绘一个具体的对象，这样的类就是抽象类。

抽象类往往用来表征我们在对问题领域进行分析、 设计中得出的抽象概念，是对一系列看上去不同，但是本质上相同的具体概念的抽象。

比如：如果我们进行一个图形编辑软件的开发，就会发现问题领域存在着圆、 三角形这样一些具体概念，它们是不同的，但是它们又都属于形状这样一个概念，形状这个概念在问题领域是不存在的，它就是一个抽象概念。正是因为抽象的概念 在问题领域没有对应的具体概念，所以用以表征抽象概念的抽象类是不能够实例化的。

下面从两个层面阐述它们二者之间的区别：

**语法**

共同点：

在abstract class方式中可以有自己的数据成员，也可以有非abstract的成员方法，而在interface方式的实现中，只能够有静态的不能被修改的数据成员（也就是必须是static final 的，不过在interface中一般不定义数据成员），所有的成员方法都是abstract的。

**从某种意义上说，interface是一种特殊形式的 abstract class。**


从编程的角度来看，abstract class和interface都可以用来实现 "design by contract" 的思想。

区别：

1.abstract class 在 Java 语言中表示的是一种继承关系，一个类只能使用一次继承关系(因为Java不支持多继承 -- 转注)。但是，一个类却可以实现多个interface。

2.在abstract class的定义中，我们可以赋予方法的默认行为。但是在interface的定义中，方法却不能拥有默认行为，为了绕过这个限制，必须使用委托，但是这会增加一些复杂性，有时会造成很大的麻烦。

3.实现抽象类和接口的类必须实现其中的所有方法。抽象类中可以有非抽象方法。接口中则不能有实现方法

**设计理念**

abstract class在Java语言中体现了一种继承关系，要想使得 继承关系合理，父类和派生类之间必须存在"is-a"关系，即父类和派生类在概念本质上应该是相同的。

对于interface来说则不然，并不要求interface的实现者和interface定义在概念本质上是一致的， 仅仅是实现了interface定义的契约而已。

比如：

如果用一个灯的例子。

如果用一个抽象类：

		abstract class Light {
			public void on();
			public void off();
		}

如果用一个接口

		interface Light {
			public void on();
			public void off();
		}

看似一样的效果。

但是，如果要再增加一个闪光效果，怎么样呢？


		abstract class Light {
			public void on();
			public void off();
			public void blink();
		}

		interface Light {
			public void on();
			public void off();
			public void blink();
		}

这样的话，就改变了一个灯最固有的性质。尤其是用接口实现的话，就更麻烦了，所有实现原来接口的灯，都必须实现闪光效果，这样显然是不合理的。

其实最合理的一种方式就是：

		abstract class Light {
			public void on();
			public void off();
		}

		interface Bling {
			public void blink();
		}

		class MyLight extends Light implements Bling {
		}