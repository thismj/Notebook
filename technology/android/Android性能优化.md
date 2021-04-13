# Android性能优化

## 性能优化工具

### **Profile GPU Rendering**

| 柱状图颜色 | 渲染阶段            | 说明                                                         |
| ---------- | ------------------- | ------------------------------------------------------------ |
| 橘色       | 交换缓冲区          | 代表cpu在等待gpu完成工作,如果过高代表GPU需要完成的工作过多.  |
|            |                     |                                                              |
|            |                     |                                                              |
| 深蓝色     | 绘制                | 代表创建更新DisplayList的时间，过高代表在onDraw中花费过多时间,可能是自定义画图操作太多或执行了其它操作. |
| 淡绿色     | 测量/布局           | 代表了 onLayout 和 onMeasure 方法消耗的总时间，这段很高代表遍历整个view树结构花费了太多时间 |
|            |                     |                                                              |
|            |                     |                                                              |
| 墨绿色     | 其他时间/VSync 延迟 | 代表在连续两帧间的时间间隔,可能是因为子线程执行时间过长抢占了UI线程被cpu执行的机会. |

### Systrace

#### 使用

```bash
systrace工具位于：
android-sdk/platform-tools/systrace/

生成 html 报告：（Systrace does not support Python 3.7. Please use Python 2.7.）
python systrace.py [options] [categories]
```

options：

| options                                        | 解释                                                |
| :--------------------------------------------- | :-------------------------------------------------- |
| -o `<FILE`>                                    | 输出的目标文件                                      |
| -t N, –time=N                                  | 指定执行时间（秒），不指定则按Enter键结束           |
| -b N, –buf-size=N                              | buffer大小（单位KB),用于限制trace总大小，默认无上限 |
| -k `<KFUNCS`>，–ktrace=`<KFUNCS`>              | 追踪kernel函数，用逗号分隔                          |
| -a `<APP_NAME`>,–app=`<APP_NAME`>              | 追踪应用包名，用逗号分隔                            |
| –from-file=`<FROM_FILE`>                       | 从文件中创建互动的systrace                          |
| -e `<DEVICE_SERIAL`>,–serial=`<DEVICE_SERIAL`> | 指定设备                                            |
| -l, –list-categories                           | 列举可用的categories                                |

categories：

```bash
 gfx - Graphics
       input - Input
        view - View System
     webview - WebView
          wm - Window Manager
          am - Activity Manager
          sm - Sync Manager
       audio - Audio
       video - Video
      camera - Camera
         hal - Hardware Modules
         app - Application
         res - Resource Loading
      dalvik - Dalvik VM
          rs - RenderScript
      bionic - Bionic C Library
       power - Power Management
          pm - Package Manager
          ss - System Server
    database - Database
     network - Network
       sched - CPU Scheduling
        freq - CPU Frequency
        idle - CPU Idle
        load - CPU Load
  memreclaim - Kernel Memory Reclaim
  binder_driver - Binder Kernel driver
  binder_lock - Binder global lock trace
```

示例：

```bash
$ python systrace.py -o mynewtrace.html sched freq idle am wm gfx view \
binder_driver hal dalvik camera input res
```

APP自定义事件，需要指定 -a 包名选项

```java
Trace.beginSection("Event");
Trace.endSection();
```

如果您多次调用 `beginSection()`，调用 `endSection()` 只会结束最后调用的 `beginSection()` 方法。请务必将每次对 `beginSection()` 的调用与一次对 `endSection()` 的调用正确匹配。此外，不能在一个线程上调用 `beginSection()`，而在另一个线程上结束它；您必须在同一个线程上调用这两个方法。如果在Application beginSection() 并在MainActivity endSection() ，系统的 Event 会使自己定义 beginSection() 错误结束，所以需要另外搞个线程。

#### 分析

