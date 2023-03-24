---
title: OkHttp 使用及源码分析
date created: 2023-03-23
date modified: 2023-03-24
---

[https://juejin.cn/post/6881436122950402056](https://juejin.cn/post/6881436122950402056)

# 请求流程

## <strong>同步请求</strong>

<em>MainActivity.kt</em>

```kotlin
val user = "guanpj"

val client = OkHttpClient.Builder()
    .connectTimeout(15, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .proxy(Proxy.NO_PROXY)
    .addInterceptor(HttpLoggingInterceptor { message ->
        if (BuildConfig.DEBUG) {
            Log.i("OkHttp", message)
        }
    })
    .build()
    
val request: Request = Request.Builder()
    .url("https://api.github.com/users/$user/repos")
    .build()

val response = client.newCall(request).execute()

println("Response status code: ${response.code}")
```

<em>OkHttpClient.kt</em>

```kotlin
/** Prepares the [request] to be executed at some point in the future. */
override fun newCall(request: Request): Call =
         RealCall(this, request, forWebSocket = false)
```

<em>RealCall.execute</em>

```kotlin
override fun execute(): Response {
  //保证每个 call 只能被执行一次
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  timeout.enter()
  callStart()
  try {
    client.dispatcher.executed(this)
    return getResponseWithInterceptorChain()
  } finally {
    client.dispatcher.finished(this)
  }
}
```

<em>Dispatcher.executed</em>

```kotlin
@Synchronized internal fun executed(call: RealCall) {
  runningSyncCalls.add(call)
}
```

可以看到，在 extcuted 方法中并没有具体执行请求的方法，仅仅是把当前对象存入 runningSyncCalls 集合中。同步请求的核心部分在接下来的 getResponseWithInterceptorChain 中，此方法直接返回了请求结果 Response，事实上这个方法调用栈非常复杂，这里先按下不表。最后在 finially 代码块中调用了 `client.dispatcher.finished(this)`，也是传入了 this 对象，我们有理由猜测这一步就是将它从 runningSyncCalls 集合中移除，这里同样等到异步请求流程一并分析。

## <strong>异步请求</strong>

<em>MainActivity.kt</em>

```kotlin
val user = "guanpj"

val client = OkHttpClient.Builder()
    .build()
val request: Request = Request.Builder()
    .url("https://api.github.com/users/$user/repos")
    .build()

client.newCall(request)
    .enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            e.printStackTrace()
        }

        override fun onResponse(call: Call, response: Response) {
            println("Response status code: ${response.code}")
        }
    })
```

<em>RealCall.enqueue</em>

```kotlin
override fun enqueue(responseCallback: Callback) {
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  callStart()
  //包装成 AsyncCall，放入待执行的异步请求请求列表
  client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

<em>Dispatcher.enqueue</em>

```kotlin
internal fun enqueue(call: AsyncCall) {
  synchronized(this) {
    //首先将 call 放入 readyAsyncCalls 集合
    readyAsyncCalls.add(call)

    // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
    // the same host.
    if (!call.call.forWebSocket) {
      val existingCall = findExistingCallWithHost(call.host)
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
    }
  }
  //然后调用此方法执行
  promoteAndExecute()
}
```

readyAsyncCalls 意思温为“准备好的异步请求”，而 promoteAndExecute 方法单从命名来看就是提升和执行，具体分析如下：

<em>Dispatcher.promoteAndExecute</em>

```kotlin
private fun promoteAndExecute(): Boolean {
  this.assertThreadDoesntHoldLock()
  //新建一个列表存放可执行的请求
  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    val i = readyAsyncCalls.iterator()
    //遍历准备好的异步请求
    while (i.hasNext()) {
      val asyncCall = i.next()
      //总请求数不能超过 64 个
      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      //每个主机请求数量不能超过 5 个
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.
      //从 readyAsyncCalls 集合中移除
      i.remove()
      //安全自增
      asyncCall.callsPerHost.incrementAndGet()
      //加入到 executableCalls 集合
      executableCalls.add(asyncCall)
      //加入到 runningAsyncCalls 集合
      runningAsyncCalls.add(asyncCall)
    }
    //获取正在执行中的请求（同步和异步）总数
    isRunning = runningCallsCount() > 0
  }
  //遍历可执行请求的集合
  for (i in 0 until executableCalls.size) {
    val asyncCall = executableCalls[i]
    //调用它们的 executeOn 方法
    asyncCall.executeOn(executorService)
  }
  //返回正在执行中的请求（同步和异步）总数
  return isRunning
}

@Synchronized fun runningCallsCount(): Int 
        = runningAsyncCalls.size + runningSyncCalls.size
