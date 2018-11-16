---
title: Spring事务管理
date: 2018-11-16 15:06:45
categories: Spring
tags: [Spring, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是<<精通Spring 4.x 企业应用开发实战>>学习笔记, 以便自己查阅.

# 2. 核心接口

Spring事务管理的实现有许多细节,如果对整个接口框架有个大体了解会非常有利于我们理解事务,下面通过讲解Spring的事务接口来了解Spring实现事务的具体策略.
Spring事务管理涉及的接口的联系如下:
![事务管理](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20160324011156424.png)

# 3. 事务管理器

Spring并不直接管理事务,而是提供了多种事务管理器,他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现.
Spring事务管理器的接口是org.springframework.transaction.PlatformTransactionManager,通过这个接口,Spring为各个平台如JDBC,Hibernate等都提供了对应的事务管理器,但是具体的实现就是各个平台自己的事情了.此接口的内容如下:

```java
Public interface PlatformTransactionManager()...{  
    // 由TransactionDefinition得到TransactionStatus对象
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException; 
    // 提交
    Void commit(TransactionStatus status) throws TransactionException;  
    // 回滚
    Void rollback(TransactionStatus status) throws TransactionException;  
}
```

从这里可知具体的具体的事务管理机制对Spring来说是透明的,它并不关心那些,那些是对应各个平台需要关心的,所以Spring事务管理的一个优点就是为不同的事务API提供一致的编程模型,如JTA,JDBC,Hibernate,JPA.下面分别介绍各个平台框架实现事务管理的机制.

## 3.1 JDBC事务

如果应用程序中直接使用JDBC来进行持久化,DataSourceTransactionManager会为你处理事务边界.为了使用DataSourceTransactionManager,你需要使用如下的XML将其装配到应用程序的上下文定义中:

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
</bean>
```

实际上,DataSourceTransactionManager是通过调用java.sql.Connection来管理事务,而后者是通过DataSource获取到的.通过调用连接的commit()方法来提交事务,同样,事务失败则通过调用rollback()方法进行回滚.

## 3.2 Hibernate事务

如果应用程序的持久化是通过Hibernate实现的,那么你需要使用HibernateTransactionManager.对于Hibernate3,需要在Spring上下文定义中添加如下的 < bean > 声明:

```xml
<bean id="transactionManager" class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
</bean>
```

sessionFactory属性需要装配一个Hibernate的session工厂,HibernateTransactionManager的实现细节是它将事务管理的职责委托给org.hibernate.Transaction对象,而后者是从Hibernate Session中获取到的.当事务成功完成时,HibernateTransactionManager将会调用Transaction对象的commit()方法,反之,将会调用rollback()方法.

## 3.3 Java持久化API事务(JPA)

Hibernate多年来一直是事实上的Java持久化标准,但是现在Java持久化API作为真正的Java持久化标准进入大家的视野.如果你计划使用JPA的话,那你需要使用Spring的JpaTransactionManager来处理事务.你需要在Spring中这样配置JpaTransactionManager:

```xml
<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
</bean>
```

JpaTransactionManager只需要装配一个JPA实体管理工厂(javax.persistence.EntityManagerFactory接口的任意实现).JpaTransactionManager将与由工厂所产生的JPA EntityManager合作来构建事务.

## 3.4 Java原生API事务

如果你没有使用以上所述的事务管理,或者是跨越了多个事务管理源(比如两个或者是多个不同的数据源),你就需要使用JtaTransactionManager:

```xml
<bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager">
        <property name="transactionManagerName" value="java:/TransactionManager" />
</bean>
```

JtaTransactionManager将事务管理的责任委托给javax.transaction.UserTransaction和javax.transaction.TransactionManager对象,其中事务成功完成通过UserTransaction.commit()方法提交,事务失败通过UserTransaction.rollback()方法回滚.

# 4. 基本事务属性的定义

上面讲到的事务管理器接口PlatformTransactionManager通过getTransaction(TransactionDefinition definition)方法来得到事务,这个方法里面的参数是TransactionDefinition类,这个类就定义了一些基本的事务属性.

那么什么是事务属性呢?事务属性可以理解成事务的一些基本配置,描述了事务策略如何应用到方法上.事务属性包含了5个方面,如图所示:

![事务属性](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20160325003448793.png)

而TransactionDefinition接口内容如下:

```java
public interface TransactionDefinition {
    int getPropagationBehavior(); // 返回事务的传播行为
    int getIsolationLevel(); // 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
    int getTimeout();  // 返回事务必须在多少秒内完成
    boolean isReadOnly(); // 事务是否只读，事务管理器能够根据这个返回值进行优化，确保事务是只读的
}
```

我们可以发现TransactionDefinition正好用来定义事务属性,下面详细介绍一下各个事务属性.

## 4.1 传播行为

不展开,下面单独列一章介绍.

## 4.2 隔离级别

事务的第二个维度就是隔离级别(isolation level).隔离级别定义了一个事务可能受其他并发事务影响的程度. 

1. 并发事务引起的问题

在典型的应用程序中,多个事务并发运行,经常会操作相同的数据来完成各自的任务.并发虽然是必须的,但可能会导致以下的问题.

```TXT
脏读（Dirty reads）——脏读发生在一个事务读取了另一个事务改写但尚未提交的数据时。如果改写在稍后被回滚了，那么第一个事务获取的数据就是无效的。
不可重复读（Nonrepeatable read）——不可重复读发生在一个事务执行相同的查询两次或两次以上，但是每次都得到不同的数据时。这通常是因为另一个并发事务在两次查询期间进行了更新。
幻读（Phantom read）——幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录。
```

2. 隔离级别

|          隔离级别          |                                                                         含义                                                                         |
|:--------------------------:|:----------------------------------------------------------------------------------------------------------------------------------------------------:|
|      ISOLATION_DEFAULT     |                                                             使用后端数据库默认的隔离级别                                                             |
| ISOLATION_READ_UNCOMMITTED |                                     最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读                                     |
|  ISOLATION_READ_COMMITTED  |                                    允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生                                    |
|  ISOLATION_REPEATABLE_READ |                   对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生                   |
|   ISOLATION_SERIALIZABLE   | 最高的隔离级别，完全服从ACID的隔离级别，确保阻止脏读、不可重复读以及幻读，也是最慢的事务隔离级别，因为它通常是通过完全锁定事务相关的数据库表来实现的 |

## 4.3 只读

事务的第三个特性是它是否为只读事务.如果事务只对后端的数据库进行该操作,数据库可以利用事务的只读特性来进行一些特定的优化.通过将事务设置为只读,你就可以给数据库一个机会,让它应用它认为合适的优化措施.

## 4.4 事务超时

为了使应用程序很好地运行,事务不能运行太长的时间.因为事务可能涉及对后端数据库的锁定,所以长时间的事务会不必要的占用数据库资源.事务超时就是事务的一个定时器,在特定时间内事务如果没有执行完毕,那么就会自动回滚,而不是一直等待其结束.

## 4.5 回滚规则

事务五边形的最后一个方面是一组规则,这些规则定义了哪些异常会导致事务回滚而哪些不会.默认情况下,事务只有遇到运行期异常时才会回滚,而在遇到检查型异常时不会回滚(这一行为与EJB的回滚行为是一致的),但是你可以声明事务在遇到特定的检查型异常时像遇到运行期异常那样回滚.同样,你还可以声明事务遇到特定的异常不回滚,即使这些异常是运行期异常.

# 5. 事务状态

上面讲到的调用PlatformTransactionManager接口的getTransaction()的方法得到的是TransactionStatus接口的一个实现,这个接口的内容如下:

```java
public interface TransactionStatus{
    boolean isNewTransaction(); // 是否是新的事物
    boolean hasSavepoint(); // 是否有恢复点
    void setRollbackOnly();  // 设置为只回滚
    boolean isRollbackOnly(); // 是否为只回滚
    boolean isCompleted; // 是否已完成
}
```

可以发现这个接口描述的是一些处理事务提供简单的控制事务执行和查询事务状态的方法,在回滚或提交的时候需要应用对应的事务状态.

# 6. 事务传播行为

事务的第一个方面是传播行为(propagation behavior).当事务方法被另一个事务方法调用时,必须指定事务应该如何传播.例如:方法可能继续在现有事务中运行,也可能开启一个新事务,并在自己的事务中运行.Spring定义了七种传播行为:

|          传播行为         |                                                                                                                                   含义                                                                                                                                   |
|:-------------------------:|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|    PROPAGATION_REQUIRED   |                                                                                     表示当前方法必须运行在事务中。如果当前事务存在，方法将会在该事务中运行。否则，会启动一个新的事务                                                                                     |
|    PROPAGATION_SUPPORTS   |                                                                                           表示当前方法不需要事务上下文，但是如果存在当前事务的话，那么该方法会在这个事务中运行                                                                                           |
|   PROPAGATION_MANDATORY   |                                                                                                     表示该方法必须在事务中运行，如果当前事务不存在，则会抛出一个异常                                                                                                     |
|  PROPAGATION_REQUIRED_NEW |                                             表示当前方法必须运行在它自己的事务中。一个新的事务将被启动。如果存在当前事务，在该方法执行期间，当前事务会被挂起。如果使用JTATransactionManager的话，则需要访问TransactionManager                                            |
| PROPAGATION_NOT_SUPPORTED |                                                            表示该方法不应该运行在事务中。如果存在当前事务，在该方法运行期间，当前事务将被挂起。如果使用JTATransactionManager的话，则需要访问TransactionManager                                                           |
|     PROPAGATION_NEVER     |                                                                                              表示当前方法不应该运行在事务上下文中。如果当前正有一个事务在运行，则会抛出异常                                                                                              |
|     PROPAGATION_NESTED    | 表示如果当前已经存在一个事务，那么该方法将会在嵌套事务中运行。嵌套的事务可以独立于当前事务进行单独地提交或回滚。如果当前事务不存在，那么其行为与PROPAGATION_REQUIRED一样。注意各厂商对这种传播行为的支持是有所差异的。可以参考资源管理器的文档来确认它们是否支持嵌套事务 |

## 6.1 PROPAGATION_REQUIRED

假如当前正要运行的事务不在另外一个事务里, 那么就起一个新的事务, 比方说, ServiceB.methodB的事务级别定义为PROPAGATION_REQUIRED, 那么因为执行ServiceA.methodA的时候, ServiceA.methodA已经起了事务. 这时调用ServiceB.methodB, ServiceB.methodB看到自己已经执行在ServiceA.methodA的事务内部. 就不再起新的事务. 而假如ServiceA.methodA执行的时候发现自己没有在事务中, 他就会为自己分配一个事务. 这样, 在ServiceA.methodA或者在ServiceB.methodB内的不论什么地方出现异常. 事务都会被回滚. 即使ServiceB.methodB的事务已经被提交, 可是ServiceA.methodA在接下来fail要回滚, ServiceB.methodB也要回滚.

![事务传播行为](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20140420132157531.png)

## 6.2 PROPAGATION_SUPPORTS

假设当前在事务中. 即以事务的形式执行. 假设当前不再一个事务中, 那么就以非事务的形式执行.

## 6.3 PROPAGATION_MANDATORY

必须在一个事务中执行. 也就是说, 他仅仅能被一个父事务调用. 否则,他就要抛出异常.

## 6.4 PROPAGATION_REQUIRES_NEW

这个就比較绕口了. 比方我们设计ServiceA.methodA的事务级别为PROPAGATION_REQUIRED, ServiceB.methodB的事务级别为PROPAGATION_REQUIRES_NEW. 那么当运行到ServiceB.methodB的时候, ServiceA.methodA所在的事务就会挂起. ServiceB.methodB会起一个新的事务. 等待ServiceB.methodB的事务完毕以后, 他才继续运行. 他与PROPAGATION_REQUIRED 的事务差别在于事务的回滚程度了. 由于ServiceB.methodB是新起一个事务, 那么就是存在两个不同的事务. 假设ServiceB.methodB已经提交, 那么ServiceA.methodA失败回滚. ServiceB.methodB是不会回滚的. 假设ServiceB.methodB失败回滚, 假设他抛出的异常被ServiceA.methodA捕获, ServiceA.methodA事务仍然可能提交.

![事务传播行为](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20140420132504937.png)

## 6.5 PROPAGATION_NOT_SUPPORTED

当前不支持事务.比方ServiceA.methodA的事务级别是PROPAGATION_REQUIRED.而ServiceB.methodB的事务级别是PROPAGATION_NOT_SUPPORTED,那么当执行到ServiceB.methodB时.ServiceA.methodA的事务挂起.而他以非事务的状态执行完,再继续ServiceA.methodA的事务.

## 6.6 PROPAGATION_NEVER

不能在事务中执行.如果ServiceA.methodA的事务级别是PROPAGATION_REQUIRED.而ServiceB.methodB的事务级别是PROPAGATION_NEVER,那么ServiceB.methodB就要抛出异常了.

## 6.7 PROPAGATION_NESTED

理解Nested的关键是savepoint.他与PROPAGATION_REQUIRES_NEW的差别是,PROPAGATION_REQUIRES_NEW另起一个事务.将会与他的父事务相互独立.而Nested的事务和他的父事务是相依的,他的提交是要等和他的父事务一块提交的.也就是说,假设父事务最后回滚.他也要回滚的.而Nested事务的优点是他有一个savepoint.

# 7. 参考链接

[Spring事务管理（详解+实例）](https://blog.csdn.net/trigl/article/details/50968079)
[浅析Spring事务传播行为和隔离级别](https://www.cnblogs.com/zsychanpin/p/7074071.html)
<<精通Spring 4.x 企业应用开发实战>>