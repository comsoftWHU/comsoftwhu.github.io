---
layout: default
title: PMR-Fast Application Response via Parallel Memory Reclaim on Mobile Devices
nav_order: 3
parent: 论文
author:  Anonymous Committer
---

# PMR

> PMR: Fast Application Response via Parallel Memory Reclaim on Mobile Devices
>
> USENIX ATC 2025

PMR 主要针对 Android Kernel Memory Reclaim 吞吐量低、延迟高的问题。

传统 Linux 内存回收将 Page Shrinking（筛选并隔离候选页） 和 Page Writeback（检查、解除映射及写回页面） 串行执行，导致回收过程无法充分利用现代 UFS 存储设备的性能。当内存回收跟不上应用的分配速度时，会进一步触发 Direct Reclaim，甚至 LMKD 杀死后台进程，最终增加应用的响应和重新启动延迟。

论文将瓶颈归纳为三个方面：

1. Sequential Execution：Page Shrinking 和 Page Writeback 串行执行，后者必须等待前者准备候选页面。
2. Inefficient Page Shrinking：每轮扫描无法保证找到足够的可回收页面，导致反复扫描 LRU。
3. Inefficient Page Writeback：逐页 Unmap 和小粒度 Write I/O 引入较大开销，无法充分利用 Flash Storage 的并行能力。

论文测得 Page Shrinking 占整个回收路径耗时的 54.8%（4.48 ms），Page Writeback 占 45.2%（3.66 ms）。因此，瓶颈并不只是存储 I/O 本身。





PMR 因此设计两个互补模块：

```
PPS (Proactive Page Shrinking)
    提前准备可供回收的候选页面
    将 Page Shrinking 与 Page Writeback 解耦

SPW (Storage-friendly Page Writeback)
    将 Page Unmap 与 Page Writeback 批量化
    提高存储设备的写入效率
```

## Architecture

```
+------------------------------------------------------------------+
|                       Linux Kernel                               |
|                                                                  |
|  +----------------- Proactive Page Shrinking ------------------+  |
|  |                                                            |  |
|  |  [System LRU Lists]                                        |  |
|  |          |                                                 |  |
|  |          v                                                 |  |
|  |      [kshrinkd]  (Independent Kernel Thread)               |  |
|  |          |                                                 |  |
|  |          | Proactive Page Isolation                         |  |
|  |          v                                                 |  |
|  |   [Victim Page List / LRU_VICTIM]                          |  |
|  |          |                                                 |  |
|  |          | Maintain sufficient victim pages                |  |
|  +----------|-------------------------------------------------+  |
|             |                                                    |
|             | Prepared Victim Pages                              |
|             v                                                    |
|  +----------------- Memory Reclaim ----------------------------+  |
|  |                                                            |  |
|  |  [kswapd / Direct Reclaim]                                  |  |
|  |          |                                                 |  |
|  |          v                                                 |  |
|  |       Page Check                                           |  |
|  |          |                                                 |  |
|  |          v                                                 |  |
|  |  [Application-aware Page Unmap]                            |  |
|  |          |                                                 |  |
|  |          v                                                 |  |
|  |  [Batch Write I/Os]                                        |  |
|  |                                                            |  |
|  +----------|-------------------------------------------------+  |
+-------------|----------------------------------------------------+
              |
              v
+------------------------------------------------------------------+
|                 Flash-based Storage (UFS)                        |
|                                                                  |
|       [Swap Partition]           [Data Partition]               |
+------------------------------------------------------------------+
```

其中：

- kshrinkd：PMR 新引入的独立 Kernel Thread，负责提前扫描 LRU 并准备 Victim Pages。
- Victim Page List：连接 Page Shrinking 和 Page Writeback 的中间队列，用于缓存已经隔离但尚未真正回收的候选页面。
- kswapd / Direct Reclaim：保持原有内存回收触发机制，但可以直接消费预先准备好的页面。
- Application-aware Page Unmap：按照页面关联的应用/进程组织批量 Unmap。
- Batch Write I/Os：将页面组织成适合底层存储设备的较大写入请求。

