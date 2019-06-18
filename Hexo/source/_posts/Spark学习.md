---
title: Spark学习
date: 2019-6-14 22:17:28
categories: 基础储备
tags: [大数据,Spark]
---

----

<!-- more -->

# 1. 介绍

Spark是MapReduce的替代方案,而且兼容HDFS,Hive,可融入Hadoop的生态系统,以弥补MapReduce的不足.

Spark最突出的表现在于,它能将作业与作业之间产生的大规模的工作数据集存储在内存中,这使得Spark在性能上超过了等效的MapReduce.(MapReduce的数据集始终需要从磁盘上加载)

## 1.1 Spark的组成

SparkCore:将分布式数据抽象为弹性分布式数据集(RDD),实现了应用任务调度,RPC,序列化和压缩,并为运行在其上的上层组件提供API.

SparkSQL:Spark Sql 是Spark来操作结构化数据的程序包,可以让用户使用SQL语句的方式来查询数据,Spark支持多种数据源,包含Hive表,parquest以及JSON等内容.

SparkStreaming:是Spark提供的实时数据进行流式计算的组件.

MLlib:提供常用机器学习算法的实现库.

GraphX:提供一个分布式图计算框架,能高效进行图计算.

BlinkDB:用于在海量数据上进行交互式SQL的近似查询引擎.

Tachyon:以内存为中心高容错的的分布式文件系统.

![spark生态](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616083023.jpg)

![spark生态](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616083530.jpg)

# 2. Spark RDD及编程接口

RDD(Resilient Distributed Dataset)叫做弹性分布式数据集,是Spark中最基本的数据抽象,它代表一个不可变,可分区,里面的元素可并行计算的集合.

创建操作:RDD的初始创建都是由SparkContext来负责的,将内存中的集合或者外部文件系统作为输入源.

转换操作:将一个RDD通过一定的操作变换成另一个RDD.

控制操作:对RDD进行持久化,可以让RDD保存在磁盘或者内存中,以便后续重复使用.

行动操作:由于Spark是惰性计算的,所以对于任何RDD进行行动操作,都会触发Spark作业的运行,从而产生最终的结果.

![Spark RDD及编程接口](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616094534.jpg)

## 2.1 Spark RDD

RDD是弹性分布式数据集,一个RDD的生成只有两种途径,一是来自于内存集合和外部存储系统,另一种是通过转换操作来自于其他RDD,比如map,filter,join等.

对RDD开源进行两个方面的控制操作:持久化和分区.一方面,可以指明需要重用哪些RDD,选择一种存储策略进行保存;另一方面,可以让RDD根据记录中的键值在机器中分区.

![Spark RDD](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616102944.jpg)

# 3. Spark运行模式及原理

## 3.1 Spark运行模式

![Spark运行模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616105355.jpg)

## 3.2 Spark基本工作流程

![Spark基本工作流程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616105640.jpg)

Cluster Manager:在standalone模式中即为Master主节点,控制整个集群,监控worker.在YARN模式中为资源管理器
Worker节点:从节点,负责控制计算节点,启动Executor或者Driver.
Driver:运行Application 的main()函数.
Executor:执行器,是为某个Application运行在Worker Node上的一个进程.

Spark运行流程图如下
![Spark基本工作流程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616111256.jpg)

1. 构建Spark Application的运行环境,启动SparkContext;
2. SparkContext向资源管理器(可以是Standalone,Mesos,Yarn)申请运行Executor资源,并启动StandaloneExecutorbackend;
3. Executor向SparkContext申请Task;
4. SparkContext将应用程序分发给Executor;
5. SparkContext构建成DAG图,将DAG图分解成Stage,将Taskset发送给Task Scheduler,最后由Task Scheduler将Task发送给Executor运行;
6. Task在Executor上运行,运行完释放所有资源.

## 3.3 术语

Application:Appliction都是指用户编写的Spark应用程序,其中包括一个Driver功能的代码和分布在集群中多个节点上运行的Executor代码.

Driver:Spark中的Driver即运行上述Application的main函数并创建SparkContext,创建SparkContext的目的是为了准备Spark应用程序的运行环境,在Spark中有SparkContext负责与ClusterManager通信,进行资源申请,任务的分配和监控等,当Executor部分运行完毕后,Driver同时负责将SparkContext关闭,通常用SparkContext代表Driver.

Executor:某个Application运行在Worker节点上的一个进程,该进程负责运行某些Task,并且负责将数据存到内存或磁盘上,每个Application都有各自独立的一批Executor, 在Spark on Yarn模式下,其进程名称为CoarseGrainedExecutor Backend.一个CoarseGrainedExecutor Backend有且仅有一个Executor对象, 负责将Task包装成TaskRunner,并从线程池中抽取一个空闲线程运行Task, 这使得每一个CoarseGrainedExecutor Backend能并行运行.并行运行的Task的数量取决与分配给它的CPU个数.

