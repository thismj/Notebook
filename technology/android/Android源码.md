# Android源码

## Init进程的启动过程

[Android系统启动流程（一）解析init进程启动过程](http://liuwangshu.cn/framework/booting/1-init.html)

引导芯片代码 -> 加载Bootloader到RAM执行 -> Linux内核启动 -> init进程启动 init.cpp main()

* 创建一些文件夹并挂载设备
* property_init 初始化属性，start_property_service启动属性服务
* 解析 init.rc、init.zygote32(64).rc

-> builtins.cpp do_class_start() 

* fork 子进程
* app_main.cpp main()
* AndroidRuntime.start("com.android.internal.os.ZygoteInit",, args, zygote)

## Zygote进程启动

[Android系统启动流程（二）解析Zygote进程启动过程](http://liuwangshu.cn/framework/booting/2-zygote.html)

AndroidRuntime.start(）

* startVm() 启动 JavaVM(Dalvik、ART) 虚拟机
* startReg() 为虚拟机注册JNI
* 通过 JNI 调用 ZygoteInit.java 的 main()，从此进 Java 的世界

 ZygoteInit.java main()

* registerZygoteSocket(), 创建Server端的监听Socket
  * 创建sServerSocket (LocalServerSocket)  
* preload()，预加载一些常用的类和资源
* startSystemServer() 启动系统服务
  * Zygote.forkSystemServer() (uid gid设置为1000)
  * ZygoteInit.handleSystemServerProcess 启动SystemServer进程
* runSelectLoop() 等待客户端连接(AMS 请求 Zygote 创建新的应用程序进程)

## SystemServer进程启动

[Android系统启动流程（三）解析SyetemServer进程启动过程](http://liuwangshu.cn/framework/booting/3-syetemserver.html)

ZygoteInit.java startSystemServer()

* Zygote.forkSystemServer()： fork SystemServer进程
* handleSystemServerProcess() ：closeServerSocket()关闭从Zygote进程复制来的Socket；SystemServer进程启动之后的工作

RuntimeInit.zygoteInit：

* nativeZygoteInit()：
  * AndroidRuntime.cpp com_android_internal_os_RuntimeInit_nativeZygoteInit()
  * AppRuntime.onZygoteInit()
  * ProcessState.startThreadPool()：启动一个Binder线程池，这样SyetemServer进程就可以使用
Binder来与其他进程进行通信了
* applicationInit()



## Activity的启动过程

冷启动，点击 Launcher 应用图标进入 APP 时序图：

Instrumentation：类负责监控系统与应用之间的所有交互

ActivityManagerProxy: 即 *ActivityManagerNative.getDefault()* ，是 AMS 在应用进程的 Binder 代理对象的**封装**。

```mermaid
sequenceDiagram
    autonumber
    ...->>Launcher: startActivitySafely()
    activate Launcher
    Launcher->>Activity: startActivity()
    activate Activity
    Activity->>Activity: startActivityForResult()
    Activity->>Instrumentation: execStartActivity()
    deactivate Activity
    deactivate Launcher
    activate Instrumentation
    Instrumentation->>ActivityManagerProxy: startActivity()
    activate ActivityManagerProxy
    deactivate Instrumentation
    ActivityManagerProxy->>ActivityManagerService: startActivity()
    deactivate ActivityManagerProxy
    
```

* ActivityRecord：系统服务中的 Activity 实例
* TaskRecord：Activity任务栈，内部维护了一个 ActivityRecord 的列表
* ActivityStack：内部维护了一个 TaskRecord 的列表，方便管理所有的 TaskRecord；一般来说Launcher的Task属于单独的一个ActivityStack，称为mHomeStack，System UI 如 rencentActivity 的Task属于一个单独ActivityStack，其他App的Task属于另一个ActivityStack
* ActivityManagerService：启动 Activity 的核心系统服务，ActivityStarter 负责启动，ActivityStackSupervisor 负责管理 ActivityStack，ActivityStack 负责管理 TaskRecord 和 ActivityRecord
* startActivityMayWait：通过 PMS 根据 Intent 获取 Activity 的启动信息 (ResolveInfo和ActivityInfo), 获取调用者的 Pid 和 Uid
* startActivityLocked： 创建ActivityRecord, 含有 Activity 的核心信息
* startActivityUnchecked：根据启动的 Flag 信息以及启动模式, 设置 TaskRecord, 完成后执行
* startSpecificActivityLocked：判断当前应用进程是否创建，没有创建则 AMS 跟 Zygote 通过 Socket 的方式新建进程
* scheduleLaunchActivity: 系统服务通过 Binder 代理对象的封装对象 ApplicationThreadProxy 远程调用应用进程的 ApplicationThread Binder（ActivityThread的内部类）
```mermaid
sequenceDiagram
    autonumber
    activate ActivityManagerService
    ActivityManagerService->>ActivityManagerService: startActivityAsUser()
    ActivityManagerService->>ActivityStarter: startActivityMayWait()
    deactivate ActivityManagerService
    activate ActivityStarter
    ActivityStarter->>ActivityStarter: startActivityLocked()
    ActivityStarter->>ActivityStarter: startActivityUnchecked()
    ActivityStarter->>ActivityStarter: computeLaunchingTaskFlags()
    ActivityStarter->>ActivityStarter: startActivityLocked()
    ActivityStarter->>ActivityStackSupervisor: resumeFocusedStackTopActivityLocked()
    activate ActivityStackSupervisor
    deactivate ActivityStarter
    ActivityStackSupervisor->>ActivityStack: resumeTopActivityUncheckedLocked()
    deactivate ActivityStackSupervisor
    activate ActivityStack
    ActivityStack->>ActivityStack: resumeTopActivityInnerLocked()
    ActivityStack->>ActivityStackSupervisor: startSpecificActivityLocked()
    deactivate ActivityStack
    ActivityStackSupervisor->>ActivityStackSupervisor: realStartActivityLocked()
    activate ActivityStackSupervisor
    ActivityStackSupervisor->>ApplicationThreadProxy: scheduleLaunchActivity()
    deactivate ActivityStackSupervisor
    activate ApplicationThreadProxy
    ApplicationThreadProxy->>ApplicationThread: scheduleLaunchActivity()
    deactivate ApplicationThreadProxy
```

* ApplicationThread接收到 AMS 的远程调用，然后把事件抛给 ActivityThread 的 H（Handler）对象，H.handleMessage() 调用 ActivityThread 的方法（例如 handleLaunchActivity()）顺序处理这些事件，并最终通过 Instrumentation 依次回调目标 Activity 的生命周期方法
* createBaseContextForActivity: 创建 ContextImpl 对象，
* Activity.sttach(): 内部调用 Activity（ContextThemeWrapper）的 attachBaseContext() 方法，将创建的 ContextImpl  对象设置为 Activity（ContextWrapper）的 mBase 属性；内部创建 PhoneWindow 对象

```mermaid
sequenceDiagram
    ApplicationThread->>ActivityThread: sendMessage()
    activate ApplicationThread
    activate ActivityThread
    ActivityThread->>H: handleMessage()
    deactivate ApplicationThread
    deactivate ActivityThread
    activate H
    H->>ActivityThread: handleLaunchActivity()
    deactivate H
    activate ActivityThread
    ActivityThread->>ActivityThread: performLaunchActivity()
    ActivityThread->>Instrumentation: newActivity()
    deactivate ActivityThread
    activate Instrumentation
    Instrumentation-->>ActivityThread: Activity实例
    activate ActivityThread
    deactivate Instrumentation
    ActivityThread->>ActivityThread: createBaseContextForActivity()
    ActivityThread->>目标Activity: attach()
    activate 目标Activity
    deactivate ActivityThread
    目标Activity->>目标Activity: attachBaseContext()
    deactivate 目标Activity
    ActivityThread->>Instrumentation: callActivityOnCreate()
    activate Instrumentation
    activate ActivityThread
    deactivate ActivityThread
    Instrumentation->>目标Activity: performCreate()
    activate 目标Activity
    deactivate Instrumentation
    目标Activity->>目标Activity: onCreate()
    deactivate 目标Activity
    ActivityThread->>目标Activity: performStart()
    activate ActivityThread
    deactivate ActivityThread
    activate 目标Activity
    目标Activity->>Instrumentation: callActivityOnStart()
    deactivate 目标Activity
    activate Instrumentation
    Instrumentation->>目标Activity: onStart()
    deactivate Instrumentation
    activate 目标Activity
    deactivate 目标Activity
    activate ActivityThread
    ActivityThread->>ActivityThread: handleResumeActivity()
    ActivityThread->>ActivityThread: performResumeActivity()
    ActivityThread->>目标Activity: performResume()
    deactivate ActivityThread
    activate 目标Activity
    目标Activity->>Instrumentation: callActivityOnResume()
    deactivate 目标Activity
    activate Instrumentation
    Instrumentation->>目标Activity: onResume()
    deactivate Instrumentation
    activate 目标Activity
    deactivate 目标Activity
```

## setContentView()

![](https://huangyu.github.io/archives/199d23fd/View%E7%BB%93%E6%9E%84.png)

Activity 的 setContentView() 会调用 PhoneWindow 的 setContentView() 方法

PhoneWindow.setContentView()

```java
public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            //mContentParent即为id为 android.R.id.content 的ViewGroup
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //将我们指定的 layoutResId inflate 到 mContentParent
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            //回调 onContentChanged()
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

PhoneWindow.installDecor()

```java
private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            //创建 PhoneWindow 里面的 DecorView 对象
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
            //获取 DecorView 中 id 为 android.R.id.content 的 Viewgroup
            mContentParent = generateLayout(mDecor);
            
            ......
              
        }
    }
```

PhoneWindow.generateDecor()

```java
protected DecorView generateDecor(int featureId) {
        // System process doesn't have application context and in that case we need to directly use
        // the context we have. Otherwise we want the application context, so we don't cling to the
        // activity.
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext().getResources());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        //New 根布局 DecorView
        return new DecorView(context, featureId, this, getAttributes());
    }
```

PhoneWindow.generateLayout()

```java
protected ViewGroup generateLayout(DecorView decor) {
        // 处理主题的各种窗口属性、Flag，一大段逻辑
        TypedArray a = getWindowStyle();

        ......

        mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
        int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
                & (~getForcedWindowFlags());
        if (mIsFloating) {
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }

        if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestFeature(FEATURE_ACTION_BAR);
        }
        ......
          
        // Inflate the window decor.
        // DecorView 根据上面主题解析得到的各种属性、Flag，获取对应布局
        int layoutResource;
        int features = getLocalFeatures();
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
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource = R.layout.screen_progress;
            // System.out.println("Progress!");
        } 
        
        ......

        mDecor.startChanging();
        //DecorView 添加对应布局的 View
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
        //获取 id 为 android.R.id.content 的 ViewGroup
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        ......

        return contentParent;
    }
