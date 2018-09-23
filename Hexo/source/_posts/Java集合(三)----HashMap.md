---
title: Java集合(三)----HashMap
date: 2018-09-22 22:22:19
categories: Java基础
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 前言

集合是Java基础中非常重要的一部分, 也是日常中用到非常多的类, 这篇博客主要记录集合HashMap相关的知识.

# 2. HashMap

## 2.1 概述
HashMap 是用于映射(键值对)处理的数据类型, 基于哈希表的 Map 接口的非同步实现, 允许插入最多一条key为null的记录, 允许插入多条value为null的记录. 此外, HashMap 不保证元素顺序, 根据需要该容器可能会对元素重新哈希, 元素的顺序也会被重新打散, 因此在不同时间段迭代同一个 HashMap 的顺序可能会不同. HashMap 非线程安全, 即任一时刻有多个线程同时写 HashMap 的话可能会导致数据的不一致.

<font color=red>HashMap 实际上是数组+链表+红黑树的结合体, 其底层包含一个数组, 数组中的每一项元素的可能值有四种: null, 单独一个结点, 链表, 红黑树(JDK1.8 开始 HashMap 通过使用红黑树来提高元素查找效率).</font>

当往 HashMap 中 put 元素的时候, 需要先根据 key 的哈希值得到该元素在数组中的位置(即下标), 如果该位置上已经存放有其他元素了, 那么在这个位置上的元素将以链表或者红黑树的形式来存放, 如果该位置上没有元素, 就直接向该位置存放元素.
HashMap 要求映射中的 key 是不可变对象，即要求该对象在创建后它的哈希值不会被改变，否则 Map 对象很可能就定位不到映射的位置了.

## 2.2 类声明






```Java
public class HashMap<K, V> extends AbstractMap<K, V>
        implements Map<K, V>, Cloneable, Serializable
```

## 2.3 常量

HashMap 中声明的常量有以下几个, 其中需要特别关注的是装载因子 DEFAULT_LOAD_FACTOR 和 TREEIFY_THRESHOLD.

装载因子用于规定数组在自动扩容之前可以数据占有其容量的最高比例, 即当数据量占有数组的容量达到这个比例后, 数组将自动扩容. 装载因子衡量的是一个散列表的空间的使用程度, 装载因子越大表示散列表的装填程度越高, 反之愈小. 因此如果装载因子越大, 则对空间的利用程度更高, 相对应的是查找效率的降低. 如果装载因子太小, 那么数组的数据将过于稀疏, 对空间的利用率低, 官方默认的装载因子为0.75, 是平衡空间利用率和运行效率两者之后的结果. 如果在实际情况中, 内存空间较多而对时间效率要求很高, 可以选择降低装载因子的值; 如果内存空间紧张而对时间效率要求不高, 则可以选择提高装载因子的值.

此外, 即使装载因子和哈希算法设计得再合理, 也不免会出现由于哈希冲突导致链表长度过长的情况, 这将严重影响 HashMap 的性能. 为了优化性能, 从 JDK1.8 开始引入了红黑树, 当链表长度超出 TREEIFY_THRESHOLD 规定的值时, 链表就会被转换为红黑树, 利用红黑树快速增删改查的特点以提高 HashMap 的性能.
```Java
    //序列化ID
    private static final long serialVersionUID = 362498820763181265L;

    //哈希桶数组的默认容量
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

    //网上很多文章都说这个值是哈希桶数组能够达到的最大容量，其实这样说并不准确
    //从 resize() 方法的扩容机制可以看出来，HashMap 每次扩容都是将数组的现有容量增大一倍
    //如果现有容量已大于或等于 MAXIMUM_CAPACITY ，则不允许再次扩容
    //否则即使此次扩容会导致容量超出 MAXIMUM_CAPACITY ，那也是允许的
    static final int MAXIMUM_CAPACITY = 1 << 30;

    //装载因子的默认值
    //装载因子用于规定数组在自动扩容之前可以数据占有其容量的最高比例，即当数据量占有数组的容量达到这个比例后，数组将自动扩容
    //装载因子衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，反之愈小
    //对于使用链表的散列表来说，查找一个元素的平均时间是O(1+a)，因此如果负载因子越大，则对空间的利用程度更高，相对应的是查找效率的降低
    //如果负载因子太小，那么数组的数据将过于稀疏，对空间的利用率低
    //官方默认的负载因子为0.75，是平衡空间利用率和运行效率两者之后的结果
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    //为了提高效率，当链表的长度超出这个值时，就将链表转换为红黑树
    static final int TREEIFY_THRESHOLD = 8;
```

