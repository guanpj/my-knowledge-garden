# 六、View 绘制流程分析

在我的系列文章上一篇：[App 竟然是这样跑起来的 —— Android App/Activity 启动流程分析](https://guanpj.cn/2017/10/23/Android-App-Startup-Flow-Analyze/)中已经分析了一个 App 从点击它的图标到 Activity 的 onCreate()、onStart() 和 onResume() 等生命周期被调用的整个流程。我们都知道，普通 App 屏幕上显示的内容都是由一个个自己设计的界面被系统加载而来的，而这些界面中的元素又是怎么被渲染出来的呢？本文将继续基于 Android Nougat 从源码的角度来进一步分析整个过程。

在开始之前，回顾一下上一篇文章中分析的从 ActivityThread 到 Activity 过程的时序图：

![](static/boxcn8eZqN1LvM9KIKY3R4mfwde.)

## 步骤一：初始化 PhoneWindow 和 WindowManager

如上图所示，在 Activity 的 onCreate()、onStart() 和 onResume() 等生命周期被调用之前，它的 attach() 方法将会先被调用，因此，我们将 attach() 方法作为这篇文章主线的开头：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window) {
    attachBaseContext(context);
    ...
    // mWindow 是一个 PhoneWindow 对象实例
    mWindow = new PhoneWindow(this, window);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ...
    // 调用 Window 的 setWindowManager 方法
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    // 从 Window 中获取 WindowManager
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
}
```

mWindow 是一个 Window 类型的变量，在 attach() 方法中，创建了一个 PhoneWindow 对象实例并赋值给了 mWindow，PhoneWindow 直接继承自 Window 类。然后调用了 Window 的 setWindowManager() 方法：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    ...
    // mWindowManager 就是 WindowManagerImpl 对象的实例
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

因此，Acitvity 中的 mWindow 变量就是 PhoneWindow 类的实例，而 mWindowManager 是 WindowManagerImpl 类的实例，attach() 方法的主要工作就是初始化这两个变量。

## 步骤二：初始化 DecorView

### 源码分析

接下来到了 onCreate 方法，我们都知道，如果想要让自己设计的 layout 布局文件或者 View 显示在 Activity 中，必须要在 Activity 的 onCreate() 方法中应该调用 setContentView() 方法将我们的布局 id 或者 View 传递过去，查看其中一个 setContentView() 方法：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

继续查看 PhoneWindow 类的 setContentView() 方法：

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {// 是否首次调用
        // 初始化 Decor
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {// 转场动画，默认 false
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        ...
    } else {
        // 解析布局文件
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    ...
}
```

如果是首次调用这个方法，则 mContentParent 为 null，否则如果没有转场动画的话就移除 mContentParent 的全部子 View，继续跟踪 installDecor() 方法：

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        // 生成 DecorView 对象
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        // 调用 generateLayout 方法
        mContentParent = generateLayout(mDecor);
        ...
    }
}
```

当 mDecor 为 null 的时候会调用 generateDecor() 方法创建一个 DecorView 类的实例，DecorView 继承自 FrameLayout。接下来判断 mContentParent 是否为 null（前面已经提到过，首次加载的时候就是 null），如果是则调用 generateLayout() 方法，这个方法就会创建 mContentParent 对象，跟踪进去：

```java
protected ViewGroup generateLayout(DecorView decor) {
    TypedArray a = getWindowStyle();
    ...
    // 通过 WindowStyle 中设置的各种属性对 Window 进行各种初始化操作
    mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
    int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
            & (~getForcedWindowFlags());
    if (mIsFloating) {
        setLayout(WRAP_CONTENT, WRAP_CONTENT);
        setFlags(0, flagsToUpdate);
    } else {
        setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
    }
    ....

    int layoutResource;
    int features = getLocalFeatures();

    // 根据设定好的 features 值获取相应的布局文件并赋值给 layoutResource
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleIconsDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
            && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        layoutResource = R.layout.screen_progress;
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogCustomTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_custom_title;
        }
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                    R.styleable.Window_windowActionBarFullscreenDecorLayout,
                    R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        layoutResource = R.layout.screen_simple;
    }

    mDecor.startChanging();
    // 调用 onResourcesLoaded 方法
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    // 在 layoutResource 中根据 id:com.android.internal.R.id.content 获取一个 ViewGroup 并赋值给 contentParent  对象
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }
    ...
    mDecor.finishChanging();
    // 返回 contentParent
    return contentParent;
}
```

DecorView 的 onResourcesLoaded() 方法：

```java
void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
    ...
    mDecorCaptionView = createDecorCaptionView(inflater);
    // 解析 layoutResource 文件
    final View root = inflater.inflate(layoutResource, null);
    if (mDecorCaptionView != null) {
        ...
    } else {
        // 作为根布局添加到 mDecor 中
        addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }
    mContentRoot = (ViewGroup) root;
    initializeElevation();
}
```

可以看到，generateLayout() 方法主要分为四个部分：

1. 首先会通过 com.android.internal.R.styleable.Window 中设置的各种属性对 Window 进行各 requestFeature 或者 setFlags 等操作。
2. 接下来是根据设定好的 features 值选择不同的窗口修饰布局文件，得到 layoutResource 值。

> 因此在自己的 Activity 中设置全屏的 requestFeature() 等方法也需要在 setContentView 之前调用，才能够根据你的设置来选择不同的根布局。

3. 将 layoutResource 值传给 DecorView 的 onResourcesLoaded() 方法，通过 LayoutInflater 把布局转化成 View 作为根视图并将其添加到 mDecor。
4. 在 mDecor 中查找 id 为 com.android.internal.R.id.content 的 ViewGroup 并作为返回值返回，这个 ViewGroup 一般为 FrameLayout。

关于第四点，可能有人会有疑问，为什么根据 id 为 com.android.internal.R.id.content 就一定能找到对应的 ViewGroup？答案就在前面我们分析过的 generateLayout() 方法中，这里会根据设定好的 features 值获取相应的布局文件并赋值给 layoutResource，而这所有的布局文件中都包括了一个 id 为 content 的 FrameLayout，除此之外有些布局文件中还可能有 ActiionBar 和 Title 等，这些布局文件存放于[该目录下]([https://android.googlesource.com/platform/frameworks/base/](https://android.googlesource.com/platform/frameworks/base/) /nougat-release/core/res/res/layout/)。以 R.layout.screen_simple 为例，它的内容如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
          android:inflatedId="@+id/action_mode_bar"
          android:layout="@layout/action_mode_bar"
          android:layout_width="match_parent"
          android:layout_height="wrap_content"
          android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

回到 PhoneWindow 的 setContentView() 方法，执行完 installDecor() 之后，mDecor 被初始化了，同时 mContentParent 也被赋了值， 回到 setContentView() 方法，最后一个重要步骤就是通过 mLayoutInflater.inflater 将我们的 layout 布局文件压入 mDecor 中 id 为 content 的 FrameLayout 中。

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {// 是否首次调用
        // 初始化 Decor
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {// 转场动画，默认 false
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        ...
    } else {
        // 解析布局文件
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    ...
}
```

