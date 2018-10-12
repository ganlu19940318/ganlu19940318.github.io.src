---
title: Maven实战(三)--Maven仓库
date: 2018-10-12 10:24:24
categories: Maven基础
tags: [Maven, 基础储备]
---

----

<!-- more -->

# 1. 前言

Java开发过程中, 使用Maven管理依赖. 这是Maven系列文章, 主要记录<<Maven实战>>的学习过程和一些重要知识点, 以方便自己查阅.

# 2. 仓库概述

Maven坐标是一个构件的逻辑表示, 构件的物理表示是文件, Maven通过仓库来统一管理这些文件.

得益于坐标机制, Maven项目能够以统一的方式来使用任何构件, 在此基础上, Maven可以在某个位置统一存储所有Maven项目共享的构建, 这个统一位置就是仓库.

# 3. 仓库分类

Maven中的仓库分为: 本地仓库和远程仓库.

Maven根据坐标寻找构件时, 先查看本地仓库是否存在该构件, 存在则直接使用; 否则就查找远程仓库, 找到之后就下载到本地仓库; 本地和远程都没找到, 直接报错.

![仓库分类](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/6875230-6bf26fb0b4190a62.png)

中央仓库是Maven核心自带的远程仓库, 含绝大多数开源的构件;

私服是在局域网搭建的仓库服务器, 用于代理外部的远程仓库, 可以节省带宽和时间, 内部的项目还能部署到私服供其他项目使用; 使用私服可以加速Maven构建以及提高稳定性, 内网访问不需要依赖于网络.

其他公共服, 如阿里云等.

本地仓库: 通过修改settings.xml来配置, 默认是${user.home}/.m2/repository.

```xml
<!-- localRepository
 | The path to the local repository maven will use to store artifacts
 | Default: ${user.home}/.m2/repository
<localRepository>/path/to/local/repo</localRepository>
-->
```

构件进入本地仓库有两种方式: Maven从远程仓库下载到本地仓库; 通过在项目执行mvn install安装到本地.

对Maven而言, 用户的本地仓库只有一个, 但可以配置访问很多远程仓库. 而中央仓库是默认的远程仓库, 在$M2_HOME/lib/maven-model-builder-{version}.jar的org/apache/maven/model/pom-4.0.0.xml文件定义了, 该POM也被称为超级POM.

# 4. 仓库的布局

构件在Maven仓库里的存储路径为:{groupId}/{artifactId}/{version}/{artifactId-version.packaging}

# 5. 远程仓库的配置

通过POM文件可以配置远程仓库.

```pom
<repositories>
    <repository>
        <id>jboss</id>
        <name>jboss repository</name>
        <url>http://repository.jboss.com/maven2/</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <layout>default</layout>
    </repository>
</repositories>
```

仓库id必须唯一!!!不唯一则会覆盖!!!
release的enable值为true表示开启JBoss仓库的发布版本下载支持.
snapshots的enable值为false表示关闭JBoos仓库的快照版本下载支持.
layout元素值为default,表示仓库的布局是Maven2和Maven3的默认布局, 而不是Maven1的布局.

对于release和snapshot来说, 除了enable, 它们还包括另外两个子元素updatePolicy和checksumPolicy.

```pom
<snapshots>
    <enabled>true</enabled>
    <updatePolicy>daily</updatePolicy>
    <checksumPolicy>ignore</checksumPolicy>
</snapshots>
```

updatePolicy配置Maven从远程仓库检查更新的频率,默认是daily.
checksumPolicy配置Maven检查检验和文件的策略.

## 5.1 远程仓库认证

通过修改settings.xml配置远程仓库认证.

```xml
<settings>
    ...
    <!--配置远程仓库认证信息-->
    <servers>
        <server>
            <id>jboss</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
    </servers>
    ...
</settings>
```

# 6. 部署构件至远程仓库

