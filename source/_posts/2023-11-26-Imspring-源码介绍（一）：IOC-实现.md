---
title: Imspring 源码介绍（一）：IOC 实现
date: 2023-09-20 20:39:19
categories:
- Spring
tags:
- Spring
---

> Imspring 源码介绍（一）：IOC 实现

<!--more-->

# 概述

Imspring-core 模块围绕 ApplicationContext 和 Beanfactory 两个 Bean 工厂实现了 Spring 的核心 IOC 功能。整个模块涉及的核心知识点如下：

* **Bean 的全生命周期。（重点）**
* **ApplicationContext 和 BeanFactory 的初始化流程。（重点）**
* 包名扫描和加载。
* 依赖注入，以及三级缓存解决循环依赖问题。
* 使用 BeanFactoryPostProcessor 实现定制化 Bean 工厂（Configuration 配置类的基础）
* 使用 BeanPostProcessor 实现定制化 Bean。（AOP的基础）
* 配置文件加载和环境切换。

为了让大家更加清楚整个流程，先介绍下 Spring 的一些基础概念和组件。

## 基础概念

### ApplicationContext 和 BeanFactory

这两个是 Spring 的核心概念。通俗易懂地解释，BeanFactory 是一个工厂接口，提供 IOC 的基本功能，包括读取 Bean 的 XML 配置文件，管理 Bean 的加载和实例化，维护 Bean 之间的依赖关系，管理 Bean 的生命周期等。这个接口大概长这样

```java
public interface BeanFactory {
    // 根据bean名称获取bean
    Object getBean(String name) throws BeansException;

    // 根据bean名称和bean类型获取bean
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    
    // 根据bean类型获取bean，使用无参构造函数
    <T> T getBean(Class<T> requiredType) throws BeansException;
    
    // 根据bean类型和指定参数获取bean，使用有参构造函数
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
    ......
}
```

这个接口的核心方法就是使用 `getBean()` 方法获取各种 Bean。我们并不需要关心这些 Bean 是怎么构造出来，只需要直接从容器取就可以了，这也符合工厂模式的理念。ApplicationContext 继承了 BeanFactory 接口，具备 BeanFactory 接口的全部功能，而且提供更丰富的特性。

```java
// ApplicationContext 继承了 ListableBeanFactory 接口，ListableBeanFactory 继承了 BeanFactory 接口
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
    // 设置多环境和加载配置文件
    void setEnvironment(ConfigurableEnvironment environment);
    
    // 设置事件监听器
    void addApplicationListener(ApplicationListener<?> listener);
    
    // ApplicationContext 的核心方法，完成初始化流程
    void refresh() throws BeansException, IllegalStateException;
    
    // 关闭 ApplicationContext 后进行资源清理
    void close();
    
    // 获取 BeanFactory 实例
    ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
    ......
}
```

由上面看出 ApplicationContext 持有 BeanFactory 的实例，所以具备 BeanFactory 的全部功能，并提供更完整的框架功能。两者的区别可以用下面表格来表示。

| 功能            | BeanFactory                                                  | ApplicationContext                                           |
| :-------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Bean 实例化时机 | BeanFactroy 采用的是延迟加载形式来注入 Bean 的，即只有在使用到某个 Bean 时（调用`getBean()`），才对该 Bean 进行加载实例化。 | ApplicationContext 采用预加载的方式注入 Bean，在启动的时候就会调用 `refresh()` 预加载所有单例 Bean，完成整个环境的初始化。 |
| Bean 扩展       | BeanPostProcessor 和 BeanFactoryPostProcessor 手动注册       | BeanPostProcessor 和 BeanFactoryPostProcessor 自动注册       |
| 上下文          | 从 XML 文件中读取上下文（XmlBeanFactory）                    | 支持通过 ClassPath、FileSystem、Web、Annotation 等方式读取上下文，并且可以设置多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次。 |
| 国际化          | /                                                            | 支持国际化 i18n（ MessageSource）                            |
| 访问资源        | /                                                            | 提供统一的资源文件访问方式，加载多种不同类型的资源（ResourceLoader） |
| 环境切换        | /                                                            | 支持多环境切换，加载多个配置文件（Environment）              |
| 事件机制        | /                                                            | 强大的事件监听机制，当 ApplicationContext 中发布一个事件时，所有扩展ApplicationListener 的 Bean 都将接受到这个事件，并进行相应的处理。（ApplicationEvent 和 ApplicationListener） |

总结：

* BeanFactory 是 Spring 框架的基础设施，面向 Spring 本身。
* ApplicationContext 提供更多面向实际开发的功能，几乎所有场合都可以直接使用，面向使用 Spring 的开发者。日常使用建议直接使用 ApplicationContext。

### Bean 生命周期

![image-20231128190205771](image-20231128190205771.png)

**Bean 的生命周期是指一个对象在 Spring 容器里面从创建到被销毁的整个过程**。这里引用一个网上非常经典的图，这个图基本涵盖了 Bean 从初始化到被销毁的整个生命周期。为了方便记忆，可以分为四个阶段：

1. 实例化（Instantiation）
2. 属性赋值（Populate）
3. 初始化（Initialization）
4. 销毁（Destruction）

