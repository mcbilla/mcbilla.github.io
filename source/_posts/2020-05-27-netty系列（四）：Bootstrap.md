---
title: netty系列（四）：Bootstrap
date: 2020-05-27 17:33:07
tags:
  - Netty
categories:
  - Netty
---

> 上篇文章我们简单介绍了 Netty 的线程模型和基本组件，后面我们开始详细介绍 Netty 每个基本组件。本文先介绍 Netty 的启动器—Bootstrap，这个组件负责把 Netty 的基本组件进行组装配置，然后启动 Netty 服务。

<!--more-->