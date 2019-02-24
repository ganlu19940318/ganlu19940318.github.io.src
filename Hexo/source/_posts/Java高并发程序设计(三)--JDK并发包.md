---
title: Java高并发程序设计(三)--JDK并发包
date: 2019-02-23 09:52:24
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 同步控制

这部分介绍多线程控制方法.

## 1.1 重入锁

重入锁是synchronized的替代品,但是JDK6.0开始,synchronized做了大量的优化,两者性能差距不大.

```java
import java.util.concurrent.locks.ReentrantLock;
public class Test {
    public static class Demo extends Thread{
        public static ReentrantLock reentrantLock = new ReentrantLock();
        public static int i = 0;
        @Override
        public void run() {
            for(int j = 0; j < 1000000; j++){
                reentrantLock.lock();
                i++;
                reentrantLock.unlock();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Demo();
        Thread thread2 = new Demo();
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        System.out.println(Demo.i);
    }
}
```

**重入锁特性**
重入锁之所以叫这个名字,是因为这种锁可以反复进入,就是说,一个线程可以连续两次获得同一把锁,但是在释放锁的时候,也必须释放相同次数.

```java
reentrantLock.lock();
reentrantLock.lock();
i++;
reentrantLock.unlock();
reentrantLock.unlock();
```

**中断响应**
就是让锁可以响应中断,使用lock1.lockInterruptibly()来进行上锁即可收到中断请求.

```java
import java.util.concurrent.locks.ReentrantLock;
public class Test {
    public static ReentrantLock lock1 = new ReentrantLock();
    public static ReentrantLock lock2 = new ReentrantLock();
    public static class Demo extends Thread{
        int lock;
        public Demo(int lock) {
            this.lock = lock;
        }
        @Override
        public void run() {
            try {
                if(lock == 1){
                    lock1.lockInterruptibly();
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        System.out.println("Interrupted Point1");
                    }
                    lock2.lockInterruptibly();
                }else {
                    lock2.lockInterruptibly();
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        System.out.println("Interrupted Point2");
                    }
                    lock1.lockInterruptibly();
                }
            } catch (InterruptedException e) {
                System.out.println("Interrupted Point3");
            }finally {
                if(lock1.isHeldByCurrentThread()){
                    lock1.unlock();
                }
                if(lock2.isHeldByCurrentThread()){
                    lock2.unlock();
                }
                System.out.println(Thread.currentThread().getId()+":线程退出.");
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Demo(1);
        Thread thread2 = new Demo(2);
        thread1.start();
        thread2.start();
        Thread.sleep(1000);
        thread2.interrupt();
    }
}
```

