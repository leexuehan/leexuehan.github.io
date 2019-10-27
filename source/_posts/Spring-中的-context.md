---
title: Spring 中的 context
date: 2019-05-02 23:04:38
tags: Spring
categories: 技术
---

本文主要从源码的角度来分析下 Spring 中负责存储 bean 定义的类和接口定义。

## BeanFactory 

首先看下，官方在代码中给出的注释：

> The **root interface for accessing a Spring bean container**.This is the basic client view of a bean container。

是接触所有Spring bean容器的根接口，这个接口被持有大量bean定义的对象实现。

![1557203530675](BeanFactory.png)

从其接口中定义的方法即可看出，其为bean容器的真正含义。

ListableBeanFactory 继承了 BeanFactory，为其提供了枚举所有 beans 的方法：

![1557228601676](ListerableFactory.png)



## ApplicationContext

而 ApplicationContext 类通过对该接口的扩展，提供了为application提供配置的扩展接口。

根据官方给出的注释，其总共功能有以下几点：

1. 对应用的 component 的工厂方法；
2. 一种通用的加载资源文件的方式；
3. 对注册的监听器发布事件的能力；
4. 解析message，支持国家化的能力。

这些能力主要通过继承各种接口获得，其自身接口主要扩展的重要方法主要有一个：

![1557229049849](ApplicationContext.png)

```
@Nullable
ApplicationContext getParent();
```

在 SpringBoot 中，context 有其继承体系：**每个不同的 servlet 都有自己独立的 context，而整个 application 有一个根 context，这个 context 是所有 servlet 所共享的**。

所以此处提供了获得父 context 的能力。



## ConfigurableApplicationContext

全部或者绝大多数 application 的context都会实现或者继承该接口。

扩展了一些配置 context 的方法，但是该接口中的方法**仅仅在应用启动或者关闭时**调用。

![1557229635757](ConfigurableApplicationContext.png)

提供了set/get environment的方法；注册监听器的方法；当然还有最重要的 refresh 方法。

```
/**
 * Load or refresh the persistent representation of the configuration,
 * which might an XML file, properties file, or relational database schema.
 * <p>As this is a startup method, it should destroy already created singletons
 * if it fails, to avoid dangling resources. In other words, after invocation
 * of that method, either all or no singletons at all should be instantiated.
 * @throws BeansException if the bean factory could not be initialized
 * @throws IllegalStateException if already initialized and multiple refresh
 * attempts are not supported
 */
void refresh() throws BeansException, IllegalStateException;
```

从注释中我们可以看到该方法的作用：

> 加载或者刷新持久层的配置资源，比如 xml、property 或者数据库文件。
>
> 因为该方法在应用启动时调用，所以如果失败，则要销毁掉所有已经创建好的单例对象。
>
> 换句话说就是：**一旦该方法被调用，要么创建好所有的单例对象，要么一个单例对象都不会创建**。

### AbstractApplicationContext

对于 ConfigurableApplicationContext，该抽象类提供了大部分的模板实现方法。

这里重点看下 refresh 方法：

```
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
```

该方法比较长，但是注释还是比较详细的，我们可以逐块分解的看：

先看**第5行**（以下提到具体的行数均指的是 refresh 方法中的）中的prepareRefresh：

```
protected void prepareRefresh() {
   // 设置一些容器 active flag 标志位
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   // 初始化一些 environment 中的占位符属性
   initPropertySources();

   // 校验一些标记为 required 的必要属性
   getEnvironment().validateRequiredProperties();

   // 存储“准备刷新”的一些监听器.
   if (this.earlyApplicationListeners == null) {
      this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
   }
   else {
      // Reset local application listeners to pre-refresh state.
      this.applicationListeners.clear();
      this.applicationListeners.addAll(this.earlyApplicationListeners);
   }

   // 存储事件
   this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

继续看**第8行**的代码 obtainFreshBeanFactory 发生了什么：

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   refreshBeanFactory();
   return getBeanFactory();
}
```

这里使用了模板方法，refreshBeanFactory 的实现留给子类，官方给出的注释是：

```
/**
 * 子类必须实现该方法来执行实际的加载工作。
 */
protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;
```