```

至此，Activity 的 setContentView() 就完毕了，首先通过 PhoneWindow 来创建根布局 DecorView，然后 DecorView 根据 Activity 主题的不同属性和窗口功能来添加对应布局的View，最后在DecorView 中 id 为 android.R.id.content 的 ViewGroup 里面 inflate 我们指定的布局 id。



## Service的启动过程

Application 跟 Activity 的 `startService()` 都继承自 ContextImpl，以下为启动 Service 的时序图：

```mermaid
sequenceDiagram
.->>ContextImpl: startService()
ContextImpl->>ContextImpl: startServiceCommon()
ContextImpl->>ContextImpl: validateServiceIntent()
ContextImpl->>ActivityManagerProxy: startService()
ActivityManagerProxy->>ActivityManagerService: startService()
ActivityManagerService->>ActiveServices: startServiceLocked()
ActiveServices->>ActiveServices: startServiceInnerLocked()
ActiveServices->>ActiveServices: bringUpServiceLocked()
ActiveServices->>ActiveServices: realStartServiceLocked()
ActiveServices->>ApplicationThreadProxy: scheduleCreateService()
ApplicationThreadNative->>ApplicationThread: scheduleCreateService()
ApplicationThread->>H: sendMessage(H.CREATE_SERVICE)
H->>ActivityThread: handleCreateService()
ActivityThread->>Service: attach()
ActivityThread->>Service: onCreate()
ActivityThread->>ActivityManagerProxy: serviceDoneExecuting()
ActivityManagerProxy->>ActivityManagerService: serviceDoneExecuting()
ActivityManagerService->>ActiveServices: serviceDoneExecutingLocked()
ActiveServices->>ActiveServices: sendServiceArgsLocked()
ActiveServices->>ApplicationThreadProxy: scheduleServiceArgs()
ApplicationThreadProxy->>ApplicationThread: scheduleServiceArgs()
ApplicationThread->>H: sendMessage(H.SERVICE_ARGS)
H->>ActivityThread: handleServiceArgs()
ActivityThread->>Service: onStartCommand()
ActivityThread->>ActivityManagerProxy: serviceDoneExecuting()
ActivityManagerProxy->>ActivityManagerService: serviceDoneExecuting()
ActivityManagerService->>ActiveServices: serviceDoneExecutingLocked()

