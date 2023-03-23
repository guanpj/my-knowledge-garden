# 三、Handler 原理分析

[看完这篇还不明白 Handler 你砍我 - 掘金](https://juejin.cn/post/6866015512192876557)

Android 应用层通常使用 Handler 实现线程之间的消息通讯，Handler 是 Android 消息机制中非常重要的一员。以下分析通过剖析 Handler 的工作原理来深入了解 Android 应用开发过程中最常见也是最实用的消息收发机制。

在分析之前，先回顾一下 Handler 的使用方式：首先，最常用的是子线程往主线程发送消息：

```java
Handler handler = new Handler() {
    @Override
    public void handleMessage(final Message msg) {
        Log.e("gpj", "Main thread handler received msg:" + msg.what);
    }
};

//发送Message消息对象
Message message = handler.obtainMessage();
message.what = 0;
message.obj = "Hello";
handler.sendMessage(message);

//发送Runnable对象
handler.post(new Runnable() {
    @Override
    public void run() {
        Log.e("gpj", "Runnable has been called");
    }
});

Log输出：
Main thread handler received msg:0
Runnable has been called
```

如果需要反过来在主线程中往某个子线程发送消息，可以使用如下方法：

```java
public class WorkThread extends Thread {

    private Handler mHandler;
    private Looper mLooper;

    @Override
    public void run() {
        Looper.prepare();

        synchronized (this) {
            mLooper = Looper.myLooper();
            Log.e("gpj", Thread.currentThread().getName() +
                 ":Looper prepared");
            //通知等待Looper的线程
            notifyAll();
        }

        Looper.loop();
    }


    public Handler getWorkHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper()) {
                @Override
                public void handleMessage(@NonNull Message msg) {
                    Log.e("gpj", "WorkThread received msg:" + msg.what);
                }
            };
        }
        return mHandler;
    }

    public Looper getLooper() {
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    //等待Looper
                    Log.e("gpj", Thread.currentThread().getName() + 
                        ":Wait for looper");
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
}

//创建WorkThread线程并运行
WorkThread workThread = new WorkThread();
workThread.start();

//获取WorkThread线程中的Handler
Handler handler = workThread.getWorkHandler();

//向WorkThread线程发送消息
Message msg = handler.obtainMessage();
msg.what = 1;
handler.sendMessage(msg);

Log输出：
main:Wait for looper
Thread-3:Looper prepared
WorkThread received msg:1
```

需要注意的是，虽然 WorkThread 线程启动方法 start() 先执行，但是由于线程调度的原因，通过 getWorkHandler() 获取 WorkThread 中的 Handler 时，里面的 getLooper() 方法往往会被更早地执行，因此这里需要加入 wait/notifiy 操作，以确保 Looper 先被创建。

这里会涉及到 Looper、MessageQueue、Message、ThreadLocal 等概念，下面就围绕这几个概念深度分析 Handler 的工作原理。

# ThreadLocal 原理分析

## ThreadLocal 概念和使用方式

ThreadLocal——顾名思义，即用于存储线程本地变量的类。通常情况下，普通变量是可以被任何一个线程访问的，而使用 ThreadLocal 创建的变量只能被当前线程访问，其它线程不能访问也不能够对它进行修改。

在实现上，每个使用 ThreadLocal 变量的线程都会初始化一个完全独立的实例副本，当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。

总的来说，ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景。

```java
public static void main(String[] args) {
    ThreadLocal<String> threadLocal = new ThreadLocal<>();
    threadLocal.set("a");

    System.out.println("Thread:" + Thread.currentThread().getName() 
        + ":" + threadLocal.get());

    new Thread(new Runnable() {
        @Override
        public void run() {
            threadLocal.set("b");
            System.out.println("Thread:" + Thread.currentThread().getName()
                + ":" + threadLocal.get());
        }
    }).start();

    new Thread(new Runnable() {
        @Override
        public void run() {
            threadLocal.set("c");
            System.out.println("Thread:" + Thread.currentThread().getName() 
                + ":" + threadLocal.get());
        }
    }).start();

    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

## ThreadLocal 的 set 和 get 方法

首先，查看它的 set 方法源码：

```java
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

Set 方法可分为三个步骤：

- 首先获取当前线程实例；
- 利用当前线程作为参数获取一个 ThreadLocalMap 的对象；
- 如果上述 ThreadLocalMap 对象不为空，则设置值，否则创建这个 ThreadLocalMap 对象并设置值。

获取 ThreadLocalMap 的 getMap 方法如下：

```java
/**
 * Get the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param  t the current thread
 * @return the map
 */
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

threadLocals 是 Thread 类的一个成员变量：

```java
pulbic class Thread implements Runnable {
    ...
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
}
```

前面的 set 方法中，如果 ThreadLocalMap 对象未创建，则新建 ThreadLocalMap 对象，并设置初始值。createMap 方法如下：

```java
/**
 * Create the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param t the current thread
 * @param firstValue value for the initial entry of the map
 */
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

相应地，它的 get 方法就是获取当前线程的 ThreadLocalMap 对象，并根据 Key 获取对应的 Value：

```java
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

## 关于内存泄漏

有人分析 ThreadLocal 将会导致内存泄漏，理由如下：

- 首先 ThreadLocal 实例被线程的 ThreadLocalMap 实例持有，也可以看成被线程持有
- 如果应用使用了线程池，那么之前的线程实例处理完之后出于复用的目的依然存活
- 所以，ThreadLocal 设定的值被持有，会导致内存泄露

```java
/**
 * ThreadLocalMap is a customized hash map suitable only for
 * maintaining thread local values. No operations are exported
 * outside of the ThreadLocal class. The class is package private to
 * allow declaration of fields in class Thread.  To help deal with
 * very large and long-lived usages, the hash table entries use
 * WeakReferences for keys. However, since reference queues are not
 * used, stale entries are guaranteed to be removed only when
 * the table starts running out of space.
 */
static class ThreadLocalMap {

    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    ...
}
```

ThreadLocalMap 的 Key 并不是直接使用 ThreadLocal 实例，而是 ThreadLocal 实例的弱引用。弱引用的特点是，如果这个对象只存在弱引用，那么在下一次垃圾回收的时候必然会被清理掉。

但是这里仅仅回收作为 Key 的 ThreadLocal 对象，Entry 对象和 Value 并没有回收，ThreadLocalMap 里就可能存在很多 Key 为 null 的 Entry。ThreadLocalMap 实现中已经考虑了这种情况，在调用 set()、get()、remove() 方法的时候，会清理掉 key 为 null 的记录。以 get 方法为例，源码如下:

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        //当前位置没有找到,可能Hash碰撞了
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            //找到了直接返回
            return e;
        if (k == null)
            //检测到有个ThreadLocal对象被回收了,这个时候去清理后面所有Key为null的Entry
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

从源码中可以看到，ThreadLocalMap 清理掉 Key 为 null 的 Entry 的时机是根据 ThreadLocal 对象的 hashCode 去获取 Entry 时发生了 hash 碰撞的情况下，并且下一个 Entry 的 Key 为 null 的时候。因为 ThreadLocalMap 的 Entry 和普通 Map 的不太一样，一般的 Map 是一个链表数组，而它的数组每个元素就是一个 Entry，如果出现 Hash 碰撞了就放到数组的下一个位置，因此如果 get 的时候发现没有碰撞可以认为当前 Map 中的元素还不多，一旦检测到碰撞了并且下一个 Entry 的 Key 被回收了，就调用                             expungeStaleEntry 方法来释放 ThreadLocal 为 null 的那些 Entry，进一步避免了内存泄露。

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            //清理key为null的entry
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            //hashCode发生了变化,重新放置entry
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

那只如果在出现了 key 为 null 的记录后，没有手动调用 remove() 方法，并且之后也不再调用 get()、set()、remove() 方法的情况下，还是有可能会出现内存泄漏的情况。因此，强烈建议回收自定义的 ThreadLocal 变量，尤其在线程池场景下，因为线程经常会被复用，如果不清理自定义的 ThreadLocal 变量，可能会影响后续业务逻辑和造成内存泄露等问题。 并且尽量在代理中使用 try-finally 块进行回收：

```java
objectThreadLocal.set(userInfo); 
try {
    ... 
} finally {
    objectThreadLocal.remove(); 
}
```

## <strong>使用场景</strong>

根据以上分析，ThreadLocal 适用于如下两种场景

- 每个线程需要有自己单独的实例
- 实例需要在多个方法中共享，但不希望被多线程共享

对于第一点，每个线程拥有自己实例，实现它的方式很多。例如可以在线程内部构建一个单独的实例。ThreadLocal 可以以非常方便的形式满足该需求。

对于第二点，可以在满足第一点（每个线程有自己的实例）的条件下，通过方法间引用传递的形式实现。ThreadLocal 使得代码耦合度更低，且实现更优雅。

## 小结

ThreadLocal 可以保证每个线程拥有一个变量的副本，实现了线程隔离。在实现上，每个 Thread 类持有有一个 ThreadLocalMap 对象，ThreadLocalMap 保存了该线程拥有的所有 ThreadLocal 对象，并利用弱引用机制避免了 Map 无限增大并避免了内存泄露的问题。

# Message

## 消息对象

Message 类是应用层消息的载体，是消息机制中的流通对象。它的结构如下：

## 消息池

由于 Handler 消息机制在 framework 层和应用层充当了非常广泛的作用，这也意味着它的使用会非常频繁，因此对于 Message 的创建和回收的优化就显得非常必要。当消息池不为空时，可以直接从消息池中获取 Message 对象，而不是直接创建，提高效率。

```java
private static Message sPool;
private static int sPoolSize = 0;

private static final int MAX_POOL_SIZE = 50;

private static boolean gCheckRecycle = true;
```

静态变量 sPool 的 Message，可以认为是消息池的头部，每个 Message 通过 next 成员变量引用下一个 Message 对象；sPoolSize 代表当前消息池的长度；静态变量 MAX_POOL_SIZE 代表消息池的可用大小；消息池的默认大小为 50。

### Message.obtain()

静态方法 obtain() 可用来获取一个消息：

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null; //从sPool中取出一个Message对象，并消息链表断开
            m.flags = 0; // 清除in-use flag
            sPoolSize--; //消息池的可用大小进行减1操作
            return m;
        }
    }
    return new Message(); // 当消息池为空时，直接创建Message对象
}
```

从消息池获取 Message 操作就是把消息池的头部 Message 取走，再将头部指向它的 next。

### recycler()

通过 recycler() 方法可以把不再使用的消息加入消息池：

```java
public void recycle() {
    if (isInUse()) { //判断消息是否正在使用
        if (gCheckRecycle) { //Android 5.0以后的版本默认为true,之前的版本默认为false.
            throw new IllegalStateException("This message cannot be recycled because it is still in use.");
        }
        return;
    }
    recycleUnchecked();
}

//对于不再使用的消息，加入到消息池
void recycleUnchecked() {
    //将消息标示位置为IN_USE，并清空消息所有的参数。
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) { //当消息池没有满时，将Message对象加入消息池
            next = sPool;
            sPool = this;
            sPoolSize++; //消息池的可用大小进行加1操作
        }
    }
}
```

recycle() 操作将 Message 加入到消息池的过程，其实就是把 Message 加到消息池的头部。

## setAsynchronous(boolean async)

设置消息类型为同步或者异步：

```java
/**
 * Sets whether the message is asynchronous, meaning that it is not
 * subject to {@link Looper} synchronization barriers.
 * <p>
 * Certain operations, such as view invalidation, may introduce synchronization
 * barriers into the {@link Looper}'s message queue to prevent subsequent messages
 * from being delivered until some condition is met.  In the case of view invalidation,
 * messages which are posted after a call to {@link android.view.View#invalidate}
 * are suspended by means of a synchronization barrier until the next frame is
 * ready to be drawn.  The synchronization barrier ensures that the invalidation
 * request is completely handled before resuming.
 * </p><p>
 * Asynchronous messages are exempt from synchronization barriers.  They typically
 * represent interrupts, input events, and other signals that must be handled independently
 * even while other work has been suspended.
 * </p><p>
 * Note that asynchronous messages may be delivered out of order with respect to
 * synchronous messages although they are always delivered in order among themselves.
 * If the relative order of these messages matters then they probably should not be
 * asynchronous in the first place.  Use with caution.
 * </p>
 *
 * @param async True if the message is asynchronous.
 *
 * @see #isAsynchronous()
 */
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

# Looper 工作原理分析

## Looper 与 Handler 的关联

通常我们会通过以下构造方法创建 Handler 对象：

```java
@Deprecated
public Handler() {
    this(null, false);
}

@Deprecated
public Handler(@Nullable Callback callback) {
    this(callback, false);
}

public Handler(@Nullable Callback callback, boolean async) {
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

public Handler(@NonNull Looper looper) {
    this(looper, null, false);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback) {
    this(looper, callback, false);
}

@UnsupportedAppUsage
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

可以看到，前面两个 Handler 已被标记为废弃，因为的创建必需依赖与 Looper，如果 Handler 被创建于没有 Looper 的线程中将会抛出异常，因此推荐使用后面两种构造方法。

在构造方法中，成员变量 mQueue 将被赋值为 Looper 中的 MessageQueue 示例；mCallback 则被赋值为调用者传入的对象；mAsynchronous 默认为 false（@UnsupportedAppUsage 标记的构造方法不能直接被外面调用）。

## Looper 的创建与获取

那么 Looper 怎么获取呢？如果要在主线程或者当前线程中接收消息，可以这样创建：

```java
Handler handler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(final Message msg) {
        ...
    }
};

Handler handler = new Handler(Looper.myLooper()) {
    @Override
    public void handleMessage(final Message msg) {
        ...
    }
};
```

当然，也可以像本文开头部分示例的方式一样，传入一个在其它线程中创建的 Looper，表示该 Handler 将在对应的线程中处理消息。那么如何为一个线程创建 Looper 呢？只需要调用 Looper.prepare() 方法即可：

```java
public class WorkThread extends Thread {

    private Handler mHandler;

    @Override
    public void run() {
        Looper.prepare();

        mHandler = new Handler(Looper.myLooper());

        Looper.loop();
    }
    
    ...
}
```

prepare() 是 Looper 类的一个静态方法：

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

可以看到，这里的 Looper 来自于 ThreadLocal 中，并且每个线程最多只能拥有一个 Looper 对象。结合前面的分析可知，这个 Looper 对象只属于当前线程。在创建 Looper 的同时，mQueue 也将被初始化，quitAllowed 表示是否可以退出，意味着每个线程也最多拥有一个 MessageQueue。

## Looper.loop()

接着看 Looper 的 loop() 方法，提取关键部分后的代码如下：

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    ...
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        // 不断获取Message消息
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        ...
        try {
            msg.target.dispatchMessage(msg);
            ...
        } finally {
            ...
        }
        ...
        msg.recycleUnchecked();
    }
}
```

可以看到，loop() 方法是一个死循环，这个死循环唯一的退出途径就是 queue.next() 方法返回 null 的时候。循环里面会不断调用 MessageQueue.next() 方法来获取 Message 对象，next() 方法在没有返回 Message 时方法将被阻塞。Looper 处理 Message 的方式是调用它的 target 对象的 dispatchMessage 方法，并把 Message 对象作为参数传进去。

## MessageQueue 原理分析

MessageQueue 字面意义为消息队列，但其实它的结构是一个单向链表。它的 next() 和 enqueueMessage() 方法分别对应该链表的获取和添加元素操作。为了方便理解，下面仍然称之为消息队列。

### MessageQueue 的创建

MessageQueue 只有一个构造方法：

```java
private long mPtr; // used by native code

MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    // 初始化消息队列，mPtr是供native代码调用的一个变量    
    mPtr = nativeInit();
}
```

### next() 方法分析

MessageQueue 的 next() 方法如下：

```java
@UnsupportedAppUsage
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {//表示当MessageQueue已经退出，则直接返回
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //阻塞操作，等待时长nextPollTimeoutMillis后，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 碰到同步屏障，则跳过同步消息，只取出异步消息，并且不会移除屏障
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 当前Message的触发事件在当前时间时间之后，则继续等待
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 将目标Message对象从队列中移除，并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    // 设置消息的使用状态：flags |= FLAG_IN_USE
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

这里也同样创建了一个死循环，nativePollOnce() 方法在 JNI 层会调用到内核中的 epoll_wait() 方法，这是阻塞方法（阻塞时线程休眠，释放 CPU 资源），用于等待事件发生或者超时，当被事件到来或者等待 nextPollTimeoutMillis（时间） 后会被唤醒。因此，这里的 `for(;;)` 并不会造成无限循环。

> 当 nextPollTimeoutMillis = -1 时，表示消息队列中无消息，会一直等待；当 nextPollTimeoutMillis = 0 时会直接返回。

msg.target == null 表示队列头部的消息是一个同步屏障，此时只会处理异步消息，直到同步屏障被移除。文章后面会补充更详细的关于同步屏障的理解。

这里后半部分代码中，当处于空闲时，往往会执行 IdleHandler 中的方法。当 nativePollOnce() 返回后， next() 从 mMessages 中提取一个消息。

### enqueueMessage() 方法分析

再来看 enqueueMessage() 方法：

```java
boolean enqueueMessage(Message msg, long when) {
    // 普通Message必须设置target
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    synchronized (this) {
        // 如果Message具有FLAG_IN_USE标记，则不允许再使用
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        // MessageQueue正在退出时，需要回收该Message并加入到消息池，返回false
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;// 头节点
        boolean needWake; // 需要唤醒MessageQueue
        if (p == null || when == 0 || when < p.when) {
            // p == null表示MessageQueue为空；when == 0表示msg为插队消息；
            // when < p.when表示该消息触发时间是最早的（mMessages为链表头结点）
            // 这三种情况都把msg加入到队列头部
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 将消息按时间顺序插入到MessageQueue。一般不需要唤醒MessageQueue，
            // 除非最头部存在barrier，并且同时Message是队列中最早的异步消息。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            // 遍历队列，对比when，找到合适的位置并插入
            for (;;) {
                prev = p;
                p = p.next;
                // 如果到达队尾或者when在当前结点之前，表示找到插入位置
                if (p == null || when < p.when) {
                    break;
                }
                // 当前结点也是异步消息，则不需要唤醒了
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            //插入链表
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

根据以上分析可知，MessageQueue 是按照 Message 触发时间的先后顺序排列的，队列头部的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头部开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

消息插入一般不需要唤醒 MessageQueue，但是在 MessageQueue 当前处于阻塞状态的大前提下，消息的插入位置为 MessageQueue 头部，或者最头部存在同步障碍时 Message 是队列中最早的异步消息，意味着当前消息需要马上处理，插入后就需要唤醒阻塞中的 MessageQueue。

当需要唤醒时，通过 nativeWake() 方法在 JNI 层会最终调用到内核中的调用 Looper::wake() 方法向管道 mWakeEventfd 写入字符，此时会唤醒 MessageQueue.next() 方法 nativePollOnce() 处的代码继续向下执行。

### removeMessage() 方法分析

```java
void removeMessages(Handler h, int what, Object object) {
    if (h == null) {
        return;
    }

    synchronized (this) {
        Message p = mMessages;

        // Remove all messages at front.
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            p.recycleUnchecked();
            p = n;
        }

        // Remove all messages after front.
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```

这个移除消息的方法，采用了两个 while 循环，第一个循环是从队头开始，移除符合条件的消息，第二个循环是从头部移除完连续的满足条件的消息之后，再从队列后面继续查询是否有满足条件的消息需要被移除。

### 同步屏障 SyncBarrier、同步消息和异步消息

默认情况下，我们向某个 Handler 发送的消息默认都是异步消息。试想一下，如果某些时候想要让消息快速地发送到指定线程，但是如果该线程的 MessageQueue 中积累了大量未处理消息，则达不到预期的效果。

因此 Handler 体系提供了一套同步屏障机制，通过区分同步消息和异步消息，让某些特殊的消息得以更快被执行的机制。这也从前面 next() 方法分析中得到验证。

那么如何设置和取消一个同步屏障呢？方法就是调用它的 postSyncBarrier() 和 removeSyncBarrier () 方法：

```java
@UnsupportedAppUsage
@TestApi
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}

public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

前面提到过，普通 Message 必须设置一个 target，对于特殊的 Message 是没有 target 的。 这个消息的价值就是用于拦截同步消息，所以并不会唤醒 Looper。

postSyncBarrier 方法会返回一个 int 类型的 token，调用 removeSyncBarrier 并传入这个 token 则可以移除同步屏障。

这两个方法都被标记了 @UnsupportedAppUsage 注解，因此普通开发者无法直接调用。那么它在什么时候会被调用呢？Android 应用框架中为了更快的响应 UI 刷新事件，在 ViewRootImpl.scheduleTraversals 中使用了同步屏障：

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 设置同步障碍，暂停处理后面的同步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 在下一帧到来的时候执行 mTraversalRunnable
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}

