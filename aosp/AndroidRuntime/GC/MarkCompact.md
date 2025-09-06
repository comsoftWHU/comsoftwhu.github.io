---
layout: default
title: MarkCompact
nav_order: 0
parent: GC算法和实现
grand_parent: AndroidRuntime
author: Anonymous Committer
---


## TL;DR MarkCompact GC

```mermaid

```

## 具体的 MarkCompact GC 实现

下面我们以 `RunPhases()` 函数为脉络，梳理整个 Mark Compact GC 的生命周期。

```cpp
void MarkCompact::RunPhases() {
  Thread* self = Thread::Current();
  thread_running_gc_ = self;
  Runtime* runtime = Runtime::Current();
  InitializePhase();
  GetHeap()->PreGcVerification(this);
  {
    ReaderMutexLock mu(self, *Locks::mutator_lock_);
    MarkingPhase();
  }
  {
    // Marking pause
    ScopedPause pause(this);
    MarkingPause();
    if (kIsDebugBuild) {
      bump_pointer_space_->AssertAllThreadLocalBuffersAreRevoked();
    }
  }
  bool perform_compaction;
  {
    ReaderMutexLock mu(self, *Locks::mutator_lock_);
    LOG(INFO) << "Reclaim phase starting";
    ReclaimPhase();
    perform_compaction = PrepareForCompaction();
  }

  if (perform_compaction) {
    // Compaction pause
    ThreadFlipVisitor visitor(this);
    FlipCallback callback(this);
    LOG(INFO) << "Flipping thread roots";
    runtime->GetThreadList()->FlipThreadRoots(
        &visitor, &callback, this, GetHeap()->GetGcPauseListener());

    if (IsValidFd(uffd_)) {
      ReaderMutexLock mu(self, *Locks::mutator_lock_);
      LOG(INFO) << "Compaction phase starting with userfaultfd";
      CompactionPhase();
    }
  }
  FinishPhase();
  GetHeap()->PostGcVerification(this);
  thread_running_gc_ = nullptr;
}
```

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
  * **计算存活密度**：分析 `moving_space` 中对象的存活密度。如果某块区域的对象存活率非常高（例如超过95%），则将其标记为**黑色密集区 (`black_dense_end_`)**。这部分区域的对象将**不被移动\*\*，只更新其内部指针。
  * **计算新地址**：这是最关键的一步。通过 `live_words_bitmap_` 和 `chunk_info_vec_`，对存活字节进行一次**前缀和（`std::exclusive_scan`）计算**。完成后，`chunk_info_vec_` 就成了一张“地址映射表”，可以通过任何一个对象的旧地址，瞬间计算出它的新地址。
  * **计算滑动距离**：计算出黑色对象（新分配的对象）需要整体滑动的距离 `black_objs_slide_diff_`。

### Phase 5: `FlipThreadRoots` & `CompactionPhase` - 整理暂停与并发整理

#### FlipThreadRoots

`ThreadList::FlipThreadRoots` 安全、原子性地将所有线程的根引用（Thread Roots）从指向旧空间（from-space）“翻转”到指向新空间（to-space）。

```cpp
// 这是一个“检查点/全体暂停”混合模式的函数，用于将线程的根引用从
// from-space（旧空间）切换到 to-space（新空间）。
// 它通过同步所有线程，来标志着（并发）标记阶段的正式开始，
// 同时维护了“to-space 不变性”（即一旦翻转完成，所有代码只能看到指向 to-space 的引用）。
void ThreadList::FlipThreadRoots(Closure* thread_flip_visitor,
                                 Closure* flip_callback,
                                 gc::collector::GarbageCollector* collector,
                                 gc::GcPauseListener* pause_listener) {
  TimingLogger::ScopedTiming split("ThreadListFlip", collector->GetTimings());
  Thread* self = Thread::Current();
  // ... 一系列锁状态的断言 ...
  CHECK_NE(self->GetState(), ThreadState::kRunnable);

  // 1. 准备阶段：与JNI临界区代码同步，为暂停做准备。
  collector->GetHeap()->ThreadFlipBegin(self);

  const uint64_t suspend_start_time = NanoTime();
  VLOG(threads) << "Suspending all for thread flip";
  {
    ScopedTrace trace("ThreadFlipSuspendAll");
    // 2. 执行暂停：暂停虚拟机中所有其他应用线程（Stop-The-World）。这是暂停的开始。
    SuspendAllInternal(self);
  }

  std::vector<Thread*> flipping_threads;  // 存储所有需要被翻转的线程。
  int thread_count;
  // 用于处理线程在翻转过程中退出的竞争条件。
  std::unique_ptr<ThreadExitFlag[]> exit_flags;

  {
    TimingLogger::ScopedTiming t("FlipThreadSuspension", collector->GetTimings());
    if (pause_listener != nullptr) {
      // 通知监听器，GC暂停开始。
      pause_listener->StartPause();
    }

    // 3. 进入关键暂停区：获取独占的 mutator_lock_，完全阻止应用代码运行。
    Locks::mutator_lock_->ExclusiveLock(self);
    suspend_all_histogram_.AdjustAndAddValue(NanoTime() - suspend_start_time);
    // 执行由GC收集器传入的回调，通常用于交换堆空间等关键操作。
    flip_callback->Run(self);

    {
      MutexLock mu(self, *Locks::thread_list_lock_);
      MutexLock mu2(self, *Locks::thread_suspend_count_lock_);
      thread_count = list_.size();
      exit_flags.reset(new ThreadExitFlag[thread_count]);
      flipping_threads.resize(thread_count, nullptr);
      int i = 1;
      // 遍历所有线程，为它们设置“翻转函数”。
      for (Thread* thread : list_) {
        // 为每个线程“安装”翻转任务。线程被恢复后，必须先执行此函数才能继续。
        DCHECK(thread == self || thread->IsSuspended());
        thread->SetFlipFunction(thread_flip_visitor);
        // 把当前GC线程放在列表首位，以便优先处理。
        int thread_index = thread == self ? 0 : i++;
        flipping_threads[thread_index] = thread;
        // 注册一个退出标志，以处理线程在翻转过程中退出的竞争条件。
        thread->NotifyOnThreadExit(&exit_flags[thread_index]);
      }
      DCHECK(i == thread_count);
    }

    if (pause_listener != nullptr) {
      pause_listener->EndPause();
    }
  }
  
  {
    MutexLock mu(self, *Locks::thread_list_lock_);
    Locks::thread_suspend_count_lock_->Lock(self);
    // 4. 恢复所有线程。线程醒来后会发现有翻转任务等待执行，从而被“困住”，无法立即执行应用代码。
    ResumeAllInternal(self);
  }
  // 记录本次暂停的总时长。
  collector->RegisterPause(NanoTime() - suspend_start_time);

  // --- “翻转竞赛”开始 ---
  // 所有线程（包括本GC线程）都会尝试执行翻转函数。
  // 通过一个原子状态确保每个线程的翻转函数只会被成功执行一次。

  // 尝试在其他线程上运行翻转闭包。
  TimingLogger::ScopedTiming split3("RunningThreadFlips", collector->GetTimings());
  
  // 5. GC线程重新获取共享锁。此时其他线程都被翻转任务阻塞，所以能无竞争地获取。
  AcquireMutatorLockSharedUncontended(self);

  Locks::thread_suspend_count_lock_->Unlock(self);
  
  // JNI同步结束。
  collector->GetHeap()->ThreadFlipEnd(self);

  // 6. 开始“翻转竞赛”：GC线程主动尝试为其他线程启动翻转函数，以加速进程。
  for (int i = 0; i < thread_count; ++i) {
    bool finished;
    // 这是一个非阻塞的尝试。
    Thread::EnsureFlipFunctionStarted(
        self, flipping_threads[i], Thread::StateAndFlags(0), &exit_flags[i], &finished);
    if (finished) {
      // 如果翻转已完成，则注销其退出标志。
      MutexLock mu2(self, *Locks::thread_list_lock_);
      flipping_threads[i]->UnregisterThreadExitFlag(&exit_flags[i]);
      flipping_threads[i] = nullptr;
    }
  }
  
  // 7. 等待所有翻转完成：阻塞等待任何尚未完成的翻转任务，确保所有线程都已更新。
  for (int i = 0; i < thread_count; ++i) {
    if (UNLIKELY(flipping_threads[i] != nullptr)) {
      flipping_threads[i]->WaitForFlipFunctionTestingExited(self, &exit_flags[i]);
      MutexLock mu2(self, *Locks::thread_list_lock_);
      flipping_threads[i]->UnregisterThreadExitFlag(&exit_flags[i]);
    }
  }

  // 调试检查：确保所有退出标志都已被正确注销。
  Thread::DCheckUnregisteredEverywhere(&exit_flags[0], &exit_flags[thread_count - 1]);

  // 8. 清理并结束：释放共享锁，所有线程现在都可以在新的内存布局上自由运行。
  Locks::mutator_lock_->SharedUnlock(self);
}
```

##### CompactionPause

这个函数在 STW 暂停期间，按顺序执行了以下几个关键任务：

###### 1. 同步最新分配的对象

在并发标记和这次暂停之间，应用线程可能已经分配了新的对象（尤其是在 TLAB - 线程本地分配缓冲区中）。这些新对象默认都是存活的。

`UpdateMovingSpaceBlackAllocations()`: 扫描这些新分配的“黑色”对象，并将它们信息更新到 GC 的内部数据结构中（如 `live_words_bitmap_`），确保它们也会被正确处理。

`UpdateNonMovingSpaceBlackAllocations()`: 对非移动空间执行同样的操作。

`compacting_ = true;`: 设置一个全局标志位，通知系统的其他部分（尤其是 `userfaultfd` 信号处理器），整理阶段正式开始。

