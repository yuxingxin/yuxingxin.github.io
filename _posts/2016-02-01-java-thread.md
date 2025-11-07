---
layout: post
title: Java多线程之基础
tags: 多线程
categories: Java
date: 2016-02-01
---

## 基本概念

说起线程，就不得不先提下进程，在计算机中，我们把一个任务称为一个进程，浏览器就是一个进程，视频播放器是另一个进程，类似的，音乐播放器和Word都是进程。某些进程内部还需要同时执行多个子任务。例如，我们在使用Word时，Word可以让我们一边打字，一边进行拼写检查，同时还可以在后台进行打印，我们把子任务称为线程。

进程和线程的关系就是：一个进程可以包含一个或多个线程，但至少会有一个线程。而操作系统调度的最小任务单位就是进程，同一个应用程序，既可以有多个进程，也可以有多个线程。

## 线程创建

Java中常见的两种创建线程的方法：

### 继承Thread类

通常我们实例化一个Thread实例，然后调用它的start()方法：

```java
public class Main {
    public static void main(String[] args) {
        Thread t = new MyThread();
        t.start(); // 启动新线程
    }
}

class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("start new thread!");
    }
}
```

### 实现Runnable接口

创建Thread实例时，并传入一个Runnable实例：

```java
public class Main {
    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable());
        t.start(); // 启动新线程
    }
}

class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("start new thread!");
    }
}
```

## 守护线程

在Java线程中，有两种线程，一种是用户线程，一种是守护线程。

守护线程是一种特殊的线程，有"陪伴"的含义，当进程中不存在非守护线程了，则守护线程自动销毁。典型的守护线程就是垃圾回收线程。用个通俗的比喻：任何一个守护线程都是整个JVM非守护线程的『保姆』，只要当前JVM实例中还存在任何一个非守护线程没有结束，那么守护线程就在工作。

```java
public class Mythread extends Thread {
    private int i = 0;
    @Override
    public void run() {
        try {
            while(true){
                i++;
                System.out.println("i=" + i);
                Thread.sleep(1000);
            }
        } catch(InterruptedException e){
            e.printStackTrace();
		}
    }
}

public class Main {
    public static void main(String[] args) {
        Mythread t = new Mythread();
        t.setDaemon(true);
        t.start(); // 启动新线程
        try{
            Thread.sleep(5000);
            System.out.println("主线程已经结束了，t线程（守护线程）也不再打印了")
        }catch(InterruptedException e){
            e.printStackTrace();
		}
        
    }
}
```



## 线程方法

### 暂停执行sleep

我们可以在指定的毫秒数内让当前正在执行的线程休眠（暂停执行），这个正在执行的线程即是`this.currentThread()`返回的线程。

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("main start...");
        Thread t = new Thread() {
            public void run() {
                System.out.println("thread run...");
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {}
                System.out.println("thread end.");
            }
        };
        t.start();
        try {
            Thread.sleep(20);
        } catch (InterruptedException e) {}
        System.out.println("main end...");
    }
}
```

### join方法

在很多情况下，主线程生成并起动了子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程处理完其它的事务后，需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到 join() 方法了。

JDK 中对 join 方法解释为：“等待该线程终止”（*Waits for this thread to die*），换句话说就是：“当前线程等待子线程的终止”。也就是在子线程调用了 join() 方法后面的代码，只有等到子线程结束了当前线程才能执行。

### 中断线程interrupt

我们使用interrupt()方法效果并不会像for循环中break那样，马上停止循环，而是在当前线程中打了一个停止的标记，并不是真的停止线程。我们可以通过调用isInterrupted()方法来判断当前线程是否已经结束运行。

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new MyThread();
        t.start();
        Thread.sleep(1); // 暂停1毫秒
        t.interrupt(); // 中断t线程
        t.join(); // 等待t线程结束
        System.out.println("end");
    }
}

class MyThread extends Thread {
	
    @Override
    public void run() {
        int n = 0;
        while (!isInterrupted()) {
            n++;
            System.out.println(n + " hello!");
        }
    }
}
```

上面实例中，主线程通过调用`t.interrupt()`方法中断`t`线程，但是`interrupt()`方法仅仅向`t`线程发出了“中断请求”，至于`t`线程是否能立刻响应，要看具体代码。而`t`线程的`while`循环会检测`isInterrupted()`，所以上述代码能正确响应`interrupt()`请求，使得自身立刻结束运行`run()`方法。

如果线程处于等待状态，例如，`t.join()`会让`main`线程进入等待状态，此时，如果对`main`线程调用`interrupt()`，`join()`方法会立刻抛出`InterruptedException`，因此，目标线程只要捕获到`join()`方法抛出的`InterruptedException`，就说明有其他线程对其调用了`interrupt()`方法，通常情况下该线程应该立刻结束运行。

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new MyThread();
        t.start();
        Thread.sleep(1000);
        t.interrupt(); // 中断t线程
        t.join(); // 等待t线程结束
        System.out.println("end");
    }
}

class MyThread extends Thread {
    @Override
    public void run() {
        Thread hello = new HelloThread();
        hello.start(); // 启动hello线程
        try {
            hello.join(); // 等待hello线程结束
        } catch (InterruptedException e) {
            System.out.println("interrupted!");
        }
        hello.interrupt();
    }
}

