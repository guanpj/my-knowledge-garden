# Retrofit 使用及源码分析

# 使用

1. 导入依赖。

```groovy
implementation "com.squareup.okhttp3:okhttp:4.9.0"
implementation "com.squareup.okhttp3:logging-interceptor:4.9.0"

implementation "com.squareup.retrofit2:retrofit:2.9.0"
implementation "com.squareup.retrofit2:converter-gson:2.9.0"
implementation "com.squareup.retrofit2:adapter-rxjava3:2.9.0"

implementation "io.reactivex.rxjava3:rxjava:3.0.0"
implementation "io.reactivex.rxjava3:rxandroid:3.0.0"
```

2. 创建一个 interface 作为 WebService 的请求集合，在里面用注解写入需要配置的请求方法。

```kotlin
interface GitHubService {
  @GET("users/{user}/repos")
  fun listRepos(@Path("user") user: String?): Call<List<Repo>>

  @GET("users/{user}/repos")
  fun listReposRx(@Path("user") user: String?): Observable<List<Repo>>
  
  @GET("users/{user}/repos")
  fun listReposCompletable(@Path("user") user: String?): CompletableFuture<List<Repo>>

  @GET("users/{user}/repos")
  suspend fun listReposSuspend(@Path("user") user: String?): List<Repo>
  
  @GET("users/{user}/repos")
  fun listReposOptional(@Path("user") user: String?): Call<Optional<List<Repo>>>
}
```

3. 在正式代码里用 Retrofit 创建出 interface 的实例。

```kotlin
val user = "guanpj"

val okHttpClient = OkHttpClient.Builder()
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

val retrofit = Retrofit.Builder()
    .client(okHttpClient)
    .baseUrl("https://api.github.com/")
    .addCallAdapterFactory(RxJava3CallAdapterFactory.create())
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    
val service = retrofit.create(GitHubService::class.java)
```

4. 调用创建出的 Service 实例的对应方法，创建出相应的可以用来发起网络请求的 Call 对象 / Obsevable 对象 / suspend 挂起函数，并发起请求。

```kotlin
val listRepos = service.listRepos(user)
listRepos.enqueue(object : retrofit2.Callback<List<Repo>?> {
    override fun onFailure(call: retrofit2.Call<List<Repo>?>, t: Throwable) {

    }

    override fun onResponse(call: retrofit2.Call<List<Repo>?>, response: retrofit2.Response<List<Repo>?>) {
      Log.e("gpj", "Response: ${response.body()!![0].name}")
    }
})

val listReposRx = service.listReposRx(user)
listReposRx.subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(object : Observer<List<Repo>> {
        override fun onSubscribe(d: Disposable?) {
        }

        override fun onNext(t: List<Repo>?) {
          Log.e("gpj", "size:${t?.size}")
        }

        override fun onComplete() {
        }

        override fun onError(e: Throwable?) {
        }
    })

val future: CompletableFuture<List<Repo>> = service.listReposCompletable(user)
future.thenAccept { Log.e("gpj", "size:${it.size}") }

MainScope().launch {
    val list = service.listReposSuspend(user)
    Log.e("gpj", "size:${list.size}")
}

val call = service.listReposOptional(user)
thread {
    // 实际上这里是会抛出异常的，原因见后文分析
    val optionalList = call.execute().body()
    optionalList?.ifPresent { list -> list.map { Log.e("gpj", it.full_name) } }
}
```

# 解析

在正式开始前，先简单介绍一下几个关键词，供备忘：

- `CallAdapter<R, T>`：将一个 Call 从响应类型 R 适配成 T 类型的适配器。

  - `Type responseType()` 适配器将 HTTP 响应体转换为 Java 对象时，该对象的类型。比如 `Call<Repo>` 的返回值是 Repo
  - `T adapt(Call<R> call)`：返回一个代理了 call 的 T
- `CallAdapter.Factory`：用于创建 CallAdapter 实例的工厂

  - `CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit)`：返回一个可以返回 returnType 的接口方法的 CallAdapter，如果不能处理，则返回 null
- `Converter<F, T>`：将 F 转换为 T 类型的值的转换器。

  - `T convert(F value) throws IOException`
- `Converter.Factory`：基于一个类型和目标类型创建一个 Converter 实例的工厂

  - `Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit)`：返回一个可以转换 HTTP 响应体到 type 的转换器
  - `Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit)`：返回一个可以转换 type 到 HTTP 请求体的转换器
  - `Converter<?, String> stringConverter(Type type, Annotation[] annotations, Retrofit retrofit)`：返回一个可以转换 type 到 String 的转换器

## Retrofit.Builder

如果想知道 Retrofit 在创建时干了些什么，就不得不看 Retrofit.Builder ，它的构造方法如下：

```java
Builder(Platform platform) {
  this.platform = platform;
}

public Builder() {
  this(Platform.get());
}
```

这里首先是调用了 `Platform.get()` 获取了一个 Platform 实例，然后保存到变量 platform 中。

