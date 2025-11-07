---
layout: post
title: Android Framework源码分析之View的绘制流程
tags: 源码分析 framework
categories: Android
date: 2019-06-13
---

通过分析Activity的启动过程，我们知道，View的绘制开始是从ViewRootImpl类中通过调用setView方法执行了requestLayout后开始的。

## requestLayout过程

```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	// Schedule the first layout -before- adding to the window
    // manager, to make sure we do the relayout before receiving
    // any other events from the system.
    requestLayout();
    if ((mWindowAttributes.inputFeatures
         & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
        mInputChannel = new InputChannel();
    }
    mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                                 & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
    try {
        mOrigWindowType = mWindowAttributes.type;
        mAttachInfo.mRecomputeGlobalAttributes = true;
        collectViewAttributes();
        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                                          getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                                          mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                                          mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
    } catch (RemoteException e) {
        mAdded = false;
        mView = null;
        mAttachInfo.mRootView = null;
        mInputChannel = null;
        mFallbackEventHandler.setView(null);
        unscheduleTraversals();
        setAccessibilityFocus(null, null);
        throw new RuntimeException("Adding window failed", e);
    } finally {
        if (restore) {
            attrs.restore();
        }
    }
 ...
}
```

requestLayout第一次调用是在setView方法中，从它开始执行View的测量、布局、绘制。

```java
// ViewRootImpl.java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

上面代码先是检查线程是否为合法线程，即是否为主线程，然后设置请求布局标示符设置为true，这个参数决定了后续是否执行measure和layout操作。最后执行scheduleTraversals()方法：

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

首先向主线程消息队列插入同步屏障消息，该方法发送了一个没有 target 的 Message 到 MessageQueue 中，在 next 方法中获取消息时，如果发现没有 target 的 Message，则在一定的时间内跳过同步消息，优先执行异步消息。这里通过调用此方法，保证 UI 绘制操作优先执行。

紧接着调用Choreographer的postCallback方法，发送一个Message到主线程消息队列：

```java
// Choreographer.java
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}
@TestApi
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
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {
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

最终通过Message的setAsynchronous设置为一个异步消息，然后发送到MessageQueue中去。

另外，mTraversalRunnable是TraversalRunnable类型对象，它实现了一个Runnable的接口，内部执行doTraversal()方法：

```java
// ViewRootImpl.java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }

        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

最终执行了perforTraversals()方法，而View的绘制也是从这个方法开始的：

```java
// ViewRootImpl.java

private void performTraversals() {
    ...
    if (layoutRequested) {
        ...
        // 执行perforMeasure进行测量工作
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);   
    }
    
    ...
        
    if (didLayout) {
        // 执行布局工作
        performLayout(lp, mWidth, mHeight);
    }
    ...
        
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

    if (!cancelDraw && !newSurface) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }
		// 执行绘制工作
        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }
    
}

```

首先看measureHierarchy方法：

```java
// ViewRootImpl.java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
	int childWidthMeasureSpec;
    int childHeightMeasureSpec;
    boolean windowSizeMayChange = false;
    ... 
    if (!goodMeasure) {
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        // 执行测量工作
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }
    ... 
}
```

在这个方法中，通过 getRootMeasureSpec 方法获取了根 View的MeasureSpec，实际上 MeasureSpec 中的宽高此处获取的值是 Window 的宽高。紧接着 调用performMeasure方法：

```java
// ViewRootImpl.java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

这个方法只是执行了mView的measure方法，这个mView就是DecorView，其中DecorView的measure方法中会调用View的measure 方法，measure() 这个方法是 final 的，所以 View 子类只能通过重载 onMeasure()来实现自己的测量逻辑。

```java
// View.java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int oWidth  = insets.left + insets.right;
        int oHeight = insets.top  + insets.bottom;
        widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
        heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
    }
    ...

    if (forceLayout || needsLayout) {
        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        resolveRtlPropertiesIfNeeded();

        int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            // 执行onMeasure方法
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        // flag not set, setMeasuredDimension() was not invoked, we raise
        // an exception to warn the developer
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("View with id " + getId() + ": "
                                            + getClass().getName() + "#onMeasure() did not set the"
                                            + " measured dimension by calling"
                                            + " setMeasuredDimension()");
        }

        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                      (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}
```

