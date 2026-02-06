---
layout: default
title: dex2oat
nav_order: 0
parent: AndroidRuntime
has_children: true
---

`dex2oat` 用于将 Android 应用的 **DEX (Dalvik Executable)** 转换为运行时可加载的 **AOT 编译产物**（面向具体 ISA，例如 ARM64），从而提升应用在设备上的执行效率。它通常在 **dexopt 场景**触发运行，例如：**应用安装/更新**、**设备空闲充电时的后台维护编译**、**系统 OTA/恢复** 等；并不是“每次启动应用时”都运行一遍。

## dex2oat 的作用

1. **优化性能**
   `dex2oat` 将 Dex 字节码编译为机器码，并进行多轮优化，减少解释执行与部分运行时开销，从而提升启动速度和运行效率。

2. **AOT 与 JIT 的关系**
   AOT 会降低运行时解释/JIT 的比例，但现代 ART 通常是 **AOT + JIT 的混合策略**：

   * 未被 AOT 覆盖的方法仍可能解释执行或被 JIT 编译；
   * 热点方法/热点路径仍可能由 JIT 进一步优化（取决于 compiler filter、profile、运行时行为等）。

3. **编译产物**
   在现代 Android 系统中，dex2oat 往往会生成一组与运行时加载相关的产物（命名和组合可能随版本/设备策略变化），常见包括：

   * `.odex`：包含 AOT 编译的本地代码（及相关信息）

   * `.vdex`：包含验证/加速用的元数据（有时也与 dex 内容的组织有关）

   * `.art`：在某些场景下用于启动相关的额外数据

   > 对第三方应用而言，APK 里的 `classes.dex` 通常仍然存在；系统额外生成上述产物供运行时加速使用。

---

## 从 Dex 到 ARM64：总体流水线

从 Dex 字节码到 ARM64 机器码，ART 优化编译器总体遵循：

> **构建 IR → SSA 转换 → 多轮优化 → 寄存器分配 → 机器码生成**

各阶段中 `HInstruction` 作为核心 IR 节点不断变化：
从初始直接对应 Dex 操作，到引入 Phi/临时节点，再到 SSA 形式；经过优化可能被折叠、删除、替换；最终每个“存活”的 `HInstruction` 都会映射为一段具体的机器指令序列。

---

## AOT vs JIT：编译器复用与差异

ART 的优化编译器同时服务于 AOT 和 JIT，因此在设计上需要权衡“编译耗时”和“执行性能”。

* **AOT**：通常对应用较大范围的方法执行完整流程，生成质量更高的静态代码。
* **JIT**：在运行时对热点方法执行相同/相近的流程（某些极端场景可能裁剪），并可能利用更多运行时信息做更精准的优化（如更好的类型信息、更强的去虚化机会、OSR 等）。

两者生成的代码语义等价，但在**内联深度**、**类型假设**、**去虚化程度**等细节上可能不同。

---

## Dex 字节码到 ARM64 机器码编译流程概览

ART（Android Runtime）的优化编译器将在 **Ahead-of-Time (AOT)** 与 **Just-in-Time (JIT)** 两种模式下将 Dex 字节码编译为本地机器码。两者复用同一套编译器与大部分优化 Pass，只是 JIT 由于拥有更多运行时信息，最终代码细节可能略有差异。

---

## 1. Dex → HGraph：构建中间表示 (IR)

编译流程的第一步是将 Dex 字节码转换为 IR。ART 使用 `HGraphBuilder` 完成。

### 1.1 构建基本块与 CFG

* 扫描 Dex 字节码
* 依据跳转与异常处理信息划分基本块（`HBasicBlock`）
* 建立前驱/后继关系形成初步 CFG
* 异常路径会额外创建入口/退出块、try/catch 相关块等特殊结构

### 1.2 填写基本块指令（Dex → HInstruction）

在 CFG 建好之后：

* 遍历每条 Dex 指令
* 翻译为对应的 `HInstruction` 并插入所在基本块

常见映射：