PMR 的关键不是启动更多 kswapd，而是把原来串行的内存回收路径拆成可以并行工作的生产和消费阶段。&#x20;





原生 Android：

```
Memory Pressure
      |
      v
Page Shrinking
      |
      | wait
      v
Page Writeback
      |
      v
Memory Reclaimed
```

PMR：

```
                kshrinkd
                   |
                   v
             Page Shrinking
                   |
                   v
             Victim Page List
                   |
          +--------+--------+
          |                 |
          v                 v
   Reclaim Request     Replenish Pages
          |                 |
          v                 |
     Page Writeback         |
          |                 |
          v                 v
   Memory Reclaimed    Page Shrinking
```

注意：PPS 提前做的是候选页筛选与隔离，不是提前执行 Swap-out 或释放物理内存。 真正的 Page Unmap / Page-out 仍然在正常内存回收被触发后执行。这也是它与提前换出（Ahead-of-Swap）技术的区别。





## PPS（Proactive Page Shrinking）

PPS 的目标是：让 Page Shrinking 与 Page Writeback 脱离串行依赖，提前维护足够多的 Victim Pages，避免真正发生 Memory Reclaim 时仍需要临时扫描 LRU。

### 1. Independent kshrinkd Thread

原生 Linux 中，`kswapd` 或执行 Direct Reclaim 的线程通常需要先完成 Page Shrinking：

```
kswapd / Direct Reclaim
        |
        v
  scan LRU Lists
        |
        v
  isolate_lru_pages()
        |
        v
   Temporary Page List
        |
        v
  Page Writeback
```

PMR 新增一个独立的 Kernel Thread：

```
kshrinkd
```

由其专门执行 Page Shrinking：

```
kshrinkd
    |
    v
scan LRU Lists
    |
    v
isolate candidate pages
    |
    v
Victim Page List
```

`kswapd` 和 Direct Reclaim 则主要消费已经准备好的 Victim Pages，继续执行实际内存回收。

具体实现中：

1. 在 `start_kernel()` 初始化路径中，通过 `kshrinkd_init()` 初始化。
2. 使用 `kthread_run()` 为每个 Memory Node 创建相应的 `kshrinkd`。
3. `kshrinkd` 复用原生内核的 Page Shrinking 算法，包括对各 LRU List 的扫描比例计算。
4. 但它的唤醒条件不再直接依赖全局低内存水位，而是候选页面队列是否需要补充。

因此：

```
Before PMR:
Memory Pressure -> Shrinking -> Writeback

After PMR:
kshrinkd -> Prepare Victim Pages
                  |
                  v
Memory Pressure -> Writeback
```

两个阶段可以在不同时间提前执行，也可以在系统回收内存时并行执行。





### 2. Victim Page List

为了连接 Page Shrinking 和 Page Writeback，PMR 引入新的内核页面链表：

```
LRU_VICTIM
```

整体页面流转如下：

```
+------------------------+
| System LRU Lists       |
|                        |
| Active / Inactive      |
| Anonymous / File       |
+-----------+------------+
            |
            v
       [kshrinkd]
            |
            | select eligible pages
            v
+------------------------+
| Victim Page List       |
|                        |
| LRU_VICTIM             |
|                        |
| Isolated candidates    |
+-----------+------------+
            |
            v
       Page Writeback
            |
            v
       Memory Freed
```

`kshrinkd` 从原有 LRU 链表中挑选符合条件的页面，例如：

- 未被锁定的页面；
- 没有被频繁引用的页面；
- 满足原有回收条件的候选页面。

论文将已隔离页面标记为 `PG_ISOLATED`，描述为使用页表项的保留位记录该状态，并通过轻量级 Spinlock 协调 Victim Page List 的并发访问。





这里需要注意：

> Victim Page List 存放的是已经从原 LRU 中隔离出来的候选页，并不意味着这些页面已经被 Unmap、Swap-out 或释放。

页面的可回收性在后续 Writeback 阶段仍然需要检查，因此不能直接把 `PG_ISOLATED` 理解为物理内存已经回收。

### 3. Always-Ready Page Shrinking

PPS 的核心问题是：

> 如何保证 Victim Page List 中始终存在足够多的候选页面？

