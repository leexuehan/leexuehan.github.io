---
title: Struts 源码学习之初始化
tags:
  - Struts2
date: 2015-07-30 22:53:23
categories: 技术
---

#### 前面的话

最近在看陆舟的《Struts2技术内幕》，写的文采斐然（技术书籍能写成这样，印象中只有侯捷了），但是怎奈水平较低，有些地方还是不太明白。于是就想按照自己的思路去看源码，一步一步跟踪调试，顺便参考书上的东西加点注释之类的，这样下来，感觉比只看书明白多了，于是就有了这篇和下篇文章。这一篇从Struts2的初始化开始讲起。

用自己的话理解Struts2框架就是：接收 http 请求，然后经过服务器过滤器处理，通过 Dispatcher 从Struts.xml找到相应的Action，执行逻辑，根据结果，返回视图资源供浏览器解析。

当然上面说得比较粗糙，下面的是官方的图示，可以让我们对整个流程一目了然。

![](http://i.imgur.com/hDFwrD5.png)

在后面会从源码一步一步顺着思路开始介绍。本篇文章先从 Struts2 初始化主线开始介绍。


Struts2最重要的一个过滤器：

![](http://i.imgur.com/rFiry1D.png)

这个过滤器就是在 web.xml 配置的过滤器。

![](http://i.imgur.com/qLEkKSv.png)

## Strut2 初始化主线的入口

![](http://i.imgur.com/3fkVj3O.png)

### Dispatcher

> Dispatcher 是 Struts2 与 XWork 的分界点，也是将MVC实现与Web容器隔离的分界点。

Dispatcher 在之后的操作中占有相当重要的地位，在后面会专门提到它的作用。

这里从其属性和方法两方面做一个简要的介绍。

**属性**：

![](http://i.imgur.com/sxS2zI1.png)

**方法**：

![](http://i.imgur.com/7aIG6iI.png)


![](http://i.imgur.com/xmQwtzZ.png)


在过滤器的 init 方法中可以看到在 InitOperations 类中的 initDispatcher 分两步，一步是创建 Dispatcher，第二步就是对其进行初始化，从这里开始带动全局的初始化。

![](http://i.imgur.com/Z4fo9rm.png)

createDispatcher() 方法如下图所示

![](http://i.imgur.com/gVpq1WQ.png)

initDispatcher() 方法如下图所示

![](http://i.imgur.com/4oYSGO0.png)

#### initDispatcher() 方法：

**第一步**，创建 ConfigurationManager。

![](http://i.imgur.com/iKtWttP.png)

可以看到在ConfigurationManager 中有一个构造函数，此构造函数的参数仅仅是一个String型的变量。这个String型的变量就是配置元素中的<bean>节点中的name属性。

这个 ConfigurationManager 有什么用处呢？可以看缩略图中的方法，就能猜到一二：

> 如同 ActionProxy 在整个XWork中起到的作用，它是配置元素的操作接口。它的内部封装了容器加载器和事件映射加载器。

这是在 ConfigurationManager 中的 getConfiguration 的实现：

		public synchronized Configuration getConfiguration() {
	        if (configuration == null) {
	            setConfiguration(createConfiguration(defaultFrameworkBeanName));
	            try {
	                configuration.reloadContainer(getContainerProviders());
	            } catch (ConfigurationException e) {
	                setConfiguration(null);
	                throw new ConfigurationException("Unable to load configuration.", e);
	            }
	        } else {
	            conditionalReload();
	        }
	
	        return configuration;
	    }

可以看到 Configuration 实例与 ConfigurationProvider 被封装在了 ConfigurationManager 内部。


**第二步**，初始化各种配置加载器

[1]初始化 Struts2 默认的 Properties 文件，该文件是 org/apache/struts2/default 目录下的 default.properties

[2]初始化 XML 文件配置加载器

[3]初始化Properties文件配置加载器

[5]用户自定义的配置加载器

[6]初始化由 web.xml 传入的运行参数

[7]初始化默认容器内置对象

配置元素加载器有一个统一的接口 ConfigurationProvider

![](http://i.imgur.com/Hm5OBVZ.png)

以 [3] 为例进行分析，其代码如下：

	private void init_DefaultProperties() {
        configurationManager.addContainerProvider(new DefaultPropertiesProvider());
    }

在 ConfigurationManager 里有一个方法 addContainerProvider：

	public void addContainerProvider(ContainerProvider provider) {
        if (!containerProviders.contains(provider)) {
            containerProviders.add(provider);
            providersChanged = true;
        }
    }


> 可以看出配置加载接口的各种实现类架起了各种配置的表现形式到Java世界的桥梁。

配置的加载方式起着翻译的作用，通过翻译，架构可以透明地看待所有的配置形式。

ContainerProvider 是容器的一个加载接口，用于加载容器配置元素。下面是这个接口的定义：

	public interface ContainerProvider {
	
	    /**
	     * Called before removed from the configuration manager
	     */
	    public void destroy();
	    
	    /**
	     * Initializes with the configuration
	     * @param configuration The configuration
	     * @throws ConfigurationException If anything goes wrong
	     */
	    public void init(Configuration configuration) throws ConfigurationException;
	    
	    /**
	     * Tells whether the ContainerProvider should reload its configuration
	     *
	     * @return <tt>true</tt>, whether the ContainerProvider should reload its configuration, <tt>false</tt>otherwise.
	     */
	    public boolean needsReload();
	    
	    /**
	     * Registers beans and properties for the Container
	     * 
	     * @param builder The builder to register beans with
	     * @param props The properties to register constants with
	     * @throws ConfigurationException If anything goes wrong
	     */
	    public void register(ContainerBuilder builder, LocatableProperties props) throws ConfigurationException;
	    
	}

从注释中可以看到，register方法用于注册所有的beans和Properties。

这里出现的 ContainerBuilder 是什么呢？

下面是其结构图

![](http://i.imgur.com/1EUU7Af.png)

可以看到主要有两类方法：一类用于搜集参数；一类用于构造对象。

先创建一个 ContainerBuilder 实例，接着调用 factory 方法和 constant 方法搜集各种配置元素，最后调用 create 方法把容器创建出来。

以上介绍了容器构造器，用于加载 web.xml 中的框架级别的配置元素；还有一类构造器叫事件映射构造器用于加载 struts.xml中的事件处理元素。

![](http://i.imgur.com/2tSEwnk.png)

上图是PackageConfig的接口方法。注意到最底下有一个builder内部静态类。这个类的作用是构建此对象。

先构造一个新的构造器实例，接着往构造器中设置参数，最后调用 build 方法完成对象创建。

builder 内部类的架构

![](http://i.imgur.com/NCqJvNW.png) 


**第三步**，初始化容器

就如图中所标注的那样，先创建容器对象，后对其进行依赖注入。

从第一张图可以看到，容器的初始化是通过 init_PreloadConfiguration() 这个方法完成的。

		private Container init_PreloadConfiguration() {
	        Container container = getContainer();
	
	        boolean reloadi18n = Boolean.valueOf(container.getInstance(String.class, StrutsConstants.STRUTS_I18N_RELOAD));
	        LocalizedTextUtil.setReloadBundles(reloadi18n);
	
	        boolean devMode = Boolean.valueOf(container.getInstance(String.class, StrutsConstants.STRUTS_DEVMODE));
	        LocalizedTextUtil.setDevMode(devMode);
	
	        return container;
	    }

		public Container getContainer() {
	        if (ContainerHolder.get() != null) {
	            return ContainerHolder.get();
	        }
	        ConfigurationManager mgr = getConfigurationManager();
	        if (mgr == null) {
	            throw new IllegalStateException("The configuration manager shouldn't be null");
	        } else {
	            Configuration config = mgr.getConfiguration();
	            if (config == null) {
	                throw new IllegalStateException("Unable to load configuration");
	            } else {
	                Container container = config.getContainer();
	                ContainerHolder.store(container);
	                return container;
	            }
	        }
	    }


在一个单独的文件里，ContainerHolder是这样定义的：

		class ContainerHolder {
		
		    private static ThreadLocal<Container> instance = new ThreadLocal<Container>();
		
		    public static void store(Container instance) {
		        ContainerHolder.instance.set(instance);
		    }
		
		    public static Container get() {
		        return ContainerHolder.instance.get();
		    }
		
		    public static void clear() {
		        ContainerHolder.instance.remove();
		    }
		
		}

可以看到这是将Container的一个引用存放到一个ThreadLocal 引用中。

另外，要注意到在 mgr.getConfiguration() 方法中有一个 reloadContainer 方法，这个方法完成容器的初始化。

		public synchronized List<PackageProvider> reloadContainer(List<ContainerProvider> providers) throws ConfigurationException {
				//内部缓存对象的清理工作
		        packageContexts.clear();
		        loadedFileNames.clear();

		        List<PackageProvider> packageProviders = new ArrayList<PackageProvider>();
				//创建 ContainerProperties 对象，这将是初始化所有 Properties 文件中相关运行参数
				//的主要操作对象
		        ContainerProperties props = new ContainerProperties();
				//创建 Container 的构造器
		        ContainerBuilder builder = new ContainerBuilder();
				//创建启动容器
		        Container bootstrap = createBootstrapContainer(providers);
				
		        for (final ContainerProvider containerProvider : providers)
		        {
		            bootstrap.inject(containerProvider);
					//初始化 containerProvider
		            containerProvider.init(this);
					//调用 register 方法，完成所有 Container 中的对象注册
		            containerProvider.register(builder, props);
		        }
		        props.setConstants(builder);
		
		        builder.factory(Configuration.class, new Factory<Configuration>() {
		            public Configuration create(Context context) throws Exception {
		                return DefaultConfiguration.this;
		            }
		        });
		
		        ActionContext oldContext = ActionContext.getContext();
		        try {
		            // Set the bootstrap container for the purposes of factory creation
		
		            setContext(bootstrap);
		            container = builder.create(false);
		            setContext(container);
		            objectFactory = container.getInstance(ObjectFactory.class);
		
		            // Process the configuration providers first
		            for (final ContainerProvider containerProvider : providers)
		            {
		                if (containerProvider instanceof PackageProvider) {
		                    container.inject(containerProvider);
		                    ((PackageProvider)containerProvider).loadPackages();
		                    packageProviders.add((PackageProvider)containerProvider);
		                }
		            }
		
		            // Then process any package providers from the plugins
		            Set<String> packageProviderNames = container.getInstanceNames(PackageProvider.class);
		            for (String name : packageProviderNames) {
		                PackageProvider provider = container.getInstance(PackageProvider.class, name);
		                provider.init(this);
		                provider.loadPackages();
		                packageProviders.add(provider);
		            }
		
		            rebuildRuntimeConfiguration();
		        } finally {
		            if (oldContext == null) {
		                ActionContext.setContext(null);
		            }
		        }
		        return packageProviders;
		    }

**第四步**，进行额外的初始化


#### PrepareOperations 结构

![](http://i.imgur.com/DkYvBhV.png)

可以看到图中用红色框标出来的方法，其参数都是 HttpServletResponse 或者 HttpServletRequest。注意到其中有一个方法： createActionContext().

![](http://i.imgur.com/rLtsKAM.png)

另外，注意其中的属性，我们看到有一个 dispatcher 的属性构成了对于 Dispatcher 的引用。

#### ExecuteOperations 结构

![](http://i.imgur.com/fyIXFUc.png)

至此，Struts2 的初始化主线已经完成。
