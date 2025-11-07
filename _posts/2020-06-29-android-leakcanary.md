---
layout: post
title: Android三方开源库之LeakCanary2.4源码分析
tags: 源码分析
categories: Android
date: 2020-06-29
---

LeanCanary内部主要使用了Reference以及ReferenceQueue配合来实现对象被回收时的监听，这是它的核心逻辑，因此我们先了解下这部分内容：

## Reference

Reference是一个泛型抽象类，其中软引用、弱引用、虚引用都继承自它，它主要有几个成员变量：

- 泛型引用对象 T：被回收时被赋值为null
- 引用队列 ReferenceQueue：一个单链表实现的队列，保存即将被回收的引用对象
- queueNext：指向下一个待处理的Reference引用对象
- pendingNext：指向下一个待入列的Reference引用对象

## ReferenceQueue

Reference配合ReferenceQueue就可以实现对象回收监听，示例代码如下：

```java
//创建一个引用队列
ReferenceQueue queue = new ReferenceQueue();
//创建弱引用，并关联引用队列queue
WeakReference reference = new WeakReference(new Object(),queue);
System.out.println(reference);
System.gc();
//当reference被成功回收后，可以从queue中获取到该引用
System.out.println(queue.remove());
```

被回收后的对象如果在引用队列中找到该引用，则说明对象被回收了，否则就表明该引用有内存泄漏的风险，这也就是LeakCanary的基本原理。

## 源码分析

LeakCanary在2.0之前版本，通过install方法来完成初始化，但是2.0之后，内部继承ContentProvider完成初始化工作，我们知道App启动后调用一系列声明周期方法：Application->attachBaseContext =====>ContentProvider->onCreate =====>Application->onCreate =====>Activity->onCreate，可见这里面会调用到ContentProvider的onCreate方法，我们的初始化工作在这里面完成就可以了：

找到`leakcanary-object-watcher-android`项目的manifest文件，可以看到：

```xml
<application>
    <provider
         android:name="leakcanary.internal.AppWatcherInstaller$MainProcess"
         android:authorities="${applicationId}.leakcanary-installer"
         android:enabled="@bool/leak_canary_watcher_auto_install"
         android:exported="false" />
</application>
```

AppWatcherInstaller该类继承自ContentProvider，App启动后调用它的onCreate方法：

```kotlin
// AppWatcherInstaller.kt
internal sealed class AppWatcherInstaller : ContentProvider() {
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    // 开始初始化工作
    AppWatcher.manualInstall(application)
    return true
  }
}
```

紧接着看下AppWatcher的manualInstall方法：

```kotlin
// AppWatcher.kt

fun manualInstall(application: Application) {
    InternalAppWatcher.install(application)
}
```

内部又调用了InternalAppWatcher的install方法：

```kotlin
// InternalAppWatcher.kt

fun install(application: Application) {
    checkMainThread()
    if (this::application.isInitialized) {
        return
    }
    // 日志初始化
    SharkLog.logger = DefaultCanaryLog()
    InternalAppWatcher.application = application

    val configProvider = { AppWatcher.config }
    // Activity内存泄漏监听
    ActivityDestroyWatcher.install(application, objectWatcher, configProvider)
    // Fragment内存泄漏监听
    FragmentDestroyWatcher.install(application, objectWatcher, configProvider)
    // 注册内存泄漏事件回调
    onAppWatcherInstalled(application)
}
```

这个方法中首先检查是否在主线程中，然后判断Application是否完成初始化，经过一系列配置，进入到Activity和Fragment的内存泄漏监听，最后注册回调事件。

## Activity内存泄漏监听

```kotlin
// ActivityDestroyWatcher.kt
internal class ActivityDestroyWatcher private constructor(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
          objectWatcher.watch(
              activity, "${activity::class.java.name} received Activity#onDestroy() callback"
          )
        }
      }
    }

  companion object {
    fun install(
      application: Application,
      objectWatcher: ObjectWatcher,
      configProvider: () -> Config
    ) {
      val activityDestroyWatcher =
        ActivityDestroyWatcher(objectWatcher, configProvider)
  application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
    }
  }
}
```

