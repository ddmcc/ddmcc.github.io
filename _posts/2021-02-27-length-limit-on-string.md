---

layout: post
title:  "String字符串的长度限制"
date:   2021-02-27 16:56:23
categories: Java基础
tags:  String
author: ddmcc
---

* content
{:toc}




### **字符串的长度限制**

最近和第三方进行系统对接，约定是我们这边将文件的转成BASE64编码后，进行传输。对方反馈将BASE64字符串转成文件后，文件损坏打不开，怀疑是BASE64字符串的问题。我在本地尝试将base64字符串嵌入到img标签，
图片正常显示，说明字符串是没问题的。然后尝试将字符串保存为文件：

```java
String a = "data:image/jpg;base64,/9j/4AAQSkZJRgABAQAASABIAAD/4QBARXhpZgAA......";  // 7w多个字符
// 省略转文件代码...
```

点击运行时，在编译阶段提示 `字符串常量` 过长提示报错：

```java
Error:(6, 20) java: constant string too long
```


##### **常量池限制**

在 [Java语言规范 3.10.5](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.5) 中定义，被 `""` 括起来的字符称为 `字符串字面常量`，通常叫它字面量。
字面量会被放到 `字符串常量池` 中。

在.java文件编译成.class的过程中，编译器会进行类型检查，数值范围检查等一系列的检查操作。并且编译成的字节码文件也是有一定格式的，才能被虚拟机正确的执行。

根据 [Java虚拟机规范 4.4](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4) 常量池中定义，`CONSTANT_String_info` 用于表示 java.lang.String 类型的常量对象，格式如下：

```java
CONSTANT_String_info {
    u1 tag;
    u2 string_index;
}
```

其中，string_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必须是 CONSTANT_Utf8_info 结构，表示一组 Unicode 码点序列，这组 Unicode 码点序列最终会被初始化为一个 String 对象。

CONSTANT_Utf8_info 结构用于表示字符串常量的值：

```java
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```

其中，`length` 代表数组 `bytes[]` 的长度，其类型为u2，`bytes[]` 是表示字符串值的byte数组。

在规范 [Java虚拟机规范 4](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html) 中说明：u2表示两个字节的无符号数，那么1个字节有8位，2个字节就有16位。
在[Java虚拟机规范 4.11](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.11) 中也说明：字段和方法名称、字段和方法描述符以及其他常量字符串值（包括由ConstantValue（§4.7.2）属性引用的值）的长度被常量信息结构（§4.4.7）的16位无符号长度项限制为65535个字符。

16位无符号数可表示的最大值位2^16 - 1 = 65535

也就是说，Class文件中常量池的格式规定了，其字符串常量的长度不能超过65535。

尝试定义长度为65535的字符串，来证实以上：

```java
    public static void main(String[] args) {
        String a = "a111111111111111111111111111111111111111111111111111111.............";  // 65535个
        System.out.println(a);
    }
```


尝试使用javac编译，同样会得到"错误: 常量字符串过长"，那么原因是什么呢？

其实在编译时是有对字符串常量长度进行检查的。在javac代码中可以找到，具体在 `Gen` 类 `checkStringConstant` 方法：

```java
    private void checkStringConstant(DiagnosticPosition var1, Object var2) {
        if (this.nerrs == 0 && var2 != null && var2 instanceof String && ((String)var2).length() >= 65535) {
            this.log.error(var1, "limit.string", new Object[0]);
            ++this.nerrs;
        }
    }
```

![markdown](https://ddmcc-1255635056.file.myqcloud.com/fbc43a83-00ea-4c66-a30a-c20115c8217d.png)

可以看到当类型为String，并且长于 `>=` 65535 时，就会导致编译失败。尝试65534个字符，则可以正常编译通过。

那为什么是 `65534` ，而不是 `65535` 呢？在 [Java虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7.3) 也有做解释：

>if the Java Virtual Machine code for a method is exactly 65535 bytes long and ends with an instruction that is 1 byte long, then that instruction cannot be protected by an exception handler. A compiler writer can work around this bug by limiting the maximum size of the generated Java Virtual Machine code for any method, instance initialization method, or static initializer (the size of any code array) to 65534 bytes.


这其实是为了弥补java虚拟机的一个bug，所以将长度限制为65534

![markdown](https://ddmcc-1255635056.file.myqcloud.com/7bc76521-ce8b-47e7-82bd-70d1be67d1fc.png)


##### **运行时限制**

上面提到的这种String长度的限制是编译期的限制，也就是使用String s= ""; 这种字面值方式定义的时候才会有的限制。

String在运行期有没有限制呢，答案是有的，进入String类源码。里面有许多重载的构造方法，其中有的是支持用户输入长度的

```java
    public String(byte bytes[], int offset, int length) 

    public int length() {
        return value.length;
    }
```

`length` 使用 `int` 类型来定义的，也就说 `length` 最大长度也就是 Integer.MAX_VALUE。

int 是一个 32 位变量类型，取正数部分来算的话，他们最长可以有 `2^31-1 = 2147483647 个` 。这个值约等于4G，在运行期，如果String的长度超过这个范围，就可能会抛出异常。(在jdk 1.9之前）

```java
-2^31 ~ 2^31-1 = -2147483648 ~ 2147483647 个 16-bit Unicodecharacter

2147483647 * 16 = 34359738352 位
34359738352 / 8 = 4294967294 (Byte)
4294967294 / 1024 = 4194303.998046875 (KB)
4194303.998046875 / 1024 = 4095.9999980926513671875 (MB)
4095.9999980926513671875 / 1024 = 3.99999999813735485076904296875 (GB)
```


### **总结**

字符串有长度限制，在编译期，要求字符串常量池中的常量不能超过65535，并且在javac执行过程中控制了最大值为65534。

在运行期，长度不能超过Int的范围，否则会抛异常