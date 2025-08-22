---
layout: default
title: GC算法和实现
nav_order: 0
parent: AndroidRuntime
has_children: true
---

## ART GC 的核心

### 1. 分代假说与 ART 堆的精细化管理：Spaces

这是现代 GC 的基石理论，它基于一个简单的观察：**绝大多数对象都是“朝生夕死”的**。ART 不仅实现了分代，更是将堆内存划分成了多种不同生命周期和用途的 **Space**，并为它们配置了不同的GC策略 (`GcRetentionPolicy`)。

* **Image Space**: `kGcRetentionPolicyNeverCollect`
  * 存放预加载的类和对象（来自 `boot.oat`）。这部分内存在 Zygote 启动时加载，所有 App 进程共享，并且是**只读**的。因此，它**永远不会被GC**。

* **Zygote Space**: `kGcRetentionPolicyFullCollect`
  * 存放 Zygote 进程创建的对象和类。App 进程通过写时复制（Copy-on-Write）的方式共享这部分内存。它通常在 App 进程中不会被回收，但在 Zygote 自身或进行堆整理时可能被处理。

* **Alloc Space (Bump Pointer Space)**: `kGcRetentionPolicyAlwaysCollect`
  * **这就是我们通常意义上的“Java堆”**，是 App 分配对象的主要场所，也是 GC 最活跃的区域。它是一个“移动空间”（Moving Space），内部的对象在GC时会被移动以消除碎片。ART 的分代策略主要在这里实现：
    * **年轻代 (Young Generation)**：通过 TLABs 在此空间上实现。新对象在这里分配。针对该区域的 **Minor GC** 采用高效的**半区复制算法**，非常频繁和快速。
    * **老年代 (Old Generation)**：存活过数轮 Minor GC 的对象会被提升（逻辑上）到老年代。当老年代空间紧张时，会触发一次 **Major GC**，采用**并发标记整理算法**。

* **Large Object Space (LOS)**: `kGcRetentionPolicyAlwaysCollect`
  * 专门用于存放超过一定大小的大对象，通常是大的原生数组。每个大对象都独占一个或多个内存页。LOS 不进行整理，而是采用标记-清除（Mark-Sweep）算法。

* **Non-Moving Space**: `kGcRetentionPolicyAlwaysCollect`
  * 一个特殊的空间，用于存放一些不希望被移动的对象，比如 `art::mirror::Class` 对象本身。这可以简化一些 VM 的内部实现。它同样采用标记-清除策略。

### 2. 三色标记法 (Tri-Color Abstraction)

为了在应用运行时**并发地**找出存活对象，ART（以及许多其他现代 GC）在概念上使用了三色标记法来追踪对象的状态：

* **白色 (White)**：对象初始状态，表示尚未被 GC 访问，是潜在的垃圾。
* **灰色 (Gray)**：对象本身已被标记为存活，但它引用的其他对象（它的“孩子”）还没有被扫描。灰色对象是 GC 的**工作集**。
* **黑色 (Black)**：对象本身和它引用的所有对象都已被扫描并标记为存活。

GC 的并发标记过程，就是从根（Roots）对象开始，将它们从白色变为灰色，然后不断从灰色集合中取出对象，将其引用的白色对象变为灰色，最后将自身变为黑色。当不再有灰色对象时，标记阶段结束，所有剩余的白色对象就是可以回收的垃圾。

### 3. 并发与分代GC的“记忆”：Card Table, Mod-Union Table 与 Remembered Set

并发标记和分代回收有一个共同的难题：如何追踪跨区域的指针变化？

* **并发标记**：需要知道应用线程在并发期间修改了哪些对象的引用。
* **分代回收 (Minor GC)**：需要知道老年代中有哪些对象引用了年轻代的对象，否则就必须扫描整个老年代来保证年轻代回收的正确性，这违背了分代的初衷。

ART 通过一套层层递进的数据结构来高效地解决这个问题。

* **Card Table (卡表) - 底层基础**
  * **是什么**：一个字节数组，将整个堆（尤其是老年代）划分为许多个小的“卡片”，卡表中的每个字节对应一个卡片。
  * **怎么工作**：ART 通过**写屏障 (Write Barrier)** 技术，在每次**写操作** (`a.field = b`) 发生时，将对象 `a` 所在卡片对应的字节标记为**“脏” (Dirty)**。
  * **特点**：非常快速和轻量。但它只提供了粗粒度的信息：“这里有指针被修改了”，但不知道是哪个指针，也不知道指向了哪里。

* **Remembered Set (记忆集) - 抽象概念**
  * **是什么**：一个逻辑上的集合，精确地记录了所有**从老年代指向年轻代**的引用。
  * **怎么工作**：在进行 Minor GC 时，GC 只需要将**线程栈等根**和 **Remembered Set 中记录的对象**作为扫描的起点，就能找到所有在年轻代中的存活对象，无需扫描整个老年代。

* **Mod-Union Table**
  * **是什么**：ART 中针对 Zygote Space 和 Image Space 的一种 Remembered Set 的具体实现。
  * **怎么工作**：
        1. 当 Zygote/Image Space（属于老年代）中的一个对象引用了 Alloc Space（年轻代）中的对象时，写屏障触发，其所在的 **Card Table** 条目被标记为“脏”。
        2. `ModUnionTable` 会在合适的时机（如 GC 工作的某个阶段）去处理这些脏卡。它会**精确地扫描**脏卡覆盖的内存区域，找出其中**具体**的、从老年代指向年轻代的指针。
        3. 然后将这些指针信息记录在一个更紧凑、更精确的列表中。
  * **关系总结**：**写屏障**是行为，**Card Table** 是粗粒度的记录，**Mod-Union Table** 负责将粗粒度的“脏卡”信息提炼成精确的 **Remembered Set**，最终服务于高效的 Minor GC。

### 4. 线程本地分配缓冲 (Thread-Local Allocation Buffers - TLABs)

在多线程环境中，如果所有线程都在同一个内存空间上分配对象，就需要加锁来避免冲突，这会成为性能瓶颈。

TLABs 是一种巧妙的优化：ART 会为每个应用线程在 **Alloc Space** 中分配一小块专属的内存区域。线程可以在自己的 TLAB 中通过简单的**指针碰撞 (Bump-Pointer)** 方式来分配对象，这个过程无需任何加锁，速度极快。只有当一个线程的 TLAB 用完后，才需要加锁去申请一块新的 TLAB。这项技术极大地提升了并发场景下的对象分配吞吐量。