* 算术：`HAdd` / `HSub` / …
* 调用：`HInvoke`
* 字段访问：`HInstanceFieldGet/Set`
* 控制流：终结块并链接目标块

构建过程中还会插入必要的辅助节点以满足运行时语义，例如：

* `HLoadClass`（类加载检查）
* `HNullCheck`（空指针检查）

### 1.3 数据流与 Phi（SSA 预备）

* 每个基本块内形成按顺序排列的 HInstruction 链表
* 在控制流汇合处预先创建 Phi（`HPhi`）占位
* 通过维护 Dex vreg 当前值，建立“多前驱合并”的基本结构

### 1.4 基础分析

构建完成后通常还会进行：

* 方法寄存器数/out 寄存器数计算
* 支配树构造
* 循环信息分析
  这些为后续优化 Pass 提供基础。

---

## 2. SSA 转换：SsaBuilder

ART 使用 SSA 形式以便优化。`HGraphBuilder` 完成基础图后，会调用 `SsaBuilder`：

### 2.1 插入/修正 Phi

* 在汇合块中对需要合并的 vreg 插入 `HPhi`
* 将每个 Phi 的输入与各前驱块的输出正确连接

### 2.2 消除临时 Load/Store（如存在）

某些实现路径中可能存在显式的 “load/store 本地变量” 类节点；SSA 阶段会：

* 将其替换为直接的值引用
* 删除无用的中间节点
  最终值从定义点沿数据流直接传递给使用点或 Phi。

### 2.3 原始类型修正（Primitive Type Fixup）

Dex 常量初始可能按 int 处理，但后续可能用于 float/double 运算。SSA 阶段会：

* 将用于浮点运算的 int 常量替换为 `HFloatConstant`/`HDoubleConstant`
* 对 Phi：通过输入推导类型；无法解决则标记为 dead phi 以待删除

### 2.4 引用类型修正：首次 RTP

SSA 初步完成后会运行一次 **引用类型传播**（RTP）：

* 推断引用类的上界/精确类型与可空性（nullability）
* 为 HInstruction 附加 `ReferenceTypeInfo`
* 遇到类型冲突（如 Phi 合并不兼容类型）可能插入 `HBoundType` 或将 Phi 标记 dead

---

## HInstruction 是什么

`HInstruction` 是 ART 优化编译器的核心 IR 节点类型，每个节点表示一个操作（如算术、逻辑、方法调用等）。
它们之间通过输入/使用关系形成依赖图：

* 每个 HInstruction 持有指向其操作数的引用（Input），自身也可被后续指令用作输入（Use）。
* 构建阶段，每个 Dex 寄存器通常映射为一个 HInstruction 值：HGraphBuilder 维护“当前 vreg 值表”，写寄存器时更新条目，读寄存器时取出条目，从而把 Dex 的数据流依赖显式化。
* 注意：由于循环与 Phi 的存在，这个依赖图 **不保证无环**。

完成上述步骤后，SSA 形式的 HGraph 构建完毕，并移除了冗余/未使用的 Phi 节点。此时每个 HInstruction（包括 Phi）都有确定的类型，每个数据流变量只有单一路径定义，IR 图进入优化阶段。

---

## 3. 优化阶段：HInstruction 的演化

ART 在 SSA IR 上按顺序执行一系列优化 Pass。HInstruction 可能：

* 被替换（ReplaceWith）
* 被折叠为常量
* 被删除（死代码）
* 被移动（调度/循环外提等）

优化集合通常包括：常量折叠、指令简化、死代码消除、引用类型传播、GVN、循环优化、边界检查消除、类层次分析等。AOT/JIT 下大体相同，JIT 可能利用更多运行时信息，并在热点/OSR 等场景走特殊路径。

### 3.1 常量折叠（HConstantFolding）与指令简化

**常量折叠**：输入全为常量时，编译期求值并替换原指令。
例：`HAdd(2, 3)` → `HIntConstant(5)`
关键机制：`HInstruction::ReplaceWith` 更新 use 链，然后删除旧节点。

**指令简化（InstructionSimplifier 等）**：不一定产生新常量的模式替换。
例：乘 1、加 0、冗余 cast 消除；乘以 2 替换为左移 1 等。

