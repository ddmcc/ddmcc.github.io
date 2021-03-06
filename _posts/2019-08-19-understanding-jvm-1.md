---
layout: post
title:  "jvm之运行时数据区"
date:   2019-08-19 21:18:50
categories: jvm
tags:  jvm
author: ddmcc
---

* content
{:toc}


 　　Java虚拟机运行Java程序时会将所管理的内存分为不同的数据区域。这些区域的作用以及生命周期都不同。有的随虚拟机进程创建而创建，有的则随线程的启动和结束而建立和销毁。下图是 **JDK1.7**的运行时数据区




## 1.7运行时数据区

![](https://i.loli.net/2019/08/19/zjiESkyLQuWcCHD.png)


如上图所示，可分为线程共享（ **堆**， **方法区**）和线程私有数据区（ **虚拟机栈**， **本地方法栈**， **程序计数器**）。


### 程序计数器

　　程序计数器是一块较小的空间，可以看成是 **当前线程**所执行字节码的行号指示器。字节码解释器就是通过改变计数器的值来选取下一个指令是什么。Java多线程应用是通过
线程切换竞争CPU时间片实现的。在单核处理器中，同一确定时刻，只会执行一个线程中的指令。所以为了切换后能够恢复到正确的位置，每个线程都需要独自保存程序计数器。
各线程计数器互不影响，独立存储。

　　如果线程执行的是一个Java方法，那么计数器记录的是字节码指令的地址；如果执行的是native方法，那么计时器为undefined。 **此区域是唯一一块没有规定会发生内存溢出（OutOfMemoryError）的区域**


### Java虚拟机栈

　　和程序计数器一样Java虚拟机栈也是线程私有的，生命周期和线程相同。虚拟机栈描述的是Java方法的执行的内存模型：每个方法在执行的时候都会创建一个 **栈帧**，用于保存 **局部变量**， **操作数栈**， **动态链接**， **方法出口等**。
每一个方法的执行都对应着一个栈帧在虚拟机栈的的入栈与出栈过程。


　　我们平常说的`栈内存`就是指的虚拟机栈的`局部变量表`，局部变量表保存着 **8大基本数据类型**和 **对象引用变量（`可能指向堆内存的地址，也可能指向句柄地址`）**以及 **returnAddress（返回类型？）类型** （指向一条字节码指令的地址）。


　　64长度的long和double会占用两个局部变量空间，其他都只会占用一个。当进入一个方法时，这个方法需要在帧中分配多少的空间是确定的，在方法执行期间不会改变局部变量表大小，即 **栈内存不会动态改变**。


　　在这区域规定了两种异常：当线程请求的栈深度大于虚拟机的所允许深度，将抛出 **StackOverFlowError**（内存泄露）异常；如果虚拟机可以动态扩展（Java虚拟机栈可以设置长度），如果动态扩展无法申请到足够的长度，那么将抛出
**OutOfMemoryError**（内存溢出）异常。


### 本地方法栈

　　本地方法栈的作用与Java虚拟机栈类似，它们的区别是虚拟机栈为虚拟机执行Java方法（字节码）服务，而本地方法栈为虚拟机使用Native方法。在Sun HotSpot中（JDK使用的虚拟机），将
Java虚拟机栈和本地方法栈合二为一。在本地方法栈中也会抛出 **StackOverFlowError**（内存泄露）异常和 **OutOfMemoryError**（内存溢出）异常。


### 堆内存


　　堆内存（Heap Memory）是虚拟机所管理的较大一块内存。也是被所有线程所共享的，在 **虚拟机启动而创建**。此内存区域的作用就是保存对象实例， 几乎所有的对象实例都在这里分配内存， **包括对象和数组**。


　　Java堆也是垃圾回收器管理的主要区域。从辣鸡回收角度来说，由于基本都采用 **分代收集算法**，所以在Java堆中还可以分为： **新生代和老年代**：再细致一点有`Eden空间`、`From Survivor`、`To Survivor`空间等。
从内存分配角度来看，线程共享的堆内存可能划分出几个线程私有的分配缓存区。但 **其存储的还是对象实例，只不过为了更好的回收内存或更快的分配内存**。


　　堆内存可以处于在一块地址不连续的内存空间上。其大小可以通过-Xmx和-Xms控制，如果堆中没有内存完成实例分配，并且内存大小不可扩展那么将抛出OutOfMemoryError（内存溢出）异常。


### 方法区

　　`方法区`和`Java堆`一样，也是被所有线程共享的一块区域。用于存储已经被虚拟机加载的`类信息`，`常量`，`静态变量`，`Jit编译后的代码`，`动态代理生成的字节码文件等数据`。在HotSpot虚拟机中，方法区还被称为`永久代`，他们两个的关系可以看成是一种实现的关系。
`方法区`是Java虚拟机的规范，而`永久代`是HotSpot对这个规范的实现。其它的虚拟机中是没有`永久代`这个东西的。如IBMJ9等。


　　`永久代`有 **-XX:MaxPermSize大小限制** 会抛出OOM异常。 **在JDK1.7中，把`字符串常量池`从永久代中移出，放到了`Java堆`中。而在JDK1.8中，已经取消了`永久代`，而采用本地内存（Native Memory）来实现方法区了**，下面会详细介绍。


　　 **方法区分配不要求连续的内存，可以选择固定的大小，还可以扩展，并且可以选择不实现垃圾收集** 。当方法区无法满足内存分配时，将抛出 **OutOfMemoryError**（内存溢出）异常。

---
#### 运行时常量池

　　`运行时常量池`是方法区的一部分。Class文件中（字节码文件.class）除了有类的版本，字段，方法，接口等信息外，还有`常量池`，用于存储编译期生成的各种 **字面量**（基本类型数据，字符串）和 **符号引用** （即不知道引用的实际地址或不知道对象的值，而用 **符号** 来代替），这部分内容（常量池）将在类加载后进入 **运行时常量池** 存放。
一般， **直接引用** （直接指向目标内存地址，指向可以找到目标的中间句柄，相对偏移量）也存储在运行时常量池中。 **运行时常量池相对于Class文件常量池另一个特征是具备动态性** 。即常量不一定在编译时产生，也就是并非在`Class常量池`中的内容才能进入`运行时常量池`。运行时也可能加入新的常量到常量池。如String类的intern()方法。


　　`运行时常量池`在无法申请到内存时会抛出 **OutOfMemoryError**（内存溢出）异常。


### 直接内存

　　对HotSpot来说，不受GC管理的内存都是`Native Memory`（即除了虚拟机用的内存，宿主机的其他内存都统称本地内存？）。在JDK1.4中引入NIO，它可以用Native函数直接分配堆外内存（本地内存），然后通过一个存储在Java堆中的`DirectByteBuffer`对象作为这块内存的引用进行操作。
**所以"Direct Memory"可以看成：Java程序通过一组特定的API访问`Native Memory`，API就是`DirectByteBuffer`，被访问的内存就是`Direct Memory`** 。

　　直接内存也会受到本机内存的限制，动态扩展也可能出现 **OutOfMemoryError**（内存溢出）异常。