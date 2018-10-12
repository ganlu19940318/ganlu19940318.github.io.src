---
title: Maven实战(一)--Maven配置和使用
date: 2018-10-11 15:47:18
categories: Maven基础
tags: [Maven, 基础储备]
---

----

<!-- more -->

# 1. 前言

Java开发过程中, 使用Maven管理依赖. 这是Maven系列文章, 主要记录<<Maven实战>>的学习过程和一些重要知识点, 以方便自己查阅.

# 2. Maven的安装

基于Windows 10系统安装和配置Maven.

1. 安装JDK

JDK的安装不做过多赘述.

2. 下载Maven

link: http://maven.apache.org/download.cgi

3. 配置环境变量

解压并配置环境变量.
新建环境变量M2_HOME, 值为Maven安装目录, 比如, 我的安装目录是D:\Software\Maven\apache-maven-3.5.4
在环境变量Path里面添加新的值, %M2_HOME%\bin

4. 测试配置

在命令行输入mvn -v

5. 版本升级

解压并修改M2_HOME的值即可.

# 3. ~/.m2目录解读

~是指代用户目录.

比如windows 10下, 我的~目录就是C:\Users\ganlu

1. ~/.m2/repository是仓库位置

2. 一般情况下, 需要复制M2_HOME/conf/settings.xml文件到~/.m2目录下, 作为当前用户的Maven配置文件

# 4. 设置HTTP代理

修改settings.xml文件

```XML
<settings>
   ...
   <proxies>
      <proxy>
         <id>my-proxy</id>
         <active>true</active>
         <protocol>http</protocol>
         <host>100.10.10.10</host>
         <port>3333</port>
         <username>ganlu</username>
         <password>123456</password>
         <nonProxyHosts>repository.mycom.com|*.google.com</nonProxyHosts>
      </proxy>
    </proxies>
   ...
</settings>
```

activate表示激活该代理;
protocol表示所使用的代理协议;
nonProxyHosts用来设置不需要使用代理的域名.

# 5. 参考链接

<<Maven实战>>(许晓斌)