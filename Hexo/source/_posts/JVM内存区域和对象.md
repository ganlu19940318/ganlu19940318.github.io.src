---
title: JVM内存区域和对象
date: 2018-09-24 09:15:14
categories: Java基础
tags: [Java, 基础储备, JVM]
---

----

<!-- more -->

# 1. 前言

作为一名Java后台开发的程序员, 深入理解JVM, 重要性不言而喻, 这篇文章主要是记录JVM内存区域相关知识.

# 2. 运行时数据区

Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域, 如下图所示:
![Java内存区域](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/1056036-20161103200608783-1135813981.png)

## 2.1 程序计数器

当前线程所执行的字节码的行号指示器. 如果线程正在执行的是一个Java方法, 这个计数器记录的是正在执行的虚拟机字节码指令的地址; 如果正在执行的是Native方法, 这个计数器值则为空(Undefined).

### 2.1.1 程序计数器的作用

1. 生活中的案例
比如老王正在看电影, 他看到三十五分钟的时候, 突然他的QQ好友给他开视频聊天, 这时候肯定打断他看电影了, 假设他qq好友和他视频完了, 他肯定要接着他那35分钟的进度去继续看, 这时候他怎么知道我看到35分钟了? 这时候程序计数器就起了作用, 他负责管理进度.

2. 代码层面的案例
A线程正在执行HelloWorld.class的第三十五行. 这时候CPU时间片被B线程抢走了, 当A线程重新被分配到时间片时, 他怎么知道我的class运行到哪了? 这时候他可以看程序计数器在哪个位置.

## 2.2 Java虚拟机栈

描述Java方法执行的内存模型: 每个方法被执行的时候都会同时创建一个栈帧, 用于存储局部变量表, 操作数栈, 动态链接, 方法出口等信息. 每一个方法从调用直至执行完成的过程, 就对应着一个栈帧在虚拟机中入栈到出栈的过程.

### 2.2.1 局部变量表

存放了编译期可知的各种基本数据类型(boolean, byte, char, short, int, float, long, double), 对象引用(不等同于对象本身), returnAddress类型(指向了一条字节码指令的地址).
long 和 double 类型的数据会占用2个局部变量空间, 其余的数据类型只占用1个.
局部变量表所需的内存空间在编译期间完成分配.

### 2.2.2 两种异常状况

1. 如果线程请求的栈深度大于虚拟机所允许的深度, 将抛出StackOverflowError异常;
2. 如果虚拟机栈可以动态扩展, 并且扩展时无法申请到足够的内存, 就会抛出OutOfMemoryError异常.

## 2.3 本地方法栈

与虚拟机栈所发挥的作用是非常相似的, Java虚拟机栈为虚拟机执行Java方法服务, 而本地方法栈为虚拟机使用的Native方法服务.

## 2.4 Java堆

Java堆是被所有线程共享的一块内存区域, 在虚拟机启动时创建. 所有的<font color=red>对象实例</font>以及<font color=red>数组</font>都要在堆上分配. 这里是垃圾收集器管理的主要区域.
Java堆可以处于物理上不连续的内存空间中.

### 2.4.1 异常

如果在堆中没有内存完成实例分配, 并且堆也无法再扩展时, 将会抛出OutOfMemoryError异常.

## 2.5 方法区

线程共享的内存区域, 存储已被虚拟机加载的类信息, 常量, 静态变量, 即时编译器编译后的代码数据等(这个区域的内存回收目标主要是针对<font color=red>常量池的回收</font>和对<font color=red>类型的卸载</font>). 

### 2.5.1 异常

当方法区无法满足内存分配需求时, 将会抛出OutOfMemoryError异常.

### 2.5.2 运行时常量池

在方法区中有一个非常重要的部分就是运行时常量池(<font color=red>JDK7之后, 已经挪到堆区里面了</font>), 它是每一个类或接口的常量池的运行时表示形式, <font color=red>在类和接口被加载到JVM后, 对应的运行时常量池就被创建出来</font>. 当然并非Class文件常量池中的内容才能进入运行时常量池, 在运行期间也可将新的常量放入运行时常量池中, 比如String的intern方法.
当运行时常量池无法申请到内存时, 将会抛出OutOfMemoryError异常.

## 2.6 直接内存