Cluter Manager:指的是在集群上获取资源的外部服务.目前有三种类型:

1. Standalon:spark原生的资源管理,由Master负责资源的分配
2. Apache Mesos:与hadoop MR兼容性良好的一种资源调度框架
3. Hadoop Yarn: 主要是指Yarn中的ResourceManager

Worker:集群中任何可以运行Application代码的节点,在Standalone模式中指的是通过slave文件配置的Worker节点,在Spark on Yarn模式下就是NodeManager节点.

Task:被送到某个Executor上的工作单元,但hadoopMR中的MapTask和ReduceTask概念一样,是运行Application的基本单位,多个Task组成一个Stage,而Task的调度和管理等是由TaskScheduler负责.

Job:包含多个Task组成的并行计算,往往由Spark Action触发生成, 一个Application中往往会产生多个Job

Stage: 每个Job会被拆分成多组Task, 作为一个TaskSet, 其名称为Stage,Stage的划分和调度是有DAGScheduler来负责的,Stage有非最终的Stage(Shuffle Map Stage)和最终的Stage(Result Stage)两种,Stage的边界就是发生shuffle的地方

DAGScheduler: 根据Job构建基于Stage的DAG(Directed Acyclic Graph有向无环图),并提交Stage给TaskScheduler. 其划分Stage的依据是RDD之间的依赖的关系找出开销最小的调度方法.

TaskSedulter: 将TaskSET提交给worker运行,每个Executor运行什么Task就是在此处分配的. TaskScheduler维护所有TaskSet,当Executor向Driver发生心跳时,TaskScheduler会根据资源剩余情况分配相应的Task.另外TaskScheduler还维护着所有Task的运行标签,重试失败的Task.

将这些术语串起来的运行层次图如下:

![运行层次图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616114450.jpg)

### 3.3.1 Job,Stage,Task之间的关系

什么是Job?
Job简单讲就是提交给spark的任务.

什么是Stage?
Stage是每一个Job处理过程要分为的几个阶段.

什么是Task?
Task是每一个Job处理过程要分几为几次任务.Task是任务运行的最小单位,最终是要以Task为单位运行在executor中.

Job和Stage和Task之间有什么关系?
Job----> 一个或多个Stage---> 一个或多个Task

下图是一个Job分成了三个Stage.
![Job,Stage,Task之间的关系](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616121542.jpg)

一个Stage的Task的数量是由谁来决定的?
是由输入文件的切片个数来决定的.在HDFS中不大于128m的文件算一个切片(默认128m),通过算子修改了某一个RDD的分区数量,Task数量也会同步修改.

一个Job任务的Task数量是由谁来决定的?
一个Job任务可以有一个或多个Stage,一个Stage又可以有一个或多个Task.所以一个Job的Task数量是(Stage数量 * Task数量)的总和.

每一个Stage中的Task最大的并行度?
并行度:是指指令并行执行的最大条数.在指令流水中,同时执行多条指令称为指令并行.
理论上:每一个Stage下有多少的分区,就有多少的Task,Task的数量就是我们任务的最大的并行度.
(一般情况下,我们一个Task运行的时候,使用一个Core)
实际上:最大的并行度,取决于我们的application任务运行时使用的executor拥有的Cores的数量.

如果我们的Task数量超过这个Cores的总数怎么办?
先执行Cores个数量的Task,然后等待cpu资源空闲后,继续执行剩下的Task.

## 3.4 Spark on YARN模式

Spark on YARN模式根据Driver在集群中的位置分为两种模式:一种是YARN-Client模式,另一种是YARN-Cluster(或称为YARN-Standalone模式)

### 3.4.1 Yarn-Client模式

Yarn-Client模式中,Driver在客户端本地运行,这种模式可以使得Spark Application和客户端进行交互.

YARN-Client模式的工作流程步骤为:

1. Spark Yarn Client向YARN的ResourceManager申请启动Application Master.同时在SparkContent初始化中将创建DAGScheduler和TASKScheduler等,由于我们选择的是Yarn-Client模式,程序会选择YarnClientClusterScheduler和YarnClientSchedulerBackend
2. ResourceManager收到请求后,在集群中选择一个NodeManager,为该应用程序分配第一个Container,要求它在这个Container中启动应用程序的ApplicationMaster,与YARN-Cluster区别的是在该ApplicationMaster不运行SparkContext,只与SparkContext进行联系进行资源的分派
3. Client中的SparkContext初始化完毕后,与ApplicationMaster建立通讯,向ResourceManager注册,根据任务信息向ResourceManager申请资源(Container)
4. 一旦ApplicationMaster申请到资源(也就是Container)后,便与对应的NodeManager通信,要求它在获得的Container中启动CoarseGrainedExecutorBackend,CoarseGrainedExecutorBackend启动后会向Client中的SparkContext注册并申请Task
5. Client中的SparkContext分配Task给CoarseGrainedExecutorBackend执行,CoarseGrainedExecutorBackend运行Task并向Driver汇报运行的状态和进度,以让Client随时掌握各个任务的运行状态,从而可以在任务失败时重新启动任务
6. 应用程序运行完成后,Client的SparkContext向ResourceManager申请注销并关闭自己