## 2.4 成员变量

```Java
    //哈希桶数组，在第一次使用时才初始化
    //容量值应是2的整数倍
    transient Node<K, V>[] table;

    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     */
    transient Set<Map.Entry<K, V>> entrySet;

    //Map的大小
    transient int size;

    //每当Map的结构发生变化时，此参数就会递增
    //当在对Map进行迭代操作时，迭代器会检查此参数值
    //如果检查到此参数的值发生变化，就说明在迭代的过程中Map的结构发生了变化，因此会直接抛出异常
    transient int modCount;

    //数组的扩容临界点，当数组的数据量达到这个值时就会进行扩容操作
    //计算方法：当前容量 x 装载因子
    int threshold;

    //使用的装载因子值
    final float loadFactor;
```

## 2.5 构造函数

```Java
    //设置Map的初始化大小和装载因子
    public HashMap(int initialCapacity, float loadFactor) {
        //检查参数合法性
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    //设置Map的初始化大小
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    //都使用默认值
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }

    //传入初始数据
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

## 2.6 结点类

```Java
    //结点
    static class Node<K, V> implements Map.Entry<K, V> {
        //当前结点的 key 的哈希值
        final int hash;
        //键
        final K key;
        //值
        V value;
        //下一个结点
        Node<K, V> next;
        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        ···     
    }
