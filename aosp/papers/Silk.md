---
layout: default
title: Silk-Runtime-Guided Memory Management for Reducing Application Running Janks on Mobile Devices
nav_order: 3
parent: 论文
author:  Anonymous Committer
---

# Silk

> **Silk: Runtime-Guided Memory Management for Reducing Application Running Janks on Mobile Devices**

Silk 主要针对 Android 中 ART 与 Linux Kernel 对“内存热度”的理解不一致问题。

Linux Kernel 只能在 **page granularity** 上执行 LRU 和 swap，而 Java/Kotlin 应用实际上以 **object granularity** 访问 ART heap。一个 4 KB page 中可能存在几十甚至上百个不同热度的对象，因此 page hotness 并不能准确代表 object hotness。另一方面，GC 对对象的扫描又会污染 Kernel LRU 所观察到的访问历史，使 GC-cold object 被误认为 hot、GC-hot object 被错误 swap out。论文将这两类现象统称为 **Object Hotness Inversion**。

Silk 因此设计两个互补组件：

```txt
MO-App
    解决 Application Thread 的 Object Hotness Inversion

MO-GC
    解决 GC Thread 的 Object Hotness Inversion
```

---

## Architecture

```txt
+---------------------------------------------------------------------+
|                     Android Framework / ART                         |
|                                                                     |
|                    [FG App Detector]                                |
|                           |                                         |
|                           +----------------------+                  |
|                                                  |                  |
|  Application Thread                              |                  |
|        |                                         |                  |
|        v                                         v                  |
|  [ART JIT Compiler]                        [GC Profiler]             |
|        |                                         |                  |
|        | sampled object access                   | FG/BG state       |
|        v                                         | region type       |
|  [Object Access Time]                            | GC history        |
|  [stored in Lock Word]                           |                  |
|        |                                         v                  |
|        v                                [GC Region Hotness]         |
|  [MO-App Hotness Recognition]                    |                  |
|        |                                         |                  |
|        v                                         |                  |
|  [CC GC Copying Phase]                           |                  |
|        |                                         |                  |
|        v                                         |                  |
|  [Group Objects by Hotness]                      |                  |
|        |                                         |                  |
|  Hot/Warm/Cold Regions                           |                  |
+--------------------------------------------------|------------------+
                                                   |
                                      Runtime GC semantics
                                                   |
+--------------------------------------------------v------------------+
|                        Linux Kernel                                 |
|                                                                     |
|                  [GC Hotness Metadata]                              |
|                           |                                         |
|                           v                                         |
|             [kswapd / isolate_lru_pages()]                         |
|                           |                                         |
|              coldest GC regions first                              |
|                           |                                         |
|                           v                                         |
|                     [Page Reclaim]                                  |
|                           |                                         |
|                           v                                         |
|                   [Swap Partition]                                  |
|                                                                     |
|       if insufficient reclaim -> fallback to normal LRU             |
+---------------------------------------------------------------------+
```

Silk 的核心其实是两种不同的 Runtime-Guided Memory Management：

```txt
MO-App:

Object Hotness
      ↓
改变 object physical layout
      ↓
让 page hotness 更接近 object hotness
      ↓
Kernel 原本的 LRU 就能做出更正确的判断
```

而：

```txt
MO-GC:

GC Semantics
      ↓
Region Hotness
      ↓
直接指导 Kernel reclaim page 的选择
```

所以 **MO-App 主要改变对象布局，MO-GC 主要改变内核 reclaim 的 page-selection policy**。这也是理解整个 Silk 最重要的一点。


# MO-App

## 目标

MO-App，即：

> **Managing Objects with Hotness for Application Threads**

用于解决 Application Thread 的 **Object Hotness Inversion**。

普通 Android 中：

```txt
Page A
+------------------------------------------------+
| Hot Object | Cold | Cold | Cold | Cold | Cold |
+------------------------------------------------+
```

Kernel 只能看到：

```txt
Page A recently accessed
        ↓
Page A = Hot
```

于是里面大量真正 cold 的对象也因为与一个 hot object 共处一页而被认为是 hot。

论文实验中，一个 ART page 往往包含大量对象，并且 hottest pages 中平均约 **85.1% 的对象实际上属于 pseudo-hot objects**。
Silk 希望最终形成：

