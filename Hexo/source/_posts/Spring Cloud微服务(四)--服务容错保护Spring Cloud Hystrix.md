---
title: Spring Cloud微服务(四)--服务容错保护Spring Cloud Hystrix
date: 2018-11-25 19:06:57
categories: Spring Cloud
tags: [Spring Cloud, 微服务, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是 << Spring Cloud微服务实战 >> 学习笔记, 以便自己查阅.

# 2. 概述

在微服务架构中,我们将系统拆分为很多个服务,各个服务之间通过注册与订阅的方式相互依赖,由于各个服务都是在各自的进程中运行,就有可能由于网络原因或者服务自身的问题导致调用故障或延迟,随着服务的积压,可能会导致服务崩溃.为了解决这一系列的问题,断路器等一系列服务保护机制出现了.

断路器本身是一种开关保护机制,用于在电路上保护线路过载,当线路中有电器发生短路时,断路器能够及时切断故障电路,防止发生过载,发热甚至起火等严重后果.

在分布式架构中,断路器模式的作用也是类似的.

针对上述问题,Spring Cloud Hystrix 实现了断路器,线路隔离等一系列服务保护功能.它也是基于 Netflix 的开源框架 Hystrix 实现的,该框架的目标在于通过控制那些访问远程系统,服务和第三方库的节点,从而对延迟和故障提供更强大的容错能力.Hystrix 具备服务降级,服务熔断,线程和信号隔离,请求缓存,请求合并以及服务监控等强大功能.

# 3. 原理分析

## 3.1 工作流程

![工作流程图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/hystrix-command-flow-chart.png)

### 3.1.1 构建一个HystrixCommand或者HystrixObservableCommand对象

第一步就是构建一个HystrixCommand或者HystrixObservableCommand对象,该对象将代表你的一个依赖请求,向构造函数中传入请求依赖所需要的参数.
如果构建HystrixCommand中的依赖返回单个响应,例如:

```java
HystrixCommand command = new HystrixCommand(arg1, arg2);
```

如果依赖需要返回一个Observable来发射响应, 就需要通过构建HystrixObservableCommand对象来完成,例如:

```java
HystrixObservableCommand command = new HystrixObservableCommand(arg1, arg2);
```

### 3.1.2 执行命令

有4种方式可以执行一个Hystrix命令.
execute()-该方法是阻塞的,从依赖请求中接收到单个响应(或者出错时抛出异常).
queue()-从依赖请求中返回一个包含单个响应的Future对象.
observe()-订阅一个从依赖请求中返回的代表响应的Observable对象.
toObservable()-返回一个Observable对象,只有当你订阅它时,它才会执行Hystrix命令并发射响应.

```java
K             value   = command.execute();
Future<K>     fValue  = command.queue();
Observable<K> ohValue = command.observe();         //hot observable
Observable<K> ocValue = command.toObservable();    //cold observable
```

同步调用方法execute()实际上就是调用queue().get()方法,queue()方法的调用的是toObservable().toBlocking().toFuture().也就是说,最终每一个HystrixCommand都是通过Observable来实现的,即使这些命令仅仅是返回一个简单的单个值.

### 3.1.3 响应是否被缓存

如果这个命令的请求缓存已经开启,并且本次请求的响应已经存在于缓存中,那么就会立即返回一个包含缓存响应的Observable.

### 3.1.4 断路器是否打开

当命令执行执行时,Hystrix会检查断路器是否被打开.
如果断路器被打开,那么Hystrix就不会再执行命名,而是直接路由到第8步,获取fallback方法,并执行fallback逻辑.
如果断路器关闭,那么将进入第5步,检查是否有足够的容量来执行任务.(其中容量包括线程池的容量,队列的容量等等).

### 3.1.5 线程池,队列,信号量是否已满

如果与该命令相关的线程池或者队列已经满了,那么Hystrix就不会再执行命令,而是立即跳到第8步,执行fallback逻辑.

### 3.1.6 HystrixObservableCommand.construct() 或者 HystrixCommand.run()

在这里,Hystrix通过你写的方法逻辑来调用对依赖的请求,通过下列之一的调用:

1. HystrixCommand.run()-返回单个响应或者抛出异常.

2. HystrixObservableCommand.construct()-返回一个发射响应的Observable或者发送一个onError()的通知。.

如果执行run()方法或者construct()方法的执行时间大于命令所设置的超时时间值,那么该线程将会抛出一个TimeoutException异常(如果该命令没有运行在它自己的线程中,则会通过单独的计时线程来抛出),在这种情况下,Hystrix将会路由到第8步,执行fallback逻辑,并且如果run()或者construct()方法没有被取消或者中断,会丢弃这两个方法最终返回的结果.

如果命令最终返回了响应并且没有抛出任何异常,Hystrix在返回响应后会执行一些log和指标的上报,如果是调用run()方法,Hystrix会返回一个Observable,该Observable会发射单个响应并且会调用onCompleted方法来通知响应的回调,如果是调用construct()方法,Hystrix会通过construct()方法返回相同的Observable对象.

### 3.1.7 计算断路器的健康度

Hystrix会报告成功,失败,拒绝和超时的指标给断路器,断路器包含了一系列的滑动窗口数据,并通过该数据进行统计.
它使用这些统计数据来决定断路器是否应该熔断,如果需要熔断,将在一定的时间内不在请求依赖,直到恢复期结束.

### 3.1.8 fallback处理

当命令执行失败的时候,Hystrix会进入fallback尝试回退处理, 我们通常也称为该操作为"服务降级", 而能够引起服务降级处理的情况有以下几种:

1. 命令处于"熔断"状态,即断路器是打开的.
2. 当前命令的线程池,请求队列或者信号量被占满的时候.
3. HystrixObservableCommand.construct() 或者 HystrixCommand.run()抛出异常的时候.

<font color=red>在服务降级逻辑中, 我们需要实现一个通用的响应结果, 并且该结果的处理逻辑应当是从缓存或是根据一些静态逻辑来获取,而不是依赖网络请求获取.如果一定要在降级逻辑中包含网络请求, 那么该请求也必须被包装在HystrixObservableCommand 或者 HystrixCommand 中, 从而形成级联的降级策略, 而最终的降级逻辑一定不是一个依赖网络请求的处理, 而是一个能够稳定地返回结果的处理逻辑.</font>

### 3.1.9 返回成功的响应

如果Hystrix命令执行成功,它将以Observable形式返回响应给调用者.根据你在第2步的调用方式不同,在返回Observable之前可能会做一些转换.

![断路器](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/484085657-5a334a6b3a9dc_articlex.png)

## 3.2 依赖隔离

Hystrix则使用"舱壁隔离"模式实现线程池的隔离,它会为每一个Hystrix命令创建一个独立的线程池,这样就算某个在Hystrix命令包装下的依赖服务出现延迟过高的情况,也只是对该依赖服务的调用产生影响,而不会拖慢其他的服务.

通过对依赖服务的线程池隔离实现,可以带来如下优势:

1. 应用自身得到完全的保护,不会受不可控的依赖服务影响.即便给依赖服务分配的线程池被填满,也不会影响应用自身的其余部分.

2. 可以有效的降低接入新服务的风险,如果新服务接入后运行不稳定或存在问题,完全不会影响到应用其他的请求.

3. 当依赖的服务从失效恢复正常后,它的线程池会被清理并且能够马上恢复健康的服务,相比之下容器级别的清理恢复速度要慢得多.

4. 当依赖的服务出现配置错误的时候,线程池会快速的反应出此问题(通过失败次数,延迟,超时,拒绝等指标的增加情况).同时,我们可以在不影响应用功能的情况下通过实时的动态属性刷新来处理它.

5. 当依赖的服务因实现机制调整等原因造成其性能出现很大变化的时候,此时线程池的监控指标信息会反映出这样的变化.同时,我们也可以通过实时动态刷新自身应用对依赖服务的阈值进行调整以适应依赖方的改变.

6. 除了上面通过线程池隔离服务发挥的优点之外,每个专有线程池都提供了内置的并发实现,可以利用它为同步的依赖服务构建异步的访问.

总之,通过对依赖服务实现线程池隔离,让我们的应用更加健壮,不会因为个别依赖服务出现问题而引起非相关服务的异常.同时,也使得我们的应用变得更加灵活,可以在不停止服务的情况下,配合动态配置刷新实现性能配置上的调整.

# 4. 参考链接

<< Spring Cloud微服务实战 >>