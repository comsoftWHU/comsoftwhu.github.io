---
layout: default
title: BumpPointerSpace
nav_order: 0
parent: GC算法和实现
grand_parent: AndroidRuntime
author: Anonymous Committer
---

## TL;DR

`BumpPointerSpace` = “**一块连续映射 + 一个全局 bump 指针 + 多个 TLAB 子块**”。

- 分配时要么原子推进 `end_`，要么给线程切一块 TLAB；

- 遍历时混合“主块顺扫 + TLAB 分块顺扫 + 黑密区用位图扫”的三套逻辑；

- 与 Mark-Compact 的衔接点是**撤销 TLAB、对齐 `end_`（黑分配起点）、导出/更新块列表**，以及可选的黑密区长度标注。这套设计让分配极简、遍历/压实可控，同时保留了在 GC 期间精确操作空间布局的能力。

## `bump_pointer_space`是什么（Space 的定位）

- 这是连续内存上的“顺推（bump）”分配空间：只靠把一个 `end_` 指针不断向前推进完成分配，不提供逐对象的 `free`（因为设计上会被 evacuate/compact 掉）。接口里也明确 `Free/FreeList` 返回 0，`CanMoveObjects()` 为真，说明对象可移动、靠 GC 迁出回收。

- 与 RegionSpace 不同，这里是单块连续空间，强调 TLAB（线程本地缓冲）与“主块(main block)”的组合来维持顺推分配；并维护一个 `block_sizes_` 队列，用于遍历/压实时知道每个 TLAB 区段的大小。

## 空间如何被创建（Create / 构造）

- `Create(name, capacity)`：按页对齐后，用 `MemMap::MapAnonymous` 在 **低 4GB** 地址空间内映射一整块可读写内存（便于 32 位引用/指针压缩场景）。成功后构造 `BumpPointerSpace`。
- 构造函数保存 `begin/limit`，把 `growth_end_` 初始化为 `limit`（增长上限），并创建一张“live bitmap”（实际走的是 `GetMarkBitmap()`，`GetLiveBitmap()` 在此类里返回空）。

## 分配路径（无锁/原子、TLAB、暂停态）

### 3.1 常规分配（线程安全、原子推进）

- `Alloc()` 会把字节数按对象对齐（`kAlignment == kObjectAlignment`）向上取整，然后走 `AllocNonvirtual()`。后者内部再调用无Accounting版本 `AllocNonvirtualWithoutAccounting()`（真正推进 `end_` 的逻辑）。
- `AllocNonvirtualWithoutAccounting()` 使用自旋 CAS：读 `old_end`，计算 `new_end = old_end + num_bytes`，若未越过 `growth_end_` 则 CAS 设置，成功后返回 `old_end` 作为对象起址。这保证了并发分配的无锁推进。
- `AllocNonvirtual()` 在成功推进后，用 `fetch_add` 记账对象数和字节数。

### 3.2 暂停态（mutator stopped）的无锁分配

- `AllocThreadUnsafe()` 要求持有独占的 mutator lock，直接加载 `end_`，检查空间，线性写回新的 `end_`，并用 `store` 做近似计数更新（避免 CAS 开销）。用于 semispace GC 等暂停态路径。

### 3.3 线程本地缓冲（TLAB）

- 为减少全局竞争，线程可向空间申请一个 TLAB：`AllocNewTlab(self, bytes, &bytes_tl_bulk_allocated)`。它会：

  1. 按对齐放大 `bytes`；
  2. 先把当前线程的 TLAB 统计“回收”到全局计数（`RevokeThreadLocalBuffersLocked`），
  3. 再用 `AllocBlock(bytes)` 在空间末尾切出一段，并把它设成线程的 TLAB（`SetTlab(start, start+bytes, start+bytes)`）。

- `AllocBlock()` 内部会先确保“主块”尺寸记录已更新，然后调用 `AllocNonvirtualWithoutAccounting()` 切块；若成功，把这块的 `bytes` 推入 `block_sizes_`（后续遍历依赖）。
- 统计口径：总的 `GetBytesAllocated()/GetObjectsAllocated()` = 全局已累计值 + 所有线程各自 TLAB 内的局部增量（需遍历线程列表汇总）。

## 遍历（Walk）：如何在“主块 + TLAB 块 + 黑密区”里找对象

`Walk` 通过“**黑密区（位图） + 主块（线性） + TLAB 块（线性，遇空即止）**”三段组合式遍历，既保证了在并发分配背景下的安全性，又兼顾了并发/压缩 GC 等特殊时期的正确性与效率。

