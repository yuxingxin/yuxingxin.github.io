---
layout: post
title: Java并发编程之Callable、Future和FutureTask
tags: 并发编程
categories: Java
date: 2015-07-14
---

## 基本概念

我们知道创建线程，经常使用两种方式，即：

- 直接继承Thread
- 另一种是实现Runnable接口

但是上面两种方式都有一个问题就是，在执行完任务之后无法直接获取执行的结果。而Callable的出现就是为了解决这个问题，下面是这两个接口的定义：

```java
// Callable.java
public interface Callable<V> {
    V call() throws Exception;
}

// Runnable.java
public interface Runnable {
    public abstract void run();
}
```

从上面源码中，我们可以看出来区别：

- Runnable没有返回值；Callable可以返回执行结果（泛型）
- Runnable异常只能在内部处理，不能往上继续抛出；Callable接口的call()方法允许抛出异常

Callable在使用过程中经常需要配合FutureTask或Future使用，因为它们可以对Callable任务执行取消，查询，获取结果等操作：

- 判断任务是否执行完成
- 取消任务并可以判断任务是否取消成功
- 获取任务执行的结果

下面是Future的定义：

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

FutureTask则实现了RunnableFuture接口，RunnableFuture继承了Runnable接口和Future接口

```java
public class FutureTask<V> implements RunnableFuture<V>{
    
}


public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

## 简单用法

1. 先来看下Runnable：

   ```java
   Runnable runnable = new Runnable() {
       @Override
       public void run() {
           System.out.println("线程执行中...");
       }
   };
   Thread thread = new Thread(runnable);
   thread.start();
   ```

1. 接下来是Callable

   - 配合Future

   ```java
   Callable <String> callable = new Callable <String> () {
       @Override
       public String call() throws Exception {
           System.out.println("线程执行中...");
           return "Success";
       }
   };
   ExecutorService service = Executors.newCachedThreadPool();
   Future<String> future = service.submit(callable);
   // 等待1秒，让线程执行
   try {
       Thread.sleep(1000);
   } catch (InterruptedException e) {
       e.printStackTrace();
   }
   if(futureTask.isDone()) {
       System.out.println("获取执行结果：" + future.get());
   }
   ```

   - 配合FutureTask

   ```java
   Callable <String> callable = new Callable <String> () {
       @Override
       public String call() throws Exception {
           System.out.println("线程执行中...");
           return "Success";
       }
   };
   FutureTask <String> futureTask = new FutureTask <String> (callable);
   new Thread(futureTask).start();
   // 第二种方式
   // ExecutorService service = Executors.newCachedThreadPool();
   // service.execute(futureTask);  // 执行Runnable无返回结果
   // Future<String> future = service.submit(futureTask); // 执行Runnable有返回结果
   
   // 等待1秒，让线程执行
   try {
       Thread.sleep(1000);
   } catch (InterruptedException e) {
       e.printStackTrace();
   }
   if(futureTask.isDone()) {
       System.out.println("获取执行结果：" + futureTask.get());
   }
   ```

   综上我们可以得出Runnable与Callable的一些区别：

   1. Runnable是自从java1.1就有了，而Callable是1.5之后才加上去的。

   1. Callable要实现的方法是call(),Runnable要实现的方法是run()。

   1. Callable的任务执行后可返回值，而Runnable的任务是不能返回值(是void)。

   1. call方法可以抛出异常，run方法不可以。

   1. 运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。

   1. 加入线程池运行，Runnable使用ExecutorService的execute方法，Callable使用submit方法。

   ## FutureTask

   FutureTask内部提供了一个Callable的引用，在执行构造函数时将其传入，当运行FutureTask任务时，会执行其run方法，在run方法里面会调用上面Callable引用的call方法，并获得一个结果。
   其部分源码如下：

   ```java
   public class FutureTask<V> implements RunnableFuture<V> {
       ...
       private Callable<V> callable;
       private Object outcome; // non-volatile, protected by state reads/writes
       public FutureTask(Callable<V> callable) {
           if (callable == null)
               throw new NullPointerException();
           this.callable = callable;
           this.state = NEW;       // ensure visibility of callable
       }
       public V get() throws InterruptedException, ExecutionException {
           int s = state;
           if (s <= COMPLETING)
               s = awaitDone(false, 0L);
           return report(s);
       }
       
       private V report(int s) throws ExecutionException {
           Object x = outcome;
           if (s == NORMAL)
               return (V)x;    // 返回结果
           if (s >= CANCELLED)
               throw new CancellationException();
           throw new ExecutionException((Throwable)x);
       }
       
       protected void set(V v) {
           if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
               outcome = v;
               U.putOrderedInt(this, STATE, NORMAL); // final state
               finishCompletion();
           }
       }
       
       public void run() {
           if (state != NEW ||
               !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
               return;
           try {
               Callable<V> c = callable;
               if (c != null && state == NEW) {
                   V result;
                   boolean ran;
                   try {
                       result = c.call();  // 执行Callable的call方法
                       ran = true;
                   } catch (Throwable ex) {
                       result = null;
                       ran = false;
                       setException(ex);
                   }
                   if (ran)
                       set(result);  //并把结果赋值给outcome
               }
           } finally {
               // runner must be non-null until state is settled to
               // prevent concurrent calls to run()
               runner = null;
               // state must be re-read after nulling runner to prevent
               // leaked interrupts
               int s = state;
               if (s >= INTERRUPTING)
                   handlePossibleCancellationInterrupt(s);
           }
       }
       
       ...
   }
   ```

   