final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 移除同步障碍
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...
        // 正式进入 View 绘制流程
        performTraversals();
         ...
    }
}
```

postCallback 方法跟踪：

```java
@UnsupportedAppUsage
@TestApi
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}

@UnsupportedAppUsage
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
            // 将消息类型设置为异步
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

可以看到，最后通过 setAsynchronous() 方法可以将消息设置成异步消息。

通过设置同步障碍，并且发送异步消息，就可以确保 mTraversalRunnable 优先被执行，mTraversalRunnable 最终会调用 performTraversals 方法触发 View 树的 measure、layout 和 draw 等操作，因此使用同步屏障可以防止 UI 显示出现卡顿。

这里也可以看到，调用 performTraversals 方法之前会将同步障碍移除掉。

综合以上分析可知，同步消息必须配合同步障碍才能达到屏蔽异步消息的作用，也就是说，如果没有设置同步屏障，同步消息与异步消息在 Handler 体系中没有任何区别。

# Handler 发送和接收消息

前面的分析中，Looper 在 loop() 方法中获取到每一个 Message 对象后，会调用它的 target 对象的 dispatchMessage() 方法，并把 Message 本身作为参数传入。那么 这个 target 对象又是何方神圣？

## 发送 Message

再来看 Handler 发送消息的过程：