进而执行DecorView的onMeasure方法:

```java
// DecorView.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    final DisplayMetrics metrics = getContext().getResources().getDisplayMetrics();
    final boolean isPortrait =
        getResources().getConfiguration().orientation == ORIENTATION_PORTRAIT;

    final int widthMode = getMode(widthMeasureSpec);
    final int heightMode = getMode(heightMeasureSpec);

    ...
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    int width = getMeasuredWidth();
    boolean measure = false;

    widthMeasureSpec = MeasureSpec.makeMeasureSpec(width, EXACTLY);

    if (!fixedWidth && widthMode == AT_MOST) {
        final TypedValue tv = isPortrait ? mWindow.mMinWidthMinor : mWindow.mMinWidthMajor;
        if (tv.type != TypedValue.TYPE_NULL) {
            final int min;
            if (tv.type == TypedValue.TYPE_DIMENSION) {
                min = (int)tv.getDimension(metrics);
            } else if (tv.type == TypedValue.TYPE_FRACTION) {
                min = (int)tv.getFraction(mAvailableWidth, mAvailableWidth);
            } else {
                min = 0;
            }
            if (DEBUG_MEASURE) Log.d(mLogTag, "Adjust for min width: " + min + ", value::"
                                     + tv.coerceToString() + ", mAvailableWidth=" + mAvailableWidth);

            if (width < min) {
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(min, EXACTLY);
                measure = true;
            }
        }
    }

    // TODO: Support height?

    if (measure) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
}

```

又因为DecorView继承自FrameLayout，因此最终会执行FrameLayout的onMeasure方法，递归调用子 View 的 onMeasure 方法。

```java
// FrameLayout.java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    final boolean measureMatchParentChildren =
        MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
        MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            maxWidth = Math.max(maxWidth,
                                child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                                 child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                    lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }

    // Account for padding too
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // Check against our minimum height and width
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                         resolveSizeAndState(maxHeight, heightMeasureSpec,
                                             childState << MEASURED_HEIGHT_STATE_SHIFT));

    count = mMatchParentChildren.size();
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            final View child = mMatchParentChildren.get(i);
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

            final int childWidthMeasureSpec;
            if (lp.width == LayoutParams.MATCH_PARENT) {
                final int width = Math.max(0, getMeasuredWidth()
                                           - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                                           - lp.leftMargin - lp.rightMargin);
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                    width, MeasureSpec.EXACTLY);
            } else {
                childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                                                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                                                            lp.leftMargin + lp.rightMargin,
                                                            lp.width);
            }

            final int childHeightMeasureSpec;
            if (lp.height == LayoutParams.MATCH_PARENT) {
                final int height = Math.max(0, getMeasuredHeight()
                                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                                            - lp.topMargin - lp.bottomMargin);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    height, MeasureSpec.EXACTLY);
            } else {
                childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                                                             getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                                                             lp.topMargin + lp.bottomMargin,
                                                             lp.height);
            }

            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

performLayout与上面流程类似。

接着看performDraw方法：

```java
// ViewRootImpl.java
private void performDraw() {
	...
    try {
        boolean canUseAsync = draw(fullRedrawNeeded);
        if (usingAsyncReport && !canUseAsync) {
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
            usingAsyncReport = false;
        }
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    ...
        
}


private boolean draw(boolean fullRedrawNeeded) {
	Surface surface = mSurface;
    if (!surface.isValid()) {
        return false;
    }
    ...
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            ...
            // 开启硬件加速 ，启动硬件加速绘制
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);
        } else {
            ...
             
			// 使用软件绘制
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                              scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }
	}
}
```

ViewRootImpl 中有一个非常重要的对象 Surface，之所以说 ViewRootImpl 的一个核心功能就是负责 UI 渲染，原因就在于在 ViewRootImpl 中会将我们在 draw 方法中绘制的 UI 元素，绑定到这个 Surface 上。Surface 中的内容最终会被传递给底层的 SurfaceFlinger，最终将 Surface 中的内容进行合成并显示的屏幕上。

接下来看drawSoftware:

```java
// ViewRootImpl.java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
                             boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
    // Draw with software renderer.
    final Canvas canvas;

    ...

    canvas = mSurface.lockCanvas(dirty);
    ...
    try {
        ...
        mView.draw(canvas);
        ...
    } finally {
        surface.unlockCanvasAndPost(canvas);
    }
     
	...
}
```

上面代码中调用 DecorView 的 draw 方法将 UI 元素绘制到画布 Canvas 对象中，最后请求将 Canvas 中的内容显示到屏幕上，实际上就是将 Canvas 中的内容提交给 SurfaceFlinger 进行合成处理。

默认情况下软件绘制没有采用 GPU 渲染的方式，drawSoftware 工作完全由 CPU 来完成。因为DecorView 并没有复写 draw 方法，因此实际是调用的顶层 View 的 draw 方法：

```java
// View.java
@CallSuper
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
        (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

    // Step 1, draw the background, if needed
    int saveCount;

    if (!dirtyOpaque) {
        // 绘制View的背景
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        // 绘制View自身内容
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        // 进行事件分发，实际调用的是ViewGroup的方法，并递归调用子View的draw方法
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);

        if (debugDraw()) {
            debugDrawFocus(canvas);
        }

        // we're done...
        return;
    }
    ...
}
```

上面代码最终会调用ViewGroup的dispatchDraw方法，进而递归调用到子View的draw方法。

## 启用硬件加速

可以在 ViewRootImpl 的 draw 方法中，通过如下方法判断是否启用硬件加速：

```java
// ViewRootImpl.java
if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()){
    // 执行硬件加速绘制
}

