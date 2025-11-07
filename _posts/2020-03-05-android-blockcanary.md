---
layout: post
title: Android三方开源库之BlockCanary源码分析
tags: 源码分析
categories: Android
date: 2020-03-05
---

我们手机屏幕帧率通常是60，也就意味着每秒钟有60个画面出现，即16.6ms就要有一个画面渲染出来，Android系统每隔16.6ms发出一个Vsync信号，触发对View进行渲染，如果在这个时间内渲染成功，那么画面正常显示，否则就会出现丢帧的情况，如果掉帧频率很高，也就导致了卡顿。

我们回顾一下View刷新机制，App启动时，会通过ActivityThread类的main方法，创建一个主线程Looper，并通过Looper.loop方法不断轮询，从MessageQueue队列中取出Message来更新UI，而UI更新往往会通过ViewRootImpl类的scheduleTraversals方法来进行一次View树的遍历绘制，最终通过Choreographer的postCallback将该绘制任务添加到待执行队列里面，由主线程looper的loop方法不断取出消息执行任务。

```java
// Looper.java

public static void loop() {
    final Looper me = myLooper();
    ...
    // 获取当前Looper的消息队列
    final MessageQueue queue = me.mQueue;
    ...
    for (; ; ) {
        // 取出一个消息
        Message msg = queue.next(); // might block
        ...
        // "此mLogging可通过Looper.getMainLooper().setMessageLogging方法设置自定义"
        final Printer logging = me.mLogging;
        if (logging != null) {// 消息处理前
            // "若mLogging不为null，则此处可回调到该类的println方法"
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                                msg.callback + ": " + msg.what);
        }

        ...
        try {
           // 消息处理
           msg.target.dispatchMessage(msg);
           dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
           if (traceTag != 0) {
               Trace.traceEnd(traceTag);
           }
        }
        ...

        if (logging != null) {// 消息处理后
            // "消息处理后，也可调用logging的println方法"
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        ...
    }
}

```

通过上面方法可以看出，最终的消息处理发生在dispatchMessage方法中，所以对于设置一个Printer可以记录该方法的耗时。那么其实BlockCanary原理其实也是这样，通过自定义Printer来实现println方法，然后在println方法中监控是否有卡顿发生，从上面也可以发现，logging.println成对出现在消息处理方法的前后，那么就可以通过在自定义的pringln方法中定义标识来分辨消息前后，并计算时间差与我们自己设置的阈值进行对比，从而判断卡顿是否发生。

## 源码分析

BlockCanary通过install方法完成初始化，并执行start方法:

```java
// BlockCanary.java

public static BlockCanary install(Context context, BlockCanaryContext blockCanaryContext) {
    // 根据用户配置参数进行初始化
    BlockCanaryContext.init(context, blockCanaryContext);
    // 开启配置用户消息栏
    setEnabled(context, DisplayActivity.class, BlockCanaryContext.get().displayNotification());
    return get();
}
```

上面方法中先根据用户继承的BlockCanaryContext，配置参数初始化，然后开启消息栏，最后通过get方法返回一个BlockCanary单例对象。

在BlockCanary构造方法中：

```java
// BlockCanary.java
private BlockCanary() {
    BlockCanaryInternals.setContext(BlockCanaryContext.get());
    mBlockCanaryCore = BlockCanaryInternals.getInstance();
    mBlockCanaryCore.addBlockInterceptor(BlockCanaryContext.get());
    if (!BlockCanaryContext.get().displayNotification()) {
        return;
    }
    mBlockCanaryCore.addBlockInterceptor(new DisplayService());
}
```

上面代码先是初始化blockCanaryInternals调度类，然后为该类添加拦截器责任链，当用户开启消息栏通知，就在卡顿发生时通过DisplayService发送通知栏消息。

紧接着我们看下这个调度类的构造方法：

