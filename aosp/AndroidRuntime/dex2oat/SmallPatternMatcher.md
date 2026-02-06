---
layout: default
title: SmallPatternMatcher
nav_order: 7
parent: dex2oat
author: Anonymous Committer
---
# JIT Small Pattern Matcher

## 摘要

Android ART 在 JIT 编译链路中包含 Small Pattern Matcher（小模式匹配器），用于识别一类指令序列极短、语义高度确定的 Dex 方法实现。该组件通过对 Dex 指令“形状”的快速检查，在满足严格安全约束的前提下，将方法入口直接绑定到预定义的共享 stub（C++ 实现的通用执行入口），从而避免为此类方法生成专用机器码，降低编译开销并提升总体运行时效率。

---

## 1. 设计目标与使用位置

Small Pattern Matcher 的目标是处理“极小方法体”：

* 指令数非常少（通常为 1、2、3、4、6 个 code units）
* 无控制流结构（无分支、无循环、无异常边的复杂交织）
* 语义可由少量局部约束完全确定（例如返回常量、简单字段读写、简化构造函数）

当匹配成功时，系统不进入常规编译流程（IR 构建、优化、寄存器分配、代码生成等），而是直接返回一个共享 stub 的入口地址，并用于更新该方法的执行入口（entrypoint）。

---

## 2. 总体工作流程

Small Pattern Matcher 的核心流程可概括为：

1. 读取方法的 Dex CodeItem（主要关注 `insns_size_in_code_units`、寄存器数、参数数等元信息）。
2. 以指令长度 `insns_size` 为第一层分流依据，将方法划分到有限的候选桶（如 1/2/3/4/6）。
3. 在候选桶中对 opcode 序列进行形状匹配（例如 `const + return`、`iget + return`）。
4. 对潜在语义风险点执行严格检查（例如 volatile、final、空指针检查语义、字段解析与访问检查、offset 范围等）。
5. 成功则返回 stub 指针；失败则返回 `nullptr`，交由常规编译路径处理。

该策略的关键在于：以极低成本判断“是否可安全替代为 stub”，且在不满足任何约束时坚决退回正常路径，避免语义偏差。

---

## 3. 可匹配情形汇总表

下表按常见实现逻辑整理 Small Pattern Matcher 覆盖的主要情形（以 Dex 形状示意，非完整语法）：

| 类别                                | Dex 形状（示意）                                                       | insns_size（code units） | 关键约束（常见拒绝原因）                                                                             | 返回的 stub（典型）                                        |
| --------------------------------- | ---------------------------------------------------------------- | ---------------------: | ---------------------------------------------------------------------------------------- | --------------------------------------------------- |
| 空方法                               | `return-void`                                                    |                      1 | 无                                                                                        | `EmptyMethod`                                       |
| 仅调用 `Object.<init>` 的构造函数         | `invoke-direct vThis, Object.<init>` + `return-void`             |                      4 | 必须为实例 `<init>`；父类必须是 `java.lang.Object`；目标 `Object.<init>` 必须为“空构造”形态                    | `EmptyMethod`                                       |
| 返回第一个参数（`return this` / identity） | `return-object vX`                                               |                      1 | `vX` 必须是第一个参数寄存器（实例方法通常为 `this`）                                                         | `ReturnFirstArgMethod(ArtMethod*, int32_t)`         |
| 返回常量 0/1                          | `const vX, 0/1` + `return vX`                                    |                      2 | 只允许常量 0 或 1；部分实现对 float 返回类型不处理                                                          | `ReturnZero` / `ReturnOne`                          |
| 实例字段 getter（primitive/object）     | `iget* vTmp, vThis, field` + `return* vTmp`                      |                      3 | receiver 必须为 `this`；方法静态性受限（通常拒绝 static 访问实例字段）；字段非 volatile；字段 offset 归一化后必须落在有限范围与离散集合 | `ReturnFieldAt<off,T>` / `ReturnFieldObjectAt<off>` |
| 实例字段 setter（primitive/object）     | `iput* vArg1, vThis, field` + `return-void`                      |                      3 | 写入值寄存器必须为 `this` 后第一个参数；字段非 volatile；offset 约束同上；final 字段仅允许构造期写并需要内存序处理                 | `SetFieldAt...` / `ConstructorSetField...`          |
| 静态字段 getter（object）               | `sget-object vTmp, field` + `return-object vTmp`                 |                      3 | 通常仅处理 `SGET_OBJECT`；字段必须属于本类；非 volatile；offset 约束同上                                      | `ReturnStaticFieldObjectAt<off>`                    |
| 构造函数：super 调用 + 单字段写（两种顺序）        | `iput*...` + `invoke-direct Object.<init>` + `return-void` 或相反顺序 |                      6 | 必须为 `<init>` 且父类为 `Object`；`Object.<init>` 必须匹配空构造；字段写需满足 setter 全约束（含 final fence 等）    | `ConstructorSetField...` 等                          |

---

## 4. 关键约束与设计动机

Small Pattern Matcher 的严格约束主要服务于“语义等价”与“实现成本可控”两个目标。

### 4.1 volatile 字段拒绝

volatile 读写对内存序有明确要求，通常需要插入内存屏障或使用特定原子操作语义。为了避免 stub 未严格实现导致并发可见性错误，matcher 通常直接拒绝 volatile 字段的匹配。

### 4.2 空指针检查语义与 static 方法限制

对实例字段访问而言，常规 quick 机器码往往依赖“隐式空指针检查”（通过实际访存触发 fault 并转换为 NPE）。stub 走 C++ 路径时若不实现等价机制，将导致异常语义偏差。为降低风险，matcher 对“静态方法访问实例字段”等场景通常采取拒绝策略，确保不引入行为差异。

### 4.3 offset 范围与离散化

字段读写 stub 多通过模板实例化生成一组有限的函数指针，例如按 offset `{0,4,8,...,64}` 形成固定表。matcher 将字段 offset 归一化后限制在有限范围并要求落在离散集合中，目的是：

* 避免为任意 offset 生成无限多版本 stub
* 用较小的 stub 表覆盖最常见对象布局情形
* 控制代码尺寸与维护成本

### 4.4 final 字段在构造期写入的特殊处理

对 final 字段的写入，在构造函数阶段通常需要特定的发布语义（例如构造结束前后确保可见性与顺序）。因此 matcher 对 final setter 往往仅允许“构造期写 final”的特定形状，并在 stub 内加入必要的内存序操作。

---

## 5. stub 形式说明：以 ReturnFirstArgMethod 为例

典型 stub 可能具有如下签名：

```cpp
static int32_t ReturnFirstArgMethod(ArtMethod* method, int32_t first_arg);
```

其用途是为“返回第一个参数”的方法提供通用实现。

---

## 6. 结论

Small Pattern Matcher 是 ART 编译体系中的一种工程化优化：通过对 Dex 极小方法体进行形状识别，在满足严格语义约束的前提下，以共享 stub 替代专用机器码生成，减少编译成本并提升整体运行效率。其覆盖面刻意保持有限，但对常见模式（空方法、常量返回、简单 getter/setter、简化构造函数）具有稳定收益。