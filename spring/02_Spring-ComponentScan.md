```xml
<context:component-scan base-package="com.noob.storage.aop"/>
```

我们可以在spring.xml配置文件里边配置context自动扫描，作用是扫描制定范围的组件并注册到 Spring

Spring 怎么识别出需要代理的目标对象? 这个问题属于 spring ioc 的范围，看过 ioc 部分的同学基本应该能够理解，入手点就是对 `context` 这个标签的处理 我们全局搜索 **`component-scan`**这个关键字，最后代码定位到 **`ContextNamespaceHandler`**类


```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {
   @Override
   public void init() {
      registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
      registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
      registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
      registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
      registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
      registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
      registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
      registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
   }
}
```

spring 加载完 xml 配置文件，会对配置文件中的标签进行解析，spring 默认会加载自己的解析器，这些解析器散布在各个不同的 jar 包中。

spring在解析标签时，通过实现`NamespaceHandlerResolver`接口的`DefaultNamespaceHandlerResolver`加载所有classpath下的**spring.handlers** 文件中的映射，并在解析标签时，寻找这些标签对应的处理器，然后用这些处理器来处理标签。所以，当遇到`component-scan`标签，spring 容器就会使用`ComponentScanBeanDefinitionParser`标签进行解析，我们看下`ComponentScanBeanDefinitionParser`来的`parse`方法

```java
@Override
@Nullable
public BeanDefinition parse(Element element, ParserContext parserContext) {
   // 解析 basePackage，转换成数组
   String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
   basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
   String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
         ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

   // Actually scan for bean definitions and register them.
   ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
   // 扫描 basepacket 下所有的 bean
   Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
   // 注册到 spring 中
   registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

   return null;
}
```

观察下如何创建扫描器，以及相关的初始操作

```java
protected ClassPathBeanDefinitionScanner configureScanner(ParserContext parserContext, Element element) {
   //默认使用 spring 自带的注解过滤
   boolean useDefaultFilters = true;
   //解析`use-default-filters`，类型为boolean
   if (element.hasAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE)) {
      useDefaultFilters = Boolean.valueOf(element.getAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE));
   }

   // Delegate bean definition registration to scanner class.
   // 此处如果`use-default-filters`为true，则添加`@Component`、`@Service`、`@Controller`、`@Repository`、			
   // `@ManagedBean`、`@Named`添加到 includeFilters 的集合过滤
   ClassPathBeanDefinitionScanner scanner = createScanner(parserContext.getReaderContext(), useDefaultFilters);
   scanner.setBeanDefinitionDefaults(parserContext.getDelegate().getBeanDefinitionDefaults());
   scanner.setAutowireCandidatePatterns(parserContext.getDelegate().getAutowireCandidatePatterns());
  
   //设置`resource-pattern`属性，扫描资源的模式匹配，支持正则表达式
   if (element.hasAttribute(RESOURCE_PATTERN_ATTRIBUTE)) {
      scanner.setResourcePattern(element.getAttribute(RESOURCE_PATTERN_ATTRIBUTE));
   }

   try {
      //解析name-generator属性 beanName生成器
      parseBeanNameGenerator(element, scanner);
   } catch (Exception ex) {
      parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
   }

   try {
      //解析 scope-resolver 属性和 scoped-proxy 属性，但两者只可存在其一
      //后者值为 targetClass：cglib 代理、interfaces：JDK 代理、no：不使用代理
      parseScope(element, scanner);
   } catch (Exception ex) {
      parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
   }

   //解析子节点`context:include-filter`、`context:exclude-filter`主要用于对扫描class类的过滤
   //例如<context:include-filter type="annotation" expression="org.springframework.stereotype.Controller.RestController" />
   parseTypeFilters(element, scanner, parserContext);

   return scanner;
}
```

进入扫描方法

