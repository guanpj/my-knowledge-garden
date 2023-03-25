---
title: EventBus 使用及源码解析
tags:
 - EventBus
 - 源码解析
date created: 2023-03-24
date modified: 2023-03-24
---

# 使用

1. 首先引入依赖

```groovy
apply plugin: 'kotlin-kapt'

dependencies {
    implementation 'org.greenrobot:eventbus:3.2.0'
    kapt 'org.greenrobot:eventbus-annotation-processor:3.2.0'
}

kapt {
    arguments {
        arg('eventBusIndex', 'com.me.guanpj.myapplication.MyEventBusIndex')
    }
}
```

从在 3.0 版本开始，EventBus 提供了一个 EventBusAnnotationProcessor 注解处理器来在编译期通过读取 @Subscribe 注解，并解析和处理其中所包含的信息，然后生成 Java 类索引来保存订阅者中所有的事件响应函数，这样就比在运行时使用反射来获得订阅者中所有事件响应函数的速度要快。

以下是来自官方对 EventBus 各个版本性能的对比图，可以看到，EventBus 3.x 如果没有使用索引的话性能相较于之前的版本是倒退的。使用索引能让 EventBus 的性能大大增加。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/EventBus/clipboard_20230323_031615.png)

2. 添加混淆规则：

```plain text
-keepattributes *Annotation*
-keepclassmembers class * {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }
 # And if you use AsyncExecutor:
-keepclassmembers class * extends org.greenrobot.eventbus.util.ThrowableFailureEvent {
    <init>(java.lang.Throwable);
}
```

3. 使用索引文件创建全局 EventBus 实例

经过以上配置，就可以使用生成的 `com.me.guanpj.myapplication.MyEventBusIndex` 创建 EventBus 了。

> 注意：如果项目中没有被 @Subscribe 标记的事件接收方法， 则不会生成索引类。

推荐在 Application 中使用生成的索引创建全局 EventBus 实例：

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        EventBus.builder().addIndex(MyEventBusIndex()).installDefaultEventBus()
    }
}
```

这样后面通过 `EventBus.getDefault()` 获取的 EventBus 实例都是刚才通过索引创建的。

如果既有 Library 的索引，也有 App 的索引，可以在 EventBus 设置过程中一起添加：

```kotlin
EventBus.builder().addIndex(MyEventBusAppIndex())
    .addIndex(MyEventBusLibIndex()).build()
    .installDefaultEventBus()
```

当然，如果在某些情况下不想使用全局实例，也可以单独生成一个实例：

```kotlin

val eventBus = EventBus.builder()
    // 事件接收对象是否有层次结构（默认为 true，超类也将被通知）
    .eventInheritance(true)
    // 忽略索引文件
    .ignoreGeneratedIndex(false)
    // 打印没有订阅消息，默认为 true
    .logNoSubscriberMessages(true)
    // 打印订阅异常的消息，默认为 true
    .logSubscriberExceptions(false)
    // 设置发送的的事件在没有订阅者的情况时，EventBus是否保持静默，默认为 true
    .sendNoSubscriberEvent(true)
    // 发送分发事件的异常，默认为 true
    .sendSubscriberExceptionEvent(true)
    //如果 onEvent*** 方法出现异常，是否将此异常分发给订阅者，默认为 false
    .throwSubscriberException(BuildConfig.DEBUG)
    // 定义一个线程池用于处理后台线程和异步线程分发事件
    .executorService(Executors.newSingleThreadExecutor())
    // 启用严格的方法验证（在 3.0 以前，接收处理事件的方法名必须以 onEvent 开头）
    // 默认为 false
    .strictMethodVerification(true)
    // 为特定时间跳过方法验证，包括方法名、限定符
    .skipMethodVerificationFor(ButtonEvent::class.java)
    .build()
eventBus.register(this)
```

4. 定义事件

新建一个实体类用于事件的传递：

```kotlin
package com.me.guanpj.myapplication.event

data class MessageEvent(val message: String)
```

5. 发送和接收事件

在需要接收事件的 Activity、Fragment 或者其它地方调用一次 `EventBus.getDefault().register(this)` 进行注册，然后使用 `@Subscribe` 注解标记接收事件的方法，方法参数为一个实体类，或者基本数据类型。

```kotlin
class EventBusActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_event)

        EventBus.getDefault().register(this)
    }

    fun buttonClick(view: android.view.View) {
        //EventBus.getDefault().post(MessageEvent("click"))
        EventBus.getDefault().postSticky(MessageEvent("click"))
    }

    fun jump(view: View) {
        startActivity(Intent(this, AnotherActivity::class.java))
    }

    @Subscribe(threadMode = ThreadMode.MAIN, priority = 0)
    fun onEvent(event : MessageEvent) {
        Log.e("gpj", "EventBusActivity onEvent receive msg: ${event.message} in thread: ${Thread.currentThread().name}")
    }

    @Subscribe(threadMode = ThreadMode.BACKGROUND, priority = 1)
    fun onEvent1(event : MessageEvent) {
        Log.e("gpj", "EventBusActivity onEvent1 receive msg: ${event.message} in thread: ${Thread.currentThread().name}")
    }

    @Subscribe(threadMode = ThreadMode.BACKGROUND, priority = 2)
    fun onEvent2(event : MessageEvent) {
        Log.e("gpj", "EventBusActivity onEvent2 receive msg: ${event.message} in thread: ${Thread.currentThread().name}")
    }

    override fun onDestroy() {
        super.onDestroy()
        EventBus.getDefault().unregister(this)
    }
}
```

```kotlin
class AnotherActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_another)

        EventBus.getDefault().register(this)
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    fun onEvent(event : MessageEvent) {
        Log.e("gpj", "AnotherActivity onEvent receive msg: ${event.message} in thread: ${Thread.currentThread().name}")
        // 粘性事件不会被消耗，除非手动移除
        EventBus.getDefault().removeStickyEvent(MessageEvent::class.java)
    }

    override fun onDestroy() {
        super.onDestroy()
        EventBus.getDefault().unregister(this)
    }
}
```

每次输出的日志都是一样的：

```plain text
EventBusActivity onEvent receive msg: click in thread: main
EventBusActivity onEvent2 receive msg: click in thread: pool-2-thread-1
EventBusActivity onEvent1 receive msg: click in thread: pool-2-thread-1
//打开 MainActivity 后
AnotherActivity receive msg: click in thread: main
```

# 分析

## 获取 EventBus 实例

不管是用 `EventBus.getDefault()` 还是 `EventBus.builder()` 都能够获取到 EventBus 对象，只不过前者是获取一个全局的单例，后者是使用 builder() 配置出一个新的对象。

```java
public class EventBus {
    ...
    private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
    
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    private final Map<Object, List<Class<?>>> typesBySubscriber;
    private final Map<Class<?>, Object> stickyEvents;
    ...
    
