---
layout: post
title:  "设计模式之单例模式"
date:   2019-07-17 22:30:03
categories: 设计模式
tags: 设计模式 单例模式
author: ddmcc
---

* content
{:toc}


## 什么是单例模式

确保一个类只有一个实例，并提供一个全局访问点。





## 如何设计

#### 饿汉模式

在类加载的时候就先创建类实例，任务线程访问的时候直接返回实例。

	public class SingletonPatternTest1 {

	    private static SingletonPatternTest1 instance = new SingletonPatternTest1();

	    private SingletonPatternTest1() { }

	    public static SingletonPatternTest1 getInstance() {
		return instance;
	    }

	}


这么做的好处是它是 **线程安全的** ，并且确保了只有一个实例存在。缺点是如果没有用到这个实例，这个实例也会被
创建，浪费资源。

#### 懒汉模式

延迟实例化。先不创建实例，在访问获取实例时，在判断是否已经创建。

	public class SingletonPatternTest1 {

	    private static SingletonPatternTest1 instance;

	    private SingletonPatternTest1() { }

	    public static SingletonPatternTest1 getInstance() {
		if (instance == null) {
		    instance = new SingletonPatternTest1();
		}
		return instance;
	    }
	}


它的优点是在第一次访问的时候才会被创建，避免了创建了没有使用而浪费资源的问题。但是每次访问都需要判断是否创建，
也会影响性能。而且在 **多线程的情况下**，它 **并不是线程安全的**。有可能会产生不同的对象。



#### 懒汉同步式

通过增加synchronized关键字，使得getInstance方法成同步方法，同时只能一个线程能访问。

	public class SingletonPatternTest1 {

	    private static SingletonPatternTest1 instance;

	    private SingletonPatternTest1() { }

	    public static synchronized SingletonPatternTest1 getInstance() {
		if (instance == null) {
		    instance = new SingletonPatternTest1();
		}
		return instance;
	    }
	}


确保了每次只有一个线程进入方法，解决了线程安全的问题。但因为是同步方法，所以会很影响性能。而且我们只需要确保第一次访问的时候不被重复创建实例，
在第一次创建之后，同步方法就成了累赘了。

#### 双重检查加锁

先判断判断实例是否已经创建，否则在进行加锁。保证了只有第一次才会进行同步。

	public class SingletonPatternTest1 {

	    private static volatile SingletonPatternTest1 instance;

	    private SingletonPatternTest1() { }

	    public static SingletonPatternTest1 getInstance() {
		if (instance == null) {
		    synchronized (SingletonPatternTest1.class) {
			if (instance == null) {
			    instance = new SingletonPatternTest1();
			}
		    }
		}
		return instance;
	    }
	}


>这里有个疑问就是：在设计模式书上说，没有volatile会使'双重检查加锁'失效。（原话是JDK1.4或更早的volatile实现会失效）
我们知道volatile是保证了可见性，synchronized关键字不是已经保证了可见性了吗？


#### 注册表式
待续...

#### 枚举式