```java
Message message = handler.obtainMessage();
message.what = 0;
message.obj = "Hello";
handler.sendMessage(message);
```

这里会将包装好的 Message 对象传入 sendMessage 方法中：

```java
public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

public final boolean sendEmptyMessage(int what){
    return sendEmptyMessageDelayed(what, 0);
}

public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}

public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageAtTime(msg, uptimeMillis);
}
```

可以看到，无论是发送普通消息还是延时消息，最终都会调用到 enqueueMessage() 方法。它们之间的关系如下：

![](static/boxcnlWO0sk2OHdIuDwFGKevW01.png)

enqueueMessage() 方法源码如下：

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, 
        @NonNull Message msg, long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

可以看到，每个消息的 target 都被赋值为 Handler 本身了。这里传进来的 MessageQueue 就是 Handler 中的 mQueue 成员变量，它来自 Looper.mQueue。enqueueMessage() 方法就是将封装好的消息加入到 MessageQueue 中。

## 接收 Message

回过头来看 Handler 的 dispatchMessage() 方法：

```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

可以看到，这里一共有三种不同的途径处理 Message。

### msg.callback.run()

```java
if (msg.callback != null) {
    handleCallback(msg);
}

private static void handleCallback(Message message) {
    message.callback.run();
}
```

首先，会判断 Message 的 callback 是否为空，如果不为空则会调用它的 run 方法。那什么时候这个 callback 会被赋值呢？原来 Handler 还有另外一种打开方式：

```java
public final boolean post(@NonNull Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}