###### 2. 更新Immune Spaces
`Immune Spaces`是指那些自身不参与 GC 的特殊内存区域，例如 `Zygote` 空间和 `Boot Image` 空间。

为什么需要更新？：虽然这些空间本身不被回收，但它们内部的对象可能引用了普通堆（`Moving Space`）中的对象。当普通堆中的对象被移动后，这些引用就需要被更新。

如何高效更新？：通过**卡片表（Card Table）**或 **Mod-Union Table**。这些数据结构记录了免疫空间中哪些“卡片”（小的内存块）被“弄脏”（即被写入过）。GC 只需扫描这些被标记为“脏”的卡片，而不需要扫描整个巨大的免疫空间，从而极大地提高了效率。

###### 3. 全面根引用更新与特殊空间处理
这是暂停期间最繁重的工作：更新虚拟机中所有的根引用（GC Roots）。

`runtime->VisitConcurrentRoots(...)`, `runtime->VisitNonThreadRoots(...)`, `linker->VisitClassLoaders(...)` 等：这些调用会地毯式地扫描所有类型的根，包括 JNI 全局引用、类的静态字段、活动的类加载器等，并将其中所有指向旧地址的指针，全部更新为指向新地址。

`SweepSystemWeaks(...)`: 处理系统中的弱引用、软引用等。

处理 `LinearAllocSpace`：这是用于 ART 内部对象（如 ArtMethod）的特殊空间，其对象不被移动。

并发模式：在常规模式下，函数会收集所有需要处理的 `LinearAlloc` 内存块（Arenas），并将它们存入 `linear_alloc_arenas_` 列表。这些内存块的指针修复工作将在暂停结束之后，与应用线程并发地进行。

后备模式/Zygote 场景：在后备模式或 Zygote 首次 fork 前的特殊情况下，没有并发处理阶段，因此所有 `LinearAlloc` 空间的指针修复工作会直接在本次暂停内完成。

`KernelPreparation()`: 准备内核。这是为并发整理做准备的关键一步，它很可能会在这里通过 `ioctl` 调用来设置 `userfaultfd`，将需要并发整理的内存区域“注册”给内核。

###### 4. 准备并发或执行后备方案
`UpdateNonMovingSpace()`: 更新非移动空间中的对象引用。

后备模式 (`if (uffd_ == kFallbackMode)`): 如果 userfaultfd 不可用或初始化失败，GC 会放弃并发，进入“后备模式”。此时，它会调用 `CompactMovingSpace<kFallbackMode>`，在本次 STW 暂停内，直接完成所有对象的移动和整理工作。这会导致一次较长的 GC 暂停。

并发模式 (`else`): 在正常情况下，函数在完成所有准备工作后就结束了。真正的对象移动（`CompactMovingSpace<kCopyMode>`）将在暂停结束之后，由 GC 线程与恢复运行的应用线程并发地进行。

```cpp
void MarkCompact::CompactionPause() {
  // 本函数是 Mark-Compact GC 中最核心的“Stop-The-World” (STW) 暂停。
  // 在此期间，所有应用线程都被暂停，以便GC可以安全地完成所有
  // 无法并发执行的关键更新和准备工作，为对象整理阶段铺平道路。
  TimingLogger::ScopedTiming t(__FUNCTION__, GetTimings());
  Runtime* runtime = Runtime::Current();
  non_moving_space_bitmap_ = non_moving_space_->GetLiveBitmap();
  // ... 调试相关的栈地址记录 ...

  {
    // --- 阶段1：数据追赶 ---
    TimingLogger::ScopedTiming t2("(Paused)UpdateCompactionDataStructures", GetTimings());
    ReaderMutexLock rmu(thread_running_gc_, *Locks::heap_bitmap_lock_);
    // 刷新数据结构，以包含自并发标记暂停以来新分配的对象（“黑色对象”）。
    // 这些新对象（尤其是在TLAB中）默认都是存活的。
    UpdateMovingSpaceBlackAllocations();
    UpdateNonMovingSpaceBlackAllocations();
    
    // 设置全局标志位，通知其他部分（如uffd信号处理器）整理阶段已正式开始。
    compacting_ = true;
    // 执行一轮初步的根更新。
    heap_->GetReferenceProcessor()->UpdateRoots(this);
  }

  {
    // --- 阶段2：更新“免疫空间”（如Zygote空间、Boot Image空间）---
    // 这些空间本身不被GC，但它们内部的指针可能指向需要移动的对象，因此需要更新。
    TimingLogger::ScopedTiming t2("(Paused)UpdateImmuneSpaces", GetTimings());
    accounting::CardTable* const card_table = heap_->GetCardTable();
    for (auto& space : immune_spaces_.GetSpaces()) {
      // ...
      accounting::ModUnionTable* table = heap_->FindModUnionTableFromSpace(space);
      ImmuneSpaceUpdateObjVisitor visitor(this);
      if (table != nullptr) {
        // 优先使用 ModUnionTable，它更精确地记录了被修改的区域。
        table->ProcessCards();
        table->VisitObjects(ImmuneSpaceUpdateObjVisitor::Callback, &visitor);
      } else {
        // 如果没有 ModUnionTable，则回退到扫描卡片表（Card Table）来寻找“脏”卡片。
        WriterMutexLock wmu(thread_running_gc_, *Locks::heap_bitmap_lock_);
        card_table->Scan<false>(live_bitmap, space->Begin(), space->Limit(), visitor, ...);
      }
    }
  }

  {
    // --- 阶段3：全面根引用更新与特殊空间处理 ---
    TimingLogger::ScopedTiming t2("(Paused)UpdateRoots", GetTimings());
    // 地毯式扫描并更新所有非线程的根引用，如JNI全局/弱全局引用、类的静态字段等。
    runtime->VisitConcurrentRoots(this, kVisitRootFlagAllRoots);
    runtime->VisitNonThreadRoots(this);
    {
      // 更新所有类加载器及其类表中的根。
      ClassLinker* linker = runtime->GetClassLinker();
      ClassLoaderRootsUpdater updater(this);
      ReaderMutexLock rmu(thread_running_gc_, *Locks::classlinker_classes_lock_);
      linker->VisitClassLoaders(&updater);
      linker->GetBootClassTable()->VisitRoots(updater, /*skip_classes=*/true);
    }
    // 处理系统中的弱引用、软引用等。
    SweepSystemWeaks(thread_running_gc_, runtime, /*paused=*/true);

    bool has_zygote_space = heap_->HasZygoteSpace();
    GcVisitedArenaPool* arena_pool =
        static_cast<GcVisitedArenaPool*>(runtime->GetLinearAllocArenaPool());
    // 更新类表。
    UpdateClassTableClasses(runtime, uffd_ == kFallbackMode || !has_zygote_space);
    
    // 为处理LinearAllocSpace加锁，确保在处理期间没有新的Arena被分配或修改。
    WriterMutexLock pool_wmu(thread_running_gc_, arena_pool->GetLock());

    auto arena_visitor = [this](...) { /* ... 定义一个用于更新LinearAlloc页面的访问器 ... */ };
    
    // 根据模式决定如何处理 LinearAlloc 空间（ART内部对象，不移动，但需更新指针）。
    if (uffd_ == kFallbackMode || (!has_zygote_space && runtime->IsZygote())) {
      // 在后备模式或Zygote首次fork前，直接在暂停期间更新所有LinearAlloc Arena。
      arena_pool->VisitRoots(arena_visitor);
    } else {
      // 在并发模式下，收集需要并发处理的Arena，并推迟它们的释放以避免竞争。
      arena_pool->DeferArenaFreeing();
      // 遍历所有Arena，将需要并发处理的加入列表，其余的在暂停期间直接处理。
      arena_pool->ForEachAllocatedArena(
          [this, arena_visitor, has_zygote_space](const TrackedArena& arena) {
            // ... (根据arena类型决定是现在处理还是加入并发处理列表) ...
          });
    }

    // 重置SIGBUS信号计数器，为并发整理阶段做准备。
    sigbus_in_progress_count_[0].store(0, std::memory_order_relaxed);
    sigbus_in_progress_count_[1].store(0, std::memory_order_release);
    
    // 【关键】准备内核，通常是设置userfaultfd并注册需要并发整理的内存范围。
    KernelPreparation();
  }

  // --- 阶段4：准备并发或执行后备方案 ---
  // 更新非移动空间中的对象引用。
  UpdateNonMovingSpace();

  if (uffd_ == kFallbackMode) {
    // 【后备路径】如果userfaultfd不可用，则进入后备模式。
    // 在本次STW暂停内，直接完成所有对象的移动整理工作。这将导致一次较长的暂停。
    CompactMovingSpace<kFallbackMode>(nullptr);
    // 记录释放的内存。
    int32_t freed_bytes = black_objs_slide_diff_;
    bump_pointer_space_->RecordFree(freed_objects_, freed_bytes);
    RecordFree(ObjectBytePair(freed_objects_, freed_bytes));
  } else {
    // 【并发路径】正常情况下，暂停在此结束。真正的对象移动将在暂停后并发进行。
    DCHECK_EQ(compaction_buffer_counter_.load(std::memory_order_relaxed), 1);
  }
  stack_low_addr_ = nullptr;
}
```


#### CompactionPhase
可以把`CompactionPhase`的工作流程看作以下几个步骤：

