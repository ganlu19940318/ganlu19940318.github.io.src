---
title: Java 如何在线程池中寻找堆栈
date: 2019-02-24 17:01:41
categories: 高并发程序设计
tags: [Java, 高并发程序设计, 基础储备]
---

----

<!-- more -->

# 1. 问题

```java
import java.util.concurrent.*;
public class Test {
    public static class DivTask implements Runnable{
        int a, b;
        public DivTask(int a, int b) {
            this.a = a;
            this.b = b;
        }
        @Override
        public void run() {
            double re = a / b;
            System.out.println(re);
        }
    }
    public static void main(String[] args){
        ExecutorService executorService = new ThreadPoolExecutor(0, 5, 0L, TimeUnit.MILLISECONDS, new SynchronousQueue<>());
        for(int i = 0; i < 5; i++){
            executorService.submit(new DivTask(100, i));
        }
    }
}
```

输出结果

```text
100.0
33.0
25.0
50.0
```

没有报错信息.

# 2. 解决方案探索

## 2.1 方法一:放弃submit改用execute

```java
executorService.execute(new DivTask(100, i));
```

```java
Exception in thread "pool-1-thread-1" java.lang.ArithmeticException: / by zero
    at Test$DivTask.run(Test.java:15)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)
```

## 2.2 方法二:将submit的返回值打印

```java
Future re = executorService.submit(new DivTask(100, i));
re.get();
```

```java
java.util.concurrent.ExecutionException: java.lang.ArithmeticException: / by zero
    at java.util.concurrent.FutureTask.report(FutureTask.java:122)
    at java.util.concurrent.FutureTask.get(FutureTask.java:192)
    at Test.main(Test.java:24)
Caused by: java.lang.ArithmeticException: / by zero
    at Test$DivTask.run(Test.java:15)
    at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)
```

# 3. 解决方案对比

上面的解决方案一无法定位到问题是从哪里提交的

方案二可以知道问题是从哪里提交的

推荐采用方案二.

# 4. 参考链接

<< Java高并发程序设计 >>(葛一鸣 郭超)
