---
layout: post
title: Java多线程之并发工具类
tags: 多线程
categories: Java
date: 2016-03-05
---

## CountDownLatch

CountDownLatch 允许一个或多个线程等待其他线程完成操作。假如有一个需求是我们需要下载多组数据，此时可以考虑多线程，每个线程下载一组，直到所有数据现在完，这里如果要实现主线程等待所有线程完成下载工作，最简单就会用到join方法。：

```java
public static class MultiThreadDownload {
    public static void main(String[] args) throws InterruptedException {
        Thread task1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("下载数据1");
            }
        });
        Thread task2 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("下载数据2");
            }
        });
        task1.start();
        task2.start();
        task1.join();
        task2.join();
        System.out.println("所有数据下载完成");
    }
}
```

其中上面代码中join方法用于让当前执行线程等待join线程执行结束。其实现原理是不停检查join线程是否存活， 如果 join 线程存活则让当前线程永远等待。

而JDK1.5提供了CountDownLatch同于可以实现上面join的功能，并且比join功能还要多。

```java
public static class MultiThreadDownload {
    static CountDownLatch c = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {
        Thread task1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("下载数据1");
                c.countDown();
            }
        });
        Thread task2 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("下载数据2");
                c.countDown();
            }
        });
        task1.start();
        task2.start();
        c.await();
        System.out.println("所有数据下载完成");
    }
}

```

CountDownLatch 的构造函数接收一个int 类型的参数作为计数器，如果你想等待N个点完成，这里就传入N。
 当我们调用 CountDownLatch 的 countDown 方法时，N 就会 -1，CountDownLatch 的 await 方法会阻塞当前线程，直到 N 变成零。由于 countDown 方法可以用在任何地方，所以这里说的 N 个点，可以是 N个线程，也可以是 1个线程里的N个执行步骤。用在多个线程时，只需要把这个 CountDownLatch 的引用传递到线程里即可。

## 同步屏障CyclicBarrier

CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。 它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

CyclicBarrier 默认的构造方法是 CyclicBarrier(int parties)，其参数表示 屏障拦截的线程数量，每个线程调用 await 方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。

```java
public class Main {
   	private static CyclicBarrier c = new CyclicBarrier(2);
	public static void main(String[] args) {
        new Thread(new Thread() {
            @Override
            public void run() {
                try {
                    c.await();
                }catch(Exception e) {
                    e.printStackTrace();
                }
            }
        }).start()
            
         try {
             c.await();
         }catch(Exception e) { 
             e.printStackTrace();
         }
    }
}

```

由于主线程和子线程都由CPU决定，两个线程都有可能先执行，所以执行的顺序不确定。

CyclicBarrier 还提供一个更高级的构造函数CyclicBarrier(int parties，Runnable barrierAction)，用于在线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景。

```java
public static class Main {
    static CyclicBarrier c = new CyclicBarrier(2, new BarrierAction());

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
        try {
            c.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static class BarrierAction implements Runnable {
        @Override
        public void run() {
            System.out.println("执行后续工作");
        }
    }
}
```

CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。 所以CyclicBarrier能处理更为复杂的业务场景。 例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。

## 信号量Semaphore

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。

最简单的理解信号量就是，一个计数器、一个等待队列、两个方法（在Java实现的Semaphore中就是acquire和release）。
调用一次acquire方法，计数器就减1，如果此时计数器小于0，则阻塞当前线程，否则当前线程可继续执行。
调用一次release方法，计数器就加1，如果此时计数器小于等于0，则唤醒一个等待队列中的线程，并将其中等待队列中移除。

Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。如果需要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。

```java
public class Main {

    private static final int THREAD_COUNT = 30;

    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);

    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {

            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                        System.out.println("save data");
                        s.release();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```

代码中，虽然有 30 个线程执行，但只允许 10 个线程并发执行。Semaphore 的构造方法 Semaphore(int permits) 接受一个整型的数字，表示可用的许可证数量。Semaphore 的用法也很简单，首先线程使用 Semaphore 的 acquire() 方法获取一个许可证，使用完之后调用 release() 方法归还即可，还可以使用 tryAcquire() 方法尝试获取许可证

#### 实现互斥锁

```java
public class SemaphoreMutex {

    private static final Semaphore SEMAPHORE = new Semaphore(1);

    public static void main(String[] args) {
        SemaphoreMutex semaphoreMutex = new SemaphoreMutex();
        for (int i = 0; i < 10; i++) {
            new Thread(semaphoreMutex::method).start();
        }
    }

    @SneakyThrows
    public void method() {
        // 同时只会有一个线程执行此方法！
        SEMAPHORE.acquire();
        try {
            System.out.println(Thread.currentThread().getName() + "线程正在执行！");
            Thread.sleep(1000);
        } finally {
            SEMAPHORE.release();
            System.out.println(Thread.currentThread().getName() + "线程执行结束！");
        }
    }
}

```

从代码看上面每次都是一个线程执行结束后，才会有另一个开始执行，实现了互斥锁的功能。

## 线程间交换数据的Exchanger

Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交换。它提供一个同步点，在这个同步点两个线程可以交换彼此的数据。这两个线程通过exchange方法交换数据， 如果第一个线程先执行exchange方法，它会一直等待第二个线程也执行exchange，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。

假设现在有一个需求：我们需要将纸质银行流水通过人工的方式录入电子银行流水，为了避免错误，采用 AB 岗两人进行录入，录入完成后，系统需加载这两人录入的数据进行比较，看看是否录入一致

```java
public class Main {

    private static final Exchanger<String> exchanger = new Exchanger<>();

    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {

        threadPool.execute(new Runnable() {

            @Override
            public void run() {
                try {
                    // A录入银行流水数据
                    String A = "银行流水A";
                    exchanger.exchange(A);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        threadPool.execute(new Runnable() {

            @Override
            public void run() {
                try {
                    // B录入银行流水数据
                    String B = "银行流水B";
                    String A = exchanger.exchange(B);
                    System.out.println("A 和 B 数据是否一致：" + A.equals(B) + ", A 录入的是：" + A
                        + ", B 录入的是：" + B);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        threadPool.shutdown();
    }
}


```

如果两个线程一个没有执行exchange方法，则会一直等待，如果担心有特殊情况发送，避免一直等待，可以使用exchange(V x, longtimeout, TimeUnit unit）设置最大等待时长。