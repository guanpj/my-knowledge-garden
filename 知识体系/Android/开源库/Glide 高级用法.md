---
title: Glide 高级用法
tags:
 - Glide
 - 源码解析
date created: 2023-03-23
date modified: 2023-03-24
---

# 回调与监听

## Target

我们都知道，使用 Glide 在界面上加载并展示一张图片只需要一行代码：

```java
Glide.with(this).load(url).into(imageView);
```

将 ImageView 的实例传入到 into() 方法当中，Glide 将图片加载完成之后，图片就能显示到 ImageView 上了。这是怎么实现的呢？来看一下 into() 方法的源码：

```java
@NonNull
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread();
  Preconditions.checkNotNull(view);

  BaseRequestOptions<?> requestOptions = this;
  if (!requestOptions.isTransformationSet()
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    // Clone in this method so that if we use this RequestBuilder to load into a View and then
    // into a different target, we don't retain the transformation applied based on the previous
    // View's scale type.
    switch (view.getScaleType()) {
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER:
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

  return into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions,
      Executors.mainThreadExecutor());
}
```

可以看到，最后一行代码会调用 glideContext.buildImageViewTarget() 方法构建出一个 Target 对象，然后再把它传入到另一个接收 Target 参数的 into() 方法中。Target 对象则是用来最终展示图片用的，跟进方法 glideContext.buildImageViewTarget：

```java
@NonNull
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
    @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
  return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
}
```

继续跟进：

```java
public class ImageViewTargetFactory {
  @NonNull
  @SuppressWarnings("unchecked")
  public <Z> ViewTarget<ImageView, Z> buildTarget(
      @NonNull ImageView view, @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
}
```

回顾之前的分析流程，在 into(ImageView) 过程中，会将 ImageView 包装成为一个 ViewTarget 类。如果调用过 asBitmap() 方法，那么此处会是 BitmapImageViewTarget，否则都将会是 DrawableImageViewTarget。BitmapImageViewTarget 和 DrawableImageViewTarget 除了 setResource 方法中调用的设置图片的 API 不同外，没有任何区别。

DrawableImageView 的继承链如下：DrawableImageView -> ImageViewTarget -> ViewTarget -> BaseTarget -> Target。

- Target 是一个继承了 LifecycleListener 接口的接口类，该类提供了资源加载过程中的回调操作。

典型的生命周期为 onLoadStarted -> onResourceReady/onLoadFailed -> onLoadCleared，但不保证所有的都是这样。如果资源在内存中或者由于 model 为 null 而加载失败，onLoadStarted 不会被调用。同样，如果 target 不会清除，那么 onLoadCleared 方法也不会被调用。

- BaseTarget 是一个实现了 Target 接口的抽象类。

该类实现了 setRequest(Request)、getRequest() 两个方法，其他方法相当于适配器模式的实现。

- ViewTarget

该类虽然继承了 BaseTarget 类，但其重写了 setRequest(Request)、getRequest() 两个方法，这两个方法会调用 View.setTag 方法将 Request 对象传入。

In addition, <strong>for</strong> ViewTarget<strong>s only</strong>, you can pass in a new instance to each load or clear call and allow Glide to retrieve information about previous loads from the Views tags

This will <strong>not</strong> work unless your Target extends ViewTarget or implements setRequest() and getRequest() in a way that allows you to retrieve previous requests in new Target instances.

[Cancellation and re-use](http://bumptech.github.io/glide/doc/targets.html#cancellation-and-re-use)

- ImageViewTarget

该类的作用就是在加载的生命周期回调中给 ImageView 设置对应的资源。但由于加载成功后返回的资源可能是 Bitmap 或者 Drawable，所以这个不确定类型的加载由 setResource 抽象方法声明，待子类 BitmapImageViewTarget 和 DrawableImageViewTarget 实现。

- DrawableImageViewTarget

继承了 ImageViewTarget 类，唯一的作用就是实现 setResource(Drawable) 方法。

在了解了 DrawableImageViewTarget 以及相关的类之后，我们看一下其他的 Target。下面是 Glide v4 中出现的所有的 Target：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-3/clipboard_20230323_031827.png)

虽然 Target 很多，但是自定义只需要继承 CustomViewTarget 或者 CustomTarget 就行了。

<strong>为什么要继承 </strong>CustomViewTarget <strong>而不是 </strong>ViewTarget？
ViewTarget 已经被标记为废弃了，建议我们使用 CustomViewTarget。这是因为，如果子类没有实现 ViewTarget.onLoadCleared 方法，将会导致被回收的 bitmap 仍然被 UI 所引用，从而导致崩溃。而 CustomViewTarget.onLoadCleared 方法是 final 类型的，并且提供了一个抽象方法 onResourceCleared 强制我们实现。除此之外，两个类基本没有任何区别。

<strong>为什么要继承 </strong>CustomTarget <strong>而不是 </strong>SimpleTarget？原因同上。

下面举一个实际例子，在某些场景下，此时我们需要获取到加载成功后的 Bitma p 对象：

```kotlin
Glide.with(this)
    .asBitmap()
    .load(file)
    .into(object : CustomTarget<Bitmap>() {
        override fun onResourceReady(
            resource: Bitmap,
            transition: Transition<in Bitmap>?
        ) {
            ivFace.setImageBitmap(resource)
        }

        override fun onLoadCleared(placeholder: Drawable?) {
            ivFace.setImageDrawable(placeholder)
        }
    })
```

## RequestBuilder 高级 API

在了解了 Target 之后，我们再看看 RequestBuilder 中高级一点的 API。

下面这些都是 Target 的应用，内部调用的是 **修改过配置** 的 into/submit 方法，但 RequestBuilder.downloadOnly 方法已经被废弃；建议采用 RequestManager 的 downloadOnly() 方法和 into/submit 方法

此外还有还需要注意的一个 API：listener/addListener

### preload

作用：将资源预加载到缓存中

preload 的重载方法如下：

```java
/**
  * Preloads the resource into the cache using the given width and height.
  *
  * <p> Pre-loading is useful for making sure that resources you are going to to want in the near
  * future are available quickly. </p>
  *
  * @param width  The desired width in pixels, or {@link Target#SIZE_ORIGINAL}. This will be
  *               overridden by
  *               {@link com.bumptech.glide.request.RequestOptions#override(int, int)} if
  *               previously called.
  * @param height The desired height in pixels, or {@link Target#SIZE_ORIGINAL}. This will be
  *               overridden by
  *               {@link com.bumptech.glide.request.RequestOptions#override(int, int)}} if
  *               previously called).
  * @return A {@link Target} that can be used to cancel the load via
  * {@link RequestManager#clear(Target)}.
  * @see com.bumptech.glide.ListPreloader
  */
@NonNull
public Target<TranscodeType> preload(int width, int height) {
  final PreloadTarget<TranscodeType> target = PreloadTarget.obtain(requestManager, width, height);
  return into(target);
}

/**
  * Preloads the resource into the cache using {@link Target#SIZE_ORIGINAL} as the target width and
  * height. Equivalent to calling {@link #preload(int, int)} with {@link Target#SIZE_ORIGINAL} as
  * the width and height.
  *
  * @return A {@link Target} that can be used to cancel the load via
  * {@link RequestManager#clear(Target)}
  * @see #preload(int, int)
  */
@NonNull
public Target<TranscodeType> preload() {
  return preload(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL);
}
```

注意，在注释中出现了一个 ListPreload 类，该类是在 ListView 中做 item 预加载的一个工具类，使用方法为 AbsListView#setOnScrollListener(android.widget.AbsListView.OnScrollListener)。该类代码很简单，要点就是在滚动时计算需要预处理的 item。

