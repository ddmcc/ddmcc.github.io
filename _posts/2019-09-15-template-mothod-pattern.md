---
layout: post
title:  "设计模式之模板方法模式"
date:   2019-09-15 22:44:29
categories: 设计模式
tags:  设计模式 模板方法模式
author: ddmcc
---

* content
{:toc}


## 定义

 　　在一个方法中定义一个算法的骨架，将一些步骤由子类来实现。模版方法使得子类可以在不改变算法结构的情况下，重复定义算法中的步骤。

 　　**在父类中定义一个模版方法，里面去调用其它的方法，如果被调用的方法是可以复用的话，则可以在父类内部实现，如果需要子类自己实现，则可以
 声明成抽象的方法，由具体的子类实现。如果有一些不确定是否要执行的步骤或者在执行某一些步骤中需要做一些事情，则可以创建一些[钩子函数(#钩子函数)]**




## 类图

![XQBF7ZO`JWXOM1H24L_7MK4.png](https://i.loli.net/2019/09/15/7NAg8T3nt4OIFxj.png)

---
 　　此模式可以将一些相似的事物进行抽离成一个父类，就和我们平常封装差不多，但是和平常不同的是，我们会提供一个模版方法，并将不同的行为进行泛化，在模版方法中调用泛化后的方法
 ，泛化的方法由不同的子类自己实现不同的操作。就比如茶和咖啡，它们有共同的地方如：要先煮开水；也有不同的地方：加入咖啡粉或者加入茶叶。这样我们就可以将煮开水的操作由父类来实现，
进行复用，而加咖啡粉或者茶叶则可以泛化成一个方法，如 `冲泡brew()` ，茶或咖啡的类自己实现如何冲泡。冲泡之后可以选择加一些料，咖啡可以加奶啊，糖啊，茶叶也可以加奶啊，柠檬啊。这时
我们就可以用上 `hooks` ，用来询问是否要执行加料的操作。当然如果是必须要加的也就没必要用钩子了。

## 来个例子

 　　尝试一下上面说的茶和咖啡的例子，会有煮开水操作：`boilWater()` ，冲泡操作：`brew()` 和加料操作：`addCondiments()`
 
 
#### 类图

![_U_8_K40TGP1OVN0T_T2Q3O.png](https://i.loli.net/2019/09/15/xfR4TBvkcsXG69y.png)

#### 代码

父类AbstractBeverage：

```java
public abstract class AbstractBeverage {

    public final void prepare() {
        boilWater();
        brew();

        if (hooks()) {
            addCondiments();
        }
    }

    /**
     *
     * 冲泡方法，由子类实现
     *
     */
    public abstract void brew();


    public void boilWater() {
        System.out.println("开水煮沸了。。。");
    }

    /**
     * 询问是否需要加料，默认为false
     *
     * @return   boolean
     */
    public boolean hooks() {
        return false;
    }


    /**
     *
     * 具体加料方法
     *
     */
    public abstract void addCondiments();

}
```

咖啡类：


```java
public class Coffee extends AbstractBeverage {


    @Override
    public void brew() {
        System.out.println("加入咖啡粉冲泡。。");
    }

    @Override
    public boolean hooks() {
        return true;
    }

    @Override
    public void addCondiments() {
        System.out.println("加奶。。");
    }
}
```


茶类：不需要加料，所以不需要重写钩子，不管加料方法


```java
public class Tea extends AbstractBeverage {
    
    @Override
    public void brew() {
        System.out.println("将茶叶放入开水中冲泡。。");
    }
    
    @Override
    public void addCondiments() {
        // 茶不需要加料 所以不管
    }
}
```


#### 执行

```java
public class Main {

    public static void main(String[] args) {
        AbstractBeverage tea = new Tea();
        tea.prepare();

        AbstractBeverage coffee = new Coffee();
        coffee.prepare();
        
        // 开水煮沸了。。。
        //将茶叶放入开水中冲泡。。
        // 开水煮沸了。。。
        // 加入咖啡粉冲泡。。
        // 加奶。。
    }
}
```


## jdk中的🌰
 　　 `ArrayList` 继承了抽象类 `AbstractList` ，有许多方法在AbstractList中已经有默认的实现了。如：
 
 `indexOf`,`equals`等等等，还有的是抽象方法等着子类去实现的，如：`get`
 
    public abstract E get(int index);
    
在`AbstractList`内部有Iterator的私有实现类，这是AbstractList提供的默认迭代器，也是继承者们默认拥有的迭代器（如果没有自己实现`iterator()`,`listIterator()`等方法）。在迭代器方法`next()`, `remove()`中，调用了`get()`，`remove()`方法。
可以看到这两个方法在 AbstractList 内部并没有真正的实现，而是直接抛出异常！意思就是必须你自己去实现，不实现就用不了了！所以迭代器中方法也可以看做是模板方法，其中一部分等待子类去实现。

    // AbstractList提供的迭代器中方法
    public E next() {
        checkForComodification();
        try {
            int i = cursor;
            
            // 调用抽象方法get
            E next = get(i);
            lastRet = i;
            cursor = i + 1;
            return next;
        } catch (IndexOutOfBoundsException e) {
            checkForComodification();
            throw new NoSuchElementException();
        }
    } 
    
    
    // AbstractList提供的迭代器中方法
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
            checkForComodification();

        try {
            // 调用AbstractList的remove方法
            AbstractList.this.remove(lastRet);
            if (lastRet < cursor)
                cursor--;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException e) {
            throw new ConcurrentModificationException();
        }
    }
    
    
    // AbstractList中被调用的remove，默认实现直接抛异常
    public E remove(int index) {
        throw new UnsupportedOperationException();
    }
    
---
 　　在上面例子中，就可以看到模板方法在jdk中的使用。父类实现公共的方法（`indexOf`,`equals`），让子类实现某些步骤即实现某些方法（`get`），或是父类提供一些默认的实现（迭代器）。
 或许还可以把`remove()`等默认抛异常的看成是“钩子函数”！

## 总结

- 模板方法定义了算法的步骤，并把这些步骤延迟到子类
- 模板方法中可以定义具体方法，抽象方法，钩子。抽象方法由子类实现，钩子是一种方法，在抽象类中不做事，或者只做默认的事。
- 为了子类修改模板方法，可以将模板方法申明为final


## 钩子函数

 　　通俗一点讲，可以看成是在执行到某一个步骤时调用的方法，比如 [React组件生命周期的钩子函数](https://ddmcc.space/2019/09/02/react-component-lifecycle-methods/)，当执行到的时候就会调用钩子，来询问是否要做些什么
