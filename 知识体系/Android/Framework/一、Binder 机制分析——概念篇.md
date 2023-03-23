
[上次没砍我的,这次我又来了。看完这篇还不明白 Binder 你砍我(一) - 掘金](https://juejin.cn/post/6867139592739356686)

[不懂砍我之看完这篇还不明白 Binder 你砍我(二) - 掘金](https://juejin.cn/post/6868901776368926734)

[不懂砍我之看完这篇还不明白 Binder 你砍我(三) - 掘金](https://juejin.cn/post/6869953788388573192)

[不懂砍我之看完这篇还不明白 Binder 你砍我(四)完结篇 - 掘金](https://juejin.cn/post/6872987936753876999)

[听说你 Binder 机制学的不错，来解决下这几个问题(一) - 掘金](https://juejin.cn/post/6844903469971685390)

[听说你 Binder 机制学的不错，来解决下这几个问题(二) - 掘金](https://juejin.cn/post/6844903469984268296)

[听说你 Binder 机制学的不错，来解决下这几个问题(三) - 掘金](https://juejin.cn/post/6844903469988446221)

[3 分钟带你看懂 android 的 Binder 机制 - 掘金](https://juejin.cn/post/6844903764986462221)

[图解 Android 中的 binder 机制 - 掘金](https://juejin.cn/post/6844904115777044488)

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

[Android 中 mmap 原理及应用简析 - 掘金](https://juejin.cn/post/6844903762146885640)

[认真分析 mmap:是什么 为什么 怎么用 - 胡潇 - 博客园](https://www.cnblogs.com/huxiao-tee/p/4660352.html)

[图解 | 不得错过的 Binder 浅析(一) - 掘金](https://juejin.cn/post/6890088205916307469)

[图解 | 不得错过的 Binder 浅析(二) - 掘金](https://juejin.cn/post/6897868762410811405)

[Android Frameworks 研究 - 悠然红茶的个人页面 - OSCHINA - 中文开源技术交流社区](https://my.oschina.net/youranhongcha?tab=newest&catalogId=373547)

# 前言

在开始之前有必要了解一下几点知识，方便文章的理解。

## 进程隔离

以下内容来自[维基百科](https://link.juejin.cn/?target=https://zh.wikipedia.org/wiki/%E8%BF%9B%E7%A8%8B%E9%9A%94%E7%A6%BB)：

进程隔离是为保护操作系统中进程互不干扰而设计的一组不同硬件和软件的技术。这个技术是为了避免进程 A 写入进程 B 的情况发生。 进程的隔离实现，使用了<strong>虚拟地址空间</strong>。进程 A 的虚拟地址和进程 B 的虚拟地址不同，这样就防止进程 A 将数据信息写入进程 B。

## 进程空间划分：用户空间(User Space)/内核空间(Kernel Space)

现在操作系统都是采用的虚拟存储器，对于 32 位系统而言，它的寻址空间（虚拟存储空间）就是 2 的 32 次方，也就是 4GB。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也可以访问底层硬件设备的权限。为了保护用户进程不能直接操作内核，保证内核的安全，操作系统从逻辑上将虚拟空间划分为用户空间（User Space）和内核空间（Kernel Space）。针对 Linux 操作系统而言，将最高的 1GB 字节供内核使用，称为内核空间；较低的 3GB 字节供各进程使用，称为用户空间。

简单的说就是，内核空间（Kernel）是系统内核运行的空间，用户空间（User Space）是用户程序运行的空间。为了保证安全性，它们之间是隔离的。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044233.png)

## 系统调用：用户态与内核态

虽然从逻辑上进行了用户空间和内核空间的划分，但不可避免的用户空间需要访问内核资源，比如文件操作、访问网络等等。为了突破隔离限制，就需要借助<strong>系统调用</strong>来实现。系统调用是用户空间访问内核空间的唯一方式，保证了所有的资源访问都是在内核的控制下进行的，避免了用户程序对系统资源的越权访问，提升了系统安全性和稳定性。

Linux 使用两级保护机制：0 级供系统内核使用，3 级供用户程序使用：

当一个任务（进程）执行系统调用而陷入内核代码中执行时，称进程处于<strong>内核运行态（内核态）</strong>。此时处理器处于特权级最高的（0 级）内核代码中执行。当进程处于内核态时，执行的内核代码会使用当前进程的内核栈。每个进程都有自己的内核栈。

而当进程在执行用户自己的代码的时候，我们称其处于<strong>用户运行态（用户态）</strong>。此时处理器在特权级最低的（3 级）用户代码中运行。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044239.png)

## Linux IPC 原理

由于进程 A 和进程 B 占用的虚拟内存地址不同，因此它们之间是相互透明的，都以为自己独享了系统的资源，当然也不能直接跟对方交互。但是，有些情况下有些进程难免会需要跟其他进程进行交互，这个交互过程就叫 IPC(Inter-Process Communication，进程间通信)。IPC 的实质就是数据的交互，因此我们这里将进行 IPC 过程中的通信调用方和被调用方分别称为数据发送方和数据接收方，IPC 通信的过程如下：

1. 数据发送方进程将数据放在<strong>内存缓存区</strong>，通过系统调用陷入内核态
2. 内核程序在内核空间开辟一块<strong>内核缓存区</strong>，通过系统调用 copy_from_user 函数将数据从数据发送方用户空间的内存缓存区拷贝到内核空间的内核缓存区中
3. 数据接收方进程在自己的用户空间开辟一块内存缓存区
4. 内核程序将内核缓存区中通过系统调用 copy_to_user 函数将数据拷贝到数据接收方进程的内存缓存区

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044242.png)

通过以上过程，一次 IPC 就完成了，但是这种传统的 IPC 机制有两个问题：

- 性能比较低：整个过程数据的传递需要经历发送方内存缓存区——内核缓存区——接收方内存缓存区的过程
- 接收方进程事先不知道需要开辟多大的内存用于存放数据，因此需要开辟尽可能大的空间或者事先调

# 为什么要学习 Binder 机制

作为 Android 工程师，开发过程中难免会产生如下疑问：

- 为什么 Activity 之前传递数据需要进行序列化，而且数据大小超过 1M 的时候会 Crash？
- 四大组件的启动流程为何那么复杂，它们之间的通讯机制是怎么建立的？
- AIDL 通讯的实现原理是什么？

其实上面的问题都和 Binder 的概念密切相关，事实上，在应用程序底层 Binder 的存在感相当高：四大组件之间跨进程通讯、卸载/安装应用程序、启动/关闭一个 Activity 和 show/hide 一个 Dialog 等场景都要依赖 Binder 机制，Binder 是 Android 系统用于跨进程通讯的框架，因此了解 Binder 机制则是回答以上问题的关键。

# 为什么是 Binder

Android 系统是基于 Linux 内核的，Linux 已经提供了管道、消息队列、共享内存和 Socket 等传统的 IPC 机制，为什么 Android 还要提供 Binder 来实现 IPC 呢？主要是基于<strong>性能</strong>、<strong>稳定性</strong>和<strong>安全性</strong>几方面的原因。

## 性能

从性能的角度来说，传统的 IPC 方式有如下特点：

- socket：传输效率低，开销大，它主要用在跨网络的进程间通信和本机上进程间的低速通信
- 消息队列和管道：采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程
- 共享内存：虽然无需拷贝，但控制复杂，难以使用

Socket、管道和消息队列通讯过程：

## 安全性

Android 作为一个开放性的平台，市场上有各类海量的应用供用户选择安装，因此安全性对于 Android 平台而言极其重要。作为用户当然不希望我们下载的 APP 偷偷读取通信录并上传隐私数据、后台偷跑流量和消耗手机电量等。传统的 IPC 没有任何安全措施，完全依赖上层协议来保证。

Android 为每个安装好的 APP 分配了自己的 UID，故而进程的 UID 是鉴别进程身份的重要标志。但是传统的 IPC 接收方无法获得对方可靠的进程用户 ID/进程 ID（UID/PID），从而无法鉴别对方身份。只能由用户在数据包中填入 UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标识只有由 IPC 机制在内核中添加。

其次，传统 IPC 访问接入点是开放的，无法建立私有通道。比如命名管道的名称，system V 的键值，socket 的 ip 地址或文件名都是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。

而同时 Binder 既支持实名 Binder，又支持匿名 Binder，安全性高。

## 稳定性

再说说稳定性，Binder 基于 C/S 架构，客户端（Client）有什么需求就丢给服务端（Server）去完成，架构清晰、职责明确又相互独立，自然稳定性更好。共享内存虽然无需拷贝，但是控制负责，难以使用。从稳定性的角度讲，Binder 机制是优于内存共享的。

## 小结

Binder 相对于其它通讯方式的特点：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044250.png)

# Binder 原理解析

从字面上看 Binder 是胶水的意思，在这里，Binder 的职责是在不同的进程之间扮演一个桥梁的角色，让它们之间能够相互通信。进程间的通讯少不了 Linux 内核的支持，而 Binder 并不属于内核的一部分，但是，得益于 Linux 的 LKM(Loadable Kernel Module) 机制：

> 模块是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间运行

因此，Binder 作为这种模块存在于内核之中，也称为 Binder 驱动。回顾上一小节的内容，传统 Linux IPC 的过程需要经历两次数据拷贝，Binder 借助 Linux 的另一个特性，只用一次数据拷贝，就能实现 IPC 过程，这就是<strong>内存映射</strong>：

> Binder IPC 机制中涉及到的内存映射通过 mmap() 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间，映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。

内存映射能减少数据拷贝次数，实现用户空间和内核空间的高效互动。两个空间各自的修改能直接反映在映射的内存区域，从而被对方空间及时感知。也正因为如此，内存映射能够提供对进程间通信的支持。

## Binder 通讯模型

Binder IPC 通信过程大致如下：

1. Binder 驱动在内核空间创建一个<strong>数据接收缓存区</strong>
2. 然后在内核空间开辟一块<strong>内存缓存区</strong>并与数据接收缓存区建立映射关系，同时，通过 mmap 函数建立<strong>数据接收缓存区</strong>与<strong>数据接收方的内存缓存区</strong>的映射关系
3. 数据发送方通过系统调用 copy_from_user() 函数将数据从内存缓存区拷贝到内核缓存区，由于内核缓存区通过数据接收缓存区跟数据接收方的内存缓存区存在间接的映射关系，相当于将数据直接拷贝到了接收方的用户空间，这样便完成了一次 IPC 的过程。

Binder 通讯过程示意：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044253.png)

在进行 Binder IPC 的时候，实际情况比上面介绍的要复杂，Binder 通讯模型是基于 C/S 架构的，通信调用方进程称为 Client 进程，被调用方称为 Server 进程，除此之外还需要 ServiceManager 和 Binder 驱动的参与，它们都是通过 open/mmap/iotl 等系统调用来访问设备文件 dev/binder 来实现 IPC 过程的。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044256.png)

其中，Client、Server 和 ServiceManager 运行在用户空间，Binder Driver 运行在内核空间，Client 和 Server 需由用户自己实现，ServiceManager 和 Binder Driver 则由系统提供。

[Android Binder 设计与实现](https://blog.csdn.net/universus/article/details/6211589) 文章中对 Client 和 Server 等角色有详细的描述：

> <strong>Binder 驱动</strong>

> Binder 驱动就如同路由器一样，是整个通信的核心；驱动负责进程之间 Binder 通信的建立，Binder 在进程之间的传递，Binder 引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。

>

> <strong>ServiceManager 与实名 Binder</strong>

> ServiceManager 和 DNS 类似，作用是将字符形式的 Binder 名字转化成 Client 中对该 Binder 的引用，使得 Client 能够通过 Binder 的名字获得对 Binder 实体的引用。注册了名字的 Binder 叫实名 Binder，就像网站一样除了除了有 IP 地址以外还有自己的网址。Server 创建了 Binder，并为它起一个字符形式，可读易记得名字，将这个 Binder 实体连同名字一起以数据包的形式通过 Binder 驱动发送给 ServiceManager ，通知 ServiceManager 注册一个名为“张三”的 Binder，它位于某个 Server 中。驱动为这个穿越进程边界的 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager。ServiceManger 收到数据后从中取出名字和引用填入查找表。

>

> 细心的读者可能会发现，ServierManager 是一个进程，Server 是另一个进程，Server 向 ServiceManager 中注册 Binder 必然涉及到进程间通信。当前实现进程间通信又要用到进程间通信，这就好像蛋可以孵出鸡的前提却是要先找只鸡下蛋！Binder 的实现比较巧妙，就是预先创造一只鸡来下蛋。ServiceManager 和其他进程同样采用 Bidner 通信，ServiceManager 是 Server 端，有自己的 Binder 实体，其他进程都是 Client，需要通过这个 Binder 的引用来实现 Binder 的注册，查询和获取。ServiceManager 提供的 Binder 比较特殊，它没有名字也不需要注册。当一个进程使用 BINDER<em>SET</em>CONTEXT_MGR 命令将自己注册成 ServiceManager 时 Binder 驱动会自动为它创建 Binder 实体（<strong>这就是那只预先造好的那只鸡</strong>）。其次这个 Binder 实体的引用在所有 Client 中都固定为 0 而无需通过其它手段获得。也就是说，一个 Server 想要向 ServiceManager 注册自己的 Binder 就必须通过这个 0 号引用和 ServiceManager 的 Binder 通信。类比互联网，0 号引用就好比是域名服务器的地址，你必须预先动态或者手工配置好。要注意的是，这里说的 Client 是相对于 ServiceManager 而言的，一个进程或者应用程序可能是提供服务的 Server，但对于 ServiceManager 来说它仍然是个 Client。

>

> <strong>Client 获得实名 Binder 的引用</strong>

> Server 向 ServiceManager 中注册了 Binder 以后， Client 就能通过名字获得 Binder 的引用了。Client 也利用保留的 0 号引用向 ServiceManager 请求访问某个 Binder: 我申请访问名字叫张三的 Binder 引用。ServiceManager 收到这个请求后从请求数据包中取出 Binder 名称，在查找表里找到对应的条目，取出对应的 Binder 引用作为回复发送给发起请求的 Client。从面向对象的角度看，Server 中的 Binder 实体现在有两个引用：一个位于 ServiceManager 中，一个位于发起请求的 Client 中。如果接下来有更多的 Client 请求该 Binder，系统中就会有更多的引用指向该 Binder ，就像 Java 中一个对象有多个引用一样。

因此，ServiceManager 是 Binder IPC 通信过程的核心，是上下文的管理者。

## ServiceManager 启动流程

好多文章称 ServiceManager 是 Binder 驱动的守护进程、大管家，其实 ServiceManager 的作用很简单就是提供了查询服务和注册服务的功能。下面我们来看一下 ServiceManager 启动的过程：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044300.png)

1. ServiceManager 分为 framework 层和 native 层，framework 层只是对 native 层进行了封装方便调用，图上展示的是 native 层的 ServiceManager 启动过程。
2. ServiceManager 的启动是系统在开机时，init 进程解析 init.rc 文件调用 service_manager.c 中的 main() 方法入口启动的。 native 层有一个 binder.c 封装了一些与 Binder 驱动交互的方法。
3. ServiceManager 的启动分为三步，首先打开驱动创建全局链表 binder_procs，然后将自己当前进程信息保存到 binder_procs 链表，最后开启 loop 不断的处理共享内存中的数据，并处理 BR_xxx 命令（ioctl 的命令，BR 可以理解为 binder reply 驱动处理完的响应）。

## Binder 通讯过程

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044304.png)

1. 首先，一个进程使用 <strong>BINDER_SET_CONTEXT_MGR</strong> 命令通过 Binder 驱动将自己注册成为 ServiceManager；
2. 各个 Server 通过 Binder 驱动向 ServiceManager 注册 Binder 实体，表明自己可以对外提供服务，这时 Binder 驱动会为这个 Binder 创建位于内核中的实体节点以及 ServiceManager 对该节点的引用，并将名字和该引用打包给 ServiceManager，ServiceManager 接收到数据包后将数据包中的名字和引用填入查找表中（svcinfo）。
3. Client 通过上面 Server 的名字在 Binder 驱动的帮助下从 ServiceManager 中获取到该 Server 对应的 Binder <strong>引用对象 </strong>BinderProxy，由于 BinderProxy  同样具有 Server 的能力，因此 Client 可以通过这个引用与真实的 Server 进行交互。

```java
//获取WindowManager服务引用
WindowManager wm = (WindowManager)getSystemService(getApplication()
        .WINDOW_SERVICE);
```

4. 当 Client 端发送数据到 Server 时，首先通过 BinderProxy 可将请求参数发送给内核，再通过共享内存的方式使用内核方法 copy_from_user() 将参数先拷贝到内核空间，这时的 Client 进入等待状态。而 Server 端（作为数据接收端）与内核共享数据，不再需要拷贝数据，而是通过内存地址空间的偏移量，即可获悉内存地址，从而获取到 Client 发送过来的数据，整个过程只发生一次内存拷贝。

还是 [universus 老师](https://blog.csdn.net/universus)的图：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044308.png)

## <strong>Binder 机制跨进程原理</strong>

上文给出了 Binder 的通讯模型，指出了通讯过程的四个角色: Client、Server、ServiceManager 和 Binder 驱动， 但是我们仍然不清楚 Client 具体是如何与 Server 完成通讯的。

假设 Client 进程想要调用 Server 进程的 object 对象的一个方法 add，对于这个跨进程通讯过程，我们来看看 Binder 是如何做的。 （通讯是一个广泛的概念，只要一个进程能调用另外一个进程里面某对象的方法，那么具体要完成什么通讯内容就很容易了。）

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-1/clipboard_20230323_044311.png)

1. 首先，Server 进程要向 SM 注册，告诉自己是谁，自己有什么能力；在这个场景就是 Server 告诉 SM，它叫 zhangsan，它有一个 object 对象，可以执行 add 操作；于是 SM 在表上新增一条记录：zhangsan 这个名字对应进程 Server。
2. 然后 Client 通过 SM 向上述表中查询：我需要联系一个名字叫做 zhangsan 的进程里面的 object 对象；这时候关键来了：进程之间通信的数据都会经过运行在内核空间里面的驱动，驱动在数据流过的时候做了一点手脚，它并不会给 Client 进程返回一个真正的 object 对象，而是返回一个看起来跟 object 一模一样的代理对象 objectProxy，这个 objectProxy 也有一个 add 方法，但是这个 add 方法没有 Server 进程里面 object 对象的 add 方法那个能力；objectProxy 的 add 只是一个傀儡，它唯一做的事情就是把参数包装然后交给驱动。(这里我们简化了 SM 的流程，见下文)
3. 但是 Client 进程并不知道驱动返回给它的对象动过手脚，毕竟伪装的太像了，如假包换。Client 开开心心地拿着 objectProxy 对象然后调用 add 方法；我们说过，这个 add 什么也不做，直接把参数做一些包装然后直接转发给 Binder 驱动。
4. 驱动收到这个消息，发现是这个 objectProxy；一查表就明白了：我之前用 objectProxy 替换了 object 发送给 Client 了，它真正应该要访问的是 object 对象的 add 方法；于是 Binder 驱动通知 Server 进程：<em>调用你的 object 对象的 add 方法，然后把结果发给我</em>，Sever 进程收到这个消息，照做之后将结果返回驱动，驱动然后把结果返回给 Client 进程；于是整个过程就完成了。

由于驱动返回的 objectProxy 与 Server 进程里面原始的 object 是如此相似，给人感觉好像是<strong>直接把 Server 进程里面的对象 object 传递到了 Client 进程</strong>；因此，我们可以说 <strong>Binder 对象是可以进行跨进程传递的对象。</strong>

但事实上我们知道，Binder 跨进程传输并不是真的把一个对象传输到了另外一个进程；传输过程好像是 Binder 跨进程穿越的时候，它在一个进程留下了一个真身，在另外一个进程幻化出一个影子（这个影子可以很多个）；Client 进程的操作其实是对于影子的操作，影子利用 Binder 驱动最终让真身完成操作。

理解这一点非常重要，务必仔细体会。另外，Android 系统实现这种机制使用的是<em>代理模式</em>, 对于 Binder 的访问，如果是在同一个进程（不需要跨进程），那么直接返回原始的 Binder 实体；如果在不同进程，那么就给他一个代理对象（影子）；我们在系统源码以及 AIDL 的生成代码里面可以看到很多这种实现，在下一篇文章中会进行详细分析。

另外我们为了简化整个流程，隐藏了 SM 这一部分驱动进行的操作；实际上，由于 SM 与 Server 通常不在一个进程，Server 进程向 SM 注册的过程也是跨进程通信，驱动也会对这个过程进行暗箱操作：SM 中存在的 Server 端的对象实际上也是代理对象，后面 Client 向 SM 查询的时候，驱动会给 Client 返回另外一个代理对象。Sever 进程的本地对象仅有一个，其他进程所拥有的全部都是它的代理。

## 小结

现在我们可以对 Binder 做个更加全面的定义了：

- 从进程间通信的角度看，Binder 是一种进程间通信的机制；
- 从 Server 进程的角度看，Binder 指的是 Server 中的 Binder 本地对象；
- 从 Client 进程的角度看，Binder 指的是对 Binder 代理对象，是 Binder 实体对象的一个远程代理；
- 从传输过程的角度看，Binder 是一个可以跨进程传输的对象；Binder 驱动会对这个跨越进程边界的对象做一点点特殊处理，自动完成代理对象和本地对象之间的转换。

一句话总结就是：<strong>Client 进程只不过是持有了 Server 端的代理对象；代理对象协助驱动完成了跨进程通信。</strong>

# 总结

进程隔离虽然使操作系统的安全性和应用程序的稳定性得到了提升，但同时也给 IPC 带来了一定的难度，Android 系统巧妙地应用了 Binder 机制，使得系统得以在存储空间和硬件性能等有限的移动设备上能够流畅地运行。关于 Binder 在应用层的使用和分析，请看下一篇文章内容：[二、Binder 机制分析——应用篇](https://ywue4d2ujm.feishu.cn/docs/doccnmOm0Lqi9mmFqMhwuQYxvUd)

<strong>参考文章</strong>

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

[Binder 学习指南](https://cloud.tencent.com/developer/article/1329601)

> 如果你对文章内容有疑问或者有不同意见，欢迎留言，我们一同探讨。