```java
class Platform {
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {
    return "Dalvik".equals(System.getProperty("java.vm.name"))
        ? new Android() //
        : new Platform(true);
  }

  private final boolean hasJava8Types;
  private final @Nullable Constructor<Lookup> lookupConstructor;

  Platform(boolean hasJava8Types) {
    this.hasJava8Types = hasJava8Types;

    Constructor<Lookup> lookupConstructor = null;
    if (hasJava8Types) {
      try {
        // Because the service interface might not be public, we need to use a MethodHandle lookup
        // that ignores the visibility of the declaringClass.
        lookupConstructor = Lookup.class.getDeclaredConstructor(Class.class, int.class);
        lookupConstructor.setAccessible(true);
      } catch (NoClassDefFoundError ignored) {
        // Android API 24 or 25 where Lookup doesn't exist. Calling default methods on non-public
        // interfaces will fail, but there's nothing we can do about it.
      } catch (NoSuchMethodException ignored) {
        // Assume JDK 14+ which contains a fix that allows a regular lookup to succeed.
        // See https://bugs.openjdk.java.net/browse/JDK-8209005.
      }
    }
    this.lookupConstructor = lookupConstructor;
  }

  @Nullable
  Executor defaultCallbackExecutor() {
    return null;
  }

  // 1、如果 API < 24，这里列表中只有一个 DefaultCallAdapterFactory
  // 否则多一个 CompletableFutureCallAdapterFactory 用于支持 Java8
  List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
      @Nullable Executor callbackExecutor) {
    DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
    return hasJava8Types
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
  }

  int defaultCallAdapterFactoriesSize() {
    return hasJava8Types ? 2 : 1;
  }

  // 2、如果 API < 24，这里列表为空；
  // 否则会有一个 OptionalConverterFactory 用于支持 Java8
  List<? extends Converter.Factory> defaultConverterFactories() {
    return hasJava8Types ? singletonList(OptionalConverterFactory.INSTANCE) : emptyList();
  }

  int defaultConverterFactoriesSize() {
    return hasJava8Types ? 1 : 0;
  }

  @IgnoreJRERequirement // Only called on API 24+.
  boolean isDefaultMethod(Method method) {
    return hasJava8Types && method.isDefault();
  }

  @IgnoreJRERequirement // Only called on API 26+.
  @Nullable
  Object invokeDefaultMethod(Method method, Class<?> declaringClass, Object object, Object... args)
      throws Throwable {
    Lookup lookup =
        lookupConstructor != null
            ? lookupConstructor.newInstance(declaringClass, -1 /* trusted */)
            : MethodHandles.lookup();
    return lookup.unreflectSpecial(method, declaringClass).bindTo(object).invokeWithArguments(args);
  }

  static final class Android extends Platform {
    Android() {
      //API 24 以上才支持 Java8
      super(Build.VERSION.SDK_INT >= 24);
    }

    @Override
    public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Nullable
    @Override
    Object invokeDefaultMethod(
        Method method, Class<?> declaringClass, Object object, Object... args) throws Throwable {
      if (Build.VERSION.SDK_INT < 26) {
        throw new UnsupportedOperationException(
            "Calling default methods on API 24 and 25 is not supported");
      }
      return super.invokeDefaultMethod(method, declaringClass, object, args);
    }

    static final class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override
      public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
}
```

findPlatform 方法会返回一个 Android 类型的 Platform 对象，它继承自 Platform 并重写了 defaultCallbackExecutor 返回一个 MainThreadExecutor。

同时注意 43 行注释一和 57 行注释二关于 defaultCallAdapterFactories 和 defaultConverterFactories 集合的赋值情况。

接下来我们在创建 Retrofit 对象的时候通过 Retrofit.Builder 设置了一些参数，这些参数都会保存到成员变量中，我们对照 Retrofit.Builder.build 方法进行解释：

```java
public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }

  // callFactory 就是传入的 OkHttpClient
  // 因为 OkHttpClient 实现了 okhttp3.Call.Factory这个接口
  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    callFactory = new OkHttpClient();
  }

  // 如果没有设置 callbackExecutor，则获取 platform 的默认 callbackExecutor
  // 这里被赋值为了 MainThreadExecutor
  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  // 在配置的时候设置了一个 RxJava3CallAdapterFactory
  // 然后这里另外添加了一个 DefaultCallAdapterFactory，并传入刚获取的 callbackExecutor 
  // 如果 Android API >= 24，默认还有一个 CompletableFutureCallAdapterFactory
  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
  callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

  // Make a defensive copy of the converters.
  List<Converter.Factory> converterFactories =
      new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  // 添加内置 ConverterFactory
  converterFactories.add(new BuiltInConverters());
  // 添加我们设置的 GsonConverterFactory
  converterFactories.addAll(this.converterFactories);
  // 如果 Android API >= 24，添加默认的 OptionalConverterFactory 支持 Java8
  converterFactories.addAll(platform.defaultConverterFactories());

  return new Retrofit(
      callFactory,
      baseUrl,
      unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories),
      callbackExecutor,
      validateEagerly);
}
```

这里首先是根据配置的内容和 Platform 获取到 callAdapterFactories 和 converterFactories 集合，然后利用这些参数创建一个 Retrofit 实例。

需要注意的是，第 37-39 行添加用户配置的 ConverterFactory 和 Platform 默认的 ConverterFactory 的顺序值得怀疑。后文分析过程将会进行解释。

## Retrofit.create

```java
public <T> T create(final Class<T> service) {
  validateServiceInterface(service);
  return (T)
      Proxy.newProxyInstance(
          service.getClassLoader(),
          new Class<?>[] {service},
          new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {
              // If the method is a method from Object then defer to normal invocation.
              if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
              }
              args = args != null ? args : emptyArgs;
              // 如果是 default 方法，则通过 platform 调用此方法
              // 普通方法则进行 loadServiceMethod(method).invoke(args)
              return platform.isDefaultMethod(method)
                  ? platform.invokeDefaultMethod(method, service, proxy, args)
                  : loadServiceMethod(method).invoke(args);
            }
          });
}

private void validateServiceInterface(Class<?> service) {
  // service 必需是接口
  if (!service.isInterface()) {
    throw new IllegalArgumentException("API declarations must be interfaces.");
  }

  Deque<Class<?>> check = new ArrayDeque<>(1);
  check.add(service);
  while (!check.isEmpty()) {
    Class<?> candidate = check.removeFirst();
    if (candidate.getTypeParameters().length != 0) {
      StringBuilder message =
          new StringBuilder("Type parameters are unsupported on ").append(candidate.getName());
      if (candidate != service) {
        message.append(" which is an interface of ").append(service.getName());
      }
      throw new IllegalArgumentException(message.toString());
    }
    Collections.addAll(check, candidate.getInterfaces());
  }

  // 如果 Retrofit.Builder 中设置了此参数
  if (validateEagerly) {
    Platform platform = Platform.get();
    // 提前解析并验证所有非 default 和非 static 方法是否配置正确，并缓存解析结果
    for (Method method : service.getDeclaredMethods()) {
      if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
        loadServiceMethod(method);
      }
    }
  }
}
```

