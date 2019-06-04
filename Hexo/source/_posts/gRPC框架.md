---
title: gRPC框架
date: 2019-6-4 12:53:40
categories: 技术储备
tags: [gRPC]
---

----

<!-- more -->

# 1. 简介

## 1.1 认识RPC

### 1.1.1 什么是RPC

RPC(Remote Procedure Call)中文名「远程过程调用」.「过程」也叫方法或函数,「远程」就是说方法不在当前进程里,而是在其他进程或机器上面. 合起来 RPC 就是调用其他进程或机器上面的函数.

在没有网络的时代,程序都是单机版的,所有逻辑都必须在同一个进程里.进程之间就像高楼大厦里面陌生的邻居,大家无法共享,遇到同样的功能只能重复实现一次.显然进程的障碍是逆天的,不符合先进生产力的发展方向,这个时候「进程间通信」的需求出现了,大家要求进程之间能够相互交流,相互共享和调用.这样再写程序,就可以利用进程间通信机制来调用和共享已经存在的功能了.随着网络的出现,进程的隔阂进一步消除,不光同一栋楼里的邻居可以共享资源,其他小区,甚至其他城市的居民都可以通过互联网互相调用,这就是 RPC.概念很容易理解,但是远程和本地的实现原理有很大区别,架构设计者的职责就是设计一个机制让远程调用服务就像调本地服务一样简单,这就是 RPC 框架.

### 1.1.2 基本原理

RPC 框架的目标就是让远程服务调用更加简单,透明,RPC 框架负责屏蔽底层的传输方式(TCP 或者 UDP),序列化方式(XML/Json/ 二进制)和通信细节.服务调用者可以像调用本地接口一样调用远程的服务提供者,而不需要关心底层通信细节和调用过程.

RPC 首要解决的是通讯的问题,主流的 RPC 框架分为基于 HTTP 和基于 TCP 的两种.

基于 HTTP 的 RPC 调用很简单,就和我们访问网页一样,只是它的返回结果更单一(JSON 或 XML).它的优点在于实现简单,标准化和跨语言,比较适合对外提供 OpenAPI 的场景,而它的缺点是 HTTP 协议传输效率较低,短连接开销较大(HTTP 2.0 后有很大改进).

基于 TCP 的 RPC 调用,由于 TCP 协议处于协议栈的下层,能够更加灵活地对协议字段进行定制,减少网络开销,提高性能,实现更大的吞吐量和并发数.但是需要更多地关注底层复杂的细节,跨语言和跨平台难度大,实现的代价更高,它比较适合内部系统之间追求极致性能的场景.

RPC 调用的基本流程:

![调用的基本流程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/1528342678252-8cdab943-2583-4b5c-9f7e-559d05bcff61.png)

1. 调用方(Client)通过本地的 RPC 代理(Proxy)调用相应的接口
2. 本地代理将 RPC 的服务名,方法名和参数等等信息转换成一个标准的 RPC Request 对象交给 RPC 框架
3. RPC 框架采用 RPC 协议(RPC Protocol)将 RPC Request 对象序列化成二进制形式,然后通过 TCP 通道传递给服务提供方 (Server)
4. 服务端(Server)收到二进制数据后,将它反序列化成 RPC Request 对象
5. 服务端(Server)根据 RPC Request 中的信息找到本地对应的方法,传入参数执行,得到结果,并将结果封装成 RPC Response 交给 RPC 框架
6. RPC 框架通过 RPC 协议(RPC Protocol)将 RPC Response 对象序列化成二进制形式,然后通过 TCP 通道传递给服务调用方(Client)
7. 调用方(Client)收到二进制数据后,将它反序列化成 RPC Response 对象,并且将结果通过本地代理(Proxy)返回给业务代码

因为在 TCP 通道里传输的数据只能是二进制形式的,所以必须将数据结构或对象转换成二进制串传递给对方,这个过程就叫「序列化」.而相反,收到对方的二进制串后把它转换成数据结构或对象的过程叫「反序列化」.而序列化和反序列化的规则就叫「协议」.

## 1.2 认识gRPC

### 1.2.1 gRPC简介

gRPC 是一个高性能,开源和通用的 RPC 框架,面向服务端和移动端,基于 HTTP/2 设计.

gRPC 的调用示例如下所示:

![调用示例](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/grpc-01-02.png)

### 1.2.2 gRPC特点

1. 语言中立,支持多种语言;
2. 基于 IDL 文件定义服务,通过 proto3 工具生成指定语言的数据结构,服务端接口以及客户端 Stub;
3. 通信协议基于标准的 HTTP/2 设计,支持双向流,消息头压缩,单 TCP 的多路复用,服务端推送等特性,这些特性使得 gRPC 在移动端设备上更加省电和节省网络流量;
4. 序列化支持 PB(Protocol Buffer)和 JSON,PB 是一种语言无关的高性能序列化框架,基于 HTTP/2 + PB, 保障了 RPC 调用的高性能.

### 1.2.3 gRPC原理

