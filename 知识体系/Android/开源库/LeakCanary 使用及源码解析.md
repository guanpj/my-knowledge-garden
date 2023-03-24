---
title: LeakCanary 使用及源码解析
tags:
 - LeakCanary
 - 源码解析
date created: 2023-03-23
date modified: 2023-03-24
---

## Reference 介绍

Reference 即是我们平时所说的“引用”，与之对应的是一个泛型抽象类。四种引用类型：SoftReference(软引用)、WeakReference(弱引用)、PhantomReference（虚引用）都继承自 Reference。它的声明如下：

```java
public abstract class Reference<T> {
    //引用对象
    volatile T referent;
    //保存即将被回收的Reference对象
    final ReferenceQueue<? super T> queue;
    
    //在Enqueued状态下即引用加入队列时，指向下一个待处理Reference对象,默认为null
    Reference queueNext;
    //在Pending状态下，待入列引用，默认为null
    Reference<?> pendingNext;
    
    ...
}
```

Reference 有四种状态：Active、Pending、Enqueued、Inactive，默认为 Active 状态。

ReferenceQueue 则是一个单向链表实现的队列数据结构，存储的是 Reference 对象。包含了入列 enqueue、出列 poll 和移除 remove 操作。

## ReferenceQueue 原理和使用示例

Reference 配合 ReferenceQueue 可以实现对象回收监听，使用方法如下：

```java
//创建一个引用队列
ReferenceQueue queue = new ReferenceQueue();
//创建对象
Object object = new Object();
//创建 object 对象的弱引用，并关联引用队列 queue
WeakReference reference = new WeakReference(object, queue);
System.out.println(reference);
System.gc();
//当 reference 被成功回收后，可以从 queue 中获取到该引用
System.out.println(queue.remove());

输出结果：
java.lang.ref.WeakReference@3d833955
java.lang.ref.WeakReference@3d833955
```

示例中的对象当然是可以正常回收的，所以回收后可以在关联的引用队列 queue 中获取到该引用。反之，若某个应该被回收的对象，GC 结束后在 queue 中未找到该引用，则表明该引用存在内存泄漏风险，这也就是 LeakCanary 的基本原理。

# LeakCanary 基本使用

## 2.0 之前

导入依赖：

```plain text
debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.1'
releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.1'
debugImplementation 'com.squareup.leakcanary:leakcanary-support-fragment:1.6.1'
```

在项目的自定义 Application 中进行初始化：

```kotlin
private fun initLeakCanary(){
    if (LeakCanary.isInAnalyzerProcess(this)) {
        return
    }
    LeakCanary.install(this)
}
```

## 2.0 开始

LeakCanary 从 2.0 开始采用 Kotlin 编写，并且只需要导入依赖即可，不用再手动进行初始化操作：

```plain text
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.7'
```

并且 Leakcanary 2.0 版本还增加了 shark 模块，用于 dumpFile 分析，相比于之前的方法大幅减少了内存的占用。

接下来的分析流程也是基于 2.7 版本进行的。

# LeakCanary 原理分析

## 初始化

