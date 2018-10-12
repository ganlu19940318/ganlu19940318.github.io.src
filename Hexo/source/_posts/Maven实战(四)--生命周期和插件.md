---
title: Maven实战(四)--生命周期和插件
date: 2018-10-12 11:43:35
categories: Maven基础
tags: [Maven, 基础储备]
---

----

<!-- more -->

# 1. 前言

Java开发过程中, 使用Maven管理依赖. 这是Maven系列文章, 主要记录<<Maven实战>>的学习过程和一些重要知识点, 以方便自己查阅.

# 2. 生命周期概述

Maven的生命周期是对所有构建过程的抽象和统一. 包含了项目的清理, 初始化, 编译, 测试, 打包, 集成测试, 验证, 部署和站点生成等几乎所有构建步骤.

Maven的生命周期是抽象的, 其实际行为是由插件来完成的, 生命周期和插件两者协同合作, 密不可分.

这种思想与设计模式中的模板方法非常相似. 模板方法模式在父类定义算法的整体结构, 子类通过实现或者重写父类的方法来控制实际行为, 这样既能保证算法有足够的可扩展性, 又能严格控制算法的整体结构.

# 3. 生命周期详解

Maven拥有3套独立的生命周期:clean, default, site.

clean生命周期的目的是清理项目.

default生命周期的目的是构建项目.

site生命周期的目的是建立项目站点.

每个生命周期包含一些阶段(phase), 这些阶段是有序的, 后面的阶段会依赖于前面的阶段.

## 3.1 clean生命周期

clean生命周期的3个阶段:

1. pre-clean:执行一些清理前需要完成的动作

2. clean:清理上一次构建生成的文件

3. post-clean:执行一些清理后需要完成的动作

## 3.2 default生命周期

1. validate

2. initialize

3. generate-sources

4. process-sources 处理项目主资源文件, 一般来说. 是对src/main/resources目录的内容进行变量替换等工作, 复制到项目输出的主classpath目录中.

5. generate-resources

6. process-resources

7. compile 编译项目的主源码到主classpath目录中.

8. process-classes

9. generate-test-sources

10. process-test-sources 处理项目测试资源文件, 一般来说, 是对src/test/resources目录的内容进行变量替换等工作, 复制到项目输出的测试classpath目录中.

11. generate-test-resources

12. process-test-resources

13. test-compile编译项目的测试源码到测试classpath目录中.

14. process-test-classes

15. test使用单元测试框架进行测试, 测试代码不会被打包或部署

16. prepare-package

17. package 将编译好的代码, 打包成可发布的格式, 如jar

18. pre-integration-test

19. integration-test

20. post-integration-test

21. verify

22. install 将包安装到Maven本地仓库, 供本地其他Maven项目使用

23. deploy 将最终的包复制到远程仓库, 供其他开发人员和Maven项目使用

## 3.3 site生命周期

site生命周期的目的是建立和发布项目站点, Maven能够基于POM所包含的信息, 自动生成一个友好的站点, 方便团队交流和发布项目信息, 含如下阶段:

1. pre-site 执行一些在生成项目站点之前需要完成的工作

2. site 生成项目站点文档

3. post-site 执行一些在生成项目站点之后需要完成的工作

4. site-deploy 将生成的项目站点发布到服务器上

# 4. 插件目标

对于一个插件, 为了复用代码, 它往往能够完成多个任务, 例如maven-dependency-plugin, 能够分析项目依赖; 列出项目依赖树; 列出项目已解析的依赖, 为这样每个功能独立编写一个插件, 显然是不可取的, 因为这些功能背后有相同的代码, 因此将这些功能聚集在一个插件里, 每个功能就是一个插件目标.

# 5. 插件绑定

Maven的生命周期与插件相互绑定, 用于完成实际的构建任务, 具体而言, 是生命周期的阶段与插件的目标相互绑定, 以完成某个具体的构建任务.

## 5.1 内置绑定

下面是一些内置的绑定:
![插件绑定](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-8137c64354e6f2c4.png)
![插件绑定](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-33944749982b4280.png)
![插件绑定](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-4aa6214722460660.png)

## 5.2 自定义绑定

除了内置绑定外, 用户能够自己选择将某个插件目标绑定到生命周期的某个阶段上, 以便在项目构建过程中执行更丰富的任务.

```pom
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <version>2.1.1</version>
    <executions>
        <execution>
            <id>attach-source</id>
            <phase>verify</phase>
            <goals>
                <goal>jar-no-fork</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

除了基本的插件坐标配置, executions元素下的每个execution元素用来配置执行一个任务. 有时候, 即使不配置phase阶段, 插件目标也能绑定到生命周期中去, 这是因为很多插件的目标在编写时已经定义了默认的绑定阶段, 可以通过maven-help-plugin查看插件的详细信息:
mvn help:describe –Dplugin=org.apache.maven.plugins:maven-source-plugin:2.1.1 -Ddetail

如果多个目标被绑定到同一个阶段, 它们的执行顺序会是怎样?
这些插件声明的先后顺序决定了目标的执行顺序.

# 6. 插件解析机制

为了方便用户使用和配置插件, Maven不需要用户提供完整的插件坐标信息, 就可以解析得到正确的插件.

与依赖构件一样, 插件构件同样基于坐标存储在Maven仓库中, 但Maven会区别对待依赖的远程仓库与插件的远程仓库.

通过repositories及其子元素repository可以配置依赖的远程仓库;
插件的远程仓库需要使用pluginRepositories和pluginRepository进行配置.

```pom
<pluginRepositories>
    <pluginRepository>
        <id>central</id>
        <name>Maven plugin</name>
        <url>htpp://repo1.maven.org/maven2</url>
        <layout>default</layout>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <releases>
            <enabled>true</enabled>
        </releases>
    </pluginRepository>
</pluginRepositories>
```

默认情况下, 如果该插件是Maven官方插件, 则可以省略groupId(org.apache.maven.plugins), Maven在解析该插件的时候, 会自动将groupId补上.

当插件没有添加版本号时, 若该插件是核心插件, 则在超级POM已经定义了版本号, 若不是核心插件, Maven会遍历本地仓库和远程仓库, 计算出latest和release的值, Maven 2使用latest, 但因为latest可能是快照版本, Maven 3更改为使用release.

# 7. 参考链接

<<Maven实战>>(许晓斌)