    public static EventBus getDefault() {
        EventBus instance = defaultInstance;
        if (instance == null) {
            synchronized (EventBus.class) {
                instance = EventBus.defaultInstance;
                if (instance == null) {
                    instance = EventBus.defaultInstance = new EventBus();
                }
            }
        }
        return instance;
    }

    public static EventBusBuilder builder() {
        return new EventBusBuilder();
    }

    public EventBus() {
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        // 1
        subscriptionsByEventType = new HashMap<>();
        // 2
        typesBySubscriber = new HashMap<>();
        // 3
        stickyEvents = new ConcurrentHashMap<>();
        // 4
        mainThreadSupport = builder.getMainThreadSupport();
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        // 5
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
            builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        // 6
        executorService = builder.executorService;
    }
    ...
}
```

可以看出，两种获取对象的方式都会调用到 `EventBus(EventBusBuilder builder)` 这个构造方法，`getDefault()` 只是最终传入了一个空的 `EventBusBuilder` 对象而已。

在 `EventBus(EventBusBuilder builder)` 构造方法中：

在注释 1 处，创建了一个 subscriptionsByEventType 对象，可以看到它是一个 HashMap，并且其 key 表示 Event 的类型，value 为 `CopyOnWriteArrayList<Subscription>`。这里的 Subscription 是一个订阅信息对象，它里面保存了两个重要的字段，一个是类型为 Object 的 subscriber，该字段即为注册的对象（在 Android 中时通常是 Activity 或着 Fragment 对象）；另一个是类型为 SubscriberMethod 的实例，它就是被 @Subscribe 注解的那个订阅方法，里面保存了一个重要的字段：eventType，它的类型为 `Class<?>`，表示 Event 的类型。

在注释 2 处，新建了一个类型为 HashMap 的 typesBySubscriber 对象，它的 key 为 subscriber 对象， value 为 subscriber 对象中所有的 Event 类型 List，日常使用中仅用于判断某个对象是否注册过。

在注释 3 处新建了一个类型为 ConcurrentHashMap 的 stickyEvents 对象，专用于粘性事件处理的，key 同样为 Event 的类型，value 为当前的事件。

> 普通事件是先注册，然后发送事件才能收到；而粘性事件在发送事件之后再订阅也能收到。并且，粘性事件会保存在内存中，每次进入都会去内存中查找获取最新的粘性事件，除非手动移除事件。

在注释 4 处，新建了三个不同类型的事件发送器：

- mainThreadPoster：主线程事件发送器，通过它的 `mainThreadPoster.enqueue(subscription, event)` 方法可以将订阅信息和对应的事件进行入队，然后通过 handler 去发送一个消息，在 handler 的 `handleMessage()` 中去执行方法。
- backgroundPoster：后台事件发送器，通过它的 `enqueue()` 将方法加入到后台的一个队列，最后通过线程池去执行，同时它保证任一时间只且仅能有一个任务会被线程池执行。
- asyncPoster：实现逻辑类似于 backgroundPoster，不同于 backgroundPoster 的保证任一时间只且仅能有一个任务会被线程池执行的特性，asyncPoster 则是异步运行的，可以同时接收多个任务。

再看注释 5 这行代码，这里新建了一个 SubscriberMethodFinder 对象，这是从 EventBus 中抽离出的订阅方法查询的一个对象，在优秀的源码中，我们经常能看到<strong>组合优于继承</strong>的这种实现思想。

在注释 6 处，从 builder 中取出了一个默认的线程池对象，它由 <strong>Executors 的 newCachedThreadPool() 方法创建，它是一个有则用、无则创建、无数量上限</strong>的线程池。

## Index 文件分析

### Index 文件概览

生成的索引类如下：

```java
public class MyEventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        putIndex(new SimpleSubscriberInfo(com.me.guanpj.myapplication.event.EventBusActivity.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onEvent", com.me.guanpj.myapplication.event.MessageEvent.class, ThreadMode.MAIN),
            new SubscriberMethodInfo("onEvent1", com.me.guanpj.myapplication.event.MessageEvent.class,
                    ThreadMode.BACKGROUND, 1, false),
            new SubscriberMethodInfo("onEvent2", com.me.guanpj.myapplication.event.MessageEvent.class,
                    ThreadMode.BACKGROUND, 2, false),
        }));

        putIndex(new SimpleSubscriberInfo(MainActivity.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onEvent3", com.me.guanpj.myapplication.event.MessageEvent.class, ThreadMode.MAIN,
                    3, true),
        }));

    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```

### 类介绍

#### SubscriberInfo 继承关系

可以看到，这里新建了一个 Map 集合，用于存放事件订阅类和它们的订阅方法

`SubscriberInfo` 之间的映射。`SubscriberInfo` 的定义如下：

```java
public interface SubscriberInfo {
    Class<?> getSubscriberClass();