```txt
Hot Page
+--------------------------------+
| Hot | Hot | Hot | Hot | Hot    |
+--------------------------------+

Warm Page
+--------------------------------+
| Warm | Warm | Warm | Warm      |
+--------------------------------+

Cold Page
+--------------------------------+
| Cold | Cold | Cold | Cold      |
+--------------------------------+
```

从而让：

```txt
Page Hotness ≈ Object Hotness
```

---

## 1. Tracking Object Access

Silk 修改 ART 的 JIT Compiler，在编译应用代码时插入少量 tracking instructions。

例如：

```cpp
obj1 = obj2.f;
```

或：

```cpp
obj2.f = obj3;
```

访问的实际对象是：

```txt
obj2
```

因此 Compiler 可以在这类 object access 发生时记录 `obj2` 最近一次被访问的时间。

但是如果每一次 object access 都记录，CPU 开销会很大。

Silk 因此采用：

```txt
Sampling
+
Object Hotness Inheritance
```

论文的观察是：

```txt
Object A 被访问
      ↓
A 引用的对象
      ↓
很可能在之后也被访问
```

即对象访问存在一定的 **hotness inheritance**。

因此没有必要跟踪所有对象，只需要：

```txt
Sample Object
     ↓
记录 Hotness
     ↓
沿 reference chain
     ↓
推测附近对象的 Hotness
```

论文默认 tracking sampling interval 设置为 **3**。同时，在后续热度推断时，对没有直接采样到的对象，会检查 reference distance \(k=3\) 内已经记录过的对象。

## 2. 使用 Lock Word 记录 Access Time

如果为每个 Java object 单独增加一个字段，例如：

```txt
Object Header
+8 Bytes
```

由于 Android 中大量对象只有几十字节，会造成很高的额外内存开销。

Silk 发现绝大多数对象长期处于：

```txt
Unlocked
```

状态，而 ART 的 32-bit lock word 在该状态下存在大量未使用 bit。

因此直接复用 lock word：

```txt
31                 28 27             16 15                 0
+--------------------+-----------------+---------------------+
|    State Flag      | Accessed Flag   | Last Access Time    |
+--------------------+-----------------+---------------------+
      4 bits              12 bits             16 bits
```

其中：

```txt
Accessed Flag = 0x000
    ↓
普通 Unlocked

Accessed Flag = 0xFFF
    ↓
Accessed State
```

低 16 bit 用于记录：

```txt
Object Last Access Time
```

因此不扩大 Object Header，论文称其为 **zero additional memory overhead**。
---

### Lock 状态变化

Silk 还需要处理 lock word 本身原来的用途。

例如：

```txt
Accessed
   ↓ lock()
ThinLocked
```

进入 ThinLocked/FatLocked 时，原先保存的 access time 会被覆盖。

因此 unlock 时 Silk 会重新：

```txt
Object -> Accessed
          +
Current Access Time
```

对于 GC relocation：

```txt
Original Lock Word
        ↓
save
        ↓
ForwardingAddress
        ↓
GC Copy Object
        ↓
restore
        ↓
Original Lock Word
```

所以 GC 搬迁对象之后，其 access information 仍可以保存。

---

## 3. Object Hotness Recognition

MO-App 根据：

```txt
Last Access Time
+
LRU-like Policy
```

判断 object hotness。

可以把它抽象理解为：

$$
Age(o)=T_{now}-T_{access}(o)
$$

其中：

```txt
Age 小
    ↓
最近访问
    ↓
Hot

Age 大
    ↓
长期没有访问
    ↓
Cold
```

论文实验中将 object hotness 划分为 **9 个 level**：

```txt
Hot1
Hot2
Hot3

Warm1
Warm2
Warm3

Cold1
Cold2
Cold3
```

其本质仍然是依据最近访问时间的 LRU-style classification。
对于没有直接记录 access time 的对象：

```txt
Accessed
    ↓
直接使用记录的 Access Time

Unlocked
    ↓
寻找 reference distance <= 3
附近的 Accessed Object
    ↓
继承其中最新的 Access Time

ThinLocked / FatLocked
    ↓
认为 Hot

HashCode
    ↓
认为 Cold
```

