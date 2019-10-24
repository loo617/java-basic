# spring IOC
## 引言
spring容器的启动依赖于spring-context这个jar包。spring-context会自动将spring-core,spring-beans,spring-aop,spring-expression这几个基础jar包带进来。
## BeanFactory
- BeanFactory
BeanFactory是最底层的接口，它提供了容器中实例化，拿对象等基本方法。
- ApplicationContext
ApplicationContext继承了BeanFactory，它是一个更高级的spring容器，实现了很多有用的方法。例如： 国际化（MessageSource），事件机制（ApplicationEventPublisher），访问资源，如URL和文件（ResourceLoader）等。
- AbstractApplicationContext
AbstractApplicationContext是一个抽象类使用了模板方法方式，它基本地实现了ApplicationContext的方法，同时提供了一些抽象方法供子类实现。
- DefaultListableBeanFactory
DefaultListableBeanFactory是spring注册以及加载bean的实现。默认实现了ListableBeanFactory和BeanDefinitionRegistry接口，基于bean definition对象，是一个成熟的bean factroy。最典型的应用是：在访问bean前，先注册所有的definition（可能从bean definition配置文件中）。使用预先建立的bean定义元数据对象，从本地的bean definition表中查询bean definition因而将不会花费太多成本。DefaultListableBeanFactory既可以作为一个单独的beanFactory，也可以作为自定义beanFactory的父类。
- ClassPathXmlApplicationContext
ClassPathXmlApplicationContext是读取src目录下的配置文件
```java
ApplicationContext app = new ClassPathXmlApplicationContext("applicationContext.xml"); 
```
- ClassPathXmlApplicationContext和FileSystemXmlApplicationContext的区别
classpath:前缀是不需要的,默认就是指项目的classpath路径下面;如果要使用绝对路径,需要加上file:前缀表示这是绝对路径;
对于FileSystemXmlApplicationContext:默认表示的是两种:1.没有盘符的是项目工作路径,即项目的根目录;2.有盘符表示的是文件绝对路径.如果要使用classpath路径,需要前缀classpath:
![ClassPathXmlApplicationContext和FileSystemXmlApplicationContext的继承结构](/pic/douya20191022174842.png)
- AnnotationConfigApplicationContext
使用AnnotationConfigApplicationContext可以实现基于Java的配置类加载Spring的应用上下文.避免使用application.xml进行配置
![AnnotationConfigApplicationContext的继承结构](/pic/douya20191022174320.png)
## BeanDefinition
BeanDefinition中保存了我们的Bean信息，比如这个Bean指向的是哪个类、是否是单例的、是否懒加载、这个Bean依赖了哪些Bean等等。
```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
 
   // 我们可以看到，默认只提供 sington 和 prototype 两种，
   // 很多读者都知道还有 request, session, globalSession, application, websocket 这几种，
   // 不过，它们属于基于 web 的扩展。
   String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
   String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
 
   // 比较不重要，直接跳过吧
   int ROLE_APPLICATION = 0;
   int ROLE_SUPPORT = 1;
   int ROLE_INFRASTRUCTURE = 2;
 
   // 设置父 Bean，这里涉及到 bean 继承，不是 java 继承。请参见附录介绍
   void setParentName(String parentName);
 
   // 获取父 Bean
   String getParentName();
 
   // 设置 Bean 的类名称
   void setBeanClassName(String beanClassName);
 
   // 获取 Bean 的类名称
   String getBeanClassName();
 
 
   // 设置 bean 的 scope
   void setScope(String scope);
 
   String getScope();
 
   // 设置是否懒加载
   void setLazyInit(boolean lazyInit);
 
   boolean isLazyInit();
 
   // 设置该 Bean 依赖的所有的 Bean，注意，这里的依赖不是指属性依赖(如 @Autowire 标记的)，
   // 是 depends-on="" 属性设置的值。
   void setDependsOn(String... dependsOn);
 
   // 返回该 Bean 的所有依赖
   String[] getDependsOn();
 
   // 设置该 Bean 是否可以注入到其他 Bean 中，只对根据类型注入有效，
   // 如果根据名称注入，即使这边设置了 false，也是可以的
   void setAutowireCandidate(boolean autowireCandidate);
 
   // 该 Bean 是否可以注入到其他 Bean 中
   boolean isAutowireCandidate();
 
   // 主要的。同一接口的多个实现，如果不指定名字的话，Spring 会优先选择设置 primary 为 true 的 bean
   void setPrimary(boolean primary);
 
   // 是否是 primary 的
   boolean isPrimary();
 
   // 如果该 Bean 采用工厂方法生成，指定工厂名称。对工厂不熟悉的读者，请参加附录
   void setFactoryBeanName(String factoryBeanName);
   // 获取工厂名称
   String getFactoryBeanName();
   // 指定工厂类中的 工厂方法名称
   void setFactoryMethodName(String factoryMethodName);
   // 获取工厂类中的 工厂方法名称
   String getFactoryMethodName();
 
   // 构造器参数
   ConstructorArgumentValues getConstructorArgumentValues();
 
   // Bean 中的属性值，后面给 bean 注入属性值的时候会说到
   MutablePropertyValues getPropertyValues();
 
   // 是否 singleton
   boolean isSingleton();
 
   // 是否 prototype
   boolean isPrototype();
 
   // 如果这个 Bean 原生是抽象类，那么不能实例化
   boolean isAbstract();
 
   int getRole();
   String getDescription();
   String getResourceDescription();
   BeanDefinition getOriginatingBeanDefinition();
}
```
## 启动过程
```java
@Override
	public void refresh() throws BeansException, IllegalStateException {
		//加锁,同步，防止重复的加载和销毁容器
		synchronized (this.startupShutdownMonitor) {
			//准备工作,记录下容器的启动时间,标记已启动状态
			prepareRefresh();

			// 注册Bean到BeanFactory中,生产beanName->beanDefintion的map
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 设置BeanFactory的类加载器 添加几个BeanPostProcessor 手动注册几个特殊的bean
			prepareBeanFactory(beanFactory);

			try {
				// 此时bean还没有初始化 添加一些BeanFactoryPostProcessor的实现类
				postProcessBeanFactory(beanFactory);

				// 调用BeanFactoryPostProcessor实现类的postProcessBeanFactory(factory)方法
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册BeanPostProcessor的实现类 该类和BeanFactoryPostProcessor的区别在于前者的方法在初始化前后执行
				registerBeanPostProcessors(beanFactory);

				// 初始化当前ApplicationContext的MessageSource（国际化）
				initMessageSource();

				// 初始化当前ApplicationContext的事件广播器
				initApplicationEventMulticaster();

				// 钩子方法 具体的子类会在这里初始化一些特殊的Bean(在初始化单例之前)
				onRefresh();

				// 注册事件监听器
				registerListeners();

				// 初始化所有单例 懒加载除外
				finishBeanFactoryInitialization(beanFactory);

				// 最后广播事件 ApplicationContext初始化完成
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
	}
```
### obtainFreshBeanFactory
注意，这个方法是全文最重要的部分之一，这里将会初始化BeanFactory、加载Bean、注册Bean等等。
当然，这步结束后，Bean并没有完成初始化。这里指的是Bean实例并未在这一步生成。
- AbstractApplicationContext.java
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   // 关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
   refreshBeanFactory();

   // 返回刚刚创建的 BeanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
   return beanFactory;
}
```
- GenericApplicationContext.refreshBeanFactory()

这里使用AbstractApplicationContext. refreshBeanFactory()在不同实现容器中有点区别，如果是以xml方式配置bean，会使用AbstractRefreshableApplicationContext容器中的实现，该容器中实现xml配置文件定位，并通过BeanDefinition载入和解析xml配置文件。而如果是注解的方式，则并没有解析项目包下的注解，而是通过在refresh()方法中执行ConfigurationClassPostProcessor后置处理器完成对bean的加载.
```java
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
   if (!this.refreshed.compareAndSet(false, true)) {
      throw new IllegalStateException(
            "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
   }
   this.beanFactory.setSerializationId(getId());
}
```
### prepareBeanFactory
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // Tell the internal bean factory to use the context's class loader etc.
   beanFactory.setBeanClassLoader(getClassLoader());  //设置类加载器
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader())); //bean表达式解析器
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // Configure the bean factory with context callbacks.
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));  //添加一个BeanPostProcessor实现ApplicationContextAwareProcessor
//设置忽略的自动装配接口，表示这些接口的实现类不允许通过接口自动注入
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
//注册可以自动装配的组件，就是可以在任何组件中允许自动注入的组件
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // Register early post-processor for detecting inner beans as ApplicationListeners.
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   //添加编译时的AspectJ
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // 给beanfactory容器中注册组件ConfigurableEnvironment、systemProperties、systemEnvironment
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
### invokeBeanFactoryPostProcessors 执行bean工厂后置处理器

-  AbstractApplicationContext. invokeBeanFactoryPostProcessors方法
```java
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
invokeBeanFactoryPostProcessors方法内部执行实现了BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor这两个接口的Processor，先获取所有BeanDefinitionRegistryPostProcessor的实现，按优先级执行（是否实现PriorityOrdered优先级接口，是否实现Ordered顺序接口）；再以相同的策略执行所有BeanFactoryPostProcessor的实现。

