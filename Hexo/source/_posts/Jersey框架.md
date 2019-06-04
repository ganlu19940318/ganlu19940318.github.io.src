---
title: Jersey框架
date: 2019-6-3 22:16:30
categories: 技术储备
tags: [Jersey]
---

----

<!-- more -->

# 1. 简介

开发RESTful WebService意味着需要抽象和实现底层的客户端-服务器通信细节,如果没有一个好的工具包可用,这将是一个困难的任务.

为了简化使用JAVA开发RESTful WebService及其客户端, 一个轻量级的标准被提出: JAX-RS API.

Jersey RESTful WebService框架是一个开源的,产品级别的JAVA框架,支持JAX-RS API并且是一个JAX-RS(JSR 311和 JSR 339)的参考实现

Jersey不仅仅是一个JAX-RS的参考实现,Jersey提供自己的API,其API继承自JAX-RS,提供更多的特性和功能以进一步简化RESTful service和客户端的开发.

# 2. 服务端使用示例

## 2.1 通过Maven导入必要的依赖包

```pom
<dependencies>
    <!-- https://mvnrepository.com/artifact/com.sun.jersey/jersey-server -->
    <dependency>
        <groupId>com.sun.jersey</groupId>
        <artifactId>jersey-server</artifactId>
        <version>1.19.4</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/com.sun.jersey/jersey-grizzly2 -->
    <dependency>
        <groupId>com.sun.jersey</groupId>
        <artifactId>jersey-grizzly2</artifactId>
        <version>1.19.4</version>
    </dependency>
</dependencies>
```

## 2.2 服务端程序

```Java
import com.sun.jersey.api.container.grizzly2.GrizzlyServerFactory;
import com.sun.jersey.api.core.PackagesResourceConfig;
import com.sun.jersey.api.core.ResourceConfig;
import com.sun.jersey.spi.resource.Singleton;
import org.glassfish.grizzly.http.server.HttpServer;
import javax.ws.rs.*;
import javax.ws.rs.core.*;
import java.net.URI;
import java.util.Iterator;

@Singleton
@Path("service")
public class Server {
    @Path("{sub_path:[a-zA-Z0-9]*}")
    @GET
    @Consumes({MediaType.TEXT_PLAIN, MediaType.APPLICATION_JSON})
    @Produces(MediaType.TEXT_PLAIN)
    public String getResourceName(
            @PathParam("sub_path") String resourceName,
            @DefaultValue("Just a test!") @QueryParam("desc") String description,
            @Context Request request,
            @Context UriInfo uriInfo,
            @Context HttpHeaders httpHeader) {
        System.out.println(this.hashCode());

        // 将HTTP请求打印出来
        System.out.println("****** HTTP request ******");
        StringBuilder strBuilder = new StringBuilder();
        strBuilder.append(request.getMethod() + " ");
        strBuilder.append(uriInfo.getRequestUri().toString() + " ");
        strBuilder.append("HTTP/1.1[\\r\\n]");
        System.out.println(strBuilder.toString());
        MultivaluedMap<String, String> headers = httpHeader.getRequestHeaders();
        Iterator<String> iterator = headers.keySet().iterator();
        while(iterator.hasNext()){
            String headName = iterator.next();
            System.out.println(headName + ":" + headers.get(headName) + "[\\r\\n]");
        }
        System.out.println("[\\r\\n]");
        String responseStr =resourceName + "[" + description + "]";
        return responseStr;
    }

    public static void main(String[] args) throws Exception{
        URI uri = UriBuilder.fromUri("http://127.0.0.1").port(10000).build();
        ResourceConfig rc = new PackagesResourceConfig("");
        HttpServer server = GrizzlyServerFactory.createHttpServer(uri, rc);
        server.start();
        Thread.sleep(1000 * 1000);
    }
}
```

# 3. 客户端使用示例(可选)

## 3.1 通过Maven导入必要的依赖包

```pom
<dependencies>
    <!-- https://mvnrepository.com/artifact/com.sun.jersey/jersey-grizzly2 -->
    <dependency>
        <groupId>com.sun.jersey</groupId>
        <artifactId>jersey-grizzly2</artifactId>
        <version>1.19.4</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/com.sun.jersey/jersey-client -->
    <dependency>
        <groupId>com.sun.jersey</groupId>
        <artifactId>jersey-client</artifactId>
        <version>1.19.4</version>
    </dependency>
</dependencies>
```

## 3.2 客户端程序

```Java
import com.sun.jersey.api.client.Client;
import com.sun.jersey.api.client.ClientResponse;
import com.sun.jersey.api.client.WebResource;
import com.sun.jersey.api.client.config.ClientConfig;
import com.sun.jersey.api.client.config.DefaultClientConfig;

import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.MultivaluedMap;
import javax.ws.rs.core.UriBuilder;
import java.net.URI;
import java.util.Iterator;

public class Client {
    public static void main(String[] args) {
        // 要使用Jersey Client API，必须首先创建Client的实例
        // 有以下两种创建Client实例的方式
        // 方式一
        ClientConfig cc = new DefaultClientConfig();
        cc.getProperties().put(ClientConfig.PROPERTY_CONNECT_TIMEOUT, 10*1000);
        // Client实例很消耗系统资源，需要重用
        // 创建web资源，创建请求，接受响应都是线程安全的
        // 所以Client实例和WebResource实例可以在多个线程间安全的共享
        Client client = Client.create(cc);
        // 方式二
        // Client client = Client.create();
        // client.setConnectTimeout(10*1000);
        // client.getProperties().put(ClientConfig.PROPERTY_CONNECT_TIMEOUT, 10*1000);

        // WebResource将会继承Client中timeout的配置
        // 由Client发起请求到Server
        // 方式一
        // WebResource resource = client.resource("http://127.0.0.1:10000/service/sean?desc=description");
        // String str = resource
        //       .accept(MediaType.TEXT_PLAIN)
        //       .type(MediaType.TEXT_PLAIN)
        //       .get(String.class);
        // System.out.println("String:" + str);
        // 方式二
        URI uri = UriBuilder.fromUri("http://127.0.0.1/service/sean").port(10000)
                .queryParam("desc", "description").build();
        WebResource resource = client.resource(uri);

        // header方法可用来添加HTTP头
        ClientResponse response = resource.header("auth", "123456")
                .accept(MediaType.TEXT_PLAIN)
                .type(MediaType.TEXT_PLAIN)
                .get(ClientResponse.class);
        // 将HTTP响应打印出来
        System.out.println("****** HTTP response ******");
        StringBuilder strBuilder = new StringBuilder();
        strBuilder.append("HTTP/1.1 ");
        strBuilder.append(response.getStatus() + " ");
        strBuilder.append(response.getStatusInfo() + "[\\r\\n]");
        System.out.println(strBuilder.toString());
        MultivaluedMap<String, String> headers = response.getHeaders();
        Iterator<String> iterator = headers.keySet().iterator();
        while(iterator.hasNext()){
            String headName = iterator.next();
            System.out.println(headName + ":" + headers.get(headName) + "[\\r\\n]");
        }
        System.out.println("[\\r\\n]");
        System.out.println(response.getEntity(String.class) + "[\\r\\n]");
    }
}
```

# 4. 参考链接

[Jersey框架一：Jersey RESTful WebService框架简介](https://blog.csdn.net/a19881029/article/details/43056429)
[Jersey框架二：Jersey对JSON的支持](https://blog.csdn.net/a19881029/article/details/43205457)