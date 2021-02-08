---

layout: post
title:  "自动拆箱与装箱"
date:   2021-02-08 16:56:23
categories: Java基础
tags:  integer 自动拆箱 自动装箱
author: ddmcc
---

* content
{:toc}

  
### **自动拆装箱（1.5引入）**

#### **什么是自动拆、装箱？**

- 自动装箱：就是将基本数据类型自动转换成对应的包装类
- 自动拆箱：就是将包装类自动转换成对应的基本数据类型



### **自动拆装箱有什么好处**

- 方便

基本类型和包装类型转换无需手动编码实现，由编译器帮我们完成

- 节省空间

大部分包装类型都会有缓存的存在，缓存一定范围大小的数据。如果是在范围内大小，则直接返回缓存内的对象，而不新建。节省的存储空间


### **自动装箱与自动拆箱的实现原理**

**自动装箱都是通过包装类的 valueOf() 方法来实现的,自动拆箱都是通过包装类对象的 xxxValue() 来实现的**

如有以下代码：

```java
    public static void main(String[] args) {
        Integer a = 1;
        int b = a;
    }
```

进行反编译后的代码如下：

```java
    public static void main(String[] args) {
        Integer a = Integer.valueOf((int)1);
        int b = a.intValue();
    }
```

从上面反编译后的代码可以看出，int 的自动装箱是通过 Integer.valueOf() 方法来实现的。Integer 的自动拆箱是通过 integer.intValue 来实现的。

如果将八种类型都反编译一遍就会发现上面说的规律：装箱valueOf(),拆箱XXXValue



### **自动拆装箱带来的问题**


##### **==比较问题**

```java
    public static void main(String[] args) {
        Integer a = 100;
        Integer b = 100;
        Integer c = 200;
        Integer d = 200;
        Integer e = new Integer(100);
        Integer f = Integer.valueOf(100);
        System.out.println(a == b);
        System.out.println(c == d);
        System.out.println(a == e);
        System.out.println(a == f);
    }
```

上面代码运行后输出：
```java
true
false
false
true
```


由于自动装箱机制，在一定数值范围内返回的是同一个对象，在==比较时为true。但如果未命中缓存则会为false。所以比较时应该使用equals或先转成基本类型在进行


##### **空指针问题**

自动拆箱时返回的是包装对象里 的xxxValue的值，如果包装类型为null，那么在拆箱的时候就会抛NPE

```java
    public static void main(String[] args) {
        Integer a = null;
        int b = a;
    }
```


##### **循环内自动装箱问题**

```java
    public static void main(String[] args) {
        Integer num = 0;
        for (int i = 0; i < 5000; i++) {
            num += 1;
        }
    }
```

反编译后代码：

```java
    public static void main(String[] args) {
        Integer num = Integer.valueOf((int)0);
        for (int i = 0; i < 5000; ++i) {
            num = Integer.valueOf((int)(num.intValue() + 1));
        }
    }
```


在循环中自动装箱，相当于循环内创建对象，会造成大量的对象产生，浪费系统资源同时加重垃圾回收的工作。


### **[Integer缓存机制](http://www.hollischuang.com/archives/1174)**

再回到上面 `==比较问题` 中的代码：

```java
    public static void main(String[] args) {
        Integer a = 100;
        Integer b = 100;
        Integer c = 200;
        Integer d = 200;
        System.out.println(a == b);
        System.out.println(c == d);
    }
```

实际执行结果 `a = b` 和 `c != d`。原因就和 Integer 中的缓存机制有关。在JDK1.5中引入缓存来节省使用包装类时内存和提高性能。实现原理是在包装内部有一个静态内部类，在内部类加载时会先创建好一定范围内的对象存到一个数组里。在调用valueOf时，在判断是不是在缓存范围内，如果是直接返回缓存数组里的，否则再新建一个对象。

缓存在第一次使用时被初始化。大小可以由JVM参数-XX:AutoBoxCacheMax=size来指定。JVM初始化时此值被设置成java.lang.Integer.IntegerCache.high属性并作为私有的系统属性保存在sun.misc.vm.class中。

在[Java语言规范规定](https://docs.oracle.com/javase/specs/jls/se8/html/jls-5.html#jls-5.1.7)

- 缓存的范围

>如果一个变量p的值是：
> 
>-128至127之间的整数(§3.10.1)
>
>true 和 false的布尔值 (§3.10.3)
   
>‘\u0000’至 ‘\u007f’之间的字符(§3.10.4)
   
>将p包装成a和b两个对象时，可以直接使用a==b判断a和b的值是否相等。也就是这个范围中的值会使用缓存


- 缓存的对象

缓存行为不仅适用于Integer对象。针对所有的整数类型的类都有类似的缓存机制

- - 有ByteCache用于缓存Byte对象 **有固定范围: -128 到 127**
- - 有ShortCache用于缓存Short对象 **有固定范围: -128 到 127**
- - 有LongCache用于缓存Long对象 **有固定范围: -128 到 127**
- - 有CharacterCache用于缓存Character对象 **范围是 0 到 127**

- 其它约束

除了Integer以外，缓存范围都不能改变