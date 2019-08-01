---
layout: post
title:  "设计模式之责任链模式"
date:   2019-07-31 20:37:49
categories: 设计模式
tags: 设计模式 责任链模式
author: ddmcc
---

* content
{:toc}


## 定义


让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。



## 例子

假设有一个请假的流程。员工发起请假请求，如果小于等于1天需要组长审批，小于等于三天需要经理审批，大于三天需要总监审批。如果不用责任链
模式，代码可能会如下：

首先定义请假信息类，包含字段：`name`(请假人),`days`(请假天数)

```java
public class LeaveInfo {

    private double days;

    private String name;

    public LeaveInfo() {}

    public LeaveInfo(String name, double days) {
        this.days = days;
        this.name = name;
    }

    public double getDays() {
        return days;
    }

    public void setDays(double days) {
        this.days = days;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```


再定义请假类型


```java
public enum LeaveType {

    // 组长
    TEAM_LEADER(1),

    // 经理
    MANAGER(3);

    double days;

    LeaveType(double days) {
        this.days = days;
    }

    public double getDays() {
        return days;
    }
}
```

编写请假处理类

```java
public class LeaveService {

    public void leave(LeaveInfo leaveInfo) {
        if (leaveInfo.getDays() <= 0) {

        } else if (leaveInfo.getDays() <= LeaveType.TEAM_LEADER.getDays()) {
            System.out.println("组长审批 -> " + leaveInfo.getDays() +" 天");
        } else if (leaveInfo.getDays() <= LeaveType.MANAGER.getDays()) {
            System.out.println("经理审批 -> " + leaveInfo.getDays() +" 天");
        } else {
            System.out.println("总监审批 -> " + leaveInfo.getDays() +" 天");
        }
    }

}
```

还有可能是一级一级审批上去的

```java
public void transferLeave(LeaveInfo leaveInfo) {
    if (leaveInfo.getDays() <= 0) {

    } else if (leaveInfo.getDays() <= LeaveType.TEAM_LEADER.getDays()) {
        System.out.println("组长审批 -> " + leaveInfo.getDays() +" 天");
    } else if (leaveInfo.getDays() <= LeaveType.MANAGER.getDays()) {
        System.out.println("组长审批 -> " + leaveInfo.getDays() +" 天");
        System.out.println("经理审批 -> " + leaveInfo.getDays() +" 天");
    } else {
        System.out.println("组长审批 -> " + leaveInfo.getDays() +" 天");
        System.out.println("经理审批 -> " + leaveInfo.getDays() +" 天");
        System.out.println("总监审批 -> " + leaveInfo.getDays() +" 天");
    }
}
````


在上面的代码中虽然实现了需求，但是代码并不美观。

- 假设增加超过五天还需要总经理审批，就不得不重新来修改`if···else···`代码。
- 如果审批顺序发生变化，也要对leave进行修改。

如果流程足够的复杂，这样的代码对后期维护是很麻烦的，假设这些代码不是自己写的，或者过了很久回过头来看这些代码。
则必须先去理解每个if的条件是什么。

重要的是这样的写法违背了`开闭原则`，后期代码维护拓展就不得不打开leave方法来修改，并且可能会影响流程中结果。


## 引入责任链模式

在每个处理对象中都有下一个处理对象的引用，形成一个责任链。

![](https://i.loli.net/2019/08/01/5d42462960dc169192.png)


1，抽象出一个处理类

```java
public abstract class AbstractLeaveHandle {

    protected AbstractLeaveHandle nextLeaveHandle;

    public AbstractLeaveHandle(AbstractLeaveHandle leaveHandle) {
        this.nextLeaveHandle = leaveHandle;
    }

    protected void leave(LeaveInfo leaveInfo) {
        prepare(leaveInfo);
        if (nextLeaveHandle != null) {
            nextLeaveHandle.leave(leaveInfo);
        }
    }

    protected void transferLeave(LeaveInfo leaveInfo) {
        transferPrepare(leaveInfo);
        if (nextLeaveHandle != null) {
            nextLeaveHandle.transferLeave(leaveInfo);
        }
    }

