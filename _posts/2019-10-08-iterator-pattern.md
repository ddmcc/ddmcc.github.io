---
layout: post
title:  "设计模式之迭代器模式"
date:   2019-10-08 21:44:26
categories: 设计模式
tags:  设计模式 迭代器模式
author: ddmcc
---

* content
{:toc}


## 定义

 　　提供一种方法顺序访问一个聚合对象中的各个元素，而又不暴露其内部的表示

 　　**聚合对象** 是什么呢？可以看成将多个对象用某种数据结构聚集在一起。可以是一个数组Array，可以是一个列表List，也可以是一个散列表Map。




## 实例

 　　教育局要求每个学校提供一个接口，要求该接口返回每个学校的学生列表。由于没有统一的接口标准，所以该接口由学校自己设计实现。A校返回的是一个ArrayList，
 B校则是一个Array。具体的代码如下：
 
 ```java
/**
 * @author ddmcc
 */

@Data
@AllArgsConstructor
public class Student {

    private String name;

    private int age;

    private String sex;

    private String address;

}
```


```java
/**
 * A校返回ArrayList
 *
 * @author ddmcc
 */
public class ListSchool {

    private ArrayList<Student> list = new ArrayList<>();

    public ListSchool addStudent(String name, int age, String sex, String address) {
        list.add(new Student(name, age, sex, address));
        return this;
    }

    public ArrayList<Student> getStudentList() {
        return this.list;
    }

}
```

```java
/**
 * B校返回Array
 *
 * @author ddmcc
 *
 *
 * B校返回Array
 *
 * @author ddmcc
 */
public class ArraySchool {

    private Student[] students;

    private static final int MAX_LENGTH = 100;

    private int position = 0;

    public ArraySchool() {
        students = new Student[MAX_LENGTH];
    }

    public ArraySchool addStudent(String name, int age, String sex, String address) {
        if (position < MAX_LENGTH) {
            students[position] = new Student(name, age, sex, address);
            position += 1;
        }
        return this;
    }

    public Student[] getStudentArray() {
        return this.students;
    }

}
```
---

 　　由于两个学校返回的类型并不相同，教育局要遍历学生列表就不得不针对两个列表进行实现。遍历代码如下：
 
---
 
```java
/**
 * 教育局
 *
 * @author ddmcc
 */
public class EducationBureau {


    public void printStudent() {
        ListSchool listSchool = new ListSchool();
        listSchool.addStudent("大锤", 22, "男", "上海")
                .addStudent("二狗", 23, "男", "北京")
                .addStudent("菜花", 22, "女", "东北");

        ArraySchool arraySchool = new ArraySchool();
        arraySchool.addStudent("铁蛋", 24, "男", "厦门")
                .addStudent("铁锤", 23, "男", "上海");

        ArrayList<Student> arrayList = listSchool.getStudentList();
        for (int i = 0; i < arrayList.size(); i++) {
            System.out.println(arrayList.get(i));
        }

        Student[] array = arraySchool.getStudentArray();
        for (int i = 0; i < array.length; i++) {
            System.out.println(array[i]);
        }
    }
}
```

 　　从上面的代码可以看出由于两个学校返回的类型不一致，所以写了两遍的循环代码。其实不单单是遍历，如有其它对两个列表进行操作，也必须进行不同的实现。
 而且教育局还知道了两个学校存储对象的方式，这也不是学校想要的。后期还有学校加入，返回的又是不同的数据类型，还得再针对实现。。。如果我们让学校自己提供遍历的方法或遍历的工具呢？
 这样我们就不需要去考虑数据结构的问题了。但是这样又有新的问题，没有统一的标准，可能A校提供的方法叫`printList` ,B校提供的确叫`printArray`。
 
  　　封装代码变化部分是我们代码设计的一个原则。针对上面的问题，也就是封装对象的遍历。现在由我们来提供遍历的工具（Iterator迭代器），由各自学校来实现这个迭代器，教育局在拿着迭代器自己去遍历，
  这样也就可以解决了不同学校返回不同列表的问题，这样做也封装了遍历，教育局现在并不知道学校是如何维护对象的，也不知道迭代器是如何实现的。
  
  
```java
/**
 * 自定义迭代器接口
 *
 * @author ddmcc
 */
public interface Iterator<T> {

    boolean hasNext();

    T next();

}
```

  　　上面是教育局提供的接口，由学校去实现。就像Java提供的JDBC接口，mysql，oracle去实现一样。。。下面是A校提供的迭代器：
  
  
