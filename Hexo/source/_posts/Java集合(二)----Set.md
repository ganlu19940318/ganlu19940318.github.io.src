---
title: Java集合(二)----Set
date: 2018-09-22 22:22:19
categories: Java基础
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 前言

集合是Java基础中非常重要的一部分, 也是日常中用到非常多的类, 这篇博客主要记录集合Set相关的知识.
<font color=red>特别声明: 大部分内容并非原创, 引用自参考链接.</font>

# 2. Set
Set继承于Collection接口, 是一个不允许出现重复元素, 并且无序的集合, 主要有HashSet和TreeSet两大实现类.
在判断重复元素的时候, Set集合会调用hashCode()和equal()方法来实现.
HashSet是哈希表结构, 主要利用HashMap的key来存储元素, 计算插入元素的hashCode来获取元素在集合中的位置;
TreeSet是红黑树结构, 每一个元素都是树中的一个节点, 插入的元素都会进行排序;
## 2.1 Set常用操作
与List接口一样, Set接口也提供了集合操作的基本方法.
但与List不同的是, Set还提供了equals(Object o)和hashCode(), 供其子类重写, 以实现对集合中插入重复元素的处理;
```Java
public interface Set<E> extends Collection<E> {

    A:添加功能
    boolean add(E e);
    boolean addAll(Collection<? extends E> c);

    B:删除功能
    boolean remove(Object o);
    boolean removeAll(Collection<?> c);
    void clear();
    
    C:长度功能
    int size();

    D:判断功能
    boolean isEmpty();
    boolean contains(Object o);
    boolean containsAll(Collection<?> c);
    boolean retainAll(Collection<?> c); 

    E:获取Set集合的迭代器：
    Iterator<E> iterator();
    
    F:把集合转换成数组
    Object[] toArray();
    <T> T[] toArray(T[] a);

    //判断元素是否重复，为子类提高重写方法
    boolean equals(Object o);
    int hashCode();
}
```
## 2.2 初识HashSet
HashSet实现Set接口, 底层由HashMap(后面讲解)来实现, 为哈希表结构, 新增元素相当于HashMap的key, value默认为一个固定的Object. HashSet相当于一个阉割版的HashMap;
当有元素插入的时候, 会计算元素的hashCode值, 将元素插入到哈希表对应的位置中来;
它继承于AbstractSet, 实现了Set, Cloneable, Serializable接口.
(1)HashSet继承AbstractSet类, 获得了Set接口大部分的实现, 减少了实现此接口所需的工作, 实际上是又继承了AbstractCollection类;
(2)HashSet实现了Set接口, 获取Set接口的方法, 可以自定义具体实现, 也可以继承AbstractSet类中的实现;
(3)HashSet实现Cloneable, 得到了clone()方法, 可以实现克隆功能;
(4)HashSet实现Serializable, 表示可以被序列化, 通过序列化去传输, 典型的应用就是hessian协议.
具有如下特点:
(1)不允许出现重复因素;
(2)允许插入Null值;
(3)元素无序(添加顺序和遍历顺序不一致);
(4)线程不安全, 若2个线程同时操作HashSet, 必须通过代码实现同步;
### 2.2.1 HashSet元素添加
Set集合不允许添加重复元素, 那么到底是个怎么情况呢?
来看一个简单的例子:
```Java
public class HashSetTest {
    public static void main(String[] agrs){
        //hashCode() 和 equals()测试：
        hashCodeAndEquals();
    }
    public static void hashCodeAndEquals(){
        //第一个 Set集合：
        Set<String> set1 = new HashSet<String>();
        String str1 = new String("jiaboyan");
        String str2 = new String("jiaboyan");
        set1.add(str1);
        set1.add(str2);
        System.out.println("长度："+set1.size()+",内容为："+set1);
        //第二个 Set集合：
        Set<App> set2 = new HashSet<App>();
        App app1 = new App();
        app1.setName("jiaboyan");
        App app2 = new App();
        app2.setName("jiaboyan");
        set2.add(app1);
        set2.add(app2);
        System.out.println("长度："+set2.size()+",内容为："+set2);
        //第三个 Set集合：
        Set<App> set3 = new HashSet<App>();
        App app3 = new App();
        app3.setName("jiaboyan");
        set3.add(app3);
        set3.add(app3);
        System.out.println("长度："+set3.size()+",内容为："+set3);
    }
}
```
测试结果:
```
长度：1,内容为：[jiaboyan]
长度：2,内容为：[App@74a14482, App@4554617c]
长度：1,内容为：[App@1540e19d]
```
可以看到, 第一个Set集合中最终只有一个元素; 第二个Set集合保留了2个元素; 第三个集合也只有1个元素;
究竟是什么原因呢?
让我们来看看HashSet的add(E e)方法:
```Java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```
在底层HashSet调用了HashMap的put(K key, V value)方法:
```Java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```
简单概括如下:
在向HashMap中添加元素时, 先判断key的hashCode值是否相同, 如果相同, 则调用==, equals()进行判断, 若相同则覆盖原有元素; 如果不同, 则直接向Map中添加元素;
反过来, 我们在看下上面的例子:
在第一个Set集合中, 我们new了两个String对象, 赋了相同的值. 当传入到HashMap中时, key均为"jiaboyan", 所以hash和i的值都相同. 进行if (e.hash == hash && ((k = e.key) == key || key.equals(k)))判断, 由于String对象重写了equals()方法,所以在((k = e.key) == key || key.equals(k))判断时, 返回了true, 所以第二次的插入并不会增加Set集合的长度;
第二个Set集合中, 也是new了两个对象, 但没有重写equals()方法(底层调用的Object的equals()，也就是==判断), 所以会增加2个元素;
第三个Set集合中, 只new了一个对象, 调用的两次add方法都添加的这个新new的对象,所以也只是保留了1个元素;
## 2.3 初识TreeSet
从名字上可以看出, 此集合的实现和树结构有关.与HashSet集合类似, TreeSet也是基于Map来实现, 具体实现TreeMap, 其底层结构为红黑树;
与HashSet不同的是, TreeSet具有排序功能, 分为自然排序(123456)和自定义排序两类, 默认是自然排序; 在程序中, 我们可以按照任意顺序将元素插入到集合中, 等到遍历时TreeSet会按照一定顺序输出--倒序或者升序;
它继承AbstractSet, 实现NavigableSet, Cloneable, Serializable接口.
(1)与HashSet同理, TreeSet继承AbstractSet类,获得了Set集合基础实现操作;
(2)TreeSet实现NavigableSet接口, 而NavigableSet又扩展了SortedSet接口. 这两个接口主要定义了搜索元素的能力, 例如给定某个元素,查找该集合中比给定元素大于, 小于, 等于的元素集合, 或者比给定元素大于, 小于, 等于的元素个数; 简单地说, 实现NavigableSet接口使得TreeSet具备了元素搜索功能;
(3)TreeSet实现Cloneable接口, 意味着它也可以被克隆;
(4)TreeSet实现了Serializable接口, 可以被序列化, 可以使用hessian协议来传输;
具有如下特点:
(1)对插入的元素进行排序, 是一个有序的集合(主要与HashSet的区别);
(2)底层使用红黑树结构, 而不是哈希表结构;
(3)允许插入Null值;
(4)不允许插入重复元素;
(5)线程不安全;
### 2.3.1 TreeSet元素排序
在前面的章节, 我们讲到了TreeSet是一个有序集合,可以对集合元素排序,其中分为自然排序和自定义排序,那么这两种方式如何实现呢?
首先,我们通过JDK提供的对象来展示, 我们使用String, Integer:
```Java
public class TreeSetTest {
    public static void main(String[] agrs){
        naturalSort();
    }
    //自然排序顺序：升序
    public static void naturalSort(){
        TreeSet<String> treeSetString = new TreeSet<String>();
        treeSetString.add("a");
        treeSetString.add("z");
        treeSetString.add("d");
        treeSetString.add("b");
        System.out.println("字母顺序：" + treeSetString.toString());
        TreeSet<Integer> treeSetInteger = new TreeSet<Integer>();
        treeSetInteger.add(1);
        treeSetInteger.add(24);
        treeSetInteger.add(23);
        treeSetInteger.add(6);
        System.out.println("数字顺序：" + treeSetInteger.toString());
    }
}
```
测试结果:
```
字母顺序：[a, b, d, z]
数字顺序：[1, 6, 23, 24]
```
----
接下来, 我们自定义对象, 看能否实现:
```Java
import java.util.TreeSet;
public class Main {
    public static void main(String[] agrs){
        customSort();
    }
    //自定义排序顺序：升序
    public static void customSort(){
        TreeSet<App> treeSet = new TreeSet<App>();
        //排序对象：
        App app1 = new App("hello",10);
        App app2 = new App("world",20);
        App app3 = new App("my",15);
        App app4 = new App("name",25);
        //添加到集合：
        treeSet.add(app1);
        treeSet.add(app2);
        treeSet.add(app3);
        treeSet.add(app4);
        System.out.println("TreeSet集合顺序为："+treeSet);
    }
}
class App{
    private String name;
    private Integer age;
    public App(){}
    public App(String name,Integer age){
        this.name = name;
        this.age = age;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Integer getAge() {
        return age;
    }
    public void setAge(Integer age) {
        this.age = age;
    }
    public static void main(String[] args ){
        System.out.println( "Hello World!" );
    }
}
```
测试结果:

