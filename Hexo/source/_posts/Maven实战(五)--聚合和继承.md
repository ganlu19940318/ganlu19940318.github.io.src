---
title: Maven实战(五)--聚合和继承
date: 2018-10-12 15:27:18
categories: Maven基础
tags: [Maven, 基础储备]
---

----

<!-- more -->

# 1. 前言

Java开发过程中, 使用Maven管理依赖. 这是Maven系列文章, 主要记录<<Maven实战>>的学习过程和一些重要知识点, 以方便自己查阅.

# 2. 基本概述

Maven的聚合特性能够把项目的各个模块聚合在一起构建, 而继承特性则能够帮助抽取各模块相同的依赖和插件等配置, 在简化POM的同时, 还能促进各个模块配置的一致性.

# 3. 聚合

Maven聚合也称多模块, 能够一次构建多个模块. 聚合模块本身是一个Maven项目, 所以也有自己的POM文件, 该POM文件的packaging为pom, 并且含有modules和module元素.

![聚合](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-5df76e82367f0df9.png)

```pom
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <groupId>com.glennlgan</groupId>
   <artifactId>springboot.demo</artifactId>
   <version>1.0.0-SNAPSHOT</version>
   <packaging>pom</packaging>
   <modules>
     <module>springboot-mybatis</module>
     <module>springboot-web</module>
     <module>springboot-quickstart</module>
   </modules>
</project>
```

这里每个module的值都是一个当前POM的相对目录, 一般而言, 为了方便快速定位内容, 模块所处的目录名称应该与其artifactId一致, 不过这不是Maven的要求.

因此, 聚合模块与其他模块的目录结构并非一定要父子关系, 通过修改module的值, 也能更改为平级关系:

```pom
    <module>../springboot-quickstart</module>
```

Maven首先会解析聚合模块的POM, 分析要构建的模块, 并计算出一个反应堆构建顺序, 然后根据这个顺序依次构建各个模块. 反应堆包含了模块之间继承和依赖的关系. 模块间的依赖关系会将反应堆构成一个有向非循环图.

# 4. 继承

继承解决的是对重复依赖和插件配置的抽取. 通过定义一个父模块, 将其他模块相同的配置抽离到父模块中, 然后继承父模块, 并且父模块也是一个Maven项目, 其POM文件的packaging为pom.

![继承](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-c90899981f83afe7.png)

子模块需要增加parent元素配置:

```pom
<parent>
    <artifactId>springboot.demo</artifactId>
    <groupId>com.glennlgan</groupId>
    <version>1.0.0-SNAPSHOT</version>
    <relativePath>../pom.xml</relativePath>
</parent>
```

relativePath定义了父模块POM文件的位置. 默认值为../pom.xml, Maven默认父POM在上一层目录下.

POM文件可被继承的元素有:

```text
groupId:项目组ID,项目坐标的核心元素
version:项目版本,项目坐标的核心元素
description:项目的描述信息
organization:项目所在组织机构信息
inceptionYear:项目的创始年份
url:项目的URL地址
developers:项目的开发者信息
contributors:项目的贡献者信息
distributionManagement:项目的部署配置
issueManagement:项目的缺陷跟着系统信息
ciManagement:项目的持续集成系统信息
scm:项目的版本控制系统信息
mailingLists:项目的邮件列表信息
properties:自定义的Maven属性
dependencies:项目的依赖配置
dependencyManagement:项目的依赖管理配置
repositories:项目的仓库配置
build:项目的源码目录配置,输出目录配置,插件配置,插件配置管理等
reporting:项目的报告输出目录配置,报告插件配置等
```

# 5. 依赖管理

子模块继承父模块时, 也会继承父模块的依赖配置, 假设添加一个util的子模块, 该模块只提供一些简单的帮助工具, 与springframework完全无关, 难道也让它依赖spring-core,spring-beans,spring-context么?这显然是不合理的.

Maven提供的dependencyManagement元素既能让子模块继承父模块的依赖配置, 又能保证子模块依赖使用的灵活性. 在dependencyManagement元素下声明的依赖不会引入实际的依赖, 不过它能够约束dependencies下的依赖使用.

在父POM使用dependencyManagement声明的依赖能够统一项目范围中依赖的版本, 在子模块使用依赖的时候就不需要版本了, 只需要简单的配置groupId和artifactId就能获得对应的依赖信息, 从而引进正确的依赖.

如果子模块不声明依赖的使用, 即使该依赖已经在父POM的dependencyManagement中声明了, 也不会产生实际的效果.

如, 在父POM定义如下dependencyManagement:

```pom
<dependencyManagement>
    <dependencies>  
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>4.3.14.RELEASE</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

在子模块使用时, 只需要:

```pom
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
</dependency>
```

对于子模块而言, 可以按需添加依赖, 对于整个项目而言, 可以规范对依赖的版本号管理.

# 6. 参考链接

<<Maven实战>>(许晓斌)