`ForwardingAddress` 状态则在进入对象复制前完成 hotness 判断。

---

## 4. 借助 GC 实现 Object Grouping

知道 object hotness 之后还有一个问题：

> 怎么把相似热度的对象放在一起？

如果 Silk 自己主动移动 object：

```txt
Move Object
    ↓
修改所有引用
    ↓
可能触发 swap-in
    ↓
大量 Read / Write I/O
```

开销会非常大。

Silk 因此没有额外执行 object relocation，而是**复用 ART Concurrent Copying GC 本身的 Copying Phase**。

原本 CC GC 就要执行：

```txt
Victim Region
      ↓
copy live objects
      ↓
Free Region
      ↓
update references
```

Silk 将其变成：

```txt
Victim Region
      ↓
Recognize Object Hotness
      ↓
GC Copy
      ↓
choose destination region according to hotness
      ↓

Hot Object  ──→ Hot Region
Warm Object ──→ Warm Region
Cold Object ──→ Cold Region
```

所以：

- 不需要额外把 swapped-out object 读回来；
- 不需要额外完成一次 reference fix-up；
- 利用了 GC 本来就必须进行的 object copying。

ART 的一个 Region 为：

$$
256\text{ KB}
$$

也就是：

$$
256\text{ KB}/4\text{ KB}=64
$$

个物理 page。

当相似 hotness 的对象被放入同一个 Region 后，其对应的物理 pages 中 object hotness 也会更加一致，从而减少 Object Hotness Inversion。
因此 MO-App 的核心链路就是：

```txt
Object Access
      ↓
Compiler Sampling
      ↓
Record Last Access Time
      ↓
Recognize Hotness
      ↓
GC Copying
      ↓
Group Similar-Hotness Objects
      ↓
Page Hotness ≈ Object Hotness
      ↓
Normal Kernel LRU becomes more accurate
```

---

# MO-GC

## 目标

MO-GC：

> **Managing Objects with Hotness for GC Threads**

解决另一种 completely different hotness：

```txt
Application Hotness
!=
GC Hotness
```

一个 object 即使 application thread 不访问它，也可能被 GC 经常扫描。

反过来，某对象刚刚因为 GC scan 被访问：

```txt
GC scans object
      ↓
Kernel sees memory access
      ↓
LRU says "Hot"
```

但这个 region 可能之后很长时间都不会再被 GC 扫描。

于是 Kernel 观察到的：

```txt
Page Access Recency
```

并不能代表真正的：

```txt
Future GC Access Probability
```

这就是 **GC Thread Object Hotness Inversion**。

---

## 1. GC Hotness Recognition

Silk 根据两个主要信息判断 GC hotness：

```txt
Application State
    FG / BG

        +

Region Type
    New / Old
```

ART 的 Generational Concurrent Copying 中：

```txt
New Region
    ↓
Minor GC 和 Major GC 都经常扫描
    ↓
GC Hot

Old Region
    ↓
主要由 Major GC 选择性扫描
    ↓
GC Colder
```

同时：

```txt
Foreground App
    ↓
allocation 快
GC 频率高
    ↓
GC Hotter

Background App
    ↓
allocation 慢
GC 频率低
    ↓
GC Colder
```

最终 Silk 得到四级顺序：

$$
FG-N > BG-N > FG-O > BG-O
$$

即：

```txt
GC Hot
  ↑
FG-New

BG-New

FG-Old

BG-Old
  ↓
GC Cold
```

其中，对于不同 Background Applications 的 `BG-O`，Silk 还会使用历史 GC collection frequency 进一步判断热度。

---

## 2. Region-Level Hotness Metadata

MO-GC 不需要像 MO-App 一样给每个 object 保存 GC hotness。

因为 GC 本身工作的基本单元就是：

```txt
Region = 256 KB
```

所以直接以 **Region granularity** 保存 metadata。

其结构可以理解为：

```txt
+------------------------------------------+
| PID | Region Hotness | Region ID List   |
+------------------------------------------+
|6793 |      Cold      | 15, 207, ...      |
+------------------------------------------+
```

Region 地址通过：

$$
reg_{start}
=
regionspace_{start}
+
reg_{ino}\times reg_{len}
$$

