---
layout: default
title: Object-Aware Memory Compression for Smartphones
nav_order: 3
parent: 论文
author:  Anonymous Committer
---

# oaRAM

> Object-Aware Memory Compression for Smartphones
>
> ACM TACO 2025 — 上海交通大学 IPADS

oaRAM 主要针对 Android 中 ART Garbage Collection 与 Linux zRAM 内存压缩机制相互干扰的问题。

传统 Android 的 zRAM 以 Page Granularity 压缩内存，压缩后原始页面被 Unmap。当 ART 的 Tracing GC 需要遍历 Heap 中的对象时，即使只需要读取对象的引用信息，也必须通过 Page Fault 将页面从 zRAM 中解压并 Swap-in。

但 GC 访问过的很多页面对 Application Thread 来说仍然是 Cold Pages，因此很快又会被 Kernel Swap-out，形成：

```
Kernel Swap-out
      |
      v
Compress Page into zRAM
      |
      v
GC Traverses Heap
      |
      v
Page Fault
      |
      v
Decompress + Swap-in
      |
      v
Kernel Reclaims Again
      |
      v
Compress + Swap-out
      |
     ...
```

论文认为，问题的根源在于：

1. Compression Semantic Gap：Kernel 只知道 Page 的字节内容，不知道其中哪些字段是 GC 必须访问的 Object References。
2. Runtime–Kernel Access Gap：即使压缩后的数据保留了对象引用，ART GC 也无法直接访问 Kernel 中管理的压缩内存。
3. GC–Swap Interference：GC 遍历低热度页面触发大量 Swap-in，而 Kernel 又会将这些页面重新 Swap-out，造成重复压缩和解压。

论文在 10 个应用共同运行的实验中观察到，GC 只消耗约 5.1% 的总 CPU 时间，却触发了约 20.8% 的 Swap-in 操作。





为解决这些问题，oaRAM 设计了三个相互配合的核心机制：

```
Object-Aware Memory Format
    保留 Object References
    压缩 Non-reference Fields

Object-Aware Kernel Compression
    Kernel 利用 ART 提供的对象布局信息
    将 Page 压缩为 GC 可以直接解析的格式

Shadow Heap
    将 Kernel 中的压缩数据映射给 ART
    允许 GC 不经 Swap-in 直接遍历和清理压缩对象
```

其中最重要的思想是：

> GC 并不需要访问对象的所有内容，只需要访问对象图所依赖的引用信息。因此，没有必要为了 GC 遍历而完整解压整个 Page。

## Architecture

```
+------------------------------------------------------------------+
|                      User Space (ART)                            |
|                                                                  |
|  +---------------- Application / Mutator ---------------------+  |
|  |                                                          |  |
|  |  [Normal Heap]                                            |  |
|  |       |                                                  |  |
|  |       +--> Uncompressed Objects                           |  |
|  |       |                                                  |  |
|  |       +--> Swapped-out Pages --> Page Fault on Access     |  |
|  +----------------------------------------------------------+  |
|                                                                  |
|  +------------------ Garbage Collector ---------------------+  |
|  |                                                          |  |
|  |  [CMS Mark / Sweep]                                      |  |
|  |         |                                                |  |
|  |         v                                                |  |
|  |  [Shadow Offset Map]                                     |  |
|  |         |                                                |  |
|  |         +--> Normal Page: Original GC                     |  |
|  |         |                                                |  |
|  |         +--> Compressed Page                              |  |
|  |                    |                                     |  |
|  |                    v                                     |  |
|  |               [Shadow Heap]                              |  |
|  |                    |                                     |  |
|  |                    v                                     |  |
|  |          Direct Reference Access                         |  |
|  |          Mark / Sweep / Cleanup                          |  |
|  +--------------------|-------------------------------------+  |
|                       | mmap / shared metadata                |
+-----------------------|---------------------------------------+
                        |
+-----------------------v---------------------------------------+
|                       Linux Kernel                             |
|                                                               |
|  [Page Reclaim / kswapd]                                      |
|           |                                                   |
|           v                                                   |
|  [oaRAM Compression Module]                                   |
|           |                                                   |
|           +--> ART Page Map                                   |
|           |                                                   |
|           +--> Class Hashmap                                  |
|           |                                                   |
|           v                                                   |
|  [Object-Aware Compression]                                   |
|           |                                                   |
|           +--> References (Uncompressed)                      |
|           |                                                   |
|           +--> Non-reference Data (Compressed)                |
|           |                                                   |
|           v                                                   |
|  [Compressed Memory / oaRAM Storage]                          |
|           |                                                   |
|           +--> mmap to Shadow Heap                            |
|           |                                                   |
|           +--> Decompress on Mutator Swap-in                   |
|                                                               |
+---------------------------------------------------------------+
```