上面代码中，通过调用Application的registerActivityLifecycleCallbacks方法，注册lifecycleCallback，进而可以通过回调获取app的的每一个Activity生命周期变化，这里只重写了onActivityDestroyed方法，原因是在于by noOpDelegate(),通过类委托机制将其他回调实现都交给noOpDelegate，而noOpDelegate是一个空实现的动态代理。在遇到只需要实现接口的部分方法时，就可以这么做，其他方法实现都委托给空实现代理类就好了。

## Fragment内存泄漏监听

```kotlin
// FragmentDestroyWatcher.kt

fun install(
    application: Application,
    objectWatcher: ObjectWatcher,
    configProvider: () -> AppWatcher.Config
) {
    val fragmentDestroyWatchers = mutableListOf<(Activity) -> Unit>()

    // Android O后构建AndroidOFragmentDestroyWatcher
    if (SDK_INT >= O) {
        fragmentDestroyWatchers.add(
            AndroidOFragmentDestroyWatcher(objectWatcher, configProvider)
        )
    }
	// androidx.fragment.app.Fragment
    getWatcherIfAvailable(
        ANDROIDX_FRAGMENT_CLASS_NAME,
        ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
        objectWatcher,
        configProvider
    )?.let {
        fragmentDestroyWatchers.add(it)
    }
	// android.support.v4.app.Fragment
    getWatcherIfAvailable(
        ANDROID_SUPPORT_FRAGMENT_CLASS_NAME,
        ANDROID_SUPPORT_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
        objectWatcher,
        configProvider
    )?.let {
        fragmentDestroyWatchers.add(it)
    }

    if (fragmentDestroyWatchers.size == 0) {
        return
    }

    // 注册Activity生命周期回调，在Activity的onActivityCreated()方法中遍历这些watcher方法类型，实际调用的是对应的invoke方法
    application.registerActivityLifecycleCallbacks(object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
        override fun onActivityCreated(
            activity: Activity,
            savedInstanceState: Bundle?
        ) {
            for (watcher in fragmentDestroyWatchers) {
                watcher(activity)
            }
        }
    })
}
```

Fragment要分情况了：

- 如果系统是Android O以后版本，使用AndroidOFragmentDestroyWatcher
- 如果App中使用了androidx中的fragment，则添加对应的AndroidXFragmentDestroyWatcher
- 如果App中使用了support库中的fragment，则添加AndroidSupportFragmentDestroyWatcher

但是最终都会在invoke方法中使用对应的fragmentManager注册Fragment的生命周期回调，在onFragmentViewDestroyed()和onFragmentDestroyed()方法中使用ObjectWatcher来检测fragment。源码如下：

```java
// AndroidOFragmentDestroyWatcher.kt
internal class AndroidOFragmentDestroyWatcher(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) : (Activity) -> Unit {
  private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

    override fun onFragmentViewDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      val view = fragment.view
      if (view != null && configProvider().watchFragmentViews) {
        objectWatcher.watch(
            view, "${fragment::class.java.name} received Fragment#onDestroyView() callback " +
            "(references to its views should be cleared to prevent leaks)"
        )
      }
    }

    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      if (configProvider().watchFragments) {
        objectWatcher.watch(
            fragment, "${fragment::class.java.name} received Fragment#onDestroy() callback"
        )
      }
    }
  }

  override fun invoke(activity: Activity) {
    val fragmentManager = activity.fragmentManager
    fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
  }
}

```

这里拿AndroidOFragmentDestroyWatcher源码来看，其他两个类似，从上面可以看出在onFragmentDestroyed回调里面来处理检查Fragment是否正常被回收的检测逻辑，在onFragmentViewDestroyed回调里面来处理检查Fragment的View是否正常被回收的检测逻辑。另外在AndroidXFragmentDestroyWatcher中，LeakCanary增加了对ViewModel的检测，ViewModelClearedWatcher继承自ViewModel，里面使用viewModelMap来存储ViewModelStoreOwner中的ViewModel,并使用伴生对象来初始化自己，关联到ViewModelStoreOwner；在onCleared()方法中使用ObjectWatcher来监测。源码如下：

