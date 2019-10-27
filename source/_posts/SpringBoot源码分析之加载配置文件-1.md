---
title: SpringBoot源码分析之加载配置文件
date: 2018-07-06 18:18:35
tags: SpringBoot
categories: 技术
---

上一篇从使用角度介绍了如何在 springboot 启动时对 yml 文件进行配置，这一篇尝试从源码角度去探讨其加载配置文件的机制。<!--more-->

我们回到 springboot 初始启动时的 run 方法，为了突出主干逻辑，对代码做了一些精简：

```
public ConfigurableApplicationContext run(String... args) {
  //省略其他代码
  ...

   Collection<SpringApplicationRunListener> runListeners = getRunListeners(args);
   for (SpringApplicationRunListener runListener : runListeners) {
      runListener.started();
   }

   try {
      // Create and configure the environment
      ConfigurableEnvironment environment = getOrCreateEnvironment();
      configureEnvironment(environment, args);
      //广播 environment 已准备事件到各个监听器
      for (SpringApplicationRunListener runListener : runListeners) {
         runListener.environmentPrepared(environment);
      }

      // Create, load, refresh and run the ApplicationContext
      context = createApplicationContext();
      if (this.registerShutdownHook) {
         try {
            context.registerShutdownHook();
         }
         catch (AccessControlException ex) {
            // Not allowed in some environments.
         }
      }
      context.setEnvironment(environment);
      postProcessApplicationContext(context);
      applyInitializers(context);
      for (SpringApplicationRunListener runListener : runListeners) {
         runListener.contextPrepared(context);
      }

      // Load the sources
      Set<Object> sources = getSources();
      Assert.notEmpty(sources, "Sources must not be empty");
      load(context, sources.toArray(new Object[sources.size()]));
      for (SpringApplicationRunListener runListener : runListeners) {
         runListener.contextLoaded(context);
      }

      // Refresh the context
      refresh(context);
      afterRefresh(context, args);
      for (SpringApplicationRunListener runListener : runListeners) {
         runListener.finished(context, null);
      }
      return context;
   }
   catch (Throwable ex) {
      //省略
      ....
   }
```

对于 spring run listener 如何通过事件类型找到监听器，前文已有叙述。此处从一个监听器开始说起。

## ConfigFileApplicationListener

正是这个监听器监听到 environmentPrepared event后，开始加载配置文件逻辑。

```
@Override
public void onApplicationEvent(ApplicationEvent event) {
	//判断类型，执行逻辑
   if (event instanceof ApplicationEnvironmentPreparedEvent) {
      onApplicationEnvironmentPreparedEvent(
            (ApplicationEnvironmentPreparedEvent) event);
   }
   if (event instanceof ApplicationPreparedEvent) {
      onApplicationPreparedEvent((ApplicationPreparedEvent) event);
   }
}
```

跟着代码走，进入下面的方法：

```
private void onApplicationEnvironmentPreparedEvent(
      ConfigurableEnvironment environment, SpringApplication application) {
   //加载所有的配置属性到 environment
   addPropertySources(environment, application.getResourceLoader());
   //将 environment 绑定到 springApplication
   bindToSpringApplication(environment, application);
}
```

### 加载属性

```
protected void addPropertySources(ConfigurableEnvironment environment,
      ResourceLoader resourceLoader) {
   RandomValuePropertySource.addToEnvironment(environment);
   try {
   	  //核心方法，使用 resourceLoader 从 environment 中加载属性
      new Loader(environment, resourceLoader).load();
   } catch (IOException ex) {
      throw new IllegalStateException("Unable to load configuration files", ex);
   }
}
```

load 方法代码如下：

