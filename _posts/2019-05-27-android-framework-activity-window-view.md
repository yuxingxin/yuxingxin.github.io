---
layout: post
title: Android Framework源码分析之Activity、Window、View之间的关系
tags: 源码分析 framework
categories: Android
date: 2019-05-27
---

每当我们显示一个界面的时候，都是通过start一个Activity的方式，对于显示布局内容，也只是通过在onCreate方法里面setContentView就可以，剩下的都是Activity帮我们做了，我们自始至终也没有创建过window或者view，那么这背后都发生了什么 ？这篇文章梳理一下这三者的关系

## setContentView

我们从setContentView源码开始：

```java
// Activity.java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

这里直接把操作交给了window来处理，getWindow返回的是Activity中的全局变量mWindow，它是Window类型，在分析startActivity过程时，还记得最后会调用到 ActivityThread 中的 performLaunchActivity 方法。通过反射创建Activity对象，并执行其attach方法，mWindow就是在这个方法中被创建：

```java
// ActivityThread.java    
@UnsupportedAppUsage
final void attach(Context context, ActivityThread aThread,
                  Instrumentation instr, IBinder token, int ident,
                  Application application, Intent intent, ActivityInfo info,
                  CharSequence title, Activity parent, String id,
                  NonConfigurationInstances lastNonConfigurationInstances,
                  Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                  Window window, ActivityConfigCallback activityConfigCallback) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);
	// 创建window对象
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    // 设置callback
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token;
    mIdent = ident;
    mApplication = application;
    mIntent = intent;
    mReferrer = referrer;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    mTitle = title;
    mParent = parent;
    mEmbeddedID = id;
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
    if (voiceInteractor != null) {
        if (lastNonConfigurationInstances != null) {
            mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
        } else {
            mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                                                   Looper.myLooper());
        }
    }
	// 绑定windowManager引用
    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;

    mWindow.setColorMode(info.colorMode);

    setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
}
```

从上面代码中，我们可以看出mWindow被赋值为一个PhoneWindow对象，实际上它也是整个Android系统中Window的一个唯一的实现类。

紧接着调用setWindowManager方法，将WindowManager传给PhoneWindow：

```java
// PhoneWindow.java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                             boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
        || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

最终，交给了WindowManagerImpl来创建一个实例对象，即PhoneWindow持有了WindowManagerImpl的引用。

紧接着上面看PhoneWindow的setContentView方法：

```java
// PhoneWindow.java

@Override
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                                                       getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

上面代码中档mContentParent为null时，会执行installDecor方法来初始化DecorView和mContentParent：

```java
// PhoneWindow.java

    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            // 创建DecorView
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            // 创建FrameLayout布局
            mContentParent = generateLayout(mDecor);

            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);

            if (decorContentParent != null) {
                mDecorContentParent = decorContentParent;
                mDecorContentParent.setWindowCallback(getCallback());
                if (mDecorContentParent.getTitle() == null) {
                    mDecorContentParent.setWindowTitle(mTitle);
                }

                final int localFeatures = getLocalFeatures();
                for (int i = 0; i < FEATURE_MAX; i++) {
                    if ((localFeatures & (1 << i)) != 0) {
                        mDecorContentParent.initFeature(i);
                    }
                }

                mDecorContentParent.setUiOptions(mUiOptions);

                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
                        (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
                    mDecorContentParent.setIcon(mIconRes);
                } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
                        mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                    mDecorContentParent.setIcon(
                            getContext().getPackageManager().getDefaultActivityIcon());
                    mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
                }
                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
                        (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
                    mDecorContentParent.setLogo(mLogoRes);
                }

                // Invalidate if the panel menu hasn't been created before this.
                // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
                // being called in the middle of onCreate or similar.
                // A pending invalidation will typically be resolved before the posted message
                // would run normally in order to satisfy instance state restoration.
                PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
                if (!isDestroyed() && (st == null || st.menu == null) && !mIsStartingWindow) {
                    invalidatePanelMenu(FEATURE_ACTION_BAR);
                }
            } else {
                mTitleView = findViewById(R.id.title);
                if (mTitleView != null) {
                    if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                        final View titleContainer = findViewById(R.id.title_container);
                        if (titleContainer != null) {
                            titleContainer.setVisibility(View.GONE);
                        } else {
                            mTitleView.setVisibility(View.GONE);
                        }
                        mContentParent.setForeground(null);
                    } else {
                        mTitleView.setText(mTitle);
                    }
                }
            }

            if (mDecor.getBackground() == null && mBackgroundFallbackResource != 0) {
                mDecor.setBackgroundFallback(mBackgroundFallbackResource);
            }

            ...
        }
    }

