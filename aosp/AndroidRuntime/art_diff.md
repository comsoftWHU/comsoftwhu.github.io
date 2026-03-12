---
layout: default
title: AOSP16 ART的变化
nav_order: 13
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---


# ART 代码设计与实现差异总结
对比范围：
- 起点：`795d594fd825385562da6b089ea9b2033f3abf5a`
- 终点：`1690c6912a7972c9e62c39b48c706de9b8b18b4a`

对比对象：
- 目录：`./art`
- 重点：代码设计、实现路径、模块职责、接口形态、关键数据结构和运行时行为的变化

说明：
- 本文基于 `git diff`、`git log`、`git show` 做模块级分析。
- 代码片段默认来自终点 commit `1690c691...`，只有明确注明“旧状态”的片段来自起点 commit `795d594...`。

## 1. 先看整体规模

总体规模：
- `1278 files changed, 85261 insertions(+), 30348 deletions(-)`

主要目录占比：
- `test/`：25.4%
- `runtime/`：23.0%
- `compiler/`：16.6%
- `compiler/optimizing/`：13.4%
- `tools/`：9.2%
- `libartbase/`：5.1%

关键热点文件：
- `compiler/optimizing/fast_compiler_arm64.cc`：`+3675`
- `compiler/optimizing/graph.cc`：`+1232`
- `compiler/optimizing/graph.h`：`+564`
- `compiler/optimizing/register_allocator_linear_scan.cc`：`+1567`
- `compiler/optimizing/ssa_liveness_analysis.h`：`+853 / -...`
- `runtime/gc/collector/mark_compact.cc`：`+3041`
- `runtime/gc/collector/mark_compact.h`：`+564`
- `runtime/class_linker.cc`：`+1512 / -...`
- `runtime/hidden_api.cc`：`+487`
- `artd/artd.cc`：`+405`

这组数字反映出两个事实：
- 这不是“某个 pass 修一修”或者“某个 runtime bugfix 的堆积”，而是一轮架构层面的扩张。
- 新代码主要集中在三条主线：编译器分层、runtime/GC/hiddenapi 语义增强、artd/dexopt 服务化。

## 2. 一句话结论

从 `795d594...` 到 `1690c691...`，ART 的整体形态发生了明显变化：

- 编译层从“解释器/nterp + optimizing compiler”为主，演进为“解释器/nterp + baseline fast compiler + optimizing compiler”的多层结构。
- `optimizing/` 内部从“核心类型集中在几个大文件里”演进为“Graph / InstructionList / LoopInformation / RegisterSet 分离”的更模块化结构。
- runtime 在 GC、virtual thread、hiddenapi、JIT 交互上更系统化，强调完整语义而不是局部修补。
- artd 和 pre-reboot dexopt 从“执行 dexopt 的服务”演进为“带 staged metadata、事务性提交、OTA 感知、产物生命周期管理”的系统服务。

下面按模块展开。

---

## 3. Compiler / Optimizing：最大的设计变化

### 3.1 新增 ARM64 Fast Compiler：ART 出现新的 baseline 编译层

这是这段区间里最重要的编译器变化。起点没有这条路径，终点新增：

- `compiler/optimizing/fast_compiler.h`
- `compiler/optimizing/fast_compiler_arm64.cc`

相关文件的变化量：
- `fast_compiler.h`：`+97`
- `fast_compiler_arm64.cc`：`+3675`

这不是“优化编译器里多了个 helper”，而是新增一条独立编译路径。它的定位是：
- 低编译成本
- one-pass / baseline 风格 codegen
- 尽量少建复杂中间结构
- 复杂 case 直接保守回退

代表性代码：

```cc
/**
 * A lightweight, one-pass compiler. Goes over each dex instruction and emits
 * native code for it.
 */
class FastCompiler {
 public:
  static std::unique_ptr<FastCompiler> Compile(...) {
    ...
    switch (compiler_options.GetInstructionSet()) {
#ifdef ART_ENABLE_CODEGEN_arm64
      case InstructionSet::kArm64:
        return CompileARM64(...);
#endif
      default:
        return nullptr;
    }
  }
};
```

这段代码已经把设计说得很清楚了：
- 这是一个“轻量 one-pass compiler”
- 它不是 generic backend，而是按 ISA 分派
- 当前阶段只真正支持 ARM64

更关键的是它的主循环。终点 commit 里的 `ProcessInstructions()` 基本就是 baseline 编译器的核心骨架：

```cc
bool FastCompilerARM64::ProcessInstructions() {
  DexInstructionIterator it = GetCodeItemAccessor().begin();
  DexInstructionIterator end = GetCodeItemAccessor().end();
  bool flow_continues = false;
  do {
    DexInstructionPcPair pair = *it;
    ++it;

    const Instruction* next = nullptr;
    if (it != end) {
      const DexInstructionPcPair& next_pair = *it;
      next = &next_pair.Inst();
      if (GetLabelOf(next_pair.DexPc())->IsLinked()) {
        next = nullptr;
      }
    }

    vixl::aarch64::Label* label = GetLabelOf(pair.DexPc());
    if (label->IsLinked()) {
      StartBranchTarget(flow_continues, pair.DexPc());
      __ Bind(label);
    }

    if (!ProcessDexInstruction(pair.Inst(), pair.DexPc(), next)) {
      return false;
    }
    ResetTempRegisters();
    ...
    flow_continues = pair.Inst().CanFlowThrough();
  } while (it != end);
  return true;
}
```

