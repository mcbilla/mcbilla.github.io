---
title: Imspring 源码介绍（二）：IOC 实现
date: 2023-09-20 20:39:19
categories:
- Spring
tags:
- Spring
---

> Imspring 源码介绍（二）：IOC 实现

<!--more-->

# 项目结构

<img src="image-20231127202003918.png" width="60%" height="60%">

Imspring-core 项目结构如上所示，最外层是 BeanFactory 和 ApplicationContext 接口及其实现类。其中

* `ConfigurableXXX` 开头的接口相当于扩展了原接口的功能。
* `AbstractXXX` 开头的类是抽象类，实现了 `ConfigurableXXX` 接口。和 Spring 的设计理念类似，抽象类实现了绝大部分的功能逻辑，把差异性的功能放到子类去实现。
* `DefaultListableBeanFactory` 是 `BeanFactory` 接口的最终实现类，`AnnotationConfigApplicationContext` 是 `ApplicationContext` 接口的最终实现类。

目录细分功能如下：

* annotation：存放所有的注解。
* common：从 Spring 拷贝过来的一些常用类，该目录可以忽略。
* context：存放 Spring 的核心接口，包括 BeanPostProcessor、InitializingBean 等。
* env：存放 Environment 的相关内容。
* exception：Spring 异常类。
* resource：存放 Resource 的相关内容。
* support：核心接口的一些实现类。
* utils：工具类目录。

# 整体流程

![ApplicationContext初始化流程.drawio](ApplicationContext初始化流程.drawio.png)

Applicationcontext 的整体启动流程图如上所示，其中 **`AnnotationConfigApplicationContext` 是 Spring 中使用 Annotation 配置方式启动容器的入口，也是我们学习项目代码的入口**。这个类的继承结构如下所示：

![AnnotationConfigApplicationContext](AnnotationConfigApplicationContext.png)

由此看出 ApplicationContext 接口直接继承了 BeanFactory 接口，所以说 ApplicationContext 是对 BeanFactory 的功能进行了增强扩展。

# 代码分析

## 1、容器启动

使用 `AnnotationConfigApplicationContext` 启动 Spring 容器，一般我们的启动代码会这样写。

```java
// 第一种方式，加载 @Configuration 配置文件启动 Spring 容器
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ServiceConfig.class);
MyService bean = (MyService) context.getBean("myService");

// 第二种方式，扫描类路径启动 Spring 容器
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext("com.mcb.study.spring.boot.core.test");
MyService bean = (MyService) context.getBean("myService");
```

AnnotationConfigApplicationContext 是调用构造函数的时候会初始化两个非常重要的属性，分别对应以上的两种启动方式。

* **AnnotatedBeanDefinitionReader**：对应第一种使用 @Configuaration 配置类来启动容器的方式。读取配置类的 @Bean、@Import 等注解，将这些类解析成 BeanDefinition，注册到 Spring 容器中。
* **ClassPathBeanDefinitionScanner**：对应第二种扫描指定路径来启动容器的方式。扫描指定路径下的@Component 类，将这些类解析成 BeanDefinition，注册到 Spring 容器中。

```java
public class AnnotationConfigApplicationContext extends AbstractApplicationContext {

    /**
     * 读取@Configuration配置类的信息，将这些类解析成BeanDefinition，注册到Spring容器中
     */
    private final AnnotatedBeanDefinitionReader reader;

    /**
     * 扫描指定路径下的@Component类，将这些类解析成BeanDefinition，注册到Spring容器中
     */
    private final ClassPathBeanDefinitionScanner scanner;

    /**
     * 构造DefaultListableBeanFactory、AnnotatedBeanDefinitionReader、ClassPathBeanDefinitionScanner
     * 其中DefaultListableBeanFactory在父类里面构造
     */
    public AnnotationConfigApplicationContext() {
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }

    /**
     * 扫描类路径下的文件，注册为BeanDefinition
     * @param basePackages
     */
    public AnnotationConfigApplicationContext(String scanPackage) {
        this();
        scan(scanPackage);
        refresh();
    }
  
    /**
     * 扫描配置类，注册为BeanDefinition
     * @param componentClasses
     */
    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        this();
        register(componentClasses);
        refresh();
    }

    @Override
    public void register(Class<?>... componentClasses) {
        Assert.notEmpty(componentClasses, "At least one component class must be specified");
        this.reader.register(componentClasses);
    }

    @Override
    public void scan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        this.scanner.scan(basePackages);
    }

}
```