可以看出，通过 Retrofit.create(Class) 方法的调用，会创建出 GitHubService 的实例，从而使得 GitHubService 中配置的方法变得可用，这是 Retrofit 代码结构的核心。

Retrofit.create() 方法的核心，使用的是 Proxy.newProxyInstance() 方法来创建 GitHubService 实例。这个方法会为参数中的多个 interface（具体到 Retrofit 来说，是固定传入一个 interface）创建一个对象，这个对象实现了所有 interface 的每个方法，并且每个方法的实现都是雷同的：调用对象实例内部的一个 InvocationHandler 成员变量的 invoke() 方法，并把自己的方法信息传递进去。这样就在实质上实现了代理逻辑：interface 中的方法全部由一个另外设定的 InvocationHandler 对象来进行代理操作。并且，这些方法的具体实现是在运行时生成 interface 实例时才确定的，而不是在编译时（虽然在编译时就已经可以通过代码逻辑推断出来）。这就是<strong>「动态代理机制」</strong>的具体含义。

因此，invoke() 方法中的逻辑，就是 Retrofit 创建 Service 实例的关键。

当调用 GitHubService 中配置好的请求方法时，`InvocationHandler.invoke()` 方法就会被回调。在 invoke 方法中，会进行判断，如果调用的是我们配置的请求方法，则会走到 `loadServiceMethod(method).invoke(args)`。

## loadServiceMethod(method)

```java
ServiceMethod<?> loadServiceMethod(Method method) {
  ServiceMethod<?> result = serviceMethodCache.get(method);
  if (result != null) return result;

  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      result = ServiceMethod.parseAnnotations(this, method);
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

可以看到，这里对 ServiceMethod 进行了缓存设计。具体解析方法还是在 ServiceMethod 中：

```java
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    // 解析方法注解并创建 RequestFactory
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(
          method,
          "Method return type must not include a type variable or wildcard: %s",
          returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }
    // 解析方法注解并创建 HttpServiceMethod 
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```

HttpServiceMethod 继承自 ServiceMethod，parseAnnotations 方法如下：

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
  boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
  boolean continuationWantsResponse = false;
  boolean continuationBodyNullable = false;

  Annotation[] annotations = method.getAnnotations();
  Type adapterType;
  // 如果是 Kotlin suspend 方法
  if (isKotlinSuspendFunction) {
    Type[] parameterTypes = method.getGenericParameterTypes();
    Type responseType =
        Utils.getParameterLowerBound(
            0, (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
    if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {
      // Unwrap the actual body type from Response<T>.
      responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
      continuationWantsResponse = true;
    } else {
      // TODO figure out if type is nullable or not
      // Metadata metadata = method.getDeclaringClass().getAnnotation(Metadata.class)
      // Find the entry for method
      // Determine if return type is nullable or not
    }

    adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
    annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
  } else {
    // 普通方法返回泛型参数
    adapterType = method.getGenericReturnType();
  }

  // 1、根据 method 的返回值类型以及方法注解返回第一个可以处理的 CallAdapter
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);
  Type responseType = callAdapter.responseType();
  if (responseType == okhttp3.Response.class) {
    throw methodError(
        method,
        "'"
            + getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  if (responseType == Response.class) {
    throw methodError(method, "Response must include generic type (e.g., Response<String>)");
  }
  // TODO support Unit for Kotlin?
  if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
    throw methodError(method, "HEAD method must use Void as response type.");
  }

  // 2、根据 responseType 以及方法注解返回第一个可以处理的 Converter
  Converter<ResponseBody, ResponseT> responseConverter =
      createResponseConverter(retrofit, method, responseType);

  // 3、根据不同的场景返回不同的 HttpServiceMethod 实现
  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    // 3.1、根据
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  } else if (continuationWantsResponse) {
    //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
    return (HttpServiceMethod<ResponseT, ReturnT>)
        new SuspendForResponse<>(
            requestFactory,
            callFactory,
            responseConverter,
            (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
  } else {
    //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
    return (HttpServiceMethod<ResponseT, ReturnT>)
        new SuspendForBody<>(
            requestFactory,
            callFactory,
            responseConverter,
            (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
            continuationBodyNullable);
  }
}
```

这行代码负责读取 interface 中原方法的信息（包括返回值类型、方法注解、参数类型、参数注解），并将这些信息做初步分析后根据这些信息创建出每个方法对应的 CallAdapter 和 Convetor 等对象，并将这些对象传入一个 CallAdapted 对象并返回此对象。

这里分别分析 32 行 `createCallAdapter` 和 51 行 `createResponseConverter` 的代码。

### createCallAdapter

```java
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
    Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
  try {
    //noinspection unchecked
    return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create call adapter for %s", returnType);
  }
}
```

还是交给 Retrofit 类处理：

