---
layout: post
title: Android Framework源码分析-Activity的启动过程
tags: 源码分析 framework
categories: Android
date: 2019-05-18
---

Activity是我们平时用到最多的一个组件之一，它提供给用户一个可以交互的页面，通常启动一个Activity可以使用下面代码：

```java
Intent intent = new Intent(this, DemoActivity.class);
this.startActivity(intent);
```

可以看出代码很简洁，但是背后有着复杂的执行流程，这篇文章就介绍下Activity启动的执行过程和工作原理。

## 流程分析

### Activity启动

我们先从Activity的startActivity说起：

```java
// Activity.java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
```

最终调用到了Activity的startActivityForResult方法，我们烂了看喜爱这个方法：

```java
// Activity.java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                   @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
            this, mMainThread.getApplicationThread(), mToken, this,
            intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
            // activity hidden during this time, to avoid flickering.
            // This can only be done when a result is requested because
            // that guarantees we will get information back when the
            // activity is finished, no matter what happens to it.
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            // Note we want to go through this method for compatibility with
            // existing applications that may have overridden it.
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

这里调用了调用了mInstrumentation.execStartActivity方法，其中第一个参数mMainThread.getApplicationThread()，它返回的类型是ApplicationThread，ApplicationThread是ActivityThread的内部类，继承IApplicationThread.Stub，也是个Binder对象，用来实现进程间通信，具体来说就是AMS所在系统进程通知应用程序进程进行一系列操作，而mInstrumentation常用来跟踪Application和Activity生命周期，一般我们会在Android应用测试框架中看到它。

```java
// Instrumentation.java
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, Activity target,
    Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    Uri referrer = target != null ? target.onProvideReferrer() : null;
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }
    ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                           intent.resolveTypeIfNeeded(who.getContentResolver()),
                           token, target != null ? target.mEmbeddedID : null,
                           requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

在 Instrumentation 中，会通过 ActivityManger.getService 获取 AMS 的实例，然后调用其 startActivity 方法

```java
// ActivityManager.java
@UnsupportedAppUsage
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

@UnsupportedAppUsage
private static final Singleton<IActivityManager> IActivityManagerSingleton =
    new Singleton<IActivityManager>() {
    @Override
    protected IActivityManager create() {
        final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
        final IActivityManager am = IActivityManager.Stub.asInterface(b);
        return am;
    }
};
```

从上面代码可以看出，这里获取一个跨进程的系统调用，返回一个binder实例，通过IActivityManager.Stub.asInterface方法返回一个AMS的代理对象，从而调用AMS 的startActivity方法。

而下面的checkStartActivityResult(result, intent)方法则是检查Activity启动结果：

```java
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 115609023)
public static void checkStartActivityResult(int res, Object intent) {
    if (!ActivityManager.isStartResultFatalError(res)) {
        return;
    }

    switch (res) {
        case ActivityManager.START_INTENT_NOT_RESOLVED:
        case ActivityManager.START_CLASS_NOT_FOUND:
            if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                throw new ActivityNotFoundException(
                "Unable to find explicit activity class "
                + ((Intent)intent).getComponent().toShortString()
                + "; have you declared this activity in your AndroidManifest.xml?");
            throw new ActivityNotFoundException(
                "No Activity found to handle " + intent);
        case ActivityManager.START_PERMISSION_DENIED:
            throw new SecurityException("Not allowed to start activity "
                                        + intent);
        case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
            throw new AndroidRuntimeException(
                "FORWARD_RESULT_FLAG used while also requesting a result");
        case ActivityManager.START_NOT_ACTIVITY:
            throw new IllegalArgumentException(
                "PendingIntent is not an activity");
        case ActivityManager.START_NOT_VOICE_COMPATIBLE:
            throw new SecurityException(
                "Starting under voice control not allowed for: " + intent);
        case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
            throw new IllegalStateException(
                "Session calling startVoiceActivity does not match active session");
        case ActivityManager.START_VOICE_HIDDEN_SESSION:
            throw new IllegalStateException(
                "Cannot start voice activity on a hidden session");
        case ActivityManager.START_ASSISTANT_NOT_ACTIVE_SESSION:
            throw new IllegalStateException(
                "Session calling startAssistantActivity does not match active session");
        case ActivityManager.START_ASSISTANT_HIDDEN_SESSION:
            throw new IllegalStateException(
                "Cannot start assistant activity on a hidden session");
        case ActivityManager.START_CANCELED:
            throw new AndroidRuntimeException("Activity could not be started for "
                                              + intent);
        default:
            throw new AndroidRuntimeException("Unknown error code "
                                              + res + " when starting " + intent);
    }
}
```

