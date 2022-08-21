---
layout: post
title: JVM内存区域以及内存管理
tags: JVM
categories: Java
date: 2019-04-02
---

## 内存区域划分

JVM内存布局在Java8以及之后做了一些修改，方法区做了一些变化：将原来永久代（Permanent Generation）移除了，改为使用元空间（Metaspace）来代替，因此原来的两个调优参数：`-XX:PermSize`和`-XX:MaxPermSize`被弃用了，改为使用`-XX:MetaspaceSize` 和 `-XX:MaxMetaspaceSize` 来指定元空间的大小。如下图：

Java8之前：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h5et3mftrtj20tr0h60uu.jpg)

Java8之后：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h5et569uokj20s20guwgm.jpg)

方法区的变化总结一下就是：

1. 移除了永久代（PermGen）, 替换为元空间（Metaspace）
2. 永久代中的class metadata 转移到了 native memory(本地内存，而不是虚拟机)
3. 永久代中的interned Strings（字符串常量池） 和 class static variables（类静态变量）转移到了 Java heap
4. 上面说的调优参数做了变化

## 程序计数器

程序计数器是一块较小的内存单元，它可以看做当前线程所执行的字节码的行号指示器，也就是里面存的是当前线程的执行进度，由于JVM可以并发执行线程，因此会存在线程之间的切换，而这个时候就程序计数器会记录下当前程序执行到的位置，以便在其他线程执行完毕后，恢复现场继续执行。如果线程正在执行的是应该Java方法，这个计数器记录的是正在执行虚拟机字节码指令的地址。如果正在执行的是Native方法，计数器的值则为空（undefined），**注意程序计数器是唯一一个在 Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域。**除此之外，程序计数器还存储了当前正在运行的流程，包括正在执行的指令、跳转、分支、循环、异常处理等。我们可以使用javap命令输出字节码，在每个opcode的前面都有一个序号，这就可以理解为程序计数器的内容。举例如下：

```java
public class Test {
    public static void main(String[] args) {
        String s = "Hello World";
        System.out.println(s);
    }
}
```

将上面一段代码执行javac指令后生成Test.class文件，然后我们对该class文件执行`javap -p -v Test.class`指令后，看下输出结果：

```
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: ldc           #2                  // String Hello World
         2: astore_1
         3: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         6: aload_1
         7: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        10: return
      LineNumberTable:
        line 3: 0
        line 4: 3
        line 5: 10
```

上面的0，2，3，6，7，10即为程序计数器内容。

## 虚拟机栈

上图我们看到虚拟机栈是基于线程的，也就是说即便上面的例子只有一个main方法，也是以线程方式运行的，在线程的生命周期中，参与计算的数据会频繁地入栈和出栈，栈的生命周期是和线程一样的。每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于存储以下四个信息：

1. 局部变量表

   局部变量表是存放**方法参数**和**局部变量**的区域。全局变量是放在堆的，有两次赋值的阶段，一次在类加载的准备阶段，赋予系统初始值；另外一次在类加载的初始化阶段，赋予代码定义的初始值。

2. 操作数栈

   当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈/入栈操作

3. 动态链接

   每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用。持有这个引用是为了支持方法调用过程中的动态连接(Dynamic Linking)。

4. 方法出口

   即方法返回地址，指向特定指令内存地址的指针，返回分为 正常返回 和 异常退出。一般来说，方法正常退出时，调用者的PC计数器的值可以作为返回地址，栈帧中会保存这个计数器值。方法退出的过程相当于弹出当前栈帧。

每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。等所有栈帧都出栈以后，线程也就结束了。

最后它的调优参数为：-Xss

![image-20220822001204293](https://tva1.sinaimg.cn/large/e6c9d24ely1h5euaaeiyyj20rh0gywhm.jpg)

## 本地方法栈

和Java虚拟机栈比较类似，区别是Java虚拟机栈是调用Java方法；本地方法栈是调用本地native方法。

## 堆

Java 堆是被所有**线程共享**的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存，所以它也是最大的内存区域，我们常说的垃圾回收，操作的对象就是堆，所以也称为"GC堆"。

由于现在垃圾收集器都采用分代收集算法，所以又把堆分为新生代和老年代，新生代 又分为 Eden + From Survivor + To Survivor区。

我们都知道Java对象分为基本数据类型和普通对象，对于普通对象来说，JVM 会首先在堆上创建对象，然后在其他地方使用的其实是它的引用。比如，把这个引用保存在虚拟机栈的局部变量表中。而对于基本数据类型来说，当你在方法体内声明了基本数据类型的对象，它就会在栈上直接分配。其他情况，都是在堆上分配。而像 int[] 数组这样的内容，是在堆上分配的。因为数组并不是基本数据类型。上面就是JVM的基本内存分配策略。

最后它的调优参数为：-Xmx 和 -Xms。

## 方法区

方法区（Method Area）与 Java 堆一样，是所有**线程共享**的内存区域。方法区用于存储已经被虚拟机加载的类信息（即加载类时需要加载的信息，包括版本、field、方法、接口等信息）、final常量、静态变量、编译器即时编译的代码等。

方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。

Java 8 以前，类的信息都放在一个叫永久代的内存里，这个区域有大小限制，很容易造成 JVM 内存溢出，从而造成 JVM 崩溃。所以后来就别去除了。取而代之的就是开头说的元空间。**需要注意的是原来的永久代是在堆上的，而现在的元空间是在非堆上的**。

JVM中存在多个常量池。1、字符串常量池，已经移动到堆上（jdk8之前是perm区），也就是执行intern方法后存的地方。2、类文件常量池，constant_pool，是每个类每个接口所拥有的，第四节字节码中“#n”的那些都是。这部分数据在方法区，也就是元数据区。而运行时常量池是在类加载后的一个内存区域，它们都在元空间。

总结了下变化：

1. 移除了永久代（PermGen）, 替换为元空间（Metaspace）
2. 永久代中的class metadata 转移到了 native memory(本地内存/非堆)
3. 永久代中的interned Strings（字符串常量池） 和 class static variables（类静态变量非基本类型）转移到了 Java heap
4. 调优参数也由`-XX:PermSize`和`-XX:MaxPermSize`改为使用`-XX:MetaspaceSize` 和 `-XX:MaxMetaspaceSize`

## 总结

综上，JVM 的运行时区域是栈，而存储区域是堆。线程私有的区域有：程序计数器、Java虚拟机栈、本地方法栈，而线程共享的区域有方法区和Java堆，而多个线程访问，就会造成数据同步的问题。