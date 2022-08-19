---
layout: post
title: Android线程间通信之Handle机制原理
tags: Handler
categories: Android
date: 2016-03-12
---

在Android开发中，我们 经常会用到Handler，主要是子线程完成耗时操作后，通过Handler向主线程发送消息Message，用来刷新UI。我们都知道下面这两个原则：

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

    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
            ……
        }
    }

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
public static Message obtain() {
    // 保证线程安全
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            // flags为移除使用标志
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
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

而前面的如果callback不为空就通过handleCallback执行Runnable的run方法，注意这里的Runnable就是一个回调方法，和线程没有关系。综上：如果Message的callback为空，则一般通过sendMessage发送消息，交给handlerMessage方法来处理，而不为空则通过handleCallback来处理。另外Google官方不建议我们通过new Message来创建消息对象，而是通过obtain方法复用Message来获得，这里不得不提一下sPool这个消息池。我们看下Message是如何回收的：

```java
//Message.java
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    // 添加正在使用标志位，其他情况就除掉
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    //拿到同步锁，以避免线程不安全
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}

// Handler.java
public static Message obtain() {
    // 保证线程安全
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            // flags为移除使用标志
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

详细解释一下上面的sPool和next，将sPool看成一个指针，通过next来将对象组成一个链表，因为每次只需要从池子里拿出一个对象，所以不需要关心池子里具体有多少个对象，而是拿出当前这个sPool所指向的这个对象就可以了，sPool从思路上理解就是通过左右移动来完成复用和回收，

假设消息对象池为空，从new message开始，到这个message被取出使用后，准备回收 

> 1. **next=sPool**，因为消息对象池为空，所以此时sPool为null，同时next也为null。
> 2. **spool = this**，将当前这个message作为消息对象池中下一个被复用的对象。
> 3. **sPoolSize++**，默认为0，此时为1，将消息对象池的数量+1，这个数量依然是全系统共共享的。

这时候假设又调用了这个方法，之前的原来的第一个Message对象假定为以为**msg1**，依旧走到上面的循环。

> 1. **next=sPool**，因为消息对象池为msg1，所以此时sPool为msg1，同时next，即下一个Message对象也为msg1。
> 2. **sPool = this**，将当前这个message作为消息对象池中下一个被复用的对象。
> 3. **sPoolSize++**，此时为1，将消息对象池的数量+1，sPoolSize为2，这个数量依然是全系统共共享的。

以此类推，直到sPoolSize=50(MAX_POOL_SIZE = 50)，这里回收可以看到采用的是链表的头插法。

假设上面已经回收了一个Message对象，又从这里获取一个message，看看obtain如何复用？

> 1. 判断**sPool**是否为空，如果消息对象池为空，则直接new Message并返回
> 2. **Message m = sPool**，将消息对象池中的对象取出来，为m。
> 3. **sPool = m.next**，将消息对象池中的下一个可以复用的Message对象(m.next)赋值为消息对象池中的当前对象。(如果消息对象池就之前就一个，则此时sPool=null)
> 4. 将m.next置为null，因为之前已经把这个对象取出来了。
> 5. **m.flags = 0**，设置m的标记位，标记位正在被使用
> 6. **sPoolSize--**，因为已经把m取出了，这时候要把消息对象池的容量减一。

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

而MessageQueue的enqueueMessage方法中，可以看出MessageQueue其实是一个队列。另外还有一个问题就是，如果Message是一个阻塞消息，那么先postDelay10秒一个Runnable A，消息队列会一直阻塞，然后再 post一个Runnable B，B岂不是会等A执行完了再执行？正常使用时显然不是这样的，那么问题出在哪呢？我们先来看下上面的needWake变量，它被mBlocked赋值，而mBlocked是在上面MessageQueue的next 方法中被赋值:

```java
Message next() {
    ...
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
            ……
        }
    }
}
```

在上面的next方法内部，如果有阻塞（可能是没有消息或者是只有Delay的消息），就会把Blocked这个变量标记为true，在下一个Message插入队列的时候，会判断这个message的位置，如果在队首就会调用nativeWake()方法。另外postDelay能够精确的延迟指定时间，也可以从这里看出来，在enqueueMessage()的时候把应该执行的时间（上面Hanlder调用路径的第三步延迟已经加上了现有时间，所以叫when）设置到msg里面，即上面的msg.when，如果头部的这个Message是有延迟而且延迟时间没到的话（now < msg.when），会计算一下时间（保存为变量nextPollTimeoutMillis），然后在循环开始的时候判断如果这个Message有延迟，就调用nativePollOnce(ptr, nextPollTimeoutMillis)进行阻塞。nativePollOnce()的作用类似与object.wait()，只不过是使用了Native的方法对这个线程精确时间的唤醒。

那么上面那个问题就比较清晰了：

1. 先postDelay10秒一个Runnable A，消息插入队列，MessageQueue调用nativePollOnce阻塞，Looper阻塞。
2. 紧接着再post一个Runnable B，消息插入队列，判断现在A时间还没到，正在阻塞，就把B消息插入队列的头部，即A的前面，然后调用nativeWake()方法唤醒线程。
3. 当调用MessageQueue的next()方法取出消息时，重新开始读取消息链表，第一个消息B无延时，直接返回给Looper。
4. Looper处理完这个消息再次调用next方法，MessageQueue继续读取消息链表，第二个消息A还没到时间，计算一下剩余时间，如果大于0那么继续调用nativePollOnce()本地方法进行阻塞，直到阻塞时间到或者下一次有Message插入队列。

最后说下Handler消息的同步屏障机制

Handler发送的消息分为普通消息、屏障消息、异步消息，一旦Looper在处理消息时遇到屏障消息，那么就不再处理普通的消息，而仅仅处理异步的消息。不再使用屏障后，需要撤销屏障，不然就再也执行不到普通消息了。这样设计的目的是为了让某些特殊的消息得以更快被执行的机制。比如绘制界面，这种消息可能会明显的被用户感知到，稍有不慎就会引起卡顿、掉帧之类的，所以需要及时处理（可能消息队列中有大量的消息，如果像平时一样挨个进行处理，那绘制界面这个消息就得等很久，这是不想看到的）。

平时我们发送普通消息时，这个target是不能为空的，

```java
/MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
}
```

但是在发送屏障消息的时候，target是可以为空的，它本身仅仅是起屏蔽普通消息的作用，所以不需要target。MessageQueue中提供了postSyncBarrier()方法用于插入屏障消息。

```java
//MessageQueue.java
/**
 * @hide
 */
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        //这个token在移除屏障时会使用到
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        //在屏障的时间到来之前的普通消息，不会被屏蔽
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }

         //插入到单链表中
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