这么好用，那我要是 RecyclerView 怎么办？Glide 也提供了 RecyclerView 的版本，不过需要添加新的依赖 recyclerview-integration，详情可以查看文档 [INTEGRATION LIBRARIES - RecyclerView](http://bumptech.github.io/glide/int/recyclerview.html)。

我们可以看到，在 preload 的实现中关键点就在于 PreloadTarget 类。该类实现非常简单，就是在 onResourceReady 回调发生后，经过 Handler 中转，最后由构造参数之一的 RequestManager 对象 clear 掉。PreloadTarget 实现代码如下：

```java
/**
 * A one time use {@link com.bumptech.glide.request.target.Target} class that loads a resource into
 * memory and then clears itself.
 *
 * @param <Z> The type of resource that will be loaded into memory.
 */
public final class PreloadTarget<Z> extends SimpleTarget<Z> {
  private static final int MESSAGE_CLEAR = 1;
  private static final Handler HANDLER = new Handler(Looper.getMainLooper(), new Callback() {
    @Override
    public boolean handleMessage(Message message) {
      if (message.what == MESSAGE_CLEAR) {
        ((PreloadTarget<?>) message.obj).clear();
        return true;
      }
      return false;
    }
  });

  private final RequestManager requestManager;

  /**
   * Returns a PreloadTarget.
   *
   * @param width  The width in pixels of the desired resource.
   * @param height The height in pixels of the desired resource.
   * @param <Z>    The type of the desired resource.
   */
  public static <Z> PreloadTarget<Z> obtain(RequestManager requestManager, int width, int height) {
    return new PreloadTarget<>(requestManager, width, height);
  }

  private PreloadTarget(RequestManager requestManager, int width, int height) {
    super(width, height);
    this.requestManager = requestManager;
  }

  @Override
  public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    HANDLER.obtainMessage(MESSAGE_CLEAR, this).sendToTarget();
  }

  @SuppressWarnings("WeakerAccess")
  @Synthetic void clear() {
    requestManager.clear(this);
  }
}
```

### submit

作用：返回一个 Future 对象，其 get() 方法会阻塞住，所以需要在后台线程中调用

submit 的两个重载方法如下：

```java
/**
  * Returns a future that can be used to do a blocking get on a background thread.
  *
  * <p>This method defaults to {@link Target#SIZE_ORIGINAL} for the width and the height. However,
  * since the width and height will be overridden by values passed to {@link
  * RequestOptions#override(int, int)}, this method can be used whenever {@link RequestOptions}
  * with override values are applied, or whenever you want to retrieve the image in its original
  * size.
  *
  * @see #submit(int, int)
  * @see #into(Target)
  */
@NonNull
public FutureTarget<TranscodeType> submit() {
  return submit(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL);
}

/**
  * Returns a future that can be used to do a blocking get on a background thread.
  *
  * @param width  The desired width in pixels, or {@link Target#SIZE_ORIGINAL}. This will be
  *               overridden by
  *               {@link com.bumptech.glide.request.RequestOptions#override(int, int)} if
  *               previously called.
  * @param height The desired height in pixels, or {@link Target#SIZE_ORIGINAL}. This will be
  *               overridden by
  *               {@link com.bumptech.glide.request.RequestOptions#override(int, int)}} if
  *               previously called).
  */
@NonNull
public FutureTarget<TranscodeType> submit(int width, int height) {
  final RequestFutureTarget<TranscodeType> target = new RequestFutureTarget<>(width, height);
  return into(target, target, Executors.directExecutor());
}
```

由于方法会生成一个 RequestFutureTarget 对象，而其 getSize 的实现就是构造参数。所以，此处的值会覆盖掉 RequestOptions 设置的值。

submit 之后生成了一个 RequestFutureTarget 对象，调用该对象的 get 方法可以在资源加载成功后立即获得资源对象，在获得之前会阻塞，所以 get 方法需要在后台线程中执行，否则会报错。

RequestFutureTarget 的示例代码如下：

```java
FutureTarget<File> target = null;
RequestManager requestManager = Glide.with(context);
try {
  target = requestManager
     .downloadOnly()
     .load(model)
     .submit();
  File downloadedFile = target.get();
  // ... do something with the file (usually throws IOException)
} catch (ExecutionException | InterruptedException | IOException e) {
  // ... bug reporting or recovery
} finally {
  // make sure to cancel pending operations and free resources
  if (target != null) {
    target.cancel(true); // mayInterruptIfRunning
  }
}
```

### downloadOnly

作用：下载原始的无修改的 data 文件。

downloadOnly 内部调用的是修改过配置的 into/submit 方法，但 downloadOnly 方法已经被废弃；建议采用 RequestManager 的 downloadOnly() 方法和 into/submit 方法。

实际上 RequestBuilder.downloadOnly 方法与 RequestManager.downloadOnly()、RequestBuilder.into/submit 方法组合没有什么区别。

两处代码如下：

<em>RequestBuilder.downloadOnly</em>

```java
@Deprecated
@CheckResult
public <Y extends Target<File>> Y downloadOnly(@NonNull Y target) {
  return getDownloadOnlyRequest().into(target);
}

@Deprecated
@CheckResult
public FutureTarget<File> downloadOnly(int width, int height) {
  return getDownloadOnlyRequest().submit(width, height);
}

@NonNull
@CheckResult
protected RequestBuilder<File> getDownloadOnlyRequest() {
  return new RequestBuilder<>(File.class, this).apply(DOWNLOAD_ONLY_OPTIONS);
}
```

<em>RequestManager.downloadOnly、RequestBuilder.into/submit</em>

```typescript
@NonNull
public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
  return into(target, /*targetListener=*/ null, Executors.mainThreadExecutor());
}

public FutureTarget<TranscodeType> submit(int width, int height) {
  final RequestFutureTarget<TranscodeType> target = new RequestFutureTarget<>(width, height);
  return into(target, target, Executors.directExecutor());
}

@NonNull
@CheckResult
public RequestBuilder<File> downloadOnly() {
  return as(File.class).apply(DOWNLOAD_ONLY_OPTIONS);
}
```

所以，这里的 DOWNLOAD_ONLY_OPTIONS 才是 downloadOnly 的精髓，我们看看该变量的值：

```java
protected static final RequestOptions DOWNLOAD_ONLY_OPTIONS =
    new RequestOptions().diskCacheStrategy(DiskCacheStrategy.DATA).priority(Priority.LOW)
        .skipMemoryCache(true);
```

果然是下载的是原始的无修改的 data 资源。

### listener/addListener

listener 与 addListener 不同之处在于，前者只会保留当前的 Listener，而后者会保留之前的 Listener。

```kotlin
@NonNull
@CheckResult
@SuppressWarnings("unchecked")
public RequestBuilder<TranscodeType> listener(
    @Nullable RequestListener<TranscodeType> requestListener) {
  this.requestListeners = null;
  return addListener(requestListener);
}

@NonNull
@CheckResult
public RequestBuilder<TranscodeType> addListener(
    @Nullable RequestListener<TranscodeType> requestListener) {
  if (requestListener != null) {
    if (this.requestListeners == null) {
      this.requestListeners = new ArrayList<>();
    }
    this.requestListeners.add(requestListener);
  }
  return this;
}
```

这些 listener 会在资源记载失败或者成功的时候被调用，SingleRequest 中关于 requestListeners 的代码：

```java
private synchronized void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
  // We must call isFirstReadyResource before setting status.
  boolean isFirstResource = isFirstReadyResource();
  status = Status.COMPLETE;
  this.resource = resource;

  isCallingCallbacks = true;
  try {
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

    if (!anyListenerHandledUpdatingTarget) {
      Transition<? super R> animation =
          animationFactory.build(dataSource, isFirstResource);
      target.onResourceReady(result, animation);
    }
  } finally {
    isCallingCallbacks = false;
  }

  notifyLoadSuccess();
}

private synchronized void onLoadFailed(GlideException e, int maxLogLevel) {
  stateVerifier.throwIfRecycled();
  e.setOrigin(requestOrigin);

  loadStatus = null;
  status = Status.FAILED;

  isCallingCallbacks = true;
  try {
    //TODO: what if this is a thumbnail request?
    boolean anyListenerHandledUpdatingTarget = false;
    if (requestListeners != null) {
      for (RequestListener<R> listener : requestListeners) {
        anyListenerHandledUpdatingTarget |=
            listener.onLoadFailed(e, model, target, isFirstReadyResource());
      }
    }
    anyListenerHandledUpdatingTarget |=
        targetListener != null
            && targetListener.onLoadFailed(e, model, target, isFirstReadyResource());

    if (!anyListenerHandledUpdatingTarget) {
      setErrorPlaceholder();
    }
  } finally {
    isCallingCallbacks = false;
  }

  notifyLoadFailed();
}
```

调用逻辑是这样：在 requestListeners 集合、targetListener 中依次调用对应的回调，找到第一个能够处理的 (返回 true)，后面的就不再调用。

同时，如果有一个回调返回了 true，那么资源的对应方法会被拦截：

1. 对于 onResourceReady 方法来说，Target 的 onResourceReady 方法不会被回调
2. 对于 onLoadFailed 方法来说，setErrorPlaceholder 调用不会调用，即不会显示任何失败的占位符

# Transform

## Transform 行为分析

### 场景还原

布局文件：ImageView 宽高都是 wrap_content

```html
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ImageView
        android:id="@+id/ivGlide"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"/>

    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="24dp"
        android:src="@drawable/btn_camera"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"/>

</android.support.constraint.ConstraintLayout>
```

代码：测试的可选项为代码中被注释的两行代码

```kotlin
...
import com.bumptech.glide.request.target.Target as GlideTarget

class GlideFragment : Fragment() {
    ...
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        fab.setOnClickListener {
            load()
        }
    }

    private fun load() {
        Glide.with(this)
            .load(URL)
//            .dontTransform()
//            .override(GlideTarget.SIZE_ORIGINAL, GlideTarget.SIZE_ORIGINAL)
            .into(ivGlide)
    }

    companion object {
        // 540*258  1080/(759-243)-(540/258)=0
        private const val URL = "https://www.baidu.com/img/bd_logo1.png"
    }
}
```

整个例子非常简单，点击一下 FAB 就会调用 load 方法加载网路图片。注意这里 ImageView 的宽高都是 wrap_content。

下面从左至右 4 张图分别对应以下配置开启时的效果（下面分析时会称为例 1、例 2、例 3、例 4）：

1. 直接加载
2. 只使用 dontTransform
3. 只使用 override
4. dontTransform 和 override 同时

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-3/clipboard_20230323_031837.png)

可以很明显的看到，显示的图片也有所差异，ImageView 的宽高有所差异。

如果没有调用 override(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL)，ImageView 宽度会是 match_parent 的效果，且如果 dontTransform()，那么高度也是 match_parent 效果。调用了 override 之后会显示的图片的真实尺寸。

带着这个问题，到源码中看看是怎么回事。

### 代码分析

首先 dontTransform() 方法实现在 BaseRequestOptions 文件中，该方法的意思就是不使用任何 transform：

```java
public T dontTransform() {
  if (isAutoCloneEnabled) {
    return clone().dontTransform();
  }

  transformations.clear();
  fields &= ~TRANSFORMATION;
  isTransformationRequired = false;
  fields &= ~TRANSFORMATION_REQUIRED;
  isTransformationAllowed = false;
  fields |= TRANSFORMATION_ALLOWED;
  isScaleOnlyOrNoTransform = true;
  return selfOrThrowIfLocked();
}
```

