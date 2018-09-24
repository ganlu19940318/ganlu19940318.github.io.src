---
title: Java自动装箱拆箱
date: 2018-09-24 08:12:57
categories: Java基础
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 前言

在后台开发的过程中, 使用框架开发, 经常会不得已遇到自动拆装箱的问题, 这篇文章主要是记录自动拆箱装箱的相关知识, 便于自己写出高质量的代码.

# 2. 概述

## 2.1 什么是自动装箱拆箱

很简单, 下面两句代码就可以看到装箱和拆箱过程.

```Java
//自动装箱
Integer total = 99;
//自定拆箱
int totalprim = total;
```

简单一点说, 装箱就是自动将基本数据类型转换为包装器类型; 拆箱就是自动将包装器类型转换为基本数据类型.
下面我们来看看需要装箱拆箱的类型有哪些:
![自动装箱拆箱](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20160329101454749.png)

## 2.2 执行过程

这个过程是自动执行的, 那么我们需要看看它的执行过程:

```Java
public class Main {
    public static void main(String[] args) {
    //自动装箱
    Integer total = 99;
    //自定拆箱
    int totalprim = total;
    }
}
```

反编译class文件之后得到如下内容:

```Java
javap -c Main.class
```

![执行过程](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/201809061551111.png)
Integer total = 99;
执行上面那句代码的时候, 系统为我们执行了:
Integer total = Integer.valueOf(99);

int totalprim = total;
执行上面那句代码的时候，系统为我们执行了:
int totalprim = total.intValue();

我们现在就以Integer为例, 来分析一下它的源码

### 2.2.1 Integer.valueOf函数

```Java
public static Integer valueOf(int i) {
    return  i >= 128 || i < -128 ? new Integer(i) : SMALL_VALUES[i + 128];
}
```

它会首先判断i的大小: 如果i小于-128或者大于等于128. 就创建一个Integer对象. 否则执行SMALL_VALUES[i + 128].
首先我们来看看Integer的构造函数:

```Java
private final int value;
public Integer(int value) {
    this.value = value;
}
public Integer(String string) throws NumberFormatException {
    this(parseInt(string));
}
```

它里面定义了一个value变量, 创建一个Integer对象, 就会给这个变量初始化. 第二个传入的是一个String变量, 它会先把它转换成一个int值, 然后进行初始化.
下面看看SMALL_VALUES[i + 128]是什么东西:

```Java
private static final Integer[] SMALL_VALUES = new Integer[256];
```

它是一个静态的Integer数组对象, 也就是说最终valueOf返回的都是一个Integer对象.
所以我们这里可以总结一点: 装箱的过程会创建对应的对象, 这个会消耗内存, 所以装箱的过程会增加内存的消耗, 影响性能.

### 2.2.2 intValue函数

```Java
@Override
public int intValue() {
    return value;
}
```

这个很简单, 直接返回value值即可.

### 2.2.3 相关问题

上面我们看到在Integer的构造函数中, 它分两种情况:

1. i >= 128 || i < -128 =====> new Integer(i)
2. i < 128 && i >= -128 =====> SMALL_VALUES[i + 128]

```Java
private static final Integer[] SMALL_VALUES = new Integer[256];
```

SMALL_VALUES本来已经被创建好, 也就是说在i >= 128 || i < -128是会创建不同的对象, 在i < 128 && i >= -128会根据i的值返回已经创建好的指定的对象.

```Java
public class Main {
    public static void main(String[] args) {
        Integer i1 = 100;
        Integer i2 = 100;
        Integer i3 = 200;
        Integer i4 = 200;
        System.out.println(i1==i2);  //true
        System.out.println(i3==i4);  //false
    }
}
```

代码的后面, 我们可以看到它们的执行结果是不一样的, 为什么, 在看看我们上面的说明.

1. i1和i2会进行自动装箱, 执行了valueOf函数, 它们的值在[-128,128)这个范围内, 它们会拿到SMALL_VALUES数组里面的同一个对象SMALL_VALUES[256], 它们引用到了同一个Integer对象, 所以它们肯定是相等的.
2. i3和i4也会进行自动装箱, 执行了valueOf函数, 它们的值大于128, 所以会执行new Integer(200), 也就是说它们会分别创建两个不同的对象, 所以它们肯定不等.

下面我们来看看另外一个例子:

```Java
public class Main {
    public static void main(String[] args) {
        Double i1 = 100.0;
        Double i2 = 100.0;
        Double i3 = 200.0;
        Double i4 = 200.0;
        System.out.println(i1==i2); //false
        System.out.println(i3==i4); //false
    }
}
```

