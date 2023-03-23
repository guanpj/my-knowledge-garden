---
title: CAS 和 AQS 原理
comments: true
date created: 2023-03-23
date modified: 2023-03-23
id: home
layout: page
tags:
  - CAS
  - AQS
  - 原子类
  - ConcurrentHashMap
  - 并发
  - 锁
  - 面试
  - Java
---
# CAS

## 介绍

CAS 英文全称是 Compare-And-Swap，中文叫做“比较并交换”，它是一种思想、一种算法。

在大多数处理器的指令中，都会实现 CAS 相关的指令，这一条指令就可以完成“比较并交换”的操作，也正是由于这是一条（而不是多条）CPU 指令，所以 CAS 相关的指令是具备原子性的，这个组合操作在执行期间不会被打断，这样就能保证并发安全。由于这个原子性是由 CPU 保证的，所以无需我们程序员来操心。

CAS 有三个操作数：内存值 V、预期值 A、要修改的值 B。CAS 最核心的思路就是，<strong>仅当预期值 A 和当前的内存值 V 相同时，才将内存值修改为 B。</strong>

## <strong>使用及原理</strong>

### ConcurrentHashMap

截取 ConcurrentHashMap 部分 putVal 方法的代码，如下所示：

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new   
                   NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break; // no lock when adding to empty bin
        }
    ...
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

该方法里面只有一行代码，即调用变量 U 的 compareAndSwapObject 的方法，U 是 Unsafe 类型的实例。

### ConcurrentLinkedQueue

接下来看并发容器的第二个案例。非阻塞并发队列 ConcurrentLinkedQueue 的 offer 方法里也有 CAS 的身影，offer 方法的代码如下所示：

```java
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            if (p.casNext(null, newNode)) {
                if (p != t) 
                    casTail(t, newNode); 
                return true;
            }
        }
        else if (p == q)
            p = (t != (t = tail)) ? t : head;
        else
            p = (p != t && t != (t = tail)) ? t : q;
    }
}

boolean casNext(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
}
```

可以看出，在 offer 方法中，有一个 for 循环，这是一个死循环，在第 8 行有一个与 CAS 相关的方法，是 casNext 方法，用于更新节点。那么如果执行 p 的 casNext 方法失败的话，casNext 会返回 false，那么显然代码会继续在 for 循环中进行下一次的尝试。而 casNext 方法中也用到了 UnSafe 类。

### 原子类

在编程领域里，原子性意味着“一组操作要么全都操作成功，要么全都失败，不能只操作成功其中的一部分”。而 java.util.concurrent.atomic 下的类，就是具有原子性的类，可以原子性地执行添加、递增、递减等操作。

原子类的作用和锁有类似之处，是为了保证并发情况下线程安全。不过原子类相比于锁，有一定的优势：

- 粒度更细：原子变量可以把竞争范围缩小到变量级别，通常情况下，锁的粒度都要大于原子变量的粒度。
- 效率更高：除了高度竞争的情况之外，使用原子类的效率通常会比使用同步互斥锁的效率更高，因为原子类底层利用了 CAS 操作，不会阻塞线程。

原子类一共可以分为以下这 6 类：

| 类型                               | 具体类                                                                         |
| ---------------------------------- | ------------------------------------------------------------------------------ |
| Atomic* 基本类型原子类             | AtomicInteger、AtomicLong、AtomicBoolean                                       |
| Atomic*Array 数组类型原子类        | AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray                      |
| Atomic*Reference 引用类型原子类    | AtomicReference、AtomicStampedReference、AtomicMarkableReference               |
| Atomic*FieldUpdater 升级类型原子类 | AtomicIntegerfieldupdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater |
| Adder 累加器                       | LongAdder、DoubleAdder                                                         |
| Accumulator 积累器                 | LongAccumulator、DoubleAccumulator                                             |

下面以 AtomicInteger 为例，分析在 Java 中如何利用 CAS 实现原子操作。

getAndAdd 方法是 AtomicInteger 类的重要方法，这个方法的代码在 Java 1.8 中的实现如下：

