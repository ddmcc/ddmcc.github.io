---
layout: post
title:  "饿汉单例真的是空间换时间吗？"
date:   2021-07-13 16:56:23
categories: jvm
tags:  jvm 设计模式
author: ddmcc
---

* content
{:toc}




## 饿汉、懒汉单例模式

先复习回顾一下这两种单例的写法：


#### **饿汉模式**

```java
public class SingletonPatternTest1 {

    private static SingletonPatternTest1 instance = new SingletonPatternTest1();

    private SingletonPatternTest1() { }

    public static SingletonPatternTest1 getInstance() {
	    return instance;
    }

}
```


#### **懒汉模式**

```java
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
```


通常我们对这两种写法的认识是：**懒汉式在类加载阶段不会初始化单例，在调用 `getInstance()` 时，对象才会被创建。而~~饿汉式单例在类加载阶段就已经初始化了，典型的空间换时间...~~**

---

#### **饿汉单例在类加载阶段真的会被初始化吗？**

>先说答案：不会

我们先复习一下类加载过程：包括`加载`、`验证`、`准备`、`解析`、`初始化`、`使用` 和 `卸载` 阶段

---

![](https://ddmcc-1255635056.file.myqcloud.com/eeaf25b4-5baa-43f9-8b2d-87d60e572026.png)

---

其中在 **准备阶段** 会给类变量（即静态变量）初始化 **零值** ，如上面示例中的 `instance` 为引用类型，所以会被初始化为 `null`。类变量 `显示初始化` 工作是在 **初始化阶段** 
 `clinit` 方法中， 顺序完成父类子类静态成员变量显示初始化和父类子类静态代码块语句

>特殊情况：对于被 static final 修饰的基本数据类型常量，会在 “准备” 阶段显示初始化


所以 **不管饿汉、懒汉单例在准备阶段都不会被显示初始化，实例化工作都是在初始化阶段完成的**，那么这两种方式又在什么时候会被显示初始化呢？（什么时候执行初始化阶段）


对于 [初始化阶段](http://ddmcc.cn/2021/05/29/jvm-class-file-loading-process/#%E5%88%9D%E5%A7%8B%E5%8C%96initialization) ，虚拟机严格规范了有且只有 `5` 种情况下，必须对类进行初始化(只有主动去使用类才会初始化类)：

- 当 jvm 执行 new 指令时会初始化类。即当程序创建一个类的实例对象

- 当 jvm 执行 getstatic 指令时会初始化类。即程序访问类的静态变量(不是静态常量，常量会被加载到运行时常量池)

- 当 jvm 执行 putstatic 指令时会初始化类。即程序给类的静态变量赋值

- 当 jvm 执行 invokestatic 指令时会初始化类。即程序调用类的静态方法

- 使用 `java.lang.reflect` 包的方法对类进行反射调用时如 `Class.forname("...")`, `newInstance()` 等等。如果类没初始化，需要触发其初始化

> 初始化一个类，如果其父类还未初始化，则先触发该父类的初始化。
>
> 当虚拟机启动时，用户需要定义一个要执行的主类 (包含 main 方法的那个类)，虚拟机会先初始化这个类。
>
>当一个接口中定义了 JDK8 新加入的默认方法（被 default 关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化


## **结论**

也就是说 **只有在第一次被调用的时候，单例对象才会被显示初始化（new SingletonPatternTest1()），在这种情况下饿汉和懒汉是没什么区别的（懒汉还需要判断、加锁）**。所以不存在空间换时间的说法，因为饿汉的单例对象也没创建


## **特殊情况**

在单例模式中虽然大多数都是调用 `getInstance` 方法，但是也不能确定有其他的方式（或者调用其他的静态方法）导致类执行显示初始化。这时候懒汉模式对于饿汉就有懒加载的优势


## **其它**

最后还是推荐静态内部类单例的写法，在未调用 `getInstance` 时，内部类不会被加载内存，实例对象自然也不会被实例化！并且它是 [线程安全的](https://blog.csdn.net/qq_35590091/article/details/107348114)

>对于 clinit 方法的调用，虚拟机会自己确保其在多线程环境中的安全性。因为 <clinit> () 方法是带锁线程安全

```java
public class SingletonPattenTest {

	private SingletonPattenTest(){}

	private static class SingletonHolder {
		private static SingletonPattenTest instance = new SingletonPattenTest();
	}

	public static SingletonPattenTest getInstance(){
		return SingletonHolder.instance;
	}
}
```