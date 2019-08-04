---
layout: post
title:  "设计模式之装饰者模式"
date:   2019-08-03 16:50:22
categories: 设计模式
tags: 设计模式 装饰者模式
author: ddmcc
---

* content
{:toc}


## 定义


**动态的**扩展对象的功能，是继承的一种替代方案。


## 为什么要用装饰者模式

我们如果需要扩展一个类的功能，通常我们会有两种做法。





- 直接在类里面增加

- 创建一个新的类来继承这个类，扩展的功能由新的类来实现

对于第一种方式，我是拒绝的。首先它违背了 **开闭原则**，一个类对扩展是开放的，对修改是关闭的（注：对修改是关闭的不是说不让修改，而是在
不会在影响现有功能代码的情况下修改），所以它可能会对原有的代码产生影响，并且会让类里的代码变得更加臃肿。

对于用继承的方法，它解决了开闭原则问题。但是继承它是在编译的时候就已经决定好的，也就是这个类的行为与属性不能动态的修改。但是对于
装饰者模式，它利用组合的方式来扩展。在我们编写代码时可以动态的更改我们想要组合的对象，即动态更改我们相要扩展的功能。
**多用组合，少用继承** 也是我们程序设计的原则。


## UML类图


![](https://i.loli.net/2019/08/03/NhvUZtnB4J8pTu6.png)


#### 疑问

![](https://i.loli.net/2019/08/04/kIQ2heNiwWXy5PS.jpg)

看你这类图不也用了继承了么！

**注意：**这里继承不是要继承父类的行为，而是为了它们有相同的超类型。装饰者与被装饰者都依赖于组件接口。这就是 **依赖倒置原则**

等等..为什么装饰者要继承超类？

这是因为如果装饰者不继承超类，装饰者与被装饰者没有公共的超类类型，尽管装饰者实例中有被装饰者的实例，但这样一来每个被装饰者只能够被装饰一次了！！

这要怎么理解呢？？

↓ ↓ ↓ ↓ ↓  接着往下看

## 使用场景

比如有个家具代理公司，卖各种家具。支持客户自由的挑选家具组合。现在需要设计一个收银系统，能够计算不同家具组合下的总价。

假设有房间家具要购买。有床，衣柜，书桌。


#### UML类图

![](https://i.loli.net/2019/08/03/39K2RWqu1sGNziV.png)


#### 创建代码

1，共同的组件接口`Furniture`，里面有属性`name`用来记录家具的名字，还有`cost()`方法，这是计算价格的方法，由子类自己去实现。

```java
public abstract class Furniture {

    protected String name;

    protected abstract double cost();

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

2，创建普通的床，衣柜，书桌，注意这三个类就是被装饰者，即使它们不被装饰，也是可用的。

在cost方法中返回自己的价格。

```java
public class Bed extends Furniture {

    public Bed() {
        name = "床";
    }

    @Override
    protected double cost() {
        return 500;
    }
}


public class Desk extends Furniture {

    public Desk() {
        name = "书桌";
    }

    @Override
    protected double cost() {
        return 200;
    }
}


public class Wardrobe extends Furniture {

    public Wardrobe() {
        name = "衣柜";
    }

    @Override
    protected double cost() {
        return 300;
    }
}
```


3，接口和被装饰者已经创建好了，该创建装饰者了。

原料装饰与品牌装饰：

	Decorator
	  |
	  |--- MaterialDecorator  // 原料装饰
	  |      |--- SolidWoodFurniture  // 实木家具
	  |	 |--- MahoganyFurniture  // 红木家具
	  |
	  |--- TrademarkDecorator  // 品牌装饰
		 |--- IKEAFurniture  // 宜家
		 |--- QYFurniture  // 全友
	


```java
public abstract class MaterialDecorator extends Furniture {

    protected Furniture wrappedObj;

    @Override
    protected abstract double cost();

    public MaterialDecorator(Furniture wrappedObj) {
        this.wrappedObj = wrappedObj;
    }
}



public class SolidWoodFurniture extends MaterialDecorator {

    public SolidWoodFurniture(Furniture component) {
        super(component);
    }

    @Override
    protected double cost() {
        // 实木家具+500
        return this.wrappedObj.cost() + 500;
    }

    @Override
    public String getName() {
        return this.wrappedObj.getName() + ",实木";
    }
}



public class MahoganyFurniture extends MaterialDecorator {

    public MahoganyFurniture(Furniture component) {
        super(component);
    }

    @Override
    protected double cost() {
        // 红木家具 + 2000
        return this.wrappedObj.cost() + 2000;
    }

    @Override
    public String getName() {
        return this.wrappedObj.getName() + ",红木";
    }
}


public abstract class TrademarkDecorator extends Furniture {

    protected Furniture wrappedObj;

    @Override
    protected abstract double cost();

    public TrademarkDecorator(Furniture wrappedObj) {
        this.wrappedObj = wrappedObj;
    }
}


public class QYFurniture extends TrademarkDecorator {

    public QYFurniture(Furniture component) {
        super(component);
    }

    @Override
    protected double cost() {
        // 全友家居品牌+ 300
        return this.wrappedObj.cost() + 300;
    }

    @Override
    public String getName() {
        return this.wrappedObj.getName() + ",全友";
    }
}


public class IKEAFurniture extends TrademarkDecorator {

    public IKEAFurniture(Furniture component) {
        super(component);
    }

    @Override
    protected double cost() {
        // 打上宜家招牌 价格+1000
        return this.wrappedObj.cost() + 1000;
    }

    @Override
    public String getName() {
        return this.wrappedObj.getName() + ",宜家";
    }
}
```


4，测试运行，我想要买一个宜家的实木床

```java
public class Main {

    public static void main(String[] args) {
        Furniture furniture = new SolidWoodFurniture(new IKEAFurniture(new Bed()));
        System.out.println(furniture.getName());
        System.out.println(furniture.cost());
	// 输出
	床,宜家,实木
	2000.0
    }
}
```

#### 解决疑问

上面我们说到装饰者也继承了家具超类的问题。我们来看一下，如果我要一个宜家的桌子。那么代码可能如下：

Furniture IKEADesk = new IKEAFurniture(new Desk());

上面是装饰者继承了`Furniture`的情况下的代码，用超类类型变量来接受桌子。如果没有继承的情况下，那么代码就是：

IKEAFurniture IKEADesk = new IKEAFurniture(new Desk());

看了看这个普通的宜家桌子不太喜欢，我想要这个桌子是红木的！！

![](https://i.loli.net/2019/08/04/5jVBMY4U1zsuCec.jpg)

很简单，我们只需要在包装一层，包装层红木的：`Furniture desk = new MahoganyFurniture(IKEADesk);` 。嗯，的确很简单。
但是如果没有继承，`IKEAFurniture`实例并不是一个可以被装饰的对象。也就无法组合成宜家的红木桌子了。

所以可以说 **装饰者继承组件就是为了能够被装饰多次，也可以说是能让装饰者也能成为被修饰者。**


#### 组合替换继承

装饰者模式中利用了组合的方式，在不更改原有类的基础上动态的去修改，增加类的功能。如果要用继承实现，上面的例子。那么
需要更多的类。

上面其实我们只创建了4个具体的装饰者，就能实现他们之间的随意组合。而如果用继承的方式则需要4*4也就是创建16个类。如果品类多的话，无疑
是个类爆炸。并且可能出来后期修改一个功能，会要在很多类中修改。

![](https://i.loli.net/2019/08/04/fPmReJyraNt2YLQ.jpg)



## JDK中的栗子

![](https://i.loli.net/2019/08/04/63aXuDk7i2AbWef.jpg)


这是IO流的部分类图，`inputStream`组件提供了基本的字节读写功能。`FilterInputStream` 是抽象的装饰者，它下面的子类是有具体实现的装饰者。

如LineNumberInputStream提供了行数的统计功能，BufferedInputStream利用缓冲来提高性能和提供readLine()一次读取一行来增强接口。

编写一个自己的I/O装饰者：

继承`FilterInputStream`，并拓展功能，把所有输入的大写字母转为小写。

```java
public class LowerCaseInputStream extends FilterInputStream {

    protected LowerCaseInputStream(InputStream in) {
        super(in);
    }

    @Override
    public int read() throws IOException {
        int c = super.read();
        return (c == -1 ? c : Character.toLowerCase((char)c));
    }
}
```

运行：

```java
public static void main(String[] args) {
    try {
        int c;
        InputStream in = new LowerCaseInputStream(new BufferedInputStream(new FileInputStream("C:\\Users\\Administrator\\Desktop\\text.txt")));
        while ((c = in.read()) >= 0) {
            System.out.print((char)c);
        }
        in.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```


## 优缺点

优点：

- 提供比继承更加灵活的扩展方式。利用组合和委托能动态的加上新的行为。继承是编译期就已确定的，静态的。
- 可以在被装饰者的行为前后加上自己的行为，也可以完全取代。
- 能够利用不同的装饰器和调整它们的排序，设计出不同的组合。

缺点：

- 利用装饰者模式可以比利用继承来创建更少的类，但在使用时会有更多的小对象。特别是在复杂的时候，一层装饰一层，不易排查错误。
- 如果设计了很多的装饰者，产生大量的小类，会对使用者产生困扰。如Java I/O流