PMR 为此设置一个目标值：

\\[ \delta = nr\\\_ideal\\\_victim\\\_page \\]

表示 Victim Page List 希望维护的页面数量。

当前候选页数量为：

\\[ V=nr\\\_victim\\\_page \\]

当：

\\[ V<\delta \\]

说明候选页面不足，需要通过 `kshrinkd` 补充。

论文给出的默认配置为：

\\[ \boxed{\delta=462\text{ MB}} \\]

约等于原生配置下一轮 Memory Swapping 目标规模（154 MB）的三倍。





#### 系统启动

```
System Start
      |
      v
Create kshrinkd
      |
      v
Victim Page List = Empty
      |
      v
kshrinkd scans LRU
      |
      v
Collect Victim Pages
      |
      v
Victim Pages >= δ ?
      |
   +--+--+
   |     |
   No   Yes
   |     |
   v     v
Continue Sleep
Scanning
```

即：

```
Boot
  ↓
Fill Victim Page List
  ↓
Reach δ
  ↓
kshrinkd Sleep
```

#### 发生 Memory Reclaim

当 Memory Pressure 触发 `kswapd` 或 Direct Reclaim 时：

```
Memory Reclaim
       |
       +----------------------------+
       |                            |
       v                            v
Consume Victim Pages          Wake kshrinkd
       |                            |
       v                            v
Page Writeback                Replenish Pages
       |                            |
       v                            |
Reclaim Memory                       |
                                    v
                           Maintain Victim List
```

这意味着两个阶段可以并行：

```
Time ---------------------------------------->

kshrinkd:
[Prepare] [Sleep] [Prepare] [Prepare] [Sleep]
                     |          |
                     v          v
Writeback:
          [Writeback] [Writeback] [Writeback]
```

每次补充所采用的 Shrinking Batch Size 是：

```
shrink_size = nr_to_reclaim
```

对于论文的默认配置，一轮 Memory Swapping 的目标约为 154 MB，即 \\(\delta/3\\)。`kshrinkd` 持续扫描，直到 Victim Page List 恢复到目标数量。





### 4. 为什么不能无限增加 Victim Page List？

直觉上：

```
More Victim Pages
       ↓
Less Shrinking Wait
       ↓
Faster Reclaim
```

但是维护过大的 Victim Page List 也有问题：

1. 提前隔离的页面可能再次被应用访问。
2. 页面可能需要在 Victim Page List 与普通 LRU 之间转移。
3. 维护、锁竞争及页面状态检查会产生额外开销。

所以：

\\[ \delta \uparrow \\]

不一定意味着：

\\[ Performance \uparrow \\]

论文的敏感性分析表明，随着 \\(\delta\\) 增大，应用响应时间先降低后升高，因而最终选择了约 462 MB 的配置，而不是越大越好。





## SPW（Storage-friendly Page Writeback）

SPW 的目标是：减少逐页 Unmap 的开销，并把分散的小粒度写入变成更适合 Flash Storage 的批量写入。

PPS 解决的是：

```
How to prepare victim pages efficiently?
```

SPW 解决的是：

```
How to reclaim/write back victim pages efficiently?
```

### 1. 原生 Page Writeback 的问题

传统内核 Page Writeback 大致分为：

```
Victim Page
      |
      v
Page Check
      |
      v
Add to Swap Cache
(for anonymous pages)
      |
      v
Page Unmap
      |
      v
Page Out / Free
```

其中：

- Page Check：检查页的引用、锁定状态等，判断是否能够继续回收。
- Add to Swap Cache：对于需要换出的匿名页，准备相关 Swap 状态。
- Page Unmap：通过 Reverse Mapping 等机制修改对应进程的页表映射。
- Page Out / Free：需要写回的页面提交存储 I/O，能够直接丢弃的页面则释放。

原生路径往往逐页处理：

```
Page 1: Unmap -> Write
Page 2: Unmap -> Write
Page 3: Unmap -> Write
Page 4: Unmap -> Write
```

存在两个问题：

1. `Page Unmap` 本身开销不稳定，尤其涉及共享页面的 Reverse Mapping。
2. 每次提交较小的 I/O，难以发挥 Flash Storage 的内部并行能力。