- `Walk(visitor)` 的结构：

  - 先在持锁区复制出 `block_sizes_`（避免遍历时与分配竞争），并得到主块的结束位置 `main_end = Begin() + main_block_size_`。如没有 TLAB（`block_sizes_` 空），则仅遍历主块部分。
  - **黑密区（black-dense region）**：如果 `black_dense_region_size_ > 0`，这段对象不是连续紧密排布（比如并发 MC 的 moving-space 部分），需要借助 mark-bitmap 扫描 `[pos, pos+black_dense_size)` 的“已标记对象”再移动 `pos`；若最后一个对象跨越了边界，要把 `pos` 调整到它的下一个对齐对象处。
  - **主块扫描**：从 `pos` 到 `main_end`，把 `pos` 解释成对象指针；若对象 `klass` 还没写入（并发竞态），就停止主块扫描，否则访问并用 `GetNextObject(obj)`（= `RoundUp(obj+SizeOf, kAlignment)`）前进。
  - **TLAB 块扫描**：依次用 `block_sizes_copy` 的每个块界限定范围，从 `pos` 连续扫描对象直到遇到 `klass==nullptr`（TLAB 尾部可能未完全写满），再把 `pos` 加上该块大小。最后检查 `pos == end`。

### Walk函数详解

#### 1）函数签名与前置条件

```cpp
template <typename Visitor>
ALWAYS_INLINE void Walk(Visitor&& visitor)
    REQUIRES_SHARED(Locks::mutator_lock_)
    REQUIRES(!lock_);
```

要点：

- 需要持有 **mutator\_lock\_（共享）**：保证读对象头、类指针等是安全的。
- 要求 **当前不持有 space 的内部互斥锁 `lock_`**。函数内部会“短暂加锁”做一次快照，然后**释放锁**再做长时间的遍历，避免阻塞并发分配。

---

#### 2）关键局部变量与整体流程

核心局部变量（概念化）：

```cpp
uint8_t* pos  = Begin();   // 当前遍历指针
uint8_t* end  = End();     // 空间“逻辑末尾”（可能会调小，见下）
uint8_t* main_end = pos;   // 主块(main block)的末尾，稍后确定
size_t   black_dense_size; // 黑密区长度（若存在）
unique_ptr<vector<size_t>> block_sizes_copy; // TLAB 等“块”的大小快照
```

整体流程分三段：

1. **持锁做快照**（非常短）
2. **遍历黑密区（如果有）**：用位图（mark-bitmap）走
3. **遍历主块 + 其后的每个块（TLAB 等）**：逐对象线性走

最后用断言确保“刚好走到 `end`”。

#### 3）持锁快照：确定 main\_end、end、黑密区 & 块列表

函数先用 `MutexLock mu(Thread::Current(), lock_)` 进临界区，做三件事：

1. **更新主块大小**

   - 如果 `block_sizes_` 为空，说明现在整个空间就一个“主块”（bump 到哪算哪）。这时调用 `UpdateMainBlock()`，把 `main_block_size_ = Size()`。
   - 设 `main_end = Begin() + main_block_size_`。

2. **确定 `end`**

   - **如果没有其他块（`block_sizes_` 为空）**：把 `end` **收紧成 `main_end`**。
     这么做的原因：此时别人可能**正在**往主块末尾继续 bump 分配；为了遍历稳定，我们只遍历“快照时刻的主块末尾”为止，避免读到**正在构造**、class 尚未写好的对象。
   - **否则（存在额外块，比如 TLAB 切出来的段）**：保留 `end = End()`，并把 `block_sizes_` 复制一份到 `block_sizes_copy`（快照），以便在不持锁的情况下遍历。

3. **记录黑密区长度**

   - 把 `black_dense_region_size_` 快照到本地 `black_dense_size`（只在 **并发/压缩 GC** 的“moving space”情况下非零）。
   - 黑密区表示：**从 `Begin()` 起的头一段区域里，黑对象并非“对象对象紧密排布”，必须靠位图来找**（因为这段的分配/搬移策略会破坏“逐对象线性可走”的假设）。

#### 4）遍历黑密区（如存在）

如果 `black_dense_size > 0`：

- 用 `GetMarkBitmap()->VisitMarkedRange(begin = pos, end = pos + black_dense_size, cb)` **按位图里“被标记的对象起址”**回调。
- 这里的回调 `cb(obj)` 内部就是调用你的 `visitor(obj)`；另外把最后一个访问到的对象保存为 `last_obj`。
- 遍历结束后：

  - 先把 `pos += black_dense_size`（指针跳到黑密区尾）
  - 如果 `last_obj` **跨越**了黑密区尾（一个大对象可能覆盖到下一节区域），就把 `pos` 再校正到 `GetNextObject(last_obj)`，以保证接下来“逐对象线性遍历”**从下一个对象边界起步**。

关键点：**黑密区不保证对象紧凑**，不能用“读 class → SizeOf → 跳到下一个”的线性方法；必须按位图来找每个活对象的起始地址。同时要处理“最后一个对象伸出界”的边界对齐问题。

