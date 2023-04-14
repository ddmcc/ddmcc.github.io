---
layout: post
title:  "设计模式之适配器模式"
date:   2019-08-08 20:51:20
categories: 设计模式
tags:  设计模式 适配器模式
author: ddmcc
---

* content
{:toc}


## 定义

将一个类的接口转换成使用者所期望的另一个接口。适配器让原本接口不兼容的类可以合作无间。

**上面这句话解释就是使用者要求的接口与提供的接口不一致，所以在它们在中间加了一个适配器类，适配器的接口是符合要求的。在适配器接口中调用所提供的接口，并解决两者之间的差异**





## 类适配器与对象适配器

适配器模式分为对象适配器和类适配器，下面是类图

---
#### 对象适配器

![markdown](https://ddmcc-1255635056.file.myqcloud.com/d6998d87-fd83-4f46-bd86-b15d913ff80d.png)

---
#### 类适配器

![markdown](https://ddmcc-1255635056.file.myqcloud.com/49b7d6cc-653a-4a9e-b937-2b0612372c07.png)

---
适配器模式主要分为四个角色

Client：接口调用者

Target：目标接口（调用者需要的接口）

Adapter：适配器用于包装转换被适配者

Adaptee：被适配者（实际提供的接口）

---
解释一下类图是这样的：

客户端：我要调用`operation`方法

Adaptee：没有，只有`method`方法

客户端：我就要`operation`方法！

Adaptee：没有，快滚！

----
![markdown](https://ddmcc-1255635056.file.myqcloud.com/76049c8b-aec6-47fd-ade6-5945cd4e4658.png)
----

然后它们就打起来了....


这时和事老适配器兄出现了，将它们拉开。

**适配器实现了目标接口的operation方法，当收到请求时，再委托或转接给被适配者**



---
#### 两者区别

它们两的区别就是对象适配器使用了**组合**的方式，而类适配器使用了**继承**的方式（因为不支持多继承，所以实现Target）。这两种实现的所产生的差异就是
使用组合的对象适配器更有弹性，它不仅适配了该适配者，适配者的子类也适配了。而类适配者的好处是因为使用的是继承，所以可以不需要重新实现整个被适配者（这边的
意思是如果有不需要适配的接口，即接口方法名，参数什么的都和Target里的一致，则不需要重写，如下代码）。当然也可以重写父类的方法。


```java
public class Main {

    public static void main(String[] args) {
        Adapter adapter = new Adapter();
        adapter.operation();
    }
    
    interface Target {
        void operation();
    }

    static class Adaptee {
        public void operation(){
            System.out.println(111);
        }
    }

    static class Adapter extends Adaptee implements Target {

    }
}
```


## 来个栗子

现在大部分手机产商都不送耳机了，而且也取消了3.5mm的耳机孔，改成充电器孔和耳机孔一起。。假如你买了一个新手机是无耳机孔，但是你的耳机是3.5mm接口的。这时就出现了手机与耳机无法适配的情况。


![markdown](https://ddmcc-1255635056.file.myqcloud.com/0eb9e727-810e-48f4-947f-f63377e8fae1.png)
---
这时候就需要用转接器了！而适配器就相当于做了转接器的工作。类图如下
---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/55e8c59c-cbe1-44d5-b266-5fece4518d47.png)
---
代码如下：


```java
public interface Phone {

    void earPhonePlug();

}


public class Samsung implements Phone {

    @Override
    public void earPhonePlug() {
        System.out.println("三星手机Type-C接口");
    }
}

public interface EarPhoneAdapter {

    void earPhonePlugThreeMM();

    void earPhonePlugTypeC();

}


public class SamsungAdapter implements EarPhoneAdapter {

    private Phone phone;

    public SamsungAdapter(Phone phone) {
        this.phone = phone;
    }

    @Override
    public void earPhonePlugThreeMM() {
        System.out.println("3.5mm接口");
        earPhonePlugTypeC();
    }

    @Override
    public void earPhonePlugTypeC() {
        System.out.println("Type-C接口");
        phone.earPhonePlug();
    }
}
```

运行，即耳机插入适配器

```java
public static void main(String[] args) {
    Phone phone = new Samsung();
    EarPhoneAdapter earPhoneAdapter = new SamsungAdapter(phone);
    earPhoneAdapter.earPhonePlugThreeMM();
	// 输出
	3.5mm接口
    Type-C接口
    三星手机Type-C接口
}
```

如果有个华为的手机，也是Type-C接口的，这时并不需要修改适配器代码，就能够直接复用。**这就是组合的威力**

```java
public class HuaWei implements Phone {
    @Override
    public void earPhonePlug() {
        System.out.println("华为Type-C接口");
    }
}


public static void main(String[] args) {
    Phone phone = new HuaWei();
    EarPhoneAdapter earPhoneAdapter = new SamsungAdapter(phone);
    earPhoneAdapter.earPhonePlugThreeMM();
}
```

---
如果用类适配器的实现方式的话，代码如下：


主要是适配器的变化，由组合的方式变成继承

```java
public class SamsungAdapter extends Samsung implements EarPhoneAdapter {

    @Override
    public void earPhonePlugThreeMM() {
        System.out.println("3.5mm接口");
        earPhonePlugTypeC();
    }

    @Override
    public void earPhonePlugTypeC() {
        System.out.println("Type-C接口");
        super.earPhonePlug();
    }
}
```

运行

```java
public static void main(String[] args) {
    EarPhoneAdapter earPhoneAdapter = new SamsungAdapter();
    earPhoneAdapter.earPhonePlugThreeMM();
	// 输出
	3.5mm接口
    Type-C接口
    三星手机Type-C接口
}
```

这样你会发现这个三星的耳机适配器和三星手机已经绑死了，因为继承了三星手机。如果我这时想给华为手机用，就用不了了。就不得不再买一个华为的适配器。


## 实际一点的栗子

早期HashTable的遍历是用Enumeration接口的，后来出了迭代器Iterator，现在想让HashTable也用上迭代器怎么做呢？？

Enumeration<E> 接口：

>    boolean hasMoreElements();

>   E nextElement();


Iterator<E> 接口：

>    boolean hasNext();
>
>    E next();
>   
>    default void remove() {
>        throw new UnsupportedOperationException("remove");
>    }



在HashTable有一个Enumerator<T> 类实现了Enumeration<E>和Iterator<E>接口，并对两个接口进行了实现。 **实际上是只实现了Enumeration接口(也实现了remove方法)，Iterator的接口则直接调用对应的方法！**

```java
private class Enumerator<T> implements Enumeration<T>, Iterator<T> {
        /**
         * Indicates whether this Enumerator is serving as an Iterator
         * or an Enumeration.  (true -> Iterator).
         */
        boolean iterator;

        Enumerator(int type, boolean iterator) {
            this.type = type;
            this.iterator = iterator;
        }

        public boolean hasMoreElements() {
            Entry<?,?> e = entry;
            int i = index;
            Entry<?,?>[] t = table;
            /* Use locals for faster loop iteration */
            while (e == null && i > 0) {
                e = t[--i];
            }
            entry = e;
            index = i;
            return e != null;
        }

        // Iterator methods
        public boolean hasNext() {
            return hasMoreElements();
        }

        // 省略了其它方法...
    }
```


Enumerator就相当于一个适配器，能够适配两种方式的遍历！通过判断属性iterator来判断是哪种迭代器。iterator属性由构造函数传入，下面代码为新建Enumerator

```java
private <T> Enumeration<T> getEnumeration(int type) {
     if (count == 0) {
         return Collections.emptyEnumeration();
     } else {
         return new Enumerator<>(type, false);
     }
}

private <T> Iterator<T> getIterator(int type) {
    if (count == 0) {
        return Collections.emptyIterator();
    } else {
        return new Enumerator<>(type, true);
    }
}
```


## 适配器模式与装饰者模式

#### 装饰者模式
---
装饰者模式是利用组合的方式，动态的对对象增加新的行为或者增强，而无需更改现有的代码。


#### 适配器模式
---
主要是对接口的转换，将旧的接口转换成新的符合客户要求的接口。

**适配器模式主要是为了接口的转换，而装饰者模式是通过组合来动态的为被装饰者注入新的功能或修改行为。**

## 总结

- 适配器模式用于将一个接口转换成客户希望的另一个接口，适配器模式使接口不兼容的那些类可以一起工作。适配器模式有类适配器和对象适配器。
- 适配器模式包含四个角色
    - 目标抽象类（Target）即客户要用的接口
    - 适配器类（Adapter）实现了目标类的接口，并拥有被适配者的抽象类型引用。作为一个转换器，对被适配者和抽象目标类进行适配
    - Adaptee是被适配的角色，它定义了一个已经存在的接口，这个接口需要适配
    - 在客户类（Client）中针对目标抽象类进行编程，调用在目标抽象类中定义的业务方法

- 在类适配器模式中，适配器类实现了目标抽象类接口并继承了适配者类，并在目标抽象类的实现方法中调用所继承的适配者类的方法；在对象适配器模式中，适配器类实现了目标抽象类并定义了一个被适配者类的对象实例，在所实现的目标抽象类方法中调用被适配者类的相应业务方法。
- 适配器模式适用于系统需要使用现有的类，而这些类的接口不符合系统的需要，通常用于对原有的系统进行升级改造的时候。


## 优点和缺点

#### 优点
---

- 有更好的复用性。系统需要使用现有的类，但此类接口不符合系统需要，通过适配器模式让这些功能得到很好的复用

- 同时因为使用了组合的方式，所有被适配对象实例的子类都可以被适配，使系统更有弹性。

#### 缺点
---

- 过多使用适配器会使得系统非常凌乱，明明调用的是A接口，内部却被适配成了B接口