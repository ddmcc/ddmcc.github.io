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
     * 确认购买饮料操作
     */
    public void returnDrink() {
        if (currentState == HAS_COIN) {
            currentState = SOLD;
            System.out.println("请收好饮料！");
            dispense();
        } else if (currentState == SOLD_OUT) {
            System.out.println("饮料已售罄！");
        } else if (currentState == CHOOSE_DRINK) {
            System.out.println("请勿重复选择！");
        } else if (currentState == NO_COIN) {
            System.out.println("请先投币！");
        } else if (currentState == SOLD) {
            System.out.println("饮料已退出！");
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
只有在`待售`的状态才能选择饮料，否则提示错误信息。选择饮料后则将状态转为`选择饮料的状态`等待投入硬币

- **投币**
只有在`选择饮料状态`下才能够投币，投币后变为`已投币`状态等待用户确认购买操作

- **退币**
在`已投币`状态下用户可以选择退币，退币后状态转为`未投币`

- **确认购买饮料** 
在`已投币`状态下除了可以退币，还可以确认购买，确认购买后饮料机将状态转为`SOLD`状态并调用`dispense()`方法，执行发放饮料操作


- **发放饮料**
将count - 1并判断剩余数量，转为售罄或未投币


---

 　　上面的实现代码满足了贩卖机的需求，但是并不是健壮的代码。首先违反了开-闭原则，当有新的状态加入我们就不得不重新打开代码这个类来修改，并且每个方法都会被影响。。。
 而且代码可读性也不佳。if-else是很影响代码可读性的，state的变化也隐藏在if语句里，并不明显。很可能对后面维护这些代码的人带来麻烦。重要的是没有将变化的部分 **封装** 起来！


## 引入状态模式
 　　现在我们尝试将“变化的部分”封装起来，变化的部分就是 **状态**。利用面向对象的思想，将每个状态封装为一个具体的类，将状态各自的行为放到自己的类中，那么每个状态只要实现自己的规则即可。
 然后将贩卖机委托给当前的状态对象，当调用某个方法时，再调用状态对象的方法。
 
- 首先，定义一个State接口。接口里有贩卖机每个动作的方法

- 然后每个状态创建一个具体的状态类，状态类实现State接口，并实现在这个状态下贩卖机的行为

- 最后将贩卖机的动作委托到状态类

### 封装的状态类图
![](https://i.loli.net/2019/11/04/WPZofAc6IpliEnU.png)



---
 　　我们已经知道每个状态下什么行为应该做什么，上面代码已经实现过了，现在只是把它们分散在各自类中。
 
**定义状态接口**

 ```java
 
 
/**
 * 状态接口
 *
 * @author ddmcc
 */
public interface State {

    /**
     * 选择饮料
     */
    void chooseDrink();

    /**
     * 投币操作
     */
    void insertCoin();


    /**
     * 退币操作
     */
    void ejectCoin();


    /**
     * 确认购买饮料操作
     */
    void returnDrink();


    /**
     * 退出饮料
     */
    void dispense();

}
```

**实现待售状态类**

```java
/**
 * 待售状态
 *
 * @author ddmcc
 */
public class NoCoinState implements State {

    /**
     * 贩卖机实例变量，通过它控制贩卖机的状态，饮料数量
     */
    private DrinkMachine drinkMachine;


    public NoCoinState(DrinkMachine drinkMachine) {
        this.drinkMachine = drinkMachine;
    }


    @Override
    public void chooseDrink() {
        // 待售状态下 进行选择饮料操作
        drinkMachine.setCurrentState(drinkMachine.getChooseDrinkState());
        System.out.println("饮料选择成功！请投币");
    }

    @Override
    public void insertCoin() {
        System.out.println("请先选择饮料！");
    }

    @Override
    public void ejectCoin() {
        System.out.println("请先投币！");
    }

    @Override
    public void returnDrink() {
        System.out.println("请先选择饮料！");
    }

    @Override
    public void dispense() {
        System.out.println("请先选择饮料！");
    }
}
```


**新的贩卖机**


```java
/**
 * @author ddmcc
 */
@Data
public class DrinkMachine {


    /**
     * 状态常量都由状态对象代替了
     */
    private State noCoinState;

    private State hasCoinState;

    private State soldOutState;

    private State soldState;

    private State chooseDrinkState;

    private int count = 0;


    /**
     * 记录当前状态
     */
    private State currentState;


    public DrinkMachine(int count) {
        noCoinState = new NoCoinState(this);
        hasCoinState = new HasCoinState(this);
        soldOutState = new SoldOutState(this);
        soldState = new SoldState(this);
        chooseDrinkState = new ChooseDrinkState(this);
        if (count > 0) {
            currentState = noCoinState;
        } else {
            currentState = soldOutState;
        }
    }


    /**
     * 选择饮料
     */
    public void chooseDrink() {
        // 动作都委托给当前的状态对象了
        currentState.chooseDrink();
    }


    /**
     * 投币操作
     */
    public void insertCoin() {
        currentState.insertCoin();
    }


    /**
     * 退币操作
     */
    public void ejectCoin() {
        currentState.ejectCoin();
    }


    /**
     * 退出饮料操作
     */
    public void returnDrink() {
        currentState.returnDrink();
    }


    /**
     * 退出饮料
     */
    public void dispense() {
        currentState.dispense();
    }

}
```


---
 　　上面只实现了一个状态的伪代码，其它状态类也差不多，也就是把一开始的代码“局部化”。上面版本中针对第一版本进行了以下优化：
 
- 将每个状态的行为封装进各自的类中

- 删除if语句，方便日后维护

- 让每一个状态“对修改关闭”，让贩卖机对“扩展开放”，因为可以加入新的状态，而我们也把所有的状态都放在贩卖机中了。对修改关闭怎么理解呢？其实之前已经说过了，修改关闭不是说
不让修改，而是修改不应该对其它带来影响。而为什么要把所有状态放到贩卖机呢？首先贩卖机总是在这些状态中游走，其次可以降低状态类间的依赖。（如果把状态变化放到贩卖机中，则可以让状态类
之间完全没有依赖，而我们选择了在状态类中转换状态，贩卖机提供getter方法，把状态之间的的依赖降到最低）


## 定义

 　　**允许对象在内部状态改变时改变它的行为，对象看起来好像改变了它的类。**

- 允许对象在内部状态改变时改变它的行为

因为我们将状态封装成不同的类，并将动作委托到当前状态中，当状态改变时，行为也就变了

- 对象看起来好像改变了它的类

对于客户端而言，并不知道内部如何实现，看起来就好像一个新的实例。其实是通过引用不同状态来造成的假象

## 类图

![](https://i.loli.net/2019/11/02/9wsS3tdYjGVqr4U.png)

## 状态模式的优缺点
- 优点
    - 将不同行为“局部化”，符合开-闭原则
    - 通过组合，委托的方式动态的改变行为

- 缺点
    - 导致设计中的类数目大大增加

## 状态模式与策略模式
- 状态模式
    - 我们将一群行为封装在状态对象中，Context可以随时委托到那些对象中的一个。随着时间流逝，当前状态在状态集合中游走，因此Context的行为也会跟着改变。但是对于Context的客户来说
这是浑然不觉的。
    - 状态模式可以看成在Context中不用放很多判断条件，将不同的行为封装到不同的状态中，通过改变当前状态来改变行为

**在固定的状态集合中游走**，**用户浑然不觉**

- 策略模式
    - 客户通常主动的指定要组合的策略对象是哪一个，摆脱继承的束缚，通过组合的方式动态的改变策略。

**客户指定组合策略**，**比继承的更有弹性替代方案**