```kotlin
// ViewModelClearedWatcher.kt
internal class ViewModelClearedWatcher(
  storeOwner: ViewModelStoreOwner,
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) : ViewModel() {

  private val viewModelMap: Map<String, ViewModel>?

  init {
    // We could call ViewModelStore#keys with a package spy in androidx.lifecycle instead,
    // however that was added in 2.1.0 and we support AndroidX first stable release. viewmodel-2.0.0
    // does not have ViewModelStore#keys. All versions currently have the mMap field.
    viewModelMap = try {
      val mMapField = ViewModelStore::class.java.getDeclaredField("mMap")
      mMapField.isAccessible = true
      @Suppress("UNCHECKED_CAST")
      mMapField[storeOwner.viewModelStore] as Map<String, ViewModel>
    } catch (ignored: Exception) {
      null
    }
  }

  override fun onCleared() {
    if (viewModelMap != null && configProvider().watchViewModels) {
      viewModelMap.values.forEach { viewModel ->
        objectWatcher.watch(
            viewModel, "${viewModel::class.java.name} received ViewModel#onCleared() callback"
        )
      }
    }
  }

  companion object {
    fun install(
      storeOwner: ViewModelStoreOwner,
      objectWatcher: ObjectWatcher,
      configProvider: () -> Config
    ) {
      val provider = ViewModelProvider(storeOwner, object : Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel?> create(modelClass: Class<T>): T =
          ViewModelClearedWatcher(storeOwner, objectWatcher, configProvider) as T
      })
      provider.get(ViewModelClearedWatcher::class.java)
    }
  }
}

```

综上，我们发现最后都是交给ObjectWatcher来检测Activity、Fragment、Fragment中的View和ViewModel的，这里以Activity为例来看下：

```kotlin
// ObjectWatcher.kt
private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()

private val queue = ReferenceQueue<Any>()
@Synchronized fun watch(
    watchedObject: Any,
    description: String
) {
    if (!isEnabled()) {
        return
    }
    removeWeaklyReachableObjects()
    val key = UUID.randomUUID()
    .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
        "Watching " +
        (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
        (if (description.isNotEmpty()) " ($description)" else "") +
        " with key $key"
    }

    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
        moveToRetained(key)
    }
}

private fun removeWeaklyReachableObjects() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    do {
        ref = queue.poll() as KeyedWeakReference?
        if (ref != null) {
            watchedObjects.remove(ref.key)
        }
    } while (ref != null)
}
```

上面首先从watchedObjects集合Map中移除之前回收的引用，这里面的key为UUID，value为包装过的引用对象KeyedWeakReference，它继承自WeakReference：

```kotlin
// KeyedWeakReference.kt
class KeyedWeakReference(
  referent: Any,
  val key: String,
  val description: String,
  val watchUptimeMillis: Long,
  referenceQueue: ReferenceQueue<Any>
) : WeakReference<Any>(
    referent, referenceQueue
) {
  /**
   * Time at which the associated object ([referent]) was considered retained, or -1 if it hasn't
   * been yet.
   */
  @Volatile
  var retainedUptimeMillis = -1L

  companion object {
    @Volatile
    @JvmStatic var heapDumpUptimeMillis = 0L
  }

}
```

最终会执行一个后台线程来检查在5秒后，引用对象是否回收。

```kotlin
// InternalAppWatcher.kt
private val checkRetainedExecutor = Executor {
    mainHandler.postDelayed(it, AppWatcher.config.watchDurationMillis)
}

// AppWatcher.kt
val watchDurationMillis: Long = TimeUnit.SECONDS.toMillis(5)
```

上面被执行的任务就是moveToRetained方法：

```kotlin
// ObjectWatcher.kt
@Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
        retainedRef.retainedUptimeMillis = clock.uptimeMillis()
        onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
}
```

可以看到这里面会再次执行removeWeaklyReachableObjects方法，将引用队列中的引用对象从监听列表watchedObjects中移除，对于没有被移除的对象，则说明该引用对象未被添加到引用队列，即该引用对象可能存在内存泄漏的风险。最后如果有有内存泄露的引用对象，则遍历执行回调OnObjectRetainedListener的onObjectRetained方法。而onObjectRetained方法在哪实现的呢 ？

我们回到前面InternalAppWatcher.install初始化时

```kotlin
// InternalAppWatcher.kt
init {
    val internalLeakCanary = try {
        val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
        leakCanaryListener.getDeclaredField("INSTANCE")
        .get(null)
    } catch (ignored: Throwable) {
        NoLeakCanary
    }
    @kotlin.Suppress("UNCHECKED_CAST")
    onAppWatcherInstalled = internalLeakCanary as (Application) -> Unit
}
fun install(application: Application) {
    checkMainThread()
    if (this::application.isInitialized) {
        return
    }
    SharkLog.logger = DefaultCanaryLog()
    InternalAppWatcher.application = application

    val configProvider = { AppWatcher.config }
    ActivityDestroyWatcher.install(application, objectWatcher, configProvider)
    FragmentDestroyWatcher.install(application, objectWatcher, configProvider)
    onAppWatcherInstalled(application)
}
```

