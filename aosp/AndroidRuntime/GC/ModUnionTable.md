---
layout: default
title: ModUnionTable数据结构
nav_order: 6
parent: GC算法和实现
author: Anonymous Committer
---

## TL;DR

GC 在增量/分代/黏性（sticky）等场景里，需要反复根据 **Card Table** 去找“被修改过的对象区域”。但卡表会越积越脏，导致每次都要扫描很多卡，成本高。
**Mod Union Table（MUT）** 的目标是：在若干次 GC 之间**安全清空卡表**，但把“真正可能跨区引用的那一小撮卡”**提炼出来缓存**，只扫这些，**显著缩小扫描集合**。

直白说：MUT ≈ “脏卡的精炼版”，让我们能清卡表、少扫描。它让我们能**清卡表**却不丢关键信息，并在后续 GC 中只扫描一小撮真正相关的卡/字段，从而**降低增量/分代/黏性 GC 的开销**；两种实现分别在**精度（字段级）**与**轻量（卡级）**之间做取舍。

## 核心思路（流程）

以一个空间 `space_`（如 Zygote 或 Image/Alloc 空间的一段连续空间）为例：

1. **ProcessCards()**

   * 扫描 `space_` 的卡表，把当前标记为 **Dirty** 的卡**收集起来**，同时**把卡表那一段清理/老化**（通过 `CardTable::ModifyCardsAtomic(..., AgeCardVisitor(), ...)`）。
   * 注意：此时只是“记下哪些卡脏过”，并没有立刻展开对象字段检查。

2. **UpdateAndMarkReferences(visitor)**
   * 对第 1 步缓存下来的卡，借助空间的 **Live Bitmap**（只看存活对象）在卡覆盖的地址范围内 **VisitMarkedRange**。
   * 找到对象里的引用字段，按策略“是否要纳入”（见下文 `ShouldAddReference`）进行处理：

     * **ReferenceCache 版本**：把**具体的字段地址（HeapReference 指针）**缓存起来，并用 `visitor->MarkObject` 标记/更新（移动 GC 时可回写新地址）。
     * **CardCache 版本**：只记住“这张卡上**确实**有跨区引用”；若这张卡上**没有**对“其他空间（非免疫空间）”的引用，会清掉它在 MUT 的标记。
   * 这一步天然就把“对同区/免疫区（如 boot image）的引用”过滤掉了，**只留下跨区引用的卡或引用地址**。

3. （可选）**FilterCards()**

   * 用一个空的 `MarkObjectVisitor` 调 `UpdateAndMarkReferences`，会**自动把不需要的卡／引用过滤掉**（例如不再有跨区引用的就被清除）。

4. **VisitObjects() / Verify() / ContainsCardFor() / ClearTable() / SetCards()**

   * 辅助接口：遍历受影响对象、做一致性校验、查询某地址是否落在记录的卡中、强制设置整段卡、清空缓存等。

## 两种实现（抽象类 + 两个具体子类）

抽象基类：`ModUnionTable`

* 统一定义了上面提到的虚函数与公共数据：`heap_`、`space_`、`name_`。

### 1) 引用缓存版：`ModUnionTableReferenceCache`

* **缓存粒度更细**：对每张卡，记录“这张卡上**哪些具体 HeapReference 字段**可能指向目标区域”。

  * 成员：

    * `cleared_cards_`：已清理的卡集合（地址集合）。
    * `references_`：`card_ptr -> vector<HeapReference*>`。
* **ProcessCards()**：把脏卡插入 `cleared_cards_`，并清理/老化卡表那一段。
* **UpdateAndMarkReferences()**：

  * 针对 `cleared_cards_` 中的每张卡，用 **Live Bitmap** 只遍历该卡覆盖的存活对象。
  * 用访问器把**满足条件**的字段指针塞进 `references_`；同时用 `visitor->MarkObject` 进行标记/更新。
  * **特殊情况：GcRoot 压缩引用**（例如类加载器里有集合存放 GcRoot）：如果卡上出现“匹配选择条件的 GcRoot”，就**保留该卡**在 `cleared_cards_` 里（因为这些根的物理位置可能变化，不能假设稳定），以便下次继续处理。
  * **空引用清除**：下次再跑时会检查缓存的 `HeapReference*` 是否都变成了 null，如果“**这张卡的所有已缓存字段都为 null**”，就把这张卡从 `references_` 中移除（因为**写 null 不触发卡标记**，这里通过主动检查来“自愈”）。
* **ShouldAddReference(const Object* ref)**：留给子类决定“哪些引用需要记录”。
  * 例如 `ModUnionTableToZygoteAllocspace`：只要 `ref` **不在本 `space_`**，就认为是“值得记录的外部引用”，常用于 **Image/Zygote → Alloc** 的跨区跟踪。
* **Verify()**：

  * 校验 `references_` 里记录的对象都仍然**存活**；
  * 若卡现在在卡表里是 **Clean** 但 `references_` 仍声称它有外部引用，会再遍历这张卡的对象验证，发现不一致会报错（帮助发现漏标/不一致）。

**优点**：精确到字段地址，扫描最小化；移动 GC 时可直接更新这些字段。
**代价**：维护 `references_` 的内存/时间开销更高，逻辑更复杂。

### 2) 卡缓存版：`ModUnionTableCardCache`

* **缓存粒度更粗**：只记录“**哪些卡** 仍需要关注”，用一个 **bitmap**（`CardBitmap`）在 `space_[Begin, Limit)` 上标 bit。
* **ProcessCards()**：把本次发现脏的卡对应的 bit 设上，同时清理/老化卡表。
* **UpdateAndMarkReferences()**：

  * 对 bitmap 上的**置位 bit**（每个 bit 对应一张卡跨度，`CardTable::kCardSize`），利用 **Live Bitmap** 遍历卡覆盖的对象，用访问器查看是否有“对**非免疫空间**（通常是 boot image 外的空间）”的引用：

    * 找到则用 `visitor->MarkObject` 标记/更新；
    * **如果没找到外部引用**，就把该 bit 清掉（这张卡不用再记了）。
  * 访问器里有“免疫空间（immune\_space）”的概念：对免疫区的引用可以忽略（例如从 Image 指向 Image 本身无需记录）。
* **ContainsCardFor(addr)**：看 bitmap 对应位是否为 1。

**优点**：数据结构简单、内存小。
**代价**：每次仍需在卡范围内**重扫对象字段**（尽管只在“缩小后的卡集合”上进行），没有字段级的缓存与空引用自愈的粒度。

### 什么时候用哪一种？

* **Image/Zygote 指向 Alloc** 的典型场景、并且希望**最小化重复字段扫描** → `ReferenceCache` 版更合适（字段粒度缓存，移动 GC 可直接回写）。
* 空间简单、追求实现/内存更轻量、可接受每次对卡范围再次遍历字段 → `CardCache` 版。