### 小结

至此，setContentView() 方法的流程就走完了，总体来看分为三个步骤：

1. 初始化 mDecor，它是 DecorView 类的实例，DecorView 继承自 FrameLayout；
2. 根据 theme 中的属性值，选择相应的布局文件并通过 infalter.inflater() 方法将它加载出来并 add 到 mDecor 中；这些布局文件都有一个 id 为 content 的 FrameLayout；
3. 在 Activity 的 setContentView() 方法中设置的 layout 布局文件，会通过 mLayoutInflater.inflater() 压入 mDecor 中 id 为 content 的 FrameLayout 中。

这个过程的时序图如下：

![](static/boxcn9HmhHqsLD6wQwcboR4VGsf.)

Activity、PhoneWindow、DecorView 和 ContentView 的关系如下图所示：

![](static/boxcn7rIKlhR936mPuakAn6LGrc.)

但是，此时我们的布局还没有显示出来，接着往下看。

## 步骤三：初始化 ViewRootImpl 并关联 DecorView

### 源码分析

在开篇的时序图中我们可以看到，ActivityThread 在 handleLaunchActivity() 方法中的 performLaunchActivity() 操作间接调用了 Activity 的 attach() 和 onCreate()：

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    WindowManagerGlobal.initialize();
    // 执行 performLaunchActivity 方法
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        // 执行 handleResumeActivity 方法，最终调用 onStart 和 onResume 方法
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);

        if (!r.activity.mFinished && r.startsNotResumed) {
            r.activity.mCalled = false;
            mInstrumentation.callActivityOnPause(r.activity);
            r.paused = true;
        }
    } else {
        // 停止该 Activity
        ActivityManagerNative.getDefault()
            .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
    }
}
```

接着会调用 handleResumeActivity() 方法：

```java
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    ...
    // 最终会调用 onStart() 和 onResume() 等方法
    r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;
        ...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            // 获取 DecorView 
            View decor = r.window.getDecorView();
            // 将 DecorView 设置成不可见
            decor.setVisibility(View.INVISIBLE);
            // 获取 ViewManager，这里是 WindowManagerImpl 实例
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            ...
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                // 标记设置为 true
                a.mWindowAdded = true;
                // 调用 WindowManagerImpl 的 addView 方法
                wm.addView(decor, l);
            } 
        } else if (!willBeVisible) {
            ...
        }
        ...
        if (!r.activity.mFinished && willBeVisible
                && r.activity.mDecor != null && !r.hideForNow) {
            ...
            if (r.activity.mVisibleFromClient) {
                // 调用 makeVisible 方法将 DecorView 设置为可见
                r.activity.makeVisible();
            }
        }
        ...
    } else {
        try {
            // 在此过程出现异常，则直接杀死 Activity
            ActivityManagerNative.getDefault()
                .finishActivity(token, Activity.RESULT_CANCELED, null,
                        Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

执行完 performResumeActivity() 方法之后，接着会取出 ActivityClientRecord 中的 Activity 对象，并得到之前在 setContentView 流程中初始化好的 DecorView 对象，然后会将它作为参数传入 ViewManager 类型的对象 wm 的 addView 方法，ViewManager 是一个接口，那么它是由谁来实现的呢？

其实这就是文章开头部分提到过的，在 Activity 的 attach() 方法中，会初始化 PhoneWindow 和 WindowManager 对象，而这个 WindowManager 对象则是由 WindowManagerImpl 来实现的。

所以，Activity 的 getWindowManager() 此时获取到的就是 WindowManagerImpl 对象的实例，再看它的 addView() 方法：

```java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    // mGlobal 是 WindowManagerGlobal 对象的实例
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

这里将任务委托给了 mGlobal ，而它又是 WindowManagerGlobal 对象的实例，查看它的 addView() 方法：

[core/java/android/view/WindowManagerGlobal.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/nougat-release/core/java/android/view/WindowManagerGlobal.java)

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ...
    
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        // If there's no parent, then hardware acceleration for this view is
        // set from the application's hardware acceleration setting.
        final Context context = view.getContext();
        if (context != null
               && (context.getApplicationInfo().flags
                    & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
        
    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        ...
        // 创建 ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }

    try {
        // 将传过来的 DecorView 添加到 ViewRootImpl 中
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        ...
    }
 }
```

上面的变量中，mViews 存储的是所有 Window 对应的 View，mRoots 存储的是所有 Window 对应的 ViewRootImpl 对象，mParams 存储的是所有 Window 对应的布局参数。

### 小结

可以看到，ViewRootImpl 是 DecorView 的管理者，它负责 View Tree 的测量、布局和绘制，以及后面会说到的通过 Choreographer 来控制 View Tree 的刷新操作。

## 步骤四：建立 PhoneWindow 和 WindowManagerService 之间的连接

WMS 是所有 Window 窗口的管理员，负责 Window 的添加和删除、Surface 的管理和事件的派发等等，因此每一个 Activity 中的 PhoneWindow 对象如果需要显示等操作，就必须通过与 WMS 的交互才能进行。

### 源码分析

接着上一个步骤的流程，查看 ViewRootImpl 的 setView 方法：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            ...
            // 1
            requestLayout();
            ...
            try {
                ...
                // 2
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                      getHostVisibility(), mDisplay.getDisplayId(),
                      mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                      mAttachInfo.mOutsets, mInputChannel);
            } catch (RemoteException e) {
                ...
            } finally {
                if (restore) {
                    attrs.restore();
                }
            }
            ...
            view.assignParent(this);
            ...
        }
    }
}
```

忽略 requestLayout 方法，先看注释 2 处：这里调用了 mWindowSession.addToDisplay() 方法来完成最终的添加流程，mWindowSession 为 IWindowSession 的实例，而 IWindowSession 是一个 Binder 的 Client 代理对象，对应的 Server 端的实现为 Session 类。在这之前代码都是运行在 app 进程的，而 Session 则是运行在 WMS 所在的进程（即 SystemServer 进程）中。

如此，往 Window 中添加 View 的流程就交给 WindowManagerService 去处理了，app 进程的 ViewRootImpl 要想和 WMS 通讯则需要借助 Binder 机制并通过 Session 来进行操作。Session 的 addToDisplay() 方法如下：

[services/core/java/com/android/server/wm/Session.java]([https://android.googlesource.com/platform/frameworks/base/](https://android.googlesource.com/platform/frameworks/base/) /refs/heads/nougat-release/services/core/java/com/android/server/wm/Session.java)

```java
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
        Rect outOutsets, InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```

可以看到，这里调用了 WMS 的 addWindow() 方法并将自身作为参数传了进去，每个应用程序都会对应一个 Session，WMS 会用一个 List 来保存这些 Session。接着 WMS 会为这个添加的窗口分配 Surface，并确定窗口显示次序，因此负责显示界面的是画布 Surface，而不是窗口本身。WMS 会将它管理的 Surface 交由 SurfaceFlinger 处理，SurfaceFlinger 会将这些 Surface 混合并绘制到屏幕上。

关于 WMS 的详细分析会在后续的文章中进行。

这个是一个典型的 Binder 双向通讯模型，Binder 机制的文章可参考 [借助 AIDL 理解 Android Binder 机制](https://www.jianshu.com/p/73e351d25773)。这个过程如下图所示：

![](static/boxcnArtDBCR0JvJLPZehxmirSY.png)

### 小结

用一张图总结 Activity、Window 和 WMS 之间的关系：

![](static/boxcn4h7pNRYuGoHKXKfOrRDVkf.png)

## 步骤五：建立与 SurfaceFlinger 的连接

SurfaceFlinger 是 Android 最重要的系统服务之一，它的职责主要是进行 Layer(Surface 在 Native 叫法) 的合成和渲染。

### 源码分析

接上面的步骤，WindowState 的 attach() 方法将会调用到 Session 的 windowAddedLocked() 方法：

[services/core/java/com/android/server/wm/WindowState.java]([https://android.googlesource.com/platform/frameworks/base/](https://android.googlesource.com/platform/frameworks/base/) /refs/heads/nougat-release/services/core/java/com/android/server/wm/WindowState.java)

```java
void attach() {
    if (WindowManagerService.localLOGV) Slog.v(
        TAG, "Attaching " + this + " token=" + mToken
        + ", list=" + mToken.windows);
    mSession.windowAddedLocked();
}
```

Session.java

```java
void windowAddedLocked(String packageName) {
    ...
    if (mSurfaceSession == null) { 
        ...
        mSurfaceSession = new SurfaceSession();
        ...
    }
}

