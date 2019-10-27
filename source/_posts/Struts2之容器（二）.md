---
title: Struts2之容器（二）
tags:
  - Struts2
date: 2015-07-25 16:57:22
categories: 技术
---

这一篇将从代码方面介绍，容器的两个基本功能：容器如何获取对象实例，容器如何获取对象之间的依赖关系也就是进行依赖注入。

## 容器功能框架

### 容器获取对象实例

看一个代码示例：

		package com.opensymphony.xwork2;
		
		import com.opensymphony.xwork2.config.Configuration;
		import com.opensymphony.xwork2.config.ConfigurationException;
		import com.opensymphony.xwork2.config.entities.ActionConfig;
		import com.opensymphony.xwork2.config.entities.UnknownHandlerConfig;
		import com.opensymphony.xwork2.inject.Container;
		import com.opensymphony.xwork2.inject.Inject;
		
		import java.util.ArrayList;
		import java.util.HashMap;
		import java.util.List;
		import java.util.Set;
		
		/**
		 * Default implementation of UnknownHandlerManager
		 *
		 * @see com.opensymphony.xwork2.UnknownHandlerManager
		 */
		public class DefaultUnknownHandlerManager implements UnknownHandlerManager {
		
		    private Container container;
		
		    protected ArrayList<UnknownHandler> unknownHandlers;
		
		    @Inject
		    public void setContainer(Container container) {
		        this.container = container;
		        try {
		            build();
		        } catch (Exception e) {
		            throw new ConfigurationException(e);
		        }
		    }
		
		    
			
		    protected void build() throws Exception {
		        Configuration configuration = container.getInstance(Configuration.class);
		        ObjectFactory factory = container.getInstance(ObjectFactory.class);

				//如果 configuration 对象不为空，则一次从configuration对象以及Container中读取UnknownHandler的实例
		        if (configuration != null && container != null) {
		            List<UnknownHandlerConfig> unkownHandlerStack = configuration.getUnknownHandlerStack();
		            unknownHandlers = new ArrayList<UnknownHandler>();
		
		            if (unkownHandlerStack != null && !unkownHandlerStack.isEmpty()) {

		                //get UnknownHandlers in the specified order
		                for (UnknownHandlerConfig unknownHandlerConfig : unkownHandlerStack) {
		                    UnknownHandler uh = factory.buildUnknownHandler(unknownHandlerConfig.getName(), new HashMap<String, Object>());
		                    unknownHandlers.add(uh);
		                }
		            } else {

		                //调用 Container对象的getInstanceNames方法获取所有受容器管理的
						//UnknownHandler实例名称
		                Set<String> unknowHandlerNames = container.getInstanceNames(UnknownHandler.class);
		                for (String unknowHandlerName : unknowHandlerNames) {

							//根据名称调用 container 对象的getInstance方法来获取实例
		                    UnknownHandler uh = container.getInstance(UnknownHandler.class, unknowHandlerName);
		                    unknownHandlers.add(uh);
		                }
		            }
		        }
		    }

			.....
		
		 }

重点观察build方法中的 getInstance 方法和 getInstanceNames 方法，前者用于获取接受容器托管的具体对象实例，后者用于对一个接口的多个不同实现类之间的实例获取的管理。

代码实例中build方法调用的前提是setContainer方法对容器对象的正确初始化。

### 容器的依赖注入

下面是XWork框架中的核心类ActionSupport的源代码的一部分：

		public class ActionSupport implements Action, Validateable, ValidationAware, 
		TextProvider, LocaleProvider, Serializable {
		
		.....

		private TextProvider getTextProvider() {
		        if (textProvider == null) {
		            TextProviderFactory tpf = new TextProviderFactory();
		            if (container != null) {
		                container.inject(tpf);
		            }
		            textProvider = tpf.createInstance(getClass(), this);
		        }
		        return textProvider;
		    }
		
		    @Inject
		    public void setContainer(Container container) {
		        this.container = container;
		    }
		}


可以看出，getTextProvider 和 setContainer 是两个主要方法。从逻辑上讲，getTextProvider以setContainer的存在为基础。

@inject方法这个Annotation的使用，使得setContainer方法在ActionSupport初始化时被注入全局的Container对象。

getTextProvider在运行期被调用，此时全局的容器对象的接口函数可以被随意调用，并完成依赖注入操作。就是代码中看到的 container.inject(tpf) 操作。


几个要点：

1.通过操作容器进行对象操作的基本前提是当前的操作主体能够获得全局容器实例，因而，全局的容器的实例的获取在操作主体的初始化阶段完成；

2.通过操作容器进行的对象操作都是运行期操作；