```java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
  return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?, ?> nextCallAdapter(
    @Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
  Objects.requireNonNull(returnType, "returnType == null");
  Objects.requireNonNull(annotations, "annotations == null");

  // 遍历 callAdapterFactories 集合
  int start = callAdapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
    // 通过 CallAdapter.Factory.get 判断是否为符合要求的 CallAdapter
    CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) {
      return adapter;
    }
  }

  StringBuilder builder =
      new StringBuilder("Could not locate call adapter for ").append(returnType).append(".\n");
  if (skipPast != null) {
    builder.append("  Skipped:");
    for (int i = 0; i < start; i++) {
      builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
    }
    builder.append('\n');
  }
  builder.append("  Tried:");
  for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
    builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
  }
  throw new IllegalArgumentException(builder.toString());
}
```

逻辑比较简单，就是遍历 callAdapterFactories 集合，调用每个 CallAdapter.Factory.get() 方法并传入请求方法的注解信息和返回值信息，根据这些信息每个 CallAdapter.Factory 去判断此请求方法适不适合自己。

#### DefaultCallAdapterFactory.get()

```java
@Override
public @Nullable CallAdapter<?, ?> get(
    Type returnType, Annotation[] annotations, Retrofit retrofit) {
  // 方法返回值为 Call 类型
  if (getRawType(returnType) != Call.class) {
    return null;
  }
  
  // 方法返回值必须包含泛型 Call<Foo>
  if (!(returnType instanceof ParameterizedType)) {
    throw new IllegalArgumentException(
        "Call return type must be parameterized as Call<Foo> or Call<? extends Foo>");
  }
  final Type responseType = Utils.getParameterUpperBound(0, (ParameterizedType) returnType);

  // executor = callbackExecutor = MainThreadExecutor
  final Executor executor =
      Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
          ? null
          : callbackExecutor;

  // 返回匿名对象实现 CallAdapter
  return new CallAdapter<Object, Call<?>>() {
    @Override
    public Type responseType() {
      return responseType;
    }

    @Override
    public Call<Object> adapt(Call<Object> call) {
      return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
    }
  };
}
```

1. DefaultCallAdapterFactory 首先通过判断方法返回值是否为 Call 类型并且是否带有泛型参数来确定能不能处理这个请求；
2. 然后将 executor 赋值为前面 Retrofit.create 流程中获取到的 MainThreadExecutor；
3. 最后返回一个 CallAdapter 的匿名实现类。

#### RxJava3CallAdapterFactory.get()

```java
@Override
public @Nullable CallAdapter<?, ?> get(
  Type returnType, Annotation[] annotations, Retrofit retrofit) {
    Class<?> rawType = getRawType(returnType);

    if (rawType == Completable.class) {
      // Completable is not parameterized (which is what the rest of this method deals with) so it
      // can only be created with a single configuration.
      return new RxJava3CallAdapter(
          Void.class, scheduler, isAsync, false, true, false, false, false, true);
    }

    boolean isFlowable = rawType == Flowable.class;
    boolean isSingle = rawType == Single.class;
    boolean isMaybe = rawType == Maybe.class;
    if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
      return null;
    }

    boolean isResult = false;
    boolean isBody = false;
    Type responseType;
    if (!(returnType instanceof ParameterizedType)) {
      String name =
          isFlowable ? "Flowable" : isSingle ? "Single" : isMaybe ? "Maybe" : "Observable";
      throw new IllegalStateException(
          name
              + " return type must be parameterized"
              + " as "
              + name
              + "<Foo> or "
              + name
              + "<? extends Foo>");
    }

    Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
    Class<?> rawObservableType = getRawType(observableType);
    if (rawObservableType == Response.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException(
            "Response must be parameterized" + " as Response<Foo> or Response<? extends Foo>");
      }
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    } else if (rawObservableType == Result.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException(
            "Result must be parameterized" + " as Result<Foo> or Result<? extends Foo>");
      }
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      isResult = true;
    } else {
      responseType = observableType;
      isBody = true;
    }

    return new RxJava3CallAdapter(
        responseType, scheduler, isAsync, isResult, isBody, isFlowable, isSingle, isMaybe, false);
  }
}
```

#### CompletableFutureCallAdapterFactory.get()

```java
@Override
public @Nullable CallAdapter<?, ?> get(
    Type returnType, Annotation[] annotations, Retrofit retrofit) {
  if (getRawType(returnType) != CompletableFuture.class) {
    return null;
  }
  if (!(returnType instanceof ParameterizedType)) {
    throw new IllegalStateException(
        "CompletableFuture return type must be parameterized"
            + " as CompletableFuture<Foo> or CompletableFuture<? extends Foo>");
  }
  Type innerType = getParameterUpperBound(0, (ParameterizedType) returnType);

  if (getRawType(innerType) != Response.class) {
    // Generic type is not Response<T>. Use it for body-only adapter.
    return new BodyCallAdapter<>(innerType);
  }

  // Generic type is Response<T>. Extract T and create the Response version of the adapter.
  if (!(innerType instanceof ParameterizedType)) {
    throw new IllegalStateException(
        "Response must be parameterized" + " as Response<Foo> or Response<? extends Foo>");
  }
  Type responseType = getParameterUpperBound(0, (ParameterizedType) innerType);
  return new ResponseCallAdapter<>(responseType);
}
```

#### 小结

对于例子中 GithubService 中的请求方法：

1. `fun listRepos(@Path("user") user: String?): Call<List<Repo>>`

返回匿名的 CallAdapter 实现类

2. `fun listReposRx(@Path("user") user: String?): Observable<List<Repo>>`

返回 RxJava3CallAdapter 实例

3. `fun listReposCompletable(@Path("user") user: String?): CompletableFuture<List<Repo>>`

返回 CompletableFutureCallAdapterFactory.ResponseCallAdapter 实例

Retrofit.callAdapter() 方法的逻辑如下：

