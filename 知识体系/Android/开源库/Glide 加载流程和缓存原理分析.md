# Glide 加载流程和缓存原理分析

# <strong>加载流程</strong>

Glide 最普通的用法如下：

Glide.with(this).load(url).into(textView);

以首次加载 url 指向的资源到 textView 对象为例，由于代码实在太过冗长，下面用流程图的方式表示各个环节的执行顺序。

## with

with 流程的主要职责：

- 创建 RequestManager 对象
- 初始化各式各样的配置信息（缓存、请求线程池、图片大小和格式等等）以及 Glide 单例对象。
- 将 Glide 请求和 application/Activity/SupportFragment/Fragment 的生命周期绑定在一起<strong>，从而实现自动执行请求，暂停操作</strong>。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034132.png)

```java
public class Glide implements ComponentCallbacks2 {
    ...
    @NonNull
    public static RequestManager with(@NonNull Context context) {
        return getRetriever(context).get(context);
    }

    @NonNull
    public static RequestManager with(@NonNull Activity activity) {
        return getRetriever(activity).get(activity);
    }

    @NonNull
    public static RequestManager with(@NonNull FragmentActivity activity) {
        return getRetriever(activity).get(activity);
    }

    @NonNull
    public static RequestManager with(@NonNull Fragment fragment) {
        return getRetriever(fragment.getActivity()).get(fragment);
    }

    @SuppressWarnings("deprecation")
    @Deprecated
    @NonNull
    public static RequestManager with(@NonNull android.app.Fragment fragment) {
        return getRetriever(fragment.getActivity()).get(fragment);
    }

    @NonNull
    public static RequestManager with(@NonNull View view) {
        return getRetriever(view.getContext()).get(view);
    }
}
```

每个重载方法内部都首先调用 getRetriever(@Nullable Context context) 方法获取一个 RequestManagerRetriever 对象，然后调用其 get 方法来返回 RequestManager。

传入 getRetriever 的参数都是 Context，而 RequestManagerRetriever.get 方法传入的参数各不相同，所以生命周期的绑定肯定发生在 get 方法中。把 Glide.with 方法里面的代码分成两部分来分析。

### getRetriever(Context)

getRetriever(Context) 方法会根据 @GlideModule 注解的类以及 AndroidManifest.xml 文件中 meta-data 配置的 GlideModule 来创建一个 Glide 实例，然后返回该实例的 RequestManagerRetriever。

首先从 getRetriever(Context)开始：

```java
@NonNull
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
  // Context could be null for other reasons (ie the user passes in null), but in practice it will
  // only occur due to errors with the Fragment lifecycle.
  Preconditions.checkNotNull(
      context,
      "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
          + "returns null (which usually occurs when getActivity() is called before the Fragment "
          + "is attached or after the Fragment is destroyed).");
  return Glide.get(context).getRequestManagerRetriever();
}


public static Glide get(@NonNull Context context) {
  if (glide == null) {
    // 如果有配置 @GlideModule 注解的 Module
    // 之类会反射构造 kapt 生成的 GeneratedAppGlideModuleImpl 类
    GeneratedAppGlideModule annotationGeneratedModule =
        getAnnotationGeneratedGlideModules(context.getApplicationContext());
    synchronized (Glide.class) {
      if (glide == null) {
        checkAndInitializeGlide(context, annotationGeneratedModule);
      }
    }
  }

  return glide;
}

private static void checkAndInitializeGlide(@NonNull Context context) {
  // In the thread running initGlide(), one or more classes may call Glide.get(context).
  // Without this check, those calls could trigger infinite recursion.
  if (isInitializing) {
    throw new IllegalStateException("You cannot call Glide.get() in registerComponents(),"
        + " use the provided Glide instance instead");
  }
  isInitializing = true;
  initializeGlide(context);
  isInitializing = false;
}

private static void initializeGlide(@NonNull Context context) {
  initializeGlide(context, new GlideBuilder());
}

private static void initializeGlide(
    @NonNull Context context,
    @NonNull GlideBuilder builder,
    @Nullable GeneratedAppGlideModule annotationGeneratedModule) {
  Context applicationContext = context.getApplicationContext();
  // 如果 GeneratedAppGlideModuleImpl 存在，且允许解析 manifest 文件
  // 则遍历 manifest 中的 meta-data，解析出所有的 GlideModule 类
  List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
  if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
    manifestModules = new ManifestParser(applicationContext).parse();
  }

  // 根据 Impl 的排除名单，剔除 manifest 中的 GlideModule 类
  if (annotationGeneratedModule != null
      && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
    Set<Class<?>> excludedModuleClasses = annotationGeneratedModule.getExcludedModuleClasses();
    Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
    while (iterator.hasNext()) {
      com.bumptech.glide.module.GlideModule current = iterator.next();
      if (!excludedModuleClasses.contains(current.getClass())) {
        continue;
      }
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
      }
      iterator.remove();
    }
  }

  if (Log.isLoggable(TAG, Log.DEBUG)) {
    for (com.bumptech.glide.module.GlideModule glideModule : manifestModules) {
      Log.d(TAG, "Discovered GlideModule from manifest: " + glideModule.getClass());
    }
  }

  // 如果 Impl 存在，那么设置为该类的 RequestManagerFactory；否则设置为 null
  RequestManagerRetriever.RequestManagerFactory factory =
      annotationGeneratedModule != null
          ? annotationGeneratedModule.getRequestManagerFactory()
          : null;
  builder.setRequestManagerFactory(factory);
  // 依次调用 manifest 中 GlideModule 类的 applyOptions 方法，将配置写到 builder 里
  for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    module.applyOptions(applicationContext, builder);
  }
  // 写入 Impl 的配置
  // 也就是说 Impl 配置的优先级更高，如果有冲突的话
  if (annotationGeneratedModule != null) {
    annotationGeneratedModule.applyOptions(applicationContext, builder);
  }
  // 调用GlideBuilder.build方法创建 Glide
  Glide glide = builder.build(applicationContext);
  // 依次调用 manifest 中 GlideModule 类的 registerComponents 方法
  // 来替换 Glide 的默认配置
  for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    try {
      module.registerComponents(applicationContext, glide, glide.registry);
    } catch (AbstractMethodError e) {
      throw new IllegalStateException(
          "Attempting to register a Glide v3 module. If you see this, you or one of your"
              + " dependencies may be including Glide v3 even though you're using Glide v4."
              + " You'll need to find and remove (or update) the offending dependency."
              + " The v3 module name is: "
              + module.getClass().getName(),
          e);
    }
  }
  // 调用 Impl 中替换 Glide 配置的方法
  if (annotationGeneratedModule != null) {
    annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
  }
  // 注册内存管理的回调，因为 Glide 实现了 ComponentCallbacks2 接口
  applicationContext.registerComponentCallbacks(glide);
  // 保存 glide 实例到静态变量中
  Glide.glide = glide;
}
```

如果没有在 AndroidManifest 和 @GlideModule 注解中进行过配置，上面的代码可以简化为：

```java
@SuppressWarnings("deprecation")
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
  Context applicationContext = context.getApplicationContext();
  // 调用 GlideBuilder.build 方法创建 Glide
  Glide glide = builder.build(applicationContext);
  // 注册内存管理的回调，因为 Glide 实现了 ComponentCallbacks2 接口
  applicationContext.registerComponentCallbacks(glide);
  // 保存 glide 实例到静态变量中
  Glide.glide = glide;
}
```

看一下 GlideBuilder.build 方法：

```java
@NonNull
Glide build(@NonNull Context context) {
  ...

  GlideExperiments experiments = glideExperimentsBuilder.build();
  RequestManagerRetriever requestManagerRetriever =
      new RequestManagerRetriever(requestManagerFactory, experiments);

  return new Glide(
      context,
      engine,
      memoryCache,
      bitmapPool,
      arrayPool,
      requestManagerRetriever,
      connectivityMonitorFactory,
      logLevel,
      defaultRequestOptionsFactory,
      defaultTransitionOptions,
      defaultRequestListeners,
      experiments);
}
```

这里直接调用了 RequestManagerRetriever 构造器，且传入参数实际上为 null，在 RequestManagerRetriever 的构造器方法中会为此创建一个默认的 DEFAULT_FACTORY：

```java
public class RequestManagerRetriever implements Handler.Callback {

  private final Handler handler;
  private final RequestManagerFactory factory;


  public RequestManagerRetriever(
      @Nullable RequestManagerFactory factory, GlideExperiments experiments) {
    this.factory = factory != null ? factory : DEFAULT_FACTORY;
    handler = new Handler(Looper.getMainLooper(), this /* Callback */);

    frameWaiter = buildFrameWaiter(experiments);
  }

  /**
   * Used internally to create {@link RequestManager}s.
   */
  public interface RequestManagerFactory {
    @NonNull
    RequestManager build(
        @NonNull Glide glide,
        @NonNull Lifecycle lifecycle,
        @NonNull RequestManagerTreeNode requestManagerTreeNode,
        @NonNull Context context);
  }

  private static final RequestManagerFactory DEFAULT_FACTORY = new RequestManagerFactory() {
    @NonNull
    @Override
    public RequestManager build(@NonNull Glide glide, @NonNull Lifecycle lifecycle,
        @NonNull RequestManagerTreeNode requestManagerTreeNode, @NonNull Context context) {
      return new RequestManager(glide, lifecycle, requestManagerTreeNode, context);
    }
  };
}
```

目前为止，Glide <strong>单例已经被创建出来了</strong>，其 RequestManagerRetriever 会作为 getRetriever(Context) 的返回值返回。

接下来回到 Glide.with 方法中，接着执行的是 RequestManagerRetriever.get 方法，该方法根据入参是对生命周期可感的。

### RequestManagerRetriever.get

RequestManagerRetriever.get 方法与 Glide.with 一样，也有很多重载方法：

```java
@NonNull
private RequestManager getApplicationManager(@NonNull Context context) {
  // Either an application context or we're on a background thread.
  if (applicationManager == null) {
    synchronized (this) {
      if (applicationManager == null) {
        // Normally pause/resume is taken care of by the fragment we add to the fragment or
        // activity. However, in this case since the manager attached to the application will not
        // receive lifecycle events, we must force the manager to start resumed using
        // ApplicationLifecycle.

        // TODO(b/27524013): Factor out this Glide.get() call.
        Glide glide = Glide.get(context.getApplicationContext());
        applicationManager =
            factory.build(
                glide,
                new ApplicationLifecycle(),
                new EmptyRequestManagerTreeNode(),
                context.getApplicationContext());
      }
    }
  }

  return applicationManager;
}

@NonNull
public RequestManager get(@NonNull Context context) {
  if (context == null) {
    throw new IllegalArgumentException("You cannot start a load on a null Context");
  } else if (Util.isOnMainThread() && !(context instanceof Application)) {
    if (context instanceof FragmentActivity) {
      return get((FragmentActivity) context);
    } else if (context instanceof Activity) {
      return get((Activity) context);
    } else if (context instanceof ContextWrapper
        // Only unwrap a ContextWrapper if the baseContext has a non-null application context.
        // Context#createPackageContext may return a Context without an Application instance,
        // in which case a ContextWrapper may be used to attach one.
        && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
      return get(((ContextWrapper) context).getBaseContext());
    }
  }

  return getApplicationManager(context);
}

@NonNull
public RequestManager get(@NonNull FragmentActivity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    frameWaiter.registerSelf(activity);
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}

@NonNull
public RequestManager get(@NonNull Fragment fragment) {
  Preconditions.checkNotNull(
      fragment.getContext(),
      "You cannot start a load on a fragment before it is attached or after it is destroyed");
  if (Util.isOnBackgroundThread()) {
    return get(fragment.getContext().getApplicationContext());
  } else {
    // In some unusual cases, it's possible to have a Fragment not hosted by an activity. There's
    // not all that much we can do here. Most apps will be started with a standard activity. If
    // we manage not to register the first frame waiter for a while, the consequences are not
    // catastrophic, we'll just use some extra memory.
    if (fragment.getActivity() != null) {
      frameWaiter.registerSelf(fragment.getActivity());
    }
    FragmentManager fm = fragment.getChildFragmentManager();
    return supportFragmentGet(fragment.getContext(), fm, fragment, fragment.isVisible());
  }
}

@SuppressWarnings("deprecation")
@NonNull
public RequestManager get(@NonNull Activity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else if (activity instanceof FragmentActivity) {
    return get((FragmentActivity) activity);
  } else {
    assertNotDestroyed(activity);
    frameWaiter.registerSelf(activity);
    android.app.FragmentManager fm = activity.getFragmentManager();
    return fragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}

@SuppressWarnings("deprecation")
@NonNull
public RequestManager get(@NonNull View view) {
  if (Util.isOnBackgroundThread()) {
    return get(view.getContext().getApplicationContext());
  }

  Preconditions.checkNotNull(view);
  Preconditions.checkNotNull(
      view.getContext(), "Unable to obtain a request manager for a view without a Context");
  Activity activity = findActivity(view.getContext());
  // The view might be somewhere else, like a service.
  if (activity == null) {
    return get(view.getContext().getApplicationContext());
  }

  // Support Fragments.
  // Although the user might have non-support Fragments attached to FragmentActivity, searching
  // for non-support Fragments is so expensive pre O and that should be rare enough that we
  // prefer to just fall back to the Activity directly.
  if (activity instanceof FragmentActivity) {
    Fragment fragment = findSupportFragment(view, (FragmentActivity) activity);
    return fragment != null ? get(fragment) : get((FragmentActivity) activity);
  }

  // Standard Fragments.
  android.app.Fragment fragment = findFragment(view, activity);
  if (fragment == null) {
    return get(activity);
  }
  return get(fragment);
}
```

在这些 get 方法中，<strong>首先判断当前线程是不是后台线程</strong>，如果是后台线程那么就会调用 getApplicationManager 方法返回一个 RequestManager：

```java
Glide glide = Glide.get(context.getApplicationContext());
applicationManager =
    factory.build(
        glide,
        new ApplicationLifecycle(),
        new EmptyRequestManagerTreeNode(),
        context.getApplicationContext());
```

由于此处 factory 是 DEFAULT_FACTORY，所以 RequestManager 就是下面的值：

```java
RequestManager(glide,
        new ApplicationLifecycle(),
        new EmptyRequestManagerTreeNode(),
        context.getApplicationContext());
```

