# ApplicationContext

先从最基本的启动 Spring 代码开始

```java
ApplicationContext context 
= new ClassPathXmlApplicationContext("classpath:applicationcontext.xml");
```

上述代码是根据 classpath 去寻找 spring applicationcontext.xml 配置文件，从而完成 spring 上下文的初始化。除了 ClassPathXmlApplicationContext 以外，我们也还有其他构建 ApplicationContext 的方案可供选择，我们先来看看大体的继承结构：

![Spring-ApplicationContext](ref/Spring-ApplicationContext.png)

其中：

**FileSystemXmlApplicationContext** 的构造函数需要一个 xml 配置文件在**系统中的路径**，其他和 ClassPathXmlApplicationContext 基本上一样。

**AnnotationConfigApplicationContext** 是基于**注解**来使用的，它不需要配置文件，采用 java 配置类和各种注解来配置，是比较简单的方式。

# BeanFactory 类结构图

![Spring-BeanFactory.png](ref/Spring-BeanFactory.png)

**说明**

1. **ApplicationContext** 继承了 **ListableBeanFactory**，这个 Listable 的意思就是，通过这个接口，我们可以获取多个 Bean，最顶层 BeanFactory 接口的方法都是获取单个 Bean 的。
2. **ApplicationContext** 继承了 **HierarchicalBeanFactory**，Hierarchical(分级的)，也就是说我们可以在应用中起多个 BeanFactory，然后可以将各个 BeanFactory 设置为**父子关系**。
3. **AutowireCapableBeanFactory** 这个名字中的 Autowire 大家都非常熟悉，它就是用来**自动装配 Bean** 用的，但是仔细看上图，ApplicationContext 并没有继承它，不过不用担心，不使用继承，不代表不可以使用组合，如果你看到 ApplicationContext 接口定义中的最后一个方法 getAutowireCapableBeanFactory() 就知道了。
4. **ConfigurableListableBeanFactory** 也是一个特殊的接口，看图，特殊之处在于它继承了第二层所有的三个接口，而 ApplicationContext 没有。这点之后会用到。

**BeanFactory**  定义了 `getBean` 的一些接口，除此之外，还定义了一些 `getType`  `getAliases` 等工具方法。

**ListableBeanFactory** 定义了获取所有 bean 信息的一些接口，比如 `getBeanDefinitionNames` 获取所有 BeanName，`getBeanNamesForType` 根据 Type 查找符合条件的 Bean ，检测 Bean 是否存在 `containsBeanDefinition` 等等。

**HierarchicalBeanFactory** 只定义了 2 个方法。 `BeanFactory getParentBeanFactory();`  用来获取父BeanFactory。 `boolean containsLocalBean(String name);` 判断在不考虑父Factory的情况下，当前BeanFactory 是否包含指定的 Bean。

**AutowireCapableBeanFactory** 定义了一些 创建Bean，自动注入 Bean 和 配置Bean 的一些方法

**ApplicationContext** 继承了 `ListableBeanFactory` 和 `HierarchicalBeanFactory` ，所以具有这2个类的所有功能，同时，**内部还定义了`AutowireCapableBeanFactory getAutowireCapableBeanFactory()` 方法，用于获取一个 `AutowireCapableBeanFactory`**。

# 启动过程分析

首先看一下 ClassPathXmlApplicationContext 及其构造方法