![Yarn-Client模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616124223.jpg)

### 3.4.2 YARN-Cluster模式

在YARN-Cluster模式中,当用户向YARN中提交一个应用程序后,YARN将分两个阶段运行该应用程序:
第一个阶段是把Spark的Driver作为一个ApplicationMaster在YARN集群中先启动;
第二个阶段是由ApplicationMaster创建应用程序,然后为它向ResourceManager申请资源,并启动Executor来运行Task,同时监控它的整个运行过程,直到运行完成.

YARN-Cluster的工作流程分为以下几个步骤:

1. Spark Yarn Client向YARN中提交应用程序,包括ApplicationMaster程序,启动ApplicationMaster的命令,需要在Executor中运行的程序等
2. ResourceManager收到请求后,在集群中选择一个NodeManager,为该应用程序分配第一个Container,要求它在这个Container中启动应用程序的ApplicationMaster,其中ApplicationMaster进行SparkContext等的初始化
3. ApplicationMaster向ResourceManager注册,这样用户可以直接通过ResourceManage查看应用程序的运行状态,然后它将采用轮询的方式通过RPC协议为各个任务申请资源,并监控它们的运行状态直到运行结束
4. 一旦ApplicationMaster申请到资源(也就是Container)后,便与对应的NodeManager通信,要求它在获得的Container中启动CoarseGrainedExecutorBackend,CoarseGrainedExecutorBackend启动后会向ApplicationMaster中的SparkContext注册并申请Task.
5. ApplicationMaster中的SparkContext分配Task给CoarseGrainedExecutorBackend执行,CoarseGrainedExecutorBackend运行Task并向ApplicationMaster汇报运行的状态和进度,以让ApplicationMaster随时掌握各个任务的运行状态,从而可以在任务失败时重新启动任务
6. 应用程序运行完成后,ApplicationMaster向ResourceManager申请注销并关闭自己

![YARN-Cluster模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616124936.jpg)

### 3.4.3 Spark Client 和 Spark Cluster的区别

理解YARN-Client和YARN-Cluster深层次的区别之前先清楚一个概念:Application Master.在YARN中,每个Application实例都有一个ApplicationMaster进程,它是Application启动的第一个容器.它负责和ResourceManager打交道并请求资源,获取资源之后告诉NodeManager为其启动Container.从深层次的含义讲YARN-Cluster和YARN-Client模式的区别其实就是ApplicationMaster进程的区别.

YARN-Cluster模式下,Driver运行在AM(Application Master)中,它负责向YARN申请资源,并监督作业的运行状况.当用户提交了作业之后,就可以关掉Client,作业会继续在YARN上运行,因而YARN-Cluster模式不适合运行交互类型的作业.

YARN-Client模式下,Application Master仅仅向YARN请求Executor,Client会和请求的Container通信来调度他们工作,也就是说Client不能离开.

## 3.5 RDD运行流程

RDD在Spark中运行大概分为以下三步:

1. 创建RDD对象;
2. DAGScheduler模块介入运算,计算RDD之间的依赖关系,RDD之间的依赖关系就形成了DAG;
3. 每一个Job被分为多个Stage.划分Stage的一个主要依据是当前计算因子的输入是否是确定的,如果是则将其分在同一个Stage,避免多个Stage之间的消息传递开销.

示例图如下.
![RDD运行流程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190616130154.jpg)

# 4. 参考链接

[Spark大数据处理技术](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/606318%20Spark%E5%A4%A7%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86%E6%8A%80%E6%9C%AF%20%20%E5%AE%8C%E6%95%B4%E7%89%88.pdf)

[Spark学习之路 (一)Spark初识](https://www.cnblogs.com/qingyunzong/p/8886338.html)
[Spark(一): 基本架构及原理](https://www.cnblogs.com/cxxjohnson/p/8909578.html)
[Spark学习之路 (三)Spark之RDD](https://www.cnblogs.com/qingyunzong/p/8899715.html)
[spark中job,stage,task之间的关系](https://blog.csdn.net/mys_35088/article/details/80864092)