这段循环暴露了 fast compiler 的几个核心思想：

1. 它是按 dex 指令顺序扫描的。
2. 它不做大规模 IR 优化，而是边走边发 ARM64 机器码。
3. 它用 `label`、`branch target`、`flow_continues` 去维持基本控制流。
4. 它用轻量状态保存 dex vreg 当前位置，而不是跑一整套 optimizing pipeline。

它内部最关键的数据结构不是 SSA，而是 `vreg_locations_`：

```cc
ArenaVector<Location> vreg_locations_;
```

这意味着 fast compiler 的核心问题是：
- 当前 dex vreg 在哪
- 当前值是寄存器 / FPU 寄存器 / 栈槽 / 常量
- 合流点之前如何把状态收敛
- 调用/异常/branch 处如何保守同步

从提交历史看，这条路径在本区间内是“逐步扩功能”的，而不是一次性做完。演进顺序很像典型 baseline compiler 落地过程：
- 先做 move / const / goto / result / return
- 再补数组读写、字段访问、`instanceof`、`filled-new-array`
- 再补浮点
- 再补 `monitor`、`move-exception`
- 最后再处理 spilling、branch handling、read barrier 等细节

这说明它不是实验性 demo，而是在被认真补成可用编译层。

### 3.2 `optimizing/` 内部结构被拆散：从 `nodes.h` 中抽出独立基础组件

在起点 commit 里，`HGraph`、`HInstructionList`、`HLoopInformation` 等核心类型是集中放在 `compiler/optimizing/nodes.h` 里的。

旧状态代码节选（起点 commit）：

```cc
class HInstructionList : public ValueObject {
  ...
};

class HGraph : public ArenaObject<kArenaAllocGraph> {
  ...
};

class HLoopInformation : public ArenaObject<kArenaAllocLoopInfo> {
  ...
};
```

对应变化量：
- `compiler/optimizing/nodes.cc`：`1682` 行大幅缩减
- `compiler/optimizing/nodes.h`：`1706` 行大幅缩减

终点把这些能力拆成了多个独立文件：
- `graph.cc` / `graph.h`
- `instruction_list.cc` / `instruction_list.h`
- `loop_information.cc` / `loop_information.h` / `loop_information-inl.h`

新增文件变化量：
- `graph.cc`：`+1232`
- `graph.h`：`+564`
- `instruction_list.cc`：`+158`
- `instruction_list.h`：`+76`
- `loop_information.cc`：`+271`
- `loop_information.h`：`+177`
- `loop_information-inl.h`：`+58`

新的 `HInstructionList` 很轻，职责很单一：

```cc
class HInstructionList final : public ValueObject {
 public:
  HInstructionList() : first_instruction_(nullptr), last_instruction_(nullptr) {}

  void AddInstruction(HInstruction* instruction);
  void RemoveInstruction(HInstruction* instruction);
  void InsertInstructionBefore(HInstruction* instruction, HInstruction* cursor);
  void InsertInstructionAfter(HInstruction* instruction, HInstruction* cursor);
  bool Contains(HInstruction* instruction) const;
  bool FoundBefore(const HInstruction* instruction1,
                   const HInstruction* instruction2) const;
  ...
};
```

新的 `HGraph` 也不再是 `nodes.h` 里的一个“巨型配角”，而是明确成为优化 IR 的中心对象：

```cc
class HGraph : public ArenaObject<kArenaAllocGraph> {
 public:
  HGraph(ArenaAllocator* allocator,
         ArenaStack* arena_stack,
         VariableSizedHandleScope* handles,
         const DexFile& dex_file,
         uint32_t method_idx,
         InstructionSet instruction_set,
         InvokeType invoke_type,
         bool dead_reference_safe = false,
         bool debuggable = false,
         CompilationKind compilation_kind = CompilationKind::kOptimized,
         int start_instruction_id = 0)
      : allocator_(allocator),
        arena_stack_(arena_stack),
        ...
        compilation_kind_(compilation_kind),
        useful_optimizing_(false),
        cha_single_implementation_list_(allocator->Adapter(kArenaAllocCHA)) {
    blocks_.reserve(kDefaultNumberOfBlocks);
  }
  ...
};
```

新的 `HLoopInformation` 也从“嵌在大文件里的一段 loop 逻辑”变成独立组件：

```cc
class HLoopInformation final : public ArenaObject<kArenaAllocLoopInfo> {
 public:
  HLoopInformation(HBasicBlock* header, HGraph* graph);
  ...
  void Populate();
  void PopulateInnerLoopUpwards(HLoopInformation* inner_loop);
  bool Contains(const HBasicBlock& block) const;
  bool IsIn(const HLoopInformation& other) const;
  auto GetBlocks() const;
  auto GetBlocksPostOrder() const;
  auto GetBlocksReversePostOrder() const;
  ...
};
```

这轮拆分的设计含义很大：

- 原来 `nodes.*` 既像 IR 定义文件，又像 CFG/loop/container 组合包，职责太厚。
- 新结构把“指令容器”“图结构”“loop 元数据”“寄存器集合”拆开，后续 pass 改动不必反复碰 `nodes.h`。
- 头文件依赖变少，编译器的增量演进成本变低。

这是典型的“系统做大以后必须拆基础设施”的信号。

### 3.3 `RegisterSet` 从 `Location` 旁边独立出来：寄存器集合抽象变干净

这个变化虽然文件不大，但设计意义很强。

起点 commit 中，`RegisterSet` 是放在 `compiler/optimizing/locations.h` 里的，而且接口带明显 `Location` 耦合：

