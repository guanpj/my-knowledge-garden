# RxJava 使用及源码分析

[https://juejin.cn/post/6881436122950402056](https://juejin.cn/post/6881436122950402056)

# 介绍

Rx 是 ReactiveX 的简写，后者是 Reactive Extensions 的缩写，Rx 是一种编程模型，用于方便处理异步数据流。

RxJava 是响应式编程（Reactive Extensions）在 Java VM 上的实现，是一个在 Java VM 上使用可观察序列来组成异步的、基于事件的程序库。

它扩展了观察者模式以支持数据/事件序列，并添加了运算符，允许以声明方式将序列组合在一起，同时抽象出对低级线程、同步、线程安全和并发数据结构等事物的关注。

RxJava 是一个 基于事件流、实现异步操作的库，因此它具有如下特点：

- 异步。这里主要就是两个核心的方法 subscribeOn 和 observeOn。这两个方法都传入一个 Scheduler 对象，subscribeOn 指定产生事件的线程，observeOn 指定消费事件的线程。
- 强大的操作符。提供了一系列的转换操作符，可以将事件序列中的某个事件或整个事件序列进行加工处理，转换成不同的事件或事件序列，然后再发送出去。
- 简洁。由于采用链式调用的方式进行事件流的处理，RxJava 在应对较复杂的逻辑的时候，也能展现出清晰的思路。异步操作很关键的一点是程序的简洁性，因为在调度过程比较复杂的情况下，异步代码经常会既难写也难被读懂。 Android 创造的 AsyncTask 和 Handler ，其实都是为了让异步代码更加简洁。RxJava 的优势也是简洁，但它的简洁的与众不同之处在于，随着程序逻辑变得越来越复杂，它依然能够保持简洁。

## 观察者模式

RxJava 有四个基本概念：Observable (可观察者，被观察者，生产者)、 Observer (观察者，消费者)、 subscribe (订阅)、Event (事件)。Observable 和 Observer 通过 subscribe() 方法实现订阅关系，Observable 可以在需要的时候发出事件来通知 Observer，且 RxJava 支持事件以队列的形式连续发送。

| 角色                   | 作用                         |
| ---------------------- | ---------------------------- |
| 被观察者（Observable） | 产生事件                     |
| 观察者（Observer）     | 接收事件，并给出响应动作     |
| 订阅（Subscribe）      | 连接 被观察者 & 观察者       |
| 事件（Event）          | 被观察者 & 观察者 沟通的载体 |

RxJava 的整体结构是一条链，其中：

1. 链的最上游：生产者 Observable
2. 链的最下游：观察者 Observer
3. 链的中间：各个中介节点，既是下游的 Observable，又是上游的 Observer

## 函数响应式编程结构

### 响应式编程（Reactive Programming）

响应式编程是一种<strong>面向数据流和变化传播</strong>的一种编程范式。是发送，流转，监听，响应数据流的一套编程范式。在流转的过程中可以对数据流进行过滤，转变，合并，去重等方式的处理。其中变化传播在程序中也是转换为数据流的形式进行处理。

什么是响应式编程？ 正常情况下，"a = b + c;" 这句代码将 b+c 的值赋给 a，而之后如果 b 和 c 的值改变了不会影响到 a。然而，在响应式编程的框架之下，之后 b 和 c 的值的改变也动态影响着 a，意味着 a 会随着 b 和 c 的变化而变化。

响应式编程的终极思想，一切皆流（everything is stream）。根据唯物辩证法的思想，物质世界是普遍联系和不断运动变化的统一整体，而一切“运动变化”这一“客观现象”都可以通过数据流进行“抽象描述”，也可以说，物质世界是数据流的客观存在。

在程序中数据流是轻量而常见的，变量，数组，集合，对象，事件都可当做数据流来发送处理。

例如：

- 界面数据的展示：可以将要展示的数据由其数据源（网络请求，数据库查询等）将其以数据流的形式进行发出，通过一系列的传递，转变（后台线程传递到 UI 线程，对数据进行条件过滤等），交给界面，界面在拿到数据后，做出相应的响应，将其展示出来。
- 用户动作的交互：可以将一些用户输入事件（触摸屏幕，点击鼠标，敲击键盘等），转换为约定的数据符号，将其以数据流的形式发送，通过层层传递，交给相应的窗口，窗口交给相应的控件，控件监听到相应的事件后，响应用户的行为。

### 函数式编程（Functional programming）

函数式编程是一种通过函数或者函数的组合调用来处理数据，获取结果的一种编程范式。

函数是函数式编程的核心，相当于对象在面向对象编程中的地位一样。在函数式编程中，函数可以独立地解决特定的问题，可以通过与其他函数的组合调用来解决复杂的问题，可以作为另一个函数的参数，可以返回一个新的函数，也可以当做变量在函数之间或函数内部传递。

在函数式编程中，纯函数和高阶函数是两大重要的角色。纯函数具有相互独立性和对外封闭性特点：

1. 纯函数的返回结果只受函数参数的影响，如果输入参数相同不论在哪调用，何时调用，调用多少次其输出结果都是一样的。
2. 纯函数内部的数据处理不受外部环境的影响也不会影响外部环境，每一个函数内部均有一套属于自己的局部变量，只在本函数内部调用也只在本函数内部起作用，其取值由函数的初始参数决定，不受外部变量的影响，同时函数的计算结果只影响函数的返回值，不影响外部变量的值。

高阶函数（Higher-order function）：允许将函数作为参数传入，或者将函数作为返回值返回的函数称为高阶函数。通过高阶函数可以对纯函数进行传递，组合，链接等操作来解决不能靠单一函数解决的复杂问题。

当遇到单一函数无法解决的复杂问题时，可以将其化整为零，拆分成能被单一函数处理的小问题，然后通过高阶函数对这些单一函数进行组合，链接，顺序调用进行解决。

### 函数响应式编程（Functional Reactive Programming）

Rxjava，包括一个事件流的发送源，后面跟着 0 ~ N 个消费者消费事件流，是一种通过一系列函数的组合调用来发送，转变，监听，响应数据流的编程范式。

RxJava 的函数响应式编程具体表现为一个 Observer 订阅一个 Observable，通过创建 Observable 对象发送数据流，经过一系列 Operators（操作符）加工处理和 Scheduler 在不同线程间的转发，最后由观察者接受并做出响应的一个过程。

RxJava 响应式编程的组成：

<strong>Observable/Operator/ Observer</strong>

RxJava 响应式编程中的基本流程：

<strong>Observable -> Operator1 -> Operator2 -> Operator3 -> Observer</strong>

1. Observable 发出一系列事件，他是事件的产生者；
2. Observer 负责处理事件，他是事件的最终消费者；
3. Operator 是对 Observable 发出的事件进行修改和变换（线程，数据类型，中间计算等等）；
4. 若事件从产生到消费不需要其他处理，则可以省略掉中间的 Operator，从而流程变为 Obsevable -> Observer；
5. Observer 通常在主线程执行，所以原则上不要去处理太多的事务，而这些复杂的处理则交给 Operator；

## 操作符 Operator

1. 基于原 Observable 创建一个新的 Observable
2. Observable 内部创建一个 Observer
3. 通过定制 Observable 的 subscribeActual() 方法和 Observer 的 onXxx() 方法，来实现自己的中介角色（例如数据转换、线程切换等）。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/RxJava/clipboard_20230323_040144.png)