整体设计可以分成两条路径：

普通 Application Thread：

```
Mutator
    |
    v
Normal Heap
    |
    +--> Page Resident
    |        |
    |        v
    |    Normal Access
    |
    +--> Page Swapped Out
             |
             v
         Page Fault
             |
             v
      oaRAM Decompression
             |
             v
        Normal Access
```

ART GC Thread：

```
GC Thread
    |
    v
Check Object Page State
    |
    +--> Uncompressed
    |        |
    |        v
    |    Original GC Logic
    |
    +--> oaRAM Compressed
             |
             v
         Shadow Heap
             |
             v
      Reference Location
             |
             v
        Mark / Sweep
             |
             v
      No Full Page Swap-in
```

因此 oaRAM 并不是消除所有 Page Fault，也不是禁止 Mutator 使用 Swap，而是允许 GC 对支持的压缩页面执行对象图遍历和清理操作，而不触发完整的页面解压。







## Object-Aware Memory Format

### 1. 核心思想：Reference / Non-reference Separation

传统 zRAM 将 Page 视为一个没有语义的字节数组：

```
Original Page (4 KB)
+--------------------------------------------------+
| Object A | Object B | Object C | Free Space ... |
+--------------------------------------------------+
                       |
                       v
                 LZ4 Compress
                       |
                       v
              Compressed Byte Stream
```

GC 无法直接理解压缩字节流：

```
GC needs Object References
           |
           v
Compressed Page
           |
           v
Must Decompress Entire Page
```

oaRAM 观察到：对于 CMS 这样的 Tracing GC，Marking 需要对象的 Class Information 和 Reference Fields，而不是对象中的 Primitive Data。

因此将对象字段拆分成：

```
Object
+---------+----------+----------------+-------------------+
| K       | M        | R              | NR                |
+---------+----------+----------------+-------------------+

K  = Klass / Class Reference
M  = Object Metadata
R  = Object References
NR = Non-reference Data
```

例如 Java 对象：

```
class Node {
    Node next;
    int value;
    long timestamp;
}
```

在 oaRAM 中：

```
Reference Part:
    Klass Pointer
    next

Non-reference Part:
    Object Metadata
    value
    timestamp
```

最终变为：

```
Original Objects
        |
        v
Separate Fields
        |
        +---------------------+
        |                     |
        v                     v
References (K + R)       Non-references (M + NR)
        |                     |
        v                     v
Keep Uncompressed         LZ4 Compress
        |                     |
        +----------+----------+
                   |
                   v
             oaRAM Format
```

这里需要注意：

- `K` 不压缩，是因为 GC 需要通过 Class Information 判断对象中 Reference Fields 的数量与布局。
- `R` 不压缩，使 GC 可以直接读取引用并继续 Object Graph Traversal。
- `M` 和 `NR` 可以压缩，因为 CMS Marking 不需要读取普通对象数据。
- 不逐对象单独压缩，而是将页面内的 Non-reference Fields 汇聚后统一压缩，以改善压缩效率。

这是 oaRAM 最核心的数据格式创新。





### 2. Page-Level oaRAM Layout

虽然 oaRAM 理解 Object Semantics，但 Linux Swap 仍然以 Page 为基本单位，因此 oaRAM 需要将对象级数据重新组织成页面级压缩格式。

论文 Figure 4 给出了四个部分：