1. 记录统计信息
2. 执行核心整理任务: `CompactMovingSpace<kCopyMode>(...)`: 调用 CompactMovingSpace 函数来处理**Moving Space**中的对象。绝大多数的 Java 对象都分配在这里。这个函数会把所有存活的对象紧凑地移动到空间的一端。`ProcessLinearAlloc()`: 处理线性分配空间。这里的对象通常不会被移动（例如 Class 对象、ArtMethod 等），但它们内部可能引用了“可移动空间”中的对象。因此，这个函数的主要任务是更新这些引用，让它们指向移动后的新地址。
3. 与应用线程（Mutator）同步: 使用了一个精巧的两阶段同步机制来确保 GC 线程和应用线程不会互相干扰。`wait_for_compaction_counter(0)`: 第一阶段等待。GC 线程在这里设置一个“整理完成”的标志位，并等待所有正在处理内存访问冲突（SIGBUS 信号）的应用线程完成它们当前的操作。`UnregisterUffd(...)`: 通知内核（通过 userfaultfd 机制）GC 不再关心这些内存区域的访问了。这是为并发整理做清理。`wait_for_compaction_counter(1)`: 第二阶段等待。GC 线程再次设置一个“清理完成”的标志位，并等待。
4. 清理和安全防护:`from_space_map_.MadviseDontNeedAndZero()`: 释放“from-space”（对象移动前的旧空间）的物理内存。通过 madvise 告诉操作系统这些内存可以回收了。`mprotect(..., PROT_NONE)`: 这是一个非常重要的安全措施。它将对象移动前的旧内存区域设置为完全不可读写。如果在 GC 之后，程序中还有任何代码（因为 bug）试图访问对象的旧地址，程序会立刻崩溃。这能帮助开发者迅速定位到“悬挂指针”或“引用未更新”的严重错误。

```cpp
void MarkCompact::CompactionPhase() {
  // 开始计时，用于性能分析。
  TimingLogger::ScopedTiming t(__FUNCTION__, GetTimings());
  {
    // 记录本次GC整理后释放的对象数量和字节数。
    int32_t freed_bytes = black_objs_slide_diff_;
    bump_pointer_space_->RecordFree(freed_objects_, freed_bytes);
    RecordFree(ObjectBytePair(freed_objects_, freed_bytes));
  }

  // 【核心步骤1】调用核心函数，整理和移动“可移动空间”（Moving Space）中的对象。
  // 这是对象搬运的主要工作。
  CompactMovingSpace<kCopyMode>(compaction_buffers_map_.Begin());

  // 【核心步骤2】处理“线性分配空间”（Linear Alloc Space）。
  // 这个空间的对象本身不移动，但需要更新它们内部指向“可移动空间”的指针。
  ProcessLinearAlloc();

  // 定义一个等待函数，用于和应用线程（mutator）进行同步。
  // 它通过一个原子计数器来工作，等待所有正在处理内存访问冲突（SIGBUS信号）的应用线程完成。
  auto wait_for_compaction_counter = [this](size_t idx) {
    // 设置“整理完成”标志位，并获取当前计数。
    SigbusCounterType count = sigbus_in_progress_count_[idx].fetch_or(
        kSigbusCounterCompactionDoneMask, std::memory_order_acq_rel);
    // 等待所有已经在处理SIGBUS信号的应用线程完成它们的工作。
    for (uint32_t i = 0; count > 0; i++) {
      BackOff(i);
      count = sigbus_in_progress_count_[idx].load(std::memory_order_acquire);
      // 清除标志位，只检查真实的计数器值。
      count &= ~kSigbusCounterCompactionDoneMask;
    }
  };
  
  // 【第一阶段同步点】
  // GC线程在此设置第一个标志位，通知应用线程：“对象整理已经完成”。
  // 然后等待，确保没有应用线程卡在旧的内存访问处理流程中。
  wait_for_compaction_counter(0);

  // 注销 userfaultfd 对内存区域的监控。
  // GC线程告诉内核：“我不再负责处理这些内存区域的页错误了”。
  // 首先是“可移动空间”。
  size_t moving_space_size = bump_pointer_space_->Capacity();
  size_t used_size = (moving_first_objs_count_ + black_page_count_) * gPageSize;
  if (used_size > 0) {
    UnregisterUffd(bump_pointer_space_->Begin(), used_size);
  }
  // 然后是“线性分配空间”。
  for (auto& data : linear_alloc_spaces_data_) {
    DCHECK_EQ(data.end_ - data.begin_, static_cast<ssize_t>(data.shadow_.Size()));
    UnregisterUffd(data.begin_, data.shadow_.Size());
  }

  // 【第二阶段同步点】
  // GC线程在此设置第二个标志位，通知应用线程：“userfaultfd注销已经完成”。
  // 再次等待，确保所有应用线程都感知到这个变化。之后，应用线程如果再次访问失败，就知道这是真正的错误，而不是GC过程中的临时状态。
  wait_for_compaction_counter(1);

  // 释放“from-space”（对象移动前的旧空间）所占用的物理内存。
  // MadviseDontNeed是一个给操作系统的提示，表明这些内存页可以被回收了。
  from_space_map_.MadviseDontNeedAndZero();
  
  // 【重要安全措施】
  // 将“from-space”的内存保护设置为PROT_NONE（不可读、不可写、不可执行）。
  // 如果在GC之后，程序中还有任何代码（通常是bug）试图访问对象的旧地址，
  // 就会立刻触发段错误（Segmentation Fault），从而帮助开发者快速定位问题。
  DCHECK_EQ(mprotect(from_space_begin_, moving_space_size, PROT_NONE), 0)
      << "mprotect(PROT_NONE) for from-space failed: " << strerror(errno);

  // 释放为“线性分配空间”服务的页面状态数组所占用的内存。
  for (auto& data : linear_alloc_spaces_data_) {
    data.page_status_map_.MadviseDontNeedAndZero();
  }
}
```

#### CompactMovingSpace

`CompactMovingSpace`函数负责把“可移动空间”中的存活对象从零散的旧位置（from-space）拷贝到连续的新位置（to-space）。

##### 反向遍历

函数从内存空间的末尾向开头处理。注释中解释了原因：这样做可以优先处理 TLAB 和最近分配的对象。根据“代际假说”，新创建的对象更有可能存活，优先处理它们可以提高效率。

##### 分段处理

`CompactMovingSpace` 将整个“可移动空间”逻辑上分为三段，并用三个 while 循环来处理：

1. 第一段：处理“纯黑页” (Allocated-black pages): 这些页面是在 GC 标记阶段开始后分配的，ART 知道这些页面上的所有对象都是存活的。处理方式：整体滑动 (Slide)。因为整个页面都是存活对象，所以不需要逐个对象判断，可以直接把整个页面的内容移动到新的位置，效率极高。

2. 第二段：处理“混合页”: 这些页面上既有存活对象，也有死亡对象。处理方式：选择性拷贝 (CompactPage)。GC 线程需要借助位图（live-words bitmap）来识别出哪些是存活对象，然后只将这些存活的对象紧凑地拷贝到目标空间。

3. 第三段：处理“非移动页” (Non-moving pages): 这些页面位于堆的起始位置，经过整理后，它们的位置可能不需要改变。处理方式：原地更新引用 (UpdateNonMovingPage)。虽然这些页面上的对象本身不动，但它们内部可能引用了前面两段中被移动过的对象。因此，这个循环的主要工作是更新这些对象内部的指针，让它们指向正确的新地址。