```

可以看到上面代码通过generateDecor方法和generateLayout方法分别创建出了DecorView和FrameLayout布局:

```java
// PhoneWindow.java
protected DecorView generateDecor(int featureId) {
    // System process doesn't have application context and in that case we need to directly use
    // the context we have. Otherwise we want the application context, so we don't cling to the
    // activity.
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, getContext());
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}

protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.

        TypedArray a = getWindowStyle();

        ...

        WindowManager.LayoutParams params = getAttributes();

        ... 

        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();
        ...

        mDecor.startChanging();
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

        // 通过findViewById创建FrameLayout布局
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
            ProgressBar progress = getCircularProgressBar(false);
            if (progress != null) {
                progress.setIndeterminate(true);
            }
        }

        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            registerSwipeCallbacks(contentParent);
        }

        // Remaining setup -- of background and title -- that only applies
        // to top-level windows.
        if (getContainer() == null) {
            final Drawable background;
            if (mBackgroundResource != 0) {
                background = getContext().getDrawable(mBackgroundResource);
            } else {
                background = mBackgroundDrawable;
            }
            mDecor.setWindowBackground(background);

            final Drawable frame;
            if (mFrameResource != 0) {
                frame = getContext().getDrawable(mFrameResource);
            } else {
                frame = null;
            }
            mDecor.setWindowFrame(frame);

            mDecor.setElevation(mElevation);
            mDecor.setClipToOutline(mClipToOutline);

            if (mTitle != null) {
                setTitle(mTitle);
            }

            if (mTitleColor == 0) {
                mTitleColor = mTextColor;
            }
            setTitleColor(mTitleColor);
        }

        mDecor.finishChanging();

        return contentParent;
    }

```

其中mContentParent就是通过findViewById被创建出来，就是com.android.internal.R.id.content这个FrameLayout。

紧接着PhoneWindow的setContentView方法往下看，通过mLayoutInflater.inflate(layoutResID, mContentParent)方法将我们自己创建的布局添加到mContentParent布局上。最后，它们的关系如下图：

![image-20220831100418935](https://tva1.sinaimg.cn/large/e6c9d24ely1h5ppzc7y1zj20dz0jo752.jpg)

## onResume

上面只是创建了 PhoneWindow，和DecorView，但目前二者真正产生关系是在ActivityThread.performResumeActivity 中，再调用 r.activity.performResume()，调用 r.activity.makeVisible，将 DecorView 添加到当前的 Window 上。

```java
// ActivityThread.java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                                 String reason) {
	... 
     if (a.mVisibleFromClient) {
         if (!a.mWindowAdded) {
             a.mWindowAdded = true;
             wm.addView(decor, l);
         } else {
             // The activity will get a callback for this {@link LayoutParams} change
             // earlier. However, at that time the decor will not be set (this is set
             // in this method), so no action will be taken. This call ensures the
             // callback occurs with the decor set.
             a.onWindowAttributesChanged(l);
         }
     }
    
    ...
    if (r.activity.mVisibleFromClient) {
        // 添加视图
        r.activity.makeVisible();
    }

}

```

在Activity的makeVisible方法中，我们调用了windowManager的addView方法：

```java
// Activity.java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

WindowManager 的 addView 的具体实现在 WindowManagerImpl 中，而 WindowManagerImpl 的 addView 又会调用 WindowManagerGlobal.addView()，其中WindowManagerGlobal是WindowManagerImpl的一个单例对象。

