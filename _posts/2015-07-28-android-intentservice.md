---
layout: post
title: Andriod源码之IntentService用法与原来
tags: 源码分析
categories: Android
date: 2015-07-28
---

IntentService从名字来看就知道是一个Service，它是Service的子类，由于在Service里不能执行耗时操作，所以Google设计了一个IntentService，它在IntentService内部维护了一个工作线程来处理耗时操作，其实也就是HandlerThread，当任务执行完成后，IntentService会自动停止。先来看下如何使用：

```java
public class MyService extends IntentService {
    /**
     * Creates an IntentService.  Invoked by your subclass's constructor.
     *
     * @param name Used to name the worker thread, important only for debugging.
     */
    public MyService(String name) {
        super(name);
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {

        String action = intent.getStringExtra("action");

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
// 记得在AndroidManifest.xml注册

Intent intent = new Intent(this,MyService.class);
intent.putExtra("action", "download");
startService(intent);

```

## 源码分析

```java
public abstract class IntentService extends Service {
    ...

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    ...
}
```

如果之前对HandlerThread和Handler机制比较了解，上面源码也比较简单，当我们启动服务时，在onCreate方法里，其实就是HandlerThread的使用，我们在ServiceHandler里处理传过来的消息，并暴露给子类去覆写onHandleIntent来实现，这个方法我们可以执行我们的工作任务或者耗时操作。然后我们在onStart方法中，通过Message将Intent任务请求，发送给mServiceHandler，Intent任务请求就是我们上面的startActivity(intent)传入的intent。mServiceHandler接受任务请求，调用onHandlerIntent方法处理请求任务，处理完所有请求后，调用stopSelf()结束IntentService，因此我们不需要再额外调用停止服务的操作。