```
public void load() throws IOException {
	//初始化属性加载器，用于后面的属性加载工作
   this.propertiesLoader = new PropertySourcesLoader();
   this.activatedProfiles = false;
   //初始化 profiles 集合，包装成为一个 LIFO（后进先出）队列，为了实现后面覆盖前面的特性
   this.profiles = Collections.asLifoQueue(new LinkedList<String>());
   //初始化 active profile 集合
   Set<String> initialActiveProfiles = initializeActiveProfiles();
   this.profiles.addAll(getUnprocessedActiveProfiles(initialActiveProfiles));
   this.profiles.add(null);

	//取出来所有的profile，依次load，如果后次加载的属性会覆盖上次加载的属性值
   while (!this.profiles.isEmpty()) {
      String profile = this.profiles.poll();
      for (String location : getSearchLocations()) {
         if (!location.endsWith("/")) {
            // location is a filename already, so don't search for more
            // filenames
            load(location, null, profile);
         }
         else {
            for (String name : getSearchNames()) {
               load(location, name, profile);
            }
         }
      }
   }

	//绑定属性到 environment 中
   addConfigurationProperties(this.propertiesLoader.getPropertySources());
}
```

#### profile

profile 用于区别不同环境下的配置，比如现在有两套环境：生产环境和测试环境，如果有两个配置文件对于同一个属性，进行了不同的配置（比如服务器port），那我只需要针对两套环境写两个配置文件，只需要更改 profile 的值即可让应用自动选择加载哪个配置文件。

这里通过使用一个 *Collections.asLifoQueue(new LinkedList<String>());*方法构建了一个 LIFO（后进先出）的队列，类似栈的性质。目的就是为了后面的 profile 具有更高的优先级。比如指定了 

> spring.profiles.active=dev,hsqldb

则对于同样一个属性的配置，hsqldb 的属性值会覆盖 dev 的属性值。

对于 profile 的配置分三步进行：

```load
   Set<String> initialActiveProfiles = initializeActiveProfiles();
   this.profiles.addAll(getUnprocessedActiveProfiles(initialActiveProfiles));
   this.profiles.add(null)
  
```

#### 取到所有 active 的 profile

第一步：**初始化 active profile 的集合**。

```
private Set<String> initializeActiveProfiles() {
	//ACTIVE_PROFILES_PROPERTY = "spring.profiles.active"
	//如果环境中没有该属性，则返回一个空集
   if (!this.environment.containsProperty(ACTIVE_PROFILES_PROPERTY)) {
      return Collections.emptySet();
   }
   //Property source（比如系统属性）设置的 profile 优先级高于配置文件中的配置
   Set<String> activeProfiles = getProfilesForValue(
         this.environment.getProperty(ACTIVE_PROFILES_PROPERTY));
   maybeActivateProfiles(activeProfiles);
   return activeProfiles;
}
```

我们进去 *getProfilesForValue* 往方法里面一直走：

```
List<String> list = Arrays
      .asList(StringUtils.commaDelimitedListToStringArray(value != null
            ? this.environment.resolvePlaceholders(value) : fallback));
Collections.reverse(list);
return new LinkedHashSet<String>(list);
```

此段代码总共做了三件事：

1. 将环境中的 profile 配置属性使用符号进行分割；
2. 使用集合工具类将之反转（为了**使后面的 profile 优先级高于前面的**）
3. 加入一个 hashset（为了去重） 中返回，其中 LinkedHashSet 底层使用一个 LinkedHashMap来保存数据，故能使得**插入的数据保持顺序**。

获取到 profile集合之后，按顺序入队。

```
private void addProfiles(Set<String> profiles) {
   for (String profile : profiles) {
      this.profiles.add(profile);
      if (!this.environment.acceptsProfiles(profile)) {
         // If it's already accepted we assume the order was set
         // intentionally
         prependProfile(this.environment, profile);
      }
   }
}
```

第二步：**拿到命令行配置的 profile**

在初始化 profile 完毕之后，又通过下面的一行代码对一些 profile 进行了合并。

```
this.profiles.addAll(getUnprocessedActiveProfiles(initialActiveProfiles));
```

我们可以继续看下并入的profile都是什么？

