---
layout:     post
title:      "Spring源码解析(一)"
subtitle:   ""
date:       2020-12-27
author:     "Deer"
header-img: "img/star-universe-space-all-clouds-cumulus-clouds.jpg"
tags:
    - Spring源码系列
---

叨叨好久的源码，终于要开始了。

## Catagory

1. [AnnotationConfigApplicationContext的构造函数](#AnnotationConfigApplicationContext的构造函数)
	1. [this()](#this())
	2. [register(annotatedClasses)](#register(annotatedClasses))
    3. [refresh()](#refresh())
        1. [prepareRefresh()](#prepareRefresh())
        2. [obtainFreshBeanFactory()](#obtainFreshBeanFactory())
        3. [prepareBeanFactory(beanFactory)](#prepareBeanFactory(beanFactory))
2. [参考资料](#参考资料)

---


## AnnotationConfigApplicationContext的构造函数
从第一句开始吧
```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(TestBean.class);

// 点进AnnotationConfigApplicationContext
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    register(annotatedClasses);
    refresh();
}
```
### this()
```java
// 点进去this(),是进行初始化
public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
AnnotatedBeanDefinitionReader，ClassPathBeanDefinitionScanner它俩的主要功能是注册BeanDefinition。（AnnotatedBeanDefinitionReader支持注册单个BeanDefinition，ClassPathBeanDefinitionScanner是通过注解(@PropertySources,@ComponentScan,@ComponentScans,@Import,@ImportResource等)支持批量注册多个）。这里先详细描述AnnotatedBeanDefinitionReader。

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    this(registry, getOrCreateEnvironment(registry));
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    Assert.notNull(environment, "Environment must not be null");
    this.registry = registry;
    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    // 主要就是这个方法
    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}

public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
    registerAnnotationConfigProcessors(registry, null);
}

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

    // 获取beanFactory
    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    if (beanFactory != null) {
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }

    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

    // 注册常用的处理器
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                    AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
        }
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }

    return beanDefs;
}
```
注册常用的处理器:

ConfigurationClassPostProcessor:处理@Configuration,@Component,@ComponentScan,@Import,@ImportSource。(ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry,ConfigurationClassPostProcessor#processConfigBeanDefinitions)

AutowiredAnnotationBeanPostProcessor:处理@Autowired

CommonAnnotationBeanPostProcessor:处理@Resource

若支持JPA，注册PersistenceAnnotationBeanPostProcessor

EventListenerMethodProcessor:处理@EventListener

所以，此时的beanFactory已经有了一些属性，如下图：
![beanFactory](/img/spring-code/spring-code1.png "beanFactory")

#### register(annotatedClasses)
OK，了解了构造函数，那么返回去继续看下一句：register(annotatedClasses);
这个方法真正的执行方法是AnnotatedBeanDefinitionReader#doRegisterBean，那就来看这个方法！

```java
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

    // 这句话代表着通过AnnotatedBeanDefinitionReader注册的都是AnnotatedGenericBeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    // (1)判断是否需要注册 根据@Conditional注解
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(instanceSupplier);
    // 设置scope
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    // (2)生成beanName
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
    // 设置一些注解属性
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // 正常程序调用qualifiers均为null，除非第三方拓展，手动传入了qualifiers参数。
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    // bean的回调函数（此处还没有仔细研究，后期研究后更新）
    for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
        customizer.customize(abd);
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    //（3）解析proxyMode属性
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 注册beanDefinition,即放入一个Map<String, BeanDefinition> beanDefinitionMap中
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

（1）this.conditionEvaluator.shouldSkip(abd.getMetadata())
conditionEvaluator 是解析@Conditional注解的，若有@Conditional注解而且注解属性值为true，那么就skip

（2）生成beanName：这里beanName的作用就是作为beanDefinitionMap的key

（3）解析proxyMode属性：如果proxyMode =ScopedProxyMode.INTERFACES/TARGET_CLASS的话，那么要进行处理。简单来说，对象A为单例，它有属性B，B的scope为session/request，那么在Spring应用上下文加载创建A的时候，此时B还没有创建，那么会生成一个B的代理对象进行注入。proxyMode指明了是采用jdk还是cglib生成代理对象

### refresh()
紧接着，refresh()方法

```java
// 先记录这三个方法啦，后续的下篇博客见～
// Prepare this context for refreshing.
prepareRefresh();

// Tell the subclass to refresh the internal bean factory.
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

// Prepare the bean factory for use in this context.
prepareBeanFactory(beanFactory);

```
#### prepareRefresh()  
首先prepareRefresh()方法

```java
protected void prepareRefresh() {
    // Switch to active.
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    if (logger.isDebugEnabled()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Refreshing " + this);
        }
        else {
            logger.debug("Refreshing " + getDisplayName());
        }
    }

    // Initialize any placeholder property sources in the context environment.
    // 这个方法是初始化对象属性，但在这里没有做任何操作，如果有需要的话，可以靠子类扩展
    initPropertySources();

    // Validate that all properties marked as required are resolvable:
    // see ConfigurablePropertyResolver#setRequiredProperties
    // 验证一些必须的属性值，若为null，则抛出异常
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

####  obtainFreshBeanFactory()  
然后ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()这个方法获取beanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}

/**
    * Return the single internal BeanFactory held by this context
    * (as ConfigurableListableBeanFactory).
    */
@Override
public final ConfigurableListableBeanFactory getBeanFactory() {
    return this.beanFactory;
}
```
#### prepareBeanFactory(beanFactory)
最后prepareBeanFactory(beanFactory);这个方法设置beanFactory的一些属性，为后期的工作做准备。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
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
对于prepareBeanFactory方法，打算对其中的ignoreDependencyInterface，registerResolvableDependency这两个方法做一些解释。

ignoreDependencyInterface这个方法，看起来是忽略指定接口，但其实不是。忽略指定的类和接口是ignoreDependencyType。而ignoreDependencyInterface方法是在自动装配时忽略该接口实现类中和setter方法入参相同的类型，也就是忽略该接口实现类中存在依赖外部的bean属性注入。
```java
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);

public interface EnvironmentAware extends Aware {
	void setEnvironment(Environment environment);
}

```
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);所以拿这句代码来说，它的意思就是忽略EnvironmentAware实现类中的setEnvironment方法注入。


registerResolvableDependency方法是在当接口的多个实现类都已经注入到容器中，spring不知道用哪个时，就可以通过BeanFactoryPostProcessor实现类中的postProcessBeanFactory方法指定要注入的实现类.
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);这句代码的意思就是说虽然BeanFactory接口的实现类有很多，但在这里注入的是beanFactory。


---

## 参考资料
<small>源码分析总算是开始了，在阅读源码的过程中，网上看了很多博主的优秀文章，真的很棒！优秀的人好多呀～现在的自己还是在别人博客的基础上进行学习，希望以后自己也可以有一套很完美属于自己的见解，一定要加油！</small>
- <small>[registerResolvableDependency方法](https://blog.csdn.net/yuge1123/article/details/106053857)</small>
- <small>[ignoreDependencyInterface方法](https://www.jianshu.com/p/3c7e0608ff1f?from=singlemessage)</small>
- <small>[EnvironmentAware接口](https://blog.csdn.net/bazhuayu_1203/article/details/78658196)</small>
- <small>[极其优秀！榜样！](https://daimingzhi.blog.csdn.net/article/details/105667900)</small>