不管是上面哪种启动方式，目的都是为了把类文件加载成 Beandefinition，最后都会调用到 `refresh()` å这个方法。这个方法是 ApplicationContext 的核心方法。**ApplicationContext 相较于 BeanFactory 的增强功能，例如自动注册 BeanPostProcessor、提前实例化所有 Bean 等，都在 `refresh()` 这个方法里面实现**。这个方法下面再细说。

## 2、资源扫描

两种扫描方式思路类似，以第二种扫描指定路径来启动容器的方式为例，ClassPathBeanDefinitionScanner 完成的工作如下：

1. 调用 ClassPathBeanDefinitionScanner 构造函数的时候需要 BeanDefinitionRegistry 类型的参数，从上面的继承接口看出 AnnotationConfigApplicationContext 就实现了 BeanDefinitionRegistry 接口，所以直接把 AnnotationConfigApplicationContext 本身作为参数传递。`this.scanner = new ClassPathBeanDefinitionScanner(this);`。

2. 使用默认资源加载器 DefaultResourceLoader 扫描指定的路径下的所有文件，过滤掉一些接口/抽象类等，把符合条件的文件加载成 Class。
3. 从上面的结果集扫描带 @Component 的 Class，并把这些 Class 封装成待注册的 BeanDefinition。
4. 待注册的 BeanDefinition 集合注册到 BeanDefinitionRegistry，也就是刚才传进来的 AnnotationConfigApplicationContext 实例。

```java
public class ClassPathBeanDefinitionScanner extends AbstractBeanDefinitionReader{
    // 资源加载器
    private final DefaultResourceLoader loader = new DefaultResourceLoader();

    public void scan(String... basePackages) {
        Set<BeanDefinition> beanDefinitions = new LinkedHashSet<>();
        for (String basePackage : basePackages) {
            // 1、扫描指定的路径下的所有文件，把符合条件（非接口/抽象类等）的文件加载成Class
            Map<String, Class<?>> candidateClasses = findCandidateClasses(basePackage);

            // 2、查找带@Component的Class，封装成待注册的BeanDefinition
            Set<BeanDefinition> candidates = findCandidateComponents(candidateClasses);
            beanDefinitions.addAll(candidates);
        }
        // 3、把封装成注册的BeanDefinition注册到registry中
        beanDefinitions.forEach(def -> {
            this.registry.registerBeanDefinition(def.getName(), def);
        });
    }
}
```

## 3、容器 refresh

上面提到的 `refresh()` 方法主流程在 `AbstractApplicationContext#refresh()` 里面实现，Spring 源码的流程实现比较复杂，这里进行了部分简化。完成的功能如下：

1. 初始化 refresh 的上下文环境
2. 初始化 BeanFactory
3. 对 BeanFactory 进行功能增强
4. 执行 BeanFactoryPostProcessor
5. 注册 BeanPostProcessor
6. 实例化所有非延迟加载的单例

```java
public abstract class AbstractApplicationContext implements ConfigurableApplicationContext, BeanDefinitionRegistry{
    @Override
    public void refresh() throws BeansException, IllegalStateException {
        // 1、初始化 refresh 的上下文环境
        prepareRefresh();

        // 2、初始化 BeanFactory
        this.beanFactory = this.obtainFreshBeanFactory();

        /*--至此，已经完成了简单容器的所有功能，下面开始对简单容器进行增强--*/

        // 3、对 BeanFactory 进行功能增强
        prepareBeanFactory(beanFactory);

        // 4、执行 BeanFactoryPostProcessor
        invokeBeanFactoryPostProcessors(beanFactory);

        // 5、注册 BeanPostProcessor
        registerBeanPostProcessors(beanFactory);

        // 6、实例化所有非延迟加载的单例
        finishBeanFactoryInitialization(beanFactory);
    }
}
```

## 4、Bean 实例化

