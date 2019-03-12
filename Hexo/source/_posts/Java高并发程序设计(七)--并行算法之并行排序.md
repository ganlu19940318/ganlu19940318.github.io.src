---
title: Java高并发程序设计(七)--并行算法之并行排序
date: 2019-3-12 15:57:22
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 分离数据相关性:奇偶交换排序

选择排序.

冒泡排序的过程

![冒泡排序](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190312171724.jpg)

奇偶交换排序的过程

![奇偶交换排序](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190312171738.jpg)

奇偶交换排序说白了就是将整个比较交换分割为奇阶段和偶阶段,使得每一个阶段内的所有比较和交换都是没有数据相关性的.因此,每一次比较和交换都可以独立执行,也就可以并行化了.

## 1.1 奇偶交换排序的串行实现

```java
public static void oddEvenSort(int [] arr){
    int exchFlag = 1;
    int start = 0;
    while (exchFlag == 1 || start == 1){
        exchFlag = 0;
        for(int i = start; i < arr.length - 1; i += 2){
            if(arr[i] > arr[i+1]){
                int temp = arr[i];
                arr[i] = arr[i+1];
                arr[i+1] = temp;
                exchFlag = 1;
            }
        }
        if(start == 0){
            start = 1;
        }else {
            start = 0;
        }
    }
}
```

## 1.2 奇偶交换排序的并行实现

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        pOddEvenSort(arr);
        for(int i = 0; i < arr.length; i++){
            System.out.println(arr[i]);
        }
        pool.shutdown();
    }
    static int [] arr = new int[] {2,3,6,6,8,9,6,2};
    static ExecutorService pool = Executors.newCachedThreadPool();

    static int exchFlag = 1;
    static synchronized void setExchFlag(int v){
        exchFlag = v;
    }
    static synchronized int getExchFlag(){
        return exchFlag;
    }
    public static class OddEvenSortTask implements Runnable{
        int i;
        CountDownLatch countDownLatch;

        public OddEvenSortTask(int i, CountDownLatch countDownLatch) {
            this.i = i;
            this.countDownLatch = countDownLatch;
        }
        @Override
        public void run() {
            if(arr[i] > arr[i+1]){
                int temp = arr[i];
                arr[i] = arr[i+1];
                arr[i+1] = temp;
                setExchFlag(1);
            }
            countDownLatch.countDown();
        }
    }
    public static void pOddEvenSort(int [] arr) throws InterruptedException {
        int start = 0;
        while (getExchFlag() == 1 || start == 1){
            setExchFlag(0);
            int num = arr.length/2 - (arr.length % 2 == 0?start:0);
            CountDownLatch countDownLatch = new CountDownLatch(num);
            for(int i = start; i < arr.length - 1; i = i + 2){
                pool.submit(new OddEvenSortTask(i, countDownLatch));
            }
            countDownLatch.await();
            if(start == 0){
                start = 1;
            }else {
                start = 0;
            }
        }
    }
}
```

# 2. 希尔排序

简单的插入排序很难并行化,因为这一次的数据插入依赖于上一次得到的有序序列,因此多个步骤之间无法并行.为此,我们可以对插入排序进行扩展,这就是希尔排序.

希尔排序将整个数组根据建个h分割为若干个子数组.子数组相互穿插在一起,在每一次排序时,分别对每一个子数组进行排序.当h为3时,希尔排序将整个数组分为交织在一起的三个子数组,其中,所有的方块为一个子数组,所有的圆形,三角形分别组成另外两个子数组,每次排序时,总算交换间隔为h的两个元素.

![希尔排序](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20190312190003.png)

## 2.1 串行实现

```java
public class Solution {
    public void shellSort(int[] array) {
        int d = array.length;
        while (d > 0) {
            d = d / 2;
            for (int i = d; i < array.length; i++) {
                int number = array[i];
                int j = i - d;
                while (j >= 0 && array[j] > number) {
                    array[j + d] = array[j];
                    j -= d;
                }
                array[j + d] = number;
            }
        }
    }
}
```

## 2.2 并行实现

```java
public class Solution {
    static ExecutorService es = Executors.newCachedThreadPool();
    static int[] array = {1, 3, 4, 5, 2};

    public static class shellSort implements Runnable {
        static int d;
        static int start;
        static CountDownLatch latch;

        public shellSort(int d, int start, CountDownLatch latch) {
            this.d = d;
            this.start = start;
            this.latch = latch;
        }

        @Override
        public void run() {
            for (int i = start; i < array.length; i++) {
                int number = array[i];
                int j = i - d;
                while (j >= 0 && array[j] > number) {
                    array[j + d] = array[j];
                    j -= d;
                }
                array[j + d] = number;
                latch.countDown();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException{
        int d = array.length;
        while (d > 0) {
            d /= 2;
            CountDownLatch latch = new CountDownLatch(d);
            for (int i = 0; i < d; i++) {
                es.submit(new shellSort(d, i, latch));
            }
            latch.await();
        }
    }
}
```

# 3. 参考文献

<< Java高并发程序设计 >>(葛一鸣 郭超)

[插入排序及希尔排序的并行化实现](https://www.jianshu.com/p/f476a305a5d5)