public final class SurfaceSession {
    private long mNativeClient; // SurfaceComposerClient*
    private static native long nativeCreate();
    ...

    public SurfaceSession() {
        mNativeClient = nativeCreate(); 
    }
    
    ...
}
```

nativeCreate() 方法是一个 native 方法：

[core/jni/android_view_SurfaceSession.cpp]([https://android.googlesource.com/platform/frameworks/base/](https://android.googlesource.com/platform/frameworks/base/) /refs/heads/nougat-release/core/jni/android_view_SurfaceSession.cpp)

```c++
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}
```

nativeCreate 方法主要构造了一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁，SurfaceComposerClient 指针在第一次使用的时候会调用以下方法：

```c++
void SurfaceComposerClient::onFirstRef() {
    ....
    sp<ISurfaceComposerClient> conn;
    // sf 就是 SurfaceFlinger 对象指针
    conn = (rootProducer != nullptr) ? sf->createScopedConnection(rootProducer) :
            sf->createConnection();
    ...
}
```

它通过 SurfaceFlinger 的 createScopedConnection 方法创建了一个 ISurfaceComposerClient 对象：

```c++
SurfaceFlinger.cpp

sp<ISurfaceComposerClient> SurfaceFlinger::createConnection() {
    return initClient(new Client(this)); 
}

