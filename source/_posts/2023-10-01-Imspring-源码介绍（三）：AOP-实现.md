---
title: Imspring 源码介绍（三）：AOP 实现
date: 2023-10-01 20:42:27
categories:
- Spring
tags:
- Spring
---

> Imspring 源码介绍（三）：AOP 实现

<!--more-->

imspring-aop 的作用是为满足条件的 Bean 创建一个代理对象，本质是通过 BeanPostProcessor 来实现，如果不理解原理可以参考上一篇文章 [Imspring 源码介绍（二）：IOC 实现]()。在介绍 imspring-aop 之前先介绍一下 AOP 涉及的一些概念。

# 基本概念

说到 spring-aop，我们经常会提到 **Joinpoint**、**Advice**、**Pointcut**、**Aspect**、**Advisor** 等等概念，它们都是抽象出来的标准，有的来自 aopalliance，有的来自 AspectJ，也有的是 spring-aop 原创。这些概念构成 spring-aop 设计图的基础。理解它们非常难，一个原因是网上能讲清楚的不多，第二个原因是这些组件本身抽象得不够直观（spring 官网承认了这一点）。

## 对Joinpoint做Advice

在 spring-aop 的包中内嵌了 aopalliance 的包（aopalliance 就是一个制定 AOP 标准的联盟、组织），这个包是 AOP 联盟提供的一套标准，提供了 AOP 一些通用的组件。完整的 aopalliance 包，除了 aop 和 intercept，还包括了 instrument 和 reflect，后面这两个部分 spring-aop 没有引入，这里就不说了。aop 和 intercept 的包结构大致如下。

```
└─org
    └─aopalliance
        ├─aop
        │      Advice.class
        │      AspectException.class
        │
        └─intercept
                ConstructorInterceptor.class
                ConstructorInvocation.class
                Interceptor.class
                Invocation.class
                Joinpoint.class
                MethodInterceptor.class
                MethodInvocation.class
```

使用 UML 表示以上类的关系如下所示。可以看到，这主要包含两个部分：**Joinpoint **和 **Advice**（**这是 AOP 最核心的两个概念**）。

![image-20231204180055949](image-20231204180055949.png)

### Joinpoint

**Joinpoint 表示对某个方法（构造方法或成员方法）或属性的调用**。

例如，我调用了 user.save() 方法，这个调用动作就属于一个 Joinpoint。Joinpoint 是一个“动态”的概念，Field、Method、Constructor 等对象是它的静态部分。

如上图所示，**Joinpoint 是 Advice 操作的对象**。

在 spring-aop 中，主要使用 Joinpoint 的子接口——MethodInvocation。

### Advice

## 其他的几个概念

# 项目结构

<img src="image-20231204163001538.png" width="60%" height="60%">

# 整体流程



# 代码分析



# 总结
