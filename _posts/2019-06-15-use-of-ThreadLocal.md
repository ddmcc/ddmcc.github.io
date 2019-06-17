---
layout: post
title:  "ThreadLocal的使用与源码"
date:   2019-06-15 20:24:01
categories: 并发编程
tags: ThreadLocal
author: ddmcc
---

* content
{:toc}




## 使用

当有一个单例类中有实例变量，而业务逻辑又要对变量进行处理，当有多个线程同时操作时，如果没有给处理代码加上锁，就有可能出现线程安全问题。如：我们最常见的获取JDBC连接的
连接，还有我们交给Spring容器管理的类等。

```java
public static void main(String[] args) throws InterruptedException {
    // 假设此时有两个线程 main代表线程a thread代表线程b，当线程a执行到设置完name的值，这时线程b拿到了cpu的时间片，执行setName
    // 这是线程b的name也变成了线程b的name，因为Test对象是单例，两个线程操作的是同一个对象。

    Test test = Test.getInstance();
    System.out.println(Thread.currentThread().getName());
    test.setName(Thread.currentThread().getName());
    Thread thread = new Thread(() -> {
        Test test1 = Test.getInstance();
        test1.setName(Thread.currentThread().getName());
        System.out.println(test1.getName());
    });
    thread.start();
    thread.join();
    System.out.println(test.getName());
}

// 一个单例类
public class Test {

    private String name;

    private Test() {

    }

    private static Test test = null;

    public static Test getInstance() {
        if (null == test) {
		test = new Test();
	}
	return test;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }		
	
}
```

输出结果为：


![](http://ws3.sinaimg.cn/large/005BYqpggy1g4274s666fj30wz0cqjs2.jpg)



在上面的Test单例的，就如我们交给Spring管理的Bean， **当有两个线程同时操作时，其中一个先操作修改了，刚好另一个线程拿到cpu时间片，就会引发线程安全问题。**

所以我们可能需要在修改数据的地方加上同步锁，但这样性能又不好。这时就可以使用ThreadLocal了。

```java
public class Test {

    private ThreadLocal<String> name = new ThreadLocal<>();

    private Test() {

    }

    private static Test test = null;

    public static Test getInstance() {
        if (null == test) {
            test = new Test();
        }
        return test;
    }

    public String getName() {
        return name.get();
    }

    public void setName(String name) {
        this.name.set(name);
    }
}
```

再运行：

![](http://ws3.sinaimg.cn/large/005BYqpggy1g4286ow803j315g0buaar.jpg)


发现两个线程已经互不影响了，即使线程a设置了自己线程名，b线程输出的还是b线程名。


## 源码

待续。。。