通过调用install方法，对应的调用了onAppWatcherInstalled方法，进而调用了leakcanary.internal.InternalLeakCanary类的invoke方法来完成注册监听，其中该类通过反射拿到InternalLeakCanary.INSTANCE单例对象，这个类位于另一个包leakcanary-android-core下：

```kotlin
// InternalLeakCanary.kt
internal object InternalLeakCanary : (Application) -> Unit, OnObjectRetainedListener {
    ...
    override fun invoke(application: Application) {
        _application = application

        checkRunningInDebuggableBuild()
		// 注册监听
        AppWatcher.objectWatcher.addOnObjectRetainedListener(this)
		// 创建AndroidHeapDumper对象，用于虚拟机dump hprof产生内存快照文件
        val heapDumper = AndroidHeapDumper(application, createLeakDirectoryProvider(application))
		// 用来触发GC
        val gcTrigger = GcTrigger.Default

        val configProvider = { LeakCanary.config }
		// 创建子线程及对应looper
        val handlerThread = HandlerThread(LEAK_CANARY_THREAD_NAME)
        handlerThread.start()
        val backgroundHandler = Handler(handlerThread.looper)

        // 创建HeapDumpTrigger对象，调用dumpHeap方法来创建hprof文件
        heapDumpTrigger = HeapDumpTrigger(
            application, backgroundHandler, AppWatcher.objectWatcher, gcTrigger, heapDumper,
            configProvider
        )
        // 注册应用可见监听
        application.registerVisibilityListener { applicationVisible ->
                                                this.applicationVisible = applicationVisible
                                                heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
                                               }
        registerResumedActivityListener(application)
        addDynamicShortcut(application)

        disableDumpHeapInTests()
    }
    override fun onObjectRetained() {
        if (this::heapDumpTrigger.isInitialized) {
            heapDumpTrigger.onObjectRetained()
        }
    }
}
```

而前面执行回调OnObjectRetainedListener的onObjectRetained方法，就是在这里被调用的，它会调用HeapDumpTrigger.onObjectRetained()方法：