```java
/**
 * Standalone XML application context, taking the context definition files
 * from the class path, interpreting plain paths as class path resource names
 * that include the package path (e.g. "mypackage/myresource.txt"). Useful for
 * test harnesses as well as for application contexts embedded within JARs.
 *
 * <p>The config location defaults can be overridden via {@link #getConfigLocations},
 * Config locations can either denote concrete files like "/myfiles/context.xml"
 * or Ant-style patterns like "/myfiles/*-context.xml" (see the
 * {@link org.springframework.util.AntPathMatcher} javadoc for pattern details).
 *
 * <p>Note: In case of multiple config locations, later bean definitions will
 * override ones defined in earlier loaded files. This can be leveraged to
 * deliberately override certain bean definitions via an extra XML file.
 *
 * <p><b>This is a simple, one-stop shop convenience ApplicationContext.
 * Consider using the {@link GenericApplicationContext} class in combination
 * with an {@link org.springframework.beans.factory.xml.XmlBeanDefinitionReader}
 * for more flexible context setup.</b>
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see #getResource
 * @see #getResourceByPath
 * @see GenericApplicationContext
 */
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {

   @Nullable
   private Resource[] configResources;

   /**
    * Create a new ClassPathXmlApplicationContext with the given parent,
    * loading the definitions from the given XML files.
    * @param configLocations array of resource locations
    * @param refresh whether to automatically refresh the context,
    * loading all bean definitions and creating all singletons.
    * Alternatively, call refresh manually after further configuring the context.
    * @param parent the parent context
    * @throws BeansException if context creation failed
    * @see #refresh()
    */
   public ClassPathXmlApplicationContext(
         String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
         throws BeansException {
      super(parent);
     	// 根据提供的configLocations，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
      setConfigLocations(configLocations);
      if (refresh) {
         refresh();// 核心方法
      }
   }
```

接下来，就是 refresh()，这里简单说下为什么是 refresh()，而不是 init() 这种名字的方法。因为 ApplicationContext 建立起来以后，其实我们是**可以通过调用 refresh() 这个方法重建的，这样会将原来的 ApplicationContext 销毁，然后再重新执行一次初始化操作。**

往下看，refresh() 方法是由 AbstractApplicationContext 抽象类来实现的，具体逻辑如下，这里只看一下大概流程，后续在详细介绍。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   // synchronized 避免并发重复刷新
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
     	// 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
     	// 这步比较关键，这步完成后，配置文件就会解析成一个个 beanDefinition，注册到 BeanFactory 中，
      // 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
      // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
      // 这块待会会展开说
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         // 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
         // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】
         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         // 初始化当前 ApplicationContext 的 MessageSource，国际化这里就不展开说了，不然没完没了了
         initMessageSource();

         // Initialize event multicaster for this context.
         // 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         // 从方法名就可以知道，典型的模板方法(钩子方法)，
         // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
         onRefresh();

         // Check for listener beans and register them.
         // 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         // 重点，重点，重点
         // 初始化所有的 singleton beans
         //（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         // 最后，广播事件，ApplicationContext 初始化完成
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
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

下面再一步步分析 refresh 方法

# 创建 Bean 容器前的准备工作

这个比较简单，直接看代码中的几个注释即可

```java
/**
 * Prepare this context for refreshing, setting its startup date and
 * active flag as well as performing any initialization of property sources.
 */
protected void prepareRefresh() {
   // Switch to active.
   this.startupDate = System.currentTimeMillis(); // 记录启动时间
   this.closed.set(false); 
   this.active.set(true); // 设置状态

   if (logger.isDebugEnabled()) {
      if (logger.isTraceEnabled()) {
         logger.trace("Refreshing " + this);
      }
      else {
         logger.debug("Refreshing " + getDisplayName());
      }
   }

   // Initialize any placeholder property sources in the context environment.
   initPropertySources();

   // Validate that all properties marked as required are resolvable:
   // see ConfigurablePropertyResolver#setRequiredProperties
   // 校验 xml 配置文件
   getEnvironment().validateRequiredProperties();

   // Store pre-refresh ApplicationListeners...
   if (this.earlyApplicationListeners == null) {
      this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
   }
   else {
      // Reset local application listeners to pre-refresh state.
      this.applicationListeners.clear();
      this.applicationListeners.addAll(this.earlyApplicationListeners);
   }

   // Allow for the collection of early ApplicationEvents,
   // to be published once the multicaster is available...
   this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

# 创建 Bean 容器，加载并注册 Bean

注意，这个方法是全文最重要的部分之一，这里将会初始化 BeanFactory、加载 Bean、注册 Bean 等等。当然，这步结束后，Bean 并没有完成初始化。

**AbstractApplicationContext** =>

```java
/**
 * Tell the subclass to refresh the internal bean factory.
 * @return the fresh BeanFactory instance
 * @see #refreshBeanFactory()
 * @see #getBeanFactory()
 */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	 // 关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
   refreshBeanFactory();
   // 返回刚刚创建的 BeanFactory
   return getBeanFactory();
}
```

**AbstractRefreshableApplicationContext** =>

```java
/**
 * This implementation performs an actual refresh of this context's underlying
 * bean factory, shutting down the previous bean factory (if any) and
 * initializing a fresh bean factory for the next phase of the context's lifecycle.
 */