#### 5）遍历主块（对象线性密排）

接下来遍历主块的剩余部分：

```cpp
while (pos < main_end) {
  auto* obj = reinterpret_cast<mirror::Object*>(pos);
  // 不加 read barrier，因为 obj 可能还不是一个有效对象（刚 bump 出来但 class 未写好）
  if (obj->GetClass<kDefaultVerifyFlags, kWithoutReadBarrier>() == nullptr) {
    // 发现“未成型的对象”（类指针还没写）：说明别的线程正写入主块末端
    // ——直接把 pos 拉到 main_end 并退出这段遍历
    pos = main_end;
    break;
  } else {
    visitor(obj);                         // 交给你的回调
    pos = (uint8_t*)GetNextObject(obj);  // obj->SizeOf() 后按对齐取整
  }
}
```

这段体现了**并发友好性**：如果撞上**类指针为 null** 的“半成品对象”，就不再冒险继续猜尺寸，干脆停止主块扫描（到 `main_end` 为止）。

#### 6）遍历其他块（TLAB 等）

若 `block_sizes_copy != nullptr`（说明快照到了若干“块”，通常对应各线程的 TLAB 切块），再遍历每个块。这里有两个细节：

##### 6.1 跳过已经被“黑密区”覆盖的部分

黑密区位于开头 `[Begin, Begin + black_dense_size)`，而主块随后用线性法走到了 `pos`；`pos` 可能恰好落在**某个块的中间**。
代码用下面的方法把 `iter` 推进到“第一个**尚未**在黑密区/主块内访问过”的块，并且如果 `pos` 落在块中间，就**把该块的大小改成“剩余未访问的部分”**：

```cpp
size_t iter = 0, num_blks = block_sizes_copy->size();
for (uint8_t* ptr = main_end; iter < num_blks; iter++) {
  size_t block_size = (*block_sizes_copy)[iter];
  ptr += block_size;
  if (ptr > pos) {
    // 把“第 iter 个块”的尺寸减到还没被访问的尾段
    if ((ssize_t)block_size > ptr - pos) {
      (*block_sizes_copy)[iter] = ptr - pos;
    }
    break;
  }
}
```

##### 6.2 逐块逐对象线性遍历

对剩下的每个块：

```cpp
for (; iter < num_blks; ++iter) {
  size_t block_size = (*block_sizes_copy)[iter];
  auto* obj     = reinterpret_cast<mirror::Object*>(pos);
  auto* end_obj = reinterpret_cast<const mirror::Object*>(pos + block_size);
  CHECK_LE(reinterpret_cast<const uint8_t*>(end_obj), End());

  // 在这个块内线性扫对象；遇到 class==nullptr 视作块结束
  while (obj < end_obj &&
         obj->GetClass<kDefaultVerifyFlags, kWithoutReadBarrier>() != nullptr) {
    visitor(obj);
    obj = GetNextObject(obj);
  }
  pos += block_size;   // 跳到下一个块的开始
}
```

- 为什么块内也要判 `class==nullptr`？
  因为 **TLAB 可能没被完全写满**，尾部尚未形成有效对象；一旦撞到“半成品”，就当作此块结束。

最后有两条一致性检查：

```cpp
if (!block_sizes_copy) CHECK_EQ(end, main_end); // 没有块就应该只走到 main_end
CHECK_EQ(pos, end);                             // 最终必须走到快照的 end
```

---

#### 7）并发与正确性设计要点

- **短锁 + 脱锁长遍历**：只在开头持 `lock_` 快照 `main_block_size_ / block_sizes_ / black_dense_region_size_`，随后释放锁遍历，减少与分配线程的冲突。
- **类指针哨兵**：用 `obj->GetClass(..., kWithoutReadBarrier)` 判定对象是否“成型”。发现 `nullptr` 就停止当前段的线性遍历，避免读坏内存或误解析大小。
- **黑密区必须用位图**：那里对象并不保证紧密排列；位图能精确地给出“真实活对象”的起始地址。为了与后续线性遍历衔接，记录 `last_obj` 并把 `pos` 对齐到 `GetNextObject(last_obj)`。
- **块表在堆外维护**（`std::deque<size_t> block_sizes_` 的快照）：这让 **Mark-Compact** 在决定新布局时能随意合并/吞并块（典型是把活对象尽量压回主块），而 `Walk` 也能据此可靠地定位每个块的边界。

---

#### 8）你在写 `visitor` 时要注意什么？

- **不要依赖持有 `lock_`**：`Walk` 在遍历阶段是**不持锁**的，避免在回调里做会与分配路径互相等待的动作。
- 可以安全地**读对象字段/klass**（有 mutator\_lock\_ 共享保护）。
- 如果 `visitor` 会“移动/重定位”对象，那就别在普通 `Walk` 用它；移动需配合专门的压实流程与写屏障策略。