```java
// BlockCanaryInternals.java
public BlockCanaryInternals() {
    // 初始化栈采集器
    stackSampler = new StackSampler(
        Looper.getMainLooper().getThread(),
        sContext.provideDumpInterval());
	// 实例化CPU采集器
    cpuSampler = new CpuSampler(sContext.provideDumpInterval());

    // 设置LooperMonitor，并实现onBlockEvent回调，在触发阈值时执行
    setMonitor(new LooperMonitor(new LooperMonitor.BlockListener() {

        @Override
        public void onBlockEvent(long realTimeStart, long realTimeEnd,
                                 long threadTimeStart, long threadTimeEnd) {
            // Get recent thread-stack entries and cpu usage
            ArrayList<String> threadStackEntries = stackSampler
                .getThreadStackEntries(realTimeStart, realTimeEnd);
            if (!threadStackEntries.isEmpty()) {
                BlockInfo blockInfo = BlockInfo.newInstance()
                    .setMainThreadTimeCost(realTimeStart, realTimeEnd, threadTimeStart, threadTimeEnd)
                    .setCpuBusyFlag(cpuSampler.isCpuBusy(realTimeStart, realTimeEnd))
                    .setRecentCpuRate(cpuSampler.getCpuRateInfo())
                    .setThreadStackEntries(threadStackEntries)
                    .flushString();
                LogWriter.save(blockInfo.toString());

                if (mInterceptorChain.size() != 0) {
                    for (BlockInterceptor interceptor : mInterceptorChain) {
                        interceptor.onBlock(getContext().provideContext(), blockInfo);
                    }
                }
            }
        }
    }, getContext().provideBlockThreshold(), getContext().stopWhenDebugging()));

    LogWriter.cleanObsolete();
}

```

在该构造函数中进行一系列的初始化操作，初始化栈采集器，初始化CPU采集器，初始化LooperMonitor，当准备工作完成后，就执行BlockCanary的start方法：

```java
// BlockCanary.java
public void start() {
    if (!mMonitorStarted) {
        mMonitorStarted = true;
        Looper.getMainLooper().setMessageLogging(mBlockCanaryCore.monitor);
    }
}
```

在这里设置了主线程Looper的setMessageLogging方法，参数monitor就是上面创建的LooperMonitor实例。

也就是我们前面介绍的自定义Printer：

```java
class LooperMonitor implements Printer {
    ...
    @Override
    public void println(String x) {
        if (mStopWhenDebugging && Debug.isDebuggerConnected()) {
            return;
        }
        if (!mPrintingStarted) {
            // 记录开始时间
            mStartTimestamp = System.currentTimeMillis();
            mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
            mPrintingStarted = true;
            // 开始采集栈和CPU信息
            startDump();
        } else {
            // 记录结束时间
            final long endTime = System.currentTimeMillis();
            mPrintingStarted = false;
            // 判断耗时是否超过阈值
            if (isBlock(endTime)) {
                notifyBlockEvent(endTime);
            }
            stopDump();
        }
    }
    private boolean isBlock(long endTime) {
        return endTime - mStartTimestamp > mBlockThresholdMillis;
    }
    private void notifyBlockEvent(final long endTime) {
        final long startTime = mStartTimestamp;
        final long startThreadTime = mStartThreadTimestamp;
        final long endThreadTime = SystemClock.currentThreadTimeMillis();
        HandlerThreadFactory.getWriteLogThreadHandler().post(new Runnable() {
            @Override
            public void run() {
                mBlockListener.onBlockEvent(startTime, endTime, startThreadTime, endThreadTime);
            }
        });
    }
    
    private void startDump() {
        if (null != BlockCanaryInternals.getInstance().stackSampler) {
            BlockCanaryInternals.getInstance().stackSampler.start();
        }

        if (null != BlockCanaryInternals.getInstance().cpuSampler) {
            BlockCanaryInternals.getInstance().cpuSampler.start();
        }
    }

    private void stopDump() {
        if (null != BlockCanaryInternals.getInstance().stackSampler) {
            BlockCanaryInternals.getInstance().stackSampler.stop();
        }

        if (null != BlockCanaryInternals.getInstance().cpuSampler) {
            BlockCanaryInternals.getInstance().cpuSampler.stop();
        }
    }
        
}
```

