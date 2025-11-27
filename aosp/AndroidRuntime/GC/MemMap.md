---
layout: default
title: MemMap数据结构
nav_order: 6
parent: GC算法和实现
author: Anonymous Committer
---

## TL;DR

> 统一封装 mmap 与 munmap 等平台接口，管理 ART 进程中的映射区域与元数据；在 64 位但不支持 MAP_32BIT 的平台上，提供低 4GB 线性扫描分配器，满足 JIT 与指针压缩等对 32bit 可寻址区域的要求。

## `mmap` 基础速览

* 文件映射：将文件页映射到进程虚拟地址空间。
* 匿名映射：不关联文件（如大块 malloc 背后常用）。
* 访问：首次触发缺页，内核按需填充页表。
* 写回：MAP_SHARED 可写回文件；MAP_PRIVATE 写时复制 COW。
* 解除与同步：munmap 解除映射；msync 同步页缓存到后端。
* 安全页判定（Linux）：对未映射页调用 msync 会立刻返回 ENOMEM，可用来判定一段地址确实空白。低 4GB 分配器在 Linux 上用此技巧确认可用区间。

## AOSP 的 MemMap 封装层

```cpp
// 统一出口（Unix Linux Fuchsia Windows 各有实现）
static void* TargetMMap(void* start, size_t len, int prot, int flags, int fd, off_t off);
static int   TargetMUnmap(void* start, size_t len);
````

* Unix / Linux：直接转调 mmap 与 munmap。
* Windows：翻译为 CreateFileMapping 与 MapViewOfFile，限制较多（不支持 MAP_FIXED，PROT_EXEC 组合受限）。
* Fuchsia：使用 VMAR/VMO 机制封装底层映射，通常会在低地址段预留“低内存区域”，在其上做匿名映射；文件映射仍经由统一的 TargetMMap 接口。

**适配图：**

```mermaid
flowchart LR
  A[MemMap API] --> B[Unix Linux 适配 mmap 与 munmap]
  A --> C[Windows 适配 CreateFileMapping 与 MapViewOfFile]
  A --> D[Fuchsia 适配 低地址 VMAR/VMO 与 mmap]
```

---

## 平台差异与宏

* USE_ART_LOW_4G_ALLOCATOR

  * 在 LP64 且非 Fuchsia 的 aarch64 / riscv / Apple 平台启用低 4GB 线性扫描分配器。
  * x86_64 上可以直接依赖内核 MAP_32BIT 支持低地址映射。

* kMadviseZeroes 与 HAVE_MREMAP_SYSCALL（Linux）

  * kMadviseZeroes 为真时，madvise 的 DONTNEED（等）可以在逻辑内容不变的前提下，让内核丢弃驻留页、下次访问再按需拉回。
  * HAVE_MREMAP_SYSCALL 为真时，mremap 支持做原子替换映射（用于 ReplaceWith）。

---

## 成员布局与生命周期

```cpp
  std::string name_;
  uint8_t* begin_ = nullptr;   // 用户可用起点 可能不是页对齐
  size_t size_ = 0;            // 用户可用长度

  void*  base_begin_ = nullptr;// 页对齐起点
  size_t base_size_  = 0;      // 页对齐长度
  int    prot_ = 0;
  bool   reuse_ = false;       // 仅视图 不负责 unmap
  bool   already_unmapped_ = false;
  size_t redzone_size_ = 0;    // Sanitizer redzone
  ...
