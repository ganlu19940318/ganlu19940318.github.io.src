---
title: Redis设计与实现--数据结构与对象
date: 2019-3-15 09:49:52
categories: Redis
tags: [Redis, 基础储备]
---

----

<!-- more -->

# 1. 前言

Redis数据库里面的每个键值对都是由对象组成的.

其中:
数据库键总是一个字符串对象.
而数据库键的值则可以是字符串对象,列表对象,哈希对象,集合对象,有序集合对象,这五种对象中的一种.

每种对象底层又使用着不同的数据结构.

# 2. 数据结构

## 2.1 简单动态字符串

简单动态字符串(SDS)是Redis的默认字符串表示.

### 2.1.1 SDS定义

Redis没有使用C语言的字符串,C语言的字符串只会用在不需要对字符串修改而只使用其值地方.Redis使用SDS表示字符串,结构定义

```c
struct sdshdr {
    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;
    // 记录 buf 数组中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
};
```

SDS也是以'\0'表示结束,这一个字节不会计入已使用的长度.这样做的好处是可以重用C字符串函数库里面的一部分函数.

### 2.1.2 SSD和C字符串的区别

C语言使用长度为N+1的数组表示长度为N的字符串,最后一个元素总是空字符'\0',这种方式不能满足Redis对字符串效率,安全性,以及功能方面的要求.下面是SDS比C语言字符串的更加适用Redis的原因.

1. 常数时间获取字符串长度
C字符串需要遍历,时间复杂度为O(n).
SDS直接获取,时间复杂度为O(1).

2. 防止缓冲区溢出
C语言不记录自身长度,容易误操作造成缓冲区溢出.SDS的空间分配策略完全杜绝了这种可能性.当API需要对SDS进行修改时,API会首先会检查SDS的空间是否满足条件,如果不满足,API会自动对它动态扩展,然后再进行修改,这个过程是完全透明的.

3. 减少修改字符串带来的内存重分配次数
C语言对字符串修改后都需要手动重新分配内存;当增加长度时需要扩展内存,否则会产生缓冲区溢出;当缩小长度时需要释放内存,否则会产生内存泄露.
由于Redis频繁操作数据,内存分配和释放耗时可能对性能造成影响,SSD避免了这种缺陷,实现空间预分配和惰性空间释放两种优化策略.
**空间预分配**
如果修改后len长度将小于1M,这时分配给free的大小和len一样,例如修改过后为13字节,那么给free也是13字节.buf实际长度变成了13 byte+ 13 byte + 1 byte = 27byte.如果修改后len长度将大于等于1 M,这时分配给free的长度为1M,例如修改过后为30M,那么给free是1M.buf实际长度变成了30M + 1M + 1 byte. 在修改时,首先检查空间是不是够,如果足够,直接使用, 否则执行内存重分配.
**惰性空间释放**
当缩短SDS长度时,不进行内存释放,而是记录到free字段中,等待下次使用.与此同时,也提供相应的API,可以手动释放内存.
4. 二进制安全
C字符串只有末尾能保存空格,中间如果有空格会被截取,认作结束标识.这样就不能保存图片,音频视频等二进制数据了.所有的SDS API会以二进制的方式处理SDS buf数组里面的数据,程序不会对其中数据做任何限制,过滤,修改和假设,数据写入是什么样子,读取出来就是什么样子.例如,保留的数据中间出现'\0',这是没有任何问题的,因为它使用len而不是空字符判断结束.
5. 兼容部分C字符串函数

## 2.2 链表

Redis实现为双链表结构,列表键的底层实现之一就是链表,发布与订阅,慢查询,监视器等功能都用到了链表.Redis本身也使用链表维持多个客户端.

### 2.2.1 链表定义

```c
// 节点定义
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
```

```c
// 链表结构定义
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;
} list;
```

### 2.2.2 链表特性总结

1. 双端链表,获取前驱和后置都为O(1);
2. 无环,表头的前驱和表尾的后置都为NULL;
3. 带头指针和尾指针;
4. 带链表长度计数器;
5. 多态,void * 保存点值,通过list结构的dup,free,match三个属性设置节点特定函数,所以可以用链表保存不同类型的值.

## 2.3 字典

### 2.3.1 字典实现

```c
/*
* 哈希表
* 每个字典都使用两个哈希表，从而实现渐进式 rehash 。
*/
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

```c
/*哈希表节点*/
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

```c
/*
 * 字典
 */
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */
} dict;
```

### 2.3.2 哈希算法

使用MurmurHash2算法计算哈希值.

### 2.3.3 决键冲突

链地址法,采用头插法,时间复杂度为O(1).

### 2.3.4 rehash

rehash步骤:

1. 为ht[1]分配空间:如果是扩容,分配大小为第一个大于等于 ht[0].used*2的2的n次幂;如果是收缩,分配空间大小为第一个大于等于 ht[0].used的2的n次幂.
2. 将ht[0]上的键值对通过重新计算键的哈希值和索引重新分配到ht[1]上.
3. 这时ht[0]将变为空表,释放ht[0],将ht[1]设置为ht[0],并在ht[1]创建一个空表,为下次rehash做准备.