既然不用手动初始化，那必定有自动初始化的入口。在 leakcanary-object-watcher-android 模块的 [Manifest 文件](https://github.com/square/leakcanary/blob/main/leakcanary-object-watcher-android/src/main/AndroidManifest.xml) 中声明了一个 Provider 组件：

```xml
<application>
	<provider
	    android:name="leakcanary.internal.AppWatcherInstaller$MainProcess"
	    android:authorities="${applicationId}.leakcanary-installer"
	    android:enabled="@bool/leak_canary_watcher_auto_install"
	    android:exported="false"/>
</application>
```

顺蔓摸瓜，查看 AppWatcherInstaller：

```kotlin
override fun onCreate(): Boolean {
  val application = context!!.applicationContext as Application
  AppWatcher.manualInstall(application)
  return true
}
```

果然找到了初始化方法，接着跟踪 AppWatcher 类：

```kotlin
@JvmOverloads
fun manualInstall(
  application: Application,
  retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
  watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
) {
  //1、确保在主线程，否则抛出UnsupportedOperationException异常
  checkMainThread()
  if (isInstalled) {
    throw IllegalStateException(
      "AppWatcher already installed, see exception cause for prior install call", installCause
    )
  }
  check(retainedDelayMillis >= 0) {
    "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
  }
  installCause = RuntimeException("manualInstall() first called here")
  this.retainedDelayMillis = retainedDelayMillis
  if (application.isDebuggableBuild) {
    LogcatSharkLog.install()
  }
  //2、Requires AppWatcher.objectWatcher to be set
  LeakCanaryDelegate.loadLeakCanary(application)
  //3、初始化 Activity 和 Fragment 监听器
  watchersToInstall.forEach {
    it.install()
  }
}
```

跟踪注释 2 处：

```kotlin
internal object LeakCanaryDelegate {

  //使用 lazy 实现懒汉单例
  @Suppress("UNCHECKED_CAST")
  val loadLeakCanary by lazy {
    try {
      //反射调用 InternalLeakCanary
      val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
      leakCanaryListener.getDeclaredField("INSTANCE")
        .get(null) as (Application) -> Unit
    } catch (ignored: Throwable) {
      NoLeakCanary
    }
  }

  object NoLeakCanary : (Application) -> Unit, OnObjectRetainedListener {
    override fun invoke(application: Application) {
    }

    override fun onObjectRetained() {
    }
  }
}
```

接着查看 InternalLeakCanary 的 invoke 方法：

```kotlin
override fun invoke(application: Application) {
  _application = application

  checkRunningInDebuggableBuild()

  //为 objectWatcher 对象添加 this 作为对象保留监听器
  AppWatcher.objectWatcher.addOnObjectRetainedListener(this)
  //初始化 AndroidHeapDumper
  val heapDumper = AndroidHeapDumper(application, createLeakDirectoryProvider(application))
  //GcTrigger
  val gcTrigger = GcTrigger.Default

  val configProvider = { LeakCanary.config }

  //初始化 backgroundHandler
  val handlerThread = HandlerThread(LEAK_CANARY_THREAD_NAME)
  handlerThread.start()
  val backgroundHandler = Handler(handlerThread.looper)

  //初始化 HeapDumpTrigger
  heapDumpTrigger = HeapDumpTrigger(
    application, backgroundHandler, AppWatcher.objectWatcher, gcTrigger, heapDumper,
    configProvider
  )
  
  //监听 app 进入后台/返回前台事件
  application.registerVisibilityListener { applicationVisible ->
    this.applicationVisible = applicationVisible
    heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
  }
  registerResumedActivityListener(application)
  addDynamicShortcut(application)

  // We post so that the log happens after Application.onCreate()
  mainHandler.post {
    // https://github.com/square/leakcanary/issues/1981
    // We post to a background handler because HeapDumpControl.iCanHasHeap() checks a shared pref
    // which blocks until loaded and that creates a StrictMode violation.
    backgroundHandler.post {
      SharkLog.d {
        when (val iCanHasHeap = HeapDumpControl.iCanHasHeap()) {
          is Yup -> application.getString(R.string.leak_canary_heap_dump_enabled_text)
          is Nope -> application.getString(
            R.string.leak_canary_heap_dump_disabled_text, iCanHasHeap.reason()
          )
        }
      }
    }
  }
}
```

至此，InternalLeakCanary 初始化完成。回到 manualInstall 方法中继续查看注释 3 处代码：

```kotlin
watchersToInstall.forEach {
  it.install()
}
watchersToInstall 来自 appDefaultWatchers 方法：
fun appDefaultWatchers(
  application: Application,
  reachabilityWatcher: ReachabilityWatcher = objectWatcher
): List<InstallableWatcher> {
  return listOf(
    ActivityWatcher(application, reachabilityWatcher),
    FragmentAndViewModelWatcher(application, reachabilityWatcher),
    RootViewWatcher(reachabilityWatcher),
    ServiceWatcher(reachabilityWatcher)
  )
}
```

以 ActivityWatcher 为例：

```kotlin
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {
  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }
    
  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }
  
  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}
```

install 方法将 lifecycleCallbacks 注册为 application 的 ActivityLifecycleCallbacks，用来感知所有 Activity 的声明周期。

by noOpDelegate() 通过类委托机制将其他回调实现都交给 noOpDelegate，而 noOpDelegate 是一个空实现的动态代理。这里只需要监听 Activity 销毁事件，因此只需要重写 onActivityDestroyed 即可。

## Activity 回收监听

在 onActivityDestroyed 方法被回调时，调用了 reachabilityWatcher 的 expectWeaklyReachable 方法并将 activity 对象传进去。这里 reachabilityWatcher 的唯一实现类为 ObjectWatcher，查看它的 expectWeaklyReachable 方法内容：

```kotlin
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  if (!isEnabled()) {
    return
  }
  //将 ReferenceQueue 中出现的弱引用移除，即忽略已被回收的 Activity
  removeWeaklyReachableObjects()
  //生成随机的 key 值
  val key = UUID.randomUUID().toString()
  //记录时间
  val watchUptimeMillis = clock.uptimeMillis()
  //将 Activity 对象（watchedObject）封装成 KeyedWeakReference
  //并关联引用队列 queue
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  SharkLog.d {
    "Watching " +
      (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
      (if (description.isNotEmpty()) " ($description)" else "") +
      " with key $key"
  }

  //将弱引用 reference 存入监听 map 集合
  watchedObjects[key] = reference
  //5 秒之后在主线程执行 moveToRetained(key) 方法
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}

private fun removeWeaklyReachableObjects() {
  // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
  // reachable. This is before finalization or garbage collection has actually happened.
  var ref: KeyedWeakReference?
  do {
    //从 queue 中取出对象
    ref = queue.poll() as KeyedWeakReference?
    if (ref != null) {
      //将该对象从集合中移除
      watchedObjects.remove(ref.key)
    }
  } while (ref != null)
}
```

为什么会是 5 秒，这里猜测与 Android GC 有关。在 Activity.H 中，收到 GC_WHEN_IDLE 消息时会进行 Looper.myQueue().addIdleHandler(mGcIdler)，而 mGcIdler 最后会触发 doGcIfNeeded 操作，在该方法中会判断上次 GC 与现在时间的差值，而这个值就是 MIN_TIME_BETWEEN_GCS = 5*1000。

查看 moveToRetained 方法：

```kotlin
@Synchronized private fun moveToRetained(key: String) {
  //再次移除已被回收的对象对应的弱引用 Reference
  removeWeaklyReachableObjects()
  val retainedRef = watchedObjects[key]
  //如果用这个 key 对应的 Reference 没有被移除，说明已经发生泄漏
  if (retainedRef != null) {
    retainedRef.retainedUptimeMillis = clock.uptimeMillis()
    //通知 listener
    onObjectRetainedListeners.forEach { it.onObjectRetained() }
  }
}
```

在经过 5s 后，再次移除被回收的对象对应的 Reference。然后再判断 watchedObjects 集合中是否是否仍然存在该 key 对应的 value，如果存在则认为该 value 对应的对象发生了泄露。随后记录下发生时间，并通知 listener 发生对象残留情况。

这里的 listener 就是 InternalLeakCanary 的 invoke 方法中设置的 InternalLeakCanary.this 对象，查看 InternalLeakCanary 的 onObjectRetained 方法：

```kotlin
override fun onObjectRetained() = scheduleRetainedObjectCheck()
继续跟进：
fun scheduleRetainedObjectCheck() {
  if (this::heapDumpTrigger.isInitialized) {
    heapDumpTrigger.scheduleRetainedObjectCheck()
  }
}
继续追踪 HeapDumpTrigger 的 scheduleRetainedObjectCheck 方法：
fun scheduleRetainedObjectCheck(
  delayMillis: Long = 0L
) {
  val checkCurrentlyScheduledAt = checkScheduledAt
  if (checkCurrentlyScheduledAt > 0) {
    return
  }
  checkScheduledAt = SystemClock.uptimeMillis() + delayMillis
  //将后面的流程 post 到后台线程执行
  backgroundHandler.postDelayed({
    checkScheduledAt = 0
    checkRetainedObjects()
  }, delayMillis)
}
查看 checkretainedObjects 方法：
private fun checkRetainedObjects() {
  val iCanHasHeap = HeapDumpControl.iCanHasHeap()

  val config = configProvider()

  if (iCanHasHeap is Nope) {
    ...
    return
  }
  //获取没有被回收对象的个数
  var retainedReferenceCount = objectWatcher.retainedObjectCount
  //如果没有被回收的对象个数大于 0
  if (retainedReferenceCount > 0) {
    //执行一次GC
    gcTrigger.runGc()
    //再次获取没有被回收对象的个数
    retainedReferenceCount = objectWatcher.retainedObjectCount
  }
  //检查没有被回收对象的个数，如果少于 5 个则返回
  if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

  val now = SystemClock.uptimeMillis()
  val elapsedSinceLastDumpMillis = now - lastHeapDumpUptimeMillis
  //WAIT_BETWEEN_HEAP_DUMPS_MILLIS 为 60_000L
  //即 60 秒内再次发现泄漏只会发出通知并返回
  if (elapsedSinceLastDumpMillis < WAIT_BETWEEN_HEAP_DUMPS_MILLIS) {
    onRetainInstanceListener.onEvent(DumpHappenedRecently)
    showRetainedCountNotification(
      objectCount = retainedReferenceCount,
      contentText = application.getString(R.string.leak_canary_notification_retained_dump_wait)
    )
    scheduleRetainedObjectCheck(
      delayMillis = WAIT_BETWEEN_HEAP_DUMPS_MILLIS - elapsedSinceLastDumpMillis
    )
    return
  }

  dismissRetainedCountNotification()
  val visibility = if (applicationVisible) "visible" else "not visible"
  //获取内存快照，即.hprof文件
  dumpHeap(
    retainedReferenceCount = retainedReferenceCount,
    retry = true,
    reason = "$retainedReferenceCount retained objects, app is $visibility"
  )
}
```

## 获取 dumpHeap 文件

继续查看：

```kotlin
private fun dumpHeap(
  retainedReferenceCount: Int,
  retry: Boolean,
  reason: String
) {
  saveResourceIdNamesToMemory()
  val heapDumpUptimeMillis = SystemClock.uptimeMillis()
  KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
  //调用 AndroidHeapDumper 的 dumpHeap 方法
  when (val heapDumpResult = heapDumper.dumpHeap()) {
    is NoHeapDump -> {
      if (retry) {
        SharkLog.d { "Failed to dump heap, will retry in $WAIT_AFTER_DUMP_FAILED_MILLIS ms" }
        scheduleRetainedObjectCheck(
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
    }
    is HeapDump -> {
      lastDisplayedRetainedObjectCount = 0
      lastHeapDumpUptimeMillis = SystemClock.uptimeMillis()
      //清理 objectWatcher 中 heapDumpUptimeMillis 之前保存的键值对
      objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
      //分析 dump 文件
      HeapAnalyzerService.runAnalysis(
        context = application,
        heapDumpFile = heapDumpResult.file,
        heapDumpDurationMillis = heapDumpResult.durationMillis,
        heapDumpReason = reason
      )
    }
  }
}

@Synchronized fun clearObjectsWatchedBefore(heapDumpUptimeMillis: Long) {
  val weakRefsToRemove =
    watchedObjects.filter { it.value.watchUptimeMillis <= heapDumpUptimeMillis }
  weakRefsToRemove.values.forEach { it.clear() }
  watchedObjects.keys.removeAll(weakRefsToRemove.keys)
}
```

## heapDump 文件解析

继续查看 HeapAnalyzerService 类：

```kotlin
fun runAnalysis(
  context: Context,
  heapDumpFile: File,
  heapDumpDurationMillis: Long? = null,
  heapDumpReason: String = "Unknown"
) {
  val intent = Intent(context, HeapAnalyzerService::class.java)
  intent.putExtra(HEAPDUMP_FILE_EXTRA, heapDumpFile)
  intent.putExtra(HEAPDUMP_REASON_EXTRA, heapDumpReason)
  heapDumpDurationMillis?.let {
    intent.putExtra(HEAPDUMP_DURATION_MILLIS_EXTRA, heapDumpDurationMillis)
  }
  //启动一个 Service，并将 heapDump 文件信息传入进去
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
```

启动 HeapAnalyzerService 后：

```kotlin
override fun onHandleIntentInForeground(intent: Intent?) {
  if (intent == null || !intent.hasExtra(HEAPDUMP_FILE_EXTRA)) {
    SharkLog.d { "HeapAnalyzerService received a null or empty intent, ignoring." }
    return
  }

  // Since we're running in the main process we should be careful not to impact it.
  Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)
  val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File
  val heapDumpReason = intent.getStringExtra(HEAPDUMP_REASON_EXTRA)
  val heapDumpDurationMillis = intent.getLongExtra(HEAPDUMP_DURATION_MILLIS_EXTRA, -1)

  val config = LeakCanary.config
  val heapAnalysis = if (heapDumpFile.exists()) {
    // 分析heapDump 文件
    analyzeHeap(heapDumpFile, config)
  } else {
    missingFileFailure(heapDumpFile)
  }
  val fullHeapAnalysis = when (heapAnalysis) {
    is HeapAnalysisSuccess -> heapAnalysis.copy(
      dumpDurationMillis = heapDumpDurationMillis,
      metadata = heapAnalysis.metadata + ("Heap dump reason" to heapDumpReason)
    )
    is HeapAnalysisFailure -> heapAnalysis.copy(dumpDurationMillis = heapDumpDurationMillis)
  }
  onAnalysisProgress(REPORTING_HEAP_ANALYSIS)
  //将分析结果回调给 onHeapAnalyzedListener
  config.onHeapAnalyzedListener.onHeapAnalyzed(fullHeapAnalysis)
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
```

查看 heapAnalyzer.analyzeHeap 方法：

```kotlin
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
      heapDumpFile = heapDumpFile,
      createdAtTimeMillis = System.currentTimeMillis(),
      analysisDurationMillis = since(analysisStartNanoTime),
      exception = HeapAnalysisException(exception)
    )
  }

  return try {
    listener.onAnalysisProgress(PARSING_HEAP_DUMP)
    val sourceProvider = ConstantMemoryMetricsDualSourceProvider(FileSourceProvider(heapDumpFile))
    //从文件中解析获取对象关系图结构 graph
    //并获取图中的所有 GC roots 根节点
    sourceProvider.openHeapGraph(proguardMapping).use { graph ->
      //创建 FindLeakInput 对象
      val helpers =
        FindLeakInput(graph, referenceMatchers, computeRetainedHeapSize, objectInspectors)
       //查找内存泄漏对象
       val result = helpers.analyzeGraph(
        metadataExtractor, leakingObjectFinder, heapDumpFile, analysisStartNanoTime
      )
      val lruCacheStats = (graph as HprofHeapGraph).lruCacheStats()
      val randomAccessStats =
        "RandomAccess[" +
          "bytes=${sourceProvider.randomAccessByteReads}," +
          "reads=${sourceProvider.randomAccessReadCount}," +
          "travel=${sourceProvider.randomAccessByteTravel}," +
          "range=${sourceProvider.byteTravelRange}," +
          "size=${heapDumpFile.length()}" +
          "]"
      val stats = "$lruCacheStats $randomAccessStats"
      result.copy(metadata = result.metadata + ("Stats" to stats))
    }
  } catch (exception: Throwable) {
    HeapAnalysisFailure(
      heapDumpFile = heapDumpFile,
      createdAtTimeMillis = System.currentTimeMillis(),
      analysisDurationMillis = since(analysisStartNanoTime),
      exception = HeapAnalysisException(exception)
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

  //通过过滤 graph 中的 KeyedWeakReference 类型对象来
  //找到对应的内存泄漏对象
  val retainedClearedWeakRefCount = KeyedWeakReferenceFinder.findKeyedWeakReferences(graph)
    .filter { it.isRetained && !it.hasReferent }.count()

  // This should rarely happens, as we generally remove all cleared weak refs right before a heap
  // dump.
  val metadataWithCount = if (retainedClearedWeakRefCount > 0) {
    metadata + ("Count of retained yet cleared" to "$retainedClearedWeakRefCount KeyedWeakReference instances")
  } else {
    metadata
  }

  listener.onAnalysisProgress(FINDING_RETAINED_OBJECTS)
  val leakingObjectIds = leakingObjectFinder.findLeakingObjectIds(graph)
  //计算内存泄漏对象到 GC roots 的路径
  val (applicationLeaks, libraryLeaks, unreachableObjects) = findLeaks(leakingObjectIds)

  return HeapAnalysisSuccess(
    heapDumpFile = heapDumpFile,
    createdAtTimeMillis = System.currentTimeMillis(),
    analysisDurationMillis = since(analysisStartNanoTime),
    metadata = metadataWithCount,
    applicationLeaks = applicationLeaks,
    libraryLeaks = libraryLeaks,
    unreachableObjects = unreachableObjects
  )
}

private fun FindLeakInput.findLeaks(leakingObjectIds: Set<Long>): LeaksAndUnreachableObjects {
  val pathFinder = PathFinder(graph, listener, referenceMatchers)
  //计算并获取目标对象到 GC roots 的最短路径
  val pathFindingResults =
    pathFinder.findPathsFromGcRoots(leakingObjectIds, computeRetainedHeapSize)

  val unreachableObjects = findUnreachableObjects(pathFindingResults, leakingObjectIds)

  val shortestPaths =
    deduplicateShortestPaths(pathFindingResults.pathsToLeakingObjects)

  val inspectedObjectsByPath = inspectObjects(shortestPaths)

  val retainedSizes =
    if (pathFindingResults.dominatorTree != null) {
      computeRetainedSizes(inspectedObjectsByPath, pathFindingResults.dominatorTree)
    } else {
      null
    }
  val (applicationLeaks, libraryLeaks) = buildLeakTraces(
    shortestPaths, inspectedObjectsByPath, retainedSizes
  )
  return LeaksAndUnreachableObjects(applicationLeaks, libraryLeaks, unreachableObjects)
}
```

## 展示通知

回到 HeapAnalyzerService 中，analyzeHeap 方法得到分析结果后，会将结果回调给 onHeapAnalyzedListener。

这里 onHeapAnalyzedListener 的唯一实现类为 DefaultOnHeapAnalyzedListener：

```kotlin
override fun onHeapAnalyzed(heapAnalysis: HeapAnalysis) {
  SharkLog.d { "\u200B\n${LeakTraceWrapper.wrap(heapAnalysis.toString(), 120)}" }

  val db = LeaksDbHelper(application).writableDatabase
  val id = HeapAnalysisTable.insert(db, heapAnalysis)
  db.releaseReference()

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
    showToast(heapAnalysis)
    printIntentInfo()
  } else {
    //展示通知
    showNotification(screenToShow, contentTitle)
  }
}
```

最后将结果通过通知的方式展示出来：

```kotlin
private fun showNotification(
  screenToShow: Screen,
  contentTitle: String
) {
  val pendingIntent = LeakActivity.createPendingIntent(
    application, arrayListOf(HeapDumpsScreen(), screenToShow)
  )

  val contentText = application.getString(R.string.leak_canary_notification_message)

  Notifications.showNotification(
    application, contentTitle, contentText, pendingIntent,
    R.id.leak_canary_notification_analysis_result,
    LEAKCANARY_MAX
  )
}
```

## 总结

总结下 LeakCanary 对于 Activity 内存泄漏分析过程：

1. 初始化 LeakCanary 需要的对象
2. 注册监听 Activity 生命周期 onDestroy 事件
3. 在 Activity 的 onDestroy 事件回调后，创建 KeyedWeakReference 对象，并关联 ReferenceQueue
4. 延时 5 秒检查目标对象是否回收
5. 未回收则开启服务，dump heap 获取内存快照 hprof 文件
6. 解析 hprof 文件根据 KeyedWeakReference 类型过滤找到内存泄漏对象
7. 计算对象到 GC roots 的最短路径，并合并所有最短路径为一棵树
8. 输出分析结果，并根据分析结果通过通知的方式展示出来

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/LeakCanary/clipboard_20230323_035231.png)

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/LeakCanary/clipboard_20230323_035234.png)