```

## 2.7 哈希算法

在查询, 添加和移除键值对时, 定位到哈希桶数组的指定位置都是很关键的第一步, 只有 HashMap 中的元素尽量分布均匀, 才能在定位键值对时快速地查找到相应位置, 避免频繁地去遍历链表或者红黑树, 这就需要依靠于一个比较好的哈希算法了.

以下是 HashMap 中计算 key 值的哈希值以及根据哈希值获取其在哈希桶数组中位置的算法.
```Java
    //计算哈希值
    static final int hash(Object key) {
        int h;
        //高位参与运算
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    //根据 key 值获取 Value
    public V get(Object key) {
        Node<K, V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    //查找指定结点
    final Node<K, V> getNode(int hash, Object key) {
        ···
        //只有当 table 不为空且 hash 对应的位置不为 null 才有可获取的元素值
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
           ···
        }
        return null;
    }
```
确定键值对在哈希桶数组的位置的步骤分为三步: 计算 key 的 hashCode（h = key.hashCode()）, 高位运算（h >>> 16）、取模运算（(n - 1) & hash）

## 2.8 插入数据

在上边说过, HashMap 是数组+链表+红黑树的结合, 数组包含的元素的可能值分为四种类型: null, 单个结点, 链表, 红黑树. 在插入结点时(每一个待存数据都会被包装为结点对象), 会根据待插入 Key 的哈希值来决定结点在数组中的位置, 如果计算得出的位置此时包含的元素为 null , 则直接将结点存入该位置, 如果不为 null , 则说明发生了哈希碰撞, 此时就需要将结点插入到链表或者是红黑树中. 当哈希算法的计算结果越分散均匀, 哈希碰撞的概率就越小, map 的存取效率就会越高.

如果待插入结点的 key 与链表或红黑树中某个已有结点的 key 相等(hash 值相等且两者 equals 成立), 则新添加的结点将覆盖原有数据.
插入数据对应的是 put(K key, V value) 方法.
```Java
    //插入数据
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    //计算哈希值
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    /**
     * Implements Map.put and related methods
     *
     * @param hash         hash for key
     * @param key          the key
     * @param value        the value to put
     * @param onlyIfAbsent 为 true 表示不会覆盖有相同 key 的非 null value，否则会覆盖原有值
     * @param evict        if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, i;
        //如果 table 还未初始化，则调用 resize 方法进行初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //判断要存入的 key 是否存在哈希冲突，等于 null 说明不存在冲突
        if ((p = tab[i = (n - 1) & hash]) == null)
            //直接在索引 i 处构建包含待存入元素的结点
            tab[i] = newNode(hash, key, value, null);
        else { //走入本分支，说明待存入的 key 存在哈希冲突
            Node<K, V> e;
            K k;
            //p 值已在上一个 if 语句中赋值了，此处就直接来判断 key 值之间的相等性
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                //指向冲突的头结点
                e = p;
            //如果头结点的 key 与待插入的 key 不相等，且头结点是 TreeNode 类型，说明该 hash 值是采用红黑树来处理冲突
            else if (p instanceof TreeNode)
                //如果红黑数中包含有相同 key 的结点，则返回该结点，否则返回 null
                e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
            else { //采用链表来处理 hash 值冲突
                for (int binCount = 0; ; ++binCount) {
                    //当遍历到链表尾部时
                    if ((e = p.next) == null) {
                        //构建一个新的结点添加到链表尾部
                        p.next = newNode(hash, key, value, null);
                        //如果链表的长度已达到允许的最大长度 TREEIFY_THRESHOLD - 1 时，就将链表转换为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //当 e 指向的结点的 key 值与待插入的 key 相等时则跳出循环
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果 e != null，说明原先已存在相同 key 的键
            if (e != null) {
                V oldValue = e.value;
                //只有当 onlyIfAbsent 为 true 且 oldValue 不为 null 时才不会覆盖原有值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //用于 LinkedHashMap ，在 HashMap 中是空实现
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //当元素数量达到扩容临界点时，需要进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

## 2.9 读取数据

读取数据对应的是 get(Object key)方法
```Java
    //根据 key 值获取 Value
    public V get(Object key) {
        Node<K, V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    //查找指定结点
    final Node<K, V> getNode(int hash, Object key) {
        Node<K, V>[] tab;
        Node<K, V> first, e;
        int n;
        K k;
        //只有当 table 不为空且 hash 对应的位置不为 null 才有可获取的元素值
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
            //如果头结点的 hash 值与 Key 与待插入数据相等的话，则说明找到了对应值
            if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // first.next != null 说明存在哈希冲突
            if ((e = first.next) != null) {
                //如果是由红黑树来处理哈希冲突，则由此查找相应结点
                if (first instanceof TreeNode)
                    return ((TreeNode<K, V>) first).getTreeNode(hash, key);
                //遍历链表
                do {
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

## 2.10 移除结点

从 Map 中移除键值对的操作, 在底层数据结构的体现就是移除对某个结点对象的引用, 可能是从数组中, 也可能是链表或者红黑树.
```Java
public V remove(Object key) {
        Node<K, V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
                null : e.value;
    }

    /**
     * Implements Map.remove and related methods
     *
     * @param hash       key 的哈希值
     * @param key        the key
     * @param value      key对应的值，只有当 matchValue 为 true 时才需要使用到，否则忽略该值
     * @param matchValue 如果为 true ，则只有当 Map 中存在某个键 equals key 且 value 相等时才会移除该元素，否则只要 key 的 hash 值相等就直接移除该元素
     * @param movable    if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K, V> removeNode(int hash, Object key, Object value,
                                boolean matchValue, boolean movable) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, index;
        //只有当 table 不为空且 hash 对应的索引位置存在值时才有可移除的对象
        if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
            Node<K, V> node = null, e;
            K k;
            V v;
            //如果与头结点的 key 相等
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) { //存在哈希冲突
                //用红黑树来处理哈希冲突
                if (p instanceof TreeNode)
                    //查找对应结点
                    node = ((TreeNode<K, V>) p).getTreeNode(hash, key);
                else { //用链表来处理哈希冲突
                    do {
                        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //node != null 说明存在相应结点
            //如果 matchValue 为 false ，则通过之前的判断可知查找到的结点的 key 与 参数 key 的哈希值一定相等，此处就可以直接移除结点 node
            //如果 matchValue 为 true ，则当 value 相等时才需要移除该结点
            if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
                if (node instanceof TreeNode) //对应红黑树
                    ((TreeNode<K, V>) node).removeTreeNode(this, tab, movable);
                else if (node == p) //对应 key 与头结点相等的情况，此时直接将指针移向下一位即可
                    tab[index] = node.next;
                else //对应的是链表的情况
                    p.next = node.next;
                ++modCount;
                --size;
                //用于 LinkedHashMap ，在 HashMap 中是空实现
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

## 2.11 扩容

如果哈希桶数组很大, 即使用的是较差的哈希算法元素也会比较分散, 如果哈希桶数组很小, 即使用的是好的哈希算法也会出现较多哈希碰撞的情况, 所以就需要在空间成本和时间成本之间权衡, 除了设计较好的哈希算法以减少哈希冲突外, 也需要在合适的的时机对哈希桶数组进行必要的扩容.

当 HashMap 中的元素越来越多时, 因为数组的长度是固定的, 所以哈希冲突的几率也就越来越高, 为了提高效率, 此时就需要对 HashMap 中的数组进行扩容, 而扩容操作最消耗性能的地方就在于: 原数组中的数据必须重新计算其在新数组中的位置并存放到新数组中.

那么 HashMap 扩容操作的触发时机是什么时候呢? 当 HashMap 中的元素个数超出 threshold 时(数组容量 与 loadFactor 的乘积), 就会进行数组扩容. 默认情况下, 数组的默认值为 16, loadFactor 的默认值为 0.75, 这是平衡空间利用率和运行效率两者之后的结果. 也就是说, 假设数组当前大小为16, loadFactor 值为0.75, 那么当 HashMap 中的元素个数达到12个时, 就会自动触发扩容操作, 把数组的大小扩充到 2 * 16 = 32, 即扩大一倍, 然后重新计算每个元素在新数组中的位置, 而这是一个非常消耗性能的操作, <font color=red>所以如果已经预知到待存入 HashMap 的数据量, 那么在初始化 HashMap 时直接指定初始化大小会是一种更为高效的做法.</font>

----

<font color=red>更改: 那么 HashMap 扩容操作的触发时机是什么时候呢?

同时满足下面的两个条件:
1. 存放新值的时候当前已有元素的个数必须大于等于阈值
2. 存放新值的时候当前存放数据发生hash碰撞（当前key计算的hash值换算出来的数组下标位置已经存在值）</font>

扩容操作对应的是 resize()方法

----

### 2.11.1 JDK1.8引入的扩容巧妙设计

经过rehash之后, 元素的位置要么是在原位置, 要么是在原位置再移动2次幂的位置.
图(a)表示扩容前的key1和key2两种key确定索引位置的示例;
图(b)表示扩容后key1和key2两种key确定索引位置的示例, 其中hash1是key1对应的哈希与高位运算结果.
![扩容](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/hashMap01.png)
元素在重新计算hash之后, 因为n变为2倍, 那么n-1的mask范围在高位多1bit(红色), 因此新的index就会发生这样的变化
![扩容](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/hashMap02.png)
因此, 我们在扩充HashMap的时候, 不需要像JDK1.7的实现那样重新计算hash, 只需要看看原来的hash值新增的那个bit是1还是0就好了, 是0的话索引没变, 是1的话索引变成"原索引+oldCap", 可以看看下图为16扩充为32的resize示意图:
![扩容](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/hashMap03.png)
这个设计确实非常的巧妙, 既省去了重新计算hash值的时间, 而且同时, 由于新增的1bit是0还是1可以认为是随机的, 因此resize的过程, 均匀的把之前的冲突的节点分散到新的bucket了. 这一块就是JDK1.8新增的优化点. 

----
有一点注意区别, JDK1.7中rehash的时候, 旧链表迁移新链表的时候, 如果在新表的数组索引位置相同, 则链表元素会倒置, 但是从上图可以看出, JDK1.8不会倒置. 

### 2.11.2 resize源码解读

```Java
    final Node<K, V>[] resize() {
        Node<K, V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 超过最大值就不再扩充了，就只好随你碰撞去吧
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 没超过最大值，就扩充为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        } else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 计算新的resize上限
        if (newThr == 0) {
            float ft = (float) newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ?
                    (int) ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes"，"unchecked"})
        Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 把每个bucket都移动到新的buckets中
            for (int j = 0; j < oldCap; ++j) {
                Node<K, V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                    else { // 链表优化重hash的代码块
                        Node<K, V> loHead = null, loTail = null;
                        Node<K, V> hiHead = null, hiTail = null;
                        Node<K, V> next;
                        do {
                            next = e.next;
                            // 原索引
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }// 原索引+oldCap
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 原索引放到bucket里
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 原索引+oldCap放到bucket里
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

## 2.12 效率测试

这里来测试下不同的初始化大小以及 key 值的 HashCode 值的分布情况的不同对 HashMap 效率的影响
首先来定义作为 Key 的类, hashCode() 方法直接返回其包含的属性 value
```Java
import java.util.Objects;
public class Key {
    private int value;
    public Key(int value) {
        this.value = value;
    }
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Key key = (Key) o;
        return value == key.value;
    }
    @Override
    public int hashCode() {
        return Objects.hash(value);
    }
}
```
初始化大小从 100 到 100000 之间以 10 倍的倍数递增，向 HashMap 存入同等数据量的数据，观察不同 HashMap 存入数据消耗的总时间
```Java
import java.util.HashMap;
import java.util.Map;
public class KeyMain {
    private static final int MAX_KEY = 20000;
    private static final Key[] KEYS = new Key[MAX_KEY];
    static {
        for (int i = 0; i < MAX_KEY; i++) {
            KEYS[i] = new Key(i);
        }
    }
    private static void test(int size) {
        long startTime = System.currentTimeMillis();
        Map<Key, Integer> map = new HashMap<>(size);
        for (int i = 0; i < MAX_KEY; i++) {
            map.put(KEYS[i], i);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("初始化大小是：" + size + " , 所用时间：" + (endTime - startTime) + "毫秒");
    }
    public static void main(String[] args) {
        for (int i = 20; i <= MAX_KEY; i *= 10) {
            test(i);
        }
    }
}
```
运行结果:
```
初始化大小是：20 , 所用时间：9毫秒
初始化大小是：200 , 所用时间：13毫秒
初始化大小是：2000 , 所用时间：5毫秒
初始化大小是：20000 , 所用时间：3毫秒
```
在上述使用的例子中, 各个 Key 对象之间的哈希码值各不相同, 所以键值对在哈希桶数组中的分布可以说是很均匀的了, 此时主要影响性能的就是扩容机制了, 由上图可以看出各个初始化大小对 HashMap 的性能影响还是很大的
接下来再看看各个 Key 对象之间频繁发生哈希冲突时 HashMap 的性能
令 Key 类的 hashCode() 方法固定返回 100, 则每个键值对在存入 HashMap 时, 一定会发生哈希冲突
```Java
    @Override
    public int hashCode() {
        return 100;
    }
```
运行结果:
```
初始化大小是：20 , 所用时间：6192毫秒
初始化大小是：200 , 所用时间：6004毫秒
初始化大小是：2000 , 所用时间：5633毫秒
初始化大小是：20000 , 所用时间：5914毫秒
```
此时主要影响性能的点就在于对哈希冲突的处理了

## 2.13 equals()和hashCode()

<font color=red>在使用Map存对象的时候, 要记得, 一定要重写此类的 equals() 和 hashCode() 方法哦!!!</font>
# 3. 参考链接
[HashMap源码，你知道多少？](https://mp.weixin.qq.com/s/9lr96QekwOvOwwm7g7NhJQ)
[HashMap的扩容机制---resize()](https://blog.csdn.net/aichuanwendang/article/details/53317351)
[深入理解HashMap的扩容机制](https://www.cnblogs.com/yanzige/p/8392142.html)
