# Android虚拟机

## 64K引用限制跟 LinearAlloc 问题

`Android`应用 `APK` 中包含 `class.dex` 文件，`Android`虚拟机规范规定单个`DEX` 文件可引用的方法总数限制为`65536`,这个限制不是因为`DEX`文件格式的限制，而是虚拟机的 `invoke-kind`指令集中指定方法索引的字段只有`16`位，所以只能调用`DEX`文件的前`65535`个方法[Dalvik 字节码](https://source.android.google.cn/devices/tech/dalvik/dalvik-bytecode?hl=zh-cn)

在 `Android 5.0`以前，`Dalvik`虚拟机可以通过`MultiDex`的库解决该问题，这个库会成为主`DEX`文件的一部分，然后管理对其他`DEX`文件的访问：

```groovy
android {
    defaultConfig {
        ...
        minSdkVersion 15
        targetSdkVersion 28
        multiDexEnabled true
    }
    ...
}

dependencies {
  //依赖 MultiDex 库，然后把 Application 替换为 MultiDexApplication，或者在原有 Application 的 attchBaseContext 方法执行 MultiDex.install(this) 即可。
  implementation "androidx.multidex:multidex:2.0.1"
}
```

在`Android 5.0`以及之后，`Android`虚拟机使用`ART`，本身就支持从`APK`中加载多个`DEX`文件，所以会默认启用`MultiDex`。当构建应用时，`Android`构建工具会根据需要构造主`DEX`文件 (`classes.dex`) 和辅助`DEX`文件（`classes2.dex` 和 `classes3.dex` 等）。然后，构建系统会将所有`DEX`文件打包到`APK`中。

对于`Dalvik`虚拟机来说，在应用的安装过程中会执行一个`dexopt`的程序，它会使用`LinearAlloc`来储存应用的方法信息，`LinearAlloc`是一个固定大小的缓冲区，在`Android 2.3`以及以下版本只有`5MB`（`4.x`版本提高到了`8MB`/`16MB`），当`DEX`文件中方法数过多时，会导致`dexopt`崩溃从而引发`INSTALL_FAILED_DEXOPT`。该问题主要发生在低版本上(`2.3`)，解决办法是在`dx`时通过设置`--set-max-idx-number`指定单个`DEX`文件的最大方法数。

`Dalvik`使用`MultiDex`有哪些问题（`ART`下采用`Ahead-of-time AOT`，会通过`dex2oat`把`DEX`文件编译生成本地机器可运行的`oat`文件，所以不存在下述问题）：

* 在冷启动时因为需要安装DEX文件，如果DEX文件过大时，处理时间过长，很容易引发ANR（Application Not Responding）
* 采用MultiDex方案的应用可能不能在低于Android 4.0 (API level 14) 机器上启动，这个主要是因为Dalvik linearAlloc的一个bug ([Issue 22586](http://b.android.com/22586))
* 采用MultiDex方案的应用因为需要申请一个很大的内存，在运行时可能导致程序的崩溃，这个主要是因为Dalvik linearAlloc 的一个限制([Issue 78035](http://b.android.com/78035))





## Dalvik

## ART
[ART运行时Java堆创建过程分析](https://blog.csdn.net/luoshengyang/article/details/42379729)

查看 ART GC 日志消息，ART 只有在 GC 暂停时间超过 5ms 或者持续时间超过 100ms 时才输出 GC 消息，消息格式如下：

```bash
I/art: GC_Reason GC_Name Objects_freed(Size_freed) AllocSpace Objects,
    Large_objects_freed(Large_object_size_freed) Heap_stats LOS objects, Pause_time(s)
```



| GC_Reason  | 解释                              |
| ---------- | --------------------------------- |
| Concurrent | 在后台线程并发 GC，不影响内存分配 |
| Alloc | 应用进程堆已满时触发的 GC，会影响内存分配 |
| Explicit | 应用调用 `System.gc()` 主动触发的 GC |
| NativeAlloc | native内存分配过重时的 GC（Bitmap、RenderScript）|
| CollectorTransition | 运行时更改 GC 策略触发 |
| HomogeneousSpaceCompact | homogeneous space 压缩，对堆进行碎片整理 |
| DisableMovingGc | 并发堆压缩时使用了GetPrimitiveArrayCritical，GC 遭到阻塞 |
| HeapTrim | 堆整理时，GC 遭到阻塞 |


| GC_Name  | 解释                              |
| ---------- | --------------------------------- |
| Concurrent mark sweep (CMS) | 除了 image space 之外都回收 |
| Concurrent partial mark sweep | 除了 image space 跟 zygote space 之外都回收 |
| Concurrent sticky mark sweep | 分代收集，只回收自上次 GC 后分配的对象，比前两种更频繁、更快速、暂停时间更短 |
| Marksweep + semispace | 用于堆转换和 homogeneous space 压缩(对堆进行碎片整理)的非并发、复制的GC。|


| 其它  | 解释                              |
| ---------- | --------------------------------- |
| Objects freed | non large object space 回收的对象数 |
| Size freed| non large object space 回收的内存大小（bytes） |
| Large objects freed |  large object space 回收的对象数 |
| Large object size freed | large object space 回收的内存大小（bytes）|
| Heap stats | 堆的可用空间百分比和 活跃对象内存大小/堆总大小|
| Pause times | GC 暂停时间，CMS 只在即将完成时暂停一次，移动 GC 暂停的时间持续比较长|



