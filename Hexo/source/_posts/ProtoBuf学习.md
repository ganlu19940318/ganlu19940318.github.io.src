---
title: ProtoBuf学习
date: 2019-6-3 12:10:34
categories: 技术储备
tags: [ProtoBuf]
---

----

<!-- more -->

# 1. 简介

## 1.1 ProtoBuf是什么

Protocol Buffers是一种平台无关, 语言无关, 可扩展且轻便高效的序列化数据结构的协议, 可以用于网络通信和数据存储.

## 1.2 ProtoBuf, XML, JSON比较

将ProtoBuf, XML, JSON三者放到一起去比较, 应该区分两个维度.
一个是**数据结构化**, 另一个是**数据序列化**.
这里的数据结构化主要面向开发或业务层面, 数据序列化面向通信或存储层面, 当然数据序列化也需要"结构"和"格式", 所以这两者之间的区别主要在于面向领域和场景不同, 一般要求和侧重点也会有所不同. 
数据结构化侧重人类可读性甚至有时会强调语义表达能力, 而数据序列化侧重效率和压缩.

XML, JSON, ProtoBuf 都具有数据结构化和数据序列化的能力
XML, JSON 更注重数据结构化, 关注人类可读性和语义表达能力.
ProtoBuf 更注重数据序列化, 关注效率, 空间, 速度, 人类可读性差, 语义表达能力不足(为保证极致的效率, 会舍弃一部分元信息)

# 2. 语法

参考[Protocol Buffers 3.0 技术手册](https://blog.csdn.net/shensky711/article/details/69696392)
离线备份:[Protocol Buffers 3.0 技术手册](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/Protocol.mhtml)

# 3. 参考链接

[[索引]文章索引](https://www.jianshu.com/p/b33ca81b19b5)
[[翻译] ProtoBuf 官方文档（二）- 语法指引（proto2）](https://www.jianshu.com/p/6f68fb2c7d19)
[Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-cn)
[Protocol Buffers 3.0 技术手册](https://blog.csdn.net/shensky711/article/details/69696392)