```

可以看到，该方法可拆分为三个步骤：

1. 首先，一个请求从上一步被放入 readyAsyncCalls 集合中，在这里将遍历该集合并在条件允许的情况下将集合中的请求移除并分别加入到 runningAsyncCalls 和 executableCalls 集合；
2. 接着遍历 executableCalls 集合并调用每个 AysncCall 的 executeOn 方法，并传入 executorService 对像用于执行这个 Call。AsyncCall 是 RealCall 中的一个内部类；
3. 最后返回正在执行中的同步和异步请求的总数。

这里重点看第二点：

<em>RealCall.AsyncCall</em>

```kotlin
internal inner class AsyncCall(
  private val responseCallback: Callback
) : Runnable {
  @Volatile var callsPerHost = AtomicInteger(0)
    private set
  fun reuseCallsPerHostFrom(other: AsyncCall) {
    this.callsPerHost = other.callsPerHost
  }
  
  val host: String
    get() = originalRequest.url.host
  val request: Request
      get() = originalRequest
  val call: RealCall
      get() = this@RealCall
  /**
   * Attempt to enqueue this async call on [executorService]. This will attempt to clean up
   * if the executor has been shut down by reporting the call as failed.
   */
  fun executeOn(executorService: ExecutorService) {
    client.dispatcher.assertThreadDoesntHoldLock()
    var success = false
    try {
      //调用 executorService 执行自己
      executorService.execute(this)
      success = true
    } catch (e: RejectedExecutionException) {
      val ioException = InterruptedIOException("executor rejected")
      ioException.initCause(e)
      noMoreExchanges(ioException)
      responseCallback.onFailure(this@RealCall, ioException)
    } finally {
      if (!success) {
        client.dispatcher.finished(this) // This call is no longer running!
      }
    }
  }
  
  //被 executorService.execute 时调用
  override fun run() {
    threadName("OkHttp ${redactedUrl()}") {
      var signalledCallback = false
      timeout.enter()
      try {
        //1、在 executorService 中异步执行
        val response = getResponseWithInterceptorChain()
        signalledCallback = true
        //2、回调结果
        responseCallback.onResponse(this@RealCall, response)
      } catch (e: IOException) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
        } else {
          //回调结果
          responseCallback.onFailure(this@RealCall, e)
        }
      } catch (t: Throwable) {
        cancel()
        if (!signalledCallback) {
          val canceledException = IOException("canceled due to $t")
          canceledException.addSuppressed(t)
          //回调结果
          responseCallback.onFailure(this@RealCall, canceledException)
        }
        throw t
      } finally {
        //3、后续操作
        client.dispatcher.finished(this)
      }
    }
  }
}
```

因为是异步请求，所以请求操作必然是在异步线程中执行的，可以看到，在 run 方法中，异步请求正常情况下分为三个步骤：

1. 调用 getResponseWithInterceptorChain 方法获取 Resonse 结果，这点和同步请求一样。
2. 将 Response 对象通过 responseCallback 进行回调。
3. 在 finally 中调用 `client.dispatcher.finished(this)` 方法。

第一步还是留到后面，第二步无须多解释，来看第三步，在同步请求流程的足后也同样执行了这个步骤。

<em>Dispatcher.finish</em>

```kotlin
internal fun finished(call: AsyncCall) {
  call.callsPerHost.decrementAndGet()
  finished(runningAsyncCalls, call)
}

internal fun finished(call: RealCall) {
  finished(runningSyncCalls, call)
}