上面代码通过成员变量mPrintingStarted来判断方法处理前和后，根据我们设置的阈值mBlockThresholdMillis来判断是否产生卡顿，该值默认是3s，如果发生卡顿，则进入notifyBlockEvent方法，通过调用HandlerThread内部创建的Handler的post方法，将任务交给子线程来处理。该任务执行的逻辑即是mBlockListener的onBlockEvent方法

```java
// HandlerThreadFactory.java
final class HandlerThreadFactory {

    private static HandlerThreadWrapper sLoopThread = new HandlerThreadWrapper("loop");
    private static HandlerThreadWrapper sWriteLogThread = new HandlerThreadWrapper("writer");

    private HandlerThreadFactory() {
        throw new InstantiationError("Must not instantiate this class");
    }

    public static Handler getTimerThreadHandler() {
        return sLoopThread.getHandler();
    }

    public static Handler getWriteLogThreadHandler() {
        return sWriteLogThread.getHandler();
    }

    private static class HandlerThreadWrapper {
        private Handler handler = null;

        public HandlerThreadWrapper(String threadName) {
            HandlerThread handlerThread = new HandlerThread("BlockCanary-" + threadName);
            handlerThread.start();
            handler = new Handler(handlerThread.getLooper());
        }

        public Handler getHandler() {
            return handler;
        }
    }
}
```

紧接着在该方法里面，从栈采集器里面获取记录信息，然后构建BlockInfo对象，并将该对象信息写入日志文件。最后遍历拦截器进行通知，如果还有DisplayService，就会发送前台通知。

另外上面开始后会调用startDump方法，该方法会开启采集栈和CPU信息，处理工作分别是由StackSampler和CpuSampler类负责，通过调用start方法开始处理：

```java
// AbstractSampler.java
abstract class AbstractSampler {

    private static final int DEFAULT_SAMPLE_INTERVAL = 300;

    protected AtomicBoolean mShouldSample = new AtomicBoolean(false);
    protected long mSampleInterval;

    private Runnable mRunnable = new Runnable() {
        @Override
        public void run() {
            doSample();
			// 延迟卡顿阈值执行任务
            if (mShouldSample.get()) {
                HandlerThreadFactory.getTimerThreadHandler()
                        .postDelayed(mRunnable, mSampleInterval);
            }
        }
    };

    public AbstractSampler(long sampleInterval) {
        if (0 == sampleInterval) {
            sampleInterval = DEFAULT_SAMPLE_INTERVAL;
        }
        mSampleInterval = sampleInterval;
    }

    public void start() {
        if (mShouldSample.get()) {
            return;
        }
        mShouldSample.set(true);
        // 移除上一次任务
        HandlerThreadFactory.getTimerThreadHandler().removeCallbacks(mRunnable);
        // 延迟卡顿阀值*0.8 的时间执行Runnable
        HandlerThreadFactory.getTimerThreadHandler().postDelayed(mRunnable,
                BlockCanaryInternals.getInstance().getSampleDelay());
    }

    public void stop() {
        if (!mShouldSample.get()) {
            return;
        }
        mShouldSample.set(false);
        HandlerThreadFactory.getTimerThreadHandler().removeCallbacks(mRunnable);
    }

    abstract void doSample();
}
```

在start方法中，会延时执行doSample方法，该方法是抽象方法，由StackSampler和CpuSampluer覆写:

```java
// StackSampler.java
protected void doSample() {
    StringBuilder stringBuilder = new StringBuilder();

    for (StackTraceElement stackTraceElement : mCurrentThread.getStackTrace()) {
        stringBuilder
            .append(stackTraceElement.toString())
            .append(BlockInfo.SEPARATOR);
    }
	
    // 根据LRU算法来移除最早添加进来的数据
    synchronized (sStackMap) {
        if (sStackMap.size() == mMaxEntryCount && mMaxEntryCount > 0) {
            sStackMap.remove(sStackMap.keySet().iterator().next());
        }
        // 以当前时间为key，存储堆栈信息
        sStackMap.put(System.currentTimeMillis(), stringBuilder.toString());
    }
}

// CpuSampler.java
@Override
protected void doSample() {
    BufferedReader cpuReader = null;
    BufferedReader pidReader = null;

    try {
        cpuReader = new BufferedReader(new InputStreamReader(
            new FileInputStream("/proc/stat")), BUFFER_SIZE);
        String cpuRate = cpuReader.readLine();
        if (cpuRate == null) {
            cpuRate = "";
        }

        if (mPid == 0) {
            mPid = android.os.Process.myPid();
        }
        pidReader = new BufferedReader(new InputStreamReader(
            new FileInputStream("/proc/" + mPid + "/stat")), BUFFER_SIZE);
        String pidCpuRate = pidReader.readLine();
        if (pidCpuRate == null) {
            pidCpuRate = "";
        }
		// 解析CPU信息
        parse(cpuRate, pidCpuRate);
    } catch (Throwable throwable) {
        Log.e(TAG, "doSample: ", throwable);
    } finally {
        try {
            if (cpuReader != null) {
                cpuReader.close();
            }
            if (pidReader != null) {
                pidReader.close();
            }
        } catch (IOException exception) {
            Log.e(TAG, "doSample: ", exception);
        }
    }
}

// CpuSampler.java
private void parse(String cpuRate, String pidCpuRate) {
    String[] cpuInfoArray = cpuRate.split(" ");
    if (cpuInfoArray.length < 9) {
        return;
    }

    long user = Long.parseLong(cpuInfoArray[2]);
    long nice = Long.parseLong(cpuInfoArray[3]);
    long system = Long.parseLong(cpuInfoArray[4]);
    long idle = Long.parseLong(cpuInfoArray[5]);
    long ioWait = Long.parseLong(cpuInfoArray[6]);
    long total = user + nice + system + idle + ioWait
        + Long.parseLong(cpuInfoArray[7])
        + Long.parseLong(cpuInfoArray[8]);

    String[] pidCpuInfoList = pidCpuRate.split(" ");
    if (pidCpuInfoList.length < 17) {
        return;
    }

    long appCpuTime = Long.parseLong(pidCpuInfoList[13])
        + Long.parseLong(pidCpuInfoList[14])
        + Long.parseLong(pidCpuInfoList[15])
        + Long.parseLong(pidCpuInfoList[16]);

    if (mTotalLast != 0) {
        StringBuilder stringBuilder = new StringBuilder();
        long idleTime = idle - mIdleLast;
        long totalTime = total - mTotalLast;

        stringBuilder
            .append("cpu:")
            .append((totalTime - idleTime) * 100L / totalTime)
            .append("% ")
            .append("app:")
            .append((appCpuTime - mAppCpuTimeLast) * 100L / totalTime)
            .append("% ")
            .append("[")
            .append("user:").append((user - mUserLast) * 100L / totalTime)
            .append("% ")
            .append("system:").append((system - mSystemLast) * 100L / totalTime)
            .append("% ")
            .append("ioWait:").append((ioWait - mIoWaitLast) * 100L / totalTime)
            .append("% ]");

        synchronized (mCpuInfoEntries) {
            mCpuInfoEntries.put(System.currentTimeMillis(), stringBuilder.toString());
            if (mCpuInfoEntries.size() > MAX_ENTRY_COUNT) {
                for (Map.Entry<Long, String> entry : mCpuInfoEntries.entrySet()) {
                    Long key = entry.getKey();
                    mCpuInfoEntries.remove(key);
                    break;
                }
            }
        }
    }
    mUserLast = user;
    mSystemLast = system;
    mIdleLast = idle;
    mIoWaitLast = ioWait;
    mTotalLast = total;

    mAppCpuTimeLast = appCpuTime;
}
```

综上，可以看出按照固定时间间隔循环执行采集任务，在结束时调用stop方法移除该任务。

## 总结

最后用一张图总结其工作流程图：

![image-20220902162859753](https://tva1.sinaimg.cn/large/e6c9d24ely1h5scc7hzxrj20xp0ojtbc.jpg)