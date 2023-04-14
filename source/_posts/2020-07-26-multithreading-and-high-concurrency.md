---
layout: post
title:  "多线程与高并发"
date:   2020-07-26 16:40:00
categories: 并发编程
tags:  synchronized cas volatile
---

`synchronized` 锁的是对象而不是代码，`synchronized(this)` 和 `synchronized方法` 是一样的，都是锁定当前对象。锁升级从偏向锁到自旋锁再到重量级锁...

<!-- more -->

## synchronized 

`synchronized` 锁的是对象而不是代码，`synchronized(this)` 和 `synchronized方法` 是一样的，都是锁定当前对象。锁升级从偏向锁到自旋锁再到重量级锁。



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



---

运行输出：

```java
doSomething
super doSomething
doAnotherThing
```



---

 在上面的事例中，锁对象只有一个，那就是 **reetrantLock** 对象。当执行 `reetrantLock.doSomething()` 时，该线程获得 **reetrantLock** 对象的锁，在 `doSomething` 方法中调用 `doAnotherThing` 方法时，再次请求 ** reetrantLock** 对象的锁，因为 **synchronized** 是可重入锁，所以可以得到该锁，继续在 `doSomething` 中调用父类的方法时，第三次请求这把锁同样可以得到。如果 **synchronized** 不是可重入锁，那么后面这两次请求会被一直阻塞，从而导致死锁。**同一线程在调用锁对象中其他 synchronized 方法/块或调用父类的 synchronized 方法/块都不会阻碍该线程的执行。就是说同一线程对同一个对象锁是可重入的，而且同一个线程可以获取同一把锁多次，也就是可以多次重入**





### **synchronized原理**



- 早期，底层实现是 **重量级锁** ，需要找系统去申请锁，这就造成效率非常低下

- 改进，后来改进了有了锁升级的概念。

  第一个去访问某把锁的线程先在锁对象的头上面记录一下这个线程（如果只有一个线程访问的时候，实际没有给锁对象加锁，只是记录一下这个线程的ID（**偏向锁**））

  偏向锁如果有线程争用的话，就升级为 **自旋锁** 。也就是后面来的线程不会到cpu就绪队列里去，而是进行自旋操作，等待着占用的cpu，等待锁释放。

  在自旋10次（jdk1.6）或有一定等待的数量线程（超过cpu内核数的一半）之后，升级为 **重量级锁**。



### synchronized 锁对象不能用String









## volatile



#### volatile 有什么用？

总结一句话是：`volatile` 保证线程的可见性，同时防止指令重新排序。



说得详细点就是：对 `volatile` 的写具有与锁释放相同的效果，它可以确保在对变量赋值之后将其从高速缓存中刷新到主内存，以便变量的值立即对其它线程可见；同样的，对`volatile` 的读具有与获得锁相同的存储效果，在读取 `volatile` 变量之前，会高速缓存中的变量值失效，以便重新去主内存中获取值，而不是缓存中的。`volatile` 变量不能相互重新排序，并且对前后的非volatile也进行严格的限制，在线程A对 `volatile` 变量 f 进行赋值时所有可见的内容，在线程B读取 f 时都可见。



简单事例：



```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  
  public void writer() {
    x = 42;
    v = true;
  }

  public void reader() {
    if (v == true) {
      //uses x - guaranteed to see 42.
    }
  }
}
```

---



假定一个线程正在调用 `writer` ，而另一个线程正在调用 `reader` 。在 writer 中将true赋值给 v ，如果 reader 读取到的 v 为 true 那么此时读取x的值必然是 42。这是因为 v 被 **volatile** 修饰了，如果v没被volatile修饰，则编译器可以对 `writer` 的写入进行重排序，而reader对x的读取可能会为0。



jdk官方形容说： **“volatile 的语义几乎达到了同步的水平。出于可见性目的，对易失性字段的每次读取或写入都像“半”同步一样”** 



#### 重新排序是什么意思？

在许多情况下，对变量的赋值，访问与程序指定的顺序是不一致的。编译器会以 “优化” 的名义对指令进行重新排序。比如



---

```
class Reordering {
  int x = 0, y = 0;
  public void writer() {
    x = 1;
    y = 2;
  }

  public void reader() {
    int r1 = y; // 读取到值为2
    int r2 = x;
  }
}
```

---

假设此代码在两个线程中同时执行，并且y的读取看到值2。由于x的赋值在y之后，所以会认为x的值必定为1。但是，写入可能已重新排序。如果发生这种情况，则可能发生对y的写入，随后是两个变量的读取，然后可能发生对x的写入。结果将是r1的值为2，而r2的值为0。









