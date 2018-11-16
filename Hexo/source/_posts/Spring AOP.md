---
title: Spring AOP
date: 2018-11-16 08:48:09
categories: Spring
tags: [Spring, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是<<精通Spring 4.x 企业应用开发实战>>学习笔记, 以便自己查阅.

# 2. AOP 产生的背景

"存在即合理", 任何一种理论或技术的产生, 必然有它的原因. 了解它产生的背景, 为了解决的问题有助于我们更好地把握AOP的概念. 软件开发一直在寻求一种高效开发, 扩展, 维护的方式. 从面向过程的开发实践中, 前人将关注点抽象出来, 对行为和属性进行聚合, 形成了面向对象的开发思想, 其在一定程度上影响了软件开发的过程. 鉴于此, 我们在开发的过程中会对软件开发进行抽象, 分割成各个模块或对象. 例如, 我们会对API进行抽象成四个模块: Controller, Service, Gateway, Command. 这很好地解决了业务级别的开发, 但对于系统级别的开发我们很难聚焦. 比如, 对于每一个模块需要进行打日志, 代码监控, 异常处理. 以打日志为例, 我只能将日志代码嵌套在各个对象上, 而无法关注日志本身, 而这种现象又偏离了OOP思想.

![AOP](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/d954be75ebca469617d4.webp)
![AOP](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/efa47f28e884f98d3a0c.webp)

为了能够更好地将系统级别的代码抽离出来, 去掉与对象的耦合, 就产生了面向AOP(面向切面). 如上图所示, OOP属于一种横向扩展, AOP是一种纵向扩展. AOP依托于OOP, 进一步将系统级别的代码抽象出来, 进行纵向排列, 实现低耦合.

# 3. AOP 的家庭成员

## 3.1 PointCut

即在哪个地方进行切入,它可以指定某一个点,也可以指定多个点.比如类A的methord函数,当然一般的AOP与语言(AOL)会采用多用方式来定义PointCut,比如说利用正则表达式,可以同时指定多个类的多个函数.

## 3.2 Advice

在切入点干什么,指定在PointCut地方做什么事情(增强),打日志,执行缓存,处理异常等等.

## 3.3 Advisor/Aspect

PointCut + Advice 形成了切面Aspect,这个概念本身即代表切面的所有元素.但到这一地步并不是完整的,因为还不知道如何将切面植入到代码中,解决此问题的技术就是PROXY

## 3.4 Proxy

Proxy 即代理,其不能算做AOP的家庭成员,更相当于一个管理部门,它管理了AOP的如何融入OOP.之所以将其放在这里,是因为Aspect虽然是面向切面核心思想的重要组成部分,但其思想的践行者却是Proxy,也是实现AOP的难点与核心所在.

# 4. AOP的技术实现Proxy

AOP仅仅是一种思想,那为了让这种思想发光,必然脱离语言本身的技术支持,Java在实现该技术时就是采用的代理Proxy,那我们就去了解一下,如何通过代理实现面向切面.

## 4.1 静态代理

就像我们去买二手房要经过中介一样,房主将房源委托给中介,中介将房源推荐给买方.中间的任何手续的承办都由中介来处理,不需要我们和房主直接打交道.无论对买方还是卖房都都省了很多事情,但同时也要付出代价,对于买房当然是中介费,对于代码的话就是性能.下面我们来介绍实现AOP的三种代理方式.下面我就以买房的过程中需要打日志为例介绍三种代理方式静态和动态是由代理产生的时间段来决定的.静态代理产生于代码编译阶段,即一旦代码运行就不可变了.下面我们来看一个例子.

```java
public interface IPerson {
    public void doSomething();
}
```

```java
public class Person implements IPerson {
    public void doSomething(){
        System.out.println("I want wo sell this house");
    }
}
```

```java
public class PersonProxy {
    private IPerson iPerson;
    private final static Logger logger = LoggerFactory.getLogger(PersonProxy.class);

    public PersonProxy(IPerson iPerson) {
        this.iPerson = iPerson;
    }
    public void doSomething() {
        logger.info("Before Proxy");
        iPerson.doSomething();
        logger.info("After Proxy");
    }

    public static void main(String[] args) {
        PersonProxy personProxy = new PersonProxy(new Person());
        personProxy.doSomething();
    }
}
```

通过代理类我们实现了将日志代码集成到了目标类,但从上面我们可以看出它具有很大的局限性:需要固定的类编写接口(或许还可以接受,毕竟有提倡面向接口编程),需要实现接口的每一个函数(不可接受)同样会造成代码的大量重复,将会使代码更加混乱.

## 4.2 动态代理

那能否通过实现一次代码即可将logger织入到所有函数中呢,答案当然是可以的,此时就要用到java中的反射机制.

```java
public class PersonProxy implements InvocationHandler{
    private Object delegate;
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    public Object bind(Object delegate) {
        this.delegate = delegate;
        return Proxy.newProxyInstance(delegate.getClass().getClassLoader(), delegate.getClass().getInterfaces(), this);
    }
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = null;
        try {
            logger.info("Before Proxy");
            result = method.invoke(delegate, args);
            logger.info("After Proxy");
        } catch (Exception e) {
            throw e;
        }
        return result;
    }

    public static void main(String[] args) {
        PersonProxy personProxy = new PersonProxy();
        IPerson iperson = (IPerson) personProxy.bind(new Person());
        iperson.doSomething();
    }
}
```

它的好处理时可以为我们生成任何一个接口的代理类,并将需要增强的方法织入到任意目标函数.但它仍然具有一个局限性,就是只有实现了接口的类,才能为其实现代理.

## 4.3 CGLIB

CGLIB解决了动态代理的难题,它通过生成目标类子类的方式来实现来实现代理,而不是接口,规避了接口的局限性.CGLIB是一个强大的高性能代码生成包,其在运行时期(非编译时期)生成被代理对象的子类,并重写了被代理对象的所有方法,从而作为代理对象.

```java
public class PersonProxy implements MethodInterceptor {
    private Object delegate;
    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    public Object intercept(Object proxy, Method method, Object[] args,  MethodProxy methodProxy) throws Throwable {
        logger.info("Before Proxy");
        Object result = methodProxy.invokeSuper(method, args);
        logger.info("After Proxy");
        return result;
    }

    public static Person getProxyInstance() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Person.class);

        enhancer.setCallback(new PersonProxy());
        return (Person) enhancer.create();
    }
}
```

当然CGLIB也具有局限性,对于无法生成子类的类(final类),肯定是没有办法生成代理子类的.

以上就是三种代理的实现方式,但千成别被迷惑了,在Spring AOP中这些东西已经被封装了,不需要我们自己实现.要不然得累死,但了解AOP的实现原理(即基于代理)还是很有必要的.

# 5. 参考链接

[Spring-aop 全面解析（从应用到原理）](https://juejin.im/post/591d8c8ba22b9d00585007dd)
<<精通Spring 4.x 企业应用开发实战>>