    /**
     * 由具体处理着自己去实现
     *
     * @param leaveInfo     请假信息
     */
    protected abstract void prepare(LeaveInfo leaveInfo);

    protected abstract void transferPrepare(LeaveInfo leaveInfo);

}
```

在每个处理着中都有下一个处理者的引用，当有下一个处理者则传递下去。

2，编写三个具体处理者继承`AbstractLeaveHandle`

组长处理者：

```java
public class TeamLeaderHandle extends AbstractLeaveHandle {

    public TeamLeaderHandle(AbstractLeaveHandle leaveHandle) {
        super(leaveHandle);
    }

    @Override
    protected void prepare(LeaveInfo leaveInfo) {
        if (leaveInfo.getDays() <= LeaveType.TEAM_LEADER.getDays()) {
            System.out.println(String.format("组长审批 -> %s 请假 %s 天", leaveInfo.getName(), leaveInfo.getDays()));
        }
    }

    @Override
    protected void transferPrepare(LeaveInfo leaveInfo) {
        if (leaveInfo.getDays() > 0) {
            System.out.println(String.format("组长审批 -> %s 请假 %s 天", leaveInfo.getName(), leaveInfo.getDays()));
        }
    }
}
```

经理处理者：

```java
public class ManagerHandle extends AbstractLeaveHandle {


    public ManagerHandle(AbstractLeaveHandle leaveHandle) {
        super(leaveHandle);
    }

    @Override
    protected void prepare(LeaveInfo leaveInfo) {
        if (leaveInfo.getDays() <= LeaveType.MANAGER.getDays()) {
            System.out.println(String.format("经理审批 -> %s 请假 %s 天", leaveInfo.getName(), leaveInfo.getDays()));
        }
    }

    @Override
    protected void transferPrepare(LeaveInfo leaveInfo) {
        if (leaveInfo.getDays() >= LeaveType.MANAGER.getDays()) {
            System.out.println(String.format("经理审批 -> %s 请假 %s 天", leaveInfo.getName(), leaveInfo.getDays()));
        }
    }
}
```

总监处理者：

```java
public class DirectorHandle extends AbstractLeaveHandle {

    public DirectorHandle(AbstractLeaveHandle leaveHandle) {
        super(leaveHandle);
    }

    @Override
    protected void prepare(LeaveInfo leaveInfo) {
        if (leaveInfo.getDays() > LeaveType.MANAGER.getDays()) {
            System.out.println(String.format("总监审批 -> %s 请假 %s 天", leaveInfo.getName(), leaveInfo.getDays()));
        }
    }

    @Override
    protected void transferPrepare(LeaveInfo leaveInfo) {
        if (leaveInfo.getDays() > LeaveType.MANAGER.getDays()) {
            System.out.println(String.format("总监审批 -> %s 请假 %s 天", leaveInfo.getName(), leaveInfo.getDays()));
        }
    }
}
```


3，调用的方法

```java
public class Main {

    public static void main(String[] args) {
        AbstractLeaveHandle directorHandle = new DirectorHandle(null);
        AbstractLeaveHandle managerHandle = new ManagerHandle(directorHandle);
        AbstractLeaveHandle leaderHandle = new TeamLeaderHandle( managerHandle);
        leaderHandle.leave(new LeaveInfo("大锤", 6));

        leaderHandle.transferLeave(new LeaveInfo("大锤", 3));
    }
}
```


在引入责任链模式之后解决的违背`开闭原则`的问题，在新增或减少审批流程不需要去修改具体的审批实现，修改某一个环节的审批也不会对其他
的代码产生影响。并且遵守了 **单一职责原则**，各个审批者只做自己的事。



但是这样你会发现每个对象引用下一个处理对象，如果多的话它们之间的关系会感觉很乱。


## 升级的责任链模式


主要多了一个离职责任链类

```java
public interface Chain {

    void doFilter(LeaveInfo leaveInfo);

