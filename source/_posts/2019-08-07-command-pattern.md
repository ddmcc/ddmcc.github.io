---
layout: post
title:  "设计模式之命令模式"
date:   2019-08-07 15:11:26
categories: 设计模式
tags:  设计模式 命令模式
toc: true
---


## 定义

将一个请求封装为一个对象，从而使我们可用不同的请求对客户进行参数化；可以宏命令操作，对请求进行排队或者记录请求日志，以及支持可撤销的操作

<!-- more -->


## 使用场景


- 对请求发送者和请求接收者解耦，使得调用者和接收者不直接交互，而是通过请求对象
- 在某些场合，需要对请求进行记录，撤销/重做等处理，或者需要将多个请求组合在一起，即宏命令
- 对请求进行排队执行

## 模式结构


![markdown](https://ddmcc-1255635056.file.myqcloud.com/9c95864c-be47-4e3f-9868-8047cc288279.png)
----

在客户端生成命令对象，调用调用者对象`Invoker`中的**setCommand()**方法，设置命令对象（命令对象可以是一个或多个，命令也可以不执行或记录下来执行多次），在某个时间点再调用命令对象中的`execute()`方法执行命令，
这将导致调用接受者`receiver`中的具体操作。

**具体也就是把Invoker和Receiver解耦，他们之间没有直接的引用关系，将请求发送者与请求处理中责任分开。**

![markdown](https://ddmcc-1255635056.file.myqcloud.com/9d53b538-c77d-4d76-850b-81a2973cba1a.png)
---

时序图（网上扣得）

![markdown](https://ddmcc-1255635056.file.myqcloud.com/0c4d370b-4982-4175-b31a-c9fa3109d0a6.png)

## 使用栗子

假设有一个餐厅，里面有中餐（`宫保鸡丁`）和西餐（`汉堡`），中餐由中餐厨师负责，西餐由西餐厨师负责，点单由前台小妹负责。

在以往的餐厅流程可能是这样的：

客人点菜 -> 菜单给前台小妹 -> 前台小妹根据菜去通知不同厨师 -> 厨师煮 -> 出菜

引入了命令模式后把菜单封装成命令对象，前台小妹不需要自己去通知厨师，只需开单即可！**不用关注是谁处理了这个订单，因为这个定单已经有了负责的厨师。**

客人点菜 -> 菜单（命令对象）给前台小妹 -> 前台小妹开单（调用订单orderUp()方法） -> 厨师煮 -> 出菜

---
#### 类图

![markdown](https://ddmcc-1255635056.file.myqcloud.com/fac21ee6-3e6a-41d6-9aa7-5cd8c2a40512.png)

---
#### 代码

厨师接口与具体的厨师，即命令的接受者

```java
public interface Chef {
    void doCook(String name);
}


public class ChineseChef implements Chef {

    private String name;

    public ChineseChef(String name) {
        this.name = name;
    }

    @Override
    public void doCook(String foodName) {
        System.out.println(String.format("中餐厨师：%s 正在烹饪 %s", name, foodName));
    }
}



public class WesternChef implements Chef {

    private String name;

    public WesternChef(String name) {
        this.name = name;
    }

    @Override
    public void doCook(String foodName) {
        System.out.println(String.format("西餐厨师：%s 正在烹饪 %s", name, foodName));
    }
}
```

---
订单接口与具体的菜品，即封装的命令对象

```java
public interface Order {

    void execute();

}


public class Hamburger implements Order {

    private Chef chef;

    public Hamburger(WesternChef westernChef) {
        this.chef = westernChef;
    }

    @Override
    public void execute() {
        chef.doCook("汉堡");
    }
}


public class KungPaoChicken implements Order {

    private Chef chef;

    public KungPaoChicken(ChineseChef chineseChef) {
        this.chef = chineseChef;
    }

    @Override
    public void execute() {
        chef.doCook("宫保鸡丁");
    }
}
```

--- 
前台小妹Waiter，她是订单的发送者，但是她并不知道订单的内容是什么，到底由哪个厨师来处理。只知道调用订单里的`execute`就可以了。

通过封装订单，来减少前台小妹和厨师的接触机会，因为她并不喜欢油腻大叔。


```java
public class Waiter {

    private String name;

    public Waiter(String name) {
        this.name = name;
    }

    public void takeOrder(Order newOrder) {
        System.out.println(name +" 接到新的订单");
        newOrder.execute();
    }


    public void orderUp(Order newOrder) {
        System.out.println(name +" 来新订单了！！");
        newOrder.execute();
    }
}
```

---
调用


```java
public static void main(String[] args) {
    Waiter waiter = new Waiter("菜花");
    Order hamburger = new Hamburger(new WesternChef("John"));
    Order chicken = new KungPaoChicken(new ChineseChef("大锤"));

    waiter.takeOrder(hamburger);
    waiter.takeOrder(chicken);
    // 输出
    菜花 接到新的订单
    西餐厨师：John 正在烹饪 汉堡
    菜花 接到新的订单
    中餐厨师：大锤 正在烹饪 宫保鸡丁
}
```


**在实际开发中，命令不一样要在处理者中处理（厨师），可以直接在封装的命令对象中自己处理。**


--- 

上面一个订单只能点一个菜，如何客人需要点多个呢？

对！没错！！封装一个支持多个命令的宏命令对象！

```java
public class MacroOrder implements Order {

    private Order[] orders;

    public MacroOrder(Order[] orders) {
        this.orders = orders;
    }

    @Override
    public void execute() {
        for (Order order : orders) {
            order.execute();
        }
    }
}
```

在宏命令对象中有一个对象数组用于装载所有的订单，然后循环执行`execute`。

调用

```java
public class Main {

    public static void main(String[] args) {
        Waiter waiter = new Waiter("菜花");
        Order hamburger = new Hamburger(new WesternChef("John"));
        Order chicken = new KungPaoChicken(new ChineseChef("大锤"));

        Order macroOrder = new MacroOrder(new Order[]{hamburger,chicken});
        waiter.takeOrder(macroOrder);
	// 输出
	菜花 接到新的订单
        西餐厨师：John 正在烹饪 汉堡
        中餐厨师：大锤 正在烹饪 宫保鸡丁
    }
}
```


宏命令也是一个具体命令，不过它包含了对其他命令对象的引用，在调用宏命令的execute()方法时，将递归调用它所包含的每个成员命令的execute()方法，一个宏命令的成员对象可以是简单命令，还可以继续是宏命令。
执行一个宏命令将执行多个具体命令，从而实现对命令的批处理。

---

一会儿有两个客人同时来吃饭了，这时前台小妹可以先招待玩一位客人点完菜，再招待另一位点菜，再一起处理。
但是必须保证先来的客人的菜先做！

这时就可以通过队列来保证命令按顺序执行。

```java
public class Waiter {

    private String name;

    private Queue<Order> orders;

    private int size;

    public Waiter(String name) {
        this.name = name;
    }

    public Waiter takeOrder(Order newOrder) {
        System.out.println(name +" 接到新的订单");
        if (orders == null) {
            orders = new LinkedBlockingDeque<>();
        }
        orders.add(newOrder);
        size ++;
        return this;
    }

    public void orderUp() {
        while (size > 0) {
            Objects.requireNonNull(orders.poll()).execute();
            size --;
        }
    }
}
```

假设客人A先点了宫保鸡丁,B客人后点了汉堡

```java
public static void main(String[] args) {
    Waiter waiter = new Waiter("菜花");
    Order hamburger = new Hamburger(new WesternChef("John"));
    Order chicken = new KungPaoChicken(new ChineseChef("大锤"));

    waiter.takeOrder(chicken).takeOrder(hamburger).orderUp();
    // 输出
    菜花 接到新的订单
    菜花 接到新的订单
    中餐厨师：大锤 正在烹饪 宫保鸡丁
    西餐厨师：John 正在烹饪 汉堡
}
```


---

一位客人一不下心点了两个汉堡，他想退了一个。所以需要提供一个撤回订单的功能（当然要在未煮的情况下）

所以在前台小妹`Waiter`中加了撤回订单的功能！

```java
public boolean revoke(Order order) {
    System.out.println(name +" 准备为客人撤回订单");
    if (size > 0 && orders.remove(order)) {
        size --;
        return true;
    }
    return false;
}
```

调用

```java
 public static void main(String[] args) {
    Waiter waiter = new Waiter("菜花");
    Order hamburger = new Hamburger(new WesternChef("John"));
    Order hamburger1 = new Hamburger(new WesternChef("John"));
    waiter.takeOrder(hamburger1).takeOrder(hamburger);
    System.out.println(waiter.revoke(hamburger1)? "订单撤回成功" : "订单撤回失败");
    waiter.orderUp();
    System.out.println(waiter.revoke(hamburger)? "订单撤回成功" : "订单撤回失败");

    // 输出
    菜花 接到新的订单
    菜花 接到新的订单
    菜花 准备为客人撤回订单
    订单撤回成功
    西餐厨师：John 正在烹饪 汉堡
    菜花 准备为客人撤回订单
    订单撤回失败
}
````

---
>>>>>>>>>>>>>>>>请求日志功能
---

## 总结

- 在命令模式中，将一个请求封装为一个对象，从而使我们可用不同的请求对客户进行参数化；对请求排队或者记录请求日志，以及支持可撤销的操作。

- 命令模式包含四个角色：
    - 抽象命令类中声明了用于执行请求的execute()等方法，通过这些方法可以调用请求接收者的相关操作；
    - 具体命令类是抽象命令类的子类，实现了在抽象命令类中声明的方法，它对应具体的接收者对象，将接收者对象绑定其中，并由它来对接收者的调用；
    - 调用者即请求的发送者，又称为请求者，它通过命令对象来执行请求；
    - 接收者执行与请求相关的操作，它具体实现对请求的业务处理。

- 命令模式的本质是对命令进行封装，将发出命令的责任和执行命令的责任分割开。命令模式使请求本身成为一个对象。

- 命令模式适用情况包括：
    - 需要将请求调用者和请求接收者解耦，使得调用者和接收者不直接交互；
    - 需要在不同的时间指定请求、将请求排队和执行请求；
    - 需要支持命令的撤销操作和恢复操作，需要将一组操作组合在一起，即支持宏命令。


## 优缺点

#### 优点

- 命令模式的主要优点在于降低系统的耦合度，将请求调用者和请求接收者解耦。调用者无需关注具体怎么处理。
- 增加新的命令很方便，而且可以比较容易地设计一个命令队列和宏命令，并方便地实现对请求的撤销和恢复；

#### 缺点
- 主要缺点在于可能会导致某些系统有过多的具体命令类。