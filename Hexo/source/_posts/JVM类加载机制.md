---
title: JVM类加载机制
date: 2018-09-24 10:02:35
categories: Java基础
tags: [Java, 基础储备, JVM]
---

----

<!-- more -->

# 1. 前言

作为一名Java后台开发的程序员, 深入理解JVM, 重要性不言而喻, 这篇文章主要是记录JVM类加载机制相关知识.

# 2. 类加载的时机

## 2.1 初始化情况

JVM类加载分为5个过程: 加载, 验证, 准备, 解析, 初始化, 使用, 卸载, 如下图所示:
![类加载的时机](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/2fb054008ca2898e0a17f7d79ce525a1.png)
那么, 什么情况下虚拟机需要开始初始化一个类呢? 这在虚拟机规范中是有严格规定的, 虚拟机规范指明, <font color=red>有且只有</font> 五种情况必须立即对类进行初始化(而这一过程自然发生在加载, 验证, 准备之后):

1. 遇到new, getstatic, putstatic或invokestatic这四条字节码指令, 如果类没有进行过初始化, 则需要先触发其初始化.
2. 使用java.lang.reflect包的方法对类进行反射调用的时候, 如果类没有进行过初始化, 则需要先触发其初始化.
3. 当初始化一个类的时候, 如果发现其父类还没有进行过初始化, 则需要先触发其父类的初始化.
4. 当虚拟机启动时, 户需要指定一个要执行的主类(包含main()方法的那个类), 虚拟机会先初始化这个主类.
5. 当使用jdk1.7动态语言支持时, 如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getstatic,REF_putstatic,REF_invokeStatic的方法句柄, 并且这个方法句柄所对应的类没有进行初始化, 则需要先出触发其初始化.

## 2.2 主动引用与被动引用

注意, 对于这五种会触发类进行初始化的场景, 虚拟机规范中使用了一个很强烈的限定语: “有且只有”, 这五种场景中的行为称为对一个类进行 <font color=red>主动引用</font>. 除此之外, 所有引用类的方式, 都不会触发初始化, 称为 <font color=red>被动引用</font>.
特别需要指出的是, 类的实例化与类的初始化是两个完全不同的概念:

1. 类的实例化是指创建一个类的实例(对象)的过程;
2. 类的初始化是指为类中各个类成员(被static修饰的成员变量)赋初始值的过程, 是类生命周期中的一个阶段.

## 2.3 被动引用经典示例

1.通过子类引用父类的静态字段, 不会导致子类初始化

```java
public class Father {
    static {
        System.out.println("父类初试化");
    }
    public static void work(){
        System.out.println("父类开始工作");
    }
}
```

```java
public class Son extends Father {
    static {
        System.out.println("子类初试化");
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        Son.work();
    }
}
```

运行结果:

```text
父类初试化
父类开始工作
```

2.通过数组定义来引用类, 不会触发此类的初始化

```java
public class Main {
    public static void main(String[] args) {
        Son[] sons = new Son[10];
    }
}
```

运行结果:

```text

```

3.常量在编译阶段会存入调用类的常量池中, 本质上并没有直接引用到定义常量的类, 因此不会触发定义常量的类的初始化

```java
public class Son{
    static {
        System.out.println("子类初试化");
    }
    public static final String message = "Hello World!";
}
```

```java
public class Main {
    public static void main(String[] args) {
        System.out.println(Son.message);
    }
}
```

运行结果:

```text
Hello World!
```

# 3. 类加载的过程

## 3.1 加载

1. 通过一个类的全限定名来获取定义此类的二进制字节流.
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构.
3. 在内存中生成一个代表这个类的java.lang.Class对象, 作为方法区这个类的各种数据的访问入口.

注意: JVM中的ClassLoader类加载器加载Class发生在此阶段.

## 3.2 验证

1. 文件格式的验证
2. 元数据验证
3. 字节码验证
4. 符号引用验证

## 3.3 准备

准备阶段是正式为类变量(static 成员变量)分配内存并设置类变量初始值(零值)的阶段, 这些变量所使用的内存都将在方法区中进行分配. 这时候进行内存分配的仅包括类变量 , 而不包括实例变量, 实例变量将会在对象实例化时随着对象一起分配在堆中. 其次, 这里所说的初始值“通常情况”下是数据类型的零值, 假设一个类变量的定义为:

```java
public static int value = 123;
```

