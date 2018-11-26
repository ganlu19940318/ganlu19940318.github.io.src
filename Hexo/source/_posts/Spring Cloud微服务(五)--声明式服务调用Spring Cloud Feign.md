---
title: Spring Cloud微服务(五)--声明式服务调用Spring Cloud Feign
date: 2018-11-25 20:08:57
categories: Spring Cloud
tags: [Spring Cloud, 微服务, 基础储备]
---

----

<!-- more -->

# 1. 前言

本文是 << Spring Cloud微服务实战 >> 学习笔记, 以便自己查阅.

# 2. 概述

Spring Cloud Feign是一套基于Netflix Feign实现的声明式服务调用客户端. 它使得编写Web服务客户端变得更加简单. 我们只需要通过创建接口并用注解来配置它既可完成对Web服务接口的绑定. 它具备可插拔的注解支持, 包括Feign注解, JAX-RS注解. 它也支持可插拔的编码器和解码器. Spring Cloud Feign还扩展了对Spring MVC注解的支持, 同时还整合了Ribbon和Eureka来提供均衡负载的HTTP客户端实现,

<font color=red>说白了, Spring Cloud Feign = Spring Cloud Ribbon + Spring Cloud Hystrix + 其他扩展.</font>

# 3. 示例

个人觉得, Spring Cloud Feign最大的优点就是简化了服务调用的代码, 传统的服务调用, 需要通过RestTemplate操作, 无论是入参构造还是返回值的解析, 都是繁琐的重复过程, 而Spring Cloud Feign简化了这个过程.

示例代码引自网络.

## 3.1 引入依赖

```java
<dependencies>
    ...
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-feign</artifactId>
    </dependency>
</dependencies>
```

## 3.2 修改主类

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }
}
```

## 3.3 创建Feign客户端

```java
@FeignClient("eureka-client")
public interface DcClient {
    @GetMapping("/dc")
    String consumer();

}
```

## 3.4 服务调用

```java
@RestController
public class DcController {

    @Autowired
    DcClient dcClient;

    @GetMapping("/consumer")
    public String dc() {
        return dcClient.consumer();
    }

}
```

# 4. 参考链接

<< Spring Cloud微服务实战 >>
[Spring Cloud构建微服务架构：服务消费（Feign）【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-2-3/)