论文的 Figure 7 表明，Page Unmap 的开销甚至明显高于某些 Page-out 提交操作。





### 2. Application-aware Page Unmap

SPW 首先将页面 Unmap 操作按照其关联的 Application / Process 进行组织。

原生模式：

```
Page Check
    |
    v
Unmap Page A
    |
    v
Write Page A
    |
    v
Unmap Page B
    |
    v
Write Page B
    |
    v
Unmap Page C
    |
    v
Write Page C
```

SPW：

```
Page Check
    |
    v
Group Pages by Application
    |
    v
Application-aware Page Unmap
    |
    +--> Unmap A
    +--> Unmap B
    +--> Unmap C
    |
    v
Collect Unmapped Pages
    |
    v
Batch Writeback
```

其核心是：

```
Original:
U1 -> W1 -> U2 -> W2 -> U3 -> W3

SPW:
(U1 + U2 + U3) -> (W1 + W2 + W3)
```

其中：

- \\(U_i\\)：第 \\(i\\) 个 Page Unmap；
- \\(W_i\\)：对应的写入操作。

这也是论文 Figure 9 最直观表达的设计。





需要注意，这里的 Application-aware 并不是指分析应用内部的 Java Object Hotness，而是按照页面关联的进程组织 Unmap 工作。

它并不像 Silk 一样依赖 ART 的 GC 或对象信息。

### 3. 利用 big.LITTLE 提高 Unmap 效率

论文进一步观察到，Page Unmap 可能受到线程抢占或其他内核活动干扰。

因此 SPW 采用两种方法：

1. 提高 Page Unmap Thread 的执行优先级。
2. 将该线程关联到性能更强的 big CPU Core。

这样可以减少批量 Unmap 过程中被抢占的概率，降低 Unmap 执行时间的不确定性。





可以理解成：

```
Application-aware Unmap
          |
          +--> Higher Priority
          |
          +--> Big CPU Core
          |
          v
More Stable Batch Unmap
```

### 4. Batch Write I/Os

SPW 不再局限于单个 4 KB page 的写入提交，而是把多个已解除映射的页面组织成较大的 I/O 请求。

```
Before SPW:

4 KB -> Storage
4 KB -> Storage
4 KB -> Storage
4 KB -> Storage
...
```

```
After SPW:

Pages
  |
  v
Batch
  |
  v
Larger I/O Requests
  |
  v
Block Layer
  |
  v
Flash Storage
```

这是因为 Flash Storage 的内部具有一定并行能力，较大的请求可以提高吞吐量。

但 I/O Size 同样不是越大越好：

- 太小：无法充分利用存储设备带宽。
- 太大：批量准备和等待的时间增加，可能恶化延迟。

因此 PMR 引入设备相关参数：

```
mem_unmap_unit
```

并允许通过 `/proc` 接口动态调整。

论文实验得到：

| 设备                          | 合适的批量 Unmap / I/O 粒度 |
| ----------------------------- | --------------------------- |
| Google Pixel 5（UFS 2.1）     | 1 MB                        |
| Google Pixel 6 Pro（UFS 3.1） | 10 MB                       |

Pixel 6 Pro 在论文的存储测试中，10 MB I/O Size 对应约 1261 MB/s 的写吞吐量。这里是底层存储 I/O 测试结果，并不等同于 PMR 的实际 Memory Reclaim 吞吐量。







## 模型与控制参数

PMR 不像 HMS 那样引入一个预测内存压力的 \\(H_u\\) 模型。

它的核心是一个候选页生产—消费模型，以及一个面向存储设备的批量写回策略。

### 1. Victim Page Inventory

设：

\\[ V(t)=nr\\\_victim\\\_page \\]

表示当前 Victim Page List 中已经准备好的候选页数量。

目标库存为：

\\[ \delta=nr\\\_ideal\\\_victim\\\_page \\]

其控制逻辑可以简化成：

```
V < δ
  |
  v
Wake kshrinkd
  |
  v
Produce Victim Pages
  |
  v
V reaches δ
  |
  v
Sleep
```

当 Page Writeback 消费页面时：

