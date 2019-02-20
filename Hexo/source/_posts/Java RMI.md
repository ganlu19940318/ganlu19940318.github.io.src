---
title: Java RMI
date: 2019-02-20 15:14:53
categories: Java基础
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 基本使用

## 1.1 服务端

1.首先我们先创建一个实体类,这个类需要实现Serializable接口,用于信息的传输.

```java
import java.io.Serializable;
public class Student implements Serializable {
  private String name;
  private int age;
  public String getName() {
      return name;
  }
  public void setName(String name) {
      this.name = name;
  }
  public int getAge() {
      return age;
  }
  public void setAge(int age) {
      this.age = age;
  }
}
```

2.定义一个接口,这个接口需要继承Remote接口,这个接口中的方法必须声明RemoteException异常.

```java
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;
public interface StudentService extends Remote { 
  List<Student> getList() throws RemoteException;
}
```

3.创建一个类,并实现步骤2中的接口,但还需要继承UnicastRemoteObject类和显示写出无参的构造函数.

```java
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;
import java.util.ArrayList;
import java.util.List; 
public class StudentServiceImpl extends UnicastRemoteObject implements
      StudentService {
  public StudentServiceImpl() throws RemoteException {
  }
  public List<Student> getList() throws RemoteException {
      List<Student> list=new ArrayList<Student>();
      Student s1=new Student();
      s1.setName("张三");
      s1.setAge(15);
      Student s2=new Student();
      s2.setName("李四");
      s2.setAge(20);
      list.add(s1);
      list.add(s2);
      return list;
  }
}
```

4.创建服务并启动服务

```java
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
public class SetService {
    public static void main(String[] args) {
        try {
            StudentService studentService=new StudentServiceImpl();
            LocateRegistry.createRegistry(5008);//定义端口号
            Naming.rebind("rmi://127.0.0.1:5008/StudentService", studentService);
            System.out.println("服务已启动");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 1.2 客户端

1.首先我们先创建一个实体类,这个类需要实现Serializable接口,用于信息的传输.

2.定义一个接口,这个接口需要继承Remote接口,这个接口中的方法必须声明RemoteException异常.

上面两步与服务端的操作一致.

3.创建一个客户程序进行RMI调用.

```java
import java.rmi.Naming;
import java.util.List;
public class GetService {
  public static void main(String[] args) {
      try {
          StudentService studentService=(StudentService) Naming.lookup("rmi://127.0.0.1:5008/StudentService");
          List<Student> list = studentService.getList();
          for (Student s : list) {
              System.out.println("姓名："+s.getName()+",年龄："+s.getAge());
          }
      } catch (Exception e) {
          e.printStackTrace();
      }
  }
}
```

## 1.3 调用发生ClassCastException的处理方法

主要原因:是服务器端的包结构和客户端的包结构不同,这就造成了实际上你的服务器端的Interface的名字和你客户端的Interface名称不同,所以当然会造成转型异常了.

解决方法:把自己客户端涉及到RMI部分调整和服务器端完全一致.可能你的系统客户端包为com.xxx.client,而服务器端的包为com.xxx.server.你只需要将服务器端和客户端的包都改成同一个包名即可.例如都改成com.xxx.rmi.

# 2. 定义和基本认识

Java RMI: Java远程方法调用,即Java RMI(Java Remote Method Invocation)是Java编程语言里,一种用于实现远程过程调用的应用程序编程接口.它使客户机上运行的程序可以调用远程服务器上的对象.远程方法调用特性使Java编程人员能够在网络环境中分布操作.RMI全部的宗旨就是尽可能简化远程接口对象的使用.

RMI(Remote Method Invocation)为远程方法调用,是允许运行在一个Java虚拟机的对象调用运行在另一个Java虚拟机上的对象的方法.

这两个虚拟机可以是运行在相同计算机上的不同进程中,也可以是运行在网络上的不同计算机中.

RMI能让一个Java程序去调用网络中另一台计算机的Java对象的方法,那么调用的效果就像是在本机上调用一样.通俗的讲:A机器上面有一个class, 通过远程调用, B机器调用这个class 中的方法.

RMI的基础是接口,RMI构架基于一个重要的原理:定义接口和定义接口的具体实现是分开的.

# 3. 参考链接

[Java学习之路-RMI学习](https://www.cnblogs.com/qujiajun/p/4065857.html)
[Java学习笔记（十六）——Java RMI](https://www.cnblogs.com/xt0810/p/3640167.html)
[Java RMI调用发生ClassCastException的处理方法](https://blog.csdn.net/kungstriving/article/details/2541006)