- 首先会遍历 callAdapterFactories 集合，并调用每个 CallAdapter.Factory.get() 方法并传入请求方法的注解信息和返回值信息，根据这些信息每个 CallAdapterFactory 去判断此请求方法适不适合自己。
- 一旦找到合适的 CallAdapter 则会立即打破循环并返回。
- DefaultCallAdapterFactory 只能处理返回值为 `Call<Foo>` 或者 `Call<? extends Foo>` 的请求方法，并且为它们返回一个匿名的 CallAdapter 实现。
- 其它情况，如果要支持 Rxjava，则需要额外增加 CallAdapterFactory 支持。
- 如果 Android API >= 24，如果请求方法返回的是 CompletableFuture，Retrofit 默认会进行支持并返回 CompletableFutureCallAdapterFactory 进行处理。
- 如果需要支持其它的返回值类型，则需要手动通过 addCallAdapterFactory() 方法增加对 CallAdapterFactory 的支持。
- 如果请求方法返回值不受支持，则会抛出异常。

### createResponseConverter

```java
private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
    Retrofit retrofit, Method method, Type responseType) {
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create converter for %s", responseType);
  }
}
```

还是从 Retrofit 类中获取：

```java
public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
  return nextResponseBodyConverter(null, type, annotations);
}

public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
    @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
  Objects.requireNonNull(type, "type == null");
  Objects.requireNonNull(annotations, "annotations == null");

  int start = converterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = converterFactories.size(); i < count; i++) {
    Converter<ResponseBody, ?> converter =
        converterFactories.get(i).responseBodyConverter(type, annotations, this);
    if (converter != null) {
      //noinspection unchecked
      return (Converter<ResponseBody, T>) converter;
    }
  }

  StringBuilder builder =
      new StringBuilder("Could not locate ResponseBody converter for ")
          .append(type)
          .append(".\n");
  if (skipPast != null) {
    builder.append("  Skipped:");
    for (int i = 0; i < start; i++) {
      builder.append("\n   * ").append(converterFactories.get(i).getClass().getName());
    }
    builder.append('\n');
  }
  builder.append("  Tried:");
  for (int i = start, count = converterFactories.size(); i < count; i++) {
    builder.append("\n   * ").append(converterFactories.get(i).getClass().getName());
  }
  throw new IllegalArgumentException(builder.toString());
}
```

#### BuiltInConverters.responseBodyConverter()

```java
@Override
public @Nullable Converter<ResponseBody, ?> responseBodyConverter(
    Type type, Annotation[] annotations, Retrofit retrofit) {
  // 返回值类型为 ResponseBody
  if (type == ResponseBody.class) {
    return Utils.isAnnotationPresent(annotations, Streaming.class)
        ? StreamingResponseBodyConverter.INSTANCE
        : BufferingResponseBodyConverter.INSTANCE;
  }
  // 返回值为 void
  if (type == Void.class) {
    return VoidResponseBodyConverter.INSTANCE;
  }
  // 返回值为 Kotlin 的 Unit
  if (checkForKotlinUnit) {
    try {
      if (type == Unit.class) {
        return UnitResponseBodyConverter.INSTANCE;
      }
    } catch (NoClassDefFoundError ignored) {
      checkForKotlinUnit = false;
    }
  }
  return null;
}
```

据前面分析可知，BuiltInConverters 是 Retrofit 在 Android Platform 上唯一默认的 ConvertFactory。因此，请求方法的返回值的数据类型（注意不是返回值类型）只有在以上三种情况下，通过 Retrofit.Builder() 创建 Retrofit 的时候才可以不指定 ConvertFactory，否则调用该方法的时候将会抛出异常。

#### GsonConverterFactory. responseBodyConverter()

```java
@Override
public Converter<ResponseBody, ?> responseBodyConverter(
    Type type, Annotation[] annotations, Retrofit retrofit) {
  TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
  return new GsonResponseBodyConverter<>(gson, adapter);
}
```

如果通过 addConverterFactory() 添加了 GsonConverterFactory ，这里将返回一个 GsonResponseBodyConverter 实例，不用进行任何判断，当然 gson.getAdapter 中不能抛出异常。

#### OptionalConverterFactory.responseBodyConverter()

```java
@Override
public @Nullable Converter<ResponseBody, ?> responseBodyConverter(
    Type type, Annotation[] annotations, Retrofit retrofit) {
  if (getRawType(type) != Optional.class) {
    return null;
  }

  // 获取 Option 内层的泛型类型
  Type innerType = getParameterUpperBound(0, (ParameterizedType) type);
  
  Converter<ResponseBody, Object> delegate =
      retrofit.responseBodyConverter(innerType, annotations);
  return new OptionalConverter<>(delegate);
}
```

它只处理返回值为 `Call<Option<Foo>>` 类型的请求，并且脱去外衣获取 Option 内层的泛型类型 innerType，并且使用这个 innerType 查找到另外一个符合要求 Converter，最后的 Convert 操作都使用这个 delegate 进行转换操作。

#### 小结

对于 BuiltInConverters，它只会查看泛型参数内容，如果泛型参数为 ResponseBody、Void 或者 Kotlin 的 Unit，BuiltInConverters 将会返回相应的 Converter；

对于 GsonConverterFactory，它将来者不拒，返回 GsonResponseBodyConverter。

对于 OptionalConverterFactory，它只处理返回值为 `Call<Option<Foo>>` 类型的请求，并且委托另外一个符合要求 Converter 进行转换操作。

Retrofit.responseBodyConverter() 方法一旦找到合适的 CallAdapterFactory 则会立即打破循环并返回。

因此这里需要特别注意，前面提到过的 Retrofit.Builder.build() 方法中，converterFactories 的添加顺序如下：

