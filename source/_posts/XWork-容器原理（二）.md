---
title: XWork 容器原理（二）
tags:
  - Struts2
date: 2015-07-27 20:06:23
categories: 技术
---

## XWork宏观视图

### 数据流体系


> ActionContext 和 ValueStack 之间的关系是从属关系。

![](http://i.imgur.com/kmElK5E.png)

上图是一个ActionContext类的一个视图轮廓。从图中可以看出，ActionContext实现的功能应有尽有，包括设置ValueStack和Container等等。

对于XWork框架对象的访问：getContainer，getValueStack，getActionInvocation。

对于数据对象的访问：getSession，getApplication，getParameters。



> ActionContext 是 XWork 的数据流实现元素，作为一个数据载体，既负责数据存储，也负责数据共享。

ValueStack是一个具备表达式引擎计算能力的数据结构，表达式引擎的引入就是为了解决数据访问和数据传输的困境。

#### 数据存储

		public class ActionContext implements Serializable {

			private Map<String, Object> context;

		    /**
		     * Creates a new ActionContext initialized with another context.
		     *
		     * @param context a context map.
		     */
		    public ActionContext(Map<String, Object> context) {
		        this.context = context;
		    }
			
			....	

		}


ActionContext 真正的存储空间是位于内部一个Map类型的变量 context。ActionContext 将所有的数据对象以特定的键值（Key）存储于context之中。而与此同时ActionContext也提供了存取这些对象的方式。



#### 数据共享

数据共享涉及一个线程安全的问题。

			public class ActionContext implements Serializable {
	
				static ThreadLocal<ActionContext> actionContext = new ThreadLocal<ActionContext>();
	
			  	public static void setContext(ActionContext context) {
        			actionContext.set(context);
    			}

			    
			    public static ActionContext getContext() {
			        return actionContext.get();
			    }
				
				....	
	
			}

从上面的源码中可以看出，在ActionContext 内部封装了一个静态的ThreadLocal的实例。使用ThreadLocal模式解决了多线程访问的问题。


#### ValueStack

OGNL的三要素：Root对象，环境，表达式。说的通俗一点就是：对谁（Root对象）在哪儿（上下文环境）做什么（表达式）。

ValueStack的数据结构：

		public interface ValueStack {
		
			public abstract CompoundRoot getRoot();
		
			public abstract Object peek();
		
		    public abstract Object pop();
		
		    public abstract void push(Object o);
		
		    public abstract void set(String key, Object o);
		
		    public abstract int size();
	
			....
		}


ValueStack被定义为一个接口而不是实现类。真正的实现类：OgnlValueStack.

		public class OgnlValueStack implements Serializable, ValueStack, ClearableValueStack, MemberAccessValueStack 
		{
			protected OgnlValueStack(ValueStack vs, XWorkConverter xworkConverter, CompoundRootAccessor accessor, boolean allowStaticAccess) 
			{
				setRoot(xworkConverter, accessor, new CompoundRoot(vs.getRoot()), allowStaticAccess);
		    }
		
			protected void setRoot(XWorkConverter xworkConverter, CompoundRootAccessor accessor, CompoundRoot compoundRoot,
		                           boolean allowStaticMethodAccess) 
			{
		        this.root = compoundRoot;
		        this.securityMemberAccess = new SecurityMemberAccess(allowStaticMethodAccess);
		        this.context = Ognl.createDefaultContext(this.root, accessor, new OgnlTypeConverterWrapper(xworkConverter), securityMemberAccess);
		        context.put(VALUE_STACK, this);
		        Ognl.setClassResolver(context, accessor);
		        ((OgnlContext) context).setTraceEvaluations(false);
		        ((OgnlContext) context).setKeepLastEvaluation(false);
		    }

			....

		}

在这里，OgnlValueStack使用了一个装饰器模式，真正在内部起核心作用的是一个叫做 CompoundRoot 的数据结构。这个结构的源代码如下：

		public class CompoundRoot extends ArrayList {
		
		    public CompoundRoot() {
		    }
		
		    public CompoundRoot(List list) {
		        super(list);
		    }
		
		
		    public CompoundRoot cutStack(int index) {
		        return new CompoundRoot(subList(index, size()));
		    }
		
		    public Object peek() {
		        return get(0);
		    }
		
		    public Object pop() {
		        return remove(0);
		    }
		
		    public void push(Object o) {
		        add(0, o);
		    }
		}
可以看出，CompoundRoot继承了一个ArrayList的数据结构。

	public class OgnlValueStackFactory implements ValueStackFactory {
    
	    private XWorkConverter xworkConverter;
	    private CompoundRootAccessor compoundRootAccessor;
	    private TextProvider textProvider;
	    private Container container;
	    private boolean allowStaticMethodAccess;
	
	    @Inject
	    public void setXWorkConverter(XWorkConverter conv) {
	        this.xworkConverter = conv;
	    }
	    
	    @Inject("system")
	    public void setTextProvider(TextProvider textProvider) {
	        this.textProvider = textProvider;
	    }
	    
	    @Inject(value="allowStaticMethodAccess", required=false)
	    public void setAllowStaticMethodAccess(String allowStaticMethodAccess) {
	        this.allowStaticMethodAccess = "true".equalsIgnoreCase(allowStaticMethodAccess);
	    }
	
	    public ValueStack createValueStack() {
	        ValueStack stack = new OgnlValueStack(xworkConverter, compoundRootAccessor, textProvider, allowStaticMethodAccess);
	        container.inject(stack);
	        stack.getContext().put(ActionContext.CONTAINER, container);
	        return stack;
	    }
	
	    public ValueStack createValueStack(ValueStack stack) {
	        ValueStack result = new OgnlValueStack(stack, xworkConverter, compoundRootAccessor, allowStaticMethodAccess);
	        container.inject(result);
	        stack.getContext().put(ActionContext.CONTAINER, container);
	        return result;
	    }
	    
	    @Inject
	    public void setContainer(Container container) throws ClassNotFoundException {
	        Set<String> names = container.getInstanceNames(PropertyAccessor.class);
	        for (String name : names) {
	            Class cls = Class.forName(name);
	            if (cls != null) {
	                if (Map.class.isAssignableFrom(cls)) {
	                    PropertyAccessor acc = container.getInstance(PropertyAccessor.class, name);
	                }
	                OgnlRuntime.setPropertyAccessor(cls, container.getInstance(PropertyAccessor.class, name));
	                if (compoundRootAccessor == null && CompoundRoot.class.isAssignableFrom(cls)) {
	                    compoundRootAccessor = (CompoundRootAccessor) container.getInstance(PropertyAccessor.class, name);
	                }
	            }
	        }
	
	        names = container.getInstanceNames(MethodAccessor.class);
	        for (String name : names) {
	            Class cls = Class.forName(name);
	            if (cls != null) {
	                OgnlRuntime.setMethodAccessor(cls, container.getInstance(MethodAccessor.class, name));
	            }
	        }
	
	        names = container.getInstanceNames(NullHandler.class);
	        for (String name : names) {
	            Class cls = Class.forName(name);
	            if (cls != null) {
	                OgnlRuntime.setNullHandler(cls, new OgnlNullHandlerWrapper(container.getInstance(NullHandler.class, name)));
	            }
	        }
	        if (compoundRootAccessor == null) {
	            throw new IllegalStateException("Couldn't find the compound root accessor");
	        }
	        this.container = container;
	    }
	}

观察上面的源代码发现，原来OgnlValueStack是通过OgnlValueStackFactory 创建出来的，在其对 ValueStack进行初始化的过程中，会为OGNL的计算指定许多默认的实现方式。所有的初始化工作都在其中完成，由于OGNL的运行参数都属于 Struts2/Work 需要定义的内置变量，因而选择了在实施依赖注入时（在setContainer方法中），完成这些参数的实现方式的选择和设置。

> ValueStack 是XWork框架中的核心元素之一，是 XWork 框架进行 OGNL 计算的实际场所，也是整个框架进行数据访问的基础。

在XWork的框架范围之内，如果要针对某一对象进行表达式引擎的计算，只要将这个对象置于 ValueStack 之中，通过其计算接口进行操作即可。

### 控制流体系

控制流体系从元素构成上要比数据流复杂得多。在其中，有两层关系值得探讨：事件处理节点与事件驱动元素之间的关系以及事件处理节点和辅助事件处理节点之间的关系。

#### 事件处理节点与驱动元素

在XWork的控制流中，事件处理驱动元素对于事件处理节点元素形成调用关系。

ActionInvocation 对于事件处理节点的执行调度，是整个XWork控制流体系运行的核心。因为每个事件处理节点都被定义为接口。**使用XWork进行事件处理的过程就是对每个事件处理节点的实例化过程。**但是无论如何定义和扩展这些节点，都离不开 ActionInvocation 的统一调度执行。

#### 事件处理节点和辅助事件处理节点

Interceptor和Action之间形成了一个包裹结构。Action外面包着层层Interceptor，在计算机语言中是一个栈结构。通常把Action和Interceptor之间所表现出来的栈特性叫**XWork执行栈**。

ActionInvocation 在对事件处理节点进行逻辑调用时，是将Action与Interceptor这两大元素捆绑在一起进行调度的。




