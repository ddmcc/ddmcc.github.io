---

layout: post
title:  "Java虚拟机类加载过程"
date:   2021-05-29 16:56:23
categories: jvm
tags:  jvm ClassLoader
author: ddmcc
---

* content
{:toc}




## 类加载过程

虚拟机把16进制描述类的 `.class` 文件加载到内存，并对数据进行校验、解析和初始化等操作，最终变为可以被虚拟机使用的 Java 类型，这就是虚拟机的类加载机制

> 文件内容是按照 [类文件结构](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html) 规定的存储结构存储的



如下图为编译后的 class 文件：

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/0de426ed-974d-4edb-89dc-6446f42e690e.png)



---



类加载过程分为：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）等5个阶段。其中准备、验证、解析三个阶段统称为链接（Linking）



---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/eeaf25b4-5baa-43f9-8b2d-87d60e572026.png)

---



### 加载阶段（loading）

加载（创建）阶段主要是通过类的全限定名来查找这个类的文件，并读取为二进制字节流。将静态的类文件存储结构转化为运行时数据结构，并在内存中生成一个代表这个类的 java.lang.Class 对象，作为这个类的数据访问入口

>将静态的类文件存储结构转化为运行时数据结构：意思就是在 .java 被编译成 .class 文件后，class文件是按照严格的存储结构进行存储的，比如一个字符串常量



#### 类加载器

如果某个类不是数组类，类文件则通过 `类加载器` 进行加载。类加载器主要有两种：Java虚拟机提供的引导类加载器和用户自定义的类加载器



**Java虚拟机提供的：**

- 启动类（或根类）加载器（Bootstrap ClassLoader）

  这个加载器不是一个Java类，而是由虚拟机底层的 c/c++ 实现，负责将存放在 JAVA_HOME 下lib目录中的类库，比如 rt.jar。因此，启动类加载器不属于 Java 类库，无法被Java程序直接引用

  

- 扩展类加载器（ExtClassLoader）

​	  由 sun.misc.Launcher$ExtClassLoader 实现，负责加载 JAVA_HOME 下 lib.ext 目录下的，或者被 java.ext.dirs 系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器

- 应用类加载器（AppClassLoader）

  由 sun.misc.Launcher$AppClassLoader 实现的。由于这个类加载器是 ClassLoader 中的 getSystemClassLoader 方法的返回值，所以也叫系统类加载器。它负责加载用户类路径上所指定的类库，可以被直接使用。如果未自定义类加载器，默认为该类加载器



加载器对应加载路径：

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/295160ee-e04f-4989-9d66-7c2cc6b461f4.png)

---





**用户自定义类加载器：**

每个用户定义的类加载程序都是抽象类 `ClassLoader` 子类的实例。用户定义的类加载程序可用于创建源自用户定义源的类。例如，类可以通过网络下载、实时生成或从加密文件中提取



> 对于加载器的初始化：除启动类加载器外，扩展类加载器和应用类加载器都是通过类sun.misc.Launcher进行初始化，而Launcher类则由根类加载器进行加载



#### 双亲委派加载机制

**当一个类加载器接收到类加载请求时，会先请求其父类加载器进行递归，如果父类能够找到该类，则由父加载器加载；当父类加载器无法找到该类时（根据类的全限定名称），子类加载器才会尝试去加载**



加载类的源码实现在 `ClassLoader#loadClass` ，核心源码如下：

```java
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        // 进行类加载操作时首先要加锁，避免并发加载
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查类是否已经加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    // 如果父加载器不为空，递归调用
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        // 如果当前类没有被加载且父加载器为null，则请求根类加载器进行加载操作
                        // 这里面调用本地方法加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                }

                if (c == null) {
                    // 如果父类加载器加载失败，则由当前类加载器进行加载，自定义加载器实现该方法进行加载
                    long t1 = System.nanoTime();
                    c = findClass(name);
                    // 其它操作...
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

---

这种委派机制是以 **组合的方式** 实现的，如我们自定义一个 `MyClassLoader` 继承 `ClassLoader` ，并且不指定父加载器：

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/3413ea4e-f2ac-40f0-a5d9-b966fcad459a.png)



---



可以看到不同加载器通过组合的方式实现。经过观察上图也产生两个疑惑：

1. 我并没有给自定义加载器 `MyClassLoader` 指定父加载器，为什么会有父加载器，并且是 AppClassLoader？
2. AppClassLoader 父类加载器就是ExtClassLoader吗？



针对 **问题2**，在 `Launcher` 类初始化时，先初始化 `ExtClassLoader` ，然后在初始化 `AppClassLoader` 时把 `ExtClassLoader` 作为父加载器传入。下面为初始化源码：



```java
    public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
            // 初始化创建ExtClassLoader
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
            // 初始化AppClassLoader将ExtClassLoader作为父加载器传入
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }
        // 设置AppClassLoader为线程上下文loader
        Thread.currentThread().setContextClassLoader(this.loader);
        // 其它安全初始化
    }
```

---

而 **问题1** 则要看 `ClassLoader` 的无参构造方法，在构造方法中会调用 `getSystemClassLoader()` 方法取获取系统的ClassLoader，内部实现其实就是取获取 `Launcher` 类的 loader 属性，也就是上面问题2代码片段中的 `this.loader = ...`，**所以自定义loader未指定父加载器，默认就为 `AppClassLoader`**



通过以上的分析，我们也就可以知道加载器之间的关系为：

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/bba3babd-3753-4753-9f81-5538b3a2377f.png)



---





### Linking 链接阶段

JVM规范规定：

- 类或接口在链接之前完全加载
- 类或接口在初始化之前会对其进行完全验证和准备



#### 验证阶段（Verification）

验证的目的主要为了确保文件中的表示满足 [静态语法或结构的约束](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.9) ，及安全性校验。主要包括文件格式验证、元数据验证、字节码验证、符号引用验证



- 文件格式验证：验证字节流是否符合Class文件格式的规范；比如，是否以魔术0xCAFEBABE开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。只有验证通过才会进入方法区进行存储
- 元数据验证：对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言规范的要求；比如，是否有父类（除Object类）、父类是否为final修饰、是否实现抽象方法或接口、重载是否正确等
- 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。比如，保证数据类型与指令正常配合工作、指令不会跳转到方法体外的字节码上，方法体中的类型转换是有效的等
- 符号引用验证：在虚拟机将符号引用转化为直接引用的时候进行验证，可以看做是对类自身以外的信息（常量池中的各种符号引用）进行匹配性的校验。常见的异常比如：java.lang.NoSuchMethdError、java.lang.NoSuchFiledError等



#### 准备阶段（Preparation）

准备阶段主要为类或接口的静态字段分配内存，并将此类字段初始化为其默认值，**变量所使用的内存都将在元空间进行分配**。静态字段的显示初始化会在初始化阶段（Initialization）进行，而不是在准备阶段。并且准备阶段可以在加载后的任何时候进行，但必须在初始化之前完成



比如类中有字段：

```java
public static int a = 1;
public static final String b = "123";
```



那么在此阶段，对于字段 `a` 会被初始化为 0，在初始化阶段才会被赋值为 1。对于字段 `b` 有一点特殊，因为是常量，会在此阶段被初始化为指定的值，即：`b = 123`



#### 解析阶段（Resolution）

未完待续.....