```java
public final int getAndAdd(int delta) {    
    return unsafe.getAndAddInt(this, valueOffset, delta);
}
```

可以看出，里面再次用到了 Unsafe 这个类，并且调用了 unsafe.getAndAddInt 方法。

### 实现原理

从以上示例可以看出，Unsafe 其实是 CAS 的核心类，并且其核心方法都是由 native 层面来实现的。由于 Java 无法直接访问底层操作系统，而是需要通过 native 方法来实现。不过尽管如此，JVM 还是留了一个后门，Unsafe 类提供了硬件级别的原子操作，可以利用它直接操作内存数据。

看一下 AtomicInteger 的一些重要代码，如下所示：

```java
public class AtomicInteger extends Number 
           implements java.io.Serializable {
   // setup to use Unsafe.compareAndSwapInt for updates
   private static final Unsafe unsafe = Unsafe.getUnsafe();
   private static final long valueOffset;
 
   static {
       try {
           valueOffset = unsafe.objectFieldOffset
               (AtomicInteger.class.getDeclaredField("value"));
       } catch (Exception ex) { throw new Error(ex); }
   }
 
   private volatile int value;
   public final int get() {return value;}
   ...
}
```

可以看出，在数据定义的部分，首先获取了 Unsafe 实例，并且定义了 valueOffset。接下来 static 代码块会在类加载的时候执行，执行时会调用 Unsafe 的 objectFieldOffset 方法，从而得到当前这个原子类的 value 的偏移量，并且赋给 valueOffset 变量，这样就获取到了 value 的偏移量，它的含义是在内存中的偏移地址，因为 Unsafe 就是根据内存偏移地址获取数据的原值的，这样就能通过 Unsafe 来实现 CAS 了。

value 是用 volatile 修饰的，它就是原子类存储的值的变量，由于它被 volatile 修饰，可以保证在多线程之间看到的 value 是同一份，保证了可见性。