class HelloThread extends Thread {
    @Override
    public void run() {
        int n = 0;
        while (!isInterrupted()) {
            n++;
            System.out.println(n + " hello!");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}
```

`main`线程通过调用`t.interrupt()`从而通知`t`线程中断，而此时`t`线程正位于`hello.join()`的等待中，此方法会立刻结束等待并抛出`InterruptedException`。由于我们在`t`线程中捕获了`InterruptedException`，因此，就可以准备结束该线程。在`t`线程结束前，对`hello`线程也进行了`interrupt()`调用通知其中断。如果去掉这一行代码，可以发现`hello`线程仍然会继续运行，且JVM不会退出。

## 线程池

在Java中合理的利用线程池能够带来以下好处 ：

1. 降低资源消耗，通过重复利用已创建的线程，从而降低线程创建和销毁带来的消耗。
2. 提高响应速度，当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性，线程是稀缺资源，如果无限制的创建，不进会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配，调优和监控。

### 原理

当向线程池提交一个任务时，处理流程是：

1. 线程池判断核心线程池里的线程是否都在执行任务，如果不是，则创建一个新的工作线程来执行任务，如果核心线程池里的线程都在执行任务，则进入下个流程。
2. 线程池判断工作队列是否已经满，如果工作队列没有满，则将新提交的任务存储在这个工作队列中，如果工作队列满了，则进入下个流程。
3. 线程池判断线程池的线程是否都处于工作状态，如果没有，则创建一个新的工作 线程来执行任务，如果已经满了，则交给饱和策略来处理这个任务。

```java
// 1. 创建线程池
   // 创建时，通过配置线程池的参数，从而实现自己所需的线程池
   Executor threadPool = new ThreadPoolExecutor(
                                              CORE_POOL_SIZE,
                                              MAXIMUM_POOL_SIZE,
                                              KEEP_ALIVE,
                                              TimeUnit.SECONDS,
                                              sPoolWorkQueue,
                                              sThreadFactory
                                              );

// 2. 向线程池提交任务：execute（）,summit用于提交有返回值的任务(Future)
    // 说明：传入 Runnable对象
       threadPool.execute(new Runnable() {
            @Override
            public void run() {
                ... // 线程执行任务
            }
        });

// 3. 关闭线程池shutdown() 
  threadPool.shutdown();
  

```

注意上面的关闭线程池，有两种方法

shutdown()和shutdownNow()，他们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法停止，但是他们存在一定的区别：

shutdownNow首先将线程池的状态设置为STOP，然后尝试停止所有正在执行或者暂停任务的线程，并返回等待执行任务的列表。

shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。

只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true，当所有任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true，至于应该用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。

### 常见的4类线程池

根据参数的不同配置，`Java`中最常见的线程池有4类：

- 定长线程池（`FixedThreadPool`）
- 定时线程池（`ScheduledThreadPool` ）
- 可缓存线程池（`CachedThreadPool`）
- 单线程化线程池（`SingleThreadExecutor`）

#### 定长线程池（FixedThreadPool）

只有核心线程 & 不会被回收、线程数量固定、任务队列无大小限制（超出的线程任务会在队列中等待）,通常应用于控制线程最大并发数场景中。

```java
// 1. 创建定长线程池对象 & 设置线程池线程数量固定为3
ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);

// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
  public void run(){
    System.out.println("执行任务啦");
     }
  };
        
// 3. 向线程池提交任务：execute（）
fixedThreadPool.execute(task);
        
// 4. 关闭线程池
fixedThreadPool.shutdown();
```

#### 定时线程池（ScheduledThreadPool ）

核心线程数量固定、非核心线程数量无限制（闲置时马上回收），通常应用于执行定时 / 周期性 任务场景中。

```java
// 1. 创建 定时线程池对象 & 设置线程池线程数量固定为5
ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);

// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
       public void run(){
              System.out.println("执行任务啦");
       }
};
// 3. 向线程池提交任务：schedule（）
scheduledThreadPool.schedule(task, 1, TimeUnit.SECONDS); // 延迟1s后执行任务
scheduledThreadPool.scheduleAtFixedRate(task,10,1000,TimeUnit.MILLISECONDS);// 延迟10ms后、每隔1000ms执行任务

// 4. 关闭线程池
scheduledThreadPool.shutdown();
```

#### 可缓存线程池（CachedThreadPool）

 只有非核心线程、线程数量不固定（可无限大）、灵活回收空闲线程（具备超时机制，全部回收时几乎不占系统资源）、新建线程（无线程可用时），通常在执行大量、耗时少的任务时使用。

```java
// 1. 创建可缓存线程池对象
ExecutorService cachedThreadPool = Executors.newCachedThreadPool();

// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
  public void run(){
        System.out.println("执行任务啦");
  }
};

// 3. 向线程池提交任务：execute（）
cachedThreadPool.execute(task);

// 4. 关闭线程池
cachedThreadPool.shutdown();

//当执行第二个任务时第一个任务已经完成
//那么会复用执行第一个任务的线程，而不用每次新建线程。
```

#### 单线程化线程池（SingleThreadExecutor）

只有一个核心线程（保证所有任务按照指定顺序在一个线程中执行，不需要处理线程同步的问题）,它常应用于不适合并发但可能引起IO阻塞性及影响UI线程响应的操作，如数据库操作，文件操作等。

```javascript
// 1. 创建单线程化线程池
ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();

// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
  public void run(){
        System.out.println("执行任务啦");
            }
    };

// 3. 向线程池提交任务：execute（）
singleThreadExecutor.execute(task);

// 4. 关闭线程池
singleThreadExecutor.shutdown();
```