[浏览 Systrace 报告](https://developer.android.google.cn/topic/performance/tracing/navigate-report?hl=zh-cn)

### memory-profiler

[memory-profiler](https://developer.android.com/studio/profile/memory-profiler)

7.1版本及以下存在诸多限制，要在 8.0 之后才有比较完整的体验。这个工具可以如下一些事情：

* 记录应用程序一段时间内的内存分配情况，并提供图形化的界面分析工具，能够追踪到对象的分配堆栈以及跳转到对应的源码
* 捕获应用程序某个时间点的堆内存快照，例如可以在反复多次进出 Activity 之后进行捕获，如果堆快照中还存在该 Activity 实例则代表泄漏
* 保存 Android 格式的 .hprof（Heap Profiler ） 文件，可以用 android_sdk/platform-tools/ 的 hprof-conv 工具转换为 Java SE HPROF 格式

## 启动优化

[深入探索Android启动速度优化（上）](https://juejin.cn/post/6844904093786308622#heading-69)

[深入探索Android启动速度优化（下）](https://juejin.cn/post/6870457006784774152)

[一线大厂资深APP性能优化系列-异步优化与拓扑排序（二）](https://juejin.cn/post/6844904152670142477)

### windowBackground

预设Window背景，避免启动白屏，增加Splash页面

### 懒加载

按需初始化，特别是**针对于一些应用启动时不需要初始化的库**，可以等到用时才进行加载；使用 IdleHandler 进行懒加载

### 异步初始化

* 启动器思想，每个需要初始化的组件抽象为一个 Task 实例
* 指定各个 Task 之间的依赖关系，构造一个 Task 的有向无环图，通过拓扑排序优化确定 Task 的初始化顺序 
* 线程池并发执行各个 Task 的初始化，通过同步手段（例如CountDownLatch）保证 Task 执行的依赖关系
* CPU密集型任务应配置尽可能小的线程，如配置CPU数目+1个线程的线程池。由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如 2*CPU 数目。

实践经验：

确定是否每个组件的异步初始化耗时是大致平均的，这样更有优化的意义，如果总体的耗时时间很长，但是只有某个组件初始化耗时很长，其它组件的初始化耗时很少，则优化的效果不明显，发挥不出并发初始化的优势。

### 黑科技

* 启动阶段抑制GC
* CPU锁频
* 数据重排 [facebook redex](https://github.com/facebook/redex)



## 布局优化



## 内存优化

移动设备采用的 RAM 是 LPDDR（Low Power DDR），即“低功耗双倍数据速率内存”，相比于 DDR 来说功耗更低。

### 内存查看

Android设备出厂以后，java虚拟机对单个应用的最大内存分配就确定下来了，超出这个值就会OOM。这个属性值是定义在/system/build.prop文件中的

[关于heapsize & heapgrowthlimit](https://www.jianshu.com/p/766a380a9ae8)

```bash
➜  ~ adb shell cat /system/build.prop | grep heap 
//dalvik虚拟机堆初始大小
dalvik.vm.heapstartsize=16m
//单个进程在运行中能使用的最大内存
dalvik.vm.heapgrowthlimit=192m
//heapgrowthlimit属性不存在时的单个进程可用的最大内存
//如果heapgrowthlimit存在，可以通过指定android:largeHeap为true来使用该值
dalvik.vm.heapsize=512m
//当前理想的堆内存利用率
dalvik.vm.heaptargetutilization=0.75
//单次Heap内存调整的最小值
dalvik.vm.heapminfree=512k
//单次Heap内存调整的最大值.
dalvik.vm.heapmaxfree=8m
```



### LMK机制

[Android LowMemoryKiller原理分析](http://gityuan.com/2016/09/17/android-lowmemorykiller/)

在Android中，及时用户退出当前应用程序后，应用程序还是会存在于系统当中，这是为了方便程序的再次启动。但是这样的话，随着打开的程序的数量的增加，系统的内存就会不足，从而需要杀掉一些进程来释放内存空间。至于是否需要杀进程以及杀什么进程，这个就是由Android的内部机制LowMemoryKiller机制来进行的。

LMK每隔一段时间就会去检测当前空闲内存是否低于某个阈值(minfree)，如果是，则根据内存阈值(minfree)、杀进程的等级(adj)、进程的 oom_adj 值以及内存占用大小来确定需要杀死的进程，从而使内存恢复低于阈值的状态。LowMemoryKiller的值的设定，主要保存在2个文件之中，分别是/sys/module/lowmemorykiller/parameters/adj与/sys/module/lowmemorykiller/parameters/minfree。adj保存着当前系统杀进程的等级，minfree则是保存着对应的阀值。他们的对应关系如下：

```bash
➜  ~ adb shell
shamu:/ # cat /sys/module/lowmemorykiller/parameters/adj                                                                                                       
0,100,200,300,900,906
shamu:/ # cat /sys/module/lowmemorykiller/parameters/minfree(单位：4K)
18432,23040,27648,32256,36864,46080
```

查看进程的 OOM_ADJ 值（`cn.thismj.android.demo`按 Home 键进入后台）：

```bash
➜  ~ adb shell
shamu:/ $ su
shamu:/ # ps | grep demo
u0_a120   13539 7196  1183600 65168 SyS_epoll_ b0d24514 S cn.thismj.android.demo
u0_a114   15165 7196  1142444 60016 SyS_epoll_ b0d24514 S cn.yunzhisheng.prodemo
shamu:/ # ls /proc/13539 | grep oom
oom_adj
oom_score
oom_score_adj
shamu:/ # cat proc/13539/oom_adj                                                                                                                               
11
shamu:/ # cat proc/13539/oom_score                                                                                                                             
721
shamu:/ # cat proc/13539/oom_score_adj                                                                                                                         
700
```

系统关于 OOM_ADJ 的范围设定：

```c++
#define OOM_SCORE_ADJ_MIN (-1000)
#define OOM_SCORE_ADJ_MAX 1000
#define OOM_DISABLE (-17)
#define OOM_ADJUST_MIN (-16)
#define OOM_ADJUST_MAX 15
#endif
```

`oom_score_adj`：LMK使用这个值来确定杀死哪些进程，范围 [-1000,1000]

`oom_adj`：为了和旧版本的内核兼容，范围 [-17,15]，修改这个值的时候，内核会直接进行换算，将结果反映到`oom_score_adj`

`oom_score_adj` 一些常用值：

| FOREGROUND_APP_ADJ     | 0     | 前台进程                     |
| ---------------------- | ----- | ---------------------------- |
| VISIBLE_APP_ADJ        | 100   | 可见进程                     |
| PERCEPTIBLE_APP_ADJ    | 200   | 不可见，但是用户可察觉       |
| BACKUP_APP_ADJ         | 300   | 托管备份的进程               |
| SERVICE_ADJ            | 500   | 具有后台服务的app进程        |
| HOME_APP_ADJ           | 600   | 桌面进程                     |
| PREVIOUS_APP_ADJ       | 700   | 上一个跟用户交互的进程       |
| CACHED_APP_MIN_ADJ     | 900   | 缓存进程最小adj值            |
| CACHED_APP_MAX_ADJ     | 906   | 缓存进程最大adj值            |
| PERSISTENT_SERVICE_ADJ | -700  | 关联着系统或persistent进程   |
| PERSISTENT_PROC_ADJ    | -800  | 系统persistent进程，比如电话 |
| SYSTEM_ADJ             | -900  | system_server进程            |
| NATIVE_ADJ             | -1000 | native进程，不受系统管理     |

如何查看进程的 oom_adj 值：

### 内存优化的意义

	1. 防止OOM，提高应用稳定性
	2. 减少内存占用，降低被LMK机制杀死的概率
	3. 避免不合理使用内存导致GC太频繁，造成应用卡顿
	4. 减少异常发生和代码逻辑隐患

### 内存泄漏

#### 发生场景
1. 显式引用对象在其生命周期之外

   * 静态引用或者在单例中引用Context、Fragment、View等具有特定生命周期的对象；
2. 非静态内部类、匿名内部类默认持有外部类的引用

   * 继承自 `Handler` 的非静态内部类，持有外部类 Activity 的引用，而其 `MessageQueue` 持有的 `Message` 对象又通过 `target` 成员变量持有 `Handler` 的引用，或者通过 post 方法添加的 Runnable 匿名内部类对象也持有 Activity 的引用，当 `Handler` 消息没有处理完毕，而 Activity 已经结束生命周期时就会导致泄漏；
   * 使用匿名内部类方式实现 Thread、AsyncTask、Runnable 以及各类接口 Callback 等，由于它持有外部类的引用，如果外部类生命周期结束但其任务没有处理完毕，则会泄漏外部类实例；
   * RecyclerView的Adapter作为Fragment的成员变量场景。Fragment添加到栈中，只会销毁View，不会回收Fragment对象，所以Adapter对象也不会被回收，但是 Adapter->mObservable(AdapterDataObservable RV的静态内部类)->mObserver(RecyclerViewDataObserver RV的非静态内部类，持有RecyclerView的引用)，最终导致无法回收RecyclerView对象，造成RecyclerView的泄漏。
3. 集合类添加元素后，没有及时清理，导致无法被回收

   * `registerReceiver` 后没有及时 `unregisterReceiver`,（LoadedApk里面有个 mReceivers 的集合，里面持有 BroadcastReceiver 的引用，导致 Activity 无法被回收；AMS里面也会持有对端的引用）
4. IO流、数据库游标Cursor使用完毕后没有及时关闭；
5. 属性动画持有 View 的引用，没有在 Activity 销毁之时及时取消；

### 内存抖动

#### 发生场景

* 循环体内使用 + 拼接字符串或者不断地创建局部变量
* onDraw() 、getView() 中每次都创建新的对象
* 对于需要频繁创建和回收的对象没有建立全局缓存池（Handler的Message就是使用了池的设计思想）

### 内存优化

* 使用高性能的数据结构，key为 int 且数据量不大时用 SparseArray 替代 HashMap（省去装箱过程，提升效率，无需储存额外的hashcode、Entry对象等，节省内存）
* Bitmap的优化，缓存、inBitmap，inSampleSize等，图片放到 nodpi 防止缩放
* 对象复用，创建对象池








