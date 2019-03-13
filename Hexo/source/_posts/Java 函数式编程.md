---
title: Java 函数式编程
date: 2019-3-13 16:23:18
categories: Java基础
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 问题提出

Java最令人头痛的问题,是Java烦琐的语法.我们不得不花费大量的代码行数来实现一些司空见惯的功能,以至于Java程序总是冗长的.但是这一切在Java 8函数式编程中得到缓解.

# 2. 函数式编程介绍

1. 将函数作为参数传递给另外一个函数,这是函数式编程的特性之一.
2. 函数可以作为另外一个函数的返回值,也是函数式编程的重要特点.
3. 无副作用.(函数的副作用指的是函数在调用过程中,除了给出了返回值外,还修改了函数外部的状态,比如,函数在调用过程中,修改了某一个全局状态.函数式编程认为,函数的副用作应该被尽量避免.)
4. 申明式的,函数式编程是申明式的编程方式.相对于命令式而言,命令式的程序设计喜欢大量使用可变对象和指令.对于申明式的编程范式,你不在需要提供明确的指令操作,所有的细节指令将会更好的被程序库所封装,你要做的只是提出你要的要求,申明你的用意即可.
5. 不变的对象.在使用函数式编程时,几乎所有的对象都拒绝被修改.(这一点我觉得跟无副作用类似)
6. 易于并行.由于对象都处于不变的状态,因此函数式编程更加易于并行.
7. 更少的代码.

# 3. 函数式编程基础

## 3.1 FunctionalInterface注释

Java 8提出了函数式接口的概念.简单来说就是只定义了单一抽象方法的接口.比如下面的定义:

```java
@FunctionalInterface
public interface IntHandler {
    void handle(int i);
}
```

注释FunctionalInterface用于表明IntHandler接口是一个函数式接口,该接口被定义为只包含一个抽象方法handle(int i),因此它符合函数式接口的定义.如果声明两个抽象方法则会出现编译错误:

```java
@FunctionalInterface
public interface IntHandler {
    void handle(int i);
    void handle2(int i);
}
```

需要注意的是函数式接口只能有一个抽象方法,但是不是只能有一个方法:在Java8中,接口运行存在实例方法,其次任何被java.lang.Object实现的方法都不能视为抽象方法所以下面的接口并不会报错:

```java
@FunctionalInterface
public interface IntHandler {
    void handle(int i);
    String toString();
}
```

它完全是一个符合规范的函数式接口.

## 3.2 接口默认方法

在Java8之前接口只能包含抽象方法，但是在Java8之后接口也可以包含若干个实例方法。这一改进使Java8拥有了类似于多继承的能力。一个对象实例将拥有来自多个不同接口的实例方法。

在Java8中使用default关键字可以在接口内定义实例方法。（这个方法不是抽象方法，而是有特定逻辑的具体实例方法。）

```java
@FunctionalInterface
public interface IntHandler {
    void eat();
    default void wait(String name){
        System.out.println("I am waiting for " + name);
    }
}
```

从某种程度上说这种模式可以弥补Java单一继承的不便,但是需要注意的是它也会遇到和多继承相同的问题.

```java
public interface IntHandler {
    default void wait(String name){
        System.out.println("I am waiting for " + name);
    }
}

public interface LongHandler {
    default void wait(String name){
        System.out.println("I am not waiting for " + name);
    }
}

public class Hello implements IntHandler,LongHandler {
}
```

它会报编译错误.这时候的解决方法是

```java
public class Hello implements IntHandler,LongHandler {
    @Override
    public void wait(String name) {
        // todo
    }
}
```

## 3.3 lambda表达式

lambda表达式是函数式编程的核心.lambda表达式就是匿名函数它是一段没有函数名的函数体,可以作为参数直接传递给相关的调用者.

```java
// 示例
import java.util.Arrays;
import java.util.List;

public class Solution {
    public static void main(String[] args) {
        List<Integer> numbers= Arrays.asList(1,2,3,4,5);
        numbers.forEach((Integer value)->System.out.println(value));
    }
}
```

```text
// 输出结果
1
2
3
4
5
```

和匿名对象一样lambda表达式也可以访问外部的局部变量.
与匿名内部对象一样外部的i变量必须声明为final,这样才能保证在lambda表达式中合法的访问.
但是对于lambda来说即使去掉final定义也能正常运行.但是需要注意的是即使如此也不能修改外部的局部变量的值.

## 3.4 方法引用

方法引用是Java8中提出来用于简化lambda表达式的一种手段,它通过类名和方法名来定位到一个静态方法或者实例方法.

1. 静态方法引用：ClassName::methodName
2. 实例上的实例方法引用：instanceReference::methodName
3. 超类上的实例方法引用：super::methodName
4. 类型上的实例方法引用：ClassName::methodName
5. 构造方法引用：Class::new
6. 数组构造方法引用：TypeName[]::new

首先方法引用使用"::"来定义,"::"的前半部分表示类名或者实例名,后半部分表示方法名称,如果是构造函数则使用new表示.

下面给个示例

```java
public class User {
    private String name;
    private int id;

    public User(String name, int id) {
        this.name = name;
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }
}
```

```java
import java.util.ArrayList;
import java.util.List;

public class Solution {
    public static void main(String[] args) {
        List<User> user = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            user.add(new User("name" + i, i));
        }
        user.stream().map(User::getName).forEach(System.out::println);
    }
}
```

对于第一个方法引用"User::getName"表示是User类的实例方法.在执行时,Java会自动识别流中的元素(此处为User实例)是作为调用目标还是调用方法的参数.在"User::getName"中流内的元素应该作为调用目标,在这里调用了每一个User对象实例的getName()方法,并将这些User的name作为一个新的流.同时对于这里所有得到的name使用方法引用System.out::println进行处理.这里的System.out为PrintStream对象实例,因此,这里表示System.out实例的println方法,系统也会自动判断流内的元素此时应该作为方法的参数传入而不是调用目标.

一般来说如果使用的是静态方法或者调用目标明确那么流内的元素会自动作为参数使用.如果函数引用表示实例方法且不存在调动目标则流内元素自动作为调用目标.

因此如果一个类中存在同名的实例方法和静态函数,那么编译器就无法判断应该使用哪个方法进行调用. 如下编译器就会报错:

```java
import java.util.ArrayList;
import java.util.List;

public class Solution {
    public static void main(String[] args) {

        List<Double> doubles = new ArrayList<>();
        for(int i = 0; i < 10; i++){
            doubles.add(Double.valueOf(i));
        }
        doubles.stream().map(Double::toString).forEach(System.out::print);
    }
}
```

此时在Double中同时存在两个以下函数

```java
public static String toString(double d)
public String toString()
```

此时对函数引用的处理出现了歧义因此会在编译器就出错.

方法引用还可以用于直接使用构造函数.

# 4. 参考链接

<< Java高并发程序设计 >>(葛一鸣 郭超)

[函数式编程（基础部分）](https://blog.csdn.net/qq_37598011/article/details/82119783)
[关于Java8函数式编程你需要了解的几点](https://www.cnblogs.com/love-jishu/p/5383326.html)