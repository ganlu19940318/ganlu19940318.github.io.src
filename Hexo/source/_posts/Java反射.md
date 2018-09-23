---
title: Java反射
date: 2018-09-23 22:26:25
categories: Java基础
tags: [Java, 基础储备]
---

----

<!-- more -->

# 1. 前言

Java反射也是日常开发和阅读源码中经常遇到的, 掌握反射是非常有必要的.

# 2. 概述

## 2.1 什么是反射

简单来说, 反射可以帮助我们在动态运行的时候, 对于任意一个类, 可以获得其所有的方法(包括 public protected private 默认状态的), 所有的变量(包括 public protected private 默认状态的).
<font color=red>反射就是把Java类中的各种成分映射成一个个的Java对象.</font>
例如: 一个类有: 成员变量, 方法, 构造方法, 包等等信息, 利用反射技术可以对一个类进行解剖, 把个个组成部分映射成一个个对象. 
如图是类的正常加载过程: 反射的原理在于Class对象.
![反射](https://blogpictures-1257055754.cos.ap-guangzhou.myqcloud.com/20170513133210763.png)

## 2.2 反射有什么用

a. 获取某些类的一些变量, 调用某些类的私有方法
b. 增加代码的灵活性. 很多主流框架都使用了反射技术.

# 3. 反射的使用

假如有这样一个类 Person, 它拥有多个成员变量, country,city,name,province,height,age 等, 同时它拥有多个 构造方法, 多个方法, 这些变量, 方法的访问权限既有 public 也有 private 的. 下面我们以这个为例子, 一起看怎样使用反射获得相应的 Filed, Constructor, Method.
```Java
import java.lang.reflect.Method;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.HashMap;
public class Person {
    public String country;
    public String city;
    private String name;
    private String province;
    private Integer height;
    private Integer age;
    public Person() {
        System.out.println("调用Person的无参构造方法");
    }
    private Person(String country, String city, String name) {
        this.country = country;
        this.city = city;
        this.name = name;
    }
    public Person(String country, Integer age) {
        this.country = country;
        this.age = age;
    }
    private String getMobile(String number) {
        String mobile = "010-110" + "-" + number;
        return mobile;
    }
    private void setCountry(String country) {
        this.country=country;
    }
    public void getGenericHelper(HashMap<String, Integer> hashMap) {
    }
    public Class getGenericType() {
        try {
            HashMap<String, Integer> hashMap = new HashMap<String, Integer>();
            Method method = getClass().getDeclaredMethod("getGenericHelper",HashMap.class);
            Type[] genericParameterTypes = method.getGenericParameterTypes();
            if (null == genericParameterTypes || genericParameterTypes.length < 1) {
                return null;
            }
            ParameterizedType parameterizedType=(ParameterizedType)genericParameterTypes[0];
            Type rawType = parameterizedType.getRawType();
            System.out.println("----> rawType=" + rawType);
            Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
            if (actualTypeArguments==genericParameterTypes || actualTypeArguments.length<1) {
                return null;
            }
            for (int i = 0; i < actualTypeArguments.length; i++) {
                Type type = actualTypeArguments[i];
                System.out.println("----> type=" + type);
            }
        } catch (Exception e) {
        }
        return null;
    }
    @Override
    public String toString() {
        return "Person{" +
                "country='" + country + '\'' +
                ", city='" + city + '\'' +
                ", name='" + name + '\'' +
                ", province='" + province + '\'' +
                ", height=" + height +
                '}';
    }
}
```

## 3.1 获得构造方法

几个重要的方法
| 方法                                                                  | 描述                                                                         |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| public Constructor getConstructor(Class… parameterTypes)              | 获得指定的构造方法，注意只能获得 public 权限的构造方法，其他访问权限的获取不到 |
| public Constructor getDeclaredConstructor(Class… parameterTypes)      | 获得指定的构造方法，注意可以获取到任何访问权限的构造方法。 |
| public Constructor[] getConstructors() throws SecurityException         | 获得所有 public 访问权限的构造方法                                |
| public Constructor[] getDeclaredConstructors() throws SecurityException | 获得所有的构造方法，包括（public, private,protected,默认权限的） |
### 3.1.1 获得所有的构造方法
```Java
import java.lang.reflect.Constructor;
public class Main{
    public static void main(String[] args){
        String className = Person.class.getName();
        printConstructor(className);
    }
    public static void printConstructor(String className) {
        try {
            Class aClass = Class.forName(className);
            Constructor[] constructors = aClass.getConstructors();
            print(constructors);
            System.out.println("=====================");
            Constructor[] declaredConstructors = aClass.getDeclaredConstructors();
            print(declaredConstructors);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
    public static void print(Constructor[] constructors){
        for(Constructor constructor : constructors){
            System.out.println(constructor.toString());
        }
    }
}
```
运行结果:
```Java
public Person(java.lang.String,java.lang.Integer)
public Person()
=====================
public Person(java.lang.String,java.lang.Integer)
private Person(java.lang.String,java.lang.String,java.lang.String)
public Person()
```

### 3.1.2 获得指定的构造方法

```Java
import java.lang.reflect.Constructor;
public class Main{
    public static void main(String[] args){
        String className = Person.class.getName();
        Constructor constructor = getConstructor(className, String.class, Integer.class);
        try {
            Object meinv = constructor.newInstance("CHINA", 24);
            Person person = (Person) meinv;
            System.out.println("testConstructor: = " + person.toString());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static Constructor getConstructor(String className, Class<?>... clzs) {
        try {
            Class<?> aClass = Class.forName(className);
            Constructor<?> declaredConstructor = aClass.getDeclaredConstructor(clzs);
            print(declaredConstructor);
            //   if Constructor is not public,you should call this
            declaredConstructor.setAccessible(true);
            return declaredConstructor;
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
        return null;

    }
    public static void print(Constructor constructor){
        System.out.println(constructor.toString());
    }
}
```
运行结果:
```Java
public Person(java.lang.String,java.lang.Integer)
testConstructor: =Person{country='CHINA', city='null', name='null', province='null', height=null}
```
这说明我们成功通过反射调用 Person 带两个参数的沟改造方法.
### 3.1.3 注意事项
如果该方法, 或者该变量不是 public 访问权限的, 我们应该调用相应的 setAccessible(true) 方法, 才能访问得到
```Java
//if Constructor is not public,you should call this
declaredConstructor.setAccessible(true);
```
## 3.2 获得Filed变量
### 3.2.1 获得所有的Filed变量
```Java
import java.lang.reflect.Field;
public class Main{
    public static void main(String[] args){
        String className = Person.class.getName();
        printFiled(className);
    }
    public static void printFiled(String className) {
        try {
            Class aClass = Class.forName(className);
            Field[] fields = aClass.getFields();
            print(fields);
            System.out.println("============================");
            Field[] declaredFields = aClass.getDeclaredFields();
            print(declaredFields);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
    public static void print(Field[] fields){
        for(Field field : fields){
            System.out.println(field.toString());
        }
    }
}
```
运行结果:
```Java
public java.lang.String Person.country
public java.lang.String Person.city
============================
public java.lang.String Person.country
public java.lang.String Person.city
private java.lang.String Person.name
private java.lang.String Person.province
private java.lang.Integer Person.height
private java.lang.Integer Person.age
```

### 3.2.2 获得指定的Filed变量
现在假如我们要获得 Person 中的私有变量 age , 我们可以通过以下的代码获得.
```Java
import java.lang.reflect.Field;
public class Main{
    public static void main(String[] args) throws Exception{
        String className = Person.class.getName();
        Person person = new Person("CHINA", 12);
        Field field = getFiled(className, "age");
        Integer integer = (Integer) field.get(person);
        System.out.println("integer = " + integer);
    }
    public static Field getFiled(String className, String filedName) throws Exception{
        Class aClass = Class.forName(className);
        Field declaredField = aClass.getDeclaredField(filedName);
        //if not public,you should call this
        declaredField.setAccessible(true);
        return declaredField;
    }
}
```
运行结果:
```
integer = 12
```
## 3.3 执行Method
主要有以下几个方法, 
```Java
public Method[] getDeclaredMethods()
public Method[] getMethods() throws SecurityException
public Method getDeclaredMethod()
public Method getMethod(String name, Class<?> ... parameterTypes)
```
### 3.3.1 获取所有的Method
```Java
import java.lang.reflect.Method;
public class Main{
    public static void main(String[] args){
        String className = Person.class.getName();
        printMethods(className);
    }
    public static void printMethods(String className) {
        try {
            Class<?> aClass = Class.forName(className);
            Method[] declaredMethods = aClass.getDeclaredMethods();
            print(declaredMethods);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
    public static void print(Method[] declaredMethods){
        for(Method method : declaredMethods){
            System.out.println(method.toString());
        }
    }
}
```
运行结果:
```Java
public java.lang.String Person.toString()
public java.lang.Class Person.getGenericType()
private void Person.setCountry(java.lang.String)
public void Person.getGenericHelper(java.util.HashMap)
private java.lang.String Person.getMobile(java.lang.String)
```
### 3.3.2 获取指定的Method
```Java
import java.lang.reflect.Method;
public class Main{
    public static void main(String[] args) throws Exception{
        String className = Person.class.getName();
        Person person=new Person();
        Method method = getMethod(className, "setCountry", String.class);
        // 执行方法，结果保存在 person 中
        Object o = method.invoke(person, "CHINA");
        // 拿到我们传递进取的参数 country 的值 China
        String country=person.country;
        System.out.println("country : " + country);
    }
    public static Method getMethod(String className, String methodName, Class<?>... clzs) throws Exception {
        Class<?> aClass = Class.forName(className);
        Method declaredMethod = aClass.getDeclaredMethod(methodName, clzs);
        declaredMethod.setAccessible(true);
        return declaredMethod;
    }
}
```
运行结果:
```
country : CHINA
```
## 3.4 操作数组
```Java
import java.lang.reflect.Array;
public class Main{
    public static void main(String[] args) {
        testArrayClass();
    }
    /**
     * 利用反射操作数组
     * 1 利用反射修改数组中的元素
     * 2 利用反射获取数组中的每个元素
     */
    public static void testArrayClass() {
        String[] strArray = new String[]{"5","7","暑期","美女","女生","女神"};
        Array.set(strArray,0,"帅哥");
        Class clazz = strArray.getClass();
        if (clazz.isArray()) {
            int length = Array.getLength(strArray);
            for (int i = 0; i < length; i++) {
                Object object = Array.get(strArray, i);
                String className=object.getClass().getName();
                System.out.println("----> object=" + object+",className="+className);
            }
        }
    }
}
```
运行结果:
```Java
----> object=帅哥,className=java.lang.String
----> object=7,className=java.lang.String
----> object=暑期,className=java.lang.String
----> object=美女,className=java.lang.String
----> object=女生,className=java.lang.String
----> object=女神,className=java.lang.String
```
从结果可以说明, 我们成功通过 Array.set(strArray,0,”帅哥”) 改变数组的值.
## 3.5 获得泛型类型
```Java
public static void getGenericHelper(HashMap<String, Person> map) {
}
```
现在假设我们有这样一个方法, 那我们要怎样获得 HashMap 里面的 String, Person 的类型呢?
```Java
import java.lang.reflect.Method;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.HashMap;
public class Main{
    public static void main(String[] args) throws Exception{
        getGenericType();
    }
    public static void getGenericType() throws Exception{
        Method method =TestHelper.class.getDeclaredMethod("getGenericHelper", HashMap.class);
        Type[] genericParameterTypes = method.getGenericParameterTypes();
        // 检验是否为空
        if (null == genericParameterTypes || genericParameterTypes.length < 1) {
            return ;
        }
        // 取 getGenericHelper 方法的第一个参数
        ParameterizedType parameterizedType=(ParameterizedType)genericParameterTypes[0];
        Type rawType = parameterizedType.getRawType();
        System.out.println("----> rawType=" + rawType);
        Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
        if (actualTypeArguments==genericParameterTypes || actualTypeArguments.length<1) {
            return ;
        }
        //  打印出每一个类型
        for (int i = 0; i < actualTypeArguments.length; i++) {
            Type type = actualTypeArguments[i];
            System.out.println("----> type=" + type);
        }
    }
}
```
运行结果:
```Java
----> rawType=class java.util.HashMap
----> type=class java.lang.String
----> type=class Person
```
## 3.6 怎样获得Metho, Field, Constructor的访问权限(public, private, ptotected等)
其实很简单, 我们阅读文档可以发现他们都有 getModifiers() 方法, 该方法放回 int 数字,  我们在利用 Modifier.toString() 就可以得到他们的访问权限.
```Java
int modifiers = method.getModifiers();
Modifier.toString(modifiers);
```
# 4. 参考链接
[Java基础之—反射（非常重要）](https://blog.csdn.net/sinat_38259539/article/details/71799078)
[Java 反射机制详解](https://blog.csdn.net/gdutxiaoxu/article/details/68947735)