那么, 变量value在准备阶段过后的值为0而不是123.
因为这时候尚未开始执行任何java方法, 而把value赋值为123的putstatic指令是程序被编译后, 存放于类构造器方法< clinit >()之中, 所以把value赋值为123的动作将在初始化阶段才会执行.
至于“特殊情况”是指: 当类字段的字段属性是ConstantValue时, 会在准备阶段初始化为指定的值, 所以标注为final之后, value的值在准备阶段初始化为123而非0.

```java
public static final int value = 123;
```

注意:
只设置类中的静态变量(方法区中), 不包括实例变量(堆内存中), 实例变量是在对象实例化的时候初始化分配值的.

## 3.4 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程.

符号引用: 简单的理解就是字符串, 比如引用一个类, java.util.ArrayList 这就是一个符号引用, 字符串引用的对象不一定被加载.
直接引用: 指针或者地址偏移量. 引用对象一定在内存(已经加载).

## 3.5 初始化

1. 执行类构造器< clinit >
2. 初始化静态变量, 静态块中的数据等(一个类加载器只会初始化一次)
3. 子类的< clinit >调用前保证父类的< clinit >被调用

注意:
< clinit >是线程安全的, 执行< clinit >的线程需要先获取锁才能进行初始化操作, 保证只有一个线程能执行< clinit >

# 4. 类加载器

java.lang.ClassLoader类的基本职责就是根据一个指定的类的名称, 找到或者生成其对应的字节代码, 然后从这些字节代码中定义出一个Java 类, 即 java.lang.Class类的一个实例.
ClassLoader提供了一系列的方法, 比较重要的方法如:
![类加载器](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/132037411767454.png)

## 4.1 类加载器的树状层次结构

Java 中的类加载器大致可以分成两类, 一类是系统提供的, 另外一类则是由 Java 应用开发人员编写的.
![类加载器的树状层次结构](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/675733-4a67cdc8bf8c8803.webp)

### 4.1.1 引导类加载器(Bootstrap ClassLoader)

它用来加载 Java 的核心库(jre/lib/rt.jar), 是用原生C++代码来实现的, 并不继承自java.lang.ClassLoader.

### 4.1.2 扩展类加载器(Extensions ClassLoader)