- PostProcessorRegistrationDelegate. invokeBeanFactoryPostProcessors实现
```java
public static void invokeBeanFactoryPostProcessors(
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

   // Invoke BeanDefinitionRegistryPostProcessors first, if any.
   Set<String> processedBeans = new HashSet<>();

   if (beanFactory instanceof BeanDefinitionRegistry) {
      BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
      List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
      List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

      for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
         if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
            BeanDefinitionRegistryPostProcessor registryProcessor =
                  (BeanDefinitionRegistryPostProcessor) postProcessor;
            registryProcessor.postProcessBeanDefinitionRegistry(registry);
            registryProcessors.add(registryProcessor);
         }
         else {
            regularPostProcessors.add(postProcessor);
         }
      }

      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the bean factory post-processors apply to them!
      // Separate between BeanDefinitionRegistryPostProcessors that implement
      // PriorityOrdered, Ordered, and the rest.
      List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

      // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
      String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
         if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            processedBeans.add(ppName);
         }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();

      // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
      boolean reiterate = true;
      while (reiterate) {
         reiterate = false;
         postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
         for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName)) {
               currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
               processedBeans.add(ppName);
               reiterate = true;
            }
         }
         sortPostProcessors(currentRegistryProcessors, beanFactory);
         registryProcessors.addAll(currentRegistryProcessors);
         invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
         currentRegistryProcessors.clear();
      }

      // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
      invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
      invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
   }

   else {
      // Invoke factory processors registered with the context instance.
      invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let the bean factory post-processors apply to them!
   String[] postProcessorNames =
         beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

   // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      if (processedBeans.contains(ppName)) {
         // skip - already processed in first phase above
      }
      else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

   // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
   List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
   for (String postProcessorName : orderedPostProcessorNames) {
      orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

   // Finally, invoke all other BeanFactoryPostProcessors.
   List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
   for (String postProcessorName : nonOrderedPostProcessorNames) {
      nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
   }
   invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

   // Clear cached merged bean definitions since the post-processors might have
   // modified the original metadata, e.g. replacing placeholders in values...
   beanFactory.clearMetadataCache();
}
```
这里面在处理BeanDefinitionRegistryPostProcessors时有一个非常重要的过程，AnnotationConfigApplicationContext构造函数在初始化reader时为内部beanFactory容器初始化了一个id为org.springframework.context.annotation.internalConfigurationAnnotationProcessor的组件，这是一个ConfigurationClassPostProcessor组件，用来处理添加@Configuration注解的类，并将Bean定义注册到BeanFactory中。