getBeanFactory 方法的实现也留给子类，官方的注释是：

```
/**
 * 子类在这里必须返回它内部的bean factory. 
 * 这个bean factory 应该实现高效的查询工作，这样重复调用时才不会造成性能损失.
 * 注意：
 * 子类在返回 bean factory 之前必须检查 context 是否属于 active 状态；如果处于 close 状态，则不能获取该对象。
 */
@Override
public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```

**第 11 行**，prepareBeanFactory 方法：

对于一些 context 的特征给出了配置，比如加载器、post-processor等。

```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   //设置加载器
   beanFactory.setBeanClassLoader(getClassLoader());
   //设置 Spring EL 解析器
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   //设置用于注册 PropertyEditor 的注册机
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // Configure the bean factory with context callbacks.
   // 设置一些用于 bean 后期处理的 processor
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   // 忽略一些依赖接口
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
   // 注册具有相应自动装配值得依赖类型
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // 注册该监听器用于检测监听器 bean
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // Detect a LoadTimeWeaver and prepare for weaving, if found.
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // 注册默认的 environment beans.
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```

**第15行** postProcessBeanFactory 方法：

```
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```

也是一个模板方法，官方给出的注释是：

context 内部的 bean factory 初始化后，在此处进行一些修改。**此时所有的bean定义都被加载，但是还没有任何一个bean被实例化。**

**第 18 行**invokeBeanFactoryPostProcessors方法：

```
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

   // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
   // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

此方法的作用在于：实例化所有的bean 之前，实例化并调用一些 context 内部持有的 BeanFactoryPostProcessor。

**第21行**registerBeanPostProcessors方法：

```
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

注册所有的 bean 后期处理类。 

**第24行** initMessageSource 方法：

```
protected void initMessageSource() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
      this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
      // Make MessageSource aware of parent MessageSource.
      if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
         HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
         if (hms.getParentMessageSource() == null) {
            // Only set parent context as parent MessageSource if no parent MessageSource
            // registered already.
            hms.setParentMessageSource(getInternalParentMessageSource());
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using MessageSource [" + this.messageSource + "]");
      }
   }
   else {
      // Use empty MessageSource to be able to accept getMessage calls.
      DelegatingMessageSource dms = new DelegatingMessageSource();
      dms.setParentMessageSource(getInternalParentMessageSource());
      this.messageSource = dms;
      beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
      if (logger.isTraceEnabled()) {
         logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
      }
   }
}
```

该方法进行了两方面的处理：

如果此 context中的 beanFactory 有 MessageSource 属性，则直接获取其值，并将其父MessageSource 设置到此context 的父context中；

如果此 context 中的 beanFactory 没有 MessageSource 属性，则直接使用 parent context中的 MessageSource。

**第27行**initApplicationEventMulticaster：

```
protected void initApplicationEventMulticaster() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
      this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
   //去掉日志打印
   else {
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      //去掉日志打印
      }
   }
}
```

此方法初始化 ApplicationEventMulticaster 广播器。

如果本地 context中有该属性，则直接获取值，如果没有则构造一个默认的SimpleApplicationEventMulticaster。

第30行 onRefresh 方法：

```
protected void onRefresh() throws BeansException {
   // For subclasses: do nothing by default.
}
```

又是一个模板方法，留待子类去实现。

主要是增加一些特定context 的 refresh 工作，在一些特殊的 bean 初始化时调用，该调用在任何单例对象实例化之前。

**第33行**registerListeners，检出并注册所有实现了ApplicationListener的监听器，如果此时有 event，则分发之。

