---
title: Imspring 源码介绍（四）：WebMVC 实现
date: 2023-10-10 20:42:45
categories:
- Spring
tags:
- Spring
---

> Imspring 源码介绍（四）：WebMVC 实现

<!--more-->

# 基本概念

所谓 MVC（Model、View、Controller），其实是一种软件设计范式，通过将**业务逻辑、数据、显示分离的方法来组织代码，最主要的目的是降低视图和业务逻辑代码之间的双向耦合**。

Spring MVC 是 Spring 遵循 MVC 设计模式实现的一套 WEB 框架，具体实现在 spring-webmvc 模块，spring-webmvc 是 Spring 项目里面的一个子模块。所以准确来说 Spring MVC 只是 Spring 项目提供的一个功能，而不是一个独立运行的项目。

## MVC 分层结构

![image-20231205125117722](image-20231205125117722.png)

* **Front Controller**：前段控制器，负责管理 WEB 应用程序的整体流程。Spring MVC 中由 `DispatcherServlet` 类充当前端控制器的角色。
* **Controller（控制器）**：接收用户请求，然后委托给模型进行处理（状态改变），处理后再将返回的模型数据反馈给视图，然后由视图负责展示，即 `Controller` 充当 `Model` 和 `View` 之间的信鸽。Spring MVC 中通常使用 `@Controller` 注解将类标记为 Controller。
* **Model（模型）**：数据模型，用于提供要展示的数据，通常包含数据和行为，Spring MVC 一般分离为数据访问层（`Dao`）和服务层（`Service`），`Dao` 曾提供了模型数据查询和模型数据的状态更新等功能，`Service` 层处理具体的业务逻辑。
* **View（视图）**：负责模型的展示，一般就是呈现给我们用户看的东西。Spring MVC 通常使用 JSP + JSTL 来创建视图页面，此外还支持 Themeleaf 和 FreeMaker 等视图技术。

## Spring MVC 组件

在介绍 Spring MVC 工作流程之前，先介绍一下 Spring MVC 用到的组件。

### DispatcherServlet

本质上是一个 Servlet，在 `WEB-NF/web.xml` 配置好 DispatcherServlet，在 web 项目启动的时候就会自动加载。

``` xml
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/j2ee" xmlns:javaee="http://java.sun.com/xml/ns/javaee"
         xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
         xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd"
         version="2.4">

    <servlet>  
        <!-- 配置DispatcherServlet -->  
    　　<servlet-name>springMvc</servlet-name>  
    　　<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
    　　<!-- 指定spring mvc配置文件位置 不指定使用默认情况 -->  
    　　<init-param>     
        　　<param-name>contextConfigLocation</param-name>
        　　<param-value>classpath:spring/spring-mvc.xml</param-value>
   　　 </init-param>  
    　　<!-- 设置启动顺序 -->
    　　<load-on-startup>1</load-on-startup>  
　　</servlet>

　　<!-- ServLet 匹配映射 -->
　　<servlet-mapping>
    　　<servlet-name>springMvc</servlet-name>
   　　 <url-pattern>/</url-pattern>
　　</servlet-mapping>
</web-app>
```

> loadOnStartup 控制Servlet的是否随web容器启动而初始化，分为三种情况：
>
> - loadOnStartup < 0 web容器启动的时候不实例化处理，servlet首次被调用时才被实例化 ，loadOnStartupm默认是这种（即没有配置时）。
> - loadOnStartup > 0 web容器启动的时候做实例化处理，顺序是由小到大，正整数小的先被实例化。
> - loadOnStartup = 0 web容器启动的时候做实例化处理，相当于是最大整数，因此web容器启动时，最后被实例化。

DispatcherServlet 相当于一个中转站，所有的访问都会走到这个 Servlet 中，再根据配置进行中转到相应的 Handler 中进行处理，获取到数据和视图后，在使用相应视图做出响应。 

### Handler

Handler 是用来处理某个请求的对象。Spring MVC 常用的 Handler 实现是 `HandlerMethod`，对应的是用 @RequestMapping 注解修饰的 Controller 类中的某个方法，本质是对 Controller 的 Bean 本身和请求 Method 的包装。

### HandlerMapping

HandlerMapping 存放了所有 url 和 Handler（一般情况下是 HandlerMethod） 的对应关系，当请求进来的时候，根据 url 从 HandlerMapping 中查找适配的 Handler 来处理请求。HandlerMapping 是一个接口，Spring 的常用实现类是 `RequestMappingHandlerMapping`。

### HandlerInterceptor

HandlerInterceptor 是 Spring MVC 定义的拦截器，用于在执行 Handler 前后添加自定义逻辑处理。

```java
public interface HandlerInterceptor {
    // 在HandlerAdapter调用处理器之前调用
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        return true;
    }

    // 在HandlerAdapter调用处理器之后，渲染视图之前调用
    default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
    }
	
    // 在渲染视图之后调用
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
    }
}
```

