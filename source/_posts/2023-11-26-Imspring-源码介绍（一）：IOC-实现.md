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
* 依赖注入，以及解决循环依赖问题。
* 使用 BeanFactoryPostProcessor 实现定制化 Bean 工厂（Configuration 配置类的基础）
* 使用 BeanPostProcessor 实现定制化 Bean。（AOP的基础）
* 配置文件加载和环境切换。

为了让大家更加清楚整个流程，先介绍下 Spring 的一些基础概念和组件。

## 基础概念

### ApplicationContext 和 BeanFactory

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
