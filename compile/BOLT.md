---
layout: default
title: LLVM BOLT (Binary Optimization and Layout Tool)
nav_order: 1
parent: 编译
author: Anonymous Committer
---

# LLVM Binary Optimization and Layout Tool

原文档
<https://llvm.org/devmtg/2024-03/slides/practical-use-of-bolt.pdf>

## Introduction 

LLVM BOLT (Binary Optimization and Layout Tool) 是一种用于优化已编译二进制代码的工具，旨在提高程序执行的效率，尤其是针对CPU的前端瓶颈（frontend bound）进行优化。

### BOLT技术简介

BOLT主要通过重排二进制代码布局，减少指令缓存（icache）和指令翻译缓存（iTLB）未命中率，提高指令流畅性来优化程序性能。它通过以下几种方式实现这些优化：

1. **代码重新排序**：BOLT重新排序函数和基本块，使得执行路径上的代码尽可能地连续，减少跳转，提高指令缓存命中率。
2. **热路径优化**：通过分析程序运行时的性能数据，识别出程序的热路径（即执行频率最高的代码路径），并将这些路径上的代码放在更容易被缓存的区域。

### 适用场景详解

1. **CPU frontend bound workloads（CPU前端瓶颈的工作负载）**：
   - 这些工作负载在指令获取和解码阶段受到瓶颈限制，表现为CPU在等待指令缓存（icache）或指令翻译缓存（iTLB）未命中时，无法充分利用后端执行资源。BOLT通过优化代码布局，减少指令缓存未命中，从而缓解前端瓶颈，提高整体性能。

2. **大于5MB的代码（>5MB of code）**：
   - BOLT的优化效果在代码较大的应用程序中更为明显。大代码量意味着指令缓存压力较大，代码的布局和访问模式对性能影响更为显著。通过重新布局和优化，BOLT能够有效减少大代码量应用程序中的缓存未命中率。

3. **前端瓶颈超过10%（>10% FE bound）**：
   - 如果程序的前端瓶颈超过10%，意味着在运行过程中有显著的一部分时间花费在等待指令获取和解码上。BOLT的优化策略正是针对这种前端瓶颈，通过提高指令缓存命中率，减少前端等待时间。

4. **指令缓存未命中率大于10（>10 icache MPKI）**：
   - MPKI（Misses Per Kilo Instructions，指每千条指令的缓存未命中次数）是衡量指令缓存效率的一个指标。超过10的MPKI值表示指令缓存未命中率较高，程序性能受到较大影响。BOLT通过优化代码布局，减少指令缓存未命中次数，从而改善性能。

## 二进制的要求

### 1. Linux ELF X86 和 AArch64

- **支持的平台**：BOLT主要支持Linux下的ELF格式的二进制文件，尤其是X86和AArch64架构。
- **实验性支持**：BOLT还在实验性支持RISC-V架构和MacOS的MachO格式二进制文件，这些平台的支持尚在开发和完善中。

### 2. 重定位（Relocations）

- **函数重排序**：BOLT通过重排序函数和基本块来优化二进制代码布局。这一过程依赖于重定位信息，以便在重新布局代码时保持正确的地址引用。
- **选项 `-Wl,--emit-relocs`**：为了使BOLT能够重排序函数，编译时需要链接器生成并保留重定位信息。使用此选项确保生成的二进制文件包含必要的重定位信息。

### 3. PLT优化（Procedure Linkage Table Optimizations）

- **PLT优化**：BOLT可以进一步优化PLT（过程链接表），以减少间接跳转和函数调用的开销。
- **选项 `-Wl,-znow`**：此选项用于在链接时启用PLT优化，强制所有符号在程序启动时解析，而不是在第一次调用时解析。这有助于减少运行时开销。

### 4. 不支持剥离符号和拆分函数

- **剥离符号**：剥离符号的二进制文件去除了调试和符号信息，这会影响BOLT的优化过程，因为BOLT需要这些信息来进行重排序和优化。因此，输入的二进制文件不能是剥离符号的。

- **拆分函数**：GCC和LLVM编译器可以启用一些优化选项，如拆分函数，以便更好地组织代码。这些优化选项与BOLT的优化冲突，需要禁用。

#### GCC8+ 选项

- **禁用 `-freorder-blocks-and-partition`**：从GCC 8开始，默认启用了`-freorder-blocks-and-partition`，该选项会重排基本块并拆分函数。这与BOLT的重排序优化冲突，因此需要在编译时禁用此选项。

