---
title: Java高并发程序设计(五)--并行模式
date: 2019-03-11 11:44:42
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 单例模式

```java
public class StaticSingleton {
    private StaticSingleton(){
        System.out.println("StaticSingleton is create");
    }
    private static class SingletonHolder{
        private static StaticSingleton instance = new StaticSingleton();
    }
    public static StaticSingleton getInstance(){
        return SingletonHolder.instance;
    }
}
```

相比传统的单例模式的优点:
1.只有在第一次使用的时候,实例才会被创建.
2.没有锁,高并发环境下性能优越.

# 2. 不变模式

在并行软件的开发过程中,同步操作似乎是不可避免的,当多线程对同一个对象进行读写操作时,为了保证数据一致性和正确性,有必要对对象进行同步.而同步操作对系统的性能是有相当的损耗的,为了尽可能的取出这些同步操作,提高程序并行能力,可以使用一种不可变对象,依靠对象的不变性,可以确保其在没有同步操作的多线程环境中依然时钟保持内部状态一致性和正确性,这就是不变模式.

不变模式天生就是多线程友好的,它的核心思想是,一个对象一旦被创建,则它的状态将永远不会发生改变.所以,没有一个线程可以修改其内部状态和数据,同时其内部状态也绝不会自行发生改变.基于这些特性,对不变对象的多线程操作不需要进行同步控制.

不变模式和只读模式是有一定区别的.不变模式是比只读属性具有更强一致性和不变性.对只读属性的对象而言,对象本身不能被其他线程修改,但是对象的自身状态却可能自行修改.比如,一个对象的存活时间是只读的,但是这个属性,随着时间的推移,是时刻变化的.

不变模式的主要使用场景需要满足2个条件:
1.当前对象创建后,其内部状态和数据不再发生任何变化;
2.对象需要被共享,被多线程频繁访问.

在Java语言中,不变模式的实现很简单.为确保对象被创建后,不发生任何改变,并保证不变模式正常工作,只需注意以下4点:
1.取出所有setter方法及所有修改自身属性的方法;
2.将所有属性设置为私有,并用final修饰,确保其不可修改;
3.确保没有子类可以重载修改它的行为,即final class;
4.有一个可以创建完整对象的构造函数.

在JDK中,不变模式的应用非常广泛.其中,最为典型的就是java.lang.String类.此外,所有的元数据类包装类,都是使用不变模式实现的.主要的不变模式类型如下:

```java
java.lang.String
java.lang.Boolean
java.lang.Byte
java.lang.Character
java.lang.Double
java.lang.Float
java.lang.Integer
java.lang.Long
java.lang.Short
```

由于基本数据类型和String类型在实际的软件开发中应用及其广泛,使用不变模式后,所有实例的方法均不需要进行同步操作,保证了它们在多线程环境下的性能.

# 3. 生产者-消费者模式

生产者-消费者模式是一个经典的多线程设计模式.它为多线程间的协作提供了良好的解决方案.在生产者-消费者模式中,通常由两类线程,即若干个生产者线程和若干个消费者线程.生产者线程负责提交用户请求,消费者线程则负责具体处理生产者提交的任务.生产者和消费者之间则通过共享内存缓冲区进行通信.

## 3.1 示例

生产者

```java
import java.util.Random;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class Producer implements Runnable{
    private volatile boolean isRunning = true;
    private BlockingQueue<PCData> queue;
    private static AtomicInteger count = new AtomicInteger();
    private static final int SLEEPTIME = 1000;

    public Producer(BlockingQueue<PCData> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        PCData data = null;

        System.out.println("start producer id = " + Thread.currentThread().getId());

        try {
            while (isRunning){
                Thread.sleep(SLEEPTIME);
                data = new PCData(count.incrementAndGet());
                System.out.println(data + "is put into queue");
                if(!queue.offer(data, 2, TimeUnit.SECONDS)){
                    System.err.println("failed to put data: " + data);
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        }
    }

    public void stop(){
        isRunning = false;
    }
}
```

消费者

