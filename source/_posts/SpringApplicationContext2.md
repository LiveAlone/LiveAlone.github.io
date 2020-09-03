---
title: SpringApplicationContext2
tags:
  - Spring
  - SpringBoot
categories:
  - 技术
  - Spring
  - SpringBoot
date: 2020-09-02 00:40:26
---


SpringContext Refresh 刷新所有的启动配置, AbstractApplicationContext 刷新所有的配置列表内容。

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        prepareRefresh();
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        prepareBeanFactory(beanFactory);

        try {
            postProcessBeanFactory(beanFactory);
            invokeBeanFactoryPostProcessors(beanFactory);
            registerBeanPostProcessors(beanFactory);
            initMessageSource();
            initApplicationEventMulticaster();
            onRefresh();
            registerListeners();
            finishBeanFactoryInitialization(beanFactory);
            finishRefresh();
        }
        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }
            destroyBeans();
            cancelRefresh(ex);
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

prepare refesh, Context 加载预先准备

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

    initPropertySources(); // 初始化PropertySource资源, AbstractApplicationContext 是一个空实现， 不同Context 自定义添加Source 资源来源

    // Validate that all properties marked as required are resolvable:
    // see ConfigurablePropertyResolver#setRequiredProperties
    getEnvironment().validateRequiredProperties();  // 验证

    // 初始化 early earlyApplicationListeners，earlyApplicationEvents 监听事件类型
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

obtainFreshBeanFactory() 获取BeanFactory。 AbstractionContext 通知继承者， refreshBeanFactory 回调。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 添加了 Aware BeanPostProcessor 忽略所有的 Aware
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 指定 bean 依赖 Class 固定的对象类型
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 注册 loadTimeWeaver bean PostProcessor, class load 的编制, 特殊场景使用， 我没见用过
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 注册特殊Bean
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

##### bean factory processor

执行 BeanFactoryPorcessor bean factory 处理器执行过程。```postProcessBeanFactory(beanFactory);``` subclass处理执行BeanFactory， 然后执行 ```invokeBeanFactoryPostProcessors(beanFactory);``` bean Factory Processor。

invoker 执行BeanFactoryPostProcesser, static 方法，初始化loader 相关BeanDefinition 内容。

- SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor
- ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor
- ConfigFileApplicationListener$PropertySourceOrderingPostProcessor

```java
if (beanFactory instanceof BeanDefinitionRegistry) {
    BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
    List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
    List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

    // 预先注册 区分出了 BeanFactoryProcessor 和 BeanDefinitionProcessor, definition 预先执行
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

    // bean factory 获取 BeanDefinitionRegistryPostProcessor, 这个时候获取Class 循环解析结果
    List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
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

    // 重新获取BeanDefinition 解析 Bean, 
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

    // 最后循环执行, 直到没有新的BeanDefinitionRegistryPostProcessor 产生以后结束
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

    // 最后执行 BeanFactoryProcessor, 包含对BeanDefinition 增强， 添加BeanPostProcessor
    invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
}

else {
    // Invoke factory processors registered with the context instance.
    invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
}
```

后面会重新获取 BeanFactoryPostProcessor，刚刚BeanDefinition, BeanFactoryProcessor 可能影响了 BeanFactory 中 BeanFactoryProcessor 数据集合。
最后执行 priorityOrderedPostProcessors, orderedBeanFactoryPostProcessor,nonOrderedPostProcessors 顺序执行列表。
上面BeanFactoryProcessor 执行完成了， 初始化一些BeanDefinition 内容。

#### registerBeanPostProcessors 注册BeanPostProcessor

区分三种类型 priorityOrderedPostProcessors，internalPostProcessors，orderedPostProcessorNames，nonOrderedPostProcessorNames。
这些Processor 执行顺序, priorityOrderedPostProcessors > orderedPostProcessors > nonOrderedPostProcessors > internalPostProcessors
internalPostProcessors 对于类型是 MergedBeanDefinitionPostProcessor，包含前三个集合数据， bean 实例化以后， 对齐合并定义， 对BeanPostProcessor 的扩充。

initMessageSource 初始化 MessageSource bean, initApplicationEventMulticaster 注册消息分发对象。

这个时候 BeanFactory 初始化基本已经完成了， BeanDefinition 资源已经加载完毕了， 但是 并没有初始化 创建Bean。所以，下面 OnRefresh() 不同的 App 实现不同的扩展模块， web 实现 tomcat 服务启动 等等。 registerListeners() 完成不同 ApplicationListener bean 注册factory 中。 ```finishBeanFactoryInitialization(beanFactory);``` 立即初始化所有的单例Bean。 FinishRefresh() 事件分发，初始化一下 lifecycleBean。 最后清理所有 Cache 解析缓存。
```beanFactory.preInstantiateSingletons``` init singletion bean，依赖BeanDefinition 执行init 方法, SmartInitializingSingleton 等等。 DETAIL TODO