<strong>如果当前线程不是后台线程</strong>，get(View) 和 get(Context) 会根据情况调用 get(Fragment) 或 get(FragmentActivity)。其中 get(View) 为了找到一个合适的 Fragment 或 Fallback Activity，内部操作比较多，开销比较大，不要轻易使用。

get(Fragment) 和 get(FragmentActivity) 方法都会调用 supportFragmentGet 方法，只是传入参数不同：

```java
// FragmentActivity activity
FragmentManager fm = activity.getSupportFragmentManager();
supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));

// Fragment fragment
FragmentManager fm = fragment.getChildFragmentManager();
supportFragmentGet(fragment.getActivity(), fm, fragment, fragment.isVisible());
```

Glide 会使用一个加载目标所在的宿主 Activity 或 Fragment 的子 Fragment 来安全保存一个 RequestManager，而 RequestManager 被 Glide 用来开始、停止、管理 Glide 请求。

而 supportFragmentGet 就是创建 / 获取这个 SupportRequestManagerFragment，并返回其持有的 RequestManager 的方法。

```java
@NonNull
private RequestManager supportFragmentGet(
    @NonNull Context context,
    @NonNull FragmentManager fm,
    @Nullable Fragment parentHint,
    boolean isParentVisible) {
  // 获取一个SupportRequestManagerFragment
  SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
  // 获取里面的RequestManager对象
  RequestManager requestManager = current.getRequestManager();
  // 若没有，则创建一个
  if (requestManager == null) {
    // TODO(b/27524013): Factor out this Glide.get() call.
    Glide glide = Glide.get(context);
    requestManager =
        factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
    // This is a bit of hack, we're going to start the RequestManager, but not the
    // corresponding Lifecycle. It's safe to start the RequestManager, but starting the
    // Lifecycle might trigger memory leaks. See b/154405040
    if (isParentVisible) {
      requestManager.onStart();
    }
    // 设置到SupportRequestManagerFragment里面，下次就不需要创建了
    current.setRequestManager(requestManager);
  }
  return requestManager;
}

// 看看Fragment怎么才能高效
@NonNull
private SupportRequestManagerFragment getSupportRequestManagerFragment(
    @NonNull final FragmentManager fm, @Nullable Fragment parentHint) {
  // 已经添加过了，可以直接返回
  SupportRequestManagerFragment current =
      (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
  if (current == null) {
    // 从map中获取，取到也可以返回了
    current = pendingSupportRequestManagerFragments.get(fm);
    if (current == null) {
      // 都没有，那么就创建一个，此时lifecycle默认为ActivityFragmentLifecycle
      current = new SupportRequestManagerFragment();
      // 对于fragment来说，此方法会以Activity为host创建另外一个
      // SupportRequestManagerFragment作为rootRequestManagerFragment
      // 并会将current加入到rootRequestManagerFragment的
      // childRequestManagerFragments中
      // 在RequestManager递归管理请求时会使用到
      current.setParentFragmentHint(parentHint);
      // 将刚创建的fragment缓存进map起来
      pendingSupportRequestManagerFragments.put(fm, current);
      // 将fragment添加到页面中
      fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
      // 以fm为key从pendingSupportRequestManagerFragments中删除
      handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
    }
  }
  return current;
}
```

在上面的 supportFragmentGet 方法中，成功创建了一个 RequestManager 对象，由于 factory 是 DEFAULT_FACTORY，所以就是下面的值：

```java
RequestManager(glide,
  current.getGlideLifecycle(),          // ActivityFragmentLifecycle()
  current.getRequestManagerTreeNode(),  // SupportFragmentRequestManagerTreeNode()
  context);
```

在上一步中 Glide 单例完成了初始化，这一步中成功的创建并返回了一个 RequestManager。Glide.with 已经分析完毕。

### 小结

with() 方法是为得到一个 RequestManager 对象，从而将 Glide 加载图片周期与 Activity 和 Fragment 进行绑定，进而管理 Glide 加载图片周期。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034143.png)

## <strong>load</strong>

由于 .with() 返回的是一个 RequestManager 对象，所以第 2 步中调用的是 RequestManager

类的 load() 方法。而 load() 方法返回的是 RequestBuilder 对象。RequestBuilder 和 RequestOptions 都派生自抽象类 BaseRequestOptions，它们的继承关系如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034149.png)

看一下 RequestManager 的一些方法，首先看 load 的一些重载方法：

```java
@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable Bitmap bitmap) {
  return asDrawable().load(bitmap);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable Drawable drawable) {
  return asDrawable().load(drawable);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable Uri uri) {
  return asDrawable().load(uri);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable File file) {
  return asDrawable().load(file);
}

@SuppressWarnings("deprecation")
@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
  return asDrawable().load(resourceId);
}

@SuppressWarnings("deprecation")
@CheckResult
@Override
@Deprecated
public RequestBuilder<Drawable> load(@Nullable URL url) {
  return asDrawable().load(url);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable byte[] model) {
  return asDrawable().load(model);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable Object model) {
  return asDrawable().load(model);
}
```

在所有的 RequestManager.load 方法中都会先调用 asDrawable() 方法得到一个 RequestBuilder 对象，然后再调用 RequestBuilder.load 方法。

### RequestManager.asXx

asDrawable 方法同其他 as 方法（asGif、asBitmap、asFile）一样，都会先调用 RequestManager.as 方法生成一个 `RequestBuilder<ResourceType>` 对象，然后各个 as 方法会附加一些不同的 options：

```java
@NonNull
@CheckResult
public RequestBuilder<Bitmap> asBitmap() {
  return as(Bitmap.class).apply(DECODE_TYPE_BITMAP);
}

@NonNull
@CheckResult
public RequestBuilder<GifDrawable> asGif() {
  return as(GifDrawable.class).apply(DECODE_TYPE_GIF);
}  

@NonNull
@CheckResult
public RequestBuilder<Drawable> asDrawable() {
  return as(Drawable.class);
}

@NonNull
@CheckResult
public RequestBuilder<File> asFile() {
  return as(File.class).apply(skipMemoryCacheOf(true));
}

@NonNull
@CheckResult
public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
  return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

在 RequestBuilder 的构造器方法方法中将 Drawable.class 这样的入参保存到了 transcodeClass 变量中：

```kotlin
@SuppressLint("CheckResult")
@SuppressWarnings("PMD.ConstructorCallsOverridableMethod")
protected RequestBuilder(
    @NonNull Glide glide,
    RequestManager requestManager,
    Class<TranscodeType> transcodeClass,
    Context context) {
  this.glide = glide;
  this.requestManager = requestManager;
  this.transcodeClass = transcodeClass;
  this.context = context;
  this.transitionOptions = requestManager.getDefaultTransitionOptions(transcodeClass);
  this.glideContext = glide.getGlideContext();

  initRequestListeners(requestManager.getDefaultRequestListeners());
  apply(requestManager.getDefaultRequestOptions());
}
```

然后回到之前的 asGif 方法中，看看 apply(DECODE_TYPE_BITMAP) 干了些什么：

```java
// RequestManager
private static final RequestOptions DECODE_TYPE_GIF 
        = RequestOptions.decodeTypeOf(GifDrawable.class).lock();

@NonNull
@CheckResult
public RequestBuilder<GifDrawable> asGif() {
  return as(GifDrawable.class).apply(DECODE_TYPE_GIF);
}

// RequestOptions
@NonNull
@CheckResult
public static RequestOptions decodeTypeOf(@NonNull Class<?> resourceClass) {
  return new RequestOptions().decode(resourceClass);
}

// BaseRequestOptions
@NonNull
@CheckResult
public T decode(@NonNull Class<?> resourceClass) {
  if (isAutoCloneEnabled) {
    return clone().decode(resourceClass);
  }

  this.resourceClass = Preconditions.checkNotNull(resourceClass);
  fields |= RESOURCE_CLASS;
  return selfOrThrowIfLocked();
}

@NonNull
@CheckResult
public T apply(@NonNull BaseRequestOptions<?> o) {
  if (isAutoCloneEnabled) {
    return clone().apply(o);
  }
  BaseRequestOptions<?> other = o;
  ...
  if (isSet(other.fields, RESOURCE_CLASS)) {
    resourceClass = other.resourceClass;
  }
  ...
  return selfOrThrowIfLocked();
}

// RequestBuilder
@NonNull
@CheckResult
@Override
public RequestBuilder<TranscodeType> apply(@NonNull BaseRequestOptions<?> requestOptions) {
  Preconditions.checkNotNull(requestOptions);
  return super.apply(requestOptions);
}
```

不难发现，apply(DECODE_TYPE_BITMAP) 就是将 BaseRequestOptions.resourceClass 设置为了 GifDrawable.class；对于 asBitmap() 来说，resourceClass 为 Bitmap.class；而对于 asDrawable() 和 asFile() 来说，resourceClass 没有进行过设置，所以为默认值 Object.class。

现在 RequestBuilder 已经由 as 系列方法生成，现在接着会调用 RequestBuilder.load 方法

### RequestBuilder.load

RequestManager.load 方法都会调用对应的 RequestBuilder.load 重载方法；RequestBuilder.load 的各个方法基本上都会直接转发给 loadGeneric 方法，只有少数的方法才会 apply 额外的 options。

loadGeneric 方法也只是保存一下参数而已：

```typescript
@NonNull
@CheckResult
@SuppressWarnings("unchecked")
@Override
public RequestBuilder<TranscodeType> load(@Nullable Object model) {
  return loadGeneric(model);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<TranscodeType> load(@Nullable Bitmap bitmap) {
  return loadGeneric(bitmap)
      .apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
}

@NonNull
@CheckResult
@Override
public RequestBuilder<TranscodeType> load(@Nullable Drawable drawable) {
  return loadGeneric(drawable)
      .apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
}

@NonNull
@CheckResult
@Override
public RequestBuilder<TranscodeType> load(@Nullable Uri uri) {
  return loadGeneric(uri);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<TranscodeType> load(@Nullable File file) {
  return loadGeneric(file);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<TranscodeType> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
  return loadGeneric(resourceId).apply(signatureOf(ApplicationVersionSignature.obtain(context)));
}

@Deprecated
@CheckResult
@Override
public RequestBuilder<TranscodeType> load(@Nullable URL url) {
  return loadGeneric(url);
}

@NonNull
@CheckResult
@Override
public RequestBuilder<TranscodeType> load(@Nullable byte[] model) {
  RequestBuilder<TranscodeType> result = loadGeneric(model);
  if (!result.isDiskCacheStrategySet()) {
      result = result.apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
  }
  if (!result.isSkipMemoryCacheSet()) {
    result = result.apply(skipMemoryCacheOf(true /*skipMemoryCache*/));
  }
  return result;
}

@NonNull
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model;
  isModelSet = true;
  return this;
}
```

如上面最后的方法 loadGeneric，这里只是将参数保存在 model 中并设置 isModelSet=true 就完了，

### 小结

load 流程主要给 GlideRequest（RequestManager）设置了要请求的 mode（url），并将 isModelSet  变量设置为 true，表示已设置的状态，最后返回 RequestBuilder。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034156.png)

## into

RequestBuilder.into 有四个重载方法，最终都调用了参数最多的一个：

```typescript
@NonNull
public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
  return into(target, /*targetListener=*/ null, Executors.mainThreadExecutor());
}

@NonNull
@Synthetic
<Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    Executor callbackExecutor) {
  return into(target, targetListener, /*options=*/ this, callbackExecutor);
}

private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    BaseRequestOptions<?> options,
    Executor callbackExecutor) {
  Preconditions.checkNotNull(target);
  if (!isModelSet) {
    throw new IllegalArgumentException("You must call #load() before calling #into()");
  }

  Request request = buildRequest(target, targetListener, options, callbackExecutor);

  Request previous = target.getRequest();
  if (request.isEquivalentTo(previous)
      && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
    // If the request is completed, beginning again will ensure the result is re-delivered,
    // triggering RequestListeners and Targets. If the request is failed, beginning again will
    // restart the request, giving it another chance to complete. If the request is already
    // running, we can let it continue running without interruption.
    if (!Preconditions.checkNotNull(previous).isRunning()) {
      // Use the previous request rather than the new one to allow for optimizations like skipping
      // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
      // that are done in the individual Request.
      previous.begin();
    }
    return target;
  }

  requestManager.clear(target);
  target.setRequest(request);
  requestManager.track(target, request);

  return target;
}

// 最常用的一个重载

