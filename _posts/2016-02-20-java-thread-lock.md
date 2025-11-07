---
layout: post
title: Java多线程之线程同步锁机制
tags: 多线程
categories: Java
date: 2016-02-20
---

## Lock接口

锁是用来控制多个线程访问共享资源的方式，在Java SE 5之前，Java程序主要靠synchronized关键字来实现锁功能，Java SE 5之后并发包中新增了Lock接口以及相关实现类用来实现锁功能。它提供了与synchronized关键字类似的同步功能，只是在使用时需要显示的获取和释放锁，虽然它缺少了隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

例如：针对一个场景，手把手进行锁获取和释放，先获得锁A，然后再获取锁B，当锁B获得后，释放锁A同时获取锁C，当锁C获得后，再释放B同时获取锁D，以此类推。这种情况下，synchronized关键字就不那么容易实现了，而使用Lock就容易的多。

```java
Lock lock = new ReentrantLock();
lock.lock();
try{
    // ……
}finally {
    lock.unlock();
}
```

在finally块中释放锁，目的是保证在获取到锁之后，最终能够释放锁。注意不要将获取锁的过程写在try块中，因为如果在获取锁（自定义锁的实现）时发生了异常，异常抛出的同时，也会导致锁无故被释放。

它的API：

1. void lock()：获取锁，调用该方法当前线程会获取锁，当获得锁后，从方法返回
2. void lockInterruptibly() throws InterruptedException：可中断地获取锁，和lock()方法的不同之处在于该方法会响应中断，即在锁的获取中可以中断当前线程。
3. boolean tryLock()：尝试非阻塞的获取锁，调用该方法后立即返回，如果能够获取则返回true，否则返回false
4. boolean tryLock(long time , TimeUnit unit) throws InterruptedException：超时获取锁，当前线程在以下3种情况会返回：
   - 当前线程在超时时间内获得了锁
   - 当前线程在超时时间内被中断
   - 超时时间结束，返回false
5. void unlock()：释放锁
6. Condition new Condition()：获取等待通知组件，该组件和当前的锁绑定，当前线程只有获得了锁，才能调用该组件的wait方法，而调用后，当前线程将释放锁。

队列同步器AQS（AbstractQueuedSynchronizer）是用来构建锁或者其他同步组件的基础框架，它使用一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。有关同步器的更多原理可以自行学习。

## 重入锁

重入锁ReentrantLock，顾名思义，就是支持重新进入的锁，它表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁还支持获取锁时的公平和非公平性选择。

如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁就是公平的，反之，是不公平的，公平的获取锁，也就是等待时间最长的线程最优先获取锁，也可以说锁获取是顺序的。ReentrantLock提供一个构造函数，能够满足锁是否是公平的。

而像Mutex类的锁通过lock方法获取锁之后，如果再次调用lock方法，该线程将会被自己阻塞，所以这类锁就不是一个重入锁，而前面提到的synchronized关键字隐式的支持重进入，比如一个synchronized修饰的递归方法，在方法执行时，执行线程在获取了锁之后，仍能够连续多次的获得该锁。

前面的例子中，synchronized关键字用于加锁，这种锁很重，获取时一直等待，没有额外的尝试机制，而我们用ReentrantLock再看下：

```java
// 使用synchronized
public class Main {
    private int count;

    public void add(int n) {
        synchronized(this) {
            count += n;
        }
    }
}

// 使用ReentrantLock
public class Main {
    private final Lock lock = new ReentrantLock();
    private int count;

    public void add(int n) {
        lock.lock();
        try {
            count += n;
        } finally {
            lock.unlock();
        }
    }
}

```

因为`synchronized`是Java语言层面提供的语法，所以我们不需要考虑异常，而`ReentrantLock`是Java代码实现的锁，我们就必须先获取锁，然后在`finally`中正确释放锁。

顾名思义，`ReentrantLock`是可重入锁，它和`synchronized`一样，一个线程可以多次获取同一个锁。

和`synchronized`不同的是，`ReentrantLock`可以尝试获取锁：

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // ...
    } finally {
        lock.unlock();
    }
}
```

上述代码在尝试获取锁的时候，最多等待1秒。如果1秒后仍未获取到锁，`tryLock()`返回`false`，程序就可以做一些额外处理，而不是无限等待下去。所以，使用`ReentrantLock`比直接使用`synchronized`更安全，线程在`tryLock()`失败的时候不会导致死锁。

## 读写锁

之前提到的锁（Mutex类和ReentrantLock）基本都是排他锁，这些锁在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大的提升。

在没有读写锁支持的时候，如果需要完成上述工作就要使用Java的等待通知机制，就是当写操作开始时，所有晚于写操作的读操作均会进入等待状态， 只有写操作完成并进行通知之后，所有等待的读操作才能继续执行（写操作之间依靠synchronized关键字同步），这样做的目的是使读操作能读取到正确的数据，不会出现脏数据。改用读写锁实现上述功能，只需要在读操作时获取读锁，写操作时获取写锁，当写锁被获取到后，后续（非当前的写操作线程）的读写操作都会被阻塞，写锁释放之后，所有操作继续执行，编程方式相对于使用等待通知机制的实现方式而言，变得简单明了。

```java
public class Main {
  private int x = 0;
  ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
  Lock readLock = lock.readLock();
  Lock writeLock = lock.writeLock();

  private void count() {
    writeLock.lock();
    try {
      x++;
    } finally {
      writeLock.unlock();
    }
  }