```java
/**
 * Perform a scan within the specified base packages,
 * returning the registered bean definitions.
 * <p>This method does <i>not</i> register an annotation config processor
 * but rather leaves this up to the caller.
 * @param basePackages the packages to check for annotated classes
 * @return set of beans registered if any for tooling registration purposes (never {@code null})
 */
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      // 对每个基础包都进行扫描寻找并且对基础包下的所有class都注册为BeanDefinition
      // 并对得到的candidates集合进行过滤，此处便用到include-filters和exclude-filters
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         // 解析一个bean的scope属性，代表作用范围
         // prototype->每次请求都创建新的对象 singleton->单例模式，处理多请求
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         // 生成 beanName
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         /* 
          对注册的bean进行另外的赋值处理，比如默认属性的配置
          返回的candidate类型为ScannedGenericBeanDefinition，下面两者条件满足 
          */
         if (candidate instanceof AbstractBeanDefinition) {
            //设置lazy-init/autowire-code默认属性，从spring配置的<beans>节点属性读取
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            //读取bean上的注解，比如`@Lazy`、`@Dependson`的值设置相应的属性
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
        
         //查看是否已注册
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            //代理模式
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);
            //注册bean信息到工厂中
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```

在这里我们只简单的看下其父类`ClassPathScanningCandidateComponentProvider#findCandidateComponents`获取包下的所有class资源文件并实例化为`BeanDefinition`对象

```java
/**
 * Scan the class path for candidate components.
 * @param basePackage the package to check for annotated classes
 * @return a corresponding Set of autodetected bean definitions
 */
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
   if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
      return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
   }
   else {
      return scanCandidateComponents(basePackage);
   }
}
```

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
   Set<BeanDefinition> candidates = new LinkedHashSet<>();
   try {
      // 值类似为classpath*:com/question/sky/**/*.class
      String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
      //通过 PathMatchingResourcePatternResolver 来找寻资源
      //常用的 Resource 为 FileSystemResource
      Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
      boolean traceEnabled = logger.isTraceEnabled();
      boolean debugEnabled = logger.isDebugEnabled();
      for (Resource resource : resources) {
         if (traceEnabled) {
            logger.trace("Scanning " + resource);
         }
         if (resource.isReadable()) {
            try {
               //生成MetadataReader对象->SimpleMetadataReader，内部包含AnnotationMetadataReadingVisitor注解访问处理类
               MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
               //判断class是否不属于 excludeFilters 集合内但至少符合一个 includeFilters 集合
               if (isCandidateComponent(metadataReader)) {
                  //包装为 ScannedGenericBeanDefinition 对象
                  ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                  //保存文件资源
                  sbd.setResource(resource);
                  sbd.setSource(resource);
                  //判断 class 文件是否不为接口或者抽象类并且是独立的
                  if (isCandidateComponent(sbd)) {
                     if (debugEnabled) {
                        logger.debug("Identified candidate component class: " + resource);
                     }
                     //完成验证加入集合中
                     candidates.add(sbd);
                  } else {
                     if (debugEnabled) {
                        logger.debug("Ignored because not a concrete top-level class: " + resource);
                     }
                  }
               } else {
                  if (traceEnabled) {
                     logger.trace("Ignored because not matching any filter: " + resource);
                  }
               }
            } catch (Throwable ex) {
               throw new BeanDefinitionStoreException(
                     "Failed to read candidate component class: " + resource, ex);
            }
         } else {
            if (traceEnabled) {
               logger.trace("Ignored because not readable: " + resource);
            }
         }
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
   }
   return candidates;
}
```

对上面的代码解释作下补充，主要是验证beanDefinition的两个方法

- `ClassPathScanningCandidateComponentProvider#isCandidateComponent(MetadataReader metadataReader)`
  对 class 类进行 filter 集合过滤

```java
/**
 * Determine whether the given class does not match any exclude filter
 * and does match at least one include filter.
 * @param metadataReader the ASM ClassReader for the class
 * @return whether the class qualifies as a candidate component
 */
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
   for (TypeFilter tf : this.excludeFilters) {
      //满足 excludeFilter 集合中的一个便返回 false，表示不对对应的 beanDefinition 注册
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return false;
      }
   }
   for (TypeFilter tf : this.includeFilters) {
      //首先满足其中includeFilter集合中的一个
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         //判断对应的 beanDifinition 不存在 @Conditional 注解或者满足 @Conditional 中指定的条件，则返回true
         return isConditionMatch(metadataReader);
      }
   }
   return false;
}
```

- `ClassPathScanningCandidateComponentProvider#isCandidateComponent(AnnotatedBeanDefinition beanDefinition)`
  验证 beanDefinition class 类是否为具体类