    SubscriberMethod[] getSubscriberMethods();

    SubscriberInfo getSuperSubscriberInfo();

    boolean shouldCheckSuperclass();
}
```

这里 `SubscriberInfo` 的实现类为 `SimpleSubscriberInfo`，而中间还有一个 `AbstractSubscriberInfo`，它们的定义如下：

```java
public abstract class AbstractSubscriberInfo implements SubscriberInfo {
    private final Class subscriberClass;
    private final Class<? extends SubscriberInfo> superSubscriberInfoClass;
    private final boolean shouldCheckSuperclass;

    protected AbstractSubscriberInfo(Class subscriberClass, Class<? extends SubscriberInfo> superSubscriberInfoClass,
                                     boolean shouldCheckSuperclass) {
        this.subscriberClass = subscriberClass;
        this.superSubscriberInfoClass = superSubscriberInfoClass;
        this.shouldCheckSuperclass = shouldCheckSuperclass;
    }

    @Override
    public Class getSubscriberClass() {
        return subscriberClass;
    }

    @Override
    public SubscriberInfo getSuperSubscriberInfo() {
        if(superSubscriberInfoClass == null) {
            return null;
        }
        try {
            return superSubscriberInfoClass.newInstance();
        } catch (InstantiationException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public boolean shouldCheckSuperclass() {
        return shouldCheckSuperclass;
    }

    protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType) {
        return createSubscriberMethod(methodName, eventType, ThreadMode.POSTING, 0, false);
    }

    protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType, 
            ThreadMode threadMode) {
        return createSubscriberMethod(methodName, eventType, threadMode, 0, false);
    }

    protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType,
            ThreadMode threadMode,int priority, boolean sticky) {
        try {
            Method method = subscriberClass.getDeclaredMethod(methodName, eventType);
            return new SubscriberMethod(method, eventType, threadMode, priority, sticky);
        } catch (NoSuchMethodException e) {
            throw new EventBusException("Could not find subscriber method in " + subscriberClass +
                    ". Maybe a missing ProGuard rule?", e);
        }
    }
}

public class SimpleSubscriberInfo extends AbstractSubscriberInfo {

    private final SubscriberMethodInfo[] methodInfos;

    public SimpleSubscriberInfo(Class subscriberClass, boolean shouldCheckSuperclass, SubscriberMethodInfo[] methodInfos) {
        super(subscriberClass, null, shouldCheckSuperclass);
        this.methodInfos = methodInfos;
    }

    @Override
    public synchronized SubscriberMethod[] getSubscriberMethods() {
        int length = methodInfos.length;
        SubscriberMethod[] methods = new SubscriberMethod[length];
        for (int i = 0; i < length; i++) {
            SubscriberMethodInfo info = methodInfos[i];
            methods[i] = createSubscriberMethod(info.methodName, info.eventType, info.threadMode,
                    info.priority, info.sticky);
        }
        return methods;
    }
}
```

#### SubscriberMethod

```java
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, 
            ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }

    @Override
    public boolean equals(Object other) {
        if (other == this) {
            return true;
        } else if (other instanceof SubscriberMethod) {
            checkMethodString();
            SubscriberMethod otherSubscriberMethod = (SubscriberMethod)other;
            otherSubscriberMethod.checkMethodString();
            // Don't use method.equals because of http://code.google.com/p/android/issues/detail?id=7811#c6
            return methodString.equals(otherSubscriberMethod.methodString);
        } else {
            return false;
        }
    }

    private synchronized void checkMethodString() {
        if (methodString == null) {
            // Method.toString has more overhead, just take relevant parts of the method
            StringBuilder builder = new StringBuilder(64);
            builder.append(method.getDeclaringClass().getName());
            builder.append('#').append(method.getName());
            builder.append('(').append(eventType.getName());
            methodString = builder.toString();
        }
    }

    @Override
    public int hashCode() {
        return method.hashCode();
    }
}
```

#### SubscriberMethodInfo

```java
public class SubscriberMethodInfo {
    final String methodName;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;

    public SubscriberMethodInfo(String methodName, Class<?> eventType, 
            ThreadMode threadMode, int priority, boolean sticky) {
        this.methodName = methodName;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }

    public SubscriberMethodInfo(String methodName, Class<?> eventType) {
        this(methodName, eventType, ThreadMode.POSTING, 0, false);
    }