接下来继续看 Unsafe 的 getAndAddInt 方法的实现，代码如下：

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
   int var5;
   do {
       var5 = this.getIntVolatile(var1, var2);
   } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
   return var5;
}
```

首先，它是一个 do-while 循环，所以这是一个死循环，直到满足循环的退出条件时才可以退出。

接下来看 do 后面的这一行代码 var5 = this.getIntVolatile(var1, var2) ，这是个 native 方法，作用就是获取在 var1 中的 var2 偏移处的值。

传入的两个参数，第一个就是当前原子类，第二个就是最开始获取到的 offset，这样一来我们就可以获取到当前内存中偏移量的值，并且保存到 var5 里面。此时 var5 实际上代表当前时刻下的原子类的数值。

现在再来看 while 的退出条件，也就是 compareAndSwapInt 这个方法，它一共传入了 4 个参数，这 4 个参数是 var1、var2、var5、var5 + var4，为了方便理解，我们给它们取了新了变量名，分别 object、offset、expectedValue、newValue，具体含义如下：

- 第一个参数 object 就是将要操作的对象，传入的是 this，也就是 atomicInteger 这个对象本身；
- 第二个参数是 offset，也就是偏移量，借助它就可以获取到 value 的数值；
- 第三个参数 expectedValue，代表“期望值”，传入的是刚才获取到的 var5；
- 而最后一个参数 newValue 是希望修改的数值 ，等于之前取到的数值 var5 再加上 var4，而 var4 就是我们之前所传入的 delta，delta 就是我们希望原子类所改变的数值，比如可以传入 +1，也可以传入 -1。

所以 compareAndSwapInt 方法的作用就是，判断如果现在原子类里 value 的值和之前获取到的 var5 相等的话，那么就把计算出来的 var5 + var4 给更新上去，所以说这行代码就实现了 CAS 的过程。

一旦 CAS 操作成功，就会退出这个 while 循环，但是也有可能操作失败。如果操作失败就意味着在获取到 var5 之后，并且在 CAS 操作之前，value 的数值已经发生变化了，证明有其他线程修改过这个变量。

这样一来，就会再次执行循环体里面的代码，重新获取 var5 的值，也就是获取最新的原子变量的数值，并且再次利用 CAS 去尝试更新，直到更新成功为止，所以这是一个死循环。

总结一下，Unsafe 的 getAndAddInt 方法是通过循环 + CAS 的方式来实现的，在此过程中，它会通过 compareAndSwapInt 方法来尝试更新 value 的值，如果更新失败就重新获取，然后再次尝试更新，直到更新成功。

## CAS 造成的三个问题

### ABA 问题

决定 CAS 是否进行 swap 的判断标准是“当前的值和预期的值是否一致”，如果一致，就认为在此期间这个数值没有发生过变动，这在大多数情况下是没有问题的。

但是在有的业务场景下，我们想确切知道从上一次看到这个值以来到现在，这个值是否发生过变化。例如，这个值假设从 A 变成了 B，再由 B 变回了 A，此时，它不仅可以认为发生了变化，而且变化了两次。

解决 ABA 问题，可以在在变量值自身之外，再添加一个版本号，通过对比版本号来判断值是否变化过。

### 自旋时间过长

由于单次 CAS 不一定能执行成功，所以 CAS 往往是配合着循环来实现的。在高并发场景下有的时候甚至是死循环，不停地进行重试，直到线程竞争不激烈的时候，才能修改成功。

因此要根据实际情况来选择是否使用 CAS，在高并发的场景下，通常 CAS 的效率是不高的。

### 范围不能灵活控制

通常执行 CAS 的时候，是针对某一个而不是多个共享变量的，这个变量可能是 Integer 类型，也有可能是 Long 类型、对象类型等，但是不能针对多个共享变量同时进行 CAS 操作，因为这多个变量之间是独立的，简单的把原子操作组合到一起，并不具备原子性。因此如果想对多个对象同时进行 CAS 操作并想保证线程安全的话，是比较困难的。

有一个解决方案，那就是利用一个新的类，来整合刚才这一组共享变量，这个新的类中的多个成员变量就是刚才的那多个共享变量，然后再利用 atomic 包中的 AtomicReference 来把这个新对象整体进行 CAS 操作，这样就可以保证线程安全。

相比之下，如果使用 synchronized 关键字则会比较方便。

# AQS

## 介绍

队列同步器 AbstractQueuedSynchronizer，是用来构建锁或者其他同步组件的基础框架，它使用了一个 int 类型的 state 变量表示同步状态，通过内置的 FIFO 队列来完成资源获取线程的排队工作。AQS 定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如常用的 ReentrantLock、Semaphore、CountDownLatch 和 CyclicBarrior 等。

同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器。可以这样理解二者之间的关系：

锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；

同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域。

实现者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/CAS-AQS/clipboard_20230323_101329.png)

## 使用及原理

如果想使用 AQS 来写一个自己的线程协作工具类，通常而言是分为以下三步，这也是 JDK 里利用 AQS 类的主要步骤：

1. 新建一个自己的线程协作工具类，在内部写一个 Sync 类，该 Sync 类继承 AbstractQueuedSynchronizer，即 AQS；
2. 想好设计的线程协作工具类的协作逻辑，在 Sync 类里，根据是否是独占，来重写对应的方法。如果是独占，则重写 tryAcquire 和 tryRelease 等方法；如果是非独占，则重写 tryAcquireShared 和 tryReleaseShared 等方法；
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/CAS-AQS/clipboard_20230323_101333.png)
3. 在自己的线程协作工具类中，实现获取/释放的相关方法，并在里面调用 AQS 对应的方法，如果是独占则调用 acquire 或 release 等方法，非独占则调用 acquireShared 或 releaseShared 或 acquireSharedInterruptibly 等方法。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/CAS-AQS/clipboard_20230323_101337.png)

AQS 最核心的三大部分就是状态（state）、FIFO 等待队列和期望协作工具类去实现的获取）/释放等重要方法。

### state

state 的含义并不是一成不变的，它会根据具体实现类的作用不同而表示不同的含义。比如说在 Semaphore 中，state 表示的是剩余许可证的数量；在 CountDownLatch 工具类里面，state 表示的是需要“倒数”的数量。他的定义如下：

```java
/**
 * The synchronization state.
 */
