---
title: Java错误的加锁实例
date: 2019-02-22 11:09:00
categories: Java
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 前言

在学习 << Java高并发程序设计 >>(葛一鸣 郭超) 的时候看到一个错误的加锁实例. 第一时间没有反应出来错误在哪, 在这里记录一下, 以后万一遇到了也容易解决.

# 2. 错误实例

```java
public class Test {
    public static class Demo extends Thread{
        public static Integer i = 0;

        @Override
        public void run() {
            for(int j = 0; j < 100000; j++){
                synchronized (i){
                    i++;
                }
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Demo();
        Thread t2 = new Demo();
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(Demo.i);
    }

}
```

输入结果并不是 200000, 而是比 200000 小.

# 3. 问题分析

问题出在Integer变量i. 直接上源码就明白了.

```java
    //* This method will always cache values in the range -128 to 127,
    //* inclusive, and may cache other values outside of this range.
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

# 4. 参考链接

<< Java高并发程序设计 >>(葛一鸣 郭超)