```
+----------------------------------------------------------------+
|                         oaRAM Page                             |
+----------------------------------------------------------------+
|                                                                |
|  1. Page Header                                                |
|     +------------------------------------------------------+   |
|     | Page Type / Run Metadata / Free List Information    |   |
|     +------------------------------------------------------+   |
|                                                                |
|  2. Reference Area                                             |
|     +------------------------------------------------------+   |
|     | K1 | R1 | R2 | K2 | R3 | Free Slot Count | ...      |   |
|     +------------------------------------------------------+   |
|                                                                |
|  3. Compressed Non-reference Area                              |
|     +------------------------------------------------------+   |
|     |        LZ4(Metadata + Primitive Fields)             |   |
|     +------------------------------------------------------+   |
|                                                                |
|  4. Auxiliary Index                                            |
|     +------------------------------------------------------+   |
|     | Slot Index | Reference Offset | ...                 |   |
|     +------------------------------------------------------+   |
|                                                                |
+----------------------------------------------------------------+
```

其中：

1. Page Header：包含 Page Type、Run 信息以及 Free List 元数据。
2. Reference Area：将各 Object 的 Class Pointer 和 References 连续存放，不进行压缩。
3. Compressed Area：汇聚页面内需要压缩的数据，形成一个压缩块。
4. Auxiliary Index：辅助 GC 快速定位原始 Object Address 对应的引用信息。

对于 ART 的 Free-list-based Heap，原始内存中存在 Run 和 Slot。一个 Run 可以包含多个固定大小的 Slot，某些 Slot 可能未分配对象。

oaRAM 对连续 Free Slots 使用一个计数值表示，从而允许扫描算法跳过空闲 Slot。

对于 Bump-allocator-based Heap，由于不需要维护 Run Free List，其 Page Header 更简单。





### 3. Auxiliary Index

把 Reference Fields 放在一起之后，出现一个新问题：

> GC 知道 Object 的原始虚拟地址，但它对应的 Reference Fields 已经移动到了压缩格式中的新位置，该如何查找？

oaRAM 使用 Auxiliary Index + 局部线性扫描解决。

假设原始地址为：

\\[ addr \\]

先计算其所在 Page 内的 Offset：

\\[ offset=addr\bmod PAGE\\\_SIZE \\]

对于固定 Slot Size 的 Run：

\\[ slot\\\_idx= \left\lfloor \frac{offset}{slot\\\_size} \right\rfloor \\]

然后通过 Auxiliary Index 定位到包含该 Slot 的 Reference Block。

论文默认：

\\[ N=10 \\]

即每个索引块对应约 10 个压缩对象的扫描范围。

```
Original Object Address
           |
           v
Calculate Slot Index
           |
           v
Lookup Auxiliary Index
           |
           v
Find Reference Block
           |
           v
Local Linear Scan
           |
           v
Locate Class Pointer
           |
           v
Read Class Layout
           |
           v
Locate Target Reference
```

扫描过程中：

- 遇到正常 Class Pointer：根据 Class Metadata 确定 Reference Fields 数量，跳过对应字段。
- 遇到连续 Free Slots 标记：按计数跳过相应 Slot。
- 遇到已释放对象的 Free List 信息：跳过无效引用。
- 定位目标对象后：根据 Object 内的字段偏移及 Class Layout 找到对应 Reference。

因此无需恢复原始 Page Layout，GC 就能通过原始虚拟地址找到压缩数据中的引用信息。

论文还进一步利用 Reference Fields 连续存放的特点，在一次定位后批量获取某个对象的引用。





## Object-Aware Compression Module

为了让 Kernel 能生成上述 oaRAM 格式，必须解决一个问题：

> Kernel 怎样知道某个内存地址对应的 Java Object 是什么类型，以及哪些字段是 Reference？

因为默认 Linux Kernel 不理解 ART Object Layout。

oaRAM 因此设计了一个代替普通 zRAM 压缩处理的 Kernel Module。

### 1. ART Registration

每个 ART Instance 创建后，向 oaRAM Kernel Module 注册相关信息：

```
ART Instance
      |
      v
Registration Request
      |
      +--> Heap Memory Range
      |
      +--> ART Page Map
      |
      +--> Class Hashmap
      |
      v
oaRAM Kernel Module
```

其中：

Heap Memory Range

确定某个进程的 ART Heap 在虚拟地址空间中的范围。这部分信息可以从进程的 VMA 获得。

ART Page Map

用于查询页面类型及相关布局信息，例如是否属于某个 Free-list Run、对应 Slot Size 等。

