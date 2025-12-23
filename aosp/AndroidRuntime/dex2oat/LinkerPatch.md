---
layout: default
title: LinkerPatch
nav_order: 8
parent: dex2oat
author: Anonymous Committer
---

## TL;DR

**LinkerPatch 的本质：这是 ART 自己的一套“迷你 relocation 表”。**

> LinkerPatch 主要服务于 **dex2oat 的 AOT 产物**在写 OAT/ELF 时的“回填/重定位”。
> JIT 这条线有自己的一套 code cache patch/relocation 机制，不走这套 LinkerPatch。

### 版本提示

LinkerPatch 的 **Type 枚举在不同分支会有差异**：

- 有的分支有 `kTypeAppImageRelRo`，**没有** `kMethodAppImageRelRo`。
- 有的分支新增了 `kMethodAppImageRelRo`，并提供 `MethodAppImageRelRoPatch(...)` 工厂函数；语义基本就是 “`kTypeAppImageRelRo` 的 method 版”。

---

## 问题核心：为什么需要 LinkerPatch？

编译器生成机器码时，很多地址/引用 **当场不知道**，比如：

* 要加载某个 `Class*` / `String*` / `ArtMethod*`，但它的最终地址只有在写 OAT/ELF、甚至运行时装载后才能定；
* 想在代码里做 PC-relative load/call，但目标偏移要等 layout 结束才算得准；
* 有些引用希望走 `.bss` 槽位（运行时填充），有些希望走 `.data.img.rel.ro`（装载时重定位、之后只读）。

所以 CodeGen 会先发出“占位指令”（立即数字段先填 0 或伪值），同时记录一个 `LinkerPatch`：**哪个位置要补、补成什么、目标是谁、需要哪些额外信息**。OatWriter/RelativePatcher 在最终布局确定后再回填。

---

## LinkerPatch 的成员变量

```cpp
const DexFile* target_dex_file_;
uint32_t literal_offset_ : 24;  // Method code size up to 16MiB.
Type patch_type_ : 8;

union {
  uint32_t cmp1_;               // Used for relational operators.
  uint32_t boot_image_offset_;  // Data to write to the boot image .data.img.rel.ro entry.
  uint32_t method_idx_;         // Method index for Call/Method patches.
  uint32_t type_idx_;           // Type index for Type patches.
  uint32_t string_idx_;         // String index for String patches.
  uint32_t proto_idx_;          // Proto index for MethodType patches.
  uint32_t intrinsic_data_;     // Data for IntrinsicObjects.
  uint32_t entrypoint_offset_;  // Entrypoint offset in the Thread object.
  uint32_t baker_custom_value1_;
};

union {
  size_t   cmp2_;             // Used for relational operators.
  uint32_t pc_insn_offset_;   // The insn providing PC (e.g., ADRP on AArch64).
  uint32_t baker_custom_value2_;
};
```

把这些字段粗暴地分成三类就不容易迷路：

* **定位**：`literal_offset_`（主 patch 点）、`pc_insn_offset_`（PC 基准指令点）
* **分类**：`patch_type_`
* **参数**：`*_idx_ / boot_image_offset_ / entrypoint_offset_ / intrinsic_data_ / baker_custom_value*`

### `target_dex_file_`

很多 patch 的目标用的是 dex 索引（`method_idx_ / type_idx_ / string_idx_ / proto_idx_`），索引必须依附在某个 DexFile 上才有意义，所以需要带上 `target_dex_file_`。

### `literal_offset_ : 24`

补丁“主落点”：**该方法机器码内的 byte offset**。24 bit 对应上限约 16MiB（单方法机器码太大编译器通常也会直接拒掉/降级）。

> 名字叫 `literal_offset` 但不一定真的是 literal pool；历史包袱。

### `pc_insn_offset_`

专门给 **PC-relative 指令序列**用的：例如 AArch64 常见的 `ADRP + LDR/ADD`，`pc_insn_offset_` 指向 `ADRP`，`literal_offset_` 指向后面的 `LDR/ADD`（真正要改 imm 的那条）。这样 patcher 才能用正确的 PC 基准算页偏移/立即数。

### `cmp1_ / cmp2_`

它俩不是业务语义，是为了让 patch 可以被当作 raw memory 做 **比较/排序/哈希**，避免 padding 未初始化导致 hash 不稳定。

---

## LinkerPatch 补丁类型（按“目标在哪儿 / 走什么通道”来分）

> 不同 ISA（ARM64/x86_64/riscv64）补法不同，但“语义分层”相同。

### A. 目标是“编译期能确定”的引用（能直接算出地址/相对偏移）

