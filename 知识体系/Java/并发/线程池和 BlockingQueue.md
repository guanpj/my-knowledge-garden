---
title: 线程池和 BlockingQueue
comments: true
date created: 2023-03-23
date modified: 2023-03-23
id: home
layout: page
tags:
  - ThreadPool
  - BlockingQueue
  - Executor
  - ExecutorService
  - Java
---
# BlockingQueue 阻塞队列

## 队列 

队列是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。

在队列中插入一个队列元素称为入队，从队列中删除一个队列元素称为出队。因为队列只允许在一端插入，在另一端删除，所以只有最早进入队列的元素才能最先从队列中删除，故队列又称为先进先出（FIFO—first in first out）线性表。

## 什么是阻塞队列

`public interface BlockingQueue<E> extends Queue<E>`

- 支持阻塞的插入方法：意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满。
- 支持阻塞的移除方法：意思是在队列为空时，获取元素的线程会等待队列变为非空。

阻塞队列常用于生产者和消费者的场景，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095000.png)

- <strong>抛出异常：</strong>当队列满时，如果再往队列里插入元素，会抛出 IllegalStateException（"Queuefull"）异常。当队列空时，从队列里获取元素会抛出 NoSuchElementException 异常。
- <strong>返回特殊值：</strong>当往队列插入元素时，会返回元素是否插入成功，成功返回 true。如果是移除方法，则是从队列里取出一个元素，如果没有则返回 null。
- <strong>一直阻塞：</strong>当阻塞队列满时，如果生产者线程往队列里 put 元素，队列会一直阻塞生产者线程，直到队列可用或者响应中断退出。当队列空时，如果消费者线程从队列里 take 元素，队列会阻塞住消费者线程，直到队列不为空。
- <strong>超时退出：</strong>当阻塞队列满时，如果生产者线程往队列里插入元素，队列会阻塞生产者线程一段时间，如果超过了指定的时间，生产者线程就会退出。

## 常用阻塞队列

### <strong>ArrayBlockingQueue</strong>

用数组实现的有界阻塞队列。此队列按照先进先出（FIFO）的原则对元素进行排序。默认情况下不保证线程公平的访问队列。可在初始化时通过参数设置，默认是非公平的。

所谓公平访问队列是指阻塞的线程可以按照阻塞的先后顺序访问队列，即先阻塞线程先访问队列。非公平性是对先等待的线程是非公平的，当队列可用时，阻塞的线程都可以争夺访问队列的资格，有可能先阻塞的线程最后才访问队列。

### <strong>LinkedBlockingQueue</strong>

用链表实现的有界阻塞队列。此队列的默认和最大长度为 Integer.MAX_VALUE。此队列按照先进先出的原则对元素进行排序。

### <strong>PriorityBlockingQueue</strong>

支持优先级的无界阻塞队列。默认情况下元素采取自然顺序升序排列。也可以自定义类实现 compareTo() 方法来指定元素排序规则，或者初始化 PriorityBlockingQueue 时，指定构造参数 Comparator 来对元素进行排序。需要注意的是不能保证同优先级元素的顺序。

### <strong>DelayQueue</strong>

支持延时获取元素的无界阻塞队列。队列使用 PriorityQueue 来实现。队列中的元素必须实现 Delayed 接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。DelayQueue 非常有用，可以将 DelayQueue 运用在<strong>缓存系统的设计：</strong>用 DelayQueue 保存缓存元素的有效期，使用一个线程循环查询 DelayQueue，一旦能从 DelayQueue 中获取元素时，表示缓存有效期到了。

### <strong>SynchronousQueue</strong>

不存储元素的阻塞队列。每一个 put 操作必须等待一个 take 操作，否则不能继续添加元素。SynchronousQueue 可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合传递性场景。

### <strong>LinkedTransferQueue</strong>

多了 tryTransfer 和 transfer 方法。