#### LLVM 选项

- **不要启用 `-split-machine-functions`**：LLVM中的`-split-machine-functions`选项会拆分机器函数，这同样会干扰BOLT的优化过程，因此不要启用此选项。

### 示例

假设你有一个使用GCC编译的项目，你希望使用BOLT进行优化，你需要确保在编译和链接时添加以下选项：

```bash
gcc -o my_program my_program.c -Wl,--emit-relocs -Wl,-znow -fno-reorder-blocks-and-partition
```

对于使用LLVM编译的项目，命令类似：

```bash
clang -o my_program my_program.c -Wl,--emit-relocs -Wl,-znow -mllvm -disable-split-machine-functions
```

通过满足这些前提条件和配置要求，你可以确保生成的二进制文件适合BOLT的优化过程，并能最大化BOLT的优化效果。

## Profiling

### 1. 使用LBRs进行采样

**LBR（Last Branch Record）** 是一种CPU硬件功能，可以记录最近分支的历史，对于性能分析尤其有用。以下是使用LBRs进行采样的步骤：

- **命令**：`perf record -e cycles -j any,u -- /bin/ls /`
  - `perf record`：这是Linux中的性能分析工具，用于记录性能数据。
  - `-e cycles`：指定记录CPU周期事件，这通常用来测量程序的执行时间。
  - `-j any,u`：启用LBR，记录用户态（user space）程序的所有分支。
  - `-- /bin/ls /`：指定要分析的命令，这里以`/bin/ls /`为例。

这条命令的意思是使用`perf`工具记录`/bin/ls /`命令的执行过程中CPU周期和分支跳转的信息。这种方式适用于在物理机上运行的程序。

### 2. 不使用LBRs

在某些情况下，如在虚拟机中运行程序或使用ARM架构时，无法使用LBRs。这时需要使用其他方式进行性能分析：

- **在虚拟机中运行，ARM架构**：使用BOLT的插桩功能进行性能分析。
  - **BOLT插桩**：BOLT可以在二进制代码中插入额外的代码，用于收集运行时的性能数据。这种方式适用于不支持LBR的环境。

- **ARM ETM（Embedded Trace Macrocell）**：ARM ETM是一种硬件跟踪技术，可以将其输出转换为`perf`

的样本格式（带有分支堆栈信息，w/brstack format）。
  - 这种转换需要专门的工具或脚本，以便将ETM的跟踪数据转换为`perf`工具可以理解的格式。

### 3. 采样需要较长的分析时间，但几乎没有开销

采样方法需要运行较长时间以收集足够的性能数据，但对程序的性能影响很小，几乎没有额外开销。

- **增加采样频率**：`perf record -Fmax ...`
  - `-Fmax`：指定采样频率，`max`表示尽可能高的频率。这可以更密集地收集性能数据，但可能会带来一定的开销，需要根据具体情况调整。

### 4. 合并分析数据

分析过程中可能会生成多个性能数据文件，可以使用BOLT的工具将这些文件合并。

- **命令**：`merge-fdata`
  - 这个工具用于将多个性能数据文件合并为一个，以便进行统一的分析和优化。

### 5. 分析数据格式

BOLT支持多种分析数据格式，包括：

- **YAML**：一种可读性好的数据格式，适合手动编辑和查看。
- **fdata**：BOLT的自定义数据格式，适合内部处理和优化。
- **perf.data**：`perf`工具生成的标准格式文件，包含详细的性能数据。
- **pre-aggregated**：已经聚合和处理好的数据，适合直接用于优化。

这些数据格式可以根据需要进行选择和转换，以便更好地进行性能分析和优化。

## 用于二进制优化的不同选项和技术

### 1. State of the art

这些是当前LLVM编译器中最先进的优化技术：

#### Function Splitting（函数拆分）

- **选项**：`-split-functions -split-strategy=cdsplit`
- **作用**：将一个大函数拆分成多个小的部分（段落），以提高代码局部性和缓存利用率。
- **策略**：`cdsplit` 表示使用控制流图（Control Flow Graph, CFG）来决定如何拆分函数。

#### Function Reordering（函数重排序）

- **选项**：`-reorder-functions=cdsort`
- **作用**：根据调用图对函数进行重排序，使得热路径上的函数相邻，从而减少指令缓存未命中率。
- **策略**：`cdsort` 表示基于调用密度（Call Density）对函数排序。