    public SubscriberMethodInfo(String methodName, Class<?> eventType, ThreadMode threadMode) {
        this(methodName, eventType, threadMode, 0, false);
    }
}
```

### SubscriberInfo 映射

回到 `MyEventBusIndex` 的 static 代码块，对于每一个调用过 `EventBus.register()` 方法的类，APT（Annotation Processing Tool）都会为它生成一行 `putIndex(...)` 代码，并传入一个新建的 `SimpleSubscriberInfo` 对象。

在 `putIndex()` 方法中会调用传入的 `SimpleSubscriberInfo.getSubscriberClass()` 获取到订阅类的 Class 对象作为 key，同时把自身作为 value 存放进 Map 中。

## register 流程

先看 `EventBus.register` 方法，在分析过程中遇到的字段再回过头来分析。

```java
public void register(Object subscriber) {
    // 获取 subscriber 的 Class 对象
    Class<?> subscriberClass = subscriber.getClass();
    // 根据 Class 信息获取到所有加了 @Subscribe 的事件回调方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        // 遍历各个方法
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            // 执行订阅操作
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

上面的代码可以分为两个部分，2~3 行完成了获取注册对象中加了 `@Subscribe` 的事件回调方法，后面的代码真正完成了注册。

### SubscriberMethodFinder.findSubscriberMethods()

`subscriberMethodFinder` 在 EventBus 创建的时候就已经确定了，它的定义如下：

```java
class SubscriberMethodFinder {
    private static final int BRIDGE = 0x40;
    private static final int SYNTHETIC = 0x1000;

    private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
    private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

    private List<SubscriberInfoIndex> subscriberInfoIndexes;
    private final boolean strictMethodVerification;
    private final boolean ignoreGeneratedIndex;

    private static final int POOL_SIZE = 4;
    private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

    SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                           boolean ignoreGeneratedIndex) {
        this.subscriberInfoIndexes = subscriberInfoIndexes;
        this.strictMethodVerification = strictMethodVerification;
        this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    }

    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }

    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }

    private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }

    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }

    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            try {
                methods = findState.clazz.getMethods();
            } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
                String msg = "Could not inspect methods of " + findState.clazz.getName();
                if (ignoreGeneratedIndex) {
                    msg += ". Please consider using EventBus annotation processor to avoid reflection.";
                } else {
                    msg += ". Please make this class visible to EventBus annotation processor to avoid reflection.";
                }
                throw new EventBusException(msg, error);
            }
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }

    static void clearCaches() {
        METHOD_CACHE.clear();
    }

    static class FindState {
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        final StringBuilder methodKeyBuilder = new StringBuilder(128);

        Class<?> subscriberClass;
        Class<?> clazz;
        boolean skipSuperClasses;
        SubscriberInfo subscriberInfo;

        void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }

        void recycle() {
            subscriberMethods.clear();
            anyMethodByEventType.clear();
            subscriberClassByMethodKey.clear();
            methodKeyBuilder.setLength(0);
            subscriberClass = null;
            clazz = null;
            skipSuperClasses = false;
            subscriberInfo = null;
        }

        boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
            Object existing = anyMethodByEventType.put(eventType, method);
            if (existing == null) {
                return true;
            } else {
                if (existing instanceof Method) {
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);
                }
                return checkAddWithMethodSignature(method, eventType);
            }
        }

        private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());

            String methodKey = methodKeyBuilder.toString();
            Class<?> methodClass = method.getDeclaringClass();
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
                // Only add if not already found in a sub class
                return true;
            } else {
                // Revert the put, old class is further down the class hierarchy
                subscriberClassByMethodKey.put(methodKey, methodClassOld);
                return false;
            }
        }

        void moveToSuperclass() {
            if (skipSuperClasses) {
                clazz = null;
            } else {
                clazz = clazz.getSuperclass();
                String clazzName = clazz.getName();
                // Skip system classes, this degrades performance.
                // Also we might avoid some ClassNotFoundException (see FAQ for background).
                if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") ||
                        clazzName.startsWith("android.") || clazzName.startsWith("androidx.")) {
                    clazz = null;
                }
            }
        }
    }
}
```

在它的构造方法里，`subscriberInfoIndexes` 里面只有注解解释器生成的 `MyEventBusIndex` 对象，其他两个参数都是默认值 false。

先看看 `findSubscriberMethods(subscriberClass)` 是如何完成 `@Subscribe` 方法的解析的：

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // 1、如果能获取缓存，直接返回
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }

    // 2、判断是否忽略索引
    if (ignoreGeneratedIndex) {
        // 使用反射来获取事件回调方法
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        // 尝试从索引文件获取事件回调方法
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        // 3、写入缓存
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

注释 1 处是取缓存的操作。页面的生命周期可能会频繁发生变化，因而就可能导致事件的频繁注册、注销，这时候缓存就非常有用了。

注释 2 处，由于 `ignoreGeneratedIndex` 默认为 false，所以这里会执行 `findUsingInfo` 方法来获取订阅类的回调方法。

注释 3 在方法最后返回之前，会进行缓存的更新。

先分析一下 `findUsingInfo` 方法。

#### SubscriberMethodFinder.findUsingInfo()

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    // 开启循环
    while (findState.clazz != null) {
        // 获取目标方法
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);
        }
        // 在父类中查找目标方法
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

##### SubscriberMethodFinder.prepareFindState()

```java
private FindState prepareFindState() {
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    return new FindState();
}
```

通过 `prepareFindState()` 方法尝试从对象池中获取一个 `FindState` 对象，若对象池中没有可用的对象，则新建一个。

##### FindState.initForSubscriber()

```java
void initForSubscriber(Class<?> subscriberClass) {
    this.subscriberClass = clazz = subscriberClass;
    skipSuperClasses = false;
    subscriberInfo = null;
}
```

初始化 `FindState` 对象，使其内部的 `subscriberClass`、`clazz` 都是订阅类，并且注意这时候 `subscriberInfo = null`、`skipSuperClasses = false`。

接着查看 `getSubscriberInfo()` 方法：

##### SubscriberMethodFinder.getSubscriberInfo()

```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }
    if (subscriberInfoIndexes != null) {
        for (SubscriberInfoIndex index : subscriberInfoIndexes) {
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    return null;
}
```

这里首先会通过 FindState 去父类中。。。

但是这里 `findState.subscriberInfo` 是为 null 的，因此不满足条件，接下来会判断 subscriberInfoIndexes（本例中只包含创建 EventBus 对象时传入的 MyEventBusIndex 对象）集合是否为空，如果不为空就会通过 Index 类去查找目标方法。

再次回到 `findUsingInfo()` 方法，可以看到，如果没有找到目标回调方法，也会调用 `findUsingReflectionInSingleClass()` 使用来获取回调方法。因此，`findUsingInfo()` 等于是 `findUsingReflection()` 方法的加强版本。

后面的 while 循环就是从当前的订阅类开始，一直向父类进行循环操作，也就是说，<strong>这里会解析当前订阅类的父类里面的方法</strong>。

