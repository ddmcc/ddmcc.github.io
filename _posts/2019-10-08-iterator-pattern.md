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
 */
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
        if (students.length < MAX_LENGTH) {
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

 　　从上面的代码可以看出由于两个学校返回的类型不一致，所以写了两遍的循环代码，而且