```
private List<String> getUnprocessedActiveProfiles(
      Set<String> initialActiveProfiles) {
   List<String> unprocessedActiveProfiles = new ArrayList<String>();
   for (String profile : this.environment.getActiveProfiles()) {
      if (!initialActiveProfiles.contains(profile)) {
         unprocessedActiveProfiles.add(profile);
      }
   }
   // 继续反转，逻辑同上
   Collections.reverse(unprocessedActiveProfiles);
   return unprocessedActiveProfiles;
}
```

这里处理了一些通过其他方式设置的 profile，由于前面通过*ACTIVE_PROFILES_PROPERTY*属性设置的profile拥有更高的优先级，所以此处继续从后面入队。

第三步：**添加默认的profile**

```
this.profiles.add(null);
```

默认的 profile 为null，优先级最低，只要在 queue 前面有元素即可被覆盖，所以此时加入 null，是为了后续处理统一。

**到此，经过三步操作，我们已经按照优先级顺序拿到了所有的 active 的 profile。**

接下来就是出队，按照优先级顺序依次加载各个 profile 相应的属性，如果属性相同，则后面出队的覆盖前面的，与优先级特性相对应。



### 依次加载 profile 属性

```
while (!this.profiles.isEmpty()) {
   String profile = this.profiles.poll();
   for (String location : getSearchLocations()) {
      if (!location.endsWith("/")) {
         // location is a filename already, so don't search for more
         // filenames
         load(location, null, profile);
      }
      else {
         for (String name : getSearchNames()) {
            load(location, name, profile);
         }
      }
   }
}
```

上面就是依次加载的主要逻辑。

从上面的代码中可以看到，依次出队取到 profile 属性后，通过一个 getSearchLocations 方法拿到所有应该加载 property 的位置，依次进行加载。

```
private Set<String> getSearchLocations() {
   Set<String> locations = new LinkedHashSet<String>();
   // CONFIG_LOCATION_PROPERTY = "spring.config.location"
   // 用户配置优先，所以先处理配置
   if (this.environment.containsProperty(CONFIG_LOCATION_PROPERTY)) {
      for (String path : asResolvedSet(
            this.environment.getProperty(CONFIG_LOCATION_PROPERTY), null)) {
         if (!path.contains("$")) {
            path = StringUtils.cleanPath(path);
            if (!ResourceUtils.isUrl(path)) {
               path = ResourceUtils.FILE_URL_PREFIX + path;
            }
         }
         locations.add(path);
      }
   }
   // 加载默认配置
   // DEFAULT_SEARCH_LOCATIONS = "classpath:/,classpath:/config/,file:./,file:./config/"
   locations.addAll(
         asResolvedSet(ConfigFileApplicationListener.this.searchLocations,
               DEFAULT_SEARCH_LOCATIONS));
   return locations;
}
```

从上面代码中可以看到，**如果用户通过 spring.config.location 参数对加载文件的路径进行了配置，则会首先将该路径加入到 set 集合中（同样也可以保证顺序）**。之后才会加载默认的路径，而默认路径即如前文所介绍的。

在拿到路径后，先判断是否是目录，如果 location 不以 “/”结尾，则认为已经是一个文件名，直接加载；反之，则要在该路径下搜索到所有满足条件的配置文件。

```
// Search for a file with the given name
for (String ext : this.propertiesLoader.getAllFileExtensions()) {
   if (profile != null) {
      // Try the profile specific file
      loadIntoGroup(group, location + name + "-" + profile + "." + ext,
            null);
      loadIntoGroup(group, location + name + "-" + profile + "." + ext,
            profile);
   }
   loadIntoGroup(group, location + name + "." + ext, profile);
}
```

上面代码对符合目录下所有的合法的配置文件进行了搜索。

合法文件名的样式如下：

> location + name + "-" + profile + "." + ext
>
> 比如 application-dev.yml

for循环里面对于所有支持的文件扩展名进行了遍历，我们可以看看支持的文件扩展名有哪些：