1. <strong>transfer</strong><strong> 方法。</strong>如果当前有消费者正在等待接收元素（消费者使用 take 方法或带时间限制的 poll 方法时），transfer 方法可以把生产者传入的元素立刻 transfer（传输）给消费者。如果没有消费者在等待接收元素，transfer 方法会将元素存放在队列的 tail 节点，并等到该元素被消费者消费了才返回。
2. <strong>tryTransfer</strong><em> 方法。</em> tryTransfer 方法是用来试探生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素，则返回 false。和 transfer 方法的区别是 tryTransfer 方法无论消费者是否接收，方法立即返回，而 transfer 方法是必须等到消费者消费了才返回。

### <strong>LinkedBlockingDeque</strong>

由链表结构组成的双向阻塞队列。所谓双向队列指的是可以从队列的两端插入和移出元素。双向队列因为多了一个操作队列的入口，在多线程同时入队时，也就减少了一半的竞争。

同时，它还多了 addFirst、addLast、offerFirst、offerLast、peekFirst 和 peekLast 等方法，以 First 单词结尾的方法，表示插入、获取（peek）或移除双端队列的第一个元素。以 Last 单词结尾的方法，表示插入、获取或移除双端队列的最后一个元素。另外，插入方法 add 等同于 addLast，移除方法 remove 等效于 removeFirst。但是 take 方法却等同于 takeFirst，不知道是不是 JDK 的 bug，使用时还是用带有 First 和 Last 后缀的方法更清楚。在初始化 LinkedBlockingDeque 时可以设置容量防止其过度膨胀。另外，双向阻塞队列可以运用在“工作窃取”模式中。

<strong>有界和无界</strong>

有限队列就是长度有限，满了以后生产者会阻塞，无界队列就是里面能放无数的东西而不会因为队列长度限制被阻塞，当然空间限制来源于系统资源的限制，如果处理不及时，导致队列越来越大越来越大，超出一定的限制致使内存超限，操作系统或者 JVM 会直接杀死进程。

无界也会阻塞，为何？因为阻塞不仅仅体现在生产者放入元素时会阻塞，消费者拿取元素时，如果没有元素，同样也会阻塞。

<strong>Array 实现和 Linked 实现的区别</strong>

1. 队列中锁的实现不同。ArrayBlockingQueue 实现的队列中的锁是没有分离的，即生产和消费用的是同一个锁；LinkedBlockingQueue 实现的队列中的锁是分离的，即生产用的是 putLock，消费是 takeLock。
2. 生产或消费时操作不同。ArrayBlockingQueue 实现的队列中在生产和消费的时候，是直接将枚举对象插入或移除的；LinkedBlockingQueue 实现的队列中在生产和消费的时候，需要把枚举对象转换为 `Node<E>`进行插入或移除，会影响性能。
3. 队列大小初始化方式不同。ArrayBlockingQueue 实现的队列中必须指定队列的大小；LinkedBlockingQueue 实现的队列中可以不指定队列的大小，但是默认是 Integer.MAX_VALUE。

# 线程池

## 为什么要用线程池？

Java 中的线程池是运用场景最多的并发框架，几乎所有需要异步或并发执行任务的程序都可以使用线程池。在开发过程中，合理地使用线程池能够带来三个好处：

1. <strong>降低资源消耗。</strong>通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2. <strong>提高响应速度。</strong>当任务到达时，任务可以不需要等到线程创建就能立即执行。假设一个服务器完成一项任务所需时间为：T1 创建线程时间，T2 在线程中执行任务的时间，T3 销毁线程时间。如果 T1 + T3 远大于 T2，则可以采用线程池，以提高服务器性能。线程池技术正是关注如何缩短或调整 T1,T3 时间的技术，从而提高服务器程序性能的。它把 T1，T3 分别安排在服务器程序的启动和结束的时间段或者一些空闲的时间段，这样在服务器程序处理客户请求时，不会有 T1，T3 的开销了。
3. <strong>提高线程的可管理性。</strong>线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。

