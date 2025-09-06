---
layout: default
title: dex2oat编译中的优化
nav_order: 5
parent: dex2oat
author: Anonymous Committer
---
## OptimizingCompiler

在TryCompile函数中我们可以看到 Dex2oat 将字节码首先转换成 HIR 的图结构（HGraph），然后应用一系列的Optimization Pass来提升代码质量，最后生成高效的机器码。

```cpp
/**
 * 这是优化编译器的主要入口点。它接收一个 Dex 方法，
 * 并尝试将其编译为本地机器码。成功时，它返回一个包含已编译代码的 CodeGenerator 对象；
 * 失败时，返回 nullptr。
 */
CodeGenerator* OptimizingCompiler::TryCompile(ArenaAllocator* allocator,
                                              ArenaStack* arena_stack,
                                              const DexCompilationUnit& dex_compilation_unit,
                                              ArtMethod* method,
                                              CompilationKind compilation_kind,
                                              VariableSizedHandleScope* handles) const {
  // 为统计目的，记录我们正在尝试编译此方法。
  MaybeRecordStat(compilation_stats_.get(), MethodCompilationStat::kAttemptBytecodeCompilation);
  const CompilerOptions& compiler_options = GetCompilerOptions();
  InstructionSet instruction_set = compiler_options.GetInstructionSet();
  const DexFile& dex_file = *dex_compilation_unit.GetDexFile();
  uint32_t method_idx = dex_compilation_unit.GetDexMethodIndex();
  const dex::CodeItem* code_item = dex_compilation_unit.GetCodeItem();

  // 在 ARM 平台上，ART 总是生成 Thumb-2 代码以获得更好的代码密度和性能。
  // 这是一个健全性检查，确保我们没有尝试生成旧版的 ARM 代码。
  DCHECK_NE(instruction_set, InstructionSet::kArm);

  // --- 初始预编译检查，以便快速失败 ---

  // 如果编译器的此构建版本不支持目标架构，则提前退出。
  if (!IsInstructionSetSupported(instruction_set)) {
    MaybeRecordStat(compilation_stats_.get(),
                    MethodCompilationStat::kNotCompiledUnsupportedIsa);
    return nullptr;
  }

  // 检查“病态情况”：已知过于复杂或需要过长时间来编译的方法。
  // 这是一个安全网，防止编译器卡住。
  if (Compiler::IsPathologicalCase(*code_item, method_idx, dex_file)) {
    SCOPED_TRACE << "Not compiling because of pathological case";
    MaybeRecordStat(compilation_stats_.get(), MethodCompilationStat::kNotCompiledPathological);
    return nullptr;
  }

  // 如果编译过滤器设置为 'space'，我们会避免编译大型方法，
  // 以减小最终生成的 .oat 文件的大小。
  static constexpr size_t kSpaceFilterOptimizingThreshold = 128;
  if ((compiler_options.GetCompilerFilter() == CompilerFilter::kSpace)
      && (CodeItemInstructionAccessor(dex_file, code_item).InsnsSizeInCodeUnits() >
          kSpaceFilterOptimizingThreshold)) {
    SCOPED_TRACE << "Not compiling because of space filter";
    MaybeRecordStat(compilation_stats_.get(), MethodCompilationStat::kNotCompiledSpaceFilter);
    return nullptr;
  }

  CodeItemDebugInfoAccessor code_item_accessor(dex_file, code_item, method_idx);

  // 判断该方法是否是“死引用安全”的。这是一个注解，表明该方法
  // 不会访问某些敏感的 RenderScript 字段，从而允许更激进的优化。
  bool dead_reference_safe;
  if (method != nullptr) {
    const dex::ClassDef* containing_class;
    {
      ScopedObjectAccess soa(Thread::Current());
      containing_class = &method->GetClassDef();
    }
    dead_reference_safe =
        annotations::HasDeadReferenceSafeAnnotation(dex_file, *containing_class)
        && !annotations::MethodContainsRSensitiveAccess(dex_file, *containing_class, method_idx);
  } else {
    // 如果无法解析类/方法（例如，在 AOT 编译期间由于依赖损坏），
    // 我们必须保守地假设它不是安全的。
    dead_reference_safe = false;
  }

  // --- HGraph的创建和构建 ---

  // 创建 HGraph。这是核心数据结构，将用于存放方法的代码以进行分析和优化。
  HGraph* graph = new (allocator) HGraph(
      allocator,
      arena_stack,
      handles,
      dex_file,
      method_idx,
      compiler_options.GetInstructionSet(),
      kInvalidInvokeType,
      dead_reference_safe,
      compiler_options.GetDebuggable(),
      compilation_kind);

  if (method != nullptr) {
    graph->SetArtMethod(method);
  }

  // 如果我们正在进行 JIT 编译，将任何现有的性能分析信息附加到图中。
  // 这些信息（例如，热点代码路径、类型分布）对于指导优化至关重要。
  jit::Jit* jit = Runtime::Current()->GetJit();
  if (jit != nullptr) {
    ProfilingInfo* info = jit->GetCodeCache()->GetProfilingInfo(method, Thread::Current());
    graph->SetProfilingInfo(info);
  }

  // 为目标架构创建一个 CodeGenerator。该对象负责将 HIR 最终降级为机器码。
  std::unique_ptr<CodeGenerator> codegen(
      CodeGenerator::Create(graph,
                            compiler_options,
                            compilation_stats_.get()));
  if (codegen.get() == nullptr) {
    MaybeRecordStat(compilation_stats_.get(), MethodCompilationStat::kNotCompiledNoCodegen);
    return nullptr;
  }
  // 如果需要用于调试/栈回溯，则启用调用帧信息 (CFI) 的生成。
  codegen->GetAssembler()->cfi().SetEnabled(compiler_options.GenerateAnyDebugInfo());

  // PassObserver 用于在每次优化遍之后跟踪和记录图的状态。
  PassObserver pass_observer(graph,
                             codegen.get(),
                             visualizer_output_.get(),
                             compiler_options);

  {
    // 这个作用域封装了图的构建过程。
    VLOG(compiler) << "Building " << pass_observer.GetMethodName();
    PassScope scope(HGraphBuilder::kBuilderPassName, &pass_observer);
    // HGraphBuilder 负责将 Dex 字节码翻译成 HIR 图。
    HGraphBuilder builder(graph,
                          code_item_accessor,
                          &dex_compilation_unit,
                          &dex_compilation_unit,
                          codegen.get(),
                          compilation_stats_.get());
    GraphAnalysisResult result = builder.BuildGraph();

    // 如果构建图因任何原因失败，
    // 我们将中止此方法的编译。
    if (result != kAnalysisSuccess) {
      // 将该方法标记为“不要编译”，以防止未来的尝试。
      if (method != nullptr) {
        ScopedObjectAccess soa(Thread::Current());
        method->SetDontCompile();
      }
      // ... 记录具体失败原因的逻辑 ...
      pass_observer.SetGraphInBadState();
      return nullptr;
    }
  }

  // --- Optimization Passes ---

  // 对于kBaseline编译（一个快速、轻量优化的 JIT 层级），我们可能会跳过
  // 大多数优化，只运行必要的Pass来生成性能分析信息。
  if (compilation_kind == CompilationKind::kBaseline && compiler_options.ProfileBranches()) {
    graph->SetUsefulOptimizing();
    RunRequiredPasses(graph, codegen.get(), dex_compilation_unit, &pass_observer);
  } else {
    // 对于完整的优化编译，运行主要的优化流水线。
    RunOptimizations(graph, codegen.get(), dex_compilation_unit, &pass_observer);
    // 写屏障消除是一个特殊的Pass，用于使用写屏障的垃圾收集器。
    PassScope scope(WriteBarrierElimination::kWBEPassName, &pass_observer);
    WriteBarrierElimination(graph, compilation_stats_.get()).Run();
  }

  // 如果这是一次基线 JIT 编译，并且我们发现它“有用”（例如，包含循环），
  // 但它还没有性能分析信息，我们现在就构建它。如果该方法变得足够热
  // 以至于被优化编译器重新编译，这些信息将会被使用。
  if (jit != nullptr &&
      compilation_kind == CompilationKind::kBaseline &&
      graph->IsUsefulOptimizing() &&
      graph->GetProfilingInfo() == nullptr) {
    ProfilingInfoBuilder(
        graph, codegen->GetCompilerOptions(), codegen.get(), compilation_stats_.get()).Run();
    // 如果在创建性能分析信息时内存不足，则中止。
    if (graph->GetProfilingInfo() == nullptr) {
      SCOPED_TRACE << "Not compiling because of out of memory";
      MaybeRecordStat(compilation_stats_.get(), MethodCompilationStat::kJitOutOfMemoryForCommit);
      return nullptr;
    }
  }

  // --- 最后阶段：寄存器分配和代码生成 ---

  // 在所有优化之后，执行寄存器分配。这是一个关键步骤，它将 HIR 中
  // 无限的虚拟寄存器映射到有限的物理 CPU 寄存器上。
  AllocateRegisters(graph,
                    codegen.get(),
                    &pass_observer,
                    compilation_stats_.get());

  // 最后的安全检查：如果所需的栈帧太大，则中止编译。
  if (UNLIKELY(codegen->GetFrameSize() > codegen->GetMaximumFrameSize())) {
    SCOPED_TRACE << "Not compiling because of stack frame too large";
    LOG(WARNING) << "Stack frame size is " << codegen->GetFrameSize()
                 << " which is larger than the maximum of " << codegen->GetMaximumFrameSize()
                 << " bytes. Method: " << graph->PrettyMethod();
    MaybeRecordStat(compilation_stats_.get(), MethodCompilationStat::kNotCompiledFrameTooBig);
    return nullptr;
  }

  // 最后一步：指示 CodeGenerator 将优化后的、已分配寄存器的图
  // 编译成本地机器码。
  codegen->Compile();
  // 如果为调试目的而请求，则转储最终的反汇编代码。
  pass_observer.DumpDisassembly();

  MaybeRecordStat(compilation_stats_.get(), MethodCompilationStat::kCompiledBytecode);
  // 成功后，释放 CodeGenerator 的所有权并将其返回给调用者。
  return codegen.release();
}
```

