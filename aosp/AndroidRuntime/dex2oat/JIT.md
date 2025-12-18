---
layout: default
title: JIT即时编译
nav_order: 10
parent: dex2oat
author: Anonymous Committer
---

## TL;DR

**JIT 相比纯 AOT / 只解释执行，多做三件事：**

1. 运行时持续 profile（热度、类型、分支等）。
2. 为这些信息提供专用的数据结构（ProfilingInfo、InlineCache、JitCodeCache 等）。
3. 引入一套运行时链路（热度计数 → 编译任务 → 写入 code cache → code cache GC / profile 落盘）。

---

## 一、JIT 额外需要收集的“信息”

这些信息主要挂在 `ProfilingInfo` / `InlineCache` 以及 `JitCodeCache` 周边，AOT 在“离线编译阶段”不会动态更新这些东西。

### 1. 方法热度（Hotness）

* 被 JIT baseline 编译的方法会关联一个热度计数（常见是 `ProfilingInfo` 里维护的计数）。
* 解释器 / stub 会在方法调用、回边等位置累积热度，JIT 根据这些计数决定：
  * 是否触发 baseline 编译；
  * baseline 是否升级为更高质量版本；
  * 是否触发 OSR（On-Stack Replacement）。

> AOT 没有“运行时不断变化”的热度计数；JIT 的策略是边跑边更新。

### 2. 调用点类型信息（Inline Cache）

* 对选中的 `invoke-*` 调用点，维护一个 InlineCache：
  * 记录 receiver 类型 / 目标方法分布（单态、多态、megamorphic）。
* 编译器用这些信息做去虚拟化、内联、类型检查收敛等。

### 3. 分支 / 回边 profile

对特定 dex pc 记录分支倾向（taken / not taken），用于布局与分支相关优化（例如让热路径 fallthrough）。

### 4. OSR 相关信息

OSR 需要：
* 热回边 / dex pc 的触发点；
* 对应位置的 live 值分布（stack map / dex register map）以便把解释器态的 vregs 拼成 compiled frame。

OSR 入口通常由 `JitCodeCache` 持有（方法 → OSR 代码映射）。

### 5. Profile 持久化所需信息

JIT 会把热点方法与 InlineCache 信息周期性落盘（`.prof`），供后续 `dex2oat speed-profile / baseline profile` 使用。

---

## 二、JIT 额外引入的主要数据结构

核心就是：**ProfilingInfo + InlineCache + JitCodeCache + JitMemoryRegion + ProfileSaver**。

### 1. `ProfilingInfo`

典型字段包括：
* `baseline_hotness_count_`：baseline 热度计数；
* `method_`：对应 `ArtMethod*`；
* `number_of_inline_caches_` / `number_of_branch_caches_`；
* `current_inline_uses_`：用于避免清理仍在使用中的 inline cache；
* `InlineCache cache_[0]`：尾随数组保存调用点信息。

### 2. `InlineCache`

每个调用点一个小表，保存若干条 `(Class*, ArtMethod*, count)` 以及状态（空/单态/多态/megamorphic）。同时它与 profile 文件格式需要对齐（源码里会有静态断言保护）。

### 3. `jit::Jit`（主控制器）

挂在 `Runtime` 上，内部关键成员：
* `JitCodeCache* code_cache_`
* `jit_compiler_`（Optimizing 后端接口）
* `thread_pool_`
* `JitOptions`（阈值、容量、开关等）

### 4. `JitCodeCache`

统一承载：
* 机器码（code）
* 附属数据（stack map、roots 表、debug info、profiling info 相关结构）
* 方法 → code header / entry 的映射
* OSR 映射
* 与 GC 的集成（root 扫描、卸载清理）

### 5. `JitMemoryRegion`

一次编译提交使用的“分区”，负责：
* code/data 空间分配
* RW/RX 双映射（避免 RWX）
* cache flush / membarrier 等平台要求

