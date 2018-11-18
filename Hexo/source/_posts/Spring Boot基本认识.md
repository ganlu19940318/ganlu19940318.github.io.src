---
title: Spring Boot基本认识
date: 2018-11-18 17:33:10
categories: Spring Boot
tags: [Spring Boot, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是<< Spring Boot 实战 >>和<< Java EE开发的颠覆者 Spring Boot 实战 >>的学习笔记, 以便自己查阅.

# 2. 什么是Spring Boot

## 2.1 Spring Boot本质

**简单的说,spring boot就是整合了很多优秀的框架,不用我们自己手动的去写一堆xml配置然后进行配置.**

从本质上来说,Spring Boot就是Spring,它做了那些没有它你也会去做的Spring Bean配置.它使用"习惯优于配置"(项目中存在大量的配置,此外还内置了一个习惯性的配置,让你无需手动进行配置)的理念让你的项目快速运行起来.使用Spring Boot很容易创建一个独立运行(运行jar,内嵌Servlet容器),准生产级别的基于Spring框架的项目,使用Spring Boot你可以不用或者只需要很少的Spring配置.

## 2.2 Spring Boot精要

Spring将很多魔法带入了Spring应用程序的开发之中,其中最重要的是以下四个核心.

1. 自动配置: 针对很多Spring应用程序常见的应用功能, Spring Boot能自动提供相关配置;(简化配置)

2. 起步依赖: 告诉Spring Boot需要什么功能, 它就能引入需要的库.(简化导入依赖)

3. 命令行界面: 这是Spring Boot的可选特性, 借此你只需写代码就能完成完整的应用程序, 无需传统项目构建.(感觉没啥用)

4. Actuator: 让你能够深入运行中的Spring Boot应用程序, 一探究竟.(监控)

# 3. 参考链接

<< Spring Boot 实战 >>(Craig Walls 著, 丁雪丰 译)
<< Java EE开发的颠覆者 Spring Boot 实战 >>