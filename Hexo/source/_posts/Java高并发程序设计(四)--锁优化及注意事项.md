---
title: Java高并发程序设计(四)--锁优化及注意事项
date: 2019-03-01 15:38:09
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 提高锁性能的几点建议

锁优化的核心思想就是尽可能减少锁带来的额外开销.

## 1.1 减少锁持有时间

减少锁的持有时间有助于降低锁冲突的可能性,进而提升系统的并发能力.

对比两段代码

```java
    // 代码一
    public synchronized void syncMethod(){
        othercode1();
        metextMethod();
        othercode2();
    }
```

```java
    // 代码二
    public void syncMethod2(){
        othercode1();
        synchronized (this) {
            metextMethod();
        }
        othercode2();
    }
```

其中, othercode1()othercode2()都没有持有锁的必要性.

## 1.2 减少锁粒度

所谓减少锁粒度,就是指缩小锁定对象的范围,从而减少锁冲突的可能性,进而提高系统的并发能力.

ConcurrentHashMap就是典型的实例(相对于HashMap),使用分段锁减少了锁粒度.

缺点是,当系统需要取得全局锁时,其消耗的资源会比较多,因为当试图访问全局信息时,就会需要同时取得所有段的锁方能顺利实施.(参考ConcurrentHashMap的size方法)

## 1.3 读写分离锁来替换独占锁

本质上是减少锁粒度的一种情况,上面ConcurrentHashMap是通过分割结构实现,而读写分离锁则是对系统功能点的分割.适用于读多写少的场景.

## 1.4 锁分离

锁分离是读写锁思想的进一步延伸.说白了就是,根据应用程序的功能特点,使用分离思想.比如LinkedBlockingQueue中对队列的put和take,是两个不存在锁竞争的操作,如果用一把独占锁,就会导致锁竞争相对比较激烈,因此可以用两把不同的锁控制,从而减少冲突.

## 1.5 锁粗化

虚拟机在遇到一连串地对同一锁不断进行请求和释放的操作时,便会把所有的锁操作整合成对锁的一次请求,从而减少对锁的请求同步次数,这个操作叫做锁的粗化.

```java
// 优化前
synchronized (lock){
    // 1. do sth.
}
synchronized (lock){
    // 2. do sth.
}

// 优化后
synchronized (lock){
    // 1. do sth.
    // 2. do sth.
}
```

在开发过程中,应该有意识地在合理的场合进行锁的粗化,尤其是当在循环内请求锁时.

```java
// 优化前
for(int i = 0; i < CIRCLE; i++){
    synchronized (lock){
        // do sth.
    }
}

// 优化后
synchronized (lock){
    for(int i = 0; i < CIRCLE; i++){
        // do sth.
    }
}
```

锁粗化的思想和减少锁持有时间是相反的,在不同的场合,它们的效果不同,应该根据实际情况,进行权衡.

# 2. JVM在锁优化方面的工作

## 2.1 锁偏向

如果一个线程获得了锁,那么锁就进入了偏向模式,当这个线程再次请求锁时,无须再做任何同步操作,这样就节省了大量有关锁申请的操作,从而提高了程序性能.因此,对于几乎没有锁竞争的场合,偏向锁有比较好的优化效果,因为连续多次极有可能是同一个线程请求相同的锁.而对于锁竞争比较激烈的场合,其效果不佳.因为在竞争激烈的场合,最有可能的情况是每次都是不同的线程来请求相同的锁.这样偏向模式会失效,还不如不启用偏向锁.

可以使用JVM参数来显式开启偏向锁.

## 2.2 轻量级锁

如果偏向锁失败,虚拟机并不会立即挂起线程,它还会使用一种称为轻量级锁的优化手段.轻量级锁的操作也很轻便,它只是简单地将对象头部作为指针,指向持有锁的线程堆栈的内部,来判断一个线程是否持有对象锁.如果线程获得轻量级锁成功,则可以顺利进入临界区,如果轻量级锁加锁失败,则表示其他线程抢先争夺到了锁,那么当前线程的请求就会膨胀为重量级锁.

## 2.3 自旋锁

锁膨胀后,虚拟机为避免线程真实地在操作系统层面挂起,虚拟机还会做最后的努力--自旋锁,由于当前线程暂时无法获得锁,但是什么时候可以获得锁是一个未知数.也许在几个CPU时钟周期后,就可以得到锁.如果这样,简单粗暴地挂起线程可能是一种得不偿失的操作.因此,系统会进行一次赌注:它会假设在不久的将来,线程可以得到这把锁.因此,虚拟机会让当前线程做几个空循环,在经过若干次循环后,如果可以得到锁,那么就顺利进入临界区.如果还不能得到锁,才会真实地将线程在操作系统层面挂起.

## 2.4 锁消除

锁消除是一种更彻底的锁优化.Java虚拟机在JIT编译时,通过对上下文的扫描,去除不可能存在共享资源竞争的锁.通过锁消除,可以节省毫无意义的请求锁时间.

锁消除涉及的关键技术为逃逸分析,所谓逃逸分析,就是观察某一个变量是否会逃出某一个作用域.

逃逸分析必须在-server模式下进行,逃逸分析和锁消除都需要通过JVM参数开启.

# 3. ThreadLocal

ThreadLocal,顾名思义,线程的局部变量,只有当前线程可以访问.

## 3.1 基本使用