public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis) {
    return sendMessageAtTime(getPostMessage(r), uptimeMillis);
}

public final boolean postAtTime(
        @NonNull Runnable r, @Nullable Object token, long uptimeMillis) {
    return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
}

public final boolean postDelayed(@NonNull Runnable r, long delayMillis) {
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}

public final boolean postDelayed(Runnable r, int what, long delayMillis) {
    return sendMessageDelayed(getPostMessage(r).setWhat(what), delayMillis);
}

public final boolean postDelayed(
        @NonNull Runnable r, @Nullable Object token, long delayMillis) {
    return sendMessageDelayed(getPostMessage(r, token), delayMillis);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

通过传入一个 Runnable 对象，最终也会被封装成一个 Message 对象，而这个 Message 对象的 callback 就是传入的 Runnable。

### mCallback.handleMessage(msg)

```java
if (mCallback != null) {
    if (mCallback.handleMessage(msg)) {
        return;
    }
}
```

在 msg.callback == null 的情况下，如果 mCallback != null 则调用 mCallback.handleMessage(msg)。这个 mCallback 对象又是从何而来？答案在 Handler 的创建过程：

```typescript
public Handler(@NonNull Looper looper) {
    this(looper, null, false);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback) {
    this(looper, callback, false);
}

@UnsupportedAppUsage
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

public interface Callback {
    /**
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    boolean handleMessage(@NonNull Message msg);
}
```

需要注意的是，如果 mCallback.handleMessage(msg) 返回了 true，那么代表消息处理完成，直接返回；如果返回 false，则会继续调用 handleMessage(msg) 方法。

### handleMessage(msg)

```java
/**
 * Subclasses must implement this to receive messages.
 */
public void handleMessage(@NonNull Message msg) {
}
```

从注释中可以看出，Handler 的子类必须实现这个方法才能接收 Message，因此这种方式是提供给它的子类接收并处理消息的。

# 总结

<strong>Message:</strong> 消息载体，用于携带消息参数、消息目标等。可以通过它的 obtain() 和 recycle() 方法进行复用和回收消息。

<strong>MessageQueue:</strong> 消息队列，用来存放 Handler 发送过来的 Message，按照 when 字段进行优先级排序，其本身是一个以 Message 串联起来的一个单链表。enqueueMessage() 和 next() 方法是生产、消费关系，next() 在队列中没有消息或者消息处理时间未到时会进行阻塞；enqueueMessage() 方法会将消息插入到队列指定位置，并且如果当前消息需要马上处理的时候会唤醒阻塞中的队列。

<strong>Handler:</strong> 调用目标线程的 MessageQueue 的 enqueueMessage() 方法发送消息，并且处理接收到的消息。

<strong>Looper:</strong> 轮询调用 MessageQueue 的 next() 方法获取 Message，并调用目标 Handler 进行处理。

![](static/boxcnJ4AvWmx6ybN9CN73XuodWe.png)