这里首先清空了 transformations，然后设置了一些标志位。这些标志位后面看到再说。然后是 override 方法，该方法的意思是只加载指定宽高的像素的图片，方法实现只是保存了参数，待后面使用：

```kotlin
@NonNull
@CheckResult
public T override(int width, int height) {
  if (isAutoCloneEnabled) {
    return clone().override(width, height);
  }

  this.overrideWidth = width;
  this.overrideHeight = height;
  fields |= OVERRIDE;

  return selfOrThrowIfLocked();
}
```

在配置完参数后，调用 into 方法开始加载图片。整个加载流程非常冗长，详情请看前面的分析文章。所以这里只挑选相关的代码，其他的会一笔带过。

先是 RequestBuilder.into(ImageView) 的实现，该方法会默认进行一些 transform 的配置：

```java
@NonNull
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread();
  Preconditions.checkNotNull(view);

  BaseRequestOptions<?> requestOptions = this;
  if (!requestOptions.isTransformationSet()
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    // Clone in this method so that if we use this RequestBuilder to load into a View and then
    // into a different target, we don't retain the transformation applied based on the previous
    // View's scale type.
    switch (view.getScaleType()) {
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER:
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

  return into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions,
      Executors.mainThreadExecutor());
}
```

在上面代码第 7 行的判断语句 !requestOptions.isTransformationSet()，判断的是 fields 是否含有标记位 TRANSFORMATION。

第 8 行判断语句 requestOptions.isTransformationAllowed() 判断的是 isTransformationAllowed。

对于这两个判断条件，如果调用了 dontTransform() 方法（例 2、例 4），那么显然条件 1 满足，但是条件 2 就不满足了。否则（例 1、例 3），这两个条件都是满足的，也就意味着 Glide 会有一个默认的 transform。默认的 transform 会根据 ImageView 的 ScaleType 有不同的取值。

ScaleType 与默认的 transform 之间关系如下表：

在 ImageView.initImageView() 方法中可以看到，默认的 ScaleType 为 ScaleType.FIT_CENTER。所以，此时 Glide 会默认调用 optionalFitCenter() 方法。

注意，optionalFitCenter() 方法与非 optional 的 fitCenter() 方法的差别在于 isTransformationRequired 参数的值不同。在最后 DecodeHelper.getTransformation(Class) 方法中，如果找不到可用的 transform，且 isTransformationRequired 为 true，就会抛出 IllegalArgumentException。除此之外两者没有别的差别。

optionalFitCenter() 方法会针对四种不同类型的数据产生不同的 transformation：

现在可以开始加载图片了，最开始会调用 SingleRequest.begin 方法，在该方法中会判断需要显示多大尺寸的图片。如果指定了 override(int, int)（例 3、例 4），那么就无需 ViewTarget 参与计算，直接调用 onSizeReady(int, int) 开始下一步；否则（例 1、例 2）就要计算了：

```java
@Override
public synchronized void begin() {
  if (model == null) {
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      width = overrideWidth;
      height = overrideHeight;
    }
    return;
  }

  if (status == Status.RUNNING) {
    throw new IllegalArgumentException("Cannot restart a running request");
  }

  if (status == Status.COMPLETE) {
    onResourceReady(resource, DataSource.MEMORY_CACHE);
    return;
  }

  // Restarts for requests that are neither complete nor running can be treated as new requests
  // and can run again from the beginning.

  status = Status.WAITING_FOR_SIZE;
  if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
    onSizeReady(overrideWidth, overrideHeight);
  } else {
    target.getSize(this);
  }

  if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
      && canNotifyStatusChanged()) {
    target.onLoadStarted(getPlaceholderDrawable());
  }
}
```

显然，造成示例中图片尺寸差异的一个原因就是这个第 27 行的 target.getSize(this) 方法了。对应方法的实现在 ViewTarget 中，这个请求会转发给内部的 SizeDeterminer 来完成。看一下 ViewTarget.SizeDeterminer 类：

```java
static final class SizeDeterminer {
  void getSize(@NonNull SizeReadyCallback cb) {
    int currentWidth = getTargetWidth();
    int currentHeight = getTargetHeight();
    if (isViewStateAndSizeValid(currentWidth, currentHeight)) {
      cb.onSizeReady(currentWidth, currentHeight);
      return;
    }
    ...
  }

  private int getTargetHeight() {
    int verticalPadding = view.getPaddingTop() + view.getPaddingBottom();
    LayoutParams layoutParams = view.getLayoutParams();
    int layoutParamSize = layoutParams != null ? layoutParams.height : PENDING_SIZE;
    return getTargetDimen(view.getHeight(), layoutParamSize, verticalPadding);
  }

  private int getTargetWidth() {
    int horizontalPadding = view.getPaddingLeft() + view.getPaddingRight();
    LayoutParams layoutParams = view.getLayoutParams();
    int layoutParamSize = layoutParams != null ? layoutParams.width : PENDING_SIZE;
    return getTargetDimen(view.getWidth(), layoutParamSize, horizontalPadding);
  }
}
```

首先在第 3、4 行调用方法获取 Target 的宽高，由于在本例中宽高都是 WRAP_CONTENT，而且也没有其他的限制，所以最后调用 getTargetDimen 时传入的参数都是一样的 (viewSize=0, paramSize=WRAP_CONTENT=-2, paddingSize=0)。然后，看看 getTargetDimen 方法的实现：

```java
private int getTargetDimen(int viewSize, int paramSize, int paddingSize) {
  // We consider the View state as valid if the View has non-null layout params and a non-zero
  // layout params width and height. This is imperfect. We're making an assumption that View
  // parents will obey their child's layout parameters, which isn't always the case.
  int adjustedParamSize = paramSize - paddingSize;
  if (adjustedParamSize > 0) {
    return adjustedParamSize;
  }

  // Since we always prefer layout parameters with fixed sizes, even if waitForLayout is true,
  // we might as well ignore it and just return the layout parameters above if we have them.
  // Otherwise we should wait for a layout pass before checking the View's dimensions.
  if (waitForLayout && view.isLayoutRequested()) {
    return PENDING_SIZE;
  }

  // We also consider the View state valid if the View has a non-zero width and height. This
  // means that the View has gone through at least one layout pass. It does not mean the Views
  // width and height are from the current layout pass. For example, if a View is re-used in
  // RecyclerView or ListView, this width/height may be from an old position. In some cases
  // the dimensions of the View at the old position may be different than the dimensions of the
  // View in the new position because the LayoutManager/ViewParent can arbitrarily decide to
  // change them. Nevertheless, in most cases this should be a reasonable choice.
  int adjustedViewSize = viewSize - paddingSize;
  if (adjustedViewSize > 0) {
    return adjustedViewSize;
  }

  // Finally we consider the view valid if the layout parameter size is set to wrap_content.
  // It's difficult for Glide to figure out what to do here. Although Target.SIZE_ORIGINAL is a
  // coherent choice, it's extremely dangerous because original images may be much too large to
  // fit in memory or so large that only a couple can fit in memory, causing OOMs. If users want
  // the original image, they can always use .override(Target.SIZE_ORIGINAL). Since wrap_content
  // may never resolve to a real size unless we load something, we aim for a square whose length
  // is the largest screen size. That way we're loading something and that something has some
  // hope of being downsampled to a size that the device can support. We also log a warning that
  // tries to explain what Glide is doing and why some alternatives are preferable.
  // Since WRAP_CONTENT is sometimes used as a default layout parameter, we always wait for
  // layout to complete before using this fallback parameter (ConstraintLayout among others).
  if (!view.isLayoutRequested() && paramSize == LayoutParams.WRAP_CONTENT) {
    if (Log.isLoggable(TAG, Log.INFO)) {
      Log.i(TAG, "Glide treats LayoutParams.WRAP_CONTENT as a request for an image the size of"
          + " this device's screen dimensions. If you want to load the original image and are"
          + " ok with the corresponding memory cost and OOMs (depending on the input size), use"
          + " .override(Target.SIZE_ORIGINAL). Otherwise, use LayoutParams.MATCH_PARENT, set"
          + " layout_width and layout_height to fixed dimension, or use .override() with fixed"
          + " dimensions.");
    }
    return getMaxDisplayLength(view.getContext());
  }

  // If the layout parameters are < padding, the view size is < padding, or the layout
  // parameters are set to match_parent or wrap_content and no layout has occurred, we should
  // wait for layout and repeat.
  return PENDING_SIZE;
}
```

在 getTargetDimen 方法中会通过参数计算 Target 的尺寸，显然这里最后会调用 getMaxDisplayLength(view.getContext()) 来返回 View 的尺寸。至于为什么 wrap_content 时会这样做，在注释中说的很清楚了，主要是考虑到加载的资源的不确定性，万一是一个大图片就直接 OOM 了。而且 Glide 会打印出 log 告诉我们一些解决方法，如果我们不满意这种处理方式。

getMaxDisplayLength(Context) 会返回 Display size 宽高的较大者：

```java
// Use the maximum to avoid depending on the device's current orientation.
private static int getMaxDisplayLength(@NonNull Context context) {
  if (maxDisplayLength == null) {
    WindowManager windowManager =
        (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    Display display = Preconditions.checkNotNull(windowManager).getDefaultDisplay();
    Point displayDimensions = new Point();
    display.getSize(displayDimensions);
    maxDisplayLength = Math.max(displayDimensions.x, displayDimensions.y);
  }
  return maxDisplayLength;
}
```

