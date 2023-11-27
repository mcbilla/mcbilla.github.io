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

### Resource

### BeanDefinition

### SingletonBeanRegistry

### BeanPostProcessor 和 BeanFactoryPostProcessor

### Environment、Profile 和 PropertyResolver

# 代码分析

![image-20231127202003918](2023-11-26-Imspring-源码介绍（一）：IOC-实现/image-20231127202003918.png)

整个项目结构如上所示：

## 1、资源扫描



## 2、环境初始化



## 3、创建 Bean 实例



## 4、属性填充



## 5、初始化 Bean



## 6、环境初始化完成
