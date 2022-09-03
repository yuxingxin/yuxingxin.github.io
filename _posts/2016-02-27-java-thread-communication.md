---
layout: post
title: Java多线程之线程通信
tags: 多线程
categories: Java
date: 2016-02-27
---

## 等待/通知机制

在不使用等待/通知机制的程序，如果实现两个线程间通信，往往使用的就是while语句轮询来检测某一个条件，这样会浪费CPU资源。

在前面的介绍中，`synchronized`解决了多线程竞争的问题。但是`synchronized`并没有解决多线程协调的问题。多个线程之间也可以实现通信，原因就是多个线程共同访问同一个变量。但是这种通信机制不是 “等待/通知” ，两个线程完全是主动地读取一个共享变量。简单的说，等待/通知机制就是一个【线程A】等待，一个【线程B】通知（线程A可以不用再等待了）。

在Java语言中，实现等待通知机制主要是用：wait()/notify()方法实现

在wait()方法 ：wait()方法是使当前执行代码的线程进行等待，wait()方法是Object类的方法，该方法用来将当前线程置入“欲执行队列”中，并且在wait()所在的代码处停止执行，直到接到通知或被中断为止。在调用wait()方法之前，线程必须获得该对象的对象级别锁，即只能在同步方法或者同步块中调用wait()方法。在执行wait()方法后，当前线程释放锁。
notify()方法：方法notify()也要在同步方法或同步块中调用，即在调用前，线程也必须获得该对象的对象级别锁。
这两个方法都需要同synchronized关键字联用。

```java
public class Main {   
 	public static void main(String[] args) {
        String username = new String("yuxingxin");
        try {
            username.wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

运行抛出如下异常：java.lang.IllegalMonitorStateException，具体的原因是：没有“对象监视器”，也就是没有同步加锁。修改如下：

```java
public class Main {   
    private Object object = new Object();
 	public static void main(String[] args) {
        synchronized(object) {
            try {
                object.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

此示例运行成功，但是会运行到object.wait()时卡住，因为没有notify()唤醒它。

这里有几点需要注意：

1. 执行完上面同步代码块，就会释放对象的锁。
2. 执行同步代码块的过程中，如果遇到了异常，比如调用interrupt方法，而导致线程终止，锁也会被释放
3. 在执行同步代码块的过程中，执行了锁所属对象的wait方法，这个线程会释放对象锁，而此线程对象也会进入线程等待池中，等待被唤醒。

notify()方法一次只随机通知一个线程进行唤醒，唤醒所有线程可以使用notifyAll()方法。

## 生产/消费模型

等待/通知最典型的案例就是生产者/消费者模型

其中等待方遵循如下原子：

1. 获取对象的锁。
2. 如果条件不满足，那么调用对象的wait()方法，被通知后仍要检查条件。
3. 条件满足则执行对应的逻辑。

```java
// 伪代码
synchronized(object) {
    while(条件不满足) {
    	object.wait();
    }
    // 对应的处理逻辑
}
```

通知方遵循如下原则:

1. 获得对象的锁。
2. 改变条件。
3. 通知所有等待在对象上的线程。

```java
// 伪代码
synchronized(object) {
    // 改变条件
    // 执行通知
    object.notifyAll();
}

```

## ThreadLocal

变量值的共享可以使用public static变量的形式，所有线程都是用同一个变量，如果想实现每一个线程都有自己的共享变量，那么ThreadLocal就派上用场了。

类ThreadLocal主要解决的就是每个线程绑定自己的值，可以将ThreadLocal类比喻成全局存放数据的盒子，盒子中可以存储每个线程的私有数据。

### 创建和使用

支持泛型

```java
ThreadLocal<String> stringThreadLocal = new ThreadLocal<>();
stringThreadLet.set("yuxingxin");
stringThreadLocal.get();

// 设置初始值
ThreadLocal<String> mThreadLocal = new ThreadLocal<String>() {
    @Override
    protected String initialValue() {
      return Thread.currentThread().getName();
    }
};
```

在Android中，Looper类就是利用了ThreadLocal的特性，保证每个线程只存在一个Looper对象。

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));t
}
```