旧状态代码节选（起点 commit）：

```cc
class RegisterSet : public ValueObject {
 public:
  static RegisterSet Empty() { return RegisterSet(); }
  static RegisterSet AllFpu() { return RegisterSet(0, -1); }

  void Add(Location loc) {
    if (loc.IsRegister()) {
      core_registers_ |= (1 << loc.reg());
    } else {
      DCHECK(loc.IsFpuRegister());
      floating_point_registers_ |= (1 << loc.reg());
    }
  }

  void Remove(Location loc) { ... }
  bool OverlapsRegisters(Location out) { ... }
  ...
};
```

终点 commit 里它被移到独立文件 `compiler/optimizing/register_set.h`，接口更像一个纯 bitset/集合抽象：

```cc
class RegisterSet : public ValueObject {
 public:
  static RegisterSet Empty() { return RegisterSet(); }
  static RegisterSet AllFpu() { return RegisterSet(0, -1); }

  void AddCoreRegisterSet(uint32_t registers) { core_register_set_ |= registers; }
  void AddFpuRegisterSet(uint32_t registers) { fpu_register_set_ |= registers; }
  void AddCoreRegister(uint32_t reg) { core_register_set_ |= 1u << reg; }
  void AddFpuRegister(uint32_t reg) { fpu_register_set_ |= 1u << reg; }

  RegisterSet Union(const RegisterSet& other) const { ... }
  RegisterSet Intersect(const RegisterSet& other) const { ... }
  RegisterSet Subtract(const RegisterSet& other) const { ... }
  ...
};
```

对应变化量：
- `compiler/optimizing/locations.h`：`213` 行明显缩减
- `compiler/optimizing/register_set.h`：`+103`

设计差异：

- 旧设计：`RegisterSet` 更像 `Location` 的辅助工具。
- 新设计：`RegisterSet` 是一等概念，服务于 codegen、RA、liveness、blocked/caller-save 管理。

这会直接减少两类耦合：
- `RegisterSet` 不再需要理解太多 `Location` 形态。
- RA/codegen 可以直接操作 core/fpu bitmask，而不是绕一层 `Location` 解释。

### 3.4 Pass 遍历框架统一到 `CRTPGraphVisitor`

这是 `optimizing/` 内部另一条非常清晰的重构主线。提交历史显示多个 pass 和多个架构 simplifier 都切到 `CRTPGraphVisitor`。

代表性代码，来自 `instruction_simplifier.cc`：

```cc
class InstructionSimplifierVisitor final
    : public CRTPGraphVisitor<InstructionSimplifierVisitor> {
 public:
  InstructionSimplifierVisitor(HGraph* graph,
                               CodeGenerator* codegen,
                               OptimizingCompilerStats* stats,
                               bool be_loop_friendly)
      : CRTPGraphVisitor(graph),
        codegen_(codegen),
        stats_(stats),
        be_loop_friendly_(be_loop_friendly) {}

  using CRTPGraphVisitor::ForwardVisit;

  static constexpr auto ForwardVisit(void (CRTPGraphVisitor::*visit)(HShl*)) {
    DCHECK(visit == &CRTPGraphVisitor::VisitShl);
    return &InstructionSimplifierVisitor::HandleShift;
  }
  ...
};
```

这段代码说明的不是“用了模板”这么简单，而是 pass 的组织方式发生了变化：

- 旧风格更多依赖层层 visitor/虚调用/委托访问。
- 新风格用 CRTP 做静态分派，把遍历骨架统一起来。
- 某些 opcode 家族可以通过 `ForwardVisit()` 直接路由到共享处理函数，减少模板/visitor 样板。

设计收益：
- 更少虚调用
- 更一致的 pass 实现风格
- 更容易把架构相关 simplifier 做成统一套路

从提交历史可见，迁移范围包括：
- `InstructionSimplifier`
- `LoadStoreElimination`
- `ReferenceTypePropagation`
- `HeapLocationCollector`
- `PrepareForRegisterAllocation`
- `ProfilingInfoBuilder`
- `HScheduler`
- WBE
- 各架构 simplifier

这说明它不是局部试点，而是 `optimizing/` 的实现范式升级。

### 3.5 线性扫描寄存器分配器被系统性重写

相关变化量非常大：
- `register_allocator_linear_scan.cc`：`+1567 / -...`
- `ssa_liveness_analysis.h`：`+853 / -...`
- `code_generator.cc`：`+311 / -...`
- `code_generator.h`：`+184 / -...`

终点代码里，`RegisterAllocatorLinearScan` 已经深度使用新的 `RegisterSet`，而不是散落的寄存器掩码：

```cc
registers_blocked_for_call_(
    register_allocator->registers_blocked_for_call_.GetCoreRegisterSet()),
available_registers_(register_allocator->available_registers_.GetCoreRegisterSet()),
...
registers_blocked_for_call_(
    register_allocator->registers_blocked_for_call_.GetFpuRegisterSet()),
available_registers_(register_allocator->available_registers_.GetFpuRegisterSet()),
```

它在核心分配逻辑里也更显式地区分了：
- `blocked for call`
- `available registers`
- `hint`
- `register pair`
- `spill slot hint`

例如寄存器 hint 选择：