#### Block Reordering（基本块重排序）

- **选项**：`-reorder-blocks=ext-tsp`
- **作用**：在函数内部，对基本块进行重排序，以优化指令流和分支预测。
- **策略**：`ext-tsp` 表示使用扩展的旅行商问题（Extended Travelling Salesman Problem）算法进行重排序。

### 2. Extra

这些是可以额外启用的优化选项，以进一步提高性能：

#### Use THP Pages for Hot Text（使用THP页面优化热点代码）

- **选项**：`-hugify`
- **作用**：启用透明大页面（Transparent Huge Pages, THP）来存放热点代码段，提高内存访问效率。

#### PLT Optimization（过程链接表优化）

- **选项**：`-plt`
- **作用**：优化过程链接表（PLT）以减少间接跳转开销，提高函数调用性能。

#### More Aggressive ICF（更积极的相同代码折叠）

- **选项**：`-icf`
- **作用**：更积极地进行相同代码折叠（Identical Code Folding），减少重复代码，提高代码密度。

#### Indirect Call Promotion（间接调用提升）

- **选项**：`-indirect-call-promotion`
- **作用**：将间接函数调用提升为直接调用，提高调用效率。这通常通过分析间接调用的目标并将其替换为直接调用实现。

## 调试信息相关的LLVM编译器选项和支持的功能

### 1. 调试信息默认不更新

- **选项**：`-update-debug-sections`
- **作用**：默认情况下，LLVM在进行优化时不会自动更新调试信息的部分。启用这个选项可以确保在优化过程中同步更新调试信息，使其与优化后的代码保持一致。
- **使用场景**：在进行代码优化的同时，需要保留准确的调试信息，以便后续调试和分析时使用。

### 2. 支持DWARF5

- **DWARF5**：这是调试信息格式的第五版，提供了更强大的功能和更高效的编码方式，相比之前的版本改进了调试信息的表达能力。
- **支持**：LLVM编译器支持生成和使用DWARF5格式的调试信息，这意味着可以利用DWARF5的所有新特性，如更高效的调试数据结构和更多的调试信息类型。

### 3. 支持Split DWARF

- **Split DWARF**：这是DWARF调试信息的一种优化技术，将调试信息分离到单独的文件中，以减少最终可执行文件的大小并提高链接速度。
- **支持**：LLVM支持Split DWARF，这意味着可以在编译时生成分离的调试信息文件（.dwo文件），这些文件可以独立于可执行文件存储和传输。

### 4. 能够创建加速表（accelerator tables）

- **加速表**：这些表（如`gdb_index`和`debug_names`）用于加速调试器查找和访问调试信息。加速表通过预先构建的索引，减少调试器在加载和解析调试信息时的开销。
- **`gdb_index`**：这是GNU调试器（GDB）使用的一种加速表，可以显著加快调试信息的查找速度。
- **`debug_names`**：这是DWARF5引入的一种新的加速表，用于更高效地索引和访问调试信息。

## 减少二进制文件膨胀

### 1. Reuse .text Section（重用.text段）

- **选项**：`-use-old-text`
- **作用**：在进行优化时，BOLT默认会创建一个新的`.text`段（代码段）来存放优化后的代码。启用这个选项后，BOLT将重用原来的`.text`段，而不是创建新的段。
- **好处**：通过重用原来的`.text`段，可以减少二进制文件的膨胀，避免新增的段导致文件大小增加。这对二进制文件大小敏感的应用程序特别有用。

### 2. Disable Hugify（禁用大页面优化）

- **解释**：`hugify`选项用于将热点代码对齐到2MB的大页面（Transparent Huge Pages, THP），以提高内存访问效率。但是，这样做可能会增加二进制文件的大小，因为它会在内存中为代码段分配更大的对齐空间。
- **作用**：禁用`hugify`选项可以避免将代码对齐到2MB，从而减少二进制文件的膨胀。
- **使用场景**：在优化过程中，如果不需要对热点代码进行大页面对齐，禁用`hugify`可以减少二进制文件的大小，适用于内存和存储空间有限的环境。

## 日志信息

### 1. 默认打印大量有用的信息