可以看到主要的优化分为了 `OptimizingCompiler::RunOptimizations` 和 `OptimizingCompiler::RunArchOptimizations` 两个部分。

## IR 层级的优化

```cpp
void OptimizingCompiler::RunOptimizations(HGraph* graph,
                                          CodeGenerator* codegen,
                                          const DexCompilationUnit& dex_compilation_unit,
                                          PassObserver* pass_observer) const {
  // 检查是否有通过命令行传入的自定义优化Pass列表。
  const std::vector<std::string>* pass_names = GetCompilerOptions().GetPassesToRun();
  if (pass_names != nullptr) {
    // 如果在命令行上定义了优化Pass，则构建并运行这些Pass，
    // 而不是使用内置的默认优化流程。
    const size_t length = pass_names->size();
    std::vector<OptimizationDef> optimizations;
    for (const std::string& pass_name : *pass_names) {
      std::string opt_name = ConvertPassNameToOptimizationName(pass_name);
      optimizations.push_back(OptDef(OptimizationPassByName(opt_name), pass_name.c_str()));
    }
    RunOptimizations(graph,
                       codegen,
                       dex_compilation_unit,
                       pass_observer,
                       optimizations.data(),
                       length);
    return;
  }

  // 默认的优化流水线定义。
  OptimizationDef optimizations[] = {
      // --- 阶段一：初始优化 ---
      // 进行第一轮“打扫”，清理掉最明显的冗余代码。
      OptDef(OptimizationPass::kConstantFolding),             // 常量折叠
      OptDef(OptimizationPass::kInstructionSimplifier),        // 指令简化
      OptDef(OptimizationPass::kDeadCodeElimination,           // 死代码消除
             "dead_code_elimination$initial"),

      // --- 阶段二：函数内联 ---
      // 这是最重要的优化之一，它会为后续优化创造大量机会。
      OptDef(OptimizationPass::kInliner),

      // --- 阶段三：内联后的清理 ---
      // 内联操作会引入新的代码，因此需要立即进行一轮清理，
      // 以便简化图结构，为后续更复杂的分析做准备。
      OptDef(OptimizationPass::kConstantFolding,
             "constant_folding$after_inlining",
             OptimizationPass::kInliner),
      OptDef(OptimizationPass::kInstructionSimplifier,
             "instruction_simplifier$after_inlining",
             OptimizationPass::kInliner),
      OptDef(OptimizationPass::kDeadCodeElimination,
             "dead_code_elimination$after_inlining",
             OptimizationPass::kInliner),

      // --- 阶段四：全局值编号 (GVN) ---
      // 目标是消除跨基本块的冗余计算。
      OptDef(OptimizationPass::kSideEffectsAnalysis,          // 副作用分析 (为 GVN 提供数据)
             "side_effects$before_gvn"),
      OptDef(OptimizationPass::kGlobalValueNumbering),        // 全局值编号
      OptDef(OptimizationPass::kReferenceTypePropagation,     // 引用类型传播 (利用 GVN 的结果)
             "reference_type_propagation$after_gvn",
             OptimizationPass::kGlobalValueNumbering),

      // --- 阶段五：GVN 后的清理 ---
      // GVN 等操作会改变代码结构，同样需要一轮清理。
      OptDef(OptimizationPass::kSelectGenerator),             // 生成 Select 指令 (if-else -> a?b:c)
      OptDef(OptimizationPass::kConstantFolding,
             "constant_folding$after_gvn"),
      OptDef(OptimizationPass::kInstructionSimplifier,
             "instruction_simplifier$after_gvn"),
      OptDef(OptimizationPass::kDeadCodeElimination,
             "dead_code_elimination$after_gvn"),

      // --- 阶段六：高级循环优化 ---
      // 专注于优化程序中最耗时的部分——循环。
      OptDef(OptimizationPass::kSideEffectsAnalysis,          // 副作用分析 (为 LICM 提供数据)
             "side_effects$before_licm"),
      OptDef(OptimizationPass::kInvariantCodeMotion),         // 循环不变代码外提 (LICM)
      OptDef(OptimizationPass::kInductionVarAnalysis),         // 归纳变量分析
      OptDef(OptimizationPass::kBoundsCheckElimination),       // 数组边界检查消除 (BCE)
      OptDef(OptimizationPass::kLoopOptimization),             // 循环优化 (核心是强度削减)

      // --- 阶段七：循环优化后的清理 ---
      // 循环优化会大幅重构代码，需要一轮更激进的清理。
      OptDef(OptimizationPass::kConstantFolding,
             "constant_folding$after_loop_opt"),
      OptDef(OptimizationPass::kAggressiveInstructionSimplifier, // 激进的指令简化
             "instruction_simplifier$after_loop_opt"),
      OptDef(OptimizationPass::kDeadCodeElimination,
             "dead_code_elimination$after_loop_opt"),

      // --- 阶段八：其他高级优化 ---
      OptDef(OptimizationPass::kLoadStoreElimination),          // 加载/存储消除
      OptDef(OptimizationPass::kCHAGuardOptimization),         // 基于类层次分析的守卫优化
      OptDef(OptimizationPass::kCodeSinking),                  // 代码下沉

      // --- 阶段九：代码生成前的最终清理 ---
      OptDef(OptimizationPass::kConstantFolding,
             "constant_folding$before_codegen"),
      // 代码生成器有一些假设，只有指令简化器能满足。
      // 例如，代码生成器不希望看到一个类型到其自身类型的 HTypeConversion。
      OptDef(OptimizationPass::kAggressiveInstructionSimplifier,
             "instruction_simplifier$before_codegen"),
      // 简化过程可能产生死代码，应在代码生成前移除。
      OptDef(OptimizationPass::kDeadCodeElimination,
             "dead_code_elimination$before_codegen"),
      // 在代码下沉之后消除构造函数内存屏障，以避免
      // 复杂的下沉逻辑来分割一个有许多输入的屏障。
      OptDef(OptimizationPass::kConstructorFenceRedundancyElimination)
  };

  // 运行上面定义的平台无关的优化流水线。
  RunOptimizations(graph,
                       codegen,
                       dex_compilation_unit,
                       pass_observer,
                       optimizations);

  // 运行特定于目标 CPU 架构的优化。
  RunArchOptimizations(graph, codegen, dex_compilation_unit, pass_observer);
}

```