## 线程池的创建各个参数含义

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

### <strong>corePoolSize</strong>

线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于 corePoolSize；

如果当前线程数为 corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；

如果执行了线程池的 prestartAllCoreThreads() 方法，线程池会提前创建并启动所有核心线程。

### <strong>maximumPoolSize</strong>

线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于 maximumPoolSize

### <strong>keepAliveTime</strong>

线程空闲时的存活时间，即当线程没有任务执行时，继续存活的时间。默认情况下，该参数只在线程数大于 corePoolSize 时才有用

### <strong>TimeUnit</strong>

keepAliveTime 的时间单位

### <strong>workQueue</strong>

workQueue 必须是 BlockingQueue 阻塞队列。当线程池中的线程数超过它的 corePoolSize 的时候，线程会进入阻塞队列进行阻塞等待。通过 workQueue，线程池实现了阻塞功能。

一般来说，我们应该<strong>尽量使用有界队列</strong>，因为使用无界队列作为工作队列会对线程池带来如下影响:

1. 当线程池中的线程数达到 corePoolSize 后，新任务将在无界队列中等待，因此线程池中的线程数不会超过 corePoolSize。
2. 由于 1，使用无界队列时 maximumPoolSize 将是一个无效参数。
3. 由于 1 和 2，使用无界队列时 keepAliveTime 将是一个无效参数。
4. 更重要的，使用无界 queue 可能会耗尽系统资源，有界队列则有助于防止资源耗尽，同时即使使用有界队列，也要尽量控制队列的大小在一个合适的范围。

### <strong>threadFactory</strong>

创建线程的工厂，通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名，当然还可以更加自由的对线程做更多的设置，比如设置所有的线程为守护线程。Executors 静态工厂里默认的 threadFactory，线程的命名规则是“pool-数字-thread-数字”。

### <strong>RejectedExecutionHandler</strong>

线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了 4 种策略：

1. AbortPolicy：直接抛出异常，默认策略；
2. CallerRunsPolicy：用调用者所在的线程来执行任务；
3. DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
4. DiscardPolicy：直接丢弃任务；

当然也可以根据应用场景实现 RejectedExecutionHandler 接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。

## 线程池的工作机制

```
1）如果当前运行的线程少于 corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）。

2）如果运行的线程等于或多于 corePoolSize，则将任务加入 BlockingQueue。

3）如果无法将任务加入 BlockingQueue（队列已满），则创建新的线程来处理任务。

4）如果创建新线程将使当前运行的线程超出 maximumPoolSize，任务将被拒绝，并调用 RejectedExecutionHandler.rejectedExecution() 方法。
```

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095008.png)

## Executor 框架

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095011.png)

### <strong>Executor </strong>

<strong>Executor</strong><strong> </strong>是一个接口，它是 Executor 框架的基础，它将任务的提交与任务的执行分离开来。

### <strong>ExecutorService </strong>

ExecutorService 接口继承了 Executor，在其上做了一些 shutdown()、submit() 的扩展，可以说是真正的线程池接口；

### <strong>AbstractExecutorService </strong>

AbstractExecutorService 抽象类实现了 ExecutorService 接口中的大部分方法；

### <strong>ThreadPoolExecutor</strong><em> </em>

ThreadPoolExecutor 是线程池的核心实现类，用来执行被提交的任务。

### <strong>ScheduledExecutorService </strong>

ScheduledExecutorService 接口继承了 ExecutorService 接口，提供了带"周期执行"功能 ExecutorService；

### <strong>ScheduledThreadPoolExecutor </strong>

ScheduledThreadPoolExecutor 是一个实现类，可以在给定的延迟后运行命令，或者定期执行命令。ScheduledThreadPoolExecutor 比 Timer 更灵活，功能更强大。

### <em>ForkJoinPool（since 1.7）</em>