private volatile int state;
```

state 的访问方式有三种：

- getState()：获取当前同步状态。
- setState(int newState)：设置当前同步状态，volatile 保证了它的多线程状态下的可见性。
- compareAndSetState(int expect, int update)：使用 CAS 设置当前状态，保证状态设置的原子性。

### FIFO 队列

即先进先出队列，这个队列最主要的作用是存储等待的线程。假设很多线程都想要同时抢锁，那么大部分的线程是抢不到的，那怎么去处理这些抢不到锁的线程呢？就得需要有一个队列来存放、管理它们。所以 AQS 的一大功能就是充当线程的“排队管理器”。

当多个线程去竞争同一把锁的时候，就需要用排队机制把那些没能拿到锁的线程串在一起；而当前面的线程释放锁之后，这个管理器就会挑选一个合适的线程来尝试抢刚刚释放的那把锁。所以 AQS 就一直在维护这个队列，并把等待的线程都放到队列里面。

在队列中，分别用 head 和 tail 来表示头节点和尾节点，两者在初始化的时候都指向了一个空节点。头节点可以理解为“当前持有锁的线程”，而在头节点之后的线程就被阻塞了，它们会等待唤醒，唤醒也是由 AQS 负责操作的。

### 获取/释放方法

#### <strong>获取</strong>

获取操作通常会依赖 state 变量的值，根据 state 值不同，协作工具类也会有不同的逻辑，并且在获取的时候也经常会阻塞。

比如 ReentrantLock 中的 lock 方法就是其中一个“获取方法”，执行时，如果发现 state 不等于 0 且当前线程不是持有锁的线程，那么就代表这个锁已经被其他线程所持有了。这个时候，当然就获取不到锁，于是就让该线程进入阻塞状态。

再比如，Semaphore 中的 acquire 方法就是其中一个“获取方法”，作用是获取许可证，此时能不能获取到这个许可证也取决于 state 的值。如果 state 值是正数，那么代表还有剩余的许可证，数量足够的话，就可以成功获取；但如果 state 是 0，则代表已经没有更多的空余许可证了，此时这个线程就获取不到许可证，会进入阻塞状态，所以这里同样也是和 state 的值相关的。

再举个例子，CountDownLatch 获取方法就是 await 方法（包含重载方法），作用是“等待，直到倒数结束”。执行 await 的时候会判断 state 的值，如果 state 不等于 0，线程就陷入阻塞状态，直到其他线程执行倒数方法把 state 减为 0，此时就代表现在这个门闩放开了，所以之前阻塞的线程就会被唤醒。

#### <strong>释放</strong>

释放方法是站在获取方法的对立面的，通常和刚才的获取方法配合使用。我们刚才讲的获取方法可能会让线程阻塞，比如说获取不到锁就会让线程进入阻塞状态，但是释放方法通常是不会阻塞线程的。

比如在 Semaphore 信号量里面，释放就是 release 方法（包含重载方法），release() 方法的作用是去释放一个许可证，会让 state 加 1；而在 CountDownLatch 里面，释放就是 countDown 方法，作用是倒数一个数，让 state 减 1。所以也可以看出，在不同的实现类里面，他们对于 state 的操作是截然不同的，需要由每一个协作类根据自己的逻辑去具体实现。

### CountDownLatch 的实现

```java
public class CountDownLatch {

    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    
    public void countDown() {
        //AQS 类中的方法
        sync.releaseShared(1);
    }
    
    public void await() throws InterruptedException {
        //AQS 类中的方法
        sync.acquireSharedInterruptibly(1);
    }
    
    private final Sync sync;
   
    private static final class Sync 
              extends AbstractQueuedSynchronizer {
        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }

       public final void acquireSharedInterruptibly(int arg) 
              throws InterruptedException {
           if (Thread.interrupted())
               throw new InterruptedException();
           if (tryAcquireShared(arg) < 0)
               doAcquireSharedInterruptibly(arg);
       }
        
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
    }
    
}
```