static sp<ISurfaceComposerClient> initClient(const sp<Client>& client) {
    status_t err = client->initCheck();
    // 只是做了错误检查，然后返回 client 本身
    if (err == NO_ERROR) {
        return client;
    }
    return nullptr;
}
```

Client 类实现了 ISurfaceComposerClient(继承了 IInterface) 接口，因此它可以进行跨进程通信，SurfaceComposerClient 就是通过它来和 SurfaceFlinger 通信。除此之外它还可以创建 Surface，并且维护一个应用程序的所有 Layer。

initClicent 方法做了一些错误检查，然后返回 Client 本身。

### 小结

这个过程如下图：

![](static/boxcnyz8oHMf8mfrLJYCs1e9hlf.png)

## 步骤六：申请 Surface

经过步骤四和步骤五，ViewRootImpl 与 WMS、SurfaceFlinger 都已经建立连接，但是这时 View 还是没有显示出来。我们都知道，所有的 UI 最终都是要通过 Surface 来显示的，那么 Surface 是什么时候创建的呢？回到步骤四开头部分 ViewRootImpl 的 setView 方法：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            ...
            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            requestLayout();
            ...
            try {
                ...
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(),
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mOutsets, mInputChannel);
            } 
            ...
        }
    }
}
```

注释处保留了官方原文，意思大概是 WMS 除了窗口管理，还需要处理事件派发，因此在与 WMS 进行关联之前，要确保 View Tree 已经进行了 layout 操作，以接收来自 WMS 的事件。

跟踪 ViewRootImpl 的 requestLayout() 方法：

```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

scheduleTraversals() 方法如下：

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 设置同步障碍，暂停处理后面的同步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 在下一帧到来的时候执行 mTraversalRunnable
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}
```

首先，设置同步障碍，暂停处理后面的同步消息，然后利用 Choreographer 类在下一绘制帧来临的时候执行 mTraversalRunnable 对象（关于 Choreographer 原理，可查看 [Android 系统 Choreographer 机制实现过程](https://blog.csdn.net/yangwen123/article/details/39518923)）。

简单地说，Choreographer 内部会接收来自 SurfaceFlinger 发出的 VSync 垂直同步信号，这个信号周期一般为 16ms 左右。SurfaceFlinger 是由 init 进程启动的运行在底层的一个系统进程，它的主要职责是合成和渲染 Surface(Layer)，并向目标进程发送垂直同步信号 VSync。因此，如果需要接收 Vsync 信号，必须先与 SurfaceFlinger 建立连接，而这正是先走步骤五流程的原因。

mTraversalRunnable 是一个 Runnable 对象：

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

run() 方法里面只有一句代码，doTraversal() 方法如下：

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 移除同步障碍
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...
        // 正式进入 View 绘制流程
        performTraversals();
         ...
    }
}
```

移除了同步障碍之后，所有绘制之前的准备工作已经执行完毕，接下来会调用 performTraversals() 方法正式进入 View 的绘制流程。

```java
private void performTraversals() {
    finalView host = mView; // mView 其实就是 DecorView
    ...
    relayoutWindow(params, viewVisibility, insetsPending);
    ...
    // 执行 Measure 流程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
    // 执行 Layout 流程
    performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    ...
    // 执行 Draw 流程
    performLayout();
    ...
}
```

跟踪 relayoutWindow 方法：

```java
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
        boolean insetsPending) throws RemoteException {
    ...
    int relayoutResult = mWindowSession.relayout(
            mWindow, mSeq, params,
            (int) (mView.getMeasuredWidth() * appScale + 0.5f),
            (int) (mView.getMeasuredHeight() * appScale + 0.5f),
            viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
            mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
            mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingConfiguration,
            mSurface);
    ...
}
```

这里又是 Binder 调用（强烈建议掌握），它将调用到 WMS 的 relayout 方法，注意最后一个参数就是 SurfaceView 对象，它在 ViewRootImpl 中定义的时候就已经进行了初始化：

final Surface mSurface = new Surface();

再来看 WMS 的 relayoutWindow：

```java
public int relayoutWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int requestedWidth,
            int requestedHeight, int viewVisibility, int flags,
            Rect outFrame, Rect outOverscanInsets, Rect outContentInsets,
            Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame,
            Configuration outConfig, Surface outSurface){ 
    ...
    result = createSurfaceControl(outSurface, result, win, winAnimator);  
    ...
}

