---
title: Spring MVC
date: 2018-11-16 19:24:03
categories: Spring
tags: [Spring, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是<<精通Spring 4.x 企业应用开发实战>>学习笔记, 以便自己查阅.

# 2. Spring MVC核心流程

![Spring MVC](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/620f63e1-ee68-30c9-a53d-13107e634364.png)

![Spring MVC](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM20181116195406.png)

SpringMVC 框架中,DispatcherServlet 处于核心位置,它负责协调和组织不同组件已完成请求处理并返回响应的工作.
和大多数Web MVC 框架一样,SpringMVC 通过一个前端的Servlet 接收所有的请求,并将具体工作委托给其他的组件进行处理,DispatcherServlet 就是Spring MVC 的前端Servlet.

Spring MVC 处理请求的整体过程如下:

1. 整个过程始于客户端发出的一个HTTP 请求, Web 应用服务器接收到这个请求, 如果匹配 DispatcherServlet 的请求映射路径(在web.xml中指定), Web容器就将该请求转交给DispatcherServlet 处理.

2. DispatcherServlet 接收到这个请求后, 将根据请求的信息(包括URL,HTTP方法,请求报文头,请求参数,coookie等)及HandlerMapping的配置找到处理请求的处理器(Handler). 可将 HandlerMapping看成路由控制器, 将 Handler 看成目标主机.值得注意的是:Spring MVC 中并没有定义一个Handler 接口,实际上任何一个 Object 都可以成为请求处理器.

3. 当DispatcherServlet 根据 HandlerMapping 得到对应当前请求的 Handler 后,通过HandlerAdapter 对 Handler 进行封装,再以统一的适配器接口调用 Handler. HandlerAdapter 是SpringMvc 框架级接口,顾名思义, HandlerAdapter 是一个适配器, 它用统一的接口对各种Handler 方法进行调用.

4. 处理器完成业务逻辑的处理后将返回一个 ModelAndView 给 DispatcherServlet, ModelAndView 包含了视图逻辑名和模型数据信息.

5. ModelAndView 中包含的是"逻辑视图名"而非真正的视图对象,DispatcherServlet 借由ViewResolver 完成逻辑视图名到真实视图对象的解析工作.

6. 当得到真实的视图对象View 后, DispatcherServlet 就使用这个View 对象对ModelAndView中的模型数据进行视图渲染.

7. 最终客户端得到的响应消息可能是一个HTML页面, 也可能是一个XML或JSON串, 甚至是一张图片或者一个PDF文档等不同的媒体形式.

# 3. 参考链接

[Spring MVC 教程,快速入门,深入分析](http://elf8848.iteye.com/blog/875830)
[深入理解Spring MVC 思想](https://www.cnblogs.com/baiduligang/p/4247164.html)
<<精通Spring 4.x 企业应用开发实战>>