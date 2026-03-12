---
layout: default
title: CC算法的时间分解
nav_order: 7
parent: GC算法和实现
grand_parent: AndroidRuntime
author: Anonymous Committer
---

通过阅读 C++ 源码中 `TimingLogger::ScopedTiming` 的打点位置，我们可以把收集到的时间数据严格映射到 CC 算法的生命周期中。核心执行流程由 `ConcurrentCopying::RunPhases()` 函数统领。

源码中大量使用了 `TimingLogger::ScopedTiming split("PhaseName", GetTimings());`。这是 ART 内部的一个树状计时器。当你看到 `Total`（代码里直接叫 split 的名字）时，它代表整个块的运行时间；而 `Exclusive` 就是 ART 的打点系统在输出日志时，自动用 `Total` 减去它内部嵌套的所有子 `split` 耗时后计算出来的。

## 1. 准备阶段 (Concurrent)

这是 GC 刚启动时的初始化工作，此时应用线程仍在正常运行。

* **`InitializePhase_Total_s`**: 整个初始化阶段的总耗时。在这里 GC 会重置内部状态，判断本次 GC 是否需要强制全量疏散 (`force_evacuate_all_`)，并绑定各类内存空间的 Bitmap。
* **`MarkZygoteLargeObjects_s`**: 位于 `InitializePhase` 内部。GC 会遍历大对象空间 (Large Object Space)，显式标记那些属于 Zygote 的大对象，以防止它们被意外回收。

## 2. 并发标记阶段 (Concurrent)

如果是全量 GC (Full GC) 而不是分代年轻代 GC，就会执行此阶段。

* **`MarkingPhase_Total_s`**: 标记阶段总耗时。
* **`ScanImmuneSpaces_s`**: 扫描“免疫空间”（如 Image Space 和 Zygote Space），这些空间的对象通常不需要被回收，但需要扫描它们指向其他空间的引用。
* **`VisitConcurrentRoots_s` & `VisitNonThreadRoots_s`**: 扫描虚拟机运行时的全局根节点（非线程特定的根）。
* **`CaptureThreadRootsForMarking_s`**: 触发一个 Checkpoint，让所有应用线程短暂挂起，将其局部的根节点引用捕获到标记栈中。

## 3. 翻转与 STW 阶段 (Stop-The-World)

**真正的全局卡顿 (STW) 发生在这个阶段**。

* **`FlipThreadRoots_Total_s`**: 整个线程根节点翻转的生命周期总时间。
* **`(Paused)FlipCallback_Total_s`**: 核心的 STW 操作阶段。源码中明确写有 `Locks::mutator_lock_->AssertExclusiveHeld(self);`，意味着此时 GC 线程**独占**了全局锁，所有应用线程被强行暂停。
* **`(Paused)SetFromSpace_s`**: 将当前被占用的区域标记为 `From-Space`，为后续的拷贝腾出逻辑空间。
* **`(Paused)GrayAllNewlyDirtyImmuneObjects_Total_s`**: 极速扫描在 STW 前夕刚刚被应用线程“弄脏”的免疫对象，防止并发执行时发生漏标。
* **`(Paused)ClearCards_s`**: 在扫描完脏卡片后，将其清理。

## 4. 并发拷贝与处理阶段 (Concurrent)

STW 结束，应用线程恢复运行，GC 线程在后台默默搬运对象。

* **`CopyingPhase_Total_s`**: 并发拷贝阶段总耗时。
* **`ScanCardsForSpace_s`**: 扫描之前标记的老年代或未疏散区域中的脏卡片，追踪跨代/跨区引用。
* **`Process_mark_stacks_and_References_Total_s`**: 处理标记栈并处理各种引用（软引用、弱引用等）。
* **`ProcessReferences_Total_s`**: 处理弱引用，此阶段会短暂禁用弱引用访问。
* **`SweepSystemWeaks_s`**: 清理系统级别的弱引用（如 JNI WeakGlobalRefs）。

## 5. 内存回收阶段 (Concurrent)

对象搬运完毕，开始真正释放不再使用的内存。

* **`ReclaimPhase_Total_s`**: 回收阶段总耗时。
* **`Sweep_Total_s` / `SweepAllocSpace_s` / `SweepLargeObjects_s`**: 遍历并清理那些没有被拷贝到 `To-Space` 的死对象所占用的内存，包括普通分配空间和大对象空间。
* **`RecordFree_Total_s`**: 记录释放的内存量，计算回收比例。此时 `region_space_->ClearFromSpace` 会被调用。
* **`ClearFromSpace_s`**: 物理上清空 `From-Space` 区域的数据。
* **`Release_free_regions_s`**: 如果系统内存紧张，会在此刻通过 `madvise` 系统调用，急切地将空闲物理内存归还给 Linux 内核。

## 6. 收尾阶段 (Concurrent)

* **`EmptyRBMarkBitStack_s`**: 位于 `FinishPhase` 中，用于清空 Baker 读屏障所使用的标记位栈，重置状态以备下一次 GC 使用。
