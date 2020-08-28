---
title: SpringBoot Jar 运行方式
date: 2020-08-28 10:56:29
tags: ['Spring', 'SpringBoot']
categories: ['技术', 'Spring', 'SpringBoot']
---


本次想说说Spring boot构建运行方式。项目中， 一般SpringBoot提供的maven插件 ```spring-boot-maven-plugin``` 打包构建项目， 生成Jar,War构建，线上通过命令方式运行构建。Spring构建时候，会把对应的资源都打包进入输出构建中。 联想到以前的Client项目， Web项目，手动去拷贝部署一堆资源，方便了很多。之前做一个需求，自定义 ClassLoader 动态加载模块，惊奇发现，业务ClassLoader是 ```org.springframework.boot.loader.LaunchedURLClassLoader```，并不是我们熟知的 ```sun.misc.Launcher$AppClassLoader```, 显然 BootLoader 做了一些事情。

```java
org.springframework.boot.loader.LaunchedURLClassLoader@238e0d81
sun.misc.Launcher$AppClassLoader@42a57993
sun.misc.Launcher$ExtClassLoader@531d72ca
```

通过打印 ClassLoader 继承关系, 可以发现Spring 自定义了LaunchedUrlClassLoader -> UrlClassLoader(一般自定义Jar Rresource 通过继承这个ClassLoader去实现), 就是通过这个Loader, jar 运行时刻，都通加载 BOOT-INF 中lib 数据。

我们知道通过 ``` java -jar XXX.jar ``` 运行的java进程 系统默认loader继承关系是 ``` AppClassLoader -> ExtClassLoader -> BootClassLoader(null) ```， web 项目 tomcat 通过继承ClassLoader 实现项目资源隔离，SpringBoot 通过 LaunchedURLClassLoader 动态加载我们依赖的Lib 资源文件。

boot jar 资源目录结构

- \
  - BOOT-INF (springBootLoader 加载资源文件)
    - lib  (项目依赖资源文件)
    - classes (项目业务文件)
    - classpath.idx (依赖文件列表)
  - META-INF (maven 和 MANIFEST.MF 相关信息)
  - org.springframework.boot.loader (spring 自定义Loader 逻辑)

MAINFEST.MF 定义 java 启动信息, MainClass 替换Spring 提供的 JarLauncher 启动Class，代码可以通过SpringBoot-tools-loader 源码逻辑

```mf
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Implementation-Title: source-demo
Implementation-Version: 1.0-SNAPSHOT
Start-Class: org.yqj.source.demo.BootDemoApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.3.3.RELEASE
Created-By: Maven Jar Plugin 3.2.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

Launcher 之间的继承关系

- Launcher (mainThread 启动类, abstract)
  - ExecutableArchiveLauncher ()
    - JarLauncher ()
    - WarLauncher ()
  - PropertiesLauncher ()

项目通过配置 spring-boot-maven-plugin 参数设定打包方式, 项目一般打包成可运行Jar, JarLauncher Main 方法运行入口， 而不同构建 Launcher.launch() 运行启动上下文, ``` org.springframework.boot.loader.Launcher#launch(java.lang.String[]) ```

```java
protected void launch(String[] args) throws Exception {
    if (!isExploded()) {
        JarFile.registerUrlProtocolHandler();
    }
    ClassLoader classLoader = createClassLoader(getClassPathArchivesIterator());
    String jarMode = System.getProperty("jarmode");
    String launchClass = (jarMode != null && !jarMode.isEmpty()) ? JAR_MODE_LAUNCHER : getMainClass();
    launch(args, launchClass, classLoader);
}
```

1. registerUrlProtocolHandler 注册 ```java.protocol.handler.pkgs``` 使用 URLStreamHandler 动态Loader Url Jar目录文件
2. 创建ClassLoader，这里 LaunchedURLClassLoader spring 自定义ClassLoader, 加载了依赖的 Archives(依赖模块)
3. Jarmode vs Main-Class
   1. JarMode 模式获取所有 spring_factory 配置中 jarmode 方法。(for run 方式，感觉打包多模块定时任务)
   2. 业务 Main-Class 启动方法入口。

剩余的launcher, 设置线程ContextClassLoader, 保证业务代码资源正常加载。反射执行 MainClass。

JarLauncher WarLauncher 继承来自 ExecutableAchiveLauncher, Archive 中定义资源类型接口, 定义Jar 文件, NestJar 不同数据信息的解析。 不再赘述了。
特别Launcher, PropertiesLauncher 这个Launcher帮助我们在包外指定配置加载，通过启动参数方式。 