```
ContextImpl.validateServiceIntent(): 检查 Intent 是否显示指定 Service 组件，Android L 之后不允许隐式启动 Service。



## View绘制流程

Activity创建之后，调用 attch() 方法：

```java
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        attachBaseContext(context);

        ......

        //创建PhoneWindow
        mWindow = new PhoneWindow(this, window);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        
        ......

        //设置mWindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        ......
    }
```

* 在 `handleResumeActivity()` 里面，通过上面创建的 `mWindowManager` 添加 `PhoneWindow` 的 `DecorView`，最终调用到 `WindowManagerGlobal` 的 `addView(...)`，在里面更新了 `DecorView` 的 `WindowManager.LayoutParams`，创建了 `ViewRootImpl` 对象（初始化成员变量`mThread`、`mWindowSession`、`mChoreographer`等），最后通过 `ViewRootImpl` 的 `setView()` 方法绑定了 `DecorView` 
* `ViewRootImpl`创建时会通过 `WindowManagerGlobal.getWindowSession() `获取 `WindowSession` 对象（WMS 的 `openSession()` 方法）
* 一个`ViewRootImpl `对象对应着一次 `WindowManager.addView` 中的`View`，并管理着这个` View` 树的测量布局和绘制等工作，通过 `WindowSession` 对象与 `WMS` 交互
* 在`ViewRootImpl`的`setView()` 方法里面，会首次调用自己的 `requestlayout()` 方法，如果 `checkThread()` 没问题，则会调用 `scheduleTraversals()` 开始对 `View` 树进行遍历绘制，在绘制之前，会调用主线程 `MessageQueue` 的 `postSyncBarrier()` 方法，确保及时处理异步消息。然后通过 `mChoreographer` 请求最近的绘制信号（vsync，此处使用了Handler发送异步消息），vsync信号回调时调用 `doTraversal()` 开始真正遍历，并调用主线程 `MessageQueue` 的 `removeSyncBarrier()` 方法，移除异步屏障。
* 如果是首次遍历`View`树，调用 `dispatchAttachedToWindow()` ，分发 View 树的 `onAttachedToWindow()` 方法
```mermaid
sequenceDiagram
activate ActivityThread
ActivityThread->>ActivityThread: performResumeActivity()
ActivityThread->>WindowManagerImpl: addView(decorView, params)
WindowManagerImpl->>WindowManagerGlobal: addView(decorView, params...)
WindowManagerGlobal->>ViewRootImpl: new
ViewRootImpl-->>WindowManagerGlobal: ViewRootImpl实例
WindowManagerGlobal->>ViewRootImpl: setView(decorview...)
ViewRootImpl->>ViewRootImpl: requestLayout()
ViewRootImpl->>ViewRootImpl: checkThread()
ViewRootImpl->>ViewRootImpl: scheduleTraversals()
ViewRootImpl->>Choreographer: postCallback(TraversalRunnable)
TraversalRunnable->>ViewRootImpl: doTraversal()
ViewRootImpl->>ViewRootImpl: performTraversals()
ViewRootImpl->>ViewRootImpl: performMeasure()
ViewRootImpl->>ViewRootImpl: performLayout()
ViewRootImpl->>ViewRootImpl: performDraw()
ViewRootImpl->>IWindowSession: addToDisplay(IWindow...)


