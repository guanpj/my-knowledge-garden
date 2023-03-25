---
title: Java 内存模型
date created: 2023-03-23
date modified: 2023-03-23
tags:
  - 内存模型
  - JMM
  - 原子类
  - happens-before
  - volitale
  - synchrionized
  - final
  - 面试
  - Java
---
# JMM 是什么？

JMM 是和多线程相关的一组规范，它 定义了 JVM 在计算机内存中的工作方式，需要各个 JVM 的实现来遵守 JMM 规范，以便于开发者可以利用这些规范，更方便地开发多线程程序。这样，即便同一个程序在不同的虚拟机上运行，得到的程序结果也是一致的，从而保证了“一次编译，处处运行”。

因此，JMM 与处理器、缓存、并发、编译器有关。它解决了 CPU 多级缓存、处理器优化、指令重排等导致的结果不可预期的问题。

比如关键字 synchronized，JVM 就会在 JMM 的规则下，“翻译”出合适的指令，包括限制指令之间的顺序，以便在即使发生了重排序的情况下，也能保证必要的“可见性”。这样一来，不同的 JVM 对于相同的代码的执行结果就变得可预期了，Java 程序员就只需要用同步工具和关键字就可以开发出正确的并发程序了。

# JMM 抽象结构

Java 作为高级语言，他向开发者屏蔽了多层缓存等底层细节，用 JMM 定义了一套读写数据的规范。Java 线程之间的通信由 JMM 控制，JMM 决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM 定义了线程和主内存之间的抽象关系：

- 线程之间的共享变量存储在主内存中；
- 每个线程只能够直接接触到本地内存，无法直接操作主内存；
- 每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。

本地内存是 JMM 的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Java-Memory-Model/clipboard_20230323_103037.png)

# 内存间交互操作

Java 内存模型定义了 8 个操作来完成主内存和工作内存的交互操作。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Java-Memory-Model/clipboard_20230323_103049.png)

- read：把一个变量的值从主内存传输到工作内存中
- load：在 read 之后执行，把 read 得到的值放入工作内存的变量副本中
- use：把工作内存中一个变量的值传递给执行引擎
- assign：把一个从执行引擎接收到的值赋给工作内存的变量
- store：把工作内存的一个变量的值传送到主内存中
- write：在 store 之后执行，把 store 得到的值放入主内存的变量中
- lock：作用于主内存的变量
- unlock

# JMM 三大特性

## 原子性

如果一个或者一系列的操作，要么全部执行成功，要么全部不执行，不会出现执行一半就终止的情况，则认为此操作具有原子性。

具有原子性的操作有：

- （非 64 位 JDK）除 long 和 double 之外的基本类型（int、byte、boolean、short、char、float）的读 / 写操作；
- 所有引用 reference 的读 / 写操作；
- java.concurrent.Atomic.* 包中的一部分类的一部分方法是具备原子性的，比如 AtomicInteger 的 incrementAndGet 方法；
- 加了 volatile 后，所有变量的读 / 写操作（包含 long 和 double）。这也就意味着 long 和 double 加了 volatile 关键字之后，对它们的读写操作同样具备原子性。

## 可见性

即内存可见性，是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

因为工作内存之间的通信，都需要通过主内存来中转。正是由于所有的共享变量都存在于主内存中，每个线程有自己的工作内存，其中存储的是变量的副本，所以这个副本就有可能是过期的，也就是与主内存不是同步的。比如，当 B 线程从主内存中同步共享变量 V 的时候，就有可能出现与 A 线程的变量 V 的副本不一致的情况，则称线程 A 对变量 V 的操作对于线程 B 不具备可见性了。

主要有三种实现可见性的方式：

- <strong>volatile </strong>—— 用于保证单个变量的可见性；
- <strong>synchronized </strong>—— 对一个变量执行 unlock 操作之前，必须把变量值同步回主内存。
- <strong>final </strong>—— 被 final 关键字修饰的字段在构造器中一旦初始化完成，并且没有发生 this 逃逸（其它线程通过 this 引用访问到初始化了一半的对象），那么其它线程就能看见 final 字段的值。

单个变量要保证内存可见性，可以使用 volitale 关键字修饰。synchronized 关键字也可以保证内存可见性。

## 有序性

一个 Java 程序，包含一系列的语句，开发者默认期望这些语句的实际运行顺序和写的代码顺序一致。但实际上，编译器、JVM 或者 CPU 都有可能出于优化等目的，对于实际指令执行的顺序进行调整，这就是重排序。

这些重排序在单线程环境中不会影响程序执行，但是可能会导致<strong>多线程程序</strong>出现内存可见性问题。JMM 属于语言级的内存模型，它确保在不同的编译器和不同的处理器平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。

要实现多线程环境下的有序性，可以通过两种方式实现：

<strong>volatile </strong>—— 通过添加内存屏障的方式来禁止指令重排，即重排序时不能把后面的指令放到内存屏障之前。

<strong>synchronized</strong> —— 保证每个时刻只有一个线程执行同步代码，相当于是让线程顺序执行同步代码。

# happens-before 规则

Happens-before 关系是用来描述和可见性相关问题的：如果第一个操作 happens-before 第二个操作（也可以描述为，第一个操作和第二个操作之间满足 happens-before 关系），那么我们就说第一个操作对于第二个操作一定是可见的，也就是第二个操作在执行时就一定能保证看见第一个操作执行的结果。