3.通过操作容器所获取的对象实例都是受到容器托管的对象实例；

4.通过操作容器进行的依赖注入操作，可以针对任意对象进行，该操作可以建立起任意对象和容器托管对象之间的联系。


## XWork容器存储结构

### 对象制造工厂

下面是 ContainerImpl 一段实现代码

	class ContainerImpl implements Container {
	
		final Map<Key<?>, InternalFactory<?>> factories;
		final Map<Class<?>, Set<String>> factoryNamesByType;
	
		ContainerImpl( Map<Key<?>, InternalFactory<?>> factories ) {
			this.factories = factories;
			Map<Class<?>, Set<String>> map = new HashMap<Class<?>, Set<String>>();
			for ( Key<?> key : factories.keySet() ) {
				Set<String> names = map.get(key.getType());
				if (names == null) {
					names = new HashSet<String>();
					map.put(key.getType(), names);
				}
				names.add(key.getName());
			}
	
			for ( Entry<Class<?>, Set<String>> entry : map.entrySet() ) {
				entry.setValue(Collections.unmodifiableSet(entry.getValue()));
			}
	
			this.factoryNamesByType = Collections.unmodifiableMap(map);
		}
		....
	}


ContainerImpl内部封装了两个内部实例变量：factories和factoryNamesByType。它们都是Map结构，其中factories是由构造函数传递进入并缓存与内部，而factoryNamesByType则在factories的基础之上做了一个根据名称进行寻址的缓存关系。

factories 的key是什么样子呢？

		class Key<T> {
		
		  final Class<T> type;
		  final String name;
		  final int hashCode;
		
		  private Key(Class<T> type, String name) {
		    if (type == null) {
		      throw new NullPointerException("Type is null.");
		    }
		    if (name == null) {
		      throw new NullPointerException("Name is null.");
		    }
		
		    this.type = type;
		    this.name = name;
		
		    hashCode = type.hashCode() * 31 + name.hashCode();
		  }
		
		  Class<T> getType() {
		    return type;
		  }
		
		  String getName() {
		    return name;
		  }
		
		  @Override
		  public int hashCode() {
		    return hashCode;
		  }
		
		  @Override
		  public boolean equals(Object o) {
		    if (!(o instanceof Key)) {
		      return false;
		    }
		    if (o == this) {
		      return true;
		    }
		    Key other = (Key) o;
		    return name.equals(other.name) && type.equals(other.type);
		  }
		
		  @Override
		  public String toString() {
		    return "[type=" + type.getName() + ", name='" + name + "']";
		  }
		
		  static <T> Key<T> newInstance(Class<T> type, String name) {
		    return new Key<T>(type, name);
		  }
		}
可以看出，Key实际上是由一个二元组构成的对象集合。这个二元组的构成对象 type 和 name 正好可以和之前提到的容器配置元素 bean 节点对应起来。

那么，factories 中的Value呢？

它的具体类型是InternalFactory。其相关源码如下：

		interface InternalFactory<T> extends Serializable {
		
		  /**
		   * Creates an object to be injected.
		   *
		   * @param context of this injection
		   * @return instance to be injected
		   */
		  T create(InternalContext context);
		}

这个泛型接口只有一个create方法，**一旦实现这个接口，就要指定对象的创建机制**。


> 在容器内部缓存的是对象实例构建方法，是Java对象的实例构成方法，而不是实例对象本身，这就让容器看起来像一个工厂的集合，能够根据不同的要求，制造出不同种类的实例对象。

### 注入器

XWork容器内部除了缓存一个对象制造工厂factories用以在运行期能够创建对象实例并返回之外，还需要另一类缓存的帮助，**这类缓存记录对象与对象之间的依赖关系。这一类缓存数据在XWork内部被称之为注入器。**

相关源码如下：

		class ContainerImpl implements Container {

			/**
			 * Field and method injectors.
			 */
			final Map<Class<?>, List<Injector>> injectors =
					new ReferenceCache<Class<?>, List<Injector>>() {
						@Override
						protected List<Injector> create( Class<?> key ) {
							List<Injector> injectors = new ArrayList<Injector>();
							addInjectors(key, injectors);
							return injectors;
						}
					};
			}
			....
		}

这里ReferenceCache是XWork框架中对于缓存的一种简单实现。它提供了一种运行期构建Map内容的机制。可以看一下它的源码：


		public abstract class ReferenceCache<K, V> extends ReferenceMap<K, V> {
		
		  private static final long serialVersionUID = 0;
		
		  transient ConcurrentMap<Object, Future<V>> futures =
		      new ConcurrentHashMap<Object, Future<V>>();
		
		  transient ThreadLocal<Future<V>> localFuture = new ThreadLocal<Future<V>>();
		
		  public ReferenceCache(ReferenceType keyReferenceType,
		      ReferenceType valueReferenceType) {
		    super(keyReferenceType, valueReferenceType);
		  }
		
		  
	      protected abstract V create(K key);
		  ....
		}

