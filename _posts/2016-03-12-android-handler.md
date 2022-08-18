---
layout: post
title: Android线程间通信之Handle机制原理
tags: Handler
categories: Android
date: 2016-03-12
---

在Android开发中，我们经常会用到Handler，主要是子线程完成耗时操作后，通过Handler向主线程发送消息Message，用来刷新UI。我们都知道下面这两个原则：

1. 不能在子线程更新UI
2. 不能在主线程执行耗时操作

Android中，界面主要是由主线程绘制的，所以界面的更新一般都限制在主线程内，这个异常是在viewRootIimpl.checkThread()方法中抛出来的，如果我们在它还没创建出来的时候就可以偷偷更新ui了。阅读过Activity启动流程的读者知道，ViewRootImpl是在onCreate方法之后被创建的，所以我们可以在onCreate方法中创建个子线程更新UI，那么就可以绕过这个异常限制，但是往往我们也不会这样做，谷歌这样设计也是为了App更加的安全。不能在子线程更新UI是因为会产生不可预期的后果，这一点还是由于线程安全引起的，而传统的加锁来解决的话，我们知道，加锁很重，会有性能损耗，尤其是对于UI操作来说。

在主线程执行耗时操作，这样很容易导致ANR，所以我们通常会把耗时操作放在子线程，然后子线程执行完会返回结果，这时候更新UI如果放在主线程，就需要切换线程来执行。综合上面两点，handler也就显得越来越重要了，我们先来看下一般在程序中如何使用它：

```java
public class MainActivity extends AppComposeActivity{
    ...;
    // 第一种方法：使用callBack创建handler
    public void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        Handler handler = Handler(Looper.myLooper(),new CallBack(){
            @Override
            public Boolean handleMessage(Message msg) {
                TODO("Not yet implemented")
            }
        });
    }
    
    // 第二种方法：继承Handler并重写handlerMessage方法
    static MyHandler extends Hanlder{
        public MyHandler(Looper looper){
            super(looper);
        }
        @Override
        public void handleMessage(Message msg){
            super.handleMessage(msg);
            // TODO(重写这个方法)
        }
    }
}
```

我们一点一点分析源码来分析：

## 先从构造函数

```java
// Handler.java
public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
		
    	// 初始化Looper
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
    	// 初始化MessageQueue
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
}
```

Handler中最重要的两个对象都是由Looper提供，具体代码如下：

```java
// Looper.java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

mylooper方法通过一个线程本地变量来取出Looper对象。而MessageQueue是我们的应用启动时初始化的，我们都知道当Activity启动时，ActivityThread的main方法是一个新的App进程入口，具体实现：

```java
// ActivityThread.java
public static void main(String[] args) {
    ……
    Looper.prepareMainLooper();
    ……
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                                            LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    ……
    Looper.loop();
}

```

上面代码中Looper.prepareMainLooper()就是初始化当前Looper对象， Looper.loop()方法开启无限循环。

```java
// Looper.java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

……

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}


private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

上面代码中prepare方法创建了一个Looper对象，并且把它存储在了线程本地变量中，和当前线程绑定在了一起。

而MessageQueue在Looper构造函数中创建了。上面第8行sMainLooper从sThreadLocal变量中取出Looper对象并赋给它。

注意prepare方法中会对Looper进行判断，如果已经存在，就会抛出上面异常，即一个线程中Looper.prepare()只能调用一次，实例如下：

```java
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        Looper.prepare()
    }
}
```

上面代码中由于在Activity启动时，Looper.prepare()方法已经在main方法中执行过一次，所以再次执行就会报错，所以一个线程只有一个Looper，又因为MessageQueue在Looper构造函数中被初始化，所以MessageQueue在一个线程中也只有一个。后续我们通过Handler发送的消息都会被发送到MessageQueue中。

## Looper.loop

接着上面，Looper.loop开启了一个无限循环，这个无限循环也是Android App进程能够保持运行的原因，它并不会阻塞UI线程，反而还能接受来自屏幕的各种事件。这一点可以通过MessageQueue的next方法可以看出：

```java
//Looper.java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    
    ……
        
    for (;;) {
        if (!loopOnce(me, ident, thresholdOverride)) {
            return;
        }
    }
}

private static boolean loopOnce(final Looper me,
            final long ident, final int thresholdOverride) {
    
    Message msg = me.mQueue.next(); // might block
    if (msg == null) {
        // No message indicates that the message queue is quitting.
        return false;
    }

    ……

    try {
        msg.target.dispatchMessage(msg);
        ……
    } 
    ……

    return true;
}


// MessageQueue.java
Message next() {
    ……

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

     nativePollOnce(ptr, nextPollTimeoutMillis);
}
```

