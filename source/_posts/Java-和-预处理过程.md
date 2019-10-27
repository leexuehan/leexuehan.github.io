---
title: Java 和 预处理过程
tags:
  - Java
date: 2015-07-10 18:08:08
categories: 技术
---

今天，在看一篇关于编译过程的文章的过程中，看到了C语言有预处理过程，就想着Java有没有预处理过程。在网上查询后，看到了下面的答案。

### 为什么Java没有预处理过程？

> 从编译的过程能看出原因，因为java没有预处理过程。c语言的编译过程是这样的 C源程序－>编译预处理－>编译－>优化程序－>汇编程序－>链接程序－>可执行文件。预处理过程会对于宏定义、根据条件决定编译时是否包含某些代码，以及空白字符和注释进行替换和处理，这一步是独立在编译过程之外的，编译程序不会处理这些内容。
> 而对于java来说，编译的全过程就是，编译-》字节码-》解释执行。在编译的过程，会进行一些常量替换，去掉注释的工作。

在StackOverFlow上有下面的一段答案：


> A 'static final' of a primitive or String type is defined by the Java compiler spec (not JIT) to be a constant. It is not "inlined" in the sense of JIT, it is actually directly compiled into that part of the code just like a #define is in C.


### Java中的import类似于 C++的 #include 吗？

> **Import despite the name doesn't "import" anything, it just let you call the classes without the fully qualified name.**
> 
> To clarify, if I do an import java.util.ArrayList;, now I can refer to ArrayList class as just ArrayList. If I don't do it, I can still use the class, I just have to call it java.util.ArrayList.
> 
> If you import whole packages with *, the worst thing it can happen is that there is a name clash, thus, you've to use the full name to refer to a Java class, but it doesn't use more memory at runtime.
> 
> Classes in java.lang are automatically "imported".
> 
> Java 1.5 introduced static imports, which enables programmers to refer to imported static members as if they were declared in the class that uses them. They should be used sparingly. An acceptable use is for importing JUnit Assert methods. For instance, with a traditional import:

		import org.junit.Assert;
		...
		Assert.assertEquals(expected, actual);
		With static import:
		
		import static org.junit.Assert.assertEquals;
		...
		assertEquals(expected, actual);