可以看出来，它里面维护着一个ConcurrentMap，并且在内部应用ThreadLocal很好地规避了对象操作的多线程问题。当调用 Map 中的接口 get 时，ReferenceCache 首先会检查自己内部是否已经存在相应的缓存对象，如果存在，则直接返回；如果不存在，则会调用其抽象方法create根据 key 的内容产生对象并缓存起来。


> 实际上，ReferenceCache缓存中的Key是每一个Class对象，而缓存的Value则是根据Class对象查找到的所有隶属于Class中的注入器。Key和Value之间的建立过程，通过ReferenceCache中的create方法完成。

**注入器就是XWork容器中实施依赖注入的核心**。

在上面关于ReferenceCache定义中，我们看到所有注入器是在运行期动态添加的，其核心方法create会调用addInjectors扫面所有满足条件的注入器，并加入到其中。相关源码如下：

		/**
		 * Recursively adds injectors for fields and methods from the given class to the given list. Injects parent classes
		 * before sub classes.
		 */
		void addInjectors( Class clazz, List<Injector> injectors ) {
			if (clazz == Object.class) {
				return;
			}
	
			// Add injectors for superclass first.
			addInjectors(clazz.getSuperclass(), injectors);
	
			// 针对所有属性查找满足条件的注入器，并加入injectors 中进行缓存
			addInjectorsForFields(clazz.getDeclaredFields(), false, injectors);
			// 针对所有方法查找满足条件的注入器，并加入injectors 中进行缓存
			addInjectorsForMethods(clazz.getDeclaredMethods(), false, injectors);
		}
	
	
		void addInjectorsForMethods( Method[] methods, boolean statics,
									 List<Injector> injectors ) {
			addInjectorsForMembers(Arrays.asList(methods), statics, injectors,
					new InjectorFactory<Method>() {
						public Injector create( ContainerImpl container, Method method,
												String name ) throws MissingDependencyException {
							return new MethodInjector(container, method, name);
						}
					});
		}
	
		void addInjectorsForFields( Field[] fields, boolean statics,
									List<Injector> injectors ) {
			addInjectorsForMembers(Arrays.asList(fields), statics, injectors,
					new InjectorFactory<Field>() {
						public Injector create( ContainerImpl container, Field field,
												String name ) throws MissingDependencyException {
							return new FieldInjector(container, field, name);
						}
					});
		}
	
		//统一的injector查找方式
		<M extends Member & AnnotatedElement> void addInjectorsForMembers(
				List<M> members, boolean statics, List<Injector> injectors,
				InjectorFactory<M> injectorFactory ) {
			for ( M member : members ) {
				if (isStatic(member) == statics) {
					//查找当前传入的member是否具备 @inject 的Annotation
					Inject inject = member.getAnnotation(Inject.class);
					if (inject != null) {
						try {
							//调用传入的injectorFactory中的create方法创建真正的Injector实例
							injectors.add(injectorFactory.create(this, member, inject.value()));
						} catch ( MissingDependencyException e ) {
							if (inject.required()) {
								throw new DependencyException(e);
							}
						}
					}
				}
			}
		}



## XWork容器实现机理

### 模板方法调用

以下这个方法时所有容器操作的调用基础。

		/**
		 * Looks up thread local context. Creates (and removes) a new context if necessary.
		 */
		<T> T callInContext( ContextualCallable<T> callable ) {
			//从ThreadLocal变量中获取一个执行上下文环境
			Object[] reference = localContext.get();
			if (reference[0] == null) {
				//如果执行上下文环境不存在，创建一个新的环境
				reference[0] = new InternalContext(this);
				try {
					//直接调用回调接口，完成逻辑
					return callable.call((InternalContext) reference[0]);
				} finally {
					// Only remove the context if this call created it.
					reference[0] = null;
					// WW-3768: ThreadLocal was not removed
					localContext.remove();
				}
			} else {
				// 如果上下文环境已经存在，直接调用回调接口完成逻辑
				return callable.call((InternalContext) reference[0]);
			}
		}
	
		interface ContextualCallable<T> {
	
			T call( InternalContext context );
		}

这里使用模板方法作为统一的操作入口的主要原因在于将所有的接口操作进行规范话定义，并将它们纳入一个线程安全的上下文执行环境中。