```
V decreases
     |
     v
kshrinkd replenishes
```

需要注意：这里的 \\(V(t)\\) 是对论文中计数器的符号化表达，并非论文另外定义的预测公式。

### 2. Page Shrinking Batch Size

在补充候选页时：

\\[ S=nr\\\_to\\\_reclaim \\]

论文默认的关系近似为：

\\[ \delta=3S \\]

即：

```
Target Victim Inventory: 462 MB

One Shrinking Batch:     154 MB
```

### 3. Writeback Batch Size

令：

\\[ B=mem\\\_unmap\\\_unit \\]

其大小取决于底层存储设备。

```
Small B
   |
   v
Small I/O Requests
   |
   v
Poor Flash Parallelism
```

```
Excessively Large B
   |
   v
Longer Batch Processing / Waiting
   |
   v
Potential Latency Increase
```

因此：

```
Choose device-specific B
        |
        v
Balance Write Throughput and Latency
```

### 4. 两种不同层面的并行

PMR 实际上同时利用了：

Pipeline Parallelism

```
kshrinkd        -> Page Shrinking
kswapd/reclaim  -> Page Writeback
```

使候选页准备与实际回收重叠。

I/O Parallelism

```
Multiple Pages
      |
      v
Batch Write Requests
      |
      v
Flash Internal Parallelism
```

因此 PMR 的名字虽然是 Parallel Memory Reclaim，但不是简单增加多个 `kswapd` 来并行扫描。

## 运行时控制流

PMR 的完整执行流程可以分为六个阶段。

### 1. System Initialization

系统启动期间：

```
start_kernel()
      |
      v
kshrinkd_init()
      |
      v
kthread_run()
      |
      v
Create kshrinkd
      |
      v
Initialize LRU_VICTIM
```

为后续提前准备候选页提供独立的执行线程。

### 2. Proactive Page Shrinking

```
kshrinkd
    |
    v
Check Victim Page Count
    |
    v
V < δ ?
    |
    +--- Yes ---> Scan System LRU
    |                   |
    |                   v
    |             Isolate Pages
    |                   |
    |                   v
    |             LRU_VICTIM
    |                   |
    |                   v
    |             Continue until δ
    |
    +--- No ----> Sleep
```

此阶段不需要等待真正的低内存事件。

### 3. Memory Pressure Detection

系统运行期间：

```
Application Allocations
         |
         v
Available Memory Drops
         |
         v
Kernel Watermark / Allocation Failure
         |
         +-------------------+
         |                   |
         v                   v
       kswapd          Direct Reclaim
```

这里 PMR 没有修改原生的 Memory Reclaim 触发条件。

也没有改变原有 LRU Victim Selection 的基本策略。论文在 Section 4.4 明确将这些机制视为与 PMR 正交的优化方向。





### 4. Consume Prepared Victim Pages

```
kswapd / Direct Reclaim
          |
          v
Read LRU_VICTIM
          |
          v
Get Prepared Pages
          |
          +----------------------+
          |                      |
          v                      v
     Page Check            Wake kshrinkd
          |                      |
          v                      v
       SPW Path            Replenish List
```

这里 PPS 的好处体现出来了：Writeback 不必在每次回收时都从头等待一次完整的 LRU 扫描。

### 5. Storage-friendly Page Writeback

```
Victim Pages
      |
      v
Page Check
      |
      v
Swap Cache Preparation
(if needed)
      |
      v
Application-aware Page Unmap
      |
      v
Batch Page-out
      |
      v
UFS Storage
      |
      v
Complete Reclaim
```

匿名页可以换出至 Swap Partition，需要写回的脏文件页写回文件系统；可直接丢弃的干净文件页不需要额外的 Flash 写入。

需要强调，论文的实验配置是 Flash-based Swapping，而非把 zRAM 压缩写入作为 SPW 的主要优化对象；实验设备统一启用了 2 GB 的 Flash Swap Partition。





### 6. Replenishment

一轮 Page Writeback 消费完候选页面后：

```
Victim Page List Decreases
           |
           v
kshrinkd Continues Shrinking
           |
           v
Restore Inventory to δ
           |
           v
Ready for Next Reclaim
```