![image-20231128190523202](image-20231128190523202.png)

Imspring 项目严格遵循以上 Bean 生命周期的顺序，最后为了简化跳过了销毁阶段。

### Resource 和 ResourceLoader

Spring 用 Resource 接口抽象所有的底层资源，包括 File、ClassPath、URL 等。Spring 通过 Resource 接口将对物理资源的访问方式统一为  URL 格式和 Ant 风格带通配符的资源地址。

```java
public interface Resource extends InputStreamSource {
	// 判断某个资源是否以物理形式存在
	boolean exists();
	
	// 获取资源对象的URL，不能表示为URL就抛异常
	URL getURL() throws IOException;
	
	// 获取资源的File表示对象，不能表示为File就抛异常
	File getFile() throws IOException;
	......
}
```

Spring 提供多种 Resource 的实现类，我们可以直接使用。常用的有：

* `FileSystemResource`：针对 java.io.File 的 Resource 实现类。可以消除操作系统底层差异，对不同的操作系统使用同一的 API 来访问。
* `ClassPathResource`：访问类加载路径下的资源，对于 WEB 应用可以自动搜索位于 WEB-INF/classes 下的资源文件，是 Spring 中是非常常用的一种资源类型。
* `UrlResource`：java.net.URL 的 Resource 实现类。可以访问网络资源，也可以访问本地资源。支持 http、https、file、ftp、jar 等协议。

ResourceLoader 接口是 Resource 的加载器，用于快速加载一个 Resource 资源对象。Spring 中的默认实现是 `DefaultResourceLoader`。

```java
public interface ResourceLoader {
 	// 根据路径信息返回一个对应的 Resource 资源实例
    Resource getResource(String location);
 
 	// 返回该 ResourceLoader 所使用的 ClassLoader
    @Nullable
    ClassLoader getClassLoader();
}
```

**简单来说，Spring 通过 Resource 和 ResourceLoader 对外提供统一的资源访问方式**。

### BeanDefinition 和 BeanDefinitionRegistry

BeanDefinition 是用来描述 Bean 的元数据信息。除了我们熟悉的 Bean 的 Class 对象之外，还保存了 bean是否是单例、bean是否在容器中是懒加载、bean在容器中的名字等信息。Spring 依赖这些信息来完成一个 Bean 的实例化。

**简单来说，Spring 将 Bean 的定义和 Bean 的实例分开处理。首先将类文件 Class 解析为 BeanDefinition，然后再根据 BeanDefinition 的定义，在合适的时机（例如延迟创建）创建正确类型的 Bean 实例（单例或原型）。**这个 Bean 实例就是我们日常使用的 Bean。

BeanDefinitionRegistry 可以理解为 BeanDefinition 的容器接口，完成 BeanDefinition 的注册、移除和删除等功能。Spring 的 BeanFactory 默认实现类 `DefaultListableBeanFactory` 实现了 BeanDefinitionRegistry 接口，使用 `beanDefinitionMap` 集合来存放所有的 BeanDefinition。

> Spring 中命名为 `XXXRegistry` 的接口或者类，在 `Spring` 中通常扮演注册中心的职责，维护管理对应的一组实例。

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
		
	// beanDefinitionMap 存放所有 Bean 的元数据信息
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
}
```

### SingletonBeanRegistry

上面谈到 Spring 中 Bean 的定义和 Bean 的实例是分开处理。我们日常使用 Spring 的时候，99% 的场景都是使用 Spring Bean 的单例实例。SingletonBeanRegistry 相当于 Bean 单例实例的容器接口，提供了统一访问单例 Bean 的功能。

Spring 中 SingletonBeanRegistry 的 默认实现类是 `DefaultSingletonBeanRegistry`。这个类非常非常的重要，可以认为是 Spring IOC 容器的核心内容，大名鼎鼎的三级缓存就放在这里面。除此之外还有非常多的缓存用于提高性能，并且做的事情也很多，比如解决 Bean 的循环依赖问题。

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
	......
    // 一级缓存，存放完全初始化好的 bean，从该缓存中取出的 bean 可以直接使用，我们日常使用的 Bean 就是从这里面取
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 二级缓存，提前曝光的单例对象的cache，存放原始的 bean 对象（尚未填充属性），用于解决循环依赖
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

    // 三级缓存，存放 bean 工厂对象，这个对象其实是一个函数式接口，用来解决 AOP 的循环依赖
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    ......
    
    // 注册单例对象
    public void registerSingleton(String beanName, Object singletonObject) {...}
    
    // 获取单例对象，使用三级缓存解决循环依赖的核心
    public Object getSingleton(String beanName, boolean allowEarlyReference) {...}
    ......
}
```



### BeanPostProcessor 和 BeanFactoryPostProcessor

### Environment、Profile 和 PropertyResolver

# 代码分析

<img src="image-20231127202003918.png" width="60%" height="60%">

整个项目结构如上所示：

## 1、资源扫描



## 2、环境初始化



## 3、创建 Bean 实例



## 4、属性填充



## 5、初始化 Bean



## 6、环境初始化完成
