---
layout: post
title:  "设计模式之策略模式"
date:   2018-12-18 22:51:05
categories: 设计模式
tags: 设计模式 策略模式
author: ddmcc
---

* content
{:toc}




## 什么是策略模式

定义算法（行为）族，将它们分别封装起来，让它们之间可以替换，让算法或行为与此算法的使用者或行为的拥有者解耦。



## 如何设计

### 改造前的代码

有一个需求，根据不同客户计算价格的程序，输入价格与用户类型会输入计算后的价格。客户类型有VIP,一般的客户，一级新客户
分别打6折，9折和8折。

代码如下：
```java
public class Customer {

    public static final int VIP = 1;
    public static final int GENERAL = 2;
    public static final int NEW = 3;

    public void getPrice(Double price, int cusType) {
        switch (cusType) {
            case VIP:
                System.out.println("VIP客户打6折：" + price * 0.6);
                break;
            case GENERAL:
                System.out.println("普通客户打9折：" + price * 0.9);
                break;
            case NEW:
                System.out.println("新客户打8折：" + price * 0.8);
                break;
            default:
                System.out.println("无打折");
                break;
        }
    }
}
```

运行一下

```java
public class Main {

    public static void main(String[] args) {
        Customer customer = new Customer();
        customer.getPrice(120.5, Customer.NEW);
    }
}
```

运行结果


![markdown](https://ddmcc-1255635056.file.myqcloud.com/89cf5660-550c-4dc0-926c-34ea1106d65d.png)


以现在需求看上去感觉还行。但是如果现在需求变更，增加一个常客，打7折呢？就得在原来方法上加一个分支。
下面是更改后的代码
```java

public class Customer {

    public static final int VIP = 1;
    public static final int GENERAL = 2;
    public static final int NEW = 3;
    public static final int REGULAR = 4;

    public void getPrice(Double price, int cusType) {
        switch (cusType) {
            case VIP:
                System.out.println("VIP客户打6折：" + price * 0.6);
                break;
            case GENERAL:
                System.out.println("普通客户打9折：" + price * 0.9);
                break;
            case NEW:
                System.out.println("新客户打8折：" + price * 0.8);
                break;
            case REGULAR:
                System.out.println("常客打7折：" + price * 0.7);
            default:
                System.out.println("无打折");
                break;
        }
    }
}
```


**这样设计我们发现每次需求有变更都需要去更改原来的代码逻辑，所以现在我们需要把会变的这一部分单独的提出来，**
**把它们封装起来。下次再有新的需求我们只需要更改提出来的部分。**


### 改造后的代码

- 先建一个处理价格的接口`Handle`，里面有方法handle(double price); 让实现类自己去实现。

```java
public interface Handle {

    void handle(double price);

}
```

- 在创建具体的客户处理类取实现`Handle`接口，在handle方法中实现具体的计算规则。


VIP客户
```java
public class VipHandle implements Handle {


    @Override
    public void handle(double price) {
        System.out.println("VIP客户打6折：" + price * 0.6);
    }
}
```

新客户
```java
public class NewHandle implements Handle {


    @Override
    public void handle(double price) {
        System.out.println("新客户打8折：" + price * 0.8);
    }
}
```

普通客户
```java
public class GenralHandle implements Handle {


    @Override
    public void handle(double price) {
        System.out.println("普通客户打9折：" + price * 0.9);
    }
}
```

常客
```java
public class RegularHanndle implements Handle {


    @Override
    public void handle(double price) {
        System.out.println("常客打7折：" + price * 0.7);
    }
}
```
- **我们把不同的计算分为不同的类去实现，当下一次需求变更需要增加不同种类的客户或者改变计算规则，我们就可以直接**
**增加新的计算类，或者更改对应的计算类，而不会影响到原来的代码。**
- 在客户类中拥有一个`Handle`接口的实例变量，当我们根据不同的客户去把不同的计算类对象引用给Handle变量，
利用多态绑定运行实现类的方法。

```java
public class Customer {

    private Handle handle;

    public void getPrice(double price) {
        handle.handle(price);
    }

    public void setHandle(Handle handle) {
        this.handle = handle;
    }
}
```

调用

```java
public class Main {

    public static void main(String[] args) {
        Customer customer = new Customer();
	// 根据不同的客户设置不同的计算类
        customer.setHandle(new VipHandle());
        customer.getPrice(150.5);

	customer.setHandle(new GenralHandle());
        customer.getPrice(120.0);
    }
}
```

运行结果

![markdown](https://ddmcc-1255635056.file.myqcloud.com/7bc1c45d-e84b-4d04-a855-423435c50c92.png)

## 模式使用

在 [easypoi](http://easypoi.mydoc.io/) 中可以自定义设置excel数据校验接口，自定义校验类必须实现`IExcelVerifyHandler<T>`。
它是一个接口，里面只有一个`ExcelVerifyHandlerResult verifyHandler(T var1)` 方法。所以我们可以实现这个接口，然后在`verifyHandler`方法对
记录进行校验，每条记录会作为参数传进来。

```java
public class TestHandle implements IExcelVerifyHandler<StudentFields> {

    @Override
    public ExcelVerifyHandlerResult verifyHandler(StudentFields studentFields) {
        // 这里实现自定义数据校验，每条记录将作为参数传进来

        return null;
    }
    
}
```

在我们需要校验的地方，只需要打开校验，并动态的设置校验接口，就可以实现很灵活的数据校验。因为在`ImportParams`
导入参数类中有`private IExcelVerifyHandler verifyHandler`属性。当导入时打开了校验就会调用`verifyHandler.verifyHandler(StudentFields record);`对每条记录进行校验。


```java
 @Override
    public ResponseInfo importStudent(File file, HttpServletRequest request) {
        ImportParams params = new ImportParams();
        // 开启数据验证
        params.setNeedVerfiy(true);
        // 从第二行开始读取数据
        params.setTitleRows(1);
        // 设置验证数据接口
        params.setVerifyHandler(importHandle);
        try {
            ExcelImportResult<StudentFields> result = ExcelImportUtil.importExcelMore(file, StudentFields.class, params);
            // 成功记录
            List<StudentFields> successList = result.getList();
            for (StudentFields studentFields : successList) {
                
            }
            // 有失败的记录
            if (result.isVerfiyFail()) {
                // 失败列表
                List<StudentFields> failList = result.getFailList();
                
            }
            return this.getResult(SUCCESS, DO_SUCCESS);
        } catch (ExcelImportException e) {
            return this.getResult(FAIL, e.getMessage());
        } finally {
            if (file.exists()) {
                file.delete();
            }
        }
    }
```

这样利用策略模式把数据校验独立出来并进行封装，在后期需要导入不同的excel，进行不同的数据校验只需要新建一个
校验实现类继承`IExcelVerifyHandler<T>`，实现`verifyHandler`方法，在导入时会调用verifyHandler.verifyHandler(T var)，从而调用到
动态链接到我们实现的方法中。。当校验规则改变时，也不需要去修改原来的代码，只需要去独立出来的部分或者新建一个校验类。


## 设计原则

- 把可能会变化的部分独立并封装起来，以便以后可以轻易的改动，而部影响不需要变化的部分。
- 针对接口编程而不是针对实现编程。针对接口的真正意思是针对"超类型"，把变量声明为超类型。即利用多态动态绑定，去执行具体的实现类的方法。而不会
绑死在父类方法中。
- 多用组合，少用继承。
- "有一个"可能比"是一个"好。