> 常见约定：能产生常量的放入 ConstantFolding；不产生常量的放入 Simplifier。

### 3.2 死代码消除（HDeadCodeElimination）

* 删除无用计算（结果无人使用且无副作用）
* 清理不可达块/空块并调整 CFG
* 清理 dead Phi（用户都消失后）

注意：带副作用的指令（写内存、调用可能抛异常等）不会随意删除，通常需结合副作用分析判断。

### 3.3 引用类型传播（RTP）再次运行

在内联及多轮优化之后，需要再次运行 RTP 以更新类型信息：

* `HNewInstance/HNewArray`：标记为精确类型
* 字段/数组读取：根据声明类型推断；无法确定则保守为 `Object`
* `HInstanceOf/HCheckCast`：可能插入 `HBoundType` 收紧分支内类型
* `HInvoke`：根据解析到的方法返回类型推断；JIT 更可能拿到更精确类型
* `HPhi`：求输入类型交集/共同父类；冲突时降级或拆分/标记 dead

RTP 通常使用 worklist 迭代直至收敛。

### 3.4 其他常见优化

* GVN（消除重复计算）
* LICM（循环不变外提）
* Bounds Check Elimination（边界检查消除）
* 写屏障/读屏障相关优化
* 指令调度（重排同一基本块内指令顺序）

---

## 4. 寄存器分配：SSA 值 → 物理寄存器/栈槽

生成机器码之前，需要将每个 HInstruction 的值放到具体位置（寄存器或栈）。

### 4.1 活跃区间（Liveness）与线性扫描

* 计算每个 SSA 值 live interval
* 寄存器不足时 spill 到栈
* 考虑调用约定与保留寄存器（如参数寄存器 x0-x7 等）

### 4.2 分配与 LocationSummary 落地

* 位置需求由 `LocationSummary` 描述
* 分配完成后，`LocationSummary` 的虚拟位置被替换为具体物理寄存器/栈槽

### 4.3 插入 Parallel Move：RegisterAllocationResolver

寄存器分配可能导致同一个值在不同点位置不同，需要插入搬移：

* **区间分裂连接**：同值不同片段需 move 衔接
* **Phi 解析（块边界）**：前驱出口到 Phi 目标位置的 move
* **异常/非线性控制流**：进入 catch 等路径时恢复值位置

实现方式：插入 `HParallelMove` 伪指令，内部保存多对 move（逻辑并行），最终在生成阶段展开。

---

## 5. 指令选择与调度：生成 ARM64 机器码

最终阶段由 `CodeGeneratorARM64` 将 IR 转为 ARM64 指令：

### 5.1 生成Prologue

* 分配栈帧
* 保存 LR/callee-save 寄存器等

### 5.2 遍历基本块并生成指令

* 按 linear order 遍历 CFG
* 块头打 label
* 逐条访问 HInstruction：`VisitXxx` 输出具体汇编

示例映射：

* `HAdd` → `ADD`
* `HIf` → `CMP/FCMP` + 条件跳转 `B.eq/B.ne`
* `HInstanceFieldGet` → `LDR`（必要时拆成多条地址构造指令）

### 5.3 调用与返回

* `HInvoke`：按 ABI 搬运参数到 x0-x7 / d0-d7 或栈
* 然后 `BL` 调用目标
* 返回值在 x0/d0，后续直接使用
* `HReturn`：返回序列 + epilogue

### 5.4 ParallelMove 展开：ParallelMoveResolverARM64

`HParallelMove` 不对应单条机器指令，会被 resolver 展开为 MOV/LDR/STR 等序列：

* 先处理 stack-to-stack（需要借助寄存器）
* 再处理 reg<->reg / reg<->stack
* 遇到环依赖用临时寄存器/交换策略打破
* 最后处理常量搬移

### 5.5 Epilogue 与结束

* 恢复现场
* `RET`

AOT 会把机器码及元数据写入对应的编译产物；JIT 则写入 code cache 并修正入口以便后续直接执行。