```cc
int hint = FindFirstRegisterHint(current, available_registers, free_until);
if ((hint != kNoRegister) &&
    !(current->IsPair() &&
      (available_registers & (1u << GetHighForLowRegister(hint))) == 0u)) {
  DCHECK(!IsBlocked(hint));
} else if (current->IsPair()) {
  hint = FindAvailableRegisterPair(available_registers, free_until, current->GetStart());
} else {
  hint = FindAvailableRegister(available_registers, free_until, current);
}
if (hint == kNoRegister) {
  return false;
}
```

再例如 spill slot reuse：

```cc
if (com::android::art::flags::reg_alloc_spill_slot_reuse()) {
  LiveInterval* hint_phi_interval = parent->GetHintPhiInterval();
  if (hint_phi_interval != nullptr && hint_phi_interval->HasSpillSlotOrHint()) {
    size_t hint = hint_phi_interval->GetSpillSlotHint();
    ...
    if (std::all_of(range.begin(), range.end(),
                    [=](SpillSlotData& data) { return data.CanUseFor(parent, start, end); })) {
      ...
      used_hint = true;
      slot = hint;
    }
  }
}
```

从设计角度看，这里发生了几件事：

1. 寄存器集合建模更统一。
2. pair 分配不再只是“碰到 wide 类型时特判一下”，而是系统进入 RA 决策。
3. spill slot reuse 开始显式利用 Phi/hint 关系，说明 RA 更重视栈布局质量。
4. `blocked for call` 和 caller-save 语义被明确成单独维度，而不是散落在 codegen/RA 之间。

换句话说，优化编译器的 RA 从“历史积累逻辑可用”往“模型显式化”演进了一大步。

### 3.6 `CodeGenerator` 接口也在向更清晰的寄存器模型收敛

虽然本文不逐个展开 `code_generator.*`，但从提交历史可以明确看到这些方向：

- 移除 `CodeGenerator::number_of_register_pairs_`
- 重写 `CodeGenerator::NeedsTwoRegisters()`
- 调整 caller-save register handling
- 重构 safepoint handling
- `Location::kNoOutputOverlap` 语义清理

这几条和上面的 RA 改造是同一件事的两面：
- 旧模型里“一个值需要几个寄存器”“一组寄存器如何 blocked”“pair 怎样表示”这些问题分散在多个层次。
- 新模型里这些边界正在逐步被收敛成统一约定。

这对后续所有非标准值模型都很重要，例如：
- 宽类型
- pair register
- spill slot 宽度
- 机器位宽和 dex vreg 宽度不一致时的处理

---

## 4. Runtime：GC、虚拟线程、JIT 和新产物类型

### 4.1 CMC GC 不再只是“收集器实现”，而是进入分代模型

这段区间里 GC 最大的变化点是：
- `Make CMC GC generational`

文件变化量非常大：
- `runtime/gc/collector/mark_compact.cc`：`+3041 / -...`
- `runtime/gc/collector/mark_compact.h`：`+564 / -...`
- `runtime/gc/heap.cc`：`+539 / -...`
- `runtime/gc/heap.h`：`+125 / -...`

终点代码已经明确把 generational 作为 `MarkCompact` 的一等能力：

```cc
// Set to true when doing young gen collection.
bool young_gen_;
const bool use_generational_;
...
// In generational-mode, we maintain 3 generations: young, mid, and old.
// Mid generation is collected during young collections. This means objects
// need to survive two GCs before they get promoted to old-gen.
union {
  uint8_t* black_dense_end_;
  uint8_t* old_gen_end_;
};
...
uint8_t* mid_gen_end_;
```

这几行很关键，因为它说明：

- ART 在这里不是只做“young/old 两代”的最简单模型。
- 终点实现里显式维护 `young / mid / old` 三层概念。
- `mid-gen` 的存在是为了避免“刚活过一次 young GC 的对象被过早晋升 old-gen”。

相关接口也显示出了 generational 配套机制，例如：
- old-gen scan
- card marks
- native roots 指向 young-gen 的记录
- from-space/to-space slide 计算

从 `mark_compact.h` 的注释还能看出一个实现思路：同一个 `MarkCompact` 类同时承载 full compaction 和 young GC 逻辑，避免为 generational 再复制一整套数据结构。

这说明设计上选择的是：
- 复用已有 compaction 基础设施
- 在同一 collector 内引入代际语义
- 通过 card/bitmap/space 边界分层，而不是重起一个完全平行的 collector

### 4.2 virtual thread / continuation 从“概念支持”走到“运行时共同基础设施”

终点新增：
- `runtime/virtual_thread_common.cc`
- `runtime/virtual_thread_common.h`

同时新增大量测试：
- `test/2390-virtual-thread-carrier-leak`
- `test/2390-virtual-thread-context-leak`
- `test/2390-virtual-thread-parking-error-leak`
- `test/2391-virtual-thread-sleeps`
- `test/2392-virtual-thread-pinning-jni`
- `test/2393-virtual-thread-pinning-monitor`
- `test/2394-continuation-yields`
- `test/2395-virtual-thread-reentrantlock`
- `test/2395-virtual-thread-sleep-api`

这说明 runtime 对 virtual thread 的支持已经不是“留接口”，而是开始认真处理：
- parking/unparking
- stack walking
- frame materialization
- pinning reason
- JNI / monitor 等不可迁移条件

代表性代码：