Interceptor 的功能类似于 Servlet  的 Filter，但是两者在使用上较为不同。两者的区别如下：

* 规范不同：Filter 是 Servlet 规范规定的，被 Tomcat 调用；Interceptor 是 Spring 框架提供的，被 Spring 调用。
* Filter 不能够使用 Spring 容器资源，Interceptor 作为 Spring 容器的组件，可以使用 Spring 容器的任何资源。
* 作用范围不同：Filter 只在所有 Servlet 的前后起作用，不管是 Spring MVC 提供的 `DispatcherServlet` 还是其他 Servlet 组件；Interceptor 可以针对 `DispatcherServlet`  的 Controller 前后进行拦截，相比 Filter 更深入方法内部。

<img src="image-20231205194756544.png" width="50%" height="50%">

### HandlerExecutionChain

HandleExcutionChains 是对 Handler 的二次封装，HandlerExecutionChain =    Handler + 一组特定顺序的 HandlerInterceptor。

### HandlerAdapter

HandlerAdapter 是用来执行 Handler 逻辑的适配器。`DispatcherServlet` 拿到 Handler 之后，不是直接执行 Handler 的逻辑，而是交给 HandlerAdapter 去执行。

之所以需要 HandlerAdapter 这个角色，是因为 `DispatcherServlet` 作为一个 servlet，他的原始参数只有 `request` 和 `response`，但是每个 Handler 的入参和返回结果可能都不同。HandlerAdapter 的作用就是把 `request` 和 `response` 解析成不同 Handler 的入参，然后把 Handler 的返回结果统一封装成 ModelAndView。

### ModelAndView

从名称上就可以知道，ModelAndView = Model + View。因为 Java 的方法返回值每次只能返回一个对象，所以索性把 Model 和 View 封装在 ModelAndView 里面一起返回。

```java
public class ModelAndView {
    // 视图对象名称
    @Nullable
    private Object view;

    // 数据的 map 集合
    @Nullable
    private ModelMap model;
    ......
}
```

### ViewResolver

ModelAndView 里面的 View 不是完整的，仅仅是一个页面视图名称（viewName），且没有后缀名。ViewResolver 的作用就是根据 ModelAndView 对象里面的 viewName 获取真正的 View 对象。

## Spring MVC 工作流程

**Spring MVC 的核心逻辑都是围绕 DispatcherServlet 展开**，一个完整的 Spring MVC 工作流程如下图所示：

![image-20231205140014461](image-20231205140014461.png)

1. 用户发送请求，被前端控制器 DispatcherServlet 拦截。
2. DispatcherServlet 从 HandlerMapping 根据请求 url 查找 Handler。这里的 Handler 是我们自定义的 Controller 的某个方法。
3. 找到 Handler 之后，封装为 HandlerExecutionChain 返回给 DispatcherServlet。这里的 HandlerInterceptor 包含我们自定义的拦截器。
4. 选择一个合适的 HandlerAdapter 去执行 HandlerExecutionChain。
5. 执行 HandlerExecutionChain 里面的 Handler 和 HandlerInterceptor。
6. 执行完之后返回 ModelAndView 对象给 DispatcherServlet，同时包含了数据和模型。
7. DispatcherServlet 选择合适的视图解析器 ViewResolver 解析 ModelAndView。
8. 返回具体 View 给 DispatcherServlet。
9. DispatcherServlet 根据 View 使用 JSP/Freemarker 等技术渲染视图，即将模型数据填充至视图中。
10. DispatcherServlet 把渲染好的页面返回给用户。

# 代码整体流程

![SpringWebMVC时序图](SpringWebMVC时序图.png)

imspring-mvc 的整体流程如上图所示，整个流程和上面的 *Spring MVC 工作流程示意图* 基本一致。

1. 在 `WEB-INF/web.xml` 里面配置好 `SpringServletContextListener`，注意 Listener 会在所有 servlet 初始化之前执行。在 tomcat 启动的时候，在 `SpringServletContextListener` 里面创建 ApplicationContext 容器，并设置到 servlet 上下文。
2. `DispatchServlet` 设置了跟随 tomcat 启动而启动，所以 `DispatchServlet` 进行 `init()` 初始化，从 servlet 上下文获取 ApplicationContext 容器，然后从 ApplicationContext 容器里面获取 `HandlerMapping` 和 `HandlerAdapter` 的实例，并把上面的内容保存起来，至此 `DispatchServlet` 已经具备使用 Spring 容器的能力！
3. 请求进来，统一由 `DispatchServlet` 拦截。不管是什么请求方式的请求（GET/POST），最终都进去 `doDispatch()` 方法，进行主流程处理，这就是 Spring WebMVC 的整体流程。