private fun <T> finished(calls: Deque<T>, call: T) {
  val idleCallback: Runnable?
  synchronized(this) {
    if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
    idleCallback = this.idleCallback
  }

  val isRunning = promoteAndExecute()

  if (!isRunning && idleCallback != null) {
    idleCallback.run()
  }
}
```

不管同步还是异步的 Call，最后都会走到最后一个方法，果然这里分别移除了 runningAsyncCalls 和 runningSyncCalls 集合中的 Call 对象。除此之外，这里再次调用了 promoteAndExecute 方法，前面提到过，这里返回的是正在执行中的同步和异步请求个数。当所有请求都执行完成后，会调用 idleCallback.run 方法进行回调。

## getResponseWithInterceptorChain

```kotlin
internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = mutableListOf<Interceptor>()
  interceptors += client.interceptors
  interceptors += RetryAndFollowUpInterceptor(client)
  interceptors += BridgeInterceptor(client.cookieJar)
  interceptors += CacheInterceptor(client.cache)
  interceptors += ConnectInterceptor
  if (!forWebSocket) {
    interceptors += client.networkInterceptors
  }
  interceptors += CallServerInterceptor(forWebSocket)
  
  val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
  )
  
  var calledNoMoreExchanges = false
  try {
    val response = chain.proceed(originalRequest)
    if (isCanceled()) {
      response.closeQuietly()
      throw IOException("Canceled")
    }
    return response
  } catch (e: IOException) {
    calledNoMoreExchanges = true
    throw noMoreExchanges(e) as Throwable
  } finally {
    if (!calledNoMoreExchanges) {
      noMoreExchanges(null)
    }
  }
}
```

首先将以下拦截器依次加入到 List 中：

1. OkHttpClient 设置的拦截器 interceptors()
2. 重试、重定向拦截器 RetryAndFollowUpInterceptor
3. 把用户请求转换为服务器请求、把服务器返响应转换为用户响应的 BridgeInterceptor
4. 读取缓存直接返回、将响应写入到缓存中的 CacheInterceptor
5. 与服务器建立连接的 ConnectInterceptor
6. OkHttpClient 设置的网络拦截器 networkInterceptors()
7. 真正执行网络请求的 CallServerInterceptor

将所有的拦截器保存在 interceptors 集合中后，创建一个拦截器责任链 RealInterceptorChain，并调用其 proceed 开始处理网络请求。那么责任链模式是如何工作的呢？

首先查看 RealInterceptorChain 的源码：

```kotlin
class RealInterceptorChain(
  internal val call: RealCall,
  private val interceptors: List<Interceptor>,
  private val index: Int,
  internal val exchange: Exchange?,
  internal val request: Request,
  internal val connectTimeoutMillis: Int,
  internal val readTimeoutMillis: Int,
  internal val writeTimeoutMillis: Int
) : Interceptor.Chain {

  private var calls: Int = 0

  internal fun copy(
    index: Int = this.index,
    exchange: Exchange? = this.exchange,
    request: Request = this.request,
    connectTimeoutMillis: Int = this.connectTimeoutMillis,
    readTimeoutMillis: Int = this.readTimeoutMillis,
    writeTimeoutMillis: Int = this.writeTimeoutMillis
  ) = RealInterceptorChain(call, interceptors, index, exchange, request, connectTimeoutMillis,
      readTimeoutMillis, writeTimeoutMillis)

  ...

  @Throws(IOException::class)
  override fun proceed(request: Request): Response {
    check(index < interceptors.size)

    calls++

    ...

    // Call the next interceptor in the chain.
    val next = copy(index = index + 1, request = request)
    val interceptor = interceptors[index]

    @Suppress("USELESS_ELVIS")
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")

    ...

    return response
  }
}
```

在不考虑 OkHttpClient.interceptor() 的情况下，首次执行 getResponseWithInterceptorChain 方法时上面这两段代码的解释如下：

1. 在 getResponseWithInterceptorChain 创建了一个 index 为 0 的 RealInterceptorChain，接着调用了其 proceed 方法；
2. 在 RealInterceptorChain.proceed 方法中，将 index+1 并且和其它成员变量一起 copy 出一个新的 RealInterceptorChain 对象 next；
3. 然后对当前 index 的拦截器（即 RetryAndFollowUpInterceptor）执行 interceptor.intercept(next)。在 RetryAndFollowUpInterceptor 方法中执行了 next.proceed 方法，而这里的 next 同样是 RealInterceptorChain 实例，所以回到了 RealInterceptorChain.proceed 方法中；
4. 此时 index=1，同理链条可以一直执行下去直到 index 等于 n-1；
5. 遇到最后一个拦截器 CallServerInterceptor，链不能继续下去了，CallServerInterceptor.intercept 方法中也不会再 proceed 了；
6. CallServerInterceptor 建立连接后开始递归返回，Response 的返回与 Request 相反，会从最后一个开始依次往前经过这些 Intercetor。

下图为 OkHttp 工作的大致流程，参考自 [拆轮子系列：拆 OkHttp](https://blog.piasy.com/2016/07/11/Understand-OkHttp/index.html)

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/OkHttp/clipboard_20230323_035335.png)

同步请求和异步请求流程时序图如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/OkHttp/clipboard_20230323_035341.png)

# OkHttpClient

OkHttpClient 相当于配置中心，所有的请求都会共享这些配置（例如出错是否重试、共享的连接池）。 OkHttpClient 中的配置主要有:

- `Dispatcher dispatcher` : 调度器，用于调度后台发起的网络请求， 有后台总请求数和单主机总请求数的控制。
- `List<Protocol> protocols` : 支持的应用层协议，即 HTTP/1.1、 HTTP/2 等。
- `List<ConnectionSpec> connectionSpecs` : 应用层支持的 Socket 设置，即使用明文传输（用于 HTTP）还是某个版本的 TLS（用于 HTTPS）。
- `List<Interceptor> interceptors` : 大多数时候使用的 Interceptor 都应该配置到这里。
- `List<Interceptor> networkInterceptors` : 直接和网络请求交互的 Interceptor 配置到这里，例如如果要查看返回的 301 报文或者未解压的 Response Body，需要在这里看。
- CookieJar cookieJar : 管理 Cookie 的控制器。OkHttp 提供了 Cookie 存取的判断支持（即什么时候需要存 Cookie，什么时候需要读取 Cookie，但没有给出具体的存取实现。如果需要存取 Cookie，需要自己写实现，例如用 Map 存在内存里，或者用别的方式存在本地存储或者数据库。
- Cache cache : Cache 存储的配置。默认是没有，如果需要用，得自己配置出 Cache 存储的文件位置以及存储空间上限。
- HostnameVerifier hostnameVerifier : 用于验证 HTTPS 握手过程中下载到的证书所属者是否和自己要访问的主机名一致。
- CertificatePinner certificatePinner : 用于设置 HTTPS 握手 过程中针对某个 Host 额外的的 Certificate Public Key Pinner，即把网站证书链中的每一个证书公钥直接拿来提前配置进 OkHttpClient 里去，作为正常的证书验证机制之外的一次额外验证。
- Authenticator authenticator : 用于自动重新认证。配置之后，在请求收到 401 状态码的响应时，会直接调用 authenticator ，手动加入 Authorization header 之后自动重新发起请求。
- boolean followRedirects : 遇到重定向的要求是，是否自动 follow。
- boolean followSslRedirects : 在重定向时，如果原先请求的是 http 而重定向的目标是 https，或者原先请求的是 https 而重定向的目标是 http，是否依然自动 follow。注意：不是「是否自动 follow HTTPS URL 重定向的意思，而是是否自动 follow 在 HTTP 和 HTTPS 之间切换的重定向。
- boolean retryOnConnectionFailure : 在请求失败的时候是否自动重试。注意：大多数的请求失败并不属于 OkHttp 所定义的「需要重试」， 这种重试只适用于「同一个域名的多个 IP 切换重试」「Socket 失效重试」 等情况。
- int connectTimeout : 建立连接（TCP 或 TLS）的超时时间。
- int readTimeout : 发起请求到读到响应数据的超时时间。
- int writeTimeout : 发起请求并被目标服务器接受的超时时间。（因为有时候对方服务器可能由于某种原因而不读取你的 Request）

# <strong>Request 和 Response</strong>

Request 是发送请求封装类，内部有 url, header , method，body 等常见的参数。

```kotlin
class Request internal constructor(
  @get:JvmName("url") val url: HttpUrl,
  @get:JvmName("method") val method: String,
  @get:JvmName("headers") val headers: Headers,
  @get:JvmName("body") val body: RequestBody?,
  internal val tags: Map<Class<*>, Any>
) {
  ...
}
```

Response 是请求的结果，包含 code、message、header、body:

```kotlin
class Response internal constructor(
  /**
   * The wire-level request that initiated this HTTP response. This is not necessarily the same
   * request issued by the application:
   *
   * * It may be transformed by the HTTP client. For example, the client may copy headers like
   *   `Content-Length` from the request body.
   * * It may be the request generated in response to an HTTP redirect or authentication
   *   challenge. In this case the request URL may be different than the initial request URL.
   */
  @get:JvmName("request") val request: Request,

  /** Returns the HTTP protocol, such as [Protocol.HTTP_1_1] or [Protocol.HTTP_1_0]. */
  @get:JvmName("protocol") val protocol: Protocol,

  /** Returns the HTTP status message. */
  @get:JvmName("message") val message: String,

  /** Returns the HTTP status code. */
  @get:JvmName("code") val code: Int,

  /**
   * Returns the TLS handshake of the connection that carried this response, or null if the
   * response was received without TLS.
   */
  @get:JvmName("handshake") val handshake: Handshake?,

  /** Returns the HTTP headers. */
  @get:JvmName("headers") val headers: Headers,

  /**
   * Returns a non-null value if this response was passed to [Callback.onResponse] or returned
   * from [Call.execute]. Response bodies must be [closed][ResponseBody] and may
   * be consumed only once.
   *
   * This always returns null on responses returned from [cacheResponse], [networkResponse],
   * and [priorResponse].
   */
  @get:JvmName("body") val body: ResponseBody?,

  /**
   * Returns the raw response received from the network. Will be null if this response didn't use
   * the network, such as when the response is fully cached. The body of the returned response
   * should not be read.
   */
  @get:JvmName("networkResponse") val networkResponse: Response?,

  /**
   * Returns the raw response received from the cache. Will be null if this response didn't use
   * the cache. For conditional get requests the cache response and network response may both be
   * non-null. The body of the returned response should not be read.
   */
  @get:JvmName("cacheResponse") val cacheResponse: Response?,

  /**
   * Returns the response for the HTTP redirect or authorization challenge that triggered this
   * response, or null if this response wasn't triggered by an automatic retry. The body of the
   * returned response should not be read because it has already been consumed by the redirecting
   * client.
   */
  @get:JvmName("priorResponse") val priorResponse: Response?,

  /**
   * Returns a [timestamp][System.currentTimeMillis] taken immediately before OkHttp
   * transmitted the initiating request over the network. If this response is being served from the
   * cache then this is the timestamp of the original request.
   */
  @get:JvmName("sentRequestAtMillis") val sentRequestAtMillis: Long,

  /**
   * Returns a [timestamp][System.currentTimeMillis] taken immediately after OkHttp
   * received this response's headers from the network. If this response is being served from the
   * cache then this is the timestamp of the original response.
   */
  @get:JvmName("receivedResponseAtMillis") val receivedResponseAtMillis: Long,

  @get:JvmName("exchange") internal val exchange: Exchange?
) : Closeable {
  ...
}
```

这两个类的定义是完全符合 Http 协议所定义的请求内容和响应内容。

# RealCall

OkHttpClient 的 newCall(Request) 方法会返回一个 RealCall 对象，它是 Call 接口的实现。

当调用 RealCall.execute() 的时候， RealCall.getResponseWithInterceptorChain() 会被调用，它会发起网络请求并拿到返回的响应，装进一个 Response 对象并作为返回值返回；

RealCall.enqueue() 被调用的时候大同小异，区别在于 enqueue() 会使用 Dispatcher 的线程池来把请求放在后台线程进行，但实质上使用的也是 getResponseWithInterceptorChain() 方法。

getResponseWithInterceptorChain() 方法做的事：把所有配置好的 Interceptor 放在一个 List 里，然后作为参数，创建一个 RealInterceptorChain 对象，并调 chain.proceed(request) 来发起请求和获取响应。

```kotlin
internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = mutableListOf<Interceptor>()
  interceptors += client.interceptors
  interceptors += RetryAndFollowUpInterceptor(client)
  interceptors += BridgeInterceptor(client.cookieJar)
  interceptors += CacheInterceptor(client.cache)
  interceptors += ConnectInterceptor
  if (!forWebSocket) {
    interceptors += client.networkInterceptors
  }
  interceptors += CallServerInterceptor(forWebSocket)
  
  val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
  )
  ...
}
```

在 RealInterceptorChain 中，多个 Interceptor 会依次调用自己的 intercept() 方法。这个方法会做三件事:

1. 对请求进行预处理
2. 预处理之后，重新调用 RealIntercepterChain.proceed() 把请求交给下一个 Interceptor
3. 在下一个 Interceptor 处理完成并返回之后，拿到 Response 进行后续处理

结合源码和该示意图，可以得到拦截器具有如下特点：

- 拦截器按照添加顺序依次执行
- 拦截器的执行从 RealInterceptorChain.proceed() 开始，进入到第一个拦截器的执行逻辑
- 每个拦截器在执行之前，会将剩余尚未执行的拦截器组成新的 RealInterceptorChain
- 拦截器的逻辑被新的责任链调用 next.proceed() 切分为 start、next.proceed、end 这三个部分依次执行
- next.proceed() 所代表的其实就是剩余所有拦截器的执行逻辑
- 所有拦截器最终形成一个层层内嵌的嵌套结构

# Interceptor

上面提到过，OkHttp 的所有拦截器会组成链式结构：各个拦截器完成前置工作后调用下一个拦截器的 proceed 方法，再执行后置工作。通过递归调用，每个拦截器完成各自的任务。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/OkHttp/clipboard_20230323_035349.png)

## client.interceptors

首先是开发者使用 addInterceptor(Interceptor) 所设置的，它们会按照开发者的要求，在所有其他 Interceptor 处理之前，进行最早的预处理工作，以及在收到 Response 之后，做最后的善后工作。如果你有统一的 header 要添加，可以在这里设置。

## RetryAndFollowUpInterceptor

它会对连接做一些初始化工作，并且负责在请求失败时的重试，以及重定向的自动后续请求。它的存在，可以让<strong>重试和重定向</strong>对于开发者是无感知的。

RetryAndFollowUpInterceptor.intercept 代码如下：

```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  var request = chain.request
  val call = realChain.call
  var followUpCount = 0
  var priorResponse: Response? = null
  var newExchangeFinder = true
  var recoveredFailures = listOf<IOException>()
  while (true) {
    call.enterNetworkInterceptorExchange(request, newExchangeFinder)

    var response: Response
    var closeActiveExchange = true
    try {
      //先判断请求是否已被取消
      if (call.isCanceled()) {
        throw IOException("Canceled")
      }

      try {
        //执行下一个链的 proceed
        response = realChain.proceed(request)
        newExchangeFinder = true
      } catch (e: RouteException) {
        //判断是否能恢复，如果能则继续，否则结束
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
          throw e.firstConnectException.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e.firstConnectException
        }
        newExchangeFinder = false
        continue
      } catch (e: IOException) {
        // An attempt to communicate with a server failed. The request may have been sent.
        if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
          throw e.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e
        }
        newExchangeFinder = false
        continue
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
            .build()
      }

      val exchange = call.interceptorScopedExchange
      val followUp = followUpRequest(response, exchange)

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex) {
          call.timeoutEarlyExit()
        }
        closeActiveExchange = false
        return response
      }

      val followUpBody = followUp.body
      if (followUpBody != null && followUpBody.isOneShot()) {
        closeActiveExchange = false
        //返回
        return response
      }

      response.body?.closeQuietly()

      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
      }

      request = followUp
      priorResponse = response
    } finally {
      call.exitNetworkInterceptorExchange(closeActiveExchange)
    }
  }
}