里面比较熟悉的就是我们没有在清单文件注册Activity时报的错误：ActivityManager.START_CLASS_NOT_FOUND:

`Unable to find explicit activity class XXX; have you declared this activity in your AndroidManifest.xml?`

### ActivityManagerService

接下来我们一起看下AMS的startActivity方法：

```java
// ActivityManagerService.java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
                               Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                               int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                               resultWho, requestCode, startFlags, profilerInfo, bOptions,
                               UserHandle.getCallingUserId());
}

@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                     Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                                     int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                               resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                               true /*validateIncomingUser*/);
}

public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                     Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                                     int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
                                     boolean validateIncomingUser) {
    enforceNotIsolatedCaller("startActivity");

    userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                                                      Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
        .setCaller(caller)
        .setCallingPackage(callingPackage)
        .setResolvedType(resolvedType)
        .setResultTo(resultTo)
        .setResultWho(resultWho)
        .setRequestCode(requestCode)
        .setStartFlags(startFlags)
        .setProfilerInfo(profilerInfo)
        .setActivityOptions(bOptions)
        .setMayWait(userId)
        .execute();

}
```

这里经过一系列调用，最终通过getActivityStartController().obtainStarter方法获取ActivityStarter实例，然后经过方法调用，最后执行execute启动Activity。

```java
// ActivityStarter.java
int execute() {
    try {
        // TODO(b/64750076): Look into passing request directly to these methods to allow
        // for transactional diffs and preprocessing.
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
             mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
             mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
             mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
             mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
             mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
             mRequest.inTask, mRequest.reason,
             mRequest.allowPendingRemoteAnimationRegistryLookup,
             mRequest.originatingPendingIntent);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, 			                mRequest.ephemeralIntent,
                                 mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                                 mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                                 mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                                 mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                                 mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                                 mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                                 mRequest.outActivity, mRequest.inTask, mRequest.reason,
                                 mRequest.allowPendingRemoteAnimationRegistryLookup,
                                 mRequest.originatingPendingIntent);
        }
    } finally {
        onExecutionComplete();
    }
}
```

分了两种情况，不过不论startActivityMayWait还是startActivity最终都是走到下面这个startActivity方法:

```java
// ActivityStarter.java
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
                          String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
                          String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
                          SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
                          ActivityRecord[] outActivity, TaskRecord inTask, String reason,
                          boolean allowPendingRemoteAnimationRegistryLookup,
                          PendingIntentRecord originatingPendingIntent) {

    if (TextUtils.isEmpty(reason)) {
        throw new IllegalArgumentException("Need to specify a reason.");
    }
    mLastStartReason = reason;
    mLastStartActivityTimeMs = System.currentTimeMillis();
    mLastStartActivityRecord[0] = null;

    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                                             aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                                             callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                                             options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                                             inTask, allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent);

    if (outActivity != null) {
        // mLastStartActivityRecord[0] is set in the call to startActivity above.
        outActivity[0] = mLastStartActivityRecord[0];
    }

    return getExternalResult(mLastStartActivityResult);
}
```

在这里调用了重载的startActivity方法，而最终会调用的 ActivityStarter 中的 startActivityUnchecked 方法来获取启动 Activity 的结果:

```java
// ActivityStarter.java
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                          ActivityRecord[] outActivity) {
    int result = START_CANCELED;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                                        startFlags, doResume, options, inTask, outActivity);
    } finally {
        // If we are not able to proceed, disassociate the activity from the task. Leaving an
        // activity in an incomplete state can lead to issues, such as performing operations
        // without a window container.
        final ActivityStack stack = mStartActivity.getStack();
        if (!ActivityManager.isStartResultSuccessful(result) && stack != null) {
            stack.finishActivityLocked(mStartActivity, RESULT_CANCELED,
                                       null /* intentResultData */, "startActivity", true /* oomAdj */);
        }
        mService.mWindowManager.continueSurfaceLayout();
    }

    postStartActivityProcessing(r, result, mTargetStack);

    return result;
}
```