  private void print(int time) {
    readLock.lock();
    try {
      System.out.print(x + " ");
    } finally {
      readLock.unlock();
    }
  }
}
```

把读写操作分别用读锁和写锁来加锁，在读取时，多个线程可以同时获得读锁，这样就大大提高了并发读的执行效率。

如果我们深入分析`ReentrantReadWriteLock`，会发现它有个潜在的问题：如果有线程正在读，写线程需要等待读线程释放锁后才能获取写锁，即读的过程中不允许写，这是一种悲观的读锁。要进一步提升并发执行效率，Java 8引入了新的读写锁：`StampedLock`。

`StampedLock`和`ReentrantReadWriteLock`相比，改进之处在于：读的过程中也允许获取写锁后写入！这样一来，我们读的数据就可能不一致，所以，需要一点额外的代码来判断读的过程中是否有写入，这种读锁是一种乐观锁。乐观锁的意思就是乐观地估计读的过程中大概率不会有写入，因此被称为乐观锁。反过来，悲观锁则是读的过程中拒绝有写入，也就是写入必须等待。显然乐观锁的并发效率更高，但一旦有小概率的写入导致读取的数据不一致，需要能检测出来，再读一遍就行。

```java
public class Main {
  private int x = 0;
  private final StampedLock stampedLock = new StampedLock();
  Lock readLock = lock.readLock();
  Lock writeLock = lock.writeLock();

  private void count() {
    long stamp = stampedLock.writeLock(); // 获取写锁
    try {
      x++;
    } finally {
      stampedLock.unlockWrite(stamp); // 释放写锁
    }
  }

  private void print(int time) {
    long stamp = stampedLock.tryOptimisticRead(); // 获得一个乐观读锁
    double currentX = x;
    if (!stampedLock.validate(stamp)) { // 检查乐观读锁后是否有其他写锁发生
        stamp = stampedLock.readLock(); // 获取一个悲观读锁
        try {
          currentX = x;
        } finally {
          stampedLock.unlockRead(stamp); // 释放悲观读锁
        }
    }
    System.out.print(currentX + " ");
  }
}
```

和`ReentrantReadWriteLock`相比，写入的加锁是完全一样的，不同的是读取。注意到首先我们通过`tryOptimisticRead()`获取一个乐观读锁，并返回版本号。接着进行读取，读取完成后，我们通过`validate()`去验证版本号，如果在读取过程中没有写入，版本号不变，验证成功，我们就可以放心地继续后续操作。如果在读取过程中有写入，版本号会发生变化，验证将失败。在失败的时候，我们再通过获取悲观读锁再次读取。由于写入的概率不高，程序在绝大部分情况下可以通过乐观读锁获取数据，极少数情况下使用悲观读锁获取数据。

可见，`StampedLock`把读锁细分为乐观读和悲观读，能进一步提升并发效率。但这也是有代价的：一是代码更加复杂，二是`StampedLock`是不可重入锁，不能在一个线程中反复获取同一个锁。

`StampedLock`还提供了更复杂的将悲观读锁升级为写锁的功能，它主要使用在if-then-update的场景：即先读，如果读的数据满足条件，就返回，如果读的数据不满足条件，再尝试写。

## Condition接口

前面我们提到，在没有读写锁支持的时候，如果需要完成读写锁的需求就要使用Java的等待通知机制（wait/notify），`synchronized`可以配合`wait`和`notify`实现线程在条件不满足时等待，条件满足时唤醒，用`ReentrantLock`我们怎么编写`wait`和`notify`的功能呢？答案是使用`Condition`对象来实现`wait`和`notify`的功能。

```java
// 使用ReentrantLock
public class Main {
    private final Lock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();

    public viud conditionalWait() throws InterruptedException {
        lock.lock();
        try{
            condition.await();
        } finally {
            lock.unlock();
        }
    }
    
    public void conditionSignal() throws InterruptedException {
        lock.lock();
        try{
            condition.signal();
        }finally {
            lock.unlock();
        }
    }
}
```

可见，使用`Condition`时，引用的`Condition`对象必须从`Lock`实例的`newCondition()`返回，这样才能获得一个绑定了`Lock`实例的`Condition`实例。

`Condition`提供的`await()`、`signal()`、`signalAll()`原理和`synchronized`锁对象的`wait()`、`notify()`、`notifyAll()`是一致的，并且其行为也是一样的：

- `await()`会释放当前锁，进入等待状态；
- `signal()`会唤醒某个等待线程；
- `signalAll()`会唤醒所有等待线程；
- 唤醒线程从`await()`返回后需要重新获得锁。

此外，和`tryLock()`类似，`await()`可以在等待指定时间后，如果还没有被其他线程通过`signal()`或`signalAll()`唤醒，可以自己醒来：

```java
if (condition.await(1, TimeUnit.SECOND)) {
    // 被其他线程唤醒
} else {
    // 指定时间内没有被其他线程唤醒
}
```

可见，使用`Condition`配合`Lock`，我们可以实现更灵活的线程同步。

## 并发容器

我们知道多线程环境中，集合类都不是线程安全的类，而Java提供了一套标准的线程安全集合，如List类，Map类，Set类，Queue类，Deque类等

| 接口  | 非线程安全              | 线程安全                                                     |
| ----- | ----------------------- | ------------------------------------------------------------ |
| List  | ArrayList               | CopyOnWriteArrayList                                         |
| Map   | HashMap                 | ConcurrentHashMap                                            |
| Set   | HashSet / TreeSet       | CopyOnWriteArraySet                                          |
| Queue | ArrayDeque / LinkedList | ArrayBlockingQueue / LinkedBlockingQueue / ConcurrentLinkedQueue等 |
| Deque | ArrayDeque / LinkedList | LinkedBlockingDeque                                          |

使用这些并发集合与使用非线程安全的集合类完全相同。

