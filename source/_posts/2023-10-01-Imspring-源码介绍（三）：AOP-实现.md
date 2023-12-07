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

## 对 Joinpoint 做 Advice

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

**Advice 表示对 Joinpoint 执行的某些操作**。

为了更好地理解这两个概念，我再举一个例子：当我们对用户进行新增操作前，需要进行权限校验。其中，调用 user.save() 的动作就是一个的 Joinpoint，权限校验就是一个 Advice，即对 Joinpoint（新增用户的动作）做 Advice（权限校验）。

**在 spring-aop 中，Joinpoint 对象持有了一条 Advice chain ，调用 Joinpoint 的 proceed() 方法将采用责任链的形式依次执行各个 Advice**（注意，Advice 的执行可以互相嵌套，不是单纯的先后顺序）。

JDK 动态代理使用的 InvocationHandler、cglib 使用的 MethodInterceptor，在抽象概念上可以算是 Advice（即使它们没有继承Advice）。但是它们没有所谓的 Advice chain，一个 Joinpoint 一般只能分配一个 Advice，当需要使用多个 Advice 时，需要像套娃一样层层代理。

在 spring-aop 中，主要使用 Advice 的子接口——MethodInterceptor。

## 其他的几个概念

在 spring-aop 中，还会使用到其他的概念，例如 Advice Filter、Advisor、Pointcut、Aspect 等。

### Advice Filter

**Advice Filter 一般和 Advice 绑定，它用来告诉我们，Advice 是否作用于指定的 Joinpoint**，如果 true，则将 Advice 加入到当前 Joinpoint 的 Advice chain，如果为 false，则不加入。

在 spring-aop 中，常用的 Advice Filter 包括 ClassFilter 和 MethodMatcher，前者过滤的是类，后者过滤的是方法。

### Pointcut

**Pointcut是 AspectJ 的组件，它是一种 Advice Filter**。

在 spring-aop 中，Pointcut = ClassFilter + MethodMatcher 。

### Advisor

Advisor是 spring-aop 原创的组件，**一个 Advisor = 一个 Advice Filter + 一个 Advice**。

在 spring-aop 中，主要有两种 Advisor：IntroductionAdvisor 和 PointcutAdvisor。前者为 ClassFilter + Advice，后者为 Pointcut + Advice。

### Aspect

Aspect 也是 AspectJ 的组件，一组同类的 PointcutAdvisor 的集合就是一个 Aspect。

## 示例

在下面代码中，`printRequest` 和 `printResponse` 都是 Advice，`genericPointCut` 是 Pointcut，`printRequest + genericPointCut` 是 PointcutAdvisor，`UserServiceAspect` 是 Aspect。

```java
@Aspect
public class UserServiceAspect {
        
    @Pointcut("execution(* cn.zzs.spring.UserService+.*(..)))")
    public void genericPointCut() {
    }
    
    @Before(value = "genericPointCut()")
    public void printRequest(JoinPoint joinPoint) throws InterruptedException {
        //······
    }  
    
    @After(value = "genericPointCut()")
    public void printResponse(JoinPoint joinPoint) throws InterruptedException {
        //······;
    }  
}
```

# 整体流程

![SpringAOP时序图.png](SpringAOP时序图.png)

Imspring-aop 实现 AOP 的整体流程图如上所示，这个流程和 Spring 实现 AOP 的流程基本是一致的。为了让大家更清楚这个流程，先介绍几个核心类：

* **AnnotationAwareAspectJAutoProxyCreator**。这个类实现了 `InstantiationAwareBeanPostProcessor` 接口，因此也间接实现了 `BeanPostProcessor `接口。在 Instantiation 前置处理器里面查找所有切面和对应的增强通知，在 Initialization 后置处理器里面是否有适用于当前 bean 的通知并创建代理对象，这就是 AOP 的原理。
* **AbstractAspectJAdvice**：Advice 抽象类，实现了 Advice 的子接口 `MethodInterceptor`，本质上是一个拦截器，子类实现是我们常说的 `BeforeAdvice`、`AfterAdvice` 等。
* **AspectJExpressionPointcutAdvisor**：Advisor 实现类。AspectJExpressionPointcutAdvisor = 上面的xxxAdvice + AspectJExpressionPointcut。
* **AspectJExpressionPointcut**：Pointcut 实现类。
* **ReflectiveMethodInvocation**：可以理解为 **Joinpoint + Advisor List**。Joinpoint 简单来说就是哪个地方需要被增强。ReflectiveMethodInvocation 里面保存了原始的 Bean 实例和要被增强的 Method，这两个加起来就是一个 Joinpoint。Advisor List 是从代理对象里面获取的。
* **ProxyFactory**：用来创建 AOP 代理对象的工厂类，根据类是否实现接口，自动选择 `JdkDynamicAopProxy` 或 `CglibAopProxy` 创建代理对象。

上面的整体流程可以概括为：

1. ApplicationContext 容器启动，在 Bean 实例化之前，进入 `AnnotationAwareAspectJAutoProxyCreator` 的 Instantiation 前置处理。这一步会扫描容器中所有 @Aspect 切面类，为类中的每个 Advice 方法（包含`@Before`、`@After`等注解）创建一个 `AspectJExpressionPointcutAdvisor`，然后把每个 Aspect 切面对应的 Advisor List 缓存到  `advisorsCache`。

2. Bean 进入初始化阶段，进入 `AnnotationAwareAspectJAutoProxyCreator` 的 Initialization 后置处理。这一步从 `advisorsCache` 获取所有的 Advisor List，找出和当前 Bean 匹配的所有 Advisor，实现方法是挨个使用 Advisor 里面的 Pointcut 和当前的 Bean 进行匹配。如果匹配的 Advisor 为空，就返回原始 Bean；否则就为当前 Bean 和 Advisor 创建 JDK 或者 CGLIB 代理对象。至此 ApplicationContext 容器启动完成。
3. 用户调用目标方法，实际上会调用上一步创建的代理对象。如果代理对象的 Advisor 为空（这里为空是指 Advice 类型为 EMPTY_ADVICE），就通过反射直接调用原始 Bean 的目标方法；否则就原始 Bean 和 Advisor 等信息封装成 `ReflectiveMethodInvocation`，执行 `proceed()`。
4.  `proceed()`方法里面，对原始 Bean 和 Advice 方法进行递归链式调用。这里会调用 `AbstractAspectJAdvice#invokeAdviceMethod`，真正去执行 Advice 方法。这就是 Spring AOP 的整体流程。