**锁申请等待限时**
会不停地去获取锁,但是最大等待时间不会超过给定的值.
通过lock.tryLock()实现,带参用法如下实例,不带参可以理解为时长为0.
获取锁成功返回值为true,否则返回值为false.

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;
public class Test {
    public static ReentrantLock lock = new ReentrantLock();
    public static class Demo extends Thread{
        @Override
        public void run() {
            try {
                if(lock.tryLock(5, TimeUnit.SECONDS)){
                    System.out.println(Thread.currentThread().getId()+":get lock success");
                    Thread.sleep(6000);
                }else {
                    System.out.println(Thread.currentThread().getId()+":get lock failed");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if(lock.isHeldByCurrentThread()){
                    lock.unlock();
                }
            }
        }
    }
    public static void main(String[] args){
        Thread t1 = new Demo();
        Thread t2 = new Demo();
        t1.start();
        t2.start();
    }
}
```

**公平锁**
公平锁保证先到者先得,后到者后得.不会产生饥饿现象.
可重入锁默认是非公平锁.公平锁性能相对非常低下,因为要求系统维护一个有序队列.
synchronized锁是非公平锁.
通过new ReentrantLock(true)设置参数指定开启公平锁,true表示开启.

```java
import java.util.concurrent.locks.ReentrantLock;
public class Test {
    public static ReentrantLock lock = new ReentrantLock(true);
    public static class Demo extends Thread{
        @Override
        public void run() {
            while(true){
                lock.lock();
                System.out.println(Thread.currentThread().getId()+":获得锁");
                lock.unlock();
            }
        }
    }
    public static void main(String[] args){
        Thread t1 = new Demo();
        Thread t2 = new Demo();
        t1.start();
        t2.start();
    }
}
```

## 1.2 Condition条件

与 重入锁 配合使用, 达到synchronized锁中的wait notify的效果.
condition.await()会使当前线程等待并释放锁.
使用condition.signal()之前一定要先获得锁,用完之后一定要记得释放锁.

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;
public class Test {
    public static ReentrantLock lock = new ReentrantLock();
    public static Condition condition = lock.newCondition();
    public static class Demo extends Thread{
        @Override
        public void run() {
            lock.lock();
            try {
                condition.await();
                System.out.println("Thread is going on");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Demo();
        t1.start();
        Thread.sleep(2000);
        lock.lock();
        condition.signal();
        lock.unlock();
    }
}
```

## 1.3 信号量Semaphore

可以指定多个线程同时访问某一个资源,并指定准入数量.
有很多用法,跟ReentrantLock类似,下面给一个典型的demo.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
public class Test {
    public static Semaphore semaphore = new Semaphore(5);
    public static class Demo extends Thread{
        @Override
        public void run() {
            try {
                semaphore.acquire();
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getId()+":DONE!");
                semaphore.release();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args){
        ExecutorService executorService = Executors.newFixedThreadPool(20);
        Thread thread = new Demo();
        for(int i = 0; i < 20; i++){
            executorService.submit(thread);
        }
    }
}
```

## 1.4 读写锁ReadWriteLock

读写分离锁可以减少锁竞争,因为如果用重入锁,所有的读之间,读写之间,写写之间都是串行的.但是从日常需求上看,读读应该是允许并行的.适用于读多写少.

```java
import java.util.Random;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;
public class Test {
    public static Lock lock = new ReentrantLock();
    public static ReentrantReadWriteLock reentrantReadWriteLock = new ReentrantReadWriteLock();
    public static Lock readLock = reentrantReadWriteLock.readLock();
    public static Lock writeLock = reentrantReadWriteLock.writeLock();
    public int value;
    public int read(Lock lock){
        try {
            lock.lock();
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
            return value;
        }
    }
    public void write(Lock lock, int value){
        try {
            lock.lock();
            Thread.sleep(2000);
            this.value = value;
        }catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    public static void main(String[] args){
        Test test = new Test();
        Runnable readRunnable = new Runnable() {
            @Override
            public void run() {
                test.read(readLock);
//               test.read(lock);
            }
        };
        Runnable writeRunnable = new Runnable() {
            @Override
            public void run() {
                test.write(writeLock, new Random().nextInt());
//                test.write(lock, new Random().nextInt());
            }
        };
        for(int i = 0; i < 18; i++){
            new Thread(readRunnable).start();
        }
        for(int i = 18; i < 20; i++){
            new Thread(writeRunnable).start();
        }
    }
}
```

## 1.5 倒计时器CountDownLatch

让线程等待,直到倒计时结束(满足停止等待的条件),再开始执行.

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class Test {
    public static CountDownLatch end = new CountDownLatch(10);
    public static class Demo implements Runnable {
        @Override
        public void run() {
            try {
                Thread.sleep(new Random().nextInt(10) * 1000);
                System.out.println("Check complete");
                end.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for(int i = 0; i < 10; i++){
            executorService.submit(new Demo());
        }
        end.await();
        System.out.println("Fire");
        executorService.shutdown();
    }
}
```

## 1.6 循环栅栏CyclicBarrier

功能和 倒计时器CountDownLatch 类似, 但是 循环栅栏CyclicBarrier 功能更加强大. 表现在 可以反复使用.
用法:new CyclicBarrier(N, new BarrierRun(flag, N)),N表示计数器,new BarrierRun(flag, N)表示每次计数器满足之后所执行的操作.

```java
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
public class Test {
    public static class Soldier implements Runnable{
        private String soldier;
        private final CyclicBarrier cyclicBarrier;
        public Soldier(CyclicBarrier cyclicBarrier, String soldierName) {
            this.cyclicBarrier = cyclicBarrier;
            this.soldier = soldierName;
        }
        @Override
        public void run() {
            try {
                cyclicBarrier.await();
                dowork();
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
        void dowork(){
            try {
                Thread.sleep(Math.abs(new Random().nextInt()%10000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(soldier+":任务完成");
        }
    }
    public static class BarrierRun implements Runnable{
        boolean flag;
        int N;
        public BarrierRun(boolean flag, int n) {
            this.flag = flag;
            N = n;
        }
        @Override
        public void run() {
            if(flag){
                System.out.println("司令:[士兵"+N+"个,完成任务!]");
            }else {
                System.out.println("司令:[士兵"+N+"个,集合完毕!]");
                flag = true;
            }
        }
    }
    public static void main(String[] args){
        int N = 10;
        Thread [] allSoldier = new Thread[N];
        boolean flag = false;
        CyclicBarrier cyclicBarrier = new CyclicBarrier(N, new BarrierRun(flag, N));
        System.out.println("集合队伍!");
        for(int i = 0; i < N; i ++){
            System.out.println("士兵"+ i + "报道!");
            allSoldier[i] = new Thread(new Soldier(cyclicBarrier, "士兵"+i));
            allSoldier[i].start();
        }
    }
}
```

## 1.7 线程阻塞工具类LockSupport

和suspend+resume相比, 它不会因为resume而造成线程无法继续执行.
和wait+notify相比, 它不需要先获取某个对象的锁.
我觉得这个类本质上应该是用来替换suspend+resume的,因为线程挂起后并不会释放锁.

```java
import java.util.concurrent.locks.LockSupport;
public class Test {
    public static Object object = new Object();
    public static class ChangeObjectThread extends Thread{
        public ChangeObjectThread(String name) {
            super.setName(name);
        }
        @Override
        public void run() {
            synchronized (object){
                System.out.println("in " + getName());
                LockSupport.park();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new ChangeObjectThread("t1");
        Thread t2 = new ChangeObjectThread("t2");
        t1.start();
        t2.start();
        LockSupport.unpark(t1);
        LockSupport.unpark(t2);
        t1.join();
        t2.join();
    }
}
```

# 2. 线程池

**为什么需要线程池?**
线程的创建和关闭需要花费时间.
线程本身也是要占用内存空间.
线程的回收会给GC带来压力,延长GC时间.

在实际生产环境中,线程的数量必须得到控制,盲目的大量创建线程对系统性能是有伤害的.

使用线程池后,创建线程变成了从线程池获得空闲线程,关闭线程变成了向池子归还线程.

## 2.1 使用线程池

### 2.1.1 固定大小的线程池

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class Test {
    public static class MyTask implements Runnable{
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis()+":Thread ID:"+Thread.currentThread().getId());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args){
        MyTask myTask = new MyTask();
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        for(int i = 0; i < 10; i++){
            executorService.submit(myTask);
        }
    }
}
```

### 2.1.2 计划任务

周期性的执行任务.
scheduleAtFixedRate和scheduleWithFixedDelay的区别.
![scheduleAtFixedRate和scheduleWithFixedDelay的区别](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190224155524.jpg)

如果任务遇到异常,那么后续的所有子任务都会停止调度,因此,必须保证异常被及时处理,为周期性任务的稳定调度提供条件.

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
public class Test {
    public static class MyTask implements Runnable{
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis()+":Thread ID:"+Thread.currentThread().getId());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args){
        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(10);
        scheduledExecutorService.scheduleAtFixedRate(new MyTask(), 0, 2, TimeUnit.SECONDS);
        scheduledExecutorService.scheduleWithFixedDelay(new MyTask(), 0, 2, TimeUnit.SECONDS);
    }
}
```

### 2.1.3 核心线程池内部实现

都是基于下面这个.

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
```

### 2.1.4 拒绝策略

当任务数量超过系统实际承载能力时(或者线程池规定的能力),该如何处理.

### 2.1.5 自定义线程创建ThreadFactory

```java
import java.util.concurrent.*;
public class Test {
    public static class MyTask implements Runnable{
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis()+":Thread ID:"+Thread.currentThread().getId());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        MyTask myTask = new MyTask();
        ExecutorService executorService = new ThreadPoolExecutor(5, 5, 0L, TimeUnit.MILLISECONDS, new SynchronousQueue<Runnable>(), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setDaemon(true);
                System.out.println("create "+t);
                return t;
            }
        });
        for(int i = 0; i < 5; i++){
            executorService.submit(myTask);
        }
        Thread.sleep(2000);
    }
}
```

### 2.1.6 扩展线程池

```java
import java.util.concurrent.*;
public class Test {
    public static class MyTask implements Runnable{
        public String name;

