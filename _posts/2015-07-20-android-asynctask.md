---
layout: post
title: Android源码之AsyncTask用法与原理
tags: AsyncTask 源码分析
categories: Android
date: 2015-07-20
---

在android开发中，主线程不能执行耗时操作，否则程序容易出现ANR，并且崩溃掉，所以我们常常会把主线程耗时操作放在子线程去做，同时把子线程执行的结果通过handler传递到主线程，执行刷新UI操作。否则就会抛出异常：

> android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.

上面过程，Android帮我们很好的封装了一个类AsyncTask，它可以很方便的让异步耗时任务放在子线程去执行，并且将执行结果返回给主线程，好让我们执行操作UI的逻辑。

## 基本用法

我们先来创建一个下载任务看下：

```java
class DownloadTask extends AsyncTask<Void, Integer, Boolean> {

    @Override
    protected void onPreExecute() {
        progressDialog.show();
    }

    @Override
    protected Boolean doInBackground(Void... params) {
        try {
            while (true) {
                int downloadPercent = doDownload();
                publishProgress(downloadPercent);
                if (downloadPercent >= 100) {
                    break;
                }
            }
        } catch (Exception e) {
            return false;
        }
        return true;
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        progressDialog.setMessage("当前下载进度：" + values[0] + "%");
    }

    @Override
    protected void onPostExecute(Boolean result) {
        progressDialog.dismiss();
        if (result) {
            Toast.makeText(context, "下载成功", Toast.LENGTH_SHORT).show();
        } else {
            Toast.makeText(context, "下载失败", Toast.LENGTH_SHORT).show();
        }
    }
}

// 执行
DownloadTask mDownloadTask = new DownloadTask();
mDownloadTask.execute();
```

### 参数

从上面代码中，我们知道，AsynTask是一个抽象类，使用AsynTask需要自定义自己的任务类并继承AsynTask。它接收三个泛型参数,都继承自Object：

1. Params 

   用于指定方法doInBackground方法中的参数类型以及实例运行execute方法中的参数类型。

1. Progress

   这个参数用于指定onProgressUpdate(Params... values)方法的参数类型。其实也就是指定后台任务的进度单位。示例中指定的是Integer，表示用整型来指定进度单位，如10%,20%,..,90%,100%。同样地，这里的values也是一个不定长参数。

1. Result

   这个参数指定doInBackground 方法的返回类型，同时也是 onPostExecute(Boolean result) 的参数类型。在这里指定的是Boolean，表示doInBackground结束后会返回一个Boolean类型的数据到onPostExecute中，接下来 onPostExecute就可以使用这个返回的数据来做UI更新的逻辑了。

### 方法

1. onPreExecute(): 主线程执行
   这个方法是AsyncTask最先执行的方法，在这里一般可以做一些简单的UI初始化操作，如进度条的显示等。
2. doInBackground(Params... params): 子线程执行
    这个方法是AsyncTask一个最重要的方法。这个方法不会由主线程来执行，因此可以把耗时的逻辑操作交给这个方法来处理。当这个方法结束后，能够把处理完的结果返回到主线程中。这时，主线程中的界面元素就能够使用该返回结果更新数据了。
3. publishProgress(Progress... values):  子线程执行
    这个方法用于通知任务进度的更新情况。举个例子，比如你想每完成10%就更新一次进度条，那就可以每隔10%调用一次这个函数来告诉AsynTask进度条可以更新了，AsynTask得到这个通知后就会尝试去执onProgressUpdate这个函数来更新进度显示了。
4. onProgressUpdate(Progress... values): 主线程执行
    这个方法用于更新进度的显示，如进度条的更新。
5. onPostExecute(Result result): 主线程执行
   这个方法用于UI组件的更新。doInBackground返回的结果会被传到这个方法中作为更新UI的数据。

## 源码分析

在进行分析之前需要对Java并发编程的一些知识有一些了解：Executor线程池、Callable、Future、FutureTask等

在示例中，我们先从mDownloadTask.execute()方法入手：