```cpp
template <int kMode>
void MarkCompact::CompactMovingSpace(uint8_t* page) {
  TimingLogger::ScopedTiming t(__FUNCTION__, GetTimings());
  // 初始化循环变量和指针。
  size_t page_status_arr_len = moving_first_objs_count_ + black_page_count_;
  size_t idx = page_status_arr_len; // 页索引，从后往前遍历。
  // black_dense_end_idx 是整理后存活对象区域的末尾页索引。
  size_t black_dense_end_idx = (black_dense_end_ - moving_space_begin_) / gPageSize;
  // to_space_end 指向目标空间的当前末尾，对象将被拷贝到这里。
  uint8_t* to_space_end = moving_space_begin_ + page_status_arr_len * gPageSize;
  // pre_compact_page 指向源空间（整理前）的当前页。
  uint8_t* pre_compact_page = black_allocations_begin_ + (black_page_count_ * gPageSize);

  DCHECK(IsAlignedParam(pre_compact_page, gPageSize));

  // 初始化用于回收源空间（from-space）页面的变量。
  last_reclaimed_page_ = pre_compact_page;
  last_reclaimable_page_ = last_reclaimed_page_;
  cur_reclaimable_page_ = last_reclaimed_page_;
  last_checked_reclaim_page_idx_ = idx;
  class_after_obj_iter_ = class_after_obj_map_.rbegin();
  
  mirror::Object* next_page_first_obj = nullptr;

  // --- 阶段1: 处理“纯黑页”(Allocated-black pages) ---
  // 这些页面是在GC标记开始后分配的，因此页面上的所有对象都保证是存活的。
  // 处理方式是最高效的“整体滑动”（Slide）。
  while (idx > moving_first_objs_count_) {
    idx--;
    pre_compact_page -= gPageSize;
    to_space_end -= gPageSize;
    if (kMode == kFallbackMode) {
      page = to_space_end;
    }
    mirror::Object* first_obj = first_objs_moving_space_[idx].AsMirrorPtr();
    uint32_t first_chunk_size = black_alloc_pages_first_chunk_size_[idx];
    if (first_obj != nullptr) {
      // DoPageCompactionWithStateChange 封装了页面状态变更和实际的整理操作。
      DoPageCompactionWithStateChange<kMode>(idx,
                                            to_space_end,
                                            page,
                                            /*map_immediately=*/true,
                                            [&]() REQUIRES_SHARED(Locks::mutator_lock_) {
                                              // 直接将整个页面的存活内容“滑动”到新位置。
                                              SlideBlackPage(first_obj,
                                                             next_page_first_obj,
                                                             first_chunk_size,
                                                             pre_compact_page,
                                                             page,
                                                             kMode == kCopyMode);
                                            });
      // 为了效率，不是每处理一页就回收一页，而是累积一定数量后再批量回收。
      if (idx % DivideByPageSize(kMinFromSpaceMadviseSize) == 0) {
        FreeFromSpacePages(idx, kMode, /*end_idx_for_mapping=*/0);
      }
    }
    next_page_first_obj = first_obj;
  }
  DCHECK_EQ(pre_compact_page, black_allocations_begin_);
  
  // 预留一个页面，用于在找不到可回收页面作为临时缓冲区时的备用方案。
  uint8_t* reserve_page = page;
  size_t end_idx_for_mapping = idx;

  // --- 阶段2: 处理“混合页” ---
  // 这些页面上既有存活对象，也有死亡对象（内存碎片）。
  // 这是最典型的整理场景，需要“选择性拷贝”（Compact）。
  while (idx > black_dense_end_idx) {
    idx--;
    to_space_end -= gPageSize;
    if (kMode == kFallbackMode) {
      page = to_space_end;
    } else {
      DCHECK_EQ(kMode, kCopyMode);
      // 尝试从源空间（from-space）的已处理部分找到一个空闲页面作为临时拷贝缓冲区。
      if (cur_reclaimable_page_ > last_reclaimable_page_) {
        cur_reclaimable_page_ -= gPageSize;
        page = cur_reclaimable_page_ + from_space_slide_diff_;
      } else {
        // 如果找不到，则使用预留的页面。
        page = reserve_page;
      }
    }
    mirror::Object* first_obj = first_objs_moving_space_[idx].AsMirrorPtr();
    bool success = DoPageCompactionWithStateChange<kMode>(
        idx,
        to_space_end,
        page,
        /*map_immediately=*/page == reserve_page,
        [&]() REQUIRES_SHARED(Locks::mutator_lock_) {
          // 扫描当前页，只把存活的对象拷贝到新位置。
          CompactPage(first_obj, pre_compact_offset_moving_space_[idx], page, kMode == kCopyMode);
        });
    // 在某些情况下，需要提前将已整理好的页面映射回内存，使其对应用线程可见。
    if (kMode == kCopyMode && (!success || page == reserve_page) && end_idx_for_mapping - idx > 1) {
      MapMovingSpacePages(idx + 1,
                          end_idx_for_mapping,
                          /*from_fault=*/false,
                          /*return_on_contention=*/true,
                          /*tolerate_enoent=*/false);
    }
    // 尝试回收源空间的已处理页面。
    if (FreeFromSpacePages(idx, kMode, end_idx_for_mapping)) {
      end_idx_for_mapping = idx;
    }
  }

  // --- 阶段3: 处理“非移动页” ---
  // 这些页面位于整理后存活区域的内部，因此页面上的对象不需要移动。
  // 但是，这些对象内部可能引用了被移动过的对象，需要“原地更新引用”。
  while (idx > 0) {
    idx--;
    to_space_end -= gPageSize;
    mirror::Object* first_obj = first_objs_moving_space_[idx].AsMirrorPtr();
    if (first_obj != nullptr) {
      DoPageCompactionWithStateChange<kMode>(
          idx,
          to_space_end,
          to_space_end + from_space_slide_diff_,
          /*map_immediately=*/false,
          [&]() REQUIRES_SHARED(Locks::mutator_lock_) {
            // 遍历页面上的所有对象，更新它们内部的指针，使其指向移动后的新地址。
            UpdateNonMovingPage(
                first_obj, to_space_end, from_space_slide_diff_, moving_space_bitmap_);
            if (kMode == kFallbackMode) {
              memcpy(to_space_end, to_space_end + from_space_slide_diff_, gPageSize);
            }
          });
    } else {
      // 这个页面上没有任何可达对象，直接标记为已处理和已映射。
      DCHECK_EQ(moving_pages_status_[idx].load(std::memory_order_relaxed),
                static_cast<uint8_t>(PageState::kUnprocessed));
      moving_pages_status_[idx].store(static_cast<uint8_t>(PageState::kProcessedAndMapped),
                                      std::memory_order_release);
    }
    if (FreeFromSpacePages(idx, kMode, end_idx_for_mapping)) {
      end_idx_for_mapping = idx;
    }
  }
  
  // 最后一次映射，确保所有剩余的页面都对应用线程可见。
  if (kMode == kCopyMode && end_idx_for_mapping > 0) {
    MapMovingSpacePages(idx,
                        end_idx_for_mapping,
                        /*from_fault=*/false,
                        /*return_on_contention=*/false,
                        /*tolerate_enoent=*/false);
  }
  
  // 检查点：确保目标空间的指针最终回到了空间的起始位置，表示整个空间都已处理完毕。
  DCHECK_EQ(to_space_end, bump_pointer_space_->Begin());
}
```

##### 单页的整理操作

DoPageCompactionWithStateChange函数为单页的整理操作提供了一个线程安全的“事务”封装。它通过原子状态机来确保一个页面只被处理一次。

实际执行Compact操作的函数为参数`CompactionFn`。
包括了`UpdateNonMovingPage`, `SlideBlackPage`,`CompactPage`。

```cpp
template <int kMode, typename CompactionFn>
bool MarkCompact::DoPageCompactionWithStateChange(size_t page_idx,
                                                  uint8_t* to_space_page,
                                                  uint8_t* page,
                                                  bool map_immediately,
                                                  CompactionFn func) {
  
  uint32_t expected_state = static_cast<uint8_t>(PageState::kUnprocessed); // 期望的初始状态
  uint32_t desired_state = static_cast<uint8_t>(map_immediately ? PageState::kProcessingAndMapping
                                                                  : PageState::kProcessing); // 想要设置的目标状态
  
  // 【核心并发控制】
  // 使用原子CAS（比较并交换）操作尝试“声明”对该页面的处理权。
  // 只有当页面的当前状态是 kUnprocessed 时，CAS才会成功。
  // 使用 acquire 内存顺序，确保在CAS失败时，能看到其他线程 release 的所有内存写入。
  if (kMode == kFallbackMode || moving_pages_status_[page_idx].compare_exchange_strong(
                                  expected_state, desired_state, std::memory_order_acquire)) {
    // CAS 成功，本线程获得处理权。
    
    // 1. 执行由调用者传入的、实际的整理工作（如 SlideBlackPage 或 CompactPage）。
    func();
    
    // 2. 根据模式和参数，完成“事务提交”。
    if (kMode == kCopyMode) { // 只在并发模式下需要复杂的提交逻辑。
      if (map_immediately) {
        // --- 路径A：立即映射 ---
        // 立即调用 ioctl，使更新对所有线程可见。
        CopyIoctl(to_space_page,
                  page,
                  gPageSize,
                  /*return_on_contention=*/false,
                  /*tolerate_enoent=*/false);
        // 更新状态为最终的“已处理并映射”。
        // relaxed即可，因为ioctl本身起到了内存屏障的作用。
        moving_pages_status_[page_idx].store(static_cast<uint8_t>(PageState::kProcessedAndMapped),
                                              std::memory_order_relaxed);
      } else {
        // --- 路径B：延迟映射（用于批处理）---
        // 不立即映射，而是将源页面的信息和新状态打包存储，待后续统一处理。
        DCHECK(from_space_map_.HasAddress(page));
        // 计算源页面的地址偏移。
        uint32_t store_val = page - from_space_begin_;
        DCHECK_EQ(store_val & kPageStateMask, 0u);
        // 将状态 kProcessed 打包到低位。
        store_val |= static_cast<uint8_t>(PageState::kProcessed);
        
        // 将打包后的值存回状态数组。
        // 【关键】必须使用 release 内存顺序！
        // 这确保 func() 中的所有内存写入，对于任何之后读到这个新状态的线程都是可见的。
        moving_pages_status_[page_idx].store(store_val, std::memory_order_release);
      }
    }
    return true; // 表示本线程成功处理了此页面。
  } else {
    // CAS失败，意味着其他线程已经处理或正在处理此页面，本线程放弃操作。
    DCHECK_NE(expected_state, static_cast<uint8_t>(PageState::kProcessed));
    return false; // 表示本线程未处理此页面。
  }
}
```

###### UpdateNonMovingPage函数

`UpdateNonMovingPage`函数负责更新一个“非移动页面”上所有存活对象的内部引用。对象本身不移动，但它们指向其他已移动对象的指针需要被修正。主要挑战在于正确处理跨越页面边界的对象。

`RefsUpdateVisitor`这是一个“访问者”对象。它的内部逻辑封装了**旧地址 -> 新地址**的转换规则。它会查询一个映射表（`info_map_`），找到被移动对象的转发地址。

