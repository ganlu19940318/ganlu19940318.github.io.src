---
title: Linux常用命令
date: 2018-09-24 15:46:39
categories: 后台开发
tags: [基础储备, 解决方案, 收藏夹]
---

----

<!-- more -->

# 1. 前言

后台开发过程中, Linux的熟练使用有助于提高开发效率, 自动化部署等. 这篇文章主要是记录工作中会遇到的一些有用的命令, 持续更新.

# 2. 常用命令

## 2.1 网络流量监控

```linux
dstat -nf
```

## 2.2 查看端口被占用

```linux
lsof -i:8080
```

## 2.3 生成指定大小的文件

```linux
// 生成100M的文件
dd if=/dev/zero of=filename bs=1M count=100
```
