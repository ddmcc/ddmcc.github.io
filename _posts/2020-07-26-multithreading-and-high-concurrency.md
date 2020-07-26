---

layout: post
title:  "多线程与高并发"
date:   2020-07-26 16:40:00
categories: 并发编程
tags:  synchronized cas volatile
author: ddmcc
---

* content
{:toc}






## synchronized 

`synchronized(this)` 和 `synchronized方法` 是一样的，都是锁定当前对象。



```java
// this
public void test() {
  	synchronized(this) {
      	....
    }
}

// 同步方法
public synchronized void test() {
  	....
}

```

---

静态方法没有 **this** 对象，如果是 **静态的同步方法，那么锁对象就是类对象**





``` java
public class T {

    // 	静态同步方法 等同于 synchronized(T.class)
    public synchronized static void test() {
        ....
    }

}
```





### **synchronized可重入**

**可重入锁** 通俗来讲，当线程请求一个由其它线程持有的对象锁时，该线程会阻塞，而当线程请求由自己持有的对象锁时，如果该锁是重入锁，请求就会成功，否则阻塞。

 

事例代码：



```java
package com.soft.ot;

/**
 * @author jiangrz
 */
public class ReentrantLockTest extends SuperReentrantLockTest {

    @Override
    public synchronized void doSomething() {
        System.out.println("doSomething");
        doAnotherThing();
    }

    public void doAnotherThing() {
        super.doSomething();
        System.out.println("doAnotherThing");
    }


    public static void main(String[] args) {
        ReetrantLockTest reetrantLock = new ReentrantLockTest();
				reetrantLock.doSomething();
    }
}


class SuperReentrantLockTest {

    public synchronized void doSomething() {
        System.out.println("super doSomething");
    }

}

```





运行输出：

```java
doSomething
super doSomething
doAnotherThing
```





 在上面的事例中，锁对象只有一个，那就是 **reetrantLock** 对象。当执行 `reetrantLock.doSomething()` 时，该线程获得 **reetrantLock** 对象的锁，在 `doSomething` 方法中调用 `doAnotherThing` 方法时，再次请求 ** reetrantLock** 对象的锁，因为 **synchronized** 是可重入锁，所以可以得到该锁，继续在 `doSomething` 中调用父类的方法时，第三次请求这把锁同样可以得到。如果 **synchronized** 不是可重入锁，那么后面这两次请求会被一直阻塞，从而导致死锁。**同一线程在调用锁对象中其他 synchronized 方法/块或调用父类的 synchronized 方法/块都不会阻碍该线程的执行。就是说同一线程对同一个对象锁是可重入的，而且同一个线程可以获取同一把锁多次，也就是可以多次重入**





### **synchronized原理**



- 早期，底层实现是 **重量级锁** ，需要找系统去申请锁，这就造成效率非常低下
- 改进，后来改进了有了锁升级的概念