```java
// 添加内置 ConverterFactory
converterFactories.add(new BuiltInConverters());
// 添加我们设置的 GsonConverterFactory
converterFactories.addAll(this.converterFactories);
// 如果 Android API >= 24，添加默认的 OptionalConverterFactory 支持 Java8
converterFactories.addAll(platform.defaultConverterFactories());
```

这时候如果要像这样使用 Optional 去接收数据：

```kotlin
@GET("users/{user}/repos")
fun listReposOptional(@Path("user") user: String?): Call<Optional<List<Repo>>>
```

就会抛出异常：

```plain text
com.google.gson.JsonSyntaxException: java.lang.IllegalStateException: Expected BEGIN_OBJECT but was BEGIN_ARRAY at line 1 column 2 path $
        at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:226)
        at retrofit2.converter.gson.GsonResponseBodyConverter.convert(GsonResponseBodyConverter.java:40)
        at retrofit2.converter.gson.GsonResponseBodyConverter.convert(GsonResponseBodyConverter.java:27)
        at retrofit2.OkHttpCall.parseResponse(OkHttpCall.java:243)
        at retrofit2.OkHttpCall$1.onResponse(OkHttpCall.java:153)
```

很明显这是来自 gson 的错误，根据前面的分析，如果在 Retrofit.Builder 过程添加了 GsonConverterFactory，responseBodyConverter() 方法中会直接返回一个 GsonResponseBodyConverter 对象，在解释返回结果的时候会拿 `Optional<List<Repo>>` 去接收 JSON 数组，导致解析出错。

如果要使这个流程走得通，就需要将第 4 行和第 6 行代码进行交换。（本人亲自验证可行）

可能这是 Retrofit 的一个 bug，如果有其他不同的理解，欢迎讨论。

### return

从以上分析可知，parseAnnotations() 方法最终会返回 HttpServiceMethod 实例，而 HttpServiceMethod 类又三个子类，parseAnnotations() 方法的最后会根据不同的条件来返回不同的子类实例。

如果我们的请求方法不是 Kotlin 挂起函数，返回的是 CallAdapted 对象。

#### CallAdapted

此时 loadServiceMethod() 返回的对象为 CallAdapted：

```java
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
  private final CallAdapter<ResponseT, ReturnT> callAdapter;

  CallAdapted(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, ReturnT> callAdapter) {
    super(requestFactory, callFactory, responseConverter);
    this.callAdapter = callAdapter;
  }

  @Override
  protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
    return callAdapter.adapt(call);
  }
}
```

#### SuspendForResponse

```java
static final class SuspendForResponse<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
  private final CallAdapter<ResponseT, Call<ResponseT>> callAdapter;

  SuspendForResponse(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, Call<ResponseT>> callAdapter) {
    super(requestFactory, callFactory, responseConverter);
    this.callAdapter = callAdapter;
  }

  @Override
  protected Object adapt(Call<ResponseT> call, Object[] args) {
    call = callAdapter.adapt(call);

    //noinspection unchecked Checked by reflection inside RequestFactory.
    Continuation<Response<ResponseT>> continuation =
        (Continuation<Response<ResponseT>>) args[args.length - 1];

    // See SuspendForBody for explanation about this try/catch.
    try {
      return KotlinExtensions.awaitResponse(call, continuation);
    } catch (Exception e) {
      return KotlinExtensions.suspendAndThrow(e, continuation);
    }
  }
}
```

#### SuspendForBody

```java
static final class SuspendForBody<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
  private final CallAdapter<ResponseT, Call<ResponseT>> callAdapter;
  private final boolean isNullable;

  SuspendForBody(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, Call<ResponseT>> callAdapter,
      boolean isNullable) {
    super(requestFactory, callFactory, responseConverter);
    this.callAdapter = callAdapter;
    this.isNullable = isNullable;
  }

  @Override
  protected Object adapt(Call<ResponseT> call, Object[] args) {
    call = callAdapter.adapt(call);

    //noinspection unchecked Checked by reflection inside RequestFactory.
    Continuation<ResponseT> continuation = (Continuation<ResponseT>) args[args.length - 1];

    // Calls to OkHttp Call.enqueue() like those inside await and awaitNullable can sometimes
    // invoke the supplied callback with an exception before the invoking stack frame can return.
    // Coroutines will intercept the subsequent invocation of the Continuation and throw the
    // exception synchronously. A Java Proxy cannot throw checked exceptions without them being
    // declared on the interface method. To avoid the synchronous checked exception being wrapped
    // in an UndeclaredThrowableException, it is intercepted and supplied to a helper which will
    // force suspension to occur so that it can be instead delivered to the continuation to
    // bypass this restriction.
    try {
      return isNullable
          ? KotlinExtensions.awaitNullable(call, continuation)
          : KotlinExtensions.await(call, continuation);
    } catch (Exception e) {
      return KotlinExtensions.suspendAndThrow(e, continuation);
    }
  }
}
```

## HttpServiceMethod.invoke(Object[] args)

```java
@Override
final @Nullable ReturnT invoke(Object[] args) {
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
  return adapt(call, args);
}

protected abstract @Nullable ReturnT adapt(Call<ResponseT> call, Object[] args);
```

### new OkHttpCall<>()

OkHttpCall 的创建：`new OkHttpCall<>(requestFactory, args, callFactory, responseConverter)`

OkHttpCall 是 retrofit2.Call 的子类。这行代码负责将 ServiceMethod 解读到的信息（主要是一个 RequestFactory 、一个 CallFactory 和一个 ResponseConverter）封装进 OkHttpCall；而这个对象可以在需要的时候（例如它的 enqueue() 方法被调用的时候）， 利用 RequestFactory 和 CallFactory 来创建一个 okhttp3.Call 对象，并调用这个 okhttp3.Call 对象来进行网络请求的发起，然后利用 ResponseConverter 对结果进行预处理之后，交回给 Retrofit 的 Callback 。