上面next方法中nativePollOnce 方法是一个 native 方法，当调用此 native 方法时，主线程会释放 CPU 资源进入休眠状态，直到下条消息到达或者有事务发生，通过往 pipe 管道写端写入数据来唤醒主线程工作，这里采用的 epoll 机制。另外每次通过next()方法从MessageQueue中取出一个Message消息，如果消息不为空，则通过dispatchMessage方法分发出去，而这个target就是我们前面的Handler对象，这一点我们可以从Message源码中看出来。

```java
public final class Message implements Parcelable {
    public int what;

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg1;

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg2;
    
    ……
        
    Handler target;
    ……
}
```

而进入到Handler的dispatchMessage方法，我们看下：

```java
//Handler.java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

首先判断msg.callback是否为空，而这个callback就是我们post一个Runnable，如果是普通对象则通过handleMessage方法来处理，而这个方法也正是我们创建Handler时需要覆写的方法。

```java
// Handler.java
public final boolean post(@NonNull Runnable r) {
    return  sendMessageDelayed(getPostMessage(r), 0);
}
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private static void handleCallback(Message message) {
    message.callback.run();
}
```

而前面的如果callback不为空就通过handleCallback执行Runnable的run方法，注意这里的Runnable就是一个回调方法，和线程没有关系。

综上：如果Message的callback为空，则一般通过sendMessage发送消息，交给handlerMessage方法来处理，而不为空则通过handleCallback来处理。

## Handler

上面Message中的target是如何赋值呢 ？我们接着往下看消息的发送相关代码：

```java
// Handler.java
public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

通过sendMessage方法，经过几层调用，最后我们来到了sendMessageAtTime方法中，最终会调用enqueueMessage方法将Message插入到消息队里中，即前面在ActivityThread中通过Looper创建的MessageQueue中去

接着往下看这个enqueueMessage方法：

```java
// Handler.java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
                               long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}

// MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    synchronized (this) {
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

上面代码可以看到，在Handler的enqueueMessage方法中，将Handler自身设置为msg.target对象，因此就对应上了前面的msg.target.dispatchMessage方法，即会用此Handler来调用dispatchMessage方法。

而MessageQueue的enqueueMessage方法中，可以看出MessageQueue其实是一个队列。上面就是完整的一个发送消息和处理消息的过程。

## 总结

1. 应用启动是从 ActivityThread 的 main 开始的，先是执行了 Looper.prepare()，该方法先是 new 了一个 Looper 对象，在私有的构造方法中又创建了 MessageQueue 作为此 Looper 对象的成员变量，Looper 对象通过 ThreadLocal 绑定在了MainThread 中，这样以来，Looper就和线程绑定在了一起；

1. 当我们创建 Handler 子类对象时，在构造方法中通过 ThreadLocal 获取绑定的 Looper 对象，并获取此 Looper 对象的成员变量 MessageQueue 作为该 Handler 对象的成员变量；

1. 在子线程中调用上一步创建的 Handler 子类对象的 sendMesage(msg) 或者post(Runnable)方法时，在该方法中将 msg 的 target 属性设置为Handler自己本身，同时调用成员变量 MessageQueue 对象的 enqueueMessag() 方法将 msg 放入 MessageQueue 中；

1. 主线程创建好之后，会执行 Looper.loop() 方法，该方法中获取与线程绑定的 Looper 对象，继而获取该 Looper 对象的成员变量 MessageQueue 对象，并开启一个会阻塞（不占用资源）的死循环，只要 MessageQueue 中有 msg，就会通过其next()获取该 msg，并执行 msg.target.dispatchMessage(msg) 方法（msg.target 即上一步引用的 handler 对象），此方法中调用了我们第二步创建 handler 子类对象时覆写的 handleMessage() 或者handleCallback()方法。这样一来，一个发送消息和处理消息的完整流程就梳理完了。

2. 就是这个消息发送的过程，我们在不同的线程发送消息，线程之间的资源是共享的。也就是任何变量在任何线程都可以修改，只要做并发操作就好了。上述代码中插入队列就是加锁的synchronized，Handler中我们使用的是同一个MessageQueue对象，同一时间只能一个线程对消息进行入队操作。消息存储到队列中后，主线程的Looper还在一直循环loop()处理。这样主线程就能拿到子线程存储的Message对象，在我们没有看见的时候完成了线程的切换。

3. 所以总结来讲就是：

   1.创建了一个Looper对象保存在ThreadLocal中。这个Looper同时持有一个MessageQueue对象。

   2.创建Handler获取到Looper对象和MessageQueue对象。在调用sendMessage方法的时候在不同的线程（子线程）中把消息插入MessageQueue队列。

   3.在主线程中（UI线程），调用Looper的loop()方法无限循环查询MessageQueue队列是否有消息保存了。有消息就取出来调用dispatchMessage()方法处理。这个方法最终调用了我们自己重写了消息处理方法handleMessage(msg);这样就完成消息从子线程到主线程的无声切换。
   