<font color=red>直接内存不是虚拟机运行时数据区的一部分</font>, 也不是java虚拟机规范中定义的内存区域. 但是这部分内存也被频繁使用, 可能抛出OutOfMemoryError异常. NIO类引入了一种基于通道与缓冲区(Buffer)的I/O方式, 它可以使用Native函数库直接分配堆外内存, 然后通过一个存储在java堆中的DirectByteBuffer对象作为这块内存的引用进行操作, 这样能在一些场景中显著提高性能, 因此避免了在java堆和Native堆中来回复制数据. 服务器管理员容易忽略直接内存, 使得内存区域总和大于物理内存限制, 从而导致动态抛出OutOfMemoryError异常.

# 3. 对象

## 3.1 创建过程

下面我们详细了解Java程序中new一个普通对象时, 虚拟机是怎么样创建这个对象的, 包括5个步骤: 相应类加载检查过程、在Java堆中为对象分配内存、分配后内存初始化为零、对对象进行必要的设置、以及执行对象实例方法< init >.

### 3.1.1 相应类加载检查

JVM遇到new指令时, 先检查指令参数是否能在常量池中定位到一个类的符号引用:

1. 如果能定位到，检查这个符号引用代表的类是否已被加载、解析和初始化过；
2. 如果不能定位到，或没有检查到，就先执行相应的类加载过程；

### 3.1.2 为对象分配内存

对象所需内存的大小在类加载完成后便完全确定(JVM可以通过普通Java对象的类元数据信息确定对象大小)；
为对象分配内存相当于把一块确定大小的内存从Java堆里划分出来；
<font color=red>(A). 分配方式</font>
<font color=blue>(1). 指针碰撞</font>
如果Java堆是绝对规整的: 一边是用过的内存. 一边是空闲的内存. 中间一个指针作为边界指示器;
分配内存只需向空闲那边移动指针, 这种分配方式称为<font color=orange>"指针碰撞"(Bump the Pointer)</font>;
<font color=blue>(2). 空闲列表</font>
如果Java堆不是规整的: 用过的和空闲的内存相互交错;
需要维护一个列表, 记录哪些内存可用;
分配内存时查表找到一个足够大的内存, 并更新列表, 这种分配方式称为<font color=orange>"空闲列表"(Free List)</font>;

----

Java堆是否规整由JVM采用的垃圾收集器是否带有压缩功能决定的；
所以，使用Serial、ParNew等带Compact过程的收集器时，JVM采用指针碰撞方式分配内存；而使用CMS这种基于标记-清除（Mark-Sweep）算法的收集器时，采用空闲列表方式；

<font color=red>(B). 线程安全问题</font>
并发时, 上面两种方式分配内存的操作都不是线程安全的, 有两种解决方案:
<font color=blue>(1). 同步处理</font>
对分配内存的动作进行同步处理:
JVM采用<font color=orange>CAS(Compare and Swap)</font>机制加上失败重试的方式, 保证更新操作的原子性;
CAS: 有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做；
<font color=blue>(2). 本地线程分配缓冲区</font>
把分配内存的动作按照线程划分在不同的空间中进行:
在每个线程在Java堆预先分配一小块内存, 称为本地线程分配缓冲区<font color=orange>(Thread Local Allocation Buffer,TLAB)</font>;
哪个线程需要分配内存就从哪个线程的TLAB上分配；
只有TLAB用完需要分配新的TLAB时, 才需要同步处理；
JVM通过"-XX：+/-UseTLAB"指定是否使用TLAB；

### 3.1.3 分配后内存初始化为零

内存分配完成后, 虚拟机需要将分配到的内存空间都初始化为零值(不包括对象头), 如果使用TLAB, 提前至分配TLAB时;
这保证了程序中对象(及实例变量)不显式初始赋零值, 程序也能访问到零值.

### 3.1.4 对对象进行必要的设置

主要设置对象头信息，包括类元数据引用、对象的哈希码、对象的GC分代年龄等;

### 3.1.5 执行对象实例方法< init >

该方法把对象(实例变量)按照程序中定义的初始赋值进行初始化;

## 3.2 内存布局

下面我们详细了解Java普通对象创建后, 在虚拟机Java堆中的内存布局是怎样的, 可以分为3个区域: 对象头(Header), 实例数据(Instance)和对齐填充(Padding).

### 3.2.1 对象头

