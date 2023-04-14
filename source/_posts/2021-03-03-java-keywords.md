---

layout: post
title:  "Java中各种关键字"
date:   2021-03-03 16:56:23
categories: Java
tags:  Java基础
toc: true
---

简单点说，就是被transient修饰的成员变量，在序列化的时候其值会被忽略，在被反序列化后， transient 变量的值被设为初始值， 如 int 型的是 0，引用类型的是 null...

<!-- more -->

### **transient**

Java语言的关键字，变量修饰符，如果用transient声明一个实例变量，当对象存储时，它的值不需要维持。

>这里的对象存储是指，Java的serialization提供的一种持久化对象实例的机制。当一个对象被序列化的时候，transient型变量的值不包括在序列化的表示中，然而非transient型的变量是被包括进去的。使用情况是：当持久化对象时，可能有一个特殊的对象数据成员，我们不想用serialization机制来保存它。为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。

简单点说，就是被transient修饰的成员变量，在序列化的时候其值会被忽略，在被反序列化后， transient 变量的值被设为初始值， 如 int 型的是 0，引用类型的是 null。


`ArrayList` 中的保存数据的数组被 `transient` 所修饰。

```java
    transient Object[] elementData; // non-private to simplify nested class access
```


如果如上面的所说，被 `transient` 修饰的变量值在序列化、反序列化后，列表的数据将会为空：

```java
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        ArrayList<String> arrayList = new ArrayList<>();
        arrayList.add("2");
        arrayList.add("3");
        FileOutputStream fileOut = new FileOutputStream("./array.txt");
        ObjectOutputStream out = new ObjectOutputStream(fileOut);
        out.writeObject(arrayList);
        out.close();
        fileOut.close();

        FileInputStream fileIn = new FileInputStream("./array.txt");
        ObjectInputStream in = new ObjectInputStream(fileIn);
        ArrayList<String> arrayList1 = (ArrayList<String>) in.readObject();
        System.out.println(arrayList1.toString());
        in.close();
        fileIn.close();
    }
```

执行后输出：

```java
[2, 3]
```

数据并没有被设为初始值，这是因为 `ArrayList` 中定义了 `writeObject` 和 `readObject` 方法，实现了自定义序列化。序列化的时候ObjectStream会判断类中有没有自定义序列化方法？如果有，使用自定义序列化方法，否则使用默认的序列化方法。
进一步了解 [序列化、反序列化]() 

如果是默认的序列化方法是不会序列化 `transient` 字段的：

定义实体类：

```java
public class Person implements Serializable {

    private String name;

    private transient int age;

    private transient String[] tags;

}
```
   
```java
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Person person = new Person();
        person.setAge(11);
        person.setName("ddmcc");
        person.setTags(new String[]{"1", "2"});
        FileOutputStream fileOut = new FileOutputStream("./object.txt");
        ObjectOutputStream out = new ObjectOutputStream(fileOut);
        System.out.println("序列化前：" + person.toString());
        out.writeObject(person);
        out.close();
        fileOut.close();

        FileInputStream fileIn = new FileInputStream("./object.txt");
        ObjectInputStream in = new ObjectInputStream(fileIn);
        Person person1 = (Person) in.readObject();
        System.out.println("序列化后：" + person1.toString());
        in.close();
        fileIn.close();
    }
```

输出结果为：

```java
序列化前：Person{name='ddmcc', age=11, tags=[1, 2]}
序列化后：Person{name='ddmcc', age=0, tags=null}
```

 `transient` 变量的值被设为初始值，int 型的是 0，对象型的是 null
 
 
### **instanceof**

instanceof 是 Java 的保留关键字。它的作用是测试它左边的对象是否是它右边的类的实例，返回 boolean 的数据类型。

Java语言规范关于 `instanceof` 关键字的解释：