```

* IsValid 近似等价于 base_size_ 不为零。
* 析构与 Reset 负责 munmap（除非 reuse 视图或 already_unmapped）。
* 对齐与切分：有 AlignBy、RemapAtEnd、TakeReservedMemory 等工具方法。

---

## 匿名映射 MapAnonymous

> 这是 ART 中最常走的入口，兼顾提示地址、低 4GB 与预留或复用的多分支。

**关键点：**

* 复用 / 预留

  * 复用已有 MemMap 区间时，将标志加入 MAP_FIXED，让新映射强制覆盖在原地址上。
  * 使用预留（reservation）时，同样加入 MAP_FIXED，并在成功后把预留对象释放、移交所有权。

* 匿名映射

  * 默认使用 MAP_PRIVATE | MAP_ANONYMOUS。
  * 若调用方设置了 MAP_FIXED，则完全按调用方要求执行，失败直接返回。

* 低 4GB

  * 若启用线性扫描分配器且未指定地址（`USE_ART_LOW_4G_ALLOCATOR && low_4gb && addr == nullptr`），先以去掉执行位的 prot 通过 MapInternalArtLow4GBAllocator 探测映射，成功后再 mprotect 加回执行位。
  * 否则在 x86_64 上，若需要 low_4gb 且未指定地址，则给 flags 加上 MAP_32BIT，交给内核选择低 2GB/4GB 范围的地址。

**流程图：**

```mermaid
flowchart TD
  A[MapAnonymous 入口] --> B[校验 size 大于零 并向页对齐]
  B --> C[是否复用或使用预留]
  C -- 复用 --> C1[检查在既有映射内 标志加入 MAP_FIXED]
  C -- 预留 --> C2[检查预留覆盖 标志加入 MAP_FIXED]
  C -- 否 --> C3[标志使用 MAP_PRIVATE 和 MAP_ANONYMOUS]

  C3 --> D[进入 MapInternal<br/>根据 low_4gb / 平台选择 mmap 路径]
  C1 --> D
  C2 --> D

  D --> F[获得实际地址 actual]
  F --> G[actual 是否失败]
  G -- 是 --> G1[记录错误并返回 Invalid]
  G -- 否 --> H[提示地址 addr 非空 且 非 MAP_FIXED]
  H -- 是 --> H1[检查 actual!=addr 则 munmap 并报错]
  H -- 否 --> I[设备端设置匿名 VMA 名称]
  H1 -->|失败| G1
  I --> J[是否来自预留]
  J -- 是 --> J1[释放预留并移交所有权]
  J -- 否 --> K[返回 MemMap]
```

### MapInternal 内部逻辑（精简视图）

```mermaid
flowchart TD
    S1a[LP64 且 low4gb 且 指定地址超出低4GB范围]
    S1a -- 是 --> S1err[直接返回失败]
    S1a -- 否 --> S1b[USE_ART_LOW_4G_ALLOCATOR 且 未指定地址 且 low4gb]
    S1b -- 是 --> S1c[临时去除执行位 使用低4GB线性扫描]
    S1c -->|失败| S1fail[失败返回]
    S1c -->|成功| S1d[是否需要执行位]
    S1d -- 是 --> S1e[mprotect 恢复执行位]
    S1d -- 否 --> S1ok[返回地址]
    S1b -- 否 --> S1f[x86_64 支持 MAP_32BIT 且 low4gb 且 addr 为 nullptr]
    S1f -- 是 --> S1g[flags 加 MAP_32BIT 后调用 mmap]
    S1f -- 否 --> S1h[直接调用 mmap 使用 flags]
```

---

## 低 4GB 线性扫描分配器

### 从 low_4gb 请求到分配器入口

> 当调用方传入 `low_4gb = true` 时，整体从 API 到低 4GB 扫描分配器的大致路径如下（仅 LP64 场景）：

```mermaid
flowchart TD
    Call[调用 MemMap::MapAnonymous / MapFileAtAddress<br/>参数 low_4gb = true] --> MInternal[进入 MemMap::MapInternal]

    MInternal --> C1{是否 LP64}
    C1 -- 否 --> Path32[32 位进程 本身地址空间<br/>已经在 4GB 内 直接 mmap 返回]
    C1 -- 是 --> CCheck[如果 addr 非空<br/>检查 addr 和 addr+length 是否都在低 4GB]
    CCheck --> CCheckFail{检查失败?}
    CCheckFail -- 是 --> FailRange[打印错误<br/>返回 MAP_FAILED]
    CCheckFail -- 否 --> C2{USE_ART_LOW_4G_ALLOCATOR 为 1?}

    C2 -- 否 --> X86Path[x86_64 等平台<br/>flags 加 MAP_32BIT（如有）<br/>然后直接 mmap]
    C2 -- 是 --> C3[线性扫描逻辑]