显然，在示例中，最后获取到的宽高都是一样的，在竖屏状态下会是 Display 的高。

<strong>Display#getSize 与 Display#getRealSize 的区别？</strong>

Display#getRealSize 返回的是屏幕真实的尺寸；而 Display#getSize 返回的是“可用”的屏幕尺寸，常见的情况是，有 NavigationBar 就会等于 RealSize 减去 NavigationBar 的高度。

获取到了 Target 的尺寸之后，会调用 isViewStateAndSizeValid 方法检查是否 OK：OK 就调用 SingleRequest.onSizeReady 方法进入下一步了；否则会给 ViewTree 添加 OnPreDrawListener 监听器，在 onPreDraw 方法回调中再次获取一下尺寸。

在经过一系列操作，加载到原始数据之后，会现在 DecodePath.decode 中对原始 data 进行 decode，这样就得到了 Bitmap，但是过程中涉及到 Bitmap 尺寸的操作。具体经过 ByteBufferBitmapDecoder 到 Downsampler#decodeFromWrappedStreams 方法中。里面绝大多数都是计算，这里拿例 1 来描述计算过程：

1. 获取 Bitmap 的宽高：sourceWidth=540,sourceHeight=258；获取 Target 的宽高：targetWidth=targetHeight=1776
2. 调用 calculateScaling 方法计算 inSampleSize、inTargetDensity、inDensity 等一系列参数，这里会用到 DownsampleStrategy.FitCenter 类，该对象在前面 optionalFitCenter() 过程保存的 KV 表格中提到过。其 getScaleFactor 方法实现就是取 target 宽高 / source 宽高的较小者，也就是 3.288889，该值由 inTargetDensity/inDensity 保存。
3. 最后计算 expectedWidth=round(540*3.288889)=1776，expectedHeight=round(258*3.288889)=849，并从 BitmapPool 中获取这么一个宽高的 Bitmap 设置给 Options.inBitmap
4. 再次进行解码，解码完毕后获得的图片就是 1776x849 大小的图片了。

如果是 dontTransform（例 2），DownsampleStrategy 会是默认值 DownsampleStrategy.CenterOutside。其 getScaleFactor 方法实现就是取 target 宽高 / source 宽高的较大者，也就是 6.883721。因此最后解码完毕的图片大小为 3717x1776。

如果是 override 过的（例 3、例 4），由于不需要缩放，所以返回的就是原始大小的图片。

在 decode 完毕后，会回到 DecodeJob.onResourceDecoded 方法中接着进行 transform 操作：

```java
@Synthetic
@NonNull
<Z> Resource<Z> onResourceDecoded(DataSource dataSource,
    @NonNull Resource<Z> decoded) {
  @SuppressWarnings("unchecked")
  Class<Z> resourceSubClass = (Class<Z>) decoded.get().getClass();
  Transformation<Z> appliedTransformation = null;
  Resource<Z> transformed = decoded;
  if (dataSource != DataSource.RESOURCE_DISK_CACHE) {
    appliedTransformation = decodeHelper.getTransformation(resourceSubClass);
    transformed = appliedTransformation.transform(glideContext, decoded, width, height);
  }
  ...
}
```

在这里首先判断是不是 DataSource.RESOURCE_DISK_CACHE，如果不是才需要进行 transform。

<strong>But, why?</strong> Glide 中很重要的概念——resource 和 data：差别在于 resource 是已经 transform 过了的，即 transform(data)=resource。所以，我们知道 resource 已经经过 transform 过了，自然就不再需要。

对于调用过 dontTransform() 方法的例子（例 2、例 4）来说，decodeHelper.getTransformation 返回的是一个 UnitTransformation，其 transform 没有干任何有意义的事情，也就是说不进行 transform。

对于普通的 Glide 加载请求（例 1、例 3）来说，一个 URL 已经经过一系列 Registry 的变换，到这里就变成 了 Bitmap.class，所以在第 11 行调用的是 FitCenter().transform(Context, Resource<Bitmap>, int, int ) 方法。而 FitCenter 又是 BitmapTransformation 的子类，所以先看看 BitmapTransformation：

```java
public abstract class BitmapTransformation implements Transformation<Bitmap> {

  @NonNull
  @Override
  public final Resource<Bitmap> transform(
      @NonNull Context context, @NonNull Resource<Bitmap> resource, int outWidth, int outHeight) {
    if (!Util.isValidDimensions(outWidth, outHeight)) {
      throw new IllegalArgumentException(
          "Cannot apply transformation on width: " + outWidth + " or height: " + outHeight
              + " less than or equal to zero and not Target.SIZE_ORIGINAL");
    }
    BitmapPool bitmapPool = Glide.get(context).getBitmapPool();
    Bitmap toTransform = resource.get();
    int targetWidth = outWidth == Target.SIZE_ORIGINAL ? toTransform.getWidth() : outWidth;
    int targetHeight = outHeight == Target.SIZE_ORIGINAL ? toTransform.getHeight() : outHeight;
    Bitmap transformed = transform(bitmapPool, toTransform, targetWidth, targetHeight);

    final Resource<Bitmap> result;
    if (toTransform.equals(transformed)) {
      result = resource;
    } else {
      result = BitmapResource.obtain(transformed, bitmapPool);
    }
    return result;
  }

  protected abstract Bitmap transform(
      @NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth, int outHeight);
}
```

首先做 sanity check，然后获取 targetWidth、targetHeight。如果指定了 override(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL)，这里会是图片的真实尺寸，否则就是 Target 的尺寸了。

然后调用抽象方法进行 transform 操作，并返回 transform 后的结果。显然，transform 抽象方法就是子类需要实现的了，所以看看 FitCenter 的方法：

```java
public class FitCenter extends BitmapTransformation {
  private static final String ID = "com.bumptech.glide.load.resource.bitmap.FitCenter";
  private static final byte[] ID_BYTES = ID.getBytes(CHARSET);

  @Override
  protected Bitmap transform(@NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth,
      int outHeight) {
    return TransformationUtils.fitCenter(pool, toTransform, outWidth, outHeight);
  }

  @Override
  public boolean equals(Object o) {
    return o instanceof FitCenter;
  }

  @Override
  public int hashCode() {
    return ID.hashCode();
  }

  @Override
  public void updateDiskCacheKey(@NonNull MessageDigest messageDigest) {
    messageDigest.update(ID_BYTES);
  }
}
```

可以看到 FitCenter.transform 方法直接调用了 TransformationUtils 的 fitCenter 方法。除此之外，需要注意一下 equals、hashCode、updateDiskCacheKey 方法，这三个方法要重写的原因是因为 transform 会作为 key 的组成部分，参与到 key 的比较、缓存读写时 safeKey 生成，具体内容可以参考之前分析的文章。

至于 TransformationUtils 类，该类里面有很多处理图片的静态方法，内置的几种 BitmapTransformation 都是调用该工具类里面对应的方法来完成 transform 功能的。这里只看 fitCenter 方法：

```java
public static Bitmap fitCenter(@NonNull BitmapPool pool, @NonNull Bitmap inBitmap, int width,
    int height) {
  if (inBitmap.getWidth() == width && inBitmap.getHeight() == height) {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "requested target size matches input, returning input");
    }
    return inBitmap;
  }
  final float widthPercentage = width / (float) inBitmap.getWidth();
  final float heightPercentage = height / (float) inBitmap.getHeight();
  final float minPercentage = Math.min(widthPercentage, heightPercentage);

  // Round here in case we've decoded exactly the image we want, but take the floor below to
  // avoid a line of garbage or blank pixels in images.
  int targetWidth = Math.round(minPercentage * inBitmap.getWidth());
  int targetHeight = Math.round(minPercentage * inBitmap.getHeight());

  if (inBitmap.getWidth() == targetWidth && inBitmap.getHeight() == targetHeight) {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "adjusted target size matches input, returning input");
    }
    return inBitmap;
  }

  // Take the floor of the target width/height, not round. If the matrix
  // passed into drawBitmap rounds differently, we want to slightly
  // overdraw, not underdraw, to avoid artifacts from bitmap reuse.
  targetWidth = (int) (minPercentage * inBitmap.getWidth());
  targetHeight = (int) (minPercentage * inBitmap.getHeight());

  Bitmap.Config config = getNonNullConfig(inBitmap);
  Bitmap toReuse = pool.get(targetWidth, targetHeight, config);

  // We don't add or remove alpha, so keep the alpha setting of the Bitmap we were given.
  TransformationUtils.setAlpha(inBitmap, toReuse);

  if (Log.isLoggable(TAG, Log.VERBOSE)) {
    Log.v(TAG, "request: " + width + "x" + height);
    Log.v(TAG, "toFit:   " + inBitmap.getWidth() + "x" + inBitmap.getHeight());
    Log.v(TAG, "toReuse: " + toReuse.getWidth() + "x" + toReuse.getHeight());
    Log.v(TAG, "minPct:   " + minPercentage);
  }

  Matrix matrix = new Matrix();
  matrix.setScale(minPercentage, minPercentage);
  applyMatrix(inBitmap, toReuse, matrix);

  return toReuse;
}
```

该方法最开始会检查尺寸是否需要进行缩放，对于 override(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL)（例 3）来说，显然是不需要的，所以就直接返回了。

否则，对于例 1 来说，需要继续执行后面的代码显示合适的图片。由于 Bitmap 的宽高在 encode 过程中计算出来了，为 1776x849，而传入的 width、height 为 1776，显然 minPercentage 会取 widthPercentage，也就意味着会按照屏幕的宽度进行缩放。但 widthPercentage 为 1，所以导致第 18 行的判断条件为 true，因此直接返回了图片。

