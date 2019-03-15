---
title: Java高并发程序设计(八)--并行流与并行排序
date: 2019-3-14 09:34:16
categories: 高并发程序设计
tags: [Java, 并发设计, 基础储备]
---

----

<!-- more -->

# 1. 使用并行流过滤数据

统计1~1000000内所有质数的数量.

```java
public static boolean isPrime(int number){
    if(number < 2){
        return false;
    }
    for(int i = 2; Math.sqrt(number) > i; i++){
        if(number % i == 0){
            return false;
        }
    }
    return true;
}
```

```java
// 串行做法
long val = IntStream.range(0, 1000000).filter(x -> isPrime(x)).count();
System.out.println(val);
// 并行做法
long val2 = IntStream.range(0, 1000000).parallel().filter(x -> isPrime(x)).count();
System.out.println(val2);
```

# 2. 从集合得到并行流

统计集合内所有学生的平均分

```java
List<Student> students = new ArrayList<>();
students.add(new Student("张三", 100));
students.add(new Student("李四", 98));
students.add(new Student("王五", 99));

// 串行做法
double ave = students.stream().mapToInt(x -> x.score).average().getAsDouble();
System.out.println(ave);
// 并行做法
double ave2 = students.parallelStream().mapToInt(x -> x.score).average().getAsDouble();
System.out.println(ave2);
```

# 3. 并行排序

对数组进行并行排序.

```java
Arrays.parallelSort(arr);
```

数组中的数据赋值

```java
int [] arr = new int[1000000];
Random random = new Random();

// 串行做法
Arrays.setAll(arr, (x) -> random.nextInt());
System.out.println(Arrays.toString(arr));
// 并行做法
Arrays.parallelSetAll(arr, (x) -> random.nextInt());
System.out.println(Arrays.toString(arr));
```

# 4. 参考文献

<< Java高并发程序设计 >>(葛一鸣 郭超)