---

#### 9）伪码

```cpp
Walk(visitor):
  pos = Begin()
  end = End()
  main_end = pos
  block_sizes_copy = nullptr
  black_dense_size = 0

  // --- 快照阶段（短锁） ---
  lock_(mu)
    if block_sizes_.empty():
      UpdateMainBlock()                     // main_block_size_ = Size()
    main_end = Begin() + main_block_size_
    if block_sizes_.empty():
      end = main_end                        // 只遍历到主块快照末尾
    else:
      block_sizes_copy = copy(block_sizes_) // TLAB 等块大小
    black_dense_size = black_dense_region_size_
  unlock(mu)

  // --- 黑密区（若有）按位图走 ---
  if black_dense_size > 0:
    last_obj = nullptr
    bitmap.VisitMarkedRange(pos, pos+black_dense_size,
                            [&](obj){ visitor(obj); last_obj = obj; })
    pos += black_dense_size
    if (last_obj != nullptr):
      pos = max(pos, GetNextObject(last_obj)) // 处理“越界的大对象”

  // --- 主块线性走 ---
  while pos < main_end:
    obj = (Object*)pos
    if obj->GetClass(..., kWithoutReadBarrier) == nullptr:
      pos = main_end   // 撞到半成品，结束主块扫描
      break
    visitor(obj)
    pos = (uint8_t*)GetNextObject(obj)

  // --- 其他块（TLAB 等） ---
  if block_sizes_copy:
    // 跳过已被黑密区+主块覆盖的部分；若 pos 在块中间，缩小该块的 size
    advance iter to first block whose end > pos; adjust first block size (end - pos)
    // 逐块线性扫，class==nullptr 视为块尾
    for each remaining block_size:
      obj = (Object*)pos; end_obj = (Object*)(pos + block_size)
      while obj < end_obj and obj->GetClass(..., no_rb) != nullptr:
        visitor(obj)
        obj = GetNextObject(obj)
      pos += block_size
  else:
    assert(end == main_end)

  assert(pos == end)
```

## 与 Mark-Compact（MC）收紧/挪动的配合

- **TLAB 撤销**：MC 标记阶段会撤销全部线程的 TLAB，确保累计计数准确，避免遗留“悬空”的私有增量。
- **黑色新分配起点**：MC 在停顿前把 `end_` 对齐到页边界（`AlignEnd(gPageSize)`），后续的新分配都算“黑（已存活）”，并能确保卡表清理的区域对齐。这就是“黑密区”的开始地址。
- **块尺寸与重排**：MC 需要按照 `block_sizes_` 知道各 TLAB/区段的边界（尤其是标记完成后分块滑动压实）。它通过 `GetBlockSizes()` 取回主块大小与各块列表，完成新布局决定后用 `SetBlockSizes(main_block_size, first_valid_idx)` 吞并前若干块到主块，并把 `end_` 设到新的“已占用末端”，保证空间内部一致。
- **黑密区尺寸**：MC 在完成后可调用 `SetBlackDenseRegionSize(size)`，告诉 Walk 哪段必须用位图遍历（因为这段不是紧密排布）。

## 容量限制、清空与碎片报告

- **Capacity / GrowthLimit**：`Capacity()` 返回 `growth_end_ - begin_`（可能被限制）；`NonGrowthLimitCapacity()` 返回底层 `MemMap` 的映射总大小。
- **ClampGrowthLimit(new\_cap)**：在仅允许与 TLAB 并发的条件下，尝试把 `Limit` 收紧到 `new_cap`；会根据当前空闲推导实际能收紧的值，同时更新 `MemMap` 的可见大小与 `mark_bitmap` 的 heap 大小。
- **Clear()**：将整段内存 `madvise(..., MADV_DONTNEED)` 返还给内核（在某些配置下先 `memset`），把 `end_` 复位到 `Begin()`，清零统计，并清空 `block_sizes_`。
- **碎片诊断**：`LogFragmentationAllocFailure()` 会报告“可分配的最大连续块 = Limit() - End()”，若申请超出就判定为“碎片导致失败”。虽然是连续空间，但在收紧 limit 或切块后，这个提示有助于定位失败原因。

## 其他要点

- **对象大小与可用尺寸**：`AllocationSizeNonvirtual(obj, usable_size)` 直接取 `obj->SizeOf()`，可根据对齐上取整返回 `usable_size`。
- **对齐辅助**：`GetNextObject(obj)` 用对象真实大小加上对齐算下一个候选对象地址；Walk 和扫描大量使用它。
- **统计汇总**：撤销 TLAB 时把每线程的对象数/字节数累加进全局原子计数，然后 `ResetTlab()`。
