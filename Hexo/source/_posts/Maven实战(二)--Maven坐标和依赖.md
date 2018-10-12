---
title: Maven实战(二)--Maven坐标和依赖
date: 2018-10-11 17:07:00
categories: Maven基础
tags: [Maven, 基础储备]
---

----

<!-- more -->

# 1. 前言

Java开发过程中, 使用Maven管理依赖. 这是Maven系列文章, 主要记录<<Maven实战>>的学习过程和一些重要知识点, 以方便自己查阅.

# 2. 坐标详解

```POM
<groupId>com.hust.schedule</groupId>
<artifactId>schedule-calculation</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>jar</packaging>
```

groupId: 定义当前Maven项目隶属的实际项目, 比如com.hust.schedule就是指华中科技大学的一个schedule项目.

artifactId: 该元素定义实际项目中的一个Maven项目(模块), 比如schedule-calculation就是schedule项目中的calculation模块.

<font color=red>artifactId的规范的做法是使用项目名作为前缀,模块名作为后缀, 用-分割.</font>

version: 定义Maven项目当前所处的版本.

packaging: 定义Maven项目的打包方式.

# 3. 依赖配置

```POM
<dependencies>
    <dependency>
        <groupId>...</groupId>
        <artifactId>...</artifactId>
        <version>...</version>
        <type>...</type>
        <scope>...</scope>
        <optional>...</optional>
        <exclusions>
            <exclusion>
            ...
            <exclusion>
        <exclusions>
    </dependency>
</dependencies>
```

groupId, artifactId, version: 作为依赖的基本坐标, 用于定位依赖
type: 依赖的类型, 对应于packaging, 默认为jar
scope: 依赖的范围
optional: 标记依赖是否可选
exclusions: 用来排除传递性依赖

# 4. 依赖范围

1. Maven在编译项目主代码的时候需要使用一套classpath; Maven在编译和执行测试的时候, 会使用另外一套classpath. 实际运行Maven项目的时候, 又会使用一套classpath.

2. 依赖范围就是用来控制依赖与这三种classpath(编译classpath, 测试classpath, 运行classpath)的关系.

3. 依赖的范围通过scope指定.

## 4.1 Maven依赖范围

compile: 编译依赖范围, 默认值. 使用此依赖范围的依赖对于编译, 测试, 运行三种classpath都有效.
test: 测试依赖范围, 使用此依赖范围的依赖只对于测试classpath有效. 典型的就是Junit依赖一般声明为测试依赖范围.
provided: 已提供依赖范围, 使用此依赖范围的依赖, 对于编译和测试classpath有效, 但在运行时无效, 典型的例子是servlet-api, 编译和测试的时候需要, 但运行时由容器提供.
runtime: 运行时依赖范围, 使用此依赖范围的依赖, 对于测试和运行classpath有效, 但在编译主代码是无效. 典型的例子是JDBC驱动实现, 项目主代码的编译仅需要JDK的JDBC接口, 只有在测试和运行时才需要实现.
system: 系统依赖范围, 该依赖与三种classpath的关系与provided依赖范围完全一致, 但是使用system依赖范围的依赖必须通过systemPath元素显式的指定依赖文件的路径.由于此类依赖不是通过Maven仓库解析, 而且往往与本机系统绑定, 因此需要谨慎使用. 比如

```pom
<dependency>
  <groupId>javax.sql</groupId>
  <artifactId>jdbc-stdext</artifactId>
  <version>2.0</version>
  <scope>system</scope>
  <systemPath>${java.home}/lib/rt.jar</systemPath>
</dependency>
```

import: 导入依赖范围, 该依赖范围不会对三种classpath产生实际影响. 这里具体看依赖管理.

![Maven依赖范围](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-5465cb24042faef0.png)

# 5. 传递性依赖

Maven的传递性依赖是指不需要考虑你依赖的库文件所需要依赖的库文件, 能够将依赖模块的依赖自动的引入.

依赖的范围不仅可以控制依赖与三种classpath的关系, 还会对传递性依赖产生影响. 假设A依赖于B, B依赖于C, 则说A对于B是第一直接依赖, B对C是第二直接依赖, A对于C是传递依赖. 第一直接依赖范围和第二直接依赖范围决定了传递性依赖的范围, 其结果如下:
![传递性依赖](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-448a85ae15007924.png)

# 6. 依赖调解

一般情况下, 只关心项目的直接依赖, 而不关心直接依赖引入的传递性依赖, 但当传递性依赖出现问题时, 需要知道该传递性依赖是怎么引进来的.

Maven依赖调解第一原则: 路径最近者优先, 如：A->B->C->X(1.0), A->D-X(2.0), 则X的2.0版本会被解析使用;
Maven依赖调解第二原则: 第一声明者优先, 如: A->B->X(1.0)、A->D->X(2.0), 若B的依赖声明在D之前, 则使用X的1.0版本, 否则使用X的2.0版本.

# 7. 可选依赖

假设有下面的依赖关系:A->B、B->X(可选), B->Y(可选), 由于X和Y是可选的, 所以依赖不会传递, X和Y不会对A有任何影响.

可选依赖的必要性:项目B实现2种特性, 特性一依赖于X, 特性二依赖于Y, 而且这两个特性是互斥的, 用户不可能同时适用这两个特性, 这时候可选依赖就有用了.

原则上说, 是不应该使用可选依赖的, 根据面向对象的单一职责性原则, 该原则同样适用于Maven项目的规划.

# 8. 最佳实践

## 8.1 排除依赖

传递性依赖会给项目隐式的引入很多依赖, 这极大的简化了项目依赖的管理, 但是有时某些依赖会带来问题, 这时需要把带来问题的依赖排除掉.

```POM
<dependencies>
    <dependency>
        <groupId>com.cc.maven</groupId>
        <artifactId>project-b</artifactId>
        <version>1.0.0</version>
        <exclusions>
            <exclusion>
                <groupId>com.cc.maven</groupId>
                <artifactId>project-c</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
```

![排除依赖](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20170620201859704.png)

## 8.2 归类依赖

来自同一个项目的不同模块, 其版本号应该是相同的, 如springframework项目有spring-core、spring-beans模块, 对这些模块的版本号通过属性定义, 再进行引用, 这样可以进行版本的整体升级:

```pom
<properties>
    <springframework.version>4.3.13.RELEASE</springframework.version>
</properties>
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>${springframework.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-beans</artifactId>
        <version>${springframework.version}</version>
    </dependency>
</dependencies>
```

## 8.3 优化依赖

去掉多余的依赖, 显示声明某些必要的依赖.
通过mvn dependency:list 查看项目已解析的依赖
通过mvn dependency:tree 查看项目的依赖树
通过mvn dependency:analyze工具可以帮助分析当前项目的依赖

# 9. 参考链接

<<Maven实战>>(许晓斌)
[Maven的排除依赖、归类依赖、优化依赖](https://blog.csdn.net/cckevincyh/article/details/73512722)
[Maven实战读书笔记（三）：Maven依赖](https://www.cnblogs.com/Jxwz/p/8372372.html)