---
title: HDFS
date: 2019-6-5 18:20:52
categories: 技术储备
tags: [HDFS]
---

----

<!-- more -->

# 1. 入门

## 1.1 HDFS是什么

HDFS(Hadoop Distributed File System, Hadoop分布式文件系统).

## 1.2 HDFS优缺点

优点：

1. 高容错性:数据自动保存多个副本,它通过增加副本的形式,提高容错性,某一个副本丢失以后,它可以自动恢复.
2. 适合批处理:它是通过移动计算而不是移动数据实现的,它会把数据位置暴露给计算框架.
3. 适合大数据处理:处理数据达到 GB,TB,甚至PB级别的数据,能够处理百万规模以上的文件数量,数量相当之大,能够处理10K节点的规模.
4. 流式文件访问:一次写入,多次读取.文件一旦写入不能修改,只能追加,它能保证数据的一致性.
5. 可构建在廉价机器上,它通过多副本机制,提高可靠性.它提供了容错和恢复机制,比如某一个副本丢失,可以通过其它副本来恢复.

缺点:

1. 低延时数据访问:比如毫秒级的来存储数据,这是不行的,它做不到.它适合高吞吐率的场景,就是在某一时间内写入大量的数据.但是它在低延时的情况下是不行的,比如毫秒级以内读取数据,这样它是很难做到的.
2. 小文件存储:存储大量小文件(这里的小文件是指小于HDFS系统的Block大小的文件,默认64M)的话,它会占用 NameNode大量的内存来存储文件,目录和块信息.这样是不可取的,因为NameNode的内存总是有限的.小文件存储的寻道时间会超过读取时间,它违反了HDFS的设计目标.
3. 并发写入,文件随机修改.一个文件只能有一个写,不允许多个线程同时写.仅支持数据 append,不支持文件的随机修改.

## 1.3 HDFS架构

HDFS 采用Master/Slave的架构来存储数据,这种架构主要由四个部分组成,分别为HDFS Client,NameNode,DataNode和Secondary NameNode.

![HDFS架构图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190608023623.jpg)

1. Client
文件切分.文件上传 HDFS 的时候,Client 将文件切分成 一个一个的Block,然后进行存储.
与 NameNode 交互,获取文件的位置信息.
与 DataNode 交互,读取或者写入数据.
Client 提供一些命令来管理 HDFS,比如启动或者关闭HDFS.
Client 可以通过一些命令来访问 HDFS.

2. NameNode(Master)
管理 HDFS 的名称空间
管理数据块(Block)映射信息
配置副本策略
处理客户端读写请求.

3. DataNode(Slave)
NameNode 下达命令,DataNode 执行实际的操作.
存储实际的数据块.
执行数据块的读/写操作.

4. Secondary NameNode
并非 NameNode 的热备.当NameNode 挂掉的时候,它并不能马上替换 NameNode 并提供服务.
辅助 NameNode,分担其工作量.
定期合并 fsimage和fsedits,并推送给NameNode.
在紧急情况下,可辅助恢复 NameNode.

## 1.4 HDFS读取文件流程

![HDFS读取文件流程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190608023949.jpg)

1. 首先调用FileSystem对象的open方法,其实获取的是一个DistributedFileSystem的实例.
2. DistributedFileSystem通过RPC获得文件的第一批block的locations,同一block按照重复数会返回多个locations,这些locations按照hadoop拓扑结构排序,距离客户端近的排在前面.
3. 前两步会返回一个FSDataInputStream对象,该对象会被封装成 DFSInputStream对象,DFSInputStream可以方便的管理datanode和namenode数据流.客户端调用read方法,DFSInputStream就会找出离客户端最近的datanode并连接datanode.
4. 数据从datanode源源不断的流向客户端.
5. 如果第一个block块的数据读完了,就会关闭指向第一个block块的datanode连接,接着读取下一个block块.这些操作对客户端来说是透明的,从客户端的角度来看只是读一个持续不断的流.
6. 如果第一批block都读完了,DFSInputStream就会去namenode拿下一批blocks的location,然后继续读,如果所有的block块都读完,这时就会关闭掉所有的流.

## 1.5 HDFS写入文件流程

![HDFS写入文件流程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190608024256.jpg)

1. 客户端通过调用 DistributedFileSystem 的create方法,创建一个新的文件.
2. DistributedFileSystem 通过 RPC调用 NameNode,去创建一个没有blocks关联的新文件.创建前,NameNode 会做各种校验,比如文件是否存在,客户端有无权限去创建等.如果校验通过,NameNode 就会记录下新文件,否则就会抛出IO异常.
3. 前两步结束后会返回 FSDataOutputStream 的对象,和读文件的时候相似,FSDataOutputStream 被封装成 DFSOutputStream,DFSOutputStream 可以协调 NameNode和 DataNode.客户端开始写数据到DFSOutputStream,DFSOutputStream会把数据切成一个个小packet,然后排成队列 data queue.
4. DataStreamer 会去处理接受 data queue,它先问询 NameNode 这个新的 block 最适合存储的在哪几个DataNode里,比如重复数是3,那么就找到3个最适合的 DataNode,把它们排成一个 pipeline.DataStreamer 把 packet 按队列输出到管道的第一个 DataNode 中,第一个 DataNode又把 packet 输出到第二个 DataNode 中,以此类推.
5. DFSOutputStream 还有一个队列叫 ack queue,也是由 packet 组成,等待DataNode的收到响应,当pipeline中的所有DataNode都表示已经收到的时候,这时akc queue才会把对应的packet包移除掉.
6. 客户端完成写数据后,调用close方法关闭写入流.
7. DataStreamer 把剩余的包都刷到 pipeline 里,然后等待 ack 信息,收到最后一个 ack 后,通知 DataNode 把文件标示为已完成.

# 2. 参考文献

[<<Hadoop权威指南>>](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/626566%20Hadoop%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97%20%E5%A4%A7%E6%95%B0%E6%8D%AE%E7%9A%84%E5%AD%98%E5%82%A8%E4%B8%8E%E5%88%86%E6%9E%90-%E7%AC%AC4%E7%89%88-%E4%BF%AE%E8%AE%A2%E7%89%88-%E5%8D%87%E7%BA%A7%E7%89%88.pdf)
[初步掌握HDFS的架构及原理](https://www.cnblogs.com/codeOfLife/p/5375120.html#hadoop2x%E6%96%B0%E7%89%B9%E6%80%A7)