### 2.3.5 渐进式rehash

渐进式rehash步骤:

1. 在字典维持一个索引计数器rehashidx,设置为0, 表示rehash工作正式开始.
2. 在对字典执行添加,删除,查找或修改时,将rehashidx索引上的所有键值对rehash到hd[1],完成后rehashidx++;
3. 当ht[0] 上的所有键值对被rehash到 ht[1],设置 rehashidx = - 1, 表示操作已经完成了.

在此期间操作数据, 会使用ht[0] 和 ht[1]两个hash表,  例如要找某个键值, 会先在ht[0] 找, 没有就去ht[1]找.新增操作都保存到ht[1]中.保证了ht[0]只减不增,最终为空.

## 2.4 跳跃表

### 2.4.1 跳跃表节点实现

```c
// 跳跃表节点
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    robj *obj;  /*成员对象*/
    double score;   /*分值*/
    struct zskiplistNode *backward; /*后退指针*/
    struct zskiplistLevel { /*层*/
        struct zskiplistNode *forward;  /*前进指针*/
        unsigned int span;  /*跨度*/
    } level[];
} zskiplistNode;
```

### 2.4.2 跳跃表节点的几个概念

1. 分值和成员
节点的分值(score属性)是一个double类型的浮点数,跳跃表中的所有节点都按分值从小到大来排序.
节点的成员对象(obj属性)是一个指针,它指向一个字符串对象,而字符串对象则保存着一个SDS值.
2. 后退指针
节点的后退指针(backward属性)用于从表尾向表头方向访问节点:跟可以一次跳过多个节点的前进指针不同,因为每个节点只有一个后退指针,所以每次只能后退至前一个节点.
3. 层
跳跃表节点的level数组可以包含多个元素,每个元素都包含一个指向其他节点的指针,程序可以通过这些层来加快访问其他节点的速度,一般来说,层的数量越多,访问其他节点的速度就越快.
4. 前进指针
每个层都有一个指向表尾方向的前进指针(level[i].forward属性),用于从表头向表尾方向访问节点.
5. 跨度
层的跨度(level[i].span属性)用于记录两个节点之间的距离.

### 2.4.3 跳跃表

```c
// 跳跃表
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;    //header指向跳跃表的表头节点，tail指向跳跃表的表尾节点
    unsigned long length;   //记录跳跃表的长度，也即是，跳跃表目前包含节点的数量(表头节点不计算在内)
    int level;  //记录目前跳跃表内，层数最大的那个节点的层数(表头节点的层数不计算在内)
} zskiplist;
```

仅靠多个跳跃表节点就可以组成一个跳跃表,但通过使用一个zskiplist结构来持有这些节点,程序可以更方便地对整个跳跃表进行处理,比如快速访问跳跃表的表头节点和表尾节点,或者快速地获取跳跃表节点的数量(也即是跳跃表的长度)等信息.

## 2.5 整数集合

整数集合的底层实现是数组,这个数组以有序,无重复的方式保存集合元素.

### 2.5.1 实现

```c
typedef struct intset {
    // 保存元素所使用类型的长度
    uint32_t encoding;
    // 元素个数
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];  //数组的内存与结构体的内存是相邻的
} intset;
```

### 2.5.2 升级

当一个要加到整数集合的新元素,当前集合的contents数组的元素类型不够长来容纳,则会升级,例如从int8_t升级到int16_t;
升级的过程需要维持数组的有序性,会逐个元素移动.
因为向intset添加元素有可能引起升级,所以添加元素的时间复杂度是O(N).
升级机制提升了灵活性,节约了存储空间.
不支持降级操作.

## 2.6 压缩列表

压缩列表是一种数据结构,这种数据结构的功能是将一系列数据与其编码信息存储在一块连续的内存区域,这块内存物理上是连续的,逻辑上被分为多个组成部分,其目的是在一定可控的时间复杂读条件下尽可能的减少不必要的内存开销,从而达到节省内存的效果.

### 2.6.1 压缩列表构成

上面说到压缩列表是一块连续的内存区域,这块内存区域布编码示意图大致如下:

![Redis压缩列表内存编码示意图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/214654_bjC6_1759553.png)

zlbytes:存储一个无符号整数,固定四个字节长度,用于存储压缩列表所占用的字节,当重新分配内存的时候使用,不需要遍历整个列表来计算内存大小.

zltail:存储一个无符号整数,固定四个字节长度,代表指向列表尾部的偏移量,偏移量是指压缩列表的起始位置到指定列表节点的起始位置的距离.

zllen:压缩列表包含的节点个数,固定两个字节长度,源码中指出当节点个数大于2^16-2个数的时候,该值将无效,此时需要遍历列表来计算列表节点的个数.