看看上面的执行结果, 跟Integer不一样, 这样也不必奇怪, 因为它们的valueOf实现不一样, 结果肯定不一样, 那为什么它们不统一一下呢?
这个很好理解, 因为对于Integer, 在[-128,128)之间只有固定的256个值, 所以为了避免多次创建对象, 我们事先就创建好一个大小为256的Integer数组SMALL_VALUES, 所以如果值在这个范围内, 就可以直接返回我们事先创建好的对象就可以了.
但是对于Double类型来说, 我们就不能这样做, 因为它在这个范围内个数是无限的.
总结一句就是: 在某个范围内的整型数值的个数是有限的, 而浮点数却不是.
所以在Double里面的做法很直接, 就是直接创建一个对象, 所以每次创建的对象都不一样.

```Java
public static Double valueOf(double d) {
    return new Double(d);
}
```

下面我们进行一个归类:
Integer派别: Integer, Short, Byte, Character, Long这几个类的valueOf方法的实现是类似的.
Double派别: Double, Float的valueOf方法的实现是类似的. 每次都返回不同的对象.
下面对Integer派别进行一个总结, 如下图:
![归类](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20150922153039509.png)
下面我们来看看另外一种情况:

```Java
public class Main {
    public static void main(String[] args) {
        Boolean i1 = false;
        Boolean i2 = false;
        Boolean i3 = true;
        Boolean i4 = true;
        System.out.println(i1==i2);//true
        System.out.println(i3==i4);//true
    }
}
```

可以看到返回的都是true, 也就是它们执行valueOf返回的都是相同的对象.

```Java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

可以看到它并没有创建对象, 因为在内部已经提前创建好两个对象, 因为它只有两种情况, 这样也是为了避免重复创建太多的对象.

```Java
public static final Boolean TRUE = new Boolean(true);
public static final Boolean FALSE = new Boolean(false);
```

上面把几种情况都介绍到了 , 下面来进一步讨论其他情况.

```Java
Integer num1 = 400;  
int num2 = 400;  
System.out.println(num1 == num2); //true
```

说明num1 == num2进行了拆箱操作

```Java
Integer num1 = 100;  
int num2 = 100;  
System.out.println(num1.equals(num2));  //true
```

我们先来看看equals源码:

```Java
@Override
public boolean equals(Object o) {
    return (o instanceof Integer) && (((Integer) o).value == value);
}
```

我们指定equal比较的是内容本身, 并且我们也可以看到equal的参数是一个Object对象, 我们传入的是一个int类型, 所以首先会进行装箱, 然后比较, 之所以返回true , 是由于它比较的是对象里面的value值.

```Java
Integer num1 = 100;  
int num2 = 100;  
Long num3 = 200l;  
System.out.println(num1 + num2);  //200
System.out.println(num3 == (num1 + num2));  //true
System.out.println(num3.equals(num1 + num2));  //false
```

1. 当一个基础数据类型与封装类进行==, +, -, *, /运算时, 会将封装类进行拆箱, 对基础数据类型进行运算.  
2. 对于num3.equals(num1 + num2)为false的原因很简单, 我们还是根据代码实现来说明:

```Java
@Override
public boolean equals(Object o) {
    return (o instanceof Long) && (((Long) o).value == value);
}
```

它必须满足两个条件才为true:

1. 类型相同
2. 内容相同

上面返回false的原因就是类型不同.

```Java
Integer num1 = 100;
Integer num2 = 200;
Long num3 = 300l;
System.out.println(num3 == (num1 + num2)); //true
```

我们来反编译一些这个class文件:
![自动装箱拆箱](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20180906163749315.png)
可以看到运算的时候首先对num3进行拆箱(执行num3的longValue得到基础类型为long的值300), 然后对num1和mum2进行拆箱(分别执行了num1和num2的intValue得到基础类型为int的值100和200), 然后进行相关的基础运算.
我们来对基础类型进行一个测试:

```Java
int num1 = 100;
int num2 = 200;
long num3 = 300;
System.out.println(num3 == (num1 + num2)); //true
```

就说明了为什么最上面会返回true.
所以, 当 “==”运算符的两个操作数都是 包装器类型的引用, 则是比较指向的是否是同一个对象, 而如果其中有一个操作数是表达式(即包含算术运算)则比较的是数值(即会触发自动拆箱的过程).
陷阱:

```Java
Integer integer100=null;  
int int100=integer100;
```

这两行代码是完全合法的, 完全能够通过编译的,但 是在运行时, 就会抛出空指针异常.其 中, integer100为Integer类型的对象, 它当然可以指向null. 但在第二行时, 就会对integer100进行拆箱. 也就是对一个null对象执行intValue()方法, 当然会抛出空指针异常.所 以, 有拆箱操作时一定要特别注意封装类对象是否为null.

# 3. 特别强调

当两种不同类型用==比较时, 包装器类的会拆箱;
当同种类型用==比较时, 会直接比较.
(int 与 Integer 属于不同类型)

# 4. 参考链接

[详解Java的自动装箱与拆箱(Autoboxing and unboxing)](https://www.cnblogs.com/wang-yaz/p/8516151.html)