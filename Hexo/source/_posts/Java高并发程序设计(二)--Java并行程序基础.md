---
title: Java高并发程序设计(二)--Java并行程序基础
date: 2019-02-22 11:07:12
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 线程的状态

![线程状态图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81%E5%9B%BE20190220.png)

```java
// Thread中定义的State
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

# 2. 线程的操作

## 2.1 新建线程

```java
// 新建线程
Thread t1 = new Thread();
t1.start();

//错误的做法
Thread t1 = new Thread();
t1.run();
```

t1.run()只是调用了run方法,并不是新建线程.

## 2.2 终止线程

```java
// 不推荐的做法
t1.stop();

// 推荐的做法
volatile boolean stopme = false;
public void stopMe(){
    stopme = true;
}
@Override
public void run(){
    while(true){
        if(stopme == true){
            System.out.printf("bye bye");
            break;
        }

        // do something
    }
}
```

使用stop容易造成数据不一致.

## 2.3 线程中断

线程中断不会使线程立即退出,而是给线程发送一个通知,告知目标线程,有人希望你退出了.至于目标线程接到通知后如何处理,则完全由目标线程自行决定.

```java
public class Test {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new Thread(){
            @Override
            public void run(){
                while (true){
                    if(Thread.currentThread().isInterrupted()){
                        System.out.println("Interrupted");
                        break;
                    }
                    Thread.yield();
                    System.out.println("working");
                }
            }
        });
        t1.start();
        Thread.sleep(2000);
        t1.interrupt();
    }
}
```

## 2.4 线程等待和线程通知

```java
public class Test {
    public static void main(String[] args) {
        Thread t1 = new T1();
        Thread t2 = new T2();
        t1.start();
        t2.start();
    }
    final static Object object = new Object();
    public static class T1 extends Thread{
        @Override
        public void run() {
            synchronized (object){
                System.out.println(System.currentTimeMillis()+":T1 start!");
                System.out.println(System.currentTimeMillis()+":T1 wait!");
                try {
                    object.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(System.currentTimeMillis()+":T1 end!");
            }
        }
    }
    public static class T2 extends Thread{
        @Override
        public void run() {
            synchronized (object){
                System.out.println(System.currentTimeMillis()+":T2 start! notify T1.");
                object.notify();
                System.out.println(System.currentTimeMillis()+":T2 end!");
            }
        }
    }
}
```

wait方法不是可以随便调用,必须在对应的同步代码块里,wait或者notify都需要先获得目标对象的监视器.线程执行wait方法前必须先获得对应Object的监视器,wait方法执行后会释放监视器,这时其他线程就可以获取这个Object的监视器了.这就实现了线程间通信.

<font color=red>wait和sleep都是让线程等待.wait会释放目标对象锁,sleep不会释放任何资源.</font>

## 2.5 挂起线程和继续执行线程

suspend和resume是一对相关的操作,也已经废弃了,<font color=red>不推荐使用</font>.

suspend挂起线程后不会释放任何资源,其他等待被占用资源的线程都无法执行,使用不当会导致所有相关线程都无法运行.

suspend挂起线程后,线程还是Runnable状态,影响问题分析.

相关需求可以用wait和notify来实现, 下面是例子.

```java
public class Test {
    public static Object u = new Object();

    public static class ChangeObjectThread extends Thread{
        volatile boolean suspendme = false;
        public void suspendMe(){
            suspendme = true;
        }
        public void resumeMe(){
            suspendme = false;
            synchronized (this){
                notify();
            }
        }

        @Override
        public void run() {
            while (true){
                synchronized (this){
                    while (suspendme){
                        try {
                            wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }

                synchronized (u){
                    System.out.println("in ChangeObjectThread");
                }
                Thread.yield();
            }
        }
    }

    public static class ReadObjectThread extends Thread{
        @Override
        public void run() {
            while (true){
                synchronized (u){
                    System.out.println("in ReadObjectThread");
                }
                Thread.yield();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ChangeObjectThread t1 = new ChangeObjectThread();
        ReadObjectThread t2 = new ReadObjectThread();
        t1.start();
        t2.start();
        Thread.sleep(2000);
        t1.suspendMe();
        System.out.println("suspend t1 2 s");
        Thread.sleep(2000);
        System.out.println("resume t1");
        t1.resumeMe();
    }
}
```

## 2.6 等待线程结束和谦让

等待线程结束(join), 谦让(yield)

join方法会阻塞当前线程,直到目标线程执行结束.本质就是让当前线程wait()在目标线程对象上,目标线程执行完成后会调用notifyAll通知所有等待线程继续执行.

```java
public class Test {
    public volatile static int i = 0;

    public static class AddThread extends Thread{
        @Override
        public void run() {
            for(i = 0; i < 10000000; i++){

            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
      AddThread addThread = new AddThread();
      addThread.start();
      addThread.join();
      System.out.println(i);
    }

}


```

Thread.yield()方法会让当前线程让出CPU,不过让出后还会进行CPU资源的争夺,能否再次分配到就看系统了.

# 3. 线程组

用来管理线程. 建议在创建线程和线程组的时候,取一个好听的名字.

下个给个例子,了解一下有这么回事.

```java
public class Test {

    public static void main(String[] args){
        ThreadGroup tg = new ThreadGroup("PrintGroup");
        Thread thread1 = new Thread(tg, new Service(), "T1");
        Thread thread2 = new Thread(tg, new Service(), "T2");
        thread1.start();
        thread2.start();
        System.out.println(tg.activeCount());
        tg.list();
    }

    public static class Service extends Thread{
        @Override
        public void run() {
            String groupAndName = Thread.currentThread().getThreadGroup().getName() + "-" + Thread.currentThread().getName();
            while (true){
                System.out.println("I am "+ groupAndName);
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

# 4. 守护线程

守护线程是一种特殊的线程,就和它的名字一样,它是系统的守护者,在后台默默完成一些系统性的服务.当一个Java应用内只有守护线程时,Java虚拟机就会自然退出.

```java
public class Test {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new DaemonDemo();
        thread.setDaemon(true);
        thread.start();
        Thread.sleep(2000);
    }
    public static class DaemonDemo extends Thread{
        @Override
        public void run() {
            while(true){
                System.out.println("I am alive.");
            }
        }
    }
}
```

# 5. 线程优先级

直接看例子. 注意: 高优先级也可能抢占失败,这只是一个概率问题.

```java
public class Test {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new HighPriority();
        Thread t2 = new LowPriority();

        t1.setPriority(Thread.MAX_PRIORITY);
        t2.setPriority(Thread.MIN_PRIORITY);

        t2.start();
        t1.start();
    }
    public static class HighPriority extends Thread{
        int count = 0;
        @Override
        public void run() {
            while (true){
                synchronized (Test.class){
                    count++;
                    if(count > 10000000){
                        System.out.println("HighPriority is completed");
                        break;
                    }
                }
            }
        }
    }
    public static class LowPriority extends Thread{
        int count = 0;
        @Override
        public void run() {
            while (true){
                synchronized (Test.class){
                    count++;
                    if(count > 10000000){
                        System.out.println("LowPriority is completed");
                        break;
                    }
                }
            }
        }
    }
}
```

# 6. 参考链接

<< Java高并发程序设计 >>(葛一鸣 郭超)

[Java高并发程序设计](https://www.jianshu.com/p/c7a56825c658)