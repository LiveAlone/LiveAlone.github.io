---
title: SpringApplicationContext1
date: 2020-09-01 15:04:17
tags:
  - Spring
  - SpringBoot
categories:
  - 技术
  - Spring
  - SpringBoot
---

ApplicationContext 数据创建build, SpringApplication 不同的服务类型， 创建不同的ApplicationContext。

```java
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
        + "annotation.AnnotationConfigApplicationContext";
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
        + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
        + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";
```

Spring BeanUtils 初始化 org.springframework.context.annotation.AnnotationConfigApplicationContext, init constructor。 ApplicationContext 继承来自 BeanFactory 接口继承关系 ![BeanFactory](/images/20200901/BeanFactory.png)

- BeanFactory spring bean 管理工厂
  - ListableBeanFactory 支持通过Name Class Annotation 获取BeanList 遍历查询操作
  - HierarchicalBeanFactory BeanFactory 继承关系
    - ConfigurableBeanFactory, ConfigurableListableBeanFactory 配置化 benaFactory 工厂
  - AutowireCapableBeanFactory 自动注入非Spring 管理Bean 类型

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
    MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
    }
```

ApplicationContext 继承了 BeanFactory,EventPublisher,ResourceLoader,MessageSource 上下列表。 ApplicationContext 继承接口实现 ![ApplicationContext](/images/20200901/ApplicationContext.png)

- ConfigurableApplicationContext 基础实现，支持 add BeanFactoryPostProcessor, beanFactory 处理器添加
  - WebServerApplicationContext, ReactiveWebApplicationContext web page 上下文， WebServer 服务获取
    - ConfigurableWebServerApplicationContext，ConfigurableReactiveWebApplicationContext webApplicationContext上下文支持

不同ApplicationContext 上下文的实现环境 ![AbstractApplicationContext](/images/20200901/AbstractApplicationContext.png)

AnnotationConfigApplicationContext applicationContext bean 上下文定义注册方式

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
    private final AnnotatedBeanDefinitionReader reader; // annotation 注册使用BeanDefinition
    private final ClassPathBeanDefinitionScanner scanner;   // scanner 扫描对应的 package 加载 Bean 定义方式
}
```

SpringApplication 核心Context ```prepareContext, refreshContext, afterRefresh``` context 配置修改

##### prepareContext() context

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
        SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
                .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // Load the sources
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);
}
```

1. postProcessApplicationContext(context) 预处理相关ApplicationContext, 设置BeanNameGenerate, resourceLoader, ConversionService 配置Bean 属性Config。
2. ApplicationContextInitializer context 环境初始化， 下面会详细讲述
3. 执行 ApplicationListeners 线程同步执行监听器, 发放 ContextPreparedEvent 提供监听。
4. 注册 ```springApplicationArguments, springBootBanner``` application Spring bean 对象
5. allowBeanDefinitionOverriding 默认true， 定义false， 不允许同名BeanDefinition 覆盖操作。
6. load beans to ApplicationContext， app 上下文加载beans
7. contextedLoaded 事件监听分发

不同的Listener 注册对应的Bean， 但是不希望直接操作 ApplicationContext, 通过添加BeanFactoryProcessor, 走正常的 ApplicationContext 初始化流程。

##### ApplicationContextInitializer context 初始化

通过 spring.factory 配置文件， 获取 ApplicationContextInitializer 初始化Bean。 initApplicationContext 包含

- DelegatingApplicationContextInitializer ```context.initializer.classes``` 执行这些用户定制化的 init class bean
- SharedMetadataReaderFactoryContextInitializer 添加 CachingMetadataReaderFactoryPostProcessor beanPostProcessor
- ContextIdApplicationContextInitializer 初始化ContextId, 设置ApplicationContext ContextId, 注册Bean
- ConfigurationWarningsApplicationContextInitializer 添加 ConfigurationWarningsPostProcessor
- RSocketPortInfoApplicationContextInitializer 添加 RSocketServerInitializedEvent rsocket 初始化监听事件
- ServerPortInfoApplicationContextInitializer WebServerInitializedEvent webServer 事件监听。
- ConditionEvaluationReportLoggingListener ConditionEvaluationReportListener 添加Listener

可以看到，执行初始化监听者时候， 注入不同的 Processor Listener， ApplicaitonContext 本身留有很多 hook， 执行定制化需求。

#### applicationContext load resources

AnnotationBean 注册启动Class, BootDemoApplication(用户自定义启动bean类型)， 通过Class 注册第一个 BeanDefinition 定义类型。

#### refresh application context

刷新application context 这里会 load 所有的Bean，完整执行流程， 后面沟通

#### after refresh

customer 自定义继承实现， 这个时候， application 已经完成 Context 完成加载。 这个地方个人理解， 是一个和 ApplicationContext(BeanFactory) 不相关的一个初始化。 其实在 ApplicaitonContext 中， 也会有Hook, customer 自定义容器相关的初始化。 FIN