@NonNull
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread();
  Preconditions.checkNotNull(view);

  BaseRequestOptions<?> requestOptions = this;
  // 若没有指定transform，isTransformationSet()为false
  // isTransformationAllowed()一般为true，除非主动调用了dontTransform()方法
  if (!requestOptions.isTransformationSet()
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    // Clone in this method so that if we use this RequestBuilder to load into a View and then
    // into a different target, we don't retain the transformation applied based on the previous
    // View's scale type.
    // 根据ImageView的ScaleType设置不同的down sample和transform选项
    switch (view.getScaleType()) {
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER: // 默认值
      case FIT_START:
      case FIT_END:
        requestOptions = requestOptions.clone().optionalFitCenter();
        break;
      case FIT_XY:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case CENTER:
      case MATRIX:
      default:
        // Do nothing.
    }
  }

  // 调用上面的重载方法
  return into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions,
      Executors.mainThreadExecutor());
}
```

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034202.png)

## <strong>整体流程</strong>

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034206.png)

# <strong>缓存</strong>

## Glide 缓存机制

Glide 之所以被广泛使用的一个重要原因就是它强大的缓存机制。在 Glide 官方文档中，在[缓存部分](https://muyangmin.github.io/glide-docs-cn/doc/caching.html)有如下描述：

默认情况下，Glide 会在开始一个新的图片请求之前检查以下多级的缓存：

1、活动资源 (Active Resources) - 现在是否有另一个 View 正在展示这张图片？

2、内存缓存 (Memory Cache) - 该图片是否最近被加载过并仍存在于内存中？

3、资源类型（Resource） - 该图片是否之前曾被解码、转换并写入过磁盘缓存？

4、数据来源 (Data) - 构建这个图片的资源是否之前曾被写入过文件缓存？

前两步检查图片是否在内存中，如果是则直接返回图片。后两步则检查图片是否在磁盘上，以便快速但异步地返回图片。

如果四个步骤都未能找到图片，则 Glide 会返回到原始资源以取回数据（原始文件，Uri, Url 等）。

Glide 缓存机制的流程图如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034424.png)

Glide 的 memory cache 和 disk cache 在 Glide 创建的时候就确定了。代码在 GlideBuilder.build(Context) 方法里面：

```java
@NonNull
Glide build(@NonNull Context context) {
  ...
  if (memoryCache == null) {
    memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
  }
  if (diskCacheFactory == null) {
    diskCacheFactory = new InternalCacheDiskCacheFactory(context);
  }
  if (engine == null) {
    engine = new Engine(memoryCache, diskCacheFactory, ...);
  }
  ...
  return new Glide(context, engine, memoryCache, ...);
}
```

如果没有在任何 GlideModule 中进行自定义的话，memoryCache 和 diskCacheFactory 会使用一个默认值，大部分情况下使用这个默认值去实现缓存就可以了。

## <strong>配置缓存</strong>

### 磁盘缓存策略

DiskCacheStrategy 可被 diskCacheStrategy() 方法应用到每一个单独的请求。 目前支持的策略允许你阻止加载过程使用或写入磁盘缓存，选择性地仅缓存无修改的原生数据，或仅缓存变换过的缩略图，或是兼而有之。磁盘缓存一共有以下几种策略：

- DiskCacheStrategy.AUTOMATIC：尝试对本地和远程图片使用最佳的策略，它是默认使用的策略。当加载远程数据（比如从 URL 下载）时，仅会存储未被加载过程修改过（比如变换）的原始数据。对于本地数据，则会仅存储变换过的缩略图，因为即使需要再次生成另一个尺寸或类型的图片，取回原始数据也很容易。
- DiskCacheStrategy.NONE： 表示不缓存任何内容。
- DiskCacheStrategy.DATA： 表示只缓存原始图片。
- DiskCacheStrategy.RESOURCE： 表示只缓存转换过后的图片。
- DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。

<strong>指定 DiskCacheStrategy :</strong>

```java
Glide.with(fragment)
  .load(url)
  .diskCacheStrategy(DiskCacheStrategy.ALL)
  .into(imageView);
```

### 仅从缓存加载图片

某些情形下，可能希望只要图片不在缓存中则加载直接失败（比如省流量模式？）。如果要达到这种效果，可以在单个请求的基础上使用 onlyRetrieveFromCache 方法：

```java
Glide.with(fragment)
  .load(url)
  .onlyRetrieveFromCache(true)
  .into(imageView);
```

这时如果图片在内存缓存或在磁盘缓存中，它会被展示出来。否则只要这个选项被设置为 true ，这次加载会视同失败。

### 跳过缓存

如果你想确保一个特定的请求跳过磁盘和/或内存缓存（<em>比如，图片验证码</em>），Glide 也提供了一些替代方案。

仅跳过内存缓存，请使用 skipMemoryCache() :

```java
Glide.with(fragment)
  .load(url)
  .skipMemoryCache(true)
  .into(view);
```

仅跳过磁盘缓存，请使用 DiskCacheStrategy.NONE :

```java
Glide.with(fragment)
  .load(url)
  .diskCacheStrategy(DiskCacheStrategy.NONE)
  .into(view);
```

这两个选项可以同时使用:

```java
Glide.with(fragment)
  .load(url)
  .diskCacheStrategy(DiskCacheStrategy.NONE)
  .skipMemoryCache(true)
  .into(view);
```

虽然提供了这些选项跳过缓存，但不建议这么做。因为从缓存中加载一个图片，要比拉取-解码-转换成一张新图片的完整流程快得多。

## 活动资源 ActiveResource

ActiveResources 在 Engine 的构造器中被创建，它的源码如下：

```java
final class ActiveResources {
  private final boolean isActiveResourceRetentionAllowed;
  private final Executor monitorClearedResourcesExecutor;
  final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
  private final ReferenceQueue<EngineResource<?>> resourceReferenceQueue = new ReferenceQueue<>();

  private ResourceListener listener;

  private volatile boolean isShutdown;
  private volatile DequeuedResourceCallback cb;

  ActiveResources(boolean isActiveResourceRetentionAllowed) {
    this(
        isActiveResourceRetentionAllowed,
        java.util.concurrent.Executors.newSingleThreadExecutor(
            new ThreadFactory() {
              @Override
              public Thread newThread(@NonNull final Runnable r) {
                return new Thread(
                    new Runnable() {
                      @Override
                      public void run() {
                        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                        r.run();
                      }
                    },
                    "glide-active-resources");
              }
            }));
  }

  ActiveResources(
      boolean isActiveResourceRetentionAllowed, Executor monitorClearedResourcesExecutor) {
    this.isActiveResourceRetentionAllowed = isActiveResourceRetentionAllowed;
    this.monitorClearedResourcesExecutor = monitorClearedResourcesExecutor;

    monitorClearedResourcesExecutor.execute(
        new Runnable() {
          @Override
          public void run() {
            cleanReferenceQueue();
          }
        });
  }

  void setListener(ResourceListener listener) {
    synchronized (listener) {
      synchronized (this) {
        this.listener = listener;
      }
    }
  }

  synchronized void activate(Key key, EngineResource<?> resource) {
    ResourceWeakReference toPut =
        new ResourceWeakReference(
            key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);

    ResourceWeakReference removed = activeEngineResources.put(key, toPut);
    if (removed != null) {
      removed.reset();
    }
  }

  synchronized void deactivate(Key key) {
    ResourceWeakReference removed = activeEngineResources.remove(key);
    if (removed != null) {
      removed.reset();
    }
  }

  synchronized EngineResource<?> get(Key key) {
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
      return null;
    }

    EngineResource<?> active = activeRef.get();
    if (active == null) {
      cleanupActiveReference(activeRef);
    }
    return active;
  }

  void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    synchronized (this) {
      activeEngineResources.remove(ref.key);

      if (!ref.isCacheable || ref.resource == null) {
        return;
      }
    }

    EngineResource<?> newResource =
        new EngineResource<>(
            ref.resource, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ false, ref.key, listener);
    listener.onResourceReleased(ref.key, newResource);
  }

  void cleanReferenceQueue() {
    while (!isShutdown) {
      try {
        ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
        cleanupActiveReference(ref);

        DequeuedResourceCallback current = cb;
        if (current != null) {
          current.onResourceDequeued();
        }
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }

  void setDequeuedResourceCallback(DequeuedResourceCallback cb) {
    this.cb = cb;
  }

  interface DequeuedResourceCallback {
    void onResourceDequeued();
  }

  void shutdown() {
    isShutdown = true;
    if (monitorClearedResourcesExecutor instanceof ExecutorService) {
      ExecutorService service = (ExecutorService) monitorClearedResourcesExecutor;
      Executors.shutdownAndAwaitTermination(service);
    }
  }

  static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
    final Key key;
    final boolean isCacheable;
    Resource<?> resource;
    ResourceWeakReference(Key key, EngineResource<?> referent,
        ReferenceQueue<? super EngineResource<?>> queue,
        boolean isActiveResourceRetentionAllowed) {
      super(referent, queue);
      this.key = Preconditions.checkNotNull(key);
      this.resource =
          referent.isMemoryCacheable() && isActiveResourceRetentionAllowed
              ? Preconditions.checkNotNull(referent.getResource())
              : null;
      isCacheable = referent.isMemoryCacheable();
    }

    void reset() {
      resource = null;
      clear();
    }
  }
}
```

先来看 ResourceWeakReference，`extends WeakReference<EngineResource<?>>` 表示它继承自 WeakReference 并且用于保存 EngineResource 类型的引用，此外添加了一个 reset 方法用来清理资源。

构造方法中调用了 super(referent, queue)，这样如果 referent 指向的对象将要被 GC 的时候，referent 就会被放入 queue 中。

ActiveResources 初始化时，会启动一个名为“glide-active-resources”的 BACKGROUND 的线程，在该线程中会调用 cleanReferenceQueue() 方法：

```java
void cleanReferenceQueue() {
    while (!isShutdown) {
      try {
        //remove 方法会发生阻塞
        ResourceWeakReference ref = (ResourceWeakReference) 
            resourceReferenceQueue.remove();
        //清除资源
        cleanupActiveReference(ref);

        DequeuedResourceCallback current = cb;
        if (current != null) {
          current.onResourceDequeued();
        }
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
```

而 resourceReferenceQueue.remove() 方法会一直尝试从 queue 中获取将要被 GC 的 EngineResource。当发生 GC 时，如果 EngineResource 只 ResourceWeakReference  对象持有，ResourceWeakReference 对象就会被插入到 queue 中，这时在 remove() 方法就能获取到一个 ResourceWeakReference  对象（这是一个典型的生产者-消费者模型）。

然后调用 cleanupActiveReference 方法：

```java
void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    synchronized (this) {
      //从集合中移除
      activeEngineResources.remove(ref.key);

      if (!ref.isCacheable || ref.resource == null) {
        return;
      }
    }

    EngineResource<?> newResource =
        new EngineResource<>(
            ref.resource, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ false, ref.key, listener);
    //将被 GC 的 EngineResource 回调给 listener
    listener.onResourceReleased(ref.key, newResource);
  }
```

首先会将该对象从集合中移除，然后再将 ResourceWeakReference  对象中的资源取出并回调给 listener，这个 listener 的唯一实现在 Engine 类中，后面还会有地方调用这个方法，这里暂时不进行分析。

该方法除了在此时被调用外，还在 ActiveResources.get(key) 方法中也可能会因为获取到的 resource 为 null 而被调用。

## 内存缓存 MemoryCache

MemoryCache 发生存取操作是在 Engine 中，但是我们看到 MemoryCache 还被放入了 Glide 实例中。这是因为 Glide 实现了 ComponentCallbacks2 接口，在 Glide 创建完成后，实例就注册了该接口。这样在内存紧张的时候，可以通过回调 onTrimMemory 方法通知释放内存。

```java
@Override
public void onTrimMemory(int level) {
  trimMemory(level);
}

public void trimMemory(int level) {
  // Engine asserts this anyway when removing resources, fail faster and consistently
  Util.assertMainThread();
  // Request managers need to be trimmed before the caches and pools, in order for the latter to
  // have the most benefit.
  synchronized (managers) {
    for (RequestManager manager : managers) {
      manager.onTrimMemory(level);
    }
  }
  // memory cache needs to be trimmed before bitmap pool to trim re-pooled Bitmaps too. See #687.
  memoryCache.trimMemory(level);
  bitmapPool.trimMemory(level);
  arrayPool.trimMemory(level);
}
```

前面提到过，Glide 初始化的时候如果没有设置过内存缓存方案，则会采用默认实现方案：

```java
if (memoryCache == null) {
    memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
}
```

LruResourceCache 继承自 LruCache 类并实现了 MemoryCache 接口，LruCache 使用 LRU（least recently used，即最近最少使用）算法实现的内存淘汰算法，它的核心思想是当缓存对象的数量达到阈值时，会优先删除那些近期最少使用的缓存对象。

LruCache 定义了 LRU 相关的操作，而 MemoryCache 定义的是内存缓存相关的操作。LruResourceCache 源码如下：

```java
public class LruResourceCache extends LruCache<Key, Resource<?>> implements MemoryCache {
  private ResourceRemovedListener listener;

  /**
   * Constructor for LruResourceCache.
   *
   * @param size The maximum size in bytes the in memory cache can use.
   */
  public LruResourceCache(long size) {
    super(size);
  }

  @Override
  public void setResourceRemovedListener(@NonNull ResourceRemovedListener listener) {
    this.listener = listener;
  }

  @Override
  protected void onItemEvicted(@NonNull Key key, @Nullable Resource<?> item) {
    if (listener != null && item != null) {
      listener.onResourceRemoved(item);
    }
  }

  @Override
  protected int getSize(@Nullable Resource<?> item) {
    if (item == null) {
      return super.getSize(null);
    } else {
      return item.getSize();
    }
  }

  @Override
  public void trimMemory(int level) {
    if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
      // Entering list of cached background apps
      // Evict our entire bitmap cache
      clearMemory();
    } else if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN
        || level == android.content.ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL) {
      // The app's UI is no longer visible, or app is in the foreground but system is running
      // critically low on memory
      // Evict oldest half of our bitmap cache
      trimToSize(getMaxSize() / 2);
    }
  }
}
LruCache 的源码如下：
public class LruCache<T, Y> {
  private final Map<T, Entry<Y>> cache = new LinkedHashMap<>(100, 0.75f, true);
  private final long initialMaxSize;
  private long maxSize;
  private long currentSize;

  public LruCache(long size) {
    this.initialMaxSize = size;
    this.maxSize = size;
  }

  public synchronized void setSizeMultiplier(float multiplier) {
    if (multiplier < 0) {
      throw new IllegalArgumentException("Multiplier must be >= 0");
    }
    maxSize = Math.round(initialMaxSize * multiplier);
    evict();
  }

  protected int getSize(@Nullable Y item) {
    return 1;
  }

  protected synchronized int getCount() {
    return cache.size();
  }

  protected void onItemEvicted(@NonNull T key, @Nullable Y item) {
    // optional override
  }

  public synchronized long getMaxSize() {
    return maxSize;
  }

  public synchronized long getCurrentSize() {
    return currentSize;
  }

  public synchronized boolean contains(@NonNull T key) {
    return cache.containsKey(key);
  }

  @Nullable
  public synchronized Y get(@NonNull T key) {
    Entry<Y> entry = cache.get(key);
    return entry != null ? entry.value : null;
  }

  @Nullable
  public synchronized Y put(@NonNull T key, @Nullable Y item) {
    final int itemSize = getSize(item);
    if (itemSize >= maxSize) {
      onItemEvicted(key, item);
      return null;
    }
    if (item != null) {
      currentSize += itemSize;
    }
    @Nullable Entry<Y> old = cache.put(key, item == null ? null : new Entry<>(item, itemSize));
    if (old != null) {
      currentSize -= old.size;
      if (!old.value.equals(item)) {
        onItemEvicted(key, old.value);
      }
    }
    evict();
    return old != null ? old.value : null;
  }

  @Nullable
  public synchronized Y remove(@NonNull T key) {
    Entry<Y> entry = cache.remove(key);
    if (entry == null) {
      return null;
    }
    currentSize -= entry.size;
    return entry.value;
  }
  
  public void clearMemory() {
    trimToSize(0);
  }