## 背压（Backpressure）

当<strong>上下游在不同的线程</strong>中，通过 Observable 发射、处理、响应数据流时，如果上游发射数据的速度快于下游接收处理数据的速度，这样对于那些没来得及处理的数据就会造成积压，这些数据既不会丢失，也不会被垃圾回收机制回收，而是存放在一个异步缓存池中，如果缓存池中的数据一直得不到处理，越积越多，最后就会造成内存溢出，这便是响应式编程中的背压（backpressure）问题。

为此，RxJava 带来了 backpressure 的概念。背压是一种流量的控制步骤，在不知道上流还有多少数据的情形下控制内存的使用，表示它们还能处理多少数据。背压是指在异步场景中，被观察者发送事件速度远快于观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的策略

在 Rxjava 1.0 中，有的 Observable 支持背压，有的不支持，为了解决这种问题，RxJava 2.0 把支持背压和不支持背压的 Observable 区分开来：支持背压的有 Flowable 类，不支持背压的有 Observable、Single、Maybe 和 Completable 类。

1. 在订阅的时候如果使用 FlowableSubscriber，需要通过 s.request(Long.MAX_VALUE) 去主动请求上游的数据项。如果遇到背压报错的时候，FlowableSubscriber 默认已经将错误 try-catch，并通过 onError() 进行回调，程序并不会崩溃；
2. 在订阅的时候如果使用 Consumer，那么不需要主动去请求上游数据，默认已经调用了 s.request(Long.MAX_VALUE)。如果遇到背压报错、且对 Throwable 的 Consumer 没有 new 出来，则程序直接崩溃；
3. 背压策略的上游的默认缓存池是 128。