```pom
<distributionManagement>
    <repository>
        <id>releases</id>
        <name>public</name>
        <url>http://127.0.0.1:8081/nexus/content/repositories/releases</url>
    </repository>
    <snapshotRepository>
        <id>snapshots</id>
        <name>Snapshots</name>
        <url>http://127.0.0.1:8081/nexus/content/repositories/snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

distributionManagement包含repository和snapshotRepository子元素, 前者表示发布版本(稳定版本)构件的仓库,后者表示快照版本(开发测试版本)的仓库. 这两个元素都需要配置id, name和url, id为远程仓库的唯一标识, name是为了方便人阅读, 关键的url表示该仓库的地址.

往远程仓库部署构件的时候, 往往需要认证, 配置认证的方式同上.

配置正确后, 运行命令mvn clean deploy, Maven就会将项目构建输出的构件部署到配置对应的远程仓库, 如果项目当前的版本是快照版本, 则部署到快照版本的仓库地址, 否则就部署到发布版本的仓库地址.

Maven会根据模块的版本号(pom文件中的version)中是否带有-SNAPSHOT来判断是快照版本还是正式版本.

# 7. 从仓库解析依赖的机制

当本地仓库没有依赖构件的时候, maven会自动从远程仓库下载; 当依赖版本为快照版本的时候Maven会自动找到最新的快照. 这背后的依赖解析机制可以概括如下:

1. 当依赖的范围是system的时候Maven直接从本地文件系统解析构件.

2. 根据依赖坐标计算仓库路径后, 尝试直接从本地仓库寻找构件, 如果发现相应构件则解析成功.

3. 在本地仓库不存在相应构件的情况下, 如果依赖的版本显示的发布版本构件, 如1.2,2.1-beta-1等, 则遍历所有的远程仓库, 发现后下载并解析使用.

4. 如果依赖的版本是RELEASE或者LATEST, 则基于更新策略读取所有远程仓库的元数据groupId/artifactId/maven-metadata.xml,将其与本地仓库对应元数据合并后计算出RELEASE或者LATEST真实的值, 然后基于这个真实的值检查本地和远程仓库如步骤2和3.

5. 如果依赖的版本是SNAPSHOT, 则基于更新策略读取所有远程仓库的元数据groupId/artifactId/version/maven-metadata.xml, 将其与本地仓库对应元数据合并后得到最新快照版本的值, 然后基于该值检查本地仓库或者从远程仓库下载.

6. 如果最后解析得到的构件版本时间是时间戳格式的快照, 如1.4.1-20091104.121450-121, 则复制其时间戳格式的文件至非时间戳格式, 如SNAPSHOT, 并使用该非时间戳格式的构件.

# 8. 镜像

如果仓库X可以提供仓库Y存储的所有内容, 那么就可以认为X是Y的一个镜像. 换句话说, 任何一个可以从仓库Y获得的构件, 都能够从它的镜像中获取. 举个例子, http://maven.oschina.net/content/groups/public/ 是中央仓库http://repo1.maven.org/maven2/ 在中国的镜像, 由于地理位置的因素, 该镜像往往能够提供比中央仓库更快的服务. 因此, 可以配置Maven使用该镜像来替代中央仓库, 编辑settings.xml, 代码如下:

```xml
<mirrors>
    <mirror>
        <id>maven.oschina.net</id>
        <name>maven mirror in China</name>
        <url>http://maven.oschina.net/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
</mirrors>
```

该例中, mirrorOf的值为central, 表示该配置为中央仓库的镜像, 任何对于中央仓库的请求都会转至该镜像, 用户也可以使用同样的方法配置其他仓库的镜像.id表示镜像的唯一标识符, name表示镜像的名称, url表示镜像的地址.

关于镜像的一个更为常见的用法是结合私服. 由于私服可以代理任何外部的公共仓库(包括中央仓库), 因此, 对于组织内部的Maven用户来说, 使用一个私服地址就等于使用了所有需要的外部仓库, 这可以将配置集中到私服, 从而简化Maven本身的配置. 在这种情况下, 任何需要的构件都可以从私服获得, 私服就是所有仓库的镜像. 这时, 可以配置这样的一个镜像:

```xml
<!--配置私服镜像-->
<mirrors>
    <mirror>  
        <id>nexus</id>  
        <name>internal nexus repository</name>  
        <url>http://192.168.0.1:8081/nexus/content/groups/public/</url>  
        <mirrorOf>*</mirrorOf>  
    </mirror>  
</mirrors>
```

## 8.1 特别强调

<font color=red>需要注意的是, 由于镜像仓库完全屏蔽了被镜像仓库, 当镜像仓库不稳定或者停止服务的时候, Maven仍将无法访问被镜像仓库, 因而将无法下载构件.</font>

# 9. 参考链接

<<Maven实战>>(许晓斌)