进入到startActivityUnchecked方法里：

```java
// ActivityStarter.java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                                   IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                   int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                                   ActivityRecord[] outActivity) {
setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);

    computeLaunchingTaskFlags();
    computeSourceStack();
    mIntent.setFlags(mLaunchFlags);
    ActivityRecord reusedActivity = getReusableIntentActivity();
    
    ...
        
    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,mOptions);
    mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
	...
}
```

上面方法最后会调用ActivityStack的resumeTopActivityUncheckedLocked方法：

```java
// ActivityStack.java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mStackSupervisor.inResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mStackSupervisor.inResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);

        // When resuming the top activity, it may be necessary to pause the top activity (for
        // example, returning to the lock screen. We suppress the normal pause logic in
        // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
        // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
        // to ensure any necessary pause logic occurs. In the case where the Activity will be
        // shown regardless of the lock screen, the call to
        // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
        if (next == null || !next.canTurnScreenOn()) {
            checkReadyForSleep();
        }
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }

    return result;
}
```

在上面方法里会执行resumeTopActivityInnerLocked(prev, options)方法：

```java
// ActivityStack.java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
        
    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
    if (mResumedActivity != null) {
        if (DEBUG_STATES) Slog.d(TAG_STATES,"resumeTopActivityLocked: Pausing " + mResumedActivity);
        // 暂停上一个Activity
        pausing |= startPausingLocked(userLeaving, false, next, false);
    }
    
    ...
        
    if (next.app != null && next.app.thread != null) {
        ... 
        try {
            
            final ClientTransaction transaction = ClientTransaction.obtain(next.app.thread,                                                             next.appToken);
            // Deliver all pending results.
            ArrayList<ResultInfo> a = next.results;
            if (a != null) {
                final int N = a.size();
                if (!next.finishing && N > 0) {
                    if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                       "Delivering results to " + next + ": " + a);
                    transaction.addCallback(ActivityResultItem.obtain(a));
                }
            }

            if (next.newIntents != null) {
                transaction.addCallback(NewIntentItem.obtain(next.newIntents,
                                                             false /* andPause */));
            }

            // Well the app will no longer be stopped.
            // Clear app token stopped state in window manager if needed.
            next.notifyAppResumed(next.stopped);
            EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                            System.identityHashCode(next), next.getTask().taskId,
                            next.shortComponentName);

            next.sleeping = false;
            mService.getAppWarningsLocked().onResumeActivity(next);
            mService.showAskCompatModeDialogLocked(next);
            next.app.pendingUiClean = true;
            next.app.forceProcessStateUpTo(mService.mTopProcessState);
            next.clearOptionsLocked();
            transaction.setLifecycleStateRequest(
                ResumeActivityItem.obtain(next.app.repProcState,
                                          mService.isNextTransitionForward()));
            
            mService.getLifecycleManager().scheduleTransaction(transaction);
    }  else {
            ....
            if (SHOW_APP_STARTING_PREVIEW) {
            	    // 冷启动时展示的白屏窗口
                    next.showStartingWindow(null , false ,false);
                }
            ...
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
    }
    
}
```

上面流程先对上一个Activity暂停，再执行创建操作最终进入到ActivityStackSupervisor.startSpecificActivityLocked方法中，这里有一个地方需要注意下，就是在启动前先展示了一个白屏窗口。

```java
// ActivityStackSupervisor.java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                                                        r.info.applicationInfo.uid, true);

    if (app != null && app.thread != null) {
        try {
            if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                || !"android".equals(r.info.packageName)) {
                // Don't add this if it is a platform component that is marked
                // to run in multiple processes, because this is actually
                // part of the framework so doesn't make sense to track as a
                // separate apk in the process.
                app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                               mService.mProcessStats);
            }
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                   + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }

    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                                "activity", r.intent.getComponent(), false, false, true);
}

```

在上面这个方法中调用了realStartActivityLocked方法:

```java
// ActivityStackSupervisor.java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
                                      boolean andResume, boolean checkConfig) throws RemoteException {
	...
        
    // Create activity launch transaction.
        final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                                                                             r.appToken);
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                                                            System.identityHashCode(r), r.info,
                                                            // TODO: Have this take the merged configuration instead of separate global
                                                            // and override configs.
                                                            mergedConfiguration.getGlobalConfiguration(),
                                                            mergedConfiguration.getOverrideConfiguration(), r.compat,
                                                            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                                                            r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                                                            profilerInfo));

    // Set desired final state.
    final ActivityLifecycleItem lifecycleItem;
    if (andResume) {
        lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
    } else {
        lifecycleItem = PauseActivityItem.obtain();
    }
    clientTransaction.setLifecycleStateRequest(lifecycleItem);

    // Schedule transaction.
    mService.getLifecycleManager().scheduleTranssaction(clientTransaction);
    
    ...
}
```

上面方法中创建了Activity启动事务，并传入 app.thread 参数，它是 ApplicationThread 类型。在上文 startActivity 阶段已经提过 ApplicationThread 是为了实现进程间通信的，是 ActivityThread 的一个内部类。

我们看下obtain方法：

```java
// ClientTransaction.java
public static ClientTransaction obtain(IApplicationThread client, IBinder activityToken) {
    ClientTransaction instance = ObjectPool.obtain(ClientTransaction.class);
    if (instance == null) {
        instance = new ClientTransaction();
    }
    
    // 在这里把IApplicationThread 赋值给了mClient成员变量
    instance.mClient = client;
    instance.mActivityToken = activityToken;

    return instance;
}
```

而ClientTransaction是包含一系列的待客户端处理的事务的容器，客户端接收后取出事务并执行。

紧接着使用clientTransaction.addCallback添加了LaunchActivityItem实例：

```java
// ClientTransaction.java
private List<ClientTransactionItem> mActivityCallbacks;
...
public void addCallback(ClientTransactionItem activityCallback) {
    if (mActivityCallbacks == null) {
        mActivityCallbacks = new ArrayList<>();
    }
    mActivityCallbacks.add(activityCallback);
}
```

LaunchActivityItem实例的获取：

```java
// LaunchActivityItem.java
/** Obtain an instance initialized with provided params. */
public static LaunchActivityItem obtain(Intent intent, int ident, ActivityInfo info,
                                        Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo,
                                        String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state,
                                        PersistableBundle persistentState, List<ResultInfo> pendingResults,
                                        List<ReferrerIntent> pendingNewIntents, boolean isForward, ProfilerInfo profilerInfo) {
    LaunchActivityItem instance = ObjectPool.obtain(LaunchActivityItem.class);
    if (instance == null) {
        instance = new LaunchActivityItem();
    }
    setValues(instance, intent, ident, info, curConfig, overrideConfig, compatInfo, referrer,
              voiceInteractor, procState, state, persistentState, pendingResults,
              pendingNewIntents, isForward, profilerInfo);

    return instance;
}
```

最后调用了mService.getLifecycleManager().scheduleTransaction(clientTransaction)，其中mService是ActivityManagerService，getLifecycleManager()方法获取的是ClientLifecycleManager实例：

```java
// ClientLifecycleManager.java
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    transaction.schedule();
    if (!(client instanceof Binder)) {
        // If client is not an instance of Binder - it's a remote call and at this point it is
        // safe to recycle the object. All objects used for local calls will be recycled after
        // the transaction is executed on client in ActivityThread.
        transaction.recycle();
    }
}
```

上面就是调用ClientTransaction的schedule方法：

```java
// ClientTransaction.java
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

这里的mClient就是上面被赋值的IApplicationThread对象，然后调用IApplicationThread的scheduleTransaction方法。由于IApplicationThread是ApplicationThread所在系统进程的代理，所以真正执行的地方就是 客户端的ApplicationThread中了。所以到这里startActivity操作又从AMS转移到了应用进程ApplicationThread中。

###  ApplicationThread

接着分析scheduleTransaction方法，ApplicationThread是ActivityThread的一个内部类

```java
private class ApplicationThread extends IApplicationThread.Stub {
    @Override
    public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
    }
}
```

可以看出，这里还是调用了ActivityThread 的 scheduleTransaction 方法。但是这个方法实际上是在 ActivityThread 的父类 ClientTransactionHandler 中实现：

```java
// ClientTransactionHandler.java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

