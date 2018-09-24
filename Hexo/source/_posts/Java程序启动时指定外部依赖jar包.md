---
title: Java程序启动时指定外部依赖jar包
date: 2018-09-24 15:38:40
categories: 后台开发
tags: [基础储备, Java, 解决方案]
---

----

<!-- more -->

# 1. 前言

Java程序启动, 经常会遇到需要指定外部依赖jar包的情况, 在学校开发的时候, 不需要考虑那么多, 一般是通过修改配置文件实现的, 但是这种做法是可能会对服务器上运行的其他程序造成影响的, 所以写这篇博客, 记录一下常用做法.

# 2. 常用做法

## 2.1 方法一: 使用Bootstrap Classloader来加载

我们可以在运行时使用如下参数：

-Xbootclasspath:完全取代系统Java classpath.最好不用.
-Xbootclasspath/a: 在系统class加载后加载. 一般用这个.
-Xbootclasspath/p: 在系统class加载前加载,注意使用, 和系统类冲突就不好了.

```linux
java -Xbootclasspath/a: some.jar:some2.jar: -jar test.jar
```

<font color=red>我个人并不推荐这个做法, 因为当jar包很多的时候, 这个得一个个指定, 并不好用.</font>

## 2.2 方法二: 使用Extension Classloader来加载

首先介绍下java.ext.dirs参数的使用和环境变量:
java中系统属性java.ext.dirs指定的目录由ExtClassLoader加载器加载
如果您的程序没有指定该系统属性(-Djava.ext.dirs=sss/lib), 那么该加载器默认加载\$JAVA_HOME/lib/ext目录下的所有jar文件
但如果你手动指定系统属性且忘了把\$JAVA_HOME/lib/ext路径给加上, 那么ExtClassLoader不会去加载\$JAVA_HOME/lib/ext下面的jar文件, 这意味着你将失去一些功能, 例如java自带的加解密算法实现.

一般命令行如下:

```linux
java -Djava.ext.dirs=$JAVA_HOME/jre/lib/ext:/home/ganlu/dir -jar test.jar
```

<font color=red>我一般用这种方式. 并且一定要记得加上\$JAVA_HOME/jre/lib/ext</font>

# 3. 说明

网上流传的还有诸如把jar包放到环境变量下, 或者修改环境变量, 个人并不倾向于使用, 因为会对其他应用程序造成影响.

# 4. 参考链接

[java -jar命令运行jar包时指定外部依赖jar包](https://blog.csdn.net/w47_csdn/article/details/80254459)
