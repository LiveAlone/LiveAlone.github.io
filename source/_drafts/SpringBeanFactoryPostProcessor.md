---
title: bean factory processor 
date: 2020-09-03 20:56:43
tags:
  - Spring
  - SpringBoot
categories:
  - 技术
  - Spring
  - SpringBoot
---

SpringBeanFactoryProcessor 初始化BeanFactory, 读取不同的Source 来源的配置文件， 解析 AnnotationClass 注入 BeanDefinition。
通过 ```org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors()``` 方法，执行BeanFactoryProcessor。

首先执行 ```BeanDefinitionRegistryPostProcessor```，通过注释，注册更多的BeanDefinition, 在正式的BeanFactoryPostProcessor执行之前。

SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor

```java
// 注册 intereralCacheingMetadataReaderFactory bean definition
public static final String BEAN_NAME = "org.springframework.boot.autoconfigure."
        + "internalCachingMetadataReaderFactory";

private void register(BeanDefinitionRegistry registry) {
    BeanDefinition definition = BeanDefinitionBuilder
            .genericBeanDefinition(SharedMetadataReaderFactoryBean.class, SharedMetadataReaderFactoryBean::new)
            .getBeanDefinition();
    registry.registerBeanDefinition(BEAN_NAME, definition);
}
```

注册Bean 是类型 SharedMetadataReaderFactoryBean， 这是一个FactoryBean, bean 工厂， getObject 通过 ```ConcurrentReferenceCachingMetadataReaderFactory(classLoader)``` 创建只一个 ConfigurationClassPostProcessor。 具体Bean 解析过程后面说。

ConfigurationClassPostProcessor 执行 BeanFactoryPostProcessor 处理方式， ```org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions``` 解析BeanDefinition

1. 解析查找 ConfigBean, bootApplicaitonConfigBean，自定义 @SpringApplicationContext bean。
2. do-while 循环解析， @configuration 注解， 注入对应的 BeanDefinition 内容， 完成 BeanDefiniotnRegister 初始化。

PS: 这个时候 getBean 初始化Bean 对象， 并没有注册 BeanPostProcessor, 等待 Definition 解析完成以后， 初始化所有的 BeanPostProcessor。
之前的BeanFactoryProcessor 中部分监听，已经添加了 BeanPostProcessor, 解析 BeanDefinition 以后，重新获取 BeanPostProcessor, 进行初始化。
后面 init singleton， 保证数据的正确解析方式。

---

BeanDefinition 继承关系 ![BeanDefinition](/images/20200905/BeanDefinition.png)

BeanDefinition 继承制 AttributeAccessor， BeanMetadataElement 提供属性获取, MetadataObject对象原数据获取。提供实例化方式，sigleton, prototype。 bean 类型框架服务，业务类型等。 Bean 名称，属性，构造等。 依赖Bean 等信息。
AnnotatedBeanDefinition 注解方式配置Bean，支持获取 AnnotationMetadata 注解配置信息。

AbstractBeanDefinition 抽象Bean定义， 公用字段， 定义构造函数，构造方法，sourceObject 等。

- RootBeanDefinition 多BeanDefinition 支持合并， 多BeanDefinition 可能存在继承关系。
- ChildBeanDefinition 子类 BeanDefinition 类型，parentName 父类对象名称
- GenericBeanDefinition 通用，parentName.
  - AnnotatedGenericBeanDefinition 注解类型Bean 定义方式
  - ScannedGenericBeanDefinition 基于ASM ClassLoader
  - ConfigurationPropertiesValueObjectBeanDefinition 定义类型,  属性注入配置方式。

internalCachingMetadataReaderFactory BeanDefinition 构建方式, 可以看到通过构建 ```org.springframework.boot.autoconfigure.internalCachingMetadataReaderFactory SharedMetadataReaderFactoryBean.Class``` factoryBean进入Definition

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
  register(registry);
  configureConfigurationClassPostProcessor(registry);
}

private void register(BeanDefinitionRegistry registry) {
  BeanDefinition definition = BeanDefinitionBuilder
      .genericBeanDefinition(SharedMetadataReaderFactoryBean.class, SharedMetadataReaderFactoryBean::new)
      .getBeanDefinition();
  registry.registerBeanDefinition(BEAN_NAME, definition);
}