```kotlin
// HeapDumpTrigger.kt
fun onObjectRetained() {
    scheduleRetainedObjectCheck(
        reason = "found new object retained",
        rescheduling = false
    )
}

private fun scheduleRetainedObjectCheck(
    reason: String,
    rescheduling: Boolean,
    delayMillis: Long = 0L
) {
    val checkCurrentlyScheduledAt = checkScheduledAt
    if (checkCurrentlyScheduledAt > 0) {
        val scheduledIn = checkCurrentlyScheduledAt - SystemClock.uptimeMillis()
        SharkLog.d { "Ignoring request to check for retained objects ($reason), already scheduled in ${scheduledIn}ms" }
        return
    } else {
        val verb = if (rescheduling) "Rescheduling" else "Scheduling"
        val delay = if (delayMillis > 0) " in ${delayMillis}ms" else ""
        SharkLog.d { "$verb check for retained objects${delay} because $reason" }
    }
    checkScheduledAt = SystemClock.uptimeMillis() + delayMillis
    backgroundHandler.postDelayed({
        checkScheduledAt = 0
        checkRetainedObjects(reason)
    }, delayMillis)
}

private fun checkRetainedObjects(reason: String) {
    val config = configProvider()
    // A tick will be rescheduled when this is turned back on.
    if (!config.dumpHeap) {
        SharkLog.d { "Ignoring check for retained objects scheduled because $reason: LeakCanary.Config.dumpHeap is false" }
        return
    }
	// 记录未回收对象的数量
    var retainedReferenceCount = objectWatcher.retainedObjectCount

    if (retainedReferenceCount > 0) {
        // 主动触发GC
        gcTrigger.runGc()
        // 重新更新未回收对象的数量
        retainedReferenceCount = objectWatcher.retainedObjectCount
    }
	// 如果未回收对象个数未达到阈值5个，就返回
    if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

    if (!config.dumpHeapWhenDebugging && DebuggerControl.isDebuggerAttached) {
        onRetainInstanceListener.onEvent(DebuggerIsAttached)
        showRetainedCountNotification(
            objectCount = retainedReferenceCount,
            contentText = application.getString(
                R.string.leak_canary_notification_retained_debugger_attached
            )
        )
        scheduleRetainedObjectCheck(
            reason = "debugger is attached",
            rescheduling = true,
            delayMillis = WAIT_FOR_DEBUG_MILLIS
        )
        return
    }

    val now = SystemClock.uptimeMillis()
    val elapsedSinceLastDumpMillis = now - lastHeapDumpUptimeMillis
    if (elapsedSinceLastDumpMillis < WAIT_BETWEEN_HEAP_DUMPS_MILLIS) {
        onRetainInstanceListener.onEvent(DumpHappenedRecently)
        showRetainedCountNotification(
            objectCount = retainedReferenceCount,
            contentText = application.getString(R.string.leak_canary_notification_retained_dump_wait)
        )
        // 60s内执行一次
        scheduleRetainedObjectCheck(
            reason = "previous heap dump was ${elapsedSinceLastDumpMillis}ms ago (< ${WAIT_BETWEEN_HEAP_DUMPS_MILLIS}ms)",
            rescheduling = true,
            delayMillis = WAIT_BETWEEN_HEAP_DUMPS_MILLIS - elapsedSinceLastDumpMillis
        )
        return
    }

    SharkLog.d { "Check for retained objects found $retainedReferenceCount objects, dumping the heap" }
    dismissRetainedCountNotification()
    // 获取内存快照
    dumpHeap(retainedReferenceCount, retry = true)
}

private fun dumpHeap(
    retainedReferenceCount: Int,
    retry: Boolean
  ) {
	...
    // 获取当前内存快照文件hprof
    val heapDumpFile = heapDumper.dumpHeap()
    // 获取失败处理
    if (heapDumpFile == null) {
      if (retry) {
        SharkLog.d { "Failed to dump heap, will retry in $WAIT_AFTER_DUMP_FAILED_MILLIS ms" }
        scheduleRetainedObjectCheck(
            reason = "failed to dump heap",
            rescheduling = true,
            delayMillis = WAIT_AFTER_DUMP_FAILED_MILLIS
        )
      } else {
        SharkLog.d { "Failed to dump heap, will not automatically retry" }
      }
      showRetainedCountNotification(
          objectCount = retainedReferenceCount,
          contentText = application.getString(
              R.string.leak_canary_notification_retained_dump_failed
          )
      )
      return
    }
    lastDisplayedRetainedObjectCount = 0
    lastHeapDumpUptimeMillis = SystemClock.uptimeMillis()
    // 清理注册的监听
    objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
    // 开启服务分析hprof文件，即解析生成报告
    HeapAnalyzerService.runAnalysis(application, heapDumpFile)
  }
```

经过一系列调用，最后会执行到checkRetainedObjects方法，在该方法中先记录未回收对象的数量，然后主动GC一次，并更新该数量，如果未超过阈值5个就返回，否则就每隔60s执行一次，并通过dumpHeap获取内存快照。该方法内会调用AndroidHeapDumper的dumpHeap方法

```kotlin
// AndroidHeapDumper.kt
override fun dumpHeap(): File? {
    val heapDumpFile = leakDirectoryProvider.newHeapDumpFile() ?: return null

    val waitingForToast = FutureResult<Toast?>()
    showToast(waitingForToast)

    if (!waitingForToast.wait(5, SECONDS)) {
        SharkLog.d { "Did not dump heap, too much time waiting for Toast." }
        return null
    }

    val notificationManager =
    context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    if (Notifications.canShowNotification) {
        val dumpingHeap = context.getString(R.string.leak_canary_notification_dumping)
        val builder = Notification.Builder(context)
        .setContentTitle(dumpingHeap)
        val notification = Notifications.buildNotification(context, builder, LEAKCANARY_LOW)
        notificationManager.notify(R.id.leak_canary_notification_dumping_heap, notification)
    }

    val toast = waitingForToast.get()

    return try {
        // 写入数据
        Debug.dumpHprofData(heapDumpFile.absolutePath)
        if (heapDumpFile.length() == 0L) {
            SharkLog.d { "Dumped heap file is 0 byte length" }
            null
        } else {
            heapDumpFile
        }
    } catch (e: Exception) {
        SharkLog.d(e) { "Could not dump heap" }
        // Abort heap dump
        null
    } finally {
        cancelToast(toast)
        notificationManager.cancel(R.id.leak_canary_notification_dumping_heap)
    }
}
```

