---
title: CPU Cache的优化--解决伪共享问题
date: 2019-3-11 19:39:42
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 伪共享问题(false sharing)

对于解释伪共享问题,就需要了解一下缓存行的相关概念.缓存行是主存复制到高速缓存的最小单位,一般情况下缓存行的大小为32~128字节(通常为64字节).

在多线程程序执行的过程中,有可能将2个或多个需要频繁修改的变量存储在同一个缓存行当中.这样以来,会频繁的造成缓存头失效的问题.如下图所示:

![伪共享问题](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/02B12F63-E709-49E8-9D97-60661CB1E449.png)

从编码的角度,为了解决上面的问题,可以使用额外字段来填充缓存行数据.从而达到不同变量之间占用不用的缓存行,增加缓存的命中率.优化后如下图所示:

![伪共享问题](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/388F0DDE-2298-492E-BEE1-373C073A1B89.png)

# 2. 示例

```java
public class Test implements Runnable {
    //public static final int NUM_THREAD = Runtime.getRuntime().availableProcessors(); // CUP核数
    public static final int NUM_THREAD = 4; // CPU核数
    public static final long INTERATIONS = 500L * 1000L * 1000L;
    private final int arrayIndex;

    private static VolatileLong[] longs = new VolatileLong[NUM_THREAD];

    static {
        for (int i = 0; i < longs.length; i++) {
            longs[i] = new VolatileLong();
        }
    }

    public Test(final int arrayIndex) {
        this.arrayIndex = arrayIndex;
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println(Runtime.getRuntime().availableProcessors());
        final long start = System.currentTimeMillis();
        runTest();
        System.out.println("duration = " + (System.currentTimeMillis() - start));
    }

    private static void runTest() throws InterruptedException {
        Thread[] threads = new Thread[NUM_THREAD];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(new Test(i));
        }
        for (Thread thread : threads) {
            thread.start();
        }
        for (Thread thread : threads) {
            thread.join();
        }
    }

    @Override
    public void run() {
        long i = INTERATIONS + 1;
        while (0 != --i) {
            longs[arrayIndex].value = i;
        }
    }

    public static final class VolatileLong {
        public volatile long value = 0L;
        public long p1, p2, p3, p4, p5, p6, p7; // 用来填充缓存行
    }
}
```

```txt
# 有填充代码
duration = 5100
# 无填充代码
duration = 32064
```

# 3. 参考链接

[CPU Cache的优化：解决伪共享问题](https://blog.csdn.net/fouy_yun/article/details/77824803)