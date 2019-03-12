---
title: Java高并发程序设计(六)--并行算法之并行搜索
date: 2019-3-12 11:21:22
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 并行搜索

并行的无序数组的搜索实现.

给定一个数组,我们要查找满足条件的元素.

# 2. 思考

对于串行程序来说,只要遍历一下数组就可以得到结果.但是如果要使用并行方式,则需要额外增加一些线程间的通信机制,使各个线程可以有效运行.

一种简单的策略就是将原始数据集合按照期望的线程数进行分割.如果我们计划使用两个线程进行搜索,那么就可以把一个数组或集合分割成两个.每个线程各自独立搜索,当其中有一个线程找到数据后,立即返回结果即可.

# 3. 示例

SearchTask

```java
import java.util.concurrent.Callable;
public class SearchTask implements Callable<Integer> {
    int begin, end, searchValue;
    public SearchTask(int begin, int end, int searchValue) {
        this.begin = begin;
        this.end = end;
        this.searchValue = searchValue;
    }
    @Override
    public Integer call(){
        int re = Main.search(begin, end, searchValue);
        return re;
    }
}
```

Main

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.atomic.AtomicInteger;
public class Main {
    public static int [] arr = new int[]{1, 2, 3};
    public static AtomicInteger result = new AtomicInteger(-1);
    public static final int Thread_Num = 2;
    public static ExecutorService pool = Executors.newCachedThreadPool();
    public static int search(int begin, int end, int searchValue){
        for(int i = begin; i < end; i++){
            if(result.get() >= 0){
                return result.get();
            }
            if(arr[i] == searchValue){
                if(!result.compareAndSet(-1, i)){
                    return result.get();
                }
                return i;
            }
        }
        return -1;
    }
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        int val = 3;
        int subArrSize = arr.length / Thread_Num + 1;
        List<Future<Integer>> re = new ArrayList<>();
        for(int i = 0 ; i < arr.length; i += subArrSize){
            int end = i + subArrSize;
            if(end > arr.length){
                end = arr.length;
            }
            re.add(pool.submit(new SearchTask(i, end, val)));
        }
        for(Future<Integer> fu : re){
            if(fu.get() >= 0){
                System.out.println(fu.get());
            }
        }
    }
}
```

# 4. 参考文献

<< Java高并发程序设计 >>(葛一鸣 郭超)