```

我们可以在 AndroidManifest 清单文件中，指定 Application 或者某一个 Activity 支持硬件加速，如下：

```java
<application android:hardwareAccelerated="true">
	<activity ... />
    <activity android:hardwareAccelerated="false">
</application>
```

另外也可以更加细粒度的进行硬件加速配置

```java
// window级别
getWindow().setFlags(
	WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED,
    WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED
);

// view级别
View.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
```

之所以会有这么多级的支持区分，主要是因为并不是所有的 2D 绘制操作都支持硬件加速，当在自定义 View 中使用了如下 API，则有可能造成程序工作不正常：

```java
// Canvas

clipPath()
clipRegion()
drawPicture()
drawPosText()
drawTextOnPath()
drawVertices()

// Paint

setLinearText()
setMaskFilter()
setRasterizer()
```

## invalidate与postInvalidate

上面方法与requestLayout的区别是它们不一定会触发 View 的 measure 和 layout 的操作，多数情况下只会执行 draw 操作。

在View的measure方法中，可以看下：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
	...
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
    ...
    if (forceLayout || needsLayout) {
        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        resolveRtlPropertiesIfNeeded();

        int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        // flag not set, setMeasuredDimension() was not invoked, we raise
        // an exception to warn the developer
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("View with id " + getId() + ": "
                                            + getClass().getName() + "#onMeasure() did not set the"
                                            + " measured dimension by calling"
                                            + " setMeasuredDimension()");
        }

        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
	...
}

```

可以看出，如果要触发 onMeasure 方法，需要对 View 设置 PFLAG_FORCE_LAYOUT 标志位，而这个标志位在 requestLayout 方法中被设置，invalidate 并没有设置此标志位。

onLayout方法也是如此：

```java
// View.java
public void layout(int l, int t, int r, int b) {
    ...
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
    }
    ...
}

```

可以看出，当 View 的位置发送改变，或者添加 PFLAG_LAYOUT_REQUIRED 标志位后 onLayout 才会被执行。当调用 invalidate 方法时，如果 View 的位置并没有发生改变，则 View 不会触发重新布局的操作。

而postInvalidate与它类似，不过有一点区别： invalidate 是在 UI 线程调用，postInvalidate 是在非 UI 线程调用

```java
// View.java

public void postInvalidate() {
    postInvalidateDelayed(0);
}
public void postInvalidateDelayed(long delayMilliseconds) {
    // We try only with the AttachInfo because there's no point in invalidating
    // if we are not attached to our window
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    }
}
```

最终交给了ViewRootImpl来执行：

```java
// ViewRootImpl.java
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
    Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
    mHandler.sendMessageDelayed(msg, delayMilliseconds);
}

final class ViewRootHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_INVALIDATE:
                ((View) msg.obj).invalidate();
                break;
                ...
        }
    }
}
```