上一步的最后一步实例化所有非延迟加载的单例，主要流程放在 `AbstractBeanFactory#doGetBean` 实现。

```java
public abstract class AbstractBeanFactory implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, SingletonBeanRegistry {

    public <T> T doGetBean(String beanName, @Nullable Class<T> requiredType, @Nullable Object[] args) throws InvocationTargetException, IllegalAccessException {
        // 从三级缓存获取实例
        Object beanInstance = getSingleton(beanName);
        if (beanInstance != null) {
            // 如果找到实例，不管是否正在初始化，都直接返回
            return (T) beanInstance;
        } else {
            // 如果没找到实例，就走创建流程
            String finalBeanName = beanName;
            beanInstance = getSingleton(beanName, () -> {
                return createBean(finalBeanName, def);
            });
        }
        return (T) beanInstance;
    }
}
```

上面调用 `DefaultListableBeanFactory#getSingleton` 从三级缓存获取实例。三级缓存是解决循环依赖的关键：

* `singletonObjects`：一级缓存，存放完全初始化好的 bean，从该缓存中取出的 bean 可以直接使用。
* `earlySingletonObjects`：二级缓存，提前曝光的单例对象的 cache，存放原始的 bean 对象（尚未填充属性），用于解决循环依赖。
* `singletonFactories`：三级缓存，存放 bean 工厂对象，这个对象其实是一个函数式接口，接口实现是创建一个 bean 对象，用来解决 AOP 的循环依赖。

这个三级缓存是这样用的，先调用 `getSingleton()` 从三级缓存里面获取实例

1. 先从一级缓存获取实例
2. 一级缓存拿不到，再从二级缓存获取实例
3. 二级缓存拿不到，从三级缓存获取创建的函数式接口，如果拿到的话就使用其创建一个实例并放入二级缓存，然后清除三级缓存

```java
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
      	// 从一级缓存获取实例
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            // 从二级缓存获取实例
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                // 从三级缓存创建实例并放入二级缓存，然后清除三级缓存
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
        return singletonObject;
    }
```

如果三级缓存都没有获取到实例，说明是第一次初始化，就会调用 `createBean()` 创建实例。`createBean()` 是作为函数式接口的实现传进来的，在 `createBean()` 前后会有一些前后置处理，整个流程是

1. 设置 bean 正在初始化的标记
2. 调用 `createBean()` 真正实例化bean
3. 清除 bean 正在初始化的标记
4. 清除二级和三级缓存，把 bean 添加到一级缓存

```java
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Object singletonObject = this.getSingleton(beanName);
        if (singletonObject == null) {
            // 设置bean正在初始化的标记
            beforeSingletonCreation(beanName);

            // 真正实例化bean，这里其实就是调用createBean方法
            singletonObject = singletonFactory.getObject();

            // 清除bean正在初始化的标记
            afterSingletonCreation(beanName);

            // 清除二级和三级缓存，把bean添加到一级缓存
            addSingleton(beanName, singletonObject);
        }
        return singletonObject;
    }
```

`createBean()` 的实现里面，调用 `createBeanInstance()` 创建完 Bean 实例之后，把一个函数式接口的实现放入三级缓存 `addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, def, finalBeanInstance));`

```java
    private Object createBean(String beanName, BeanDefinition def) {
        // bean实例化前置处理
        applyBeanPostProcessorsBeforeInstantiation(def.getTargetType(), beanName);

        // bean实例化
        Object beanInstance = createBeanInstance(beanName, def);

        // bean实例化后置处理
        applyBeanPostProcessorsAfterInstantiation(beanInstance, beanName);

        // 这里是解决循环依赖的关键，
        // 第一次getSingleton的时候，执行到这里，earlySingletonExposure一定为true，会把bean的工厂方法添加到三级缓存，相当于提前暴露bean
        // 如果产生了循环依赖，第二次getSingleton的时候从三级缓存创建bean，放入二级缓存，如果没有产生循环依赖就跳过这一步
        // 最后addSingleton，清除一级缓存和二级缓存的信息，把bean放入一级缓存
        boolean earlySingletonExposure = isSingletonCurrentlyInCreation(beanName);
        if (earlySingletonExposure) {
             Object finalBeanInstance = beanInstance;
             addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, def, finalBeanInstance));
        }
        Object exposedObject = beanInstance;

        // 属性填充
        populateBean(exposedObject, beanName);

        // bean初始化
        exposedObject = initializeBean(exposedObject, beanName, def);
      
        if (earlySingletonExposure) {
             Object earlySingletonReference = getSingleton(beanName, false);
             if (earlySingletonReference != null && exposedObject == beanInstance) {
                  exposedObject = earlySingletonReference;
             }
        }

        return exposedObject;
   }
```

