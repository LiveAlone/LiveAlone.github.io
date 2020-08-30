---
title: SpringApplication builder 启动
tags:
  - Spring
  - SpringBoot
categories:
  - 技术
  - Spring
  - SpringBoot
date: 2020-08-29 01:46:07
---


这一章讲讲 SpringApplication app 应用启动方式，参考上一章 [springJar资源加载](/2020/08/28/SpringBootJarLauncher/index.html)。
要启动一个SpringBoot应用， 通过构建一个 SpringApplication 我们引用， 同时提供了 SpringApplicationBuilder 构建器。这一章主要分析这两Class。

#### SpringApplicationBuilder Class 解析

```java
private final SpringApplication application;    // application SpringApplication 构建应用对象。

private ConfigurableApplicationContext context; // application 运行上下文， 熟悉的ApplicationContext 管理Spring 运行上下文。

private SpringApplicationBuilder parent;    // ApplicationContext 支持父子容器，支持属性的继承，记得 SpringCloud 通过父子容器，实现配置获取和覆盖。

private final AtomicBoolean running = new AtomicBoolean(false);

private final Set<Class<?>> sources = new LinkedHashSet<>();

private final Map<String, Object> defaultProperties = new LinkedHashMap<>();

private ConfigurableEnvironment environment;    // 配置环境运行上下文

private Set<String> additionalProfiles = new LinkedHashSet<>();

private boolean registerShutdownHookApplied;

private boolean configuredAsChild = false;
```

其他通过标识Application 运行状态，build 通过 run 方法 启动运行SpringAplication 上下文。

```java
public ConfigurableApplicationContext run(String... args) {
    if (this.running.get()) {
        // If already created we just return the existing context
        return this.context;
    }
    configureAsChildIfNecessary(args);
    if (this.running.compareAndSet(false, true)) {
        synchronized (this.running) {
            // If not already running copy the sources over and then run.
            this.context = build().run(args);
        }
    }
    return this.context;
}
private void configureAsChildIfNecessary(String... args) {
    if (this.parent != null && !this.configuredAsChild) {
        this.configuredAsChild = true;
        if (!this.registerShutdownHookApplied) {
            this.application.setRegisterShutdownHook(false);
        }
        initializers(new ParentContextApplicationContextInitializer(this.parent.run(args)));
    }
}
```

代码比较清晰, 通过 build() 构建 SpringApplication, run(args) 启动我们的引用对象。**这里 parent.run() 预先执行 ParentContextApplicationContexntInitializer 初始化接口，子容器创建添加 parentApplicationContext**

#### SpringApplication 项目容器的启动

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();  // spring 内置StopWatch 用于统计 Spring Application 启动时间统计
    stopWatch.start();
    ConfigurableApplicationContext context = null;  // applicationContext 应用运行上下文对象, 后面初始化
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();    // java.awt.headless 设置运行环境，
    SpringApplicationRunListeners listeners = getRunListeners(args);    // 获取 SpringApplication 构建监听上下文
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);       // environment 运行上下文准备
        Banner printedBanner = printBanner(environment);
        context = createApplicationContext();
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);    // application context 刷新
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

1. headless, Headless模式是在缺少显示屏、键盘或者鼠标是的系统配置，时候一些图片处理类库使用
2. getListeners 这个是 SpringApplication应用启动类监听上下文。区别后面ApplicationContextEventListener监听。 应用启动的不同阶段，用户定制化一些逻辑，输出一些监听信息。
3. 去除监听，干扰项。 应用启动流程比较简单
   - env 运行环境对象 创建,初始化
   - ApplicationContext 创建
   - ApplicationContext 初始化
   - ContextRefresh 核心Bean 加载load
   - refreshAfter 加载一些CommandRunner Bean
   - 运行 RunnerCommand
4. 最后对app 启动异常处理, handleRunFailure(exceptionReports) 处理异常结果。

##### SpringApplicationRunListener 容器运行启动监听者

SpringApplicationRunListeners 是SpinrgApplicationRunListener监听器的管理者, Facade 执行事件分发，下面是监听事件

- starting() starting 没有任何参数, 这个时候 env, context 还没有创建
- environmentPrepared(ConfigurableEnvironment environment) {
- contextPrepared(ConfigurableApplicationContext context)
- contextLoaded(ConfigurableApplicationContext context)
- started(ConfigurableApplicationContext context)
- running(ConfigurableApplicationContext context)
- failed(ConfigurableApplicationContext context, Throwable exception)

SpringApplicationRunListener 可以用户自定义实现，完成定制化需求。同时有一个实现者，EventPublishingRunListener 用于分发Spring 内部事件，后面详细说明。
说说 SpringApplicationRunnerListener 初始化过程

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {  // 通过这个函数获取 Listeners
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };      // SpringApplicationRunListener 构造参数必须 springApplications, args
    return new SpringApplicationRunListeners(logger,
            getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));     // 获取 spring.factory 配置文件 ClassName
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names); // 反射Construct 构造Listeners
    AnnotationAwareOrderComparator.sort(instances); // 比较Ordered 顺序方式, 执行列表
    return instances;
}
```

所以用户定制化添加 ApplicationListener, META-INF spring.factories 添加定制化对的 Listener。

#### SpringFactoriesLoader factories 加载

```java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"; // 加载固定spring.factories 文件位置
private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();  // classloader 级别， 缓存不同property 属性配置
```

可以看到，factoriesLader 缓存是基于 ClassLoader，应该为了应对多 ApplicationContext 不同通loader。 获取依赖jar FACTORIES_RESOURCE_LOCATION resource 配置文件, 通过 PropertiesLoaderUtils load 对应的Properties配置属性文件。OK

#### AnnotationAwareOrderComparator

Spring 通过AnnotationAwareOrderComparator进行对象排序, 继承来自 OrderComaprator

```java
private int doCompare(@Nullable Object o1, @Nullable Object o2, @Nullable OrderSourceProvider sourceProvider) {
    boolean p1 = (o1 instanceof PriorityOrdered);   // priorityOrdered 一定优先 Ordered, 
    boolean p2 = (o2 instanceof PriorityOrdered);
    if (p1 && !p2) {
        return -1;
    }
    else if (p2 && !p1) {
        return 1;
    }

    int i1 = getOrder(o1, sourceProvider);
    int i2 = getOrder(o2, sourceProvider);
    return Integer.compare(i1, i2);
}
```

OrderedSourceProvider spring 提供排序对象转换工具，list 对象进行排序操作。
Ordered 递增排序方式， Ordered.LOWEST_PRECEDENCE = Interger.MAX_VALUE，而且默认的排序。 OrderComparator 通过 Order 继承获取排序数值， Annotation 通过获取Annotation 获取排序的 value。

#### callRunner

运行 ApplicationRunner, CommandLineRunner component 组件bean, 可能一些启动业务 Runner 在其中运行， 但是 bean applicationContext 不会再这里。

剩余 env, appplicationContext 创建，初始化，刷新，spring核心，后面讲。