### registerBeanPostProcessors注册BeanPostProcessor（Bean的后置处理器）,用于拦截bean创建过程
注册后置处理器的大致逻辑是：1.获取所有的 BeanPostProcessor2.根据处理器实现的接口区分出4中类型：a.实现PriorityOrdered接口的处理器b.实现Ordered接口的处理器，c.实现MergedBeanDefinitionPostProcessor接口的处理器，d.普通后置处理器3.按这个4中类型依次注册到容器中4.注册一个特殊的后置处理器ApplicationListenerDetector，ApplicationListenerDetector本身也实现了MergedBeanDefinitionPostProcessor接口，有个问题，这个为什么没有在上面c,d之间注册，而是放到最后？

- AbstractApplicationContext .registerBeanPostProcessors(beanFactory)
```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}

public static void registerBeanPostProcessors(
      ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

   String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

   // Register BeanPostProcessorChecker that logs an info message when
   // a bean is created during BeanPostProcessor instantiation, i.e. when
   // a bean is not eligible for getting processed by all BeanPostProcessors.
   int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
   beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

   // Separate between BeanPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
//按优先级分类
   List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
         priorityOrderedPostProcessors.add(pp);
         if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
         }
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   //先注册实现PriorityOrdered接口的处理器，添加到beanfactory容器中beanFactory.addBeanPostProcessor(postProcessor);
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

   //注册实现Ordered接口的处理器
   List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
   for (String ppName : orderedPostProcessorNames) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      orderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         internalPostProcessors.add(pp);
      }
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, orderedPostProcessors);

   // 注册没有实现Ordered或PriorityOrdered的处理器（nonOrderedPostProcessors）
   List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
   for (String ppName : nonOrderedPostProcessorNames) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      nonOrderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         internalPostProcessors.add(pp);
      }
   }
   registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

   // Finally, re-register all internal BeanPostProcessors.
　　//最后，重新注册所有internal BeanPostProcessors（实现MergedBeanDefinitionPostProcessor接口的后置处理器
65    sortPostProcessors(internalPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, internalPostProcessors);

   //注册ApplicationListenerDetector，用于Bean创建完时检查是否是ApplicationListener
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```
### finishBeanFactoryInitialization(beanFactory)
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   //组件转换器相关
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

   //aspectj相关.
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   // Stop using the temporary ClassLoader for type matching.
   beanFactory.setTempClassLoader(null);

   // Allow for caching all bean definition metadata, not expecting further changes.
   beanFactory.freezeConfiguration();

   // 初始化后剩下的单实例bean
   beanFactory.preInstantiateSingletons();
}
```

- DefaultListableBeanFactory. preInstantiateSingletons()方法

```java
public void preInstantiateSingletons() throws BeansException {
   if (logger.isDebugEnabled()) {
      logger.debug("Pre-instantiating singletons in " + this);
   }

   // Iterate over a copy to allow for init methods which in turn register new bean definitions.
   // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
　　//容器中所有bean名称
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   // Trigger initialization of all non-lazy singleton beans...
   for (String beanName : beanNames) {
　　//获取Bean的定义信息；RootBeanDefinition
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
　　//非抽象，单例，非延迟加载
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
　　//是否是FactoryBean
         if (isFactoryBean(beanName)) {
    // 通过"&beanName"获取工厂Bean实例
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                              ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
　　　　　　//不是FactoryBean,则利用getBean(beanName)实例化bean 
            getBean(beanName);
         }
      }
   }

   // Trigger post-initialization callback for all applicable beans...
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

### finishRefresh()
```java
protected void finishRefresh() {
   // Clear context-level resource caches (such as ASM metadata from scanning).
   clearResourceCaches();

   //初始化和生命周期有关的后置处理器LifecycleProcessor，默认DefaultLifecycleProcessor
   initLifecycleProcessor();

   // 回调生命周期处理器
   getLifecycleProcessor().onRefresh();

   //发布容器刷新完成事件：ContextRefreshedEvent
   publishEvent(new ContextRefreshedEvent(this));

   LiveBeansView.registerApplicationContext(this);
}
```
## 小结
以上基本分析了AnnotationConfigApplicationContext容器的初始化过程， Spring容器在启动过程中，会先保存所有注册进来的Bean的定义信息；Spring容器根据条件创建Bean实例，区分单例，还是原型，后置处理器等（后置处理器会在容器创建过程中通过getBean创建，并执行相应的逻辑）；Spring容器在创建bean实例后，会使用多种后置处理器来增加bean的功能，比如处理自动注入，AOP，异步，这种后置处理器机制也丰富了bean的功能。