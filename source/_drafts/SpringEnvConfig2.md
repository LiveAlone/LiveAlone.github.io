---
title: Spring Environment Config 2
date: 2020-09-01 01:38:10
tags:
  - Spring
  - SpringBoot
categories:
  - 技术
  - Spring
  - SpringBoot
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

不同Processor 配置文件加载方式 配置文件处理方式 TODO org.springframework.boot.context.config.ConfigFileApplicationListener#onApplicationEnvironmentPreparedEvent
postProcessEnvironment
