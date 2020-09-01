---
title: Spring Environment Config 1
tags:
  - Spring
  - SpringBoot
categories:
  - 技术
  - Spring
  - SpringBoot
date: 2020-08-30 22:16:00
---


上一篇大概了解了 SpringApplication.Run() 运行初始化过程， 这次了解 EnviromentConfig 配置初始化。

#### propertySource 资源管理

application 初始化时候， 创建DefaultApplicationArguments args 启动参数分装， 内部会对 args 分装PropertySource。

```java
public abstract class PropertySource<T> {
    protected final String name; // source 配置资源的名称
    protected final T source;   // 资源配置类型, properties map 等等, MustableProperty 管理 name - sourceProperty 映射关系。
}
```

下面展示PropertySource 集成关系图片, 我们解析不同继承链实现方式。
![SourceProperty](/images/20200831/PropertySource.png)

- StubPropertySource source 为null 站位资源类型
  - ComparisonPropertySource 为了集合比较 Source 资源类型
- FilteredPropertySource PropertSource 封装， 包含一个 FilsterSet, 过滤PropertySource 中部分属性不可见
- ConfigurationPropertySourcesPropertySource 对 ConfigurationPropertySourcesPropertySource list 封装， 通过这个集成多souce 来源
- EnumerablePropertySource 我们知道 source 资源 T 抽象类型， Enum 定义 sourceName 可枚举比较
  - CompositePropertySource 多source来源组合封装方式
  - DynamicValuesPropertySource source 抽象类型 ```Map<String, Supplier<Object>>``` 通过Supplier customer 自定提供对象逻辑
  - CommandLinePropertySource，SimpleCommandLinePropertySource 这里 命令行参数 a=b,b=c 装换Source 格式化数据结构
  - MapPropertySource source 定义类型 ```Map<String, Object>```
    - SystemEnvironmentPropertySource 系统配置Property 做一些参数名格式转换
    - PropertiesPropertySource， JsonPropertySource， spring source 格式管理， 后面详细说明。

#### Environment 配置环境上下文Context

Spring通过 Enviroment 管理启动环境上下文配置信息，profile 激活环境配置，不同Source Config yaml 配置来源。
Environment 继承关系关系图片 ![PropertyResolver](/images/20200831/PropertyResolver.png)

相关接口实现方式

- PropertyResolver 底层通过 PropertySource 支持 Property 属性获取
  - Environment 不同 profiles，理解激活环境，不同配置。@Profile 注解提供支持环境
    - ConfigurableEnvironment 支持配置换环境注入， profile 动态激活，MutablePropertySources 底层依赖，支持 merge(env) 多ConfigurableEnvironment 组合
      - ConfigurableReactiveWebEnvironment web 环境env
  - ConfigurablePropertyResolver ConversionService 用户定制化 source 对象转换

实现类型

- AbstractEnvironment 抽象env 实现对象
  - StandardEnvironment MutablePropertySource 多资源属性配置
  - StandardReactiveWebEnvironment web reactive

笔记环境创建StandardEnvironment对象, SpringApplication 初始化创建 Env 过程

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
  // Create and configure the environment
  ConfigurableEnvironment environment = getOrCreateEnvironment(); // 当前App 类型， 创建对应的Env
  configureEnvironment(environment, applicationArguments.getSourceArgs()); // SpringApplication 添加添加环境启动一些 Env 配置信息
  ConfigurationPropertySources.attach(environment);   // 移除 attach properties 后面说
  listeners.environmentPrepared(environment);   // 通过 applicationListener env 加载对应的EnvProcessor
  bindToSpringApplication(environment); // 额外绑定操作等等
  if (!this.isCustomEnvironment) {
    environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
        deduceEnvironmentClass());
  }
  ConfigurationPropertySources.attach(environment);
  return environment;
}
```

PropertySourcesPropertyResolver env 依赖底层配置Prperty Resolver 处理工具。 追踪 getProperty 处理方式。

```java
@Nullable
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
  if (this.propertySources != null) {
    for (PropertySource<?> propertySource : this.propertySources) { // 遍历所有的PropertySouce 资源列表内容，之前说过
      if (logger.isTraceEnabled()) {
        logger.trace("Searching for key '" + key + "' in PropertySource '" +
            propertySource.getName() + "'");
      }
      Object value = propertySource.getProperty(key); // 查找包含了 key 的 PropertySource value
      if (value != null) {
        if (resolveNestedPlaceholders && value instanceof String) {
          value = resolveNestedPlaceholders((String) value);
        }
        logKeyFound(key, propertySource, value);
        return convertValueIfNecessary(value, targetValueType); // ConversionService objectValue 进行数据转换
      }
    }
  }
  if (logger.isTraceEnabled()) {
    logger.trace("Could not find key '" + key + "' in any property source");
  }
  return null;
}
```

#### ConfigurablePropertyResolver ConfigurablePropertyResolver

ConfigurablePropertyResolver 提供用户定制化的Property 属性对象转换方式， 通过 ConversionService定制化
![ConversionService](/images/20200831/ConversionService.png)

- ConversionService 类型转换服务，sourceClassType to targetClassType，通过convert TargetType 类型转换
  - CompositeConversionService 多ConversionService 组合转换方式
  - ConfigurableConversionService interface 继承 支持 Converter 类型转换配置 config
  - GenericConversionService 定义通用转换类型，分装不同的Converter 转换器
    - FormattingConversionService 格式化Format 类型转换方式
    - TypeConverterConversionService type 类型转换
    - DefaultConversionService 默认注册不同 Conversion

GenericConverter 通用Converter 单一类型转换方式， 通过ConvertiblePair 提供转换类型信息。

#### configure 配置 Env

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
  if (this.addConversionService) {
    ConversionService conversionService = ApplicationConversionService.getSharedInstance();
    environment.setConversionService((ConfigurableConversionService) conversionService);  // env 注册 ConversionService
  }
  configurePropertySources(environment, args);
  configureProfiles(environment, args);
}
```

1. ApplicationConversionService 注册配置String Number 不同类型转换器执行。 StanderEnv 添加类型转换器。
2. args 作为一种 PropertySource, 添加进入 Env 数据源中，我们知道 args 是优先级最高， source.addFirst
3. 添加env 环境的 Profile 上下文，支持激活上下文配置方式。

```ConfigurationPropertySources.attach(environment);``` configurationProperties 添加 PropertySource, 事件注入, 后面Listener 通过监听事件注入 (这个下一篇再讲)

#### bindingToSpringApplication binder 绑定SpringApplication 属性变量

```java
protected void bindToSpringApplication(ConfigurableEnvironment environment) {
  try {
    Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));  // spring.main 配置内容 绑定我们 SpringApplication 中
  }
  catch (Exception ex) {
    throw new IllegalStateException("Cannot bind to SpringApplication", ex);
  }
}
```

Binder 对象属性绑定这， prefix "spaing.main" 配置绑定进入 Bindable 封装Instance 对象, 这里先不细说， 关注流程。

接下来执行, 上面Application 设置 spring.main 参数注入， 可能影响env 对象类型， 比如 spring.main 指定 web 类型转换 reactive, 需要转换 env 进入 ReactiveEnviroment 类型。

```java
if (!this.isCustomEnvironment) {
  environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
      deduceEnvironmentClass());
}
```

beanInfo加载判断, spring.beaninfo.ignore，lazy load 变得 代价较高

到这里 Enviroment 加载已经完成了， 后面是 ApplicationContext 加载过程， 第二章介绍 ApplicationListener env 事件监听， 底层后面专题沟通。
