---
title: Shiro框架入门
date: 2019-6-4 06:56:34
categories: 技术储备
tags: [Shiro]
---

----

<!-- more -->

# 1. 介绍

Apache Shiro是Java的一个安全框架.

对比Spring Security: 没有Spring Security做的功能强大,但是在实际工作时可能并不需要那么复杂的东西,所以使用小而简单的Shiro就足够了.

Shiro可以帮助完成:认证,授权,加密,会话管理,与Web集成,缓存等.

## 1.1 基本功能点

![基本功能点](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190604091319.jpg)

Authentication:身份认证/登录,验证用户是不是拥有相应的身份;
Authorization:授权,即权限验证,验证某个已认证的用户是否拥有某个权限,即判断用户是否能做事情;
Session Manager:会话管理,即用户登录后就是一次会话,在没有退出之前,它的所有信息都在会话中;
Cryptography：加密,保护数据的安全性,如密码加密存储到数据库,而不是明文存储;
Web Support：Web支持,可以非常容易的集成到Web环境;
Caching：缓存,比如用户登录后,其用户信息,拥有的角色/权限不必每次去查,这样可以提高效率;
Concurrency：shiro支持多线程应用的并发验证,即如在一个线程中开启另一个线程,能把权限自动传播过去;
Testing：提供测试支持;
Run As：允许一个用户假装为另一个用户(如果他们允许)的身份进行访问;
Remember Me：记住我,这个是非常常见的功能,即一次登录后,下次再来的话不用登录了.

## 1.2 工作机制

![工作机制](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190604092342.jpg)

应用代码直接交互的对象是Subject,也就是说Shiro的对外API核心就是Subject;

Subject:主体.代表了当前"用户",与当前应用交互的任何东西都是Subject,所有Subject都绑定到SecurityManager,与Subject的所有交互都会委托给SecurityManager,可以把Subject认为是一个门面,SecurityManager才是实际的执行者;

SecurityManager:安全管理器.即所有与安全有关的操作都会与SecurityManager交互,它管理着所有Subject,可以看出它是Shiro的核心,它还负责与其他组件进行交互;

Realm:域.Shiro从从Realm获取安全数据,就是说SecurityManager要验证用户身份,那么它需要从Realm获取相应的用户进行比较以确定用户身份是否合法,也需要从Realm得到用户相应的角色/权限进行验证用户是否能进行操作,可以把Realm看成DataSource,即安全数据源.

最简单的一个Shiro应用工程流程:

1. 应用代码通过Subject来进行认证和授权,而Subject又委托给SecurityManager;
2. 给Shiro的SecurityManager注入Realm,从而让SecurityManager能得到合法的用户及其权限进行判断.

# 2. 入门使用

## 2.1 Maven环境

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

## 2.2 身份认证

### 2.2.1 Java代码

```Java
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.mgt.DefaultSecurityManager;
import org.apache.shiro.realm.SimpleAccountRealm;
import org.apache.shiro.subject.Subject;
import org.junit.Before;
import org.junit.Test;
public class AuthenticationTest {
    SimpleAccountRealm simpleAccountRealm = new SimpleAccountRealm();

    @Before // 在方法开始前添加一个用户
    public void addUser() {
        simpleAccountRealm.addAccount("ganlu", "123456");
    }

    @Test
    public void testAuthentication() {
        // 1.构建SecurityManager环境
        DefaultSecurityManager defaultSecurityManager = new DefaultSecurityManager();
        defaultSecurityManager.setRealm(simpleAccountRealm);
        // 2.主体提交认证请求
        SecurityUtils.setSecurityManager(defaultSecurityManager); // 设置SecurityManager环境
        Subject subject = SecurityUtils.getSubject(); // 获取当前主体
        UsernamePasswordToken token = new UsernamePasswordToken("ganlu", "123456");
        subject.login(token); // 登录
        // subject.isAuthenticated()方法返回一个boolean值,用于判断用户是否认证成功
        System.out.println("isAuthenticated:" + subject.isAuthenticated()); // 输出true
        subject.logout(); // 登出
        System.out.println("isAuthenticated:" + subject.isAuthenticated()); // 输出false
    }
}
```

## 2.3 授权

### 2.3.1 Java代码

```Java
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.mgt.DefaultSecurityManager;
import org.apache.shiro.realm.SimpleAccountRealm;
import org.apache.shiro.subject.Subject;
import org.junit.Before;
import org.junit.Test;

public class AuthorizationTest {
    SimpleAccountRealm simpleAccountRealm = new SimpleAccountRealm();

    @Before // 在方法开始前添加一个用户,让它具备admin和user两个角色
    public void addUser() {
        simpleAccountRealm.addAccount("ganlu", "123456", "admin", "user");
    }

    @Test
    public void testAuthentication() {
        // 1.构建SecurityManager环境
        DefaultSecurityManager defaultSecurityManager = new DefaultSecurityManager();
        defaultSecurityManager.setRealm(simpleAccountRealm);
        // 2.主体提交认证请求
        SecurityUtils.setSecurityManager(defaultSecurityManager); // 设置SecurityManager环境
        Subject subject = SecurityUtils.getSubject(); // 获取当前主体
        UsernamePasswordToken token = new UsernamePasswordToken("ganlu", "123456");
        subject.login(token); // 登录
        // subject.isAuthenticated()方法返回一个boolean值,用于判断用户是否认证成功
        System.out.println("isAuthenticated:" + subject.isAuthenticated()); // 输出true
        // 判断subject是否具有admin和user两个角色权限,如没有则会报错
        subject.checkRoles("admin","user");
        // subject.checkRole("xxx"); // 报错
    }
}
```