```

测量、布局、绘制三部曲，首先是进入到 `performTraversals()` 方法，然后调用 `measureHierarchy` 进行 View 树的测量工作。

```java
/**
 * desiredWindowWidth desiredWindowHeight 代表所需的窗口宽度，一般是屏幕的宽高
 */
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        boolean windowSizeMayChange = false;
        ......
        if (!goodMeasure) {
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }
        ......
        return windowSizeMayChange;
    }
```

`ViewRootImpl.getRootMeasureSpec()`：

```java
/**
 * 获取RootMeasureSpec
 */
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        //根据添加DecorView的LayoutParams的width跟height来设置不同的测量规格
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            //DecorView高宽与窗口一致
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            //DecorView高宽最大为窗口的宽高
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            //DecorView高宽为指定的宽高
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```

`ViewRootImpl.performMeasure()`：

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    //systrace事件
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        //调用DecoreView的measure方法
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

`DecorView` 继承自 `FrameLayout`，所以实际上是



`View` 树测量完毕之后，则开始调用 `performLayout()` 进行布局，`ViewRootImpl.performLayout()`：

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        ......
        mInLayout = true;

        final View host = mView;
        ......

        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
        try {
            //调用DecorView的layout方法
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            mInLayout = false;
            
            ......
            
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
```





