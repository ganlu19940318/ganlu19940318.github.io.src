---
title: Yarn
date: 2019-6-5 08:00:57
categories: 技术储备
tags: [Yarn, Hadoop]
---

----

<!-- more -->
# 0. 前言

概念了解参考[<< 架构大数据  大数据技术及算法解析 >>](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/638722%20%E6%9E%B6%E6%9E%84%E5%A4%A7%E6%95%B0%E6%8D%AE%20%20%E5%A4%A7%E6%95%B0%E6%8D%AE%E6%8A%80%E6%9C%AF%E5%8F%8A%E7%AE%97%E6%B3%95%E8%A7%A3%E6%9E%90_2015.06.pdf)
深入理解参考[<< Hadoop Yarn权威指南 >>](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/631938%20Hadoop%20YARN%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97.pdf)

# 1. Hadoop MapReduce

MapReduce是一种用于大规模数据集的并行运算的编程模型.它方便了编程人员在不会分布式并行编程的情况下,将自己的程序运行在分布式系统上.

## 1.1 Hadoop MapReduce第一代(MRv1)

### 1.1.1 组件图

![组件图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190608003309.jpg)

### 1.1.2 存在的问题

1. JobTracker是MapReduce的集中处理点,存在单点故障.
2. JobTracker完成了太多的任务,造成了过多的资源消耗,当MapReduce作业非常多的时候,会造成很大的内存开销,也增加了JobTracker失败的风险.
3. 在TaskTracker端,以Map/Reduce Task的数目作为资源的表示过于简单,没有考虑到CPU/内存的占用情况,如果两个大内存消耗的Task被调度到了一起,很容易出现内存不足.
4. 在TaskTracker端,把资源强化分为Map Task Slot和Reduce Task Slot,如果系统中只有Map Task或者只有Reduce Task,会造成资源的浪费,也就是集群资源利用的问题.
5. 其他:源代码层面分析时,会发现代码非常难读,常常因为一个class做了太多的事情,代码量达3000多行,造成class的任务不清晰,增加bug修复和版本维护的难度; 从操作的角度来看,MapReduce框架在有任何变化时,都会强制进行系统级别的升级更新.不管用户的喜好,强制让分布式集群系统的每一个用户端同时更新,这些更新会让用户为了验证他们之前的应用程序是不是适用新的Hadoop版本而浪费大量时间.

## 1.2 Hadoop MapReduce第二代(Yarn)

### 1.2.1 组件图

![组件图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190608003320.jpg)

# 2. Yarn的工作流程

![Yarn的工作流程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20160315144119726)

步骤1: 用户向Yarn提交应用程序,其中包括用户程序、相关文件、启动ApplicationMaster命令、ApplicationMaster程序等.

步骤2: ResourceManager为该应用程序分配第一个Container,并且与Container所在的NodeManager通信,并且要求该NodeManager在这个Container中启动应用程序对应的ApplicationMaster.

步骤3: ApplicationMaster首先会向ResourceManager注册,这样用户才可以直接通过ResourceManager查看到应用程序的运行状态,然后它为准备为该应用程序的各个任务申请资源,并监控它们的运行状态直到运行结束,即重复后面4~7步骤.

步骤4: ApplicationMaster采用轮询的方式通过RPC协议向ResourceManager申请和领取资源.

步骤5: 一旦ApplicationMaster申请到资源后,便会与申请到的Container所对应的NodeManager进行通信,并且要求它在该Container中启动任务.

步骤6: 任务启动.NodeManager为要启动的任务配置好运行环境,包括环境变量、JAR包、二进制程序等,并且将启动命令写在一个脚本里,通过该脚本运行任务.

步骤7: 各个任务通过RPC协议向其对应的ApplicationMaster汇报自己的运行状态和进度,以让ApplicationMaster随时掌握各个任务的运行状态,从而可以再任务运行失败时重启任务.