如果找到目标回调方法，在通过 `findState.checkAdd()` 校验之后，事件回调方法都被加入到了 `findState.subscriberMethods` 中，然后由 `getMethodsAndRelease(findState)` 方法返回。`getMethodsAndRelease()` 方法就是将 `findState.subscriberMethods` 复制了出来，然后回收了 FindState 对象：

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    // 1
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    // 2
    findState.recycle();
    // 3
    synchronized(FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    return subscriberMethods;
}
```

首先会从 findState 中取出了保存的 subscriberMethods；然后将 findState 里的保存的所有对象进行回收；接着会把把 findState 存储在 FindState 池中方便下一次使用，以提高性能。最后，返回 subscriberMethods。

#### SubscriberMethodFinder.findUsingReflection()

上面就是注解处理器生成的 Index 参与的时候的流程。在这里还是有必要说一下没有 Index 参与时的流程，也就是 `findUsingReflectionInSingleClass()` 方法是如何获取订阅类的回调方法的。

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findUsingReflectionInSingleClass(findState);
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

这里同样走到了 `findUsingReflectionInSingleClass()`：

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // This is faster than getMethods, especially when subscribers are fat classes like Activities
        // 获取该类的所有方法，不包括父类
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
        try {
            // 获取该类以及父类的所有 public 方法，同时指定忽略父类
            methods = findState.clazz.getMethods();
        } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
            String msg = "Could not inspect methods of " + findState.clazz.getName();
            if (ignoreGeneratedIndex) {
                msg += ". Please consider using EventBus annotation processor to avoid reflection.";
            } else {
                msg += ". Please make this class visible to EventBus annotation processor to avoid reflection.";
            }
            throw new EventBusException(msg, error);
        }
        // 标记为跳过父类方法
        findState.skipSuperClasses = true;
    }
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        // 判断方法是否是 PUBLIC 且不是 ABSTRACT、STATIC、SYNTHETIC
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();
            // 判断该方法的参数是不是只有一个
            if (parameterTypes.length == 1) {
                // 判断方法是否有 Subscribe 注解
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    // 以上条件都满足后，就进行最后的校验，然后添加到 findState.subscriberMethods 中
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                // 方法参数 ！= 1,如果开启了严格验证则抛出异常
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            // 方法修饰符不合要求，如果开启了严格验证，则抛出异常
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

这个方法大概分为两部分：

1. 首先通过反射的方式获取订阅者类中的所有声明方法，然后在这些方法里面寻找以 `@Subscribe` 作为注解的方法进行处理。
2. 再经过经过一轮检查，看看 `findState.subscriberMethods` 是否存在，如果没有，将方法名、threadMode、优先级、是否为 sticky 方法等信息封装到 SubscriberMethod 对象中，最后添加到 subscriberMethods 列表中。

从上面方法的分析我们可以看出，订阅类的方法要满足以下条件，才能够顺利的进行注册：

1. 方法必须是 `public` 的，且不能是 `abstract`、`static`、`synthetic` 的
2. 方法参数必须只有一个
3. 方法必须带有 `@Subscribe` 注解

### EventBus.subscribe()

上面这些内容就是订阅类回调方法的获取过程了，下面说说回调方法是如何进行注册的。方法为 `EventBus.subscribe()`：

```java
// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // eventType 为回调方法的入参类型，Object 类型
    Class<?> eventType = subscriberMethod.eventType;
    // 将订阅者以及订阅方法封装为一个 Subscription 类
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        // subscriptionsByEventType：以订阅方法的参数为 key，
        // Subscription 为 value 进行保存
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    // 在订阅参数对应的 Subscription 列表中按照订阅方法的优先级进行排序
    // priority 的数值越高，越在列表的前面；相同 priority 的方法，后来的方法会在后面
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    // 从 typesBySubscriber 中获取
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        // typesBySubscriber：以订阅者为 key，订阅方法参数为 value 进行保存
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);

    // 如果订阅方法订阅的是粘性事件，则会在注册时接受到此粘性事件
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

在上面的方法中，会先将订阅者以及订阅事件包装成为一个 `Subscription` 对象，然后与其他参数一起保存起来，具体为下面两个保存的地方：

- `subscriptionsByEventType`：以订阅事件参数为 key，对应 `Subscription` 对象数组
- `typesBySubscriber`：以订阅者对象为 key，对应订阅事件参数数组。主要是在 EventBus 的 `isRegister()` 方法中去使用的，用来判断这个 Subscriber 对象 是否已被注册过。

另外，Subscription 对象还会根据 priority 进行排序，具体来说：priority 的数值越高，越在列表的前面；相同 priority 的方法，后来的方法会在后面。

同时我们还可以看出粘性事件的触发机制：

1. 粘性事件发出时，会主动通知所有可以处理的方法，不管方法是否是粘性的

```java
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        stickyEvents.put(event.getClass(), event);
    }
    // Should be posted after it is putted, in case the subscriber wants to remove immediately
    post(event);
}
```

2. 在订阅者进行注册时，如果有可以响应的粘性事件，粘性方法会被触发

```javascript
if (subscriberMethod.sticky) {
    if (eventInheritance) {
        ...
        for (...) {
             ...
            if (...) {
                Object stickyEvent = entry.getValue();
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    } else {
        Object stickyEvent = stickyEvents.get(eventType);
        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
    }
}
```

## post 流程

我们可以调用 `EventBus.post()` 以及 `EventBus.postSticky()` 这两个方法来发送事件，前者是普通事件，后者是粘性事件。

粘性事件的发送比较简单，该方法除了保存了事件之外，还调用了 `post` 方法当做非粘性事件进行了分发。此时，会主动通知所有可以处理的方法，不管方法是否是粘性的：

```java
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        stickyEvents.put(event.getClass(), event);
    }
    // Should be posted after it is putted, in case the subscriber wants to remove immediately
    post(event);
}
```