  protected synchronized void trimToSize(long size) {
    Map.Entry<T, Entry<Y>> last;
    Iterator<Map.Entry<T, Entry<Y>>> cacheIterator;
    while (currentSize > size) {
      //从头开始遍历
      cacheIterator = cache.entrySet().iterator();
      //第一个元素
      last = cacheIterator.next();
      final Entry<Y> toRemove = last.getValue();
      currentSize -= toRemove.size;
      final T key = last.getKey();
      cacheIterator.remove();
      onItemEvicted(key, toRemove.value);
    }
  }
  
  private void evict() {
    trimToSize(maxSize);
  }
  
  @Synthetic
  static final class Entry<Y> {
    final Y value;
    final int size;
    @Synthetic
    Entry(Y value, int size) {
      this.value = value;
      this.size = size;
    }
  }
}
```

LruCache 内部持有一个 LinkedHashMap 的变量 cache ：

```java
private final Map<T, Entry<Y>> cache = 
      new LinkedHashMap<>(100, 0.75f, true);
```

当第三个参数 accessOrder 为 true 时，表示按照访问顺序排序，每次调用 LinkedHashMap 的 get(key) 或 getOrDefault(key, defaultValue) 方法都会触发 afterNodeAccess(Object) 方法，此方法会将对应的 node 移动到链表的末尾。也就是说 LinkedHashMap 末尾的数据是最近最多使用的。

而 LruCache 清除内存时都会调用 trimToSize(size) 方法时，会从头到尾进行清理，即清理掉最少使用的元素。

LinkedHashMap 主要代码如下：

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {
    ...
    public LinkedHashMap(int initialCapacity, 
            float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

    public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;
       if (accessOrder)
           afterNodeAccess(e);
       return e.value;
    }
    
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
    ...
}
```

## <strong>磁盘缓存 DiskCache</strong>

Glide 初始化时，如果 disCacheFactory 如果没有被指定，则会使用一个默认值，而这个 InternalCacheDiskCacheFactory 就是实现磁盘缓存的入口。

```java
if (diskCacheFactory == null) {
    diskCacheFactory = new InternalCacheDiskCacheFactory(context);
}
```

InternalCacheDiskCacheFactory 源码如下：

```java
public final class InternalCacheDiskCacheFactory extends DiskLruCacheFactory {

  public InternalCacheDiskCacheFactory(Context context) {
    this(
        context,
        DiskCache.Factory.DEFAULT_DISK_CACHE_DIR,
        DiskCache.Factory.DEFAULT_DISK_CACHE_SIZE);
  }

  public InternalCacheDiskCacheFactory(Context context, long diskCacheSize) {
    this(context, DiskCache.Factory.DEFAULT_DISK_CACHE_DIR, diskCacheSize);
  }

  public InternalCacheDiskCacheFactory(
      final Context context, final String diskCacheName, long diskCacheSize) {
    super(
        new CacheDirectoryGetter() {
          @Override
          public File getCacheDirectory() {
            File cacheDirectory = context.getCacheDir();
            if (cacheDirectory == null) {
              return null;
            }
            if (diskCacheName != null) {
              return new File(cacheDirectory, diskCacheName);
            }
            return cacheDirectory;
          }
        },
        diskCacheSize);
  }
}
```

它的父类 DiskLruCacheFactory 和 DiskCache 类源码如下：

```java
public class DiskLruCacheFactory implements DiskCache.Factory {
  private final long diskCacheSize;
  private final CacheDirectoryGetter cacheDirectoryGetter;

  /** Interface called out of UI thread to get the cache folder. */
  public interface CacheDirectoryGetter {
    File getCacheDirectory();
  }

  public DiskLruCacheFactory(final String diskCacheFolder, long diskCacheSize) {
    this(
        new CacheDirectoryGetter() {
          @Override
          public File getCacheDirectory() {
            return new File(diskCacheFolder);
          }
        },
        diskCacheSize);
  }

  public DiskLruCacheFactory(
      final String diskCacheFolder, final String diskCacheName, long diskCacheSize) {
    this(
        new CacheDirectoryGetter() {
          @Override
          public File getCacheDirectory() {
            return new File(diskCacheFolder, diskCacheName);
          }
        },
        diskCacheSize);
  }

  @SuppressWarnings("WeakerAccess")
  public DiskLruCacheFactory(CacheDirectoryGetter cacheDirectoryGetter, long diskCacheSize) {
    this.diskCacheSize = diskCacheSize;
    this.cacheDirectoryGetter = cacheDirectoryGetter;
  }

  @Override
  public DiskCache build() {
    File cacheDir = cacheDirectoryGetter.getCacheDirectory();

    if (cacheDir == null) {
      return null;
    }

    if (cacheDir.isDirectory() || cacheDir.mkdirs()) {
      return DiskLruCacheWrapper.create(cacheDir, diskCacheSize);
    }

    return null;
  }
}

public interface DiskCache {
  /** An interface for lazily creating a disk cache. */
  interface Factory {
    /** 250 MB of cache. */
    int DEFAULT_DISK_CACHE_SIZE = 250 * 1024 * 1024;
    String DEFAULT_DISK_CACHE_DIR = "image_manager_disk_cache";
    /** Returns a new disk cache, or {@code null} if no disk cache could be created. */
    @Nullable
    DiskCache build();
  }
  /** An interface to actually write data to a key in the disk cache. */
  interface Writer {
    boolean write(@NonNull File file);
  }

  @Nullable
  File get(Key key);

  void put(Key key, Writer writer);

  void delete(Key key);

  void clear();
}
```

DiskLruCacheFactory.build() 方法会返回一个 DiskLruCacheWrapper 类的实例。回溯到前面 InternalCacheDiskCacheFactory 的创建流程可知，默认创建的 DiskLruCacheWrapper 传入的 diskCacheSize 为 250M 并且 cacheDir 为 <em>/data/data/{package}/cache/image_manager_disk_cache/ </em>，分别对应缓存大小和目录。

接下来查看 DiskLruCacheWrapper 源码：

```java
public class DiskLruCacheWrapper implements DiskCache {
  private static final String TAG = "DiskLruCacheWrapper";

  private static final int APP_VERSION = 1;
  private static final int VALUE_COUNT = 1;
  private static DiskLruCacheWrapper wrapper;

  private final SafeKeyGenerator safeKeyGenerator;
  private final File directory;
  private final long maxSize;
  private final DiskCacheWriteLocker writeLocker = new DiskCacheWriteLocker();
  private DiskLruCache diskLruCache;

  ...
  
  @SuppressWarnings("deprecation")
  public static DiskCache create(File directory, long maxSize) {
    return new DiskLruCacheWrapper(directory, maxSize);
  }

  @SuppressWarnings({"WeakerAccess", "DeprecatedIsStillUsed"})
  protected DiskLruCacheWrapper(File directory, long maxSize) {
    this.directory = directory;
    this.maxSize = maxSize;
    this.safeKeyGenerator = new SafeKeyGenerator();
  }

  private synchronized DiskLruCache getDiskCache() throws IOException {
    if (diskLruCache == null) {
      diskLruCache = DiskLruCache.open(directory, APP_VERSION, VALUE_COUNT, maxSize);
    }
    return diskLruCache;
  }

  @Override
  public File get(Key key) {
    String safeKey = safeKeyGenerator.getSafeKey(key);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Get: Obtained: " + safeKey + " for for Key: " + key);
    }
    File result = null;
    try {
      final DiskLruCache.Value value = getDiskCache().get(safeKey);
      if (value != null) {
        result = value.getFile(0);
      }
    } catch (IOException e) {
      if (Log.isLoggable(TAG, Log.WARN)) {
        Log.w(TAG, "Unable to get from disk cache", e);
      }
    }
    return result;
  }

  @Override
  public void put(Key key, Writer writer) {
    String safeKey = safeKeyGenerator.getSafeKey(key);
    writeLocker.acquire(safeKey);
    try {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Put: Obtained: " + safeKey + " for for Key: " + key);
      }
      try {
        DiskLruCache diskCache = getDiskCache();
        Value current = diskCache.get(safeKey);
        if (current != null) {
          return;
        }

        DiskLruCache.Editor editor = diskCache.edit(safeKey);
        if (editor == null) {
          throw new IllegalStateException("Had two simultaneous puts for: " + safeKey);
        }
        try {
          File file = editor.getFile(0);
          if (writer.write(file)) {
            editor.commit();
          }
        } finally {
          editor.abortUnlessCommitted();
        }
      } catch (IOException e) {
        if (Log.isLoggable(TAG, Log.WARN)) {
          Log.w(TAG, "Unable to put to disk cache", e);
        }
      }
    } finally {
      writeLocker.release(safeKey);
    }
  }

  @Override
  public void delete(Key key) {
    String safeKey = safeKeyGenerator.getSafeKey(key);
    try {
      getDiskCache().remove(safeKey);
    } catch (IOException e) {
      if (Log.isLoggable(TAG, Log.WARN)) {
        Log.w(TAG, "Unable to delete from disk cache", e);
      }
    }
  }

  @Override
  public synchronized void clear() {
    try {
      getDiskCache().delete();
    } catch (IOException e) {
      if (Log.isLoggable(TAG, Log.WARN)) {
        Log.w(TAG, "Unable to clear disk cache or disk cache cleared externally", e);
      }
    } finally {
      resetDiskCache();
    }
  }

  private synchronized void resetDiskCache() {
    diskLruCache = null;
  }
}
```

该类主要提供了一个生成 Key 的 SafeKeyGenerator 对象以及写锁 DiskCacheWriteLocker。从类名来看这是一个包装类，在代码中也可以看到，它的 get 和 put 方法最终都是通过 DiskLruCache 对象去实现的。

前面提到过，disCacheFactory 对象会被作为参数参与 Engine 的构造，在 Engine 的构造方法中会被包装成为一个 LazyDiskCacheProvider：

```java
Engine(MemoryCache cache, DiskCache.Factory diskCacheFactory, ...) {
  this.cache = cache;
  this.diskCacheProvider = new LazyDiskCacheProvider(diskCacheFactory);
  ...
}
LazyDiskCacheProvider 源码如下：
private static class LazyDiskCacheProvider implements DecodeJob.DiskCacheProvider {

  private final DiskCache.Factory factory;
  private volatile DiskCache diskCache;

  LazyDiskCacheProvider(DiskCache.Factory factory) {
    this.factory = factory;
  }

  @VisibleForTesting
  synchronized void clearDiskCacheIfCreated() {
    if (diskCache == null) {
      return;
    }
    diskCache.clear();
  }

  @Override
  public DiskCache getDiskCache() {
    if (diskCache == null) {
      synchronized (this) {
        if (diskCache == null) {
          diskCache = factory.build();
        }
        if (diskCache == null) {
          diskCache = new DiskCacheAdapter();
        }
      }
    }
    return diskCache;
  }
}
```

getDiskCache() 方法被调用时，会通过 DCL 的方式进而调用 factory 的 build( ) 方法返回一个 DiskCache 作为单例。

以上就是 Glide 缓存的介绍，前面加载流程的分析只是在首次加载图片的时候的流程，并没有涉及到缓存的概念，因此接下来继续在以上 into 流程中重点分析缓存部分的源码。

## 缓存流程

### EngineKey

从之前的分析中可知，整个加载的过程体现在 Engine.load 方法中，该方法注释和代码片段如下：

```java
public <R> LoadStatus load(...) {
  long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

  EngineKey key =
      keyFactory.buildKey(
          model,
          signature,
          width,
          height,
          transformations,
          resourceClass,
          transcodeClass,
          options);

  EngineResource<?> memoryResource;
  synchronized (this) {
    memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

    if (memoryResource == null) {
      //没有获取到活动资源和 MemoryCache，则启动 Job
      return waitForExistingOrStartNewJob(...);
    }
  }

  // Avoid calling back while holding the engine lock, doing so makes it easier for callers to
  // deadlock.
  cb.onResourceReady(
      memoryResource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
  return null;
}

@Nullable
private EngineResource<?> loadFromMemory(
    EngineKey key, boolean isMemoryCacheable, long startTime) {
  if (!isMemoryCacheable) {
    return null;
  }
  EngineResource<?> active = loadFromActiveResources(key);
  if (active != null) {
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Loaded resource from active resources", startTime, key);
    }
    return active;
  }
  EngineResource<?> cached = loadFromCache(key);
  if (cached != null) {
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Loaded resource from cache", startTime, key);
    }
    return cached;
  }
  return null;
}

@Nullable
private EngineResource<?> loadFromActiveResources(Key key) {
  EngineResource<?> active = activeResources.get(key);
  if (active != null) {
    active.acquire();
  }
  return active;
}

private EngineResource<?> loadFromCache(Key key) {
  EngineResource<?> cached = getEngineResourceFromCache(key);
  if (cached != null) {
    cached.acquire();
    activeResources.activate(key, cached);
  }
  return cached;
}

private EngineResource<?> getEngineResourceFromCache(Key key) {
  Resource<?> cached = cache.remove(key);
  final EngineResource<?> result;
  if (cached == null) {
    result = null;
  } else if (cached instanceof EngineResource) {
    // Save an object allocation if we've cached an EngineResource (the typical case).
    result = (EngineResource<?>) cached;
  } else {
    result =
        new EngineResource<>(
            cached, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ true, key, /*listener=*/ this);
  }
  return result;
}
```

从注释和代码中可以看出，首先会根据 key 尝试从 loadFromActiveResources() 方法中获取 ActiveResource，如果能够获取到则直接返回；否则从 loadFromCache() 方法中获取 memory cache；如果仍然没有获取到，最后就交给了 Job 从网络中获取资源。没有意外的话，磁盘缓存也是由 Job 完成的。

只要是缓存，就有 get 和 put 操作，也就离不开 Key，所以先看看从 ActiveResource 和  MemoryCache 中取缓存时的 Key——EngineKey 的组成参数：