如上面代码：

- 屏障消息和普通消息区别在于屏幕没有target，普通消息有target是因为它需要将消息分发给对应的target，而屏幕不需要被分发，它就是用来挡住普通消息来保证异步消息优先处理的
- 屏障和普通消息一样可以根据时间来插入到消息队列中的适当位置，并且只会挡住它后面的同步消息的分发
- postSyncBarrier会返回一个token，利用这个token可以撤销屏障
- postSyncBarrier是hide的，使用它得用反射
- 插入普通消息会唤醒消息对了，但插入屏障不会

这时候屏障消息已经插入到队列中了，那么它是如何挡住普通消息而只去执行异步消息的呢？还是看下上面MessageQueue的next方法

```java
//MessageQueue.java
Message next() {
    ...
    int pendingIdleHandlerCount = -1;
    int nextPollTimeoutMillis = 0;
    for (;;) {
        //如有消息被插入到消息队列或者超时时间到，就被唤醒，否则会阻塞在这里
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //遇到屏障  它的target是空的
            if (msg != null && msg.target == null) {
                //找出屏障后面的异步消息，
                do {
                    prevMsg = msg;
                    msg = msg.next;
                    //isAsynchronous()返回true才是异步消息
                } while (msg != null && !msg.isAsynchronous());
            }

            //如果找到了异步消息
            if (msg != null) {
                if (now < msg.when) {
                    //还没到处理时间，再等一会儿
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    //到处理时间了，就从链表中移除，返回这个消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                //如果没有异步消息就一直休眠，等待被唤醒
                nextPollTimeoutMillis = -1;
            }
            ...
        }
        ...
    }
}
```

在MessageQueue中取下一个消息时，如果遇到屏障，就遍历消息队列，取最近的一个异步消息，然后返回出去。如果没有异步消息，则一直休眠在那里，等待着被唤醒。

上面postSyncBarrier()被标记位hide，google不让使用，但可以通过系统调用这个方法添加屏障（在ViewRootImpl中requestLayout时会使用到），添加异步消息有两种方法：

- 使用异步类型的Handler发送的全部Message都是异步的
- 通过Message的setAsynchronous()方法给Message标记异步

