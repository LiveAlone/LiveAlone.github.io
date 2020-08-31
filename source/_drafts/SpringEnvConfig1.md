---
title: Spring Environment Config 1
date: 2020-08-30 22:16:00
tags: ["Spring", "SpringBoot"]
categories: ["技术", "Spring", "SpringBoot"]
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

TODO env