Class Hashmap

记录 Class Metadata，使 Kernel 能够确定 Object 内：

```
Reference Fields
Non-reference Fields
```

分别在哪里。

Class Hashmap 还需要随 ART 加载新 Class 而更新。

论文说明，注册过程额外传递的指针等信息只需约 16 Bytes；这不代表整个 Class Hashmap 和 Page Map 只占 16 Bytes。





### 2. Swap-out Compression

当系统决定将某个 ART Heap Page 换出时，oaRAM Kernel Module 执行：

```
Page Selected for Swap-out
             |
             v
    Identify ART Instance
             |
             v
      Lookup ART Page Map
             |
             v
    Determine Page / Slot Type
             |
             v
      Lookup Class Hashmap
             |
             v
     Traverse Object Slots
             |
             v
      Extract References
             |
             v
     Compress Other Fields
             |
             v
       Build oaRAM Page
             |
             v
     Store Compressed Data
```

具体分为：

1. 根据 Page Address 和注册信息找到对应 ART Instance。
2. 使用 Page Map 获得页面布局。
3. 遍历 Page 中的 Object Slots。
4. 利用 Class Hashmap 判断每个 Object 的字段布局。
5. 将引用字段存放到 Reference Area。
6. 将其他字段合并并压缩。
7. 保存 Page Header 和 Auxiliary Index。
8. 将构造的 oaRAM 数据交给 Kernel Compressed Memory Storage 管理。

该流程同时支持：

```
Synchronous Swap-out
    |
    +--> 由应用缺页处理中的回收需求触发

Asynchronous Swap-out
    |
    +--> 由 kswapd 在内存压力下触发
```

二者使用相同的 Object-Aware Compression 逻辑。





### 3. Swap-in Decompression

当 Application Mutator 真正访问一个已经压缩的 Object 时，不能只给它 Reference Area，因为应用还可能访问普通字段。

因此仍然需要：

```
Mutator Accesses Compressed Page
                |
                v
            Page Fault
                |
                v
        oaRAM Module
                |
                v
   Decompress Non-reference Data
                |
                v
   Reconstruct Original Object Layout
                |
                v
      Restore Normal Page
                |
                v
        Resume Mutator
```

也就是说：

oaRAM 对 GC 提供直接压缩态访问，对 Mutator 仍然执行完整的 Swap-in 和对象布局重建。

此外，对于 ART Heap 之外的 Native Memory，论文继续使用传统 zRAM 方式进行压缩，而不尝试解析 Native Object Semantics。





## Shadow Heap

即使 Kernel 已经生成可解析的 oaRAM Format，还存在第二个问题：

> GC Thread 运行在 User Space，而压缩数据存储在 Kernel 管理的内存中，GC 如何访问？

oaRAM 引入 Shadow Heap 来解决这个问题。

### 1. Shadow Heap Mapping

ART 在创建 Normal Heap 之后，通过：

```
mmap(...)
```

映射 Kernel Module 提供的设备：

```
/dev/oaRAM
```

获得一个 Shadow Heap 虚拟地址区域。

```
ART Virtual Address Space

+-------------------------------------------+
|                                           |
|              Normal Heap                  |
|                                           |
|    Object A   Object B   Object C         |
|                                           |
+-------------------------------------------+
                   |
                   | Preset Offset
                   v
+-------------------------------------------+
|                                           |
|              Shadow Heap                  |
|                                           |
|      oaRAM Compressed Page Mappings       |
|                                           |
+-------------------------------------------+
```

Shadow Heap 与 Normal Heap 具有相同的虚拟地址空间大小。

设两个 Heap 的基地址差为：

\\[ \Delta\_{shadow} \\]

则可以根据原始地址计算对应 Shadow Heap 地址：

\\[ addr\_{shadow} = addr\_{normal} + \Delta\_{shadow} \\]

注意：这只定位 Shadow Heap 中的对应虚拟页面位置，还需要后面介绍的 In-page Offset，才能定位实际压缩数据。

Shadow Heap 也不是一份完整复制出来的 Java Heap，而是用于访问 Kernel 中压缩数据的另一组虚拟映射。





### 2. Shadow Offset Map