```cpp
void MarkCompact::UpdateNonMovingPage(mirror::Object* first,
                                      uint8_t* page,
                                      ptrdiff_t from_space_diff,
                                      accounting::ContinuousSpaceBitmap* bitmap) {

  DCHECK_LT(reinterpret_cast<uint8_t*>(first), page + gPageSize);

  mirror::Object* curr_obj = first; // 当前正在处理的对象（位于to-space）。
  uint8_t* from_page = page + from_space_diff; // 当前页面在from-space的对应地址。
  uint8_t* from_page_end = from_page + gPageSize;

  // 使用标记位图来遍历当前页面上所有的“存活”对象。
  // VisitMarkedRange 会为每个存活对象（的起始点）调用一次lambda。
  bitmap->VisitMarkedRange(
      // 从页面上第一个对象的头部之后开始扫描。
      reinterpret_cast<uintptr_t>(first) + mirror::kObjectHeaderSize,
      reinterpret_cast<uintptr_t>(page + gPageSize), // 扫描直到页面末尾。
      [&](mirror::Object* next_obj) {
        // from_obj 指向对象在整理前的原始位置，我们需要从这里读取旧的引用信息。
        mirror::Object* from_obj = reinterpret_cast<mirror::Object*>(
            reinterpret_cast<uint8_t*>(curr_obj) + from_space_diff);

        // --- 处理跨页对象的核心逻辑 ---
        if (reinterpret_cast<uint8_t*>(curr_obj) < page) {
          // 情况1：当前对象是从前一个页面延伸过来的。
          // 创建一个引用更新访问器，它负责具体的“旧地址 -> 新地址”转换和写入操作。
          RefsUpdateVisitor</*kCheckBegin*/ true, /*kCheckEnd*/ false> visitor(
              this, from_obj, from_page, from_page_end);
          // 计算在当前页面内的起始扫描偏移量。
          MemberOffset begin_offset(page - reinterpret_cast<uint8_t*>(curr_obj));
          // 调用对象的访问方法，遍历其在当前页面内的所有引用字段。
          // 因为对象头部在前一页，其NativeRoots已处理过，此处不再访问。
          from_obj->VisitRefsForCompaction</*kFetchObjSize*/ false, /*kVisitNativeRoots*/ false>(
              visitor, begin_offset, MemberOffset(-1));
        } else {
          // 情况2：当前对象完整地起始于本页面内。
          RefsUpdateVisitor</*kCheckBegin*/ false, /*kCheckEnd*/ false> visitor(
              this, from_obj, from_page, from_page_end);
          // 从对象的0偏移处开始扫描所有引用。
          from_obj->VisitRefsForCompaction</*kFetchObjSize*/ false>(
              visitor, MemberOffset(0), MemberOffset(-1));
        }
        // 移动到下一个由VisitMarkedRange找到的存活对象。
        curr_obj = next_obj;
      });

  // VisitMarkedRange的lambda处理了除最后一个对象外的所有对象。
  // 这里需要单独处理页面上的最后一个存活对象，因为它可能延伸到下一页。
  mirror::Object* from_obj =
      reinterpret_cast<mirror::Object*>(reinterpret_cast<uint8_t*>(curr_obj) + from_space_diff);
  // 计算结束扫描的偏移量，确保不超过页面边界。
  MemberOffset end_offset(page + gPageSize - reinterpret_cast<uint8_t*>(curr_obj));
  
  if (reinterpret_cast<uint8_t*>(curr_obj) < page) {
    // 情况A：最后一个对象也是从前一页延伸过来的。
    RefsUpdateVisitor</*kCheckBegin*/ true, /*kCheckEnd*/ true> visitor(
        this, from_obj, from_page, from_page_end);
    from_obj->VisitRefsForCompaction</*kFetchObjSize*/ false, /*kVisitNativeRoots*/ false>(
        visitor, MemberOffset(page - reinterpret_cast<uint8_t*>(curr_obj)), end_offset);
  } else {
    // 情况B：最后一个对象起始于本页面内。
    RefsUpdateVisitor</*kCheckBegin*/ false, /*kCheckEnd*/ true> visitor(
        this, from_obj, from_page, from_page_end);
    from_obj->VisitRefsForCompaction</*kFetchObjSize*/ false>(visitor, MemberOffset(0), end_offset);
  }
}
```

###### CompactPage函数

* 阶段一：拷贝所有存活的数据条带`VisitLiveStrides`。此函数不关心对象的边界，而是使用 `live_words_bitmap_`。`live_words_bitmap_->VisitLiveStrides(...)`会扫描位图，找出所有连续的、被标记为存活的内存区间，称之为“条带”（Stride）。一个条带可能包含多个小对象，也可能只是一个大对象的一部分。在 `VisitLiveStrides` 的回调中，函数对找到的每一个条带执行一次 `memcpy`，将这块连续的存活数据从 `from_space_` 拷贝到目标地址 `addr`。这种方式比逐个对象拷贝要高效得多。

* 阶段二：修复拷贝后数据中的指针: 此时，所有存活数据都已紧凑地排列在 `start_addr` 开始的内存区域里，但内部指针全是错的。接下来的代码就是遍历这块新内存，逐个对象进行修复。

1. Part A: 处理第一个对象（可能跨页而来）: 页面上的第一个存活数据块，可能只是一个从前一页延伸过来的大对象的“尾巴”。代码首先处理这个特殊情况。它通过 `offset` 参数计算出这个对象在当前页内的起始偏移 `offset_within_obj`，然后调用 `VisitRefsForCompaction` 只修复这个对象落在当前页内的那部分引用。
2. Part B: 处理中间的完整对象: 第一个 while 循环负责处理后续的、可以确定是完整位于新拷贝数据区域内部的对象。它通过 `ref->VisitRefsForCompaction` 修复对象的引用，获取该对象的实际大小 `obj_size`,循环可以准确地“跳”到下一个对象的起始位置 (`bytes_done += obj_size`)，从而完成遍历。这个阶段的边界检查较少，效率更高。
3. Part C: 处理最后一个条带中的对象: 第二个 while 循环专门处理最后一个数据条带中的对象。情况更复杂，因为最后一个对象可能会延伸到当前页的范围之外。因此，在调用 `VisitRefsForCompaction` 时，需要传入页面末尾的边界，并启用 `kCheckEnd` 模板参数，进行更严格的边界检查。

```cpp
void MarkCompact::CompactPage(mirror::Object* obj,
                              uint32_t offset,
                              uint8_t* addr,
                              bool needs_memset_zero) {
  // 本函数是“混合页面”的整理核心。它分两步：
  // 1. 使用 live_words_bitmap_ 将所有存活的内存“条带”(strides) 快速 memcpy 到目标地址。
  // 2. 遍历拷贝后的紧凑数据，识别出对象，并逐个修复其内部的引用指针。
  
  // ... 一系列调试断言 ...
  
  uint8_t* const start_addr = addr; // 记录目标区域的起始地址。
  size_t stride_count = 0;
  // ...

  obj = GetFromSpaceAddr(obj);

  // --- 阶段一：拷贝所有存活的数据“条带”(Live Strides) ---
  // 这是“粗活”，只管搬数据，不管内容是什么。
  live_words_bitmap_->VisitLiveStrides(
      offset,
      black_allocations_begin_,
      gPageSize,
      [&](uint32_t stride_begin, size_t stride_size, [[maybe_unused]] bool is_last)
          REQUIRES_SHARED(Locks::mutator_lock_) {
        const size_t stride_in_bytes = stride_size * kAlignment;
        // ...
        // 从 from_space_ 拷贝连续的存活数据到目标地址 addr。
        memcpy(addr, from_space_begin_ + stride_begin * kAlignment, stride_in_bytes);
        // ... (Debug build 的校验代码) ...
        last_stride = addr;
        addr += stride_in_bytes; // 移动目标地址指针。
        stride_count++;
      });
  
  // ... 调试断言 ...

  // --- 阶段二：修复指针 ---
  // 这是“细活”，遍历刚拷贝过来的数据，更新里面的指针。
  size_t obj_size = 0;
  // 计算页面上第一个对象在页面内的偏移量。
  uint32_t offset_within_obj = offset * kAlignment
                               - (reinterpret_cast<uint8_t*>(obj) - from_space_begin_);
  
  // Part 2a: 处理第一个对象（可能从前一页延伸而来）。
  if (offset_within_obj > 0) {
    mirror::Object* to_ref = reinterpret_cast<mirror::Object*>(start_addr - offset_within_obj);
    if (stride_count > 1) {
      // 如果有多个条带，说明对象不会延伸出页面。
      RefsUpdateVisitor</*kCheckBegin*/true, /*kCheckEnd*/false> visitor(this, to_ref, start_addr, nullptr);
      obj_size = obj->VisitRefsForCompaction</*kFetchObjSize*/true, /*kVisitNativeRoots*/false>(
          visitor, MemberOffset(offset_within_obj), MemberOffset(-1));
    } else {
      // 只有一个条带，对象可能延伸出页面，需要进行边界检查。
      RefsUpdateVisitor</*kCheckBegin*/true, /*kCheckEnd*/true> visitor(this, to_ref, start_addr, start_addr + gPageSize);
      obj_size = obj->VisitRefsForCompaction</*kFetchObjSize*/true, /*kVisitNativeRoots*/false>(
          visitor, MemberOffset(offset_within_obj), MemberOffset(offset_within_obj + gPageSize));
    }
    obj_size = RoundUp(obj_size, kAlignment);
    // ... 调试断言 ...
    obj_size -= offset_within_obj; // 计算对象在本页内的剩余大小。
    // ...
  }

  uint8_t* const end_addr = addr; // 记录拷贝数据的末尾地址。
  addr = start_addr;
  size_t bytes_done = obj_size; // bytes_done 记录已处理（修复完指针）的字节数。

  // Part 2b: 处理中间的、完整的对象（位于除最后一个条带外的区域）。
  // 这个循环的边界检查较少，效率较高。
  DCHECK_LE(addr, last_stride);
  size_t bytes_to_visit = last_stride - addr;
  while (bytes_to_visit > bytes_done) {
    mirror::Object* ref = reinterpret_cast<mirror::Object*>(addr + bytes_done);
    // ...
    RefsUpdateVisitor</*kCheckBegin*/false, /*kCheckEnd*/false>
        visitor(this, ref, nullptr, nullptr);
    // 修复引用，并获取对象大小，以便跳到下一个对象。
    obj_size = ref->VisitRefsForCompaction(visitor, MemberOffset(0), MemberOffset(-1));
    obj_size = RoundUp(obj_size, kAlignment);
    bytes_done += obj_size;
  }

  // Part 2c: 处理最后一个条带中的对象（需要进行严格的边界检查）。
  uint8_t* from_addr = from_space_begin_ + last_stride_begin * kAlignment;
  bytes_to_visit = end_addr - addr;
  while (bytes_to_visit > bytes_done) {
    mirror::Object* ref = reinterpret_cast<mirror::Object*>(addr + bytes_done);
    obj = reinterpret_cast<mirror::Object*>(from_addr);
    // ...
    // 此处的 Visitor 启用了末尾边界检查（kCheckEnd=true）。
    RefsUpdateVisitor</*kCheckBegin*/false, /*kCheckEnd*/true>
        visitor(this, ref, nullptr, start_addr + gPageSize);
    obj_size = obj->VisitRefsForCompaction(visitor,
                                          MemberOffset(0),
                                          MemberOffset(end_addr - (addr + bytes_done)));
    obj_size = RoundUp(obj_size, kAlignment);
    // ... 调试断言 ...
    from_addr += obj_size;
    bytes_done += obj_size;
  }
  
  // 如果需要，将页面末尾的松弛空间清零。
  if (needs_memset_zero && UNLIKELY(bytes_done < gPageSize)) {
    std::memset(addr + bytes_done, 0x0, gPageSize - bytes_done);
  }
}

```