Handler 构造函数有一个参数async可以决定是否是异步Handler,但是被标记为@hide，从API28以后又提供了2个方法可以帮助创建异步的Handler，但是需要API版本是28以上。

```java
//Handler.java
/**
 * @hide
 */
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
public static Handler createAsync(@NonNull Looper looper) {
    if (looper == null) throw new NullPointerException("looper must not be null");
    return new Handler(looper, null, true);
}
public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback) {
    if (looper == null) throw new NullPointerException("looper must not be null");
    if (callback == null) throw new NullPointerException("callback must not be null");
    return new Handler(looper, callback, true);
}
```

而系统添加同步屏障是在View的绘制时，即在 viewRootImpl.requestLayout() 方法开始，这个方法会去执行上面的三大绘制任务，就是测量、布局和绘制。**调用requestLayout()方法之后，并不会马上开始进行绘制任务，而是会给主线程设置一个同步屏障，并设置 ASYNC 信号监听。当 ASYNC 信号的到来，会发送一个异步消息到主线程Handler，执行我们上一步设置的绘制监听任务，并移除同步屏障。**

```java
//ViewRootImpl.java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //插入屏障
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //监听Vsync信号，然后发送异步消息 -> 执行绘制任务
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

在等待Vsync信号的时候主线程什么事都没干，这样的好处是保证在Vsync信号到来时，绘制任务可以被及时执行，不会造成界面卡顿。

这样的话，我们发送的普通消息可能会被延迟处理，在Vsync信号到了之后，移除屏障，才得以处理普通消息。改善这个问题的办法是使用异步消息，发送异步消息之后，即时是在等待Vsync期间也可以执行我们的任务，让我们设置的任务可以更快得被执行（如有必要才这样搞，UI绘制高于一切）且减少主线程的Looper压力。

## 总结

1. 应用启动是从 ActivityThread 的 main 开始的，先是执行了 Looper.prepare()，该方法先是 new 了一个 Looper 对象，在私有的构造方法中又创建了 MessageQueue 作为此 Looper 对象的成员变量，Looper 对象通过 ThreadLocal 绑定在了MainThread 中，这样以来，Looper就和线程绑定在了一起；

2. 当我们创建 Handler 子类对象时，在构造方法中通过 ThreadLocal 获取绑定的 Looper 对象，并获取此 Looper 对象的成员变量 MessageQueue 作为该 Handler 对象的成员变量；

3. 在子线程中调用上一步创建的 Handler 子类对象的 sendMesage(msg) 或者post(Runnable)方法时，在该方法中将 msg 的 target 属性设置为Handler自己本身，同时调用成员变量 MessageQueue 对象的 enqueueMessag() 方法将 msg 放入 MessageQueue 中；

4. 主线程创建好之后，会执行 Looper.loop() 方法，该方法中获取与线程绑定的 Looper 对象，继而获取该 Looper 对象的成员变量 MessageQueue 对象，并开启一个会阻塞（不占用资源）的死循环，只要 MessageQueue 中有 msg，就会通过其next()获取该 msg，并执行 msg.target.dispatchMessage(msg) 方法（msg.target 即上一步引用的 handler 对象），此方法中调用了我们第二步创建 handler 子类对象时覆写的 handleMessage() 或者handleCallback()方法。这样一来，一个发送消息和处理消息的完整流程就梳理完了。

5. 就是这个消息发送的过程，我们在不同的线程发送消息，线程之间的资源是共享的。也就是任何变量在任何线程都可以修改，只要做并发操作就好了。上述代码中插入队列就是加锁的synchronized，Handler中我们使用的是同一个MessageQueue对象，同一时间只能一个线程对消息进行入队操作。消息存储到队列中后，主线程的Looper还在一直循环loop()处理。这样主线程就能拿到子线程存储的Message对象，在我们没有看见的时候完成了线程的切换。

6. 所以总结来讲就是：

   1.创建了一个Looper对象保存在ThreadLocal中。这个Looper同时持有一个MessageQueue对象。

   2.创建Handler获取到Looper对象和MessageQueue对象。在调用sendMessage方法的时候在不同的线程（子线程）中把消息插入MessageQueue队列。

   3.在主线程中（UI线程），调用Looper的loop()方法无限循环查询MessageQueue队列是否有消息保存了。有消息就取出来调用dispatchMessage()方法处理。这个方法最终调用了我们自己重写了消息处理方法handleMessage(msg);这样就完成消息从子线程到主线程的无声切换。