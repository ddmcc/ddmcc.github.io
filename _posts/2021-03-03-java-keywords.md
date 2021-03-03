---

layout: post
title:  "Java中各种关键字"
date:   2021-03-03 16:56:23
categories: Java基础
tags:  transient static final
author: ddmcc
---

* content
{:toc}




### **transient**

Java语言的关键字，变量修饰符，如果用transient声明一个实例变量，当对象存储时，它的值不需要维持。

>这里的对象存储是指，Java的serialization提供的一种持久化对象实例的机制。当一个对象被序列化的时候，transient型变量的值不包括在序列化的表示中，然而非transient型的变量是被包括进去的。使用情况是：当持久化对象时，可能有一个特殊的对象数据成员，我们不想用serialization机制来保存它。为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。

简单点说，就是被transient修饰的成员变量，在序列化的时候其值会被忽略，在被反序列化后， transient 变量的值被设为初始值， 如 int 型的是 0，对象型的是 null。


`ArrayList` 中的保存数据的数组被 `transient` 所修饰。

```java
    transient Object[] elementData; // non-private to simplify nested class access
```


如果如上面的所说，序列化、反序列化后，列表的数据将会为空：

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