后面就是纯粹的展示问题了，在 DrawableImageViewTarget.setResource 方法中会将上面的图片设置到 ImageView 中，此时 ImageView 宽高都是 0，resource 的宽高为 1776x849。下面怎么显示就是 ImageView 决 定的了。

下面对四个例子做一个小结：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-3/clipboard_20230323_031849.png)

各例子之所以表现各不一样，其原因就在于 decode 出来的图片的尺寸不一致，而使用 wrap_content 宽高的 ImageView 加载这些图片，就会造成这些效果。

## Glide 自带的 Transformation

在上一节遇到的 FitCenter、CenterOutside 这两个 BitmapTransformation 都是 Glide 自带的，除了这两个外还有其他四个自带的。这些 BitmapTransformation 的名称和作用如下：

- CenterInside：如果图片的尺小于等于 target，图片保持不变；否则会调用 fitCenter。
- FitCenter：等比缩放，使得图片的一条边和 target 相同，另外一边小于给定的 target。
- CenterCrop：首先缩放图片，使图片的宽和给定的宽相等，且图片的高大于给定的高。或者高相等，图片的宽大于给定的宽。然后 crop，使较大的一边的像素和给定的相同。这种缩放最终效果显然不是等比缩放。
- Rotate：旋转 Bitmap
- RoundedCorners：圆角，单位是 pixel
- CircleCrop：圆形

前面三个我们非常熟悉了，和 ImageView 的 ScaleType 有对应关系，所以下面的示例用来演示一下后面的三个 Transformation。

布局文件很简单，一个参照的 ImageView + 三个显示对应效果的 ImageView:

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.constraint.Guideline
        android:id="@+id/guideline"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintGuide_percent="0.5"
        android:orientation="vertical"/>

    <ImageView
        android:id="@+id/ivGlide1"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="@id/guideline"
        app:layout_constraintTop_toTopOf="parent"/>

    <ImageView
        android:id="@+id/ivGlide2"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:layout_constraintStart_toStartOf="@id/guideline"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent"/>

    <ImageView
        android:id="@+id/ivGlide3"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="@id/guideline"
        app:layout_constraintTop_toBottomOf="@id/ivGlide1"/>

    <ImageView
        android:id="@+id/ivGlide4"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:layout_constraintStart_toStartOf="@id/guideline"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toBottomOf="@id/ivGlide2"/>

    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="24dp"
        android:src="@drawable/btn_camera"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"/>

</android.support.constraint.ConstraintLayout>
```

源文件也比较简单：

```kotlin
// private const val URL = "http://cn.bing.com/az/hprichbg/rb/Dongdaemun_ZH-CN10736487148_1920x1080.jpg"

Glide.with(this)
    .load(URL)
    .into(ivGlide1)

Glide.with(this)
    .load(URL)
    .transform(FitCenter(), Rotate(180))
    .into(ivGlide2)

Glide.with(this)
    .load(URL)
    .transform(MultiTransformation(FitCenter(), RoundedCorners(303 ushr 1)))
    .into(ivGlide3)

Glide.with(this)
    .load(URL)
    .transform(CircleCrop())
    .into(ivGlide4)
```

Rotate、RoundedCorners、CircleCrop 效果图如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-3/clipboard_20230323_031855.png)

看完图后，这三个 Transformation 的特点一目了然了。注意上面第 2、3 个加载时，因为 ImageView 高度为 wrap_content，所以需要加上 FitCenter，使 Bitmap 的宽高达到我们的预期（图片宽高为 1920*1080，ImageView 宽度为 540，因此高度为 (540/1920*1080=303.75，转为 int 后为 303)），然后在此基础上进行圆角或圆形处理。

Glide 使用 map 保存 transformation，所以调用多个 transform 方法，只有最后一个才会生效。如果我们想要使用多个 transformation，可以使用 MultiTransformation 类或者 transforms(Transformation<Bitmap>...) 以及 transform(Transformation<Bitmap>...) 方法。

## 自定义 BitmapTransformation

如果我们想 transform Bitmap，继承 BitmapTransformation 是最好的方式了。BitmapTransformation 为我们处理了一些基础的东西，例如，如果 Transformation 返回了一个新的 Bitmap， BitmapTransformation 将负责提取和回收原始的 Bitmap。

下面是一个简单的实现：

```java
public class FillSpace extends BitmapTransformation {
    private static final String ID = "com.bumptech.glide.transformations.FillSpace";
    private static final String ID_BYTES = ID.getBytes(STRING_CHARSET_NAME);

    @Override
    public Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        if (toTransform.getWidth() == outWidth && toTransform.getHeight() == outHeight) {
            return toTransform;
        }

        return Bitmap.createScaledBitmap(toTransform, outWidth, outHeight, /*filter=*/ true);
    }

    @Override
    public void equals(Object o) {
      return o instanceof FillSpace;
    }

    @Override
    public int hashCode() {
      return ID.hashCode();
    }

    @Override
    public void updateDiskCacheKey(MessageDigest messageDigest)
        throws UnsupportedEncodingException {
      messageDigest.update(ID_BYTES);
    }
}
```

上面的例子虽然简单，但是包含了必须实现的方法：

1. transform 是该类存在的意义，作用不言而喻。
2. 第一个参数 pool，这个是 Glide 中的一个 Bitmap 缓存池，用于对 Bitmap 对象进行重用，否则每次图片变换都重新创建 Bitmap 对象将会非常消耗内存。
3. 第二个参数 toTransform，这个是原始图片的 Bitmap 对象，我们就是要对它来进行图片变换。
4. 第三和第四个参数比较简单，分别代表图片变换后的宽度和高度，其实也就是 override() 方法中传入的宽和高的值了。
5. equals、hashCode、updateDiskCacheKey 三个方法必须实现，因为磁盘缓存和内存缓存都需要这个。详情可以看之前的分析文章。

此外，如果自定义的 BitmapTransformation 有构造参数，那么 equals、hashCode、updateDiskCacheKey 三个方法也需要对构造参数做处理。比如 RoundedCorners：

```java
public final class RoundedCorners extends BitmapTransformation {
  private static final String ID = "com.bumptech.glide.load.resource.bitmap.RoundedCorners";
  private static final byte[] ID_BYTES = ID.getBytes(CHARSET);

  private final int roundingRadius;

  /**
   * @param roundingRadius the corner radius (in device-specific pixels).
   * @throws IllegalArgumentException if rounding radius is 0 or less.
   */
  public RoundedCorners(int roundingRadius) {
    Preconditions.checkArgument(roundingRadius > 0, "roundingRadius must be greater than 0.");
    this.roundingRadius = roundingRadius;
  }

  @Override
  protected Bitmap transform(
      @NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth, int outHeight) {
    return TransformationUtils.roundedCorners(pool, toTransform, roundingRadius);
  }

  @Override
  public boolean equals(Object o) {
    if (o instanceof RoundedCorners) {
      RoundedCorners other = (RoundedCorners) o;
      return roundingRadius == other.roundingRadius;
    }
    return false;
  }

  @Override
  public int hashCode() {
    return Util.hashCode(ID.hashCode(),
        Util.hashCode(roundingRadius));
  }

  @Override
  public void updateDiskCacheKey(@NonNull MessageDigest messageDigest) {
    messageDigest.update(ID_BYTES);

    byte[] radiusData = ByteBuffer.allocate(4).putInt(roundingRadius).array();
    messageDigest.update(radiusData);
  }
}
```

# AppGlideModule

Glide 之所以强大，另外一个重要的功能就是可以自定义模块，自定义模块功能可以将更改 Glide 配置，替换 Glide 组件等操作独立出来，使得我们能轻松地对 Glide 的各种配置进行自定义，并且又和 Glide 的图片加载逻辑没有任何交集，这也是一种低耦合编程方式的体现。

需要注意的是，更改 Glide 默认配置应该关注是否会造成性能倒退。

## 使用

首先，使用 kapt 将 compiler 引入到项目中：

```groovy
apply plugin: 'kotlin-kapt'

dependencies {
  implementation 'com.github.bumptech.glide:glide:4.12.0'
  kapt 'com.github.bumptech.glide:compiler:4.12.0'
}
```

并且在你的 `proguard.cfg` 中 keep 住你的 AppGlideModule 实现：

```plain text
-keep public class  extends com.bumptech.glide.module.AppGlideModule
-keep class com.bumptech.glide.GeneratedAppGlideModuleImpl
```

接着，新建一个类继承自 AppGlideModule：

```kotlin
@GlideModule
class MyAppGlideModule : AppGlideModule() {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
    }

    override fun isManifestParsingEnabled() = super.isManifestParsingEnabled()

    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
    }
}
```

可以看到，一共有三个方法可以进行重写，它们分别代表什么呢？

### applyOptions

applyOptions 方法主要用于更改 Glide 默认配置，比如修改硬盘缓存路径、设置 Bitmap 的解码格式、设置 LRUCahce 大小甚至使用自定义 LRUCahce 实现或者自定义磁盘缓存实现等。下面将罗列一些常用的配置选项。

#### <strong>内存缓存（MemoryCache）</strong>

Glide 使用 LruResourceCache 作为 MemoryCache 接口的一个默认实现，使用固定大小的内存和 LRU 算法。LruResourceCache 的大小由 Glide 的 MemorySizeCalculator 类来决定，这个类主要关注设备的内存类型，设备 RAM 大小，以及屏幕分辨率。

使用 applyOptions(Context, GlideBuilder) 方法配置 MemorySizeCalculator 可以自定义 MemoryCache 的大小：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    val calculator = MemorySizeCalculator.Builder(context)
        .setMemoryCacheScreens(2f)
        .build()
    builder.setMemoryCache(LruResourceCache(calculator.memoryCacheSize.toLong()))
}
```

