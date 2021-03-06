---
layout: post
title:  "如何理解线程安全"
date:   2019-06-24 23:33:17
categories: 并发编程
tags: Thread volatile synchronized
author: ddmcc
---

* content
{:toc}


>线程安全问题都是由全局变量及静态变量引起的。

JVM运行时数据区包括了程序计数器，本地方法栈，jvm栈，堆。在这四个区中，前三个都是线程间隔离的。
只有堆内存是线程间共享的。而全局变量放在堆内存中，各线程内jvm栈只保存了对象引用，所以各线程更改的还是一个
内存地址的数据。

在JDK1.8中元数据区取代了永久代，元数据区并不在虚拟机中，而在本地内存中。静态变量是保存在元数据区中的。所以对于
线程来说，操作的还是同一内存地址上的数据。





>若每个线程中对全局变量、静态变量只有读操作，而无写操作，一般来说，这个全局变量是线程安全的；
>若有多个线程同时执行写操作，一般都需要考虑线程同步，否则的话就可能影响线程安全。

### 可见性问题
jvm有一个主内存，每个线程有自己的高速缓存，当运行时每个线程都会在自己的高速缓存中建立一个变量副本，
操作完后再把值写入到主内存。在多线程的情况下，有可能当线程A操作完，但值还未写入，这时线程B获取时间片在执行变量值还是未改变的。

**线程A改变了值，线程B没有立即看到线程A修改的值这就是可见性问题。**

### 原子性问题 
**即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。**
比如：size += 1 相当于 size = size + 1，这边就有两个操作了，先size+1，再把值赋给size。如果这两个操作不具备原子性，就会造成线程不安全。
在多线程情况下，有可能线程A刚做完size+1的操作，这时被线程B抢走了时间片，而B读到的就是错误的数据。

### 有序性问题
**即程序执行的顺序按照代码的先后顺序执行** ，但是有时候并不是如此。

指令重排序： 
**源代码中的顺序与得到的字节码顺序不一样或者字节码顺序与实际的执行顺序不一样**
在编译的时候静态编译器会.java编译成.class，动态编译器jit会将.class编译成jvm执行的文件，在编译的时候还会对性能进行优化，
这时候就有可能改变了代码的顺序，即源代码和执行代码的顺序是不一致的。


**可以在下面的代码中看到一个简单的示例：**

```java


public class Reordering {
  int x = 0, y = 0;
  public void writer() {
    x = 1;
    y = 2;
  }

  public void reader() {
    int r1 = y; // y的读取
    int r2 = x;
  }
}
```

假设此代码在两个线程中同时执行，并且y的读取看到值2。由于此写入是在写入x之后发生的，因此程序员可能会假设x的读取必须看到值1。但是，写入可能已重新排序。如果发生这种情况，则可能发生对y的写入，随后是两个变量的读取，然后可能发生对x的写入。结果将是r1的值为2，而r2的值为0。

### 如何保证线程安全
- 上面说了线程安全是因为全局变量及静态变量引起的。所以把全局变量和静态变量改成局部变量就不会出现线程安全的问题。
因为局部变量的值保存在jvm栈内存里。
- 用同步锁（synchronized，Lock）可以解决原子性问题，因为线程间是互斥的，所以能保证原子性。
- 用同步锁还可以保证可见性问题，在synchronized关键字中，获得锁后会先清除工作内存的变量副本，然后从主内存拷贝变量的最新副本到线程工作内存，
执行代码，将更改后的最新的共享变量的值刷新到主内存，释放互斥锁。而Lock，在Java并发编程实战中,有句话是"线程A拿到lock对象对数据进行修改, 线程B拿到lock对象后,
对于线程A的修改线程B是可见的, 线程B可以看到线程A的所有操作结果"，也可以保证可见性。
volatile变量也可以保证该变量对所有线程可见，但不能保证原子性。
- 对于有序性volatile可以保证，在volatile变量前的代码不会再其后面执行，再其后面的代码不会再其前面执行，但不能其中的代码重排序。
- Atomic***即可以保证原子性，又可以保证可见性，底层是通过CAS和volatile实现的

### 总结 
线程安全是因为全局变量及静态变量引起的，如果有全局变量或静态变量，并且存在多线程对变量的写操作，则要考虑解决可见性，有序性，有序性问题。


附：

- lock：作用于主内存，把变量标识为线程独占状态。
- unlock：作用于主内存，解除独占状态。
- read：作用主内存，把一个变量的值从主内存传输到线程的工作内存。
- load：作用于工作内存，把read操作传过来的变量值放入工作内存的变量副本中。
- use：作用工作内存，把工作内存当中的一个变量值传给执行引擎。
- assign：作用工作内存，把一个从执行引擎接收到的值赋值给工作内存的变量。
- store：作用于工作内存的变量，把工作内存的一个变量的值传送到主内存中。
- write：作用于主内存的变量，把store操作传来的变量的值放入主内存的变量中



### 参考

http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html