private int createSurfaceControl(Surface outSurface, int result, WindowState win,WindowStateAnimator winAnimator) {
    ...
    surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
    ...
    surfaceController.getSurface(outSurface);
}
```

这里首先调用 WindowStateAnimator 的 createSurfaceLocked 生成一个真正有效的 Surface 对象，这个对象是在 Native 层的，接着 getSurface 方法将 Java 层的 Surface 对象与其关联起来。

## 步骤七：正式绘制 View

接上一步骤的内容，申请了 Surface 之后，ViewRootImpl 的 performTraversals() 方法将会继续执行，这个方法内容相当多，忽略条件判断，精简过后如下：

```java
private void performTraversals() {
    ...
    relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
    ...
    WindowManager.LayoutParams lp = mWindowAttributes;
    ...
    // 获取 DecorView 宽和高的 MeasureSpec
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    ...
    // 执行 Measure 流程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
    // 执行 Layout 流程
    performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    ...
    // 执行 Draw 流程
    performLayout();
    ...
}
```

首先，根据 getRootMeasureSpec() 方法获取到 childWidthMeasureSpec 和 childHeightMeasureSpec 的值，用于 DecorView 的绘制。因为 DecorView 是所有子元素的根元素，子元素的布局层层嵌套，因此会接着从 DecorView 开始进行一层层地对所有子元素进行测量、布局和绘制，分别对应 performMeasure()、performLayout() 和 performLayout() 方法，整个过程的示意图如下：

![](static/boxcnTwYkJnUXwTHzXoDsDXqmhc.)

### 理解 MeasureSpec

MeasureSpec 是 View 的一个内部类，简单来说就是一个 32 位的 int 值，采用它的高 2 位表示三种 SpecMode（测量模式），低 30 位用来表示 SpecSize（某种测量模式下的规格大小）。采用这种表示方法是为了避免创建过多的对象以减少内存分配，MeasureSpec 的定义如下：

```java
public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK = 0x3 << MODE_SHIFT;

    // 不限定测量模式：父容器不对 View 作任何限制，View 想要多大给多大，
    // 这种模式通常用于系统内部。
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;

    // 精确为 SpecSize：父容器已经确定 View 需要的大小，就是 SpecSize，
    // 对应布局参数是 match_parent 或具体数值时的情况
    public static final int EXACTLY = 1 << MODE_SHIFT;

    // 最大只能是 SpecSize：父容器规定 View 最大只能是 SpecSize，
    // 对应布局参数是 wrap_content 时的情况
    public static final int AT_MOST = 2 << MODE_SHIFT;

    // 根据 SpecMode 和 SpecSize 创建一个 MeasureSpec
    public static int makeMeasureSpec(int size, int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }
    
    // 获取 SpecMode
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }

    // 获取 SpecSize
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }

    // 调整 MeasureSpec
    static int adjust(int measureSpec, int delta) {
        final int mode = getMode(measureSpec);
        if (mode == UNSPECIFIED) {
            return make MeasureSpec(0, UNSPECIFIED);
        }
        int size = getSize(measureSpec) + delta;
        if (size < 0) {
            size = 0;
        }
        return makeMeasureSpec(size, mode);
    }
}
```

接着看 getRootMeasureSpec() 方法，传入的第一个参数是整个屏幕的宽或者高，第二个参数是 Window 的 LayoutParams：

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

从代码逻辑来看，DecorView 的 MeasureSpec 的产生遵循如下规律：

- LayoutParams = MATCH_PARENT：精确模式，大小就是屏幕的宽或者高；
- LayoutParams = WRAP_CONTENT：最大不能超过屏幕的宽或者高；
- 固定大小：精确模式，大小为 LayoutParams 中指定的值。

但是对于普通的 View 来说，View 的 measure() 方法是由父容器 ViewGroup 调用的，看一下 ViewGroup 的 measureChildWithMargins() 方法：

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    // 获取子元素的布局参数
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    // 产生子元素的 MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

可以看出，在调用子元素 的 measure() 方法之前，先要调用 getChildMeasureSpec() 方法产生子元素的 MeasureSpec，子元素的产生除了跟父容器的 MeasureSpec 和子元素本身的 LayoutParams 有关之外，还与子元素的 Margin 和父容器的 Padding 值以及父容器当前占用空间有关，具体的过程可以看 getChildMeasureSpec() 方法：

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimesion) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    // 子元素最大可用空间为父容器的尺寸减去父容器中已被占用的空间的大小
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (sepcMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimesion == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us 
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size....
                // find out how big it should be
                resultSize = 0;
                resultMode == MeasureSpec.UNSPECIFIED;
            }
            break;
        }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

整个过程稍微有点复杂，可以参考以下表格：

![](static/boxcn1rakNHuBCAijUF5Upqyylf.)

### Measure 流程分析

回到 performTraversals() 方法中，获取到 DecorView 的 MeasureSpec 后接着会调用 performMeasure() 方法：

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    ...
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
}
```

mView 就是之前通过 setView() 方法传递过来的 DecorView 实例，它继承自 FrameLayout，而 FrameLayout 又是一个 ViewGroup 同时继承自 View。View 的 measure() 方法是 final 类型的，不允许子类去重写，因此这里调用的实际上是 View 的 measure 方法：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    onMeasure(widthMeasureSpec, heightMeasureSpec);
    ...
}
```

View 的 onMeasure() 实现：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

可以看到，View 默认的 onMeasure() 方法首先会调用 getDefaultSize() 获取宽和高的默认值，然后再调用 setMeasuredDimension() 将获取的值进行设置，查看 getDefaultSize() 的代码：

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
    }
    return result;
}
```

无论是 AT_MOST 还是 EXACTLY，最终返回的都是 measureSpec 中的 specSize，这个 specSize 就是测量后的最终结果。至于 UNSPECIFIED 的情况，则会返回一个建议的最小值，这个值和子元素设置的最小值它的背景大小有关。