ForkJoinPool 是自 JDK 7 起引入的线程池，核心思想是将大的任务拆分成多个小任务（fork），然后在将多个小任务处理汇总到一个结果上（join），非常像 MapReduce 处理原理。同时，它提供基本的线程池功能，支持设置最大并发线程数，支持任务队，支持线程池停止，支持线程池使用情况监控，也是 AbstractExecutorService 的子类，主要引入了“工作窃取”机制，在多 CPU 计算机上处理性能更佳。

ForkJoinPool 提供了一个更有效的利用线程的机制，当 ThreadPoolExecutor 还在用单个队列存放任务时，ForkJoinPool 已经分配了与线程数相等的队列；当有任务加入线程池时，会被平均分配到对应的队列上，各线程进行正常工作；当有线程提前完成时，会从队列的末端“窃取”其他线程未执行完的任务。

## <strong>Executors 类</strong>

Excutors 类相当于一个线程池工厂，可以创建不同类型的线程池：

### <em>newFixedThreadPool</em>

```java
public static ExecutorService newFixedThreadPool(int nThreads){
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, 
        TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}

public static ExecutorService newFixedThreadPool(int nThreads,     
            ThreadFactory threadFactory){
    return new ThreadPoolExecutor(nThreads, nThreads, 0L,   
        TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>  
        (), threadFactory);
}
```

它的核心线程数和最大线程数是一样的，所以可以把它看作是固定线程数的线程池，它的特点是线程池中的线程数除了初始阶段需要从 0 开始增加外，之后的线程数量就是固定的，就算任务数超过线程数，线程池也不会再创建更多的线程来处理任务，而是会把超出线程处理能力的任务放到任务队列中进行等待。而且就算任务队列满了，到了本该继续增加线程数的时候，由于它的最大线程数和核心线程数是一样的，所以也无法再增加新的线程了。

### <em>newSingleThreadExecutor</em>

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newSingleThreadExecutor(ThreadFactory   
            threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(), threadFactory));
}
```

它会使用唯一的线程去执行任务，原理和 FixedThreadPool 是一样的，只不过这里线程只有一个，如果线程在执行任务的过程中发生异常，线程池也会重新创建一个线程来执行后续的任务。这种线程池由于只有一个线程，所以非常适合用于<strong>所有任务都需要按被提交的顺序依次执行</strong>的场景，而前几种线程池不一定能够保障任务的执行顺序等于被提交的顺序，因为它们是多线程并行执行的。

### <em>newCachedThreadPoo</em>

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,60L,   
              TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
}

public static ExecutorService newCachedThreadPool(ThreadFactory 
            threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L,   
              TimeUnit.SECONDS, new SynchronousQueue<Runnable>(),
              threadFactory);
}
```

可以称作可缓存线程池，它的特点在于线程数是几乎可以无限增加的（实际最大可以达到 Integer.MAX_VALUE，为 2^31-1，这个数非常大，所以基本不可能达到），而当线程闲置时还可以对线程进行回收。也就是说该线程池的线程数量不是固定不变的，当然它也有一个用于存储提交任务的队列，但这个队列是 SynchronousQueue，队列的容量为  0，实际不存储任何任务，它只负责对任务进行中转和传递，所以效率比较高。

当提交一个任务后，线程池会判断已创建的线程中是否有空闲线程，如果有空闲线程则将任务直接指派给空闲线程，如果没有空闲线程，则新建线程去执行任务，这样就做到了动态地新增线程。

### <em>newScheduledThreadPool</em>

```java
public static ScheduledExecutorService 
        newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
  
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize,   
          threadFactory);
}

public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}

public ScheduledThreadPoolExecutor(int corePoolSize,
        ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue(), threadFactory);
}
```

ScheduledExecutorService 有三种方法执行定时任务：

```java
ScheduledExecutorService service = Executors.newScheduledThreadPool(10);
 
service.schedule(new Task(), 10, TimeUnit.SECONDS);
 
service.scheduleAtFixedRate(new Task(), 10, 10, TimeUnit.SECONDS);
 
service.scheduleWithFixedDelay(new Task(), 10, 10, TimeUnit.SECONDS);
```

