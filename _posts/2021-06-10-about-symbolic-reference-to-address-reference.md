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




在 [类加载过程](http://ddmcc.cn/2021/05/29/jvm-class-file-loading-process/#%E7%B1%BB%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B) 中介绍了 **解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，也就是得到类或者字段、方法在内存中的指针或者偏移量**



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

```typescript
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

```typescript
1: getfield      #2                  // Field address:Lcom/example/demo/Address;
```



其中 **#2** 代表在常量池中的下标，在常量池中找到下标为2的信息：

```typescript
 #2 = Fieldref           #4.#25         // com/example/demo/User.address:Lcom/example/demo/Address;
```



**Fieldref** 是字段的结构表示，具体查看 [虚拟机规范4.4.2](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4.2) ，后面的 **#4.#25** 分别表示 `class_index` 和 `name_and_type_index` ，这两个都是常量池下标（即所在类结构在常量池的下标和字段名与字段类型在常量池的下标），引用着另外两个常量池项。顺着这个方法把所有引用的常量都找出来，可以得到：



```typescript
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



```shell
                    #2：Fieldref   #4.#25
                    /                   \
               #4：Class #28         #25：NameAndType #12:#13
               /                            /              \
    #28：Utf8 com/example/demo/User    #12：Utf8 address   #13：Utf8 Lcom/example/demo/Address;
```





上面是字段的符号引用表示，我们再来看一个方法的符号引用表示，还是在 **printName()** 方法：



```typescript
4: invokevirtual #3                  // Method com/example/demo/Address.printAddress:()V
```



和上面相同，这个虚拟机指令是调用 `Address` 类的 `printAddress` 方法。找到常量池中下标为 **#3** 的项：



```typescript
 #3 = Methodref          #26.#27        // com/example/demo/Address.printAddress:()V
```



``Methodref``  和 `Fieldref` 的结构相同，由 1 个字节的 tag / 2 个字节 class_index / 2 个字节的 name_and_type_index 组成：

> u1 为1个字节，u2为两个字节  [The class File Format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)



```typescript
CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}

CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```



所以可以知道 **#26** 为类符号结构的下标，**#27** 则是方法名和返回值的结构下标。根据上面的方法得到引用关系：



```typescript
 #3 = Methodref          #26.#27        // com/example/demo/Address.printAddress:()V
#26 = Class              #30            // com/example/demo/Address
#30 = Utf8               com/example/demo/Address
#27 = NameAndType        #31:#15        // printAddress:()V
#31 = Utf8               printAddress
#15 = Utf8               ()V    // 这是方法返回值的符号引用
```





**由此可以看出，Class文件中的invokevirtual指令的操作数经过几层引用之后，最后都是由字符串来表示的**



## 直接引用



上面都是说的 `“符号引用”`，下面在看看直接引用：

大致是在类加载的时候会把 `Class` 文件的各个部分分别解析（parse）为 JVM 的内部数据结构。例如说类的元数据记录在 `Class` 结构体里，每个方法的元数据记录在各自的 `methodblock` 结构体里等等

在刚加载好一个类的时候，`Class` 文件里的常量池和每个方法的字节码（Code属性）会被基本原样的拷贝到内存里先放着，也就是说仍然处于使用 “符号引用” 的状态，直到真的要被使用到的时候才会被解析（resolve）为直接引用

假定我们要第一次执行到 **printName()** 方法里调用 **printAddress()** 方法的那条invokevirtual指令了，此时 JVM 会发现该指令尚未被解析（resolve），所以会先去解析一下。通过其操作数所记录的常量池下标找到常量池项 **#3**，发现该常量池项也尚未被解析（resolve），于是进一步去解析一下。通过 `Methodref` 所记录的 `class_index` 找到类名，进一步找到被调用方法的类的 `Class` 结构体，然后通过`name_and_type_index` 找到方法名和方法返回类型，到找到的 `Class` 结构体上记录的方法列表里找到匹配的那个 `methodblock`，最终把找到的 `methodblock` 的指针写回到常量池项 **#3** 里

**也就是说，原本常量池项 #3 在类加载后的运行时常量池里的内容跟Class文件里的一致，只是解析后它的内容变了，由原来的字符串表示的 “符号引用” 变为一个能直接找到 Java 方法元数据的 methodblock 了。这里的 methodblock 就是一个  “直接引用”**，**#3** 常量项保存着 `methodblock` 直接指针



解析好常量池项 **#3** 之后回到 `invokevirtual` 指令的解析，在解析后虚拟机指令从 `invokevirtual` 改写为 `invokevirtual_quick`表示该指令已经解析完毕。原本存储操作数的 **2** 字节空间现在分别存了 **2个1** 字节信息，第一个是 **虚方法表的下标（vtable index）**，第二个是 **方法的参数个数（args_size）**。这两项信息都由前面解析常量池项 **#3** 得到的 methodblock 读取而来



```typescript
invokevirtual_quick vtable_index=5, args_size=1
```



在 `Address` 类的虚方法表就会有：

```typescript
[0]: java.lang.Object.hashCode:()I
[1]: java.lang.Object.equals:(Ljava/lang/Object;)Z
[2]: java.lang.Object.clone:()Ljava/lang/Object;
[3]: java.lang.Object.toString:()Ljava/lang/String;
[4]: java.lang.Object.finalize:()V
[5]: com.example.demo.Address.printAddress:()V
```



`User` 类的虚方法表则：

```typescript
[0]: java.lang.Object.hashCode:()I
[1]: java.lang.Object.equals:(Ljava/lang/Object;)Z
[2]: java.lang.Object.clone:()Ljava/lang/Object;
[3]: java.lang.Object.toString:()Ljava/lang/String;
[4]: java.lang.Object.finalize:()V
[5]: com.example.demo.User.printName:()V
```



在执行 `invokevirtual_quick` 调用 **printAddress()** 时，通过对象引用查找到虚方法表后，从中取出第 **#5** 项的`methodblock`，就可以找到实际应该调用的目标然后调用过去了



**直接引用是运行时所能直接使用的形式，即可以表现为直接指针（如上面#3常量项解析为 methodblock），也可能是其它形式（如invokevirtual_quick 指令里的vtable的下标），关键点不在于是否为直接指针或其它偏移量，在于jvm能不能直接使用**



## jvm中多态的实现

假如有一个类 `Staff` 继承了类 `User` ，并重写了 `printName()方法` ，那么类 Staff 的虚方法表就会有：



```typescript
[0]: java.lang.Object.hashCode:()I
[1]: java.lang.Object.equals:(Ljava/lang/Object;)Z
[2]: java.lang.Object.clone:()Ljava/lang/Object;
[3]: java.lang.Object.toString:()Ljava/lang/String;
[4]: java.lang.Object.finalize:()V
[5]: com.example.demo.User.otherMethod:()V    // 继承而来的方法
[5]: com.example.demo.Staff.printName:()V   // 重写了父类的方法
[6]: com.example.demo.Staff.printStaff:()V  // 子类自己的方法
```



虚方法表中方法存放顺序是先父类-再子类的顺序，所有的类都继承自 Object 类，所以表中 **最先存放的是Object类的方法**，接下来是该类的父类 **User** 的方法，最后是该类本身的方法。当调用的时候通过传入的实际指向this来确定方法的接收者（receiver），动态绑定（分派）具体对象的类型（因为是多态，所以指向的是子类对象的类型），继而找到方法区里子类的方法表，根据偏移量找到子类方法表对应的方法

 



## 参考

[JVM里的符号引用如何存储？](https://www.zhihu.com/question/30300585)
