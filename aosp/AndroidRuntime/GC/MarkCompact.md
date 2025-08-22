---
layout: default
title: MarkCompact
nav_order: 0
parent: GC算法和实现
grand_parent: AndroidRuntime
author: Anonymous Committer
---


## 1. 实际移动对象的代码在哪里？

在 ART 的并发 Mark Compact GC 中，对象的移动（Compaction）是一个非常精巧的过程。它并不是简单地在暂停期间移动所有对象，而是利用了 Linux 内核的一个高级特性 `userfaultfd` 来实现**并发移动**。

实际的内存拷贝代码主要在以下两个函数中，它们都使用了 `memcpy` 来执行最底层的内存复制操作：

1. **`MarkCompact::CompactPage()`**

      * **作用**：这个函数负责整理（compact）“普通”的存活对象页。这些是 GC 标记阶段发现的、在标记暂停前就已存在的对象。
      * **核心逻辑**：它并不逐个对象地移动，而是先通过 `live_words_bitmap_` 找到**连续的存活内存块（Live Strides）**，然后一次性拷贝一个“块”。这是一种优化，可以减少 `memcpy` 的调用次数。
      * **代码位置**：在 `CompactPage` 函数内部，`live_words_bitmap_->VisitLiveStrides(...)` 的 lambda 表达式中：

        ```cpp
        // ... Inside VisitLiveStrides lambda ...
        memcpy(addr, from_space_begin_ + stride_begin * kAlignment, stride_in_bytes);
        ```

        这里的 `memcpy` 就是将存活对象从临时的 `from_space` 拷贝到它们在 `moving_space` 中的新位置。

2. **`MarkCompact::SlideBlackPage()`**

      * **作用**：这个函数负责移动那些在“标记暂停”后由应用分配的新对象（所谓的“黑色对象”），这些对象通常位于线程本地分配缓冲区（TLABs）中。
      * **核心逻辑**：因为这些对象默认全部存活，所以这个过程更像是一个“滑动”（Slide）。它将几块连续分配的对象内存（中间可能有 TLAB 的空隙）紧凑地复制到新的位置。
      * **代码位置**：在 `SlideBlackPage` 函数中，同样是通过 `memcpy` 实现：

        ```cpp
        std::memcpy(dest, src_addr, first_chunk_size);
        // ...
        std::memcpy(dest, src_addr, remaining_bytes);
        ```

**关键点**：这些函数并不是在最终的 STW (Stop-The-World) 暂停中被 GC 主线程调用的。而是在应用线程恢复运行后，当它们访问一个尚未被移动的对象时，触发一个**缺页异常 (Page Fault)**，然后由异常处理函数 `SigbusHandler` -\> `ConcurrentlyProcessMovingPage` 来调用它们。GC 主线程自己也会在后台主动调用 `CompactMovingSpace` 来推进这个过程。

---

## 2. 具体的 GC 实现流程讲解

下面我们以 `RunPhases()` 函数为脉络，梳理整个 Mark Compact GC 的生命周期。

### Phase 1: `InitializePhase` - 初始化阶段

这是 GC 的准备阶段。

* **重置状态**：清空标记栈 (`mark_stack_`)、免疫空间列表 (`immune_spaces_`) 和各种计数器。
* **设定边界**：记录当前堆的末尾位置，标记暂停后分配的对象都将被视为“黑色”（默认存活）。
* **计算地址偏移**：计算出 `moving_space`（当前堆）和 `from_space`（用于整理的临时源空间）之间的地址差 `from_space_slide_diff_`。

### Phase 2: `MarkingPhase` - 并发标记阶段

这是 GC 的核心工作之一，大部分时间与应用程序**并发执行**。

* `PrepareCardTableForMarking`: 准备卡表（Card Table），用于记录并发期间被应用修改过的内存区域（“脏卡”）。
* `MarkRoots`: 短暂暂停（STW），从根（线程栈、全局变量等）开始标记。
* `MarkReachableObjects`: 从根出发，遍历整个对象图，标记所有存活对象。这是最耗时的部分。
* `PreCleanCards`: **关键优化**。在并发标记的同时，GC 线程会反复扫描并处理脏卡，这能极大减少最终暂停所需处理的工作量，从而缩短暂停时间。

### Phase 3: `MarkingPause` - 标记暂停阶段

一个**短暂的 STW 暂停**，用于确保标记的最终一致性。