具体的执行逻辑则被封装到 ContextualCallable 接口的回调实现之中，对于不同接口的实现，它们会拥有不同的实现逻辑，这些由 ContextualCallable 的实现类自行处理。

### 获取对象的实现

	public <T> T getInstance( final Class<T> type, final String name ) {
		return callInContext(new ContextualCallable<T>() {
			public T call( InternalContext context ) {
				return getInstance(type, name, context);
			}
		});
	}

	@SuppressWarnings("unchecked")
	<T> T getInstance( Class<T> type, String name, InternalContext context ) {
		ExternalContext<?> previous = context.getExternalContext();
		Key<T> key = Key.newInstance(type, name);
		context.setExternalContext(ExternalContext.newInstance(null, key, this));
		try {
			InternalFactory o = getFactory(key);
			if (o != null) {
				return getFactory(key).create(context);
			} else {
				return null;
			}
		} finally {
			context.setExternalContext(previous);
		}
	}

在这里可以看到getInstance方法的核心步骤在于根据type和name构成的Key去获取其对应的InternalFactory实现。只需要直接通过其内部缓存的factories对象寻址即可完成，剩下的过程仅仅在于使用InternalFactory所规定的对象构造方法返回兑现给的实例即可。


### 依赖注入实现

基于Field和Method两个最基本的对象元素实现。

		static class FieldInjector implements Injector {
			final Field field;
			final InternalFactory<?> factory;
			final ExternalContext<?> externalContext;
			//构造函数，为实施依赖注入做数据准备
			public FieldInjector( ContainerImpl container, Field field, String name )
					throws MissingDependencyException {
				//缓存field
				this.field = field;
				//检查 field是否可写，并设置field的可写属性
				if (!field.isAccessible()) {
					SecurityManager sm = System.getSecurityManager();
					try {
						if (sm != null) {
							sm.checkPermission(new ReflectPermission("suppressAccessChecks"));
						}
						field.setAccessible(true);
					} catch ( AccessControlException e ) {
						throw new DependencyException("Security manager in use, could not access field: "
								+ field.getDeclaringClass().getName() + "(" + field.getName() + ")", e);
					}
				}
				//根据type和name到container内部工厂中查找相应的对象构造工厂
				Key<?> key = Key.newInstance(field.getType(), name);
				factory = container.getFactory(key);
				//如果没有找到相应的对象构造工厂，则注入失败
				if (factory == null) {
					throw new MissingDependencyException(
							"No mapping found for dependency " + key + " in " + field + ".");
				}
				//为对象构建设置externalContext
				this.externalContext = ExternalContext.newInstance(field, key, container);
			}

			public void inject( InternalContext context, Object o ) {
				ExternalContext<?> previous = context.getExternalContext();
				context.setExternalContext(externalContext);
				try {
					//使用初始化时找到的对象构造工厂创建对象，并使用反射进行注入
					field.set(o, factory.create(context));
				} catch ( IllegalAccessException e ) {
					throw new AssertionError(e);
				} finally {
					context.setExternalContext(previous);
				}
			}
		}


		static class MethodInjector implements Injector {
	
			final Method method;
			final ParameterInjector<?>[] parameterInjectors;
			//构造函数，为实施依赖注入做数据准备
			public MethodInjector( ContainerImpl container, Method method, String name )
					throws MissingDependencyException {
				//检查method是否可写，并设置method的可写属性
				this.method = method;
				if (!method.isAccessible()) {
					SecurityManager sm = System.getSecurityManager();
					try {
						if (sm != null) {
							sm.checkPermission(new ReflectPermission("suppressAccessChecks"));
						}
						method.setAccessible(true);
					} catch ( AccessControlException e ) {
						throw new DependencyException("Security manager in use, could not access method: "
								+ name + "(" + method.getName() + ")", e);
					}
				}
	
				//利用反射方法来查找方法的每一个参数
				Class<?>[] parameterTypes = method.getParameterTypes();
				if (parameterTypes.length == 0) {
					throw new DependencyException(
							method + " has no parameters to inject.");
				}
				//针对每个参数查找对应的 injector
				parameterInjectors = container.getParametersInjectors(
						method, method.getParameterAnnotations(), parameterTypes, name);
			}
	
			public void inject( InternalContext context, Object o ) {
				try {
					//调用反射完成方法的调用，实施依赖注入
					method.invoke(o, getParameters(method, context, parameterInjectors));
				} catch ( Exception e ) {
					throw new RuntimeException(e);
				}
			}
		}

在容器内部通过type和name进行对象构造工厂factory寻址，之后inject调用过程，只不过是调用factory构建对象，并使用Java中**最为普通的反射机制**来完成对象的依赖注入。