从 onMeasure() 的默认实现可以看出，如果我们自定义一个直接继承自 View 的控件如果不重写 onMeasure() 方法，在使用这个控件并把 layout_width 或 layout_height 设置成 wrap_content 的时候，效果将会和 match_parent 一样！因为布局中使用 wrap_content 的时候，根据上面 getChildMeasureSpec() 方法总结出来的表格可以知道，此时的 specMode 是 AT_MOST，specSize 是 parentSize，而 parentSize 是父容器当前剩余的空间大小，此时 getDefaultSize() 就会返回 specSize，因此子元素的宽或高就被设置成了等于当前父容器剩余空间的大小了，这显然不符合我们的预期，如何解决这个问题呢？一个通用的方案就是像如下方式重写 onMeasure() 方法：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);

    int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    // 默认的宽/高
    int mWidth = default_width;
    int mHeight = default_height;

    // 当布局参数设置为 wrap_content 时，使用默认值
    if (getLayoutParams().width == ViewGroup.LayoutParams.WRAP_CONTENT && getLayoutParams().height == ViewGroup.LayoutParams.WRAP_CONTENT) {
        setMeasuredDimension(mWidth, mHeight);
    // 宽 / 高任意一个布局参数为 wrap_content 时，都使用默认值
    } else if (getLayoutParams().width == ViewGroup.LayoutParams.WRAP_CONTENT) {
        setMeasuredDimension(mWidth, heightSize);
    } else if (getLayoutParams().height == ViewGroup.LayoutParams.WRAP_CONTENT) {
        setMeasuredDimension(widthSize, mHeight);
    }
}
```

因为 DecorView 继承自 FrameLayout，它是一个 ViewGroup，ViewGroup 是一个抽象类，它并没有定义一个具体的测量过程，默认使用 View 的 onMeasure() 进行测量。它的测量过程需要各个子类通过重写 onMeasure() 方法去实现，因为不同的子类具有不同的布局特性，因此需要不一样的测量逻辑，DecorView 也自然重写了 onMeasure() 方法来实现自己的测量逻辑，简略后方法内容如下：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    final int widthMode = getMode(widthMeasureSpec);
    final int heightMode = getMode(heightMeasureSpec);
    ...
    if (widthMode == AT_MOST) {
        ...
    }
    ...
    if (heightMode == AT_MOST) {
        ...
    }
    ...
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    ...
}
```

最终会调用父类 FrameLayout 的 onMeasure() 方法，简略后方法内容如下：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    ...
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            ...
        }
    }
    ...
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));
    ...
}
```

可以看到，这里会遍历它的每一个子元素，并调用 measureChildWithMargins() 方法，这个方法其实前面已经出现过，它的作用是计算出子元素的 MeasureSpec 后调用子元素本身的 measure() 方法：

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    // 获取子元素的布局参数
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    // 产生子元素的 MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

此时实际上调用的也是 View 的 measure() 方法，从上面的内容可以知道，子元素的 onMeasure() 方法又会被调用，这样便实现了层层递归地调用到了每个子元素的 onMeasure() 方法进行测量。

### Layout 流程分析

再次回到 performTraversals() 方法，执行完 performMeasure() 遍历测量所有子元素之后，接着会调用 performLayout() 方法：

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
    ...
    // 调用 DecorView 的 layout() 方法
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    ...
}
```

这里的 getMeasuredWidth() 和 getMeasuredHeight() 就是 DecorView 在前面 Measure 流程中计算得到的测量值，它们都被作为参数传入 layout() 方法中，这里调用的是 View 的 layout() 方法：

```java
public void layout(int l, int t, int r, int b) {
    ...
    // View 状态是否发生了变化
    boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    // 如果 View 状态有变化，则重新布局
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        ...
    }
    ...
}
```

setOpticalFrame() 内部也直接调用了 setFrame() 方法，查看 setFrame() 方法的实现：

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;

    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
        ...
        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

        // Invalidate our old position
        invalidate(sizeChanged);

        // 变量初始化

        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        // 更新用于渲染的显示列表
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
        ...
        if (sizeChanged) {
            // 如果 View 大小发生变化，则会在里面回调 onSizeChanged 方法
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);
        }
        ...
    }
    // 返回是否发生变化
    return changed;
}
```

setFrame() 方法的主要作用有以下几点：

1. 判断 View 位置是否发生变化，并根据变化情况进行相应的处理；
2. 初始化 mLeft、mBottom、mRight 和 mTop 变量，首次调用这个方法的时候返回值为 <em>true</em>；
3. 调用 RenderNode 中原生方法更新用于渲染的显示列表。

回到 layout() 方法，根据 setFrame() 方法返回的状态判断是否需要调用 onLayout() 进行重新布局，查看 onLayout() 方法：

```java
/**
 * Assign a size and position to a view and all of its
 * descendants
 *
 * <p>This is the second phase of the layout mechanism.
 * (The first is measuring). In this phase, each parent calls
 * layout on all of its children to position them.
 * This is typically done using the child measurements
 * that were stored in the measure pass().</p>
 *
 * <p>Derived classes should not override this method.
 * Derived classes with children should override
 * onLayout. In that method, they should
 * call layout on each of their children.</p>
 */
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

onLayout() 方法是一个空方法，从注释最后一段内容可以了解到，单一的 View 并不需要重写这个方法，当 View 的子类具有子元素（即 ViewGroup）的时候，应该重写这个方法并调用每个子元素的 layout() 方法，因此作为一个 ViewGroup，我们查看 DecorView 的 onLayout() 方法：

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
    ...
}
```

这里的主要逻辑是调用父类的 onLayout() 方法，继续跟踪 FrameLayout 的 onLayout() 方法：

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}

void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
    final int count = getChildCount();

    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();

    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();

            int childLeft;
            int childTop;

            int gravity = lp.gravity;
            if (gravity == -1) {
                gravity = DEFAULT_CHILD_GRAVITY;
            }

            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                    lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }

            switch (verticalGravity) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                    lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }
            // 调用每个子元素的 layout 方法
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```

在 layoutChildren() 方法中，根据自己的布局逻辑，计算出每个子元素的 left、top、right 和 bottom 值，并调用它们的 layout() 方法。

Layout 流程的作用是 ViewGroup 用来确定它的子元素的位置， 当 ViewGroup 的位置被确定后，在它的 onLayout() 方法中就会遍历调用所有子元素的 layout() 方法，子元素的 layout() 方法被调用的时候它的 onLayout() 方法又会被调用，这样就实现了层层递归。

### Draw 流程分析

最后，又一次回到主线中的 performTraversals() 方法，此时，经过 Measure 流程确定了每个 View 的大小并且经过 Layout 流程确定了每个 View 的摆放位置，下面将进入下一个流程确定每个 View 的具体绘制细节。查看 performDraw() 方法内容：