- `OptimizationPass::kConstantFolding` (常量折叠):  在编译期直接计算出那些结果是常量的表达式，并用结果替换掉整个表达式。

- `OptimizationPass::kInstructionSimplifier` (指令简化): 将复杂的或冗余的指令替换为等价的、但更简单或更高效的指令。这是一种“窥孔优化”（Peephole Optimization）的扩展。

- `OptimizationPass::kDeadCodeElimination` (死代码消除):  移除那些执行结果永远不会被使用，或者永远不会被执行到的代码。

- `OptimizationPass::kInliner` (内联器): 将一个函数调用的地方，用该函数的函数体代码直接替换。消除函数调用的开销（如压栈、跳转）。更重要的是，它将不同函数的代码合并到同一个上下文中，极大地扩展了其他优化的分析范围，从而触发更多的常量折叠、指令简化和死代码消除。

- `OptimizationPass::kSideEffectsAnalysis` (副作用分析): 分析代码中的指令，标记哪些指令会产生副作用（如修改全局变量、抛出异常、进行I/O操作），哪些是纯计算。后续的 GVN、LICM 等优化需要精确的副作用信息，来判断一个计算是否可以被安全地移动或删除。

- `OptimizationPass::kGlobalValueNumbering` (全局值编号): 为程序中每个计算出的值分配一个唯一的“编号”。如果两个表达式计算的是相同的值（即使它们在代码的不同位置），它们会被赋予相同的编号。这样，编译器就可以保留第一次计算，并将后续所有对相同值的计算替换为对第一次计算结果的复用。

