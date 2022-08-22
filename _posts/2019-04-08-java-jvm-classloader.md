---
layout: post
title: JVM类加载机制及类加载器
tags: JVM
categories: Java
date: 2019-04-08
---

## 类加载时机

我们都知道JVM通过加载.class文件，能够将其中的字节码解析成操作系统机器码。那么JVM加载时类的初始化是什么时候呢？大致有这么5种情况：

1. 遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类没有进行过初始化，则需要先对其进行初始化。生成这四条指令的最常见的Java代码场景是：
   - 使用new关键字实例化对象的时候；
   - 读取或设置一个类的静态字段（被final修饰，已在编译器把结果放入常量池的静态字段除外）的时候；
   - 调用一个类的静态方法的时候
2. 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
3. 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。
5. 当使用jdk1.7动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getstatic,REF_putstatic,REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行初始化，则需要先出触发其初始化。

对于这五种会触发类进行初始化的场景，虚拟机规范中使用了一个很强烈的限定语：“有且只有”，这五种场景中的行为称为对一个类进行 **主动引用**。除此之外，所有引用类的方式，都不会触发初始化，称为 被动引用。

被动引用的场景举例：

1. 通过子类引用父类的静态字段，并不会导致子类初始化，只有直接定义这个字段的类才会初始化（这里指父类）
2. 通过数组定义来引用类，不会触发此类的初始化
3. 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义的常量的类的初始化。

## 类加载过程

JVM加载类的过程分为这么几个阶段：加载、验证、准备、解析、初始化。

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h5faseft9ij20p70d9dhn.jpg)

### 加载

加载的主要作用是将外部的class文件，加载到方法区，我们可以从多个地方加载，比如Jar、War等Zip包，也可以从网络加载，或者通过动态代理技术等运行时计算生成等等

### 验证

这个阶段主要是为了安全验证文件是否符合JVM的规范要求，不符合规范的将抛出 java.lang.VerifyError 错误，主要包括这么几个阶段

- 文件格式验证
- 元数据验证
- 字节码验证
- 符号引用验证

### 准备

这一阶段主要市为类变量（static修饰的，不包括实例变量）分配内存并将其初始化为默认值。此时实例对象还没有分配内存，所以这些动作是在方法区上面进行的。而实例变量将会在对象实例化时随着对象一起分配在Java堆中。

```java
public class TestMain {
    static int x;
    public static void main(String[] args) {
        System.out.println(x);
    }
}

public class TestMain {
    public static void main(String[] args) {
        int x;  // 局部变量未初始化，编译报错
        System.out.println(x);
    }
}

```

上面代码中，类变量有两次赋初始值的过程，一次在准备阶段，赋予初始值（也可以是指定值）；另外一次在初始化阶段，赋予程序员定义的值。所以可以编译通过。但是局部变量并没有在准备阶段给它赋初始值，所以编译不通过。

### 解析

该过程是将符号引用替换为直接引用的过程，符号引用是一种定义，可以是任何字面上的含义，而直接引用就是直接指向目标的指针、相对偏移量。后者的对象都存在于内存当中。大体可以分为这么几个部分：

- 类或者接口的解析
- 字段解析
- 类方法解析
- 接口方法解析

其中几个经常发生的异常，就与这个阶段有关。

java.lang.NoSuchFieldError 根据继承关系从下往上，找不到相关字段时的报错。

java.lang.IllegalAccessError 字段或者方法，访问权限不具备时的错误。

java.lang.NoSuchMethodError 找不到相关方法时的错误。

解析过程保证了相互引用的完整性，把继承与组合推进到运行时。

## 初始化

这是类加载过程的最后一步，主要是对成员变量进行初始化，在这一阶段，才开始执行类中定义的Java程序代码（或者叫字节码），从另一个角度来说，初始化阶段是执行类构造器`<clinit>()`方法的过程，该方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块(static{}块)中的语句合并产生的，编译器收集的顺序是由语句在源文件中的顺序锁决定的。静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。如下代码示例：

```java
public class TestMain {
    static int x = 0;
    static {
        x = 1;
        y = 1;
    }
    static int y = 0;
    public static void main(String[] args) {
        System.out.println(x);   // 1
        System.out.println(y);   // 0
    }
}
```

上面代码中x和y的区别就是他们相对于static静态代码块的位置。

这就引出一个规则：static 语句块，只能访问到定义在 static 语句块之前的变量。所以下面的代码是无法通过编译的。

```java
static {
    y++;  // 报错，非法向前引用
}
static int y = 0;
```

第二个规则是：JVM 会保证在子类的初始化方法执行之前，父类的初始化方法已经执行完毕。所以第一个被执行的类初始化方法一定是java.lang.Object

上面`<clinit>`方法与类的构造函数（或者实例构造器`<init>`）不同，它不用显示调用父类构造器，虚拟机会保证在子类`<clinit>`方法执行之前，父类的`<clinit>`已经执行完毕，所以第二个规则也就可以理解了。

```java
class TestMain {

    static {
        System.out.println(1);
    }

    TestMain() {
        System.out.println(2);
    }

    public static class Child extends TestMain{
        static {
            System.out.println('x');
        }
        Child() {
            System.out.println('y');
        }
    }



    public static void main(String[] args) {
        TestMain testMain = new Child();
        TestMain testMain1 = new Child();
    }
}

// 上面示例输出顺序：
// 1
// x
// 2
// y
// 2
// y
```