### 6. `ProfileSaver` / `ProfileSaverOptions`

后台线程周期性把热点方法、InlineCache 等信息写入 `.prof`，形成 “JIT → AOT” 的 PGO 输入。

---

## 三、JIT 额外引入的“机制”

### 1. 热度统计与触发

* 解释器 / stub 在方法入口、回边等位置累积 profile。
* 达到阈值后，`Jit` 把编译任务塞进线程池队列，由 worker 执行编译与提交。

### 2. On Stack Replacement（OSR）

#### 2.1 OSR 解决的问题

方法已经在解释器里跑进热循环时，不用等“下次调用”再用 compiled code，而是在循环中间切入 compiled code 继续执行。

#### 2.2 OSR 两条链路

**编译侧**
* 把 OSR 视作单独的编译 kind（`CompilationKind::kOsr`）。
* 编译完成后写入 `osr_code_map_`，供解释器查询。

**执行侧**
1. 解释器执行到回边 / 热 dex pc；
2. 安全检查（debug/instrumentation/栈空间等）；
3. 查 OSR 入口；没有则继续解释；
4. 用 stack map 把 vregs 写入目标 compiled frame；
5. 通过 `art_quick_osr_stub` 切栈并跳转到 OSR entry。

#### 2.3 OSR 流程图

**OSR 编译侧**

```mermaid
flowchart TD
    A[运行时判断方法在热路径/热循环] --> B{是否已有OSR版本?}
    B -- 否 --> C[Jit::AddCompileTask<br/>CompilationKind::kOsr]
    C --> D[JitThreadPool OSR队列/更高优先级]
    D --> E[JIT线程执行OSR编译]
    E --> F[JitCodeCache::Commit]
    F --> G[写入 osr_code_map_<br/>method -> osr_entry]
    B -- 是 --> H[无需再发OSR编译任务]
````

**OSR 执行侧**

```mermaid
flowchart TD
    A[解释器/nterp 执行到loop-back或热点dex_pc] --> B[MaybeDoOnStackReplacement]
    B --> C{安全检查通过?}
    C -- 否 --> Z[继续解释执行]
    C -- 是 --> D["PrepareForOsr(method, dex_pc)"]
    D --> E{是否已有OSR入口?}
    E -- 否 --> Z
    E -- 是 --> F[获取OSR stack map<br/>dex_pc -> native_offset]
    F --> G{stack map有效?}
    G -- 否 --> Z
    G -- 是 --> H[按DexRegisterMap把vregs写入目标frame镜像]
    H --> I[计算 native_pc = entry + native_offset]
    I --> J[art_quick_osr_stub<br/>切栈并跳转]
    J --> K[进入OSR编译代码继续执行]