背压策略：

- error: 缓冲区大概在 128
- buffer: 缓冲区大小在 1000 左右
- drop: 把存不下的事件丢弃
- latest: 只保留最新的
- missing: 缺省设置，不做任何操作

# 观察者基类

## Flowable/Subscriber

Flowable 允许发送 0 - N 个的数据，支持 Reactive-Streams 和背压。

```java
Flowable.range(0, 5)
        .subscribe(new Subscriber<Integer>() {
            Subscription subscription;
            //当订阅后，会首先调用这个方法
            //Subscription 可以用于请求数据或者取消订阅
            @Override
            public void onSubscribe(Subscription s) {
                System.out.println("onSubscribe");
                subscription = s;
                //下游订阅时请求一个数据
                subscription.request(1);
            }
            @Override
            public void onNext(Integer o) {
                System.out.println("onNext:" + o);
                //之后下游每接收到一个数据就请求下一个数据
                subscription.request(1);
            }
            @Override
            public void onError(Throwable t) {
                System.out.println("onError:" + t.getMessage());
            }
            @Override
            public void onComplete() {
                System.out.println("onComplete");
            }
        });
        
输出结果：
onSubscribe
onNext:0
onNext:1
onNext:2
onNext:3
onNext:4
onComplete
```

Flowable 也可以通过 create 操作符来创建：

```java
Flowable.create(new FlowableOnSubscribe<Integer>() {
    @Override
    public void subscribe(FlowableEmitter<Integer> e) throws Exception {
        for (int i = 0; i < 5; i++) {
            e.onNext(i);
        }
        e.onComplete();
    }
}, BackpressureStrategy.BUFFER);
```

## Observable/Observer

Observable 允许发送 0 - N 个 的数据，但不支持背压。

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> e) throws Exception {
        for (int i = 0; i < 5; i++) {
            e.onNext(i);
        }
        e.onComplete();
    }
}).subscribe(new Observer<Integer>() {
    Disposable disposable;

    @Override
    public void onSubscribe(Disposable d) {
        System.out.println("onSubscribe");
        disposable = d;
    }

    @Override
    public void onNext(Integer value) {
        System.out.println("onNext:" + value);
    }

    @Override
    public void onError(Throwable e) {
        System.out.println("onError:" + e.getMessage());
    }

    @Override
    public void onComplete() {
        System.out.println("onComplete");
    }
});

输出结果：
onSubscribe
onNext:0
onNext:1
onNext:2
onNext:3
onNext:4
onComplete
```

这种观察者模型不支持背压：当被观察者快速发送大量数据时，下游不会做其他处理，即使数据大量堆积，调用链也不会报 MissingBackpressureException，消耗内存过大只会 OOM。所以，当我们使用 Observable/Observer 的时候，我们需要考虑的是，数据量是不是很大(官方给出以 1000 个事件为分界线作为参考)。

## Single/SingleObserver

Single 类似于 Observable，不同的是，它总是只发射一个值，或者一个错误通知，而不是发射一系列的值（当然就不存在背压问题），所以可以使用 Single 处理单一时间流。Single 观察者只包含两个事件，一个是正常处理成功的 onSuccess，另一个是处理失败的 onError。因此，不同于 Observable 需要三个方法 onNext, onError, onCompleted，订阅 Single 只需要两个方法：

- onSuccess - Single 发射单个的值到这个方法
- onError - 如果无法发射需要的值，Single 发射一个 Throwable 对象到这个方法

Single 只会调用这两个方法中的一个，而且只会调用一次，<strong>调用了任何一个方法之后，订阅关系终止</strong>。

```java
Single.create(new SingleOnSubscribe<String>() {
    @Override
    public void subscribe(SingleEmitter<String> e) throws Exception {
        e.onSuccess("test");
        e.onSuccess("test2");//错误写法，重复调用也不会处理，因为只会调用一次
    }
}).subscribe(new SingleObserver<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        System.out.println("onSubscribe");
    }

    @Override
    public void onSuccess(String s) {
        //相当于onNext和onComplete
        System.out.println("onSuccess:" + s );
    }

    @Override
    public void onError(Throwable e) {
        System.out.println("onError:" + e.getMessage());
    }
});