| 参数             | 含义                                                                                                                                                                                |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| model            | load 的参数。如：File、Url、Url 等。如果使用自定义的 model, 它需要正确地实现 hashCode() 和 equals()                                                                                 |
| signature        | BaseRequestOptions 的成员变量，默认会是 EmptySignature.obtain()<br/>在加载本地 resource 资源时会变成 ApplicationVersionSignature.obtain(context)                                    |
| width<br/>height | 如果没有指定 override(int size)，则为 view 的 size                                                                                                                                  |
| transformations  | 默认会基于 ImageView 的 scaleType 设置对应的四个 Transformation；<br/>如果指定了 transform，那么就基于该值进行设置；<br/>详见 BaseRequestOptions.transform(Transformation, boolean) |
| resourceClass    | 解码后的资源，如果没有 asBitmap、asGif，一般会是 Object                                                                                                                             |
| transcodeClass   | 最终要转换成的数据类型，根据 as 方法确定，加载本地 res 或者网络 URL，都会调用 asDrawable，所以为 Drawable                                                                           |
| options          | 如果没有设置过 transform，此处会根据 ImageView 的 scaleType 默认指定一个 KV                                                                                                         |

再来看 EngineKey 的内部：

```java
class EngineKey implements Key {
  ...
  @Override
  public boolean equals(Object o) {
    if (o instanceof EngineKey) {
      EngineKey other = (EngineKey) o;
      return model.equals(other.model)
          && signature.equals(other.signature)
          && height == other.height
          && width == other.width
          && transformations.equals(other.transformations)
          && resourceClass.equals(other.resourceClass)
          && transcodeClass.equals(other.transcodeClass)
          && options.equals(other.options);
    }
    return false;
  }
  
  @Override
  public int hashCode() {
    if (hashCode == 0) {
      hashCode = model.hashCode();
      hashCode = 31 * hashCode + signature.hashCode();
      hashCode = 31 * hashCode + width;
      hashCode = 31 * hashCode + height;
      hashCode = 31 * hashCode + transformations.hashCode();
      hashCode = 31 * hashCode + resourceClass.hashCode();
      hashCode = 31 * hashCode + transcodeClass.hashCode();
      hashCode = 31 * hashCode + options.hashCode();
    }
    return hashCode;
  }
  ...
}
```

可以看到，它的 equals 和 hashCode 方法都是标准的写法，所有成员变量都参与了判断和运算。因此在多次加载同一个 model 的过程中，即使有稍许改动（比如 View 宽高、是否经过变换等），都会导致没有获取内存缓存失败。

### 内存缓存

#### ActiveResource 和  MemoryCache

ActiveResource 的 Value 为 ResourceWeakReference，它是 EngineResource 的包装类； MemoryCache 则是直接保存 EngineResource 对象。看看 EngineResource 的内容：

```java
class EngineResource<Z> implements Resource<Z> {
  private final boolean isMemoryCacheable;
  private final boolean isRecyclable;
  private final Resource<Z> resource;
  private final ResourceListener listener;
  private final Key key;

  private int acquired;
  private boolean isRecycled;

  interface ResourceListener {
    void onResourceReleased(Key key, EngineResource<?> resource);
  }

  EngineResource(
      Resource<Z> toWrap,
      boolean isMemoryCacheable,
      boolean isRecyclable,
      Key key,
      ResourceListener listener) {
    resource = Preconditions.checkNotNull(toWrap);
    this.isMemoryCacheable = isMemoryCacheable;
    this.isRecyclable = isRecyclable;
    this.key = key;
    this.listener = Preconditions.checkNotNull(listener);
  }

  Resource<Z> getResource() {
    return resource;
  }

  boolean isMemoryCacheable() {
    return isMemoryCacheable;
  }

  @NonNull
  @Override
  public Class<Z> getResourceClass() {
    return resource.getResourceClass();
  }

  @NonNull
  @Override
  public Z get() {
    return resource.get();
  }

  @Override
  public int getSize() {
    return resource.getSize();
  }

  @Override
  public synchronized void recycle() {
    if (acquired > 0) {
      throw new IllegalStateException("Cannot recycle a resource while it is still acquired");
    }
    if (isRecycled) {
      throw new IllegalStateException("Cannot recycle a resource that has already been recycled");
    }
    isRecycled = true;
    if (isRecyclable) {
      resource.recycle();
    }
  }

  synchronized void acquire() {
    if (isRecycled) {
      throw new IllegalStateException("Cannot acquire a recycled resource");
    }
    ++acquired;
  }

  void release() {
    boolean release = false;
    synchronized (this) {
      if (acquired <= 0) {
        throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
      }
      if (--acquired == 0) {
        release = true;
      }
    }
    if (release) {
      listener.onResourceReleased(key, this);
    }
  }
  ...
}
```

可以看到，acquire() 方法内部会对一个整型变量 acquired 进行自增操作，这个变量表示该资源被引用的次数，当它大于 0 的时候表示图片正在使用中。相应地，在 release() 方法内部会 acquired 进行自减，并且当变量为 0 的时候，表示没有被任何地方引用，这时会调用 listener.onResourceReleased(key, this) 通知监听器资源已被释放，从而进行后续操作。

acquire() 和 release() 方法在一般情况下外部是不能主动调用的，只有 Glide 框架会在合适的时机下会调用这两个方法。

前面介绍活动资源的时候已经提到过，EngineResource.ResourceListener 唯一实现类为 Engine，来看它的实现内容：

```java
@Override
public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
  //从 ActiveResource 中删除
  activeResources.deactivate(cacheKey);
  if (resource.isMemoryCacheable()) {
    //如果开启了缓存则放入 MemoryCache 
    cache.put(cacheKey, resource);
  } else {
    resourceRecycler.recycle(resource, /*forceNextFrame=*/ false);
  }
}
```

首先会将缓存图片从活动资源中移除，然后再将它存入 MemoryCache 中。回到刚刚分析的 Engine.load 方法中，再看一次获取 ActiveResource 和 MemoryCache 的方法：

```java
private EngineResource<?> loadFromActiveResources(Key key) {
  EngineResource<?> active = activeResources.get(key);
  if (active != null) {
    active.acquire();
  }
  return active;
}

private EngineResource<?> loadFromCache(Key key) {
  //从 MemoryCache 中移除
  EngineResource<?> cached = getEngineResourceFromCache(key);
  if (cached != null) {
    cached.acquire();
    //放入 ActiveResource 中
    activeResources.activate(key, cached);
  }
  return cached;
}
```

除了从根据 Key 获取到 EngineResource 之外，中间还调用了 active.acquire() 方法。只要命中了缓存，那么该资源的引用计数就会加一。而且，如果命中的是 MemoryCache，那么此资源会被移动到 ActiveResource 中。

结合刚才的分析过程可知，EngineResource 在 ActiveResource 和 MemoryCache 中是互斥存在的。正在使用中的图片使用弱引用（ActiveResource）来进行缓存，不在使用中和被 GC 回收的图片使用 LruCahce（MemoryCache） 来进行缓存。

### 磁盘缓存

#### DecoceJob

回到 Engine.load() 方法中，如果是首次加载资源，那么 ActiveResouce 和 MemoryCache 中必然没有缓存下来，接着会走 waitForExistingOrStartNewJob() 方法：

```java
private <R> LoadStatus waitForExistingOrStartNewJob(...) {
  ...
  EngineJob<R> engineJob =
      engineJobFactory.build(
          key,
          isMemoryCacheable,
          useUnlimitedSourceExecutorPool,
          useAnimationPool,
          onlyRetrieveFromCache);

  DecodeJob<R> decodeJob =
      decodeJobFactory.build(...);

  jobs.put(key, engineJob);

  engineJob.addCallback(cb, callbackExecutor);
  engineJob.start(decodeJob);

  if (VERBOSE_IS_LOGGABLE) {
    logWithTimeAndKey("Started new load", startTime, key);
  }
  return new LoadStatus(cb, engineJob);
}
```

DecoceJob 实现了 Runnable 接口，然后会被 EngineJob.start() 方法提交到对应的线程池中去执行，查看 DecoceJob.run() 方法的调用栈：

```java
@Override
public void run() {
  ...
  try {
    ...
.    
    runWrapped();
  } catch (Throwable t) {
    ...
.    
  } finally {
    ...
  }
}

private void runWrapped() {
  switch (runReason) {
    case INITIALIZE:
      stage = getNextStage(Stage.INITIALIZE);
      currentGenerator = getNextGenerator();
      runGenerators();
      break;
    case SWITCH_TO_SOURCE_SERVICE:
      runGenerators();
      break;
    case DECODE_DATA:
      decodeFromRetrievedData();
      break;
    default:
      throw new IllegalStateException("Unrecognized run reason: " + runReason);
  }
}

private Stage getNextStage(Stage current) {
  switch (current) {
    case INITIALIZE:
      return diskCacheStrategy.decodeCachedResource()
          ? Stage.RESOURCE_CACHE
          : getNextStage(Stage.RESOURCE_CACHE);
    case RESOURCE_CACHE:
      return diskCacheStrategy.decodeCachedData()
          ? Stage.DATA_CACHE
          : getNextStage(Stage.DATA_CACHE);
    case DATA_CACHE:
      // Skip loading from source if the user opted to only retrieve the resource from cache.
      return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
    case SOURCE:
    case FINISHED:
      return Stage.FINISHED;
    default:
      throw new IllegalArgumentException("Unrecognized stage: " + current);
  }
}

private DataFetcherGenerator getNextGenerator() {
  switch (stage) {
    case RESOURCE_CACHE:
      return new ResourceCacheGenerator(decodeHelper, this);
    case DATA_CACHE:
      return new DataCacheGenerator(decodeHelper, this);
    case SOURCE:
      return new SourceGenerator(decodeHelper, this);
    case FINISHED:
      return null;
    default:
      throw new IllegalStateException("Unrecognized stage: " + stage);
  }
}

private void runGenerators() {
  currentThread = Thread.currentThread();
  startFetchTime = LogTime.getLogTime();
  boolean isStarted = false;
  while (!isCancelled
      && currentGenerator != null
      && !(isStarted = currentGenerator.startNext())) {
    stage = getNextStage(stage);
    currentGenerator = getNextGenerator();
    if (stage == Stage.SOURCE) {
      reschedule();
      return;
    }
  }
  // We've run out of stages and generators, give up.
  if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
    notifyFailed();
  }
  // Otherwise a generator started a new load and we expect to be called back in
  // onDataFetcherReady.
}
```

DecoceJob.run() 中主要调用了 runWrapped() 方法。在此方法中，首次加载时 runReason 为初始化即 RunReason.INITIALIZE 状态 ，又 diskCacheStrategy 默认为 DiskCacheStrategy.AUTOMATIC，且没有设置过 onlyRetrieveFromCache(true)。所以， decode data 的状态依次为 INITIALIZE -> RESOURCE_CACHE -> DATA_CACHE -> SOURCE -> FINISHED。对应的 DataFectcherGenerator 的 list 依次为 ResourceCacheGenerator -> DataCacheGenerator -> SourceGenerator。

在 runGenerators() 方法中会依次调用各个状态生成的 DataFetcherGenerator 的 startNext() 方法尝试 fetch 数据，直到有某个状态的 DataFetcherGenerator.startNext() 方法可以胜任。若状态抵达到了 Stage.FINISHED 或 Job 被取消，且所有状态的 DataFetcherGenerator.startNext() 都无法满足条件，则调用 SingleRequest.onLoadFailed() 进行错误处理。

这里总共有三个 DataFetcherGenerator，依次是：

1. <strong>ResourceCacheGenerator</strong>：获取采样后、transformed 后资源文件的缓存文件
2. <strong>DataCacheGenerator</strong>：获取原始的没有修改过的资源文件的缓存文件
3. <strong>SourceGenerator</strong>：获取原始源数据，即从网络下载或者从本地资源中加载

#### ResourceCacheKey 和 DataCacheKey

看以看到，ResourceCacheGenerator 和 DataCacheGenerator 分别对应 Resource 和 Data 类型的缓存数据，都属于磁盘缓存的一部分，那么这两种数据的存取规则有什么区别呢？

先看看 ResourceCacheGenerator 中查找缓存时 key 的组成部分：

```java
currentKey =
    new ResourceCacheKey(
        helper.getArrayPool(),
        sourceId,
        helper.getSignature(),
        helper.getWidth(),
        helper.getHeight(),
        transformation,
        resourceClass,
        helper.getOptions());
```

它的各个参数含义如下：

| 参数                                     | 含义                                                                                                                                             |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| helper.getArrayPool()                    | GlideBuilder.build 时初始化，默认为 LruArrayPool；但不参与 key 的 equals 方法                                                                    |
| sourceId                                 | 如果请求的是 URL，那么此处会是一个 GlideUrl                                                                                                      |
| helper.getSignature()                    | BaseRequestOptions 的成员变量，默认会是 EmptySignature.obtain()<br/>在加载本地 resource 资源时会变成 ApplicationVersionSignature.obtain(context) |
| helper.getWidth()<br/>helper.getHeight() | 如果没有指定 override(int size)，那么将得到 view 的 size                                                                                         |
| transformation                           | 默认会根据 ImageView 的 scaleType 设置对应的 BitmapTransformation；<br/>如果指定了 transform，那就是指定的值                                     |
| resourceClass                            | 可以被编码成的资源类型，比如 BitmapDrawable 等                                                                                                   |
| helper.getOptions()                      | 如果没有设置过 transform，此处会根据 ImageView 的 scaleType 默认指定一个 KV                                                                      |

在 ResourceCacheKey 中，arrayPool 并没有参与 equals 方法，所以判断两个 ResourceCacheKey 是否相等只需要其他 7 个参数。

然后看看 DataCacheGenerator 中 Key 的组成部分：

Key originalKey = new DataCacheKey(sourceId, helper.getSignature());

它的各个参数含义如下：

| 参数                  | 含义                                                                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| sourceId              | 如果请求的是 URL，那么此处会是一个 GlideUrl                                                                                                      |
| helper.getSignature() | BaseRequestOptions 的成员变量，默认会是 EmptySignature.obtain()<br/>在加载本地 resource 资源时会变成 ApplicationVersionSignature.obtain(context) |

在 DataCacheKey 中，仅有的两个变量都参与了 equals 方法。这两个变量就是上面 ResourceCacheKey 中的两个变量。

显然，Data Cache 就是比较原始的数据构成的缓存，而 Resource Cache 是被解码、转换后的 Data Cache，与前面的分析呼应起来了。

#### ResourceCacheGenerator

先来看 ResourceCacheGenerator 的 startNext() 方法：

