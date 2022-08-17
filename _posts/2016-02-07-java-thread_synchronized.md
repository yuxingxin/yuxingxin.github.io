---
layout: post
title: Java多线程之线程同步
tags: 多线程
categories: Java
date: 2016-02-07
---

## synchronized

非线程安全其实是会在多个线程对同一个对象中的实例进行访问时发生，产生的后果就是"脏读"，也就是说取到的数据是被修改过的，即多个线程同时读写共享变量，而"线程安全"就是以获得的实例变量的值经过同步处理的，不会出现脏读的现象，  提起线程同步，我们首先想到的方法就是synchronized关键字。

- 修饰实例方法，作用于当前实例加锁，进入同步代码钱要获得当前实例的锁
- 修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁
- 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码块前要获得给定对象的锁

接下来我们看下实例：

```java
public class Main implements Runnable{
    //共享资源(临界资源)
    static int i=0;

    public void increase(){
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<10000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Main instance=new Main();
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
    /**
     * 输出结果:
     * 2000000
     */
}

```

从上面实例可以看出，两个线程同时对一个`int`变量进行操作，每个都分别加10000次，最后结果应该是20000，但是，每次运行，结果实际上都是不一样的。

这是因为变量的读取和写入需要是原子操作，而上面的很明显不是，针对上面代码修改：

```java
/**
  * synchronized 修饰实例方法
*/
public synchronized void increase(){
    i++;
}
```

上面synchronized修饰的是类方法，锁的是实例，当多个线程操作不同实例时，会使用不同实例的锁，就无法保证static变量的有序性了。如下：

```java
public class Main implements Runnable{
    static int i=0;
    public synchronized void increase(){
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<10000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        //new新实例
        Thread t1=new Thread(new Main());
        //new新实例
        Thread t2=new Thread(new Main());
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();
        System.out.println(i);
    }
}
```

所以当sychronized修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁。而作用于静态方法时，锁就是当前类到class对象锁。由于静态成员变量不专属于任何一个实例对象，是类成员，因此通过class对象锁可以控制静态成员的并发操作。

```java
public class Main implements Runnable {
    static int i=0;
    public synchronized static void increase() {
        // ……
    }
}


// 相当于
public class Main implements Runnable {
    static int i=0;
    public static void increase() {
        synchronized(Main.class){
            // ……
        } 
    }
}

```

修饰代码块时，指定加锁对象，对给定加锁对象，进入同步代码块前要获得给定对象的锁。

```java
public class Main implements Runnable{
    static Object instance=new Object();
    static int i=0;
    @Override
    public void run() {
        //省略其他耗时操作....
        //使用同步代码块对变量i进行同步操作,锁对象为instance
        synchronized(instance){
            for(int j=0;j<10000;j++){
                    i++;
              }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
}
```

上述代码，将synchronized作用于一个给定的实例对象instance, 即当前实例对象就是锁对象，每次当线程进入synchronized包裹到代码块时，就会要求当前线程持有instance实例对象锁，如果当前有其他线程正持有该对象锁，那么新到到线程就必须等待，这样也就保证了每次只有一个线程执行`i++`操作。当然， 还可以使用this或者class

```java
//this,当前实例对象锁
synchronized(this){
    for(int j=0;j<10000;j++){
        i++;
    }
}

//class对象锁
synchronized(Main.class){
    for(int j=0;j<10000;j++){
        i++;
    }
}
```

## 多线程死锁

Java线程锁为可重入的锁，所谓可重入锁是JVM允许同一个线程重复获取同一个锁，这种能被同一个线程反复获取的锁，就叫做可重入锁。

```java
public class Counter {
    private int count = 0;

    public synchronized void add(int n) {
        if (n < 0) {
            dec(-n);
        } else {
            count += n;
        }
    }

    public synchronized void dec(int n) {
        count += n;
    }
}
```

观察`synchronized`修饰的`add()`方法，一旦线程执行到`add()`方法内部，说明它已经获取了当前实例的`this`锁。如果传入的`n < 0`，将在`add()`方法内部调用`dec()`方法。可以看到`dec()`方法也需要获取`this`锁，由于Java的线程锁是可重入锁，所以，获取锁的时候，不但要判断是否是第一次获取，还要记录这是第几次获取。每获取一次锁，记录+1，每退出`synchronized`块，记录-1，减到0的时候，才会真正释放锁。