在该方法内会调用leakDirectoryProvider的newHeapDumpFile方法生成hprof文件：

```kotlin
// LeakDirectoryProvider.kt
fun newHeapDumpFile(): File? {
    cleanupOldHeapDumps()

    var storageDirectory = externalStorageDirectory()
    if (!directoryWritableAfterMkdirs(storageDirectory)) {
        if (!hasStoragePermission()) {
            if (requestExternalStoragePermission()) {
                SharkLog.d { "WRITE_EXTERNAL_STORAGE permission not granted, requesting" }
                requestWritePermissionNotification()
            } else {
                SharkLog.d { "WRITE_EXTERNAL_STORAGE permission not granted, ignoring" }
            }
        } else {
            val state = Environment.getExternalStorageState()
            if (Environment.MEDIA_MOUNTED != state) {
                SharkLog.d { "External storage not mounted, state: $state" }
            } else {
                SharkLog.d {
                    "Could not create heap dump directory in external storage: [${storageDirectory.absolutePath}]"
                }
            }
        }
        // Fallback to app storage.
        storageDirectory = appStorageDirectory()
        if (!directoryWritableAfterMkdirs(storageDirectory)) {
            SharkLog.d {
                "Could not create heap dump directory in app storage: [${storageDirectory.absolutePath}]"
            }
            return null
        }
    }

    val fileName = SimpleDateFormat("yyyy-MM-dd_HH-mm-ss_SSS'.hprof'", Locale.US).format(Date())
    // 创建hprof文件
    return File(storageDirectory, fileName)
}
```

紧接着我们看下hprof文件解析：

```kotlin
// HeapAnalyzerService.kt
internal class HeapAnalyzerService : ForegroundService(
    HeapAnalyzerService::class.java.simpleName,
    R.string.leak_canary_notification_analysing,
    R.id.leak_canary_notification_analyzing_heap
), OnAnalysisProgressListener {

    override fun onHandleIntentInForeground(intent: Intent?) {
        if (intent == null || !intent.hasExtra(HEAPDUMP_FILE_EXTRA)) {
            SharkLog.d { "HeapAnalyzerService received a null or empty intent, ignoring." }
            return
        }

        // Since we're running in the main process we should be careful not to impact it.
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)
        // 获取到hprof文件
        val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File

        val config = LeakCanary.config
        val heapAnalysis = if (heapDumpFile.exists()) {
            // 解析hprof文件
            analyzeHeap(heapDumpFile, config)
        } else {
            missingFileFailure(heapDumpFile)
        }
        onAnalysisProgress(REPORTING_HEAP_ANALYSIS)
        // 解析完成回调
        config.onHeapAnalyzedListener.onHeapAnalyzed(heapAnalysis)
    }

    private fun analyzeHeap(
        heapDumpFile: File,
        config: Config
    ): HeapAnalysis {
        val heapAnalyzer = HeapAnalyzer(this)

        val proguardMappingReader = try {
            ProguardMappingReader(assets.open(PROGUARD_MAPPING_FILE_NAME))
        } catch (e: IOException) {
            null
        }
        return heapAnalyzer.analyze(
            heapDumpFile = heapDumpFile,
            leakingObjectFinder = config.leakingObjectFinder,
            referenceMatchers = config.referenceMatchers,
            computeRetainedHeapSize = config.computeRetainedHeapSize,
            objectInspectors = config.objectInspectors,
            metadataExtractor = config.metadataExtractor,
            proguardMapping = proguardMappingReader?.readProguardMapping()
        )
    }

    private fun missingFileFailure(heapDumpFile: File): HeapAnalysisFailure {
        val deletedReason = LeakDirectoryProvider.hprofDeleteReason(heapDumpFile)
        val exception = IllegalStateException(
            "Hprof file $heapDumpFile missing, deleted because: $deletedReason"
        )
        return HeapAnalysisFailure(
            heapDumpFile = heapDumpFile,
            createdAtTimeMillis = System.currentTimeMillis(),
            analysisDurationMillis = 0,
            exception = HeapAnalysisException(exception)
        )
    }

    override fun onAnalysisProgress(step: OnAnalysisProgressListener.Step) {
        val percent =
        (100f * step.ordinal / OnAnalysisProgressListener.Step.values().size).toInt()
        SharkLog.d { "Analysis in progress, working on: ${step.name}" }
        val lowercase = step.name.replace("_", " ")
        .toLowerCase(Locale.US)
        val message = lowercase.substring(0, 1).toUpperCase(Locale.US) + lowercase.substring(1)
        showForegroundNotification(100, percent, false, message)
    }

    companion object {
        private const val HEAPDUMP_FILE_EXTRA = "HEAPDUMP_FILE_EXTRA"
        private const val PROGUARD_MAPPING_FILE_NAME = "leakCanaryObfuscationMapping.txt"

        fun runAnalysis(
            context: Context,
            heapDumpFile: File
        ) {
            val intent = Intent(context, HeapAnalyzerService::class.java)
            intent.putExtra(HEAPDUMP_FILE_EXTRA, heapDumpFile)
            startForegroundService(context, intent)
        }

        private fun startForegroundService(
            context: Context,
            intent: Intent
        ) {
            if (SDK_INT >= 26) {
                context.startForegroundService(intent)
            } else {
                // Pre-O behavior.
                context.startService(intent)
            }
        }
    }
}
```