```java
/**
 * Determine whether the given bean definition qualifies as candidate.
 * <p>The default implementation checks whether the class is not an interface
 * and not dependent on an enclosing class.
 * <p>Can be overridden in subclasses.
 * @param beanDefinition the bean definition to check
 * @return whether the bean definition qualifies as a candidate component
 */
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
   AnnotationMetadata metadata = beanDefinition.getMetadata();
   //非抽象类、接口类并且有独立特性[它是一个顶级类还是一个嵌套类(静态内部类)，可以独立于封闭类构造。]
   return (metadata.isIndependent() && (metadata.isConcrete() ||
         // 如果是抽象类并且包含 Lookup 注解方法也行（任意一个方法有就行）
         (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
}
```

> An annotation that indicates 'lookup' methods, to be **overridden by the container to redirect them back to the org.springframework.beans.factory.BeanFactory for a getBean call**. This is essentially`本质上` an annotation-based version of the XML `lookup-method` attribute, resulting in the same runtime arrangement.
> The resolution of the target bean can either be based on the return type (`getBean(Class)`) or on a suggested bean name (`getBean(String)`), in both cases passing the method's arguments to the `getBean` call for applying them as target factory method arguments or constructor arguments.
> Such `lookup` methods can have `default (stub)` implementations that will simply get replaced by the container, or they can be declared as `abstract` - for the **container to fill them in at runtime**. **In both cases, the container will generate runtime subclasses of the method's containing class via CGLIB**, which is why such <u>lookup methods can only work on beans that the container instantiates through **regular** constructors</u>: i.e. lookup methods cannot get replaced on beans returned from factory methods where we cannot dynamically provide a subclass for them.
> Concrete limitations in typical Spring configuration scenarios: When used with component scanning or any other mechanism that filters out abstract beans, provide stub implementations of your lookup methods to be able to declare them as concrete classes. And please remember that lookup methods won't work on beans returned from @Bean methods in configuration classes; you'll have to resort to @Inject Provider<TargetBean> or the like instead

在扫描包内的 class 文件注册为 beanDefinition 之后，ComponentScanBeanDefinitionParser 还需要注册其他的组件，具体是什么可简单看下相关的源码

```java
protected void registerComponents(
      XmlReaderContext readerContext, Set<BeanDefinitionHolder> beanDefinitions, Element element) {

   Object source = readerContext.extractSource(element);
   //包装为 CompositeComponentDefinition 对象，内置多 ComponentDefinition 对象
   CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), source);
   //将已注册的所有beanDefinitionHolder对象放到上述对象中
   for (BeanDefinitionHolder beanDefHolder : beanDefinitions) {
      compositeDef.addNestedComponent(new BeanComponentDefinition(beanDefHolder));
   }

   // Register annotation config processors, if necessary.
   //获取 annotation-config 的属性值，默认为 true
   boolean annotationConfig = true;
   if (element.hasAttribute(ANNOTATION_CONFIG_ATTRIBUTE)) {
      annotationConfig = Boolean.valueOf(element.getAttribute(ANNOTATION_CONFIG_ATTRIBUTE));
   }
   if (annotationConfig) {
      //注册多个 BeanPostProcessor 接口，具体什么可自行查看，返回的是包含BeanPostProcessor接口的beanDefinitionHolder对象集合
      Set<BeanDefinitionHolder> processorDefinitions =
            AnnotationConfigUtils.registerAnnotationConfigProcessors(readerContext.getRegistry(), source);
      //继续装入CompositeComponentDefinition对象
      for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
         compositeDef.addNestedComponent(new BeanComponentDefinition(processorDefinition));
      }
   }
   //此处为空
   readerContext.fireComponentRegistered(compositeDef);
}
```

此处的目的主要是注册多个BeanPostProcessor接口实现类【供后续spring调用统一接口进行解析，比如AbstractApplicationContext#invokeBeanFactoryPostProcessors可执行下述的`@Configuration`解析】具体的有

- ConfigurationClassPostProcessor解析`@Configuration`注解类
- AutowiredAnnotationBeanPostProcessor解析`@Autowired/@Value`注解
- RequiredAnnotationBeanPostProcessor解析`@Required`注解
- CommonAnnotationBeanPostProcessor解析`@Resource`注解
- PersistenceAnnotationBeanPostProcessor 解析 JPA 注解，持久层