在获取多个锁的时候，不同线程获取多个不同对象的锁可能导致死锁。改造下上面代码：

```java
public class Counter {
    private int count = 0;
    private Object monitor1 = new Object();
    private Object monitor2 = new Object();

    public void add() {
        synchronized(monitor1){
            this.count++;
            synchronized(monitor2){
                this.count++;
            }
        }
    }

    public void dec() {
        synchronized(monitor2){
            this.count--;
            synchronized(monitor1){
                this.count--;
            }
        }
    }
}
```

如上，如果线程1和线程2分别执行add和dec方法：

- 线程1：进入`add()`，获得`monitor1`；
- 线程2：进入`dec()`，获得`monitor2`。

随后：

- 线程1：准备获得`monitor2`，失败，等待中；
- 线程2：准备获得`monitor1`，失败，等待中。

此时，两个线程各自持有不同的锁，然后各自试图获取对方手里的锁，造成了双方无限等待下去，这就是死锁。

死锁发生后，没有任何机制能解除死锁，只能强制结束JVM进程。

## volatile

volatile的主要作用是使变量在多个线程间可见。举例如下：

```java
public class Main {
  private boolean isRunning = true;

  private void setRunning(boolean isRunning) {
    this.isRunning = isRunning;
  }

  
  public void print() {
    new Thread() {
      @Override
      public void run() {
        while (isRunning) {
            System.out.println("print running log")
        }
      }
    }.start();
    
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
      
    // 执行停止
    setRunning(false);
  }
}

```

上面代码程序运行后，根本停不下来，处于死循环中。解决的办法就是给变量加上volatile关键字，使强制从公共堆栈中读取变量的值，而不是从线程私有数据栈中读取变量的值。而setRunning方法更新的是公共堆栈中的isRunning为false，但是线程私有堆栈中取得的isRunning一直为true，所以就一直为死循环状态。

这个问题其实就是线程私有堆栈当中的值和公关堆栈中的值不同步造成的，解决这样的问题就是使用volatile关键字了，它主要作用就是当线程方位isRunning这个变量时，强制从公告堆栈中进行取值。

```java
public class Main {
  private volatile boolean isRunning = true;

  private void setRunning(boolean isRunning) {
    this.isRunning = isRunning;
  }

  
  public void print() {
    new Thread() {
      @Override
      public void run() {
        while (isRunning) {
            System.out.println("print running log")
        }
      }
    }.start();
    
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
      
    // 执行停止
    setRunning(false);
  }
}

```

由此可见，使用volatile增加了实例变量在多个线程之间的可见性。但是它不能保证原子性，而sychronized可以保证原子性。

```java
public class Main implements Runnable{
    volatile static int i=0;
    public static void increase(){
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<10000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Main instance=new Main();
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();
        System.out.println(i);
    }
}
```

上面这个例子，最后的记过却不对，不是20000，原因也就是因为i++本身不是原子操作，它的步骤分为：

1. 从内存中取出i的值
2. 计算i的值
3. 将i的值写到内存中

上面示例中加上synchronized关键字就可以了，关键字volatile使用的场合是在多个线程中可以感知实例变量被更改了， 并且可以获得最新的值使用，也就是多线程读取共享变量时可以获得最新的值使用。

当然上面这个问题我们也可以使用原子类来解决。

## 原子类

原子操作是不可分割的整体，没有其他线程能够中断或者检查正在原子操作中的变量，一个原子（atomic）类型就是一个原子操作可用的类型，他可以在没有锁的情况下做到线程安全(thread-safe)，代码如下：

```java
public class Main implements Runnable{
    private AtomicInteger count = new AtomicInteger()
    
    @Override
    public void run() {
        for(int j=0;j<10000;j++){
            count.incrementAndGet();
            // count.getAndIncrement();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Main instance=new Main();
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();
        System.out.println(count.get());
    }
}

```

