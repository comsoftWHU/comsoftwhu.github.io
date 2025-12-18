---
layout: default
title: Intrinsic编译
nav_order: 10
parent: dex2oat
author: Anonymous Committer
---


## TryCompile vs TryCompileIntrinsic：流程设计差在哪？

### TryCompile：**“从字节码出发”的常规编译**

`TryCompile()`拿到的是 `DexCompilationUnit` 里的 `code_item`（也就是方法字节码），然后走标准 optimizing pipeline：

* 建 `HGraph`（图里会记录 `compilation_kind`）
* `HGraphBuilder::BuildGraph()` 解析字节码把 IR 搭出来
* 跑一串优化 pass（常量折叠、inlining、GVN、BCE、LICM、LSE…一直到 codegen 前清理）
* **如果是 baseline 编译**，还会在某些条件下创建 `ProfilingInfo`（为后续更激进的优化收集 profile）

**TL;DR**: **TryCompile 是“真编译 bytecode”，并且 baseline或optimized 的差异主要体现在 compilation kind + 各 pass 内部策略 + 是否构建 profiling info。** 

---

### TryCompileIntrinsic：**“不看字节码，只编一个 intrinsic 图”**

`TryCompileIntrinsic()` 的核心是：

* 直接 new `HGraph`，并且用 **“空 code item”** 的 `CodeItemDebugInfoAccessor()`（注释里就写 Null code item）
* 调 `builder.BuildIntrinsicGraph(method)`：**用 method 上的 intrinsic 标记，搭一个“纯 intrinsic 的 IR 图”**
* 只跑极少量必要优化（“codegen 有些假设必须靠 instruction simplifier 满足”，所以至少跑 `kInstructionSimplifier`），再跑架构相关优化、寄存器分配、codegen
* 还会要求 **必须是 leaf method**，否则直接放弃这个路径
* 关键：函数里有 `DCHECK(Runtime::Current()->IsAotCompiler())`，也就是**它本质上是为 AOT（尤其 boot image）准备的**

**TL;DR**: **TryCompileIntrinsic 是“把这个方法当作一个 intrinsic 模板来编”，绕开字节码解析与大部分优化，只生成一个可直接落地的极简机器码实现。** 

---

## 那现在 JIT 会走 TryCompileIntrinsic 吗？

**一般不会。**在 `OptimizingCompiler::JitCompile()` 的“普通 Java 方法（非 native）”路径里，它就是直接 `TryCompile(..., compilation_kind, ...)`，然后 build stackmaps、commit 到 code cache。这里没有 `TryCompileIntrinsic` 的分支。

---

## 但是：JIT 仍然会“用到 intrinsics”，只是方式不同

这里有个特别容易混淆的点：

* **TryCompileIntrinsic**：编译“这个方法本身就是 intrinsic 方法”，用 `BuildIntrinsicGraph()` 直接搭图（主要 AOT 用）。
* **JIT和AOT中 TryCompile 里的 intrinsics**：编译别的方法时，IR 里出现某些 `invoke`，会被 intrinsic 机制“识别并内建化”，最后 codegen 直接出更快的指令序列。

`compiler/optimizing/intrinsics.cc` 这套就是干这个的：`IntrinsicVisitor` 会给 `HInvoke` 创建 `LocationSummary(..., kIntrinsified)`，并按不同 intrinsic 做专门处理。

* **JIT 不走 TryCompileIntrinsic**

* **JIT 仍会大量“intrinsify”**，但那是在 TryCompile 的 IR 优化阶段完成的。