View 树布局完成之后，则开始执行绘制，`ViewRootImpl.performDraw()`：

```java
private void performDraw() {
        ......
        final boolean fullRedrawNeeded = mFullRedrawNeeded;
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
        try {
            //调用绘制方法draw
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        ......
    }
```

`ViewRootImpl.draw()`：

```java
 private void draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
        if (!surface.isValid()) {
            return;
        } 
        ......
        final Rect dirty = mDirty;
        ......
        final float appScale = mAttachInfo.mApplicationScale;
        final boolean scalingRequired = mAttachInfo.mScalingRequired;
        ......
        if (fullRedrawNeeded) {
            mAttachInfo.mIgnoreDirtyState = true;
            dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        }

        int xOffset = -mCanvasOffsetX;
        int yOffset = -mCanvasOffsetY + curScrollY;
        final WindowManager.LayoutParams params = mWindowAttributes;
        final Rect surfaceInsets = params != null ? params.surfaceInsets : null;
        if (surfaceInsets != null) {
            xOffset -= surfaceInsets.left;
            yOffset -= surfaceInsets.top;

            // Offset dirty rect for surface insets.
            dirty.offset(surfaceInsets.left, surfaceInsets.right);
        }

        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
               ......
            } else {
                ......
                 //调用绘制方法drawSoftware
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                    return;
                }
            }
        }
    ......
    }
```

`ViewRootImpl.drawSoftware()`：

```java
    /**
     * 软件渲染
     */
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

        // Draw with software renderer.
        final Canvas canvas;
        try {
            ...
            //锁定画布
            canvas = mSurface.lockCanvas(dirty);
            ...
        } catch (Surface.OutOfResourcesException e) {
            ...
            return false;
        } catch (IllegalArgumentException e) {
            ...
            return false;
        }

        try {
            ...
            try {
                ...
                //调用DecorView的draw方法
                mView.draw(canvas);
                ...
            } finally {
                ...
            }
        } finally {
            try {
                
                surface.unlockCanvasAndPost(canvas);
            } catch (IllegalArgumentException e) {
                ...
                return false;
            }
        }
        return true;
    }
```



## View绘图引擎

`canvas` 底层使用的是 Skia 绘制引擎



## Choreographer