```

### ART_LOW_4G_ALLOCATOR 线性扫描逻辑

```mermaid
flowchart TD
    C2[开始]  -->  C3{addr 是否为 nullptr}

    C3 -- 否 --> DirectMmap["直接 mmap(addr, ...)<br/>caller 自己保证低 4GB"] --> ReturnDirect[返回结果或失败]
    C3 -- 是 --> Low4GBAlloc[进入 ART 低 4GB 分配器<br/>使用 next_mem_pos_ 线性扫描]

    Low4GBAlloc --> Scan1[第一轮 利用 MemMap::maps_<br/>跳过已知映射 找空洞 调用 TryMemMapLow4GB]
    Scan1 --> R1{mmap 成功且在低 4GB?}
    R1 -- 是 --> OK1[更新 next_mem_pos_ = actual + length<br/>返回指针]
    R1 -- 否 --> SpaceCheck{4GB 内剩余空间是否不足 length?}

    FailNoSpace[没有足够连续空间 ENOMEM]

    SpaceCheck -- 是 --> FirstRun{当前是否 first_run?}
    FirstRun -- 否 --> FailNoSpace
    FirstRun -- 是 --> Restart[ptr 从 LOW_MEM_START 重新扫描<br/>second run] --> Scan1

    SpaceCheck -- 否 --> Scan2[第二轮（Linux）: 用 msync 探测真实空洞<br/>找到全 ENOMEM 区域再试 TryMemMapLow4GB]
    Scan2 --> Done2[成功映射则返回 否则继续扫描直到 4GB 边界]
```

> 仅在启用低 4GB 扫描、addr 为空且 low4gb 为真时走。
> 目标是在 4GB 以下找到连续空闲页：使用随机化起点、用 gMaps 跳过已占区间，并在 Linux 上通过对每页 msync 期待 ENOMEM 的方式确认这一段没有其他匿名/文件映射。

---

## 文件映射与对齐

对齐规则

* 起始偏移向下页对齐；
* 实际映射长度加上页内偏移后再向上页对齐；
* 最终 begin 等于 base_begin 加上页内偏移，size 为用户可见长度。

```mermaid
flowchart LR
  A[文件偏移 start] --> B[计算 base_off 向下到页边界]
  B --> C[以 base_off 调用 mmap 获得 base_begin]
  C --> D[begin 等于 base_begin 加上 start 减去 base_off]
  C --> E[size 等于用户长度]
  D --> F[得到用户可见区间]
```

提示地址检查

* 若传入 addr 但未使用 MAP_FIXED，成功后需检查实际 begin 是否等于 addr，否则立刻 munmap 并报错。

---

## 保护、同步与回收

* Protect：调用 mprotect，更新 prot_。
* Sync：调用 msync（如 Linux 上使用 MS_SYNC），用于持久化或失效缓存。
* ZeroMemory / MadviseZero（Linux 上的典型行为）：

  * 对齐到页边界后，对目标区间调用 madvise(..., MADV_DONTNEED)。
  * 内核可以丢弃驻留物理页，逻辑内容仍被视为零页，会在后续访问时按需重新拉入。

```mermaid
flowchart TD
  Z0[MemMap 区间] --> Z1[按页对齐待回收区间]
  Z1 --> Z2["madvise(..., MADV_DONTNEED)"]
  Z2 --> Z3[逻辑视角不变 下次访问触发缺页<br/>物理可被回收与清零]
```

> 某些 GC 空间实现可能在此基础上使用 mincore 聚合驻留页，以便只对“实际驻留的页面”调用 madvise；这一层属于上层策略，MemMap 自身只提供基础接口。

---

## 原子替换映射 ReplaceWith

仅 Linux 支持，依赖 mremap。
前置条件：

* 源与目标不重叠，且都拥有真实映射（非 reuse 视图）。
* redzone、页对齐、页内偏移等参数一致。
* 目标区间足以容纳源区间。

```mermaid
sequenceDiagram
  participant Src as Source MemMap
  participant Dst as Dest MemMap
  participant OS as Kernel mremap

  Src->>OS: mremap(Source, Dst.base_begin, MREMAP_FIXED)
  OS-->>Src: 返回结果
  alt 成功
    Src->>Dst: 转移所有权 Source 失效
  else 失败
    Src->>Src: 保持不变 返回错误
  end
```
