# Glide 基本使用

# 准备

## 添加依赖

```plain text
implementation 'com.github.bumptech.glide:glide:4.12.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.12.0'
```

## 添加网路权限

```plain text
<uses-permission android:name="android.permission.INTERNET" />
```

## 定义控件

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="load"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        android:onClick="load"/>

    <ImageView
        android:id="@+id/image"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:src="@mipmap/ic_launcher"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

# 加载图片

添加按钮监听，使用 Glide 加载网路图片：

```java
public class MainActivity extends AppCompatActivity {
    ImageView imageView;
    private String url = "https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fimg.zcool.cn%2Fcommunity%2F01229b55427cae0000019ae90743f7.jpg&refer=http%3A%2F%2Fimg.zcool.cn&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1634203057&t=ddd9257730ce04906923aaa088f0b226";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        imageView = findViewById(R.id.image);
    }

    public void load(View view) {
        Glide.with(this).load(url).into(imageView);
    }
    
    @Override
    protected void onDestroy() {
      Glide.with(this).clear(imageView);
      super.onDestroy();
    }
}
```

这样，就是 Glide 最基本的用法。 点击按钮之后，Glide 会把图片加载进 ImageView 中，并在 onDestory 方法中取消 Glide 加载。尽管及时取消不必要的加载是很好的实践，但这并不是必须的操作。实际上，当 Glide.with() 中传入的 Activity 实例销毁时，Glide 会自动取消加载并回收资源。

在 ListView 或 RecyclerView 中加载图片的代码和在单独的 View 中加载完全一样。Glide 已经自动处理了 View 的复用和请求的取消。

## with

Glide.with() 方法有很多重载：

- `with(@NonNull Context context)`
- `with(@NonNull View view)`
- `with(@NonNull Activity activity)`
- `with(@NonNull FragmentActivity activity)`
- `with(@NonNull Fragment fragment)`

在上面的重载方法中，除了前两个重载方法外，其他三个都有很直观的生命周期；至于前两个，会尝试绑定到 Activity 或 Fragment 上面，如果失败了就会绑定到 Application 级别的生命周期上。

Glide.with() 方法返回的是 RequestManager 实例。

## load

load 方法也有很多重载：

- `load(@Nullable Bitmap bitmap)`
- `load(@Nullable Drawable drawable)`
- `load(@Nullable String string)`
- `load(@Nullable Uri uri)`
- `load(@Nullable File file)`
- `load(@RawRes @DrawableRes @Nullable Integer resourceId)`
- `load(@Nullable byte[] model)`
- `load(@Nullable Object model)`

RequestManager 除了上面的方法外，还有其他一些有用的方法：

控制方法：

- `isPaused()`
- `pauseRequests()`
- `pauseAllRequests()`
- `pauseRequestsRecursive()`
- `resumeRequests()`
- `resumeRequestsRecursive()`
- `clear(@NonNull View view)`
- `clear(@Nullable final Target<?> target)`

生命周期方法：

- `onStart()`
- `onStop()`
- `onDestroy()`

其他方法：

- `downloadOnly()`
- `download(@Nullable Object model)`
- `asBitmap()`
- `asGif()`
- `asDrawable()`
- `asFile()`
- `as(@NonNull Class<ResourceType> resourceClass)`

RequestManager.load() 方法返回了一个 RequestBuilder 对象，调用该对象的 into(@NonNull ImageView view) 方法就完成了 Glide 加载的三步。当然此方法还有一些高级的重载方法，我们后面在说。

此外，上面提到的 RequestManager 的 7 个其他方法也都会返回一个 RequestBuilder 对象，而此时还没有设置要加载的资源，所以 RequestBuilder 也提供了很多 load 方法来设置要加载资源。

# 占位图

占位符类型有三种，分别在三种不同场景使用：

- placeholder

placeholder 是当请求正在执行时被展示的 Drawables。当请求成功完成时，placehodler 会被请求到的资源替换。如果被请求的资源是从内存中加载出来的，那么 placehodler 不会显示。如果请求失败并且没有设置 error，则 placehodler 将被持续展示。类似地，如果请求的 url/model 为 null，并且 error 和 fallback 都没有设置，那么 placehodler 也会继续显示。

- error

error 在请求永久性失败时展示。error 同样也在请求的 url/model 为 null，且并没有设置 fallback 时展示。

- fallback

fallback 在请求的 url/model 为 null 时展示。设计 fallback 的主要目的是允许用户指示 null 是否为可接受的正常情况。例如，一个 null 的个人资料 url 可能暗示这个用户没有设置头像，因此应该使用默认头像。然而，null 也可能表明这个元数据根本就是不合法的，或者取不到。<strong>默认情况下 Glide 将 null 作为错误处理</strong>，所以可以接受 null 的应用应当显式地设置一个 fallback。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-1/clipboard_20230323_032742.png)

占位图显示逻辑的代码如下：

```java
private synchronized void setErrorPlaceholder() {
  if (!canNotifyStatusChanged()) {
    return;
  }

  Drawable error = null;
  if (model == null) {
    error = getFallbackDrawable();
  }
  // Either the model isn't null, or there was no fallback drawable set.
  if (error == null) {
    error = getErrorDrawable();
  }
  // The model isn't null, no fallback drawable was set or no error drawable was set.
  if (error == null) {
    error = getPlaceholderDrawable();
  }
  target.onLoadFailed(error);
}
```

我们准备使用这段代码演示一下。

注意，为了忽略缓存的影响，这里设置了忽略内存缓存 `skipMemoryCache(true)` 并将磁盘缓存策略设置为不缓存 `diskCacheStrategy(DiskCacheStrategy.NONE)`。