其中：

$$
reg_{len}=262144\text{ bytes}=256\text{ KB}
$$

以及：

$$
reg_{end}=reg_{start}+reg_{len}
$$

来定位。
默认一个应用最多管理 4096 个 Region，因此每个应用的 GC-hotness metadata 大约：

$$
4096\times4B=16KB
$$

10 个同时运行的应用大约只有：

$$
160KB\approx0.16MB
$$

额外 metadata。

---

## 3. Runtime-Guided Kernel Reclaim

当内存不足时，Linux：

```txt
Memory Pressure
      ↓
kswapd
      ↓
isolate_lru_pages()
      ↓
select pages
      ↓
reclaim / swap out
```

原生 Linux 主要依据：

```txt
Page-level LRU
```

判断 page 是否值得 reclaim。

Silk 修改 `isolate_lru_pages()` 的 page-selection policy：

```txt
kswapd
   ↓
isolate_lru_pages()
   ↓
query Runtime GC hotness
   ↓
GC-Coldest Regions
   ↓
reclaim their pages first
   ↓
warmer regions
   ↓
...
```

也就是说：

```txt
BG-Old
  ↓
优先回收

FG-Old
  ↓

BG-New
  ↓

FG-New
  ↓
尽量保留
```

直到 memory pressure 得到缓解。

如果 GC-cold region 无法提供足够的 reclaimable memory：

```txt
Silk policy
     ↓
not enough pages
     ↓
fallback
     ↓
Native Kernel LRU
```


这里和 HMS 有一个非常重要的区别：

> **Silk 本身主要没有改变“什么时候发生 reclaim”；kswapd 仍然是在 memory pressure 下被触发。Silk重点改变的是“reclaim 哪些 page”。**

---

# 模型

Silk 实际上存在两套不同的 Hotness Model。

---

## Application Hotness Model

针对 application thread：

```txt
            Last Access Time
                   |
                   v
        LRU-style Classification
                   |
       +-----------+-----------+
       |           |           |
      Hot         Warm        Cold
```

更进一步划分成 9 个等级：

```txt
Hot1 / Hot2 / Hot3
Warm1 / Warm2 / Warm3
Cold1 / Cold2 / Cold3
```

其中核心变量是：

$$
T_{access}(o)
$$

因此可抽象成：

$$
H_{app}(o)
=
f(T_{now}-T_{access}(o))
$$

需要强调：**这个公式是对论文机制的抽象表达，不是论文给出的显式公式**。论文实际实现使用基于 last access time 的 lightweight LRU threshold classification。

---

## GC Hotness Model

GC Thread 使用的不是 access timestamp，而是 Runtime semantics：

$$
H_{gc}
=
f(
ApplicationState,
RegionType,
GCHistory
)
$$

最基本的排序：

$$
FG-N > BG-N > FG-O > BG-O
$$

所以 Silk 对同一块内存实际上维护两种不同意义上的 hotness：

```txt
Object
 ├── App Hotness
 │      "Application thread 会不会很快访问？"
 │
 └── GC Hotness
        "GC thread 会不会很快扫描？"
```

这一点非常重要。

一个 object 完全可能：

```txt
App-Cold
GC-Hot
```

或者：

```txt
App-Hot
GC-Cold
```

Silk 不试图用一个统一 hotness 指标描述二者，而是分别优化。

---

# 运行时控制流

Silk 的运行过程最好拆成两条路径理解。

---

## Path 1：Application Thread

### 1. Application executes

```txt
Java / Kotlin Code
       ↓
DEX
       ↓
ART JIT Compiler
       ↓
Native ARM64
```

Silk 修改 compiler，使其可以在 object access 上插入轻量 tracking code。

---

### 2. Sample Object Access

```txt
App Thread
    ↓
Object Access
    ↓
Compiler Sampling
    ↓
record current access time
```

访问时间保存在 object lock word 的空闲 bit 中。

默认：

```txt
Sampling Interval = 3
```

论文测得这一配置下 object-access tracking 平均 CPU overhead 大约 **1.2%**。

---

### 3. GC Trigger

ART 正常触发 Concurrent Copying GC：

```txt
GC
 ↓
Mark
 ↓
Copy
 ↓
Reclaim
```