```cc
struct VirtualThreadParkingVisitor final : public StackVisitor {
  ...
  bool VisitFrame() override {
    ShadowFrame* shadow_frame = GetCurrentShadowFrame();

    if (shadow_frame == nullptr) {
      ArtMethod** quick_frame = GetCurrentQuickFrame();
      ArtMethod* method = quick_frame != nullptr ? *quick_frame : nullptr;
      if (method != nullptr && method->IsNative()) {
        if (method == park_method_) {
          return true;
        } else if (method == enter_method_) {
          return false;
        }
        reason_ = kNativeMethod;
        return false;
      }
      reason_ = kUnsupportedFrame;
      return false;
    }

    if (!shadow_frame->GetLockCountData().IsEmpty()) {
      reason_ = kMonitor;
      return false;
    }
    ...
  }
};
```

这段代码很能说明设计思路：

- 虚拟线程停车前，runtime 需要遍历栈，确认每一帧是否可序列化/可迁移。
- 一旦遇到 native frame、monitor、carrier thread 边界等情况，就记录 pinning reason。
- 这不是单纯“把 Java stack 保存一下”，而是带运行时语义约束的 stack capture。

后续它会把 shadow frame 内容真正转成 Java 可持有对象：

```cc
for (size_t i = 0; i < num_frames; i++) {
  const ShadowFrame* sf = dump_visitor.shadow_frames_[i];
  ...
  size_t num_vergs = sf->NumberOfVRegs();
  int32_t non_vref_size = ShadowFrame::ComputeSizeWithoutReferences(num_vergs);
  ...
  frame_bytes->Memcpy(0, reinterpret_cast<const int8_t*>(sf), 0, non_vref_size);
  ...
}
```

实现含义非常清楚：
- 非引用部分按字节复制
- 引用单独收集进 `refs`
- 虚拟线程 parked state 不是“保留原生栈”，而是“转成 runtime 可管理对象”

这和传统线程模型的设计心智是完全不一样的。

### 4.3 JIT / root table / intern 行为更保守、更抗 GC 交互问题

这一段 JIT 相关变化不像 fast compiler 那么显眼，但从提交主题看，方向很稳定：

- `Use jit_mutator_lock in JitCodeCache::VisitRootTables`
- `Fix JitCodeCache::RemoveUnmarkedCode()`
- `Do not strongly intern strings for JIT root table`
- `Make const-string interns weak`
- `Use InternWeak() for dex cache location ...`

这些变化放在一起看，表达的是同一个设计目标：
- 减少 JIT code cache、root table、string intern table 之间的强耦合
- 避免 GC、code cache 清理、字符串 intern 生命周期之间出现脆弱路径

也就是说，这里不是“加功能”，而是 runtime 内部一致性在变强。

### 4.4 新增 SDC 文件：runtime/oat 产物不再只有传统 oat/vdex/art

终点新增：
- `runtime/oat/sdc_file.cc`
- `runtime/oat/sdc_file.h`
- `runtime/oat/sdc_file_test.cc`

这说明 OAT 产物链路里新增了一个新的 companion file 类型，而且它不是随意文本，而是被 runtime 显式解析和验证。

代表性代码：

```cc
std::unique_ptr<SdcReader> SdcReader::Load(const std::string& filename,
                                           std::string* error_msg) {
  ...
  if ((it = map.find("sdm-timestamp-ns")) == map.end()) {
    *error_msg = ART_FORMAT("Missing key 'sdm-timestamp-ns' in sdc file '{}'", filename);
    return nullptr;
  }
  ...
  if ((it = map.find("apex-versions")) == map.end()) {
    *error_msg = ART_FORMAT("Missing key 'apex-versions' in sdc file '{}'", filename);
    return nullptr;
  }
  ...
}
```

写入时也很直接：

```cc
std::string content =
    ART_FORMAT("sdm-timestamp-ns={}\napex-versions={}\n",
               sdm_timestamp_ns_, apex_versions_);
```

这个格式设计说明：
- 它不是大而全的 metadata 容器
- 它只记录 companion 文件判定所需要的最小信息
- 其中 `sdm timestamp` 和 `apex versions` 是一致性判断关键

而且因为它位于 `runtime/oat/`，说明这不是单纯服务层私有逻辑，而是 ART runtime 认可的产物类型。

---

## 5. Verifier：从“历史路径堆叠”走向“更平直、更快”的验证流程

### 5.1 `runtime/verifier/` 这一段不是小修，而是重写热路径

相关变化量：
- `runtime/verifier/method_verifier.cc`：`1499 insertions / 1201 deletions`
- `runtime/verifier/register_line*.{cc,h}`：多处变更
- `runtime/verifier/reg_type*.{cc,h}`：多处变更

这不是单纯 patch，而是明显的代码路径重整。

代表性代码先看整个 verifier pipeline：

```cc
bool result = ComputeWidthsAndCountOps();
result = result && ScanTryCatchBlocks();
result = result && VerifyInstructions();
result = result && VerifyCodeFlow();
return result;
```

这个顺序很重要，说明终点实现把验证拆成了四段：

1. 指令宽度与 opcode 边界扫描
2. try/catch 区间和 handler 地址处理
3. 静态逐指令合法性验证
4. code-flow 级别验证

`ComputeWidthsAndCountOps()` 也明显是重写后的热点函数，强调边界安全和局部内联：

```cc
bool MethodVerifierImpl::ComputeWidthsAndCountOps() {
  const uint32_t insns_size = code_item_accessor_.InsnsSizeInCodeUnits();
  const Instruction* inst = &code_item_accessor_.InstructionAt(0u);
  uint32_t dex_pc = 0u;
  while (dex_pc != insns_size) {
    const uint32_t remaining_code_units = insns_size - dex_pc;
    const uint16_t inst_data = inst->Fetch16(0);
    const Instruction::Code opcode = inst->Opcode(inst_data);
    ...
    if (!ok) {
      Fail(VERIFY_ERROR_BAD_CLASS_HARD) << "code did not end where expected";
      return false;
    }
    GetModifiableInstructionFlags(dex_pc).SetIsOpcode();
    dex_pc += instruction_size;
    inst = inst->RelativeAt(instruction_size);
  }
  return true;
}
```