它用来加载 Java 的扩展库(jre/ext/*.jar). Java 虚拟机的实现会提供一个扩展库目录. 该类加载器在此目录里面查找并加载 Java 类.

### 4.1.3 系统类加载器(System ClassLoader)

它根据 Java 应用的类路径(CLASSPATH)来加载 Java 类. 一般来说, Java 应用的类都是由它来完成加载的. 可以通过 ClassLoader.getSystemClassLoader()来获取它.

### 4.1.4 自定义类加载器(Custom ClassLoader)

除了系统提供的类加载器以外, 开发人员可以通过继承 java.lang.ClassLoader类的方式实现自己的类加载器, 以满足一些特殊的需求.

### 4.1.5 测试

```java
public class Main {
    public static void main(String[] args) {
        //application class loader
        System.out.println(ClassLoader.getSystemClassLoader());
        //extensions class loader
        System.out.println(ClassLoader.getSystemClassLoader().getParent());
        //bootstrap class loader
        System.out.println(ClassLoader.getSystemClassLoader().getParent().getParent());
    }
}
```

结果:

```text
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@4554617c
null
```

可以看出ClassLoader类是由AppClassLoader加载的. 他的父类是ExtClassLoader, ExtClassLoader的父亲无法获取是因为它是用C++实现的.

## 4.2 双亲委派机制

某个特定的类加载器在接到加载类的请求时, 首先将加载任务委托交给父类加载器, 父类加载器又将加载任务向上委托, 直到最父类加载器, 如果最父类加载器可以完成类加载任务, 就成功返回, 如果不行就向下传递委托任务, 由其子类加载器进行加载.

### 4.2.1 双亲委派机制的好处

保证java核心库的安全性(例如: 如果用户自己写了一个java.lang.String类就会因为双亲委派机制不能被加载, 不会破坏原生的String类的加载)

### 4.2.2 代理模式

与双亲委派机制相反, 代理模式是先自己尝试加载, 如果无法加载则向上传递. Tomcat就是代理模式.

## 4.3 双亲委派模型的破坏者-线程上下文类加载器

在Java应用中存在着很多服务提供者接口(Service Provider Interface, SPI), 这些接口允许第三方为它们提供实现, 如常见的 SPI 有 JDBC, JNDI等, 这些 SPI 的接口属于 Java 核心库, 一般存在rt.jar包中, 由Bootstrap类加载器加载, 而 SPI 的第三方实现代码则是作为Java应用所依赖的 jar 包被存放在classpath路径下, 由于SPI接口中的代码经常需要加载具体的第三方实现类并调用其相关方法, 但SPI的核心接口类是由引导类加载器来加载的, 而Bootstrap类加载器无法直接加载SPI的实现类, 同时由于双亲委派模式的存在, Bootstrap类加载器也无法反向委托AppClassLoader加载器SPI的实现类. 在这种情况下, 我们就需要一种特殊的类加载器来加载第三方的类库, 而线程上下文类加载器就是很好的选择.
线程上下文类加载器(contextClassLoader)是从 JDK 1.2 开始引入的, 我们可以通过java.lang.Thread类中的getContextClassLoader()和 setContextClassLoader(ClassLoader cl)方法来获取和设置线程的上下文类加载器. 如果没有手动设置上下文类加载器, 线程将继承其父线程的上下文类加载器, 初始线程的上下文类加载器是AppClassLoader, 在线程中运行的代码可以通过此类加载器来加载类和资源, 如下图所示, 以jdbc.jar加载为例.
![线程上下文类加载器](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20170625143404387.png)
从图可知rt.jar核心包是由Bootstrap类加载器加载的, 其内包含SPI核心接口类, 由于SPI中的类经常需要调用外部实现类的方法, 而jdbc.jar包含外部实现类(jdbc.jar存在于classpath路径)无法通过Bootstrap类加载器加载, 因此只能委派线程上下文类加载器把jdbc.jar中的实现类加载到内存以便SPI相关类使用. 显然这种线程上下文类加载器的加载方式破坏了“双亲委派模型”, 它在执行过程中抛弃双亲委派加载链模式, 使程序可以逆向使用类加载器, 当然这也使得Java类加载器变得更加灵活. 为了进一步证实这种场景, 不妨看看DriverManager类的源码, DriverManager是Java核心rt.jar包中的类, 该类用来管理不同数据库的实现驱动即Driver, 它们都实现了Java核心包中的java.sql.Driver接口, 如mysql驱动包中的com.mysql.jdbc.Driver, 这里主要看看如何加载外部实现类,在 DriverManager初始化时会执行如下代码

```java
//DriverManager是Java核心包rt.jar的类
public class DriverManager {
    //省略不必要的代码
    static {
        loadInitialDrivers();//执行该方法
        println("JDBC DriverManager initialized");
    }

//loadInitialDrivers方法
 private static void loadInitialDrivers() {
     sun.misc.Providers()
     AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                //加载外部的Driver的实现类
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
              //省略不必要的代码......
            }
        });
    }
```

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
     //通过线程上下文类加载器加载
      ClassLoader cl = Thread.currentThread().getContextClassLoader();
      return ServiceLoader.load(service, cl);
  }
```

很明显了确实通过线程上下文类加载器加载的, 实际上核心包的SPI类对外部实现类的加载都是基于线程上下文类加载器执行的, 通过这种方式实现了Java核心代码内部去调用外部实现类. 我们知道线程上下文类加载器默认情况下就是AppClassLoader, 那为什么不直接通过getSystemClassLoader()获取类加载器来加载classpath路径下的类的呢?
其实是可行的, 但这种直接使用getSystemClassLoader()方法获取AppClassLoader加载类有一个缺点, 那就是代码部署到不同服务时会出现问题, 如把代码部署到Java Web应用服务或者EJB之类的服务将会出问题, 因为这些服务使用的线程上下文类加载器并非AppClassLoader, 而是Java Web应用服自家的类加载器, 类加载器不同. 所以我们应用该少用getSystemClassLoader(). 总之不同的服务使用的可能默认ClassLoader是不同的, 但<font color=red>使用线程上下文类加载器总能获取到与当前程序执行相同的ClassLoader</font>, 从而避免不必要的问题.

## 4.4 自定义类加载器

### 4.4.1 普通类加载器

通常情况下, 我们都是直接使用系统类加载器. 但是, 有的时候, 我们也需要自定义类加载器. 比如应用是通过网络来传输 Java 类的字节码, 为保证安全性, 这些字节码经过了加密处理, 这时系统类加载器就无法对其进行加载, 这样则需要自定义类加载器来实现. 自定义类加载器一般都是继承自 ClassLoader类, 我们只需要重写findClass方法即可. 下面我们通过一个示例来演示自定义类加载器的流程:

```java
import java.io.*;
public class MyClassLoader extends ClassLoader {
    private String root;
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }
    private byte[] loadClassData(String className) {
        String fileName = root + File.separatorChar
                + className.replace('.', File.separatorChar) + ".class";
        try {
            InputStream ins = new FileInputStream(fileName);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 1024;
            byte[] buffer = new byte[bufferSize];
            int length = 0;
            while ((length = ins.read(buffer)) != -1) {
                baos.write(buffer, 0, length);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
    public void setRoot(String root) {
        this.root = root;
    }
}
```

```java
public class Te {
    static {
        System.out.println("hello world");
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        MyClassLoader classLoader = new MyClassLoader();
        classLoader.setRoot("C:\\Users\\ganlu\\IdeaProjects\\untitled3\\out\\production\\untitled3");
        Class<?> testClass = null;
        try {
            testClass = classLoader.loadClass("Te");
            Object object = testClass.newInstance();
            System.out.println(object.getClass().getClassLoader());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

运行结果:

```text
hello world
sun.misc.Launcher$AppClassLoader@18b4aac2
```

### 4.4.2 热部署类加载器

所谓的热部署就是利用同一个class文件不同的类加载器在内存创建出两个不同的class对象(即利用不同的类加载实例), 由于JVM在加载类之前会检测请求的类是否已加载过(即在loadClass()方法中调用findLoadedClass()方法), 如果被加载过, 则直接从缓存获取, 不会重新加载. 注意同一个类加载器的实例和同一个class文件只能被加载器一次, 多次加载将报错, 因此我们实现的热部署必须让同一个class文件可以根据不同的类加载器重复加载, 以实现所谓的热部署. 通过直接调用findClass()方法, 而不是调用loadClass()方法即可实现, 因为ClassLoader中loadClass()方法体中调用findLoadedClass()方法进行了检测是否已被加载，因此我们直接调用findClass()方法就可以绕过这个问题, 当然也可以重写loadClass方法, 但强烈不建议这么干.

```java
MyClassLoader.java 和 Te.java 都用上面的定义.
```

```java
public class Main {
    public static void main(String[] args) {
        MyClassLoader classLoader1 = new MyClassLoader();
        MyClassLoader classLoader2 = new MyClassLoader();
        classLoader1.setRoot("C:\\Users\\ganlu\\IdeaProjects\\untitled3\\out\\production\\untitled3");
        classLoader2.setRoot("C:\\Users\\ganlu\\IdeaProjects\\untitled3\\out\\production\\untitled3");
        try {
            Class<?> testClass1 = classLoader1.loadClass("Te");
            Class<?> testClass2 = classLoader2.loadClass("Te");

            System.out.println(testClass1.hashCode());
            System.out.println(testClass1.getClassLoader());
            System.out.println(testClass2.hashCode());
            System.out.println(testClass2.getClassLoader());

            Class<?> testClass3 = classLoader1.findClass("Te");
            Class<?> testClass4 = classLoader2.findClass("Te");
            System.out.println(testClass3.hashCode());
            System.out.println(testClass3.getClassLoader());
            System.out.println(testClass4.hashCode());
            System.out.println(testClass4.getClassLoader());

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }
}
```

运行结果:

```Java
1956725890
sun.misc.Launcher$AppClassLoader@18b4aac2
1956725890
sun.misc.Launcher$AppClassLoader@18b4aac2
1836019240
MyClassLoader@1540e19d
325040804
MyClassLoader@14ae5a5
```

# 5. 参考链接

<<深入理解Java虚拟机—-JVM高级特性与最佳实践>>(第二版, 周志明)
[JVM类生命周期概述：加载时机与加载过程](https://blog.csdn.net/justloveyou_/article/details/72466105)
[JVM 类加载机制深入浅出](https://www.jianshu.com/p/3cab74a189de)
[JVM(四)—一道面试题搞懂JVM类加载机制](https://blog.csdn.net/noaman_wgs/article/details/74489549)
[深入探讨 Java 类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/index.html)
[Java 类加载机制详解](https://www.jianshu.com/p/808a36134da5)
[全面解析Java类加载器](https://www.cnblogs.com/sunniest/p/4574080.html)
[深入理解Java类加载器(ClassLoader)](https://blog.csdn.net/javazejian/article/details/73413292)
[JVM 类加载机制详解](http://www.importnew.com/25295.html)