```java
@Override
public boolean startNext() {
 ...
 while (modelLoaders == null || !hasNextModelLoader()) {
   ...
   currentKey =
       new ResourceCacheKey(// NOPMD AvoidInstantiatingObjectsInLoops
           helper.getArrayPool(),
           sourceId,
           helper.getSignature(),
           helper.getWidth(),
           helper.getHeight(),
           transformation,
           resourceClass,
           helper.getOptions());
   //根据 ResourceCacheKey 获取缓存
   cacheFile = helper.getDiskCache().get(currentKey);
   //如果找到了缓存文件，那么循环条件则会为 false，也就退出循环了
   if (cacheFile != null) {
     sourceKey = sourceId;
     modelLoaders = helper.getModelLoaders(cacheFile);
     modelLoaderIndex = 0;
   }
 }
 // 如果找到了，hasNextModelLoader() 方法则会为 true，可以执行循环
 // 没有找到缓存文件，则不会进入循环，会直接返回 false
 loadData = null;
 boolean started = false;
 while (!started && hasNextModelLoader()) {
   ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
   // 在循环中会依次判断某个 ModelLoader 能不能加载此文件
   loadData = modelLoader.buildLoadData(cacheFile,
       helper.getWidth(), helper.getHeight(), helper.getOptions());
   if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
     started = true;
     // 如果某个 ModelLoader 可以，那么就调用其 fetcher 进行加载数据
     // 加载成功或失败会通知自身
     loadData.fetcher.loadData(helper.getPriority(), this);
   }
 }
 return started;
}
```

获取缓存的那行代码 helper.getDiskCache().get(currentKey) 将会调用到 DiskLruCacheWrapper.get 方法：

```java
@Override
public File get(Key key) {
  //生成一个 String 类型的 SafeKey
  String safeKey = safeKeyGenerator.getSafeKey(key);
  if (Log.isLoggable(TAG, Log.VERBOSE)) {
    Log.v(TAG, "Get: Obtained: " + safeKey + " for for Key: " + key);
  }
  File result = null;
  try {
    //真正去获取磁盘文件缓存
    final DiskLruCache.Value value = getDiskCache().get(safeKey);
    if (value != null) {
      result = value.getFile(0);
    }
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.WARN)) {
      Log.w(TAG, "Unable to get from disk cache", e);
    }
  }
  return result;
}
```

首先会使用 Key 映射成一个 String 类型的 safeKey，接着根据这个 safeKey 从 DiskCache 中获取缓存。

回到 ResourceCacheGenerator 中，如果确实有缓存，那么会加载该缓存文件。对于 URL 来说，调用了 ByteBufferFetcher 进行缓存文件的加载，加载成功了返回了一个 ByteBuffer，并调用了 callback 也 就是 ResourceCacheGenerator 的 onDataReady 方法。

```java
@Override
public void onDataReady(Object data) {
  cb.onDataFetcherReady(sourceKey, data, loadData.fetcher, DataSource.RESOURCE_DISK_CACHE,
      currentKey);
}
```

然后 ResourceCacheGenerator 又会回调 DecodeJob 的 onDataFetcherReady 方法进行后续的解码操作。

如果没有在 ResourceCacheGenerator 中被 fetch，则会回调 onLoadFailed() 方法：

```java
@Override
public void onLoadFailed(@NonNull Exception e) {
  cb.onDataFetcherFailed(currentKey, e, loadData.fetcher, DataSource.RESOURCE_DISK_CACHE);
}
```

后面 DataCacheGenerator 和 SourceGenerator 的成功回调都会到这里来。

接着回调到 DecodeJob 中继续执行 runGenerators() 方法：

```java
@Override
public void onDataFetcherFailed(
    Key attemptedKey, Exception e, DataFetcher<?> fetcher, DataSource dataSource) {
  fetcher.cleanup();
  GlideException exception = new GlideException("Fetching data failed", e);
  exception.setLoggingDetails(attemptedKey, dataSource, fetcher.getDataClass());
  throwables.add(exception);
  if (Thread.currentThread() != currentThread) {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
  } else {
    runGenerators();
  }
}
```

#### DataCacheGenerator

DecodeJob.runGenerators 方法中，继续执行 while 循环将会调用 DataCacheGenerator 的 startNext() 方法：

```java
@Override
public boolean startNext() {
  while (modelLoaders == null || !hasNextModelLoader()) {
    sourceIdIndex++;
    if (sourceIdIndex >= cacheKeys.size()) {
      return false;
    }
    Key sourceId = cacheKeys.get(sourceIdIndex);
    // PMD.AvoidInstantiatingObjectsInLoops The loop iterates a limited number of times
    // and the actions it performs are much more expensive than a single allocation.
    @SuppressWarnings("PMD.AvoidInstantiatingObjectsInLoops")
    Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
    cacheFile = helper.getDiskCache().get(originalKey);
    if (cacheFile != null) {
      this.sourceKey = sourceId;
      modelLoaders = helper.getModelLoaders(cacheFile);
      modelLoaderIndex = 0;
    }
  }
  loadData = null;
  boolean started = false;
  while (!started && hasNextModelLoader()) {
    ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
    loadData =
        modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(),
            helper.getOptions());
    if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
      started = true;
      loadData.fetcher.loadData(helper.getPriority(), this);
    }
  }
  return started;
}

private boolean hasNextModelLoader() {
  return modelLoaderIndex < modelLoaders.size();
}
```

过程和 ResourceCacheGenerator 相似，由于获取的是原始的源数据，所以这里的 key 的组成非常简单。

#### SourceGenerator

由于是第一次加载，本地缓存文件肯定是没有的，接着会调用 SourceGenerator 的 startNext() 方法从网络或者本地资源中加载。

```java
private int loadDataListIndex;

@Override
public boolean startNext() {
  // 首次运行 dataToCache 为 null
  if (dataToCache != null) {
    Object data = dataToCache;
    dataToCache = null;
    cacheData(data);
  }
  // 首次运行 sourceCacheGenerator 为 null
  if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
    return true;
  }
  sourceCacheGenerator = null;
  // 准备加载数据
  loadData = null;
  boolean started = false;
  // 这里直接调用了 DecodeHelper.getLoadData() 方法
  // 该方法在前面在 ResourceCacheGenerator 中被调用过，且被缓存了下来
  while (!started && hasNextModelLoader()) {
    loadData = helper.getLoadData().get(loadDataListIndex++);
    if (loadData != null
        && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
        || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
      started = true;
      //调用 MultiFetcher.loadData，然后用 this 接收回调
      loadData.fetcher.loadData(helper.getPriority(), this);
    }
  }
  return started;
}

private boolean hasNextModelLoader() {
  return loadDataListIndex < helper.getLoadData().size();
}
```

helper.getLoadData() 的值在 ResourceCacheGenerator 中就已经被获取并缓存下来了，这是一个 MultiModelLoader 对象生成的 LoadData 对象，LoadData 对象里面有两个 fetcher。

在上面的方法中，会遍历 LoadData list，找出符合条件的 LoadData，然后调用 loadData.fetcher.loadData 加载数据。

在 loadData 不为空的前提下，会判断 Glide 的缓存策略是否可以缓存此数据源，或者是否有加载路径。默认情况下 Glide 的缓存策略是 DiskCacheStrategy.AUTOMATIC，它的 isDataCacheable 实现如下：

```java
@Override
public boolean isDataCacheable(DataSource dataSource) {
  //DataSource.REMOTE 代表网络资源
  return dataSource == DataSource.REMOTE;
}
```

再看下 loadData.fetcher.getDataSource()：

```java
static class MultiFetcher<Data> implements DataFetcher<Data>, DataCallback<Data> {
  @NonNull
  @Override
  public DataSource getDataSource() {
    //fetchers 数组保存的两个 DataFetcher 都是 HttpUrlFetcher
    return fetchers.get(0).getDataSource();
  }
}

public class HttpUrlFetcher implements DataFetcher<InputStream> {
  @NonNull
  @Override
  public DataSource getDataSource() {
    return DataSource.REMOTE;
  }
}
```

显然，Glide 的缓存策略是可以缓存此数据源的，所以会进行数据加载。接着看看 MultiFetcher.loadData 方法：

```java
static class MultiFetcher<Data> implements DataFetcher<Data>, DataCallback<Data> {
  private final List<DataFetcher<Data>> fetchers;
  private int currentIndex;
  private Priority priority;
  private DataCallback<? super Data> callback;
  
  @Override
  public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super Data> callback) {
    this.priority = priority;
    this.callback = callback;
    exceptions = throwableListPool.acquire();
    fetchers.get(currentIndex).loadData(priority, this);
    // If a race occurred where we cancelled the fetcher in cancel() and then called loadData here
    // immediately after, make sure that we cancel the newly started fetcher. We don't bother
    // checking cancelled before loadData because it's not required for correctness and would
    // require an unlikely race to be useful.
    if (isCancelled) {
      cancel();
    }
  }
  
  @Override
  public void onDataReady(@Nullable Data data) {
    if (data != null) {
      callback.onDataReady(data);
    } else {
      startNextOrFail();
    }
  }
  
  @Override
  public void onLoadFailed(@NonNull Exception e) {
    Preconditions.checkNotNull(exceptions).add(e);
    startNextOrFail();
  }
  
  private void startNextOrFail() {
    if (isCancelled) {
      return;
    }
    if (currentIndex < fetchers.size() - 1) {
      currentIndex++;
      loadData(priority, callback);
    } else {
      Preconditions.checkNotNull(exceptions);
      callback.onLoadFailed(new GlideException("Fetch failed", new ArrayList<>(exceptions)));
    }
  }
}
```

这里面两个 DataFetcher 都是参数相同的 HttpUrlFetcher 实例，我们直接看里面如何从网络加载图片的。

```java
@Override
public void loadData(@NonNull Priority priority,
    @NonNull DataCallback<? super InputStream> callback) {
  long startTime = LogTime.getLogTime();
  try {
    //获取 InputStream 
    InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
    //将结果回调给 MultiFetcher
    callback.onDataReady(result);
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "Failed to load data for url", e);
    }
    callback.onLoadFailed(e);
  } finally {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
    }
  }
}
```

这里将请求操作放到了 loadDataWithRedirects() 方法中，然后将请求结果通过回调返回上一层也就是 MultiFetcher.onDataReady() 中。

```java
private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
    Map<String, String> headers) throws IOException {
  // 检查重定向次数
  if (redirects >= MAXIMUM_REDIRECTS) {
    throw new HttpException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
  } else {
    // Comparing the URLs using .equals performs additional network I/O and is generally broken.
    // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
    try {
      // 检查是不是重定向到自身了
      if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
        throw new HttpException("In re-direct loop");
      }
    } catch (URISyntaxException e) {
      // Do nothing, this is best effort.
    }
  }
  // connectionFactory默认是DefaultHttpUrlConnectionFactory
  // 其build方法就是调用了url.openConnection()
  urlConnection = connectionFactory.build(url);
  for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
    urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
  }
  urlConnection.setConnectTimeout(timeout);
  urlConnection.setReadTimeout(timeout);
  urlConnection.setUseCaches(false);
  urlConnection.setDoInput(true);
  // Stop the urlConnection instance of HttpUrlConnection from following redirects so that
  // redirects will be handled by recursive calls to this method, loadDataWithRedirects.
  // 禁止HttpUrlConnection自动重定向，重定向功能由本方法自己实现
  urlConnection.setInstanceFollowRedirects(false);
  // Connect explicitly to avoid errors in decoders if connection fails.
  urlConnection.connect();
  // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
  stream = urlConnection.getInputStream();
  if (isCancelled) {
    return null;
  }
  final int statusCode = urlConnection.getResponseCode();
  if (isHttpOk(statusCode)) {
    // statusCode=2xx，请求成功
    return getStreamForSuccessfulRequest(urlConnection);
  } else if (isHttpRedirect(statusCode)) {
    // statusCode=3xx，需要重定向
    String redirectUrlString = urlConnection.getHeaderField("Location");
    if (TextUtils.isEmpty(redirectUrlString)) {
      throw new HttpException("Received empty or null redirect url");
    }
    URL redirectUrl = new URL(url, redirectUrlString);
    // Closing the stream specifically is required to avoid leaking ResponseBodys in addition
    // to disconnecting the url connection below. See #2352.
    cleanup();
    return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
  } else if (statusCode == INVALID_STATUS_CODE) {
    // -1 表示不是HTTP响应
    throw new HttpException(statusCode);
  } else {
    // 其他HTTP错误
    throw new HttpException(urlConnection.getResponseMessage(), statusCode);
  }
}
```

到这一步已经获得网络图片的 InputStream 了，该资源会通过回调经过 MultiFetcher 然后再次回调到 SourceGenerator 中。

```java
@Override
public void onDataReady(Object data) {
  DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
  if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
    dataToCache = data;
    // We might be being called back on someone else's thread. Before doing anything, we should
    // reschedule to get back onto Glide's thread.
    cb.reschedule();
  } else {
    cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
        loadData.fetcher.getDataSource(), originalKey);
  }
}

@Override
public void onLoadFailed(@NonNull Exception e) {
  cb.onDataFetcherFailed(originalKey, e, loadData.fetcher, loadData.fetcher.getDataSource());
}
```

onLoadFailed() 方法很简单，直接回调 DecodeJob.onDataFetcherFailed 方法。onDataReady() 方法会首先判 data 能不能缓存，若能缓存则缓存起来，然后调用 DataCacheGenerator 进行加载缓存；若不能缓存，则直接回调 DecodeJob.onDataFetcherReady() 方法通知外界 data 已经准备好了。

获取 DiskCacheStrategy 并判断能不能被缓存的逻辑在 SourceGenerator.startNext() 中出现过，之类显然是可以的。然后将 data 保存到 dataToCache，并调用 cb.reschedule()。

cb.reschedule() 的作用就是将 DecodeJob 提交到 Glide 的 source 线程池中。然后执行 DecodeJob.run() 方法，经过 runWrapped()、 runGenerators() 方法后，又回到了 SourceGenerator.startNext() 方法。

```java
@Override
public boolean startNext() {
  //首先判断是否有数据待缓存
  if (dataToCache != null) {
    Object data = dataToCache;
    dataToCache = null;
    cacheData(data);
  }
  
  //刚刚构建了一个 DataCacheGenerator，这里不为空
  //调用 startNext() 后返回 true
  if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
    return true;
  }
  ...
}

private void cacheData(Object dataToCache) {
  long startTime = LogTime.getLogTime();
  try {
    Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
    DataCacheWriter<Object> writer =
        new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
    originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
    //缓存 data
    helper.getDiskCache().put(originalKey, writer);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Finished encoding source to cache"
          + ", key: " + originalKey
          + ", data: " + dataToCache
          + ", encoder: " + encoder
          + ", duration: " + LogTime.getElapsedMillis(startTime));
    }
  } finally {
    loadData.fetcher.cleanup();
  }
  //构建一个 DataCacheGenerator，传入 this 作为回调
  sourceCacheGenerator =
      new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
}
```