设计特点：
- 尽早做结构边界检查
- 对复杂 NOP payload（switch/array-data）做单独长度处理
- 使用 64-bit 运算避免乘法溢出
- 在最前阶段就把 opcode 边界标出来，为后续 verifier 复用

### 5.2 静态逐指令校验变得更模板化、更可预测

终点实现中的 `VerifyInstructions()` 已经不是“巨型 switch 里顺手夹各种逻辑”的旧风格，而是先做 compile-time dispatch 归类，再调用模板化 `VerifyInstruction<Opcode>()`。

代表性代码：

```cc
switch (dispatch_opcode) {
#define DEFINE_CASE(opcode, c, p, format, index, flags, eflags, vflags)             \
  case opcode: {                                                                     \
    constexpr Instruction::Code kOpcode = enum_cast<Instruction::Code>(opcode);     \
    if (!VerifyInstruction<kOpcode>(dex_pc, end_dex_pc, inst, inst_data)) {         \
      return false;                                                                  \
    }                                                                                \
    is_return = Instruction::IsReturn(kOpcode);                                     \
    instruction_size = ...;                                                          \
    break;                                                                           \
  }
  DEX_INSTRUCTION_LIST(DEFINE_CASE)
#undef DEFINE_CASE
}
```

这个设计变化带来的好处：
- 相同 format / verify-flag 的 opcode 可以共享模板路径
- dispatch 行为更规则
- 便于把 `CheckUnaryOp`、`CheckBinaryOp`、`CheckLiteralOp` 等 helper 全部内联进热路径

所以提交历史里的这些变更是互相配合的：
- `Inline Check.*Op*()`
- `Rewrite branch target verification`
- `Speed up ComputeWidthsAndCountOps()`
- `Speed up failure recording`
- `Remove FailOrAbort()`

它们共同的结果是：
- verifier 更少依赖历史分支
- 错误记录路径更轻
- 热路径更容易被编译器优化

### 5.3 分支、异常处理器、`move-exception` 的约束被写得更硬

终点代码对 branch target 和 handler 入口约束非常明确：

```cc
// Verify that the target of a branch instruction is valid.
// We don't expect code to jump directly into an exception handler, but it's
// valid to do so as long as the target isn't a "move-exception" instruction.
```

以及：

```cc
if (UNLIKELY(next_opcode == Instruction::MOVE_EXCEPTION)) {
  Fail(VERIFY_ERROR_BAD_CLASS_HARD) << "Can flow through to move-exception";
  return false;
}
```

还有：

```cc
if (work_insn_idx_ == 0) {
  Fail(VERIFY_ERROR_BAD_CLASS_HARD) << "move-exception at pc 0x0";
  return {false, false};
}
```

这些检查表达出一个很明确的实现目标：
- 对异常路径入口的合法性做更严格定义
- 减少历史上“某些奇怪字节码能侥幸通过”的空间
- 把 handler、fallthrough、branch target 的语义边界写死

### 5.4 `filled-new-array`、宽类型、构造函数 return 等边界都被细化

终点注释里已经明确写出：

```cc
// For `filled-new-array*`, check for a valid component type; `I` is accepted,
// `J` and `D` are rejected in line with the specification ...
```

还有构造函数 return 的专门检查：

```cc
if (IsInstanceConstructor() && UNLIKELY(!work_line_->CheckConstructorReturn(this))) {
  return false;
}
```

这说明 verifier 的变化不是“只提速”，而是：
- 规范边界更明确
- 宽类型和对象初始化语义更收紧
- 失败原因更集中、更稳定

---

## 6. Hidden API：从 Java/Reflection 检查扩展到 Native/JNI/Core Platform

### 6.1 hiddenapi 的域模型更完整

终点 `runtime/hidden_api.cc` 已经明确把访问主体和被访问方按 domain 划分：

```cc
std::ostream& operator<<(std::ostream& os, Domain domain) {
  switch (domain) {
    case Domain::kCorePlatform:
      os << "core-platform";
      break;
    case Domain::kPlatform:
      os << "platform";
      break;
    case Domain::kApplication:
      os << "app";
      break;
  }
}
```

并且会根据 dex location / APEX location 推导 domain：

```cc
if (LocationIsOnArtModule(location) || LocationIsOnConscryptModule(location)) {
  return Domain::kCorePlatform;
}
if (LocationIsOnApex(location)) {
  return Domain::kPlatform;
}
...
return Domain::kApplication;
```

这意味着 hiddenapi 不再只是在“当前调用是不是 app”这种粗粒度上决策，而是在建立更细的信任域模型：
- core-platform
- platform
- app

### 6.2 JNI caller 现在也进入 hiddenapi enforcement 路径

这段区间最大的 hiddenapi 设计变化，是 native/JNI caller 被系统纳入检查。

代表性代码：