- **默认日志信息**：
  - **Profile quality（性能分析质量）**：BOLT会默认打印性能分析的质量信息，包括函数覆盖率和性能分析数据的新旧程度（staleness）。
    - **函数覆盖率**：指示哪些函数在性能分析中被覆盖，以及覆盖的程度。
    - **分析数据的新旧程度**：显示当前使用的性能分析数据是否过时，有助于判断是否需要更新性能分析数据以获得更准确的优化效果。

### 2. 基于动态指令计数的优化效果

- **选项**：`-dyno-stats`
- **作用**：启用基于动态指令计数的优化效果统计信息。动态指令计数表示在程序运行过程中实际执行的指令数量。
- **经验法则**：执行指令数量达到1B（10亿）条。这个数量级的指令执行通常能提供足够的数据以评估优化效果。

使用`-dyno-stats`选项后，BOLT会打印出优化前后的动态指令计数，以便评估优化的效果。例如，可以看到优化后总的指令执行次数是否减少，从而判断优化是否成功。

### 3. 如果出现问题，启用详细日志

- **选项**：`-v=2`
- **作用**：启用详细日志记录。当优化过程中出现问题时，可以使用这个选项获取更详细的日志信息，以便进行调试和排查问题。

详细日志记录可以提供以下信息：
  - 每个优化步骤的详细输出。
  - 具体的错误和警告信息。
  - 优化过程中每个函数和基本块的处理情况。

## 使用LLVM BOLT进行优化时进行调试

### 1. Build in Debug Mode（在调试模式下构建）

- **作用**：在调试模式下构建程序，以便生成包含调试信息的二进制文件。这些信息可以帮助开发者在优化和调试过程中更容易定位和解决问题。

### 2. Debug Logging（调试日志）

- **选项**：`-debug-only=bolt`
- **作用**：启用与BOLT相关的调试日志。这个选项只会启用与BOLT相关的调试信息，方便开发者专注于BOLT的调试输出。
  - **警告**：`-debug` 选项会启用所有调试信息，可能会导致输出过于庞大和难以管理。因此，建议使用`-debug-only=bolt`来限制调试信息的范围。

- **如何找到相关文件和LLVM_DEBUG**：
  - 查看相关的源文件和`LLVM_DEBUG`宏定义的位置。这些宏用于在代码中插入调试信息，开发者可以根据需要在代码中添加或修改这些宏，以便生成更有用的调试日志。

### 3. Suspect Function（可疑函数）

#### 读取CFG（Read CFG）

- **选项**：`-print-only=func.* -print-cfg`
- **作用**：只打印匹配`func.*`的函数及其控制流图（CFG）。这有助于开发者专注于特定函数，查看其控制流图以理解函数的执行路径和逻辑。
  - `-print-only=func.*`：只打印名称匹配`func.*`的函数。
  - `-print-cfg`：打印控制流图（CFG），显示函数内部的基本块及其跳转关系。

#### 查看CFG（Look at CFG）

- **选项**：`-dump-dot-all`
- **作用**：生成所有函数的控制流图，并以`.dot`格式输出。这种格式可以用Graphviz等工具进行可视化，方便开发者直观地查看和分析控制流图。

#### 跳过可疑函数（Skip Suspect Function）

- **选项**：`-skip-funcs=func.*`
- **作用**：跳过匹配`func.*`的函数，不对其进行优化或分析。这对于快速排除某些函数可能导致的问题非常有用。

### 4. Bughunter Script（Bughunter脚本）

- **作用**：Bughunter脚本是一种自动化工具，帮助开发者在BOLT优化过程中快速定位和修复错误。它可以自动执行各种测试用例，记录错误日志，并提供调试信息。
- **用途**：
  - **自动化测试**：自动执行预定义的测试用例，以验证BOLT优化后的二进制文件的正确性。
  - **错误捕捉**：自动记录和报告优化过程中出现的错误，帮助开发者快速定位问题。
  - **调试辅助**：提供详细的错误日志和调试信息，帮助开发者更高效地调试和修复问题。

### 实践示例

假设你在使用BOLT优化过程中遇到问题，并希望进行调试和错误排查，可以按照以下步骤进行：

1. **在调试模式下构建程序**：

```bash
clang -O0 -g -Wl,--emit-relocs -o my_program my_program.c
```

2. **使用BOLT进行优化，并启用调试日志和控制流图打印**：

```bash
llvm-bolt my_program -o my_program.bolt -debug-only=bolt -print-only=func.* -print-cfg -dump-dot-all
```

3. **查看生成的控制流图**