###### SlideBlackPage函数

SlideBlackPage 是一个高度优化的页面整理函数，专门用于处理**纯黑页(Pure-Black Pages)**。

“纯黑页”是指在 GC 标记阶段开始后新分配的内存页，因此 ART 虚拟机可以保证该页面上的所有对象都是存活的（即“黑色的”）。由于没有死亡对象，页面上不存在需要消除的内存碎片。

因此，这个函数的目标不是像 CompactPage 那样去“压缩”页面，而是更简单、更高效地**滑动(Slide)** 整个页面的有效内容：

* 整体拷贝：将页面上连续的、存活的对象块作为一个整体，通过 memcpy 快速拷贝到新的目标位置。

* 指针修复：在整体拷贝完成后，遍历新位置上的这些对象，并更新它们内部的引用指针。

```cpp
void MarkCompact::SlideBlackPage(mirror::Object* first_obj,
                                 mirror::Object* next_page_first_obj,
                                 uint32_t first_chunk_size,
                                 uint8_t* const pre_compact_page,
                                 uint8_t* dest,
                                 bool needs_memset_zero) {
  // 本函数是一个优化，用于高效处理“纯黑页”（页面上所有对象都存活）。
  // 它将页面上连续的活对象块作为一个整体“滑动”（memcpy）到新位置，然后修复内部指针。

  // ... 调试断言和变量初始化 ...
  uint8_t* src_addr = reinterpret_cast<uint8_t*>(GetFromSpaceAddr(first_obj));
  uint8_t* pre_compact_addr = reinterpret_cast<uint8_t*>(first_obj);
  // ...

  // 如果页面起始处有空白（因为第一个对象是从前一页延伸过来的），则先处理这部分空白。
  if (pre_compact_addr > pre_compact_page) {
    bytes_copied = pre_compact_addr - pre_compact_page;
    if (needs_memset_zero) {
      std::memset(dest, 0x0, bytes_copied);
    }
    dest += bytes_copied;
  } else {
    // ... 处理对象起始地址在页面之前的情况 ...
  }
  
  // --- 步骤1：拷贝第一个已知的连续存活对象块 ---
  std::memcpy(dest, src_addr, first_chunk_size);
  
  // --- 步骤2：修复刚拷贝的这个块中的指针 ---
  {
    size_t bytes_to_visit = first_chunk_size;
    size_t obj_size;
    
    // Part 2a: 特殊处理第一个对象，因为它可能从前一页延伸而来，需要计算偏移。
    size_t offset = pre_compact_addr - reinterpret_cast<uint8_t*>(first_obj);
    if (bytes_copied == 0 && offset > 0) {
      mirror::Object* to_obj = reinterpret_cast<mirror::Object*>(dest - offset);
      mirror::Object* from_obj = reinterpret_cast<mirror::Object*>(src_addr - offset);
      
      // 根据下一个页面的起始对象位置，决定是否需要进行末尾边界检查。
      if (next_page_first_obj == nullptr || ...) {
        RefsUpdateVisitor</*kCheckBegin*/true, /*kCheckEnd*/false> visitor(...);
        obj_size = from_obj->VisitRefsForCompaction</*kFetchObjSize*/true, ...>(visitor, ...);
      } else {
        RefsUpdateVisitor</*kCheckBegin*/true, /*kCheckEnd*/true> visitor(...);
        obj_size = from_obj->VisitRefsForCompaction</*kFetchObjSize*/true, ...>(visitor, ...);
        if (first_obj == next_page_first_obj) {
          // 如果此对象是本页唯一的对象，修复完直接返回。
          return;
        }
      }
      obj_size = RoundUp(obj_size, kAlignment);
      obj_size -= offset; // 计算对象在本页内的剩余大小。
      dest += obj_size;
      bytes_to_visit -= obj_size;
    }
    bytes_copied += first_chunk_size;
    
    // ... 预处理最后一个对象，判断是否需要边界检查 ...
    bool check_last_obj = false;
    if (next_page_first_obj != nullptr && ...) {
        // ...
        check_last_obj = true;
    }
    
    // Part 2b: 遍历并修复第一个块中的其余对象。
    while (bytes_to_visit > 0) {
      mirror::Object* dest_obj = reinterpret_cast<mirror::Object*>(dest);
      RefsUpdateVisitor</*kCheckBegin*/false, /*kCheckEnd*/false> visitor(...);
      // 修复引用，并获取对象大小，以便跳到下一个对象。
      obj_size = dest_obj->VisitRefsForCompaction(visitor, ...);
      obj_size = RoundUp(obj_size, kAlignment);
      bytes_to_visit -= obj_size;
      dest += obj_size;
    }
    DCHECK_EQ(bytes_to_visit, 0u);
    
    // Part 2c: 如果需要，对最后一个对象进行带边界检查的修复。
    if (check_last_obj) {
      // ... (对最后一个跨页对象的特殊处理) ...
      return;
    }
  }

  // --- 步骤3：处理页面上可能存在的、后续的存活对象块（例如，新的TLAB）---
  if (bytes_copied < gPageSize) {
    src_addr += first_chunk_size;
    pre_compact_addr += first_chunk_size;
    
    // 使用标记位图在页面剩余部分寻找下一个活对象的起始点。
    uintptr_t start_visit = reinterpret_cast<uintptr_t>(pre_compact_addr);
    uintptr_t page_end = reinterpret_cast<uintptr_t>(pre_compact_page_end);
    mirror::Object* found_obj = nullptr;
    moving_space_bitmap_->VisitMarkedRange</*kVisitOnce*/true>(start_visit, page_end, ...);
    
    size_t remaining_bytes = gPageSize - bytes_copied;
    if (found_obj == nullptr) {
      // 页面剩余部分没有活对象了，清零并返回。
      if (needs_memset_zero) {
        std::memset(dest, 0x0, remaining_bytes);
      }
      return;
    }
    
    // Part 3a: 拷贝页面剩余的所有内容（包括对象和之间可能的小间隙）。
    std::memcpy(dest, src_addr, remaining_bytes);
    
    // Part 3b: 遍历新拷贝的区域，修复所有活对象的指针。
    moving_space_bitmap_->VisitMarkedRange(..., [&](mirror::Object* obj) {
        // ... (找到对象在新位置的地址 ref)
        RefsUpdateVisitor</*kCheckBegin*/false, /*kCheckEnd*/false> visitor(...);
        ref->VisitRefsForCompaction</*kFetchObjSize*/false>(visitor, ...);
        // ...
    });
    // 对最后一个找到的对象进行带边界检查的修复。
    // ...
  }
}
```

#### ProcessLinearAlloc

`ProcessLinearAlloc`函数安全地更新非移动对象内部的指针，确保它们指向移动后的新地址。

`LinearAlloc` 里的对象自身不动，但它们可能包含指向普通 Java 对象（在 `MovingSpace` 中）的字段。当 `MovingSpace` 中的对象被 `CompactMovingSpace` 移动后，这些字段里的指针就变成了无效的旧地址。

修复机制：**影子空间 (Shadow Space) + 原子替换**

遍历与更新: 函数遍历所有 LinearAlloc 的内存区域 (arena)。对于每个区域，它会逐个检查里面的对象，并更新所有指向被移动对象的指针。

写入影子空间: 更新操作不是在原始内存上直接进行的。而是将修改后的对象写入一个临时的、地址平行的**影子空间 (shadow_)**。这样做是为了避免在更新过程中，应用线程读到不一致的数据（一个对象里有些指针更新了，有些还没更新）。

原子替换: 当一个或多个页面的影子空间都准备好后，函数会调用 `MapUpdatedLinearAllocPages`。这个函数利用 `userfaultfd` 机制，原子性地将这些更新过的影子页面映射到原始地址上。对于应用线程来说，这个地址上的内容仿佛在“一瞬间”就从旧版本变成了新版本，从而保证了数据的一致性和线程安全。