```java
/**
 * A提供的列表迭代器
 * 
 * @author ddmcc
 */
public class ArrayListIterator<T> implements Iterator<T> {

    /**
     * 下一个返回元素索引
     */
    private int cursor = 0;

    /**
     * 列表数据
     */
    private ArrayList<T> list;

    public ArrayListIterator(ArrayList<T> list) {
        this.list = list;
    }

    @Override
    public boolean hasNext() {
        return cursor < list.size();
    }

    @Override
    public T next() {
        T t = list.get(cursor);
        cursor += 1;
        return t;
    }
}
```

  　　这样A校就不在需要提供获取列表的接口，而提供一个返回迭代器的接口：
  
  
  > ~~public ArrayList<Student> getStudentList() {~~
  >     ~~return this.list;~~
  > ~~}~~
  >    
  > public Iterator<Student> getIterator() {
  >     return new ArrayListIterator<>(this.list);
  > }


  　　再利用迭代器进行遍历操作

```java
public void printStudent() {
    ListSchool listSchool = new ListSchool();
    listSchool.addStudent("大锤", 22, "男", "上海")
            .addStudent("二狗", 23, "男", "北京")
            .addStudent("菜花", 22, "女", "东北");
        
    Iterator<Student> iterator = listSchool.getIterator();
    while (iterator.hasNext()) {
        Student student = iterator.next();
        System.out.println(student.toString());
    }
}
```

  　　会发现现在教育局并不知道A校是怎么存储学生对象的，也不知道迭代器是怎么实现遍历的。。。让各个对象更专注在自己所应该专注的地方上。
  
  　　下面为B校代码：
  
```java
/**
 *   迭代器
 * 
 * @author ddmcc
 */
public class ArrayIterator<T> implements Iterator<T> {

    private int position = 0;
    private T[] items;

    public ArrayIterator(T[] items) {
        this.items = items;
    }

    @Override
    public boolean hasNext() {
        return position < items.length && items[position] != null;
    }

    @Override
    public T next() {
        T t = items[position];
        position += 1;
        return t;
    }
}


/**
 * B校返回Array
 *
 * @author ddmcc
 */
public class ArraySchool {

    private Student[] students;

    private static final int MAX_LENGTH = 100;

    private int position = 0;

    public ArraySchool() {
        students = new Student[MAX_LENGTH];
    }

    public ArraySchool addStudent(String name, int age, String sex, String address) {
        if (position < MAX_LENGTH) {
            students[position] = new Student(name, age, sex, address);
            position += 1;
        }
        return this;
    }

    // 注意这个方法名
    public Iterator<Student> iterator() {
        return new ArrayIterator(students);
    }

}


/**
 * 教育局
 *
 * @author ddmcc
 */
public class EducationBureau {


    public void printStudent() {
        ArraySchool arraySchool = new ArraySchool();
        arraySchool.addStudent("铁蛋", 24, "男", "厦门")
                .addStudent("铁锤", 23, "男", "上海");

        Iterator<Student> iterator = arraySchool.iterator();
        while (iterator.hasNext()) {
            Student student = iterator.next();
            System.out.println(student.toString());
        }
    }
}
```

  　　从上面的代码会发现其实还存在一些问题，比如两个学校因为没有一个标准，所以提供的迭代器的方法并不一致。用户同时依赖了具体的类ListSchool和ArraySchool。
所以还可以创建一个学校接口，提供一个获取迭代器的方法，用户也就可以多态编程了。

```java
/**
 * @author ddmcc
 */
public interface School {

    Iterator iterator();

    School addStudent(String name, int age, String sex, String address);

}


/**
 * A校返回ArrayList
 *
 * @author ddmcc
 */
public class ListSchool implements School{

    private ArrayList<Student> list = Lists.newArrayList();

    @Override
    public School addStudent(String name, int age, String sex, String address) {
        list.add(new Student(name, age, sex, address));
        return this;
    }

    @Override
    public Iterator iterator() {
        return new ArrayListIterator<>(this.list);
    }
}


/**
 * 教育局
 *
 * @author ddmcc
 */
public class EducationBureau {
    
    public void printStudent() {
        School arraySchool = new ArraySchool();
        arraySchool.addStudent("铁蛋", 24, "男", "厦门")
                .addStudent("铁锤", 23, "男", "上海");

        Iterator<Student> iterator = arraySchool.iterator();
        while (iterator.hasNext()) {
            Student student = iterator.next();
            System.out.println(student.toString());
        }
    }
}
```

## 上面代码的类图


## JDK中的迭代器


## 总结

- 创建一个迭代器接口用于封装聚合对象的遍历，由聚合对象提供具体的遍历方式
- 将用户与具体的聚合对象解耦，无须关注其内部实现。
- 便于后期扩展，其实更改聚合对象，也无须更改处理对象的逻辑部分。只需提供不同的迭代器即可。