它启动了一个前台服务，该服务继承自ForegroundService，而ForegroundService又继承自IntentService:

```kotlin
// ForegroundService.kt
internal abstract class ForegroundService(
  name: String,
  private val notificationContentTitleResId: Int,
  private val notificationId: Int
) : IntentService(name) {

  override fun onCreate() {
    super.onCreate()
    showForegroundNotification(
        max = 100, progress = 0, indeterminate = true,
        contentText = getString(R.string.leak_canary_notification_foreground_text)
    )
  }

  protected fun showForegroundNotification(
    max: Int,
    progress: Int,
    indeterminate: Boolean,
    contentText: String
  ) {
    val builder = Notification.Builder(this)
        .setContentTitle(getString(notificationContentTitleResId))
        .setContentText(contentText)
        .setProgress(max, progress, indeterminate)
    val notification =
      Notifications.buildNotification(this, builder, LEAKCANARY_LOW)
    startForeground(notificationId, notification)
  }

  override fun onHandleIntent(intent: Intent?) {
    onHandleIntentInForeground(intent)
  }

  protected abstract fun onHandleIntentInForeground(intent: Intent?)

  override fun onDestroy() {
    super.onDestroy()
    stopForeground(true)
  }

  override fun onBind(intent: Intent): IBinder? {
    return null
  }
}
```

IntentService内部会初始化一个HandlerThread，即带有looper的线程，在服务启动时，会发送一个消息到与该线程关联的handler，并调用onHandleIntent方法，所以该方法也执行在子线程，进而又调用到HeapAnalyzerService的onHandleIntentInForeground方法，执行analyzeHeap方法解析文件，在该方法内部创建一个HeapAnalyzer对象，通过调用其analyze方法完成解析：

```kotlin
// HeapAnalyzer.kt
fun analyze(
    heapDumpFile: File,
    leakingObjectFinder: LeakingObjectFinder,
    referenceMatchers: List<ReferenceMatcher> = emptyList(),
    computeRetainedHeapSize: Boolean = false,
    objectInspectors: List<ObjectInspector> = emptyList(),
    metadataExtractor: MetadataExtractor = MetadataExtractor.NO_OP,
    proguardMapping: ProguardMapping? = null
): HeapAnalysis {
    val analysisStartNanoTime = System.nanoTime()

    if (!heapDumpFile.exists()) {
        val exception = IllegalArgumentException("File does not exist: $heapDumpFile")
        return HeapAnalysisFailure(
            heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
            HeapAnalysisException(exception)
        )
    }

    return try {
        listener.onAnalysisProgress(PARSING_HEAP_DUMP)
        Hprof.open(heapDumpFile)
        .use { hprof ->
              // 从文件中解析获取对象关系图结构graph并获取图中的所有GC roots根节点
              val graph = HprofHeapGraph.indexHprof(hprof, proguardMapping)
              val helpers =
              FindLeakInput(graph, referenceMatchers, computeRetainedHeapSize, objectInspectors)
              // 查找内存泄漏对象
              helpers.analyzeGraph(
                  metadataExtractor, leakingObjectFinder, heapDumpFile, analysisStartNanoTime
              )
             }
    } catch (exception: Throwable) {
        HeapAnalysisFailure(
            heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
            HeapAnalysisException(exception)
        )
    }
}

private fun FindLeakInput.analyzeGraph(
    metadataExtractor: MetadataExtractor,
    leakingObjectFinder: LeakingObjectFinder,
    heapDumpFile: File,
    analysisStartNanoTime: Long
): HeapAnalysisSuccess {
    listener.onAnalysisProgress(EXTRACTING_METADATA)
    val metadata = metadataExtractor.extractMetadata(graph)

    listener.onAnalysisProgress(FINDING_RETAINED_OBJECTS)
    // 通过过滤graph中的KeyedWeakReference类型对象来找到对应的内存泄漏对象
    val leakingObjectIds = leakingObjectFinder.findLeakingObjectIds(graph)

    val (applicationLeaks, libraryLeaks) = findLeaks(leakingObjectIds)

    return HeapAnalysisSuccess(
        heapDumpFile = heapDumpFile,
        createdAtTimeMillis = System.currentTimeMillis(),
        analysisDurationMillis = since(analysisStartNanoTime),
        metadata = metadata,
        applicationLeaks = applicationLeaks,
        libraryLeaks = libraryLeaks
    )
}
```