<strong>schedule </strong>比较简单，表示延迟指定时间后执行一次任务，如果代码中设置参数为 10 秒，也就是 10 秒后执行一次任务后就结束。

<strong>scheduleAtFixedRate</strong> 表示以固定的频率执行任务，它的第二个参数 initialDelay 表示第一次延时时间，第三个参数 period 表示周期，也就是第一次延时后每次延时多长时间执行一次任务。

<strong>scheduleWithFixedDelay </strong>与第二种方法类似，也是周期执行任务，区别在于对周期的定义，之前的 scheduleAtFixedRate 是以任务开始的时间为时间起点开始计时，时间到就开始执行第二次任务，而不管任务需要花多久执行；而 scheduleWithFixedDelay 方法以任务结束的时间为下一次循环的时间起点开始计时。

### <em>newSingleThreadScheduledExecutor</em>

```java
public static ScheduledExecutorService 
       newSingleThreadScheduledExecutor() {
    return new ScheduledThreadPoolExecutor(1);
}

public static ScheduledExecutorService 
       newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(1, threadFactory);
}
```

它只是 ScheduledThreadPool 的一个特例，内部只有一个线程。它只是将核心线程数设置为了 1。

### 五种线程池参数对比

总结上述的五种线程池，我们以核心线程数、最大线程数，以及线程存活时间三个维度进行对比：

<strong>FixedThreadPool</strong>，它的核心线程数和最大线程数都是由构造函数直接传参的，而且它们的值是相等的。重点是使用的队列是容量没有上限的 LinkedBlockingQueue，如果我们对任务的处理速度比较慢，那么随着请求的增多，队列中堆积的任务也会越来越多，最终大量堆积的任务会占用大量内存，并发生 OOM ，也就是 OutOfMemoryError，这几乎会影响到整个程序，会造成很严重的后果。

<strong>SingleThreadExecutor</strong>，newSingleThreadExecutor 和 newFixedThreadPool 的原理是一样的，只不过把核心线程数和最大线程数都直接设置成了 1，但是任务队列仍是无界的 LinkedBlockingQueue，所以也会导致同样的问题。

<strong>CachedThreadPool</strong>，和前面两种线程池不一样的地方在于任务队列使用的是 SynchronousQueue，SynchronousQueue 本身并不存储任务，而是对任务直接进行转发，这本身是没有问题的，但你会发现构造函数的第二个参数被设置成了 Integer.MAX_VALUE，这个参数的含义是最大线程数，所以由于 CachedThreadPool 并不限制线程的数量，当任务数量特别多的时候，就可能会导致创建非常多的线程，最终超过了操作系统的上限而无法创建新线程，或者导致内存不足。

<strong>ScheduledThreadPool</strong>  和 <strong>SingleThreadScheduledExecutor  </strong>的原理是一样的，它采用的任务队列是 DelayedWorkQueue，这是一个延迟队列，同时也是一个无界队列，所以和 LinkedBlockingQueue 一样，如果队列中存放过多的任务，就可能导致 OOM。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095020.png)

### <em>newWorkStealingPool (since 1.8)</em>

```java
public static ExecutorService newWorkStealingPool() {
    return new ForkJoinPool
        (Runtime.getRuntime().availableProcessors(),
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}

public static ExecutorService newWorkStealingPool(int parallelism) {
    return new ForkJoinPool
        (parallelism,
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}

public ForkJoinPool(int parallelism,
                    ForkJoinWorkerThreadFactory factory,
                    UncaughtExceptionHandler handler,
                    boolean asyncMode) {
    this(checkParallelism(parallelism),
         checkFactory(factory),
         handler,
         asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
         "ForkJoinPool-" + nextPoolId() + "-worker-");
    checkPermission();
}
```