在非 UI 线程中，通过 Handler 发送了一个延时 Message，因为 Handler 是在主线程中创建的，所以最终 handlerMessage 会在主线程中被执行， msg.obj 就是发送 postInvalidate 的 View 对象，可以看出最终还是回到 UI 线程执行了 View 的 invalidate 方法。

## 子线程可以更新UI吗？ 

我们知道通常子线程不会更新UI，因为现在屏幕刷新率最低是 60Hz，意味着最多每 16ms 就会刷新一次屏幕，所以 UI 更新要尽可能快速，否则会丢帧导致卡顿。另外如果多个子线程更新，那么 UI 更新操作就需要加锁，频繁的加锁释放锁可能会延长 UI 渲染时间，但是不加锁如果允许子线程更新 UI 会导致多个线程对 UI 同时更新，造成线程不安全而导致 UI 最终效果无法想象，所以 Android 直接限制了子线程更新 UI，那么有没有办法可以更新UI呢？从源码中来看下：

```java
// ViewRootImpl.java

@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
            "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

上面是导致有时候子线程更新UI时异常崩溃的直接原因。

另外我们也看下View的requestLayout方法：

```java
public void requestLayout(){
    ...
    // 如果当前 View 存在 ViewParent，且 isLayoutRequested() 为 false 则调用 ViewParent 的 requestLayout()
   if (mParent != null && !mParent.isLayoutRequested()) {
      mParent.requestLayout();
   }
    ...
}
```

上面代码中View 的 `requestLayout()` 会调用其父布局的 `requestLayout()`，ViewGrop 并没有重写这个方法，所以还是调用的 View 的 `requestLayout()`，即一直递归到最上层。最上层的View就是DecorView，而DecorView其实也有mParent，即ViewRootImpl:

```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
      int userId) {
   synchronized (this) {
      if (mView == null) {
         ...
         view.assignParent(this);
         ...
      }
   }
}
```

这里的view即为参数传进来的DecorView，其调用了View的assignParent方法：

```java
// View.java
void assignParent(ViewParent parent) {
   if (mParent == null) {
      mParent = parent;
   } else if (parent == null) {
      mParent = null;
   } else {
      throw new RuntimeException("view " + this + " being added, but"
               + " it already has a parent");
   }
}
```

分析后总结：

1. 子线程更新 View 会调用 View的requestLayout()，然后开始递归查找父 View，找到了 Activity 的顶层 View 是 DecorView。
2. DecorView 的 ViewParent 是 ViewRootImpl，所以调用了 ViewRootImpl的requestLayout()，继而调用了 ViewRootImpl的checkThread()。
3. 由于ViewRootImpl 在主线程初始化，因此子线程调用检查线程会抛出异常。

所以这要让上述几个条件不成立就可以不让程序崩溃掉：

打破mParent == null或者mParent.isLayoutRequested() 为true

- 在DecorView和ViewRootImpl关联之前，更新View，比如onResume及以前更新View

  这时候mParent为null，所以条件不成立

- 在子线程更新 View 之前先 requestLayout

  ```java
  // View.java
  public boolean isLayoutRequested() {
     return (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
  }
  
  public void requetLayout(){
      ...
      mPrivateFlags |= PFLAG_FORCE_LAYOUT;
      mPrivateFlags |= PFLAG_INVALIDATED;
  
      ...
      if (mParent != null && !mParent.isLayoutRequested()) {
         mParent.requestLayout();
      }
      ...
  }
  
  public void layout(int l, int t, int r, int b) {
      ...
      mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
      ...
  }
  ```

  上面方法的结果与`mPrivateFlags` 是否存在 `PFLAG_FORCE_LAYOUT` 有关，而第一层请求requestLayout时就会在 `mPrivateFlags` 加入 `PFLAG_FORCE_LAYOUT`,在 `View的layout()` 里会去掉该标志，所以我们可以先在主线程调用requestLayout方法，然后马上调用子线程更新View就不会发生异常了。

- 子线程初始化ViewRootImpl

  这个比价好理解，让ViewRootImpl和更新View的线程在一个线程即可：

  ```java
  new Thread(new Runnable() {
      Looper.prepare()
      val imageView = ImageView(this)
      windowManager.addView(imageView, WindowManager.LayoutParams())
      // 更新View
      Looper.loop()
  })
  ```

- 在硬件加速的情况下只调用 View的invalidate()不会触发线程检查。

  ```java
  // View.java
  public void invalidate(boolean invalidateCache) {
     invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
  }
  
  void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
     ...
     if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
              || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
              || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
              || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        ...
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
              final Rect damage = ai.mTmpInvalRect;
              damage.set(l, t, r, b);
              p.invalidateChild(this, damage);
        }
     }
  }
  ```

  `p.invalidateChild(this, damage)` 表示使 ViewParent 重绘这个 View，所以看下这个方法：

  ```java
  // ViewGroup.java
  public final void invalidateChild(View child, final Rect dirty) {
     final AttachInfo attachInfo = mAttachInfo;
     if (attachInfo != null && attachInfo.mHardwareAccelerated) {
        // HW accelerated fast path
        onDescendantInvalidated(child, child);
        return;
     }
     ...
  }
  public void onDescendantInvalidated(@NonNull View child, @NonNull View target) {
     ...
     if (mParent != null) {
        mParent.onDescendantInvalidated(this, target);
     }
  }
  ```

  首先就会判断是否开启了硬件加速，如果开启了会进入onDescendantInvalidated这个方法：

  ```java
  // ViewRootImpl.java
  public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
     if ((descendant.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
        mIsAnimating = true;
     }
     invalidate();
  }
  
  @UnsupportedAppUsage
  void invalidate() {
     mDirty.set(0, 0, mWidth, mHeight);
     if (!mWillDrawSoon) {
        scheduleTraversals();
     }
  }
  
  ```

  最终调用了ViewRootImpl的onDescendantInvalidated方法，这里执行了scheduleTraversals方法但是没有checkThread。

- 如果不修改 Drawable 的固有宽高不变就不会调用 `requestLayout()`

  `mDrawableWidth` 和 `mDrawableHeight` 的改变在 `updateDrawable()`中。

- TextView 在固定尺寸下更新文本，它的setText方法会调用checkForRelayout()方法：

  ```java
  // TextView.java
  private void checkForRelayout() {
     if ((mLayoutParams.width != LayoutParams.WRAP_CONTENT
           || (mMaxWidthMode == mMinWidthMode && mMaxWidth == mMinWidth))
           && (mHint == null || mHintLayout != null)
           && (mRight - mLeft - getCompoundPaddingLeft() - getCompoundPaddingRight() > 0)) {
        // 上述三个条件为：
        // TextView 的宽度是固定的
        // 没有设置提示文本，或者提示文本已经被渲染完成
        // TextView 的宽度大于 0
  
        int oldht = mLayout.getHeight();
        int want = mLayout.getWidth();
        int hintWant = mHintLayout == null ? 0 : mHintLayout.getWidth();
  
        makeNewLayout(want, hintWant, UNKNOWN_BORING, UNKNOWN_BORING,
                 mRight - mLeft - getCompoundPaddingLeft() - getCompoundPaddingRight(),
                 false);
  
        if (mEllipsize != TextUtils.TruncateAt.MARQUEE) {
           // 不是跑马灯模式
           if (mLayoutParams.height != LayoutParams.WRAP_CONTENT
                 && mLayoutParams.height != LayoutParams.MATCH_PARENT) {
              // TextView 的高度是固定的
              autoSizeText();
              invalidate();
              return;
           }
  
           if (mLayout.getHeight() == oldht
                 && (mHintLayout == null || mHintLayout.getHeight() == oldht)) {
              // 没有改变高度
              autoSizeText();
              invalidate();
              return;
           }
        }
  
        requestLayout();
        invalidate();
     } else {
        nullLayouts();
        requestLayout();
        invalidate();
     }
  }
  ```

  可以看到满足源码中注释的条件就不会触发 `View#requestLayout()`。

- SurfaceView使用自带 Surface 去做画面渲染 ， TextureView可以通过 `lockCanvas()`使用临时的 Surface，所以都不会触发 View的requestLayout()方法。

上面分享只是为了熟悉底层原理，了解了 Activity View 树的构建流程、更新 UI 的基础流程。但是根据 Android 的设计理念，还是不应使用在子线程中更新 UI，以免造成未知的异常出现。