- `OptimizationPass::kReferenceTypePropagation` (引用类型传播): 在 GVN 之后，很多关于对象类型的信息变得更加明确。此 Pass 会将这些类型信息在图中传播，让编译器更精确地了解每个引用的具体类型。 为后续的优化（如基于类层次结构的守卫优化）提供更精确的类型信息。

- `OptimizationPass::kSelectGenerator` (Select 指令生成): 识别 if-then-else 结构，并将其转换为一个 Select 指令。Select 指令类似于 C++ 的三元运算符 cond ? true_val : false_val。将分支结构转换为数据流，有助于消除分支，改善指令流水线。

- `OptimizationPass::kInvariantCodeMotion` (循环不变代码外提 - LICM): 识别那些在循环体内，但其计算结果在每次迭代中都保持不变的指令，并将它们移动到循环开始前执行。 避免在循环中进行大量重复计算。

- `OptimizationPass::kInductionVarAnalysis` (归纳变量分析): 分析循环中以固定步长变化的变量（归纳变量），并识别出它们之间的线性关系。为后续的边界检查消除和强度削减等优化提供关键信息。

- `OptimizationPass::kBoundsCheckElimination` (数组边界检查消除 - BCE): 基于归纳变量分析的结果，判断循环中的数组访问是否绝对不会越界。如果可以证明其安全性，就删除掉运行时的数组边界检查指令。