这个线程池是在 JDK 7 加入的，它的名字 ForkJoin 也描述了它的执行机制，主要用法和之前的线程池是相同的，也是把任务交给线程池去执行，线程池中也有任务队列来存放任务。但是 ForkJoinPool 线程池和之前的线程池有两点非常大的不同之处。第一点它非常适合执行可以产生子任务的任务。

如图所示，我们有一个 Task，这个 Task 可以产生三个子任务，三个子任务并行执行完毕后将结果汇总给 Result，比如说主任务需要执行非常繁重的计算任务，我们就可以把计算拆分成三个部分，这三个部分是互不影响相互独立的，这样就可以利用 CPU 的多核优势，并行计算，然后将结果进行汇总。这里面主要涉及两个步骤，第一步是拆分也就是 Fork，第二步是汇总也就是 Join，到这里你应该已经了解到 ForkJoinPool 线程池名字的由来了。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095024.png)

比如，处理斐波那契数列问题：

```java
class Fibonacci extends RecursiveTask<Integer> { 
    int n;
 
    public Fibonacci(int n) { 
        this.n = n;
    } 
 
    @Override
    public Integer compute() { 
        if (n <= 1) { 
            return n;
        } 
        Fibonacci f1 = new Fibonacci(n - 1);
        f1.fork();
        Fibonacci f2 = new Fibonacci(n - 2);
        f2.fork();
        return f1.join() + f2.join();
    } 
 }

public static void main(String[] args) throws ExecutionException, 
            InterruptedException { 
    ForkJoinPool forkJoinPool = new ForkJoinPool();
    for (int i = 0; i < 10; i++) { 
        ForkJoinTask task = forkJoinPool.submit(new Fibonacci(i));
        System.out.println(task.get());
    } 
 }

输出内容：
0
1
1
2
3
5
8
13
21
34
```

第二点不同之处在于内部结构，之前的线程池所有的线程共用一个队列，但 ForkJoinPool 线程池中每个线程都有自己独立的任务队列，如图所示。

ForkJoinPool 线程池内部除了有一个共用的任务队列之外，每个线程还有一个对应的双端队列 deque，这时一旦线程中的任务被 Fork 分裂了，分裂出来的子任务放入线程自己的 deque 里，而不是放入公共的任务队列中。如果此时有三个子任务放入线程 t1 的 deque 队列中，对于线程 t1 而言获取任务的成本就降低了，可以直接在自己的任务队列中获取而不必去公共队列中争抢也不会发生阻塞（除了后面会讲到的 steal 情况外），减少了线程间的竞争和切换，是非常高效的。

再考虑一种情况，此时线程有多个，而线程 t1 的任务特别繁重，分裂了数十个子任务，但是 t0 此时却无事可做，它自己的 deque 队列为空，这时为了提高效率，t0 就会想办法帮助 t1 执行任务，这就是“work-stealing”的含义。

双端队列 deque 中，线程 t1 获取任务的逻辑是后进先出，也就是 LIFO（Last In Frist Out），而线程 t0 在“steal”偷线程 t1 的 deque 中的任务的逻辑是先进先出，也就是 FIFO（Fast In Frist Out），如下图所示，图中很好的描述了两个线程使用双端队列分别获取任务的情景。你可以看到，使用“work-stealing”算法和双端队列很好地平衡了各线程的负载。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095028.png)

用一张全景图来描述 ForkJoinPool 线程池的内部结构，可以看到 ForkJoinPool 线程池和其他线程池很多地方都是一样的，但重点区别在于它每个线程都有一个自己的双端队列来存储分裂出来的子任务。ForkJoinPool 非常适合用于递归的场景，例如树的遍历、最优路径搜索等场景。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095032.png)

综合以上，Excutors 创建各种线程池对应的适用场景如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Concurrent-Threadpool/clipboard_20230323_095035.png)

## 提交任务

