---
layout: post
title:  "设计模式之外观模式"
date:   2019-08-27 21:52:10
categories: 设计模式
tags:  设计模式 外观模式
author: ddmcc
---

* content
{:toc}


## 定义

 　　提供一个统一接口，用来访问一群的子系统接口。外观模式定义了一个高层接口，让子系统更容易使用。


 　　 **在上一个适配者模式中，将一个对象包起来（当然也可以是多个）， 转换其接口。而外观模式则是将对象包起来，简化其接口**。




## 类图


![markdown](https://ddmcc-1255635056.file.myqcloud.com/2e07676c-88e9-4b94-9d1e-78e782364287.png)

---
 　　外观对象中有许多子系统的实例，在外观提供的接口中统一对子系统调用（上图对Sub1和Sub2接口调用），子系统也可能再调用用其它的子系统（Sub1对Sub3和Sub5的接口调用，Sub2对Sub4和Sub5），即使它内部再复杂，对于客户端来说只要调用外观`Facade`的接口就行了，内部是由谁实现的，怎么实现的并不关心。
也就是 **将客户端与子系统的类解耦**。



## 来个栗子

 　　如洗衣机，我们一般洗衣的步骤是`加水 -> 洗涤 -> 漂洗 -> 脱干` 这几个步骤当然可能还会有烘干。如果不是全自动的洗衣机，可能我们把衣服放进去，然后按开始加水的按钮，慢慢等待加水，终于加满了！！
放满后我们再按洗涤的按钮。。。一步步的。。更可怕的是有可能水还没加足我们就按了开始洗涤；水还没放完就按了开始脱干。。。


 　　这时如果将这几个操作封装成一个操作，当加完水后洗衣机自己就会开始洗涤，接着漂洗。。。而且重要的是它知道何时开始脱干！

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/e25843d0-028c-4e52-8df9-75f71da05058.png)
----

ps：这个类图和上面那个不是一软件画的。觉得上面那个太难用了。这个是一个在线网页上画的 [diagrams.visual-paradigm.com](https://diagrams.visual-paradigm.com/#proj=0&type=ClassDiagram)

---
 　　在上面的图中可以看到有一个洗衣机`WashingMachine`，里面有洗涤`Washing`，漂洗`Rinse`，脱干`Dehydration`，自动洗衣`autoWashing`（里面包含前面三个操作）



下面看代码：


先是加水，放水功能

```js
/**
 *
 * 加水功能
 *
 */
public class Water {

    private int water = 0;

    public void turnOn() {
        while (water <= 40) {
            water ++;
            System.out.println(String.format("加水中------->当前水量：%d", water));
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void turnOff() {
        while (water >= 0) {
            water --;
            System.out.println(String.format("放水中------->当前水量：%d", water));
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public int getWater() {
        return water;
    }
}
```
---

洗涤功能


在洗涤之前应该要先加水，所以在洗涤类中应该有放水的功能

```js
/**
 *
 * 洗涤功能
 *
 */
public class Washing {

    public Washing(Water water) {
        this.water = water;
    }

    private Water water;

    public void start() {
        // 洗涤前先加水
        water.turnOn();

        System.out.println("开始洗涤。。。");
        try {
            Thread.sleep(2000);
            stop();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void stop() {
        System.out.println("结束洗涤。。。");
    }
}
```

---

漂洗功能


在漂洗之前应该要先放水，再加水。


```js
/**
 *
 * 漂洗功能
 *
 */
public class Rinse {

    private Water water;

    public Rinse(Water water) {
        this.water = water;
    }

    public void start() {
        // 先放水
        water.turnOff();
        // 加水
        water.turnOn();

        System.out.println("开始漂洗。。。。");

        try {
            Thread.sleep(2000);
            stop();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    public void stop() {
        System.out.println("结束漂洗。。。。");
    }
}
```

---

脱干功能


在脱干之前也应该要先放水


```js
/**
 *
 * 脱干功能
 *
 */
public class Dehydration {

    private Water water;

    public Dehydration(Water water) {
        this.water = water;
    }

    public void start() {
        // 放水
        water.turnOff();

        System.out.println("开始脱干。。。");
        try {
            Thread.sleep(2000);
            stop();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void stop() {
        System.out.println("脱干结束。。。");
    }
}
```

---

洗衣机类

 **这里直接把功能的创建直接放在了洗衣机里，为了体现具体的功能实现和客户端是没有关系的**


```js
/**
 *
 * 洗衣机
 *
 */
public class WashingMachine {

    private static Water water = new Water();

    private static Washing washing = new Washing(water);

    private static Rinse rinse = new Rinse(water);

    private static Dehydration dehydration = new Dehydration(water);
    
    /**
     *
     * 自动
     *
     */
    public void autoWashing() {
        washing.start();
        rinse.start();
        dehydration.start();
    }

    /**
     *
     * 洗涤
     *
     */
    public void doWashing(){
        washing.start();
    }

    /**
     *
     * 漂洗
     *
     */
    public void doRinse() {
        rinse.start();
    }

    /**
     *
     * 脱干
     *
     */
    public void doDehydration() {
        dehydration.start();
    }
}
```

---
最后是调用

```js
public class Main {

    public static void main(String[] args) {
        new WashingMachine().autoWashing();
    }
}
```


 　　就可以看到洗衣机自动的就完成了洗涤，漂白，脱干的功能，而不用一个一个去操作，等待，对于使用者来说更加的方便了，懒人福音！而且我们如果只想要洗涤或脱干任一功能也是可以的。


## 总结
 　　 **最少知识原则**：尽量减少类之间的耦合。意思就是如果一个类依赖了太多的类，那么系统会变得很复杂，而且修改其中一部分会影响到其他部分。比如有A，B，C三个类，如果能做到A依赖B，B依赖C，那么绝不要让A直接依赖B和C。
如上面的洗衣机，对于客户端来说，只和洗衣机有关系。即使洗衣机内部再怎么修改，也不会影响到客户端。


 　　 **适配器模式** 将一个对象包装起来，改变其接口，以符合客户期望； **外观模式** 将一群对象包起来，委托子系统们去执行，简化接口； **装饰者模式** 将一个对象包装起来，增加新行为或责任


- `不要试图通过外观类为子系统增加新行为`：外观模式的用意是集合子系统，简化接口。如果要新增新的行为，正确的做法是修改现有的子系统或增加新的子系统。 
- `引入抽象外观`：如果有新的子系统增加或删除子系统，那么势必会修改现有的外观。可以做的是客户端针对抽象外观进行编程，有的子系统更改则可以
 创建新的外观实现类。

 
## 优缺点

#### 缺点

- 增加新的子系统可能需要修改外观类或客户端的代码，违背了`开闭原则`。


#### 优点

- 客户端与多个子系统之间存在很大的依赖性，引入外观类将子系统与客户端以及其他子系统解耦，可以提高子系统的独立性和可移植性。


## 其他

感觉外观模式和命令模式的宏命令挺像的，都可以执行一组"命令"。不过命令模式将动作和执行者封装成一个命令，将请求发送者和执行者解耦。外观模式也可以看成是发送者和真正的执行者解耦。