```java
private void performDraw() {
    ...
    final boolean fullRedrawNeeded = mFullRedrawNeeded;
    mFullRedrawNeeded = false;

    mIsDrawing = true;
    try {
        draw(fullRedrawNeeded);
    } finally {
        mIsDrawing = false;
    }
    ...
}
```

跟踪 draw() 方法：

```java
private void draw(boolean fullRedrawNeeded) {
    ...
    // “脏”区域，即需要重绘的区域
    final Rect dirty = mDirty;
    ...
    if (fullRedrawNeeded) {
        mAttachInfo.mIgnoreDirtyState = true;
        dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
    }
    ...

    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
            ...
            // 调用 drawSoftware 方法
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }
    ...
}
```

查看 drawSoftware() 方法实现：

```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty) {

    // Draw with software renderer.
    final Canvas canvas;
    try {
        final int left = dirty.left;
        final int top = dirty.top;
        final int right = dirty.right;
        final int bottom = dirty.bottom;

        // 锁定canvas区域，由 dirty 区域决定
        canvas = mSurface.lockCanvas(dirty);
        ...
        canvas.setDensity(mDensity);
    } catch (Surface.OutOfResourcesException e) {
        handleOutOfResourcesException(e);
        return false;
    } catch (IllegalArgumentException e) {
        mLayoutRequested = true;    // ask wm for a new surface next time.
        return false;
    }

    try {
        ...
        try {
            ...
            // 调用 DecorView 的 draw 方法
            mView.draw(canvas);
            ...
        } finally {
            ...
        }
    } finally {
        mLayoutRequested = true;    // ask wm for a new surface next time.
        //noinspection ReturnInsideFinallyBlock
        return false;
    }
    return true;
}
```

继续查看 DecorView 的 draw() 方法：

```java
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);

    if (mMenuBackground != null) {
        mMenuBackground.draw(canvas);
    }
}
```

主要是调用了父类的 draw 方法，FrameLayout 和 ViewGroup 都没有重写 draw() 方法，所以直接看 View 的 draw() 方法：

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;

    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // we're done...
        return;
    }

    /*
     * Here we do the full fledged routine...
     * (this is an uncommon case where speed matters less,
     * this is why we repeat some of the tests that have been
     * done above)
     */

    boolean drawTop = false;
    boolean drawBottom = false;
    boolean drawLeft = false;
    boolean drawRight = false;

    float topFadeStrength = 0.0f;
    float bottomFadeStrength = 0.0f;
    float leftFadeStrength = 0.0f;
    float rightFadeStrength = 0.0f;

    // Step 2, save the canvas' layers
    int paddingLeft = mPaddingLeft;

    final boolean offsetRequired = isPaddingOffsetRequired();
    if (offsetRequired) {
        paddingLeft += getLeftPaddingOffset();
    }

    int left = mScrollX + paddingLeft;
    int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
    int top = mScrollY + getFadeTop(offsetRequired);
    int bottom = top + getFadeHeight(offsetRequired);

    if (offsetRequired) {
        right += getRightPaddingOffset();
        bottom += getBottomPaddingOffset();
    }

    final ScrollabilityCache scrollabilityCache = mScrollCache;
    final float fadeHeight = scrollabilityCache.fadingEdgeLength;
    int length = (int) fadeHeight;

    // clip the fade length if top and bottom fades overlap
    // overlapping fades produce odd-looking artifacts
    if (verticalEdges && (top + length > bottom - length)) {
        length = (bottom - top) / 2;
    }

    // also clip horizontal fades if necessary
    if (horizontalEdges && (left + length > right - length)) {
        length = (right - left) / 2;
    }

    if (verticalEdges) {
        topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
        drawTop = topFadeStrength * fadeHeight > 1.0f;
        bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
        drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
    }

    if (horizontalEdges) {
        leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
        drawLeft = leftFadeStrength * fadeHeight > 1.0f;
        rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
        drawRight = rightFadeStrength * fadeHeight > 1.0f;
    }

    saveCount = canvas.getSaveCount();

    int solidColor = getSolidColor();
    if (solidColor == 0) {
        final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

        if (drawTop) {
            canvas.saveLayer(left, top, right, top + length, null, flags);
        }

        if (drawBottom) {
            canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
        }

        if (drawLeft) {
            canvas.saveLayer(left, top, left + length, bottom, null, flags);
        }

        if (drawRight) {
            canvas.saveLayer(right - length, top, right, bottom, null, flags);
        }
    } else {
        scrollabilityCache.setFadeColor(solidColor);
    }

    // Step 3, draw the content
    if (!dirtyOpaque) onDraw(canvas);

    // Step 4, draw the children
    dispatchDraw(canvas);

    // Step 5, draw the fade effect and restore layers
    final Paint p = scrollabilityCache.paint;
    final Matrix matrix = scrollabilityCache.matrix;
    final Shader fade = scrollabilityCache.shader;

    if (drawTop) {
        matrix.setScale(1, fadeHeight * topFadeStrength);
        matrix.postTranslate(left, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, top, right, top + length, p);
    }

    if (drawBottom) {
        matrix.setScale(1, fadeHeight * bottomFadeStrength);
        matrix.postRotate(180);
        matrix.postTranslate(left, bottom);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, bottom - length, right, bottom, p);
    }

    if (drawLeft) {
        matrix.setScale(1, fadeHeight * leftFadeStrength);
        matrix.postRotate(-90);
        matrix.postTranslate(left, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, top, left + length, bottom, p);
    }

    if (drawRight) {
        matrix.setScale(1, fadeHeight * rightFadeStrength);
        matrix.postRotate(90);
        matrix.postTranslate(right, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(right - length, top, right, bottom, p);
    }

    canvas.restoreToCount(saveCount);

    // Overlay is part of the content and draws beneath Foreground
    if (mOverlay != null && !mOverlay.isEmpty()) {
        mOverlay.getOverlayView().dispatchDraw(canvas);
    }

    // Step 6, draw decorations (foreground, scrollbars)
    onDrawForeground(canvas);
}
```

从代码注释中可以看到，draw() 过程分为六个步骤，分别是：

1. 绘制 View 的背景；
2. 保存当前 canvas 的图层；
3. 绘制 View 的内容；
4. 如果有子元素，则调用子元素的 draw() 方法；
5. 绘制淡入淡出效果并恢复图层；
6. 绘制 View 的装饰（滚动条等）。

其中第二步和第五步一般情况下不会用到，我们继续看第三步，看看它是如何分配子元素的绘制的：

```java
protected void dispatchDraw(Canvas canvas) {
}
```

显然，单一的 View 并没有子元素，因此，看看 ViewGroup 是怎么实现这个过程的：

```java
@Override
protected void dispatchDraw(Canvas canvas) {
    ...
    for (int i = 0; i < childrenCount; i++) {
        while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            ...
        }
        ...
    }
    ...  
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