#### 1) `kIntrinsicReference`

某些 intrinsic 需要直接引用 boot image 里的“固定对象”（`IntrinsicObjects` 管），用 `intrinsic_data_` 编码是哪一个对象。

ARM64 常见形态（示意）：

```asm
adrp  x16, <obj@page>             ; pc_insn_offset_
ldr   x16, [x16, <obj@pageoff>]   ; literal_offset_
```

#### 2) `kTypeRelative` / `kStringRelative` / `kMethodRelative`

“把目标地址编码成相对某个基准（通常是 oat data begin / code base）的偏移，然后 patch 到指令里”。

* `kTypeRelative`：目标是 `Class*`
* `kStringRelative`：目标是 `String*`
* `kMethodRelative`：目标是 `ArtMethod*`（注意：不是 call，是“拿到 method 指针”）

---

### B. 目标在 Boot Image，但代码侧不适合直接 embed：走 `.data.img.rel.ro`

#### 3) `kBootImageRelRo`

给“引用 boot image 对象，但代码本身不在 boot image 内（或不能直接算到 boot image）”的场景用：

* 把目标对象指针写进 `.data.img.rel.ro`（装载时由 linker/loader 做重定位）
* 代码只需 PC-relative 去读这张表的某个槽位

示意：

```asm
adrp x17, <.data.img.rel.ro@page>
ldr  x16, [x17, #entry_off]    ; 读出指针
```

关键点：**代码只要求能 PC-relative 到“表”，不要求能 PC-relative 到 boot image 本体。**

---

### C. 目标在运行时才解析/绑定：走 `.bss` 槽位（lazy resolve + cache）

`.bss` 这条线的共同模式是：

1. 代码先从 `.bss` 槽 load 指针
2. 空则走 resolve slow path，填槽
3. 之后热路径只做一次 load

#### 4) `kMethodBssEntry`

给 dex method 分配 `.bss` 槽，运行时填 `ArtMethod*`。

#### 5) `kTypeBssEntry / kPublicTypeBssEntry / kPackageTypeBssEntry`

给 dex type 分配 `.bss` 槽，区别是把 type entry 放在不同的分区（普通/public/package）：

* 本质目的：**在不破坏访问检查语义的前提下，提高/限制跨 dex/oat 共享与复用的粒度**（public 可更广泛复用，package 受包可见性约束）。

#### 6) `kStringBssEntry`

给 dex string 分配 `.bss` 槽，运行时填 `String*`。

#### 7) `kMethodTypeBssEntry`

给 `invoke-polymorphic` / `MethodType` 相关的 proto 分配 `.bss` 槽，运行时填 `MethodType*`。

ARM64 典型形态（示意）：

```asm
adrp x16, <.bss@page>            ; pc_insn_offset_
ldr  x16, [x16, #slot_off]       ; literal_offset_
cbnz x16, resolved
bl   <resolve_stub>              ; 慢路径填槽
resolved:
; x16 = Class*/String*/ArtMethod*/MethodType*
```

---

### D. “不是加载指针”，而是 patch **调用目标 / 分支目标** 的

#### 8) `kCallRelative`

对另一个已编译方法做相对 call（如 AArch64 `bl imm26`）。

#### 9) `kCallEntrypoint`

调用 Thread 里的 quick entrypoint（runtime stub）。patch 里存 `entrypoint_offset_`（Thread 对象内偏移），通常是先 load 出函数指针再 `blr/call`。

#### 10) `kJniEntrypointRelative`

给 **native 方法**相关的调用路径用：patch 的目标仍然用 `method_idx_` 标识，但语义是“走该方法的 JNI entrypoint/桥接入口”的那条路径（具体补法由 ISA patcher 决定）。

#### 11) `kBakerReadBarrierBranch`

Baker read barrier 专用分支补丁：编译器会生成固定模板/布局，patch 用 `baker_custom_value1_/2_` 编码 read barrier 相关信息，finalize/patch 阶段会校验模板并把分支距离补对（ARM64 后端常见会校验 patch 点附近指令形态）。

---

### 新增：`kMethodAppImageRelRo`（方法的 app image relro 表项）

语义一句话：**`kTypeAppImageRelRo` 的 method 版本**。

当你在“带 app image”的场景里，代码需要引用 **app image 侧的 `ArtMethod*`**，但 app image 的装载地址与 oat/code 不一定有固定相对关系时：

* 把该 `ArtMethod*` 放进 `.data.img.rel.ro` 的 method 表项（装载时重定位）
* 代码侧 PC-relative 去读这个表项

形态与 `kTypeAppImageRelRo` 几乎一致，只是索引从 type_idx 换成 method_idx。