```
protected void registerListeners() {
   // Register statically specified listeners first.
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let post-processors apply to them!
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }

   // Publish early application events now that we finally have a multicaster...
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
   this.earlyApplicationEvents = null;
   if (earlyEventsToProcess != null) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

**第36行**finishBeanFactoryInitialization方法：

完成此 context 中所有 factory bean 的初始化工作，**实例化剩下的非惰性加载单例对象**。

```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   // Initialize conversion service for this context.
   if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
         beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
   }

   // Register a default embedded value resolver if no bean post-processor
   // (such as a PropertyPlaceholderConfigurer bean) registered any before:
   // at this point, primarily for resolution in annotation attribute values.
   if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
   }

   // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   // Stop using the temporary ClassLoader for type matching.
   beanFactory.setTempClassLoader(null);

   // Allow for caching all bean definition metadata, not expecting further changes.
   beanFactory.freezeConfiguration();

   // Instantiate all remaining (non-lazy-init) singletons.
   beanFactory.preInstantiateSingletons();
}
```

重点放在下面两行上：

```
// Allow for caching all bean definition metadata, not expecting further changes.
beanFactory.freezeConfiguration();

// Instantiate all remaining (non-lazy-init) singletons.
beanFactory.preInstantiateSingletons();
```

第一步 freeze 所有的 bean定义数据，该定义不能被更改；

第二步 开始真正实例化所有的单例。

**第39行**finishRefresh方法：

```
protected void finishRefresh() {
   // Clear context-level resource caches (such as ASM metadata from scanning).
   clearResourceCaches();

   // Initialize lifecycle processor for this context.
   initLifecycleProcessor();

   // Propagate refresh to lifecycle processor first.
   getLifecycleProcessor().onRefresh();

   // Publish the final event.
   publishEvent(new ContextRefreshedEvent(this));

   // Participate in LiveBeansView MBean, if active.
   LiveBeansView.registerApplicationContext(this);
}
```

此时正式宣告完成 context 的refresh 工作，调用 LifecycleProcessor 的 onRefresh 方法，之后发布 ContextRefreshEvent 事件。

#### GenericApplicationContext

GenericApplicationContext 继承了 AbstractApplicationContext。

官方给出的注释着重强调了一个功能：

> register a variety of bean definitions.

意思即注册各种bean定义。这里的各种包括形式上的，可以 xml，也可以是其他类型。

比如下面的官方给出的例子：

```
GenericApplicationContext ctx = new GenericApplicationContext();
//可以是 xml 形式的
XmlBeanDefinitionReader xmlReader = new XmlBeanDefinitionReader(ctx);
xmlReader.loadBeanDefinitions(new ClassPathResource("applicationContext.xml"));
//也可以是 property 形式的
PropertiesBeanDefinitionReader propReader = new PropertiesBeanDefinitionReader(ctx);
propReader.loadBeanDefinitions(new ClassPathResource("otherBeans.properties"));

//最终只要调用 refresh 方法进行加载和实例化即可
ctx.refresh();

//获取我们在资源文件中注册的bean
MyBean myBean = (MyBean) ctx.getBean("myBean");
```

它的注册功能主要通过继承BeanDefinitionRegistry来扩展。关于Spring中的bean，后面专门介绍。

##### GenericWebApplicationContext

此类继承 GenericApplicationContext，专门为 web 环境提供上下文。

## WebApplicationContext

![1557271387505](WebApplicationContextRelation.png)



从上图可以看到 WebApplicationContext 和 ConfigurableApplicationContext 级别相同，都是 ApplicationContext 的直接子类，其提供的功能不相同。

前面介绍 ConfigurableApplicationContext ，其核心功能就是管理 Spring 中的 beans，包括定义、实例化、启动环境、资源文件加载解析等等，为我们提供了一种管理 context 的接口；

而WebApplicationContext ，顾名思义就是专门为 web application 提供配置的。

![1557271831768](WebApplicationContext.png))

从上图中可以看到，WebApplicationContext 只扩展了一个方法：

```
/**
 * Return the standard Servlet API ServletContext for this application.
 */
@Nullable
ServletContext getServletContext();
```

ServletContext 提供了一组方法用于一个Servlet与其Servlet 容器交流，关于 Servlet 后面会继续探究。

而从其成员变量属性来看，都是跟请求与响应相关。所以这是一个专门为 web application提供的 context。

同上面的 ConfigurableApplicationContext 一样，WebApplicationContext  也具有继承属性。

**每一个 application 中只有一个 root context，但是每一个 servlet 都具有自己的 child context。**

## 总结

此文对 Spring 中的几个重要的 context 进行了分析，尤其是对 AbstractApplicationContext 的主要脉络进行了梳理。