![markdown](https://ddmcc-1255635056.file.myqcloud.com/5552a05b-d33a-4f07-b013-615e61d75467.png)



举个🌰：

```java
    if (a instanceof b) {
    
    }
    
    if (o instanceof Vector) {
         System.out.println("对象是 java.util.Vector 类的实例");
    } else if (o instanceof ArrayList) {
        System.out.println("对象是 java.util.ArrayList 类的实例");
    } else {
        System.out.println("对象是 " + o.getClass() + " 类的实例");
    }
```

- a的类型必须是 `引用类型` 或 `空类型`(null)，否则就会产生编译时错误。

- 如果b表示的是不可具化的引用类型，也是一个编译时错误

- 如果 a 强转成 b 作为编译时错误，那么 a instanceof b 关系表达式也就同样的产生编译时错误。

- 运行时 a 不为 null，并且a可以被强转成b而不会产生 `ClassCastException` ，那么 instanceof 操作符的结果是 true，否则 false


##### **instanceof、isInstance和isAssignableFrom的区别**



### **final**

使用 `final` 可以定义 ：变量、方法、类

##### **final类**

如果把任何一个类声明为 `final`，则不能有子类



##### **final方法**

如果任何方法声明为 `final`，则不能覆盖它。即不能被重写


##### **final变量**


变量可以被声明为 `final` ，而 `final` 变量只能被赋值一次。如果对 `final` 变量赋值，除非在赋值之前该变量是明确
未赋值的，否则就是一种编译时错误。

一旦 `final` 变量被赋值，那么它就始终持有同一个值。如果一个 `final` 变量持有的是对象的引用，那么该对象的状态可以被修改，但是该变量会始终指向这个对象。这条规则也同样试用于数组，
因为数组也是对象。我们可以修改数组的元素，但是该变量会始终指向该数组。


### **static**

Java的静态形式有5中类型：静态变量、静态方法、静态块、内部静态类和静态接口方法（Java8以上支持）


##### **静态变量**

我们用 `static` 表示变量的级别，一个类中的静态变量，不属于类的对象或者实例。因为该类所有的对象实例共享静态变量，因此他们不具线程安全性。

通常，静态变量常用 `final` 关键字来修饰，表示通用资源或可以被所有的对象所使用。如果静态变量未被私有化，可以用 `类名.变量名` 的方式来使用。


##### **静态方法**

与静态变量一样，静态方法是属于类而不是实例。

在静态方法中只能使用静态变量和调用静态方法。通常静态方法用于想给其他的类使用而不需要创建实例。例如：`Collections`。

Java的包装类和实用类包含许多静态方法。`main()` 方法就是Java程序入口点，是静态方法。

**从Java8以上版本开始接口也可以有静态方法了**


##### **静态代码块**

Java的静态代码块是一组指令，类装载的时候在内存中由Java ClassLoader执行。

静态块常用于初始化类的静态变量。大多时候还用于在类装载时候创建静态资源。

不允许在静态代码块中使用非静态变量。一个类中可以有多个静态块。静态块只在类装载入内存时，执行一次。


```java
static {
    // 在类装载的时候初始化一些资源，如：jdbc驱动等
   
    // 只能访问静态变量和静态方法
}
```


##### **静态类**

Java可以嵌套使用静态类，但是静态类不能用于嵌套的顶层。

让我们来看一个使用静态的样例程序：

```java
public class StaticExample {

    //static block
    static{
        //can be used to initialize resources when class is loaded
        System.out.println("StaticExample static block");
        //can access only static variables and methods
        str="Test";
        setCount(2);
    }

    //multiple static blocks in same class
    static{
        System.out.println("StaticExample static block2");
    }

    //static variable example
    private static int count; //kept private to control it's value through setter
    public static String str;

    public int getCount() {
        return count;
    }

    //static method example
    public static void setCount(int count) {
        if(count > 0) {
            StaticExample.count = count;
        }
    }

    //static util method
    public static int addInts(int i, int...js){
        int sum=i;
        for(int x : js) {
            sum+=x;
        }
        return sum;
    }

    //static class example - used for packaging convenience only
    public static class MyStaticClass{
        public int count;

    }


```

接下来，在测试程序中使用这些静态变量、静态方法和静态类。

```java

    public static void main(String[] args) {
        StaticExample.setCount(5);

        //non-private static variables can be accessed with class name
        StaticExample.str = "abc";
        StaticExample se = new StaticExample();
        System.out.println(se.getCount());
        //class and instance static variables are same
        System.out.println(StaticExample.str +" is same as "+se.str);
        System.out.println(StaticExample.str == se.str);

        //static nested classes are like normal top-level classes
        StaticExample.MyStaticClass myStaticClass = new StaticExample.MyStaticClass();
        myStaticClass.count=10;

        StaticExample.MyStaticClass myStaticClass1 = new StaticExample.MyStaticClass();
        myStaticClass1.count=20;

        System.out.println(myStaticClass.count);
        System.out.println(myStaticClass1.count);
    }
}
```


执行结果如下：

```java
StaticExample static block
StaticExample static block2
5
abc is same as abc
true
10
20
```

可见，静态块代码是最先被执行的，而且是只在类载入内存时执行。


##### **静态import**

一般，Java允许用静态成员使用类引用，从Java1.5开始，我们可以使用静态import而不用类引用。下面是一个简单的静态import的例子。


```java
public class A {

	public static int MAX = 1000;
	
	public static void foo(){
		System.out.println("foo static method");
	}
}
```


```java

import static A.MAX;
import static A.foo;

public class B {

	public static void main(String args[]){
		System.out.println(MAX); //normally A.MAX
		foo(); // normally A.foo()
	}
}
```


第2段代码用了import语句，导入静态类使用import static，后面跟着的是静态成员的全称。

为了导入一个类中的所有的静态成员，可以这样写“import static A.*”，这只有在我们要多次使用一个类的静态变量时，才这样写，但这种写法的可读性不好。


### **synchronized**


### **volatile**