在 ViewGroup 的 dispatchDraw() 中，遍历每个子元素并调用了它们的 draw() 方法，由此实现了层层递归调用，最终完成绘制。

### 小结

纵观整个 Measure、Layout 和 Draw 过程，使用流程图表示如下：

![](static/boxcnMzknL6Ff4vHQZfdDuz9P4b.)

## 步骤五：显示 View

不知道你还记不记得，上一步骤中执行的 View 的 Measure、Layout 和 Draw 流程都是前面 handleResumeActivity() 中的 wm.addView() 方法为源头的，回顾 handleResumeActivity() 方法：

```java
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    ...
    // 最终会调用 onStart() 和 onResume() 等方法
    r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;
        ...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            // 获取 DecorView 
            View decor = r.window.getDecorView();
            // 将 DecorView 设置成不可见
            decor.setVisibility(View.INVISIBLE);
            // 获取 ViewManager，这里是 WindowManagerImpl 实例
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            ...
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                // 标记设置为 true
                a.mWindowAdded = true;
                // 调用 WindowManagerImpl 的 addView 方法
                wm.addView(decor, l);
            } 
        } else if (!willBeVisible) {
            ...
        }
        ...
        if (!r.activity.mFinished && willBeVisible
                && r.activity.mDecor != null && !r.hideForNow) {
            ...
            if (r.activity.mVisibleFromClient) {
                // 调用 makeVisible 方法将 DecorView 设置为可见
                r.activity.makeVisible();
            }
        }
        ...
    } else {
        try {
            // 在此过程出现异常，则直接杀死 Activity
            ActivityManagerNative.getDefault()
                .finishActivity(token, Activity.RESULT_CANCELED, null,
                        Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

可以看到在调用 wm.addView() 方法之前，DecorView 是处于不可见的状态的，因此，即使经过了 Measure、Layout 和 Draw 流程，我们的 View 仍然没有显示在屏幕上，看看 Activity 的 makeVisible() 方法：

```java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

执行 DecorView 的 setVisibility() 之后，我们的 View 才正式出现在屏幕上！

## 总结

- Window 是一个抽象类，提供了各种窗口操作的方法；
- PhoneWindow 是 Window 的唯一实现类，每个 Acitvity 中都会有一个 PhoneWindw 实例；
- DecorView —— 顶级视图，它继承自 FrameLayout，setContentView() 时，PhoneWindow 会创建 DecorView 并与 DecorView 建立关联；
- PhoneWindow 会根据 Theme 和 Feature 等将对应的布局文件将布局解析并添加到 DecorView 中，这些布局中都包含了一个 id 为 content 的 FrameLayout；
- setContentView() 方法中设置的 layout 布局文件会被 PhoneWindow 解析并压入 DecorView 中 id 为 content 的 FrameLayout 中；
- View 的正式绘制是从 ViewRootImpl 的 performTraversals() 方法开始的；
- 单一 View 一般需要重写 onMeasure() 方法根据布局参数和父 View 的测量规格计算自己的宽高并保存；
- ViewGroup 需要重写 onMeasure() 方法计算所有子元素的尺寸然后计算自己的尺寸并保存；
- 单一 View 一般不需要重写 onLayout() 方法；
- ViewGroup 需要重写 onLayot() 方法根据测量的值确定所有子元素的位置；
- 单一 View 需要重写 onDraw() 方法绘制自身；
- ViewGroup 需要重写 onDraw() 方法绘制自身以及遍历子元素对它们进行绘制。
- 在 Activity 的 onResume() 生命周期被调用后，ActivityThread 才会调用 activity.makeVisible() 让 DecorView 可见。

<strong>系列文章</strong>

[按下电源键后竟然发生了这一幕 —— Android 系统启动流程分析](https://guanpj.cn/2017/09/17/Android-System-Startup-Flow-Analyze/)

[App 竟然是这样跑起来的 —— Android App/Activity 启动流程分析](https://guanpj.cn/2017/10/23/Android-App-Startup-Flow-Analyze/)

[屏幕上内容究竟是怎样画出来的 —— Android View 工作原理详解](https://guanpj.cn/2017/11/09/Android-View-Workflow/)（本文）

<strong>参考文章</strong>

[Android 应用层 View 绘制流程与源码分析](https://blog.csdn.net/yanbober/article/details/46128379)

[（3）自定义 View Layout 过程 - 最易懂的自定义 View 原理系列](https://www.jianshu.com/p/158736a2549d)

> 如果你对文章内容有疑问或者有不同的意见，欢迎留言，我们一同探讨。
