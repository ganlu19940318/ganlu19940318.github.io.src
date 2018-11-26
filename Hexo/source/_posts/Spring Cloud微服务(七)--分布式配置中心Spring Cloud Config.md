---
title: Spring Cloud微服务(七)--分布式配置中心Spring Cloud Config
date: 2018-11-26 15:14:11
categories: Spring Cloud
tags: [Spring Cloud, 微服务, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是 << Spring Cloud微服务实战 >> 学习笔记, 以便自己查阅.

# 2. 概述

Spring Cloud Config是Spring Cloud团队创建的一个全新项目,用来为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持,它分为服务端与客户端两个部分.

其中服务端也称为分布式配置中心,它是一个独立的微服务应用,用来连接配置仓库并为客户端提供获取配置信息,加密/解密信息等访问接口;

而客户端则是微服务架构中的各个微服务应用或基础设施,它们通过指定的配置中心来管理应用资源与业务相关的配置内容,并在启动的时候从配置中心获取和加载配置信息.

Spring Cloud Config实现了对服务端和客户端中环境变量和属性配置的抽象映射,所以它除了适用于Spring构建的应用程序之外,也可以在任何其他语言运行的应用程序中使用.由于Spring Cloud Config实现的配置中心默认采用Git来存储配置信息,所以使用Spring Cloud Config构建的配置服务器,天然就支持对微服务应用配置信息的版本管理,并且可以通过Git客户端工具来方便的管理和访问配置内容.当然它也提供了对其他存储方式的支持,比如:SVN仓库,本地化文件系统.

# 3. 服务端详解

![服务端详解](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20181104101755803.png)

客户端应用从配置管理中获取配置信息遵从下面的执行流程:

1. 应用启动时,根据bootstrap.properties中配置的应用名(application},环境名(profile},分支名(label},向Config Server请求获取配置信息.

2. Config Server根据自己维护的Git仓库信息和客户端传递过来的配置定位信息去查找配置信息.

3. 通过git clone命令将找到的配置信息下载到Config Server的文件系统中.

4. Config Server创建Spring的ApplicationContext实例,并从Git本地仓库中加载配置文件,最后将这些配置内容读取出来返回给客户端应用.

5. 客户端应用在获得外部配置文件后加载到客户端的ApplicationContext实例,该配置内容的优先级高于客户端Jar包内部的配置内容,所以在Jar包中重复的内容将不再被加载.

# 4. 高可用问题

## 4.1 传统作法

通常在生产环境,Config Server与服务注册中心一样,我们也需要将其扩展为高可用的集群.在之前实现的config-server基础上来实现高可用非常简单,不需要我们为这些服务端做任何额外的配置,只需要遵守一个配置规则:将所有的Config Server都指向同一个Git仓库,这样所有的配置内容就通过统一的共享文件系统来维护,而客户端在指定Config Server位置时,只要配置Config Server外的均衡负载即可,就像如下图所示的结构:

![传统做法](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/3-13.png)

## 4.2 注册为服务

虽然通过服务端负载均衡已经能够实现,但是作为架构内的配置管理,本身其实也是可以看作架构中的一个微服务.所以,另外一种方式更为简单的方法就是把config-server也注册为服务,这样所有客户端就能以服务的方式进行访问.通过这种方法,只需要启动多个指向同一Git仓库位置的config-server就能实现高可用了.

# 5. 参考链接

<< Spring Cloud微服务实战 >>
[Spring Cloud Config 服务端详解](https://blog.csdn.net/qq_25484147/article/details/83714630)
[Spring Cloud构建微服务架构（四）分布式配置中心（续）](http://blog.didispace.com/springcloud4-2/)