从 JDK 5 开始，Java 使用新的 JSR-133 内存模型，而 JSR-133 使用 happens-before 的概念来阐述操作之间的内存可见性。在 JMM 中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在 happens-before 关系。这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。

happens-before 规则如下：

- 程序顺序规则：一个线程中的每个操作，happens-before 于该线程中的任意后续操作。
- 监视器锁规则：对一个锁的解锁，happens-before 于随后对这个锁的加锁。
- volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。
- start() 规则：如果线程 A 执行操作 ThreadB.start()（启动线程 B），那么 A 线程的 ThreadB.start() 操作 happens-before 于线程 B 中的任意操作。
- join() 规则：如果线程 A 执行操作 ThreadB.join() 并成功返回，那么线程 B 中的任意操作 happens-before 于线程 A 从 ThreadB.join() 操作成功返回。
- interupt()规则：对线程 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过 interrupted() 方法检测到是否有中断发生。
- 传递性：如果 A happens-before B，且 B happens-before C，那么 A happens-before C。

# volitale

第一个作用是保证可见性。如果变量被 volatile 修饰，那么每次修改之后，接下来在读取这个变量的时候一定能读取到该变量最新的值。这是由 hanppens-before 规则保证的。

第二个的作用就是禁止重排序。先介绍一下 as-if-serial 语义：不管怎么重排序，（单线程）程序的执行结果不会改变。在满足 as-if-serial 语义的前提下，由于编译器或 CPU 的优化，代码的实际执行顺序可能与我们编写的顺序是不同的，这在单线程的情况下是没问题的，但是一旦引入多线程，这种乱序就可能会导致严重的线程安全问题。用了 volatile 关键字就可以在一定程度上禁止这种重排序。

但是特别注意，使用 volitale 修饰的变量并不能保证对它操作的原子性，比如 int 类型数据的 ++ 操作（它会分三步：Load、Add、Store）。

volatile 的读性能消耗与普通变量几乎相同，但是写操作稍慢，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。

# synchronized

synchronized 包裹的代码块或者它所修饰的方法中的操作是具备原子性的。它会设立一个临界区，在一个线程操作临界区内的数据的时候，另一个线程无法进来同时操作。它利用对其他线程的排他性来实现保证操作原子性。

synchronized 不仅保证了临界区内最多同时只有一个线程执行操作，同时还保证了在前一个线程释放锁之后，之前所做的所有修改，都能被获得同一个锁的下一个线程所看到，也就是能读取到最新的值。因此 synchronized 也能够避免内存可见性问题。

在 Java 中，每个对象都会有一个 monitor 对象，这个对象其实就是 Java 对象的锁，通常称之为“内置锁”或“对象锁”。类的对象可以有多个，所以每个对象有其独立的对象锁，互不干扰。针对每个类也有一个锁，可以称为“类锁”，类锁实际上是就是类的 Class 对象锁。每个类只有一个 Class 对象，所以每个类只有一个类锁。

Monitor 是线程私有的数据结构，每一个线程都有一个可用 monitor record 列表，同时还有

一个全局的可用列表。每一个被锁住的对象都会和一个 monitor 关联，同时 monitor 中有

一个 Owner 字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。Monitor 是

依赖于底层的操作系统的 Mutex Lock（互斥锁）来实现的线程同步。

# final

JMM 对 final 域遵守如下两个重排序规则：

1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
2. 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

以上两个规则就限制了 final 域的初始化必须在构造函数内，不能重排序到构造函数之外，普通变量则可以。具体的操作为：

1. JMM 在 final 域写入和构造函数返回之前，插入一个 StoreStore 内存屏障，静止处理器将 final 域重排序到构造函数之外。
2. JMM 在初次读 final 域的对象和读对象内 final 域之间插入一个 LoadLoad 内存屏障。

# 单例模式中的 DCL

DCL 单例标准写法：

```java
public class Singleton {

    private static volatile Singleton singleton;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

<strong>为什么需要使用 volatile 关键字？</strong>

singleton = new Singleton() ，它并非是一个原子操作，事实上，在 JVM 中上述语句至少做了以下这 3 件事：

1. 给 singleton 分配内存空间；
2. 调用 Singleton 的构造函数等，来初始化 singleton；
3. 将 singleton 对象指向分配的内存空间（执行完这步 singleton 就不是 null 了）。

因为存在指令重排序的优化，也就是说第 2 步和第 3 步的顺序是不能保证的，最终的执行顺序，可能是 1-2-3，也有可能是 1-3-2。

如果是 1-3-2，那么在第 3 步执行完以后，singleton 就不是 null 了，如果这时第 2 步并没有执行，singleton 对象未完成初始化，它的属性的值可能不是所预期的值。假设此时线程 2 进入 getInstance 方法，由于 singleton 已经不是 null 了，所以会通过第一重检查并直接返回，但其实这时的 singleton 并没有完成初始化，所以使用这个实例的时候就会报错。

<strong>为什么两次判断 singleton 是否为空？</strong>

如果去掉第一次 check，那么所有线程获取实例的时候都要进行同步，都会串行执行，效率低下。

对于第二次 check，如果有两个线程同时调用 getInstance 方法，由于 singleton 是空的 ，因此两个线程都可以通过第一重的 if 判断；然后由于锁机制的存在，会有一个线程先进入同步语句，并进入第二重 if 判断 ，而另外的一个线程就会在 synchronized 处等待。当第一个线程执行完 new Singleton() 语句后，就会退出 synchronized 保护的区域，这时如果没有第二重 if (singleton == null) 判断，那么第二个线程也会创建一个实例。
