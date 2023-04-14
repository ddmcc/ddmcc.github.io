---
layout: post
title:  "设计模式之观察者模式"
date:   2019-04-16 22:03:16
categories: 设计模式
tags: 设计模式 观察者模式
author: ddmcc
---

## 什么是观察者模式

定义对象间的一对多依赖，当对象的状态发生改变，它的依赖者都会收到通知。

<!-- more -->



## 什么是观察者模式

定义对象间的一对多依赖，当对象的状态发生改变，它的依赖者都会收到通知。


## 如何设计

### 自己实现观察者模式

观察者模式可以看成是报社与报纸订阅者的关系，但有新的报纸时，此报纸的订阅者都会收到新的报纸。
报社(subject主题)，订阅者(observer观察者)

代码如下：

```java
/**
 * 
 * 主题接口
 * 
 */
public interface Subject {

    void addObserver(Observer var);

    void removeObserver(Observer var);

    void notifyObserver();

}
```


观察者接口


```java
/**
 * 
 * 观察者接口
 * 
 */
public interface Observer {

    void update(Object data);
    
}
```


定义报社实现主题接口



```java
public class Newspaper implements Subject {

    private List<Observer> observers;

    private Object data;

    public Newspaper () {
        observers = new ArrayList<>();
    }

    @Override
    public void addObserver(Observer var) {
        if (!observers.contains(var)) {
            observers.add(var);
        }
    }

    @Override
    public void removeObserver(Observer var) {
        if (observers.indexOf(var) > -1) {
            observers.remove(var);
        }
    }

    @Override
    public void notifyObserver() {
        for (Observer observer : observers) {
            observer.update(data);
        }
    }

    // 测试设置值的改变并通知观察者
    public void setData(Object data) {
        this.data = data;
        notifyObserver();
    }
}
```

报社订阅者实现观察者代码



```java
public class Subscriber implements Observer{

    private String name;

    public Subscriber(Subject subject, String name) {
        this.name = name;
        subject.addObserver(this);
    }

    @Override
    public void update(Object data) {
        System.out.println(String.format("%s数据更新 -> %s", this.name, data));
    }
}
```


运行一下



```java
public class Main {

    public static void main(String[] args) {
        Subject subject = new Newspaper();
        new Subscriber(subject, "张三");
        Subscriber subscriber = new Subscriber(subject, "李四");
        new Subscriber(subject, "王五");
        ((Newspaper) subject).setData("123456");

        subject.removeObserver(subscriber);
        ((Newspaper) subject).setData("654321");
    }
}
```

运行结果


![markdown](https://ddmcc-1255635056.file.myqcloud.com/2e6c65fa-7c76-4c3c-944a-35ff058a4253.png)





### 使用Java内置观察者模式

- 内置的观察者模式提供了推/拉数据两种模式，调用`notifyObservers()`时需要观察者自己去拉数据，调用`notifyObservers(Object data)`,
则会推数据。
- 用了内置观察者模式后只剩下了两个类，报社和订阅者。
- 在订阅者数据更新方法中，还会把被观察者通过参数传进来。
- 在报社通知观察者之前需要调用`setChanged();`，确定状态已经改变了，才会去通知观察者。
在源码中，通知之前有做状态标记的判断。这么做有个好处就是，比如报社已经开始印刷报纸了，但是还没印完，
这时是不应该通知订阅者的。需要全部印刷完，再设置状态已改变的标记，才能去通知。

```java
public void notifyObservers(Object arg) {
        /*
         * a temporary array buffer, used as a snapshot of the state of
         * current Observers.
         */
        Object[] arrLocal;

        synchronized (this) {
            /* We don't want the Observer doing callbacks into
             * arbitrary code while holding its own Monitor.
             * The code where we extract each Observable from
             * the Vector and store the state of the Observer
             * needs synchronization, but notifying observers
             * does not (should not).  The worst result of any
             * potential race-condition here is that:
             * 1) a newly-added Observer will miss a
             *   notification in progress
             * 2) a recently unregistered Observer will be
             *   wrongly notified when it doesn't care
             */
            if (!changed)
                return;
            arrLocal = obs.toArray();
            clearChanged();
        }

        for (int i = arrLocal.length-1; i>=0; i--)
            ((Observer)arrLocal[i]).update(this, arg);
    }
```



订阅者代码



```java
public class Subscriber implements Observer {

    private String name;

    public Subscriber(Observable observable, String name) {
        this.name = name;
        observable.addObserver(this);
    }

    /**
     * This method is called whenever the observed object is changed. An
     * application calls an <tt>Observable</tt> object's
     * <code>notifyObservers</code> method to have all the object's
     * observers notified of the change.
     *
     * @param o   the observable object.
     * @param arg an argument passed to the <code>notifyObservers</code>
     */
    @Override
    public void update(Observable o, Object arg) {
        if (o instanceof Newspaper) {
            System.out.println(String.format("%s数据更新 -> %s", this.name, arg));
        }
    }
}
```


主题类代码


```java
public class Newspaper extends Observable {

    public void setData(Object data) {
        // 设置状态已改变
        setChanged();
	// 推数据方式
        notifyObservers(data);
	// 拉数据方式
	//notifyObservers();
    }
}
```


### Java内置观察者模式的问题

- 内置的观察者模式和自己实现的其实是差不多的，只不过内置的主题是个类，而不是接口。所以这也限制了它。
因为Java单继承多实现，所以继承了`Observable`就不能拥有其他类的功能了。并且它把关键方法`setChanged`设为
protected，这意味着你必须继承`Observable`才能创建实例组合到自己的对象中。
- 内置观察者模式违反了OO设计原则，针对接口编程而不是针对实现编程。
- 违反了'多用组合，少用继承'，意思是因为`setChanged`方法，你必须继承`Observable`，才能把实例组合到我们的对象中。


## 模式使用