entryX:列表节点区域,长度不定,由列表节点紧挨着组成.

zlend:固定一字节长度, 固定值为255, 用于表示列表结束.

### 2.6.2 压缩列表节点构成

上面介绍了压缩列表的总体内存布局,下面再看看entry区域的编码情况.

每个列表节点由三部分组成:

![压缩列表节点编码示意图](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/220002_oKEX_1759553.png)

1. previous length
用于存储上一个节点的长度,因此压缩列表可以从尾部向头部遍历,即当前节点位置减去上一个节点的长度即得到上一个节点的起始位置.previous length的长度可能是1个字节或者是5个字节,如果上一个节点的长度小于254,则该节点只需要一个字节就可以表示前一个节点的长度了,如果前一个节点的长度大于等于254,则previous length的第一个字节为254,后面用四个字节表示当前节点前一个节点的长度.这么做很有效地减少了内存的浪费.

2. encoding
节点的encoding保存的是节点的content的内容类型以及长度,encoding类型一共有两种,一种字节数组一种是整数,encoding区域长度为1字节,2字节或者5字节长.Redis作者巧妙的利用了前两个字节来表示content存储的内容类型和encoding区域的长度.
我们先看看字节数组类型的encoding内容:
![content为字节数组的encoding内容](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/222413_VDcG_1759553.png)
再看看整数编码类型的encoding内容:
![content为整数的encoding内容](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/223313_rZcZ_1759553.png)

3. content
content区域用于保存节点的内容,节点内容类型和长度由encoding决定,上面可以看出目前content的内容类型有整数类型和字节数组类型,且某些条件下content的长度可能为0.

压缩列表并不是对数据利用某种算法进行压缩,而是将数据按照一定规则编码在一块连续的内存区域,目的是节省内存.

### 2.6.3 连锁更新

添加新节点到压缩列表,或者从压缩列表中删除节点,可能会引发连锁更新操作,但这种操作出现的几率并不高.

连锁更新产生的原因是插入节点后,该节点后面那个节点的previous length的长度可能会变化(比如从1字节变成5字节,又或者从5字节变成1字节),而previous length变化后,使这个节点的长度发生了变化,并且也是临界变化(长度从小于254变成大于等于254,或者长度从大于等于254变成小于254),类似地,从而引起连锁变化.

# 3. 对象

对象系统包括字符串对象,列表对象,哈希对象,集合对象,有序集合对象.

## 3.1 对象的类型和编码

```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    //...
} robj;
```

### 3.1.1 类型

对象的type属性记录了对象的类型.比如REDIS_STRING(字符串对象),REDIS_LIST(列表对象)等.

### 3.1.2 编码和底层实现

对象的prt指针指向对象的底层实现数据结构,而这些数据结构由对象的encoding属性决定.encoding属性记录了对象所使用的编码,也就是说对象使用什么数据结构作为对象的底层实现.

![对象的编码](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/TIM%E6%88%AA%E5%9B%BE20190315192216.png)

## 3.2 各对象及其对应的编码

1. 字符串对象
编码可以是int,raw或者embstr.
2. 列表对象
编码可以是ziplist或者linkedlist.
3. 哈希对象
编码可以是ziplist或者hashtable.
4. 集合对象
编码可以是intset或者hashtable.
5. 有序集合对象
编码可以是ziplist或者skiplist.

由于每种对象对应着多种不同的编码方式,所以,在不同条件下,会存在编码转换.

## 3.3 内存回收

C语言不具备自动内存回收的功能.Redis使用引用计数技术实现内存回收机制.

## 3.4 对象共享

为了节约内存.但是只对包含整数值的字符串对象进行共享.因为一个共享对象保存的值越复杂,验证共享对象和目标对象是否相同所需的复杂度就会越高,消耗的CPU时间也会越多.(其实这里类似于Java里面的常量池)

## 3.5 对象空转时长

redisObject其实还有一个属性叫做lru,用来记录对象最后一次被命令程序访问的时间.

通过这个属性可以计算对象的空转时长,当服务器占用的内存数超过了maxmemory设置的上限值时,空转时长较高的那部分会优先被服务器释放,从而回收内存.

# 4. 参考文献

<< Redis设计与实现 >> (黄健宏 著)
[Redis设计与实现 (一): 简单动态字符串](https://www.cnblogs.com/tanxing/p/6688406.html)
[Redis设计与实现 (二): 链表](https://www.cnblogs.com/tanxing/p/6711421.html)
[Redis设计与实现 (三): 字典](https://www.cnblogs.com/tanxing/p/6715002.html)
[redis跳跃表](https://blog.csdn.net/weixin_39138071/article/details/80306970)
[底层实现-intset 整数集合](https://blog.csdn.net/whiteoldbig/article/details/51541519)
[Redis压缩列表原理与应用分析](https://my.oschina.net/andylucc/blog/715325)
[我知道点redis-数据结构与对象(对象)-对象存储](https://www.cnblogs.com/wangxin201492/p/4754110.html)