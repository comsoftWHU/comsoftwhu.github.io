---
layout: default
title: Call Frame Information: asynchronous or synchronous?
nav_order: 5
parent: 编译
author:  Anonymous Committer
---

# Asynchronous Call Frame Information

## 0. TL;DR

1. **`-fno-asynchronous-unwind-tables` 不等价于 `-fno-unwind-tables`。**
它关闭的是“异步精确”的 DWARF unwind table 生成；若 C++ 异常、`-fexceptions`、`-funwind-tables` 或 ABI 仍要求 unwind table，`.so` 里仍可能存在 `.eh_frame`，但粒度可能只满足普通同步 unwinding，而不是任意指令地址精确 unwinding。GCC 文档说明 `-funwind-tables` 只生成所需静态数据，不影响生成代码；`-fasynchronous-unwind-tables` 生成的 DWARF 表在每个指令边界精确，用于 debugger 或 GC 等异步事件。LLVM 也把 `uwtable(sync)` 定义为 normal unwind tables，把 `uwtable(async)` 定义为 asynchronous / instruction precise unwind tables。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html))

1. **只支持 sync unwind，对 C++ exception / 正常 `_Unwind_*` 路径通常可接受，但前提是仍有 `.eh_frame` FDE。**
C++ 异常和 forced unwinding 走语言运行时、personality routine、LSDA、FDE 等约定；这些主要需要“正常异常传播点”的 unwind 信息。GCC 也提示 C 代码如果要和 C++ exception handlers 互操作，可能需要 `-fexceptions`。([itanium-cxx-abi.github.io](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html))

1. **受影响最大的是异步栈回溯：perf DWARF call graph、eBPF profiler、crash reporter、信号处理器 backtrace、debugger 在任意 PC 停止、GC/safepoint 外扫描。**
因为 sample、signal、SIGSEGV、breakpoint、single-step 都可能发生在函数 prologue/epilogue 或任意非 call-site 指令上；sync CFI 不保证这些位置的 CFA / return address / saved register 规则正确。GCC 对 async unwind table 的描述正是“每个指令边界精确，可用于异步事件”。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html))

1. **对 `.so` 的实际风险不是“所有 unwind 都坏”，而是“可观测性和崩溃诊断质量显著下降”。**
正常异常传播、同步 `backtrace()`、同步 libunwind 栈走查往往仍可用；但 CPU profiler、crash 栈、信号中栈、debugger 任意停点、采样型观测工具会更容易截断、错帧、缺帧，尤其在 `-fomit-frame-pointer`、tail call、inline、hand-written asm、复杂 prologue/epilogue、shrink wrapping 等情况下。

1. **发布 `.so` 的推荐策略：**
   * 需要 C++ exception / pthread cancellation 穿过该 `.so`：至少保留 `-fexceptions` 或 `-funwind-tables`，不要用 `-fno-unwind-tables` 破坏 FDE。
   * 需要线上 profiling / crash diagnostics：保留 `-fasynchronous-unwind-tables`，或至少启用 `-fno-omit-frame-pointer` 作为可观测性兜底。
   * 只追求 size，且接受 perf/crash/debugger 异步回溯降级：可以考虑 `-fno-asynchronous-unwind-tables`，但需要专项验证。

---

## 1. CFI 的标准与 ELF `.so` 中的实际格式

