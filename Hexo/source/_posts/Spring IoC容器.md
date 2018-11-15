---
title: Spring IoC容器
date: 2018-11-15 22:13:44
categories: Spring
tags: [Spring, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是<<精通Spring 4.x 企业应用开发实战>>学习笔记, 以便自己查阅.

# 2. IoC容器的概念

IoC容器就是具有依赖注入功能的容器, IoC容器负责实例化, 定位, 配置应用程序中的对象及建立这些对象间的依赖. 应用程序无需直接在代码中new相关的对象, 应用程序由IoC容器进行组装. 在Spring中BeanFactory是IoC容器的实际代表者.

Spring IoC容器如何知道哪些是它管理的对象呢? 这就需要配置文件, Spring IoC容器通过读取配置文件中的配置元数据, 通过元数据对应用中的各个对象进行实例化及装配. 一般使用基于xml配置文件进行配置元数据, 而且Spring与配置文件完全解耦的, 可以使用其他任何可能的方式进行配置元数据, 比如注解, 基于java文件的, 基于属性文件的配置都可以.

那Spring IoC容器管理的对象叫什么呢?

# 3. Bean的概念

由IoC容器管理的那些组成你应用程序的对象我们就叫它Bean, Bean就是由Spring容器初始化,装配及管理的对象, 除此之外, bean就与应用程序中的其他对象没有什么区别了. 那IoC怎样确定如何实例化Bean, 管理Bean之间的依赖关系以及管理Bean呢? 这就需要配置元数据.

# 4. 详解IoC容器

在Spring Ioc容器的代表就是org.springframework.beans包中的BeanFactory接口, BeanFactory接口提供了IoC容器最基本功能; 而org.springframework.context包下的ApplicationContext接口扩展了BeanFactory, 还提供了与Spring AOP集成, 国际化处理, 事件传播及提供不同层次的context实现 (如针对web应用的WebApplicationContext). 简单说, BeanFactory提供了IoC容器最基本功能, 而 ApplicationContext 则增加了更多支持企业级功能支持. ApplicationContext完全继承BeanFactory, 因而BeanFactory所具有的语义也适用于ApplicationContext.

让我们来看下IoC容器到底是如何工作. 在此我们以xml配置方式来分析一下:

1. 准备配置文件: 在配置文件中声明Bean定义也就是为Bean配置元数据.

2. 由IoC容器进行解析元数据: IoC容器的Bean Reader读取并解析配置文件, 根据定义生成BeanDefinition配置元数据对象, IoC容器根据BeanDefinition进行实例化, 配置及组装Bean.

3. 实例化IoC容器: 由客户端实例化容器, 获取需要的Bean.

执行过程如图.
![IoC](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/2e57867020e686671fde7923f891ab69__%E6%9C%AA%E6%A0%87%E9%A2%98-5.jpg)

# 5. 参考链接

[【第二章】 IoC 之 2.2 IoC 容器基本原理 ——跟我学Spring3](http://jinnianshilongnian.iteye.com/blog/1413851)
<<精通Spring 4.x 企业应用开发实战>>