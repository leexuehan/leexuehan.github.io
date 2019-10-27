---
title: 《Core Java》学习系列（一）
tags:
  - Java
date: 2015-07-13 22:52:28
categories: 技术
---

## Java里面的String



>Conceptually, Java Strings are sequences of Unicode characters.
>
>Java里面的String都是Unicode字符串
>

>Java does not have a built-in string type.
>
>java没有内建的字符串类型。

>Each quoted string is an instance of the String class.

>每个引用的字符串都是一个String类的实例。

### 字符串是永恒的：

>****Strings Are Immutable****

>The String class gives no methods that let you change a character in an existing string.

>Just as the number 3 is always number 3,the string "hello" will always contain the code unit sequence describing the characters h,e,l,l,o.You cannot change these values.You can change the contents of the String variable and make it refer to a different string,just as you can make a numeric variable currently holding the value 3 hold the value 4.

这段话说得非常明白，就像数字3总是3一样，也就是Java里面的所有字符串都是不可更改的，你只能更改引用它的那个变量，就像让一个原来存储值为3的变量存储4一样。

关于效率的论述：

>Isn't that a lot less efficient?It would be simpler to change the code units than to build up a whole new string from scratch.Well,yes and no.Indeed,it isn't efficient to generate a new string that holds the concatenation.But immutable strings have one great advantage:the compiler can arrange that strings are shared.

>Overall,the designers of Java decided that the efficiency of sharing outweighs the efficiency of string editing by extracting substrings and concatening.

这段话意思是：虽然有人说改变原来的编码会变得简单一点，但是让字符串不能变化有一个优点：编译器可以安排字符串共享。Java 的发明者自有自己的考虑，他认为共享的效率要胜过通过截取拼接来编辑字符串的效率。

而且我们大部分时间只拿字符串来作比较。

### 和C的区别

关于字符串，作者拿来与C和C++做了区别：

>C程序员眼中的字符串：

>char greeting[] = "Hello";//字符数组

>在Java中可不如此，他认为，"A Java string is roughly analogous to a char* pointer."即，在Java里面字符串更像一个char型的常量指针，如下:

>char* greeting = "Hello";

然后他用C的方式来模拟了改变Java里面的字符串变量的引用的过程：

		char *temp = malloc(6);
		strncpy(temp, greeting, 3);
		strncpy(temp+3,"p!",3);
		greeting = temp;

即改变字符串变量的内容只是改变了变量指向的地址。而原来地址里面的内容，将会被JVM自动回收。


怎么测试字符串的相等？

>To test whether two strings are equal, use *equals* method. Never use == to compare strings lest you end up with a program with the worst kind of bug.

“==”号只是比较两个字符串变量指向的地址是否相等。注意区分。


### charAt()方法的雷区

看下面的一个例子：

这是一句话：

> @ is the set of integers.

执行以下代码：

		char ch = setence.charAt(1);

猜一猜输出结果。。。。

一般情况下肯定是空格。

但是如果句首的字符是一个UTF-16编码的字母，则结果就不一样了。用charAt()方法只能得到其编码的后半部分。要注意这一点。

### String API

列出了常用的方法：

		char charAt(int index);
		int compareTo(String other);
		boolean endWith(String other);
		boolean equals(Object other);
		boolean equalsIgnoreCase(String other);
		int lastIndexOf(String str);
		int length();
		String replace(CharSequence oldString, CharSequence newString);
		boolean startWith(String prefix);
		String toLowerCase();
		String toUpperCase();
		String trim();

### 构造字符串

>Occasionally, you need to build up strings from shorter strings,such as keystrokes or words from files.It would be iniefficient to use string concatenation for this purpose.

因为每次拼接字符串都会产生一个废弃的就字符串对象。所以需要用到StringBuilder对象，直接在原来的字符串上进行操作。

每次拼接的时候，用append方法即可。

常用的API

	int length();
	StringBuilder append(String str);
	StringBuilder insert(int offset, String str);
	StringBuilder delete(int startIndex, int endIndex);

StringBuilder 的前任是 StringBuffer,效率较低，但是支持多线程。通常，如果只是单线程进行编辑的话，还是使用StringBuilder较好。


### 说点废话

Core Java上确实有很多值得细细品味啊，看了这一篇关于字符串的介绍，感觉基础还是不牢固啊，还得加强，后面还会有系列文章，讲Core Java。