        public MyTask(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis()+":Thread ID:"+Thread.currentThread().getId());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = new ThreadPoolExecutor(5, 5, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>()){
            @Override
            protected void beforeExecute(Thread t, Runnable r) {
                System.out.println("准备执行:"+((MyTask)r).name);
            }
            @Override
            protected void afterExecute(Runnable r, Throwable t) {
                System.out.println("执行完成:"+((MyTask)r).name);
            }
            @Override
            protected void terminated() {
                System.out.println("线程池退出");
            }
        };
        for(int i = 0; i < 5; i++){
            MyTask myTask = new MyTask("TASK-"+i);
            executorService.execute(myTask);
            Thread.sleep(10);
        }
        executorService.shutdown();
    }
}
```

### 2.1.7 优化线程池线程数量

最优的池的大小等于:
Nthreads = Ncpu \* Ucpu \* (1 + W/C);

Ncpu: CPU数量
Ucpu: 目标CPU的使用率, 0~1.
W/C: 等待时间与计算时间的比率

### 2.1.8 Fork/Join框架

这是一个分而治之的框架.直接看个demo.

```java
import java.util.ArrayList;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveTask;
public class CountTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10000;
    private long start;
    private long end;
    public CountTask(long start, long end) {
        this.start = start;
        this.end = end;
    }
    @Override
    protected Long compute() {
        long sum = 0;
        boolean canCompute = (end - start) < THRESHOLD;
        if(canCompute){
            for(long i = start; i <= end; i++){
                sum = sum + i;
            }
        }else {
            long step = (end - start) / 100;
            ArrayList<CountTask> subTasks = new ArrayList<CountTask>();
            long pos = start;
            for(int i = 0; i < 100; i++){
                long lastOne = pos + step;
                if(lastOne > end) lastOne = end;
                CountTask countTask = new CountTask(pos, lastOne);
                pos = lastOne + 1;
                subTasks.add(countTask);
                countTask.fork();
            }
            for(CountTask countTask : subTasks){
                sum += countTask.join();
            }
        }
        return sum;
    }
    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        CountTask task = new CountTask(0 ,200000L);
        ForkJoinTask<Long> result = forkJoinPool.submit(task);
        try {
            long res = result.get();
            System.out.println("sum = "+ res);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

# 3. JDK的并发容器

## 3.1 线程安全的HashMap

```java
// 方法一: 性能低,线程安全
Map m = Collections.synchronizedMap(new HashMap<>())

// 方法二: 性能高,线程安全
ConcurrentHashMap concurrentHashMap = new ConcurrentHashMap();
```

## 3.2 List的线程安全

```java
// 线程不安全, 底层数组
ArrayList arrayList = new ArrayList();
// 线程安全, 底层数组
Vector vector = new Vector();

// 线程不安全, 底层链表
LinkedList linkedList = new LinkedList();
// 线程安全, 底层链表
List list = Collections.synchronizedList(new LinkedList<>());
```

## 3.3 高效读写的队列

```java
// 高并发环境中性能最好的队列,线程安全.
ConcurrentLinkedQueue concurrentLinkedQueue = new ConcurrentLinkedQueue();
```

## 3.4 高效读取CopyOnWriteArrayList

适用于读远大于写的场景.
只有写入与写入之间需要同步等待,其他情况都不用加锁.

```java
CopyOnWriteArrayList copyOnWriteArrayList = new CopyOnWriteArrayList();
```

## 3.5 数据共享通道BlockingQueue

用于多线程之间的数据共享. 这是一个接口, 可以选择很多实现.

```java
BlockingQueue blockingQueue = new LinkedBlockingQueue();
```

## 3.6 跳表SkipList

用来快速查找的数据结构,类似于平衡树.

与平衡树的区别在于: 对平衡树的插入和删除往往很可能导致平衡树进行一次全局的调整.而对跳表的插入和删除只需要对整个数据结构的局部进行操作即可.

这样做的好处在于: 在高并发的情况下,你会需要一个全局锁来保证整个平衡树的线程安全.而对于跳表,你只需要部分锁即可,从而拥有更好的性能.

就查询性能来说, 跳表的时间复杂度也是O(log n),所以在并发数据结构中,使用跳表来实现Map.

```java
ConcurrentSkipListMap concurrentSkipListMap = new ConcurrentSkipListMap();
```

# 4. 参考文献

<< Java高并发程序设计 >>(葛一鸣 郭超)