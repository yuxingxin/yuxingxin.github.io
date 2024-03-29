---
layout: post
title: Android源码之HandlerThread用法与原理
tags: 源码分析
categories: Android
date: 2015-07-26
---

在android开发中，当我们用Thread和Handler构造一个消息循环的需求时，往往这样做：

```java
Handler mHandler;
private void createThreadWithHandler() {
  new Thread() {
      @Override
        public void run() {
            super.run();
            Looper.prepare();  // 创建一个与当前线程绑定的Looper实例
            mHandler = new Handler(Looper.myLooper()); // 创建Handler实例
            Looper.loop(); // 生成消息循环
        }
    }.start();
}
```

上面几行代码也加了注释，而Google也在这里给我们设计了一个便捷的类，方便我们管理线程的交互。一起来看下它的使用

## 使用

```java
public class DownloadHandlerThread extends HandlerThread implements Handler.Callback {

    private Handler mWorkHandler;
    private Handler mUIHandler;

    public DownloadHandlerThread(String name) {
        super(name);
    }

    @Override
    protected void onLooperPrepared() {
        super.onLooperPrepared();
        mWorkHandler = new Handler(getLooper(), this);

        Message message = mWorkHandler.obtainMessage();
        Bundle bundle = new Bundle();
        bundle.putString("URL", "https://XXX");
        message.setData(bundle);
        mWorkHandler.sendMessage(message);
    }

    public void setUIHandler(Handler uiHandler){
        this.mUIHandler = uiHandler;
    }

    @Override
    public boolean handleMessage(@NonNull Message msg) {
        if(msg == null || msg.getData() == null) {
            return false;
        }
        String url = (String)msg.getData().get("URL");

        //省略执行下载任务的逻辑
        

        //通知主线程更新UI
        this.mUIHandler.sendEmptyMessage(1);

        return true;
    }
}

//MainActivity.java
DownloadHandlerThread downloadHandlerThread = new DownloadHandlerThread("DownloadTask");
downloadHandlerThread.setUIHandler(this);
downloadHandlerThread.start();
```
上面实现了一个简单的下载任务，子线程Handler发送下载任务的通知,在handleMessage中处理耗时操作，完了之后借助主线程Handler发送通知消息操作UI。

## 源码分析
```java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}

public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }

    boolean wasInterrupted = false;

    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
                wasInterrupted = true;
            }
        }
    }

    /*
         * We may need to restore the thread's interrupted flag, because it may
         * have been cleared above since we eat InterruptedExceptions
         */
    if (wasInterrupted) {
        Thread.currentThread().interrupt();
    }

    return mLooper;
}
```

在run方法内部调用Looper.prepare方法，创建了一个Looper对象和当前线程绑定，另外在创建Looper的过程中，也同时创建了一个消息队列MessageQueue，然后通过Hander的消息的方式通知HandlerThread执行下一个具体的任务，这里通过Looper.loop方法实现消息循环，同时我们也可以使用quit或者quitSafely方法来结束循环。这里注意下quit方法可能会让正在发送的消息发送失败，所以如果还有延迟的任务没有结束可以考虑使用后者quitSafely方法。

```java
public boolean quit() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quit();
        return true;
    }
    return false;
}

public boolean quitSafely() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quitSafely();
        return true;
    }
    return false;
}
```

## 总结

1. 当我们使用HandlerThread构造函数创建一个对象，同时执行它的run方法，这时候在run方法内部，创建了一个Looper对象，并和当前线程绑定，同时也初始化了一个消息队列MessageQueue，并通过调用Looper的loop方法实现消息循环。
2. 根据上一步创建的Looper对象传入我们在主线程创建的Handler，就能将子线程的消息发送到MessageQueue队列，经Looper不断的取出消息交给我们的handler来处理。
3. 最后我们在不用的时候可以通过quit或者quitSafely终止这个循环。
4. HandlerThread 是一个自带 Looper 的线程，因此只能作为子线程使用。
5. HandlerThread必须配合Handler使用，通过覆写Handler的callback开实现HanderThread中做的事情
6. 子线程的Handler与HandlerThread建立关系是通过构造子线程Handler时传入HandlerThread的Looper，所以在次之前，必须先调用HandlerThread的run方法，让Looper创建出来。