其中 static 字段和 static 代码块，是属于类的，在类的加载的初始化阶段就已经被执行。类信息会被存放在方法区，在同一个类加载器下，这些信息有一份就够了，所以上面的 static 代码块只会执行一次，它对应的是 <clinit> 方法。而对象初始化就不一样了。通常，我们在 new 一个新对象的时候，都会调用它的构造方法，就是 <init>，用来初始化对象的属性。每次新建对象的时候，都会执行。

## 类加载器

对于任意一个类，都需要由它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间，通俗点讲：比较两个类是否"相等"，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个class文件，被同一个虚拟机加载，只要加载他们的类加载器不同，那么这两个类也就不相等。

- 启动类加载器（Bootstrap ClassLoader）

  主要用于加载<JAVA_HOME>/lib中的或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的类库。

- 扩展类加载器（Extension ClassLoader）

  主要负责加载<JAVA_HOME>/lib/ext目录中的或者被java.ext.dirs系统变量所指定的路径中的所有类库。

- 应用程序类加载器（Application ClassLoader）

  这是我们写Java类的默认加载器，一般也称作系统类加载器，它负责加载用户类路径所指定的类库，如果应用程序中没有定义过自己的类加载器，就由它加载。

- 自定义加载器（Custom ClassLoader）

  支持一些个性化的扩展功能。

### 双亲委派机制

它的工作过程是：如果一个类加载器收到了类加载的请求，他首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传递到顶层的启动类加载器中，只有当父类加载器自己无法完成这个加载请求，子加载器才会尝试自己去加载。

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h5fdc4c11ej20oe0kntaq.jpg)

这一点我们可以通过JDK源代码看出

```java
// ClassLoader.java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false); // 先有父类加载器加载
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {  // 再由子类加载器加载
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

```

上面代码中，它首先使用parent尝试进行类加载，如果失败后，才轮到自己，同时我们也发现这个方法是可以被覆盖的，也就是说双亲委派机制是可以不生效。

这个模型的好处在于 Java 类有了一种优先级的层次划分关系。比如 Object 类，这个毫无疑问应该交给最上层的加载器进行加载，即使是你覆盖了它，最终也是由系统默认的加载器进行加载的。

那试想一个问题，为什么设计成先由应用程序类加载器加载类，这样岂不是更省事？为什么不直接由启动类加载器加载，而是选择从下往上，往返两次？

在生产应用中，95%的类其实都是由应用类加载器加载的，如果类已经被加载过一次，就会被保存在应用类加载器中，下次使用可直接从应用类加载器中获取。但是如果由启动类加载器加载，大部分的类的获取都要走一个 启动类加载器 > 扩展类加载器 > 应用类加载器的过程，下次使用时也要走一遍上述过程，效率显然比直接从应用类加载器中获取要慢不少，所以为了后续调用节省时间，宁可第一次加载浪费点时间，也要选择应用程序类加载器。

第二个问题，如何打破双亲委派机制呢？

这就需要我们在自定义类加载器时重写ClassLoader中的loadClass方法，试看下面代码：

```java
protected Class<?> loadClass(String name, boolean resolve)
                throws ClassNotFoundException
        {
            synchronized (getClassLoadingLock(name)) {
                // First, check if the class has already been loaded
                Class<?> c = findLoadedClass(name);
                if (c == null) {
                    long t0 = System.nanoTime();


                    if (c == null) {
                        long t1 = System.nanoTime();
                        //如果不是这个包下的类,比如Object，使用 自定义类加载器的父类加载器 即应用类加载器，双亲委派
                        if(!name.startsWith("com.yuxingxin.user.entity")){
                            c = this.getParent().loadClass(name);
                        }else {
                            //如果是这个包下的类 ，比如User，打破双亲委派机制
                            c = findClass(name);
                        }

                        // this is the defining class loader; record the stats
                        sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                        sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                        sun.misc.PerfCounter.getFindClasses().increment();
                    }
                }
                if (resolve) {
                    resolveClass(c);
                }
                return c;
            }
        }
```

最后就是通过覆写findClass查找我们指定的路径：

```java
//自定义类加载器MyClassLoade extends ClassLoader
class MyClassLoader extends ClassLoader {

    private String classPath;  //定义要加载的类的路径

    public MyClassLoader(String classPath) {
        this.classPath = classPath;
    }

    //把路径下的文件转化化为字节数组
    private byte[] loadByte(String name) throws Exception {
        name = name.replaceAll("\\.", "/");
        FileInputStream fis = new FileInputStream(classPath + "/" + name + ".class");
        int len = fis.available();
        byte[] data = new byte[len];
        fis.read(data);
        fis.close();
        return data;
    }
	
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] data = loadByte(name);
            //defineClass将一个字节数组转为Class对象，这个字节数组是class文件读取后最终的字节 数组, 由native方法实现
            return defineClass(name, data, 0, data.length);
        } catch (Exception e) {
            e.printStackTrace();
            throw new ClassNotFoundException();
        }
    }
    
    @Overirde
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException{
        ...
    }
}
```