## 2.4 自定义Realm

### 2.4.1 Java代码

```Java
import org.apache.shiro.authc.AuthenticationException;
import org.apache.shiro.authc.AuthenticationInfo;
import org.apache.shiro.authc.AuthenticationToken;
import org.apache.shiro.authc.SimpleAuthenticationInfo;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
public class MyRealm extends AuthorizingRealm {

    super.setName("myRealm"); // 设置自定义Realm的名称，取什么无所谓..

    /**
     * 授权
     *
     * @param principalCollection
     * @return
     */
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        String userName = (String) principalCollection.getPrimaryPrincipal();
        // 从数据库获取角色和权限数据
        Set<String> roles = getRolesByUserName(userName);
        Set<String> permissions = getPermissionsByUserName(userName);
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        simpleAuthorizationInfo.setStringPermissions(permissions);
        simpleAuthorizationInfo.setRoles(roles);
        return simpleAuthorizationInfo;
    }

    /**
     * 模拟从数据库中获取权限数据
     *
     * @param userName
     * @return
     */
    private Set<String> getPermissionsByUserName(String userName) {
        Set<String> permissions = new HashSet<>();
        permissions.add("user:delete");
        permissions.add("user:add");
        return permissions;
    }

    /**
     * 模拟从数据库中获取角色数据
     *
     * @param userName
     * @return
     */
    private Set<String> getRolesByUserName(String userName) {
        Set<String> roles = new HashSet<>();
        roles.add("admin");
        roles.add("user");
        return roles;
    }

    /**
     * 认证
     *
     * @param authenticationToken 主体传过来的认证信息
     * @return
     * @throws AuthenticationException
     */
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        // 1.从主体传过来的认证信息中，获得用户名
        String userName = (String) authenticationToken.getPrincipal();
        // 2.通过用户名到数据库中获取凭证
        String password = getPasswordByUserName(userName);
        if (password == null) {
            return null;
        }
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(userName, password, "myRealm");
        return authenticationInfo;
    }

    /**
     * 模拟从数据库取凭证的过程
     *
     * @param userName
     * @return
     */
    private String getPasswordByUserName(String userName) {
        /**
         * 模拟数据库数据
         */
        Map<String, String> userMap = new HashMap<>(16);
        userMap.put("ganlu", "123456");
        return userMap.get(userName);
    }
}
```

```Java
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.mgt.DefaultSecurityManager;
import org.apache.shiro.realm.SimpleAccountRealm;
import org.apache.shiro.subject.Subject;
import org.junit.Before;
import org.junit.Test;
public class AuthenticationTest {
    @Test
    public void testAuthentication() {

        MyRealm myRealm = new MyRealm(); // 实现自己的 Realm 实例

        // 1.构建SecurityManager环境
        DefaultSecurityManager defaultSecurityManager = new DefaultSecurityManager();
        defaultSecurityManager.setRealm(myRealm);

        // 2.主体提交认证请求
        SecurityUtils.setSecurityManager(defaultSecurityManager); // 设置SecurityManager环境
        Subject subject = SecurityUtils.getSubject(); // 获取当前主体

        UsernamePasswordToken token = new UsernamePasswordToken("ganlu", "123456");
        subject.login(token); // 登录

        // subject.isAuthenticated()方法返回一个boolean值,用于判断用户是否认证成功
        System.out.println("isAuthenticated:" + subject.isAuthenticated()); // 输出true
        // 判断subject是否具有admin和user两个角色权限,如没有则会报错
        subject.checkRoles("admin", "user");
//        subject.checkRole("xxx"); // 报错
        // 判断subject是否具有user:add权限
        subject.checkPermission("user:add");
    }
}
```

### 2.4.2 流程分析

```Java
subject.login(token)
当上述代码执行时,doGetAuthorizationInfo(PrincipalCollection principalCollection)会被调用

subject.isAuthenticated()
subject.checkRoles("admin", "user")
subject.checkPermission("user:add")
当上述代码执行时,doGetAuthenticationInfo(AuthenticationToken authenticationToken)会被调用
```

# 3. 参考链接

[Shiro入门这篇就够了【Shiro的基础知识、回顾URL拦截】](https://segmentfault.com/a/1190000013875092)
[跟我学Shiro教程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E8%B7%9F%E6%88%91%E5%AD%A6Shiro%E6%95%99%E7%A8%8B.pdf)
[Shiro安全框架【快速入门】就这一篇！](https://www.cnblogs.com/wmyskxz/p/10229148.html)