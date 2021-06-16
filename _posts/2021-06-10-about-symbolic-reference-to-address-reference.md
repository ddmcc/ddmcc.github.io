---

layout: post
title:  "关于类解析阶段符号引用转为直接引用的过程"
date:   2021-06-10 16:56:23
categories: jvm
tags:  jvm
author: ddmcc
---

* content
{:toc}




在 [类加载过程]() 中介绍了 **解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，也就是得到类或者字段、方法在内存中的指针或者偏移量**



那么什么是 **`符号引用`**? 什么又是 **`直接引用`**？二者之间又有什么关联？字段、方法在内存中又是如何存储和访问的？下面就通过 `class` 文件来具体分析



## 符号引用



有下面两个类`User` 和 `Address`：



```java
/**
 * @author jiangrz
 */
public class User {

    private boolean sex;

    private String name;

    private Integer age;

    private Address address;
    
    public void printName() {
        address.printAddress();
    }
}


/**
 * @author jiangrz
 */
public class Address {

    private String province;

    public void printAddress() {

    }
}
```



---

其中 **User** 类编译出来的 `class` 文件的文本形式如下：

---

```shell
Classfile /Users/jiangrz/workspace/other/demo/target/classes/com/example/demo/User.class
  Last modified 2021年6月14日; size 553 bytes
  SHA-256 checksum d9a9b9e80018881155b444972b9bcc32d83988780b8fbdf5e717ca49c1a5b944
  Compiled from "User.java"
public class com.example.demo.User
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #4                          // com/example/demo/User
  super_class: #5                         // java/lang/Object
  interfaces: 0, fields: 4, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #5.#24         // java/lang/Object."<init>":()V
   #2 = Fieldref           #4.#25         // com/example/demo/User.address:Lcom/example/demo/Address;
   #3 = Methodref          #26.#27        // com/example/demo/Address.printAddress:()V
   #4 = Class              #28            // com/example/demo/User
   #5 = Class              #29            // java/lang/Object
   #6 = Utf8               sex
   #7 = Utf8               Z
   #8 = Utf8               name
   #9 = Utf8               Ljava/lang/String;
  #10 = Utf8               age
  #11 = Utf8               Ljava/lang/Integer;
  #12 = Utf8               address
  #13 = Utf8               Lcom/example/demo/Address;
  #14 = Utf8               <init>
  #15 = Utf8               ()V
  #16 = Utf8               Code
  #17 = Utf8               LineNumberTable
  #18 = Utf8               LocalVariableTable
  #19 = Utf8               this
  #20 = Utf8               Lcom/example/demo/User;
  #21 = Utf8               printName
  #22 = Utf8               SourceFile
  #23 = Utf8               User.java
  #24 = NameAndType        #14:#15        // "<init>":()V
  #25 = NameAndType        #12:#13        // address:Lcom/example/demo/Address;
  #26 = Class              #30            // com/example/demo/Address
  #27 = NameAndType        #31:#15        // printAddress:()V
  #28 = Utf8               com/example/demo/User
  #29 = Utf8               java/lang/Object
  #30 = Utf8               com/example/demo/Address
  #31 = Utf8               printAddress
{
  public com.example.demo.User();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/example/demo/User;

  public void printName();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field address:Lcom/example/demo/Address;
         4: invokevirtual #3                  // Method com/example/demo/Address.printAddress:()V
         7: return
      LineNumberTable:
        line 19: 0
        line 20: 7
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       8     0  this   Lcom/example/demo/User;
}
```



---

在 class 文件中有一段 **Constant pool**  信息，里面存储的该Class文件里的大部分常量的内容，如：类和接口的全限定名、字段的名称和描述符以及方法的名称和描述符



看到 **printName()** 方法有一条字节码指令：

```sh
1: getfield      #2                  // Field address:Lcom/example/demo/Address;
```



其中 **#2** 代表在常量池中的下标，在常量池中找到下标为2的信息：

```sh
 #2 = Fieldref           #4.#25         // com/example/demo/User.address:Lcom/example/demo/Address;
```



**Fieldref** 是字段的结构表示，具体查看 [虚拟机规范4.4.2](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4.2) ，后面的 **#4.#25** 分别表示 `class_index` 和 `name_and_type_index` ，这两个都是常量池下标（即所在类结构在常量池的下标和字段名与字段类型在常量池的下标），引用着另外两个常量池项。顺着这个方法把所有引用的常量都找出来，可以得到：



```sh
 #2 = Fieldref           #4.#25         // com/example/demo/User.address:Lcom/example/demo/Address;
 #4 = Class              #28            // com/example/demo/User
#28 = Utf8               com/example/demo/User
#25 = NameAndType        #12:#13        // address:Lcom/example/demo/Address;
#12 = Utf8               address
#13 = Utf8               Lcom/example/demo/Address;
```



上面的常量池项对应文件结构表示为：

- Class：`CONSTANT_Class_info`
- Utf8：`CONSTANT_Utf8_info`
- NameAndType：`CONSTANT_NameAndType_info`



**标记为 Utf8 的常量池项在 class 文件中实际为 CONSTANT_Utf8_info，是以 UTF-8 编码的字符串文本** ，比如 `#12 = Utf8` 实际值为 `address`

这就是`class` 文件里的 **符号引用** 实际存储的值：带有类型（tag） /  结构（符号间引用层次）的字符串。将上面的引用关系画成树：



```sh
                    #2：Fieldref   #4.#25
                    /                   \
               #4：Class #28         #25：NameAndType #12:#13
               /                            /              \
    #28：Utf8 com/example/demo/User    #12：Utf8 address   #13：Utf8 Lcom/example/demo/Address;
```





上面是字段的符号引用表示，我们再来看一个方法的符号引用表示，还是在 **printName()** 方法：



```xaml
4: invokevirtual #3                  // Method com/example/demo/Address.printAddress:()V
```