一个RPC框架必须有两个基础的组成部分:数据的序列化和进程数据通信的交互方式.

对于序列化gRPC采用了自家公司开源的Protobuf.

关于进程间的通讯,无疑是Socket.Java方面gRPC同样使用了成熟的开源框架Netty,使用Netty Channel作为数据通道,传输协议使用了HTTP2.

通过以上的分析,可以将一个完整的gRPC流程总结为以下几步:

1. 通过.proto文件定义传输的接口和消息体.
2. 通过protocol编译器生成server端和client端的stub程序.
3. 将请求封装成HTTP2的Stream.
4. 通过Channel作为数据通信通道使用Socket进行数据传输.

# 2. 示例

## 2.1 Maven配置

```pom
<!--添加grpc依赖（包含TCP通信和protobuf序列化和反序列化）-->
<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.21.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.21.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.21.0</version>
    </dependency>
</dependencies>
<!--添加编译proto文件的编译程序和对应的编译插件。-->
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.5.0.Final</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.5.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.7.1:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.21.0:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>6</source>
                <target>6</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 2.2 编写并编译proto文件

### 2.2.1 编写proto文件

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "io.grpc.examples.helloworld";
option java_outer_classname = "HelloWorldProto";
option objc_class_prefix = "HLW";

package helloworld;

// The greeting service definition.
service Greeter {
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
    string name = 1;
}

// The response message containing the greetings
message HelloReply {
    string message = 1;
}
```

### 2.2.2 编译proto文件

参考[java下使用gRPC的helloworld的demo实现](https://blog.csdn.net/u013992365/article/details/81698531)

编译proto文件生成对应的java文件,方法是:

1. 右击Maven.Projects\protobuf\protobuf:compile,选择run,生成用于序列化的java文件.
2. 再右击Maven.Projects\protobuf\protobuf:compile-custom,选择run,生成用于rpc的java代码.

## 2.3 客户端Java代码

```Java
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.examples.helloworld.GreeterGrpc;
import io.grpc.examples.helloworld.HelloReply;
import io.grpc.examples.helloworld.HelloRequest;
import java.util.concurrent.TimeUnit;
import java.util.logging.Logger;

public class HelloWorldClient {
    private final ManagedChannel channel;
    private final GreeterGrpc.GreeterBlockingStub blockingStub;
    private static final Logger logger = Logger.getLogger(HelloWorldClient.class.getName());

    public HelloWorldClient(String host,int port){
        channel = ManagedChannelBuilder.forAddress(host,port)
                .usePlaintext(true)
                .build();
        blockingStub = GreeterGrpc.newBlockingStub(channel);
    }

    public  void greet(String name){
        HelloRequest request = HelloRequest.newBuilder().setName(name).build();
        HelloReply response;
        response = blockingStub.sayHello(request);
        logger.info("Message from gRPC-Server: "+response.getMessage());
    }
    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }

    public static void main(String[] args) throws Exception{
        HelloWorldClient client = new HelloWorldClient("127.0.0.1",50051);
        String user = "world";
        client.greet(user);
        client.shutdown();
    }
}
```

## 2.4 服务端Java代码

```Java
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.examples.helloworld.GreeterGrpc;
import io.grpc.examples.helloworld.HelloReply;
import io.grpc.examples.helloworld.HelloRequest;
import io.grpc.stub.StreamObserver;

import java.io.IOException;
import java.util.logging.Logger;

public class HelloWorldServer {
    private static final Logger logger = Logger.getLogger(HelloWorldServer.class.getName());

    private int port = 50051;
    private Server server;

    private void start() throws IOException {
        server = ServerBuilder.forPort(port)
                .addService(new GreeterImpl())
                .build()
                .start();
        logger.info("Server started, listening on "+ port);
    }

    public  static  void main(String[] args) throws Exception{

        final HelloWorldServer server = new HelloWorldServer();
        server.start();
        Thread.sleep(3600000);
    }

    // 实现 定义一个实现服务接口的类
    private class GreeterImpl extends GreeterGrpc.GreeterImplBase {
        @Override
        public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver){
            HelloReply reply = HelloReply.newBuilder().setMessage(("Hello "+req.getName())).build();
            responseObserver.onNext(reply);
            responseObserver.onCompleted();
            System.out.println("Message from gRPC-Client:" + req.getName());
        }
    }
}
```

# 3. 参考链接

[聊聊 Node.js RPC（一）— 协议](https://www.yuque.com/egg/nodejs/dklip5#kozonf)
[gRPC基于Golang和Java的简单实现](http://jia-shun.cn/2018/08/12/gRPC/)
[gRPC 官方文档中文版](http://doc.oschina.net/grpc?t=58008)
[深入浅出 gRPC 01：gRPC 服务端创建和调用原理](http://jiangew.me/grpc-01/)
[java下使用gRPC的helloworld的demo实现](https://blog.csdn.net/u013992365/article/details/81698531)
[Github官方示例代码:grpc-java/examples/](https://github.com/grpc/grpc-java/tree/v1.21.0/examples)