另外，前面提到过，在订阅者进行注册时，如果有可以响应的粘性事件，粘性方法会被触发。

粘性事件不会被消耗掉，除非手动 remove 掉：

- `removeStickyEvent(Class<T>)`
- `removeStickyEvent(Object)`
- `removeAllStickyEvents()`

接下来，我们看看 `EventBus.post()` 是如何触发订阅者的回调事件的。

```java
/** For ThreadLocal, much faster to set (and get multiple values). */
final static class PostingThreadState {
    final List<Object> eventQueue = new ArrayList<>();
    boolean isPosting;
    boolean isMainThread;
    Subscription subscription;
    Object event;
    boolean canceled;
}

private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};

/** Posts the given event to the event bus. */
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

首先会调用 `currentPostingThreadState.get()` 获取一个 PostingThreadState 对象， currentPostingThreadState 是一个 ThreadLocal 类型的对象，而 PostingThreadState 中包含了一个 eventQueue 和其他一些标志位。

接着把传入的 event，保存到了当前线程中的一个变量 PostingThreadState 的 eventQueue 事件队列里面，然后开始执行。当在极短时间内多次调用 `post` 方法时，只会将事件添加到队列里面，第 24 行的代码不成立。而在 31-33 行代码里面，会取时间队列的头进行事件分发。最后所有事件处理完成后，标志位复位。

下面我们看看 `postSingleEvent()` 方法，注释都在里面：

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    // eventInheritance 表示事件接收类是否有层次结构
    if (eventInheritance) {
        // lookupAllEventTypes 方法的作用是查询 class 的父类以及接口
        // 显然示例中就是一个 Object
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            // 调用 postSingleEventForEventType 方法
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        // 直接以当前的 eventClass 调用 postSingleEventForEventType 方法
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    // 最后，当没有订阅者可以处理该事件时，EventBus 会抛出一个 NoSubscriberEvent 事件
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

lookupAllEventTypes 方法的作用就是取出 Event 及其父类和接口的 class 列表，当然重复取的话会影响性能，所以它也做了一个 eventTypesCache 的缓存。

接着看看 `postSingleEventForEventType` 方法

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    // subscriptions 就是示例中对应订阅方法
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                // 调用 postToSubscription 进行真正的分发
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

首先，根据 Event 类型从 subscriptionsByEventType 中取出对应的 subscriptions 对象，最后调用了 `postToSubscription()` 方法。这里需要注意一个细节，只要 eventClass 可以找到对应的 Subscription，那么该方法就会返回 true，也就是说已经发送给订阅者了。

最后看看 `postToSubscription()` 方法，该方法会根据方法的 threaMode 值，决定在哪如何触发回调方法：

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

这里面 5 种 ThreadMode，采取的手段也不一样。从该枚举值的定义可以看出，分为 5 种情况，下表就是每种情况的含义：

无论在哪个线程中执行，最后都会调用 `invokeSubscriber` 方法来触发回调任务：

```javascript
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

这里通过反射调用了订阅者的回调方法。至此，事件的发送已经分析完毕。

## unregister 流程

注销过程原理比较简单，就是将注册时保存到 `subscriptionsByEventType`、`typesBySubscriber` 两个集合中的元素删除。

这两个集合在注册过程中分析过：

- `subscriptionsByEventType` 以订阅事件参数为 key，对应 Subscription 对象数组
- `typesBySubscriber` 以订阅者对象为 key，对应订阅事件参数数组

`EventBus.unregister()` 代码如下：

```java
/** Unregisters the given subscriber from all event classes. */
public synchronized void unregister(Object subscriber) {
    // 先通过 subscriber 订阅者对象从`typesBySubscriber`中获取订阅事件参数数组
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        // 以订阅事件参数为 key，从`subscriptionsByEventType`中获取到对应的`Subscrition`对象
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}

/** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

先通过 subscriber 订阅者对象从 `typesBySubscriber` 中获取订阅事件参数数组；然后以订阅事件参数为 key，从 `subscriptionsByEventType` 中获取到对应的 `Subscrition` 对象。

以上就是 EventBus 的注销过程了。

## 跨进程 EventBus

跨进程 EventBus 的难点在于如何将一个进程中的事件传递到另一个进程中，技术点还是离不来 IPC 的几种常用方式，这里比较实用的是 AIDL 方式。

下面先简单的说一下原理： 1. 弄一个主进程的 Service 作为消息中心，用来向各个进程转发事件 2. 主进程和子进程在初始化的时候都绑定到该 Service 上面，这样就得到了各进程向消息中心 Service 发送消息的通道 3. 在绑定到 Service 后，各个进程向 Service 注册本进程的消息转发器，这样消息中心 Service 就可以对各进程发送消息了，至此 C/S 之间双向的联系通道已经打通

下面分别考虑一下消息的传递，可以分为三种情况讨论。 1. 主进程向子进程的通信：获取主进程 Service 中各个进程（包括主进程自己）的消息转发器，依次转发消息 2. 子进程和主进程的通信：通过初始化绑定到 Service 时获取到的 Service 代理对象，向 Service 发送消息。Service 收到消息后，会向各个进程的消息转发器依次转发消息，这样主进程以及所有子进程都会收到消息 3. 子进程和子进程的通信：先发送到主进程，主进程会进行转发。

注意这里，主进程 Service 持有主进程以及所有子进程的消息转发器，这里不对主进程进行特别处理。此外，所有的事件都会由 Service 进行分发，Service 分发时会对主进程和所有子进程进行分发。

也就是说，如果子进程内部调用了跨进程 EventBus 进行事件分发的话，会在两次 IPC 之后再由改子进程内部的 EventBus 进行触发。这两次 IPC 是：子进程向主进程 Service 发出消息，主进程 Service 接收到消息后向所有进程进行分发。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/EventBus/clipboard_20230323_031638.png)

### 简单实现

文件清单如下：

1. 服务端：IEventBusManager.aidl
2. 客户端：IEventDispatcher.aidl
3. 传输的事件：IPCEvent.java、IPCEvent.aidl
4. 核心类：IPCEventBus.kt

<strong>src/main/aidl/xyz/yorek/eventbus/IEventBusManager.aidl</strong>

```java
// IEventBusManager.aidl
package xyz.yorek.eventbus;