从而形成：

```
Page Production
       |
       v
Victim Buffer
       |
       v
Page Consumption
       |
       +----> Replenish Page Production
```

## 完整控制流

```
                            PMR
                             |
               +-------------+-------------+
               |                           |
               v                           v
              PPS                         SPW
               |                           |
               v                           |
         System Startup                    |
               |                           |
               v                           |
          kshrinkd                         |
               |                           |
               v                           |
       Scan System LRU                     |
               |                           |
               v                           |
      Isolate Victim Pages                 |
               |                           |
               v                           |
       LRU_VICTIM List                     |
               |                           |
         Maintain δ Pages                  |
               |                           |
               |                     Memory Pressure
               |                           |
               |                           v
               |                  kswapd / Direct Reclaim
               |                           |
               +---------------------------+
               |                           |
               |                           v
               |                    Consume Victims
               |                           |
               |                           v
               |                       Page Check
               |                           |
               |                           v
               |                  Application-aware Unmap
               |                           |
               |                           v
               |                     Batch Write I/Os
               |                           |
               |                           v
               |                      UFS Storage
               |                           |
               |                           v
               |                     Memory Reclaimed
               |
               v
       Replenish Victim Pages
               |
               v
        Ready for Next Reclaim
```

从整体看，PMR 通过两种机制加速回收：

```
PPS:
Hide Page Shrinking Latency

SPW:
Reduce Page Unmap / Writeback Overhead
```

最终：

```
Faster Memory Reclaim
          |
          v
Less Direct Reclaim
          |
          v
Fewer LMKD Kills
          |
          v
Faster Application Response
```

## 实验结果

论文在三款 Android 13、Linux 5.10 手机上进行了实验：

| 设备               | 内存  | 存储    |
| ------------------ | ----- | ------- |
| Google Pixel 5     | 8 GB  | UFS 2.1 |
| Redmi Note 11      | 6 GB  | UFS 2.2 |
| Google Pixel 6 Pro | 12 GB | UFS 3.1 |

实验采用 36 个应用组成的多任务工作负载，重点测量应用切换时的响应时间，以及 Kernel Reclaim 的吞吐量和 LMKD 行为。

### 主要性能提升

| 指标                      | PMR 报告的结果                   |
| ------------------------- | -------------------------------- |
| Application Response Time | 相比原生系统最多降低 43.6%       |
| Peak Reclaim Throughput   | 相比原生系统提升 82.8%           |
| LMKD Kill Count           | 相比原生系统减少 82%             |
| Direct Reclaim Count      | 相比 Acclaim 减少 45%            |
| PMR + Fleet               | 相比原生系统，响应时间降低 67.4% |
| CPU Overhead              | 相比原生回收路径约增加 5.3%      |
| Flash Write Volume        | 增加约 12.1%                     |

以上是论文在相应实验和比较基线下报告的结果，并不是对所有负载的普遍性能保证。

其中 CPU 与 Flash Write Overhead 很重要：

- CPU 开销：独立 `kshrinkd`、Victim Page List 同步以及批量 Unmap 调度增加了处理成本。
- Flash 写入量：PMR 因为能够更有效地执行 Swap-out，也产生了更多 Flash 写入；这是一个明确的效率与写放大/存储寿命权衡。

此外，论文特别指出：由于 PMR 没有改变 Victim Page 的基本选择策略，因此它的 Page Fault Rate 与原生回收方案基本相同。这说明 PMR 的主要收益来自更高的回收执行效率，而不是更准确的热页识别。



## 一句话理解 PMR

> PMR 的本质是将 Linux 原本串行执行的 Page Shrinking 和 Page Writeback 改造成基于独立 `kshrinkd` 与 Victim Page List 的生产者—消费者流水线，通过提前隔离候选页隐藏扫描延迟，再利用 Application-aware Batch Unmap 和 Storage-friendly Batch I/O 加速实际回收，在不改变原有内存压力触发机制和基本 LRU 选择策略的情况下，提高 Kernel Reclaim 吞吐量，减少 Direct Reclaim 与 LMKD 引起的应用响应延迟。