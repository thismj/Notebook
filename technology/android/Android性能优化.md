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

实践经验：

确定是否每个组件的异步初始化耗时是大致平均的，这样更有优化的意义，如果总体的耗时时间很长，但是只有某个组件初始化耗时很长，其它组件的初始化耗时很少，则优化的效果不明显，发挥不出并发初始化的优势。

### 黑科技

* 启动阶段抑制GC
* CPU锁频
* 数据重排 [facebook redex](https://github.com/facebook/redex)

## 内存优化

移动设备采用的 RAM 是 LPDDR（Low Power DDR），即“低功耗双倍数据速率内存”，相比于 DDR 来说功耗更低。

### 内存优化的意义

	1. 防止OOM，提高应用稳定性
 	2. 减少内存占用，降低被LMK机制杀死的概率
 	3. 避免不合理使用内存导致GC太频繁，造成应用卡顿
 	4. 减少异常发生和代码逻辑隐患

