---
title: 设计模式之MVC模式
date: 2019-02-20 11:20:31
categories: 设计模式
tags: [设计模式, 基础储备]
---

----

<!-- more -->

# 1. 什么是MVC模式

MVC 模式代表 Model-View-Controller(模型-视图-控制器)模式. 这种模式用于应用程序的分层开发.

Model(模型) - 模型代表一个存取数据的对象或 JAVA POJO。它也可以带有逻辑，在数据变化时更新控制器。
View(视图)  - 视图代表模型包含的数据的可视化。
Controller(控制器) - 控制器作用于模型和视图上。它控制数据流向模型对象，并在数据变化时更新视图。它使视图与模型分离开。

![MVC模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190220113832.png)

# 2. 示例UML图

![MVC模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190220114511.png)

# 3. 示例代码地址

https://github.com/ganlu19940318/Head-First/tree/master/Model-View-Controller

# 4. 参考链接

<< Head First 设计模式 >>

[MVC 模式|菜鸟教程](http://www.runoob.com/design-pattern/mvc-pattern.html)