在该方法实现了解析hprof文件找到内存泄漏对象，并计算对象到GC roots的最短路径，输出报告。

上面通过调用KeyedWeakReferenceFinder的findLeakingObjectIds方法过滤出泄露对象，再通过findLeaks方法计算到GC Roots的路径：

```kotlin
// HeapAnalyzer.kt
private fun FindLeakInput.findLeaks(leakingObjectIds: Set<Long>): Pair<List<ApplicationLeak>, List<LibraryLeak>> {
    val pathFinder = PathFinder(graph, listener, referenceMatchers)
    // 计算并获取目标对象到GC roots的最短路径
    val pathFindingResults =
    pathFinder.findPathsFromGcRoots(leakingObjectIds, computeRetainedHeapSize)

    SharkLog.d { "Found ${leakingObjectIds.size} retained objects" }
	// 将这些内存泄漏对象的最短路径合并成树结构返回
    return buildLeakTraces(pathFindingResults)
}
```

最后通过可视化界面将hprof分析结果HeapAnalysisSuccess展示出来。

而解析完成的回调方法是onHeapAnalyzedListener.onHeapAnalyzed，它的默认实现类是DefaultOnHeapAnalyzedListener，源码如下：

```kotlin
// DefaultOnHeapAnalyzedListener.kt
override fun onHeapAnalyzed(heapAnalysis: HeapAnalysis) {
    SharkLog.d { "$heapAnalysis" }

    val id = LeaksDbHelper(application).writableDatabase.use { db ->
        HeapAnalysisTable.insert(db, heapAnalysis)
                                                             }

    val (contentTitle, screenToShow) = when (heapAnalysis) {
        is HeapAnalysisFailure -> application.getString(
            R.string.leak_canary_analysis_failed
        ) to HeapAnalysisFailureScreen(id)
            is HeapAnalysisSuccess -> {
            val retainedObjectCount = heapAnalysis.allLeaks.sumBy { it.leakTraces.size }
            val leakTypeCount = heapAnalysis.applicationLeaks.size + heapAnalysis.libraryLeaks.size
                application.getString(
                R.string.leak_canary_analysis_success_notification, retainedObjectCount, leakTypeCount
            ) to HeapDumpScreen(id)
        }
    }

    if (InternalLeakCanary.formFactor == TV) {
        // toast展示
        showToast(heapAnalysis)
            printIntentInfo()
    } else {
        // 通知展示
        showNotification(screenToShow, contentTitle)
    }
}
```

## 总结

首先注册监听Activity的生命周期onDestory，然后在onDestory方法中，通过ObjectWatcher来watch该Activity,在该方法中，创建带有标识的KeyedWeakReference对象，并关联ReferenceQueue，并存在监控集合Map中，延时5秒来检查目标对象是否回收，如果有未回收的对象则主动触发GC，并每60s检查一次，满足5个阈值，就开始dump heap获取内存快照文件，并解析该文件，找到内存泄露对象，然后计算该对象到GC Roots的最短路径，合并所有路径为树结构返回，最后以可视化的方式展示界面。