private void configureConfigurationClassPostProcessor(BeanDefinitionRegistry registry) {
  try {
    BeanDefinition definition = registry
        .getBeanDefinition(AnnotationConfigUtils.CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME);
    definition.getPropertyValues().add("metadataReaderFactory", new RuntimeBeanReference(BEAN_NAME));
  }
  catch (NoSuchBeanDefinitionException ex) {
  }
}
```

通过 internalCachingMetadataReaderFactory(SharedMetadataReaderFactoryBean) BeanDefinition 获取 实例Bean, ```org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean``` 获取实例化Bean 对象初始化内容。获取 ```org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton()``` 获取实例的Instance, 通过实例获取 Object ```org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance``` getBean 后面详细说明

获取ConfigClassPostProcessor 执行 processConfigBeanDefinitions 注册BeanDefinition。 通过Class 名称可以了解， 解析@Configuration 注解Class 类，加载配置。 ```org.springframework.context.annotation.ConfigurationClassParser``` 创建ConfigurationParse 转换类型。 通过 ```ConfigurationClassBeanDefinitionReader``` 读取ClassConfig 定义BeanDefinition。

org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass 处理 Import ComponeScan 不同注解Bean 扫描加载。

```java
protected final SourceClass doProcessConfigurationClass(
    ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
    throws IOException {

  if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
    // Recursively process any member (nested) classes first
    processMemberClasses(configClass, sourceClass, filter);
  }

  // Process any @PropertySource annotations
  for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
      sourceClass.getMetadata(), PropertySources.class,
      org.springframework.context.annotation.PropertySource.class)) {
    if (this.environment instanceof ConfigurableEnvironment) {
      processPropertySource(propertySource);
    }
    else {
      logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
          "]. Reason: Environment must implement ConfigurableEnvironment");
    }
  }

  // Process any @ComponentScan annotations
  Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
      sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
  if (!componentScans.isEmpty() &&
      !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
    for (AnnotationAttributes componentScan : componentScans) {
      // The config class is annotated with @ComponentScan -> perform the scan immediately
      Set<BeanDefinitionHolder> scannedBeanDefinitions =
          this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
      // Check the set of scanned definitions for any further config classes and parse recursively if needed
      for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
        BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
        if (bdCand == null) {
          bdCand = holder.getBeanDefinition();
        }
        if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
          parse(bdCand.getBeanClassName(), holder.getBeanName());
        }
      }
    }
  }

  // Process any @Import annotations
  processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

  // Process any @ImportResource annotations
  AnnotationAttributes importResource =
      AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
  if (importResource != null) {
    String[] resources = importResource.getStringArray("locations");
    Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
    for (String resource : resources) {
      String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
      configClass.addImportedResource(resolvedResource, readerClass);
    }
  }

  // Process individual @Bean methods
  Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
  for (MethodMetadata methodMetadata : beanMethods) {
    configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
  }

  // Process default methods on interfaces
  processInterfaces(configClass, sourceClass);

  // Process superclass, if any
  if (sourceClass.getMetadata().hasSuperClass()) {
    String superclass = sourceClass.getMetadata().getSuperClassName();
    if (superclass != null && !superclass.startsWith("java") &&
        !this.knownSuperclasses.containsKey(superclass)) {
      this.knownSuperclasses.put(superclass, configClass);
      // Superclass found, return its annotation metadata and recurse
      return sourceClass.getSuperClass();
    }
  }

  // No superclass -> processing is complete
  return null;
}
```

转换 ClassConfig 获取BeanConfigutration 配置内容， BeanFactory 转换Bean 类型。

---

BeanFactory getBean 通过BeanDefinition 初始化Bean 实例配置。 ```org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean``` 作为入口解析Spring 获取 Bean 过程。

- transformedBeanName beanName转换，两件事情 1. & Spring内置Bean名称获取转换 2. 通过aliasName 依赖名称转换。
- 获取ParentBeanFactory 父级工厂， 和Class 加载机制类似， 优先父级 GetBean, 存在返回。
- 获取 BeanNameDefinition 定义类型
- 获取 BeanDefinition 依赖 DependOn, 1. 注册依赖Bean  2. getBean 方式实例化依赖bean 类型。
- 创建当前 Bean 实例， 通过 BeanDefinition 判断 Bean 类型
  - Singleton ProtoType 通过 GetBean 获取实例Bean 类型
  - 用户支持自定义 Scope 通过Scope 自定义 Bean 的作用域范围

通过监控```org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean()``` Bean初始化过程， BeanPostProcessor 不同的继承实现接口, ```DestructionAwareBeanPostProcessor, InstantiationAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor``` 不同初始化接口， 返回对象类型。

对象注入， init 初始化 等等 不同的处理类型， 都是通过 Processor, 具体感觉使用查看， 毕竟分析也记不住。
