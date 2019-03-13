---
title: Java NIO和AIO
date: 2019-3-12 20:42:57
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 0. 前言

这里只是简单介绍和了解Java NIO和AIO,以后补上深入的部分.

# 1. 网络NIO

Java NIO是New IO的简称,它是一种可以替代Java IO的一套新的IO机制.使用NIO技术可以大大提高线程的使用效率.

## 1.1 基于Socket的服务端的多线程模式

传统的网络IO是基于Socket的多线程模式.

服务器会为每一个客户端连接启用一个线程,这个新的线程将全心全意为这个客户端服务.为了接受客户端连接,服务器还会额外使用一个派发线程.但是这个模式有一个重大的**弱点**,那就是它倾向于让CPU进行IO等待.比如服务器要先读入客户端的输入,而客户端缓慢的处理速度(也可能是拥塞的网络环境造成的),使得服务器花费了不少等待时间.

![Socket的多线程模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190313103622.jpg)

## 1.2 使用NIO进行网络编程

NIO就可以将上面的网络IO等待时间从业务处理线程中抽取出来.

具体而言, NIO里面有一个关键组件Channel(通道), Channel有点类似于流, 一个Channel可以和文件或者网络Socket对应, 如果Channel对应着一个Socket, 那么往这个Channel中写数据, 就等同于向Socket中写入数据.

和Channel一起使用的另一个重要组件就是Buffer, 可以简单地把Buffer理解成一个内存区域或者byte数组.数据需要包装成Buffer的形式才能和Channel交互(写入或者读取).

另一个和Channel密切相关的是Selector(选择器).在Channel的众多实现中,有一个SelectableChannel实现,表示可被选择的通道,任何一个SelectableChannel都可以将自己注册到一个Selector中,这样,这个Channel就能被Selector所管理,而一个Selector可以管理多个SelectableChannel.当SelectableChannel的数据准备好时,Selector就会接到通知,得到那些已经准备好的数据.而SocketChannel就是SelectableChannel就是的一种.

![NIO](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190313105438.jpg)

一个Selector可以由一个线程进行管理,而一个SocketChannel则表示一个客户端连接.当客户端连接的数据没有准备好时,Selector就会处于等待状态,而一旦有任何一个SocketChannel准备好了数据,Selector就能立即得到通知,获得数据进行处理.

# 2. AIO

AIO是异步IO的缩写, 即Asynchronized. 虽然NIO在网络操作中提供了非阻塞的方法, 但是NIO的IO行为还是同步的. 对于NIO来说,我们的业务线程是在IO操作准备好时,得到通知,接着就由着这个线程自行进行IO操作,IO操作本身还是同步的.

对于AIO来说,它不是在IO准备好时再通知线程,而是在IO操作已经完成后再给线程发出通知.因此,AIO是完全不会阻塞的.此时,我们的业务逻辑将变成一个回调函数,等待IO操作完成后,由系统自动触发.

# 3. 参考文献

<< Java高并发程序设计 >>(葛一鸣 郭超)