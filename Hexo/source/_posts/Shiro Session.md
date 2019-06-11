---
title: Shiro Session
date: 2019-6-6 20:44:29
categories: 技术储备
tags: [Shiro]
---

----

<!-- more -->

# 1. 会话

所谓会话,即用户访问应用时保持的连接关系,在多次交互中应用能够识别出当前访问的用户是谁,且可以在多次交互中保存一些数据.

## 1.1 Java使用

```pom
<dependencies>
    <dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-core</artifactId>
        <version>1.4.0</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>
```

```Java
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.mgt.DefaultSecurityManager;
import org.apache.shiro.realm.SimpleAccountRealm;
import org.apache.shiro.session.Session;
import org.apache.shiro.subject.Subject;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;

public class Main {
    SimpleAccountRealm simpleAccountRealm = new SimpleAccountRealm();

    @Before // 在方法开始前添加一个用户
    public void addUser() {
        simpleAccountRealm.addAccount("ganlu", "123456");
    }

    @Test
    public void testAuthentication() throws Exception {
        // 1.构建SecurityManager环境
        DefaultSecurityManager defaultSecurityManager = new DefaultSecurityManager();
        defaultSecurityManager.setRealm(simpleAccountRealm);
        // 2.主体提交认证请求
        SecurityUtils.setSecurityManager(defaultSecurityManager); // 设置SecurityManager环境
        Subject subject = SecurityUtils.getSubject(); // 获取当前主体
        UsernamePasswordToken token = new UsernamePasswordToken("ganlu", "123456");
        subject.login(token); // 登录
        Session session = subject.getSession(); // 获得Session
        System.out.println("Id : "+session.getId()); // 当前会话的唯一标识
        System.out.println("Host : "+session.getHost()); // 当前subject的主机地址
        System.out.println("Timeout : "+session.getTimeout()); // 获取当前 Session 的过期时间;如果不设置默认是会话管理器的全局过期时间.
        session.setTimeout(10000); // 设置当前 Session 的过期时间
        System.out.println("Timeout : "+session.getTimeout());
        System.out.println("StartTime : "+session.getStartTimestamp()); // 获取会话的启动时间
        System.out.println("LastAccessTime : "+session.getLastAccessTime()); // 获取会话的最后访问时间
        Thread.sleep(1000);
        session.touch(); // 更新会话的最后访问时间
        System.out.println("LastAccessTime : "+session.getLastAccessTime()); // 获取会话的最后访问时间
        session.setAttribute("key", "123"); // 设置会话属性
        Assert.assertEquals("123", session.getAttribute("key")); // 获取会话属性
        session.removeAttribute("key"); // 删除会话属性
        session.stop(); // 销毁会话

        subject.logout(); // 登出
    }
}
```

# 2. 会话管理器

会话管理器管理着应用中所有Subject的会话的创建,维护,删除,失效,验证等工作.

![会话管理器](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/13126787-a649f3c2bc4de7e1.png)

```java
SecurityManager提供了如下接口
Session start(SessionContext context); //启动会话
Session getSession(SessionKey key) throws SessionException;//根据会话Key获取会话

另外,
1. 用于Web环境的WebSessionManager又提供了如下接口:
boolean isServletContainerSessions();//是否使用Servlet容器的会话
2. Shiro还提供了ValidatingSessionManager用于验资并过期会话:
void validateSessions();//验证所有会话是否过期

Shiro提供了三个默认实现:
DefaultSessionManager:DefaultSecurityManager使用的默认实现,用于JavaSE环境;
ServletContainerSessionManager:DefaultWebSecurityManager使用的默认实现,用于Web环境;其直接使用Servlet容器的会话;
DefaultWebSessionManager:用于Web环境的实现,可以替代ServletContainerSessionManager,自己维护着会话,直接废弃了Servlet容器的会话管理.
```

## 2.1 整合

```Java
DefaultWebSecurityManager defaultWebSecurityManager = new DefaultWebSecurityManager();

// 创建会话管理器
DefaultWebSessionManager defaultWebSessionManager = new DefaultWebSessionManager();

// 设置会话管理器
defaultWebSecurityManager.setSessionManager(defaultWebSessionManager);

SecurityUtils.setSecurityManager(defaultWebSecurityManager);
```

# 3. 会话监听器

会话监听器用于监听会话创建,过期及停止事件.

## 3.1 Java使用

```pom
<!--工程中单独引入了slf4j，不去除的话jar包冲突-->
<!--可以使用该命令查看jar包是否有冲突，mvn dependency:tree -Dverbose|grep conflict-->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-core</artifactId>
    <version>1.3.2</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-web</artifactId>
    <version>1.3.2</version>
</dependency>
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.3.2</version>
</dependency>
```