import xyz.yorek.eventbus.IEventDispatcher;
import xyz.yorek.eventbus.IPCEvent;

interface IEventBusManager {
    void register(IEventDispatcher dispatcher);

    void unregister(IEventDispatcher dispatcher);

    /** 向主进程发送Event */
    void postToService(in IPCEvent event);
}
```

<strong>src/main/aidl/xyz/yorek/eventbus/IEventDispatcher.aidl</strong>

```java
// IEventDispatcher.aidl
package xyz.yorek.eventbus;

import xyz.yorek.eventbus.IPCEvent;

interface IEventDispatcher {
    /** 接收其他进程发送过来的Event */
    void dispatch(in IPCEvent event);
}
```

<strong>src/main/aidl/xyz/yorek/eventbus/IPCEvent.aidl</strong>

```css
// IPCEvent.aidl
package xyz.yorek.eventbus;

parcelable IPCEvent;
```

<strong>src/main/java/xyz/yorek/eventbus/IPCEvent.java</strong>

```typescript
public class IPCEvent implements Parcelable {
    public int code;
    public String msg;

    public IPCEvent() {}

    public IPCEvent(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    protected IPCEvent(Parcel in) {
        this.code = in.readInt();
        this.msg = in.readString();
    }

    @Override
    public String toString() {
        return "IPCEvent{" +
                "code=" + code +
                ", msg='" + msg + '\'' +
                '}';
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(this.code);
        dest.writeString(this.msg);
    }

    public static final Creator<IPCEvent> CREATOR = new Creator<IPCEvent>() {
        @Override
        public IPCEvent createFromParcel(Parcel source) {
            return new IPCEvent(source);
        }

        @Override
        public IPCEvent[] newArray(int size) {
            return new IPCEvent[size];
        }
    };
}
```

<strong>src/main/java/xyz/yorek/eventbus/IPCEventBus.kt</strong>

内含一个内部类 `EventBusManagerService`，管理所有进程的事件转发器。

需要注意一下这里面的 `RemoteCallbackList` 的独特用法

```kotlin
object IPCEventBus {
    // 本进程的事件分发器，服务端向本进程客户端发送数据
    private val mEventDispatcher = object : IEventDispatcher.Stub() {
        override fun dispatch(event: IPCEvent?) {
            EventBus.getDefault().post(event)
        }
    }

    // 本进程获取的服务端代理，可以向服务端注册事件分发器，还可以发送数据
    private var mEventBusManagerService: IEventBusManager? = null
    private val mServiceConnection = object : ServiceConnection {
        override fun onServiceDisconnected(name: ComponentName?) {
            mEventBusManagerService = null
        }

        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            service ?: return
            mEventBusManagerService = IEventBusManager.Stub.asInterface(service)

            mEventBusManagerService?.register(mEventDispatcher)
        }
    }

    /**
     * 初始化方法
     */
    fun init(context: Application) {
        this.app = context

        val intent = Intent(app, EventBusManagerService::class.java)
        app.bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE)
    }

    /**
     * 销毁方法，进程不需要时调用
     */
    fun destory() {
        if (mEventBusManagerService?.asBinder()?.isBinderAlive == true) {
            mEventBusManagerService?.unregister(mEventDispatcher)
            app.unbindService(mServiceConnection)
        }
    }

    fun register(any: Any) {
        EventBus.getDefault().register(any)
    }

    fun unregister(any: Any) {
        EventBus.getDefault().unregister(any)
    }

    /**
     * 向服务端发送数据，经过服务端转发到各个进程（包括主进程）
     */
    fun post(event: IPCEvent?) {
        mEventBusManagerService?.postToService(event)
    }

    private lateinit var app: Application

    /**
     * 服务端
     */
    class EventBusManagerService : Service() {
        // 各个进程的事件分发器
        private val mDispatchers = RemoteCallbackList<IEventDispatcher>()
        // 服务端实现
        private val mEventBusManagerService = object : IEventBusManager.Stub() {
            override fun register(dispatcher: IEventDispatcher?) {
                dispatcher ?: return
                mDispatchers.register(dispatcher)
            }

            override fun unregister(dispatcher: IEventDispatcher?) {
                dispatcher ?: return
                mDispatchers.unregister(dispatcher)
            }

            override fun postToService(event: IPCEvent?) {
                dispatchEvent(event)
            }
        }

        override fun onBind(intent: Intent?) = mEventBusManagerService

        /**
         * 分发事件到各个进程
         */
        private fun dispatchEvent(event: IPCEvent?) {
            val n = mDispatchers.beginBroadcast()

            for (i in 0 until n) {
                val dispatcher = mDispatchers.getBroadcastItem(i)
                dispatcher.dispatch(event)
            }

            mDispatchers.finishBroadcast()
        }
    }
}
```

### 使用示例

我们首先在 AndroidMenifest 中注册一下 `IPCEventBus$EventBusManagerService`；顺便注册一下三个在不同进程的 Activity，方便我们测试：

```html
<!-- 主进程的Activity -->
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<!-- 子进程1的Activity -->
<activity android:name=".SecondActivity" android:process=":second" />
<!-- 子进程2的Activity -->
<activity android:name=".ThirdActivity" android:process=":third" />