由于 oaRAM 可以将多个压缩块打包在同一个物理 Page 中，单纯通过 Shadow Heap Address 还不足以确定压缩块的起始位置。

因此引入：

```
Shadow Offset Map
```

其每个 Entry 为 16 Bits（2 Bytes）。

```
+--------------------------------------------------+
|             Shadow Offset Map Entry              |
+---------------------------+----------------------+
| In-page Offset            | Flags                |
| 12 bits                   | 3 bits + 1 reserved  |
+---------------------------+----------------------+
```

三个状态位分别为：

- `P`（Present）：对应的压缩页面映射是否存在。
- `C`（Compressed）：该页面内容是否采用真正的压缩格式。
- `L`（Lock）：用于 GC 与 Kernel 访问该压缩数据时的同步。

这里的 In-page Offset 用于定位压缩块在对应物理页中的起始偏移。

由于：

\\[ PAGE\\\_SIZE=4096=2^{12} \\]

因此 12 Bits 足够记录页内偏移。







### 3. GC 如何访问 Shadow Heap？

```
GC Wants to Access Object
             |
             v
       Original Address
             |
             v
    Lookup Shadow Offset Map
             |
             v
     Is Compressed Page?
             |
        +----+----+
        |         |
        No       Yes
        |         |
        v         v
    Normal    Check Present
    Heap          |
                  v
             Lock Page
                  |
                  v
         Calculate Shadow Address
                  |
                  v
          Get In-page Offset
                  |
                  v
         Reference Location
                  |
                  v
          Read Object References
```

如果一个 4 KB 物理页包含多个 oaRAM 压缩块，Kernel 可以把同一物理页映射到 Shadow Heap 中的多个不同虚拟页面位置。

虽然这些虚拟地址映射到相同的物理页，但通过各自保存的 In-page Offset，GC 可以区分对应的压缩块。

因此不需要为了读取不同压缩块而复制物理页面。

## oaRAM-Compatible CMS

论文以 ART 的 Concurrent Mark Sweep（CMS） 作为主要实现对象。

核心是让 Marking 和 Sweeping 能够直接操作 oaRAM Compressed Pages。

### 1. Marking Phase

原始 CMS：

```
GC Roots
    |
    v
Read Object
    |
    v
Read Object References
    |
    v
Mark Reachable Objects
    |
    v
Continue Traversal
```

如果 Object 被 Swap-out：

```
GC
 |
 v
Page Fault
 |
 v
Swap-in
 |
 v
Decompress
 |
 v
Mark
```

oaRAM 改成：

```
GC Roots
    |
    v
Object Address
    |
    v
Check Shadow Offset Map
    |
    +--> Normal Page
    |       |
    |       v
    |   Original CMS Marking
    |
    +--> Compressed Page
            |
            v
       Shadow Heap
            |
            v
     reference_location()
            |
            v
       Read References
            |
            v
       Mark Object
```

其中 `reference_location()` 就是前面介绍的辅助索引定位算法。

对于 Compression Format 中的 Object，GC 无须恢复 Metadata 和 Primitive Fields，即可识别 Object Graph 的引用关系。

并发标记后的 Rescanning Phase 同样能够通过该方式访问压缩对象。





### 2. Sweeping Phase

CMS Marking 完成之后，通过 Mark Bitmap 区分：

```
Live Object
Dead Object
```

对于正常页面，使用原有 CMS Sweep。

对于压缩页面，oaRAM 支持直接修改保留下来的 Reference Area 和相关 Page Header 信息。

原生 CMS Free List 的基本思想是：

```
Dead Object
     |
     v
Insert into Free List
```

oaRAM 同样可以直接在压缩格式中操作 Free List：

1. 使用 Reference Location 找到 Dead Object。
2. 将其 Class Reference 所在位置改写为 Free List 指针。
3. 更新 Page Header 中的 Free List Head。
4. 将相关 Reference Fields 标记为无效，避免后续扫描误认为是正常引用。

```
Compressed Page
      |
      v
GC Sweeping
      |
      v
Find Dead Object
      |
      v
Modify Reference Area
      |
      v
Update Free List
      |
      v
Keep Non-reference Area Compressed
```

此时并不需要 Decompress 整个 Page。