可以主要分为两部分:
<font color=red>(A). 存储对象自身运行时数据</font>
称为"Mark Word", 包括哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等；
这部分长度为32bit（32位JVM）或64bit（64位JVM）；
被设计成一个非固定的数据结构，会根据对象的状态复用自己的存储空间，以便在极小的空间内存储尽量多信息；
例如，32bit的Mark Word在未被锁定状态下，前25bit存储对象哈希码，4bit存储对象分代年龄，2bit存储锁标志位，1bit固定为0，如图所示
![对象头](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20161229205402094.png)
<font color=red>(B). 存储指向对象类型数据的指针</font>
通过这个指针确定这个对象是哪个类的实例;
不是必须的, 看对象的访问定位方式:
对HotSpot虚拟机来说, 由于JVM栈本地变量表中对象的reference类型引用使用直接指针, 该指针指向堆内存中的对象, 所以对象头中是需要存储它的类元数据指针, 该指针指向方法区中对象类型数据.
<font color=red>(C). 如果是Java数组, 对象头还需要存储数组长度</font>
因为数组对象类型数据中没有数组长度信息;
而JVM可以通过普通Java对象的类元数据信息确定对象大小;

### 3.2.2 实例数据

它是对象真正存储的有效信息, 程序代码所定义的各种类型字段内容, 以及包括父类继承或子类定义的;
存储顺序:
受到JVM分配策略参数(FiedAllocationStyle)和字段在Java源码中定义顺序影响;
JVM默认分配策略为: longs/doubles. ints. shorts/chars. booleans. oops(Ordiary Object Pointers);
JVM默认分配策略使得, 相同宽度的字段总被分配到一起;
这个前提下, 父类定义的变量出现在子类之前;
如果虚拟机的"CompactFields"参数为true, 子类中较窄的变量可能插入到父类变量空隙中, 以压缩节省空间;

### 3.2.3 对齐填充

不是必然存在的;
只起占位符作用, 没有其他含义;
HotSpot虚拟机要求对象大小必须是8字节的整数倍;
对象头是8字节整数倍, 所以填充是对实例数据没有对齐的情况来说的.

## 3.3 访问定位

下面我们详细了解在Java堆中的Java对象是如何访问定位的: 先来了解reference类型数据是什么, 再来了解两种访问方式: 使用句柄或者使用直接指针(HotSpot虚拟机使用了直接指针的方式访问对象).

### 3.3.1 reference

Java程序通过reference类型数据操作堆上的具体对象;
reference类型是引用类型(Reference Types)的一种;
JVM规范规定reference类型来表示对某个对象的引用, 可以想象成类似于一个指向对象的指针;
对象的操作、传递和检查都通过引用它的reference类型的数据进行操作;

### 3.3.2 对象访问方式

 虽然定义的reference类型数据来作为对象内存数据的引用, 但JVM规范没有定义这个引用应该通过何种方式定位, 访问堆上的对象, 也没有不强制规定对象的内部结构应当如何表示;  
这些都取决于JVM的实现, 目前主流的对象访问方式有两种:句柄访问 和 直接指针访问

#### 3.3.2.1 句柄访问

Java堆划分一块内存作为句柄池, reference中存储就是对象的句柄地址;
对象句柄包含两个地址:

1. 在堆中分配的对象实例数据的地址;
2. 这个对象类型数据地址;  

如图所示
![使用句柄 ](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20161229210448955.png)
优点: 对象移动时(垃圾回收时常见的动作), reference不需要修改, 只改变句柄中实例数据指针;

#### 3.3.2.2 直接指针访问

reference中存储就是在堆中分配的对象实例数据的地址;
而对象实例数据中需要有这个对象类型数据的相关信息(前面章节讨论了HotSpot使用对象头来存储对象类型数据地址);
如图所示
![直接指针访问](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20161229210449562.png)
 优点: 对象访问时节省了一次指针定位的时间开销, 速度更快;
由于对象访问非常频繁进行, 所以能较好提升性能;
HotSpot虚拟机使用了直接指针的方式访问对象;

# 4. 参考链接

<<深入理解Java虚拟机----JVM高级特性与最佳实践>>(第二版, 周志明)
[JVM程序计数器](https://www.cnblogs.com/thiaoqueen/p/8455521.html)
[Java中的常量池(字符串常量池、class常量池和运行时常量池)](https://blog.csdn.net/zm13007310400/article/details/77534349)
[Java对象与JVM（一） Java对象在Java虚拟机中的创建过程](https://blog.csdn.net/tjiyu/article/details/53923392)
[Java对象与JVM（二） Java对象在Java虚拟机中的内存布局](https://blog.csdn.net/tjiyu/article/details/53932122)
[Java对象与JVM（三） Java对象在Java虚拟机中的引用访问方式](https://blog.csdn.net/tjiyu/article/details/53932199)