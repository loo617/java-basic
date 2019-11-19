## refresh
refresh()方法首先会获取beanFactory，然后注册bean到beanFactory中，之后添加一些beanFactoryPostProcessor处理器并调用，接着注册beanPostProcessor处理器，再然后初始化国际化消息源、事件广播器，接着执行钩子方法onRefresh()，接着注册事件监听器，接着初始化所有单例，最后广播事件ApplicationContext初始化完成

## beanFactoryPostProcessor与beanPostProcessor(BFPP and BPP)
1.触发时机不同BFPP是在refresh()方法中调用，BPP实际调用时机是在getBean方法获取对象时调用。
2.两者处理的对象不同，BFPP处理的是解析完配置文件注册在容器中的BeanDefinition，BPP处理的是通过反射生产的实例Bean
3.接口样式不同，BFPP只有一个后处理方法，BPP有一个前置处理方法和一个后置处理方法。
### beanFactoryPostProcessor的几个实现
- PropertyPlaceholderConfigurer允许我们在xml文件中使用占位符来加载properties中对应的配置
- PropertyOverrideConfigurer对容器中配置的任何你想处理的bean定义的property信息进行覆盖替换
- CustomEditorConfigurer配置自定义的属性编辑器
### beanPostProcessor的几个实现
- ApplicationContextAwareProcessors实现一个bean在初始化时获取spring容器内容
- ServletContextAwareProcessor实现一个bean在初始化时获取ServletContext
## bean的生命周期

## spring如何解决循环依赖
Spring通过将实例化后的对象提前暴露给Spring容器中的singletonFactories，解决了循环依赖的问题。
### 场景
- A的构造方法中依赖了B的实例对象，同时B的构造方法中依赖了A的实例对象
- A的构造方法中依赖了B的实例对象，同时B的某个field或者setter需要A的实例对象，以及反之
- A的某个field或者setter依赖了B的实例对象，同时B的某个field或者setter依赖了A的实例对象，以及反之
Spring对于循环依赖的解决不是无条件的，首先前提条件是针对scope单例并且没有显式指明不需要解决循环依赖的对象，而且要求该对象没有被代理过。同时Spring解决循环依赖也不是万能，以上三种情况只能解决两种，第一种在构造方法中相互依赖的情况Spring也无力回天。

### 循环依赖发生的位置
- createBeanInstance 实例化
- populateBean 填充属性
- initializeBean 调用spring xml中指定的init方法
在二步会发生循环依赖

### 三级缓存
- singletonObjects用于存放完好的bean,从缓存中取出的bean可以直接使用
- earlySingletonObjects存放原始的bean对象（尚未装配属性),用于解决循环依赖 
- singletonFactories存放bean工厂对象，用于解决循环依赖 

## springAOP原理
- Spring加载自动代理器AnnotationAwareAspectJAutoProxyCreator，当作一个系统组件
- 当一个bean加载到Spring中时，会触发自动代理器中的bean后置处理
- bean后置处理，会先扫描bean中所有的Advisor
- 然后用这些Adviosr和其他参数构建ProxyFactory
- ProxyFactory会根据配置和目标对象的类型寻找代理的方式（JDK动态代理或CGLIG代理）
- 然后代理出来的对象放回context中，完成Spring AOP代理