[Android的16ms和垂直同步以及三重缓存](https://cloud.tencent.com/developer/article/1176313)

[Android-Choreographer原理](https://ljd1996.github.io/2020/09/07/Android-Choreographer%E5%8E%9F%E7%90%86/)



## ANR

### 触发原理

#### Service

启动或者绑定 Service 时，一个生命周期流程（APP进程和System_Server进程交互）create、start、bind 等，每个周期都会在 AMS 中记录一个超时时间，前台服务为 20s，后台服务为 200s，如果超出时间还没有处理完，则会出发 ANR 事件。

举例，记录 Service 创建即 create 周期超时的调用在 AMS 的 `realStartServiceLocked` 方法中：

```java
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        ......
        //Service创建超时记录
        bumpServiceExecutingLocked(r, execInFg, "create");
        ......
        try {
            ......
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            ......
        } catch (DeadObjectException e) {
            ......
        } finally {
            ......
        }
        ......
    }
```

`bumpServiceExecutingLocked`

```java
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        ......
        if (r.executeNesting == 0) {
            r.executeFg = fg;
            ServiceState stracker = r.getTracker();
            if (stracker != null) {
                stracker.setExecuting(true, mAm.mProcessStats.getMemFactorLocked(), now);
            }
            if (r.app != null) {
                r.app.executingServices.add(r);
                r.app.execServicesFg |= fg;
                if (r.app.executingServices.size() == 1) {
                   //前台服务？？？
                    scheduleServiceTimeoutLocked(r.app);
                }
            }
        } else if (r.app != null && fg && !r.app.execServicesFg) {
            r.app.execServicesFg = true;
            //后台服务？？？
            scheduleServiceTimeoutLocked(r.app);
        }
        ......
    }
```

`scheduleServiceTimeoutLocked`

```java
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        long now = SystemClock.uptimeMillis();
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        //前台服务20s，后台服务是它的10倍
        mAm.mHandler.sendMessageAtTime(msg,
                proc.execServicesFg ? (now+SERVICE_TIMEOUT) : (now+ SERVICE_BACKGROUND_TIMEOUT));
    }
```

如果在超时时间内完成了操作，即移除掉超时消息，在 AMS 的 `serviceDoneExecutingLocked` 方法中

```java
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        ......
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);
                if (r.app.executingServices.size() == 0) {
                    //当前服务所在进程中没有正在执行的service
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                }
           ......
        }
    }
```

如果超时时间内没有完成，则在 AMS 的 MainHandler 处理该超时消息：

```java
final class MainHandler extends Handler {
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case SERVICE_TIMEOUT_MSG: {
                ......
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;
            ......
        }
        ......
    }
}
```

`ActiveServices.serviceTimeout`

```java
void serviceTimeout(ProcessRecord proc) {
        String anrMessage = null;
        ......
        if (anrMessage != null) {
            //触发ANR
            mAm.mAppErrors.appNotResponding(proc, null, null, false, anrMessage);
        }
    }
```

### BroadcastReceiver

BroadcastQueue.BroadcastHandler 收到`BROADCAST_TIMEOUT_MSG`消息时触发

对于广播队列有两个: foreground 队列和 background 队列:

- 对于前台广播，则超时为BROADCAST_FG_TIMEOUT = 10s；
- 对于后台广播，则超时为BROADCAST_BG_TIMEOUT = 60s

### ContentProvider

AMS.MainHandler收到 `CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG` 消息时触发, 超时时间为 `CONTENT_PROVIDER_PUBLISH_TIMEOUT = 10s`,跟前面的Service和BroadcastQueue完全不同, 由Provider[进程启动](http://gityuan.com/2016/10/09/app-process-create-2/)过程相关

### input事件

[Input系统—启动篇](http://gityuan.com/2016/12/10/input-manager/)

1. EventHub 通过 `getEvents` 获取 /dev/input 设备节点的输入事件
2. InputReader线程循环地通过 EventHub 去读取输入事件，一旦有输入事件则将其放在 mInBoundQueue 队列，并唤醒 InputDispatcher 线程
3. InputDispatcher线程是一个Looper线程，循环从 mInBoundQueue 队列中获取输入事件，然后根据焦点找到对应的 window 来处理

事件1过来了，此时没有正在处理的事件，则分发事件1，并重置事件开始处理的时间timestamp，如果此时事件2过来了，但是事件1还在处理中，则通过之前记录的 timestamp 来判断是否超过5s，如果超过了则通知系统上报 ANR 了。

### 信息收集

[Android EventLog含义](http://gityuan.com/2016/05/15/event-log/)

[linux /proc/loadavg(平均负载)](https://www.cnblogs.com/276815076/archive/2012/06/11/2544937.html)

不管是Service、Broadcast还是 input 的 ANR，最终的收集都会走到  `AppErrors.appNotResponding()`方法:

```java
final void appNotResponding(ProcessRecord app, ActivityRecord activity,
            ActivityRecord parent, boolean aboveSystem, final String annotation) {
        ......
        //记录当前anr的时间
        long anrTime = SystemClock.uptimeMillis();
        
        synchronized (mService) {
            ......
            //输出EventLog，它的格式定义在/system/etc/event-log-tags，平常开发时可以通过 logcat -b events 
            EventLog.writeEvent(EventLogTags.AM_ANR, app.userId, app.pid,
                    app.processName, app.info.flags, annotation);
            ......
        }
        ......

        StringBuilder info = new StringBuilder();
        info.setLength(0);
        //ANR进程名字
        info.append("ANR in ").append(app.processName);
        if (activity != null && activity.shortComponentName != null) {
            info.append(" (").append(activity.shortComponentName).append(")");
        }
        info.append("\n");
        //ANR进程ID
        info.append("PID: ").append(app.pid).append("\n");
        if (annotation != null) {
            //ANR原因，是Service、Broadcast还是input事件导致的
            info.append("Reason: ").append(annotation).append("\n");
        }
        if (parent != null && parent != activity) {
            info.append("Parent: ").append(parent.shortComponentName).append("\n");
        }

        ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);

        String[] nativeProcs = NATIVE_STACKS_OF_INTEREST;
        // don't dump native PIDs for background ANRs
        File tracesFile = null;
        if (isSilentANR) {
            tracesFile = mService.dumpStackTraces(true, firstPids, null, lastPids,
                null);
        } else {
            //写入trace文件（路径定义在dalvik.vm.stack-trace-file属性），第一个标志位clearTraces为true时，如果trace文件存在则先删除
            tracesFile = mService.dumpStackTraces(true, firstPids, processCpuTracker, lastPids,
                nativeProcs);
        }

        String cpuInfo = null;
        if (ActivityManagerService.MONITOR_CPU_USAGE) {
            mService.updateCpuStatsNow();
            synchronized (mService.mProcessCpuTracker) {
                //ANR时间点之前的CPU使用信息
                cpuInfo = mService.mProcessCpuTracker.printCurrentState(anrTime);
            }
            //1、5、15分钟内CPU的平均负载（记录在设备节点/proc/loadavg），查看cpu核心数可以通过 cat /proc/cpuinfo
            info.append(processCpuTracker.printCurrentLoad());
            info.append(cpuInfo);
        }
        //ANR时间点之前的CPU使用信息
        info.append(processCpuTracker.printCurrentState(anrTime));
        //输出 ANR in 日志到控制台.
        Slog.e(TAG, info.toString());
        if (tracesFile == null) {
            // There is no trace file, so dump (only) the alleged culprit's threads to the log
            Process.sendSignal(app.pid, Process.SIGNAL_QUIT);
        }

        mService.addErrorToDropBox("anr", app, app.processName, activity, parent, annotation,
                cpuInfo, tracesFile, null);

        ......

        synchronized (mService) {
            ......
            // Bring up the infamous App Not Responding dialog
            Message msg = Message.obtain();
            HashMap<String, Object> map = new HashMap<String, Object>();
            msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
            msg.obj = map;
            msg.arg1 = aboveSystem ? 1 : 0;
            map.put("app", app);
            if (activity != null) {
                map.put("activity", activity);
            }
            //ANR弹窗
            mService.mUiHandler.sendMessage(msg);
        }
    }
```

分析ANR：

1. 查看 event log 中的 `am_anr (User|1|5),(pid|1|5),(Package Name|3),(Flags|1|5),(reason|3)` 打印，确定anr发生的进程信息以及原因（Service、Brocast、input）
2. 查看 main log 中 `ANR in` 的打印， 这里的日志也可以查看一些进程必要信息以及原因、CPU 在过去 1、5、15 分钟的平均负载，各个进程的CPU占用率等
3. 查看 /data/anr/trace.txt 文件
4. trace.txt文件只会存最近一次发生的 ANR，会把之前发生过的覆盖掉，所有有时还需要 /data/system/dropbox 文件夹下面 `system_app_anr@时间戳.txt.gz` 文件

#### ANR例子

* 主线程执行耗时的计算或者I/O操作，例如 SharedPrefeces.commit() 或者 apply() 方法，日志保存等
* 主线程与子线程发生锁竞争
* 通过Binder执行跨进程同步调用时，对端阻塞
* CPU负载过高，资源紧张

## RecyclerView

四级缓存：

`mAttachedScrap`：屏幕内 `itemView`缓存，无需`createViewHolder()`和`bindViewHolder()`。

`mCachView`：保存最近移除出屏幕的 `ViewHolder`，包含 `data`和`position`数据，复用时必须时相同位置的`ViewHolder`才能复用，应用于来回滑动的场景，复用时不需要重新`bindViewHoder`，默认缓存大小为 2。

`mViewCacheExtension`：自定义缓存实现，默认无。

`mRecyclerPool`：`mCachView`满了之后，按照先进先出的原则，把最先缓存的 `ViewHolder` 转移到`mRecyclerPool`中，此时会清除数据，复用需要重新`bindViewHoder()`。

性能优化：

设置`RecyclerView.addOnScrollListener()，来在滑动过程中停止加载的操作。

局部刷新`notifyItemXXX`、`payload`

通过`RecyclerView.setRecycledViewPool(pool)`共用缓存池

增大 `mCachView`缓存的最大数量，空间换时间

如果高度固定，可以设置`setHasFixedSize(true)`来避免`requestLayout`浪费资源，否则每次更新数据都会重新测量高度