```cc
bool ShouldDenyJniAccessToMember(T* member,
                                 Thread* self,
                                 AccessMethod access_kind,
                                 void* native_caller_addr) {
  ...
  return ShouldDenyAccessToMember(
      member,
      [&ctx]() {
        if (com::android::art::flags::hiddenapi_jni_api_callers()) {
          AccessContext& context = ctx.GetNativeCallerContext();
          if (context.GetNativeCallerAddr() != nullptr) {
            switch (context.GetDomain()) {
              case Domain::kApplication:
                ...
              case Domain::kPlatform:
                ...
              case Domain::kCorePlatform:
                ...
            }
          }
        }
        AccessContext& context = ctx.GetJavaCallerContext();
        return context;
      },
      access_kind);
}
```

这段逻辑的设计含义非常强：

- hiddenapi 已经不再只看 Java 栈上的 managed caller。
- 对 JNI 下来的访问，runtime 尝试识别 native caller 的 DSO/domain。
- 如果 native caller 可识别，就按 native caller 的 domain 做策略决策。
- 识别失败时，才退回 Java caller。

这和旧阶段相比，是 enforcement 范围的实质扩张。

### 6.3 平台和 core platform 的边界被真正确立

终点代码里不仅有 domain，还有专门的 core-platform violation 处理逻辑：

```cc
case Domain::kPlatform: {
  DCHECK(callee_context.GetDomain() == Domain::kCorePlatform);
  ...
  return detail::HandleCorePlatformApiViolation(
      member, api_list, runtime_flags, caller_context, callee_context, access_method, policy);
}
```

这意味着：
- 平台访问 core-platform 已经不是“模糊允许/仅日志”
- ART 开始把它当成单独语义边界
- policy 可以是 deny / just-warn / check-only 等

提交历史里与之配套的变化包括：
- enforce core platform hiddenapi checks
- JNI `GetMethodID` / `GetFieldID` 也做 hiddenapi check
- 记录 caller/callee 到 denial message
- just-warn 模式下也保持信息完整

### 6.4 hiddenapi 测试不再只测 Java 路径

新增/调整的测试很说明问题：
- `test/674-hiddenapi`
- `test/817-hiddenapi`
- `libarttest(d)_external`

这说明测试目标已经扩大到：
- Java 调用者
- native 调用者
- 平台 / app / external 变体

即，hiddenapi 的设计从“反射限制”变成“ART 访问控制的一部分”。

---

## 7. artd / Dexopt：从工具服务进化成带状态机的系统服务

### 7.1 `IArtd.aidl` 扩张非常明显，接口职责已经变了

终点 `IArtd.aidl` 新增了多类能力：
- artifacts location 返回
- SDC/SDM 相关接口
- pre-reboot staged files 检查与提交
- runtime artifacts 管理
- pre-reboot init / path 校验

代表性接口节选：

```aidl
void maybeCreateSdc(in com.android.server.art.OutputSecureDexMetadataCompanion outputSdc);

boolean commitPreRebootStagedFiles(
        in List<com.android.server.art.ArtifactsPath> artifacts,
        in List<com.android.server.art.ProfilePath.WritableProfilePath> profiles);

@nullable com.android.server.art.PreRebootStagedFilesStatus
        checkPreRebootStagedFilesStatus();

boolean preRebootInit(
        in @nullable com.android.server.art.IArtdCancellationSignal cancellationSignal);
```

这类接口组合说明 artd 的职责已经从“帮 Java 服务跑 dexopt”变成：
- 产物定位与状态查询服务
- staged 文件生命周期服务
- pre-reboot dexopt 执行环境服务
- 新产物类型 companion 文件管理服务

### 7.2 Pre-reboot Dexopt 不再是临时逻辑，而是有 metadata 的正式工作流

终点 `artd.cc` 明确新增了 `PreRebootStagedMetadata`，而且特地考虑了“旧版本 ART 只需要看创建时间”的兼容场景。

代表性代码：

```cc
// File format:
//   <magic>
//   <build_fingerprint>
//   <apex_timestamps>
class PreRebootStagedMetadata {
 public:
  static Result<PreRebootStagedFilesStatus> Check(...);
  static Result<void> Save(...);
 private:
  static constexpr std::string_view kMagic = "PRE_REBOOT_STAGED_METADATA_001";
};
```

检查逻辑也明确绑定了：
- build fingerprint
- apex timestamps
- 文件创建时间

```cc
if (build_fingerprint != expected_build_fingerprint) {
  ret.isCommittable = false;
  ret.reason = ART_FORMAT("Build fingerprint mismatch ...");
  return std::move(ret);
}

if (apex_timestamps != expected_apex_timestamps) {
  ret.isCommittable = false;
  ret.reason = ART_FORMAT("APEX timestamps mismatch ...");
  return std::move(ret);
}
```

这说明 pre-reboot dexopt 的设计目标已经变成：
- staged 文件是否还能提交，不能只看文件是否存在
- 必须看系统版本、APEX 状态是否仍匹配
- ART 自己要维护这套可判断、可回滚、可清理的 metadata

### 7.3 SDC 的生命周期由 artd 管起来了

终点 `Artd::maybeCreateSdc()` 把 `runtime/oat/sdc_file.*` 真正接到了服务层工作流里：

```cc
ndk::ScopedAStatus Artd::maybeCreateSdc(const OutputSecureDexMetadataCompanion& in_outputSdc) {
  ...
  std::string sdm_path = OR_RETURN_FATAL(BuildSdmPath(in_outputSdc.sdcPath));
  std::string sdc_path = OR_RETURN_FATAL(BuildSdcPath(in_outputSdc.sdcPath));
  ...
  std::unique_ptr<SdcReader> sdc_reader = SdcReader::Load(sdc_path, &error_msg);
  if (sdc_reader != nullptr &&
      sdc_reader->GetSdmTimestampNs() == TimeSpecToNs(sdm_st.st_mtim)) {
    return ScopedAStatus::ok();
  }
  ...
  writer.SetSdmTimestampNs(TimeSpecToNs(sdm_st.st_mtim));
  writer.SetApexVersions(OR_RETURN_NON_FATAL(injector_->GetApexVersions(this)));
  ...
}
```