```

### 3. 在线编译任务调度

* `Jit::MaybeEnqueueCompilation()` 判断是否入队。
* `JitCompileTask` 在线程池里运行，调用 Optimizing 后端编译，然后 `JitCodeCache::Commit` 落入 code cache，并更新 `ArtMethod` entrypoint。

### 4. Code cache 管理与 JIT GC

接近容量上限时触发 code cache GC：

* 回收不再使用/不再热的方法代码与附属数据；
* 清理 OSR 映射；
* 类卸载时同步清理相关条目。

### 5. Profile 持久化与 AOT 交互

`ProfileSaver` 周期性抽取热点与 InlineCache，写入 `.prof`，后续 `dex2oat` 以 speed-profile / baseline profile 消费这些数据。

### 6. 调试与崩溃分析

JIT 代码与调试信息会注册给运行时，tombstone/调试器可在 `memfd:jit-cache` 等映射区间符号化 JIT 栈。

### 7. JIT patches（ARM64）：运行时“回填”与 roots 表

AOT 可以依赖离线 linker（或 oat writer）做 `LinkerPatch` 回填；JIT 代码直接写入 `JitCodeCache`，没有独立的链接阶段，所以需要在 **commit 期间**把“占位 literal”补成最终值。`jit_patches_arm64.h/.cc` 就是 ARM64 下这套补丁逻辑的实现。

#### 7.1 关键场景：String/Class roots patch（让 compiled code 安全引用对象）

JIT 代码经常需要引用 `String*`、`Class*` 等对象：

* 直接把对象地址硬编码进机器码不可靠（对象可能被 GC 移动）。
* ART 的做法：为每次提交建立一个 `GcRoot<Object>[]` roots 表放在 JIT data 区；机器码里只保留“roots 表项地址”，真正对象引用由 roots 表承载并可被 GC 扫描/更新。

按执行顺序拆开：

1. **codegen：登记 root + literal 先占位**

   * 遇到需要的 string/class，登记为 JIT root；
   * literal 池里创建/复用一个初值为 0 的 32-bit literal，记录“这个 dex 引用 → 对应 literal”。

2. **Commit 前：确定 root index**

   * 收集本次编译所有 roots，形成向量；
   * 为每个 string/type 计算其在 roots 表里的 index。

3. **CommitData：写入 roots 表与 stack map**

   * `JitMemoryRegion::CommitData()` 把 `GcRoot[]` 连续写入 data 区；
   * 在 roots 表末尾写入 length，然后紧跟 stack map 数据。

4. **EmitJitRootPatches：回填 literal**

   * 对每个记录的 string/class patch，把 literal 回填为：
     `roots_data + index * sizeof(GcRoot<Object>)`
   * 这样 compiled code 通过 literal 拿到 roots 表项地址，再间接得到对象引用。

#### 7.2 其它补丁/去重：减少 literal 池与复用常量

`jit_patches_arm64` 还提供：

* `uint32/uint64` literal 去重（同值复用）；
* boot image 地址 literal 去重；
* JIT string/class literal 的去重与 patch 记录。

这些主要是为了减少 literal 池膨胀，并让同一份编译产物内部尽量复用常量。

---

## 四、baseline 与 optimized JIT

ART 的 JIT 会区分编译质量（tier），常见是 baseline vs optimized：

* **baseline**：更强调编译速度与低延迟，优化更保守，用于尽快替换解释执行。
* **optimized**：更强调稳态性能，允许更重的优化（例如更激进的 inlining、循环相关优化等）。

### 4.1 编译 kind 的调整（runtime 侧）

`Jit::CompileMethodInternal()` 里有两条典型的 kind 调整逻辑：

* 如果编译器被标记为 baseline compiler，而请求的是 optimized，会改成 baseline。
* 如果请求 baseline，但当前 code cache 不能分配 profiling info，则改成 optimized。

```cpp
if (jit_compiler_->IsBaselineCompiler() && compilation_kind == CompilationKind::kOptimized) {
  compilation_kind = CompilationKind::kBaseline;
}
if ((compilation_kind == CompilationKind::kBaseline) &&
    !GetCodeCache()->CanAllocateProfilingInfo()) {
  compilation_kind = CompilationKind::kOptimized;
}
```

### 4.2 为什么 baseline 分配不到 profiling info 会转 optimized？

baseline 编译通常伴随“可持续 profile/升级”的设计：如果 profiling info 无法分配（例如 region 约束、共享区限制等），baseline 的后续收益（inline cache 等信息积累、为更高 tier 提供输入）会明显缩水。此时直接编译为 optimized 更接近“自洽的一份最终代码”。

### 4.3 为什么需要升级（baseline → optimized）

* **编译延迟 vs 稳态性能**：baseline 先把解释器开销打下来；方法更热后再用 optimized 把稳态性能拉满。
* **profile 积累**：类型信息、分支信息越充分，optimized 越能做有效的去虚拟化与内联。
* **OSR 场景**：热循环中切入 compiled code 后，optimized 通常能放大收益。