```java
public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow) {
    ...

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
            && (context.getApplicationInfo().flags
                & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        ...

        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

这个过程创建一个 ViewRootImpl对象，并通过setView方法将之前创建的 DecorView 作为参数传入，以后 DecorView 的事件都由 ViewRootImpl 来管理了，比如，DecorView 上添加和删除View。ViewRootImpl 实现了 ViewParent 这个接口，这个接口最常见的一个方法是 requestLayout()。

ViewRootImpl 是个 ViewParent，在 DecoView 添加View 时，就会将 View 中的 ViewParent 设为 DecoView 所在的 ViewRootImpl，View 的 ViewParent 相同时，理解为这些 View 在一个 View 链上。所以每当调用 View 的 requestLayout()时，其实是调用到 ViewRootImpl，ViewRootImpl 会控制整个事件的流程。可以看出一个 ViewRootImpl 对添加到 DecoView 的所有 View 进行事件管理。

```java
// ViewRootImpl.java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    
    // 获取WindowSession的代理类
    mWindowSession = WindowManagerGlobal.getWindowSession();
    mDisplay = display;
    mBasePackageName = context.getBasePackageName();
    mThread = Thread.currentThread();
    ...
    mWindow = new W(this);
    ...
    mViewVisibility = View.GONE;
    mFirst = true; // true for the first time the view is added
    mAdded = false;
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
                                      context);
    ...
    // 初始化Choreographer对象
    mChoreographer = Choreographer.getInstance();
    mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);

    if (!sCompatibilityDone) {
        sAlwaysAssignFocus = mTargetSdkVersion < Build.VERSION_CODES.P;

        sCompatibilityDone = true;
    }

    loadSystemProperties();
}


public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

            ...
            mAdded = true;
            int res; /* = WindowManagerImpl.ADD_OKAY; */

            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            // View的绘制流程
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
                // binder调用，实际上是System的Session类
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

            

            if (view instanceof RootViewSurfaceTaker) {
                mInputQueueCallback =
                    ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
            }
            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                                                                   Looper.myLooper());
            }

            view.assignParent(this);
            mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
            mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

            if (mAccessibilityManager.isEnabled()) {
                mAccessibilityInteractionConnectionManager.ensureConnection();
            }

            if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
            }

            // Set up the input pipeline.
            CharSequence counterSuffix = attrs.getTitle();
            mSyntheticInputStage = new SyntheticInputStage();
            InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
            InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                                                                        "aq:native-post-ime:" + counterSuffix);
            InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
            InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                                                    "aq:ime:" + counterSuffix);
            InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
            InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                                                                      "aq:native-pre-ime:" + counterSuffix);

            mFirstInputStage = nativePreImeStage;
            mFirstPostImeInputStage = earlyPostImeStage;
            mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
        }
    }
}
```

上面的mWindowSession类是通过WindowManagerGlobal.getWindowSession()方法创建的，mWindowSession 实际上是 IWindowSession 类型，是一个 Binder 类型，真正的实现类是 System 进程中的 Session，可从下面分析看出：

```java
// WindowManagerGlobal.java
@UnsupportedAppUsage
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                IWindowManager windowManager = getWindowManagerService();
                sWindowSession = windowManager.openSession(
                    new IWindowSessionCallback.Stub() {
                        @Override
                        public void onAnimatorScaleChanged(float scale) {
                            ValueAnimator.setDurationScale(scale);
                        }
                    },
                    imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}

@UnsupportedAppUsage
public static IWindowManager getWindowManagerService() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowManagerService == null) {
            sWindowManagerService = IWindowManager.Stub.asInterface(
                ServiceManager.getService("window"));
            try {
                if (sWindowManagerService != null) {
                    ValueAnimator.setDurationScale(
                        sWindowManagerService.getCurrentAnimatorScale());
                }
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowMannagerService;
    }
}
```

上面IWindowManager是WMS的代理对象，openSession其实现如下：

```java
// WindowManagerService.java
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
                                  IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
    return session;
}
```

由此可见，IWindowSession其实就是Session类，然后调用它的addToDisplay方法：

```java
// Session.java
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
                        int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
                        Rect outStableInsets, Rect outOutsets,
                        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                              outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel);
}
```

方法中的mService 就是 WMS，该方法直接调用了WMS的addWindow方法。至此，Window 已经成功的被传递给了 WMS。

## 事件传递

再回看ViewRootImpl的setView方法：

```java
// ViewRootImpl.java


