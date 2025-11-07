---
layout: post
title: Android Framework源码分析之屏幕刷新机制以及Choreographer
tags: 源码分析 framework
categories: Android
date: 2019-06-18
---

生活中，我们经常遇到别人说手机画面卡，这里的卡对应我们开发者来说，表示的就是掉帧（**jank**）或者画面撕裂（**tearing**），我们先来说说一些概念：

- 帧（Frame）：动画中的单幅画面，相当于电影胶片中的一个镜头，一帧就是一幅静止的画面，连续帧动起来就是我们看到的动画。

- 帧率（Frame Rate）：每秒传输的图片画面的帧数，也可以理解为画面每秒钟刷新几次，通常用FPS（Frames Per Second）表示

- 刷新率（Refresh Rate）: 每秒屏幕刷新次数，这取决去硬件固定参数，常用Hz表示。

- 掉帧（jank）: 一个帧在屏幕上连续出现2次，看起来就像画面静止了一样。

- 撕裂（tearing）：一个画面的数据来自2个不同的帧，导致画面出现错误。

  ![image-20220901165727092](https://tva1.sinaimg.cn/large/e6c9d24ely1h5r7jia213j20i1081aag.jpg)

我们都知道一个图形显示系统，往往涉及的这三部分：CPU、GPU、屏幕Display。它们的作用分别是：

- CPU擅长复杂的逻辑运算，用作计算数据和信息处理，然后交给GPU
- GPU擅长数学计算，用作图形处理绘制，然后存储到缓存区
- 屏幕Display用于显示从缓存区读取出来的数据

每一帧都重复上面的这个过程，然后1秒钟如果重复60次，每个过程所花费的时间就是16.6ms，也就是Android中常说的每隔16.6ms刷新一次屏幕。

首先我们看看画面撕裂原因，屏幕显示画面并不是一下子把整个图像完整的显示出来，这里有一个词叫逐行扫描，就是一行一行显示，只不过显示过快，我们肉眼看不出来而已。前面说如果屏幕Display要从缓存区buffer取的数据在屏幕逐行读取数据过程中发生变化了，就有可能读取到下一帧的画面数据，导致出现错误。**这种情况一般是因为显卡输出帧的速率高于屏幕显示器的刷新速率**导致显示器并不能及时处理输出的帧，而最终出现了多个帧的画面都留在了显示器上的问题

同样，如果掉帧， 即图像绘制的速度低于屏幕显示器的刷新速率，就会出现卡顿，即下一帧要显示了，可是图像数据还没准备好，就只能仍然显示上一帧的数据。

## 垂直同步信号Vsync

一般为了解决这个问题，显示系统通常会引入垂直同步信号（Vsync）的概念：通俗点讲就是讲GPU的速率限制和屏幕Display的FPS一样。

系统会在屏幕Display绘制完一帧之后发送一个垂直同步信号（Vsync），然后CPU和GPU就准备下一帧的内容，等待屏幕Display下一帧绘制完，又会发送一个垂直同步信号（Vsync）。如此反复，这样就保证了GPU的工作效率和屏幕Display保持同步，我们先来看下没有Vsync的情况：

![image-20220901171603558](https://tva1.sinaimg.cn/large/e6c9d24ely1h5r82tyj17j20mi0abt99.jpg)

这个图中有三个元素，Display 是显示屏幕，GPU 和 CPU 负责渲染帧数据，每个帧以方框表示，并以数字进行编号，如0、1、2等等

- Display显示第0帧数据，此时CPU和GPU渲染第1帧画面，而且赶在Display显示下一帧前完成。
- 因为渲染及时，Display在第0帧显示完成后，也就是第1个VSync后，正常显示第1帧。
- 由于某些原因，比如CPU资源被占用，系统没有及时地开始处理第2帧，直到第2个VSync快来前才开始处理
- 第2个VSync来时，由于第2帧数据还没有准备就绪，显示的还是第1帧。
- 当第2帧数据准备完成后，它并不会马上被显示，而是要等待下一个VSync的到来。

我们可以从图中看出，是因为新的一帧开始的时候，CPU 在处理其他任务，并没有马上执行下一帧的任务，而等第2帧的数据准备好后，并不能马上显示需要等第3个Vsync信号到来才可以显示，这样第2个Vsync到来只能仍然显示第1帧的画面。所以我们看下引入Vsync后的情况：

![image-20220901172355096](https://tva1.sinaimg.cn/large/e6c9d24ely1h5r8b00o8ij20oc0b2gm4.jpg)

引入了Vsync后，一旦VSync到来，立刻就开始执行下一帧的绘制工作，这样就可以大大降低Jank出现的概率。另外，Vsync引入后，要求绘制也只能在收到Vsync消息之后才能进行，因此，也就杜绝了另外一种极端情况的出现:CPU（GPU）一直不停的进行绘制，帧的生成速度高于屏幕的刷新速度，导致生成的帧不能被显示，只能丢弃，这样就出现了丢帧的情况，这样引入Vsync后，绘制的速度就和屏幕刷新的速度保持一致了。

我们现在Android设备主流的刷新率还在60Hz，即每一帧最多需要1/60=16.6ms要准备完成。如果CPU/GPU性能差一些，达不到这个要求会出现什么情况？

## 双缓冲

主要是两个缓冲区组成，缓存区`backBuffer`用于CPU/GPU图形处理 ，缓存区`frameBuffer`用于显示器显示，这样分工明确之后，屏幕只会读取`framebuffer`的内容，是一帧完整的画面。而CPU/GPU计算的新一帧内容会放到backbuffer中，不会影响到`framebuffer`的内容。

只有当屏幕绘制完一帧内容之后，才会将CPU/GPU计算好的新一帧内容也就是`backbuffer`内容和`framebuffer`进行交换，这一过程也叫Swap。所以当屏幕显示完第1帧画面后，系统发生Vsync信号，接着三个步骤：

1. 交换两个缓存区（framebuffer、backbuffer）数据内容。
2. 显示器开始显示第2帧内容，也就是交换后的framebuffer内容。
3. CPU/GPU开始计算处理第三帧的内容，并在处理好内容后放到backbuffer中。

下面A和B分别代表两个缓冲区，它们不断地交换来正确显示画面。

![image-20220901173223726](https://tva1.sinaimg.cn/large/e6c9d24ely1h5r8jtykssj20o60am74v.jpg)

而这时问题其实还没有解决，正如前面所说，如果 CPU/GPU性能不好，其帧率小于 Display 的帧率，就会出现下面的情况：

![image-20220901173521800](https://tva1.sinaimg.cn/large/e6c9d24ely1h5r8mwrzfoj20n00afgmc.jpg)

当CPU/GPU的处理时间超过16.6ms时，第一个VSync到来时，缓冲区B中的数据还没有准备好，于是只能继续显示之前A缓冲区中的内容。而B完成后，又因为这时候没有VSync信号到来，所以它只能等待下一个信号的来临。于是在这一过程中，有一大段时间是被浪费的。当下一个VSync出现时，CPU/GPU马上执行操作，此时它可操作的buffer是A，相应的显示屏对应的就是B。这时看起来就是正常的。只不过由于执行时间仍然超过16ms，导致下一次应该执行的缓冲区交换又被推迟了——如此循环反复，便出现了越来越多的“Jank”，即我们所说的卡顿。很显然，第一次的Jank看起来是没有办法的，除非升级硬件配置来加快FPS。但是后面其他的Jank，我们其实有办法可以减少的，仔细分析原因，会发现主要是由于当前已经没有可用的buffer了，因为 A Buffer 被 Display 在使用。B Buffer 被 GPU 在使用，而一旦过了 VSync 时间点，CPU 就不能被触发以处理绘制工作了，其实主要就是提高CPU/GPU的利用率。

为了解决这个问题，Android引入了Triple Buffer 机制。

## Triple Buffer

它主要由3个缓冲区组成，除了前面介绍的2个缓冲区外，又增加了一个Triple Buffer，即：

1、缓存区`backBuffer`用于CPU/GPU图形处理 

2、缓存区`TripleBuffer`用于CPU/GPU图形处理 

3、缓存区`frameBuffer`用于屏幕Display显示

前面说的由于在第2个VSync信号来的时候，`backBuffer`被GPU占用，导致CPU无法去开始新一帧的计算，加入了第3个缓冲区，那么在第2个VSync信号来的时候，CPU就可以利用`TripleBuffer`开始新一帧的计算，而无视正在被GPU占用的`backBuffer`。

而三缓存和Vsync垂直同步信号都是Android4.1以后引入的，该项目也被称为黄油计划（Project Butter）

![image-20220901174342434](https://tva1.sinaimg.cn/large/e6c9d24ely1h5r8vloa5zj20n20aodgn.jpg)

## Choreographer（编舞者）

在Android中View 的绘制会经过 Measure、Layout、Draw 这 3 个阶段，而这 3 个阶段的工作都是由 CPU 来负责完成，另外CPU也会负责一些用户输入、动画等事件，这些都是在UI线程完成的。当CPU计算好数据后，会在 RenderThread 线程中将这部分数据提交给 GPU并被缓存到Back Buffer中。GPU 负责对这些数据进行栅格化（Rasterization）操作，完成之后将其交换（swap）到 Frame Buffer 中，最后屏幕从 Frame Buffer 中读取数据显示，实际上真正对 Buffer 中的数据进行合成显示到屏幕上的是 SurfaceFlinger。		

Choreographer 的引入，主要是配合 Vsync 为上层消息处理有一个稳定的时机。Vsync 信号到来唤醒 Choreographer 来做 App 的绘制操作 ，如果每个 Vsync 周期应用都能渲染完成，那么画面就会感觉到很流畅。所以它在这里扮演着调度者的角色，一方面负责接收和处理 App 的各种更新消息和回调，等到 Vsync 到来的时候统一处理。比如集中处理 Input(主要是 Input 事件的处理) 、Animation(动画相关)、Traversal(包括 measure、layout、draw 等操作)，另一方面负责请求和接收 Vsync 信号。接收 Vsync 事件回调(通过 FrameDisplayEventReceiver.onVsync )；请求 Vsync(FrameDisplayEventReceiver.scheduleVsync)。

我们先来看下View的invalidate过程，最终都会走到ViewRootImpl的scheduleTraversals方法中：

```java
// ViewRootImpl.java
@UnsupportedAppUsage
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
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

在这个方法里我们可以看到mChoreographer.postCallback 方法，这个方法将 TraversalRunnable 以参数的形式传递给 Choreographer

```java
// Choreographer.java
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}
public void postCallbackDelayed(int callbackType,
                                Runnable action, Object token, long delayMillis) {
    if (action == null) {
        throw new IllegalArgumentException("action must not be null");
    }
    if (callbackType < 0 || callbackType > CALLBACK_LAST) {
        throw new IllegalArgumentException("callbackType is invalid");
    }

    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
                                         Object action, Object token, long delayMillis) {
    if (DEBUG_FRAMES) {
        Log.d(TAG, "PostCallback: type=" + callbackType
              + ", action=" + action + ", token=" + token
              + ", delayMillis=" + delayMillis);
    }

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        // 将Runnable加入到队列中
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
            // 订阅信号
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

经过一系列调用将传入的 TraversalRunnable 插入到一个 CallbackQueue 中。后续当 Choreographer 接收到 vsync 信号时，会将此 TraversalRunnable 从队列中取出执行

另外接着看下sheduleFrameLocked方法：

```java
// Choreographer.java
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            if (DEBUG_FRAMES) {
                Log.d(TAG, "Scheduling next frame on vsync.");
            }

            // If running on the Looper thread, then schedule the vsync immediately,
            // otherwise post a message to schedule the vsync from the UI thread
            // as soon as possible.
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            final long nextFrameTime = Math.max(
                mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
            if (DEBUG_FRAMES) {
                Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
            }
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}

private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}

private final class FrameHandler extends Handler {
    public FrameHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME:
                doFrame(System.nanoTime(), 0);
                break;
            case MSG_DO_SCHEDULE_VSYNC:
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK:
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}

void doScheduleVsync() {
    synchronized (mLock) {
        if (mFrameScheduled) {
            scheduleVsyncLocked();
        }
    }
}
```

上面总会执行到scheduleVsyncLocked方法，它会执行FrameDisplayEventReceiver的scheduleVsync方法，FrameDisplayEventReceiver是Choreographer的内部类，继承于DisplayEventReceiver这个抽象类，  即会调用DisplayEventReceiver的scheduleVsync方法：

```java
// DisplayEventReceiver.java
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
              + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

该方法会调用一个本地方法nativeScheduleVsync来向系统订阅Vsync信号，Android 系统每过 16.6ms 会发送一个 vsync 信号。但这个信号并不是所有 App 都能收到的，只有订阅的才能收到。这样设计的合理之处在于，当 UI 没有变化的时候就不会去调用 nativeScheduleVsync 去订阅，也就不会收到 vsync 信号，减少了不必要的绘制操作。需要注意的是每次订阅只能收到一次 vsync 信号，如果需要收到下次信号需要重新订阅。比如 Animation 的实现就是在订阅一次信号之后，紧接着再次调用 nativeScheduleVsync 方法订阅下次 vsync 信号，因此会不断地刷新 UI。

而Vsync信号的接收，其实也是由FrameDisplayEventReceiver来完成的，它回调了onVsync方法：

```java
// Choreographer#FrameDisplayEventReceiver.java

@Override
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
	...

    // Post the vsync event to the Handler.
    // The idea is to prevent incoming vsync events from completely starving
    // the message queue.  If there are no messages in the queue with timestamps
    // earlier than the frame time, then the vsync event will be processed immediately.
    // Otherwise, messages that predate the vsync event will be handled first.
    long now = System.nanoTime();
    if (timestampNanos > now) {
        Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
              + " ms in the future!  Check that graphics HAL is generating vsync "
              + "timestamps using the correct timebase.");
        timestampNanos = now;
    }

    if (mHavePendingVsync) {
        Log.w(TAG, "Already have a pending vsync event.  There should only be "
              + "one at a time.");
    } else {
        mHavePendingVsync = true;
    }

    mTimestampNanos = timestampNanos;
    mFrame = frame;
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}
```

该方法会将FrameDisplayEventReceiver自身通过异步消息的方式发送到MessageQueue，所以查看其run方法：

```java
@Override
public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame);
}
```

最终调用了Choreographer的doFrame方法，其实也就是在这个方法中将之前插入到 CallbackQueue 中的 Runnable 取出来执行：

```java
// Choreographer.java

void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        ...
         
		// 掉帧逻辑计算
        long intendedFrameTimeNanos = frameTimeNanos;
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                      + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            if (DEBUG_JANK) {
                Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                      + "which is more than the frame interval of "
                      + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                      + "Skipping " + skippedFrames + " frames and setting frame "
                      + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
            }
            frameTimeNanos = startNanos - lastFrameOffset;
        }

        if (frameTimeNanos < mLastFrameTimeNanos) {
            if (DEBUG_JANK) {
                Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                      + "previously skipped frame.  Waiting for next vsync.");
            }
            scheduleVsyncLocked();
            return;
        }

        if (mFPSDivisor > 1) {
            long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
            if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                scheduleVsyncLocked();
                return;
            }
        }

        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
        mFrameScheduled = false;
        mLastFrameTimeNanos = frameTimeNanos;
    }

    try {
        // 执行这一帧做的事情
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

    if (DEBUG_FRAMES) {
        final long endNanos = System.nanoTime();
        Log.d(TAG, "Frame " + frame + ": Finished, took "
              + (endNanos - startNanos) * 0.000001f + " ms, latency "
              + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
    }
}
```

上面主要完成2部分工作，第一部分是计算掉帧的逻辑，第二部分就是执行我们一帧完成的工作，主要就是前面说的Input(主要是 Input 事件的处理) 、Animation(动画相关)、Traversal(包括 measure、layout、draw 等操作)等工作。

我们看下doCallbacks方法：

```java
// Choreographer.java
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        // We use "now" to determine when callbacks become due because it's possible
        // for earlier processing phases in a frame to post callbacks that should run
        // in a following phase, such as an input event that causes an animation to start.
        final long now = System.nanoTime();
        // 从队列中取出来Runnable
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
            now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;
        }
        mCallbacksRunning = true;

        ...
    }
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            if (DEBUG_FRAMES) {
                Log.d(TAG, "RunCallback: type=" + callbackType
                      + ", action=" + c.action + ", token=" + c.token
                      + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
            }
            // 开始执行Runnable
            c.run(frameTimeNanos);
        }
    } finally {
        synchronized (mLock) {
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}

```

从上面代码也可以看出前面Choreographer保存到队列中的Runnable，从这里取出来然后被立即执行。

综上，Choreographer起着承上启下的作用：

- 承上：接收应用层的各种 callback 输入，包括 input、animation、traversal 绘制。但是这些 callback 并不会被立即执行。而是会缓存在 Choreographer 中的 CallbackQueue 中。

- 启下：内部的 FrameDisplayEventReceiver 负责接收硬件层发送的 vsync 信号。当接收到 vsync 信号之后，会调用 onVsync 方法，进而调用 doFrame方法，最终到 doCallbacks，在 doCallbacks方法 中从 CallbackQueue 中取出先前保存的Runnable，并调用其 run 方法开始执行工作。

另外Choreographer也向外部提供一个 FrameCallback 接口，来监听 doFrame 方法的执行过程。

```java
// Choreographer.java
public interface FrameCallback {
    public void doFrame(long frameTimeNanos);
}
```

这个接口中的 doFrame 方法会在绘制每一帧时被调用，所以我们可以在 App 层主动向 Choreographer 中添加 Callback，然后通过检测两次 doFrame 方法执行的时间间隔来判断是否发生“丢帧”。

```java
Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
    long lastFrameTimeNanos = 0;
    long currentFrameTimeNanos = 0;

    @Override
    public void doFrame(long frameTimeNanos) {
        Log.i("DemoActivity", "DoFrame");
        if(lastFrameTimeNanos == 0) {
            lastFrameTimeNanos = frameTimeNanos;
        }

        currentFrameTimeNanos = frameTimeNanos;
        long diffMs = TimeUnit.MICROSECONDS.convert(currentFrameTimeNanos - lastFrameTimeNanos, TimeUnit.NANOSECONDS);
        if(diffMs > 16.6f){
            long droppedFrameCount = (long) (diffMs / 16.6f);
            Log.w("DemoActivity", "发生丢帧");
        }
        Choreographer.getInstance().postFrameCallback(this);
    }
});
```

这里每一次订阅都只会接收一次 vsync 信号，而我们需要一直监听 doFrame 的回调，因此在方法最后需要递归的执行 postFrameCallback 方法。事实上，市面上很多APM工具都利用了这一特性。

另外还有细节需要注意的就是同步屏障消息。

我们在前面的scheduleTraversals 方法

```java
// ViewRootImpl.java
@UnsupportedAppUsage
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
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

可以看到在插入Callback之前执行了postSyncBarrier方法来设置同步屏障。Android中Message类型分为2类，同步消息和异步消息 ，默认情况是同步消息，只有在创建Handler时传入了第二个参数为true时，该handler发送的消息才是异步消息：

```java
// Handler.java
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
            (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                  klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
            + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

```

还有一种是通过Message方法来设置：

```java
// Message.java
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

一般情况下，Looper 从 MessageQueue 取消息时对这两者并不会做区别对待。但是如果 MessageQueue 设置了同步屏障就会出现差异。
当 MessageQueue 设置同步屏障之后，在 next 方法获取消息时会忽略所有的同步消息，只取异步消息，也就是说异步消息在此时的优先级更高。而 TraversalRunnable 会被封装到一个异步的 Message 中，因此 View 绘制的一系列操作会被优先执行，这也是提高渲染性能，优化的一种手段。

```java
// MessageQueue.java
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
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
            ...
        }
        ...
    }
}

```

可以看到上面do while循环外面的条件，如果是异步消息就结束循环，取出该Message先处理。