- `OptimizationPass::kLoopOptimization` (循环优化): 这是一个复合 Pass，包含多种循环转换技术，最核心的是强度削减 (Strength Reduction)。它利用归纳变量分析的结果，将循环中昂贵的乘法操作替换为廉价的加法操作。包括了简单循环的完全移除、循环向量化（包含传统的向量化和实验性的通过掩码在单条 SIMD 指令内部实现条件执行的“谓词向量化”）、零次循环消、完全展开 (Full Unrolling)、循环剥离 (Peeling)、为减少分支惩罚而展开 (Unrolling for Branch Penalty Reduction)、挂起检查消除 (Suspend Check Removal)等操作。

- `OptimizationPass::kAggressiveInstructionSimplifier`: 循环优化会彻底重构循环体内的代码，因此需要一轮更激进的指令简化清理。

- `OptimizationPass::kLoadStoreElimination` (加载/存储消除): 分析内存访问模式，消除冗余的加载和存储操作。例如，如果先将一个值存入内存，紧接着又从同一位置加载它，那么加载操作就是多余的。

- `OptimizationPass::kCHAGuardOptimization` (类层次结构分析守卫优化): 基于类层次结构分析（CHA），如果编译器能确定一个虚方法调用在当前上下文只有一个可能的实现，它会用直接调用替换虚方法调用，并在前面插入一个类型守卫（Guard）来保证这个假设的正确性。将高开销的虚调用（需要查虚表）转换为低开销的直接调用，即“去虚拟化”。