public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

            ...
            mAdded = true;
            int res; /* = WindowManagerImpl.ADD_OKAY; */

            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            // View的绘制流程
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
                // binder调用，实际上是System的Session类
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

            

            if (view instanceof RootViewSurfaceTaker) {
                mInputQueueCallback =
                    ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
            }
            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                                                                   Looper.myLooper());
            }

            view.assignParent(this);
            mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
            mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

            if (mAccessibilityManager.isEnabled()) {
                mAccessibilityInteractionConnectionManager.ensureConnection();
            }

            if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
            }

            // Set up the input pipeline.
            CharSequence counterSuffix = attrs.getTitle();
            mSyntheticInputStage = new SyntheticInputStage();
            InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
            InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                                                                        "aq:native-post-ime:" + counterSuffix);
            InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
            InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                                                    "aq:ime:" + counterSuffix);
            InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
            InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                                                                      "aq:native-pre-ime:" + counterSuffix);

            mFirstInputStage = nativePreImeStage;
            mFirstPostImeInputStage = earlyPostImeStage;
            mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
        }
    }
}

```

看上面最后面几行，设置一系列的输入管道，一个触屏事件的发生是由屏幕发起，然后经过驱动层一系列的优化计算通过 Socket 跨进程通知 Android Framework 层（实际上就是 WMS），最终屏幕的触摸事件会被发送到上面的输入管道中。

当某一个屏幕触摸事件到达其中的 ViewPostImeInputState 时，会经过 onProcess 来处理：

```java
// ViewPostImeInputState.java
@Override
protected int onProcess(QueuedInputEvent q) {
    if (q.mEvent instanceof KeyEvent) {
        return processKeyEvent(q);
    } else {
        final int source = q.mEvent.getSource();
        if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
            return processPointerEvent(q);
        } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
            return processTrackballEvent(q);
        } else {
            return processGenericMotionEvent(q);
        }
    }
}

private int processPointerEvent(QueuedInputEvent q) {
    final MotionEvent event = (MotionEvent)q.mEvent;

    mAttachInfo.mUnbufferedDispatchRequested = false;
    mAttachInfo.mHandlingPointerEvent = true;
    // 调用到了mView的方法
    boolean handled = mView.dispatchPointerEvent(event);
    maybeUpdatePointerIcon(event);
    maybeUpdateTooltip(event);
    mAttachInfo.mHandlingPointerEvent = false;
    if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
        mUnbufferedInputDispatch = true;
        if (mConsumeBatchedInputScheduled) {
            scheduleConsumeBatchedInputImmediately();
        }
    }
    return handled ? FINISH_HANDLED : FORWARD;
}

```

可以看到在 onProcess 中最终调用了一个 mView的dispatchPointerEvent 方法，mView 实际上就是 PhoneWindow 中的 DecorView，而 dispatchPointerEvent 是被 View.java 实现的：

```java
// View.java
@UnsupportedAppUsage
public final boolean dispatchPointerEvent(MotionEvent event) {
    if (event.isTouchEvent()) {
        return dispatchTouchEvent(event);
    } else {
        return dispatchGenericMotionEvent(event);
    }
}
```

在dispatchPointerEvent方法中调用了DecorView的dispatchTouchEvent方法：

```java
// DecorView.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
        ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

热这里的callback是在Activity启动阶段，实例化Activity对象后通过其attach方法设置callback：

```java
// Activity.java
@UnsupportedAppUsage
final void attach(Context context, ActivityThread aThread,
                  Instrumentation instr, IBinder token, int ident,
                  Application application, Intent intent, ActivityInfo info,
                  CharSequence title, Activity parent, String id,
                  NonConfigurationInstances lastNonConfigurationInstances,
                  Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                  Window window, ActivityConfigCallback activityConfigCallback) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    // 设置callback
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ...
}
```

至此，事件由DecorView传递到了Activity中，交由其dispatchTouchEvent来处理：

```java
// Activity.java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

Touch 事件在 Activity 中只是绕了一圈最后还是回到了 PhoneWindow 中的 DecorView 来处理:

```java
// PhoneWindow.java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

接下来就是从DecorView往下开始交给子View进行事件传递了。

## 总结

上面整个过程，大部分View的操作都被添加到Window中，而Activity中有一个 window，也就是 PhoneWindow 对象，在 PhoneWindow 中有一个 DecorView，在 setContentView 中会将 layout 填充到此 DecorView 中。

而Window对View的操作是通过WindowManager来处理的。WindowManager提供在Window上添加View、移除View和更新View的操作。

从上面可以看出addView的操作是通过ViewRootImpl完成Window的添加工作的，即WindowMangerGlobal 通过调用 ViewRootImpl 的 setView 方法，在其方法中通过WindowSession完成 window 的添加过程，这期间经历了一次IPC调用。setView 方法中主要完成两件事情：View 渲染（requestLayout）以及接收触屏事件。