```kotlin
val option = RequestOptions()
    .placeholder(ColorDrawable(Color.GRAY))
    .error(ColorDrawable(Color.RED))
    .fallback(ColorDrawable(Color.CYAN))
    .skipMemoryCache(true)
    .diskCacheStrategy(DiskCacheStrategy.NONE)

Glide.with(this)
    .load(URL)
    .apply(option)
    .into(ivGlide)
```

上面的代码也可以这么写：

```kotlin
Glide.with(this)
    .load(URL)
    .placeholder(ColorDrawable(Color.GRAY))
    .error(ColorDrawable(Color.RED))
    .fallback(ColorDrawable(Color.CYAN))
    .skipMemoryCache(true)
    .diskCacheStrategy(DiskCacheStrategy.NONE)
    .into(ivGlide)
```

使用 RequestOptions() 更方便，因为可以多个 Glide 加载语句共用这些通用设置。

下面展示了正确加载时、加载字符空串时的图：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-1/glide-placeholder-success-example.gif)

<div style="text-align: center"><em>Glide正确加载</em></div>

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-1/glide-error-example.gif)

<div style="text-align: center"><em>Glide加载空串</em></div>

<strong>1. 占位符是异步加载的吗？</strong>

No。占位符是在主线程从 Android Resources 加载的。我们通常希望占位符比较小且容易被系统资源缓存机制缓存起来。

<strong>2. 变换是否会被应用到占位符上？</strong>

No。Transformation 仅被应用于被请求的资源，而不会对任何占位符使用。

在应用中包含必须在运行时做变换才能使用的图片资源是很不划算的。相反，在应用中包含一个确切符合尺寸和形状要求的资源版本几乎总是一个更好的办法。假如你正在加载圆形图片，你可能希望在你的应用中包含圆形的占位符。另外你也可以考虑自定义一个 View 来剪裁 (clip) 你的占位符，而达到你想要的变换效果。

<strong>3. 在多个不同的 View 上使用相同的 Drawable 可行么？</strong>

通常可以，但不是绝对的。任何无状态 (non-stateful )的 Drawable（例如 BitmapDrawable）通常都是 ok 的。但是有状态的 Drawable 不一样，在同一时间多个 View 上展示它们通常不是很安全，因为多个 View 会立刻修改 (mutate) Drawable。对于有状态的 Drawable，建议传入一个资源 ID，或者使用 newDrawable() 来给每个请求传入一个新的拷贝。

# 指定图片格式

Glide 支持加载 GIF，Picasso 不支持。而且 Glide 加载 GIF 不需要额外的代码，其内部会判断图片格式。

我们可以直接使用下面的示例代码加载 GIF：

```kotlin
val option = RequestOptions()
    .placeholder(ColorDrawable(Color.GRAY))
    .error(ColorDrawable(Color.RED))
    .fallback(ColorDrawable(Color.CYAN))
    .skipMemoryCache(true)
    .diskCacheStrategy(DiskCacheStrategy.NONE)

Glide.with(this)
    .load(GIF_URL)
    .apply(option)
    .into(ivGlide)
```

运行结果如下图所示：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-1/glide-load-gif.gif)

<div style="text-align: center"><em>Glide加载GIF</em></div>

现在我们只想加载静态图片，我们可以在 Glide.with 后面追加 asBitmap() 方法实现：

```kotlin
Glide.with(this)
    .asBitmap()
    .load(GIF_URL)
    .apply(option)
    .into(ivGlide)
```

由于调用了 asBitmap() 方法，现在 GIF 图就无法正常播放了，而是会在界面上显示第一帧的图片。

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-1/glide-load-gif-with-asbitmap.gif)

<div style="text-align: center"><em>Glide asBitmap加载GIF</em></div>

同理，我们在加载普通图片时追加 asGif() 会怎么样呢：

```kotlin
Glide.with(this)
    .asGif()
    .load(URL)
    .apply(option)
    .into(ivGlide)
```

很不幸，显示加载错误图片：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-1/glide-load-url-with-asgif.gif)

<div style="text-align: center"><em>Glide asGif加载普通图片</em></div>

# 指定图片大小

实际上，使用 Glide 在绝大多数情况下我们都是不需要指定图片大小的。

在学习本节内容之前，你可能还需要先了解一个概念，就是我们平时在加载图片的时候很容易会造成内存浪费。什么叫内存浪费呢？比如说一张图片的尺寸是 1000*1000 像素，但是我们界面上的 ImageView 可能只有 200*200 像素，这个时候如果你不对图片进行任何压缩就直接读取到内存中，这就属于内存浪费了，因为程序中根本就用不到这么高像素的图片。

而使用 Glide，我们就完全不用担心图片内存浪费，甚至是内存溢出的问题。因为 Glide 从来都不会直接将图片的完整尺寸全部加载到内存中，而是用多少加载多少。Glide 会自动判断 ImageView 的大小，然后只将这么大的图片像素加载到内存当中，帮助我们节省内存开支。

也正是因为 Glide 是如此的智能，所以刚才在开始的时候我就说了，在绝大多数情况下我们都是不需要指定图片大小的，因为 Glide 会自动根据 ImageView 的大小来决定图片的大小。

不过，如果你真的有这样的需求，必须给图片指定一个固定的大小，Glide 仍然是支持这个功能的。修改 Glide 加载部分的代码，如下所示：

```kotlin
Glide.with(this)
    .load(URL)
    .apply(option)
    .override(100)
    .into(ivGlide)
```

仍然非常简单，这里使用 override() 方法指定了一个图片的尺寸，也就是说，Glide 现在只会将图片加载成 100*100 像素的尺寸，而不会管你的 ImageView 的大小是多少了。

对比图如下所示：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Glide-1/clipboard_20230323_033342.png)

<div style="text-align: center"><em>override加载前后对比</em></div>