之后如果 Mutator 需要重新使用该页面中的空间，再通过 Page Fault 解压并恢复正常布局。





### 3. Direct Cleanup

这是 oaRAM 的另一个重要设计。

在 ART CMS 中，当某个 Run 内的对象全部死亡时，整个 Run 可以释放。

但传统 zRAM 下，即使所有 Object 都已死亡，GC 仍可能需要 Swap-in 页面才能更新 Free List Metadata。

oaRAM 则允许：

```
Compressed Run / Page
          |
          v
GC Sweeping
          |
          v
All Objects Dead?
          |
      +---+---+
      |       |
      No     Yes
      |       |
      v       v
 Update   Direct Cleanup
 Free List     |
               v
      Remove Compressed Data
               |
               v
         Free Memory
```

即：

当压缩页面所属的 Run 已全部死亡时，可以直接从 Compressed Memory Storage 中清除相应压缩数据，不需要先 Swap-in。

这与单纯在压缩格式中支持 Marking 不同，它让 GC 可以在压缩态下完成真正的内存清理。





### 4. 对 Concurrent Copying GC 的适用性

需要特别强调：论文真正实现和评估的主要是 CMS + oaRAM。

对于 ART Concurrent Copying（CC）这种 Moving Collector，情况更加复杂：

```
CC GC
   |
   v
Mark Live Objects
   |
   v
Evacuate / Copy Objects
   |
   v
Update References
```

oaRAM 可以帮助 Marking 或部分引用更新，但 Object Evacuation 需要访问完整对象内容。

由于 oaRAM 把多个对象的 Non-reference Fields 汇聚压缩：

```
Evacuate Object
       |
       v
Need Full Object Contents
       |
       v
Decompress
       |
       v
Copy Object
       |
       v
Recompress if Necessary
```

因此不能直接声称 oaRAM 已经实现对 CC 的全流程免解压支持。

论文提出过选择性 Evacuation、在应用刚进入后台且大量页面尚未压缩时进行 Compaction 等方向，但将对 Moving Collector 的完整支持列为未来工作。





## Synchronization Between OS and ART

Shadow Heap 允许 GC 直接访问 Kernel 管理的压缩页面，但同时带来并发安全问题。

例如：

```
GC Thread
    |
    v
Access Compressed Page
           |
           | Concurrently
           v
Mutator Page Fault
    |
    v
Kernel Decompresses Page
    |
    v
Kernel Frees Compressed Data
```

如果 Kernel 在 GC 仍然访问某个压缩页面时释放它，就可能造成 Use-after-free。

因此 oaRAM 在 Shadow Offset Map 中加入原子同步协议。

### 1. GC Access

```
GC Thread
     |
     v
Atomically Acquire L Bit
     |
     v
Check P Bit
     |
     +--> Not Present
     |       |
     |       v
     |   Use Normal Heap Path
     |
     +--> Present
             |
             v
       Access Compressed Data
             |
             v
         Mark / Sweep
             |
             v
         Release L Bit
```

### 2. Kernel Decompression / Removal

```
oaRAM Kernel Module
          |
          v
Atomically Acquire L Bit
          |
          v
Clear Present State
          |
          v
Prevent New GC Access
          |
          v
Decompress / Reconstruct
          |
          v
Remove Old Compressed Data
```

其中 Kernel 使用原子操作协调 `L` 与 `P` 的变化，避免 GC 在页面已经失效后仍继续访问旧映射。

论文强调，GC 对压缩页面的临界区通常较短，仅涉及 Marking 或 Free List Reference 更新，因此同步等待时间有限。





## 模型与关键数据结构

与 HMS 不同，oaRAM 没有引入新的 Memory Pressure Prediction Model。

它的核心是 Object-Aware Compression Format + 地址映射 + 并发访问协议。

### 1. 数据访问模式

```
Mutator Access:
Original Object Layout Required
              |
              v
        Decompression

GC Access:
Reference Fields Required
              |
              v
       Direct Compressed Access
```

### 2. 主要数据结构