也可以直接覆写缓存大小：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    val memoryCacheSizeBytes = 1024 * 1024 * 20 // 20mb
    builder.setMemoryCache(LruResourceCache(memoryCacheSizeBytes.toLong()))
}
```

甚至可以提供自己的 MemoryCache 实现：

```scala
override fun applyOptions(context: Context, builder: GlideBuilder) {
    builder.setMemoryCache(YourAppMemoryCacheImpl())
}
```

#### <strong>BitmapPool</strong>

Glide 使用 LruBitmapPool 作为默认的 BitmapPool 。LruBitmapPool 是一个内存中的固定大小的 BitmapPool，使用 LRU 算法清理。默认大小基于设备的分辨率和密度，同时也考虑内存类和 isLowRamDevice 的返回值。具体的计算通过 Glide 的 MemorySizeCalculator 来完成，与 Glide 的 MemoryCache 的大小检测方法相似。

可以在 AppGlideModule 中定制 BitmapPool 的尺寸，使用 applyOptions(Context, GlideBuilder) 方法并配置 MemorySizeCalculator:

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    val calculator = MemorySizeCalculator.Builder(context)
        .setBitmapPoolScreens(3f)
        .build()
    builder.setBitmapPool(LruBitmapPool(calculator.bitmapPoolSize.toLong()))
}
```

也可以直接复写这个池的大小：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    val bitmapPoolSizeBytes = 1024 * 1024 * 30 // 30mb
    builder.setBitmapPool(LruBitmapPool(bitmapPoolSizeBytes.toLong()))
}
```

当然也可以提供 BitmapPool 的完全自定义实现：

```scala
override fun applyOptions(context: Context, builder: GlideBuilder) {
    builder.setBitmapPool(YourAppBitmapPoolImpl())
}
```

#### <strong>磁盘缓存（DiskCache）</strong>

Glide 使用 DiskLruCacheWrapper 作为默认的 磁盘缓存 。 DiskLruCacheWrapper 是一个使用 LRU 算法的固定大小的磁盘缓存。默认磁盘大小为 250 MB ，位置是在应用的缓存文件夹中的一个特定目录（<em>/data/data/{package}/cache/image_manager_disk_cache/</em>） 。

假如使用 Glide 展示图片内容是公开的，那么应用可以将这个缓存位置改到外部存储：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    builder.setDiskCache(ExternalPreferredCacheDiskCacheFactory(context))
}
```

无论使用内部或外部磁盘缓存，应用程序都可以改变磁盘缓存的大小：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    val diskCacheSizeBytes = 1024 * 1024 * 100;  //100 MB
    builder.setDiskCache(ExternalPreferredCacheDiskCacheFactory(context,
         diskCacheSizeBytes.toLong()))
}
```

还可以自己命名外存或内存上的缓存文件夹：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    val diskCacheSizeBytes = 1024 * 1024 * 100;  //100 MB
    builder.setDiskCache(InternalCacheDiskCacheFactory(context, 
        cacheFolderName, diskCacheSizeBytes.toLong()))
}
```

此外，还可以自定义 DiskCache 接口的实现，并提供自己的 DiskCache.Factory 来创建缓存。Glide 使用一个工厂接口来在后台线程中打开磁盘缓存 ，这样方便缓存做诸如检查路径存在性等的 IO 操作而不用触发严格模式（StrictMode） 。

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    builder.setDiskCache(DiskCache.Factory { YourAppCustomDiskCache() })
}
```

#### <strong>默认请求选项（DefaultRequestOptions）</strong>

虽然 RequestOptions<strong> </strong>通常由每个请求单独指定，也可以通过 AppGlideModule 应用一个 RequestOptions 的集合以作用于每次请求，比如设置 Bitmap 的解码格式为 RGB_565 (默认为 ARGB_8888)：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    builder.setDefaultRequestOptions(
        RequestOptions()
            .format(DecodeFormat.PREFER_RGB_565)
            .disallowHardwareConfig()
    )
}
```

一旦创建了新的请求，这些选项将通过 GlideBuilder 中的 setDefaultRequestOptions 被应用上。因此，任何单独请求里应用的选项将覆盖 GlideBuilder 里设置的冲突选项。

类似地，RequestManagers 允许你为这个特定的 RequestManager 启动的所有加载请求设置默认的 RequestOptions。 因为每个 Activity 和 Fragment 都拥有自己的 RequestManager，你可以使用 RequestManager 的 applyDefaultRequestOptions 方法来设置默认的 RequestOption，并仅作用于一个特定的 Activity 或 Fragment：

```kotlin
Glide.with(fragment)
    .applyDefaultRequestOptions(
        RequestOptions()
            .format(DecodeFormat.PREFER_RGB_565)
            .disallowHardwareConfig()
    )
```

RequestManager 还有一个 setDefaultRequestOptions 方法，可以完全替换掉之前设置的任意的默认 RequestOptions，无论它是通过 AppGlideModule 的 [GlideBuilder] 还是 RequestManager。使用 [setDefaultRequestOptions] 要小心，因为很容易意外覆盖掉其他地方设置的重要默认选项。 通常 applyDefaultRequestOptions 更安全，使用起来更直观。

#### <strong>未捕获异常策略 (UncaughtThrowableStrategy)</strong>

在加载图片时假如发生了一个异常 (例如, OOM), Glide 将会使用一个 GlideExecutor.UncaughtThrowableStrategy 。

默认策略是将异常打印到设备的 LogCat 中。 这个策略从 Glide 4.2.0 起将可被定制。 你可以传入一个 DiskCacheExecutor 和 / 或一个 SourceExecutor：

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    val myUncaughtThrowableStrategy = UncaughtThrowableStrategy.LOG
    builder.setDiskCacheExecutor(GlideExecutor.newDiskCacheBuilder()
        .setUncaughtThrowableStrategy { myUncaughtThrowableStrategy }
        .build())
    builder.setSourceExecutor(GlideExecutor.newSourceBuilder()
        .setUncaughtThrowableStrategy { myUncaughtThrowableStrategy }
        .build())
}
```

#### <strong>日志级别（LogLevel ）</strong>

可以使用 setLogLevel (结合 Android 的 Log 定义的值) 来获取格式化日志的子集，包括请求失败时的日志行。通常来说 Log.VERBOSE 粒度太细，而 Log.ERROR 又会让日志更趋向静默。

```kotlin
override fun applyOptions(context: Context, builder: GlideBuilder) {
    builder.setLogLevel(Log.DEBUG)
}
```

所有的配置选项如下

| 方法                                | 作用 |
| ----------------------------------- | ---- |
| setBitmapPool                       |      |
| setArrayPool                        |      |
| setMemoryCache                      |      |
| setDiskCache                        |      |
| setResizeExecutor                   |      |
| setSourceExecutor                   |      |
| setDiskCacheExecutor                |      |
| setAnimationExecutor                |      |
| setDefaultRequestOptions            |      |
| setDefaultTransitionOptions         |      |
| setMemorySizeCalculator             |      |
| setConnectivityMonitorFactory       |      |
| setLogLevel                         |      |
| setIsActiveResourceRetentionAllowed |      |
| addGlobalRequestListener            |      |
| setLogRequestOrigins                |      |
| setImageDecoderEnabledForBitmaps    |      |

### isManifestParsingEnabled

在 Glide v3 中，使用自定义模块需要在 AndroidManifest.xml 文件中通过 meta-data 进行配置：

```xml
<application
    ...>

    <meta-data
        android:name="xxx.xxx.MyGlideModule"
        android:value="GlideModule" />

    <activity ... />
</application>
```

MyGlideModule 代码如下：

```kotlin
class MyGlideModule : GlideModule {
   override fun applyOptions(context: Context, builder: GlideBuilder) {
       ...
   }
   
