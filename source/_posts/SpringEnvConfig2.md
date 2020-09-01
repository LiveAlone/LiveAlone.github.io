---
title: Spring Environment Config 2
tags:
  - Spring
  - SpringBoot
categories:
  - 技术
  - Spring
  - SpringBoot
date: 2020-09-01 01:38:10
---


想说明一下 Spring Enviroment 加载不同来源的 配置文件。

##### Spring 内置事件分发器

SpringApplicationRunnerListener 之前在 spring.factories 配置 EventPublishingRunListener, 负责事件发放

```yaml
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

```java
public EventPublishingRunListener(SpringApplication application, String[] args) {
    this.application = application;
    this.args = args;
    this.initialMulticaster = new SimpleApplicationEventMulticaster();
    for (ApplicationListener<?> listener : application.getListeners()) {
        this.initialMulticaster.addApplicationListener(listener);
    }
}
```

通过构造函数, 通过 application 获取ApplicationListener 事件监听对象。

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E event);
}

public abstract class ApplicationEvent extends EventObject {

    private static final long serialVersionUID = 7099057708183571937L;

    private final long timestamp;

    public ApplicationEvent(Object source) {
        super(source);
        this.timestamp = System.currentTimeMillis();
    }

    public final long getTimestamp() {
        return this.timestamp;
    }
}
```

可以看到事件监听者， 通过继承ApplicationListener 实现对应的事件监听， 监听事件类型 ApplicaiotnEvent, spring 通过 spring.factory 获取所有的 Listeners, **这个时候SpringFactory 还没有启动，Listener Bean 不会Env 初始化中进行**， ApplicationListener 通过监听参数 Class 类型进行事件分发，很巧妙，参考 ```org.springframework.context.event.SimpleApplicationEventMulticaster``` 实现方式。

下面我们看看 Env 初始化需要执行的 Listeners, event 类型 ```ApplicationEnvironmentPreparedEvent```, listeners 包含

- ConfigFileApplicationListener
- AnsiOutputApplicationListener
- LoggingApplicationListener
- ClasspathLoggingApplicationListener
- BackgroundPreinitializer
- DelegatingApplicationListener
- LocalApplicationListener
- FileEncodingApplicationListener

ConfigFileApplicationListener 注入监听数据

```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
    List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
    postProcessors.add(this);
    AnnotationAwareOrderComparator.sort(postProcessors);
    for (EnvironmentPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
    }
}
List<EnvironmentPostProcessor> loadPostProcessors() {
    return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class, getClass().getClassLoader());
}
```

通过SpringFactoryLoad 对应 EnvironmentPostProcessor 处理不同的文件加载load, load processor 包括:

- SystemEnvironmentPropertySourceEnvironmentPostProcessor
- SpringApplicationJsonEnvironmentPostProcessor
- CloudFoundryVcapEnvironmentPostProcessor
- ConfigFileApplicationListener
- DebugAgentEnvironmentPostProcessor
- SpringBootTestRandomPortEnvironmentPostProcessor

ConfigFileApplicationListener 添加配置文件 PropertySource 配置资源文件，加载配置方式

```java
protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    RandomValuePropertySource.addToEnvironment(environment);
    new Loader(environment, resourceLoader).load();
}

// loader 初始化
Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    this.environment = environment;
    this.placeholdersResolver = new PropertySourcesPlaceholdersResolver(this.environment);
    this.resourceLoader = (resourceLoader != null) ? resourceLoader : new DefaultResourceLoader(null);
    this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class,
            getClass().getClassLoader());
}

// spring.factories
# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader
```

上面loader 初始化，通过SpringFactory 获取 PropertySourceLoader 配置文件加载类目， 可以看到 properties xml， yaml yml, 配置文件加载。 ```org.springframework.boot.context.config.ConfigFileApplicationListener.Loader``` 加载配置文件，结合但是 profile， 相对复杂，专题沟通。 所以， 配置环境中扩展配置，可以 spring.factories 继承PropertySourceLoader 实现配置load, 或者自定义 Env Listener 插入对应 PropertySource 资源。