| 数据结构              | 作用                   |
| ----------------- | -------------------- |
| ART Page Map      | 提供页面类型、Slot Size 等信息 |
| Class Hashmap     | 提供 Object 字段布局       |
| oaRAM Page Header | 保存压缩页面及 Run 元数据      |
| Reference Area    | 保存 GC 可直接访问的引用字段     |
| Auxiliary Index   | 加速压缩对象引用定位           |
| Shadow Heap       | 将压缩数据暴露给 ART GC      |
| Shadow Offset Map | 记录映射、偏移和同步状态         |

### 3. 压缩效率的权衡

传统 zRAM：

```
Compress Entire Page
        |
        v
Better Compression Ratio
        |
        v
But GC Requires Decompression
```

oaRAM：

```
Keep References Uncompressed
        |
        v
Compress Other Fields
        |
        v
Lower Compression Ratio
        |
        v
But GC Avoids Many Swap-ins
```

论文在真实应用实验中测得：

| 指标       | Vanilla zRAM | oaRAM    |
| -------- | ------------ | -------- |
| 压缩比      | 2.92         | 2.55     |
| 平均每页压缩耗时 | 15.87 μs     | 18.81 μs |
| 平均每页解压耗时 | 7.63 μs      | 8.43 μs  |

因此，oaRAM 单次压缩/解压并不比普通 zRAM 快，压缩率反而更低。

真正的收益来自显著减少 GC 导致的重复 Swap-in / Swap-out 次数。&#x20;





## 运行时控制流

oaRAM 的运行过程可以分为两条路径：Kernel 压缩路径和 ART GC 访问路径。

### 1. ART Initialization

```
ART Instance Start
       |
       v
Create Normal Heap
       |
       v
Register with oaRAM
       |
       +--> Page Map
       +--> Class Hashmap
       |
       v
mmap(/dev/oaRAM)
       |
       v
Create Shadow Heap
       |
       v
Initialize Shadow Offset Map
```

### 2. Kernel Swap-out

```
Memory Pressure
      |
      v
Kernel Reclaim
      |
      v
Select Heap Page
      |
      v
oaRAM Compression Module
      |
      v
Read Object Layout Metadata
      |
      v
Extract Reference Fields
      |
      v
Compress Non-reference Fields
      |
      v
Build oaRAM Page
      |
      v
Store Compressed Data
      |
      v
Map into Shadow Heap
      |
      v
Update Shadow Offset Map
```

### 3. CMS Marking

```
GC Triggered
      |
      v
Scan Roots
      |
      v
Visit Object
      |
      v
Check Page State
      |
      +--> Normal Page
      |        |
      |        v
      |    Original CMS
      |
      +--> Compressed Page
               |
               v
          Acquire Lock
               |
               v
          Shadow Heap
               |
               v
        Locate References
               |
               v
         Mark Referents
               |
               v
          Release Lock
```

### 4. CMS Sweeping

```
Marking Completed
       |
       v
Sweep Objects
       |
       v
Compressed Page?
       |
       +--> No
       |     |
       |     v
       | Normal Sweep
       |
       +--> Yes
             |
             v
      Find Dead Objects
             |
             +--> Some Alive
             |       |
             |       v
             |  Update Free List
             |  in Compressed Format
             |
             +--> Entire Run Dead
                     |
                     v
                Direct Cleanup
                     |
                     v
            Release Compressed Data
```

### 5. Mutator Swap-in

```
Mutator Access
       |
       v
Normal Heap Page Missing
       |
       v
Page Fault
       |
       v
oaRAM Kernel Module
       |
       v
Synchronize with GC
       |
       v
Decompress Non-reference Data
       |
       v
Reconstruct Object Layout
       |
       v
Restore Normal Page
       |
       v
Resume Application
```

## 完整控制流

```
                            oaRAM
                              |
               +--------------+--------------+
               |                             |
               v                             v
        Kernel Compression               ART GC
               |                             |
               v                             v
         Memory Pressure                GC Triggered
               |                             |
               v                             v
         Page Swap-out                  Scan Roots
               |                             |
               v                             v
       Query ART Metadata              Visit Object
               |                             |
               v                             v
      Object-Aware Compression       Check Page State
               |                             |
         +-----+-----+                  +----+----+
         |           |                  |         |
         v           v                  v         v
      Keep K/R    Compress M/NR      Normal   Compressed
         |           |                  |         |
         +-----+-----+                  v         v
               |                    Original  Shadow Heap
               v                       GC         |
       oaRAM Compressed Page                       v
               |                          Reference Location
               v                                  |
       Kernel Storage                             v
               |                             Mark / Sweep
               |                                  |
               +------------+---------------------+
                            |
                            v
                Less GC-induced Swap-in
                            |
                            v
                Less Recompression / Swap
                            |
                            v
                 Lower Memory Overhead
                            |
                            v
              Better App Caching / Lifetime
```