```
public String[] getFileExtensions() {
   return new String[] { "yml", "yaml" };
}
```

这是负责加载 yml 文件的 ***YamlPropertySourceLoader*** 返回的符合条件的文件扩展名。

```
public String[] getFileExtensions() {
   return new String[] { "properties", "xml" };
}
```

这是负责加载 property 文件的 ***PropertiesPropertySourceLoader*** 返回的符合条件的文件扩展名。

这两个加载器都是 springboot 自带的加载器，当然也可以自定义加载属性类，即可加载指定扩展名的文件。

接下来重点看下实际进行加载的 loadIntoGroup 方法：

这里 group 的名字为“profile=dev ”样式。

```
private PropertySource<?> loadIntoGroup(String identifier, String location,
      String profile) throws IOException {
   Resource resource = this.resourceLoader.getResource(location);
   PropertySource<?> propertySource = null;
   if (resource != null) {
      String name = "applicationConfig: [" + location + "]";
      String group = "applicationConfig: [" + identifier + "]";
      propertySource = this.propertiesLoader.load(resource, group, name,
            profile);
      if (propertySource != null) {
         handleProfileProperties(propertySource);
      }
   }
   //省略日志打印
   ....
   return propertySource;
}
```

由 yml 加载器或者 property 加载器对文件加载后得到 propertySource（里面存放各种key-value变量），具体第三方加载器的加载细节此处暂时不讨论。可以看下 handleProfileProperties 方法的处理逻辑：

```
private void handleProfileProperties(PropertySource<?> propertySource) {
   Set<String> activeProfiles = getProfilesForValue(
         propertySource.getProperty(ACTIVE_PROFILES_PROPERTY));
   maybeActivateProfiles(activeProfiles);
   // INCLUDE_PROFILES_PROPERTY = "spring.profiles.include"
   Set<String> includeProfiles = getProfilesForValue(
         propertySource.getProperty(INCLUDE_PROFILES_PROPERTY));
   addProfiles(includeProfiles);
}
```

这里又对 spring.profiles.include 指定的 profile 参数进行了处理。



### 绑定属性到 environment 中

```
bindToSpringApplication(environment, application);
```

对该方法进一步跟进，可以发现最终进入下面方法：

```
private void doBindPropertiesToTarget() throws BindException {
   RelaxedDataBinder dataBinder = (this.targetName != null
         ? new RelaxedDataBinder(this.target, this.targetName)
         : new RelaxedDataBinder(this.target));
   //验证
   if (this.validator != null) {
      dataBinder.setValidator(this.validator);
   }
   if (this.conversionService != null) {
      dataBinder.setConversionService(this.conversionService);
   }
   //设置各种属性
   dataBinder.setIgnoreNestedProperties(this.ignoreNestedProperties);
   dataBinder.setIgnoreInvalidFields(this.ignoreInvalidFields);
   dataBinder.setIgnoreUnknownFields(this.ignoreUnknownFields);
   customizeBinder(dataBinder);
   Set<String> names = getNames();
   PropertyValues propertyValues = getPropertyValues(names);
   //执行真正的绑定
   dataBinder.bind(propertyValues);
   if (this.validator != null) {
      validate(dataBinder);
   }
}
```

dataBinder.bind(propertyValues) 方法执行真正的数据绑定工作。

```
protected void doBind(MutablePropertyValues mpvs) {
   checkAllowedFields(mpvs);
   checkRequiredFields(mpvs);
   applyPropertyValues(mpvs);
}
```

```
protected void applyPropertyValues(MutablePropertyValues mpvs) {
   try {
      // 执行绑定
      getPropertyAccessor().setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields());
   }
   //异常处理
   ...
}
```



## 总结

此文对 springboot 加载配置文件的流程进行了一个粗线条的分析，很多细节还没有分析到位，但是主干逻辑已然分析清楚了，不足之处，留待后面完善了。