```cpp
void MarkCompact::ProcessLinearAlloc() {
  GcVisitedArenaPool* arena_pool =
      static_cast<GcVisitedArenaPool*>(Runtime::Current()->GetLinearAllocArenaPool());
  // 调试检查：确保当前线程是执行GC的线程。
  DCHECK_EQ(thread_running_gc_, Thread::Current());
  
  // 用于追踪连续的、待映射的内存范围。
  uint8_t* unmapped_range_start = nullptr;
  uint8_t* unmapped_range_end = nullptr;
  
  // 指向当前正在处理的 LinearAlloc 空间元数据。
  LinearAllocSpaceData* space_data = nullptr;
  // 指向当前空间的状态数组，用于线程同步。
  Atomic<PageState>* state_arr = nullptr;
  // 原始空间与其影子空间之间的地址偏移量。
  ptrdiff_t diff = 0;

  // 定义一个lambda函数，负责将更新过的影子页面映射回原始地址。
  auto map_pages = [&]() {
    // 一系列健全性检查。
    DCHECK_NE(diff, 0);
    DCHECK_NE(space_data, nullptr);
    DCHECK_GE(unmapped_range_start, space_data->begin_);
    // ...
    DCHECK_ALIGNED_PARAM(unmapped_range_end - unmapped_range_start, gPageSize);
    
    size_t page_idx = DivideByPageSize(unmapped_range_start - space_data->begin_);
    // 调用核心函数，执行内存映射，将 unmapped_range_start + diff (影子空间) 的内容
    // 映射到 unmapped_range_start (原始空间)。
    // MapUpdatedLinearAllocPages函数它通过 userfaultfd 的 UFFDIO_COPY 命令，请求 Linux 内核执行内存页的替换。
    // 内核会把 start_shadow_page 指向的物理内存，直接映射到 start_page 指向的虚拟地址上。
    MapUpdatedLinearAllocPages(unmapped_range_start,
                               unmapped_range_start + diff,
                               state_arr + page_idx,
                               unmapped_range_end - unmapped_range_start,
                               /*free_pages=*/true,
                               /*single_ioctl=*/false,
                               /*tolerate_enoent=*/false);
  };
  
  // 遍历所有需要处理的内存块（Arenas）。
  for (auto& pair : linear_alloc_arenas_) {
    const TrackedArena* arena = pair.first;
    size_t arena_size = arena->Size();
    uint8_t* arena_begin = arena->Begin();
    
    // arenas是按地址排序的，这里确保我们是顺序处理。
    DCHECK_LE(unmapped_range_end, arena_begin);
    DCHECK(space_data == nullptr || arena_begin > space_data->begin_)
        << "space-begin:" << static_cast<void*>(space_data->begin_)
        << " arena-begin:" << static_cast<void*>(arena_begin);

    // 如果当前arena属于一个新的LinearAlloc空间（或者这是第一个arena）
    if (space_data == nullptr || space_data->end_ <= arena_begin) {
      // 切换空间之前，先把上一个空间里已处理的页面进行映射。
      if (space_data != nullptr && unmapped_range_end != nullptr) {
        map_pages();
        unmapped_range_end = nullptr;
      }
      // 找到当前arena所属的LinearAlloc空间，并设置好space_data, diff等变量。
      LinearAllocSpaceData* curr_space_data = space_data;
      for (auto& data : linear_alloc_spaces_data_) {
        if (data.begin_ <= arena_begin && arena_begin < data.end_) {
          // ...
          diff = data.shadow_.Begin() - data.begin_; // 计算与影子空间的偏移
          state_arr = reinterpret_cast<Atomic<PageState>*>(data.page_status_map_.Begin());
          space_data = &data;
          break;
        }
      }
      CHECK_NE(space_data, curr_space_data)
          << "Couldn't find space for arena-begin:" << static_cast<void*>(arena_begin);
    }
    
    // 如果在同一个空间内发现了不连续的arena（有空洞），则先把前面的部分映射掉。
    if (unmapped_range_end != nullptr && unmapped_range_end < arena_begin) {
      map_pages();
      unmapped_range_end = nullptr;
    }
    
    // 开始一个新的连续处理范围。
    if (unmapped_range_end == nullptr) {
      unmapped_range_start = unmapped_range_end = arena_begin;
    }
    
    // 扩大当前待处理的范围，将当前arena包含进来。
    unmapped_range_end += arena_size;
    {
      // 加读锁，防止在更新arena时，它被其他线程释放。
      ReaderMutexLock rmu(thread_running_gc_, arena_pool->GetLock());
      // 如果arena在GC暂停后已被标记为待删除，则跳过。
      if (arena->IsWaitingForDeletion()) {
        continue;
      }
      
      uint8_t* last_byte = pair.second;
      DCHECK_ALIGNED_PARAM(last_byte, gPageSize);

      // 定义一个“访问者”lambda，用于处理arena中的每一页。
      auto visitor = [space_data, last_byte, diff, this, state_arr](
          uint8_t* page_begin,        // 当前页的起始地址（在原始空间）
          uint8_t* first_obj,         // 页面上第一个对象的地址
          size_t page_size) REQUIRES_SHARED(Locks::mutator_lock_) {
        
        // 如果页面已经超出了需要更新的范围，则直接返回。
        if (page_begin >= last_byte) {
          return;
        }

        LinearAllocPageUpdater updater(this);
        size_t page_idx = DivideByPageSize(page_begin - space_data->begin_);
        DCHECK_LT(page_idx, space_data->page_status_map_.Size());
        PageState expected_state = PageState::kUnprocessed;
        
        // 使用原子操作CAS（比较并交换）来确保每个页面只被处理一次。
        // 将页面状态从 kUnprocessed 切换到 kProcessing。
        if (state_arr[page_idx].compare_exchange_strong(
                expected_state, PageState::kProcessing, std::memory_order_acquire)) {
          // 根据页面类型（多对象或单对象），调用updater来更新指针。
          // 更新操作是写入到影子空间中 (page_begin + diff)。
          if (first_obj != nullptr) {
            updater.MultiObjectArena(page_begin + diff, first_obj + diff);
          } else {
            updater.SingleObjectArena(page_begin + diff, page_size);
          }
          
          expected_state = PageState::kProcessing;
          // 使用原子store操作，将页面状态更新为 kProcessed 或 kProcessedAndMapped。
          // release内存顺序确保了对影子页面的所有写入都已完成。
          if (updater.WasLastPageTouched()) {
            state_arr[page_idx].store(PageState::kProcessed, std::memory_order_release);
          } else {
            // 如果页面未被触碰（没有需要更新的指针），则直接用0填充并标记为已映射。
            ZeropageIoctl(
                page_begin, gPageSize, /*tolerate_eexist=*/false, /*tolerate_enoent=*/false);
            state_arr[page_idx].store(PageState::kProcessedAndMapped, std::memory_order_release);
          }
        }
      };

      // 调用arena的VisitRoots方法，它会用上面定义的visitor来遍历arena中的所有对象和页面。
      arena->VisitRoots(visitor);
    }
  }
  
  // 循环结束后，处理最后一批待映射的页面。
  if (unmapped_range_end > unmapped_range_start) {
    map_pages();
  }
}
```

`MapUpdatedLinearAllocPages`通过 `userfaultfd` 的 `ioctl(UFFDIO_COPY)` 命令，将 start_shadow_page (影子空间) 的内容原子性地映射到 start_page (原始空间) 的虚拟地址上。

```cpp
bool MarkCompact::MapUpdatedLinearAllocPages(uint8_t* start_page,
                                             uint8_t* start_shadow_page,
                                             Atomic<PageState>* state,
                                             size_t length,
                                             bool free_pages,
                                             bool single_ioctl,
                                             bool tolerate_enoent) {

  DCHECK_ALIGNED_PARAM(length, gPageSize);
  
  // 用于最后 madvise 释放内存的指针和变量。
  Atomic<PageState>* madv_state = state;
  size_t madv_len = length;
  uint8_t* madv_start = start_shadow_page;
  // 标记是否在处理过程中遇到了并发操作，如果是，则在释放内存前需要额外等待。
  bool check_state_for_madv = false;
  uint8_t* end_page = start_page + length;

  // 主循环，批量处理待映射的页面。
  while (start_page < end_page) {
    size_t map_len = 0;
    // 步骤1：寻找一个连续的、状态为 kProcessed 的页面范围，以便一次性映射。
    // kProcessed 意味着GC线程已将更新内容写入影子页，只待映射。
    for (Atomic<PageState>* cur_state = state;
         map_len < length && cur_state->load(std::memory_order_acquire) == PageState::kProcessed;
         map_len += gPageSize, cur_state++) {
      // 循环体为空，仅用于计算连续的长度 map_len。
    }

    if (map_len == 0) {
      // 如果找不到任何可立即映射的页面。
      if (single_ioctl) {
        // 如果只允许一次ioctl，则检查当前页面是否已映射并返回。
        return state->load(std::memory_order_relaxed) == PageState::kProcessedAndMapped;
      }
      // 跳过所有非 kProcessed 的页面 (如 kUnprocessed, kProcessing, kProcessedAndMapped)。
      while (length > 0) {
        PageState s = state->load(std::memory_order_relaxed);
        if (s == PageState::kProcessed) {
          // 找到了下一个可以开始处理的页面，退出内层循环。
          break;
        }
        // 如果发现页面正被其他线程（mutator）处理，则设置标志位。
        check_state_for_madv |= s > PageState::kUnprocessed && s < PageState::kProcessedAndMapped;
        // 移动指针，继续寻找。
        state++;
        length -= gPageSize;
        start_shadow_page += gPageSize;
        start_page += gPageSize;
      }
    } else {
      // 步骤2：【核心操作】对找到的连续页面范围执行 ioctl 映射。
      map_len = CopyIoctl(start_page,
                          start_shadow_page,
                          map_len,
                          /*return_on_contention=*/false,
                          tolerate_enoent);
      DCHECK_NE(map_len, 0u);
      
      // 步骤3：映射成功后，将这些页面的状态更新为 kProcessedAndMapped，
      // 通知其他线程它们已可用。'release'内存顺序确保状态更新前的所有操作都已完成。
      for (size_t l = 0; l < map_len; l += gPageSize, state++) {
        PageState s = state->load(std::memory_order_relaxed);
        DCHECK(s == PageState::kProcessed || s == PageState::kProcessedAndMapped) << "state:" << s;
        state->store(PageState::kProcessedAndMapped, std::memory_order_release);
      }
      
      if (single_ioctl) {
        break;
      }
      
      // 更新指针，准备处理下一批页面。
      start_page += map_len;
      start_shadow_page += map_len;
      length -= map_len;
    }
  }

  // --- 清理阶段：回收影子页面的物理内存 ---
  if (free_pages) {
    if (check_state_for_madv) {
      // 如果之前检测到有并发操作，必须在此等待，确保所有页面都已映射完毕，才能安全释放。
      // 这是为了防止在mutator还在使用影子页时，GC线程就将其释放（Use-After-Free）。
      for (size_t l = 0; l < madv_len; l += gPageSize, madv_state++) {
        uint32_t backoff_count = 0;
        PageState s = madv_state->load(std::memory_order_relaxed);
        while (s > PageState::kUnprocessed && s < PageState::kProcessedAndMapped) {
          BackOff(backoff_count++);
          s = madv_state->load(std::memory_order_relaxed);
        }
      }
    }
    // 调用 madvise(MADV_DONTNEED) 将影子页面的物理内存归还给操作系统。
    ZeroAndReleaseMemory(madv_start, madv_len);
  }
  return true;
}
```

