---
title: Java字符串处理
date: 2018-09-23 21:16:18
categories: Java基础
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 前言
对字符串的操作是后台开发中非常常见的, 对字符串相关的类有深刻的认识有助于写出高质量的代码. 这篇文章主要是介绍String, StringBuffer, StringBuilder这三个字符串相关的类.
# 2. 基本认识
## 2.1 String 
```Java
/** Strings are constant; their values cannot be changed after they
 * are created. String buffers support mutable strings.
 * Because String objects are immutable they can be shared. 
 * 字符串是不变的，他们的值在创造后不能改变。
 * 字符串缓冲区支持可变字符串，因为字符串对象是不可变的，所以它们可以共享。
 * 
 * @see StringBuffer
 * @see StringBuilder
 * @see Charset
 * @since 1.0
 */
public final class String implements Serializable, Comparable<String>, CharSequence {
    private static final long serialVersionUID = -6849794470754667710L;
    private static final char REPLACEMENT_CHAR = (char) 0xfffd;
```
这句话总结归纳了String的两个最重要的特点:
String是值不可变的常量, 是线程安全的(can be shared).
String类使用了final修饰符, String类是不可继承的.
## 2.2 StringBuffer
StringBuffer字符串变量(线程安全)是一个容器, 最终会通过toString方法变成字符串;
```Java
public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, Appendable, CharSequence
{
    /**
     * Constructs a string buffer with no characters in it and an
     * initial capacity of 16 characters.
     */
    public StringBuffer() {
        super(16);
    }
    public synchronized StringBuffer append(int i) {
        super.append(i);
        return this;
    }
    public synchronized StringBuffer delete(int start, int end) {
        super.delete(start, end);
        return this;
    }
}
```
## 2.3 StringBuilder
StringBuilder 字符串变量(非线程安全).
```Java
public final class StringBuilder extends AbstractStringBuilder implements java.io.Serializable, Appendable, CharSequence {
   public StringBuilder() {
        super(16);
   }
   public StringBuilder append(String str) {
        super.append(str);
        return this;
    }
    public StringBuilder delete(int start, int end) {
        super.delete(start, end);
        return this;
    }
}
```
# 3. 源码理解
## 3.1 String 与 StringBuffer/StringBuilder 的区别
<font color=red>String和StringBuffer/StringBuilder底层都是一个char数组, 但是
String的char数组是final的;
StringBuffer/StringBuilder的char数组不是final的.</font>
(1)String在修改时不会改变对象自身, 在每次对 String 类型进行改变的时候其实都等同于生成了一个新的 String 对象, 然后将指针指向新的 String 对象, 所以经常改变内容的字符串最好不要用 String.
```Java
public class Main {
    public static void main(String[] args)  {
        String str = "abc";
        String str2 = str + "";
        System.out.println(str == str2);
    }
}
```
结果:
```
false
```
(2)StringBuffer/StringBuilder在修改时会改变对象自身, 每次结果都会对StringBuffer/StringBuilder对象本身进行操作, 而不是生成新的对象, 再改变对象引用. 所以在一般情况下我们推荐使用StringBuffer/StringBuilder, 特别是字符串对象经常改变的情况下.StringBuffer/StringBuilder上的主要操作是 append 和 insert 方法.
```Java
public class Main {
    public static void main(String[] args)  {
        StringBuffer stringBuffer = new StringBuffer("abc");
        StringBuffer stringBuffer2 = stringBuffer.append("a");
        System.out.println(stringBuffer == stringBuffer2);
    }
}
```
结果:
```
true
```
## 3.2 StringBuffer 与 StringBuilder 的区别
StringBuffer: 线程安全的, 通过synchronized实现; 
StringBuilder: 线程非安全的.

# 4. 总结

1. 如果要操作少量的数据用 String; 
2. 多线程操作字符串缓冲区下操作大量数据 StringBuffer; 
3. 单线程操作字符串缓冲区下操作大量数据 StringBuilder.

# 5. 参考链接

[Java基础之String、StringBuffer与StringBuilder的区别及应用场景](https://blog.csdn.net/chenliguan/article/details/51911906)