- <strong>execute </strong>方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。
- <strong>submit </strong>方法用于提交需要返回值的任务。线程池会返回一个 future 类型的对象，通过这个 future 对象可以判断任务是否执行成功，并且可以通过 future 的 get() 方法来获取返回值，get() 方法会阻塞当前线程直到任务完成，而使用 get(long timeout，TimeUnit unit) 方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

## 关闭线程池

### shutdown()

它可以安全地关闭一个线程池，调用 shutdown 方法之后线程池并不是立刻就被关闭，因为这时线程池中可能还有很多任务正在被执行，或是任务队列中有大量正在等待被执行的任务，调用 shutdown 方法后线程池会<strong>在执行完正在执行的任务和队列中等待的任务后才彻底关闭</strong>。但这并不代表 shutdown 操作是没有任何效果的，调用 shutdown 方法后如果还有新的任务被提交，线程池则会根据拒绝策略直接拒绝后续新提交的任务。

### isShutdown()

它可以返回 true 或者 false 来判断线程池是否已经开始了关闭工作，也就是是否执行了 shutdown 或者 shutdownNow 方法。这里需要注意，如果调用 isShutdown 方法的返回的<strong>结果为 true 并不代表线程池此时已经彻底关闭了，这仅仅代表线程池开始了关闭的流程</strong>，也就是说，此时可能线程池中依然有线程在执行任务，队列里也可能有等待被执行的任务。

### isTerminated()

这个方法可以检测线程池是否真正“终结”了，这不仅代表线程池已关闭，同时代表线程池中的所有任务都已经都执行完毕了。因为刚才说过，调用 shutdown 方法之后，线程池会继续执行里面未完成的任务，不仅包括线程正在执行的任务，还包括正在任务队列中等待的任务。比如此时已经调用了 shutdown 方法，但是有一个线程依然在执行任务，那么此时调用 isShutdown 方法返回的是 true ，而调用 isTerminated 方法返回的便是 false ，因为线程池中还有任务正在在被执行，线程池并没有真正“终结”。<strong>直到所有任务都执行完毕了，调用 isTerminated 方法才会返回 true</strong>，这表示线程池已关闭并且线程池内部是空的，所有剩余的任务都执行完毕了。

### awaitTermination()

此方法本身并不是用来关闭线程池的，而是主要用来判断线程池状态的。比如我们给 awaitTermination 方法传入的参数是 10 秒，那么它就会陷入 10 秒钟的等待，直到发生以下三种情况之一：

等待期间（包括进入等待状态之前）线程池已关闭并且所有已提交的任务（包括正在执行的和队列中等待的）都执行完毕，相当于线程池已经“终结”了，方法便会返回 true；

等待超时时间到后，第一种线程池“终结”的情况始终未发生，方法返回 false；

等待期间线程被中断，方法会抛出 InterruptedException 异常。

也就是说，调用 awaitTermination 方法后<strong>当前线程会尝试等待一段指定的时间，如果在等待时间内，线程池已关闭并且内部的任务都执行完毕了，也就是说线程池真正“终结”了，那么方法就返回 true，否则超时返回 fasle。</strong>

我们可以根据 awaitTermination 返回的布尔值来判断下一步应该执行的操作。

### shutdownNow()

最后一个方法是 shutdownNow()，也是 5 种方法里功能最强大的，它与第一种 shutdown 方法不同之处在于名字中多了一个单词 Now，也就是表示立刻关闭的意思。在执行 shutdownNow 方法之后，首先会给所有线程池中的线程发送 interrupt 中断信号，尝试中断这些任务的执行，然后会将任务队列中正在等待的所有任务转移到一个 List 中并返回，我们可以根据返回的任务 List 来进行一些补救的操作，例如记录在案并在后期重试。

这里需要注意的是，由于 Java 中不推荐强行停止线程的机制的限制，即便我们调用了 shutdownNow 方法，如果被中断的线程对于中断信号不理不睬，那么依然有可能导致任务不会停止。可见我们在开发中落地最佳实践是很重要的，我们自己编写的线程应当具有响应中断信号的能力，正确停止线程的方法在前面有讲过，应当利用中断信号来协同工作。