### HttpServiceMethod.adapt()

调用 adapt() 方法：`adapt(call, args)`

创建 OkHttpCall 对象后，会调用 abstract 方法 `adapt(Call<ResponseT> call, Object[] args)` 将 OkHttpCall 对象和请求参数传入。

根据前面的分析，HttpServiceMethod 有三个子类并且都是 HttpServiceMethod 的内部类，分别是

CallAdapted、SuspendForResponse 和 SuspendForBody。

#### CallAdapted.adapt

```java
@Override
protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
  return callAdapter.adapt(call);
}
```

可以看到，这里是直接调用了 callAdapter 的 adapt 方法，这个 callAdapter 就是前面 loadServiceMethod() 流程中通过 createCallAdapter() 方法获取到的 CallAdapter 对象。

因此，根据前面的分析可知，此时的 CallAdapter 会根据请求方法的类型来使相应的实现类。

##### DefaultCallAdapterFactory 中的匿名 CallAdapter

```java
new CallAdapter<Object, Call<?>>() {
  @Override
  public Type responseType() {
    return responseType;
  }

  @Override
  public Call<Object> adapt(Call<Object> call) {
    return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
  }
}
```

可以看到，adapt 方法返回的是 ExecutorCallbackCall 对象。

##### RxJava3CallAdapterFactory.RxJava3CallAdapter

```java
@Override
public Object adapt(Call<R> call) {
  Observable<Response<R>> responseObservable =
      isAsync ? new CallEnqueueObservable<>(call) : new CallExecuteObservable<>(call);

  Observable<?> observable;
  if (isResult) {
    observable = new ResultObservable<>(responseObservable);
  } else if (isBody) {
    observable = new BodyObservable<>(responseObservable);
  } else {
    observable = responseObservable;
  }

  if (scheduler != null) {
    observable = observable.subscribeOn(scheduler);
  }

  if (isFlowable) {
    // We only ever deliver a single value, and the RS spec states that you MUST request at least
    // one element which means we never need to honor backpressure.
    return observable.toFlowable(BackpressureStrategy.MISSING);
  }
  if (isSingle) {
    return observable.singleOrError();
  }
  if (isMaybe) {
    return observable.singleElement();
  }
  if (isCompletable) {
    return observable.ignoreElements();
  }
  return RxJavaPlugins.onAssembly(observable);
}
```

##### CompletableFutureCallAdapterFactory.ResponseCallAdapter

```java
@Override
public CompletableFuture<R> adapt(final Call<R> call) {
  CompletableFuture<R> future = new CallCancelCompletableFuture<>(call);
  call.enqueue(new BodyCallback(future));
  return future;
}
```

##### 小结

对于例子中 GithubService 中的请求方法：

1. `fun listRepos(@Path("user") user: String?): Call<List<Repo>>`

返回 DefaultCallAdapterFactory.ExecutorCallbackCall 实例

2. `fun listReposRx(@Path("user") user: String?): Observable<List<Repo>>`

异步方式返回 CallEnqueueObservable 实例，同步方式返回 CallExecuteObservable 实例

3. `fun listReposCompletable(@Path("user") user: String?): CompletableFuture<List<Repo>>`

返回 CompletableFutureCallAdapterFactory.CallCancelCompletableFuture 实例

因此，CallAdatper 的作用就是把 OkHttpCall 根据请求方法的返回值包装成一个新的对象，并且返回给调用者。

#### SuspendForResponse.adapt

#### SuspendForBody.adapt

## Call.enqueue()

根据前面的分析，在 Android 平台上，对于返回 Call 类型的请求方法被调用时，会被一个匿名 CallAdapter 在 adapt 方法中包装成 ExecutorCallbackCall 并返回给调用者。不用多说也应该知道，这个类继承自 Call 接口。那么它是怎么实现请求的呢？

```java
static final class ExecutorCallbackCall<T> implements Call<T> {
  final Executor callbackExecutor;
  final Call<T> delegate;

  ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
    this.callbackExecutor = callbackExecutor;
    this.delegate = delegate;
  }

  @Override
  public void enqueue(final Callback<T> callback) {
    Objects.requireNonNull(callback, "callback == null");

    delegate.enqueue(
        new Callback<T>() {
          @Override
          public void onResponse(Call<T> call, final Response<T> response) {
            callbackExecutor.execute(
                () -> {
                  if (delegate.isCanceled()) {
                    // Emulate OkHttp's behavior of throwing/delivering an IOException on
                    // cancellation.
                    callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                  } else {
                    callback.onResponse(ExecutorCallbackCall.this, response);
                  }
                });
          }

          @Override
          public void onFailure(Call<T> call, final Throwable t) {
            callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
          }
        });
  }

  @Override
  public boolean isExecuted() {
    return delegate.isExecuted();
  }

  @Override
  public Response<T> execute() throws IOException {
    return delegate.execute();
  }

  @Override
  public void cancel() {
    delegate.cancel();
  }

  @Override
  public boolean isCanceled() {
    return delegate.isCanceled();
  }

  @SuppressWarnings("CloneDoesntCallSuperClone") // Performing deep clone.
  @Override
  public Call<T> clone() {
    return new ExecutorCallbackCall<>(callbackExecutor, delegate.clone());
  }

  @Override
  public Request request() {
    return delegate.request();
  }

  @Override
  public Timeout timeout() {
    return delegate.timeout();
  }
}
```