* **同步状态**：暂停所有应用线程，并最后一次扫描它们的根。
* **处理脏卡**：`RecursiveMarkDirtyObjects`，处理在 `PreCleanCards` 之后新产生的脏卡。
* **处理特殊引用**：`GetHeap()->GetReferenceProcessor()->EnableSlowPath()`，为处理 `WeakReference`、`SoftReference` 等做准备。
* **最终确定存活集**：此时，所有存活对象都已被准确标记。

### Phase 4: `ReclaimPhase` & `PrepareForCompaction` - 回收与整理准备阶段

* `ReclaimPhase`:
  * **回收非移动空间**：对大对象空间（LOS）和非移动空间（Non-moving space）执行传统的标记-清除（Mark-Sweep）。
  * **处理引用队列**：处理 `WeakReference` 等，将可回收的引用放入队列。
* `PrepareForCompaction`:
  * **计算存活密度**：分析 `moving_space` 中对象的存活密度。如果某块区域的对象存活率非常高（例如超过95%），则将其标记为\*\*“黑色密集区” (`black_dense_end_`)**。这部分区域的对象将**不被移动\*\*，只更新其内部指针。
  * **计算新地址**：这是最关键的一步。通过 `live_words_bitmap_` 和 `chunk_info_vec_`，对存活字节进行一次**前缀和（`std::exclusive_scan`）计算**。完成后，`chunk_info_vec_` 就成了一张“地址映射表”，可以通过任何一个对象的旧地址，瞬间计算出它的新地址。
  * **计算滑动距离**：计算出黑色对象（新分配的对象）需要整体滑动的距离 `black_objs_slide_diff_`。

### Phase 5: `CompactionPause` & `CompactionPhase` - 整理暂停与并发整理

这部分是整个算法最精妙的地方，它将一个漫长的“移动对象”暂停，拆解成一个短暂的“准备移动”暂停和后续的并发移动。

* **`CompactionPause` (短暂 STW 暂停)**:

    1. **更新根引用**：`FlipThreadRoots`，遍历所有根，使用 `chunk_info_vec_` 计算并更新它们指向的新地址。这是所谓的“指针翻转（Flip）”。
    2. **更新其他空间引用**：更新非移动空间、免疫空间等区域内指向移动对象的引用。
    3. **调整 TLAB**：根据 `black_objs_slide_diff_` 调整每个线程的 TLAB 指针。
    4. **`KernelPreparation` (内核准备)**：**魔术发生的地方！** 🎩
          * 使用 `mremap` 系统调用，将 `moving_space` 的物理内存页**原子地移动**到 `from_space_map_` 的虚拟地址上。此时，`moving_space` 的虚拟地址空间被保留，但背后不再有物理内存。
          * 使用 `userfaultfd` 系统调用，向内核**注册** `moving_space` 的虚拟地址范围，告诉内核：“如果有人访问这片内存，不要报错，而是通知我。”
    5. **恢复应用线程**：暂停结束，应用恢复执行。

* **`CompactionPhase` (并发整理)**:

    1. **触发缺页异常**：当一个恢复运行的应用线程尝试访问 `moving_space` 里的某个对象时，由于物理内存已经不在那里，CPU 会产生一个**缺页异常 (Page Fault)**。
    2. **内核通知 ART**：内核通过 `userfaultfd` 机制捕获这个异常，并唤醒 ART 的异常处理线程 (`SigbusHandler`)。
    3. **按需整理**：处理线程收到通知后，定位到发生异常的那个内存页，然后调用我们之前提到的 `CompactPage` 或 `SlideBlackPage`。
          * 它从 `from_space_map_` 中拷贝出存活对象数据到一块**临时缓冲区**。
          * 在缓冲区内更新所有对象的内部引用。
          * 使用 `UFFDIO_COPY` `ioctl` 命令，让内核将修复好的页面内容**复制回** `moving_space` 中发生异常的地址。
    4. **异常解决**：内核操作完成后，应用线程从异常中恢复，它对内存的访问成功了，并且它看到的是已经整理好、指针也已修复的新对象。整个过程对应用线程是**透明的**。
    5. **后台推进**：与此同时，GC 主线程自己也在后台循环调用 `CompactMovingSpace`，主动地整理那些尚未被访问的页面，以加快整体进度。

### Phase 6: `FinishPhase` - 结束阶段

* **清理**：取消 `userfaultfd` 注册，释放 `from_space_map_` 和其他临时数据结构。
* **重置状态**：为下一次 GC 做好准备。

这个流程通过 `userfaultfd` 将繁重的对象移动和指针修复工作分摊到了实际访问时，化整为零，从而避免了因全堆整理而导致的长时间应用卡顿。