```Java
Exception in thread "main" java.lang.ClassCastException: App cannot be cast to java.lang.Comparable
	at java.util.TreeMap.compare(TreeMap.java:1294)
	at java.util.TreeMap.put(TreeMap.java:538)
	at java.util.TreeSet.add(TreeSet.java:255)
	at Main.customSort(Main.java:15)
	at Main.main(Main.java:4)
```
----
为什么会报错呢?
```Java
compare(key, key); // type (and possibly null) check
final int compare(Object k1, Object k2) {
    return comparator==null ? ((Comparable<? super K>)k1).compareTo((K)k2)
        : comparator.compare((K)k1, (K)k2);
}
```
通过查看源码发现, 在TreeSet调用add方法时, 会调用到底层TreeMap的put方法, 在put方法中会调用到compare(key, key)方法, 进行key大小的比较;
在比较的时候, 会将传入的key进行类型强转, 所以当我们自定义的App类进行比较的时候, 自然就会抛出异常, 因为App类并没有实现Comparable接口;
将App实现Comparable接口, 再做比较:
```Java
class App implements Comparable<App>{
    private String name;
    private Integer age;
    public App(){}
    public App(String name,Integer age){
        this.name = name;
        this.age = age;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Integer getAge() {
        return age;
    }
    public void setAge(Integer age) {
        this.age = age;
    }
    public static void main(String[] args ){
        System.out.println( "Hello World!" );
    }
    @Override
    //自定义比较：先比较name的长度，在比较age的大小；
    public int compareTo(App app) {
        //比较name的长度：
        int num = this.name.length() - app.name.length();
        //如果name长度一样，则比较年龄的大小：
        return num == 0 ? this.age - app.age : num;
    }
    @Override
    public String toString() {
        return "App{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
测试结果:
```
TreeSet集合顺序为：[App{name='my', age=15}, App{name='name', age=25}, App{name='hello', age=10}, App{name='world', age=20}]
```
----
此外, 还有另一种方式, 那就是实现Comparetor接口, 并重写compare方法;
```Java
//自定义App类的比较器：
public class AppComparator implements Comparator<App> {
    //比较方法：先比较年龄，年龄若相同在比较名字长度；
    public int compare(App app1, App app2) {
        int num = app1.getAge() - app2.getAge();
        return num == 0 ? app1.getName().length() - app2.getName().length() : num;
    }
}
```
此时, App不用在实现Comparerable接口了, 单纯的定义一个类即可;
```Java
import java.util.Comparator;
import java.util.TreeSet;
public class Main {
    public static void main(String[] agrs){
        customSort();
    }
    //自定义排序顺序：升序
    public static void customSort(){
        TreeSet<App> treeSet = new TreeSet<App>(new AppComparator());
        //排序对象：
        App app1 = new App("hello",10);
        App app2 = new App("world",20);
        App app3 = new App("my",15);
        App app4 = new App("name",25);
        //添加到集合：
        treeSet.add(app1);
        treeSet.add(app2);
        treeSet.add(app3);
        treeSet.add(app4);
        System.out.println("TreeSet集合顺序为："+treeSet);
    }
}
class App{
    private String name;
    private Integer age;
    public App(){}
    public App(String name,Integer age){
        this.name = name;
        this.age = age;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Integer getAge() {
        return age;
    }
    public void setAge(Integer age) {
        this.age = age;
    }
    public static void main(String[] args ){
        System.out.println( "Hello World!" );
    }
    @Override
    public String toString() {
        return "App{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
//自定义App类的比较器：
class AppComparator implements Comparator<App> {
    //比较方法：先比较年龄，年龄若相同在比较名字长度；
    public int compare(App app1, App app2) {
        int num = app1.getAge() - app2.getAge();
        return num == 0 ? app1.getName().length() - app2.getName().length() : num;
    }
}
```
测试结果:
```
TreeSet集合顺序为：[App{name='hello', age=10}, App{name='my', age=15}, App{name='world', age=20}, App{name='name', age=25}]
```
最后, 再说下关于compareTo(), compare()方法:
```
结果返回大于0时，方法前面的值大于方法中的值；
结果返回等于0时，方法前面的值等于方法中的值；
结果返回小于0时，方法前面的值小于方法中的值；
```
## 2.4 HashSet源码分析(基于JDK1.7.0_75)
HashSet基于HashMap, 底层方法是通过调用HashMap的API来实现, 因此HashSet源码结构比较简单, 代码较少.
### 2.4.1 成员变量
在HashSet中, 有两个成员变量比较重要--map, PRESENT;
其中,map就是存储元素的地方, 实际是一个HashMap. 当有元素插入到HashSet中时, 会被当做HashMap的key保存到map属性中去.
对于HashMap来说,光有key还不够, 在HashSet的实现中, 每个key对应的value都默认为PRESENT属性, 也就是new了一个Object对象而已;
```Java
public class HashSet<E> 
    extends AbstractSet<E> 
    implements Set<E>, Cloneable, java.io.Serializable{
    static final long serialVersionUID = -5024744406713321676L;
    //HashSet通过HashMap保存集合元素的：
    private transient HashMap<E,Object> map;
    //HashSet底层由HashMap实现，新增的元素为map的key，而value则默认为PRESENT。
    private static final Object PRESENT = new Object();
}
```
### 2.4.2 构造方法
HashSet的构造方法很简单, 主要是在方法内部初始化map属性, new了一个HashMap对象;
```Java
public class HashSet<E>
        extends AbstractSet<E>
        implements Set<E>, Cloneable, java.io.Serializable {
    //无参构造方法：
    public HashSet() {
        //默认new一个HashMap
        map = new HashMap<>();
    }
    // 带集合的构造函数
    public HashSet(Collection<? extends E> c) {
        // 进行初始化HashMap容量判断，
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
    // 指定HashSet初始容量和加载因子的构造函数：主要用于Map内部的扩容机制
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }
    // 指定HashSet初始容量的构造函数
    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
    //与前4个不同，此构造最终new了一个LinkedHashMap对象：
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
}
```
### 2.4.3 add()
HashSet的add(E e)方法, 主要是调用底层HashMap的put(K key, V value)方法.
其中key就是HashSet集合插入的元素, 而value则是默认的PRESENT属性(一个new Object());
```Java
//调用HashMap中的put()方法:
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```
### 2.4.4 remove()
与add(E e)方法类似, HashSet的remove(Object o)也是调用了底层HashMap的(Object key)方法;
主要是计算出要删除元素的hash值, 在HashMap找到对应的对象, 然后从Entry[]数组中删除;
```Java
//调用HashMap中的remove方法：
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```
## 2.5 TreeSet源码分析(基于JDK1.7.0_75)
与HashSet类似, TreeSet底层也是采用了一个Map来保存集合元素, 这个Map就是NavigableMap.
不过, NavigableMap仅仅是一个接口, 具体的实现还是使用了TreeMap类;
### 2.5.1 成员变量
成员变量m是一个NavigableMap类型的Map集合, 常用实现是TreeMap对象;
在TreeMap中, key是我们TreeSet插入的元素, 而value则是TreeSet中另一个成员变量PRESENT, 一个普通的不能再普通的Object对象;
```Java
public class TreeSet<E> extends AbstractSet<E>
        implements NavigableSet<E>, Cloneable, java.io.Serializable {
    //TreeSet中保存元素的map对象：
    private transient NavigableMap<E,Object> m;
    //map对象中保存的value:
    private static final Object PRESENT = new Object();
}
```
### 2.5.2 构造方法
```Java
public class TreeSet<E> extends AbstractSet<E>
implements NavigableSet<E>, Cloneable, java.io.Serializable {
  //最底层的构造方法，不对外。传入一个NavigableMap接口的实现类
  TreeSet(NavigableMap<E,Object> m) {
      this.m = m;
  }
  //无参构造：向底层构造传入一个TreeMap对象：
  public TreeSet() {
      this(new TreeMap<E,Object>());
  }
  //传入比较器的构造：通常传入一个自定义Comparator的实现类；
  public TreeSet(Comparator<? super E> comparator) {
      this(new TreeMap<>(comparator));
  }
  //将集合Collection传入TreeSet中：
  public TreeSet(Collection<? extends E> c) {
      this();
      addAll(c);
  }
  //将集合SortedSet传入TreeSet中：
  public TreeSet(SortedSet<E> s) {
      this(s.comparator());
      addAll(s);
  }
}
```
### 2.5.3 add()
向TreeSet中添加元素
```Java
public boolean add(E e) {
    return m.put(e, PRESENT)==null;
}
```
### 2.5.4 remove()
删除TreeSet中元素o
```Java
public boolean remove(Object o) {
    return m.remove(o)==PRESENT;
}
```
## 2.6 SortedSet和NavigableSet到底是什么
在一些关于TreeSet讲解的文章中, 在介绍TreeSet的时候都会提到NavigableSet, 接着会说下NavigableSet是个"导航Set集合", 提供了一系列"导航"方法. 那么, 什么是"导航"方法?
通过接口的定义, 我们可以看到NavigableSet继承了SortedSet接口(后面说), 实现了对其的扩展;
而通过下面的方法, 我们得出NavigableSet实际提供了一系列的搜索匹配元素的功能, 能获取到某一区间内的集合元素;
```Java
public interface NavigableSet<E> extends SortedSet<E> {
     E lower(E e);//返回此set集合中小于e元素的最大元素
     E floor(E e);//返回此set集合中小于等于e元素的最大元素
     E ceiling(E e);//返回此set集合中大于等于e元素的最小元素
     E higher(E e);//返回此set集合中大于e元素的最小元素
     E pollFirst(); //获取并移除此set集合中的第一个元素
     E pollLast();//获取并移除此set集合中的最后一个元素
     Iterator<E> iterator();//返回此set集合的迭代器--升序
     NavigableSet<E> descendingSet();//以倒序的顺序返回此set集合
     Iterator<E> descendingIterator();//返回此set集合的迭代器--倒序
     //返回此set集合的部分元素--从fromElement开始到toElement结束，其中fromInclusive、toInclusive意为返回的集合是否包含头尾元素
     NavigableSet<E> subSet(E fromElement, boolean fromInclusive, E toElement, boolean toInclusive);
     //返回此set集合的部分元素--小于toElement，inclusive意味返回的集合是否包含toElement
     NavigableSet<E> headSet(E toElement, boolean inclusive);
     //返回此set集合的部分元素--从fromElement开始到toElement结束，包含头不含为尾
     SortedSet<E> subSet(E fromElement, E toElement);
     //返回此set集合的部分元素--小于toElement
     SortedSet<E> headSet(E toElement);
     //返回此set集合的部分元素--大于等于toElement
     SortedSet<E> tailSet(E fromElement);
}
```
说完了NavigableSet, 我们在一起儿看下其父类SortedSet接口:
通过名字, 我们可以得出此接口跟排序有关, 会提供跟排序的方法;
```Java
public interface SortedSet<E> extends Set<E> {
    //返回与排序有关的比较器
    Comparator<? super E> comparator();
    //返回从fromElement到toElement的元素集合：
    SortedSet<E> subSet(E fromElement, E toElement);
    //返回从第一个元素到toElement元素的集合：
    SortedSet<E> headSet(E toElement);
    //返回从fromElement开始到最后元素的集合：
    SortedSet<E> tailSet(E fromElement);
    //返回集合中的第一个元素：
    E first();
    //返回集合中的最后一个元素：
    E last();
}
```
# 3. 参考链接
[Java集合：Set源码详细分析(一)](https://mp.weixin.qq.com/s/R2iFPE2eO2KdkRp0vEpdlA)
[Java集合：Set源码详细分析(二)](https://mp.weixin.qq.com/s/ZTNJIbrDfLUdkEU5aiDRFQ)