步骤8: 应用程序运行完毕后,其对应的ApplicationMaster会向ResourceManager通信,要求注销和关闭自己.

# 3. MapReduce的几个阶段

![MapReduce的几个阶段](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190608005345.jpg)

整个流程可以分为 Split, Map, Shuffle, Reduce, Output 五个阶段.

## 3.1 Split

在 Split 阶段会把需要处理的数据划分为不同的切片; 把个切片交给不同 Map 程序进行处理; 切片后数据会被解析为 key-value 对输入到 Map 进行处理.

## 3.2 Map

在 Map 阶段可以对输入的 key-value 对进行处理后再以 key-value 对的形式输出.

## 3.3 Shuffle

shuffle 的目的是保证 Reduce 处理的数据是有序的,并且尽可能的减少时间和资源消耗. Shuffle 过程需要分别在 Map 端和 Reduce 端进行.

### 3.3.1 Map Shuffle

![MapReduce的几个阶段](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190608005742.jpg)

在 Map 端的 Shuffle 过程是对 Map 的输出结果进行一系列处理,最终得到分区有序的文件.分区指的是需要同一个 Reduce 处理的数据存储在一起;有序指的是同一个分区内的数据按照 key 升序排列.整体流程如下:

1. Partition: Partition 阶段计算并保存 Map 输出的每对 key-value 需要发送到的 Reduce 编号.

2. Spill: Map 输出的数据需要先输出到一个叫做 kvbuffer 的内存缓冲区, 等缓冲区到达阈值的时候一次性把数据写入硬盘, 这个过程叫做 spill. kvbuffer 的大小默认是 100M, 阈值是 80%. 这样设计的目的是为了提高写数据的效率, 同时保证写数据的时候不会阻塞 Map 的输出.

3. Sort: 在溢写之前会对缓冲区中的数据按照 Partition 和 key 进行升序排列,然后再写入硬盘.排序的结果就是 kvbuffer 中的数据是按照 Partition 聚集在一起,同一 Partition 中的数据按照 key 有序排列.

4. Merge: 如果 Map 的输出数据量很大, 可能会进行好几次 Spill 操作, 产生很多溢写文件. Map 输出完毕后我们还需把所有的 Spill 文件 Merge 为一个文件.最终的合并结果仍然保证数据按照 Partition 聚集在一起,同一 Partition 中的数据按照 key 有序排列.

### 3.3.2 Reduce Shuffle

在 Reduce 端 Shuffle 的目的是从不同的 Map 端获取数据,进行排序合并后输入给 Reduce.

1. copy: Reduce 端会从 Map 端的合并文件中下载对应自己的那一部分数据.如果数据量较小会放在内存中,数据量较大的话会写入磁盘.
2. megre && sort && grouping: 把来自不同 Map 的数据 边合并 边按照 key 进行排序.把所有 value 按照 key 聚合到一个个的迭代器中, 形成新的 key-value 对. 这些新的 key-value 对会输入给 Reduce 进行处理.

## 3.4 Reduce

在 Reduce 阶段可以对输入的 key-value 对进行处理后再以 key-value 对的形式输出.

# 4. 参考链接

[<< 架构大数据  大数据技术及算法解析 >>](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/638722%20%E6%9E%B6%E6%9E%84%E5%A4%A7%E6%95%B0%E6%8D%AE%20%20%E5%A4%A7%E6%95%B0%E6%8D%AE%E6%8A%80%E6%9C%AF%E5%8F%8A%E7%AE%97%E6%B3%95%E8%A7%A3%E6%9E%90_2015.06.pdf)
[<< Hadoop Yarn权威指南 >>](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/631938%20Hadoop%20YARN%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97.pdf)
[MapReduce运行机制](https://blog.csdn.net/xiushuiguande/article/details/79507956)
[Yarn框架和工作流程研究](https://www.cnblogs.com/itboys/p/9184381.html)