在 startNext() 方法的开头，会判断 dataToCache 是否为空，此时显然不为空，所以会调用 cacheData(Object) 方法进行 data 的缓存处理。缓存完毕后，会为该缓存文件生成一个 SourceCacheGenerator。然后在 startNext() 方法中会直接调用该变量进行加载。

由于在构造 DataCacheGenerator 时，指定了 FetcherReadyCallback 为 this，所以这时 DataCacheGenerator 加载结果会由 SourceGenerator 转发给 DecodeJob。

#### DecodeJob.FetcherReadyCallback

先看一下 fetch 失败时干了什么，DecodeJob.onDataFetcherFailed() 方法的实现代码如下：

```java
@Override
public void onDataFetcherFailed(Key attemptedKey, Exception e, DataFetcher<?> fetcher,
    DataSource dataSource) {
  fetcher.cleanup();
  GlideException exception = new GlideException("Fetching data failed", e);
  exception.setLoggingDetails(attemptedKey, dataSource, fetcher.getDataClass());
  throwables.add(exception);
  if (Thread.currentThread() != currentThread) {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
  } else {
    runGenerators();
  }
}
```

显然，如果 fetch 失败了，如果不在 source 线程中就会回调 EnginJob 的 reschedule() 方法切换到 source 线程，然后重新调用 runGenerators() 方法尝试使用下一个 DataFetcherGenerator 进行加载，一直到没有一个可以加载，这时会调用 notifyFailed() 方法，正式宣告加载失败。

然后再看成功的时候回调的 onDataFetcherReady() 方法：

```java
@Override
public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
    DataSource dataSource, Key attemptedKey) {
  this.currentSourceKey = sourceKey;
  this.currentData = data;
  this.currentFetcher = fetcher;
  this.currentDataSource = dataSource;
  this.currentAttemptingKey = attemptedKey;
  //判断当前线程是否为 currentThread
  if (Thread.currentThread() != currentThread) 
    runReason = RunReason.DECODE_DATA;
    //如果不是，则让 EngineJob 重新调度，最后也会调用 decodeFromRetrievedData
    callback.reschedule(this);
  } else {
    GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
    try {
      decodeFromRetrievedData();
    } finally {
      GlideTrace.endSection();
    }
  }
}
```

首先会保存传入的参数，然后确认执行线程后调用 decodeFromRetrievedData() 方法进行解码：

```java
private void decodeFromRetrievedData() {
  if (Log.isLoggable(TAG, Log.VERBOSE)) {
    logWithTimeAndKey("Retrieved data", startFetchTime,
        "data: " + currentData
            + ", cache key: " + currentSourceKey
            + ", fetcher: " + currentFetcher);
  }
  Resource<R> resource = null;
  try {
    resource = decodeFromData(currentFetcher, currentData, currentDataSource);
  } catch (GlideException e) {
    e.setLoggingDetails(currentAttemptingKey, currentDataSource);
    throwables.add(e);
  }
  if (resource != null) {
    notifyEncodeAndRelease(resource, currentDataSource);
  } else {
    runGenerators();
  }
}
```

先调用 decodeFromData() 方法进行解码，然后调用 notifyEncodeAndRelease() 方法进行缓存，同时也会通知 EngineJob 资源已经准备好了。

decodeFromData() 方法调用栈如下：

```java
private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data,
    DataSource dataSource) throws GlideException {
  try {
    if (data == null) {
      return null;
    }
    long startTime = LogTime.getLogTime();
    Resource<R> result = decodeFromFetcher(data, dataSource);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Decoded result " + result, startTime);
    }
    return result;
  } finally {
    fetcher.cleanup();
  }
}

@SuppressWarnings("unchecked")
private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
    throws GlideException {
  LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
  return runLoadPath(data, dataSource, path);
}

private <Data, ResourceType> Resource<R> runLoadPath(Data data, DataSource dataSource,
    LoadPath<Data, ResourceType, R> path) throws GlideException {
  Options options = getOptionsWithHardwareConfig(dataSource);
  DataRewinder<Data> rewinder = glideContext.getRegistry().getRewinder(data);
  try {
    // ResourceType in DecodeCallback below is required for compilation to work with gradle.
    return path.load(
        rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
  } finally {
    rewinder.cleanup();
  }
}
```

`path.load(..., new DecodeCallback<ResourceType>(dataSource))` 这行代码中，最后传入了一个 DecodeCallback 回调，该类的回调方法会回调给 DecodeJob 对应的方法：

```java
private final class DecodeCallback<Z> implements DecodePath.DecodeCallback<Z> {
  private final DataSource dataSource;
  @Synthetic
  DecodeCallback(DataSource dataSource) {
    this.dataSource = dataSource;
  }
  
  @NonNull
  @Override
  public Resource<Z> onResourceDecoded(@NonNull Resource<Z> decoded) {
    return DecodeJob.this.onResourceDecoded(dataSource, decoded);
  }
}
```

然后来看一下 LoadPath.load() 方法的实现：

```java
public Resource<Transcode> load(DataRewinder<Data> rewinder, @NonNull Options options, int width,
    int height, DecodePath.DecodeCallback<ResourceType> decodeCallback) throws GlideException {
  List<Throwable> throwables = Preconditions.checkNotNull(listPool.acquire());
  try {
    return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
  } finally {
    listPool.release(throwables);
  }
}

private Resource<Transcode> loadWithExceptionList(DataRewinder<Data> rewinder,
    @NonNull Options options,
    int width, int height, DecodePath.DecodeCallback<ResourceType> decodeCallback,
    List<Throwable> exceptions) throws GlideException {
  Resource<Transcode> result = null;
  //noinspection ForLoopReplaceableByForEach to improve perf
  for (int i = 0, size = decodePaths.size(); i < size; i++) {
    DecodePath<Data, ResourceType, Transcode> path = decodePaths.get(i);
    try {
      result = path.decode(rewinder, width, height, options, decodeCallback);
    } catch (GlideException e) {
      exceptions.add(e);
    }
    if (result != null) {
      break;
    }
  }

  if (result == null) {
    throw new GlideException(failureMessage, new ArrayList<>(exceptions));
  }

  return result;
}
```

对于每条 DecodePath，都调用其 decode() 方法，直到有一个 DecodePath 可以 decode 出资源。继续看看 DecodePath.decode() 方法：

```java
public Resource<Transcode> decode(DataRewinder<DataType> rewinder, int width, int height,
    @NonNull Options options, DecodeCallback<ResourceType> callback) throws GlideException {
  Resource<ResourceType> decoded = decodeResource(rewinder, width, height, options);
  Resource<ResourceType> transformed = callback.onResourceDecoded(decoded);
  return transcoder.transcode(transformed, options);
}
```

可以看到，这里有三个步骤：

1. 使用 ResourceDecoder List 进行 decode，将原始数据解码成原始图片
2. 将 decoded 的资源进行 transform
3. 将 transformed 的资源进行 transcode

首先是 decodeResource() 的过程：

```java
@NonNull
private Resource<ResourceType> decodeResource(DataRewinder<DataType> rewinder, int width,
    int height, @NonNull Options options) throws GlideException {
  List<Throwable> exceptions = Preconditions.checkNotNull(listPool.acquire());
  try {
    return decodeResourceWithList(rewinder, width, height, options, exceptions);
  } finally {
    listPool.release(exceptions);
  }
}

@NonNull
private Resource<ResourceType> decodeResourceWithList(DataRewinder<DataType> rewinder, int width,
    int height, @NonNull Options options, List<Throwable> exceptions) throws GlideException {
  Resource<ResourceType> result = null;
  //noinspection ForLoopReplaceableByForEach to improve perf
  for (int i = 0, size = decoders.size(); i < size; i++) {
    //decoders 只有一条，就是 ByteBufferBitmapDecoder
    ResourceDecoder<DataType, ResourceType> decoder = decoders.get(i);
    try {
      //rewinder 自然是 ByteBufferRewind
      //data 为 ByteBuffer
      DataType data = rewinder.rewindAndGet();
      //ByteBufferBitmapDecoder 内部会调用 Downsampler 的 hanldes 方法
      // 它对任意的 InputStream 和 ByteBuffer 都返回 true
      if (decoder.handles(data, options)) {
        //调用 ByteBuffer.position(0) 复位
        data = rewinder.rewindAndGet();
        //开始解码
        result = decoder.decode(data, width, height, options);
      }
      // Some decoders throw unexpectedly. If they do, we shouldn't fail the entire load path, but
      // instead log and continue. See #2406 for an example.
    } catch (IOException | RuntimeException | OutOfMemoryError e) {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Failed to decode data for " + decoder, e);
      }
      exceptions.add(e);
    }

    if (result != null) {
      break;
    }
  }

  if (result == null) {
    throw new GlideException(failureMessage, new ArrayList<>(exceptions));
  }
  return result;
}
ByteBufferBitmapDecoder.decode() 方法：
@Override
public Resource<Bitmap> decode(@NonNull ByteBuffer source, int width, int height,
    @NonNull Options options)
    throws IOException {
  InputStream is = ByteBufferUtil.toStream(source);
  return downsampler.decode(is, width, height, options);
}
```

先将 ByteBuffer 转换成 InputStream，然后调用  Downsampler.decode() 方法进行解码。

这里执行完毕，会将 decode 出来的 Bitmap 包装成为一个 BitmapResource 对象。然后就一直往上返回，返回到 DecodePath.decode 方法中。接下来执行第二行：

```java
Resource<ResourceType> transformed = callback.onResourceDecoded(decoded);
```

这里的 callback 在前面提到过，这会调用 `DecodeJob.onResourceDecoded(DataSource, Resource<Z>)` 方法：

```java
@Synthetic
@NonNull
<Z> Resource<Z> onResourceDecoded(DataSource dataSource,
    @NonNull Resource<Z> decoded) {
  @SuppressWarnings("unchecked")
  Class<Z> resourceSubClass = (Class<Z>) decoded.get().getClass();// Bitmap.class
  Transformation<Z> appliedTransformation = null;
  Resource<Z> transformed = decoded;
  //dataSource为DATA_DISK_CACHE，所以满足条件
  if (dataSource != DataSource.RESOURCE_DISK_CACHE) {
    //Bitmap.class对应的正是FitCenter()
    appliedTransformation = decodeHelper.getTransformation(resourceSubClass);
    //对decoded资源进行transform
    transformed = appliedTransformation.transform(glideContext, decoded, width, height);
  }
  //TODO: Make this the responsibility of the Transformation.
  if (!decoded.equals(transformed)) {
    decoded.recycle();
  }
  final EncodeStrategy encodeStrategy;
  final ResourceEncoder<Z> encoder;
  //Bitmap有注册对应的BitmapEncoder，所以是available的
  if (decodeHelper.isResourceEncoderAvailable(transformed)) {
    //encoder就是BitmapEncoder
    encoder = decodeHelper.getResultEncoder(transformed);
    //encodeStrategy为EncodeStrategy.TRANSFORMED
    encodeStrategy = encoder.getEncodeStrategy(options);
  } else {
    encoder = null;
    encodeStrategy = EncodeStrategy.NONE;
  }
  Resource<Z> result = transformed;
  //isSourceKey显然为true，所以isFromAlternateCacheKey为false，所以就返回了
  boolean isFromAlternateCacheKey = !decodeHelper.isSourceKey(currentSourceKey);
  //diskCacheStrategy为AUTOMATIC，该方法返回false
  if (diskCacheStrategy.isResourceCacheable(isFromAlternateCacheKey, dataSource,
      encodeStrategy)) {
    if (encoder == null) {
      throw new Registry.NoResultEncoderAvailableException(transformed.get().getClass());
    }
    final Key key;
    switch (encodeStrategy) {
      case SOURCE:
        key = new DataCacheKey(currentSourceKey, signature);
        break;
      case TRANSFORMED:
        key =
            new ResourceCacheKey(
                decodeHelper.getArrayPool(),
                currentSourceKey,
                signature,
                width,
                height,
                appliedTransformation,
                resourceSubClass,
                options);
        break;
      default:
        throw new IllegalArgumentException("Unknown strategy: " + encodeStrategy);
    }
    LockedResource<Z> lockedResult = LockedResource.obtain(transformed);
    deferredEncodeManager.init(key, encoder, lockedResult);
    result = lockedResult;
  }
  return result;
}
```

当 dataSource != DataSource.RESOURCE_DISK_CACHE 时会进行 transform，这是因为 resource cache 肯定已经经历过 transform 了，所以就不用重新来一遍了。

然后就回到 DecodePath.decode 方法的第三行了：

```java
return transcoder.transcode(transformed, options);
```

这里的 transcoder 就是 BitmapDrawableTranscoder，该方法返回了一个 LazyBitmapDrawableResource。

至此，resource 已经 decode 完毕。下面一直返回到 DecodeJob.decodeFromRetrievedData() 方法中。下面会调用 notifyEncodeAndRelease() 方法完成后面的事宜。

```java
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
  // resource是BitmapResource类型，实现了Initializable接口
  if (resource instanceof Initializable) {
    // initialize方法调用了bitmap.prepareToDraw()
    ((Initializable) resource).initialize();
  }
  Resource<R> result = resource;
  LockedResource<R> lockedResource = null;
  if (deferredEncodeManager.hasResourceToEncode()) {
    lockedResource = LockedResource.obtain(resource);
    result = lockedResource;
  }
  // 通知回调，资源已经就绪
  notifyComplete(result, dataSource);
  stage = Stage.ENCODE;
  try {
    // TODO
    if (deferredEncodeManager.hasResourceToEncode()) {
      deferredEncodeManager.encode(diskCacheProvider, options);
    }
  } finally {
    // lockedResource为null, skip
    if (lockedResource != null) {
      lockedResource.unlock();
    }
  }
  // Call onEncodeComplete outside the finally block so that it's not called if the encode process
  // throws.
  // 进行清理工作
  onEncodeComplete();
}
```

重点在于 notifyComplete() 方法，该方法内部会调用 callback.onResourceReady(resource, dataSource) 将结果传递给回调，这里的回调是 EngineJob：