它的主要原理就是利用了CAS：Compare and Set。CAS 操作包含三个操作数 ： 内存位置、预期数值和新值。CAS 的实现逻辑是将内存位置处的数值与预期数值想比较，若相等，则将内存位置处的值替换为新值。若不相等，则不做任何操作。在 Java 中，Java 并没有直接实现 CAS，CAS 相关的实现是通过 C++ 内联汇编的形式实现的。Java 代码需通过 JNI 才能调用。

CAS 与 Synchronized 的使用情景：　　　

1. 对于资源竞争较少（线程冲突较轻）的情况，使用synchronized同步锁进行线程阻塞和唤醒切换以及用户态内核态间的切换操作额外浪费消耗cpu资源；而CAS基于硬件实现，不需要进入内核，不需要切换线程，操作自旋几率较少，因此可以获得更高的性能。
2. 对于资源竞争严重（线程冲突严重）的情况，CAS自旋的概率会比较大，从而浪费更多的CPU资源，效率低于synchronized。

补充： synchronized 在 jdk1.6 之后，已经改进优化。synchronized 的底层实现主要依靠 Lock-Free 的队列，基本思路是自旋后阻塞，竞争切换后继续竞争锁，稍微牺牲了公平性，但获得了高吞吐量。在线程冲突较少的情况下，可以获得和 CAS 类似的性能；而线程冲突严重的情况下，性能远高于 CAS。

当然CAS也存在一些问题：

### ABA问题

谈到 CAS，基本上都要谈一下 CAS 的 ABA 问题。CAS 由三个步骤组成，分别是“读取-比较-写回”。考虑这样一种情况，线程1和线程2同时执行 CAS 逻辑，两个线程的执行顺序如下：

- 时刻1：线程1执行读取操作，获取原值 A，然后线程被切换走
- 时刻2：线程2执行完成 CAS 操作将原值由 A 修改为 B
- 时刻3：线程2再次执行 CAS 操作，并将原值由 B 修改为 A
- 时刻4：线程1恢复运行，将比较值(compareValue)与原值(oldValue)进行比较，发现两个值相等。

然后用新值(newValue)写入内存中，完成 CAS 操作

如上流程，线程1并不知道原值已经被修改过了，在它看来并没什么变化，所以它会继续往下执行流程。对于 ABA 问题，通常的处理措施是对每一次 CAS 操作设置版本号。

ABA问题的解决思路其实也很简单，就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么A→B→A就会变成1A→2B→3A了。

java.util.concurrent.atomic 包下提供了一个可处理 ABA 问题的原子类 AtomicStampedReference，这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。 

### 自旋CAS

自旋CAS（不成功，就一直循环执行，直到成功） 如果长时间不成功，会给 CPU 带来非常大的执行开销。如果JVM能支持处理器提供的 pause 指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline），使 CPU 不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起 CPU 流水线被清空（CPU pipeline flush），从而提高 CPU 的执行效率。 

### 其他原子类

我们总结下Java提供的原子类：

#### 原子更新基本类型类

1. AtomicInteger：原子更新整型
2. AtomicBoolean：原子更新布尔类型
3. AtomicLong：原子更新长整型

#### 原子更新数组

1. AtomicIntegerArray ： 原子更新整型数组里的元素
2. AtomicLongArray: 原子更新长整型数组里的元素
3. AtomicReferenceArray： 原子更新引用类型数组里的元素

#### 原子更新引用类型

1. AtomicReference：原子更新引用类型
2. AtomicReferenceFieldUpdater：原子更新引用类型里的字段
3. AtomicMarkableReference：原子更新带有标记为的引用类型

#### 原子更新字段类

1. AtomicIntegerFieldUpdater：原子更新整型字段的更新器
2. AtomicLongFieldUpdater：原子更新长整型字段的更新器
3. AtomicStampedReference：原子更新带有版本号的引用类型

## 单例模式

我们使用DCL(double-check-locking)双检查锁机制来实现单例模式

```java
public class SingleInstance {
  private static volatile SingleInstance mInstance;

  private SingleInstance() {
  }

  public static SingleInstance newInstance() {
    if (mInstance == null) {
      synchronized (SingleInstance.class) {
        if (mInstance == null) {
          mInstance = new SingleInstance();
        }
      }
    }
    return mInstance;
  }
}
```