生成的 `.dot` 文件可以用 Graphviz 等工具进行可视化。例如，使用以下命令将 `.dot` 文件转换为 PDF 格式：

```bash
dot -Tpdf my_program_func.dot -o my_program_func.pdf
```

4. **使用Bughunter脚本**

运行Bughunter脚本进行自动化测试和错误捕捉：

```bash
./bughunter.sh my_program.bolt
```

通过这些步骤，你可以在调试模式下构建程序，启用BOLT相关的调试日志，查看特定函数的控制流图，并使用Bughunter脚本进行自动化测试和错误排查，从而更好地理解和解决优化过程中遇到的问题。

## Bughunter Script（Bughunter脚本）

### 作用

- **功能**：Bughunter脚本是一种自动化工具，帮助开发者在BOLT优化过程中快速定位和修复错误。特别是在某个函数导致程序崩溃的情况下，可以使用该脚本自动识别出问题函数。
- **用途**：自动执行各种测试用例，记录错误日志，并提供调试信息，帮助开发者快速定位问题函数。

### 使用方法

#### 1. 二分查找导致崩溃的函数

- **Bisecting to a function which causes a crash**：脚本会通过二分查找法自动定位导致程序崩溃的具体函数。

#### 2. 传递函数名称以重现问题

- **Pass the resulting function as --funcs=funcname to reproduce the issue**：找到导致崩溃的函数后，可以将其名称作为参数传递给BOLT，以便重现并进一步分析问题。

### 执行Bughunter脚本

#### 环境变量和命令

```bash
BOLT=/build/llvm-bolt \
BOLT_OPTIONS="-v=1" \
INPUT_BINARY=/path/to/binary \
# COMMAND_LINE="--version" or \
# OFFLINE=1 \
bolt/utils/bughunter.sh
```

- **BOLT**：指定BOLT工具的路径。
- **BOLT_OPTIONS**：设置BOLT的选项，这里设置为`-v=1`以启用详细日志。
- **INPUT_BINARY**：指定要优化和测试的输入二进制文件路径。
- **COMMAND_LINE**：可以指定命令行选项，例如`--version`，用于执行特定命令；或者设置`OFFLINE=1`进行离线测试。
- **bolt/utils/bughunter.sh**：执行Bughunter脚本。

### 输出结果

- **Output**：脚本的输出是一个文本文件，其中包含导致程序崩溃的具体函数名称。这有助于开发者快速定位问题函数，并进行针对性的调试和修复。

### 实践示例

假设你有一个二进制文件在使用BOLT优化后崩溃，可以按照以下步骤使用Bughunter脚本定位问题：

1. **设置环境变量和命令**：

```bash
export BOLT=/build/llvm-bolt
export BOLT_OPTIONS="-v=1"
export INPUT_BINARY=/path/to/binary
```

2. **运行Bughunter脚本**：

```bash
bolt/utils/bughunter.sh
```

3. **查看输出结果**：

脚本执行完毕后，会生成一个文本文件，文件中包含导致崩溃的函数名称。你可以将该函数名称传递给BOLT，以重现问题并进行进一步分析。

例如：

```bash
llvm-bolt /path/to/binary -o /path/to/output/binary --funcs=problematic_function
```

通过这些步骤，你可以使用Bughunter脚本自动定位和分析导致程序崩溃的问题函数，从而更高效地进行调试和修复。

## Performance Debugging

### 1. 如果使用BOLT优化后的二进制文件变慢

#### 检查日志（Check logs!）

- **作用**：首先要检查优化过程中生成的日志文件，这些日志可能包含关于优化过程和结果的重要信息，帮助识别问题。

#### 性能分析数据是否具有代表性？（Profile is representative? Profile is correct?）

- **作用**：确保用于优化的性能分析数据是准确和具有代表性的。如果性能分析数据不准确或不适用于目标工作负载，可能导致优化效果不佳。

#### 使用相同的二进制文件进行性能分析和优化？（Same binary used for profiling and optimization?）

- **作用**：确认进行性能分析和优化时使用的是同一个二进制文件。如果二进制文件不同，性能分析数据可能不匹配，从而导致优化效果不佳。

#### 噪音？（Noise?）

- **作用**：性能分析数据可能受到环境噪音的影响，例如其他进程的干扰。确保在一个受控的环境中收集性能数据，以减少噪音的影响。

#### 双重检查统计数据？（Double-check stats?）