`getEarlyBeanReference` 的作用是判断目前缓存的 BeanPostProcessor 集合里面有没有 InstantiationAwareBeanPostProcessor 类型的 BeanPostProcessor（AOP就属于该类型），如果有的话就调用该 BeanPostProcessor 的 `getEarlyBeanReference` 方法，提前创建一个代理对象；如果没有的话就返回原来的 bean 对象。**注意此时三级缓存里面只有第三级缓存保存了一个函数式接口的实现，其他两级缓存都是空的**。

```java
    private Object getEarlyBeanReference(String beanName, BeanDefinition def, Object bean) {
        Object exposedObject = bean;
        List<BeanPostProcessor> beanPostProcessors = this.getBeanPostProcessors();
        // 判断有没有 InstantiationAwareBeanPostProcessor 类型的 BeanPostProcessor
        if (!beanPostProcessors.isEmpty()) {
            for (BeanPostProcessor bp : beanPostProcessors) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    exposedObject = ((InstantiationAwareBeanPostProcessor) bp).getEarlyBeanReference(exposedObject, beanName);
                }
            }
        }
        return exposedObject;
    }
```

**根据是否发生循环依赖有三种情况：**

1. **如果没有发生循环依赖，三级缓存根本不会被触发调用，上面的 createBean 按照正常的流程返回一个 bean 实例。此时二级是空的，只有三级缓存有内容。最后清除二级和三级缓存，把该 bean 实例放到一级缓存里面。**
2. **如果发生了循环依赖，没有发生 AOP。调用 `getSingleton()` 获取正在初始化中的实例的时候，就会触发三级缓存的调用，创建 bean 实例放入二级缓存并清空三级缓存，然后返回二级缓存的 bean 实例。最后清除二级和三级缓存，把该 bean 实例放到一级缓存里面。**
3. **如果发生了循环依赖，且发生 AOP。和第 2 点唯一的区别是，此时二级缓存的 bean 实例不是原始对象，而是代理对象。因为在触发三级缓存的调用时，调用 BeanPostProcessor 的 `getEarlyBeanReference()` 创建了代理对象。**

## 5、Bean 初始化

在 `createBean()` 里面，`initializeBean()` 完成 Bean 的初始化流程，AOP 就是在这一步创建了代理对象。初始化的流程如下：

1. 检查 Aware 相关接口并设置依赖
2. BeanPostProcessor 前置处理
3. 调用 InitializingBean#afterPropertiesSet
4. 调用 init-method
5. BeanPostProcessor 后置处理

```java
    private Object initializeBean(Object bean, String beanName, BeanDefinition def) throws InvocationTargetException, IllegalAccessException {
        // 检查Aware相关接口并设置依赖
        invokeAwareInterfaces(bean, beanName);

        // 调用BeanPostProcessor的postProcessBeforeInitialization方法
        bean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);

        // 调用初始化方法，先调用bean的InitializingBean接口方法，后调用bean的自定义初始化方法
        invokeInitMethods(beanName, bean, def);

        // 调用BeanPostProcessor的applyBeanPostProcessorsAfterInitialization方法
        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);

        return bean;
    }
```

# 总结

上面介绍了 ApplicationContext 的核心流程。整个过程并不复杂，相比起 Spring 源码做了很多的简化操作。简单来说分为以下几个步骤：

1. 容器启动，从指定路径或者配置文件扫描资源并加载为 BeanDefinition。
2. 容器 refresh，完成自动注册 BeanFactory 等操作，最后提前实例化所有的 Bean 实例。
3. Bean 实例化，依次完成创建实例、依赖注入、实例初始化等操作，中途使用三级缓存解决循环依赖的问题。
4. 容器启动完成。