<service android:name="xyz.yorek.eventbus.IPCEventBus$EventBusManagerService" />
```

上面三个 Activity 引用了同样的布局文件，拥有同样的代码。唯一的区别就是三者的 TAG 不一样，用来区分发出的消息。

下面是 MainActivity 的布局以及代码：

<strong>src/main/res/layout/activity_main.xml</strong>

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/btnPost"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="post"
        android:onClick="post"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/btnFirst"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="first"
        android:onClick="first"
        app:layout_constraintTop_toBottomOf="@id/btnPost"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toStartOf="@+id/btnSecond"/>

    <Button
        android:id="@+id/btnSecond"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="second"
        android:onClick="second"
        app:layout_constraintTop_toBottomOf="@id/btnPost"
        app:layout_constraintStart_toEndOf="@id/btnFirst"
        app:layout_constraintEnd_toStartOf="@+id/btnThird"/>

    <Button
        android:id="@+id/btnThird"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="third"
        android:onClick="third"
        app:layout_constraintTop_toBottomOf="@id/btnPost"
        app:layout_constraintStart_toEndOf="@id/btnSecond"
        app:layout_constraintEnd_toEndOf="parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

<strong>app/src/main/java/xyz/yorek/eventbus/MainActivity.kt</strong>

```kotlin
private const val TAG = "MainActivity"

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        IPCEventBus.init(application)
        IPCEventBus.register(this)

        setContentView(R.layout.activity_main)
    }

    override fun onDestroy() {
        IPCEventBus.unregister(this)
        super.onDestroy()
    }

    @Subscribe
    fun onEventReceived(any: Any?) {
        Log.e(TAG, "receive message : $any")
    }

    fun post(view: View) {
        val msg = "Hello World, from $TAG"
        IPCEventBus.post(IPCEvent(1, msg))
    }

    fun first(view: View) {
        startActivity(Intent(this, MainActivity::class.java))
    }

    fun second(view: View) {
        startActivity(Intent(this, SecondActivity::class.java))
    }

    fun third(view: View) {
        startActivity(Intent(this, ThirdActivity::class.java))
    }
}
```

在 MainActivity 中我们 post 一下消息，然后进入 SecondActivity 再次 post，进入 ThirdActivity 后最后一次 post。

各个进程日志如下（按照日志时间顺序）：

```apache
1569683305.912 31157-31157/xyz.yorek.eventbus E/MainActivity: receive message : IPCEvent{code=1, msg='Hello World, from MainActivity'}

1569683307.622 31157-31169/xyz.yorek.eventbus E/MainActivity: receive message : IPCEvent{code=2, msg='Hello World, from SecondActivity'}
1569683307.625 31093-31093/xyz.yorek.eventbus:second E/SecondActivity: receive message : IPCEvent{code=2, msg='Hello World, from SecondActivity'}

1569683309.245 31157-31169/xyz.yorek.eventbus E/MainActivity: receive message : IPCEvent{code=3, msg='Hello World, from ThirdActivity'}
1569683309.245 31210-31210/xyz.yorek.eventbus:third E/ThirdActivity: receive message : IPCEvent{code=3, msg='Hello World, from ThirdActivity'}
1569683309.246 31093-31246/xyz.yorek.eventbus:second E/SecondActivity: receive message : IPCEvent{code=3, msg='Hello World, from ThirdActivity'}
```

从上面的日志来看，我们已经完成了跨进程 EventBus 的功能。

下面的日志是一份详细记载了各个事件的日志。从日志的中可以看出：本进程的某线程发出的事件会由同一个线程来触发；其他进程发出的事件，会由本进程 Binder 线程池里面的线程触发。

```apache
1569683981.405 31484-31484/xyz.yorek.eventbus E/EventBusManagerService: dispatch begin : main
1569683981.405 31484-31484/xyz.yorek.eventbus E/MainActivity: receive message : main IPCEvent{code=1, msg='Hello World, from MainActivity'}
1569683981.406 31484-31484/xyz.yorek.eventbus E/EventBusManagerService: dispatch end : main
1569683981.406 31484-31484/xyz.yorek.eventbus E/MainActivity: post message done : main

1569684013.794 31484-31526/xyz.yorek.eventbus E/EventBusManagerService: dispatch begin : Binder:31484_1
1569684013.794 31484-31526/xyz.yorek.eventbus E/MainActivity: receive message : Binder:31484_1 IPCEvent{code=2, msg='Hello World, from SecondActivity'}
1569684013.799 32016-32016/xyz.yorek.eventbus:second E/SecondActivity: receive message : main IPCEvent{code=2, msg='Hello World, from SecondActivity'}
1569684013.803 31484-31526/xyz.yorek.eventbus E/EventBusManagerService: dispatch end : Binder:31484_1
1569684013.804 32016-32016/xyz.yorek.eventbus:second E/SecondActivity: post message done : main

1569684035.762 31484-31526/xyz.yorek.eventbus E/EventBusManagerService: dispatch begin : Binder:31484_1
1569684035.762 31484-31526/xyz.yorek.eventbus E/MainActivity: receive message : Binder:31484_1 IPCEvent{code=3, msg='Hello World, from ThirdActivity'}
1569684035.764 32063-32063/xyz.yorek.eventbus:third E/ThirdActivity: receive message : main IPCEvent{code=3, msg='Hello World, from ThirdActivity'}
1569684035.765 31484-31526/xyz.yorek.eventbus E/EventBusManagerService: dispatch end : Binder:31484_1
1569684035.765 32016-32089/xyz.yorek.eventbus:second E/SecondActivity: receive message : Binder:32016_4 IPCEvent{code=3, msg='Hello World, from ThirdActivity'}
1569684035.766 32063-32063/xyz.yorek.eventbus:third E/ThirdActivity: post message done : main
```