@Override
protected final void refreshBeanFactory() throws BeansException {
   // 如果 ApplicationContext 中已经加载过 BeanFactory 了，销毁所有 Bean，关闭 BeanFactory
   // 注意，应用中 BeanFactory 本来就是可以多个的，这里可不是说应用全局是否有 BeanFactory，而是当前
   // ApplicationContext 是否有 BeanFactory
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
			// 初始化一个 DefaultListableBeanFactory，为什么用这个，我们马上说。
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      // 用于 BeanFactory 的序列化，我想大部分人应该都用不到
      beanFactory.setSerializationId(getId());

      // 下面这两个方法很重要，别跟丢了，具体细节之后说
      // 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
      customizeBeanFactory(beanFactory);

      // 加载 Bean 到 BeanFactory 中
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}
```

> 看到这里的时候，我觉得读者就应该站在高处看 `ApplicationContext` 了，**<u>`ApplicationContext` 继承自 BeanFactory，但是它不应该被理解为 `BeanFactory` 的实现类，而是说其内部持有一个实例化的 `BeanFactory` (`DefaultListableBeanFactory`),以后所有的 `BeanFactory` 相关的操作其实是给这个实例来处理的。</u>**

我们说说为什么选择实例化 `DefaultListableBeanFactory` ？前面我们说了有个很重要的接口 `ConfigurableListableBeanFactory`，它实现了 `BeanFactory` 下面一层的所有三个接口,  可以回顾一下 `BeanFactory` 的类图。

从类图上可以看到 `ConfigurableListableBeanFactory` 只有一个实现类 `DefaultListableBeanFactory`，而且实现类 `DefaultListableBeanFactory` 还通过实现右边的 `AbstractAutowireCapableBeanFactory` 通吃了右路。所以结论就是，最底下这个家伙 `DefaultListableBeanFactory` 基本上是最牛的 `BeanFactory` 了，这也是为什么这边会使用这个类来实例化的原因。

在继续往下之前，我们需要先了解 `BeanDefinition`。**我们说 `BeanFactory` 是 `Bean` 容器，那么 `Bean` 又是什么呢？**

这里的 `BeanDefinition` 就是我们所说的 `Spring` 的 `Bean`，我们自己定义的各个 `Bean` 其实会转换成一个个 `BeanDefinition` 存在于 `Spring` 的 `BeanFactory` 中。

所以，如果有人问你 `Bean` 是什么的时候，你要知道 `Bean` 在代码层面上是 `BeanDefinition` 的实例。

> BeanDefinition 中保存了我们的 Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等等。

我们来看下 **BeanDefinition** 的接口定义

```java
/**
 * A BeanDefinition describes a bean instance, which has property values,
 * constructor argument values, and further information supplied by
 * concrete implementations.
 *
 * <p>This is just a minimal interface: The main intention is to allow a
 * {@link BeanFactoryPostProcessor} to introspect and modify property values
 * and other bean metadata.
 *
 * @see ConfigurableListableBeanFactory#getBeanDefinition
 * @see org.springframework.beans.factory.support.RootBeanDefinition
 * @see org.springframework.beans.factory.support.ChildBeanDefinition
 */
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

   // 我们可以看到，默认只提供 sington 和 prototype 两种，
   // 很多读者都知道还有 request, session, globalSession, application, websocket 这几种，
   // 不过，它们属于基于 web 的扩展。
   String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
   String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

   // Bean 的角色分类，比较不重要，直接跳过吧
   int ROLE_APPLICATION = 0;
   int ROLE_SUPPORT = 1;
   int ROLE_INFRASTRUCTURE = 2;
  
   // Modifiable attributes

	 // 设置父 Bean，这里涉及到 bean 继承，不是 java 继承。请参见附录介绍
   void setParentName(@Nullable String parentName);

   // 获取父 Bean 名称
   @Nullable
   String getParentName();
   // 设置 Bean 的类名称
   void setBeanClassName(@Nullable String beanClassName);
	 // 获取 Bean 的类名称
   @Nullable
   String getBeanClassName();
	 // 设置 bean 的 scope
   void setScope(@Nullable String scope);
   @Nullable
   String getScope();

	 // 设置是否懒加载
   void setLazyInit(boolean lazyInit);
   boolean isLazyInit();

   // 设置该 Bean 依赖的所有的 Bean，注意，这里的依赖不是指属性依赖(如 @Autowire 标记的)，
   // 是 depends-on="" 属性设置的值
   void setDependsOn(@Nullable String... dependsOn);
   @Nullable
   String[] getDependsOn();

   /**
    * Set whether this bean is a candidate for getting autowired into some other bean.
    * <p>Note that this flag is designed to only affect type-based autowiring.
    * It does not affect explicit references by name, which will get resolved even
    * if the specified bean is not marked as an autowire candidate. As a consequence,
    * autowiring by name will nevertheless inject a bean if the name matches.
    */
   // 设置该 Bean 是否可以注入到其他 Bean 中，只对根据类型注入有效，
   // 如果根据名称注入，即使这边设置了 false，也是可以的
   void setAutowireCandidate(boolean autowireCandidate);
   // 该 Bean 是否可以注入到其他 Bean 中
   boolean isAutowireCandidate();

	 // 主要的。同一接口的多个实现，如果不指定名字的话，Spring 会优先选择设置 primary 为 true 的 bean
   void setPrimary(boolean primary);
   boolean isPrimary();

   // 如果该 Bean 采用工厂方法生成，指定工厂名称。对工厂不熟悉的读者，请参加附录
   void setFactoryBeanName(@Nullable String factoryBeanName);
   @Nullable
   String getFactoryBeanName();

   /**
    * Specify a factory method, if any. This method will be invoked with
    * constructor arguments, or with no arguments if none are specified.
    * The method will be invoked on the specified factory bean, if any,
    * or otherwise as a static method on the local bean class.
    * @see #setFactoryBeanName
    * @see #setBeanClassName
    */
   // 指定工厂类中的 工厂方法名称
   void setFactoryMethodName(@Nullable String factoryMethodName);
   @Nullable
   String getFactoryMethodName();

	 // 构造器参数
   ConstructorArgumentValues getConstructorArgumentValues();
   default boolean hasConstructorArgumentValues() {
      return !getConstructorArgumentValues().isEmpty();
   }

   /**
    * Return the property values to be applied to a new instance of the bean.
    * <p>The returned instance can be modified during bean factory post-processing.
    * @return the MutablePropertyValues object (never {@code null})
    */
   // Bean 中的属性值，后面给 bean 注入属性值的时候会说到
   MutablePropertyValues getPropertyValues();
   default boolean hasPropertyValues() {
      return !getPropertyValues().isEmpty();
   }


   void setInitMethodName(@Nullable String initMethodName);
   @Nullable
   String getInitMethodName();

   void setDestroyMethodName(@Nullable String destroyMethodName);
   @Nullable
   String getDestroyMethodName();


   void setRole(int role);
   int getRole();
   void setDescription(@Nullable String description);
   @Nullable
   String getDescription();

   // Read-only attributes

   boolean isSingleton();
   boolean isPrototype();
   boolean isAbstract();
   @Nullable
   String getResourceDescription();
   @Nullable
   BeanDefinition getOriginatingBeanDefinition();

}
```