```java
import java.text.MessageFormat;
import java.util.Random;
import java.util.concurrent.BlockingQueue;

public class Consumer implements Runnable {

    private BlockingQueue<PCData> queue;
    private static final int SLEEPTIME = 1000;

    public Consumer(BlockingQueue<PCData> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        System.out.println("start Consumer id = " + Thread.currentThread().getId());
        Random r = new Random();

        try {
            while (true){
                PCData data = queue.take();
                if(data != null){
                    int re = data.getData() * data.getData();
                    System.out.println(MessageFormat.format("{0} * {1} = {2}", data.getData(), data.getData(), re));
                    Thread.sleep(r.nextInt(SLEEPTIME));
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        }
    }
}
```

任务

```java
public class PCData {
    private final int data;

    public PCData(int data) {
        this.data = data;
    }

    public int getData() {
        return data;
    }

    @Override
    public String toString() {
        return "PCData{" +
                "data=" + data +
                '}';
    }
}
```

客户端

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<PCData> queue = new LinkedBlockingQueue<>();
        Producer producer1 = new Producer(queue);
        Producer producer2 = new Producer(queue);
        Producer producer3 = new Producer(queue);
        Consumer consumer1 = new Consumer(queue);
        Consumer consumer2 = new Consumer(queue);
        Consumer consumer3 = new Consumer(queue);

        ExecutorService service = Executors.newCachedThreadPool();
        service.execute(producer1);
        service.execute(producer2);
        service.execute(producer3);
        service.execute(consumer1);
        service.execute(consumer2);
        service.execute(consumer3);

        Thread.sleep(10 * 1000);
        producer1.stop();
        producer2.stop();
        producer3.stop();
        Thread.sleep(3000);
        service.shutdown();
    }
}
```

高性能的生产者-消费者可以通过无锁的方式的实现,如第三方框架Disruptor.

# 4. Futrue模式

Future模式是多线程开发中非常常见的一种设计模式.它的核心思想是异步调用.当我们需要调用一个函数方法时,如果这个函数执行很慢,那么我们就要进行等待.但有时候,我们可能并不急着要结果.因此,我们可以让被调用者立即返回,让他在后台慢慢处理这个请求.对于调用者来说,则可以先处理一些其他任务,在真正需要数据的场合再去尝试获取需要的数据.

![Futrue模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/2199790-15f7063b326e6039.webp)

## 4.1 示例

![Futrue模式](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190311211338.jpg)

Data

```java
public interface Data {
    public String getResult();
}
```

RealData

```java
public class RealData implements Data {

    protected final String result;

    public RealData(String result) {
        StringBuffer stringBuffer = new StringBuffer();
        for(int i = 0; i < 10; i++){
            stringBuffer.append(result);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        this.result = stringBuffer.toString();
    }

    @Override
    public String getResult() {
        return result;
    }
}
```

FutureData

```java
public class FutureData implements Data {

    protected RealData realData = null;
    protected boolean isReady = false;

    public synchronized void setRealData(RealData realData){
        if(isReady == true){
            return;
        }
        this.realData = realData;
        isReady = true;
        notifyAll();
    }

    @Override
    public synchronized String getResult() {
        while (!isReady){
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return realData.getResult();
    }
}
```

Client

```java
public class Client {
    public Data request(final String queryStr){
        final FutureData futureData = new FutureData();
        new Thread(){
            @Override
            public void run() {
                RealData realData = new RealData(queryStr);
                futureData.setRealData(realData);
            }
        }.start();
        return futureData;
    }
}
```

Main

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Client client = new Client();
        Data data = client.request("name");
        System.out.println("请求完毕");
        Thread.sleep(2000);
        System.out.println("数据 = " + data.getResult());
    }
}
```

## 4.2 说明

JDK内部已经有一套完整的实现.可以直接使用.

# 5. 并行流水线

并行算法可以充分发挥多核CPU的性能,但是,并非所有的计算都可以改造成并发的形式.

执行过程中有数据相关性的运算都是无法完美并行化的.

解决的思路就是流水线的思想.

![并行流水线](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190312105956.jpg)

# 6. 参考文献

<< Java高并发程序设计 >>(葛一鸣 郭超)

[Java多线程 - Future模式](https://www.jianshu.com/p/949d44f3d9e3)