在掌握了这 5 种关闭线程池相关的方法之后，就可以根据自己的业务需要，选择合适的方法来停止线程池，比如通常可以用 shutdown 方法来关闭，这样可以让已提交的任务都执行完毕，但是如果情况紧急，那就可以用 shutdownNow 方法来加快线程池“终结”的速度。

合

## 合理地配置线程池

要想合理地配置线程池，就必须首先分析任务特性，可以从以下几个角度来分析。

- 任务的性质：CPU 密集型任务、IO 密集型任务和混合型任务。
- 任务的优先级：高、中和低。
- 任务的执行时间：长、中和短。
- 任务的依赖性：是否依赖其他系统资源，如数据库连接。

性质不同的任务可以用不同规模的线程池分开处理：

1. CPU 密集型任务应配置尽可能小的线程池，如配置 Ncpu + 1 个线程的线程池。比如加密、解密、压缩、计算等。
2. IO 密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如 2*Ncpu。比如数据库、文件的读写，网络通信等任务等。
3. 《Java 并发编程实战》的作者 Brain Goetz 推荐的计算方法：<strong>线程数 = CPU 核心数 *（1+ 平均等待时间/平均工作时间）。</strong>通过这个公式，我们可以计算出一个合理的线程数量，如果任务的平均等待时间长，线程数就随之增加，而如果平均工作时间长，也就是对于我们上面的 CPU 密集型任务，线程数就随之减少。太少的线程数会使得程序整体性能降低，而过多的线程也会消耗内存等其他资源，所以如果想要更准确的话，可以进行压测，监控 JVM 的线程情况以及 CPU 的负载情况，根据实际情况衡量应该创建的线程数，合理并充分利用资源。
4. 混合型的任务，如果可以拆分，将其拆分成一个 CPU 密集型任务和一个 IO 密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两个任务执行时间相差太大，则没必要进行分解。可以通过 Runtime.getRuntime().availableProcessors() 方法获得当前设备的 CPU 个数。
5. 优先级不同的任务可以使用优先级队列 PriorityBlockingQueue 来处理。它可以让优先级高的任务先执行。
6. 执行时间不同的任务可以交给不同规模的线程池来处理，或者可以使用优先级队列，让执行时间短的任务先执行。
7. 建议使用有界队列。有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点儿，比如几千。
8. 如果当时我们设置成无界队列，那么线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。

```java
public class FairAndUnfair {

    public static void main(String args[]) {
        PrintQueue printQueue = new PrintQueue();


        Thread thread[] = new Thread[10];
        for (int i = 0; i < 10; i++) {
            thread[i] = new Thread(new Job(printQueue), "Thread " + i);
        }


        for (int i = 0; i < 10; i++) {
            thread[i].start();
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}


class Job implements Runnable {

    private PrintQueue printQueue;

    public Job(PrintQueue printQueue) {
        this.printQueue = printQueue;
    }

    @Override
    public void run() {
        System.out.printf("%s: Going to print a job\n", Thread.currentThread().getName());
        printQueue.printJob(new Object());
        System.out.printf("%s: The document has been printed\n", Thread.currentThread().getName());
    }
}


class PrintQueue {

    private final Lock queueLock = new ReentrantLock(false);

    public void printJob(Object document) {
        queueLock.lock();

        try {
            Long duration = (long) (Math.random() * 10000);
            System.out.printf("%s: PrintQueue: Printing a Job during %d seconds\n",
                    Thread.currentThread().getName(), (duration / 1000));
            Thread.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            queueLock.unlock();
        }

        queueLock.lock();
        try {
            Long duration = (long) (Math.random() * 10000);
            System.out.printf("%s: PrintQueue: Printing a Job during %d seconds\n",
                    Thread.currentThread().getName(), (duration / 1000));
            Thread.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            queueLock.unlock();
        }
    }
}

```
