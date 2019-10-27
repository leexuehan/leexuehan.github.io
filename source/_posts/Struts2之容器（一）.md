---
title: Struts2之容器（一）
tags:
  - Struts2
date: 2015-07-24 20:55:36
categories: 技术
---

本文从原理性角度对于XWork容器进行介绍。

先看两大问题：

> 在程序的运行期间，应该如何创建我们所需要的对象？
> 
> 当创建一个新的对象时，如何保证与这个对象所关联的依赖关系也能够被正确地创建出来？

这是每个框架在进行设计时必须考虑的问题。

业界的一个公认的解决办法就是:


> 为了更好地管理对象的生命周期，有必要在程序逻辑之外引入一个额外的编程元素，这个元素就是**容器**。

本文将介绍此容器。

### IOC

Inverse Of Control,即控制反转。控制反转是对象生命周期管理的一个核心概念。
什么是控制反转呢？

在面向对象编程语言中，对于实现业务逻辑，离不开多个对象的彼此合作。而动作特性对象的嵌套引用是对象之间合作完成业务逻辑的基本途径，也是对象形成依赖关系的根本原因。

**每个对象自身对于逻辑的执行能力，被其所依赖的对象反向控制了，这就是控制反转的本质含义。**

当对象之间的这种依赖关系与运行期对象的构建机制相结合时，情况就变得更为复杂了。因为在对象的创建时候，除了需要考虑这个对象本身作为一个实例的创建过程，还需要考虑与这个对象形成**依赖关系**的实例化过程。

前一个过程能够使对象自身的生命周期得到控制和管理，后一过程则保证了程序在运行时的使用不会受到其所**依赖的对象的生命周期的限制。**


### 容器的引入

如果不引入容器会发生什么？

获取依赖对象的过程将有种种弊端：

1.对象将被频繁创建，效率大大降低

2.对象的创建逻辑与业务逻辑代码高度耦合，使得一个对象的逻辑关注度大大分散

3.程序的执行效率大大降低

于是，业界的软件大师就提出引入容器的想法，几乎所有的开源框架都有自己的容器实现。

容器设计的原则？

1.容器应该被设计成一个全局的、统一的编程元素

2.在最大程度上降低容器对业务逻辑的入侵

3.容器应该提供简单而全面的对象操作接口


### XWork 概述

XWork框架的容器被定义称为一个Java接口，源代码框架如下：

		public interface Container extends Serializable {

		String DEFUALT_NAME = "default";

		//实施依赖注入
		void inject(Object object);

		//根据type和name作为唯一标识，获取容器中的Java类实例
		<T> T getInstance(Class<T> type,String name);

		<T> T getInstanceNames(Class<?> type);
		
		//设置当前线程的作用范围
		void setScopeStrategy(Scope.Strategy scopeStrategy);
	
		void removeScopeStrategy(); 

		}

从上面的代码来看，容器的接口包括两类重要的方法，获取对象实例和实施依赖注入。

#### 获取对象实例

当调用容器的getInstance方法来获取对象实例时，我们只能够获取那些**被容器接管的对象**的实例。

哪些时被容器接管的对象的实例呢？

XWork 容器管理的对象包括所有的框架配置定义的容器配置元素。

这些对象主要分为三类：

1.在bean节点中声明的框架内部对象

2.在bean 节点中声明的自定义对象

3.在constant节点和Properties文件中声明的系统运行参数

#### 实施依赖注入

我们对inject方法的定义是：只要传入一个对象的实例，容器将负责建立起传入对象与容器托管对象的依赖关系。

> 调用XWork容器的inject方法能够帮助我们将容器所管理的对象（包括框架内置对象以及系统的运行参数）注入到任意的对象实例中去，从而建立任意对象与框架元素沟通的桥梁。
> 


inject方法的调用流程：


> 当某个对象实例作为参数传入方法之后，该方法会扫描传入对象内部声明有 @inject 这个Annotation的字段、方法和构造函数，方法参数并将容器托管的对象注入它们内部，从而建立起传入对象与容器托管对象之间的依赖关系。

由此可见，整个流程的调用过程被一个神秘的 Annotation 有效驱动。

这个Annotation可以被设置到任何对象的方法、构造函数、内部实例变量、或者参数变量中。**它既可以加入到Struts2/XWork的内置对象上，也可以加到任意自行编写的对象上。一旦加入我们自己定义的对象之中，那么他就建立起了自定义对象和容器托管对象之间的联系。**

因为被加入了@inject这个Annotation的方法、构造函数、内部实例变量或者方法参数变量，实际上在告诉容器：请为我注入由容器托管的对象实例。

当我们需要找容器帮忙时，只要在恰当的地方加入一个标识符Annotation，容器在进行依赖注入操作时，就会知晓并接管整个过程了。

所以，可以知道XWork容器进行对象依赖注入操作主要分为两个步骤：

1.为某个对象的方法、构造函数、内部实例、方法参数变量加入@inject的Annotation；

2.调用容器的inject方法，完成被加入Annotation的那些对象的依赖注入。

### 一个依赖注入的代码实例

		public interface IPerson {
			void say();
		}
		====================================  
		public class Person implements IPerson {
			public void say() {
				System.out.println("this is person!");
			}
		}
		====================================  
		public interface IPersonService {
			void say();
		}
		====================================  
		import com.opensymphony.xwork2.inject.Inject;
		public class PersonService implements IPersonService {
			private IPerson person;
			public void say() {
				System.out.println("user person");
				person.say();
			}
			public IPerson getPerson() {
				return person;
			}
			@Inject
			public void setPerson(IPerson person) {
				this.person = person;
			}
		}
		====================================  
		import com.opensymphony.xwork2.inject.Container;
		import com.opensymphony.xwork2.inject.ContainerBuilder;
		public class ContainerTest {
			private Container container;
			public static void main(String[] args) {
				ContainerTest test = new ContainerTest();
				ContainerBuilder builder = new ContainerBuilder();
				builder = builder.factory(IPerson.class, Person.class);
				builder = builder.factory(IPersonService.class, PersonService.class);
				test.container = builder.create(true);
				IPersonService ps = test.container.getInstance(IPersonService.class);
				ps.say(); // 输出user person   this is person!
			}
		}




下一篇将从源码的实现讲容器的实现。