- `OptimizationPass::kCodeSinking` (代码下沉): 将指令尽可能地移动到它被使用的基本块中。特别是对于在 if 的两个分支中都需要的计算，可以将其下沉到 if 结束后的汇合点。减少代码重复，并可能减少只在特定路径上执行的代码的执行次数。

- `OptimizationPass::kConstructorFenceRedundancyElimination` (构造函数内存屏障冗余消除): 在 Java 构造函数中，为了保证 final 字段的可见性，会插入内存屏障（Fence）。此 Pass 会分析并移除那些不必要的、冗余的内存屏障。

在进入代码生成阶段前，必须进行最后一轮彻底的指令简化和死代码消除，以确保 HIR 图处于一个干净、规范的状态，满足代码生成器的各种假设。

## 架构特定优化

```cpp
bool OptimizingCompiler::RunArchOptimizations(HGraph* graph,
                                              CodeGenerator* codegen,
                                              const DexCompilationUnit& dex_compilation_unit,
                                              PassObserver* pass_observer) const {
  switch (codegen->GetCompilerOptions().GetInstructionSet()) {
  // ......
#ifdef ART_ENABLE_CODEGEN_arm64
    case InstructionSet::kArm64: {
      OptimizationDef arm64_optimizations[] = {
          OptDef(OptimizationPass::kInstructionSimplifierArm64),
          OptDef(OptimizationPass::kSideEffectsAnalysis),
          OptDef(OptimizationPass::kGlobalValueNumbering, "GVN$after_arch"),
          OptDef(OptimizationPass::kScheduling)
      };
      return RunOptimizations(graph,
                              codegen,
                              dex_compilation_unit,
                              pass_observer,
                              arm64_optimizations);
    }
#endif
  // ......
  }
}
```

- `kInstructionSimplifierArm64` : 利用特定 CPU 架构的指令集特性来简化指令。如 在 ARM64 上，`a + (b << 3)` 可被合并成一条单一的 ADD 指令，因为 ARM64 的 ADD 指令支持带移位的操作数。
- `kScheduling`: 重排指令的执行顺序，以更好地利用现代 CPU 的超标量和流水线特性。它会避免指令之间的数据依赖导致流水线停顿，并将可以并行执行的指令放在一起。