- **作用**：仔细检查性能分析和优化过程中的统计数据，以确保数据的准确性和一致性。如果统计数据不准确，可能需要重新进行性能分析。

### 2. 如果问题依然存在

#### 收集BOLT优化后的二进制文件的性能数据（Collect perf.data from BOLTed binary）

- **作用**：使用`perf`工具收集BOLT优化后的二进制文件的运行时性能数据。这些数据可以帮助进一步分析性能问题。

```bash
perf record -e cycles:u -o perf.data -- /path/to/BOLTed/binary
```

#### 运行llvm-bolt-heatmap并检查布局（Run llvm-bolt-heatmap and check layout）

- **作用**：使用`llvm-bolt-heatmap`工具生成热力图，并检查代码布局。热力图可以显示代码的热点区域，帮助识别性能瓶颈和优化机会。

```bash
llvm-bolt-heatmap -o heatmap.png perf.data
```

- **分析布局**：检查生成的热力图，查看哪些代码段执行频率较高，是否有不合理的布局或热点集中情况。根据这些信息，可以进一步调整优化策略和代码布局，以提高性能。

### 实践示例

假设你在使用BOLT优化后的二进制文件运行时发现其性能变慢，可以按照以下步骤进行调试：

1. **检查日志**：

- 检查BOLT优化过程中生成的日志文件，寻找可能的警告或错误信息。

2. **验证性能分析数据**：

- 确认用于性能分析的数据是准确和具有代表性的，并确保使用相同的二进制文件进行分析和优化。

3. **收集性能数据**：

- 使用`perf`工具收集优化后的二进制文件的性能数据。

```bash
perf record -e cycles -j all,u -o perf.data -- /path/to/BOLTed/binary
```

4. **生成和分析热力图**：

- 使用`llvm-bolt-heatmap`工具生成热力图，并检查代码布局。

```bash
llvm-bolt-heatmap -o heatmap.png perf.data
```

- 分析热力图，识别性能瓶颈和不合理的代码布局，根据这些信息进行进一步优化。

通过这些步骤，你可以有效地进行性能调试，找出导致性能下降的原因，并采取相应的措施来改进和优化代码布局，提升程序的运行效率。

## LLVM BOLT与PGO（Profile-Guided Optimization）之间的交互关系

### 1. BOLT是一种上下文敏感的PGO形式

- **BOLT作为上下文敏感PGO**：
  - BOLT（Binary Optimization and Layout Tool）是一种上下文敏感的PGO形式，通过直接操作已编译的二进制文件进行优化。它利用运行时收集的性能数据，对代码布局和执行路径进行优化，从而提高程序的性能。

- **其他PGO形式**：
  - **CSSPGO**（Context-Sensitive Sample-based PGO）：基于样本的上下文敏感PGO，通过采样收集性能数据。
  - **CSIR PGO**（Context-Sensitive Inline Result PGO）：基于内联结果的上下文敏感PGO。
  - **FS-AFDO**（Feedback-Directed Optimization）：基于反馈的优化技术，通过性能分析数据指导优化过程。
  - **Propeller**：Google的一个PGO项目，通过重新布局函数和基本块来优化程序。

- **BOLT的优势**：
  - **100%精确且快速**：BOLT的优化过程直接作用于二进制文件，无需重新编译或重新链接，优化效果精确且快速。
  - **BOLT与PGO的结合**：BOLT可以与其他PGO技术结合使用，进一步提高优化效果。

### 2. 处理非零二进制间隙（Dealing with non-zero binary

 gap）

- **选项**：`-infer-stale-profile`
- **作用**：启用“过时配置文件匹配”（Stale Profile Matching），该技术在CC 2024会议上提出，用于处理非零二进制间隙的问题。
  - **非零二进制间隙**：指优化后的二进制文件与原始性能分析数据之间可能存在的不一致。这种不一致可能导致性能分析数据无法准确反映优化后的二进制文件的执行情况。
  - **过时配置文件匹配**：通过推断和匹配过时的性能分析数据，BOLT可以更好地处理这种不一致，从而提高优化效果。

### 3. 从BOLT优化后的二进制文件收集BOLT性能分析数据（Collecting BOLT profile from BOLTed binary）

- **选项**：`-enable-bat`
- **作用**：启用从BOLT优化后的二进制文件收集BOLT性能分析数据的功能。
  - **WIP（正在进行中）**：此功能正在优化和完善中，以便更好地与过时匹配技术结合使用。
