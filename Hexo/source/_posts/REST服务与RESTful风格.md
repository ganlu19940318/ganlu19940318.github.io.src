---
title: REST服务与RESTful风格
date: 2018-09-24 15:33:30
categories: 后台开发
tags: [基础储备]
---

----

<!-- more -->

# 1. 前言

进行后台开发的时候, 我们经常说, 我们的接口是Restful的风格, 那Restful到底是什么呢?Restful风格有哪些特点呢?REST又跟Restful有什么关系?这篇文章主要是记录Restful API风格及其相关的一些知识点.

# 2. 什么是REST

REST: representational state transfer 表述性状态转移, 是一种架构风格.
它是轻量级,跨平台,跨语言的架构设计; 它是一种设计风格,不是一种标准, 是一种思想.

# 3. REST原则

1. 网络上的所有事物都被抽象为资源
2. 每个资源都有一个唯一的资源标识符
3. 同一个资源具有多种表现形式(xml,json等)
4. 对资源的各种操作不会改变资源标识符
5. 所有的操作都是无状态的

# 4. 关于RESTful

RESTful: 遵守了REST原则的web服务
理解: REST与RESTful相比, 多了一个ful, 就英语层面来说是一个形容词, RESTful翻译为中文为: "REST式的".
是REST式的是什么意思呢?
意思是 是REST式的应用, REST风格的web服务也是REST式的应用, REST式的web服务是一种ROA(The Resource-Oriented Architecture)(面向资源的架构).

# 5. 为什么会出现RESTful

----
在RESTful之前的操作:
http://127.0.0.1/user/query/1 GET  根据用户id查询用户数据
http://127.0.0.1/user/save POST 新增用户
http://127.0.0.1/user/update POST 修改用户信息
http://127.0.0.1/user/delete GET/POST 删除用户信息
RESTful用法:
http://127.0.0.1/user/1 GET  根据用户id查询用户数据
http://127.0.0.1/user  POST 新增用户
http://127.0.0.1/user  PUT 修改用户信息
http://127.0.0.1/user  DELETE 删除用户信息

----
之前的操作是没有问题的,大神认为是有问题的,有什么问题呢?你每次请求的接口或者地址,都在做描述,例如查询的时候用了query,新增的时候用了save,其实完全没有这个必要,我使用了get请求,就是查询.使用post请求,就是新增的请求,我的意图很明显,完全没有必要做描述,这就是为什么有了RESTful.

# 6. 如何设计Restful风格的API

## 6.1 路径设计

在RESTdul架构中, 每个网址代表一种资源(resource), 所以网址中不能有动词, 只能有名词, 而且所用的名词往往与数据库的表名对应, 一般来说, 数据库中的表都是同种记录的"集合"(collection), 所以API中的名词也应该使用复数. 
举例来说, 有一个API提供动物园(zoo)的信息, 还包括各种动物和雇员的信息, 则它的路径应该设计成下面这样.

```vim
https://api.example.com/v1/zoos
https://api.example.com/v1/animals
https://api.example.com/v1/employees
```

## 6.2 HTTP动词设计

对于资源的具体操作类型, 由HTTP动词表示, 常用的HTTP动词如下:

| 请求方式 | 含义                                 |
| -------- | -------------------------------------- |
| GET      | 获取资源（一项或多项）      |
| POST     | 新建资源                           |
| PUT      | 更新资源（客户端提供改变后的完整资源） |
| DELETE   | 删除资源                           |

如何通过路径和http动词获悉要调用的功能:

| 请求方式               | 含义                                             |
| -------------------------- | -------------------------------------------------- |
| GET /zoos                  | 列出所有动物园                              |
| POST /zoos                 | 新建一个动物园                              |
| GET /zoos/ID               | 获取某个指定动物园的信息               |
| PUT /zoos/ID               | 更新某个指定动物园的信息（提供该动物园的全部信息） |
| DELETE /zoos/ID            | 删除某个动物园                              |
| GET /zoos/ID/animals       | 列出某个指定动物园的所有动物         |
| DELETE /zoos/ID/animals/ID | 删除某个指定动物园的指定动物         |

## 6.3 常用状态码

200 OK - [GET]: 服务器成功返回用户请求的数据, 该操作是幂等的.
201 CREATED - [POST/PUT/PATCH]: 用户新建或修改数据成功.
202 Accepted - [*]:表示一个请求已经进入后台排队(异步任务)
204 NO CONTENT - [DELETE]: 用户删除数据成功.
400 INVALID REQUEST - [POST/PUT/PATCH]: 用户发出的请求有错误, 服务器没有进行新建或修改数据的操作.
401 Unauthorized - [*]: 表示用户没有权限(令牌, 用户名, 密码错误).
403 Forbidden - [*]: 表示用户得到授权(与401错误相对), 但是访问是被禁止的.
404 NOT FOUND - [*]: 用户发出的请求针对的是不存在的记录, 服务器没有进行操作, 该操作是幂等的.
406 Not Acceptable - [GET]: 用户请求的格式不可得(比如用户请求JSON格式, 但是只有XML格式).
410 Gone -[GET]: 用户请求的资源被永久删除, 且不会再得到的.
422 Unprocesable entity - [POST/PUT/PATCH]:  当创建一个对象时, 发生一个验证错误.
500 INTERNAL SERVER ERROR - [*]: 服务器发生错误, 用户将无法判断发出的请求是否成功.

## 6.4 版本号

应该将API的版本号放入URL

```vim
如: https://api.example.com/v1/
```

另一种做法是, 将版本号放在HTTP头信息中, 但不如放入URL方便和直观. Github采用这种做法.

## 6.5 其他

服务器返回的数据格式, 应该尽量使用JSON, 避免使用XML.

# 7. 参考链接

[什么是rest？什么是restful？它们之间是什么关系](https://blog.csdn.net/maxiao124/article/details/79897229)
[【Restful】三分钟彻底了解Restful最佳实践](https://blog.csdn.net/chenxiaochan/article/details/73716617)
[理解并设计rest/restful风格接口](https://blog.csdn.net/mawming/article/details/52381740)