为每一个线程分配不同的线程对象,需要在应用层面保证.ThreadLocal只是起到了简单的容器作用.

```java
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class Course {
    private static ThreadLocal<SimpleDateFormat> t1 = new ThreadLocal<SimpleDateFormat>();
    public static class ParseDate implements Runnable{
        private int i = 0;
        public ParseDate(int i) {
            this.i = i;
        }
        @Override
        public void run() {
            try {
                if(t1.get() == null){
                    t1.set(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
                }
                Date t = t1.get().parse("2019-03-07 15:29:" + i%60);
                System.out.println(i + ":" + t);
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for(int i = 0; i < 1000; i++){
            executorService.execute(new ParseDate(i));
        }
    }
}
```

## 3.2 实现原理

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

思考: 这些变量是维护在Thread类内部的,这也意味着只要线程不退出,对象的引用将一直存在.线程退出时,Thread类会进行一些清理工作,包括清理ThreadLocalMap.

```java
    private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
```

如果使用线程池,将一些大对象设置到ThreadLocal中,就可能会出现内存泄漏.如果希望及时回收对象,最好使用ThreadLocal.remove().

另外, 通过 t1 = null 也可以实现对象的回收, 其中t1是ThreadLocal. 具体原理如下图.

![ThreadLocal回收机制](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190307185137.jpg)

## 3.3 ThreadLocal有何好处

如果共享对象对于竞争的处理容易引起性能损失, 我们就应该考虑使用ThreadLocal为每个线程分配单独的对象.

ThreadLocal 并不是为了解决线程安全问题,而是提供了一种将实例绑定到当前线程的机制,类似于隔离的效果,实际上自己在方法中 new 出来变量也能达到类似的效果.

ThreadLocal 最大的用处就是用来把实例变量共享成全局变量,在程序的任何方法中都可以访问到该实例变量而已.

<font color=red>也就是说 ThreadLocal 的用法和我们自己 new 对象一样, 然后将这个 new 的对象传递到各个方法中. 但是到处传递的话, 太麻烦了. 这个时候,就应该用 ThreadLocal.</font>

# 4. 无锁

锁是一种悲观的策略,它总是假设每一次的临界区操作会产生冲突.
无锁是一种乐观的策略,它会假设对资源的访问是没有冲突的.无锁策略核心是CAS(Compare And Set)

## 4.1 并发策略CAS

优点:非阻塞性导致对死锁问题天生免疫;线程间相互影响远小于基于锁的方式;没有锁竞争带来的系统开销,也没有线程间频繁调度带来的开销.

CAS算法的过程是这样:它包含三个参数 CAS(V,E,N). V表示要更新的变量,E表示预期的值,N表示新值. 仅当V值等于E值时,才会将V的值设置成N,否则什么都不做.最后CAS返回当前V的值.CAS算法需要你额外给出一个期望值,也就是你认为现在变量应该是什么样子,如果变量不是你想象的那样,那说明已经被别人修改过.你就重新读取,再次尝试修改即可.

## 4.2 无锁的线程安全整数AtomicInterger

JDK并发包中的atomic包,里面实现了一些直接使用CAS操作的线程安全的类型.

## 4.3 无锁的对象引用AtomicReference

AtomicReference是对应普通的对象引用.也就是它可以保证你再修改对象引用时的线程安全性.

### 4.3.1 ABA问题

比如说一个线程one从内存位置V中取出A,这时候另一个线程two也从内存中取出A,并且two进行了一些操作变成了B,然后two又将V位置的数据变成A,这时候线程one进行CAS操作发现内存中仍然是A,然后one操作成功.尽管线程one的CAS操作成功,但是不代表这个过程就是没有问题的.如果链表的头在变化了两次后恢复了原值.但是不代表链表就没有变化.

## 4.4 带有时间戳的对象引用AtomicStampedReference

针对ABA问题,AtomicReference会存在问题,于是有了AtomicStampedReference来解决ABA问题.

AtomicStampedReference它内部不仅维护了对象的值,还维护了一个时间戳(实际上可以使用任何一个整数来表示状态值),当AtomicStampedReference对应的数值被修改时,除了更新数据本身外,还必须要更新时间戳.当AtomicStampedReference设置对象值时,对象值以及时间戳都必须满足期望值,写入才会成功.因此即使对象值被反复读写,写回原值,只要时间戳发生变化,就能防止不恰当的写入.

## 4.5 无锁数组AtomicIntergerArray

本质上是对int\[\]类型的封装,并用CAS控制int\[\]在多线程下的安全性.

## 4.6 普通变量享受原子操作AtomicIntegerFieldUpdater

AtomicIntegerFieldUpdater可以让普通变量也享受CAS操作带来的线程安全性.

几点注意:

a.只能修改可见范围内的变量.
b.变量必须是volatile类型.
c.不支持static字段.

## 4.7 让线程之间互相帮助SynchronousQueue

SynchronousQueue,数据交换通道.

# 5. 死锁

死锁一旦发生,如果没有外力介入,这种等待将永远存在,从而对程序产生严重的影响.

避免死锁的办法:使用无锁函数 或者 使用重入锁的中断或者限时等待.

# 6. 参考链接

<< Java高并发程序设计 >>(葛一鸣 郭超)

[ThreadLocal的彻底理解（ThreadLocal不是用来解决多线程下访问共享变量问题的）](https://blog.csdn.net/qq_35232663/article/details/82914537)