`CopyIoctl` 是 `userfaultfd UFFDIO_COPY ioctl`系统调用的底层封装。

```cpp
size_t MarkCompact::CopyIoctl(
    void* dst, void* buffer, size_t length, bool return_on_contention, bool tolerate_enoent) {
  
  int32_t backoff_count = -1;
  int32_t max_backoff = 10;
  
  // 准备传递给内核的 ioctl 参数结构体。
  struct uffdio_copy uffd_copy;
  // 【关键】设置模式为 MMAP_TRYLOCK。这是一种非阻塞的乐观尝试。
  // 如果内核需要的 mmap_lock 被其他线程持有，ioctl 会立即返回 EAGAIN 而不是等待。
  uffd_copy.mode = gUffdSupportsMmapTrylock ? UFFDIO_COPY_MODE_MMAP_TRYLOCK : 0;
  uffd_copy.src = reinterpret_cast<uintptr_t>(buffer);
  uffd_copy.dst = reinterpret_cast<uintptr_t>(dst);
  uffd_copy.len = length;
  uffd_copy.copy = 0; // copy 是一个出参，内核会通过它返回成功处理的字节数。

  // 使用循环来处理可能发生的瞬时失败（如竞争）。
  while (true) {
    // 【核心系统调用】请求内核执行页面拷贝/重映射操作。
    int ret = ioctl(uffd_, UFFDIO_COPY, &uffd_copy);

    if (ret == 0) {
      // 成功：所有页面都已成功映射。
      DCHECK_EQ(uffd_copy.copy, static_cast<ssize_t>(length));
      break;
    } else if (errno == EAGAIN) {
      // 失败原因：EAGAIN，表示有竞争（Contention）。
      DCHECK_NE(uffd_copy.copy, 0);
      if (uffd_copy.copy > 0) {
        // 部分成功：内核在处理了 uffd_copy.copy 字节后遇到了竞争。接受这个结果。
        DCHECK(IsAlignedParam(uffd_copy.copy, gPageSize));
        break;
      } else {
        // 完全失败：内核在开始前就因为无法获取 mmap_lock 而失败。
        DCHECK_EQ(uffd_copy.copy, -EAGAIN);
        uffd_copy.copy = 0;
        if (return_on_contention) {
          // 如果调用者要求在遇到竞争时立即返回，则照做。
          break;
        }
      }
      // --- 退避（Backoff）和策略升级 ---
      if (backoff_count == -1) {
        // 根据线程优先级动态调整最大退避次数。
        int prio = Thread::Current()->GetNativePriority();
        max_backoff -= prio;
        backoff_count = 0;
      }
      if (backoff_count < max_backoff) {
        // 先进行几次短暂的等待和重试。
        BackOff</*kYieldMax=*/3, /*kSleepUs=*/1000>(backoff_count++);
      } else {
        // 如果多次重试仍然失败，则升级策略，关闭 TRYLOCK 模式。
        // 下一次 ioctl 将会是阻塞调用，会一直等待直到锁被释放。
        uffd_copy.mode = 0;
      }
    } else if (errno == EEXIST) {
      // 失败原因：EEXIST，目标页面已被映射。
      // 这是一个正常的并发场景，可能是一个mutator线程抢先完成了映射。
      DCHECK_NE(uffd_copy.copy, 0);
      if (uffd_copy.copy < 0) {
        uffd_copy.copy = 0;
      }
      // 内核返回了遇到EEXIST之前的字节数，我们需要手动加上这个已存在的页面大小。
      uffd_copy.copy += gPageSize;
      break;
    } else {
      // 其他失败情况。
      // 通常是 ENOENT，表示 userfaultfd 注册已失效（例如GC已进入FinishPhase）。
      CHECK(tolerate_enoent && errno == ENOENT)
          << "ioctl_userfaultfd: copy failed: " << strerror(errno);
      return uffd_copy.copy > 0 ? uffd_copy.copy : 0;
    }
  }
  // 返回最终成功映射的字节数。
  return uffd_copy.copy;
}
```

### Phase 6: `FinishPhase` - 结束阶段

总而言之，`FinishPhase` 函数是一个**大扫除**函数。它不做复杂的 GC 逻辑，而是专注于**清理、重置和回收资源**。其核心目标是：

1. **释放内存**：将 GC 过程中占用的所有临时内存归还给系统。
2. **恢复状态**：确保堆和分配器处于一个正确的、可用的新状态。
3. **准备未来**：重置所有 GC 相关的数据结构，为下一次垃圾回收做好准备。

```cpp
void MarkCompact::FinishPhase() {
  // 记录本次GC扫描过的字节数，用于统计和分析。
  GetCurrentIteration()->SetScannedBytes(bytes_scanned_);
  // 检查当前进程是否是Zygote进程。Zygote是Android中用于启动应用的特殊进程。
  bool is_zygote = Runtime::Current()->IsZygote();
  // 重置状态标志位，表示整理阶段和标记阶段都已经完成。
  compacting_ = false;
  marking_done_ = false;

  // 释放用于对象整理（Compaction）的临时缓冲区内存。
  ZeroAndReleaseMemory(compaction_buffers_map_.Begin(), compaction_buffers_map_.Size());
  // 释放用于存储对象信息的映射表（如对象移动后的新地址）。
  // MadviseDontNeed是一个给操作系统的提示，表明这块内存暂时不再需要，可以回收。
  info_map_.MadviseDontNeedAndZero();
  // 清除用于标记存活“字”（word）的位图。
  live_words_bitmap_->ClearBitmap();

  // 清理“可移动空间”（moving space）的位图。
  // black_dense_end_ 指向的是所有存活对象整理后末尾的位置。
  if (moving_space_begin_ == black_dense_end_) {
    // 如果没有存活对象被移动，则清空整个位图。
    moving_space_bitmap_->Clear();
  } else {
    // 否则，只清除从存活对象末尾到移动空间末尾的这部分区域的位图标记，
    // 因为这部分现在是空闲区域了。
    DCHECK_LT(moving_space_begin_, black_dense_end_);
    DCHECK_LE(black_dense_end_, moving_space_end_);
    moving_space_bitmap_->ClearRange(reinterpret_cast<mirror::Object*>(black_dense_end_),
                                     reinterpret_cast<mirror::Object*>(moving_space_end_));
  }
  // 更新“指针碰撞”（Bump Pointer）分配器的状态。
  // 告诉分配器，所有存活对象占据的“黑色密集区域”的大小，新的对象分配将从这个区域之后开始。
  bump_pointer_space_->SetBlackDenseRegionSize(black_dense_end_ - moving_space_begin_);

  // Zygote进程的特殊处理。
  if (UNLIKELY(is_zygote && IsValidFd(uffd_))) {
    // 如果是Zygote进程且使用了userfaultfd机制（用于高效的并发整理），
    // 则关闭相关的文件描述符。关闭操作会附带注销所有已注册的内存范围。
    close(uffd_);
    uffd_ = kFdUnused;
    uffd_initialized_ = false;
  }
  
  // 确保标记栈（mark stack）在GC结束时是空的，这表明所有可达对象都已被正确处理。
  CHECK(mark_stack_->IsEmpty());
  // 重置标记栈，为下一次GC做准备。
  mark_stack_->Reset();

  // 调试检查：确保执行收尾阶段的线程就是启动GC的线程。
  DCHECK_EQ(thread_running_gc_, Thread::Current());
  
  // 在Debug构建模式下，清空用于记录根（root）更新的列表。
  if (kIsDebugBuild) {
    MutexLock mu(thread_running_gc_, lock_);
    if (updated_roots_.get() != nullptr) {
      updated_roots_->clear();
    }
  }

  // 清理GC期间使用的其他临时数据结构。
  class_after_obj_map_.clear();
  linear_alloc_arenas_.clear();
  
  {
    // 加锁，以线程安全的方式清除堆上所有对象的标记位。
    ReaderMutexLock mu(thread_running_gc_, *Locks::mutator_lock_);
    WriterMutexLock mu2(thread_running_gc_, *Locks::heap_bitmap_lock_);
    // 清除所有对象的标记位，为下一次GC的标记阶段做准备。
    heap_->ClearMarkedObjects();
  }
  
  // 获取GC专用的内存池。
  GcVisitedArenaPool* arena_pool =
      static_cast<GcVisitedArenaPool*>(Runtime::Current()->GetLinearAllocArenaPool());
  // 删除内存池中不再使用的内存块（Arenas），以减少GC自身的内存占用。
  arena_pool->DeleteUnusedArenas();
}
```