Silk 不额外遍历整个 heap 来识别 hotness。

---

### 4. Hotness Recognition

在 GC 即将 copy live object 时：

```txt
Object
   ↓
Read Lock Word
   ↓
Last Access Time
   ↓
LRU Hotness Recognition
```

如果对象没有直接采样：

```txt
Reference Neighborhood
        ↓
distance <= 3
        ↓
find Accessed Objects
        ↓
inherit newest access time
```

由于需要处理的 object 本来就被 GC 拉入内存，所以避免为了 hotness analysis 单独触发 swap-in。

---

### 5. GC Copy + Object Grouping

```txt
Live Object
     ↓
Hotness
     ↓
GC Copy
     ↓
Corresponding Region
```

例如：

```txt
Hot Objects
   ↓
Hot Region

Cold Objects
   ↓
Cold Region
```

最终：

```txt
Object Hotness Alignment
          ↓
Page Hotness Alignment
          ↓
Kernel LRU becomes more accurate
```

---

## Path 2：GC Thread / Kernel

### 1. Runtime Collects GC Semantics

```txt
Android Framework
      ↓
FG App Detector

ART
 ↓
Region Type
New / Old

GC Profiler
 ↓
Historical GC Frequency
```

---

### 2. Calculate GC Region Hotness

```txt
FG-New > BG-New > FG-Old > BG-Old
```

并建立：

```txt
PID
  ↓
Region Hotness
  ↓
Region ID List
```

---

### 3. Memory Pressure

当系统进入 low-memory condition：

```txt
Memory Pressure
      ↓
kswapd
```

---

### 4. Runtime-Guided Page Selection

原生方式：

```txt
kswapd
  ↓
LRU
  ↓
Cold Page
```

Silk：

```txt
kswapd
   ↓
isolate_lru_pages()
   ↓
Runtime GC Hotness
   ↓
GC-Cold Region
   ↓
Pages in GC-Cold Region
   ↓
Reclaim First
```

---

### 5. Swap Out

```txt
GC-Cold Page
     ↓
Reclaim / Swap Out
```

从而尽可能把：

```txt
GC-Hot Pages
```

保留在 DRAM 中。

这样下一次 GC：

```txt
GC
 ↓
scan GC-Hot Region
 ↓
still resident
 ↓
less swap-in
 ↓
less CPU / memory-bandwidth contention
 ↓
less application jank
```

如果 runtime-guided reclaim 不够，则回退到默认 Kernel LRU。

---

# 完整控制流

```txt
                           Silk
                            |
              +-------------+-------------+
              |                           |
              v                           v
           MO-App                       MO-GC
              |                           |
              |                           |
     Application Object Access        FG/BG State
              |                       Region Type
              v                       GC History
       JIT Compiler Sampling              |
              |                           v
              v                    GC Hotness
      Record Access Time                  |
        in Lock Word                      v
              |                    Region Metadata
              v                           |
        GC Trigger                        |
              |                           |
              v                           |
    Recognize App Hotness                 |
              |                           |
              v                           |
      GC Copying Phase                    |
              |                           |
              v                           |
 Group Objects by Hotness                 |
              |                           |
              v                           |
  Homogeneous-Hotness Pages               |
              |                           |
              |                   Memory Pressure
              |                           |
              |                           v
              |                        kswapd
              |                           |
              |                           v
              |                 isolate_lru_pages()
              |                           |
              |                    runtime guidance
              |                           |
              |                           v
              |                 GC-Cold Pages First
              |                           |
              +-------------+-------------+
                            |
                            v
                    Fewer Hot-Page Evictions
                            |
                            v
                       Less Swap-In
                            |
                            v
                       Fewer Janks
```

---

# 一句话理解 Silk

> **Silk 的本质是把 ART 中“对象级的真实访问语义”暴露给 page-level memory management：MO-App 通过 JIT 采样对象访问并借助 GC copying 将相似热度对象聚集到相同 page/region，使 page hotness 更接近 object hotness；MO-GC 则根据前后台状态、Region 新旧和 GC 历史推断 GC working-set hotness，并指导 Linux 优先回收 GC-cold pages，从而减少 hot object 被错误 swap out 后产生的 swap-in 和 UI jank。** 