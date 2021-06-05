---

layout: post
title:  "Java虚拟机类加载过程与类加载机制"
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



类加载过程包括：加载（Loading）、连接（Linking）、初始化（Initialization）3个阶段。其中连接过程又可以分为
验证（Verification）、准备（Preparation）、解析（Resolution）三个阶段


---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/eeaf25b4-5baa-43f9-8b2d-87d60e572026.png)

---



### 加载阶段（loading）

加载（创建）阶段主要是通过类的全限定名来查找这个类的文件，将静态的类文件存储结构转化为运行时数据结构，并在内存中生成一个代表这个类的 java.lang.Class 对象，作为这个类的数据访问入口

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





### 连接阶段（linking）

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

准备阶段主要为类或接口的静态字段分配内存，并将此类字段初始化为其默认值，**变量所使用的内存都将在 方法区 进行分配**。对于该阶段有以下几点需要注意：

- 这时候进行内存分配的仅包括类变量（ Class Variables ，即静态变量，被 static 关键字修饰的变量，只与类相关，因此被称为类变量），而不包括实例变量。实例变量会在对象实例化时随着对象一块分配在 Java 堆中。
- 从概念上讲，类变量所使用的内存都应当在 方法区 中进行分配。不过有一点需要注意的是：JDK 7 之前，HotSpot 使用永久代来实现方法区的时候，实现是完全符合这种逻辑概念的。 而在 JDK 7 及之后，HotSpot 已经把原本放在永久代的字符串常量池、静态变量等移动到堆中，这个时候类变量则会随着 Class 对象一起存放在 Java 堆中
- 这里所设置的初始值"通常情况"下是数据类型默认的零值（如 0、0L、null、false 等），比如我们定义了public static int value=111 ，那么 value 变量在准备阶段的初始值就是 0 而不是 111（初始化阶段才会赋值）。特殊情况：比如给 value 变量加上了 final 关键字public static final int value=111 ，那么准备阶段 value 的值就被赋值为 111。

---

![markdown](https://ddmcc-1255635056.file.myqcloud.com/a310be8c-5a3f-4b45-9621-c4e8e4ea9b8c.png)

---

静态字段的显示初始化会在初始化阶段（Initialization）进行，而不是在准备阶段。并且准备阶段可以在加载后的任何时候进行，但必须在初始化之前完成



比如类中有字段：

```java
public static int a = 1;
public static final String b = "123";
```



那么在此阶段，对于字段 `a` 会被初始化为 0，在初始化阶段才会被赋值为 1。对于字段 `b` 有一点特殊，因为是常量，会在此阶段被初始化为指定的值，即：`b = 123`



#### 解析阶段（Resolution）

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用限定符 7 类符号引用进行

符号引用就是一组符号来描述目标，可以是任何字面量。直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。在程序实际运行时，只有符号引用是不够的，举个例子：在程序执行方法时，系统需要明确知道这个方法所在的位置。Java 虚拟机为每个类都准备了一张方法表来存放类中所有的方法。当需要调用一个类的方法的时候，只要知道这个方法在方法表中的偏移量就可以直接调用该方法了。通过解析操作符号引用就可以直接转变为目标方法在类中方法表的位置，从而使得方法可以被调用。

综上，**解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，也就是得到类或者字段、方法在内存中的指针或者偏移量**



### 初始化（Initialization）

初始化阶段就是执行初始化方法 `<clinit> ()` 方法的过程，是类加载的最后一步，这一步 JVM 才开始真正执行类中定义的 Java 程序代码(字节码)

> `<clinit> ()`方法是编译之后自动生成的

对于 `<clinit> ()` 方法的调用，虚拟机会自己确保其在多线程环境中的安全性。因为 `<clinit> ()` 方法是带锁线程安全，所以在多线程环境下进行类初始化的话可能会引起多个进程阻塞，并且这种阻塞很难被发现

对于初始化阶段，虚拟机严格规范了有且只有 5 种情况下，必须对类进行初始化(只有主动去使用类才会初始化类)：

- 当遇到 new 、 getstatic、putstatic 或 invokestatic 这 4 条直接码指令时，比如 new 一个类，读取一个静态字段(未被 final 修饰)、或调用一个类的静态方法时。
   - 当 jvm 执行 new 指令时会初始化类。即当程序创建一个类的实例对象。
   - 当 jvm 执行 getstatic 指令时会初始化类。即程序访问类的静态变量(不是静态常量，常量会被加载到运行时常量池)。
   - 当 jvm 执行 putstatic 指令时会初始化类。即程序给类的静态变量赋值。
   - 当 jvm 执行 invokestatic 指令时会初始化类。即程序调用类的静态方法。
- 使用 java.lang.reflect 包的方法对类进行反射调用时如 Class.forname("..."), newInstance() 等等。如果类没初始化，需要触发其初始化
- 初始化一个类，如果其父类还未初始化，则先触发该父类的初始化
- 当虚拟机启动时，用户需要定义一个要执行的主类 (包含 main 方法的那个类)，虚拟机会先初始化这个类
- MethodHandle 和 VarHandle 可以看作是轻量级的反射调用机制，而要想使用这 2 个调用， 就必须先使用 findStaticVarHandle 来初始化要调用的类
- 当一个接口中定义了 JDK8 新加入的默认方法（被 default 关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化


### 卸载

**卸载类即该类的 Class 对象被 GC**

卸载类需要满足 3 个要求:

- 该类的所有的实例对象都已被 GC，也就是说堆不存在该类的实例对象。
- 该类没有在其他任何地方被引用
- 该类的类加载器的实例已被 GC

所以，在 JVM 生命周期中，**由 jvm 自带的类加载器加载的类是不会被卸载的。但是由我们自定义的类加载器加载的类是可能被卸载的**

只要想通一点就好了，jdk 自带的 `BootstrapClassLoader`, `ExtClassLoader`, `AppClassLoader` 负责加载 `jdk` 提供的类，所以它们(类加载器的实例)肯定不会被回收。而我们自定义的类加载器的实例是可以被回收的，所以使用我们自定义加载器加载的类是可以被卸载掉的