```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
                                                                   Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                                                + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                                                + " the task has already been executed "
                                                + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

可以看到，上面execute方法执行了executeOnExecutor方法，并传入了两个参数sDefaultExecutor和params,其中params参数是我们执行task任务时传入的，而sDefaultExecutor是Android自己实现的一个默认的线程池。稍后我们再看这个线程池。

在executeOnExecutor方法中，我们关注下最后几行代码，其中onPreExecute方法就是我们示例中覆写的方法了，它运行在主线程，而mWorker是AsyncTask的私有变量,它实现了Callable接口，这个接口类似于Runnable，工作在子线程，不过和后者区别是它可以返回结果。

```java
private final WorkerRunnable<Params, Result> mWorker;
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}
```

最后进入到exec.execute(mFuture)方法，其中exec是参数传进来的，也就是前面的sDefaultExecutor，而方法参数mFuture同样是AsyncTask的私有变量，它是一个FutureTask对象，该对象实现了Runnable和Future接口，内部持有Callable的引用，通过执行run方法间接调用Callable的call方法，并获得结果，通过get方法可以拿到：

```java
private final FutureTask<Result> mFuture;

mFuture = new FutureTask<Result>(mWorker) {
    @Override
    protected void done() {
        try {
            postResultIfNotInvoked(get());
        } catch (InterruptedException e) {
            android.util.Log.w(LOG_TAG, e);
        } catch (ExecutionException e) {
           throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
        } catch (CancellationException e) {
            postResultIfNotInvoked(null);
        }
    }
};
```

到这里我们看下上面的executor方法，它是一个线程池的方法，调用方是sDefaultExecutor，来看下它的实现：

```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();  // 其实是调用的FutureTask的run方法
                } finally {
                    scheduleNext();
                }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
         if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
 }

```

从上面这段代码可以看出，sDefaultExecutor本质就是一个默认的串行线程池，内部维护了一个任务队列ArrayDeque，所有任务都是按照入队的顺序一个一个执行，因为mFuture是FutureTask对象，实现了Runnable接口，所以提交到这里的FutureTask会通过ArrayDeque的offer方法被加入到任务队列队尾，紧接着执行scheduleNext方法，该方法内部通过ArrayDeque在队首取出一个任务，放入线程池执行。

前面看到，在创建FutureTask实例的时候传入了一个mWorker对象，它是WorkerRunnable<Params, Result>类型，本质上就是Callable任务。它是在AsyncTask构造函数中被实例化：

```java
public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);  // 返回结果
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

```

而我们前面提到的FutureTask实例也是在AsyncTask中被实例化。在mWorker的call方法中我们看到了doInBackground(mParams)方法，那么也印证了该方法在mFuture（FutureTask类型）的run方法中被执行。紧接着，我们看到了postResult方法

```java
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,  
    new AsyncTaskResult<Result>(this, result))
    message.sendToTarget();
    return result;
}
```

这个postResult其实是把result构造进一个Message对象，然后通过getHandler()获取一个handler处理这个消息。Handler处理消息其实也就是handleMessage()的重写了:

```java
private static Handler getHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler();
        }
        return sHandler;
    }
}

private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData)                      
                break;
        }
    }
}
```

在InternalHandler的实现中，我们看到了覆写的handleMessage方法，另外这里还有一种消息类型MESSAGE_POST_PROGRESS，它执行我们的onProgressUpdate(result.mData)来更新进度，这个handler是绑定在AsyncTask的构造函数中，所以onProgressUpdate也就执行在了主线程，而发送的消息是在publishProgress中

```java
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```

我们从实例中可以看到，publishProgress执行在doInBackground方法中，这里将消息包装成一个MESSAGE_POST_PROGRESS的Message然后通过handler回到主线程，处理进度条更新。最后在handleMessage中，执行finish方法：

```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED
}
```

这个方法执行了onPostExecute回调，又因为finish执行在主线程，所以该回调也是执行在主线程。

## 总结

1. 用户调用mDownloadTask.execute()方法，进入到executeOnExecutor方法中国，执行了onPreExecute方法，将任务交给sDefaultExecutor调度。

2. mFuture(FutureTask类型)配合mWorker(Callable实现)开启子线程，即在线程池执行mFuture的run方法，间接调用了mWorker的call方法，在其call方法中，执行了doInBackground方法。

3. 在第二步将执行结果交给内部单例InternalHandler处理返回结果并返回到主线程，根据Message处理onProgressUpdate()或onPostExecute()。