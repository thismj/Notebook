# Android虚拟机

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