调用 sendMessage 方法，向 Handler 中发送了一个 EXECUTE_TRANSACTION 的消息，并且 Message 中的 obj 就是启动 Activity 的事务对象。而这个 Handler 的具体实现是 ActivityThread 中的 mH 对象：

```java
// ActivityThread.java
class H extends Handler {
    
    public void handleMessage(Message msg) {
    	...
            
        case EXECUTE_TRANSACTION:
        final ClientTransaction transaction = (ClientTransaction) msg.obj;
        mTransactionExecutor.execute(transaction);
        if (isSystem()) {
            // Client transactions inside system process are recycled on the client side
            // instead of ClientLifecycleManager to avoid being cleared before this
            // message is handled.
            transaction.recycle();
        }   
    }
}
```

最终调用了事务的 execute 方法，execute 方法如下：

```java
// TransactionExecutor.java
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();
    log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);

    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
    log("End resolving transaction");
}
```

接着看下executeCallbacks方法：

```java
// TransactionExecutor.java
public void executeCallbacks(ClientTransaction transaction) {
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
    if (callbacks == null) {
        // No callbacks to execute, return early.
        return;
    }
    log("Resolving callbacks");

    final IBinder token = transaction.getActivityToken();
    ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

    // In case when post-execution state of the last callback matches the final state requested
    // for the activity in this transaction, we won't do the last transition here and do it when
    // moving to final state instead (because it may contain additional parameters from server).
    final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
    final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
        : UNDEFINED;
    // Index of the last callback that requests some post-execution state.
    final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);

    final int size = callbacks.size();
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        log("Resolving callback: " + item);
        final int postExecutionState = item.getPostExecutionState();
        final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                                                                                 item.getPostExecutionState());
        if (closestPreExecutionState != UNDEFINED) {
            cycleToPath(r, closestPreExecutionState);
        }

        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
        if (r == null) {
            // Launch activity request will create an activity record.
            r = mTransactionHandler.getActivityClient(token);
        }

        if (postExecutionState != UNDEFINED && r != null) {
            // Skip the very last transition and perform it by explicit state request instead.
            final boolean shouldExcludeLastTransition =
                i == lastCallbackRequestingState && finalState == postExecutionState;
            cycleToPath(r, postExecutionState, shouldExcludeLastTransition);
        }
    }
}
```

在这个方法里，遍历事务中的callback，执行了item.execute(mTransactionHandler, token, mPendingActions)方法，这里的item也就是上面通过ClientTransaction.obtain方法获取到ClientTransaction对象后，添加的LaunchActivityItem对象。所以看下LaunchActivityItem的execute方法：

```java
// LaunchActivityItem.java
@Override
public void execute(ClientTransactionHandler client, IBinder token,
                    PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                                                      mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                                                      mPendingResults, mPendingNewIntents, mIsForward,
                                                      mProfilerInfo, client);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

在该方法中执行了ClientTransactionHandler的handleLaunchActivity方法，其实现类就是ActivityThread，因此最终又回到了ActivityThread类。

### ActivityThread

我们进入到handleLaunchActivity方法：

```java
// ActivityThread.java
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
                                     PendingTransactionActions pendingActions, Intent customIntent) {
    ...
        WindowManagerGlobal.initialize();

    final Activity a = performLaunchActivity(r, customIntent);

    ...
        return a;
}
```

这里初始化了WindowManager，即每一个Activity都会对应一个窗口，并调用了performLaunchActivity方法：

```java
// ActivityThread.java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
 	...
    // 创建Context的实现ContextImpl上下文对象
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        // 创建Activity对象
        activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }
    
    ...
    // 创建Application对象
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    
    Window window = null;
    if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
        window = r.mPendingRemoveWindow;
        r.mPendingRemoveWindow = null;
        r.mPendingRemoveWindowManager = null;
    }
    appContext.setOuterContext(activity);
    // 为activity关联上下文环境
    activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);
    ...
    // 调用生命周期onCreate
    if (r.isPersistable()) {
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
    } else {
        mInstrumentation.callActivityOnCreate(activity, r.state);
    }
}