    Chain addHandle(AbstractLeaveHandle addHandle);
}
```


```java
public class LeaveChain implements Chain {

    private int currentPosition = 0;

    private List<AbstractLeaveHandle> leaveHandles;

    @Override
    public void doFilter(LeaveInfo leaveInfo) {
        if (currentPosition < leaveHandles.size()) {
            ++currentPosition;
            AbstractLeaveHandle nextHandle = leaveHandles.get(currentPosition - 1);
            nextHandle.prepare(leaveInfo);
            this.doFilter(leaveInfo);
        }
    }

    @Override
    public Chain addHandle(AbstractLeaveHandle addHandle) {
        if (leaveHandles == null) {
            leaveHandles = new ArrayList<>();
        }
        leaveHandles.add(addHandle);
        return this;
    }
}
```


在客户端调用代码

```java
public class Main {

    public static void main(String[] args) {
        Chain chain = new LeaveChain();
        chain.addHandle(new TeamLeaderHandle()).addHandle(new ManagerHandle()).doFilter(new LeaveInfo("大锤", 3));
    }
}
```


这样每个对象拥有下个处理者的引用而造成的关系混乱也就不存在了。而且这样处理会更加的灵活，我们可以通过*.xml上配置列表的处理对象。

```xml
<bean id="leaveChain" class="xxx.xxx.xxx.LeaveChain">
    <property name="leaveHandles">
        <list>
            <ref bean="teamLeaderHandle" />
            <ref bean="managerHandle" />
            ··· ···
        </list>
    </property>
</bean>
```

在Servlet中的Filter就是这么处理的。在web.xml中配置Filter，如果需要增删改配置文件就好了。



## 使用场景

会发现，在jdk源码使用责任链的模式中，基本都不是我们上述的业务流程，或者说都不怎么相似。像过滤器这样都是经过所有的过滤器后，最后在处理我们的业务。

![](https://i.loli.net/2019/08/01/5d424b777ff4447368.png)


比如做吃饭这件事，在吃饭之前要买菜，洗菜，煮菜...

```java
public interface EatPrepareFilter {

    void doFilter(Eat eat, EatChain chain);

}


public class EatChain implements EatPrepareFilter {

    private int currentPosition = 0;

    private List<EatPrepareFilter> prepareFilterList;


    @Override
    public void doFilter(Eat eat, EatChain chain) {
        if (this.currentPosition == this.prepareFilterList.size()) {
            eat.eat();
        } else {
            prepareFilterList.get(currentPosition++).doFilter(eat, this);
        }
    }

    public EatChain addFilter(EatPrepareFilter filter) {
        if (prepareFilterList == null) {
            prepareFilterList = new ArrayList<>();
        }
        prepareFilterList.add(filter);
        return this;
    }
}



public class BuyFilter implements EatPrepareFilter {

    @Override
    public void doFilter(Eat eat, EatChain chain) {
        System.out.println("买菜");
        chain.doFilter(eat, chain);
    }
}


public class WashFilter implements EatPrepareFilter {

    @Override
    public void doFilter(Eat eat, EatChain chain) {
        System.out.println("洗菜");
        chain.doFilter(eat, chain);
    }
}


public class BoiledFilter implements EatPrepareFilter {

    @Override
    public void doFilter(Eat eat, EatChain chain) {
        System.out.println("煮菜");
        chain.doFilter(eat, chain);
    }
}


public class Eat {

   public void eat() {
       System.out.println("吃饭中");
   }
}


public class Main {

    public static void main(String[] args) {
        EatChain chain = new EatChain();
        chain.addFilter(new BuyFilter()).addFilter(new WashFilter()).addFilter(new BoiledFilter()).doFilter(new Eat(), chain);
    }
}
```



## 优缺点

#### 优点

- 将请求者与处理者解耦
- 将具体的处理业务抽出，由各自处理类处理(单一职责)
- 不需要知道链的具体结构与实现
- 通过改变链里的处理者与它们的顺序能够动态的增加删除

#### 缺点

- 出错不易排查