### 1.1 DWARF CFI 的标准语义
DWARF v5 第 6.4 节定义 **Call Frame Information**。标准说 debugger 经常需要查看/修改调用栈上任意子程序激活记录的状态；这些状态包括代码位置、栈上的 call frame、CFA（Canonical Frame Address）和寄存器集合。DWARF 将 CFA 通常定义为“上一帧 call site 处的 stack pointer 值”，并用虚拟 unwind 的方式恢复上一帧的 CFA、代码位置和寄存器状态。([DWARF](https://dwarfstd.org/doc/DWARF5.pdf))

DWARF CFI 抽象上是一张按代码位置索引的规则表：第一列是 LOC，后面是 CFA 和各寄存器列；在 shared object 中，LOC 是 object-relative offset。CFA 列给出如何计算 Canonical Frame Address，通常是 register + signed offset，也可以是 DWARF expression。寄存器列描述前一帧中该寄存器值如何恢复，例如 `undefined`、`same value`、`offset(N)`、`register(R)`、`expression(E)` 等。([DWARF](https://dwarfstd.org/doc/DWARF5.pdf))

DWARF 标准里的标准 debug section 是 `.debug_frame`；其中包含 CIE（Common Information Entry）和 FDE（Frame Description Entry）。FDE 描述一个函数或代码范围：`initial_location` 是范围起点，`address_range` 是覆盖的指令字节数，`instructions` 是 call frame instructions。([DWARF](https://dwarfstd.org/doc/DWARF5.pdf))

重要点：**DWARF 标准本身没有把表分成 sync / async 两种格式。** sync/async 是编译器和 ABI 层面的“生成粒度/保证”差异：同样是 CFI 编码，async 表保证任意指令边界可 unwind，sync 表通常只保证普通异常传播所需的位置。

---

### 1.2 ELF `.so` 中通常使用 `.eh_frame`，不是 `.debug_frame`
在 Linux/ELF `.so` 中，运行时异常与 unwind 通常依赖 `.eh_frame` 和 `.eh_frame_hdr`。LSB 规范说明：支持异常的语言，例如 C++，需要向运行环境提供描述 call frame unwinding 的额外信息，这些信息包含在特殊 section `.eh_frame` 与 `.eh_framehdr` 中；`.eh_frame` 格式与 DWARF `.debug_frame` 类似，但存在细微差异。`.eh_frame` 由一个或多个 CFI records 组成，每个 record 包含 CIE 和一个或多个 FDE。([Linux 基金會規範](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html))

`.eh_frame` 的 CIE augmentation string 约定很关键：
`L` 表示 LSDA 指针编码，`P` 表示 personality routine 指针编码，`R` 表示 FDE 中地址指针的编码方式。LSB 明确说 personality routine 用于处理语言和 vendor-specific tasks，system unwind library 通过 personality routine 指针访问语言特定异常语义。([Linux 基金會規範](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html))

FDE 中的核心字段是 `PC Begin`、`PC Range` 和 `Call Frame Instructions`。`PC Begin` 是该 FDE 关联的初始代码地址，`PC Range` 是该 FDE 覆盖的指令字节范围，`Call Frame Instructions` 是具体的 CFI 指令流。([Linux 基金會規範](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html))

`.eh_frame_hdr` 是索引/加速结构，包含 `.eh_frame` 指针以及可选的二分查找表；表项按 initial location 递增排序。异常运行时和 unwind 库常用它快速从 PC 找到对应 FDE。([Linux 基金會規範](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html))

---

### 1.3 CIE/FDE、personality、LSDA、`_Unwind_*`
Itanium C++ ABI 的 exception handling 约定已被 ELF/C++ 运行时广泛采用。该 ABI 定义 `_Unwind_Context`、`_Unwind_Exception` 以及 `_Unwind_*` 接口；编译器会把语言/vendor-specific 的 personality routine 存在需要异常处理的栈帧 unwind descriptor 中，unwinder 调用 personality routine 来处理语言特定任务，例如判断某帧是否捕获异常。ABI 还区分普通异常 unwinding 和 forced unwinding，例如 `longjmp` 或线程终止。([itanium-cxx-abi.github.io](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html))

GCC internals 中也列出了语言无关异常处理例程，例如 `_Unwind_Find_FDE`、`_Unwind_ForcedUnwind`、`_Unwind_RaiseException`、`_Unwind_Resume`、`_Unwind_GetIP`、`_Unwind_GetLanguageSpecificData` 等。这些 API 与 `.eh_frame` / FDE / personality / LSDA 共同构成运行时 unwind 机制。([GCC](https://gcc.gnu.org/onlinedocs/gccint/Exception-handling-routines.html))

---

## 2. 编译器选项语义：sync vs async

### 2.1 GCC
GCC 文档说明，大多数 `-f...` 选项都有正负形式，`-ffoo` 的负形式是 `-fno-foo`。因此 `-fno-asynchronous-unwind-tables` 是 `-fasynchronous-unwind-tables` 的负形式。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html))

GCC 对相关选项的定义如下：

| 选项 | GCC 语义 | 对 CFI 的意义 |
| --- | --- | --- |
| `-fexceptions` | 启用异常处理；生成传播异常所需额外代码；某些目标上会为所有函数生成 frame unwind information。C++ 默认启用，C 默认不启用。 | C++ exception / cleanup / landing pad 所需。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html)) |
| `-funwind-tables` | 类似 `-fexceptions`，但只生成所需静态数据，不以其他方式影响生成代码。 | 生成 unwind table，但不启用完整异常语义。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html)) |
| `-fasynchronous-unwind-tables` | 生成 DWARF unwind table；表在每个指令边界精确，可用于 debugger 或 GC 等异步事件。 | 这是 async unwind 的关键保证。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html)) |
| `-fnon-call-exceptions` | 允许 trapping instructions 抛异常；该选项启用 `-fexceptions`，但并不允许任意 signal handler 抛异常。 | 若异常可从非 call 指令产生，仅 sync call-site unwind 可能不足。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html)) |