   override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
       ...
   }
}
```

为了维持对 Glide v3 的 GlideModules 的向后兼容性，Glide 仍然会解析应用程序和所有被包含的库中的 AndroidManifest.xml 文件，并包含在这些清单中列出的旧 GlideModules 模块类。

如果你已经迁移到 Glide v4 的 AppGlideModule 和 LibraryGlideModule ，你可以完全禁用清单解析。这样可以改善 Glide 的初始启动时间，并避免尝试解析元数据时的一些潜在问题。要禁用清单解析，可以在 AppGlideModule 实现中复写 isManifestParsingEnabled() 方法：

```kotlin
override fun isManifestParsingEnabled() = false
```

### registerComponents

应用程序和库都可以注册很多组件来扩展 Glide 的功能。可用的组件包括：

1. ModelLoader, 用于加载自定义的 Model(Url, Uri,任意的 POJO ) 和 Data(InputStreams, FileDescriptors)。
2. ResourceDecoder, 用于对新的 Resources(Drawables, Bitmaps) 或新的 Data 类型 (InputStreams, FileDescriptors) 进行解码。
3. Encoder, 用于向 Glide 的磁盘缓存写 Data (InputStreams, FileDesciptors)。
4. ResourceTranscoder，用于在不同的资源类型之间做转换，例如，从 BitmapResource 转换为 DrawableResource 。
5. ResourceEncoder，用于向 Glide 的磁盘缓存写 Resources(BitmapResource, DrawableResource)。

组件通过 Registry 类来注册。例如，添加一个 ModelLoader ，使其能从自定义的 Model 对象中创建一个 InputStream ：

```kotlin
override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
    registry.append(Photo::class.java, InputStream::class.java, Factory())
}
```

在一个 GlideModule 里可以注册很多组件。ModelLoader 和 ResourceDecoder 对于同样的参数类型还可以有多种实现。

在前文的分析中知道了，Glide 加载网络图片默认使用的是 HttpUrlConnection，如果想把这个位置的实现替换成 OkHttp3 或者 Volley 实现可以吗？显然是可以的，而且官方也提供了这个功能：

- [OkHttp3](http://bumptech.github.io/glide/int/okhttp3.html)
- [Volley](http://bumptech.github.io/glide/int/volley.html)

以 OkHttp3 为例，用起来很简单，只需要集成一个 library 即可，非常符合 Glide 的风格。build.gradle 的配置如下：

```groovy
implementation "com.github.bumptech.glide:okhttp3-integration:4.12.0"
```

首先找到里面的 @GlideModule 修饰的类，而且应该是一个 LibraryGlideModule 类：

```scala
@GlideModule
public final class OkHttpLibraryGlideModule extends LibraryGlideModule {
  @Override
  public void registerComponents(@NonNull Context context, @NonNull Glide glide,
      @NonNull Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
  }
}
```

此外，里面还发现了一个实现了 com.bumptech.glide.module.GlideModule 接口的废弃类 OkHttpGlideModule：

```java
@Deprecated
public class OkHttpGlideModule implements com.bumptech.glide.module.GlideModule {
  @Override
  public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
    // Do nothing.
  }

  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
  }
}
```

由于我们使用的 Glide 版本为 4.x，使用了上面的 OkHttpLibraryGlideModule 并且无需在 AndroidManifest.xml 中配置，自然也用不到这个类。

回到 OkHttpLibraryGlideModule 类中，看看 registerComponents 方法的实现：

```java
registry.replace(GlideUrl.class, InputStream.class,
     new OkHttpUrlLoader.Factory());
```

对于此项，Glide 的默认配置中有如下代码：

```java
.append(GlideUrl.class, InputStream.class, 
        new HttpGlideUrlLoader.Factory())
```

所以可以理解为，原本交给 HttpGlideUrlLoader.Factory() 处理的任务会交给 OkHttpUrlLoader.Factory() 处理。

<strong>OkHttpUrlLoader</strong> 源码如下：

```java
public class OkHttpUrlLoader implements ModelLoader<GlideUrl, InputStream> {

  private final Call.Factory client;

  // Public API.
  @SuppressWarnings("WeakerAccess")
  public OkHttpUrlLoader(@NonNull Call.Factory client) {
    this.client = client;
  }

  @Override
  public boolean handles(@NonNull GlideUrl url) {
    return true;
  }

  @Override
  public LoadData<InputStream> buildLoadData(
      @NonNull GlideUrl model, int width, int height, @NonNull Options options) {
    return new LoadData<>(model, new OkHttpStreamFetcher(client, model));
  }

  /** The default factory for {@link OkHttpUrlLoader}s. */
  // Public API.
  @SuppressWarnings("WeakerAccess")
  public static class Factory implements ModelLoaderFactory<GlideUrl, InputStream> {
    private static volatile Call.Factory internalClient;
    private final Call.Factory client;

    private static Call.Factory getInternalClient() {
      if (internalClient == null) {
        synchronized (Factory.class) {
          if (internalClient == null) {
            internalClient = new OkHttpClient();
          }
        }
      }
      return internalClient;
    }

    /** Constructor for a new Factory that runs requests using a static singleton client. */
    public Factory() {
      this(getInternalClient());
    }

    /**
     * Constructor for a new Factory that runs requests using given client.
     *
     * @param client this is typically an instance of {@code OkHttpClient}.
     */
    public Factory(@NonNull Call.Factory client) {
      this.client = client;
    }

    @NonNull
    @Override
    public ModelLoader<GlideUrl, InputStream> build(MultiModelLoaderFactory multiFactory) {
      return new OkHttpUrlLoader(client);
    }

    @Override
    public void teardown() {
      // Do nothing, this instance doesn't own the client.
    }
  }
}
```

可以看到，OkHttpUrlLoader.Factory 的无参构造器会使用 DCL 单例模式创建一个 OkHttpClient() 对象，其 build 方法会返回一个 new OkHttpUrlLoader(client)。

OkHttpUrlLoader.buildLoadData 方法会返回一个 fetcher 为 <strong>OkHttpStreamFetcher</strong> 的 LoadData。当需要进行加载的时候，会调用 fetcher 的 loadData 方法：

```java
@Override
public void loadData(@NonNull Priority priority,
    @NonNull final DataCallback<? super InputStream> callback) {
    Request.Builder requestBuilder = new Request.Builder().url(url.toStringUrl());
    for (Map.Entry<String, String> headerEntry : url.getHeaders().entrySet()) {
        String key = headerEntry.getKey();
        requestBuilder.addHeader(key, headerEntry.getValue());
    }
    Request request = requestBuilder.build();
    this.callback = callback;

    call = client.newCall(request);
    call.enqueue(this);
}

@Override
public void onFailure(@NonNull Call call, @NonNull IOException e) {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "OkHttp failed to obtain result", e);
    }

    callback.onLoadFailed(e);
}