```java
@Override
public void onResourceReady(Resource<R> resource, DataSource dataSource) {
  synchronized (this) {
    this.resource = resource;
    this.dataSource = dataSource;
  }
  notifyCallbacksOfResult();
}

void notifyCallbacksOfResult() {
  ResourceCallbacksAndExecutors copy;
  Key localKey;
  EngineResource<?> localResource;
  synchronized (this) {
    stateVerifier.throwIfRecycled();
    if (isCancelled) {
      // TODO: Seems like we might as well put this in the memory cache instead of just recycling
      // it since we've gotten this far...
      resource.recycle();
      release();
      return;
    } else if (cbs.isEmpty()) {
      throw new IllegalStateException("Received a resource without any callbacks to notify");
    } else if (hasResource) {
      throw new IllegalStateException("Already have resource");
    }
    // engineResourceFactory默认为EngineResourceFactory
    // 其build方法就是new一个对应的资源
    // new EngineResource<>(resource, isMemoryCacheable, /*isRecyclable=*/ true)
    engineResource = engineResourceFactory.build(resource, isCacheable);
    // Hold on to resource for duration of our callbacks below so we don't recycle it in the
    // middle of notifying if it synchronously released by one of the callbacks. Acquire it under
    // a lock here so that any newly added callback that executes before the next locked section
    // below can't recycle the resource before we call the callbacks.
    hasResource = true;
    copy = cbs.copy();
    incrementPendingCallbacks(copy.size() + 1);

    localKey = key;
    localResource = engineResource;
  }

  // listener就是Engine，该方法会将资源保存到activeResources中
  listener.onEngineJobComplete(this, localKey, localResource);

  // 这里的ResourceCallbackAndExecutor就是我创建EngineJob和DecodeJob
  // 并在执行DecodeJob之前添加的回调
  // entry.executor就是Glide.with.load.into中出现的Executors.mainThreadExecutor()
  // entry.cb就是SingleRequest
  for (final ResourceCallbackAndExecutor entry : copy) {
    entry.executor.execute(new CallResourceReady(entry.cb));
  }
  decrementPendingCallbacks();
}
```

listener.onEngineJobComplete() 的代码很简单。

```java
@Override
public synchronized void onEngineJobComplete(
    EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
  // A null resource indicates that the load failed, usually due to an exception.
  if (resource != null) {
    resource.setResourceListener(key, this);

    if (resource.isCacheable()) {
      activeResources.activate(key, resource);
    }
  }

  jobs.removeIfCurrent(key, engineJob);
}

@Override
public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
  activeResources.deactivate(cacheKey);
  if (resource.isCacheable()) {
    cache.put(cacheKey, resource);
  } else {
    resourceRecycler.recycle(resource);
  }
}
```

首先会设置资源的回调为自己，这样在资源释放时会通知自己的回调方法，将资源从 active 状态变为 cache 状态，如 onResourceReleased() 方法；然后将资源放入 activeResources 中，资源变为 active 状态；最后将 engineJob 从 Jobs 中移除。

然后看下 entry.executor.execute(new CallResourceReady(entry.cb)) 方法的实现，关注点在 CallResourceReady 上面：

```java
private class CallResourceReady implements Runnable {

    private final ResourceCallback cb;

    CallResourceReady(ResourceCallback cb) {
      this.cb = cb;
    }

    @Override
    public void run() {
      synchronized (EngineJob.this) {
        if (cbs.contains(cb)) {
          // Acquire for this particular callback.
          engineResource.acquire();
          callCallbackOnResourceReady(cb);
          removeCallback(cb);
        }
        decrementPendingCallbacks();
      }
    }
  }
```

首先调用 callCallbackOnResourceReady(cb) 调用 callback，然后调用 removeCallback(cb) 移除 callback。看看 callCallbackOnResourceReady(cb)：

```java
void callCallbackOnResourceReady(ResourceCallback cb) {
  try {
    // This is overly broad, some Glide code is actually called here, but it's much
    // simpler to encapsulate here than to do so at the actual call point in the
    // Request implementation.
    cb.onResourceReady(engineResource, dataSource, isLoadedFromAlternateCacheKey);
  } catch (Throwable t) {
    throw new CallbackException(t);
  }
}
```

这里就调用了 cb.onResourceReady()，这里说到过 entry.cb 就是 SingleRequest。所以继续看看 SingleRequest.onResourceReady() 方法:

```java
@Override
public synchronized void onResourceReady(Resource<?> resource, DataSource dataSource) {
  ...

  onResourceReady((Resource<R>) resource, (R) received, dataSource);
}

private synchronized void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
  // We must call isFirstReadyResource before setting status.
  // 由于requestCoordinator为null，所以返回true
  boolean isFirstResource = isFirstReadyResource();
  // 将status状态设置为COMPLETE
  status = Status.COMPLETE;
  this.resource = resource;

  if (glideContext.getLogLevel() <= Log.DEBUG) {
    Log.d(GLIDE_TAG, "Finished loading " + result.getClass().getSimpleName() + " from "
        + dataSource + " for " + model + " with size [" + width + "x" + height + "] in "
        + LogTime.getElapsedMillis(startTime) + " ms");
  }

  isCallingCallbacks = true;
  try {
     // 尝试调用各个listener的onResourceReady回调进行处理
    boolean anyListenerHandledUpdatingTarget = false;
    if (requestListeners != null) {
      for (RequestListener<R> listener : requestListeners) {
        anyListenerHandledUpdatingTarget |=
            listener.onResourceReady(result, model, target, dataSource, isFirstResource);
      }
    }
    anyListenerHandledUpdatingTarget |=
        targetListener != null
            && targetListener.onResourceReady(result, model, target, dataSource, isFirstResource);

    // 如果没有一个回调能够处理，那么自己处理
    if (!anyListenerHandledUpdatingTarget) {
      // animationFactory默认为NoTransition.getFactory()，生成的animation为NO_ANIMATION
      Transition<? super R> animation =
          animationFactory.build(dataSource, isFirstResource);
      // target为DrawableImageViewTarget
      target.onResourceReady(result, animation);
    }
  } finally {
    isCallingCallbacks = false;
  }

  // 通知requestCoordinator
  notifyLoadSuccess();
}
```

其处理过程和 onLoadFailed() 方法非常类似。

DrawableImageViewTarget 的基类 ImageViewTarget 实现了此方法：

```java
// ImageViewTarget.java
@Override
public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
  // NO_ANIMATION.transition返回false，所以直接调用setResourceInternal方法
  if (transition == null || !transition.transition(resource, this)) {
    setResourceInternal(resource);
  } else {
    maybeUpdateAnimatable(resource);
  }
}

private void setResourceInternal(@Nullable Z resource) {
  // Order matters here. Set the resource first to make sure that the Drawable has a valid and
  // non-null Callback before starting it.
  // 先设置图片
  setResource(resource);
  // 然后如果是动画，会执行动画
  maybeUpdateAnimatable(resource);
}

private void maybeUpdateAnimatable(@Nullable Z resource) {
  // BitmapDrawable显然不是一个Animatable对象，所以走else分支
  if (resource instanceof Animatable) {
    animatable = (Animatable) resource;
    animatable.start();
  } else {
    animatable = null;
  }
}

// DrawableImageViewTarget
@Override
protected void setResource(@Nullable Drawable resource) {
  view.setImageDrawable(resource);
}
```

OK，至此网络图片已经通过 view.setImageDrawable(resource) 加载完毕。

最后，回到 DecodeJob.notifyEncodeAndRelease() 方法，notifyComplete() 被调用后，会判断 deferredEncodeManager.hasResourceToEncode()，如果为 true 则会调用 deferredEncodeManager.encode() 方法：

```java
private static class DeferredEncodeManager<Z> {
  private Key key;
  private ResourceEncoder<Z> encoder;
  private LockedResource<Z> toEncode;
  
  @Synthetic
  DeferredEncodeManager() { }
  
  // We just need the encoder and resource type to match, which this will enforce.
  @SuppressWarnings("unchecked")
  <X> void init(Key key, ResourceEncoder<X> encoder, LockedResource<X> toEncode) {
    this.key = key;
    this.encoder = (ResourceEncoder<Z>) encoder;
    this.toEncode = (LockedResource<Z>) toEncode;
  }
  
  void encode(DiskCacheProvider diskCacheProvider, Options options) {
    GlideTrace.beginSection("DecodeJob.encode");
    try {
      diskCacheProvider.getDiskCache().put(key,
          new DataCacheWriter<>(encoder, toEncode, options));
    } finally {
      toEncode.unlock();
      GlideTrace.endSection();
    }
  }
  
  boolean hasResourceToEncode() {
    return toEncode != null;
  }
  
  void clear() {
    key = null;
    encoder = null;
    toEncode = null;
  }
}
```

DiskLruCacheWrapper 在写入的时候会使用到写锁 DiskCacheWriteLocker，锁对象由对象池创建，写锁 WriteLock 实现是一个重入锁 ReentrantLock，该锁默认是一个不公平锁。

在缓存写入前，会判断 key 对应的 value 存不存在，若存在则放弃写入。缓存的真正写入会由 DataCacheWriter 交给 ByteBufferEncoder 和 StreamEncoder 两个具体类来写入，前者负责将 ByteBuffe 写入到文件，后者负责将 InputStream 写入到文件。

至此，磁盘缓存的读写都已经完毕，剩下的就是内存缓存的两个层次了。回到 DecodeJob.notifyEncodeAndRelease 方法中，经过 notifyComplete、EngineJob.onResourceReady、notifyCallbacksOfResult 方法中。

在该方法中一方面会将原始的 resource 包装成一个 EngineResource，然后通过回调传给 Engine.onEngineJobComplete()，在这里会将资源保持在 ActiveResource 中：

```java
@Override
public synchronized void onEngineJobComplete(
    EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
  // A null resource indicates that the load failed, usually due to an exception.
  if (resource != null) {
    resource.setResourceListener(key, this);
    if (resource.isCacheable()) {
      activeResources.activate(key, resource);
    }
  }
  jobs.removeIfCurrent(key, engineJob);
}
```

另一方面会使用 Executors.mainThreadExecutor() 调用 SingleRequest.onResourceReady() 回调进行资源的显示。

在触发回调前后各有一个地方会对 engineResource 进行 acquire() 和 release() 操作。该过程发生在 notifyCallbacksOfResult() 方法的 incrementPendingCallbacks()、decrementPendingCallbacks() 调用中。这一顿操作下来，engineResource 的引用计数应该是 1：

```java
@Synthetic
void notifyCallbacksOfResult() {
  ResourceCallbacksAndExecutors copy;
  Key localKey;
  EngineResource<?> localResource;
  
  synchronized (this) {
    ...
    engineResource = engineResourceFactory.build(resource, isCacheable);
    // Hold on to resource for duration of our callbacks below so we don't recycle it in the
    // middle of notifying if it synchronously released by one of the callbacks. Acquire it under
    // a lock here so that any newly added callback that executes before the next locked section
    // below can't recycle the resource before we call the callbacks.
    hasResource = true;
    copy = cbs.copy();
    incrementPendingCallbacks(copy.size() + 1);
    localKey = key;
    localResource = engineResource;
  }
  listener.onEngineJobComplete(this, localKey, localResource);
  for (final ResourceCallbackAndExecutor entry : copy) {
    entry.executor.execute(new CallResourceReady(entry.cb));
  }
  decrementPendingCallbacks();
}

synchronized void incrementPendingCallbacks(int count) {
  ...
  if (pendingCallbacks.getAndAdd(count) == 0 && engineResource != null) {
    engineResource.acquire();
  }
}

synchronized void decrementPendingCallbacks() {
  ...
  int decremented = pendingCallbacks.decrementAndGet();
  if (decremented == 0) {
    if (engineResource != null) {
      engineResource.release();
    }
    release();
  }
}
```

engineResource 的引用计数会在 RequestManager.onDestory() 方法中经过 clear()、untrackOrDelegate()、untrack()、RequestTracker.clearRemoveAndRecycle()、clearRemoveAndMaybeRecycle() 方法，调用 SingleRequest.clear() 方法经过 releaseResource()、Engine.release()，进行释放。这样引用计数就为 0 了。

前面在看 EngineResource 的代码时我们知道，一旦引用计数为 0，就会通知 Engine 将此资源从 active 状态变成 memory cache 状态。如果我们再次加载资源时可以从 memory cache 中加载，那么资源又会从 memory cache 状态变成 active 状态。

也就是说，在资源第一次显示后，我们关闭页面，资源会由 active 变成 memory cache；然后我们再次进入页面，加载时会命中 memory cache，从而又变成 active 状态。

本章 Glide 缓存机制的源码内容到此为止了，现在看看文章最开始的流程图，是不是有一点点熟悉的味道。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034514.png)

DecodeJob.onDataFetcherReady 该方法完成两个事情：

1. 将比较原始的 data 数据转变为可以供 ImageView 显示的 resource 数据
2. 将 resource 数据显示出来

在过程 1 中，将原始 data encode 成 resource 数据后，会调用 DecodeJob.onResourceDecoded 对 resource 数据进行进一步的处理。DecodeJob.onResourceDecoded 首先会对 resource 进行 transform，然后可能会进行磁盘缓存。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-2/clipboard_20230323_034509.png)
# 面试题

内存缓存分两个原因：

1、活动缓存主要是当前正在使用的图片资源。图片资源存放在弱引用中。下面是只用活动缓存的缺点

场景：上下滑动列表，图片会不断创建，当 GC 来临时候，会将当前不使用的图片资源进行回收，由于存放在弱引用中。这样就会导致下次滑动回来还需要加载到内存中。

2、Lru 缓存主要存放：最近被加载过的资源。下面是只用 Lru 的缺点

场景：滑动列表头部头像不动，列表不断滑动，由于 Lru 的机制缓存到达一定大小，可能会将头部图片资源回收。

3、配合使用，两者互斥存在。当资源正在使用的时候放在活动缓存中，当不使用的时候放在 Lru 中（引用为 0 的时候）。so,两者结合性能最佳

磁盘缓存分两个原因：

1、资源缓存主要是变换后的资源

2、原始缓存是原始的资源文件

场景：同一张图片，先在 100*100 的 view 上显示，然后在 200*200 的 view 上显示。

分析：如果不缓存变换后的图，那么每次都需要变换。如果不缓存原图，那么每次都需要重新网络下载原图