这段实现很说明设计取向：

- companion 文件不是随 dexopt 顺带生成，而是有单独 API 和一致性规则。
- artd 会拿 SDM mtime 和 SDC 内容做匹配。
- APEX versions 被直接写入 SDC，用来判断 companion 是否还能复用。

这是明显的“产物元数据服务化”。

### 7.4 staged 文件提交从“简单 rename”升级成“事务式提交”

终点 `commitPreRebootStagedFiles()` 会统一收集待移动文件，再通过 `MoveAllOrAbandon()` 提交：

```cc
ScopedAStatus Artd::commitPreRebootStagedFiles(const std::vector<ArtifactsPath>& in_artifacts,
                                               const std::vector<WritableProfilePath>& in_profiles,
                                               bool* _aidl_return) {
  ...
  std::vector<std::pair<std::string, std::string>> files_to_move;
  std::vector<std::string> files_to_remove;
  ...
  OR_RETURN_NON_FATAL(MoveAllOrAbandon(files_to_move, files_to_remove));
  ...
}
```

设计差异：

- 旧思路更像“dexopt 生成产物，然后放到目的地”。
- 新思路更像“先 staging，确认系统条件成立，再批量 commit”。

和下面这个接口配套看就更明显：

```cc
ScopedAStatus Artd::checkPreRebootStagedFilesStatus(
    std::optional<PreRebootStagedFilesStatus>* _aidl_return) {
  ...
  Result<PreRebootStagedFilesStatus> status = PreRebootStagedMetadata::Check(...);
  ...
}
```

这已经是带状态判断的事务流了，不是简单文件工具。

### 7.5 Pre-reboot 是否允许运行，也被正规化成策略判断

终点还有：

```cc
ScopedAStatus Artd::checkPreRebootSystemRequirements(const std::string& in_chrootDir,
                                                     bool* _aidl_return) {
  ...
  if (new_release - old_release >= 2) {
    LOG(WARNING) << ART_FORMAT(
        "Pre-reboot Dexopt not supported due to large difference in release versions ...");
    *_aidl_return = false;
    return ScopedAStatus::ok();
  }
  *_aidl_return = true;
  return ScopedAStatus::ok();
}
```

这说明：
- pre-reboot dexopt 已经不是“只要 chroot 搭起来就干”
- ART 开始对旧系统/新系统版本差、支持矩阵、可测试性做显式策略控制

换句话说，artd 已经有了小型状态机的味道。

---

## 8. 测试、Fuzzer、Build：新增能力正在被系统性护栏化

### 8.1 测试目录是全仓变化最多的目录

`test/` 占比达到 `25.4%`，说明这段区间里大量工作是在为新增能力补稳定性护栏，而不是只做实现。

热点测试主题：
- fast compiler
- hiddenapi
- virtual thread / continuation
- 异常/分支/寄存器/对象边界

这很重要，因为它反映出新增设计已经不是“内部实验”：
- fast compiler 有了专门 regression tests
- hiddenapi 开始覆盖 native caller 场景
- virtual thread 开始覆盖 leak / pinning / park/yield 行为

### 8.2 Fuzzer 体系明显更完整

新增：
- `tools/fuzzer/libart_baseline_compiler_fuzzer.cc`
- `tools/fuzzer/libart_optimizing_compiler_fuzzer.cc`
- `tools/fuzzer/fuzzer_common.cc`
- `tools/fuzzer/fuzzer_common.h`
- baseline / optimizing / class-verifier / dex-verifier corpora

这很能说明本区间的工程哲学：
- 既然新增了 baseline compiler，就给它单独加 fuzzer
- verifier 既然在重写热路径，就补专门 corpus
- 不是只靠单元测试，而是主动扩大输入空间覆盖

### 8.3 Build / flags / boot image 也在配合新能力

本区间 build 侧也有不小变化，例如：
- `Android.mk => art.mk`
- `build/boot/boot-image-profile.txt` 大幅变化
- `build/boot/preloaded-classes` 大幅变化
- `build/flags/art-rw-flags.aconfig` 新增

这说明新增 runtime/编译器能力并不是孤立存在，而是已经开始进入：
- boot image profile
- 构建 flag
- APEX / release build 配置

即，整个系统在吸收这些新能力。

## 9. 最终总结

从 `795d594...` 到 `1690c691...`，ART 不是简单“多了很多 patch”，而是发生了以下层面的系统演进：

- 编译器层：新增 ARM64 fast compiler，确立 baseline compiler 层级。
- 优化器内部：把长期堆积在 `nodes.*` / `locations.*` 周围的基础设施拆开，开始形成更清晰的 IR、loop、instruction list、register set、visitor、RA 模型。
- runtime 层：GC 进入 generational CMC，virtual thread/continuation 进入共同 runtime 支撑，JIT/GC/intern 交互更稳。
- 验证与访问控制：verifier 热路径更直接，hiddenapi 从 managed caller 扩展到 native/JNI/core-platform 边界。
- 服务层：artd 和 pre-reboot dexopt 进入“带 metadata、带 staged 状态、带 companion artifact”的事务化管理阶段。