```Java
public class MySessionListener implements SessionListener {
    @Override
    public void onStart(Session session) {//会话创建时触发
        System.out.println("会话创建：" + session.getId());
    }
    @Override
    public void onExpiration(Session session) {//会话过期时触发
        System.out.println("会话过期：" + session.getId());
    }
    @Override
    public void onStop(Session session) {//退出/会话过期时触发
        System.out.println("会话停止：" + session.getId());
    }
}
```

```Java
// 如果只想监听某一个事件,可以继承SessionListenerAdapter实现:
public class MySessionListener2 extends SessionListenerAdapter {
    @Override
    public void onStart(Session session) {
        System.out.println("会话创建：" + session.getId());
    }
}
```

## 3.2 整合

```Java
DefaultWebSecurityManager defaultWebSecurityManager = new DefaultWebSecurityManager();
defaultWebSecurityManager.setRealm(simpleAccountRealm);
DefaultWebSessionManager defaultWebSessionManager = new DefaultWebSessionManager();

// 创建监听器
Collection <SessionListener> collection = new HashSet<>();
collection.add(new MySessionListener());
collection.add(new MySessionListener2());
// 设置监听器
defaultWebSessionManager.setSessionListeners(collection);
defaultWebSecurityManager.setSessionManager(defaultWebSessionManager);

SecurityUtils.setSecurityManager(defaultWebSecurityManager);
```

# 4. 会话存储/持久化

SessionDAO是用于session持久化的,SessionManager只是负责session的管理,持久化的工作是由SessionDAO完成的.

## 4.1 Java使用

```java
// 自定义DAO

package com.hust.hustshiro;

import org.apache.shiro.session.Session;
import org.apache.shiro.session.mgt.eis.AbstractSessionDAO;
import org.apache.shiro.session.mgt.eis.SessionDAO;
import org.springframework.util.SerializationUtils;
import redis.clients.jedis.Jedis;

import java.io.Serializable;
import java.util.Collection;
import java.util.HashSet;
import java.util.Set;

public class MyRedisSessionDao extends AbstractSessionDAO {
    //session过期时间
    private static final int SESSION_EXPIRE = 60 * 30;

    // Jedis客户端
    private static Jedis client = new Jedis("127.0.0.1",6379);

    @Override
    protected Serializable doCreate(Session session) {
        Serializable sessionId = generateSessionId(session);
        assignSessionId(session,sessionId);
        byte[] keyByte = sessionId.toString().getBytes();
        byte[] sessionByte = SerializationUtils.serialize(session);
        client.setex(keyByte, SESSION_EXPIRE, sessionByte);
        System.out.println("doCreate:"+sessionId);
        return sessionId;
    }
    @Override
    protected Session doReadSession(Serializable sessionId) {
        if (sessionId == null) {
            return null;
        }
        byte[] keyByte = sessionId.toString().getBytes();
        byte[] sessionByte = client.get(keyByte);
        return (Session) SerializationUtils.deserialize(sessionByte);
    }
    @Override
    public void update(Session session){
        byte[] keyByte = session.getId().toString().getBytes();
        byte[] sessionByte = SerializationUtils.serialize(session);
        client.setex(keyByte, SESSION_EXPIRE, sessionByte);
    }
    @Override
    public void delete(Session session) {
        byte[] keyByte = session.getId().toString().getBytes();
        client.del(keyByte);
    }
    @Override
    public Collection<Session> getActiveSessions() {
        Set<byte[]> keyByteSet = client.keys("*".getBytes());
        Set<Session> sessionSet = new HashSet<>();
        for (byte[] keyByte : keyByteSet) {
            byte[] sessionByte = client.get(keyByte);
            Session session = (Session) SerializationUtils.deserialize(sessionByte);
            if (session != null) {
                sessionSet.add(session);
            }
        }
        return sessionSet;
    }
}
```

## 4.2 整合

```java
DefaultWebSecurityManager defaultWebSecurityManager = new DefaultWebSecurityManager();

DefaultWebSessionManager defaultWebSessionManager = new DefaultWebSessionManager();
// 装配自定义SessionDAO
defaultWebSessionManager.setSessionDAO(new MyRedisSessionDao());

defaultWebSecurityManager.setSessionManager(defaultWebSessionManager);
```

# 5. 源代码地址

GitHub链接:https://github.com/ganlu19940318/shiro-session

# 6. 参考链接

[跟我学Shiro教程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E8%B7%9F%E6%88%91%E5%AD%A6Shiro%E6%95%99%E7%A8%8B.pdf)
[一步一步教你用shiro——1引入shiro框架](https://www.jianshu.com/p/8276bbea7774)
[SpringBoot+Shiro学习（五）：Session会话管理](https://www.jianshu.com/p/5aa03c2d118e)
[shiro会话监听器](https://blog.csdn.net/zsg88/article/details/69140133)
[一步一步教你用shiro——4配置并自定义sessionDao](https://www.jianshu.com/p/f85e50f41100)
[shiro的SessionDao（十二）](https://blog.csdn.net/qq_29347295/article/details/79084673)