可以看到，这是一个典型的代理模式的实现，ExecutorCallbackCall 的全部实现都交给了 delegate 对象，而这个 delegate 正是 HttpServiceMethod.invoke() 流程中生成的 OkHttpCall 对象。

在 enqueue() 方法中，首先会调用 OkHttpCall.enqueue() 方法获取请求结果，然后在 CallBack 回调中在 callbackExecutor.execute() 中将回调结果再次回调给 callback。

这里 callbackExecutor 就是 Platform.Android.MainThreadExecutor，因此外层获取的回调实在主线程中的，不用再切换线程。

那么再看 OkHttpCall.enqueue() 是如何工作的：

```java
@Override
public void enqueue(final Callback<T> callback) {
  Objects.requireNonNull(callback, "callback == null");

  okhttp3.Call call;
  Throwable failure;

  synchronized (this) {
    if (executed) throw new IllegalStateException("Already executed.");
    executed = true;

    call = rawCall;
    failure = creationFailure;
    if (call == null && failure == null) {
      try {
        call = rawCall = createRawCall();
      } catch (Throwable t) {
        throwIfFatal(t);
        failure = creationFailure = t;
      }
    }
  }

  if (failure != null) {
    callback.onFailure(this, failure);
    return;
  }

  if (canceled) {
    call.cancel();
  }

  call.enqueue(
      new okhttp3.Callback() {
        @Override
        public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
          Response<T> response;
          try {
            // 解析结果
            response = parseResponse(rawResponse);
          } catch (Throwable e) {
            throwIfFatal(e);
            callFailure(e);
            return;
          }

          try {
            callback.onResponse(OkHttpCall.this, response);
          } catch (Throwable t) {
            throwIfFatal(t);
            t.printStackTrace(); // TODO this is not great
          }
        }

        @Override
        public void onFailure(okhttp3.Call call, IOException e) {
          callFailure(e);
        }

        private void callFailure(Throwable e) {
          try {
            callback.onFailure(OkHttpCall.this, e);
          } catch (Throwable t) {
            throwIfFatal(t);
            t.printStackTrace(); // TODO this is not great
          }
        }
      });
}

Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  // Remove the body's source (the only stateful object) so we can pass the response along.
  rawResponse =
      rawResponse
          .newBuilder()
          .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
          .build();

  int code = rawResponse.code();
  if (code < 200 || code >= 300) {
    try {
      // Buffer the entire body to avoid future I/O.
      ResponseBody bufferedBody = Utils.buffer(rawBody);
      return Response.error(bufferedBody, rawResponse);
    } finally {
      rawBody.close();
    }
  }

  if (code == 204 || code == 205) {
    rawBody.close();
    return Response.success(null, rawResponse);
  }

  ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
  try {
    // 使用 responseConverter 完成转换
    T body = responseConverter.convert(catchingBody);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // If the underlying source threw an exception, propagate that rather than indicating it was
    // a runtime exception.
    catchingBody.throwIfCaught();
    throw e;
  }
}
```

可以看到，这里是依赖 OkHttp3 来实现网络请求的。获取到请求结果后，第 40 行将会调用 parseResponse() 方法将结果解析出来。parseResponse() 方法第 100 行将会调用 responseConverter 进行转换。

这一整个过程可以一下时序图直观展示：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Retrofit/clipboard_20230323_035658.png)

## Observable.subscribe()

对于返回 Observable 的请求方法，我们知道经过 RxJava3CallAdapter.adapt 方法的包装，最终返回的是 CallEnqueueObservable 或者 CallExecuteObservable。以前者为例，来看看具体的实现方式。

```java
final class CallEnqueueObservable<T> extends Observable<Response<T>> {
  private final Call<T> originalCall;

  CallEnqueueObservable(Call<T> originalCall) {
    this.originalCall = originalCall;
  }

  @Override
  protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallCallback<T> callback = new CallCallback<>(call, observer);
    observer.onSubscribe(callback);
    if (!callback.isDisposed()) {
      call.enqueue(callback);
    }
  }

  private static final class CallCallback<T> implements Disposable, Callback<T> {
    private final Call<?> call;
    private final Observer<? super Response<T>> observer;
    private volatile boolean disposed;
    boolean terminated = false;

    CallCallback(Call<?> call, Observer<? super Response<T>> observer) {
      this.call = call;
      this.observer = observer;
    }

    @Override
    public void onResponse(Call<T> call, Response<T> response) {
      if (disposed) return;

      try {
        observer.onNext(response);

        if (!disposed) {
          terminated = true;
          observer.onComplete();
        }
      } catch (Throwable t) {
        Exceptions.throwIfFatal(t);
        if (terminated) {
          RxJavaPlugins.onError(t);
        } else if (!disposed) {
          try {
            observer.onError(t);
          } catch (Throwable inner) {
            Exceptions.throwIfFatal(inner);
            RxJavaPlugins.onError(new CompositeException(t, inner));
          }
        }
      }
    }

    @Override
    public void onFailure(Call<T> call, Throwable t) {
      if (call.isCanceled()) return;

      try {
        observer.onError(t);
      } catch (Throwable inner) {
        Exceptions.throwIfFatal(inner);
        RxJavaPlugins.onError(new CompositeException(t, inner));
      }
    }

    @Override
    public void dispose() {
      disposed = true;
      call.cancel();
    }

    @Override
    public boolean isDisposed() {
      return disposed;
    }
  }
}
```

在 subscribeActual() 方法中，同样调用的是 OkHttpCall.enqueue() 方法实现请求，并且在 CallCallback 回调中把请求结果通过 onNext(response) 发射给下游的 Observer。

这个过程的时序图如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Retrofit/clipboard_20230323_035702.png)

# 总结

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Retrofit/clipboard_20230323_035708.png)