```

上面代码通过反射常见目标Activity对象，紧接着通过activity的attach方法，建立了Activity与Context的联系，并且创建了PhoneWindow对象，最后通过Instrumentation的callActivityOnCreate调用Activity的onCreate方法：

```java
// Instrumentation.java
public void callActivityOnCreate(Activity activity, Bundle icicle,
                                 PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

```java
// Activity.java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    mCanEnterPictureInPicture = true;
    restoreHasCurrentPermissionRequest(icicle);
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
    writeEventLog(LOG_AM_ON_CREATE_CALLED, "performCreate");
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
        com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
}
```

至此，目标Activity已经被创建成功并执行生命周期方法。

### onResume

上面方法执行到了onCreate方法，那么接下来会跟进onStart、onResume方法：其实这里有几个类：

- LaunchActivityItem 远程App端的onCreate生命周期事务

- ResumeActivityItem 远程App端的onResume生命周期事务

- PauseActivityItem 远程App端的onPause生命周期事务

- StopActivityItem 远程App端的onStop生命周期事务

- DestroyActivityItem 远程App端onDestroy生命周期事务

还记得上面ActivityStackSupervisor的realStartActivityLocked方法：

```java
// ActivityStackSupervisor.java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
                                      boolean andResume, boolean checkConfig) throws RemoteException {
	...
        
    // Create activity launch transaction.
        final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                                                                             r.appToken);
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                                                            System.identityHashCode(r), r.info,
                                                            // TODO: Have this take the merged configuration instead of separate global
                                                            // and override configs.
                                                            mergedConfiguration.getGlobalConfiguration(),
                                                            mergedConfiguration.getOverrideConfiguration(), r.compat,
                                                            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                                                            r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                                                            profilerInfo));

    // Set desired final state.
    final ActivityLifecycleItem lifecycleItem;
    // 返回ResumeActivityItem实例
    if (andResume) {
        lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
    } else {
        lifecycleItem = PauseActivityItem.obtain();
    }
    clientTransaction.setLifecycleStateRequest(lifecycleItem);

    // Schedule transaction.
    mService.getLifecycleManager().scheduleTranssaction(clientTransaction);
    
    ...
}
```

在上面的clientTransaction.setLifecycleStateRequest(lifecycleItem)方法中，lifecycleItem是ResumeActivityItem或PauseActivityItem实例，这里我们关注ResumeActivityItem，先看下setLifecycleStateRequest方法：

```java
// ClientTransaction.java
public void setLifecycleStateRequest(ActivityLifecycleItem stateRequest) {
    mLifecycleStateRequest = stateRequest;
}
```

mLifecycleStateRequest表示执行transaction后的最终的生命周期状态。继续看处理ActivityThread.H.EXECUTE_TRANSACTION这个消息的处理，即TransactionExecutor的execute方法：

```java
// TransactionExecutor.java
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();
    log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);

    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
    log("End resolving transaction");
}
```

进入到executeLifecycleState方法中：

```java
// TransactionExecutor.java
private void executeLifecycleState(ClientTransaction transaction) {
    final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
    if (lifecycleItem == null) {
        // No lifecycle request, return early.
        return;
    }
    log("Resolving lifecycle state: " + lifecycleItem);

    final IBinder token = transaction.getActivityToken();
    final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

    if (r == null) {
        // Ignore requests for non-existent client records for now.
        return;
    }

    // Cycle to the state right before the final requested state.
    cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

    // Execute the final transition with proper parameters.
    lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
    lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
}

```

最终取到了ActivityLifecycleItem对象，即上面的ResumeActivityItem 对象，然后执行其execute方法：

```java
// ResumeActivityItem.java
@Override
public void execute(ClientTransactionHandler client, IBinder token,
                    PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
    client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                                "RESUME_ACTIVITY");
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

和上面类似，这里其实又调用了ActivityThread的handleResumeActivity方法：

```java
// ActivityThread.java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,String reason) {
    ...
    // TODO Push resumeArgs into the activity for consideration
    // 这里会执行onStart 、 onResume方法
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        if (r.mPreserveWindow) {
            a.mWindowAdded = true;
            r.mPreserveWindow = false;
            // Normally the ViewRoot sets up callbacks with the Activity
            // in addView->ViewRootImpl#setView. If we are instead reusing
            // the decor view we have to notify the view root that the
            // callbacks may have changed.
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
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

        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }
    
    ...
        
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        if (r.newConfig != null) {
            performConfigurationChangedForActivity(r, r.newConfig);
            if (DEBUG_CONFIGURATION) {
                Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                       + r.activity.mCurrentConfig);
            }
            r.newConfig = null;
        }
        if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
        WindowManager.LayoutParams l = r.window.getAttributes();
        if ((l.softInputMode
             & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
            != forwardBit) {
            l.softInputMode = (l.softInputMode
                               & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                | forwardBit;
            if (r.activity.mVisibleFromClient) {
                ViewManager wm = a.getWindowManager();
                View decor = r.window.getDecorView();
                wm.updateViewLayout(decor, l);
            }
        }

        r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        if (r.activity.mVisibleFromClient) {
            // 添加window并设置为可见
            r.activity.makeVisible();
        }
    }
}
```

上面我们看下r.activity.makeVisible()方法：

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

这里把Activity的顶级布局mDecor通过windowManager的addView方法，添加到window，并设置mDecor可见，所以到这里，我们的视图才是真正可见了，即onResume方法调用后。另外Activity视图渲染到Window后，会设置window焦点变化，先走到DecorView的onWindowFocusChanged方法，最后是到Activity的onWindowFocusChanged方法，表示首帧绘制完成，此时Activity可交互。

接着我们看下performResumeActivity方法：

```java
// ActivityThread.java
public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
                                                  String reason) {
    final ActivityClientRecord r = mActivities.get(token);
    
    ...
        
    r.activity.performResume(r.startsNotResumed, reason);
    ...
}
```

内部执行了Activity的performResume方法：

```java
final void performResume(boolean followedByPause, String reason) {
    performRestart(true /* start */, reason);

    ...
    // mResumed is set by the instrumentation
    mInstrumentation.callActivityOnResume(this);
    ...
}
```

这里先调用了performRestart()，performRestart()又会调用performStart()，performStart()内部调用了mInstrumentation.callActivityOnStart(this)，也就是Activity的onStart()方法了。

然后是mInstrumentation.callActivityOnResume，即调用Activity的onResume()方法了，到这里启动后的生命周期走完了。

## 总结

最后我们把这一启动过程用图梳理出来：

![image-20220830200947970](https://tva1.sinaimg.cn/large/e6c9d24ely1h5p1v180x8j21bx0u07eq.jpg)

涉及的类：

| 类                      | 作用                                                         |
| ----------------------- | ------------------------------------------------------------ |
| ActivityThread          | 应用的入口类，系统通过调用main函数，开启消息循环队列。ActivityThread所在线程被称为应用的主线程（UI线程） |
| ApplicationThread       | 是ActivityThread的内部类，继承IApplicationThread.Stub，是一个IBinder，是ActiivtyThread和AMS通信的桥梁，AMS则通过代理调用此App进程的本地方法，运行在Binder线程池 |
| H                       | 继承Handler，在ActivityThread中初始化，即主线程Handler，用于主线程所有消息的处理。本片中主要用于把消息从Binder线程池切换到主线程 |
| Intrumentation          | 追踪application及activity生命周期的功能，用于监控app和系统的交互 |
| ActivityManagerService  | Android中最核心的服务之一，负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要，它本身也是一个Binder的实现类。 |
| ActivityStarter         | 用于解释如何启动Activity。该类收集所有逻辑，用于确定Intent和flag应如何转换为Activity以及相关的任务和堆栈 |
| ActivityStack           | 用来管理系统所有的Activity，内部维护了Activity的所有状态和Activity相关的列表等数据 |
| ActivityStackSupervisor | 负责所有Activity栈的管理。AMS的stack管理主要有三个类，ActivityStackSupervisor，ActivityStack和TaskRecord |
| ClientLifecycleManager  | 客户端生命周期执行请求管理                                   |
| ClientTransaction       | 是包含一系列的 待客户端处理的事务 的容器，客户端接收后取出事务并执行 |
| LaunchActivityItem      | 继承ClientTransactionItem，客户端要执行的事务信息，启动activity |
| ResumeActivityItem      | 继承ClientTransactionItem，客户端要执行的事务信息，启动activity |