private fun recover(
  e: IOException,
  call: RealCall,
  userRequest: Request,
  requestSendStarted: Boolean
): Boolean {
  // The application layer has forbidden retries.
  if (!client.retryOnConnectionFailure) return false

  // We can't send the request body again.
  if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

  // This exception is fatal.
  if (!isRecoverable(e, requestSendStarted)) return false

  // No more routes to attempt.
  if (!call.retryAfterFailure()) return false

  // For failure recovery, use the same route selector with a new connection.
  return true
}


@Throws(IOException::class)
private fun followUpRequest(userResponse: Response, exchange: Exchange?): Request? {
  val route = exchange?.connection?.route()
  val responseCode = userResponse.code

  val method = userResponse.request.method
  when (responseCode) {
    HTTP_PROXY_AUTH -> {
      val selectedProxy = route!!.proxy
      if (selectedProxy.type() != Proxy.Type.HTTP) {
        throw ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy")
      }
      return client.proxyAuthenticator.authenticate(route, userResponse)
    }

    HTTP_UNAUTHORIZED -> return client.authenticator.authenticate(route, userResponse)

    HTTP_PERM_REDIRECT, HTTP_TEMP_REDIRECT, HTTP_MULT_CHOICE, HTTP_MOVED_PERM, HTTP_MOVED_TEMP, HTTP_SEE_OTHER -> {
      return buildRedirectRequest(userResponse, method)
    }

    HTTP_CLIENT_TIMEOUT -> {
      // 408's are rare in practice, but some servers like HAProxy use this response code. The
      // spec says that we may repeat the request without modifications. Modern browsers also
      // repeat the request (even non-idempotent ones.)
      if (!client.retryOnConnectionFailure) {
        // The application layer has directed us not to retry the request.
        return null
      }

      val requestBody = userResponse.request.body
      if (requestBody != null && requestBody.isOneShot()) {
        return null
      }
      val priorResponse = userResponse.priorResponse
      if (priorResponse != null && priorResponse.code == HTTP_CLIENT_TIMEOUT) {
        // We attempted to retry and got another timeout. Give up.
        return null
      }

      if (retryAfter(userResponse, 0) > 0) {
        return null
      }

      return userResponse.request
    }

    HTTP_UNAVAILABLE -> {
      val priorResponse = userResponse.priorResponse
      if (priorResponse != null && priorResponse.code == HTTP_UNAVAILABLE) {
        // We attempted to retry and got another timeout. Give up.
        return null
      }

      if (retryAfter(userResponse, Integer.MAX_VALUE) == 0) {
        // specifically received an instruction to retry without delay
        return userResponse.request
      }

      return null
    }

    HTTP_MISDIRECTED_REQUEST -> {
      // OkHttp can coalesce HTTP/2 connections even if the domain names are different. See
      // RealConnection.isEligible(). If we attempted this and the server returned HTTP 421, then
      // we can retry on a different connection.
      val requestBody = userResponse.request.body
      if (requestBody != null && requestBody.isOneShot()) {
        return null
      }

      if (exchange == null || !exchange.isCoalescedConnection) {
        return null
      }

      exchange.connection.noCoalescedConnections()
      return userResponse.request
    }

    else -> return null
  }
}
```

这个过程的大致流程如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/OkHttp/clipboard_20230323_035354.png)

从上图中可以看出，RetryAndFollowUpInterceptor 开启了一个 while(true) 的循环，并在循环内部完成两个重要的判定，如图中的蓝色方框：

1. 当请求内部抛出异常时，判定是否需要重试
2. 当响应结果是 3xx 重定向时，构建新的请求并发送请求

重试的逻辑相对复杂，有如下的判定逻辑（具体代码在 RetryAndFollowUpInterceptor 类的 recover 方法）：

- 规则 1: client 的 retryOnConnectionFailure 参数设置为 false，不进行重试
- 规则 2: 请求的 body 已经发出，不进行重试
- 规则 3: 特殊的异常类型不进行重试（如 ProtocolException，SSLHandshakeException 等）
- 规则 4: 没有更多的 route（包含 proxy 和 inetaddress），不进行重试

## BridgeInterceptor

负责一些不影响开发者开发，但影响 HTTP 交互的一些额外预处理。

```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
  val userRequest = chain.request()
  val requestBuilder = userRequest.newBuilder()

  val body = userRequest.body
  if (body != null) {
    val contentType = body.contentType()
    if (contentType != null) {
      requestBuilder.header("Content-Type", contentType.toString())
    }

    val contentLength = body.contentLength()
    if (contentLength != -1L) {
      requestBuilder.header("Content-Length", contentLength.toString())
      requestBuilder.removeHeader("Transfer-Encoding")
    } else {
      requestBuilder.header("Transfer-Encoding", "chunked")
      requestBuilder.removeHeader("Content-Length")
    }
  }

  if (userRequest.header("Host") == null) {
    requestBuilder.header("Host", userRequest.url.toHostHeader())
  }

  if (userRequest.header("Connection") == null) {
    requestBuilder.header("Connection", "Keep-Alive")
  }

  // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
  // the transfer stream.
  var transparentGzip = false
  if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
    transparentGzip = true
    requestBuilder.header("Accept-Encoding", "gzip")
  }

  val cookies = cookieJar.loadForRequest(userRequest.url)
  if (cookies.isNotEmpty()) {
    requestBuilder.header("Cookie", cookieHeader(cookies))
  }

  if (userRequest.header("User-Agent") == null) {
    requestBuilder.header("User-Agent", userAgent)
  }

  val networkResponse = chain.proceed(requestBuilder.build())

  //收到响应后，存储 Cookie：
  cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

  val responseBuilder = networkResponse.newBuilder()
      .request(userRequest)

  if (transparentGzip &&
      "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
      networkResponse.promisesBody()) {
    val responseBody = networkResponse.body
    if (responseBody != null) {
      val gzipSource = GzipSource(responseBody.source())
      val strippedHeaders = networkResponse.headers.newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build()
      responseBuilder.headers(strippedHeaders)
      val contentType = networkResponse.header("Content-Type")
      responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
    }
  }

  return responseBuilder.build()
}
```

BridgeInterceptor 拦截器的功能如下：

1. 负责把用户构造的请求转换为发送到服务器的请求 、把服务器返回的响应转换为用户友好的响应，是从应用程序代码到网络代码的桥梁
2. 设置内容长度，内容编码
3. 设置 gzip 压缩，并在接收到内容后进行解压。省去了应用层处理数据解压的麻烦
4. 添加 cookie
5. 设置其他报头，如 Content-Length、User-Agent、Host、Keep-alive 等。其中 Keep-Alive 是实现连接复用的必要步骤

## CacheInterceptor

负责 Cache 的处理。把它放在后面的网络交互相关 Interceptor 的前面的好处是，如果本地有了可用的 Cache，一个请求可以在没有发生实质网络交互的情况下就返回缓存结果，而完全不需要开发者做出任何的额外工作，让 Cache 更加无感知。

```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
  val call = chain.call()
  val cacheCandidate = cache?.get(chain.request())

  val now = System.currentTimeMillis()

  val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
  val networkRequest = strategy.networkRequest
  val cacheResponse = strategy.cacheResponse

  cache?.trackResponse(strategy)
  val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

  if (cacheCandidate != null && cacheResponse == null) {
    // The cache candidate wasn't applicable. Close it.
    cacheCandidate.body?.closeQuietly()
  }

  // If we're forbidden from using the network and the cache is insufficient, fail.
  if (networkRequest == null && cacheResponse == null) {
    return Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(HTTP_GATEWAY_TIMEOUT)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build().also {
          listener.satisfactionFailure(call, it)
        }
  }

  // If we don't need the network, we're done.
  if (networkRequest == null) {
    return cacheResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build().also {
          listener.cacheHit(call, it)
        }
  }

  if (cacheResponse != null) {
    listener.cacheConditionalHit(call, cacheResponse)
  } else if (cache != null) {
    listener.cacheMiss(call)
  }

  // 没有命中缓存
  var networkResponse: Response? = null
  try {
    networkResponse = chain.proceed(networkRequest)
  } finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
      cacheCandidate.body?.closeQuietly()
    }
  }

  // 如果缓存策略中，网络响应和响应缓存都不为null，需要更新响应缓存
  // (比如 需要向服务器确认缓存是否可用的情况)
  // If we have a cache response too, then we're doing a conditional get.
  if (cacheResponse != null) {
    // 304 的返回是不带 body 的,此时必须获取 cache 的 body
    if (networkResponse?.code == HTTP_NOT_MODIFIED) {
      val response = cacheResponse.newBuilder()
          .headers(combine(cacheResponse.headers, networkResponse.headers))
          .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
          .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
          .cacheResponse(stripBody(cacheResponse))
          .networkResponse(stripBody(networkResponse))
          .build()

      networkResponse.body!!.close()

      // Update the cache after combining headers but before stripping the
      // Content-Encoding header (as performed by initContentStream()).
      cache!!.trackConditionalCacheHit()
      cache.update(cacheResponse, response)
      return response.also {
        listener.cacheHit(call, it)
      }
    } else {
      cacheResponse.body?.closeQuietly()
    }
  }

  // 构建响应对象，等待返回
  val response = networkResponse!!.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build()

  if (cache != null) {
    // 将请求放到缓存中
    if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
      val cacheRequest = cache.put(response)
      return cacheWritingResponse(cacheRequest, response).also {
        if (cacheResponse != null) {
          // This will log a conditional cache miss only.
          listener.cacheMiss(call)
        }
      }
    }

    // 如果请求不能被缓存，则移除该请求
    if (HttpMethod.invalidatesCache(networkRequest.method)) {
      try {
        cache.remove(networkRequest)
      } catch (_: IOException) {
        // The cache cannot be written.
      }
    }
  }

  return response
}
```

这个过程的流程如下：

1. 通过 Request 尝试到 Cache 中拿缓存，当然前提是 OkHttpClient 中配置了缓存，默认是不支持的。
2. 根据 response、time、request 创建一个缓存策略，用于判断怎样使用缓存。
3. 如果缓存策略中设置禁止使用网络，并且缓存又为空，则构建一个 Response 直接返回（返回码为 504）
4. 缓存策略中设置不使用网络，但是又缓存，直接返回缓存
5. 接着走后续过滤器的流程，chain.proceed(networkRequest)
6. 当缓存存在的时候，如果网络返回的 Response 为 304，则使用缓存的 Response。
7. 构建网络请求的 Response
8. 当在 OkHttpClient 中配置了缓存，则将这个 Response 缓存起来。
9. 缓存起来的步骤也是先缓存 header，再缓存 body。
10. 返回 Response

## ConnectInterceptor

负责建立连接，它的源码如下：

```kotlin
object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.call.initExchange(chain)
    val connectedChain = realChain.copy(exchange = exchange)
    //开始下一个拦截器的工作
    return connectedChain.proceed(realChain.request)
  }
}
```

这个类本身很简单，从源码来看，关键的代码只有注释 1 处的那一句。它的作用是：从 RealCall 中获得了一个新的 ExChange 的对象。跟踪它的执行，可以发现它有如下调用逻辑：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/OkHttp/clipboard_20230323_035402.png)

在这个步骤中，ConnectInterceptor 最终会通过 Socket 创建出网络请求所需要的 TCP 连接（如果是 HTTP），或者是建立在 TCP 连接之上的 TLS 连接（如果是 HTTPS）。并且会创建出对应的 HttpCodec 对象 （用于编码解码 HTTP 请求），并且最终封装成 Exchange 对象并返回。

## client.networkInterceptors

client.networkInterceptors 是由开发者使用 addNetworkInterceptor(Interceptor) 所设置的，它们的行为逻辑和使用 addInterceptor(Interceptor) 创建的一样，但由于位置不同，所以这里创建的 Interceptor 会看到每个请求和响应的数据（包括重定向以及重试的一些中间请求和响应），并且看到的是完整原始数据，而不是没有加 Content-Length 的请求数据，或者 Body 还没有被 gzip 解压的响应数据。多数情况，这个方法不需要被使用，不过如果需要做网络调试，可以用它来实现。

## CallServerInterceptor

CalllServerInterceptor 是最后一个拦截器了，它负责实质的请求与响应的 I/O 操作，即：往 Socket 里写入请求数据和从 Socket 里读取响应数据，进行 http 请求报文的封装与请求报文的解析。

```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  val exchange = realChain.exchange!!
  val request = realChain.request
  val requestBody = request.body
  val sentRequestMillis = System.currentTimeMillis()

  exchange.writeRequestHeaders(request)

  var invokeStartEvent = true
  var responseBuilder: Response.Builder? = null
  if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
    // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
    // Continue" response before transmitting the request body. If we don't get that, return
    // what we did get (such as a 4xx response) without ever transmitting the request body.
    if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
      exchange.flushRequest()
      responseBuilder = exchange.readResponseHeaders(expectContinue = true)
      exchange.responseHeadersStart()
      invokeStartEvent = false
    }
    if (responseBuilder == null) {
      if (requestBody.isDuplex()) {
        // Prepare a duplex body so that the application can send a request body later.
        exchange.flushRequest()
        val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
        requestBody.writeTo(bufferedRequestBody)
      } else {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
        requestBody.writeTo(bufferedRequestBody)
        bufferedRequestBody.close()
      }
    } else {
      exchange.noRequestBody()
      if (!exchange.connection.isMultiplexed) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        exchange.noNewExchangesOnConnection()
      }
    }
  } else {
    exchange.noRequestBody()
  }

  if (requestBody == null || !requestBody.isDuplex()) {
    exchange.finishRequest()
  }
  if (responseBuilder == null) {
    responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
    if (invokeStartEvent) {
      exchange.responseHeadersStart()
      invokeStartEvent = false
    }
  }
  var response = responseBuilder
      .request(request)
      .handshake(exchange.connection.handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build()
  var code = response.code
  if (code == 100) {
    // Server sent a 100-continue even though we did not request one. Try again to read the actual
    // response status.
    responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
    if (invokeStartEvent) {
      exchange.responseHeadersStart()
    }
    response = responseBuilder
        .request(request)
        .handshake(exchange.connection.handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
    code = response.code
  }

  exchange.responseHeadersEnd(response)

  response = if (forWebSocket && code == 101) {
    // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
    response.newBuilder()
        .body(EMPTY_RESPONSE)
        .build()
  } else {
    response.newBuilder()
        .body(exchange.openResponseBody(response))
        .build()
  }
  if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
      "close".equals(response.header("Connection"), ignoreCase = true)) {
    exchange.noNewExchangesOnConnection()
  }
  if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
    throw ProtocolException(
        "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
  }
  return response
}
```

CallServerInterceptor 由以下步骤组成：

1. 向服务器发送 request header
2. 如果有 request body，就向服务器发送
3. 读取 response header，先构造一个 Response 对象
4. 如果有 response body，就在 3 的基础上加上 body 构造一个新的 Response 对象

可以看到，核心工作都由 <strong>HttpCodec</strong> 对象完成，而 HttpCodec 实际上利用的是 Okio，而 Okio 实际上还是用的 Socket，只不过是经过了层层嵌套。

# 总结

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/OkHttp/clipboard_20230323_035408.png)