GCC 生成的汇编会用 `.cfi_startproc` / `.cfi_endproc` 等 CFI directive 包围函数，供 assembler/linker 生成 CFI 数据。手写汇编如果缺少等价 CFI directive，就可能没有可用 FDE 或规则不完整。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html))

---

### 2.2 Clang / LLVM
Clang 命令行文档列出了 `-fasynchronous-unwind-tables` / `-fno-asynchronous-unwind-tables`，也列出了 `-funwind-tables` / `-fno-unwind-tables`。([Clang](https://clang.llvm.org/docs/ClangCommandLineReference.html))

LLVM IR 里更直接区分了 `uwtable(sync|async)`：
`uwtable(sync)` 表示生成 normal unwind tables；`uwtable(async)` 表示生成 asynchronous / instruction-precise unwind tables；未带参数的 `uwtable` 等价于 `uwtable(async)`。LLVM 还说明 x86-64 ELF ABI 通常要求为函数产生 unwind table entry，即使可以证明没有异常穿过该函数，但某些 compilation unit 可以禁用。([LLVM](https://llvm.org/docs/LangRef.html))

因此，**sync unwind 的准确含义可以理解为：保留普通 unwind table，但不承诺每个指令边界可 unwind。**
`-fno-asynchronous-unwind-tables` 的合理预期是把 async 粒度降为 sync 粒度；但如果某个 C compilation unit 没有 `-fexceptions`、没有 `-funwind-tables`、目标 ABI 也未强制 `uwtable`，那么关闭 async 后可能连该函数的 FDE 都不生成。

---

## 3. 当前使用 CFI 的工具与它们的约定

### 3.1 运行时异常：C++ exception、Rust/C++ FFI、pthread cancellation、`_Unwind_*`
**依赖方式。**
C++ exception 通过 `_Unwind_RaiseException`、personality routine、LSDA 和 FDE 完成两阶段 unwind。Itanium ABI 明确 personality routine 存放在栈帧 unwind descriptor 中，并由 unwinder 调用以完成语言特定处理；forced unwinding 也通过 `_Unwind_ForcedUnwind` 一类接口完成。([itanium-cxx-abi.github.io](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html))

**约定。**
这些运行时通常消费 `.eh_frame`，而不是 `.debug_frame`；`.eh_frame` 的 CIE/FDE、augmentation `P`/`L`、FDE PC range、call frame instructions 是关键。LSB 对这些字段和 augmentation 的定义很明确。([Linux 基金會規範](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html))

**sync-only 影响。**
若 `.so` 仍保留正常 FDE 和异常相关 LSDA/personality，普通 C++ exception 穿过该 `.so` 通常不受影响。若对 C/C++ 混合库使用 `-fno-asynchronous-unwind-tables` 但没有 `-funwind-tables` / `-fexceptions`，导致某些 C 函数没有 FDE，那么异常穿过这些帧、pthread cancellation、forced unwind 可能失败或行为异常。Red Hat 对 DWARF CFI 的概述也指出，DWARF CFI 通常用于 C++ exceptions 和 POSIX Thread cancellation。([Red Hat Developer](https://developers.redhat.com/articles/2023/07/31/frame-pointers-untangling-unwinding))

---

### 3.2 `libgcc_s` / LLVM libunwind / nongnu libunwind / libdw
**依赖方式。**
这些库负责从当前或远端上下文中逐帧恢复 caller frame。libunwind 文档描述了典型流程：先 `unw_getcontext()` 获取寄存器快照，再 `unw_init_local()` 初始化 cursor，然后重复 `unw_step()` 走到更早的栈帧；remote unwinding 也有类似机制，只是通过 callback 读取远端进程内存、寄存器和 unwind information。([man.archlinux.org](https://man.archlinux.org/man/extra/libunwind/libunwind.3.en))

**约定。**
libunwind 的文档入口明确包含 local unwinding、remote unwinding、`unw_backtrace()`、`unw_get_proc_info()` 等 API；其 wiki 也说明 DWARF-2/3 unwind conventions 见 DWARF 6.4，并提醒 GCC 使用扩展版本，其中一部分由 LSB 记录。([nongnu.org](https://www.nongnu.org/libunwind/docs.html))

**sync-only 影响。**
库本身能否成功取决于调用场景：
同步 `unw_backtrace()`、日志路径 `backtrace()`、异常传播路径通常落在 call-site 或函数调用后的返回地址上，sync CFI 多数可工作。异步信号、采样、崩溃现场、ptrace 停在任意 PC 时，sync CFI 不保证当前 PC 处规则正确，第一帧或中间帧更容易断。

---

### 3.3 glibc `backtrace()`
glibc manual 定义 backtrace 是当前线程活动函数调用列表；`backtrace()` 将一组指针写入用户提供的 buffer，用于日志或诊断。该 API 本身不等于 DWARF 规范，但其可靠性取决于底层 unwind 机制、frame pointer、CFI、符号信息等。([sourceware.org](https://sourceware.org/glibc/manual/2.43/html_node/Backtraces.html))

**sync-only 影响。**
从普通函数中主动调用 `backtrace()`，上一层帧通常是从 call instruction 返回后的地址，sync CFI 往往够用。若在 signal handler、崩溃处理器、异步采样回调中调用，则当前 PC 可能位于任意指令边界，sync-only 风险上升。glibc manual 还标注 `backtrace` 有 AS-Unsafe 等安全属性，这也意味着不要把它当作可靠的 async-signal-safe 机制。([sourceware.org](https://sourceware.org/glibc/manual/2.43/html_node/Backtraces.html))

---

### 3.4 GDB
GDB 的 `backtrace` 命令展示程序如何到达当前位置，从 frame zero 当前执行帧开始，再到 caller frames。([sourceware.org](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Backtrace.html))

GDB 源码中有专门的 `dwarf2-frame.c`，注释为 “Frame unwinder for frames with DWARF Call Frame Information”；其 FDE 结构包含 initial location、address range、instruction sequence，并区分 FDE 是否来自 `.eh_frame` 而不是 `.debug_frame`。([Chromium](https://chromium.googlesource.com/chromiumos/third_party/gdb/%2B/0f2bd1cfe3c10d7249ca64a51d7c59eed08bed21/gdb/dwarf2-frame.c))

**sync-only 影响。**
GDB 常见场景是 breakpoint、watchpoint、signal、single-step、core dump。它们都可能停在任意 PC，而不是普通 call-site。sync-only CFI 可能导致 prologue/epilogue 中栈回溯错误、参数/寄存器恢复不准确、frame 截断。若 `.debug_frame` 存在、frame pointer 存在、或 GDB 的 prologue analysis 成功，可能有 fallback；但 stripped release `.so` 通常主要依赖 `.eh_frame`。

---

### 3.5 LLDB
LLDB 的 `UnwindPlan` 源码注释说明：`UnwindPlan` 是 DWARF CFI 编码信息的 expanded table form；unwind source 可以是 `.eh_frame` FDE、DWARF `.debug_frame` FDE，或基于汇编 prologue analysis 的信息；`UnwindPlan` 是 unwinder walk stack 时使用的 canonical form。([lldb.llvm.org](https://lldb.llvm.org/cpp_reference/UnwindPlan_8h_source.html))

**sync-only 影响。**
与 GDB 类似，LLDB 在 attach、breakpoint、signal、core dump 上需要任意 PC 的 unwind 规则。只支持 sync CFI 会降低任意停点处的 unwind 成功率，尤其在无 frame pointer、无 debug info、优化级别较高的 `.so` 上。

---

### 3.6 Linux `perf`
`perf record -g` 开启 call graph / stack chain / backtrace 记录。`--call-graph` 对 user space 支持 `fp`、`dwarf` 和 `lbr`：其中 `dwarf` 明确是 DWARF CFI / Call Frame Information；当二进制用 `--fomit-frame-pointer` 构建时，`fp` 方法可能产生 bogus call graphs，若 perf 链接了 libunwind 或 libdw，可以使用 `dwarf`。使用 `dwarf` 时 perf 还会记录用户栈 dump，默认 8192 字节。([man7.org](https://man7.org/linux/man-pages/man1/perf-record.1.html))

**sync-only 影响：高。**
perf 采样发生在任意指令上。若 `.so` 只有 sync CFI，则 DWARF call graph 可能在样本 PC 所在函数 unwind 失败，或在 prologue/epilogue、stack realignment、callee-saved register 尚未保存/已恢复的位置得到错误 CFA。
替代方案：
* `--call-graph fp`：不依赖 DWARF CFI，但要求 frame pointer 链可靠。
* `--call-graph lbr`：不要求编译器选项，但只支持特定 Intel 平台、只得到用户态分支链，perf 文档也列出其限制。([man7.org](https://man7.org/linux/man-pages/man1/perf-record.1.html))

---

### 3.7 eBPF / continuous profiling：Parca、Elastic Universal Profiling 等
现代 eBPF profiler 为了在无 frame pointer 的用户程序上取栈，会把 `.eh_frame` 解析成 BPF 友好的表。Polar Signals 的文章说明，BPF helper `bpf_get_stackid(..., BPF_F_USER_STACK)` 走 frame pointer；完整 DWARF unwinder 不太可能放进内核，所以他们在用户态解析 `.eh_frame`，生成按 PC 排序的 unwind table，再由 BPF 程序查表。([Polar Signals](https://www.polarsignals.com/blog/posts/2022/11/29/dwarf-based-stack-walking-using-ebpf))

Elastic 也描述了类似结构：当遇到无 frame pointer 的 executable 或 shared object 时，用户态 agent 打开该文件，读取 `.eh_frame`，转换成更适合 eBPF 使用的数据结构，再提供给内核 BPF 代码。([Elastic](https://www.elastic.co/blog/universal-profiling-frame-pointers-symbols-ebpf))

**sync-only 影响：高。**
continuous profiler 的样本 PC 高基数、任意分布。只支持 sync CFI 会直接降低采样栈完整率和准确率。eBPF profiler 虽然可以把 CFI 预计算成 PC→CFA/RA 规则表，但如果输入 CFI 本身不覆盖任意指令边界，就无法凭空恢复 async 精度。

---

### 3.8 Breakpad / Crashpad 类 crash reporter
Breakpad stack walking 文档说明，查找 caller frame 时通常先查询详细 unwind 信息：Windows 用 PDB 中的 STACK WIN，DWARF CFI 从 binary 中提取为 STACK CFI；找到 unwind info 后，用规则从当前寄存器状态和线程栈内存恢复 caller frame。若没有 unwind info，才尝试 frame pointer 等 fallback。([GitHub](https://github.com/google/breakpad/blob/main/docs/stack_walking.md))

Breakpad 的 DWARF reader 注释也说明，DWARF CFI 描述如何 unwind stack frames，即使函数不遵循固定寄存器保存约定、frame size 执行中变化等；CFI 在每条机器指令处描述如何计算 stack frame base、return address 以及 caller 寄存器保存位置。([Chromium](https://chromium.googlesource.com/breakpad/breakpad/src/%2B/refs/heads/chrome_55/common/dwarf/dwarf2reader.h))

**sync-only 影响：高。**
crash PC 通常是任意 faulting instruction，不是 call-site。若 fault 正好发生在 prologue/epilogue、栈指针调整后但 CFI 行没有更新的位置、寄存器保存前后不一致的位置，crash reporter 的第一步 unwind 可能失败；第一步失败后，后续父帧即使在 return address 上也拿不到。若 crash 是 `abort()`、assert、日志路径主动生成 minidump，则更接近同步场景，影响较小。

---

### 3.9 Sanitizers：ASan/TSan/MSan/LSan/UBSan 运行时栈
Clang AddressSanitizer 文档建议，为了更好的错误栈，加 `-fno-omit-frame-pointer`；为了更完美的栈，还可能需要禁用 inlining 和 tail call elimination。([Clang](https://clang.llvm.org/docs/AddressSanitizer.html))

Sanitizer common flags 中，`fast_unwind_on_check`、`fast_unwind_on_fatal`、`fast_unwind_on_malloc` 等选项说明 sanitizer 有 fast frame-pointer-based unwinder；`malloc_context_size` 控制 allocation/deallocation 保留的栈帧数。([GitHub](https://github.com/google/sanitizers/wiki/SanitizerCommonFlags))

**sync-only 影响：中等。**
sanitizer 在 malloc/free、错误报告等同步路径上取栈时，sync CFI 通常比异步采样更容易工作；但 sanitizer 默认/推荐经常依赖 frame pointer 的 fast unwinder。若无 frame pointer、又禁用 async CFI，fatal signal、SEGV handler、崩溃现场报告的栈质量会下降。对于 allocation stack trace，`-fno-omit-frame-pointer` 比 async CFI 更直接改善 fast unwinder 的质量。

---

### 3.10 静态检查与 dump 工具：`readelf`、`llvm-dwarfdump`、`objdump`
`readelf --unwind` 可以显示 unwind section；若架构支持不完整，文档建议用 `--debug-dump=frames` 或 `--debug-dump=frames-interp` dump `.eh_frame` 内容。`readelf --debug-dump=frames` 显示 `.debug_frame` 原始内容，`frames-interp` 显示解释后的 frame 信息。([sourceware.org](https://sourceware.org/binutils/docs/binutils/readelf.html))

`llvm-dwarfdump` 支持 `--debug-frame` 和 `--eh-frame`，并说明这两个选项是 aliases；默认只显示 `.debug_info`，指定 section option 才 dump 对应 section。([LLVM](https://llvm.org/docs/CommandGuide/llvm-dwarfdump.html))

**sync-only 影响：低。**
这些工具只是读取和展示现有 CFI。`-fno-asynchronous-unwind-tables` 会让它们看到更少或更粗粒度的 CFI 行，但工具功能本身不坏。它们可用于验证 `.so` 是否仍有 FDE、FDE 覆盖范围是否完整、CFI row 是否覆盖 prologue/epilogue 关键点。

---

## 4. 对 `.so` 功能的影响矩阵

| 场景 / 工具 | 是否依赖 CFI | `-fno-asynchronous-unwind-tables` + 只支持 sync 的影响 | 风险等级 |
| --- | --- | --- | --- |
| C++ exception 穿过 `.so` | 是，主要 `.eh_frame` / LSDA / personality | 只要仍有 `-fexceptions` 或必要 FDE，通常可用；若 C 帧无 FDE，异常穿越可能失败。 | 中 |
| `pthread_cancel` / forced unwind | 是，通常走 `_Unwind_ForcedUnwind` 类机制 | deferred cancellation 多数接近同步 unwind；若帧缺 FDE 或异步取消在任意 PC，风险上升。 | 中 |
| `backtrace()` 同步调用 | 可能依赖 CFI 或 frame pointer | 普通日志/诊断路径通常可用；信号 handler 中调用风险高。 | 中 |
| GDB/LLDB breakpoint/signal/core | 是，可用 `.eh_frame`、`.debug_frame`、FP、prologue analysis | 任意 PC 停止时不保证正确；调试 optimized stripped `.so` 更易断栈。 | 高 |
| `perf --call-graph dwarf` | 是，明确使用 DWARF CFI | 采样 PC 任意分布；sync-only 会降低 call graph 完整性和准确性。 | 高 |
| eBPF continuous profiler | 是，常把 `.eh_frame` 预处理成 BPF 表 | 样本 PC 高基数，async 精度缺失会直接导致错帧/缺帧。 | 高 |
| Breakpad/Crashpad/minidump | 是，使用 STACK CFI / `.eh_frame` 提取信息 | fault PC 任意；第一帧 unwind 易失败，父帧可能丢失。 | 高 |
| Sanitizers | 可能依赖 CFI，但常用 frame pointer fast unwinder | 同步 allocation/error 栈影响较小；fatal signal/无 FP 时影响明显。 | 中 |
| `readelf` / `llvm-dwarfdump` | 读取 CFI | 只显示更少/更粗 CFI；工具本身不受损。 | 低 |
| `addr2line` / `llvm-symbolizer` | 通常符号化地址，不负责 unwind | 若上游 unwinder 给不出栈，symbolizer 没有地址可符号化；自身不直接受 CFI 粒度影响。 | 低 |

---

## 5. 关键边界：什么时候 sync 够，什么时候必须 async

### 5.1 sync 通常够用的情况
sync unwind 更适合这些路径：
* C++ exception 从函数调用点传播。
* 正常 `_Unwind_RaiseException` / `_Unwind_Resume`。
* 普通函数中主动调用 `backtrace()` 或 libunwind。
* 日志、诊断、错误处理代码主动收集栈。
* crash 是通过 `abort()` / assert / fatal logging 主动触发，而不是任意 faulting instruction。

这些场景的父帧 PC 多数是 call instruction 之后的 return address，和“普通 unwind table”契合。

### 5.2 async 需要保留的情况
async unwind table 对这些场景更关键：
* `perf`、Parca、Elastic 等采样型 profiler。
* signal handler 内栈回溯。
* `SIGSEGV` / `SIGBUS` / `SIGILL` / `SIGFPE` 崩溃现场。
* GDB/LLDB 在 breakpoint、watchpoint、single-step、signal 处停止。
* GC 或 runtime 在 safepoint 外扫描 native stack。
* 支持 `-fnon-call-exceptions`、硬件 trap 抛异常或类似非 call-site 异常语义。

GCC 对 `-fasynchronous-unwind-tables` 的定义非常直接：表在每个指令边界精确，因此可用于 debugger 或 garbage collector 这类异步事件。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html))

---

### 5.3 总结

| 目标 | 推荐配置 | 理由 |
| --- | --- | --- |
| 最可靠 crash/profiler/debugger 栈 | `-fasynchronous-unwind-tables`，可叠加 `-fno-omit-frame-pointer` | async CFI 提供任意指令边界规则；FP 提供快速 fallback。 |
| C++ exception 正确性优先，size 也重要 | `-fexceptions`，必要时 `-funwind-tables`，可考虑 `-fno-asynchronous-unwind-tables` | 保留同步异常传播能力，减少 async CFI size。 |
| 纯 C `.so`，可能被 C++ exception 穿过 | `-funwind-tables` 或 `-fexceptions` | 避免 C 帧无 FDE 导致跨语言异常/forced unwind 失败。 |
| 线上 continuous profiling 重要 | 保留 `-fasynchronous-unwind-tables` 或强制 `-fno-omit-frame-pointer` | perf/eBPF 样本 PC 是任意指令。 |
| 最小体积、可观测性不重要 | `-fno-asynchronous-unwind-tables`，但不要误用 `-fno-unwind-tables` | 只牺牲 async 栈质量，不破坏正常 unwind。 |
| hand-written asm / trampoline | 显式写 `.cfi_*`，并测试 FDE | 编译器不会自动为任意手写栈操作生成正确 CFI。 |


* **C++ exception / 正常同步 unwind：**
只要 `.eh_frame` FDE、personality、LSDA 仍存在，通常可用。不要同时破坏 `-fexceptions` / `-funwind-tables`。
* **调试器、采样 profiler、crash reporter、signal backtrace：**
功能不会完全消失，但栈质量会降级；最典型表现是首帧 unwind 失败、栈截断、缺中间帧、错误 caller、崩溃报告不可用。
* **对外发布 `.so`：**
若 `.so` 是基础库、运行在线上、需要被 perf/eBPF/Crashpad/GDB 诊断，建议保留 async CFI，或至少启用 frame pointer。若 `.so` 是 size 极端敏感、异常路径可控、线上诊断可接受降级的组件，可以使用 `-fno-asynchronous-unwind-tables`，但要保留普通 unwind tables 并跑上述验证。

---

## 主要引用源

* DWARF Debugging Information Format Version 5，第 6.4 节 Call Frame Information。([DWARF](https://dwarfstd.org/doc/DWARF5.pdf))
* Linux Standard Base `.eh_frame` / `.eh_frame_hdr` / DWARF EH encoding 规范。([Linux 基金會規範](https://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html))
* GCC Code Generation Options：`-fexceptions`、`-funwind-tables`、`-fasynchronous-unwind-tables`、`-fnon-call-exceptions`。([GCC](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html))
* Clang Command Line Reference 与 LLVM LangRef：`-fasynchronous-unwind-tables`、`-funwind-tables`、`uwtable(sync|async)`。([Clang](https://clang.llvm.org/docs/ClangCommandLineReference.html))
* Itanium C++ ABI Exception Handling 与 GCC `_Unwind_*` 例程。([itanium-cxx-abi.github.io](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html))
* glibc `backtrace()` manual。([sourceware.org](https://sourceware.org/glibc/manual/2.43/html_node/Backtraces.html))
* libunwind documentation / man page。([nongnu.org](https://www.nongnu.org/libunwind/docs.html))
* GDB backtrace manual、GDB DWARF frame unwinder source、LLDB UnwindPlan source。([sourceware.org](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Backtrace.html))
* Linux `perf record` manual。([man7.org](https://man7.org/linux/man-pages/man1/perf-record.1.html))
* Breakpad stack walking / DWARF CFI reader docs。([GitHub](https://github.com/google/breakpad/blob/main/docs/stack_walking.md))
* Clang AddressSanitizer docs 与 SanitizerCommonFlags。([Clang](https://clang.llvm.org/docs/AddressSanitizer.html))
* eBPF DWARF unwinding examples：Polar Signals、Elastic。([Polar Signals](https://www.polarsignals.com/blog/posts/2022/11/29/dwarf-based-stack-walking-using-ebpf))
* `readelf` 与 `llvm-dwarfdump` 文档。([sourceware.org](https://sourceware.org/binutils/docs/binutils/readelf.html))