## 特殊情况与限制

论文还讨论了几类不能完全避免 Swap-in 的情况。

### 1. Object Arrays

Object Array 中大部分数据本身就是 Reference。

如果全部保留为 Uncompressed，压缩率可能非常差。

因此当前 oaRAM 实现会对 Object Array 的 Reference Data 进行压缩，从而牺牲部分 GC 直接访问能力。

```
Object Array
      |
      v
Many References
      |
      v
Compress References
      |
      v
GC May Require Swap-in
```

论文统计，Reference Arrays 约占 Object Count 的 8.9%，但包含全部 References 的 35.9%。





### 2. Non-strong References

对于 Weak / Soft 等非强引用相关处理流程，当前实现没有完整支持压缩态直接访问，因此某些阶段仍可能产生 Swap-in。

### 3. Moving GC

由于 Object Evacuation 需要完整对象数据，所以对 CC 的直接支持有限，不能将 CMS 上的结果直接推广到所有 ART Collector。

### 4. 压缩比下降

oaRAM 的压缩比低于 Vanilla zRAM。在某些较大的 zRAM Budget 配置下，oaRAM 反而可能产生更多 Flash Writes。

论文发现，在约 3%–9% 的 zRAM Memory Budget 下 oaRAM 的 Flash Write 表现更有利，而预算扩大到 10%–11% 时，较低压缩比可能抵消其收益。





## 实验结果

论文在以下平台实现：

```
Google Pixel 4a
Snapdragon 730G
6 GB RAM

Android 11.0.0_r38
Linux 4.14.212
ART CMS + oaRAM
```

并使用 Synthetic Applications 和多种真实 Android 应用进行测试。





### Synthetic Workloads

在论文构造的高内存压力负载中：

- Swap-in Operations 相比 Vanilla 最多降低 99.0%。
- 相比 Fleet 最多降低 98.3%。
- Application CPU Time 最多降低 33.6%。
- Application Caching Capacity 在部分配置下达到约 1.35× 原生系统。

这些属于特定 Synthetic Workload 的实验结果，不应作为真实应用上的平均改善。







### Real-world Workloads

论文同时给出了 10 个真实 App 共同运行时的结果：

| 指标                 | Vanilla    | oaRAM      |
| ------------------ | ---------- | ---------- |
| GC In-heap Swap-in | 62.25K     | 22.93K     |
| All Swap-in        | 306.33K    | 283.21K    |
| GC Throughput      | 52.11 MB/s | 62.30 MB/s |
| Flash Write Volume | 585.24 MB  | 420.85 MB  |
| Overall CPU Time   | 247.95 s   | 246.09 s   |

即真实应用实验中：

- GC 导致的 Heap Swap-in 降低约 63.2%。
- 全系统 Swap-in 降低约 7.5%。
- GC Throughput 提升约 19.5%。
- Flash Write Volume 降低约 28.1%。

这里值得注意的是：虽然减少了大量 GC Swap-in，但全系统 CPU Time 相比 Vanilla 只略有改善，因为 GC 在整体应用运行时间中所占比例有限。论文报告的 18.2% CPU Time 改善是相对 Fleet 的比较。

## 一句话理解 oaRAM

> oaRAM 的核心是重新设计 Android 压缩内存的表示方式，将 GC 必需的 Object References 保留为未压缩状态，把其他 Object Fields 聚合压缩；同时通过 Kernel Object-Aware Compression Module、ART Shadow Heap、Shadow Offset Map 及原子同步协议，使 CMS GC 可以直接对压缩页面执行 Marking、Sweeping 和部分 Direct Cleanup，而不必反复触发 Swap-in，从而减少 GC 与 zRAM 相互干扰造成的内存抖动、压缩/解压开销及 Flash Writes。