输出结果：
onSubscribe
onSuccess:test
```

## Completable/CompletableObserver

Completable 没有发送任何数据，但只处理 onComplete 和 onError 事件。如果你的观察者连 onNext 事件都不关心，可以使用 Completable，它只有 onComplete 和 onError 两个事件：

```java
Completable.create(new CompletableOnSubscribe() {
    @Override
    public void subscribe(CompletableEmitter e) throws Exception {
        e.onComplete();//单一 onComplete 或者 onError
    }

}).subscribe(new CompletableObserver() {
    @Override
    public void onSubscribe(Disposable d) {
        System.out.println("onSubscribe");
    }

    @Override
    public void onComplete() {
        System.out.println("onComplete");
    }

    @Override
    public void onError(Throwable e) {
        System.out.println("onError:" + e.getMessage());
    }
});

输出结果：
onSubscribe
onComplete
```

要转换成其他类型的被观察者，也是可以使用 toFlowable()、toObservable() 等方法去转换。

## Maybe/MaybeObserver

Maybe 模型能够发射 0 个或者 1 个数据，要么成功，要么失败。

如果你有一个需求是可能发送一个数据或者不会发送任何数据，这时候你就需要 Maybe，它类似于 Single 和 Completable 的混合体。

Maybe 可能会调用以下其中一种情况（也就是所谓的 Maybe）：

- 发送数据时：onSuccess 或者 onError
- 不发送数据时：onComplete 或者 onError

可以看到 onSuccess 和 onComplete 是互斥的存在，例子代码如下：

```java
Maybe.create(new MaybeOnSubscribe<String>() {
    @Override
    public void subscribe(MaybeEmitter<String> e) throws Exception {
        e.onSuccess("test");//发送一个数据的情况，或者 onError，不需要再调用 onComplete(调用了也不会触发 onComplete 回调方法)
        //e.onComplete();//不需要发送数据的情况，或者 onError
    }
}).subscribe(new MaybeObserver<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        System.out.println("onSubscribe");
    }

    @Override
    public void onSuccess(String s) {
        //发送一个数据时，相当于 onNext 和 onComplete，但不会触发另一个方法 onComplete
        System.out.println("onSuccess:" + s );
    }

    @Override
    public void onComplete() {
        //无数据发送时候的 onComplete 事件
        System.out.println("onComplete");
    }

    @Override
    public void onError(Throwable e) {
        System.out.println("onError:" + e.getMessage());
    }
});
```

# Disposable

RxJava 通过 Disposable（RxJava1 为 Subscription）在适当的时机取消订阅、停止数据流的发射。这在 Android 等具有 Lifecycle 概念的场景中非常重要，能够避免造成一些不必要 bug 以及内存泄露。

可以通过 Disposable.dispose() 方法来让上游或内部调度器（或两者都有）停止工作，达到“丢弃”的效果。

下面分几种情况分别介绍 Disposable 的工作原理。

无后续，无延迟

有后续，有延迟

无后续，无延迟，有上下游

无后续，有延迟

有后续，无延迟

无后续，有延迟

# Scheduler

1. Scheduler：线程控制器 / 调度器，RxJava 通过它来指定每一段代码应该运行在什么样的线程。RxJava 内置几种 Scheduler 类型，基本适合大多数的使用场景：
2. Schedulers.newThread(): 总是启用新线程，并在新线程执行操作。
3. Schedulers.io():用于 IO 密集型的操作，例如读写 SD 卡文件，查询数据库，网络信息交互等，具有线程缓存机制。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程。调度器接收到任务后，先检查线程缓存池中，如果有空闲的线程，则复用，否则创建新的线程，并加入到线程池中，如果每次都没有空闲线程使用，可以无上限的创建新线程。因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。
4. Schedulers.computation(): 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
5. Schedulers.trampoline( )：在当前线程立即执行任务，如果当前线程有任务在执行，则会将其暂停，等插入进来的任务执行完之后，再将未完成的任务接着执行。
6. Schedulers.single()：拥有一个线程单例，所有的任务都在这一个线程中执行，当此线程中有任务执行时，其他任务将会按照先进先出的顺序依次执行。
7. Scheduler.from(@NonNull Executor executor)：指定一个线程调度器，由此调度器来控制任务的执行策略。
8. 另外，Android 还有一个专用的 AndroidSchedulers.mainThread()，它指定的操作将在 Android 主线程运行。

## subscribeOn()

指定 subscribe() 所发生的线程，即事件产生的线程。若多次设定，则只有一次起作用。

## observeOn()

指定 Observer 所运行在的线程，事件消费的线程。若多次设定，每次均起作用。
