---
title: Spring Boot之Actuator
date: 2018-11-18 17:51:12
categories: Spring Boot
tags: [Spring Boot, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是<< Spring Boot 实战 >>和<< Java EE开发的颠覆者 Spring Boot 实战 >>的学习笔记, 以便自己查阅.

Sprng Boot 2 actuator变动加大, 网上很多资料都都已经过期, 所以记录一下.

# 2. 配置

在 application.properties 配置文件, actuator 的设置项 management.endpoints(设置 actuator 全局级的属性) 和 management.endpoint(设置 具体endpoint 属性) 开头.

## 2.1 全局级的控制

```properties
#定制管理功能的 port, 如果端口为 -1 代表不暴露管理功能 over HTTP
management.server.port=8081
# 设定 /actuator 入口路径
management.endpoints.web.base-path=/actuator
# 所有endpoint缺省为禁用状态
management.endpoints.enabled-by-default=false
# 暴露所有的endpoint, 但 shutdown 需要显示enable才暴露, * 表示全部, 如果多个的话,用逗号隔开
management.endpoints.web.exposure.include=*
# 排除暴露 loggers和beans endpoint
management.endpoints.web.exposure.exclude=loggers,beans
# 定制化 health 端点的访问路径
management.endpoints.web.path-mapping.health=healthcheck
```

## 2.2 endpoint 级别的控制

所有的endpoint都有 enabled 属性, 可以按需开启或关闭特定端点.

```properties
#启用 shutdown
management.endpoint.shutdown.enabled=true
```

## 2.3 health端点配置

```properties
management.endpoint.health.enabled=true
#show-details属性的取值有: never/always/when-authorized, 默认值是 never
management.endpoint.health.show-details=always  
#增加磁盘空间health 统计, 还有其他health indicator
management.health.diskspace.enabled=true
```

## 2.4 actuator 缺省的设置

缺省 actuator 的根路径为 /actuator
缺省仅开放 health 和 info, 其他都不开放.
有些 endpoint 是GET, 有些是POST方法, 比如 health 为GET, shutdown 为 POST, 从SpringBoot程序启动日志中,可以看出到底有哪些endpoint被开启.

## 2.5 endpoint 清单

actuator 支持的所有 endpoint, 可以查官网

https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html

下面是一些重要的端点,(以访问路径列出):

```txt
/actuator: actuator 严格地说不能算 endpoint, /actuator 返回要给 endpoint 的清单.
/actuator/health: 展现系统运行状态, 在和spring cloud consul集成时候, 就使用该端点检查Spring应用的状态.
/actuator/metric/: 显示更全面的系统指标
/actuator/configprops:展现 SpringBoot 配置项
/actuator/env: 显示系统变量和SpringBoot应用变量, actuator 非常贴心, 如果属性名包含 password/secret/key 这些关键词, 对应的属性值将用 * 号代替.
/actuator/httptrace: 显示最近100条 request-response 信息
/actuator/autoconfig: 生成Spring boot自动化配置报告, 该报告非常有用, 说明如下: Spring Boot项目倾向于使用很多auto config技术, 包括散落在很多config java类. 我们在开发过程中偶尔会遇到, 为什么我的配置没起作用这样的问题. 这时候查看 /actuator/autoconfig 的报告非常有用, 它会告诉你哪些自动化装配成功了,哪些没有成功. 
/actuator/beans: 该端点可以获取 application context 中创建的所有 bean, 并列出它们的scope 和 type 等详细信息
/actuator/mappings: 该端点列出了所有 controller 的路由信息.
```

# 3. 使用

在pom文件导入依赖即可.

```pom
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

## 3.1 其他说明

actuator的UI显示不好,所以网上有很多结合actuator使用的UI界面. 不过日常一般也不会去用这个actuator, 只是spring cloud里面会有需要.

# 4. 参考链接

<< Spring Boot 实战 >>(Craig Walls 著, 丁雪丰 译)
<< Java EE开发的颠覆者 Spring Boot 实战 >>
[SpringBoot系列: Actuator监控](https://www.cnblogs.com/harrychinese/p/SpringBoot_actuator.html)