@Override
public void onResponse(@NonNull Call call, @NonNull Response response) {
    responseBody = response.body();
    if (response.isSuccessful()) {
        long contentLength = Preconditions.checkNotNull(responseBody).contentLength();
        stream = ContentLengthInputStream.obtain(responseBody.byteStream(), contentLength);
        callback.onDataReady(stream);
    } else {
        callback.onLoadFailed(new HttpException(response.message(), response.code()));
    }
}
```

OkHttpStreamFetcher 的实现比 HttpUrlFetcher 简单多了，而且看起来也没有什么难度。

如果我们想使用 App 现有的 OkHttpClient 而不是默认创建一个新的，我们可以先 @Excludes 掉 OkHttpLibraryGlideModule；然后在 replace 的同时，在 OkHttpUrlLoader.Factory(Call.Factory) 构造时注入现有的 OkHttpClient：

```kotlin
@GlideModule
@Excludes(value = [OkHttpLibraryGlideModule::class, MyLibraryGlideModule::class, MyGlideModule::class])
class MyAppGlideModule : AppGlideModule() {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        builder.setDiskCache(ExternalPreferredCacheDiskCacheFactory(context))
    }

    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
        val okHttpClient = OkHttpClient.Builder()
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .addNetworkInterceptor(
                HttpLoggingInterceptor {
                    Log.i("MyAppGlideModule", it)
                }.apply {
                    level = HttpLoggingInterceptor.Level.BODY
                }
            ).build()

        registry.replace(GlideUrl::class.java, InputStream::class.java,
             OkHttpUrlLoader.Factory(okHttpClient))
    }
}
```

## 原理

### Glide 初始化流程分析

之前在分析 [Glide 加载流程](https://ywue4d2ujm.feishu.cn/docs/doccnY69t6P4JXuanj2oz8GBp5g) 的文章中已经知道，with 方法调用过程中会调用到 Glide.get(context) 方法生成并返回全局 Glide 单例：

```java
public static Glide get(@NonNull Context context) {
  if (glide == null) {
    // 如果有配置 @GlideModule 注解的 Module
    // 这里会反射构造 kapt 生成的 GeneratedAppGlideModuleImpl 类
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


@GuardedBy("Glide.class")
private static void checkAndInitializeGlide(@NonNull Context context, @Nullable GeneratedAppGlideModule generatedAppGlideModule) {
    if (isInitializing) {
        throw new IllegalStateException("You cannot call Glide.get() in registerComponents(), use the provided Glide instance instead");
    } else {
        isInitializing = true;
        initializeGlide(context, generatedAppGlideModule);
        isInitializing = false;
    }
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

可以看到，这里首先会调用 getAnnotationGeneratedGlideModules() 方法反射构造出注解处理器生成的 GeneratedAppGlideModuleImpl 对象。

接着，Glide 的创建（line 84）发生在 applyOptions 之后，registerComponents 之前。也好理解，因为 applyOptions 是更改配置，肯定是初始化时就要确定的；而 registerComponents 针对的运行时的功能扩展，而且需要调用 Glide 对象的方法，所以在 Glide 创建之后调用。

看看 GlideBuilder.build 方法：

```java
@NonNull
Glide build(@NonNull Context context) {
  if (sourceExecutor == null) {
    sourceExecutor = GlideExecutor.newSourceExecutor();
  }

  if (diskCacheExecutor == null) {
    diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
  }

  if (animationExecutor == null) {
    animationExecutor = GlideExecutor.newAnimationExecutor();
  }

  if (memorySizeCalculator == null) {
    memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
  }

  if (connectivityMonitorFactory == null) {
    connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
  }

  if (bitmapPool == null) {
    int size = memorySizeCalculator.getBitmapPoolSize();
    if (size > 0) {
      bitmapPool = new LruBitmapPool(size);
    } else {
      bitmapPool = new BitmapPoolAdapter();
    }
  }

  if (arrayPool == null) {
    arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
  }

  if (memoryCache == null) {
    memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
  }

  if (diskCacheFactory == null) {
    diskCacheFactory = new InternalCacheDiskCacheFactory(context);
  }

  if (engine == null) {
    engine =
        new Engine(
            memoryCache,
            diskCacheFactory,
            diskCacheExecutor,
            sourceExecutor,
            GlideExecutor.newUnlimitedSourceExecutor(),
            animationExecutor,
            isActiveResourceRetentionAllowed);
  }

  if (defaultRequestListeners == null) {
    defaultRequestListeners = Collections.emptyList();
  } else {
    defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
  }

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

可以看到，build 方法基本对每一个参数都进行了 null 判断，如果为 null 则使用默认的参数。那么，这些参数什么时候不为空呢？当在 AppliesOptions 接口的实现（即前面所说的 applyOptions 方法）中通过传入参数 GlideBuilder 设置后，这里 build 时就不会为 null 了。

比如，前面举例的通过 GlideBuilder.setDiskCache 来将 diskCacheFactory 设置为我们指定的值，从而将磁盘缓存位置切换到 externalCacheDir 中。

```kotlin
@GlideModule
class MyAppGlideModule : AppGlideModule() {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        builder.setDiskCache(ExternalPreferredCacheDiskCacheFactory(context))
    }
}
```

### kapt 生成文件过程分析

为了更好地理解 Glide 中 annotation processor 的作用，这里分别实现 AppGlideModule、LibraryGlideModule 以及 GlideModule ：

```kotlin
@GlideModule
@Excludes(value = [MyGlideModule::class])
class MyAppGlideModule : AppGlideModule()

@GlideModule
class MyLibraryGlideModule : LibraryGlideModule()

class MyGlideModule : GlideModule {
    override fun applyOptions(context: Context, builder: GlideBuilder) {}

    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {}
}
```

在 AndroidManifest 文件中配置好 MyGlideModule 之后，我们先 rebuild 一下，看一下生成的 GeneratedAppGlideModuleImpl 文件：

```java
@SuppressWarnings("deprecation")
final class GeneratedAppGlideModuleImpl extends GeneratedAppGlideModule {
  private final MyAppGlideModule appGlideModule;

  public GeneratedAppGlideModuleImpl(Context context) {
    appGlideModule = new MyAppGlideModule();
    if (Log.isLoggable("Glide", Log.DEBUG)) {
      Log.d("Glide", "Discovered AppGlideModule from annotation: com.me.guanpj.myapplication.MyAppGlideModule");
      Log.d("Glide", "Discovered LibraryGlideModule from annotation: com.me.guanpj.myapplication.MyLibraryGlideModule");
    }
  }

  @Override
  public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
    appGlideModule.applyOptions(context, builder);
  }

  @Override
  public void registerComponents(@NonNull Context context, @NonNull Glide glide,
      @NonNull Registry registry) {
    new MyLibraryGlideModule().registerComponents(context, glide, registry);
    appGlideModule.registerComponents(context, glide, registry);
  }

  @Override
  public boolean isManifestParsingEnabled() {
    return appGlideModule.isManifestParsingEnabled();
  }

  @Override
  @NonNull
  public Set<Class<?>> getExcludedModuleClasses() {
    Set<Class<?>> excludedClasses = new HashSet<Class<?>>();
    excludedClasses.add(com.me.guanpj.myapplication.MyGlideModule.class);
    return excludedClasses;
  }

  @Override
  @NonNull
  GeneratedRequestManagerFactory getRequestManagerFactory() {
    return new GeneratedRequestManagerFactory();
  }
}
```

代码很简单，属于 AppGlideModule 的三个方法基本都调用了 MyAppGlideModule 的三个配置方法。在此基础上，每个 LibraryGlideModule 的 registerComponents 方法都会在 GeneratedAppGlideModuleImpl.registerComponents 方法中被调用。

另外，我们关注一下最后两个方法，这两个方法都是基类 GeneratedAppGlideModule 额外提供了的，代码如下所示：

```java
abstract class GeneratedAppGlideModule extends AppGlideModule {
  /**
   * This method can be removed when manifest parsing is no longer supported.
   */
  @NonNull
  abstract Set<Class<?>> getExcludedModuleClasses();

  @Nullable
  RequestManagerRetriever.RequestManagerFactory getRequestManagerFactory() {
    return null;
  }
}
```

getRequestManagerFactory 在子类的实现是固定的，就是 return new GeneratedRequestManagerFactory()。

#### @Excludes

GeneratedAppGlideModuleImpl.getExcludedModuleClasses() 的实现，与 @Excludes 注解有关，使用该注解可以让 Glide 忽略指定的 GlideModule 或 LibraryGlideModule。@Excludes 注解只能用在 AppGlideModules 上，下面的例子将会让 Glide 忽略掉 MyLibraryGlideModule、MyGlideModule 的配置：

```scala
@GlideModule
@Excludes(value = [MyLibraryGlideModule::class, MyGlideModule::class])
class MyAppGlideModule : AppGlideModule()
```

此时 rebuild 之后，生成的 GeneratedAppGlideModuleImpl 文件的相关方法如下：

```typescript
@Override
public void registerComponents(@NonNull Context context, @NonNull Glide glide,
    @NonNull Registry registry) {
    // 注意，没有new MyLibraryGlideModule().registerComponents(context, glide, registry);了
    appGlideModule.registerComponents(context, glide, registry);
}

@Override
@NonNull
public Set<Class<?>> getExcludedModuleClasses() {
    Set<Class<?>> excludedClasses = new HashSet<Class<?>>();
    excludedClasses.add(com.me.guanpj.myapplication.MyGlideModule.class);
    excludedClasses.add(com.me.guanpj.myapplication. MyLibraryGlideModule.class);
    return excludedClasses;
}
```

然后在 Glide 初始化的时候，在方法返回结果 set 里面的 GlideModule 将会从集合中移除。

#### @GlideExtension

@GlideExtension 注解修饰的类可以扩展 Glide 的 API。该类必须是工具类的形式，里面的方法必须都是静态的，除了私有的空实现的构造器。

Application 可以实现多个 @GlideExtension 注解类，Library 也可以实现任意数量的 @GlideExtension 注解类。Glide 在编译时，一旦发现一个 AppGlideModule，所有可用的 GlideExtension 都会合并，并生成单个的 API 文件。任何冲突都会导致 Glide 注解生成器的编译错误。

GlideExtension 注解类可以定义两种扩展方法：

1. @GlideOption——为 RequestOptions 添加自定义的配置，扩展 RequestOptions 的静态方法。常见作用有：
2. 定义在整个应用程序中经常使用的一组选项
3. 添加新选项，通常与 Glide 的 com.bumptech.glide.load.Option 一起使用。
4. @GlideType——为新资源类型（GIFs、SVG 等）添加支持，扩展 RequestManager 的静态方法

下面的示例来自于官方文档 GlideExtension 的示例：

```kotlin
@GlideExtension
object MyAppExtension {
    // Size of mini thumb in pixels.
    private const val MINI_THUMB_SIZE = 100

    private val DECODE_TYPE_GIF = RequestOptions.decodeTypeOf(GifDrawable::class.java).lock()

    @GlideOption
    @JvmStatic
    fun miniThumb(options: BaseRequestOptions<*>): BaseRequestOptions<*> {
        return options
            .fitCenter()
            .override(MINI_THUMB_SIZE)
    }

    @GlideType(GifDrawable::class)
    @JvmStatic
    fun asGifTest(requestBuilder: RequestBuilder<GifDrawable>): RequestBuilder<GifDrawable> {
        return requestBuilder
            .transition(DrawableTransitionOptions())
            .apply(DECODE_TYPE_GIF)
    }
}
```

这里为 RequestOptions 扩展了 miniThumb 方法，为 RequestManager 扩展了 asGifTest 方法。所以我们可以这样使用：

```kotlin
GlideApp.with(this)
    .asGifTest()
    .load(URL)
    .miniThumb()
    .into(ivGlide1)
```

注意这里使用的不再是 Glide，而是 GlideApp。GlideApp 是专门用来处理这种扩展 API 的。

在 Glide 初始化的时候，会将 GeneratedAppGlideModuleImpl.getRequestManagerFactory() 方法返回的 GeneratedRequestManagerFactory 作为 requestManagerFactory 参数，这样创建 RequestManager 时都会调用 GeneratedRequestManagerFactory.build 方法生成 GlideRequests。

<strong>GlideRequests</strong> 继承至 RequestManager，里面包含了 @GlideType 注解修饰的 API：

```java
public class GlideRequests extends RequestManager {
  public GlideRequests(@NonNull Glide glide, @NonNull Lifecycle lifecycle,
                       @NonNull RequestManagerTreeNode treeNode, @NonNull Context context) {
    super(glide, lifecycle, treeNode, context);
  }
  ...
  /**
   * @see MyAppExtension#asGifTest(RequestBuilder)
   */
  @NonNull
  @CheckResult
  public GlideRequest<GifDrawable> asGifTest() {
    return (GlideRequest<GifDrawable>) MyAppExtension.asGifTest(this.as(GifDrawable.class));
  }
  ...
}
```

<strong>GlideRequest（注意没有 s）</strong> 则继承至 RequestBuilder，包含了 @GlideOption 提供的 API：

```scala
public class GlideRequest<TranscodeType> extends RequestBuilder<TranscodeType> implements Cloneable {
  /**
   * @see MyAppExtension#miniThumb(BaseRequestOptions)
   */
  @SuppressWarnings("unchecked")
  @CheckResult
  @NonNull
  public GlideRequest<TranscodeType> miniThumb() {
    return (GlideRequest<TranscodeType>) MyAppExtension.miniThumb(this);
  }
}
```

此外，如果需要使用到 RequestOptions，要使用 Generated API 生成的 GlideOptions。

总的来说，如果想使用 Generated API，注意一下三个类的关系

- RequestManager -> GlideRequests
- RequestBuilder -> GlideRequest
- RequestOptions -> GlideOptions

## 实战分析