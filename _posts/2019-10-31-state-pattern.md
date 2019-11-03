---
layout: post
title:  "设计模式之状态模式"
date:   2019-10-31 22:00:59
categories: 设计模式
tags:  设计模式 状态模式
author: ddmcc
---

* content
{:toc}

 
## 状态模式

 　　我们用饮料贩卖机工作的例子来讲解状态模式的实现。把贩卖机的工作流程分解，可以分为一般为 **待售 -> 选择饮料 -> 插入硬币 -> 售出 -> 退出饮料 -> 回到待售状态。**
每一次售出饮料都是这个步骤，贩卖机的状态始终在这些状态中游走。同时为了我们的贩卖机更加的安全，需要在每次请求的时候判断当前的状态是否允许这么做。比如贩卖机要退出饮料，
要先确认当前是否为售出的状态？以及饮料的库存






```java
/**
 * @author ddmcc
 */
@Data
public class DrinkMachine {

    /**
     * 售罄状态
     */
    private static final int SOLD_OUT = 0;

    /**
     * 没有硬币 也就是待售状态
     */
    private static final int NO_COIN = 1;

    /**
     * 选择饮料
     */
    private static final int CHOOSE_DRINK = 2;

    /**
     * 投入硬币
     */
    private static final int HAS_COIN = 3;

    /**
     * 售出状态 等待退出饮料
     */
    private static final int SOLD = 4;

    /**
     * 当前剩余饮料数量
     */
    int count = 0;

    /**
     * 当前状态 默认为售罄
     */
    int currentState = SOLD_OUT;


    public DrinkMachine(int count) {
        this.count = count;
        if (count > 0) {
            currentState = NO_COIN;
        }
    }



    /**
     * 选择饮料
     */
    public void chooseDrink() {
        if (currentState == NO_COIN) {
            currentState = CHOOSE_DRINK;
            System.out.println("饮料选择成功，请投币！");
        } else if (currentState == SOLD_OUT) {
            System.out.println("饮料已售罄！");
        } else if (currentState == CHOOSE_DRINK) {
            System.out.println("请勿重复选择！");
        } else if (currentState == HAS_COIN) {
            System.out.println("请勿重复选择！");
        } else if (currentState == SOLD) {
            System.out.println("请勿重复选择！");
        }
    }


    /**
     * 投币操作
     */
    public void insertCoin() {
        if (currentState == CHOOSE_DRINK) {
            currentState = HAS_COIN;
            System.out.println("投币成功，请稍等！");
        } else if (currentState == SOLD_OUT) {
            System.out.println("投币失败，饮料已售罄");
        } else if (currentState == NO_COIN) {
            System.out.println("请先选择饮料！");
        } else if (currentState == HAS_COIN) {
            System.out.println("请勿重复投币！");
        } else if (currentState == SOLD) {
            System.out.println("请勿重复投币！");
        }
    }


    /**
     * 退币操作
     */
    public void ejectCoin() {
        if (currentState == HAS_COIN) {
            currentState = NO_COIN;
            System.out.println("退币成功！");
        } else if (currentState == SOLD_OUT) {
            System.out.println("退币失败！");
        } else if (currentState == CHOOSE_DRINK) {
            System.out.println("退币失败！");
        } else if (currentState == NO_COIN) {
            System.out.println("请先投币！");
        } else if (currentState == SOLD) {
            System.out.println("退币失败！");
        }
    }


    /**
     * 退出饮料操作
     */
    public void returnDrink() {
        if (currentState == HAS_COIN) {
            currentState = SOLD;
            System.out.println("请收好饮料！");
        } else if (currentState == SOLD_OUT) {
            System.out.println("饮料已售罄！");
        } else if (currentState == CHOOSE_DRINK) {
            System.out.println("请勿重复选择！");
        } else if (currentState == NO_COIN) {
            System.out.println("请先投币！");
        } else if (currentState == SOLD) {
            System.out.println("请稍等！");
            dispense();
        }
    }


    /**
     * 退出饮料
     */
    public void dispense() {
        if (currentState == SOLD) {
            count -= 1;
            System.out.println("欢迎下次光临！");
            if (count >= 1) {
                currentState = NO_COIN;
            } else {
                currentState = SOLD_OUT;
            }
        } else if (currentState == NO_COIN) {
            System.out.println("请先选择饮料！");
        } else if (currentState == CHOOSE_DRINK) {
            System.out.println("请先投币！");
        } else if (currentState == HAS_COIN) {
            System.out.println("请点击退出饮料！");
        } else if (currentState == SOLD_OUT) {
            System.out.println("饮料已售罄！");
        }
    }

}
```
---

 　　上面是根据贩卖机的工作流程编写的代码。首先贩卖机类内部有几种状态常量，还有记录当前状态和当前的数量的变量。然后是几个操作方法分别为：`选择饮料`，`投币`，`退币`，`机器退出饮料`，`发放饮料`。

- **选择饮料**


- **投币**

- **退币**

- **退出饮料** 

- **发放饮料**


## 引入状态模式


## 定义

 　　允许对象在内部状态改变时改变它的行为，对象看起来好像改变了它的类。

## 类图

![](https://i.loli.net/2019/11/02/9wsS3tdYjGVqr4U.png)


## 状态模式的优缺点

## 状态模式与策略模式

## 总结
